# HIPPO-Agent Prompts（提取与中文翻译）

本文件按源码提取了“会被模型直接看到的提示词（Prompt）”，并给出中文翻译。  
说明：

- `已启用`：当前代码路径会实际使用。
- `模板/未接线`：代码中存在，但当前主流程未直接调用。
- 变量占位符（如 `TASK_DESCRIPTION`、`CURRENT_OS`）保留原样。

---

## 1) Worker 主系统提示词（已启用）

- 来源：`muscle_mem/memory/procedural_memory.py` `construct_simple_worker_procedural_memory`
- 使用：`muscle_mem/agents/worker.py`（构造 `self.generator_agent`）
- 状态：已启用

原文即中文（节选特征）：

- 角色：GUI + Python/Bash 自动化专家
- 策略：Code Agent 优先，GUI 兜底，复杂任务用 Todo，完成后必须验证
- 约束：每步一个工具，禁止用 Code Agent 创建图表/视觉元素，完成调用 `done()`，不可完成调用 `fail()`

中文翻译：原文即中文，无需翻译。

---

## 2) 可行性代理系统提示词（已启用）

- 来源：`muscle_mem/memory/procedural_memory.py` `INFEASIBLE_AGENT_PROMPT`
- 使用：`muscle_mem/agents/infeasible_agent.py`
- 状态：已启用

原文即中文（节选特征）：

- 强调“只验证可行性，不执行任务”
- 默认环境假设（有网、有权限、字体齐全）
- 明确“能力隔离”：验证者工具仅用于只读核验
- 明确触发 `report_infeasible` 的四类条件

中文翻译：原文即中文，无需翻译。

---

## 3) 验证代理系统提示词（当前实现可用，主流程接线停用）

- 来源：`muscle_mem/memory/procedural_memory.py` `VERIFICATION_AGENT_PROMPT`
- 使用：`muscle_mem/agents/verification_agent.py`
- 状态：模板在，但在 `grounding.py` 主流程中未接线

原文即中文（节选特征）：

- 目标：基于任务描述 + 前后截图验证任务是否完成
- 工具：GUI 核验、只读 `call_code_agent`、`report_verification_plan`、`report_verification_result`
- 结论枚举：`IMPOSSIBLE / ERROR / SUCCESS`

中文翻译：原文即中文，无需翻译。

---

## 4) 代码代理系统提示词（已启用，来自 `build_system_prompt`）

- 来源：`muscle_mem/agents/motor_code_agent.py` `build_system_prompt`
- 使用：`CodeAgent.__init__`
- 状态：已启用

原文即中文（节选特征）：

- 角色：资深软件工程师，必须用工具执行，完成后调用 `Done`
- 规则：数据类型保真、文档结构保真、优先就地修改、最后一步打印修改后的文件内容
- 细节：sudo 使用方式、文档自动编号结构校验、Excel/Docx 修改验证要求

中文翻译：原文即中文，无需翻译。

---

## 5) 格式纠错提示词（已启用）

- 来源：`muscle_mem/memory/procedural_memory.py` `FORMATTING_FEEDBACK_PROMPT`
- 使用：`muscle_mem/utils/common_utils.py` `call_llm_formatted`
- 状态：已启用（当格式检查失败时）

原文：

```text
Your previous response was not formatted correctly. You must respond again to replace your previous response. Do not make reference to this message while fixing the response. Please address the following issues below to improve the previous response:
FORMATTING_FEEDBACK
```

中文翻译：

```text
你上一条回复的格式不正确。你必须重新回复以替换上一条回复。修复时不要提及这条消息本身。请根据下面的问题修正你的回复：
FORMATTING_FEEDBACK
```

---

## 6) 文本定位系统提示词（已启用）

- 来源：`muscle_mem/memory/procedural_memory.py` `PHRASE_TO_WORD_COORDS_PROMPT`
- 使用：`muscle_mem/agents/grounding.py` `self.text_span_agent`
- 状态：已启用

原文（核心）：

```text
You are an expert in graphical user interfaces. Your task is to process a phrase of text, and identify the most relevant word on the computer screen.
...
1. First, think step by step and generate your reasoning about which word id to click on.
2. Then, output the unique word id.
3. If there are multiple occurrences of the same word, use the surrounding context...
```

中文翻译：

```text
你是图形界面专家。你的任务是根据给定短语，在屏幕文本表与截图中识别最相关的“单词ID”。

你会看到：
1) 一个短语；
2) 屏幕文本表（每行含“word id + 单词”）；
3) 屏幕截图。

要求：
1. 先逐步思考并给出推理；
2. 然后只输出唯一的目标 word id（即文本表每行第一个数字）；
3. 若同词多次出现，必须结合上下文、标点和大小写定位正确项。
```

---

## 7) Grounding 坐标查询用户提示词（已启用）

- 来源：`muscle_mem/agents/grounding.py` `generate_coords`
- 状态：已启用

原文：

```text
Query:{ref_expr}
Output only the coordinate of one point in your response.
```

中文翻译：

```text
查询：{ref_expr}
请只输出一个点的坐标。
```

---

## 8) OCR 文本对齐附加提示词（已启用）

- 来源：`muscle_mem/agents/grounding.py` `generate_text_coords`
- 状态：已启用

原文：

```text
**Important**: Output the word id of the FIRST word in the provided phrase.
**Important**: Output the word id of the LAST word in the provided phrase.
Phrase: {phrase}
...
Screenshot:
```

中文翻译：

```text
**重要**：输出所给短语中“第一个词”的 word id。
**重要**：输出所给短语中“最后一个词”的 word id。
短语：{phrase}
...
截图：
```

---

## 9) Worker 的 Todo 重采样提示词（已启用）

- 来源：`muscle_mem/agents/worker.py` `_build_todo_retry_messages`
- 状态：已启用（首次 TodoWrite 过短时）

原文即中文：

```text
上一次 TodoWrite 候选如下，请给出更有利于 Agent 替用户完成操作的 TodoWrite 候选。
请直接返回 TodoWrite 工具调用。

需要执行的任务：{instruction}

上一次 TodoWrite items：
{items_payload}
```

中文翻译：原文即中文，无需翻译。

---

## 10) Worker 的 Todo 候选选择提示词（已启用）

- 来源：`muscle_mem/agents/worker.py` `_build_selection_messages`
- 状态：已启用（Todo guardrail 触发时）

原文即中文：

```text
你将看到原任务、当前截图，以及若干个 TodoWrite 候选结果。
请只选择一个最有可能正确执行任务的候选。
只返回 JSON：{"choice": <数字>}，不要解释。

任务：{instruction}

候选结果（编号从 1 开始）：
{candidates_text}
```

中文翻译：原文即中文，无需翻译。

---

## 11) Worker 过早 Done 纠正提示词（已启用）

- 来源：`muscle_mem/agents/worker.py`（done guardrail）
- 状态：已启用

原文即中文：

```text
你的任务不是提供建议，而是替用户执行他想做的事情。请确认任务是否完成。
```

中文翻译：原文即中文，无需翻译。

---

## 12) Sub-Agent（Pac-Agent）系统提示词（当前默认未通过 `@tool_action` 暴露）

- 来源：`muscle_mem/agents/subagent.py` `SUB_AGENT_TYPES["Pac-Agent"]["prompt"]`
- 状态：子代理框架存在，但调用入口默认注释

原文即中文：

```text
你是 **Meticulous GUI Executor (极度严谨的 GUI 执行者)**。在处理图形界面上的重复任务（如逐张处理图片、逐个清除红点）时，你必须确保列表中的**每一个**目标都被成功处理。绝不允许在未确认“清零”的情况下草率结束任务。
```

中文翻译：原文即中文，无需翻译。

---

## 13) Sub-Agent 摘要用户提示词（可用）

- 来源：`muscle_mem/agents/subagent.py` `_generate_summary`
- 状态：可用

原文（模板）：

```text
Task: {task_instruction}
Execution Steps...
Final Response...

Please provide a concise summary of the sub-agent session. Focus on:
1. The tools used and their outcomes
2. The sequence of actions taken
3. Any final response or conclusions
Keep the summary under 150 words and use clear, factual language.
```

中文翻译：

```text
任务：{task_instruction}
执行步骤...
最终回复...

请对本次子代理会话给出简洁摘要，重点包括：
1. 使用了哪些工具及其结果；
2. 动作执行顺序；
3. 最终回复或结论。

摘要限制在 150 词以内，语言清晰、客观、基于事实。
```

---

## 14) 代码执行摘要系统提示词（已启用）

- 来源：`muscle_mem/memory/procedural_memory.py` `CODE_SUMMARY_AGENT_PROMPT`
- 使用：`CodeAgent._generate_summary`
- 状态：已启用

原文（核心）：

```text
You are a code execution summarizer...
Summarize logic, outputs, progression...
Do not judge success/failure.
Include verification instructions for GUI agent when files were modified.
```

中文翻译：

```text
你是代码执行会话摘要器。你的职责是用清晰、客观的方式总结代码执行过程：
1. 每一步采用的逻辑与方法；
2. 每次工具执行产生的输出与结果；
3. 解法推进路径。

要求：
- 使用中立语气，不对成功/失败做主观评价；
- 聚焦“做了什么、产生了什么结果”；
- 若修改了文件，必须补充给 GUI Agent 的核验指引（哪些文件、预期状态、如何验证、是否还需后续 GUI 操作）。
```

---

## 15) 代码执行摘要用户提示词（已启用）

- 来源：`muscle_mem/agents/motor_code_agent.py` `_generate_summary`
- 状态：已启用

原文（模板）：

```text
Task: {task_instruction}
Execution Steps...
Please provide a concise summary...
1. logic used at each step
2. outputs/results of each tool execution
3. progression of solution approach
Do not make judgments about success/failure.
Keep under 150 words.
```

中文翻译：

```text
任务：{task_instruction}
执行步骤...

请提供简洁摘要，重点包括：
1. 每一步使用的逻辑；
2. 每次工具执行的输出与结果；
3. 解法推进过程。

不要评价成功或失败，只描述尝试与结果。摘要控制在 150 词内。
```

---

## 16) Code Agent 继续执行提示词（已启用）

- 来源：`muscle_mem/agents/motor_code_agent.py` `continue_prompt`
- 状态：已启用

原文即中文：

```text
请继续执行任务。完成后请务必调用 Done 工具结束。
```

中文翻译：原文即中文，无需翻译。

---

## 17) 可行性代理用户输入模板（已启用）

- 来源：`muscle_mem/agents/infeasible_agent.py`
- 状态：已启用

原文：

```text
用户任务:
{task_instruction}
```

中文翻译：原文即中文，无需翻译。

---

## 18) 验证代理用户输入模板（实现可用）

- 来源：`muscle_mem/agents/verification_agent.py`
- 状态：实现可用（主流程接线停用）

原文：

```text
Task description:
{task_instruction}

You will receive the second submitted and current screenshots.
Use them to verify completion and respond in the required format.
```

可选附加：

```text
CRITICAL: You are verifying the Code Agent's work.
```

中文翻译：

```text
任务描述：
{task_instruction}

你将收到“第二次提交截图”和“当前截图”。
请使用这些截图验证任务是否完成，并按要求格式回复。
```

可选附加：

```text
关键：你正在验证 Code Agent 的工作结果。
```

---

## 19) `web_fetch` 字段抽取系统提示词（已启用）

- 来源：`muscle_mem/agents/tools/exec_tools.py` `extract_prompt`
- 状态：已启用（`web_fetch(fields=...)` 时）

原文：

```text
Extract the requested fields from the markdown content.
Return a JSON object with keys exactly matching the requested fields.
If a field is missing, return an empty string for that field.
Return only JSON with no extra text.
```

中文翻译：

```text
请从 Markdown 内容中提取所请求字段。
返回一个 JSON 对象，键名必须与请求字段完全一致。
若某字段不存在，返回该字段为空字符串。
只返回 JSON，不要附加任何额外文本。
```

---

## 20) 通用默认系统提示词（回退）

- 来源：`muscle_mem/core/mllm.py`
- 状态：回退（未传 system prompt 时）

原文：

```text
You are a helpful assistant.
```

中文翻译：

```text
你是一个乐于助人的助手。
```

---

## 21) 反思代理系统提示词（模板/未接线）

- 来源：`muscle_mem/memory/procedural_memory.py` `REFLECTION_ON_TRAJECTORY`
- 状态：模板存在；当前 Worker 将 `enable_reflection` 置为 `False`

原文（核心）：

```text
You are an expert computer use agent designed to reflect on the trajectory...
Case 1: off-track/cycle; Case 2: on-track; Case 3: task completed.
Do not suggest specific next actions.
```

中文翻译：

```text
你是一个用于“轨迹反思”的计算机使用专家。你会看到任务描述与另一代理的当前轨迹（图像+思考+动作序列），并输出反思结论。

你只能输出三类之一：
1) 轨迹偏离计划（常见于循环无进展）：说明为什么偏离，并建议调整方向（但不能给具体下一步动作）；
2) 轨迹符合计划：简洁确认继续；
3) 任务已完成：明确指出已完成。

约束：
- 必须落在三类之一；
- 不得给具体行动计划；
- 注意代码代理可能导致“文件变化/应用重启”是正常行为，不应直接判错。
```

---

## 22) 行为叙述器系统提示词（模板/未见主流程接线）

- 来源：`muscle_mem/memory/procedural_memory.py` `BEHAVIOR_NARRATOR_SYSTEM_PROMPT`
- 状态：模板存在

原文（核心）：

```text
Analyze before/after screenshots for a computer action...
Use visual markers (Click/MoveTo/DragTo) and zoomed-in region.
Output with <thoughts> and <answer> tags.
```

中文翻译：

```text
你是“动作后行为变化分析器”。给定动作前后截图与动作代码，分析由该动作引起的可见变化。

要求：
- 关注动作标记（Click/MoveTo/DragTo）及其局部放大区域；
- 聚焦由动作导致的变化，不要被无关细节（如系统时间）干扰；
- 不要仅凭坐标猜测，必须以图像证据为准；
- 必须用 `<thoughts>...</thoughts>` 与 `<answer>...</answer>` 输出。
```

---

## 23) 轨迹对比评测系统提示词（模板/未见主流程接线）

- 来源：`muscle_mem/memory/procedural_memory.py` `VLM_EVALUATOR_PROMPT_COMPARATIVE_BASELINE`
- 状态：模板存在

原文（核心）：

```text
Judge <NUMBER OF TRAJECTORIES> trajectories and choose the better one.
Strictly evaluate against judge guidelines and user request.
Use final screenshots as factual evidence.
Output reasoning in <thoughts> and index in <answer>.
```

中文翻译：

```text
你是严格且中立的评测器，需要比较 `<NUMBER OF_TRAJECTORIES>` 条桌面操作轨迹，判断哪条更好完成用户请求（或更正确地判断“不可完成”）。

评测要求：
- 严格按评测准则逐条核对“满足/部分满足/不满足”；
- 必须基于截图事实，尤其 final screenshot；
- 明确比较差异与优劣，必要时说明“不可推进”；
- 推理写在 `<thoughts>`，最终答案仅输出最佳轨迹编号到 `<answer>`。
```

---

## 24) `CODE_AGENT_PROMPT`（模板存在，当前主流程未直接使用）

- 来源：`muscle_mem/memory/procedural_memory.py` `CODE_AGENT_PROMPT`
- 状态：模板存在（当前 CodeAgent 实际使用 `build_system_prompt`）

原文（核心）：

```text
You are a code execution agent with a limited step budget...
Incremental step-by-step coding, strict output format with <thoughts>/<answer>,
and must return exactly Python/Bash/DONE/FAIL.
```

中文翻译：

```text
你是一个预算受限的代码执行代理，需要分步推进任务。

核心要求：
- 每步写一个自包含的代码片段推进任务；
- 先检查再修改再验证，必要时循环修正；
- 强制响应格式 `<thoughts>` + `<answer>`；
- `<answer>` 只能返回：Python 代码块 / Bash 代码块 / DONE / FAIL；
- 强调数据类型保真、就地修改、结构保真、最终输出修改文件内容、提供 GUI 核验说明。
```

---

## 25) `model_test.py` 的测试提示词（调试脚本）

- 来源：`muscle_mem/utils/model_test.py`
- 状态：调试用，不属于主执行链路

原文：

```text
system_prompt: You are a helpful assistant.
prompt: Reply with 'pong' to confirm you can answer.
```

中文翻译：

```text
系统提示：你是一个乐于助人的助手。
用户提示：请回复 “pong” 以确认你可以正常回答。
```

---

## 26) 覆盖说明（本文件如何定义“所有模型 Prompt”）

本文件覆盖了以下两类：

1. 系统级 Prompt：通过 `LMMAgent(..., system_prompt=...)` 或 `messages_with_system` 提交给模型的系统文本。
2. 固定用户模板 Prompt：在代码中显式拼装、并直接发给模型的固定模板文本（例如 Todo 重试、摘要请求、字段抽取请求）。

不单独列出的内容：

- 纯数据 payload（例如用户任务正文、截图 base64、本次工具结果）
- 非模型提示的日志文本/异常信息

