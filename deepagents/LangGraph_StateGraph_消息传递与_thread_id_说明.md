# LangGraph StateGraph 消息传递与 thread_id 说明

## 1. StateGraph 如何在 agent 之间传递消息

LangGraph 的 `StateGraph` 不是 agent 直接互发消息，而是通过共享的 `state` 传递。`messages` 只是 state 里的一个字段。

- 每个节点接收当前 state
- 节点返回的是 state 的增量更新
- LangGraph 用 reducer 合并这些更新

典型写法：

```python
class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

这里的 `add_messages` 会把新消息追加到历史里，或者在消息 id 相同的时候覆盖旧消息。

## 2. agent 之间能不能传特定 messages

可以，但通常不是直接改全局 `messages`，而是放到专门的 state 字段里，例如：

```python
class State(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
    agent_b_input: list[AnyMessage]
```

然后 `agent_a` 写入 `agent_b_input`，`agent_b` 只读取这个字段。

## 3. 能不能直接跳转子 agent 并传新 dict

可以用 `Command(update=..., goto=...)` 先更新 state 再跳转：

```python
return Command(
    update={
        "agent_b_input": [HumanMessage(content="只给 agent_b 的上下文")],
        "task": "继续分析",
    },
    goto="agent_b",
)
```

注意：这仍然是更新共享 state，不是给下一个节点塞一个完全独立的新 dict。

## 4. 如果是子图（subgraph）

子图有两种常见方式：

- 父图和子图共享一部分 state key，直接通过 `goto` 和 `update` 传递
- 父图手动调用子图 `invoke(new_dict)`，显式控制输入和输出映射

如果子图 schema 和父图不同，通常更适合用 wrapper 节点自己做映射。

## 5. 恢复会话时用什么

LangGraph 的会话恢复主要依赖：

```python
config = {
    "configurable": {
        "thread_id": "some-thread-id"
    }
}
```

只要 graph 编译时用了 checkpointer，就需要传 `thread_id`、`checkpoint_id` 或 `checkpoint_ns` 之一。最常见的是 `thread_id`。

## 6. 如果不传 thread_id 会怎样

分两种情况：

- 没有 checkpointer：可以直接执行，但不会恢复历史，会话是一次性的
- 有 checkpointer：如果没有传 `thread_id` 等 checkpoint 标识，会报错

常见错误信息是：

```text
Checkpointer requires one or more of the following configurable keys: thread_id, checkpoint_ns, checkpoint_id
```

## 7. 在 Deep Agents 里的封装

Deep Agents 的上层封装通常会自动帮你补 `thread_id`。例如：

- ACP server 会用 `session_id` 作为 `thread_id`
- CLI 侧会生成新的 thread id

所以在这些场景里，用户通常不用手动传，但底层仍然是依赖 `thread_id` 识别同一条会话。

## 8. 拒绝 human-in-the-loop 时 messages 怎么处理

拒绝不是写一条 `HumanMessage`，而是中间件把拒绝结果转成 `ToolMessage(status="error")` 写回 `messages`。

也就是说，模型看到的是“这个工具调用失败/被拒绝了”的工具结果，而不是单独一条用户拒绝消息。

---

结论：

- `StateGraph` 通过共享 state 传递消息
- 特定上下文最好放到单独 state 字段
- 子图可以通过 `update + goto` 或显式 `invoke` 传参
- 会话恢复主要看 `thread_id`
- 有 checkpointer 时，不传 `thread_id` 通常会报错
