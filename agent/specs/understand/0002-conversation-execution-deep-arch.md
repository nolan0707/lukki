# Claude-code-open 对话执行架构深度解读

## 1. 文档定位

本文面向 AI 应用架构师、Agent Runtime 设计者、算法/系统工程专家。

目标不是解释“这段代码怎么跑起来”，而是系统性回答以下问题：

1. Claude-code-open 的对话执行链采用了怎样的 runtime 分层？
2. `QueryEngine -> query.ts -> services/api/claude.ts -> services/tools/*` 的边界是如何划定的？
3. 该架构如何把 LLM 推理、工具调用、权限控制、上下文治理、错误恢复、持久化收敛为一个统一执行系统？
4. 它有哪些关键接口、核心状态结构、状态迁移机制与设计约束？
5. 哪些部分体现了明显的工业级工程设计，而不只是功能堆叠？

本文建立在以下两份已有文档之上：

- [0001-project-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0001-project-arch.md)
- [0002-conversation-execution-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0002-conversation-execution-arch.md)

并进一步基于以下关键代码阅读整理：

- `src/QueryEngine.ts`
- `src/query.ts`
- `src/query/config.ts`
- `src/query/deps.ts`
- `src/query/tokenBudget.ts`
- `src/query/stopHooks.ts`
- `src/services/api/claude.ts`
- `src/services/api/client.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/toolExecution.ts`
- `src/services/tools/StreamingToolExecutor.ts`
- `src/services/tools/toolHooks.ts`
- `src/utils/processUserInput/processUserInput.ts`
- `src/utils/processUserInput/processTextPrompt.ts`
- `src/utils/messages.ts`
- `src/utils/toolResultStorage.ts`
- `src/services/compact/autoCompact.ts`
- `src/services/compact/microCompact.ts`
- `src/Tool.ts`
- `src/types/permissions.ts`

---

## 2. 核心判断

Claude-code-open 的“对话执行架构”本质上不是普通聊天式 request/response，而是一个**消息驱动、可恢复、可扩展的 Agent Turn Runtime**。

它有 5 个显著特征：

1. **会话态与 turn 态分离**  
   `QueryEngine` 管会话生命周期，`query.ts` 管单个 turn 的循环状态机。

2. **消息链是一等公民**  
   所有能力最终都映射成消息变换：用户输入、tool_use、tool_result、hook 输出、compact boundary、系统错误、resume 恢复，都围绕消息链收敛。

3. **对外接口稳定，对内执行多阶段**  
   对上层入口看起来只是 `submitMessage()`；对内部 runtime 来说，实际是“输入预处理 -> 上下文治理 -> LLM sampling -> 工具调度 -> 恢复/压缩 -> stop hooks -> 持久化”的多阶段流水线。

4. **错误被设计成 runtime 的正常分支，而不是异常路径**  
   `prompt_too_long`、`max_output_tokens`、`streaming fallback`、`tool_result mismatch`、`user interrupt` 都有显式恢复路径。

5. **协议正确性优先于功能继续**  
   尤其体现在 `ensureToolResultPairing()`、`normalizeMessagesForAPI()`、`stripExcessMediaItems()`、schema validation、permission result typing 上。系统宁可修复/丢弃/降级，也不能把非法协议状态继续推给模型 API。

---

## 3. 设计总图

```text
入口层
  REPL / SDK / Remote / Direct Connect
    -> QueryEngine.submitMessage()

输入建模层
  processUserInput / processTextPrompt
    -> UserMessage / AttachmentMessage / local command result

会话编排层
  QueryEngine
    -> system prompt 装配
    -> ToolUseContext 构造
    -> transcript/usage/result 输出协调

turn 执行层
  query.ts
    -> 上下文治理
    -> sampling
    -> tool loop
    -> 恢复逻辑
    -> stop hooks

模型通信层
  services/api/claude.ts
    -> Anthropic/Bedrock/Vertex/Foundry
    -> streaming / non-streaming fallback
    -> prompt cache / beta headers / output_config

工具运行层
  toolOrchestration.ts
  toolExecution.ts
  StreamingToolExecutor.ts
  toolHooks.ts

上下文治理层
  autoCompact / microCompact / contextCollapse / toolResultBudget / snip

持久化与外部可观察层
  sessionStorage / history / telemetry / tracing / content replacement records
```

最重要的理解点：

- `QueryEngine` 不是 query loop
- `query.ts` 不是模型调用器
- `claude.ts` 不是“整个对话系统”
- 工具执行不是 query 的附属小步骤，而是同等级子系统

---

## 4. 分层职责边界

## 4.1 `processUserInput`: 输入语义归一层

职责：

- 解析 slash command 与普通 prompt
- 将原始输入翻译为 `Message[]`
- 决定 `shouldQuery`
- 注入附件、图片、hook 结果、local command 输出

关键输出：

```ts
type ProcessUserInputBaseResult = {
  messages: Message[]
  shouldQuery: boolean
  allowedTools?: string[]
  model?: string
  effort?: EffortValue
  resultText?: string
}
```

边界含义：

- 它负责“输入理解”，不负责 turn 执行
- 它可以完全短路模型查询
- 它是本地命令系统与 LLM runtime 的分流点

这是一种非常关键的工程设计：把“是否进入 LLM”前置成一个显式判定，而不是在 query loop 中临时分叉。

## 4.2 `QueryEngine`: 会话编排器

职责：

- 维护 `mutableMessages`
- 持有累计 `usage`、`permissionDenials`、`readFileState`
- 生成 system prompt / userContext / systemContext
- 构造 `ToolUseContext`
- 负责 transcript 记录时机
- 负责将 `query()` 的流式内部事件翻译为 SDK 输出

关键类型：

```ts
type QueryEngineConfig = {
  cwd: string
  tools: Tools
  commands: Command[]
  mcpClients: MCPServerConnection[]
  agents: AgentDefinition[]
  canUseTool: CanUseToolFn
  getAppState: () => AppState
  setAppState: (f: (prev: AppState) => AppState) => void
  initialMessages?: Message[]
  readFileCache: FileStateCache
  customSystemPrompt?: string
  appendSystemPrompt?: string
  userSpecifiedModel?: string
  fallbackModel?: string
  thinkingConfig?: ThinkingConfig
  maxTurns?: number
  maxBudgetUsd?: number
  taskBudget?: { total: number }
  jsonSchema?: Record<string, unknown>
  verbose?: boolean
  replayUserMessages?: boolean
  includePartialMessages?: boolean
}
```

设计要点：

- 这是典型“composition root inside runtime”
- 不直接依赖 UI 实现
- 通过注入 `getAppState/setAppState` 与外界解耦
- 可被 REPL、SDK、remote、headless 共用

## 4.3 `query.ts`: 单个 turn 的状态机

职责：

- 管理一次 turn 内部循环
- 多轮调用模型
- 驱动工具执行
- 执行上下文治理与恢复逻辑
- 在自然结束前执行 stop hooks

关键接口：

```ts
export type QueryParams = {
  messages: Message[]
  systemPrompt: SystemPrompt
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  canUseTool: CanUseToolFn
  toolUseContext: ToolUseContext
  fallbackModel?: string
  querySource: QuerySource
  maxOutputTokensOverride?: number
  maxTurns?: number
  skipCacheWrite?: boolean
  taskBudget?: { total: number }
  deps?: QueryDeps
}
```

边界含义：

- query 只关心“当前 turn 如何推进”
- 不负责 transcript 管理策略
- 不关心 UI 最终渲染格式
- 不直接构造系统提示，只消费已经构造好的 prompt/context

## 4.4 `services/api/claude.ts`: 模型通信与协议装配层

职责：

- 选择 provider 与 client
- 构建 API 参数
- 处理 prompt caching / task_budget / effort / thinking / betas
- 执行 streaming 与 non-streaming fallback
- 产出标准 `AssistantMessage`
- 维护 usage / cost / request chain telemetry

关键接口：

```ts
export type Options = {
  getToolPermissionContext: () => Promise<ToolPermissionContext>
  model: string
  toolChoice?: BetaToolChoiceTool | BetaToolChoiceAuto
  isNonInteractiveSession: boolean
  extraToolSchemas?: BetaToolUnion[]
  maxOutputTokensOverride?: number
  fallbackModel?: string
  onStreamingFallback?: () => void
  querySource: QuerySource
  agents: AgentDefinition[]
  allowedAgentTypes?: string[]
  hasAppendSystemPrompt: boolean
  fetchOverride?: ClientOptions['fetch']
  enablePromptCaching?: boolean
  skipCacheWrite?: boolean
  temperatureOverride?: number
  effortValue?: EffortValue
  mcpTools: Tools
  hasPendingMcpServers?: boolean
  queryTracking?: QueryChainTracking
  agentId?: AgentId
  outputFormat?: BetaJSONOutputFormat
  fastMode?: boolean
  advisorModel?: string
  addNotification?: (notif: Notification) => void
  taskBudget?: { total: number; remaining?: number }
}
```

这是一个高度工业化的 API facade，而不是简单 SDK wrapper。

## 4.5 `services/tools/*`: 工具运行子系统

内部进一步分层：

- `toolOrchestration.ts`：批次切分与并发调度
- `toolExecution.ts`：单工具执行管线
- `StreamingToolExecutor.ts`：流式增量执行器
- `toolHooks.ts`：工具生命周期切面

这表明工具执行被视为一等 runtime，不是“模型输出后的 callback”。

---

## 5. 核心状态结构

## 5.1 `ToolUseContext`: 工具运行时世界

`ToolUseContext` 是这套架构里最关键的数据结构。

它不仅是工具调用参数容器，而是“当前 conversation thread 的运行时视图”。

核心字段可分为 7 类：

### A. 能力与配置

- `options.commands`
- `options.tools`
- `options.mainLoopModel`
- `options.thinkingConfig`
- `options.mcpClients`
- `options.agentDefinitions`
- `options.querySource?`

### B. 状态访问能力

- `getAppState()`
- `setAppState()`
- `setAppStateForTasks?()`

### C. 生命周期控制

- `abortController`
- `setInProgressToolUseIDs()`
- `setHasInterruptibleToolInProgress?()`
- `setResponseLength()`

### D. 上下文与消息

- `messages`
- `readFileState`
- `nestedMemoryAttachmentTriggers`
- `loadedNestedMemoryPaths`
- `dynamicSkillDirTriggers`
- `discoveredSkillNames`

### E. UI / 交互桥接

- `setToolJSX?`
- `addNotification?`
- `appendSystemMessage?`
- `sendOSNotification?`
- `requestPrompt?`

### F. 跨子系统状态

- `updateFileHistoryState()`
- `updateAttributionState()`
- `handleElicitation?`
- `contentReplacementState?`

### G. Agent / tracing / identity

- `agentId?`
- `agentType?`
- `toolUseId?`
- `queryTracking?`
- `renderedSystemPrompt?`

这套结构的意义在于：工具执行可以访问完整 conversation runtime，而不仅是一次纯函数调用输入。

这也是工业级 Agent Runtime 和简单 function-calling client 的根本差异。

## 5.2 `State`: query loop 局部状态机

```ts
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined
}
```

解释：

- `messages`：下一次模型调用要看的消息视图，不一定等于会话全量消息
- `toolUseContext`：下一次工具执行时使用的上下文快照
- `autoCompactTracking`：连续 compact 链路状态
- `maxOutputTokensRecoveryCount`：continuation 重试计数
- `hasAttemptedReactiveCompact`：reactive compact circuit breaker
- `pendingToolUseSummary`：跨轮 deferred summary future
- `stopHookActive`：是否已进入 stop hook blocking retry
- `transition`：上一轮为何继续，用于调试、测试和行为约束

这是一个非常标准的显式状态机设计。  
不是在一个大函数里靠局部变量“自然流动”，而是把每个继续分支都变成对 `State` 的明确重建。

## 5.3 `PermissionResult` / `PermissionDecisionReason`

权限系统不是简单布尔值，而是强类型决策对象。

```ts
type PermissionResult<Input> =
  | PermissionDecision<Input>
  | {
      behavior: 'passthrough'
      message: string
      decisionReason?: PermissionDecisionReason
      suggestions?: PermissionUpdate[]
      blockedPath?: string
      pendingClassifierCheck?: PendingClassifierCheck
    }
```

```ts
type PermissionDecisionReason =
  | { type: 'rule'; rule: PermissionRule }
  | { type: 'mode'; mode: PermissionMode }
  | { type: 'subcommandResults'; reasons: Map<string, PermissionResult> }
  | { type: 'permissionPromptTool'; permissionPromptToolName: string; toolResult: unknown }
  | { type: 'hook'; hookName: string; hookSource?: string; reason?: string }
  | { type: 'asyncAgent'; reason: string }
  | { type: 'sandboxOverride'; reason: 'excludedCommand' | 'dangerouslyDisableSandbox' }
  | { type: 'classifier'; classifier: string; reason: string }
  | { type: 'workingDir'; reason: string }
  | { type: 'safetyCheck'; reason: string; classifierApprovable: boolean }
  | { type: 'other'; reason: string }
```

这是一个明显的工业级设计：

- 权限不是 `allow/deny` 二值，而是**可解释、可追踪、可被下游日志/OTel 消费的决策对象**
- 这让系统能够在 runtime、telemetry、UI、hooks 之间保持一致语义

## 5.4 消息类型体系

虽然 vendor 快照未直接展开 `src/types/message.ts`，但从 `utils/messages.ts` 和调用侧可以稳定推导出执行期消息体系：

- `UserMessage`
- `AssistantMessage`
- `AttachmentMessage`
- `ProgressMessage`
- `SystemMessage`
- `ToolUseSummaryMessage`
- `TombstoneMessage`

其中最重要的抽象是：

- `tool_result` 被编码成 `UserMessage`
- `tool_use` 被编码在 `AssistantMessage.message.content[]`
- `ProgressMessage` 不参与 API 对话，只参与 UI/runtime
- `AttachmentMessage` 是 side-channel 语义载体

这是一种“主消息流 + 附件/进度旁路”的设计。

---

## 6. turn 状态机与状态迁移

## 6.1 单次 turn 的标准闭环

标准闭环如下：

```text
S0: 接收输入
  -> processUserInput

S1: 构造会话上下文
  -> QueryEngine 组装 prompt/context/tools

S2: 预处理消息视图
  -> budget / snip / microcompact / collapse / autocompact

S3: 调用模型
  -> streaming or fallback non-streaming

S4: 收集 assistant messages
  -> tool_use?
    yes -> S5
    no  -> S7

S5: 工具调度与执行
  -> toolOrchestration / StreamingToolExecutor
  -> 生成 tool_result user messages

S6: 继续下一轮
  -> 把 assistant + tool_result 拼回 messages
  -> return to S2

S7: stop hooks / token budget / completion
  -> 结束 turn
```

## 6.2 为什么 `query.ts` 在每轮前都重做上下文治理

因为一次 turn 内，工具执行后消息上下文会变化很大：

- 新增 `tool_result`
- 新增 hook attachment
- 新增 compact boundary
- token 数量突增

所以系统不能只在 turn 开头 compact 一次，而必须在每次继续调用模型前重新治理上下文。

这意味着 `query.ts` 不是线性 pipeline，而是**带治理环节的循环执行系统**。

## 6.3 `transition` 的意义

`transition` 记录上一轮为什么 `continue`：

- `collapse_drain_retry`
- `reactive_compact_retry`
- `max_output_tokens_escalate`
- `max_output_tokens_recovery`
- `stop_hook_blocking`

它的价值在于：

- 让恢复路径显式化
- 避免重复恢复导致的死循环
- 为测试与调试提供行为证据

这是一种低成本但高收益的状态机可观察性设计。

---

## 7. 输入到 query 的接口收敛

## 7.1 `processUserInput` 的分流价值

这层的关键创新不是解析字符串，而是将“本地控制平面”和“LLM 对话平面”统一到一个接口下，再通过 `shouldQuery` 显式分流。

优点：

- slash commands 与 prompt 共享输入入口
- 本地命令也可以产生消息并进入 transcript
- 上层 `QueryEngine` 无需区分输入来源

这是一种很强的 runtime 收敛设计。

## 7.2 立即持久化用户消息

`QueryEngine.submitMessage()` 在进入 query loop 前，就对用户消息执行 `recordTranscript()`。

其工程意义非常大：

- 避免“请求发出后、模型未回前进程被杀”导致 resume 失败
- 让会话的持久化边界从“响应完成”前移到“输入接收成功”

对于协作式 IDE/remote 场景，这是比算法更重要的稳定性设计。

---

## 8. 模型请求装配：`services/api/claude.ts`

## 8.1 它不是 SDK adapter，而是请求编译器

`claude.ts` 的核心不是简单调用 `anthropic.beta.messages.create()`，而是将上游 runtime 状态“编译”为一个满足多种约束的请求：

- provider 兼容
- 模型能力兼容
- betas/header 兼容
- cache strategy 兼容
- structured outputs 兼容
- tool search / defer loading 兼容
- effort / thinking / task_budget 兼容
- previousRequestId / fingerprint / telemetry 兼容

所以更准确的说法是：

**`claude.ts` 是 LLM sampling request compiler。**

## 8.2 请求参数装配的关键约束

### 约束 1：tool search 是模型相关、动态的

`useToolSearch` 不是简单 flag，而是根据：

- model support
- permission mode
- available deferred tools
- pending MCP servers

动态决定。

这意味着：**工具集合不是静态 prompt 的一部分，而是 runtime 可变视图。**

### 约束 2：prompt caching 不是开关，而是策略

缓存相关决策涉及：

- `getPromptCachingEnabled(model)`
- `getCacheControl()`
- 1h TTL eligibility
- global cache scope
- cache editing beta
- cached microcompact pinned edits

这里显然不是“是否缓存”的问题，而是“以什么粒度和语义缓存”。

### 约束 3：thinking / effort / max_tokens 相互制约

代码里有明确约束：

- `max_tokens > thinking.budget_tokens`
- 某些模型支持 adaptive thinking，某些不支持
- 非 streaming fallback 需要重新裁剪 `max_tokens`

这表明 runtime 已把模型协议约束编码进装配层。

### 约束 4：request payload 必须 API 合法且 cache 稳定

因此会调用：

- `normalizeMessagesForAPI()`
- `ensureToolResultPairing()`
- `stripExcessMediaItems()`
- `stripAdvisorBlocks()`

这说明 payload 构造的目标不是“尽量保留原始消息”，而是“保证合法、稳定、可缓存”。

## 8.3 `normalizeMessagesForAPI()` 的工业价值

这是消息层最重要的协议编译函数。

它做的事情包括：

- 附件上浮重排
- virtual 消息剔除
- synthetic API error 剔除
- 连续 user 消息合并
- 工具引用剥离或修正
- oversized media error 后的块剥离

从架构角度看，这个函数实现了：

**UI 消息域 -> API 协议消息域 的显式映射**

很多 Agent 系统把这层做得很弱，导致 UI 边界和协议边界混杂。Claude-code-open 在这里做得非常稳。

## 8.4 `ensureToolResultPairing()` 的协议一致性设计

该函数执行：

- forward repair：缺失 tool_result 时补 synthetic result
- reverse repair：孤立 tool_result 时移除
- strict mode：在 HFI 等场景直接 fail fast

这个设计非常值得注意：

- 它优先保证协议一致性
- 其次才是业务连续性
- 并且提供严格模式给数据采集/训练场景

这体现出系统既服务交互 runtime，也服务 trajectory/data 质量约束。

---

## 9. 工具执行流水线

## 9.1 执行管线

单个工具调用的完整流程是：

```text
tool_use block
  -> findToolByName
  -> inputSchema.safeParse
  -> validateInput
  -> PreToolUse hooks
  -> permission resolution
  -> tool.call
  -> PostToolUse hooks / failure hooks
  -> mapToolResultToToolResultBlockParam
  -> processToolResultBlock / persistence
  -> createUserMessage(tool_result)
```

注意这里是“多阶段有状态管线”，而不是简单 `tool(input) -> result`。

## 9.2 schema validation 与 business validation 分离

分两层的原因非常专业：

- `inputSchema.safeParse`：解决格式/类型问题
- `tool.validateInput`：解决业务语义约束

这样可以把“模型参数生成错误”和“业务规则不允许”区分开。

这对于：

- prompt 调优
- 模型行为分析
- telemetry 分类

都非常重要。

## 9.3 Permission 不是 side effect，而是 first-class outcome

在 `toolExecution.ts` 中，权限结果被当作正式执行结果处理：

- `allow`：继续执行
- `deny` / `ask`：生成 `tool_result` 错误消息返回模型

这有两个深层价值：

1. 保持执行链闭环，不需要抛出流程外异常
2. 允许模型在被拒绝后进行策略调整

这就是 Agent 系统和传统 RPC 系统的根本区别之一。

## 9.4 Hooks 是切面，不是 callback

`toolHooks.ts` 不是简单“工具完成后顺便通知一下”，而是生命周期切面：

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`

并且 hooks 本身可以：

- 产出 attachment message
- 产出 blocking error
- 更新 MCP 输出
- 阻止 continuation

这相当于在工具执行流水线中嵌入了可编排 policy plane。

---

## 10. 工具并发模型

## 10.1 `partitionToolCalls()`：基于语义的批次划分

算法目标：

- 最大化只读工具并发
- 保留副作用工具顺序与互斥

其本质不是一般并发控制，而是：

**根据工具语义做局部并行执行规划。**

这是一个很实用的 Agent orchestration 模式。

## 10.2 `StreamingToolExecutor`：与采样并行的执行器

这是架构里的一个亮点。

传统 function-calling 流程通常是：

1. 等模型完整返回
2. 收集所有工具调用
3. 执行工具

而这里支持：

1. 模型 streaming 中出现 `tool_use`
2. 立即尝试启动工具
3. progress 先行回流
4. 结果按接收顺序产出

也就是说，runtime 在做**采样和执行的流水线重叠**。

从系统设计角度，它在降低：

- end-to-end latency
- tool idle time
- 长工具链等待成本

## 10.3 失败传播不是统一广播

`StreamingToolExecutor` 中只有 Bash 错误会中止兄弟工具。

这是非常工程化的语义优化：

- Shell 命令常有依赖链
- 读类工具更接近数据并行

这说明作者没有偷懒用统一失败策略，而是根据工具域特性做了 differentiated failure propagation。

---

## 11. 上下文治理子系统

## 11.1 query loop 前治理，而不是失败后治理

在每轮 sampling 前，系统会依次尝试：

- aggregate tool result budget enforcement
- history snip
- microcompact
- context collapse
- autocompact

这说明架构不是“超了再补救”，而是：

**每轮调用前做一次上下文可发送性编译。**

这是非常像数据库 optimizer / compiler pipeline 的思路。

## 11.2 `autoCompact.ts`：阈值治理 + circuit breaker

几个设计点很关键：

- context window 先减去输出保留量
- auto compact threshold 不是 hard limit
- 有 `MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`
- 某些 querySource 显式禁用 autocompact，避免递归/互相破坏

这说明 autocompact 不是一个“能开就开”的功能，而是被视为一种高代价恢复机制。

## 11.3 `microCompact.ts`：局部清理而非全局摘要

microcompact 的设计目标不是总结整个会话，而是：

- 优先处理旧 tool result
- 使用时间阈值或 cached microcompact
- 保持上下文尽量结构化

其中 cached microcompact 非常有代表性：

- 不直接改本地 message content
- 而是在 API 层通过 `cache_edits` / `cache_reference` 对缓存前缀做手术

这是一个很先进的工业化技巧：

**通过 server-side cache editing 降低上下文体积，同时尽量不破坏 prompt cache 命中。**

## 11.4 `toolResultStorage.ts`：大结果外置

大 tool result 不直接粗暴截断，而是：

1. 落盘到 session 专属目录
2. 在消息中替换为 preview + persisted-output reference

这是一个很典型的“context externalization”策略。

价值：

- 降低 token 污染
- 保留结果可追溯性
- 保持模型仍有 preview 可用

相比纯截断，这是一种更适合 coding agent 的工程设计。

---

## 12. 错误恢复机制

## 12.1 `prompt_too_long` 恢复链

恢复优先级：

1. `contextCollapse.recoverFromOverflow()`
2. `reactiveCompact.tryReactiveCompact()`
3. 最终暴露错误

这种顺序很有讲究：

- 先尝试保留结构化上下文
- 不行再退化成摘要式 compact

这体现出系统在“延迟/稳定性”之外，还在优化“上下文信息保真度”。

## 12.2 `max_output_tokens` 恢复链

恢复策略：

1. 尝试提高 `maxOutputTokensOverride`
2. 若仍不够，则插入 continuation meta message
3. 限定最大恢复次数

这里最重要的不是 continuation 本身，而是它被纳入显式状态机：

- `maxOutputTokensRecoveryCount`
- `transition.reason = 'max_output_tokens_recovery'`

这避免了“无限续写”死循环。

## 12.3 streaming fallback

`claude.ts` 支持：

- streaming watchdog
- mid-stream failure -> non-streaming fallback
- 404 stream creation fallback

并且 fallback 后仍然产出统一 `AssistantMessage`。

这意味着对 query 层而言，streaming/non-streaming 是同构的，这是一种非常好的 abstraction barrier。

## 12.4 interrupt handling

用户中断时，query 层不会简单 return，而是：

- 消费 StreamingToolExecutor 剩余结果
- 必要时合成缺失 `tool_result`
- 追加中断消息

其核心目标是：

**即使被打断，消息链也保持协议合法。**

这是成熟 Agent Runtime 的典型特征。

---

## 13. 设计约束

对这套系统最重要的理解之一，是它有大量“非显式业务需求”的底层约束。

## 13.1 API 协议约束

- `tool_use` / `tool_result` 必须严格配对
- tool_result 中的非文本块与 `is_error` 有兼容限制
- 某些 beta header 缺失时，某些 block 类型必须剥离
- media item 数量存在硬上限
- `max_tokens > thinking.budget_tokens`

## 13.2 cache 稳定性约束

- mid-session 改 header 可能 bust cache
- defer_loading 工具不应影响 cache fingerprint
- cache edits 要插入到固定 user message 位置
- fire-and-forget fork 不能污染主链 cache key

## 13.3 运行模式约束

- main thread 与 subagent 不能共享某些全局状态语义
- 某些 compact/cleanup 逻辑必须限制在主线程
- `querySource` 决定了哪些恢复/compact 可以开启

## 13.4 可恢复性约束

- 错误消息不能比恢复逻辑更早暴露
- transcript 需要在 query 前就可 resume
- 非法消息链要修复后才能继续推 API

这些约束共同决定了为什么代码会看起来“过度工程化”。  
实际上这不是冗余，而是 Agent Runtime 在真实生产场景下的必需复杂度。

---

## 14. 关键工业级设计与创新点

## 14.1 会话编排与 turn 状态机的双层结构

很多系统把会话管理和 turn 执行写在一起，导致：

- 恢复逻辑混乱
- 测试困难
- 多入口复用困难

Claude-code-open 把二者拆开，是非常正确的 industrial decomposition。

## 14.2 消息域与协议域分离

`normalizeMessagesForAPI()`、`ensureToolResultPairing()`、payload post-processing 这一整套机制说明：

- UI message domain
- transcript domain
- API protocol domain

三者是不同域，需要显式映射。

这是成熟系统才会认真处理的问题。

## 14.3 流式工具执行

`StreamingToolExecutor` 是本架构的重要亮点。

它不是简单并发，而是：

- stream-aware
- order-preserving
- failure-propagating
- context-modifier aware

这是明显带有 runtime 优化思维的设计。

## 14.4 cache editing / persisted tool result / context governance 组合

这三者一起构成了非常实用的“长上下文工程体系”：

- tool result 太大 -> externalize
- 可编辑缓存前缀 -> cache edits
- 实在不行 -> compact / collapse / snip

这是一个分层式 context management stack，不是单一摘要方案。

## 14.5 强类型权限解释链

Permission 决策不是 UI 层临时逻辑，而是：

- runtime 一等结果
- 结构化解释对象
- 可被 telemetry / OTel / hook 消费

这让系统具备了更强的：

- 观测性
- 可审计性
- 行为诊断能力

---

## 15. 对 AI/Agent 系统设计的启示

从架构方法论看，这套系统有几条非常值得借鉴：

1. **把消息链当成主数据模型，而不是辅助日志。**
2. **把工具执行视为独立 runtime，不是模型输出后的附属回调。**
3. **把错误恢复设计成状态机分支，而不是异常处理。**
4. **把协议合法性校验放在真正的 API 边界，而不是 UI 边界。**
5. **把长上下文治理做成分层技术栈，而不是单一 summarize 按钮。**
6. **把权限系统建模成强类型决策图，而不是布尔开关。**

---

## 16. 结论

Claude-code-open 的对话执行架构，本质上是一个面向生产环境的 Agent Runtime：

- 上层是会话编排器 `QueryEngine`
- 中层是显式状态机 `query.ts`
- 下层是请求编译器 `services/api/claude.ts`
- 旁路是工具运行子系统 `services/tools/*`
- 底座是消息协议、权限决策、上下文治理、持久化与恢复体系

它最值得关注的，不是“功能很多”，而是这些功能被收敛进了一个**可恢复、可观察、协议安全、支持长上下文的统一执行框架**。

如果把它看成聊天 CLI，会低估其设计价值。  
如果把它看成一个本地优先的 Agent Runtime，这套架构是相当成熟的。
