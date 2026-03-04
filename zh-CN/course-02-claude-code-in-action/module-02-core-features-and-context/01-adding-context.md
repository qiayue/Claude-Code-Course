# 课时 4：添加上下文

[← 上一课：Claude Code 初体验](../module-01-coding-assistant-fundamentals/03-claude-code-in-action.md) | [返回目录](../README.md) | [下一课：进行代码修改 →](./02-making-changes.md)

---

## 核心概念

Claude Code 的每次会话都从一个**全新的上下文窗口**开始。这意味着它不会自动记住之前的对话或你的项目偏好。为了让 Claude Code 持续有效地理解你的项目，你需要学习如何管理**上下文**。

本课时将介绍 Claude Code 最重要的上下文管理机制：`CLAUDE.md` 文件系统。

## CLAUDE.md 是什么？

`CLAUDE.md` 是一个 Markdown 文件，用于给 Claude Code 提供持久化的项目指令。每次启动会话时，Claude Code 会自动读取这个文件。

你可以把 `CLAUDE.md` 想象成给一位新入职同事的**项目指南** —— 它包含了项目的关键信息、规范和注意事项。

## CLAUDE.md 文件层级

Claude Code 支持多个层级的 `CLAUDE.md`，从最广泛到最具体：

| 层级 | 位置 | 用途 | 共享范围 |
|------|------|------|----------|
| **组织级** | `/Library/Application Support/ClaudeCode/CLAUDE.md`（macOS）<br>`/etc/claude-code/CLAUDE.md`（Linux） | 公司级编码标准和安全策略 | 组织内所有用户 |
| **项目级** | `./CLAUDE.md` 或 `./.claude/CLAUDE.md` | 团队共享的项目指令 | 通过版本控制共享给团队 |
| **用户级** | `~/.claude/CLAUDE.md` | 个人偏好设置 | 仅限个人（所有项目） |
| **本地级** | `./CLAUDE.local.md` | 个人的项目特定偏好 | 仅限个人（当前项目） |

**优先级规则**：更具体的位置优先级更高。本地级 > 项目级 > 用户级 > 组织级。

## 使用 `/init` 自动生成

最快速的方式是使用 `/init` 命令：

```
> /init
```

Claude Code 会自动：
1. 扫描你的代码库结构
2. 识别使用的框架和工具
3. 发现构建和测试命令
4. 生成一份初始的 `CLAUDE.md`

生成的文件示例：

```markdown
# Project: e-commerce-app

## Build & Test
- Install: `pnpm install`
- Dev server: `pnpm dev`
- Build: `pnpm build`
- Test all: `pnpm test`
- Test single: `pnpm test -- path/to/test`
- Lint: `pnpm lint`

## Architecture
- Monorepo with pnpm workspaces
- Frontend: Next.js 14 (App Router) + TypeScript
- Backend: Express.js + TypeScript
- Database: PostgreSQL with Drizzle ORM
- Auth: NextAuth.js v5

## Conventions
- Use `async/await` over `.then()` chains
- Components in PascalCase, utilities in camelCase
- All API responses follow `{ data, error, message }` format
- Commits follow conventional commits format
```

> **提示**：`/init` 生成的内容只是起点。你应该手动补充 Claude 无法自动发现的信息，比如业务上下文、架构决策的原因等。

## 编写有效的 CLAUDE.md

### 大小控制

目标：**每个 CLAUDE.md 文件控制在 200 行以内**。

文件越长，消耗的上下文窗口越多，Claude 遵循指令的一致性也会下降。如果指令太多，考虑拆分到 `.claude/rules/` 目录。

### 结构化组织

使用 Markdown 标题和列表来组织内容：

```markdown
# 项目信息

## 构建命令
- 安装依赖：`npm install`
- 开发模式：`npm run dev`
- 运行测试：`npm test`

## 编码规范
- 使用 2 空格缩进
- 提交前必须通过 lint 检查
- API handlers 放在 `src/api/handlers/` 目录

## 架构决策
- 使用 Redis 做会话存储（而非 JWT），因为需要支持主动注销
- 文件上传使用 S3 预签名 URL，不经过应用服务器
```

### 具体明确

**好的指令：**
```markdown
- 使用 2 空格缩进
- 提交前运行 `npm test`
- API handlers 放在 `src/api/handlers/`
```

**不好的指令：**
```markdown
- 代码格式化要好
- 记得测试
- 文件要放对地方
```

## `@` 引用文件

在 `CLAUDE.md` 中，你可以使用 `@` 语法导入其他文件的内容：

```markdown
参考 @README.md 了解项目概述。
查看 @package.json 了解可用的 npm 命令。

# 额外指令
- Git 工作流参考 @docs/git-workflow.md
```

导入的文件会在启动时自动加载到上下文中。支持相对路径和绝对路径，最多支持 5 层嵌套导入。

> **注意**：首次遇到外部导入时，Claude Code 会显示一个审批对话框，列出要导入的文件。

## `.claude/rules/` 目录

对于大型项目，可以将指令拆分到 `.claude/rules/` 目录下的多个文件中：

```
your-project/
├── .claude/
│   ├── CLAUDE.md           # 主要项目指令
│   └── rules/
│       ├── code-style.md   # 代码风格规范
│       ├── testing.md      # 测试约定
│       └── security.md     # 安全要求
```

### 按路径限定规则

规则文件可以使用 YAML frontmatter 限定只在特定文件上生效：

```markdown
---
paths:
  - "src/api/**/*.ts"
---

# API 开发规则

- 所有 API 端点必须包含输入验证
- 使用标准错误响应格式
- 包含 OpenAPI 文档注释
```

常用的路径模式：

| 模式 | 匹配 |
|------|------|
| `**/*.ts` | 所有 TypeScript 文件 |
| `src/**/*` | src 目录下的所有文件 |
| `*.md` | 项目根目录的 Markdown 文件 |
| `src/components/*.tsx` | 特定目录的 React 组件 |

## 对话中的 `@` 引用

在与 Claude Code 对话时，你也可以使用 `@` 来引用文件：

```
> 看看 @src/auth/login.ts 和 @src/auth/register.ts，找出共同的模式并提取到一个共享工具函数中
```

这让 Claude Code 立即获得相关文件的上下文，而不需要它自己去搜索。

## CLAUDE.local.md — 个人偏好

`CLAUDE.local.md` 用于存放**不需要提交到版本控制**的个人项目偏好：

```markdown
# 我的本地设置

## 开发环境
- 本地 API 地址：http://localhost:3001
- 测试数据库：postgresql://localhost:5432/myapp_test

## 个人偏好
- 代码注释用中文
- 生成的测试使用详细的描述性名称
```

这个文件自动被添加到 `.gitignore`。

## 实用技巧

1. **先运行 `/init`，再手动优化**：自动生成提供基础，人工补充提供深度
2. **项目级 CLAUDE.md 提交到 Git**：让团队成员共享统一的项目上下文
3. **定期审查和更新**：随着项目演进，确保 CLAUDE.md 保持最新
4. **避免指令冲突**：如果两条规则矛盾，Claude 可能会随机选择其中一条
5. **使用 `/memory` 查看加载的文件**：确认你的指令是否被正确加载

---

## 关键要点

- 每次会话从空白上下文开始，`CLAUDE.md` 是跨会话持久化信息的主要方式
- 使用 `/init` 快速生成初始 `CLAUDE.md`，然后手动完善
- 四个层级：组织级 → 项目级 → 用户级 → 本地级
- 使用 `@` 语法导入文件和引用上下文
- `.claude/rules/` 支持模块化的、按路径限定的规则
- `CLAUDE.local.md` 用于不提交到 Git 的个人偏好

---

[← 上一课：Claude Code 初体验](../module-01-coding-assistant-fundamentals/03-claude-code-in-action.md) | [返回目录](../README.md) | [下一课：进行代码修改 →](./02-making-changes.md)
