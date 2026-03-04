# MCP 高级主题 (MCP Advanced Topics)

[← 返回课程目录](../README.md) | [English](../../README.md)

> 本指南基于 Anthropic 官方课程 **MCP Advanced Topics** 的公开内容和 [MCP 规范文档](https://modelcontextprotocol.io/) 编写。

---

## 课程简介

本课程是 **MCP 入门** 的进阶课程，深入探讨 MCP 的高级功能和生产环境实践。你将学习采样（Sampling）、通知与进度报告、传输机制选择，以及将 MCP 服务器部署到生产环境的最佳实践。

### 学习目标

完成本课程后，你将能够：

- 实现服务器端发起的 LLM 请求（采样）
- 使用通知和进度报告机制实现实时反馈
- 理解并选择合适的传输机制（stdio / HTTP+SSE）
- 在生产环境中安全、可靠地部署 MCP 服务器
- 实施监控、日志和错误处理的最佳实践

### 前置要求

- 完成 **MCP 入门** 课程（Course 04）
- Python 3.10+ 编程经验
- 了解异步编程（`async`/`await`）
- 基本的 Web 服务部署经验（对生产环境章节有帮助）

---

## 课程目录

| 课时 | 标题 | 内容概要 |
|------|------|----------|
| 1 | [采样 (Sampling)](./01-sampling.md) | 服务器发起的 LLM 请求、采样处理器、使用场景 |
| 2 | [通知与进度](./02-notifications-and-progress.md) | 通知系统、进度报告、事件处理机制 |
| 3 | [文件系统与传输机制](./03-filesystem-and-transports.md) | 文件系统访问模式、stdio/HTTP+SSE 传输、选型指南 |
| 4 | [生产环境实践](./04-production-patterns.md) | 安全性、错误处理、扩展、监控和最佳实践 |

---

## 技术栈

本课程在 MCP 入门课程的基础上，额外涉及以下技术：

| 技术 | 用途 |
|------|------|
| `mcp` | MCP Python SDK |
| `starlette` / `uvicorn` | HTTP/SSE 传输 |
| `structlog` | 结构化日志 |
| `prometheus-client` | 监控指标 |

---

## 参考资源

- [MCP 官方规范](https://modelcontextprotocol.io/)
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)
- [MCP 入门课程](../course-04-intro-to-mcp/README.md)
- [Claude Code 文档](https://code.claude.com/docs/en/overview)
