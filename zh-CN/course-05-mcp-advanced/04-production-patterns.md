# 课时 4：生产环境实践

[← 上一课：文件系统与传输机制](./03-filesystem-and-transports.md) | [返回目录](./README.md)

---

## 核心概念

将 MCP 服务器部署到生产环境需要考虑安全性、可靠性、可观测性和可扩展性。本课时汇总了在生产环境中运行 MCP 服务器的关键实践和模式。

## 安全性

### 输入验证

所有来自客户端的输入都必须经过严格验证。永远不要信任外部输入。

```python
import re
from pathlib import Path
from mcp.server import Server

server = Server("secure-server")

# 路径遍历防护
def sanitize_path(user_path: str, base_dir: str) -> Path:
    """Sanitize a file path to prevent directory traversal."""
    base = Path(base_dir).resolve()
    target = (base / user_path).resolve()

    # 确保目标路径在基础目录内
    if not target.is_relative_to(base):
        raise ValueError(f"Path traversal detected: {user_path}")

    return target

# SQL 注入防护
def validate_identifier(name: str) -> str:
    """Validate a SQL identifier (table/column name)."""
    if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', name):
        raise ValueError(f"Invalid identifier: {name}")
    return name

@server.tool()
async def read_project_file(file_path: str) -> str:
    """Read a file from the project directory.

    Args:
        file_path: Relative path within the project
    """
    try:
        safe_path = sanitize_path(file_path, "/app/project")
        return safe_path.read_text(encoding="utf-8")
    except ValueError as e:
        return f"Security error: {e}"
    except FileNotFoundError:
        return f"File not found: {file_path}"

@server.tool()
async def query_table(table_name: str, limit: int = 100) -> str:
    """Query records from a database table.

    Args:
        table_name: Name of the table to query
        limit: Maximum rows to return (1-1000)
    """
    # 验证表名
    try:
        safe_table = validate_identifier(table_name)
    except ValueError:
        return "Error: Invalid table name."

    # 验证 limit 范围
    if not 1 <= limit <= 1000:
        return "Error: Limit must be between 1 and 1000."

    # 使用参数化查询（永远不要拼接 SQL）
    query = f"SELECT * FROM {safe_table} LIMIT %s"
    results = await db.execute(query, (limit,))
    return format_results(results)
```

### 权限控制

为不同的操作级别实施权限控制：

```python
from enum import Enum
from functools import wraps

class Permission(Enum):
    READ = "read"
    WRITE = "write"
    ADMIN = "admin"

class PermissionManager:
    """Manage tool-level permissions."""

    def __init__(self):
        self.permissions: dict[str, set[Permission]] = {
            "default": {Permission.READ},
            "admin": {Permission.READ, Permission.WRITE, Permission.ADMIN}
        }

    def check(self, role: str, required: Permission) -> bool:
        """Check if a role has the required permission."""
        role_perms = self.permissions.get(role, set())
        return required in role_perms

permissions = PermissionManager()

def require_permission(permission: Permission):
    """Decorator to enforce permission checks on tools."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # 从请求上下文获取用户角色
            ctx = server.request_context
            role = getattr(ctx, "role", "default")

            if not permissions.check(role, permission):
                return f"Permission denied: requires '{permission.value}' permission."

            return await func(*args, **kwargs)
        return wrapper
    return decorator

@server.tool()
@require_permission(Permission.READ)
async def read_config() -> str:
    """Read server configuration."""
    config = await load_config()
    # 过滤敏感字段
    safe_config = {k: v for k, v in config.items() if k not in ("api_key", "secret")}
    return json.dumps(safe_config, indent=2)

@server.tool()
@require_permission(Permission.WRITE)
async def update_config(key: str, value: str) -> str:
    """Update a configuration value.

    Args:
        key: Configuration key
        value: New value
    """
    if key in ("api_key", "secret", "database_url"):
        return "Error: Cannot modify sensitive configuration via this tool."

    await save_config(key, value)
    return f"Configuration '{key}' updated."

@server.tool()
@require_permission(Permission.ADMIN)
async def reset_database() -> str:
    """Reset the database to its initial state."""
    await db.reset()
    return "Database reset complete."
```

### 敏感数据处理

```python
import re

class DataSanitizer:
    """Sanitize sensitive data from tool outputs."""

    PATTERNS = {
        "api_key": re.compile(r'(?:api[_-]?key|token)\s*[:=]\s*["\']?([a-zA-Z0-9_\-]{20,})["\']?', re.I),
        "password": re.compile(r'(?:password|passwd|pwd)\s*[:=]\s*["\']?([^\s"\']+)["\']?', re.I),
        "email": re.compile(r'[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}'),
    }

    @classmethod
    def sanitize(cls, text: str) -> str:
        """Remove sensitive data patterns from text."""
        result = text
        for name, pattern in cls.PATTERNS.items():
            result = pattern.sub(f"[REDACTED_{name.upper()}]", result)
        return result

@server.tool()
async def get_system_info() -> str:
    """Get system information (with sensitive data redacted)."""
    info = await collect_system_info()
    raw_output = json.dumps(info, indent=2)
    return DataSanitizer.sanitize(raw_output)
```

### 请求频率限制

```python
import time
from collections import defaultdict

class RateLimiter:
    """Simple rate limiter for tool calls."""

    def __init__(self, max_calls: int = 60, window_seconds: int = 60):
        self.max_calls = max_calls
        self.window = window_seconds
        self.calls: dict[str, list[float]] = defaultdict(list)

    def check(self, tool_name: str) -> bool:
        """Check if a tool call is within rate limits."""
        now = time.time()
        window_start = now - self.window

        # 清理过期记录
        self.calls[tool_name] = [
            t for t in self.calls[tool_name] if t > window_start
        ]

        if len(self.calls[tool_name]) >= self.max_calls:
            return False

        self.calls[tool_name].append(now)
        return True

rate_limiter = RateLimiter(max_calls=30, window_seconds=60)

@server.tool()
async def rate_limited_tool(query: str) -> str:
    """A tool with rate limiting.

    Args:
        query: Search query
    """
    if not rate_limiter.check("rate_limited_tool"):
        return "Error: Rate limit exceeded. Please wait before trying again."

    return await perform_search(query)
```

## 错误处理模式

### 结构化错误响应

```python
import json
import traceback
from enum import Enum

class ErrorCode(Enum):
    VALIDATION_ERROR = "VALIDATION_ERROR"
    NOT_FOUND = "NOT_FOUND"
    PERMISSION_DENIED = "PERMISSION_DENIED"
    RATE_LIMITED = "RATE_LIMITED"
    INTERNAL_ERROR = "INTERNAL_ERROR"
    TIMEOUT = "TIMEOUT"

def error_response(code: ErrorCode, message: str, details: dict = None) -> str:
    """Create a structured error response."""
    response = {
        "error": {
            "code": code.value,
            "message": message
        }
    }
    if details:
        response["error"]["details"] = details
    return json.dumps(response, indent=2)

@server.tool()
async def robust_tool(item_id: str) -> str:
    """A tool with comprehensive error handling.

    Args:
        item_id: ID of the item to process
    """
    # 输入验证
    if not item_id or len(item_id) > 100:
        return error_response(
            ErrorCode.VALIDATION_ERROR,
            "Invalid item_id: must be 1-100 characters."
        )

    try:
        item = await db.get_item(item_id)
        if item is None:
            return error_response(
                ErrorCode.NOT_FOUND,
                f"Item '{item_id}' not found."
            )

        result = await process_item(item)
        return json.dumps({"success": True, "data": result}, indent=2)

    except PermissionError:
        return error_response(
            ErrorCode.PERMISSION_DENIED,
            "Insufficient permissions to access this item."
        )
    except TimeoutError:
        return error_response(
            ErrorCode.TIMEOUT,
            "Operation timed out. Please try again."
        )
    except Exception as e:
        # 记录完整错误，但只返回安全信息给客户端
        logger.error(f"Error processing item {item_id}: {traceback.format_exc()}")
        return error_response(
            ErrorCode.INTERNAL_ERROR,
            "An internal error occurred. Please try again later."
        )
```

### 优雅降级

```python
@server.tool()
async def get_enriched_data(item_id: str) -> str:
    """Get item data with enrichment from multiple sources.

    Args:
        item_id: Item ID to look up
    """
    # 核心数据（必须成功）
    try:
        core_data = await db.get_item(item_id)
        if not core_data:
            return error_response(ErrorCode.NOT_FOUND, "Item not found.")
    except Exception as e:
        return error_response(ErrorCode.INTERNAL_ERROR, "Failed to fetch core data.")

    result = {"item": core_data, "enrichments": {}}

    # 增强数据（可以失败，降级处理）
    enrichment_sources = {
        "analytics": fetch_analytics,
        "recommendations": fetch_recommendations,
        "related_items": fetch_related,
    }

    for source_name, fetch_func in enrichment_sources.items():
        try:
            data = await asyncio.wait_for(fetch_func(item_id), timeout=5.0)
            result["enrichments"][source_name] = data
        except asyncio.TimeoutError:
            result["enrichments"][source_name] = {"error": "timeout"}
            logger.warning(f"Enrichment '{source_name}' timed out for item {item_id}")
        except Exception as e:
            result["enrichments"][source_name] = {"error": "unavailable"}
            logger.warning(f"Enrichment '{source_name}' failed: {e}")

    return json.dumps(result, indent=2)
```

## 扩展 MCP 服务器

### 连接池

```python
import asyncpg

class DatabasePool:
    """Managed database connection pool."""

    def __init__(self):
        self.pool: asyncpg.Pool | None = None

    async def initialize(self, dsn: str, min_size: int = 5, max_size: int = 20):
        """Initialize the connection pool."""
        self.pool = await asyncpg.create_pool(
            dsn,
            min_size=min_size,
            max_size=max_size,
            command_timeout=30
        )

    async def execute(self, query: str, *args):
        """Execute a query using a pooled connection."""
        async with self.pool.acquire() as conn:
            return await conn.fetch(query, *args)

    async def close(self):
        """Close all connections."""
        if self.pool:
            await self.pool.close()

db_pool = DatabasePool()

@server.tool()
async def query_with_pool(sql: str) -> str:
    """Execute a read-only SQL query.

    Args:
        sql: SQL SELECT query to execute
    """
    # 只允许 SELECT 查询
    if not sql.strip().upper().startswith("SELECT"):
        return "Error: Only SELECT queries are allowed."

    try:
        rows = await db_pool.execute(sql)
        return json.dumps([dict(row) for row in rows], indent=2, default=str)
    except Exception as e:
        return f"Query error: {e}"
```

### 缓存层

```python
import time
import hashlib
from typing import Any

class SimpleCache:
    """In-memory cache with TTL support."""

    def __init__(self, default_ttl: int = 300):
        self.cache: dict[str, tuple[Any, float]] = {}
        self.default_ttl = default_ttl

    def get(self, key: str) -> Any | None:
        """Get a cached value if it exists and hasn't expired."""
        if key in self.cache:
            value, expiry = self.cache[key]
            if time.time() < expiry:
                return value
            del self.cache[key]
        return None

    def set(self, key: str, value: Any, ttl: int = None):
        """Cache a value with TTL."""
        expiry = time.time() + (ttl or self.default_ttl)
        self.cache[key] = (value, expiry)

    def make_key(self, *args) -> str:
        """Generate a cache key from arguments."""
        raw = json.dumps(args, sort_keys=True)
        return hashlib.md5(raw.encode()).hexdigest()

cache = SimpleCache(default_ttl=60)

@server.tool()
async def cached_search(query: str, category: str = "all") -> str:
    """Search with caching for repeated queries.

    Args:
        query: Search query
        category: Category filter
    """
    cache_key = cache.make_key("search", query, category)

    # 检查缓存
    cached = cache.get(cache_key)
    if cached is not None:
        return cached

    # 执行搜索
    results = await perform_search(query, category)
    response = json.dumps(results, indent=2)

    # 缓存结果
    cache.set(cache_key, response, ttl=120)

    return response
```

### 并发控制

```python
import asyncio

class ConcurrencyLimiter:
    """Limit concurrent executions of a tool."""

    def __init__(self, max_concurrent: int = 5):
        self.semaphore = asyncio.Semaphore(max_concurrent)
        self.active = 0

    async def acquire(self):
        await self.semaphore.acquire()
        self.active += 1

    def release(self):
        self.active -= 1
        self.semaphore.release()

limiter = ConcurrencyLimiter(max_concurrent=3)

@server.tool()
async def heavy_computation(data: str) -> str:
    """Perform a resource-intensive computation.

    Args:
        data: Input data to process
    """
    try:
        await limiter.acquire()
        result = await run_expensive_operation(data)
        return json.dumps(result, indent=2)
    finally:
        limiter.release()
```

## 监控与日志

### 结构化日志

```python
import structlog
import time

# 配置结构化日志
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer()
    ]
)

logger = structlog.get_logger()

def logged_tool(func):
    """Decorator to add structured logging to tools."""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start_time = time.time()
        tool_name = func.__name__

        logger.info("tool_call_started",
                     tool=tool_name,
                     args=kwargs)
        try:
            result = await func(*args, **kwargs)
            duration = time.time() - start_time

            logger.info("tool_call_completed",
                        tool=tool_name,
                        duration_ms=round(duration * 1000, 2),
                        result_length=len(result))

            return result
        except Exception as e:
            duration = time.time() - start_time
            logger.error("tool_call_failed",
                         tool=tool_name,
                         duration_ms=round(duration * 1000, 2),
                         error=str(e),
                         error_type=type(e).__name__)
            raise

    return wrapper

@server.tool()
@logged_tool
async def monitored_operation(input_data: str) -> str:
    """An operation with full monitoring.

    Args:
        input_data: Data to process
    """
    return await process(input_data)
```

### 指标收集

```python
from collections import defaultdict
import time

class Metrics:
    """Simple metrics collector for MCP server monitoring."""

    def __init__(self):
        self.call_counts: dict[str, int] = defaultdict(int)
        self.error_counts: dict[str, int] = defaultdict(int)
        self.latencies: dict[str, list[float]] = defaultdict(list)
        self.start_time = time.time()

    def record_call(self, tool_name: str, duration: float, error: bool = False):
        """Record a tool call."""
        self.call_counts[tool_name] += 1
        self.latencies[tool_name].append(duration)
        if error:
            self.error_counts[tool_name] += 1

    def get_summary(self) -> dict:
        """Get a summary of all metrics."""
        uptime = time.time() - self.start_time
        summary = {
            "uptime_seconds": round(uptime, 2),
            "tools": {}
        }

        for tool_name in self.call_counts:
            lats = self.latencies[tool_name]
            summary["tools"][tool_name] = {
                "total_calls": self.call_counts[tool_name],
                "errors": self.error_counts[tool_name],
                "avg_latency_ms": round(sum(lats) / len(lats) * 1000, 2) if lats else 0,
                "max_latency_ms": round(max(lats) * 1000, 2) if lats else 0,
                "p95_latency_ms": round(sorted(lats)[int(len(lats) * 0.95)] * 1000, 2) if lats else 0
            }

        return summary

metrics = Metrics()

# 暴露指标作为资源
@server.resource("metrics://summary")
async def get_metrics() -> str:
    """Return server metrics summary."""
    return json.dumps(metrics.get_summary(), indent=2)
```

### 健康检查

```python
@server.tool()
async def health_check() -> str:
    """Check the health status of the server and its dependencies."""
    checks = {}

    # 检查数据库连接
    try:
        await asyncio.wait_for(db_pool.execute("SELECT 1"), timeout=5.0)
        checks["database"] = {"status": "healthy"}
    except Exception as e:
        checks["database"] = {"status": "unhealthy", "error": str(e)}

    # 检查外部 API
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get("https://api.example.com/health", timeout=5.0)
            checks["external_api"] = {
                "status": "healthy" if resp.status_code == 200 else "degraded"
            }
    except Exception as e:
        checks["external_api"] = {"status": "unhealthy", "error": str(e)}

    # 检查缓存
    checks["cache"] = {
        "status": "healthy",
        "entries": len(cache.cache)
    }

    # 汇总
    all_healthy = all(c.get("status") == "healthy" for c in checks.values())
    result = {
        "status": "healthy" if all_healthy else "degraded",
        "checks": checks,
        "metrics": metrics.get_summary()
    }

    return json.dumps(result, indent=2)
```

## 最佳实践清单

### 安全性

- 对所有输入进行验证和清理
- 实施最小权限原则
- 使用环境变量存储敏感配置，不要硬编码
- 过滤输出中的敏感数据
- 实施请求频率限制
- 使用参数化查询防止注入攻击
- 定期审查和更新依赖

### 可靠性

- 为所有外部调用设置超时
- 实现优雅降级——非核心功能失败时不影响核心功能
- 使用结构化错误响应，向客户端返回有意义的错误信息
- 实现重试机制（带指数退避）
- 使用连接池管理数据库连接

### 可观测性

- 使用结构化日志记录所有工具调用
- 收集关键指标：调用次数、延迟、错误率
- 实现健康检查端点
- 暴露指标作为 MCP 资源，便于 AI 自检

### 可扩展性

- 使用缓存减少重复计算
- 实施并发控制，防止资源耗尽
- 设计无状态服务器，便于水平扩展
- 使用连接池管理外部资源

```
┌──────────────────────────────────────────┐
│          MCP 服务器生产架构               │
│                                          │
│  ┌──────────┐  ┌──────────┐             │
│  │ 输入验证  │  │ 权限控制  │  ← 安全层   │
│  └────┬─────┘  └────┬─────┘             │
│       ↓              ↓                   │
│  ┌──────────────────────────┐            │
│  │      业务逻辑层           │            │
│  │  ┌──────┐  ┌──────────┐ │            │
│  │  │ 缓存  │  │ 并发控制  │ │            │
│  │  └──────┘  └──────────┘ │            │
│  └────────────┬─────────────┘            │
│               ↓                          │
│  ┌──────────────────────────┐            │
│  │    外部资源层              │            │
│  │  ┌──────┐  ┌──────────┐ │            │
│  │  │连接池 │  │ 超时控制  │ │            │
│  │  └──────┘  └──────────┘ │            │
│  └──────────────────────────┘            │
│                                          │
│  ┌──────────────────────────┐            │
│  │   可观测性层               │            │
│  │  日志 │ 指标 │ 健康检查   │            │
│  └──────────────────────────┘            │
└──────────────────────────────────────────┘
```

---

## 关键要点

- **安全性**是第一位的：输入验证、权限控制、敏感数据过滤和频率限制缺一不可
- 使用**结构化错误响应**，在保护内部信息的同时给客户端有用的反馈
- 通过**优雅降级**确保核心功能在部分依赖失败时仍可用
- 使用**连接池、缓存和并发控制**来管理服务器资源
- **结构化日志和指标收集**是生产环境中排查问题和优化性能的关键
- 实现**健康检查**用于自动化监控和部署验证
- 遵循最佳实践清单，系统性地确保服务器的安全、可靠和可扩展

---

[← 上一课：文件系统与传输机制](./03-filesystem-and-transports.md) | [返回目录](./README.md)
