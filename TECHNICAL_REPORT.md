# HIPPO Agent Technical Report (Code Reverse Engineering)

> 版本：基于当前仓库源码静态分析（`HIPPO-Agent`）  
> 说明：由于尚无官方 paper/technical report，本报告完全依据代码实现整理，不包含未经代码证据支持的推断。

---

## 1. 研究范围与方法

本报告覆盖以下源码区域：

- `muscle_mem/`（核心 Agent 框架、工具、记忆、模型调用）
- `osworld_setup/`（OSWorld 评测入口与运行脚本）
- `README.md` / `README_zh.md`（高层设计声明）

分析方法：

- 以执行链路为主线，从入口脚本到每步动作执行反向还原。
- 对关键模块（Worker、Grounding、Infeasible/Code/Sub Agent、ToolRegistry、Memory）进行逐函数级别梳理。
- 对 README 中的关键 claim（memory-first、guardrail、verification、tool-calling）做代码对应核对。

---

## 2. 代码库整体结构

仓库核心包名是 `muscle_mem`，可视为一个“面向 OSWorld 的 GUI 自动化 Agent 系统”：

- `muscle_mem/agents/`
  - `agent.py`：顶层 `AgentMm`（主状态机：不可行性检查阶段 + 执行阶段）
  - `worker.py`：主执行器（Anthropic tool-use 循环）
  - `grounding.py`：OSWorld ACI（坐标 grounding、工具注册、共享状态）
  - `motor_code_agent.py`：Code Agent（代码执行子代理）
  - `infeasible_agent.py`：可行性判断代理
  - `subagent.py`：子代理框架（Pac-Agent）
  - `verification_agent.py`：验证代理（当前集成层面停用）
  - `tool_loop.py`：统一处理工具响应块
- `muscle_mem/agents/tools/`
  - `registry.py`：工具注册/Schema 构造/dispatch
  - `ui_actions.py`：GUI 操作工具（click/type/scroll/hotkey...）
  - `exec_tools.py`：bash、web_search、web_fetch、scholarly 查询等
  - `todo.py`：TodoWrite 工作流工具
  - `scratchpad.py`：任务 scratchpad 工具
- `muscle_mem/core/`
  - `mllm.py`：`LMMAgent` 统一消息层
  - `engine.py`：多后端 LLM 引擎适配
- `muscle_mem/memory/procedural_memory.py`
  - 核心系统提示词模板（Worker/Code/Verification/Infeasible/Summary）
- `osworld_setup/`
  - `run_muscle_mem_agent.py`：多进程评测入口
  - `run_muscle_mem_agent_local.py`：单进程入口
  - `lib_run_single.py`：单样本 roll-out 逻辑

---

## 3. 总体技术框架（从代码重建）

### 3.1 双阶段主流程（AgentMm）

`muscle_mem/agents/agent.py` 中 `AgentMm` 使用 `_phase` 管理流程：

1. `infeasible` 阶段：调用 `InfeasibleAgentManager.generate_next_action()`  
2. `execute` 阶段：调用 `Worker.generate_next_action()`  
3. `done` 阶段：直接返回 `FAIL`（当 infeasible 结论为不可行时）

关键点：

- 不是单一 monolithic agent，而是“可行性前置 + 执行器”组合。
- infeasible 阶段输出若为 `FEASIBLE`，会缓存简报（`pending_feasible_report`）；但当前注入主执行提示的逻辑被显式禁用（见第 10 节）。

### 3.2 核心控制面：Tool Calling

Worker 基于 Anthropic 的 tool-use 协议做循环调用：

- 模型返回 `tool_use` block。
- Agent dispatch 到 `ToolRegistry`。
- 工具结果再作为 `tool_result` 回灌给模型。

这是一个“模型决策 - 工具执行 - 结果反馈”的闭环，而不是纯文本规划后 `eval`。

### 3.3 Grounding 与执行解耦

`OSWorldACI` 负责：

- 把自然语言元素描述映射为屏幕坐标（视觉 grounding 模型）
- 提供 UI action 生成（pyautogui 脚本字符串）
- 维护任务共享状态（screenshot、todo、scratchpad、execution_history）
- 统一管理工具注册和调用

Worker 负责“决策与调度”，ACI 负责“动作落地”。

---

## 4. 端到端执行链路（OSWorld）

### 4.1 评测入口

`osworld_setup/run_muscle_mem_agent.py`（多进程版本）：

- 构建 `engine_params`（主模型）
- 构建 `engine_params_for_grounding`（坐标模型）
- 可选独立 image grounding 模型（`click_image`）
- 每个进程创建：
  - `DesktopEnv`
  - `OSWorldACI`
  - `AgentMm`
- 逐任务调用 `lib_run_single.run_single_example(...)`

### 4.2 单样本运行 (`lib_run_single.py`)

每个 episode：

1. `agent.reset()`
2. `env.reset(task_config=example)`
3. 固定等待 60 秒（环境稳定）
4. 每步：
   - `response, actions = agent.predict(instruction, obs)`
   - 对每个 action 调 `env.step(action, sleep_after_execution)`
   - 写截图、轨迹 `traj.jsonl`
5. `env.evaluate()` 计算分数

### 4.3 Agent 预测路径

`AgentMm.predict()`：

- 先走 infeasible 阶段，若 `INFEASIBLE`，输出 `FAIL`。
- 若可行，进入 Worker。
- Worker 输出 `exec_code`（字符串），外层当作 OSWorld 动作执行。

---

## 5. Worker（主执行器）实现细节

文件：`muscle_mem/agents/worker.py`

### 5.1 模型约束

- 强制使用 Anthropic tool-use 分支：
  - 若 `model` 形如 `anthropic/...`，自动标准化
  - 非 anthropic 引擎直接抛错

### 5.2 系统提示构建

- 使用 `PROCEDURAL_MEMORY.construct_simple_worker_procedural_memory(...)`
- 动态替换：
  - `TASK_DESCRIPTION`
  - `CURRENT_OS`
  - sudo 密码占位

### 5.3 每步执行循环（核心）

每步大致流程：

1. 绑定当前截图给 grounding：`assign_screenshot(obs)`
2. 首步注入任务到 system prompt
3. 向模型提交当前 observation（截图 + 文本）
4. 期望模型返回 `tool_use`
5. dispatch 工具并把 `tool_result` 回灌
6. 根据工具类型决定继续循环或结束当步

### 5.4 Guardrail 1：TodoWrite 多候选重采样

当首次 `TodoWrite` 项目数 `< 4`：

- 触发至少 3 次、最多 9 次重采样
- 之后再用一次模型做“候选选择”（要求输出 `{"choice": n}`）

目的：减少“计划太短导致跑偏”。

### 5.5 Guardrail 2：早期 `done` 拦截

若模型过早调用 `done` 且历史中未出现关键操作（`call_code_agent/click/click_image/hotkey`）：

- 返回纠正性 `tool_result`（“你的任务不是提供建议...”）
- 最多重试 `max_initial_done_retries`（默认 3）

目的：防止“未实操即完成”。

### 5.6 历史与上下文管理

- `worker_history`：记录计划与工具调用摘要
- `execution_history`（写入 grounding）：按 step 存 plan/tool/error
- `flush_messages()`：保留文本历史，仅裁剪旧图（最多 `max_trajectory_length` 张）

---

## 6. Grounding 子系统

文件：`muscle_mem/agents/grounding.py`

### 6.1 双 grounding 模型

- `grounding_model`：常规元素点击
- `image_grounding_model`：`click_image` 专用（可选独立模型）

两者都通过 `LMMAgent` 调用多模态接口，输入截图并输出坐标。

### 6.2 坐标生成

`generate_coords(ref_expr, obs, use_image_model=False)`：

- prompt：要求“只输出一个坐标点”
- 首选解析 `<point>x y</point>`
- 兜底解析首两个数字
- 最终用 `resize_coordinates` 映射回 OSWorld 分辨率

### 6.3 文本跨度定位（OCR）

`generate_text_coords(...)`：

- 用 `pytesseract.image_to_data` 提词级 bbox
- 构造 `word id -> text` 表
- 调 `text_span_agent` 预测目标 word id
- 返回中点/首点/末点坐标

用于 `highlight_text_span` 等精细文本操作。

### 6.4 共享状态（“工作记忆”载体）

ACI 保存大量任务状态：

- `scratchpad`
- `execution_history`
- `initial_screenshot` / `second_screenshot`
- `pending_feasible_report`
- `last_todo_board_view` / `last_todo_summary`
- 各子代理最近结果（`last_code_agent_result` 等）

---

## 7. 工具系统设计

### 7.1 ToolRegistry（反射式注册）

文件：`muscle_mem/agents/tools/registry.py`

- `@tool_action` 标记可暴露方法
- 自动从函数签名推导 JSON Schema
- 支持 `tool_input_schema` 手工覆盖
- `build_tools(allow, deny)` 构造提交给 Anthropic 的 tool 列表
- `dispatch(name, tool_input)` 执行调用

这是整套系统的“统一控制面”。

### 7.2 UI 操作工具 (`ui_actions.py`)

核心思想：工具不直接执行，而是返回 pyautogui 代码字符串，再交给环境执行。

主要工具：

- `click` / `click_image`
- `type`（支持 unicode/剪贴板，Linux 下会尝试安装 `xclip/xsel`）
- `drag_and_drop`
- `scroll` / `hotkey` / `hold_and_press`
- `switch_applications` / `open`
- `done` / `fail` / `report_infeasible`
- `set_cell_values`（UNO 自动写表）

### 7.3 执行工具 (`exec_tools.py`)

主要工具：

- `bash(command, timeout_sec, description)`
- `web_search`（Tavily API）
- `web_fetch`（Jina Reader + 可选字段抽取）
- `scholarly_author` / `scholarly_publication`

实现特征：

- 输出统一做长度截断保护
- 输入输出调试日志（base64 脱敏）
- `web_fetch` 字段抽取会二次调用 LLM 解析 JSON

### 7.4 Todo / Scratchpad

`todo.py`：

- 强约束 Todo schema（`content/activeForm/status`）
- 限制最多 20 项
- 同时最多一个 `in_progress`
- 渲染可配置（主 Agent 与 Code Agent 使用不同风格）

`scratchpad.py`：

- 保存结构化字符串列表
- 支持读取最近 N 项
- 输出中可附带当前任务描述

---

## 8. Code Agent 子系统（代码执行代理）

文件：`muscle_mem/agents/motor_code_agent.py`

### 8.1 设计目标

把“适合代码完成”的任务从 GUI 主代理中剥离，形成独立工具循环。

### 8.2 核心执行循环

`query(...)`：

- 每轮提交 `messages + system_prompt + tools`
- 解析 `tool_use`
- 执行工具后追加 `tool_result`
- 若调用 `Done`，结束
- 否则继续，直到 budget 耗尽

### 8.3 Code Agent 工具集

- `ExecutionToolProvider`
- `TodoToolProvider`
- `ScratchpadToolProvider`
- `DoneToolProvider`

并显式 deny `web_fetch`（`CODE_AGENT_TOOL_DENY = ["web_fetch"]`）。

### 8.4 输出与摘要

执行完成后会返回：

- `completion_reason`（DONE/FAIL/UNKNOWN）
- `execution_history`
- `summary`（由 summary prompt 生成）

---

## 9. Infeasible Agent（可行性代理）

文件：`muscle_mem/agents/infeasible_agent.py`

### 9.1 作用定位

在主执行前进行“可行性前置判断”，并可输出：

- `report_feasible(reason, evidence)`
- `report_infeasible(reason, evidence)`

### 9.2 两种运行模式

- `generate_next_action(...)`：逐步状态机（被 `AgentMm` 使用）
- `run_task(...)`：独立完整执行（供工具调用）

### 9.3 只读约束强化

当 infeasible 代理调用 `call_code_agent` 时，会把任务改写成只读核验任务：

- “只读核验任务可行性，禁止修改任何文件或系统状态”

这体现了“验证与执行能力隔离”的设计。

---

## 10. Sub Agent 与 Verification Agent 的当前状态

### 10.1 SubAgent（部分存在但当前未真正暴露）

`subagent.py` 里定义了 Pac-Agent 与完整执行逻辑，但关键方法 `call_subagent` 的 `@tool_action` 被注释掉，意味着默认不会注册为可调用工具。

### 10.2 Verification Agent（实现存在、主流程停用）

`verification_agent.py` 实现完整，但在 `grounding.py` 中其集成代码被注释，且工具入口 `call_verification_agent` 的 `@tool_action` 也被注释。

### 10.3 Feasible report 注入被禁用

代码中多处明确注释：

- feasible report 注入主 Agent/Code Agent 被禁用（仅标记已使用）

因此当前版本更像是“可行性前置判断 + 主执行器”，而不是“执行后再自动验证”的闭环。

---

## 11. LLM 抽象层与多后端适配

### 11.1 `LMMAgent`（统一消息层）

文件：`muscle_mem/core/mllm.py`

职责：

- 统一 system/user/assistant 消息格式
- 按引擎类型封装图像 block（OpenAI 风格、Anthropic 风格、vLLM 风格）
- `get_response()` 前做消息清洗（空文本块删除）

### 11.2 `engine.py`（多引擎实现）

支持：

- OpenAI
- Anthropic / AnthropicLR
- Azure OpenAI
- Gemini（OpenAI 兼容端点）
- OpenRouter
- vLLM
- HuggingFace TGI
- Parasail

共同特征：

- `backoff` 指数重试（连接/限流/服务错误）

Anthropic 特征：

- 可选 prompt caching（`cache_control=ephemeral` + beta header）
- tool_use 返回归一化为统一 dict 结构
- thinking 模式可保留思考内容

---

## 12. “Memory-first” 在代码中的真实落地

README 强调 memory-first。代码层面对应为“任务内工作记忆”，不是外部向量库长期记忆：

### 12.1 显式状态记忆

- `scratchpad`：可由模型主动写入/读取
- `todo_board`：结构化任务分解与进度状态
- `execution_history`：执行轨迹（step/tool/input/error）
- `last_*_result`：各子代理最近结果缓存

### 12.2 上下文记忆策略

- 保留文本历史
- 仅保留最近 K 张图（`flush_messages`）
- 通过工具结果回灌维持短期闭环

### 12.3 Prompt-level procedural memory

`procedural_memory.py` 提供大量行为策略模板（Worker/Code/Infeasible/Verification/Summary），在实践中扮演“可执行经验记忆”的角色。

---

## 13. 稳定性与 pass@1 导向机制

从代码可确认的稳定性机制：

1. 可行性前置判断（Infeasible phase）
2. 早期 Done 拦截（必须先有关键执行证据）
3. Todo 低质量计划重采样 + 候选选择
4. LLM 调用重试（最多 10 次）
5. 工具输入/输出结构化与 schema 约束
6. 消息清洗（空文本过滤）降低 API 格式错误
7. 进程级评测容错（worker 死亡自动拉起）

---

## 14. 与 README 声明的对应关系

### 已在代码中明显体现

- Tool-calling first：是主执行范式
- Feasibility validation：存在并前置
- Memory working set：todo/scratchpad/history 有落地
- Guardrail：done/todo 等规则化控制明确

### 当前版本与声明存在差距/变体

- Verification agent 已实现但默认停用
- feasible report 注入逻辑默认停用
- reflection 开关参数存在，但 Worker 内部将 `enable_reflection` 固定为 `False`
- SubAgent 能力框架存在，但默认未暴露为工具

---

## 15. 关键实现风险与潜在缺陷（代码事实）

以下是从源码直接观察到的风险点：

1. **`Worker` 仅支持 Anthropic tool-use**
   - 非 anthropic engine 直接报错，主代理后端可移植性受限。

2. **若干功能“存在代码但未接入”**
   - Verification/SubAgent 的关键工具入口注释掉，可能导致文档预期与运行行为不一致。

3. **命名遗留与分支残留**
   - `call_pac_agent`、`click_image_area` 在多个地方被引用，但对应工具名与现有实现存在不一致风险。

4. **`exec_tools.py` 潜在字段错误**
   - `scholarly_publication` 中存在对 `result["publication_title_html"]` 的访问，但该 key 未见稳定赋值路径，可能触发异常。

5. **`subagent.py` 中 `gui_batch_automation` 日志变量**
   - 使用未定义变量 `subagent`（该路径当前默认不暴露，若启用会成为运行风险）。

6. **`set_cell_values` sudo 密码模板**
   - UNO 脚本中包含硬编码 `"password"`，与动态密码策略不完全一致。

---

## 16. 可复现实验与运行配置特征

从 `osworld_setup` 可总结出该项目常见实验形态：

- Ubuntu OSWorld 环境
- `pyautogui` action space
- 主模型 + grounding 模型分离
- 可选 image-grounding 模型（处理 `click_image`）
- 支持多进程并发评测（`--num_envs`）
- 结果目录结构标准化：
  - `result_dir/action_space/observation_type/model/domain/example_id/`
  - 包含 `traj.jsonl`、逐步截图、`result.txt`

---

## 17. 技术结论

从实现上看，HIPPO Agent 当前版本的核心创新点并不在复杂层级规划，而在“**流程工程化与可控性增强**”：

- 通过 **前置可行性检查 + 工具调用闭环**，把不确定任务拆为“能不能做”和“怎么做”两层。
- 通过 **Todo 重采样、Done 拦截、消息裁剪、重试机制**，针对 pass@1 的失败模式做了较强工程修补。
- 通过 **Code Agent/GUI Agent 分工**，将“代码高效路径”显式化，减少纯 GUI 路径的低效探索。

同时，当前分支处于“功能收敛”状态：验证代理、反思代理、子代理虽有实现，但默认运行路径更偏“务实最小闭环”。这与 README 的宏观叙事基本一致，但具体能力边界应以代码启用状态为准。

---

## 18. 附：核心模块速查表

- 顶层入口：`osworld_setup/run_muscle_mem_agent.py`
- 单样本执行：`osworld_setup/lib_run_single.py`
- 主状态机：`muscle_mem/agents/agent.py`
- 主执行器：`muscle_mem/agents/worker.py`
- Grounding/工具总线：`muscle_mem/agents/grounding.py`
- Code Agent：`muscle_mem/agents/motor_code_agent.py`
- Infeasible Agent：`muscle_mem/agents/infeasible_agent.py`
- Verification Agent（实现在、集成停用）：`muscle_mem/agents/verification_agent.py`
- 工具注册：`muscle_mem/agents/tools/registry.py`
- UI 工具：`muscle_mem/agents/tools/ui_actions.py`
- 执行工具：`muscle_mem/agents/tools/exec_tools.py`
- Todo 工具：`muscle_mem/agents/tools/todo.py`
- Scratchpad 工具：`muscle_mem/agents/tools/scratchpad.py`
- LLM 抽象：`muscle_mem/core/mllm.py`
- 多后端引擎：`muscle_mem/core/engine.py`
- Prompt 模板：`muscle_mem/memory/procedural_memory.py`

