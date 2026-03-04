# 课时 3：配置与控制

[← 上一课：创建 Skills](./02-creating-skills.md) | [返回目录](./README.md) | [下一课：分享与最佳实践 →](./04-sharing-skills.md)

---

## 核心概念

Skills 的配置选项让你能够精确控制 Skill 的调用方式、执行环境和可用工具。本课时将详细讲解每个配置选项的含义和使用场景，以及 Claude Code 内置的三个 Skills。

## 调用权限控制

Claude Code 的 Skills 支持两种调用方式：用户手动调用和 Claude 自动调用。通过配置，你可以精确控制这两种调用方式的开关。

### 默认行为

如果不添加任何权限配置，Skill 同时支持用户手动调用和 Claude 自动调用：

```yaml
---
name: explain-code
description: 解释代码的功能和逻辑
---

详细解释指定代码的功能...
```

此 Skill 既可以通过 `/explain-code` 调用，也会在用户说"帮我解释这段代码"时自动触发。

### disable-model-invocation：禁止 Claude 自动调用

对于有副作用的操作（部署、发送消息、删除数据等），你不希望 Claude 自己决定何时运行：

```yaml
---
name: deploy-prod
description: 将应用部署到生产环境
disable-model-invocation: true
---

部署到生产环境：
1. 运行完整测试套件
2. 构建生产版本
3. 推送到生产服务器
4. 验证部署状态
```

设置 `disable-model-invocation: true` 后：
- 用户可以通过 `/deploy-prod` 手动调用
- Claude **不会**自动触发此 Skill（即使上下文匹配）

**适用场景：**
- 生产环境部署
- 发送邮件或通知
- 删除数据或文件
- 修改系统配置
- 任何不可逆的操作

### user-invocable：禁止用户手动调用

某些 Skills 作为背景知识存在，由 Claude 在需要时自动加载，但作为命令直接调用没有意义：

```yaml
---
name: coding-standards
description: 项目编码标准和规范。当 Claude 编写或审查代码时自动参考。
user-invocable: false
---

## 编码标准

### 命名规范
- 变量和函数：camelCase
- 类和接口：PascalCase
- 常量：UPPER_SNAKE_CASE
- 文件名：kebab-case

### 代码风格
- 使用 TypeScript 严格模式
- 所有函数必须有返回类型注解
- 优先使用 const 声明
- 避免 any 类型
```

设置 `user-invocable: false` 后：
- 此 Skill **不会**出现在 `/` 命令菜单中
- Claude 会在编写或审查代码时**自动**参考这些标准

**适用场景：**
- 编码规范和标准
- 架构指南
- API 设计原则
- 项目约定文档

### 权限配置总结

| 配置 | 用户可调用 | Claude 可自动调用 | 适用场景 |
|------|-----------|-----------------|----------|
| 默认（无配置） | 是 | 是 | 通用 Skill |
| `disable-model-invocation: true` | 是 | 否 | 有副作用的操作 |
| `user-invocable: false` | 否 | 是 | 背景知识 |

## allowed-tools：工具限制

通过 `allowed-tools` 字段，你可以限制 Skill 执行时可以使用的工具集：

```yaml
---
name: read-only-review
description: 只读模式的代码审查
allowed-tools: Read, Grep, Glob
---

以只读模式审查代码，不做任何修改：

1. 读取指定文件
2. 搜索相关模式
3. 分析代码质量
4. 输出审查报告
```

此 Skill 只能使用 `Read`、`Grep` 和 `Glob` 工具，无法使用 `Write`、`Edit` 或 `Bash` 等可能修改文件的工具。

**常见工具限制场景：**

| 场景 | 推荐工具集 |
|------|-----------|
| 只读审查 | `Read, Grep, Glob` |
| 文件编辑 | `Read, Edit, Write, Grep, Glob` |
| 完整操作 | 不设置限制（默认所有工具） |

## context: fork — 子 Agent 执行

`context: fork` 让 Skill 在独立的子 Agent 中执行，不影响主对话的上下文窗口：

```yaml
---
name: analyze-repo
description: 全面分析代码仓库
context: fork
---

对整个代码仓库进行全面分析：

1. 扫描目录结构
2. 统计代码行数和语言分布
3. 分析依赖关系
4. 识别技术债务
5. 生成分析报告
```

### 为什么使用 context: fork？

| 特性 | 主对话 | fork（子 Agent） |
|------|--------|-----------------|
| 上下文影响 | 占用主对话上下文 | 独立上下文，不影响主对话 |
| 适用任务 | 短小交互 | 长时间运行、大量读取 |
| 结果返回 | 直接展示 | 将摘要返回主对话 |

**适用场景：**
- 需要读取大量文件的分析任务
- 长时间运行的操作
- 不想污染主对话上下文的任务

### agent 字段

与 `context: fork` 搭配使用，指定子 Agent 的类型：

```yaml
---
name: explore-codebase
description: 探索和理解代码库结构
context: fork
agent: Explore
---

探索代码库并生成架构概览...
```

## 完整配置示例

### 安全审计 Skill

```yaml
---
name: security-audit
description: 对代码进行安全审计
allowed-tools: Read, Grep, Glob
context: fork
agent: Explore
---

执行全面的安全审计：

1. **依赖检查**
   - 扫描 package.json/requirements.txt 中的已知漏洞
   - 检查过期的依赖

2. **代码扫描**
   - 硬编码的密钥和凭证
   - SQL 注入风险
   - XSS 漏洞
   - 不安全的反序列化
   - 路径遍历风险

3. **配置检查**
   - CORS 配置
   - HTTPS 强制
   - 安全头部设置
   - 环境变量使用

4. **输出报告**
   - 按严重程度分类
   - 提供修复建议
   - 引用 OWASP Top 10
```

### CI/CD 辅助 Skill

```yaml
---
name: ci-fix
description: 分析并修复 CI/CD 构建失败
disable-model-invocation: true
argument-hint: <pipeline-url>
---

## CI/CD 信息
- 最近构建状态: !`gh run list --limit 3 --json status,name,conclusion --jq '.[] | .name + ": " + .conclusion'`
- 失败日志: !`gh run list --limit 1 --status failure --json databaseId --jq '.[0].databaseId' | xargs -I{} gh run view {} --log-failed 2>/dev/null | tail -30`

## 任务
分析 CI/CD 失败原因并修复：

1. 阅读失败日志
2. 识别根本原因
3. 实施修复
4. 本地验证
5. 推送修复代码
```

## 内置 Skills 详解

Claude Code 提供了三个内置 Skills，覆盖常见的开发场景。

### /simplify

审查最近修改的代码，检查可复用性、代码质量和效率问题，然后自动修复：

```
> /simplify
```

**工作流程：**
1. 识别最近修改的文件
2. 分析代码复杂度
3. 查找重复逻辑
4. 简化复杂表达式
5. 应用修复

**适用场景：**
- 完成一轮开发后快速清理代码
- 减少代码复杂度
- 提取公共逻辑

### /batch

在代码库中编排大规模并行变更：

```
> /batch 将所有 console.log 替换为 logger.info
```

**工作流程：**
1. 分析代码库，找到所有匹配的位置
2. 将变更分组为可并行执行的批次
3. 逐批执行变更
4. 验证每批变更的正确性

**适用场景：**
- 大规模重命名
- 批量更新 API 调用
- 统一代码风格
- 迁移废弃的 API

### /debug

通过读取调试日志来排查 Claude Code 会话问题：

```
> /debug 为什么上一个命令执行失败了
```

**工作流程：**
1. 读取 Claude Code 的调试日志
2. 分析错误信息和上下文
3. 识别问题原因
4. 提供解决建议

**适用场景：**
- 排查 Claude Code 异常行为
- 诊断工具执行失败
- 理解 Claude 的决策过程

## 配置组合策略

不同场景下的推荐配置组合：

| 场景 | disable-model-invocation | user-invocable | allowed-tools | context |
|------|------------------------|----------------|---------------|---------|
| 代码审查 | - | - | Read, Grep | fork |
| 部署操作 | true | - | 不限 | - |
| 编码标准 | - | false | - | - |
| 大规模分析 | - | - | Read, Grep, Glob | fork |
| 危险操作 | true | - | 限定工具 | - |

---

## 关键要点

- `disable-model-invocation: true` 防止 Claude 自动调用有副作用的 Skill
- `user-invocable: false` 让 Skill 仅作为 Claude 自动参考的背景知识
- `allowed-tools` 可以限制 Skill 可用的工具集，增强安全性
- `context: fork` 让 Skill 在独立子 Agent 中运行，不污染主对话上下文
- 内置 Skills（`/simplify`、`/batch`、`/debug`）覆盖了代码简化、批量变更和调试排查
- 根据 Skill 的用途选择合适的配置组合

---

[← 上一课：创建 Skills](./02-creating-skills.md) | [返回目录](./README.md) | [下一课：分享与最佳实践 →](./04-sharing-skills.md)
