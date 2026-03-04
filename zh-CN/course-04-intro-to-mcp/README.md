# MCP 入门 (Introduction to Model Context Protocol)

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程 **Introduction to MCP** 的公开内容和 [MCP 规范文档](https://modelcontextprotocol.io/) 编写。

---

## 课程简介

**MCP（Model Context Protocol，模型上下文协议）** 是 Anthropic 推出的开放标准，用于将 AI 模型连接到外部数据源和工具。本课程将从零开始，系统地讲解 MCP 的核心概念、架构设计和实际开发方法，帮助你构建自己的 MCP 服务器和客户端。

MCP 的设计理念类似于 USB 接口——它为 AI 模型与外部世界之间提供了一个标准化的通信协议。无论是数据库查询、文件操作还是第三方 API 调用，MCP 都能以统一的方式将这些能力暴露给 AI 模型。

### 学习目标

完成本课程后，你将能够：

- 理解 MCP 的架构设计和核心概念
- 掌握 MCP 的三大原语：工具（Tools）、资源（Resources）和提示（Prompts）
- 使用 Python 构建功能完整的 MCP 服务器
- 实现 MCP 客户端并连接到服务器
- 理解 MCP 与传统 API 集成方式的区别

### 前置要求

- Python 3.10+ 编程经验
- 了解基本的客户端-服务器架构
- 熟悉命令行操作
- 对大语言模型（LLM）有基本了解

---

## 课程目录

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [什么是 MCP](./01-what-is-mcp.md) | MCP 概述、动机、客户端-服务器架构和与传统 API 的对比 |
| 2 | [MCP 三大原语](./02-mcp-primitives.md) | 工具、资源、提示的详细讲解和使用场景 |
| 3 | [构建 MCP 服务器](./03-building-mcp-server.md) | 使用 Python 搭建 MCP 服务器，实现工具、资源和提示 |
| 4 | [构建 MCP 客户端](./04-building-mcp-client.md) | 客户端设置、连接服务器、调用工具和错误处理 |

---

## 技术栈

本课程使用以下技术：

| 技术 | 用途 |
|------|------|
| Python 3.10+ | 主要开发语言 |
| `mcp` | MCP Python SDK |
| `httpx` | HTTP 客户端库 |
| `uvicorn` | ASGI 服务器 |

### 环境搭建

```bash
# 创建虚拟环境
python -m venv mcp-env
source mcp-env/bin/activate

# 安装 MCP SDK
pip install mcp
```

---

## 参考资源

- [MCP 官方规范](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Claude Code 文档 — MCP 服务器](https://code.claude.com/docs/en/overview)
- [Anthropic Academy](https://anthropic.skilljar.com/)
