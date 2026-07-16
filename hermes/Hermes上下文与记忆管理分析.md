# Hermes 上下文与记忆管理分析

本文整理 Hermes 项目中上下文、内置记忆文件 `USER.md` / `MEMORY.md`、外部 memory provider，以及 mem0 的检索、同步、写入行为。

## 1. USER.md / MEMORY.md 保存在哪里

`USER.md` 和 `MEMORY.md` 是 Hermes 内置的本地记忆文件，保存在当前 profile 的 Hermes home 下：

```text
$HERMES_HOME/memories/USER.md
$HERMES_HOME/memories/MEMORY.md
```

它们由 `hermes_constants.py` 中的 profile-aware 路径逻辑决定实际位置。

这两个文件的定位不同：

- `USER.md`：更偏用户画像、偏好、长期稳定事实。
- `MEMORY.md`：更偏通用长期记忆、项目/任务/历史事实。

## 2. USER.md / MEMORY.md 如何进入模型上下文

Agent 初始化时会读取本地 memory 文件：

- `agent/agent_init.py` 中调用 `MemoryStore.load_from_disk()`。
- 读取后的内容会进入系统提示词。
- 注入位置在 `agent/system_prompt.py`，属于系统 prompt 的一部分。

因此，大模型能看到的是 agent 初始化时读取到的 `USER.md` / `MEMORY.md` 快照。

重要结论：

- 当前会话中，模型看到的通常是初始化时的快照。
- 文件后续被修改后，不会自动实时刷新到当前系统提示词里。
- 工具写入前会重新读取磁盘，但那是工具内部的安全写入逻辑，不等于把最新版重新注入模型上下文。

重新加载通常发生在：

- 新建 agent / 新会话。
- 恢复会话时重新初始化 agent。
- 上下文压缩或 prompt 重建相关路径。
- 其他代码显式重新加载 memory store。

## 3. 修改 USER.md 时是否会先读取

会。修改 `USER.md` 不是直接盲写，而是在 `memory` 工具内部执行 read-modify-write。

以 `add` 为例，逻辑在 `tools/memory_tool.py`：

```python
with self._file_lock(self._path_for(target)):
    self._reload_target(target, skip_drift=True)
    entries = self._entries_for(target)
    ...
    entries.append(content)
    self.save_to_disk(target)
```

流程是：

1. 获取目标文件锁。
2. 在锁内重新读取磁盘上的最新 `USER.md`。
3. 解析为 entries。
4. 做 add / replace / remove。
5. 写回磁盘。

不同操作的差异：

- `add`：重读磁盘，但跳过 drift guard，因为它是追加，通常不会覆盖已有内容。
- `replace` / `remove` / `batch`：重读磁盘，并检查 external drift。如果文件内容无法安全 round-trip，会中止修改，避免覆盖人工编辑或异常格式。

所以：模型负责决定是否调用工具、要写什么；工具负责内部安全读取最新文件并落盘。

## 4. 什么情况下会触发 USER.md / MEMORY.md 更新

内置记忆文件主要由 `memory` 工具修改。

大模型看到 `memory` 工具后，会在认为需要长期保存信息时调用它，例如：

- 用户明确表达长期偏好。
- 用户纠正了模型对自己的理解。
- 用户给出稳定身份、习惯、项目背景。
- 用户希望以后记住某件事。
- 模型判断某条信息对未来会话有持续价值。

写入前工具还会做安全检查：

- 内容不能为空。
- 扫描 prompt injection / exfiltration 风险模式。
- 检查重复内容。
- 检查字符上限。
- 加锁并重新读取最新磁盘状态。

## 5. memory 工具 add 功能如何实现

`memory` 工具的 `add` 实现在 `tools/memory_tool.py` 的 `MemoryStore.add()`。

核心逻辑：

1. `content.strip()`。
2. 空内容直接拒绝。
3. `_scan_memory_content(content)` 做安全扫描。
4. 获取目标文件锁。
5. `_reload_target(target, skip_drift=True)` 读取最新磁盘内容。
6. 检查是否完全重复。
7. 计算追加后的总字符数是否超过限制。
8. append 到 entries。
9. `save_to_disk(target)` 写回文件。
10. 返回成功结果。

它不是直接字符串拼接文件，而是解析成 entries 后再保存。

## 6. 写入后通知外部 memory provider 的 on_memory_write() 是什么

当内置 `memory` 工具执行 add / replace 后，Hermes 会把这次写入通知外部 memory provider。

调用位置包括：

- `agent/tool_executor.py`
- `agent/agent_runtime_helpers.py`

最终会调用：

```python
_memory_manager.on_memory_write(...)
```

`MemoryManager.on_memory_write()` 会把 action、target、content、metadata 转发给外部 provider。

注意：

- 它会跳过 builtin provider。
- 通常只通知 add / replace，不通知 remove。
- 这是“内置记忆写入后，外部记忆系统也有机会同步”的扩展点。

## 7. mem0 是什么角色

mem0 是 Hermes 的外部 memory provider，不同于本地 `USER.md` / `MEMORY.md`。

它可以运行在两种模式：

- Platform 模式：使用 `mem0.MemoryClient`，连接 mem0 云服务。
- OSS 模式：使用 `mem0.Memory.from_config`，连接自托管向量库和模型配置。

mem0 的作用更像外部语义记忆库：

- 每轮结束后把当前单轮 user + assistant 同步过去。
- 下一轮前基于 query 检索相关记忆。
- 检索结果注入当前轮上下文。
- 支持模型显式调用 `mem0_search` / `mem0_add` / `mem0_update` / `mem0_delete`。

## 8. USER.md / MEMORY.md 和 mem0 的区别

核心区别：

| 维度 | USER.md / MEMORY.md | mem0 |
| --- | --- | --- |
| 存储位置 | 本地 Markdown 文件 | mem0 平台或 OSS 向量库 |
| 加载方式 | Agent 初始化时全量读取 | 每轮前按语义检索 |
| 注入位置 | 系统提示词 | 当前轮用户消息附近的 memory context |
| 更新方式 | `memory` 工具修改文件 | 自动同步或 mem0 工具写入 |
| 生效方式 | 通常下次初始化/重建 prompt 后完整生效 | 检索命中后动态生效 |
| 透明度 | 文件可直接查看编辑 | 依赖外部存储和检索 |
| 适合场景 | 稳定、重要、少量长期事实 | 大量事实、语义召回、跨会话检索 |

简单理解：

- `USER.md` / `MEMORY.md` 是本地长期档案。
- mem0 是外部语义记忆库。

## 9. mem0 的工具描述

当前 mem0 插件中，`mem0_search` 的工具描述是：

```text
Search memories by meaning. Returns relevant facts ranked by relevance.
```

`mem0_add` 的工具描述是：

```text
Store a durable fact about the user. Stored verbatim (no LLM extraction). Use for explicit preferences, corrections, or decisions.
```

所以模型看到的含义是：

- `mem0_search`：按语义搜索已有记忆。
- `mem0_add`：保存一条明确、长期的用户事实，并且原样保存，不再让 mem0 做 LLM 抽取。

## 10. mem0_search 和 session_search 的区别

两者不是同一个东西。

`mem0_search`：

- 搜索外部 mem0 记忆库。
- 结果是长期记忆事实。
- 支持跨会话、跨历史召回。
- 依赖 mem0 provider 配置。

`session_search`：

- 搜索 Hermes 本地 session 历史。
- 结果来自 SQLite session store / FTS。
- 更像搜索过去对话记录。
- 不等同于 mem0 的语义记忆库。

它们最后都会暴露成模型可调用的工具，但描述、名称、参数和返回内容不同。模型理论上通过工具名和描述区分用途。

## 11. 每轮前检索和每轮后同步是什么意思

mem0 provider 有两个自动流程。

### 每轮后同步

一轮对话完成后，Hermes 会把当前轮内容同步给 mem0。

调用链：

```text
turn_finalizer
-> agent._sync_external_memory_for_turn(...)
-> memory_manager.sync_all(...)
-> mem0.sync_turn(...)
-> backend.add(..., infer=True, ...)
```

mem0 收到的是当前单轮：

```python
[
    {"role": "user", "content": user_content},
    {"role": "assistant", "content": assistant_content},
]
```

这里 `infer=True`，表示让 mem0 自己从这轮对话里判断、抽取值得记忆的事实。

### 每轮前检索

下一轮开始前，Hermes 会把预取到的外部记忆结果注入当前轮上下文。

mem0 的 `queue_prefetch()` 会在上一轮结束后用上一轮 user_text 发起后台检索，结果缓存在 provider 内。

下一轮开始时：

- `memory_manager.prefetch_all(...)` 获取 provider 的预取结果。
- `conversation_loop.py` 把结果包装成 `<memory-context>`。
- 注入当前轮用户消息附近。

这个 memory context 不会持久写入 session messages，只用于当前 API 调用。

## 12. 每轮前检索是默认的吗，没有 mem0 会怎样

每轮前的外部记忆检索依赖 memory provider。

如果没有配置 mem0 或其他外部 memory provider：

- 不会有 mem0 的 prefetch。
- 不会调用 mem0 search。
- 不会注入 mem0 memory context。
- 仍然会有内置 `USER.md` / `MEMORY.md` 的系统提示词快照。

也就是说，mem0 不是内置记忆文件的必要条件。

## 13. 检索结果注入到大模型输入的哪个位置

外部 memory provider 的检索结果会被包装成 memory context block，大致形式是：

```xml
<memory-context>
...
</memory-context>
```

然后注入当前轮用户消息附近，作为当前 API 调用的一部分。

它不是 system prompt，也不是持久写入历史 messages。

系统说明会告诉模型：这是 recalled memory context，不是新的用户输入，是可参考的权威上下文。

## 14. 检索用上一轮问题，当前轮才使用，会不会有问题

这个设计有 trade-off。

当前 mem0 的预取机制是：

- 上一轮结束后，用上一轮 user_text 后台检索。
- 下一轮开始时，把上次检索结果拿来用。

优点：

- 不阻塞当前轮用户请求。
- 外部 memory 慢或失败时，不影响主链路响应速度。
- 可以提前 warm context。

风险：

- 如果当前轮话题突然切换，上一轮预取结果可能不相关。
- 模型可能看到一些对当前问题帮助不大的记忆。

缓解方式：

- 注入块明确标注为 memory context，不是用户新输入。
- 模型仍然可以显式调用 `mem0_search`，按当前问题实时检索。
- 不相关记忆一般只会增加噪声，不会改变用户原始问题。

所以它的主要作用是低延迟召回近期相关记忆，而不是保证每次都针对当前输入实时检索。

## 15. mem0.add 什么时候触发

有两种 add。

### 第一种：每轮结束后的自动同步

这不是大模型主动调用工具，而是 Hermes 代码自动执行。

触发条件大致是：

- 配置了 mem0 provider。
- 当前轮没有被 interrupt。
- `original_user_message` 非空。
- `final_response` 非空。

然后调用：

```python
backend.add(messages, user_id=..., agent_id=..., infer=True, metadata=...)
```

其中 messages 只有当前单轮 user + assistant。

### 第二种：大模型显式调用 mem0_add 工具

这是模型根据工具描述主动决定调用。

当模型认为用户表达了明确偏好、纠正、决定等长期事实时，可以调用：

```python
mem0_add(content="...")
```

最后执行：

```python
backend.add(
    [{"role": "user", "content": content}],
    user_id=...,
    agent_id=...,
    infer=False,
    metadata=...,
)
```

这里 `infer=False`，表示内容原样保存，不让 mem0 再做 LLM 抽取。

## 16. mem0.add 执行的具体是什么

Hermes 插件本身只负责参数组装和转发。

Platform 模式：

```python
self._client.add(messages, **kwargs)
```

也就是调用 `mem0.MemoryClient.add()`。

OSS 模式：

```python
self._memory.add(messages, **kwargs)
```

也就是调用 `mem0.Memory.add()`。

真正的抽取、向量化、去重、存储，由 `mem0` SDK / 服务内部完成。

## 17. 传给 mem0 的 messages 是否只有 user 和 assistant

对自动同步来说，是的。

传给 mem0 的 messages 是：

```python
[
    {"role": "user", "content": user_content},
    {"role": "assistant", "content": assistant_content},
]
```

不会包含：

- `system`
- `tool`
- assistant 的 `tool_calls`
- 工具返回结果
- 完整历史 messages

虽然 `MemoryManager.sync_all()` 的通用接口支持把完整 `messages` 传给 provider，但 mem0 插件的 `sync_turn()` 不接收 `messages` 参数，所以 mem0 不会拿到中间工具调用轨迹。

只有当最终助手回复自然提到了工具结果，例如“我查到订单状态是已发货”，这句话才会作为助手最终回复的一部分被传给 mem0。

显式 `mem0_add` 更少，只传：

```python
[
    {"role": "user", "content": content}
]
```

## 18. 总结

Hermes 的记忆体系可以分成三层：

1. 内置本地记忆：`USER.md` / `MEMORY.md`，初始化时进入系统提示词。
2. 内置 memory 工具：由模型调用，用来安全修改本地记忆文件。
3. 外部 memory provider：例如 mem0，负责语义检索、自动同步和跨会话召回。

关键结论：

- `USER.md` / `MEMORY.md` 是系统提示词里的启动快照，不是每轮实时读取。
- 修改本地记忆时，工具内部会加锁并重新读取最新文件后再写。
- mem0 自动同步每轮只传当前单轮 user + assistant。
- mem0 不接收中间工具调用和工具返回结果。
- 自动同步的 mem0 add 使用 `infer=True`，让 mem0 抽取事实。
- 显式 `mem0_add` 使用 `infer=False`，原样保存模型给出的事实。
