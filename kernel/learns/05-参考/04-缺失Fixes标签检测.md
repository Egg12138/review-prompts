<!-- Source: missing-fixes-tag.md -->

# 缺失 Fixes 标签检测

## 用途

当一个补丁修复了之前提交中的 bug 时，应包含 Fixes 标签——即使修复不需要 stable 反向移植。缺失 Fixes 标签会导致以下问题：

- 难以跟踪 bug 起源
- 难以确定 stable 反向移植范围
- 代码审查时难以理解修复上下文
- 难以将修复与原始 bug 关联

## 何时标记缺失的 Fixes 标签

**风险**：丢失归因、不完整的 stable 反向移植、糟糕的 git 考古

## 查找被修复的提交

如果这是一个 bug 修复补丁，搜索 git 历史（使用代码分析工具或 git log），找到被修复的提交。

如果能够识别出被修复的提交，创建一个建议的 Fixes 标签：

```
Fixes: <short SHA> ("<commit subject>")
```

- `<short SHA>` 是 SHA 的前 12 个字符
- `<commit subject>` 是整个主题，用 (" ") 包围

示例：

```
Fixes: 54a4f0239f2e ("KVM: MMU: make kvm_mmu_zap_page() return the number of pages it actually freed")
```

在这种情况下，将缺失 Fixes 标签视为回归问题，确保将其添加到审查报告中。说明被审查的提交如何修复了识别出的提交。

## 如果无法确定被修复的提交

### 进行主观审查时

将缺失的 Fixes 标签视为回归问题，报告如下：

```
This commit appears to fix a bug, but the commit that introduced the bug has
not been identified.  Please consider searching for the commit being fixed.
```

### 不进行主观审查时

不将缺失的 Fixes 标签视为回归问题。

### 输出格式

```
Fixes: tag missing (y/n) [Fixes: line if discovered]
```
