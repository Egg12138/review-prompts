<!-- Source: subsystem/rcu.md -->

# RCU 子系统核心知识

## RCU 读侧临界区

- `rcu_read_lock()` / `rcu_read_unlock()` 界定读侧临界区
- 经典 RCU 读段内不能阻塞或睡眠
- 嵌套 `rcu_read_lock()` 调用是安全的
- `CONFIG_PREEMPT_RCU` 下允许读段被抢占

---

## 发布（Publishing）和读取（Reading）

- `rcu_assign_pointer(p, v)`：以 release 语义（`smp_store_release`）发布指针，确保先前的初始化对读者可见
- `rcu_dereference(p)`：以依赖顺序（`READ_ONCE`）加载指针，确保通过指针的后续访问看到已发布的数据
- 这两个函数形成一对：`rcu_assign_pointer()` 中的 release 与 `rcu_dereference()` 中的依赖顺序配对

---

## Grace Period 和资源回收

### Invariants

- `synchronize_rcu()`：阻塞直到所有已存在的读侧临界区完成
- `call_rcu(&head, callback)`：延迟回调直到 grace period 之后（非阻塞）
- `kfree_rcu(ptr, rhf)`：`call_rcu()` 的简写，在回调中调用 `kfree()`；`rhf` 是 `rcu_head` 字段名
- `kfree_rcu_mightsleep(ptr)`：无头变体，不需要 `rcu_head` 字段，但必须在可睡眠上下文中调用

### Failure Modes

**RCU-001：移除先于回收顺序**
对象必须在调用 `call_rcu()` 或 `synchronize_rcu()` **之前**从 RCU 保护的数据结构中移除。这是因为 `call_rcu()` 只等待调用时已存在的读者 —— 它不保护在 grace period 之后开始的读者。如果对象仍链接在数据结构中，新读者可以在释放后找到并访问它。

正确的顺序：
1. **从数据结构中移除** —— 阻止新读者找到对象
2. **`call_rcu()` 或 `synchronize_rcu()`** —— 等待现有读者完成
3. **释放资源** —— 在回调中或 `synchronize_rcu()` 返回后

使用适当的 RCU 感知移除辅助函数：`hlist_del_rcu()`, `list_del_rcu()`, `rhashtable_remove_fast()` 等。

```c
// 错误 —— 在 call_rcu 之后移除，导致 use-after-free
call_rcu(&obj->rcu, free_callback);

void free_callback(struct rcu_head *rhp) {
    struct obj *obj = container_of(rhp, struct obj, rcu);
    hlist_del_rcu(&obj->node);  // 太晚：新读者已经找到它
    kfree(obj);
}
```

```c
// 正确 —— 先移除，再延迟释放
hlist_del_rcu(&obj->node);         // 没有新读者能找到它
call_rcu(&obj->rcu, free_callback);

void free_callback(struct rcu_head *rhp) {
    struct obj *obj = container_of(rhp, struct obj, rcu);
    kfree(obj);                    // 安全：所有先前的读者都已完成
}
```

**REPORT as bugs**：在对象仍可通过 RCU 保护的数据结构访问时调用 `call_rcu()` 或 `kfree_rcu()` 的代码，或者在 RCU 回调（而非在之前）中执行移除的代码。

---

## kvfree_call_rcu() / kfree_rcu() 调用上下文

`kvfree_call_rcu()`（通过 `kfree_rcu`/`kvfree_rcu` 宏调用）在 `raw_spinlock_t` 下调用（`kernel/sched/core.c` 中的 `pi_lock`），可能从 hardirq 上下文。在 `kvfree_call_rcu()` 或其被调用者中添加 `spinlock_t`, `local_lock` 或 `local_trylock` 的获取会导致 lockdep "Invalid wait context" 警告 —— `!IS_ENABLED(CONFIG_PREEMPT_RT)` 守卫不能阻止此警告，因为 `CONFIG_PROVE_RAW_LOCK_NESTING`（默认 `y`）检查声明的等待类型，而非运行时行为。

不要因为相同锁类型存在于其他地方就忽略此问题。

---

## RCU 变体对比

| 变体 | 读侧 API | 可睡眠 |
|------|---------|--------|
| RCU | `rcu_read_lock()` / `rcu_read_unlock()` | 否 |
| SRCU | `srcu_read_lock()` / `srcu_read_unlock()` | 是 |
| Tasks RCU | （隐式） | 不适用 |
| Tasks Trace RCU | `rcu_read_lock_trace()` / `rcu_read_unlock_trace()` | 是 |

Tasks RCU 没有显式的读侧锁 —— 任何不自愿上下文切换的代码都隐式处于读侧临界区。

---

## Quick Checks

- `rcu_read_lock_held()` 用于 lockdep 调试断言
- `INIT_RCU_HEAD` 已从内核中完全移除
- `rcu_barrier()` 等待所有待处理的 `call_rcu()` 回调完成（模块卸载时需要）
