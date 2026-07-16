# session_search 四种调用形态

`session_search` 是一个单一工具，代码通过参数组合自动判断调用形态。入口函数在 `tools/session_search_tool.py`：

```python
def session_search(
    query: str = "",
    role_filter: str = None,
    limit: int = 3,
    db=None,
    current_session_id: str = None,
    session_id: str = None,
    around_message_id: int = None,
    window: int = 5,
    sort: str = None,
    profile: str = None,
) -> str:
```

判断优先级是：

1. `session_id + around_message_id` -> `SCROLL`
2. 只有 `session_id` -> `READ`
3. 没有有效 `query` -> `BROWSE`
4. 有 `query` -> `DISCOVERY`

## 1. DISCOVERY

### 触发方式

传入有效 `query`，且没有 `session_id + around_message_id`，也没有单独 `session_id`。

```python
session_search(query="RAG 评测", limit=5, sort="newest")
```

### 适用场景

不知道具体是哪次会话，需要按关键词搜索历史会话：

- “之前我们做过 RAG 评测吗？”
- “找一下我之前说过的美团订单接入”
- “上次 Codex payload 那次会话在哪里？”

### 代码路径

```python
return _discover(
    db=db,
    query=query.strip(),
    role_filter=role_list,
    limit=limit,
    sort=sort_norm,
    current_session_id=current_session_id,
)
```

`_discover()` 内部调用：

```python
raw_results = db.search_messages(
    query=query,
    role_filter=role_list,
    exclude_sources=list(_HIDDEN_SESSION_SOURCES),
    limit=50,
    offset=0,
    sort=sort,
)
```

### 检索方式

普通文本走 SQLite FTS5：

```sql
WHERE messages_fts MATCH ?
```

FTS5 索引内容是：

```text
messages.content + messages.tool_name + messages.tool_calls
```

不检索：

- `session_id`
- `timestamp`
- `started_at`
- `title`

但 title 有单独的 `_title_match_result()` 辅助匹配。

### 排序

默认按 FTS5 rank，也就是 BM25 相关性：

```text
sort=None -> ORDER BY rank
```

如果传时间排序：

```text
sort="newest" -> ORDER BY m.timestamp DESC, rank
sort="oldest" -> ORDER BY m.timestamp ASC, rank
```

注意：这是对已经命中的结果排序，不是按时间过滤。

### CJK/中文特殊逻辑

如果 query 包含中文、日文、韩文字符，会走 CJK 分支：

- CJK token 足够长且 trigram 可用：走 `messages_fts_trigram`
- CJK token 太短或 trigram 不可用：降级为 SQL `LIKE`

短中文降级后，多 token 是 OR 逻辑。例如：

```text
20260628 6月28 RAG 评测
```

可能变成类似：

```sql
content LIKE '%20260628%'
OR content LIKE '%6月28%'
OR content LIKE '%RAG%'
OR content LIKE '%评测%'
OR tool_calls LIKE ...
```

所以只要命中 `RAG` 或 `评测` 也会返回，不代表按日期字段命中。

### 去重逻辑

FTS5 原始结果可能包含同一个 session 的多条消息，但 `_discover()` 会按 session lineage 去重：

```python
resolved_sid = _resolve_to_parent(db, raw_sid)
if resolved_sid not in seen_sessions:
    seen_sessions[resolved_sid] = row
```

因此最终结果是“最多 N 个不同会话”，不是“最多 N 条消息命中”。

同一会话多条消息命中时，只保留第一条命中消息。

### 默认参数

```python
limit = 3
role_filter = None  # 实际默认为 ["user", "assistant"]
sort = None
```

`_discover()` 默认只搜索：

```python
["user", "assistant"]
```

### 返回内容

每个 result item 主要包含：

```json
{
  "session_id": "...",
  "when": "...",
  "source": "cli",
  "model": "...",
  "title": "...",
  "matched_role": "assistant",
  "match_message_id": 123,
  "snippet": "...>>>命中<<<...",
  "bookend_start": [],
  "messages": [],
  "bookend_end": [],
  "messages_before": 5,
  "messages_after": 5
}
```

字段含义：

- `snippet`：FTS5 或 LIKE 命中的摘要
- `bookend_start`：会话开头的前 3 条 user/assistant 消息
- `messages`：命中消息前后各 5 条窗口，anchor 标记命中消息
- `bookend_end`：会话结尾的后 3 条 user/assistant 消息
- `messages_before` / `messages_after`：窗口前后是否还有更多消息

### 是否包含工具执行结果

默认不包含真正的 `role="tool"` 工具结果。

原因是 discovery 使用：

```python
db.get_anchored_view(hit_sid, msg_id, window=5, bookend=3)
```

而 `get_anchored_view()` 默认：

```python
keep_roles=("user", "assistant")
```

所以：

- `user` 消息会保留
- `assistant` 消息会保留
- `assistant.tool_calls` 会保留
- `role="tool"` 的工具执行结果会过滤掉

例外：如果 anchor 本身是 tool 消息，会保留 anchor。但默认 discovery 的 `role_filter` 不搜 tool，所以通常不会出现。

## 2. SCROLL

### 触发方式

同时传入 `session_id` 和 `around_message_id`。

```python
session_search(
    session_id="20260629_144214_5f25bc",
    around_message_id=4785,
    window=10,
)
```

### 适用场景

已经知道具体会话和某条消息，需要围绕该消息继续展开上下文：

- discovery 返回的 `messages` 不够，需要继续看前后文
- 用户说“继续往前看”
- 用户说“展开这条命中附近的更多消息”
- 需要查看工具执行结果

### 代码路径

`session_search()` 中优先判断：

```python
if session_id and around_message_id is not None:
    return _scroll(...)
```

`_scroll()` 内部调用：

```python
view = db.get_messages_around(
    session_id,
    around_message_id,
    window=window,
)
```

### 查询方式

不走 FTS5，不搜索关键词。

它只围绕指定消息 ID 查局部窗口：

```sql
SELECT * FROM messages
WHERE session_id = ? AND id <= ?
ORDER BY id DESC LIMIT window + 1
```

```sql
SELECT * FROM messages
WHERE session_id = ? AND id > ?
ORDER BY id ASC LIMIT window
```

然后拼成：

```text
anchor 前 window 条 + anchor 本身 + anchor 后 window 条
```

### 默认参数

```python
window = 5
```

`_scroll()` 会把窗口限制在：

```python
1 <= window <= 20
```

### 返回内容

```json
{
  "success": true,
  "mode": "scroll",
  "session_id": "...",
  "around_message_id": 4785,
  "window": 10,
  "messages": [],
  "messages_before": 10,
  "messages_after": 10
}
```

### 是否会打开整个会话

不会。

`SCROLL` 只查局部窗口，不读取整个 session。

### 是否包含工具执行结果

会包含。

`SCROLL` 使用 `get_messages_around()`，这个函数不按 role 过滤，所以返回的 `messages` 可以包含：

- `user`
- `assistant`
- `tool`
- assistant 的 `tool_calls`
- tool 的执行结果内容

因此如果想看工具返回结果，通常要用 `SCROLL` 或 `READ`。

## 3. READ

### 触发方式

传入 `session_id`，但不传 `around_message_id`。

```python
session_search(session_id="20260626_113158_6463d5")
```

跨 profile 读取：

```python
session_search(session_id="20260626_113158_6463d5", profile="work")
```

也支持从 `@session:<profile>/<id>` 形式拆分：

```python
session_search(session_id="work/20260626_113158_6463d5")
```

### 适用场景

已经知道明确的 session，需要读取这个会话：

- 用户直接给了 session id
- 用户给了 `@session:profile/id` 链接
- discovery 找到目标会话后，需要读完整会话摘要
- 想知道某个 session 里发生了什么

### 代码路径

```python
if isinstance(session_id, str) and session_id.strip():
    sid = session_id.strip()
    result = _read_session(db, sid)
```

`_read_session()` 内部：

```python
meta = db.get_session(session_id)
rows = db.get_messages(session_id)
```

### 查询方式

不走 FTS5。

先查 `sessions`：

```sql
SELECT * FROM sessions WHERE id = ?
```

再查 `messages`：

```sql
SELECT * FROM messages
WHERE session_id = ? AND active = 1
ORDER BY id
```

### 截断逻辑

默认：

```python
head = 20
tail = 10
```

如果会话消息数：

```text
total > head + tail
```

则只返回：

```text
前 20 条 + 后 10 条
```

否则返回完整消息。

### 返回内容

```json
{
  "success": true,
  "mode": "read",
  "session_id": "...",
  "session_meta": {
    "when": "...",
    "source": "cli",
    "model": "...",
    "title": "..."
  },
  "message_count": 94,
  "truncated": true,
  "messages": [],
  "message": "Session has 94 messages; showing first 20 + last 10. Pass around_message_id (any id above) to scroll the middle."
}
```

### 是否包含工具执行结果

会包含。

`READ` 使用 `db.get_messages()`，默认只过滤 `active=1`，不按 role 过滤，所以返回的 `messages` 里可能有：

- `user`
- `assistant`
- `tool`
- `tool_calls`
- `tool_call_id`
- `tool_name`

### 读取失败时的兜底

如果当前 profile 的 DB 没找到该 session，代码会扫描其他 profile：

```python
located, owner = _locate_session_db(sid)
```

找到后会把结果加上：

```json
{
  "profile": "..."
}
```

## 4. BROWSE

### 触发方式

没有 `session_id`，也没有有效 `query`。

```python
session_search()
```

或 query 为空：

```python
session_search(query="")
```

### 适用场景

用户没有给关键词，只想看最近会话：

- “我最近在忙什么？”
- “最近有哪些会话？”
- “打开历史会话列表”
- “看看最近的 session”

### 代码路径

```python
if not query or not isinstance(query, str) or not query.strip():
    return _list_recent_sessions(db, limit, current_session_id)
```

`_list_recent_sessions()` 内部：

```python
sessions = db.list_sessions_rich(
    limit=limit + 5,
    exclude_sources=list(_HIDDEN_SESSION_SOURCES),
    order_by_last_active=True,
)
```

### 查询方式

不走 FTS5。

它查 `sessions` 表，并通过子查询拿：

- 第一条用户消息作为 preview
- 最后一条消息时间作为 last_active

### 默认参数

```python
limit = 3
```

`limit` 会被限制在：

```python
1 <= limit <= 10
```

### 过滤逻辑

BROWSE 会：

- 排除当前 session
- 排除当前 session 的 lineage root
- 排除 child/delegation session
- 排除隐藏来源：
  - `subagent`
  - `tool`

### 返回内容

```json
{
  "success": true,
  "mode": "browse",
  "results": [
    {
      "session_id": "...",
      "title": "...",
      "source": "cli",
      "started_at": 1782715357.9,
      "last_active": 1782716036.4,
      "message_count": 94,
      "preview": "我最近在忙什么"
    }
  ],
  "count": 3,
  "message": "Showing 3 most recent sessions. Pass a query= to search, or session_id+around_message_id to scroll."
}
```

### 是否返回 bookend_start / messages / bookend_end

不返回。

`bookend_start`、`messages`、`bookend_end` 是 discovery 结果里的字段。BROWSE 只返回最近会话摘要。

## 四种形态对比

| 形态 | 触发参数 | 是否走 FTS5 | 是否读取完整会话 | 是否包含 tool 结果 | 主要用途 |
|---|---|---:|---:|---:|---|
| DISCOVERY | `query` | 是 | 否 | 默认否 | 按关键词找相关历史会话 |
| SCROLL | `session_id + around_message_id` | 否 | 否 | 是 | 围绕某条消息展开局部上下文 |
| READ | `session_id` | 否 | 小会话全量，大会话前 20 + 后 10 | 是 | 读取确定会话 |
| BROWSE | 无参数 / 空 query | 否 | 否 | 否 | 浏览最近会话摘要 |

## 常见调用示例

### 搜历史关键词

```python
session_search(query="RAG 评测", limit=5, sort="newest")
```

### 只搜用户消息

```python
session_search(query="默认模型", role_filter="user", limit=3)
```

### 读取指定 session

```python
session_search(session_id="20260626_113158_6463d5")
```

### 读取其他 profile 的 session

```python
session_search(session_id="20260626_113158_6463d5", profile="work")
```

### 围绕命中消息继续展开

```python
session_search(
    session_id="20260629_144214_5f25bc",
    around_message_id=4785,
    window=10,
)
```

### 浏览最近会话

```python
session_search()
```
