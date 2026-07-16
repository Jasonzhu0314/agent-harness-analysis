# session_search 设计优缺点

## 总体设计

`session_search` 的核心思路是：一个工具承载四种调用形态，通过参数组合推断意图。

- `DISCOVERY`：按关键词搜索历史会话
- `SCROLL`：围绕某条消息展开局部上下文
- `READ`：读取确定的 session
- `BROWSE`：浏览最近会话摘要

默认流程是先轻量 discovery，再按需 read/scroll 深挖。

## 优点

### 1. 工具面少，prompt footprint 小

只暴露一个 `session_search`，避免把 `search_sessions`、`read_session`、`scroll_session`、`browse_sessions` 多个工具都放进模型上下文。

这符合 Hermes “core 工具要克制”的设计原则。

### 2. 符合模型工作流

模型通常先不知道具体 `session_id`，所以先用 `query` discovery。

命中后再用：

```python
session_search(session_id="...", around_message_id=...)
```

继续 scroll 展开。

如果用户直接给了 `session_id`，再走 read。

### 3. 默认结果省 token

Discovery 不返回整段 transcript，而是返回：

- `snippet`
- 命中点前后 `±5` 条消息
- 会话开头 `bookend_start`
- 会话结尾 `bookend_end`

模型能快速判断“这是不是我要的会话”，又不会一次吃完整历史。

### 4. 历史上下文保真

`session_search` 不走 LLM 摘要，不做二次生成，返回的是 DB 里的真实消息。

对“之前到底说过什么”这种问题更可靠。

### 5. 本地 FTS5 检索成本低

它不依赖向量库、不依赖外部服务、无 LLM cost，离线可用，速度和可维护性都不错。

### 6. SCROLL / READ 可以追溯工具结果

默认 discovery 会过滤 `role="tool"`，降低噪声。

但需要追查证据时，`SCROLL` 和 `READ` 会保留 tool 消息，可以看到真实工具调用结果。

### 7. 按 lineage 去重合理

压缩、续写、分支场景下，discovery 按逻辑会话 lineage 去重，避免同一个会话刷屏。

## 缺点

### 1. 一个工具多形态，心智负担高

`query`、`session_id`、`around_message_id` 的组合决定模式。

对模型和开发者都不如显式参数直观，例如：

```python
session_search(mode="discover", query="...")
session_search(mode="read", session_id="...")
```

### 2. 默认 discovery 可能隐藏关键 tool 结果

很多事实依据在 `role="tool"` 消息中。

Discovery 默认只保留：

- `user`
- `assistant`

因此模型可能看到 assistant 的结论，但看不到工具返回的原始证据。

### 3. 同一 session 多命中只保留第一条

FTS5 原始结果里，同一个 session 可能有多个 message 命中。

但 discovery 会按 session lineage 去重，每个 session 只保留第一条命中。

这能降噪，但如果一个会话里多个位置都强相关，后面的命中会丢失。

### 4. CJK 短词 fallback 容易扩大召回

中文短词会走 `LIKE` fallback，并且多 token 是 OR 逻辑。

例如：

```text
20260628 6月28 RAG 评测
```

可能因为 `RAG` 或 `评测` 命中大量结果，而不是因为 `20260628` 命中时间字段。

这会提高召回率，但降低精确率。

### 5. 不支持真正的时间过滤

当前只有：

```python
sort="newest"
sort="oldest"
```

这只是对已命中的结果排序，不是时间过滤。

没有：

```python
start_time
end_time
date
```

所以用户问“6月28那天做了什么”，模型不能直接按时间范围查，只能绕关键词或 browse。

### 6. FTS5 不是语义检索

关键词检索对同义表达、别名、语义改写不敏感。

例如：

- “评测集”
- “benchmark dataset”
- “测试样本集”

不一定互相命中，除非文本里都有这些词。

### 7. READ 大会话只返回前 20 + 后 10

这能控制 token，但可能跳过关键中段。

模型需要再选择一个 `around_message_id` 做 scroll，才能继续看中间内容。

### 8. title 匹配和 message 匹配是两套逻辑

主 FTS 索引不包含 title。

Title 有单独的 `_title_match_result()` 辅助匹配。

这可能让使用者误解“query 到底检索了哪些字段”。

### 9. 返回结构可能偏大

Discovery 一个 result 就包含：

- `bookend_start`
- `messages`
- `bookend_end`
- `snippet`
- 元信息

`limit=5` 时 token 压力不小，尤其 assistant 消息里带很长 `tool_calls` 参数时。

### 10. role_filter 容易误用

默认只搜 `user` / `assistant`，不搜 `tool`。

如果模型想查工具输出，必须知道：

- 传 `role_filter="tool"`
- 或用 `SCROLL`
- 或用 `READ`

这对模型判断要求较高。

## 改进方向

### 1. 增加显式 mode

保留当前参数推断兼容旧调用，但允许显式指定：

```python
session_search(mode="discover", query="...")
session_search(mode="read", session_id="...")
session_search(mode="scroll", session_id="...", around_message_id=...)
session_search(mode="browse")
```

### 2. 增加时间过滤

支持：

```python
session_search(query="RAG", start_time=..., end_time=...)
session_search(date="2026-06-28")
```

对应 SQL 可加：

```sql
AND m.timestamp BETWEEN ? AND ?
```

或对 session 层加：

```sql
AND s.started_at BETWEEN ? AND ?
```

### 3. 返回同 session 还有更多命中的信号

例如：

```json
{
  "match_count_in_session": 4,
  "more_matches_in_session": true
}
```

这样模型知道同一个 session 里还有其他相关片段。

### 4. 优化 CJK LIKE fallback

可以把 OR 策略改得更谨慎：

- 中文短词和英文 token 分组
- 日期样 token 不参与 OR 扩召回
- 支持 `match_mode="all"` / `match_mode="any"`

### 5. Discovery 可选包含 tool 结果

例如：

```python
session_search(query="...", include_tool_results=True)
```

只返回 anchor 附近相关 tool 结果，而不是全量 tool 噪声。

### 6. 明确可检索字段

工具描述中可以更明确说明：

- FTS5 搜 `content + tool_name + tool_calls`
- 不搜 `session_id`
- 不搜 `timestamp`
- 不搜 `started_at`
- title 是额外匹配逻辑

这样能减少模型误以为 “query=20260628” 是按时间检索。
