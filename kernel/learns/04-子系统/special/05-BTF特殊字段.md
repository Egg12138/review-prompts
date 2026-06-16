<!-- Source: subsystem/btf.md -->

# BTF 特殊字段

## 概述

BPF map 的值（value）中可以包含特殊的 BTF 类型字段（BTF-typed fields），如自旋锁（spin locks）、定时器（timers）、kptr、链表头（list heads）等。这些字段持有内核资源，不能简单地用 `memcpy` 复制。

两个核心函数负责特殊字段的处理：

- **`check_and_init_map_value(map, dst)`**：将 map value 复制到临时/用户态缓冲区后调用。重新初始化副本中的特殊字段，防止内核指针和锁状态泄露到用户态。
- **`bpf_obj_free_fields(map->record, ptr)`**：当覆盖或释放 map value 时调用。释放旧值持有的资源（取消定时器、释放 kptr 引用、清理链表头等）。

---

## 支持的 Map 类型

并非所有 map 类型都支持特殊 BTF 字段。允许列表（allowlist）在 `map_check_btf()` 中实现，位于 `kernel/bpf/syscall.c`。如果 map 类型不在对应字段的 allowlist 中，在 map 创建时会返回 `-EOPNOTSUPP`。

| 字段类型 | 允许的 Map 类型 |
|---|---|
| `BPF_SPIN_LOCK`, `BPF_RES_SPIN_LOCK` | HASH, ARRAY, CGROUP_STORAGE, SK_STORAGE, INODE_STORAGE, TASK_STORAGE, CGRP_STORAGE |
| `BPF_TIMER`, `BPF_WORKQUEUE`, `BPF_TASK_WORK` | HASH, LRU_HASH, ARRAY |
| `BPF_KPTR_UNREF`, `BPF_KPTR_REF`, `BPF_KPTR_PERCPU`, `BPF_REFCOUNT` | HASH, PERCPU_HASH, LRU_HASH, LRU_PERCPU_HASH, ARRAY, PERCPU_ARRAY, SK_STORAGE, INODE_STORAGE, TASK_STORAGE, CGRP_STORAGE |
| `BPF_UPTR` | TASK_STORAGE |
| `BPF_LIST_HEAD`, `BPF_RB_ROOT` | HASH, LRU_HASH, ARRAY |

如果一个 map 类型不在上述任何 allowlist 中，它就不能拥有特殊 BTF 字段，缺少 `check_and_init_map_value` 或 `bpf_obj_free_fields` 也不是缺陷。

---

## Map 操作中的处理要求

### Lookup（内核到用户态复制）

当 `copy_map_value()` 或 `copy_map_value_long()` 将 map value 复制到最终返回给用户态的缓冲区时，必须在目标缓冲区上调用 `check_and_init_map_value()`。这会清零特殊字段，防止内核地址和锁状态暴露。

参考实现：

- **系统调用路径**：`bpf_map_copy_value()` → `map->ops->map_lookup_elem()` 获取指针，然后 `copy_map_value()` + `check_and_init_map_value()`
- **Percpu**：`bpf_percpu_array_copy()`、`bpf_percpu_hash_copy()` 内部自行处理
- **批量**：`generic_map_lookup_batch()` 委托给 `bpf_map_copy_value()`

### Update（用户态到内核复制）

当 `copy_map_value()` 或 `copy_map_value_long()` 用用户态数据覆盖已有的 map value 时，必须在复制前或复制后调用 `bpf_obj_free_fields()` 释放旧值持有的资源。否则会出现：定时器持续触发、kptr 引用泄漏、链表项无法访问。

参考实现（这些函数内部调用 `bpf_obj_free_fields()`）：

- **常规**：`array_map_update_elem()`、`htab_map_update_elem()`（通过 `check_and_free_fields()`）
- **Percpu**：`bpf_percpu_array_update()`、`bpf_percpu_hash_update()`（通过 `pcpu_copy_value()`）
- **批量**：`generic_map_update_batch()` 委托给 map 的 `map_update_elem` 回调

---

## 检查要点

当新增 map 操作或 map 类型并使用 `copy_map_value()` / `copy_map_value_long()` 时，需要依次验证：

1. 该 map 类型是否支持特殊 BTF 字段？（检查上面 allowlist 表格）
2. Lookup：复制后是否调用了 `check_and_init_map_value()`？
3. Update：覆盖前是否调用了 `bpf_obj_free_fields()` 清理旧值？
4. 与同类型操作的参考实现进行对比。

**注意**：同一 map 类型的 percpu 变体和非 percpu 变体可能有不同的 allowlist——务必核实确切的 `BPF_MAP_TYPE_*` 枚举值。

**报告为缺陷**：在支持特殊字段的映射类型上，使用 `copy_map_value()` 复制值而没有相应的 `check_and_init_map_value()`（查找）或 `bpf_obj_free_fields()`（更新）的映射操作。对于不在 `map_check_btf()` 允许列表中的映射类型不要报告。
