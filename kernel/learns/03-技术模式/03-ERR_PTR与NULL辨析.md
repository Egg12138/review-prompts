<!-- Source: technical-patterns.md -->

# ERR_PTR 与 NULL 辨析

## 问题：为什么内核需要区分 "错误" 和 "空"

在 Linux 内核中，很多函数通过 **返回指针** 来传递结果。但指针只有一个返回值，而调用者需要区分三种情况：

1. **成功**——返回一个有效指针
2. **失败（预期中的）**——例如 "没找到"，返回 **NULL**
3. **失败（异常错误）**——例如 "内存不足"、"权限拒绝"，需要返回一个 **错误码**

单一返回值无法同时传达 "这是一个指针" 和 "这是错误码"。内核的解决方案是 `ERR_PTR()` 机制。

---

## ERR_PTR 的机制

```c
#define ERR_PTR(err)    ((void *)((long)(err)))
```

`ERR_PTR(err)` 将一个负数错误码（如 `-ENOMEM`）转换为一个看起来像指针的值，但它指向的地址 **是无效的**。

关键问题来了：

```c
foo = ERR_PTR(-ENOMEM);
if (foo) → TRUE（因为 ERR_PTR 返回的值不是零！）
但 *foo 会崩溃（因为它不是有效地址）
```

**这就是内核 API 中最容易出错的陷阱之一**：

- NULL 表示 "没有对象"（空指针，值为 0）
- ERR_PTR 表示 "发生了错误"（值为一个负数错误码经类型转换后的值，非零）
- 两者都是检查指针有效性的方式，但 **不能互换使用**

---

## 为什么混淆 NULL 和 ERR_PTR 会出 bug

### 场景一：用 `if (ptr)` 检查 ERR_PTR 返回值

```c
struct dentry *d = __d_alloc(...);  // 可能返回 ERR_PTR 或 NULL
if (!d)                             // 只检查了 NULL，漏掉了 ERR_PTR
    return -ENOMEM;
// 如果 d 是 ERR_PTR(-EINVAL)，会走到这里然后解引用 ERR_PTR → 崩溃
```

### 场景二：用 IS_ERR() 检查 NULL

```c
struct page *p = alloc_pages(...);   // 失败时返回 NULL，不会返回 ERR_PTR
if (IS_ERR(p))                        // IS_ERR(NULL) 是 FALSE
    return PTR_ERR(p);               // 走到这里 → PTR_ERR(NULL) = 0（误以为成功）
```

### 场景三：将 ERR_PTR 当作 NULL 使用

```c
if (IS_ERR(foo))
    foo = NULL;                      // 将错误转为 "空"——丢失了错误信息
```

虽然这不会崩溃，但调用者拿到 NULL 后不知道具体错误原因，可能做了错误的处理决定。

---

## 内核提供的检查宏

### `IS_ERR(ptr)` —— 检查是否 ERR_PTR

```c
bool IS_ERR(const void *ptr);
```

- 返回 `true` 当 `ptr` 是一个 **ERR_PTR**（值在 `[-MAX_ERRNO, -1]` 范围内）
- 返回 `false` 当 `ptr` 是 **NULL** 或 **有效指针**

### `IS_ERR_OR_NULL(ptr)` —— 检查是否 ERR_PTR 或 NULL

```c
bool IS_ERR_OR_NULL(const void *ptr);
```

- 返回 `true` 当 `ptr` 是 **ERR_PTR** 或 **NULL**
- 返回 `false` 当 `ptr` 是 **有效指针**

### `PTR_ERR(ptr)` —— 从 ERR_PTR 中提取错误码

```c
int PTR_ERR(const void *ptr);
```

- 从 ERR_PTR 中提取出负数错误码（如 `-ENOMEM`、`-EINVAL`）
- **前提**：`ptr` 必须是 ERR_PTR，否则结果是垃圾值

### `ERR_PTR(err)` —— 将错误码转为指针

```c
void *ERR_PTR(int error);
```

- 将负数错误码转为指针形式
- **前提**：`error` 必须是负数（典型的如 `-ENOMEM`）

### 完整的使用模式

```c
// 分配操作，可能返回 ERR_PTR 或 NULL
struct my_struct *p = alloc_foo(...);

// 正确检查方式
if (IS_ERR(p))
    return PTR_ERR(p);      // 提取错误码并传播

if (!p)
    return -ENOENT;         // 没有对象，返回特定的错误码

// 此时 p 是有效指针
p->data = 42;
```

**注意**：不能同时用 `IS_ERR()` 和 `!p` 来检查同一个返回值——你需要知道这个函数 **约定** 返回什么。

---

## 判断约定：什么函数返回什么

内核对每种返回值有约定俗成的模式，但 **必须查文档和实现** 来确认：

| 函数类型 | 失败时通常返回 | 检查方式 |
|---------|---------------|---------|
| 内存分配（`kmalloc`、`alloc_pages`） | NULL | `if (!ptr)` |
| 文件操作（`d_alloc`、`__d_alloc`） | NULL 或 ERR_PTR | `IS_ERR_OR_NULL()` |
| VFS 操作（`lock_rename`、`mount`） | ERR_PTR | `IS_ERR()` |
| 设备操作（`class_create`、`device_create`） | ERR_PTR | `IS_ERR()` |

**关键规则**：不要基于函数名猜测。**读取实现代码** 确认失败时到底返回什么。函数注释可能过时，但代码不会说谎。

---

## 最佳实践总结

### 检查代码时的三问

1. **这个函数失败时返回什么？** NULL？ERR_PTR？还是两者都可能？
2. **调用方是如何检查的？** 用了 `IS_ERR()`？`!ptr`？还是 `IS_ERR_OR_NULL()`？
3. **检查方式是否匹配返回约定？** 如果不匹配，必须标记为 bug。

### 常见模式速查

| 返回值约定 | 正确的检查方式 | 错误的检查方式 |
|-----------|--------------|--------------|
| 仅 NULL | `if (!ptr)` | `IS_ERR(ptr)` → 漏检 |
| 仅 ERR_PTR | `if (IS_ERR(ptr))` | `if (!ptr)` → 漏检 |
| NULL 或 ERR_PTR | `if (IS_ERR_OR_NULL(ptr))` | 单独用 `!ptr` 或 `IS_ERR()` 都漏检 |

### 陷阱总结

```
如果函数返回 ERR_PTR：
   if (ptr)       → TRUE（崩溃！）
   if (!ptr)      → FALSE（漏检！）
   必须用 IS_ERR(ptr)

如果函数返回 NULL：
   IS_ERR(ptr)    → FALSE（漏检！）
   必须用 !ptr

如果函数可能返回 ERR_PTR 或 NULL：
   只检查一个会漏掉另一个
   必须用 IS_ERR_OR_NULL(ptr)
```

### 提取和使用错误码

```c
if (IS_ERR(foo)) {
    int err = PTR_ERR(foo);    // 提取错误码（负数）
    // ... 处理或传播 ...
    return err;
}
```

### 传播错误

在嵌套函数调用中正确传播错误：

```c
// 正确：保持类型一致
void *bar(void) {
    struct foo *p = alloc_foo();
    if (IS_ERR(p))
        return p;               // 直接返回 ERR_PTR，不转换
    if (!p)
        return NULL;
    // ... 使用 p ...
    return p;
}
```

**不要** 做 `return ERR_PTR(PTR_ERR(p))`——这只是一个无意义的转换。**不要** 做 `if (IS_ERR(p)) return NULL`——这会丢失错误信息。
