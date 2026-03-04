# 课时 4：分享与最佳实践

[← 上一课：配置与控制](./03-configuring-skills.md) | [返回目录](./README.md)

---

## 核心概念

Skills 不仅是个人生产力工具，更是团队协作和知识共享的载体。本课时将讲解如何通过版本控制分享 Skills、如何作为插件分发、团队协作模式，以及常见问题的排查方法。

## 提交到版本控制

将 Skills 提交到项目仓库是最直接的团队共享方式。

### 项目级 Skills 的 Git 管理

项目级 Skills 存储在 `.claude/skills/` 目录，可以直接提交到 Git：

```bash
# 查看 Skills 目录
ls -la .claude/skills/

# 添加到 Git
git add .claude/skills/
git commit -m "feat: add team skills for code review and deployment"
```

### .gitignore 配置

确保 `.claude/skills/` 不在 `.gitignore` 中：

```bash
# .gitignore
# Claude Code 配置
.claude/settings.json      # 个人设置（通常不提交）
# .claude/skills/          # Skills 应该提交！不要忽略！
```

**建议提交的文件：**
- `.claude/skills/` 目录及所有 Skill 文件
- 项目级 `CLAUDE.md`

**建议不提交的文件：**
- `.claude/settings.json`（包含个人偏好设置）
- 包含个人凭证的 Skill 文件

### 推荐的目录结构

```
project-root/
├── .claude/
│   ├── skills/
│   │   ├── review-code/
│   │   │   ├── SKILL.md
│   │   │   └── checklist.md
│   │   ├── deploy/
│   │   │   └── SKILL.md
│   │   ├── create-pr/
│   │   │   └── SKILL.md
│   │   └── fix-issue/
│   │       └── SKILL.md
│   └── settings.json        # 个人设置，加入 .gitignore
├── CLAUDE.md                 # 项目上下文
└── src/
    └── ...
```

## 插件分发

Skills 可以作为独立的"插件"分发，供其他项目或团队使用。

### 创建可分发的 Skill 包

一个高质量的可分发 Skill 应该包含：

```
my-awesome-skill/
├── SKILL.md              # 主指令文件
├── README.md             # 使用文档
├── examples/             # 示例目录
│   ├── basic-usage.md    # 基本使用示例
│   └── advanced.md       # 高级使用示例
└── templates/            # 模板文件
    └── output-format.md  # 输出格式模板
```

### 通过 Git 仓库分发

将 Skill 发布为独立的 Git 仓库：

```bash
# 安装其他人的 Skill
cd your-project
git clone https://github.com/user/skill-name .claude/skills/skill-name

# 或者使用 git submodule
git submodule add https://github.com/user/skill-name .claude/skills/skill-name
```

### 通过复制分发

最简单的方式，直接复制文件：

```bash
# 从其他项目复制 Skill
cp -r /path/to/other-project/.claude/skills/useful-skill .claude/skills/

# 从网络下载
curl -o .claude/skills/skill-name/SKILL.md https://example.com/skill.md
```

## 团队共享模式

### 模式一：项目仓库集中管理

适合团队共同维护的项目级 Skills：

```
流程：
1. 团队成员创建 Skill
2. 提交 PR 到项目仓库
3. 团队审核 Skill 内容
4. 合并后所有成员自动获取
```

**优点：**
- 版本控制，有审核流程
- 所有成员自动同步
- 与项目代码一起管理

### 模式二：共享 Skills 仓库

创建一个专门的 Skills 仓库，集中管理团队通用的 Skills：

```bash
# 团队 Skills 仓库结构
team-skills/
├── code-review/
│   └── SKILL.md
├── commit-message/
│   └── SKILL.md
├── deploy-staging/
│   └── SKILL.md
├── deploy-production/
│   └── SKILL.md
├── create-pr/
│   └── SKILL.md
└── README.md
```

团队成员可以将需要的 Skills 复制或链接到个人目录：

```bash
# 方法一：符号链接（推荐）
ln -s /path/to/team-skills/code-review ~/.claude/skills/code-review

# 方法二：直接克隆
git clone https://github.com/team/skills ~/.claude/team-skills
ln -s ~/.claude/team-skills/code-review ~/.claude/skills/code-review
```

### 模式三：企业级统一部署

由管理员统一部署的 Skills，自动对组织内所有用户生效：

```
特点：
- 管理员集中管理
- 自动分发给所有用户
- 适合合规性和标准化要求
- 用户无需手动安装
```

## 编写高质量 Skills 的最佳实践

### 1. 描述要精准

```yaml
# 不好的描述
---
name: helper
description: 帮助处理代码
---

# 好的描述
---
name: optimize-query
description: 分析并优化 SQL 查询性能。当用户提到数据库查询慢、SQL 优化或查询性能时使用。
---
```

### 2. 指令要具体

```yaml
# 不好的指令
---
name: test
description: 编写测试
---
为代码编写测试。

# 好的指令
---
name: write-tests
description: 为指定函数编写单元测试
argument-hint: <文件路径>
---

为 $ARGUMENTS 中的函数编写单元测试：

1. 读取源文件，理解每个导出函数的功能
2. 使用项目已有的测试框架（检查 package.json）
3. 为每个函数编写：
   - 正常输入的测试用例
   - 边界条件的测试用例
   - 错误输入的测试用例
4. 使用描述性的测试名称
5. 确保测试可以独立运行
6. 运行测试确认全部通过
```

### 3. 控制 Skills 数量

每个 Skill 的 `description` 会占用 Claude 的上下文空间。过多的 Skills 会降低 Claude 的效率。

**建议：**
- 项目级 Skills：5-10 个
- 个人级 Skills：10-15 个
- 定期清理不再使用的 Skills

### 4. 合理使用 context: fork

```yaml
# 需要大量读取的 Skill，使用 fork
---
name: codebase-analysis
description: 分析整个代码库
context: fork
---

# 简单快速的 Skill，不需要 fork
---
name: format-code
description: 格式化当前文件
---
```

### 5. 处理错误情况

在 Skill 指令中明确错误处理：

```yaml
---
name: deploy
description: 部署应用
disable-model-invocation: true
---

部署应用：

1. 运行测试 - 如果测试失败，停止部署并报告失败原因
2. 构建应用 - 如果构建失败，检查错误日志并提供修复建议
3. 推送部署 - 如果部署失败，回滚并通知
4. 健康检查 - 如果检查失败，回滚到上一版本
```

### 6. 保持幂等性

Skill 的执行应该是幂等的，多次运行不会产生意外副作用：

```yaml
# 好的实践：检查状态再操作
---
name: setup-project
description: 初始化项目配置
disable-model-invocation: true
---

初始化项目：
1. 检查是否已存在配置文件，如果存在则跳过
2. 检查依赖是否已安装，如果已安装则跳过
3. 仅创建缺失的配置
4. 报告实际执行了哪些操作
```

## 故障排查

### 问题一：Skill 没有被加载

**症状：** 输入 `/skill-name` 没有响应或显示"未找到"

**排查步骤：**

```bash
# 1. 确认文件存在
ls -la .claude/skills/skill-name/SKILL.md

# 2. 检查文件权限
stat .claude/skills/skill-name/SKILL.md

# 3. 验证 YAML frontmatter 格式
head -10 .claude/skills/skill-name/SKILL.md
```

**常见原因：**
- 文件名不是 `SKILL.md`（大小写敏感）
- YAML frontmatter 格式错误（缺少 `---` 分隔符）
- 文件权限不足

### 问题二：Skill 没有自动触发

**症状：** 期望 Claude 自动使用 Skill，但没有触发

**排查步骤：**

1. 检查 `description` 是否包含触发关键词
2. 确认没有设置 `disable-model-invocation: true`
3. 检查是否有太多 Skills 导致匹配不准确

**解决方案：**
```yaml
# 优化 description，添加更多触发词
---
name: review-code
description: >
  对代码进行审查。当用户要求 code review、代码审查、
  检查代码质量、review PR 时使用。
---
```

### 问题三：动态上下文注入失败

**症状：** `` !`command` `` 的输出为空或报错

**排查步骤：**

```bash
# 1. 手动测试命令
gh pr diff --name-only

# 2. 检查命令是否在当前目录有效
pwd
git status

# 3. 添加错误处理
```

**解决方案：**

```yaml
# 添加默认值处理
- PR diff: !`gh pr diff 2>/dev/null || echo "没有活跃的 PR"`
- 最近提交: !`git log --oneline -5 2>/dev/null || echo "无 Git 历史"`
```

### 问题四：Skill 执行超出预期

**症状：** Skill 修改了不应该修改的文件或执行了意外操作

**解决方案：**

```yaml
# 1. 限制可用工具
---
allowed-tools: Read, Grep, Glob
---

# 2. 使用 fork 隔离
---
context: fork
---

# 3. 在指令中明确约束
不要修改 src/core/ 目录下的任何文件。
仅修改测试文件（*.test.ts, *.spec.ts）。
```

### 问题五：Skills 之间冲突

**症状：** 多个 Skills 被同时触发，或者 Claude 选择了错误的 Skill

**解决方案：**
- 让每个 Skill 的 `description` 更加具体，减少重叠
- 对于容易混淆的 Skills，使用 `disable-model-invocation: true` 改为手动调用
- 减少 Skills 总数，只保留高频使用的

---

## 关键要点

- 将项目级 Skills 提交到版本控制，确保 `.claude/skills/` 不在 `.gitignore` 中
- 插件分发可以通过 Git 仓库、子模块或直接复制实现
- 团队共享推荐三种模式：项目仓库集中管理、共享 Skills 仓库、企业级统一部署
- 编写高质量 Skills 的要点：精准描述、具体指令、控制数量、处理错误
- 常见问题多数由文件路径、YAML 格式或 description 不准确导致
- 使用 `allowed-tools` 和 `context: fork` 防止 Skill 执行超出预期

---

[← 上一课：配置与控制](./03-configuring-skills.md) | [返回目录](./README.md)
