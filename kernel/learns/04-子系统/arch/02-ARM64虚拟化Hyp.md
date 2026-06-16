<!-- Source: subsystem/hyp-arm64.md -->

# ARM64 虚拟化 Hyp（EL2）知识

本文覆盖 ARM64 虚拟机监视器（Hypervisor）在 EL2 的实现，重点关注 pKVM（Protected KVM）和 nVHE 隔离，内容来自历史修复和 ARM 架构参考手册（ARM ARM）。

---

## pKVM 威胁模型与范围

pKVM 在 EL2 针对不同层次的攻击者提供一组固定的保证。审查必须根据此模型确定发现的范围：模型内的违规是缺陷；模型外的不期望行为（例如主机自 DoS）是加固改进，而非缺陷。

### pKVM 保证的内容

- **客户机密性 vs 主机**：主机 EL1 不能读取受保护 VM 的内存或寄存器状态。
- **客户完整性 vs 主机**：主机 EL1 不能带外地修改受保护 VM 的内存或寄存器状态。
- **Hypervisor 完整性 vs 主机和客户**：任一方都不能破坏 EL2 内存、重定向 EL2 控制流或逃逸阶段-2 隔离。
- **主机可用性 vs 客户**：客户机不能使主机内核崩溃或击垮 EL2（击垮 EL2 会连带击垮主机）。

### pKVM 不保证的内容

- **主机可用性 vs 自身**：主机内核可以使自己恐慌，或通过其特权代码路径触发 hyp 恐慌。pKVM 不防御主机自身。
- **固件/硬件错误的可靠性**：如果 EL3、SMMU、GIC 或其他固件/硬件行为异常，pKVM 无法恢复。

### 任何可恐慌的 EL2 路径的可达性测试

对于任何能在 EL2 到达致命原语的代码（`BUG`、`BUG_ON`、`WARN_ON`、直接 `hyp_panic()`），问：**谁能触发它？**

| 触发源 | 判定 |
|---|---|
| EL2 内部不变量违规 | EL2 本身的缺陷，与恐慌分开 |
| 硬件/固件错误 | 超出范围（信任边界） |
| 主机内核，通过特权代码路径 | 不是缺陷。如果代价低则是加固改进 |
| 客户，通过 DMA / 超调用副作用 / 共享内存 | **缺陷**（违反主机可用性 vs 客户） |
| 主机用户空间，通过 syscall/ioctl → 主机内核 → 超调用 | **缺陷**（违反 Linux 内核安全模型：用户空间不能使内核崩溃） |

主机用户空间行很重要，因为许多超调用通过主机的系统调用表面从用户空间可达；即使 EL1 的直接调用者是主机内核，在用户空间影响的输入上发生 hyp 恐慌也破坏了 Linux 标准的"用户空间不能使内核崩溃"属性。

---

## EL2 执行上下文（nVHE/pKVM）

在 nVHE/pKVM 的 EL2，超调用（或其他陷阱）处理器在捕获 CPU 上作为单个**原子、不可抢占的单元**运行，物理中断被掩蔽，完成后通过 `eret` 返回到 EL1。EL2 hyp 不是内核：**没有调度器、没有抢占、也没有内核上下文中的延迟工作机制**——没有 `sleep`/`schedule`、工作队列、软中断（softirqs）、RCU 回调、内核线程、`copy_{from,to}_user`、`printk`/`pr_*`，也没有互斥锁或 `irqsave` 锁（中断已被掩蔽，EL2 锁是 `hyp_spin_lock`）。处理器不能中途被抢占，也不能将工作交给后续上下文：无论它做什么，都必须在 `eret` 之前同步完成。

### 误报示例

"这个 `READ_ONCE()` 和后面的 `cmpxchg()` 可能竞态，因为存储只有经过*延迟写入*后才可见"的发现假设处理器可在两者之间被抢占。在 EL2 nVHE 没有这样的本地间隙——序列在捕获 CPU 上原子地运行到完成。

### 仍在范围内——跨 CPU 并发

此规则仅移除了**本地**抢占/延迟假设。两个各自运行对共享 EL2 状态的处理器的物理 CPU 之间的真正并发是真实的，仍必须在 LKMM（内存序、cmpxchg 跨 CPU 可见性）下进行分析。不要让"EL2 是原子的"坍缩为"EL2 是单线程的"。

**不要标记：**
- 要求单个 EL2 nVHE 处理器在同一 CPU 上被抢占或其工作延迟到后续上下文的竞态或重排序。

---

## 主机去特权化边界（pKVM 生命周期）

在 nVHE/pKVM 中，主机内核在 EL2 启动，一旦 hyp 设置完成就*自降特权*到 EL1。这个边界是审查 pKVM 补丁最重要的架构状态：相同的代码可以在完全不同的信任机制下运行，取决于它在哪一侧执行。

边界是 `finalize_pkvm()`（`arch/arm64/kvm/pkvm.c`），注册在 `device_initcall_sync`。它调用 `pkvm_drop_host_privileges()`，后者将主机永久切换出 EL2。在此之前，主机内核仍在 EL2 执行——代码有特权，可以直接设置 EL2 状态、安装 hyp 文本/数据、填充每 CPU 状态和最终确定陷阱配置；与未来-EL2 共享的内存在原地可写。之后，主机在 EL1，只能通过超调用调用 EL2；EL2 私有内存变得不可访问。

### 审查者可 grep 的标记

- `__init` / `__initdata` 在触及 hyp 的代码上：仅在去特权化之前。
- **Initcall 级别**是规范的前-vs-后标记。`finalize_pkvm()` 注册在 `device_initcall_sync`。严格在此之前注册的函数（`early_initcall`、`pure_initcall`、`core_initcall`、`postcore_initcall`、`arch_initcall`、`subsys_initcall`、`fs_initcall`、`rootfs_initcall`、`device_initcall` 及这些的 `_sync` 变体）在 `finalize_pkvm` 之前运行，属于去特权化前。`late_initcall` 及更晚的级别运行在去特权化后。`module_init(fn)` 展开为 `device_initcall(fn)`。因此从 `kvm_arm_init`（`device_initcall`，包括 `init_hyp_mode`、`init_subsystems`、`finalize_init_hyp_mode`）可达的代码属于去特权化前。
- `__pkvm_init` 和 **`__pkvm_init_finalise`**（`arch/arm64/kvm/hyp/nvhe/setup.c`）在去特权化过程*期间*执行，在 `kvm_arm_init` 之后、主机退出 EL2 之前执行。这些是从特权上下文中设置 EL2 私有状态的最后机会。
- `is_kvm_arm_initialised()`（`arch/arm64/include/asm/virt.h`）是"KVM-arm 初始化已完成"的规范谓词（去特权化后）。形式为 `if (... || is_kvm_arm_initialised()) return -EINVAL;` 的保护拒绝*之后*的调用，所以该调用仅在特权窗口中被允许。
- `is_protected_kvm_enabled()`（`arch/arm64/include/asm/virt.h`）是"pKVM 模式已配置"的规范谓词。它通过 cpufeature 检测很早（在任何 initcall 运行之前）就变为 true，并且独立于去特权化状态——所以它本身*不*告诉你特权窗口是否仍然开放。Hypervisor 实际使用的鉴别器是 `kvm_protected_mode_initialized` static key（主机端通过 `is_pkvm_initialized()` 读取），在 pKVM 最终确定期间启用：当它关闭时，早期的"特权"超调用是可达的。
- **超调用 ID 区域**决定新超调用的可用阶段。在 `__KVM_HOST_SMCCC_FUNC_MIN_PKVM` 之前的新 `enum __kvm_host_smccc_func` 项是仅 init 的（pKVM 最终确定后消失）；在 `MIN_PKVM` 和 `__KVM_HOST_SMCCC_FUNC_PKVM_ONLY` 之间是一直可用的；在 `PKVM_ONLY` 之后是仅 pKVM 最终确定后可用。错区域的新 ID 在错误阶段可达。

### 检查要点

**报告为缺陷：**
- 处理主机共享内存而未识别它在去特权化的哪一侧执行的代码。
- 将设置代码移动到跨越 `finalize_pkvm` 边界而没有更新移动代码的信任假设的补丁。
- 运行时超调用处理器引用了 `__init` / `__initdata` 符号（那些在去特权化后被释放/不可访问；特别是 `kmemleak_free_part` 在 `finalize_pkvm` 中调用于 hyp `.bss` / `.data` / `.rodata`）。

---

## EL2 安全与信任边界（pKVM）

去特权化后（`finalize_pkvm` 之后），主机（EL1）是客户机密性、客户完整性和 hypervisor 完整性的攻击者。EL2 **不能**从任何主机控制的源派生安全敏感值。

> 未能验证主机提供的数据会导致 **Hypervisor 级内存损坏**或**信息泄漏**。严重性分类按照 pKVM 威胁模型与范围：主机自身的主机可用性不在范围内，但机密性、完整性或客户可达可用性的泄露是。

### 不可信的主机数据源

- **主机内存**（`kern_hyp_va` 解引用）：受 **TOCTOU**（Time-of-Check to Time-of-Use）攻击影响。数据必须复制到私有 EL2 内存（例如通过结构体赋值或 `memcpy` 到本地栈）**在**验证或操作之前。
- **系统寄存器**携带主机写入的状态（例如 `SCTLR_EL1`、`HCR_EL2`）：EL2 必须使用**硬编码的架构常量**或已知好的 EL2 私有状态，而不是读回主机上次写入的内容。
- **超调用参数**：每个主机提供的值必须在使用前经过验证和边界检查。绝不要对原始超调用参数进行操作。寄存器中传递的标量（例如 `u64`、通过 `DECLARE_REG` 捕获的句柄、索引）一旦保存在本地变量中就变成了 EL2 私有的；复制后验证规则适用于通过 `kern_hyp_va` 解引用的主机内存指针，不适用于寄存器传递的值。
- **双重读取风险**：绝不要多次解引用主机提供的指针。将必要字段一次复制到 EL2 私有内存。
- **分配源**：EL2 绝不能从空闲列表或 memcache 进行分配，如果其头指针位于主机内存中——主机可以将分配重定向到攻击者选择的物理地址。主机驻留的缓存如 `stage2_teardown_mc` 是主机端回收页的接收器，绝不是分配源。
- **VA 翻译（`AT`）**：要执行主机 VA 的阶段 1+2 翻译，EL2 必须使用 `AT S12E1R`。如果翻译失败，硬件在 `PAR_EL1` 中报告错误（`.F = 1` 位）。
  - **注意**：目标翻译机制（EL1&0 vs EL2&0）取决于 `HCR_EL2.{E2H, TGE}`。

### 检查要点

**报告为缺陷：**
- 多次解引用主机指针，中间没有复制到私有 EL2 内存。
- 假设主机共享结构（如 `struct kvm_vcpu`）中的值在检查和使用之间保持不变的逻辑。
- 从已知可被主机修改的寄存器读取安全敏感状态（如陷阱配置）的代码。

```c
// 错误：双重读取主机共享内存 或 栈溢出风险
void handle_hcall(struct kvm_vcpu *host_vcpu) {
    // 1. 双重从主机内存读取！
    if (vcpu_has_sve(kern_hyp_va(host_vcpu))) {
        do_sve(kern_hyp_va(host_vcpu));
    }
    // 2. 不安全的栈分配（~4KB 溢出）
    struct kvm_vcpu local_vcpu = *kern_hyp_va(host_vcpu); 
}

// 正确：原子复制后验证（小结构体）
void handle_hcall(struct vcpu_reset_args *host_args) {
    struct vcpu_reset_args local_args;
    
    // 一次性复制到私有内存，防止 TOCTOU
    memcpy(&local_args, kern_hyp_va(host_args), sizeof(local_args));
    
    // 只验证和操作 PRIVATE 副本
    if (local_args.flags & VALID_FLAG) {
        update_hyp_state(&local_args);
    }
}
```

---

## pKVM/nVHE 不变量（EL2）

违反 hypervisor 不变量会导致 **Hypervisor 恐慌**、**静默隔离破坏**或**内存保护绕过**。

### 安全元数据初始化（pKVM）

跟踪物理资源的安全关键元数据（例如 `hyp_vmemmap` 中的页所有权表）必须使用计算为最低特权或"未拥有"状态的初始化模式。这确保意外访问零填充或未初始化的内存不会导致未授权的所有权或许可授予。在 pKVM 中，这是通过基于补码的状态实现的，其中零初始化计算为 `PKVM_NOPAGE`。

> **报告为缺陷：** 通过直接与零比较或假设零填充元数据意味着"hypervisor 拥有"来检查页状态的代码。

### 阶段-2 VMID 与一致性

架构不要求 `VTTBR_EL2` 和 `VTCR_EL2` 对所有 PE 上的一个 VMID 相同，**前提**是 `VTTBR_EL2.CnP == 0`。

> **报告为缺陷：** 设置 `VTTBR_EL2.CnP = 1`（Common not Private）在多个 PE 上，如果它们对同一 VMID 的转换表指针不同。这会导致 **CONSTRAINED UNPREDICTABLE** 翻译。

### 细粒度陷阱（FEAT_FGT）

对 FGT 寄存器的访问（例如 `HFGRTR_EL2`）可能被重排序。要保证新启用的陷阱对后续指令生效，必须发生**上下文同步事件（CSE）**（例如 `isb` 或异常返回）。

### SMC 捕获

对于 AArch64 客户，通过 `HCR_EL2.TSC = 1` 从 EL1 捕获的 `SMC` 指令导致 `ESR_EL2.EC = 0x17`；对于 AArch32 客户，`SMC32` 使用 `EC = 0x13`。`SPSR_EL2.SS` 捕获被捕获的 EL1 上下文的 `PSTATE.SS`（Software Step）位。

---

## 状态发散与初始化边界

pKVM 状态与主机物理隔离。假设主机状态变更对 EL2 可见会导致**状态失同步**。

| 主机状态 | EL2 Hyp 状态 | 同步机制 | 源参考 |
| :--- | :--- | :--- | :--- |
| `struct kvm` | `struct pkvm_hyp_vm` | `pkvm_create_hyp_vm()` | `arch/arm64/kvm/hyp/nvhe/pkvm.c` |
| `struct kvm_vcpu` | `struct pkvm_hyp_vcpu` | 超调用参数 | `arch/arm64/kvm/hyp/nvhe/hyp-main.c` |
| ID 寄存器 | Hyp 私有 ID 寄存器 | `pkvm_hyp_vm` 中的清洗 | `arch/arm64/kvm/hyp/nvhe/sys_regs.c` |

- **Hyp vs 主机反向指针**：`struct pkvm_hyp_vm` *嵌入*了一个 `struct kvm kvm`——EL2 私有的 hyp 副本——并分别持有 `struct kvm *host_kvm`，一个指向主机（不可信）实例的反向指针。通过 `hyp_vm->kvm.X` 访问的字段是 EL2 私有的，初始化后可视为可信；通过 `hyp_vm->host_kvm->X` 访问的字段是主机可写的，受 TOCTOU 影响。相同的区别适用于 `pkvm_hyp_vcpu`（`hyp_vcpu->vcpu` 是 EL2 私有的；`hyp_vcpu->host_vcpu` 是不可信的反向指针）。
- **持久状态规则**：从主机内存到栈的副本仅供**验证**使用。持久的客户状态变更必须同步到 EL2 私有的 hyp 结构（例如 `pkvm_hyp_vcpu`）以在跨客户进入时保持可见。

---

## `kern_hyp_va` 幂等性与指针来源

错误判断何时 `kern_hyp_va()` 转换指针会导致两个方向的误判：在它实际保持不变的指针上误报"双重 `kern_hyp_va()` 损坏指针"，以及在真正弄乱指针的情况下漏检"防御性 `kern_hyp_va()` 在已经是 hyp 指针上的应用"。鉴别器是指针的来源，**不是**地址是否在 `PAGE_OFFSET` 之下。

`kern_hyp_va()` 将主机内核线性（TTBR1）指针转换为用于在 EL2 解引用主机内存的 EL2 hyp 线性 VA。它是一个**掩码和标记**操作：清除高 VA 位，然后插入一个常量的 hyp 标记。它**没有偏移累积**，所以它在任何**已经携带 hyp 标记**的指针上是幂等的。两类这样的指针都在 **`PAGE_OFFSET` 之下**但仍然是幂等的：

- `kern_hyp_va()` 本身产生的 hyp 线性映像——所以第二次应用是空操作。
- 每个 EL2 **线性映射**指针：hyp 页分配器的输出（`hyp_phys_to_virt` / `hyp_page_to_virt`）和通过 `hyp_virt_to_phys` / `hyp_virt_to_page` 可达的任何内容。这涵盖了大型 EL2 私有对象——`pkvm_hyp_vm`、`pkvm_hyp_vcpu` 和嵌入的 `hyp_vm->kvm`。

所以真正的划分是**线性映射**（携带标记 → `kern_hyp_va` 是空操作）vs **私有 VA 范围**（无标记 → 被损坏），而不是高于/低于 `PAGE_OFFSET`。

幂等性**不**扩展至 EL2 **私有 VA 范围**指针——`pkvm_alloc_private_va_range` 的输出：`hyp_vmemmap`、fixmap、ioremap/MMIO 映射、hyp 栈。这些位于线性映射之外，不携带 hyp 标记，所以掩码会重定位它们，**第一次**应用会损坏指针。

真正误用的 `kern_hyp_va()` 的严重性取决于目标树的 `__kern_hyp_va()`。上游无条件掩码，所以私有范围输入被损坏。Android 添加了 `if (!is_ttbr1_addr(v)) return v;`（`>= PAGE_OFFSET` 检查），所以每个已经是 hyp 的指针短路到无害的空操作，只有真正的 TTBR1 指针被转换。在分配严重性前检查树携带的是哪种形式。

### 误报示例

一个初始化为 hyp VA 的指针稍后再次通过 `kern_hyp_va()` 是反复出现的陷阱。`hyp_vcpu->vcpu.arch.sve_state` 在 `pkvm_vcpu_init_sve()` 中被设置为 hyp VA，而 `unpin_host_sve_state()` 曾经再次调用 `kern_hyp_va()` 于其上；因为该转换是幂等的，调用是冗余的，而非缺陷。同样适用于 `kern_hyp_va(&hyp_vm->kvm)` 或 `kern_hyp_va(vcpu->kvm)` 当 `vcpu` 是 hyp vCPU 时。不要将这些升级为内存损坏或数据中止；最多记一个清理注释。

**不要标记：**
- 在线性映射指针上的冗余或双重 `kern_hyp_va()` 为损坏、错误或任何严重性缺陷。

---

## EL2 伙伴分配器（`hyp_pool`）

EL2 使用私有伙伴分配器（`struct hyp_pool`）。它是 EL2 **唯一**的页分配器——没有 `kmalloc`、没有 `alloc_pages`。有一个全局池（`hpool`，在 `__pkvm_init_finalise` 中设置）加上每个受保护 VM 一个（`hyp_vm->pool`）。

### API

- `hyp_alloc_pages(pool, order)`——返回一个 refcount=1 的页；OOM 时返回 NULL。页已清零。
- `hyp_get_page(pool, addr)`——增加引用计数。
- `hyp_put_page(pool, addr)`——减少引用计数；最后一个引用时页被**清零并重新附加到伙伴树**。清零发生在释放时，而非分配时。
- `hyp_split_page(page)`——将高阶块拆分为独立计数的 order-0 页；每个必须单独 put。
- `hyp_pool_init(pool, pfn, nr_pages, reserved_pages)`——`reserved_pages` 保持 refcount 1 且从不进入空闲树。
- `hyp_page_count(addr)`——返回页的当前引用计数。

### 不变量

- **锁纪律**：`pool->lock` 保护*伙伴树*和*每页元数据*。可能触发树更新的引用计数变更必须在与树更新相同的临界区内。不要在外部的 `pool->lock` 中读取或修改 `page->refcount` / `page->order` 并假设一致性。
- **`HYP_NO_ORDER` 约定**：对于高阶块，只有头部 `struct hyp_page` 携带其 order；尾部 `struct hyp_page` 携带 `HYP_NO_ORDER`。检查 `->order` 的遍历器必须处理此情况。
- **跨池**：每个 `hyp_put_page` 必须使用与分配该页相同的池。混用 `hpool` 和每 VM 池会破坏两者。
- **外部页**：`__hyp_attach_page` 接受 `[range_start, range_end)` 之外的页，在 order 0 插入而不合并——用于主机捐赠。

### 检查要点

**报告为缺陷：**
- 在没有 `pool->lock` 的情况下访问 `page->refcount` 或 `page->order`。
- 将 `hyp_alloc_pages()` 失败视为致命（`WARN_ON` / `BUG_ON`）——`-ENOMEM` 是正常的运行时结果。
- 从一个池分配并释放到另一个池。

---

## EL2 临时映射（`hyp_fixmap`）

EL2 使用 `hyp_fixmap_map(phys)` / `hyp_fixmap_unmap()` 来访问线性 hyp 映射之外的主机页。该槽是**每 CPU** 的，调用**必须配对**，映射**不能嵌套**：未能在所有出口上取消映射的路径会持有该槽并破坏同一 CPU 上的下一个用户，嵌套的映射会覆盖活动的槽。取消映射也是执行槽的 TLB 失效的操作（映射不执行），所以错过取消映射会留下陈旧的 TLB 有效条目，下一个映射的新写入 PTE 被绕过：新用户读取或写入的是前一个映射的物理页。

### 检查要点

**报告为缺陷：**
- `hyp_fixmap_map()` 的错误或提前退出路径没有全部到达匹配的 `hyp_fixmap_unmap()`。
- 在第一个取消映射之前，在同一 CPU 上的第二个 `hyp_fixmap_map()`。

---

## pKVM 页所有权转换

pKVM 跟踪跨三个角色（主机、hypervisor、客户）的物理页所有权。所有权转换（share、unshare、donate）是密度最高的 pKVM 特定缺陷模式，并且处于攻击者从主机的直接可达范围内。

- **转换前的参数验证**：每个启动所有权转换的超调用必须在修改任何 EL2 元数据之前验证提供的范围参数（基地址、大小）与当前所有权状态。未验证的范围会导致越界元数据损坏。
- **转换后的交叉检查**：完成转换后，EL2 必须验证结果所有权状态与请求的一致。静默的不一致性会传播到后续转换。
- **状态更新的原子性**：从 EL2 的角度看，受影响范围的所有权元数据和页表项必须原子更新。对其他 CPU 可观察的部分更新会创建 TOCTOU 窗口。
- **传播错误前回滚**：当 EL2 在*调用*可失败原语之前修改了所有权元数据时，错误路径必须在返回前撤销修改。`return err` 后留下部分修改的状态会导致 EL2 元数据不一致。
- **回收路径纪律**：垂死客户的回收路径必须按其记录的所有权状态枚举页，而不是通过页表遍历。已捐赠但其元数据未更新的页否则会泄漏。

### 检查要点

**报告为缺陷：**
- 未在完全验证提供的范围与当前 EL2 元数据之前就进行的所有权转换超调用。
- 遍历页表而非所有权元数据来枚举要释放的页的回收或拆除路径。

---

## FF-A 接口验证

Firmware Framework for Arm (FF-A) 在 EL2 和 EL3 之间的接口是一个大的攻击面，具有反复出现的范围/偏移量检查缺失模式。

- **偏移量和长度验证**：每个接受缓冲区描述符的 FF-A 内存共享超调用必须在解引用之前验证 `offset` 和 `length` 字段与实际的缓冲区大小。缺少检查允许主机构造导致 EL2 读取越界 hypervisor 内存的描述符。
- **版本协商**：EL2 必须强制执行商定的 FF-A 版本；EL3 的降级响应必须被正确拒绝，而不是静默地用作更高版本。对于跨 CPU 可见的版本协商状态，需要正确的 acquire/release 序。
- **不支持的接口掩码**：pKVM 未实现的可选 FF-A 1.1/1.2 接口必须在 `FFA_FEATURES` 查询的响应中被掩码。未掩码的可选接口会被通告为支持，导致客户行为不正确。

### 检查要点

**报告为缺陷：**
- 使用主机提供的偏移量或长度而没有对声明缓冲区大小进行边界检查的 FF-A 处理器。
- 通告 pKVM 未实现的可选接口的 `FFA_FEATURES` 响应。

---

## 陷阱寄存器初始化与受保护-非受保护发散

- **EL2 中的初始化**：陷阱配置寄存器必须在 EL2 私有上下文中初始化，不能依赖于主机写入的值。在受保护 VM 上，hyp 从硬编码的架构常量初始化陷阱状态；在非受保护 VM 上，VCPU 加载时复制主机设置的值。缺少初始化会在热重启后使陷阱处于 **UNKNOWN** 状态。
- **FGT 寄存器仅为 EL2**：`HFGRTR_EL2`、`HFGWTR_EL2`、`HFGITR_EL2`、`HDFGRTR_EL2`、`HDFGWTR_EL2` 和 `HAFGRTR_EL2` 在 EL1/EL0 访问时 UNDEF。主机不能架构性地写入它们，所以 EL2 侧写入后读回它们（例如验证刚应用的配置）是安全的，不是"读取主机写入状态"的违规。
- **VCPU 加载时的复制**：对于非受保护的 pKVM 客户，FGT 陷阱寄存器必须在每次 VCPU 加载时从主机 VCPU 复制到 hyp VCPU。陈旧的 hyp 侧副本会导致下一个客户入口出现错误的陷阱行为。
- **MTE/ID 寄存器初始化**：MTE 标志初始化和 ID 寄存器初始化对于受保护 VM 必须走受保护 VM 路径，而不是非受保护路径。使用错误路径会静默地使每 VM 安全标志处于不正确状态。

### 检查要点

**报告为缺陷：**
- 从主机写入的寄存器而非 EL2 私有 hyp 状态读取陷阱配置的 hyp 代码。
- 不从主机 VCPU 刷新 hyp 侧 FGT 寄存器副本的 VCPU 加载路径。

---

## 主机拥有 vs Hyp 拥有 `HCR_EL2` 位（受保护 vCPU）

对于受保护 vCPU，`HCR_EL2` 由 hyp 在 vCPU 初始化时从架构和特性状态计算，**不是**来自主机。主机的 `hcr_el2` 值绝不能被允许定义受保护客户的机制。但有一小部分 `HCR_EL2` 位是*主机拥有的运行时信号*，这些必须从主机 vCPU 副本流入 hyp vCPU（在每次加载/进入时）并在退出时流回。因此每次进入的集合是一个**允许列表**，审查者必须将代码触及的每个 `HCR_EL2` 位分类为两类之一：

- **Hyp 拥有的机制/配置位**：定义客户的翻译、执行和陷阱环境：例如 `VM`（阶段-2 使能）、`RW`（AArch64/AArch32 执行状态）、`TGE`、`IMO`/`FMO`/`AMO`（中断路由）、`TSC`（SMC 捕获）、`E2H` 和 `TID*`/`TACR` 特性陷阱位。这些在 hyp vCPU 初始化时固定。主机绝不能能为受保护客户设置或清除它们。
- **主机拥有的运行时信号**：由主机动态设置/清除以应用良性策略或注入事件，并且只有到达运行中的 hyp vCPU 才有意义：`TWI`/`TWE`（WFI/WFE 捕获策略）和 `VSE`（待处理的*虚拟 SError* 要传递给客户；在 FEAT_RAS 下主机还写入 `VSESR_EL2` 提供综合症，然后设置 `VSE` 使其待处理）。虚拟 IRQ/FIQ 待处理位（`VI`/`VF`）在架构上是同一类的注入信号，但不属于此允许列表：KVM 通过 vGIC 传递客户中断，而非通过这些 `HCR_EL2` 位。

因为每次进入的集合是一个允许列表，**两个方向都是缺陷**：

- **过度包含（安全）**：从主机向受保护的 hyp vCPU 传入机制/配置位，让主机重新配置客户的陷阱/执行环境。
- **包含不足（功能）**：从允许列表中遗漏一个主机拥有的信号意味着主机无法再将其传递到客户。典型例子：`flush_hyp_vcpu()` 将主机的 `hcr_el2` 掩码到只有 `HCR_TWI | HCR_TWE`，静默地丢弃了 `HCR_VSE`，所以主机注入的（并延迟/掩码的）虚拟 SError 从未传递给 pKVM 客户。

### 检查要点

**报告为缺陷：**
- 受保护 vCPU 加载/进入路径从主机传入除主机拥有信号外的任何 `HCR_EL2` 位。
- 每次进入的 `HCR_EL2` 允许列表遗漏了主机依赖的主机拥有注入信号，导致事件从未到达客户。

**不要标记：**
- 每次进入路径只流经允许列表中的主机拥有位，同时保持所有其他 `HCR_EL2` 位为 hyp 初始值。这种限制是正确的，不是"部分 HCR 同步"。

---

## FPSIMD/SVE 保存机制（pKVM 模式 vs 标准 nVHE）

EL2 懒切换 FPSIMD/SVE/SME 状态：客户第一次访问 FP/SIMD/SVE 时陷阱到 `kvm_hyp_handle_fpsimd()`，它解除相关的 CPTR 陷阱，保存活动的宿主机上下文并恢复客户上下文。*主机*状态的保存方式根据 **KVM 模式**而不同——是否启用 pKVM——在 pKVM 下该发散是机密性不变量，而非性能调整：

- **pKVM 已启用**：陷阱处理器在 hyp 中急切保存主机的 FP/SVE 状态，基于 `is_protected_kvm_enabled() && host_owns_fp_regs()` 门控。门控是全局模式，所以这适用于 hyp 运行的**每个**客户——受保护 VM 和非受保护客户都走 `is_protected_kvm_enabled()` 分支。主机 SVE 状态以*主机*的最大 VL（`kvm_host_sve_max_vl`）保存，而不是客户的。
- **标准 nVHE（pKVM 禁用）**：hyp **不**急切保存主机 FP 状态。主机完全可信，其 vCPU 直接运行；它自己的懒-FPSIMD 机制在运行后恢复主机状态，hyp 只通过 `fpsimd_lazy_switch_to_guest()` / `fpsimd_lazy_switch_to_host()` 在进入/退出周围编排向量长度。这些懒切换帮助函数在 pKVM 下不使用。

### 检查要点

**报告为缺陷：**
- 在 pKVM 下，一个 FP 路径恢复（或暴露）客户 FP 而没有首先确保 hyp 已保存并擦除了主机活动 FP/SVE 状态。这会泄漏主机寄存器内容到客户。
- 以客户的 VL（或任何比主机最大 VL 窄的 VL）保存*主机* FP/SVE 状态，会截断主机的寄存器。
- 基于每客户受保护标志而非 `is_protected_kvm_enabled()` 门控急切保存决策的 FP 陷阱代码，或假设 `fpsimd_lazy_switch_to_*` 路径在 pKVM 下运行。

**不要标记：**
- 标准 nVHE（pKVM 禁用）路径上缺少 `kvm_hyp_save_fpsimd_host()`。那里可信主机的懒恢复自己处理；hyp 侧急切保存不是必需的。
- 在 pKVM 下为*非受保护*客户运行的急切主机保存。门控是全局模式，所以这是正确的，而非冗余或误用。

---

## 每次加载 vs 每次进入同步接缝（pKVM 生命周期）

当指南或提交消息说某些状态"必须同步"时，它命名的*接缝*是要求的一部分。pKVM 有两个完全不同节奏的同步点：

| 接缝 | 函数 | 节奏 | 处理的内容 |
| :--- | :--- | :--- | :--- |
| **每次加载** | `handle___pkvm_vcpu_load`（调用 `pkvm_load_hyp_vcpu`） | 一次，当主机的 `KVM_RUN` 将 vCPU 加载到物理 CPU 上 | 跨多次进入持续存在的粗略配置（例如非受保护客户的 `arch.fgt` 块复制） |
| **每次进入** | `flush_hyp_vcpu`（和退出时的匹配 `sync_hyp_vcpu`） | 每次客户重新进入——在加载之间可能数千次 | 细粒度的每次进入状态变动（例如 `hcr_el2`、`mdcr_el2`、`arch.iflags`） |

在标记"缺少同步"之前，引用指南的*确切*节奏词——"在每个 vCPU **加载**" vs "在每次**进入**"/"**运行**"——并确认检查是在匹配的函数上。要求放在 vCPU 加载的复制甚至当它不在每次进入路径中也满足，反之亦然。

### 误报示例

FGT 陷阱寄存器被要求"在每次 vCPU 加载时"复制。该复制位于 `handle___pkvm_vcpu_load` 中，其非受保护 `else` 分支将整个 `arch.fgt` 块从 `host_vcpu` 复制到 hyp vCPU。标记 `flush_hyp_vcpu`（每次进入路径）为"未同步 `arch.fgt`"是误报：它是错误的接缝。

**不要标记：**
- 每次加载状态"缺失"在每次进入路径中，当它正确地在 vCPU 加载时处理。

---

## SMC imm16 / SMCCC 直通规则

EL2 控制主机被允许将哪些 `SMC` 编码转发到 EL3。这个门控是 pKVM 信任边界的一部分。

- **imm16 == 0 强制执行**：主机只被允许使用 `imm16 == 0` 的 `SMC` 指令。任何非零 `imm16` 的 `SMC` 必须在转发到 EL3 前被 EL2 拒绝。这防止主机利用不打算给客户使用的固件接口。

### 检查要点

**报告为缺陷：**
- 在未检查 `imm16 == 0` 的情况下将 SMC 指令转发到 EL3 的 SMC 直通路径。

---

## pKVM 下的大页处理

保护模式下阶段-2 中的大页（块描述符）处理是与正常页错误不同的失败模式。

- **保护模式大小验证**：为受保护 VM 安装阶段-2 映射时，映射粒度必须与请求的错误粒度进行验证。安装比预期页大小映射更大的块映射会静默地破坏隔离。
- **阶段-2 错误上的范围调整**：阶段-2 错误覆盖的范围必须从 IPA 和错误条目的级别计算，而不是从主机提供的 VMA shift。过时的或被主机操纵的 VMA shift 会产生不正确的范围调整。

### 检查要点

**报告为缺陷：**
- 安装块描述符而未验证块粒度与预期错误大小匹配的保护模式错误处理器。
- 从主机 VMA shift 派生映射范围而未根据 IPA 对齐重新验证的错误处理器。

---

## `WARN_ON` 在 EL2 的语义

在 EL2 的 nVHE/pKVM 中，`BUG_ON()` 和 `WARN_ON()` 都展开为 `BRK`，hyp 恐慌处理器将其视为致命。在 EL2 下这些宏没有"警告并继续"的语义——`WARN_ON` 触发后的代码是不可达的。

**对 EL2 的任何 `WARN_ON(cond)` 进行测试：能否 `cond` 通过任何契约允许的输入求值为真——还是只能通过违反 EL2 自身的不变量？**

- *通过契约输入*（`WARN_ON` 错误）：去特权化后的主机提供输入、分配器/查找结果、硬件/固件返回值、并发竞态、任何超时/轮询驱动的因素。
- *只能通过不变量违反*（`WARN_ON` 正确）：来自 EL2 自身刚完成的状态——EL2 刚填充的槽为 NULL、EL2 刚增加的引用计数为零、EL2 刚初始化的内部数据结构不一致。

如果其他地方的一个独立缺陷使 EL2 内部不变量可违反（例如一个槽因不同的生命周期缺陷而保持为 NULL），标记*那个*缺陷——不要将本地 `WARN_ON` 降级。`WARN_ON` 捍卫不变量；链接"不变量可能在别处被违反"到"因此这个断言是错误的"否定了它的目的。

被此测试标记为"错误"的 `WARN_ON` 是*正确性*发现：断言是错的工具，函数应该返回错误。它是否也是*缺陷*由可达性测试单独判断。仅主机内核可达的错误 `WARN_ON` 是加固改进。客户或主机用户空间可达的是缺陷。

**死分支陷阱**：EL2 的模式如 `WARN_ON(err); do_fallback();` 或 `if (WARN_ON(err)) goto out;` 中，恢复路径不会执行——WARN 路径恐慌。错误处理代码是死的。熟悉宿主机侧 `WARN_ON` 语义的审查者经常忽略这一点。

不要调用 `CONFIG_BUG=n` 将防御性 `WARN_ON` 升级为"空解引用"声明。`CONFIG_BUG=n` 仅 EXPERT 可选退出；上述测试（通过契约输入的可达性）决定正确性。

### BUG() 和 hyp_panic() 在 EL2 nvhe 是等价的

EL2 nvhe 的普通 `BUG()`（不仅是 `BUG_ON()`）展开为 `BRK BUG_BRK_IMM`。该指令被 EL2 异常向量捕获，最终调用 `hyp_panic()`。`BUG()` 和直接 `hyp_panic()` 调用因此在 EL2 是功能上等价的——两者都以 hyp 恐慌终止，不返回。

`arch/arm64/kvm/hyp/nvhe/` 中流行的约定是 `BUG()`：6 个调用点 vs 2 个直接 `hyp_panic()` 调用点。**不要将 `BUG()` <-> `hyp_panic()` 替换标记为回归**——任一种形式都不比另一种更正确或更不正确。

---

## 快速检查清单

- **死亡状态不变量**：在任何致命的初始化错误或主机触发的损坏上，VM 必须被标记为确定的"死亡"（例如通过 `struct kvm_protected_vm` 中的 `is_dying`）。这充当安全保险，防止资源重新分配逻辑在已损坏的元数据上运行。
- **异步引擎同步（推测）**：清除异步硬件引擎（如性能分析、跟踪或调试扩展）的控制/使能位**不足以**停止推测性页表遍历或内存访问。切换翻译机制时（例如 EL2 的客户 <-> 主机切换），软件必须执行所有活动引擎的架构强制同步序列（例如性能分析的 `PSB CSYNC`），后跟推测执行屏障（例如 `SB`、`DSB` + `ISB`）以明确停止越上下文执行。
  - **报告为缺陷：** 禁用异步特性但在更改翻译上下文之前未执行后续同步屏障的代码。
