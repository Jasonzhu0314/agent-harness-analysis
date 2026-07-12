# OpenAI Codex CLI

> OpenAI 推出的官方 Agent CLI 工具，用于编程和自动化任务。

## 源码仓库

https://github.com/openai/codex

## 笔记

- [Goal 执行原理说明](Goal执行原理说明.md)
- [Loop 工程与 Codex 项目实现调研](Loop工程与Codex项目实现调研.md)

## 分析要点

- [x] CLI 入口与命令分发
- [x] Agent 循环（plan → act → observe）
- [x] 工具注册与调用机制
- [ ] 上下文管理与 token 策略
- [ ] 沙箱与安全隔离
