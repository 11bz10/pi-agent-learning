# 七天源码学习计划

本计划以每天约 3 小时为基准。每天都有明确的阅读范围、必须回答的问题、动手任务和验收标准。不要因为提前读完文件就跳过当天产出；架构学习的关键是能否重建关系，而不是阅读速度。

## 每日固定流程

每一天都执行以下四个阶段：

### 阶段 A：回顾，30 分钟

1. 不看笔记画出当前掌握的架构图。
2. 写出昨天仍不确定的三个问题。
3. 用一句话描述今天要补齐的架构缺口。

### 阶段 B：主阅读，90 分钟

1. 先看文件的 imports、exports 和公开方法。
2. 沿当天指定调用链阅读。
3. 在调用链上标记状态读写点、异步边界和扩展点。
4. 暂时跳过与当天目标无关的实现细节。

### 阶段 C：结构化输出，40 分钟

至少完成一种输出：

- 调用链图。
- 组件职责表。
- 状态所有权表。
- 事件时序图。
- 设计取舍记录。

### 阶段 D：验收，20 分钟

1. 合上源码口述当天主题。
2. 回答当天的必答问题。
3. 对不确定结论回到源码找直接证据。

---

## Day 1：建立仓库地图和启动链

### 当天目标

理解仓库为什么拆成多个 package，找到真正的产品入口，并追踪一次 CLI 启动如何创建运行时。

当天不要研究 Agent Loop、Provider 协议和 Tool 实现。

### 阅读范围

按顺序阅读：

1. 根目录 `README.md`
2. 根目录 `package.json`
3. 各 package 的 `package.json`
4. `packages/coding-agent/src/cli.ts`
5. `packages/coding-agent/src/main.ts`
6. `packages/coding-agent/src/core/agent-session-runtime.ts`
7. `packages/coding-agent/src/core/agent-session-services.ts`
8. `packages/coding-agent/src/core/sdk.ts` 的公开接口和创建流程
9. `packages/coding-agent/src/modes/index.ts`

### 阅读步骤

#### 1. 从 workspace 依赖建立层级

记录内部依赖：

```text
coding-agent → agent-core / ai / client / protocol / tui
agent-core   → ai / telemetry
ai           → telemetry
client       → protocol
server       → protocol / ai
sqlite       → agent-core / ai
```

不要只记录“依赖谁”，还要判断依赖方向为什么不能反过来。例如 `agent-core` 不能依赖 `coding-agent`，否则通用运行时会被 CLI、TUI 和文件系统策略污染。

#### 2. 区分入口和编排器

阅读 `cli.ts` 时只回答：

- 它设置了哪些进程级环境？
- 它为什么很薄？
- 它把实际工作交给谁？

阅读 `main.ts` 时先列出大阶段，不要逐行阅读：

```text
参数与模式
→ 配置和认证命令
→ SessionManager
→ Runtime 服务
→ AgentSession
→ 初始输入
→ 模式启动
```

#### 3. 理解 Runtime 和 Session 的关系

`AgentSessionRuntime` 不是 Agent 主循环。它拥有“当前 Session + 与 cwd 绑定的服务”，负责 new/resume/fork/switch 等 Session 替换操作。

`AgentSessionServices` 则是创建 Session 所需的基础服务集合，重点关注：

- SettingsManager
- ModelRuntime
- ResourceLoader
- diagnostics

#### 4. 识别运行模式

建立表格：

| 模式 | 输入来源 | 输出形式 | 是否使用 TUI |
|---|---|---|---|
| Interactive | 终端编辑器 | 流式终端 UI | 是 |
| Print | CLI 参数或 stdin | 最终文本 | 否 |
| JSON | CLI 参数或 stdin | JSON 事件 | 否 |
| RPC | stdin JSONL | stdout JSONL | 否 |
| SDK | 应用代码 | 事件/API | 由宿主决定 |

### 当天必须回答

1. `cli.ts` 和 `main.ts` 的职责差异是什么？
2. 为什么 `AgentSessionRuntime` 需要支持 Session 替换？
3. `createAgentSessionServices()` 和 `createAgentSession()` 为什么分开？
4. Interactive、Print 和 RPC 最终是否共享同一个 AgentSession？
5. 哪些 package 属于当前 CLI 必经路径？哪些属于实验性远程路径？

### 当天产出

创建一张启动时序图，至少包含：

```text
process
→ cli
→ main
→ SessionManager
→ services
→ AgentSession
→ mode
```

再创建一个 package 职责表，每个 package 只允许写一句话，强迫自己抓住边界。

### 验收标准

- 能在 3 分钟内从 `pi` 命令讲到 `AgentSession` 创建完成。
- 不再把 `main.ts` 误认为 Agent 主循环。
- 能区分当前 CLI 主链与实验性远程/Harness 代码。

---

## Day 2：掌握 Agent 状态和主循环

### 当天目标

理解一次 Agent 运行如何从用户消息开始，经过 LLM、Tool 和队列，最终结束。

### 阅读范围

1. `packages/agent/src/types.ts`
2. `packages/agent/src/agent.ts`
3. `packages/agent/src/agent-loop.ts`
4. `packages/agent/src/stream-fn.ts`
5. `packages/agent/README.md` 中 Message Flow、Event Flow、Tools 和 Steering 部分

### 阅读步骤

#### 1. 先建立状态所有权表

在 `Agent` 中找到并分类：

| 状态类别 | 例子 | 生命周期 |
|---|---|---|
| 配置状态 | model、systemPrompt、tools | 可跨多轮存在 |
| transcript | messages | Session 运行期间 |
| 运行状态 | isStreaming、streamingMessage | 单次 run |
| Tool 状态 | pendingToolCalls | 单次 Tool batch |
| 队列状态 | steering、follow-up | 跨 turn，通常不跨 Session |

#### 2. 阅读 `Agent.prompt()` 到循环入口

追踪：

```text
prompt()
→ normalizePromptInput()
→ runPromptMessages()
→ runWithLifecycle()
→ runAgentLoop()
```

标记 AbortController 在哪里创建、何时释放，以及 `waitForIdle()` 为什么不仅等待模型流。

#### 3. 阅读双层循环

在 `runLoop()` 中画出：

- 外层循环：Agent 本来要停止时，检查 follow-up。
- 内层循环：处理 ToolCall 和 steering。
- 单个 turn：一次 LLM 响应，加上该响应触发的 Tool 执行。

必须准确区分：

- run：从 `agent_start` 到 `agent_end`。
- turn：从 `turn_start` 到 `turn_end`。
- message：user、assistant 或 toolResult 的生命周期。

#### 4. 阅读流式事件

整理事件顺序：

```text
agent_start
turn_start
message_start(user)
message_end(user)
message_start(assistant)
message_update*
message_end(assistant)
tool_execution_* 可选
turn_end
agent_end
```

思考为什么 `Agent` 订阅者会被 await，以及这对 Session 持久化和 Tool 执行顺序有什么意义。

#### 5. 阅读 Tool batch 策略

只理解高层决策：

- 全局可以选择 parallel 或 sequential。
- 某个 Tool 可以强制整个 batch sequential。
- Tool 参数先准备和校验。
- before hook 可以阻止执行。
- after hook 可以修改结果。
- ToolResult 保持原 ToolCall 顺序进入 transcript。

### 当天必须回答

1. `Agent` 和 `agent-loop.ts` 为什么分开？
2. run、turn、message 三者是什么包含关系？
3. ToolResult 为什么必须重新进入 Context？
4. steering 和 follow-up 在什么时机被消费？
5. `agent_end` 已发出后，为什么 `Agent` 可能还没有完全 idle？
6. 并行 Tool 的完成事件顺序和 transcript 顺序是否一定相同？

### 当天产出

画两张时序图：

1. 不调用 Tool 的 Prompt。
2. 调用两个 Tool 后再回复的 Prompt。

每张图必须标出 Agent state 何时写入 assistant 和 toolResult。

### 验收标准

- 能完整口述双层循环。
- 能根据事件名判断当前处于 run、turn、message 或 Tool 哪个阶段。
- 能准确指出真正的 Agent 主循环文件。

---

## Day 3：掌握 LLM 抽象和 Provider 分发

### 当天目标

理解 Agent Core 如何在不知道 OpenAI、Anthropic 等协议细节的情况下调用任意模型。

### 阅读范围

1. `packages/ai/src/index.ts`
2. `packages/ai/src/types.ts` 中 Model、Context、Message、Stream 相关类型
3. `packages/ai/src/models.ts`
4. `packages/ai/src/providers/all.ts`
5. `packages/ai/src/providers/anthropic.ts` 或 `openai.ts`
6. `packages/ai/src/api/` 中与所选 Provider 对应的一个 API Adapter
7. `packages/coding-agent/src/core/model-runtime.ts`
8. `packages/coding-agent/src/core/provider-composer.ts`
9. `packages/coding-agent/src/core/sdk.ts` 中 streamFn 的注入

### 阅读步骤

#### 1. 建立四层模型

```text
Model
  描述一个具体模型及能力

Provider
  拥有模型集合、认证和 stream 行为

Models / ModelRuntime
  管理多个 Provider 并完成请求分发

API Adapter
  把统一 Context 转成具体供应商协议并解析流
```

#### 2. 追踪一次 `streamSimple`

从 Agent Loop 的 `streamFunction` 开始，依次记录：

```text
Agent Loop
→ SDK 注入的 streamFn
→ ModelRuntime.streamSimple
→ Provider.streamSimple
→ API Adapter.streamSimple
→ API Adapter.stream
→ SDK/fetch
```

在每层旁边写出它新增的职责：timeout、retry、header、auth、payload transform、协议转换等。

#### 3. 理解为什么存在 `ModelRuntime`

`pi-ai Models` 是通用 Provider 容器，而 Coding Agent 还需要：

- 文件形式的 auth/models 配置。
- Remote catalog。
- Extension Provider。
- Provider overlay/composition。
- UI 使用的同步可用模型快照。
- Coding Agent 自身 Header attribution。

因此 `ModelRuntime` 是产品层适配器，不是对 `pi-ai` 的简单重复。

#### 4. 只深入一个 Provider

第一周只选择一个 Provider，回答：

- Provider factory 提供哪些元数据？
- 认证如何声明？
- Model 列表从哪里来？
- lazy API 如何加载？
- 统一 Message 如何转成供应商请求？
- 流式事件如何转回统一 AssistantMessageEvent？

不要在同一天比较所有 Provider。

### 当天必须回答

1. Model、Provider 和 API Adapter 分别是什么？
2. Agent Core 为什么接收 `streamFn` 而不直接导入 Provider？
3. `Models.streamSimple()` 在分发前做什么？
4. `ModelRuntime` 比通用 `Models` 多负责什么？
5. Extension 如何添加或覆盖 Provider？
6. 认证、Header 和 payload 分别在哪些层被处理？

### 当天产出

画一张 LLM 请求分层图。每条边都写明输入和输出，例如：

```text
AgentContext
→ pi-ai Context
→ Provider-specific request
→ AssistantMessageEventStream
→ AssistantMessage
```

### 验收标准

- 能解释增加新 Provider 通常需要修改哪些位置。
- 能说明 `streamSimple` 为什么是 Agent 和模型层之间的重要边界。
- 不再把 Provider catalog 和实际 HTTP API Adapter 混为一谈。

---

## Day 4：掌握 Session、Context 和 Compaction

### 当天目标

理解“完整会话历史”和“当前发给 LLM 的 Context”不是同一个东西，并掌握它们之间的转换。

### 阅读范围

1. `packages/coding-agent/docs/sessions.md`
2. `packages/coding-agent/docs/session-format.md`
3. `packages/coding-agent/docs/compaction.md`
4. `packages/coding-agent/src/core/session-manager.ts`
5. `packages/coding-agent/src/core/messages.ts`
6. `packages/coding-agent/src/core/compaction/compaction.ts`
7. `packages/coding-agent/src/core/compaction/branch-summarization.ts`
8. `packages/coding-agent/src/core/agent-session.ts` 中持久化、恢复和 compaction 相关路径

### 阅读步骤

#### 1. 先画 Session 树

使用一个简单例子：

```text
A user
└─ B assistant
   ├─ C user
   │  └─ D assistant   当前 leaf
   └─ E user
      └─ F assistant
```

回答：

- JSONL 中是否删除了另一条分支？
- leaf 如何决定当前 Context？
- `getEntries()`、`getBranch()`、`getTree()` 的语义分别是什么？

#### 2. 区分持久化 Entry 和 LLM Message

Session Entry 不只包含 LLM Message，还包含：

- model change
- thinking level change
- compaction
- branch summary
- active tools 或扩展数据
- label/session info

`buildSessionContext()` 的任务是把这些会话事实投影成当前 LLM 所需的消息和运行配置。

#### 3. 追踪消息保存

从 `Agent` 的 `message_end` 事件开始：

```text
Agent event
→ AgentSession._handleAgentEvent
→ SessionManager.appendMessage
→ JSONL
```

理解为什么保存发生在 AgentSession，而不是 Agent Core。

#### 4. 追踪 Context 构建

```text
完整 Session Entries
→ 当前 leaf 到 root 的 branch
→ 应用最近 Compaction
→ Session Entry 转 AgentMessage
→ Extension context hook
→ convertToLlm
→ pi-ai Context
```

#### 5. 阅读 Compaction

只掌握以下高层步骤：

1. 判断是否接近 context window。
2. 找到保留近期消息的切分点。
3. 对旧内容生成结构化摘要。
4. 追加 CompactionEntry，而不是修改或删除旧 Entry。
5. 重新构建 Context，使 LLM 看到摘要加近期消息。

然后比较 Branch Summary：它用于切换分支时保留离开分支的重要上下文。

### Memory 的准确结论

本项目当前没有独立的向量长期 Memory 主模块。它的记忆机制主要是：

- JSONL Session：完整、持久化的事件/消息树。
- 当前分支：工作中的对话历史。
- Compaction：受限 Context Window 下的摘要记忆。
- Branch Summary：跨分支的上下文迁移。
- 通用 Session Backend：InMemory、JSONL 和 SQLite。

不要把 SQLite FTS 搜索直接等同于语义向量记忆。

### 当天必须回答

1. Session Entry 和 AgentMessage 有什么区别？
2. 为什么 Session 采用追加式树，而不是覆盖式消息数组？
3. leaf 如何影响 LLM Context？
4. Compaction 后旧消息是否仍存在？
5. Compaction Summary 和 Branch Summary 的触发场景有什么区别？
6. Context 在进入 Provider 前经历哪几次转换？
7. 为什么 Session 持久化不应该进入 Agent Core？

### 当天产出

画一张“Session 全量历史 → LLM 有效 Context”的转换图，并使用至少一个分支和一次 Compaction 示例。

### 验收标准

- 能准确解释“会话历史不等于 Context”。
- 能说明 Compaction 为什么是追加 Entry，而不是破坏历史。
- 能指出当前 CLI Session 与通用 Harness Session 的不同实现位置。

---

## Day 5：掌握 Tools 和 Extension 机制

### 当天目标

理解 Tool 如何从定义进入 Agent，以及 Extension 如何影响系统各阶段。

### 阅读范围

1. `packages/coding-agent/src/core/tools/index.ts`
2. `packages/coding-agent/src/core/tools/tool-definition-wrapper.ts`
3. `read.ts`、`bash.ts`、`edit.ts`、`write.ts` 各选入口和 execute 路径
4. `packages/agent/src/agent-loop.ts` 的 Tool 执行部分
5. `packages/coding-agent/docs/extensions.md`
6. `packages/coding-agent/src/core/resource-loader.ts`
7. `packages/coding-agent/src/core/extensions/loader.ts`
8. `packages/coding-agent/src/core/extensions/runner.ts`
9. `packages/coding-agent/src/core/extensions/types.ts` 按事件查阅
10. `packages/coding-agent/examples/extensions/` 中 3 至 5 个示例

### 阅读步骤

#### 1. 区分 Tool Definition 和 AgentTool

记录两者分别服务谁：

- Tool Definition 可能包含系统提示词贡献、UI 渲染、Extension Context 等产品信息。
- AgentTool 是 Agent Core 执行所需的最小运行接口。
- wrapper 负责把产品级 Tool 适配为 Agent Core Tool。

#### 2. 追踪 Tool 注册

```text
内置 Tool factories
 + Extension registered tools
 + SDK custom tools
→ AgentSession._buildRuntime
→ 有效 Tool 集合
→ agent.state.tools
```

同时记录 tools allowlist、excludeTools、noTools 和 active tools 对最终集合的影响。

#### 3. 追踪 Tool 执行

```text
Assistant ToolCall
→ 查找 tool.name
→ prepareArguments
→ schema validation
→ Extension tool_call
→ execute
→ streaming update
→ Extension tool_result
→ ToolResultMessage
```

#### 4. 建立 Extension 生命周期地图

把 Extension 分为四个阶段：

1. 发现：全局、项目、本地参数和 Pi Package。
2. 加载：执行 TypeScript factory，收集注册项。
3. 绑定：AgentSession 创建 ExtensionRunner 和 Context。
4. 运行：分发 input、context、provider、tool、session、UI 等事件。

#### 5. 精读示例

建议选择：

- `permission-gate.ts`：Tool 前置阻断。
- `tools.ts`：自定义 Tool。
- `custom-provider-*`：Provider 注册。
- `dynamic-resources`：动态 Skill/Prompt/Theme。
- `session-name.ts` 或 `todo.ts`：Session 持久化状态。

每个示例只回答：注册了什么、监听了什么、状态放在哪里、影响了哪条主链。

### 当天必须回答

1. Tool 从 factory 到 `agent.state.tools` 经过哪些步骤？
2. 参数验证在 Tool 执行前还是执行后？
3. Extension 如何阻止 Tool？如何修改 ToolResult？
4. ResourceLoader、Extension Loader 和 ExtensionRunner 的差异是什么？
5. Extension 如何注册 Provider？
6. Session 切换或 reload 后，为什么旧 Extension Context 必须失效？
7. Extension 的安全边界是什么？

### 当天产出

画一张 Extension 影响面图，至少连接：

```text
Input / Context / Provider / Tool / Session / UI / Resources
```

### 验收标准

- 能说明 Tool 定义、注册、执行和结果回传的完整链。
- 能说清 Loader 与 Runner 的区别。
- 能指出 Extension 在主循环之外如何影响主循环。

---

## Day 6：掌握模式、TUI 和远程架构

### 当天目标

理解同一个 AgentSession 如何被不同前端模式驱动，以及远程 Client/Server 架构为什么与本地 RPC 不同。

### 阅读范围

1. `packages/coding-agent/src/modes/print-mode.ts`
2. `packages/coding-agent/src/modes/json-event.ts`
3. `packages/coding-agent/src/modes/rpc/rpc-mode.ts`
4. `packages/coding-agent/src/modes/interactive/interactive-mode.ts` 的初始化、订阅和输入入口
5. `packages/tui/src/index.ts`
6. `packages/tui/src/tui.ts`
7. `packages/tui/src/tui-main-screen.ts` 与 `tui-alt-screen.ts` 的职责
8. `packages/protocol/README.md` 和 `src/schemas.ts`
9. `packages/client/README.md` 和 `src/client.ts`
10. `packages/server/README.md` 和 `src/server.ts`

### 阅读步骤

#### 1. 比较本地模式

建立统一抽象：

```text
输入适配器
→ AgentSession API
→ AgentSession events
→ 输出适配器
```

Print、JSON、RPC 和 Interactive 的主要差异在输入输出与 UI 能力，而不是 Agent 主循环。

#### 2. 阅读 Interactive Mode 的方法

`interactive-mode.ts` 很大。第一遍只找：

- 构造和 init/run。
- AgentSession 订阅。
- 用户提交入口。
- Agent event 到组件更新的映射。
- Session rebind。
- Extension UI Context。

跳过具体选择器、主题、动画和终端细节。

#### 3. 理解 TUI 边界

`packages/tui` 是通用终端 UI 库，不应该知道 Agent、Provider 或 Session。它提供：

- Terminal 抽象。
- Component 和布局。
- 输入、键盘和 Editor。
- 差分渲染。
- Markdown 与图片展示。

Coding Agent 的 interactive components 才负责把 Agent 事件渲染成产品 UI。

#### 4. 区分 RPC 与远程 Protocol

本地 RPC：

- 位于 `coding-agent/src/modes/rpc`。
- 通过 stdin/stdout JSONL 控制当前进程内的 AgentSession。
- 主要用于进程嵌入。

实验性远程 Protocol：

- `protocol/client/server` 独立 packages。
- 使用长度前缀 CBOR。
- 支持连接、握手、快照、session attach/lease。
- Server 需要应用提供 `PiServerService`，本身不是完整 Coding Agent 服务。

### 当天必须回答

1. Interactive、Print、JSON 和 RPC 是否使用不同 Agent Loop？
2. 为什么 TUI package 不应该导入 AgentSession？
3. Coding Agent interactive components 和通用 TUI components 有什么区别？
4. 本地 RPC 与远程 protocol/client/server 有什么区别？
5. 为什么远程快照被视为 authoritative，而 progress 只是临时提示？
6. Server 为什么要求应用提供 Service 实现？

### 当天产出

画一张“多个前端，共享一个 AgentSession”的图；再画一张远程 Client/Protocol/Server 边界图。

### 验收标准

- 不会把 TUI 当成 Agent Runtime。
- 能准确区分 RPC 与实验性远程服务。
- 能解释为什么模式层可以替换而 Agent Loop 不变。

---

## Day 7：整合、验证和架构复述

### 当天目标

把前六天的局部知识整合成完整架构模型，并通过一次小型实践验证理解。

### 阅读范围

1. 回看本目录全部文档和个人笔记。
2. `packages/telemetry/README.md` 与 `src/index.ts`
3. `packages/evals/src/pi-harness.ts`
4. `packages/agent/src/harness/agent-harness.ts`
5. `packages/agent/src/harness/session/`
6. `packages/session-backends/sqlite-node/README.md`
7. 与前六天不确定结论对应的测试文件。

### 阅读步骤

#### 1. 认识横切模块

Telemetry 不拥有业务状态，它提供显式传递的 span/context 契约。确认它如何被 `ai` 和 `agent` 使用，而不改变业务行为。

Evals 通过真实 `AgentSession` 组装端到端 Harness，用于验证模型在具体任务上的表现。它与普通单元测试的目标不同。

#### 2. 审阅通用 Harness 演进线

只回答：

- 它希望统一哪些能力？
- Session lane、operation record、resume 和 backend 抽象解决什么问题？
- 哪些方法尚未实现？
- 为什么它不能被当作当前 CLI 的完成实现？

#### 3. 做一次完整追踪

选择一个场景：

```text
用户要求读取一个文件并总结
```

从 Interactive/Print 输入开始，沿以下路径记录所有关键对象：

```text
Mode
→ AgentSession
→ Agent
→ Agent Loop
→ ModelRuntime
→ Provider
→ LLM ToolCall
→ read Tool
→ ToolResult
→ 第二次 LLM
→ Session persistence
→ UI output
```

每个节点至少写出：输入、输出、拥有者和失败时由谁处理。

#### 4. 做一个最小实践

二选一：

1. 不改源码，只使用现有 Extension 示例，跟踪它如何注册 Tool 并接入 AgentSession。
2. 写一份伪代码 Extension 设计：注册 `architecture_note` Tool，把结构化笔记写入 Session Custom Entry。

本周目标是架构理解，不要求提交功能代码。

#### 5. 完成十五分钟架构讲解

讲解结构：

1. 1 分钟：项目解决什么问题。
2. 2 分钟：package 分层。
3. 3 分钟：启动与 Prompt 主链。
4. 3 分钟：LLM 与 Tool 循环。
5. 2 分钟：Session、Context 和 Compaction。
6. 2 分钟：Extension。
7. 1 分钟：模式和远程架构。
8. 1 分钟：当前主链与未来演进。

### 当天必须回答

1. 如果新增一个 Tool，应主要修改哪一层？
2. 如果新增一个 Provider，应主要修改哪一层？
3. 如果改变 Session 文件格式，应避免影响哪些层？
4. 如果增加 Web UI，哪些核心层可以复用？
5. 如果 Tool 权限需要统一治理，最合理的拦截边界在哪里？
6. 当前架构最大的扩展优势和复杂性来源分别是什么？
7. 通用 AgentHarness 演进线与当前产品主线可能如何收敛？

### 当天产出

必须完成：

- 一张最终架构图。
- 一张完整 Prompt + Tool 时序图。
- 一张状态所有权表。
- 一份 15 分钟讲解提纲。
- 一份“仍需第二周深入”的问题清单。

### 最终验收标准

- 不看源码可以重建完整主链。
- 任意指出一个核心类，都能说出上游、下游和状态所有权。
- 能根据需求变化定位 package，而不是全仓库搜索后碰运气。
- 能明确区分已运行的产品架构和仍在演进的架构。

---

## 时间不足时的最小路线

如果一周中断，只保留以下顺序：

1. Day 1 启动链。
2. Day 2 Agent Loop。
3. Day 3 LLM 分发。
4. Day 4 Session/Context。
5. Day 5 Tool/Extension。

Day 6 和 Day 7 的远程架构、TUI、Telemetry、Evals 和通用 Harness 可以延后，但必须在笔记中明确标记为未学习，不能假设已经掌握。

## 第二周候选主题

第一周完成后，再选择以下方向深入：

- Provider 协议转换和流式事件解析。
- Tool 并发、取消、输出截断和文件写入队列。
- 自动重试、Context Overflow 恢复和 Compaction 边界。
- Extension Context 失效、Session 替换和 UI 能力差异。
- TUI 差分渲染、终端协议和布局系统。
- Protocol 安全、帧限制、连接状态和 Session Lease。
- 通用 AgentHarness、durable operation 和 SQLite backend。
- Eval Harness、judge 和 Agent 能力回归评测。
