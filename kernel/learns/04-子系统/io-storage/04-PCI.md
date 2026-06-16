<!-- Source: subsystem/pci.md -->

# PCI 子系统详解

## PCI Endpoint 错误返回约定（PCI Endpoint Error Return Conventions）

将错误指针（error pointer）传递给期望有效 `struct pci_epc *` 或 `struct pci_epf *` 的函数会导致内核崩溃。例如，`pci_epf_destroy()` 无条件解引用其参数（调用 `device_unregister()`），因此向它传递 `ERR_PTR` 值会导致崩溃。注意 `pci_epc_put()` 可以安全处理 `ERR_PTR` 值，因为它在继续前检查了 `IS_ERR_OR_NULL()`。

以下 PCI endpoint 函数在失败时返回 `ERR_PTR()`，**而非 NULL**：

- `pci_epc_get()`（`drivers/pci/endpoint/pci-epc-core.c`）——失败时返回 `ERR_PTR(-EINVAL)`；使用 `IS_ERR()` 检查，而不是 `!ptr`
- `pci_epf_create()`（`drivers/pci/endpoint/pci-epf-core.c`）——失败时返回 `ERR_PTR(-ENOMEM)` 或其他错误码；使用 `IS_ERR()` 检查，而不是 `!ptr`

## Failure Modes（失败模式）

- 将 `pci_epc_get()` 或 `pci_epf_create()` 返回的 `ERR_PTR` 传递给 `pci_epf_destroy()` → 内核崩溃。
- 使用 `!ptr` 而非 `IS_ERR()` 检查这些函数的返回值 → 漏检错误，传递无效指针。

---

## 传统 PCI MSI API（Legacy PCI MSI APIs）

使用传统 MSI API 的新代码会缺少 MSI-X 支持和现代 IRQ 向量接口的错误处理灵活性。内核源码（`drivers/pci/msi/api.c`）显式将 `pci_enable_msi()` 和 `pci_disable_msi()` 标记为"传统设备驱动 API"，指示调用者改用 `pci_alloc_irq_vectors()` / `pci_free_irq_vectors()`。

**传统 API（已废弃）：**
- `pci_enable_msi()` / `pci_disable_msi()` —— 已被通用 IRQ 向量分配接口取代

**现代替代：** 使用 `pci_alloc_irq_vectors()` 和 `pci_free_irq_vectors()`：

```c
// 错误 - 传统 API，不支持 MSI-X
ret = pci_enable_msi(pdev);
if (ret)
    return ret;

// 正确 - 支持 MSI、MSI-X 和传统 INTx 中断
ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_ALL_TYPES);
if (ret < 0)
    return ret;
```

---

## PCI IRQ 向量清理（PCI IRQ Vector Cleanup in Error Paths）

在 `pci_alloc_irq_vectors()` 成功后，如果在错误路径中未调用 `pci_free_irq_vectors()`，会泄漏 IRQ 资源，阻止未来的分配并可能耗尽系统 IRQ 容量。

## Invariants（不变规则）

每个在 `pci_alloc_irq_vectors()` 成功后的错误路径，必须在返回前调用 `pci_free_irq_vectors()`。

```c
// 错误：初始化失败时 IRQ 向量泄漏
ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_ALL_TYPES);
if (ret < 0)
    return ret;

ret = some_init(pdev);
if (ret)
    return ret;  // BUG: IRQ 向量未释放

// 正确：错误时正确清理
ret = pci_alloc_irq_vectors(pdev, 1, 1, PCI_IRQ_ALL_TYPES);
if (ret < 0)
    return ret;

ret = some_init(pdev);
if (ret)
    goto free_irq;

return 0;

free_irq:
    pci_free_irq_vectors(pdev);
    return ret;
```

---

## 设备命名约定（Device Naming Conventions）

驱动从其自身重命名父设备或总线设备会导致 sysfs 中的混乱，破坏期望标准命名的用户空间工具，并干扰 PCI 子系统的设备管理。

## Invariants（不变规则）

驱动**不得**在其父 PCI 设备上调用 `dev_set_name()`。PCI 子系统拥有设备命名权。

```c
// 错误：驱动重命名其父 PCI 设备
static int driver_probe(struct pci_dev *pdev)
{
    struct device *dev = &pdev->dev;
    dev_set_name(dev, "my-device");  // 不要这样做
}

// 正确：命名驱动拥有的新创建子设备
struct device *child = kzalloc(sizeof(*child), GFP_KERNEL);
dev_set_name(child, "child-%d", id);
```

---

## Quick Checks（快速检查要点）

- **EPC/EPF 返回值**：确认 `pci_epc_get()` 和 `pci_epf_create()` 的返回值用 `IS_ERR()` 检查，而非 `!ptr`；且错误指针不传递给 `pci_epf_destroy()`
- **传统 MSI API**：标记新代码中对 `pci_enable_msi()` 和 `pci_disable_msi()` 的使用
- **IRQ 向量清理**：`pci_alloc_irq_vectors()` 成功后，确认所有错误路径调用 `pci_free_irq_vectors()`
- **设备命名**：确认 `dev_set_name()` 不是在 `&pdev->dev` 或其他总线拥有的设备结构上调用
