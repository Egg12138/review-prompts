<!-- Source: subsystem/sunrpc.md -->

# SunRPC 子系统

## 概述

SunRPC（`net/sunrpc/`）为 NFS 及相关服务提供 RPC 传输层。包括客户端（`rpc_clnt`、`rpc_task`、`xprt`）和服务器端（`svc_serv`、`svc_rqst`、`svc_xprt`）基础设施，支持 TCP、UDP 和 RDMA 传输，以及 RPCSEC_GSS 认证。

## 核心文件

| 文件 | 领域 |
|------|------|
| `svc.c`、`svc_xprt.c`、`clnt.c`、`sched.c`、`xprt.c` | Core |
| `svcsock.c`、`xprtsock.c` | Socket |
| `xprtrdma/svc_rdma_*.c`、`xprtrdma/*.c` | RDMA |
| `auth_gss/*.c`、`svcauth_gss.c`、`gss_krb5_*.c` | GSS |

## 核心基础设施模式 [SUNRPC-CORE]

### SUNRPC-CORE-001：异步工作的传输引用

- **风险**：Use-after-free
- **详情**：在 `queue_work` 之前持有 `svc_xpt_get()` / `xprt_get()`；在 handler 中释放

### SUNRPC-CORE-002：传输状态标志检查

- **风险**：Use-after-free、无效操作
- **详情**：在传输访问前检查 `XPT_CLOSE` / `XPT_DEAD`（服务端）或 `XPRT_CONNECTED` / `XPRT_CLOSING`（客户端）

### SUNRPC-CORE-003：线程池关闭（服务端）

- **风险**：控制器挂起
- **详情**：所有线程停止路径上必须调用 `svc_exit_thread()` 以清除 `SP_VICTIM_REMAINS`；当 `kthread_stop()` 赢得竞争时，控制器必须调用它

### SUNRPC-CORE-004：页数组边界（服务端）

- **风险**：缓冲区溢出
- **详情**：在 `svc_rqst_replace_page()` 之前检查 `rq_next_page < rq_page_end`

### SUNRPC-CORE-005：XPT_BUSY 生命周期（服务端）

- **风险**：传输永久卡住
- **详情**：每个 `svc_handle_xprt()` 退出路径必须调用 `svc_xprt_received()` 清除 `XPT_BUSY`，包括预留失败路径

### SUNRPC-CORE-006：延迟请求上下文（服务端）

- **风险**：Double-free
- **详情**：延迟时，设置 `dr->xprt_ctxt = rqstp->rq_xprt_ctxt` 然后将 `rqstp->rq_xprt_ctxt = NULL`；重新访问时反向操作

### SUNRPC-CORE-007：RPC 任务状态机（客户端）

- **风险**：任务停滞或状态损坏
- **详情**：每个 `call_*` 状态必须设置 `tk_action` 为下一个状态；当 `tk_action` 为 NULL（任务正在退出）时不要修改 `tk_status`

### SUNRPC-CORE-008：槽位释放（客户端）

- **风险**：槽位耗尽
- **详情**：在 `xprt_reserve()` 之后，所有路径上必须调用 `xprt_release()`

### SUNRPC-CORE-009：拥塞窗口加锁（客户端）

- **风险**：竞争条件
- **详情**：更新 `xprt->cong` 时持有 `transport_lock`

### SUNRPC-CORE-010：超时溢出（客户端）

- **风险**：整数溢出
- **详情**：移位后钳制 `rq_timeout`：如果 `rq_timeout > to_maxval` 则 `rq_timeout = to_maxval`

### SUNRPC-CORE-011：rq_flags 原子性（服务端）

- **风险**：与 `svc_xprt_enqueue` 竞争
- **详情**：对 `rq_flags` 使用 `set_bit()` / `clear_bit()`，而不是 `__set_bit()` / `__clear_bit()`

### SUNRPC-CORE-012：传输重新分配（客户端）

- **风险**：引用泄漏
- **详情**：在分配新传输之前释放旧的 `xprt` 引用

## Socket 传输模式 [SUNRPC-SOCK]

### SUNRPC-SOCK-001：短读/写处理

- **风险**：数据损坏、部分处理
- **详情**：TCP 可能返回更少的字节；循环直到完成，或对固定大小读取（如 4 字节记录标记）使用 `MSG_WAITALL`

### SUNRPC-SOCK-002：记录标记边界

- **风险**：内存耗尽、溢出
- **详情**：分配前验证传入的记录大小

### SUNRPC-SOCK-003：监听器回调继承（服务端）

- **风险**：Use-after-free
- **详情**：子 socket 从监听器继承 `sk_user_data`；回调在解引用 `sk_user_data` 前必须检查 `sk->sk_state == TCP_LISTEN`

### SUNRPC-SOCK-004：回调拆除（客户端）

- **风险**：与正在进行的回调竞争
- **详情**：`lock_sock()`；`xs_restore_old_callbacks()`；`sk->sk_user_data = NULL`；`release_sock()`；然后是 `sock_release()`

### SUNRPC-SOCK-005：Cork 平衡（服务端）

- **风险**：Cork 泄漏
- **详情**：在所有路径（包括错误路径）上调用 `tcp_sock_set_cork(sk, false)`

### SUNRPC-SOCK-006：重连退避（客户端）

- **风险**：连接风暴
- **详情**：使用 `xprt_reconnect_delay()` 和 `queue_delayed_work()`，不要使用立即的 `queue_work()`

### SUNRPC-SOCK-007：NOFS 分配（客户端）

- **风险**：回收路径死锁
- **详情**：在 socket 操作周围使用 `memalloc_nofs_save() / restore()`

### SUNRPC-SOCK-008：状态标志屏障

- **风险**：不一致的状态可见性
- **详情**：在相关的标志更改之间使用 `smp_mb__after_atomic()`

### SUNRPC-SOCK-009：新的 XPRT_SOCK_* 标志（客户端）

- **风险**：标志在重置后持续存在
- **详情**：在 `xs_sock_reset_state_flags()` 中为新标志添加 `clear_bit()`

### SUNRPC-SOCK-010：写空间回调（服务端）

- **风险**：死锁
- **详情**：当写空间可用时，`svc_write_space()` 必须调用 `svc_xprt_enqueue()`；读/写耦合意味着服务端在无法写入时会停止读取

## RDMA 传输模式 [SUNRPC-RDMA]

### SUNRPC-RDMA-001：DMA 映射生命周期

- **风险**：Use-after-free、数据损坏
- **详情**：DMA 映射必须持续到完成回调触发；检查 `ib_dma_map_sg()` 返回值（`mr_nents == 0` 是失败）；在完成处理程序或同步等待后取消映射

### SUNRPC-RDMA-002：MR/FRWR 重用前失效

- **风险**：数据损坏、协议错误
- **详情**：内存区域必须完成失效后才能重用；`frwr_unmap_sync()` 是阻塞的，`frwr_unmap_async()` 不是——异步失效后不要立即重用 MR

### SUNRPC-RDMA-003：完成状态检查

- **风险**：垃圾数据、崩溃
- **详情**：工作完成中只有 `wr_cqe` 和状态可靠；在访问 `byte_len` 或其他字段前检查 `wc->status == IB_WC_SUCCESS`

### SUNRPC-RDMA-004：设备移除处理

- **风险**：崩溃
- **详情**：在 `IB_WC_WR_FLUSH_ERR` 时，设备可能已不存在；在 flush 错误路径中不要调用 `ib_dma_*` 函数

### SUNRPC-RDMA-005：Post-send 后 Use-after-Free（服务端）

- **风险**：Use-after-free
- **详情**：完成处理程序可能在 `ib_post_send()` 后立即触发；如果需要用于错误处理或跟踪，在 posting 之前将上下文字段复制到栈上

### SUNRPC-RDMA-006：SQ 计数和标志顺序（服务端）

- **风险**：竞争性重分配
- **详情**：在 `svc_rdma_*_ctxt_put()` 之前设置 `set_bit(XPT_CLOSE)`，以防止竞争的完成处理程序重分配已释放的上下文

### SUNRPC-RDMA-007：CM 事件引用平衡（客户端）

- **风险**：引用下溢
- **详情**：`ESTABLISHED` 通过 `rpcrdma_ep_get()` 获取引用；`DISCONNECTED` 通过 `rpcrdma_ep_put()` 释放；`DEVICE_REMOVAL` / `ADDR_CHANGE` 可能在 `ESTABLISHED` 之前触发——需要跟踪引用是否已获取

### SUNRPC-RDMA-008：重连 DMA 重映射（客户端）

- **风险**：重连时出现 `LOCAL_PROT_ERR`
- **详情**：如果拆除调用 `rpcrdma_regbuf_dma_unmap()`，验证重连路径在 posting 接收前进行了重映射

### SUNRPC-RDMA-009：信用流顺序（客户端）

- **风险**：RNR（接收端未就绪）
- **详情**：在 `rpcrdma_update_cwnd()` 之前调用 `rpcrdma_post_recvs()`；开放信用会唤醒发送者，而发送者需要已发布的接收

## GSS 认证模式 [SUNRPC-GSS]

### SUNRPC-GSS-001：序列窗口加锁（服务端）

- **风险**：重放攻击
- **详情**：所有序列窗口操作（检查、推进、设置位）都需持有 `sd_lock`；验证算术运算处理 `MAXSEQ` 附近的溢出和 `sd_max - GSS_SEQ_WIN` 的下溢

### SUNRPC-GSS-002：上下文缓存引用计数（服务端）

- **风险**：Use-after-free
- **详情**：`gss_svc_searchbyctx()` 返回已获取引用的条目；使用上下文，然后 `cache_put()`；put 后不要访问

### SUNRPC-GSS-003：加密结果检查

- **风险**：认证绕过、伪造请求
- **详情**：检查 `gss_verify_mic()` / `gss_wrap()` / `gss_unwrap()` 返回状态；在任何 `GSS_S_*` 错误时中止请求

### SUNRPC-GSS-004：缓冲区松弛空间计算（客户端）

- **风险**：缓冲区溢出
- **详情**：对于 Kerberos v2（RFC 4121），`au_ralign != au_rslack`，因为校验和跟在明文之后；需要计算 `GSS_KRB5_TOK_HDR_LEN` + 校验和

### SUNRPC-GSS-005：凭据生命周期

- **风险**：Use-after-free
- **详情**：服务端：使用 `rsci->cred` 前调用 `get_group_info()`，通过 `free_svc_cred()` 释放；客户端：所有使用完成后才能调用 `put_rpccred()`

### SUNRPC-GSS-006：Upcall 匹配

- **风险**：返回错误的上下文
- **详情**：`__gss_find_upcall()` 必须匹配 uid + service + in-flight 状态；条件不足可能配对上错误的 upcall/downcall

### SUNRPC-GSS-007：延迟请求验证（服务端）

- **风险**：重复验证开销
- **详情**：跳过 `rqstp->rq_deferred` 请求的重新验证；首次通过时已验证过

## 网络命名空间（Network Namespace）

所有 SunRPC 代码必须尊重网络命名空间边界：

- 服务端传输：`xprt->xpt_net`、`serv->sv_net`
- 客户端传输：`xprt->xprt_net`、`clnt->cl_xprt->xprt_net`
- Socket 创建：`sock_create_kern(net, ...)`
- RDMA：`rdma_create_id(net, ...)`、`rdma_dev_access_netns()`
- 永远不要使用 `&init_net` 或 `current->nsproxy->net_ns`

## Invariants（不变式）

- 传输访问前检查 CLOSE/DEAD 标志
- 异步工作入队前持有引用
- 所有错误路径释放资源
- 验证 GSS 结果后再继续

## Quick Reference（快速参考）

**传输状态标志：**
- 服务端：`XPT_BUSY`、`XPT_CLOSE`、`XPT_DEAD`、`XPT_DATA`、`XPT_CONN`
- 客户端：`XPRT_CONNECTED`、`XPRT_CONNECTING`、`XPRT_CLOSING`

**引用计数函数：**
- 服务端传输：`svc_xprt_get()` / `svc_xprt_put()`
- 客户端传输：`xprt_get()` / `xprt_put()`
- RPC 任务：`rpc_get_task()` / `rpc_put_task()`
- GSS 上下文：`gss_get_ctx()` / `gss_put_ctx()`

## 需要专家评审的触发条件

当以下内容被修改时标记为需专家评审：
- 服务/池分配或线程管理
- 引入新的 `XPT_*` / `XPRT_*` 标志
- FSM 状态转换或槽位分配算法变更
- 回调注册/注销
- FRWR 注册/失效顺序修改
- CM 事件处理程序或完成顺序变更
- GSS 上下文建立或序列窗口逻辑变更
- 加密算法选择或 upcall 机制修改
- 变更超过 100 行，触及多个核心文件

## Quick Checks（快速检查清单）

- [ ] 异步工作前是否持有传输引用？
- [ ] 传输访问前检查了 `XPT_CLOSE` / `XPT_DEAD`？
- [ ] 所有错误路径都释放了资源？
- [ ] GSS 加密结果检查是否完备？
- [ ] RDMA 完成回调中使用了正确的状态检查？
- [ ] CM 事件引用是否平衡？
- [ ] 是否使用了正确的网络命名空间？
