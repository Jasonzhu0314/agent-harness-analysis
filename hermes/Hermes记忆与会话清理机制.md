# Hermes 记忆与会话清理机制

## 内置长期记忆

Hermes 的内置长期记忆保存在两个文件里：

- `~/.hermes/memories/MEMORY.md`
- `~/.hermes/memories/USER.md`

这部分记忆不是按时间自动过期，而是按字符容量限制。默认配置是：

- `memory.memory_char_limit`: 2200 字符
- `memory.user_char_limit`: 1375 字符

内置 `memory` 工具支持三种操作：

- `add`：新增记忆
- `replace`：替换已有记忆
- `remove`：删除已有记忆

当记忆写满时，Hermes 不会自动删除最旧的内容，而是拒绝新增，并要求模型显式合并、替换或删除旧条目。

可以用下面命令清空内置记忆：

```bash
hermes memory reset --target memory
hermes memory reset --target user
hermes memory reset --target all
```

记忆变更会立即写入磁盘，但系统 prompt 中使用的是 session 启动时捕获的冻结快照。也就是说，本轮 session 中新增或删除的记忆，要到下一个新 session 才会进入系统 prompt。这是为了保持 prompt cache 稳定。

## 会话历史自动清理

按时间丢弃适用于 session 历史，不适用于内置长期记忆。

配置示例：

```yaml
sessions:
  auto_prune: true
  retention_days: 90
  min_interval_hours: 24
  vacuum_after_prune: true
```

含义如下：

- `auto_prune: true`：开启自动清理。
- `retention_days: 90`：保留已结束 session 90 天。
- `min_interval_hours: 24`：自动清理最多每 24 小时真正执行一次。
- `vacuum_after_prune: true`：删除数据后执行 SQLite `VACUUM`，回收磁盘空间。

默认的保留天数是 90 天，但自动清理默认是关闭的。只有显式配置 `auto_prune: true` 后才会自动清理。

## 实际删除规则

自动清理执行时，Hermes 会删除满足以下条件的 session：

```sql
started_at < 当前时间 - retention_days * 86400
AND ended_at IS NOT NULL
```

关键点：

- 只删除已结束的 session。
- 活跃 session 永远不会被自动清理。
- 是否过期看的是 `started_at`，不是 `last_active`。
- 删除内容包括 session 记录、message 记录，以及 `~/.hermes/sessions/` 下对应的 transcript 文件。
- 如果被删 session 有子 session，子 session 不会被连带删除，而是把 `parent_session_id` 置空。
- 如果开启 `vacuum_after_prune` 且确实删除了数据，会执行 SQLite `VACUUM` 来回收空间。

手动清理命令：

```bash
hermes sessions prune --older-than 90
```

## 简短结论

Hermes 有记忆删除功能，也有会话历史按时间清理功能。但只有 session 历史支持按时间自动丢弃；内置长期记忆不会因为时间过长自动删除。

