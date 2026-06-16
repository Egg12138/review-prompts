<!-- Source: subsystem/kvm.md -->

# KVM 通用知识

本文覆盖跨架构 KVM 不变量、锁层次和内存管理 API 契约，内容来自 `Documentation/virt/kvm/` 和历史修复。

---

## API 与 ABI 快速检查

在深入锁和 MMU 细节之前，扫描差异（diff）中以下跨领域问题。它们在 KVM 子系统中反复出现，在聚焦审查中容易被忽略：

- **新的客户可见特性默认关闭且可枚举**。客户可以观察到的任何新行为（新的 ioctl、新的 VM 或 vCPU 能力、新的退出原因、新的模拟指令）必须默认关闭，并且通过架构的标准枚举接口可发现，例如 `KVM_CHECK_EXTENSION` / `KVM_CAP_*`、x86 上的 `KVM_GET_SUPPORTED_CPUID2`、或 ARM64 上的 ID 寄存器特性位。静默开启的特性会破坏在线迁移并使能力协商不可能。
- **没有客户或主机用户空间可达的 `WARN_ON` / `BUG_ON`**。条件可被恶意客户或非特权主机用户空间进程驱动的 `WARN_ON` 或 `BUG_ON` 是主机侧拒绝服务。应转换为 `pr_warn_once()`、向用户空间返回错误，或直接删除断言。断言适用于"内核本身有缺陷"的路径，不适用于敌手可达的输入。
- **长循环是重复出现的缺陷类型**，但修复是流特定的。遍历客户驱动的计数（memslot、vCPU、GFN 范围、rmaps/SPTE、固定页）并带有每次迭代 MMU 活动（失效、zap、clflush、unmap）的循环有软锁和 RCU 停顿时长修复的历史。没有通用的缓解措施：在 `kvm->mmu_lock` 下让出可能损害错误吞吐量，极端情况下丢弃锁并强制重试可能使客户饿死。将此视为新长运行路径的背景上下文，而非要求 `cond_resched()` 或任何特定让出机制的检查项。
- **新的 memslot 和 vCPU 标志默认为不可变**。可以在首次设置后被清除或翻转的标志会创建几乎没有任何调用者准备好应对的状态机转换；`KVM_MEM_GUEST_MEMFD` 和 `KVM_ARM_VCPU_INIT` 后的 vCPU 模型更改是具体例子。新标志默认设置为只设一次，并要求显式的理由使其可变。

---

## KVM 锁层次

KVM 使用全局和每 VM 锁的复杂层次。违反此顺序会导致循环死锁或释放后使用（UAF）。

> 不正确的锁序导致 **ABBA 死锁**和**系统挂起**。

权威的锁序是 `Documentation/virt/kvm/locking.rst`。每当变更触及 `kvm_lock`、`kvm->lock`、`vcpu->mutex`、`kvm->slots_lock`、`kvm->srcu` 或 `kvm->mmu_lock` 时请阅读它。不要将此指南作为替代。下面的项目符号捕获审查者应标记的具体失败模式。

### 不变规则

- **SRCU 约束**：`synchronize_srcu(&kvm->srcu)` 在持有 `kvm->lock`、`vcpu->mutex` 或 `kvm->slots_lock` 时被调用。因此，持有 `srcu_read_lock(&kvm->srcu)` 时**不能**获取这些互斥锁中的**任一个**。
- **`slots_arch_lock` 例外**：架构特定的 memslot 锁通常不涉及 SRCU 同步，**可以**在 SRCU 读侧临界区内获取。
- **MMU 通知器睡眠安全**：MMU 通知器回调必须不获取 `kvm->slots_lock` 或 `kvm->slots_arch_lock`，因为 memslot 修改在通知器静默内部等待这些锁。

### 检查要点

**报告为缺陷：**
- 在 SRCU 读侧临界区内获取 `kvm->lock`、`vcpu->mutex` 或 `kvm->slots_lock`。
- 持有 `vcpu->mutex` 然后尝试获取父 `kvm->lock`。
- 持有 `kvm->mmu_lock` 时执行任何可睡眠操作（没有 `GFP_ATOMIC` 的 `kzalloc`、`mutex_lock`、`copy_from_user`）。

---

## 内存管理与 Memslot

KVM 通过"memslot"管理客户内存。在没有适当 SRCU 保护或同步的情况下访问这些会导致**释放后使用**或过时翻译使用。

> 未能保护 memslot 迭代导致**内核恐慌**（UAF）或**虚拟机迁移期间的静默数据损坏**。

### 不变规则

- **Memslot 一致性（读侧）**：从快速路径（如页错误或指令模拟）访问客户内存映射元数据（memslot）需要非可抢占或 RCU 保护的环境。这确保读者在并发的 memslot 更新期间永远不会观察到部分更新或已释放的映射结构。在 KVM 中，这通过持有 `kvm->srcu` 实现。
- **写入者上下文例外**：仅当持有 **`kvm->slots_lock`** 时允许没有 SRCU 的访问（例如在 memslot 更新期间）。
- **更新 API**：所有对 memslot 的更改（标志、地址范围）必须通过 `kvm_set_memory_region()`。禁止手动修改 memslot 结构。

### 检查要点

**报告为缺陷：**
- 没有持有 `kvm->srcu` 的情况下访问 memslot 或调用地址翻译辅助函数（除非在写者上下文中持有 `slots_lock`）。
- 在官方更新路径之外手动翻转 memslot 标志位。

```c
// 错误：没有 SRCU 保护时访问 memslot
struct kvm_memslots *slots = kvm_memslots(vcpu->kvm);
hva = __gfn_to_hva_memslots(slots, gfn); // 潜在 UAF

// 正确：用 SRCU 保护
int idx = srcu_read_lock(&kvm->srcu);
struct kvm_memslots *slots = kvm_vcpu_memslots(vcpu);
hva = __gfn_to_hva_memslots(slots, gfn);
srcu_read_unlock(&kvm->srcu, idx);
```

---

## 失效重试协议（MMU 通知器）

当 KVM 处理页错误时，它必须与并发的宿主机 MMU 失效同步，以确保它不会安装过时的翻译。

> 缺少重试检查允许 KVM 安装**过时的翻译**，导致内存损坏或客户可见的数据泄漏。

### 不变规则

- **强制序列：**
  1. **捕获**：存储当前的 `mmu_invalidate_seq`。
  2. **解析**：将客户地址（HVA/GPA）翻译为物理帧（PFN）。
  3. **加锁**：获取 `kvm->mmu_lock`。
  4. **检查**：验证 `!mmu_invalidate_retry(kvm, captured_seq)`。
  5. **安装**：向 KVM 页表提交映射。
  6. **解锁**：释放 `kvm->mmu_lock`。
- **代次计数不变量**：重试检查结合全局 `mmu_invalidate_seq` 与正在进行的 gfn 范围；单独一个都不足够。序列计数器是主要的代次安全检查；gfn 范围减少误报。
- **加锁不变量（门控检查）**：门控安装的重试检查必须在持有 `kvm->mmu_lock` 时执行，并且锁必须保持直到翻译完全安装。通过 `mmu_invalidate_retry_gfn_unsafe()` 的预获取"不安全"重试检查是避免锁争用的合法快速路径优化；它们不能替代门控检查，审查者不应将其标记为缺陷。

### 检查要点

**报告为缺陷：**
- 仅基于不安全重试检查的结果安装映射，而没有在 `kvm->mmu_lock` 下重新检查。
- 在*捕获序列之前*或*获取锁之后*解析 PFN。
- 在重试检查和页表项安装之间释放 `kvm->mmu_lock`。

---

## VCPU 生命周期与抢占

`vcpu_load()` 和 `vcpu_put()` 管理虚拟 CPU 到物理 CPU 的附着。

> 未能管理 VCPU 附着导致**泄漏的抢占通知器**，引起调度器中的 NULL 解引用和**硬件状态损坏**。

### 不变规则

- **强制使用**：这些必须配对，并且对于与物理 CPU 相关状态（硬件寄存器、定时器、抢占通知器）交互的操作是强制性的。
- **副作用**：`vcpu_load()` 注册抢占通知器。未能调用 `vcpu_put()` 导致泄漏的通知器和 `kvm_sched_out()` 中的空指针解引用。

### 检查要点

**报告为缺陷：**
- 在修改架构或硬件切换状态的 KVM IOCTL 周围缺少 `vcpu_load()` 或 `vcpu_put()`。

---

## 快速检查清单

- **VCPU 请求**：`kvm_make_request()` 内部在设置请求位前发出 `smp_wmb()`，与 `kvm_check_request()` 中的 `smp_mb__after_atomic()` 配对。调用者不得在 `kvm_make_request()` 周围添加手动屏障。IPI 唤醒路径与完整的 `smp_mb()` 和 vCPU 的 `IN_GUEST_MODE` 标志配对。
- **更新硬件共享 SPTE 位时要保留并发的硬件写入**。当 KVM 更新硬件页表遍历器也写入的 SPTE 位时，更新不能破坏并发的硬件更新。原子单指令操作（如 x86 XCHG、原子 AND、`cmpxchg`）都是可接受的；不要求 `cmpxchg` 循环。架构当前不跟踪位的非叶子 SPTE 可以使用普通写入。GICv4 虚拟待处理状态是 ARM64 特定的；参见 kvm-arm64 指南。
- **cpus_read_lock() 放置**：`cpus_read_lock()` 是最外层的 KVM 锁。在 `kvm_lock` 外获取它是官方序，但很容易在持有 `kvm_lock` 时无意触发。审查者应标记任何在已持有 `kvm_lock` 时调用 `cpus_read_lock()` 的新路径。
- **MMU 通知器配对不变量**：每个 `invalidate_range_start()` 与恰好一个使用相同 memslots 数组的 `invalidate_range_end()` 配对。重试协议序列计数器依赖此配对；破损的配对会产生错误的"不需要重试"结果。

---

## Dirty Ring 与 Dirty Bitmap

KVM 支持两个可以并发运行的脏跟踪机制（ring 和 bitmap）。它们之间不正确的同步会导致脏信息丢失或空指针解引用。

- **Ring 到 Bitmap 刷新**：当 dirty ring 满时，脏信息必须在 ring 被清除前无条件刷新到备份 bitmap。条件刷新可能静默丢失其脏条目被覆盖的页。
- **Bitmap 范围对齐**：`KVM_CLEAR_DIRTY_LOG` 操作在一个必须按 64 位对齐的 bitmap 范围上；未对齐的范围会破坏相邻的脏状态。

### 检查要点

**报告为缺陷：**
- 条件跳过 bitmap 备份的 dirty ring 刷新路径。
- 不检查 bitmap 大小对齐的 `KVM_CLEAR_DIRTY_LOG` 处理器。

---

## PFN 缓存与私有 Memslot

- **gfn→pfn 缓存刷新协议**：`gfn_to_pfn_cache` 刷新使用与 MMU 通知器重试协议相同的序列计数器纪律。缓存刷新必须在固定页后重新检查 `mmu_invalidate_seq` 以关闭 TOCTOU 窗口。多个并发失效可以在固定期间重叠；序列计数器捕获它们全部。
- **私有 Memslot 重叠**：公共和私有（guest_memfd）memslot 在 GPA 空间中**不能**重叠。在 `KVM_SET_USER_MEMORY_REGION2` 处理期间必须强制重叠检测；重叠的私有/公共 memslot 在错误注入期间会产生 UAF 或静默数据损坏。

### 检查要点

**报告为缺陷：**
- 在固定后不重新检查失效序列的 `gfn_to_pfn_cache` 刷新路径。
- 允许私有和公共 memslot 之间 GPA 重叠的 `KVM_SET_USER_MEMORY_REGION2` 处理器。
