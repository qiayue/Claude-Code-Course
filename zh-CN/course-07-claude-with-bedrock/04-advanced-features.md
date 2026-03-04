# 课时 4：高级功能

[← 上一课：API 集成](./03-api-integration.md) | [返回目录](./README.md)

---

## 核心概念

Amazon Bedrock 不仅提供基础的模型调用，还提供了一系列企业级高级功能。本课时将讲解通过 Bedrock 使用 Claude 的工具调用（Tool Use）、知识库（Knowledge Bases）、Bedrock Agents、Guardrails 以及生产部署的最佳实践。

## 工具调用（Tool Use）

通过 Bedrock 使用 Claude 的工具调用功能，让 Claude 能够与外部系统交互。

### 使用 Converse API 的工具调用

```python
import boto3
import json

bedrock_runtime = boto3.client('bedrock-runtime', region_name='us-east-1')

# 定义工具
tool_config = {
    "tools": [
        {
            "toolSpec": {
                "name": "get_weather",
                "description": "获取指定城市的当前天气信息",
                "inputSchema": {
                    "json": {
                        "type": "object",
                        "properties": {
                            "city": {
                                "type": "string",
                                "description": "城市名称，如：北京、上海"
                            },
                            "unit": {
                                "type": "string",
                                "enum": ["celsius", "fahrenheit"],
                                "description": "温度单位"
                            }
                        },
                        "required": ["city"]
                    }
                }
            }
        }
    ]
}

# 发送请求
response = bedrock_runtime.converse(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    messages=[
        {
            "role": "user",
            "content": [{"text": "北京今天天气怎么样？"}]
        }
    ],
    toolConfig=tool_config,
    inferenceConfig={"maxTokens": 1024}
)
```

### 处理工具调用响应

```python
def process_tool_call(response):
    """处理 Claude 的工具调用"""
    output = response['output']['message']
    stop_reason = response['stopReason']

    if stop_reason == 'tool_use':
        # Claude 请求调用工具
        for content in output['content']:
            if 'toolUse' in content:
                tool_use = content['toolUse']
                tool_name = tool_use['name']
                tool_input = tool_use['input']
                tool_use_id = tool_use['toolUseId']

                print(f"Claude wants to call: {tool_name}")
                print(f"With input: {json.dumps(tool_input, ensure_ascii=False)}")

                return tool_name, tool_input, tool_use_id

    return None, None, None


# 模拟工具执行
def execute_tool(tool_name, tool_input):
    """执行工具并返回结果"""
    if tool_name == "get_weather":
        # 这里应该调用真实的天气 API
        return {
            "city": tool_input["city"],
            "temperature": 22,
            "condition": "晴",
            "humidity": 45
        }
    return {"error": "Unknown tool"}


# 完整的工具调用循环
def chat_with_tools(user_message, tool_config):
    """支持工具调用的对话函数"""
    messages = [
        {"role": "user", "content": [{"text": user_message}]}
    ]

    while True:
        response = bedrock_runtime.converse(
            modelId='anthropic.claude-sonnet-4-20250514-v1:0',
            messages=messages,
            toolConfig=tool_config,
            inferenceConfig={"maxTokens": 1024}
        )

        output_message = response['output']['message']
        messages.append(output_message)

        if response['stopReason'] == 'tool_use':
            # 处理工具调用
            tool_results = []
            for content in output_message['content']:
                if 'toolUse' in content:
                    tool_use = content['toolUse']
                    result = execute_tool(tool_use['name'], tool_use['input'])
                    tool_results.append({
                        "toolResult": {
                            "toolUseId": tool_use['toolUseId'],
                            "content": [{"json": result}]
                        }
                    })

            # 将工具结果发回给 Claude
            messages.append({
                "role": "user",
                "content": tool_results
            })
        else:
            # Claude 完成回复
            final_text = output_message['content'][0]['text']
            return final_text


# 使用示例
result = chat_with_tools("北京今天天气怎么样？", tool_config)
print(result)
```

## Bedrock Knowledge Bases（知识库 RAG）

Knowledge Bases 让你可以将 Claude 与你的私有数据连接，实现检索增强生成（RAG）。

### 知识库工作原理

```
┌──────────────────────────────────────┐
│           用户查询                     │
├──────────────────────────────────────┤
│        Bedrock Knowledge Base          │
│  ┌──────────┐    ┌──────────────┐    │
│  │ 向量检索   │ →  │ Claude 生成   │    │
│  └──────────┘    └──────────────┘    │
├──────────────────────────────────────┤
│          数据源                        │
│   S3 文档  │  网页  │  数据库          │
└──────────────────────────────────────┘
```

### 创建知识库（通过控制台）

1. 在 Bedrock 控制台选择 **Knowledge bases**
2. 点击 **Create knowledge base**
3. 配置数据源（如 S3 存储桶）
4. 选择嵌入模型（如 Amazon Titan Embedding）
5. 选择向量存储（如 OpenSearch Serverless）
6. 上传文档并同步

### 通过 API 查询知识库

```python
bedrock_agent_runtime = boto3.client(
    'bedrock-agent-runtime',
    region_name='us-east-1'
)

# 检索并生成（RAG）
response = bedrock_agent_runtime.retrieve_and_generate(
    input={
        "text": "我们的退款政策是什么？"
    },
    retrieveAndGenerateConfiguration={
        "type": "KNOWLEDGE_BASE",
        "knowledgeBaseConfiguration": {
            "knowledgeBaseId": "YOUR_KB_ID",
            "modelArn": "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-sonnet-4-20250514-v1:0"
        }
    }
)

print(response['output']['text'])

# 查看引用来源
for citation in response.get('citations', []):
    for ref in citation.get('retrievedReferences', []):
        print(f"Source: {ref['location']['s3Location']['uri']}")
```

### 仅检索（不生成）

```python
# 仅检索相关文档片段
response = bedrock_agent_runtime.retrieve(
    knowledgeBaseId="YOUR_KB_ID",
    retrievalQuery={
        "text": "退款政策"
    },
    retrievalConfiguration={
        "vectorSearchConfiguration": {
            "numberOfResults": 5
        }
    }
)

for result in response['retrievalResults']:
    print(f"Score: {result['score']:.4f}")
    print(f"Content: {result['content']['text'][:200]}")
    print(f"Source: {result['location']['s3Location']['uri']}")
    print()
```

## Bedrock Agents

Bedrock Agents 是一个更高层次的抽象，让 Claude 能够自主执行多步骤任务，包括调用 API、查询数据库和使用工具。

### Agent 的工作原理

```
用户请求 → Agent 理解意图
              ↓
         制定执行计划
              ↓
      ┌───────┼───────┐
      ↓       ↓       ↓
   查询KB   调用API  执行操作
      ↓       ↓       ↓
      └───────┼───────┘
              ↓
         整合结果
              ↓
         返回回答
```

### 调用 Agent

```python
# 调用已创建的 Bedrock Agent
response = bedrock_agent_runtime.invoke_agent(
    agentId="YOUR_AGENT_ID",
    agentAliasId="YOUR_ALIAS_ID",
    sessionId="session-123",
    inputText="帮我查询订单 #12345 的状态，如果已发货请给客户发确认邮件"
)

# 处理流式响应
event_stream = response['completion']
for event in event_stream:
    if 'chunk' in event:
        chunk = event['chunk']
        text = chunk['bytes'].decode('utf-8')
        print(text, end='')
```

### Agent 的关键组件

| 组件 | 说明 |
|------|------|
| **Foundation Model** | 底层 LLM（Claude） |
| **Instructions** | Agent 的行为指令 |
| **Action Groups** | Agent 可以执行的操作（Lambda 函数） |
| **Knowledge Bases** | Agent 可以查询的知识库 |
| **Guardrails** | 安全和合规约束 |

## Guardrails

Bedrock Guardrails 让你可以为 Claude 的输入和输出设置安全护栏，防止不当内容的生成。

### Guardrails 的功能

| 功能 | 说明 |
|------|------|
| **内容过滤** | 过滤仇恨、暴力、性等不当内容 |
| **话题屏蔽** | 阻止讨论特定敏感话题 |
| **敏感信息过滤** | 自动识别和屏蔽 PII（个人身份信息） |
| **词汇过滤** | 屏蔽特定词汇或短语 |
| **上下文接地** | 确保回答基于提供的上下文，减少幻觉 |

### 使用 Guardrails

```python
# 在模型调用中应用 Guardrails
response = bedrock_runtime.converse(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    messages=[
        {
            "role": "user",
            "content": [{"text": "用户的问题..."}]
        }
    ],
    guardrailConfig={
        "guardrailIdentifier": "YOUR_GUARDRAIL_ID",
        "guardrailVersion": "1"
    },
    inferenceConfig={"maxTokens": 1024}
)

# 检查是否被 Guardrail 拦截
if response.get('stopReason') == 'guardrail_intervened':
    print("Response was filtered by guardrails")
    # 处理被拦截的情况
else:
    print(response['output']['message']['content'][0]['text'])
```

### 创建 Guardrail（通过 API）

```python
bedrock = boto3.client('bedrock', region_name='us-east-1')

response = bedrock.create_guardrail(
    name='customer-service-guardrail',
    description='客户服务场景的安全护栏',
    topicPolicyConfig={
        "topicsConfig": [
            {
                "name": "competitor-discussion",
                "definition": "讨论或推荐竞争对手产品",
                "examples": [
                    "你们的产品不如 X 公司的好",
                    "推荐一些其他品牌的产品"
                ],
                "type": "DENY"
            }
        ]
    },
    contentPolicyConfig={
        "filtersConfig": [
            {
                "type": "VIOLENCE",
                "inputStrength": "HIGH",
                "outputStrength": "HIGH"
            },
            {
                "type": "HATE",
                "inputStrength": "HIGH",
                "outputStrength": "HIGH"
            }
        ]
    },
    sensitiveInformationPolicyConfig={
        "piiEntitiesConfig": [
            {"type": "EMAIL", "action": "ANONYMIZE"},
            {"type": "PHONE", "action": "ANONYMIZE"},
            {"type": "CREDIT_DEBIT_CARD_NUMBER", "action": "BLOCK"}
        ]
    },
    blockedInputMessaging="抱歉，我无法处理此类请求。",
    blockedOutputMessaging="抱歉，我无法提供此类信息。"
)

guardrail_id = response['guardrailId']
print(f"Guardrail created: {guardrail_id}")
```

## 生产部署最佳实践

### 1. 架构设计

```
┌─────────┐     ┌─────────────┐     ┌──────────┐
│  客户端   │ →  │  API Gateway │ →  │  Lambda   │
└─────────┘     └─────────────┘     └────┬─────┘
                                         │
                                    ┌────┴─────┐
                                    │ Bedrock   │
                                    │ (Claude)  │
                                    └──────────┘
```

### 2. Lambda 集成示例

```python
import boto3
import json

bedrock_runtime = boto3.client('bedrock-runtime')

def lambda_handler(event, context):
    """Lambda 函数处理 Claude 请求"""
    user_message = event.get('body', {}).get('message', '')

    if not user_message:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Message is required'})
        }

    try:
        response = bedrock_runtime.converse(
            modelId='anthropic.claude-sonnet-4-20250514-v1:0',
            messages=[
                {
                    "role": "user",
                    "content": [{"text": user_message}]
                }
            ],
            inferenceConfig={"maxTokens": 2048}
        )

        reply = response['output']['message']['content'][0]['text']
        usage = response['usage']

        return {
            'statusCode': 200,
            'body': json.dumps({
                'reply': reply,
                'usage': {
                    'inputTokens': usage['inputTokens'],
                    'outputTokens': usage['outputTokens']
                }
            }, ensure_ascii=False)
        }

    except Exception as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

### 3. 监控与可观测性

```python
import boto3

cloudwatch = boto3.client('cloudwatch')

# 自定义指标：跟踪 token 使用量
def track_usage(input_tokens, output_tokens, model_id):
    cloudwatch.put_metric_data(
        Namespace='Bedrock/Claude',
        MetricData=[
            {
                'MetricName': 'InputTokens',
                'Value': input_tokens,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'ModelId', 'Value': model_id}
                ]
            },
            {
                'MetricName': 'OutputTokens',
                'Value': output_tokens,
                'Unit': 'Count',
                'Dimensions': [
                    {'Name': 'ModelId', 'Value': model_id}
                ]
            }
        ]
    )
```

### 4. 成本控制

```python
# 设置每请求的 token 上限
MAX_INPUT_TOKENS = 4096
MAX_OUTPUT_TOKENS = 2048

def controlled_invoke(messages, max_output=MAX_OUTPUT_TOKENS):
    """带成本控制的调用"""
    # 估算输入 token 数（粗略估计）
    total_chars = sum(len(m['content']) for m in messages)
    estimated_tokens = total_chars // 4  # 大约 4 字符 = 1 token

    if estimated_tokens > MAX_INPUT_TOKENS:
        raise ValueError(f"Input too large: ~{estimated_tokens} tokens "
                        f"(max: {MAX_INPUT_TOKENS})")

    response = bedrock_runtime.converse(
        modelId='anthropic.claude-sonnet-4-20250514-v1:0',
        messages=[
            {"role": m["role"], "content": [{"text": m["content"]}]}
            for m in messages
        ],
        inferenceConfig={"maxTokens": max_output}
    )

    usage = response['usage']
    track_usage(usage['inputTokens'], usage['outputTokens'],
                'anthropic.claude-sonnet-4-20250514-v1:0')

    return response
```

### 5. 高可用设计

```python
# 多区域回退
REGIONS = ['us-east-1', 'us-west-2']

def invoke_with_failover(messages):
    """多区域回退调用"""
    for region in REGIONS:
        try:
            client = boto3.client('bedrock-runtime', region_name=region)
            response = client.converse(
                modelId='anthropic.claude-sonnet-4-20250514-v1:0',
                messages=[
                    {"role": m["role"], "content": [{"text": m["content"]}]}
                    for m in messages
                ],
                inferenceConfig={"maxTokens": 2048}
            )
            return response
        except Exception as e:
            print(f"Region {region} failed: {e}")
            continue

    raise Exception("All regions failed")
```

---

## 关键要点

- 工具调用（Tool Use）让 Claude 能够与外部系统交互，通过 Converse API 实现格式统一
- Knowledge Bases 提供开箱即用的 RAG 功能，支持 S3 文档、向量检索和自动引用
- Bedrock Agents 实现多步骤自主任务执行，结合 Action Groups 和 Knowledge Bases
- Guardrails 提供输入/输出安全防护，包括内容过滤、话题屏蔽和 PII 检测
- 生产部署建议使用 Lambda + API Gateway 架构，配合 CloudWatch 监控
- 成本控制通过 token 上限、模型选择和使用量追踪实现

---

[← 上一课：API 集成](./03-api-integration.md) | [返回目录](./README.md)
