# Mem0 检索过程

下面记录的是仓库里 Python OSS `Memory.search()` 的实际检索链路。

## 总体结论

Mem0 的检索不是单一路径，而是一个组合流程：

1. 入参校验与过滤条件整理
2. 语义向量检索
3. 关键词/BM25 检索
4. 实体提取与实体加权
5. 分数融合与排序
6. 可选 rerank 重排
7. 结果格式化返回

如果使用的是 `MemoryClient`，本地不会执行完整检索，只会把参数发给服务端接口 `/v3/memories/search/`。

## `Memory.search()` 先做什么

代码位置：`mem0/memory/main.py`

检索入口会先处理这些内容：

- 拒绝顶层 `user_id`、`agent_id`、`run_id`，要求放进 `filters`
- 校验 `top_k` 和 `threshold`
- 清理 `query`
- 校验 `filters` 至少包含一个实体 ID
- 处理高级过滤语法：
  - `AND`
  - `OR`
  - `NOT`
  - `eq` / `ne` / `gt` / `gte` / `lt` / `lte`
  - `contains` / `icontains`
  - `*`
- 把过滤条件转换成底层 vector store 可用的格式
- 记录 telemetry 事件 `mem0.search`

## 实际检索流程

### 1. 查询预处理

会先对 query 做两件事：

- `lemmatize_for_bm25(query)`，供关键词检索使用
- `extract_entities(query)`，供实体加权使用

### 2. 向量语义检索

会调用 embedding 模型把 query 编码成向量，然后执行：

- `self.vector_store.search(...)`

这里不是只取最终 `top_k`，而是会先“多取一些”候选：

- `internal_limit = max(limit * 4, 60)`

也就是说，先拿更大的候选池，再做后续融合排序。

### 3. 关键词 / BM25 检索

会再调用：

- `self.vector_store.keyword_search(...)`

如果底层存储支持关键词检索，就会返回结果；不支持则返回 `None`。

之后 Mem0 会把 keyword search 的原始分数归一化成 `bm25_scores`。

### 4. 实体加权

如果 query 里提取出了实体，会走 `_compute_entity_boosts()`：

- 对实体去重，最多 8 个
- 对实体文本做 batch embedding
- 去 `entity_store.search(...)` 查关联实体
- 只保留相似度 `>= 0.5` 的命中
- 从实体 payload 的 `linked_memory_ids` 找到关联 memory
- 计算 entity boost

这个步骤的作用是：如果 query 里出现了明确实体，相关 memory 会被额外加分。

### 5. 分数融合

最后会把三种信号合并：

- 语义相似度分数
- BM25 分数
- entity boost

然后交给 `score_and_rank(...)` 做最终排序、阈值过滤和 top_k 截断。

## 最终返回

排序完成后，结果会被整理成统一结构：

- `id`
- `memory`
- `score`
- `created_at`
- `updated_at`
- `hash`
- 常见字段会被提升到顶层，例如：
  - `user_id`
  - `agent_id`
  - `run_id`
  - `actor_id`
  - `role`
  - `attributed_to`

额外 metadata 会放到 `metadata` 里。

如果传了 `explain=True`，还会附带 `score_details`。

## 可选 rerank

如果：

- `rerank=True`
- 且配置了 reranker

那么在初始排序后还会再做一次重排：

- 同步版：直接调用 `self.reranker.rerank(...)`
- 异步版：用 `asyncio.to_thread(...)` 避免阻塞事件循环

如果 rerank 失败，会回退到原始结果。

## 运行时层面的注意点

- 这条链路里**没有 graph 检索参与**，核心是向量检索 + 关键词检索 + 实体加权 + 可选 rerank。
- `MemoryClient.search()` 只是把参数发给服务端，不在本地执行上述完整流程。

## 代码位置

- `mem0/memory/main.py`
- `mem0/client/main.py`
- `mem0/vector_stores/base.py`

