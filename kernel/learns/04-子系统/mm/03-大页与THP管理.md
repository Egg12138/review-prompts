<!-- Source: mm-largepage.md -->

# MM 大页、THP 与 Hugetlb

## 大页状态跟踪（State Tracking）

混淆哪些标志是按页（per-page）的、哪些是按 folio（per-folio）的，会导致在错误的结构页上检查或设置状态。常见错误是在操作大 folio 的子页时假设所有标志的工作方式与小页相同。

**跟踪级别**：Per-folio = 头页上的单个值适用于整个 folio；Per-page = 每个子页携带独立的的值；PTE-level = 状态在页表项中而非 struct page 中；Mixed = 粒度取决于 folio 的映射方式。

| 状态 | 跟踪级别 | 说明 |
|------|---------|------|
| `PageAnonExclusive` | **Mixed** | PTE 映射的 THP 为 per-page；PMD 映射和 HugeTLB 为 per-folio（头页） |
| `PG_hwpoison` | **Per-page** | 标记特定损坏的子页；与 `PG_has_hwpoisoned`（per-folio，快速指示至少有子页被 poison）不同 |
| `PG_dirty` | **Per-folio** | 通过 `PF_HEAD` 策略在头页上的单个标志；PTE 级别的脏位在页表项中单独跟踪 |
| Accessed/young | **PTE-level** | 在页表项中跟踪，不在 struct page 中；folio 级别的 `PG_referenced` 是独立的 LRU 老化标志 |
| 引用计数（Reference count） | **Per-folio** | 头页上的单个 `_refcount` 由所有子页共享 |
| Mapcount | **Per-page** | 默认每个子页有 `_mapcount`；`CONFIG_NO_PAGE_MAPCOUNT` 使用 folio 级别的 `_large_mapcount` 和 `_entire_mapcount` |

**页标志策略**：控制 folio 内哪个 struct page 携带每个标志。使用错误的 struct page 会静默读取过期数据或损坏无关状态：
- `PF_HEAD`：标志操作重定向到头页（大多数标志）
- `PF_ANY`：标志对头页、尾页和小页均相关
- `PF_NO_TAIL`：修改仅在头页/小页上进行，尾页允许读取
- `PF_SECOND`：标志存储在第一尾页（如 `PG_has_hwpoisoned`、`PG_large_rmappable`、`PG_partially_mapped`）

### 原子 vs 非原子页标志操作

非原子标志操作（`__set_bit` / `__clear_bit`，由 `__FOLIO_SET_FLAG` / `__FOLIO_CLEAR_FLAG` 生成）对整个 `unsigned long` 标志字执行读-改-写。只有当调用者对 **整个标志字** 有排他访问权时才安全——不仅是对单个位进行锁序列化。多个页标志共享同一个 `unsigned long`，因此序列化一个位的锁不能保护同一字中另一个位被不同锁下的代码并发修改。

这对 `PF_SECOND` 标志尤为重要：`PG_has_hwpoisoned`、`PG_large_rmappable`、`PG_partially_mapped` 和 `PG_anon_exclusive` 都分享同一标志字，由不同子系统在不同锁下修改。

**应报告为 bug**：使用非原子页标志操作（`__folio_set_*` / `__folio_clear_*` / `__SetPage*` / `__ClearPage*`）的代码，除非调用者对整个页有排他访问权。

## 复合页 PFN 自然对齐

order-`n` 的复合页起始 PFN 天然对齐到 `1 << n`。该复合页中的任何子页 PFN 可用 `pfn & ~((1UL << order) - 1)` 或 `ALIGN_DOWN(pfn, 1UL << order)` 向下取整到头页 PFN。

## 大 Folio 的页缓存引用计数

`__filemap_add_folio()`（`mm/filemap.c`）添加 `folio_nr_pages(folio)` 个额外引用。页缓存 folio 的 refcount = 1（基础）+ `folio_nr_pages()`（页缓存）+ 其他持有者。移除时必须通过 `folio_put_refs(folio, folio_nr_pages(folio))` 释放所有页缓存引用。在 `__filemap_remove_folio()` 后使用 `folio_put()`（单次引用释放）会泄漏 `folio_nr_pages() - 1` 个引用——仅在有大 folio 时可见的静默内存泄漏。

## 大 Folio 拆分最小 Order

将文件后端的大 folio 拆分到低于映射最小 folio order 会失败并返回 `-EINVAL`。假设成功拆分总是产生 order-0 folio 的调用者会在 LBS 文件系统上遇到警告、操作意外大的 folio 或错误处理不当。

文件后端的地址空间可以通过 `mapping_set_folio_min_order()` 设置最小 folio order。`__folio_split()`（`mm/huge_memory.c`）中的拆分基础设施强制执行：如果 `new_order < min_order`，则返回 `-EINVAL`。

**拆分 API 行为：**
- `split_huge_page()` 和 `split_folio_to_list()` 总是请求 order-0。对 min order > 0 的映射的文件后端 folio 会失败
- `try_folio_split_to_order()` 接受显式的 `new_order` 参数
- `min_order_for_split()` 对文件后端 folio 返回 `mapping_min_folio_order(folio->mapping)`，对匿名 folio 返回 0

**审查清单**：
- 如果代码调用 `split_huge_page()` 或 `split_folio_to_list()` 然后假设结果是 order-0，验证它处理了来自 min order > 0 的映射的 `-EINVAL`
- 如果需要拆分到最低可能 order，必须先调用 `min_order_for_split()` 并将该 order 显式传递给 `try_folio_split_to_order()`
- 成功拆分到非零 order 后，结果 folio 仍然是大的（`folio_test_large()` 返回 true）

## 大 Folio 拆分 Refcount 前提条件

在大 folio 上持有额外引用时调用 `split_folio()` 会导致返回 `-EAGAIN`，如果调用者在循环中重试，会在多个任务操作同一 folio 时造成活锁。

**危险顺序（refcount 在 lock 前）**：
```c
// 错误：提升 refcount，然后在 lock 上阻塞；其他 task 做同样的事情
// 会抬高 refcount，导致 split_folio() 失败
folio_get(folio);
folio_lock(folio);          // 阻塞时持有额外引用
err = split_folio(folio);   // 失败：refcount 过高
```

**安全顺序（lock 在 refcount 前）**：
```c
// 正确：先 lock 确保只有一个 task 继续；然后提升 refcount
if (!folio_trylock(folio))
    return -EAGAIN;         // 没有提升 refcount，不会活锁
folio_get(folio);
err = split_folio(folio);   // 预期 refcount 匹配
```

**应报告为 bug**：在大 folio 上先调用 `folio_get()` 再在 `folio_lock()` 上阻塞，然后再调用 `split_folio()` 的代码路径，尤其是在重试循环中。

## 大 Folio i_size 边界检查

映射文件后端大 folio 时不检查 `i_size` 会破坏 POSIX SIGBUS 语义：VMA 内但超出 `i_size` 向上对齐到 `PAGE_SIZE` 的访问必须产生 SIGBUS，但大 folio 的过度映射会静默为零填充页面服务这些访问。

**不变式：**
- 文件页的 PTE 不能安装在 `DIV_ROUND_UP(i_size_read(mapping->host), PAGE_SIZE)` 之外
- PMD 映射不能在 folio 延伸超出 `i_size` 时安装
- 在截断操作中，如果横跨新 `i_size` 的大 folio 不能拆分，必须完全取消映射

**例外 —— shmem/tmpfs**：`shmem_mapping()` 对 shmem/tmpfs 地址空间返回 true。这些映射不受 i_size 边界检查约束，允许用 PMD 跨 i_size 映射。

## 大 Folio 换入（Swapin）与 Swap 缓存冲突

大 folio（mTHP）换入时不检查现有更小的 swap 缓存条目会导致无界重试循环。`swapcache_prepare()` 在范围内任何槽位设置了 `SWAP_HAS_CACHE` 时失败并返回 `-EEXIST`，调用者在 `-EEXIST` 上重试会永远循环，因为预读或并发 swapin 可以持久填充单个 order-0 条目。

在 mTHP swapin 前，使用 `non_swapcache_batch(entry, nr_pages)` 验证没有槽位被占用；如果结果 < `nr_pages` 则回退到 order-0。

## Hugetlb Folio 类型转换竞态

在无锁的 `folio_test_hugetlb()` 检查后访问 hugetlb 特定的 folio 元数据（如调用 `folio_hstate()`）会导致空指针解引用，当另一个 CPU 同时清除 hugetlb 类型并释放 folio 时。

`__update_and_free_hugetlb_folio()` 在 `hugetlb_lock` 下调用 `__folio_clear_hugetlb()`。类型被清除后，`folio_hstate()` 调用 `size_to_hstate(folio_size())`，由于 folio 大小不再匹配任何注册的 hstate 而返回 NULL。

```c
// 错误：TOCTOU 竞态
if (folio_test_hugetlb(folio)) {
    h = folio_hstate(folio);  // 如果类型被同时清除，可能返回 NULL
}

// 正确：在同一锁下检查和和使用
spin_lock_irq(&hugetlb_lock);
if (folio_test_hugetlb(folio)) {
    h = folio_hstate(folio);
}
spin_unlock_irq(&hugetlb_lock);
```

无锁的 `folio_test_hugetlb()` 可作为初步快速路径过滤器，但结果不可信——需在锁下重新检查。

## Hugetlb 缺页路径锁定

锁顺序：`hugetlb_fault_mutex` -> `vma_lock` -> `i_mmap_rwsem` -> `folio_lock`。`hugetlb_wp()` 在操作中途释放 mutex 和 vma_lock。不要在到达 `hugetlb_wp()` 的路径中在持有 `hugetlb_fault_mutex` 时使用 `filemap_lock_folio()`。使用 `folio_trylock()` 并在失败时退出，在释放所有锁后等待。

## Hugetlb 池记账

`hstate` struct 有四个计数器（均由 `hugetlb_lock` 保护）：`nr_huge_pages`（总数）、`free_huge_pages`、`surplus_huge_pages`（超出持久池）、`resv_huge_pages`（预留的）。可用 = `free_huge_pages - resv_huge_pages`。每个有 per-node 变体，除了 `resv_huge_pages`。

**关键规则：**
- `alloc_hugetlb_folio()` 使用 `vma_needs_reservation()` 的 `gbl_chg` 区分预留与非预留分配；只在 `!gbl_chg` 时递减 `resv_huge_pages`
- 派生值（`persistent_huge_pages()` = `nr_huge_pages - surplus_huge_pages`）组合了必须在同一次 `hugetlb_lock` 持有中更新的计数器，避免瞬时不一致
- `remove_hugetlb_folio()` / `add_hugetlb_folio()` 接受 `bool adjust_surplus`；调用者必须检查 `surplus_huge_pages_node[nid]` 并传递结果——硬编码 `false` 会静默跳过 surplus 调整

## Hugetlb PMD 页表共享与取消共享（Unsharing）

取消共享 hugetlb PMD 页表时，释放的页表页必须通过 `tlb_remove_table()`（而非直接 `free_page()`）以与 GUP-fast 同步。`tlb_remove_table_sync_one()` 发送 IPI 确保在重用时没有并发的 GUP-fast。

**锁定**：PMD 共享/取消共享需要 `i_mmap_rwsem` 的写模式。调用 `huge_pmd_share()` 的缺页路径持读锁，失败时以写锁重试。

## 内存错误（Memory Failure）Folio 处理

**`memory_failure()` 返回值**：`0` = 已恢复（无需信号），`-EHWPOISON` = 已被 poison，`-EOPNOTSUPP` = 被 `hwpoison_filter()` 过滤，其他负值 = 恢复失败（进程被 kill）。

**大 folio hwpoison**：`PG_hwpoison` 是 per-page（`PF_ANY`）；`PG_has_hwpoisoned` 是 per-folio（`PF_SECOND`）作为快速指示器。两者必须保持同步。拆分被 poison 的 folio 时，`PG_has_hwpoisoned` 必须传播到正确的子 folio。在获取 folio lock 后重新检查 `folio_test_large()`（可能发生并发拆分）。

**HWPoison 内容访问守卫**：访问被 poison 的页内容会触发不可恢复的 MCE/panic。在访问内容前按子页检查 `PageHWPoison(page)` 或使用 `folio_contain_hwpoisoned_page(folio)` 提前退出。**应报告为 bug**：THP 拆分、迁移、KSM 或压缩中没有 hwpoison 检查的内容读取。

## 快速检查清单

- **大 folio mapcount 字段一致性**：`_large_mapcount`、`_entire_mapcount`、per-page `_mapcount`、`_nr_pages_mapped` 在 rmap 操作中非原子更新。读取多个字段需要 `folio_lock_large_mapcount()` 获得一致快照
- **大 folio 在 PFN 迭代中的边界跨越**：跨越子范围边界的大 folio 会在多个子范围中出现。没有去重的话，操作会应用多次。跟踪最后处理的 folio 或按 `folio_size()` 前进
- **大 folio 大小在 hwpoison 路径上的前提条件**：`unmap_poisoned_folio()` 不能处理大的非 hugetlb folio。调用者必须先检查 `folio_test_large() && !folio_test_hugetlb()` 并拆分
- **Hugetlb 页缓存插入协议**：在 `hugetlb_add_to_page_cache()` 前：用 `folio_zero_user()` 清零，用 `__folio_mark_uptodate()` 标记 uptodate，持有 `hugetlb_fault_mutex_table[hash]`
- **Hugetlb HPG 标志在降级（demotion）期间的传播**：`init_new_hugetlb_folio()` 不从 `folio->private` 传播 HPG 标志。从现有 hugetlb folio 创建新 folio 的代码必须显式复制这些标志
- **通用 folio 路径中拒绝 Hugetlb folio**：仅处理 PTE 级别 folio 的通用 rmap/迁移回调必须通过 `folio_test_hugetlb()` 提前拒绝 hugetlb
- **Hugetlb 子池预留回滚**：`hugepage_subpool_get_pages()` 从 `rsv_hpages` 吸收部分页面并返回较小的 `gbl_reserve`。错误路径必须向 `hugepage_subpool_put_pages()` 返回 `chg - gbl_reserve`（不是 `chg`），并将其返回值传给 `hugetlb_acct_memory()`。超额计入（over-crediting）会导致 `resv_huge_pages` 下溢
- **大 folio 的延迟拆分队列**：部分取消大 folio 映射（取消部分而非全部子页的映射）时，应通过 `deferred_split_folio()` 将 folio 加入延迟拆分队列。缺少此调用会浪费内存
- **`try_to_unmap()` 在 PMD 映射的大 folio 上**：需要 `TTU_SPLIT_HUGE_PMD` 标志，否则 `try_to_unmap_one()` 中会触发 `VM_BUG_ON_FOLIO(!pvmw.pte, folio)`
- **`pmd_trans_huge()` 同时匹配 THP 和 hugetlb PMD**：在 THP 特定操作前先检查 `is_vm_hugetlb_page(vma)`。Hugetlb 使用不同的页表布局、锁定和拆分语义
- **内存错误记账一致性**：`action_result()` 同时更新 `num_poisoned_pages`（全局）和 `memory_failure_stats`（per-node）。调用 `num_poisoned_pages_inc()` 而不调用 `update_per_node_mf_stats()` 会导致 `/proc/meminfo` 与 sysfs 不一致
