# 课时 3：工具使用 (Function Calling)

[← 上一课：提示工程](./02-prompt-engineering.md) | [返回目录](./README.md) | [下一课：多模态能力 →](./04-multimodal-capabilities.md)

---

## 概述

工具使用（Tool Use），也称函数调用（Function Calling），允许 Claude 调用你预定义的函数来获取外部信息或执行操作。本课时详细讲解工具定义、调用流程、多工具协作和错误处理。

---

## 基本概念

工具使用遵循以下流程：

```
用户请求 → Claude 分析 → 工具调用请求 → 你执行函数 → 返回结果 → Claude 生成回复
```

1. **你定义工具**：在 API 请求中描述可用工具及参数
2. **Claude 决定调用**：分析用户请求后决定是否使用工具
3. **你执行工具**：收到调用请求后在你的代码中执行函数
4. **返回结果**：将执行结果返回给 Claude
5. **Claude 生成回复**：基于工具结果生成最终回复

---

## 定义工具

工具通过 JSON Schema 定义名称、功能和参数结构：

```python
import anthropic
import json

client = anthropic.Anthropic()

tools = [
    {
        "name": "get_weather",
        "description": "获取指定城市的当前天气信息。当用户询问天气时使用此工具。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，例如：北京、上海"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位，默认摄氏度"
                }
            },
            "required": ["city"]
        }
    }
]
```

> **提示**：`description` 字段非常重要，直接影响 Claude 判断何时使用该工具。应清楚说明用途、输入要求和返回内容。

---

## 完整工具调用流程

### 第一步：发送请求

```python
response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "北京今天天气怎么样？"}]
)
# response.stop_reason == "tool_use"
```

### 第二步：提取工具调用信息

```python
for block in response.content:
    if block.type == "tool_use":
        tool_name = block.name       # "get_weather"
        tool_input = block.input     # {"city": "北京"}
        tool_use_id = block.id       # "toolu_01A09q..."
```

### 第三步：执行工具并返回结果

```python
def get_weather(city: str) -> dict:
    """模拟天气查询（实际应调用真实 API）"""
    data = {
        "北京": {"temp": 22, "condition": "晴", "humidity": 45},
        "上海": {"temp": 26, "condition": "多云", "humidity": 72},
    }
    return data.get(city, {"temp": 20, "condition": "未知", "humidity": 50})

result = get_weather(tool_input["city"])

final_response = client.messages.create(
    model="claude-sonnet-4-6",
    max_tokens=1024,
    tools=tools,
    messages=[
        {"role": "user", "content": "北京今天天气怎么样？"},
        {"role": "assistant", "content": response.content},
        {
            "role": "user",
            "content": [{
                "type": "tool_result",
                "tool_use_id": tool_use_id,
                "content": json.dumps(result, ensure_ascii=False)
            }]
        }
    ]
)
print(final_response.content[0].text)
```

---

## 工具调用循环

实际应用中需构建循环处理可能的多次工具调用：

```python
def chat_with_tools(user_message: str, tools: list, tool_executor: callable) -> str:
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            return response.content[0].text

        if response.stop_reason == "tool_use":
            messages.append({"role": "assistant", "content": response.content})

            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = tool_executor(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": result
                    })
            messages.append({"role": "user", "content": tool_results})
```

---

## 多工具定义

一个请求中可定义多个工具，Claude 会根据需求选择：

```python
tools = [
    {
        "name": "search_products",
        "description": "在产品数据库中搜索商品",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"},
                "category": {"type": "string", "enum": ["electronics", "clothing", "books"]},
                "max_price": {"type": "number", "description": "最高价格（元）"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "get_product_details",
        "description": "获取指定商品的详细信息，包括库存和评价",
        "input_schema": {
            "type": "object",
            "properties": {
                "product_id": {"type": "string", "description": "商品 ID"}
            },
            "required": ["product_id"]
        }
    },
    {
        "name": "create_order",
        "description": "创建新订单",
        "input_schema": {
            "type": "object",
            "properties": {
                "product_id": {"type": "string"},
                "quantity": {"type": "integer", "minimum": 1}
            },
            "required": ["product_id", "quantity"]
        }
    }
]
```

---

## 强制使用工具

通过 `tool_choice` 参数控制工具使用行为：

```python
# 自动决定（默认）
tool_choice = {"type": "auto"}

# 强制使用任意工具
tool_choice = {"type": "any"}

# 强制使用指定工具
tool_choice = {"type": "tool", "name": "search_products"}

# 禁止使用工具
tool_choice = {"type": "none"}
```

| 选项 | 行为 |
|------|------|
| `auto` | Claude 自行决定（默认） |
| `any` | 必须使用至少一个工具 |
| `tool` + `name` | 必须使用指定工具 |
| `none` | 不使用任何工具 |

---

## 错误处理

工具执行失败时，使用 `is_error: True` 标记，让 Claude 优雅处理异常：

```python
def process_tool_call_safe(tool_name: str, tool_input: dict) -> dict:
    try:
        if tool_name == "get_weather":
            result = get_weather(tool_input["city"])
            return {
                "type": "tool_result",
                "content": json.dumps(result, ensure_ascii=False)
            }
        else:
            return {
                "type": "tool_result",
                "content": json.dumps({"error": f"未知工具: {tool_name}"}),
                "is_error": True
            }
    except Exception as e:
        return {
            "type": "tool_result",
            "content": json.dumps({"error": f"执行失败: {str(e)}"}),
            "is_error": True
        }
```

Claude 收到 `is_error: True` 后会生成适当的错误提示，例如："抱歉，目前无法查询天气信息，请稍后再试。"

---

## 实用技巧

**1. 编写清晰的工具描述**

```python
# 不好
{"name": "search", "description": "搜索"}

# 好
{"name": "search_knowledge_base",
 "description": "在公司知识库中搜索文档。支持关键词和语义搜索。返回最相关的前 10 条结果。"
               "当用户询问公司政策、操作流程或技术文档时使用。"}
```

**2. 使用枚举限制输入**：`"enum": ["low", "medium", "high"]`

**3. 设置参数约束**：`"minimum": 1, "maximum": 100`

---

## 关键要点

- 工具使用让 Claude 能与外部系统交互，极大扩展其能力
- 工具通过 JSON Schema 定义，`description` 是正确使用工具的关键
- 调用流程：定义工具 → Claude 请求调用 → 执行函数 → 返回结果 → 生成回复
- `tool_choice` 控制策略：`auto`、`any`、`tool`、`none`
- 使用 `is_error: True` 标记让 Claude 优雅处理工具执行错误
- 构建工具调用循环，处理 Claude 可能的多次连续调用

---

[← 上一课：提示工程](./02-prompt-engineering.md) | [返回目录](./README.md) | [下一课：多模态能力 →](./04-multimodal-capabilities.md)
