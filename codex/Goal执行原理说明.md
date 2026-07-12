# Goal 执行原理说明

## 1. Goal 是什么

`goal` 不太像 todo list，更像“当前线程的单一长期任务”。

它的特点是：

- 每个 thread 只有一个当前 goal。
- goal 有状态机：`active`、`paused`、`blocked`、`usageLimited`、`budgetLimited`、`complete`。
- goal 会自动续跑：只要状态还是 `active`，线程 idle 后 runtime 会自动启动下一轮 turn。
- goal 会记账：累计 tokens 和 elapsed time。
- 模型只能通过 `update_goal` 把 goal 标记为 `complete` 或 `blocked`。

更准确地说：

```text
goal = 持久化的单一任务目标 + 自动续跑机制 + 预算/用量记账 + 状态通知
```

## 2. 谁可以判断或改变 Goal 状态

有三类角色会判断或改变 goal 状态。

### 模型自己

模型通过 `update_goal` 工具，只能设置：

- `complete`
- `blocked`

也就是说，模型只能判断“目标完成了”或“确实卡住了”。

### 用户 / 外部客户端

用户或外部客户端通过 app-server RPC `thread/goal/set` 可以设置状态，例如：

- `active`
- `paused`
- `blocked`
- `usageLimited`
- `budgetLimited`
- `complete`

SDK 里的 `pause_goal()` 就是用 `thread/goal/set(status=paused)` 实现的。

### 系统 runtime

系统根据执行过程自动设置：

- token 超过预算：`budgetLimited`
- usage limit 错误：`usageLimited`
- 非 usage limit 的 turn 错误：`blocked`

系统也会在每轮 turn/tool 结束时累计 `tokens_used` 和 `time_used_seconds`，并据此判断是否触发预算状态。

## 3. Turn 是什么

一个 `turn` 基本就是“一次模型响应周期”的开始到结束。

例如：

- 用户发一条消息，系统启动一次 turn。
- 模型开始推理、调用工具、接收工具结果、继续推理。
- 最后模型给出回复，或者因为错误/中断/用量限制结束。
- 这个完整过程就是一个 turn。

在 goal 场景里，一个长期 goal 不是一个超长 turn，而是由多个普通 turn 串起来：

```text
goal active
  -> turn 1 执行一部分
  -> thread idle
  -> runtime 发现 goal 还是 active
  -> 自动启动 turn 2
  -> thread idle
  -> runtime 发现 goal 还是 active
  -> 自动启动 turn 3
  -> ...
  -> 模型 update_goal(complete/blocked) 或系统停止
```

所以：

- 普通对话：一次用户输入通常对应一个 turn。
- goal 执行：系统可能在没有新用户输入的情况下，自动继续启动后续 turn。
- 每个 turn 都会单独记 token、耗时、工具调用，然后累计到同一个 goal 上。

## 4. Goal 自动续跑的执行过程

第一轮 turn 结束后，后续大模型的输入不是用户再发的一句话，而是系统自动塞进去的一条内部 `ResponseItem`。

核心代码逻辑类似：

```rust
let item = continuation_steering_item(&protocol_goal_from_state(goal));
thread.try_start_turn_if_idle(vec![item]).await
```

这个 `item` 的文本来自：

```text
codex-rs/ext/goal/templates/goals/continuation.md
```

渲染后大概像这样：

```text
Continue working toward the active thread goal.

<objective>
Improve benchmark coverage
</objective>

Budget:
- Tokens used: 12345
- Token budget: 30000
- Tokens remaining: 17655

Use the current worktree and external state as authoritative...
If the objective is achieved, call update_goal with status "complete"...
Do not call update_goal unless the goal is complete or strictly blocked...
```

所以后续 turn 的输入可以理解为：

```text
历史上下文 + 当前 system/developer 指令 + 自动注入的 goal continuation 上下文
```

其中最关键的是自动注入的 goal 上下文，它告诉模型：

- 原始目标是什么。
- 已用多少 token、预算还剩多少。
- 要继续推进，不要缩小目标。
- 完成时调用 `update_goal(status="complete")`。
- 真正阻塞时才调用 `update_goal(status="blocked")`。

## 5. 自动注入的是 user 指令吗

它不是普通意义上“新增一条用户消息”，但它会以 model-visible input item 的形式进入模型上下文，效果上很像给模型加了一条“继续 goal”的用户侧上下文。

代码里是这样构造的：

```rust
fn goal_context_input_item(prompt: String) -> ResponseItem {
    ContextualUserFragment::into(InternalModelContextFragment::new(
        InternalContextSource::from_static("goal"),
        prompt,
    ))
}
```

也就是说它不是来自真实 user 的聊天输入，而是：

```text
InternalModelContextFragment
source = "goal"
content = continuation.md 渲染后的文本
```

然后启动后续 turn 时：

```rust
thread.try_start_turn_if_idle(vec![item]).await
```

所以可以这样理解：

- 不是用户真的发了一条新消息。
- 也不是 system/developer 指令。
- 而是系统内部生成的一段 goal 上下文，被包装成 `ContextualUserFragment` 注入到模型输入里。

如果按最终模型上下文的“角色”理解，它是 user-context 类输入，不是 system，也不是 developer。它用于表达“用户目标相关上下文”。

## 6. 什么时候判断是否需要注入 continuation

判断是在程序里做的，不是先问模型“要不要继续”。

大致逻辑是：

1. 上一轮 turn 结束。
2. `GoalExtension.on_turn_stop` / `on_turn_abort` 先做记账：
   - 计算本轮新增 tokens。
   - 计算本轮耗时。
   - 更新 `thread_goals.tokens_used`。
   - 如果超预算，可能把状态改成 `budgetLimited`。
3. 线程进入 idle。
4. `GoalExtension.on_thread_idle -> runtime.continue_if_idle()`。
5. 程序从 state db 读取当前 goal 状态。
6. 只有 `goal.status == active` 才继续。
7. 如果还是 active，程序构造 `continuation_steering_item(goal)`，然后调用 `thread.try_start_turn_if_idle(vec![item])`。

简化成：

```text
上一轮 turn 结束
  -> 程序更新/读取 goal 状态
  -> 如果 goal.status == active
  -> 并且线程确实 idle、非 Plan mode、没有用户新 turn 排队
  -> 自动注入 goal continuation 上下文
  -> 启动下一轮 turn
```

核心不是判断“目标有没有更新”，而是判断当前持久化 goal 是否仍然是 `active`。

## 7. 自动启动后续 Turn 的输入和输出

自动 turn 的输入是一个 `ResponseItem`，来源是 `continuation_steering_item(goal)`。

它不是用户可见的新消息，也不是普通聊天输入，而是 `InternalModelContextFragment`，source 是 `"goal"`。

自动 turn 的输出和普通 turn 一样：

- 模型消息。
- 工具调用。
- 工具结果。
- 最终 assistant 回复。
- turn completed / error / interrupted 等事件。

但在 goal 场景里，输出还会额外影响 goal 状态：

- 如果模型调用 `update_goal({"status":"complete"})`，goal 结束。
- 如果模型调用 `update_goal({"status":"blocked"})`，goal 停止在 blocked。
- 如果 turn 结束但 goal 还是 active，下一次 idle 会再自动启动一个 turn。
- 如果 token budget 到了，系统把状态设为 `budgetLimited`。
- 如果 usage limit 到了，系统设为 `usageLimited`。
- 如果普通错误导致 turn 失败，系统设为 `blocked`。

自动启动也有保护条件。`try_start_turn_if_idle` 只会在这些条件满足时启动：

- 当前没有 active turn。
- 没有用户/client 触发的新 turn 正在排队。
- 当前不是 Plan mode。
- 没有其他任务占用线程。

## 8. 如何测试

测试主要有三类。

### 自动 turn 启动门禁测试

`try_start_turn_if_idle` 的测试覆盖：

- 已有 active turn 时拒绝。
- Plan mode 时拒绝。
- 有用户/client pending turn 时拒绝。
- review turn 占用时拒绝。

相关位置：

```text
codex-rs/core/src/session/tests.rs
```

### App-server / Rust 集成测试

测试会 mock 一个模型 response，比如 `goal-continuation`，验证设置 active goal 后会触发后续 continuation turn，并检查 goal 状态、token usage、通知等。

相关位置：

```text
codex-rs/app-server/tests/suite/v2/thread_resume.rs
```

### Python SDK 测试

`test_private_goal_operation_coalesces_runtime_continuations` 会模拟三次模型请求：

1. 第一轮返回 `Initial pass complete.`
2. 后续 continuation 调用 `update_goal({"status":"complete"})`
3. 最后一轮输出 `Goal complete.`

测试会断言：

- SDK 对外只暴露成一个逻辑 goal operation。
- routed turn id 都归到同一个 logical turn。
- 模型请求里确实包含 `<objective>Improve benchmark coverage</objective>`。
- 最终结果是 `Goal complete.`。
- goal operation 完成后路由被清理。

相关位置：

```text
sdk/python/tests/test_app_server_goal_operations.py
```

## 9. 关键文件

```text
codex-rs/ext/goal/src/tool.rs
codex-rs/ext/goal/src/extension.rs
codex-rs/ext/goal/src/runtime.rs
codex-rs/ext/goal/src/api.rs
codex-rs/ext/goal/src/steering.rs
codex-rs/ext/goal/templates/goals/continuation.md
codex-rs/state/src/runtime/goals.rs
codex-rs/app-server/src/request_processors/thread_goal_processor.rs
sdk/python/src/openai_codex/_goal.py
sdk/python/src/openai_codex/client.py
```

## 10. 一句话总结

`goal` 不是单个长时间运行的 turn，而是多个普通 turn 通过持久化 goal 状态和 idle 自动续跑串起来的“逻辑长任务”。

自动续跑时，程序会在上一轮 turn 结束后读取当前 goal 状态；如果仍然是 `active`，就把 `continuation.md` 渲染成一段内部 user-context 类输入，注入模型上下文，并启动下一轮普通 turn。
