<!-- Source: subsystem/ata.md -->

# ATA 子系统详解

## 设备验证和兼容性（Device Validation and Compatibility）

在设备初始化、日志读取或能力检测路径中的严格验证检查，可能会破坏之前正常工作设备的兼容性，导致功能被错误禁用或设备初始化失败。ATA 设备经常以不影响功能的方式偏离 ACS 规范。

### ATA 规格合规的现实（ATA Specification Compliance Reality）

ATA 设备普遍存在不影响功能的规格偏差：
- 版本字段报告 `0x0000` 而非规格定义的值（例如，通用目的日志目录版本，预期为 `0x0001`）
- 保留位（reserved bits）不为零
- 日志页面格式略有不同
- 可选字段缺失或填充为零

### 验证严格度原则（Validation Strictness Principles）

审查在设备初始化路径中添加新验证的补丁时（如 `ata_dev_configure()`、`ata_read_log_directory()`、日志页面解析）：

- **严格验证需要理由**：可能拒绝之前正常工作设备的新检查，需要明确讨论为什么需要严格实施以及哪些设备可能受影响
- **失败前先警告**：对于不影响数据完整性的规格合规性检查，更安全的模式是使用 `ata_dev_warn()` 或 `ata_dev_warn_once()` 发出警告，而非返回错误
- **避免永久禁用功能**：在验证失败时，以编程方式设置 quirk（如 `ATA_QUIRK_NO_LOG_DIR`）或通过 `ata_clear_log_directory()` 清除缓存数据，可能会对本来可以工作的设备禁用功能

### 何时严格失败 vs 发出警告

| 条件 | 操作 |
|------|------|
| 数据完整性问题（无效校验和、结构损坏） | 失败返回错误 |
| 读取设备数据的 I/O 错误 | 失败返回错误 |
| 安全关键功能（NCQ、TRIM、security）配置无效 | 失败返回错误 |
| 有效结构上的版本不匹配 | 警告并继续 |
| 可选字段未按规格格式化 | 警告并继续 |
| 保留位非零 | 忽略或警告 |

```c
// 错误：严格的版本检查直接禁用功能，无回退
if (version != EXPECTED_VERSION) {
    ata_dev_err(dev, "Invalid version 0x%04x", version);
    ata_clear_log_directory(dev);
    dev->quirks |= ATA_QUIRK_NO_LOG_DIR;
    return -EINVAL;
}
```

```c
// 正确：对不合规发出警告但继续运行
if (version != EXPECTED_VERSION)
    ata_dev_warn_once(dev, "Unexpected version 0x%04x", version);
// 如果数据看起来有效则继续使用
```

## Invariants（不变规则）

- 新的验证不能破坏已有工作设备的兼容性。严格检查需要明确的理由论证。
- 警告优于粗暴拒绝：对不影响数据完整性的 spec 偏差先 `ata_dev_warn_once()` 再继续。
- 避免因验证失败而编程设置 `ATA_QUIRK_*` 或调用 `ata_clear_log_directory()`，这可能导致永久禁用本可工作的设备。

## Failure Modes（失败模式）

- 严格的版本检查导致设备初始化失败 → 之前工作的设备变成"不支持"。
- 验证失败时编程设置 `ATA_QUIRK_NO_LOG_DIR` → 永久禁用日志功能。
- 对非关键（non-critical）规格偏差返回错误而非警告 → 不必要的兼容性破坏。

## Quick Checks（快速检查要点）

- **初始化路径中的新验证**：补丁新增对 `ata_dev_configure()`、`ata_read_log_directory()` 或类似函数的验证时，确认提交信息论证了严格实施的理由，并考虑了与现有设备的兼容性。
- **编程 quirk 设置**：基于运行时验证（而非 `__ata_dev_quirks` 中的静态 quirk 表）设置 `ATA_QUIRK_*` 标志的代码，应有回退或恢复机制。
- **错误与警告的区分**：区分功能性错误（I/O 失败、损坏）和纯粹的 spec 违规（版本不匹配、格式化差异）。
