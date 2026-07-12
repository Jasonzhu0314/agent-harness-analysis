# DeepAgents skill 加载说明

## 结论

`SkillsMiddleware` 会调用传入的 backend，但不固定是 `FilesystemBackend`。如果 `create_deep_agent` 里传的是 `FilesystemBackend`，它就通过这个 backend 扫描 skill 目录并读取 `SKILL.md` 的 frontmatter。

## 关键分层

- `FilesystemMiddleware`：负责把 `ls`、`read_file`、`write_file`、`edit_file`、`glob`、`grep`、`execute` 等工具暴露给模型。
- `FilesystemBackend`：负责这些工具背后的实际文件系统读写。
- `SkillsMiddleware`：负责在 agent 运行前扫描 skill source，解析 skill 元数据，并把可用 skill 列表追加到 system prompt。

## `SkillsMiddleware.before_agent()` 做什么

1. 如果 state 里已经有 `skills_metadata`，直接跳过，避免同一个 thread 重复扫描。
2. 通过 `_get_backend(state, runtime, config)` 获取 backend，支持直接 backend 或 backend factory。
3. 遍历 `sources` 中配置的 skill 目录。
4. 调用 `backend.ls(source_path)` 找 skill 子目录。
5. 调用 `backend.download_files([.../SKILL.md])` 读取每个 skill 的 `SKILL.md`。
6. 解析 YAML frontmatter，得到 `name`、`description`、`path`、`metadata` 等元数据。
7. 同名 skill 后面的 source 覆盖前面的 source。
8. 返回 state update：`skills_metadata` 和可选的 `skills_load_errors`。

## 是否加载全文

不会。`before_agent()` 只加载 `SKILL.md` 的元数据，不会把完整 `SKILL.md` 内容塞进模型上下文。

完整内容通常由模型后续通过 `read_file` 读取，或者 CLI/TUI 中通过 `/skill:<name>` 显式调用时读取并包成用户消息。

## 按 thread_id 控制权限

可以通过 backend factory 从运行时 config 里读取 `thread_id`：

```python
config = {"configurable": {"thread_id": thread_id}}
```

然后返回一个带权限过滤的 backend。这个 backend 至少要拦截：

- `ls()`：只列出当前 thread 有权限的 skill 目录。
- `download_files()` / `read()`：防止模型知道路径后直接读取未授权的 `SKILL.md`。

还要注意：`skills_metadata` 会缓存在 thread state 中。如果某个 thread 的权限变更，需要清理该 thread 的 checkpoint/state，或使用新的 `thread_id`。
