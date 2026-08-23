# Pi Tool 系统源码分析

本文专门分析 Pi 当前 CLI 主链中的 Tool 系统，回答六个问题：

1. Tool 接口如何定义？
2. Tool schema 如何生成？
3. LLM 如何知道有哪些工具？
4. ToolCall 参数如何解析？
5. Tool 执行异常如何处理？
6. Tool 结果如何进入下一轮 Context？

本文不仅描述调用顺序，还区分 Provider 协议解析、schema 校验、策略拦截、工具执行、结果持久化这几个容易混淆的边界，并总结企业 Agent 可以直接复用的设计。

源码基线：

- Git commit：<code>a69bef789bc95abf0acee16f7b4660b70b650bb9</code>
- npm package：<code>@earendil-works/pi-ai@0.84.2</code>、<code>@earendil-works/pi-agent-core@0.84.2</code>、<code>@earendil-works/pi-coding-agent@0.84.2</code>
- 模型层：<code>packages/ai</code>
- Runtime 层：<code>packages/agent</code>
- 产品层：<code>packages/coding-agent</code>

## 0. 六个问题的直接答案

| 问题 | 结论 | 核心源码 |
|---|---|---|
| Tool 接口如何定义 | Pi 分三层定义 Tool：<code>pi-ai.Tool</code> 是发给模型的协议描述；<code>AgentTool</code> 增加本地执行函数；<code>ToolDefinition</code> 再增加产品层的提示词、Extension Context 和 UI renderer。 | <code>packages/ai/src/types.ts:514</code>、<code>packages/agent/src/types.ts:386</code>、<code>packages/coding-agent/src/core/extensions/types.ts:449</code> |
| Tool schema 如何生成 | schema 不是从 TypeScript interface 反射生成的，而是用 TypeBox 的 <code>Type.Object()</code> 直接构造运行时 JSON Schema；<code>Static&lt;typeof schema&gt;</code> 再从同一对象推导 TypeScript 参数类型。Provider adapter 按各厂商协议转换或收紧 schema。 | <code>packages/coding-agent/src/core/tools/edit.ts:34</code>、<code>:45</code>、<code>packages/ai/src/api/constrained-sampling.ts:117</code> |
| LLM 如何知道有哪些工具 | 当前活动工具随 <code>Context.tools</code> 传到 Provider adapter，再被转换成 OpenAI <code>tools</code>、Anthropic <code>tools[].input_schema</code> 或 Gemini <code>functionDeclarations</code>。system prompt 中的工具列表只是辅助提示，不是调用协议。 | <code>packages/agent/src/agent-loop.ts:298</code>、<code>packages/ai/src/api/openai-responses.ts:313</code> |
| ToolCall 参数如何解析 | Provider adapter 先把流式 JSON 文本解析为统一的 <code>ToolCall.arguments</code>；agent-loop 再执行 <code>prepareArguments → validateToolArguments</code>。解析回答“JSON 是什么”，校验回答“参数是否合法”，两者不是同一步。 | <code>packages/ai/src/utils/json-parse.ts:104</code>、<code>packages/agent/src/agent-loop.ts:586</code>、<code>packages/ai/src/utils/validation.ts:317</code> |
| Tool 执行异常如何处理 | 找不到工具、参数准备/校验失败、hook 阻止或抛错、<code>execute()</code> 抛错、after hook 抛错，都会被归一化为 <code>isError: true</code> 的 ToolResult，而不是默认终止 Agent。模型可在下一轮看到错误并修正。 | <code>packages/agent/src/agent-loop.ts:600</code>、<code>:670</code>、<code>:713</code> |
| Tool 结果如何进入下一轮 Context | Runtime 把 <code>AgentToolResult</code> 转为标准 <code>ToolResultMessage</code>，追加到本次 loop 的 <code>currentContext.messages</code>；因为仍有 ToolCall，loop 再次调用模型。Provider adapter 用 <code>toolCallId</code> 把结果与原调用配对。 | <code>packages/agent/src/agent-loop.ts:216</code>、<code>:777</code>、<code>packages/ai/src/api/openai-responses-shared.ts:308</code> |

最重要的结论是：

> Tool 系统不是一个 <code>execute()</code> 函数，而是一条跨越 schema、Provider 协议、运行时校验、策略控制、副作用执行、消息回注和持久化的完整数据链。

## 1. 三层 Tool 模型

Pi 没有用一个巨型接口同时承担模型协议、执行和 UI。它把 Tool 分成三层。

~~~mermaid
flowchart LR
    TD[ToolDefinition<br/>coding-agent 产品定义]
    AT[AgentTool<br/>agent-core 可执行工具]
    T[Tool<br/>pi-ai 模型协议]
    PA[Provider Adapter]
    LLM[LLM API]

    TD -->|wrapToolDefinition| AT
    AT -->|extends| T
    T --> PA
    PA -->|厂商原生 tools schema| LLM

    TD -. promptSnippet / renderer / ExtensionContext .-> Product[产品层]
    AT -. execute / AbortSignal / onUpdate .-> Runtime[运行时层]
    T -. name / description / parameters .-> Protocol[模型协议层]
~~~

### 1.1 <code>pi-ai.Tool</code>：模型可见的能力描述

定义位于 <code>packages/ai/src/types.ts:514</code>：

~~~ts
export interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
  constrainedSampling?: false | ConstrainedSamplingConfig;
}
~~~

它只有模型请求需要的数据：

- <code>name</code>：模型返回 ToolCall 时使用的稳定名称。
- <code>description</code>：告诉模型何时使用这个工具。
- <code>parameters</code>：TypeBox/JSON Schema 参数契约。
- <code>constrainedSampling</code>：请求 Provider 尽量或必须按 schema/grammar 约束生成。

它没有 <code>execute()</code>。因此 <code>pi-ai</code> 只知道如何向模型描述工具、解析 ToolCall 和转换 ToolResult，不负责真实副作用。

### 1.2 <code>AgentTool</code>：Runtime 可执行能力

定义位于 <code>packages/agent/src/types.ts:386</code>：

~~~ts
export interface AgentTool<TParameters extends TSchema, TDetails>
  extends Tool<TParameters> {
  label: string;
  prepareArguments?: (args: unknown) => Static<TParameters>;
  execute(
    toolCallId: string,
    params: Static<TParameters>,
    signal?: AbortSignal,
    onUpdate?: AgentToolUpdateCallback<TDetails>,
  ): Promise<AgentToolResult<TDetails>>;
  executionMode?: "sequential" | "parallel";
}
~~~

相对模型层，它增加了：

- <code>label</code>：UI 显示名称。
- <code>prepareArguments</code>：校验前兼容旧 schema 或模型常见格式错误。
- <code>execute</code>：真实执行逻辑。
- <code>AbortSignal</code>：协作式取消。
- <code>onUpdate</code>：流式工具输出。
- <code>executionMode</code>：决定工具批次能否并发。

<code>AgentToolResult</code> 定义于 <code>packages/agent/src/types.ts:361</code>：

~~~ts
export interface AgentToolResult<T> {
  content: (TextContent | ImageContent)[];
  details: T;
  usage?: Usage;
  addedToolNames?: string[];
  terminate?: boolean;
}
~~~

其中 <code>content</code> 面向模型，<code>details</code> 主要面向 UI、日志和持久化。这个分离对企业 Agent 很重要：机器可读审计信息不必全部暴露给模型。

### 1.3 <code>ToolDefinition</code>：Coding Agent 产品定义

定义位于 <code>packages/coding-agent/src/core/extensions/types.ts:449</code>。它在 <code>AgentTool</code> 能力之上增加：

- <code>promptSnippet</code>：是否在默认 system prompt 的 Available tools 中显示。
- <code>promptGuidelines</code>：工具启用时追加的行为规则。
- <code>renderCall</code>、<code>renderResult</code>：TUI 渲染。
- <code>renderShell</code>：使用标准工具外壳还是自定义外壳。
- <code>execute(..., ctx: ExtensionContext)</code>：让扩展工具访问 cwd、session、UI、model 等产品上下文。

<code>packages/coding-agent/src/core/tools/tool-definition-wrapper.ts:5</code> 的 <code>wrapToolDefinition()</code> 把它适配成 <code>AgentTool</code>。关键字段基本原样透传，UI renderer 和 prompt metadata 不进入 agent-core。

这说明 Pi 的边界是：

~~~text
模型协议字段  → pi-ai
执行语义      → agent-core
产品提示/UI   → coding-agent
~~~

## 2. Tool 如何注册和组装

### 2.1 agent-core 没有全局注册中心

基础 <code>Agent</code> 只保存 <code>state.tools: AgentTool[]</code>。运行开始时，<code>Agent.createContextSnapshot()</code> 在 <code>packages/agent/src/agent.ts:437</code> 对工具数组做一次浅复制：

~~~ts
return {
  systemPrompt: this._state.systemPrompt,
  messages: this._state.messages.slice(),
  tools: this._state.tools.slice(),
};
~~~

因此 agent-core 中的“注册”本质上是依赖注入，不是进程级全局 Registry。

### 2.2 coding-agent 有产品级定义 Registry

<code>AgentSession</code> 会组合三类工具：

1. 内置 ToolDefinition：<code>createAllToolDefinitions()</code>。
2. Extension 工具：<code>pi.registerTool()</code>。
3. SDK 工具：<code>createAgentSession({ customTools })</code>。

组装入口：

- <code>packages/coding-agent/src/core/agent-session.ts:2588</code>：<code>_refreshToolRegistry()</code>
- <code>packages/coding-agent/src/core/agent-session.ts:2681</code>：<code>_buildRuntime()</code>
- <code>packages/coding-agent/src/core/tools/index.ts:156</code>：<code>createAllToolDefinitions()</code>

<code>_refreshToolRegistry()</code> 的主要流程是：

~~~text
内置 ToolDefinition
  + Extension RegisteredTool
  + SDK customTools
  → allowed/excluded 过滤
  → 按 name 建 definitionRegistry
  → wrapRegisteredTools()
  → 按 name 建 _toolRegistry
  → setActiveToolsByName()
  → agent.state.tools
~~~

Extension/SDK 工具使用相同名称时会覆盖 definition registry 和 runtime registry 中的内置工具。这提供了工具替换能力，但也意味着企业系统必须对名称冲突、来源可信度和覆盖权限制定显式策略。

### 2.3 活动工具和已配置工具不同

<code>_toolDefinitions</code> 保存所有可配置工具，<code>agent.state.tools</code> 只保存当前活动工具。<code>setActiveToolsByName()</code> 位于 <code>packages/coding-agent/src/core/agent-session.ts:939</code>，它同时：

1. 从 Registry 中解析名称。
2. 更新 <code>agent.state.tools</code>。
3. 根据活动工具重建 system prompt。

未知名称会被忽略。实际发给模型的是活动工具，不是所有已注册工具。

## 3. Tool schema 如何生成

### 3.1 schema 是运行时对象，不是类型反射结果

以内置 <code>edit</code> 为例，<code>packages/coding-agent/src/core/tools/edit.ts:34</code> 和 <code>:45</code> 直接构造 TypeBox schema：

~~~ts
const replaceEditSchema = Type.Object({
  oldText: Type.String({ description: "..." }),
  newText: Type.String({ description: "..." }),
});

const editSchema = Type.Object({
  path: Type.String({ description: "Path to the file to edit..." }),
  edits: Type.Array(replaceEditSchema, { description: "..." }),
});

export type EditToolInput = Static<typeof editSchema>;
~~~

这里的方向是：

~~~text
TypeBox schema（运行时事实）
  ├─ Provider 请求使用它作为 JSON Schema
  ├─ Runtime 使用它校验 ToolCall
  └─ Static<typeof schema> 推导 TypeScript 类型
~~~

不是：

~~~text
TypeScript interface → 运行时反射 → JSON Schema
~~~

这样避免 TypeScript 类型在编译后被擦除，也避免维护一份 interface 和一份 JSON Schema 产生漂移。

### 3.2 ToolDefinition 不重新生成 schema

<code>createEditToolDefinition()</code> 在 <code>packages/coding-agent/src/core/tools/edit.ts:316</code> 直接设置：

~~~ts
return {
  name: "edit",
  description: "...",
  parameters: editSchema,
  prepareArguments: prepareEditArguments,
  async execute(...) { ... },
};
~~~

<code>wrapToolDefinition()</code> 又把 <code>definition.parameters</code> 原样放入 <code>AgentTool.parameters</code>。正常路径中没有额外的 schema generator。

### 3.3 Provider adapter 做协议转换

同一个 <code>Tool.parameters</code> 会被转换成不同厂商字段：

| Provider | 工具描述格式 | 结果格式 | 源码 |
|---|---|---|---|
| OpenAI Responses | <code>{ type: "function", name, description, parameters, strict }</code> | <code>function_call_output</code> | <code>openai-responses-shared.ts:359</code>、<code>:308</code> |
| Anthropic Messages | <code>{ name, description, input_schema, strict }</code> | user content 中的 <code>tool_result</code> | <code>anthropic-messages.ts:1326</code>、<code>:1130</code> |
| Google Gemini | <code>functionDeclarations[].parametersJsonSchema</code> 或 <code>parameters</code> | user part 中的 <code>functionResponse</code> | <code>google-shared.ts:318</code>、<code>:249</code> |

Provider adapter 解决的是“同一个内部协议如何落到不同厂商 wire format”，不应该反向污染 Tool 的业务实现。

### 3.4 strict schema 是一次派生转换

当 Tool 请求 JSON-schema constrained sampling 时，<code>packages/ai/src/api/constrained-sampling.ts:117</code> 的 <code>makeStrictJsonSchema()</code> 会克隆原 schema，再转换成 Provider strict 子集：

- object 的所有 properties 都进入 <code>required</code>。
- 原可选且非 nullable 字段变成 <code>anyOf: [原类型, null]</code>。
- <code>additionalProperties = false</code>。
- 不支持的 <code>$ref</code>、<code>allOf</code>、<code>oneOf</code> 等结构会使 strict 转换失败。
- <code>strict: "prefer"</code> 可回退为普通 schema；<code>strict: "require"</code> 不允许回退。

内置工具只有在 <code>PI_EXPERIMENTAL=1</code> 时通过 <code>getExperimentalToolSampling()</code> 请求 <code>{ type: "json_schema", strict: "prefer" }</code>。因此“工具有 schema”和“Provider 被要求严格按 schema 采样”是两个不同能力。

## 4. LLM 如何知道有哪些工具

### 4.1 权威通道：Provider 请求的原生 tools 字段

每次模型调用前，<code>streamAssistantResponse()</code> 在 <code>packages/agent/src/agent-loop.ts:281</code> 构造：

~~~ts
const llmContext: Context = {
  systemPrompt: context.systemPrompt,
  messages: llmMessages,
  tools: context.tools,
};
~~~

随后 Provider adapter 把 <code>Context.tools</code> 转为厂商协议。例如 OpenAI Responses 的 <code>buildParams()</code> 在 <code>packages/ai/src/api/openai-responses.ts:313</code> 设置：

~~~ts
params.tools = convertResponsesTools(toolPlacement.immediate, ...);
~~~

这才是 LLM 获得可调用工具名称、描述和参数 schema 的机器协议。

### 4.2 辅助通道：system prompt 的 Available tools

<code>packages/coding-agent/src/core/system-prompt.ts:28</code> 还会把带 <code>promptSnippet</code> 的活动工具写入默认 system prompt：

~~~text
Available tools:
- read: ...
- edit: ...
~~~

这个通道用于补充使用习惯和产品级指导，但不是 ToolCall 的协议定义。直接证据是 <code>packages/coding-agent/test/agent-session-dynamic-tools.test.ts</code>：

- 自定义工具没有 <code>promptSnippet</code> 时，不出现在 system prompt。
- 它仍存在于 <code>agent.state.tools</code>，仍会作为原生 tool schema 发给模型。

因此企业 Agent 应采用：

~~~text
原生 tools schema = 能力契约
system prompt      = 使用策略与行为建议
~~~

不要只把工具说明拼进 prompt，再要求模型输出自定义 JSON 文本。原生 ToolCall 协议更容易流式解析、配对、校验和跨 Provider 适配。

### 4.3 动态工具何时生效

一次 run 开始时，Agent 使用 tools 快照。直接修改 <code>agent.state.tools</code> 不会自动改写已经运行中的 <code>currentContext.tools</code>。

Coding Agent 通过 <code>packages/coding-agent/src/core/agent-session.ts:540</code> 的 <code>_installAgentNextTurnRefresh()</code>，在每个 turn 结束后返回新的：

- system prompt
- tools
- model
- thinking level

因此 Extension 在运行期间注册/启用工具后，可以从下一次 Provider 请求开始生效。

Pi 还支持 <code>AgentToolResult.addedToolNames</code>。支持延迟工具加载的 Provider adapter 可以从某个 ToolResult 位置开始注入新工具，而不是从对话开头就发送全部 schema。这对大型企业工具目录的 token 成本和权限按需开放很有价值。

## 5. ToolCall 参数如何解析

参数处理分成四个阶段。

~~~text
Provider 流式文本
  → JSON 解析/修复
  → 统一 ToolCall { id, name, arguments }
  → prepareArguments 兼容转换
  → TypeBox schema 校验与有限类型转换
  → beforeToolCall 策略检查
  → execute(validatedArgs)
~~~

### 5.1 第一阶段：Provider 流解析

以 OpenAI Responses 为例：

1. <code>response.output_item.added</code> 创建临时 ToolCall block。
2. <code>response.function_call_arguments.delta</code> 把 JSON 片段追加到 <code>partialJson</code>。
3. 每次 delta 调用 <code>parseStreamingJson()</code>，供 UI 实时展示部分参数。
4. <code>response.output_item.done</code> 使用最终 arguments 再解析一次。
5. 删除临时 <code>partialJson</code>，发出 <code>toolcall_end</code>。

对应源码：

- <code>packages/ai/src/api/openai-responses-shared.ts:432</code>
- <code>packages/ai/src/api/openai-responses-shared.ts:652</code>
- <code>packages/ai/src/utils/json-parse.ts:104</code>

Anthropic 也使用同一个 <code>parseStreamingJson()</code> 解析 <code>input_json_delta</code>，位置是 <code>packages/ai/src/api/anthropic-messages.ts:680</code>。

<code>parseStreamingJson()</code> 会依次尝试：

1. 标准 <code>JSON.parse</code>。
2. 修复非法转义和字符串内控制字符后再 parse。
3. 使用 <code>partial-json</code> 解析不完整 JSON。
4. 仍失败则返回空对象。

这一步是“容错解析”，不是业务参数校验。

### 5.2 第二阶段：截断保护

如果 AssistantMessage 的 <code>stopReason === "length"</code>，agent-loop 不会执行其中任何 ToolCall。<code>failToolCallsFromTruncatedMessage()</code> 位于 <code>packages/agent/src/agent-loop.ts:381</code>。

原因是部分 JSON parser 可能把截断参数修复成一个恰好可通过 schema 的对象，但值在语义上已经不完整。例如原字符串 <code>"production-database"</code> 被截成 <code>"production"</code>，类型仍然是合法 string，却可能操作错误目标。

这是一个值得企业 Agent 直接采用的安全规则：

> 模型输出因 token limit 截断时，不能只依赖 JSON 可解析和 schema 可通过来判断能否执行副作用。

### 5.3 第三阶段：prepareArguments

<code>prepareToolCallArguments()</code> 位于 <code>packages/agent/src/agent-loop.ts:586</code>，在 schema 校验前调用 <code>tool.prepareArguments</code>。

Edit 工具用它兼容：

- 旧版顶层 <code>oldText/newText</code>。
- 模型把 <code>edits</code> 数组输出成 JSON 字符串。
- 模型把单个 edit 对象输出成对象而不是单元素数组。

转换后的对象再接受当前公开 schema 校验。这种设计允许兼容旧 transcript 或模型差异，同时保持对模型公开的 schema 干净，不必把所有历史字段永久写入新 schema。

### 5.4 第四阶段：TypeBox 校验与转换

<code>validateToolArguments()</code> 位于 <code>packages/ai/src/utils/validation.ts:317</code>。它执行：

1. <code>structuredClone(toolCall.arguments)</code>，避免直接修改 Provider 解析出的原始 ToolCall。
2. 把 optional 且 non-nullable 字段上的 <code>null</code> 当成“省略”并删除。
3. 使用 TypeBox <code>Value.Convert</code> 做类型转换。
4. 对序列化后的普通 JSON Schema 补充兼容转换。
5. 从 WeakMap 取得或编译 schema validator。
6. <code>validator.Check(args)</code>。
7. 失败时把字段路径、错误原因和原始参数组成错误消息并抛出。

当前转换包括一些实用容错，例如：

- <code>"42" → 42</code>
- <code>"true" → true</code>
- <code>1 → true</code>
- optional non-nullable 字段 <code>null → 删除该字段</code>

因此 execute 收到的是“经过准备、转换并校验”的参数，不一定与 Provider 返回对象逐字段相同。

### 5.5 beforeToolCall 在校验之后

<code>prepareToolCall()</code> 的顺序是：

~~~text
lookup
→ prepareArguments
→ validateToolArguments
→ beforeToolCall
→ execute
~~~

Coding Agent 把 Extension <code>tool_call</code> 事件接到这个 hook，位置是 <code>packages/coding-agent/src/core/agent-session.ts:484</code>。

一个重要细节是：Extension 可以原地修改 <code>event.input</code>，修改后不会再次校验。源码类型注释和 <code>packages/agent/test/agent-loop.test.ts</code> 都明确验证了这个行为。

这给 Extension 很强的改写能力，但在企业安全模型中应谨慎处理：

- 仅做审批或拒绝时，不应修改已校验对象。
- 必须修改参数时，最好修改后再执行 schema 和业务校验。
- 高风险工具可把 hook 输入视为只读，要求策略层返回显式的新参数对象。

## 6. Tool 如何执行

### 6.1 查找和执行模式

<code>executeToolCalls()</code> 位于 <code>packages/agent/src/agent-loop.ts:411</code>。ToolCall 通过 <code>name</code> 在当前 Context 的活动工具中查找。

满足任一条件时，整个 batch 顺序执行：

- <code>config.toolExecution === "sequential"</code>。
- 当前 batch 中任一工具声明 <code>executionMode: "sequential"</code>。

否则默认并行执行。并行模式的精确语义是：

1. lookup、prepare、validation、before hook 仍按 ToolCall 源顺序执行。
2. 允许执行的工具并发运行。
3. <code>tool_execution_end</code> 按真实完成顺序发出。
4. ToolResultMessage 按 AssistantMessage 中的原始 ToolCall 顺序进入 transcript。

这样可以同时获得执行吞吐和稳定上下文顺序。

### 6.2 execute 契约

<code>executePreparedToolCall()</code> 位于 <code>packages/agent/src/agent-loop.ts:670</code>：

~~~ts
const result = await tool.execute(
  toolCall.id,
  validatedArgs,
  signal,
  onUpdate,
);
~~~

工具应该：

- 成功时返回 <code>AgentToolResult</code>。
- 失败时抛异常。
- 长操作响应 <code>AbortSignal</code>。
- 需要实时输出时调用 <code>onUpdate(partialResult)</code>。

<code>onUpdate</code> 只产生 <code>tool_execution_update</code> 事件，不直接写入模型 transcript。工具 Promise settle 之后的迟到 update 会被忽略。

### 6.3 副作用工具需要额外业务校验

TypeBox 只能验证结构。例如 Edit 的 schema 能验证 <code>path</code> 是 string、<code>edits</code> 是数组，却不能仅靠 JSON Schema 保证：

- 文件存在并且可读写。
- <code>oldText</code> 唯一匹配。
- 多个 edit 不重叠。
- 路径位于允许工作区。
- 操作仍未被取消。

因此 <code>createEditToolDefinition().execute()</code> 内部还会执行：

- <code>validateEditInput()</code> 业务校验。
- <code>resolveToCwd()</code> 路径解析。
- 文件访问检查。
- 内容匹配和重叠检查。
- mutation queue 串行化同一文件写入。
- 每个 await 后检查 AbortSignal。

企业 Tool 应保留同样的两层校验：

~~~text
schema validation  = 数据结构合法
domain validation  = 业务操作安全且有意义
~~~

## 7. Tool 执行异常如何处理

### 7.1 错误被转换为 ToolResult 数据

Pi 的默认策略是：单个 Tool 失败不让整个 Agent loop 抛出，而是把错误作为模型可见数据回传。

<code>createErrorToolResult()</code> 位于 <code>packages/agent/src/agent-loop.ts:760</code>：

~~~ts
return {
  content: [{ type: "text", text: message }],
  details: {},
};
~~~

随后 <code>createToolResultMessage()</code> 设置 <code>isError: true</code>。

### 7.2 各错误点的行为

| 失败点 | Runtime 行为 | 是否执行真实 Tool | 是否产生 ToolResult |
|---|---|---:|---:|
| Tool 名称不存在 | <code>Tool X not found</code> | 否 | 是，error |
| Assistant 因 length 截断 | 标记参数可能不完整 | 否 | 是，error |
| <code>prepareArguments</code> 抛错 | 捕获错误文本 | 否 | 是，error |
| schema 校验失败 | 捕获格式化校验错误 | 否 | 是，error |
| <code>beforeToolCall</code> 返回 block | 使用 reason 或默认原因 | 否 | 是，error |
| <code>beforeToolCall</code> 抛错 | 捕获错误文本 | 否 | 是，error |
| Signal 在预检后 aborted | <code>Operation aborted</code> | 否 | 是，error |
| <code>tool.execute()</code> 抛错 | 捕获 Error.message | 已进入执行 | 是，error |
| <code>afterToolCall</code> 抛错 | 用 hook 错误替换原结果 | 已完成 | 是，error |
| <code>tool_result</code> Extension handler 抛错 | ExtensionRunner 记录 ExtensionError，保留当前结果 | 已完成 | 是，沿用原状态 |

Tool 本身如果返回一段“失败文本”但不抛异常，agent-core 会把它视为成功；除非 <code>afterToolCall</code> 显式覆盖 <code>isError</code>。因此 Tool 实现必须统一遵守“失败抛异常”的契约。

### 7.3 为什么错误要回给模型

例如模型调用：

~~~text
read({ path: "missing.ts" })
~~~

ToolResult 返回：

~~~text
isError: true
content: "File not found: missing.ts"
~~~

下一轮模型可以：

- 改用正确路径重试。
- 先调用 find/ls。
- 向用户解释无法完成的原因。

如果 runtime 在第一次工具异常时直接终止 Agent，上述自修复能力就不存在。

### 7.4 afterToolCall 可以重写最终结果

<code>finalizeExecutedToolCall()</code> 位于 <code>packages/agent/src/agent-loop.ts:713</code>。after hook 可覆盖：

- <code>content</code>
- <code>details</code>
- <code>usage</code>
- <code>isError</code>
- <code>terminate</code>

Coding Agent 使用它接入 Extension <code>tool_result</code> hook，并统一规范化工具返回图片。

这适合企业场景中的：

- 结果脱敏。
- 审计字段补充。
- 将第三方错误归一化。
- 输出大小限制。
- DLP（数据防泄漏）扫描。

### 7.5 terminate 不是 error，也不是 abort

ToolResult 可以设置 <code>terminate: true</code>。只有一个 batch 中每个 finalized result 都是 terminate，runtime 才跳过自动下一次 LLM 调用。

它不会：

- 取消当前工具。
- 删除 ToolResult。
- 把结果标记为错误。

它表示“当前工具批次已经决定运行应在此结束”。

## 8. Tool 结果如何进入下一轮 Context

### 8.1 从执行结果到标准消息

<code>createToolResultMessage()</code> 位于 <code>packages/agent/src/agent-loop.ts:777</code>，把内部结果转成 <code>pi-ai.ToolResultMessage</code>：

~~~ts
{
  role: "toolResult",
  toolCallId: toolCall.id,
  toolName: toolCall.name,
  content: result.content ?? [],
  details: result.details,
  usage: result.usage,
  addedToolNames: result.addedToolNames,
  isError,
  timestamp: Date.now(),
}
~~~

关键关联键是 <code>toolCallId</code>。名称用于语义和 Provider 格式，ID 用于把结果精确配对到某一次调用。

### 8.2 写入当前 loop Context

<code>runLoop()</code> 在 <code>packages/agent/src/agent-loop.ts:219</code> 执行：

~~~ts
for (const result of toolResults) {
  currentContext.messages.push(result);
  newMessages.push(result);
}
~~~

如果工具 batch 没有 terminate，<code>hasMoreToolCalls = true</code>，内层 while 开始新 turn，再次调用 <code>streamAssistantResponse()</code>。新的 <code>llmContext.messages</code> 已包含：

~~~text
user
assistant(toolCall)
toolResult
~~~

所以模型下一轮不是通过某个隐藏变量获得工具结果，而是通过标准会话消息看到它。

### 8.3 Provider adapter 再转换成厂商结果协议

| Provider | Assistant ToolCall | ToolResult 回注 |
|---|---|---|
| OpenAI Responses | <code>function_call(call_id, name, arguments)</code> | <code>function_call_output(call_id, output)</code> |
| Anthropic | assistant <code>tool_use(id, name, input)</code> | user <code>tool_result(tool_use_id, content, is_error)</code> |
| Google | model <code>functionCall(name, args, id)</code> | user <code>functionResponse(name, response, id)</code> |

OpenAI Responses 的 wire format 没有单独序列化 Pi 的 <code>isError</code> 字段，只把错误文本作为 <code>function_call_output</code> 回传；Anthropic 和 Google 会把错误状态映射到各自协议。

### 8.4 同时写入 Agent 长期状态

agent-loop 发出 <code>message_start/end(toolResult)</code>。<code>Agent.processEvents()</code> 在 <code>packages/agent/src/agent.ts:544</code> 收到 <code>message_end</code> 后把 ToolResult 追加到 <code>state.messages</code>。

因此有两份不同生命周期的数组：

| 数组 | 用途 |
|---|---|
| <code>currentContext.messages</code> | 当前 run 内下一次模型请求立即使用 |
| <code>agent.state.messages</code> | 跨 prompt 保留的 Agent transcript |

它们通过同一个 <code>message_end</code> 提交点保持一致，而不是共享同一个数组引用。

### 8.5 Coding Agent 还会持久化到 JSONL session

<code>AgentSession._handleAgentEvent()</code> 在 <code>packages/coding-agent/src/core/agent-session.ts:651</code> 监听 <code>message_end</code>。user、assistant、toolResult 都通过 <code>sessionManager.appendMessage()</code> 写入会话树。

完整写入路径是：

~~~text
Tool.execute()
  → AgentToolResult
  → ToolResultMessage
  → currentContext.messages
  → message_end(toolResult)
  → agent.state.messages
  → SessionManager.appendMessage()
  → JSONL session tree
~~~

其中 <code>details</code> 不会被 Provider adapter 当成模型内容发送，但会留在 ToolResultMessage 和 session 中。企业系统仍需把它视为持久化敏感数据，不能因为“模型看不到”就忽略数据分级。

## 9. 一次完整 Tool 回路

~~~mermaid
sequenceDiagram
    actor U as User
    participant S as AgentSession
    participant A as Agent
    participant L as agent-loop
    participant P as Provider Adapter
    participant M as LLM
    participant T as AgentTool
    participant J as SessionManager

    U->>S: prompt()
    S->>A: agent.prompt(user)
    A->>L: context snapshot(tools + messages)
    L->>P: Context { messages, tools }
    P->>M: 厂商 tools schema + messages
    M-->>P: 流式 function/tool call JSON
    P->>P: parseStreamingJson()
    P-->>L: AssistantMessage(toolCall)
    L->>L: prepareArguments()
    L->>L: validateToolArguments()
    L->>S: beforeToolCall / Extension tool_call
    L->>T: execute(id, validatedArgs, signal, onUpdate)
    T-->>L: AgentToolResult
    L->>S: afterToolCall / Extension tool_result
    L->>L: createToolResultMessage()
    L-->>A: message_end(toolResult)
    A->>A: state.messages.push(toolResult)
    S->>J: appendMessage(toolResult)
    L->>P: 下一 turn Context 含 toolResult
    P->>M: function_call_output / tool_result / functionResponse
    M-->>P: final assistant text 或下一次 tool call
    P-->>L: AssistantMessage
~~~

用最小 transcript 表示：

~~~text
Turn 1 request:
  user("修改 README")
  tools=[edit schema]

Turn 1 response:
  assistant(toolCall id=call_1, name=edit, arguments={...})

Local execution:
  toolResult(toolCallId=call_1, isError=false, content="Successfully replaced...")

Turn 2 request:
  user("修改 README")
  assistant(toolCall call_1)
  toolResult(call_1)

Turn 2 response:
  assistant("README 已修改。")
~~~

## 10. 企业 Agent 开发可以复用的设计

### 10.1 把 Tool 分成描述面、控制面和执行面

推荐保持三类数据分离：

| 平面 | 内容 | Pi 对应实现 |
|---|---|---|
| 描述面 | name、description、schema、Provider constrained sampling | <code>pi-ai.Tool</code> |
| 控制面 | allowlist、权限、审批、before/after hook、并发策略、超时和审计 | <code>AgentSession</code> + hooks |
| 执行面 | 真实 API、数据库、文件或任务系统副作用 | <code>AgentTool.execute()</code> |

这样可以在不修改业务 Tool 的情况下接入权限、脱敏和审计，也可以在不修改 Agent loop 的情况下替换 Provider。

### 10.2 三道参数防线缺一不可

~~~text
Provider constrained sampling
  ↓ 降低非法输出概率
JSON/stream parser
  ↓ 得到统一对象
本地 schema + domain validation
  ↓ 决定是否允许真实执行
~~~

strict schema 不能取代本地验证。模型和 Provider 都属于进程外输入，服务端也可能存在兼容模式或 bug。

### 10.3 错误应“模型可恢复”，但不能泄漏内部细节

Pi 把 Tool 错误回注模型，这支持自修复。企业实现应进一步把异常拆成：

- <code>code</code>：稳定错误码，如 <code>NOT_FOUND</code>、<code>PERMISSION_DENIED</code>。
- <code>retryable</code>：模型是否值得重试。
- <code>modelMessage</code>：可安全发送给模型的简短错误。
- <code>internalDetails</code>：仅日志/审计可见的 stack、request id、第三方响应。

Pi 核心当前主要使用错误字符串和 <code>isError</code>，没有统一的结构化 Tool 错误分类。企业封装层应补上这一层，避免把数据库 SQL、内部 URL、token 或堆栈直接发给模型。

### 10.4 高风险 Tool 要求幂等和审批

ToolCall ID 可以用于跟踪一次调用，但 Pi 核心没有提供跨进程的幂等去重存储。支付、发消息、删除数据、发布配置等工具应增加：

- 业务幂等键。
- 调用状态表。
- 重复调用去重。
- 人工审批或双重确认。
- 最小权限凭据。
- 明确的 dry-run/preview。

<code>beforeToolCall</code> 适合做策略入口，但审批状态和幂等事实应存入可靠持久化系统，而不是只存在内存 hook 中。

### 10.5 并发必须由资源语义决定

默认并行适合互不影响的查询；以下工具应顺序执行或按资源键串行化：

- 写同一个文件。
- 更新同一订单。
- 修改同一数据库实体。
- 依赖前一调用副作用的操作。

Pi 支持 Tool 级 <code>executionMode: "sequential"</code>，Edit 还使用文件 mutation queue。这说明“Agent 可以并发调用 Tool”与“业务资源允许并发写入”是两层不同判断。

### 10.6 区分模型结果和内部详情

推荐结果结构：

~~~ts
interface EnterpriseToolResult<TDetails> {
  content: ModelVisibleContent[];
  details: TDetails;
  audit: {
    toolCallId: string;
    durationMs: number;
    actor: string;
    policyDecisionId?: string;
  };
}
~~~

模型只看到完成任务必要的数据。原始响应、PII、内部标识和调试信息进入 details/audit，并接受独立的数据保留策略。

## 11. Pi 当前设计的优点与需要补强的地方

### 11.1 值得复用的部分

1. Tool 描述与执行分层，Provider adapter 不依赖业务实现。
2. TypeBox schema 同时服务运行时 JSON Schema 和 TypeScript 类型推导。
3. Provider 参数解析与本地 schema 校验分离。
4. 参数校验失败也回注模型，支持自动修复。
5. 截断 ToolCall 禁止执行，避免“类型合法但语义被截断”。
6. before/after hook 提供策略和结果治理边界。
7. ToolResult 按 call ID 与 ToolCall 配对。
8. 并行执行与稳定 transcript 顺序兼得。
9. 模型可见 content 与 UI/日志 details 分离。
10. ToolResult 同时进入当前 Context、Agent state 和 durable session。

### 11.2 企业实现通常需要额外增加

1. 结构化错误码、retryable 分类和安全错误映射。
2. Tool 级 timeout、重试、熔断和限流策略。
3. 跨进程幂等、去重和长任务状态机。
4. RBAC/ABAC、审批流和凭据最小权限。
5. schema 版本与旧 transcript 迁移策略。
6. Tool 输出大小上限、DLP 和敏感字段脱敏。
7. 审计事件的持久化与不可抵赖性。
8. hook 修改参数后的重新校验。
9. 对 Tool 来源、同名覆盖和动态启用的信任治理。

这些不是所有 Agent 都必须内置到核心循环的复杂度，但对生产企业 Agent 通常是必要的宿主层能力。

## 12. 一个符合 Pi 设计的最小自定义 Tool

~~~ts
import { Type, type Static } from "typebox";
import type { ToolDefinition } from "@earendil-works/pi-coding-agent";

const getOrderSchema = Type.Object({
  orderId: Type.String({
    minLength: 1,
    description: "Internal order identifier",
  }),
});

type GetOrderInput = Static<typeof getOrderSchema>;

export const getOrderTool: ToolDefinition<
  typeof getOrderSchema,
  { requestId: string }
> = {
  name: "get_order",
  label: "Get order",
  description: "Get the current status of one order.",
  parameters: getOrderSchema,
  executionMode: "parallel",
  async execute(toolCallId, params: GetOrderInput, signal) {
    const response = await orderService.get(params.orderId, {
      signal,
      requestId: toolCallId,
    });

    return {
      content: [
        {
          type: "text",
          text: JSON.stringify({
            orderId: response.orderId,
            status: response.status,
          }),
        },
      ],
      details: { requestId: response.requestId },
    };
  },
};
~~~

这个例子保留了四个关键原则：

- schema 是运行时单一事实源。
- execute 只接收已验证参数。
- ToolCall ID 进入下游 request id，便于追踪和幂等扩展。
- 模型只看到必要订单字段，内部 request id 留在 details。

## 13. 源码阅读顺序

第一次验证 Tool 主链时，建议按以下顺序打断点：

1. <code>packages/coding-agent/src/core/tools/edit.ts:316</code>：一个完整 ToolDefinition。
2. <code>packages/coding-agent/src/core/tools/tool-definition-wrapper.ts:5</code>：产品定义适配为 AgentTool。
3. <code>packages/coding-agent/src/core/agent-session.ts:2588</code>：Registry 和活动工具组装。
4. <code>packages/agent/src/agent.ts:437</code>：创建本次运行的工具快照。
5. <code>packages/agent/src/agent-loop.ts:281</code>：工具放入 LLM Context。
6. <code>packages/ai/src/api/openai-responses.ts:313</code>：转换成 Provider tools 参数。
7. <code>packages/ai/src/api/openai-responses-shared.ts:432</code>：解析模型 ToolCall。
8. <code>packages/agent/src/agent-loop.ts:600</code>：lookup、prepare、validate、before hook。
9. <code>packages/agent/src/agent-loop.ts:670</code>：真实 execute。
10. <code>packages/agent/src/agent-loop.ts:713</code>：after hook。
11. <code>packages/agent/src/agent-loop.ts:777</code>：构造 ToolResultMessage。
12. <code>packages/agent/src/agent-loop.ts:219</code>：结果进入下一轮 Context。
13. <code>packages/coding-agent/src/core/agent-session.ts:651</code>：结果持久化。

## 14. 测试证据

| 行为 | 测试 |
|---|---|
| ToolCall 执行并产生 ToolResult | <code>packages/agent/test/agent-loop.test.ts:274</code> |
| length 截断的 ToolCall 不执行 | <code>packages/agent/test/agent-loop.test.ts:371</code> |
| before hook 修改参数后不重新校验 | <code>packages/agent/test/agent-loop.test.ts:444</code> |
| prepareArguments 在 validation 前生效 | <code>packages/agent/test/agent-loop.test.ts:506</code> |
| 并行完成事件和 ToolResult 持久化顺序不同 | <code>packages/agent/test/agent-loop.test.ts:586</code> |
| 单个 sequential 工具强制整个 batch 顺序执行 | <code>packages/agent/test/agent-loop.test.ts:870</code> |
| ToolResult 已在 shouldStopAfterTurn 的 Context 中 | <code>packages/agent/test/agent-loop.test.ts:1104</code> |
| Provider 临时 partialJson 不进入持久化 ToolCall | <code>packages/ai/test/openai-responses-partial-json-cleanup.test.ts:67</code> |
| 参数转换、optional null 和 validation failure | <code>packages/ai/test/validation.test.ts:36</code> |
| Google schema 的 Provider 特定清理和 strict mode | <code>packages/ai/test/google-shared-convert-tools.test.ts:17</code>、<code>:187</code> |
| 内置工具仅在实验模式请求 strict-prefer | <code>packages/coding-agent/test/experimental-tool-strict-mode.test.ts:26</code> |
| Edit legacy/stringified 参数准备 | <code>packages/coding-agent/test/edit-tool-legacy-input.test.ts:27</code>、<code>:93</code> |
| 动态工具、promptSnippet 和活动工具行为 | <code>packages/coding-agent/test/agent-session-dynamic-tools.test.ts:99</code>、<code>:211</code> |

当前工作区未安装 <code>node_modules</code>，因此本文依据源码和现有测试交叉分析，未执行 Vitest。

## 15. 最终心智模型

可以用下面六句话记住 Pi 的 Tool 系统：

1. <code>Tool</code> 描述模型能力，<code>AgentTool</code> 执行能力，<code>ToolDefinition</code> 提供产品体验。
2. TypeBox schema 是运行时单一事实源，TypeScript 参数类型从它推导。
3. LLM 通过 Provider 原生 tools 字段获得能力，system prompt 只是辅助说明。
4. Provider 负责把流式 JSON 解析成 ToolCall，agent-core 负责本地准备、校验和策略检查。
5. Tool 失败通常被转换成 error ToolResult，让模型有机会修正，而不是直接终止运行。
6. ToolResult 通过 call ID 回注标准消息序列，同时进入当前 Context、Agent state 和 Session 持久化。
