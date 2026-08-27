# 架构与核心方法专题

本目录用“一个问题一篇文档”的方式解释 Pi 中值得单独研究的架构和核心方法。它与 `docs/source-reading` 的区别是：源码阅读指南负责建立全局地图，这里负责把一个局部机制讲透。

## 专题列表

- [Agent Loop 双层循环架构](agent-loop.md)：解释 `agent-loop.ts` 如何启动、续转和停止一次 Agent 运行，以及 steering、follow-up、动态切换和事件流等机制。
