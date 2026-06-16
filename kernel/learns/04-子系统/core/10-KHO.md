<!-- Source: subsystem/kho.md -->

# KHO（Kexec Handover）子系统核心知识

## 启用状态与初始化

调用序列化侧 KHO API（如 `kho_add_subtree()` 或 `kho_remove_subtree()`）时，如果子系统未启用，会导致在 `kho_out.fdt` 上发生 NULL 指针解引用 —— `kho_out.fdt` 仅在 `kho_init()` 中 `kho_is_enabled()` 为 true 时才分配。反序列化侧 API 如 `kho_retrieve_subtree()` 安全地返回 `-ENOENT`，但保留 API 如 `kho_preserve_folio()` 会静默添加永远不会被使用的跟踪状态。

### Invariants

所有调用者必须在 KHO 使用上添加 `kho_is_enabled()` 守卫：

- `kho_is_enabled()`：返回 KHO 子系统是否活跃。后备变量 `kho_enable` 是 `__ro_after_init` 的，通过 `kho=` 启动参数或 `CONFIG_KEXEC_HANDOVER_ENABLE_DEFAULT` 设置
- `is_kho_boot()`：返回运行中的内核是否通过 KHO 启用的 kexec 加载（即传入的 FDT 是否存在）。仅在早期启动期间 `kho_populate()` 运行后才可靠
- 在模块初始化或使用 KHO API 的任何代码路径的入口处检查 `kho_is_enabled()`

```c
// 错误：缺少启用检查
static int __init my_kho_init(void)
{
    err = kho_add_subtree("my_node", fdt);
    // 如果 KHO 禁用，kho_out.fdt 上 NULL 解引用
}

// 正确：首先检查启用状态
static int __init my_kho_init(void)
{
    if (!kho_is_enabled())
        return 0;

    err = kho_add_subtree("my_node", fdt);
}
```

### Failure Modes

- 子系统禁用时调用序列化 API -> `kho_out.fdt` NULL 解引用
- 混淆 `kho_is_enabled()` 和 `is_kho_boot()`：`is_kho_boot()` 在 `kho_populate()` 运行前不可靠

---

## 保留（Preserve）和恢复（Restore）API 契约

### Invariants

- `kho_preserve_folio()` / `kho_unpreserve_folio()`：对完整的 folio 操作；folio order 在 kexec 中保持不变，`kho_restore_folio()` 将其重建为 compound page
- `kho_preserve_pages()` / `kho_unpreserve_pages()`：对连续的 order-0 页面范围操作；必须使用 `kho_restore_pages()` 恢复，而非 `kho_restore_folio()`，因为恢复路径设置 per-page 引用计数的方式不同（每个页面获得 refcount 1，而 folio 仅头页面获得 refcount 1）
- `kho_unpreserve_pages()` 必须以与对应的 `kho_preserve_pages()` 调用**完全相同**的 `page` 和 `nr_pages` 调用；不支持取消保留任意子范围
- `kho_preserve_vmalloc()` / `kho_unpreserve_vmalloc()`：保留 vmalloc 区域；仅支持 `VM_ALLOC` 和 `VM_ALLOW_HUGE_VMAP` 标志（`kernel/liveupdate/kexec_handover.c` 中的 `KHO_VMALLOC_SUPPORTED_FLAGS`）；其他标志返回 `-EOPNOTSUPP`
- `kho_alloc_preserve()`：一步分配零化的 2 的幂次 folio 并保留；与 `kho_unpreserve_free()` 配对以撤销，或 `kho_restore_free()` 在后继内核中回收

### Failure Modes

不匹配的 preserve 和 restore 调用会：
- 损坏页面元数据
- 使内存保留跨 kexec 泄漏
- 导致后继内核错误解释页面状态（错误顺序、错误引用计数）

---

## 子树生命周期

### Invariants

- `kho_add_subtree()` 在 KHO 根树中记录调用者拥有的 FDT blob 的物理地址；调用者必须**单独保留**该 FDT 的 backing 页面（例如通过 `kho_preserve_folio()` 或 `kho_preserve_pages()`）
- `kho_remove_subtree()` 通过匹配 FDT 指针的物理地址来移除子树；它**不会**释放或取消保留 FDT 内存 —— 调用者必须这样做
- `kho_retrieve_subtree()` 在后继内核中按名称查找子树；返回原始物理地址，必须通过 `phys_to_virt()` 转换后才能使用
- 子树名称必须唯一；`kho_add_subtree()` 在名称已存在时返回 `-EEXIST`

### Failure Modes

未能保留传递给 `kho_add_subtree()` 的 FDT 内存会导致后继内核收到悬空的物理地址，当它调用 `kho_retrieve_subtree()` 时得到垃圾数据或触发 fault。

---

## Scratch Region 约束

### Invariants

- `kho_preserve_folio()` 和 `kho_preserve_pages()` 通过 `kho_scratch_overlap()` 检查 scratch 重叠，如果保留范围与 scratch 内存相交则返回 `-EINVAL`（带 `WARN_ON`）
- Scratch region 在 `kho_reserve_scratch()` 中保留为 CMA 后备的连续区域，大小由 `kho_scratch=` 启动参数调整（默认为 memblock 保留内核内存的 200%）

### Failure Modes

保留与 KHO scratch region 重叠的内存会损坏为后继内核早期启动分配保留的连续区域，可能使下一个 kexec 无法启动。

---

## FDT 字节序

KHO 使用自定义 FDT 格式，其中所有值都使用**本地主机字节序（native endianness）**存储，而非标准 devicetree 规范（dtspec）要求的大端格式。这是有意为之，因为 KHO FDT 仅由产生它们的同一架构消费（同一台机器上的后继内核），因此字节交换是不必要的开销。**不要将 KHO FDT 属性的本地字节序读取或写入标记为 bug** —— 它们设计上是正确的。

---

## Quick Checks

- 验证调用 KHO 序列化 API（`kho_add_subtree()`、`kho_preserve_folio()` 等）的每个代码路径在调用函数或其已验证的祖先中由 `kho_is_enabled()` 守卫
- 验证 `kho_preserve_pages()` / `kho_unpreserve_pages()` 以匹配的 `page` 和 `nr_pages` 参数调用
- 验证传递给 `kho_add_subtree()` 的 FDT 内存是独立保留的；`kho_add_subtree()` 仅记录物理地址
- 验证错误路径调用适当的取消保留函数（`kho_unpreserve_folio()`、`kho_unpreserve_pages()`、`kho_unpreserve_vmalloc()` 或 `kho_remove_subtree()` + `kho_unpreserve_*()`）
