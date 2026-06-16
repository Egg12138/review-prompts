<!-- Source: subsystem/smb-ksmbd.md -->

# SMB/ksmbd 子系统

## SMB Direct（RDMA）信用授予顺序（Credit Grant Ordering）

在协商响应（negotiation response）之前发送信用授予消息会导致协议违规、连接失败或客户端未定义行为。协商响应必须是第一个授予对端信用的消息。

信用（Credits）控制每一侧可以有多少个未完成的发送请求。协商响应（`smb_direct_send_negotiate_response()` 在 `fs/smb/server/transport_rdma.c` 中）和数据传输消息（`smb_direct_create_header()`）都通过 `manage_credits_prior_sending()` 设置 `credits_granted`。协议要求任何携带非零 `credits_granted` 的数据传输消息不能先于协商响应发送。

`smbdirect_socket` 结构体（`fs/smb/common/smbdirect/smbdirect_socket.h`）上的两个工作项可以触发信用授予发送：

| 工作项 | 处理函数（ksmbd） | 效果 |
|--------|-----------------|------|
| `recv_io.posted.refill_work` | `smb_direct_post_recv_credits()` | 发布接收缓冲区，然后如果发布了任何信用，排队 `idle.immediate_work` |
| `idle.immediate_work` | `smb_direct_send_immediate_work()` | 调用 `smb_direct_post_send_data()` 带零载荷，发送一个携带 `credits_granted` 的数据传输 PDU |

链条是：`recv_io.posted.refill_work` 处理函数发布接收缓冲区，然后调用 `queue_work(sc->workqueue, &sc->idle.immediate_work)`。`idle.immediate_work` 处理函数发送一个空的数据传输消息，向对端授予信用。

### 禁用工作项初始化模式

`smbdirect_socket_init()` 使用一个空处理函数（`__smbdirect_socket_disabled_work`，触发 `WARN_ON_ONCE`）初始化所有工作项，并立即通过 `disable_work_sync()` 禁用它们。这确保了在已禁用的工作项上调用 `queue_work()` 会被静默丢弃。真正的处理函数稍后通过 `INIT_WORK()` 在协议状态机的正确时间点分配：

```c
// 在 smb_direct_prepare_negotiation() 序列中（transport_rdma.c）：
// 1. refill_work 获取其真正的处理函数并同步运行
INIT_WORK(&sc->recv_io.posted.refill_work, smb_direct_post_recv_credits);
smb_direct_post_recv_credits(&sc->recv_io.posted.refill_work);
// 此时 idle.immediate_work 仍然被禁用，
// 因此 smb_direct_post_recv_credits 内部的 queue_work() 是空操作。

// 2. 只有在这之后 idle.immediate_work 才获取其真正的处理函数
INIT_WORK(&sc->idle.immediate_work, smb_direct_send_immediate_work);

// 3. 最后发送协商响应（第一个信用授予）
ret = smb_direct_send_negotiate_response(sc, ret);
```

任何在协商响应发送之前启用 `idle.immediate_work` 的变更——或绕过了 `disable_work` 机制——都会打破信用授予顺序的不变式。

## Invariants（不变式）

- 携带非零 `credits_granted` 的第一个消息必须是协商响应
- `idle.immediate_work` 在协商完成之前必须保持禁用
- 协议状态机中的工作项初始化顺序必须严格遵守

## Failure Modes（故障模式）

| 违规操作 | 结果 |
|---------|------|
| 在协商响应前通过 `idle.immediate_work` 发送信用 | 协议违规、连接失败、客户端未定义行为 |
| 移除了 `disable_work_sync()` 调用 | 过早的信用授予 |
| 重新排序 `INIT_WORK()` 赋值 | 暴露过早信用授予的时序窗口 |
| 将 `delayed_work` 转换为 `work_struct`（或反之） | 改变处理程序首次可运行的时间，可能在协商响应前打开时序窗口 |

## Quick Checks（快速检查清单）

- [ ] 对 `smbdirect_socket_init()` 的变更是否移除了 `disable_work_sync()` 调用？
- [ ] `INIT_WORK()` 赋值是否在正确的顺序中（先 `refill_work`，后 `idle.immediate_work`，最后发送协商响应）？
- [ ] 是否转换了 `delayed_work` 与 `work_struct`？这可能改变处理程序首次可运行的时间窗口
- [ ] 任何路径是否可能绕过 `disable_work` 机制，在协商响应前启用 `idle.immediate_work`？
