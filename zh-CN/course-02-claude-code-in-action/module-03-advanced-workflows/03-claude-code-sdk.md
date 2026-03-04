# 课时 12：Claude Code SDK

[← 上一课：Hooks 系统](./02-hooks.md) | [返回目录](../README.md) | [下一课：课程总结 →](./04-course-summary.md)

---

## 核心概念

到目前为止，我们都是通过交互式终端使用 Claude Code。但 Claude Code 也提供了 **SDK（软件开发工具包）**，让你能够以编程方式将 Claude Code 的能力嵌入到你自己的工具和脚本中。

## 什么是 Claude Code SDK？

Claude Code SDK 是 `@anthropic-ai/claude-code` npm 包，提供了：

- **编程式访问**：在 JavaScript/TypeScript 代码中调用 Claude Code
- **自定义集成**：将 Claude Code 嵌入你的 CLI 工具、自动化脚本、CI/CD 管道
- **Agent 编排**：创建多 Agent 工作流，协调多个 Claude 实例

```
┌──────────────────────────┐
│     你的自定义工具         │
│                          │
│   ┌──────────────────┐   │
│   │  Claude Code SDK │   │
│   │                  │   │
│   │  ┌────────────┐  │   │
│   │  │ Claude Code │  │   │
│   │  │   Engine    │  │   │
│   │  └────────────┘  │   │
│   └──────────────────┘   │
│                          │
│   自定义逻辑 + 编排      │
└──────────────────────────┘
```

## 安装

```bash
npm install @anthropic-ai/claude-code
```

## 基本使用

### 简单调用

```typescript
import { claude } from "@anthropic-ai/claude-code";

// 基本使用：发送一个提示并获取回复
const result = await claude({
  prompt: "解释这个项目的架构",
  workingDirectory: "/path/to/project",
});

console.log(result.text);
```

### 非交互式模式

SDK 默认以非交互式模式运行，类似于 CLI 的 `-p` 标志：

```typescript
const result = await claude({
  prompt: "为 src/utils/math.ts 编写单元测试",
  workingDirectory: "/path/to/project",
  options: {
    maxTurns: 10,  // 限制最大执行轮次
  },
});
```

### 流式输出

```typescript
import { claude } from "@anthropic-ai/claude-code";

const stream = claude({
  prompt: "重构 auth 模块",
  workingDirectory: "/path/to/project",
  stream: true,
});

for await (const event of stream) {
  if (event.type === "text") {
    process.stdout.write(event.text);
  } else if (event.type === "tool_use") {
    console.log(`使用工具: ${event.name}`);
  }
}
```

## CLI 管道模式

除了 SDK，你也可以通过 CLI 的管道模式使用 Claude Code：

```bash
# 基本管道
echo "分析这个文件的性能" | claude -p

# 传入文件内容
cat error.log | claude -p "分析这个错误日志，找出根因"

# 链式操作
git diff main --name-only | claude -p "审查这些变更文件的安全问题"

# 监控日志
tail -f app.log | claude -p "发现异常时发送 Slack 通知"
```

## 子 Agent（Sub-agents）

Claude Code SDK 支持创建多个子 Agent 来并行处理任务：

### 概念

```
┌─────────────────────────────────┐
│          主 Agent（协调者）       │
│                                 │
│   ┌──────┐ ┌──────┐ ┌──────┐  │
│   │子Agent│ │子Agent│ │子Agent│  │
│   │前端修复│ │后端修复│ │测试编写│  │
│   └──────┘ └──────┘ └──────┘  │
│                                 │
│   收集结果 → 整合 → 报告        │
└─────────────────────────────────┘
```

### 使用示例

```typescript
import { claude } from "@anthropic-ai/claude-code";

// 启动多个子 Agent 并行工作
const [frontendResult, backendResult, testResult] = await Promise.all([
  claude({
    prompt: "修复前端的样式问题",
    workingDirectory: "/path/to/project",
  }),
  claude({
    prompt: "优化后端 API 的响应时间",
    workingDirectory: "/path/to/project",
  }),
  claude({
    prompt: "为新功能编写集成测试",
    workingDirectory: "/path/to/project",
  }),
]);

console.log("前端:", frontendResult.text);
console.log("后端:", backendResult.text);
console.log("测试:", testResult.text);
```

## 实际应用场景

### 场景一：自定义 Code Review 工具

```typescript
import { claude } from "@anthropic-ai/claude-code";

async function reviewPR(prNumber: number) {
  // 获取 PR diff
  const diff = execSync(`gh pr diff ${prNumber}`).toString();

  // 让 Claude 审查
  const result = await claude({
    prompt: `审查这个 PR 的代码变更，关注安全性和性能：\n\n${diff}`,
    workingDirectory: process.cwd(),
  });

  // 发布审查评论
  execSync(`gh pr comment ${prNumber} --body "${result.text}"`);
}
```

### 场景二：自动化 Bug 修复流水线

```typescript
async function autofixBug(issueNumber: number) {
  // 1. 读取 Issue
  const issue = execSync(`gh issue view ${issueNumber} --json body`);

  // 2. 让 Claude 分析和修复
  const result = await claude({
    prompt: `分析并修复这个 Bug：${issue}`,
    workingDirectory: process.cwd(),
    options: { maxTurns: 20 },
  });

  // 3. 创建 PR
  execSync(`git checkout -b fix/issue-${issueNumber}`);
  execSync(`git add -A && git commit -m "fix: resolve #${issueNumber}"`);
  execSync(`gh pr create --title "Fix #${issueNumber}" --body "${result.text}"`);
}
```

### 场景三：批量代码迁移

```typescript
import { claude } from "@anthropic-ai/claude-code";
import { glob } from "glob";

async function migrateFiles(pattern: string, instruction: string) {
  const files = glob.sync(pattern);

  // 并行处理所有文件
  const results = await Promise.all(
    files.map((file) =>
      claude({
        prompt: `${instruction}\n\n文件: ${file}`,
        workingDirectory: process.cwd(),
      })
    )
  );

  console.log(`已处理 ${results.length} 个文件`);
}

// 使用
migrateFiles("src/**/*.jsx", "将这个 JSX 组件转换为 TypeScript TSX");
```

## Agent SDK

对于更高级的自定义 Agent 开发，Anthropic 还提供了 **Agent SDK**，允许你构建完全自定义的 Agent，拥有对编排、工具访问和权限的完全控制。

Agent SDK 适用于：
- 需要完全自定义工具集的场景
- 复杂的多 Agent 协作系统
- 特定领域的专业 Agent

更多信息请参阅 [Agent SDK 文档](https://platform.claude.com/docs/en/agent-sdk/overview)。

## 最佳实践

### 1. 设置合理的限制

```typescript
const result = await claude({
  prompt: "...",
  options: {
    maxTurns: 15,     // 限制执行轮次，防止无限循环
  },
});
```

### 2. 错误处理

```typescript
try {
  const result = await claude({
    prompt: "重构 auth 模块",
    workingDirectory: "/path/to/project",
  });
} catch (error) {
  console.error("Claude Code 执行失败:", error);
  // 实现重试逻辑或降级方案
}
```

### 3. 结合 CLAUDE.md

SDK 调用也会读取项目的 `CLAUDE.md`，确保你的项目级指令在编程式调用中也生效。

## 实用技巧

1. **从 CLI 管道开始**：先用 `claude -p` 验证想法，再转成 SDK 代码
2. **设置 `maxTurns`**：防止 Claude 在复杂任务上无限循环
3. **善用并行**：多个独立任务可以使用 `Promise.all` 并行执行
4. **日志记录**：在生产环境中记录每次 SDK 调用的输入输出
5. **幂等性**：确保你的自动化脚本可以安全地重复执行

---

## 关键要点

- Claude Code SDK（`@anthropic-ai/claude-code`）允许编程式地使用 Claude Code
- CLI 管道模式（`claude -p`）是最简单的自动化方式
- SDK 支持流式输出、子 Agent 并行、自定义工具限制
- 实际应用：自动化 Code Review、Bug 修复流水线、批量代码迁移
- Agent SDK 提供更底层的自定义 Agent 开发能力
- SDK 调用也会读取项目的 `CLAUDE.md`，保持一致的行为

---

[← 上一课：Hooks 系统](./02-hooks.md) | [返回目录](../README.md) | [下一课：课程总结 →](./04-course-summary.md)
