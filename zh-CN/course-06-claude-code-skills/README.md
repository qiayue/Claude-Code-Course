# Claude Code Skills — Claude Code 技能

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程和 [Claude Code 官方文档](https://code.claude.com/docs/en/skills) 编写，旨在帮助中文用户深入理解和使用 Claude Code 的 Skills 系统。

---

## 课程简介

**Skills** 是 Claude Code 中强大的扩展机制，允许你通过 Markdown 文件定义可复用的指令集。Skills 可以被手动调用（如 `/skill-name`），也可以由 Claude 在适当时自动使用。本课程将系统讲解 Skills 的创建、配置、分享和最佳实践。

Skills 遵循 [Agent Skills](https://agentskills.io) 开放标准，意味着你编写的 Skills 不仅适用于 Claude Code，还可以在支持该标准的其他 AI 工具中使用。

### 学习目标

完成本课程后，你将能够：

- 理解 Skills 的核心概念和运作机制
- 编写包含 YAML frontmatter 的 `SKILL.md` 文件
- 使用参数传递和动态上下文注入构建高级 Skills
- 通过配置选项精确控制 Skill 的调用权限和执行环境
- 利用内置 Skills（`/simplify`、`/batch`、`/debug`）提升开发效率
- 在团队和社区中分享和管理 Skills

### 前置要求

- 基本的 Claude Code 使用经验（建议先完成 [Claude Code 实战](../course-02-claude-code-in-action/README.md) 课程）
- 熟悉 Markdown 和 YAML 语法
- 命令行基本操作能力

---

## 课程目录

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [什么是 Skills](./01-what-are-skills.md) | Skills 概述、SKILL.md 格式、存储位置、开放标准 |
| 2 | [创建 Skills](./02-creating-skills.md) | 编写 SKILL.md、frontmatter 配置、参数传递、动态上下文注入 |
| 3 | [配置与控制](./03-configuring-skills.md) | 调用权限、工具限制、子 Agent 执行、内置 Skills |
| 4 | [分享与最佳实践](./04-sharing-skills.md) | 版本控制、插件分发、团队协作、故障排查 |

---

## 学习建议

1. **动手实践**：每学完一个概念，立即在你的项目中创建一个对应的 Skill
2. **从简单开始**：先创建只有几行指令的 Skill，逐步添加参数和配置
3. **参考内置 Skills**：研究 `/simplify`、`/batch`、`/debug` 的实现方式
4. **团队协作**：将有用的 Skills 提交到项目仓库，与团队共享

---

## 参考资源

- [Claude Code Skills 文档](https://code.claude.com/docs/en/skills)
- [Agent Skills 开放标准](https://agentskills.io)
- [Claude Code 官方文档](https://code.claude.com/docs/en/overview)
- [Anthropic Academy](https://anthropic.skilljar.com/)
