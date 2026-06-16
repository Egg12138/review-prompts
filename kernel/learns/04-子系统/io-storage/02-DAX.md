<!-- Source: subsystem/dax.md -->

# DAX（Direct Access）子系统详解

## 概述

DAX（Direct Access）允许应用程序和文件系统直接访问持久内存（persistent memory，持久性内存）设备，绕过传统页面缓存（page cache）。DAX 映射的页面错误处理方式与标准页面缓存不同。

## DAX 映射要求（DAX Mapping Requirements）

- **CPU 可寻址持久内存**：DAX 将持久内存映射到 CPU 地址空间，允许直接 load/store 访问。
- **页面错误处理**：DAX 的 page fault（缺页中断）处理与页面缓存不同——它直接映射设备的物理页面，而非从块设备读取填充缓存页。
- **部分映射无 struct page**：某些 DAX 映射**没有** `struct page` 结构体。这意味着在无 `struct page` 的映射上使用需要 page 结构的 API（如 `kmap()`、`virt_to_page()`）会导致错误。
- **缓存刷写（Cache Flushing）保证持久性**：对 DAX 映射的 CPU 写入在到达持久内存介质之前可能停留在 CPU 缓存中。必须通过显式缓存刷写指令（如 `clwb`、`clflushopt`）或架构提供的持久化刷写原语来保证持久性。

## 同步机制（Synchronization）

- **DAX 条目在 radix tree 中需要加锁**：DAX 使用 radix tree（基数树）或 xarray 来管理映射条目，并发访问需要通过适当的锁保护。
- **大页（2MB / 1GB）的特殊处理**：DAX 支持 huge page（大页），这些大页的映射、分裂（split）和合并（collapse）需要特殊处理。
- **与文件系统的截断/打孔协调**：DAX 操作必须与文件系统的 truncate（截断）和 punch-hole（打孔）操作协调，确保元数据和 DAX 映射的一致性。

## Invariants（不变规则）

- 对 DAX 映射的 CPU 写入后必须执行缓存刷写以确保持久性。
- DAX entries（条目）的 radix tree / xarray 操作必须持有相应锁。
- huge page 的分裂和合并操作必须正确管理引用计数。
- 文件系统元数据必须在 DAX 数据操作期间保持一致。

## Failure Modes（失败模式）

- 在无 `struct page` 的 DAX 映射上调用 `kmap()` 或 `virt_to_page()` → 崩溃。
- 未先刷写 CPU 缓存就认为数据已持久化 → 掉电时数据丢失。
- DAX entry 锁缺失 → 并发的截断和映射读取导致数据损坏。
- 大页分裂/合并未正确处理 → 映射损坏或内核 panic。
- 文件系统元数据与 DAX 数据操作不同步 → 系统崩溃后元数据不一致。

## Quick Checks（快速检查要点）

- CPU 写入后是否正确执行缓存刷写？
- DAX entry 锁是否正确保护并发访问？
- huge page 的分裂/合并是否处理正确？
- 文件系统元数据与 DAX 操作是否一致？
- 是否在无 `struct page` 的映射上使用了需要 page 结构的 API？
