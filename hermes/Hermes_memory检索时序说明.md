# Hermes memory 检索时序说明

结论：Hermes 在每一轮对话开始前，会先走 memory 的 prefetch 流程，再把结果临时注入到本轮的 user message 里，然后才发起模型调用。

## 实际顺序

1. 用户消息进入 `run_conversation`
2. 当前轮 user message 被加入 `messages`
3. 触发 `on_turn_start`
4. 调用 `prefetch_all(current_user_query)` 做 recall
5. 把返回的 memory context 包成 `<memory-context>` 注入到本轮 user message
6. 再发起第一次 LLM 请求

## 结束后一轮会发生什么

上一轮结束后，会异步执行：

- `sync_all(...)`：把本轮对话写入外挂 memory
- `queue_prefetch_all(...)`：为下一轮预热 recall

所以下一轮开始时，Hermes 仍然会把“新的用户问题”传给 memory 入口，但具体是否立刻用这个新问题做同步检索，取决于外挂 provider 的实现。

## 代码层面的关键点

- `agent/turn_context.py`：在 turn prologue 里先调用 `prefetch_all(...)`
- `agent/conversation_loop.py`：把 prefetch 返回的内容注入当前 user message
- `run_agent.py`：在 turn 结束后调用 `sync_all(...)` 和 `queue_prefetch_all(...)`

## 需要注意

有些 memory provider 更偏向异步预热型：

- `prefetch()` 负责快速返回已预热结果
- `queue_prefetch()` 负责后台查询下一轮要用的内容

所以“每轮都会检索 memory”是对的，但“这一轮一定会同步查到新问题对应的结果并马上注入”不一定成立。
