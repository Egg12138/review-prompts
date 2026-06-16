<!-- Source: subsystem/mips.md -->

# MIPS 架构知识

## TLB 重复条目危害

在早期初始化期间使用内容寻址的 TLB 操作会在多个 MIPS CPU 家族上触发 TLB 关闭（机器检查异常）。TLB 关闭在 `CP0_Status` 中设置 `ST0_TS` 位，是致命的——处理器停止，内核在 `arch/mips/kernel/traps.c` 中的 `do_mcheck()` 处理器无法恢复。

### 受影响的 CPU 家族

TLB 关闭受 R4x00、microAptiv/M5150、Cavium OCTEON3 和 SB1 内核影响。`TLBP`、`TLBWI` 和 `TLBWR` 指令如果 TLB 中存在多个匹配条目都可能触发关闭。

### 危险的初始状态

引导加载器和固件可能使 TLB 处于触发关闭的病态状态。例如，SGI IP22 PROM 将所有 TLB 条目初始化为同一个虚拟地址。其他有问题的状态包括来自不完整的前一次启动尝试的重复条目，以及恰好创建冲突的垃圾值。

### 安全 vs 不安全的 TLB 初始化操作

| 操作 | 指令 | 初始化期间的安全性 |
|---|---|---|
| 索引读取 | `TLBR` via `tlb_read()` | 安全——按索引读取条目 |
| 索引写入 | `TLBWI` via `tlb_write_indexed()` | 不安全——如果创建重复条目 |
| 内容探测 | `TLBP` via `tlb_probe()` | 不安全——可能在副本上关闭 |
| 随机写入 | `TLBWR` via `tlb_write_random()` | 不安全——如果创建重复条目 |

### 安全的初始化模式

内核的 `r4k_tlb_uniquify()` 在 `arch/mips/mm/tlb-r4k.c` 中演示了正确的方法——首先按索引读取所有条目，在软件中检测重复，然后使用不会创建新冲突的索引写入覆盖重复：

```c
// 错误：在 TLB 状态验证前进行探测
for (entry = 0; entry < tlbsize; entry++) {
    write_c0_entryhi(UNIQUE_ENTRYHI(entry));
    tlb_probe();  // 危险：如果存在重复则关闭
    if (read_c0_index() >= 0) {
        /* 处理冲突 */
    }
}

// 正确：首先使用索引操作读取所有条目（r4k_tlb_uniquify 模式）
for (i = 0; i < tlbsize; i++) {
    write_c0_index(i);
    mtc0_tlbr_hazard();
    tlb_read();  // 安全的索引读取
    tlb_read_hazard();
    existing_vpns[i] = read_c0_entryhi() & vpn_mask;
}
// 在软件中检测重复，然后使用唯一值覆盖
// 使用 tlb_write_indexed() —— 安全，因为每次写入移除一个重复
```

TLB 指令包装器（`tlb_read()`、`tlb_write_indexed()`、`tlb_probe()` 等）和危险屏障（`mtc0_tlbr_hazard()`、`tlb_read_hazard()` 等）定义在 `arch/mips/include/asm/mipsregs.h` 中。

---

## 快速检查清单

- **初始化期间的 `tlb_probe()`**：在 TLB 被唯一化之前，任何在早期启动代码中使用 `tlb_probe()` 的行为都是可疑的。检查引导加载器状态是否可能导致重复条目。
- **`TLBWI`/`TLBWR` 创建重复**：如果索引和随机写入创建了与现有条目匹配的第二个条目，它们也可能触发关闭。在初始化期间，条目必须以已知唯一的值写入。
- **引导加载器状态假设**：假设内核入口时 TLB 状态干净或为零的代码是不安全的。不同引导加载器的行为不同（例如，SGI IP22 PROM 将所有条目设置为相同的 VPN）。
- **危险屏障**：TLB 操作在 CP0 写入和 TLB 指令之间需要架构特定的危险屏障。参见 `mtc0_tlbr_hazard()`、`tlb_read_hazard()`、`mtc0_tlbw_hazard()`、`tlbw_use_hazard()`。
