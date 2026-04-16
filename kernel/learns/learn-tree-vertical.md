# Kernel Review Prompts 指向树（全纵向 Mermaid 版）

这份文件是 `learn-tree.md` 的补充版本，目标不是把所有节点挤成一张“大树”，而是让每一组下级节点都按纵向顺序展开。

因此，这里采用的表达方式是：

- 先给一张总览图
- 再把每一组 guide 拆成独立的小图
- 每张小图都只保留一条纵向链路

这样阅读时不会再出现单层横向扇出太宽的问题。

## 1. 总体工作机制：入口、主协议、条件加载

```mermaid
flowchart TD
    A[review-prompts/kernel/]
    A --> B[README.md]
    A --> C[skills/kernel.md]
    A --> D[slash-commands/kreview.md]
    A --> E[review-core.md]
    A --> F[technical-patterns.md]
    A --> G[subsystem/subsystem.md]

    B -->|项目说明 / 人工阅读入口| E

    C -->|进入 kernel tree 自动加载| F
    C -->|patch review| E
    C -->|subsystem context| G
    C -->|debugging task| H[debugging.md]
    C -->|coccinelle task| I[coccinelle.md]

    D -->|执行 /kreview| E

    E -->|always load first| F
    E -->|must load| G
    E -->|non-trivial patch| J[callstack.md]
    E -->|lore available| K[lore-thread.md]
    E -->|Fixes tag check required| L[missing-fixes-tag.md]
    E -->|subjective review + existing Fixes tag| M[fixes-tag.md]
    E -->|regressions found| N[false-positive-guide.md]
    E -->|confirmed regressions| O[inline-template.md]

    F -->|RCU primitive seen| P[subsystem/rcu.md]
    J -->|RCU primitive seen| P
    N -->|NULL deref issue| Q[pointer-guards.md]
```

## 2. 渐进式披露主干：总览

```mermaid
flowchart TD
    A[review-core.md]
    A --> B[technical-patterns.md]
    A --> C[subsystem/subsystem.md]

    C --> D1[MM family]
    C --> D2[Core kernel family]
    C --> D3[I/O and storage family]
    C --> D4[Net and protocol family]
    C --> D5[Drivers and platform family]
    C --> D6[Tools and config family]
    C --> D7[Language and special topics]
    C --> D8[Optional patterns]
```

## 3. 每组节点纵向排列

### 3.1 MM family

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[MM family]
    B --> C1[mm-pagetable.md]
    C1 --> C2[mm-folio.md]
    C2 --> C3[mm-largepage.md]
    C3 --> C4[mm-vma.md]
    C4 --> C5[mm-alloc.md]
    C5 --> C6[mm-reclaim.md]
```

### 3.2 Core kernel family

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[Core kernel family]
    B --> C1[vfs.md]
    C1 --> C2[locking.md]
    C2 --> C3[scheduler.md]
    C3 --> C4[timers.md]
    C4 --> C5[rcu.md]
    C5 --> C6[tracing.md]
    C6 --> C7[workqueue.md]
    C7 --> C8[syscall.md]
    C8 --> C9[cleanup.md]
    C9 --> C10[kho.md]
```

### 3.3 I/O and storage family

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[I/O and storage family]
    B --> C1[block.md]
    C1 --> C2[dax.md]
    C2 --> C3[io_uring.md]
    C3 --> C4[pci.md]
    C4 --> C5[ata.md]
    C5 --> C6[usb-storage.md]
    C6 --> C7[io-accessors.md]
    C7 --> C8[cxl.md]
    C8 --> C9[dpll.md]
```

### 3.4 Net and protocol family

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[Net and protocol family]
    B --> C1[networking.md]
    C1 --> C2[bpf.md]
    C2 --> C3[sunrpc.md]
    C3 --> C4[nfsd.md]
    C4 --> C5[bluetooth.md]
    C5 --> C6[wireless.md]
    C6 --> C7[can.md]
    C7 --> C8[smb-ksmbd.md]
```

### 3.5 Drivers and platform family

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[Drivers and platform family]
    B --> C1[drm.md]
    C1 --> C2[pmdomain.md]
    C2 --> C3[pm.md]
    C3 --> C4[sysfs.md]
    C4 --> C5[tty.md]
    C5 --> C6[of.md]
    C6 --> C7[hwmon.md]
    C7 --> C8[media.md]
    C8 --> C9[input.md]
    C9 --> C10[i3c.md]
    C10 --> C11[irqchip.md]
    C11 --> C12[fscrypt.md]
    C12 --> C13[btrfs.md]
```

### 3.6 Tools and config family

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[Tools and config family]
    B --> C1[perf.md]
    C1 --> C2[selftests.md]
    C2 --> C3[objtool.md]
    C3 --> C4[kconfig.md]
    C4 --> C5[dt-bindings.md]
```

### 3.7 Language and special topics

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[Language and special topics]
    B --> C1[mips.md]
    C1 --> C2[rust.md]
```

### 3.8 Optional patterns

```mermaid
flowchart TD
    A[subsystem/subsystem.md]
    A --> B[Optional patterns]
    B --> C1[subjective-review.md]
```

## 4. 这种画法的含义

这种版本不是在强调“真实依赖关系是一个链表”，而是在强调“阅读顺序应该是纵向展开的”。

也就是说：

- 逻辑上，`subsystem/subsystem.md` 仍然是一个分发表
- 视觉上，我们故意把每一组 guide 画成纵向链，避免一层横向展开得太宽
- 这更适合在 Markdown 渲染器里阅读，也更适合在窄屏或普通编辑器里查看

## 5. 一句话总结

如果目标是表达“知识如何分组并逐层展开”，那么全纵向 Mermaid 比单张大树更容易读。
