---
{"dg-publish":true,"permalink":"/01-技术分享/快速开发 AI Agent 的五种方式（2026）/","tags":["AI-agent","入门教程","LLM","开发方式"],"noteIcon":"","created":"2026-04-28T12:23:43.403+08:00","updated":"2026-04-28T12:23:43.403+08:00"}
---


# 快速开发 AI Agent 的五种方式（2026）

> 从零代码拖拽到纯代码框架，从单 Agent 到多 Agent 协作——当前开发 AI Agent 的路径已经非常清晰。这篇文章按"上手速度"排序，列出五种主流方式，帮你根据自己的需求选对路。

## 先理解一个前提：Agent 到底是什么

在讨论"怎么开发"之前，先对齐概念。AI Agent 不是聊天机器人。聊天机器人只回答问题，Agent 能**自主规划、调用工具、执行多步任务**。

一个最小的 Agent 包含三个部分：

```
LLM（大脑）→ 决定做什么
工具（手脚）→ 搜索、调 API、读写文件、操作浏览器……
循环（骨架）→ 观察结果 → 思考 → 行动 → 再观察……直到任务完成
```

不同的开发方式，本质上是在不同层面帮你搭建这三个部分。

---

## 方式一：低代码平台（5 分钟出活）

**代表项目**：[Dify](https://github.com/langgenius/dify)（129k+ stars）、[Flowise](https://github.com/FlowiseAI/Flowise)、[Coze](https://www.coze.com)

### 怎么做

不写代码，在 Web 界面上拖拽节点搭建 Agent 工作流。

以 Dify 为例：

```bash
git clone https://github.com/langgenius/dify.git
cd dify/docker
cp .env.example .env
docker compose up -d
```

打开 `http://localhost/install`，在 UI 里：
1. 配置模型（OpenAI / Gemini / 本地 Ollama 都行）
2. 上传文档建知识库（自动做 RAG）
3. 创建 Agent 应用，配置工具和提示词
4. 发布，得到一个带聊天界面的应用 + API

### 适合谁

- 非技术人员或产品经理验证想法
- 内部工具、客服机器人、文档问答
- 不需要深度定制 Agent 行为的场景

### 局限

- 复杂的多步推理和条件分支不好表达
- 工具扩展受平台限制
- 想做精细控制（自定义记忆、动态工具选择）会碰壁

---

## 方式二：LLM 厂商的 Agent SDK（30 分钟出活）

**代表项目**：[OpenAI Agents SDK](https://github.com/openai/openai-agents-python)（21k stars）、[Google ADK](https://github.com/google/adk-python)（17.8k stars）

### 怎么做

用 LLM 厂商提供的官方 SDK，几十行代码搭一个 Agent。

OpenAI Agents SDK 示例：

```python
from agents import Agent, Runner, function_tool
import asyncio

@function_tool
def get_weather(city: str) -> str:
    """获取指定城市的天气"""
    return f"{city}今天晴，25°C"

agent = Agent(
    name="助手",
    instructions="你是一个有用的助手，可以查询天气。",
    tools=[get_weather],
)

result = asyncio.run(Runner.run(agent, input="北京今天天气怎么样？"))
print(result.final_output)
```

```bash
pip install openai-agents
OPENAI_API_KEY=sk-xxx python app.py
```

Google ADK 示例：

```python
from google.adk.agents import Agent

agent = Agent(
    model="gemini-2.0-flash",
    name="助手",
    instruction="你是一个有用的助手。",
)
```

### 适合谁

- 已经在用某家 LLM API 的开发者
- 想快速搭原型，不想引入重框架
- 简单的单 Agent + 工具调用场景

### 局限

- 跟特定厂商绑定（虽然 OpenAI SDK 声称支持 100+ 模型，但体验最好的还是自家模型）
- 多 Agent 协作能力有限
- 复杂工作流（状态管理、人工审批、长期记忆）需要自己补

---

## 方式三：多 Agent 框架（1-2 小时出活）

**代表项目**：[CrewAI](https://github.com/crewAIInc/crewAI)（44k stars）、[LangGraph](https://github.com/langchain-ai/langgraph)（24.8k stars）、[AutoGen](https://github.com/microsoft/autogen)（54.6k stars）

这是当前最主流的 Agent 开发方式。三个框架代表三种不同的思路：

### CrewAI：角色扮演式

给每个 Agent 定义角色和职责，它们按分工协作。

```python
from crewai import Agent, Task, Crew

researcher = Agent(
    role="研究员",
    goal="深入研究给定主题",
    backstory="你是一位资深研究员",
)

writer = Agent(
    role="写手",
    goal="把研究结果写成通俗易懂的文章",
    backstory="你是一位技术博客作者",
)

research_task = Task(
    description="研究 AI Agent 的最新发展趋势",
    agent=researcher,
    expected_output="一份研究报告",
)

write_task = Task(
    description="基于研究报告写一篇博客",
    agent=writer,
    expected_output="一篇博客文章",
)

crew = Crew(agents=[researcher, writer], tasks=[research_task, write_task])
result = crew.kickoff()
```

**特点**：API 最简洁，20 分钟能跑起来。适合任务分工明确的场景。

### LangGraph：状态图式

用有向图定义 Agent 的状态流转，每个节点是一个处理步骤。

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    query: str
    result: str

def research(state: State) -> State:
    return {"result": f"关于 {state['query']} 的研究结果..."}

def review(state: State) -> State:
    return {"result": f"审核通过：{state['result']}"}

graph = StateGraph(State)
graph.add_node("research", research)
graph.add_node("review", review)
graph.add_edge(START, "research")
graph.add_edge("research", "review")
graph.add_edge("review", END)

app = graph.compile()
result = app.invoke({"query": "AI Agent 趋势", "result": ""})
```

**特点**：控制力最强，支持条件分支、人工介入、持久化状态。企业级首选（Klarna、Uber 在用）。学习曲线比 CrewAI 陡。

### AutoGen：对话式

Agent 之间通过对话协作，像开会一样讨论问题。

**特点**：适合需要 Agent 之间"讨论"和"辩论"的场景。注意 AutoGen 已进入维护模式，微软推荐迁移到 [Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/)。

### 三者怎么选

| 框架 | 核心思路 | 上手难度 | 适合场景 |
|------|---------|---------|---------|
| CrewAI | 角色分工 | ⭐ | 任务流水线、内容生成 |
| LangGraph | 状态图 | ⭐⭐⭐ | 复杂工作流、企业应用 |
| AutoGen | 对话协作 | ⭐⭐ | 研究、头脑风暴（维护模式） |

---

## 方式四：纯 LLM API + 手写循环（2-4 小时）

**不用任何框架**，直接调 LLM API，自己写 Agent 循环。

### 怎么做

核心就是一个 while 循环 + function calling：

```python
from openai import OpenAI
import json

client = OpenAI()

tools = [{
    "type": "function",
    "function": {
        "name": "search",
        "description": "搜索信息",
        "parameters": {
            "type": "object",
            "properties": {"query": {"type": "string"}},
            "required": ["query"],
        },
    },
}]

def search(query: str) -> str:
    return f"搜索结果：关于 {query} 的最新信息..."

def run_agent(task: str):
    messages = [{"role": "user", "content": task}]

    while True:
        resp = client.chat.completions.create(
            model="gpt-4o", messages=messages, tools=tools,
        )
        msg = resp.choices[0].message
        messages.append(msg)

        if not msg.tool_calls:
            return msg.content

        for call in msg.tool_calls:
            result = search(**json.loads(call.function.arguments))
            messages.append({
                "role": "tool",
                "tool_call_id": call.id,
                "content": result,
            })

print(run_agent("帮我调研一下 2026 年 AI Agent 的发展趋势"))
```

这 30 行代码就是一个完整的 Agent——有工具调用、有循环、LLM 自己决定什么时候停。

### 适合谁

- 想彻底理解 Agent 原理的开发者
- 对框架的抽象不满意，想完全控制每一步
- 场景简单，不值得引入框架

### 局限

- 多 Agent 协作、状态持久化、错误恢复都要自己写
- 容易重复造轮子
- 生产环境需要补大量基础设施（日志、监控、重试）

---

## 方式五：让 Agent 自己写工具（实验性）

**代表项目**：[Browser Harness](https://github.com/browser-use/browser-harness)（7.2k stars）

这是最激进的方式——不预设工具集，让 Agent 在运行时自己编写需要的工具。

### 怎么做

Browser Harness 只有 592 行 Python，给 LLM 一根 CDP 管道和一个可编辑的 `helpers.py`。Agent 缺什么函数就自己写：

```
● Agent 需要上传文件
│
● helpers.py → upload_file() 不存在
│
● Agent 编辑 helpers.py，写入 upload_file()
│
✓ 文件上传成功
```

这依赖 Claude Code / Codex 等编码 Agent 的文件读写能力。Agent 不是在"调用工具"，而是在"创造工具"。

### 适合谁

- 搭配 Claude Code / Codex 等编码 Agent 使用
- 任务不可预测，无法提前定义工具集
- 愿意接受"让 AI 改自己代码"这种模式

### 局限

- 需要强力 LLM（Claude Sonnet / GPT-4o 级别）
- 不可控性高，不适合生产环境
- 目前主要用于浏览器自动化场景

---

## 总结对比

| 方式 | 代表 | 上手时间 | 代码量 | 可控性 | 适合阶段 |
|------|------|---------|--------|--------|---------|
| 低代码平台 | Dify、Flowise | 5 分钟 | 0 | ⭐ | 验证想法 |
| 厂商 SDK | OpenAI SDK、Google ADK | 30 分钟 | 几十行 | ⭐⭐ | 快速原型 |
| 多 Agent 框架 | CrewAI、LangGraph | 1-2 小时 | 百行级 | ⭐⭐⭐ | 正式开发 |
| 纯 API 手写 | OpenAI API + 循环 | 2-4 小时 | 百行级 | ⭐⭐⭐⭐⭐ | 深度定制 |
| Agent 自写工具 | Browser Harness | — | 极少 | ⭐ | 实验探索 |

### 我的建议

1. **先用 Dify 跑通想法**，确认需求是真实的
2. **用 CrewAI 或 OpenAI SDK 写原型**，验证 Agent 的推理和工具调用是否靠谱
3. **需要上生产时迁移到 LangGraph**，补上状态管理、监控、人工审批
4. 如果以上都不满足，再考虑手写或实验性方案

不要一上来就选最复杂的框架。Agent 开发最大的坑不是"框架不够强"，而是"LLM 的推理能力不够稳定"。先用最简单的方式验证 LLM 能不能完成你的任务，再决定架构。

---

*参考资料：*
- [OpenAI - A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
- [Anthropic - Claude Code Best Practices](https://www.anthropic.com/engineering/claude-code-best-practices)
- [The Best Open Source Frameworks For Building AI Agents in 2026](https://www.firecrawl.dev/blog/best-open-source-agent-frameworks)
