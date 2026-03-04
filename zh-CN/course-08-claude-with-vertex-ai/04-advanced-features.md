# 课时 4：高级功能

[← 上一课：API 集成](./03-api-integration.md) | [返回目录](./README.md)

---

## 核心概念

Google Vertex AI 不仅提供基础的 Claude 调用，还提供了一系列增值功能和与 GCP 生态的深度集成。本课时将讲解工具调用（Tool Use）、RAG 集成、Google Search Grounding 以及生产部署的最佳实践。

## 工具调用（Tool Use）

通过 Vertex AI 使用 Claude 的工具调用功能，接口与 Anthropic 直接 API 一致。

### 定义工具

```python
from anthropic import AnthropicVertex

client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

# 定义工具
tools = [
    {
        "name": "get_weather",
        "description": "获取指定城市的当前天气信息。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，如：北京、上海、东京"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位，默认为 celsius"
                }
            },
            "required": ["city"]
        }
    },
    {
        "name": "search_database",
        "description": "在产品数据库中搜索商品信息。",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词"
                },
                "category": {
                    "type": "string",
                    "description": "商品类别"
                }
            },
            "required": ["query"]
        }
    }
]
```

### 工具调用循环

```python
def execute_tool(name, input_data):
    """模拟工具执行"""
    if name == "get_weather":
        return {"city": input_data["city"], "temp": 22, "condition": "晴天"}
    elif name == "search_database":
        return {"results": [{"name": "示例商品", "price": 99.9}]}
    return {"error": "Unknown tool"}


def chat_with_tools(user_message):
    """支持工具调用的对话"""
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-sonnet-4@20250514",
            max_tokens=2048,
            tools=tools,
            messages=messages
        )

        # 收集助手回复
        messages.append({
            "role": "assistant",
            "content": response.content
        })

        if response.stop_reason == "tool_use":
            # 处理工具调用
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    print(f"Calling tool: {block.name}")
                    print(f"Input: {block.input}")

                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })

            # 发送工具结果
            messages.append({
                "role": "user",
                "content": tool_results
            })
        else:
            # Claude 完成回复
            for block in response.content:
                if hasattr(block, 'text'):
                    return block.text


# 使用示例
result = chat_with_tools("北京今天天气怎么样？")
print(result)
```

### 流式工具调用

```python
def stream_chat_with_tools(user_message):
    """流式工具调用"""
    messages = [{"role": "user", "content": user_message}]

    while True:
        # 使用流式 API
        with client.messages.stream(
            model="claude-sonnet-4@20250514",
            max_tokens=2048,
            tools=tools,
            messages=messages
        ) as stream:
            # 收集完整消息
            response = stream.get_final_message()

        messages.append({
            "role": "assistant",
            "content": response.content
        })

        if response.stop_reason == "tool_use":
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })

            messages.append({
                "role": "user",
                "content": tool_results
            })
        else:
            for block in response.content:
                if hasattr(block, 'text'):
                    return block.text
```

## RAG 集成

虽然 Vertex AI 不像 Bedrock 那样提供开箱即用的 Knowledge Bases，但你可以通过 Vertex AI RAG API 或自定义方案实现 RAG。

### 方案一：Vertex AI Search（推荐）

使用 Vertex AI Search 作为检索引擎，结合 Claude 生成回答：

```python
from google.cloud import discoveryengine_v1 as discoveryengine

def search_documents(query, project_id, datastore_id):
    """使用 Vertex AI Search 检索文档"""
    client = discoveryengine.SearchServiceClient()

    serving_config = (
        f"projects/{project_id}/locations/global/"
        f"dataStores/{datastore_id}/servingConfigs/default_search"
    )

    request = discoveryengine.SearchRequest(
        serving_config=serving_config,
        query=query,
        page_size=5
    )

    response = client.search(request=request)

    results = []
    for result in response.results:
        doc = result.document
        results.append({
            "title": doc.derived_struct_data.get("title", ""),
            "snippet": doc.derived_struct_data.get("snippet", ""),
            "link": doc.derived_struct_data.get("link", "")
        })

    return results


def rag_chat(question, project_id, datastore_id):
    """RAG 模式对话"""
    # 步骤 1：检索相关文档
    search_results = search_documents(question, project_id, datastore_id)

    # 步骤 2：构建带上下文的提示
    context = "\n\n".join([
        f"**{r['title']}**: {r['snippet']}"
        for r in search_results
    ])

    message = client.messages.create(
        model="claude-sonnet-4@20250514",
        max_tokens=2048,
        system="你是一个知识助手。根据提供的参考资料回答问题。如果参考资料中没有相关信息，请明确说明。",
        messages=[
            {
                "role": "user",
                "content": f"参考资料：\n{context}\n\n问题：{question}"
            }
        ]
    )

    return message.content[0].text
```

### 方案二：自定义向量检索

使用 Cloud SQL（pgvector）或其他向量数据库：

```python
import numpy as np
from google.cloud import aiplatform

def get_embedding(text, project_id, region="us-central1"):
    """使用 Vertex AI 获取文本嵌入"""
    aiplatform.init(project=project_id, location=region)

    model = aiplatform.TextEmbeddingModel.from_pretrained(
        "text-embedding-004"
    )

    embeddings = model.get_embeddings([text])
    return embeddings[0].values


def vector_search(query_embedding, stored_embeddings, documents, top_k=5):
    """简单的向量相似度搜索"""
    query_vec = np.array(query_embedding)
    similarities = []

    for i, emb in enumerate(stored_embeddings):
        similarity = np.dot(query_vec, np.array(emb)) / (
            np.linalg.norm(query_vec) * np.linalg.norm(np.array(emb))
        )
        similarities.append((i, similarity))

    similarities.sort(key=lambda x: x[1], reverse=True)
    return [documents[i] for i, _ in similarities[:top_k]]


def custom_rag(question, documents, embeddings, project_id):
    """自定义 RAG 流程"""
    # 获取查询嵌入
    query_embedding = get_embedding(question, project_id)

    # 检索相关文档
    relevant_docs = vector_search(query_embedding, embeddings, documents)

    # 构建上下文
    context = "\n\n---\n\n".join(relevant_docs)

    # 调用 Claude 生成回答
    message = client.messages.create(
        model="claude-sonnet-4@20250514",
        max_tokens=2048,
        system="基于提供的参考文档回答问题。引用具体的文档内容支持你的回答。",
        messages=[
            {
                "role": "user",
                "content": f"参考文档：\n{context}\n\n问题：{question}"
            }
        ]
    )

    return message.content[0].text
```

## Grounding with Google Search

Google Search Grounding 是 Vertex AI 的独有功能，让 Claude 可以访问 Google 搜索结果来获取最新信息。

### 使用 Google Search Grounding

```python
from google.cloud import aiplatform
import vertexai
from vertexai.generative_models import GenerativeModel, Tool, grounding

# 注意：Google Search Grounding 目前主要通过 Google SDK 使用
# 以下为使用 Vertex AI Python SDK 的方式

vertexai.init(project="my-claude-project", location="us-east5")

# 定义 Google Search 工具
google_search_tool = Tool.from_google_search_retrieval(
    grounding.GoogleSearchRetrieval()
)

# 注意：Grounding 功能的可用性取决于模型和区域
# 请查看最新文档了解 Claude 模型的 Grounding 支持情况
```

### 手动实现搜索增强

如果原生 Grounding 不可用，可以通过工具调用实现类似功能：

```python
import requests

def google_search(query, api_key, cx):
    """使用 Google Custom Search API"""
    url = "https://www.googleapis.com/customsearch/v1"
    params = {
        "key": api_key,
        "cx": cx,
        "q": query,
        "num": 5
    }
    response = requests.get(url, params=params)
    results = response.json().get("items", [])

    return [
        {
            "title": item["title"],
            "snippet": item["snippet"],
            "link": item["link"]
        }
        for item in results
    ]


# 将搜索定义为 Claude 的工具
search_tool = {
    "name": "google_search",
    "description": "搜索互联网获取最新信息。当需要实时数据或最新事件时使用。",
    "input_schema": {
        "type": "object",
        "properties": {
            "query": {
                "type": "string",
                "description": "搜索查询词"
            }
        },
        "required": ["query"]
    }
}

# 在工具调用循环中使用
def execute_search_tool(name, input_data):
    if name == "google_search":
        results = google_search(
            input_data["query"],
            api_key="YOUR_API_KEY",
            cx="YOUR_SEARCH_ENGINE_ID"
        )
        return {"search_results": results}
```

## 生产部署模式

### 模式一：Cloud Run 部署

```python
# app.py - Cloud Run 服务
from flask import Flask, request, jsonify
from anthropic import AnthropicVertex
import os

app = Flask(__name__)

client = AnthropicVertex(
    project_id=os.environ["GCP_PROJECT_ID"],
    region=os.environ.get("VERTEX_REGION", "us-east5")
)

@app.route("/chat", methods=["POST"])
def chat():
    data = request.get_json()
    user_message = data.get("message", "")

    if not user_message:
        return jsonify({"error": "Message is required"}), 400

    try:
        message = client.messages.create(
            model="claude-sonnet-4@20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": user_message}]
        )

        return jsonify({
            "reply": message.content[0].text,
            "usage": {
                "input_tokens": message.usage.input_tokens,
                "output_tokens": message.usage.output_tokens
            }
        })

    except Exception as e:
        return jsonify({"error": str(e)}), 500


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```

部署到 Cloud Run：

```bash
# Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 app:app
EOF

# requirements.txt
cat > requirements.txt << 'EOF'
flask==3.0.0
gunicorn==21.2.0
anthropic[vertex]==0.40.0
EOF

# 部署
gcloud run deploy claude-api \
  --source . \
  --region us-east5 \
  --set-env-vars GCP_PROJECT_ID=my-claude-project \
  --service-account claude-vertex-sa@my-claude-project.iam.gserviceaccount.com \
  --allow-unauthenticated
```

### 模式二：Cloud Functions 部署

```python
# main.py - Cloud Function
import functions_framework
from anthropic import AnthropicVertex
import os
import json

client = AnthropicVertex(
    project_id=os.environ["GCP_PROJECT_ID"],
    region="us-east5"
)

@functions_framework.http
def claude_chat(request):
    """HTTP Cloud Function"""
    # CORS 处理
    if request.method == "OPTIONS":
        headers = {
            "Access-Control-Allow-Origin": "*",
            "Access-Control-Allow-Methods": "POST",
            "Access-Control-Allow-Headers": "Content-Type"
        }
        return ("", 204, headers)

    data = request.get_json(silent=True)
    if not data or "message" not in data:
        return json.dumps({"error": "Message is required"}), 400

    try:
        message = client.messages.create(
            model="claude-sonnet-4@20250514",
            max_tokens=2048,
            messages=[{"role": "user", "content": data["message"]}]
        )

        return json.dumps({
            "reply": message.content[0].text,
            "usage": {
                "input_tokens": message.usage.input_tokens,
                "output_tokens": message.usage.output_tokens
            }
        }, ensure_ascii=False)

    except Exception as e:
        return json.dumps({"error": str(e)}), 500
```

部署：

```bash
gcloud functions deploy claude-chat \
  --gen2 \
  --runtime python311 \
  --trigger-http \
  --entry-point claude_chat \
  --region us-east5 \
  --set-env-vars GCP_PROJECT_ID=my-claude-project \
  --service-account claude-vertex-sa@my-claude-project.iam.gserviceaccount.com
```

### 模式三：GKE 部署

适合高负载、需要精细控制的场景：

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: claude-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: claude-api
  template:
    metadata:
      labels:
        app: claude-api
    spec:
      serviceAccountName: claude-vertex-ksa
      containers:
      - name: claude-api
        image: gcr.io/my-claude-project/claude-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: GCP_PROJECT_ID
          value: "my-claude-project"
        - name: VERTEX_REGION
          value: "us-east5"
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## 监控与可观测性

### Cloud Monitoring 集成

```python
from google.cloud import monitoring_v3
import time

def track_claude_metrics(project_id, input_tokens, output_tokens,
                         latency_ms, model_id):
    """发送自定义指标到 Cloud Monitoring"""
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"

    # 创建时间序列数据
    series = monitoring_v3.TimeSeries()
    series.metric.type = "custom.googleapis.com/claude/token_usage"
    series.metric.labels["model"] = model_id
    series.resource.type = "global"

    now = time.time()
    interval = monitoring_v3.TimeInterval(
        {"end_time": {"seconds": int(now)}}
    )
    point = monitoring_v3.Point({
        "interval": interval,
        "value": {"int64_value": input_tokens + output_tokens}
    })
    series.points = [point]

    client.create_time_series(
        request={"name": project_name, "time_series": [series]}
    )
```

### 结构化日志

```python
import json
import logging

# Cloud Logging 兼容的结构化日志
class StructuredLogFormatter(logging.Formatter):
    def format(self, record):
        log_entry = {
            "severity": record.levelname,
            "message": record.getMessage(),
            "component": "claude-api"
        }
        if hasattr(record, "extra"):
            log_entry.update(record.extra)
        return json.dumps(log_entry)


logger = logging.getLogger("claude-api")
handler = logging.StreamHandler()
handler.setFormatter(StructuredLogFormatter())
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# 使用
def log_api_call(model, input_tokens, output_tokens, latency):
    logger.info("Claude API call", extra={
        "extra": {
            "model": model,
            "input_tokens": input_tokens,
            "output_tokens": output_tokens,
            "latency_ms": latency
        }
    })
```

## 最佳实践总结

### 1. 安全

- 使用服务账号而非用户凭证
- 启用 VPC Service Controls
- 使用 Secret Manager 管理密钥
- 启用 Cloud Audit Logs

### 2. 性能

- 选择离用户最近的区域
- 使用流式响应改善体验
- 实现连接池和请求复用
- 设置合理的超时时间

### 3. 成本

- 根据任务选择合适的模型
- 实现应用层缓存
- 设置预算告警
- 监控 token 使用量

### 4. 可靠性

- 实现重试和指数退避
- 考虑多区域回退
- 设置健康检查
- 监控错误率和延迟

```python
# 多区域回退示例
REGIONS = ["us-east5", "us-central1", "europe-west1"]

def invoke_with_failover(messages, project_id):
    """多区域回退"""
    for region in REGIONS:
        try:
            regional_client = AnthropicVertex(
                project_id=project_id,
                region=region
            )
            message = regional_client.messages.create(
                model="claude-sonnet-4@20250514",
                max_tokens=2048,
                messages=messages
            )
            return message
        except Exception as e:
            print(f"Region {region} failed: {e}")
            continue

    raise Exception("All regions failed")
```

---

## 关键要点

- 工具调用接口与 Anthropic 直接 API 一致，定义工具后通过 stop_reason 判断是否需要执行
- RAG 可通过 Vertex AI Search 或自定义向量检索实现
- Google Search Grounding 是 Vertex AI 独有功能，让模型获取最新信息
- 生产部署推荐 Cloud Run（简单）、Cloud Functions（事件驱动）或 GKE（高负载）
- 监控使用 Cloud Monitoring 自定义指标和结构化日志
- 最佳实践涵盖安全、性能、成本和可靠性四个维度

---

[← 上一课：API 集成](./03-api-integration.md) | [返回目录](./README.md)
