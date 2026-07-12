# Loop 工程与 Codex 项目实现调研

日期：2026-07-12

## Loop engineer 的大致原理

公开资料里更常见的叫法是 agent loop / loop engineering，而不是一个非常固定的专有名词。它的核心模式是：

1. agent 持有一个目标或任务状态。
2. 每一轮根据当前证据做 reasoning / plan。
3. 调工具或操作环境，例如读代码、改文件、跑测试。
4. 观察结果，更新状态。
5. 如果目标未完成且未被阻塞或预算耗尽，就自动进入下一轮。

这个模式和 ReAct 的 reasoning + acting + observation 很接近；SWE-agent 则是把这个 loop 放到软件工程场景里，通过专门的 agent-computer interface 让模型能浏览仓库、编辑代码、执行测试。

## 本项目里有没有类似功能

结论：有类似功能，但项目里不叫 loop engineer，主要分成两块。

### 1. Automations 工具桥接

这部分不是完整排程系统，而是让宿主应用把 automation 作为 dynamic tool 暴露给模型。真正的 automation 创建、更新、排程、持久化逻辑在宿主侧，不在这个 Rust 仓库里。

相关代码：

- `codex-rs/core/src/tools/handlers/dynamic.rs`：core 里的 dynamic tool handler，负责把模型的 `automation_update` 这类调用转成 `DynamicToolCallRequest`，等待 app-server 客户端回 `DynamicToolResponse`。
- `codex-rs/app-server/src/request_processors/thread_processor.rs`：app-server 启线程时校验并注册 dynamic tools。
- `codex-rs/protocol/src/dynamic_tools.rs`：协议结构体，定义 `DynamicToolSpec`、`DynamicToolCallRequest`、`DynamicToolResponse`。
- `codex-rs/app-server/src/dynamic_tools.rs`：app-server 收到客户端工具回包后，提交回 core。
- `codex-rs/core/tests/suite/search_tool.rs`：覆盖 deferred dynamic tool 通过 `tool_search` 被发现并调用。

### 2. Goals / thread goals：真正的 goal loop

这部分更接近 loop engineer：保存线程级目标，线程 idle 时自动继续下一轮，直到目标状态变成 complete / blocked / paused / usage_limited / budget_limited。

关键代码：

- `codex-rs/ext/goal/src/extension.rs`：在线程 start / resume / idle / stop 生命周期挂 goal runtime。`on_thread_idle` 会触发 `runtime.continue_if_idle()`。
- `codex-rs/ext/goal/src/runtime.rs`：核心 runtime。`continue_if_idle()` 会读取 active goal，并在 idle 时通过 `try_start_turn_if_idle` 自动启动下一轮。
- `codex-rs/ext/goal/src/tool.rs`：模型可用的 `get_goal`、`create_goal`、`update_goal` 工具。
- `codex-rs/ext/goal/src/spec.rs`：goal 工具的 Responses API tool 定义。
- `codex-rs/ext/goal/templates/goals/continuation.md`：每轮自动继续时注入给模型的指令。
- `codex-rs/app-server/src/request_processors/thread_goal_processor.rs`：app-server 的 `thread/goal/set`、`thread/goal/get`、`thread/goal/clear` 请求处理。
- `sdk/python/src/openai_codex/client.py` 和 `sdk/python/src/openai_codex/_goal.py`：Python SDK 把多个物理 turn 聚合成一个逻辑 goal operation stream。

## 实现机制摘要

Goals 功能的执行链路大致是：

1. 用户或宿主通过 `thread/goal/set` 设置一个 active goal。
2. `GoalService` 持久化 goal，并把 runtime 状态设置为 active。
3. 当前线程 idle 时，`GoalExtension::on_thread_idle` 触发 `GoalRuntimeHandle::continue_if_idle()`。
4. runtime 读取 active goal，生成 continuation steering item。
5. 线程通过 `try_start_turn_if_idle(vec![item])` 自动开始下一轮。
6. 模型继续工作，必要时使用 `update_goal` 标记 complete 或 blocked。
7. 系统负责 usage limit、budget limit、pause/resume 等状态，不允许模型直接设置这些状态。

## 边界和安全点

- `update_goal` 只允许模型设置 `complete` 或 `blocked`。
- `paused`、`usage_limited`、`budget_limited` 由用户或系统控制。
- goal continuation prompt 明确要求：不要缩小目标，不要把部分进展当完成，完成前要做证据审计。
- goal 有 token budget 统计，预算耗尽时会进入 budget-limited 状态。

## 和 automations 的区别

- automations：外部产品能力，通过 dynamic tool 暴露给模型；Rust 侧只是桥接和上下文注入。
- goals：仓库内实现的持续推进 loop；Rust 侧有完整 runtime、状态持久化、生命周期挂钩、工具、SDK 路由。

因此，如果说 “loop engineer = 自动持续推进一个软件工程目标的 agent loop”，本项目里对应的是 Goals / thread goals，而不是 automations。
