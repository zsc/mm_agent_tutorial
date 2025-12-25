# 第 3 章 Tool Call：工具调用设计与编排

## 3.1 开篇：从“大脑”到“手脚”

如果说大模型（LLM/VLM）是 Agent 的大脑，那么**工具（Tools）** 就是它的手、脚、眼睛和耳朵。没有工具，模型只是一个被困在 GPU 显存里的、知识渊博但行动瘫痪的哲学家。

**Tool Call（工具调用）**，在某些框架中也被称为 Function Calling，是 Agent 跨越“语义世界”与“物理/数字世界”鸿沟的唯一桥梁。它不仅仅是简单的 API 调用，而是一套完整的**协议**：模型根据上下文判断意图，生成符合规范的结构化指令，宿主系统执行指令，并将结果（无论是文本、图片还错误堆栈）回传给模型，从而形成闭环。

### 本章学习目标

1. **掌握 Schema Engineering**：如何通过精准的接口定义（JSON Schema）控制模型的行为，减少幻觉。
2. **精通编排模式**：理解串行（Chain）、并行（Fan-out）、以及异步轮询等复杂的调用拓扑。
3. **多模态工具设计**：学会如何在工具调用中传递“非文本”信息（如图片引用、音频片段）。
4. **生产级鲁棒性**：处理超时、脏数据、上下文溢出以及死循环的工程策略。

---

## 3.2 工具调用的核心机制

### 3.2.1 交互协议解构

一个标准的 Tool Call 并不是“模型直接运行代码”，而是“模型生成文本 -> 系统解析并运行 -> 系统注入结果”的三明治结构。

**ASCII 流程图：工具调用的生命周期**

```text
       [Context / History]
               |
+--------------v--------------+
|   LLM / VLM (Reasoning)     | <--- "这个用户想查库存，我需要调用 inventory_tool"
+--------------+--------------+
               |
      (Output: Structured Intent)
               v
+--------------+--------------+
|   Orchestrator (System)     | <--- 1. 拦截模型输出
|   - Parse & Validate JSON   | <--- 2. 校验参数类型 (Schema Validation)
|   - Check Permissions       | <--- 3. 权限风控
+--------------+--------------+
               |
      (Execute: API / DB / Code)
               v
+--------------+--------------+
|    Real World / Environment | <--- 执行动作 (副作用发生处)
+--------------+--------------+
               |
       (Result: "Count: 42")
               v
+--------------+--------------+
|   Orchestrator (System)     | <--- 4. 格式化结果 (截断、清洗)
|   - Format as ToolMessage   |
+--------------+--------------+
               |
      (Input: Append to History)
               v
+--------------v--------------+
|   LLM / VLM (Synthesis)     | <--- "根据库存数据，我们可以..."
+-----------------------------+

```

### 3.2.2 Schema 即 Prompt (Schema Engineering)

在 Agent 开发中，**接口定义（Function Signature）就是 Prompt 的一部分**。模型读不懂 Python 代码逻辑，它读的是你提供的 JSON Schema 描述。

一个高质量的 Schema 包含三个维度：

1. **语义描述 (Description)**：不仅要写“这个工具做什么”，还要写“**何时使用**”、“**典型用例**”以及“**返回什么**”。
* *Bad*: `search_web: 搜索网络`
* *Good*: `search_web: 当用户询问 2023 年之后发生的实时事件、新闻或具体事实（如股价、天气）时使用。返回结果包含标题、URL 和页面摘要。不要用于回答通用常识。`


2. **参数约束 (Constraints)**：利用 JSON Schema 的 `enum`（枚举）和 `pattern`（正则）来物理限制模型的发挥空间。
* *Rule of Thumb*: 能用 `enum` 的地方绝不让模型填空。能用 `number` 的地方绝不让模型填 `string`。


3. **示例注入 (Few-shot in Docstring)**：对于复杂参数（如 SQL 语句或 Pandas 查询），在 description 中直接给出 valid/invalid 的 example 是最有效的。

---

## 3.3 编排模式：串行、并行与动态规划

随着任务复杂度提升，单一的工具调用不再够用。我们需要设计不同的编排模式。

### 3.3.1 串行链 (Sequential Chain)

这是最自然的“思考-行动-观察”循环。

* **模式**：`Step 1 (Find ID) -> Step 2 (Get Details) -> Step 3 (Process)`
* **适用**：后一个工具的输入强依赖前一个工具的输出。
* **瓶颈**：延迟高，每一步都需要一轮完整的 LLM 推理。

### 3.3.2 并行扇出 (Parallel Tool Use / Fan-out)

现代模型（如 GPT-4, Claude 3.5）支持一次输出多个工具调用。

* **场景**：用户问“比较特斯拉和比亚迪的财报”。
* **行为**：
```json
// 模型一次性输出：
[
  {"name": "get_report", "args": {"company": "Tesla"}},
  {"name": "get_report", "args": {"company": "BYD"}}
]

```


* **工程实现**：宿主程序使 `asyncio` 或线程池并发执行这两个请求，收集所有结果后，拼接成一个包含两个 ToolMessage 的列表返回给模型。
* **收益**：将 `N` 次网络延迟压缩为 `1` 次。

### 3.3.3 异步与长任务 (Async / Long-running)

有些工具执行很慢（如“训练一个模型”或“深度扫描 PDF”），不能让 HTTP 连接挂起等待。

* **模式**：**提交-轮询（Submit-Poll）** 或 **回调（Callback）**。
* **工具设计**：
1. `start_task(...)` -> 返回 `task_id`，立即结束本次 Tool Call。
2. `check_status(task_id)` -> 模型在数轮对话后，或者通过系统定时的“唤醒机制”去查询状态。


* **Rule of Thumb**：对于超过 30 秒的任务，必须设计成异步模式，否则 HTTP 超时会导致模型以为任务失败并触发无限重试。

---

## 3.4 多模态工具设计 (Multimodal Tools)

在多模态 Agent 中，工具的输入输出不再局限于文本。

### 3.4.1 图像作为输入 (Image-in)

当模型要“看”图并操作时（例如：`crop_image(bbox)` 或 `verify_signature(image)`）。

* **引用传递 (Pass by Reference)**：**永远不要**把 Base64 编码的图片在 Prompt 和工具参数间来回传递。这会瞬间耗尽 Token 预算并增加延迟。
* **最佳实践**：
* 模型上下文中：图片以 `image_id` 或 `url` 的形式存在。
* 工具调用时：模型输出 `{"tool": "crop", "args": {"image_id": "img_123", "bbox": [0,0,100,100]}}`。
* 执行层：根据 `image_id` 从内存/缓存中提取实际像素数据传给 OpenCV/PIL。



### 3.4.2 图像作为输出 (Image-out)

当工具生成了图表、渲染了 PDF 或修改了图片。

* **视觉占位符**：工具返回结果不应是二进制流，而是一个描述性结构：
```json
{
  "status": "success",
  "generated_image_id": "img_new_456",
  "description": "A line chart showing revenue growth...",
  "preview_url": "https://..."
}

```


* **多模态闭环**：宿主系统检测到工具返回了 `image_id`，自动将其渲染到用户的 UI 上，同时将该图片作为**新的多模态消息**注入到模型的下一轮对话历史中，让模型“看到”自己的工作成果。

---

## 3.5 鲁棒性与错误处理 (Engineering Reliability)

### 3.5.1 容错与自修复 (Self-Correction)

模型生成的参数经常会出错（格式错误、幻觉参数）。

* **策略**：把错误当成反馈。
* **流程**：
1. 工具执行抛出 Python Exception（如 `ValueError: 'aple' is not a valid stock symbol`）。
2. 系统捕获异常，不要崩溃。
3. 系统将异常信息作为 `ToolMessage` (Role: Tool) 返回给模型。
4. 模型看到错误信息，进行“反思”，并在下一轮输出修正后的调用 `{"symbol": "AAPL"}`。



### 3.5.2 结果截断与摘要 (Truncation & Summary)

工具（如 `cat file.txt`）可能返回 10MB 的文本。

* **硬截断**：设置阈值（如 2000 tokens）。超过部分显示 `...[content truncated, total length 50000 chars]`。
* **智能摘**：如果超长，系统自动调用一个小模型（如 GPT-3.5-Turbo 或 Haiku）对工具输出进行摘要，再返回给主模型。
* **文件句柄**：如果内容过长，工具应返回 `File saved to context memory (id: doc_1). Use 'read_chunk(doc_1, start_line)' to read details.`

### 3.5.3 幂等性与副作用 (Idempotency)

* **风险**：网络超时导致模型重试，结果用户被扣了两次款。
* **设计**：对于有副作用的工具（Create, Update, Delete, Pay），必须强制要求 `idempotency_key` 参数，或在系统层通过 `trace_id` 进行去重拦截。

---

## 3.6 常见陷阱与错误 (Gotchas)

| 陷阱 (Gotcha) | 典型症状 | 解决方案 (Rule of Thumb) |
| --- | --- | --- |
| **工具争抢 (Tool Confusion)** | 有两个相似工具（如 `search_google` 和 `search_wiki`），模型随机乱用或犹豫不决。 | **合并同类项**。做一个通用的 `search(query, source="auto")`，让代码内部去路由，而不是让模型纠结。或者在 description 中明确界定边界。 |
| **参数依赖丢失** | 模型想调用 `summarize(text)`，但直接把上一轮对话里 5000 字的内容塞进参数，导致输入 Token 爆炸。 | **强制引用**。Schema 中要求 `document_id` 而不是 `content`。强迫模型先调用 `upload` 或从上下文引用 ID。 |
| **无限循环 (The Loop of Death)** | 模型调用工具 -> 报错 -> 重试 -> 报错 -> 重试... 耗尽额度。 | **熔断机制**。系统层记录连续错误次数。如果同一个工具连续失败 3 次，强制插入 System Prompt：“该工具已损坏，请停止尝试并告知用户失败原因”。 |
| **JSON 格式灾难** | 模型输出的 JSON 包含单引号、未转义的换行符，或者 Python 风格的 `True` (而不是 `true`)。 | 1. 使用容错性强的解析库（如 `json5` 或 `dirty-json`）。<br>

<br>2. 在 System Prompt 中强调“Strict JSON”。<br>

<br>3. 使用 Pydantic 的 Validator 做清洗。 |
| **幽灵工具调用** | 模型在没注册任何工具的情况下，依然幻觉出 `<tool_code>...`。 | 这是训练数据的残留。在 Prompt 中明确“如果不需要使用工具，直接回答”。 |

---

## 3.7 练习题

### 基础题 (50%)

<details>
<summary><strong>练习 1：精确的 Schema 定义</strong></summary>

**题目**：为一个 `get_customer_info` 工具编写 JSON Schema。
要求：

1. 用户可以通过 `id` (int) 或 `email` (string) 查询。
2. 必须且只能提供其中一个。
3. `email` 必须符合邮箱格式。

**提示**：

* 思考 JSON Schema 的 `oneOf` 属性。
* 如何用 `pattern` (正则) 约束 email？

**参考答案思路**：

```json
{
  "name": "get_customer_info",
  "description": "根据 ID 或 Email 获取客户详情。ID 和 Email 必须二选一。",
  "parameters": {
    "type": "object",
    "properties": {
      "id": {"type": "integer", "description": "客户的唯一数字 ID"},
      "email": {"type": "string", "format": "email", "description": "客户注册邮箱"}
    },
    "oneOf": [
      {"required": ["id"]},
      {"required": ["email"]}
    ]
  }
}

```

</details>

<details>
<summary><strong>练习 2：解释并发调用的优势</strong></summary>

**题目**：假设你有两个工具 `tool_A` (耗时 2s) 和 `tool_B` (耗时 3s)。

1. 如果串行执行，模型获得结果需要多久？
2. 如果并行执行，模型获得结果需要多久？
3. 在什么场景下，你**不能**使用并行调用？

**提示**：

* 关键路径分析。
* 数据依赖性。

**参考答案思路**：

1. 串行：约 5s + 两次模型推理时间。
2. 并行：约 3s (取最大值) + 一次模型推理时间。
3. 当 `tool_B` 的输入参数依赖于 `tool_A` 的输出结果时（例如：A 查 ID，B 用 ID 查详情），绝对不能并行。

</details>

<details>
<summary><strong>练习 3：错误处理模拟</strong></summary>

**题目**：用户问“把这个文件发给 jack”。模型调用 `send_file(user="jack")`。工具返回错误 `404 User 'jack' not found. Did you mean 'jack@company.com'?`。
请描述接下来 Agent 内部会发生什么？

**提示**：Context 此时变成了什么样？模型看到了什么？

**参考答案思路**：

1. 系统将错误信息封装为 `ToolMessage(content="Error: 404...", status="error")` 存入历史。
2. LLM 读取历史，看到自己的调用和对应的错误提示。
3. LLM 进行推理：“我应该使用完整邮箱”。
4. LLM 生成新的调用 `send_file(user="jack@company.com")`。

</details>

---

### 挑战题 (50%)

<details>
<summary><strong>练习 4：设计“人类确认 (Human-in-the-loop)”工具模式</strong></summary>

**题目**：你的 Agent 有一个 `delete_database` 的高危工具。
请设计一套“工具+流程”的组合，使得模型在调用该工具时，必须先获得用户的显式批准。要求模型能感知到“正在等待审批”和“审批通过/拒绝”的状态。

**提示**：

* 工具可以立即执行吗？
* 是否需要拆分成 `propose_deletion` 和 `execute_deletion`？或者利用回调？

**参考答案思路**：

* **方案**：工具本身并不删除数据，而是发起申请。
* **执行流**：
1. 模型调用 `request_delete_db(reason="...")`。
2. 工具返回：`"Request submitted. Confirmation ID: 888. Waiting for user approval..."`。
3. 系统此时在 UI 上弹出“允许删除吗？[Yes/No]”。
4. **中断**：Agent 暂停，等待。
5. 用户点击 Yes。系统构建一个特殊的 ToolOutput：`"User approved request 888."` 发回给模型。
6. 模型收到批准，再次调用 `perform_delete_db(confirmation_id=888)`（此工具会校验 ID 是否已授权）。



</details>

<details>
<summary><strong>练习 5：多模态截图工具的坐标系对齐</strong></summary>

**题目**：Agent 正在操作一个网页。它调用 `get_screenshot()` 拿到了一张 1920x1080 的图。模型觉得按钮在右上角，于是输出 `click(x=1800, y=100)`。
但是，由于 Image Token 压缩或 Resize，模型“看”到的图片实际上被缩放到了 512x512。
这会导致什么后果？如何从工程上解决这个问题？

**提示**：

* 坐标空间变换。
* Set-of-Mark (SoM) 提示法。

**参考答案思路**：

* **后果**：点击位置严重偏移，点到了错误的地方。
* **解决方案 1（归一化）**：要求模型总是输出 0.0-1.0 的相对坐标（如 `x=0.93, y=0.09`），执行层再根据真实分辨率（1920x1080）还原。
* **解决方案 2（视觉标记 SoM）**：不在像素级操作。先运行一个 OCR/检测模型，在按钮上打上标签（1, 2, 3...），把带标签的图给模型看。模型只需输出 `click(element_id=2)`。

</details>

<details>
<summary><strong>练习 6：工具调用的上下文污染与清洗</strong></summary>

**题目**：你有一个 `search_internal_docs` 工具，它会返回相关文档的全文。有时这会返回 50,000 tokens 的乱码或无关数据，直接把 Context Window 撑爆，导致模型忘记了 System Prompt 中指令（例如“不要撒谎”）。
请设计一个“中间件（Middleware）”策略来保护 Context。

**提示**：

* 谁来决定保留什么？是规则还是另一个模型？
* 压缩率 vs 信息丢失率。

**参考答案思路**：

* **策略**：在 Tool Output 进入 LLM 之前增加一个 **Sanitizer Layer**。
* **步骤**：
1. **长度检查**：如果 `len(output) > 2000 tokens`，触发压缩流程。
2. **格式清洗**：移除 HTML 标签、Base64 字符串、重复的换行。
3. **重排序/截断**：仅保留最相关的 Top-K 片段（如果工具本身支持检索打分）。
4. **摘要模型**：（可选）调用一个廉价模型（如 GPT-3.5）执行：“Extract key facts relevant to user's question: [User Query]”。
5. **注入提示**：在返回给主模型时加上标记：`[System Note: Output was summarized due to length.]`



</details>

---

## 3.8 本章小结

* **Schema 是契约**：写好 JSON Schema 是让 Agent 变聪明的最廉价方式。
* **闭环是核心**：Agent 不只是生成调用，它是“生成-执行-观察-修正”的无限循环。
* **并发是常态**：利用并行调用（Parallel Tool Use）来掩盖网络延迟，提升用户体验。
* **错误是朋友**：不要向模型隐藏工具的报错，合理的错误信息能引导模型自我纠正。
* **多模态需引用**：图片、音频、大文件在工具间传递时，传 ID 而不是传值。
