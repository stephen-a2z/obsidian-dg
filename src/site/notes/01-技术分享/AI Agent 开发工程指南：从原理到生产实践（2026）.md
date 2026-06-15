---
{"dg-publish":true,"permalink":"/01-技术分享/AI Agent 开发工程指南：从原理到生产实践（2026）/","tags":["AI-agent","工程指南","LLM","架构设计","MCP","LangGraph","生产实践"],"noteIcon":"","created":"2026-06-15T14:56:21.271+08:00","updated":"2026-06-15T14:56:21.271+08:00"}
---


# AI Agent 开发工程指南：从原理到生产实践（2026）

> 这是一篇由表及里、自顶向下的 Agent 开发工程指南。从"Agent 是什么"到"怎么在生产环境跑起来"，覆盖定义、设计哲学、技术原理、架构模式、框架选型、协议生态、开发方法论和实战案例。所有涉及的技术栈均为 2025-2026 年最前沿。

---

## 一、什么是 AI Agent

### 1.1 定义

AI Agent 不是聊天机器人。聊天机器人只回答问题，Agent 能**自主规划、调用工具、执行多步任务**。

一个最小的 Agent 包含三个部分：

```
LLM（大脑）    → 决定做什么
工具（手脚）   → 搜索、调 API、读写文件、操作浏览器……
循环（骨架）   → 观察结果 → 思考 → 行动 → 再观察……直到任务完成
```

用更正式的语言说：Agent 是一个以大语言模型为决策内核，通过感知环境、推理规划、调用工具来自主完成目标的系统。

### 1.2 Agent vs 传统 LLM 应用

| 维度 | 传统 LLM 应用 | AI Agent |
|------|--------------|----------|
| 交互模式 | 一问一答 | 多轮自主循环 |
| 工具使用 | 无或固定管道 | 动态选择和调用 |
| 规划能力 | 无 | 分解任务、制定计划 |
| 错误处理 | 返回错误 | 自我诊断、重试、换路径 |
| 状态管理 | 无状态 | 有状态，跨轮次记忆 |
| 典型复杂度 | 1 步 | 5-50 步 |

### 1.3 Agent 的能力光谱

从简单到复杂，Agent 可以被放在一个能力光谱上：

```
Prompt → RAG → Function Calling → ReAct Agent → Multi-Agent → Autonomous Agent
  (0步)  (1步)     (1-2步)         (3-10步)      (10-50步)      (无限)
```

越往右，自主性越高，可控性越低，工程复杂度指数增长。**不要追求最右端**——选择与业务需求匹配的复杂度。

---

## 二、设计哲学

### 2.1 The Bitter Lesson：不要包装 LLM

Browser Harness 团队（Browser Use）的核心洞察：

> **不要包装 LLM，也不要包装它的工具。**

你写的每一个封装层，都是你替模型做的决定。模型必须在你画好的框框里工作。如果你的封装没覆盖某个场景，模型就卡住了。

LLM 在训练时已经见过海量 API 文档、协议规范、代码示例。直接暴露原始能力（如 CDP 协议）给 LLM，它能做到的事情远超你预设的 API。

**实践启示**：
- 给 Agent 最大的行动自由度，用 guardrails 约束边界而非限制手段
- 工具描述写清楚"能做什么"，而不是限制"怎么做"
- 当 Agent 反复卡在某个场景时，先检查是不是你的抽象层在挡路

### 2.2 Spec 驱动开发：AI 不会"补脑"

AI 时代软件开发的重心从编码转向了需求定义：

```
传统：需求 10% → 设计 15% → 编码 50% → 测试 15% → 部署 10%
AI 时代：需求 15% → 设计与 Spec 30% → 编码 20% → 测试 25% → 部署 10%
```

AI 写代码很快，但不知道该写什么。Spec 的质量直接决定 AI 产出的质量。这一原则同样适用于 Agent 开发——Agent 的 instructions 就是它的 Spec。

### 2.3 测试先行：后写的测试什么也证明不了

在 Agent 开发中，TDD 被执行得更加严格：

> 先写了代码再补测试？**删掉代码，重来。**

AI 看到已有实现，就会让测试去适配实现。先写测试定义"Agent 该做什么"，再实现让测试通过。

### 2.4 最小可用原则

不要一上来就选最复杂的框架。Agent 开发最大的坑不是"框架不够强"，而是"LLM 的推理能力不够稳定"。

**正确路径**：
1. 先用最简单方式验证 LLM 能不能完成你的任务
2. 再决定架构复杂度
3. 在已验证的基础上逐步增加能力

### 2.5 可观测性优先

你不可能调试看不见的东西。Agent 的每一步决策、每一次工具调用、每一个中间状态都要可追踪。**没有可观测性的 Agent 系统不能上生产。**

---

## 三、技术原理

### 3.1 核心循环：ReAct

Agent 的基础运行机制是 ReAct（Reasoning + Acting）循环：

```
┌─────────────────────────────────────────┐
│                                         │
│  Observe → Think → Act → Observe → ... │
│     ▲                         │         │
│     └─────────────────────────┘         │
│         (直到任务完成)                    │
└─────────────────────────────────────────┘
```

最小实现只需 30 行代码：

```python
from openai import OpenAI
import json

client = OpenAI()
tools = [{"type": "function", "function": {
    "name": "search", "description": "搜索信息",
    "parameters": {"type": "object", "properties": {"query": {"type": "string"}}}
}}]

def run_agent(task: str):
    messages = [{"role": "user", "content": task}]
    while True:
        resp = client.chat.completions.create(model="gpt-4o", messages=messages, tools=tools)
        msg = resp.choices[0].message
        messages.append(msg)
        if not msg.tool_calls:
            return msg.content  # 任务完成
        for call in msg.tool_calls:
            result = execute_tool(call.function.name, json.loads(call.function.arguments))
            messages.append({"role": "tool", "tool_call_id": call.id, "content": result})
```

这就是一个完整的 Agent——有工具调用、有循环、LLM 自己决定什么时候停。

### 3.2 七种架构模式

#### 模式一：ReAct（推理+行动）

LLM 交替进行推理和工具调用。简单、灵活，适合 1-5 步任务。容易陷入循环。

#### 模式二：Plan-and-Execute（计划+执行）

将规划与执行分离。Planner 分解任务，Executor 逐步执行，可在每步后重新规划。

```
┌──────────┐     ┌──────────────┐     ┌──────────┐
│ Planner  │────►│  Step Queue  │────►│ Executor │
│  (LLM)  │     │              │     │ (Agent)  │
└──────────┘     └──────────────┘     └─────┬────┘
      ▲                                      │
      └──────────── Replan ──────────────────┘
```

适合 5 步以上的复杂任务、研究任务。

#### 模式三：Reflection（反思/自我批评）

Agent 评估自己的输出并迭代改进：

```python
draft = generator.invoke("写一份分析报告...")
critique = critic.invoke(f"评估这份草稿，列出改进点：\n{draft}")
if "满意" not in critique:
    final = generator.invoke(f"根据反馈修改：\n{critique}\n\n原稿：\n{draft}")
```

适合写作、代码审查、方案优化。

#### 模式四：Tool-Augmented Generation

LLM + 实时工具访问。核心是工具描述的质量——LLM 基于描述选择工具。

**最佳实践**：
- 工具数量控制在 7-15 个（超过用工具检索）
- 描述写清楚用途和参数含义
- 校验工具输入，优雅处理错误

#### 模式五：Supervisor + Worker

最常见的多 Agent 模式。一个 Supervisor 路由任务给专业 Worker。

```
           ┌────────────┐
           │ Supervisor │
           └─────┬──────┘
      ┌──────────┼──────────┐
┌─────▼────┐ ┌───▼───┐ ┌───▼─────┐
│Researcher│ │ Coder │ │ Writer  │
└──────────┘ └───────┘ └─────────┘
```

简单易理解，适合 5-10 个 Worker。

#### 模式六：Hierarchical（层级式）

多层管理结构，适合复杂组织式任务。Director → Managers → Workers。

#### 模式七：Swarm（群体智能）

去中心化，Agent 之间点对点交接，没有中央协调者。上下文随对话流转。适合客服路由、多部门分流。OpenAI Agents SDK 就是基于这个模式。

### 3.3 记忆架构

Agent 的记忆分四层：

| 层级 | 作用 | 实现方式 |
|------|------|---------|
| 短期记忆 | 当前对话上下文 | 滑动窗口 / Token 预算 / 摘要压缩 |
| 长期记忆 | 持久化的知识和事实 | 向量数据库 + 知识图谱 |
| 情景记忆 | 过去任务执行的完整经验 | 结构化存储（任务/步骤/结果/教训） |
| 共享状态 | 多 Agent 间的协作数据 | LangGraph State / Redis / 事件总线 |

**2025 最佳实践——混合检索**：

```
查询 → 向量搜索（语义相关） ──┐
                              ├── 合并排序 → 注入 Context
查询 → 图谱遍历（关系相关） ──┘
```

### 3.4 Function Calling 原理

LLM 的工具调用本质是**结构化输出**——模型被训练为在合适时机输出特定 JSON 格式，表示"我要调用这个工具"。

```json
{
  "tool_calls": [{
    "id": "call_abc123",
    "function": {
      "name": "search_database",
      "arguments": "{\"query\": \"2026 AI trends\", \"limit\": 5}"
    }
  }]
}
```

开发者的工作：
1. 定义工具的 JSON Schema（名称、描述、参数）
2. 把 schema 放入 system prompt 或 tools 参数
3. 解析 LLM 输出的 tool_calls
4. 执行工具，把结果以 tool role 返回给 LLM
5. LLM 根据结果继续推理或返回最终答案

---

## 四、协议生态

### 4.1 MCP（Model Context Protocol）——Agent 连接工具的标准

Anthropic 于 2024 年 11 月开源，到 2026 年已成为 Agent 工具连接的事实标准。2300+ 个 MCP Server，Claude/Cursor/VS Code 等 200+ 工具原生支持。

**一句话理解**：以前每个 Agent 平台要单独写集成，MCP 让你定义一次工具，所有兼容的 Agent 都能调用。

**架构**：

```
┌──────────┐      ┌──────────┐      ┌──────────┐
│   Host   │      │  Client  │      │  Server  │
│(LLM App) │─────►│(协议处理) │─────►│(工具提供) │
└──────────┘      └──────────┘      └──────────┘
```

**三种原语（Primitive）**：

| 原语 | 方向 | 说明 |
|------|------|------|
| **Tools** | Server→Client（模型控制） | LLM 可调用的函数，JSON Schema 参数 |
| **Resources** | Server→Client（应用控制） | 应用可读取的数据/内容（文件、DB、API） |
| **Prompts** | Server→Client（用户控制） | 暴露给用户的提示词模板/工作流 |

**三种传输方式**：

| 传输 | 场景 | 特点 |
|------|------|------|
| stdio | 本地进程 | 子进程 stdin/stdout，最简单 |
| SSE | 远程服务 | HTTP Server-Sent Events，原始方案 |
| Streamable HTTP | 远程服务（推荐） | 单端点，支持请求-响应和流式，2025 新增 |

**Server 实现示例**：

```python
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("my-tools")

@server.list_tools()
async def list_tools():
    return [Tool(
        name="search_docs",
        description="搜索技术文档",
        inputSchema={"type": "object", "properties": {"query": {"type": "string"}}}
    )]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "search_docs":
        results = await search(arguments["query"])
        return [TextContent(type="text", text=results)]
```

### 4.2 A2A（Agent-to-Agent Protocol）——Agent 间通信标准

Google 于 2025 年 4 月提出。MCP 解决"Agent 怎么调工具"，A2A 解决"Agent 之间怎么对话"。

**核心概念**：

- **Agent Card**：JSON 元数据（`.well-known/agent.json`），声明 Agent 的能力、技能、认证方式
- **Task 生命周期**：`submitted → working → [input-required] → completed/failed/canceled`
- **消息交换**：JSON-RPC，任务包含消息，消息包含 parts（文本/文件/数据）

```json
{
  "name": "Research Agent",
  "url": "https://research-agent.example.com",
  "capabilities": {"streaming": true, "pushNotifications": true},
  "skills": [{"id": "web-research", "name": "Web Research"}]
}
```

### 4.3 MCP + A2A 协同

```
┌──────────────────────────────────────────────┐
│              Agent 系统                        │
│                                              │
│  Agent A ◄──── A2A ────► Agent B ◄── A2A ──► Agent C  │
│    │                       │                   │       │
│    │ MCP                   │ MCP               │ MCP   │
│    ▼                       ▼                   ▼       │
│  DB Tool               API Tool             FS Tool   │
└──────────────────────────────────────────────┘
```

- **MCP** = 纵向（Agent ↔ 工具）
- **A2A** = 横向（Agent ↔ Agent）

---

## 五、框架选型

### 5.1 主流框架对比（2026）

| 框架 | Stars | 语言 | 核心特点 | 适合场景 |
|------|-------|------|---------|---------|
| **LangGraph** | 24.8k | Python | 状态图编排、企业级、持久化 | 复杂工作流、金融/医疗 |
| **CrewAI** | 44.3k | Python | 角色扮演、API 简洁 | 快速多 Agent 协作 |
| **OpenAI Agents SDK** | 21k | Python | 轻量、Handoff、100+ LLM | 通用原型、Swarm 模式 |
| **Google ADK** | 17.8k | Python | Gemini 集成、层级组合 | Google 生态 |
| **Mastra** | 21.2k | TypeScript | 持久化工作流、Zod Schema | JS/TS 团队 |
| **Agno** | 36.4k | Python | 高性能运行时、多模态 | 大规模部署 |
| **Dify** | 129.8k | Python | 低代码可视化 | 非技术团队、快速验证 |

### 5.2 怎么选

```
你的需求是什么？
│
├── "不想写代码" → Dify / Langflow
│
├── "快速原型，几十行代码" → OpenAI Agents SDK / Google ADK
│
├── "多 Agent 协作，1-2 小时搭起来" → CrewAI
│
├── "企业级，需要状态管理/人工审批/持久化" → LangGraph
│
├── "TypeScript 生态" → Mastra
│
└── "大规模部署，性能敏感" → Agno
```

### 5.3 各框架核心代码示例

**LangGraph——状态图式**：

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresSaver
from typing import TypedDict, Annotated
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
    plan: list[str]

def planner(state: State) -> dict:
    # LLM 制定计划
    plan = llm.invoke("制定计划...", state["messages"])
    return {"plan": plan.steps}

def executor(state: State) -> dict:
    # 执行计划中的下一步
    step = state["plan"][0]
    result = execute_step(step)
    return {"messages": [result], "plan": state["plan"][1:]}

def should_continue(state: State) -> str:
    return "executor" if state["plan"] else END

graph = StateGraph(State)
graph.add_node("planner", planner)
graph.add_node("executor", executor)
graph.add_edge(START, "planner")
graph.add_conditional_edges("planner", should_continue)
graph.add_conditional_edges("executor", should_continue)

# 持久化 + 人工审批
checkpointer = PostgresSaver.from_conn_string("postgresql://...")
app = graph.compile(checkpointer=checkpointer, interrupt_before=["executor"])
```

**CrewAI——角色扮演式**：

```python
from crewai import Agent, Task, Crew
from crewai.flow.flow import Flow, listen, start

researcher = Agent(role="研究员", goal="深入研究给定主题", backstory="资深研究员")
writer = Agent(role="写手", goal="写成通俗文章", backstory="技术博客作者")

class ContentFlow(Flow):
    @start()
    def research(self):
        task = Task(description=f"研究 {self.state.topic}", agent=researcher)
        crew = Crew(agents=[researcher], tasks=[task])
        return crew.kickoff()

    @listen(research)
    def write(self, research_result):
        task = Task(description=f"基于研究写文章：{research_result}", agent=writer)
        crew = Crew(agents=[writer], tasks=[task])
        return crew.kickoff()
```

**OpenAI Agents SDK——Swarm 式**：

```python
from agents import Agent, Runner, handoff, function_tool

@function_tool
def lookup_order(order_id: str) -> str:
    """查询订单状态"""
    return f"订单 {order_id} 已发货"

billing_agent = Agent(name="账单专员", tools=[lookup_order])
technical_agent = Agent(name="技术支持", tools=[...])

triage_agent = Agent(
    name="客服分流",
    instructions="根据用户问题路由到合适的专员。",
    handoffs=[billing_agent, technical_agent],
)

result = await Runner.run(triage_agent, input="我的订单 12345 到哪了？")
```

---

## 六、生产工程

### 6.1 可观测性

Agent 系统必须能回答三个问题：
1. Agent 做了什么决策？（Tracing）
2. 花了多少钱/时间？（Metrics）
3. 输出质量如何？（Evaluation）

**Trace 结构（OpenTelemetry 标准）**：

```
Trace: "user_query_123"
├── Span: "agent_run" (总延迟: 4.2s)
│   ├── Span: "llm_call_1" (模型: gpt-4o, tokens: 1200, 延迟: 1.1s)
│   ├── Span: "tool_call: search_db" (延迟: 0.3s)
│   ├── Span: "llm_call_2" (模型: gpt-4o, tokens: 800, 延迟: 0.9s)
│   └── Span: "guardrail: pii_check" (通过: true, 延迟: 0.05s)
```

**工具选择**：
- **LangSmith**：全链路追踪 + 评估 + 数据集管理（LangGraph 生态首选）
- **Phoenix (Arize)**：开源、OpenTelemetry 原生、嵌入漂移检测
- **Braintrust**：评估导向、A/B 测试 prompt 和模型

### 6.2 护栏（Guardrails）

**输入护栏**：
- Prompt 注入检测（分类器）
- 话题边界限制
- PII 检测与脱敏
- 用户级限流

**输出护栏**：
- 结构化输出校验（JSON Schema / Pydantic）
- 事实性检查（对照检索源）
- 安全/毒性分类器
- 品牌调性合规

```python
from agents import Agent, InputGuardrail, OutputGuardrail, GuardrailFunctionOutput

@input_guardrail
async def injection_check(ctx, agent, input):
    result = await Runner.run(injection_detector, input)
    if result.final_output.is_injection:
        return GuardrailFunctionOutput(tripwire_triggered=True)

agent = Agent(
    name="assistant",
    input_guardrails=[injection_check],
    output_guardrails=[pii_filter, schema_validator],
)
```

### 6.3 Human-in-the-Loop

四种模式：

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| 审批门 | Agent 暂停，等人批准高风险操作 | 金融交易、数据删除 |
| 升级 | Agent 信心不足时交给人类 | 客服、医疗问诊 |
| 审核队列 | Agent 起草，人审批/编辑 | 内容发布、合同 |
| 反馈循环 | 人类评分改善未来表现 | 持续优化 |

**LangGraph 实现**：

```python
# 编译时声明中断点
app = graph.compile(
    checkpointer=checkpointer,
    interrupt_before=["execute_trade"]  # 在此节点前暂停
)

# 人工审批后恢复
app.invoke(Command(resume={"approved": True}), config=thread_config)
```

### 6.4 成本优化

| 策略 | 效果 | 实现 |
|------|------|------|
| Prompt 缓存 | 缓存命中减 90% | Anthropic/OpenAI 自动缓存相同前缀 |
| 模型路由 | 简单任务用便宜模型 | 复杂度分类 → 选模型 |
| Token 预算 | 控制上下文长度 | 智能截断、摘要压缩 |
| 工具结果缓存 | 避免重复调用 | 确定性工具输出缓存（文档查询等） |
| Batch API | 非实时任务减 50% | OpenAI/Anthropic 批量端点 |

```python
def select_model(complexity: str):
    match complexity:
        case "simple": return "gpt-4o-mini"     # $0.15/1M input
        case "medium": return "gpt-4o"          # $2.50/1M input
        case "complex": return "o1"             # $15/1M input
```

### 6.5 错误恢复

- **工具失败**：指数退避重试（默认 3 次）
- **LLM 解析错误**：带错误上下文重新提示
- **死循环**：最大迭代次数 + 优雅退出
- **崩溃恢复**：从最近 checkpoint 恢复
- **模型降级**：限流/宕机时切备用模型

```python
def tool_node(state):
    try:
        result = execute_tool(state["pending_call"])
    except ToolError as e:
        if state.get("retry_count", 0) < 3:
            return {"retry_count": state["retry_count"] + 1, "error": str(e)}
        return {"messages": [ToolMessage(f"工具失败: {e}。请换一种方式。")]}
```

---

## 七、开发方法论

### 7.1 Agent 开发的 TDD

```
RED（写失败测试）→ 验证失败 → GREEN（最小实现）→ 验证通过 → REFACTOR → 下一轮
```

在 Agent 语境下：
- **测试定义 Agent 的预期行为**，不是验证它的实现细节
- 测试 Agent 的输出（完成了任务吗？）而不是过程（调用了哪个工具？）
- 先写集成测试（端到端），再补单元测试（工具函数）

### 7.2 Spec-Driven Development（规格驱动）

```
需求描述 → Spec 文档 → Agent instructions/tools 定义 → 实现 → 验证
```

Spec 内容：
- Agent 的角色和目标
- 可用工具列表及每个工具的输入输出
- 决策边界（什么情况做什么）
- 失败处理策略
- 输出格式要求

### 7.3 Agentic Coding 开发流

让编码 Agent（Claude Code / Codex / OpenHands）帮你开发 Agent 系统：

```
┌─────────────────────────────────────────────────────────┐
│              Coding Agent 工作循环                        │
│                                                         │
│  1. 读取任务/Issue                                       │
│  2. 探索代码库（搜索、读文件、理解依赖）                    │
│  3. 制定修改计划（改哪些文件、什么方案）                    │
│  4. 实现（写/编辑文件）                                   │
│  5. 验证（跑测试、lint、类型检查）                        │
│  6. 测试失败 → 诊断 → 修复 → 再验证                      │
│  7. 提交结果（diff、总结）                               │
└─────────────────────────────────────────────────────────┘
```

**最佳实践**：
1. 提供清晰的 Spec/设计文档
2. 拆分大任务为聚焦的小任务
3. 依赖测试套件验证正确性
4. Review diff 而不仅仅看结果
5. 使用 AGENTS.md / CLAUDE.md 定义项目规范
6. 始终在分支上工作，review 后合并

---

## 八、实战案例

### 案例一：智能客服 Agent（Swarm 模式）

**需求**：电商客服，自动分流、查询订单、处理退款。

**架构选择**：OpenAI Agents SDK（Swarm/Handoff 模式）

```python
from agents import Agent, Runner, function_tool, handoff

# 工具定义
@function_tool
def query_order(order_id: str) -> str:
    """查询订单状态和物流信息"""
    return db.get_order(order_id).to_json()

@function_tool
def process_refund(order_id: str, reason: str) -> str:
    """处理退款申请（金额<500自动，>=500需人工）"""
    order = db.get_order(order_id)
    if order.amount >= 500:
        return "NEEDS_HUMAN_APPROVAL: 退款金额>=500，需人工审批"
    return refund_service.process(order_id, reason)

# Agent 定义
order_agent = Agent(
    name="订单专员",
    instructions="处理订单查询和物流追踪。礼貌、简洁。",
    tools=[query_order],
)

refund_agent = Agent(
    name="退款专员",
    instructions="处理退款请求。确认原因后执行退款。",
    tools=[process_refund],
)

triage_agent = Agent(
    name="客服分流",
    instructions="根据用户意图路由到合适的专员。订单相关→订单专员，退款→退款专员。",
    handoffs=[order_agent, refund_agent],
)

# 运行
result = await Runner.run(triage_agent, input="我买的东西想退货，订单号 ORD-789")
```

### 案例二：研究助手（Plan-and-Execute + Reflection）

**需求**：给定一个主题，自动调研、整理、输出结构化报告。

**架构选择**：LangGraph（Plan-Execute + 反思循环）

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class ResearchState(TypedDict):
    topic: str
    plan: list[str]
    findings: list[str]
    draft: str
    critique: str
    final_report: str

def create_plan(state):
    """Planner：分解研究主题为子问题"""
    plan = llm.invoke(f"将以下主题分解为 3-5 个研究子问题：\n{state['topic']}")
    return {"plan": plan.steps}

def research_step(state):
    """Executor：搜索并总结当前子问题"""
    current = state["plan"][0]
    results = search_tool.invoke(current)
    summary = llm.invoke(f"总结以下搜索结果：\n{results}")
    return {"findings": state["findings"] + [summary], "plan": state["plan"][1:]}

def write_draft(state):
    """Writer：将发现整合成报告"""
    draft = llm.invoke(f"基于以下发现写一份报告：\n{state['findings']}")
    return {"draft": draft}

def critique_draft(state):
    """Critic：评估报告质量"""
    critique = llm.invoke(f"评估报告完整性和准确性，给出改进建议：\n{state['draft']}")
    return {"critique": critique}

def should_revise(state):
    if "满意" in state["critique"] or "无需修改" in state["critique"]:
        return "finalize"
    return "revise"

def revise(state):
    revised = llm.invoke(f"根据反馈修改：\n{state['critique']}\n\n原稿：\n{state['draft']}")
    return {"draft": revised}

def finalize(state):
    return {"final_report": state["draft"]}

# 构建图
graph = StateGraph(ResearchState)
graph.add_node("plan", create_plan)
graph.add_node("research", research_step)
graph.add_node("write", write_draft)
graph.add_node("critique", critique_draft)
graph.add_node("revise", revise)
graph.add_node("finalize", finalize)

graph.add_edge(START, "plan")
graph.add_conditional_edges("plan", lambda s: "research" if s["plan"] else "write")
graph.add_conditional_edges("research", lambda s: "research" if s["plan"] else "write")
graph.add_edge("write", "critique")
graph.add_conditional_edges("critique", should_revise)
graph.add_edge("revise", "critique")  # 反思循环
graph.add_edge("finalize", END)

app = graph.compile()
```

### 案例三：浏览器自动化 Agent（全自主模式）

**需求**：自动完成网页操作任务（填表、下单、数据采集）。

**架构选择**：Browser Use + LangGraph

```python
from browser_use import Agent
from langchain_openai import ChatOpenAI

agent = Agent(
    task="在 GitHub 上搜索 'AI Agent framework'，收集前 5 个结果的名称、star 数和描述",
    llm=ChatOpenAI(model="gpt-4o"),
)

result = await agent.run()
print(result.final_result())
```

**进阶——Browser Harness 模式**（Agent 自己写工具）：

```python
# Agent 发现 helpers.py 中没有 upload_file()
# 它自己编辑 helpers.py 添加：
def upload_file(selector, file_path):
    """上传文件到指定的 input[type=file] 元素"""
    cdp("DOM.querySelector", nodeId=document_node, selector=selector)
    cdp("DOM.setFileInputFiles", files=[file_path], nodeId=result_node)

# 然后直接调用自己写的函数
upload_file("#avatar-input", "/tmp/photo.jpg")
```

---

## 九、评估体系

### 9.1 评估维度

| 维度 | 问题 | 方法 |
|------|------|------|
| 完成率 | Agent 完成任务了吗？ | 自动化判定（测试通过/目标达成） |
| 正确性 | 输出是对的吗？ | 单元测试 + LLM-as-judge |
| 效率 | 花了多少步/tokens/钱？ | 指标追踪 |
| 鲁棒性 | 能处理边界情况吗？ | 对抗性输入测试 |
| 安全性 | 是否越界？ | 红队测试 + 护栏验证 |

### 9.2 可靠性公式

```
Agent 可靠性 = P(正确最终答案) × P(无有害副作用) × P(成本可控)
```

### 9.3 评估实践

```python
# 统计评估：每个任务跑 N 次（N=5-10）衡量一致性
results = [agent.run(task) for _ in range(N)]
success_rate = sum(1 for r in results if r.passed) / N

# LLM-as-judge 评估
eval_prompt = """
评估 Agent 的输出：
1. 正确性 (1-5)：是否解决了问题？
2. 完整性 (1-5)：是否覆盖所有要求？
3. 效率 (1-5)：路径是否合理？

任务: {task}
输出: {output}
预期: {expected}
"""
```

### 9.4 主要 Benchmark

- **SWE-bench Verified**：真实 GitHub Issue 修复，当前 SOTA 约 50-60%
- **WebArena**：真实网页任务（导航、填表、数据提取）
- **AgentBench**：多环境评估（OS、DB、Web、游戏），8 种场景

---

## 十、决策树：从 0 到 1 构建 Agent

```
第 1 步：验证可行性
├── LLM 能完成这个任务的单步吗？
├── 用纯 API + 手写循环跑通一个 demo
└── 如果单步都不行 → 任务本身可能不适合 Agent

第 2 步：选择架构
├── 单步工具调用够了 → Function Calling，无需框架
├── 3-5 步线性流程 → ReAct + 简单循环
├── 5+ 步，需要规划 → Plan-and-Execute
├── 需要多个专业角色 → Multi-Agent（CrewAI 或 Supervisor）
└── 需要状态持久化/人工审批 → LangGraph

第 3 步：选择框架
├── 快速原型 → OpenAI Agents SDK / CrewAI
├── 生产级 → LangGraph + LangSmith
├── TypeScript → Mastra
└── 不写代码 → Dify

第 4 步：工具设计
├── 先查 MCP 生态有没有现成 Server
├── 工具数量 7-15 个为宜
├── 描述清晰、参数明确、错误处理完善
└── 高风险工具加审批门

第 5 步：补齐生产能力
├── 可观测性（Tracing + Metrics）
├── 护栏（Input + Output）
├── 错误恢复（重试 + 降级 + Checkpoint）
├── 成本控制（缓存 + 路由 + 预算）
└── 评估体系（自动 + 人工）
```

---

## 十一、2026 年 Agent 生态全景

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Production Agent System                           │
│                                                                     │
│  ┌─────────┐     ┌──────────────────────────────────────────────┐  │
│  │  用户   │     │            编排层                              │  │
│  └────┬────┘     │   LangGraph / CrewAI / Agents SDK / ADK      │  │
│       │          │                                              │  │
│       ▼          │  ┌────────┐  ┌──────┐  ┌───────────────┐    │  │
│  ┌─────────┐     │  │ Agents │  │Router│  │  Guardrails   │    │  │
│  │输入护栏 │────►│  │        │  │      │  │ (Input/Output)│    │  │
│  └─────────┘     │  └───┬────┘  └──────┘  └───────────────┘    │  │
│                  └──────┼───────────────────────────────────────┘  │
│                         │                                          │
│  ┌──────────────────────┼───────────────────────────────────────┐  │
│  │            工具与记忆层                                        │  │
│  │                      │                                        │  │
│  │  ┌──────┐  ┌───────┐│  ┌──────────┐  ┌──────────────┐       │  │
│  │  │ MCP  │  │Vector ││  │Knowledge │  │   Episodic   │       │  │
│  │  │Servers│ │Store  ││  │  Graph   │  │   Memory     │       │  │
│  │  └──────┘  └───────┘│  └──────────┘  └──────────────┘       │  │
│  └──────────────────────┼───────────────────────────────────────┘  │
│                         │                                          │
│  ┌──────────────────────┼───────────────────────────────────────┐  │
│  │         可观测性与评估                                         │  │
│  │  ┌─────────┐  ┌──────────┐  ┌────────────────────────┐      │  │
│  │  │ Tracing │  │ Metrics  │  │  Evals (LLM-judge,     │      │  │
│  │  │(OTel)   │  │(cost,lat)│  │   test suites)         │      │  │
│  │  └─────────┘  └──────────┘  └────────────────────────┘      │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 十二、总结与建议

### 核心认知

1. **Agent = LLM + 工具 + 循环**。三者缺一不可，工程的核心是让这个循环稳定运转。
2. **复杂度是敌人**。从最简单的能工作的方案开始，按需增加能力。
3. **Spec 决定上限**。Agent 的表现取决于你对任务的定义精度。
4. **可观测性是生命线**。看不见的 Agent 不能上生产。
5. **评估驱动迭代**。没有评估就没有改进方向。

### 技术选型速查

| 你的情况 | 推荐方案 |
|---------|---------|
| 验证想法，不写代码 | Dify + 低代码 |
| 快速原型，Python | OpenAI Agents SDK |
| 多 Agent 协作 | CrewAI（简单）/ LangGraph（复杂） |
| 企业级生产 | LangGraph + LangSmith + PostgreSQL |
| TypeScript 项目 | Mastra |
| Google 生态 | Google ADK + Vertex AI |
| 工具连接 | MCP Server 生态（2300+ 可用） |
| Agent 间通信 | A2A 协议 |

### 这个领域变化极快

框架会变，但 **"LLM + 工具调用 + 多步推理"** 这个基本范式短期内不会变。把精力投在理解原理而非绑定框架上。

---

*本文基于 2026 年 6 月的技术生态撰写，具体版本号和 Star 数请以实际为准。*

*参考资料：*
- [OpenAI - A Practical Guide to Building Agents](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf)
- [Anthropic - Building Effective Agents](https://docs.anthropic.com/en/docs/build-with-claude/agent)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [MCP Specification](https://spec.modelcontextprotocol.io/)
- [Google A2A Protocol](https://github.com/google/A2A)
- [The Bitter Lesson of Agent Harnesses](https://browser-use.com/posts/bitter-lesson-agent-harnesses)
- [CrewAI Documentation](https://docs.crewai.com/)
- [OpenAI Agents SDK](https://github.com/openai/openai-agents-python)
