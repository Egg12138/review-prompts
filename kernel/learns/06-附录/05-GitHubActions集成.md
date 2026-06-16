<!-- Source: docs/github-actions-claude-integration.md -->

# GitHub Actions 集成

## 概述

Claude Code 官方支持通过 GitHub 工作流触发代码审查。本文档说明了如何将内核审查提示嵌入到 [claude-code-action](https://github.com/anthropics/claude-code-action) 中。

## 配置步骤

以下步骤改编自 [claude-code-action 设置文档](https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md)，并添加了更详细的说明。

### 步骤 1：安装 Claude App

访问 [Marketplace](https://github.com/apps/claude) 为你的仓库安装 Claude App。这个仓库可以是新创建的，也可以是从其他来源克隆的。

### 步骤 2：配置 GitHub App

在你的 Claude 交互式命令行中执行 `/install-github-app`。这将一步步引导你完成应用配置过程。最终，它会为你的项目生成并保存一个 TOKEN，并在 `.github/workflows/claude.yml` 中创建一个基本工作流。

你也可以按照 `claude-code-action` 设置文档中的描述手动完成上述步骤。

### 步骤 3：编辑 GitHub Actions 配置

编辑仓库中的 `.github/workflows/claude.yml`。以下 YAML 配置与官方的 [claude-code-action 示例](https://github.com/anthropics/claude-code-action/blob/main/examples/claude.yml)略有不同。关键新增内容包括"Checkout prompts repo"步骤和自定义审查提示。

```yaml
name: Claude Code

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
  issues:
    types: [opened, assigned]
  pull_request_review:
    types: [submitted]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review_comment' && contains(github.event.comment.body, '@claude')) ||
      (github.event_name == 'pull_request_review' && contains(github.event.review.body, '@claude')) ||
      (github.event_name == 'issues' && (contains(github.event.issue.body, '@claude') || contains(github.event.issue.title, '@claude')))
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
      issues: write
      id-token: write
      actions: read # Required for Claude to read CI results on PRs
    steps:
      - name: Checkout repository
        uses: actions/checkout@v5
        with:
          fetch-depth: 1
      - name: Checkout prompts repo
        uses: actions/checkout@v5
        with:
          repository: 'masoncl/review-prompts'
          path: 'review'
      - name: Run Claude Code
        id: claude
        uses: anthropics/claude-code-action@v1
        with:
          claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
          track_progress: true
          prompt: |
            Current directory is the root of a Linux Kernel git repository.
            Using the prompt `review/review-core.md` and the prompt directory `review`
            do a code review.
```

### 配置要点说明

| 元素 | 说明 |
|------|------|
| `Checkout prompts repo` | 额外步骤，检出审查提示仓库到 "review" 目录 |
| `track_progress: true` | 保留 claude-code-action 相关的 MCP 服务 |
| `prompt` | 自定义提示内容，指定使用 review 目录中的审查核心提示 |
| 触发条件 | 当 PR 评论、issue 评论或审查中包含 `@claude` 时触发 |

## 使用方式

完成上述配置后，在创建 PR 后，可以通过在 PR 评论中提及 `@claude` 来触发代码审查。你也可以根据需要修改触发条件。

审查结果将以评论形式发布到你的 PR 中。

### 触发条件说明

配置中的触发条件包括：

- **issue_comment** — 当有人在 Issue 或 PR 中创建评论，且评论内容包含 `@claude` 时触发
- **pull_request_review_comment** — 当有人在 PR 审查中创建行级评论，且内容包含 `@claude` 时触发
- **issues** — 当 Issue 被创建或分配，且 Issue 正文或标题包含 `@claude` 时触发
- **pull_request_review** — 当 PR 审查被提交，且审查正文包含 `@claude` 时触发
