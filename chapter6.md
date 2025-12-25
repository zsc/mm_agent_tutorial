# 第 6 章 Agent Handoff：任务移交与协作协议

### 1. 开篇段落

在构建复杂的智能体系统时，**"单体全能"（One Model Fits All）** 的幻想往往是项目失败的根源。尽管 GPT-4o 或 Gemini 1.5 Pro 等模型拥有巨大的上下文窗口，但试图让一个 Agent 连续完成“读取 500 页技术文档 -> 编写 Python 代码验证 -> 撰写商业报告”的全流程，通常会导致以下问题：

1. **注意力稀释（Attention Dilution）**：随着上下文变长，模型对早期指令或中间约束的遵循能力显著下降（Lost-in-the-Middle 现象）。
2. **角色混淆（Role Confusion）**：同一个 System Prompt 难以同时包含“严谨的审计员”和“富有创意的营销专家”两种立的人格约束。
3. **成本失控**：在长对话后期，为了修改一个标点符号而重新输入 100k tokens 是极大的浪费。

**Agent Handoff（智能体移交）** 是解决这些问题的核心架构模式。它不只是简单的函数调用，而是**责任、上下文与状态的正式转移**。本章将深入探讨如何设计一套“防弹”的移交协议，确保任务在不同专家 Agent（或人类）之间流转时，信息不丢失、目标不偏移、死循环被阻断。

**本章学习目标：**

1. **架构视角**：掌握线性、星型、分层等多种移交拓扑及其适用场景。
2. **协议设计**：构建标准化的 **Handoff Packet（移交包）**，实现“无状态重启”与“有损压缩”。
3. **多模态流转**：解决图片、音频、长文档在 Agent 间传递时的引用与带宽问题。
4. **防御性工程**：设计握手、拒绝与回退机制，防止“击鼓传花”式的死循环。

---

### 2. 核心论述：移交的工程艺术

#### 2.1 移交的本质：降低熵增

智能体系统的运行过程是一个熵增过程（噪声积累）。Handoff 的本质是一次**熵减操作**。
通过 Handoff，我们将前序 Agent 产生的冗长、混乱的思维链（Chain of Thought）和试错过程，压缩为清晰的**“结论”**和**“资产”**，传递给下一个拥有全新、干净上下文窗口（Context Window）的 Agent。

#### 2.2 移交拓扑结构 (Handoff Topologies)

不同的业务流需要不同的移交模式。不要被“多智能体”这个词迷惑，即使是两个 Agent，也存在复杂的拓扑。

**A. 线性接力 (The Relay / Pipeline)**
最简单、最稳健的模式。适用于标准化作业流程（SOP）。

* **适用**：文档处理流水线（OCR -> 翻译 -> 摘要 -> 格式化）。
* **特点**：单向流动，无回环。

```text
[ Source ] -> (Agent A: Ingest) -> [Structured Data] -> (Agent B: Analyze) -> [Report] -> (Agent C: Publish)

```

**B. 星型路由 (Hub-and-Spoke / Router)**
通过一个中央大脑（Router）来分发任务。

* **适用**：用户意图不明确的复杂对话助手（如客服总台）。
* **特点**：中央节点必须足够“聪明”且“克制”，只做分发，不做执行。
* **难点**：Router 容易成为瓶颈。如果 Router 判断错误，后续全是错的。

```text
                 +-----------+
                 |  Router   |
                 +-----+-----+
   (1) Intent?      /  |  \      (3) Aggregate
                   /   |   \
        +---------+ +--+--+ +---------+
        | Search  | | Code| | Draw    |
        +---------+ +-----+ +---------+
             (2) Execute & Return

```

**C. 升级与回退 (Escalation & Fallback)**
这是生产环境中最关键的模式。

* **一级移交**：快速模型（如 GPT-4o-mini）处理简单任务。
* **二级移交**：遇到低置信度或复杂逻辑，移交给强模型（如 GPT-4o/Claude 3.5）。
* **三级移交**：**Human-in-the-Loop**。当机器搞不时，生成一张工单（Ticket），移交给人类操作员。

**D. 动态协作 (Dynamic Collaboration / Swarm)**
Agent 之间可以任意互相呼叫（类似于微服务网格）。

* **风险**：极易产生死循环（Agent A 叫 B，B 叫 A）。必须引入 TTL（生存时间）或最大跳数限制。

#### 2.3 核心协议：Handoff Packet (移交包)

移交**绝对不应该**是把整个 Chat History 扔给下一个 Agent。这既昂贵又充满了上一轮的“思维垃圾”。
你需要定义一个标准化的 **JSON Schema** 作为移交包。

一个健壮的 Handoff Packet 必须包含以下四个象限：

| 字段类别 | 关键内容 | 说明 |
| --- | --- | --- |
| **1. Meta (元数据)** | `source_agent`, `target_agent`, `timestamp`, `handoff_id`, `priority` | 用于追踪、日志审计和防死循环（Traceability）。 |
| **2. Mission (任务)** | `objective`, `definition_of_done` (DOD), `constraints` | 下一个 Agent **必须**做什么？什么算做完？有什么禁？ |
| **3. Context (上下文)** | `summary_so_far`, `user_original_intent`, `key_facts` | 对前序工作的**有损压缩**。只保留结论，丢弃推理过程。 |
| **4. Artifacts (资产)** | `file_uris`, `image_ids`, `db_record_ids`, `extracted_data` | **引用指针**。不要传文件内容，传 S3 链接或数据库 ID。 |

> **多模态特别说明 (Rule of Thumb)**：
> * **图片**：传递 `image_id` 或 `url`。如果 Agent A 已经在图片坐标 (100, 200) 处发现异常，Packet 中应包含 `{"region_of_interest": [100, 200, 50, 50], "desc": "crack detected"}`，而不是让 Agent B 重新去“找茬”。
> * **文档**：传递文档 ID + 页码引用（Page Ref）。例如 `{"ref": "doc_123", "pages": [5, 6], "snippet": "..."}`。
> 
> 

#### 2.4 上下文压缩策略：如何“断章取义”？

移交时的最大痛点是信息丢失。如何保证 Agent B 能接得住？

1. **摘要链 (Summary Chain)**：Agent A 在移交前，必须执行一个内部思考步骤：“总结我目前已知的信息，并为下一个接手者写一份简报”。这份简报是 Packet 的核心。
2. **不可变事实 (Immutable Facts)**：维护一个全局的 KV 存储或清单。例如用户的姓名、订单号、已确认的需求。这些信息不随 Handoff 压缩，而是作为附件始终跟随。
3. **回溯指针 (Back-links)**：如果 Agent B 对简报有疑问，Packet 中应包含指向 Agent A 完整日志的链接（Trace ID），允许 B 在极少数情况下“查阅原档”（虽然我们尽量避免）。

#### 2.5 拒绝与握手协议 (Rejection & Handshake)

Agent Handoff 不总是成功的。Agent B 可能认为任务描述不清，或者自己无法处理。

* **ACK/NACK 机制**：
* Agent A 发送 Packet。
* Agent B 先进行 **Sanity Check（健全性检查）**：输入是否缺字段？图片是否无法下载？
* 如果检查失败，B 返回 `NACK` (Negative Acknowledgement) + `reason`，把球踢回给 A。
* A 收到 `NACK` 后，触发**自我修正 (Self-Correction)** 或报错。


* **能力边界声明**：每个 Agent 的 System Prompt 中应明确定义：“如果你收到不属于你能力范围的任务，请立即发起 Handoff 退回或转交，不要尝试强行回答”。

---

### 3. 本章小结

1. **分而治之**：Handoff 是解决 Context 限制和能力专业化的唯一路径。
2. **协议至上**：定义清晰的 Handoff Packet 结构（Meta, Mission, Context, Artifacts）比优化 Prompt 更重要。
3. **引用代替拷贝**：在多模态系统中，始终传递资产的 URI 和 ID，而非 Base64 或原始文本。
4. **防卫性设计**：必须设计死循环检测（TTL）、拒绝机制（NACK）和人工接管通道（Escalation）。
5. **有损压缩**：移交必然丢失信息，关键在于保留“结论”和“事实”，丢弃“过程”和“噪音”。

---

### 4. 练习题

#### 基础题 (50%) —— 熟悉协议与流程

<details>
<summary><strong>Q1: 为什么在移交时，不建议直接把上轮的 Chat History 完整拼接到下一个 Agent 的 Prompt 中？请列举三个原因。</strong> (点击展开)</summary>

1. **成本与配额**：历史记录可能非常长，导致 Token 消耗指数级增长，且容易超出上下文窗口限制。
2. **注意力干扰（噪音）**：上一个 Agent 的试错过程、失败尝试和纠结的思维链，对下一个 Agent 来说是干扰信息，可能导致它重蹈覆辙或产生幻觉。
3. **角色污染**：上一个 Agent 的 System Prompt 或对话风格可能会“泄漏”到下一个 Agent，导致角色扮演失败（例如严谨的审核员开始像推销员一样说话）。

**提示**：想象一下，你接手同事的工作时，是希望他给你一份 500 页的聊天记录，还是一份 1 页的交接文档？

</details>

<details>
<summary><strong>Q2: 设计一个简单的 JSON Handoff Packet，用于将一个“PDF 总结任务”从“阅读者 Agent”移交给“翻译 Agent”。</strong> (点击展开)</summary>

```json
{
  "meta": {
    "from": "reader_agent_v1",
    "to": "translator_agent_v1",
    "trace_id": "trace_88219"
  },
  "task": {
    "action": "translate_summary",
    "target_language": "Spanish",
    "tone": "professional"
  },
  "context": {
    "summary_content": "The Q3 financial report shows a 20% increase in revenue...",
    "key_terms": ["EBITDA", "YoY"]
  },
  "artifacts": {
    "source_doc_id": "s3://docs/q3_report.pdf",
    "source_page_refs": [3, 4]
  }
}

```

**提示**：包含元数据、具体要做什么、原始内容是什么、以及原始文件的引用。

</details>

<details>
<summary><strong>Q3: 什么是“死循环移交”（Infinite Handoff Loop）？如何通过协议字段防止它？</strong> (点击展开)</summary>

**现象**：Agent A 觉得这事该 B 做，移交给 B；B 看了觉得缺信息，又移回给 A。两者反复踢皮球。

**防止手段**：

1. **TTL / Hop Count**：在 Packet 的 `meta` 中增加 `hop_count` 字段，每次移交 +1。如果超过阈值（如 5），强制终止或转人工。
2. **Chain History**：记录已经经手过的 Agent 列表 `["A", "B"]`。B 看到 A 已经在列表里，且任务状态未变，应触发报错而非回传。

**提示**：网络数据包中的 TTL 字段是做什么的？

</details>

---

#### 挑战题 (50%) —— 系统设计与工程思维

<details>
<summary><strong>Q4: 场景：DeepResearch Agent 正在撰写报告，需要一张图表。它移交给 Code Agent 画图，但 Code Agent 报错说“数据列缺失”。请设计一个带有“自我修复”能力的移交流程。</strong> (点击展开)</summary>

这是一个典型的 **Request-Reply-Repair** 模式：

1. **Attempt 1**: Research Agent -> Handoff Packet (含数据) -> Code Agent。
2. **Failure**: Code Agent 运行 Python 失败，捕获 `KeyError: 'revenue'`。
3. **Return (NACK)**: Code Agent 不只是报错停止，而是发起一个 **Reverse Handoff** 回给 Research Agent。
* Content: "Execution Failed."
* Reason: "Missing column 'revenue' in provided dataset."
* Suggestion: "Please check extraction or provide correct CSV."


4. **Repair**: Research Agent 收到退回包，读取 Reason。它的 Policy 决定重新读取文档提取 'revenue' 列。
5. **Attempt 2**: Research Agent 发送新的 Packet (含修复后的数据) -> Code Agent。
6. **Success**: Code Agent 绘图成功，返回 Image URI。

**关键点**：Code Agent 的报错必须结构化返回给 Research Agent，而不是抛给用户。

</details>

<details>
<summary><strong>Q5: 在“人机移交”（Human Handoff）场景中，如何处理“人类操作员下班了”导致的长延迟问题？这对系统架构有什么影响？</strong> (点击展开)</summary>

**核心挑战**：**状态持久化与唤醒（State Hydration）**。

1. **异步架构**：系统不能让 Python 线程或 API 连接一直 `sleep()` 等待人类。
2. **序列化存储**：当决定移交给人时，必须将当前 Handoff Packet 序列化存入数据库（PostgreSQL/Redis），标记状态为 `WAITING_FOR_HUMAN`，并释放计算资源。
3. **回调钩子 (Webhook)**：当人类在 UI 上完成操作并点击“提交”时，前端触发 Webhook。
4. **状态恢复**：系统根据 ID 从数据库拉取 Packet，将人类的输入注入到 `context` 中，然后**重新实例化**一个新的 Agent 继续运行。

**结论**：人机移交强制要求系统架构必须是**无状态（Stateless）**且**事件驱动（Event-driven）**的。

</details>

<details>
<summary><strong>Q6 (开放题): 假设你有一个包含 5 个不同专家 Agent 的系统。请设计一个评分机制，让 Router 自动决定把任务派给谁，而不是写死规则。</strong> (点击展开)</summary>

**思路：基于描述符的语义匹配（Semantic Routing）。**

1. **注册机制**：每个 Agent 在启动时，向 Router 注册自己的 `Manifest`（清单）。
* `Agent A (Coder)`: "擅长 Python, 数据处理, 绘图。输入需要 CSV。"
* `Agent B (Writer)`: "擅长 本润色, 创意写作, 总结。输入需要文本。"


2. **任务向量化**：Router 接收到任务 "帮我分析这个 Excel 并画个图" 时，将其 Embedding 为向量。
3. **相似度匹配**：计算 任务向量 与 Agent 描述向量 的相似度（Cosine Similarity）。
4. **动态选择**：选择得分最高的 Agent。
5. **反馈闭环 (RLHF)**：如果 Router 派给 A，但 A 失败了（返回 NACK），Router 应降低该类任务对 A 的权重，并尝试得分第二高的 Agent。

**提示**：这类似于 RAG 中的检索过程，只不过检索的是“工具人”而不是“文档”。

</details>

---

### 5. 常见陷阱与错误 (Gotchas)

#### 5.1 传声筒效应 (The Telephone Game)

* **现象**：Agent A 读了 PDF，总结给 B；B 读了总结，精简给 C。到了 C，"利润增长 50%" 变成了 "利润有增长"，关键细节丢失。
* **调试技巧**：
* **证据穿透**：允许 C 直接访问 A 提取的原始数据片段（Raw Snippets），而不仅是 B 的转述。
* **检查点测试**：人工检查中间环节的 Packet 内容，确认信息熵减是否过度。



#### 5.2 格式承诺崩溃 (Schema Promise Break)

* **现象**：Agent A 的 Prompt 里写着“请输出 JSON”，但因为模型抽风输出了 Markdown 代码块包裹的 JSON。Agent B 的解析器直接崩溃。
* **调试技巧**：
* **中间件清洗**：在 Handoff 发生前，增加一个非 AI 的代码层（Parser），负责把 Markdown 清洗掉，提取纯 JSON。如果提取失败，直接在代码层打回给 A 重试，**严禁**把脏数据传给 B。



#### 5.3 幻觉引用 (Hallucinated References)

* **现象**：Agent A 并没有生成图片，但为了满足输出要求，捏造了一个 `image_id: "img_1234"`。Agent B 拿着这个 ID 去数据库查，发现为空，程序崩溃。
* **调试技巧**：
* **存在性校验**：Artifacts 字段必须经过系统校验。如果 ID 在数据库不存在，系统层拦截 Packet，标记为“非法移交”。



#### 5.4 幽灵权限 (Phantom Privileges)

* **现象**：用户把“删除文件”的权限授给了 Agent A（管家），但 A 移交给了 Agent B（外部插件）。B 继承了 A 的权限删除了文件。这违反了安全原则。
* **调试技巧**：
* **权限降级**：Handoff 时，Packet 中应包含 `scope` 字段。Agent B 启动时，只能获得 Packet 中明确授权的最小权限集，而不是继承 A 的所有权限。


