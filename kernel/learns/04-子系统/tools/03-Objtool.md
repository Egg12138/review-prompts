<!-- Source: subsystem/objtool.md -->

# Objtool 子系统详解

## 指令分类语义

在 `tools/objtool/arch/*/decode.c` 中错误分类指令会破坏 objtool 的控制流分析：

- 将 `INSN_BUG` 错误分类为 `INSN_TRAP`，会丢失 dead-end 传播，可能导致 "unreachable instruction" 警告被错误抑制
- 将 `INSN_TRAP` 错误分类为 `INSN_BUG`，会使 objtool 将每条后续指令视为死代码（dead code），隐藏真正的错误

### INSN_BUG vs INSN_TRAP 语义

`INSN_BUG` 和 `INSN_TRAP` 在 `tools/objtool/check.c` 中有不同的效果：

| 分类 | 效果 |
|---|---|
| `INSN_BUG` | 在 `decode_instructions()` 中将指令的 `dead_end` 设为 `true`，标记所有后续代码为不可达。还用于 `validate_retpoline()` 识别间接调用保护模式，以及 `ignore_unreachable_insn()` 抑制已知死端后代码的警告 |
| `INSN_TRAP` | 在 `ignore_unreachable_insn()` 中被视为可忽略的填充（padding，类似 `INSN_NOP`）。在 `ret` 和间接跳转之后，`validate_sls()` 需要它用于 Straight-Line Speculation 缓解 |

### 架构特定映射

`tools/objtool/arch/*/decode.c` 中的架构特定映射：

| 架构 | `INSN_BUG` | `INSN_TRAP` |
|---|---|---|
| x86 | `ud2`、`ud1`、`udb` | `int3` |
| LoongArch | `amswap.w $zero, $ra, $zero`；`break 0x1` | `break 0x0` |

### Invariants

- `INSN_BUG` 标记基本块结束，其后所有指令被视为不可达的死代码
- `INSN_TRAP` 是可忽略的填充指令，不会截断控制流
- 指令分类必须与该架构的 trap handler 在运行时处理该指令的方式匹配（例如 LoongArch 上 `break 0x1` 触发 `BUG()` 处理，与其 `INSN_BUG` 分类一致）

### Failure Modes

**假阴性（False Negative）**：`INSN_BUG` 被误分类为 `INSN_TRAP`，dead-end 标记丢失，导致本应报告的 "unreachable instruction" 警告被静默抑制。

**假阳性（False Positive）**：`INSN_TRAP` 被误分类为 `INSN_BUG`，objtool 认为后续所有指令都是死代码，隐藏了这些路径中的真实错误。

### Quick Checks

- 验证 `tools/objtool/arch/*/decode.c` 中的指令分类是否与该架构 trap handler 的运行时行为一致

---

## 编译器生成的陷阱指令

重新分类 objtool 对特定机器指令的处理方式，可能会暴露编译器生成的同种指令实例，导致在其他方面正确的构建中出现 spurious objtool warnings。

### 生成陷阱/断点指令的 GCC 优化

可能独立于显式内核代码生成 trap/break 指令的 GCC 优化：

- `-fisolate-erroneous-paths-dereference`（由 `-O2` 启用）：GCC 在可证明会解引用空指针的代码路径上插入 trap 指令
- `-fsanitize=unreachable`、`-fsanitize=undefined`：sanitizers 在可证明的未定义行为处插入 trap 指令

### Invariants

- 当 objtool 分类发生变更时，架构 Makefile（`arch/*/Makefile`）可能需要相应的编译器标志变更，以抑制不必要的指令生成
- 例如 `arch/loongarch/Makefile` 同时设置了 `-mno-check-zero-division` 和 `-fno-isolate-erroneous-paths-dereference`，以防止 GCC 生成会被 objtool 误解的 `break` 指令

### Failure Modes

**Spurious Warnings**：objtool 分类变更后，编译器生成的 trap 指令（如空指针解引用隔离路径上的 `ud2`/`break`）被 objtool 识别为错误分类，触发非预期的警告。

**构建失败**：未在架构 Makefile 中添加对应的编译器标志来抑制不需要的 trap 指令生成。

### Quick Checks

- `tools/objtool/arch/*/decode.c` 中的指令解码器变更可能需要 `arch/*/Makefile` 中对应的编译器标志变更
