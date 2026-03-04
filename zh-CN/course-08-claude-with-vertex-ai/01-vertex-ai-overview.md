# 课时 1：Vertex AI 概述

[返回目录](./README.md) | [下一课：配置与设置 →](./02-setup-and-configuration.md)

---

## 核心概念

**Google Vertex AI** 是 Google Cloud 的统一机器学习平台，涵盖模型训练、部署和预测服务。通过 Vertex AI 的 **Model Garden**，你可以访问来自多家提供商的基础模型，包括 Anthropic 的 Claude 系列。

## 什么是 Google Vertex AI？

Vertex AI 是 Google Cloud 的端到端 AI/ML 平台，提供了从数据准备到模型部署的完整工具链：

```
┌─────────────────────────────────────────────┐
│              你的应用程序                      │
├─────────────────────────────────────────────┤
│            Vertex AI API                      │
├──────┬──────┬──────┬──────┬─────────────────┤
│Claude│Gemini│Llama │Mistral│  更多模型...     │
├──────┴──────┴──────┴──────┴─────────────────┤
│          Google Cloud 基础设施                 │
│    (安全、网络、计算、存储)                      │
└─────────────────────────────────────────────┘
```

### Vertex AI 的核心特性

| 特性 | 说明 |
|------|------|
| **Model Garden** | 统一入口，访问 Google 和第三方的基础模型 |
| **全托管** | 无需管理 GPU、服务器或模型部署 |
| **GCP 原生集成** | 与 BigQuery、Cloud Storage、Cloud Run 等深度集成 |
| **企业安全** | VPC Service Controls、CMEK 加密、IAM 权限 |
| **Google 独有功能** | Grounding with Google Search、RAG API |
| **按需计费** | 按输入/输出 token 计费 |

## 为什么通过 Vertex AI 使用 Claude？

### 直接 API vs Vertex AI

| 维度 | Anthropic 直接 API | Google Vertex AI |
|------|-------------------|-----------------|
| 认证方式 | Anthropic API Key | GCP IAM / 服务账号 |
| 数据位置 | Anthropic 基础设施 | Google Cloud 环境 |
| 网络控制 | 公网访问 | VPC Service Controls |
| 合规认证 | SOC 2、HIPAA | 继承 GCP 认证（SOC、HIPAA、ISO 等） |
| 计费方式 | Anthropic 账单 | GCP 统一账单 |
| 附加功能 | 原生 API 功能 | Google Search Grounding、RAG API |
| 适合场景 | 快速原型、个人项目 | GCP 用户、企业生产环境 |

### 选择 Vertex AI 的主要理由

1. **GCP 生态集成**：如果你已经在使用 GCP，Vertex AI 提供无缝集成
2. **统一计费**：通过 GCP 统一账单管理，利用承诺使用折扣（CUD）
3. **数据合规**：数据在 Google Cloud 环境中处理，满足数据驻留要求
4. **Google Search Grounding**：Claude 可以访问 Google 搜索结果，获取最新信息
5. **VPC Service Controls**：精细的网络安全控制
6. **IAM 统一管理**：使用 GCP IAM 统一管理 Claude 的访问权限

## 可用的 Claude 模型

Vertex AI 通过 Model Garden 提供 Claude 模型：

### Claude 模型系列

| 模型 | Model ID (Vertex AI) | 特点 | 适用场景 |
|------|---------------------|------|----------|
| Claude Opus 4 | `claude-opus-4@20250514` | 最强推理能力 | 复杂分析、高级编程 |
| Claude Sonnet 4 | `claude-sonnet-4@20250514` | 性能与成本平衡 | 通用任务、生产应用 |
| Claude Haiku 3.5 | `claude-3-5-haiku@20241022` | 最快速度 | 实时响应、大量请求 |

> **注意：** Vertex AI 上的 Model ID 格式与 Anthropic 直接 API 不同。

### 模型选择建议

```
需要最强推理和创造力 → Claude Opus 4
  ├── 复杂数学和科学问题
  ├── 高级代码架构设计
  └── 深度内容创作

日常任务、高性价比 → Claude Sonnet 4
  ├── 代码生成和调试
  ├── 客户服务
  └── 数据分析和报告

低延迟、高吞吐 → Claude Haiku 3.5
  ├── 实时对话
  ├── 文本分类和提取
  └── 简单问答
```

## 区域可用性

Claude 模型在以下 Google Cloud 区域可用：

| 区域 | 区域代码 | Claude Opus 4 | Claude Sonnet 4 | Claude Haiku 3.5 |
|------|---------|-----------|------------|------------|
| 美国（爱荷华） | us-central1 | - | 可用 | 可用 |
| 美国（弗吉尼亚） | us-east4 | - | 可用 | - |
| 美国（南卡罗来纳） | us-east5 | 可用 | 可用 | 可用 |
| 欧洲（伦敦） | europe-west1 | - | 可用 | 可用 |
| 亚太（东京） | asia-northeast1 | - | 可用 | - |

> **提示：** 区域可用性经常更新，请查看 [Google Cloud 文档](https://cloud.google.com/vertex-ai/generative-ai/docs/partner-models/use-claude#regions) 获取最新信息。

### 多区域策略

如果你的应用需要全球覆盖，建议：

```python
# 按用户地理位置选择最近的区域
REGION_MAP = {
    "asia": "asia-northeast1",
    "europe": "europe-west1",
    "americas": "us-east5"
}
```

## 定价

### Vertex AI Claude 定价结构

| 模型 | 输入价格 (每百万 token) | 输出价格 (每百万 token) |
|------|----------------------|----------------------|
| Claude Opus 4 | $15.00 | $75.00 |
| Claude Sonnet 4 | $3.00 | $15.00 |
| Claude Haiku 3.5 | $0.80 | $4.00 |

> **注意：** 以上价格为参考值，实际价格请查看 [Vertex AI 定价页面](https://cloud.google.com/vertex-ai/pricing)。价格可能因区域和合同而异。

### 成本优化策略

1. **选择合适的模型**：根据任务复杂度选择模型，避免"杀鸡用牛刀"
2. **优化提示词长度**：减少冗余的上下文信息
3. **使用承诺使用折扣（CUD）**：对于稳定负载，CUD 可以显著降低成本
4. **缓存策略**：对于重复查询，使用应用层缓存减少 API 调用
5. **设置配额限制**：在 GCP 中设置 API 配额，防止意外超支

## Vertex AI vs Amazon Bedrock

如果你在选择云平台，以下是两个平台的对比：

| 维度 | Vertex AI | Amazon Bedrock |
|------|-----------|---------------|
| 云平台 | Google Cloud | AWS |
| Claude 模型 | 通过 Model Garden | 原生支持 |
| SDK | Anthropic Vertex SDK | boto3 |
| 认证 | GCP IAM + 服务账号 | AWS IAM |
| 独有功能 | Google Search Grounding | Knowledge Bases、Agents |
| RAG | Vertex AI RAG API | Bedrock Knowledge Bases |
| 护栏 | 需要自行实现 | Bedrock Guardrails |
| 适合人群 | GCP 用户 | AWS 用户 |

**建议：** 选择你团队已经在使用的云平台。如果两者都可以，根据具体功能需求决定。

## Vertex AI 与 GCP 服务的集成

```
┌──────────────────────────────────────────────┐
│                 应用层                         │
│ Cloud Run │ Cloud Functions │ GKE │ App Engine │
├──────────────────────────────────────────────┤
│            Vertex AI (Claude)                  │
├──────────────────────────────────────────────┤
│                 数据层                         │
│  BigQuery │ Cloud Storage │ Firestore │ Spanner│
├──────────────────────────────────────────────┤
│              安全与监控                         │
│  IAM │ KMS │ Cloud Audit Logs │ Cloud Monitoring│
└──────────────────────────────────────────────┘
```

**关键集成：**
- **Cloud Run / Cloud Functions**：无服务器调用 Claude
- **BigQuery**：分析 Claude 生成的结构化数据
- **Cloud Storage**：存储文档，配合 RAG 使用
- **Cloud Monitoring**：监控 API 调用延迟和错误率
- **Cloud Audit Logs**：审计所有 Vertex AI API 调用
- **Secret Manager**：安全管理 API 密钥和凭证

---

## 关键要点

- Google Vertex AI 通过 Model Garden 提供 Claude 模型访问
- 核心优势：GCP 生态集成、Google Search Grounding、统一 IAM 管理
- Claude 在 Vertex AI 上有多个模型可选：Opus（最强）、Sonnet（平衡）、Haiku（最快）
- Vertex AI 的 Model ID 格式与直接 API 不同，如 `claude-sonnet-4@20250514`
- 区域可用性因模型而异，选择离用户最近的可用区域
- 成本优化可通过模型选择、CUD 和缓存策略实现

---

[返回目录](./README.md) | [下一课：配置与设置 →](./02-setup-and-configuration.md)
