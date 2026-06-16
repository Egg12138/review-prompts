<!-- Source: subsystem/block.md -->

# Block 层详解

## 队列冻结同步（Queue Freezing Synchronization）

在设备拆除或重新配置期间，如果未能持有队列冻结（queue freeze），bio 会并发完成，导致队列状态的 use-after-free、过时的 elevator 数据，或 `q->nr_requests` 的撕裂读取。

`blk_mq_freeze_queue()` 等待 `q->q_usage_counter`（一个 `percpu_ref`）归零。每个 bio 提交路径通过 `blk_try_enter_queue()`（由 `blk_queue_enter()` 调用）获取该计数器，并通过 `blk_queue_exit()` 释放。队列冻结期间，新的 `blk_try_enter_queue()` 调用会失败，确保没有新 bio 进入队列。受冻结保护的代码可以安全地拆除或修改队列状态，而不与 bio 完成路径竞态。

`blk_mq_freeze_queue()` 返回一个来自 `memalloc_noio_save()` 的 `memflags` 值，必须传递给 `blk_mq_unfreeze_queue()`。这可以防止冻结区内的内存回收重新进入块层。

## Invariants（不变规则）

- `blk_mq_freeze_queue()` 返回 `unsigned int`（memflags），而非 `void`。调用者必须捕获返回值并传递给 `blk_mq_unfreeze_queue()`。
- `q->nr_requests` 必须在调用 `depth_updated()` 回调 **之前** 写入。交换顺序会导致所有内置 elevator 出现过期值 bug。
- 所有三个内置 elevator（mq-deadline、BFQ、kyber）在 `init_sched()` 结束时调用各自的 `depth_updated()` 实现。新 elevator 也必须如此。

## Bio 操作类型安全（Bio Operation Type Safety）

在无数据缓冲区的 bio（如 discard、flush）上访问 bio 数据字段会导致 NULL 指针解引用。处理 bio 的代码必须在使用前检查操作类型。

| 操作 | `bi_io_vec` | 有数据 |
|-----|-------------|--------|
| `REQ_OP_READ` | 有效 | 是 |
| `REQ_OP_WRITE` | 有效 | 是 |
| `REQ_OP_DISCARD` | NULL | 否 |
| `REQ_OP_FLUSH` | NULL | 否 |
| `REQ_OP_WRITE_ZEROES` | NULL | 否 |
| `REQ_OP_SECURE_ERASE` | NULL | 否 |

**需要守卫的数据字段访问：**
- 直接访问：`bio->bi_io_vec`、`bio->bi_vcnt`、`bio->bi_iter.bi_bvec_done`
- 间接访问：`bio_get_first_bvec()`、`bio_get_last_bvec()`、`bio_for_each_bvec()`、`bio_for_each_segment()`

**必需的守卫：** `bio_has_data()` 必须在访问任何数据字段前调用。注意 `op_is_write()` **不是** 有效的守卫——它检查操作码的 bit 0，因此对 `REQ_OP_DISCARD`（3）、`REQ_OP_SECURE_ERASE`（5）和 `REQ_OP_WRITE_ZEROES`（9）也返回 true，而这些操作都没有数据。`bio_has_data()` 通过显式排除这些操作（并要求 `bi_iter.bi_size` 非零）来正确处理。

## Failure Modes（失败模式）

- 在 `GFP_NOIO` / `GFP_NOFS` 下将 mempool 支持的 bio 分配失败视为可达路径 → 死代码。
- 省略 `bio_kmalloc()` 或 `GFP_NOWAIT` 分配的错误处理 → 内存压力下 NULL 解引用。
- `bio_alloc_bioset()` 即使有 `__GFP_DIRECT_RECLAIM` 也可能返回 NULL：当 `nr_vecs > 0` 且 bioset 没有初始化 bvec 池时（触发 `WARN_ON_ONCE` 并返回 NULL）。

## Bio Mempool 分配保证

- `bio_alloc()` / `bio_alloc_bioset()` — mempool 支持；当设置了 `__GFP_DIRECT_RECLAIM`（`GFP_NOIO` 和 `GFP_NOFS` 都包含此标志）时不会失败。失败路径仅在 `GFP_NOWAIT` / `GFP_ATOMIC` 下可达。
- `bvec_alloc()` — 先尝试 slab 分配；若失败且设置了 `__GFP_DIRECT_RECLAIM`，回退到 mempool（不会失败）。
- `bio_integrity_prep()` — 从 mempool 以 `GFP_NOIO` 分配；始终返回 `true`。
- `bio_integrity_alloc_buf()` — 尝试带 `__GFP_DIRECT_RECLAIM` 的 `kmalloc()`（排除 `__GFP_DIRECT_RECLAIM`）；失败时回退到 `mempool_alloc()` 以 `GFP_NOFS`（不会失败）。
- `bio_kmalloc()` — 使用普通 `kmalloc()`，**没有** mempool 支持。无论 GFP 标志如何都可能失败。

## Elevator `depth_updated` 回调

在 `q->nr_requests` 被写入 **之前** 调用 `depth_updated()`，会导致 elevator 基于过期值计算内部限制（如 mq-deadline 和 kyber 中的 `async_depth`，BFQ 中的 `async_depths` 数组）。在 `init_sched()` 期间省略 `depth_updated()` 会使这些限制保持为零，直到首次 sysfs 写入 `nr_requests`。

回调签名固定：`void (*depth_updated)(struct request_queue *)`，定义在 `struct elevator_mq_ops`（`block/elevator.h`）中。所有派生自 `q->nr_requests` 的 elevator 状态是 per-queue 的（存储在 `elevator_data` 中）。

**时序不变规则：** 在 `blk_mq_update_nr_requests()`（`block/blk-mq.c`）中，`q->nr_requests` 在 `depth_updated()` 之前设置。任何改变此赋值顺序的修改都将在所有三个内置 elevator（mq-deadline、BFQ、kyber）中引入过期值 bug。

**初始化不变规则：** 三个内置 elevator 在 `init_sched()` 结束时调用各自的 `depth_updated()` 实现：
- `bfq_init_queue()` 调用 `bfq_depth_updated()`（`block/bfq-iosched.c`）
- `dd_init_sched()` 调用 `dd_depth_updated()`（`block/mq-deadline.c`）
- `kyber_init_sched()` 调用 `kyber_depth_updated()`（`block/kyber-iosched.c`）

新的从 `q->nr_requests` 派生限制的 elevator 也必须在初始化时调用 `depth_updated()`，而非仅在运行时。

## Quick Checks（快速检查要点）

- `REQ_OP_ZONE_APPEND`（7）的 bit 0 为 1，因此 `op_is_write()` 返回 true，并且它确实携带数据——与 DISCARD / WRITE_ZEROES / SECURE_ERASE 不同。
- `blk_mq_freeze_queue()` 返回值是 `unsigned int`（memflags），必须捕获并传递给 `blk_mq_unfreeze_queue()`。
- `bio_alloc_bioset()` 即使有 `__GFP_DIRECT_RECLAIM` 也可能返回 NULL：当 `nr_vecs > 0` 且 bioset 的 bvec 池未初始化时（触发 `WARN_ON_ONCE` 并返回 NULL）。
