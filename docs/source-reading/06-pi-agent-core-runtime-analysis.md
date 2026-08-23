# pi-agent-core Runtime 源码分析

本文集中分析 <code>packages/agent</code>，回答以下八个问题：

1. Agent Runtime 负责什么？
2. Agent 状态保存在哪里？
3. Message 如何流转？
4. Tool 如何注册？
5. Tool 如何调用？
6. 如何处理中断？
7. 如何处理多轮对话？
8. 如何恢复 session？

源码基线：

- Git commit：<code>a69bef789bc95abf0acee16f7b4660b70b650bb9</code>
- npm package：<code>@earendil-works/pi-agent-core@0.84.2</code>
- 核心入口：<code>packages/agent/src/index.ts</code>

## 0. 先给结论

| 问题 | 结论 |
|---|---|
| Agent Runtime 负责什么 | <code>Agent</code> 管理一次运行的内存状态、事件、队列和生命周期；<code>agent-loop</code> 负责调用模型、解析 ToolCall、执行工具并决定是否继续下一 turn。 |
| Agent 状态保存在哪里 | 基础 <code>Agent</code> 的状态只保存在当前进程内的 <code>MutableAgentState</code>、<code>activeRun</code> 和两个消息队列中，不自动落盘。 |
| Message 如何流转 | Prompt 进入 <code>Agent</code>，经 <code>transformContext → convertToLlm → streamFn</code> 发给模型；流式响应被转成 AgentEvent，最终消息在 <code>message_end</code> 时写入 <code>agent.state.messages</code>。 |
| Tool 如何注册 | 没有全局 Tool Registry。调用方把 <code>AgentTool[]</code> 放进 <code>initialState.tools</code> 或赋给 <code>agent.state.tools</code>，运行开始时复制到 <code>AgentContext</code>。 |
| Tool 如何调用 | 模型返回 ToolCall 后，runtime 按名称查找工具，准备并校验参数，执行 before hook、<code>tool.execute()</code>、after hook，再构造 ToolResultMessage 回注上下文。 |
| 如何处理中断 | 硬中断使用每次 run 独立的 <code>AbortController</code>，属于协作式取消；软中断使用 steering queue，在当前 assistant turn 和工具批次完成后注入下一 turn。 |
| 如何处理多轮对话 | 同一 run 内由工具回路、steering 和 follow-up 产生多个 turn；不同用户轮次通过下一次 <code>prompt()</code> 复用 <code>state.messages</code>。并发调用 <code>prompt()</code> 会被拒绝。 |
| 如何恢复 session | 基础 <code>Agent</code> 没有持久化或自动恢复 API。宿主必须先加载消息、模型和 thinking level，再通过 <code>initialState</code> 或 <code>state.messages</code> 恢复。包内 Harness session 后端可以重开 JSONL，但 <code>AgentHarness.create.restore</code> 和 <code>resume()</code> 目前仍未实现。 |

最重要的边界是：

> <code>Agent</code> 是内存运行时，不是 session 数据库；<code>sessionId</code> 是转发给 Provider 的标识，不会让 Agent 自动保存或恢复对话。

## 1. 包内有两条不同成熟度的架构线

<code>pi-agent-core</code> 目前同时包含两条线。

### 1.1 当前可工作的核心运行线

~~~text
Agent
  → runAgentLoop / runAgentLoopContinue
  → streamAssistantResponse
  → 注入的 StreamFn
  → pi-ai Provider
  → AssistantMessage / ToolCall
  → executeToolCalls
  → AgentTool.execute
  → ToolResultMessage
  → 下一次模型调用或 agent_end
~~~

关键文件：

| 文件 | 职责 |
|---|---|
| <code>packages/agent/src/agent.ts</code> | 有状态包装器、运行生命周期、事件归约、abort、steering/follow-up 队列 |
| <code>packages/agent/src/agent-loop.ts</code> | 模型调用循环、消息转换、工具执行和继续/终止判断 |
| <code>packages/agent/src/types.ts</code> | AgentState、AgentContext、AgentTool、AgentEvent 和 loop hooks |
| <code>packages/agent/src/stream-fn.ts</code> | Provider 无关的 StreamFn 注入边界 |

### 1.2 通用 Harness/session 演进线

~~~text
AgentHarness API
  → durable Session / SessionRepo
  → InMemory 或 JSONL backend
  → record log
  → reduceLaneState 恢复状态
~~~

关键事实：

- <code>Session</code>、<code>SessionState</code>、<code>JsonlSessionRepo</code> 和恢复 reducer 已有完整实现。
- <code>AgentHarness</code> 目前主要是 API scaffold。
- <code>AgentHarness.create()</code> 一旦发现已有 record，会抛出 <code>HarnessNotImplemented("create.restore")</code>。
- <code>prompt()</code>、<code>abort()</code>、<code>resume()</code>、<code>steer()</code> 等运行方法也仍返回未实现错误。

因此本文在解释实际运行时，以 <code>Agent + agent-loop</code> 为主；在解释持久化数据格式和未来 crash recovery 时，再单独分析 Harness session。

## 2. Agent Runtime 负责什么？

### 2.1 Agent 与 agent-loop 的职责拆分

<code>Agent</code> 类定义在 <code>packages/agent/src/agent.ts:173</code>。它是面向宿主应用的有状态外壳，负责：

1. 保存系统提示词、模型、thinking level、消息和工具。
2. 保证同一 Agent 同时最多只有一个 active run。
3. 为每次 run 创建独立的 AbortController。
4. 把 steering 和 follow-up 消息保存在两个 FIFO 队列。
5. 调用 <code>runAgentLoop()</code> 或 <code>runAgentLoopContinue()</code>。
6. 把 loop 事件归约成公开的 <code>AgentState</code>。
7. 顺序等待订阅者，提供消息提交和持久化 barrier。
8. 在 provider 或 callback 直接抛异常时，合成一个 error/aborted AssistantMessage。

<code>agent-loop</code> 定义在 <code>packages/agent/src/agent-loop.ts</code>。它是无 UI、无存储的编排器，负责：

1. 把本轮 prompt 合并进上下文。
2. 调用 <code>transformContext()</code> 和 <code>convertToLlm()</code>。
3. 调用注入的 <code>StreamFn</code> 并消费流式事件。
4. 从最终 AssistantMessage 中提取 ToolCall。
5. 串行预检、顺序或并行执行工具。
6. 把工具结果转换成 ToolResultMessage。
7. 轮询 steering/follow-up 队列。
8. 根据 stop reason、工具 terminate、<code>shouldStopAfterTurn</code> 和队列状态决定继续或结束。

### 2.2 Runtime 不负责什么

Runtime 刻意不负责：

- 不选择具体 Provider 或 HTTP SDK。
- 不管理 API key 文件或 OAuth。
- 不知道 TUI、CLI、RPC 或 Web UI。
- 不自动把 transcript 保存到磁盘。
- 不负责当前 coding-agent 产品的 JSONL SessionManager。
- 不决定模型为何选择某个工具；模型只会收到可用工具的 schema。

这些能力由宿主通过 <code>streamFn</code>、事件订阅、<code>transformContext</code>、tool hooks 和 session 管理器注入。

### 2.3 为什么需要两层

如果把所有逻辑都放进 <code>Agent</code>，内存状态、流式协议和工具循环会互相耦合。当前拆分让两种调用方式共存：

- 高层调用：使用 <code>Agent</code>，获得状态管理、队列、事件 barrier 和 abort。
- 低层调用：直接使用 <code>agentLoop()</code>，自行管理上下文和事件。

低层 EventStream 只保证事件产生顺序，不等待外部异步消费者完成；高层 <code>Agent.processEvents()</code> 会逐个 await listener。因此需要在 assistant <code>message_end</code> 后持久化完成再开始工具预检时，应使用 <code>Agent</code>。

## 3. Agent 状态保存在哪里？

### 3.1 基础 Agent 的内存状态

公开状态接口在 <code>packages/agent/src/types.ts:333</code>：

~~~ts
interface AgentState {
  systemPrompt: string;
  model: Model;
  thinkingLevel: ThinkingLevel;
  tools: AgentTool[];
  messages: AgentMessage[];
  readonly isStreaming: boolean;
  readonly streamingMessage?: AgentMessage;
  readonly pendingToolCalls: ReadonlySet<string>;
  readonly errorMessage?: string;
}
~~~

实际对象由 <code>createMutableAgentState()</code> 创建，位置是 <code>packages/agent/src/agent.ts:68</code>。

状态可分三组：

| 状态 | 保存位置 | 生命周期 |
|---|---|---|
| systemPrompt、model、thinkingLevel、tools、messages | <code>MutableAgentState</code> | 跨多次 prompt 保留，直到修改或 reset |
| isStreaming、streamingMessage、pendingToolCalls、errorMessage | <code>MutableAgentState</code> | 由运行事件更新 |
| AbortController、run promise | <code>Agent.activeRun</code> | 每次 run 独立 |
| steering/follow-up | 两个 <code>PendingMessageQueue</code> | Agent 实例级，消费或 clear 前保留 |

数组有一个容易忽略的语义：

- 构造或赋值 <code>state.tools = value</code>、<code>state.messages = value</code> 时，会复制顶层数组。
- getter 返回内部数组，因此 <code>agent.state.messages.push(message)</code> 是允许的。
- 数组内的 Message、Tool 或 Model 对象不会深拷贝。

### 3.2 运行时状态如何更新

状态归约集中在 <code>Agent.processEvents()</code>，位置是 <code>packages/agent/src/agent.ts:544</code>：

| Event | 状态变化 |
|---|---|
| message_start | 设置 <code>streamingMessage</code> |
| message_update | 用最新 partial 更新 <code>streamingMessage</code> |
| message_end | 清空 partial，把 finalized message 追加进 <code>state.messages</code> |
| tool_execution_start | 把 toolCallId 加入新的 pending Set |
| tool_execution_end | 从新的 pending Set 中删除 toolCallId |
| turn_end | 若 assistant 有 errorMessage，更新 <code>state.errorMessage</code> |
| agent_end | 清空 <code>streamingMessage</code> |

<code>finishRun()</code> 最后把 <code>isStreaming</code> 设为 false，清空运行态并 resolve <code>waitForIdle()</code>。

### 3.3 state.messages 与 currentContext.messages 不是同一个数组

<code>Agent.createContextSnapshot()</code> 在 <code>packages/agent/src/agent.ts:437</code> 对 messages 和 tools 做浅复制。loop 修改的是本次 run 的 <code>currentContext.messages</code>；<code>Agent</code> 则通过 <code>message_end</code> 事件把 finalized message 追加到长期 <code>state.messages</code>。

这样做有两个作用：

1. 一次 run 内部可以安全维护自己的上下文。
2. Agent 的长期状态只在 finalized message 的提交点变化，不会把每个 partial delta 当成独立历史消息。

### 3.4 sessionId 不是存储

<code>Agent.sessionId</code> 最终只是由 <code>createLoopConfig()</code> 放进 SimpleStreamOptions，再传给 Provider。它可用于后端 cache affinity 或请求关联，但没有任何代码根据它读写 transcript。

## 4. Message 如何流转？

### 4.1 输入正规化

<code>Agent.prompt()</code> 接受三类输入：

- string
- 单个 AgentMessage
- AgentMessage 数组

字符串由 <code>normalizePromptInput()</code> 变成：

~~~ts
{
  role: "user",
  content: [
    { type: "text", text: input },
    ...images
  ],
  timestamp: Date.now()
}
~~~

如果 Agent 正在运行，第二个 <code>prompt()</code> 不会排队，而是直接抛错。调用方必须使用 <code>steer()</code>、<code>followUp()</code> 或等待当前 run 完成。

### 4.2 Prompt 进入 loop

<code>runAgentLoop()</code> 位于 <code>packages/agent/src/agent-loop.ts:95</code>，它执行：

1. 创建 <code>newMessages = [...prompts]</code>。
2. 创建 <code>currentContext.messages = [...oldMessages, ...prompts]</code>。
3. 发出 <code>agent_start</code>。
4. 发出首个 <code>turn_start</code>。
5. 对每个 prompt 发出 <code>message_start → message_end</code>。
6. 进入共享的 <code>runLoop()</code>。

Agent 收到 prompt 的 <code>message_end</code> 后，立即把 user message 追加进长期 state。

### 4.3 LLM 边界的两次转换

每次调用模型前，<code>streamAssistantResponse()</code> 都执行：

~~~text
currentContext.messages: AgentMessage[]
  → transformContext(messages, signal)
  → AgentMessage[]
  → convertToLlm(messages)
  → pi-ai Message[]
  → Context { systemPrompt, messages, tools }
  → streamFn(model, Context, options)
~~~

两者职责不同：

- <code>transformContext</code> 处理 AgentMessage 层的裁剪、压缩或外部上下文注入。
- <code>convertToLlm</code> 把自定义消息投影为模型支持的 user/assistant/toolResult，或过滤 UI-only 消息。

关键语义：

> transformContext 的返回值只影响当前这次 Provider 请求，不会自动替换 Agent 的原始 transcript。

默认 <code>convertToLlm</code> 只保留 role 为 user、assistant、toolResult 的消息。自定义 AgentMessage 若不提供转换逻辑，就不会被默认发送给模型。

### 4.4 Assistant 流式响应

StreamFn 返回 <code>AssistantMessageEventStream</code>。runtime 的处理方式是：

| Provider event | runtime 行为 |
|---|---|
| start | 把 partial assistant 放入 currentContext，发出 message_start |
| text/thinking/toolcall start/delta/end | 替换 currentContext 尾部 partial，发出 message_update |
| done/error | 调用 <code>response.result()</code> 取得 finalized AssistantMessage，替换 partial，发出 message_end |

Agent 的 <code>streamingMessage</code> 随 update 改变，但 finalized assistant 只在 <code>message_end</code> 时进入 <code>state.messages</code>。

### 4.5 完整事件顺序

无工具调用时：

~~~text
agent_start
turn_start
message_start(user)
message_end(user)
message_start(assistant partial/final)
message_update(assistant) × N
message_end(assistant final)
turn_end
agent_end
~~~

有工具调用时：

~~~text
agent_start
turn_start
message_start/end(user)
message_start/update/end(assistant with toolCall)
tool_execution_start
tool_execution_update × N
tool_execution_end
message_start/end(toolResult)
turn_end
turn_start
message_start/update/end(assistant final)
turn_end
agent_end
~~~

### 4.6 消息序列图

~~~mermaid
sequenceDiagram
    actor U as 调用方
    participant A as Agent
    participant L as agent-loop
    participant P as StreamFn / Provider
    participant T as AgentTool
    participant S as Event Subscriber

    U->>A: prompt(input)
    A->>A: 创建 ActiveRun + AbortController
    A->>L: runAgentLoop(prompts, context snapshot)
    L-->>A: message_end(user)
    A->>A: state.messages.push(user)
    A-->>S: await listener(message_end)
    L->>L: transformContext + convertToLlm
    L->>P: streamFn(model, llmContext)
    P-->>L: start / delta / done
    L-->>A: message_start/update/end(assistant)
    A->>A: 保存 finalized assistant
    A-->>S: await listeners
    alt assistant 含 ToolCall
        L-->>A: tool_execution_start
        L->>T: execute(id, validatedArgs, signal, onUpdate)
        T-->>L: AgentToolResult
        L-->>A: tool_execution_end
        L-->>A: message_end(toolResult)
        A->>A: state.messages.push(toolResult)
        L->>P: 下一次模型调用，Context 含 toolResult
        P-->>L: 最终 assistant
    end
    L-->>A: agent_end
    A-->>S: await agent_end listeners
    A->>A: finishRun()
    A-->>U: prompt() resolve
~~~

## 5. Tool 如何注册？

### 5.1 Tool 定义

<code>AgentTool</code> 定义在 <code>packages/agent/src/types.ts:386</code>。它继承 pi-ai 的 Tool schema，并增加 runtime 能力：

~~~ts
interface AgentTool extends Tool {
  name: string;
  label: string;
  description: string;
  parameters: TSchema;
  prepareArguments?: (rawArgs) => preparedArgs;
  execute: (
    toolCallId,
    validatedParams,
    signal?,
    onUpdate?
  ) => Promise<AgentToolResult>;
  executionMode?: "sequential" | "parallel";
}
~~~

其中：

- name：模型 ToolCall 和 runtime 查找使用的稳定标识。
- label：主要供 UI 显示。
- description + parameters：发给模型，帮助模型决定是否调用并生成参数。
- prepareArguments：在 schema 校验前兼容模型或旧版本参数。
- execute：真实副作用。
- executionMode：控制包含它的整个工具 batch 是否必须串行。

### 5.2 注册实际上是依赖注入

基础 Agent 没有 <code>registerTool()</code> 或全局单例 Registry。工具通过数组注入：

~~~ts
const agent = new Agent({
  initialState: {
    model,
    tools: [readTool, writeTool]
  },
  streamFn
});

agent.state.tools = [...agent.state.tools, customTool];
~~~

运行开始时，<code>createContextSnapshot()</code> 把当前 tools 浅复制到 <code>AgentContext.tools</code>。Provider 请求看到的是这个运行快照。

这意味着：

1. Tool 的可用性是 Agent 实例级，不是进程全局。
2. 同一进程中的两个 Agent 可以有完全不同的工具集合。
3. 在 run 中途修改 <code>agent.state.tools</code>，不会自动修改当前 loop 的 context。
4. 若要让中途变更在下一 turn 生效，需要通过 <code>prepareNextTurn</code> 返回新的 context；coding-agent 产品层正是这样刷新工具和系统提示词。

### 5.3 名称是运行时查找键

<code>prepareToolCall()</code> 使用：

~~~ts
currentContext.tools?.find(tool => tool.name === toolCall.name)
~~~

因此 name 必须唯一。核心代码没有注册时的重复名称校验；如果数组中存在重复 name，实际会选择第一个匹配项。

## 6. Tool 如何调用？

### 6.1 从模型输出到执行

Tool 不是由本地关键字匹配触发。流程是：

1. Runtime 把工具的 name、description 和 parameters 随 Context 发给模型。
2. 模型在 AssistantMessage.content 中返回 <code>{ type: "toolCall", id, name, arguments }</code>。
3. <code>runLoop()</code> 过滤所有 ToolCall。
4. <code>executeToolCalls()</code> 选择 sequential 或 parallel。
5. 每个 ToolCall 进入预检、执行和结果归一化。

### 6.2 执行模式

入口是 <code>packages/agent/src/agent-loop.ts:411</code>。

满足任一条件时，整个 batch 串行：

- 全局 <code>config.toolExecution === "sequential"</code>。
- batch 中任一 Tool 对应定义声明 <code>executionMode: "sequential"</code>。

否则走并行模式。默认全局模式是 parallel。

并行模式的精确语义不是“发现 ToolCall 就立即并发”：

1. 按 assistant 源顺序依次发出 tool_execution_start。
2. 按源顺序依次执行 lookup、prepare、schema validation 和 beforeToolCall。
3. 全部预检完成后，允许执行的工具才通过 Promise.all 并发运行。
4. tool_execution_end 按真实完成顺序发出。
5. 等全部工具完成后，ToolResultMessage 按 assistant 原始 ToolCall 顺序写入 transcript。

这样既允许并发，又保证模型看到的结果顺序稳定。

### 6.3 单个 ToolCall 的调用管线

~~~text
Assistant ToolCall
  → tool_execution_start
  → 按 name 查找 AgentTool
  → prepareArguments(raw)
  → validateToolArguments(schema)
  → beforeToolCall
      ├─ block → error result
      └─ allow
  → tool.execute(id, validatedArgs, signal, onUpdate)
      ├─ onUpdate → tool_execution_update
      ├─ return → success result
      └─ throw → error result
  → afterToolCall
  → tool_execution_end
  → ToolResultMessage
  → message_start/end(toolResult)
  → 回注 currentContext
~~~

对应源码：

- 参数准备与 before hook：<code>agent-loop.ts:600</code>
- 真实 execute 与 update：<code>agent-loop.ts:670</code>
- after hook：<code>agent-loop.ts:713</code>
- ToolResultMessage：<code>agent-loop.ts:777</code>

### 6.4 参数和错误处理

顺序是：

~~~text
prepareArguments → schema validation → beforeToolCall → execute
~~~

如果以下任一阶段失败，runtime 会构造 error ToolResult：

- 找不到工具。
- prepareArguments 抛错。
- schema 校验失败。
- beforeToolCall 抛错或 block。
- execute 抛错。
- afterToolCall 抛错。

工具失败不会直接让整个 agent-loop throw；错误会作为 <code>isError: true</code> 的 ToolResultMessage 返回模型，由模型决定如何修正或向用户解释。

### 6.5 截断 ToolCall 的保护

如果 assistant 的 <code>stopReason === "length"</code>，即使 content 中已经出现可解析 ToolCall，runtime 也不会执行。原因是 arguments 可能在 token limit 处被截断，恰好仍能解析和通过 schema，但语义不完整。

<code>failToolCallsFromTruncatedMessage()</code> 会为每个 ToolCall 生成 error ToolResult，要求模型重新发起完整调用。

### 6.6 terminate 的含义

Tool、before hook 或 after hook 都可以设置 <code>terminate: true</code>。它不是 abort：

- 当前工具仍正常完成。
- ToolResultMessage 仍写入 transcript。
- 只有当前 batch 的每一个 finalized result 都是 terminate，才跳过自动 follow-up LLM 调用。
- 混合 batch 中只要有一个结果未 terminate，loop 就继续。

## 7. 如何处理中断？

pi-agent-core 有三种不同性质的停止机制，不能混为一谈。

### 7.1 硬中断：AbortController

每次 <code>runWithLifecycle()</code> 创建一个新的 AbortController，位置是 <code>packages/agent/src/agent.ts:486</code>。<code>agent.abort()</code> 只做一件事：

~~~ts
this.activeRun?.abortController.abort();
~~~

Signal 会传给：

- Provider 的 streamFn options。
- transformContext。
- beforeToolCall 和 afterToolCall。
- AgentTool.execute。
- Agent 事件订阅者。
- shouldStopAfterTurn 和 prepareNextTurn 包装回调。

这是协作式取消，不是强制终止线程：

- Provider 必须监听 signal 并结束流。
- Tool 必须监听 signal 并停止自己的 IO/子进程。
- Hook 也必须自行尊重 signal。
- runtime 无法强行停止一段忽略 signal 的 JavaScript Promise。

顺序工具模式会在每个 finalized tool 后检查 signal 并停止准备后续 ToolCall。并行模式会等待已启动的 Promise 完成；这些工具是否快速结束，取决于它们是否正确处理 signal。

### 7.2 aborted 消息如何落入状态

StreamFn 的契约要求普通 provider/request 失败不要直接 reject，而应以流事件结束，并返回：

- <code>stopReason: "aborted"</code> 或 <code>"error"</code>
- <code>errorMessage</code>

loop 收到这样的 AssistantMessage 后仍会正常发出：

~~~text
message_end
turn_end
agent_end
~~~

如果外部实现直接 throw，<code>Agent.handleRunFailure()</code> 会合成一个空文本 AssistantMessage，并根据 AbortSignal 是否已 aborted 设置 stopReason。

### 7.3 软中断：steering

<code>agent.steer(message)</code> 不调用 abort，只把消息放入 steering queue。

steering 的消费点：

1. loop 开始时先轮询一次。
2. 每个 assistant turn 完成后轮询。
3. 当前 assistant 已发出的所有工具调用完成后才轮询。

因此 steering 的真实语义是：

> 让下一 turn 改变方向，而不是取消当前正在执行的工具。

如果需要立刻取消正在运行的外部命令，宿主通常要先 <code>abort()</code>，等待 idle，再发起新的 prompt；单独 steer 不提供强制停止。

### 7.4 follow-up

<code>followUp()</code> 也是队列，但只在 agent 原本准备结束时检查：

1. 没有更多 ToolCall。
2. 没有 steering message。
3. then 轮询 follow-up。

有 follow-up 时，外层 loop 继续并把它作为下一 turn 的输入。

### 7.5 graceful stop

<code>shouldStopAfterTurn</code> 在当前 assistant 和工具全部完成、<code>turn_end</code> 已发出之后运行。返回 true 时：

- 发出 agent_end。
- 不再轮询 steering。
- 不再轮询 follow-up。
- 不开始下一次 Provider 调用。
- 不修改当前 AssistantMessage 的 stopReason。

另外，<code>prepareNextTurn</code> 在它之前执行，因此宿主可以先持久化或刷新下一 turn 状态，再决定是否 graceful stop。

### 7.6 idle 的精确定义

<code>agent_end</code> 是最后一个生产者事件，但不是 <code>Agent</code> 立即 idle 的时刻。<code>Agent.processEvents()</code> 会等待所有 agent_end listener 完成，之后 <code>finishRun()</code> 才：

- 设 <code>isStreaming = false</code>。
- 清理 activeRun。
- resolve <code>waitForIdle()</code>。

这保证 session 持久化 listener 可以在 prompt Promise 返回前完成。

## 8. 如何处理多轮对话？

“轮”在这里有两个层次。

### 8.1 Turn：一次模型响应加其工具执行

源码对 turn 的定义是：

~~~text
turn_start
  → 一次 AssistantMessage
  → 该 AssistantMessage 发起的全部 ToolCall 和 ToolResult
turn_end
~~~

工具执行后若需要模型读取 ToolResult，会开始新的 turn。

例如：

~~~text
Turn 1: user → assistant(toolCall)
        → toolResult
Turn 2: assistant(final text)
~~~

### 8.2 Conversation round：一次新的用户 prompt

第一次 <code>prompt("A")</code> 完成后：

~~~text
state.messages = [user(A), assistant(A-result)]
~~~

下一次 <code>prompt("B")</code>：

1. <code>createContextSnapshot()</code> 复制已有 transcript。
2. <code>runAgentLoop()</code> 追加 user(B)。
3. 下一次模型请求看到 A 的历史和 B 的新输入。
4. finalized message 继续追加到同一个 state.messages。

因此基础多轮能力来自“复用同一个 Agent 实例及其 messages”，不是来自隐式数据库。

### 8.3 同一 run 为什么能产生多次模型调用

<code>runLoop()</code> 有两层循环：

~~~text
outer while:
  处理 follow-up

  inner while:
    当有 ToolCall 或 pending steering 时继续
~~~

继续条件包括：

- 上一 assistant 产生 ToolCall，且工具 batch 没有全部 terminate。
- 有 steering message。
- agent 本来要结束，但有 follow-up message。

### 8.4 QueueMode

steering 和 follow-up 各自支持：

- <code>all</code>：一次 drain 全部消息。
- <code>one-at-a-time</code>：每个消费点只取最早一条。

Agent 默认两者都是 one-at-a-time。队列是 FIFO。

### 8.5 continue() 的含义

<code>continue()</code> 位于 <code>packages/agent/src/agent.ts:361</code>。它不会添加新的 user message，主要用于：

- 已有 transcript 尾部是 user 或 toolResult 时，继续请求模型。
- 错误恢复后，从当前内存上下文重试。

限制：

- transcript 不能为空。
- 尾部是 assistant 时通常不能 continue，因为 Provider 协议需要可继续的 user/toolResult 尾部。
- 如果 assistant 尾部存在 queued steering 或 follow-up，Agent 会先 drain 队列，再用这些消息启动新的 prompt run。
- 运行中调用 continue 会被拒绝。

因此 <code>continue()</code> 是“从已加载的内存上下文继续”，不是“从磁盘恢复 session”。

### 8.6 每个 turn 都重新转换上下文

<code>transformContext</code> 和 <code>convertToLlm</code> 在每次 Provider 调用前执行，不只在整个 prompt 开始时执行。这让以下变化能影响后续 turn：

- context compaction。
- 自定义消息投影。
- 外部上下文注入。
- 通过 prepareNextTurn 更新的 system prompt、model、thinking level 或 tools。

## 9. 如何恢复 session？

这个问题必须分三层回答。

### 9.1 基础 Agent：没有自动 session 恢复

<code>Agent</code> 不读文件，也没有 <code>loadSession()</code>。恢复必须由宿主完成：

~~~text
持久化存储
  → 读取 transcript、model、thinking level、tool configuration
  → 解析自定义消息
  → 创建 Agent(initialState)
  → 或给 agent.state.messages 赋值
  → prompt(new input) / continue()
~~~

最小思路：

~~~ts
const agent = new Agent({
  initialState: {
    systemPrompt,
    model: restoredModel,
    thinkingLevel: restoredThinkingLevel,
    tools: rebuiltTools,
    messages: restoredMessages
  },
  streamFn
});
~~~

必须重建的不只是 messages：

- Model 对象需要从当前 Model Registry 重新解析，不能只靠旧字符串。
- Tool 的 execute 是函数，不能从 JSON 反序列化，必须由应用重新注册。
- 自定义消息需要相同的 declaration merging 和 convertToLlm。
- system prompt、thinking level 和 Provider 配置也应按宿主语义恢复。

### 9.2 当前 pi CLI：由 coding-agent 产品层恢复

当前产品主链不使用 <code>AgentHarness.resume()</code>，而由 <code>packages/coding-agent/src/core/sdk.ts</code> 恢复：

1. <code>sessionManager.buildSessionContext()</code> 读取当前分支上下文，位置 <code>sdk.ts:190</code>。
2. 从 session context 的 provider/modelId 在 ModelRuntime 中重新查找 Model。
3. 恢复 thinking level。
4. 创建新的 <code>Agent</code>，位置 <code>sdk.ts:304</code>。
5. 将已有消息赋给 <code>agent.state.messages</code>，位置 <code>sdk.ts:374</code>。
6. AgentSession 再重新组装工具、扩展和产品级 hooks。

所以当前 CLI 的 session 恢复公式是：

~~~text
SessionManager JSONL
  → buildSessionContext
  → 恢复 model / thinking
  → new Agent
  → agent.state.messages = restored messages
  → 重新注册工具与扩展
~~~

### 9.3 包内通用 Session：数据恢复已实现

通用 session API 位于 <code>packages/agent/src/harness/session</code>。

#### 数据模型

持久化内容分为 Entry 和 LaneRecord。

Entry 构成对话树：

| Entry | 含义 |
|---|---|
| message | user、assistant、toolResult 或自定义 AgentMessage |
| model_change | 模型变化 |
| thinking_level_change | thinking level 变化 |
| active_tools_change | 活动工具名称变化 |
| compaction | 压缩摘要和 retained tail |
| branch_summary | 分支摘要 |
| custom | 应用自定义持久化数据 |

每个 Entry 都有：

- id
- 全局递增 seq
- parentId
- timestamp

Lane 保存一个 leaf pointer，因此多个 lane 可以指向同一棵 Entry tree 的不同分支。

LaneRecord 保存“运行过程事实”：

- operation_started / operation_finished
- abort_requested
- step_attempt
- tool_started
- queue_enqueued / queue_cancelled
- write_deferred
- usage

Entry 负责“对话事实”，Record 负责“一个操作进行到哪里”。

#### JSONL 重开流程

<code>JsonlSessionRepo.open()</code> 位于 <code>jsonl/repo.ts:130</code>：

~~~text
metadata
  → loadJsonlSessionStorage
  → JsonlSessionStorage.load(path)
  → 解析 v4 header
  → 逐行 parseMutation
  → SessionState.applyMutation
  → new Session(storage)
~~~

<code>SessionState</code> 通过重放 mutation 重建：

- entries 和 entriesById。
- lanes 的 leaf pointer。
- records 和每个 lane 的 open operation。
- name、labels。
- token/cost stats。

JSONL loader 还处理 crash 尾部：

- 最后一行是语法不完整的 torn tail：原子发布有效前缀，丢弃未确认的部分 append。
- 最后一行内容有效但缺换行：补一个换行。
- 中间行损坏或完整但 schema 非法：拒绝打开，不静默修复。

#### 从 Entry tree 构建模型上下文

<code>buildSessionContext()</code> 位于 <code>harness/session/context.ts:90</code>。

输入应是当前 leaf 到 root 路径按 oldest-first 排列的 entries。它：

1. 扫描路径，恢复最新 model、thinking level 和 active tool names。
2. 找到最新 compaction，只保留 compaction 和之后的 entries。
3. 把 compaction 变成 summary message，并展开 retainedTail。
4. 把 branch summary 变成上下文消息。
5. 通过 projector 把 custom entry 投影为 AgentMessage。
6. 忽略 stopReason 为 deferred 的占位 assistant。

#### 恢复进行中的操作

<code>validateRecordLog()</code> 和 <code>reduceLaneState()</code> 位于：

- <code>packages/agent/src/harness/reducer.ts:312</code>
- <code>packages/agent/src/harness/reducer.ts:506</code>

恢复分两步：

1. validate：拒绝单写者协议不可能产生的矛盾状态，例如同一 lane 多个 open operation、finish 后仍有 record、tool ordinal 不匹配或 provisioned entry 内容冲突。
2. reduce：纯函数重建 pending initial messages、steer/follow-up、deferred writes、unfinished step、tool batch、deferred handle、abort 状态和有效配置。

这套 reducer 已经定义了 crash recovery 需要的确定性状态，但尚未接入可运行的 AgentHarness。

### 9.4 当前未完成的部分

<code>AgentHarness.create()</code> 位于 <code>packages/agent/src/harness/agent-harness.ts:347</code>：

~~~ts
const [record] = await options.session.findRecords({ limit: 1 });
if (record !== undefined) {
  throw new HarnessNotImplemented("create.restore");
}
~~~

<code>AgentHarness.resume()</code> 位于同文件 <code>:380</code>，也直接返回 <code>HarnessNotImplemented</code>。

准确结论是：

| 能力 | 当前状态 |
|---|---|
| 重新打开 JSONL session 数据 | 已实现 |
| 重建 Entry tree、lane、facts、stats | 已实现 |
| 从分支构建 LLM Context | 已实现 |
| 从 record log 纯函数重建 suspended operation | reducer 已实现 |
| AgentHarness 自动恢复并继续运行 suspended operation | 尚未实现 |
| 当前 pi CLI 恢复历史对话 | 已由 coding-agent SessionManager 实现 |

## 10. 八条链路放在一起

~~~mermaid
flowchart TB
    Host[CLI / SDK / App]
    Agent[Agent]
    State[MutableAgentState]
    Queue[Steering / Follow-up Queues]
    Loop[runAgentLoop]
    Transform[transformContext]
    Convert[convertToLlm]
    Provider[StreamFn / Provider]
    ToolLookup[Tool lookup + validation + hooks]
    Tool[AgentTool.execute]
    Events[AgentEvent]
    Store[Host Session Store]

    Host -->|initialState + tools| Agent
    Agent --> State
    Agent --> Queue
    Agent -->|context snapshot| Loop
    Loop --> Transform
    Transform --> Convert
    Convert --> Provider
    Provider -->|Assistant stream| Loop
    Loop -->|ToolCall| ToolLookup
    ToolLookup --> Tool
    Tool -->|AgentToolResult| Loop
    Loop -->|ToolResultMessage| Provider
    Loop --> Events
    Events --> Agent
    Agent --> State
    Agent -->|await subscribers| Store
    Queue -->|drain points| Loop
~~~

## 11. 测试如何验证这些结论

以下是现有测试中的直接证据。当前工作区未安装 node_modules，因此本次只完成源码交叉阅读，未执行 Vitest。

| 行为 | 测试 |
|---|---|
| 默认状态和 initialState | <code>packages/agent/test/agent.test.ts:106</code>、<code>:121</code> |
| message/tool 数组赋值会复制 | <code>agent.test.ts:442</code> |
| async subscriber 是 prompt 和 idle 的 barrier | <code>agent.test.ts:190</code>、<code>:228</code> |
| AbortSignal 会传给 subscriber | <code>agent.test.ts:263</code> |
| 运行中拒绝第二次 prompt/continue | <code>agent.test.ts:542</code>、<code>:582</code> |
| assistant 尾部可消费 queued follow-up/steering | <code>agent.test.ts:618</code>、<code>:656</code> |
| transformContext 先于 convertToLlm | <code>agent-loop.test.ts:221</code> |
| ToolCall 到 ToolResult 的完整链 | <code>agent-loop.test.ts:274</code> |
| length 截断的 ToolCall 不执行 | <code>agent-loop.test.ts:371</code> |
| 并行完成事件与持久化结果顺序不同 | <code>agent-loop.test.ts:586</code> |
| steering 等全部当前工具完成后注入 | <code>agent-loop.test.ts:681</code> |
| 单个 sequential Tool 强制整个 batch 串行 | <code>agent-loop.test.ts:870</code> |
| 全部 parallel Tool 可并发 | <code>agent-loop.test.ts:957</code> |
| shouldStopAfterTurn 跳过队列轮询 | <code>agent-loop.test.ts:1104</code> |
| JSONL 写入 mutation 并重建 sequence | <code>packages/agent/test/harness/session/jsonl.test.ts:267</code> |
| JSONL 重开恢复 fork lanes/facts | <code>jsonl.test.ts:345</code> |
| JSONL 修复缺换行和 torn tail | <code>jsonl.test.ts:426</code>、<code>:472</code> |
| context 从最新 compaction 开始 | <code>packages/agent/test/harness/session/context.test.ts:40</code> |
| reducer 重建队列、pending write 和 unfinished step | <code>packages/agent/test/harness/reducer.test.ts:768</code> |
| abort 清空 steer/follow-up 恢复态 | <code>reducer.test.ts:808</code> |

## 12. 容易误判的细节

### 12.1 message_end 才是 finalized message 的提交点

message_update 只更新 <code>streamingMessage</code>。持久化层应监听 message_end，而不是把每个 delta 追加为历史消息。

### 12.2 transformContext 不会改写原 transcript

它只构造某一次 LLM 请求看到的消息。要永久替换上下文，需要宿主显式更新 state.messages 或持久化树。

### 12.3 steering 不是立即取消

steer 会等当前工具 batch 全部结束。要求立即停止时，需要 abort，并且 Tool/Provider 必须实现 AbortSignal。

### 12.4 sessionId 不等于 session persistence

只设置 sessionId 而不保存 state.messages，进程退出后没有任何对话可以恢复。

### 12.5 ToolResult 顺序与完成事件顺序不同

并行工具的 tool_execution_end 按完成顺序；写入模型 transcript 的 ToolResultMessage 按原 ToolCall 顺序。

### 12.6 Tool 函数不能从 JSON 恢复

持久化层最多保存 activeToolNames。真正的 execute 函数必须由宿主重新注册，再根据名称恢复启用状态。

### 12.7 基础 Agent 与 AgentHarness 不能当成同一个 runtime

基础 Agent 已完整运行；AgentHarness 当前仍是 scaffold。看到 Harness 的 resume 类型和 reducer，不能推断现有 Harness 已经支持 crash resume。

## 13. 建议的源码断点顺序

验证普通对话：

1. <code>Agent.prompt()</code>
2. <code>Agent.runPromptMessages()</code>
3. <code>runAgentLoop()</code>
4. <code>runLoop()</code>
5. <code>streamAssistantResponse()</code>
6. <code>Agent.processEvents()</code>

验证工具：

1. <code>executeToolCalls()</code>
2. <code>prepareToolCall()</code>
3. <code>executePreparedToolCall()</code>
4. <code>finalizeExecutedToolCall()</code>
5. <code>createToolResultMessage()</code>
6. 第二次 <code>streamAssistantResponse()</code>

验证 abort：

1. <code>Agent.runWithLifecycle()</code>
2. <code>Agent.abort()</code>
3. Provider 对 signal 的处理
4. Tool 对 signal 的处理
5. <code>message_end(aborted)</code>
6. <code>finishRun()</code>

验证 JSONL session 重开：

1. <code>JsonlSessionRepo.open()</code>
2. <code>JsonlSessionStorage.load()</code>
3. <code>parseMutation()</code>
4. <code>SessionState.applyMutation()</code>
5. <code>buildSessionContext()</code>
6. 如果研究未来 suspended operation：<code>validateRecordLog()</code> 和 <code>reduceLaneState()</code>

## 14. 最终心智模型

可以用下面五句话记住 pi-agent-core：

1. <code>Agent</code> 是有状态外壳，<code>agent-loop</code> 是无存储编排器。
2. AgentMessage 在 LLM 边界才转换成 Provider 可理解的 Message。
3. Tool 是注入的能力对象，模型只提出 ToolCall，本地 runtime 校验并执行。
4. abort 是协作式硬取消，steer 是 turn 边界上的软改向。
5. transcript 持久化属于宿主；包内 session 后端能恢复数据，但 AgentHarness 自动恢复运行尚未完成。
