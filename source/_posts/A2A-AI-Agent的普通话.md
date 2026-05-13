---
title: A2A：AI Agent 的"普通话"
date: 2026-05-13
tags:
  - AI
  - Agent
  - A2A
  - 协议
  - 多智能体
categories:
  - AI 技术
description: 深入解析 Google 主导的 A2A（Agent-to-Agent）协议，涵盖核心概念、技术架构、安全机制、与 MCP 的关系，以及生态系统全景。
---

## 引言：为什么需要 Agent 之间对话

2024 年，AI Agent 从概念走向实践。从客服机器人到自动化工作流，各种 Agent 框架层出不穷——LangChain、CrewAI、AutoGen、Semantic Kernel……每个框架都有自己的 Agent 实现方式。

但问题随之而来：**这些 Agent 是孤岛**。

Salesforce 的 CRM Agent 不知道如何与 ServiceNow 的工单 Agent 协作；LangChain 构建的 Agent 无法与 CrewAI 构建的 Agent 对话。每个 Agent 都活在自己的生态里，就像早期的即时通讯工具——QQ 用户不能给 MSN 用户发消息。

2025 年 4 月，Google 主导发布了 A2A（Agent-to-Agent）协议，目标是解决这个问题：让不同框架、不同公司、不同服务器上的 AI Agent 能够互相发现、协商和协作。

A2A 不是又一个 Agent 框架，而是一个通信协议。就像 HTTP 让不同网站能互相通信，A2A 让不同 Agent 能互相协作。

![AI Agent 孤岛问题](https://static.xiangdangnian.net.cn/blog/01-scene-agent-silos.png?v=2)

## A2A 是什么：核心定义

Agent2Agent（A2A）协议是一个开放标准，专门设计用于让 AI Agent 之间能够"通信和互操作"。

这里有一个关键的设计哲学：**不透明协作**。

传统的系统集成要求各方暴露 API、共享数据格式、甚至开放内部状态。但 A2A 不同——它允许 Agent 在不暴露内部记忆、工具或专有逻辑的情况下进行协作。

打个比方：你请一个翻译帮你和外国人交流。翻译不需要知道你脑子里在想什么，只需要知道你说的话，然后翻译给对方。A2A 就是这个"翻译协议"——Agent 之间传递的是任务和结果，而不是内部状态。

这个设计很重要，因为：
- **保护知识产权**：Agent 的内部逻辑不需要公开
- **降低集成成本**：不需要统一数据格式
- **提升安全性**：内部状态不会泄露

## 架构设计：Client-Remote Agent 模型

A2A 采用经典的 Client-Server 架构，但有一些独特之处：

**Client Agent**，接收用户请求的 Agent，负责将任务委派给其他 Agent。

**Remote Agent**，暴露 A2A 端点的 Agent，负责处理具体任务并返回结果。

关键点在于：Client Agent 和 Remote Agent 可以运行在不同的框架上。一个用 LangGraph 构建的 Client Agent 可以与一个用 CrewAI 构建的 Remote Agent 协作，因为它们通过统一的 A2A 协议通信。

这种架构支持三种交互模式：同步请求/响应、流式（SSE）、异步推送通知。Client 发送任务后可以等待 Remote Agent 完成，也可以实时接收进展，或者在长时间运行的任务完成后收到主动通知。

![A2A 架构图](https://static.xiangdangnian.net.cn/blog/02-framework-architecture.png)

## 核心技术组件详解

### Agent Card：发现机制的核心

Agent Card 是 A2A 的"名片"——一个 JSON 元数据文档，描述了 Agent 的身份、能力、技能、端点和认证要求。

它通过 well-known URI 发布，格式类似：

```
https://example.com/.well-known/agent.json
```

Agent Card 包含以下关键信息：身份信息（名称、描述、版本、提供商）、能力声明（是否支持流式、推送通知、扩展卡片）、技能列表（Agent 能做什么，包括输入/输出模式）、接口声明（支持的协议绑定，比如 JSON-RPC、gRPC、HTTP+REST）、安全方案（API Key、OAuth2、OpenID Connect、Mutual TLS），以及可选的密码学签名用于验证真实性。

还有一个**扩展 Agent Card**，当 Client 通过认证后，可以获取更详细的信息。

![Agent Card 结构](https://static.xiangdangnian.net.cn/blog/03-framework-agent-card.png)

### Task：有状态的工作单元

Task 是 A2A 中的基本工作单元，有完整的生命周期。

每个 Task 都有唯一 ID，包含状态、消息历史、输出（Artifacts）和元数据。

Task 支持多轮交互——Agent 可以通过 `INPUT_REQUIRED` 状态暂停任务，等待用户提供更多信息后继续。

![Task 生命周期](https://static.xiangdangnian.net.cn/blog/04-flowchart-task-lifecycle.png)

### Message 与 Part：通信内容结构

Message 是通信轮次，包含 role（USER 或 AGENT）和 parts（内容数组）。每个 Part 可以是文本、原始字节（base64 编码）、文件引用或结构化 JSON。

规范强调：不要用 Message 传递任务输出，应该用 Artifact。Message 用于通信，Artifact 用于输出。

### Artifact：任务输出

Artifact 是 Agent 生成的任务输出——文档、图片、结构化数据等，由 Parts 组成。

Artifact 支持流式交付：通过 `TaskArtifactUpdateEvent` 逐步传递增量更新，新内容可以追加到现有 Artifact，`lastChunk` 标志表示最终交付。

## 通信机制：三种更新传递方式

A2A 提供三种互补的更新传递机制，适用于不同场景：

| 机制 | 延迟 | 复杂度 | 适用场景 |
|------|------|--------|----------|
| **轮询** | 高 | 低 | 简单集成、兼容性要求高 |
| **流式（SSE）** | 低 | 中 | 实时进度更新、UI 交互 |
| **推送通知** | 低 | 高 | 长时间运行的任务、异步工作流 |

如果只是简单集成，用轮询；如果需要实时反馈，用 SSE；如果任务运行时间长（比如超过 30 秒），用推送通知。

## 安全与企业特性

A2A 在设计时就考虑了企业级需求。

认证方案包括 API Key（简单场景）、HTTP Auth（基本认证）、OAuth2（企业级授权）、OpenID Connect（身份验证）和 Mutual TLS（双向证书认证）。

Agent Card 可以用密码学签名，确保来源可信。Client 可以验证签名来确认 Agent 的真实性。

Agent 之间只交换任务和结果，不暴露内部状态、记忆或工具。这意味着内部逻辑不会泄露，不需要统一数据格式，可以安全地与外部 Agent 协作。

## A2A vs MCP：互补而非竞争

很多人会问：A2A 和 MCP（Model Context Protocol）有什么区别？

MCP 让 Agent 能使用工具和数据源，走的是 Client-Server 模式，Agent 通过 MCP 连接到外部工具。

A2A 让 Agent 能与 Agent 协作，走的是对等模式，Agent 通过 A2A 互相委派任务。

MCP 像 USB 接口，让你的电脑能连接各种设备；A2A 像 HTTP 协议，让不同的网站能互相通信。

在实际应用中，一个 Agent 通常同时使用 MCP 和 A2A：通过 MCP 连接到数据库、API 等工具，通过 A2A 与其他 Agent 协作处理复杂任务。

![A2A vs MCP 对比](https://static.xiangdangnian.net.cn/blog/05-comparison-a2a-mcp.png)

## 生态系统与合作伙伴

A2A 发布时，Google 拉来了 50+ 家企业站台。

企业级玩家包括 Salesforce（CRM）、SAP（企业软件）、ServiceNow（IT 服务管理）、Atlassian（Jira、Confluence）、MongoDB（数据库）、Box（云存储）。咨询巨头有 Accenture、Deloitte、Capgemini。技术伙伴包括 IBM Research（联合研发）、Cisco（agntcy 项目），项目归属 Linux Foundation。

SDK 方面，Python、Go、JavaScript、Java、.NET 都有官方支持。

DeepLearning.AI 与 Google Cloud、IBM Research 合作开发了 A2A 短课程，涵盖构建 A2A 兼容 Agent、通过客户端连接 Agent、工作流编排、跨框架医疗多 Agent 系统，以及 A2A 与 MCP 的互补关系。

![A2A 生态系统](https://static.xiangdangnian.net.cn/blog/06-infographic-ecosystem.png?v=2)

## 实际应用场景

### 企业跨系统协作

![企业跨系统协作](https://static.xiangdangnian.net.cn/blog/07-flowchart-enterprise.png)

### 医疗多 Agent 系统

![医疗多 Agent 系统](https://static.xiangdangnian.net.cn/blog/08-flowchart-medical.png)

### 供应链协调

![供应链协调](https://static.xiangdangnian.net.cn/blog/09-flowchart-supply-chain.png)

### DevOps 自动化

![DevOps 自动化](https://static.xiangdangnian.net.cn/blog/10-flowchart-devops.png)

## 路线图与未来展望

A2A 协议还在快速演进中。

Agent 发现方面，正在完善授权方案和可选凭证。协作方面，计划引入 `QuerySkill()` 方法，支持运行时查询未知技能。任务 UX 方面，目标是在任务运行中动态协商交互方式，比如添加音视频。传输方面，正在扩展客户端方法，提升流式和推送通知的可靠性。

## 快速上手指南

### 安装 SDK

```bash
# Python
pip install a2a-sdk

# JavaScript
npm install @a2a-js/sdk

# Go
go get github.com/a2aproject/a2a-go
```

### 创建 Agent Card

```json
{
  "name": "My Agent",
  "description": "一个能处理数据分析的 Agent",
  "url": "https://myagent.example.com",
  "capabilities": {
    "streaming": true,
    "pushNotifications": true
  },
  "skills": [
    {
      "id": "data-analysis",
      "name": "数据分析",
      "description": "分析 CSV/JSON 数据并生成报告",
      "tags": ["data", "analysis", "report"],
      "inputModes": ["text", "file"],
      "outputModes": ["text", "file"]
    }
  ],
  "security": {
    "schemes": ["oauth2"]
  }
}
```

### 实现 A2A 服务器

```python
from a2a.server import A2AServer
from a2a.types import Task, TaskStatus, Artifact

server = A2AServer(host="0.0.0.0", port=8000)

@server.handle_task
async def handle_task(task: Task) -> Task:
    # 处理任务逻辑
    result = await process_task(task)
    
    # 返回结果
    task.status = TaskStatus(state="completed")
    task.artifacts = [Artifact(parts=[{"text": result}])]
    return task

server.run()
```

### 客户端调用

```python
from a2a.client import A2AClient

client = A2AClient("https://remote-agent.example.com")

# 发送任务
task = client.send_task({
    "message": {
        "role": "user",
        "parts": [{"text": "分析这份销售数据"}]
    }
})

# 等待结果
result = client.get_task(task.id)
print(result.artifacts)
```

## 总结

A2A 协议解决了一个关键问题：如何让不同的 AI Agent 能够互相协作。

它的核心价值在于：统一的通信协议、保护 Agent 内部状态的不透明性、与 MCP 各司其职的互补关系，以及 50+ 合作伙伴的生态支持。

对于开发者来说，你可以用任何框架构建 Agent，你的 Agent 可以与任何其他 Agent 协作，不需要担心底层实现差异。

对于企业来说，可以整合来自不同供应商的 Agent，构建跨系统的自动化工作流，安全地与外部 Agent 协作。

A2A 还在早期阶段，但它代表了 AI Agent 互操作的方向。随着生态系统的成熟，更多创新的多 Agent 应用场景会出现。