# 课时 1：API 基础

[返回目录](./README.md) | [下一课：提示工程 →](./02-prompt-engineering.md)

---

## 概述

本课时将带你从零开始使用 Anthropic API。你将学会如何获取 API 密钥、发起第一次 API 调用、理解 Messages API 的请求和响应格式，以及如何选择合适的模型和配置参数。

---

## 获取 API 密钥

1. 访问 [Anthropic Console](https://console.anthropic.com/) 并注册登录
2. 导航到 **API Keys** 页面，点击 **Create Key**
3. 命名密钥并复制保存

> **安全提示**：API 密钥是敏感信息，请勿提交到 Git 或在客户端代码中暴露。建议使用环境变量管理。

```bash
# Linux / macOS
export ANTHROPIC_API_KEY="sk-ant-api03-xxxxx"

# Windows (PowerShell)
$env:ANTHROPIC_API_KEY = "sk-ant-api03-xxxxx"
```

---

## 安装 SDK

```bash
# Python SDK
pip install anthropic

# TypeScript SDK
npm install @anthropic-ai/sdk
```

---

## 发起第一次 API 调用

### Python 示例

```python
import anthropic

client = anthropic.Anthropic()  # 自动读取 ANTHROPIC_API_KEY 环境变量

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "你好，Claude！请用一句话介绍你自己。"}
    ]
)

print(message.content[0].text)
```

### TypeScript 示例

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic();

async function main() {
  const message = await client.messages.create({
    model: "claude-sonnet-4-6",
    max_tokens: 1024,
    messages: [
      { role: "user", content: "你好，Claude！请用一句话介绍你自己。" },
    ],
  });
  console.log(message.content[0].text);
}

main();
```

### 使用 cURL 直接调用

```bash
curl https://api.anthropic.com/v1/messages \
  -H "content-type: application/json" \
  -H "x-api-key: $ANTHROPIC_API_KEY" \
  -H "anthropic-version: 2023-06-01" \
  -d '{
    "model": "claude-sonnet-4-6",
    "max_tokens": 1024,
    "messages": [{"role": "user", "content": "你好"}]
  }'
```

---

## 理解 Messages API

### 请求格式

```python
message = client.messages.create(
    model="claude-sonnet-4-6",          # 必填：模型名称
    max_tokens=1024,                     # 必填：最大输出 token 数
    system="你是一个专业的翻译助手。",     # 可选：系统提示词
    messages=[                           # 必填：消息列表
        {"role": "user", "content": "将以下英文翻译成中文：Hello, world!"}
    ]
)
```

### 响应格式

```json
{
    "id": "msg_01XFDUDYJgAACzvnptvVoYEL",
    "type": "message",
    "role": "assistant",
    "content": [{"type": "text", "text": "你好，世界！"}],
    "model": "claude-sonnet-4-6",
    "stop_reason": "end_turn",
    "usage": {"input_tokens": 25, "output_tokens": 8}
}
```

| 字段 | 说明 |
|------|------|
| `content` | 响应内容数组，每个元素包含 `type` 和 `text` |
| `stop_reason` | `end_turn`（正常结束）、`max_tokens`（达到上限）、`tool_use`（工具调用） |
| `usage` | Token 使用量统计，用于计算费用 |

---

## 模型选择

| 模型 | 模型 ID | 特点 | 适用场景 |
|------|---------|------|----------|
| Claude Opus 4.6 | `claude-opus-4-6` | 最强推理能力 | 高级分析、复杂编程、研究 |
| Claude Sonnet 4.6 | `claude-sonnet-4-6` | 性能与速度平衡 | 通用任务、代码生成、内容创作 |
| Claude Haiku 4.5 | `claude-haiku-4-5` | 速度最快、成本最低 | 简单分类、摘要、实时应用 |

**选择建议**：开发阶段用 Sonnet，复杂任务用 Opus，高吞吐量/低延迟用 Haiku。

---

## 核心参数配置

### max_tokens（最大输出 token 数）

控制 Claude 回复的最大长度，这是**必填参数**。如果 `stop_reason` 返回 `max_tokens`，说明回复被截断。

### temperature（温度）

控制输出的随机性，取值 `0.0` - `1.0`：

```python
# 精确模式（代码生成）：temperature=0.0
# 创意模式（故事创作）：temperature=0.8

message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    temperature=0.0,
    messages=[{"role": "user", "content": "写一个 Python 快速排序函数。"}]
)
```

### system（系统提示词）

定义 Claude 的角色、行为和限制条件，在整个对话中持续生效：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    system="你是一名资深的 Python 开发者。请用简洁的中文回答问题，并提供代码示例。",
    messages=[{"role": "user", "content": "如何使用 Python 读取 JSON 文件？"}]
)
```

### stop_sequences（停止序列）

指定字符串，当 Claude 输出包含这些字符串时立即停止生成：

```python
message = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    stop_sequences=["---END---"],
    messages=[{"role": "user", "content": "列出三种编程语言，完成后输出 ---END---"}]
)
```

---

## 错误处理

```python
import anthropic

client = anthropic.Anthropic()

try:
    message = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": "你好"}]
    )
    print(message.content[0].text)
except anthropic.AuthenticationError:
    print("认证失败：请检查你的 API 密钥是否正确。")
except anthropic.RateLimitError:
    print("请求频率过高：请稍后重试。")
except anthropic.APIError as e:
    print(f"API 错误：{e.status_code} - {e.message}")
```

| 错误码 | 说明 | 处理方式 |
|--------|------|----------|
| 401 | 认证失败 | 检查 API 密钥 |
| 429 | 速率限制 | 实现退避重试 |
| 500 | 服务器错误 | 等待后重试 |
| 529 | API 过载 | 等待后重试 |

---

## 关键要点

- 通过 [Anthropic Console](https://console.anthropic.com/) 获取 API 密钥，使用环境变量安全管理
- Messages API 是与 Claude 交互的核心接口，支持多轮对话
- 根据任务复杂度选择模型：Opus（复杂）、Sonnet（平衡）、Haiku（快速）
- `max_tokens` 是必填参数，`temperature` 控制随机性，`system` 定义角色行为
- 生产环境中必须实现完善的错误处理和重试机制

---

[返回目录](./README.md) | [下一课：提示工程 →](./02-prompt-engineering.md)
