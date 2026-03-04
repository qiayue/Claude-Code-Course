# 课时 1：什么是 Skills

[返回目录](./README.md) | [下一课：创建 Skills →](./02-creating-skills.md)

---

## 核心概念

**Skills** 是 Claude Code 中用于扩展能力的可复用 Markdown 指令文件。你可以把 Skills 理解为 Claude Code 的"技能包"——通过编写一个 `SKILL.md` 文件，你可以教 Claude 新的工作流程、专业知识和自动化操作。

与传统的脚本或插件不同，Skills 使用自然语言编写，不需要学习新的编程语言或 API。

## Skills 的工作原理

Skills 的核心机制非常直观：

1. 你编写一个包含指令的 Markdown 文件（`SKILL.md`）
2. Claude Code 在启动时发现并加载这些文件
3. 当用户调用或场景匹配时，Claude 读取并遵循 Skill 中的指令

```
用户输入 /review-code
        ↓
Claude Code 查找名为 "review-code" 的 Skill
        ↓
加载 SKILL.md 内容
        ↓
Claude 按照指令执行代码审查
```

## SKILL.md 文件格式

每个 Skill 由一个 `SKILL.md` 文件定义，包含两部分：

### 1. YAML Frontmatter（元数据）

位于文件顶部的 `---` 标记之间，定义 Skill 的配置信息：

```yaml
---
name: review-code
description: 对代码进行全面审查，检查安全性、性能和最佳实践
argument-hint: <文件路径>
---
```

### 2. Markdown 内容（指令）

Frontmatter 之后的部分是 Claude 执行时遵循的具体指令：

```markdown
---
name: review-code
description: 对代码进行全面审查
---

对指定的代码文件进行全面审查：

1. **安全性检查**：识别注入漏洞、认证问题、敏感数据泄露
2. **性能分析**：查找性能瓶颈、不必要的计算、内存泄漏
3. **代码质量**：评估命名规范、代码结构、注释质量
4. **错误处理**：检查异常处理是否完善

输出格式：
- 每个问题附带严重程度（高/中/低）
- 提供具体的修复建议和代码示例
```

## Skills 的存储位置

Skills 可以存放在三个层级，适用范围从窄到宽：

| 层级 | 路径 | 适用范围 | 典型用途 |
|------|------|----------|----------|
| 项目级 | `.claude/skills/<name>/SKILL.md` | 仅当前项目 | 项目特定的工作流 |
| 个人级 | `~/.claude/skills/<name>/SKILL.md` | 你的所有项目 | 个人通用的开发习惯 |
| 企业级 | 由管理员部署 | 组织内所有用户 | 团队标准和规范 |

### 项目级 Skills

适用于与特定项目相关的工作流：

```bash
# 创建项目级 Skill
mkdir -p .claude/skills/deploy
cat > .claude/skills/deploy/SKILL.md << 'EOF'
---
name: deploy
description: 部署当前项目到生产环境
disable-model-invocation: true
---

按照以下步骤部署项目：

1. 运行 `npm run test` 确保所有测试通过
2. 运行 `npm run build` 构建生产版本
3. 运行 `npm run deploy` 执行部署
4. 验证部署是否成功
EOF
```

### 个人级 Skills

适用于你在所有项目中都会使用的通用工作流：

```bash
# 创建个人级 Skill
mkdir -p ~/.claude/skills/commit
cat > ~/.claude/skills/commit/SKILL.md << 'EOF'
---
name: commit
description: 智能生成 Git 提交信息
disable-model-invocation: true
---

执行以下步骤：

1. 运行 `git diff --staged` 查看暂存的变更
2. 分析变更内容和目的
3. 生成符合 Conventional Commits 规范的提交信息
4. 展示提交信息并等待用户确认
5. 执行 `git commit`
EOF
```

### 企业级 Skills

由组织管理员统一部署，确保团队遵循一致的标准和流程。这些 Skills 对组织内所有用户自动生效。

## 两种调用方式

Skills 可以通过两种方式被使用：

### 1. 手动调用（斜杠命令）

在 Claude Code 中输入 `/skill-name`：

```
> /review-code src/auth/login.ts
```

### 2. 自动触发

Claude 根据 `description` 字段判断是否自动使用某个 Skill：

```
> 帮我检查一下这个文件有没有安全问题
# Claude 自动识别并使用 review-code Skill
```

## Agent Skills 开放标准

Claude Code 的 Skills 系统遵循 **[Agent Skills](https://agentskills.io)** 开放标准。这意味着：

- **跨工具兼容**：你编写的 Skills 可以在其他支持该标准的 AI 工具中使用
- **社区共享**：通过开放标准，开发者社区可以共享和复用 Skills
- **标准化格式**：统一的 `SKILL.md` 格式降低了学习和迁移成本
- **生态系统**：随着更多工具采用该标准，Skills 的价值会持续增长

### 标准的核心要素

Agent Skills 标准定义了以下内容：

1. **文件格式**：使用 Markdown + YAML frontmatter
2. **目录结构**：`skills/<name>/SKILL.md` 的约定
3. **元数据字段**：`name`、`description`、`argument-hint` 等
4. **参数传递**：`$ARGUMENTS`、`$0`、`$1` 等变量
5. **动态上下文**：`` !`command` `` 语法

## Skills vs CLAUDE.md

你可能会问：Skills 和 CLAUDE.md 有什么区别？

| 特性 | CLAUDE.md | Skills |
|------|-----------|--------|
| 用途 | 项目整体上下文和规则 | 特定任务的指令集 |
| 加载方式 | 每次会话自动加载 | 按需加载或自动匹配 |
| 调用方式 | 始终生效 | `/命令名` 或自动触发 |
| 参数支持 | 不支持 | 支持 `$ARGUMENTS` |
| 存储位置 | 项目根目录 | `.claude/skills/` 目录 |

简单来说：**CLAUDE.md 告诉 Claude "你在哪里工作"，Skills 告诉 Claude "如何完成特定任务"**。

## 内置 Skills 预览

Claude Code 自带几个实用的内置 Skills：

| 命令 | 功能描述 |
|------|----------|
| `/simplify` | 审查最近修改的代码，检查复用性、质量和效率问题，然后修复 |
| `/batch <指令>` | 在代码库中编排大规模并行变更 |
| `/debug [描述]` | 通过读取调试日志来排查 Claude Code 会话问题 |

这些内置 Skills 是学习 Skill 编写的绝佳参考。

---

## 关键要点

- Skills 是用 Markdown 编写的可复用指令文件，存储为 `SKILL.md`
- 每个 Skill 包含 YAML frontmatter（配置）和 Markdown 内容（指令）
- Skills 存储在三个层级：项目级、个人级、企业级
- 支持手动调用（`/命令名`）和 Claude 自动触发两种方式
- 遵循 Agent Skills 开放标准，具有跨工具兼容性
- Skills 与 CLAUDE.md 互补：CLAUDE.md 定义项目上下文，Skills 定义具体任务

---

[返回目录](./README.md) | [下一课：创建 Skills →](./02-creating-skills.md)
