# mem0 记忆机制梳理

日期：2026-07-10
仓库：/Users/jasonzhu/Projects/mem0

## 1. add 数据时如何抽取记忆

在当前 OSS 本地 SDK 中，主要入口是：

- `mem0/memory/main.py` 的 `Memory.add()`
- 默认参数 `infer=True`

调用示例：

```python
from mem0 import Memory

memory.add(
    [
        {"role": "user", "content": "我最近换到 Shopify 的支付团队了，下个月开始。"},
        {"role": "assistant", "content": "恭喜！我推荐你看看支付平台相关资料。"},
    ],
    user_id="u1",
)
```

当前主路径会进入 `_add_to_vector_store(..., infer=True)`，走 V3 phased batch pipeline。

抽取前会准备这些上下文：

- 当前输入消息：通过 `parse_messages(messages)` 拼成 `user: ... assistant: ...`
- 最近 10 条同 session 消息：`self.db.get_last_messages(session_scope, limit=10)`
- 向量库中与当前输入相似的 top 10 旧 memory：`self.vector_store.search(..., top_k=10)`

然后调用 LLM：

```python
self.llm.generate_response(
    messages=[
        {"role": "system", "content": ADDITIVE_EXTRACTION_PROMPT},
        {"role": "user", "content": user_prompt},
    ],
    response_format={"type": "json_object"},
)
```

系统 prompt 是 `ADDITIVE_EXTRACTION_PROMPT`，位置：

- `mem0/configs/prompts.py`

它要求 LLM 从 user 和 assistant 消息中抽取值得记忆的信息，输出 JSON：

```json
{
  "memory": [
    {"id": "0", "text": "User is switching to Shopify's payments team next month"}
  ]
}
```

注意：当前这条路径是 ADD-only，只负责新增 memory。

## 2. 抽取示例

输入：

```python
memory.add(
    [
        {"role": "user", "content": "我最近换到 Shopify 的支付团队了，下个月开始。周末我和老婆 Elena 去了 Osteria Francescana 庆祝。"},
        {"role": "assistant", "content": "恭喜！如果你们喜欢纪念日晚餐，我也推荐试试 Le Bernardin。"},
    ],
    user_id="u1",
)
```

LLM 可能抽取：

```json
{
  "memory": [
    {
      "id": "0",
      "text": "User is switching to Shopify's payments team next month"
    },
    {
      "id": "1",
      "text": "User has a wife named Elena, and they celebrated at Osteria Francescana over the weekend"
    },
    {
      "id": "2",
      "text": "User was recommended Le Bernardin for a special anniversary dinner"
    }
  ]
}
```

之后 mem0 会给每条 memory 生成 embedding、UUID，并写入 vector store。

最终返回类似：

```json
{
  "results": [
    {"id": "uuid-1", "memory": "User is switching to Shopify's payments team next month", "event": "ADD"},
    {"id": "uuid-2", "memory": "User has a wife named Elena, and they celebrated at Osteria Francescana over the weekend", "event": "ADD"},
    {"id": "uuid-3", "memory": "User was recommended Le Bernardin for a special anniversary dinner", "event": "ADD"}
  ]
}
```

## 3. hash 去重

mem0 会对抽取出来的 memory 文本做 MD5：

```python
mem_hash = hashlib.md5(text.encode()).hexdigest()
```

如果这个 hash 已经存在，就跳过写入。

它是精确文本去重，不是语义去重。

例如：

```text
User likes spicy food
User likes spicy food
```

这两条文本完全一样，hash 一样，会去重。

但：

```text
User likes spicy food
User enjoys spicy dishes
```

语义接近，但文本不同，hash 不同，代码层面不会强制去重。

语义去重主要依赖 LLM prompt：抽取前会把相似旧 memory 放入 `Existing Memories`，要求模型如果新信息和旧 memory 语义等价且没有新上下文，就不要重复抽取。但这不是硬约束。

## 4. delete 什么时候触发

当前 OSS 的 `add()` 不会自动删除旧 memory。

真正执行删除的情况是业务主动调用：

```python
memory.delete(memory_id="...")
memory.delete_all(user_id="u1")
memory.delete_all(agent_id="a1")
memory.delete_all(run_id="r1")
```

删除逻辑在 `_delete_memory()` 中：

- 从 vector store 删除记录
- history 写入 `event="DELETE"`
- 从 entity store 中移除该 memory id 的关联

旧 prompt `DEFAULT_UPDATE_MEMORY_PROMPT` 里保留了 ADD / UPDATE / DELETE / NONE 的决策规则，但当前 `Memory.add()` 主路径没有调用它。

所以比如已有：

```text
User likes cheese pizza
```

新输入：

```text
I don't like cheese pizza anymore.
```

当前主路径更可能新增：

```text
User no longer likes cheese pizza
```

而不是自动删除旧 memory。

## 5. 时间久了会不会自动删除

当前 OSS 核心不会因为 memory 时间太久自动删除。

memory 会保存：

- `created_at`
- `updated_at`

但没有基于这些字段做 TTL 或定时清理。

如果业务想实现“超过 N 天自动删除”，需要业务层自己做定时任务：

1. 查询超过时间的 memory
2. 调用 `delete(memory_id)`

## 6. 检索时老记忆如何释权

这里要区分 OSS 和 Platform。

### OSS 本地 SDK

当前 OSS 的 `Memory.search()` 没有实现 decay 降权。

OSS 检索主要做：

```text
semantic score + BM25 score + entity boost
```

然后归一化排序。

相关代码：

- `mem0/memory/main.py` 的 `_search_vector_store()`
- `mem0/utils/scoring.py` 的 `score_and_rank()`

### Mem0 Platform

Memory Decay 是 Platform 功能，OSS 中 `project.update(decay=True)` 会报不支持。

Platform decay 是 search-time ranking bias，不删除、不过滤，只改变排序。

核心公式可以理解为：

```text
final_internal_score = original_ranking_score * decay_scaling_factor
```

`decay_scaling_factor` 范围：

```text
0.3x ~ 1.5x
```

大致效果：

```text
刚被检索过        ≈ 1.5x
今天被检索过      1.2~1.4x
闲置几天          0.6~1.0x
闲置几周          0.4~0.6x
闲置几个月/几年   ≈ 0.3x
```

它不会把 memory 的分数降到 0，最低只是乘 0.3。

## 7. 如何判断最近是否被检索过

Platform decay 里不是靠 `created_at` 判断最近是否被检索过，而是靠 access history。

每次 search 返回某条 memory 后，后台会记录一次 reinforcement，也就是“这条 memory 刚被检索返回过”。

每条 memory 最多保留最近 20 次访问时间。

示例：

```text
memory A:
access_history = [
  "2026-07-02T10:00:00Z",
  "2026-07-01T09:30:00Z"
]
当前时间 = 2026-07-02T11:00:00Z
=> 最近 1 小时刚被检索过，强 boost
```

```text
memory B:
access_history = [
  "2026-03-01T10:00:00Z"
]
当前时间 = 2026-07-02
=> 4 个月没被检索过，明显降权
```

如果开启 decay 前已经存在旧 memory，没有 access history，则 fallback 到 `updated_at`，把它当作一次历史访问。

## 8. update 什么时候触发

当前 OSS 的 `add()` 不会自动更新已有 memory。

`update()` 只有业务主动调用时才会执行：

```python
memory.update(
    memory_id="mem_123",
    data="User lives in Shanghai after moving from Beijing in June 2026",
    metadata={"source": "user_correction"},
)
```

参数：

```text
memory_id: 要更新的 memory ID
data:      更新后的完整 memory 文本
metadata:  可选，合并或覆盖到原 payload 中
```

它不是让大模型判断怎么改。传入的 `data` 就是最终的新 memory 文本。

内部步骤：

1. `vector_store.get(memory_id)` 获取旧 memory
2. 保存旧文本 `prev_value`
3. 用传入的 `data` 覆盖 payload["data"]
4. 重新计算 hash
5. 重新计算 `text_lemmatized`
6. 重新计算 embedding
7. 调 `vector_store.update(...)`
8. history 记录 `event="UPDATE"`
9. 清理旧实体链接，基于新文本重新抽实体并链接

会用模型的地方主要是 embedding 模型：

```python
self.embedding_model.embed(data, "update")
```

不是聊天大模型。

## 9. 什么叫当前 OSS 主流程

当前 OSS 主流程指的是本地开源 SDK：

```python
from mem0 import Memory

memory = Memory.from_config(...)
memory.add(...)
memory.search(...)
memory.update(...)
memory.delete(...)
```

也就是实际执行 `mem0/memory/main.py` 里的代码。

它不包括：

- Hosted Mem0 Platform 后端隐藏逻辑
- `MemoryClient(api_key=...)` 云端 API 的内部实现
- 仓库中保留但当前没有被 `add()` 主路径调用的旧 prompt

因此，当前 OSS 主路径的特点是：

- `add()`：LLM ADD-only 抽取并新增 memory
- `search()`：语义检索 + BM25 + entity boost，不做 decay
- `update()`：显式覆盖式更新，不让 LLM 决策
- `delete()`：显式删除，不由 add 自动触发
