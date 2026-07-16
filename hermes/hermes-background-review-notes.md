# Hermes background_review 与 skills 进化机制梳理

## 范围

这份笔记整理前面讨论过的内容：Hermes Agent 的 `background_review` 如何触发、如何判断 `review_memory` 和 `review_skills`、`skill_manage` 如何被调用、长上下文和压缩后的上下文如何处理。

## 核心概念

`background_review` 是一次后台自我改进流程。正常对话 turn 结束后，Hermes 可能会 fork 一个单独的 review agent。

这个 review agent 会拿到当前对话消息快照，再加上一段 review prompt，然后判断是否需要：

- 写入 memory
- patch 已有 skill
- 给 skill 增加 `references/`、`templates/`、`scripts/`
- 创建新的本地 skill

这个流程不会阻塞主对话。它是 best-effort 的后台任务，并且工具权限被限制在 memory 和 skills 相关工具里。

## review agent 看到什么

review fork 收到的主要输入是：

- `messages_snapshot`：触发 review 时对当前 `messages` 做的浅拷贝，也就是 `list(messages)`
- review prompt：`_MEMORY_REVIEW_PROMPT`、`_SKILL_REVIEW_PROMPT` 或 `_COMBINED_REVIEW_PROMPT`
- 一句额外限制：只能调用 memory 和 skill management 工具

review prompt 不是放进 system prompt，而是作为新的 `user_message` 传给 `review_agent.run_conversation(...)`。

最终结构可以理解为：

```text
system:
  继承主 agent 的 cached system prompt

conversation_history:
  messages_snapshot
  或者在 routed auxiliary model 场景下使用 digest + 最近消息尾巴

new user:
  选中的 review prompt
  + "You can only call memory and skill management tools..."
```

## review prompt 在哪里

提示词内容在：

```text
agent/background_review.py
```

三个核心常量是：

- `_MEMORY_REVIEW_PROMPT`
- `_SKILL_REVIEW_PROMPT`
- `_COMBINED_REVIEW_PROMPT`

选择哪个 prompt 的逻辑在 `spawn_background_review_thread(...)`。

实际传给模型的位置是 `review_agent.run_conversation(user_message=..., conversation_history=...)`。

## 如何判定 review_memory

`review_memory` 是按用户 turn 数量触发的。

Hermes 维护一个计数器：

```text
_turns_since_memory
```

每来一轮用户消息，计数器加 1。

达到 `memory.nudge_interval` 后，设置：

```text
review_memory = True
```

默认配置是：

```yaml
memory:
  nudge_interval: 10
```

还需要满足：

- `memory` 在 `agent.valid_tool_names` 里
- `agent._memory_store` 存在
- 本轮有 `final_response`
- 本轮没有被 interrupt

如果有历史 `conversation_history`，会先统计历史里的 user turn 数量，用来恢复 `_turns_since_memory`，避免恢复会话后计数从 0 开始。

## 如何判定 review_skills

`review_skills` 不是直接数历史消息里有多少 `tool`，也不是直接数 assistant 消息。

它看的是 agent loop iteration 计数器：

```text
_iters_since_skill
```

主 agent loop 每跑一轮模型/工具循环，就会累加：

```text
_iters_since_skill += 1
```

在 turn 结束时检查：

```text
_iters_since_skill >= skills.creation_nudge_interval
```

满足后设置：

```text
review_skills = True
```

并立刻重置：

```text
_iters_since_skill = 0
```

默认配置是：

```yaml
skills:
  creation_nudge_interval: 10
```

还需要满足：

- `skill_manage` 在 `agent.valid_tool_names` 里
- 本轮有 `final_response`
- 本轮没有被 interrupt

注意：计数器是在“决定触发 review_skills”时重置，不是等 review agent 真的成功 patch/create skill 后才重置。所以即使后台 review 最后判断 `Nothing to save`，计数也已经重新开始。

## agent loop 是什么

`agent loop` 不是一次会话，也不是最近一次 user。

它是一次用户 turn 内部的模型/工具循环。

例如用户说：

```text
帮我查一下 browser_navigate 是干什么的
```

可能发生：

```text
loop 1:
  assistant 决定搜索代码
  tool: rg browser_navigate

loop 2:
  assistant 决定读 tools/browser_tool.py
  tool: sed / read_file

loop 3:
  assistant 决定读 toolsets.py
  tool: sed / read_file

loop 4:
  assistant 给出最终回答
```

这一次 user turn 里用了 4 次 loop iteration。

如果 `skills.creation_nudge_interval: 3`，这轮结束后会触发 `review_skills`。

如果使用默认值 10，这轮只贡献 4 次累计，还不会触发，后续 turn 继续累计。

## 什么场景会触发 skill_manage

是否更新 skill 是 review agent 这个大模型判断的，不是硬编码规则直接判定。

review prompt 里规定，以下信号值得更新 skill：

- 用户纠正了风格、语气、格式、详细程度
- 用户纠正了工作流、步骤顺序、处理方式
- 对话里出现了非平凡的技巧、修复方案、绕法、排障路径
- 当前加载或咨询过的 skill 被证明缺步骤、过时或错误

优先级是：

1. 更新当前会话里已经加载过的 skill
2. 更新已有 umbrella skill
3. 给已有 skill 添加 support file，比如 `references/`、`templates/`、`scripts/`
4. 如果没有任何已有 skill 覆盖这个类别，才创建新的 class-level umbrella skill

不应该保存成 skill 的内容包括：

- 一次性任务叙述
- 临时环境问题，比如缺依赖、命令没安装、凭证没配
- “某工具不能用”这种负面结论
- 已经通过重试解决的临时错误

更准确地说，触发 `skill_manage` 的不是“问题本身”，而是问题里暴露出的可复用流程、坑点、修复方法或用户偏好。

## skill_manage 如何使用

写入或修改 skill 走 `skill_manage`。

常见动作：

```python
skill_manage(action="create", name="...", content="...")
skill_manage(action="patch", name="...", old_string="...", new_string="...")
skill_manage(action="edit", name="...", content="...")
skill_manage(action="write_file", name="...", file_path="references/...", file_content="...")
```

`skill_view` 只负责读取 skill 内容或 linked files。

`skills_list` 只负责列出可用 skill。

## 本地个人 skill 和后台创建 skill

如果用户一开始就明确说“帮我创建一个本地个人 skill”，前台 agent 可以直接调用：

```python
skill_manage(action="create", name="...", content="...")
```

创建位置是：

```text
~/.hermes/skills/
```

这属于用户指令下的前台创建。

如果是 `background_review` fork 创建的 skill，写入来源会被标记为 `background_review`，后续 curator 可以识别它是 agent-created skill。

前台用户明确要求创建的 skill，不会自动标记为 agent-created。

## review agent 每次看到的是不是整个上下文

默认情况下，是当前会话截至触发点的整份 live `messages` 快照。

也就是说：

```text
messages_snapshot = list(messages)
```

它不只是本轮 user/tool/assistant 的局部片段，而是当前会话状态里的消息历史。

但有两个例外。

## 长上下文处理

长上下文时有两条路径。

### 默认同模型路径

默认情况下，`background_review` 继承主 agent 的模型、provider、base_url、credentials 和 cached system prompt。

在这条路径下：

```text
same model -> full replay
```

也就是把当前 `messages_snapshot` 整段传给 review fork。

设计理由是：同一个模型和同一份前缀上下文已经在主会话里 warm 过 prompt cache，所以 replay full conversation 主要是 cache read。

这保留了最完整的信息，但如果主会话本身已经非常接近 context limit，这条路径仍然会有上下文压力。

### 不同辅助模型路径

如果配置了不同的辅助模型：

```yaml
auxiliary:
  background_review:
    provider: openrouter
    model: google/gemini-3-flash-preview
```

并且这个 provider/model 和主模型不同，则不能复用主模型的 prompt cache。

此时 Hermes 会走 `_digest_history(...)`：

```text
older messages -> 一个合成的 user-role digest
recent tail -> 原样保留，默认最近 24 条消息
```

也就是：

```text
different model -> digest + recent tail
```

这个 digest 还会避免让最近消息尾巴从裸 `tool` 消息开始，以保持消息角色结构正常。

## 如果主 agent 已经压缩过上下文

如果 foreground 主 agent 在 `background_review` 触发前已经做过 context compression，review 看到的是压缩后的当前 live `messages`，不是压缩前原始全量历史。

链路是：

```text
原始长历史
-> 主 agent 触发 context compression
-> messages 被替换为压缩后的 messages
-> turn 结束触发 background_review
-> review_agent 收到 list(messages)
```

review fork 自己禁用 compression：

```python
review_agent.compression_enabled = False
```

原因是 review fork 复用父会话的 `session_id`。如果它自己再压缩，可能和主会话抢 session rotation 和压缩状态。

## 总结

整体流程是：

```text
主 turn 完成
-> turn finalizer 检查 memory / skill 计数器
-> 如果触发，fork background review agent
-> review agent 收到 conversation snapshot + review prompt
-> review prompt 作为 user message 传入
-> review agent 只能调用 memory 和 skills 工具
-> 大模型判断是否写 memory、patch skill、写 support file 或 create skill
-> 成功工具动作被汇总成 "Self-improvement review: ..."
```

默认阈值：

```yaml
memory:
  nudge_interval: 10

skills:
  creation_nudge_interval: 10
```

关键区别：

```text
review_memory:
  基于 user turn 数量

review_skills:
  基于 agent loop iteration 数量
```

