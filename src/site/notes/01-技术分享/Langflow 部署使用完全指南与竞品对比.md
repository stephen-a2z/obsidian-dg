---
{"dg-publish":true,"permalink":"/01-技术分享/Langflow 部署使用完全指南与竞品对比/","tags":["Langflow","AI-agent","LLM","低代码","workflow","开源项目","Dify","Flowise"],"noteIcon":"","created":"2026-05-09T18:18:55.108+08:00","updated":"2026-05-09T18:18:55.108+08:00"}
---


# Langflow 部署使用完全指南与竞品对比

> Langflow 是一个 148k star 的开源项目，提供可视化拖拽界面来构建 LLM 应用和 AI Agent。背后是 DataStax（已被 IBM 收购），最近刚发布 1.9 版本。这篇文章从部署、使用到竞品对比，帮你判断它是否适合你的场景。

## 一、Langflow 是什么

简单说：**一个基于 Python 的可视化 AI 应用构建器**。它不是框架，也不仅仅是工具，而是介于两者之间——你可以拖拽节点快速搭原型，也能直接改底层 Python 代码做深度定制。

三个核心定位：

1. **可视化开发**：拖拽组件连线，所见即所得
2. **源码可改**：每个组件都是 Python 类，支持自定义组件
3. **API/MCP 即部署**：画完流程，API 和 MCP Server 自动生成

技术栈：后端 FastAPI + LangChain 生态，前端 React + React Flow。

### 和 LangChain 的关系

Langflow 早期是 LangChain 的 GUI 封装，现在已经独立演进，但仍深度使用 LangChain 的组件体系。你可以把它理解成「带 UI 的 LangChain」——不过随着版本迭代，它也加入了自己的组件抽象。

## 二、部署方式：四种路径

### 方式 1：Langflow Desktop（最省心）

适合本地探索和学习。Windows 和 macOS 均有客户端，所有依赖已打包，不用折腾 Python 环境。

下载地址：<https://www.langflow.org/desktop>

**适合**：新手、本地 demo、不想碰终端的用户。

### 方式 2：pip / uv 本地安装（推荐给开发者）

需要 Python 3.10–3.13。官方推荐用 [uv](https://docs.astral.sh/uv/)（更快的 Python 包管理器）。

```bash
# 用 uv（推荐）
uv pip install langflow -U
uv run langflow run

# 或者用 pip
pip install langflow -U
python -m langflow run
```

启动后访问 `http://127.0.0.1:7860` 即可。

默认使用 SQLite，数据存在 `~/.langflow/` 目录下。

**坑位**：
- Python 3.13 还在兼容测试，生产建议 3.11 或 3.12
- Windows 用户遇到编译错误先装 Visual C++ Build Tools
- 首次启动会下载大量模型依赖，视网络情况几分钟到几十分钟不等

### 方式 3：Docker 单容器（最快验证）

```bash
docker run -p 7860:7860 langflowai/langflow:latest
```

30 秒起一个实例，适合快速体验。**但数据没持久化**，容器重启就丢。

### 方式 4：Docker Compose + PostgreSQL（生产推荐）

这是真正做生产部署的姿势。带持久化 PostgreSQL，数据不会丢。

```bash
git clone https://github.com/langflow-ai/langflow.git
cd langflow/docker_example
docker compose up -d
```

自定义配置建 `.env` 文件：

```bash
# 数据库凭据
POSTGRES_USER=langflow_user
POSTGRES_PASSWORD=your_strong_password
POSTGRES_DB=langflow

# Langflow 配置
LANGFLOW_DATABASE_URL=postgresql://langflow_user:your_strong_password@postgres:5432/langflow
LANGFLOW_CONFIG_DIR=/app/langflow
LANGFLOW_AUTO_LOGIN=False
LANGFLOW_SUPERUSER=admin
LANGFLOW_SUPERUSER_PASSWORD=admin_password
LANGFLOW_SECRET_KEY=your-random-secret-key
```

**关键环境变量**：

| 变量 | 作用 |
|------|------|
| `LANGFLOW_DATABASE_URL` | 数据库连接字符串 |
| `LANGFLOW_SECRET_KEY` | API Key 加密用，必须设置 |
| `LANGFLOW_AUTO_LOGIN` | 关闭才会强制登录 |
| `LANGFLOW_SUPERUSER` | 初始管理员账号 |
| `LANGFLOW_LOAD_FLOWS_PATH` | 预加载 JSON flow 目录 |
| `LANGFLOW_WORKERS` | Uvicorn worker 数量，生产建议 2-4 |

### 方式 5：Kubernetes（企业级）

官方提供 [Helm Charts](https://github.com/langflow-ai/langflow-helm-charts)：

```bash
helm repo add langflow https://langflow-ai.github.io/langflow-helm-charts
helm install langflow langflow/langflow-runtime \
  --set image.tag=1.9.2 \
  --set postgresql.enabled=true
```

生产部署的关键点：
- 前端后端分离部署（前端可扔 CDN）
- PostgreSQL 用云托管（RDS/Cloud SQL/Supabase）
- 加 Nginx/Caddy 做 SSL 终结
- 用 Redis 做缓存和 Celery 后端

### 方式 6：把 Flow 打包成 Docker 镜像（分发用）

你做好的 flow 可以单独打包成一个自包含镜像，给别人直接 `docker run`：

```dockerfile
FROM langflowai/langflow:latest
RUN mkdir /app/flows
COPY ./*.json /app/flows/
ENV LANGFLOW_LOAD_FLOWS_PATH=/app/flows
```

```bash
docker build -t myflow:1.0.0 .
docker run -p 7860:7860 myflow:1.0.0
```

这个模式适合发布 demo、给客户交付、或者把 flow 当微服务部署到 K8s。

## 三、核心概念速览

### Flow（流程）

Langflow 的顶层抽象。一个 flow = 若干 component（组件）连起来的 DAG。画完一个 flow，就能作为 API 调用或 MCP 工具被外部使用。

### Component（组件）

最小单位。每个组件是一个 Python 类，有 input/output 端口。官方内置数百个组件，覆盖：

- **Inputs**：Chat Input、Text Input、File
- **Models**：OpenAI、Anthropic、Ollama、Groq、xAI、各类 embedding
- **Vector Stores**：Astra DB、Pinecone、Chroma、Weaviate、Qdrant、PGVector
- **Agents**：ReAct、Tool Calling Agent、Autogen、CrewAI
- **Tools**：Search、Calculator、Python REPL、API Request、MCP Tools
- **Memory**：Message History、Redis Cache、Astra DB Memory
- **Logic**：Conditional Router、Loop、Iterator

### Playground（调试台）

点右上角「Playground」打开对话界面，可以实时测试 flow。支持单步运行单个组件，排查问题很方便。

### API & MCP Server

每个 flow 画完就是一个 HTTP API：

```bash
curl -X POST "http://localhost:7860/api/v1/run/<flow_id>?stream=false" \
  -H "Content-Type: application/json" \
  -H "x-api-key: <your_api_key>" \
  -d '{
    "input_value": "Hello",
    "output_type": "chat",
    "input_type": "chat"
  }'
```

同时可以把 flow 作为 MCP Server 暴露，给 Claude Desktop、Cursor 等 MCP 客户端调用——这是 1.9 版本强化的重点。

## 四、15 分钟上手实战

### 场景：搭一个带 RAG 的客服机器人

打开 Langflow，点「New Flow」→「Blank Flow」。

**第 1 步：加载文档**

拖一个 `File` 组件，上传 PDF 或 txt。再拖 `Split Text` 组件，按 chunk 切分。

**第 2 步：做向量化**

拖 `OpenAI Embeddings`（或 `Ollama Embeddings` 走本地），连接到 `Chroma` / `Astra DB` 向量库。

**第 3 步：检索链路**

`Chat Input` → `Chroma Retriever` → 拼到 Prompt 模板里。

**第 4 步：LLM 生成**

`Prompt` 组件连到 `OpenAI`（或 `Anthropic`、`Groq`、`Ollama`）模型组件，输出到 `Chat Output`。

**第 5 步：测试**

点 Playground，提问就能看到带文档上下文的回答。

**第 6 步：导出**

右上角「API」按钮，复制 cURL / Python / JS 示例代码，直接嵌进你的应用。

全程 15 分钟，不用写一行 LangChain 代码。

## 五、高级用法：自定义组件

内置组件不够用时，写一个 Python 组件扔进去：

```python
from langflow.custom import Component
from langflow.io import StrInput, Output
from langflow.schema import Data

class WordCountComponent(Component):
    display_name = "Word Count"
    description = "统计文本词数"
    icon = "text"

    inputs = [
        StrInput(name="text", display_name="Text", required=True),
    ]

    outputs = [
        Output(display_name="Count", name="count", method="count_words"),
    ]

    def count_words(self) -> Data:
        count = len(self.text.split())
        return Data(data={"count": count})
```

放到 `LANGFLOW_COMPONENTS_PATH` 指向的目录，重启就能在侧边栏看到。

这点是 Langflow 相比 Dify 的核心优势——你可以任意用 Python 扩展，不用等官方支持。

## 六、竞品分析

AI 应用可视化构建这个赛道现在非常挤，主要竞品分三类：

### 类别 1：同类直接竞品（可视化 LLM 工作流）

#### Dify

| 维度 | 详情 |
|------|------|
| Stars | 129.8k（2026 年数据） |
| 语言 | Python（后端）+ Next.js（前端） |
| 开源协议 | 修改版 Apache 2.0（有商用限制） |
| 内置模型 | 支持 OpenAI、Anthropic、Moonshot、通义等数百种 |
| 特色 | 内置 RAG 知识库、Dataset、Workflow、Agent 四种应用类型 |
| 生态 | 国内生态强，中文文档完善 |

**和 Langflow 比**：
- Dify 更偏向**应用平台**：一站式，上传文档→建知识库→发布对话式应用，非技术人员友好
- Langflow 更偏向**开发框架**：灵活度高，可以写 Python 深度改造
- Dify 的 License 有商用限制（多租户 SaaS 需要授权），Langflow 是纯 MIT
- Dify 的数据分析、成本统计、用户管理更完善；Langflow 在这些运营层面薄一些

**选 Dify**：要快速搭对话产品上线、团队非技术主导、对商用限制不敏感。
**选 Langflow**：要嵌入自家应用做 AI 能力层、需要深度定制、重视 Python 生态兼容。

#### Flowise

| 维度 | 详情 |
|------|------|
| Stars | ~36k |
| 语言 | TypeScript / Node.js |
| 定位 | JS 版 Langflow |
| 特色 | 轻量、部署快，对 JS/TS 团队友好 |

**和 Langflow 比**：
- Flowise 基于 LangchainJS，组件库比 Langflow 小
- TypeScript 技术栈的团队用它更顺手
- Langflow 的组件丰富度、企业级特性（RBAC、审计日志）更强

**选 Flowise**：纯 Node 技术栈团队、对性能没极端要求。

#### n8n

| 维度 | 详情 |
|------|------|
| Stars | ~62k |
| 语言 | TypeScript |
| 定位 | 通用工作流自动化（不只 AI） |
| 特色 | 400+ 集成，覆盖企业 SaaS |

**和 Langflow 比**：
- n8n 本质是 Zapier/Make 的开源版，AI 只是它的一个场景
- 集成深度（Slack、Salesforce、Jira 等）远超 Langflow
- AI 相关的原生组件（特别是 agent、RAG）弱于 Langflow

**选 n8n**：要做「邮件→提取数据→写入 CRM→通知 Slack」这类 AI + 业务系统编排。
**选 Langflow**：做纯 AI 应用（chatbot、RAG、Agent）。

### 类别 2：代码框架（非可视化）

#### LangChain / LangGraph

- LangChain 是组件库，LangGraph 是状态机编排框架
- 完全代码优先，学习曲线陡，但灵活度最高
- Langflow 底层大量使用 LangChain

**选 LangChain/LangGraph**：代码能力强的团队、复杂的多 Agent 编排（human-in-the-loop、长期记忆）。
**选 Langflow**：需要快速原型、需要给产品经理和业务方演示、对 UI 调试依赖度高。

#### CrewAI

- 基于「角色扮演」，定义 researcher、writer、reviewer 等角色协作
- API 简洁，20 分钟起一个多 Agent 系统
- 纯代码，没 GUI

**和 Langflow 比**：
- CrewAI 解决「多 Agent 协作」一个问题
- Langflow 覆盖「单 Agent + RAG + 工作流」更广的范围
- 可以组合：Langflow 里的 CrewAI 组件就是把 CrewAI 封装成了节点

### 类别 3：企业级平台

#### LangSmith / LangGraph Platform

- LangChain 官方的商业产品
- 偏重可观测性、评估、prompt 管理
- 付费，但有免费档

Langflow 没有 LangSmith 这类深度的评估体系，但可以集成 LangSmith 做观测。

#### Coze / 百炼 / 千帆 AppBuilder

国内大厂的托管平台：
- 优势：零部署，国内网络可达，集成本家 LLM
- 劣势：绑死生态、不开源、不能私有化

Langflow 的优势是完全开源 + MIT 协议 + 自主部署，数据不出域。

### 一张表对比

| 工具 | 可视化 | 开源 | 主语言 | 企业级 | 最适合 |
|------|--------|------|--------|--------|--------|
| **Langflow** | ✅ | MIT | Python | 中 | Python 团队、深度定制 |
| **Dify** | ✅ | Apache 2.0* | Python/TS | 强 | 业务团队、快速上线 |
| **Flowise** | ✅ | Apache 2.0 | TypeScript | 中 | JS 团队 |
| **n8n** | ✅ | Fair Code | TypeScript | 强 | 自动化 + AI |
| **LangGraph** | ❌ | MIT | Python | 强 | 复杂 Agent 编排 |
| **CrewAI** | ❌ | MIT | Python | 中 | 多 Agent 协作 |
| **Coze/Dify Cloud** | ✅ | 闭源 | - | 强 | 非技术用户、零运维 |

_*Dify 有商用限制_

## 七、Langflow 的优势与局限

### 优势

1. **Python 生态兼容最好**：底层就是 LangChain，pip 里的库基本都能接
2. **MIT 协议纯净**：商用零顾虑
3. **组件可直接改代码**：界面上点「Code」就能编辑 Python 源码
4. **API / MCP 双暴露**：flow 既是 API 又是 MCP Server
5. **Star 数 148k**：社区活跃，组件更新快

### 局限

1. **性能不是强项**：本质是画布驱动，大流量场景比手写 FastAPI 慢
2. **协作和版本管理弱**：flow 存在数据库里，不像代码那样 Git diff 友好
3. **前端有时候卡**：复杂 flow 在浏览器里拖拽会明显延迟
4. **企业特性成熟度不如 Dify**：RBAC、审计、成本统计等仍在补齐
5. **文档更新滞后**：1.x 版本迭代快，老教程经常失效

## 八、选型建议

**直接用 Langflow 的场景**：
- 团队用 Python，要深度改组件
- 做 AI 能力后端（给 App/Web 提供 API）
- 不想被 SaaS 绑架，要完全开源自托管
- 预算有限，希望快速验证 AI 想法

**换 Dify 的场景**：
- 要直接给终端用户用（chat UI）
- 团队不全是工程师，有产品/运营参与建 flow
- 需要完整的知识库管理和数据分析

**换 LangGraph / CrewAI 的场景**：
- 流程复杂到用画布表达反而费劲
- 需要精细的状态控制、循环、回溯
- 团队代码能力强，更习惯写代码

**换 n8n 的场景**：
- AI 只是自动化管道的一环
- 要对接大量 SaaS 系统
- 需要定时任务、webhook 触发

## 九、生产部署 checklist

如果决定用 Langflow 上生产，这些必做：

- [ ] 用 Docker Compose 或 K8s，**禁止** `docker run` 单容器
- [ ] PostgreSQL 放托管服务（RDS/Cloud SQL），不要自建
- [ ] `LANGFLOW_SECRET_KEY` 必须设置且定期轮换
- [ ] 关闭 `LANGFLOW_AUTO_LOGIN`，强制登录
- [ ] 前面加 Nginx/Caddy 做 HTTPS 和限流
- [ ] API Key 按 flow 隔离，最小权限
- [ ] 开启 LangSmith 或 Langfuse 做 trace 和成本监控
- [ ] 定期备份数据库（flow JSON、用户、配置全在里面）
- [ ] 监控 OpenAI/Anthropic 等 API 调用成本，设预算告警
- [ ] 组件代码修改走 CI，别在 UI 里直接改（不可 diff、不可回溯）

## 总结

Langflow 在「可视化 + Python 可改」这个结合点上做得比较好，适合有工程能力但想加速原型的团队。如果你正在选型：

- **入门和学习**：直接下 Desktop 版玩一周
- **个人项目或 PoC**：uv pip install 最快
- **团队共享**：Docker Compose + Postgres
- **生产**：K8s + Helm + 托管数据库

和 Dify 不是替代关系，是风格差异——Dify 更像 SaaS 平台，Langflow 更像 SDK + UI。选哪个取决于你团队的能力画像和对「自主可控」的敏感度。

参考资源：
- 官方仓库：<https://github.com/langflow-ai/langflow>
- 官方文档：<https://docs.langflow.org/>
- Desktop 下载：<https://www.langflow.org/desktop>
- Helm Charts：<https://github.com/langflow-ai/langflow-helm-charts>
