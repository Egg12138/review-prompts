<!-- Source: technical-patterns.md + subsystem/rcu.md -->

# RCU 顺序保证与生命周期

## 概述

RCU（Read-Copy-Update，读-复制-更新）是 Linux 内核中一种无锁同步机制，允许多个读者与一个写者同时操作，读者几乎不受开销。

**RCU 的核心思想**：
- 读者不需要拿锁——它们在一个被称为 RCU 读侧临界区（read-side critical section）的区域内安全地读取共享数据
- 写者创建一个新副本，用原子操作替换指针，然后等待所有已有读者完成后再释放旧副本

理解 RCU 的关键在于 **顺序保证（ordering）**——特别是 `call_rcu()` 的正确使用顺序。错误的顺序直接导致 use-after-free，这是内核中最严重的 bug 之一。

---

## 一、RCU 读侧临界区

### 基本 API

```c
rcu_read_lock();
// 在此区域内读取 RCU 保护的数据
p = rcu_dereference(global_ptr);
if (p) {
    do_something(p->data);
}
rcu_read_unlock();
```

- `rcu_read_lock()` / `rcu_read_unlock()` 标记一个 RCU 读侧临界区
- RCU 临界区内 **不允许阻塞或睡眠**（使用 SRCU 可以睡眠）
- 嵌套调用 `rcu_read_lock()` 是安全的
- 在 `CONFIG_PREEMPT_RCU` 配置下，读侧临界区内的任务可以被抢占（但不会被调度走）

### 隐式 RCU 读侧保护

某些情况下，你不需要显式调用 `rcu_read_lock()`：
- 持有 `spin_lock()` 或 `raw_spin_lock()` 时，隐式提供了 RCU 读侧保护（因为禁用了抢占，在非 RT 内核上；在 PREEMPT_RT 上，`spin_lock()` 内部会调用 `rcu_read_lock()`）

---

## 二、发布与读取（Publishing and Reading）

### RCU 的核心配对

```c
// 写者——发布新指针
struct foo *new = kmalloc(sizeof(*new), GFP_KERNEL);
new->data = 42;                          // 先初始化数据
rcu_assign_pointer(global_ptr, new);     // 然后发布指针（release semantics）

// 读者——读取并解引用
rcu_read_lock();
p = rcu_dereference(global_ptr);         // acquire semantics
if (p)
    x = p->data;                         // 保证看到 42
rcu_read_unlock();
```

### 为什么需要配对

- `rcu_assign_pointer(p, v)`：以 **release 语义** 发布指针。保证所有在它之前的写操作（如初始化 `new->data`）在指针发布前对其他 CPU 可见。
- `rcu_dereference(p)`：以 **dependency ordering** 加载指针。保证在它之后通过指针的读取能看到发布之前写入的数据。

**不配对会发生什么**：

```
CPU 0 (写者)                    CPU 1 (读者)
─────────────────               ─────────────────
data->field = 42;
WRITE_ONCE(global_ptr, data);   ← store 可能被重排到 field 写入之前！
                                p = READ_ONCE(global_ptr);
                                x = p->field; → 可能看到旧值（0）
```

这就是为什么裸的 `WRITE_ONCE()`/`READ_ONCE()` 不够——你需要 release/acquire 语义。

---

## 三、Grace Period 与回收

### Grace Period 是什么

一个 grace period（宽限期）是指 **所有在某个时刻已存在的 RCU 读侧临界区都已完成** 的那段时间。

```
Reader 1:  [========rcu_read_lock()=======rcu_read_unlock()========]
Reader 2:               [===rcu_read_lock()===rcu_read_unlock()=====]
                ^                                    ^
                |                                    |
          call_rcu() 被调用                  所有已有读者完成
                |                                    |
           [=============GRACE PERIOD==================]
                                                     |
                                               callback 执行
```

### 回收机制

| API | 行为 | 阻塞？ |
|-----|------|-------|
| `synchronize_rcu()` | 阻塞直到所有已有 RCU 读侧临界区完成 | 是（阻塞调用者） |
| `call_rcu(&head, callback)` | 注册回调，grace period 完成后执行 | 否（立即返回） |
| `kfree_rcu(ptr, rhf)` | 等价于 `call_rcu()`，但 callback 中自动 `kfree()` | 否 |
| `kfree_rcu_mightsleep(ptr)` | 无需 `rcu_head` 字段，但须在可睡眠上下文调用 | 否 |

---

## 四、最关键的规则：先移除，再回收（Remove Before Reclaim）

### 问题的本质

`call_rcu()` 保证的是：**调用时已存在的读者** 已经完成。它 **不保证** 在 grace period 开始后 **新读者** 不会找到该对象。

这就是为什么对象必须在 `call_rcu()` **之前** 从数据结构中移除——这样才能确保新读者无法再找到它。

### 正确顺序（4 步法）

```
1. 从数据结构中移除对象
   └── 防止新读者找到该对象
2. 调用 call_rcu() 或 synchronize_rcu()
   └── 等待已有读者完成
3. Grace period 结束，callback 执行
4. 在 callback 中释放对象
   └── 此时所有读者（无论新旧）都不会再访问该对象
```

### 正确的代码

```c
hlist_del_rcu(&obj->node);         // 1. 先移除：新读者找不到它
call_rcu(&obj->rcu, free_callback); // 2. 等已有读者完成

void free_callback(struct rcu_head *rhp) {
    struct obj *obj = container_of(rhp, struct obj, rcu);
    kfree(obj);                    // 4. 安全释放
}
```

### 错误的代码（use-after-free bug）

```c
call_rcu(&obj->rcu, free_callback);  // 错！对象仍在数据结构中！

void free_callback(struct rcu_head *rhp) {
    struct obj *obj = container_of(rhp, struct obj, rcu);
    hlist_del_rcu(&obj->node);       // 错！太晚了！
    kfree(obj);
}
```

**这个错误为什么会导致 use-after-free**：

```
时间线 →  (对象 obj 仍在哈希表中)
CPU 0 (写者)                    CPU 1 (读者1)                 CPU 2 (读者2)
───────────                    ────────────                  ────────────
call_rcu(&obj->rcu, ...)                                     
                                rcu_read_lock()
                                p = hlist_search(...) 
                                → 找到 obj
                                                              开始 grace period 之后
                                                              rcu_read_lock()
                                                              p = hlist_search(...)
                                                              → 也找到 obj（还在表中！）
grace period 完成
callback 执行:
  hlist_del_rcu(obj)          使用 p->data → USE-AFTER-FREE
  kfree(obj)                  或 p->data → 已释放内存
                                                              使用 p->data → USE-AFTER-FREE
```

### 检查清单

当你遇到 `call_rcu()`、`synchronize_rcu()` 或 `kfree_rcu()` 时：

1. **立即警觉**——这是一个容易出错但后果严重的地方
2. **找到数据结构的移除操作**——`list_del_rcu()`、`hlist_del_rcu()`、`rhashtable_remove_fast()` 等
3. **确认移除发生在 `call_rcu()` 之前**——而不是在 callback 中
4. **使用正确的 RCU 感知移除函数**——裸的 `list_del()` 不够，需要用 `_rcu` 变体

---

## 五、RCU 变体选择

| 变体 | 读侧 API | 可睡眠？ | 使用场景 |
|------|---------|---------|---------|
| RCU | `rcu_read_lock()` / `rcu_read_unlock()` | 否 | 通用，高性能读者 |
| SRCU | `srcu_read_lock()` / `srcu_read_unlock()` | 是 | 读侧临界区需要睡眠 |
| Tasks RCU | 隐式（不自愿切换上下文就是临界区） | N/A | 跟踪进程退出 |
| Tasks Trace RCU | `rcu_read_lock_trace()` / `rcu_read_unlock_trace()` | 是 | BPF 等追踪场景 |

**选择依据**：
- 如果读侧临界区很短（只是查指针读数据）→ 用 RCU
- 如果读侧临界区需要睡眠（如等待 I/O）→ 用 SRCU（需要定义 `struct srcu_struct`）
- 读侧临界区可嵌套：`rcu_read_lock()` 可嵌套，`srcu_read_lock()` 返回一个索引用于 `srcu_read_unlock()`

---

## 六、写者侧生命周期

### 指针替换后的释放

```c
// 错误：直接 kfree 旧指针
CPU 0 (写者)                    CPU 1 (读者)
──────────────                    ──────────────
                                  rcu_read_lock()
                                  p = rcu_dereference(ptr) → 拿到旧对象
rcu_assign_pointer(ptr, new)
kfree(old) ← BUG!
                                  use(p->val) → USE-AFTER-FREE
                                  rcu_read_unlock()
```

**正确做法**：替换后的旧指针必须通过 `call_rcu()`、`kfree_rcu()` 或在 `synchronize_rcu()` 之后释放。

```c
old = rcu_replace_pointer(ptr, new, lockdep_held);
call_rcu(&old->rcu_head, free_fn);
// callback 只在所有已有 RCU 读侧临界区完成后才执行
```

---

## 七、快速检查工具

- `rcu_read_lock_held()` —— lockdep 调试断言，检查当前是否在 RCU 读侧临界区内
- `INIT_RCU_HEAD` —— **已从内核中移除**，不要使用
- `rcu_barrier()` —— 等待所有待处理的 `call_rcu()` callback 完成；模块卸载时需要调用，确保所有 callback 在模块代码被移除前执行完毕

---

## 八、kvfree_call_rcu() / kfree_rcu() 调用上下文

`kvfree_call_rcu()`（通过 `kfree_rcu`/`kvfree_rcu` 宏调用）在某些路径中是在 `raw_spinlock_t`（`pi_lock`）下调用，甚至从硬中断上下文调用。如果在 `kvfree_call_rcu()` 或其 callee 中加入 `spinlock_t`、`local_lock` 或 `local_trylock` 的获取，会导致 lockdep 的 "Invalid wait context" 警告。

**注意**：`!IS_ENABLED(CONFIG_PREEMPT_RT)` 的编译时守卫 **不能阻止这个问题**——因为 `CONFIG_PROVE_RAW_LOCK_NESTING`（默认 y）检查的是声明的 wait-type，不是运行时行为。

不要因为其他代码使用了同样的锁类型就认为这是安全的。
