# 课时 3：API 集成

[← 上一课：配置与设置](./02-setup-and-configuration.md) | [返回目录](./README.md) | [下一课：高级功能 →](./04-advanced-features.md)

---

## 核心概念

Amazon Bedrock 提供两种主要的 API 来调用 Claude：**InvokeModel** 和 **Converse API**。本课时将详细讲解两种 API 的差异、请求/响应格式、流式响应的实现，以及完整的 Python 代码示例。

## InvokeModel API

InvokeModel 是 Bedrock 最基础的模型调用 API。你需要按照 Anthropic 的 Messages API 格式构建请求体。

### 基本请求

```python
import boto3
import json

bedrock_runtime = boto3.client(
    'bedrock-runtime',
    region_name='us-east-1'
)

# 构建请求体（遵循 Anthropic Messages API 格式）
request_body = {
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 1024,
    "messages": [
        {
            "role": "user",
            "content": "用三句话解释什么是量子计算。"
        }
    ]
}

# 调用模型
response = bedrock_runtime.invoke_model(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps(request_body)
)

# 解析响应
result = json.loads(response['body'].read())
print(result['content'][0]['text'])
```

### 完整参数示例

```python
request_body = {
    "anthropic_version": "bedrock-2023-05-31",
    "max_tokens": 2048,
    "temperature": 0.7,
    "top_p": 0.9,
    "top_k": 50,
    "system": "你是一个专业的技术文档编写者，使用简洁清晰的中文。",
    "messages": [
        {
            "role": "user",
            "content": "帮我写一份 API 设计规范。"
        }
    ],
    "stop_sequences": ["\n\nHuman:"]
}

response = bedrock_runtime.invoke_model(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps(request_body)
)
```

### 响应格式

```json
{
    "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
    "type": "message",
    "role": "assistant",
    "content": [
        {
            "type": "text",
            "text": "量子计算是..."
        }
    ],
    "model": "claude-sonnet-4-20250514",
    "stop_reason": "end_turn",
    "usage": {
        "input_tokens": 25,
        "output_tokens": 150
    }
}
```

### 多轮对话

```python
def chat_with_claude(messages, system_prompt=None):
    """多轮对话函数"""
    request_body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2048,
        "messages": messages
    }

    if system_prompt:
        request_body["system"] = system_prompt

    response = bedrock_runtime.invoke_model(
        modelId='anthropic.claude-sonnet-4-20250514-v1:0',
        contentType='application/json',
        accept='application/json',
        body=json.dumps(request_body)
    )

    result = json.loads(response['body'].read())
    return result['content'][0]['text']


# 使用示例
conversation = []

# 第一轮
conversation.append({"role": "user", "content": "什么是 REST API？"})
reply = chat_with_claude(conversation)
conversation.append({"role": "assistant", "content": reply})
print(f"Claude: {reply}\n")

# 第二轮
conversation.append({"role": "user", "content": "它和 GraphQL 有什么区别？"})
reply = chat_with_claude(conversation)
conversation.append({"role": "assistant", "content": reply})
print(f"Claude: {reply}")
```

## Converse API

**Converse API** 是 Bedrock 提供的统一对话接口，屏蔽了不同模型提供商之间的格式差异。使用 Converse API，你可以用相同的代码调用 Claude、Titan、Llama 等模型。

### 基本请求

```python
response = bedrock_runtime.converse(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    messages=[
        {
            "role": "user",
            "content": [
                {
                    "text": "用三句话解释什么是量子计算。"
                }
            ]
        }
    ],
    inferenceConfig={
        "maxTokens": 1024,
        "temperature": 0.7
    }
)

# 解析响应
output_message = response['output']['message']
print(output_message['content'][0]['text'])
```

### 带 System Prompt 的请求

```python
response = bedrock_runtime.converse(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    messages=[
        {
            "role": "user",
            "content": [{"text": "帮我分析这段代码的性能问题。"}]
        }
    ],
    system=[
        {"text": "你是一个高级软件工程师，专注于代码性能优化。"}
    ],
    inferenceConfig={
        "maxTokens": 2048,
        "temperature": 0.3,
        "topP": 0.9
    }
)
```

### 响应格式

```json
{
    "output": {
        "message": {
            "role": "assistant",
            "content": [
                {
                    "text": "量子计算是..."
                }
            ]
        }
    },
    "usage": {
        "inputTokens": 25,
        "outputTokens": 150,
        "totalTokens": 175
    },
    "stopReason": "end_turn",
    "metrics": {
        "latencyMs": 1234
    }
}
```

## InvokeModel vs Converse API 对比

| 维度 | InvokeModel | Converse API |
|------|-------------|--------------|
| 请求格式 | 原生 Anthropic 格式 | Bedrock 统一格式 |
| 跨模型兼容 | 需要按模型调整格式 | 统一格式，切换模型只需改 ID |
| 功能覆盖 | 支持模型原生所有功能 | 覆盖大部分常用功能 |
| 工具调用 | Anthropic 格式 | Bedrock 统一格式 |
| 响应格式 | 原生 Anthropic 格式 | 统一结构化格式 |
| 推荐场景 | 需要 Claude 特有功能 | 多模型切换、标准化开发 |

**建议：** 新项目优先使用 Converse API，它更简洁且易于维护。只有在需要 Claude 特有功能时才使用 InvokeModel。

## 流式响应

流式响应让你可以逐步接收模型输出，而不是等待完整回复。这对于改善用户体验至关重要。

### InvokeModel 流式调用

```python
import json

response = bedrock_runtime.invoke_model_with_response_stream(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2048,
        "messages": [
            {
                "role": "user",
                "content": "写一首关于编程的短诗。"
            }
        ]
    })
)

# 逐块读取响应
stream = response['body']
for event in stream:
    chunk = json.loads(event['chunk']['bytes'])

    if chunk['type'] == 'content_block_delta':
        text = chunk['delta'].get('text', '')
        print(text, end='', flush=True)

    elif chunk['type'] == 'message_stop':
        print()  # 换行
```

### Converse 流式调用

```python
response = bedrock_runtime.converse_stream(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    messages=[
        {
            "role": "user",
            "content": [{"text": "写一首关于编程的短诗。"}]
        }
    ],
    inferenceConfig={
        "maxTokens": 2048
    }
)

# 逐块读取响应
stream = response['stream']
for event in stream:
    if 'contentBlockDelta' in event:
        text = event['contentBlockDelta']['delta'].get('text', '')
        print(text, end='', flush=True)

    elif 'messageStop' in event:
        print()  # 换行

    elif 'metadata' in event:
        usage = event['metadata'].get('usage', {})
        print(f"\nTokens - Input: {usage.get('inputTokens')}, "
              f"Output: {usage.get('outputTokens')}")
```

## 与 Anthropic 直接 API 的格式差异

如果你之前使用过 Anthropic 的直接 API，以下是主要差异：

### 请求差异

```python
# Anthropic 直接 API
import anthropic

client = anthropic.Anthropic(api_key="sk-...")
response = client.messages.create(
    model="claude-sonnet-4-20250514",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Hello"}]
)

# Bedrock InvokeModel（需要额外的 anthropic_version 字段）
response = bedrock_runtime.invoke_model(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",  # Bedrock 特有
        "max_tokens": 1024,
        "messages": [{"role": "user", "content": "Hello"}]
    })
)

# Bedrock Converse（完全不同的格式）
response = bedrock_runtime.converse(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    messages=[{
        "role": "user",
        "content": [{"text": "Hello"}]  # content 是列表
    }],
    inferenceConfig={"maxTokens": 1024}  # camelCase
)
```

### 关键差异总结

| 差异点 | Anthropic 直接 API | Bedrock InvokeModel | Bedrock Converse |
|--------|-------------------|---------------------|------------------|
| 认证 | API Key | AWS IAM | AWS IAM |
| Model ID | `claude-sonnet-4-20250514` | `anthropic.claude-sonnet-4-20250514-v1:0` | 同 InvokeModel |
| 版本字段 | 不需要 | `anthropic_version` 必需 | 不需要 |
| content 格式 | 字符串或列表 | 字符串或列表 | 必须是列表 |
| 参数命名 | snake_case | snake_case | camelCase |

## 错误处理与重试

### 完整的错误处理示例

```python
import boto3
import json
import time
from botocore.exceptions import ClientError

bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

def invoke_claude(messages, max_retries=3):
    """带重试的 Claude 调用函数"""
    request_body = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 2048,
        "messages": messages
    }

    for attempt in range(max_retries):
        try:
            response = bedrock_runtime.invoke_model(
                modelId='anthropic.claude-sonnet-4-20250514-v1:0',
                contentType='application/json',
                accept='application/json',
                body=json.dumps(request_body)
            )
            result = json.loads(response['body'].read())
            return result

        except ClientError as e:
            error_code = e.response['Error']['Code']

            if error_code == 'ThrottlingException':
                # 限流：指数退避重试
                wait_time = (2 ** attempt) + 1
                print(f"Rate limited. Retrying in {wait_time}s...")
                time.sleep(wait_time)
                continue

            elif error_code == 'ModelTimeoutException':
                # 超时：重试
                print(f"Timeout. Retrying (attempt {attempt + 1})...")
                continue

            elif error_code == 'AccessDeniedException':
                # 权限错误：不重试
                print("Access denied. Check IAM permissions.")
                raise

            elif error_code == 'ValidationException':
                # 请求格式错误：不重试
                print(f"Validation error: {e}")
                raise

            else:
                print(f"Unexpected error: {e}")
                raise

    raise Exception(f"Failed after {max_retries} retries")


# 使用示例
try:
    result = invoke_claude([
        {"role": "user", "content": "Hello, Claude!"}
    ])
    print(result['content'][0]['text'])
except Exception as e:
    print(f"Error: {e}")
```

## 实用工具函数

### 封装 Bedrock Claude 客户端

```python
import boto3
import json
from typing import List, Dict, Optional

class BedrockClaude:
    """Bedrock Claude 客户端封装"""

    def __init__(self, model_id='anthropic.claude-sonnet-4-20250514-v1:0',
                 region='us-east-1'):
        self.client = boto3.client('bedrock-runtime', region_name=region)
        self.model_id = model_id

    def chat(self, messages: List[Dict], system: Optional[str] = None,
             max_tokens: int = 2048, temperature: float = 0.7) -> str:
        """发送对话请求，返回文本回复"""
        response = self.client.converse(
            modelId=self.model_id,
            messages=[
                {
                    "role": m["role"],
                    "content": [{"text": m["content"]}]
                }
                for m in messages
            ],
            **({"system": [{"text": system}]} if system else {}),
            inferenceConfig={
                "maxTokens": max_tokens,
                "temperature": temperature
            }
        )
        return response['output']['message']['content'][0]['text']

    def stream_chat(self, messages: List[Dict],
                    system: Optional[str] = None,
                    max_tokens: int = 2048):
        """发送流式对话请求，逐块生成文本"""
        response = self.client.converse_stream(
            modelId=self.model_id,
            messages=[
                {
                    "role": m["role"],
                    "content": [{"text": m["content"]}]
                }
                for m in messages
            ],
            **({"system": [{"text": system}]} if system else {}),
            inferenceConfig={
                "maxTokens": max_tokens
            }
        )

        for event in response['stream']:
            if 'contentBlockDelta' in event:
                yield event['contentBlockDelta']['delta'].get('text', '')


# 使用示例
claude = BedrockClaude()

# 普通对话
reply = claude.chat([
    {"role": "user", "content": "什么是微服务架构？"}
])
print(reply)

# 流式对话
for chunk in claude.stream_chat([
    {"role": "user", "content": "解释 Docker 容器化的优势。"}
]):
    print(chunk, end='', flush=True)
```

---

## 关键要点

- Bedrock 提供两种 API：InvokeModel（原生格式）和 Converse API（统一格式）
- 新项目推荐使用 Converse API，更简洁且支持多模型切换
- InvokeModel 需要 `anthropic_version` 字段，Converse API 不需要
- 流式响应通过 `invoke_model_with_response_stream` 或 `converse_stream` 实现
- 与 Anthropic 直接 API 的主要差异：认证方式、Model ID 格式、版本字段
- 生产代码必须实现错误处理和指数退避重试

---

[← 上一课：配置与设置](./02-setup-and-configuration.md) | [返回目录](./README.md) | [下一课：高级功能 →](./04-advanced-features.md)
