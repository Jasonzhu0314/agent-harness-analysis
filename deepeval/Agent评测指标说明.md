# Agent 评测指标说明

## 总览

项目中严格归类到 Agentic 的核心指标有 6 个：

| 层次 | 指标 | 评价内容 |
| --- | --- | --- |
| Reasoning Layer | `PlanQualityMetric` | 计划本身是否完整、合理、高效、可执行 |
| Reasoning Layer | `PlanAdherenceMetric` | agent 执行过程是否遵循计划 |
| Action Layer | `ToolCorrectnessMetric` | 实际工具调用是否符合 expected tools |
| Action Layer | `ArgumentCorrectnessMetric` | 工具调用参数是否合理、正确 |
| Execution Layer | `TaskCompletionMetric` | 最终 outcome 是否完成用户任务 |
| Execution Layer | `StepEfficiencyMetric` | 执行步骤是否高效，有无冗余 |

另外，多轮 agent 场景还有：

| 指标 | 评价内容 |
| --- | --- |
| `GoalAccuracyMetric` | 多轮中每个 user-assistant interaction 的目标完成和计划执行情况 |
| `ToolUseMetric` | 多轮中工具选择和参数生成质量 |
| `ConversationCompletenessMetric` | 整段对话中所有用户意图是否都被满足 |

## ArgumentCorrectnessMetric

`ArgumentCorrectnessMetric` 是 referenceless 的 LLM-as-judge 指标，没有 ground truth / expected arguments。

输入主要是：

```python
LLMTestCase(
    input="用户任务",
    tools_called=[
        ToolCall(
            name="工具名",
            input_parameters={...}
        )
    ]
)
```

它让 LLM judge 基于用户输入和实际工具调用，逐个判断每个 tool call 的参数是否相关、合理、正确。

输出包括：

```python
metric.score
metric.reason
metric.success
metric.verdicts
```

评分逻辑：

```text
score = 非 no verdict 数量 / verdict 总数
```

注意点：schema 允许 `idk`，但当前实现只把 `no` 算错，因此 `idk` 会被算作正确。

## ToolCorrectnessMetric

`ToolCorrectnessMetric` 是 ground-truth based 的工具调用评测。

输入包含：

```python
LLMTestCase(
    input="用户任务",
    tools_called=[...],
    expected_tools=[...]
)
```

默认只比较工具名。可以通过：

```python
ToolCorrectnessMetric(
    evaluation_params=[ToolCallParams.INPUT_PARAMETERS]
)
```

进一步比较工具输入参数；也可以加入 `ToolCallParams.OUTPUT` 比较输出。

如果提供 `available_tools`，还会额外用 LLM 判断 agent 是否在所有可用工具中选择了最合适的工具。最终分数是：

```text
final_score = min(tool_calling_score, tool_selection_score)
```

默认模式下，多调用额外工具通常不扣分；漏掉 expected tool 会扣分。

`should_consider_ordering=True` 会比默认更严格，因为会考虑工具调用顺序，但仍比 `should_exact_match=True` 宽松。

## PlanQualityMetric

`PlanQualityMetric` 评价 agent 生成的 plan 本身是否好，不评价执行是否成功。

它从 trace 中抽取：

- Task：用户原始目标
- Plan：agent 在 thinking / reasoning / 显式计划中体现的计划步骤

评价维度包括：

| 维度 | 含义 |
| --- | --- |
| Completeness | 是否覆盖用户任务的关键要求、前置依赖和必要子任务 |
| Logical Coherence | 步骤顺序是否合理，是否直接导向任务完成 |
| Optimality and Efficiency | 是否最小但充分，有无重复或绕远步骤 |
| Level of Detail | 每一步是否具体、可执行、无歧义 |
| Alignment with Task | 每一步是否直接服务于用户目标 |

如果 trace 中没有抽取到 plan，当前实现会直接给 `score = 1`，并提示没有计划可评估。

## TaskCompletionMetric

`TaskCompletionMetric` 判断 agent 最终是否完成用户任务。

它通常依赖 trace，先抽取：

- `task`：用户想完成什么
- `outcome`：系统实际做了什么

然后用 LLM judge 计算：

```text
Task Completion Score = AlignmentScore(Task, Outcome)
```

它评价的是结果是否完成任务，不评价计划质量、工具调用正确性或执行效率。

如果历史上下文进入 root input / trace，它可以利用上下文；如果上下文没有进入 trace，它看不到。

可以手动指定 task：

```python
TaskCompletionMetric(
    task="规划一个适合老人、节奏轻松的两天杭州行程"
)
```

## GoalAccuracyMetric

`GoalAccuracyMetric` 用于多轮 agent 评测。

输入是：

```python
ConversationalTestCase(
    turns=[
        Turn(role="user", content="..."),
        Turn(role="assistant", content="...", tools_called=[...]),
    ]
)
```

它会把 turns 切成多个 user-assistant interaction。切分规则大致是：遇到 `assistant -> user` 就开始新的 interaction。

每个 interaction 中，`user_goal` 不是 LLM 语义整理出来的，而是代码直接拼接开头连续的 user 消息：

```text
User messages:
...
```

然后对每个 interaction 分别计算：

- GoalScore：用户可见输出是否完成该轮目标
- PlanScore：计划质量和执行是否一致

最终分数：

```text
final_score = (avg_goal_score + avg_plan_score) / 2
```

所以它不是把整段对话压缩成一个全局目标，而是每个 interaction 单独评，再聚合平均。

## ConversationCompletenessMetric

`ConversationCompletenessMetric` 评估整段对话中用户提出的所有意图是否都被满足。

流程：

1. 把完整 turns 交给 LLM，抽取所有高层用户意图。
2. 对每个 user intention，用完整对话判断是否被满足。
3. 计算满足比例。

评分：

```text
score = 满足的用户意图数量 / 用户意图总数
```

它和 `GoalAccuracyMetric` 的区别：

| 指标 | 处理方式 |
| --- | --- |
| `GoalAccuracyMetric` | 按 interaction 拆开，分别评目标完成和计划执行 |
| `ConversationCompletenessMetric` | 从完整对话抽取所有用户意图，再判断每个意图是否被满足 |

`ConversationCompletenessMetric` 更适合评“整段对话有没有把用户所有需求都闭环”。
