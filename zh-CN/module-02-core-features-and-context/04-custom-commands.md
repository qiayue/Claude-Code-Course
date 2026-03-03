# 课时 7：自定义命令

[← 上一课：控制上下文](./03-controlling-context.md) | [返回目录](../README.md) | [下一课：MCP 服务器扩展 →](./05-mcp-servers.md)

---

## 核心概念

Claude Code 自带一些内置命令（如 `/init`、`/compact`、`/clear`），但它真正强大的地方在于你可以创建自己的**自定义命令**（Skills）。本课时将介绍如何创建、配置和分享自定义斜杠命令。

## 什么是 Skills？

Skills 是扩展 Claude Code 能力的 Markdown 文件。创建一个 `SKILL.md`，Claude 就会将它加入自己的工具箱。Claude 会在相关时自动使用它，或者你可以用 `/skill-name` 直接调用。

> **注意**：Skills 系统遵循 [Agent Skills](https://agentskills.io) 开放标准，可跨多个 AI 工具使用。

## 内置 Skills

Claude Code 内置了几个有用的 Skills：

| 命令 | 功能 |
|------|------|
| `/simplify` | 审查最近修改的代码，检查复用性、质量和效率问题，然后修复 |
| `/batch <指令>` | 在代码库中编排大规模并行变更 |
| `/debug [描述]` | 通过读取调试日志来排查 Claude Code 会话问题 |

## 创建你的第一个 Skill

### 步骤一：创建 Skill 目录

```bash
# 个人 Skill（所有项目可用）
mkdir -p ~/.claude/skills/review-code

# 项目 Skill（仅当前项目）
mkdir -p .claude/skills/review-code
```

### 步骤二：编写 SKILL.md

每个 Skill 需要一个 `SKILL.md` 文件，包含两部分：
- **YAML frontmatter**：配置信息（在 `---` 标记之间）
- **Markdown 内容**：Claude 执行时遵循的指令

创建 `~/.claude/skills/review-code/SKILL.md`：

```yaml
---
name: review-code
description: 对代码进行详细审查，检查安全性、性能和最佳实践。当用户要求审查代码或检查代码质量时使用。
---

对代码进行全面审查，按以下维度检查：

1. **安全性**：检查注入漏洞、认证问题、敏感数据泄露
2. **性能**：识别性能瓶颈、不必要的计算、内存泄漏
3. **可读性**：评估命名、结构、注释质量
4. **错误处理**：检查异常处理是否完善
5. **测试覆盖**：评估是否有足够的测试

对每个发现，给出：
- 问题描述
- 严重程度（高/中/低）
- 修复建议
- 代码示例
```

### 步骤三：使用 Skill

两种方式使用：

**直接调用：**
```
> /review-code src/auth/login.ts
```

**自动触发：**
```
> 帮我审查一下这个文件的代码质量
```

Claude 会自动识别并使用合适的 Skill。

## Skill 配置详解

### Frontmatter 字段

```yaml
---
name: my-skill              # Skill 名称，也是 /命令名
description: 描述信息         # 告诉 Claude 什么时候使用这个 Skill
argument-hint: [文件名]       # 自动补全时显示的参数提示
disable-model-invocation: true  # 仅手动调用（Claude 不会自动使用）
user-invocable: false        # 隐藏 / 菜单（仅 Claude 自动使用）
allowed-tools: Read, Grep    # 限制可用工具
context: fork               # 在独立子 Agent 中运行
agent: Explore              # 使用的子 Agent 类型
---
```

### Skill 存储位置

| 位置 | 路径 | 适用范围 |
|------|------|----------|
| 企业级 | 由管理员部署 | 组织内所有用户 |
| 个人级 | `~/.claude/skills/<name>/SKILL.md` | 你的所有项目 |
| 项目级 | `.claude/skills/<name>/SKILL.md` | 仅当前项目 |

## 参数传递

Skills 支持通过 `$ARGUMENTS` 接收参数：

```yaml
---
name: fix-issue
description: 修复 GitHub Issue
disable-model-invocation: true
---

修复 GitHub Issue #$ARGUMENTS：

1. 使用 gh 命令读取 Issue 描述
2. 理解问题需求
3. 实现修复
4. 编写测试
5. 创建 commit
```

使用时：
```
> /fix-issue 42
```

Claude 会收到 "修复 GitHub Issue #42" 的指令。

### 多参数

使用 `$ARGUMENTS[N]` 或简写 `$N` 访问单个参数：

```yaml
---
name: migrate-component
description: 将组件从一个框架迁移到另一个
---

将 $0 组件从 $1 迁移到 $2。
保留所有现有行为和测试。
```

使用：
```
> /migrate-component SearchBar React Vue
```

## 实用 Skill 示例

### 提交代码

```yaml
---
name: commit
description: 智能提交代码
disable-model-invocation: true
---

执行以下步骤：
1. 运行 git diff --staged 查看暂存的变更
2. 如果没有暂存的变更，运行 git diff 查看所有变更并询问用户要暂存哪些
3. 基于变更内容生成遵循 Conventional Commits 格式的 commit message
4. 显示 message 并等待用户确认
5. 执行 commit
```

### 部署

```yaml
---
name: deploy
description: 部署应用到生产环境
disable-model-invocation: true
---

部署 $ARGUMENTS 到生产环境：

1. 运行测试套件
2. 构建应用
3. 推送到部署目标
4. 验证部署是否成功
```

### 生成 API 文档

```yaml
---
name: api-docs
description: 为 API 端点生成文档
---

为指定的 API 文件生成文档：

1. 读取 API 路由文件
2. 提取端点、参数、响应格式
3. 生成 Markdown 格式的 API 文档
4. 包含请求/响应示例
```

## 控制调用权限

| 配置 | 用户可调用 | Claude 可自动调用 |
|------|-----------|-----------------|
| 默认 | ✅ | ✅ |
| `disable-model-invocation: true` | ✅ | ❌ |
| `user-invocable: false` | ❌ | ✅ |

- **`disable-model-invocation: true`**：用于有副作用的操作（部署、发消息等），你不希望 Claude 自己决定何时运行
- **`user-invocable: false`**：用于背景知识，Claude 在需要时自动加载，但作为命令直接调用没有意义

## 动态上下文注入

使用 `` !`command` `` 语法在 Skill 发送给 Claude 之前执行 shell 命令：

```yaml
---
name: pr-summary
description: 总结 PR 变更
context: fork
agent: Explore
---

## PR 上下文
- PR diff: !`gh pr diff`
- PR 评论: !`gh pr view --comments`
- 变更文件: !`gh pr diff --name-only`

## 任务
总结这个 PR 的变更...
```

命令的输出会**替换**占位符，Claude 收到的是已填充了实际数据的提示。

## 添加辅助文件

Skill 可以包含多个文件：

```
my-skill/
├── SKILL.md           # 主指令（必需）
├── template.md        # Claude 填写的模板
├── examples/
│   └── sample.md      # 示例输出
└── scripts/
    └── helper.py      # 可执行脚本
```

在 `SKILL.md` 中引用它们：

```markdown
## 参考资料
- 完整 API 文档见 [reference.md](reference.md)
- 使用示例见 [examples.md](examples.md)
```

## 实用技巧

1. **从简单开始**：先创建只有几行的 Skill，逐步完善
2. **测试两种调用方式**：确保直接调用和自动触发都正常
3. **提交到 Git**：项目级 Skills 放在 `.claude/skills/` 并提交，团队共享
4. **避免过多 Skills**：每个 Skill 的描述会占用上下文空间
5. **使用 `context: fork`**：对于复杂或长时间运行的 Skill，使用独立上下文

---

## 关键要点

- Skills 是用 Markdown 编写的自定义命令，存储在 `.claude/skills/` 目录
- 每个 Skill 包含 YAML frontmatter（配置）和 Markdown 内容（指令）
- 支持参数传递（`$ARGUMENTS`）、动态上下文注入（`` !`command` ``）、辅助文件
- 可以控制调用权限：仅用户调用 / 仅 Claude 自动调用 / 两者都可
- 内置 Skills 包括 `/simplify`、`/batch`、`/debug`
- Skills 可以在个人级、项目级和企业级共享

---

[← 上一课：控制上下文](./03-controlling-context.md) | [返回目录](../README.md) | [下一课：MCP 服务器扩展 →](./05-mcp-servers.md)
