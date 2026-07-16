# deepagents 中 todolist 的参数

`deepagents` 里的 todo list 是通过 LangChain 的 `TodoListMiddleware` 提供的，工具名是 `write_todos`。

参数结构只有这些：

```json
{
  "todos": [
    {
      "content": "任务内容",
      "status": "pending"
    }
  ]
}
```

每个 todo item 的字段：

- `content`: `string`，任务描述，必填
- `status`: 必填，只能是：
  - `pending`
  - `in_progress`
  - `completed`

注意点：

- 顶层只有一个参数：`todos`
- `todos` 是数组
- `write_todos` 每次调用会替换整个 todo list，不是只更新单个 item
- 同一轮模型输出里不能并行调用多个 `write_todos`，否则会报错

本地确认位置：

- `/Users/jasonzhu/Projects/deepagents/libs/deepagents/deepagents/graph.py:263` 说明默认工具包含 `write_todos`
- `/Users/jasonzhu/Projects/deepagents/libs/deepagents/tests/unit_tests/smoke_tests/snapshots/system_prompt_with_execute_tools.json:4` 里有完整工具 schema
- 实际类型来自 `langchain.agents.middleware.todo.Todo`：`content: str`，`status: Literal["pending", "in_progress", "completed"]`
