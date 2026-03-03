# HIPPO-Agent 可调用 Tool 说明

本文基于代码静态阅读整理（截至当前仓库版本），目标是回答两件事：

1. 项目里“定义了哪些可调用 tool”（`@tool_action`）。
2. 不同 agent 在默认配置下“实际上能调用哪些 tool”。

---

## 1. 工具系统机制

- 只有带 `@tool_action` 的方法会被注册为 tool。
- 注册入口在 `ToolRegistry.register_action_provider(...)`，它会扫描 provider 类里带 `is_tool_action` 标记的方法。
- 调用入口是 `ToolRegistry.dispatch(name, tool_input)`；未注册名称会直接报 `Unknown tool`。

关键代码：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/tools/registry.py`

---

## 2. 全部已定义（带 `@tool_action`）的 Tool

以下是“代码层面已定义可调用”的全集（不代表在每个 agent 中都可见）。

按名称汇总（去重后）：

- `click`
- `click_image`
- `switch_applications`
- `open`
- `type`
- `drag_and_drop`
- `highlight_text_span`
- `set_cell_values`
- `scroll`
- `hotkey`
- `hold_and_press`
- `wait`
- `done`
- `fail`
- `report_infeasible`
- `bash`
- `web_search`
- `scholarly_author`
- `scholarly_publication`
- `web_fetch`
- `TodoWrite`
- `save_scratchpad`
- `read_scratchpad`
- `Done`
- `call_code_agent`
- `report_feasible`
- `call_infeasible_agent`
- `report_verification_plan`
- `report_verification_result`

## 2.1 UIActions（GUI 操作）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/tools/ui_actions.py`

工具列表：

1. `click(element_description, num_clicks=1, button_type="left", hold_keys=[])`
- 含义：按文本描述定位坐标并点击。
- 坐标来源：`grounding_model`。

2. `click_image(element_description, num_clicks=1, button_type="left", hold_keys=[])`
- 含义：按“图片/图像元素”描述定位并点击。
- 坐标来源：`image_grounding_model`。

3. `switch_applications(app_code)`
- 含义：切换到已打开应用（按不同 OS 生成不同脚本）。

4. `open(app_or_filename)`
- 含义：打开应用或文件。

5. `type(element_description=None, text="", overwrite=False, enter=False)`
- 含义：定位后输入文本；支持覆盖输入、回车提交。
- 细节：遇到 Unicode 或特殊字符时会优先走剪贴板粘贴路径。

6. `drag_and_drop(starting_description, ending_description, hold_keys=[])`
- 含义：从起点拖拽到终点。

7. `highlight_text_span(starting_phrase, ending_phrase, button="left")`
- 含义：按起止文本短语高亮文本区间。

8. `set_cell_values(cell_values, app_name, sheet_name)`
- 含义：面向电子表格，按坐标批量写单元格。

9. `scroll(element_description, clicks, shift=False)`
- 含义：在目标元素处滚动（垂直/水平）。

10. `hotkey(keys)`
- 含义：组合键。

11. `hold_and_press(hold_keys, press_keys)`
- 含义：按住一组键，同时按序按下另一组键。

12. `wait(time)`
- 含义：等待指定秒数。

13. `done()`
- 含义：返回 `"DONE"`，表示成功结束。

14. `fail()`
- 含义：返回 `"FAIL"`，表示失败结束。

15. `report_infeasible(reason, evidence)`
- 含义：上报“任务不可行”原因和证据，并返回 `"FAIL"`。
- 注意：这是 UIActions 版本的 `report_infeasible`，行为与 InfeasibleResultToolProvider 的同名工具不同（后者返回结构化 payload）。

## 2.2 ExecutionToolProvider（执行/检索工具）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/tools/exec_tools.py`

工具列表：

1. `bash(command, timeout_sec, description=None)`
- 含义：在环境控制器中执行一次 bash。
- 限制：`timeout_sec` 最终会被限制在 `[30, 300]`。
- 依赖：需要 `env_controller`。

2. `web_search(query, search_depth="basic", max_results=5, include_images=False, include_answer=True)`
- 含义：调用 Tavily 搜索。
- 依赖：`TAVILY_API_KEY`。

3. `scholarly_author(author_id, fill_author=True, max_publications=5, fill_publications=False)`
- 含义：查询 Google Scholar 作者与论文概览。
- 依赖：`scholarly` Python 库。

4. `scholarly_publication(author_id, publication_id, fill_publication=True)`
- 含义：查询 Scholar 具体论文信息（含页面抓取辅助字段）。
- 依赖：`scholarly`、`JINA_API_KEY`（抓取路径相关）。

5. `web_fetch(url, fields=None)`
- 含义：抓取网页 Markdown；可选字段抽取。
- 依赖：`JINA_API_KEY`；如启用字段抽取还需要有效 `engine_params` 用于二次 LLM 解析。

## 2.3 Todo / Scratchpad

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/tools/todo.py`
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/tools/scratchpad.py`

工具列表：

1. `TodoWrite(items)`
- 含义：维护结构化待办板（`pending/in_progress/completed`）。
- 约束：唯一 `id`、最多 20 条、同一时刻最多一个 `in_progress` 等。

2. `save_scratchpad(text)`
- 含义：保存非结构化笔记列表（最多 100 条）。

3. `read_scratchpad(limit=None)`
- 含义：读取 scratchpad，可限制返回最近 N 条。

## 2.4 跨 Agent 调度工具

### Code Agent 调度

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/motor_code_agent.py`

1. `call_code_agent(task=None, max_rounds=None)`
- 含义：把任务交给 Code Agent。
- 默认行为：`task` 为空时使用原始完整任务。

2. `Done(reason=None)`（仅 Code Agent 内部）
- 含义：Code Agent 用于结束自身回合（注意首字母大写）。

### Infeasible Agent 调度/结果上报

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/infeasible_agent.py`

1. `call_infeasible_agent()`
- 含义：调用 Infeasible Agent 判定可行性。

2. `report_infeasible(reason, evidence)`（InfeasibleResultToolProvider 版本）
- 含义：上报不可行结论，返回结构化 payload。

3. `report_feasible(reason, evidence)`
- 含义：上报可行结论，返回结构化 payload。

### Verification Agent 结果工具（模块内可调用）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/verification_agent.py`

1. `report_verification_plan(task_understanding, possible_failures, screenshot_observation, verification_plan)`
- 含义：记录核验计划。

2. `report_verification_result(conclusion, explanation)`
- 含义：记录最终核验结论（`IMPOSSIBLE | ERROR | SUCCESS`）。

---

## 3. 各 Agent 默认“实际可见”工具集合

这部分是“运行时默认配置”下真正会传给模型的 tool 集合，不等于全集。

## 3.1 主 Worker（执行阶段）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/grounding.py`
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/worker.py`

Worker 使用 `grounding.get_anthropic_tools()`，该接口默认 `deny`：
- `web_search`
- `python`
- `web_fetch`
- `bash`
- `call_infeasible_agent`
- `scholarly_publication`
- `scholarly_author`

因此主 Worker 默认可见：
- UIActions 大部分工具（`click/click_image/open/type/.../done/fail/report_infeasible`）
- `TodoWrite`
- `save_scratchpad` / `read_scratchpad`
- `call_code_agent`

另外，Worker 在 `reset()` 里还会做动态过滤：
- 非 Linux 时跳过 `set_cell_values`
- 无 `env/controller` 时跳过 `call_code_agent`（以及 `call_subagent`，但后者本来未暴露）

## 3.2 Infeasible Agent（可行性判定阶段）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/infeasible_agent.py`

默认 allow-list（`DEFAULT_INFEASIBLE_TOOL_ALLOW`）：
- `web_search`
- `web_fetch`
- `call_code_agent`
- `click`
- `click_image`
- `switch_applications`
- `open`
- `scroll`
- `wait`
- `hold_and_press`
- `report_infeasible`
- `report_feasible`

注意：
- 它默认不暴露 `TodoWrite`、`save_scratchpad`、`read_scratchpad`。
- 其 `report_infeasible` 来自 `InfeasibleResultToolProvider`（结构化结论上报语义）。

## 3.3 Code Agent（代码执行阶段）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/motor_code_agent.py`

Code Agent 注册：
- `ExecutionToolProvider`
- `TodoToolProvider`
- `ScratchpadToolProvider`
- `DoneToolProvider`

默认仅 deny `web_fetch`，因此常见可见工具：
- `bash`
- `web_search`
- `scholarly_author`
- `scholarly_publication`
- `TodoWrite`
- `save_scratchpad`
- `read_scratchpad`
- `Done`

## 3.4 Verification Agent（当前主流程未挂载）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/verification_agent.py`
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/grounding.py`

Verification 管理器内部默认 allow-list（`DEFAULT_VERIFICATION_TOOL_ALLOW`）：
- `web_search`
- `web_fetch`
- `call_code_agent`
- `click`
- `switch_applications`
- `open`
- `scroll`
- `wait`
- `hold_and_press`
- `report_verification_plan`
- `report_verification_result`

但在当前 `grounding.py` 中，Verification Agent 的集成注册被注释，默认主流程不会调用它。

## 3.5 Sub-Agent（Pac-Agent，当前入口未暴露）

来源：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/subagent.py`

`SubAgentToolProvider.call_subagent` 的 `@tool_action` 被注释，默认不暴露。
即使 `grounding` 注册了 `SubAgentToolProvider`，也不会产生可调用 tool。

补充：`SUB_AGENT_TYPES` 的 allow-list 里存在 `click_image_area`，但 UIActions 实际工具名是 `click_image`。如果未来重新暴露 subagent 入口，这个命名不一致会导致该项无法命中。

---

## 4. 当前“写了但默认不可调用”的工具/入口

1. `python(code)`（ExecutionToolProvider）
- `@tool_action` 被注释。

2. `call_subagent(...)` / `gui_batch_automation(...)`（SubAgentToolProvider）
- `@tool_action` 被注释。

3. `call_verification_agent(...)`（VerificationAgentToolProvider）
- `@tool_action` 被注释；且在 `grounding.py` 中整体集成也被注释。

---

## 5. 一个容易踩坑的命名差异

1. `done`（小写）
- 来自 `UIActions`，用于主 Worker/UI 动作流结束。

2. `Done`（大写）
- 来自 Code Agent 的 `DoneToolProvider`，仅在 Code Agent 内部工具集出现。

3. `report_infeasible` 同名不同语义
- `UIActions.report_infeasible`: 主要用于执行流中标记失败并返回 `"FAIL"`。
- `InfeasibleResultToolProvider.report_infeasible`: 用于 Infeasible Agent 的结构化结论上报。

---

## 6. 主流程简图（按默认配置）

1. 任务进入 `AgentMm`，先跑 Infeasible 阶段。
2. Infeasible 阶段若判定可行，进入 Worker 执行阶段。
3. Worker 可调用 UI/Todo/Scratchpad/Code-Agent（但默认禁用 web/bash/scholarly/infeasible-call）。
4. 如 Worker 调 `call_code_agent`，进入 Code Agent 的独立工具循环（可用 bash/web_search/todo/scratchpad/Done）。

关键入口：
- `/Users/zhangxiuhui/Desktop/project/HIPPO-Agent/muscle_mem/agents/agent.py`
