# 附录 C：Trace Schema 与蒸馏数据构建 (chapter22.md)

> **本章概要**
> 在多模态智能体系统中，**Trace（轨迹）** 是连接运行时（Inference）与训练时（Training）的唯一血脉。它不仅是调试的依据，更是构建“数据飞轮”的核心资产。
> 许多团队的痛点在于：线上跑了数万次任务，但因为 Log 格式混乱、缺乏证据对齐、多模态数据丢失，导致无法利用这些数据反哺模型。
> 本章将提供一套**工业级、标准化**的 Trace Schema 设计，并详解如何构建从“原始日志”到“SFT/DPO 训练数据”的 ETL 流水线。
> **核心产出**：
> 1. **全景 Trace 协议**：支持文本、图像、工具、反思、用的统一 JSON 结构。
> 2. **证据锚点（Grounding）**：解决多模态幻觉的坐标级对齐方案。
> 3. **数据清洗流水线**：从 PII 脱敏到质量打分的七道工序。
> 4. **偏好数据集构建**：自动化构造 Chosen/Rejected 样本的策略。
> 
> 

---

## C.1 Trace 的全景架构与生命周期

在深入 JSON 字段之前，我们需要理解一条 Trace 在系统中的流动。

### Trace 数据流向图 (ASCII)

```text
[Runtime: Agent Execution]
       │
       ▼
   (RAW TRACE)  <-- 包含冗余信息、Base64图片、原始HTML、未脱敏敏感词
       │
       ▼
[ETL Pipeline: Cleaning & Formatting]
       │
       ├── 1. 图片上传至 S3 (替换 Base64 为 URL)
       ├── 2. 敏感信息 (PII) 掩码处理
       ├── 3. 工具输出截断/摘要 (Canonicalization)
       └── 4. 结构化校验 (Schema Validation)
       │
       ▼
   (GOLDEN TRACE) <-- 存入数据湖 (Data Lake)，用于检索与回放
       │
       ├── 分支 A: 行为克隆 (SFT Data Generation)
       └── 分支 B: 偏好对齐 (DPO Pair Construction)

```

---

## C.2 通用 Trace Schema 定义 (The Universal Schema)

这是一套兼容 OpenAI Chat 格式但扩展了 **Agent 特性**（如思考链、引用、执行状态）的 Schema。

### 2.1 顶层信封 (The Envelope)

```json
{
  "trace_id": "req_550e8400-e29b",      // 全局唯一请求 ID
  "session_id": "sess_a1b2c3d4",         // 会话 ID (用于多轮上下文)
  "user_id": "user_12345",               // 用户标识 (用于权限/个性化)
  "timestamp_start": "2023-10-27T10:00:00Z",
  "timestamp_end": "2023-10-27T10:00:15Z",
  "duration_ms": 15000,
  "status": "success",                   // success | failure | timeout
  
  // 环境与配置快照 (这对复现 Bug 至关重要)
  "meta": {
    "model_name": "gpt-4-vision-preview",
    "temperature": 0.1,
    "system_prompt_version": "v3.1.2_finance_analyst",
    "tools_schema_hash": "a8f9c2...",    // 工具定的哈希值，防止工具变了 Log 没变
    "environment": "production"
  },
  
  // 核心交互步骤
  "steps": [ ... ],
  
  // 最终产出与评测
  "outcome": { ... }
}

```

### 2.2 步骤详情 (The Steps) - 多模态核心

步骤必须是一个**有序列表**，严格还原时间线。我们采用 `role` + `content` + `metadata` 的结构。

#### A. 用户输入 (Multimodal User Input)

```json
{
  "step_index": 0,
  "role": "user",
  "timestamp": "...",
  "content": [
    {
      "type": "text", 
      "text": "请分析这张合同中的付款条款，并检查是否合规。"
    },
    {
      "type": "image_url",
      "image_url": {
        "url": "https://s3.bucket/path/to/img_01.jpg", // 这里的 URL 必须是持久化的
        "detail": "high"
      },
      "metadata": {
        "original_filename": "contract_scan.jpg",
        "ocr_cache_key": "ocr_img_01" // 关联 OCR 预处理结果
      }
    }
  ]
}

```

#### B. 模型思考与动作 (Assistant Thought & Action)

是 Agent 的“大脑”活动。必须区分 **思考 (Thought)** 和 **调用 (Call)**。

```json
{
  "step_index": 1,
  "role": "assistant",
  "content": null, // 对于 Tool Call 步骤，content 通常为空或包含 CoT
  
  // 显式的思维链 (CoT)
  "thought": "用户上传了合同图片。我首先需要调用 OCR 工具提取文本，然后使用关键词搜索'付款条款'。",
  
  "tool_calls": [
    {
      "id": "call_ocr_123",
      "type": "function",
      "function": {
        "name": "document_ocr",
        "arguments": "{\"image_id\": \"img_01\", \"mode\": \"dense_text\"}"
      }
    }
  ]
}

```

> **关键点**：`thought` 字段是蒸馏小模型（SLM）的核心。小模型往往缺乏规划能力，通过学习大模型的 `thought`，可以显著提升其推理能力。

#### C. 工具执行结果 (Tool Execution)

记录环境的真实反馈。

```json
{
  "step_index": 2,
  "role": "tool",
  "tool_call_id": "call_ocr_123", // 必须对应上面的 call_id
  "name": "document_ocr",
  
  // 原始输出，可能非常长。训练时需要截断，但 Trace 中建议保留全量 (或存链接)
  "content": "{\"full_text\": \"合同编号：HB-2023... 本合同付款方式如下：...\", \"pages\": 1}",
  
  // 执行状态元数据
  "metadata": {
    "status": "ok",          // ok | error
    "latency_ms": 1200,
    "is_truncated": false    // 标记内容是否因过长被截断
  }
}

```

#### D. 错误处理与自我修正 (Self-Correction)

如果工具报错，Trace 应如实记录，这构成了“负样本”或“修正样本”。

```json
// Step 3: Assistant 收到错误后的反应
{
  "role": "assistant",
  "thought": "OCR 提取成功，现在我尝试在文本中搜索关键词。哎呀，我之前的搜索参数格式好像错了，我需要修正参数。",
  "tool_calls": [...]
}

```

---

## C.3 证据对齐与 Grounding Schema

为了解决幻觉，Answer 必须“钉”在源文档上。我们不应只存储文本，应存储“带坐标的引用”。

### 引用对象 (Citation Object)

在 `outcome` 字段中：

```json
"outcome": {
  "final_answer": "根据合同第 3 页，付款账期为验收后 30 天 [1]。",
  "citations": [
    {
      "id": 1,
      "source_doc_id": "doc_contract_001",
      "quote_text": "乙方应在验收合格后 30 个工作日内支付款项", // 原文片段
      "semantic_similarity": 0.98, // 验证模型打分
      
      // 视觉定位 (Visual Grounding)
      "location": {
        "page_index": 2, // 0-indexed
        "bbox": [100, 450, 600, 500], // [x1, y1, x2, y2]
        "highlight_color": "yellow"   // 前端渲染用
      }
    }
  ]
}

```

> **Rule of Thumb**: 如果你的 Agent 是处理 PDF 的，**必须**存储 bbox。没有 bbox 的引用无法用于训练“多模态高亮”功能，也难以人工验证准确性。

---

## C.4 蒸馏数据构建：SFT 与 DPO

有了上面的 Trace，我们如何生成训练数据？

### 4.1 监督微调 (SFT) 数据构建

**目标**：教会模型“如何正确使用工具”和“遵循格式”。

**过滤逻辑 (The Filter Funnel)**：

1. **Status Filter**: 只保留 `status == "success"` 的 Trace。
2. **Length Filter**: 剔除交互轮数 < 2 的（太简单）或 > 50 的（可能陷入死循环）。
3. **Thought Check**: 剔除 `thought` 字段为空的样本（无法学习推理过程）。

**格式转换**:
将 Trace 打平为 `User -> Assistant (Thought + Call) -> Tool -> Assistant (Answer)` 的线性对话格式。

### 4.2 偏好数据 (DPO/RLHF) 构建

**目标**：教会模型“什么更好”，减少幻觉，提升安全性。需要构造 `(Prompt, Chosen, Rejected)` 三元组。

#### 策略 A: 自动构造 Rejected (Self-Rejection)

利用模型生成的错误 Trace。

* **Case**: 发生了 Python 语法错误，但最终通过重试修好了。
* **Chosen**: 将错误的中间步骤移除，拼接成一条“一次过”的完美路径。
* **Rejected**: 保留原始包含错误的路径（或者保留错误步骤作为反面材，视训练目标而定）。

#### 策略 B: 拒绝采样 (Rejection Sampling / Best-of-N)

对于同一个 Prompt，让模型生成 N=4 个不同的回答路径。

* **评分**: 使用 Reward Model 或 单元测试（针对代码 Agent）对 N 个结果评分。
* **配对**: 取分最高的为 Chosen，分最低的为 Rejected。

#### 策略 C: 幻觉植入 (Hallucination Injection) - *进阶*

* **Chosen**: 原始正确的 Trace。
* **Rejected**: 取正确的 Trace，人为修改其中的“事实性数据”（如将“利润增长 15%”改为“150%”），构造一条看起来通顺但事实错误的样本。这对训练 Factuality 极其有效。

---

## C.5 数据清洗与质量控制流水线

不要把垃圾喂给模型。以下是建议的 ETL 清洗步骤：

| 步骤 | 动作名称 | 详细操作与规则 |
| --- | --- | --- |
| **1** | **去重 (Dedup)** | 计算 User Prompt 的 MinHash。对于重复度 > 0.9 的请求，只保留 Outcome 评分最高的一条。 |
| **2** | **脱敏 (Scrub)** | **Regex 替换**：Email, Phone, IP Address, API Keys (sk-...)。<br>

<br>**实体替换**：将具体人名替换为 `[PERSON]`，公司名替换为 `[ORG]`（视任务而定，有时需要保留）。 |
| **3** | **毒性检测 (Detox)** | 使用轻量级分类器（如 Deberta-v3-small）扫描 `user_input` 和 `final_answer`。剔除色情、暴力、仇恨言论。 |
| **4** | **语言一致性** | 剔除中英文夹杂严重、乱码比例过高的文本。 |
| **5** | **代码/JSON 校验** | 提取所有 JSON 和 Code Block，尝试解析 (JSON.parse / AST parse)。无法解析的视为“格式错误样本”，作为 Negative Sample 或丢弃。 |
| **6** | **引用验证** | 检查 Trace 中的引用片段是否存在于源文档中。如果 `quote_text` 在原文找不到，标记为“幻觉样本”。 |
| **7** | **多模态对齐检查** | 检查 `image_url` 是否有效。如果是死链，该条数据必须丢弃，否则模型会学习到“无视图像”的行为。 |

---

## C.6 本章练习

### 基础题 (50%)

1. **Trace 补全**：
给定一个只有 `User` 和 `Final Answer` 的简单日志，请手动补充中间的 `Thought` 和 `Tool Call` 步骤，使其构成一个合法的 Chain-of-Thought Trace。任务：“查询 Google 股价并计算其市盈率。”
<details>
<summary>提示</summary>
你需要补全：1. 调用搜索工具查股价；2. 调用搜索工具查每股收益(EPS)；3. 调用计算器或代码解释器算出 PE ratio。注意 Thought 必须先于 Action。
</details>
2. **Schema 纠错**：
以下 JSON 片段哪里不符合本章定义的规范？
```json
{ "role": "function", "content": "Error: timeout" }

```


<details>
<summary>提示</summary>
1. 角色名应统一为 `tool` 而非 `function`（OpenAI 新版规范）。2. 缺少 `tool_call_id`，导致无法知道这是响应哪个调用的。3. 缺少 `name` 字段。
</details>


3. **脱敏实战**：
编写一个简单的 Python 正则表达式列表，用于从 Trace 字符串中 `sk-` 开头的 OpenAI Key 和 11 位手机号替换为 `[SECRET]`。

### 挑战题 (50%)

4. **多轮对话的 Context 截断策略**：
在构造 SFT 数据时，如果一个 Session 有 50 轮对话，直接放入训练会导致 Context Window 爆炸。请设计一个滑动窗口算法，既能保留 System Prompt 和最新的几轮对话，又能通过 Summary 机制保留早期的关键信息。请描述 Trace 中需要增加什么字段来存储这种 Summary？
<details>
<summary>提示</summary>
需要在 Meta 中增加 `running_summary` 字段。算法：保留第 0 轮（System），保留第 T-5 到 T 轮。第 1 到 T-6 轮的内容被压缩为一段文本，作为新的 System Prompt 的一部分插入。
</details>
5. **基于“修改距离”的 DPO 构造**：
设计一个自动化流程：取一个正确的 Python 代码生成 Trace (Chosen)。使用脚本自动修改代码中的一个变量名，使其产生 `NameError`。将运行报错后的 Trace 设为 Rejected。请分析这种成数据的优缺点。
<details>
<summary>提示</summary>
优点：低成本、大规模生成负样本。缺点：模型可能只学会了纠正简单的 NameError，而没有学到深层的逻辑错误；且这种负样本可能过于“弱智”，导致梯度下降收益极小。
</details>

---

## C.7 常见陷阱与错误 (Gotchas)

1. **Ghost References (幽灵引用)**
* **现象**：Trace 中记录了引用 `[1]`，但对应的 `citations` 列表里只有 `source_id`，没有具体的 `text_span` 或 `bbox`。
* **后果**：由于缺乏细粒度监督，蒸馏出的 Student 模型虽然会写 `[1]`，但指向的内容往往是随机的页面，造成“看起来有理有据的胡说八道”。
* **修复**：在 ETL 阶段强制校验：`assert len(citation.bbox) == 4`。


2. **Tool Output Contamination (工具输出污染)**
* **现象**：OCR 工具返回了大量乱码，或者搜索工具返回了 100kb 的 HTML 源码。直接将这些扔进 SFT 训练。
* **后果**：模型学会输出无意义的长文本，或者对 Context 长度极其敏感。
* **修复**：并在 Trace Schema 中引入 `summary` 字段。训练时使用 summary，但保留原始 content 用于审计。


3. **Trace Drift (Schema 漂移)**
* **现象**：代码库更新了 Tool 的参数名（如 `file_name` 改为 `filename`），但历史 Trace 还是旧的。
* **后果**：混合训练时，模型会混淆参数名，导致幻觉参数。
* **修复**：Trace 必须包含 `tool_version`。在构建数据集时，必须编写 Adapter 将旧 Trace 迁移到新 Schema，或者丢弃旧版本数据的工具调用部分。


4. **Only Training on Success (幸存者偏差)**
* **现象**：只使用最终成功的 Trace 进行 SFT。
* **后果**：模型从未见过“报错”的样子。一旦线上环境出问题（如网络波动），模型不知道如何重试或优雅降级，而是直接崩溃或死循环。
* **修复**：在 SFT 数据中混入 10% 的“Error -> Recovery -> Success” 样本，教模型鲁棒性。

