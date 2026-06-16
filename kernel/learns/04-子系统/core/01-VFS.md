<!-- Source: subsystem/vfs.md -->

# VFS 子系统核心知识

## Inode 锁层级与 i_rwsem 子类

Linux VFS 使用 `inode->i_rwsem`（类型为 `struct rw_semaphore`）保护 inode 的多种操作。错误的锁顺序会导致死锁，错误的子类会阻止 lockdep 检测到顺序违规。

### i_rwsem 子类（enum inode_i_mutex_lock_class）

| 子类编号 | 常量 | 用途 |
|---------|------|------|
| 0 | `I_MUTEX_NORMAL` | 当前 VFS 操作的目标对象 |
| 1 | `I_MUTEX_PARENT` | 父目录 |
| 2 | `I_MUTEX_CHILD` | rename 时的子/目标子目录 |
| 3 | `I_MUTEX_XATTR` | xattr 操作 |
| 4 | `I_MUTEX_NONDIR2` | 第二个非目录（两个非目录的 rename） |
| 5 | `I_MUTEX_PARENT2` | 第二个父目录（跨目录 rename） |

### 锁顺序规则

- **父目录先于孩子**：父目录锁 (`I_MUTEX_PARENT`, subclass 1) 必须在孩子 inode 锁 (`I_MUTEX_NORMAL`, subclass 0) 之前获取。核心辅助函数 `__start_dirop()` (`fs/namei.c`) 对 mkdir, rmdir, unlink, mknod, symlink, link creation 强制这一顺序。
- **rename 需要两个父目录锁**：`lock_rename()` 首先获取 filesystem 全局的 `s_vfs_rename_mutex`（防止并发 rename 产生目录环路），然后通过 `lock_two_directories()` 锁定两个父目录。祖先目录使用 `I_MUTEX_PARENT`，后代目录使用 `I_MUTEX_PARENT2`。改变父目录的子目录使用 `I_MUTEX_CHILD`。非目录孩子使用 `I_MUTEX_NORMAL` / `I_MUTEX_NONDIR2`（通过 `lock_two_nondirectories()`）。

### inode_lock() vs inode_lock_nested()

`inode_lock()` 是以子类 `I_MUTEX_NORMAL` (0) 获取 `i_rwsem` 的便捷封装。锁定目录以在其中创建、重命名或删除条目时，该目录作为**父目录**角色，必须使用 `inode_lock_nested(dir->d_inode, I_MUTEX_PARENT)`。错误使用子类会使 lockdep 的父-子顺序检查失效。

```c
// 错误：创建操作使用默认子类
inode_lock(workdir->d_inode);
vfs_create(...);
inode_unlock(workdir->d_inode);

// 正确：父目录使用 I_MUTEX_PARENT
inode_lock_nested(workdir->d_inode, I_MUTEX_PARENT);
vfs_create(...);
inode_unlock(workdir->d_inode);
```

重构合并多个站点的锁获取时，必须使用最严格的嵌套注释。如果某些调用者使用了 `I_MUTEX_PARENT`，合并后的代码应使用 `I_MUTEX_PARENT`。

### i_rwsem 的作用域

`inode->i_rwsem` 保护的内容远不止文件数据。它被 exclusive 持有于：
- 目录变更：`create`, `link`, `unlink`, `rmdir`, `rename`, `mkdir`, `mknod`, `symlink`
- 文件属性变更：`setattr`, `fileattr_set`
- buffered write 路径：`write_begin`/`write_end`, truncate

它被 shared 持有于：
- 目录的 `lookup` 操作
- 不带 `O_CREAT` 的 `atomic_open`

详见 `Documentation/filesystems/locking.rst`。

---

## Dentry 状态与实例化

dentry 可以是 negative（记录名称不存在）或 positive（已关联 inode）。`d_instantiate()` 将 negative dentry 转变为 positive。

### Invariants

- **Negative dentry**：`DCACHE_MISS_TYPE` 类型标志，`d_inode == NULL`。标准检查：`d_is_negative()` (`include/linux/dcache.h`)，检查类型标志。`d_really_is_negative()` 直接检查原始 `d_inode` 指针，仅应被 filesystem 用于检查自己的 dentry（对 overlay/union filesystem 有区别）。
- **Positive dentry**：非 `DCACHE_MISS_TYPE` 条目类型，`d_inode != NULL`。检查：`d_is_positive()` 或 `d_really_is_positive()`。

### d_instantiate() 的前置条件

1. dentry 不能已在任何 inode 的别名列表上 —— 由 `d_instantiate()` 中的 `BUG_ON(!hlist_unhashed(&entry->d_u.d_alias))` 强制
2. dentry 不能处于并行 lookup 中 —— 由 `__d_instantiate()` 中的 `WARN_ON(d_in_lookup(dentry))` 强制
3. 调用者必须已增加 inode 的引用计数

### 变体函数

- `d_add()`：组合了实例化和 hashing（通过 `__d_rehash()`）。`d_instantiate()` 不会 hashing dentry。
- `d_instantiate_new()`：用于设置了 `I_NEW` 标志的 inode（由 `iget_locked()`, `iget5_locked()`, `insert_inode_locked()` 设置）。它组合了 `d_instantiate()` 和 `unlock_new_inode()`（清除 `I_NEW | I_CREATING` 并唤醒等待者）。要求非 NULL inode（`BUG_ON(!inode)`），如果 `I_NEW` 未设置则 warn。不要对仅由 `new_inode()` 得到的 inode 使用它（`new_inode()` 不设 `I_NEW`）。

### Failure Modes

- 在已 positive 的 dentry 上调用 `d_instantiate()` -> `BUG_ON`，内核 crash
- 混淆 negative/positive dentry：对 negative dentry 访问 `d_inode` 导致 NULL 指针解引用；对已 positive dentry 重新实例化导致数据损坏

---

## 路径查找模式

路径解析在 `fs/namei.c` 中进行，有两种模式：

### RCU-walk（`LOOKUP_RCU`）

- 无锁路径遍历，避免所有锁和引用计数
- 使用 `rcu_read_lock()` 保持整个遍历过程
- 采样 `mount_lock`（seqlock）和 per-dentry `d_seq`（`seqcount_spinlock_t`）的序列号以检测并发修改
- 出现任何不一致时返回 `-ECHILD`，触发回退到 REF-walk
- 参考 `path_init()` 和 `lookup_fast()`

### REF-walk

- 在 dentry 上获取 `d_lockref` 引用，在 vfsmount 上获取 per-CPU `mnt_count` 引用
- dcache miss 时，`lookup_slow()` 在目录上 shared 持有 `i_rwsem` 执行 filesystem lookup
- 不会因并发而失败（绝不返回 `-ECHILD`），但会返回标准 filesystem 错误（`-ENOENT`, `-ENOTDIR`, `-EPERM`, `-ELOOP`, `-ESTALE`）

### Invariants

1. 内核总是先尝试 RCU-walk，失败时回退到 REF-walk。`filename_lookup()` 的模式：
   1. `path_lookupat()` 带 `LOOKUP_RCU`
   2. `-ECHILD` 时，不带 `LOOKUP_RCU` 重试（REF-walk）
   3. `-ESTALE` 时，带 `LOOKUP_REVAL` 重试（强制重新验证）
2. 中间从 RCU-walk 到 REF-walk 的转换由 `try_to_unlazy()` 处理：在当前路径上获取引用并验证 `d_seq`。如果验证失败，返回 `false`，调用者返回 `-ECHILD` 进行完全重启。
3. 转换是单向的：一旦进入 REF-walk，绝不切回 RCU-walk。

### Failure Modes

- 不正确的 RCU-walk 到 REF-walk 转换：如果在放弃 RCU 保护之前未获取引用，则 dentry 或 mount 结构可能发生 use-after-free
- 跳过 seqcount 验证：walk 可能跟踪到被并发重命名或移动的 dentry

---

## 权限检查

### 关键函数

- `may_open()` (`fs/namei.c`)：文件打开操作的关卡。检查文件类型限制（如不能写入目录、不能 exec 非普通文件）、设备访问（`may_open_dev()`），并委托给 `inode_permission()` 进行 POSIX 权限检查。该函数是 `static` 的，仅在 `namei.c` 内。
- `inode_permission()` (`fs/namei.c`)：导出的中心权限检查函数。链式调用：superblock 级别检查（只读 filesystem）-> 不可变文件检查 -> 未映射 ID 检查 -> `do_inode_permission()`（POSIX/filesystem 特定）-> `devcgroup_inode_permission()`（cgroup 设备控制器）-> `security_inode_permission()`（LSM hooks）。
- `file_permission()` (`include/linux/fs.h`)：从 `struct file` 提取 idmap 和 inode 的便捷封装，调用 `inode_permission()`。

### Failure Modes

- 缺少或乱序的权限检查：允许未经授权的文件访问，这是安全绕过
- 在文件已打开或修改后才检查权限：为时已晚

---

## 文件操作（file_operations）

### f_op 生命周期

- `file->f_op` 在 `init_file()` 中初始化为 `NULL`，但在文件可用前总是被设置为有效指针
- 对于 `O_PATH` 文件，设置为 `&empty_fops`（`do_dentry_open()` 本地的零初始化 `struct file_operations`），而非 `NULL`
- 对于所有其他文件，`do_dentry_open()` 通过 `fops_get(inode->i_fop)` 设置；如果返回 `NULL`，open 失败并返回 `-ENODEV`（带 `WARN_ON`）

### replace_fops()

文件创建后的 `f_op` 重新赋值需要使用 `replace_fops()` (`include/linux/fs.h`)，它会对旧操作调用 `fops_put()` 并存储新指针。调用者必须已持有新 fops 的模块引用（通过 `fops_get()`）。

### fops_get() / fops_put()

管理后备模块的引用计数（通过 `try_module_get()` / `module_put()`）。**原始指针赋值会绕过模块引用计数**，导致模块卸载时的 use-after-free。

典型例子：`chrdev_open()` (`fs/char_dev.c`) 将 `def_chr_fops` 替换为设备特定的操作。

### file->private_data

- 在 `init_file()` 中初始化为 `NULL`
- 存活于 `struct file` 的整个生命周期
- **内核不会自动释放其指向的内容** —— `.release()` 文件操作回调负责清理

### Failure Modes

- 使用陈旧或引用不当的 `f_op` 指针：后备模块卸载时 use-after-free，或操作结构体缺失时 NULL 解引用

---

## Dentry 在 TOCTOU 窗口后的有效性

当 dentry 引用在没有持有父 inode 锁的情况下获取（例如通过 lookup、creation 或缓存引用），之后锁才被获取时，存在 TOCTOU 窗口。在此窗口内 dentry 可能被并发操作失效。

### Invariants

完整的 dentry 验证需要**两个检查**：

1. **父指针稳定性**：`dentry->d_parent == expected_parent` —— 检测将 dentry 移动到不同目录的 rename 操作
2. **Dentry 在命名空间中存在**：`!d_unhashed(dentry)` —— 检测从 dcache 中移除 dentry 的 unlink/removal 操作

两个检查都是强制性的。dentry 可能在 `d_parent` 指针不变的情况下被 unhashed，因此只检查 `d_parent` 会遗漏移除竞态。

```c
// 不完整 —— 只检查 parent，遗漏 removal race
if (dentry->d_parent != expected_parent)
    return -ESTALE;

// 完整 —— 检查两个条件
if (dentry->d_parent != expected_parent || d_unhashed(dentry))
    return -ESTALE;
```

### Failure Modes

- 使用过时 dentry 导致 rename/unlink 操作静默失败
- 目录状态损坏
- 操作在错误的对象上执行

**REPORT as bugs**：仅检查 `d_parent` 而不检查 `d_unhashed()` 的 TOCTOU 验证代码。

---

## Quick Checks

- **`sb->s_umount` for superblock 操作**：`struct rw_semaphore`，保护 superblock 生命周期操作。exclusive 持有于 `put_super`, `freeze_fs`, `unfreeze_fs`, `remount_fs`；shared 持有于 `sync_fs`。
- **`dget`/`dput` for dentry 引用**：`dget()` 通过 `lockref_get()` 增加 dentry 引用计数。`dput()` 递减计数，计数归零时可能释放 dentry。在零引用计数的 dentry 上调用 `dget()` 是 bug。
- **锁释放与重新获取**：当 `i_rwsem` 或其他 VFS 锁被释放并重新获取时（例如为避免锁顺序违规或休眠），验证代码在重新获取后重新验证所有受保护状态 —— dentry 可能已变 negative，inode 可能已被驱逐，目录可能已被删除。
