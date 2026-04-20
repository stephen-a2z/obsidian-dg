---
{"dg-publish":true,"permalink":"/03-学习笔记/how-to-design-spec-driven-development/","noteIcon":"","created":"2026-04-20T13:33:40.976+08:00","updated":"2026-04-20T13:38:59.615+08:00"}
---



# 如何设计 Spec-Driven Development：一份提示词工程师的拆解指南

> 本文不是介绍 Spec-Driven Development 是什么, [上一篇](prompt-engineering-development-methodologies.md)已经做了），而是拆解它是**怎么被设计出来的**——每一条 prompt 规则背后的设计意图是什么，为什么选择这种写法而不是另一种，以及你如何为自己的场景设计类似的体系。

---

## 一、设计的起点：AI Agent 有哪些"缺陷"需要被对冲？

设计任何 prompt 体系之前，先要理解你的"执行者"有什么特点。AI Agent 有几个结构性缺陷，整个 Spec-Driven Development 体系就是围绕对冲这些缺陷来设计的：

| AI Agent 的缺陷 | 导致的问题 | 对冲手段 |
|----------------|-----------|---------|
| 急于给出答案 | 没想清楚就开始写代码 | 硬门禁：设计未批准前禁止写代码 |
| 过度生成 | 加了一堆没人要的功能 | YAGNI 原则 + 规格审查（不多不少） |
| 上下文退化 | 长对话后质量下降 | 子代理隔离，每个任务全新上下文 |
| 自信但不一定正确 | 说"没问题"但实际有 bug | TDD 强制验证 + 两阶段审查 |
| 没有长期记忆 | 不了解你的代码库 | 计划中写完整代码和精确路径 |
| 容易被已有代码影响 | 测试适配实现而非定义行为 | 先写测试，违反则删除重来 |
| 不会主动提问 | 遇到歧义自己猜 | prompt 中明确鼓励提问，定义 NEEDS_CONTEXT 状态 |

理解了这张表，你就理解了整个体系中每一条规则存在的理由。

---

## 二、整体架构：阶段分离 + 门禁设计

### 2.1 为什么要分阶段？

最朴素的做法是写一个巨大的 prompt，把所有规则塞进去。但这行不通：

- AI 的注意力有限，规则太多会互相稀释
- 不同阶段需要不同的"思维模式"（发散 vs 收敛 vs 执行）
- 没有门禁，AI 会跳过它认为"不必要"的步骤

所以体系被拆成了独立的 skill 文件，每个 skill 只负责一个阶段：

```
brainstorming/SKILL.md    → 发散思考，探索需求
writing-plans/SKILL.md    → 收敛规划，分解任务
subagent-driven-development/SKILL.md → 执行 + 审查
test-driven-development/SKILL.md     → 贯穿始终的编码纪律
```

每个 skill 有明确的**入口条件**和**出口条件**，形成流水线：

```
brainstorming 的出口 = writing-plans 的入口（spec 文档）
writing-plans 的出口 = subagent-driven-development 的入口（计划文档）
```

### 2.2 门禁的设计技巧

门禁是整个体系中最关键的设计元素。来看几种不同强度的门禁写法：

**弱门禁（几乎无效）：**
```
建议在写代码之前先做设计。
```
AI 会直接忽略"建议"。

**中等门禁（有时有效）：**
```
在写代码之前，你应该先完成设计并获得用户确认。
```
AI 在简单任务上会跳过，因为它判断"这个太简单不需要设计"。

**强门禁（实际采用的写法）：**
```xml
<HARD-GATE>
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action until you have presented a design and the
user has approved it. This applies to EVERY project regardless of perceived
simplicity.
</HARD-GATE>
```

注意这里的设计技巧：

1. **用 XML 标签包裹**——视觉上突出，AI 更容易注意到
2. **穷举禁止行为**——不只是"不要写代码"，而是列出所有可能的绕过方式（invoke skill、scaffold、implementation action）
3. **堵住最常见的绕过借口**——"regardless of perceived simplicity"直接封死了"这个太简单"的逃逸路径
4. **配合反模式说明**——紧接着有一段 "Anti-Pattern: This Is Too Simple To Need A Design"，进一步强化

**设计原则：门禁的强度要匹配 AI 绕过它的动机强度。** AI 越想跳过的步骤，门禁就要写得越严格、越具体。

### 2.3 阶段之间的衔接

每个 skill 的末尾都明确指定了"下一步调用什么"：

```markdown
# brainstorming 的末尾：
**The terminal state is invoking writing-plans.**
Do NOT invoke frontend-design, mcp-builder, or any other implementation skill.
The ONLY skill you invoke after brainstorming is writing-plans.

# writing-plans 的末尾：
**If Subagent-Driven chosen:**
- REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
```

这里的设计技巧是**白名单而非黑名单**——不是说"不要调用 X、Y、Z"，而是说"只能调用 A"。白名单更安全，因为你无法穷举所有不该调用的东西。

---

## 三、Brainstorming Skill 的设计拆解

### 3.1 交互模式设计：为什么"每次只问一个问题"？

```markdown
- Only one question per message
- Prefer multiple choice questions when possible
```

这不是随意的偏好，而是基于两个观察：

1. **AI 一次问多个问题时，用户往往只回答其中一两个**，剩下的被忽略，AI 就自己猜了
2. **选择题比开放题更容易回答**，降低用户的认知负担，加快对话节奏

这是一个**降低用户摩擦**的设计——如果流程让用户觉得麻烦，用户就会说"别问了直接写吧"，整个 Spec-Driven 流程就崩了。

### 3.2 范围控制：先拆分再深入

```markdown
Before asking detailed questions, assess scope: if the request describes
multiple independent subsystems, flag this immediately. Don't spend questions
refining details of a project that needs to be decomposed first.
```

这条规则防止了一个常见陷阱：用户说"帮我做一个电商平台"，AI 开始问"商品列表要分页吗？"——在还没搞清楚整体范围的时候就陷入了细节。

设计原则：**先确定边界，再填充细节。** 这和软件架构设计的原则一致。

### 3.3 方案探索：为什么必须提 2-3 种？

```markdown
Propose 2-3 different approaches with trade-offs
Lead with your recommended option and explain why
```

如果只让 AI 提一种方案，它会给出它认为"最好"的——但 AI 的"最好"往往是最复杂、最"全面"的方案。强制提多种方案有两个好处：

1. **暴露权衡**——用户能看到不同方案的代价，做出知情决策
2. **防止过度工程**——简单方案会作为选项之一出现，用户可能会选它

"Lead with recommended"是另一个细节——AI 先说推荐方案，用户如果没有强烈偏好就会选推荐的，减少决策疲劳。

### 3.4 分段呈现：渐进式确认

```markdown
Scale each section to its complexity: a few sentences if straightforward,
up to 200-300 words if nuanced.
Ask after each section whether it looks right so far.
```

为什么不一次性呈现完整设计？因为：

1. **一次性呈现太长，用户不会仔细看**——说"看起来不错"然后跳过
2. **分段呈现让用户在每个决策点都有机会纠偏**——比最后推倒重来成本低得多
3. **"scaled to complexity"防止了过度文档化**——简单的部分一两句话就够了

### 3.5 自审查清单：四维检查

```markdown
1. Placeholder scan: Any "TBD", "TODO", incomplete sections?
2. Internal consistency: Do any sections contradict each other?
3. Scope check: Is this focused enough for a single implementation plan?
4. Ambiguity check: Could any requirement be interpreted two different ways?
```

这四个维度不是随意选的，它们对应了 spec 文档最常见的四类问题：

| 维度 | 对应的问题 | 如果不检查会怎样 |
|------|-----------|----------------|
| 占位符 | AI 写了 "TBD" 然后忘了填 | 执行者遇到 TBD 不知道该做什么 |
| 内部一致性 | 架构说用 Redis，数据模型说用 PostgreSQL | 执行者无所适从 |
| 范围 | 一个 spec 包含了三个独立子系统 | 计划太大，无法有效执行 |
| 歧义 | "支持多种格式"——哪些格式？ | 执行者自己猜，猜错了返工 |

设计原则：**自审查清单要针对具体的、高频的问题类型，而不是泛泛的"检查质量"。**

---

## 四、Writing Plans Skill 的设计拆解

### 4.1 核心假设：为什么假设执行者"零上下文"？

```markdown
Write comprehensive implementation plans assuming the engineer has zero context
for our codebase and questionable taste.
```

这句话是整个 plan 设计的基石。它有两层含义：

1. **"零上下文"**——执行者（子代理）确实不了解你的代码库，所以计划必须自包含
2. **"品味存疑"**——执行者可能会做出糟糕的设计决策，所以计划要精确到不需要判断

这个假设直接决定了计划的粒度：不是"实现用户验证模块"，而是精确到每一行代码、每一条命令、每一个预期输出。

### 4.2 粒度设计：为什么 2-5 分钟一步？

```markdown
Each step is one action (2-5 minutes):
- "Write the failing test" - step
- "Run it to make sure it fails" - step
- "Implement the minimal code to make the test pass" - step
```

这个粒度是为 AI Agent 量身定制的：

- **太粗**（"实现整个模块"）→ AI 会过度生成，加入不需要的功能
- **太细**（"在第 3 行添加 import 语句"）→ 计划本身变得比代码还长，维护成本过高
- **2-5 分钟**恰好是一个"原子操作"——足够小以至于不需要判断，足够大以至于有意义

### 4.3 完整代码 vs 描述性指令

```markdown
# ❌ 描述性指令（不要这样写）
Step 3: 添加 URL 验证逻辑

# ✅ 完整代码（应该这样写）
Step 3: Write minimal implementation
  function validateUrl(url: string): boolean {
    return /^https?:\/\/.+/.test(url);
  }
```

为什么要在计划中写完整代码？

1. **消除歧义**——"添加验证逻辑"可以有 100 种实现方式，完整代码只有一种
2. **降低执行者的决策负担**——它不需要"设计"，只需要"执行"
3. **让审查成为可能**——审查者可以对比计划中的代码和实际代码，检查是否一致

代价是计划文档会更长，但这个代价值得——因为计划阶段的修改成本远低于代码阶段。

### 4.4 文件结构先行

```markdown
Before defining tasks, map out which files will be created or modified
and what each one is responsible for. This is where decomposition decisions
get locked in.
```

这是一个容易被忽视但极其重要的设计决策。文件结构在任务分解之前确定，原因是：

1. **文件结构决定了任务边界**——每个任务应该产出独立的、可测试的变更
2. **防止任务之间的耦合**——如果两个任务修改同一个文件的同一部分，执行时会冲突
3. **为子代理提供清晰的工作范围**——"你负责这个文件"比"你负责这个功能"更明确

### 4.5 计划审查：为什么需要独立审查者？

```markdown
Dispatch a single plan-document-reviewer subagent with precisely crafted
review context — never your session history.
```

写计划的 Agent 和审查计划的 Agent 是分开的。这不是多此一举：

1. **写作者有盲点**——它知道自己"想表达什么"，所以看不到表达不清的地方
2. **审查者用"新鲜眼睛"看**——它只看到文档本身，不知道写作者的意图，所以能发现歧义
3. **"never your session history"**——防止审查者被写作过程中的讨论影响，保持客观

审查者的 prompt 中有一个关键的校准指令：

```markdown
Only flag issues that would cause real problems during implementation.
Approve unless there are serious gaps.
```

这防止了审查者过度挑剔——如果审查者对每个小问题都报 issue，审查循环就会无限进行下去。

---

## 五、Subagent Prompt 的设计拆解

这是整个体系中最精妙的部分——如何设计子代理的 prompt，让不同角色各司其职。

### 5.1 实现者 Prompt：鼓励提问 + 允许失败

实现者 prompt 中有两段不寻常的指令：

**鼓励提问：**
```markdown
If you have questions about the requirements, approach, dependencies,
or anything unclear — ask them now. Raise any concerns before starting work.

While you work: If you encounter something unexpected or unclear,
ask questions. It's always OK to pause and clarify.
Don't guess or make assumptions.
```

大多数 prompt 告诉 AI "去做这件事"。这个 prompt 反复强调"不确定就问"。为什么？

因为 AI 的默认行为是**永远不提问，自己猜**。它宁可猜错也不愿意显得"不知道"。这条指令用反复强调来对抗这个默认行为。

**允许失败：**
```markdown
It is always OK to stop and say "this is too hard for me."
Bad work is worse than no work. You will not be penalized for escalating.
```

这段话的设计意图是**降低 AI 硬撑的概率**。AI 默认会尝试完成任何任务，即使它做不好。明确告诉它"失败是可以的"，能让它在真正卡住时选择上报而不是产出垃圾代码。

### 5.2 规格审查者 Prompt：不信任原则

规格审查者的 prompt 中有一段非常有意思的指令：

```markdown
## CRITICAL: Do Not Trust the Report

The implementer finished suspiciously quickly. Their report may be incomplete,
inaccurate, or optimistic. You MUST verify everything independently.

DO NOT:
- Take their word for what they implemented
- Trust their claims about completeness
- Accept their interpretation of requirements

DO:
- Read the actual code they wrote
- Compare actual implementation to requirements line by line
```

这段 prompt 的设计技巧：

1. **"finished suspiciously quickly"**——用叙事手法激发审查者的怀疑心态，而不是干巴巴地说"要仔细检查"
2. **DO NOT / DO 对比列表**——明确告诉审查者什么行为是被禁止的，什么行为是被要求的
3. **"Read the actual code"**——强制审查者去看代码，而不是只看实现者的报告

设计原则：**审查者的 prompt 要激发怀疑，而不是信任。** 如果审查者默认信任实现者，审查就形同虚设。

### 5.3 代码质量审查者：与规格审查者的分工

为什么要分成两个审查者，而不是一个审查者同时检查规格合规和代码质量？

1. **关注点分离**——规格审查只关心"做对了吗"，质量审查只关心"做好了吗"。混在一起容易互相干扰
2. **顺序依赖**——如果规格都不对，讨论代码质量没有意义。先确认方向正确，再优化实现
3. **不同的审查心态**——规格审查是二元的（符合/不符合），质量审查是光谱的（好/更好/最好）

### 5.4 状态机设计：四种返回状态

```
DONE → 正常完成
DONE_WITH_CONCERNS → 完成但有顾虑
NEEDS_CONTEXT → 缺少信息
BLOCKED → 无法完成
```

这四种状态覆盖了所有可能的结果，而且每种状态都有明确的处理路径。这是一个**有限状态机**的设计：

- 没有模糊状态（不存在"大概完成了"）
- 每种状态都有明确的下一步动作
- 主控制器可以机械地根据状态决定下一步

如果只有 DONE 和 BLOCKED 两种状态，就会丢失重要信息：
- 实现者完成了但不确定对不对（DONE_WITH_CONCERNS）
- 实现者不是做不了，只是缺一个信息（NEEDS_CONTEXT）

---

## 六、贯穿体系的设计模式

回顾整个体系，可以提炼出几个反复出现的设计模式：

### 模式一：反默认行为设计

AI 有很多"默认行为"是有害的。好的 prompt 设计要识别这些默认行为，然后用明确的指令来对抗：

| AI 的默认行为 | 对抗指令 |
|-------------|---------|
| 直接给答案 | HARD-GATE：设计未批准前禁止写代码 |
| 不提问 | "Ask them now"、"It's always OK to pause and clarify" |
| 不承认失败 | "It is always OK to stop and say this is too hard" |
| 过度生成 | YAGNI、规格审查检查"多余功能" |
| 信任自己的输出 | "Do Not Trust the Report" |
| 跳过验证 | TDD 的"MANDATORY. Never skip." |

### 模式二：白名单优于黑名单

```markdown
# ❌ 黑名单（总有漏网之鱼）
不要调用 frontend-design、mcp-builder、code-generator...

# ✅ 白名单（密不透风）
The ONLY skill you invoke after brainstorming is writing-plans.
```

### 模式三：穷举 + 堵借口

当一条规则特别重要时，不只是陈述规则，还要：
1. 穷举所有可能的违反方式
2. 预判 AI 可能找的借口，提前堵死

```markdown
# 规则
Do NOT invoke any implementation skill, write any code, scaffold any project,
or take any implementation action

# 堵借口
This applies to EVERY project regardless of perceived simplicity.

# 反模式说明
Anti-Pattern: "This Is Too Simple To Need A Design"
```

### 模式四：角色隔离

同一个 AI 不应该既当运动员又当裁判。体系中的角色分离：

- 写计划的 ≠ 审查计划的
- 写代码的 ≠ 审查规格的 ≠ 审查质量的
- 协调者（主控制器）≠ 执行者（子代理）

每个角色有独立的 prompt，看到不同的信息，带着不同的"心态"工作。

### 模式五：渐进式信任

体系中没有任何一步是"无条件信任"的：

```
用户的想法 → brainstorming 验证 → spec 自审查 → 用户审查
计划 → 计划审查者验证
代码 → 自审查 → 规格审查 → 质量审查
```

每一层都在验证上一层的输出。信任是通过多层验证逐步建立的，而不是假设任何一步的输出是正确的。

---

## 七、如何为自己的场景设计类似体系

### Step 1：列出你的"执行者"的缺陷

不同的 AI 模型、不同的使用场景，缺陷不同。先列出来：

- 它最容易在哪里犯错？
- 它最喜欢跳过哪些步骤？
- 它最常见的"借口"是什么？

### Step 2：为每个缺陷设计对冲手段

每个缺陷对应一条规则或一个流程步骤。规则的强度要匹配缺陷的严重程度。

### Step 3：设计阶段和门禁

把流程拆成阶段，每个阶段有明确的入口条件和出口条件。门禁要用强写法（HARD-GATE、穷举禁止行为、堵借口）。

### Step 4：设计角色和 prompt

如果有子代理能力，为不同角色设计独立的 prompt。关键是：
- 实现者的 prompt 鼓励提问和上报
- 审查者的 prompt 激发怀疑和独立验证
- 每个角色只看到它需要的信息

### Step 5：设计状态和错误处理

定义清晰的状态机——每种可能的结果都有明确的处理路径。不要留模糊地带。

### Step 6：迭代

没有完美的第一版。观察 AI 在哪里绕过了你的规则，然后加强那里的门禁。观察 AI 在哪里被过度约束了，然后适当放松。

---

## 八、总结

设计 Spec-Driven Development 体系的本质是：**你在为一个能力很强但纪律很差的"员工"设计工作流程。**

它聪明，但会走捷径。
它高效，但会过度发挥。
它听话，但会找借口绕过规则。
它不会主动提问，但会在不确定时自己猜。

你的 prompt 体系就是它的"员工手册"——不是告诉它怎么思考（它已经会了），而是告诉它什么时候该停下来、什么时候该提问、什么时候该让别人检查它的工作。

好的 prompt 体系和好的管理一样：**不是限制能力，而是引导能力往正确的方向发挥。**
