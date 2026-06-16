<!-- Source: subsystem/dt-bindings.md -->

# Device Tree Bindings 子系统详解

## Compatible String 条件块

在 YAML binding schema 中添加新的 compatible string（兼容字符串）但未更新所有现有的 `if-then` 条件块，会导致 schema 验证不完整。包含无效配置（错误数量的 interrupts、clocks 或必需的 properties）的设备树将静默通过验证。

### 条件块机制

YAML binding schemas 使用 `allOf` 搭配 `if-then` 块，根据 compatible string 应用不同的约束。`if` 子句通常通过 `properties:compatible:contains:enum:` 或 `properties:compatible:contains:const:` 进行匹配。当新的 compatible string 遵循代际模式时（如 `vendor,device-gen5` 添加到现有的 `vendor,device-gen2/gen3/gen4` 旁），如果新硬件共享相同的约束，每一个枚举了先前代系的 `if` 块都必须包含新字符串。

### 条件块通常守卫的属性

| 属性 | 典型约束 |
|----------|-------------------|
| `interrupts` | `maxItems`、`minItems` |
| `clocks` | 数量和顺序 |
| `resets` | Reset 线数量 |
| `power-domains` | Power domain 数量 |
| `required` | 哪些属性必须存在 |
| `reg` | 寄存器区域的数量和含义 |

### Invariants

- 添加到顶层 `compatible` 定义的新 compatible string，如果硬件与先前代系共享相同的约束，则必须同时添加到所有列举了先前代系的 `if:properties:compatible:contains:enum:` 块中
- 遗漏是 bug

### Failure Modes

**验证遗漏**：新的 compatible string 添加到了 top-level 的 `compatible` 枚举中，但未包含在某个 `if` 块的 `enum` 列表中。该设备树的 interrupts、clocks 或其他约束无法得到正确验证，无效的配置被静默接受。

### Quick Checks

- 当添加了带有代际标记（gen5、v5、series-5 等）的 compatible string 时，必须检查所有引用先前代系的 `if` 块是否包含新的字符串
- 当 binding 为一个设备族中的不同设备类型有多个 YAML 文件时，相关文件可能需要同步更新

---

## 硬件变体的必需属性

为具有额外能力（GPIO 控制器、PWM 输出、clock provider、interrupt controller）的硬件变体添加 compatible string，但未记录相应的必需属性，会导致不完整的设备树节点通过 schema 验证。在运行时，驱动程序或依赖子系统在尝试使用未文档化的功能时会失败。

### 能力与对应的必需属性

当硬件获得新的 provider 能力时，binding 必须在 `required` 列表中添加对应的标准属性并定义其约束：

| 能力 | 必需属性 |
|------------|---------------------|
| GPIO controller | `gpio-controller`、`#gpio-cells` |
| PWM output | `#pwm-cells` |
| Clock provider | `#clock-cells` |
| Interrupt controller | `interrupt-controller`、`#interrupt-cells` |
| Reset provider | `#reset-cells` |

### Invariants

- 每个 cell-count 属性必须具有与硬件匹配的 `const` 约束（如 `#gpio-cells: const: 2`）
- `examples` 部分必须包含所有必需属性，以通过 `dt_binding_check`
- 如果单 YAML 文件中的条件块因为变体差异变得难以管理，应将 binding 拆分为独立的 YAML 文件（参考 `Documentation/devicetree/bindings/example-schema.yaml` 中 `allOf` 部分的注释："If the conditionals become too unwieldy, then it may be better to just split the binding into separate schema documents"）

### Failure Modes

**运行时驱动失败**：设备树节点缺少必需的属性（如缺少 `#gpio-cells`），但 schema 验证通过了。驱动程序在尝试使用 GPIO 功能时失败。

**dt_binding_check 失败**：`examples` 部分遗漏了必需属性，导致 `make dt_binding_check` 无法通过。

### Quick Checks

- 当 variant compatible string 添加了 provider 能力（GPIO、PWM、clock、interrupt、reset）时，对应的属性必须出现在 `required` 列表中并带有适当的 `const` 约束

---

## `$id` 路径一致性

`$id` 字段错误会破坏 schema 的交叉引用系统。来自其他 binding 的 schema 引用（`$ref`）将无法解析，`dt_binding_check`（定义在 `Documentation/devicetree/bindings/Makefile` 中的 make 目标）可能报告误导性错误或静默跳过验证。

### 规则

`$id` 字段必须以 `http://devicetree.org/schemas/` 开头（参见 `Documentation/devicetree/bindings/writing-schema.rst`）。该前缀之后的路径必须与文件相对于 `Documentation/devicetree/bindings/` 的路径完全匹配。

```yaml
# 文件：Documentation/devicetree/bindings/gpio/vendor,device.yaml

# 正确
$id: http://devicetree.org/schemas/gpio/vendor,device.yaml#

# 错误 - 缺少子目录组件
$id: http://devicetree.org/schemas/vendor,device.yaml#
```

### Invariants

- `$id` 的值必须能通过路径前缀推导出来：`http://devicetree.org/schemas/` + 文件在 binding 目录下的相对路径

### Failure Modes

**子目录遗漏**：复制粘贴时忘记将 `gpio/`、`power/`、`clock/` 等子目录包含在 `$id` 路径中，使 `$ref` 引用无法解析。

**文件名过时**：从 `.txt` 转换为 `.yaml` 后，`$id` 仍然使用旧文件名或路径。

**Copy-Paste 错误**：从其他 binding 复制后忘记更新 `$id` 中的路径。

### Quick Checks

- `$id` 路径必须与文件位置匹配；这在 `.txt` 到 `.yaml` 的转换过程中尤其容易出错

---

## Quick Checks 汇总

- 当添加了带有代际标记（gen5、v5、series-5 等）的 compatible string 时，必须检查所有引用先前代系的 `if` 块是否包含了新字符串
- 当 binding 为一个设备族中的不同设备类型有多个 YAML 文件时，相关文件可能需要同步更新
- 当 variant compatible string 添加了 provider 能力（GPIO、PWM、clock、interrupt、reset）时，对应的属性必须出现在 `required` 列表中并带有适当的 `const` 约束
- `$id` 路径必须与文件位置匹配；这在 `.txt` 到 `.yaml` 的转换过程中尤其容易出错
