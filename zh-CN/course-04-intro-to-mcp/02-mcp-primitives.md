# 课时 2：MCP 三大原语

[← 上一课：什么是 MCP](./01-what-is-mcp.md) | [返回目录](./README.md) | [下一课：构建 MCP 服务器 →](./03-building-mcp-server.md)

---

## 核心概念

MCP 定义了三种核心原语（Primitives），它们是服务器向客户端暴露能力的三种方式。理解这三大原语是构建和使用 MCP 的基础。

```
┌─────────────────────────────────────────────┐
│              MCP 三大原语                    │
├─────────────┬──────────────┬────────────────┤
│   Tools     │  Resources   │   Prompts      │
│   工具       │  资源        │   提示模板      │
│             │              │                │
│  AI 可调用   │  AI 可读取   │  可复用的       │
│  的函数      │  的数据      │  提示模板       │
│             │              │                │
│  模型控制    │  应用控制     │  用户控制       │
└─────────────┴──────────────┴────────────────┘
```

## 原语一：工具 (Tools)

### 什么是工具？

工具是 MCP 服务器暴露给 AI 模型的**可调用函数**。当 AI 模型判断需要执行某个操作时，它可以请求调用相应的工具。

### 核心特征

- **由模型控制**：AI 模型根据上下文自动决定是否调用工具
- **有副作用**：工具通常会执行实际操作（写入数据库、发送消息等）
- **需要确认**：敏感操作在执行前需要用户确认
- **有输入输出**：接受参数，返回结果

### 工具定义示例

```python
from mcp.server import Server
from mcp.types import Tool

server = Server("demo-server")

@server.tool()
async def search_users(query: str, limit: int = 10) -> str:
    """Search for users in the database.

    Args:
        query: Search query string
        limit: Maximum number of results to return
    """
    # 执行数据库查询
    results = await db.search("users", query, limit=limit)
    return format_results(results)
```

工具在 MCP 协议中的 JSON Schema 表示：

```json
{
    "name": "search_users",
    "description": "Search for users in the database.",
    "inputSchema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "Search query string"
            },
            "limit": {
                "type": "integer",
                "description": "Maximum number of results to return",
                "default": 10
            }
        },
        "required": ["query"]
    }
}
```

### 工具的常见用途

| 用途 | 示例 |
|------|------|
| 数据查询 | 执行 SQL 查询、搜索文档 |
| 数据修改 | 创建记录、更新配置 |
| 外部调用 | 发送邮件、调用第三方 API |
| 系统操作 | 文件操作、执行命令 |
| 计算处理 | 数据转换、格式化 |

### 工具调用流程

```
用户请求 → AI 模型分析 → 选择合适的工具 → 客户端发送请求 → 服务器执行
                                                              ↓
用户看到结果 ← AI 模型整合 ← 客户端接收 ← 服务器返回结果
```

## 原语二：资源 (Resources)

### 什么是资源？

资源是 MCP 服务器暴露的**可读取数据**。与工具不同，资源是只读的，不会产生副作用。资源通过 URI 标识，类似于 REST API 的 GET 端点。

### 核心特征

- **由应用控制**：宿主应用决定何时读取资源，而非 AI 模型自主决定
- **只读操作**：资源不会修改外部状态
- **URI 标识**：每个资源都有唯一的 URI
- **支持模板**：资源 URI 可以包含动态参数

### 资源定义示例

```python
from mcp.server import Server

server = Server("demo-server")

# 静态资源
@server.resource("config://app")
async def get_app_config() -> str:
    """Return the application configuration."""
    config = await load_config()
    return json.dumps(config, indent=2)

# 动态资源（使用 URI 模板）
@server.resource("users://{user_id}/profile")
async def get_user_profile(user_id: str) -> str:
    """Return a user's profile information."""
    profile = await db.get_user(user_id)
    return json.dumps(profile, indent=2)
```

### 资源类型

MCP 支持两种资源类型：

```python
# 文本资源
@server.resource("docs://readme")
async def get_readme() -> str:
    """Return the README content as text."""
    with open("README.md") as f:
        return f.read()

# 二进制资源（使用 Base64 编码）
@server.resource("images://logo")
async def get_logo() -> bytes:
    """Return the company logo."""
    with open("logo.png", "rb") as f:
        return f.read()
```

### 资源列表与发现

客户端可以发现服务器提供的所有资源：

```python
# 服务器提供的资源列表
resources = [
    {
        "uri": "config://app",
        "name": "Application Config",
        "description": "Current application configuration",
        "mimeType": "application/json"
    },
    {
        "uri": "db://schema",
        "name": "Database Schema",
        "description": "Current database table definitions",
        "mimeType": "text/plain"
    }
]

# 资源模板（动态资源）
resource_templates = [
    {
        "uriTemplate": "users://{user_id}/profile",
        "name": "User Profile",
        "description": "Profile for a specific user"
    }
]
```

### 资源与工具的区别

| 特性 | 资源 (Resources) | 工具 (Tools) |
|------|-------------------|--------------|
| 操作类型 | 只读 | 读写 |
| 副作用 | 无 | 有 |
| 控制方 | 应用控制 | 模型控制 |
| 类比 | GET 请求 | POST/PUT/DELETE 请求 |
| 用途 | 提供上下文信息 | 执行操作 |

## 原语三：提示 (Prompts)

### 什么是提示？

提示是 MCP 服务器提供的**可复用提示模板**。它们定义了 AI 模型与用户交互的特定模式，可以包含预设的系统指令、参数化的用户消息等。

### 核心特征

- **由用户控制**：用户选择使用哪个提示模板
- **可参数化**：模板支持动态参数
- **多消息组合**：可以包含多个角色的消息
- **嵌入资源**：可以引用资源作为上下文

### 提示定义示例

```python
from mcp.server import Server
from mcp.types import Prompt, PromptArgument, PromptMessage, TextContent

server = Server("demo-server")

@server.prompt()
async def code_review(code: str, language: str = "python") -> list[PromptMessage]:
    """Generate a code review prompt.

    Args:
        code: The code to review
        language: Programming language of the code
    """
    return [
        PromptMessage(
            role="user",
            content=TextContent(
                type="text",
                text=f"""Please review the following {language} code.
Focus on:
1. Code quality and best practices
2. Potential bugs or errors
3. Performance considerations
4. Security issues

Code to review:
```{language}
{code}
```"""
            )
        )
    ]
```

### 提示的高级用法

提示可以组合多条消息和嵌入资源：

```python
@server.prompt()
async def debug_error(
    error_message: str,
    stack_trace: str = ""
) -> list[PromptMessage]:
    """Generate a debugging prompt with context."""
    messages = [
        PromptMessage(
            role="user",
            content=TextContent(
                type="text",
                text=f"""I encountered the following error:

Error: {error_message}

{f'Stack trace:{chr(10)}{stack_trace}' if stack_trace else ''}

Please help me:
1. Identify the root cause
2. Suggest a fix
3. Explain how to prevent this in the future"""
            )
        )
    ]
    return messages
```

### 提示列表

客户端可以发现可用的提示模板：

```python
prompts = [
    {
        "name": "code_review",
        "description": "Generate a code review prompt",
        "arguments": [
            {
                "name": "code",
                "description": "The code to review",
                "required": True
            },
            {
                "name": "language",
                "description": "Programming language",
                "required": False
            }
        ]
    },
    {
        "name": "debug_error",
        "description": "Generate a debugging prompt with context",
        "arguments": [
            {
                "name": "error_message",
                "description": "The error message",
                "required": True
            },
            {
                "name": "stack_trace",
                "description": "Optional stack trace",
                "required": False
            }
        ]
    }
]
```

## 三大原语的选择指南

### 何时使用工具？

- 需要**执行操作**或产生副作用时
- AI 模型需要**动态决策**调用时
- 需要**写入或修改**外部系统时
- 示例：发送消息、创建文件、执行查询

### 何时使用资源？

- 需要为 AI 提供**上下文信息**时
- 数据是**只读**的，不需要修改
- 内容可以通过 **URI 标识**时
- 示例：配置文件、数据库 schema、文档内容

### 何时使用提示？

- 需要**标准化交互模式**时
- 特定任务有**最佳实践模板**时
- 希望**用户选择**预定义的工作流时
- 示例：代码审查模板、SQL 优化分析、错误排查指南

### 决策流程图

```
需要 AI 访问外部能力？
│
├── 需要执行操作/修改数据？
│   └── 是 → 使用 工具 (Tools)
│
├── 只需要读取数据/提供上下文？
│   └── 是 → 使用 资源 (Resources)
│
└── 需要标准化的交互模板？
    └── 是 → 使用 提示 (Prompts)
```

## 实际场景示例

### 数据库管理 MCP 服务器

一个数据库管理服务器可能同时使用三种原语：

```python
from mcp.server import Server

server = Server("database-manager")

# 工具：执行查询（有副作用）
@server.tool()
async def execute_query(sql: str) -> str:
    """Execute a SQL query against the database."""
    result = await db.execute(sql)
    return format_result(result)

# 资源：读取 schema（只读）
@server.resource("db://schema")
async def get_schema() -> str:
    """Return the current database schema."""
    schema = await db.get_schema()
    return schema

# 资源：读取特定表的信息
@server.resource("db://tables/{table_name}")
async def get_table_info(table_name: str) -> str:
    """Return information about a specific table."""
    info = await db.describe_table(table_name)
    return json.dumps(info, indent=2)

# 提示：SQL 优化模板
@server.prompt()
async def optimize_query(sql: str) -> list[PromptMessage]:
    """Generate a prompt for SQL query optimization."""
    schema = await db.get_schema()
    return [
        PromptMessage(
            role="user",
            content=TextContent(
                type="text",
                text=f"""Given this database schema:
{schema}

Please optimize the following SQL query for better performance:
```sql
{sql}
```

Consider: indexes, query plan, join order, and data volume."""
            )
        )
    ]
```

---

## 关键要点

- MCP 定义了三大原语：工具（Tools）、资源（Resources）和提示（Prompts）
- **工具**由模型控制，用于执行有副作用的操作
- **资源**由应用控制，用于提供只读的上下文数据
- **提示**由用户控制，用于标准化的交互模板
- 三种原语可以灵活组合，服务器可以根据需要选择暴露哪些能力
- 选择原语时，核心考虑因素是：是否有副作用、谁来控制、以及数据流向

---

[← 上一课：什么是 MCP](./01-what-is-mcp.md) | [返回目录](./README.md) | [下一课：构建 MCP 服务器 →](./03-building-mcp-server.md)
