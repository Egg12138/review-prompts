<!-- Source: subsystem/cxl.md -->

# CXL 子系统详解

## 资源初始化给 HMAT API（Resource Initialization for HMAT APIs）

传递给 HMAT API 的 CXL 内存资源如果缺少 `IORESOURCE_MEM` 类型标志，会静默产生错误结果。`hmat_get_extended_linear_cache_size()` 在 `drivers/acpi/numa/hmat.c` 中调用 `resource_contains()` 来匹配后备资源与 HMAT 内存目标。`resource_contains()` 在 `include/linux/ioport.h` 中当两个资源的 `resource_type()` 不同时返回 `false`。不使用 `IORESOURCE_MEM` 创建的资源类型为 `0`，而 HMAT 目标资源类型为 `IORESOURCE_MEM`，因此每次比较都失败。该函数随后返回 `0`（成功）且 `*cache_size = 0`，然后 `cxl_acpi_set_cache_size()` 在 `drivers/cxl/acpi.c` 中存储这个零值。可见的影响是：带有扩展线性缓存的 CXL 区域报告其实际大小的一半，而 MCE 内存错误报告使用错误的偏移量。

## Invariants（不变规则）

- CXL HPA（Host Physical Address，主机物理地址）范围必须使用 `IORESOURCE_MEM` 类型标志创建 `struct resource`
- 使用 `DEFINE_RES_MEM(start, size)` 而非 `DEFINE_RES(start, size, 0)`——后者创建类型为 `0` 的资源，会静默导致 `resource_contains()` 检查失败

```c
// 错误：缺少 IORESOURCE_MEM 标志，cache_size 静默设置为 0
struct resource res = DEFINE_RES(start, size, 0);
rc = hmat_get_extended_linear_cache_size(&res, nid, &cache_size);

// 正确：正确类型化的内存资源
struct resource res = DEFINE_RES_MEM(start, size);
rc = hmat_get_extended_linear_cache_size(&res, nid, &cache_size);
```

## Failure Modes（失败模式）

- 用 `DEFINE_RES()` 且 flags=0 创建资源 → 类型为 0，与 `IORESOURCE_MEM` 的 `resource_contains()` 比较失败。
- `hmat_get_extended_linear_cache_size()` 返回成功（0）但 `*cache_size = 0` → 调用者认为有有效的缓存大小，实际为零。
- CXL 区域报告缓存大小为实际大小的一半 → 内存管理错误。
- MCE 内存错误报告使用错误的偏移量 → 错误的错误定位。

## Quick Checks（快速检查要点）

- **物理内存的资源类型**：为 CXL 内存范围创建 `struct resource` 时，确认使用 `DEFINE_RES_MEM()` 而非带显式标志的 `DEFINE_RES()`。
