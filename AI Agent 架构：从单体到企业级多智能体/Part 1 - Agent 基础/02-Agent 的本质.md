## Agent 的本质
Agent 是能自主完成任务的 AI 系统，核心在于"自己做决定"而不是"你说一句它动一句"。

### Agent到底是什么
一句话定义：Agent 就是一个能自己干活的 AI。

工程化定义：给它一个目标、一个工具箱和一套边界（预算 / 权限 / 审批 / 沙箱），它在循环里推进任务，直到完成或停下。

### Agent 的四个核心组件

```
Agent = 大脑 + 手脚 + 记忆 + 主见
       (LLM)  (Tools) (Memory) (Autonomy)
```

1. 大脑（LLM）：负责思考，决定下一步干什么。这是 Agent 的核心引擎。没有 LLM，Agent 就是个死板的脚本。
2. 手脚（Tools）：负责执行，比如搜索网页、读写文件、调用 API、操作浏览器。LLM 只会"想"，Tools 让它能"做"。
3. 记忆（Memory）：负责记住之前发生了什么。短期记忆是当前对话的上下文，长期记忆是跨会话的知识积累。没有记忆，Agent 每次都从零开始，效率极低。
4. 主见（Autonomy）：这个最关键——它得自己做决定，不是你说一步它动一步。自主性是 Agent 和普通 Chatbot 的根本区别。


在生产系统里，你还需要一层“护栏（Guardrails）”。它不是 Agent 的组件，但它决定 Agent 能不能上线：预算、权限、审批、审计、沙箱。

没有护栏的 Agent，迟早从“自己干活”变成“自己闯祸”。

### Agent 与传统软件的区别、挑战

区别：传统软件是确定性的；Agent 是概率性的
```
传统软件:  输入 → 固定逻辑 → 输出
Agent:    输入 → LLM思考 → 工具调用 → 观察结果 → 继续思考 → ... → 输出
```

挑战：Agent 带来了灵活性，也带来了不确定性。生产环境中，如何控制这种不确定性，是 Agent 系统设计的核心挑战之一。

### Agent 智能体做任务的流程
1. 第一步：任务分解
2. 第二步：逐个执行
3. 第三步：自我检查
4. 第四步：综合输出

这就是 Agent。


### Agent 的自主性等级

| 等级   | 名称           | 你说     | 它做                   | 典型例子           |
| :----- | :------------- | :------- | :--------------------- | :----------------- |
| **L0** | Chatbot        | 问一句   | 答一句                 | ChatGPT 基础对话   |
| **L1** | Tool Agent     | 要查天气 | 调 API 返回结果        | GPTs 的 Actions    |
| **L2** | ReAct Agent    | 复杂问题 | 思考→行动→观察，循环   | LangChain ReAct    |
| **L3** | Planning Agent | 大任务   | 先拆解计划，再逐个执行 | 本书重点           |
| **L4** | Multi-Agent    | 更大任务 | 多个 Agent 分工协作    | Shannon Supervisor |
| **L5** | Autonomous     | 模糊目标 | 长期自主运行，自我迭代 | Claude Code, Manus |

### Agent 各等级详解

**L0 - Chatbot**：纯对话，没有工具调用能力。你问它天气，它只能说"我无法获取实时信息"。

**L1 - Tool Agent**：能调用工具，但只是简单的"你让我查，我就查"。没有多步推理。

**L2 - ReAct Agent**：能进行多轮"思考-行动-观察"循环。这是 Agent 的入门级形态，也是下一章的重点。

**L3 - Planning Agent**：能把复杂任务拆解成子任务，制定计划，然后执行。比 L2 多了"先想后做"的能力。

**L4 - Multi-Agent**：多个专业化的 Agent 协作。比如一个负责搜索，一个负责分析，一个负责写作。这是企业级应用的主流形态。

**L5 - Autonomous**：能长期自主运行，根据环境反馈调整策略，甚至自我改进。目前还没有真正可靠的 L5 Agent，Claude Code 和 Manus 在这个方向探索，但还不成熟。



目前大多数人用的是 L0-L1。本项目主要教你怎么构建 **L2-L4**。

L5？说实话，**现在还没有真正可靠的 L5 Agent**。谁说有，你可以持保留态度。



### Agent 能做什么，不能做什么

我见过太多人把 Agent 捧得太高，好像它能干任何事。

**不是的。**

#### Agent 擅长的场景

| 场景特征       | 例子                   | 为什么适合         |
| :------------- | :--------------------- | :----------------- |
| **目标明确**   | "帮我总结这篇文章"     | 有清晰的成功标准   |
| **步骤可拆解** | "按这个流程处理数据"   | 可以分解成子任务   |
| **结果可验证** | "代码能跑就行"         | 能判断是否完成     |
| **信息可获取** | "查一下这个公司的信息" | 有工具可以获取数据 |
| **重复性高**   | "每天早上给我发简报"   | 自动化价值高       |

#### Agent 不擅长的场景

| 场景特征         | 例子                              | 为什么不适合           |
| :--------------- | :-------------------------------- | :--------------------- |
| **开放性创意**   | "帮我想个颠覆性的商业模式"        | 没有明确标准，难以迭代 |
| **主观判断**     | "这个设计好不好看"                | 需要人类审美和价值观   |
| **复杂人际**     | "帮我搞定这个客户"                | 涉及情商、关系、语境   |
| **高风险决策**   | "帮我决定要不要投资"              | 责任归属问题           |
| **不可逆副作用** | "帮我给全公司群发邮件/删掉生产库" | 一旦做错，损失不可逆   |
| **实时物理操作** | "帮我做饭"                        | 需要物理机器人         |

#### 选对场景是关键

选对场景，Agent 能帮你省很多时间。选错场景，它就是个**烧 Token 的机器**。

很多事情不是“不能做”，而是“必须加确认点才敢做”：付款、发布、删除、群发邮件……默认都应该是人类确认，Agent 执行。

一个简单的判断方法：

> 如果这个任务你交给一个实习生，你能用文字清楚地告诉他怎么做，那大概率适合 Agent。 如果你自己都说不清楚"怎么算做好了"，那 Agent 也搞不定。

### Agent 的技术演进
1. 2022 年之前：规则驱动：如早期的"智能助手"（如 Siri、Alexa）

2. 2023 年：LLM 成为大脑

3. 2023-2024 年：ReAct 与 Function Calling

4. 2024-2025 年：多 Agent 与生产化
   
   单 Agent 能力有限，多 Agent 协作成为主流。同时，企业开始关注：

   - 成本控制（Token 预算）
   - 安全性（沙箱执行）
   - 可靠性（持久化、重试）
   - 可观测性（监控、追踪）
   - 构建 生产级 Agent 系统。

## 智能研究助手 (Research Agent) 架构

Orchestrator层

- 任务路由与编排模式选择
- Token 预算分配与追踪
- 多 Agent 协调
- 结果综合

Agent Core

- 单次 Agent 执行 （LLM + 工具调用）
- WASI 沙箱执行 （高风险代码）
- 工具调用与隔离
- 资源限制与超时

LLM Service

- 多 Provider LLM 调用
- 工具定义与执行
- 向量存储与检索
- Prompt 管理

| 层级         | 语言   | 职责       | 为什么选这个语言                 |
| :----------- | :----- | :--------- | :------------------------------- |
| Orchestrator | Go     | 编排、调度 | 并发强，适合协调多个 Agent       |
| Agent Core   | Rust   | 执行、隔离 | 性能好，内存安全，适合边界与沙箱 |
| LLM Service  | Python | 模型与工具 | 生态丰富，SDK 与工具链齐全       |



## 智能研究助手 (Research Agent) 覆盖的Agent等级

| 你想做的事          | Shannon 用什么模式 | 对应等级 |
| :------------------ | :----------------- | :------- |
| 简单问答 + 工具调用 | SimpleTask         | L1       |
| 思考-行动循环       | ReAct              | L2       |
| 复杂任务拆解        | DAG                | L3       |
| 多 Agent 协作       | Supervisor         | L4       |


## "Router/Strategy/Pattern" 三层怎么分工
### 第一层：Router（路由器）—— "做什么类型的任务？"
职责：只看一次query，决定大方向

- 复杂度评分：< 0.3 → SimpleTask，≥ 0.3 → 复杂任务

- 策略选择：根据认知策略路由到对应 Workflow

类比：像医院分诊台——不看病，只判断你去哪个科室


### 第二层：Strategy（策略）—— "用哪些模式组合？"

**职责**：组合 Patterns，形成完整业务流程。每个 Strategy 就是一种**固定的模式组合**：

| Strategy                | 模式组合                                | 适用场景               |
| ----------------------- | --------------------------------------- | ---------------------- |
| **DAGWorkflow**         | Parallel/Sequential/Hybrid + Reflection | 多步骤、有依赖的任务   |
| **ReactWorkflow**       | Reason-Act-Observe Loop                 | 需要工具反馈的迭代问题 |
| **ResearchWorkflow**    | React 或 Parallel + Reflection          | 信息收集、对比研究     |
| **ExploratoryWorkflow** | Tree-of-Thoughts + Debate + Reflection  | 开放性问题、未知解空间 |
| **ScientificWorkflow**  | CoT + Debate + ToT + Reflection         | 假设验证、系统分析     |

```go
// Scientific Workflow 的组合示例
1. Chain-of-Thought → 生成假设
2. Debate → 测试竞争假设  
3. Tree-of-Thoughts → 探索含义
4. Reflection → 最终质量综合
```

**类比**：像主刀医生——制定手术方案，具体缝合由护士执行


### 第三层：Pattern（模式）—— "具体怎么执行？"

**职责**：最小的可复用执行单元，分两类：

**执行模式（Execution）**：

- **Parallel**：并发执行，信号量控制（最多5个并发Agent）
- **Sequential**：顺序执行，结果向下传递
- **Hybrid**：依赖图执行，拓扑排序

**推理模式（Reasoning）**：

| Pattern              | 核心逻辑                    |
| -------------------- | --------------------------- |
| **React**            | Reason → Act → Observe 循环 |
| **Reflection**       | 自我评估，质量改进          |
| **Chain-of-Thought** | 逐步推理，带置信度          |
| **Debate**           | 多Agent辩论，探索视角       |
| **Tree-of-Thoughts** | 系统探索，分支与剪枝        |

**类比**：像手术器械——每种工具有固定用途，自由组合



### 核心设计思想

1. **Router 唯一**：入口固定，只有一个路由器
2. **Strategy 有限**：策略数量可控（5-7种经典组合）
3. **Pattern 可复用**：底层模式被不同策略共享，避免代码重复

### 完整例子：ScientificWorkflow 执行流程

**Query**: "验证缓存是否能将数据库查询性能提升 50%"

------

#### 第一层：Router（分诊台）



```
输入 Query
    ↓
复杂度分析: 0.75 (≥ 0.3 → 复杂)
    ↓
认知策略: "scientific" → ScientificWorkflow
```

------

#### 第二层：Strategy（手术方案）

ScientificWorkflow 定义了**4阶段固定流程**：



```go
// 科学工作流的模式组合
Phase 1: ChainOfThought  → 生成假设
Phase 2: Debate           → 测试假设
Phase 3: TreeOfThoughts   → 探索含义
Phase 4: Reflection       → 最终质量检查
```

------

#### 第三层：Pattern（手术器械执行）

##### **Phase 1: Chain-of-Thought** —— 生成假设



```go
cotConfig := patterns.ChainOfThoughtConfig{
    MaxSteps:        3,           // 生成3个假设
    RequireExplanation: true,
    ShowIntermediateSteps: true,
}
```

**执行**:



```
Prompt: "为 '缓存是否能提升50%性能' 生成3个可测试假设"

→ Step 1: 关键因素是什么？
  - 缓存命中率
  - 数据局部性
  - 过期策略
  
→ Step 2: 不同解释？
  - H1: 缓存命中率 > 80% 时，显著提升
  - H2: 缓存对简单查询更有效
  - H3: 缓存对OLAP无效（分析型查询）
  
→ Step 3: 如何验证？
  - A/B 测试，监控 P50/P95 延迟
```

输出: `["缓存命中率>80%时延迟降低50%", ...]`

------

##### **Phase 2: Debate** —— 多智能体辩论



```go
debateConfig := patterns.DebateConfig{
    NumDebaters:   len(hypotheses),  // 3个辩手
    MaxRounds:     3,                 // 3轮辩论
    VotingEnabled: true,              // 投票机制
}
```

**执行**:



```
Query: "测试这些竞争假设..."

Round 1: 初始立场
  - H1倡导者: "80%命中率场景下，Redis查询<1ms"
  - H2倡导者: "简单SELECT查询缓存收益更大"
  - H3倡导者: "分析查询跨表扫描，缓存无效"
  
Round 2: 反驳
  - H1 vs H3: "即使OLAP，热点数据仍可缓存"
  - H2 挑战 H1: "简单查询本身已很快，缓存收益有限"
  
Round 3: 投票
  - H1: 2票 (胜出)
  - H2: 1票
  - H3: 0票
```

输出: `WinningArgument: "H1缓存命中率是关键因素"`

------

##### **Phase 3: Tree-of-Thoughts** —— 探索含义



```go
totConfig := patterns.TreeOfThoughtsConfig{
    MaxDepth:        3,
    BranchingFactor:  2,
    BacktrackEnabled: false,
}
```

**执行**:



```
Winning: "缓存命中率>80%时延迟降低50%"
    ↓
[实现层] ────────────── [策略层]
  ↓                      ↓
[Redis] [Memcached]   [LRU] [TTL]
  ↓         ↓           ↓      ↓
(分数)    (分数)      (分数)  (分数)
    ↓
最佳路径: Redis + LRU + TTL(5min)
```

输出: `BestSolution: "使用Redis LRU策略，TTL 5分钟，监控命中率"`

------

##### **Phase 4: Reflection** —— 质量审查



```go
reflectionConfig := patterns.ReflectionConfig{
    MaxRetries:          2,
    ConfidenceThreshold: 0.85,
    Criteria:            []string{
        "scientific_rigor",
        "evidence_quality",
        "logical_consistency",
    },
}
```

**执行**:



```
初版报告 → 评估(0.72)
  ↓
问题: 缺乏具体基准数据
  ↓
改进版报告 → 评估(0.88) ✓
  ↓
通过质量门槛
```

------

#### 最终输出



```markdown
# Scientific Investigation Report

**Research Question:** 验证缓存是否能将数据库查询性能提升50%

## Hypotheses Tested
1. 缓存命中率>80%时，延迟降低50%
2. 简单查询缓存收益更大
3. OLAP分析查询缓存无效

## Investigation Results
**Winning Hypothesis:** H1 - 缓存命中率是关键
**Consensus Reached:** true (2/3 投票)
**Debate Rounds:** 3

## Implications
- 实施 Redis LRU 缓存策略
- 设置 TTL = 5分钟
- 监控指标: hit_rate, p50/p95 latency

## Confidence Assessment
**Overall Confidence:** 88%
**Exploration Depth:** 3 levels
**Total Thoughts Explored:** 10
```

------

#### 三层职责总结

| 层           | 职责                                  | 类比                        |
| ------------ | ------------------------------------- | --------------------------- |
| **Router**   | 决定用 ScientificWorkflow             | 挂号台分诊                  |
| **Strategy** | 定义 CoT→Debate→ToT→Reflection 四阶段 | 医生制定手术方案            |
| **Pattern**  | 具体执行每个推理/执行模式             | 护士执行缝合/止血等具体操作 |




## 1.7 常见误区

### 误区一：Agent = ChatGPT + 插件

不完全对。插件只是"工具"，Agent 的核心是**自主决策循环**。有工具不代表是 Agent，能自己决定"什么时候用什么工具"才是。

### 误区二：Agent 能替代人类

不能，至少现在不能。Agent 是**增强工具**，不是替代品。它能帮你处理重复性、结构化的任务，但需要人类设定目标、监督过程、验收结果。

### 误区三：Agent 越自主越好

不一定。自主性越高，不确定性越大。生产环境中，往往需要在"自主性"和"可控性"之间找平衡。完全自主的 Agent 可能失控；完全受控的 Agent 又失去了意义。

### 误区四：用最强的模型就能做好 Agent

模型能力只是基础。Agent 系统的质量取决于：

- 工具设计是否合理
- Prompt 是否清晰
- 错误处理是否完善
- 架构是否支持扩展

用 GPT-4 跑一个糟糕的 Prompt，不如用 GPT-3.5 跑一个精心设计的系统。



## 1.8 本章要点回顾

1. **Agent 定义**：能自主完成任务的 AI 系统，核心是"自己做决定"
2. **四个组件**：大脑（LLM）+ 手脚（Tools）+ 记忆（Memory）+ 主见（Autonomy）
3. **自主性等级**：L0-L5，本书聚焦 L2-L4
4. **适用场景**：目标明确、步骤可拆解、结果可验证的任务
5. **Shannon 定位**：三层架构的生产级多 Agent 系统

## 练习

1. 把“帮我订机票”改写成一个可执行的 Agent 目标：必须包含**成功标准**、**需要你确认的步骤**、以及“失败了怎么办”

### 可执行的订机票 Agent 目标

------

#### ✅ 成功标准

1. **航班匹配**：出发地/目的地/日期与用户要求一致
2. **价格有效**：显示的票价可预订（未被售罄或涨价）
3. **信息完整**：包含航班号、起降时间、机场、航空公司、总价
4. **用户确认**：用户明确说"确认预订"后才真正下单
5. **失败感知**：当无结果时，主动告知用户而不是默默退出

------

#### 🔍 需要你（Agent）确认的步骤

| 步骤              | 确认内容                                  | 示例                        |
| ----------------- | ----------------------------------------- | --------------------------- |
| **出发地/目的地** | 出发城市和目的城市                        | "从北京去上海还是..."       |
| **出行日期**      | 出发日期，是否往返                        | "5月10日去，5月15日回？"    |
| **乘客信息**      | 乘客姓名、证件号（订票必需）              |                             |
| **偏好筛选**      | 航司偏好 / 直飞优先 / 价格优先 / 时间偏好 | "有东航直飞偏好？"          |
| **舱位偏好**      | 经济舱/商务舱/头等舱                      |                             |
| **价格确认**      | 列出候选后等待用户确认                    | "最便宜那班 680 元，确认？" |

------

#### ⚠️ 失败了怎么办

| 失败场景         | 处理方式                                    |
| ---------------- | ------------------------------------------- |
| **无匹配航班**   | 扩大日期范围 ±1~3天，重新搜索               |
| **航班售罄**     | 告知用户并推荐备选日期或相邻机场            |
| **价格获取失败** | 重试 1 次（网络波动），仍失败则返回错误说明 |
| **用户中断**     | 保存当前候选列表，等待用户继续              |
| **证件号缺失**   | 明确提示这是预订必需信息                    |

------

#### 📋 完整 Goal 表述示例

```
目标：帮用户预订一张机票

前置确认：
  - 出发地：___ 目的地：___ 日期：___
  - 是否往返：□单程 □往返（返程日期：___）
  - 乘客姓名：___ 证件号：___
  - 偏好：□直飞 □价格优先 □指定航司：___

执行：
  1. 查询符合条件的航班列表（≤3个候选）
  2. 展示每个候选的：航班号、起降时间、机场、总价
  3. 等待用户选择并确认"预订此航班"
  4. 执行下单操作

失败兜底：
  - 无结果 → 扩大日期/机场范围重试
  - 下单失败 → 提示失败原因，等待用户决策
```

------

**核心原则**：目标必须从模糊的"帮我订机票"转化为「**有边界的前提条件 + 明确的成功定义 + 已知失败路径**」，这样 Agent 才能正确执行而不是随意发挥。



2. 用一句话写出你自己的 Agent 定义，然后在 Shannon 里为这句话找 3 个“证据文件”（比如路由、执行、护栏）

   

   ## 一句话 Agent 定义

   ## Agent 定义 & 3 个证据文件

   > **我的 Agent 定义**：*感知输入 → 基于策略路由决策 → 执行工具 + 预算护栏约束*

   ------

   ### 1. 路由/决策 — [orchestrator_router.go](vscode-webview://1n6c3a2na5liuo5cip8v1v7b9f02om4d3206krsjfh5rqogibnst/go/orchestrator/internal/workflows/orchestrator_router.go)

   `OrchestratorWorkflow` 基于复杂度（`complexityScore`）和策略标志（`force_swarm`、`force_research`）在 SimpleTask / DAG / Research / Swarm 多条路径间**决策**。

   行 760–890：复杂度路由 switch；行 273–322：策略早期路由。

   ------

   ### 2. 执行 — [agent.py](vscode-webview://1n6c3a2na5liuo5cip8v1v7b9f02om4d3206krsjfh5rqogibnst/python/llm-service/llm_service/api/agent.py)

   Agent 执行循环：解析历史（感知）→ 调用 `generate_tool_digest` → 构建 `build_interpretation_messages` → LLM 决策 → 工具执行。

   行 18–50：附件解析；行 348–518：工具结果聚合与上下文注入。

   ------

   ### 3. 护栏 — [middleware_budget.go](vscode-webview://1n6c3a2na5liuo5cip8v1v7b9f02om4d3206krsjfh5rqogibnst/go/orchestrator/internal/workflows/middleware_budget.go)

   `BudgetPreflight` 在任务执行前调用 `CheckTokenBudgetWithBackpressureActivity` 检查预算；行 49–82 实现 provider 级别速率控制；行 88–119 估算 Token 需求。

   行 20–47：预算预检；行 88–90：每 Agent Token 预算注解。



