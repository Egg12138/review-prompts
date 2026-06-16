<!-- Source: coccinelle.md -->

# Coccinelle 语义补丁

## 用途

Coccinelle 是一种用于系统化、基于模式的代码转换工具，适用于内核树中的重复性代码变更。当需要跨多个文件执行相同模式的修改时，优先使用 Coccinelle 语义补丁 (SmPL) 而非手动逐一编辑。

## 何时使用 Coccinelle

以下请求模式适合使用 Coccinelle：

| 模式 | 示例 |
|------|------|
| 函数/宏重命名 | 将 `foo()` 重命名为 `bar()` |
| API 签名变更 | 为所有 `func()` 调用添加一个参数 |
| 模式替换 | 用辅助函数替换手写模式 |
| 包裹调用 | 为所有 `X()` 调用加上 lock/unlock |
| 消除样板代码 | 移除 `kfree()` 前冗余的 NULL 检查 |
| 类型变更 | 在所有调用者中将参数类型从 X 改为 Y |
| 添加错误处理 | 在所有 `X()` 调用后添加错误检查 |
| 参数重排 | 交换 `func()` 的第 2 和第 3 个参数 |

## SmPL 快速参考

### 规则结构

```
@ 规则名 @
元变量声明
@@

- 旧代码
+ 新代码
```

### 元变量类型

| 类型 | 匹配 | 示例 |
|------|------|------|
| `expression` | 任意 C 表达式 | `expression E;` |
| `identifier` | 变量/函数名 | `identifier func;` |
| `type` | C 类型 | `type T;` |
| `statement` | 完整语句 | `statement S;` |
| `constant` | 字面常量 | `constant C;` |
| `position` | 源代码位置（用于脚本） | `position p;` |
| `typedef` | 类型别名 | `typedef T;` |
| `declarer` | 声明宏 | `declarer name DEFINE_X;` |

### 关键语法

- `- line` : 删除此行
- `+ line` : 添加此行（跟在 `-` 行后即为替换）
- `...` : 匹配两点之间的任意代码
- `... when != expr` : 匹配不包含 expr 的任意代码
- `... when any` : 即使经过错误路径也匹配
- `<... pattern ...>` : pattern 出现在匹配代码中的某处（仅上下文）
- `<+... pattern ...+>` : pattern 出现在某处，允许修改
- `\(alt1 \| alt2 \)` : 匹配替代模式
- `f(...)` : 匹配带任意参数的函数调用

### 虚拟模式

总是使用 `virtual patch` 模式进行转换：

```
virtual patch

@ depends on patch @
expression E;
@@

- old_func(E)
+ new_func(E)
```

### 标识符正则约束

可以用正则表达式约束标识符：

```
@ rule @
identifier fn =~ "^my_prefix_";
@@

  fn(...)
```

## 重要限制

### Coccinelle 使用 POSIX 正则，而非 PCRE

Coccinelle 的正则引擎**不**支持 Perl/PCRE 的缩写语法。使用不支持的语法会导致解析时出现 `lexical error: unrecognised symbol`。

| 不要用 (PCRE) | 改用 (POSIX) |
|---------------|-------------|
| `\w` | `[a-zA-Z0-9_]` |
| `\d` | `[0-9]` |
| `\s` | `[ \t\n]` |
| `\W`, `\D`, `\S` | 否定对应的 POSIX 类 |

**错误：** `identifier fn =~ "^trace_\w+_enabled$";`

**正确：** `identifier fn =~ "^trace_[a-zA-Z0-9_]+_enabled$";`

### `...`（省略号）不能出现在 `+` 上下文中

`...` 元变量意为"匹配任意代码序列"。它只在上下文（未修改）或 `-`（删除）行中有效。将 `...` 放在 `+` 行会导致：`lexical error: invalid in a + context: ...`

**错误：**
```
  if (!enabled_fn())
      return
-         ...;
+         ...;
```

**正确**（将 `return ...;` 作为上下文——只修改实际变化的部分）：
```
  if (!enabled_fn())
      return ...;
  ...
- call_fn(ES)
+ new_fn(ES)
```

### `##` 不能用于匹配

SmPL 的 `##` 操作符**只**用于在替换端创建全新的标识符。它**不能**用于匹配相关的标识符。

**错误——这种写法无效：**
```
@ rule @
identifier name;
@@

- trace_##name##_enabled()
```

当你需要匹配一族相关的名称时（例如匹配 `trace_FOO_enabled()` 和对应的 `trace_FOO()`，其中 FOO 相同），必须使用 Python 脚本规则来派生相关名称。

### 不要声明未使用的元变量

规则中声明的每个元变量都**必须**出现在 `-` 或上下文字代码中。未使用的声明会产生警告，表明规则逻辑错误。移除所有未被引用的声明。

## Python 脚本规则（处理相关名称族）

当转换涉及相关的标识符族（共享公共子串的名称）时，使用三步模式：

### 步骤 1：匹配规则
使用正则约束捕获锚点标识符：

```
@r@
identifier anchor_fn =~ "^prefix_[a-zA-Z0-9_]+_suffix$";
position p;
@@

anchor_fn@p(...)
```

### 步骤 2：脚本规则
通过 Python 派生相关标识符：

```
@script:python s@
anchor_fn << r.anchor_fn;
related_fn;
replacement_fn;
@@

import re
m = re.match(r'^prefix_(.+)_suffix$', anchor_fn)
coccinelle.related_fn = "other_prefix_%s" % m.group(1)
coccinelle.replacement_fn = "new_prefix_%s" % m.group(1)
```

### 步骤 3：转换规则
使用捕获和派生的标识符进行转换：

```
@ depends on patch @
identifier r.anchor_fn;
identifier s.related_fn;
identifier s.replacement_fn;
expression list ES;
@@

  if (anchor_fn())
-     related_fn(ES);
+     replacement_fn(ES);
```

脚本生成的标识符在后续规则中既可用于匹配也可用于替换。

## 常见模式

### 简单函数重命名

```
virtual patch

@ depends on patch @
expression list ES;
@@

- old_name(ES)
+ new_name(ES)
```

### 添加参数

```
virtual patch

@ depends on patch @
expression E1, E2;
@@

- func(E1, E2)
+ func(E1, E2, NEW_DEFAULT)
```

### 移除参数

```
virtual patch

@ depends on patch @
expression E1, E2, E3;
@@

- func(E1, E2, E3)
+ func(E1, E3)
```

### 用辅助函数替换手写模式

```
virtual patch

@ depends on patch @
expression a, b;
identifier tmp;
type T;
@@

- T tmp;
  ...
- tmp = a;
- a = b;
- b = tmp;
+ swap(a, b);
```

### 移除冗余 NULL 检查

```
virtual patch

@ depends on patch @
expression E;
@@

- if (E)
-   kfree(E);
+ kfree(E);
```

### 为调用加锁

```
virtual patch

@ depends on patch @
expression E, lock;
@@

+ spin_lock(&lock);
  func(E);
+ spin_unlock(&lock);
```

### 多规则：先找结构体，再转换调用者

```
virtual patch

@ r @
identifier fn;
type T;
@@

  T fn(...) { ... }

@ depends on patch && r @
expression E;
@@

- old_api(E)
+ new_api(E, 0)
```

## 守卫调用点模式

当转换受启用/功能检查守卫的调用时，必须处理**所有**以下 `if` 守卫变体。遗漏任何一种都会静默丢失调用点。

### 1. 简单守卫（无花括号）

```
  if (enabled_fn())
-     call_fn(ES);
+     new_fn(ES);
```

### 2. 带额外条件的守卫（无花括号）

```
  if (enabled_fn() && COND)
-     call_fn(ES);
+     new_fn(ES);
```

### 3. 带花括号的块（可能有设置代码）

使用 `<+... ...+>` 在任意嵌套深度匹配调用（例如守卫内的循环、条件或其它块中的调用）。普通的 `...` 只匹配同一块级别，会遗漏嵌套循环中的调用。

```
  if (enabled_fn()) {
    <+...
-   call_fn(ES)
+   new_fn(ES)
    ...+>
  }
```

### 4. 带额外条件的花括号块

```
  if (enabled_fn() && COND) {
    <+...
-   call_fn(ES)
+   new_fn(ES)
    ...+>
  }
```

### 5. 否定提前返回（直接）

```
  if (!enabled_fn())
      return ...;
  ...
- call_fn(ES)
+ new_fn(ES)
```

### 5b. 否定提前返回（嵌套在循环中）

```
  if (!enabled_fn())
      return ...;
  ... when any
  {
    <+...
-   call_fn(ES)
+   new_fn(ES)
    ...+>
  }
```

### 6. `unlikely()` 包装（无花括号）

```
  if (unlikely(enabled_fn()))
-     call_fn(ES);
+     new_fn(ES);
```

### 7. `unlikely()` 包装（花括号块）

```
  if (unlikely(enabled_fn())) {
    <+...
-   call_fn(ES)
+   new_fn(ES)
    ...+>
  }
```

为**每个**变体编写**独立**的 SmPL 规则。不要尝试将它们合并为一条规则——Coccinelle 基于结构匹配，每种 `if` 形式都是不同的 AST 形状。

## 执行流程

生成 .cocci 文件后，自动执行完整流水线：

### 1. 编写 .cocci 文件

写入当前工作目录，使用描述性名称（如 `rename_foo_to_bar.cocci`）。

### 2. 测试解析错误

```
make coccicheck COCCI=./script.cocci MODE=patch 2>&1 | head -20
```

常见错误：
- `unrecognised symbol:\w` → 使用 `[a-zA-Z0-9_]` (POSIX 正则)
- `invalid in a + context: ...` → `...` 不能出现在 `+` 行
- `metavariable X not used` → 移除未使用的声明

### 3. 捕获完整补丁

```bash
make coccicheck COCCI=./script.cocci MODE=patch 2>/dev/null > /tmp/full.patch
grep '^diff -u' /tmp/full.patch | sed 's|diff -u -p a/||; s| b/.*||' | sort
```

### 4. 生成并运行按子系统应用的脚本

生成 shell 脚本 `scripts/<name>_apply.sh`，将 coccicheck 输出拆分为按子系统的提交。

### 5. 显示最终提交日志

让用户可以审查系列提交。

## 按子系统应用脚本

关键实现细节：

- **文件到子系统的映射** — 使用 case 语句，最具体的路径放在前面
- **基于文件的过滤** — 按精确的文件成员资格过滤，而非目录前缀
- **暂存特定文件** — 使用 `git add <file>`，绝不使用 `git add -A`

### 提交消息格式

```
<subsystem>: <short description>

<Explanation of what the transformation does and why.>

Generated with:
  make coccicheck COCCI=./script.cocci MODE=patch

Coccinelle SmPL rule: ./script.cocci
```

## 编写指南

- 保持规则最小化。不加 `context`、`org` 或 `report` 虚拟模式，除非被要求
- 不关心具体参数时，使用 `expression list ES;` 加 `f(ES)` 匹配所有参数
- 需要引用具体参数时，使用 `expression E1, E2;`
- 名称必须逐字匹配时使用 `identifier`（结构体字段名、声明中的函数名）
- 类型本身会变化且需要保留时使用 `type T;`
- 谨慎使用 `...`（省略号）——它会使匹配变得非常宽泛
- `...` 只在上下文或 `-` 行中有效，绝不在 `+` 行中
- 多规则优于复杂的单规则
- 为每种结构性的 `if` 变体编写独立的规则
- 在正则中使用 POSIX 字符类（`[a-zA-Z0-9_]`），绝不使用 PCRE（`\w`）
- 在复杂模式中，先用 `MODE=report` 或 `MODE=context` 测试，再用 `MODE=patch`
- 参考 `scripts/coccinelle/` 中的现有脚本获取惯用法示例
- 绝不使用 `##` 进行匹配——它只适用于创建全新的标识符。使用 Python 脚本规则派生相关标识符
