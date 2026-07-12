# LangGraph 工具调用 Human-in-the-loop 原理说明

本文整理当前 demo 中 Human-in-the-loop（HITL）的核心实现思路，重点说明：

- 如何不用 `HumanInTheLoopMiddleware`，直接用 LangGraph 原生能力实现人工审批
- 工具调用前为什么能中断
- HTTP 请求中断后，下一次请求如何恢复执行
- `thread_id`、checkpointer、`interrupt()`、`Command(resume=...)` 分别起什么作用

相关示例文件：

- `examples/human_in_loop_langgraph_demo.py`
- `examples/http_hitl_resume_demo.py`

## 一、核心思想

LangGraph 的 HITL 本质不是“阻塞一个 Python 函数一直等用户”，而是：

1. 图执行到某个节点
2. 节点调用 `interrupt(payload)`
3. LangGraph 把当前执行状态保存到 checkpointer
4. 当前调用返回 `__interrupt__`
5. 外部系统把 `payload` 展示给用户
6. 用户审批后，外部系统再次调用图
7. 使用同一个 `thread_id`
8. 通过 `Command(resume=...)` 把用户输入传回图
9. 图从上次中断的节点继续执行

所以它非常适合 HTTP 场景：第一次请求启动任务，返回“等待人工审批”；第二次请求带审批结果恢复任务。

## 二、Graph 状态设计

示例中使用 `messages` 管理上下文，并通过 `add_messages` reducer 追加消息：

```python
class DemoState(TypedDict, total=False):
    messages: Annotated[list[BaseMessage], add_messages]
    tool_decisions: list[ToolDecision]
```

这里的关键点是：

- `messages` 不会被覆盖，而是通过 reducer 追加
- 用户输入是 `HumanMessage`
- 模型输出是 `AIMessage`
- 工具执行结果是 `ToolMessage`
- 最终模型会基于完整消息上下文生成回答

典型消息流如下：

```text
HumanMessage: 用户反馈订单支付后页面 500
AIMessage: 我将调用 query_order_status 和 create_support_ticket
ToolMessage: query_order_status 的结果
ToolMessage: create_support_ticket 的结果
AIMessage: 最终总结
```

## 三、原生 LangGraph 节点结构

核心图结构是：

```text
START
  -> model
  -> approve_tools
  -> tools
  -> model
  -> END
```

各节点职责：

| 节点 | 职责 |
| --- | --- |
| `model` | 调用大模型，生成普通回复或 tool calls |
| `approve_tools` | 检查最新 `AIMessage.tool_calls`，在工具执行前 `interrupt()` |
| `tools` | 根据人工审批结果执行或拒绝工具 |
| `model` | 读取工具结果，生成最终回答 |

## 四、工具调用前如何中断

模型节点返回带 `tool_calls` 的 `AIMessage`：

```python
AIMessage(
    content="我将先查询订单状态，再创建客服工单。",
    tool_calls=[
        {
            "name": "query_order_status",
            "args": {"order_id": "ORD-1001"},
            "id": "call_query_order",
            "type": "tool_call",
        },
        {
            "name": "create_support_ticket",
            "args": {
                "summary": "订单 ORD-1001 支付后结账页返回 500 错误",
                "priority": "P1",
            },
            "id": "call_create_ticket",
            "type": "tool_call",
        },
    ],
)
```

路由函数发现最新消息里有 tool calls，就进入 `approve_tools`：

```python
def route_after_model(state: DemoState) -> Literal["approve_tools", "__end__"]:
    return "approve_tools" if _pending_tool_calls(state) else "__end__"
```

`approve_tools` 不执行工具，而是构造审批 payload 并中断：

```python
resume_payload = interrupt(
    {
        "action_requests": [
            {
                "name": tool_call["name"],
                "args": tool_call["args"],
                "description": "...",
            }
            for tool_call in tool_calls
        ],
        "review_configs": [
            {
                "action_name": tool_call["name"],
                "allowed_decisions": ["approve", "reject"],
            }
            for tool_call in tool_calls
        ],
    }
)
```

这一步会让当前 graph run 暂停，并返回类似：

```json
{
  "__interrupt__": [
    {
      "value": {
        "action_requests": [
          {
            "name": "query_order_status",
            "args": {"order_id": "ORD-1001"}
          },
          {
            "name": "create_support_ticket",
            "args": {
              "summary": "订单 ORD-1001 支付后结账页返回 500 错误",
              "priority": "P1"
            }
          }
        ],
        "review_configs": [
          {
            "action_name": "query_order_status",
            "allowed_decisions": ["approve", "reject"]
          }
        ]
      }
    }
  ]
}
```

## 五、恢复执行的关键：`Command(resume=...)`

用户审批后，外部系统传回：

```json
{
  "decisions": [
    {"type": "approve"},
    {"type": "approve"}
  ]
}
```

恢复执行时调用：

```python
graph.stream(
    Command(resume={"decisions": decisions}),
    config={"configurable": {"thread_id": thread_id}},
)
```

LangGraph 会回到上次暂停的 `approve_tools` 节点，`interrupt(...)` 这一行会“返回”用户传入的 resume 数据：

```python
resume_payload = interrupt(interrupt_payload)
decisions = resume_payload.get("decisions", [])
return {"tool_decisions": decisions}
```

然后图继续流向 `tools` 节点。

## 六、工具节点如何执行或拒绝

`tools` 节点会把 tool calls 和人工 decisions 按顺序对应起来：

```python
for index, tool_call in enumerate(_pending_tool_calls(state)):
    decision = decisions[index] if index < len(decisions) else {"type": "reject"}

    if decision.get("type") != "approve":
        tool_messages.append(
            ToolMessage(
                content=f"工具调用被人工拒绝：{name}",
                name=name,
                tool_call_id=tool_call["id"],
            )
        )
        continue

    result = TOOLS_BY_NAME[name].invoke(tool_call["args"])
    tool_messages.append(
        ToolMessage(
            content=str(result),
            name=name,
            tool_call_id=tool_call["id"],
        )
    )
```

如果 approve：

- 执行真实工具
- 把结果写成 `ToolMessage`
- 追加到 `messages`

如果 reject：

- 不执行工具
- 写入一个“工具被拒绝”的 `ToolMessage`
- 模型后续可以基于这个结果回答用户

## 七、为什么 HTTP 场景必须使用 `thread_id`

HTTP 是无状态的。第一次 `/start` 请求结束后，Python 调用栈已经返回了。

LangGraph 能继续执行，是因为：

```python
builder.compile(checkpointer=MemorySaver())
```

以及每次调用都传同一个：

```python
config = {"configurable": {"thread_id": thread_id}}
```

`thread_id` 相当于一次会话或一次任务的唯一 ID。

第一次 `/start`：

```python
thread_id = uuid4()
graph.stream(initial_state, config={"configurable": {"thread_id": thread_id}})
```

图执行到 `interrupt()` 后，LangGraph 把状态保存到 checkpointer。

第二次 `/resume`：

```python
graph.stream(
    Command(resume={"decisions": decisions}),
    config={"configurable": {"thread_id": same_thread_id}},
)
```

LangGraph 根据同一个 `thread_id` 找回上次暂停的状态，并从暂停点继续。

## 八、HTTP 接口设计

### 1. `/start`

请求：

```bash
curl -s http://127.0.0.1:8008/start \
  -H 'content-type: application/json' \
  -d '{"order_id":"ORD-1001"}'
```

核心逻辑：

```python
initial_state = {
    "messages": [
        HumanMessage(content="用户反馈：订单 ORD-1001 支付完成后，结账页面返回了 500 错误...")
    ]
}

result = _run_until_interrupt_or_end(initial_state, thread_id)
```

返回：

```json
{
  "thread_id": "xxx",
  "status": "interrupted",
  "interrupt": {
    "action_requests": [
      {
        "name": "query_order_status",
        "args": {"order_id": "ORD-1001"}
      },
      {
        "name": "create_support_ticket",
        "args": {
          "summary": "订单 ORD-1001 支付后结账页返回 500 错误",
          "priority": "P1"
        }
      }
    ]
  },
  "next": ["approve_tools"]
}
```

前端或调用方拿到这个响应后，展示审批 UI。

### 2. `/resume`

请求：

```bash
curl -s http://127.0.0.1:8008/resume \
  -H 'content-type: application/json' \
  -d '{
    "thread_id":"xxx",
    "decisions":[
      {"type":"approve"},
      {"type":"approve"}
    ]
  }'
```

核心逻辑：

```python
result = _run_until_interrupt_or_end(
    Command(resume={"decisions": decisions}),
    thread_id,
)
```

返回：

```json
{
  "thread_id": "xxx",
  "status": "completed",
  "values": {
    "messages": [
      "...",
      "ToolMessage: 订单状态查询结果",
      "ToolMessage: 工单创建结果",
      "AIMessage: 最终总结"
    ]
  }
}
```

## 九、`_run_until_interrupt_or_end` 的作用

这个函数是 HTTP demo 的核心封装：

```python
def _run_until_interrupt_or_end(stream_input: object, thread_id: str) -> dict[str, Any]:
    config = {"configurable": {"thread_id": thread_id}}
    events = []

    for update in GRAPH.stream(stream_input, config=config, stream_mode="updates"):
        events.append(update)
        interrupt_payload = _extract_interrupt(update)
        if interrupt_payload is not None:
            snapshot = GRAPH.get_state(config)
            return {
                "thread_id": thread_id,
                "status": "interrupted",
                "interrupt": interrupt_payload,
                "next": list(snapshot.next),
                "events": events,
            }

    snapshot = GRAPH.get_state(config)
    return {
        "thread_id": thread_id,
        "status": "completed",
        "values": snapshot.values,
        "events": events,
    }
```

它统一处理两种情况：

- 图中断：返回 `status=interrupted`
- 图结束：返回 `status=completed`

这也是 HTTP 服务端最需要的一层抽象。

## 十、和 Middleware 方案的区别

之前使用 `HumanInTheLoopMiddleware` 时：

- 中间件自动识别工具调用
- 自动构造 `action_requests`
- 自动在工具执行前 interrupt
- resume payload 通常是 `{"decisions": [...]}`

现在原生 LangGraph 方案：

- 自己写 `approve_tools` 节点
- 自己构造 interrupt payload
- 自己保存 `tool_decisions`
- 自己在 `tools` 节点决定执行或拒绝

优点：

- 控制更细
- 更容易对接自定义 HTTP 协议
- 更容易做审批策略、权限、审计、超时、撤销等业务逻辑

缺点：

- 代码更多
- 需要自己保证 tool call 和 decision 的对应关系
- 需要自己处理 reject/edit/respond 等扩展决策

## 十一、真实项目中的建议

如果只是快速给工具调用加审批：

```text
优先用 HumanInTheLoopMiddleware
```

如果需要接入业务系统，比如：

- Web 审批页面
- IM 审批卡片
- 审批超时
- 多人审批
- 审批记录落库
- 不同工具不同审批策略
- 审批后异步恢复任务

则建议使用当前这种原生 LangGraph 方案：

```text
model -> approve_tools -> tools -> model
```

并把 `thread_id`、interrupt payload、审批记录、最终状态一起持久化。

## 十二、生产化注意事项

当前 demo 使用：

```python
MemorySaver()
```

它只适合本地演示。生产环境应替换为持久化 checkpointer，例如数据库或 Redis 方案，否则服务重启后中断状态会丢失。

还需要考虑：

- `thread_id` 的权限校验
- 用户是否有权限恢复这个 run
- decisions 数量是否和 tool calls 数量一致
- 工具调用参数是否需要二次校验
- reject 后是否允许模型重新规划
- approve 后工具执行失败如何返回
- 审批超时如何处理
- 审计日志如何记录

核心原则是：`interrupt()` 只负责暂停和恢复，业务安全策略需要在 HTTP 层和工具执行层共同保证。
