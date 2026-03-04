# 课时 1：采样 (Sampling)

[返回目录](./README.md) | [下一课：通知与进度 →](./02-notifications-and-progress.md)

---

## 核心概念

**采样（Sampling）** 是 MCP 协议中一个强大的高级功能，它允许 MCP 服务器**反向请求客户端调用 LLM**。与普通的工具调用（客户端 → 服务器）方向相反，采样实现了服务器 → 客户端 → LLM 的请求链路。

## 什么是采样？

### 传统流程 vs 采样流程

```
传统流程（单向）：
用户 → AI 模型 → 客户端 → MCP 服务器 → 返回结果

采样流程（双向）：
用户 → AI 模型 → 客户端 → MCP 服务器
                                ↓
                         服务器需要 LLM 帮助
                                ↓
                    服务器 → 客户端 → AI 模型
                                ↓
                         LLM 返回结果给服务器
                                ↓
                    服务器继续处理 → 最终结果
```

### 为什么需要采样？

在某些场景中，MCP 服务器在处理请求时需要 LLM 的智能来完成中间步骤：

1. **多步推理**：服务器需要 LLM 分析中间结果后再继续处理
2. **内容生成**：服务器需要 LLM 生成文本（如摘要、翻译）
3. **决策辅助**：服务器需要 LLM 帮助做出决策
4. **数据分类**：服务器需要 LLM 对数据进行分类或标注

### 安全模型

采样遵循严格的安全原则——**人在回路中（Human-in-the-loop）**：

```
┌──────────────┐      采样请求      ┌──────────────┐
│  MCP 服务器   │ ───────────────► │   MCP 客户端   │
└──────────────┘                   └──────┬───────┘
                                          │
                                    ┌─────┴─────┐
                                    │  用户确认?  │ ← 客户端可以拒绝或修改请求
                                    └─────┬─────┘
                                          │ 同意
                                    ┌─────┴─────┐
                                    │  调用 LLM  │
                                    └─────┬─────┘
                                          │
                                    ┌─────┴─────┐
                                    │  返回结果   │
                                    └───────────┘
```

客户端始终拥有控制权：
- 可以**拒绝**采样请求
- 可以**修改**发送给 LLM 的消息
- 可以**过滤**或修改 LLM 的响应
- 用户可以看到并确认每个采样请求

## 实现采样处理器（客户端侧）

### 基本采样处理器

客户端需要实现一个采样处理器来响应服务器的采样请求：

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client
from mcp.types import (
    CreateMessageRequest,
    CreateMessageResult,
    TextContent,
    SamplingMessage
)
from anthropic import Anthropic

anthropic = Anthropic()

async def sampling_handler(request: CreateMessageRequest) -> CreateMessageResult:
    """Handle sampling requests from the MCP server."""

    # 将 MCP 消息转换为 Anthropic API 格式
    messages = []
    for msg in request.messages:
        content = msg.content
        if isinstance(content, TextContent):
            messages.append({
                "role": msg.role,
                "content": content.text
            })

    # 调用 LLM
    response = anthropic.messages.create(
        model=request.modelPreferences.hints[0].name if request.modelPreferences else "claude-sonnet-4-20250514",
        max_tokens=request.maxTokens or 1024,
        messages=messages,
        system=request.systemPrompt or ""
    )

    # 返回结果给服务器
    return CreateMessageResult(
        role="assistant",
        content=TextContent(
            type="text",
            text=response.content[0].text
        ),
        model=response.model
    )

async def main():
    server_params = StdioServerParameters(
        command="python",
        args=["server_with_sampling.py"]
    )

    async with stdio_client(server_params) as (read, write):
        async with ClientSession(
            read, write,
            sampling_callback=sampling_handler
        ) as session:
            await session.initialize()
            print("Connected with sampling support!")

            # 正常使用工具 - 服务器可能在处理中请求采样
            result = await session.call_tool(
                "analyze_and_summarize",
                arguments={"data_source": "sales_report.csv"}
            )
            print(f"Result: {result.content[0].text}")
```

### 带安全控制的采样处理器

在生产环境中，你应该对采样请求进行审查和过滤：

```python
async def secure_sampling_handler(request: CreateMessageRequest) -> CreateMessageResult:
    """Sampling handler with security controls."""

    # 1. 检查请求是否合理
    total_chars = sum(
        len(msg.content.text) for msg in request.messages
        if isinstance(msg.content, TextContent)
    )
    if total_chars > 100000:
        raise ValueError("Sampling request too large")

    # 2. 限制 token 数量
    max_tokens = min(request.maxTokens or 1024, 4096)

    # 3. 过滤系统提示中的敏感内容
    system_prompt = request.systemPrompt or ""
    if any(word in system_prompt.lower() for word in ["ignore previous", "override"]):
        raise ValueError("Suspicious system prompt detected")

    # 4. 记录日志
    print(f"Sampling request: {len(request.messages)} messages, max_tokens={max_tokens}")

    # 5. 调用 LLM
    messages = [
        {"role": msg.role, "content": msg.content.text}
        for msg in request.messages
        if isinstance(msg.content, TextContent)
    ]

    response = anthropic.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=max_tokens,
        messages=messages,
        system=system_prompt
    )

    return CreateMessageResult(
        role="assistant",
        content=TextContent(type="text", text=response.content[0].text),
        model=response.model
    )
```

## 实现采样请求（服务器侧）

### 在工具中请求采样

MCP 服务器可以在工具执行过程中请求客户端进行采样：

```python
from mcp.server import Server
from mcp.types import (
    SamplingMessage,
    TextContent,
    CreateMessageRequestParams,
    ModelPreferences,
    ModelHint
)

server = Server("sampling-demo")

@server.tool()
async def analyze_and_summarize(data_source: str) -> str:
    """Analyze data and generate an AI-powered summary.

    Args:
        data_source: Path to the data file to analyze
    """
    # 步骤 1：读取和处理数据
    raw_data = await read_data(data_source)
    stats = calculate_statistics(raw_data)

    # 步骤 2：请求 LLM 生成摘要（采样）
    summary_result = await server.request_context.session.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""Please summarize the following data analysis results:

Statistics:
{stats}

Raw data sample:
{raw_data[:500]}

Provide a concise business summary with key insights."""
                )
            )
        ],
        max_tokens=1024,
        system_prompt="You are a data analyst. Provide clear, actionable insights.",
        model_preferences=ModelPreferences(
            hints=[ModelHint(name="claude-sonnet-4-20250514")]
        )
    )

    # 步骤 3：结合统计数据和 AI 摘要返回结果
    return f"""## Analysis Report

### Statistics
{stats}

### AI Summary
{summary_result.content.text}
"""
```

### 多步采样

服务器可以在一个工具调用中多次请求采样：

```python
@server.tool()
async def translate_document(
    text: str,
    source_lang: str,
    target_lang: str
) -> str:
    """Translate a document with quality verification.

    Args:
        text: Text to translate
        source_lang: Source language
        target_lang: Target language
    """
    ctx = server.request_context.session

    # 步骤 1：翻译
    translation = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"Translate the following {source_lang} text to {target_lang}:\n\n{text}"
                )
            )
        ],
        max_tokens=2048,
        system_prompt=f"You are a professional translator. Translate accurately from {source_lang} to {target_lang}."
    )

    translated_text = translation.content.text

    # 步骤 2：质量审查
    review = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""Review this translation for accuracy and fluency.

Original ({source_lang}):
{text}

Translation ({target_lang}):
{translated_text}

Rate quality (1-10) and note any issues."""
                )
            )
        ],
        max_tokens=512,
        system_prompt="You are a translation quality reviewer."
    )

    return f"""## Translation Result

### Translated Text
{translated_text}

### Quality Review
{review.content.text}
"""
```

## 采样的使用场景

### 场景一：智能数据管道

```python
@server.tool()
async def process_customer_feedback(feedback_text: str) -> str:
    """Process customer feedback with AI-powered analysis."""
    ctx = server.request_context.session

    # AI 分类
    classification = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""Classify this customer feedback:
"{feedback_text}"

Categories: bug_report, feature_request, praise, complaint, question
Sentiment: positive, negative, neutral
Priority: high, medium, low

Respond in JSON format."""
                )
            )
        ],
        max_tokens=256
    )

    return classification.content.text
```

### 场景二：自动代码审查

```python
@server.tool()
async def review_code_changes(diff: str) -> str:
    """Review code changes using AI analysis."""
    ctx = server.request_context.session

    review = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""Review the following code diff:

```diff
{diff}
```

Check for:
1. Bugs or logical errors
2. Security vulnerabilities
3. Performance issues
4. Code style violations
5. Missing tests"""
                )
            )
        ],
        max_tokens=2048,
        system_prompt="You are a senior software engineer conducting a code review."
    )

    return review.content.text
```

### 场景三：自适应工作流

```python
@server.tool()
async def smart_query(question: str, database: str) -> str:
    """Answer a question by generating and executing SQL."""
    ctx = server.request_context.session

    # 获取数据库 schema
    schema = await get_db_schema(database)

    # 让 LLM 生成 SQL
    sql_result = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""Given this database schema:
{schema}

Generate a SQL query to answer: {question}

Return only the SQL query, no explanation."""
                )
            )
        ],
        max_tokens=512
    )

    sql_query = sql_result.content.text.strip()

    # 执行 SQL
    query_results = await execute_sql(database, sql_query)

    # 让 LLM 解释结果
    explanation = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(
                    type="text",
                    text=f"""Question: {question}
SQL Query: {sql_query}
Results: {query_results}

Please provide a clear, human-readable answer."""
                )
            )
        ],
        max_tokens=1024
    )

    return explanation.content.text
```

## 采样的注意事项

### 性能考虑

```python
# 避免不必要的采样调用
@server.tool()
async def efficient_tool(data: str) -> str:
    """Use sampling only when needed."""

    # 先尝试用简单逻辑处理
    if is_simple_case(data):
        return process_simple(data)

    # 只在复杂情况下使用采样
    ctx = server.request_context.session
    result = await ctx.create_message(
        messages=[
            SamplingMessage(
                role="user",
                content=TextContent(type="text", text=f"Analyze: {data}")
            )
        ],
        max_tokens=512  # 限制 token 数以控制成本
    )
    return result.content.text
```

### 错误处理

```python
@server.tool()
async def robust_sampling_tool(input_data: str) -> str:
    """Tool with robust sampling error handling."""
    ctx = server.request_context.session

    try:
        result = await ctx.create_message(
            messages=[
                SamplingMessage(
                    role="user",
                    content=TextContent(type="text", text=f"Process: {input_data}")
                )
            ],
            max_tokens=1024
        )
        return result.content.text
    except Exception as e:
        # 采样被拒绝或失败时的降级处理
        return f"AI analysis unavailable ({e}). Raw data: {input_data[:200]}"
```

---

## 关键要点

- 采样允许 MCP 服务器反向请求客户端调用 LLM，实现双向通信
- 客户端始终拥有控制权，可以拒绝、修改或过滤采样请求（人在回路中）
- 客户端通过 `sampling_callback` 参数注册采样处理器
- 服务器通过 `request_context.session.create_message()` 发起采样请求
- 典型使用场景包括：数据分析、内容生成、多步推理、智能工作流
- 使用采样时要注意性能和成本控制，只在必要时调用

---

[返回目录](./README.md) | [下一课：通知与进度 →](./02-notifications-and-progress.md)
