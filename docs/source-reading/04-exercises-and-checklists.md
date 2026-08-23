# 练习、问题与验收清单

本文件用于检验架构理解。所有答案都应能指向源码、测试或文档证据。不要用“应该是”“看起来像”作为最终结论。

## 一、源码阅读笔记模板

每个核心文件使用以下模板：

```markdown
## 文件路径

### 一句话职责

### 所属架构层

### 上游调用者

### 下游依赖

### 拥有的状态

### 主要公开接口

### 主要运行路径

### 扩展或替换边界

### 异步、取消和失败边界

### 与相邻模块的职责区别

### 第一遍跳过的内容

### 源码证据

### 尚未解决的问题
```

## 二、状态所有权练习

填写以下表格。一个状态只能有一个权威拥有者；其他对象可以缓存、投影或引用，但不能含糊地写“大家都管理”。

| 状态 | 权威拥有者 | 缓存/投影视图 | 持久化位置 | 生命周期 |
|---|---|---|---|---|
| 当前模型 |  |  |  |  |
| 当前 thinking level |  |  |  |  |
| Agent transcript |  |  |  |  |
| 完整 Session tree |  |  |  |  |
| 当前 Session leaf |  |  |  |  |
| 流式 assistant partial |  |  |  |  |
| pending Tool calls |  |  |  |  |
| steering queue |  |  |  |  |
| follow-up queue |  |  |  |  |
| 有效 Tool 列表 |  |  |  |  |
| Extension handlers |  |  |  |  |
| 可用模型 snapshot |  |  |  |  |
| Provider credentials |  |  |  |  |
| TUI components |  |  |  |  |

完成后重点检查：

- Agent transcript 和完整 Session tree 不能写成同一个状态。
- ModelRuntime snapshot 和底层 Provider catalog 不能混为一谈。
- UI 只能观察或展示状态，不应成为 Agent 或 Session 的权威来源。

## 三、边界判断练习

对每个需求写出“首先进入的 package/文件”和“不应该首先修改的层”。

### 练习 1：增加一个只读目录列表 Tool

应该判断：

- Tool schema 和执行属于哪里？
- 是否需要系统提示词贡献？
- 如何加入默认或可选 Tool 集合？
- Agent Loop 是否需要修改？通常不需要。

### 练习 2：支持一个新的 LLM Provider

应该判断：

- 是复用现有 API 类型，还是增加新 API Adapter？
- Provider factory、auth 和 model catalog 放在哪里？
- Coding Agent 是否需要 provider composition 支持？
- Agent Core 是否需要修改？通常不需要。

### 练习 3：增加 Web UI

应该判断：

- AgentSession/Runtime 是否可以复用？
- 需要新的 Mode adapter，还是使用远程 Client/Server？
- Web UI 是否应直接依赖 Provider API？不应。
- TUI package 是否可直接复用？通常不能直接作为浏览器 UI，但其上层事件模型可参考。

### 练习 4：增加 Tool 权限确认

比较三种方案：

1. 修改每个 Tool execute。
2. 修改 Agent Loop。
3. 使用 `beforeToolCall`/Extension `tool_call` hook。

说明第三种为何通常是最小且可配置的产品级方案；同时指出如果需要强制、不可绕过的全局安全边界，进程内 Extension hook 本身仍不足以替代沙箱。

### 练习 5：增加长期语义记忆

应该先明确：

- 当前 Session/Compaction 不是向量 Memory。
- 检索发生在每次 Context 构建前，还是由 Tool 主动触发？
- 持久化属于独立 backend 还是 Session Entry？
- 注入结果如何经过 `transformContext`/`convertToLlm`？
- 如何避免把完整敏感数据写入 telemetry？

### 练习 6：改变 Session 文件格式

应该保护：

- Agent Core 不感知文件格式。
- AgentSession 继续使用稳定 Session 接口。
- 恢复、branch、compaction 和扩展 Entry 语义不被破坏。
- 迁移和向后兼容策略必须明确。

## 四、调用链填空

### 普通 Prompt

```text
_____ Mode
→ AgentSession._____
→ Agent._____
→ runAgentLoop
→ streamAssistantResponse
→ _____ Runtime.streamSimple
→ Provider._____
→ API Adapter
→ AssistantMessageEventStream
```

### Tool Prompt

```text
Assistant ToolCall
→ executeToolCalls
→ _____ Tool by name
→ prepare/validate arguments
→ _____ hook
→ Tool._____
→ _____ hook
→ ToolResultMessage
→ current Context
→ next _____
```

### Session 恢复

```text
SessionManager._____
→ load JSONL entries
→ build indexes
→ buildSessionContext
→ restore model/thinking/messages
→ Agent.state._____
```

### Extension

```text
ResourceLoader._____
→ discoverAndLoadExtensions
→ execute extension _____
→ collect registrations
→ new Extension_____
→ AgentSession binds hooks and tools
```

完成后必须回源码核对名称，不凭记忆猜测。

## 五、架构口述问题

### 基础级

1. 为什么这是 monorepo？
2. 哪三个 package 构成核心纵向主链？
3. CLI 的源入口和真正 Agent Loop 分别在哪里？
4. `Agent` 和 `AgentSession` 的职责差异是什么？
5. Tool 定义在哪里，Tool 执行在哪里？
6. LLM Provider 适配在哪里？
7. 当前 Session 使用什么持久化形式？
8. Extension 的发现和执行分别由谁负责？

### 进阶级

1. 为什么 `Agent` 需要 `convertToLlm` 和 `transformContext` 两个阶段？
2. 为什么 `agent_end` 不一定表示所有副作用都已经完成？
3. 为什么 Agent Core 不应该知道 SettingsManager？
4. 为什么 ModelRuntime 不直接取代 pi-ai Models？
5. 为什么 Session 使用追加式 Entry，而不是反复重写消息数组？
6. Compaction 如何在不删除历史的情况下缩短 Context？
7. Extension Provider 为什么必须参与 Runtime rebuild？
8. Tool 并行完成时，为什么 transcript 仍保持原调用顺序？

### 高级级

1. 当前 AgentSession 为什么容易成为复杂度中心？
2. 哪些职责未来可能迁移到通用 AgentHarness？
3. durable operation record 对 crash recovery 有什么价值？
4. 远程 Session snapshot 为什么不能只靠 progress event 归约？
5. 如果 Extension 在 Session 切换后保留旧 Context，会产生什么问题？
6. 如果 Provider auth 在启动时固定，OAuth 过期会产生什么问题？
7. 如果 Session persistence 订阅者不被 await，Tool 执行前可能出现什么竞态？
8. 哪些安全边界只能由容器、沙箱或 transport auth 提供？

## 六、事件时序练习

### 练习 A：无 Tool

写出完整事件序列，并标记：

- Agent state 写入。
- Session 持久化。
- UI 可见更新。
- run settlement。

### 练习 B：两个并行 Tool

假设 Tool A 用时 2 秒，Tool B 用时 200 毫秒，但 Assistant 中 A 在 B 前面。

回答：

- start 事件顺序是什么？
- end 事件可能是什么顺序？
- ToolResultMessage 进入 transcript 的顺序是什么？
- 下一次 LLM 调用何时开始？

### 练习 C：Tool 被 Extension 阻止

写出：

- 哪个事件已经发出？
- Tool execute 是否运行？
- LLM 收到什么类型的结果？
- `terminate` 如何影响下一轮？

### 练习 D：Provider 流被取消

写出：

- AbortSignal 从哪里创建？
- 如何传到 Provider？
- Assistant stopReason 可能是什么？
- Session 是否保存最终失败/取消消息？
- Agent 何时变为 idle？

## 七、Session 树练习

给定 Entries：

```text
A user
└─ B assistant
   ├─ C user
   │  └─ D assistant
   └─ E user
      └─ F assistant
         └─ G compaction
```

分别以 D、F、G 为 leaf，回答：

1. `getEntries()` 返回什么？
2. `getBranch()` 返回什么？
3. `getTree()` 返回什么结构？
4. `buildSessionContext()` 可能发送哪些消息？
5. G 之后，A-F 是否仍保存在 Session 文件中？

再增加一个 BranchSummaryEntry，说明它应挂在哪条路径以及如何进入 Context。

## 八、Extension 设计练习

设计一个 `architecture-notes` Extension，不要求实现，但必须给出完整架构方案。

### 功能

- `/arch-note <text>`：保存一条架构笔记。
- `architecture_note` Tool：允许 LLM 保存结构化笔记。
- `/arch-list`：列出当前 Session 的笔记。
- Session 恢复后笔记仍存在。
- 切换 Session 后不应混入旧 Session 笔记。

### 必须回答

1. 注册哪些 command 和 Tool？
2. 笔记存闭包还是 Session Custom Entry？为什么？
3. Tool schema 是什么？
4. 如何在 `session_start` 恢复视图？
5. reload 后旧 Context 如何处理？
6. 是否需要把笔记自动注入 LLM Context？如果需要，在哪个 hook？
7. 如何避免无限增长 Context？
8. Interactive、Print 和 RPC 下 UI 行为有什么差异？

## 九、调试跟踪练习

不修改生产代码的前提下，选择测试、现有日志或调试器完成一次跟踪。

### 跟踪目标

```text
Prompt: 读取 package.json 并总结 workspace
```

### 观察点

1. AgentSession 收到的原始文本。
2. Agent state 中新增的 user message。
3. 第一次 Provider Context 的 message/tool 数量。
4. Assistant ToolCall。
5. read Tool 参数。
6. ToolResult 内容和截断信息。
7. 第二次 Provider Context。
8. 最终 assistant message。
9. Session JSONL 新增的 Entry。

### 跟踪报告模板

```markdown
# Trace Report

## Scenario

## Entry Point

## First LLM Call

## Tool Call

## Tool Result

## Second LLM Call

## Session Writes

## Final Output

## Observed Event Order

## Unexpected Findings
```

## 十、设计取舍记录模板

对以下架构决定各写一份简化 ADR：

- Agent Core 通过 StreamFn 解耦 Provider。
- Session 采用追加式树。
- Extension 使用进程内 TypeScript factory。
- 当前 CLI 使用 AgentSession，而通用 AgentHarness 独立演进。
- 远程协议使用 authoritative snapshot + transient progress。

模板：

```markdown
# 决策标题

## 问题

## 具体场景或短调用链

## 当前方案

## 为什么必要

## 收益

## 代价

## 可替代方案

## 失败模式

## 验证证据
```

## 十一、常见误区检查

阅读过程中定期检查：

- [ ] 没有把 `main.ts` 叫作 Agent 主循环。
- [ ] 没有把 `AgentSession` 和 `SessionManager` 当成同一个东西。
- [ ] 没有把完整 Session history 当成每次 LLM Context。
- [ ] 没有把 SQLite FTS 当成向量 Memory。
- [ ] 没有把 Provider factory 当成 API Adapter。
- [ ] 没有把 TUI 当成 Agent Runtime。
- [ ] 没有把 Coding Agent RPC 和远程 Protocol 混为一谈。
- [ ] 没有把 `AgentHarness` 的接口当成已完成实现。
- [ ] 没有认为 Extension API 是操作系统安全沙箱。
- [ ] 没有通过 generated model 文件学习整体架构。

## 十二、最终架构图验收

最终架构图至少应包含：

- 输入模式层。
- AgentSession/Runtime 产品层。
- Agent 与 Agent Loop。
- ModelRuntime 与 pi-ai。
- Provider/API/LLM。
- Tool 循环。
- Session/Context/Compaction。
- ResourceLoader/Extension。
- TUI。
- Protocol/Client/Server 实验链。
- 通用 AgentHarness/Session Backend 演进链。
- Telemetry/Evals 横切模块。

每条箭头必须有明确语义：调用、注入、持久化、事件、注册或传输。避免使用没有语义的双向箭头。

## 十三、十五分钟讲解验收表

请另一位开发者或录音进行验收：

| 项目 | 权重 | 标准 |
|---|---:|---|
| Package 分层 | 15 | 能准确说明核心包和依赖方向 |
| 启动链 | 10 | 从 CLI 到 AgentSession 无跳步 |
| Agent Loop | 20 | 能解释 turn、Tool 和 queue |
| LLM 分发 | 15 | 能区分 Model/Provider/API |
| Session/Context | 15 | 能解释树、leaf 和 compaction |
| Tool/Extension | 15 | 能解释注册、hook 和结果回传 |
| 当前/实验边界 | 10 | 不夸大 AgentHarness/Server 完成度 |

建议通过线：80/100。任何一项出现核心概念混淆，即使总分超过 80，也应回到对应章节复习。

## 十四、第一周完成清单

### 架构

- [ ] 完成 package 依赖图。
- [ ] 完成当前 CLI 主链图。
- [ ] 完成实验性远程/Harness 图。
- [ ] 完成关键对象职责表。

### 调用链

- [ ] 完成启动链。
- [ ] 完成普通 Prompt 链。
- [ ] 完成 LLM 分发链。
- [ ] 完成 Tool 链。
- [ ] 完成 Session/Context 链。
- [ ] 完成 Extension 链。

### 源码

- [ ] 阅读 coding-agent 入口和 SDK。
- [ ] 阅读 Agent 和 Agent Loop。
- [ ] 阅读 pi-ai Models 和一个 Provider/API。
- [ ] 阅读 AgentSession 关键路径。
- [ ] 阅读 SessionManager 和 Compaction。
- [ ] 阅读 Tool registry 和代表 Tool。
- [ ] 阅读 Extension loader/runner 和代表示例。
- [ ] 阅读模式边界和 TUI 总体接口。
- [ ] 阅读 Protocol/Client/Server 概览。
- [ ] 阅读 AgentHarness 当前实现状态。

### 输出和验证

- [ ] 完成一次端到端源码 trace。
- [ ] 完成一次需求定位练习。
- [ ] 完成一个 Extension 架构设计。
- [ ] 完成至少两份简化 ADR。
- [ ] 完成十五分钟架构讲解。
- [ ] 列出第二周问题清单。

## 十五、第二周问题池模板

把第一周暂时跳过的问题放入以下分类，避免边读边无限扩张范围：

```markdown
## Agent Loop 与并发

## Provider 和协议兼容

## Context 与 token 估算

## Session 恢复和迁移

## Tool 安全与沙箱

## Extension 生命周期

## TUI 和终端协议

## Remote Protocol

## AgentHarness 演进

## Telemetry 和 Evals
```
