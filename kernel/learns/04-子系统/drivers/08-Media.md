<!-- Source: (general kernel knowledge) -->

# Media（多媒体）子系统

## 概述

Linux Media 子系统为视频采集、视频输出、V4L2（Video for Linux 2）、摄像头、编解码器和 DMA 缓冲管理提供统一框架。核心包括 V4L2 框架、Media Controller（媒体控制器）、V4L2 子设备和 VB2（Videobuf2）缓冲管理。

---

## V4L2 设备模型

### 关键数据结构

V4L2 驱动使用以下核心结构体：

| 结构体 | 用途 |
|--------|------|
| `struct video_device` | 表示一个 V4L2 视频设备节点 |
| `struct v4l2_device` | 顶层 V4L2 设备实例 |
| `struct v4l2_subdev` | 表示一个子设备（如传感器、解串器） |
| `struct vb2_queue` | 视频缓冲区队列 |

### 注册顺序

V4L2 设备的注册必须遵循严格的顺序：

1. 分配并初始化 `struct video_device`
2. 设置 `v4l2_dev`、`v4l2_file_operations` 和 `ioctl_ops`
3. 调用 `video_register_device()` 使设备对用户空间可见

### 不变式

- **注册前初始化完成**：`video_register_device()` 后设备立即对用户空间可见，所有回调必须在此之前准备就绪
- **release 回调**：必须设置 `video_device->release()` 回调，用于最后引用释放时的清理
- **设备卸载**：`video_unregister_device()` 后设备可能仍有用户空间引用，release 回调负责最终清理

---

## Media Controller（媒体控制器）

### 关键知识点

Media Controller 框架实现复杂的管道拓扑，在具有多个子设备的 SoC 摄像头系统中尤为重要。

### 核心概念

| 概念 | 说明 |
|------|------|
| **Entity（实体）** | 管道中的逻辑单元（传感器、ISP、DMA 引擎） |
| **Pad（垫）** | Entity 的输入/输出端口 |
| **Link（链接）** | Pad 之间的连接 |
| **Pipeline（管道）** | 相互链接的实体链 |

### 不变式

- **链接设置**：`media_entity_setup_link()` 只能在实体未处于 busy 状态时调用
- **管道电源管理**：启用流（stream on）时，框架自动开启管道中的所有实体
- **实体状态跟踪**：`media_entity_pipeline_start()` / `media_entity_pipeline_stop()` 必须在流控制周围成对使用

### 故障模式

未正确使用 Media Controller API 导致：
- 管道电源管理不一致
- 链接配置与硬件拓扑不匹配
- 流控制期间死锁

---

## V4L2 子设备

### 子设备操作

`struct v4l2_subdev_ops` 定义子设备支持的操作，包括：

- `core`：核心操作（init、load_fw、reset）
- `video`：视频操作（s_stream、g_std、s_std）
- `pad`：pad 级操作（get_fmt、set_fmt、enum_mbus_code）
- `sensor`：传感器特定操作（g_skip_frames、get_sensor_info）

### 不变式

- `s_stream(enable=true)` 和 `s_stream(enable=false)` 必须严格配对
- 格式协商（`set_fmt`、`get_fmt`）可在流激活前或激活后进行
- 子设备的电源管理应遵循 PM 运行时框架
- 子设备初始化后调用 `v4l2_device_register_subdev()` 注册

### 故障模式

- s_stream 配对错误导致硬件保持流状态不一致
- 格式协商未正确实现软约束（不支持的格式回退到最接近的格式）

---

## Videobuf2（VB2）缓冲管理

### 缓冲生命周期

VB2 是 V4L2 的标准缓冲管理框架，支持多种内存类型（MMAP、USERPTR、DMABUF）。

### 核心操作

| 回调 | 说明 |
|------|------|
| `queue_setup` | 创建缓冲队列，返回缓冲区数量和每缓冲大小 |
| `buf_prepare` | 准备缓冲区用于硬件操作 |
| `buf_queue` | 将缓冲区排队到驱动内部队列 |
| `start_streaming` | 启动硬件流 |
| `stop_streaming` | 停止硬件流 |
| `buf_finish` | Buffer 清理 |

### 不变式

- `start_streaming` 失败时必须在返回错误前将排队的缓冲返回给 VB2 框架（通过 `return_all_buffers()`）
- `stop_streaming` 必须返回所有仍在驱动队列中的缓冲
- VB2 缓冲的同步访问必须通过 `vb2_buffer_done()` 和 `VB2_BUF_STATE_*` 状态管理
- `buf_prepare` 不应执行可能阻塞的长时间操作（其可能在原子上下文中被调用）

### 故障模式

- start_streaming 失败但未返回缓冲区，导致 VB2 框架永远等待
- stop_streaming 中未正确处理未处理的缓冲，导致 DMA 残留或悬空引用
- buf_prepare 中执行 I/O 操作或休眠

---

## 快速检查清单

- **注册顺序**：`video_register_device()` 前确保所有结构体和回调已完全初始化
- **VB2 缓冲区管理**：start_streaming 失败必须返回缓冲区；stop_streaming 必须清空队列
- **s_stream 配对**：确保 `s_stream(1)` 和 `s_stream(0)` 成对出现
- **Media Controller 管道管理**：`pipeline_start`/`pipeline_stop` 必须成对使用
- **子设备电源管理**：子设备应正确使用 PM 运行时框架管理电源
- **格式协商**：set_fmt 应实现可用的最小回退逻辑
