<!-- Source: subsystem/libbpf.md -->

# Libbpf 公共 API 错误处理

## 概述

Libbpf 公共 API 函数（标记为 `LIBBPF_API`）遵循严格的 `errno` 约定：所有错误路径必须设置 `errno`。用户态调用者依赖函数返回错误值时 `errno` 被正确设置——返回负值或 NULL 而不设置 `errno` 会静默破坏错误处理。

---

## errno 约定

三个包装函数定义在 `tools/lib/bpf/libbpf_internal.h` 中：

| 函数 | 适用场景 | 行为 |
|---|---|---|
| `libbpf_err(ret)` | 返回整数的 API | 若 `ret < 0`，设置 `errno = -ret`，返回 `ret` |
| `libbpf_err_ptr(err)` | 返回指针的 API（已知错误码） | 设置 `errno = -err`，返回 `NULL` |
| `libbpf_ptr(ret)` | 返回指针的 API（包装返回 `ERR_PTR` 的内部函数） | 若 `ret` 是错误指针（error pointer），设置 errno 并返回 `NULL`；否则直接返回 `ret` |

### 核心规则

- 包装函数必须在 `return` 语句本身调用，而不是在函数中提前使用。因为 errno 必须在返回到调用者之前**立即**设置。
- 内部/静态函数（static functions）不应使用这些包装函数——它们内部使用内核风格的负错误码或 `ERR_PTR`。

---

## 哪些函数需要包装

| 函数类型 | 是否需要包装 |
|---|---|
| 公共 API（`LIBBPF_API` 或列在 `libbpf.map` 中） | **所有错误返回都必须使用包装函数** |
| 内部/静态函数 | 不使用包装函数 |

---

## 返回类型模式

### 整数返回

```c
// 直接返回错误码
return libbpf_err(-EINVAL);

// 传播内部函数的错误
err = internal_func();
if (err)
    return libbpf_err(err);
```

### 指针返回（从错误码）

```c
// 已知错误码
return libbpf_err_ptr(-ENOMEM);

// 存储的错误值
return libbpf_err_ptr(err);
```

### 指针返回（从 ERR_PTR 内部函数）

```c
// 内部函数在失败时返回 ERR_PTR
return libbpf_ptr(internal_func_returning_ptr());
```

---

## 常见错误模式

| 错误写法 | 正确写法 | 问题说明 |
|---|---|---|
| `return -EINVAL;` | `return libbpf_err(-EINVAL);` | 未设置 errno |
| `return NULL;` | `return libbpf_err_ptr(-ESOMETHING);` | 未设置 errno，调用者无法确定错误原因 |
| `return ERR_PTR(-EINVAL);` | `return libbpf_err_ptr(-EINVAL);` | 公共 API 必须返回 NULL（调用者检查 NULL），不能返回 ERR_PTR |
| 函数中包装了错误值，但后续路径未包装 | 所有路径都使用包装 | 路径遗漏导致 errno 设置不一致 |

---

## 关键区分

记住两条黄金法则：

1. **公共 API 边界**：所有错误返回必须通过 `libbpf_err()`、`libbpf_err_ptr()` 或 `libbpf_ptr()`。
2. **内部函数**：不应使用这些包装函数，它们使用内核风格的负错误码或 `ERR_PTR` 内部传递错误。

**报告为缺陷**：公共 libbpf API 函数返回错误值（负整数或 NULL）时未经过 `libbpf_err()`、`libbpf_err_ptr()` 或 `libbpf_ptr()`。
