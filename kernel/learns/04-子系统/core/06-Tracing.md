<!-- Source: subsystem/tracing.md -->

# Tracing 子系统核心知识

## Trace Event 定义与字符串处理

`TRACE_EVENT` 宏的错误使用会导致 ring buffer 中数据损坏、字符串截断或缓冲区溢出的内核 panic。在 `TP_fast_assign()` 中使用有副作用的操作会导致行为差异取决于 tracing 是否启用，从而产生难以诊断的 bug。

### TP_STRUCT__entry() 中的宏

| 宏 | 用途 |
|-------|------|
| `__field(type, name)` | 固定大小的标量字段 |
| `__array(type, name, len)` | 嵌入在 trace record 中的固定大小数组 |
| `__string(name, src)` | 动态长度字符串；源指针自动捕获 |
| `__vstring(name, fmt, ap)` | 来自 `va_list` 格式参数的动态长度字符串 |

### TP_fast_assign() 中的赋值

- `__assign_str(name)`：复制在 `__string(name, src)` 中声明的字符串。它**只取字段名**；源通过 `__data_offsets` 从 `__string()` 声明自动捕获（参见 `include/trace/stages/stage5_get_offsets.h` 和 `include/trace/stages/stage6_event_callback.h`）
- `__assign_vstr(name, fmt, va)`：从 `va_list` 格式化为用 `__vstring()` 声明的字段

### Invariants

**`TP_fast_assign()` 绝不能有副作用。** 它仅在 tracepoint 活跃时执行，因此副作用会创建行为差异取决于 tracing 是否启用。

### TRACE_EVENT_CONDITION

`TRACE_EVENT_CONDITION` 和 `TP_CONDITION` 在条件为 false 时完全跳过 trace record，避免 `TP_fast_assign()` 和 ring buffer 分配的成本。当 trace 参数需要昂贵的解引用且应在条件为 false 时避免时使用 `TRACE_EVENT_CONDITION`。定义在 `include/linux/tracepoint.h` 中。

---

## Tracepoint Probe 注册与 RCU

Tracepoint 回调在 RCU 读侧临界区中执行：非 faultable tracepoint 使用 `preempt_disable_notrace()`（通过 `guard(preempt_notrace)`），faultable syscall tracepoint 使用 RCU Tasks Trace（通过 `guard(rcu_tasks_trace)`）。

### Invariants

- 通过 `tracepoint_probe_register()` 或 `rv_attach_trace_probe()` 注册的 probe 必须保持有效直到 `tracepoint_synchronize_unregister()` 之后被调用。此函数发出 `synchronize_rcu_tasks_trace()` 和 `synchronize_rcu()` 以确保所有正在执行的 probe 调用完成。
- Tracepoint 由 tracepoint key 上的 `static_branch_unlikely()` 门控（初始化为 `STATIC_KEY_FALSE_INIT` 的 static key）。当没有 probe 注册时，分支是 NOP，开销接近零。
- 在非 faultable tracepoint 上下文中，probe 函数不能睡眠。Faultable tracepoint（通过 `DECLARE_TRACE_SYSCALL` / `TRACE_EVENT_SYSCALL` 声明）调用 `might_fault()` 并使用 RCU Tasks Trace 保护，允许可睡眠操作。

### Failure Modes

访问已释放的 probe 数据或调用已注销的 probe 函数导致 use-after-free 或 NULL 指针解引用。

---

## Tracepoint Kconfig 依赖

使用有条件编译的 tracepoint 会在隐藏 tracepoint 目标文件的 Kconfig 选项禁用时导致链接失败。架构依赖（如 `depends on X86 || RISCV`）不能保证子系统特定的 tracepoint 可用，因为子系统功能可以独立禁用。

### 示例：page_fault tracepoint

`page_fault_kernel` 和 `page_fault_user` tracepoint 声明在 `include/trace/events/exceptions.h`，但它们的 `CREATE_TRACE_POINTS` 实例化位于架构故障处理程序（`arch/x86/mm/fault.c`, `arch/riscv/mm/fault.c`）中。在 RISC-V 上，`fault.o` 仅在启用 `CONFIG_MMU` 时构建。在 x86 上，`CONFIG_MMU` 无条件为 `y`。

代码通过 `rv_attach_trace_probe()` 或直接 `tracepoint_probe_register()` 调用附加到 tracepoint 时，其 Kconfig 必须包括与 tracepoint 构建时可用性匹配的依赖。检查包含 tracepoint 头文件的 `CREATE_TRACE_POINTS` 文件的 Makefile 和 Kconfig 守卫。

```kconfig
// 错误：X86 || RISCV 可能有 !MMU 配置（RISC-V NOMMU）
config RV_MON_PAGEFAULT
    depends on X86 || RISCV

// 正确：为 page fault tracepoint 显式要求 MMU
config RV_MON_PAGEFAULT
    depends on X86 || RISCV
    depends on MMU
```

实际例子见 `kernel/trace/rv/monitors/pagefault/Kconfig`。

---

## Quick Checks

- 非 faultable trace 上下文中无阻塞操作
- `__assign_str()` 只取字段名（单个参数）；源隐式来自 `__string()`
- Tracepoint 名遵循 `subsystem_event` 惯例
- 每个 tracepoint 头文件恰好有一个 `.c` 文件在包含它之前定义 `CREATE_TRACE_POINTS`
- Kconfig 依赖与被消费的任何 tracepoint 的构建时可用性匹配
