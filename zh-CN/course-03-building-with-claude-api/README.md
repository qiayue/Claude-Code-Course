# Claude API 开发 (Building with the Claude API)

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程 **[Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api)** 的公开内容和 [Anthropic API 官方文档](https://docs.anthropic.com/) 编写。

---

## 课程简介

本课程将教你如何使用 **Anthropic API** 将 Claude AI 集成到你的应用程序中。从基础的 API 调用到构建生产级应用，你将系统地学习 Claude API 的核心功能和最佳实践。

无论你是想构建智能聊天机器人、文档分析工具，还是复杂的 AI 代理系统，本课程都将为你提供坚实的技术基础。

### 学习目标

完成本课程后，你将能够：

- 使用 Python SDK 和 TypeScript SDK 调用 Claude API
- 编写高效的提示词（Prompt），掌握提示工程最佳实践
- 实现工具使用（Tool Use / Function Calling），让 Claude 与外部系统交互
- 处理多模态输入，包括图片和 PDF 文档
- 使用流式传输（Streaming）和扩展思考（Extended Thinking）等高级功能
- 构建 RAG、对话式 AI 和 Agent 等生产级应用架构

### 前置要求

- **编程基础**：熟悉 Python 或 JavaScript/TypeScript
- **API 概念**：了解 REST API 基本概念（HTTP 请求、JSON 格式等）
- **API 密钥**：拥有 [Anthropic Console](https://console.anthropic.com/) 账号并生成 API Key

---

## 课程目录

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [API 基础](./01-api-fundamentals.md) | 获取 API 密钥、首次调用、Messages API、模型选择与参数配置 |
| 2 | [提示工程](./02-prompt-engineering.md) | 系统提示词、多轮对话、XML 标签、思维链、少样本学习、提示缓存 |
| 3 | [工具使用](./03-tool-use.md) | 定义工具、工具调用流程、处理结果、强制使用工具、错误处理 |
| 4 | [多模态能力](./04-multimodal-capabilities.md) | 图片输入（Base64/URL）、PDF 处理、图像分析、多模态最佳实践 |
| 5 | [流式传输与高级功能](./05-streaming-and-advanced.md) | SSE 流式传输、扩展思考、批量请求、速率限制、提示缓存优化 |
| 6 | [构建应用](./06-building-applications.md) | RAG 模式、对话式 AI、Agent 架构、生产最佳实践、安全与过滤 |

---

## 学习路径建议

1. **按顺序学习**：课程内容逐步递进，建议按顺序完成
2. **动手实践**：每个课时都包含可运行的代码示例，建议在本地环境中实际运行
3. **参考文档**：遇到细节问题时，查阅 [官方 API 文档](https://docs.anthropic.com/)

---

## 环境准备

在开始学习之前，请完成以下环境配置：

```bash
# 安装 Python SDK
pip install anthropic

# 或安装 TypeScript SDK
npm install @anthropic-ai/sdk

# 设置 API 密钥环境变量
export ANTHROPIC_API_KEY="your-api-key-here"
```

---

## 参考资源

- [Anthropic API 官方文档](https://docs.anthropic.com/)
- [Anthropic Python SDK](https://github.com/anthropics/anthropic-sdk-python)
- [Anthropic TypeScript SDK](https://github.com/anthropics/anthropic-sdk-typescript)
- [Anthropic Cookbook](https://github.com/anthropics/anthropic-cookbook)
- [API 定价页面](https://www.anthropic.com/pricing)
