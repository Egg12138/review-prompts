<!-- Source: lore-thread.md -->

# Lore 线索分析

## 用途

内核补丁在 lore.kernel.org 上讨论。开发者经常在不同的版本之间遗漏了需要处理的审查意见。本文档说明了如何有效处理 lore 线索，以识别未处理的审查反馈。

这些分析输出将供维护者判断补丁是否可以合入。确保补丁上的所有邮件评论都得到了处理，尤其是来自维护者的评论，这一点非常重要。

## 自动化审查隔离规则 (Automated Review Quarantine)

自动化审查和机器人邮件**不是**本指南意义上的审查评论。绝不引用、引用、总结或报告来自以下来源的问题：Sashiko、先前的 BPF CI/Claude 审查、CI 机器人、测试机器人或任何看起来是自动化的发送者。

### 隔离条件

如果消息的发送者、主题、正文、链接或签名包含以下信号，则应隔离：

- `bot`、`robot`
- `sashiko`、`claude`、`bpf-ci`
- `AI review found`、`AI reviewed your patch`
- `CI run summary`、`sashiko.dev`、`netdev-ai.bots.linux.dev`

### 人类回复的处理

仅转发、引用或询问自动化审查反馈，但**没有添加独立的人类技术分析**的人类回复也需要隔离。

如果人类回复同时包含机器人引用文本和独立的人类分析，则忽略机器人引用部分，只处理独立的人类分析。

### 重叠问题

如果某个潜在问题与已隔离的机器人反馈重叠，则**完全压制该问题**。不要将机器人证据改写为中性措辞。

## 步骤 1：查找所有版本

使用 dig 命令查找与提交相关的邮件。从结果中识别：

- 来自作者的补丁提交（不同版本：v1、v2、v3 等）
- 来自维护者/审查者的人类回复
- 作者对审查的回应
- 已隔离的自动化审查评论（仅跟踪以便压制重叠问题）

## 步骤 2：高效处理大线索

Lore 线索可能非常庞大。**不要**使用完整线程获取。

### 正确方法

1. **列出人类审查者回复的 Message-ID**
   - 在 dig 结果中查找来自非补丁作者的 "Re:" 邮件
   - 排除所有已隔离的自动化审查评论
   - 记录每条审查评论的 Message-ID

2. **单独获取每条审查邮件**（不带线程上下文）：
   ```
   lore_search(message_id="<id>", verbose=true)
   ```

3. **分别获取作者回复**：
   ```
   lore_search(message_id="<author-reply-id>", verbose=true)
   ```

4. **比较审查评论与作者回复**，判断是否已处理。

### 如果输出仍然太大

使用 jq 和 grep 进行定向提取：

```bash
# 从 JSON 输出中提取特定邮件
jq -r '.[] | .text' file.txt | awk '/Message-ID: <specific-id>/,/--- End Message ---/'

# 查找审查模式（引用代码 + 审查者评论）
jq -r '.[] | .text' file.txt | grep -B2 -A10 "^   >"
```

## 步骤 3：识别审查评论

审查评论通常具有以下特征：

- 引用的补丁代码（以 `> +` 或 `> -` 或 `>  ` 开头的行）
- 后跟审查者文本（不以 `>` 开头）
- 关键词："nit:"、"please"、"should"、"instead"、"why"、"consider"、"missing"

在将任何内容归类为审查评论前，先应用自动化审查隔离规则。机器人评论即使包含具体的技术问题，也不是审查评论。

## 步骤 4：跟踪评论解决状态

对找到的每条审查评论，记录：

| 评论 | 在回复中处理了？ | 在下一版本中处理了？ | 状态 |
|------|----------------|-------------------|------|
| ... | 是/否 | 是/否 | 已解决/未处理 |

## 步骤 5：对照当前代码验证

对作者标记为"will fix"或"Ack"的评论：

1. 检查修复是否实际出现在当前的 HEAD 提交中。如果未找到，检查系列中的其他补丁——作者有时会通过修改不同的补丁来处理反馈
2. **严格验证修复是否正确**，尝试与任何评论或提交消息持反对意见
3. 在 MAINTAINERS 文件中搜索审查者。如果存在于 MAINTAINERS 中，则将任何未处理的请求或部分处理的评论视为回归问题，即使它只是一个风格建议或不修复 bug。通过吹毛求疵的分析视角来处理
4. 在 MAINTAINERS 文件中搜索作者。如果存在，则更相信他们的专业判断

### 输出格式

```
comment <comment>
Addressed <yes/no>
Expected response from original reviewer: <fix is sufficient / fix is not sufficient>
```

如果认为原审查者不会认为修复足够，则将其视为潜在的回归问题。

作者的同意并不意味着修复已实现。

## 步骤 6：最终验证

```
Found older version: <date> <version> <subject>
Found older version: <date> <version2> <subject>

FINAL UNADDRESSED COMMENTS: NUMBER
Original reviewer expected responses to new patch
```

如果存在未处理的评论，在审查报告中包含 lore 链接：
```
https://lore.kernel.org/bpf/<message-id>/
```

### 自查清单

- 是否按要求分析了每个先前的审查评论？[是/否]
- 是否排除了机器人和自动化审查评论，并压制了与其重叠的问题？[是/否]
- 如果有未处理的条目，必须返回并检查它们
