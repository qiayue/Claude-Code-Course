# 课时 2：配置与设置

[← 上一课：Bedrock 概述](./01-bedrock-overview.md) | [返回目录](./README.md) | [下一课：API 集成 →](./03-api-integration.md)

---

## 核心概念

在开始通过 Amazon Bedrock 调用 Claude 之前，你需要完成 AWS 环境的配置工作。本课时将逐步引导你完成 AWS 账号设置、IAM 权限配置、Claude 模型启用和本地 SDK 安装。

## AWS 账号设置

### 前提条件

确保你有一个 AWS 账号。如果没有，请前往 [AWS 注册页面](https://aws.amazon.com/) 创建。

### 确认区域支持

登录 AWS 控制台后，确认你使用的区域支持 Bedrock 和 Claude 模型：

```bash
# 推荐使用以下区域之一
us-east-1    # 美国东部（弗吉尼亚）— 模型最全
us-west-2    # 美国西部（俄勒冈）
eu-central-1 # 欧洲（法兰克福）
```

### 在控制台中启用 Bedrock

1. 登录 [AWS Management Console](https://console.aws.amazon.com/)
2. 在服务搜索栏中搜索 "Bedrock"
3. 进入 Amazon Bedrock 控制台
4. 在左侧导航栏选择 **Model access**（模型访问）
5. 点击 **Manage model access**（管理模型访问）
6. 找到 **Anthropic** 部分，勾选你需要的 Claude 模型
7. 点击 **Request model access**（请求模型访问）
8. 等待审批（通常很快，部分模型可能需要额外审批）

> **重要：** 即使你的 AWS 账号已经创建，仍然需要**显式启用** Bedrock 中的 Claude 模型。未启用的模型无法通过 API 调用。

## IAM 权限配置

### 最小权限原则

为 Bedrock API 调用创建专用的 IAM 策略，遵循最小权限原则：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BedrockInvokeModel",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream"
            ],
            "Resource": [
                "arn:aws:bedrock:us-east-1::foundation-model/anthropic.claude-*"
            ]
        }
    ]
}
```

此策略仅允许调用 Claude 模型，不授予其他 Bedrock 操作权限。

### 创建 IAM 用户/角色

#### 方式一：IAM 用户（开发环境）

```bash
# 使用 AWS CLI 创建 IAM 用户
aws iam create-user --user-name bedrock-developer

# 创建策略
aws iam create-policy \
  --policy-name BedrockClaudeAccess \
  --policy-document file://bedrock-policy.json

# 附加策略到用户
aws iam attach-user-policy \
  --user-name bedrock-developer \
  --policy-arn arn:aws:iam::123456789012:policy/BedrockClaudeAccess

# 创建访问密钥
aws iam create-access-key --user-name bedrock-developer
```

#### 方式二：IAM 角色（生产环境推荐）

```bash
# 创建信任策略文件
cat > trust-policy.json << 'EOF'
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

# 创建 IAM 角色
aws iam create-role \
  --role-name BedrockClaudeRole \
  --assume-role-policy-document file://trust-policy.json

# 附加 Bedrock 策略
aws iam attach-role-policy \
  --role-name BedrockClaudeRole \
  --policy-arn arn:aws:iam::123456789012:policy/BedrockClaudeAccess
```

### 扩展权限（高级功能）

如果需要使用 Knowledge Bases、Agents 等高级功能，需要额外权限：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BedrockFullAccess",
            "Effect": "Allow",
            "Action": [
                "bedrock:InvokeModel",
                "bedrock:InvokeModelWithResponseStream",
                "bedrock:Retrieve",
                "bedrock:RetrieveAndGenerate",
                "bedrock:InvokeAgent"
            ],
            "Resource": "*"
        },
        {
            "Sid": "S3AccessForKnowledgeBase",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-kb-bucket",
                "arn:aws:s3:::your-kb-bucket/*"
            ]
        }
    ]
}
```

## 在 Bedrock 控制台中启用 Claude 模型

### 详细步骤

1. **进入 Bedrock 控制台**

   在 AWS 控制台搜索 "Bedrock" 并进入服务页面。

2. **选择 Model access**

   在左侧导航栏底部找到 "Model access"。

3. **查看可用模型**

   你会看到所有可用的基础模型列表，按提供商分组。

4. **启用 Anthropic Claude 模型**

   找到 Anthropic 部分，选择你需要的模型。

5. **提交申请**

   某些模型可能需要填写简短的使用说明。

6. **等待激活**

   状态从 "In progress" 变为 "Access granted" 即可使用。

### 验证模型访问

```bash
# 使用 AWS CLI 验证模型可用
aws bedrock list-foundation-models \
  --by-provider anthropic \
  --query "modelSummaries[].modelId" \
  --output table
```

输出示例：

```
------------------------------------------------------
|              ListFoundationModels                    |
+-----------------------------------------------------+
|  anthropic.claude-opus-4-20250514-v1:0              |
|  anthropic.claude-sonnet-4-20250514-v1:0            |
|  anthropic.claude-3-5-haiku-20241022-v1:0           |
+-----------------------------------------------------+
```

## boto3 SDK 安装与配置

### 安装 Python SDK

```bash
# 创建虚拟环境（推荐）
python3 -m venv bedrock-env
source bedrock-env/bin/activate  # Linux/Mac
# bedrock-env\Scripts\activate   # Windows

# 安装 boto3
pip install boto3

# 验证安装
python3 -c "import boto3; print(boto3.__version__)"
```

### 配置 AWS 凭证

#### 方式一：AWS CLI 配置（推荐用于开发）

```bash
# 安装 AWS CLI（如果尚未安装）
pip install awscli

# 配置凭证
aws configure
# AWS Access Key ID [None]: YOUR_ACCESS_KEY
# AWS Secret Access Key [None]: YOUR_SECRET_KEY
# Default region name [None]: us-east-1
# Default output format [None]: json
```

#### 方式二：环境变量

```bash
export AWS_ACCESS_KEY_ID="YOUR_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET_KEY"
export AWS_DEFAULT_REGION="us-east-1"
```

#### 方式三：AWS Profile（多账号管理）

```bash
# 在 ~/.aws/credentials 中配置多个 Profile
aws configure --profile bedrock-dev

# 使用时指定 Profile
export AWS_PROFILE=bedrock-dev
```

### 验证配置

```python
import boto3

# 创建 Bedrock 客户端
bedrock = boto3.client('bedrock', region_name='us-east-1')

# 列出可用的 Claude 模型
response = bedrock.list_foundation_models(
    byProvider='anthropic'
)

for model in response['modelSummaries']:
    print(f"Model: {model['modelId']}")
    print(f"  Name: {model['modelName']}")
    print(f"  Status: {model.get('modelLifecycle', {}).get('status', 'N/A')}")
    print()
```

### 测试 API 调用

```python
import boto3
import json

# 创建 Bedrock Runtime 客户端
bedrock_runtime = boto3.client(
    'bedrock-runtime',
    region_name='us-east-1'
)

# 发送测试请求
response = bedrock_runtime.invoke_model(
    modelId='anthropic.claude-sonnet-4-20250514-v1:0',
    contentType='application/json',
    accept='application/json',
    body=json.dumps({
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 100,
        "messages": [
            {
                "role": "user",
                "content": "Hello! Please respond with a short greeting."
            }
        ]
    })
)

result = json.loads(response['body'].read())
print(result['content'][0]['text'])
```

如果看到 Claude 的回复，说明配置完成。

## 安全最佳实践

### 1. 使用 IAM 角色而非长期凭证

```python
# 在 EC2/Lambda 中，使用实例角色/执行角色
# boto3 会自动获取临时凭证，无需硬编码密钥
bedrock_runtime = boto3.client('bedrock-runtime')
```

### 2. 启用 VPC Endpoint

```bash
# 创建 Bedrock VPC Endpoint，避免流量经过公网
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-1234567890 \
  --service-name com.amazonaws.us-east-1.bedrock-runtime \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc123 \
  --security-group-ids sg-abc123
```

### 3. 启用 CloudTrail 审计

```bash
# 确保 CloudTrail 记录 Bedrock API 调用
aws cloudtrail create-trail \
  --name bedrock-audit-trail \
  --s3-bucket-name your-cloudtrail-bucket \
  --is-multi-region-trail
```

### 4. 设置预算告警

```bash
# 在 AWS Budgets 中设置 Bedrock 使用量告警
# 避免意外的高额费用
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://bedrock-budget.json \
  --notifications-with-subscribers file://budget-notification.json
```

## 常见问题排查

### 问题一：AccessDeniedException

```
An error occurred (AccessDeniedException) when calling the InvokeModel operation
```

**解决方案：**
1. 确认 IAM 用户/角色有 `bedrock:InvokeModel` 权限
2. 确认权限策略中的 Resource ARN 正确
3. 确认模型已在 Bedrock 控制台中启用

### 问题二：ValidationException - Model not found

```
An error occurred (ValidationException): The provided model identifier is invalid
```

**解决方案：**
1. 检查 Model ID 拼写是否正确
2. 确认模型在当前区域可用
3. 确认模型访问已批准

### 问题三：ThrottlingException

```
An error occurred (ThrottlingException): Rate exceeded
```

**解决方案：**
1. 实现指数退避重试
2. 考虑购买 Provisioned Throughput
3. 申请提高配额限制

---

## 关键要点

- 使用 Bedrock 前需要在控制台显式启用 Claude 模型
- IAM 权限配置遵循最小权限原则，仅授予必要的 Bedrock 操作
- 生产环境推荐使用 IAM 角色而非长期访问密钥
- boto3 是 Python 中调用 Bedrock 的主要 SDK
- 安全最佳实践包括 VPC Endpoint、CloudTrail 审计和预算告警
- 常见错误多数由权限配置或模型未启用导致

---

[← 上一课：Bedrock 概述](./01-bedrock-overview.md) | [返回目录](./README.md) | [下一课：API 集成 →](./03-api-integration.md)
