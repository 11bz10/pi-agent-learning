# 逐文件源码阅读顺序

本文不是简单的文件列表，而是一条经过优先级排序的源码路径。每个阶段都说明为什么此时阅读该文件、第一遍关注什么、暂时跳过什么，以及读完后应该能回答什么。

## 使用方法

为每个文件建立一条阅读记录：

```text
文件：
所属层：
一句话职责：
上游调用者：
下游依赖：
拥有的状态：
主要输入/输出：
扩展点：
失败/取消边界：
仍不确定的问题：
```

第一遍不要试图解释每个私有方法。先建立跨文件关系，再回到局部实现。

## 阶段 0：仓库入口和依赖图

### 1. 根 `README.md`

目的：获得项目作者给出的正式定位。

第一遍关注：

- 三个核心包：coding-agent、agent-core、pi-ai。
- 项目的最小化、自扩展和可嵌入定位。
- 构建与测试命令只需知道存在，不研究发布流程。

读完回答：这个仓库提供的是一个模型 SDK、一个 Agent Runtime，还是一个完整产品？答案应是三者分层组合。

### 2. 根 `package.json`

目的：确认 workspace、构建顺序和内部依赖方向。

第一遍关注：

- `workspaces`。
- `build` 的 package 顺序。
- 根级 check/test/release 的边界。

暂时跳过：版本和发布脚本细节。

### 3. 各 package 的 `package.json`

推荐顺序：

1. `packages/coding-agent/package.json`
2. `packages/agent/package.json`
3. `packages/ai/package.json`
4. `packages/tui/package.json`
5. `packages/telemetry/package.json`
6. `packages/protocol/package.json`
7. `packages/client/package.json`
8. `packages/server/package.json`
9. `packages/session-backends/sqlite-node/package.json`
10. `packages/evals/package.json`

对每个 package 记录：

- 是否发布。
- bin/main/exports。
- 内部 workspace 依赖。
- 是否 runtime-neutral。
- 是否实验性或 private。

阶段产出：一张只包含 package 的有向依赖图。

---

## 阶段 1：Coding Agent 启动和组装

### 4. `packages/coding-agent/src/cli.ts`

为什么先读：这是 `pi` 命令的源入口，文件很薄，适合作为调用链起点。

第一遍关注：

- 进程标题和环境变量。
- HTTP dispatcher 初始化。
- `main(process.argv.slice(2))`。

暂时跳过：进程 warning 行为的历史原因。

读完回答：真正的应用启动逻辑在哪里？

### 5. `packages/coding-agent/src/main.ts`

为什么此时读：它是 composition root，即把配置、Session、模型、资源和运行模式组合起来的地方。

第一遍只建立以下分段：

1. 参数、stdin/stdout 和模式判断。
2. auth/package 等提前退出命令。
3. Settings 和 SessionManager。
4. Runtime factory。
5. AgentSession 服务与 Session 创建。
6. 初始输入和模型可用性。
7. Interactive/Print/RPC 分流。

第二遍重点函数：

- `resolveAppMode`
- `createSessionManager`
- `buildSessionOptions`
- `main`
- `createRuntime` 内部 factory

暂时跳过：

- 全部 CLI flag 的细节。
- 迁移和 deprecation 文案。
- startup benchmark。
- 每个 auth 子命令。

读完回答：哪些对象在进程启动期间只创建一次，哪些会随 Session 切换而重建？

### 6. `packages/coding-agent/src/core/agent-session-services.ts`

目的：理解可复用服务与具体 Session 的分离。

关注：

- `AgentSessionServices` 的字段。
- `createAgentSessionServices()`。
- ResourceLoader reload 和 Extension Provider 注册。
- `createAgentSessionFromServices()`。

读完回答：为什么 Session 切换时可以重建 Session，又复用部分基础服务？

### 7. `packages/coding-agent/src/core/agent-session-runtime.ts`

目的：理解 new/resume/fork/switch 如何替换当前 Session 及 cwd-bound 服务。

关注：

- `AgentSessionRuntime` 拥有哪些引用。
- `apply()` 如何替换当前状态。
- Session 替换前后的 dispose/rebind。
- 为什么模式层持有 Runtime，而不直接永久持有一个 AgentSession。

暂时跳过：session import 的全部边界情况。

### 8. `packages/coding-agent/src/core/sdk.ts`

目的：理解如何真正创建 Agent、注入模型调用、恢复会话并创建 AgentSession。

按以下顺序读：

1. `CreateAgentSessionOptions`。
2. `CreateAgentSessionResult`。
3. `createAgentSession()` 的服务默认值。
4. 现有 Session 的 model/thinking 恢复。
5. 默认 Tool 集合。
6. `new Agent({...})`。
7. `streamFn`、`transformContext`、`convertToLlm` 的注入。
8. `new AgentSession({...})`。

这是第一周最重要的组装文件之一。

阶段产出：启动链和对象创建图。

---

## 阶段 2：Agent Core

### 9. `packages/agent/src/types.ts`

目的：认识 Agent Core 的最小领域模型。

不要从头背类型。按引用搜索阅读：

- `AgentMessage`
- `AgentState`
- `AgentContext`
- `AgentTool`
- `AgentEvent`
- `AgentLoopConfig`
- `StreamFn`
- Tool 前后置 Context/Result
- QueueMode 和 ToolExecutionMode

为每个类型写一句“谁产生、谁消费”。

### 10. `packages/agent/src/agent.ts`

目的：理解有状态 Agent facade。

第一遍关注类结构：

- mutable state。
- 订阅者。
- steering/follow-up 队列。
- active run 和 AbortController。
- prompt/continue/reset/abort/waitForIdle。

第二遍追踪：

```text
prompt
→ runPromptMessages
→ runWithLifecycle
→ runAgentLoop
→ processEvents
→ finishRun
```

重点理解 `processEvents()` 如何把事件归约为状态。

### 11. `packages/agent/src/agent-loop.ts`

目的：理解真正的 Agent 主循环。

分五段阅读：

1. `runAgentLoop` 和 `runAgentLoopContinue`：run 入口。
2. `runLoop`：turn、tool 和 queue 决策。
3. `streamAssistantResponse`：Context 到 LLM 的边界。
4. `executeToolCalls*`：Tool batch 策略。
5. `prepare/execute/finalize`：Tool 生命周期。

第一次不要深挖：

- 截断 JSON salvage 的提供方细节。
- 每种 error result 的文案。
- 所有泛型类型。

读完后必须能从 `while` 条件解释为什么循环继续或退出。

### 12. `packages/agent/src/stream-fn.ts`

目的：确认 Agent Core 的 Provider 解耦机制。

关注：

- 为什么默认 streamFn 是可注入的。
- 为什么 Agent Core 本身不导入 Provider catalog。
- Coding Agent 在哪里设置兼容 fallback。

### 13. `packages/agent/src/index.ts`

目的：最后查看 Agent package 对外公开的 API，而不是把 barrel file 当实现入口。

阶段产出：无 Tool 和有 Tool 两张事件时序图。

---

## 阶段 3：模型和 Provider

### 14. `packages/ai/src/index.ts`

目的：认识新的核心导出边界。

关注注释：root export 尽量 side-effect free，Provider factory 和具体 API 使用子路径导出。

### 15. `packages/ai/src/types.ts`

按主题阅读，而不是逐行阅读：

1. `Model` 和能力字段。
2. user/assistant/toolResult Message。
3. `Context` 和 Tool schema。
4. Assistant streaming event。
5. usage、cost、stopReason。
6. Provider request options。

暂时跳过：暂不使用 API 的所有特有 options。

### 16. `packages/ai/src/models.ts`

目的：掌握 Provider 容器、认证应用和请求分发。

阅读顺序：

1. `Provider` 接口。
2. `Models` 和 `MutableModels` 接口。
3. `ModelsImpl` 的 Provider map。
4. model refresh 和 availability 的职责边界。
5. `applyAuth()`。
6. `stream()` 和 `streamSimple()`。
7. `createModels()`。
8. `createProvider()` 的 API dispatch。

第一遍暂时跳过 OAuth 并发刷新细节；第二遍再研究 credential mutation 和 abort。

### 17. `packages/ai/src/providers/all.ts`

目的：理解内置 Provider catalog 如何被构造。

关注：

- `builtinProviders()`。
- `builtinModels()`。
- generated model data 与 Provider factory 的区别。

### 18. 一个 Provider factory

推荐：

- `packages/ai/src/providers/anthropic.ts`，或
- `packages/ai/src/providers/openai.ts`。

关注 Provider ID、baseUrl、auth、models、api 五个组成部分。

### 19. 对应 lazy API 和 API Adapter

例如：

- `api/anthropic-messages.lazy.ts`
- `api/anthropic-messages.ts`

第一遍只追：

```text
streamSimple
→ stream
→ request construction
→ SDK/fetch stream
→ event conversion
→ final message
```

暂时跳过所有兼容 header、缓存策略和特殊 thinking 分支。

### 20. `packages/coding-agent/src/core/model-runtime.ts`

目的：理解产品层为什么包装通用 Models。

分段关注：

- `create()` 如何组合 builtins、远程目录和配置。
- snapshot 的 all/available/configuredProviders。
- Provider recompose。
- `prepareRequest()`。
- `streamSimple()`。
- `registerProvider()` 和 `registerNativeProvider()`。

### 21. `packages/coding-agent/src/core/provider-composer.ts`

目的：理解内置 Provider、配置文件和 Extension overlay 如何组合。

第一遍只读输入输出和 dispatch 选择，不需要记住所有兼容规则。

阶段产出：LLM 分发图和新增 Provider 的变更点清单。

---

## 阶段 4：AgentSession 产品协调器

### 22. `packages/coding-agent/src/core/agent-session.ts`

这是仓库最复杂的核心文件之一。不要顺序逐行阅读。

#### 第一遍：类地图

只读：

- imports。
- `AgentSessionConfig`。
- class 字段。
- constructor。
- public getters 和 public commands。
- 私有方法名。

把字段分组：

- Agent/Session/Settings/Model 服务。
- Extension。
- Tool registry。
- prompt queues。
- compaction/retry。
- bash。
- event listeners。

#### 第二遍：事件和持久化

追踪：

- constructor 的 `agent.subscribe`。
- `_handleAgentEvent`。
- `_emitExtensionEvent`。
- `message_end` 到 SessionManager。
- `agent_settled`。

#### 第三遍：Prompt

追踪：

```text
prompt
→ streaming behavior
→ extension command/input
→ skill/template expansion
→ before_agent_start
→ agent.prompt
```

#### 第四遍：Runtime rebuild

追踪：

- `_buildRuntime`。
- base Tool definitions。
- Extension tools。
- active Tool selection。
- ExtensionRunner 创建。
- system prompt 更新。

#### 第五遍：Compaction 和 retry

只在完成 SessionManager/compaction 文件后阅读。

暂时跳过：全部 UI 渲染和不相关命令细节。

---

## 阶段 5：Session 和 Context

### 23. `packages/coding-agent/src/core/session-manager.ts`

同样采用分段阅读。

#### 第一段：数据模型

识别：

- SessionHeader。
- MessageEntry。
- Model/Thinking change。
- CompactionEntry。
- BranchSummaryEntry。
- Custom/Label/SessionInfo。

#### 第二段：纯 Context 函数

优先读：

- branch/path 构建。
- entry 到 context message 的转换。
- `buildSessionContext()`。

#### 第三段：SessionManager 状态

关注：

- fileEntries。
- byId。
- leafId。
- labels。
- persist/sessionFile/cwd。

#### 第四段：append API

确认追加写和 leaf 前进规则。

#### 第五段：tree/branch/context API

比较：

- `getEntries`
- `getBranch`
- `getTree`
- `buildContextEntries`
- `buildSessionContext`

#### 第六段：create/open/fork/list

最后阅读文件扫描、恢复和迁移路径。

### 24. `packages/coding-agent/src/core/messages.ts`

目的：理解产品自定义消息如何投影到标准 LLM Message。

重点：`convertToLlm()` 保留什么、过滤什么、转换什么。

### 25. `packages/coding-agent/src/core/compaction/compaction.ts`

阅读顺序：

1. settings/default。
2. token estimate。
3. shouldCompact。
4. prepareCompaction。
5. summary generation。
6. final compact result。

### 26. `packages/coding-agent/src/core/compaction/branch-summarization.ts`

对照 Compaction 阅读，重点是 common ancestor、离开分支内容和目标分支注入。

### 27. `packages/coding-agent/src/core/compaction/utils.ts`

最后了解消息序列化和文件操作累积。

阶段产出：Session 树、Context 投影和 Compaction 三合一图。

---

## 阶段 6：Tools

### 28. `packages/coding-agent/src/core/tools/index.ts`

目的：查看 Tool 对外类型、默认集合和 factory。

关注：

- `Tool`/ToolDefinition。
- `createCodingTools()`。
- `createReadOnlyTools()`。
- 文件写入队列包装。

### 29. `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`

目的：理解产品 ToolDefinition 如何适配为 AgentTool，以及 Extension Context 在哪里加入。

### 30. `read.ts`

推荐作为第一个具体 Tool：输入、路径解析、输出截断和图片结果相对直观。

### 31. `bash.ts`

重点观察：

- command schema。
- cwd/环境。
- child process 和 AbortSignal。
- streaming update。
- 输出累计和截断。

第一遍跳过所有平台兼容分支。

### 32. `edit.ts` 和 `write.ts`

重点观察文件变更为什么需要 mutation queue，以及 ToolResult 如何描述变更。

### 33. `grep.ts`、`find.ts`、`ls.ts`

作为扩展工具集合了解，不必逐行精读。

阶段产出：Tool 注册和执行生命周期图。

---

## 阶段 7：Extension 和资源系统

### 34. `packages/coding-agent/src/core/resource-loader.ts`

目的：理解 Extension、Skill、Prompt、Theme、AGENTS 文件和 Pi Package 的统一资源入口。

关注：

- ResourceLoader 接口。
- DefaultResourceLoader 状态。
- reload 阶段。
- global/project/explicit/package sources。
- diagnostics。

### 35. `packages/coding-agent/src/core/extensions/loader.ts`

目的：理解扩展发现和 factory 执行。

阅读：

- Extension entry 解析。
- 目录发现规则。
- jiti 加载。
- factory 注册结果。
- load error 隔离。

### 36. `packages/coding-agent/src/core/extensions/runner.ts`

目的：理解已加载扩展如何参与运行时。

按以下主题搜索阅读：

- constructor 和 runtime。
- `createContext`/`createCommandContext`。
- `emit*`。
- Tool、Command、Shortcut 查询。
- provider registration。
- resources_discover。
- invalidate。

### 37. `packages/coding-agent/src/core/extensions/types.ts`

这是大型契约文件。只在阅读某事件时查对应类型，不要从头到尾背诵。

按类别建立索引：

- startup/resource。
- session。
- agent/turn/message。
- context/provider。
- tool。
- UI/command。

### 38. Extension 示例

推荐顺序：

1. `examples/extensions/tools.ts`
2. `permission-gate.ts`
3. `protected-paths.ts`
4. `custom-provider-anthropic/`
5. `dynamic-resources/`
6. `todo.ts`
7. `plan-mode/`
8. `subagent/`

前五个用于理解核心扩展点；后面三个用于观察复杂功能如何在不修改核心的情况下构建。

阶段产出：Extension 生命周期与影响面图。

---

## 阶段 8：模式与 TUI

### 39. `packages/coding-agent/src/modes/print-mode.ts`

最简单的模式实现，先观察如何绑定 Extension、订阅 Session 和输出结果。

### 40. `packages/coding-agent/src/modes/json-event.ts`

理解 AgentSessionEvent 如何被序列化为稳定输出。

### 41. `packages/coding-agent/src/modes/rpc/rpc-mode.ts`

只追：输入 command、调用 AgentSession、输出 response/event、Extension UI request。

### 42. `packages/coding-agent/src/modes/interactive/interactive-mode.ts`

大型文件三遍法：

1. init/run/stop 和 Session rebind。
2. 用户提交与 AgentSession event。
3. UI 组件和特殊交互。

第一周跳过每个 slash command 和 selector 的细节。

### 43. `packages/tui/src/tui.ts`

理解 Component、Container、TUI、Terminal 和 focus/input 边界。

### 44. `tui-main-screen.ts` 与 `tui-alt-screen.ts`

只比较两种渲染模式和 viewport/scrollback 所有权。

### 45. `components/editor.ts` 与 `components/markdown.ts`

作为代表性组件了解，不需要阅读所有组件。

阶段产出：模式适配器图。

---

## 阶段 9：远程会话和通用 Harness

此阶段必须在当前主链之后阅读。

### 46. `packages/protocol/src/schemas.ts`

关注：

- handshake/version。
- command/result/event envelope。
- server/session snapshot。
- progress 与 authoritative snapshot 的区别。

### 47. `packages/protocol/src/framing.ts` 和 `codec.ts`

理解长度帧、CBOR、校验和大小限制；不必手工推导编码格式。

### 48. `packages/client/src/client.ts`

关注连接状态、请求关联、session acquisition 和 snapshot 更新。

### 49. `packages/client/src/session-handle.ts`

理解 shared/exclusive lease 和 detach/dispose。

### 50. `packages/server/src/server.ts`

关注 handshake、connection、request dispatch 和 snapshot publisher。

### 51. `packages/server/src/sessions.ts`

理解 live session attachment/locking；记住实际 Agent Session 由应用提供的 Service 创建。

### 52. `packages/agent/src/harness/session/`

建议顺序：

1. `types.ts`
2. `session.ts`
3. `context.ts`
4. `state.ts`
5. `memory.ts`
6. `jsonl/`

重点比较它与 coding-agent SessionManager：lane、record、operation、repository/backend 抽象。

### 53. `packages/agent/src/harness/agent-harness.ts`

最后读。先列出已实现的配置/读取 API和明确未实现的运行 API，避免把接口愿景当成完成行为。

### 54. `packages/session-backends/sqlite-node/src/sqlite/repo.ts`

只理解 Repository、writer lease、migration 和 storage 分层。SQL 细节放到第二周。

阶段产出：当前主链与演进链对照表。

---

## 阶段 10：横切能力和验证

### 55. `packages/telemetry/src/index.ts`

理解显式 TelemetryContext、span 和 schema；确认 telemetry 不应改变业务结果。

### 56. `packages/agent/src/harness/telemetry.ts`

观察 Agent/AI/Harness 如何声明自己的 telemetry vocabulary。

### 57. `packages/evals/src/pi-harness.ts`

理解 Eval 如何组装真实 AgentSession、隔离 cwd/agentDir，并收集 transcript、usage 和 artifact。

### 58. 测试文件

最后按问题选测试，不按目录扫读：

- Agent Loop：`packages/agent/test/`。
- Session：`packages/coding-agent/test/session-manager/`。
- 产品回归：`packages/coding-agent/test/suite/regressions/`。
- Provider：`packages/ai/test/`。
- Protocol：`packages/protocol/test/`。
- Client/Server：对应 package 的 test。

## 不推荐的起始文件

以下文件重要，但不适合作为第一入口：

- `coding-agent/src/modes/interactive/interactive-mode.ts`
- `coding-agent/src/core/extensions/types.ts`
- `ai/src/models.generated.ts`
- 任意 Provider 的完整 API Adapter
- `coding-agent/src/core/agent-session.ts` 的逐行阅读
- `agent/src/harness/agent-harness.ts`

原因不是它们质量差，而是它们需要前置架构模型。过早进入会形成“知道很多局部名词，但不知道系统如何运行”的假理解。

## 阅读完成记录

每完成一个阶段，在个人笔记中记录：

```text
[ ] 我能画出本阶段的组件关系
[ ] 我能说出每个核心对象的上游和下游
[ ] 我能指出状态写入位置
[ ] 我能指出异步/取消边界
[ ] 我能指出 Extension 或替换边界
[ ] 我有源码证据支持结论
[ ] 我列出了暂未理解的问题
```
