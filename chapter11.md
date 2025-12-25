# 第 11 章 DeepResearch 智能体：多模态研究与长文档 PDF

> **本章摘要**
> 构建一个能够进行“深度研究”的智能体，远比搭建一个简单的问答机器人复杂。它需要处理长达数百页的非结构化 PDF（包含双栏排版、跨页表格、复杂图表），需要在海量噪声中提取“原子级证据”，更需要像人类分析师一样处理相互冲突的信息，并生成带有精确引用的专业报告。
> **核心差异**：
> * **Search (搜索)**：找到包含关键词的片段。
> * **Research (研究)**：阅读  理解  关联  验证  综合  撰写。
> 
> 
> **学习目标**
> 1. 构建一套**视觉优先（Vision-First）**的多模态文档解析流水线，解决 PDF “解析即损失”的难题。
> 2. 掌握**分层索引（Hierarchical Indexing）**与**假设性问题（Hypothetical Questions）**嵌入策略。
> 3. 设计**多智能体协作流**：规划员（Planner）、搜集员（Collector）与主笔者（Writer）。
> 4. 实现**零幻觉引用系统**：确保生成的每一句话都能通过超链接回溯到原始 PDF 的具体坐标。
> 
> 

---

## 11.1 任务定义：从“模糊需求”到“结构化洞察”

DeepResearch 任务通常始于一个宏大的、非结构化的问题，终于一份结构化的、可验证的交付物。

| 维度 | 普通 RAG (Q&A) | DeepResearch Agent |
| --- | --- | --- |
| **输入** | "Model Y 的续航是多少？" | "分析 2024 年固态电池技术突破及其对电动车成本结构的影响。" |
| **数据源** | 单一文档或短文本 | 20+ 份 PDF（财报、白皮书、论文），混合图与数据表。 |
| **推理深度** | 单步检索（Single-hop） | 多步推理（Multi-hop）、跨文档比较、时序分析。 |
| **输出** | 一段话答案 | 包含摘要、目录、数据图表、带有精确引用的 3000 字报告。 |
| **容错率** | 容忍少量概括 | **零容忍**：数据必须与原始报表一致，引用不可造假。 |

### 11.1.1 核心挑战：非结构化数据的熵

PDF 是为打印而生的格式，不是为机器阅读而生的。

* **视觉语义丢失**：双栏排版被错误合并、页眉页脚切断正文、表格变成乱码字符。
* **图表黑箱**：关键趋势数据往往在折线图中，普通 OCR 无法提取。
* **上下文碎片化**：答案的“主语”在第 1 页，而“谓语”和“宾语”可能跨越到了第 2 页。

---

## 11.2 多模态文档解析流水线 (The "Vision-First" Pipeline)

为了解决上述挑战，不能仅依赖 PyPDF2 或 LangChain 的默认 Loader。我们需要构建一个**像人类一样“看文档**的流水线。

### 11.2.1 架构总览

```mermaid
graph TD;
    A[原始 PDF] --> B{视觉版面分析 VLA};
    B -- 文本区域 --> C[OCR / 文本提取];
    B -- 表格区域 --> D[表格结构化处理];
    B -- 图表区域 --> E[多模态图表摘要];
    B -- 标题/页码 --> F[元数据过滤];
    C & D & E --> G[逻辑重组器];
    G --> H[多模态分块 (Chunks)];

```

*(注：此处使用文字描述逻辑，实际文档中可使用上述逻辑)*

**处理流程详解：**

1. **光栅化 (Rasterization)**：将 PDF 每一页渲染为高分辨率图像（建议 300 DPI）。
2. **版面分析 (Layout Analysis)**：
* 使用目标检测模型（如 YOLO, LayoutLM）识别区块：`Header`, `Footer`, `Text`, `Title`, `Table`, `Figure`, `Caption`。
* **Rule of Thumb**: 必须将 `Caption`（图注）与对应的 `Figure`（图片）绑定，否则图片将失去语境。


3. **分流处理 (Router)**：
* **纯文本**：使用 OCR 或 PDF 文本层提取。
* **表格 (Tables)**：**这是最难的部分**。不要试图用 OCR 直接读。
* *策略*：将表格截图送入 VLM（如 GPT-4o, Claude 3.5 Sonnet），Prompt 要求输出为 Markdown 或 HTML 格式。
* *Prompt 技巧*："Transcribe this table to Markdown. Preserve all headers. If a cell is merged, repeat the value."


* **图表 (Charts)**：
* *策略*：**密集描述 (Dense Captioning)**。
* *Prompt*："分析这张图表。1. 读取标题。2. 提取 X 轴和 Y 轴的单位与范围。3. 描述数据的趋势（上升/下降）。4. 提取关键数据点（如峰值、拐点）。"




4. **逻辑重组 (Logical Reconstruction)**：
* 基于版面坐标，将双栏文本按阅读顺序排序（左栏  右栏）。
* 将表格和图表的描述文本插入到原文中原本出现的位置。



> **工程陷阱 (Gotcha)**：
> **页眉页脚的干扰**。每一页重复出现的 "Confidential - Project X" 会严重干扰向量检索（Retriever 认为所有页面都高度相关）。
> **解决方案**：用版面分析的 BBox 坐标，物理剔除页面顶部和底部的 10% 区域，或通过正则去重。

---

## 11.3 索引与检索策略：为“研究”而生

简单的 `Chunk size = 500` 切分策略在 DeepResearch 中是灾难性的。我们需要**保留语境**。

### 11.3.1 混合索引结构 (Hybrid Indexing)

我们需要建立三层索引：

1. **父文档索引 (Parent Document Index)**：
* 对象：整份 PDF 的摘要。
* 用途：路由。当用户问“CATL 的产能”时，只去查 CATL 的财报，不查 Tesla 的手册。


2. **语义块索引 (Semantic Chunk Index)**：
* 对象：按章节或语义边界切分的文本块（Chunk）。
* 内容：`Original Text` + `Augmented Data`。
* **增强数据 (Augmentation)**：这是关键。
* **Q-to-P (Question-to-Passage)**：让 LLM 为这个 Chunk 生成 3 个它能回答的“假设性问题”。将这些问题向量化，而非仅向量化原文。这能显著提升召回率。




3. **图像/表格索引 (Visual Index)**：
* 对象：图表和表格的“密集描述”。
* 用途：回答“销量趋势如何”这类需要视觉聚合的问题。



### 11.3.2 证据原子 (The Evidence Atom)

在 Agent 的工作记忆中，不要存储大段文本，而是存储标准化的“证据原子”。

```json
{
  "evidence_id": "ev_8a7b2c",
  "content": "2023年Q4，公司毛利率下降至18.5%，主要受锂价波动影响。",
  "source_doc": "financial_report_2023.pdf",
  "page_num": 42,
  "bbox": [100, 200, 500, 300], // 在原图上的坐标，用于高亮显示
  "timestamp": "2023-12-31",    // 用于时序冲突仲裁
  "confidence": "high"          // 来源于官方财报 vs 第三方博客
}

```

---

## 11.4 多智能体协作架构 (The Researcher-Writer Pattern)

DeepResearch 任务过于复杂，单一 Prompt 容易导致上下文溢出或逻辑混乱。推荐采用 **Map-Reduce** 变体架构。

### 11.4.1 角色定义

1. **Planner (主编)**：
* 输入：用户模糊需求。
* 职责：需求澄清拆解子任务（Sub-questions）、生成大纲。
* 输出：一个 DAG（有向无环图）任务列表。


2. **Collector (搜集员 - 可并行实例)**：
* 输入：一个具体的子问题（如“查询 BYD 2023 销量”）。
* 工具：`vector_search`, `keyword_search`, `read_full_page`。
* 职责：在文档海中打捞“证据原子”。它**不负责写**，只负责**找**。


3. **Analyst (分析师)**：
* 输入：多个搜集员找回的证据堆（Pile of Evidence）。
* 职责：**冲突消解 (De-confliction)**。
* *场景*：搜集员 A 找到“销量 300万”，搜集员 B 找到“销量 302万”。分析师需对比证据的 `timestamp` 和 `source_reliability`，决定采信哪一个，或在报告中注明差异。


4. **Writer (主笔)**：
* 输入：大纲 + 经过清洗的证据链。
* 职责：将证据转化为流畅的文本，并**强制植入引用 ID**。



---

## 11.5 核心算法：引用与冲突处理

### 11.5.1 零幻觉引用 (Zero-Hallucination Citations)

如何保证 `[1]` 真的对应那一句话？

* **错误做法**：让模型先写完全文，再通过相似度搜索去匹配引用。这会导致“张冠李戴”。
* **正确做法**：**In-context Citation (上下文中引用)**。
* 在 Prompt 中，将检索到的证据标记为：
`[ID: doc1_p5] content...`
`[ID: doc2_p8] content...`
* 强制模型在生成结论时，必须携带 ID：
`Generated: 2023年销量创新高 [doc1_p5]，但利润率有所下滑 [doc2_p8]。`
* **后处理**：在渲染给用户前，将 `[doc1_p5]` 替换为可点击的数字 `[1]`，并生成对应的悬浮窗或侧边栏链接。



### 11.5.2 时序冲突仲裁 (Temporal Arbitration)

DeepResearch 经常遇到新旧数据打架的问题。

* **Rule of Thumb**: "Recency Bias" (近因效应) 策略。
* 如果问题涉及“当前状态（Current Status）”，按 `Data_Timestamp` 排序，取最新。
* 如果问题涉及“历史演变（Evolution）”，保留所有数据点，绘制趋势。
* **实现**：在 Metadata 中必须提取 `Date`。如果 PDF 没有明确日期，尝试从文件名或第一页提取。

---

## 11.6 练习题

### 基础题 (50%)

**Q1. 版面感知的 Token 优化**

> **场景**：你有 100 份 PDF，全部转图片喂给 GPT-4o 极其昂贵。但是你又不想放弃图表信息。
> **问题**：设计一个具体的过滤算法，在进入 VLM 之前，筛选出真正“值得看”的页面或区域。
> <details><summary><strong>点击查看提示</strong></summary>

> 提示：大多数 PDF 页面是纯文字。纯文字用 OCR 就够了，不需要 VLM。你需要一个轻量级分类器。
> </details>
> <details><summary><strong>点击查看参考答案</strong></summary>

> **策略：分级漏斗 (Tiered Funnel)**
> 1. **L0 (Layout Detection)**: 使用轻量级模型 (如 YOLOv8-nano, CPU 可运行) 扫描页面。
> 2. **判别逻辑**：
> * 如果页面只包含 `Text` 类：走 OCR 通道（成本  0）。
> * 如果页面包含 `Chart`, `Table` 类将该区域（BBox）裁剪出来。
> 
> 
> 3. **L1 (Vision Processing)**: 仅将裁剪下来的 `Chart/Table` 图片发送给 VLM。
> 4. **效益**：通常 PDF 中图表区域仅占总面积的 5%-10%，此法可节省 90% 的 Vision Token 成本。
> 
> 
> </details>

**Q2. 表格的 Markdown 还原**

> **场景**：OCR 提取的表格变成了乱序的字符串。
> **问题**：请写出一段 Prompt，指导 VLM 将一张表格截图转换为结构完美的 Markdown，并处理“合并单元格”这一棘手情况。
> <details><summary><strong>点击查看提示</strong></summary>

> 提示：你需要显式地告诉模型如何处理空缺值，以及当一个单元格跨越多行时该怎么做。
> </details>
> <details><summary><strong>点击查看参考答案</strong></summary>

> **Prompt 示例**：
> "You are a data entry expert. Convert the provided table image into a Markdown table.
> Rules:
> 1. Preserve all headers exactly as shown.
> 2. **Handling Merged Cells**: If a cell spans multiple rows (vertically merged), repeat the value in each row. Do not leave it empty.
> 3. If a cell is visually empty but implies a value from above (ditto), fill it.
> 4. If the image is blurry, output `[UNREADABLE]` for that cell.
> 5. Output ONLY the markdown text."
> 
> 
> </details>

**Q3. 引用 ID 的持久化**

> **场景**：Writer Agent 生成了初稿，包含 `[doc_id_123]`。然后 Editor Agent 进行了润色，删减了一些段落。
> **问题**：如何确保最终输出的参考文献列表（Bibliography）只包含正文中实际保留下来的那些引用？
> <details><summary><strong>点击查看提示</strong></summary>

> 提示：这类似于编程中的“垃圾回收（GC）”机制。
> </details>
> <details><summary><strong>点击查看参考答案</strong></summary>

> **流程**：
> 1. **全集维护**：在 Context 中维护一个字典 `Reference_Dict = {id: metadata}`，包含所有检索到的证据。
> 2. **正则扫描**：在 Editor Agent 输出最终文本后运行正则匹配 `\[doc_id_([a-z0-9]+)\]`。
> 3. **重构列表**：提取所有**命中**的 ID，从 `Reference_Dict` 中取出对应条目，生成最终的参考文献列表。
> 4. **重编号**：将正文中的 UUID `[doc_id_123]` 替换为阅读顺序的 `[1]`, `[2]`...，同时更新底部的列表。
> 
> 
> </details>

### 挑战题 (50%)

**Q4. 跨页表格的“缝合手术”**

> **场景**：一个长表格从第 10 页底部延伸到第 11 页顶部。第 11 页的部分没有表头（Headers），只有数据。单独看第 11 页无法理解每一列的含义。
> **问题**：设计一个算法逻辑，自动检测并修复这种跨页表格。
> <details><summary><strong>点击查看提示</strong></summary>

> 提示：你需要关注页面底部的 Layout 元素和下一页顶部的 Layout 元素。
> </details>
> <details><summary><strong>点击查看参考答案</strong></summary>

> **逻辑流程**：
> 1. **检测断裂**：若 Page N 的最后一个元素是 `Table` 且未合（没有底部边框或 Footer 阻隔），且 Page N+1 的第一个元素是 `Table` 且没有 `Header`。
> 2. **上下文注入 (Context Injection)**：
> * 提取 Page N 表格的 Header 区域图片。
> * 在处理 Page N+1 的表格截图时，将 Page N 的 Header 图片拼接在顶部（Vision 层面拼接），或者在 Prompt 中输入提取出的 Header 文本。
> 
> 
> 3. **合并输出**：告诉模型“这是上一页表格的延续，请使用相同的列结构解析数据”。
> 
> 
> </details>

**Q5. “幻觉引用”的红队测试 (Red Teaming)**

> **场景**：你需要构建一个自动评测脚本，专门抓包 Agent 瞎编引用的情况。
> **问题**：给定 (生成的句子 S, 引用来源原文 P)，如何判断 S 是否忠实于 P？请描述这个验证器的逻辑。
> <details><summary><strong>点击查看提示</strong></summary>

> 提示：这本质上是一个自然语言推理（NLI）任务。
> </details>
> <details><summary><strong>点击查看参考答</strong></summary>

> **验证器设计 (Verifier)**：
> 1. **原子化**：提取句子 S 中的核心主张 (Claim)。
> 2. **召回原文**：根据引用 ID 获取原始片段 P (Ground Truth)。
> 3. **NLI 判别**：调用一个小模型（如 GPT-4o-mini），Prompt 为：
> "Premise: {P}
> Hypothesis: {S}
> Does the premise entail the hypothesis?
> Options: [Entailment (支持), Contradiction (矛盾), Neutral (无关)]"
> 4. **判定**：
> * **Entailment**: 通过。
> * **Contradiction**: 严重错误（篡改数据）。
> * **Neutral**: 幻觉（原文没提这事）。
> 
> 
> 
> 
> </details>

**Q6. 开放思考：多模态“大海捞针”**

> **场景**：用户问：“这款发动机的扭矩曲线在多少转达到峰值？” 答案不在文字里，而在第 50 页的一张无标题的图片曲线图中。
> **问题**：普通的 RAG 根本检索不到这张图（因为没有文本匹配）。如何设计索引让 Agent 能找到它？
> <details><summary><strong>点击查看提示</strong></summary>

> 提示：你不能检索像素。你必须把“视觉特征”翻译成“可检索的文本”。
> </details>
> <details><summary><strong>点击查看参考答案</strong></summary>

> **方案：Dense Captioning + Hypothetical Questions**
> 1. **预处理**：对文档中所有 Chart 运行 VLM。
> 2. **提取内容**：不仅生成 Caption，还要提取数据特征。
> * *Caption*: "Engine torque vs RPM curve chart."
> * *Data Features*: "Peak torque approx 400Nm at 3500 RPM. Drops to 300Nm at 6000 RPM."
> 
> 
> 3. **假设性提问**：让 LLM 基于图表内容生成问题："What is the peak torque RPM?"
> 4. **索引**：将 "Peak torque 3500 RPM" 和生成的“问题”存入向量库。
> 5. **检索**：用户的提问将直接匹配到这些隐式生成的文本，从而“捞”出这张图。
> 
> 
> </details>

---

## 11.7 常见陷阱与错误 (Gotchas)

1. **OCR 的 "l" 与 "1" 之殇**：
* **现象**：财务报表中的 `1,000` 经常被 OCR 识别为 `l,000` 或 `I,000`。
* **Fix**：在写入数据库前，对纯数字列进行正则清洗，或者使用专门针对数字优化的 OCR 引擎。


2. **丢失的上下文 (Context Fragmentation)**：
* **现象**：Chunk 切分切断了代词。Chunk 2 开头是“它增长了 20%”。检索时根本不知道“它”是谁。
* **Fix**：**Window Overlap (重叠窗口)** 是必须的。或者使用 **Document Summary Context**，即在每个 Chunk 前面强行拼上“本文档是关于 Tesla 2023 财报的”。


3. **过度自信的“无中生有”**：
* **现象**：当文档里找不到答案时，模型倾向于用它预训练的知识回答（可能是 2 年前的过时数据）。
* **Fix**：System Prompt 必须包含：“Strictly answer based ONLY on the provided context. If not found, state 'Evidence missing in documents'.”


4. **引用漂移 (Reference Drift)**：
* **现象**：模型在总结时，把 [1] 的结论安到了 [2] 的头上。
* **Fix**：降低 Temperature（至 0.1-0.2），并在 Prompt 中强调引用的原子性绑定。


