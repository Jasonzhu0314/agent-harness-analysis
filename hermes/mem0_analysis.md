# Mem0 Analysis

## 结论
- `mem0` 的 agent 工具是 `mem0_list`、`mem0_search`、`mem0_add`、`mem0_update`、`mem0_delete`。
- `mem0_search` 只有语义检索能力，没有按时间范围检索的显式参数。
- `_read_filters()` 目前只返回 `{"user_id": self._user_id}`，因此读取侧只按用户隔离。
- 自动注入到上下文的 mem0 记忆不带 `id`，所以模型如果只看到注入文本，无法直接 update，通常要先 search/list 拿到 `memory_id`。

## 代码位置
- `plugins/memory/mem0/__init__.py`
- `plugins/memory/mem0/_backend.py`
- `agent/memory_manager.py`

## 关键点
- Platform 模式走 `Mem0.MemoryClient`。
- OSS 模式走 `Mem0.Memory.from_config(...)`。
- 两种模式对 agent 暴露的工具名一致，差异在后端实现。
