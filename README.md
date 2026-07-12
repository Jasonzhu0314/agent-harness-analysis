# Agent Harness Analysis

![Python](https://img.shields.io/badge/language-Python-blue)

深入源码，分析主流 AI Agent 框架的 Harness（运行引擎）实现方案。

## 分析范围

| 框架 | 定位 | 笔记 |
|------|------|------|
| **Codex** | OpenAI 官方 Agent CLI/SDK | 2 篇 |
| **Hermes** | Nous Research 多智能体框架 | 1 篇 |
| **Claude Code** | Anthropic 官方编程 Agent | — |
| **DeepAgents** | LangChain 多智能体框架 | 4 篇 |

## 扩展分析

| 主题 | 内容 | 笔记 |
|------|------|------|
| **MinerU** | 文档解析工具 | 1 篇 |
| **Mem0** | 记忆存储层 | 2 篇 |
| **DeepEval** | Agent 评测框架 | 1 篇 |
| **Dify** | LLM 应用平台 | 1 篇 |
| **Embedding** | 向量嵌入 & Reranker | 2 篇 |

## 分析维度

每个框架从以下维度展开：

1. **架构设计** — 整体架构、核心模块、数据流
2. **工具调用** — Tool/Function calling 的实现机制
3. **上下文管理** — 消息历史、token 窗口、压缩策略
4. **多智能体协作** — 子 Agent 调度、通信协议
5. **错误处理** — 重试、回退、自修复机制
6. **扩展性** — 插件系统、自定义工具、中间件

## 目录结构

```
├── codex/          # OpenAI Codex CLI
├── hermes/         # Nous Research Hermes
├── claude-code/    # Anthropic Claude Code
├── deepagents/     # LangChain DeepAgents
├── mineru/         # 文档解析 MinerU
├── mem0/           # 记忆存储 Mem0
├── deepeval/       # Agent 评测 DeepEval
├── dify/           # LLM 应用平台 Dify
├── embedding/      # 向量嵌入 & Reranker
└── multi_agent_mesh_research.md  # 多智能体网状架构研究
```

## 初衷

不同 Agent 框架在 Harness 层的设计各有千秋，但社区缺乏系统性的横向对比。本项目通过源码阅读和分析，梳理各框架的核心设计思路和实现细节，帮助开发者更好地理解和选择 Agent 基础设施。
