<!-- Source: subsystem/perf.md -->

# Perf 工具子系统详解

## struct perf_tool 回调接口

`struct perf_tool` 中的事件回调如果被遗漏，对应的事件会被静默丢弃。在 pipe mode（管道模式）下，丢弃 `perf_event_header_attr` 事件会导致 evlist 和 evsel 无法创建，进而完全破坏事件处理流程。

### Invariants

- 未注册的事件类型被静默忽略（silently ignored）
- 任何注册了 `.mmap` 回调的工具**必须同时**注册 `.mmap2` 回调（反之亦然）
- 在 pipe mode 下，工具必须正确注册 attribute 和 feature 回调，以填充 evsels 和 `struct perf_env`

### Failure Modes

**Pipe Mode 数据丢失**：如果在 pipe mode 中未注册 attribute 回调，`perf_event_header_attr` 事件被丢弃，导致 evlist 无法正确构造，后续所有事件处理失败。

**Mmap 回调不配对**：注册 `.mmap` 但不注册 `.mmap2`（或反过来），导致部分内存映射事件未经处理即被丢弃，分析结果不完整。

### Quick Checks

- 验证子命令是否注册了完整的事件回调（配对 `.mmap`/`.mmap2`，在 pipe mode 下处理 `.attr`）

---

## 构建特性检测与条件编译

特性检测标志（feature detection flags）不一致会导致构建失败，或在缺少可选库时静默丢失功能。特性检测通过后，`Makefile.config` 会定义 `-DHAVE_*_SUPPORT` 等编译标志；遗漏这些定义或未提供头文件后备桩（fallback stubs），会导致在不包含该库的系统上编译失败。

### Invariants

- 特性检测通过 `tools/build/feature/test-*.c` 在构建时验证可选库的可用性
- `tools/perf/Makefile.config` 评估检测结果，设置编译器标志（如 `CFLAGS += -DHAVE_LIBELF_SUPPORT` 或 `CONFIG_*` 定义）
- C 代码必须使用 `#ifdef HAVE_*_SUPPORT` 或 Build/Makefile 中的 `CONFIG_*` 值来防护特性相关逻辑
- 头文件必须在特性定义缺失时提供兼容的哑内联桩（dummy inline stubs），例如返回 `-ENOTSUPP` 或 `NULL`
- 添加或修改特性时，`Makefile.config`、特性 makefile 和头文件防护必须严格保持同步

### Failure Modes

**构建失败**：特性检测通过但未定义对应的 `HAVE_*_SUPPORT` 宏，导致依赖该宏的代码无法编译。

**静默功能缺失**：头文件缺少 fallback stub，在不支持该特性的系统上编译失败。

**Makefile 不同步**：在 `Makefile.config` 中添加了新特性检测，但未在对应的 Build 文件中更新条件编译规则。

### Quick Checks

- 验证可选特性逻辑是否正确使用 `HAVE_*_SUPPORT` 或 `CONFIG_*` 定义防护，并伴随头文件 fallback stubs

---

## perf.data 头部校验

`perf.data` 文件可以是普通文件，也可以来自管道（pipe）。在 pipe mode 下访问事件时，流不支持 seek 操作。普通文件包含 attributes 和 features 的段；在 pipe mode 下，这些必须以合成事件（synthesized events）的方式处理。新版本 perf 工具写入的新 feature，旧版本 perf 工具无法识别；反之，旧版本生成的 `perf.data` 也不包含新 feature。加载后的 features 存放在 `struct perf_env` 中，通常由 `perf_session__new()` 填充，但在 pipe mode 下需要处理事件来填充 `perf_env`。在 live mode（如 `perf top`）中，host 的 `perf_env` 需要显式创建。访问 `perf_env` 字段前未验证其是否已初始化，是一个 bug。

### Invariants

- `perf.data` 的访问方式（普通文件 vs 管道）决定了能否使用 seek
- `struct perf_env` 的字段在访问前必须确认已被正确初始化
- pipe mode 下，attributes 和 features 需作为合成事件处理

### Failure Modes

**未初始化的 perf_env 访问**：在 pipe mode 下未处理事件就访问 `perf_env` 字段，导致读取到未初始化数据或崩溃。

**旧工具兼容性**：新写入的 feature 在不认识的旧工具上被静默忽略。

### Quick Checks

- 验证 `perf_env` 字段在访问前检查了初始化状态

---

## 架构特定代码与跨平台分析

将分析或解码逻辑放在 `tools/perf/arch/` 下，会将该功能限制在仅为该架构编译的 host 二进制文件中。这会破坏跨平台分析能力——例如，x86 上的 perf 二进制无法检查或报告在 ARM 或 RISC-V 上录制的 `perf.data` 文件。

### Invariants

- `tools/perf/arch/` 目录**只能**包含与 host 执行严格相关的代码（如原生 PMU 探测或硬件寄存器操作）
- 不鼓励向 `tools/perf/arch/` 添加新逻辑；优先采用跨平台实现
- 处理录制或分析时的架构差异，应动态检查 ELF 机器常量 `e_machine`，该值可通过 `struct perf_env`、session、machine、thread 或 evsel 结构获取

### Failure Modes

**跨平台分析失败**：架构特定逻辑放置在 `tools/perf/arch/` 下，导致跨架构的 `perf.data` 文件无法正确解析。

### Quick Checks

- 验证架构特定逻辑动态查询 `e_machine`，而不是依赖 `tools/perf/arch/` 硬编码的 host 二进制

---

## 引用计数检查与指针句柄

引用计数（reference count）不平衡会导致内存泄漏或 use-after-free 缺陷。当启用 `REFCNT_CHECKING`（由 ASAN/LSAN 启用）构建时，perf 将引用计数结构体（如 `thread`、`maps`、`dso`）包装为中间指针句柄（`DECLARE_RC_STRUCT`）。在调用 `_put()` 后访问句柄会立即触发 ASAN 的 heap-use-after-free 陷阱；遗漏 `_put()` 调用则会在精确的 `_get()` 调用位置触发 LSAN 泄漏报告。

### Invariants

- 每个通过 `_get()`（如 `thread__get()`、`maps__get()`）获取或通过 `_new()` 分配的引用句柄，必须严格配对一次 `_put()` 调用（如 `thread__put()`）
- 结构体传给 `_put()` 后，其指针句柄即失效并被释放，**绝不能在 `_put()` 后访问结构体字段**
- 避免对引用计数结构体使用裸指针赋值；使用显式的 `_get()` 和 `_put()` 生命周期辅助函数

### Failure Modes

**Use-After-Free**：`_put()` 后继续访问结构体字段，触发 ASAN 检测到的 heap-use-after-free。

**内存泄漏**：`_get()` 或 `_new()` 后缺少对应的 `_put()`，导致 LSAN 报告泄漏。

### Quick Checks

- 验证每个 `_new` 和 `_get` 指针句柄在作用域结束前是否配对了对应的 `_put`

---

## POSIX libc 头文件包含与 musl 兼容性

perf 工具在 glibc 和 musl 两种 C 库下编译。glibc 存在命名空间污染（通过其他头文件隐式包含）的问题，而 musl 严格分离声明。在 glibc 下能编译的代码在 musl 下可能因缺少显式的头文件包含而编译失败。为确保 musl 构建兼容性，所有使用了 libc 函数、变量或常量的文件，必须直接包含这些符号在 POSIX 标准中声明的头文件。

### Invariants

- 不依赖头文件隐式包含其他头文件
- 始终显式包含 POSIX 规定的头文件（如 `read`/`write`/`close` 需要 `<unistd.h>`，`printf`/`fopen` 需要 `<stdio.h>`，`malloc`/`free` 需要 `<stdlib.h>`，`strcmp`/`strlen` 需要 `<string.h>`，`PATH_MAX` 需要 `<limits.h>`）
- 头文件中优先使用前置声明（forward declarations，如 `struct evlist;`），而不是完整包含头文件——但仅限引用了结构体指针句柄的情况
- **不要**手动前置声明标准 libc 函数或类型；这些必须通过相应的 POSIX 标准头文件包含
- 验证所有需要的系统和 POSIX 标准头文件已显式放置在文件顶部

### Failure Modes

**musl 编译失败**：在 glibc 下正常编译的代码，迁移到 musl 时因缺少显式头文件包含而失败。例如使用了 `strcmp()` 但未包含 `<string.h>`，glibc 可能通过其他头文件隐式引入了它，但 musl 不会。

### Quick Checks

- 验证所有 POSIX libc 函数、常量和变量都有显式的、直接的头文件包含（如 `<unistd.h>`、`<string.h>`），以防止 musl 编译失败
- 鼓励在头文件中尽可能对内部结构体使用前置声明，避免沉重的头文件包含

---

## 错误处理与 ERR_PTR 规避

在用户空间的 perf 工具中使用 `ERR_PTR` 是强烈不推荐的。`ERR_PTR` 在内核空间很常见，但在用户空间 perf 代码中使用时，经常导致 bug：`ERR_PTR` 值被错误地与 `NULL` 比较，而不是用 `IS_ERR()` 检查。

### Invariants

- **新代码中避免使用 `ERR_PTR`**：优先使用标准的用户空间范式。返回指针的函数应在失败时返回 `NULL`
- **传播错误码**：如果需要向调用者传递具体的错误码：
  - 返回 `int`（负值的 POSIX errno，如 `-ENOMEM`），通过双指针参数（如 `struct foo **out`）回传分配的对象
  - 或者设置 `errno` 并返回 `NULL`
- **审计现有 `ERR_PTR` 使用**：如果必须使用 `ERR_PTR`（例如与返回此类值的遗留 API 交互），验证所有调用者使用 `IS_ERR()` 和 `PTR_ERR()` 而不是 `NULL` 检查

### Failure Modes

**NULL 检查遗漏**：函数返回 `ERR_PTR` 编码的错误指针，调用者却使用 `if (!ptr)` 而非 `if (IS_ERR(ptr))` 检查，导致错误码被误判为有效指针。

### Quick Checks

- 验证新代码中没有使用 `ERR_PTR`；对于现有使用，确保返回的指针用 `IS_ERR()` 检查而不是 `NULL` 比较

---

## Quick Checks 汇总

- **回调错误路径**：当函数接受回调并遍历目录时，验证回调错误在返回前触发完整清理
- **嵌套 `openat`/`fdopendir`**：当遍历嵌套目录时（如 `/proc/pid/fd` 然后 `/proc/pid/fdinfo`），分别跟踪每个资源并验证清理顺序
- **工具 API 回调**：验证子命令注册完整的事件回调（配对 `.mmap`/`.mmap2`，pipe mode 下处理 `.attr`）
- **特性检测防护**：验证可选特性逻辑是否正确使用 `HAVE_*_SUPPORT` 或 `CONFIG_*` 定义防护并伴随头文件 fallback stubs
- **`perf_env` 验证**：验证 `perf_env` 字段在访问前检查了初始化状态
- **跨平台分析**：验证架构特定逻辑动态查询 `e_machine` 而不是依赖硬编码的 `tools/perf/arch/` host 二进制
- **引用计数平衡**：验证每个 `_new` 和 `_get` 指针句柄在作用域结束前配对了对应的 `_put`
- **musl 兼容性**：验证所有 POSIX libc 函数、常量和变量有显式的直接头文件包含
- **`ERR_PTR` 使用**：验证新代码中没有使用 `ERR_PTR`；现有使用确保用 `IS_ERR()` 检查
