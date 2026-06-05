---
{"dg-publish":true,"permalink":"/01-技术分享/CLI AI 编程全栈开发必备 Skills 推荐（2026）/","tags":["AI","CLI","skills","全栈开发","kiro","claude-code","开发工具","效率工具"],"noteIcon":"","created":"2026-06-05T12:31:50.465+08:00","updated":"2026-06-05T12:31:50.466+08:00"}
---


# CLI AI 编程：全栈开发必备 Skills 推荐（2026）

> AI CLI 编程工具——Kiro、Claude Code、Cursor、Codex、Gemini CLI——已经把"AI 帮你写代码"变成日常。但大多数开发者只用到了这些工具最表层的能力：让 AI 生成一段代码、回答一个问题。真正决定生产力上限的，是 **Skills**——一套可复用的专家级工作流包，让 AI 知道"在什么阶段该做什么、怎么做"。
>
> 这篇文章系统梳理从设计到部署全流程的必备 Skills，涵盖 Kiro CLI、Claude Code 等主流工具，帮你把 AI 从"智能自动补全"升级为真正的开发伙伴。

---

## 一、先搞清楚：Skills 是什么

Skills 不是插件，也不是 Prompt 模板——它是**可移植的工作流包**。

一个 Skill 的基本结构：

```
my-skill/
├── SKILL.md          # 必须：元数据 + 详细指令
└── references/       # 可选：参考文档、检查清单、模板
    └── checklist.md
```

`SKILL.md` 里的内容决定了 AI 在这个场景下的完整行为：该问哪些问题、该走哪些步骤、该有什么验收标准、该怎么规避常见陷阱。

**工作机制是渐进式激活（Progressive Disclosure）：**

1. **Discovery**：Agent 启动时，只加载所有 Skill 的 name 和 description，context 占用极小
2. **Activation**：当你的请求匹配到某个 Skill 的描述，或你用 `/skill-name` 斜杠命令显式调用，完整指令才被加载进 context
3. **Execution**：Agent 按照 Skill 指令执行，按需读取 references/ 里的参考文档

**兼容性**：Skills 遵循 [agentskills.io](https://agentskills.io) 开放标准，写一次可以在 Kiro、Claude Code、Cursor、Codex、Gemini CLI 等多个工具里复用。

**安装位置**：
- `~/.kiro/skills/`：全局，跨项目通用
- `.kiro/skills/`（项目目录下）：项目级，随代码库 version control

---

## 二、全流程 Skills 地图

先看整体：一个功能从想法到上线，需要哪些阶段的 Skills 支撑。

```
💡 需求探索
    │
    ▼
📐 设计与 Spec        ← brainstorming / design-taste-frontend
    │
    ▼
📋 实施计划           ← writing-plans
    │
    ▼
🔍 参考实现搜索        ← github-search-before-code
    │
    ▼
🏗️ 开发实现           ← test-driven-development
    │             ← subagent-driven-development / executing-plans
    ▼
🔎 代码审查           ← /review (agent-skills)
    │
    ▼
🔒 安全加固           ← harden / security-audit
    │
    ▼
✅ 质量评估           ← audit / /test (agent-skills)
    │
    ▼
📦 部署上线           ← /ship (agent-skills) / cdk-deploy
    │
    ▼
📝 文档维护           ← technical-docs / knowledge-base
```

下面按阶段逐一展开。

---

## 三、设计阶段：先想清楚再动手

### 3.1 brainstorming — 创意工作的必经入口

**适用工具**：Kiro、Claude Code、Cursor  
**来源**：主流工具内置/社区常见  
**触发方式**：任何创意工作开始前

这是最容易被跳过、却最值得坚持的一个 Skill。

**它做什么**：

- 先探索项目现有上下文（代码、文档、最近提交）
- 逐个问清需求，不多问，不少问
- 提出 2-3 个方案并对比 tradeoff
- 分段呈现设计，逐段获取确认
- 写 Spec 文档并做自我检查
- 最终交棒给 writing-plans

**核心价值**：AI 不知道你真正想要什么。brainstorming 强制 AI 在动手之前把你的想法结构化，输出一份精确到 AI 可以据此生成代码的 Spec。

```
HARD-GATE: 未呈现设计并获得用户确认前，禁止执行任何实现操作。
```

这个硬性门控防止了 AI 最常见的问题：没理解需求就开始写代码。

**安装**：

```bash
# Kiro
kiro skills install brainstorming

# Claude Code（全局）
mkdir -p ~/.claude/skills/brainstorming
# 将 SKILL.md 放入目录
```

---

### 3.2 design-taste-frontend — 拒绝"AI 味"界面

**适用工具**：Kiro、Claude Code  
**来源**：skillsmp.com / 社区  
**触发方式**：需要设计 landing page、portfolio、SaaS 产品界面

普通让 AI 设计界面，得到的是千篇一律的蓝白渐变+圆角卡片。这个 Skill 专门解决这个问题。

**它做什么**：

- 先做 `audit-first`（重设计时先审计现状）
- 读懂 brief，推断正确的设计方向
- 在真实设计系统框架内工作（Tailwind、shadcn 等）
- 严格的 pre-flight check 防止生成模板化界面

---

### 3.3 /spec — 精确的功能规格

**适用工具**：Claude Code、Kiro（来自 addyosmani/agent-skills）  
**触发方式**：`/spec` 斜杠命令

这是 addyosmani/agent-skills 包里的核心命令之一，专门为 AI 工作流设计的 Spec 生成器。

**产出**：
- 明确的功能边界（做什么/不做什么）
- 用户场景（User Stories）
- 验收标准
- 技术约束

**安装 addyosmani/agent-skills 包**（包含 `/spec /plan /build /test /review /ship`）：

```bash
# Claude Code
/plugin marketplace add addyosmani/agent-skills
/plugin install agent-skills@addy-agent-skills

# Gemini CLI
gemini skills install https://github.com/addyosmani/agent-skills
```

---

## 四、规划阶段：动代码之前先写计划

### 4.1 writing-plans — 生成可执行的实施计划

**适用工具**：Kiro、Claude Code  
**触发方式**：有 Spec 之后、动代码之前

设计通过了，下一步不是写代码，是写计划。writing-plans 把 Spec 转化成逐步可执行的任务列表。

**计划的粒度**：

```markdown
### Task N: 实现 Token 刷新逻辑

**Files:**
- Create: `src/auth/token-refresh.ts`
- Modify: `src/auth/interceptor.ts:45-67`
- Test: `tests/auth/token-refresh.test.ts`

- [ ] 步骤 1: 写失败测试
- [ ] 步骤 2: 确认测试确实失败
- [ ] 步骤 3: 写最小实现代码
- [ ] 步骤 4: 确认测试通过
- [ ] 步骤 5: 提交
```

每个步骤 2-5 分钟，精确到文件路径和预期输出。

**计划头部模板**（每份计划必须包含）：

```markdown
# [功能名] 实施计划

> 对 Agentic workers：推荐使用 subagent-driven-development 或 executing-plans 技能来执行本计划。

**Goal:** [一句话描述目标]
**Architecture:** [2-3 句话描述方案]
**Tech Stack:** [核心技术栈]
```

---

### 4.2 /plan — 快速任务拆解

**适用工具**：Claude Code、Kiro（来自 addyosmani/agent-skills）  
**触发方式**：`/plan` 斜杠命令

比 writing-plans 更轻量的计划生成器，适合中小型任务。

---

## 五、开发阶段：高质量代码的流水线

### 5.1 github-search-before-code — 先找参考实现

**适用工具**：Kiro、Claude Code  
**触发方式**：实现全新功能、遇到重复失败

**不要重复造轮子**——但 AI 经常不知道有已经存在的高质量实现。这个 Skill 在写代码之前先搜 GitHub，找到并分析类似的实现，再根据项目需求改编。

**触发条件**：
1. 实现当前代码库里不存在的全新功能、算法或模块
2. 用户说"还是不行"、"改了很多次了"等反复失败的信号
3. AI 自己识别到需要实现不熟悉的复杂功能

**搜索策略**：

```
领域关键词 + 功能关键词 + 语言
例："harmonic analysis power metering" C
例："web scraping pagination" Python
```

---

### 5.2 test-driven-development — TDD 的铁律

**适用工具**：Kiro、Claude Code  
**触发方式**：实现任何功能或修复 bug 之前

TDD 不是选做题。这个 Skill 强制执行 Red → Green → Refactor 循环，并且有硬性规则：

```
铁律：没有先失败的测试，不写生产代码。
      先写了代码？删掉。重来。
```

**完整流程**：

```
RED：写一个预期会失败的测试
  ↓
验证 RED：运行测试，确认它真的失败，且失败原因正确
  ↓
GREEN：写最小代码让测试通过（不多写一行）
  ↓
验证 GREEN：运行测试，确认通过，且其他测试没有回归
  ↓
REFACTOR：清理代码，保持测试绿色
  ↓
重复，处理下一个行为
```

**为什么"测试后写"不等于 TDD**：

- 后写测试会立刻通过，无法证明它真的在测试正确的东西
- 先写测试强迫你在实现之前就发现边界情况
- "先写测试"是提问"应该做什么"，"后写测试"是验证"做了什么"

---

### 5.3 executing-plans — 按计划精确执行

**适用工具**：Kiro、Claude Code  
**触发方式**：有现成的 writing-plans 产出的计划

适合在独立 session 里执行一份已有计划，每个任务完成后有 checkpoint 让人类审查。

**流程**：

```
读取计划 → 批判性审查（发现问题先报告）→ 逐步执行
→ 遇到阻塞立即停下来提问（不靠猜）→ 全部完成后交收尾流程
```

关键原则：**阻塞就停，不猜测**。

---

### 5.4 subagent-driven-development — 并行多 Agent 流水线

**适用工具**：Kiro、Claude Code  
**触发方式**：计划任务互相独立、当前 session 内执行

这是开发阶段最强的提速工具。用多个独立子 Agent 并行执行计划，每个任务完成后走两阶段审查：

```
实现 subagent → spec compliance review → code quality review → 标记完成
```

**双阶段审查**：
1. **Spec 合规审查**：代码是否符合规格？有没有欠实现？有没有超范围实现？
2. **代码质量审查**：实现是否足够好？有没有技术债？

两个审查都通过才算完成，不打折扣。

**模型选择策略**（省钱又不降质）：

| 任务类型 | 推荐模型 |
|---------|---------|
| 机械性实现（1-2 个文件，spec 明确） | 快速/廉价模型 |
| 集成任务（多文件协调） | 标准模型 |
| 架构设计、审查 | 最强模型 |

---

## 六、代码审查：不让 AI 的错误溜进主干

### 6.1 /review — 生产工程级代码审查

**适用工具**：Claude Code、Kiro（来自 addyosmani/agent-skills）  
**触发方式**：`/review` 斜杠命令

不是简单的"帮我看看这段代码"，而是走完整的审查清单：

- 安全性（注入、暴露的密钥、权限）
- 边界情况和失败模式处理
- 测试覆盖率
- 代码重复和设计问题
- 文档和可维护性

---

### 6.2 /code-simplify — 消灭不必要的复杂度

**适用工具**：Claude Code、Kiro（来自 addyosmani/agent-skills）  
**触发方式**：`/code-simplify` 斜杠命令

AI 生成的代码经常过度抽象——不必要的中间层、冗余的错误处理、用不到的配置项。这个 Skill 专门做减法，把代码精简到真正需要的程度。

YAGNI（You Ain't Gonna Need It）原则的执行器。

---

## 七、安全与质量：上线前的最后防线

### 7.1 harden — 生产级健壮性

**适用工具**：Kiro、Claude Code  
**触发方式**：功能开发完成后、上线前

AI 写的代码在 happy path 上通常没问题，边界情况是重灾区。harden 系统性地检查并修复：

- **错误处理**：各种失败模式有没有优雅降级
- **输入边界**：超长文本、空值、特殊字符、RTL 文字
- **国际化（i18n）**：是否支持多语言、多时区
- **文本溢出**：UI 在极端文本长度下会不会崩
- **竞态条件**：并发操作是否安全

---

### 7.2 audit — 界面质量全面审计

**适用工具**：Kiro、Claude Code  
**触发方式**：前端功能完成后

全面扫描界面质量，生成带优先级的问题报告：

- **可访问性（A11y）**：色彩对比度（4.5:1 标准）、键盘导航、ARIA 标签
- **性能**：不必要的重渲染、图片优化、bundle 大小
- **响应式设计**：不同屏幕尺寸的适配
- **主题一致性**：是否遵循设计系统

注意：audit 只产出报告，不修改代码。修改由 harden、optimize 等专项 Skill 处理。

---

### 7.3 /test — 系统性测试

**适用工具**：Claude Code、Kiro（来自 addyosmani/agent-skills）  
**触发方式**：`/test` 斜杠命令

不只是跑现有测试，而是检查测试覆盖率、发现没有测试的关键路径、补写缺失的测试。

---

## 八、部署阶段：安全地把代码推上线

### 8.1 /ship — 发布前检查清单

**适用工具**：Claude Code、Kiro（来自 addyosmani/agent-skills）  
**触发方式**：`/ship` 斜杠命令

这是发布流程的最后一道门控，确保代码上线前满足所有标准：

- 所有测试通过
- 文档更新
- Breaking changes 已标注
- 版本号符合语义化版本规范
- 部署配置检查

---

### 8.2 cdk-deploy — AWS CDK 标准化部署

**适用工具**：Kiro（官方示例 Skill）  
**触发方式**：部署 CDK stack 时

Kiro 官方文档里的标准部署 Skill 示例：

```markdown
---
name: cdk-deploy
description: Deploy AWS CDK stacks with best practices. Use when deploying 
infrastructure, running cdk deploy, or troubleshooting CDK issues.
---

## Deployment workflow

1. cdk synth 验证模板
2. cdk diff 预览变更
3. cdk deploy 并审查 IAM 变更

## Pre-deployment checks
- 确认 AWS credentials 配置
- 确认 CDK 版本匹配
- 查阅环境专属的 stack 模式

## Rollback procedure
...
```

按这个模式，可以为你的具体部署环境（Vercel、Docker、K8s 等）编写定制 Skill。

---

## 九、效率工具：横跨全流程的辅助 Skills

### 9.1 agent-reach — 给 AI 装上眼睛

**适用工具**：Kiro  
**触发方式**：搜索网页、读文章、查 GitHub、刷社交媒体

通过 17 个平台的统一接口，让 AI 直接访问外部信息，不再局限于本地代码库：

- 网页搜索（Exa AI）
- GitHub 搜索和操作
- 社交媒体（小红书、抖音、Twitter、B站、Reddit、V2EX）
- 公众号和 RSS 阅读
- YouTube/B站字幕提取

零配置开箱可用 8 个渠道。

---

### 9.2 find-skills — 技能发现助手

**适用工具**：Kiro  
**触发方式**：用户问"有没有做 X 的 skill"、"怎么扩展某个能力"

当你不知道有没有合适的 Skill，或者想给工作流添加新能力时，这个 Skill 会帮你搜索和安装。

---

### 9.3 vercel-react-best-practices — React 性能规范

**适用工具**：Kiro  
**触发方式**：写、审查或重构 React/Next.js 代码时

Vercel 工程团队总结的 React 和 Next.js 性能优化准则，自动约束 AI 遵循最佳实践：

- 避免不必要的重渲染
- 正确使用 `use client` / `use server`
- 数据获取模式（Server Components 优先）
- Bundle 优化

---

## 十、推荐安装清单

按需分级安装，不是越多越好。每个 Skill 都会占用 context 空间，安装不用的 Skill 只会增加噪音。

### 基础包（所有全栈开发者）

```bash
# 覆盖设计→计划→TDD→审查→发布完整流程
addyosmani/agent-skills   # /spec /plan /build /test /review /code-simplify /ship

brainstorming             # 设计前必备
writing-plans             # 实施计划
test-driven-development   # TDD 执行
github-search-before-code # 找参考实现
```

### 前端开发者追加

```bash
frontend-design           # 高质量界面设计
audit                     # 界面质量审计
harden                    # 健壮性强化
vercel-react-best-practices  # React 最佳实践
```

### 团队/复杂项目追加

```bash
subagent-driven-development  # 并行多 Agent 执行
executing-plans              # 按计划逐步执行（单 session）
```

### 有外部信息需求时

```bash
agent-reach               # 网页/社交/GitHub 访问
```

---

## 十一、如何写自己的 Skill

已有的社区 Skills 覆盖通用场景。项目特定的知识——你的团队规范、技术栈约定、部署流程——需要自己写。

**一个最简 Skill**：

```markdown
---
name: api-review
description: Review API endpoints for consistency, security, and documentation. 
Use when adding or modifying REST API endpoints.
---

## Review checklist

1. URL 命名遵循 RESTful 规范（名词复数）
2. HTTP 动词使用正确（GET 不修改数据）
3. 错误码语义准确（400 客户端错误，500 服务端错误）
4. 参数校验有没有做
5. 敏感字段是否加密
6. Swagger 文档是否同步更新
7. 速率限制是否配置
```

**Description 写得好，Skill 才会在对的时机触发**：

- ✅ 好：`Review API endpoints for consistency, security, and documentation. Use when adding or modifying REST API endpoints.`
- ❌ 差：`Helps with APIs`

---

## 十二、总结

Skills 生态现在已经相当成熟。核心认知：

**Skills 不是让 AI 更聪明，而是让 AI 在正确的时机做正确的事。**

一个没有 Skills 的 AI 助手，每次对话都是从零开始。有了覆盖完整开发流程的 Skills，你的 AI 助手会知道：需求阶段该问什么问题、实现阶段该走什么循环、上线前该过哪些检查。

开发效率的提升不是来自 AI 写代码更快，而是来自**更少的返工、更早发现问题、更可预期的输出质量**。这正是一套好的 Skills 体系能带来的。

---

## 附录：Skills 资源导航

| 资源 | 说明 |
|------|------|
| [agentskills.io](https://agentskills.io) | 开放标准规范 |
| [skillsmp.com](https://skillsmp.com) | 最大的 Skills 市场（120 万+ Skills）|
| [github.com/addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) | 生产工程 Skills 包 |
| [github.com/anthropics/skills](https://github.com/anthropics/skills) | Anthropic 官方 Skills |
| [kiro.dev/docs/cli/skills](https://kiro.dev/docs/cli/skills/) | Kiro CLI Skills 文档 |
| [skillsmp.com/categories/testing-security](https://skillsmp.com/categories/testing-security) | 测试与安全类 Skills |
| [skillsmp.com/categories/devops](https://skillsmp.com/categories/devops) | DevOps 类 Skills |
