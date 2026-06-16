<!-- Source: scripts/lore-reply -->

# Lore 邮件回复

`lore-reply` 是一个 Python 脚本，用于创建格式正确的回复邮件，回复发布在 lore.kernel.org 上的补丁。它支持可选的 AI 辅助线程回复分析和补丁验证。

## 使用方法

### 基础用法

```bash
# 通过提交引用回复补丁（使用 b4 dig 在 lore 上查找）
lore-reply [--dry-run] [--force] <COMMIT-REF>

# 从本地 mbox 文件回复补丁
lore-reply [--dry-run] --mbox <MBOX-FILE> <COMMIT-REF>
```

### 选项

| 选项 | 说明 | 默认 |
|------|------|------|
| `--dry-run` | 不实际发送邮件 | 发送邮件 |
| `--force` | 跳过补丁 ID 验证和 AI 分析 | - |
| `--b4` | 使用 b4 dig 替代 semcode dig | - |
| `--mbox <file>` | 使用现有的 mbox 文件代替下载 | - |

## 工作流程

### 提交引用模式

1. 使用 dig 命令（semcode 或 b4）通过提交哈希在 lore.kernel.org 上查找补丁
2. 下载线程 mbox
3. 使用 AI 总结线程中的现有回复
4. 验证邮件和本地提交的补丁 ID 是否匹配
5. 如果补丁 ID 不同，使用 AI 解释差异
6. 创建带有正确头部的回复邮件（In-Reply-To、References）并引用正文
7. 打开 `git send-email --annotate` 供编辑和发送

### Mbox 模式

1. 直接读取指定的 mbox 文件
2. 跳过所有验证和分析
3. 创建回复邮件并打开 git send-email

## 回复分析

在不带 `--force` 运行时，脚本会检查 `./review-inline.txt` 并要求 AI：

- 总结对补丁的现有回复
- 检查是否有人报告了与审查文件中描述的类似问题

## 补丁 ID 验证

脚本使用 `git patch-id --stable` 计算补丁 ID，以验证 lore 邮件与本地提交是否匹配。如果不匹配：

- 如果提供了 `--force`，则跳过验证
- 否则，要求 AI 解释差异并询问用户是否继续

## 完整工作流示例

```bash
# 在审查提交后：
cd linux.<sha>

# 回复补丁
lore-reply HEAD

# 测试但不发送（dry-run）
lore-reply --dry-run HEAD

# 跳过 AI 分析和补丁验证
lore-reply --force HEAD

# 回复手动下载的 mbox
lore-reply --mbox thread.mbox
```

## 前置条件

- `b4` — 用于从 lore.kernel.org 查找和下载补丁
- `git send-email` — 用于发送回复
- `claude` CLI（可选）— 用于线程分析和补丁比较

## 脚本内部工作方式

### 查找提交（dig_commit 函数）

1. 先尝试 semcode dig（除非指定了 `--b4`）
2. 如果 semcode 失败，尝试 b4 dig
3. 如果 b4 也失败，使用 AI 进行 lore 搜索作为最后的回退方案

### 邮件查找（find_message_by_subject 函数）

- 解析 mbox 文件，按 "From " 行分割消息
- 检查主题是否包含提交主题
- 优先选择原始补丁邮件而非封面信或回复

### 回复文件生成（make_reply_file 函数）

- 生成正确的 In-Reply-To 和 References 头部
- 保留原始收件人和抄送列表
- 引用原始邮件正文（每行加 `> ` 前缀）
- 生成后通过 `git send-email --annotate` 发送
- 编辑后的副本保存为 `./review-email.txt`

### 主题判断辅助函数

- `is_cover_letter()` — 检查主题是否为封面信（`[PATCH 0/N]` 或 `[PATCH vX 0/N]`）
- `is_reply()` — 检查主题是否为回复（以 "Re:" 开头）
- `is_original_patch()` — 检查是否为原始补丁（非回复、非封面信）
