# 课时 1：Bedrock 概述

[返回目录](./README.md) | [下一课：配置与设置 →](./02-setup-and-configuration.md)

---

## 核心概念

**Amazon Bedrock** 是 AWS 提供的全托管生成式 AI 服务，让开发者可以通过统一的 API 访问来自多家提供商的基础模型。Bedrock 不需要你管理基础设施，模型以 API 形式提供，按使用量计费。

## 什么是 Amazon Bedrock？

Amazon Bedrock 是一个平台层服务，位于 AWS 基础设施和你的应用之间：

```
┌─────────────────────────────────────────┐
│             你的应用程序                   │
├─────────────────────────────────────────┤
│           Amazon Bedrock API              │
├──────┬──────┬──────┬──────┬─────────────┤
│Claude│Titan │Llama │Mistral│  更多模型... │
├──────┴──────┴──────┴──────┴─────────────┤
│            AWS 基础设施                    │
│    (安全、合规、网络、计算)                  │
└─────────────────────────────────────────┘
```

### Bedrock 的核心特性

| 特性 | 说明 |
|------|------|
| **全托管** | 无需管理 GPU、服务器或模型部署 |
| **多模型** | 通过同一 API 访问多个提供商的模型 |
| **安全合规** | 数据不离开你的 AWS 环境，支持 VPC、加密 |
| **企业级** | IAM 权限控制、CloudTrail 审计、CloudWatch 监控 |
| **按需计费** | 按输入/输出 token 计费，无最低承诺 |

## 为什么通过 Bedrock 使用 Claude？

### 直接 API vs Bedrock

| 维度 | Anthropic 直接 API | Amazon Bedrock |
|------|-------------------|----------------|
| 认证方式 | Anthropic API Key | AWS IAM |
| 数据位置 | Anthropic 基础设施 | 你的 AWS 环境 |
| 网络控制 | 公网访问 | VPC、PrivateLink 支持 |
| 合规认证 | SOC 2、HIPAA | 继承 AWS 全套认证（SOC、HIPAA、FedRAMP 等） |
| 计费方式 | Anthropic 账单 | AWS 统一账单 |
| 模型访问 | 最新模型优先可用 | 部分模型延迟上线 |
| 附加功能 | 原生 API 功能 | Knowledge Bases、Agents、Guardrails |
| 适合场景 | 快速原型、个人项目 | 企业生产环境 |

### 选择 Bedrock 的主要理由

1. **数据合规**：数据不离开你的 AWS 账号和区域
2. **统一管理**：使用 AWS IAM 统一管理权限，与现有 AWS 资源集成
3. **企业计费**：通过 AWS 统一账单，利用企业折扣和预留容量
4. **网络安全**：支持 VPC Endpoint，模型调用不经过公网
5. **审计追踪**：通过 CloudTrail 记录所有 API 调用
6. **附加服务**：Knowledge Bases、Agents、Guardrails 等增值功能

## 可用的 Claude 模型

Amazon Bedrock 提供多个 Claude 模型版本，适用于不同场景：

### Claude 模型系列

| 模型 | Model ID | 特点 | 适用场景 |
|------|----------|------|----------|
| Claude Opus 4 | `anthropic.claude-opus-4-20250514-v1:0` | 最强推理能力 | 复杂分析、高级编程 |
| Claude Sonnet 4 | `anthropic.claude-sonnet-4-20250514-v1:0` | 性能与成本平衡 | 通用任务、生产应用 |
| Claude Haiku 3.5 | `anthropic.claude-3-5-haiku-20241022-v1:0` | 最快速度 | 实时响应、大量请求 |

### 模型选择建议

```
复杂度高、质量优先 → Claude Opus 4
  ├── 法律文档分析
  ├── 复杂代码生成
  └── 深度研究报告

通用任务、性价比高 → Claude Sonnet 4
  ├── 客户服务对话
  ├── 内容生成
  └── 数据分析

速度优先、大批量 → Claude Haiku 3.5
  ├── 实时聊天
  ├── 文本分类
  └── 数据提取
```

## 定价对比

### Bedrock Claude 定价结构

Bedrock 的 Claude 定价按输入和输出 token 分别计费：

| 模型 | 输入价格 (每百万 token) | 输出价格 (每百万 token) |
|------|----------------------|----------------------|
| Claude Opus 4 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |

> **注意：** 以上价格为参考值，实际价格请查看 [AWS Bedrock 定价页面](https://aws.amazon.com/bedrock/pricing/)，价格可能因区域和时间而变化。

### 成本优化策略

1. **选择合适的模型**：不是所有任务都需要 Opus，很多场景 Sonnet 或 Haiku 就够了
2. **优化提示词**：减少不必要的上下文，降低输入 token 数
3. **使用流式响应**：可以在生成过程中提前终止，减少输出 token
4. **批量推理**：对于非实时任务，使用批量推理可以获得折扣
5. **预置吞吐量**：对于稳定负载，购买 Provisioned Throughput 可降低单位成本

## Bedrock 区域可用性

Claude 模型在以下 AWS 区域可用（区域列表持续扩展）：

| 区域 | 区域代码 | Claude Opus 4 | Claude Sonnet 4 | Claude Haiku 3.5 |
|------|---------|-----------|------------|------------|
| 美国东部（弗吉尼亚） | us-east-1 | 可用 | 可用 | 可用 |
| 美国西部（俄勒冈） | us-west-2 | 可用 | 可用 | 可用 |
| 欧洲（法兰克福） | eu-central-1 | - | 可用 | 可用 |
| 亚太（东京） | ap-northeast-1 | - | 可用 | 可用 |

> **提示：** 区域可用性经常更新，请查看 [AWS 文档](https://docs.aws.amazon.com/bedrock/latest/userguide/models-regions.html) 获取最新信息。

### 跨区域推理

如果你的首选区域没有所需模型，可以使用 Bedrock 的跨区域推理功能：

```python
# 跨区域推理示例
# Bedrock 会自动路由到可用区域
model_id = "anthropic.claude-sonnet-4-20250514-v1:0"
```

## Bedrock 与其他 AWS 服务的集成

Amazon Bedrock 可以与 AWS 生态中的多种服务集成：

```
┌──────────────────────────────────────────────┐
│                 应用层                         │
│   Lambda  │  ECS/EKS  │  SageMaker  │  EC2   │
├──────────────────────────────────────────────┤
│              Amazon Bedrock                    │
├──────────────────────────────────────────────┤
│                 数据层                         │
│   S3  │  OpenSearch  │  RDS  │  DynamoDB     │
├──────────────────────────────────────────────┤
│                安全与监控                       │
│  IAM  │  KMS  │  CloudTrail  │  CloudWatch   │
└──────────────────────────────────────────────┘
```

**关键集成：**
- **Lambda**：无服务器函数调用 Claude
- **S3**：存储文档和数据，配合 Knowledge Bases 使用
- **OpenSearch**：作为 Knowledge Bases 的向量存储
- **CloudWatch**：监控 API 调用延迟、错误率、token 使用量
- **CloudTrail**：审计所有 Bedrock API 调用
- **KMS**：加密静态数据和传输中的数据

---

## 关键要点

- Amazon Bedrock 是 AWS 的全托管生成式 AI 服务，提供统一的 API 访问多个模型
- 通过 Bedrock 使用 Claude 的核心优势：数据合规、统一 IAM 管理、AWS 企业级安全
- Claude 在 Bedrock 上有多个模型可选：Opus（最强）、Sonnet（平衡）、Haiku（最快）
- 定价按输入/输出 token 计费，可通过模型选择和提示优化控制成本
- Bedrock 与 AWS 生态深度集成（Lambda、S3、CloudWatch 等）
- 区域可用性持续扩展，支持跨区域推理

---

[返回目录](./README.md) | [下一课：配置与设置 →](./02-setup-and-configuration.md)
