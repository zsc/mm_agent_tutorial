# 第 12 章 Coding Agent：从仓库理解到可合并 PR

> **本章摘要**
> 编写一个能写出 `Hello World` 的脚本很容易，但构建一个能在一个百万行代码库中修复竞态条件（Race Condition）Bug 的 Agent 却极具挑战。
> 本章将从“软件工程”而非单纯“代码生成”的视角，系统拆解 Coding Agent 的架构。我们将深入探讨如何利用 **Repo Map** 解决上下文限制，如何设计**非破坏性的文件修改协议**，以及如何构建 **TDD（测试驱动开发）** 闭环来确保代码不仅能运行，而且是正确的。
> **学习目标**
> 1. **架构认知**：理解 Coding Agent 的核心不是 LLM，而是“境交互循环”（Environment Interaction Loop）。
> 2. **上下文工程**：掌握利用 AST（抽象语法树）和 Call Graph（调用图）压缩代码库的方法。
> 3. **工具设计**：学会设计 `search_and_replace` 等鲁棒的编辑工具，而非脆弱的行号编辑。
> 4. **闭环验证**：建立“复现-修复-回归”的自动化测试策略。
> 
> 

---

## 12.1 任务定义：软件工程不仅仅是 Coding

在多模态 Agent 的语境下，Coding Agent 的输入往往是一个含糊的 Issue（可能包含报错截图、日志片段、用户描述），输出则是一个经过验证的 Pull Request (PR)。

### 12.1.1 典型能力层级 (The Hierarchy of Competence)

我们将 Coding Agent 的能力划分为 L1~L4 四个层级：

* **L1 - Snippet Generator (代码片段生成)**：用户给函数名，Agent 补全函数体。这是 Copilot 的模式，不属于 Agent。
* **L2 - File-Level Editor (单文件编辑)**：用户提供文件，Agent 进行重构或 Debug。下文完全包含在 Prompt 中。
* **L3 - Repository-Level Navigator (仓库级导航)**：Agent 需要自己去 `ls`, `grep`, `find`，理解跨文件依赖，定位问题所在，并处理多个文件的修改。
* **L4 - Visual-Augmented Engineer (视觉增强工程师)**：结合前端渲染截图、UI 布局树、控制台报错截图，修复 CSS 错位或前端逻辑。

本章重点关注 **L3** 和 **L4**。

### 12.1.2 SWE-Agent 工作流参考模型

一个成熟的 Coding Agent 工作流是一个有状态的决策图：

```text
[Start: GitHub Issue]
      |
      v
[Phase 1: Exploration & Context] <-----------------------+
|  1. Read Issue Description                             |
|  2. List Files (Repo Tree)                             |
|  3. Search Keywords (Grep/Symbol Search)               |
|  4. Build "Mental Model" of relevant code              |
+-----+--------------------------------------------------+
      |
      v
[Phase 2: Reproduction (The "Red" State)] <-------------+
|  1. Create reproduction_script.py                      |
|  2. Run script -> Capture STDOUT/STDERR                |
|  3. ANALYZE: Does it fail as expected?                 |
|     (No -> Go back to Exploration)                     |
+-----+--------------------------------------------------+
      |
      v
[Phase 3: Coding & Patching (The "Fix" State)] <--------+
|  1. Read Implementation Details (Source Code)         |
|  2. Plan Modification (Think Step)                    |
|  3. Apply Edit (Tool Call)                            |
|  4. Handle Edit Errors (e.g., Linter/Syntax)          |
+-----+-------------------------------------------------+
      |
      v
[Phase 4: Validation (The "Green" State)]
|  1. Run reproduction_script.py (Expect Pass)          |
|  2. Run Existing Tests (Regression Check)             |
|  3. Visual Verification (If UI related)               |
+-----+-------------------------------------------------+
      |
      +---> [Success] ---> Submit PR
      |
      +---> [Fail] ------> Return to Phase 3 (Refine)

```

---

## 12.2 上下文经济学：如何在 Prompt 中装下 Linux 内核？

最大的工程挑战在于 Token 预算。直接把所有文件塞进 context window 是不可行的（既贵又会降低模型注意力）。我们需要一种**有损压缩**但保留**语义结构**的表示方法。

### 12.2.1 策略一：Repo Map (基于 AST 的骨架图)

Repo Map 是一种通过静态分析生成的代码摘要。它只保留“路标”，隐去“风景”。

**压缩规则 (Rule of Thumb)**：

1. 保留所有 `class` 和 `def` 定义行。
2. 保留 Docstring（如果 Token 允许，或只保留第一行）。
3. **核心技巧**：保留函数签名（Signature）和类型注解（Type Hints）。
4. 将函数体替换为 `...` 或 `pass`。
5. 根据 PageRank 或引用次数，标记“重要文件”与“边缘文件”。

**ASCII 示意图：原始代码 vs. Repo Map**

```text
[原始代码 main.py (100行)]          [Repo Map 表示 (5行)]
import utils                        
class Server:                       class Server:
    def __init__(self, port):           def __init__(self, port): ...
        self.port = port                def start(self): ...
        self.log = []               
        # ... 20 lines ...          def helper_func(x: int) -> bool: ...
    def start(self):
        print("Starting...")
        # ... 50 lines logic ...

```

### 12.2.2 策略二：动态展开 (Just-In-Time Context)

不要预加载。Agent 应该像人类在 IDE 中一样，点击哪个文件，才加载哪个文件的内容。

* **初始状态**：只给出根目录的文件树列表。
* **按需读取**：Agent 调用 `read_file(path)`。
* **滑动窗口**：对于超大文件（如 5000 行的遗留代码），强制要求 Agent 使用 `read_file(path, start_line, end_line)`，禁止全量读取。

### 12.2.3 策略三：基于检索的增强 (RAG for Code)

对于未知的库或巨大的项目，使用 embedding 检索：

* **Query**: "Where is the authentication logic handled?"
* **Retrieval**: 返回相关性最高的 5 个代码片段（chunk）。
* **注意**：代码 RAG 的 chunking 必须尊重语法边界（不能把一个函数切成两半）。通常按 Function 或 Class 为单位切分。

---

## 12.3 工具链深度解析：手术刀而非大铁锤

工具的设计直接决定了 Agent 的修改成功率。最糟糕的设计是 `overwrite_file`（重写整个文件），最好的设计是基于上下文的 `patch`。

### 12.3.1 核心工具 Schema 清单

| 工具名 | 关键参数 | 设计意图与工程细节 |
| --- | --- | --- |
| `ls` / `list_files` | `path` | 探索目录结构。**Gotcha**: 必须限制递归深度，防止打印出 `node_modules` 的几万个文件。 |
| `search` | `query`, `file_pattern` | 类似 `grep -n`。让 Agent 找到关键词所在的行号。 |
| `read` | `path`, `start_line`, `end_line` | **关键特性**: 输出必须带行号（Line Numbers），方便后续引用。 |
| `edit_replace` | `path`, `search_block`, `replace_block` | **推荐**: 类似 `sed` 但更智能。Agent 提供“要查找的几行代码（作为锚点）”和“替换后的代码”。系统自动匹配锚点并替换。 |
| `run_cmd` | `command`, `timeout` | 执行 Shell 命令。**安全限制**: 只能在沙箱内运行，禁止网络请求（除非是白名单）。 |
| `create_file` | `path`, `content` | 用于创建新的测试脚本或新模块。 |

### 12.3.2 为什么“基于行号的编辑”是陷阱？

初学者常设计 `edit_line(file, line_number, new_content)`。

* **问题**：LLM 对数字非常不敏感（Hallucination）。且在多轮对话中，Agent 脑海中的行号可能因为之前的修改而过时（Off-by-one errors）。
* **解决方案**：使用 **Search-and-Replace** 范式。
* Agent 必须引用原文件中**独一无二**的 3-5 行代码作为 `context`。
* 系统在文件中查找这个块。
* 如果找到多处或找不到，工具返回 Error，强迫 Agent 重新阅读文件。



### 12.3.3 多模态特有工具

对于前端或 GUI 任务，需要增加：

* `capture_screenshot(url/path)`: 返回图片的 base64。
* `get_render_tree()`: 返回简化版的 DOM 树或 Accessibility Tree。
* `visual_diff(img1, img2)`: 辅助工具，返回两张图的差异热力图（虽然 VLM 可以自己看，但工具辅助更精确）。

---

## 12.4 决策大脑：TDD 与反思机制

Agent 极其容易产生幻觉——它以为代码是对的，但实际上跑不通。必须用解释器作为“物理定律”来约束它。

### 12.4.1 “复现优先”原则 (Reproduction First)

在修改任何业务代码前，强制 Agent 必须先写一个 **Test Case**。

1. **Agent 思考**：根据 Issue 描述，这个 Bug 应该在输入 X 时产生输出 Y，但实际输出了 Z。
2. **Agent 行动**：创建一个 `reproduce_issue.py`。
3. **Agent 验证**：运行该脚本。
* 如果脚本**成功报错**：太好了，我们复现了 Bug。
* 如果脚本**运行通过**：说明 Agent 没理解对 Bug，或者环境不对。**禁止进入修复阶段**。



### 12.4.2 错误恢复与自愈 (Error Recovery)

当 `edit` 工具调用失败，或测试运行报错时，Agent 需要进入 ReAct 循环：

* **Syntax Error**: 将 traceback 喂回给 Agent。Agent 通常能自我修正拼写错误。
* **Linter Error**: 比如 Python 的缩进错误或未使用的 import。
* **Timeout**: 可能是死循环。系统强制 kill 并告知 Agent。

---

## 12.5 关键算法与策略：如何生成可合并的 PR

### 12.5.1 依赖感知修改 (Dependency-Aware Editing)

修改代码最怕“破坏了其他地方”。

* **策略**：在修改 `function_A` 后，Agent 应该主动调用 `search("function_A")` 查找所有调用方。
* **LSP 集成**：如果基础设施允许，集成 Language Server Protocol，提供 `find_references` 工具。

### 12.5.2 提交信息的规范化

PR 不仅仅是代码，还有描述。

* **自动生成 PR Description**：让 Agent 总结它的修改：
* *What*: 修改了哪个文件？
* *Why*: 解决了什么 Issue？
* *Risk*: 有什么潜在风险？
* *Verification*: 用什么测试验证的？



---

## 12.6 本章小结

* **从代码到工程**：Coding Agent 的核心难点不在于写出正确的语法，而在于理解庞大的上下文和复杂的依赖关系。
* **地图优于全景**：使用 Repo Map 和 AST 摘要来对抗 Token 限制。
* **锚点优于坐标**：使用基于内容的 Search-Replace 修改策略，避免行号幻觉。
* **测试即真理**：没有复现脚本的修复是不可信的。强制执行“复现 -> 修复 -> 回归”的 TDD 流程。

---

## 12.7 练习题

每道题包含提示（Hint），答案默认折叠。

### 基础题（熟悉机制）

<details>
<summary><strong>练习 1：Repo Map 的压缩策略</strong></summary>

**背景**：你有一个 200 行的 Python 文件，包含 3 个类和 15 个方法。你需要将其压缩进 Prompt，让 Agent 决定是否需要详细阅读该文件。
**问题**：请写出一个伪代码函数 `generate_skeleton(file_content)`，描述你会提取哪些信息，丢弃哪些信息？

**提示**：关注 AST (Abstract Syntax Tree) 的节点类型。保留结构，丢弃实现。

```
        # 2. 如果有 Docstring，保留第一行
        if has_docstring(node):
            skeleton.append(indent + '"""' + get_summary(node.docstring) + '"""')
        
        # 3. 丢弃具体实现，用占位符
        skeleton.append(indent + "...")
        
    # 4. 保留全局变量定义
    elif isinstance(node, Assign) and is_global(node):
        skeleton.append(get_source_segment(code, node))
        
return "\n".join(skeleton)

```

```
关键点：保留接口（Signature），隐藏实现（Implementation）。
</details>

<details>
<summary><strong>练习 2：非破坏性编辑</strong></summary>

**背景**：Agent 想要修改 `config.py` 中的数据库端口。
原代码：
```python
DB_HOST = "localhost"
DB_PORT = 5432
DB_USER = "admin"

```

Agent 发出的指是：“把 5432 改成 5433”。

**问题**：设计一个 `edit_file` 的工具调用 payload，确保只有当 `DB_PORT` 确实是 `5432` 时才修改，防止覆盖了其他人的并发修改。

**提示**：使用 `search_block` （查找块）作为锚点。

<details>
<summary><strong>练习 3：工具安全性</strong></summary>

**问题**：Agent 为了清理环境，请求执行 `run_cmd("rm -rf ./temp_output")`。如何设计安全策略防止 Agent 意外删除项目代码？

**提示**：沙箱（Sandbox）、路径白名单、Git 恢复。

### 挑战题（工程深水区）

<details>
<summary><strong>练习 4：解决“Import 循环”与幻觉依赖</strong></summary>

**场景**：Agent 在 `utils.py` 中引入了 `main.py` 的函数，导致了 Circular Import 错误。同时，它还试图 `import pandas`，但环境中根本没有安装 pandas。
**问题**：设计一个具体的 **Prompt Template** 或 **Workflow**，当发生 `ImportError` 时，引导 Agent 自主解决这个问题，而不是反复重试。

**提示**：不仅要给报错信息，还要给环境信息（如 `pip freeze` 的结果）。

<details>
<summary><strong>练习 5：处理“鬼打墙” (The Loop of Death)</strong></summary>

**场景**：Agent 修改了代码 -> 运行测试失败 -> 再修改（改回去了） -> 运行测试失败。这在 Agent 开发中极为常见。
**问题**：设计一个**检测器（Detector）**和**阻断器（Interrupter）**机制。

**提示**：状态哈希（State Hash）。

<details>
<summary><strong>练习 6：多模态前端 Debug</strong></summary>

**场景**：用户反馈“登录按钮被底部的 Banner 遮住了”。这不是代码逻辑错误，是 CSS 样式问题。
**问题**：作为一个多模态 Coding Agent，你需要哪些**特定**的观测（Observation）数据才能修复这个问题？仅看 HTML 代码够吗？

**提示**：Bounding Box, Viewport, Screenshot.

---

## 12.8 常见陷阱与错误 (Gotchas)

### 1. **"Lazy Coder" 现象**

* **错误表现**：Agent 在修改文件时，为了省事，输出了 `// ... existing code ...`。如果直接写入文件，会导致代码丢失。
* **调试技巧**：在 System Prompt 中严厉禁止省略。或者，在工具层（Tool Level）检测到输出包含 `...` 且行数大幅减少时，拒绝写入并报错：“Detected lazy output. You MUST provide the FULL code block for the replacement.”

### 2. **测试污染 (Test Pollution)**

* **错误表现**：Agent 为了通过测试，修改了数据库状态，但没有在 `teardown` 中清理。导致下一个测试因为数据冲突而失败。
* **调试技巧**：强制 Agent 使用 `pytest` 的 fixture 或事务回滚机制。在评测时，每个 Task 使用独立的 Docker 容器，确保环境隔离。

### 3. **上下文迷失 (Lost in the Middle)**

* **错误表现**：一次性读入太多文件，LLM 忘记了最开始读到的需求，开始胡乱修改。
* **调试技巧**：监控 Context Usage。当 Token 数超过阈值（如 32k）时，触发“记忆压缩”机制——让 Agent 自己总结当前已知信息，清空历史消息，只保留总结。

### 4. **行尾空格与缩进地狱**

* **错误表现**：Python 代码中，Agent 混用了 Tab 和 Space，或者在空行里留了空格，导致代码无法运行但肉眼看不出。
* **调试技巧**：在写入文件前，工具层自动运行 formatter (如 `black` 或 `autopep8`)，或者在显示给 Agent 时显式渲染空格字符（如 `·`）。
