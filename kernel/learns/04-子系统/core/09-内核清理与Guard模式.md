<!-- Source: subsystem/cleanup.md -->

# 内核清理（Cleanup）与 Guard 模式核心知识

## 清理函数兼容性

使用 `__free()` 配合不能处理变量可能持有的所有值的清理函数会导致 crash 或未定义行为。如果分配器在失败时返回 `ERR_PTR` 但清理包装器只检查 `if (_T)`，则 `ERR_PTR` 值是 truthy 的，清理函数会收到错误指针，导致内核 crash。

### 验证兼容性

1. 确定分配器可能返回的内容：NULL, ERR_PTR, 有效指针，或组合
2. 确定 `DEFINE_FREE` 包装器防范的内容：`if (_T)`（仅 NULL 安全），`if (!IS_ERR_OR_NULL(_T))`（NULL 和 ERR_PTR 安全），或不防范
3. 验证包装器在每个早期返回路径上处理所有可能的分配器返回值

### 常见清理包装器

| 包装器 | 定义 | 安全范围 |
|--------|------|---------|
| `__free(kfree)` | `if (!IS_ERR_OR_NULL(_T)) kfree(_T)` | NULL 和 ERR_PTR 安全 |
| `__free(kfree_sensitive)` | `if (_T) kfree_sensitive(_T)` | 仅 NULL 安全，**不适用于 ERR_PTR** |
| 自定义 `DEFINE_FREE` | 单独检查守卫表达式 | 取决于实现 |

### 缓解模式

- 在早期返回前将变量设置为 NULL
- 在早期返回前使用 `no_free_ptr()` 抑制清理
- 使用检查 `IS_ERR_OR_NULL()` 的清理函数

```c
// 正确：清理包装器防范 ERR_PTR
DEFINE_FREE(my_free, void *, if (!IS_ERR_OR_NULL(_T)) my_release(_T))

struct obj *p __free(my_free) = alloc_thing();  // 可能返回 ERR_PTR
if (IS_ERR(p))
    return PTR_ERR(p);  // 清理看到 ERR_PTR，跳过 my_release()

// 错误：清理包装器只检查 NULL，分配器可能返回 ERR_PTR
DEFINE_FREE(my_free, void *, if (_T) my_release(_T))

struct obj *p __free(my_free) = alloc_thing();  // 可能返回 ERR_PTR
if (IS_ERR(p))
    return PTR_ERR(p);  // 清理看到 ERR_PTR (truthy)，调用 my_release(ERR_PTR)!
```

### Failure Modes

**REPORT as bugs**：任何 `__free()` 变量，其清理包装器在**每个退出点**都不防范该变量可能持有的所有值。

---

## LIFO 定义顺序

以错误顺序定义 `__free()` 变量和 `guard()` 锁导致清理以错误的顺序执行：锁在需要锁来清理的资源之前释放，导致 use-after-free 或 lockdep 违规。

**清理按定义逆序执行（LIFO）**。根据 `include/linux/cleanup.h`：

> "当一个作用域中有多个具有清理属性的变量时，在离开作用域时，它们关联的清理函数按定义逆序执行（最后定义的最先清理）。"

### Invariants

- 保护资源的锁必须通过 `guard()` **在**它们保护的资源（通过 `__free()`）**之前**定义
- 引用其他资源的资源必须在它们的依赖**之后**定义
- 尽量在**一条语句中**定义并初始化 `__free()` 变量，而不是在顶部 `= NULL` 稍后赋值——这使得 LIFO 顺序错误更不容易发生（`include/linux/cleanup.h` 推荐）

```c
// 正确：guard 先定义，资源后定义
guard(mutex)(&lock);
struct object *obj __free(remove_free) = alloc_add();
// 作用域退出时：remove_free(obj) 先运行（仍持有锁），然后解锁

// 错误：资源在 guard 之前定义
struct object *obj __free(remove_free) = NULL;
guard(mutex)(&lock);
obj = alloc_add();
if (!obj)
    return -ENOMEM;

err = other_init(obj);
if (err)
    return err;  // remove_free(obj) 在 UNLOCK 之后运行 — 锁未持有！
```

### Failure Modes

- 锁在资源清理前释放 -> use-after-free
- 拆分定义和初始化更容易导致 LIFO 顺序错误

---

## Guard 作用域

使用受 `guard()` 锁保护但其声明作用域之外的数据会导致 use-after-free 或数据竞态，因为锁已经被释放。

根据 `include/linux/cleanup.h`：

> "由 guard() 辅助函数获取的锁的生命周期遵循自动变量声明的作用域。"

### 作用域类型

- **函数作用域**：函数级别的 `guard()` —— 锁持有直到函数返回
- **块作用域**：`if`/`else`/`while` 块内的 `guard()` —— 锁仅持有到右花括号
- **`scoped_guard()`**：锁仅为紧随其后的复合语句持有

```c
// 正确：数据在 guard 作用域内使用
guard(mutex)(&lock);
val = shared_data;  // 锁在此处持有
return val;

// 错误：guard 在块中，数据在块后使用
if (condition) {
    guard(mutex)(&lock);
    val = shared_data;  // 锁在此处持有
}  // 锁在此处释放
use(val);  // 数据竞态 — 锁不再持有
```

### Failure Modes

- `scoped_guard()` 锁仅为其复合语句持有；验证使用了正确的变体

---

## 所有权转移

未能抑制 `__free()` 变量的清理会导致双重释放：清理函数释放资源，新所有者再次释放。

### 转移原语（定义在 `include/linux/cleanup.h`）

- `no_free_ptr(p)`：返回 `p` 并将 `p` 设为 NULL，抑制清理。具有 `__must_check` 语义
- `return_ptr(p)`：`return no_free_ptr(p)` 的简写
- `retain_and_null_ptr(p)`：类似 `no_free_ptr()` 但丢弃返回值。在传递所有权给一个在成功时消耗指针的函数时使用

```c
// 正确：在成功路径上抑制清理
struct obj *p __free(kfree) = kmalloc(...);
if (!p)
    return NULL;  // 清理释放 NULL（无操作）
return_ptr(p);    // 抑制清理，调用者取得所有权

// 正确：条件所有权转移
ret = bar(f);
if (!ret)
    retain_and_null_ptr(f);  // bar() 在成功时消耗了 f
return ret;                  // 如果 bar() 失败，清理释放 f
```

---

## Goto 混合

在同一函数中混合基于 `goto` 的错误处理和 `__free()`/`guard()` 清理会产生令人困惑的所有权语义和双重释放或资源泄漏 bug。

根据 `include/linux/cleanup.h`：

> "期望是"goto"和清理辅助函数的使用**绝不在同一函数中混合**。也就是说，对于给定的例程，将所有需要"goto"清理的资源转换为基于作用域的清理，或者全部不转换。"

### Failure Modes

**REPORT as bugs**：包含 `goto` 基清理标签和 `__free()`/`guard()` 声明的函数。

---

## Quick Checks

- **分配器返回 vs 清理守卫**：审查新的 `__free()` 变量时，检查分配器返回类型（NULL vs ERR_PTR）与 `DEFINE_FREE` 守卫表达式是否匹配
- **拆分定义-初始化**：在函数顶部声明 `__free(name) = NULL` 然后稍后赋值的变量更容易出现 LIFO 顺序错误。验证赋值发生在任何所需的 `guard()` 调用之后
- **scoped_guard vs guard**：`scoped_guard()` 仅为其复合语句持有锁；`guard()` 为封闭作用域的剩余部分持有锁。验证为预期的锁生命周期使用了正确的变体
