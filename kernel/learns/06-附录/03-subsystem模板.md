<!-- Source: subsystem-template.md -->

# Subsystem 模板

本文档描述了子系统 `.md` 文件的格式。在编写新的子系统指南或重构现有指南时使用。

## 用途

每个子系统指南是一个**知识参考**——包含特定内核子系统的不变量 (invariants)、API 契约、结构体字段语义和常见 bug 模式。当补丁涉及该子系统时，在审查期间加载使用。

### 子系统指南不是

- 工作流或分析流程（列出步骤的检查清单）
- 需要机械式逐一检查的清单
- 存放泛型内核知识的地方（这些属于 `technical-patterns.md`）

## 文件结构

```
# <Name> Subsystem Details

## <Concept Section>

<Consequence paragraph>

<Rules, invariants, API details>

## <Concept Section>

...

## Quick Checks

<Short items not covered in detail above>
```

### 标题

固定为 `# <Name> Subsystem Details`。

### 概念章节

按概念或主题组织，而非编号模式。每个章节覆盖子系统的一个连贯领域。

**不要**使用 `SCHED-001`、`NET-002` 等内联模式 ID 作为章节标题。如果某个章节覆盖了之前编号模式的内容，可以在章节内保留 ID 作为子标题（例如 `### BT-001: Extent Map Field Confusion`）用于交叉引用，但章节本身应以概念命名。

### 后果段落

每个章节应以 1-3 句话开头，说明违反该章节规则会导致**什么后果**。陈述具体的后果：死锁、释放后使用、数据损坏、NULL 解引用、静默错误行为等。这给读者提供理解章节重要性的即时上下文。

**现有指南示例：**

> Accessing bio data fields on a bio that has no data buffers (e.g., discard, flush) causes a NULL pointer dereference.

> Incorrect PTE flag combinations cause data corruption (dirty data silently dropped), security holes (writable pages that should be read-only), and kernel crashes on architectures that trap invalid combinations.

> Using the wrong lock type for the execution context causes deadlocks (sleeping in atomic context), missed wakeups, or priority inversion.

### 规则和细节

后果段落之后，使用以下方式呈现实际规则、不变量和 API 细节：

- **要点列表** — 用于规则、不变量和 API 描述。所有函数名、类型名、字段名、宏和常量使用反引号
- **表格** — 用于多个条目共享相同属性集的参考数据。在表格前解释非显而易见的列含义
- **粗体子标题** (`**Like this:**`) — 用于章节内的子主题。仅在子主题足够重要时才使用 `###` 子标题
- **代码示例** — 当正确的 vs 错误的模式不明显或 bug 是微妙的顺序问题时使用。使用 `// CORRECT` 和 `// WRONG` 注释
- **ASCII 图示** — 仅当能阐明散文无法有效传达的空间或时间关系时使用
- **内核源码引用** — 包含函数名和文件路径，让审查者可以对照源码验证声明
  - **绝不使用行号**：行号会随时间变化
- **编号列表** — 用于顺序流程或有序生命周期

### 常见错误和 Bug 模式

当某个章节覆盖了代码经常出错的模式时，解释**为什么**这个错误难以发现。

使用 **`REPORT as bugs`**（粗体）标记特定的高信号模式，当发现时应始终报告。

### Quick Checks

最后一个章节。简短要点，涵盖上面章节未详细说明的审查陷阱。每个条目加粗命名并附简短说明：

> - **Lock drop and reacquire**：当锁被释放并重新获取时，验证代码在重新获取后重新验证所有受保护的状态

不要重复上面已详细解释的条目。Quick Checks 用于不需要独立章节的额外条目。

## 不要包含的内容

- **Risk / When to check / Details 样板** — 旧的模式格式。后果段落取代 Risk，章节内容取代 Details
- **TodoWrite 工作流步骤** — 分析流程属于代理提示（例如 `agent/review.md`、`callstack.md`），不属于子系统知识文件
- **泛型内核知识** — "不要在原子上下文中睡眠"或"检查返回值"等主题属于 `technical-patterns.md`，而非每个子系统指南
- **单次提交修复** — 仅适用于一个特定 bug 修复的知识不应在此。每个章节应描述一个**可重用的不变量、API 契约或 bug 模式**，适用于多个调用点或未来补丁
- **供应商特定的驱动细节** — 寄存器名、影子寄存器号、芯片特定的初始化序列属于驱动注释或供应商文档

## 新指南检查清单

1. 标题是 `# <Name> Subsystem Details`
2. 每个章节以后果段落开头
3. 所有函数/类型/字段/宏名使用反引号
4. 表格在含义不显而易见的列名处有列说明
5. 没有编号模式 ID 作为顶级标题
6. 没有 Risk/Details/When-to-check 样板
7. 没有 TodoWrite 或工作流步骤
8. 没有单次提交特定的知识——每个章节必须可重用
9. 末尾有 Quick Checks 章节（如果适用）
10. 代码示例使用 `// CORRECT` / `// WRONG` 标签
11. 已添加到 `subsystem.md` 触发表
