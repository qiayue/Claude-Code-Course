# 课时 3：文件系统与传输机制

[← 上一课：通知与进度](./02-notifications-and-progress.md) | [返回目录](./README.md) | [下一课：生产环境实践 →](./04-production-patterns.md)

---

## 核心概念

本课时深入讲解 MCP 的两个关键基础设施主题：**文件系统访问模式**（如何安全地让 AI 访问文件）和**传输机制**（客户端和服务器之间如何通信）。理解这些底层机制将帮助你做出更好的架构决策。

## 文件系统访问模式

### 为什么文件系统访问很重要？

文件操作是 AI 工具最常见的需求之一。MCP 提供了标准化的方式让 AI 安全地访问文件系统，同时确保：

- 只访问被允许的目录
- 操作透明、可审计
- 支持多种文件操作模式

### 使用官方文件系统 MCP 服务器

Anthropic 提供了官方的文件系统 MCP 服务器：

```json
{
    "mcpServers": {
        "filesystem": {
            "command": "npx",
            "args": [
                "-y",
                "@modelcontextprotocol/server-filesystem",
                "/path/to/allowed/directory"
            ]
        }
    }
}
```

这个服务器提供以下工具：

| 工具 | 功能 |
|------|------|
| `read_file` | 读取文件内容 |
| `write_file` | 写入文件内容 |
| `list_directory` | 列出目录内容 |
| `create_directory` | 创建目录 |
| `move_file` | 移动/重命名文件 |
| `search_files` | 搜索文件 |
| `get_file_info` | 获取文件元信息 |
| `list_allowed_directories` | 列出允许访问的目录 |

### 构建自定义文件系统服务器

如果你需要更精细的控制，可以构建自定义的文件系统 MCP 服务器：

```python
import os
import json
from pathlib import Path
from mcp.server import Server
import mcp.server.stdio

server = Server("custom-filesystem")

# 安全配置：限制可访问的目录
ALLOWED_DIRS = [
    Path("/home/user/projects"),
    Path("/home/user/documents"),
]

def is_path_allowed(path: Path) -> bool:
    """Check if a path is within allowed directories."""
    resolved = path.resolve()
    return any(
        resolved == allowed or resolved.is_relative_to(allowed)
        for allowed in ALLOWED_DIRS
    )

@server.tool()
async def read_file(file_path: str) -> str:
    """Read the contents of a file.

    Args:
        file_path: Absolute path to the file to read
    """
    path = Path(file_path)

    if not is_path_allowed(path):
        return f"Error: Access denied. Path is outside allowed directories."

    if not path.exists():
        return f"Error: File not found: {file_path}"

    if not path.is_file():
        return f"Error: Not a file: {file_path}"

    try:
        content = path.read_text(encoding="utf-8")
        return content
    except UnicodeDecodeError:
        return f"Error: File is not a text file: {file_path}"
    except PermissionError:
        return f"Error: Permission denied: {file_path}"

@server.tool()
async def write_file(file_path: str, content: str) -> str:
    """Write content to a file.

    Args:
        file_path: Absolute path to the file to write
        content: Content to write to the file
    """
    path = Path(file_path)

    if not is_path_allowed(path):
        return "Error: Access denied. Path is outside allowed directories."

    try:
        path.parent.mkdir(parents=True, exist_ok=True)
        path.write_text(content, encoding="utf-8")
        return f"Successfully wrote {len(content)} characters to {file_path}"
    except PermissionError:
        return f"Error: Permission denied: {file_path}"

@server.tool()
async def list_directory(dir_path: str, recursive: bool = False) -> str:
    """List the contents of a directory.

    Args:
        dir_path: Absolute path to the directory
        recursive: Whether to list subdirectories recursively
    """
    path = Path(dir_path)

    if not is_path_allowed(path):
        return "Error: Access denied."

    if not path.exists() or not path.is_dir():
        return f"Error: Directory not found: {dir_path}"

    entries = []
    try:
        if recursive:
            for item in sorted(path.rglob("*")):
                rel_path = item.relative_to(path)
                item_type = "dir" if item.is_dir() else "file"
                size = item.stat().st_size if item.is_file() else 0
                entries.append({
                    "path": str(rel_path),
                    "type": item_type,
                    "size": size
                })
        else:
            for item in sorted(path.iterdir()):
                item_type = "dir" if item.is_dir() else "file"
                size = item.stat().st_size if item.is_file() else 0
                entries.append({
                    "name": item.name,
                    "type": item_type,
                    "size": size
                })
    except PermissionError:
        return f"Error: Permission denied: {dir_path}"

    return json.dumps(entries, indent=2)

# 资源：暴露允许的目录列表
@server.resource("fs://allowed-directories")
async def get_allowed_dirs() -> str:
    """Return the list of allowed directories."""
    return json.dumps([str(d) for d in ALLOWED_DIRS], indent=2)
```

### 文件监控模式

通过资源订阅实现文件变更监控：

```python
import asyncio
from pathlib import Path
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class FileChangeHandler(FileSystemEventHandler):
    """Handle file system events and notify MCP clients."""

    def __init__(self, server: Server):
        self.server = server
        self.changes: list[dict] = []

    def on_modified(self, event):
        if not event.is_directory:
            self.changes.append({
                "type": "modified",
                "path": event.src_path
            })

    def on_created(self, event):
        self.changes.append({
            "type": "created",
            "path": event.src_path
        })

    def on_deleted(self, event):
        self.changes.append({
            "type": "deleted",
            "path": event.src_path
        })

@server.resource("fs://changes")
async def get_recent_changes() -> str:
    """Return recent file system changes."""
    return json.dumps(file_handler.changes[-50:], indent=2)

@server.tool()
async def watch_directory(dir_path: str, duration: int = 30) -> str:
    """Watch a directory for changes.

    Args:
        dir_path: Directory to watch
        duration: How long to watch in seconds
    """
    path = Path(dir_path)
    if not is_path_allowed(path):
        return "Error: Access denied."

    handler = FileChangeHandler(server)
    observer = Observer()
    observer.schedule(handler, str(path), recursive=True)
    observer.start()

    await asyncio.sleep(duration)

    observer.stop()
    observer.join()

    return json.dumps({
        "watched": dir_path,
        "duration_seconds": duration,
        "changes": handler.changes
    }, indent=2)
```

## 传输机制

MCP 支持多种传输机制，每种都有其适用场景。

### 传输层架构

```
┌──────────────────────────────┐
│      MCP 协议 (JSON-RPC)     │   ← 应用层协议
├──────────────────────────────┤
│        传输层                 │   ← 可插拔
├──────────┬─────────┬─────────┤
│  stdio   │HTTP/SSE │  自定义  │
└──────────┴─────────┴─────────┘
```

### stdio 传输

**stdio（标准输入/输出）** 是最简单的传输方式。客户端启动服务器进程，通过 stdin/stdout 进行通信。

#### 工作原理

```
┌────────────┐  stdin (JSON-RPC)  ┌────────────┐
│  MCP 客户端 │ ────────────────► │  MCP 服务器  │
│  (父进程)   │ ◄──────────────── │  (子进程)    │
└────────────┘  stdout (JSON-RPC) └────────────┘
```

#### 服务器端实现

```python
# stdio_server.py
from mcp.server import Server
import mcp.server.stdio

server = Server("stdio-example")

@server.tool()
async def echo(message: str) -> str:
    """Echo a message back."""
    return f"Echo: {message}"

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

#### 客户端连接

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def connect_stdio():
    server_params = StdioServerParameters(
        command="python",
        args=["stdio_server.py"],
        env={"PYTHONUNBUFFERED": "1"}  # 确保输出不缓冲
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.call_tool("echo", {"message": "Hello!"})
            print(result.content[0].text)
```

#### stdio 传输特点

| 优点 | 缺点 |
|------|------|
| 配置简单，无需网络 | 仅限本地通信 |
| 安全性高（进程隔离） | 一个客户端对应一个服务器进程 |
| 无端口冲突 | 无法共享服务器实例 |
| 适合开发和测试 | 不适合生产环境大规模部署 |

### HTTP/SSE 传输

**HTTP + SSE（Server-Sent Events）** 传输允许通过网络进行远程连接，是生产环境中更常用的选择。

#### 工作原理

```
┌────────────┐   HTTP POST (请求)   ┌────────────┐
│  MCP 客户端 │ ─────────────────► │  MCP 服务器  │
│            │ ◄───────────────── │  (HTTP 服务) │
└────────────┘   SSE (响应/通知)    └────────────┘
```

- **客户端 → 服务器**：通过 HTTP POST 发送 JSON-RPC 请求
- **服务器 → 客户端**：通过 SSE（Server-Sent Events）推送响应和通知

#### 服务器端实现

```python
# sse_server.py
from mcp.server import Server
import mcp.server.sse
from starlette.applications import Starlette
from starlette.routing import Route, Mount
import uvicorn

server = Server("sse-example")

@server.tool()
async def get_time() -> str:
    """Get the current server time."""
    from datetime import datetime
    return datetime.now().isoformat()

@server.tool()
async def calculate(expression: str) -> str:
    """Evaluate a mathematical expression.

    Args:
        expression: Math expression to evaluate
    """
    try:
        # 注意：生产环境中不要使用 eval
        result = eval(expression, {"__builtins__": {}}, {})
        return str(result)
    except Exception as e:
        return f"Error: {e}"

# 创建 SSE 传输
sse = mcp.server.sse.SseServerTransport("/messages/")

async def handle_sse(request):
    """Handle SSE connections."""
    async with sse.connect_sse(
        request.scope, request.receive, request._send
    ) as streams:
        await server.run(
            streams[0],
            streams[1],
            server.create_initialization_options()
        )

# 创建 Starlette 应用
app = Starlette(
    routes=[
        Route("/sse", endpoint=handle_sse),
        Mount("/messages/", app=sse.handle_post_message),
    ]
)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

#### 客户端连接

```python
from mcp.client.sse import sse_client
from mcp import ClientSession

async def connect_sse():
    async with sse_client("http://localhost:8000/sse") as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 列出工具
            tools = await session.list_tools()
            for tool in tools.tools:
                print(f"Tool: {tool.name}")

            # 调用工具
            result = await session.call_tool("get_time")
            print(f"Server time: {result.content[0].text}")
```

#### HTTP/SSE 传输特点

| 优点 | 缺点 |
|------|------|
| 支持远程连接 | 需要网络配置 |
| 多客户端共享服务器 | 配置较复杂 |
| 可集成到 Web 基础设施 | 需要处理认证和安全 |
| 适合生产环境 | SSE 有些场景的兼容性问题 |

### Streamable HTTP 传输

MCP 还定义了更新的 Streamable HTTP 传输，提供更灵活的通信模式：

```python
# streamable_http_server.py
from mcp.server import Server
import mcp.server.streamable_http
from starlette.applications import Starlette
from starlette.routing import Mount
import uvicorn

server = Server("streamable-http-example")

@server.tool()
async def hello(name: str) -> str:
    """Say hello."""
    return f"Hello, {name}!"

# 创建 Streamable HTTP 传输
app = Starlette(
    routes=[
        Mount("/mcp", app=mcp.server.streamable_http.create_app(server)),
    ]
)

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

## 选择合适的传输机制

### 决策矩阵

| 场景 | 推荐传输 | 原因 |
|------|----------|------|
| 本地开发/测试 | stdio | 配置简单，无需网络 |
| Claude Desktop 集成 | stdio | 官方推荐方式 |
| Claude Code 集成 | stdio | 配置在 settings.json 中 |
| 远程服务器 | HTTP/SSE | 需要网络访问 |
| 微服务架构 | HTTP/SSE | 多服务间通信 |
| 多客户端共享 | HTTP/SSE | 服务器实例可复用 |
| 高安全要求 | stdio | 进程隔离，无网络暴露 |
| 容器化部署 | HTTP/SSE | 便于容器编排 |

### 决策流程图

```
需要远程访问？
│
├── 否 → 使用 stdio
│   ├── Claude Desktop / Claude Code 集成
│   ├── 本地开发和测试
│   └── 高安全要求场景
│
└── 是 → 使用 HTTP/SSE
    ├── 需要多客户端连接？ → HTTP/SSE
    ├── 微服务架构？ → HTTP/SSE
    └── 容器化部署？ → HTTP/SSE
```

### 混合模式

一个服务器可以同时支持多种传输：

```python
# hybrid_server.py
import asyncio
from mcp.server import Server
import mcp.server.stdio
import mcp.server.sse
from starlette.applications import Starlette
from starlette.routing import Route, Mount
import uvicorn

server = Server("hybrid-server")

@server.tool()
async def shared_tool(data: str) -> str:
    """A tool available via both transports."""
    return f"Processed: {data}"

async def run_stdio():
    """Run the server with stdio transport."""
    async with mcp.server.stdio.stdio_server() as (read, write):
        await server.run(read, write, server.create_initialization_options())

def create_sse_app():
    """Create the SSE app for HTTP transport."""
    sse = mcp.server.sse.SseServerTransport("/messages/")

    async def handle_sse(request):
        async with sse.connect_sse(
            request.scope, request.receive, request._send
        ) as streams:
            await server.run(
                streams[0], streams[1],
                server.create_initialization_options()
            )

    return Starlette(routes=[
        Route("/sse", endpoint=handle_sse),
        Mount("/messages/", app=sse.handle_post_message),
    ])

if __name__ == "__main__":
    import sys
    if "--stdio" in sys.argv:
        asyncio.run(run_stdio())
    else:
        app = create_sse_app()
        uvicorn.run(app, host="0.0.0.0", port=8000)
```

使用方式：

```bash
# stdio 模式
python hybrid_server.py --stdio

# HTTP/SSE 模式
python hybrid_server.py
```

---

## 关键要点

- MCP 的文件系统访问应始终实施路径白名单机制，确保只能访问允许的目录
- stdio 传输适用于本地场景，配置简单、安全性高、无需网络
- HTTP/SSE 传输适用于远程和生产场景，支持多客户端连接和网络访问
- 传输层是可插拔的，同一个服务器逻辑可以在不同传输之上运行
- 选择传输时主要考虑：是否需要远程访问、是否需要多客户端、安全要求和部署方式
- 一个服务器可以同时支持多种传输方式以适应不同的使用场景

---

[← 上一课：通知与进度](./02-notifications-and-progress.md) | [返回目录](./README.md) | [下一课：生产环境实践 →](./04-production-patterns.md)
