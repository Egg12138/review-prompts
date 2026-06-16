<!-- Source: mm-alloc.md -->

# MM 内存分配与 GFP 语义

## GFP 标志上下文

使用错误的 GFP 标志会导致在原子上下文中睡眠（死锁/BUG）、文件系统或 IO 递归（死锁），或当调用者假设成功时发生静默分配失败。验证分配上下文与标志匹配。

**Reclaim 列**指示可用的内存回收机制。"kswapd only" 表示分配会唤醒后台 kswapd 线程但从不阻塞等待回收完成。"Full" 表示调用者还可能同步执行直接回收（direct reclaim），阻塞直到页面被释放。

| 标志 | 可睡眠？ | 回收机制 | 关键标志 | 使用场景 |
|------|---------|---------|---------|---------|
| GFP_ATOMIC | 否 | 仅 kswapd | `__GFP_HIGH \| __GFP_KSWAPD_RECLAIM` | IRQ/spinlock 上下文、更低水位访问 |
| GFP_KERNEL | 是 | Full（直接回收 + kswapd） | `__GFP_RECLAIM \| __GFP_IO \| __GFP_FS` | 正常内核分配 |
| GFP_NOWAIT | 否 | 仅 kswapd | `__GFP_KSWAPD_RECLAIM \| __GFP_NOWARN` | 不可睡眠，可能失败 |
| GFP_NOIO | 是 | 直接回收 + kswapd，无 IO | `__GFP_RECLAIM` | 避免块 IO 递归 |
| GFP_NOFS | 是 | 直接回收 + kswapd，无 FS | `__GFP_RECLAIM \| __GFP_IO` | 避免文件系统递归 |

参考 "Useful GFP flag combinations" 在 `include/linux/gfp_types.h`。

**注意：**
- `__GFP_RECLAIM` = `__GFP_DIRECT_RECLAIM | __GFP_KSWAPD_RECLAIM`
- GFP_NOIO 仍可以直接回收干净的页缓存和 slab 页面（无物理 IO）
- 优先使用 `memalloc_nofs_save()` / `memalloc_noio_save()` 而不是 GFP_NOFS / GFP_NOIO
- `__GFP_KSWAPD_RECLAIM`（存在于 GFP_NOWAIT 和 GFP_ATOMIC 中）会触发 `wakeup_kswapd()`，它调用 `wake_up_interruptible()` 并通过 `try_to_wake_up()` 进入调度器。这意味着即使是不可睡眠的分配也可能获取调度器和定时器锁。在调度器内部锁（如 hrtimer base lock、runqueue lock）下分配或抢占被禁用的代码必须剥离 `__GFP_KSWAPD_RECLAIM` 或使用裸标志，以避免锁递归。参见 `gfp_nested_mask()` （`include/linux/gfp.h`）了解标准化方法
- `current_gfp_context()` 在 task 运行在 `memalloc_noio_save()` 或 `memalloc_nofs_save()` 约束下时会剥离 `__GFP_IO` 和/或 `__GFP_FS`。缩窄后，`GFP_KERNEL` 分配变成了 `GFP_NOIO` 或 `GFP_NOFS`，它们仍然包含 `__GFP_DIRECT_RECLAIM`（可以睡眠）。对缩窄后的值与复合常量测试会误分类为原子。使用单标志辅助函数：`gfpflags_allow_blocking(gfp)` 测试 `__GFP_DIRECT_RECLAIM`（此分配能否睡眠？），`gfpflags_allow_spinning(gfp)` 测试 `__GFP_RECLAIM`（此分配能否获取锁？）。参见 `include/linux/gfp.h`

**放置约束**（参见 "Page mobility and placement hints" 在 `include/linux/gfp_types.h`）：
- `GFP_ZONEMASK` 选择物理内存区域。截获分配并从预分配池提供内存的代码必须跳过其无法满足的区域约束请求
- `__GFP_THISNODE` 强制在请求的 NUMA 节点上分配，无回退。它不在 `GFP_ZONEMASK` 中——仅检查 `GFP_ZONEMASK` 会遗漏此约束
- 剥离放置标志进行验证时，使用完整的标志集

## __GFP_ACCOUNT

不正确的 memcg 记账使容器能够分配内核内存而不被记账，绕过其内存限制。审查任何新的 `__GFP_ACCOUNT` 使用或 `SLAB_ACCOUNT` 缓存创建。

- 使用 `SLAB_ACCOUNT` 创建的 slab 会自动通过 `memcg_slab_post_alloc_hook()` 向 memcg 记账，即使在分配调用中没有显式 `__GFP_ACCOUNT`

**验证：**
1. 使用 `__GFP_ACCOUNT` 时，确保正确的 memcg 被记账：`old = set_active_memcg(memcg); work; set_active_memcg(old)`
2. 大多数使用不需要 `set_active_memcg()`，但 kthread 在多个 memcg 间切换上下文时可能需要
3. 确保新的 `__GFP_ACCOUNT` 使用与周围代码一致

## Mempool 分配保证

`mempool_alloc()` 在设置了 `__GFP_DIRECT_RECLAIM` 时永远重试——NULL 检查是死代码。没有它时可以失败——缺少 NULL 检查会导致崩溃。错误处理必须与 GFP 标志匹配。

## 冻结（Frozen）vs 引用计数（Refcounted）页分配

`get_page_from_freelist()` 返回 refcount 为 0 的页（"frozen"）。`__alloc_pages_noprof()` 封装此函数并调用 `set_page_refcounted()` 返回 refcount 1。`_frozen_` 变体为自行管理 refcount 的调用者返回 frozen 页。**应报告为 bug**：将 frozen 页传给期望 refcount 1 的代码而不先调用 `set_page_refcounted()`，或在应该保持 frozen 的页上调用 `set_page_refcounted()`。

## Zone 水位和 lowmem_reserve

`zone[i].lowmem_reserve[j]` 保护 zone `i`（而非 zone `j`）免于被针对 zone `j` 的分配过度消耗。有效水位为 `watermark[wmark] + lowmem_reserve[j]`。zone 自身的条目总是 0。**应报告为 bug**：用 zone 自身的索引索引 `lowmem_reserve`，或假设 `lowmem_reserve[j]` 保护 zone `j`。

**Per-CPU vmstat 计数器漂移**：`zone_page_state()` 省略 per-CPU 增量；在多 CPU 系统上，误差可能超过水位间隙。当 `zone->percpu_drift_mark` 已设置且缓存值低于它时，代码必须使用 `zone_page_state_snapshot()`。

## Zone 水位初始化顺序

Zone 水位在 `init_per_zone_wmark_min()` 作为 `postcore_initcall` 运行前为零。在此之前，`zone_watermark_ok()` 会平凡地通过，掩盖了对回收/接纳的需要。早期启动期间可达的代码必须将 `wmark == 0` 处理为"尚未初始化"。

## 分层 vmstat 记账（Node vs Memcg）

`lruvec_stat_mod_folio()` / `mod_lruvec_page_state()` 仅在 `folio_memcg(folio)` 非 NULL 时同时更新 node 和 memcg 计数器；否则只更新 node 计数器。`mod_node_page_state()` 总是仅 node；`mod_lruvec_state()` 总是两者都更新。

**延迟记账的统计协调**：当 folio 在没有 memcg 的情况下分配并记录了统计信息时，只有 node 计数器递增。如果之后被记账，后记账路径必须从 node 计数器减去并通过 lruvec 接口重新添加以填充 memcg 计数器，否则释放路径会下溢。

审查任何在分配后更改 folio 的 memcg 关联性的代码路径。

## Slab 页叠加初始化和清理

`struct slab` 叠加在 `struct page` / `struct folio` 上（由 `mm/slab.h` 中的 `SLAB_MATCH` 断言验证）。页分配器不归零元数据字段，因此 `allocate_slab()` 必须初始化每个字段——尤其是条件编译的那些。

释放时，`slab->obj_exts` 与 `folio->memcg_data` 共享存储。遗留的哨兵值会触发 `VM_BUG_ON_FOLIO` 或 `free_page_is_bad()`。`unaccount_slab()` 中的 `free_slab_obj_exts()` 必须无条件调用（不由 `mem_alloc_profiling_enabled()` 或 `memcg_kmem_online()` 门控），因为两者在分配和释放之间可能在运行时改变。

## Trylock-Only 分配路径（ALLOC_TRYLOCK）

`alloc_pages_nolock()` / `alloc_frozen_pages_nolock()` 设置 `ALLOC_TRYLOCK` 并清除回收 GFP 标志。辅助函数必须检查 `ALLOC_TRYLOCK` 或 `gfpflags_allow_spinning()` 并跳过无条件锁，否则只对 **瞬时** 条件使用粗略的退出路径。

**应报告为 bug**：从 `get_page_from_freelist()` 可达的辅助函数使用 `spin_lock()` 而没有 `ALLOC_TRYLOCK` / `gfpflags_allow_spinning()` 检查。

## Memblock 范围参数约定

Memblock 使用两种约定：`(base, size)` 用于 `memblock_add()`、`memblock_remove()` 等；`(start, end)` 用于 `reserve_bootmem_region()`、`__memblock_find_range_*()`。两个参数都是 `phys_addr_t`——没有编译器类型安全。常见错误：在同时计算 `start = region->base` 和 `end = start + region->size` 的循环中，将 `end` 传给期望 `size` 的函数（反之亦然）。

## Realloc 零化生命周期

原地 realloc 收缩路径必须在 `want_init_on_free()` 或 `want_init_on_alloc(flags)` 为 true 时零化 `[new_size, old_size)`。零化在 `want_init_on_alloc` 时是必须的，因为后续的原地增长不能重新暴露过期数据。

**常见错误**：收缩时只检查 `want_init_on_free()`——遗漏了 `init_on_alloc` 情况。

## kmemleak 跟踪对称性

分配/释放 API 必须为 kmemleak 成对对称：`kmalloc()` 与 `kfree()` / `kfree_rcu()` 等。混用会导致 "Trying to color unknown object" 警告或假阳性泄漏报告。

SLUB 在 `!gfpflags_allow_spinning(flags)` 时跳过 kmemleak 注册。当分配路径有条件地跳过注册时，所有后续的 kmemleak 状态变更调用必须由相同条件守卫。

## 快速检查清单

- **`NODE_DATA()` 前的 NUMA 节点 ID 验证**：`NODE_DATA(nid)` 没有边界检查。用户提供的节点 ID 需要：`nid >= 0 && nid < MAX_NUMNODES && node_state(nid, N_MEMORY)`
- **`get_node(s, numa_mem_id())`** 在无内存节点上可能返回 NULL。缺少 NULL 检查会导致仅在 NUMA 系统上触发的空指针解引用
- **分配循环的节点掩码选择**：`for_each_online_node()` 包含无内存节点。内存分配使用 `for_each_node_state(nid, N_MEMORY)`。早期启动期间 `N_MEMORY` 可能尚未填充
- **NUMA 节点计数 vs 节点 ID 范围**：`num_node_state()` 返回计数而非 ID 的上界。使用 `nr_node_ids` 作为原始迭代的上界
- **NUMA mempolicy 感知 vs 节点特定分配**：`alloc_pages_node()` / `__alloc_pages_node()` 绕过 task NUMA 策略。将 `alloc_pages()` / `folio_alloc()` 替换为 `_node` 变体会静默丢弃 mempolicy
- **分配辅助函数中的 GFP 标志传播**：包装分配并添加自己的 GFP 标志时，必须通过按位 OR 保留调用者的标志，而非替换
- **SLUB `!allow_spin` 重试循环**：`___slab_alloc()` 中，trylock 失败后 `goto` 回重试必须检查 `!allow_spin` 并返回 NULL
- **SLUB 内部中的 KASAN 标签重置**：访问已释放对象内存的新 `mm/slub.c` 代码必须先调用 `kasan_reset_tag()`
- **`__GFP_MOVABLE` 移动性契约**：用 `__GFP_MOVABLE` 分配的页必须是可回收或可迁移的。**应报告为 bug**：`__GFP_MOVABLE` 用在没有迁移支持的页上
- **页分配器重试循环终止**：每个 `goto retry` 必须修改状态，阻止下一次迭代走相同路径。验证 `&= ~FLAG` 而非 `&= FLAG`
- **页分配器重试 vs restart seqcount 一致性**：每个 `goto retry` 必须调用 `check_retry_cpuset()` / `check_retry_zonelist()` 在过期时重定向到 `restart`
- **高阶页的 Pageblock migratetype 更新**：使用 `change_pageblock_range()` 而非裸 `set_pageblock_migratetype()`
- **批量路径中的页分配器回退成本**：`rmqueue_bulk()` 在 `zone->lock` 下循环。回退更改在批次中每页倍增，造成延迟尖峰
- **PCP 锁包装要求**：`pcp->lock` 必须使用 PCP 特定的包装器，而非裸 `spin_lock()`
- **缓存别名架构上的用户页零化**：`__GFP_ZERO` 使用 `clear_page()`，跳过了 `clear_user_highpage()` 提供的 dcache 刷写。使用 `user_alloc_needs_zeroing()` 检查
- **vmalloc poison/unpoison 中的 KASAN 粒度对齐**：`kasan_poison()` / `kasan_unpoison()` 需要 `KASAN_GRANULE_SIZE` 对齐的地址。使用 `kasan_vrealloc()`
- **分配路径上的 `static_branch_*()`**：内部获取 `cpus_read_lock()`。从 CPU bringup 期间的页分配器调用会死锁
- **早期启动使用 MM 全局变量**：`high_memory` 和 zone PFN 在 `free_area_init()` 之前为零。使用 `memblock_end_of_DRAM()`
- **早期启动内存分配失败**：`__init` 函数通常不需要处理分配失败——此时物理内存应可用，失败通常意味着系统无法启动
- **NOWAIT 错误码转换**：NOWAIT 调用者期待 `-EAGAIN`，而非 `-ENOMEM`。降级 GFP 到 NOWAIT 时，将分配失败翻译为 `-EAGAIN`
- **回收可达路径中锁下的 GFP_KERNEL**：`GFP_KERNEL` 可能触发直接回收，通过换出、写回或 slab 收缩重新进入 MM。如果分配持有回收也获取的锁就会死锁
- **Slab freelist 指针访问必须使用访问器**：使用 `CONFIG_SLAB_FREELIST_HARDENED` 时，freelist 指针是 XOR 编码的。使用 `get_freepointer()` / `set_freepointer()`
- **Slab 后分配/释放钩子对称性**：`slab_post_alloc_hook()` 运行 KASAN、kmemleak、KMSAN 等。当后期钩子失败时，无错误路径必须撤销所有已运行的钩子
- **页释放前恢复直接映射**：当页已从内核直接映射中移除时，在页面释放回分配器之前必须恢复直接映射条目
