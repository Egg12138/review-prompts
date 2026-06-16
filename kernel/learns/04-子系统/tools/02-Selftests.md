<!-- Source: subsystem/selftests.md -->

# Selftests 子系统详解

## 构建系统与安装路径

在 selftests 目录中创建了新文件但未添加到 Makefile，当通过 `make install` 从安装位置运行时，测试会以 "No such file or directory" 失败。如果直接从源码树运行，由于文件在源码目录中存在，测试可能看起来工作正常。

### Makefile 变量

selftests 构建系统在各子系统的 Makefile 中使用以下变量来控制安装内容：

| 变量 | 用途 |
|----------|---------|
| `TEST_PROGS` | 可直接运行的测试脚本（executable test scripts） |
| `TEST_FILES` | 支持文件（库文件、数据文件、被 source 的脚本） |
| `TEST_GEN_FILES` | 构建过程中生成的二进制/文件 |
| `TEST_GEN_PROGS` | 生成的可执行测试程序 |

### Invariants

- 测试脚本中通过 `source <filename>`（bash）或 `. <filename>` 引用的文件，必须添加到 `TEST_FILES`
- 测试脚本中通过 `import <module>`（Python）引用的模块文件，必须添加到 `TEST_FILES`
- 被直接调用的可执行测试脚本放入 `TEST_PROGS`
- 在 `make` 过程中构建的辅助可执行文件放入 `TEST_GEN_PROGS` 或 `TEST_GEN_FILES`

### Failure Modes

**安装后文件缺失**：最常见的错误——创建了新的共享库或工具文件（如 `_common.sh`、`utils.py`、`lib.sh`），被测试脚本 source 或 import，但忘记添加到 `TEST_FILES`。在源码目录下测试工作正常，但 `make install` 后测试失败，报 "No such file or directory"。

**变量混淆**：将可执行测试放到 `TEST_FILES`（应该放 `TEST_PROGS`），或者将辅助文件放到 `TEST_PROGS`（应该放 `TEST_FILES`）。前者导致测试不会被自动执行，后者导致非可执行文件被当作测试运行而失败。

### Quick Checks

- **新共享文件**：当 commit 创建了被测试脚本 source 或 import 的文件时，验证它已添加到 Makefile 的 `TEST_FILES`
- **`TEST_PROGS` vs `TEST_FILES`**：可执行测试放 `TEST_PROGS`，支持文件放 `TEST_FILES`，混淆会导致执行失败或安装遗漏

---

## KVM Selftests：IRQ 芯片设置与 `vm_create` vs `vm_create_with_one_vcpu`

使用 `KVM_IRQFD`、`KVM_IRQ_LINE` 或 IRQ routing API 的测试如果在 `vm_create()` 之后调用，会因为 `vm_create()` 不创建 vCPU 而失败——在 arm64 上，VGIC 的最终化（`KVM_DEV_ARM_VGIC_CTRL_INIT`）要求所有 vCPU 已先创建。在不支持内核内 IRQ 芯片的架构（riscv、loongarch）上，这些 ioctl 会返回 `-ENODEV`。

### 关键区别

`vm_create(nr_runnable_vcpus)` 分配了一个 VM 并为给定数量的 vCPU 分配了内存，但**并没有创建**任何 vCPU。IRQ 芯片设置通过 `kvm_arch_vm_post_create()` 在 `vm_create()` 期间启动，但最终化（通过 `kvm_arch_vm_finalize_vcpus()`）**只**发生在同时创建 vCPU 的函数中，如 `vm_create_with_one_vcpu()` 和 `__vm_create_with_vcpus()`。

### 架构默认 IRQ 芯片支持

`kvm_arch_has_default_irqchip()` 返回该架构是否默认设置了内核内 IRQ 芯片：

| 架构 | 返回值 |
|--------------|-------------|
| x86 | `true`（通过 `vm_create_irqchip()` 创建 IOAPIC/PIC/LAPIC） |
| s390 | `true` |
| arm64 | `true`——当 GICv3 受支持且未被 `test_disable_default_vgic()` 禁用时 |
| riscv、loongarch | `false`（`lib/kvm_util.c` 中的 weak 默认实现） |

### Invariants

需要内核内 IRQ 芯片的测试必须：

1. 调用 `TEST_REQUIRE(kvm_arch_has_default_irqchip())` 以在不支持 IRQ 芯片的架构上跳过测试
2. 使用 `vm_create_with_one_vcpu()`（或 `__vm_create_with_vcpus()`）而不是裸 `vm_create()`，以确保 vCPU 被创建且 IRQ 芯片最终化完成，之后才能发起 IRQ 相关的 ioctl

```c
// 错误：vm_create() 不创建 vCPU，也不最终化 IRQ 芯片
vm = vm_create(1);
kvm_irqfd(vm, gsi, eventfd, 0);

// 正确：跳过不支持的架构，然后创建带 vCPU 的 VM
TEST_REQUIRE(kvm_arch_has_default_irqchip());
vm = vm_create_with_one_vcpu(&vcpu, NULL);
kvm_irqfd(vm, gsi, eventfd, 0);
```

### Failure Modes

**ARM64 VGIC 最终化失败**：使用 `vm_create()` 后调用 IRQ 相关 ioctl，因为 vCPU 尚未创建，VGIC 无法最终化，ioctl 返回错误。

**不支持的架构上执行**：在 riscv 或 loongarch 上运行需要 IRQ 芯片的测试时缺少 `TEST_REQUIRE` 防护，ioctl 返回 `-ENODEV`。

### Quick Checks

- **KVM IRQ 芯片测试**：当测试使用 `KVM_IRQFD`、`KVM_IRQ_LINE` 或 IRQ routing 时，验证使用了 `vm_create_with_one_vcpu()` 且包含了 `TEST_REQUIRE(kvm_arch_has_default_irqchip())`
