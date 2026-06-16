<!-- Source: subsystem/can.md (adapted from kernel CAN subsystem knowledge) -->

# CAN（Controller Area Network）子系统

## 概述

Linux CAN 子系统（`net/can/`）实现 Controller Area Network 协议栈，包括 CAN 总线协议（CAN 2.0）和 CAN FD（灵活数据速率）。核心组件：AF_CAN 协议族、RAW 和 BCM（Broadcast Manager）套接字、CAN ISO-TP（ISO 15765-2）、J1939 协议、以及 CAN GW（网关）模块。

## 关键文件

| 文件 | 领域 |
|------|------|
| `net/can/af_can.c` | AF_CAN 协议族，路由规则 |
| `net/can/raw.c` | CAN_RAW 套接字 |
| `net/can/bcm.c` | 广播管理器 |
| `net/can/gw.c` | CAN 网关 |
| `net/can/isotp.c` | ISO-TP 协议 |
| `net/can/j1939/` | SAE J1939 协议 |
| `include/uapi/linux/can.h` | CAN 帧格式 uAPI |

## CAN 帧格式

CAN 帧使用 `struct can_frame`（`include/uapi/linux/can.h`），包含：

- `can_id`：CAN ID（11 位或 29 位扩展 ID），最高位 `CAN_EFF_FLAG` 表示扩展帧，`CAN_RTR_FLAG` 表示远程帧
- `can_dlc`：数据长度码（0-8，CAN FD 为 0-64）
- `data`：最多 8 字节数据（CAN FD 使用 `struct canfd_frame`，支持最多 64 字节）

**常见错误：**

- `can_dlc` 允许值范围是 0-8（CAN 2.0）/ 0-64（CAN FD），但在协议层面上 `can_dlc` 的实际编码（CAN 控制器期望的 DLC 值）可能不同——例如 DLC=8 在 classical CAN 帧中编码为 8，但在 CAN FD 帧中 DLC 编码表示的实际数据长度可能有所不同。在设置和读取数据长度时，必须使用正确的映射函数 `can_dlc2len()` 和 `can_len2dlc()`。
- 未正确设置 `CAN_EFF_FLAG` 会导致标准帧和扩展帧之间混淆，导致过滤和路由错误。

## CAN RAW 套接字

`CAN_RAW` 套接字允许用户空间直接发送和接收原始 CAN 帧。

**过滤规则：**
- 通过 `setsockopt(CAN_RAW_FILTER)` 设置过滤器，使用 `struct can_filter`
- 每个过滤器包含 `can_id` 和 `can_mask`。接收的帧必须满足 `(rx_id & mask) == (filter_id & mask)`
- 如果未设置过滤器，套接字接收所有帧——可能接收到不期望的流量
- 过滤器 `can_id` 中设置 `CAN_INV_FILTER` 会反转匹配逻辑

**关键点：**

- 套接字接收缓冲区可能被快速 CAN 帧填满。UDP 风格的接收溢出处理——通过 `setsockopt(CAN_RAW_RECV_OWN_MSGS)` 控制是否接收自己发送的帧
- 对于循环，需要显式设置为 `setsockopt(CAN_RAW_LOOPBACK, ...)`。循环回帧以 `CAN_RTR_FLAG` 之外的方式标识

## CAN 广播管理器（BCM）

BCM 提供周期性和变化触发的 CAN 消息传输，减少用户空间轮询开销。

- **周期性传输：** 使用 `struct bcm_msg_head` 的 `can_msg` 和 `nframes` 字段。发送间隔由 `ival1` 和 `ival2` 控制
- **变化检测：** 监视 CAN ID 集合，仅在检测到变化时发送更新
- **删除操作：** 删除操作必须验证待删除的条目确实属于该套接字。`bcm_delete_rx_op()` 中的 `op_id` 和 `flags` 匹配不足可能导致操作用户空间的错误项目

**常见错误：**
- BCM 操作中的索引验证不足可能导致越界访问。`op->nframes` 是用户控制的，必须在复制帧数据前验证
- `bcm_rx_handler()` 中的 `CANCEL` 和 `DELETE` 操作需要适当的同步。在接收处理程序运行时删除操作会导致 use-after-free

## CAN ISO-TP（ISO 15765-2）

ISO-TP 协议用于在 CAN 总线上传输大于 8 字节的 PDU，支持分段和流控制。

**关键点：**

- **单帧（SF）/ 多帧（CF）处理：** 接收处理必须正确组装分段消息。`FC`（流控制）帧的计时参数需要验证，防止长时间挂起
- **地址格式：** 支持标准和扩展寻址。`can_id` 编码取决于地址格式类型
- **发送超时：** `send` 操作在 `TX_DL` 超时内必须组装并确认。`TX_DL` 值影响流控制帧的等待时间
- **接收缓冲区大小：** ISO-TP 模块在接收端将消息组装到缓冲区中。缓冲区大小不足会导致消息截断；用户空间需要查询并确保缓冲区足够。`SO_RCVBUF` 调整可提供更大缓冲

## CAN 网关（GW）

CAN 网关在多个 CAN 接口之间路由和转换消息。

- **转换规则：** `struct can_gw` 定义修改操作（XOR、SET、AND、OR）和过滤条件
- **CRC 修改：** 修改 CAN 帧中的数据时，如果原始应用使用 CRC，修改可能使计算出的 CRC 无效
- **路由：** 网关规则在添加时验证。添加重复规则不会出错，但会浪费资源和导致意外路由行为

## CAN J1939

SAE J1939 是基于 CAN 的高层协议，用于农用和商用车辆。

- **名称和地址管理：** J1939 使用 64 位 NAME 和 8 位 SA（源地址）。冲突解决需要正确的地址声明（`ECU` 声称地址的过程）
- **PGN（参数组编号）：** 路由和过滤基于 PGN。PGN 格式在不同 PDU 格式（PDU1 和 PDU2）间有不同编码
- **会话处理：** J1939 的传输协议（TP）用于多包消息，与 ISO-TP 类似。TP 会话跟踪和超时处理在并发连接下容易出现资源泄漏

## CAN FD 特定的注意事项

CAN FD 帧（`struct canfd_frame`）与 classical CAN 帧不同：

- **数据字段：** `data` 最大 64 字节，BRS（Bit Rate Switch）位指示是否切换到更高速率
- **flags 字段：** `flags` 字段包含 `CANFD_BRS` 和 `CANFD_ESI` 标志。驱动需要在硬件级别检查 BRS/ESI 支持，不是所有控制器都支持
- **长度映射：** `canfd_frame` 的 `len` 字段直接包含数据长度，不是 DLC。`can_dlc2len()` 和 `can_len2dlc()` 转换函数在 CAN FD 和 classical CAN 语义上可能不同

## Invariants（不变式）

- CAN ID 必须正确设置 `CAN_EFF_FLAG` 以区分标准帧和扩展帧
- 使用 `can_dlc2len()` / `can_len2dlc()` 正确映射 DLC 和数据长度
- BCM 操作中的 `nframes` 是用户控制的，必须在使用前验证
- ISO-TP 的发送/接收超时必须有合适范围检查
- J1939 TP 会话在多并发连接下必须正确处理资源释放

## Failure Modes（故障模式）

| 违规操作 | 结果 |
|---------|------|
| 未设置 `CAN_EFF_FLAG` 导致帧类型混淆 | 路由和过滤错误 |
| BCM `nframes` 越界访问 | 缓冲区溢出、崩溃 |
| CAN FD 与 classical CAN 帧混淆 | 数据损坏、控制器错误 |
| ISO-TP 超时值验证不足 | 资源泄漏、协议挂起 |
| J1939 TP 会话泄漏 | 资源耗尽 |
| BCM 删除操作缺少同步 | Use-after-free |

## Quick Checks（快速检查清单）

- [ ] CAN ID 的 `CAN_EFF_FLAG` / `CAN_RTR_FLAG` 设置正确？
- [ ] `can_dlc2len()` / `can_len2dlc()` 在必要时使用？
- [ ] BCM 的 `nframes` 是否验证边界？
- [ ] ISO-TP 超时值是否有合理范围？
- [ ] CAN FD 帧格式（`canfd_frame`）与 classical `can_frame` 正确区分？
- [ ] J1939 TP 会话是否有适当的超时和清理？
- [ ] BCM 中操作的删除是否有同步保护？
