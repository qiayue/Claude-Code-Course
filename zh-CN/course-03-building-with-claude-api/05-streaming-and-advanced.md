# 课时 5：流式传输与高级功能

[← 上一课：多模态能力](./04-multimodal-capabilities.md) | [返回目录](./README.md) | [下一课：构建应用 →](./06-building-applications.md)

---

## 概述

本课时介绍 Claude API 的高级功能：流式传输（Streaming）实时输出、扩展思考（Extended Thinking）深度推理、批量请求（Message Batches API）批处理、速率限制与错误处理，以及提示缓存成本优化。

---

## 流式传输 (Streaming)

默认 API 等待完整响应生成后才返回。流式传输允许逐步接收内容，大幅减少用户等待时间。

### 基本流式传输

```python
import anthropic

client = anthropic.Anthropic()

with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{"role": "user", "content": "用 Python 实现一个简单的 Web 服务器。"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### Server-Sent Events (SSE)

流式传输底层使用 SSE 协议。使用原始 HTTP 请求时需设置 `"stream": true`：

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{"model": "claude-sonnet-4-6", "max_tokens": 1024, "stream": true,
       "messages": [{"role": "user", "content": "你好"}]}'
```

SSE 事件类型：

| 事件类型 | 说明 |
|----------|------|
| `message_start` | 消息开始，包含元数据 |
| `content_block_start` | 内容块开始 |
| `content_block_delta` | 内容块增量更新（文本片段） |
| `content_block_stop` | 内容块结束 |
| `message_delta` | 消息级别增量（如 stop_reason） |
| `message_stop` | 消息完成 |

### 事件处理与 Token 统计

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=2048,
    messages=[{"role": "user", "content": "解释量子计算的基本原理。"}]
) as stream:
    for event in stream:
        if event.type == "content_block_delta" and event.delta.type == "text_delta":
            print(event.delta.text, end="", flush=True)

final_message = stream.get_final_message()
print(f"\n输入 tokens: {final_message.usage.input_tokens}")
print(f"输出 tokens: {final_message.usage.output_tokens}")
```

---

## 扩展思考 (Extended Thinking)

扩展思考允许 Claude 在生成最终回复前进行深层内部推理，适合复杂分析和多步推理任务。

### 启用扩展思考

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000  # 思考过程的 token 预算
    },
    messages=[{
        "role": "user",
        "content": "分析这个算法的时间复杂度并证明其正确性：\n"
                   "def mystery(arr):\n"
                   "    n = len(arr)\n"
                   "    for i in range(n):\n"
                   "        for j in range(0, n-i-1):\n"
                   "            if arr[j] > arr[j+1]:\n"
                   "                arr[j], arr[j+1] = arr[j+1], arr[j]"
    }]
)

for block in response.content:
    if block.type == "thinking":
        print("=== 思考过程 ===")
        print(block.thinking)
    elif block.type == "text":
        print("=== 最终回复 ===")
        print(block.text)
```

**关键参数**：`thinking.budget_tokens` 为思考预算上限；`max_tokens` 必须大于 `budget_tokens`。

**适用场景**：数学证明、代码调试、策略分析、复杂逻辑推理。

### 流式 + 扩展思考

```python
with client.messages.stream(
    model="claude-sonnet-4-6",
    max_tokens=16000,
    thinking={"type": "enabled", "budget_tokens": 8000},
    messages=[{"role": "user", "content": "设计一个分布式缓存系统架构。"}]
) as stream:
    for event in stream:
        if event.type == "content_block_start":
            block_type = getattr(event.content_block, "type", "")
            if block_type == "thinking":
                print("\n[思考中...]")
            elif block_type == "text":
                print("\n[回复]")
        elif event.type == "content_block_delta":
            if event.delta.type == "thinking_delta":
                print(event.delta.thinking, end="", flush=True)
            elif event.delta.type == "text_delta":
                print(event.delta.text, end="", flush=True)
```

---

## 批量请求 (Message Batches API)

处理大量互不依赖的请求时，批量 API 可异步处理，享受 **50% 价格折扣**。

### 创建批量请求

```python
requests = [
    {
        "custom_id": f"review-{i:03d}",
        "params": {
            "model": "claude-sonnet-4-6",
            "max_tokens": 1024,
            "messages": [{"role": "user", "content": f"审查代码：{code}"}]
        }
    }
    for i, code in enumerate(code_snippets)
]

batch = client.messages.batches.create(requests=requests)
print(f"批量任务 ID: {batch.id}，状态: {batch.processing_status}")
```

### 查询状态与获取结果

```python
batch_status = client.messages.batches.retrieve(batch.id)

if batch_status.processing_status == "ended":
    for result in client.messages.batches.results(batch.id):
        if result.result.type == "succeeded":
            print(f"{result.custom_id}: {result.result.message.content[0].text[:100]}...")
        else:
            print(f"{result.custom_id}: 错误 - {result.result.error}")
```

**适用场景**：大规模文档分类、批量代码审查、数据标注、不需实时响应的后台任务。

---

## 速率限制与错误处理

### 速率限制维度

| 维度 | 说明 |
|------|------|
| RPM | 每分钟请求次数 |
| TPM | 每分钟输入 token 数 |
| TPD | 每天输入 token 数 |

### 指数退避重试

```python
import time
import random

def call_with_retry(messages: list, max_retries: int = 5, base_delay: float = 1.0):
    for attempt in range(max_retries):
        try:
            return client.messages.create(
                model="claude-sonnet-4-6", max_tokens=1024, messages=messages
            )
        except anthropic.RateLimitError:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"速率限制，等待 {delay:.1f}s 重试（第 {attempt + 1} 次）...")
            time.sleep(delay)
        except anthropic.APIStatusError as e:
            if e.status_code in (500, 529) and attempt < max_retries - 1:
                time.sleep(base_delay * (2 ** attempt))
            else:
                raise
```

### SDK 内置重试与响应头

```python
# SDK 自动重试 429、500、529 错误
client = anthropic.Anthropic(max_retries=3, timeout=60.0)

# 查看速率限制信息
response = client.messages.with_raw_response.create(
    model="claude-sonnet-4-6", max_tokens=1024,
    messages=[{"role": "user", "content": "你好"}]
)
print(f"剩余请求: {response.headers.get('anthropic-ratelimit-requests-remaining')}")
print(f"剩余 Token: {response.headers.get('anthropic-ratelimit-tokens-remaining')}")
```

---

## 提示缓存成本优化

### 优化缓存命中率

将长且稳定的上下文放在前面并标记缓存，变化内容放在后面：

```python
def query_with_context(context: str, question: str) -> str:
    response = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=2048,
        system=[{
            "type": "text",
            "text": context,
            "cache_control": {"type": "ephemeral"}
        }],
        messages=[{"role": "user", "content": question}]
    )
    return response.content[0].text

# 多次查询共享同一个缓存上下文
answer1 = query_with_context(codebase_context, "这个项目使用了哪些设计模式？")
answer2 = query_with_context(codebase_context, "数据库模型有哪些关联关系？")
```

### 成本估算

10,000 token 系统提示词，每天 1,000 次请求：

- 不使用缓存：10,000 x 1,000 x 输入价格 = 基准成本
- 使用缓存：首次写入 x 1.25 + 后续 999 次 x 0.1 ≈ 基准的 ~10%
- **节省约 90% 输入成本**

---

## 关键要点

- 流式传输使用 SSE 逐步返回内容，显著改善用户体验
- 扩展思考为 Claude 提供内部推理空间，适合复杂分析和推理任务
- Message Batches API 适合大规模异步处理，享受 50% 价格折扣
- 指数退避重试策略是生产环境必备实践
- 关注响应头中的速率限制信息，合理规划请求频率
- 提示缓存可节省高达 90% 的输入成本

---

[← 上一课：多模态能力](./04-multimodal-capabilities.md) | [返回目录](./README.md) | [下一课：构建应用 →](./06-building-applications.md)
