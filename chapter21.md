# 附录 B：Tool Schema Cookbook (全场景工具定义速查)

## 1. 开篇段落与设计哲学

在 Agent 开发中，**Tool Schema 是你写给模型的“操作手册”**。一个平庸的 Schema 仅仅定义了 API 接口；而一个优秀的 Schema 能显著降低模型的幻觉率，提高参数填充的准确性，并隐式地包含“思维链（CoT）”引导。

本附录汇集了 **5 大类、20+ 个** 经过生产环境验证的 JSON Schema 模板。它们专为 GPT-4o、Claude 3.5 Sonnet 等强模型设计，特别强化了多模态感知与工程落地的细节。

**设计黄金法则 (Golden Rules of Schema Design)：**

1. **Description 是核心提示词**：不要只写 "Search Google"，要写 "Search Google when the user asks about current events or verified facts."。
2. **防御性枚举 (Defensive Enums)**：尽可能将字符串参数限制为 `enum`，这能物理阻断模型生成不存在的选项。
3. **副作用显式化**：在描述中明确告知模型该工具是“只读”的还是“有破坏性”的。
4. **多模态坐标系**：涉及视觉操作时，必须在描述中统一坐标系标准（如：[0-1000] 归一化坐标）。

---

## 2. 核心工具集详解

### 2.1 浏览器与深度检索 (Deep Web Access)

不仅仅是简单的搜索，而是构建“搜索-浏览-提取”的完整链路。

#### [Tool 01] 智能搜索引擎 (Smart Search)

* **设计亮点**：加入了 `domain_filter` 和 `search_intent`，引导模型在搜索前先思考搜索的目的。

```json
{
  "name": "web_search_advanced",
  "description": "Performs a web search using a search engine. Optimized for retrieving factual information, technical documentation, or news.",
  "parameters": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The search query. Include specific keywords to narrow down results."
      },
      "search_intent": {
        "type": "string",
        "enum": ["informational", "navigational", "troubleshooting", "academic"],
        "description": "The intent behind the search to optimize ranking algorithms."
      },
      "time_range": {
        "type": "string",
        "enum": ["any", "past_24h", "past_week", "past_month", "past_year"],
        "description": "Filter results by publication date."
      },
      "domain_filter": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Optional list of domains to include (e.g., ['github.com', 'stackoverflow.com']) or exclude (prefix with -)."
      }
    },
    "required": ["query"]
  }
}

```

#### [Tool 02] 网页多模态阅读器 (Webpage Reader)

* **设计亮点**：区分文本提取与截图。对于复杂的 CSS 布局或图表，强制模型使用截图模式。

```json
{
  "name": "read_webpage_content",
  "description": "Scrapes and parses the content of a specific URL. Can return text structure or visual snapshots.",
  "parameters": {
    "type": "object",
    "properties": {
      "url": { "type": "string" },
      "mode": {
        "type": "string",
        "enum": ["markdown_text", "screenshot_full", "screenshot_viewport"],
        "description": "Use 'markdown_text' for articles. Use 'screenshot' for dashboards, charts, or complex layouts."
      },
      "wait_for_selector": {
        "type": "string",
        "description": "Optional CSS selector to wait for (dynamic loading) before scraping."
      }
    },
    "required": ["url", "mode"]
  }
}

```

---

### 2.2 多模态文件处理 (PDF/Image/Video)

这是多模态 Agent 的“眼睛”。重点在于**节省 Token** 和 **聚焦关注点**。

#### [Tool 03] PDF 智能片与光栅化 (PDF Rasterizer)

* **设计亮点**：不一次性丢给模型整个 PDF。允许模型按需“翻页”或“放大”。

```json
{
  "name": "inspect_pdf_page",
  "description": "Renders a specific page of a PDF document into an image for visual analysis (VLM).",
  "parameters": {
    "type": "object",
    "properties": {
      "file_path": { "type": "string" },
      "page_number": {
        "type": "integer",
        "description": "1-based page index."
      },
      "zoom_level": {
        "type": "number",
        "default": 1.0,
        "description": "Zoom factor (1.0 to 3.0). Use >1.5 to read small footnotes or dense tables."
      },
      "region_of_interest": {
        "type": "array",
        "items": { "type": "integer" },
        "minItems": 4,
        "maxItems": 4,
        "description": "Optional [ymin, xmin, ymax, xmax] (0-1000) to crop specific area instead of full page."
      }
    },
    "required": ["file_path", "page_number"]
  }
}

```

#### [Tool 04] 视频抽帧与摘要 (Video Sampler)

* **设计亮点**：视频是大容量数据，必须通过采样策略来处理。

```json
{
  "name": "sample_video_frames",
  "description": "Extracts frames from a video file based on a sampling strategy.",
  "parameters": {
    "type": "object",
    "properties": {
      "video_path": { "type": "string" },
      "strategy": {
        "type": "string",
        "enum": ["uniform_interval", "scene_change_detection", "timestamp_list"],
        "description": "'uniform_interval': every N seconds. 'scene_change': smart detection. 'timestamp_list': specific moments."
      },
      "interval_seconds": {
        "type": "number",
        "description": "Required if strategy is 'uniform_interval'."
      },
      "timestamps": {
        "type": "array",
        "items": { "type": "number" },
        "description": "Required if strategy is 'timestamp_list'."
      }
    },
    "required": ["video_path", "strategy"]
  }
}

```

---

### 2.3 软件工程与代码执行 (Coding Agent)

此类工具风险最高，Schema 必须包含**安全约束**和**精确的文件定位**。

#### [Tool 05] 精准文件编辑器 (File Patcher)

* **设计亮点**：避免重写整个文件（浪费 Token 且易错）。采用 `search_block` + `replace_block` 的模式，这比 Unified Diff 对 LLM 更友好。

```json
{
  "name": "edit_source_code",
  "description": "Edits a file by replacing a specific block of text. Locate the unique block first.",
  "parameters": {
    "type": "object",
    "properties": {
      "file_path": { "type": "string" },
      "search_block": {
        "type": "string",
        "description": "The exact unique text block to be replaced. Must match the file content character-by-character."
      },
      "replace_block": {
        "type": "string",
        "description": "The new code block to insert."
      }
    },
    "required": ["file_path", "search_block", "replace_block"]
  }
}

```

#### [Tool 06] 仓库地图生成器 (Repo Mapper)

* **设计亮点**：用于型项目，解决“不知道文件在哪”的问题。

```json
{
  "name": "generate_repo_map",
  "description": "Generates a tree structure or dependency graph of the codebase.",
  "parameters": {
    "type": "object",
    "properties": {
      "root_dir": {
        "type": "string",
        "default": "."
      },
      "depth": {
        "type": "integer",
        "default": 2,
        "description": "Depth of the directory tree traversal."
      },
      "include_definitions": {
        "type": "boolean",
        "description": "If true, lists class and function names (signatures) alongside filenames."
      }
    },
    "required": ["root_dir"]
  }
}

```

---

### 2.4 具身智能与 VLA (Embodied & Simulation)

这里的关键是**动作原语 (Primitives)** 的定义。

#### [Tool 07] 机械臂末端控制 (End Effector Control)

* **设计亮点**：将复杂的逆运动学（IK）封装，只暴露目标位姿。

```json
{
  "name": "move_robot_arm",
  "description": "Moves the robot's end-effector to a target pose in 3D space.",
  "parameters": {
    "type": "object",
    "properties": {
      "target_position": {
        "type": "array",
        "items": { "type": "number" },
        "minItems": 3,
        "maxItems": 3,
        "description": "[x, y, z] coordinates in meters relative to robot base."
      },
      "target_orientation_rpy": {
        "type": "array",
        "items": { "type": "number" },
        "minItems": 3,
        "maxItems": 3,
        "description": "[roll, pitch, yaw] Euler angles in degrees."
      },
      "motion_type": {
        "type": "string",
        "enum": ["linear", "joint"],
        "description": "'linear' for straight line (cartesian), 'joint' for fastest path."
      }
    },
    "required": ["target_position"]
  }
}

```

#### [Tool 08] 视觉导航 (Visual Navigation)

* **设计亮点**：结合语义理解的导航，而非纯坐标。

```json
{
  "name": "navigate_to_landmark",
  "description": "Navigates the mobile base to a semantic landmark visible in the map or memory.",
  "parameters": {
    "type": "object",
    "properties": {
      "landmark_name": {
        "type": "string",
        "description": "e.g., 'kitchen_counter', 'charging_station', 'red_sofa'."
      },
      "proximity_threshold": {
        "type": "number",
        "default": 1.0,
        "description": "Distance in meters to consider 'arrived'."
      }
    },
    "required": ["landmark_name"]
  }
}

```

---

### 2.5 记忆管理与系统控制 (Memory & System)

赋予 Agent 长期记忆和自我反思的能力。

#### [Tool 09] 知识库写入 (Upsert Memory)

* **设计亮点**：强制模型对知识进行分类和打标，防止记忆库变成垃圾场。

```json
{
  "name": "save_to_long_term_memory",
  "description": "Saves critical information/facts to long-term vector storage.",
  "parameters": {
    "type": "object",
    "properties": {
      "content": {
        "type": "string",
        "description": "The fact, code snippet, or user preference to save."
      },
      "tags": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Keywords for retrieval (e.g., 'project_A', 'user_preference')."
      },
      "importance": {
        "type": "integer",
        "minimum": 1,
        "maximum": 5,
        "description": "1=Trivial, 5=Critical/Permanent."
      }
    },
    "required": ["content", "tags"]
  }
}

```

---

## 3. 本章小结：Schema 设计检查清单

在部署任何新工具前，请按照以下清单进行 Self-Check：

1. **[原子性]**：这个工具是否做的事情太多？（如果一个工具既搜索又下载还解析，请拆分它）。
2. **[鲁棒性]**：如果模型传入了空字符串或负数，后端会崩溃还是返回友好的 Error Message？
3. **[上下文]**：工具的返回值是否包含足够的上下文？（例如：搜索结果不仅要返回 URL，还要返回 Title 和 Snippet）。
4. **[多模态]**：如果涉及图片，ID 系统是否统一？（不要混用 File Path 和 File ID）。
5. **[成本]**：默认参数是否足够“省钱”？（例如默认只获取 Top-3 结果，而非 Top-20）。

---

## 4. 练习题

### 基础题

**1. 设计“天气查询”工具**
要求：支持城市名查询，必须包含日期（今天/明天/未来一周），且能够选择单位（摄氏度/华氏度）。

<details>
<summary>点击展开参考答案</summary>

**提示**：注意单位的默认值设置，这符合用户习惯。

```json
{
  "name": "get_weather_forecast",
  "description": "Retrieves weather forecast for a specific location and time range.",
  "parameters": {
    "type": "object",
    "properties": {
      "location": {
        "type": "string",
        "description": "City name, or 'City, Country'."
      },
      "date_scope": {
        "type": "string",
        "enum": ["current", "tomorrow", "7_days"],
        "default": "current"
      },
      "unit": {
        "type": "string",
        "enum": ["celsius", "fahrenheit"],
        "default": "celsius"
      }
    },
    "required": ["location"]
  }
}

```

</details>

**2. 设计“日程管理”工具**
要求：创建一个会议日程。字段包含：主题、开始时间、持续时长、参与者列表（邮箱）。

<details>
<summary>点击展开参考答案</summary>

**提示**：时间格式是初学者最容易出错的地方，必须在 description 中指定 ISO 8601。

```json
{
  "name": "schedule_meeting",
  "description": "Schedules a new meeting in the calendar.",
  "parameters": {
    "type": "object",
    "properties": {
      "subject": { "type": "string" },
      "start_datetime": {
        "type": "string",
        "description": "ISO 8601 format (e.g., '2023-10-27T14:00:00Z')."
      },
      "duration_minutes": {
        "type": "integer",
        "minimum": 15,
        "default": 30
      },
      "attendees": {
        "type": "array",
        "items": {
          "type": "string",
          "format": "email"
        }
      }
    },
    "required": ["subject", "start_datetime", "attendees"]
  }
}

```

</details>

### 挑战题 (进阶场景)

**3. 多模态 OCR 后处理工具**
背景：OCR 识别出的表格往往格式混乱。
任务：设计一个工具，输入是一段混乱的 OCR 文本和原始图片的 Bounding Box，功能是让 Agent 重构出 Markdown 表格。

<details>
<summary>点击展开参考答案</summary>

**思路**：这个工具的输入既有文本又有视觉坐标，是典型的多模态融合工具。

```json
{
  "name": "refine_ocr_table",
  "description": "Corrects and formats structureless OCR text into a clean Markdown table using visual layout information.",
  "parameters": {
    "type": "object",
    "properties": {
      "raw_ocr_text": {
        "type": "string",
        "description": "The messy text output from OCR."
      },
      "bbox": {
        "type": "array",
        "items": { "type": "integer" },
        "description": "[ymin, xmin, ymax, xmax] of the table in the original image."
      },
      "expected_columns": {
        "type": "integer",
        "description": "Hint: estimated number of columns based on visual inspection."
      },
      "headers_hint": {
        "type": "array",
        "items": { "type": "string" },
        "description": "Optional list of potential header names seen in the image."
      }
    },
    "required": ["raw_ocr_text", "bbox"]
  }
}

```

</details>

**4. 数据库安全查询 (SQL Generator vs Executor)**
任务：为了防止 Agent 直接执行危险 SQL，设计一套“两步走”的工具组。
工具 A：`generate_sql_plan` (只生成，不执行)
工具 B：`execute_approved_sql` (需要传入 plan_id)

<details>
<summary>点击展开参考答案</summary>

**思路**：这种模式叫做“Human-in-the-loop”或“System-Check-in-the-loop”。

```json
// Tool A: Generate Plan
{
  "name": "generate_sql_plan",
  "description": "Generates a SQL query for a user request but does NOT execute it. Returns a plan_id.",
  "parameters": {
    "type": "object",
    "properties": {
      "intent": { "type": "string" },
      "sql_draft": { "type": "string" }
    },
    "required": ["intent", "sql_draft"]
  }
}

// Tool B: Execute (Requires Plan ID)
{
  "name": "execute_approved_sql",
  "description": "Executes a pre-generated and system-validated SQL plan.",
  "parameters": {
    "type": "object",
    "properties": {
      "plan_id": {
        "type": "string",
        "description": "The ID returned by generate_sql_plan."
      }
    },
    "required": ["plan_id"]
  }
}

```

</details>

---

## 5. 常见陷阱与调试技巧 (Gotchas)

### 1. 数组与对象的混淆

* **错误**：`items: { "type": "string" }` (这是一个字符串数组) vs `items: { "type": "object", ... }` (这是对象数组)。
* **现象**：模型经常生成 `["name": "A"]` 这种非法 JSON。
* **Rule of Thumb**：如果列表项超过 1 个属性，始终使用**对象数组**。

### 2. 默认值的隐形坑

* **错误**：Schema 里写了 `default: 10`，但后端代码里没有处理 `None` 或 `null` 的况。
* **现象**：模型有时会显式传 `null`，有时完全不传参数。
* **调试**：在后端代码中，始终做一次 `value = args.get('key') or DEFAULT_VALUE` 的防御性编程，不要完全依赖 Schema 的 default 描述（因为有些 LLM 可能会忽略它）。

### 3. 多模态的“分辨率”问题

* **现象**：Agent 看着 1080p 的大图，却看不清里面的小字。
* **原因**：视觉模型通常会将图片 Resize 到固定大小（如 224x224 或 1024x1024）。
* **解决**：必须提供 `crop` 工具。如果 Agent 抱怨“看不清”，提示它使用 crop 工具截取局部放大。

### 4. JSON Schema 复杂度上限

* **现象**：Schema 定义非常完美，包含了 `oneOf`、`anyOf`、条件依赖逻辑。
* **后果**：模型（尤其是 7B/13B 级别的小模型）完全无法理解复杂的 JSON Schema 逻辑，导致调用失败。
* **Rule of Thumb**：**Keep It Simple (K.I.S.S)**。如果参数逻辑太复杂，不如拆成两个不同的工具。
