<!-- Source: subsystem/bpf.md -->

# BPF 子系统详解

## 验证器不变式（Verifier Invariants）

- 所有内存访问必须经过验证器的边界检查
- 寄存器类型通过程序流程跟踪
- 栈槽（stack slots）必须在使用前初始化
- 辅助函数有特定的参数要求

## BPF Map 操作

- Map 查找可能返回 NULL
- Map 更新需要检查 `max_entries`
- Map 中的自旋锁需要 `bpf_spin_lock()` / `bpf_spin_unlock()`
- 每-CPU map 需要 `bpf_get_cpu_ptr()` / `bpf_put_cpu_ptr()`

## 引用追踪（Reference Tracking）

- 某些辅助函数返回"获取的"引用（acquired references）
- 必须使用对应的释放辅助函数释放
- 验证器跟踪每个寄存器的引用状态

## 上下文访问（Context Access）

- 上下文指针是只读的
- 字段访问必须在 ctx 结构体大小内
- 某些字段需要特定的程序类型

## BPF 内核函数（kfuncs）

关键字包括 `__bpf_kfunc`、`BTF_KFUNCS*`、`KF_*` 标志等。详细参见 `Documentation/bpf/kfuncs.rst`。

### 验证器对 Kfunc 参数的验证

BPF 验证器在程序加载时对 kfunc 参数执行验证。理解验证器实际保证的内容对于正确识别真实 bug 至关重要。

#### 标量和枚举参数

当 kfunc 接受标量参数（int、u32、enum 等）时，验证器：

1. **检查寄存器类型**：确保寄存器是 `SCALAR_VALUE`（而不是指针）
2. **跟踪值范围**：内部维护标量值的 min/max 边界
3. **对于 `__const` 注解的参数**：要求值在验证时已知

**关键点**：验证器不强制枚举值是枚举的有效成员。枚举类型与普通整数等同处理——验证器只检查它是标量，不检查它是否在有效的枚举范围内。

来自 `kernel/bpf/verifier.c` 中 `check_kfunc_args()`：

```c
if (btf_type_is_scalar(t)) {
    if (reg->type != SCALAR_VALUE) {
        verbose(env, "R%d is not a scalar\n", regno);
        return -EINVAL;
    }
    // ... 处理 __const 和特殊情况 ...
    continue;  // 没有枚举范围验证！
}
```

#### Kfunc 必须包含数组索引的边界检查

由于验证器不验证枚举范围，使用枚举或整数参数作为数组索引的 kfunc 必须包含运行时的边界检查。

**正确边界检查的示例**：

```c
__bpf_kfunc unsigned long bpf_mem_cgroup_page_state(struct mem_cgroup *memcg, int idx)
{
    if (idx < 0 || idx >= MEMCG_NR_STAT)  // 必需的！
        return (unsigned long)-1;
    return memcg_page_state_output(memcg, idx);
}
```

**枚举参数**必须检查负值，因为在 C 语言中枚举是有符号的 int：

```c
__bpf_kfunc unsigned long bpf_mem_cgroup_memory_events(struct mem_cgroup *memcg,
                        enum memcg_memory_event event)
{
    // 必须检查两个边界——枚举是有符号 int，负值会绕过 >= 检查
    if ((unsigned int)event >= MEMCG_NR_MEMORY_EVENTS)
        return (unsigned long)-1;
    // 或：if (event < 0 || event >= MEMCG_NR_MEMORY_EVENTS)
    
    return atomic_long_read(&memcg->memory_events[event]);
}
```

**为什么负值检查重要**：如果 `event = -1`：
- 有符号比较：`-1 >= 10` 为 FALSE（检查错误通过）
- 函数访问 `memory_events[-1]`——越界读取

#### 何时值保证安全

唯一真正不需要边界检查的情况：

1. **`__const` 注解的参数**：验证器要求 `tnum_is_const()`，意味着只能传递来自 vmlinux.h 的编译时常量。

没有 `__const` 的情况下，BPF 程序可以传递来自以下来源的值：
- Map 查找（用户控制的）
- 运行时计算
- 任意标量寄存器

#### Kfunc 标量/枚举参数安全性总结

| 场景 | 需要边界检查？ | 原因 |
|------|--------------|------|
| `__const` 注解的枚举/int | 否 | 验证器强制常量值 |
| 枚举参数（无 `__const`） | 是 | 验证器不检查枚举范围 |
| 普通 int/u32 参数 | 是 | 对值没有约束 |
| 任何用于数组索引的参数 | 是 | 必须防止越界访问 |

**应报告为 bug 的情况**：使用枚举或整数参数作为数组索引的 kfunc，没有适当的边界检查（包括对有符号类型的负值检查）。

## BPF 注释风格

BPF 子系统遵循现代内核多行注释风格。多行注释必须将开头的 `/*` 放在单独一行，注释文本从下一行开始：

```c
/* 错误——文本与开头在同一行 */
/* 这不是首选的
 * 内核注释风格。
 */

/*
 * 正确——这是首选的
 * 内核注释风格。
 */
```

单行注释在适合一行时保持原样即可：`/* This is fine. */`

这适用于 `kernel/bpf/`、`net/core/filter.c`、`include/linux/bpf*.h`、`tools/lib/bpf/`、`tools/testing/selftests/bpf/` 及其他与 BPF 相关的路径，即使同一文件中的周围代码使用旧风格。

## BPF Skeleton API（Selftests）

### 生成的 Skeleton 函数

BPF skeleton 由 `bpftool gen skeleton` 生成，提供类型安全的包装。每个 skeleton 包含以下函数（其中 `example` 是对象名称）：

- `example__open()` —— 打开 BPF 对象（不加载程序）
- `example__load()` —— 创建 map、加载并验证所有 BPF 程序
- `example__open_and_load()` —— 将 open + load 合并为一次操作
- `example__destroy()` —— 分离、卸载程序、释放资源

### Skeleton 的保证（在成功 `__open_and_load()` 后）

**重要**：在成功 `skel = example__open_and_load()` 后：
- skeleton 指针有效（不是 NULL/ERR_PTR）
- **所有程序已加载，具有有效的 FD**（>= 0）
- **所有 map 已创建，具有有效的 FD**（>= 0）
- skeleton 字段如 `skel->progs.prog_name` 和 `skel->maps.map_name` 保证有效

这意味着：
- `bpf_program__fd(skel->progs.prog_name)` **在成功加载后不可能返回负值**
- `bpf_map__fd(skel->maps.map_name)` **在成功加载后不可能返回负值**
- **使用 skeleton 生成的字段时无需额外的 FD 验证**

### 何时 FD 检查是必需的

使用以下方式时需要 FD 检查（`CHECK_FAIL(fd < 0)` 或类似）：
- 手动查找 API：`bpf_object__find_program_by_name()` —— 如果名称未找到可返回 NULL
- 手动查找 API：`bpf_object__find_map_by_name()` —— 如果名称未找到可返回 NULL
- 旧式加载：`bpf_prog_test_load()` —— 不同的 API 约定

### Skeleton 与手动查找模式对比

```c
// Skeleton 模式——成功 __open_and_load() 后无需 FD 检查
skel = example__open_and_load();
if (!ASSERT_OK_PTR(skel, "open_and_load"))
    return;
prog_fd = bpf_program__fd(skel->progs.my_prog);  // 此处不可能失败
map_fd = bpf_map__fd(skel->maps.my_map);          // 此处不可能失败

// 手动查找模式——需要 FD 检查（使用现代 ASSERT_* 宏）
obj = bpf_object__open_file("example.o", NULL);
prog = bpf_object__find_program_by_name(obj, "my_prog");  // 可返回 NULL
if (!ASSERT_OK_PTR(prog, "find_program"))
    goto cleanup;
prog_fd = bpf_program__fd(prog);  // 如果 prog 无效可返回负值
if (!ASSERT_GE(prog_fd, 0, "bpf_program__fd"))
    goto cleanup;
```

## BPF Selftest 断言宏

### 现代 ASSERT_*() 宏（首选）

所有新测试和更新已有测试时使用 `ASSERT_*()` 宏。

现代 ASSERT 系列包含类型特定的宏，如 `ASSERT_OK()`、`ASSERT_ERR()`、`ASSERT_EQ()`、`ASSERT_OK_PTR()`、`ASSERT_OK_FD()` 等。完整列表参见 `tools/testing/selftests/bpf/test_progs.h`。

### 已弃用的 CHECK() 宏（新代码中避免）

**新测试或补丁中不要使用：**

- `CHECK(condition, tag, format...)` —— **已弃用** —— 改用 `ASSERT_*()`
- `CHECK_FAIL(condition)` —— **已弃用** —— 改用 `ASSERT_*()`
- `CHECK_ATTR(condition, tag, format...)` —— **已弃用** —— 改用 `ASSERT_*()`

**为什么 `ASSERT_*()` 更优：**
1. 使用静态持续期变量而非全局 `duration` 变量
2. 对不同检查类型有更具体和类型安全的宏
3. 更好的错误消息，显示实际值与期望值
4. 自 2020 年以来的现代 BPF selftest 标准

**迁移示例：**

```c
// 旧（已弃用）：
static int duration = 0;  // 需要的全局/静态变量
if (CHECK(fd < 0, "open_fd", "failed to open: %d\n", errno))
    return;

// 新（首选）：
if (!ASSERT_OK_FD(fd, "open_fd"))  // 不需要 duration 变量
    return;
```

## BPF 模式参考

- **BPF-001**（使用 `copy_map_value*` 的 map 操作） → `btf.md`
- **LIBBPF-001**（`tools/lib/bpf*`） → `libbpf.md`

## Quick Checks（快速检查清单）

- [ ] 标记有 `BPF_RET_PTR_TO_MAP_VALUE_OR_NULL` 的辅助函数需要 NULL 检查
- [ ] `ARG_PTR_TO_MEM` 参数需要大小验证
- [ ] 尾调用（Tail calls）限制为 33 层
- [ ] 栈使用限制为 512 字节
- [ ] Kfunc 中枚举/整数作为数组索引时是否包含边界检查（包括负值检查）？
- [ ] 使用 skeleton 后是否进行了不必要的 FD 检查？
- [ ] 新代码是否使用了 `ASSERT_*()` 而非已弃用的 `CHECK()`？
