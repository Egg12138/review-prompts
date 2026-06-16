<!-- Source: subsystem/pmdomain.md -->

# Power Domain（电源域）子系统

## 概述

Linux genpd（Generic Power Domain，通用电源域）框架管理设备的电源开关。本文聚焦 genpd 的 stay_on 机制、同步状态回调、电源关闭顺序等关键不变式和故障模式。

---

## genpd stay_on 与 sync_state 的交互

### 不变式

在 `pm_genpd_init()` 时已经上电的域（`is_off=false`）将**无限期保持上电**直到 `sync_state` 触发，除非设置了 `GENPD_FLAG_NO_STAY_ON`。如果消费者设备从未 probe，`sync_state` 在默认严格模式下永不触发，导致域永久上电——浪费电力并可能阻碍 regulator（稳压器）的清理。

### 行为流程

1. 当 `CONFIG_PM_GENERIC_DOMAINS_OF` 启用且域初始化为 on（`is_off=false`）时，`pm_genpd_init()` 调用 `genpd_set_stay_on()`，设置 `genpd->stay_on = true`（除非设置了 `GENPD_FLAG_NO_STAY_ON`）
2. 当 `stay_on` 为 true 时，`genpd_power_off()` 拒绝关闭该域（在该函数顶部检查）
3. `stay_on` 标志在 provider 的 `sync_state` 回调运行时清除
4. 对于基于 OF（Device Tree）的 provider，由 `genpd_provider_sync_state()` 或 `of_genpd_sync_state()` 处理——它们设置 `genpd->stay_on = false` 然后尝试 `genpd_power_off()`

### sync_state 触发条件

`sync_state` 回调**仅在所有消费者设备链接达到 `DL_STATE_ACTIVE`**（即所有消费者都 probe 成功）后调用。如果某个消费者从未 probe（没有驱动、无限期 deferred 等），sync_state 在严格模式下永不触发。

### 超时机制

`fw_devlink.sync_state=timeout` 选项（或 `CONFIG_FW_DEVLINK_SYNC_STATE_TIMEOUT`）改变此行为：在 `deferred_probe_timeout` 到期或（`!CONFIG_MODULES` 时）在 `late_initcall()` 时放弃等待。

### GENPD_FLAG_NO_STAY_ON

此标志**阻止**设置 `stay_on`，允许域在没有活跃消费者时立即断电——无需等待 `sync_state`。使用此标志的平台包括：

- Renesas R-Car：`drivers/pmdomain/renesas/rcar-sysc.c`
- Renesas R-Mobile：`drivers/pmdomain/renesas/rmobile-sysc.c`
- Rockchip：`drivers/pmdomain/rockchip/pm-domains.c`
- Tegra BPMP：`drivers/pmdomain/tegra/powergate-bpmp.c`

### 不使用 CONFIG_PM_GENERIC_DOMAINS_OF 时

如果没有 `CONFIG_PM_GENERIC_DOMAINS_OF`，`genpd_set_stay_on()` 无条件设置 `stay_on = false`，因此 stay_on 机制不活跃。

### 故障模式

- 消费者从未 probe 导致域永久上电，浪费电力
- 域保持上电阻塞 regulator 的清理
- 使用了 `of_genpd_sync_state()` 作为 `sync_state` 回调，但 provider 管理的并非所有域都应该被断电

---

## genpd_power_off_unused 与 Regulator 清理顺序

### 时序

| 事件 | 时机 | 行为 |
|------|------|------|
| `genpd_power_off_unused()` | `late_initcall_sync` | 遍历所有注册的 genpd，对无活跃消费者的域异步断电。`stay_on == true` 的域被跳过 |
| `regulator_init_complete()` | `late_initcall_sync` | 不立即禁用 regulator。调度 `regulator_init_complete_work`（延迟工作，30 秒超时），最终调用 `regulator_late_cleanup()` 禁用未使用的 regulator |

### 故障模式

如果 `genpd_power_off_unused()` 被延迟（例如因 stay_on 机制），域在 regulator 预期其断电的时间点之后仍保持上电。在 regulator 为电源域供电的平台上，这可能导致 regulator 在域仍活跃时被禁用，造成硬件故障。

### 典型场景

即使有 30 秒延迟，如果域因等待 `sync_state` 回调而通过 `stay_on` 保持上电且回调永不到达，域可能在 regulator 最终被禁用时仍处于上电状态。Rockchip 电源域驱动设置 `GENPD_FLAG_NO_STAY_ON` 正是为了避免此场景。

---

## 平台特定的变通方法条件

### 不变式

无条件应用变通方法会导致在不受影响的平台上执行不必要的代码，并可能在不需要变通方法的硬件上引入回归。

### 变通方法门控条件

针对 bootloader 状态移交（如 splash-screen handover）的变通方法必须限制在受影响的平台上：

- 架构检查（如 `IS_ENABLED(CONFIG_ARM)` 用于 ARM32 专用问题）
- 设备兼容性检查（`of_device_is_compatible()`）
- 平台特定的 Device Tree 属性

### 时序注意事项

电源域重置操作应在早期 probe、`pm_genpd_init()` 初始化域**之前**进行。`pm_genpd_init()` 后域已注册到全局 `gpd_list` 并由 genpd 框架管理，手动重置可能与框架状态冲突。

### 推荐做法

针对性的重置应使用显式的每驱动断电函数（如 `exynos_pd_power_off()` 在 `drivers/pmdomain/samsung/exynos-pm-domains.c`），而非 `of_genpd_sync_state()`——后者遍历 provider 的所有域并尝试对每个断电。

---

## 平台默认域状态

### 不变式

强制域在启动时上电而它们本应默认为 off，会浪费电力并可能违反硬件约束。

### 各厂商的表达机制

| 厂商 | 机制 | 文件位置 |
|------|------|---------|
| MediaTek | `MTK_SCPD_KEEP_DEFAULT_OFF` | `drivers/pmdomain/mediatek/mtk-pm-domains.h` |
| Renesas / Rockchip | `GENPD_FLAG_NO_STAY_ON` | 允许在无需等待 sync_state 时断电 |

MediaTek 的做法：当设置 `MTK_SCPD_KEEP_DEFAULT_OFF` 标志后，通过 `is_off=true` 调用 `pm_genpd_init()` 将域初始化为 off。

### 审查要点

修改默认 on/off 行为的核心 genpd 变更必须在依赖这些机制的平台驱动上进行验证。

---

## 快速检查清单

- **`of_genpd_sync_state()` 作为 sync_state 回调**：此函数遍历 provider 的所有域并对每个断电。如果平台驱动将其用作 `sync_state` 回调，验证这对 provider 管理的所有域是否都合适，还是应该只关闭特定域。
- **新驱动上的 `GENPD_FLAG_NO_STAY_ON`**：具有 regulator 供电的电源域或 `sync_state` 调用不可靠的平台应设置此标志。
- **核心 genpd 时序变更**：延迟或阻止域断电的新默认行为必须提供 `GENPD_FLAG_*` opt-out（退出选项），供无法容忍该变更的平台使用。
