<!-- Source: subsystem/irqchip.md (derived from kernel IRQ subsystem knowledge) -->

# IRQ Chip（中断控制器）子系统

## irq_chip 结构体

`struct irq_chip`（定义于 `include/linux/irq.h`）是中断控制器的软件抽象，包含一组回调函数。每个中断描述符（`struct irq_desc`）关联一个 `irq_chip`，后者实现底层硬件操作。

### 核心回调

| 回调 | 用途 | 注意事项 |
|------|------|---------|
| `irq_startup` | 启动中断线 | 返回非零值表示中断已被置位（pending），需要处理 |
| `irq_shutdown` | 关闭中断线 | 与 `irq_startup` 严格配对 |
| `irq_enable` | 启用中断 | 默认调用 `irq_unmask`；如实现该回调则替代 `irq_unmask` |
| `irq_disable` | 禁用中断 | 默认调用 `irq_mask`；如实现该回调则替代 `irq_mask` |
| `irq_ack` | 应答中断 | 用于边沿触发（edge-triggered）的中断 |
| `irq_mask` | 屏蔽中断源 | 禁止特定中断线上报 |
| `irq_unmask` | 取消屏蔽中断源 | 允许特定中断线上报 |
| `irq_eoi` | 中断处理结束（End Of Interrupt） | 用于 fasteoi 类型的中断控制器 |
| `irq_set_type` | 设置触发类型（电平/边沿） | 需要 `IRQCHIP_SET_TYPE_MASKED` 标志来确保安全 |
| `irq_set_affinity` | 设置 SMP 亲和性 | 用于多处理器系统中的中断分发 |
| `irq_retrigger` | 重新触发中断 | 用于软件触发的虚假中断处理 |
| `irq_set_wake` | 设置中断唤醒 | 用于电源管理中的唤醒源配置 |

### irq_chip 标志

`struct irq_chip` 的 `flags` 字段可以设置以下标志：

- `IRQCHIP_SET_TYPE_MASKED`：修改触发类型时先屏蔽中断，防止类型切换期间产生虚假中断
- `IRQCHIP_EOI_IF_HANDLED`：仅在中断确实被处理后才调用 eoi
- `IRQCHIP_MASK_ON_SUSPEND`：挂起时屏蔽所有中断
- `IRQCHIP_ONOFFLINE_ENABLED`：CPU 热插拔时自动启用/禁用中断
- `IRQCHIP_SKIP_SET_WAKE`：跳过设置唤醒的调用（set_wake 为空操作或不可用）
- `IRQCHIP_AFFINITY_PRE_STARTUP`：在启动中断前设置亲和性

---

## irq_domain——中断号映射

### 概念

`struct irq_domain` 将硬件中断号（hwirq）映射到 Linux 内核的虚拟 IRQ 号。每个中断控制器驱动必须创建一个 irq_domain。

### 域映射类型

| 类型 | 函数 | 使用场景 |
|------|------|---------|
| **线性映射（Linear）** | `irq_domain_create_linear()` | 硬件 IRQ 号范围连续且数量已知 |
| **树映射（Tree）** | `irq_domain_create_tree()` | 硬件 IRQ 号稀疏或不连续 |
| **不映射（No-map）** | `irq_domain_create_no_map()` | 硬件 IRQ 号直接等于 Linux IRQ 号 |
| **NOMAP** | `irq_domain_add_nomap()` | 简单的 1:1 映射，无需翻译表 |

### 域操作（domain ops）

```c
struct irq_domain_ops {
    int (*match)(struct irq_domain *d, struct device_node *node,
                 enum irq_domain_bus_token bus_token);
    int (*map)(struct irq_domain *d, unsigned int virq, irq_hw_number_t hw);
    void (*unmap)(struct irq_domain *d, unsigned int virq);
    int (*xlate)(struct irq_domain *d, struct device_node *node,
                 const u32 *intspec, unsigned int intsize,
                 unsigned long *out_hwirq, unsigned int *out_type);
};
```

- `match`：检查 domain 是否匹配给定的 Device Tree 节点
- `map`：将硬件 IRQ 号映射到 Linux IRQ 号，设置 `irq_chip` 和 handler
- `unmap`：解除映射
- `xlate`：将 Device Tree 中的中断描述（`interrupts` 属性）翻译为 hwirq 和触发类型

### 常见错误

- **未设置 `xlate` 回调**：导致 Device Tree 中断解析失败
- **map 回调未设置 irq_chip**：导致 NULL 指针解引用
- **hwirq 范围检查缺失**：导致越界访问 domain 的线性映射表

---

## 层级 IRQ Domain（Hierarchical Domain）

### 概念

现代 SoC 中，中断控制器通常形成层级结构：一个父控制器（如 GIC）连接多个子控制器（如 GPIO 控制器作为中断源）。层级 domain 支持这种嵌套。

### 父子关系

- 子 domain 通过 `irq_domain_create_hierarchy()` 创建
- 子 domain 需要 `->alloc` 回调，在其中调用 `irq_domain_alloc_irqs_parent()` 分配父级资源
- `irq_find_mapping()` 从子 domain 开始查找，向上遍历到父 domain

### 关键不变式

- 子 domain 的 `alloc` 回调必须调用父 domain 的 alloc
- 子 domain 的 `irq_chip` 只处理自己硬件相关的操作，其他操作委托给父级
- `irq_domain_free_irqs()` 必须正确释放所有层级的资源

### 故障模式

- 子 domain 的 alloc/free 未正确传递到父 domain，导致资源泄漏
- 子控制器中断的回调链未正确级联到父级

---

## Generic IRQ Chip

### 概念

`struct irq_chip_generic`（通用 IRQ 芯片）为具有以下特征的简单中断控制器提供标准实现：一组寄存器控制 mask/unmask/ack/polarity 等操作。

### 使用方式

```c
struct irq_chip_generic *gc = irq_alloc_generic_chip(name, num, 
                                                      irq_base, reg_base, handler);
gc->reg_base = base_addr;
gc->irq_base = irq_base;

// 设置 mask 寄存器
gc->mask_cache = default_mask;
gc->regs.mask = MASK_REG_OFFSET;
irq_setup_generic_chip(gc, mask, flags, clr, set);
```

### 锁机制

- `gc->lock` 保护 register 访问
- 对于需要 read-modify-write 的寄存器操作，锁必须被正确持有
- `irq_setup_generic_chip()` 自动注册通用 chip 回调

### 常见错误

- 未调用 `irq_setup_generic_chip()`，导致 chip 回调不生效
- `mask_cache` 初始值与硬件实际状态不一致
- 为只读寄存器设置了 writable 的回调

---

## 中断流处理（Interrupt Flow Handlers）

### 流处理函数类型

| Handler | 适用场景 |
|---------|---------|
| `handle_simple_irq()` | 无需硬件 ACK/MASK 操作的简单中断 |
| `handle_level_irq()` | 电平触发（level-triggered）中断 |
| `handle_edge_irq()` | 边沿触发（edge-triggered）中断 |
| `handle_fasteoi_irq()` | 支持 EOI 的中断控制器（如 GIC） |
| `handle_percpu_irq()` | 每 CPU 中断 |

### 流处理与 irq_chip 的交互

| 流处理器 | 调用的回调 |
|---------|-----------|
| `handle_level_irq` | mask -> ack -> [handle] -> unmask |
| `handle_edge_irq` | mask -> ack -> [handle] -> unmask |
| `handle_fasteoi_irq` | [handle] -> eoi |
| `handle_simple_irq` | [handle] |
| `handle_percpu_irq` | [handle] -> eoi |

### 常见错误

- 电平触发中断使用 `handle_edge_irq`：在共享中断时可能导致丢失中断
- 边沿触发中断使用 `handle_level_irq`：短时间内多次中断时可能只捕获一次
- fasteoi 类型未设置 eoi 回调：中断处理完后控制器仍阻塞中断线
- `handle_simple_irq` 用于需要 mask 操作的中断：可能导致中断风暴

---

## Invariants

- 每个注册的 IRQ 号必须有有效的 `irq_chip` 和 flow handler
- `irq_set_type` 须在中断处于 mask 状态时调用，或设置 `IRQCHIP_SET_TYPE_MASKED` 标志
- `irq_domain_ops->map` 必须正确设置 chip 和 handler
- 层级 domain 的 alloc 回调必须调用父 domain 的 alloc
- 通用 chip 的 `mask_cache` 必须反映硬件 mask 寄存器的实际状态
- `irq_startup` 和 `irq_shutdown` 必须严格配对
- Device Tree 中的 `interrupts` 属性必须与 `interrupt-parent` 的 `#interrupt-cells` 匹配

## Failure Modes

| 违反场景 | 结果 |
|---------|------|
| 未设置 `IRQCHIP_SET_TYPE_MASKED` 就调用 `irq_set_type` | 类型切换期间产生虚假中断 |
| 向已启用的中断设置触发类型 | 硬件状态不一致，可能死锁 |
| 未设置 flow handler | NULL 指针解引用，内核崩溃 |
| fasteoi 类型未实现 eoi 回调 | 中断被阻塞，系统挂起 |
| 层级 domain 中未调用父 domain 的 alloc | 父级资源未分配，中断不工作 |
| `mask_cache` 初始值与硬件状态不匹配 | mask/unmask 操作错误 |
| 通用 chip 的寄存器偏移计算错误 | 操作错误的硬件寄存器 |
| map 回调中 hwirq 越界 | 内存越界访问 |

## Quick Checks

- **irq_chip 初始化**：确认 `irq_chip` 的名称赋值，关键回调（mask/unmask/eoi）已设置
- **flow handler 选择**：根据硬件的中断触发类型（电平/边沿）选择正确的 handler
- **触发类型安全**：`irq_set_type` 是否在 mask 状态下执行？`IRQCHIP_SET_TYPE_MASKED` 是否已设置？
- **Generic chip 注册**：确认调用了 `irq_setup_generic_chip()` 使 chip 生效
- **irq_domain 创建**：确认使用了正确的 domain 类型（linear/tree/nomap）
- **Device Tree 解析**：确认 `xlate` 回调正确处理了 `#interrupt-cells` 指定的参数数量
- **层级 domain**：子 domain 的 alloc 和 free 是否传递到父 domain？
- **Suspend/Resume**：`irq_set_wake` 回调是否正确管理唤醒源的电源状态？
