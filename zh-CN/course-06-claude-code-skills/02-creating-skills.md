# 课时 2：创建 Skills

[← 上一课：什么是 Skills](./01-what-are-skills.md) | [返回目录](./README.md) | [下一课：配置与控制 →](./03-configuring-skills.md)

---

## 核心概念

本课时将深入讲解如何从零开始创建一个 Skill，包括 YAML frontmatter 的配置字段、参数传递机制、辅助文件的使用，以及动态上下文注入等高级功能。

## 创建 Skill 的基本步骤

### 第一步：创建目录

每个 Skill 需要自己的目录：

```bash
# 项目级 Skill
mkdir -p .claude/skills/my-skill

# 个人级 Skill
mkdir -p ~/.claude/skills/my-skill
```

### 第二步：编写 SKILL.md

在目录中创建 `SKILL.md` 文件：

```bash
cat > .claude/skills/my-skill/SKILL.md << 'EOF'
---
name: my-skill
description: 这个 Skill 的功能描述
---

这里是 Claude 执行时遵循的指令。
EOF
```

### 第三步：测试 Skill

在 Claude Code 中输入：

```
> /my-skill
```

如果一切正确，Claude 会加载并执行你的 Skill。

## Frontmatter 配置详解

YAML frontmatter 是 Skill 的"身份证"，定义了它的名称、用途和行为方式。

### name（名称）

Skill 的唯一标识符，也是斜杠命令的名称：

```yaml
---
name: review-code
---
```

调用方式：`/review-code`

**命名建议：**
- 使用小写字母和连字符：`review-code`、`fix-issue`
- 名称要简洁且描述性强
- 避免与内置命令冲突（如 `/init`、`/clear`）

### description（描述）

告诉 Claude 什么时候应该使用这个 Skill：

```yaml
---
name: review-code
description: 对代码进行详细审查，检查安全性、性能和最佳实践。当用户要求审查代码或检查代码质量时使用。
---
```

**描述的重要性：**
- Claude 根据 `description` 判断是否自动使用该 Skill
- 描述越清晰，Claude 的匹配越准确
- 包含触发场景的关键词有助于自动匹配

### argument-hint（参数提示）

在自动补全时显示的参数占位符：

```yaml
---
name: fix-issue
description: 修复指定的 GitHub Issue
argument-hint: <issue-number>
---
```

当用户输入 `/fix-issue` 时，会看到提示：`/fix-issue <issue-number>`

## 参数传递

Skills 支持灵活的参数传递机制，让你创建通用的、可配置的指令。

### $ARGUMENTS：完整参数字符串

`$ARGUMENTS` 会被替换为用户输入的完整参数：

```yaml
---
name: fix-issue
description: 修复 GitHub Issue
disable-model-invocation: true
---

修复 GitHub Issue #$ARGUMENTS：

1. 使用 `gh issue view $ARGUMENTS` 读取 Issue 详情
2. 分析问题原因
3. 实现修复方案
4. 编写测试用例
5. 创建符合规范的 commit
```

使用方式：

```
> /fix-issue 42
```

Claude 收到的指令将是 "修复 GitHub Issue #42"。

### $0, $1, $2...：位置参数

当需要传递多个参数时，使用位置参数：

```yaml
---
name: migrate-component
description: 将组件从一个框架迁移到另一个
argument-hint: <组件名> <源框架> <目标框架>
---

将 $0 组件从 $1 迁移到 $2。

迁移要求：
1. 保留所有现有功能和行为
2. 更新导入语句和依赖
3. 适配目标框架的生命周期方法
4. 确保所有测试通过
```

使用方式：

```
> /migrate-component SearchBar React Vue
```

解析结果：
- `$0` = `SearchBar`
- `$1` = `React`
- `$2` = `Vue`

### $ARGUMENTS[N]：等价写法

`$ARGUMENTS[0]` 等价于 `$0`，`$ARGUMENTS[1]` 等价于 `$1`，以此类推：

```yaml
---
name: compare-files
description: 比较两个文件的差异
argument-hint: <文件A> <文件B>
---

比较以下两个文件的差异：
- 文件 A：$ARGUMENTS[0]
- 文件 B：$ARGUMENTS[1]

分析差异并给出合并建议。
```

## 辅助文件

Skill 目录可以包含多个文件，不仅限于 `SKILL.md`。辅助文件可以提供模板、示例、配置或脚本。

### 目录结构

```
my-skill/
├── SKILL.md           # 主指令文件（必需）
├── template.md        # 模板文件
├── examples/
│   ├── good.md        # 正确示例
│   └── bad.md         # 错误示例
├── config/
│   └── rules.yaml     # 规则配置
└── scripts/
    └── validate.sh    # 辅助脚本
```

### 在 SKILL.md 中引用辅助文件

```yaml
---
name: generate-api-doc
description: 为 API 端点生成标准文档
---

根据以下模板为 API 端点生成文档。

## 文档模板
参考 [template.md](template.md) 中的格式。

## 示例
- 正确格式见 [examples/good.md](examples/good.md)
- 常见错误见 [examples/bad.md](examples/bad.md)

## 步骤
1. 读取目标 API 文件
2. 提取路由、参数、返回值信息
3. 按照模板格式生成文档
4. 包含请求和响应示例
```

### 实用示例：代码审查 Skill 带检查清单

```
review-checklist/
├── SKILL.md
└── checklist.md
```

**checklist.md：**

```markdown
## 代码审查检查清单

### 安全性
- [ ] 输入验证是否充分
- [ ] SQL 查询是否使用参数化
- [ ] 敏感数据是否加密
- [ ] 认证和授权是否正确

### 性能
- [ ] 是否有不必要的数据库查询
- [ ] 循环中是否有 O(n^2) 复杂度
- [ ] 是否正确使用缓存
- [ ] 大数据集是否分页处理

### 代码质量
- [ ] 命名是否清晰
- [ ] 函数是否单一职责
- [ ] 是否有重复代码
- [ ] 错误处理是否完善
```

## 动态上下文注入

动态上下文注入是 Skills 最强大的功能之一。使用 `` !`command` `` 语法，可以在 Skill 发送给 Claude **之前**执行 shell 命令，并将输出嵌入到指令中。

### 基本语法

```yaml
---
name: pr-review
description: 审查当前 PR
context: fork
---

## 当前 PR 信息
- PR 描述: !`gh pr view --json title,body --jq '.title + "\n" + .body'`
- 变更文件: !`gh pr diff --name-only`
- PR Diff: !`gh pr diff`

## 任务
请根据以上信息审查这个 PR，关注：
1. 代码质量和可读性
2. 潜在的 bug
3. 是否符合项目规范
```

当你调用 `/pr-review` 时：
1. Claude Code 首先执行三个 `gh` 命令
2. 将命令输出替换到对应位置
3. Claude 收到的是包含实际 PR 数据的完整提示

### 注入项目上下文

```yaml
---
name: project-status
description: 分析项目当前状态
---

## 项目信息
- Git 状态: !`git status --short`
- 最近提交: !`git log --oneline -5`
- 未解决 TODO: !`grep -r "TODO" --include="*.ts" -l 2>/dev/null | head -10`
- 测试覆盖率: !`npm test -- --coverage --silent 2>/dev/null | tail -5`

## 分析任务
基于以上信息，分析项目的当前状态，指出需要关注的问题。
```

### 注入环境信息

```yaml
---
name: env-check
description: 检查开发环境配置
---

## 环境信息
- Node.js 版本: !`node --version 2>/dev/null || echo "未安装"`
- npm 版本: !`npm --version 2>/dev/null || echo "未安装"`
- Python 版本: !`python3 --version 2>/dev/null || echo "未安装"`
- Git 版本: !`git --version`
- 操作系统: !`uname -a`

## 检查清单
验证以上环境配置是否满足项目要求。
```

### 注意事项

1. **命令必须快速执行**：避免耗时长的命令，它们会延迟 Skill 的加载
2. **处理命令失败**：使用 `|| echo "默认值"` 处理命令执行失败的情况
3. **避免敏感信息**：不要在注入命令中输出密码、密钥等
4. **输出大小**：注意命令输出不宜过大，会占用上下文窗口

## 完整实战示例

### 示例一：代码重构 Skill

```yaml
---
name: refactor
description: 重构指定的代码文件，提升可读性和可维护性
argument-hint: <文件路径>
---

对 $ARGUMENTS 进行代码重构：

## 分析阶段
1. 读取文件内容，理解功能
2. 识别代码异味（Code Smells）

## 重构步骤
1. 提取重复逻辑为独立函数
2. 改善命名（变量、函数、类）
3. 简化复杂的条件表达式
4. 应用单一职责原则
5. 添加或更新注释

## 验证
1. 确保功能行为不变
2. 运行相关测试
3. 检查是否引入了新的依赖
```

### 示例二：数据库迁移 Skill

```yaml
---
name: create-migration
description: 创建数据库迁移文件
argument-hint: <迁移描述>
disable-model-invocation: true
---

## 当前数据库状态
- 已有迁移文件: !`ls -1 migrations/ 2>/dev/null | tail -5`
- 数据库表结构: !`cat prisma/schema.prisma 2>/dev/null | head -50`

## 任务
创建一个新的数据库迁移：$ARGUMENTS

要求：
1. 遵循现有迁移文件的命名和格式规范
2. 包含 up 和 down 操作
3. 考虑数据迁移（不仅是结构变更）
4. 添加必要的索引
5. 编写迁移测试
```

### 示例三：Release Notes 生成器

```yaml
---
name: release-notes
description: 生成版本发布说明
argument-hint: <版本标签>
context: fork
---

## 变更信息
- 自上个版本以来的提交: !`git log $(git describe --tags --abbrev=0 2>/dev/null || echo "HEAD~20")..HEAD --oneline`
- 变更的文件: !`git diff $(git describe --tags --abbrev=0 2>/dev/null || echo "HEAD~20")..HEAD --stat`

## 任务
为版本 $ARGUMENTS 生成 Release Notes：

1. 将提交分类为：新功能、Bug 修复、性能改进、文档更新、破坏性变更
2. 用清晰的语言描述每个变更的用户影响
3. 标注破坏性变更和迁移步骤
4. 生成 Markdown 格式的 Release Notes
```

---

## 关键要点

- 创建 Skill 需要三步：创建目录、编写 SKILL.md、测试调用
- Frontmatter 核心字段：`name`（标识符）、`description`（触发描述）、`argument-hint`（参数提示）
- 参数传递：`$ARGUMENTS` 获取完整参数，`$0`、`$1` 获取位置参数
- 辅助文件：Skill 目录可以包含模板、示例、脚本等支持文件
- 动态上下文注入（`` !`command` ``）可以在运行时嵌入 shell 命令输出
- 动态注入的命令应快速执行，并处理失败情况

---

[← 上一课：什么是 Skills](./01-what-are-skills.md) | [返回目录](./README.md) | [下一课：配置与控制 →](./03-configuring-skills.md)
