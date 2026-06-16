<!-- 注意：源文件 subsystem/dpll.md 在仓库中不存在。以下内容基于上游 Linux 内核 DPLL 子系统和 UAPI 归纳。 -->

# DPLL（Digital Phase-Locked Loop）子系统详解

## 概述

Linux 内核 DPLL（Digital Phase-Locked Loop，数字锁相环）子系统（自 Linux 6.7 合并）为配置网络时钟同步中使用的 DPLL 设备提供统一接口。DPLL 设备常用于 SyncE（Synchronous Ethernet，同步以太网）和 IEEE 1588 Precision Time Protocol（PTP，精确时间协议）硬件时钟同步。

主要文件：`drivers/dpll/`，UAPI 定义在 `include/uapi/linux/dpll.h`。

---

## 核心概念

### DPLL 设备（struct dpll_device）

DPLL 设备表示一个物理或虚拟 DPLL 实例，能够锁定到参考时钟并产生同步输出时钟。

- 每个 DPLL 设备有唯一的 **index**（索引）标识
- 通过 netlink 与用户空间通信，使用 `NETLINK_GENERIC` 协议
- 核心操作通过 `struct dpll_device_ops` 定义：

```c
struct dpll_device_ops {
    int (*lock_status_get)(const struct dpll_device *dpll, enum dpll_lock_status *status);
    int (*mode_get)(const struct dpll_device *dpll, enum dpll_mode *mode);
    int (*mode_set)(const struct dpll_device *dpll, const enum dpll_mode mode);
    int (*source_get)(const struct dpll_device *dpll, u32 *source_id);
    int (*source_set)(const struct dpll_device *dpll, const u32 source_id);
};
```

### DPLL 引脚（struct dpll_pin）

DPLL 引脚表示 DPLL 设备的输入参考时钟或输出时钟。

- 输入引脚（input pin）：连接到外部参考时钟（如从网络恢复的时钟、本地振荡器）
- 输出引脚（output pin）：DPLL 输出的同步时钟
- 经 netlink 属性传递引脚能力（频率、类型、优先级等）

### 锁定状态（Lock Status）

DPLL 设备的锁定状态生命周期：

```
UNLOCK（未锁定） → HOLDOVER（保持） → LOCKED（已锁定）
                                    → LOCKED_HO_ACQ（已锁定-保持获取中）
```

- `HOLDOVER`：DPLL 失去输入参考但尝试保持最后已知频率
- `LOCKED`：DPLL 已锁定到参考源
- `LOCKED_HO_ACQ`：锁定但仍在保持获取期间

### 操作模式

DPLL 操作模式决定如何选择参考源：

- `DPLL_MODE_MANUAL`（手动模式）：用户显式选择参考源
- `DPLL_MODE_AUTOMATIC`（自动模式）：DPLL 根据优先级和信号质量自动选择

---

## 子系统 API 使用

### DPLL 设备注册

驱动通过下列 API 注册和注销 DPLL 设备：

```c
struct dpll_device *dpll_device_alloc(struct device *parent, u32 index,
                                       const struct dpll_device_ops *ops,
                                       void *priv);
int dpll_device_register(struct dpll_device *dpll);
void dpll_device_unregister(struct dpll_device *dpll);
```

- `dpll_device_alloc()` 分配 DPLL 设备；`ops` 不得为 NULL
- `dpll_device_register()` 向系统注册设备，使其对用户空间可见
- `priv` 指向驱动私有数据，通过 `dpll_priv()` 访问

### DPLL 引脚管理

驱动管理 DPLL 引脚的配置：

```c
struct dpll_pin *dpll_pin_alloc(struct device *dev, u32 index,
                                 const struct dpll_pin_ops *ops,
                                 void *priv);
int dpll_pin_register(struct dpll_device *dpll, struct dpll_pin *pin);
void dpll_pin_unregister(struct dpll_device *dpll, struct dpll_pin *pin);
```

### 锁状态通知

驱动通过以下函数通知用户空间状态变化：

```c
void dpll_device_lock_status_notify(struct dpll_device *dpll);
void dpll_pin_source_notify(struct dpll_pin *pin);
```

---

## Netlink 用户空间接口

DPLL 使用通用 netlink 家族（`DPLL_FAMILY_NAME`）。关键属性：

| 属性 | 含义 |
|------|------|
| `DPLL_A_DEVICE_INDEX` | DPLL 设备索引 |
| `DPLL_A_LOCK_STATUS` | 锁定状态（enum） |
| `DPLL_A_MODE` | 操作模式（enum） |
| `DPLL_A_PIN_INDEX` | 引脚索引 |
| `DPLL_A_PIN_TYPE` | 引脚类型（input/output） |
| `DPLL_A_PIN_FREQUENCY` | 引脚频率（Hz） |
| `DPLL_A_PIN_PRIORITY` | 引脚优先级 |
| `DPLL_A_SOURCE_ID` | 当前选中的源引脚 ID |

---

## Invariants（不变规则）

- DPLL 设备 ops 回调必须提供 `lock_status_get` 和 `mode_get`——它们是 netlink dump 路径必需的
- `dpll_device_alloc()` 的 `ops` 参数不能为 NULL
- 驱动私有数据通过 `dpll_priv()` 访问，在 `dpll_device_alloc()` 时设置
- 状态变化后必须调用对应的 notify 函数，否则用户空间无法感知变化
- DPLL 引脚注册必须在对应的 DPLL 设备注册之后或同时进行

## Failure Modes（失败模式）

- `ops` 中必需的 get 回调缺失 → 用户空间 netlink dump 失败或返回空数据
- 注册后未调用 notify → 用户空间状态不一致
- mode_set 或 source_set 的回调实现错误 → 选中不正确的参考源，时钟同步失败
- 引脚频率超出硬件范围但未返回错误 → 设备可能以预期外的配置运行
- `dpll_device_register()` 失败后仍访问设备 → use-after-free

## Quick Checks（快速检查要点）

- DPLL ops 是否实现了所有必需的 get 回调（`lock_status_get`、`mode_get`）？
- `dpll_device_register()` 或 `dpll_pin_register()` 的返回值是否被检查？
- 状态变化后是否调用了对应的 notify 函数？
- mode_set 实现是否验证了请求的模式是设备支持的？
- 引脚优先级和频率的配置是否在硬件能力范围内？
- DPLL 设备注销前是否确保没有用户空间正在访问？需要使用适当的同步机制（引用计数或锁）。
