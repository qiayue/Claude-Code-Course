# Claude Code 实战 (Claude Code in Action)

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程 **[Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action)** 的公开内容和 [Claude Code 官方文档](https://code.claude.com/docs/en/overview) 编写。

---

## 课程简介

**Claude Code** 是 Anthropic 推出的 AI 编程助手，运行在你的终端中，能够阅读代码库、编辑文件、执行命令，并与你的开发工具集成。本课程将从基础到高级，系统地教你如何在日常开发中高效使用 Claude Code。

### 学习目标

完成本课程后，你将能够：

- 使用 Claude Code 的核心工具进行文件操作、命令执行和代码分析
- 通过 `/init`、`CLAUDE.md` 文件和 `@` 引用有效管理上下文
- 使用快捷键和命令控制对话流程
- 启用计划模式（Plan Mode）和思考模式（Thinking Mode）处理复杂任务
- 创建自定义命令，自动化重复性开发工作流
- 通过 MCP 服务器扩展 Claude Code 功能
- 设置 GitHub 集成，实现自动化 PR 审查和 Issue 处理
- 编写 Hooks 为 Claude Code 添加自定义行为

### 前置要求

- 基本的命令行使用经验
- Claude Code 访问权限（需要 [Claude 订阅](https://claude.com/pricing) 或 [Anthropic Console](https://console.anthropic.com/) 账号）

---

## 课程目录

### 模块一：编程助手基础

> 理解 AI 编程助手的工作原理，安装并初步体验 Claude Code

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [课程介绍](./module-01-coding-assistant-fundamentals/01-introduction.md) | 课程概述与学习路线 |
| 2 | [什么是编程助手？](./module-01-coding-assistant-fundamentals/02-what-is-a-coding-assistant.md) | AI 编程助手的架构与工作原理 |
| 3 | [Claude Code 初体验](./module-01-coding-assistant-fundamentals/03-claude-code-in-action.md) | 安装、启动和核心工具使用 |

### 模块二：核心功能与上下文管理

> 掌握 Claude Code 的核心功能，学习上下文管理和工作流扩展

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 4 | [添加上下文](./module-02-core-features-and-context/01-adding-context.md) | CLAUDE.md、/init 命令和上下文注入 |
| 5 | [进行代码修改](./module-02-core-features-and-context/02-making-changes.md) | 文件编辑、代码生成和多文件操作 |
| 6 | [控制上下文](./module-02-core-features-and-context/03-controlling-context.md) | 会话管理、/compact、/clear 和对话控制 |
| 7 | [自定义命令](./module-02-core-features-and-context/04-custom-commands.md) | 创建和使用 Skills 与自定义斜杠命令 |
| 8 | [MCP 服务器扩展](./module-02-core-features-and-context/05-mcp-servers.md) | 通过 MCP 协议连接外部工具和数据源 |
| 9 | [GitHub 集成](./module-02-core-features-and-context/06-github-integration.md) | GitHub Actions、自动化 PR 审查和 Issue 处理 |

### 模块三：高级工作流

> 深入了解高级功能，包括思考模式、Hooks 系统和 SDK

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 10 | [计划模式与思考模式](./module-03-advanced-workflows/01-plan-mode-and-thinking-mode.md) | 不同推理模式的使用场景和最佳实践 |
| 11 | [Hooks 系统](./module-03-advanced-workflows/02-hooks.md) | 生命周期钩子、自动化工作流和安全控制 |
| 12 | [Claude Code SDK](./module-03-advanced-workflows/03-claude-code-sdk.md) | 编程式集成，将 Claude Code 嵌入自定义工具 |
| 13 | [课程总结](./module-03-advanced-workflows/04-course-summary.md) | 知识回顾、最佳实践和进阶资源 |

---

## 参考资源

- [Claude Code 官方文档](https://code.claude.com/docs/en/overview)
- [Claude Code in Action（Coursera）](https://www.coursera.org/learn/claude-code-in-action)
- [Claude Code 产品页面](https://claude.com/product/claude-code)
