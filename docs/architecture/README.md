# 架构与核心方法专题

本目录用“一个问题一篇文档”的方式解释 Pi 中值得单独研究的架构和核心方法。它与 `docs/source-reading` 的区别是：源码阅读指南负责建立全局地图，这里负责把一个局部机制讲透。

## 专题列表

- [Agent Loop 双层循环架构](agent-loop.md)：解释 `agent-loop.ts` 如何启动、续转和停止一次 Agent 运行，以及 steering、follow-up、动态切换和事件流等机制。
- [ProviderStreams、StreamFunction 与 AgentLoop 的统一模型调用链](provider-streams-and-agent-loop.md)：解释统一模型输入、Provider/API 分发、响应事件协议、源码阅读顺序，以及 Anthropic Messages 与 OpenAI Responses 的详细差异。
- [Tool 分层模型与完整调用生命周期](tool-system-layers-and-lifecycle.md)：解释 `ai → agent → coding-agent` 三层 Tool 的结构增量、层间适配、注册治理、执行阶段、错误语义与结果回注。
