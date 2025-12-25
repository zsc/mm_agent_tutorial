# 第 7 章 OpenAI Harmony 格式与多模态消息协议

> **开篇段落**
> 在单体 LLM 时代，Prompt 是艺术；在多模态 Agent 系统时代，协议（Protocol）是法律。
> 随着模型从单一的文本输入进化为视觉、音频、工具调用混合的复杂系统，仅仅通过字符串拼接来管理上下文已经难以为继。**OpenAI Harmony 格式**（即 ChatML 及其衍生标准）已成为业界事实上的标准接口。它不仅仅是 API 的参数规范，更是智能体**思维流（Stream of Thought）**、**行动链（Action Chain）**和**感知数据（Perception Data）**的标准化容器。
> 本章将深入解剖这一协议的每一个字节。我们将超越基础的 JSON 结构，探讨如何优雅地封装多模态切片、如何通过 ID 严格锚定工具执行流、以及如何在这一协议之上构建支持多智能体协作（Multi-Agent）的高级通信标准。掌握这一章，你就掌握了智能体系统各组件间对话的“通用语”。

---

## 7.1 为什么需要统一协议：从“文本”到“结构化流”

在构建复杂的 Agent 系统时，我们面临着“格式的巴别塔”：

1. **多模态异构性**：如何在一个数组中同时表达“用户的文字”、“PDF 的第 3 页截图”和“上一轮工具返回的 JSON 报错”？
2. **执行的可追溯性**：当智能体并行调用了 3 个工具，如何知道哪个结果对应哪个请求？
3. **微调与评估**：如果你的 Trace 日志没有标准格式，未来就无法利用这些数据进行 SFT（监督微调）或 DPO（偏好优化）。

Harmony 格式的核心哲学是：**一切皆消息（Message），消息皆有角色（Role），内容皆为数组（Array）。**

---

## 7.2 消息解剖学：角色与多模态图文流

一个标准的 Context 是一个有序的 `List[Message]`。

### 7.2.1 角色定义（The Standard Roles）

| 角色 (Role) | 核心职责 | 工程隐喻 |
| --- | --- | --- |
| **System** | 世界设定、规则边界、输出格式约定。 | `Root User / Kernel` |
| **User** | 任务发起者、外部环境信号、多模态输入源。 | `Stdin / Event Trigger` |
| **Assistant** | 智能体的大脑。输出思考（Text）或行动请求（Tool Calls）。 | `Process / Logic` |
| **Tool** | 环境反馈。只负责执行指令并返回客观结果（或报错）。 | `Stdout / System Call Return` |

### 7.2.2 内容块（Content Parts）：多模态的原子单位

为了处理图文混排，`content` 字段不再是字符串，而是对象数组。

```text
[ASCII: 多模态消息的内存布局]

Message (Role: User)
+------------------------------------------------------------+
| content: [                                                 |
|   { type: "text", text: "这是现场拍摄的仪表盘照片：" },    |
|   {                                                        |
|     type: "image_url",                                     |
|     image_url: {                                           |
|       url: "s3://bucket/img_001.jpg",                      |
|       detail: "high"  <-- 触发切片机制                     |
|     }                                                      |
|   },                                                       |
|   { type: "text", text: "请读数并判断是否异常。" }         |
| ]                                                          |
+------------------------------------------------------------+

```

### 7.2.3 图像处理的 Rule of Thumb

* **Detail Parameter**: 务必显式指定 `detail`。
* `low`: 模型仅看到缩略图（512x512），消耗 tokens 少（约 85 tokens），适合概览。
* `high`: 模型会将图片切分为多个 512x512 的 tiles 加上一张缩略图。适合 OCR、图表分析。


* **Base64 vs URL**:
* **开发/测试**：使用 Base64，方便快捷，无鉴权烦恼。
* **生产环境**：使用 URL（通常是预签名 URL）。Base64 会极大地增加网络 Payload 大小，增加延迟和序列化开销。



---

## 7.3 Tool Call：严格的闭环协议

这是 Agent 区别于 Chatbot 的分水岭。协议要求**请求与响应必须通过 ID 严格配对**。

### 7.3.1 完整的工具调用生命周期

一个交互回合（Turn）通常包含 4 个步骤：

```text
[ASCII: 工具调用时序图]

1. USER
   Content: "查一下北京天气"
       |
       v
2. ASSISTANT (生成)
   Content: null (或 "好的，我来查询...")
   Tool_Calls: [
     { id: "call_abc123", type: "function", function: {name: "get_weather", arguments: "{'city':'bj'}"} }
   ]
       |
       v (系统拦截，执行函数，获得结果 "22C")
       |
3. TOOL (提交结果)
   Role: "tool"
   Tool_Call_Id: "call_abc123"  <-- 必须匹配!
   Content: "Temperature: 22C, Sunny"
       |
       v
4. ASSISTANT (最终响应)
   Content: "北京今天晴天，气温 22 度。"

```

### 7.3.2 并行调用（Parallel Tool Calls）与错误处理

现在的模型（如 GPT-4o）倾向于一次性发出多个工具调用。

* **场景**：用户问“对比 Amazon 和 Google 的股价”。
* **Assistant 输出**：`Tool_Calls: [ {id:1, name: get_stock, args: AMZN}, {id:2, name: get_stock, args: GOOG} ]`
* **规则**：
1. **全量返回**：下一轮必须包含 2 条 Tool 消息。
2. **ID 对应**：顺序不重要，但 `tool_call_id` 必须存在。
3. **部分失败**：如果 Google 接口超时，**不要**抛出异常中断 Agent。必须构建一条 `tool` 消息，内容为 `Error: Timeout fetching data for GOOG`。让模型自己决定是重试还是报告部分结果。



---

## 7.4 Harmony 与多智能体（Multi-Agent）协作

当系统中有多个 Agent（如“研究员”和“写作者”）时，我们需要扩展协议来标识身份。

### 7.4.1 `name` 字段的战略用法

标准协议支持可选的 `name` 字段（正则约束 `^[a-zA-Z0-9_-]+$`）。

* **区分不同专家**：
```json
{"role": "assistant", "name": "Reviewer_Agent", "content": "这段代码有安全漏洞..."}
{"role": "assistant", "name": "Coder_Agent", "content": "收到，我正在修复..."}

```


*注意：并非所有模型都完美遵循 `name` 字段的注意力机制，有时需要在 content 中再次强调身份。*

### 7.4.2 Handoff Packet（移交数据包）

不要依赖自然语言说“现在转交给写作者”。应定义一个**隐式工具**来完成移交。

* **Handoff Tool Schema**:
```json
{
  "name": "transfer_to_writer",
  "description": "当研究资料收集完毕后，将任务移交给写作 Agent。",
  "parameters": {
    "type": "object",
    "properties": {
      "research_summary": {"type": "string"},
      "source_file_ids": {"type": "array", "items": {"type": "string"}},
      "instruction_for_writer": {"type": "string"}
    },
    "required": ["research_summary"]
  }
}

```



---

## 7.5 常见陷阱与错误 (Gotchas)

### 🔴 1. 幽灵 ID (The Phantom ID)

* **错误**：开发者在重试或手动构建历史时，随手编造了一个 `tool_call_id`，或者在 Tool 消息中丢失了 ID。
* **后果**：模型抛出 `BadRequest` 或 `Unmatched Tool Call` 错误。现代 API 对此校验极其严格。
* **对策**：将 Tool Call 对象视为不可变的（Immutable）。一旦生成，必须原样保存并在 Tool 消息中引用。

### 🔴 2. Base64 爆炸 (Payload Explosion)

* **错误**：在长对话中，保留了所有历史图片的 Base64 编码。
* **后果**：Context 迅速爆满，网络传输超时，API 费用激增。
* **对策**：在内存中维护对话时，如果图片超过 N 轮（如 5 轮）未被引用，应将其替换为 `(Image placeholder: omitted to save context)`，或者仅保留文本摘要。

### 🔴 3. System Prompt 的“越权”

* **错误**：将动态的业务数据（如搜索结果）放入 System Prompt。
* **后果**：随着上下文变长，System Prompt 的权重在某些模型中会衰减。且 System Prompt 通常用于定义行为，而非输入数据。
* **对策**：System 只放“人设”和“规则”。业务数据全部通过 User 消息或 Tool Output 注入。

### 🔴 4. XML 标签与 JSON 的混战

* **错误**：提示词要求模型“先思考（用XML），再行动（用JSON）”，导致模型输出 `<think>...</think> {tool_call...}`。
* **后果**：标准 API 解析器无法识别混合在文本中的 JSON，导致工具调用失败。
* **对策**：
1. 强制模型将思考过程也作为参数的一部分（如 `explanation` 字段）。
2. 或者在接收端编写鲁棒的解析层，剥离 XML 后再手动构造 Tool Call 对象（适用于非 Function Calling 模型）。



---

## 7.6 本章小结

1. **协议即法律**：Harmony 格式（User/Assistant/Tool/System）是构建稳定 Agent 的基石，任何对格式的随意修改都会导致不可预测的“智商下降”。
2. **多模态是对象流**：停止将图片视为附件，它们是与文本平级的 `content_part`。
3. **闭环至上**：工具调用必须有始有终（Call -> Result），ID 是贯穿始终的唯一线索。
4. **显式协作**：在多智能体场景下，利用 `name` 字段和显式的 `handoff` 工具来管理控制流，优于隐式的自然语言指令。

---

## 7.7 练习题

### 基础题（熟悉 Schema）

<details>
<summary><strong>练习 1：构建图文混合消息 Payload</strong>（点击展开）</summary>

**题目**：
你需要向模型发送一条 User 消息，内容包含：

1. 文本：“请分析这张发票的抬头和金额。”
2. 一张 URL 图片（`https://example.com/invoice.jpg`），要求高精度模式。
3. 一张本地 Base64 图片（缩略图，假设数据为 `...`），要求自动精度模式。

请写出该消息的 JSON 结构。

**提示**：注意 `content` 数组的顺序和每个对象的 `type`。

**参考答案**：

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "请分析这张发票的抬头和金额。"
    },
    {
      "type": "image_url",
      "image_url": {
        "url": "https://example.com/invoice.jpg",
        "detail": "high"
      }
    },
    {
      "type": "image_url",
      "image_url": {
        "url": "data:image/jpeg;base64,...",
        "detail": "auto"
      }
    }
  ]
}

```

</details>

<details>
<summary><strong>练习 2：多智能体对话历史构建</strong>（点击展开）</summary>

**题目**：
你需要模拟一个对话历史，其中：

1. System 设定：“你是一个圆桌会议记录员。”
2. User (主持人)：“开始讨论。”
3. Assistant (专家 A，名字叫 "Physics_Expert")：“主要是引力问题。”
4. Assistant (专家 B，名字叫 "Math_Expert")：“我同意，但公式需要修正。”

请写出第 3、4 条消息的 JSON 结构。

**提示**：使用 `name` 字段。

**参考答案**：

```json
[
  {
    "role": "assistant",
    "name": "Physics_Expert",
    "content": "主要是引力问题。"
  },
  {
    "role": "assistant",
    "name": "Math_Expert",
    "content": "我同意，但公式需要修正。"
  }
]

```

</details>

<details>
<summary><strong>练习 3：工具调用的配对逻辑</strong>（点击展开）</summary>

**题目**：
模型发出了一个工具调用 `id: "call_999"`, 函数名 `search_google`.
随后，程序抓取到了结果 "No results found".
请写出反馈给模型的下一条消息。

**参考答案**：

```json
{
  "role": "tool",
  "tool_call_id": "call_999",
  "name": "search_google", 
  "content": "No results found"
}

```

*注意：`name` 字段在某些 API 实现中是可选的，但 `tool_call_id` 是必须的。*

</details>

### 挑战题（工程实战与边缘情况）

<details>
<summary><strong>练习 4：并行工具调用的异常修复</strong>（点击展开）</summary>

**题目**：
模型一次性输出了 3 个工具调用：

1. `read_file("A.txt")` (ID: 101)
2. `read_file("B.txt")` (ID: 102)
3. `read_file("C.txt")` (ID: 103)

在后端执行时，文件 A 和 C 读取成功，但文件 B 不存在（抛出 FileNotFoundError）。
如果你的代码直接抛出异常并停止处理，Agent 就会崩溃。
请设计一个正确的 `List[Message]` 返回给模型，使对话能继续。

**提示**：你必须返回 3 条 Tool 消息，不能少。

**参考答案**：

```json
[
  { "role": "tool", "tool_call_id": "101", "content": "Content of A..." },
  { "role": "tool", "tool_call_id": "102", "content": "Error: FileNotFoundError: 'B.txt' does not exist." },
  { "role": "tool", "tool_call_id": "103", "content": "Content of C..." }
]

```

**关键点**：将系统异常转化为协议内的 Error Message。模型看到这个错误后，可能会在下一轮尝试 `list_dir` 或者询问用户。

</details>

<details>
<summary><strong>练习 5：长文档的 Token 预算管理</strong>（点击展开）</summary>

**题目**：
你的 Agent 正在阅读一本 100 页的 PDF。用户问了关于第 50 页的一个图表的问题。
目前的 Context 策略是：将每一页都转为图片传入。
但这会迅速耗尽 Token。请设计一个基于 Harmony 协议的“滑动窗口”或“引用”策略。

**提示**：不要真的把 100 张图放进 `content`。使用 Tool 来按需加载。

**参考答案**：
**策略**：

1. System Prompt 中告知模型：“你可以通过 `view_page(page_number)` 工具查看文档的特定页面。”
2. User 传入 PDF 的文本摘要或目录结构。
3. 模型决定调用 `view_page(50)`。
4. Tool 返回第 50 页的图片数据（高精度）。
5. **核心技巧**：在第 3 轮对话后，如果模型不再关注第 50 页，应在历史记录中将该 Tool Result 的图片数据替换为 `(Image data cleared to save context)`，仅保留纯文本描述。

</details>

<details>
<summary><strong>练习 6：设计通用的 Handoff 协议</strong>（点击展开）</summary>

**题目**：
你需要设计一个通用的 `handoff` 工具 Schema，用于在三个不同能力的 Agent（检索、代码、写作）之间流转任务。
要求该工具能传递：

1. 目标 Agent 的名称。
2. 任务的上下文（Payload）。
3. 已有的“资产”（如已下载的文件路径列表）。

**参考答案**：

```json
{
  "name": "handoff_task",
  "description": "Transfer the current task to a specialized agent.",
  "parameters": {
    "type": "object",
    "properties": {
      "target_agent": {
        "type": "string",
        "enum": ["search_agent", "coding_agent", "writer_agent"]
      },
      "handoff_context": {
        "type": "string",
        "description": "Summary of what has been done and what needs to be done next."
      },
      "assets": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "asset_type": {"type": "string", "enum": ["file_path", "memory_key"]},
            "value": {"type": "string"}
          }
        },
        "description": "List of file paths or memory keys to pass along."
      }
    },
    "required": ["target_agent", "handoff_context"]
  }
}

```

</details>
