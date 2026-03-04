# 课时 4：构建 MCP 客户端

[← 上一课：构建 MCP 服务器](./03-building-mcp-server.md) | [返回目录](./README.md)

---

## 核心概念

MCP 客户端是连接 AI 应用与 MCP 服务器的桥梁。客户端负责发现服务器能力、调用工具、读取资源和使用提示模板。本课时将教你如何构建一个功能完整的 MCP 客户端。

## 客户端基础设置

### 安装依赖

```bash
pip install mcp anthropic
```

### 最简客户端

```python
# client.py
import asyncio
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    # 定义服务器启动参数
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )

    # 建立连接
    async with stdio_client(server_params) as (read_stream, write_stream):
        async with ClientSession(read_stream, write_stream) as session:
            # 初始化连接
            await session.initialize()
            print("Connected to MCP server!")

            # 查看服务器能力
            tools = await session.list_tools()
            print(f"Available tools: {len(tools.tools)}")

asyncio.run(main())
```

## 连接到服务器

### stdio 传输方式

最常见的本地连接方式，通过标准输入/输出通信：

```python
from mcp import StdioServerParameters
from mcp.client.stdio import stdio_client

async def connect_stdio():
    """Connect to a server via stdio transport."""
    server_params = StdioServerParameters(
        command="python",
        args=["path/to/server.py"],
        env={
            "DATABASE_URL": "postgresql://localhost:5432/mydb",
            "API_KEY": "your-api-key"
        }
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 打印服务器信息
            print(f"Server name: {session.server_info.name}")
            return session
```

### SSE 传输方式

适用于远程服务器连接：

```python
from mcp.client.sse import sse_client

async def connect_sse():
    """Connect to a server via SSE transport."""
    async with sse_client("http://localhost:8000/sse") as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            print("Connected via SSE!")
            return session
```

### 管理多个服务器连接

一个客户端可以同时连接多个 MCP 服务器：

```python
class MultiServerClient:
    """Client that manages connections to multiple MCP servers."""

    def __init__(self):
        self.sessions: dict[str, ClientSession] = {}

    async def connect(self, name: str, server_params: StdioServerParameters):
        """Connect to a named MCP server."""
        transport = stdio_client(server_params)
        read, write = await transport.__aenter__()
        session = ClientSession(read, write)
        await session.__aenter__()
        await session.initialize()
        self.sessions[name] = session
        print(f"Connected to '{name}'")

    async def disconnect_all(self):
        """Disconnect from all servers."""
        for name, session in self.sessions.items():
            await session.__aexit__(None, None, None)
            print(f"Disconnected from '{name}'")
        self.sessions.clear()

    def get_session(self, name: str) -> ClientSession:
        """Get a specific server session."""
        if name not in self.sessions:
            raise ValueError(f"Not connected to server '{name}'")
        return self.sessions[name]
```

## 调用工具

### 发现可用工具

```python
async def discover_tools(session: ClientSession):
    """List all tools available on the server."""
    result = await session.list_tools()

    print("=== Available Tools ===")
    for tool in result.tools:
        print(f"\nTool: {tool.name}")
        print(f"  Description: {tool.description}")
        if tool.inputSchema:
            props = tool.inputSchema.get("properties", {})
            required = tool.inputSchema.get("required", [])
            print("  Parameters:")
            for name, schema in props.items():
                req = "(required)" if name in required else "(optional)"
                print(f"    - {name}: {schema.get('type', 'any')} {req}")
```

### 调用工具

```python
async def call_tool_example(session: ClientSession):
    """Demonstrate calling tools."""

    # 调用不带参数的工具
    result = await session.call_tool("list_todos")
    print(f"List result: {result.content[0].text}")

    # 调用带参数的工具
    result = await session.call_tool(
        "add_todo",
        arguments={
            "title": "Learn MCP",
            "description": "Complete the MCP tutorial"
        }
    )
    print(f"Add result: {result.content[0].text}")

    # 调用带可选参数的工具
    result = await session.call_tool(
        "search_todos",
        arguments={
            "query": "MCP",
            "status": "pending",
            "limit": 5
        }
    )
    print(f"Search result: {result.content[0].text}")
```

### 处理工具调用结果

工具调用返回的内容可以包含多种类型：

```python
from mcp.types import TextContent, ImageContent, EmbeddedResource

async def handle_tool_result(session: ClientSession):
    """Handle different types of tool results."""
    result = await session.call_tool("get_info")

    for content in result.content:
        if isinstance(content, TextContent):
            print(f"Text: {content.text}")
        elif isinstance(content, ImageContent):
            print(f"Image: {content.mimeType}, {len(content.data)} bytes")
        elif isinstance(content, EmbeddedResource):
            print(f"Resource: {content.resource.uri}")
```

## 读取资源

### 发现可用资源

```python
async def discover_resources(session: ClientSession):
    """List all resources and resource templates."""

    # 列出静态资源
    resources = await session.list_resources()
    print("=== Static Resources ===")
    for resource in resources.resources:
        print(f"  URI: {resource.uri}")
        print(f"  Name: {resource.name}")
        print(f"  Type: {resource.mimeType}")
        print()

    # 列出资源模板
    templates = await session.list_resource_templates()
    print("=== Resource Templates ===")
    for template in templates.resourceTemplates:
        print(f"  URI Template: {template.uriTemplate}")
        print(f"  Name: {template.name}")
        print()
```

### 读取资源内容

```python
async def read_resources(session: ClientSession):
    """Read resource content from the server."""

    # 读取静态资源
    result = await session.read_resource("todos://all")
    content = result.contents[0]
    print(f"All todos:\n{content.text}")

    # 读取动态资源（使用具体参数替换模板）
    result = await session.read_resource("todos://abc123")
    content = result.contents[0]
    print(f"Specific todo:\n{content.text}")
```

### 订阅资源变更

```python
async def subscribe_to_resources(session: ClientSession):
    """Subscribe to resource change notifications."""

    # 订阅资源变更
    await session.subscribe_resource("todos://all")

    # 设置通知处理器
    @session.on_resource_updated
    async def handle_update(uri: str):
        print(f"Resource updated: {uri}")
        # 重新读取最新内容
        result = await session.read_resource(uri)
        print(f"New content: {result.contents[0].text}")
```

## 使用提示

### 发现可用提示

```python
async def discover_prompts(session: ClientSession):
    """List all available prompts."""
    result = await session.list_prompts()

    print("=== Available Prompts ===")
    for prompt in result.prompts:
        print(f"\nPrompt: {prompt.name}")
        print(f"  Description: {prompt.description}")
        if prompt.arguments:
            print("  Arguments:")
            for arg in prompt.arguments:
                req = "(required)" if arg.required else "(optional)"
                print(f"    - {arg.name}: {arg.description} {req}")
```

### 获取提示内容

```python
async def use_prompt(session: ClientSession):
    """Get and use a prompt template."""

    # 获取带参数的提示
    result = await session.get_prompt(
        "plan_tasks",
        arguments={"goal": "Build a REST API with authentication"}
    )

    # 提示返回一组消息，可以直接传给 LLM
    print("=== Prompt Messages ===")
    for message in result.messages:
        print(f"Role: {message.role}")
        print(f"Content: {message.content.text}")
        print()
```

## 完整客户端示例

以下是一个集成了 Anthropic API 的完整 MCP 客户端：

```python
# full_client.py
import asyncio
import json
from anthropic import Anthropic
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

class MCPChatClient:
    """A chat client that uses MCP tools with Claude."""

    def __init__(self):
        self.anthropic = Anthropic()
        self.session: ClientSession | None = None
        self.tools: list[dict] = []

    async def connect(self, server_command: str, server_args: list[str]):
        """Connect to an MCP server."""
        server_params = StdioServerParameters(
            command=server_command,
            args=server_args
        )
        transport = stdio_client(server_params)
        self.read, self.write = await transport.__aenter__()
        self.session = ClientSession(self.read, self.write)
        await self.session.__aenter__()
        await self.session.initialize()

        # 获取可用工具并转换为 Anthropic 格式
        result = await self.session.list_tools()
        self.tools = [
            {
                "name": tool.name,
                "description": tool.description or "",
                "input_schema": tool.inputSchema
            }
            for tool in result.tools
        ]
        print(f"Connected! {len(self.tools)} tools available.")

    async def chat(self, user_message: str) -> str:
        """Send a message and handle tool calls."""
        messages = [{"role": "user", "content": user_message}]

        while True:
            # 调用 Claude API
            response = self.anthropic.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=4096,
                tools=self.tools,
                messages=messages
            )

            # 检查是否需要工具调用
            if response.stop_reason == "tool_use":
                # 处理工具调用
                assistant_content = response.content
                messages.append({"role": "assistant", "content": assistant_content})

                tool_results = []
                for block in assistant_content:
                    if block.type == "tool_use":
                        print(f"  Calling tool: {block.name}({block.input})")

                        # 通过 MCP 调用工具
                        result = await self.session.call_tool(
                            block.name,
                            arguments=block.input
                        )

                        tool_results.append({
                            "type": "tool_result",
                            "tool_use_id": block.id,
                            "content": result.content[0].text
                        })

                messages.append({"role": "user", "content": tool_results})
            else:
                # 没有工具调用，返回文本响应
                return "".join(
                    block.text for block in response.content
                    if hasattr(block, "text")
                )

    async def disconnect(self):
        """Disconnect from the server."""
        if self.session:
            await self.session.__aexit__(None, None, None)


async def main():
    client = MCPChatClient()

    try:
        # 连接到 MCP 服务器
        await client.connect("python", ["server.py"])

        # 交互式对话
        print("\nMCP Chat Client (type 'quit' to exit)")
        print("=" * 40)

        while True:
            user_input = input("\nYou: ").strip()
            if user_input.lower() in ("quit", "exit"):
                break

            response = await client.chat(user_input)
            print(f"\nAssistant: {response}")

    finally:
        await client.disconnect()

asyncio.run(main())
```

## 错误处理

### 连接错误

```python
from mcp.shared.exceptions import McpError

async def safe_connect(server_params: StdioServerParameters):
    """Connect with proper error handling."""
    try:
        async with stdio_client(server_params) as (read, write):
            async with ClientSession(read, write) as session:
                await session.initialize()
                return session
    except ConnectionError as e:
        print(f"Failed to connect: {e}")
        print("Make sure the server is running and accessible.")
    except McpError as e:
        print(f"MCP protocol error: {e}")
    except FileNotFoundError:
        print("Server command not found. Check the command path.")
    except Exception as e:
        print(f"Unexpected error: {e}")
```

### 工具调用错误

```python
async def safe_tool_call(session: ClientSession, tool_name: str, args: dict):
    """Call a tool with error handling."""
    try:
        result = await session.call_tool(tool_name, arguments=args)

        if result.isError:
            print(f"Tool returned an error: {result.content[0].text}")
            return None

        return result.content[0].text

    except McpError as e:
        print(f"MCP error calling '{tool_name}': {e}")
        return None
    except Exception as e:
        print(f"Error calling '{tool_name}': {e}")
        return None
```

### 超时处理

```python
import asyncio

async def call_with_timeout(session: ClientSession, tool_name: str, args: dict, timeout: float = 30.0):
    """Call a tool with a timeout."""
    try:
        result = await asyncio.wait_for(
            session.call_tool(tool_name, arguments=args),
            timeout=timeout
        )
        return result.content[0].text
    except asyncio.TimeoutError:
        print(f"Tool '{tool_name}' timed out after {timeout}s")
        return None
```

### 重试机制

```python
async def call_with_retry(
    session: ClientSession,
    tool_name: str,
    args: dict,
    max_retries: int = 3,
    delay: float = 1.0
):
    """Call a tool with retry logic."""
    for attempt in range(max_retries):
        try:
            result = await session.call_tool(tool_name, arguments=args)
            return result.content[0].text
        except McpError as e:
            if attempt < max_retries - 1:
                print(f"Attempt {attempt + 1} failed, retrying in {delay}s...")
                await asyncio.sleep(delay)
                delay *= 2  # 指数退避
            else:
                print(f"All {max_retries} attempts failed for '{tool_name}'")
                raise
```

## 客户端最佳实践

### 1. 能力检查

在调用之前先确认服务器支持所需能力：

```python
async def check_capabilities(session: ClientSession):
    """Verify server capabilities before using them."""
    tools = await session.list_tools()
    tool_names = {t.name for t in tools.tools}

    required_tools = {"add_todo", "list_todos", "complete_todo"}
    missing = required_tools - tool_names

    if missing:
        print(f"Warning: Server is missing required tools: {missing}")
        return False
    return True
```

### 2. 连接生命周期管理

使用上下文管理器确保资源正确释放：

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def managed_connection(server_params: StdioServerParameters):
    """Manage MCP connection lifecycle."""
    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            try:
                yield session
            finally:
                print("Connection closed cleanly.")
```

### 3. 日志记录

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger("mcp-client")

async def logged_tool_call(session: ClientSession, tool_name: str, args: dict):
    """Call a tool with logging."""
    logger.info(f"Calling tool '{tool_name}' with args: {args}")
    result = await session.call_tool(tool_name, arguments=args)
    logger.info(f"Tool '{tool_name}' returned: {result.content[0].text[:100]}...")
    return result
```

---

## 关键要点

- MCP 客户端通过 `ClientSession` 与服务器建立连接和通信
- 支持 stdio 和 SSE 两种传输方式，分别适用于本地和远程场景
- 客户端可以发现并调用服务器的工具、读取资源、使用提示模板
- 将 MCP 客户端与 Anthropic API 结合，可以构建具有外部能力的 AI 应用
- 完善的错误处理应覆盖连接错误、工具调用错误和超时情况
- 最佳实践包括能力检查、连接生命周期管理和日志记录

---

[← 上一课：构建 MCP 服务器](./03-building-mcp-server.md) | [返回目录](./README.md)
