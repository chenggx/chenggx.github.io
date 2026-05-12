---
title: MCP 协议详解：从原理到生态全景
date: 2026-05-12
tags:
  - MCP
  - Model Context Protocol
  - AI
  - Anthropic
categories:
  - AI 技术
description: 详解 MCP（Model Context Protocol）协议的架构、核心概念、传输机制与生态全景，从 Host-Client-Server 三层模型到 Tools/Resources/Prompts 三种能力，再到与 Function Calling 的对比和实际工作流程。
---

## 引言：AI 的碎片化集成困境

想让 AI 帮你查一下 Notion 里的笔记，或者操作一下 GitHub 上的代码仓库，结果发现每接一个数据源都要写一套专门的集成代码。Notion 一套，Slack 一套，GitHub 一套，数据库又一套。

开发者在重复造轮子，AI 应用被困在信息孤岛里——模型再强大，看不到数据也是白搭。

2024 年 11 月，Anthropic 发布了 **MCP（Model Context Protocol）**，一个开源标准协议，目标是用一个统一的接口取代所有碎片化的集成。官方的类比很直观：**MCP 就是 AI 应用的 USB-C 接口**。就像 USB-C 让你用同一根线给手机、电脑、耳机充电一样，MCP 让 AI 应用用同一个协议连接任何数据源。

这篇文章从协议原理讲起，深入理解 MCP 的架构设计、核心概念和生态全景。

![MCP 核心类比：AI 应用的 USB-C 接口](https://static.xiangdangnian.net.cn/blog/01-infographic-mcp-usb-c.png)

## MCP 是什么

MCP（Model Context Protocol）是由 **Anthropic** 主导发起的开放协议，于 2024 年 11 月首次发布。Anthropic 就是做 Claude 的那家公司，他们的 CTO Dhanji R. Prasanna 在发布时说了一句很到位的话：「开放技术如 MCP 是连接 AI 与真实应用的桥梁」。

虽然 Anthropic 发起了这个项目，但 MCP 从一开始就定位为**社区驱动的开放标准**，而不是某家公司的私有协议。规范、SDK、参考实现全部开源在 GitHub 的 `modelcontextprotocol` 组织下，任何人都可以参与贡献。

MCP 的设计灵感来自微软的 **LSP（Language Server Protocol）**。LSP 解决了编程语言支持的碎片化问题——编辑器不需要为每种语言写专门的集成，语言服务器实现一次 LSP 就行。MCP 想在 AI 领域做同样的事情：**写一个 MCP Server，所有支持 MCP 的 AI 客户端都能直接用**。

目前 Claude、ChatGPT、VS Code Copilot、Cursor、Zed、Replit 等主流 AI 工具都已支持 MCP。生态已经初具规模——有官方的 TypeScript/Python SDK，有 Google Drive、Slack、GitHub、Postgres 等预置 Server，也有社区贡献的各种集成。

## 架构全景：Host → Client → Server

MCP 的架构分三个角色：

![MCP 三层架构：宿主应用 → 客户端 → 服务端](https://static.xiangdangnian.net.cn/blog/02-framework-host-client-server.png)

- **Host**：AI 应用本身（Claude Desktop、VS Code 等），负责协调多个连接
- **Client**：Host 内部的连接器，每个 Client 与一个 Server 保持一对一连接
- **Server**：暴露数据和能力的服务程序，可以本地运行也可以远程部署

一个直观的例子：VS Code 是 Host，它同时连接了 Sentry Server（查看错误监控）和文件系统 Server（读写本地文件）。每连接一个 Server，VS Code 内部就实例化一个对应的 Client。

**本地 vs 远程 Server**：使用 STDIO 传输的 Server 通常运行在本地（如 Claude Desktop 启动的文件系统 Server），使用 Streamable HTTP 的 Server 运行在远程（如 Sentry 的 MCP Server）。

## 协议两层：数据层 + 传输层

MCP 的协议设计分为两层，各司其职。

### 数据层：JSON-RPC 2.0

数据层基于 **JSON-RPC 2.0** 消息格式，定义了客户端和服务端之间的通信语义。它包含：

- **生命周期管理**：连接初始化、能力协商、连接终止
- **Server 原语**：Tools、Resources、Prompts 的发现与调用
- **Client 原语**：Sampling（请求 LLM 补全）、Elicitation（向用户请求信息）、Logging
- **通知机制**：实时更新（如工具列表变化时主动通知客户端）

所有消息都是标准的 JSON-RPC 格式：

```json
// 请求
{"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}}

// 响应
{"jsonrpc": "2.0", "id": 1, "result": {"tools": [...]}}
```

### 传输层：STDIO 和 Streamable HTTP

传输层定义了数据的传输通道，MCP 目前支持两种：

| 传输方式 | 适用场景 | 特点 |
|---------|---------|------|
| **STDIO** | 本地进程通信 | 标准输入/输出流，零网络开销，一个 Server 通常服务一个 Client |
| **Streamable HTTP** | 远程通信 | HTTP POST + SSE 流式响应，支持标准 HTTP 认证，一个 Server 可服务多个 Client |

STDIO 的工作方式很简单：Host 启动一个子进程运行 Server，通过 stdin/stdout 交换 JSON-RPC 消息。这就是为什么本地 MCP Server 本质上就是一个命令行程序。

Streamable HTTP 则是标准的 HTTP 通信：客户端 POST JSON-RPC 请求到 Server，Server 可以返回普通 JSON，也可以返回 SSE 流式响应。适合远程部署和多客户端场景。

## MCP Server 能做什么：Tools、Resources、Prompts

MCP Server 可以向 AI 客户端提供三种能力。理解它们最好的方式就是看一个具体场景。

假设你做了一个旅行 MCP Server，它能给 AI 提供三种能力：

### Tools：AI 可以做的事

Tools 就是 AI 能调用的函数。你说"帮我搜一下明天北京到上海的航班"，AI 就会调用 Server 的 `searchFlights` 工具去查。

Tools 是**AI 自己决定什么时候用**的。你不需要手动告诉 AI "现在该调这个工具了"，它会根据你的意图自动选择合适的工具。

但有一点很重要：AI 决定用哪个工具，**不代表它可以直接执行**。敏感操作（删除文件、发邮件、付款）通常需要用户点击确认。这是安全底线。

常见的 Tool 例子：
- 搜索航班、查天气、查数据库
- 创建日历事件、发送消息
- 读写文件、调用第三方 API

### Resources：AI 能看到的信息

Resources 是只读的数据源。比如你旅行 Server 里存了用户的日历、旅行偏好、过往行程，AI 可以把这些信息读出来作为上下文。

和 Tools 不同，**Resources 不是 AI 自己去读的**，而是宿主应用（比如 Claude Desktop）决定把哪些数据塞给 AI。你可以理解为 Resources 是 AI 的参考资料，应用帮它选好了放在桌上。

常见的 Resource 例子：
- 用户的日历数据
- 数据库的 schema 定义
- API 文档
- 项目配置文件

### Prompts：用户可以直接用的模板

Prompts 是预设的工作流模板，**用户主动选择使用**。比如你的旅行 Server 提供了一个"规划假期"的 Prompt，用户点一下，填上目的地、天数、预算，AI 就会按照模板帮你一步步规划行程。

Prompts 的价值在于：把复杂任务拆解成结构化的步骤，用户不用从零开始描述需求。

常见的 Prompt 例子：
- "规划假期" — 自动搜索航班、推荐酒店、安排行程
- "总结会议" — 读取日历、整理议题、生成纪要
- "写周报" — 汇总本周工作、生成格式化文档

### 一句话总结

| | 谁在用 | 干什么 | 一句话 |
|---|--------|--------|--------|
| **Tools** | AI | 执行操作 | AI 的手，能做事 |
| **Resources** | 应用 | 提供数据 | AI 的参考资料，应用帮它选好 |
| **Prompts** | 用户 | 预设流程 | 用户的快捷按钮，一键启动复杂任务 |

还是那个旅行场景：你说"帮我规划一次巴塞罗那旅行"，AI 会先看 Resources（你的日历、旅行偏好），然后调 Tools（搜航班、订酒店），整个流程可以用一个 Prompt 模板来组织。三者配合，才能完成一个完整的任务。

![MCP 三种能力对比：工具、资源、提示词](https://static.xiangdangnian.net.cn/blog/03-comparison-tools-resources-prompts.png)

## 连接生命周期

MCP 是有状态的协议，连接需要经历完整的生命周期：

![MCP 连接生命周期：初始化 → 能力协商 → 工具发现 → 工具调用](https://static.xiangdangnian.net.cn/blog/07-flowchart-lifecycle.png)

三个关键阶段：

1. **初始化**：双方交换 `protocolVersion` 和 `capabilities`，协商支持的功能子集
2. **发现**：客户端通过 `*/list` 方法枚举 Server 暴露的 Tools、Resources、Prompts
3. **调用**：客户端发送 `tools/call`、`resources/read`、`prompts/get` 等请求，Server 返回结果

## 安全原则

MCP 规范明确要求所有实现者遵守以下安全原则：

1. **用户同意与控制**：用户必须明确同意所有数据访问和操作
2. **数据隐私**：Host 不得未经用户同意将资源数据传输到其他地方
3. **工具安全**：工具的行为描述（annotations）被视为不可信，除非来自可信 Server
4. **LLM 采样控制**：Server 请求 LLM 补全时，用户可以控制是否允许、发送什么 prompt、Server 能看到什么结果

MCP 本身无法在协议层面强制执行这些原则，但规范要求实现者在应用层做好这些控制。

## MCP 的工作流程：一个请求是怎么走完的

理解了架构和原语，我们来看一个完整的请求流程。

用户在 Claude Desktop 里说："帮我查一下明天北京到上海的航班，然后加到我的日历里。"

![MCP 请求完整流程：从用户说话到回复用户](https://static.xiangdangnian.net.cn/blog/08-flowchart-workflow.png)

整个过程中，Claude 自己决定该用哪个工具、传什么参数。MCP 只负责让这些工具调用变得标准化——不管你的 Flight Server 是用 Python、TypeScript 还是 Java 写的，Claude 都能用同样的方式调用它。

## MCP 和 Function Calling 的区别

很多人会问：ChatGPT 和 Claude 本身就有 Function Calling（函数调用）能力了，为什么还需要 MCP？

两者解决的问题不一样：

| | Function Calling | MCP |
|---|---|---|
| **本质** | 模型调用你定义的函数 | 标准化的外部服务协议 |
| **工具定义在哪** | 写在你的代码里，随请求发送 | 在 MCP Server 中，独立部署 |
| **谁管理工具** | 你的应用 | MCP Server 自己管理 |
| **复用性** | 换个 AI 客户端就要重新写 | 写一次，所有 MCP 客户端通用 |
| **发现机制** | 没有，客户端必须知道有哪些函数 | 有，通过 `tools/list` 自动发现 |

打个比方：Function Calling 就像你每次去餐厅都自己带菜谱，告诉厨师做什么。MCP 就像餐厅本身有一份标准菜单，你只需要点菜就行——不管你用什么方式去点（APP、电话、到店），菜单是一样的。

**什么时候用 Function Calling**：简单的内部工具，只有你的应用会调用。

**什么时候用 MCP**：你想让多个 AI 客户端都能用你的工具，或者你想建立一个可复用的工具生态。

![函数调用 vs MCP 协议：餐厅类比](https://static.xiangdangnian.net.cn/blog/05-comparison-fc-vs-mcp.png)

## MCP 生态：谁能用、有什么

截至 2025 年，MCP 生态已经初具规模。

### 支持 MCP 的客户端

| 客户端 | 类型 | 说明 |
|--------|------|------|
| Claude Desktop | 桌面应用 | Anthropic 官方，最早支持 MCP |
| Claude Code | CLI 工具 | Anthropic 的命令行编程助手 |
| ChatGPT | 网页/桌面 | OpenAI 已支持 MCP |
| VS Code Copilot | IDE | 微软的 AI 编程助手 |
| Cursor | IDE | AI-first 的代码编辑器 |
| Zed | 编辑器 | 高性能代码编辑器 |
| Replit | 云端 IDE | 在线编程平台 |

### 官方预置 Server

Anthropic 在 GitHub 上维护了一批官方 MCP Server，覆盖常见企业系统：

- **文件系统**：读写本地文件
- **GitHub**：仓库管理、Issue、PR
- **PostgreSQL**：数据库查询
- **Google Drive**：文件访问
- **Slack**：消息读写
- **Puppeteer**：浏览器自动化

### 各语言 SDK

| 语言 | 包名 | 说明 |
|------|------|------|
| TypeScript | `@modelcontextprotocol/sdk` | 官方，支持 Node/Bun/Deno |
| Python | `mcp` | 官方，支持 stdio/SSE/Streamable HTTP |
| Java | `io.modelcontextprotocol:sdk` | 官方 |
| Kotlin | `io.modelcontextprotocol:kotlin-sdk` | 官方 |
| C# | `ModelContextProtocol` | 官方 |
| Ruby | 社区维护 | |
| Rust | 社区维护 | |
| Go | 社区维护 | |

![MCP 生态全景](https://static.xiangdangnian.net.cn/blog/06-infographic-mcp-ecosystem.png)

## MCP Apps：从文本到交互界面

MCP 最近引入了一个叫 **MCP Apps** 的扩展：工具可以在 AI 客户端的沙盒 iframe 中渲染交互式 HTML 界面。

什么意思？以前 MCP 工具只能返回文本或 JSON。现在，一个天气 Dashboard 工具可以直接在 Claude Desktop 里渲染出一个带图表的 HTML 页面，用户可以在里面交互操作。

这让 MCP Server 的输出从纯文本扩展到了富交互界面，适合构建数据可视化、管理面板、表单等场景。

## 调试 MCP Server

官方提供了一个叫 **MCP Inspector** 的交互式调试工具。它能连接到你的 MCP Server，浏览所有可用的 Tools、Resources、Prompts，还能直接调用测试。

启动方式：

```bash
# 调试远程 Server
npx @modelcontextprotocol/inspector http://localhost:8080/mcp

# 调试本地 Server（STDIO 方式）
npx @modelcontextprotocol/inspector python my_server.py
```

Inspector 会打开一个 Web 界面，左边列出所有可用能力，右边可以填写参数并调用，实时看到返回结果。

如果你在用 Claude Desktop，也可以查看日志来排查问题：

```bash
# macOS 上 Claude Desktop 的 MCP 日志
tail -n 20 -f ~/Library/Logs/Claude/mcp*.log
```

## MCP 的安全设计

MCP 规范把安全放在了核心位置，而不是事后补丁。几个关键设计：

**用户同意优先**：AI 不能偷偷调用工具。每次工具执行前，用户都应该有机会看到并批准。这和手机 App 请求权限是一个道理——你装了一个天气 App，它要访问你的位置，系统会弹窗问你同不同意。

**工具描述不可信**：Server 给工具写的描述（比如"这个工具只读不写"）不能完全信任。恶意 Server 可能撒谎。所以客户端需要根据工具的实际行为来判断安全性，而不是只看描述。

**LLM 采样可控**：Server 可以请求 Client 让 LLM 帮忙做推理（这叫 Sampling），但用户可以控制是否允许、发什么 prompt、Server 能看到什么结果。这是防止 Server 通过 AI 做它不该做的事。

**数据不外传**：Host 拿到的资源数据不能在用户不知情的情况下被发送到其他地方。

## 未来方向

MCP 还在快速演进中。几个值得关注的方向：

- **Tasks（实验性）**：支持长时间运行的任务，可以异步执行、查询进度、延迟获取结果
- **Elicitation**：Server 可以主动向用户请求额外信息（比如"你想订哪个航班？"），而不仅仅是被动响应
- **OAuth 2.1 标准化**：远程 MCP Server 的认证正在走向标准化
- **更多语言 SDK**：Go、Rust 等社区 SDK 在持续完善
- **企业级部署**：远程 MCP Server 的生产级部署方案逐渐成熟

## 总结

MCP 正在成为 AI 应用连接外部世界的事实标准。它的核心思想很简单：

- **一个协议，取代 N 种集成**：写一次 MCP Server，所有支持 MCP 的 AI 客户端都能用
- **三层架构，职责清晰**：Host 管协调，Client 管连接，Server 管能力暴露
- **三种能力，各司其职**：Tools 是 AI 的手，Resources 是 AI 的眼，Prompts 是用户的快捷按钮
- **安全优先**：用户始终拥有数据和操作的最终控制权
