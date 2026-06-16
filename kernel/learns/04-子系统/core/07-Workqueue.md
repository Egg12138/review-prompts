<!-- Source: subsystem/workqueue.md -->

# Workqueue 子系统核心知识

## Workqueue 生命周期管理

### Invariants

一旦通过 `queue_work()` 将 work 提交到 workqueue，**必须**确保在释放包含 work 结构的内存之前，work 已完成执行或已被取消。`flush_work()` 和其他 workqueue shutdown 方法提供了这种保证：

- `flush_work(struct work_struct *work)`：等待指定 work 完成执行。它是同步的，可以被阻塞。
- `cancel_work_sync(struct work_struct *work)`：尝试取消 work，如果 work 已在执行则等待其完成。适合在释放前清理。
- `destroy_workqueue(struct workqueue_struct *wq)`：销毁 workqueue，等待所有 queued work 完成。

### Failure Modes

没有使用 `flush_work()` 或其他 workqueue shutdown 机制就释放包含 work_struct 的内存，会导致 **work struct 泄漏**：

```
CPU 0                              CPU 1 (worker)
─────                              ─────────────
释放包含 work_struct 的对象
                                   work 函数开始执行
                                   → 访问已释放的内存
                                   → USE-AFTER-FREE
```

当 work 被其他模块或子系统引用时，这种行为更加微妙。即使你是 work 结构的所有者，如果 work 已被 `queue_work()` 提交，worker 线程可能在任何 CPU 上随时执行你的回调。

---

## Quick Checks

- 释放包含 `work_struct` 的对象前，始终调用 `flush_work()` 或 `cancel_work_sync()`
- 从 timer/hrtimer 回调中调用 `flush_work()` 会导致问题（`flush_work()` 可能睡眠，timer 回调在 atomic 上下文中）
- 持有锁时调用 `flush_work()`，检查 work 回调是否也获取同一锁 —— 这会导致死锁
- `cancel_work_sync()` 只能从可睡眠上下文调用；它使用 wait 进行同步
