# 第 17 章 文档/票据/表格多模 RPA Agent：企业流程自动化

## 1. 开篇段落：从“读图”到“业务交付”

在企业数字化转型的浪潮中，最“脏、累、苦”却又最具商业价值的领域，莫过于**业务流程自动化（RPA, Robotic Process Automation）**。传统的 RPA 脚本脆弱不堪，一旦发票版式微调或网页 UI 变更，自动化流程就会崩溃。

本章我们将构建一个基于多模态大模型（VLM）的 **Cognitive RPA Agent（认知型 RPA）**。它不依赖死板的坐标定位，而是像人类一样具备“阅读理解”能力。它能处理模糊的手机照片、理解跨页的复杂表格、校验逻辑矛盾的财务数据，并将非结构化的文档转化企业 ERP 系统所需的严丝合缝的结构化数据（JSON/SQL）。

**本章学习目标：**

1. **架构设计**：搭建“OCR 辅助 + VLM 推理”的混合流水线。
2. **难点攻克**：掌握处理嵌套表格、跨页合并、印章遮挡等极端情况的策略。
3. **可靠性工程**：建立多层级校验（Verification）与人机回环（HITL）机制，确保 99.9% 的业务可用性。

---

## 2. 核心论述：构建高鲁棒性文档处理流水线

### 2.1 为什么纯 VLM 还不够？（The Hybrid Approach）

直接把一张密密麻麻的银行流水单丢给 GPT-4V 或 Claude 3.5 Sonnet，通常会遇到两个问题：

1. **幻觉（Hallucination）**：模型可能会凭空捏造一个看不清的数字。
2. **字符级精度（Character Accuracy）**：对于 18 位身份证号、20 位银行卡号或复杂的 SKU 编码，VLM 的 OCR 精度在某些字体下不如传统的专用 OCR 引擎（如 PaddleOCR, Tesseract 或商业 OCR API）。

因此，工业级的佳实践是 **Hybrid Pipeline（混合流水线）**。

**图 17.1：混合模态文档处理流水线**

```ascii
+-------------------+      +-------------------------+
| [Raw Input]       | ---> | [Preprocessing]         |
| Image / PDF / Scan|      | Deskew / Crop / Denoise |
+-------------------+      +-----------+-------------+
                                       |
           +---------------------------+---------------------------+
           | (Visual Stream)           | (Text Stream)             |
           v                           v                           v
+-----------------------+    +-----------------------+   +-------------------+
| [VLM - The Brain]     |    | [OCR Engine - The Eye]|   | [Layout Analysis] |
| Focus: Semantics      |    | Focus: Characters     |   | Focus: Structure  |
| "What is the Total?"  |    | "Recognize text"      |   | "Where is Table?" |
+----------+------------+    +-----------+-----------+   +---------+---------+
           |                             |                         |
           +-------------+---------------+-------------------------+
                         |
                         v
               +---------------------------+
               | [Context Fusion Layer]    |
               | Prompt: "Here is the img  |
               | AND the OCR text overlay" |
               +-------------+-------------+
                             |
                             v
               +---------------------------+
               | [Extraction & Normalize]  |
               | Output: JSON Schema       |
               +-------------+-------------+
                             |
                             v
               +---------------------------+
               | [Logic Gate / Validation] |
               | Sum Check / DB Lookup     |
               +---------------------------+

```

### 2.2 核心策略详解

#### 2.2.1 锚点定位与 ROI 切分 (Region of Interest)

对于长文档或复杂版面，直接全图输入会导致分辨压缩，丢失细节。

* **策略**：先用一个轻量级模型（如 YOLO 或即便是小参数的 VLM）识别关键区域（ROI），例如“发票明细栏”、“收件人信息区”。
* **Rule-of-Thumb**：永远不要让 VLM 在 1024x1024 的缩略图中去猜 6 号字体的小字。先切图（Crop），再识别。

#### 2.2.2 Schema-Driven Extraction (模式驱动提取)

不要对模型说“提取所有信息”，要告诉它具体的 JSON Schema。

* **类型约束**：明确字段是 `String`（保留原始格式）、`Float`（用于计算）还是 `Enum`（枚举值，如 "男/女"）。
* **空值处理**：明确指示当字段不存在或看不清时，是输出 `null`、`""` 还是 `N/A`。这对于后续清洗至关重要。

### 2.3 攻克“表格”难题 (The Table Boss Fight)

表格是文档理解中的“最终 BOSS”。由于缺乏显式的行列分隔符（无框线表格），或者存在合并单元格，模型极易“看串行”。

**三种有效的表格处策略：**

1. **Markdown 中间态法**：
* 要求 VLM 先将表格转换为 Markdown 格式（`| Col1 | Col2 |`）。VLM 在训练时见过大量 Markdown 数据，这种格式能很好地保留二维结构。
* *适用场景*：标准表格，线条清晰。


2. **坐标锚定法 (Coordinate Grounding)**：
* 要求模型输出每个单元格内容的 `box_2d` [x1, y1, x2, y2]。
* 后处理脚本通过计算 Y 轴中心点的重合度，将 Box 聚类为“行”；通过 X 轴重合度聚类为“列”。
* *适用场景*：错位严重的表格、手写表格。


3. **JSON Map 法 (Row-wise Extraction)**：
* 定义一个 `row_item` 的对象结构，要求模型输出一个 List。
* *技巧*：如果表格跨页，需要在 Prompt 中注入“Header Context”（即把第一页的表头文字强行加到第二页的 Prompt 中），否则模型不知道第二页的第一列代表什么。



### 2.4 校验与人机回环 (Validation & HITL)

在财务场景，**Confidence（置信度）** 是比 **Accuracy（准确率）** 更重要的指标。我们宁愿模型说“我不知道”，也不要它瞎猜。

**图 17.2：多层级校验漏斗**

```ascii
[Extracted JSON]
      |
      v
+-------------------------+
| Level 1: Syntax Check   |  --> Fail? --> [Retry with Reflection]
| (Date fmt, RegEx types) |
+-----------+-------------+
            | Pass
            v
+-------------------------+
| Level 2: Arithmetic     |  --> Fail? --> [Retry logic: "Recalculate"]
| (Qty * Price == Total?) |
+-----------+-------------+
            | Pass
            v
+-------------------------+
| Level 3: Business Logic |  --> Fail? --> [Flag for Human Review]
| (Supplier in DB? Valid?)|
+-----------+-------------+
            | Pass
            v
      [Auto-Approve]

```

* **自修复（Self-Correction）**：当算术校验失败（如总额不平），可以将错误信息反馈给 Agent：“你提取的明细加总为 100，但发票总额显示 102，请检查是否漏读了某一行或读错了数字” VLM 往往能自我修正。

---

## 3. 本章小结

* **架构观**：RPA Agent 不是单一的模型调用，而是 `Pre-process -> OCR/VLM Ensemble -> Logic Check -> Post-process` 的精密流水线。
* **数据观**：Schema 定义即开发。Schema 写得越详细（包含 description 和 pattern），提取效果越好。
* **底线思维**：必须假设模型会犯错。通过算术平衡（Math Balance）和数据库比对（Master Data Lookup）来构建安全网。
* **隐私观**：敏感字段（PII）应在本地或通过专用小模型进行掩码处理，再上传至云端大模型。

---

## 4. 练习题

### 基础题（熟悉流程与 Schema）

**练习 1：为混合票据设计通用 Schema**
假设你需要处理一个文件夹，里面混杂着“打车票”、“餐饮发票”和“酒店水单”。请设计一个兼容这三种票据的 JSON Schema，并包含分类字段。

* **提示**：使用 `oneOf` 结构或包含通用的 `metadata` 字段。

<details>
<summary>参考答案思路</summary>

```json
{
  "document_type": {
    "type": "string",
    "enum": ["taxi_receipt", "restaurant_invoice", "hotel_folio", "unknown"],
    "description": "首先判断票据类型"
  },
  "common_fields": {
    "date": "YYYY-MM-DD",
    "total_amount": "string", // 统一用字符串防止精度丢失
    "currency": "CNY"
  },
  "specific_data": {
    "taxi_info": {
      "start_time": "HH:MM",
      "end_time": "HH:MM",
      "distance_km": "number"
    },
    "invoice_info": { // 餐饮/酒店发票通用
      "seller_name": "string",
      "tax_code": "string",
      "invoice_code": "string",
      "invoice_number": "string"
    },
    "hotel_details": {
      "check_in": "YYYY-MM-DD",
      "check_out": "YYYY-MM-DD",
      "room_number": "string"
    }
  }
}

```

**解析**：先分类，再提取。利用 `common_fields` 提取所有票据都有的信息，利用 `specific_data` 存储特定业务字段。

</details>

**练习 2：算术校验逻辑编写**
提取到了个报销单的表格数据：`items: [{price: 10, qty: 2}, {price: 5, qty: 4}]`，以及总金额 `total: 45`。请写出一段伪代码逻辑，判断该提取结果是否可信，并生成错误报告。

<details>
<summary>参考答案思路</summary>

```python
# 伪代码逻辑
calculated_sum = 0
for item in items:
    # 转换为浮点数或定点数进行计算
    line_total = item.price * item.qty
    calculated_sum += line_total

tolerance = 0.05 # 允许5分的误差，处理四舍五入
diff = abs(calculated_sum - total)

if diff <= tolerance:
    return "PASS", "Data is consistent"
else:
    # 生成具体的错误原因，用于反馈给 Agent 或人类
    error_msg = f"Arithmetic Mismatch: Sum of lines is {calculated_sum}, but document total says {total}. Difference: {diff}"
    return "FAIL", error_msg

```

**关键点**：容差（Tolerance）是必须的，因为发票计算往往存在舍入规则差异。

</details>

**练习 3：Prompt 优化**
初级 Prompt：“请把图里的字读出来。”
请将其优化为适合提取**多语言混合名片**（中文、英文、日文）的 Expert Prompt。

<details>
<summary>参考答案思路</summary>

**优化后的 Prompt 结构：**

1. **角色定义**：你是一个精通多语言的 OCR 专家和数据结构化助手。
2. **任务描述**：从名片图像中提取联系人信息，并输出为 JSON。
3. **语言指令**：注意识别混合语言（中/英/日）。保留原文拼写，不要翻译人名或公司名。
4. **字段定义**：
* `name`: 全名。
* `title`: 职位（如果是多行，用 / 分隔）。
* `phone`: 优先提取手机号，格式化为 E.164。
* `email`: 必须包含 @ 符号。


5. **特殊处理**：如果名片包含正反两面（即两张图），请合并信息。
6. **输出示例**：给出期望的 JSON 样例。

</details>

---

### 挑战题（包括开放性思考题）

**练习 4：复杂嵌套表格的“分治法”**
处理一张复杂的工程物料清单（BOM），表格中包含“大类”（合并单元格，跨越 10 行）和“子项”。如果直接提取，模型往往会忘记第 5-10 行属于哪个“大类”。设计一个处理流程解决此问题。

<details>
<summary>参考答案思路</summary>

**策略：视觉增强 + 状态填充**

1. **预处理（视觉增强）**：使用 OpenCV 检测表格横竖线。如果检测到“合并单元格”，在送入模型前，通过图像处理算法在图像上绘制辅助线，或者直接把“大类”名称 `Copy` 并“印”在属于它的每一行的左侧空白处（Virtual Unmerging）。
2. **Prompt 策略（状态保持）**：指示模型：“这是一个嵌套表格。当你读取每一行时，如果第一列（大类）是空的，请自动继承上一行的值（Fill-down logic）。”
3. **后处理校验**：提取后，检查 JSON 数组，如果某行的 `category` 字段为空，代码层逻辑自动填入上一个非空值。

</details>

**练习 5：构建“自动纠错”的 Agent Loop**
设计一个 Agent 循环，当 VLM 提取的“发票日期”是 `2024-02-30`（无效日期）时，Agent 不直接报错，而是重新观察图片，并尝试自我修正。请画出流程图或写出伪代码。

<details>
<summary>参考答案思路</summary>

**Reflexion Loop (反思循环):**

1. **Act 1**: Agent 提取日期 -> "2024-02-30"。
2. **Validate**: 代码校验器检查 `isValidDate("2024-02-30")` -> False。
3. **Prompt Generator**: 构造新的 Prompt -> "之前提取的日期 '2024-02-30' 是无效的（二月没有30号）。请重新仔细查看图片的右上角日期栏。可能是 OCR 把 '03' 看成了 '30'，或者是 '28' 看成了 '30'？请给出新的置信度最高的猜测。"
4. **Act 2**: Agent 重新观察 -> 输出 "2024-02-03"。
5. **Validate**: 校验通过 -> 输出结果。
6. **Fallback**: 如果 3 次重试都失败，标记为 "Human Review Needed"。

</details>

**练习 6：对抗性测试（Red Teaming）**
如果用户上传了一张伪造的发票（如，PS 修改了金额，但字体有细微边缘锯齿；或者二维码被篡改）。设计一个“鉴伪 Agent”的思路。

* **提示**：不仅仅关注文本，还要关注图像特征。

<details>
<summary>参考答案思路</summary>

**多维鉴伪策略：**

1. **视觉一致性（Visual Consistency）**：微调一个 CV 模型（如 ResNet）专门检测“篡改痕迹”（Tampering Detection），寻找文字周围的伪影、背景纹理的不连续性。
2. **字体分析**：VLM 提示词：“请检查图中的数字字体是否一致？是否有某一行数字的大小、颜色深度与其他行明显不同？”
3. **逻辑验证（查验）**：
* 提取发票二维码内容，解码后与票面明文信息（金额、代码、号码）比对。
* 调用国税局 API（如果可用）进行真伪核验。


4. **EXIF/元数据分析**：检查图片是否由 Photoshop 保存，而非相机拍摄（虽然可被擦除，但可作为线索）。

</details>

**练习 7：隐私规的本地化方案**
某银行客户要求：所有的客户姓名和账号绝对不能传给 OpenAI/Anthropic。但你需要用 GPT-4 来分析银行流水的消费习惯。请设计架构。

<details>
<summary>参考答案思路</summary>

**Local-Cloud Hybrid Architecture (本地-云端混合架构):**

1. **Step 1 (Local)**: 使用本地部署的轻量级 OCR (PaddleOCR) 或开源 VLM (如 Qwen-VL-Chat 7B INT4)，在本地服务器运行。
2. **Step 2 (Redaction)**: 本地模型提取所有文本及其坐标。通过正则匹配（Regex）定位“姓名”和“账号”区域。
3. **Step 3 (Masking)**: 在原始图像上，将定位到的区域涂黑（Black out）。同时，在提取的文本中，将这些字段替换为 `<REDACTED_NAME_1>`, `<REDACTED_ACCOUNT_1>`。
4. **Step 4 (Cloud)**: 将涂黑后的图像 + 脱敏后的文本发送给 GPT-4。Prompt：“请分析用户 `<REDACTED_NAME_1>` 的消费趋势...”。
5. **Step 5 (Mapping)**: 得到分析结果后，在本地将 `<REDACTED_NAME_1>` 映射回真实姓名（仅在最后展示给授权员工时）。

</details>

---

## 5. 常见陷阱与错误 (Gotchas)

| 陷阱 (Gotcha) | 现象描述 | 调试与解决技巧 |
| --- | --- | --- |
| **JSON 截断** | 长文档（如 50 行的清单）导致 VLM 输出的 JSON 在中间断开，格式损坏。 | 1. **流式修复**：使用专门的 Parser 尝试补全末尾的 `]}`。<br>

<br>2. **分块策略**：按页或按视觉块（每 10 行）切分图片，分别提取后再合并。<br>

<br>3. **Output Token Limit**：检查模型配置的 `max_tokens` 是否太小。 |
| **0 与 O / 1 与 l / 8 与 B** | 最经典的 OCR 错误，尤其在验证码或哈希值中。 | **Context Is King**。不要只让模型看字符，要给语境。Prompt: "这是一个十六进制代码，只包含 0-9 和 A-F，请根据此规则纠正 OCR 错误。" |
| **方向错误** | 用户上传了倒置（旋转 180 度）的发票，VLM 乱读一气。 | 在 Pipeline 第一步加入 **Orientation Detection**（方向检测）。如果置信度显示图片旋转了，先用 OpenCV 转正，再送入 VLM。 |
| **多余的解释** | 要求输出 JSON，但模型喜欢在前面加 "Here is the JSON:"，导致代码解析失败。 | 1. 强制使用 `json_object` 模式（OpenAI API）。<br>

<br>2. 在 System Prompt 强调：`Output raw JSON only. No markdown formatting, no intro text.`<br>

<br>3. 代码层直接用正则提取第一个 `{` 和最后一个 `}` 之间的内容。 |
| **日期格式地狱** | 同样是 `01/02/2023`，是 1 月 2 日还是 2 月 1 日？ | **必须指定输出格式**。Prompt: "Extract all dates and normalize them to ISO 8601 format (YYYY-MM-DD). If ambiguous, infer from the country context (e.g., US format vs. UK format)." |
