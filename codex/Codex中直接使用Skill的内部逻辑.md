# Codex CLI 中直接使用 Skill 的内部逻辑

## 核心结论

在 CLI 命令里写 `$some-skill` 时，Codex 并不是把 skill 当作 CLI 子命令执行，也不是启动一个插件。主流程基本不变：CLI 仍然只是把用户输入当作 prompt 传入，core 在本轮 turn 构造阶段识别到 `$some-skill`，然后读取对应的 `SKILL.md`，把它作为额外上下文注入给模型。

可以理解为：

```text
原本发送给模型：
- developer/system instructions
- 历史上下文
- 用户本轮输入

用了 $some-skill 后：
- developer/system instructions
- 历史上下文
- 用户本轮输入
- <skill>
    <name>some-skill</name>
    <path>.../SKILL.md</path>
    SKILL.md 全文
  </skill>
```

## 示例

比如运行：

```bash
codex exec '用 $openai-docs 查一下 OpenAI API 里 structured outputs 应该怎么用'
```

内部大概会这样走：

1. CLI 收到整段 prompt：

   ```text
   用 $openai-docs 查一下 OpenAI API 里 structured outputs 应该怎么用
   ```

2. CLI 不单独解析 `$openai-docs`，只是把它包成 `UserInput::Text`。

3. core 在 turn 构造阶段扫描文本，识别到 `$openai-docs`。

4. core 从当前已加载、未禁用的 skills 列表中查找 name 为 `openai-docs` 的 skill。

5. 如果匹配成功，就读取对应路径下的 `SKILL.md`。

6. 读取后注入给模型，形式近似：

   ```xml
   <skill>
   <name>openai-docs</name>
   <path>.../openai-docs/SKILL.md</path>
   # OpenAI Docs

   ...
   </skill>
   ```

7. 模型收到用户原始 prompt 和这段额外 skill 上下文后，再根据 `SKILL.md` 里的规则决定如何行动。

## 关键点

- `$some-skill` 本身不会被执行。
- 它只是一个触发符，用来让 Codex 在本轮把对应 `SKILL.md` 注入上下文。
- 真正执行什么命令、读什么文件、调用什么工具，仍然由模型在读完 skill 说明后决定。
- 如果 session 配置允许，模型还会先看到一个“可用 skills 列表”，但那只是目录和使用说明，不是具体 skill 的全文。
- 显式点名 `$some-skill` 后，才会加载该 skill 的 `SKILL.md` 全文。

## CLI 使用注意

在 shell 里，`$xxx` 可能会先被 shell 当环境变量展开，所以推荐用单引号包住 prompt：

```bash
codex exec '用 $openai-docs 查一下 structured outputs'
```

不要写成：

```bash
codex exec 用 $openai-docs 查一下 structured outputs
```

否则 `$openai` 之类的内容可能在进入 Codex 前就被 shell 处理掉。
