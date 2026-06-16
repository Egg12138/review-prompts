<!-- Source: subsystem/kconfig.md -->

# Kconfig 子系统详解

## 配置符号引用

在 `select` 或 `depends on` 中引用不存在的配置符号（config symbol），会导致静默的构建失败：依赖永远无法满足，或者 select 了一个不存在的符号而没有任何效果，导致用户期望的驱动或功能被禁用。

### Invariants

- `select FOO` 和 `depends on FOO` 中的所有配置名称，必须对应内核树中某个 `config FOO` 的定义
- 注意相似前缀之间的拼写错误：`QCM_` vs `QCS_`、`IMX8M_` vs `IMX8MM_` 等
- 添加引用现有符号的新配置时，通过搜索其 `config` 定义来验证被选符号的名称

### Failure Modes

**符号不存在**：`select NON_EXISTENT_SYMBOL` 没有效果，依赖该符号的驱动永远不会被编译。构建过程不报错，但功能静默缺失。

**拼写错误**：相似前缀间的 typo（如 `IMX8M_` 误写为 `IMX8MM_`）导致引用了错误的配置。

### Quick Checks

- 验证每个 `select FOO` 引用了一个实际存在的配置

---

## 依赖传播

选择（select）一个配置符号而不继承它的依赖，会在构建时触发 Kconfig 警告（`sym_warn_unmet_dep()` in `scripts/kconfig/symbol.c`）。配置 A 可以在配置 B（A 所 select 的）无法启用的平台上被启用，产生 unmet dependency 警告和潜在的构建失败。

### Invariants

- 当 `config A` 使用 `select B`，config A 必须具有与 config B 的 `depends on` 兼容的依赖（相同或更严格）
- 常见情况：如果 B 有 `depends on ARM64 || COMPILE_TEST`，那么 A 也必须包含 `depends on ARM64 || COMPILE_TEST`

```
// 错误：缺少架构依赖
config QCS_DISPCC_615
    tristate "QCS615 Display Clock Controller"
    select QCS_GCC_615  // QCS_GCC_615 依赖 ARM64 || COMPILE_TEST
    // 缺少：depends on ARM64 || COMPILE_TEST

// 正确：正确的依赖继承（drivers/clk/qcom/Kconfig）
config QCS_DISPCC_615
    tristate "QCS615 Display Clock Controller"
    depends on ARM64 || COMPILE_TEST
    select QCS_GCC_615
```

Kconfig 警告的输出格式：
```
WARNING: unmet direct dependencies detected for QCS_GCC_615
  Depends on [n]: ARM64 || COMPILE_TEST
  Selected by [m]:
  - QCS_DISPCC_615 [=m]
```

### Failure Modes

**Unmet Dependency 警告**：config A select 了 config B，但 A 的 `depends on` 条件比 B 的更宽松，导致在某些平台上 B 无法启用而 A 却可以。Kconfig 报告 "unmet direct dependencies"。

**构建失败**：unmet dependency 导致 B 实际未被编译，A 在链接时找不到 B 提供的符号。

### Quick Checks

- 当 select 一个带有 `depends on` 的配置时，确保 selector 具有兼容的依赖
- 通过比较同一子系统中相关配置来检查命名一致性

---

## 跨配置一致性

当多个相关配置一起添加时（例如同一 SoC 系列的时钟控制器），它们之间的不一致往往指示了复制粘贴错误或 typo。

### Invariants

- 将新配置与同一文件中现有的类似配置进行比较
- 检查相关配置（如 `QCS_DISPCC_615`、`QCS_GPUCC_615`、`QCS_VIDEOCC_615`）是否遵循相同的依赖和 select 模式
- 如果一个系列中的某个配置与其他不同，验证该差异是有意的

### Failure Modes

**Copy-Paste 错误**：从现有的配置复制后修改了符号名，但忘记更新 `depends on` 或 `select` 行，导致使用了过时或不匹配的依赖。

### Quick Checks

- 检查同一子系统中相关配置的命名一致性，发现 typo

---

## COMPILE_TEST 驱动中的架构特定符号

在支持 `COMPILE_TEST` 的驱动中使用 `select` 选择架构特定符号，可能会在不支持的架构上引发 unmet dependency 警告或构建失败。`select` 语句强制启用该符号而不管其自身的依赖，如果被 selected 的符号拉入了架构特定的基础设施，编译可能失败。

### COMPILE_TEST 机制

- `COMPILE_TEST`（定义在 `init/Kconfig`）允许驱动在任何架构上被编译以进行构建覆盖测试，即使硬件只存在于某个特定架构上
- 当 Kconfig 有 `depends on ARCH_FOO || COMPILE_TEST`，该驱动可以在 `ARCH_FOO` 之外的架构上构建
- 在此类驱动中使用 `select ARCH_SPECIFIC_SYMBOL` 会在该符号自身依赖未满足的架构上触发 unmet dependency 警告

### Invariants

当 `select` 的目标具有架构特定依赖时，优先使用以下方法之一：

- 使用条件 select：
  ```
  select SOME_SUBSYSTEM if ARM64
  ```
- 将 `select` 改为 `depends on`，使驱动仅在基础设施存在时才可用：
  ```
  depends on SOME_SUBSYSTEM
  ```

### Failure Modes

**跨架构 unmet dependency**：`depends on ARCH_FOO || COMPILE_TEST` 的驱动使用了 `select ARCH_SPECIFIC_SYMBOL`，在非 `ARCH_FOO` 架构上编译时，该符号的依赖不满足，触发 Kconfig 警告。

### Quick Checks

- 当驱动使用 `depends on ... || COMPILE_TEST` 时，验证所有 `select` 语句引用的符号在所有架构上都可用

---

## Quick Checks 汇总

- **被选符号是否存在**：验证每个 `select FOO` 引用了一个实际存在的配置
- **依赖继承**：当 select 一个带有 `depends on` 的配置时，确保 selector 有兼容的依赖
- **命名一致性**：通过比较同一子系统中相关配置来检查 typo
- **COMPILE_TEST 与架构特定 select**：当驱动使用 `depends on ... || COMPILE_TEST` 时，验证所有 `select` 语句引用的符号在所有架构上都可用
