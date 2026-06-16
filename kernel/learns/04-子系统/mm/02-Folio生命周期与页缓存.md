<!-- Source: mm-folio.md -->

# MM Folio 生命周期与页缓存

## Lazyfree Folio 回收状态转换

Lazyfree folio（`!folio_test_swapbacked()`）如果为 clean 状态，可以在无需写回的情况下被丢弃。回收路径必须在检查 refcount 之前检查脏状态，并且只有在 folio 确实脏时才调用 `folio_set_swapbacked()`：

1. **脏（非 `VM_DROPPABLE`）**：调用 `folio_set_swapbacked()` 并重新映射
2. **额外引用（`ref_count != 1 + map_count`）**：重新映射并中止，但 **不** 调用 `folio_set_swapbacked()`——refcount 升高（如瞬时的 `folio_try_get()`）并不表示脏
3. **Clean，无额外引用**：丢弃

PTE 级别（`try_to_unmap_one()` 在 `mm/rmap.c`）和 PMD 级别（`__discard_anon_folio_pmd_locked()` 在 `mm/huge_memory.c`）路径都必须遵循此顺序。屏障协议与 `__remove_mapping()` 相同：在 PTE 清除和 refcount 读取之间使用 `smp_mb()`；在 refcount 和脏标志读取之间使用 `smp_rmb()`。

**应报告为 bug**：在未确认脏的情况下就调用 `folio_set_swapbacked()`，或者在任何中止路径上无条件调用（包括 refcount 升高的情况）。

## Folio 尾页覆盖布局（Tail Page Overlay Layout）

`struct folio` 将元数据叠加到尾页（tail page）的 `struct page` 槽位上。尾页的 `->mapping` 被设为 `TAIL_MAPPING`；携带元数据的尾页会覆盖它。三个消费者必须保持同步：

- `free_tail_page_prepare()`（`mm/page_alloc.c`）——对每个携带元数据的尾页跳过 `TAIL_MAPPING` 检查
- `__dump_folio()`（`mm/debug.c`）——调试打印

常见故障：更新了一个消费者但遗漏了其他（三者之间没有编译时耦合）。

## Folio Mapcount 与 Refcount 关系

**不变式**：`folio_ref_count(folio) >= folio_mapcount(folio)`。额外的 refcount 来自 swapcache、page cache、GUP pin、LRU 隔离等（参见 `folio_expected_ref_count()` 在 `include/linux/mm.h`）。

- 独占性检查：`folio_ref_count() == folio_expected_ref_count()` 表示没有意外的持有者（lazyfree 路径使用更简单的 `ref_count == 1 + map_count`）
- 健全性断言：`mapcount > refcount` 是需要警告的损坏/不可能状态，**而非** `mapcount < refcount`（后者是正常的）

## 非 Folio 复合页（Non-Folio Compound Pages）

`page_folio()` 将复合头页强制转换为 `struct folio *`，没有任何运行时有效性检查。驱动分配的复合页（通过 `vm_insert_page()` 配合 `alloc_pages(GFP_*, order > 0)`）设置了 `PG_head`，使得 `folio_test_large()` 返回 true，但 `folio->mapping` 和 LRU 状态未初始化。对这些页面调用 folio 操作（`folio_lock()`、`split_huge_page()`、`mapping_min_folio_order()`）会导致崩溃或数据损坏。

验证门控（`HWPoisonHandlable()`、`PageLRU()`、空 mapping 检查）会拒绝非 folio 页面。当代码路径对来自驱动映射的页面调用 `page_folio()` 时，需验证已有门控过滤了非 folio 复合页。`folio_test_large()` 单独不够——它只检查 `PageHead`，而任何复合页都会设置该位。

## Folio 引用计数预期（Reference Count Expectations）

`folio_expected_ref_count()`（`include/linux/mm.h`）根据 pagecache、swapcache、`PG_private` 和映射计算预期 refcount。对比 `folio_ref_count()` 可检测来自任何来源的意外引用。Per-CPU 批处理（LRU pagevecs 在 `mm/swap.c`、mlock/munlock 批处理在 `mm/mlock.c`）持有瞬时的 `folio_get()` 引用，对标志检查不可见。`lru_add_drain_all()` 排干所有 CPU 的批处理；检测到意外引用的代码应先排干再重新检查，然后才断定 folio 不可迁移。

**应报告为 bug**：使用 `folio_test_lru()` 作为"有额外引用"的代理，而不是将 `folio_ref_count()` 与 `folio_expected_ref_count()` 进行比较。

## Folio Order 与页数

`round_up()`、`round_down()`、`ALIGN()` 需要实际的页数（`1 << order`），而非 order 指数。order 为 0 和 1 时不易察觉。

```c
// 正确                          // 错误
round_up(index, 1 << order)    round_up(index, order)
ALIGN(addr, PAGE_SIZE << order)
```

**应报告为 bug**：对齐参数是裸的 `order` 变量，而非 `1 << order` 或 `PAGE_SIZE << order`。

## GUP 后的 Folio 锁定策略

GUP 固定了特定的 folio 后，使用 `folio_lock()` 或 `folio_lock_killable()`，而非 `folio_trylock()`。来自并发迁移/压缩的瞬时锁争用是预期行为，不是永久错误。`folio_trylock()` 仅适用于可以跳过已锁定 folio 的扫描/迭代路径。

当 GUP 后的页表重新验证失败时，从头重试 GUP，而非返回错误——页表变化是瞬时的竞态。

## PFN 扫描代码中的瞬态 Folio 访问

PFN 扫描循环中的 `page_folio()` 是瞬态（speculative）的——复合页结构可能同时发生变化。在稳定化之前访问 folio 标志或大小会导致 `const_folio_flags()` 中的 `VM_BUG_ON` 或读取垃圾值。

**必要的模式**（参见 `split_huge_pages_all()` 在 `mm/huge_memory.c`）：
```c
folio = page_folio(page);                        // 瞬态读取
if (!folio_try_get(folio))                       // 稳定化
    continue;
if (unlikely(page_folio(page) != folio))         // 重新验证
    goto put_folio;
// 现在可以安全访问 folio 标志和状态
```

**应报告为 bug**：在 PFN 扫描循环中对未引用（unreferenced）folio 的 `page_folio()` 调用标志访问器或大小读取。

## PFN 范围迭代与大 Folio

以 `PAGE_SIZE` 为步进的 PFN 循环中调用 `page_folio()` 会与大 folio 冲突：要么尾页被拒绝（如果头页 PFN 不在范围内则遗漏 folio），要么同一 folio 被处理 `folio_nr_pages()` 次（重复记账、重复插入列表）。

**正确模式**：对于非幂等的按 folio 操作（回收、迁移），在找到 folio 时按 `folio_size(folio)` 步进，否则按 `PAGE_SIZE` 步进。对于按 PFN 的位图，保持 `PAGE_SIZE` 步进但跳过尾页。

**应报告为 bug**：PFN 迭代中 `page_folio()` + 非幂等操作 + 无条件 `PAGE_SIZE` 步进的组合。

## 无锁页缓存 Folio 访问

依赖于复合状态的 folio 属性（`folio_mapcount()`、`folio_nr_pages()`、`folio_order()`）和头页标志测试（`folio_test_lru()`）在 RCU 下无引用地与并发 split/free 竞态。稳定化协议：`folio_try_get(folio)` 后跟 `xas_reload()` 验证 folio 仍在同一槽位；失败时重试。

以下情况不需要引用：仅 xarray 元数据操作、不透明指针使用、或直接访问 `folio->flags` 而不依赖复合分支的简单标志测试。

**应报告为 bug**：在 `xas_for_each()` 循环中，在 `rcu_read_lock()` 下调用 `folio_mapcount()`、`folio_nr_pages()` 等，而没有先调用 `folio_try_get()` + `xas_reload()`。

## 页缓存查找后的 folio->private 有效性

`filemap_get_folio()` 或 `filemap_lock_folio()` 的消费者不能假设 `folio->private` 有效。在查找和回收之间存在竞态：`release_folio()` 释放了 `folio->private`，但并行任务可能在缓存中找到该 folio、增加其 refcount，导致 `__remove_mapping()` 失败。在 `folio->private` 中分配状态并在 `release_folio()` 中释放的文件系统，在从页缓存获取 folio 后必须重新验证（并在需要时重新附加）私有数据。

## 页缓存批量迭代：find_get_entries vs find_lock_entries

`find_get_entries()` 和 `find_lock_entries()` 返回的 `indices[i]` 可能不是多阶条目（multi-order entry）的规范基址。`find_lock_entries()` 过滤基址在 `[*start, end]` 之外的条目；`find_get_entries()` 不过滤。假设 `indices[i]` 是规范基址的 `find_get_entries()` 调用者会在截断路径中无限循环。

## XArray 多索引迭代与 xas_next()

`xas_next()` 会访问多阶条目的所有兄弟槽位（包括 sibling 条目），导致重复处理。`xas_find()` / `xas_find_marked()` 内部跳过兄弟条目。使用 `xas_next()` 时，在处理后调用 `xas_advance(&xas, folio_next_index(folio) - 1)` 跳过剩余槽位。对于 order-0 folio 此 bug 不可见。

## 页缓存信息泄露

任何揭示每文件页缓存状态（驻留、脏、写回、已驱逐）的接口都必须通过写权限检查来防止侧信道攻击。需要对新的系统调用、ioctl 以及 procfs/sysfs 接口执行此检查。

## 页缓存 XArray 设置（mapping_set_update）

任何对 `mapping->i_pages` 执行可变操作的 `XA_STATE` 都必须先调用 `mapping_set_update(&xas, mapping)`。这设置了工作集影子节点跟踪的回调。没有它，xa_nodes 不会被添加到其 memcg 的 `list_lru` 中，在内存压力下会泄漏节点。

## Per-CPU LRU 缓存批处理

大 folio 永远不会出现在 per-CPU LRU 缓存中。使用 `folio_may_be_lru_cached(folio)` 守卫 per-folio 的 `lru_add_drain()` / `lru_add_drain_all()` 调用。

## Folio 驱逐和失效守卫

在未检查脏/写回状态的情况下将 folio 从页缓存中移除会导致数据丢失或正在进行的 IO 损坏。两个标志都异步变化；检查必须在 `folio_lock()` 之后，而非之前。

**必要的模式**（参见 `mapping_evict_folio()` 在 `mm/truncate.c`）：
```c
folio_lock(folio);
if (folio_test_dirty(folio) || folio_test_writeback(folio))
    goto skip;
/* 可以安全地从页缓存中移除 */
```

## 快速检查清单

- **标志测试或访问 mapping 前必须有 folio 引用**：对未引用 folio 调用 `folio_test_*()` 如果内存已被重用作尾页会导致崩溃（`const_folio_flags()` 断言非尾页）。瞬态查找中 `folio_try_get()` 必须在标志测试之前；`folio_get()` 必须在 `set_pte_at()` 之前
- **复合页尾页**：页缓存字段（`mapping`、`index`、`private`）在尾页中与 `compound_head` 共享 union——在尾页上访问它们会静默返回垃圾值。先调用 `compound_head()` 或 `page_folio()`
- **`folio_page()` vs PTE 映射的子页**：`folio_page(folio, 0)` 返回头页，而非特定 PTE 映射的子页。在大 folio 的 PTE 批量循环中，除非批处理从 folio 偏移 0 开始，否则使用 `vm_normal_page()` 获取实际子页
- **`folio_page()` 索引边界**：`folio_page(folio, n)` 执行无检查的算术；`n >= folio_nr_pages(folio)` 会访问 struct page 数组之后。当 `n` 从截断/split 路径中的字节偏移算术计算时，验证边界情况不产生越界索引
- **潜在 refcount 释放后的复合元数据**：在可能释放最后一个引用的调用后读取 `compound_order()` / `folio_nr_pages()` 会返回垃圾值。在释放引用的调用前快照元数据
- **page-to-folio 转换后的 PFN 前进**：从尾页开始时，`folio_nr_pages()` 对前进 PFN 是错误的。使用 `pfn += folio_nr_pages(folio) - folio_page_idx(folio, page) - 1`
- **非复合高阶页上的 `page_size()` / `compound_order()`**：`compound_order()` 对非复合页返回 0。当页面没有使用 `__GFP_COMP` 分配时，使用 `PAGE_SIZE << order`。Folio API 是安全的（folio 总是复合或 order-0）
- **`_mapcount` +1 偏置约定**：`_mapcount` 初始化为 -1；逻辑 mapcount = `_mapcount + 1`。当代码直接读取 `_mapcount` 时，验证消费者期望的是原始值（-1 基准）还是逻辑值（0 基准）
- **Refcount 作为语义状态**：`page_count()` / `folio_ref_count()` 是生命周期计数器，不是语义指示器。瞬态引用人为抬高它们。使用专用计数器/标志表示语义状态
- **`folio_end_read()` 在已 uptodate 的 folio 上**：对 `PG_uptodate` 使用 XOR，因此在已 uptodate 的 folio 上以 `success=true` 调用会关闭该标志。可能遇到 uptodate folio 的路径应使用 `folio_unlock()` 代替
- **refcount 释放失败后的页/folio 访问**：当 `put_page_testzero()` / `folio_put_testzero()` 返回 false 时，调用者没有引用——另一个 CPU 可能立即释放页面。在 refcount 释放前保存需要的元数据
- **非匿名 folio 的 `folio->mapping` 为 NULL**：`folio->mapping` 对 swap 缓存中的 shmem folio 和已截断的 folio 为 NULL。先检查 NULL 再访问 mapping 成员
- **XArray 多阶条目原子性**：在 `rcu_read_lock()` 下使用 `xa_get_order()` 然后在 `xa_lock` 下对 order 进行操作是 TOCTOU 竞态（条目顺序可能在操作间变化）。在一次 `xas_lock_irq()` 区间内组合 `xas_load()`、`xas_get_order()` 和 `xas_store()`
- **错误标签处的 Folio 锁定状态**：当函数提前获取 `folio_lock()` 并跳转到错误标签时，清理必须调用 `folio_unlock()`。验证 folio 在每个调用 `folio_unlock()` 的标签处确实已锁定
- **获取锁后重新检查 folio 状态**：`folio_lock()` 可能睡眠。锁定前检查的 folio 状态可能已变化。总是重新验证 `folio->mapping != NULL`
- **Zone 设备页元数据重新初始化**：ZONE_DEVICE 页面绕过 `prep_new_page()`，因此在以不同 order 重用时遗留的复合元数据会持续存在。必须在 `prep_compound_page()` 前清除所有 per-page 复合元数据
- **`page_folio()` / `compound_head()` 需要 vmemmap 驻留页**：在 `CONFIG_HUGETLB_PAGE_OPTIMIZE_VMEMMAP` 下，`page_fixed_fake_head()` 访问 `page[1].compound_head`。对栈本地或单元素拷贝调用是越界读取。对页快照，直接内联 compound_head 位测试
- **大 folio 的 per-section 元数据迭代**：在 `CONFIG_SPARSEMEM` 下，`page_ext` 数组是按 section 的，不连续。跨 section 边界的指针算术会崩溃。使用 `for_each_page_ext()` / `page_ext_iter_next()`
- **Folio 转换中的页数统计**：将单页代码转换为大 folio 时，每个 `counter++` 和硬编码的 `1` 都必须变为 `folio_nr_pages(folio)`
- **Folio order vs 映射条目 order**：swapin 路径必须验证 `folio_order()` 与映射条目 order 匹配。预读可能插入 order-0 folio 到被大映射条目覆盖的槽位中；插入时不拆分大条目会静默丢失数据
- **`kmap_local_page()` 只映射单个页面**：在 CONFIG_HIGHMEM 上，从返回地址访问 PAGE_SIZE 之外会触发故障。使用 `kmap_local_page(page + i)` 迭代多页访问
- **`pfn_valid()` vs `pfn_to_online_page()`**：`pfn_valid()` 只确认 struct page 存在；页面可能已离线。在 hwpoison、迁移和任何会访问页面元数据的路径中使用 `pfn_to_online_page()`
- **边界 PFN 上的 `pfn_to_page()`**：只有 struct page 有效的 PFN 上才安全。PFN 范围循环必须在 `pfn_to_page()` 之前检查终止，而非之后
