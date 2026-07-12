# DeepAgents 并发写资源处理笔记

## 问题

当多个 agent 并发写同一个文件或同一条记录时，框架层面的 subagent 状态隔离不能自动保护外部共享资源。可能出现后写覆盖先写、基于旧内容提交、字符串替换失败、数据库记录丢更新等问题。

## 推荐策略

优先采用单写者原则：subagent 各自输出隔离结果，由父 agent 串行合并并写入共享目标。

如果必须让多个 agent 直接写同一个资源，需要在工具或 backend 层加入两类保护：

1. 按资源 key 加锁
2. 版本校验，也就是乐观并发控制

## 按资源 key 加锁

同一个资源 key 同一时间只允许一个写流程进入。不同资源 key 不互相阻塞。

```python
import asyncio
from collections import defaultdict

_locks: dict[str, asyncio.Lock] = defaultdict(asyncio.Lock)

async def update_file(path: str, old: str, new: str) -> None:
    lock = _locks[path]
    async with lock:
        content = await read_file(path)
        updated = content.replace(old, new)
        await write_file(path, updated)
```

本进程内可以使用 `asyncio.Lock` 或 `threading.Lock`。多进程或多机器需要外部锁，例如 Redis lock、数据库 advisory lock、etcd/ZooKeeper 等。

## 版本校验

版本校验防止“拿着旧内容覆盖新内容”。常见版本可以是文件 hash、mtime、数据库 version 字段、KV etag 等。

```python
async def safe_edit(path: str, expected_hash: str, transform) -> bool:
    lock = _locks[path]

    async with lock:
        current = await read_file(path)
        current_hash = sha256(current.encode()).hexdigest()

        if current_hash != expected_hash:
            return False

        new_content = transform(current)
        await write_file(path, new_content)
        return True
```

如果版本不匹配，说明资源已被别人改过。此时应重新读取、重新 merge、再提交。

## 数据库写法

数据库里通常用 `version` 字段做 CAS：

```sql
UPDATE records
SET value = ?, version = version + 1
WHERE id = ? AND version = ?;
```

如果 `affected_rows = 0`，说明版本已变化，写入失败，需要重新读取并合并。

## 为什么锁和版本都需要

只有锁：可以避免同一时刻写冲突，但不能阻止基于很久以前读到的内容提交。

只有版本：可以发现冲突，但高并发下会频繁失败重试。

锁 + 版本：同一资源串行提交，同时拒绝过期提交。

## 在 DeepAgents 里的建议

对于并行 subagent：

1. 每个 subagent 写自己的隔离路径，例如 `scratch/<task_id>/result.md`。
2. 父 agent 收集所有结果。
3. 父 agent 串行写最终共享文件或共享记录。
4. 如果共享写不可避免，在写工具或 backend 层实现资源 key 锁和版本校验。

