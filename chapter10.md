# 第 10 章 Trace 构造、蒸馏与 Benchmark 评测体系

## 10.1 开篇与学习目标

在多模态智能体（Agent）的开发中，我们经常面临这样的困境：Demo 阶段看着很惊艳，一上生产环境就“智障”。当智能体回答错误时，是因为 OCR 没看清？还是 Prompt 没写好？亦或是检索到的文档本身就是错的？

如果没有一套完善的 **Trace（全链路追踪）** 和 **Evaluation（评测）** 体系，优化智能体就像在黑暗中扔飞镖——全靠运气。

本章将带你构建智能体的“黑匣子”与“体检中心”。我们将把运行日志转化为结构化的资产，通过自动化评测发现短板，最后利用这些数据进行“蒸馏”，训练出更、更快、更强的专用模型，形成**数据飞轮（Data Flywheel）**。

**本章学习目标**：

1. **掌握 Trace Schema 设计**：定义一套兼容多模态、工具调用、思维链的标准数据协议。
2. **构建分层评测体系**：从单元测试（L1）到轨迹质量（L2）再到端到端业务指标（L3）。
3. **实施数据蒸馏（Distillation）**：学会如何把“错误日志”清洗并修复为“黄金训练数据”。
4. **打造私有 Benchmark**：建立针对垂直领域的回归测试集，防止模型能力退化。

---

## 10.2 什么是 Trace：智能体的“心电图”

传统的软件日志（Log）记录的是**状态**（State，如变量值、报错信息），而 Agent 的 Trace 记录的是**思考与行动的流**（Flow of Thought & Action）。

### 10.2.1 剖析 Trace 的解剖学结构

一个健壮的 Trace 系统必须能够还原智能体与环境交互的每一个原子步骤。我们通常采用 **树状结构（Tree Structure）**  **DAG（有向无环图）** 来表示。

**逻辑结构图示**：

```text
Session (Trace ID: sess_8a2b...) [User: "分析这份财报的风险"]
│
├── Metadata (User_ID, Model_Config, Start_Time, Budget_Cap)
│
├── Turn 1 (User Input)
│   ├── Node 1.0: Input Processing (PDF Parsing -> Text/Layout)
│   │
│   ├── Node 1.1: Reasoning (CoT)
│   │   └── Content: "用户需要分析风险，我先看目录，定位到‘风险因素’章节..."
│   │
│   ├── Node 1.2: Action (Tool Call)
│   │   ├── Tool: `doc_search`
│   │   └── Args: `{"query": "risk factors", "top_k": 3}`
│   │
│   ├── Node 1.3: Observation (Tool Output)
│   │   └── Content: "Found 3 chunks: 1. Market risk... 2. Credit risk..."
│   │
│   └── Node 1.4: Reflection & Answer (Final Response)
│       └── Content: "根据文档，主要风险有..."
│
└── Turn 2 (Evaluation / Feedback)
    └── Labels: {"correctness": 5, "latency_ms": 4500}

```

### 10.2.2 核心 Trace Schema 设计（JSON 定义）

在工程落地中，建议参考 OpenTelemetry 或 OpenAI 的格式，但需针对 Agent 增强。以下是一个生产级的 Schema 定义建议：

#### 1. 基础节点（Span/Step）

每一个思考、调用、观察都应是一个独立的 Span。

```json
{
  "span_id": "step_1024",
  "trace_id": "sess_8a2b",
  "parent_id": "step_1023",  // 链接上一步，构成链条
  "role": "assistant",       // system | user | assistant | tool | environment
  "type": "thought",         // thought | action | observation | final_answer
  "content": {
    "text": "我需要先查询当前汇率。",
    "images": ["s3://assets/chart_snapshot_v1.png"] // 多模态引用
  },
  "metadata": {
    "model": "gpt-4o",
    "token_usage": {"prompt": 150, "completion": 40},
    "latency_ms": 800,
    "cost_usd": 0.003
  },
  "status": "success"        // success | error | timeout
}

```

#### 2. 多模态数据的“引用原则”

**Rule of Thumb**：**永远不要**把 Base64 图片或长篇 PDF 文本直接塞进 Trace 数据库的主索引字段中。

* **做法**：将多模态数据上传至对象存储（S3/OSS），在 Trace 中只保留 `URI`、`MIME type` 和 `Hash`。
* **特例**：对于 PDF 理解任务，Trace 中必须记录 **Location Reference**（如 `page: 5, bbox: [100, 200, 300, 400]`），以便调试时知道模型到底“看”到了哪里。

---

## 10.3 评测体系（Evaluation）：金字塔模型

不要试图用一个“准确率”指标来衡量复杂的 Agent。我们需要建立一个**三层评测金字塔**。

### 10.3.1 Level 1: 单元测试（Unit Testing）

**目标**：确保“零件”是好的。

* **格式依从性 (Format Compliance)**：工具调用是否输出了合法的 JSON？
* **原子能力 (Atomic Capability)**：
* *OCR 测试*：给一张标准发票，检测金额字段的提取准确率。
* *工具选择测试*：给一句指令“我想买票”，模型是否选择 `ticket_tool` 而不是 `weather_tool`？


* **成本**：低。完全自动化，基于规则或正则。

### 10.3.2 Level 2: 轨迹质量与集成评测 (Trajectory Eval)

**目标**：确保“过程”是合理的。Agent 是否走了弯路？是否发生了死循环？
这里通常引入 **LLM-as-a-Judge**（用强模型评测弱模型）。

**典型评测维度（Rubrics）**：

1. **效率 (Efficiency)**：完成任务是否用了最少的步骤？（例如：是否反复调用同一个查询？）
2. **幻觉检测 (Hallucination)**：回答中的事实是否都能在 `Observation`（工具返回结果）中找到证据？
3. **安全性 (Safety)**：是否尝试读取越权文件或执行危险命令？

**Judge Prompt 示例片段**：

> "你是一名审计员。请检查以下 Agent 的操作记录。如果 Agent 在没有搜索任何信息的情况下就凭空捏造了数据，请打 0 分；如果 Agent 正确引用了搜索结果，打 1 分。请特别注意数字的一致性..."

### 10.3.3 Level 3: 端到端业务指标 (E2E Business Metrics)

**目标**：确保“结果”对用户有用。

* **任务成功率 (Success Rate, SR)**：
* *确定性任务*（如订票）：检查最终状态是否为 `booked`。
* *开放性任务*（如写研报）：通常需要人类专家评分或与黄金标准（Golden Answer）对比语义相似度。


* **通过率 (Pass@k)**：允许尝试 k 次（例如 k=3），只要有一次做对就算过。这对代码生成 Agent 尤为重要。
* **人工接管率 (Human Intervention Rate)**：用户由于不耐烦或出错而切换到人工客服的比例。

---

## 10.4 蒸馏（Distillation）：从 Trace 到训练数据

Trace 数据的终极价值，在于将昂贵的“推理过程”内化为廉价模型的“直觉”。

### 10.4.1 数据飞轮流水线

1. **采集 (Capture)**：全量记录线上 Trace。
2. **筛选 (Filter)**：
* 基于规则：剔除报错、超时的会话。
* 基于反馈：保留用户点了“赞”或 Judge 评高的会话。


3. **修正 (Curate/Refine)** —— **这是最关键的一步**：
* 对于失败的 Trace，不要直接扔掉。
* **修正克隆 (Correction Cloning)**：人工（或更强的模型）介入，修改出错的那个 Step（例如修正错误的 SQL 语句），然后让 Agent 基于修正后的状态继续跑完。
* *价值*：这样你就得到了一条“曾经失败，但被纠正”的高价值训练数据，模型能从中学习如何避免同类错误。


4. **训练 (Train)**：
* **SFT (监督微调)**：输入 Prompt，输出修正后的 Trace。
* **DPO (直接偏好优化)**：构建配对数据 `{Prompt, Chosen_Trace, Rejected_Trace}`，让模型学习“好坏之分”。



### 10.4.2 蒸馏策略对比

| 策略 | 描述 | 适用场景 | 优点 | 缺点 |
| --- | --- | --- | --- | --- |
| **SFT (Result Only)** | 只学习 `User Input -> Final Answer` | 简单问答、分类 | 速度极快，成本低 | 丧失推理能力，无法处理复杂任务 |
| **SFT (CoT)** | 学习 `Input -> Thought -> Action -> ...` | 复杂推理、多步操作 | 保留思维链，泛化性强 | 上下文长，训练成本高 |
| **DPO / PPO** | 学习 `Trace A > Trace B` | 减少幻觉、对齐人类偏好 | 提升模型“品味” | 数据构造难，训练不稳定 |

---

## 10.5 Benchmark 构造与管理

不要迷信公开榜单（如 GAIA, MMMU）。你的业务场景需要**私有 Benchmark**。

### 10.5.1 数据集构成三要素

1. **Golden Set (金标准集)**：
* 来源：由人类专家标注的 100~500 条高质量问答对。
* 用途：每次代码提交前的回归测试（Regression Test）。


2. **Hard Negatives (困难负例集)**：
* 来源：线上运行失败的 Case 集合。
* 用途：针对性训练，解决“长尾问题”。


3. **Red Teaming Set (红队测试集)**：
* 来源：专门构造的对抗性攻击（如提示注入、视觉欺骗图）。
* 用途：安全水位线测试。



### 10.5.2 动态环境的评测难题

Agent 往往依赖实时据（如股价、网页）。如果 Benchmark 的答案是静态的，评测就会失效。

**解决方案**：

* **Mock 环境**：录制工具调用的返回结果（VCR 模式）。评测时，拦截网络请求，回放录制好的数据，确保环境状态一致。
* **程序化验证**：答案不是文本，而是一个校验脚本。例如任务是“在这个文件夹下建一个 test.txt”，评测脚本是 `os.path.exists("test.txt")`。

---

## 10.6 本章小结

* **Trace 即资产**：没有 Trace，Agent 就是黑盒。Trace Schema 设计要兼顾人类可读性和机器可训练性。
* **评测分层**：不要只看最终结果。L1 测格式，L2 测逻辑，L3 测价值。
* **化腐朽为神奇**：失败的 Trace 经过“修正克隆”是最好的 DPO 负样本来源。
* **闭环**：最佳的开发模式是 `Code -> Deploy -> Trace -> Distill -> Fine-tune -> Code` 的无限循环。

---

## 10.7 练习题

### 基础题（熟悉 Schema）

1. **Schema 设计**：请为一**代码编写 Agent** 设计 Trace 中的 `Action` 节点结构。
* *要求*：必须包含文件名、代码内容、写入模式（覆盖/追加），以及该操作对应的“回滚操作”字段（用于事务安全）。
* *输出*：提供 JSON 示例。


2. **评测指标计算**：
* Agent 运行了 10 个任务。
* 其中 8 个任务最终输出了正确答案。
* 在这 8 个成功任务中，有 2 个任务调用了多余的工具（例如想看天气却先调了股票接口）。
* *问题*：计算该 Agent 的 **Success Rate (SR)** 和 **Perfect Execution Rate (PER)**。



### 挑战题（进阶思考）

3. **多模态引用设计**：在一个处理 1000 页 PDF 的 RAG 场景中，Trace 需要记录模型每一次检索依据的“证据片段”。如果直接存文本太长，存页码太模糊（一页有 10 段）。
* *任务*：设计一个 `Citation Schema`，既能精确定位到 PDF 的特定行/区域，又能在前端 UI 上通过高亮显示给用户看。提示：参 PDF 坐标系或 DOM 结构。


4. **DPO 数据构造**：你有一个基于 ReAct 框架的 Agent。你发现它经常在“搜索结果为空”时陷入死循环（反复搜索同一个词）。
* *任务*：描述如何利用这些死循环日志构造一组 DPO 数据，教会模型“当搜索不到时，尝试换个关键词或询问用户”。



<details>
<summary>点击展开答案与提示</summary>

**1. Schema 设计示例**

```json
{
  "type": "action",
  "tool": "write_file",
  "args": {
    "file_path": "src/utils.py",
    "content": "def hello(): pass",
    "mode": "overwrite"
  },
  "rollback": { // 关键字段：定义如何撤销此操作
    "tool": "restore_file",
    "args": {
      "file_path": "src/utils.py",
      "backup_id": "bak_28192"
    }
  }
}

```

**2. 评测指标计算**

* **Success Rate (SR)** = 8 / 10 = **80%**
* **Perfect Execution Rate (PER)** = (8 - 2) / 10 = **60%**
* *解析*：PER 要求结果正确且过程无冗余。



**3. 多模态引用设计提示**

* **方案**：结合 `page_index` 和 `bbox` (Bounding Box)。
* **JSON 示例**:
```json
"citations": [
  {
    "doc_id": "doc_123",
    "page": 42,
    "bbox": [100, 200, 500, 250], // [x1, y1, x2, y2]
    "text_snippet": "Revenue grew by 20%...",
    "quote_quality_score": 0.95
  }
]

```


* 前端利用 `bbox` 在 PDF Canvas 层上绘制高亮矩形。

**4. DPO 数据构造思路**

* **Rejected (负例)**：包含重复搜索动作的 Trace片段。
* `Thought: 没搜到 X。Action: Search(X) -> Obs: Empty -> Thought: 再搜 X...`


* **Chosen (正例)**：人工介入修改后的 Trace。
* `Thought: 没搜到 X，这很奇怪。我应该尝试同义词 Y。Action: Search(Y)...`


* **训练**：将 User Prompt + Context 作为输入，让模型最大化 Chosen 的概率，最小化 Rejected 的概率。

</details>

---

## 10.8 常见陷阱与错误 (Gotchas)

### 1. 评测集污染 (Contamination)

* **现象**：模型在 Benchmark 上得分很高，实际用起来很烂。
* **原因**：发者无意中把测试集的题目（或极其相似的变体）放进了微调训练集中。
* **对策**：使用 **Canary Strings**（金丝雀字符串）。在私有测试集中埋入一段无意义的特殊代码（如 `UUID: 4a2b...`），如果训练后的模型能自动补全这段代码，说明数据泄露了。

### 2. 长度偏见 (Length Bias)

* **现象**：LLM-as-a-Judge 倾向于给回答更长、废话更多的 Agent 打高分，即使它逻辑有漏洞。
* **对策**：
* 在 Judge Prompt 中明确惩罚啰嗦。
* 进行 **Pairwise Evaluation** 时，交换两个回答的顺序再测一次（Position Bias check）。



### 3. "自嗨"式蒸馏 (Self-Correction Loop)

* **现象**：让模型自己生成数据、自己评测、自己训练。
* **风险**：如果缺乏外部真值（Ground Truth）或人类监督，模型会发生“模型坍塌”（Model Collapse），逐渐强化自己的错误偏见。
* **对策**：蒸馏的数据必须经过**强过滤**（例如通过代解释器的执行验证），或者保留至少 10% 的人类专家数据混合训练。
