<!-- Source: subsystem/io_uring.md -->

# io_uring 子系统详解

io_uring 是 Linux 内核的异步 I/O 框架，提供了一套丰富的零拷贝、缓冲管理、轮询和完成通知机制。以下覆盖关键子系统和常见的 bug 模式。

---

## 零拷贝生命周期管理（Zero-Copy Lifetime Management）

在零拷贝（zero-copy，零复制）操作中，将 `buf_node` 附加到 `req` 而非 `notif` 会导致 use-after-free，因为 `req` 在网络/块层完成使用缓冲区之前就已结束。`notif`（`struct io_kiocb`）仅在传输完成时通过 `io_tx_ubuf_complete()` 完成，因此缓冲区引用（`struct io_rsrc_node`）必须附加到 `notif`。

**零拷贝操作**：`IORING_OP_SEND_ZC` 和 `IORING_OP_SENDMSG_ZC`。内部通过 `io_send_zc_prep()` 设置 `MSG_ZEROCOPY`。

**缓冲区导入附加**：`io_import_reg_buf()` 和 `io_import_reg_vec()` 调用 `io_find_buf_node()`，该函数将 `buf_node` 附加到传入的 `io_kiocb`——第一个参数给 `io_import_reg_buf()`，第三个参数给 `io_import_reg_vec()`。

```c
io_import_reg_buf(sr->notif, ...)                        // 正确
io_import_reg_vec(ITER_SOURCE, &msg_iter, sr->notif, ...) // 正确
io_import_reg_buf(req, ...)                               // 错误
io_import_reg_vec(ITER_SOURCE, &msg_iter, req, ...)       // 错误
```

审查 vectored 零拷贝操作时，与对应的 non-vectored 版本进行一致性比较。`IORING_OP_SEND_ZC` 使用 `io_send_zc_import()`，它正确传递了 `sr->notif`。

## Failure Modes（失败模式）

- 缓冲区导入在零拷贝操作中传递 `req` 而不是 `notif` → use-after-free。
- `io_notif_flush()` 后未将指针置 NULL → 二次调用导致 use-after-free。快速路径（inline flush）和清理路径（`io_send_zc_cleanup()` 通过 `io_clean_op()`）都可能执行 flush。

---

## REQ_F_NEED_CLEANUP 和清理标志安全性

在 prep 函数的早期返回错误路径中缺失 `REQ_F_NEED_CLEANUP` 会导致资源泄漏。`io_clean_op()` 检查此标志以调用 `io_cold_defs[req->opcode].cleanup`；没有该标志则资源永远不会被释放。

## Invariants（不变规则）

在 **任何分配之后立即** 设置 `REQ_F_NEED_CLEANUP`，只要该分配的释放依赖于 opcode 的清理处理器。在分配和设置标志之间不得存在早期返回路径。

```c
open->filename = getname(fname);
if (IS_ERR(open->filename))
    return PTR_ERR(open->filename);
req->flags |= REQ_F_NEED_CLEANUP;  // 正确：在任何验证之前
// ... 可能返回 -EINVAL 的验证在这里是安全的 ...
```

**清除规则**：仅在资源成功回收或释放后清除 `REQ_F_NEED_CLEANUP` / `REQ_F_ASYNC_DATA`。`io_netmsg_recycle()` 仅在 `io_alloc_cache_put()` 成功分支内清除标志。参见 `io_uring/net.c`。

## Failure Modes（失败模式）

- `REQ_F_NEED_CLEANUP` 在可能早期返回的代码路径之后设置 → 资源泄漏。
- 在确认资源已回收/释放之前无条件清除 `REQ_F_NEED_CLEANUP` 或 `REQ_F_ASYNC_DATA` → 释放后使用或双重释放。

---

## 异步数据生命周期（Async Data Lifecycle）

`req->async_data` 和 `REQ_F_ASYNC_DATA` 必须始终保持同步。`io_clean_op()` 只检查标志；不匹配会导致 use-after-free 或 double-free。

## Invariants（不变规则）

- 使用 `io_uring_alloc_async_data(cache, req)` 分配（成功时设置标志）
- 使用 `io_req_async_data_free(req)` 释放（清除指针和标志）
- 使用 `io_req_async_data_clear(req, extra_flags)` 释放缓存归还数据
- 禁止手动赋值 `req->async_data` 而不设置标志，或使用辅助函数之外的 `kfree()` 释放
- 在 `prep` 而非 `issue` 中分配——数据必须在重试或取消前存在。参见 `io_waitid_prep()` 在 `io_uring/waitid.c`

所有辅助函数位于 `io_uring/io_uring.h`。

## Failure Modes（失败模式）

- 手动 `req->async_data` 赋值而无 `REQ_F_ASYNC_DATA` → use-after-free。
- 直接 `kfree(req->async_data)` 绕过辅助函数 → 双重释放或释放后使用。
- 异步数据在 `issue` 而非 `prep` 中分配（对于可取消操作）→ 取消时竞态。

---

## SQE 数据稳定性（uring_cmd）

`IORING_OP_URING_CMD` 将 SQE 解释委托给 `f_op->uring_cmd()`，该函数可能在 prep 之后很久才访问 SQE 字段。如果 SQE 槽在复制前被重用，处理器会读到过期数据。

## Invariants（不变规则）

- 标准 opcode 在 `prep` 期间读取所有 SQE 字段。`uring_cmd` 不同：`ioucmd->sqe` 指向环形槽，在 issue 时和异步完成时使用。
- 异步放逐（async punt）时，`io_uring_cmd_sqe_copy()` 将 SQE 复制到 `ac->sqes` 并更新 `ioucmd->sqe`。`REQ_F_SQE_COPIED` 防止重复复制。复制和指针更新必须一起发生。
- issue 时需要的 SQE 字段必须在 prep 期间用 `READ_ONCE()` 缓存。Issue 代码必须使用缓存值，而非 `cmd->sqe->`。

## Failure Modes（失败模式）

- `ioucmd->sqe` 在异步放逐后未经复制访问 → 过期数据。
- Issue 代码通过 `cmd->sqe->` 而非缓存值读取 SQE 字段 → 数据损坏。

---

## CQE 大小模式：CQE32 vs CQE_MIXED

传递给 `io_get_cqe()` / `io_get_cqe_overflow()` 的错误 `cqe32` 布尔值会导致 CQ tail 指针错误前进和数据损坏。该参数表示"混合模式下每个 CQE 需要额外推进的 32 字节条目"，而**非**"这个 CQE 是 32 字节"。

| 环形模式 | `cqe32` 参数 | 原因 |
|---------|-------------|------|
| `IORING_SETUP_CQE32` | `false` | 环已双倍所有槽；内部处理 |
| `IORING_SETUP_CQE_MIXED` + `IORING_CQE_F_32` | `true` | 每个 CQE 需要额外槽 |
| `IORING_SETUP_CQE_MIXED` 不含 `IORING_CQE_F_32` | `false` | 标准 16B 条目 |
| 默认 16B 环 | `false` | 标准 16B 条目 |

`cqe32` 必须从每个 CQE 的 `IORING_CQE_F_32` 标志派生，而非环形级 `IORING_SETUP_CQE32`。参见 `io_fill_cqe_req()` 在 `io_uring/io_uring.h`。

## Failure Modes（失败模式）

- 基于 `IORING_SETUP_CQE32` 而非 `IORING_CQE_F_32` 设置 `cqe32=true` → CQ tail 错误前进，数据损坏。

---

## Multishot 和 CQE 投递

错误的 CQE 投递上下文或不正确的 multishot 返回值会导致警告、崩溃或挂起的请求。

## Invariants（不变规则）

- **CQE 投递**：`io_req_post_cqe()` 需要 task_work 上下文且持有 `uring_lock`，绝不能在 io-wq 中调用。任何调用它的请求必须设置 `REQ_F_MULTISHOT` 或 `REQ_F_APOLL_MULTISHOT`，以便 `io_wq_submit_work()` 处理。
- **返回值**：`REQ_F_APOLL_MULTISHOT` 表示请求**支持** multishot；`issue_flags & IO_URING_F_MULTISHOT` 表示它当前**正在执行** multishot 上下文。在返回 multishot 特定状态码（如 `IOU_STOP_MULTISHOT`）前检查后者。
- **异步放逐**：Multishot 处理器不得为 io-wq 放逐返回 `-EAGAIN`。参见 `__io_read()` 在 `io_uring/rw.c`。
- **缓冲区回收**：在返回轮询等待前调用 `io_kbuf_recycle()`。参见 `io_recv()` 在 `io_uring/net.c`。

## Failure Modes（失败模式）

- 无 `REQ_F_MULTISHOT` / `REQ_F_APOLL_MULTISHOT` 时调用 `io_req_post_cqe()` → 错误路径。
- Multishot 状态码返回前未检查 `IO_URING_F_MULTISHOT` → 错误投递。
- Multishot 处理器为异步放逐返回 `-EAGAIN` → 无意义的重试循环。

---

## 提供的缓冲环语义（Provided Buffer Ring）

提供的缓冲环（`struct io_uring_buf` 在 `buf_ring` 中）位于用户空间共享内存中。不正确的访问或提交顺序会导致数据损坏、无限循环或悬挂指针。

## Invariants（不变规则）

- **共享内存**：所有 `buf->len`、`buf->addr`、`buf->bid` 读取必须使用 `READ_ONCE()` 到局部变量；写入必须使用 `WRITE_ONCE()`。传统的 `struct io_buffer`（仅内核列表）无需注解。
- **自动提交 vs 显式提交**（`io_should_commit()` 在 `io_uring/kbuf.c`）：
  - `IO_URING_F_UNLOCKED`：始终自动提交
  - 非可轮询、非 uring_cmd：自动提交
  - 可轮询或 `IORING_OP_URING_CMD`：跳过（操作显式提交）
- 新 opcode 若使用显式提交，必须在 `io_should_commit()` 中豁免。
- **地址捕获**：在 `io_kbuf_commit()` 之前捕获 `buf->addr`，因为后者可能修改缓冲区元数据。
- **重试**：部分完成时，通过 `io_kbuf_commit()` 提交并设置 `REQ_F_BL_NO_RECYCLE` 再返回。
- **增量消费**：`io_kbuf_inc_commit()` 必须在零长度缓冲区上终止。

## Failure Modes（失败模式）

- `buf_ring` 字段访问没有 `READ_ONCE()` / `WRITE_ONCE()` → 数据损坏。
- 显式提交 opcode 未在 `io_should_commit()` 中豁免 → 错误提交行为。
- 未提交就设置 `REQ_F_BL_NO_RECYCLE` → 缓冲泄露。

---

## 注册缓冲区管理（Registered Buffer Management）

预注册缓冲区的 bvec 偏移量计算错误会导致静默数据损坏或越界访问。

## Invariants（不变规则）

- 绝不要假设 `imu->ubuf` 是页/ folio 对齐的。使用 `imu->bvec[0].bv_offset` 作为子 folio 偏移量，而非地址掩码。
  ```c
  offset = buf_addr - imu->ubuf;
  offset += imu->bvec[0].bv_offset;  // 正确
  ```
- 注册时有 folio 合并（coalescing），`data.first_folio_page_idx << PAGE_SHIFT` 计入首页在其 folio 内的位置。
- 使用 `unpin_user_folio()`，绝不要用 `unpin_user_page()`——注册缓冲区按 folio 锁定（合并后），因此解锁定必须匹配。

## Failure Modes（失败模式）

- 通过虚拟地址掩码而非 `imu->bvec[0].bv_offset` 派生偏移量 → 数据损坏。
- 在 io_uring 缓冲代码中使用 `unpin_user_page()` → 引用计数不匹配。

---

## msg_ring 跨环请求生命周期

msg_ring 分配的 `io_kiocb` 在 RCU 宽限期（grace period）前释放会导致 use-after-free。`io_msg_data_remote()` 通过 `kmem_cache_alloc()` 分配并投递到远程环；`io_msg_tw_complete()` 通过 `kfree_rcu()` 释放。参见 `io_uring/msg_ring.c`。

## Invariants（不变规则）

- 通过 `kfree_rcu(req, rcu_head)` 释放，绝不用 `kmem_cache_free()` / `kfree()`
- 绝不要放入 `io_alloc_cache`（绕过 RCU 保证）
- 远程请求设置 `req->tctx = NULL`——提交者可能已退出。参见 `io_msg_remote_post()`

## Failure Modes（失败模式）

- msg_ring 请求在 `io_alloc_cache` 中 → 绕过 RCU 保证，use-after-free。
- 未用 `kfree_rcu()` 释放 → use-after-free。
- `req->tctx` 非 NULL → 使用已退出的 task 上下文。

---

## SQPOLL 线程安全

裸 `sqd->thread` 访问会导致 use-after-free——`task_struct` 在线程退出后通过 RCU 释放。`sqd->thread` 是 `__rcu` 注解的（`io_uring/sqpoll.h`）。

## Invariants（不变规则）

- 在 `sqd->lock` 下：`sqpoll_task_locked(sqd)`
- 在 RCU 下：`rcu_dereference(sqd->thread)`
- 赋值：`rcu_assign_pointer(sqd->thread, tsk)`
- 信令：使用 `req->tctx->task`，而非 `sqd->thread`
- 任务所有权：在 `io_sq_offload_create()` 的 `wake_up_new_task()` 之后，线程拥有其引用。创建者在启动后**不得**调用 `put_task_struct()`。

## Failure Modes（失败模式）

- 裸 `sqd->thread` 读取 → use-after-free。
- 线程启动后调用 `put_task_struct()` → 引用计数错误。
- 使用 `sqd->thread` 进行信令 → 竞争条件。

---

## DEFER_TASKRUN 任务工作排空

使用 `IORING_SETUP_DEFER_TASKRUN` 时，task work 进入 `ctx->work_llist`。在非提交者上下文（如 `io_ring_exit_work()`）中，只有 `io_move_task_work_from_local()` 可以排空它。在取消循环前只调用一次，会留下取消期间产生的新工作未排空，导致 100% CPU 自旋。

## Invariants（不变规则）

任何在提交者外部调用 `io_uring_try_cancel_requests()` 的取消循环，都必须在**每次迭代**中对 `io_move_task_work_from_local()` 调用。参见 `io_ring_exit_work()` 在 `io_uring/io_uring.c`。

## Failure Modes（失败模式）

- `io_move_task_work_from_local()` 在取消循环前只调用一次而非每次迭代 → 100% CPU 自旋。

---

## IOPOLL 完成和重试

IOPOLL 在设置 `iopoll_completed` 时投递 CQE。提前返回而不设置它会使请求对轮询不可见，导致挂起。

## Invariants（不变规则）

- `io_complete_rw_iopoll()` 必须始终到达 `smp_store_release(&req->iopoll_completed, 1)`。绝不要提前返回。参见 `io_uring/rw.c`。
- 重试（`-EAGAIN`）：设置 `REQ_F_REISSUE | REQ_F_BL_NO_RECYCLE` 并执行 fall-through。刷新路径在 `io_uring/io_uring.c` 中检查 `REQ_F_REISSUE` 并调用 `io_queue_iowq()`。

## Failure Modes（失败模式）

- 在 `io_complete_rw_iopoll()` 中提前返回跳过 `iopoll_completed` 的设置 → 请求对轮询不可见，导致挂起。

---

## 超时取消和锁顺序

在持有 `ctx->timeout_lock`（raw spinlock）时排队 task_work 会导致锁顺序违规——完成可能调用 `io_eventfd_signal()`，后者获取常规 spinlock，在 PREEMPT_RT 上无效。

## 双阶段模式（Two-Phase Pattern）

1. 在 `timeout_lock` 下：`io_kill_timeout()` 取消 hrtimer 并将超时移至本地列表（绝不完成它们）
2. 解锁后：`io_flush_killed_timeouts()` 调用 `io_req_queue_tw_complete()`

参见 `io_uring/timeout.c`。

## Failure Modes（失败模式）

- 持有 `ctx->timeout_lock` 时调用 task_work 或完成函数 → PREEMPT_RT 上的锁顺序违规。

---

## Quick Checks（快速检查要点）

- **Notif 在导入前分配**：`io_alloc_notif()` 必须在零拷贝路径中的缓冲区导入之前调用。
- **零拷贝标志检测**：`IORING_OP_SEND_ZC`、`IORING_OP_SENDMSG_ZC` 或带 `IORING_RECVSEND_FIXED_BUF` 的 ZC opcode 需要缓冲区生命周期验证。
- **Bundle 缓冲区放置**：`io_put_kbufs()` 使用当前传输计数（`this_ret`），而非累计总数。参见 `io_recv_finish()` 在 `io_uring/net.c`。
- **CQ 溢出临界区**：在溢出刷新期间丢弃 CQ 锁前设置 `ctx->cqe_sentinel = ctx->cqe_cached`，强制并发发射器走 `io_cqe_cache_refill()`。参见 `__io_cqring_overflow_flush()`。
- **zcrx DMA 生命周期**：在 `io_pp_zc_init()` 中创建 DMA 映射，在 `io_pp_uninstall()` 中解除映射。参见 `io_uring/zcrx.c`。
- **注册标签在失败时**：在注销失败的全有或全无注册前，通过 `io_clear_table_tags()` 清除 `node->tag`。参见 `io_uring/rsrc.c`。
- **对 MM 依赖请求的 inflight 跟踪**：在需要提交者 `mm` 的请求的 prep 中调用 `io_req_track_inflight()`。参见 `io_futex_prep()` 在 `io_uring/futex.c`。
- **Task work tokens**：绝不要在栈上伪造 `io_tw_token_t`。在 task_work 上下文之外使用 `io_req_queue_tw_complete()`，而非 `io_req_task_complete()`。
- **缓冲区列表升级安全**：在升级到 ring-mapped 缓冲区时销毁旧的 `io_buffer_list` 并分配新的。参见 `io_register_pbuf_ring()` 在 `io_uring/kbuf.c`。
- **Eventfd RCU 释放**：使用 `io_eventfd_put()`（调用 `call_rcu()`），绝不要直接使用 `io_eventfd_free()`。参见 `io_eventfd_do_signal()` 在 `io_uring/eventfd.c`。
- **跨环克隆记帐**：两个环必须共享 `ctx->user` 和 `ctx->mm_account`。参见 `io_clone_buffers()` 在 `io_uring/rsrc.c`。
- **io_wq 在拆除后为 NULL**：`io_queue_iowq()` 检查 `!tctx->io_wq` 和 `PF_KTHREAD`。参见 `io_uring/io_uring.c`。
- **SQE 标志层级**：用最宽泛的标志守卫 `READ_ONCE(sqe->field)`，覆盖所有变体。参见 `io_nop_prep()` 在 `io_uring/nop.c`。
- **SQE 字段使用前读取**：用 `READ_ONCE()` 将 SQE 字段读入请求后再使用——`req->buf_index` 可能包含重用时的过期数据。参见 `io_uring_cmd_prep()` 在 `io_uring/uring_cmd.c`。
- **RESIZE_RINGS 和 DEFER_TASKRUN**：`io_register_resize_rings()` 需要 `IORING_SETUP_DEFER_TASKRUN`。新的环形几何体突变需要相同的互斥。参见 `io_uring/register.c`。
- **Poll 事件范围**：通用 poll 代码（`io_uring/poll.c`）不得将事件位解释为错误——`POLLERR` 对某些套接字指示数据可用性（如 `MSG_ERRQUEUE`）。操作特定的解释属于 issue handler。
