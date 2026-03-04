# Claude with Amazon Bedrock

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程 **[Claude in Amazon Bedrock](https://anthropic.skilljar.com/claude-in-amazon-bedrock)** 的公开内容和 [AWS Bedrock 官方文档](https://docs.aws.amazon.com/bedrock/) 编写，旨在帮助中文用户掌握通过 Amazon Bedrock 使用 Claude 的技术方案。

---

## 课程简介

**Amazon Bedrock** 是 AWS 提供的全托管服务，让你可以通过 API 使用来自多家提供商的基础模型（Foundation Model），包括 Anthropic 的 Claude 系列。通过 Bedrock 使用 Claude，你可以利用 AWS 的安全基础设施、合规认证和企业级功能，同时保持数据不离开你的 AWS 环境。

本课程将从 Bedrock 平台概述开始，逐步讲解配置、API 集成和高级功能，帮助你在 AWS 生态中高效使用 Claude。

### 学习目标

完成本课程后，你将能够：

- 理解 Amazon Bedrock 的架构和 Claude 在其中的定位
- 配置 AWS 环境和 IAM 权限以访问 Claude 模型
- 使用 InvokeModel 和 Converse API 调用 Claude
- 通过 boto3 SDK 实现 Python 应用与 Claude 的集成
- 使用 Bedrock 的高级功能：工具调用、知识库、Agents 和 Guardrails
- 设计生产级的 Claude on Bedrock 部署方案

### 前置要求

- AWS 账号（需要有 Bedrock 访问权限）
- 基本的 Python 编程经验
- 了解 AWS IAM 权限模型
- 建议先完成 [Claude API 开发](../course-03-building-with-claude-api/README.md) 课程

---

## 课程目录

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [Bedrock 概述](./01-bedrock-overview.md) | Amazon Bedrock 介绍、Claude 模型选择、定价对比 |
| 2 | [配置与设置](./02-setup-and-configuration.md) | AWS 环境配置、IAM 权限、模型启用、SDK 安装 |
| 3 | [API 集成](./03-api-integration.md) | InvokeModel vs Converse API、请求格式、流式响应、代码示例 |
| 4 | [高级功能](./04-advanced-features.md) | 工具调用、知识库 RAG、Bedrock Agents、Guardrails、生产部署 |

---

## 学习建议

1. **准备 AWS 环境**：在开始之前，确保你有一个启用了 Bedrock 的 AWS 账号
2. **动手实践**：每个课时的代码示例都可以直接运行，建议在本地或 AWS CloudShell 中测试
3. **关注成本**：Bedrock 按 token 计费，学习时建议使用较小的模型和输入
4. **对比学习**：如果你已经熟悉 Claude 直接 API，注意与 Bedrock API 的差异

---

## 参考资源

- [Amazon Bedrock 官方文档](https://docs.aws.amazon.com/bedrock/)
- [Bedrock Claude 模型文档](https://docs.aws.amazon.com/bedrock/latest/userguide/model-parameters-anthropic-claude-messages.html)
- [boto3 Bedrock Runtime 文档](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/bedrock-runtime.html)
- [Anthropic Claude 文档](https://docs.anthropic.com/)
- [AWS Bedrock 定价](https://aws.amazon.com/bedrock/pricing/)
