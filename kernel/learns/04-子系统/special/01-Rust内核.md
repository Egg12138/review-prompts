<!-- Source: subsystem/rust.md -->

# Rust 内核编程指南

## 概述

Linux 内核中的 Rust 代码遵循 `Documentation/rust/coding-guidelines.rst` 中定义的通用编码规范。本章聚焦于 Rust 子系统的特殊约定与常见陷阱。

---

## 构建假设

- Rust 代码默认已通过编译和 lint 检查（包括 Clippy lints），CI 系统负责保证这一点。
- CI 仅构建有限的内核配置组合，因此**仍需要关注条件编译问题**：在有效的 Kconfig 符号组合（`CONFIG_*`）下可能出现的编译错误或死代码（dead code）。
- Rust 不稳定特性（unstable features）可用，内核使用 `RUSTC_BOOTSTRAP=1` 启用。

---

## Bindings 与 Helpers

### rust/helpers/

`rust/helpers/` 下的文件导出内联函数或函数宏，供 Rust 代码链接使用。

- 所有 helper 函数以 `rust_helper_` 为前缀，在 `bindings` 中暴露时会去掉此前缀。
- 所有 helper 必须标注 `__rust_helper` 属性，如果新增 helper 遗漏该属性则属于缺陷。

### rust/bindings/bindings_helper.h

该文件使用 `const` 重新定义复杂宏常量，使 bindgen 可以正确转换。

- 常量以 `RUST_CONST_HELPER_` 为前缀，在 `bindings` 中暴露时前缀会被剥离。
- 如果对某个 C API 在 `bindings::*` 中的使用方式有疑虑，应阅读对应的 C 源代码确认实际要求，而非仅凭记忆。

---

## FFI 类型

在内核中，`unsigned long` 始终等同于 `uintptr_t` 和 `size_t`。因此 `ffi::c_ulong` 始终映射为 `usize`，这与用户态 Rust 不同。

---

## 内联注解

- 使用 `build_assert!()` 且依赖函数参数的函数需要标注 `#[inline(always)]`。
- **仅针对抽象层（abstractions）**：小型函数或转发到 binding 调用的函数应标注 `#[inline]`。叶子 crate（如驱动）不受此约束。

---

## Pin 初始化

`try_pin_init!(Struct { field: expr })`（或不可失败的 `pin_init!`）用于初始化需要 pinning 的 struct。

- 需要就地初始化（in-place）的字段使用 `field <- expr` 而非 `field: expr`。
- 已初始化的字段可在后续初始化中通过名称引用。
- `_: { /* any code */}` 可在字段之间插入任意代码。

---

## 常见问题

### 导入格式

- 若 commit 涉及 import，应遵循内核垂直导入风格（vertical import style），记录在通用编码规范中。
- Vendored crate（如 `syn`、`pin-init`）不受此约束。

**报告为 nit**：如果导入格式使用不正确。

### 缺少不变性注释

- 当构造具有 `# Invariants` 文档节的 struct 时，代码应包含 `// INVARIANT:` 注释，解释不变性（invariants）为何被满足，类似于 `// SAFETY:` 的用法。

**报告为缺陷**：新的抽象层中缺少 `// INVARIANT:` 注释。

**报告为 nit**：现有代码中不变性注释的不一致。
