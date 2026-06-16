<!-- Source: mm-vma.md -->

# MM VMA 操作

## SLAB_TYPESAFE_BY_RCU 与 VMA 回收

从 `SLAB_TYPESAFE_BY_RCU` 对象中解引用 parent/owner 指针后放弃对象的 refcount，会导致在对象被回收给不同 owner 时发生 use-after-free。Owner 可以在 refcount 释放和解引用之间的窗口内退出并释放其后端结构。

VMA 缓存使用 `SLAB_TYPESAFE_BY_RCU` 创建，这意味着即使在 `vm_area_free()` 之后，RCU 读端临界区内的 VMA slab 内存仍然有效，但 VMA 在此期间可能被重新分配给完全不同的 `mm_struct`。

**Per-VMA 锁查找协议**（参见 `lock_vma_under_rcu()` 和 `vma_start_read()` 在 `mm/mmap_lock.c`）：
1. 在 `rcu_read_lock()` 下调用 `mas_walk()` 在 maple tree 中找到 VMA
2. `vma_start_read()` 递增 `vma->vm_refcnt`
3. 如果 `vma->vm_mm != mm`（VMA 被回收了），refcount 必须被释放——但 `vma_refcount_put()` 会为了 `rcuwait_wake_up()` 而解引用 `vma->vm_mm`
4. 必须先用 `mmgrab()` 稳定 foreign `mm` 再调用 `vma_refcount_put()`，然后用 `mmdrop()` 释放

**应报告为 bug**：在 `lock_vma_under_rcu()`、`lock_next_vma()` 或 `vma_start_read()` 中，对 `vm_mm` 与调用者 `mm` 不匹配的 VMA 调用 `vma_refcount_put()`，而未先用 `mmgrab()` 稳定 foreign `mm`。

## VMA 匿名 vs 文件后端分类

使用 `vma->vm_file` 判断 VMA 是否为文件后端会导致对具有 `vm_file` 但被当作匿名处理的 VMA 的错误分发。这会导致 BUG_ON 崩溃、页面偏移不对齐或走错代码路径。

**VMA 分类的工作原理**（参见 `include/linux/mm.h`）：
- `vma_is_anonymous(vma)` 返回 `!vma->vm_ops`——这是匿名 VMA 的规范测试
- `vma_set_anonymous(vma)` 设置 `vma->vm_ops = NULL` 但不清除 `vma->vm_file`
- VMA 可以有 `vma->vm_file != NULL` **且** 是匿名的（`vm_ops == NULL`）

**`vm_file` 被设置但 VMA 是匿名的情况：**
- `/dev/zero` 的私有映射：`mmap_zero_private_success()` 对私有映射调用 `vma_set_anonymous(vma)`，使 `vm_file` 仍指向 `/dev/zero` 文件
- 任何在 VMA 创建后调用 `vma_set_anonymous()` 的驱动 mmap 处理程序

**正确用法：**
- 判断 "VMA 是否是文件后端？"：使用 `!vma_is_anonymous(vma)`，**而非** `vma->vm_file != NULL`
- 判断 "VMA 是否匿名？"：使用 `vma_is_anonymous(vma)`，**而非** `vma->vm_file == NULL`
- 访问文件后端 VMA 的后端文件：先检查 `!vma_is_anonymous(vma)`，然后使用 `vma->vm_file`

**应报告为 bug**：在分发逻辑、条件判断或断言中，使用 `vma->vm_file`（或 `!vma->vm_file`）作为文件后端（或匿名）VMA 分类代理的代码。

## VMA 拆分/合并临界区

在 `vma_prepare()` / `vma_complete()` 临界区外执行的页表结构性修改会与并发的缺页（通过 VMA 锁）和 rmap 遍历（通过文件/匿名 rmap 锁）竞态。导致 use-after-free、页表损坏或状态重建。

VMA 修改路径——`__split_vma()`、`commit_merge()` 和 `vma_shrink()`——共享一个临界区：

1. `vma_start_write()`（获取 per-VMA 锁，在入口处或之前）
2. `vma_prepare()`（获取文件 rmap `i_mmap_lock_write` 和 anon_vma 锁）
3. 页表结构性修改：`vma_adjust_trans_huge()`、`hugetlb_split()`
4. VMA 范围更新（`vm_start` / `vm_end` / `vm_pgoff`）
5. `vma_complete()`（释放步骤 2 中获取的锁）

`__split_vma()` 在此序列前额外调用 `vm_ops->may_split()`。

**规则：**
- `vm_ops->may_split()` 只能验证拆分是否允许，不能修改页表或其他共享状态，因为它在 VMA 和 rmap 锁获取之前运行
- VMA 拆分所需的任何页表取消共享、拆分或拆卸必须在 `vma_prepare()` 和 `vma_complete()` 之间进行
- 调用通常自己获取锁的辅助函数时，使用 `take_locks=false` 路径并断言所需锁已持有

## 通过 vma_start_write() 实现 Per-VMA 锁定排除

`mmap_write_lock()` **单独** 不排除 per-VMA 锁持有者——在 `mmap_write_lock()` **之前** 获取的 per-VMA 读锁仍然保持，因为 `vma_start_read()` 中的 seqcount 只阻止 **新的** 获取，不撤销已有的。只有 `vma_start_write(vma)` 排干现有的 per-VMA 读锁持有者。VMA 锁定的操作修改所有级别的页表，因此任何在 `vma_start_write()` 之前访问页表的 `mmap_write_lock` 持有者都会与 per-VMA 锁定的路径竞态。

**`check_pmd_still_valid()` / `find_pmd_or_thp_or_none()`**：这些函数遍历页表然后读取 PMD 值。并发的 per-VMA 锁定 `MADV_DONTNEED` 可以在 PMD 读取和随后使用结果之间调用 `try_to_free_pte()` → `pmd_clear()` + `free_pte()`——检查成功，调用者假设 PMD 有效，但 PMD 已被清除且 PTE 页已在下面被释放。

**应报告为 bug**：持有 `mmap_write_lock` 但在调用 `vma_start_write(vma)` 之前访问页表或 PTE 页的函数。

## VMA 标志修改 API

关键区别：`vm_flags_set()` 做 OR（增位，从不清除），`vm_flags_reset()` 替换（设为精确值），`vm_flags_init()` 无锁替换（VMA 尚未在树中）。`vm_flags_clear()` 移除指定位。`vm_flags_mod()` 在一次操作中增删。

**常见错误**：用 `vm_flags_set(vma, new_flags)` 替换标志——因为它是 OR 操作，旧标志会静默保留。使用 `vm_flags_reset()` 进行精确替换。残留的 `VM_WRITE` / `VM_MAYWRITE` 会制造安全漏洞。

## mmap 回调期间的文件引用所有权

mmap 使用拆分所有权：`ksys_mmap_pgoff()` 持有一个文件引用（结束时 fput），VMA 通过 `__mmap_new_file_vma()` 中的 `get_file()` 获取自己的引用。当回调替换文件时，替换的文件已经携带了自己的引用。

**应报告为 bug**：对可能已被回调交换的文件进行无条件的 `get_file()`——替换者会得到一个泄漏的额外引用。

## 快速检查清单

- **mmap_lock 顺序**：使用错误的锁类型会导致死锁或损坏 VMA 树。写锁用于 VMA 结构性修改（插入/删除/拆分/合并、修改 vm_flags/vm_page_prot）。读锁用于 VMA 查找、缺页处理和只读遍历
- **可失败的 mmap lock 重新获取**：`mmap_write_lock_killable()` / `mmap_read_lock_killable()` 在收到 kill 信号时返回 `-EINTR`。忽略返回值意味着在无锁情况下继续操作
- **VMA 合并 anon_vma 传播**：合并未触发的 VMA 与已触发的 VMA 需要 `dup_anon_vma()`。合并时 anon_vma 属性的检查必须应用于 **有** anon_vma 的 VMA，而非无条件应用于目标
- **VMA 区间树使用 pgoff，而非 PFN**：`mapping->i_mmap` 以 `vm_pgoff` 为键；`vma_address()` 期望 `pgoff_t`。传递裸 PFN 会在错误的坐标空间中搜索。**应报告为 bug**：将裸 PFN 传给 `vma_interval_tree_foreach()` 或 `vma_address()`
- **VMA 合并/修改错误处理**：`vma_modify()` / `vma_merge_new_range()` 可能返回错误或不同的 VMA。原始 VMA 可能在成功时被释放。失败时 `vmg->start/end/pgoff` 可能被改变
- **VMA 标志顺序 vs 合并**：不在 `VM_IGNORE_MERGE` 中的标志必须在 `vma_merge_new_range()` **之前** 设置在提议的 `vm_flags` 中。合并后通过 `vm_flags_set()` 设置标志会静默破坏未来的合并
- **VMA 合并副作用 vs 页表操作**：`vma_complete()` 触发 `uprobe_mmap()` 并安装 PTE。随后移动/覆盖页表的调用者必须在 `struct vma_merge_struct` 中设置 `skip_vma_uprobe`，否则孤立的 PTE 会泄漏内存
- **Fork 时 VMA 标志分歧**：`dup_mmap()` 清除子 VMA 上的 `__VM_UFFD_FLAGS` 和 `VM_LOCKED_MASK`。Fork 时的标志检查必须使用目标 VMA，而非源 VMA
- **VMA 操作期间保留 `VM_ACCOUNT`**：在幸存的 VMA 上清除 `VM_ACCOUNT` 会永久泄漏已承诺的内存——`do_vmi_munmap()` 只对具有 `VM_ACCOUNT` 的 VMA 取消记账
- **外部 mm_struct 上的 VMA 迭代**：在 mmap lock 之后、遍历之前调用 `check_stable_address_space(mm)`。`dup_mmap()` 失败时，maple tree 槽包含 `XA_ZERO_ENTRY` 标记
- **VMA 操作结果赋值给结构成员**：`vma_merge_extend()`、`vma_merge_new_range()`、`copy_vma()` 在失败时返回 NULL。直接赋值给结构成员会在 NULL 检查之前覆盖原始 VMA 指针。先赋值给本地变量，检查 NULL，成功后更新结构成员
- **VMA 合并函数在成功时使输入无效**：`vma_merge_new_range()`、`vma_merge_existing_range()`、`vma_modify()` 可能在成功时释放原始 VMA。调用者必须使用返回的 VMA，而非原始的。丢弃返回值并使用原始的是 use-after-free
- **VMA 迭代循环中的 `vma_modify*()` 错误返回**：`vma_modify_flags()` 等返回 `ERR_PTR(-ENOMEM)`。赋值回 VMA 循环变量而不检查 `IS_ERR()` 会解引用错误指针
- **错误路径上的 VMA 锁 refcount 平衡**：`__vma_enter_locked()` 将 `VMA_LOCK_OFFSET` 加到 `vm_refcnt` 然后等待读者。使用 `TASK_KILLABLE` / `TASK_INTERRUPTIBLE` 时，`-EINTR` 路径必须减去偏移量。泄漏的偏移量永久阻止 VMA 分离/释放
- **VMA 地址用作布尔标志**：`vm_start` 可以合法为零，因此 `if (addr_var)` 表示"此值已设置"对零地址 VMA 静默失败。使用显式的 `bool` 标志
- **Maple state RCU 生命周期**：`ma_state` 缓存 RCU 保护的节点指针。在 `rcu_read_unlock()` 后，用 `mas_set()` 或 `mas_reset()` 失效后再重用
- **`mm_struct` 灵活数组大小**：尾部灵活数组打包 cpumask 和 mm_cid 区域。静态定义必须使用 `MM_STRUCT_FLEXIBLE_ARRAY_INIT`
- **Memfd 文件创建 API 层次**：直接调用 `shmem_file_setup()` 或 `hugetlb_file_setup()` 用于 memfd 会产生缺少 `O_LARGEFILE`、fmode 标志和安全初始化的文件。使用 `memfd_alloc_file()`
- **VMA 锁 vs mmap_lock 断言**：仅持有 VMA 锁时 `mmap_assert_locked(mm)` 会触发。在 per-VMA 锁下可达的路径必须使用 `vma_assert_locked(vma)`
- **VM 已承诺内存记账**：`security_vm_enough_memory_mm()` 不仅是检查——成功时通过 `vm_acct_memory()` 递增 `vm_committed_as`。成功后的每个错误路径必须调用 `vm_unacct_memory()`。泄漏的计费会永久膨胀 `vm_committed_as`
