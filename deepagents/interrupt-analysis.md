# DeepAgents interrupt 说明

不是主要用 `Input`。

实现分两段：

1. **LangGraph 产生 HITL interrupt**
   在 `agent.py` 里，如果没有 `auto_approve`，会把 `interrupt_on` 配成 `_add_interrupt_on()`。这个配置会让 `execute`、`write_file`、`edit_file`、`web_search` 等工具调用前触发人工审批。

2. **Textual UI 展示审批菜单**
   adapter 在 `textual_adapter.py` 监听 stream 里的 `__interrupt__`，收集到 `pending_interrupts` 后，在 `app.py` 调用 `_request_approval(...)`。

`_request_approval` 会创建并挂载 `ApprovalMenu`。它是 Textual 的 `Container`，不是 `Input`。它靠 `Binding` 处理快捷键：

- `y` / `1`：approve
- `a` / `2`：auto-approve
- `n` / `3`：reject
- `Enter`：确认当前选项

用户选择后，菜单会把结果写进一个 `Future`，再由 `textual_adapter.py` 组装成 `resume_payload` 继续执行。

`Input` 只在一个地方用：拒绝时填写原因。那个输入框默认隐藏，只有选中 Reject 后按 `Tab` 才出现。正常的 approve/reject 不是靠 `Input`，而是靠 `ApprovalMenu(Container)` + 快捷键绑定。
