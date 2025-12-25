# 第 4 章 Agent Loop：规划-执行-反思的闭环

## 1. 开篇：构建智能体的“心跳”

在前几章中，我们赋予了系统“眼睛”（多模态理解）和“手”（工具调用）。然而，拥有一堆工具和强大的感知能力，并不等同于拥有一个能干活的智能体。

**Agent Loop（智能体循环）** 是系统的运行时（Runtime）核心，它像操作系统的调度器一样，协调着感知、记忆、推理和行动。它决定了智能体在接收到模糊指令后，是立即行动，还是先制定计划；是盲目重试，还是停下来反思错误。

如果说 LLM 是大脑（CPU），Prompt 是指令集，那么 Agent Loop 就是 **主线程（Main Thread）**。一个设计糟糕的 Loop 会导致智能体陷入死循环、烧穿预算或在任务完成前过早停止。

**本章学习目标：**

* **深入理解 O-T-A-R 循环**：Observe（观测）、Think（思考）、Act（行动）、Reflect（反思）的微观运作。
* **掌握三种主流架构**：ReAct（单流）、Plan-and-Execute（双流）与 Reflexion（自修正）的适用场景与优劣。
* **学会“刹车”**：设计鲁棒的终止条件（Termination Conditions）与预算控制（Budgeting）。
* **处理多模态特异性**：如何在 Loop 中处理视觉反馈的滞后与不确定性。

---

## 2. 核心论述

### 2.1 智能体控制循环的标准模型 (The O-T-A-R Cycle)

最基础的 Agent Loop 是一个状态机。在每一个时间步 ，智能体接收环境状态 ，生成思考  和 动作 ，并获得新的观测 。

我们将其扩展为 **O-T-A-R** 四阶段模型：

#### 1. Observe (全模态观测)

在传统文本 Agent 中，观测仅是 API 的 JSON 返回。但在多模态 Agent 中，**观测是异构的**：

* **文本流**：工具的标准输出（stdout）、错误日志（stderr）。
* **视觉流**：网页渲染后的截图（Screenshot）、PDF 的特定区域（Crop）、摄像头的实时帧。
* **状态流**：文件系统的变更、数据库的行更新。

> **Rule of Thumb**: 永远不要假设模型能“记住”上一轮的图片。在 Loop 的每一步，必须显式地管理多模态数据的**生命周期**（是重新传入图片，还是传入图片的 Embedding/Summary）。

#### 2. Think (推理与路由)

这是 LLM 发挥作用的阶段。它需要根据历史上下文（Memory）和当前观测，决定：

* **任务是否完成？**
* **是否需要调用工具？** 哪个工具？参数是什么？
* **是否遇到了阻碍？** 需要修改策略吗？

#### 3. Act (执行与副作用)

执行具体的工具代码。

* **关键点**：Act 阶段应当捕获所有常。工具的崩溃不应导致 Agent Loop 的崩溃，而应转化为一段“错误观测文本”返回给 Think 阶段。

#### 4. Reflect (反思与评价) *[可选但推荐]*

这是区分“脚本”与“智能体”的关键。在 Action 执行后，不立即进入下一次 Observe，而是增加一个“评价”步骤：

* *“这个截图里真的包含我要的数据吗？”*
* *“我连续翻了3页都没找到，是不是方向错了？”*

#### [图示 4.1] O-T-A-R 状态流转图

```text
       Start Task
           |
           v
+-----------------------------+
|  1. Observe (Senses)        |<-----------------------+
|  - Inputs: User Query,      |                        |
|    Tool Output, Screenshot  |                        |
+----------+------------------+                        |
           |                                           |
           v                                           |
+-----------------------------+                        |
|  2. Think (Reasoning)       |                        |
|  - "Based on the error..."  |                        |
|  - "I should try..."        |                        |
+----------+------------------+                        |
           |                                           |
           v                                           |
+-----------------------------+                  +-----+------+
|  3. Act (Execution)         | --(Success)-->   | 4. Reflect |
|  - Call Tool: browser.get() |                  | (Critique) |
|  - Call Tool: pdf.read()    | <--(Failure)--   +------------+
+-----------------------------+

```

---

### 2.2 架构范式演进

工程中主要存在三种 Loop 组织形式，复杂度由低到高：

#### A. ReAct (Reason + Act) —— 鲁棒的独行侠

**核心逻辑**：将“思考”与“行动”交织在一起。模型必须在输出 Tool Call 之前，先输出一段 `Thought: ...`。

* **优势**：极高的容错率。模型通过“自言自语”可以动态调整短期计划。
* **劣**：**上下文窗口杀手**。对于长链路任务（如写代码），中间的思考过程和工具返回会迅速占满 Token，导致模型遗忘最初的目标。
* **适用**：搜索问答、简单排查、单文档分析。

#### B. Plan-and-Execute (P&E) —— 谋定而后动

**核心逻辑**：将“规划者（Planner）”与“执行者（Executor/Solver）”解耦。

1. **Planner**：查看任务，生成一个包含  个步骤的清单（DAG 图或线性列表）。
2. **Executor**：在一个简单的 Loop 中逐个执行步骤。
3. **Replanner**：如果 Executor 报错，或者发现步骤不合理，唤醒 Planner 重新规划后续步骤。

* **优势**：Token 效率高（Executor 不需要看完整的规划历史，只需看当前步骤）。
* **劣势**：不够灵活。如果第 1 步的结果彻底改变了第 2 步的前提，P&E 架构往往反应迟钝。
* **适用**：软件开发、长篇报告生成、复杂流程自动化。

#### C. Reflexion / Self-Correction —— 具有自我修复能力

**核心逻辑**：在 Loop 中引入一个**显式的“评价器（Evaluator）”**。

* 当工具报错或结果为空时，不直接扔给下一轮 Think。
* **Self-Correction**：先生成一段“为什么失败”的分析，将其作为由内向外的“伪观测”插入 Context。
* **示例**：
* *Action*: 点击按钮 `<btn id="submit">`
* *Observation*: `Element not interactable`
* *Reflexion*: “看来有弹窗遮挡了按钮，我应该先检查是否有弹窗需要关闭。”
* *New Action*: 关闭弹窗 -> 点击按钮。



---

### 2.3 任务分解与预算管理 (Decomposition & Budgeting)

#### 任务分解的艺术

对于“分析 2023 年 Q4 财报”这种模糊指令，Agent 必须具备**递归分解**的能力。

* **基于数据的分解**：将 100 页 PDF 拆分为 10 个 10 页的 Chunk 处理。
* **基于步骤的分解**：下载 -> OCR -> 提取表格 -> 计算 -> 撰写。

> **Rule of Thumb**: 给 Loop 设置一个 **"Depth Limit" (递归深度限制)**。如果子任务还需要分子任务，且深度超过 3 层，通常意味着 Prompt 定义不清或任务过于发散。

#### 预算感知 (Budget-aware Execution)

Agent 是昂贵的。Loop 必须是“资源敏感”的。
你需要维护以下计数器：

1. **Step Count**：当前是第几轮循环？（防止死循环）
2. **Token Cost**：当前会话花了多少钱？
3. **Stall Count**：连续多少次没有产出有效信息？（防止原地打转）

**当预算耗尽时（Out of Budget）：**

* **错误做法**：抛出异常 Crash。
* **正确做法**：触发 **"Wrap-up"（收尾）模式**。强制 Agent 基于手头已有的（哪怕是不完整的）信息生成一个最终回复，并附带“置信度声明”和“未完成项清单”。

---

### 2.4 结构化输出与状态机 (Structured Output)

不要让 Agent 在 Loop 结束时只吐出一堆自然语言。为了集成到下游系统，Loop 的最终产出应该是结构化的。

**推荐的最状态 Payload：**

```json
{
  "status": "success", // or "partial_success", "failed"
  "summary": "已完成财报分析，发现营收增长 20%。",
  "artifacts": [ // 产生的文件或链接
    {"type": "chart", "path": "/tmp/revenue.png"},
    {"type": "pdf", "path": "/output/report.pdf"}
  ],
  "reasoning_trace": "...", // 决策链摘要
  "needs_human_help": false // 是否需要人工介入
}

```

---

## 3. 本章小结

1. **Loop 是操作系统**：它不仅运行模型，还负责资源调度（Token/Money）和异常处理。
2. **ReAct 是默认选择，P&E 是进阶选择**：简单任务用 ReAct，复杂长流程任务用 Plan-and-Execute。
3. **Reflexion 提升上限**：通过显式的反思步骤，可以将 Agent 的成功率显著提升，特别是面对不稳定的多模态工具（如 OCR 识别错误）时。
4. **死循环是最大敌人**：必须在工程层面实现“僵局检测”和“预算强制中断”。

---

## 4. 练习题

### 基础题 (熟悉材料)

**Q1: 为什么在 Agent Loop 中直接将所有的 Tool Output 拼接到 Context 里是危险的？请列举两个负面后果。**

<details>
<summary>**提示与答案**</summary>

* **Hint**: 考虑上下文窗口大小和模型的注意力机制。
* **Answer**:
1. **Token 溢出与成本激增**：如果工具返回了整个 HTML 源码或几千行的日志，会迅速消耗 Token 预算。
2. **注意力稀释（Lost in the Middle）**：过长的无关技术细节会干扰 LLM，使其忘记最初的用户指令或之前的关键逻辑。


* *解决办法*：在存入 Memory 前进行截断（Truncation）或摘要（Summarization）。



</details>

**Q2: 解释 ReAct 模式中 "Thought" 的作用。如果去掉 "Thought" 直接输出 "Action"，会有什么影响？**

<details>
<summary>**提示与答案**</summary>

* **Hint**: 类似于人类在做复杂数学题时打草稿。
* **Answer**:
* "Thought" 提供了一个**思维链（Chain of Thought）缓冲区**，允许模型在决定行动前行推理、规划和纠错。
* 如果去掉，模型变成了直觉反应式（Reactive），处理复杂逻辑任务（如多步推理）的能力会显著下降，更容易产生幻觉或错误的工具调用。



</details>

**Q3: 在 Agent Loop 中，什么是 "Stall Detection"（停滞检测）？**

<details>
<summary>**提示与答案**</summary>

* **Hint**: 如果 Agent 连续 5 次都在翻同一页书...
* **Answer**:
* 这是一种工程保护机制。用于检测 Agent 是否陷入了无效循环。
* 例如：连续 N 次调用工具的参数完全相同，或者连续 N 次 Observation 的内容极其相似。
* 触发后通常会强制终止 Loop 或强制 Agent 更换策略。



</details>

---

### 挑战题 (开放性思考)

**Q4 (场景设计): 设计一个针对“多模态网页浏览 Agent”的 Reflexion 策略。**
*场景：Agent 试图点击网页上的“下载”按钮，但每次点击后，页面都没有发生变化（没有下载弹窗，URL 也没变）。*
*要求请写出 Agent 在 Reflect 阶段应该进行的内心独白（Inner Monologue）和下一步调整策略。*

<details>
<summary>**提示与答案**</summary>

* **Hint**: 视觉上按钮可能被遮挡，或者 DOM 结构变了，或者需要先登录。
* **Answer**:
* **Reflection (Inner Monologue)**: "我执行了 Click 动作，但 Observation 显示页面截图和 URL 均未变化。这说明点击失败了。可能有以下原因：1. 按钮被广告遮挡；2. 这是一个伪元素，需要通过 JavaScript 触发；3. 页面还在加载中。"
* **Next Strategy**: "我不应该重复点击。我将尝试：1. 调用 `Scroll` 命令确保元素在视口中心；2. 检查页面是否存在模态弹窗（Modal）遮挡；3. 尝试使用键盘事件（Enter 键）代替鼠标点击。"



</details>

**Q5 (架构权衡): 假设你要构建一个“长篇小说写作 Agent”，目标是写一本 10 万字的小说。**

* (A) 使用单体 ReAct Loop。
* (B) 使用 Plan-and-Execute Loop。
* 请选最合适的架构，并说明为什么另一种架构会失败。

<details>
<summary>**提示与答案**</summary>

* **Hint**: 上下文长度和全局一致性是关键。
* **Answer**:
* **选择 (B) Plan-and-Execute**（或者更高级的分层架构）。
* **理由**：
* 小说需要全局一致的大纲（人物、情节走向）。
* **ReAct 失败原因**：在写到第 5 章时，单体 Loop 的 Context 早已溢出，Agent 会忘记第 1 章埋下的伏笔或人物性格，导致前后矛盾。
* **P&E 优势**：Planner 维护大纲（全局记忆），Executor 每次只负责写一个章节（局部记忆），写完后将摘要回传给 Planner。





</details>

**Q6 (多模态特有): "幻觉循环" (Hallucination Loop)。**
*场景：Agent 看一张图表，OCR 识别错误把 "100" 认成了 "1000"。Agent 基于 "1000" 进行推理，觉得数据异常，于是再次查询，但 OCR 依然返回 "1000"。Agent 陷入自我怀疑和反复查询的死循环。*
*问题：如何在 Loop 层面解决这种由感知错误引起的逻辑死锁？*

<details>
<summary>**提示与答案**</summary>

* **Hint**: 引入外部校验或多路投票。
* **Answer**:
1. **多视角/多工具投票**：在 Loop 中引入规则——如果对同一个数据的置信度存疑，强制调用另一个异构工具（例如：不要只用 OCR，尝试用 VLM 直接看图；或者尝试获取网页底层的 Table 数据）。
2. **人类介入（Human-in-the-loop）**：当 Agent 发现数据违反常识（Outlier Detection）且反复验证无果时，触发 `ask_human` 工具：“我看这个数是 1000，这很不合理，请帮我确认图片。”
3. **降级策略**：在 Reflection 中承认无法看清，并在最终报告中标记该数据点为“高风险/需人工核实”，跳过该步骤继续执行。



</details>

---

## 5. 常见陷阱与错误 (Gotchas)

### 5.1 解析失败导致的 "Retry Storm"

* **现象**：LLM 输出了一段包含 JSON 的文本，但 JSON 格式由微小的法错误（如缺个逗号），导致代码层解析失败。系统提示 LLM 重试，LLM 再次输出同样的错误 JSON。
* **调试**：
* **Rule of Thumb**: 使用 **Robust Parser**（如 JSON 修复库）而不是严格的 `json.loads`。
* 在 System Prompt 中提供具体的 One-shot 示例。
* 如果在 Loop 中连续 2 次解析失败，切换到简易模式或直接报错，不要无限重试。



### 5.2 "I have done it" 幻觉

* **现象**：用户让 Agent "给客户发邮件"。Agent 在 Thought 中写道 "Calling email tool..."，但实际上它并没有输出真正的 Tool Call 格式，或者工具调用失败了，但 Agent 在下一轮直接说 "邮件已发送"。
* **原因**：模型混淆了“计划做”和“已经做完”。
* **调试**：
* 强制要求 Observation 必须来自系统注入，严禁 LLM 自己生成 Observation。
* 在 Prompt 中强调：*“你只有在收到 `Tool Output: Success` 的系统消息后，才能确认动作已完成。”*



### 5.3 忽略多模态信息的“盲视”

* **现象**：Agent 拿着一张全是字的截图，却说“我没看到任何文字”。
* **原因**：图片分辨率被压缩太厉害，或者 VLM 的视觉编码器对密集文本支持不好。
* **调试**：
* **Always check resolution**：在 Loop 开始前检查图片尺寸，过小则拒绝或 Upscale。
* **混合感知**：对于文档类任务，不要只依赖 VLM（视觉），必须配合 OCR（文字提取）作为辅助输入（Visual Grounding）。



### 5.4 僵尸进程

* **现象**：Agent 在等待一个超时的网络请求，整个 Loop 卡住，用户以为死机了。
* **调试**：
* 给所有的 Tool Call 设置严格的 **Timeout**（如 30秒）。
* 实现 **Stream Thinking**：即使工具在跑，Agent 也可以向用户流式输出“正在读取网页，请稍候...”，保持交互活性。
