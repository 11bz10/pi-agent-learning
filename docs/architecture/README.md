# 架构与核心方法专题

本目录用“一个问题一篇文档”的方式解释 Pi 中值得单独研究的架构和核心方法。它与 `docs/source-reading` 的区别是：源码阅读指南负责建立全局地图，这里负责把一个局部机制讲透。

## 专题列表

- [Agent Loop 双层循环架构](1.agent-loop.md)：解释 `agent-loop.ts` 如何启动、续转和停止一次 Agent 运行，以及 steering、follow-up、动态切换和事件流等机制。
- [ProviderStreams、StreamFunction 与 AgentLoop 的统一模型调用链](2.provider-streams-and-agent-loop.md)：解释统一模型输入、Provider/API 分发、响应事件协议、源码阅读顺序，以及 Anthropic Messages 与 OpenAI Responses 的详细差异。
- [Tool 分层模型与完整调用生命周期](3.tool-system-layers-and-lifecycle.md)：解释 `ai → agent → coding-agent` 三层 Tool 的结构增量、层间适配、注册治理、执行阶段、错误语义与结果回注。
- [从 Coding Agent 到 LLM 的消息架构](4.message-architecture-from-coding-agent-to-llm.md)：解释 SessionEntry、AgentMessage、Message、Provider payload 和事件之间的转换，以及普通消息、工具结果和 UI-only 消息的完整传递链路。
- [Agent Event 流转与扩展管道](5.agent-event-flow-and-extension-pipeline.md)：解释 AgentEvent、AgentSessionEvent 与 ExtensionEvent 的三层模型，`agent.subscribe()`、`session.subscribe()` 和 `pi.on()` 的等待语义、调度规则、错误边界与适用场景。
- [上下文工程：四类控制机制、工具端到端链路与设计精华](6.context-engineering.md)：解释工具截断、系统提示词、Compaction、分支摘要，一次 `read` 调用从 ToolCall 到下一轮请求的完整链路，以及可复用的上下文工程原则。
