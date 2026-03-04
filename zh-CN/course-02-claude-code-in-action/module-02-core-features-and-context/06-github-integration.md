# 课时 9：GitHub 集成

[← 上一课：MCP 服务器扩展](./05-mcp-servers.md) | [返回目录](../README.md) | [下一课：计划模式与思考模式 →](../module-03-advanced-workflows/01-plan-mode-and-thinking-mode.md)

---

## 核心概念

Claude Code 与 GitHub 深度集成，不仅可以在本地操作 Git，还可以通过 GitHub Actions 实现自动化的代码审查、Issue 处理和 PR 管理。这使得 Claude Code 成为你的 CI/CD 流程中的一个智能组件。

## 本地 Git 操作

Claude Code 可以直接执行各种 Git 操作：

### 基本操作

```
> 查看当前的 git 状态

> 提交所有修改，用有描述性的 commit message

> 创建一个新分支叫 feature/user-auth

> 把当前分支的修改合并到 main
```

### 创建 Pull Request

```
> 为当前分支创建一个 PR，描述清楚做了什么改动

Claude Code 会：
1. 查看当前分支的所有 commit
2. 分析变更内容
3. 编写 PR 标题和描述
4. 使用 gh 命令创建 PR
```

## GitHub Actions 集成

Claude Code 可以作为 GitHub Actions 的一部分，在 CI/CD 流程中自动执行任务。

### 自动化 PR 审查

在你的仓库中创建 `.github/workflows/claude-review.yml`：

```yaml
name: Claude Code Review
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Review PR with Claude
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

当有新的 PR 被创建或更新时，Claude 会自动：
1. 分析代码变更
2. 检查潜在问题
3. 在 PR 上留下审查评论

### `@claude` 提及

设置好 GitHub Actions 后，你可以在 Issue 或 PR 评论中使用 `@claude` 来触发 Claude：

```
# 在 Issue 评论中：
@claude 这个 bug 是什么原因导致的？帮我分析一下。

# 在 PR 评论中：
@claude 这个变更有没有性能问题？

# 请求修复：
@claude 请修复这个 Issue
```

Claude 会：
1. 读取 Issue/PR 的上下文
2. 分析代码库
3. 提供分析结果或实施修复
4. 将结果以评论形式反馈

### 自动化 Issue 处理

```yaml
name: Claude Issue Handler
on:
  issues:
    types: [opened, labeled]
  issue_comment:
    types: [created]

jobs:
  handle-issue:
    if: contains(github.event.comment.body, '@claude')
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Handle issue with Claude
        uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## 工作流示例

### 示例一：完整的 PR 流程

```
你：创建一个新分支，实现用户头像功能，写好测试，然后创建 PR

Claude Code 的执行流程：
1. git checkout -b feature/user-avatar
2. 实现用户头像上传功能
3. 编写单元测试和集成测试
4. 运行测试确认通过
5. git add & commit
6. git push -u origin feature/user-avatar
7. gh pr create --title "feat: 添加用户头像功能" --body "..."
```

### 示例二：Issue 到 PR 的自动化

```
GitHub Issue #42: "登录页面在 Safari 浏览器上显示异常"

@claude 请修复这个问题

Claude 自动：
1. 读取 Issue 描述
2. 找到相关代码
3. 创建修复分支
4. 实现修复
5. 运行测试
6. 创建 PR 并关联 Issue
7. 在 Issue 中回复进展
```

### 示例三：自动化代码审查

当有人提交 PR 时，Claude 自动审查并留下评论：

```
Claude Review Comments:

src/auth/login.ts:
⚠️ 潜在的 SQL 注入风险（第 42 行）
建议使用参数化查询替代字符串拼接。

src/utils/validator.ts:
💡 这个正则表达式可以简化。
建议：/^[\w.]+@[\w.]+\.\w+$/

src/components/Form.tsx:
✅ 组件结构清晰，错误处理完善。
```

## 设置步骤

### 1. 添加 Anthropic API Key

在你的 GitHub 仓库设置中：

**Settings → Secrets and variables → Actions → New repository secret**

添加 `ANTHROPIC_API_KEY`，值为你的 Anthropic API Key。

### 2. 创建 Workflow 文件

在仓库中创建 `.github/workflows/` 目录，添加相应的 workflow YAML 文件。

### 3. 配置权限

确保 GitHub Actions 有足够的权限：

```yaml
permissions:
  contents: read
  pull-requests: write
  issues: write
```

## GitLab CI/CD 集成

Claude Code 也支持 GitLab CI/CD 集成，配置方式类似：

```yaml
# .gitlab-ci.yml
claude-review:
  stage: review
  script:
    - claude -p "审查这个 MR 的变更并评论"
  only:
    - merge_requests
```

## 实用技巧

1. **从 PR 审查开始**：这是最简单且价值最高的集成点
2. **限制权限**：给 Claude 最小必要的 GitHub 权限
3. **保护 API Key**：使用 GitHub Secrets 存储，不要硬编码
4. **设置触发条件**：不需要每个事件都触发，选择有价值的触发点
5. **结合 CLAUDE.md**：在仓库中包含 `CLAUDE.md`，Claude 在 CI 中也会读取它

---

## 关键要点

- Claude Code 可以直接执行 Git 操作：提交、创建分支、创建 PR
- 通过 GitHub Actions 实现自动化 PR 审查和 Issue 处理
- `@claude` 提及可以在 Issue/PR 评论中触发 Claude
- 设置需要：Anthropic API Key（存为 Secret）+ Workflow 文件
- 也支持 GitLab CI/CD 集成
- 从 PR 自动审查开始是最佳入门点

---

[← 上一课：MCP 服务器扩展](./05-mcp-servers.md) | [返回目录](../README.md) | [下一课：计划模式与思考模式 →](../module-03-advanced-workflows/01-plan-mode-and-thinking-mode.md)
