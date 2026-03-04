# 课时 8：MCP 服务器扩展

[← 上一课：自定义命令](./04-custom-commands.md) | [返回目录](../README.md) | [下一课：GitHub 集成 →](./06-github-integration.md)

---

## 核心概念

**MCP（Model Context Protocol，模型上下文协议）** 是一个开放标准，用于将 AI 工具连接到外部数据源和服务。通过 MCP，Claude Code 可以从一个强大的编程助手变成一个**可扩展的开发平台**。

## 什么是 MCP？

MCP 让 Claude Code 能够：

- 读取 Google Drive 中的设计文档
- 更新 Jira 中的工单
- 从 Slack 拉取消息
- 查询数据库
- 操控浏览器
- 使用你的自定义工具

```
┌───────────────┐     MCP 协议     ┌──────────────┐
│  Claude Code  │ ◄──────────────► │  MCP 服务器   │
│  （客户端）    │                  │  （提供工具）  │
└───────────────┘                  └──────┬───────┘
                                         │
                                    ┌────┴────┐
                                    │ 外部服务 │
                                    │ 数据源   │
                                    │ API     │
                                    └─────────┘
```

### MCP 的三个核心原语

| 原语 | 说明 | 示例 |
|------|------|------|
| **Tools（工具）** | Claude 可以调用的操作 | 创建 Jira 工单、发送 Slack 消息 |
| **Resources（资源）** | Claude 可以读取的数据 | 数据库记录、文件内容 |
| **Prompts（提示）** | 预定义的交互模板 | 代码审查模板、分析模板 |

## 配置 MCP 服务器

MCP 服务器在 Claude Code 的设置中配置。

### 方式一：交互式添加

```
> /mcp
```

这会打开 MCP 管理界面，让你搜索和添加 MCP 服务器。

### 方式二：手动配置

在项目设置文件 `.claude/settings.json` 中添加：

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-filesystem"],
      "env": {}
    }
  }
}
```

或在用户级设置 `~/.claude/settings.json` 中添加（对所有项目生效）。

### 配置格式

```json
{
  "mcpServers": {
    "server-name": {
      "command": "执行命令",
      "args": ["参数列表"],
      "env": {
        "API_KEY": "your-api-key"
      }
    }
  }
}
```

## 常用 MCP 服务器

### 浏览器自动化

```json
{
  "mcpServers": {
    "puppeteer": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-puppeteer"]
    }
  }
}
```

用途：
- 自动化浏览器测试
- 截取网页截图
- 爬取网页数据

### 数据库访问

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-postgres"],
      "env": {
        "DATABASE_URL": "postgresql://localhost:5432/mydb"
      }
    }
  }
}
```

用途：
- 查询数据库结构
- 分析数据
- 帮助编写查询语句

### 文件系统扩展

```json
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@anthropic-ai/mcp-filesystem", "/path/to/allowed/dir"]
    }
  }
}
```

用途：
- 访问项目目录外的文件
- 管理配置文件

## 实际使用场景

### 场景一：调试前端问题

配置浏览器 MCP 后：

```
> 打开 http://localhost:3000/login，截个图看看渲染效果

Claude Code 通过 MCP 操作浏览器，截取页面截图，分析渲染问题。
```

### 场景二：分析数据库

配置数据库 MCP 后：

```
> 查看 users 表的结构，分析最近一周的注册趋势

Claude Code 通过 MCP 查询数据库，分析数据并给出报告。
```

### 场景三：连接项目管理

配置 Jira/Linear MCP 后：

```
> 创建一个 Bug 工单，标题是"登录页面移动端显示异常"，
  附上错误详情和修复方案

Claude Code 通过 MCP 在项目管理工具中创建工单。
```

## MCP 生态系统

MCP 拥有丰富的社区服务器生态：

| 类别 | 示例服务器 |
|------|-----------|
| 浏览器 | Puppeteer, Playwright |
| 数据库 | PostgreSQL, MySQL, SQLite |
| 云服务 | AWS, GCP, Azure |
| 项目管理 | Jira, Linear, GitHub |
| 通信 | Slack, Discord |
| 文档 | Google Drive, Notion |
| 搜索 | Brave Search, Google Search |

> 更多 MCP 服务器可以在社区仓库中找到。

## 安全注意事项

使用 MCP 服务器时需要注意安全：

1. **最小权限原则**：只授予 MCP 服务器必要的权限
2. **API Key 管理**：使用环境变量存储敏感信息，不要硬编码
3. **审查操作**：对敏感操作（如数据库写入）保持警惕
4. **信任来源**：只使用信任的 MCP 服务器实现

```json
{
  "mcpServers": {
    "database": {
      "command": "npx",
      "args": ["-y", "@my-org/mcp-postgres"],
      "env": {
        "DATABASE_URL": "${DATABASE_URL}"
      }
    }
  }
}
```

## 实用技巧

1. **从官方 MCP 开始**：先使用 Anthropic 官方提供的 MCP 服务器
2. **项目级配置**：将项目需要的 MCP 配置放在 `.claude/settings.json` 中提交到 Git
3. **敏感信息用环境变量**：API Key 等不要写在配置文件中
4. **一次添加一个**：逐个添加和测试 MCP 服务器，确认每个都正常工作
5. **查看日志**：MCP 服务器出错时，检查 Claude Code 的日志输出

---

## 关键要点

- MCP 是连接 Claude Code 与外部服务的开放标准协议
- 三个核心原语：工具（Tools）、资源（Resources）、提示（Prompts）
- 在设置文件中配置 MCP 服务器，支持项目级和用户级
- 丰富的社区生态覆盖浏览器、数据库、云服务、项目管理等场景
- 使用 MCP 时要注意安全，遵循最小权限原则

---

[← 上一课：自定义命令](./04-custom-commands.md) | [返回目录](../README.md) | [下一课：GitHub 集成 →](./06-github-integration.md)
