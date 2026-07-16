# Hermes Agent Interrupt / Queue Flow

默认 `interrupt + queue` 时，新问题不会插入当前正在请求的大模型调用里。执行流程是：

1. 用户原问题正在跑：比如“帮我搜索 A 并总结”
2. 中途用户输入新问题：“先别搜了，改成解释 B”
3. 前端把新问题放到 queue，同时调用 `agent.interrupt()`
4. 当前 turn 收尾，保存一段被中断的历史
5. 下一 turn 再把“解释 B”作为新的 `user` 消息，连同历史一起传给大模型

样例，假设第一次任务已经调用了工具：

```json
[
  {
    "role": "system",
    "content": "You are Hermes..."
  },
  {
    "role": "user",
    "content": "帮我搜索 Python 3.15 的最新变化并总结"
  },
  {
    "role": "assistant",
    "content": null,
    "tool_calls": [
      {
        "id": "call_abc",
        "type": "function",
        "function": {
          "name": "web_search",
          "arguments": "{\"query\":\"Python 3.15 latest changes\"}"
        }
      }
    ]
  },
  {
    "role": "tool",
    "tool_call_id": "call_abc",
    "name": "web_search",
    "content": "Operation interrupted."
  },
  {
    "role": "assistant",
    "content": "Operation interrupted."
  },
  {
    "role": "user",
    "content": "先别搜了，改成解释一下 Python GIL 是什么"
  }
]
```

下一轮真正发给大模型的大致就是上面这种结构：前面是历史，最后一条是新问题。模型会看到：上一个任务被中断了，然后用户提出了新的要求。

如果中断发生在还没产生工具调用、也没拿到模型回复时，历史会更短，可能接近这样：

```json
[
  {
    "role": "system",
    "content": "You are Hermes..."
  },
  {
    "role": "user",
    "content": "帮我搜索 Python 3.15 的最新变化并总结"
  },
  {
    "role": "user",
    "content": "先别搜了，改成解释一下 Python GIL 是什么"
  }
]
```

不过代码会尽量修复消息序列，避免传给严格模型时出现非法的 `tool -> user` 尾巴。所以如果中断点在工具之后，会补一个 `assistant: Operation interrupted.` 来闭合上一轮。
