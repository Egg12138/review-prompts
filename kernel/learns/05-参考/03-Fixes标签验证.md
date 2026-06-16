<!-- Source: fixes-tag.md -->

# Fixes 标签验证

## Fixes 标签的用途

Fixes 标签表示一个补丁修复了之前某个提交中的 bug。该标签：

- 便于确定问题的起源
- 帮助审查者理解 bug 修复的上下文
- 协助 stable 内核团队确定哪些 stable 版本应接收修复
- 被自动化反向移植工具使用
- 即使 bug 不需要 stable 反向移植也应包含

## 格式要求 [FIXES-001]

**风险**：解析失败、错误的 stable 反向移植

### 1. SHA-1 长度检查

- SHA-1 至少需要 12 个字符
- 仅限十六进制字符
- 示例：`c0cbe70742f4`（12 字符）正确
- 反例：`c0cbe70`（7 字符）错误

### 2. 摘要格式检查

- 主题行必须用双引号括起来
- 主题行应与原始提交的第一行匹配
- 格式：`Fixes: 12+char-SHA1 ("Original subject line")`
- 示例：`Fixes: 54a4f0239f2e ("KVM: MMU: make kvm_mmu_zap_page() return the number of pages it actually freed")` 正确

### 3. 单行要求

- 标签**不能**跨多行
- 标签不受"75 列换行"规则的约束，以简化解析脚本
- 即使行非常长，也应保持在一行上

**错误示例**（标签必须在一行上）：
```
Fixes: 54a4f0239f2e ("KVM: MMU: make kvm_mmu_zap_page()
  return the number of pages it actually freed")
```

### 4. 主题行准确性

- 使用 `git log -1 --format=%s <commit-id>` 获取原始主题
- 与 Fixes 标签中的主题进行比较
- 常见错误：
  - 主题行被截断
  - 主题被修改或改写
  - 缺少子系统前缀

## 标签位置 [FIXES-002]

**风险**：标签不被自动化工具识别

### 1. 在提交消息中的位置

- 标签应出现在签名区（主要提交描述之后）
- 典型顺序（来自 `maintainer-tip.rst`）：
  ```
  <commit description>

  Fixes: <sha1> ("subject")
  Reported-by: <reporter>
  Signed-off-by: <author>
  Reviewed-by: <reviewer>
  ```

### 2. 不在注释区

- 标签必须在 `---` 分隔符**之上**
- `---` 之下的标签不会被包含在 git 提交中

## 提交验证 [FIXES-003]

**风险**：无效的提交引用、错误的归因

### 1. 提交存在性

- 运行：`git cat-file -t <commit-id>`
- 验证其返回 "commit"
- 如果提交在当前树中不存在，检查它是否在 Linus 的树中

### 2. 提交可达性

- 运行：`git merge-base --is-ancestor <commit-id> HEAD`
- 验证提交在主线上游历史中
- 注意：对于近期提交的修复，可能在 linux-next 或子系统树中

### 3. 验证 bug 实际存在

- 使用 `git show` 或 `git log` 读取被引用的提交
- 分析当前补丁是否确实修复了该提交引入的 bug
- 需要检查的常见错误：
  - Fixes 标签指向了错误的提交
  - Fixes 标签指向一个没有引入该 bug 的提交
  - 多个提交共同导致了 bug，但只引用了一个

## Stable 内核注意事项 [FIXES-004]

**风险**：缺少 stable 反向移植、反向移植范围不正确

### 1. Fixes 标签不保证反向移植

- Fixes 标签本身并不会在所有子系统中自动触发 stable 反向移植
- 验证是否同时存在 `Cc: stable@vger.kernel.org` 标签
- 某些子系统选择不自动基于 Fixes 标签进行反向移植

### 2. Stable 标签验证

- 分析 bug 是否影响已发布的内核
- 对于过去 12 个月内的回归问题，应存在 stable 标签
- 验证 stable 标签在签名区（不是作为邮件的 Cc 收件人）

### 3. 反向移植前置条件

- 检查修复是否依赖于其他提交
- 如果有依赖，应在标签中注明：
  ```
  Cc: <stable@vger.kernel.org> # 5.10.x: abc123: dependency description
  Cc: <stable@vger.kernel.org> # 5.10.x
  ```

## 常见模式与边界情况

### 应存在 Fixes 标签的情况

1. **Bug 修复**
   - 修复崩溃、挂起、数据损坏、安全问题
   - 修复特定提交引入的不正确行为
   - 即使 bug 不需要 stable 反向移植

2. **回归问题**
   - 任何用户可见的回归问题都应带有 Fixes 标签
   - 性能回归、功能回归

### 可缺少 Fixes 标签的情况

1. **没有特定 bug 的改进**
   - 一般性优化
   - 代码重构（不修复 bug）
   - 新功能

2. **对非常旧代码的修复**
   - bug 自 git 初始历史就存在
   - 替代方案：在提交消息中注明 "bug existed since ..."

3. **多个贡献提交**
   - 如果有多个提交共同导致了 bug，通常引用最直接/最近的原因
   - 必要时可以包含多个 Fixes 标签（罕见）

## 审查者的 Git 配置

为方便生成 Fixes 标签，配置 git：

```
[core]
    abbrev = 12
[pretty]
    fixes = Fixes: %h (\"%s\")
```

使用方式：`git log -1 --pretty=fixes <commit-id>`

## 快速参考

**正确格式：**
```
Fixes: 54a4f0239f2e ("KVM: MMU: make kvm_mmu_zap_page() return the number of pages it actually freed")
```

**常见错误：**
- 太短：`Fixes: 54a4f02 (...)` 错误
- 缺少引号：`Fixes: 54a4f0239f2e (KVM: MMU: ...)` 错误
- 换行：`Fixes: 54a4f0239f2e ("KVM:\n    MMU: ...")` 错误
- 位置错误：标签出现在 `---` 分隔符下方 错误
- 发布的 bug 缺少 stable 标签 警告
