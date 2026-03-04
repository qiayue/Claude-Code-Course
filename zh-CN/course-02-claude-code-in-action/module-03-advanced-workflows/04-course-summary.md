# 课时 13：课程总结

[← 上一课：Claude Code SDK](./03-claude-code-sdk.md) | [返回目录](../README.md)

---

## 课程回顾

恭喜你完成了 **Claude Code 实战** 课程的学习！让我们回顾一下你所掌握的知识。

## 知识地图

```
Claude Code 实战
│
├── 模块一：编程助手基础
│   ├── AI 编程助手的演进：补全 → Copilot → Agent
│   ├── Claude Code 架构：LLM + 工具系统 + Agentic 循环
│   └── 安装启动：原生安装 / Homebrew / WinGet + /init
│
├── 模块二：核心功能与上下文管理
│   ├── 上下文管理：CLAUDE.md 层级体系 + @引用 + .claude/rules/
│   ├── 代码修改：Edit 精确替换 + 多文件操作 + Git 集成
│   ├── 会话控制：/compact + /clear + 自动记忆系统
│   ├── 自定义命令：Skills 系统 + SKILL.md + 参数传递
│   ├── MCP 扩展：协议概念 + 服务器配置 + 社区生态
│   └── GitHub 集成：Actions + @claude + 自动化 PR 审查
│
└── 模块三：高级工作流
    ├── 计划模式：广度优先，先规划后执行
    ├── 思考模式：深度优先，think/think hard/ultrathink
    ├── Hooks：生命周期钩子，自动化保障
    └── SDK：编程式集成，自定义工具开发
```

## 核心概念速查表

### 常用命令

| 命令 | 功能 |
|------|------|
| `claude` | 启动 Claude Code |
| `/init` | 初始化项目，生成 CLAUDE.md |
| `/compact` | 压缩对话，保留关键信息 |
| `/clear` | 清除对话历史 |
| `/memory` | 管理记忆系统 |
| `/help` | 查看帮助信息 |

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `Shift+Tab` | 切换计划模式 |
| `Esc` | 取消当前操作 |

### 关键文件

| 文件 | 用途 |
|------|------|
| `CLAUDE.md` | 项目指令（团队共享） |
| `CLAUDE.local.md` | 个人项目偏好（不提交 Git） |
| `~/.claude/CLAUDE.md` | 用户级全局偏好 |
| `.claude/rules/*.md` | 模块化规则（可按路径限定） |
| `.claude/skills/*/SKILL.md` | 自定义命令 |
| `.claude/settings.json` | 项目设置（含 MCP 和 Hooks） |
| `~/.claude/settings.json` | 用户级设置 |

### 思考模式关键词

| 关键词 | 深度 |
|--------|------|
| `think` | 基础思考 |
| `think hard` | 深入思考 |
| `think harder` | 更深入思考 |
| `ultrathink` | 最深度思考 |

## 最佳实践清单

### 日常使用

- [ ] 每个新项目运行 `/init` 生成 `CLAUDE.md`
- [ ] 定期更新 `CLAUDE.md` 以反映项目变化
- [ ] 使用 `@` 引用提供具体的文件上下文
- [ ] 描述意图而非步骤
- [ ] 粘贴错误信息而非描述错误

### 上下文管理

- [ ] `CLAUDE.md` 控制在 200 行以内
- [ ] 使用 `.claude/rules/` 拆分复杂规则
- [ ] 对话过长时使用 `/compact`
- [ ] 不相关任务使用 `/clear`
- [ ] 利用自动记忆积累项目知识

### 团队协作

- [ ] 项目级 `CLAUDE.md` 提交到版本控制
- [ ] 项目级 Skills 放在 `.claude/skills/` 并提交
- [ ] Hook 配置放在 `.claude/settings.json` 并提交
- [ ] 个人偏好放在 `CLAUDE.local.md`（自动 gitignore）
- [ ] 设置 GitHub Actions 实现自动化 PR 审查

### 安全

- [ ] 使用权限系统控制 Claude 的操作
- [ ] 使用 Hooks 保护敏感文件
- [ ] API Key 存在环境变量中，不硬编码
- [ ] 审查 Claude 对生产环境的操作
- [ ] 使用 MCP 时遵循最小权限原则

## 进阶学习路径

掌握了本课程的内容后，你可以继续探索以下方向：

### 1. 深入 Claude Code 文档

- [快速入门](https://code.claude.com/docs/en/quickstart)
- [最佳实践](https://code.claude.com/docs/en/best-practices)
- [常见工作流](https://code.claude.com/docs/en/common-workflows)
- [子 Agent](https://code.claude.com/docs/en/sub-agents)

### 2. MCP 生态

- [MCP 规范](https://modelcontextprotocol.io/)
- 探索社区 MCP 服务器
- 开发自己的 MCP 服务器

### 3. Agent SDK

- [Agent SDK 文档](https://platform.claude.com/docs/en/agent-sdk/overview)
- 构建自定义 Agent
- 多 Agent 协作系统

### 4. Anthropic Academy 其他课程

- [Claude 101](https://anthropic.skilljar.com/claude-101) — Claude 基础入门
- [Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api) — API 开发
- [MCP 课程](https://anthropic.skilljar.com/) — MCP 服务器开发

## 常见问题

### Q: Claude Code 会读取我的所有文件吗？

不会。Claude Code 只在需要时通过工具系统读取文件。每次文件访问都经过权限系统，你可以选择批准或拒绝。

### Q: Claude Code 的修改会直接保存吗？

是的。Claude Code 对文件的修改是直接写入磁盘的。建议使用 Git 进行版本控制，以便在需要时回滚。

### Q: 如何在团队中统一 Claude Code 的行为？

通过提交 `CLAUDE.md`、`.claude/rules/`、`.claude/skills/` 和 `.claude/settings.json` 到版本控制，团队成员使用相同的项目配置。

### Q: Claude Code 与 Claude API 有什么区别？

Claude Code 是一个完整的 Agent 应用，包含文件操作、命令执行等工具。Claude API 是底层的 LLM 接口。Claude Code 在内部使用 Claude API，但提供了更高层的开发者体验。

---

## 结语

Claude Code 正在改变开发者与 AI 协作的方式。从简单的代码补全到智能的 Agent 式编程助手，它代表了开发工具演进的一个重要方向。

希望本课程帮助你掌握了 Claude Code 的核心功能和高级特性。现在，是时候将它应用到你的实际项目中了！

> **官方课程**：想获得官方证书？请访问 [Anthropic Academy](https://anthropic.skilljar.com/claude-code-in-action) 完成在线课程。

---

[← 上一课：Claude Code SDK](./03-claude-code-sdk.md) | [返回目录](../README.md)
