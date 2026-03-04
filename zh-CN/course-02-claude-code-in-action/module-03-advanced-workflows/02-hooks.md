# 课时 11：Hooks 系统

[← 上一课：计划模式与思考模式](./01-plan-mode-and-thinking-mode.md) | [返回目录](../README.md) | [下一课：Claude Code SDK →](./03-claude-code-sdk.md)

---

## 核心概念

**Hooks** 是用户定义的命令，在 Claude Code 生命周期的特定时刻自动执行。类似于 Git Hooks 或面向切面编程（AOP），Hooks 让你能够在 Claude 使用工具的前后注入自定义行为，**保证**特定操作总会执行，而不依赖 AI 的判断。

## 为什么需要 Hooks？

Claude Code 本身已经很强大，但有些事情你希望**确保一定发生**：

- **每次编辑文件后自动格式化**（不管 Claude 记不记得）
- **禁止 Claude 修改 `.env` 文件**（即使它认为有必要）
- **每次文件变更都记录审计日志**
- **提交前自动运行 lint 检查**

Hooks 把这些保障从"Claude 应该记得做"变成"系统会确保做"。

## Hook 生命周期事件

Hooks 在 Claude Code 的不同阶段触发：

| 事件 | 触发时机 | 常见用途 |
|------|---------|---------|
| `PreToolUse` | 工具执行**之前** | 验证参数、阻止操作、权限检查 |
| `PostToolUse` | 工具执行**之后** | 自动格式化、日志记录、通知 |
| `Notification` | Claude 发送通知时 | 转发通知到其他系统 |
| `Stop` | Claude 完成回复时 | 后处理、总结、验证 |
| `SubagentStop` | 子 Agent 完成时 | 收集子任务结果 |

### 工具匹配

Hook 可以针对特定工具触发：

```
PreToolUse 事件可以匹配：
- Write（创建文件时）
- Edit（编辑文件时）
- Bash（执行命令时）
- 特定 MCP 工具名称
```

## 配置 Hooks

Hooks 在 Claude Code 的设置文件中配置。

### 基本格式

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "echo '即将修改文件: $CLAUDE_TOOL_INPUT'"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

### Hook 类型

Claude Code 支持多种 Hook 类型：

#### 1. Command Hook（命令钩子）

执行 shell 命令：

```json
{
  "type": "command",
  "command": "npx prettier --write $CLAUDE_FILE_PATH"
}
```

#### 2. HTTP Hook（HTTP 钩子）

发送 HTTP 请求：

```json
{
  "type": "http",
  "url": "https://your-webhook.example.com/hook",
  "method": "POST"
}
```

#### 3. Prompt Hook（提示钩子）

让 LLM 评估并做出决策：

```json
{
  "type": "prompt",
  "prompt": "检查这个代码修改是否遵循了安全最佳实践"
}
```

## 实用 Hook 示例

### 示例一：自动格式化

每次 Claude 编辑文件后，自动运行代码格式化：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write $CLAUDE_FILE_PATH 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### 示例二：文件访问控制

阻止 Claude 修改敏感文件：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "if echo $CLAUDE_FILE_PATH | grep -qE '\\.(env|pem|key)$'; then echo 'BLOCK: 禁止修改敏感文件' >&2; exit 2; fi"
          }
        ]
      }
    ]
  }
}
```

当 Hook 返回退出码 2 时，操作会被**阻止**，Claude 会收到阻止消息。

### 示例三：审计日志

记录 Claude 的所有文件操作：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write|Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo \"$(date -Iseconds) | Tool: $CLAUDE_TOOL_NAME | File: $CLAUDE_FILE_PATH\" >> ~/.claude/audit.log"
          }
        ]
      }
    ]
  }
}
```

### 示例四：自动运行测试

编辑测试文件后自动运行对应测试：

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit",
        "hooks": [
          {
            "type": "command",
            "command": "if echo $CLAUDE_FILE_PATH | grep -q '\\.test\\.'; then npm test -- $CLAUDE_FILE_PATH 2>&1 | tail -5; fi"
          }
        ]
      }
    ]
  }
}
```

## Hook 输入输出

### 输入

对于命令类型的 Hook，Claude Code 通过 **stdin** 传递 JSON 上下文信息：

```json
{
  "tool_name": "Edit",
  "tool_input": {
    "file_path": "/path/to/file.ts",
    "old_string": "...",
    "new_string": "..."
  },
  "session_id": "abc123"
}
```

同时通过环境变量提供快捷访问：

| 环境变量 | 说明 |
|---------|------|
| `CLAUDE_TOOL_NAME` | 当前工具名称 |
| `CLAUDE_TOOL_INPUT` | 工具输入（JSON） |
| `CLAUDE_FILE_PATH` | 操作的文件路径 |
| `CLAUDE_SESSION_ID` | 会话 ID |

### 输出与退出码

| 退出码 | 含义 |
|--------|------|
| 0 | 成功，继续执行 |
| 2 | **阻止操作**，Claude 收到阻止消息 |
| 其他非零 | 错误，但不阻止操作 |

Hook 的 stdout 输出会作为反馈传递给 Claude。

## 异步 Hook

对于不需要阻塞执行的 Hook，可以使用异步模式：

```json
{
  "type": "command",
  "command": "send-notification.sh",
  "async": true
}
```

异步 Hook 在后台运行，不会影响 Claude 的执行速度。

## 配置位置

Hooks 可以在不同层级配置：

| 层级 | 文件 | 范围 |
|------|------|------|
| 项目级 | `.claude/settings.json` | 当前项目 |
| 用户级 | `~/.claude/settings.json` | 所有项目 |
| 组织级 | 管理员部署 | 所有用户 |

## 注意事项

1. **Hook 命令在你的系统上执行**：确保命令是安全的
2. **同步 Hook 会阻塞**：耗时长的操作考虑使用异步 Hook
3. **错误处理**：Hook 失败不应该导致整个工作流崩溃
4. **测试 Hook**：先在安全环境中测试，确认行为正确
5. **退出码 2 = 阻止**：这是唯一能阻止 Claude 操作的退出码

## 实用技巧

1. **从简单 Hook 开始**：先实现一个自动格式化 Hook，熟悉机制
2. **善用 `matcher`**：精确匹配工具名称，避免不必要的触发
3. **日志先行**：先用日志 Hook 观察 Claude 的行为，再添加控制 Hook
4. **组合使用**：PreToolUse 做验证，PostToolUse 做后处理
5. **团队共享**：将 Hook 配置提交到 `.claude/settings.json`

---

## 关键要点

- Hooks 是在 Claude Code 生命周期特定时刻自动执行的用户定义命令
- 主要事件：`PreToolUse`（工具前）、`PostToolUse`（工具后）、`Stop`（完成时）
- 三种类型：命令钩子、HTTP 钩子、提示钩子
- 退出码 2 可以**阻止** Claude 的操作
- 常见用途：自动格式化、文件访问控制、审计日志、自动测试
- Hooks 把保障从"AI 应该记得做"变成"系统确保做"

---

[← 上一课：计划模式与思考模式](./01-plan-mode-and-thinking-mode.md) | [返回目录](../README.md) | [下一课：Claude Code SDK →](./03-claude-code-sdk.md)
