<!-- Source: subsystem/io-accessors.md -->

# MMIO 访问器详解

## MMIO 访问器语义和字节序（Endianness）

在访问同一个 FIFO 或缓冲区时使用不一致的 I/O 访问器变体，会导致大端（big-endian，大字节序）系统上的数据损坏。批量传输可能看起来正确，而剩余部分传输由于意外的字节交换（byteswapping）而产生损坏数据。

Linux 提供两族 MMIO 访问器，具有不同的字节序行为（定义在 `include/asm-generic/io.h`，架构可以覆盖）：

### 寄存器 I/O（带字节交换）

- `readl()` / `writel()` —— 32 位
- `readw()` / `writew()` —— 16 位
- `readb()` / `writeb()` —— 8 位
- **行为**：通过 `__cpu_to_leN()` / `__leN_to_cpu()` 封装在 `__raw_readN()` / `__raw_writeN()` 周围，执行 CPU 到设备的字节序转换，外加内存屏障（`__io_br()` / `__io_ar()` 用于读取，`__io_bw()` / `__io_aw()` 用于写入）。在大端 CPU 上，这些函数对数据进行字节交换以产生设备期望的小端（little-endian）数据。
- **用途**：具有定义位布局的硬件控制/状态寄存器

### FIFO / 流 I/O（无字节交换）

- `readsl()` / `writesl()` —— 32 位流
- `readsw()` / `writesw()` —— 16 位流
- `readsb()` / `writesb()` —— 8 位流
- **行为**：直接映射到 `__raw_readN()` / `__raw_writeN()`（例如 `__raw_readl()`、`__raw_writew()`），保留内存和 FIFO 之间的字节顺序。无论 CPU 字节序如何，**不执行字节交换**，也不设置每访问屏障。
- **用途**：数据 FIFO、DMA 缓冲区、面向流的硬件

### 混合访问器的错误模式

对批量 FIFO 传输使用 `writesl()` 或 `readsl()`，但对同一个 FIFO 地址的剩余/部分传输使用 `writel()` 或 `readl()` 的代码，是一个字节序可移植性 bug——剩余字节在大端系统上会被字节交换，而批量数据不会。

```c
// 错误：混合访问器语义
writesl(fifo_addr, buffer, len / 4);
if (len & 3) {
    u32 tmp = 0;
    memcpy(&tmp, buffer + (len & ~3), len & 3);
    writel(tmp, fifo_addr);  // BUG：在大端上会字节交换
}
```

```c
// 正确：保持一致的 FIFO 语义
writesl(fifo_addr, buffer, len / 4);
if (len & 3) {
    u32 tmp = 0;
    memcpy(&tmp, buffer + (len & ~3), len & 3);
    writesl(fifo_addr, &tmp, 1);  // 一致：无字节交换
}
```

同样的模式适用于读取：

```c
// 错误
readsl(fifo_addr, buffer, len / 4);
if (len & 3) {
    u32 tmp = readl(fifo_addr);  // BUG：在大端上会字节交换
    memcpy(buffer + (len & ~3), &tmp, len & 3);
}
```

```c
// 正确
readsl(fifo_addr, buffer, len / 4);
if (len & 3) {
    u32 tmp;
    readsl(fifo_addr, &tmp, 1);  // 一致：无字节交换
    memcpy(buffer + (len & ~3), &tmp, len & 3);
}
```

## Invariants（不变规则）

- 控制/状态寄存器 → 使用 `readl()` / `writel()` 族（带字节交换和屏障）
- 数据 FIFO / 流缓冲区 → 使用 `readsl()` / `writesl()` 族（不带字节交换）
- 同一 FIFO 的批量传输和剩余传输必须使用同一族访问器

## Failure Modes（失败模式）

- 对 FIFO 地址混合使用 `writesl()`（批量）和 `writel()`（剩余）→ 大端系统上剩余字节被字节交换 → 数据损坏。
- 对数据流使用寄存器 I/O 访问器 → 不必要的字节交换开销。
- 对控制寄存器使用流 I/O 访问器 → 缺少必要的屏障和字节序转换。

## Quick Checks（快速检查要点）

- **批量 vs 剩余访问器一致性**：当代码分别处理批量传输和部分/剩余传输时，确认两者使用同一 I/O 访问器家族（`writesl`/`readsl` vs `writel`/`readl`）。
- **FIFO 识别**：确定目标地址是 FIFO/缓冲区（流数据）还是寄存器（控制/状态）。FIFO 应独占使用流访问器。
- **大端测试**：标记混合访问器类型的 FIFO 辅助函数——很可能在大端架构上失败。
