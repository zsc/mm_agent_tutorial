# 附录 D：Benchmark 清单与自建评测指南 (chapter23.md)

> **摘要**：在多模态智能体（Agent）的工程落地中，评测（Evaluation）不仅是分数的比拼，更是系统稳定性的生命线。本章首先梳理了当前主流的公开 Benchmark 版图，帮助开发者进行模型选型；随后，深入剖析了如何从零构建业务专属的“金标准数据集（Golden Set）”，设计分层级的自动化评分器，并将其集成到 CI/CD 流水线中，建立起一套“可观测、可量化、可回归”的质量保障体系。

---

## D.1 Benchmark 类型地图：从选型到定标

公开 Benchmark 的主要价值在于**模型选型（Model Selection）**和**能力基线（Capability Baseline）**的建立。盲目追求综合高分不仅浪费资源，还可能导致对特定领域缺陷的忽视。我们需要根据 Agent 的应用场景，选择最相关的“标尺”。

### D.1.1 多模态理解与推理类 (Perception & Reasoning)

此类 Benchmark 主要考察 VLM（Visual Language Model）对图像、图表、文档的感知与逻辑推理能力，是所有多模态 Agent 的基石。

| Benchmark 名称 | 核心考察点 | 适用 Agent 场景 | 局限性/注意事项 |
| --- | --- | --- | --- |
| **MMMU** (Massive Multi-discipline Multimodal Understanding) | 大学水平的多学科知识推理（图表、图示、公式）。 | **文档分析、专家咨询、教育辅导** | 侧重选择题，难以反映 Agent 的“长链条”推理能力。 |
| **MMBench** | 感知与推理的综合能力，采用 CircularEval 消除猜测成分。 | **通用多模态助手** | 问题相对短小，无法评估长文档处理能力。 |
| **DocVQA / InfographicVQA** | 针对文档扫描件、海报、信息图的文字提取与问答。 | **RPA、票据处理、文档检索** | 必须关注 OCR 的精度与空间位置理解。 |
| **MathVista** | 视觉环境下的数学推理与计算。 | **数据分析、科学计算** | 对逻辑严密性要求极高。 |

### D.1.2 工具使用与 Agent 综合能力类 (Tool Use & General Agent)

此类 Benchmark 考察模型是否真的能“做事”，即调用 API、规划路径、纠正错误。

| Benchmark 名称 | 核心考察点 | 适用 Agent 场景 | 局限性/注意事项 |
| --- | --- | --- | --- |
| **GAIA** (General AI Assistants) | 概念简单但步骤极繁琐的任务，需精准组合多种工具。 | **个人助理、复杂任务规划** | 它是目前最接近“人类助理”难度的公开测试集，极具参考价值。 |
| **AgentBench** | 包含 OS、DB、KG 等 8 个不同环境的综合测试。 | **通用作系统 Agent** | 环境配置复杂，复现成本较高。 |
| **ToolBench** | 大规模的真实 API 调用指令遵循与参数填充。 | **插件调度、API 网关助手** | 数据多为合成，可能缺乏真实世界的“脏数据”噪声。 |

### D.1.3 代码与软件工程类 (Coding & SWE)

| Benchmark 名称 | 核心考察点 | 适用 Agent 场景 | 局限性/注意事项 |
| --- | --- | --- | --- |
| **SWE-bench** | 基于真实 GitHub Issue 的仓库级问题解决（复现+修复+测试）。 | **Devin 类编程助手、自动运维** | **黄金标准**。必须在 Docker 容器中运行，资源消耗巨大。 |
| **HumanEval / MBPP** | 函数级的代码生成与补全。 | **代码补全 Copilot** | 仅适合测试底座模型的代码语法能力，不足以评估工程 Agent。 |

### D.1.4 浏览器与 GUI 自动化类 (Web & Embodied)

| Benchmark 名称 | 核心考察点 | 适用 Agent 场景 | 局限性/注意事项 |
| --- | --- | --- | --- |
| **WebArena / Mind2Web** | 真/模拟网站中的端到端任务（购物、预订、表单）。 | **浏览器自动化、订票助手** | 网页 DOM 经常变动，环境维护成本极高。 |
| **AndroidWorld** | 安卓系统内的跨 App 操作与任务完成。 | **手机端助手** | 依赖模拟器，动作空间较大。 |

---

## D.2 自建评测集：构建你的 "Golden Set"

公开数据往往存在**数据泄露（Contamination）**问题，且无法覆盖特定的业务逻辑（如“只能在周一到周五调用退款接口”）。**自建 Golden Set 是 Agent 能够上线的必要条件。**

### D.2.1 数据来源的“倒金字塔”策略

构建高质量评测集应遵循优先级从高到低的策略：

1. **L0 - 线上真实故障 (Production Bad Cases)**：
* **来源**：用户反馈的报错、客服介入的对话、导致崩溃的 Trace。
* **价值**：最高。这是系统曾经“流血”的地方，必须确保 Fix 后不再复发（Regression Test）。
* **处理**：必须脱敏，并将其转为可自动化执行的测试用例。


2. **L1 - 核心业务路径 (Critical User Journeys - CUJs)**：
* **来源**：产品经理定义的“Happy Path”和常见“Sad Path”（异常流程）。
* **价值**：高。确保产品主功能（如“查询库存并下单”）始终可用。
* **形式**：通常由专家人工撰写。


3. **L2 - 对抗与边界测试 (Red Teaming & Edge Cases)**：
* **来源**：安全团队的注入攻击、Prompt Injection、无意义输入、超长文本。
* **价值**：中高。保证系统的鲁棒性和安全性。


4. **L3 - 合成增强数据 (Synthetic Data)**：
* **来源**：使用强模型（如 GPT-4）对 L1 数据进行改写（Rephrasing）、参数替换、多语言翻译。
* **价值**：中。用于提升测试覆盖率和泛化能力。



### D.2.2 评测样本的标准 Schema (JSON)

一个优秀的评测样本不应只有“输入”和“输出”，而应包含完整的**上下文**和**验证逻辑**。

```json
{
  "case_id": "refund_policy_001",
  "category": "customer_service",
  "difficulty": "hard",
  "input": {
    "user_query": "我上周买的咖啡机坏了，能退吗？",
    "context_files": ["policy_2023.pdf"],  // 关联的知识库文件
    "mock_time": "2023-10-25T10:00:00Z",   // 固定时间，防止因时间变化导致测试抖动
    "user_profile": {"vip_level": 1}       // 用户状态模拟
  },
  "gold_standard": {
    "expected_action_sequence": ["search_policy", "check_order_status"], // 必须调用的工具
    "forbidden_actions": ["direct_refund"], // 严禁调用的工具（无权限时）
    "key_facts": ["7天无理由", "质量问题30天包换"], // 必须包含的语义点
    "required_citation": {"page": 12, "text_snippet": "质量问题..."} // 必须引用的证据
  },
  "constraints": {
    "max_latency_ms": 5000,
    "max_tokens": 1000
  }
}

```

---

## D.3 自动化评分体系：分层打分机制

人工评测（Human Eval）极其昂贵且不可扩展。我们需要构建一个分级的自动化评分流水线（Scoring Pipeline）。

### D.3.1 Level 1: 确定性规则评分 (Rule-based Scorer)

* **适用场景**：格式检查、工具调用参数校验、关键词匹配。
* **成本**：极低。
* **方法**：
* **JSON Schema Validator**：输出是否符合 JSON 格式？字段是否缺失？
* **Exact Match**：SQL 生成是否完全一致？提取的身份证号是否正确？
* **Regex**：是否包含敏感词？是否包含特定的 URL 格式？



### D.3.2 Level 2: 执行结果评分 (Execution-based Scorer)

* **适用场景**：Coding Agent、SQL Agent、API 调用类。
* **成本**：高（需要沙箱环境）。
* **方法**：
* **Unit Test**：运行 Agent 生成的代码，看是否通过单元测试。
* **SQL Execution**：在影子库中执行生成的 SQL，对比返回的 Rows 是否与标准答案一致（**这是比文本对比更准确的方法**）。
* **API Dry-run**：模拟执行 API，检查请求体（Payload）是否符合后端要求。



### D.3.3 Level 3: 模型裁判评分 (LLM-as-a-Judge)

* **适用场景**：开放式问答、摘要生成、多模态推理、Tone & Style 检查。
* **成本**：中高（Token 消耗）。
* **方法**：
* 使用最强模型（如 GPT-4o, Claude 3.5 Sonnet）作为裁判。
* **Rubric-based Grading**：提供详细的评分细则（1-5分制），要求裁判先解释理由再打分。
* **Reference-guided**：传入标准答案（Gold Answer），要求裁判对比生成结果与标准答案的**语义一致性**，而非字面一致性。



> **最佳实践：Pairwise Comparison（成对比较）**
> 不要让 LLM 直接打分（存在方差），而是让它对比 `Model_A_Output` 和 `Model_B_Output`（或者是 `New_Version` vs `Baseline`），判断哪个更好。这种方式更接近人类直觉，准确率更高。

#### 附：通用 LLM 裁判 Prompt 模板

```markdown
You are an expert evaluator for an AI assistant.
[User Query]: {query}
[Context]: {context}
[Gold Standard Answer]: {gold_answer}
[Model Output]: {model_output}

Please evaluate the Model Output based on the following dimensions:
1. **Accuracy**: Does it match the Gold Standard semantically? (Yes/No)
2. **Citation**: Did it cite the correct document/page? (Yes/No/NA)
3. **Hallucination**: Does it contain information NOT present in Context? (Yes/No)
4. **Completeness**: Did it miss any key constraints?

Output a JSON with scores and a brief reasoning.

```

---

## D.4 生产级评测流水线与回归策略

将评测集成到 DevOps 流程中，是防止 Agent“变笨”的唯一手段。

### D.4.1 评测集的分级执行策略

为了平衡速度与覆盖率，建议将 Golden Set 分为三级：

1. **Smoke Test (冒烟测试)**
* **规模**：10-20 个核心 Case。
* **触发时机**：每次 Git Commit / PR 提交。
* **要求**：100% 通过，耗时 < 2分钟。主要跑 Level 1 评分。


2. **Regression Test (回归测试)**
* **规模**：100-300 个典型 Case（覆盖所有 Topic）。
* **触发时机**：每日夜间构建（Nightly Build）或上线前封板。
* **要求**：Pass Rate 不低于基线（Baseline），主要跑 Level 2 + Level 3 评分。


3. **Full Evaluation (全量评测)**
* **规模**：1000+ Case（包含合成数据、压力测试）。
* **触发时机**：更换基座模型（Base Model）或重大架构重构时。



### D.4.2 关键观测指标 (Metrics that Matter)

除了准确率，工程化评测必须关注以下维度的“退化”：

* **Pass Rate (SR)**：任务成功率。对于 Agent，**Pass@1** (一次做对) 远比 Pass@k 重要。
* **Step Count Efficiency**：完成同样任务，新版本是否用了更少的步骤？（步骤越少，出错概率越低，越省钱）。
* **Cost per Task**：平均每个任务消耗的 Token 成本。
* **Latency p95/p99**：端到端耗时的尾部延迟。
* **Hallucination Rate**：在“不可知”问题上，模型是否能够正确拒识（Refusal），而不是胡编乱造。

### D.4.3 持续改进闭环 (The Flywheel)

1. **Monitor**: 线上发现 Bad Case。
2. **Capture**: 抓取 Trace，人工标注正确逻辑。
3. **Add**: 将其加入 Golden Set 的 Regression 组。
4. **Debug**: 开发者修复 Bug，本地跑通测试。
5. **Deploy**: 上线，确保该类错误永不复发。

---

## 附：自建评测工具链推荐

* **Promptfoo**: 轻量级、开发者友好的 CLI 工具，支持各种断言（Asserts）和 LLM 裁判，非常适合集成到 CI 中。
* **Ragas**: 专注于 RAG（检索增强生成）系统的评测框架，提供 Context Recall, Faithfulness 等细粒度指标。
* **LangSmith / Langfuse**: 提供可视化的 Trace 记录和在线评测数据集管理功能，适合团队协作标注。

---

*(本章完)*
