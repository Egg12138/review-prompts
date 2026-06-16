<!-- Source: subsystem/sysfs.md -->

# Sysfs 子系统

## 概述

Sysfs 是内核暴露设备、驱动和内核对象信息到用户空间的虚拟文件系统。本文聚焦 attribute group 可见性控制、kobject 初始化和清理等关键不变式和故障模式。

---

## Attribute Group 可见性与条件存在

### 不变式

当 `struct attribute_group` 的 `is_visible` 回调对某个 attribute 返回 0 时，该 attribute 在 `sysfs_create_group()` 期间**永远不会被创建**。这与 attribute 存在但 `show()` 函数返回 `-EOPNOTSUPP` 不同：不存在的 attribute 无法被查找、修改或更改所有权。

在 `sysfs_group_attrs_change_owner()` 中，对从未创建的 attribute 调用 `kernfs_find_and_get()` 返回 NULL，导致 `-ENOENT` 向上传播，使网络命名空间迁移等更改 sysfs 所有权的操作失败。

### 关键细节

- **Binary attributes**：使用独立的 `is_bin_visible` 回调，语义相同
- **命名组**（`grp->name` 已设置）：`is_visible` 和 `is_bin_visible` 还可以返回 `SYSFS_GROUP_INVISIBLE` 来抑制**整个组目录**的创建，而非仅单个 attribute

### 审查要点

当补丁向现有 `attribute_group` 添加 `is_visible` 或 `is_bin_visible` 回调，或将可见性检查从 `show()` 函数迁移到 `is_visible()` 时：

1. 确定所有遍历该组 `grp->attrs` 或 `grp->bin_attrs` 数组的 sysfs 核心函数
2. 验证每个遍历函数在调用 `kernfs_find_and_get()` 或其他查找函数之前检查了 `is_visible` / `is_bin_visible`
3. 检查触发组遍历的调用者（如命名空间变更 `__dev_change_net_namespace()` -> `netdev_change_owner()` -> `device_change_owner()` -> `sysfs_change_owner()` -> `sysfs_groups_change_owner()`）

### 错误与正确模式

```c
// 错误：假设所有 attribute 都存在
for (i = 0, attr = grp->attrs; *attr; i++, attr++) {
    kn = kernfs_find_and_get(parent, (*attr)->name);
    // kn 为 NULL（is_visible 返回 0 时）-> 返回 -ENOENT
}

// 正确：跳过不存在的 attribute
for (i = 0, attr = grp->attrs; *attr; i++, attr++) {
    if (grp->is_visible) {
        mode = grp->is_visible(kobj, *attr, i);
        if (mode & SYSFS_GROUP_INVISIBLE)
            break;
        if (!mode)
            continue;
    }
    kn = kernfs_find_and_get(parent, (*attr)->name);
}
```

### 故障模式

遍历 `grp->attrs[]` 或 `grp->bin_attrs[]` 并对需要存在于文件系统中的 attribute 执行操作的代码，必须检查 `is_visible` / `is_bin_visible` 并跳过返回 0 的 attribute。

---

## Kobject 初始化和清理

### 不变式

在循环或层次结构中创建 kobject（父对象带多个子对象）时，每个错误路径必须清理失败前分配的所有资源。在 `__init` 或 `__initcall` 函数中，这些泄漏是不可恢复的。

### 资源清理规则

- `kobject_create_and_add()` 的分配必须通过错误路径中的 `kobject_put()` 平衡
- 每个 kobject 必须在错误时通过 `kobject_put()` 独立释放；子对象持有对父对象的引用（通过 `kobject_add()`），`kobject_cleanup()` 在子对象被释放时调用 `kobject_put(parent)`，因此释放所有子对象最终会释放父对象的引用计数
- `sysfs_create_group()` 失败也必须清理传递给它的 kobject

### 基于循环的初始化

```c
// 错误：失败时泄漏父对象和之前的子对象
for (int i = 0; i < N; i++) {
    children[i] = kobject_create_and_add(name, parent);
    if (!children[i])
        return -ENOMEM;  // parent 和 children 0..i-1 泄漏了
    ret = sysfs_create_group(children[i], &grp);
    if (ret)
        return ret;  // children[i] 也泄漏了
}

// 正确：集中式清理逐个释放子对象
for (int i = 0; i < N; i++) {
    children[i] = kobject_create_and_add(name, parent);
    if (!children[i]) {
        ret = -ENOMEM;
        goto err_cleanup;
    }
    ret = sysfs_create_group(children[i], &grp);
    if (ret)
        goto err_cleanup;
}
return 0;

err_cleanup:
    for (int j = i; j >= 0; j--)
        kobject_put(children[j]);
    kobject_put(parent);
    return ret;
```

### 故障模式

在循环内直接 `return -ERRNO` 的初始化函数泄漏了所有之前迭代分配的资源。必须使用集中式错误清理路径（通常通过 `goto`）。

---

## 快速检查清单

- **`is_visible()` 返回 0**：attribute 不会被创建，不只是隐藏。调用者无法查找、修改或更改不存在的 attribute 的所有权。
- **`is_bin_visible()` 返回 0**：与 `is_visible()` 语义相同，但用于 `grp->bin_attrs` 中的 binary attribute。
- **`SYSFS_GROUP_INVISIBLE`**：当命名组的 `is_visible` 或 `is_bin_visible` 返回此值时，抑制整个组目录的创建。
- **`fs/sysfs/group.c` 中的组遍历**：任何遍历 `grp->attrs` 或 `grp->bin_attrs` 的新函数，如果 attribute 可能有条件地不存在，必须尊重 `is_visible` / `is_bin_visible`。
- **循环中的 Kobject 清理**：在循环中创建 kobject 的初始化函数必须在任何分配失败时逐个释放每个 kobject。
