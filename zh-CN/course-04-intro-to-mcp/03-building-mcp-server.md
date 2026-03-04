# 课时 3：构建 MCP 服务器

[← 上一课：MCP 三大原语](./02-mcp-primitives.md) | [返回目录](./README.md) | [下一课：构建 MCP 客户端 →](./04-building-mcp-client.md)

---

## 核心概念

本课时将手把手教你使用 Python 构建一个功能完整的 MCP 服务器。我们将从项目初始化开始，依次实现工具、资源和提示，最后进行测试验证。

## 环境准备

### 安装依赖

```bash
# 创建项目目录
mkdir my-mcp-server && cd my-mcp-server

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
# venv\Scripts\activate   # Windows

# 安装 MCP SDK
pip install mcp
```

### 项目结构

```
my-mcp-server/
├── server.py          # 主服务器文件
├── requirements.txt   # 依赖列表
└── README.md          # 项目说明
```

`requirements.txt` 内容：

```
mcp>=1.0.0
httpx>=0.27.0
```

## 创建基础服务器

### 最小化服务器

先创建一个最基本的 MCP 服务器骨架：

```python
# server.py
from mcp.server import Server
import mcp.server.stdio

# 创建服务器实例
server = Server("my-first-server")

async def main():
    """Run the MCP server using stdio transport."""
    async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

运行服务器：

```bash
python server.py
```

此时服务器已启动，但还没有任何功能。接下来我们逐步添加三大原语。

## 实现工具 (Tools)

### 基本工具

```python
from mcp.server import Server

server = Server("todo-server")

# 内存中的待办事项存储
todos: dict[str, dict] = {}

@server.tool()
async def add_todo(title: str, description: str = "") -> str:
    """Add a new todo item.

    Args:
        title: Title of the todo item
        description: Optional description
    """
    import uuid
    todo_id = str(uuid.uuid4())[:8]
    todos[todo_id] = {
        "id": todo_id,
        "title": title,
        "description": description,
        "completed": False
    }
    return f"Todo '{title}' created with ID: {todo_id}"

@server.tool()
async def list_todos() -> str:
    """List all todo items."""
    if not todos:
        return "No todos found."

    result = []
    for todo in todos.values():
        status = "completed" if todo["completed"] else "pending"
        result.append(f"[{todo['id']}] {todo['title']} ({status})")

    return "\n".join(result)

@server.tool()
async def complete_todo(todo_id: str) -> str:
    """Mark a todo item as completed.

    Args:
        todo_id: The ID of the todo to complete
    """
    if todo_id not in todos:
        return f"Error: Todo with ID '{todo_id}' not found."

    todos[todo_id]["completed"] = True
    return f"Todo '{todos[todo_id]['title']}' marked as completed."
```

### 带输入验证的工具

良好的工具实现应该包含输入验证和错误处理：

```python
@server.tool()
async def search_todos(
    query: str,
    status: str = "all",
    limit: int = 10
) -> str:
    """Search todo items with filters.

    Args:
        query: Search query to match against title and description
        status: Filter by status - 'all', 'pending', or 'completed'
        limit: Maximum number of results (1-100)
    """
    # 输入验证
    if status not in ("all", "pending", "completed"):
        return f"Error: Invalid status '{status}'. Use 'all', 'pending', or 'completed'."

    if not 1 <= limit <= 100:
        return "Error: Limit must be between 1 and 100."

    # 执行搜索
    results = []
    for todo in todos.values():
        # 状态过滤
        if status == "pending" and todo["completed"]:
            continue
        if status == "completed" and not todo["completed"]:
            continue

        # 关键词匹配
        query_lower = query.lower()
        if (query_lower in todo["title"].lower() or
                query_lower in todo["description"].lower()):
            results.append(todo)

        if len(results) >= limit:
            break

    if not results:
        return f"No todos matching '{query}' found."

    return "\n".join(
        f"[{t['id']}] {t['title']}" for t in results
    )
```

### 返回结构化数据的工具

```python
import json

@server.tool()
async def get_todo_stats() -> str:
    """Get statistics about todo items."""
    total = len(todos)
    completed = sum(1 for t in todos.values() if t["completed"])
    pending = total - completed

    stats = {
        "total": total,
        "completed": completed,
        "pending": pending,
        "completion_rate": f"{(completed / total * 100):.1f}%" if total > 0 else "N/A"
    }

    return json.dumps(stats, indent=2)
```

## 实现资源 (Resources)

### 静态资源

```python
import json

@server.resource("todos://all")
async def get_all_todos() -> str:
    """Return all todo items as JSON."""
    return json.dumps(list(todos.values()), indent=2, ensure_ascii=False)

@server.resource("todos://stats")
async def get_stats_resource() -> str:
    """Return todo statistics."""
    total = len(todos)
    completed = sum(1 for t in todos.values() if t["completed"])
    stats = {
        "total": total,
        "completed": completed,
        "pending": total - completed
    }
    return json.dumps(stats, indent=2)
```

### 动态资源（URI 模板）

```python
@server.resource("todos://{todo_id}")
async def get_todo_by_id(todo_id: str) -> str:
    """Return a specific todo item by ID."""
    if todo_id not in todos:
        return json.dumps({"error": f"Todo '{todo_id}' not found"})

    return json.dumps(todos[todo_id], indent=2, ensure_ascii=False)
```

### 复杂资源示例

```python
from datetime import datetime

@server.resource("todos://report")
async def get_todo_report() -> str:
    """Generate a formatted report of all todos."""
    report_lines = [
        f"# Todo Report",
        f"Generated: {datetime.now().isoformat()}",
        f"",
        f"## Summary",
        f"- Total items: {len(todos)}",
        f"- Completed: {sum(1 for t in todos.values() if t['completed'])}",
        f"- Pending: {sum(1 for t in todos.values() if not t['completed'])}",
        f"",
        f"## Pending Items",
    ]

    for todo in todos.values():
        if not todo["completed"]:
            report_lines.append(f"- [{todo['id']}] {todo['title']}")
            if todo["description"]:
                report_lines.append(f"  {todo['description']}")

    report_lines.extend([
        f"",
        f"## Completed Items",
    ])

    for todo in todos.values():
        if todo["completed"]:
            report_lines.append(f"- [{todo['id']}] {todo['title']}")

    return "\n".join(report_lines)
```

## 实现提示 (Prompts)

### 基本提示模板

```python
from mcp.types import PromptMessage, TextContent

@server.prompt()
async def plan_tasks(goal: str) -> list[PromptMessage]:
    """Generate a task planning prompt.

    Args:
        goal: The project goal to plan tasks for
    """
    # 获取当前待办事项作为上下文
    existing = "\n".join(
        f"- {t['title']}" for t in todos.values() if not t["completed"]
    ) or "No existing tasks."

    return [
        PromptMessage(
            role="user",
            content=TextContent(
                type="text",
                text=f"""I need help planning tasks for the following goal:

Goal: {goal}

Current pending tasks:
{existing}

Please:
1. Break down the goal into actionable tasks
2. Identify any dependencies between tasks
3. Suggest a priority order
4. Estimate effort for each task (small/medium/large)
5. Note any tasks that overlap with existing ones"""
            )
        )
    ]
```

### 多消息提示

```python
@server.prompt()
async def review_progress() -> list[PromptMessage]:
    """Generate a progress review prompt with current data."""
    total = len(todos)
    completed = sum(1 for t in todos.values() if t["completed"])

    pending_list = "\n".join(
        f"- {t['title']}: {t['description'] or 'No description'}"
        for t in todos.values() if not t["completed"]
    ) or "None"

    completed_list = "\n".join(
        f"- {t['title']}"
        for t in todos.values() if t["completed"]
    ) or "None"

    return [
        PromptMessage(
            role="user",
            content=TextContent(
                type="text",
                text=f"""Please review my task progress and provide feedback.

Completed ({completed}/{total}):
{completed_list}

Pending ({total - completed}/{total}):
{pending_list}

Please analyze:
1. Overall progress assessment
2. Any potential blockers for pending tasks
3. Suggestions for prioritization
4. Recommendations for improving productivity"""
            )
        )
    ]
```

## 完整服务器代码

将所有组件整合到一个完整的服务器中：

```python
# server.py
import json
import uuid
from datetime import datetime

from mcp.server import Server
from mcp.types import PromptMessage, TextContent
import mcp.server.stdio

server = Server("todo-manager")

# 数据存储
todos: dict[str, dict] = {}


# ==================== 工具 ====================

@server.tool()
async def add_todo(title: str, description: str = "") -> str:
    """Add a new todo item."""
    todo_id = str(uuid.uuid4())[:8]
    todos[todo_id] = {
        "id": todo_id,
        "title": title,
        "description": description,
        "completed": False,
        "created_at": datetime.now().isoformat()
    }
    return f"Todo '{title}' created with ID: {todo_id}"

@server.tool()
async def complete_todo(todo_id: str) -> str:
    """Mark a todo item as completed."""
    if todo_id not in todos:
        return f"Error: Todo '{todo_id}' not found."
    todos[todo_id]["completed"] = True
    return f"Todo '{todos[todo_id]['title']}' marked as completed."

@server.tool()
async def delete_todo(todo_id: str) -> str:
    """Delete a todo item."""
    if todo_id not in todos:
        return f"Error: Todo '{todo_id}' not found."
    title = todos[todo_id]["title"]
    del todos[todo_id]
    return f"Todo '{title}' deleted."


# ==================== 资源 ====================

@server.resource("todos://all")
async def get_all_todos() -> str:
    """Return all todo items."""
    return json.dumps(list(todos.values()), indent=2, ensure_ascii=False)

@server.resource("todos://stats")
async def get_stats() -> str:
    """Return todo statistics."""
    total = len(todos)
    completed = sum(1 for t in todos.values() if t["completed"])
    return json.dumps({
        "total": total,
        "completed": completed,
        "pending": total - completed
    }, indent=2)


# ==================== 提示 ====================

@server.prompt()
async def plan_tasks(goal: str) -> list[PromptMessage]:
    """Generate a task planning prompt."""
    return [
        PromptMessage(
            role="user",
            content=TextContent(
                type="text",
                text=f"Help me plan tasks for: {goal}"
            )
        )
    ]


# ==================== 启动 ====================

async def main():
    async with mcp.server.stdio.stdio_server() as (read_stream, write_stream):
        await server.run(
            read_stream,
            write_stream,
            server.create_initialization_options()
        )

if __name__ == "__main__":
    import asyncio
    asyncio.run(main())
```

## 测试服务器

### 方法一：使用 MCP Inspector

MCP Inspector 是一个交互式调试工具：

```bash
# 安装并运行 Inspector
npx @modelcontextprotocol/inspector python server.py
```

Inspector 提供 Web 界面，可以：
- 查看服务器暴露的所有工具、资源和提示
- 手动调用工具并查看结果
- 读取资源内容
- 测试提示模板

### 方法二：使用 Claude Desktop

在 Claude Desktop 的配置文件中添加你的服务器：

```json
{
    "mcpServers": {
        "todo-manager": {
            "command": "python",
            "args": ["/path/to/server.py"]
        }
    }
}
```

配置文件位置：
- **macOS**: `~/Library/Application Support/Claude/claude_desktop_config.json`
- **Windows**: `%APPDATA%\Claude\claude_desktop_config.json`

### 方法三：编写测试脚本

```python
# test_server.py
import asyncio
import json
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def test():
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 测试工具列表
            tools = await session.list_tools()
            print("Available tools:")
            for tool in tools.tools:
                print(f"  - {tool.name}: {tool.description}")

            # 测试调用工具
            result = await session.call_tool(
                "add_todo",
                arguments={"title": "Learn MCP", "description": "Complete the tutorial"}
            )
            print(f"\nAdd todo result: {result.content[0].text}")

            # 测试读取资源
            resource = await session.read_resource("todos://all")
            print(f"\nAll todos: {resource.contents[0].text}")

            # 测试提示
            prompts = await session.list_prompts()
            print(f"\nAvailable prompts:")
            for prompt in prompts.prompts:
                print(f"  - {prompt.name}: {prompt.description}")

asyncio.run(test())
```

### 方法四：在 Claude Code 中使用

在项目的 `.claude/settings.json` 中配置：

```json
{
    "mcpServers": {
        "todo-manager": {
            "command": "python",
            "args": ["server.py"],
            "cwd": "/path/to/my-mcp-server"
        }
    }
}
```

然后在 Claude Code 中直接使用：

```
> 帮我添加一个待办事项："完成 MCP 教程"

Claude Code 自动调用 add_todo 工具，创建待办事项。
```

## 常见问题与调试

### 服务器无法启动

```bash
# 检查 Python 版本
python --version  # 需要 3.10+

# 检查依赖安装
pip list | grep mcp

# 查看详细错误
python server.py 2>&1
```

### 工具调用失败

```python
# 在工具中添加错误处理
@server.tool()
async def safe_tool(param: str) -> str:
    """A tool with proper error handling."""
    try:
        result = await do_something(param)
        return json.dumps({"success": True, "data": result})
    except ValueError as e:
        return json.dumps({"success": False, "error": str(e)})
    except Exception as e:
        return json.dumps({"success": False, "error": f"Unexpected error: {e}"})
```

---

## 关键要点

- 使用 `mcp` Python SDK 可以快速构建 MCP 服务器
- 通过 `@server.tool()` 装饰器注册工具，工具函数的 docstring 自动成为工具描述
- 通过 `@server.resource()` 装饰器注册资源，支持静态 URI 和 URI 模板
- 通过 `@server.prompt()` 装饰器注册提示模板
- 使用 MCP Inspector、Claude Desktop 或测试脚本进行调试
- 良好的工具设计应包含清晰的描述、输入验证和错误处理

---

[← 上一课：MCP 三大原语](./02-mcp-primitives.md) | [返回目录](./README.md) | [下一课：构建 MCP 客户端 →](./04-building-mcp-client.md)
