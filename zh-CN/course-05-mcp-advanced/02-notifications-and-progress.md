# 课时 2：通知与进度

[← 上一课：采样](./01-sampling.md) | [返回目录](./README.md) | [下一课：文件系统与传输机制 →](./03-filesystem-and-transports.md)

---

## 核心概念

在 MCP 中，**通知（Notifications）** 和 **进度报告（Progress）** 机制允许服务器和客户端之间进行实时的状态更新。这对于长时间运行的操作尤为重要——用户可以看到任务的执行进度，而不是在漫长的等待中一无所知。

## 通知系统概述

### 通知 vs 请求

MCP 中的通信分为两种模式：

```
请求（Request）—— 需要响应：
客户端 ──── 请求 ────► 服务器
客户端 ◄──── 响应 ──── 服务器

通知（Notification）—— 单向发送，无需响应：
服务器 ──── 通知 ────► 客户端    （服务器 → 客户端）
客户端 ──── 通知 ────► 服务器    （客户端 → 服务器）
```

| 特性 | 请求 (Request) | 通知 (Notification) |
|------|----------------|---------------------|
| 方向 | 双向（请求-响应） | 单向 |
| 是否需要响应 | 是 | 否 |
| 是否有 ID | 是 | 否 |
| 用途 | 工具调用、资源读取 | 状态更新、事件通知 |

### 内置通知类型

MCP 协议定义了多种内置通知：

```python
# 服务器发出的通知
"notifications/tools/list_changed"     # 工具列表变更
"notifications/resources/list_changed" # 资源列表变更
"notifications/resources/updated"      # 资源内容更新
"notifications/prompts/list_changed"   # 提示列表变更
"notifications/progress"               # 进度更新

# 客户端发出的通知
"notifications/roots/list_changed"     # 根目录列表变更
"notifications/cancelled"              # 请求被取消
```

## 服务器端：发送通知

### 工具列表变更通知

当服务器动态添加或移除工具时，应通知客户端：

```python
from mcp.server import Server

server = Server("dynamic-server")

# 动态工具注册表
dynamic_tools: dict[str, callable] = {}

async def register_tool(name: str, handler: callable):
    """Dynamically register a new tool and notify clients."""
    dynamic_tools[name] = handler

    # 通知客户端工具列表已更新
    await server.request_context.session.send_tools_list_changed()
    print(f"Tool '{name}' registered, clients notified.")

async def unregister_tool(name: str):
    """Remove a tool and notify clients."""
    if name in dynamic_tools:
        del dynamic_tools[name]
        await server.request_context.session.send_tools_list_changed()
        print(f"Tool '{name}' removed, clients notified.")
```

### 资源更新通知

当资源内容发生变化时，通知订阅了该资源的客户端：

```python
import json
from datetime import datetime

server = Server("data-server")

# 数据存储
data_store: dict = {"last_updated": None, "records": []}

@server.resource("data://records")
async def get_records() -> str:
    """Return current records."""
    return json.dumps(data_store, indent=2, default=str)

@server.tool()
async def add_record(name: str, value: str) -> str:
    """Add a new record to the data store.

    Args:
        name: Record name
        value: Record value
    """
    data_store["records"].append({
        "name": name,
        "value": value,
        "created_at": datetime.now().isoformat()
    })
    data_store["last_updated"] = datetime.now().isoformat()

    # 通知客户端资源已更新
    await server.request_context.session.send_resource_updated("data://records")

    return f"Record '{name}' added. Total records: {len(data_store['records'])}"
```

## 进度报告

### 基本进度报告

对于长时间运行的工具，可以通过进度令牌（Progress Token）报告执行进度：

```python
from mcp.server import Server
from mcp.types import ProgressNotification

server = Server("progress-demo")

@server.tool()
async def process_large_dataset(file_path: str) -> str:
    """Process a large dataset with progress reporting.

    Args:
        file_path: Path to the dataset file
    """
    ctx = server.request_context

    # 从请求中获取进度令牌
    progress_token = ctx.meta.progressToken if ctx.meta else None

    # 模拟数据处理
    total_steps = 100
    results = []

    for i in range(total_steps):
        # 执行实际处理
        chunk_result = await process_chunk(file_path, i)
        results.append(chunk_result)

        # 报告进度
        if progress_token is not None:
            await ctx.session.send_progress_notification(
                progress_token=progress_token,
                progress=i + 1,
                total=total_steps
            )

    return f"Processed {total_steps} chunks. Summary: {summarize(results)}"
```

### 带描述的进度报告

进度报告可以包含可读的描述信息：

```python
@server.tool()
async def deploy_application(environment: str) -> str:
    """Deploy application to the specified environment.

    Args:
        environment: Target environment (staging/production)
    """
    ctx = server.request_context
    progress_token = ctx.meta.progressToken if ctx.meta else None

    steps = [
        ("Building application...", build_app),
        ("Running tests...", run_tests),
        ("Creating deployment package...", create_package),
        ("Uploading to server...", upload_package),
        ("Restarting services...", restart_services),
        ("Running health checks...", health_check),
    ]

    for i, (description, step_func) in enumerate(steps):
        # 发送带描述的进度通知
        if progress_token is not None:
            await ctx.session.send_progress_notification(
                progress_token=progress_token,
                progress=i,
                total=len(steps),
                message=description
            )

        # 执行步骤
        await step_func(environment)

    # 最终进度
    if progress_token is not None:
        await ctx.session.send_progress_notification(
            progress_token=progress_token,
            progress=len(steps),
            total=len(steps),
            message="Deployment complete!"
        )

    return f"Successfully deployed to {environment}"
```

### 不确定总量的进度

有些任务无法预知总工作量，这种情况下可以只报告已完成量：

```python
@server.tool()
async def crawl_website(url: str, max_pages: int = 50) -> str:
    """Crawl a website and index its pages.

    Args:
        url: Starting URL to crawl
        max_pages: Maximum number of pages to crawl
    """
    ctx = server.request_context
    progress_token = ctx.meta.progressToken if ctx.meta else None

    visited = set()
    queue = [url]
    pages_data = []

    while queue and len(visited) < max_pages:
        current_url = queue.pop(0)
        if current_url in visited:
            continue

        # 爬取页面
        page = await fetch_page(current_url)
        visited.add(current_url)
        pages_data.append(page)

        # 添加新发现的链接
        new_links = extract_links(page)
        queue.extend(link for link in new_links if link not in visited)

        # 报告进度（总量未知或动态变化）
        if progress_token is not None:
            await ctx.session.send_progress_notification(
                progress_token=progress_token,
                progress=len(visited),
                total=min(len(visited) + len(queue), max_pages)
            )

    return f"Crawled {len(visited)} pages from {url}"
```

## 客户端端：处理通知

### 监听进度更新

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
    server_params = StdioServerParameters(
        command="python",
        args=["server.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 设置进度处理回调
            @session.on_progress
            async def handle_progress(progress_token, progress, total, message=None):
                if total:
                    percentage = (progress / total) * 100
                    bar = "=" * int(percentage / 5) + "-" * (20 - int(percentage / 5))
                    status = f"[{bar}] {percentage:.0f}%"
                else:
                    status = f"Progress: {progress}"

                if message:
                    status += f" - {message}"

                print(f"\r{status}", end="", flush=True)

            # 调用长时间运行的工具
            # 进度令牌由客户端在调用时提供
            result = await session.call_tool(
                "process_large_dataset",
                arguments={"file_path": "data.csv"},
                progress_token="my-progress-1"
            )

            print(f"\nResult: {result.content[0].text}")
```

### 监听资源变更

```python
async def watch_resources(session: ClientSession):
    """Watch for resource changes."""

    # 订阅资源更新
    await session.subscribe_resource("data://records")

    # 处理资源更新通知
    @session.on_resource_updated
    async def handle_resource_update(uri: str):
        print(f"\nResource updated: {uri}")

        # 重新读取更新后的资源
        result = await session.read_resource(uri)
        content = result.contents[0].text
        print(f"New content: {content[:200]}...")

    # 处理资源列表变更
    @session.on_resource_list_changed
    async def handle_list_change():
        print("\nResource list changed!")
        resources = await session.list_resources()
        for r in resources.resources:
            print(f"  - {r.uri}: {r.name}")
```

### 监听工具列表变更

```python
async def watch_tools(session: ClientSession):
    """Watch for tool list changes."""

    @session.on_tools_list_changed
    async def handle_tools_change():
        print("\nTool list changed!")
        tools = await session.list_tools()
        print(f"Available tools ({len(tools.tools)}):")
        for tool in tools.tools:
            print(f"  - {tool.name}: {tool.description}")
```

## 事件处理模式

### 事件驱动的客户端

```python
class EventDrivenClient:
    """An event-driven MCP client."""

    def __init__(self):
        self.session: ClientSession | None = None
        self.event_handlers: dict[str, list[callable]] = {}

    def on(self, event_type: str, handler: callable):
        """Register an event handler."""
        if event_type not in self.event_handlers:
            self.event_handlers[event_type] = []
        self.event_handlers[event_type].append(handler)

    async def emit(self, event_type: str, **kwargs):
        """Emit an event to all registered handlers."""
        handlers = self.event_handlers.get(event_type, [])
        for handler in handlers:
            await handler(**kwargs)

    async def connect(self, server_params: StdioServerParameters):
        """Connect and set up event listeners."""
        async with stdio_client(server_params) as (read, write):
            async with ClientSession(read, write) as session:
                self.session = session
                await session.initialize()

                # 注册内部事件处理器
                @session.on_progress
                async def on_progress(token, progress, total, message=None):
                    await self.emit("progress", token=token,
                                    progress=progress, total=total,
                                    message=message)

                @session.on_tools_list_changed
                async def on_tools_changed():
                    tools = await session.list_tools()
                    await self.emit("tools_changed", tools=tools.tools)

                await self.emit("connected", session=session)

# 使用示例
client = EventDrivenClient()

@client.on("progress")
async def log_progress(token, progress, total, message):
    print(f"[{token}] {progress}/{total} - {message}")

@client.on("tools_changed")
async def refresh_tools(tools):
    print(f"Tools updated: {[t.name for t in tools]}")
```

### 带取消支持的操作

```python
import asyncio

async def cancellable_operation(session: ClientSession):
    """Demonstrate cancelling a long-running operation."""

    # 启动一个长时间运行的操作
    task = asyncio.create_task(
        session.call_tool(
            "crawl_website",
            arguments={"url": "https://example.com", "max_pages": 1000},
            progress_token="crawl-1"
        )
    )

    # 监控进度，必要时取消
    try:
        result = await asyncio.wait_for(task, timeout=60.0)
        print(f"Completed: {result.content[0].text}")
    except asyncio.TimeoutError:
        task.cancel()
        # 发送取消通知
        await session.send_cancelled_notification("crawl-1")
        print("Operation cancelled due to timeout")
```

## 实际应用：带进度的文件处理器

以下是一个完整的实际示例——文件批处理服务器：

```python
# file_processor_server.py
import os
import json
from mcp.server import Server
import mcp.server.stdio

server = Server("file-processor")

@server.tool()
async def batch_process_files(
    directory: str,
    pattern: str = "*.txt"
) -> str:
    """Process all matching files in a directory with progress reporting.

    Args:
        directory: Directory to process
        pattern: File glob pattern to match
    """
    import glob

    ctx = server.request_context
    progress_token = ctx.meta.progressToken if ctx.meta else None

    # 发现文件
    files = glob.glob(os.path.join(directory, pattern))
    total = len(files)

    if total == 0:
        return "No matching files found."

    results = []

    for i, file_path in enumerate(files):
        filename = os.path.basename(file_path)

        # 报告进度
        if progress_token is not None:
            await ctx.session.send_progress_notification(
                progress_token=progress_token,
                progress=i,
                total=total,
                message=f"Processing {filename}"
            )

        # 处理文件
        try:
            with open(file_path, "r") as f:
                content = f.read()
            word_count = len(content.split())
            line_count = content.count("\n") + 1
            results.append({
                "file": filename,
                "words": word_count,
                "lines": line_count,
                "status": "success"
            })
        except Exception as e:
            results.append({
                "file": filename,
                "error": str(e),
                "status": "failed"
            })

    # 完成进度
    if progress_token is not None:
        await ctx.session.send_progress_notification(
            progress_token=progress_token,
            progress=total,
            total=total,
            message="Processing complete"
        )

    # 汇总
    success = sum(1 for r in results if r["status"] == "success")
    failed = total - success

    summary = {
        "total_files": total,
        "successful": success,
        "failed": failed,
        "details": results
    }

    return json.dumps(summary, indent=2)

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

---

## 关键要点

- 通知是 MCP 中的单向消息机制，不需要响应
- 服务器可以在工具列表、资源列表或资源内容变更时发送通知
- 进度报告通过进度令牌（Progress Token）机制实现，由客户端在调用时提供
- 进度可以包含当前值、总量和可读描述信息
- 客户端通过注册回调函数来处理各类通知和进度更新
- 事件驱动模式是构建响应式 MCP 客户端的推荐方式
- 对长时间运行的操作要同时考虑进度报告和取消机制

---

[← 上一课：采样](./01-sampling.md) | [返回目录](./README.md) | [下一课：文件系统与传输机制 →](./03-filesystem-and-transports.md)
