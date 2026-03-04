# Claude with Google Vertex AI

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程 **[Claude with Google Vertex](https://anthropic.skilljar.com/claude-with-google-vertex)** 的公开内容和 [Google Cloud Vertex AI 文档](https://cloud.google.com/vertex-ai/docs) 编写，旨在帮助中文用户掌握通过 Google Vertex AI 使用 Claude 的技术方案。

---

## 课程简介

**Google Vertex AI** 是 Google Cloud 的机器学习平台，提供了通过 Model Garden 访问第三方模型（包括 Anthropic Claude）的能力。通过 Vertex AI 使用 Claude，你可以利用 Google Cloud 的安全基础设施、与 GCP 生态系统深度集成，并使用 Google 独有的功能如 Grounding with Google Search。

本课程将从 Vertex AI 平台概述开始，逐步讲解配置、API 集成和高级功能，帮助你在 Google Cloud 生态中高效使用 Claude。

### 学习目标

完成本课程后，你将能够：

- 理解 Google Vertex AI 的架构和 Claude 在其中的定位
- 配置 GCP 项目和认证以访问 Claude 模型
- 使用 Anthropic Python SDK 通过 Vertex AI 调用 Claude
- 实现流式响应和多轮对话
- 使用工具调用、RAG 集成和 Google Search Grounding 等高级功能
- 设计生产级的 Claude on Vertex AI 部署方案

### 前置要求

- Google Cloud 账号（需要启用 Vertex AI API）
- 基本的 Python 编程经验
- 了解 GCP IAM 和服务账号
- 建议先完成 [Claude API 开发](../course-03-building-with-claude-api/README.md) 课程

---

## 课程目录

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [Vertex AI 概述](./01-vertex-ai-overview.md) | Google Vertex AI 介绍、Claude 模型选择、区域和定价 |
| 2 | [配置与设置](./02-setup-and-configuration.md) | GCP 项目配置、模型启用、认证方式、SDK 安装 |
| 3 | [API 集成](./03-api-integration.md) | Anthropic Vertex SDK、请求格式、流式响应、代码示例 |
| 4 | [高级功能](./04-advanced-features.md) | 工具调用、RAG 集成、Google Search Grounding、生产部署 |

---

## 学习建议

1. **准备 GCP 环境**：在开始之前，确保你有一个启用了 Vertex AI 的 GCP 项目
2. **动手实践**：每个课时的代码示例都可以直接运行，建议在本地或 Cloud Shell 中测试
3. **关注成本**：Vertex AI 按 token 计费，学习时建议使用较小的输入
4. **对比学习**：如果你已经了解 Bedrock，注意与 Vertex AI 方案的差异

---

## 参考资源

- [Vertex AI Claude 文档](https://cloud.google.com/vertex-ai/generative-ai/docs/partner-models/use-claude)
- [Anthropic Vertex SDK 文档](https://docs.anthropic.com/en/api/claude-on-vertex-ai)
- [Google Cloud Vertex AI 文档](https://cloud.google.com/vertex-ai/docs)
- [Anthropic Claude 文档](https://docs.anthropic.com/)
- [Vertex AI 定价](https://cloud.google.com/vertex-ai/pricing)
