# 课时 3：API 集成

[← 上一课：配置与设置](./02-setup-and-configuration.md) | [返回目录](./README.md) | [下一课：高级功能 →](./04-advanced-features.md)

---

## 核心概念

通过 Vertex AI 调用 Claude 的最佳方式是使用 **Anthropic Python SDK** 的 Vertex 扩展。这种方式提供了与 Anthropic 直接 API 几乎一致的接口，让你可以轻松地在直接 API 和 Vertex AI 之间切换。本课时将详细讲解 API 的使用方法、请求格式、流式响应和完整代码示例。

## Anthropic Vertex SDK 基础

### 创建客户端

```python
from anthropic import AnthropicVertex

# 方式一：显式指定项目和区域
client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

# 方式二：从环境变量读取
# 需要设置 CLOUD_ML_PROJECT_ID 和 CLOUD_ML_REGION
client = AnthropicVertex()
```

### 基本请求

```python
from anthropic import AnthropicVertex

client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": "用三句话解释什么是量子计算。"
        }
    ]
)

print(message.content[0].text)
```

### 完整参数示例

```python
message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=2048,
    temperature=0.7,
    top_p=0.9,
    top_k=50,
    system="你是一个专业的技术文档编写者，使用简洁清晰的中文。",
    messages=[
        {
            "role": "user",
            "content": "帮我写一份 REST API 设计最佳实践指南。"
        }
    ],
    stop_sequences=["\n\nHuman:"]
)

print(message.content[0].text)
print(f"Model: {message.model}")
print(f"Stop reason: {message.stop_reason}")
print(f"Input tokens: {message.usage.input_tokens}")
print(f"Output tokens: {message.usage.output_tokens}")
```

### 响应对象结构

```python
# message 对象的属性
message.id              # 消息 ID
message.type            # "message"
message.role            # "assistant"
message.content         # 内容列表 [ContentBlock]
message.model           # 使用的模型
message.stop_reason     # 停止原因：end_turn, max_tokens, stop_sequence
message.usage           # token 使用量
message.usage.input_tokens   # 输入 token 数
message.usage.output_tokens  # 输出 token 数
```

## 多轮对话

```python
from anthropic import AnthropicVertex

client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

conversation = []

def chat(user_message, system_prompt=None):
    """多轮对话函数"""
    conversation.append({
        "role": "user",
        "content": user_message
    })

    kwargs = {
        "model": "claude-sonnet-4@20250514",
        "max_tokens": 2048,
        "messages": conversation
    }
    if system_prompt:
        kwargs["system"] = system_prompt

    message = client.messages.create(**kwargs)
    assistant_reply = message.content[0].text

    conversation.append({
        "role": "assistant",
        "content": assistant_reply
    })

    return assistant_reply


# 使用示例
print(chat("什么是微服务架构？"))
print()
print(chat("它和单体架构相比有什么优势？"))
print()
print(chat("在什么情况下不应该使用微服务？"))
```

## 使用 System Prompt

```python
# 方式一：字符串形式
message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=1024,
    system="你是一个资深 Python 开发者，专注于代码质量和最佳实践。回答要简洁实用。",
    messages=[
        {"role": "user", "content": "如何实现一个线程安全的单例模式？"}
    ]
)

# 方式二：结构化形式（多个 system block）
message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=1024,
    system=[
        {"type": "text", "text": "你是一个技术专家。"},
        {"type": "text", "text": "回答使用中文，代码使用英文注释。"}
    ],
    messages=[
        {"role": "user", "content": "解释 Python 的 GIL 机制。"}
    ]
)
```

## 请求格式与差异

### Vertex AI vs 直接 API 的代码对比

```python
# ===== Anthropic 直接 API =====
from anthropic import Anthropic

direct_client = Anthropic(api_key="sk-ant-...")

message = direct_client.messages.create(
    model="claude-sonnet-4-20250514",        # 直接 API 的 model ID
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)

# ===== Vertex AI =====
from anthropic import AnthropicVertex

vertex_client = AnthropicVertex(
    project_id="my-project",                  # Vertex AI 需要项目 ID
    region="us-east5"                         # Vertex AI 需要区域
)

message = vertex_client.messages.create(
    model="claude-sonnet-4@20250514",         # Vertex AI 的 model ID
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
```

### 关键差异总结

| 差异点 | Anthropic 直接 API | Vertex AI |
|--------|-------------------|-----------|
| 客户端类 | `Anthropic` | `AnthropicVertex` |
| 认证 | `api_key` 参数 | GCP 凭证（自动） |
| Model ID | `claude-sonnet-4-20250514` | `claude-sonnet-4@20250514` |
| 额外参数 | 无 | `project_id`, `region` |
| API 接口 | `client.messages.create()` | `client.messages.create()` |
| 参数格式 | 完全相同 | 完全相同 |

**核心发现：** 除了客户端初始化和 Model ID 格式之外，API 的使用方式**完全相同**。这意味着你可以轻松地在两种方式之间切换。

### 编写可切换的代码

```python
import os
from anthropic import Anthropic, AnthropicVertex

def create_client():
    """根据环境变量创建合适的客户端"""
    if os.environ.get("USE_VERTEX"):
        return AnthropicVertex(
            project_id=os.environ["CLOUD_ML_PROJECT_ID"],
            region=os.environ.get("CLOUD_ML_REGION", "us-east5")
        )
    else:
        return Anthropic(api_key=os.environ["ANTHROPIC_API_KEY"])


def get_model_id():
    """根据环境返回正确的 Model ID"""
    if os.environ.get("USE_VERTEX"):
        return "claude-sonnet-4@20250514"
    else:
        return "claude-sonnet-4-20250514"


# 使用
client = create_client()
model = get_model_id()

message = client.messages.create(
    model=model,
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)
```

## 流式响应

流式响应让你逐步接收 Claude 的输出，显著改善用户体验。

### 基本流式调用

```python
from anthropic import AnthropicVertex

client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

# 使用 stream 上下文管理器
with client.messages.stream(
    model="claude-sonnet-4@20250514",
    max_tokens=2048,
    messages=[
        {"role": "user", "content": "写一首关于编程的短诗。"}
    ]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)

print()  # 换行
```

### 完整的流式事件处理

```python
with client.messages.stream(
    model="claude-sonnet-4@20250514",
    max_tokens=2048,
    messages=[
        {"role": "user", "content": "解释 Python 的装饰器模式。"}
    ]
) as stream:
    for event in stream:
        # 可以根据事件类型做不同处理
        pass

    # 获取最终消息
    final_message = stream.get_final_message()
    print(f"\nInput tokens: {final_message.usage.input_tokens}")
    print(f"Output tokens: {final_message.usage.output_tokens}")
```

### 低级别流式 API

```python
# 使用低级别 API 处理原始事件流
stream = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=2048,
    messages=[
        {"role": "user", "content": "Hello"}
    ],
    stream=True
)

for event in stream:
    if event.type == "content_block_delta":
        if event.delta.type == "text_delta":
            print(event.delta.text, end="", flush=True)
    elif event.type == "message_start":
        print(f"[Model: {event.message.model}]")
    elif event.type == "message_delta":
        print(f"\n[Stop reason: {event.delta.stop_reason}]")
        print(f"[Output tokens: {event.usage.output_tokens}]")
```

## 图像输入（多模态）

Claude 支持处理图像输入：

```python
import base64

# 方式一：Base64 编码
with open("image.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "base64",
                        "media_type": "image/png",
                        "data": image_data
                    }
                },
                {
                    "type": "text",
                    "text": "请描述这张图片的内容。"
                }
            ]
        }
    ]
)

print(message.content[0].text)

# 方式二：URL 引用
message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=1024,
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "type": "image",
                    "source": {
                        "type": "url",
                        "url": "https://example.com/image.png"
                    }
                },
                {
                    "type": "text",
                    "text": "这张图表展示了什么趋势？"
                }
            ]
        }
    ]
)
```

## 错误处理

```python
from anthropic import AnthropicVertex, APIError, RateLimitError, APIConnectionError
import time

client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

def invoke_with_retry(messages, max_retries=3):
    """带重试的 Claude 调用"""
    for attempt in range(max_retries):
        try:
            message = client.messages.create(
                model="claude-sonnet-4@20250514",
                max_tokens=2048,
                messages=messages
            )
            return message

        except RateLimitError:
            wait_time = (2 ** attempt) + 1
            print(f"Rate limited. Retrying in {wait_time}s...")
            time.sleep(wait_time)

        except APIConnectionError as e:
            print(f"Connection error: {e}. Retrying...")
            time.sleep(2)

        except APIError as e:
            print(f"API error: {e.status_code} - {e.message}")
            if e.status_code >= 500:
                time.sleep(2)
                continue
            raise

    raise Exception(f"Failed after {max_retries} retries")


# 使用
try:
    result = invoke_with_retry([
        {"role": "user", "content": "Hello, Claude!"}
    ])
    print(result.content[0].text)
except Exception as e:
    print(f"Error: {e}")
```

## 实用客户端封装

```python
from anthropic import AnthropicVertex
from typing import List, Dict, Optional, Generator

class VertexClaude:
    """Vertex AI Claude 客户端封装"""

    def __init__(self, project_id: str, region: str = "us-east5",
                 model: str = "claude-sonnet-4@20250514"):
        self.client = AnthropicVertex(
            project_id=project_id,
            region=region
        )
        self.model = model

    def chat(self, messages: List[Dict], system: Optional[str] = None,
             max_tokens: int = 2048, temperature: float = 0.7) -> str:
        """发送对话请求，返回文本"""
        kwargs = {
            "model": self.model,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "messages": messages
        }
        if system:
            kwargs["system"] = system

        message = self.client.messages.create(**kwargs)
        return message.content[0].text

    def stream_chat(self, messages: List[Dict],
                    system: Optional[str] = None,
                    max_tokens: int = 2048) -> Generator[str, None, None]:
        """发送流式对话请求，逐块生成文本"""
        kwargs = {
            "model": self.model,
            "max_tokens": max_tokens,
            "messages": messages
        }
        if system:
            kwargs["system"] = system

        with self.client.messages.stream(**kwargs) as stream:
            for text in stream.text_stream:
                yield text

    def analyze_image(self, image_path: str, question: str) -> str:
        """分析图片并回答问题"""
        import base64

        with open(image_path, "rb") as f:
            image_data = base64.standard_b64encode(f.read()).decode("utf-8")

        # 根据文件扩展名确定媒体类型
        ext = image_path.rsplit(".", 1)[-1].lower()
        media_type_map = {
            "png": "image/png",
            "jpg": "image/jpeg",
            "jpeg": "image/jpeg",
            "gif": "image/gif",
            "webp": "image/webp"
        }
        media_type = media_type_map.get(ext, "image/png")

        message = self.client.messages.create(
            model=self.model,
            max_tokens=2048,
            messages=[
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "image",
                            "source": {
                                "type": "base64",
                                "media_type": media_type,
                                "data": image_data
                            }
                        },
                        {"type": "text", "text": question}
                    ]
                }
            ]
        )
        return message.content[0].text


# 使用示例
claude = VertexClaude(project_id="my-claude-project")

# 普通对话
reply = claude.chat([
    {"role": "user", "content": "什么是 Kubernetes？"}
])
print(reply)

# 流式对话
for chunk in claude.stream_chat([
    {"role": "user", "content": "解释 Docker 和 Kubernetes 的关系。"}
]):
    print(chunk, end="", flush=True)
print()
```

---

## 关键要点

- Anthropic Vertex SDK 提供与直接 API 几乎一致的接口，差异仅在客户端初始化和 Model ID
- Model ID 格式为 `claude-sonnet-4@20250514`（Vertex）而非 `claude-sonnet-4-20250514`（直接）
- 流式响应通过 `client.messages.stream()` 实现，支持文本流和事件流两种模式
- 支持多模态输入（图像 + 文本），使用 base64 编码或 URL
- 错误处理应包含速率限制重试、连接错误重试和服务端错误处理
- 可以编写可切换的代码，通过环境变量在直接 API 和 Vertex AI 之间切换

---

[← 上一课：配置与设置](./02-setup-and-configuration.md) | [返回目录](./README.md) | [下一课：高级功能 →](./04-advanced-features.md)
