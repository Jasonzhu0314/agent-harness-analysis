# LangChain 与 LangGraph 的区别

> 整理时间：2026-07-06
> 最新版本：LangChain 1.3.11 / LangGraph 1.2.7 / LangChain-Core 1.4.8

---

## 一句话概括

- **LangChain** = 搭积木的脚手架（给你零件）
- **LangGraph** = 画流程图的引擎（帮你串流水线）

---

## 各自定位

### LangChain（组件库 + Agent 入口）

把 LLM 应用里常用的东西封装成零件，用 `create_agent` 组装起来。

核心抽象：**Agent = Model + Harness**（模型 + 工具 + 中间件）

典型场景：
- 简单 RAG：用户问 → 查知识库 → 拼 prompt → 调 LLM → 返回答案
- 单步工具调用：读网页 → 提取摘要 → 翻译

### LangGraph（底层编排引擎）

专门解决复杂流程——需要分支、循环、多步骤决策的场景。

核心抽象：**Graph**（节点 + 边），可以有条件跳转、循环、并行、人工审批。

典型场景：
- Agent 多步推理（思考 → 调工具 → 看结果 → 再思考 → ... → 出答案）
- 多 Agent 协作（A 写完 → B 审查 → 不通过回 A → 通过发出去）
- 需要人工确认的流程（LLM 生成草案 → 暂停等人点通过 → 继续）

---

## 核心对比

| 维度 | LangChain | LangGraph |
|------|-----------|-----------|
| 编排方式 | Agent + Middleware | 图（节点+边，可循环分支） |
| 控制流 | 模型自主决策 | 运行时动态决定走哪个分支 |
| 状态管理 | Agent 内置 | 内置持久化，支持断点续跑 |
| 最擅长 | 快速搭简单 Agent | 复杂多步 Agent / 多 Agent 协作 |
| 关系 | 组件库 + 薄封装 | 编排引擎（LangChain 底层用 LangGraph） |

---

## LangChain 1.x 相比 0.x 的变化

### 1. Chain 死了，Agent 上位

**0.x：** 几十种预置 Chain（LLMChain、SequentialChain、RetrievalQA...），学习成本高。

```python
# 0.x
chain = LLMChain(prompt=..., llm=...)
chain.run("hello")
```

**1.x：** 一个 `create_agent` 入口，配置模型、工具、中间件即可。

```python
# 1.x
agent = create_agent(model="openai:gpt-4o", tools=[...])
agent.invoke({"messages": [{"role": "user", "content": "hello"}]})
```

### 2. 底层全部换成了 LangGraph

1.x 的 Agent 底层就是 LangGraph，天然带上：
- 状态持久化（断点续跑）
- 条件分支 / 循环
- 时间旅行（回退到某一步重来）
- 人工审批（暂停等人确认）

### 3. 中间件（Middleware）替代回调（Callback）

**0.x 回调：** 只能监听，不能修改。

```python
class MyHandler(BaseCallbackHandler):
    def on_llm_start(self, ...): ...
```

**1.x 中间件：** 可以拦截和修改 LLM 的输入输出。

```python
@wrap_model_call
def log_middleware(request, handler):
    response = handler(request)  # 继续往下传
    return response
```

### 4. 概念大幅精简

| 概念 | 0.x | 1.x |
|------|-----|-----|
| 入口 | 几十种 Chain 类 | 1 个 create_agent |
| 编排 | 自研 Chain | 底层 LangGraph |
| 扩展 | Callback | Middleware |
| 工具 | 定义松散 | 统一 Tool 接口 |
| 消息 | Schema 混乱 | 统一 Message 体系 |

---

## 现在的产品三级体系

```
┌──────────────────────────────┐
│ Deep Agents   开箱即用       │  ← 内置浏览器/文件系统，一行代码跑
├──────────────────────────────┤
│ LangChain     灵活组装       │  ← create_agent + 拼模型/工具/中间件
├──────────────────────────────┤
│ LangGraph     底层编排       │  ← 图、状态、持久化，复杂流程
└──────────────────────────────┘
```

用买车比喻：
- Deep Agents = 买整车，即买即开
- LangChain = 买零件自己组装（选发动机、轮胎、座椅）
- LangGraph = 定制流水线（造的不只是车，而是复杂的生产线）

---

## 实际项目怎么选

- 简单场景（RAG、单步工具调用）→ 用 LangChain，省事
- 复杂 Agent（多步推理、循环调工具、多人审批）→ 用 LangGraph 自己画图
- 大部分项目 → 混用：LangChain 拿 prompt template 和 retriever，LangGraph 编排流程
