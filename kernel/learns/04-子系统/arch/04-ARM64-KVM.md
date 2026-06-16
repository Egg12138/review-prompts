<!-- Source: subsystem/kvm-arm64.md -->

# ARM64 KVM（Host/EL1）知识

本文覆盖 ARM64 特定 KVM 实现（在 EL1 运行）的不变量和缺陷模式，内容来自历史修复和 ARM 架构参考手册（ARM ARM）。

---

## VM 与 VCPU 生命周期初始化

ARM64 KVM 要求客户执行前有一个严格的初始化顺序。违反此顺序会导致未初始化的硬件状态或架构不一致。

> 不正确的初始化导致 **Hypervisor 异常**、**客户执行失败**和来自 KVM API 的 **-EPERM/-ENOEXEC** 错误。

### 不变规则

- **虚拟 ID 寄存器初始化**：诸如 `VMPIDR_EL2` 和 `VPIDR_EL2` 之类的虚拟寄存器在热重启时重置为 **UNKNOWN** 值。它们必须在首次进入 EL1 之前被软件初始化。
- **特性锁定与最终化**：修改客户可见寄存器布局或系统寄存器行为的架构特性（例如向量或标签扩展）必须在首次 VCPU 进入之前通过 `kvm_arm_vcpu_finalize()` 最终化和锁定。实现必须拒绝任何在客户已进入 `RUNNING` 状态后修改特性配置的尝试。
- **判断"客户已开始运行"的谓词**：使用 `vcpu_has_run_once(vcpu)`。不要使用 `kvm_vcpu_initialized(vcpu)` 来回答这个问题：它仅反映是否已调用 `KVM_ARM_VCPU_INIT`，且之后永远为 true，包括运行中和运行后。
  ```c
  /* 错误：一旦 KVM_ARM_VCPU_INIT 发生就通过，包括在 KVM_RUN 之后 */
  if (!kvm_vcpu_initialized(vcpu))
      return -EBUSY;

  /* 正确：一旦客户实际开始执行就拒绝 */
  if (vcpu_has_run_once(vcpu))
      return -EBUSY;
  ```
- **首次运行资源同步**：VGIC 映射和陷阱计算必须在 VCPU 首次进入客户模式之前最终化。内核利用内部安全网（例如 `kvm_arch_vcpu_run_pid_change()`）在首次 VCPU 转换期间强制执行此顺序。
- **VGIC 映射**：在首次 VCPU 运行之前，必须通过 `kvm_vgic_map_resources()` 将虚拟 GIC 资源映射到客户。
- **陷阱计算**：`kvm_calculate_traps()` 必须在所有特性标志和系统寄存器最终化之后、首次 VCPU 进入之前被调用。

### 检查要点

**报告为缺陷：**
- 未先调用相关最终化 ioctl 就尝试运行 VCPU。
- 在首个 VCPU 进入 `RUNNING` 状态后修改客户特性寄存器。
- 使用 `kvm_vcpu_initialized()` 而非 `vcpu_has_run_once()` 门控"客户已开始运行"的判断——会错误接受运行后重配置。
- **UAPI 特性暴露**：向用户空间暴露硬件特性的 `ID_AA64*` 寄存器字段，而这些特性并未被显式支持或完全实现。**审查者检查**：确保任何暴露的位在 `HCR_EL2`、`CPTR_EL2` 或 `MDCR_EL2` 中有相应的使能/陷阱配置。

---

## 架构状态管理（PSCI 与复位）

跨复位和电源转换的客户架构状态的一致性至关重要。

> 状态管理缺陷导致**不正确的执行流**、**寄存器损坏**和**安全绕过**。

### 不变规则

- **热复位（PSCI CPU_ON）**：大多数系统寄存器在热复位时重置为 **UNKNOWN** 值。软件必须显式初始化执行环境，包括禁用 EL2 陷阱并确保 AArch64 客户的 `HCR_EL2.RW == 1`。注意：在没有 `FEAT_AA32EL1` 的硬件上，`HCR_EL2.RW` 是 RAO/WI，写入在架构上是冗余的但无害。
- **客户标识与亲和性合理性**：呈现给客户用于路由或标识的标识符（例如 `MPIDR_EL1` 亲和值、GIC 目标 ID）必须在所有 VCPU 间唯一且架构一致。KVM **不能**实现工作区来适应违反 ARM ARM 的客户配置。
- **异常注入顺序**：异常注入必须跨退出/进入保持 `ELR_EL1`/`SPSR_EL1`。异常注入状态和同步异常/SError 处理路径之间的不协调导致 ELR 被破坏。
- **阶段 2 拆除**：阶段 2 拆除必须与并发的 `mmu_notifier` 回调序列化；在通知器遍历器仍能观察到它时释放页表会导致 UAF 或双重释放。

### 检查要点

**报告为缺陷：**
- 引入逻辑处理非唯一客户 CPU 亲和性，或放宽标识寄存器验证的补丁。
- 修改 `ELR_EL1`/`SPSR_EL1` 而未在下一个客户进入前原子地解析待处理异常状态的异常注入路径。
- 在 `mmu_notifier` 可能仍活动时销毁页表的拆除序列。

---

## VGIC CPU 接口访问

虚拟 GIC CPU 接口寄存器对软件访问模式高度敏感。

> 违反 GIC 接口访问规则会导致**中断挂起**、**虚假异常**或**不可预测的 CPU 行为**。

### 不变规则

- **优先级寄存器写入**：将 `ICV_AP<0|1>R<n>_EL1` 写入到最后一个读取值或全零以外的任何值（当没有 Group-N 活动优先级时），或乱序写入（AP0 必须在 AP1 之前），**可能导致**虚拟中断优先级的 **UNPREDICTABLE 行为**。这些寄存器只应在上下文切换或电源管理时被触及。
- **自我同步**：当 PE 掩蔽中断时（`PSTATE.{I,F} == {0,0}`），对 `ICV_IAR0_EL1` 和 `ICV_IAR1_EL1` 的读取是自我同步的。对 `ICV_PMR_EL1` 的写入是严格自我同步的（不需要 ISB）。
- **系统寄存器接口门控**：访问内存映射的 GIC 虚拟 CPU 接口（`GICV_*`）由 `ICC_SRE_EL1_NS.SRE` 门控（而非 `GICD_CTLR.ARE`）。当 `SRE == 1` 时，`GICV_*` 寄存器可能是 RAZ/WI；软件应使用系统寄存器接口。

---

## 快速检查清单

- **HCR_EL2 同步**：影响 TLB 缓存字段（`RW`、`NV1`、`NV`、`E2H`、`FWB`、`DCT`）的 `HCR_EL2` 写入需要 TLB 失效才能使翻译变更生效——`ERET` 本身不会刷新这些缓存字段。对于非 TLB 缓存字段，只要 `SCTLR_ELx.EOS == 1`，`ERET` 到 EL1/0 起 CSE 的作用（始终在 nVHE/VHE 路径中检查 `FEAT_ExS` 语义）。世界切换屏障放置在 nVHE 和 VHE 路径中不同；两者必须独立验证。
- **MTE 特性过滤**：验证基于硬件支持和 VM 类型（受保护 vs 非受保护）正确过滤了客户 ID 寄存器中的 MTE 特性。
- **特性 ID / RESx 可写掩码**：暴露给用户空间的 `ID_AA64*` 寄存器字段必须有正确的可写掩码和运行时清洗。在硬件中为 RES0 但暴露为可写的字段会导致静默的客户错误配置。每个新的字段暴露必须与正确的 `kvm_id_reg_rw_mask` 或 RESx 处理条目以及相应的 `HCR_EL2`、`CPTR_EL2` 或 `MDCR_EL2` 陷阱/使能配对。
- **`vcpu_sysreg` 编号是稀疏的——绝不要对它可以范围检查**：`enum vcpu_sysreg` 索引 `vcpu->arch.ctxt.sys_regs[]`，但 VNCR 映射的条目由其 VNCR 页字节偏移编号，而不是声明顺序。通过 enum 上的数值范围选择或跳过寄存器会覆盖未预期的寄存器：写成 `if (i >= CNTVOFF_EL2 && i <= CNTP_CTL_EL0) continue;` 的"跳过定时器块"的循环也会跳过 `SCTLR_EL1` / `TCR_EL1` / `MAIR_EL1` / ...，静默地丢弃客户状态。标记对 `enum vcpu_sysreg` 值的任何数值范围或 `<` / `>` 比较；要求显式的每寄存器允许列表。

---

## VGIC LPI 和 vLPI 不变量

VGIC LPI（Locality-specific Peripheral Interrupt）管理涉及分层锁定协议和 GICv4 VPE 所有权规则，是反复出现的缺陷源。

- **vgic_irq / LPI xarray 锁序**：由 `irq_lock` 保护的 `vgic_irq` 结构上的操作不能同时持有 LPI xarray 锁，除非序被显式文档化。在 xarray 锁被持有期间释放并重新获取 `irq_lock` 违反了序并导致死锁。
- **原子上下文中的 LPI xarray 访问**：在持有 LPI xarray 锁时不能从原始自旋锁上下文调用 `vgic_put_irq()`；任何可能调用 `vgic_put_irq()` 的路径必须文档化 LPI xarray 锁可能被获取。
- **GICv4 vLPI 取消映射**：从 VPE 取消映射 vLPI 必须在所有调用路径上成功；失败的取消映射留下悬空的转发条目。对取消映射失败使用 `WARN` 并将其视为缺陷。
- **vPE 分配门控**：当 vPE 分配被禁用时，**不能**尝试 vLPI 映射。在 vPE 可用性检查上门控所有转发设置。

### 检查要点

**报告为缺陷：**
- 在持有 `irq_lock` 时获取 LPI xarray 锁的新 VGIC 代码（除非序被显式建立）。
- 未验证 vPE 分配已启用就进行的 vLPI 转发设置。
- 静默忽略或吞咽 vLPI 取消映射错误的取消映射路径。

---

## 阶段-2 页错误处理器竞态

`user_mem_abort()` 路径及其 pKVM 等价物有反复出现的资源泄漏和过时状态缺陷模式。

- **错误路径上的页泄漏**：错误处理器中每个已解析 PFN 但提前退出的错误路径必须在返回前丢弃 PFN 引用。错误时缺少 PFN 释放会导致页泄漏。
- **Memcache 初始化**：传入错误处理器的 `kvm_mmu_memory_cache` 指针必须在使用前初始化；未初始化的指针解引用会产生难以追溯的 NULL 解引用崩溃。
- **vma_shift 过时**：在错误输入时计算的 `vma_shift` 值如果 VMA 被并发修改可能过时（例如嵌套的 hwpoison 注入）。获取 mmu_lock 后重新检查 VMA 属性。

### 检查要点

**报告为缺陷：**
- `user_mem_abort()` 或阶段-2 错误处理器中返回而未释放先前解析的 PFN 的错误路径。
- 解引用未初始化的 `kvm_mmu_memory_cache` 指针的错误处理器。

---

## GICv3 陷阱序（ICH_HCR_EL2）

- **ICH_HCR_EL2.En 同步**：对 `ICH_HCR_EL2.En` 的更改不保证在没有显式退出/重新进入周期的情况下被后续客户执行观察到。当启用或禁用 GICv3 虚拟 CPU 接口陷阱时，必须跟随一个强制客户退出以刷新 CPU 流水线中的陈旧 `ICH_HCR_EL2` 状态。
- **跨进入/退出的陷阱配置序**：`ICH_HCR_EL2` 中的 GICv3 陷阱位必须在客户进入早期重新同步（在 LR/VMCR 加载之前），以避免具有不正确陷阱配置的情况下传递中断的窗口。
- **受保护与非受保护的陷阱发散**：在 pKVM 中，受保护 VM 的 GICv3 陷阱初始化与非受保护客户不同；错误的路径会静默地使陷阱保持非活动状态。

### 检查要点

**报告为缺陷：**
- 修改 `ICH_HCR_EL2.En` 而没有后续客户退出以刷新变更的代码。
- 在重新同步 `ICH_HCR_EL2` 陷阱位之前加载 LR 或 VMCR 的客户进入路径。
