---
{"dg-publish":true,"permalink":"/01-技术分享/GitHub 上值得关注的 AI Agent 开源项目（2026）/","tags":["AI-agent","开源项目","综述","LLM","浏览器自动化","github","agent"],"noteIcon":"","created":"2026-04-27T16:31:06.241+08:00","updated":"2026-04-28T11:14:09.648+08:00"}
---


# GitHub 上值得关注的 AI Agent 开源项目（2026）

> AI Agent 赛道正在爆发。从框架、浏览器自动化、编码助手到操作系统级控制，GitHub 上涌现了大量优秀项目。这篇文章按类别梳理当前最值得关注的开源项目，帮你快速建立全景认知。

## 一、为什么现在要关注 AI Agent？

Gartner 预测，到 2026 年底 40% 的企业应用将内置任务型 AI Agent，而 2025 年这个数字不到 5%。全球 Agent 市场规模预计从 2025 年的 78 亿美元增长到 2030 年的 526 亿美元。

这不是概念炒作——LLM 的推理能力已经强到可以自主规划、调用工具、执行多步任务。围绕这个能力，GitHub 上形成了一个完整的开源生态。

## 二、Agent 框架：构建 Agent 的基础设施

这是最核心的赛道。框架决定了你怎么定义 Agent、怎么编排多 Agent 协作、怎么管理状态和记忆。

### LangChain / LangGraph

| 项目 | Stars | 月下载量 | 语言 |
|------|-------|---------|------|
| [LangChain](https://github.com/langchain-ai/langchain) | 122.8k | — | Python |
| [LangGraph](https://github.com/langchain-ai/langgraph) | 24.8k | 34.5M | Python |

LangChain 是 LLM 应用开发的事实标准，提供链式调用、RAG、工具集成等基础能力。LangGraph 是它的 Agent 编排层，用有向图定义 Agent 的状态流转，支持人工介入（human-in-the-loop）和长期记忆。

企业采用率最高：Klarna 用它替代了 853 名客服，年省 6000 万美元；Uber、Cisco、LinkedIn、BlackRock 都在用。

**适合**：需要精细控制状态流转的企业级应用。

### Dify

| Stars | 语言 | 定位 |
|-------|------|------|
| [129.8k](https://github.com/langgenius/dify) | Python | 低代码 Agent 平台 |

GitHub 上 star 数最高的 Agent 项目之一。提供可视化拖拽界面，支持 RAG、Function Calling、ReAct 策略，兼容数百种 LLM。非技术人员也能搭建 Agent 工作流。

**适合**：不想写代码、快速搭建 Agent 原型的团队。

### CrewAI

| Stars | 月下载量 | 语言 |
|-------|---------|------|
| [44.3k](https://github.com/crewAIInc/crewAI) | 5.2M | Python |

核心概念是"角色扮演"——你给每个 Agent 定义角色（研究员、写手、审核员），它们按分工协作完成任务。API 简洁，20 分钟就能跑起来一个多 Agent 系统。

**适合**：快速搭建多 Agent 协作，不想处理复杂依赖。

### AutoGen → Microsoft Agent Framework

| Stars | 月下载量 | 语言 |
|-------|---------|------|
| [54.6k](https://github.com/microsoft/autogen) | 856k | Python |

微软研究院出品，事件驱动的多 Agent 对话框架。2025 年 10 月微软将 AutoGen 与 Semantic Kernel 合并为统一的 [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/)，AutoGen 本身进入维护模式（只修 bug）。

**适合**：微软生态用户、复杂多 Agent 对话场景。

### OpenAI Agents SDK

| Stars | 月下载量 | 语言 |
|-------|---------|------|
| [21k](https://github.com/openai/openai-agents-python) | 10.3M | Python |

OpenAI 官方出品，轻量级多 Agent 框架。虽然叫 OpenAI，但兼容 100+ LLM。内置 tracing 和 guardrails，学习曲线低。

**适合**：快速原型、通用 Agent 开发。

### Google ADK

| Stars | 月下载量 | 语言 |
|-------|---------|------|
| [17.8k](https://github.com/google/adk-python) | 3.3M | Python |

Google 的 Agent 开发工具包，深度集成 Gemini 和 Vertex AI。支持层级式 Agent 组合，不到 100 行代码就能搭建一个 Agent。

**适合**：Google Cloud 用户、Gemini 生态。

### Agno

| Stars | 语言 | 定位 |
|-------|------|------|
| [36.4k](https://github.com/agno-agi/agno) | Python | 多 Agent 运行时 |

提供 Agent 运行时和控制平面，专注于大规模 Agent 部署和管理。

### MetaGPT

| Stars | 语言 | 定位 |
|-------|------|------|
| [61.9k](https://github.com/FoundationAgents/MetaGPT) | Python | 模拟软件公司 |

用多 Agent 模拟一个软件公司——产品经理、架构师、工程师各司其职。给它一句需求描述，它能输出 PRD、架构设计和代码。

### Mastra

| Stars | 月下载量 | 语言 |
|-------|---------|------|
| [21.2k](https://github.com/mastra-ai/mastra) | 1.77M | TypeScript |

TypeScript 生态的 Agent 框架，由 Gatsby 团队打造，YC 背书。Replit 的 Agent 3 就是基于 Mastra 构建的，任务成功率从 80% 提升到 96%。

**适合**：JavaScript/TypeScript 开发者。

### 框架对比速览

| 框架 | Stars | 核心特点 | 最适合 |
|------|-------|---------|--------|
| LangGraph | 24.8k | 状态图编排、企业级 | 复杂工作流 |
| Dify | 129.8k | 低代码可视化 | 非技术团队 |
| CrewAI | 44.3k | 角色扮演、简单 | 快速多 Agent |
| AutoGen | 54.6k | 事件驱动、微软生态 | 多 Agent 对话 |
| OpenAI SDK | 21k | 轻量、100+ LLM | 通用原型 |
| Google ADK | 17.8k | Gemini 集成 | Google 生态 |
| Mastra | 21.2k | TypeScript 原生 | JS/TS 团队 |

## 三、浏览器自动化：让 Agent 上网

这是 Agent 落地最直接的场景——让 AI 像人一样操作浏览器。

### Browser Use

| Stars | 语言 | 定位 |
|-------|------|------|
| [90k+](https://github.com/browser-use/browser-use) | Python | 全自主浏览器 Agent |

目前最火的浏览器 Agent 项目。给它一个自然语言指令，它自主规划、导航、点击、填表、提取数据。支持多步任务，有完整的 Agent 循环。

### Browser Harness

| Stars | 语言 | 定位 |
|-------|------|------|
| [7.2k](https://github.com/browser-use/browser-harness) | Python | 自愈式 CDP 工具 |

Browser Use 团队的另一个项目。只有 592 行代码，直接暴露 CDP 给 LLM，Agent 缺什么函数就自己写。设计哲学极端但有趣——"你的 helper 也是抽象，删掉它们"。

### Stagehand

| 语言 | 定位 |
|------|------|
| [TypeScript](https://github.com/browserbase/stagehand) | 混合式浏览器自动化 |

Browserbase 出品。提供三个 AI 原语：`act`（执行操作）、`extract`（提取数据）、`observe`（观察页面）。TypeScript 优先，适合在传统 Playwright 代码中混入 AI 能力。

**Browser Use vs Stagehand**：Browser Use 是全自主 Agent 循环，适合复杂多步任务；Stagehand 是混合控制，适合在确定性代码中嵌入 AI 辅助。

### Skyvern

| 语言 | 定位 |
|------|------|
| [Python](https://github.com/Skyvern-AI/skyvern) | 视觉驱动的浏览器自动化 |

用视觉模型理解页面，不依赖 DOM 选择器。对页面结构变化的鲁棒性更强。

## 四、编码 Agent：让 AI 写代码

### OpenHands

| Stars | 语言 | 定位 |
|-------|------|------|
| [70k+](https://github.com/All-Hands-AI/OpenHands) | Python | 自主编码 Agent |

前身是 OpenDevin，目前最流行的开源编码 Agent。能写代码、跑终端命令、浏览网页、提交 PR，全部在 Docker 沙箱里运行。MIT 协议，490+ 贡献者。

### SWE-agent

| 定位 | 论文 |
|------|------|
| [自动修复 GitHub Issue](https://github.com/SWE-agent/SWE-agent) | NeurIPS 2024 |

给它一个 GitHub Issue，它自动分析代码、定位问题、写修复补丁。也可用于竞赛编程和安全研究。

### Open SWE

| 出品方 | 定位 |
|--------|------|
| [LangChain](https://github.com/langchain-ai/open-swe) | 自主编码 Agent 框架 |

2026 年 3 月发布。Agent 能分析代码库、规划实现、写代码、跑测试、自我 review、提交 PR，全程异步。

## 五、Computer Use：操控整个桌面

不只是浏览器——这些项目让 AI 控制整个操作系统。

### CUA (Computer Use Agent)

| Stars | 定位 |
|-------|------|
| [trycua/cua](https://github.com/trycua/cua) | 跨平台桌面 Agent 基础设施 |

提供沙箱、SDK 和 benchmark，让 AI Agent 控制完整桌面环境（macOS、Linux、Windows）。

### UFO

| 出品方 | 定位 |
|--------|------|
| [微软](https://github.com/microsoft/UFO) | Windows 应用自动化 |

微软出品，专注 Windows 应用的 AI 自动化。用视觉理解 + UI 自动化操控桌面应用。

### Open Computer Use

| 出品方 | 定位 |
|--------|------|
| [E2B](https://github.com/e2b-dev/open-computer-use) | 开源 LLM 驱动的桌面控制 |

基于 E2B 的 Desktop Sandbox，用开源 LLM 实现 computer use。

## 六、数据采集：Agent 的信息来源

Agent 要做事，首先得能获取信息。

### Firecrawl

| Stars | 定位 |
|-------|------|
| [112.5k](https://github.com/firecrawl/firecrawl) | AI 友好的 Web 数据采集 |

把网站转成 LLM 可用的结构化数据。支持 JS 渲染、分页、多步导航。提供 `/agent` 端点，用自然语言描述你要什么数据，它自动搜索和提取。

### Crawl4AI

| 定位 |
|------|
| [开源 LLM 友好爬虫](https://github.com/unclecode/crawl4ai) |

异步 Web 爬虫，专为 LLM 优化输出格式。轻量、快速，适合给 Agent 喂数据。

## 七、协议与基础设施：Agent 的连接层

### MCP（Model Context Protocol）

| 出品方 | 定位 |
|--------|------|
| [Anthropic](https://github.com/modelcontextprotocol) | Agent 工具连接标准 |

2024 年 11 月 Anthropic 开源的协议，定义了 LLM 连接外部工具和数据源的标准方式。到 2026 年 4 月已有 2300+ 个 MCP server，Claude、Cursor、VS Code 等 200+ 工具原生支持。

**一句话理解**：以前每个 Agent 平台要单独写集成，MCP 让你定义一次工具，所有兼容的 Agent 都能调用。

### A2A（Agent-to-Agent Protocol）

| 出品方 | 定位 |
|--------|------|
| Google | Agent 间通信标准 |

Google 提出的 Agent 间通信协议。如果说 MCP 解决的是"Agent 怎么调工具"，A2A 解决的是"Agent 之间怎么对话"。

## 八、怎么选？

根据你的需求：

- **"我想快速搭一个能上网的 Agent"** → Browser Use + LangGraph 或 CrewAI
- **"我不想写代码"** → Dify
- **"我是 TypeScript 开发者"** → Mastra + Stagehand
- **"我想让 AI 帮我写代码"** → OpenHands
- **"我想让 AI 操控整个桌面"** → CUA 或 UFO
- **"我需要给 Agent 接入各种工具"** → 先看 MCP 生态有没有现成的 server
- **"我在 Google Cloud 上"** → Google ADK
- **"我在微软生态里"** → Microsoft Agent Framework（AutoGen 继任者）

## 九、总结

2026 年的 AI Agent 开源生态已经相当成熟：

- **框架层**有 LangGraph、CrewAI、Dify 等十余个选择，Python 占绝对主导
- **浏览器自动化**以 Browser Use 为代表，正在从"辅助工具"变成"自主 Agent"
- **编码 Agent** OpenHands 已有 70k+ stars，正在改变软件开发方式
- **协议层** MCP 正在成为 Agent 工具连接的事实标准

这个领域变化极快。建议挑一两个方向深入，而不是试图覆盖所有项目。框架会变，但"LLM + 工具调用 + 多步推理"这个基本范式短期内不会变。

---

*Star 数据截至 2026 年 4 月，实际数字请以 GitHub 为准。*

*参考资料：*
- [Top 10 Most Starred AI Agent Frameworks on GitHub (2026)](https://blog.agentailor.com/blog/top-ai-agent-frameworks-github-2026)
- [The Best Open Source Frameworks For Building AI Agents in 2026 - Firecrawl](https://www.firecrawl.dev/blog/best-open-source-agent-frameworks)
- [Top AI GitHub Repositories in 2026 - ByteByteGo](https://blog.bytebytego.com/p/top-ai-github-repositories-in-2026)
