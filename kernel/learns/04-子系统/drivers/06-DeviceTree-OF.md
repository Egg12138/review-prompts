<!-- Source: subsystem/of.md -->

# Device Tree / Open Firmware（OF）子系统

## 概述

Open Firmware（OF）是 Linux 内核中处理 Device Tree 的子系统。本文聚焦 OF 节点迭代器宏的引用计数管理和 MSI 控制器绑定变体的处理。

---

## OF 节点迭代器宏与引用计数

### 不变式

在 OF 节点迭代器循环中错误使用 `of_node_put()` 会导致 OF 节点引用的 double-free（双重释放），在后续访问同一节点时引发 use-after-free。

### 自动管理引用计数的迭代器宏

以下迭代器宏在前进到下一个节点前自动调用 `of_node_put()` 释放前一个节点：

- `for_each_child_of_node(parent, child)`
- `for_each_available_child_of_node(parent, child)`
- `for_each_reserved_child_of_node(parent, child)`
- `for_each_node_by_type(dn, type)`
- `for_each_compatible_node(dn, type, compatible)`
- `for_each_matching_node(dn, matches)`
- `for_each_matching_node_and_match(dn, matches, match)`
- `for_each_node_with_property(dn, prop_name)`

每个宏展开为一个 `for` 循环，其递增表达式将当前节点作为 `prev`/`from` 参数传递给底层查找函数（如 `of_get_next_child()`、`of_find_node_by_type()`）。该函数释放 `prev`/`from` 上的引用并获取返回节点上的引用。

### 引用处理语义

- 迭代器宏在每次迭代期间持有当前节点的引用
- 当循环推进时（包括通过 `continue`），宏自动释放当前节点的引用再获取下一个
- **仅**在通过 `break` 或 `return` 提前退出循环时需要手动 `of_node_put()`
- 在每次迭代中添加 `of_node_put()` 会造成 double-free

### 正确与错误模式

```c
// 正确：普通迭代，无需手动 put
for_each_child_of_node(parent, node) {
    if (!matches(node))
        continue;  // 宏处理了 put
    process(node);
}

// 正确：提前 break 需要手动 put
for_each_child_of_node(parent, node) {
    if (found(node)) {
        of_node_put(node);
        break;
    }
}

// 正确：提前 return 需要手动 put
for_each_child_of_node(parent, node) {
    if (error_condition(node)) {
        of_node_put(node);
        return -EINVAL;
    }
}

// 错误：每次迭代都运行的 goto 标签
for_each_child_of_node(parent, node) {
    if (!matches(node))
        goto next;
    process(node);
next:
    of_node_put(node);  // Double-free!
}

// 错误：continue 前显式 put
for_each_child_of_node(parent, node) {
    if (!matches(node)) {
        of_node_put(node);  // Double-free!
        continue;
    }
}
```

### Scoped 变体

`for_each_child_of_node_scoped()` 和 `for_each_available_child_of_node_scoped()` 使用 `__free(device_node)` 声明迭代变量，当变量超出作用域时引用自动释放。这些变体**不需要**在提前退出时手动 `of_node_put()`。

---

## OF 节点获取 API

### 不变式

通过显式 API 调用获取的 OF 节点引用计数已递增，需要匹配的 `of_node_put()` 来释放，否则会造成内存泄漏。

### 需要手动释放的 API

以下 API 返回引用计数已递增的节点：

- `of_find_node_by_path()`
- `of_find_node_by_name()`
- `of_find_node_by_type()`
- `of_find_compatible_node()`
- `of_find_matching_node_and_match()`
- `of_find_node_with_property()`
- `of_get_parent()`
- `of_get_next_parent()` — 在返回的父节点上获取引用，并释放输入节点上的引用
- `of_get_child_by_name()`
- `of_parse_phandle()`
- `of_find_node_by_phandle()`
- `of_node_get()` — 显式递增引用计数

### 特殊注意事项

`of_find_node_by_name()`、`of_find_node_by_type()`、`of_find_compatible_node()`、`of_find_matching_node_and_match()` 和 `of_find_node_with_property()` 还会释放在其 `from` 参数上的引用，因此调用者**不得**在调用这些函数后单独调用 `of_node_put()` 处理 `from`。

### 故障模式

- 迭代器循环内（非提前退出路径）使用 `of_node_put()` -> double-free
- 使用 `of_parse_phandle()` 后未调用 `of_node_put()` -> 内存泄漏
- 对既是 `from` 参数又被 API 自动释放的节点额外调用 `of_node_put()` -> double-free

---

## MSI 控制器 Device Tree 绑定变体

### 关键知识点

未能处理所有 MSI 控制器 DT 绑定变体会导致在不支持的绑定风格的平台上发生 MSI 分配失败。

### 两种绑定风格

1. **`msi-map` 绑定** — 将设备 ID 映射到 MSI 控制器域。使用 `msi-map-mask` 掩码输入 ID。由 `of_map_id()` 处理（从 `of_msi_xlate()` 调用，位于 `drivers/of/irq.c`）

2. **`msi-parent` 绑定** — 通过 phandle 标识 MSI 控制器：
   - 当 `#msi-cells` 缺失或为零时，表示 1:1 ID 映射（见 `of_check_msi_parent()`）
   - 当 `#msi-cells` 存在且非零时，phandle 携带额外的 specifier 参数

### 处理函数

- `of_msi_xlate()`：遍历设备层次，检查 `msi-map` 和 `msi-parent` 属性，返回映射的 MSI ID 并可选择解析 MSI 控制器节点
- `of_msi_get_domain()`：通过遍历带有 `#msi-cells` 的 `msi-parent` phandle 解析 MSI irq 域

### 故障模式

仅支持一种绑定风格而忽略另一种，导致使用其他绑定风格的平台上 MSI 分配失败。

---

## 快速检查清单

- **`of_node_put()` 在 `for_each_*` OF 迭代器循环中**：必须仅在提前退出路径（`break`/`return`）上执行。普通迭代和 `continue` 不需要手动清理。`_scoped` 变体完全不需要手动清理。
- **`of_parse_phandle()`**：返回引用计数递增的节点。调用者完成后必须调用 `of_node_put()`。
- **`for_each_property_of_node()`**：直接遍历属性链表，不涉及引用计数。
