# 课时 2：配置与设置

[← 上一课：Vertex AI 概述](./01-vertex-ai-overview.md) | [返回目录](./README.md) | [下一课：API 集成 →](./03-api-integration.md)

---

## 核心概念

在通过 Vertex AI 调用 Claude 之前，你需要完成 GCP 项目配置、模型启用、认证设置和 SDK 安装。本课时将逐步引导你完成这些准备工作。

## GCP 项目设置

### 创建或选择项目

```bash
# 安装 Google Cloud CLI（如果尚未安装）
# macOS
brew install google-cloud-sdk

# Linux
curl https://sdk.cloud.google.com | bash

# 初始化 gcloud
gcloud init

# 创建新项目
gcloud projects create my-claude-project --name="Claude on Vertex"

# 或选择已有项目
gcloud config set project my-claude-project
```

### 启用必要的 API

```bash
# 启用 Vertex AI API
gcloud services enable aiplatform.googleapis.com

# 启用 Cloud Resource Manager API（如果需要）
gcloud services enable cloudresourcemanager.googleapis.com

# 验证 API 已启用
gcloud services list --enabled --filter="name:aiplatform"
```

### 设置计费

确保项目已关联计费账号：

```bash
# 查看当前计费账号
gcloud billing accounts list

# 关联计费账号到项目
gcloud billing projects link my-claude-project \
  --billing-account=BILLING_ACCOUNT_ID
```

## 启用 Claude 模型

### 通过 Google Cloud Console

1. 前往 [Vertex AI Model Garden](https://console.cloud.google.com/vertex-ai/model-garden)
2. 搜索 "Claude" 或 "Anthropic"
3. 选择你需要的 Claude 模型
4. 点击 **Enable**（启用）
5. 同意使用条款
6. 等待模型启用完成

### 通过 gcloud CLI

```bash
# 查看可用的 Claude 模型
gcloud ai models list --region=us-east5 \
  --filter="displayName:claude" \
  2>/dev/null || echo "Use Console to enable models"
```

> **注意：** 部分 Claude 模型可能需要通过 Console 启用，gcloud CLI 支持可能有限。建议通过 Console 操作。

## 认证配置

Vertex AI 支持多种认证方式，适用于不同的开发和部署场景。

### 方式一：用户账号认证（开发环境）

最简单的方式，适合本地开发：

```bash
# 使用 gcloud 登录
gcloud auth login

# 设置应用默认凭证（ADC）
gcloud auth application-default login

# 设置默认项目和区域
gcloud config set project my-claude-project
gcloud config set compute/region us-east5
```

认证完成后，SDK 会自动使用这些凭证。

### 方式二：服务账号（生产环境推荐）

```bash
# 创建服务账号
gcloud iam service-accounts create claude-vertex-sa \
  --display-name="Claude Vertex AI Service Account"

# 授予 Vertex AI 用户角色
gcloud projects add-iam-policy-binding my-claude-project \
  --member="serviceAccount:claude-vertex-sa@my-claude-project.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"

# 创建密钥文件（本地开发用）
gcloud iam service-accounts keys create key.json \
  --iam-account=claude-vertex-sa@my-claude-project.iam.gserviceaccount.com
```

使用服务账号密钥：

```bash
# 设置环境变量
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
```

> **安全提醒：** 服务账号密钥文件（key.json）包含敏感信息。不要将其提交到版本控制系统，也不要在不安全的环境中存储。

### 方式三：工作负载身份联合（Workload Identity Federation）

适用于从非 GCP 环境（如 GitHub Actions、AWS）访问 Vertex AI：

```bash
# 创建工作负载身份池
gcloud iam workload-identity-pools create "github-pool" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# 创建提供者
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub Provider" \
  --attribute-mapping="google.subject=assertion.sub" \
  --issuer-uri="https://token.actions.githubusercontent.com"
```

### 方式四：GCP 服务内置认证

在 Cloud Run、Cloud Functions、GKE 等 GCP 服务中，可以使用内置的服务账号，无需手动管理凭证：

```python
# 在 GCP 服务中，SDK 自动使用内置凭证
# 无需显式配置
from anthropic import AnthropicVertex

client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)
```

## IAM 权限配置

### 最小权限角色

| 角色 | 权限 | 适用场景 |
|------|------|----------|
| `roles/aiplatform.user` | 调用模型预测 | 基本 API 调用 |
| `roles/aiplatform.admin` | 完整 Vertex AI 管理 | 管理员操作 |

### 自定义角色（精细控制）

```bash
# 创建自定义角色
gcloud iam roles create claudeInvoker \
  --project=my-claude-project \
  --title="Claude Invoker" \
  --description="Can invoke Claude models via Vertex AI" \
  --permissions="aiplatform.endpoints.predict"
```

### 设置权限边界

```bash
# 限制服务账号只能在特定区域调用模型
gcloud projects add-iam-policy-binding my-claude-project \
  --member="serviceAccount:claude-vertex-sa@my-claude-project.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user" \
  --condition='expression=resource.name.startsWith("projects/my-claude-project/locations/us-east5"),title=us-east5-only'
```

## Anthropic Vertex SDK 安装

### 安装 SDK

```bash
# 创建虚拟环境（推荐）
python3 -m venv vertex-env
source vertex-env/bin/activate  # Linux/Mac
# vertex-env\Scripts\activate   # Windows

# 安装 Anthropic SDK（包含 Vertex AI 支持）
pip install 'anthropic[vertex]'

# 验证安装
python3 -c "from anthropic import AnthropicVertex; print('SDK installed successfully')"
```

### 环境变量配置

```bash
# 设置项目 ID 和区域
export CLOUD_ML_PROJECT_ID="my-claude-project"
export CLOUD_ML_REGION="us-east5"

# 如果使用服务账号密钥
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/key.json"
```

### 验证配置

```python
from anthropic import AnthropicVertex

# 创建客户端
client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)

# 发送测试请求
message = client.messages.create(
    model="claude-sonnet-4@20250514",
    max_tokens=100,
    messages=[
        {
            "role": "user",
            "content": "Hello! Please respond with a short greeting."
        }
    ]
)

print(message.content[0].text)
print(f"Input tokens: {message.usage.input_tokens}")
print(f"Output tokens: {message.usage.output_tokens}")
```

如果看到 Claude 的回复和 token 使用量，说明配置完成。

## 替代方式：使用 Google Cloud SDK

除了 Anthropic Vertex SDK，你也可以使用 Google 的官方 SDK：

```bash
# 安装 Google Cloud AI Platform SDK
pip install google-cloud-aiplatform
```

```python
import vertexai
from vertexai.preview.generative_models import GenerativeModel

# 初始化 Vertex AI
vertexai.init(project="my-claude-project", location="us-east5")

# 注意：使用 Google SDK 时，API 格式可能不同
# 推荐使用 Anthropic Vertex SDK 以获得与直接 API 一致的体验
```

> **建议：** 优先使用 Anthropic Vertex SDK（`anthropic[vertex]`），它提供与 Anthropic 直接 API 几乎一致的接口，迁移成本最低。

## 安全最佳实践

### 1. 使用服务账号而非用户账号

```python
# 生产环境：使用服务账号
# SDK 自动从 GOOGLE_APPLICATION_CREDENTIALS 或内置凭证获取认证
client = AnthropicVertex(
    project_id="my-claude-project",
    region="us-east5"
)
```

### 2. 启用 VPC Service Controls

```bash
# 创建 VPC Service Perimeter
gcloud access-context-manager perimeters create claude-perimeter \
  --title="Claude API Perimeter" \
  --resources="projects/PROJECT_NUMBER" \
  --restricted-services="aiplatform.googleapis.com" \
  --policy=POLICY_ID
```

### 3. 启用审计日志

```bash
# 确保 Vertex AI 的审计日志已启用
# 在 Cloud Console → IAM & Admin → Audit Logs 中配置
# 或通过 gcloud：
gcloud projects get-iam-policy my-claude-project \
  --format=json > policy.json

# 编辑 policy.json 添加审计配置后应用
gcloud projects set-iam-policy my-claude-project policy.json
```

### 4. 设置预算告警

```bash
# 在 Cloud Console → Billing → Budgets 中创建预算
# 设置 Vertex AI 使用量的预算告警
# 可以配置在达到 50%、80%、100% 时发送通知
```

## 常见问题排查

### 问题一：PermissionDenied

```
google.api_core.exceptions.PermissionDenied: 403
```

**解决方案：**
1. 确认服务账号有 `roles/aiplatform.user` 角色
2. 检查项目 ID 是否正确
3. 确认 Vertex AI API 已启用

### 问题二：Model not found

```
Model claude-xxx not found in region yyy
```

**解决方案：**
1. 确认模型在指定区域可用（见概述课时的区域表）
2. 检查 Model ID 拼写是否正确
3. 确认模型已在 Model Garden 中启用

### 问题三：认证失败

```
google.auth.exceptions.DefaultCredentialsError
```

**解决方案：**
1. 运行 `gcloud auth application-default login`
2. 或设置 `GOOGLE_APPLICATION_CREDENTIALS` 环境变量
3. 在 GCP 服务中，确认实例已配置服务账号

### 问题四：配额超限

```
ResourceExhausted: Quota exceeded
```

**解决方案：**
1. 在 Cloud Console 中查看和调整配额
2. 实现请求限流和重试逻辑
3. 考虑在多个区域分散请求

---

## 关键要点

- GCP 项目需要启用 Vertex AI API 并在 Model Garden 中启用 Claude 模型
- 认证方式：用户账号（开发）、服务账号（生产）、工作负载身份联合（跨云）
- 推荐使用 Anthropic Vertex SDK（`anthropic[vertex]`），接口与直接 API 一致
- IAM 最小权限：`roles/aiplatform.user` 角色即可满足基本调用需求
- 安全最佳实践包括 VPC Service Controls、审计日志和预算告警
- 常见错误多数由权限配置、模型区域或认证问题导致

---

[← 上一课：Vertex AI 概述](./01-vertex-ai-overview.md) | [返回目录](./README.md) | [下一课：API 集成 →](./03-api-integration.md)
