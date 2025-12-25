# 第 8 章 Multi-Agent：从单体到群体协作

## 1. 开篇段落

在前面的章节中，我们构建的智能体大多是“单兵作战”：一个大模型核心，配备多个工具（Tools）和一块记忆（Memory）。这种 **单体智能体（Monolithic Agent）** 在处理线性任务（如“查询天气并穿衣建议”）时表现出色。然而，当我们面对真实世界的复杂性——例如“撰写一份包含市场调研、竞品图片分析和财务预测的商业计划书”时，单体智能体往往会撞上“能力墙”。

它会面临上下文窗口的拥挤（Context Crowding）、指令遵循能力的退化（Instruction Dilution），以及“全能悖”（要求一个模型同时具备顶级程序员、顶尖会计和顶级作家的微操能力）。

本章我们将视角升级，从“个体微观”转向“组织宏观”。我们将探讨 **多智能体系统（Multi-Agent Systems, MAS）** 的设计哲学。你将不再仅仅是一个 Prompt 工程师，而是一个**组织架构师**。我们需要学习如何根据任务形态选择 **层级（Hierarchy）、群组（Swarm）或流水线（Pipeline）** 结构，并制定严谨的 **通信协议（SOP）**，让多个专家模型像一支训练有素的足球队一样协作，而不是一群争抢球权的幼儿园小朋友。

---

## 2. 文字论述

### 2.1 为什么要使用 Multi-Agent？(The Trade-off)

在工程实践中，盲目引入多智能体是初学者常犯的错误。MAS 本质上是一场 **“计算成本（Cost）与延迟（Latency）”换取“质量（Quality）与鲁棒性（Robustness）”** 的交易。

* **单体智能体 (Single Agent)**：
* *优势*：全上下可见（Global Context），低延迟，一次推理完成，调试路径单一。
* *劣势*：随着 System Prompt 变长，模型对尾部指令的遵循度下降；容易在长链条推理中“迷失”初始目标。


* **多智能体系统 (MAS)**：
* *优势*：
* **关注点分离 (Separation of Concerns)**：每个 Agent 的 System Prompt 极短且聚焦（例如“你只负责审查 Python 代码安全性”），性能达到局部最优。
* **模块化迭代**：可以单独升级“视觉专家”的模型版本，而不影响“文案专家”。
* **上下文隔离**：避免了无关信息干扰推理（代码专家不需要知道用户的情绪发泄）。


* *劣势*：系统复杂度指数级上升；通信开销巨大（JSON 序列化/反序列化）；可能陷入死循环。



**Rule of Thumb（经验法则）：** 仅当任务复杂度导致单体模型的 Prompt 超过 2k token 仍无法稳定遵循指令，或者任务明显需要两种互斥的思维模式（如“发散创意的作家”与“严谨刻板的审计员”）时，才引入 Multi-Agent。

### 2.2 核心组织架构图谱 (Organizational Patterns)

我们将多智能体架构归纳为四种经典拓扑结构。

#### 2.2.1 经理-工人模式 (Manager-Worker / Hierarchical)

这是最符合人类公司制度的结构。一个高智商的“大脑”负责规划，多个专用“手脚”负责执行。

```text
                  +---------------------+
                  |     User Query      |
                  +----------+----------+
                             |
                             v
                  +---------------------+
                  |    Manager Agent    |  <-- (Planner / Orchestrator)
                  |  "The Boss / PM"    |
                  +----------+----------+
                             | 1. Decompose
        +--------------------+---------------------+
        | 2. Assign          |                     |
        v                    v                     v
+----------------+   +----------------+    +----------------+
| Research Agent |   |  Coding Agent  |    | Review Agent   |
| (Web/Doc Tool) |   | (Python Tool)  |    | (Policy Check) |
+-------+--------+   +--------+-------+    +--------+-------+
        |                    |                      |
        +--------------------+----------------------+
                             | 3. Aggregate
                             v
                      [ Final Output ]

```

* **工作流**：Manager 接收任务 -> 拆解为子任务 (Sub-goals) -> 顺序或并行分发 -> Worker 执行并返回 -> Manager 汇总。
* **多模态应用**：Manager 决定这张图片是发给 `OCR Agent` 提取文字，还是发给 `Vision Agent` 描述场景。

#### 2.2.2 路由器模式 (Router / Classifier)

主要用于意图识别和分流。与 Manager 不同，Router 往往不参与后续的汇总，它只是一个“前台导购”。

```text
User Input --> [ Router / Intent Classifier ]
                       |
           +-----------+-----------+
           |           |           |
           v           v           v
      [Refund      [Tech       [Sales
       Agent]       Support]    Agent]
         |             |           |
         +-------------+-----------+---> User

```

* **关键点**：Router 应该非常轻量（甚至可以是微调过的小模型或 BERT），它的目标是**快速**将上下文甩给正确的专家。

#### 2.2.3 辩论与自我修正模式 (Debate / Critic / Reflection)

为了解决 LLM 的幻觉（Hallucination）和逻辑漏洞，引入对抗性角色。这是一种“左脚踩右脚上天”的质量提升手段。

```text
      Phase 1: Draft              Phase 2: Critique            Phase 3: Refine
+---------------------+       +----------------------+       +---------------------+
|    Solver Agent     | ----> |     Critic Agent     | ----> |    Solver Agent     |
| (Generates Answer)  |       | (Finds flaws/bugs)   |       | (Fixes issues)      |
+---------------------+       +----------------------+       +---------------------+
           ^                                                            |
           |____________________(Loop if needed)________________________|

```

* **适用场景**：复杂数学推理、代码生成、高风险合规检查。
* **变体**：**多方辩论 (Multi-party Debate)**，引入三个不同观点的 Agent 互相打分，取加权一致性结果。

#### 2.2.4 黑板模式 (Blackboard / Shared State)

这是最复杂但也最强大的模式，源自传统人工智能。Agent 之间不直接对话，而是读写一个**共享的、结构化的状态池（Blackboard）**。

```text
       +-----------------------------------------------+
       |             SHARED BLACKBOARD                 |
       |  {                                            |
       |    "user_intent": "drive_home",               |
       |    "obstacles": [{"type": "car", "pos":...}], | <--- Updated by Perception
       |    "path_plan": [ ... ],                      | <--- Updated by Planner
       |    "status": "waiting_traffic_light"          |
       |  }                                            |
       +-----------------------------------------------+
           ^           ^             ^            ^
           |           |             |            |
     [Perception]  [Planner]     [Control]    [Safety]
       Agent         Agent         Agent       Monitor

```

* **机制**：发布-订阅（Pub/Sub）。Agent 监控黑板上的特定字段，一旦满足触发条件（Pre-condition），就执行操作并更新黑板。
* **适用场景**：非线性任务、异步协作、具身智能（VLA）、复杂的侦探推理游戏。

### 2.3 协作协议与多模态数据流 (Protocols & Data Flow)

架构定了，Agent 之间怎么“说话”？这决定了系统的稳定性。

1. **Handoff Protocol (移交协议)**：
* **热移交 (Full Context)**：将整个对话历史 `messages[]` 传给下一个 Agent。
* *优点*：无信息损失。
* *缺点*：Token 爆炸，后面的 Agent 容易被前面的废话干扰。


* **冷移交 (Summary Handoff)**：上游 Agent 必须在结束前生成一个结构化的 `Handoff Packet`。
* *Packets 示例*：`{"goal": "...", "completed_steps": [...], "key_evidence": "...", "pending_questions": [...]}`。


* **多模态引用传递**：不要传递图片的 Base64！**只传递图片的 ID 或 URI**。所有 Agent 共享一个 `Media Service`，只有在需要看图时才去加载。


2. **仲裁协议 (Arbitration)**：
* 当两个 Agent 意见不一致（如辩论模式），谁说了算？
* **Supervisor Strategy**：设置一个拥有最高权限的 Supervisor，它不生成内容，只负责打分和叫停。
* **Voting Strategy**：少数服从多数（需 3+ Agent）。



### 2.4 资源调度 (Resource Scheduling)

多智能体不仅仅是软件逻辑，也是分布式系统。

* **并行度 (Parallelism)**：Manager 可以同时调用 `Search_Agent` 和 `Doc_Reading_Agent`。这需要你的框架支持 `async/await` 并发控制。
* **优先级 (Priority)**：在资源（如 API Rate Limit）受限时，优先保障 `Safety_Agent`（安全守门员）的运行，其次才是 `Creative_Agent`。

---

## 3. 本章小结

1. **架构即智能**：当单体模型智力封顶时，合理的组织架构（Manager-Worker, Debate, Blackboard）可以涌现出更高的系统智能。
2. **专业分工**：Prompt Engineering 的终极形态是给每个 Agent 写一份极简、极专的 Job Description（职位描述）。
3. **协议重于模型**：定义清晰的输入输出 Schema（JSON）、移交数据包（Packet）和终止条件（Termination），比单纯调优模型参数更重要。
4. **多模态通信**：在 Agent 间传递多模态内容时，使用“引用（Reference）”而非“值（Value）”，避免上下文爆炸。

---

## 4. 练习题

### 基础题（帮助熟悉材料）

#### 练习 8.1：架构匹配

**题目**：请为以下三个场景分别推荐最合适的 Multi-Agent 架构，并简理由。

1. **场景 A**：一个智能客服，需要根据用户是询问“退货政策”、“技术故障”还是“VIP服务”来转接不同的知识库。
2. **场景 B**：一个自动驾驶系统的决策层，感知模块实时更新路况，导航模块规划路线，控制模块调整方向盘，它们需要异步协作。
3. **场景 C**：一个辅助编程工具，用户输入需求，系统先写代码，然后运行单元测试，如果报错则根据报错信息修改代码，直到测试通过。

<details>
<summary>点击查看参考答案</summary>

* **场景 A：路由器模式 (Router)**。理由：任务之间是正交的（互斥），只需一次分类即可分流，无需后续汇总。
* **场景 B：黑板模式 (Blackboard)**。理由：模块间存在复杂的依赖和异步更新（感知变了，规划就得变），共享状态池是最优解。
* **场景 C：循环式的经理-工人模式 或 辩论/自修模式**。理由：这是一个典型的 `Generate -> Test -> Fix` 闭环，需要迭代直到满足终止条件（测试通过）。

</details>

#### 练习 8.2：Handoff Packet 设计

**题目**：你有一个负责“深度网页搜索”的 Agent A，通过搜索找到了关于“2024年东京奥运会收视率”的 5 个关键事实。现在要移交给负责“撰写新闻稿”的 Agent B。
请设计一个 JSON 格式的 `Handoff Packet`，既能包含必要信息，又避免把成百上千行的搜索原始 HTML 扔给 Agent B。

<details>
<summary>点击查看提示与答案</summary>

**提示**：Agent B 不需要知道你是怎么搜的（搜索过程），只需要知道你搜到了什么（结论）以及来源（用于引用）。

**参考 Schema**：

```json
{
  "handoff_reason": "Search complete, ready for drafting.",
  "task_context": "Write a news brief about 2024 Tokyo Olympics viewership.",
  "findings": [
    {
      "fact": "Global viewership dropped by 15% compared to Rio.",
      "source_url": "https://news.com/report1",
      "confidence": "High"
    },
    {
      "fact": "Streaming minutes increased by 300%.",
      "source_url": "https://official.olympics/stats",
      "confidence": "Medium"
    }
  ],
  "image_candidates": ["img_ref_001", "img_ref_002"], // 图片引用ID
  "constraints": "Keep it under 200 words."
}

```

</details>

### 挑战题（包括开放性思考题）

#### 练习 8.3：死锁破解 (Breaking the Loop)

**题目**：在“代码编写 (Coder)”和“代码审查 (Reviewer)”的二人协作中，经常出现如下死锁：

* Coder: "这是修改后的代码 V5。"
* Reviewer: "变量命名风格还是不完美，请修改。"
* Coder: "修改了，这是 V6。"
* Reviewer: "这个函数的注释不够详细，请修改。"
...（无限循环，由于 Reviewer 过于吹毛求疵，导致任务无法收敛）

请设计一种**基于协议的机制**来打破这种死锁，不仅仅是限制最大轮次。

<details>
<summary>点击查看参考答案</summary>

**解题思路**：

1. **预算衰减 (Budget Decay)**：给 Reviewer 一个“挑刺预算”。每提出一个 issue，扣除一点预算。预算耗尽后，只能强制通过（PASS）。
2. **分级验收 (Tiered Acceptance)**：定义 `Critical` (必须修) 和 `Nice-to-have` (建议修) 错误。如果在 N 轮后仍未通过，Manager 介入，强制 Reviewer 忽略所有 `Nice-to-have` 问题，只检查 `Critical`。
3. **仲裁者引入 (Supervisor)**：当循环超过 3 次，引入第三个 Agent（Supervisor），让它评估 Reviewer 的修改意见是否“微不足道（trivial）”。如果是，Supervisor 强行终止循环并采纳当前版本。

</details>

#### 练习 8.4：多模态“传话筒”游戏

**题目**：设计一个系统，用户上传一张复杂的UI设计图。

* Agent A (Vision Expert) 负责看图并描述布局。
* Agent B (Frontend Expert) 负责根据描述写 HTML/CSS。
* **挑战**：Agent A 的文字描述往往不够精确（例如“左边有个按钮” vs “左上角 20px 处有一个 100x50 的蓝色主按钮”）。
请设计一个**基于反馈的协作流程**，让 Agent B 能主动向 Agent A 提问，从而提高还原度。

<details>
<summary>点击查看参考答案</summary>

**流程设计**：

1. **Initial Pass**: A 生成初步描述 -> B 生成 v1 代码。
2. **Rendering**: 系统将 B 的 v1 代码渲染为截图 (Screenshot_v1)。
3. **Visual Comparison (关键步骤)**:
* 将 `Original_Image` 和 `Screenshot_v1` 并排发给一个 **Visual Critic Agent** (或复用 Agent A)。
* Critic 比较两图差异，输出结构化差异报告：`{"diff_area": "top-left", "issue": "Button color is red, expected blue"}`。


4. **Refinement**: B 接收差异报告，修改代码生成 v2。
5. **Loop**: 重复直到差异低于阈值。
*核心在于引入了“渲染+视觉比对”闭环，而不是单纯依赖语言描述。*

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 "The Politeness Loop" (礼貌死循环)

* **现象**：两个 Agent 在任务完成后始互相感谢。
* A: "Here is the data." -> B: "Thanks!" -> A: "You're welcome!" -> B: "Have a nice day!"


* **后果**：浪费 Token，甚至导致流程挂起（如果是基于关键词终止的）。
* **调试技巧**：在 System Prompt 中加入强指令：“**禁止闲聊。禁止礼貌用语。任务完成后仅输出 `<TERMINATE>`。**”

### 5.2 羊群效应 (Herding)

* **现象**：在多方辩论或投票中，如果 Agent B 看到了 Agent A 的回答，它倾向于附和 A，而不是独立思考。
* **调试技巧**：**盲测 (Blind Box)**。在第一轮投票时，不要让 Agent 看到彼此的回答，只给它们看原始问题。收集完所有人的独立回答后，再进行第二轮“公开讨论”。

### 5.3 格式战争 (Format War)

* **现象**：Agent A 输出 Markdown 表格，Agent B 期待 CSV，Agent C 期待 JSON。解析器在中间频繁报错。
* **调试技巧**：
* **统一 Schema**：在 System Prompt 中强制定义交互格式（推荐 JSON）。
* **容错解析**：使用 `json_repair` 等库处理 LLM 输出的非标准 JSON。
* **格式转换器**：在 Agent 之间加一层轻量的 Middleware 负责格式清洗。



### 5.4 幽灵调用 (Phantom Tool Calls)

* **现象**：Manager 以为 Worker 已经执行了工具（因为 Worker 回复说“我做完了”），但实际上 Worker 只是产生了幻觉，根本没调 API。
* **调试技巧**：Manager 不应只听 Worker 的**自然语言回复**，必须检查 Worker 的 **Tool Execution Trace** 或要求 Worker 返回具体的 **Evidence ID**。不要相信“I have checked the database”，要相信 `{"tool": "db_query", "status": "success", "result_count": 5}`。
