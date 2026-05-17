---
title: AI Agent 浏览器操作方案全景：从 Playwright 到 browser-harness
date: 2026-05-17
tags: [AI, Agent, Playwright, browser-use, MCP, 浏览器自动化]
categories: [AI 技术]
description: 全面对比 AI Agent 浏览器操作的四种方案：Playwright MCP、Playwright CLI、browser-use、browser-harness，帮助开发者选择最适合的工具。
---

# AI Agent 浏览器操作方案全景：从 Playwright 到 browser-harness

AI Agent 要帮人干活，光会聊天不够，得能打开浏览器、点击按钮、填写表单、提交订单。2024-2025 年这个领域涌现了大量工具，从微软的 Playwright 到 browser-use 的 browser-harness，每种方案都有不同的设计理念。本文梳理四种主流方案的架构、优劣和适用场景。

![AI Agent 浏览器自动化概念图](https://static.xiangdangnian.net.cn/blog/01-scene-ai-agent-browser.png)

## 架构分层

整体架构分三层：

![三层架构可视化](https://static.xiangdangnian.net.cn/blog/02-framework-three-layer-architecture.png)

MCP 是 Anthropic 提出的协议标准，负责 LLM 和工具之间的通信，本身不是浏览器工具。Playwright 是底层自动化框架。browser-use 和 browser-harness 是上层 Agent 框架。

## 1. Playwright：底层基础

[Playwright](https://playwright.dev) 是微软开发的跨浏览器自动化框架，支持 Chromium、Firefox、WebKit。

核心能力包括：自动等待（元素可交互时才执行操作，不用人工 sleep）、语义化定位器（`getByRole`、`getByLabel` 等）、测试隔离（每个测试独立浏览器上下文）、执行追踪（轨迹、截图、视频录制）。

Playwright 本身不涉及 AI，但它是其他方案的底层依赖。browser-use 基于 Playwright 构建，Playwright MCP 将 Playwright 暴露为 MCP 工具，Anthropic 的 browser-use-demo 也使用 Playwright。

## 2. Playwright MCP

微软官方的 [Playwright MCP 服务器](https://github.com/microsoft/playwright-mcp)，把 Playwright 的能力通过 MCP 协议暴露给 LLM。

LLM 不看截图，而是通过结构化可访问性快照（accessibility tree）与网页交互：

```
用户 → "打开 example.com"
LLM → 调用 playwright.navigate("https://example.com")
LLM ← 返回页面结构树（可点击元素、输入框等）
LLM → 调用 playwright.click("button[登录]")
```

这种模式的优势是无需视觉模型，纯结构化数据，工具调用明确，不会像截图那样产生歧义。局限在于 token 消耗较大，每次交互都要加载工具 schema 和可访问性树，大型 SPA 的可访问性树可能很庞大。

适用场景：探索性自动化（不确定页面结构时）、自愈测试（页面变化时自动调整）、长时间运行的自主工作流。

配置示例：

```json
{
  "mcpServers": {
    "playwright": {
      "command": "npx",
      "args": ["@playwright/mcp@latest"]
    }
  }
}
```

支持 VS Code、Cursor、Claude Desktop、Goose 等主流工具。

## 3. Playwright CLI

微软 2025 年新推出的 [Playwright CLI](https://github.com/microsoft/playwright-cli)，专门为编码代理（coding agent）设计。

| 维度 | Playwright MCP | Playwright CLI |
|------|---------------|----------------|
| Token 效率 | 加载完整 schema + 可访问性树 | 只需简短命令 |
| 适用场景 | 探索性自动化、长时工作流 | 编码代理（Claude Code、Copilot） |
| 监控能力 | 无 | 可视化仪表板 |
| 安装方式 | npx @playwright/mcp | npm i -g @playwright/cli |

编码代理（如 Claude Code）的上下文窗口要同时容纳代码库、测试、推理和浏览器操作，MCP 的大 schema 会挤占上下文空间。CLI 用简短命令代替，更省 token。

使用示例：

```bash
# 启动浏览器
playwright-cli open https://example.com --headed

# 输入文本
playwright-cli type "Buy groceries"

# 截图
playwright-cli screenshot

# 可视化监控所有会话
playwright-cli show
```

安装到 Claude Code：

```bash
claude mcp add playwright npx @playwright/mcp@latest
```

## 4. browser-use：AI 原生浏览器操作

[browser-use](https://github.com/browser-use/browser-use) 是专门为 AI Agent 设计的浏览器操作库，GitHub 79k+ stars。

核心特性：视觉 + DOM 双重理解（结合截图和页面结构）、多标签管理（切换、关闭、多任务并行）、元素历史追踪（记住之前操作过的元素）、LLM 无关（支持 OpenAI、Anthropic、Google、本地模型）、自纠正（操作失败时自动重试）。

快速上手：

```python
from browser_use import Agent, Browser, ChatBrowserUse
import asyncio

async def main():
    browser = Browser()
    agent = Agent(
        task="Find the number of stars of the browser-use repo",
        llm=ChatBrowserUse(),
        browser=browser,
    )
    await agent.run()

if __name__ == "__main__":
    asyncio.run(main())
```

browser-use 经历了三个阶段的架构演进：从 Playwright 封装（抽象太多，LLM 受限），到 CDP 直连（仍有 watchdog 等抽象），再到 browser-harness（Agent 自写 helper）。

Browser Use Cloud 还提供云端版本，包含隐身浏览器、代理轮换、验证码处理、并行执行等能力。

## 5. browser-harness：极简 CDP Harness

[browser-harness](https://github.com/browser-use/browser-harness) 是 browser-use 团队的最新项目，体现了 Bitter Lesson 理念：不要给 LLM 抽象层，直接给底层 CDP，让 Agent 自己写需要的 helper。

团队之前写了大量 watchdog 服务来处理 Chrome 崩溃、target 断开、OOM 等问题。后来发现 LLM 训练数据里有大量 CDP 和 Chrome 崩溃处理的代码，它自己就能处理这些情况。把 CDP 直接暴露给 LLM，它遇到崩溃时自己重连，发现 helper 缺失时自己写，处理跨域 iframe、Shadow DOM 等复杂场景。

整个 harness 只有约 600 行代码，4 个核心文件：

```
agent-workspace/
├── SKILL.md              # 告诉 Agent 怎么用
├── agent_helpers.py      # CDP 封装，Agent 可编辑
└── domain-skills/        # 可复用的站点技能
    ├── github/
    ├── linkedin/
    └── amazon/
```

- `run.py`（13 行）：运行 Python，预加载 helpers
- `helpers.py`（192 行）：CDP 薄封装，Agent 可编辑
- `daemon.py`（220 行）：保持 CDP websocket 连接
- `SKILL.md`：Agent 使用说明

自修复循环的例子：当 Agent 需要上传文件但 `upload_file()` 不存在时，它会 grep `helpers.py`，确认函数缺失，编辑 helpers.py 添加该函数，然后重新执行。Agent 不是写全新代码，而是补全缺失的函数，和它修复任何代码库的缺失 import 一样。

![自修复循环流程图](https://static.xiangdangnian.net.cn/blog/03-flowchart-self-heal-loop.png)

真实案例：Agent 自己发现并实现了 `upload_file()`；12MB 文件超过 CDP 限制时自动切换分块模式；Azure 管理门户的 iframe 嵌套界面，Agent 用坐标级 `Input.dispatchMouseEvent` 穿透。

## 6. Anthropic Browser Use Demo

Anthropic 官方的[浏览器自动化参考实现](https://github.com/anthropics/anthropic-quickstarts/tree/main/browser-use-demo)，展示如何用 Claude + Playwright 构建浏览器自动化。

功能包括：DOM 访问（读取页面结构和元素引用）、导航控制（浏览 URL、管理历史）、表单操作（直接设置输入值）、智能滚动（滚动到特定元素）、视觉捕获（截图和区域放大）。

安全设计方面采用容器化隔离运行，不使用真实凭证，支持域名白名单限制。

| 维度 | Anthropic demo | browser-use |
|------|---------------|-------------|
| 定位 | 参考实现 | 生产级库 |
| LLM | 仅 Claude | 多 LLM 支持 |
| 功能 | 基础浏览器操作 | 完整 Agent 框架 |
| 部署 | Docker 容器 | 本地/云端 |

## 7. 方案对比与选型

| 方案 | 层级 | 核心理念 | Token 效率 | 自修复 | 适用场景 |
|------|------|----------|-----------|--------|----------|
| Playwright MCP | L1+L2 | 结构化快照 | 中 | 否 | 探索性自动化 |
| Playwright CLI | L1 | 命令行工具 | 高 | 否 | 编码代理 |
| browser-use | L3 | AI 原生库 | 中 | 是 | AI Agent 开发 |
| browser-harness | L3 | 极简 CDP | 高 | 是 | 高级自主 Agent |

选型建议：编码代理用户（Claude Code、Copilot）用 Playwright CLI；需要快速上手的 AI Agent 开发者用 browser-use；需要最大灵活性的用 browser-harness；测试工程师需要 AI 辅助测试用 Playwright MCP；想理解原理用 Anthropic demo。

场景推荐：快速验证 AI 能否操作某网站选 Playwright MCP；构建生产级 AI Agent 选 browser-use + Cloud；需要 Agent 自主适应复杂网站选 browser-harness；编码代理需要浏览器操作选 Playwright CLI。

![四种方案速览](https://static.xiangdangnian.net.cn/blog/04-comparison-four-solutions-comparison.png)

## 8. 总结

这四种方案分层互补：Playwright 是地基，所有方案都依赖它；MCP/CLI 是桥梁，让 LLM 能调用 Playwright；browser-use 是成品车，开箱即用；browser-harness 是越野车，给 Agent 最大自由度。

趋势是给 LLM 越来越底层的控制权，让它自己决定需要什么抽象。browser-harness 的 Bitter Lesson 说得直接：你的 helper 也是抽象，删掉它们，让 Agent 自己写需要的东西。

---

**参考链接**

- [Playwright](https://playwright.dev)
- [Playwright MCP](https://github.com/microsoft/playwright-mcp)
- [Playwright CLI](https://github.com/microsoft/playwright-cli)
- [browser-use](https://github.com/browser-use/browser-use)
- [browser-harness](https://github.com/browser-use/browser-harness)
- [The Bitter Lesson of Agent Harnesses](https://browser-use.com/posts/bitter-lesson-agent-harnesses)
- [Anthropic Browser Use Demo](https://github.com/anthropics/anthropic-quickstarts/tree/main/browser-use-demo)
