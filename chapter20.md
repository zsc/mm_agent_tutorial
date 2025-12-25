# 附录 A：Harmony 格式与多模态消息协议标准 (Appendix A)

## 1. 协议设计哲学与适用范围

本附录定义的 JSON 结构被称为 **"Harmony Schema"**，它是一种基于 OpenAI ChatCompletions API 的超集，旨在解决标准 API 在处理**复杂多模态、多智能体协作、以及长链路状态管理**时的不足。

**设计原则 (Rule of Thumb):**

1. **单一事实来源 (Single Source of Truth)**：所有的感知（图片/音频）、推理（文本）、行动（工具调用）都必须序列化为 Message List。
2. **原子化 (Atomicity)**：多模态内容（Content Parts）必须独立的、可寻址的对象，而不是单纯的拼接字符串。
3. **无状态性 (Statelessness)**：Handoff Packet（移交包）必须包含接手任务所需的一切信息，接收方无需查询上一轮 Agent 的原始日志。

---

## 2. A.1 通用消息体结构 (The Universal Message Object)

这是系统中流转的最小数据单元。

### A.1.1 基础骨架

```json
{
  "id": "msg_123456789",          // 消息唯一 ID（用于引用和回滚）
  "role": "user | assistant | system | tool",
  "name": "ResearchAgent_01",     // [可选] 发言者名称，多 Agent 场景必填
  "created_at": 1715000000,       // Unix 时间戳
  "content": [ ... ],             // 见下文 Content Parts
  "metadata": { ... }             // 见 A.6 元数据部分
}

```

### A.1.2 Content Parts：多模态载荷详解

`content` 字段不再是字符串，而是一个对象数组。

#### (1) 文本与推理链 (Text & Thought)

```json
{
  "type": "text",
  "text": "根据用户上传的图片，我测到了明显的裂痕。",
  "index": 0
}

```

#### (2) 图像 (Image) - 支持 URL 与 Base64

> **注意**：对于生产环境，强烈建议使用 URL（配合短期签名的 S3 链接），Base64 仅用于调试或极低延迟的本地交互。

```json
{
  "type": "image_url",
  "image_url": {
    "url": "https://s3.amazonaws.com/bucket/invoice_scan_01.jpg",
    "detail": "high", // 选项: low (85 tokens), high (按 tiles 计算), auto
    "rotation": 0     // [扩展] 预处理旋转角度，防止模型看歪
  }
}

```

#### (3) 视频片段 (Video Segment) - 抽帧序列表示

视频通常不直接发给模型，而是转为一组带时间戳的图片帧。

```json
{
  "type": "image_url",
  "image_url": { "url": "..." },
  "video_metadata": {           // [扩展字段]
    "video_id": "vid_abc",
    "timestamp_sec": 12.5,      // 该帧在视频中的秒数
    "frame_index": 300
  }
}

```

#### (4) 音频输入 (Input Audio)

```json
{
  "type": "input_audio",
  "input_audio": {
    "data": "base64_string...", // PCM 原始数据或压缩格式
    "format": "wav",
    "sample_rate": 24000
  }
}

```

---

## 3. A.2 工具调用协议 (Tool Invocation Protocol)

这是 Agent 与外部世界交互的**物理层**。

### A.2.1 模型发起的调用 (Assistant Message)

```json
{
  "role": "assistant",
  "content": null, // 或者是 "我将为您查询天气..."
  "tool_calls": [
    {
      "id": "call_890xyz",
      "type": "function",
      "function": {
        "name": "search_knowledge_base",
        "arguments": "{\"keywords\": [\"Transformer architecture\", \"Attention mechanism\"], \"year_filter\": 2023}"
      }
    }
  ]
}

```

### A.2.2 工具执行结果 (Tool Message)

**关键规则**：

* **1对1映射**：`tool_calls` 数组中有几个元素，后续就必须紧跟几个 `role: tool` 的消息。
* **错误处理**：如果工具报错，**必须**返回 JSON 格式的错误信息，而不是抛出异常，以便模型决定是否重试。

#### 成功响应：

```json
{
  "role": "tool",
  "tool_call_id": "call_890xyz",
  "name": "search_knowledge_base",
  "content": "{\"status\": \"success\", \"results\": [{\"title\": \"Attention is All You Need\", \"snippet\": \"...\"}]}"
}

```

#### 失败响应（让模型自我修复）：

```json
{
  "role": "tool",
  "tool_call_id": "call_890xyz",
  "content": "{\"status\": \"error\", \"error_type\": \"InvalidDateRange\", \"message\": \"The year 2023 is in the future relative to the database snapshot (2022). Please try an earlier year.\"}"
}

```

### A.2.3 多模态工具返回 (Advanced)

有些工具（如浏览器截图、matplotlib 绘图）返回的是图片。

```json
{
  "role": "tool",
  "tool_call_id": "call_render_chart",
  "content": [
    { "type": "text", "text": "{\"status\": \"rendered_successfully\"}" },
    { "type": "image_url", "image_url": { "url": "https://.../chart_temp.png" } }
  ]
}

```

---

## 4. A.3 任务移交与状态包 (Handoff Packet Schema)

当流程从 "Planner Agent" 转移到 "Coder Agent" 时，使用此 Schema。

```json
{
  "protocol": "harmony_handoff_v1",
  "meta": {
    "source_agent": "planner_v2",
    "target_agent": "coder_v1",
    "timestamp": "2024-03-20T10:00:00Z"
  },
  "context": {
    // 1. 任务目标（不可变）
    "primary_objective": "为用户构建一个 Python 贪吃蛇游戏",
    
    // 2. 当前进度
    "status": "in_progress",
    "completed_steps": ["需求分析", "技术选型(Pygame)"],
    "next_immediate_step": "编写核心游戏循环代码",
    
    // 3. 关键约束 (Memory)
    "constraints": [
      "必须有详细注释",
      "背景颜色设为黑色"
    ],
    
    // 4. 共享资产 (Artifacts) - 避免重复上传/下载
    "artifacts": {
      "design_doc": "s3://.../spec.md",
      "assets_folder": "./assets/"
    }
  },
  // 5. [可选] 压缩后的对话历史摘要
  "history_summary": "用户最初要求3D游戏，经协商确认为2D版本..." 
}

```

---

## 5. A.4 引用与证据锚定 (Citation & Grounding Schema)

为实现“可信回答”，每条生成内容都应关联证据。这通常存储在消息的 `metadata` 或专门的 `context_block` 中。

### A.4.1 证据对象定义

```json
{
  "citation_map": {
    "cit_01": {
      "source_type": "pdf_document",
      "source_id": "doc_fin_report_2023",
      "source_name": "2023_Financial_Report.pdf",
      "location": {
        "page_number": 42,
        "bbox": [100, 200, 500, 600], // [x1, y1, x2, y2] 用于前端在 PDF 上画框
        "text_snippet": "Revenue increased by 20% YoY." // 用于校验
      },
      "score": 0.89 // 检索相关性分数
    },
    "cit_02": {
      "source_type": "web_search",
      "url": "https://wikipedia.org/...",
      "access_date": "2024-03-20"
    }
  }
}

```

### A.4.2 文本中的锚点 (In-text Anchors)

模型输出的文本应包含如下标记：

> "2023年的营收增长强劲 **<cite id="cit_01"/>**，这主要是由于云服务市场的扩张 **<cite id="cit_02"/>**。"

---

## 6. A.6 元数据与可观性字段 (Metadata & Observability)

为了生产环境的调试（Trace）和计费，每条消息应携带以下隐形信息。

```json
"metadata": {
  // 成本追踪
  "token_usage": {
    "prompt_tokens": 500,
    "completion_tokens": 150,
    "total_cost_usd": 0.002
  },
  
  // 延迟追踪
  "latency_ms": {
    "ttft": 400,        // Time to First Token
    "total_time": 1200
  },
  
  // 模型指纹
  "model_version": "gpt-4-vision-preview-2024-04-09",
  "system_fingerprint": "fp_12345",
  
  // 业务上下文
  "session_id": "sess_user_999",
  "trace_id": "trace_uuid_v4", // 关联到 Datadog/LangSmith
  
  // 安全标记
  "safety_check": {
    "flagged": false,
    "categories": []
  }
}

```

---

## 7. 本章小结

* **结构化至上**：永远不要依赖正则表达式去解析 Agent 的输出，必须强制要求 JSON 输出或 Tool Call。
* **ID 是生命线**：在多轮对话和工具调用中，`tool_call_id` 是连接 "思考" 与 "结果" 的唯一纽带，丢失 ID 会导致上下文错乱。
* **Handoff 显式化**：在多 Agent 切换时，必须显式传递 `constraints` 和 `artifacts`，否则接手的 Agent 会像失忆一样重复询问用户。

---

## 8. 练习题

### 练习 1：构建一个“视频异常检测”的消息载荷

**场景**：安防摄像头每秒截取一帧，Agent 需要分析第 10 秒到第 15 秒的画面，判断是否有人员入侵。
**要求**：使用 `content parts` 构造 User Message，包含 5 张关键帧，并附带时间戳元数据。

<details>
<summary>点击查看参考答案</summary>

```json
{
  "role": "user",
  "content": [
    {
      "type": "text",
      "text": "请分析以下监控画面序列（10s-15s），判断是否存在未授权人员进入禁区。"
    },
    // 第 10 秒帧
    {
      "type": "image_url",
      "image_url": { "url": "https://.../frame_10s.jpg", "detail": "low" },
      "video_metadata": { "timestamp_sec": 10.0 } // 自定义字段，需模型或中间件支持解析
    },
    // ... 间帧省略 ...
    // 第 15 秒帧
    {
      "type": "image_url",
      "image_url": { "url": "https://.../frame_15s.jpg", "detail": "low" },
      "video_metadata": { "timestamp_sec": 15.0 }
    }
  ]
}

```

*提示：对于安防场景，通常设置 detail: "low" 以降低延迟，除非需要识别人脸细节。*

</details>

### 练习 2：设计一个“需要用户确认”的工具中断协议

**场景**：Agent 想要执行 `delete_database`，这是一个高危操作。设计一个 Tool Call 响应，既不执行删除，又引导模型去询问用户。

<details>
<summary>点击查看参考答案</summary>

**思路**：工具层拦截该请求，返回一个特殊的 Error 或 Info，告诉模型“需要确认”。

```json
{
  "role": "tool",
  "tool_call_id": "call_delete_db_123",
  "content": "{\"status\": \"halted\", \"reason\": \"human_confirmation_required\", \"message\": \"This is a high-risk operation. You MUST ask the user for explicit confirmation (type 'YES') before proceeding. Do not execute this tool again until confirmation is received.\"}"
}

```

*Agent 收到此消息后，会输出文本向用户提问，从而实现 Human-in-the-loop。*

</details>

---

## 9. 常见陷阱 (Gotchas)

1. **JSON 嵌套地狱**：
* *错误*：在 `tool_call` 的 `arguments` 里直接放 JSON 对象。
* *正确*：`arguments` **必须是 String**。即 `JSON.stringify(args_object)`。这是最常见的新手错误，导致 API 400 报错。


2. **图片过期**：
* 如果是使用临时签名的 URL（如 AWS S3 Presigned URL），通常有效期只有 15-60 分钟。如果用户第二天回来翻看历史记录，Agent 可能会因为链接失效而报错。
* *对策*：在应用层（Application Layer）做一层代理，或者在存入 Long-term Memory 时将图片转存到持久化存储并更新 URL。


3. **Token 溢出与截断**：
* History List 不限长，但 Context Window 有限。
* *对策*：不要盲目 `.append()`。必须实现一个 `trim_messages(history, max_tokens)` 函数。对于多模态消息，要特别注意一张 `detail: high` 的图片可能占用 1000+ tokens，比几百行文字还多。优先丢弃旧的图片，保留旧的文本摘要。


4. **Audio/Video 混淆**：
* 目前的模型通常不支持直接上传 `.mp4` 文件。你必须在客户端或服务端通过 `ffmpeg` 抽帧（Frames Extraction）并提取音轨（Audio Extraction），分别作为 Image List 和 Audio Input 传入。不要试图把二进制流直接塞进 Text 字段。
