# session_search 问答整理

## 1. 项目中的 recall 使用的是什么工具？

项目里 “recall” 不是一个单一固定工具，分两类：

- 默认跨会话召回用的是 `session_search`。
- 长期记忆 provider 的自动 recall / 工具 recall 取决于配置的 provider，例如 Honcho、Hindsight、Supermemory、Mem0、OpenViking 等插件。

如果问“默认回忆过去对话用什么工具”，答案是 `session_search`。

## 2. session_search 是什么？

`session_search` 是 Hermes 的内置工具，用来搜索历史会话记录，实现跨会话召回。

它查的是本地 `SessionDB` 里的 transcript，底层主要用 SQLite FTS5 全文搜索。典型用途包括：

- “之前我们怎么处理 X 的？”
- “上次做到哪了？”
- “找一下我之前提过的那个需求”
- “我以前有没有说过 Y？”

它不是语义向量库，也不是外部记忆服务；它主要是 Hermes 自己的历史会话全文检索工具。

## 3. 检索时使用的是什么方式？

`session_search` 检索主要用 SQLite FTS5 全文检索。

- 普通文本、英文、代码关键词：走 `messages_fts MATCH ?`
- 默认排序：`ORDER BY rank`，即 FTS5 的 BM25 相关性排序
- `sort="newest"` / `sort="oldest"`：先按时间排序，再用 rank 作为 tie-breaker
- 查询前会做 FTS5 query sanitize，支持短语、布尔、前缀等 FTS5 语法

中文/CJK 有特殊处理：

- 3 个以上 CJK 字符且 trigram 可用：走 `messages_fts_trigram`，即 SQLite FTS5 trigram tokenizer
- 1-2 个 CJK 字符，或 trigram 不可用：降级为 SQL `LIKE '%关键词%'`
- CJK 的 `LIKE` fallback 默认按时间倒序，不按 BM25

索引内容不仅包含 `message.content`，还包含：

```text
content + tool_name + tool_calls
```

所以也能搜到工具名和工具调用内容。

## 4. session_search 的工具描述是什么？

核心描述是：

```text
Search past sessions stored in the local session DB, or scroll inside one.
FTS5-backed retrieval over the SQLite message store. No LLM calls — every
shape returns actual messages from the DB.
```

完整描述包含：

- `SOURCE-FIRST LIMIT`：只搜索 Hermes 历史会话，不代表外部来源当前状态。有 URL、文件、账号、实时系统等直接来源时，应优先查原始来源。
- `FOUR CALLING SHAPES`：
  - `DISCOVERY`：`session_search(query="auth refactor", limit=3)`
  - `SCROLL`：`session_search(session_id="...", around_message_id=12345, window=10)`
  - `READ`：`session_search(session_id="...", profile="work")`
  - `BROWSE`：`session_search()`
- `FTS5 SYNTAX`：默认 AND，多词都要命中；支持 `OR`、引号短语、布尔、前缀通配。
- `WHEN TO USE`：用于回答 Hermes 历史会话相关问题，比如“之前 X 怎么处理的”“上次 Y 做到哪了”“找一下 Z 那次会话”。

## 5. SQLite 里是怎么保存会话的？都保存哪些东西？

SQLite 里主要是两张核心表：

- `sessions`：存会话元信息
- `messages`：存逐条消息

`sessions` 保存的信息包括：

- `id`：session id，主键
- `source`：来源，比如 CLI、gateway、cron 等
- `user_id`：用户标识
- `model`、`model_config`：模型和模型配置
- `system_prompt`：会话系统提示词
- `parent_session_id`：父会话，用于 branch / compression lineage
- `started_at`、`ended_at`、`end_reason`
- 统计字段：`message_count`、`tool_call_count`、`input_tokens`、`output_tokens`、`cache_read_tokens`、`cache_write_tokens`、`reasoning_tokens`、`api_call_count`
- 工作区信息：`cwd`、`git_branch`、`git_repo_root`
- 计费信息：`billing_provider`、`billing_base_url`、`billing_mode`、`estimated_cost_usd`、`actual_cost_usd`、`cost_status`、`cost_source`、`pricing_version`
- UI/状态信息：`title`、`handoff_state`、`handoff_platform`、`handoff_error`、`rewind_count`、`archived`

`messages` 保存的信息包括：

- `id`：自增消息 id
- `session_id`：所属会话
- `role`：`user` / `assistant` / `tool` / `system` 等
- `content`：消息内容
- `tool_call_id`、`tool_calls`、`tool_name`：工具调用相关信息
- `timestamp`
- `token_count`
- `finish_reason`
- 推理内容：`reasoning`、`reasoning_content`、`reasoning_details`
- Codex Responses 结构化项：`codex_reasoning_items`、`codex_message_items`
- `platform_message_id`：外部平台消息 ID
- `observed`：是否是观察到的平台消息
- `active`：是否仍在当前 transcript 中有效
- `compacted`：是否是压缩归档行

另外还有两个 FTS5 虚拟表用于搜索：

- `messages_fts`：普通全文检索
- `messages_fts_trigram`：CJK/子串检索

它们由 trigger 从 `messages` 自动同步。

## 6. 搜索出来之后，是如何添加到上下文的？

`session_search` 搜索结果不是塞进 system prompt，而是走标准工具调用上下文链路：

1. 模型发起工具调用，例如：

```json
{"name": "session_search", "arguments": {"query": "auth refactor"}}
```

2. Hermes 执行 `session_search()`，返回 JSON 字符串。

3. 执行器把结果追加到当前 `messages` 列表里，作为一条 `role="tool"` 消息：

```python
{
  "role": "tool",
  "name": "session_search",
  "tool_name": "session_search",
  "content": "...搜索结果 JSON...",
  "tool_call_id": "..."
}
```

4. 下一次调用 LLM 时，`messages` 中包含原用户消息、assistant 的 tool_call 消息、`session_search` 的 tool result 消息，模型基于这条 tool result 继续回答。

注意：Hermes 的 memory provider 自动召回是另一条路径。它会把召回内容临时拼到当前 user message 后面，并包在 `<memory-context>` 中，不改 system prompt，也不持久化到原始用户消息。

## 7. 检索出来的内容都有哪些？

`session_search(query=...)` 的 discovery 结果主要返回：

- `session_id`：命中的会话 ID
- `parent_session_id`：如果命中的是压缩/分支链路里的子会话，会带父会话信息
- `title`：会话标题
- `when`：会话时间
- `source`：会话来源
- `model`：该会话使用的模型
- `matched_role`：命中的是 `user` 还是 `assistant`
- `match_message_id`：命中的消息 ID
- `snippet`：FTS5 高亮摘要片段
- `bookend_start`：会话开头前 3 条 user/assistant 消息
- `messages`：命中消息前后窗口，默认命中点前后各 5 条，并标记 anchor
- `bookend_end`：会话末尾后 3 条 user/assistant 消息
- `messages_before`：窗口前还有多少消息
- `messages_after`：窗口后还有多少消息

最外层 JSON 还会有：

- `success`
- `mode`
- `query`
- `results`
- `count`
- `sessions_searched`

其他形态：

- `session_search()`：返回最近会话列表，包括标题、预览、时间戳等
- `session_search(session_id="...")`：返回该会话内容，大会话会截成前 20 条 + 后 10 条
- `session_search(session_id="...", around_message_id=..., window=10)`：返回围绕指定消息的窗口，不走 FTS5，也不返回 bookends

## 8. 如果命中 user，如何调用？

示例：

```python
session_search(
    query='"我要把默认模型改成 gpt-4.1"',
    role_filter="user",
    limit=3
)
```

含义：

- `query`：要检索的关键词或短语
- `role_filter="user"`：只搜用户消息
- `limit=3`：最多返回 3 个不同会话

如果命中后上下文不够，可以继续围绕命中消息展开：

```python
session_search(
    session_id="cli_abc123",
    around_message_id=12877,
    window=10
)
```

这会返回命中消息前后各 10 条消息。

## 9. 默认是几条消息？

默认分两处：

- 检索返回会话数：`limit=3`，即最多返回 3 个不同会话。
- 每个命中会话的上下文窗口：命中消息前后各 `5` 条消息，即 `±5`。
- 另外还会带 `bookend_start` 前 3 条 user/assistant 消息，以及 `bookend_end` 后 3 条 user/assistant 消息。

所以默认一次 `session_search(query="...")` 通常是最多 3 个会话，每个会话包含：命中点附近约 11 条消息（命中 + 前后各 5）+ 开头 3 条 + 结尾 3 条。

## 10. 前后各 5 条的话，带有工具执行结果吗？

默认 `DISCOVERY` 搜索的 `±5` 窗口通常不带 tool 执行结果。

原因是 discovery 路径调用：

```python
db.get_anchored_view(hit_sid, msg_id, window=5, bookend=3)
```

`get_anchored_view()` 默认 `keep_roles=("user", "assistant")`，所以窗口会过滤为：

- 保留 `user`
- 保留 `assistant`
- 丢掉 `tool` 角色消息

例外：命中的 anchor 消息永远保留。如果显式传 `role_filter="tool"` 或包含 tool，才可能出现 tool 消息作为 anchor。

另外：

- assistant 消息里的 `tool_calls` 会保留，也就是说“模型调用了什么工具”可能在 assistant 消息中出现。
- 真正的工具返回结果是 `role="tool"` 的消息，默认 discovery 窗口会过滤掉。
- 如果用 `SCROLL` 形态：`session_search(session_id=..., around_message_id=..., window=10)`，它走 `get_messages_around()`，不会按 role 过滤，所以会带上 tool 执行结果。

## 11. 默认是什么形态？

默认是 `DISCOVERY` 形态，只要传了 `query` 就是它：

```python
session_search(query="关键词")
```

默认行为：

- 默认只检索 `user` 和 `assistant` 消息
- 默认返回最多 3 个不同会话
- 每个命中会话返回命中点前后各 5 条窗口
- 窗口默认过滤掉 `tool` 角色结果
- 额外返回开头 3 条和结尾 3 条 user/assistant 消息

其他形态由参数触发：

- 不传任何参数：`BROWSE`
- 只传 `session_id`：`READ`
- 传 `session_id + around_message_id`：`SCROLL`
