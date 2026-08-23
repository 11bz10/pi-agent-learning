# 核心调用链导读

本文提供按运行路径阅读源码的方法。每条链都包含入口、关键节点、状态变化、扩展点、失败边界和阅读问题。

调用链中的函数名用于定位职责，不要求第一遍记住所有私有方法。

## 调用链 1：进程启动

### 场景

用户在终端执行：

```text
pi
```

### 主路径

```text
packages/coding-agent/src/cli.ts
  → configureHttpDispatcher()
  → main(argv)

packages/coding-agent/src/main.ts
  → parseArgs()
  → resolveAppMode()
  → SettingsManager
  → createSessionManager()
  → createRuntime factory
      → createAgentSessionServices()
      → createAgentSessionFromServices()
          → createAgentSession()
  → createAgentSessionRuntime()
  → InteractiveMode / runPrintMode / runRpcMode
```

### 关键状态

| 状态 | 拥有者 | 何时确定 |
|---|---|---|
| CLI 参数 | `main` 局部变量 | 参数解析时 |
| app mode | `main` | 参数和 TTY 状态确定后 |
| cwd | Session/Runtime | session flags 和恢复逻辑后 |
| 全局/项目设置 | SettingsManager | services 创建时 |
| 模型与认证目录 | ModelRuntime | services 创建时 |
| Extension/Skill/Prompt/Theme | ResourceLoader | reload 时 |
| 会话树 | SessionManager | create/open/continue/fork 时 |
| Agent 运行态 | Agent | createAgentSession 时 |
| 产品生命周期 | AgentSession | createAgentSession 时 |

### 关键设计

`main.ts` 是 composition root，而不是业务核心。它允许 CLI 启动、SDK 组装和 Session 替换复用相同的底层创建函数。

### 扩展点

- Settings 和 project trust 影响项目资源是否加载。
- ResourceLoader 加载 Extension 和 Pi Package。
- Extension 可以注册 Provider，影响模型可用性。
- CLI flags 可以限制 Tools、模型或资源路径。

### 失败边界

- 参数和 flag 冲突：进入 Runtime 前失败。
- Session 文件或 cwd 不可用：Session 创建阶段失败。
- Extension 加载错误：作为 diagnostics 汇总，特定模式下终止。
- 无可用模型：非 Interactive 模式通常无法继续 Prompt。

### 阅读问题

1. 为什么 auth/help/list-models 等命令可以在完整 Session 运行前退出？
2. 哪些服务需要 cwd，哪些不需要？
3. Session 恢复时，model 和 thinking level 从哪里恢复？
4. Interactive 为什么可能延后模型目录的网络刷新？

---

## 调用链 2：普通 Prompt，不调用 Tool

### 场景

用户输入一个只需要文字回答的问题。

### 主路径

```text
Mode 获取用户输入
  → AgentSession.prompt(text, options)
  → 检查 Session/compaction/streaming 状态
  → Extension command 检查
  → Extension input event
  → Skill 或 Prompt Template 展开
  → Extension before_agent_start
  → Agent.prompt(messages)
  → Agent.runWithLifecycle()
  → runAgentLoop()
  → runLoop()
  → streamAssistantResponse()
  → transformContext()
  → convertToLlm()
  → ModelRuntime.streamSimple()
  → Provider.streamSimple()
  → API Adapter
  → Assistant stream events
  → Agent processEvents()
  → AgentSession subscriber
  → SessionManager.appendMessage()
  → Mode 渲染或输出
```

### 消息变化

```text
初始 Agent messages:
  [...history]

加入用户消息后:
  [...history, user]

模型完成后:
  [...history, user, assistant]
```

### 事件顺序

```text
agent_start
turn_start
message_start(user)
message_end(user)
message_start(assistant)
message_update(assistant)*
message_end(assistant)
turn_end
agent_end
agent_settled  // 产品层事件
```

### 状态写入点

| 数据 | 写入位置 | 用途 |
|---|---|---|
| streamingMessage | Agent.processEvents | UI 显示流式内容 |
| finalized assistant | Agent.state.messages | 下一轮 Context |
| Session MessageEntry | AgentSession event handler | 持久化与恢复 |
| usage/error | assistant message 与 Session stats | 统计、UI、重试判断 |

### Context 转换点

1. Session 恢复时，Session Entries 被构建为 AgentMessage。
2. 每次模型请求前，Extension `context` hook 可以转换 AgentMessage。
3. `convertToLlm` 过滤产品自定义消息并处理图片策略。
4. Agent Loop 创建 `pi-ai Context`。
5. API Adapter 再把统一 Context 转换为供应商请求。

### 失败边界

- Extension input/before_agent_start 失败。
- Context transform 失败。
- auth/model 不可用。
- Provider stream error 或 abort。
- Agent event subscriber 失败。
- 自动重试或 overflow recovery 由 AgentSession 产品层协调。

### 阅读问题

1. 用户消息是在调用 LLM 前何时进入 Agent state？
2. 流式 partial assistant 是否立即进入持久化 Session？
3. 为什么最终 Session 持久化由 `message_end` 驱动？
4. `agent_end` 和产品层 `agent_settled` 有什么差异？

---

## 调用链 3：LLM 请求分发

### 场景

Agent Loop 已经构建好统一 Context，需要调用当前模型。

### 主路径

```text
agent-loop.streamAssistantResponse()
  → streamFunction(model, context, options)

coding-agent SDK 注入的 streamFn
  → 读取 retry/timeout/transport 设置
  → 添加 attribution headers
  → Extension before_provider_headers
  → ModelRuntime.streamSimple()

ModelRuntime
  → prepareRequest()
  → 组合 auth、model config、headers、env
  → 找到有效 Provider
  → Provider.streamSimple()

Provider
  → 根据 model.api 找到 ProviderStreams
  → API Adapter.streamSimple()

API Adapter
  → 映射 simple reasoning/options
  → 构建供应商请求
  → Extension before_provider_request（由上层 option 传入）
  → 发起请求
  → Extension after_provider_response
  → 解析 stream
  → 统一 AssistantMessageEventStream
```

### 分层输入输出

| 层 | 输入 | 输出 |
|---|---|---|
| Agent Loop | AgentContext + Model | `pi-ai Context` + Stream options |
| Coding Agent SDK | 请求 options | 加入产品 retry/header/hook 的 options |
| ModelRuntime | Model + Context | 已认证、已组合 Provider 请求 |
| Provider | Model + Context | 选择具体 API 实现 |
| API Adapter | 统一 Message/Tool | 供应商 payload 和统一 stream event |

### 关键设计

Agent Core 只依赖 `StreamFn`，因此：

- 可以使用不同 Provider registry。
- 可以使用代理或测试 Provider。
- Agent Loop 不处理认证和 HTTP。
- 浏览器、Node、Bun 或远程代理可以提供不同 stream 实现。

### Provider 组成

一个 Provider 通常包括：

```text
id/name/baseUrl
+ auth methods
+ model catalog
+ stream behavior
+ optional dynamic model refresh
```

Model 则是数据：模型 ID、API 类型、能力、context window、max tokens 和 cost。

### Extension 影响点

- 注册新的 Provider。
- 给内置 Provider 添加配置 overlay。
- 修改请求 Header。
- 检查或替换最终 provider payload。
- 观察 provider response status/headers。

### 阅读问题

1. 为什么 `model.provider` 和 `model.api` 是两个不同字段？
2. 一个 Provider 是否可以包含多个 API 类型的模型？
3. 认证为什么在请求时重新解析，而不是启动时固定 API key？
4. lazy API 对启动性能和 bundle 有什么影响？
5. ModelRuntime snapshot 为什么需要与底层 Models 分开维护？

---

## 调用链 4：ToolCall 到 ToolResult

### 场景

LLM 返回一个或多个 ToolCall。

### 主路径

```text
AssistantMessage.content 包含 ToolCall
  → Agent Loop 提取 ToolCalls
  → 判断 sequential / parallel batch
  → emit tool_execution_start
  → 按 name 查找 AgentTool
  → prepareArguments（可选）
  → validateToolArguments
  → Agent.beforeToolCall
      → Extension tool_call
      → 可 block/terminate
  → AgentTool.execute
      → onUpdate 可发 partial result
  → Agent.afterToolCall
      → Extension tool_result
      → 可修改 content/details/isError/usage/terminate
  → emit tool_execution_end
  → 构建 ToolResultMessage
  → message_start/message_end(toolResult)
  → ToolResult 加入 current Context
  → turn_end
  → 下一次 LLM turn
```

### Tool 的来源

```text
内置 ToolDefinition
+ SDK customTools
+ Extension registered tools
→ AgentSession._buildRuntime
→ allow/exclude/active 策略
→ wrapper 转 AgentTool
→ agent.state.tools
```

### Tool batch 决策

| 条件 | 行为 |
|---|---|
| 全局 `toolExecution=sequential` | 整个 batch 顺序执行 |
| 任一 Tool 声明 sequential | 整个 batch 顺序执行 |
| 其他情况 | Tool 主体可并行执行 |

即使并行执行，最终 ToolResultMessage 仍按 Assistant 原始 ToolCall 顺序进入 transcript。这使 Provider Context 稳定，而完成事件仍可以及时反映真实完成顺序。

### 错误语义

- Tool 未找到：生成 error ToolResult。
- 参数校验失败：不执行 Tool，生成 error ToolResult。
- before hook block：不执行 Tool，使用 block reason。
- execute throw：Agent Loop 捕获并生成 error ToolResult。
- after hook throw：结果转为 error。
- AbortSignal：Tool 应尽快停止；循环根据状态结束。
- Assistant 因 token length 截断：Tool 参数可能不完整，整批调用被拒绝执行。

### `terminate` 的语义

Tool、before hook 或 after hook 可以返回 `terminate: true`。只有一个 batch 中所有最终 Tool 结果都要求 terminate，循环才跳过自动的下一次 LLM 调用。

### 阅读问题

1. 为什么 `tool_execution_start` 在参数校验前发出？
2. 为什么 Tool 应通过 throw 表示失败，而不是返回普通错误文本？
3. Extension 权限 gate 最适合使用哪个 hook？
4. Tool 的 partial update 是否进入最终 LLM Context？
5. 为什么并行完成顺序不能直接决定 ToolResult transcript 顺序？

---

## 调用链 5：Session 持久化和恢复

### 场景 A：保存消息

```text
Agent emits message_end
  → AgentSession._handleAgentEvent
  → 根据 message role/type 分类
  → SessionManager.appendMessage 或专用 append API
  → 创建 id、parentId、timestamp
  → 当前 leaf 前进到新 Entry
  → 追加 JSONL
```

### 场景 B：恢复 Session

```text
main/createSessionManager
  → SessionManager.open/continueRecent
  → 读取 Session Header 和 JSONL Entries
  → 构建 byId、leaf、labels 索引
  → buildSessionContext()
  → 恢复 messages/model/thinking level
  → createAgentSession()
  → agent.state.messages = restored messages
```

### Session 数据模型

```text
Session Header
Session Entry tree
  ├─ message
  ├─ model_change
  ├─ thinking_level_change
  ├─ compaction
  ├─ branch_summary
  ├─ custom/custom_message
  ├─ label
  └─ session_info
```

### 树与 leaf

每个 Entry 的 `parentId` 指向父节点。SessionManager 保存当前 `leafId`。新的 Entry 默认成为当前 leaf 的子节点并取代 leaf。

分支操作不是删除后续历史，而是把 leaf 移到较早 Entry 后继续追加。完整 JSONL 保留所有分支。

### 恢复时的三个视图

| 视图 | 内容 | 用途 |
|---|---|---|
| all entries | 所有分支和元数据 | 管理、导出、树 UI |
| current branch | root 到当前 leaf | 当前对话路径 |
| session context | compaction-aware messages + model/thinking | Agent/LLM |

### 阅读问题

1. 为什么模型切换和 thinking level 也要写入 Session？
2. 为什么 SessionManager 同时维护 fileEntries 和 byId？
3. in-memory Session 与 persisted Session 共享哪些行为？
4. Branch 操作如何保持历史可追溯？

---

## 调用链 6：Compaction

### 场景

Context 接近模型窗口，或用户手动执行 compact。

### 主路径

```text
AgentSession 检测 threshold/overflow 或收到 manual compact
  → 获取当前 branch entries
  → prepareCompaction()
      → 估算 context tokens
      → 识别已有 compaction
      → 从后向前选择近期保留内容
      → 避免非法 ToolResult 切分
      → 形成 messagesToSummarize / retained tail
  → Extension session_before_compact
      → 可 cancel
      → 可提供自定义 summary
  → 默认 generateSummary()
      → 单独 LLM 请求
  → SessionManager.appendCompaction()
  → buildSessionContext()
  → agent.state.messages 替换为压缩后的 Context
  → Extension session_compact
```

### 关键事实

- Compaction 不删除旧 Entry。
- 新增的 CompactionEntry 保存摘要、保留边界、token 信息和可选 usage/details。
- 下一次 Context 构建只投影最近有效 Compaction 加近期消息。
- 摘要生成本身也可能有 usage 和失败。
- Overflow recovery 可能在 compaction 后重试原运行。

### 与 Branch Summary 对比

| 维度 | Compaction | Branch Summary |
|---|---|---|
| 触发 | 窗口阈值、overflow、手动 | 树导航 |
| 目标 | 缩短当前分支 Context | 把离开分支的重要信息带到目标分支 |
| Entry | compaction | branch_summary |
| 旧历史 | 保留 | 保留 |

### 阅读问题

1. 为什么切分点不能随意落在 ToolResult？
2. `reserveTokens` 和 `keepRecentTokens` 分别解决什么问题？
3. repeated compaction 如何避免丢失前一次保留的上下文？
4. 为什么摘要请求需要独立 session/cache 策略？

---

## 调用链 7：Extension 发现、绑定和运行

### 场景 A：启动发现

```text
DefaultResourceLoader.reload()
  → 收集 global/project/explicit/package sources
  → project trust 过滤
  → discoverAndLoadExtensions()
  → 解析文件或 package entry
  → jiti 加载 TypeScript
  → 执行 default extension factory
  → 收集 handlers/tools/commands/flags/providers
  → 返回 LoadExtensionsResult + diagnostics
```

### 场景 B：Session 绑定

```text
AgentSession._buildRuntime()
  → 读取 ResourceLoader extensions
  → 创建 ExtensionRunner
  → 注册/刷新 Provider
  → 合并 Extension Tools
  → 构建系统提示词和 active tools
  → 保存 extensionRunnerRef
  → Mode bindExtensions() 提供 UI Context
  → session_start/resources_discover
```

### 场景 C：运行事件

Extension 可以参与：

```text
input
before_agent_start
agent/turn/message events
context
before_provider_headers
before_provider_request
after_provider_response
tool_call/tool_result
session compact/tree/switch/fork/shutdown
resources_discover
UI commands/widgets/editor/theme
```

### Context 失效

Session new/resume/fork/switch 或 reload 后，旧 ExtensionRunner 和 Context 不应继续操作新 Session。AgentSession 会 invalidate 旧 runner，要求扩展在新的 Session Context 中继续工作。

### 安全边界

Extension 是进程内 TypeScript 模块：

- 可以访问 Node API 和文件系统。
- 可以执行任意代码。
- 可以读取当前 Session 和 Tool 数据。
- project-local Extension 受 project trust 控制。

Extension API 提供的是生命周期和组合边界，不是操作系统级沙箱。

### 阅读问题

1. ResourceLoader 与 Extension Loader 为什么不是同一个概念？
2. Loader 的输出为什么还需要 Runner？
3. Provider 注册为何必须在模型选择前生效？
4. resources_discover 为什么发生在 Extension factory 之后？
5. Extension 的持久状态应该放在普通闭包、Session Entry，还是外部存储？

---

## 调用链 8：模式层和输出

### 通用结构

```text
输入源
  → Mode adapter
  → AgentSession command
  → AgentSession events
  → Mode adapter
  → 输出目标
```

### Interactive

```text
Editor/TUI input
→ InteractiveMode
→ AgentSession.prompt
→ AgentSession event subscription
→ Assistant/Tool/Bash components
→ TUI differential rendering
```

### Print

```text
CLI args/stdin
→ runPrintMode
→ AgentSession.prompt
→ 等待 settled
→ 输出最终 assistant text 或 JSON events
```

### RPC

```text
stdin JSONL command
→ rpc-mode command dispatcher
→ AgentSession/Runtime API
→ response + AgentSession events
→ stdout JSONL
```

### 关键设计

模式层不拥有 Agent 决策逻辑，只适配交互方式。新增 Web UI 时可以复用 AgentSession/Runtime，将浏览器或服务器作为新的输入输出适配器。

---

## 调用链 9：实验性远程会话

### 主路径

```text
PiClient
  → ByteTransport
  → length-prefixed CBOR Protocol
  → PiServer listener
  → PiServer request/session manager
  → application-provided PiServerService
  → acquired session runtime
```

### 状态规则

- Client hello 必须先完成协议版本协商。
- Request/response 通过 ID 关联。
- ServerSnapshot 和 SessionSnapshot 是权威状态。
- Progress event 是低延迟提示，不应独立归约成权威状态。
- Session lease 控制 shared/exclusive ownership。
- Transport 负责认证后，才把字节交给 Pi Protocol。

### 与 Coding Agent RPC 的区别

| 维度 | Coding Agent RPC | Protocol/Client/Server |
|---|---|---|
| 位置 | `coding-agent/modes/rpc` | 独立 packages |
| 传输 | stdin/stdout JSONL | transport-neutral framed CBOR |
| 目标 | 控制当前本地进程 | 远程、多连接 session 服务 |
| Session ownership | 当前 Runtime | attach/lease/lock |
| 稳定性 | 产品模式 | 实验性 API |

### 阅读问题

1. 为什么 Protocol 不直接复用 pi-ai 的领域对象？
2. 为什么 Server 不内置完整 Coding Agent Service？
3. Transport 认证为什么在协议字节之前完成？
4. shared/exclusive lease 解决什么并发问题？

---

## 调用链 10：通用 AgentHarness 演进线

### 设计方向

```text
AgentHarness
  → AgentLane
  → durable Session
  → entries + operation records + lane pointers
  → repository/backend
      ├─ InMemory
      ├─ JSONL
      └─ SQLite
```

它试图把以下能力统一到通用运行时：

- 多 lane。
- durable operation 和恢复。
- pending/steer/follow-up/nextRun queue。
- compaction/navigation operation。
- backend-neutral Session storage。
- deferred response。
- telemetry 和 action 驱动。

### 当前阅读警告

接口设计不等于运行实现。阅读 `AgentHarness` 时应建立两张表：

| 已实现 | 尚未实现 |
|---|---|
| 配置和资源 getters/setters | prompt/skill/template |
| model/thinking/tools 状态 | compact/navigation/resume |
| create 的空 Session 路径 | queue/abort/watch/drive |
| close | 多 lane 操作 |

具体状态应以当前文件为准。不要使用未来接口推断当前 CLI 行为。

## 完整端到端练习链

选择以下 Prompt：

```text
读取 package.json，并告诉我这个项目有哪些 workspace。
```

自己完成以下记录：

| 序号 | 层 | 对象/函数 | 输入 | 输出 | 状态变化 | 可能失败 |
|---:|---|---|---|---|---|---|
| 1 | Mode |  |  |  |  |  |
| 2 | Product |  |  |  |  |  |
| 3 | Agent |  |  |  |  |  |
| 4 | Model |  |  |  |  |  |
| 5 | Provider |  |  |  |  |  |
| 6 | Tool |  |  |  |  |  |
| 7 | Session |  |  |  |  |  |
| 8 | Presentation |  |  |  |  |  |

必须覆盖两次 LLM 调用：第一次产生 read ToolCall，第二次消费 ToolResult 并生成最终回答。
