# 课时 3：Claude Code 初体验

[← 上一课：什么是编程助手？](./02-what-is-a-coding-assistant.md) | [返回目录](../README.md) | [下一课：添加上下文 →](../module-02-core-features-and-context/01-adding-context.md)

---

## 核心概念

本课时将带你完成 Claude Code 的安装、首次启动，并通过实际操作了解其核心工具。

## 安装 Claude Code

Claude Code 提供多种安装方式：

### 方式一：原生安装（推荐）

**macOS / Linux / WSL：**
```bash
curl -fsSL https://claude.ai/install.sh | bash
```

**Windows PowerShell：**
```powershell
irm https://claude.ai/install.ps1 | iex
```

**Windows CMD：**
```batch
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

> **注意**：Windows 用户需要先安装 [Git for Windows](https://git-scm.com/downloads/win)。

原生安装会**自动更新**，确保你始终使用最新版本。

### 方式二：Homebrew（macOS）

```bash
brew install --cask claude-code
```

> 通过 Homebrew 安装不会自动更新，需要定期运行 `brew upgrade claude-code`。

### 方式三：WinGet（Windows）

```powershell
winget install Anthropic.ClaudeCode
```

## 首次启动

安装完成后，进入你的项目目录并启动 Claude Code：

```bash
cd your-project
claude
```

首次启动时，系统会提示你进行身份验证。你可以选择：

- **Claude 订阅账号**：使用你的 Claude Pro/Team 账号登录
- **Anthropic Console**：使用 API Key 进行认证

认证完成后，你将看到 Claude Code 的交互界面：

```
╭─────────────────────────────────────────╮
│ ● Claude Code                           │
│                                         │
│ Welcome to Claude Code!                 │
│                                         │
│ /help for available commands            │
╰─────────────────────────────────────────╯

>
```

## 初始化项目：`/init`

启动后的第一步建议是运行 `/init` 命令：

```
> /init
```

`/init` 会做以下事情：

1. **扫描代码库**：分析项目结构、依赖、配置文件
2. **生成 `CLAUDE.md`**：创建一个包含项目信息的配置文件
3. **识别关键信息**：记录构建命令、测试命令、项目约定等

生成的 `CLAUDE.md` 文件可能包含：

```markdown
# Project: my-web-app

## Build & Test
- Install: `npm install`
- Dev: `npm run dev`
- Test: `npm test`
- Lint: `npm run lint`

## Architecture
- Framework: Next.js 14 with App Router
- Language: TypeScript
- Styling: Tailwind CSS
- Database: PostgreSQL with Prisma ORM

## Conventions
- Use functional components with hooks
- File naming: kebab-case for files, PascalCase for components
```

> **提示**：如果项目中已存在 `CLAUDE.md`，`/init` 会建议改进而不是覆盖它。

## 核心工具详解

Claude Code 通过一组核心工具与你的开发环境交互。了解这些工具有助于理解 Claude Code 的能力边界。

### Read — 读取文件

Claude Code 使用 Read 工具查看文件内容：

```
> 帮我看看 src/auth/login.ts 的内容

Claude 会调用 Read 工具读取文件，然后向你解释代码逻辑。
```

### Write — 创建文件

当需要创建全新的文件时，Claude Code 使用 Write 工具：

```
> 帮我创建一个新的 API 路由 /api/users

Claude 会使用 Write 工具创建文件，包含完整的代码。
```

### Edit — 编辑文件

修改现有文件时，Claude Code 使用 Edit 工具进行精确的字符串替换：

```
> 把 getUserName 函数改成支持多语言

Claude 会使用 Edit 工具，只修改需要变更的部分，
而不是重写整个文件。
```

### Bash — 执行命令

Claude Code 可以在终端中执行命令：

```
> 运行测试看看有没有失败的

Claude 会执行：npm test
然后分析输出结果。
```

### Glob — 文件搜索

按模式查找文件：

```
> 找到所有的 React 组件文件

Claude 会使用 Glob 搜索，例如：**/*.tsx
```

### Grep — 内容搜索

在文件中搜索特定内容：

```
> 找到所有使用了 deprecated API 的地方

Claude 会使用 Grep 在代码中搜索相关模式。
```

## 权限系统

Claude Code 的一个重要特性是其**权限系统**。默认情况下，Claude Code 在执行某些操作前会请求你的确认：

```
Claude wants to run: npm test

Allow? (y/n/always)
```

你可以选择：
- **y** — 允许这次执行
- **n** — 拒绝
- **always** — 始终允许这类操作

这确保了你始终对 Claude Code 的操作保持控制。

## 实际操作演示

让我们通过一个完整的例子来体验 Claude Code：

### 场景：探索一个不熟悉的代码库

```
> 这个项目是做什么的？主要的入口文件在哪里？
```

Claude Code 会：
1. 读取 `README.md`、`package.json` 等文件
2. 浏览项目目录结构
3. 识别主要入口点
4. 给你一个清晰的项目概述

### 场景：修复一个 Bug

```
> 用户报告说登录页面在移动端显示异常，帮我查找并修复这个问题
```

Claude Code 会：
1. 找到登录页面的组件文件
2. 分析样式代码
3. 识别响应式设计问题
4. 提出并实施修复方案
5. 建议如何验证修复

### 场景：创建一个新功能

```
> 帮我添加一个用户头像上传功能，需要支持裁剪和压缩
```

Claude Code 会：
1. 分析现有的用户模块结构
2. 规划需要创建/修改的文件
3. 逐步实现功能
4. 添加必要的错误处理
5. 建议测试方案

## 可用的运行环境

除了终端，Claude Code 还支持多个环境：

| 环境 | 说明 |
|------|------|
| **终端 CLI** | 完整功能的命令行界面 |
| **VS Code** | 编辑器内集成，支持内联 diff |
| **JetBrains** | IntelliJ/PyCharm/WebStorm 插件 |
| **桌面应用** | 独立应用，支持可视化 diff 审查 |
| **Web** | 浏览器中使用，无需本地安装 |

所有环境共享相同的底层引擎，你的 `CLAUDE.md`、设置和 MCP 服务器在所有环境中通用。

## 实用技巧

1. **用自然语言交流**：不需要特殊语法，直接描述你的需求
2. **提供具体信息**：错误信息、文件路径、期望行为越具体越好
3. **利用 `/init`**：每个新项目启动时都运行一次
4. **善用权限系统**：对不确定的操作选择逐次确认

---

## 关键要点

- Claude Code 支持多种安装方式，推荐使用原生安装以获得自动更新
- 首次使用项目时运行 `/init` 生成 `CLAUDE.md`
- 核心工具包括：Read、Write、Edit、Bash、Glob、Grep
- 权限系统确保你始终掌控 Claude Code 的操作
- Claude Code 可在终端、VS Code、JetBrains、桌面应用和 Web 中使用

---

[← 上一课：什么是编程助手？](./02-what-is-a-coding-assistant.md) | [返回目录](../README.md) | [下一课：添加上下文 →](../module-02-core-features-and-context/01-adding-context.md)
