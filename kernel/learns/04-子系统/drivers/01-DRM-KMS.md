<!-- Source: subsystem/drm.md -->

# DRM/KMS 显示子系统

## 概述

DRM (Direct Rendering Manager) 和 KMS (Kernel Mode Setting) 是 Linux 显示子系统的核心框架。本文覆盖 DRM 显示驱动开发中的关键 invariants（不变式）、常见错误模式和审查要点，包括 atomic 上下文约束、电源管理分离、GPU 虚拟地址管理、寄存器访问协议等主题。

---

## 一、Atomic 上下文中的显示硬件编程

### 不变式

在 atomic 上下文中**禁止调用可能导致休眠的函数**。DRM/KMS 显示驱动中存在多条在 atomic 上下文中执行的代码路径，在这些路径中休眠会导致内核警告、系统不稳定和潜在的死锁。

### Atomic 上下文路径

以下回调函数在 atomic 上下文中执行：

- `drm_atomic_helper_commit_tail()` 及其变体
- CRTC（显示器控制器）的 atomic enable/disable/update 回调
- Plane（显示层）的 atomic update 回调
- Encoder（编码器）的 atomic enable/disable 回调
- VBLANK（垂直消隐）中断处理函数和回调
- Page flip（页面翻转）完成处理函数
- Hardware sequencer（硬件序列器，hwseq）函数——这些函数在 AMD 显示驱动中位于 `hwseq` / `hw_sequencer` 目录，在其他厂商驱动中也有类似模式。它们实现底层显示硬件编程，许多被 atomic commit 路径调用，因此**禁止休眠**。

### 延迟函数速查表

| 函数 | 能否休眠 | 能否在 Atomic 上下文使用 |
|------|----------|------------------------|
| `udelay()` | 否 | **安全** |
| `ndelay()` | 否 | **安全** |
| `mdelay()` | 否 | **安全**（但长延迟不推荐） |
| `fsleep()` | **是** | **不安全** |
| `msleep()` | **是** | **不安全** |
| `usleep_range()` | **是** | **不安全** |

### 故障模式

- **在 display 驱动中引入延迟的补丁**：必须检查调用上下文。如果函数可能从 atomic commit 路径、VBLANK 处理函数或任何持有 spinlock 的代码段到达，只能使用非休眠延迟函数（`udelay`、`ndelay`）。
- **用固定延迟替换硬件轮询函数**：例如用 `fsleep()` 替换 `wait_for_blank_complete()`。原始轮询函数内部可能使用 busy-wait（忙等待）以保持 atomic 安全，替换为可休眠函数破坏了这一不变式。

### 审查检查点

- `fsleep`、`msleep`、`usleep_range`、`mutex_lock`、`GFP_KERNEL` 分配是否出现在 hwseq 函数或 atomic 回调中？
- CRTC、plane、encoder 的 atomic 回调在 non-blocking commit 模式下运行于 atomic 上下文。
- VBLANK 和 page flip 处理函数始终是 atomic 上下文，禁止休眠。

---

## 二、电源管理上下文分离（XE 驱动）

### 不变式

在系统 PM（电源管理）路径中使用运行时 PM 的标记位会导致系统唤醒后硬件功能异常——设备看似唤醒，但 I2C 控制器、显示引擎等组件因重新初始化被错误跳过而无法工作。

### 系统 PM vs 运行时 PM

| 特性 | 系统 PM | 运行时 PM |
|------|---------|----------|
| 对应函数 | `xe_pm_suspend()` / `xe_pm_resume()` | `xe_pm_runtime_suspend()` / `xe_pm_runtime_resume()` |
| 场景 | S3/suspend-to-RAM 等深度睡眠 | D-state 转换，正常运行中 |
| 是否掉电 | **始终掉电** | 取决于平台和 `xe->d3cold.allowed` 标志 |
| 是否需要完全重新初始化 | **总是需要** | 视情况而定 |

### 故障模式

系统 PM 函数中**不得使用**运行时 PM 标志（如 `xe->d3cold.allowed`）来决定重新初始化的深度。系统挂起总是导致掉电，因此总是需要完全重新初始化。

```c
// 错误：在系统 PM 中使用运行时 PM 标志
xe_i2c_pm_resume(xe, xe->d3cold.allowed);

// 正确：系统 PM 总是需要完全重新初始化
xe_i2c_pm_resume(xe, true);
```

### 需要警惕的字段

以下字段表示运行时 PM 上下文，**不得在系统 PM 路径中使用**：
- `xe->d3cold.allowed`
- `xe->d3cold.capable`
- `struct dev_pm_info` 中的运行时 PM 控制标志

### 审查检查点

- 如果在 `xe_pm_resume()` 或 `xe_pm_suspend()` 中看到 `d3cold.allowed` 被用于条件行为，验证这是否是故意的（通常不是）。

---

## 三、XE 驱动 GT 访问器 API 契约

### 不变式

对 `xe_device_get_gt()` 的返回值进行 NULL 检查是必需的。该函数对于无效的 GT（Graphics Tile）索引或不存在的 GT 返回 NULL，直接解引用会导致 NULL 指针崩溃。

### GT 访问器函数

| 函数 | 可返回 NULL | 用途 |
|------|------------|------|
| `xe_device_get_gt(xe, gt_id)` | **是** | 按索引查找 GT；调用者必须检查 NULL |
| `xe_root_mmio_gt(xe)` | **否** | 返回根 tile 的主 GT；用于非 GT 的 MMIO 操作 |

### 使用规则

- 当 GT 索引来自用户输入或配置、且 NULL 是合法结果时，使用 `xe_device_get_gt()`
- 当需要访问根 tile 的主 GT、且 NULL 应被视为 bug 时（如默认值、初始化路径），使用 `xe_root_mmio_gt()`

```c
// 错误：直接解引用，不做 NULL 检查
param->oa_unit = &xe_device_get_gt(oa->xe, 0)->oa.oa_unit[0];

// 正确：需要 GT 0 时使用 xe_root_mmio_gt()
param->oa_unit = &xe_root_mmio_gt(oa->xe)->oa.oa_unit[0];

// 正确：使用 xe_device_get_gt() 时检查 NULL
gt = xe_device_get_gt(xe, id);
if (!gt)
    return -EINVAL;
```

### 故障模式

直接解引用 `xe_device_get_gt()` 的返回值（如 `xe_device_get_gt(...)->field`）而**没有前置 NULL 检查**，应报告为 bug。

---

## 四、DRM GPU VM（drm_gpuvm）IOMMU 要求

### 不变式

`drm_gpuvm` 框架**要求 IOMMU 支持**。它不支持物理地址回退模式。当 IOMMU 不可用时返回 NULL 而不是错误，允许驱动初始化在不支持的配置下继续，会导致后续对 NULL VM 的操作产生未定义行为。

### 审查要点：drm_gpuvm 转换

- 查找现有的"无 IOMMU 回退"路径：检查 `if (!mmu)` 或 `if (!vm)` 并继续以 `vm = NULL` 而不是失败的代码
- 这些 NULL 回退返回必须改为 `ERR_PTR(-ENODEV)` 或类似的错误返回
- 检查 GPU 和显示（KMS/MDP）代码中的所有初始化路径

```c
// 旧模式（支持无 IOMMU 回退）-- 使用 drm_gpuvm 时错误
mmu = create_mmu(...);
if (!mmu) {
    drm_info(dev, "no IOMMU, fallback to phys contig buffers\n");
    return NULL;
}

// 使用 drm_gpuvm 时的正确写法
mmu = create_mmu(...);
if (!mmu) {
    drm_info(dev, "no IOMMU, bailing out\n");
    return ERR_PTR(-ENODEV);
}
```

### 常见位置

- `msm_kms_init_vm()` 在 `drivers/gpu/drm/msm/msm_kms.c`
- `mdp4_kms_init()` 在 `drivers/gpu/drm/msm/disp/mdp4/mdp4_kms.c`
- 其他显示控制器初始化函数中的类似模式

### 故障模式

在 drm_gpuvm 转换中，任何 IOMMU/MMU 不可用时返回 NULL 或设置 `vm = NULL` 的代码路径，**必须改为返回错误**。

---

## 五、DRM Scheduler KUnit 测试 Flag 语义

### 不变式

等待超时处理函数执行的测试中，如果等待时间跨越多个超时周期，超时处理函数可能被多次调用。在处理函数中清除控制标志会在后续调用中产生竞态条件。

### 控制标志 vs 状态标志

| 类型 | 示例 | 行为 |
|------|------|------|
| **控制标志** | `DRM_MOCK_SCHED_JOB_DONT_RESET` | 决定处理函数走哪条路径。**必须在 job 生命周期内持久存在**，确保多次调用时行为一致 |
| **状态标志** | `DRM_MOCK_SCHED_JOB_RESET_SKIPPED` | 记录特定事件已发生。事件发生时设置，**永不清除** |

```c
// 错误：清除控制标志在重执行时产生竞态
if (job->flags & DRM_MOCK_SCHED_JOB_DONT_RESET) {
    job->flags &= ~DRM_MOCK_SCHED_JOB_DONT_RESET;  // 清除了！
    return DRM_GPU_SCHED_STAT_NO_HANG;
}
// 第二次调用：标志已清，走不同路径

// 正确：控制标志保持，状态标志记录执行
if (job->flags & DRM_MOCK_SCHED_JOB_DONT_RESET) {
    job->flags |= DRM_MOCK_SCHED_JOB_RESET_SKIPPED;  // 记录事件
    return DRM_GPU_SCHED_STAT_NO_HANG;
}
// 第二次调用：行为相同，状态标志已置位
```

### 故障模式

在超时/完成/中断处理函数 mock 中修改控制标志的测试代码。在等待时间较长（如 `2 * MOCK_TIMEOUT`）时，处理函数可能触发多次，清除控制标志会导致不同行为。

---

## 六、XE GuC CT 调试基础设施初始化

### 不变式

调用 `stack_depot_save()` 等内核基础设施 API 前必须完成相应初始化。缺少初始化会导致 NULL 指针解引用。

### 初始化阶段

- `xe_guc_ct_init_noalloc()`：早期初始化（无分配），调试基础设施必须在此初始化
- `xe_guc_ct_init()`：资源分配阶段

```c
// 正确：xe_guc_ct_init_noalloc() 必须包含：
#if IS_ENABLED(CONFIG_DRM_XE_DEBUG_GUC)
    stack_depot_init();
#endif
```

### 故障模式

在 debug 配置选项下添加内核基础设施 API（如 `stack_depot_save()`）的调用，但未验证相应初始化函数在同一配置选项保护下被调用。

### 需要显式初始化的基础设施

- `stack_depot`：需要先调用 `stack_depot_init()` 才能调用 `stack_depot_save()`

---

## 七、MSM VM 延迟初始化

### 不变式

直接访问 `ctx->vm` 前必须确保 VM 已创建。msm 驱动对虚拟地址空间使用延迟初始化——VM 不是在 DRM context 创建时创建，而是在首次需要时按需创建。

### 访问器模式

| 方式 | 安全性 | 说明 |
|------|--------|------|
| `msm_context_vm(dev, ctx)` | **安全** | 在返回前确保 VM 已创建。ioctl 入口点和早期代码路径应使用此访问器 |
| `ctx->vm` 直接访问 | **不安全** | 在 VM 存在之前使用可能导致 NULL 崩溃 |

```c
// 错误：ctx->vm 可能尚未初始化
if (to_msm_vm(ctx->vm)->unusable)
    return -EPIPE;

// 正确：确保 VM 存在后再访问
if (to_msm_vm(msm_context_vm(dev, ctx))->unusable)
    return -EPIPE;
```

### 风险路径

- ioctl 处理函数的早期阶段（GPU 操作发生前）
- 功能选择加入（如 VM_BIND）后、首次映射操作前
- 在执行实际操作之前验证 context 状态的代码

### 故障模式

在 ioctl 入口点或早期验证代码中，直接访问 `ctx->vm` 或 `to_msm_vm(ctx->vm)` 而没有在同一函数中先调用 `msm_context_vm()`，应报告为 bug。

---

## 八、struct_size() 溢出语义

### 关键知识点

`struct_size()` 在溢出时**不会**返回超过 `SIZE_MAX` 的值，而是**饱和到 `SIZE_MAX`**。因此检查 `if (sz > SIZE_MAX)` 是死代码，永远不触发。

```c
// 错误：sz 不可能大于 SIZE_MAX
u64 sz = struct_size(job, ops, nr_ops);
if (sz > SIZE_MAX)
    return -ENOMEM;  // 死代码，永不执行
```

### 正确写法

```c
// 方式一：检查是否等于 SIZE_MAX
size_t sz = struct_size(ptr, member, count);
if (sz == SIZE_MAX)
    return -ENOMEM;

// 方式二：让 kzalloc 优雅失败（分配 SIZE_MAX 会失败）
ptr = kzalloc(struct_size(ptr, member, count), GFP_KERNEL | __GFP_NOWARN);
if (!ptr)
    return -ENOMEM;
```

### 故障模式

`struct_size()`、`array_size()` 等饱和到 `SIZE_MAX` 的宏后，跟随 `if (sz > SIZE_MAX)` 或 `if (size > SIZE_MAX)` 的检查——这总是死代码。

---

## 九、Intel GPU 平台和子平台架构

### 两级层次结构

Intel GPU 驱动使用两级层次结构：

| 层级 | 定义 | 示例 |
|------|------|------|
| **Platform（平台）** | 主要硬件架构 | TGL、DG2、MTL、PTL |
| **Subplatform（子平台）** | 同一平台内的硬件变体 | 区分不同 PHY 配置、PCH 支持 |

### 子平台注册的必要条件

当硬件共享相同平台架构但存在以下差异时，需要注册子平台：
- 不同的 IP 版本（如同一平台族内不同的 display IP 版本）
- 不同的 PHY 处理、PCH 支持或显示配置
- 现有代码在平台族内检查子平台级别的区分

### 注册模式

- 子平台的 PCI ID 应分离到不同的宏中（如 `INTEL_WCL_IDS` 独立于 `INTEL_PTL_IDS`）
- 平台描述符必须包含将 PCI ID 映射到子平台标识符的子平台注册
- i915 和 xe 驱动需要一致处理子平台

### 涉及文件

- `include/drm/intel/pciids.h`：PCI ID 宏定义
- `drivers/gpu/drm/i915/display/intel_display_device.c`：i915 设备表和含子平台数组的平台描述符
- `drivers/gpu/drm/xe/xe_pci.c`：xe 设备表

### 故障模式

提交将设备 ID 添加到现有平台宏中，但提交信息表明存在硬件差异（不同的 IP 版本、架构变体）却未包含子平台注册。

### 平台标志重载与硬件假设

单个平台标志（如 `display->platform.pantherlake`）可能覆盖多个具有不同配置的硬件变体。代码做出硬件特异性决策时可能在某些变体上静默失败。

**硬件假设验证**：当提交信息或注释声称：
- "Platform X doesn't have feature Y"
- "There will never be a case where..."
- "Extending this should not cause issues for platform Z"

这些是需要验证的**假设**，而非事实：
- VBT 枚举可以枚举端口，即使未连接到预期的 PHY 类型
- Type-C 配置可能使用不同的 PHY 类型（C20 而非 C10）
- 某个变体上"永远不会发生"的边界情况可能在另一个变体上出现

**PHY 类型识别函数**：
`intel_encoder_is_c10phy()` 等函数做出关键决定。审查变更时：
- 验证条件在平台标志覆盖的**所有**变体上都正确
- 追踪函数返回错误值时的下游影响（错误的 PHY 初始化、错误的电源序列、lane 配置失败）
- 检查端口是否可以通过 VBT 在代码假设不存在的配置中被枚举

### 故障模式

代码扩展现有平台特定条件（如 PHY 类型检查）以覆盖更多端口/PHY，但平台标志覆盖多个具有不同硬件配置的变体，除非提交明确处理每个变体的需求或添加子平台区分。

---

## 十、AMDGPU 缓冲对象分配契约

### 不变式

`amdgpu_bo_create_kernel()` 具有双重行为：
- 如果 `*bo_ptr == NULL`：创建并 pin 一个新的缓冲对象
- 如果 `*bo_ptr != NULL`：尝试 pin 现有 BO 在对应地址

```c
// 错误：依赖外部调用者传入已初始化的内存
int wrapper_alloc(void **buf_obj) {
    struct amdgpu_bo **bo = (struct amdgpu_bo **)buf_obj;
    return amdgpu_bo_create_kernel(adev, size, align, domain, bo, ...);
}

// 正确：显式初始化以保证创建新 BO
int wrapper_alloc(void **buf_obj) {
    struct amdgpu_bo **bo = (struct amdgpu_bo **)buf_obj;
    *bo = NULL;  // 强制创建新 BO
    return amdgpu_bo_create_kernel(adev, size, align, domain, bo, ...);
}
```

### 审查要点（AMDGPU wrapper 函数）

- 将 `void **` 转换为 `struct amdgpu_bo **` 并传递给 BO 分配函数的函数
- 在调用 `amdgpu_bo_create_kernel()` 前缺少显式 `*bo = NULL` 初始化
- 声称"分配新 BO"的 API 文档未保证内部指针为 NULL

### 故障模式

导出到外部驱动的 wrapper 函数，将不透明指针参数传递给 `amdgpu_bo_create_kernel()` 但未先初始化 `*bo = NULL`。

---

## 十一、AMDGPU 地址格式 API

### 关键知识点

AMDGPU 驱动中有多个返回不同格式地址的函数。在 debugfs 或诊断接口中使用错误的地址 API 会导致用户空间工具收到格式错误的数据。

| 函数 | 返回 | 用途 |
|------|------|------|
| `amdgpu_bo_gpu_offset()` | GPU 虚拟地址 | GPU 虚拟地址空间内的通用缓冲访问 |
| `amdgpu_gmc_pd_addr()` | 页目录地址 | 硬件寄存器格式，用于页表调试、VM 根地址 |

```c
// 错误：页表调试使用 GPU 虚拟地址格式
seq_printf(m, "address: 0x%llx\n", amdgpu_bo_gpu_offset(vm.root.bo));

// 正确：页表调试使用 PD 地址格式
seq_printf(m, "pd_address: 0x%llx\n", amdgpu_gmc_pd_addr(vm.root.bo));
```

### 故障模式

导出 VM 根或页表地址的 debugfs 文件使用 `amdgpu_bo_gpu_offset()` 而非 `amdgpu_gmc_pd_addr()`。

---

## 十二、XE Pcode 邮箱寄存器更新

### 不变式

直接赋值新值到 pcode 邮箱寄存器而不保留现有字段会破坏硬件配置——电源限制时间窗口、使能位等设置会丢失。

### Read-Modify-Write（RMW）模式

Pcode 邮箱通信传递的寄存器值包含多个打包字段（使能位、值字段、时间窗口配置）。更新一个字段时，其他字段必须通过 RMW 保留。

```c
// 错误：丢失 PWR_LIM_TIME 等字段
ret = xe_pcode_read(tile, mbox, &val0, &val1);
val0 = uval;  // 直接赋值
ret = xe_pcode_write64_timeout(tile, mbox, val0, val1, timeout);

// 正确：保留其他字段
ret = xe_pcode_read(tile, mbox, &val0, &val1);
val0 = (val0 & ~clear_mask) | set_value;  // RMW 模式
ret = xe_pcode_write64_timeout(tile, mbox, val0, val1, timeout);
```

### 怀疑缺少 RMW 的线索

- 名为 `*_write_*` 的函数先调用 `*_read_*`，然后直接赋值给读取的值（不进行掩码修改）
- 同一寄存器存在多个字段定义（多个 `REG_GENMASK` 或 `REG_BIT` 宏覆盖不同位范围）
- Hwmon 或电源管理代码更新配置值

### 故障模式

Pcode 写入序列中，读取的值被直接赋值（`val = new_value`）覆盖，而不是掩码修改（`val = (val & ~mask) | new_value`）。

---

## 十三、AMDGPU 环形缓冲写指针类型

### 不变式

所有 AMDGPU 环形缓冲代码中的写指针（wptr）变量、参数和结构体字段必须是 `u64`，而非 `u32`。类型不匹配在大环形缓冲或写指针超过 32 位范围时导致整数截断。

```c
// 错误：wptr 类型不匹配导致截断
static void func(struct amdgpu_ring *ring, u64 start_wptr, u32 end_wptr)

// 正确：两个 wptr 参数都是 u64
static void func(struct amdgpu_ring *ring, u64 start_wptr, u64 end_wptr)
```

### 已知结构体

- `struct amdgpu_ring`：`wptr` 为 `u64`
- `struct amdgpu_fence`：`wptr` 为 `u64`

### 故障模式

函数签名中部分 wptr 参数为 `u64`、部分为 `u32`，尤其是当两者表示同一环形缓冲上的写入位置时。

---

## 十四、AMDGPU Fence 序列号绕回

### 问题描述

Fence 序列号使用 `uint32_t` 循环计数器，当 `sync_seq` 绕回到小值而最后处理的序列号接近最大值时，直接比较（`i <= sync_seq`）会立即失败，即使存在未处理的 fence。

```c
// 错误：sync_seq 绕回时失败
for (i = seqno + 1; i <= ring->fence_drv.sync_seq; ++i) {
    ptr = &ring->fence_drv.fences[i & ring->fence_drv.num_fences_mask];
    // ...
}
```

场景：`seqno = 0xFFFFFFFF`、`sync_seq = 0x00000005`（已绕回），条件 `i <= 0x00000005` 立即为假，跳过所有 fence。

### 正确写法（masked do-while）

```c
// 正确：通过掩码相等处理绕回
last_seq = amdgpu_fence_read(ring) & ring->fence_drv.num_fences_mask;
seq = ring->fence_drv.sync_seq & ring->fence_drv.num_fences_mask;
do {
    last_seq = (last_seq + 1) & ring->fence_drv.num_fences_mask;
    ptr = &ring->fence_drv.fences[last_seq];
    // ...
} while (last_seq != seq);
```

### 审查要点

- 查找使用 `<=` 或 `<` 比较遍历序列号的循环
- 检查循环索引在数组访问时已掩码化，但在循环条件中未掩码化（不一致的掩码）

### 故障模式

使用直接比较（`i <= sync_seq`）而非掩码相等（`last_seq != seq`）终止的 fence 遍历循环。

---

## 十五、显示缩放模式语义（RMX_*）

### 缩放模式值

| 模式 | 行为 | 宽高比 |
|------|------|--------|
| `RMX_OFF` | 无缩放，仅原生分辨率 | 不变 |
| `RMX_FULL` | 拉伸填满整个屏幕 | **不保留**（图像变形） |
| `RMX_ASPECT` | 在保持比例前提下缩放到最大 | **保留**（添加黑边） |
| `RMX_CENTER` | 居中显示，不缩放 | **保留** |

### 不变式

当驱动自动设置缩放（如 eDP 面板上非原生分辨率且用户空间未明确请求缩放模式时）：
- 应使用 `RMX_ASPECT`，它保留用户内容外观
- `RMX_FULL` 会变形内容，仅应在用户空间明确请求或硬件限制需要时使用

```c
// 错误：作为自动默认值会变形宽高比
if (!scaling_enabled && non_native_resolution)
    dm_new_connector_state->scaling = RMX_FULL;

// 正确：自动缩放时保留宽高比
if (!scaling_enabled && non_native_resolution)
    dm_new_connector_state->scaling = RMX_ASPECT;
```

### 故障模式

代码在没有明确用户空间请求或文档化硬件要求的情况下，将 `RMX_FULL` 设置为自动或默认缩放模式。

---

## 十六、i915 CX0 PHY 寄存器访问协议

### 不变式

访问 CX0 PHY 寄存器必须使用适当的事务包装器和前置配置，否则会导致 PHY 失败、硬件超时和类似"PHY * failed after N retries"的错误。

### 强制性事务包装器

所有通过 `intel_cx0_rmw()`、`intel_cx0_read()` 或 `intel_cx0_write()` 访问 CX0 PHY 寄存器的函数必须包含在事务中：

```c
intel_wakeref_t wakeref;
wakeref = intel_cx0_phy_transaction_begin(encoder);
// ... PHY 寄存器访问 ...
intel_cx0_phy_transaction_end(encoder, wakeref);
```

### C10 VDR 寄存器编程序列

对于 C10 PHY（通过 `intel_encoder_is_c10phy()` 检查），通过 MsgBus 访问 PHY 内部寄存器需要先设置 `C10_VDR_CTRL_MSGBUS_ACCESS`。

```c
// 正确：访问内部寄存器前设置 MSGBUS_ACCESS
if (intel_encoder_is_c10phy(encoder))
    intel_cx0_rmw(encoder, owned_lane_mask, PHY_C10_VDR_CONTROL(1), 0,
                  C10_VDR_CTRL_MSGBUS_ACCESS, MB_WRITE_COMMITTED);
// ... 然后访问 PHY_CMN1_CONTROL 或其他内部寄存器 ...
```

### AUXLess ALPM 条件寄存器访问

当功能未激活时，必须使用提前返回来跳过所有寄存器访问。即使使能位在写入中条件性设置，无条件访问 PHY 寄存器也可能导致失败。

```c
// 错误：功能未激活时仍访问寄存器
bool enable = intel_alpm_is_alpm_aux_less(...);
for (i = 0; i < 4; i++) {
    intel_cx0_rmw(..., enable ? BIT : 0, ...);  // 仍然访问寄存器
}

// 正确：功能未激活时提前返回
if (!intel_alpm_is_alpm_aux_less(enc_to_intel_dp(encoder), crtc_state))
    return;
// 仅在功能激活时访问寄存器
for (i = 0; i < 4; i++) {
    intel_cx0_rmw(..., BIT, ...);
}
```

### 故障模式

调用 `intel_cx0_rmw()`、`intel_cx0_read()` 或 `intel_cx0_write()` 的新函数，要么（1）缺少 `intel_cx0_phy_transaction_begin/end` 包装器，要么（2）没有文档说明所有调用者持有一个活跃的事务。

---

## 十七、DisplayPort DPCD 寄存器访问模式

### 关键知识点

读取某些 DPCD 寄存器会触发 DP/eDP 接收器中非预期的硬件状态变化，导致屏幕闪烁、链路训练失败或电压摆幅报告错误。这是因为某些寄存器参与链路训练状态机，不适合任意读取访问。

### 有副作用的 DPCD 寄存器

链路训练状态范围内的寄存器在被读取时可能导致状态机转换（即使这种行为在技术上是非合规的）：

| 寄存器 | 地址 | 读取风险 |
|--------|------|---------|
| `DP_LANE0_1_STATUS` | 0x202 | 参与 CR/EQ 训练状态机；可能触发状态变化 |
| `DP_LANE2_3_STATUS` | 0x203 | 同上 |
| 训练状态寄存器 | 0x202-0x207 | 链路训练反馈回路的一部分 |

### 更安全的探测选择

- `DP_TRAINING_PATTERN_SET` (0x102)：配置寄存器，读取不影响接收器状态
- `DP_DPCD_REV` (0x000)：能力寄存器，无副作用
- 训练状态范围之外的简单状态/能力寄存器

### 审查要点

- 补丁修改用于探测、检测或 quirk 变通方案的寄存器地址时，需要仔细分析新寄存器的角色
- 检查该寄存器是否在正常链路训练序列（CR/EQ）中被读取
- 参与状态机的寄存器在预期序列之外读取时可能产生不同的硬件响应
- 某些接收器存在非合规行为，仅在特定寄存器访问模式下显现

```c
// 变更 DPCD 探测地址 -- 验证新寄存器是否安全
-   ret = drm_dp_dpcd_probe(aux, DP_DPCD_REV);
+   ret = drm_dp_dpcd_probe(aux, DP_LANE0_1_STATUS);  // 有风险 -- 训练状态机
```

### 故障模式

将 DPCD 探测/quirk 寄存器地址更改为链路训练状态范围（0x202-0x207）的寄存器，且未说明该寄存器在所有操作上下文中读取是安全的。

---

## 十八、AMDGPU PSP 固件版本检查

### 不变式

在不支持新命令的固件上调用新的 PSP GFX 命令会导致命令失败、硬件超时或未定义行为。PSP 固件和硬件独立版本化：较新的硬件可能运行较旧的固件。

### 双重版本保护要求

调用 `psp_cmd_submit_buf()` 或 `psp_get_fw_reservation_info()` 使用新的 GFX_CMD_ID 时，**两个检查都需要**：
- `amdgpu_ip_version(adev, MP0_HWIP, 0)` -- 硬件能力
- `adev->psp.sos.fw_version` -- 固件能力

```c
// 错误：仅检查 IP 版本，缺少固件版本保护
if (amdgpu_ip_version(adev, MP0_HWIP, 0) == IP_VERSION(14, 0, 2)) {
    psp_get_fw_reservation_info(psp, GFX_CMD_ID_NEW_FEATURE, ...);
}

// 正确：同时检查 IP 版本和固件版本
if (amdgpu_ip_version(adev, MP0_HWIP, 0) == IP_VERSION(14, 0, 2)) {
    if (adev->psp.sos.fw_version >= MINIMUM_FW_VERSION) {
        psp_get_fw_reservation_info(psp, GFX_CMD_ID_NEW_FEATURE, ...);
    }
}
```

### 审查要点

- `psp_gfx_cmd_id` 中新增的枚举值
- 检查 `amdgpu_ip_version(adev, MP0_HWIP, 0)` 但缺少对应 `adev->psp.sos.fw_version` 检查的函数
- 在 init/resume 路径中未经版本验证的 PSP 命令提交

### 故障模式

仅通过 IP 版本检查（无固件版本保护）就调用 PSP GFX 命令的代码，尤其是针对较新 IP 版本（14.0.2、14.0.3 或更新）中引入的命令。

---

## 十九、XE SR-IOV 虚拟功能（VF）约束

### 不变式

从 VF（Virtual Function）上下文尝试注册或访问 PF（Physical Function）专属的硬件资源会导致硬件失败、MMIO 超时或不正确的驱动行为。

### VF 受限的资源

- I2C 控制器（仅 PF 可访问）
- 直接 SOC_BASE MMIO 寄存器访问
- 某些电源管理功能
- GuC 固件加载（由 PF 处理）

### required VF 检查

在 probe 函数中添加新的设备或子系统注册时（特别是 I2C、PMU 或显示组件等硬件控制器），验证该资源对 VF 是否可访问。如果是 PF 专属资源，添加提前返回保护：

```c
// 正确：访问 PF 专属资源前检查 VF
int xe_foo_probe(struct xe_device *xe)
{
    if (IS_SRIOV_VF(xe))
        return 0;

    // ... PF-only 初始化 ...
}
```

### 审查要点

新增的 `*_probe()` 或设备注册函数如果：
- 访问 SOC_BASE MMIO 区域
- 为硬件控制器（I2C、PMU）注册 platform 设备
- 设置来自根设备的中断
- 直接访问 PCI 配置空间

且没有 `IS_SRIOV_VF()` 保护，标记为潜在 bug。

### 故障模式

xe 驱动中注册 platform 设备或访问硬件控制器的新 probe 函数，在资源对 VF 不可用时未经 `IS_SRIOV_VF()` 检查。

---

## 二十、fwnode API 错误返回约定

### 关键知识点

某些 fwnode API 在失败时返回 `ERR_PTR()` 而非 NULL。使用 NULL 检查会遗漏错误条件，导致后续解引用错误指针时崩溃。

| 函数 | 失败时返回 |
|------|-----------|
| `fwnode_create_software_node()` | `ERR_PTR()` |
| `software_node_register()` | `int`（不是 `ERR_PTR()`） |
| `fwnode_graph_get_*()` | NULL |

```c
// 错误：fwnode_create_software_node 返回 ERR_PTR，不是 NULL
fwnode = fwnode_create_software_node(props, NULL);
if (!fwnode)
    return -ENOMEM;  // 永不触发 -- ERR_PTR 非 NULL
// 崩溃：fwnode 是无效错误指针，解引用失败

// 正确：检查错误指针并提取错误码
fwnode = fwnode_create_software_node(props, NULL);
if (IS_ERR(fwnode))
    return PTR_ERR(fwnode);
```

### 故障模式

调用 `fwnode_create_software_node()` 或类似的返回 `ERR_PTR()` 的 fwnode API 后，使用 `if (!fwnode)` 或 `if (fwnode == NULL)` 进行检查。

---

## 快速检查清单

- **hwseq 路径中的休眠函数**：`fsleep`、`msleep`、`usleep_range`、`mutex_lock`、`GFP_KERNEL` 分配出现在硬件序列器函数中时需要 atomic 上下文分析
- **Atomic commit 回调实现**：CRTC、plane 和 encoder 的 atomic 回调在 non-blocking commit 期间运行于 atomic 上下文
- **VBLANK 和 page flip 处理函数**：始终 atomic 上下文，禁止休眠
- **系统 PM 中的运行时 PM 标志**：如果 `xe_pm_resume()` 或 `xe_pm_suspend()` 使用 `d3cold.allowed` 等标志做条件行为，验证这是否是故意的（通常不是）
