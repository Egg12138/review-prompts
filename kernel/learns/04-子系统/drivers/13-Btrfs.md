<!-- Source: subsystem/btrfs.md -->

# Btrfs（B-tree 文件系统）子系统

## Extent Map 字段详解

### 概述

`struct extent_map`（定义于 `fs/btrfs/extent_map.h`）将文件偏移（file offset）映射到磁盘上的物理位置。使用错误的字段进行大小计算或 I/O 会导致静默的数据损坏（data corruption）、过度读取（over-reads）或超量分配（oversized allocations）。

**为什么这些错误难以在测试中发现**：对于简单的未压缩（uncompressed）、非 reflink 的 extent，`len`、`ram_bytes` 和 `disk_num_bytes` 三个字段的值都相等。这些字段只有在涉及压缩或部分引用（reflinks、bookend extents）时才会分岔。

### Extent Map 字段表

| 字段 | 含义 | 对应 on-disk 字段 |
|------|------|------------------|
| `start` | 文件偏移量 | 匹配 `BTRFS_EXTENT_DATA_KEY` 的 key offset |
| `len` | 该 extent map 覆盖的文件字节数 | on-disk 的 `num_bytes`；对于 inline extent 始终为 sector 大小 |
| `disk_bytenr` | 磁盘上的物理字节地址 | `EXTENT_MAP_HOLE` / `EXTENT_MAP_INLINE` 为 sentinel 值 |
| `disk_num_bytes` | 磁盘分配的完整大小 | 压缩时等于压缩后大小 |
| `offset` | 该文件范围在解压后 extent 中的偏移量 | reflink/clone 部分引用时非零 |
| `ram_bytes` | 整个磁盘 extent 的解压后大小 | 未压缩时等于 `disk_num_bytes` |

### 未压缩 vs 压缩布局

```
未压缩：
  磁盘:       [disk_bytenr .................. disk_bytenr + disk_num_bytes]
                              |<-- offset -->|<------- len ------->|
  文件:                                       [start ... start + len]
  ram_bytes == disk_num_bytes

压缩：
  磁盘:       [disk_bytenr ... disk_bytenr + disk_num_bytes]  (更小)
  解压后:     [0 ................................ ram_bytes]
                  |<-- offset -->|<------- len ------->|
  文件:                              [start ... start + len]
  ram_bytes > disk_num_bytes
```

### 计算辅助函数

原有的 `block_start`、`block_len`、`orig_block_len` 和 `orig_start` 字段已被移除。替代方案：

| 函数 | 作用 | 返回值 |
|------|------|--------|
| `btrfs_extent_map_block_start(em)` | 获取 I/O 的物理磁盘起始位置 | 未压缩：`disk_bytenr + offset`；压缩：`disk_bytenr` |
| `extent_map_block_len(em)` | 获取磁盘块长度（文件私有，位于 `extent_map.c`） | 未压缩：`len`；压缩：`disk_num_bytes` |

外部调用者需要块长度时必须自行计算：对压缩 extent（`btrfs_extent_map_is_compressed(em)`）使用 `disk_num_bytes`，对未压缩 extent 使用 `len`。

---

## 字段混淆模式（Field Confusion Patterns）

### 常见混淆及其后果

| 混淆 | 后果 |
|------|------|
| `len` vs `ram_bytes` | `ram_bytes` 是整个解压后 extent 的大小，可能远大于 `len`（文件范围）。将 `ram_bytes` 误用作 `len` 会过度读取或超量分配 |
| `len` vs `disk_num_bytes` | 压缩 extent 的 `disk_num_bytes` 小于 `len`。用 `len` 做磁盘 I/O 会读取超出 extent 的范围；用 `disk_num_bytes` 做文件级大小计算会截断数据 |
| `disk_bytenr` vs `btrfs_extent_map_block_start()` | 原始 `disk_bytenr` 是整个磁盘 extent 的起始位置。对于未压缩的部分引用，实际数据起始位置是 `disk_bytenr + offset` |

### 正确字段速查表

| 意图 | 正确字段 | 常见错误 |
|------|---------|---------|
| 该 extent 覆盖的文件范围 | `len` | `ram_bytes` |
| 磁盘上需要读/写的字节数 | `disk_num_bytes` | `len` |
| 解压后 extent 的大小 | `ram_bytes` | `disk_num_bytes` |
| I/O 的物理磁盘位置 | `btrfs_extent_map_block_start()` | 原始的 `disk_bytenr` |

### Invariants（来自 `validate_extent_map()`）

对于真实数据 extent（`disk_bytenr < EXTENT_MAP_LAST_BYTE`）：

- `disk_num_bytes != 0`
- `offset + len <= ram_bytes`
- 未压缩：`offset + len <= disk_num_bytes`
- 未压缩：`ram_bytes == disk_num_bytes`

对于空洞（hole）/内联（inline）（`disk_bytenr >= EXTENT_MAP_LAST_BYTE`）：

- `offset == 0`

---

## Zoned Storage（分区存储）：Active vs Open Zone 限制

### 概念区分

在 ZNS（Zoned Namespace）/ 分区块设备模型中，active zone（活跃分区）和 open zone（开放分区）是语义不同的概念：

| 概念 | API | 含义 |
|------|-----|------|
| **Active zones（活跃分区）** | `bdev_max_active_zones()` | 隐式打开、显式打开或关闭中的分区 —— 消耗设备活跃资源的 zone |
| **Open zones（开放分区）** | `bdev_max_open_zones()` | 当前为写入操作打开的分区（active zones 的子集） |

### 常见错误

将 `bdev_max_active_zones()` 与 `bdev_max_open_zones()` 混淆，或在挂载验证时合成分区域限值而没有逃生通道（escape hatch），导致先前有效的文件系统挂载失败。

### 合成限值的正确做法

```c
// 错误：对现有有效文件系统导致挂载失败
max_active_zones = min_not_zero(bdev_max_active_zones(bdev),
                                bdev_max_open_zones(bdev));
if (nactive > max_active_zones)
    return -EIO;

// 正确：当活跃限值是由 open 合成时提供逃生通道
if (nactive > max_active_zones) {
    if (bdev_max_active_zones(bdev) == 0) {
        max_active_zones = 0;  // 清除合成限值
        goto validate;         // 允许挂载继续
    }
    return -EIO;  // 仅在设备有真实限值时失败
}
```

### 关键不变式

- 从多个来源合成 zone limits 的代码必须提供逃生路径：当底层设备报告无限值时应允许挂载
- Open zones 是 active zones 的子集，不能作为 active zone limits 的有效代理
- 设备可能报告无 active zone 限值（`bdev_max_active_zones() == 0`）但仍存在 open zone 限值

---

## Failure Modes

| 违反场景 | 结果 |
|---------|------|
| 使用 `ram_bytes` 代替 `len` 做文件级大小计算 | 过度读取、超量分配 |
| 使用 `len` 代替 `disk_num_bytes` 做磁盘 I/O | 压缩 extent 读取超出实际范围 |
| 使用 `disk_num_bytes` 做文件级大小计算 | 压缩 extent 数据截断 |
| 使用原始 `disk_bytenr` 定位 I/O | 未压缩的部分引用读取偏移错误的数据 |
| 合成 zone limit 时无逃生路径 | 有效的文件系统无法挂载（-EIO） |
| 将 open zone limit 直接用作 active zone limit | 挂载验证拒绝符合硬件约束的文件系统 |

## Quick Checks

- **Extent map field intent**：检查对 `len` / `ram_bytes` / `disk_num_bytes` 的使用是否匹配实际意图（文件范围 vs 磁盘大小 vs 解压后大小）
- **I/O 物理位置**：是否使用了 `btrfs_extent_map_block_start()` 而非原始 `disk_bytenr`？
- **压缩 extent 处理**：对于压缩 extent，I/O 大小是否使用 `disk_num_bytes` 而非 `len`？
- **部分引用偏移**：对于 reflink/clone 的 extent，`offset` 字段是否被正确考虑？
- **Zone limit 合成**：在 `btrfs_get_dev_zone_info()` 中合成 zone limit 时是否有逃生路径？
- **validate_extent_map invariants**：`offset + len <= ram_bytes` 不变式是否被代码逻辑无意中违反？
