# Qwen3 Embedding / Reranker Notes

Date: 2026-07-03

## 1. Qwen3-Embedding 从哪一层取结果

Qwen3-Embedding 不是直接取词表 embedding 层，也不是取中间层。

它的推理流程是：

1. 输入文本经过 Qwen3 backbone。
2. 取 `outputs.last_hidden_state`，也就是最后一层 Transformer 对每个 token 的输出。
3. 默认使用 `last_token_pool`，取最后一个有效 token 的 hidden state 作为整句 embedding。
4. 可选按维度截断，例如 `output[:, :dim]`。
5. 最后做 L2 normalize。

所以默认结果可以理解为：

```text
embedding = normalize(last_layer_hidden_state[last_valid_token])
```

对应代码：

- `examples/qwen3_embedding_transformers.py`
- `evaluation/qwen3_embedding_model.py`

## 2. Qwen3-Embedding 是怎么训练的

Qwen3-Embedding 本质上是双塔向量模型，训练方式主要是对比学习。

训练样本包含：

- anchor / query
- positive document
- negative documents

训练目标是：

- query 和 positive 的向量相似度更高
- query 和 negative 的向量相似度更低
- batch 内其他样本也可以作为 in-batch negatives

仓库的 SWIFT 微调文档推荐使用 `InfoNCE`：

```text
--loss_type infonce
```

训练时模型仍然用和推理类似的方式得到向量：

```text
文本 -> Qwen3 backbone -> 最后一层最后 token hidden state -> embedding
```

然后用 cosine similarity 参与 InfoNCE loss。

官方技术报告里提到完整训练是多阶段的：

1. 从 Qwen3 foundation model 初始化。
2. 第一阶段：大规模弱监督训练，约 150M 合成 pair 数据。
3. 第二阶段：高质量监督微调，包含约 7M 标注数据和约 12M 高质量合成数据。
4. 最后做 model merging，提高泛化和鲁棒性。

注意：仓库公开的是推理代码和 SWIFT 微调入口，不是完整复现官方 150M 数据训练管线。

## 3. Qwen3-Reranker 的结构和训练

Qwen3-Reranker 不是双塔 embedding 模型，而是 cross-encoder / generative classifier。

它不会分别编码 query 和 document 后算向量相似度，而是把 query、document、instruction 拼在一起输入 LLM，让模型判断下一 token 是 `yes` 还是 `no`。

输入格式大致是：

```text
system:
Judge whether the Document meets the requirements based on the Query and the Instruct provided.
Note that the answer can only be "yes" or "no".

user:
<Instruct>: ...
<Query>: ...
<Document>: ...

assistant:
<think>

</think>
```

推理时取最后位置 logits：

```python
batch_scores = self.lm(**inputs).logits[:, -1, :]
```

然后只比较 `yes` 和 `no` 两个 token：

```text
score = P(yes | instruction, query, document)
```

所以 reranker 的分数表示“这个 document 是否满足 query”的概率。

训练上，SWIFT 文档支持：

- pointwise：每个 query-doc pair 做二分类，正样本目标为 `yes`，负样本目标为 `no`
- listwise：一组候选文档里学习相对排序

示例训练使用的是：

```text
--task_type generative_reranker
--loss_type generative_reranker
```

## 4. 双塔和 Cross-Encoder 的结构区别

双塔 / Bi-Encoder：

```text
Query    -> Encoder -> q vector
Document -> Encoder -> d vector

score = cosine(q, d) 或 dot(q, d)
```

特点：

- query 和 document 分别编码
- 编码阶段二者互相看不到
- document embedding 可以提前离线计算并建索引
- 适合海量召回
- 速度快，但细粒度交互能力弱

Cross-Encoder：

```text
Query + Document -> LLM / Encoder -> relevance score
```

特点：

- query 和 document 拼在一起输入
- 两者在每层 attention 中都能交互
- 判断更准，能做细粒度匹配
- 每个 query-doc pair 都要跑一次模型
- 成本高，通常用于 Top-K 重排

对应到 Qwen3：

- `Qwen3-Embedding`：双塔，输出 embedding，算向量相似度
- `Qwen3-Reranker`：Cross-Encoder，输出 `P(yes)` 作为相关性分数

## 5. Embedding 里的 instruction 是什么

`instruction` 是给 query 加的一段任务说明，用来告诉 embedding 模型这次匹配的目标是什么。

仓库默认 instruction 是：

```text
Given a web search query, retrieve relevant passages that answer the query
```

query 实际输入会变成：

```text
Instruct: Given a web search query, retrieve relevant passages that answer the query
Query: What is the capital of China?
```

document 一般不加 instruction，直接输入正文。

作用是让同一个 embedding 模型根据任务意图生成不同语义方向的向量。

例如同一个 query：

```text
apple
```

如果 instruction 是：

```text
Retrieve documents about fruit.
```

模型会更偏向水果语义。

如果 instruction 是：

```text
Retrieve documents about technology companies.
```

模型会更偏向 Apple 公司语义。

## 6. 为什么默认 instruction 是那句话

默认值：

```text
Given a web search query, retrieve relevant passages that answer the query
```

它适合通用搜索 / RAG / 问答检索场景。

含义拆解：

- `Given a web search query`：输入是用户搜索问题
- `retrieve relevant passages`：目标是检索相关文本片段，不是直接生成答案
- `that answer the query`：相关性标准是这段文本是否能回答 query

这个默认 instruction 来自官方示例，适合通用检索，但不是唯一正确写法。业务场景不同应该改成对应目标。

例如：

```text
Given a customer service question, retrieve help center articles that answer it.
```

```text
Given a code search query, retrieve code snippets that implement the requested behavior.
```

一句话总结：

```text
instruction 是 query 侧的任务提示词，用来告诉 embedding 模型按什么标准组织语义和匹配文档。
```

## 7. Reranker 的 instruction 是什么

Qwen3-Reranker 里的 `instruction` 也是任务说明，但它不是用来生成 query embedding，而是放进 query-document pair 的判断输入里，告诉模型“应该按什么标准判断这个 document 是否相关”。

Reranker 的输入大致是：

```text
<Instruct>: Given the user query, retrieval the relevant passages
<Query>: 用户问题
<Document>: 候选文档
```

然后模型被要求只回答：

```text
yes / no
```

最终分数是：

```text
score = P(yes | instruction, query, document)
```

所以 reranker instruction 的作用是定义 `yes` 的判断标准。

例如默认 instruction：

```text
Given the user query, retrieval the relevant passages
```

意思是：给定用户 query，判断这个 document 是否是相关 passage。

如果业务场景变了，也应该改 instruction：

```text
Given a customer service question, judge whether the document can answer the question.
```

```text
Given a product search query, judge whether the product description matches the user's intent.
```

Embedding 和 Reranker 的 instruction 区别：

```text
Embedding instruction:
用于指导 query 向量怎么编码，后续用向量相似度召回。

Reranker instruction:
用于指导模型怎么判断一个 query-document pair 是否相关，后续用 P(yes) 重排。
```

## 8. Reranker 分数是怎么从 yes/no logits 算出来的

Reranker 在计算相关性分数时，不是直接使用某个固定的 `yes/no` 阈值。

它的流程是：

1. 把 `instruction + query + document` 输入 reranker。
2. 模型在最后一个位置输出完整词表的 logits。
3. 从完整词表 logits 中只取两个 token：
   - `yes`
   - `no`
4. 对 `[no_logit, yes_logit]` 做二分类 softmax。
5. 取 `yes` 对应的概率作为相关性分数。

代码逻辑类似：

```python
logits = model(**inputs).logits[:, -1, :]

yes_logit = logits[:, token_true_id]
no_logit = logits[:, token_false_id]

pair_logits = torch.stack([no_logit, yes_logit], dim=1)
prob = torch.softmax(pair_logits, dim=1)

score = prob[:, 1]
```

公式是：

```text
score = exp(yes_logit) / (exp(yes_logit) + exp(no_logit))
```

也就是：

```text
score = P(yes | instruction, query, document)
```

注意这里的 softmax 只在 `yes` 和 `no` 两个 token 上做，不是在整个词表上做。

因此 reranker 输出的是连续分数：

```text
0.93 -> strongly relevant
0.51 -> slightly prefers relevant
0.08 -> likely irrelevant
```

在重排场景中，通常不需要固定阈值，而是对同一个 query 下的候选 documents 按 `score` 从高到低排序。

只有在业务上要做过滤或二分类时，才会额外设置阈值，例如：

```text
score >= 0.5 -> relevant
score < 0.5  -> irrelevant
```

这个阈值是业务侧自己定的，不是 reranker 内部固定训练出来的参数。
