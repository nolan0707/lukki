# Claude Code Query Harness 深度拆解

基于 `vendor/Claude-code-open/src/` 的源码，`./specs/read/query_engine.md` 中的若干表述需要校正。最重要的一点是：**Claude Code 的 Harness 不是单点地“塞在 `QueryEngine.ts` 里”，而是一个分层治理结构**。

## 0. 总览架构图

如果要把这套设计先讲清楚，最适合的 opening 不是从某个文件讲起，而是先给出整体分层：

```text
┌──────────────────────────────────────────────┐
│ SDK / Remote Client / UI                     │
│ 消费 SDKMessage 流，渲染结果与状态            │
└──────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────┐
│ QueryEngine.ts                               │
│ 会话级 Harness                               │
│ - submitMessage() 对外异步消息流接口         │
│ - system prompt 拼装                         │
│ - 输入预处理 / transcript 持久化             │
│ - SDK message 归一化 / result 汇总           │
└──────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────┐
│ query.ts                                     │
│ turn 级控制循环 / agent 状态机               │
│ - queryLoop()                                │
│ - tool_use -> tool_result 闭环               │
│ - compact / retry / recovery / stop hooks    │
│ - token budget / turn continuation           │
└──────────────────────────────────────────────┘
          │                     │
          │                     │
          ▼                     ▼
┌──────────────────────┐   ┌──────────────────────┐
│ services/api/        │   │ services/tools/      │
│ claude.ts            │   │ toolOrchestration.ts │
│ API / streaming      │   │ + toolExecution.ts   │
│ Harness              │   │ Tool Harness         │
│ - 请求构造           │   │ - 分批调度           │
│ - thinking/effort    │   │ - 权限校验           │
│ - 流式事件解析       │   │ - hook / tool call   │
│ - retry/fallback     │   │ - tool_result /      │
│                      │   │   contextModifier    │
└──────────────────────┘   └──────────────────────┘
          │                     │
          └──────────┬──────────┘
                     ▼
┌──────────────────────────────────────────────┐
│ compact/*                                    │
│ Context Management Harness                   │
│ - microcompact / autocompact                 │
│ - reactive compact / context collapse        │
│ - compact_boundary                           │
└──────────────────────────────────────────────┘
```

这张图对应的关键理解是：

- `QueryEngine.ts` 不直接承载全部复杂性，它负责最外层的会话封装
- `query.ts` 才是主控制循环，是 Claude Code agentic harness 的中枢
- `claude.ts` 负责和模型 API 打交道，把模型流转成内部消息流
- `toolOrchestration.ts` / `toolExecution.ts` 负责工具调度、安全、权限和副作用
- `compact/*` 负责长上下文治理与自愈

因此更适合分享时使用的一句话不是“Claude Code 的核心在 `QueryEngine.ts`”，而是：

**Claude Code 的核心是一个分层 Harness：外层管会话，中层管循环，底层分别治理模型流、工具副作用和上下文恢复。**

- `QueryEngine.ts` 负责会话级封装、系统提示拼装、输入预处理、SDK 输出归一化、转录持久化和结果汇总。见 `vendor/Claude-code-open/src/QueryEngine.ts:175-183`, `209-212`, `243-271`, `675-686`, `1058-1155`
- `query.ts` 才是主查询循环与状态机核心，负责“模型请求 -> 工具执行 -> 结果回注 -> 继续下一轮”的 agentic loop。见 `vendor/Claude-code-open/src/query.ts:219-230`, `241-280`, `307-365`, `1360-1408`
- `services/api/claude.ts` 负责 API 请求构造、thinking/effort 参数装配、流式事件解析、流式失败回退到 non-streaming，以及底层 API 错误映射。见 `vendor/Claude-code-open/src/services/api/claude.ts:1538-1728`, `1778-1858`, `1868-1930`, `1979-2304`, `2404-2570`
- `services/tools/toolOrchestration.ts` 与 `services/tools/toolExecution.ts` 负责工具编排、并发/串行分批、权限校验、hook、tool_result 生成与回注。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:19-82`, `91-116`; `vendor/Claude-code-open/src/services/tools/toolExecution.ts:337-489`, `599-680`, `916-1104`, `1207-1479`

---

## 1. 核心交互：异步生成器驱动，但分层明确

原概要说“`QueryEngine` 的底层是异步生成器驱动的流式循环”，这个方向是对的，但更准确的分层是：

- `QueryEngine.submitMessage()` 是 SDK-facing 的异步生成器。见 `vendor/Claude-code-open/src/QueryEngine.ts:209-212`
- 它内部 `for await` 消费 `query()` 产出的消息流，再转成 SDK message/result。见 `vendor/Claude-code-open/src/QueryEngine.ts:675-686`, `757-969`, `1082-1155`
- `query()` 本身也是异步生成器；真正的循环状态机在 `queryLoop()`。见 `vendor/Claude-code-open/src/query.ts:219-230`, `241-280`, `307-321`
- 更底层的 API streaming 由 `claude.ts` 解析 Anthropic streaming event，再逐块 yield assistant message / stream_event。见 `vendor/Claude-code-open/src/services/api/claude.ts:1778-1858`, `1979-2304`

所以，Claude Code 不是“一个异步生成器”，而是**三层异步生成器叠加**：

1. `claude.ts`：API stream parser  
2. `query.ts`：agentic query loop / state machine  
3. `QueryEngine.ts`：session harness / SDK adapter

这体现的是典型 Harness 分层：**最底层负责感知流，最中层负责决策循环，最外层负责会话治理与对外接口。**

### 1.1 什么叫“SDK-facing 的异步生成器”

这个说法可以拆成两层：

- “异步生成器”指 `async function*`。它不是一次性返回最终结果，而是可以在异步过程中持续 `yield` 多个中间结果，调用方通过 `for await (...)` 一边消费、一边推进流程
- “SDK-facing”指这不是只给内部函数使用的生成器，而是**对外暴露给 SDK 调用方消费的那一层接口**

在 Claude Code 里，对应的是 `QueryEngine.submitMessage()`：

- 定义：`vendor/Claude-code-open/src/QueryEngine.ts:209-212`
- 返回类型：`AsyncGenerator<SDKMessage, void, unknown>`

也就是说，SDK 调用方拿到的不是“等整轮结束后一次性返回的对象”，而是一条持续产出 `SDKMessage` 的事件流。典型消费方式就是：

```ts
for await (const msg of engine.submitMessage(prompt)) {
  // 上层 SDK / UI / remote client 边收到边处理
}
```

这些 `SDKMessage` 里可能包括：

- system init
- assistant message
- stream event
- compact boundary
- tool use summary
- 最终 `result`

所以“SDK-facing 的异步生成器”本质上就是：

**面向外部 SDK 暴露的一条异步消息流接口，调用方通过 `for await` 持续接收一次查询生命周期中的所有阶段性事件。**

### 1.2 三层异步生成器的调用关系

按源码看，它不是单层生成器，而是三层套接：

```text
SDK / 调用方
  |
  | for await (msg of QueryEngine.submitMessage(...))
  v
QueryEngine.submitMessage()
  |
  | for await (message of query(...))
  v
query()
  |
  | for await (message of claude.ts / callModel(...))
  v
claude.ts / queryModelWithStreaming()
  |
  | for await (part of Anthropic Stream)
  v
Anthropic API Stream
```

这三层的职责是：

- `claude.ts`：消费 Anthropic API 的原始流，组装 `text / tool_use / thinking / stream_event`。见 `vendor/Claude-code-open/src/services/api/claude.ts:1778-1858`, `1940-2304`
- `query.ts`：驱动 agent loop，把 assistant 的 `tool_use` 接成工具执行，再把 `tool_result` 回注上下文。见 `vendor/Claude-code-open/src/query.ts:307-365`, `1360-1408`
- `QueryEngine.ts`：把内部消息流整理成 SDK 可见的 `SDKMessage`，同时处理会话、持久化、结果汇总。见 `vendor/Claude-code-open/src/QueryEngine.ts:675-686`, `757-969`, `1135-1155`

因此可以用一句话概括：

**`claude.ts` 管流，`query.ts` 管循环，`QueryEngine.ts` 管对外接口。**

### 1.3 一次带工具调用的完整时序

```text
1. SDK 调用 submitMessage(...)
2. QueryEngine 处理输入、拼 system prompt，进入 query()
3. query() 调 claude.ts 发起模型请求
4. claude.ts 从 API 流中收到 assistant + tool_use
5. claude.ts yield assistant message 给 query()
6. query() 识别到 tool_use，进入工具执行
7. toolExecution.ts 生成 user(tool_result)
8. query() 把 tool_result 回注到上下文
9. query() 再次调用 claude.ts
10. claude.ts 返回新的 assistant 文本
11. query() 判断本轮结束
12. QueryEngine 把内部消息整理为 SDKMessage 持续对外 yield
13. SDK 侧边消费边渲染
14. 最后收到 result
```

这正是 Claude Code Harness 的关键设计：**模型流、Agent 循环、对外接口是三层解耦的，但通过异步生成器自然串接成一条连续事件流。**

---

## 2. 工具调用循环：真正的 Think-Act-Observe 在 `query.ts`

原概要把工具调用循环归到 `QueryEngine`，但源码显示核心在 `query.ts`。

### 2.1 模型输出 tool_use 后，循环进入 follow-up

- `query.ts` 在流式消费模型输出时收集 `tool_use` block，只要发现 tool_use 就把 `needsFollowUp = true`。见 `vendor/Claude-code-open/src/query.ts:826-845`
- 如果本轮没有 tool_use，循环尝试结束或进入 stop hooks / recovery。见 `vendor/Claude-code-open/src/query.ts:1062-1357`
- 如果有 tool_use，则进入工具执行阶段。见 `vendor/Claude-code-open/src/query.ts:1360-1408`

### 2.2 工具执行不是一把梭，而是“分批 + 并发安全”调度

- `runTools()` 会先按 `isConcurrencySafe` 把工具调用分成“可并发读操作批次”与“必须串行的批次”。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:19-82`, `91-116`
- 并发批次走 `runToolsConcurrently()`，串行批次走 `runToolsSerially()`。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:118-177`

这不是简单的 tool loop，而是**受上下文一致性约束的调度器**。

### 2.2.1 为什么说它是“受上下文一致性约束的调度器”

这里的“受上下文一致性约束”不是源码原词，而是对实现机制的抽象总结。含义是：

**Claude Code 不是把一批 `tool_use` 无脑并发跑掉，而是在“尽可能并发”和“不能破坏后续上下文语义”之间做平衡。**

原因在于，工具执行不只是 I/O，它还会影响后续推理上下文：

- 有些工具只是读取信息，彼此独立，天然适合并发
- 有些工具会修改文件、工作目录或其它状态，必须保持顺序
- 有些工具虽然看起来只是返回结果，但会通过 `contextModifier` 改写 `ToolUseContext`

因此系统必须保证：

- 能并发的才并发
- 不能并发的严格串行
- 上下文更新顺序稳定，不依赖“谁先跑完”

这就是“上下文一致性约束”的本质。

### 2.2.2 第一层约束：并发边界由工具语义决定

- `partitionToolCalls()` 会逐个检查工具的 `isConcurrencySafe(...)`，决定该工具是否可以进入并发批次。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:91-116`
- 连续的“可安全并发”工具会被合并到同一个 batch；不安全工具则单独成批。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:109-113`

所以这里的调度不是：

```text
拿到一批 tool_use -> 全部 Promise.all
```

而是：

```text
tool_use 序列
-> 按 concurrency-safe 规则切成多个 batch
-> 每个 batch 再决定并发或串行
```

这说明 Claude Code 的第一原则不是吞吐量，而是**工具语义正确性优先**。

### 2.2.3 第二层约束：并发执行不等于乱序提交上下文

真正体现设计水平的是并发 batch 的上下文提交策略。

在 `runTools()` 的并发分支中：

- 工具可以并发执行。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:35-42`, `152-177`
- 但如果某个工具产生了 `contextModifier`，系统不会按完成时间立刻改写上下文
- 它会先把这些 modifier 暂存在 `queuedContextModifiers`
- 等整个并发 batch 的消息都处理完后，再按原始 `toolUse` 顺序依次应用。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:31-63`

这一步的意义是：

- 执行顺序可以并发
- **上下文提交顺序必须稳定**

如果没有这个机制，就会出现：

```text
tool A 与 tool B 并发
B 先完成，于是先提交 context
A 后完成，又覆盖 context
```

这样最终上下文就会依赖运行时快慢，而不是模型原始生成的调用顺序。结果是：

- 同样输入，在不同机器/不同负载下可能产生不同后续上下文
- 下一轮模型看到的状态不稳定
- agent 行为不可复现

Claude Code 通过“并发执行，顺序提交”的方式，避免了这种非确定性。

### 2.2.4 串行路径保证严格因果顺序

对于不安全 batch，`runToolsSerially()` 的策略更直接：

- 一个工具执行完成
- 若返回 `contextModifier`，立即更新 `currentContext`
- 下一个工具在更新后的上下文上执行

见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:118-149`

因此串行路径保证的是严格的：

```text
tool1 -> context1
tool2 基于 context1
tool3 基于 context2
```

这保证了带副作用工具之间的因果关系不会被调度器打乱。

### 2.2.5 为什么它更像 scheduler，而不是 executor

真正执行单个工具的是 `toolExecution.ts`：

- `runToolUse(...)` 负责单个工具的完整执行流程。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:337-489`
- 权限判定、hook、输入校验都在这里发生。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:599-733`, `916-1104`
- 真正调用工具本体的是 `tool.call(...)`。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1207-1223`

而 `toolOrchestration.ts` 负责的是：

- 哪些工具能一起跑
- 哪些必须拆开
- 上下文何时提交
- `inProgressToolUseIDs` 如何维护

所以更准确地说，它是一个**调度器**，而不是简单执行器。

### 2.2.6 这为什么对 Query Loop 至关重要

在 `query.ts` 中，工具执行结果会被回注进 `toolResults`，并成为下一轮模型调用的上下文组成部分。见 `vendor/Claude-code-open/src/query.ts:1384-1400`

因此，如果调度器不能保证上下文一致性，问题就不只是“日志顺序不好看”，而是：

- `tool_result` 顺序可能漂移
- `contextModifier` 生效顺序可能漂移
- 下一轮模型看到的上下文可能漂移
- 整个 agent 的行为不再稳定、可复现

所以“受上下文一致性约束的调度器”可以浓缩成一句话：

**Claude Code 允许工具在执行层面尽量并发，但对“是否可并发”和“上下文如何提交”施加严格约束，确保下一轮模型看到的是稳定、有序、可推理的上下文。**

### 2.2.7 什么叫“通过 `contextModifier` 改写 `ToolUseContext`”

这里的关键点是：**工具执行后的影响分成两路**，一路给模型看，一路给 Harness 自己看。

- 给模型看的，是 `tool_result`
- 给 Harness 自己看的，是 `contextModifier`

后者的作用就是：**在工具执行完成后，更新后续工具运行所依赖的内部上下文对象 `ToolUseContext`。**

#### `ToolUseContext` 是什么

`ToolUseContext` 可以理解为“工具运行环境 + 会话态封装”。它不是消息历史本身，而是后续工具执行依赖的运行时上下文，里面会带着诸如：

- 当前可用工具集合
- `abortController`
- `messages`
- `getAppState` / `setAppState`
- `setInProgressToolUseIDs`
- query tracking
- 其它工具层运行期状态

因此后续工具不是在真空里执行的，而是在 `ToolUseContext` 中执行的。

#### `contextModifier` 是什么

某些工具在执行完成后，除了返回结果，还会返回一个上下文更新函数。这个函数的语义可以抽象成：

```ts
(context: ToolUseContext) => ToolUseContext
```

也就是说：

- 输入是旧的 `ToolUseContext`
- 输出是新的 `ToolUseContext`

在源码里，tool execution 会把它挂到更新对象上：

- `contextModifier` 被包装进 `MessageUpdateLazy.contextModifier`。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1467-1472`

它的作用不是修改对话消息，而是修改**后续工具执行环境**。

#### 为什么不把这类变化直接放进消息历史

因为有些变化不是“应该让模型读到的对话事实”，而是“系统内部需要延续的运行期状态”。例如：

- 调整后续工具可见的内部执行环境
- 在同一轮工具批次里把前一个工具的运行时影响传给后一个工具
- 维护一些不适合序列化进消息流、但需要跨工具保留的内部状态

因此可以这样区分：

- `tool_result`：更新模型上下文
- `contextModifier`：更新运行时上下文

一个工具执行结束时，可能同时产生这两类影响。

#### 串行路径里怎么应用

在串行执行中，如果某个工具返回了 `contextModifier`，调度器会立刻把它应用到当前上下文，再让下一个工具继续执行。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:130-145`

也就是：

```text
tool1 执行完成
-> 修改 ToolUseContext
-> tool2 在更新后的 context 上执行
```

这样保证了严格的因果链条。

#### 并发路径里为什么不能“谁先完成谁先改”

并发 batch 中，多个工具都可能返回 `contextModifier`。如果按完成时间提交，就会出现：

```text
tool A / tool B 并发
B 先完成，先改 context
A 后完成，再改 context
```

那么最终上下文就依赖运行时快慢，而不是模型原始的 `tool_use` 顺序。为避免这种非确定性，Claude Code 的做法是：

- 并发执行时，先缓存各工具的 `contextModifier`
- 等整批完成后
- 再按原始 `tool_use` 顺序依次提交

见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:31-63`

这就是为什么前面说它是“受上下文一致性约束的调度器”。

#### 与 `tool_result` 的职责分离

这两者非常容易混淆，必须在分享里明确区分：

- `tool_result` 会进入消息历史，下一轮模型能看到。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1456-1466`
- `contextModifier` 不进入消息历史，它只影响后续工具执行时使用的 `ToolUseContext`。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1467-1472`

所以一句话总结就是：

**“通过 `contextModifier` 改写 `ToolUseContext`”指的是：工具执行完后，用函数式方式更新后续工具运行的内部上下文，而不是只把结果文本回注到对话历史。**

### 2.3 tool_result 会被重新注入消息历史，形成状态闭环

- `toolExecution.ts` 在工具成功或失败后，都会构造 user message，其中核心内容块是 `tool_result`。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:396-408`, `475-488`, `1029-1071`, `1456-1473`
- `query.ts` 收到这些工具执行更新后，把它们转成 API 继续可见的 user messages，推入 `toolResults`。见 `vendor/Claude-code-open/src/query.ts:1384-1400`
- 随后状态机把 `messages = [...messagesForQuery, ...assistantMessages, ...toolResults, ...]` 带入下一轮，继续请求模型。可从多个 `state = { messages: ... }` 的 continue site 看出，例如 `vendor/Claude-code-open/src/query.ts:1231-1249`, `1283-1305`, `1321-1339`, `1520` 附近的递进逻辑

因此 Claude Code 的 agentic 闭环不是抽象概念，而是非常具体的：

`assistant(tool_use) -> tool executor -> user(tool_result) -> next query iteration`

### 2.4 四类关键状态变化总表

为了便于分享时统一讲清 Claude Code Harness 内部“状态是如何流动的”，可以把最关键的四类变化归纳成下表：

| 状态项 | 主要作用对象 | 是否进入消息历史 | 是否暴露给模型下一轮 | 主要生产位置 | 主要消费位置 |
| --- | --- | --- | --- | --- | --- |
| `tool_result` | 模型上下文 | 是 | 是 | `toolExecution.ts` 生成 user message。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1456-1466` | `query.ts` 收集进 `toolResults`，参与下一轮请求。见 `vendor/Claude-code-open/src/query.ts:1384-1400` |
| `contextModifier` | 运行时工具上下文 | 否 | 否 | `toolExecution.ts` 挂到 update 上。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1467-1472` | `toolOrchestration.ts` 串行立即应用，或并发批次结束后顺序提交。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:31-63`, `140-145` |
| `compact_boundary` | 活跃上下文边界 | 是 | 间接是 | compaction 流程生成 post-compact messages。见 `vendor/Claude-code-open/src/query.ts:528-535`, `1148-1151` | `QueryEngine.ts` 将其输出给 SDK，并截断 pre-compact 历史。见 `vendor/Claude-code-open/src/QueryEngine.ts:917-942` |
| `result` | SDK / 外部调用方 | 否（终态消息，不是对话历史的一部分） | 否 | `QueryEngine.ts` 在会话收束时生成。见 `vendor/Claude-code-open/src/QueryEngine.ts:1135-1155` | SDK 调用方消费，作为一次 submitMessage 的最终结果 |

也可以把它们的职责关系浓缩成四句话：

- `tool_result`：把工具执行结果回注给模型
- `contextModifier`：把工具执行影响回注给 Harness 自己
- `compact_boundary`：把上下文窗口切换显式化
- `result`：把整轮执行结果交付给 SDK / 调用方

如果从“谁能看到它”来区分，会更容易讲：

- 模型能看到：`tool_result`
- Harness 内部能看到：`contextModifier`
- SDK 和 Harness 都能看到：`compact_boundary`
- 只有外部调用方最终关心：`result`

这张表的价值在于，它把 Claude Code 的状态流拆成了四条不同通道：

1. 面向模型的对话通道
2. 面向工具层的运行时通道
3. 面向上下文治理的边界通道
4. 面向 SDK 的结果通道

这也再次说明 Claude Code 的 Harness 设计不是“把所有信息都塞进消息历史”，而是**按作用对象把状态分流到不同的载体中**。

---

## 3. Thinking Mode：不是解析 `<thinking>` 标签，而是 API block + 参数编排

原概要说“识别并处理 `<thinking>` 块”，这与源码不符。源码里不是 XML 式 `<thinking>` 标签，而是 **Anthropic API 的 structured content block**。

### 3.1 Thinking 配置入口

- `QueryEngine` 初始化 `thinkingConfig`：显式配置优先，否则默认 adaptive 或 disabled。见 `vendor/Claude-code-open/src/QueryEngine.ts:278-282`
- `ThinkingConfig` 类型定义为 `adaptive | enabled(budgetTokens) | disabled`。见 `vendor/Claude-code-open/src/utils/thinking.ts:10-13`

### 3.2 Thinking 是否可用，由模型能力决定

- 是否支持 thinking：`modelSupportsThinking()`。见 `vendor/Claude-code-open/src/utils/thinking.ts:88-110`
- 是否支持 adaptive thinking：`modelSupportsAdaptiveThinking()`。见 `vendor/Claude-code-open/src/utils/thinking.ts:112-144`

### 3.3 真正的 thinking 参数装配在 `claude.ts`

- `claude.ts` 在请求构造时，根据 `thinkingConfig`、模型能力和 `max_tokens` 生成 `thinking` 参数。见 `vendor/Claude-code-open/src/services/api/claude.ts:1596-1629`
- 支持 adaptive 的模型直接走 `{ type: 'adaptive' }`。见 `vendor/Claude-code-open/src/services/api/claude.ts:1604-1613`
- 不支持 adaptive 的模型走 budget 模式，并确保 `budget_tokens < max_tokens`。见 `vendor/Claude-code-open/src/services/api/claude.ts:1615-1628`

### 3.4 流式 thinking block 是 API content block，不是文本标签

- 流式解析时，`content_block_start` 会初始化 `thinking` block。见 `vendor/Claude-code-open/src/services/api/claude.ts:2030-2037`
- `thinking_delta` 持续追加思维内容，`signature_delta` 追加签名。见 `vendor/Claude-code-open/src/services/api/claude.ts:2127-2160`

所以准确说法应是：**Claude Code 管理的是 API 级 thinking block 与 thinking policy，不是自行解析 `<thinking>` 文本标签。**

### 3.5 effort 与 thinking 是协同关系，但不是一回事

- `effort` 通过 `resolveAppliedEffort()` 解析，并写入 `output_config.effort`。见 `vendor/Claude-code-open/src/services/api/claude.ts:1458`, `1563-1569`
- 日志与 prompt cache break detection 也都把 effort 作为一级参数跟踪。见 `vendor/Claude-code-open/src/services/api/claude.ts:1471-1485`, `1736-1755`

因此“自适应思考 + effort 平衡速度/深度”这个方向是对的，但实现上是：

- `thinking`：控制思维块与思维预算
- `effort`：控制输出配置中的推理强度

两者并列协同，不应混为一个机制。

---

## 4. 上下文治理：不是单纯 token 计数，而是多级 context management

这是原概要最需要校正的部分。

### 4.1 Query loop 每轮都会做上下文投影和压缩治理

`query.ts` 在每次真正发请求前，会顺序执行：

1. `applyToolResultBudget()`：限制 tool_result 聚合尺寸。见 `vendor/Claude-code-open/src/query.ts:369-394`
2. `snipCompactIfNeeded()`：历史裁剪。见 `vendor/Claude-code-open/src/query.ts:396-410`
3. `microcompact()`：优先清理工具结果等低价值上下文。见 `vendor/Claude-code-open/src/query.ts:412-426`
4. `contextCollapse.applyCollapsesIfNeeded()`：上下文折叠投影。见 `vendor/Claude-code-open/src/query.ts:428-447`
5. `autocompact()`：必要时生成摘要并建立 compact boundary。见 `vendor/Claude-code-open/src/query.ts:453-543`

这说明 Claude Code 的策略不是“快超限了就压缩”，而是**逐层降损的 context management pipeline**。

### 4.2 “13,000 token 安全缓冲”说法不完整

源码实际有两层预算：

- `autoCompact.ts` 先从上下文窗口里**预留最多 20,000 tokens 给 compaction summary 输出**。见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:28-49`
- 然后 autocompact 触发阈值再减去 `AUTOCOMPACT_BUFFER_TOKENS = 13_000`。见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:62-76`

因此更准确的表述应是：

- **有效上下文窗口**先扣掉 compaction 摘要所需输出预算（最多 20k）
- **autocompact 触发阈值**再提前 13k 触发

不是“单纯预留 13k 给摘要”。

### 4.3 自愈循环存在，但不止一种路径

源码里至少有四条恢复路径：

- proactive autocompact：请求前主动压缩。见 `vendor/Claude-code-open/src/query.ts:453-543`
- reactive compact：收到 prompt-too-long 或 media-size 错误后回退压缩。见 `vendor/Claude-code-open/src/query.ts:1065-1183`
- context collapse overflow recovery：先尝试 drain staged collapse。见 `vendor/Claude-code-open/src/query.ts:1085-1116`
- max_output_tokens recovery：提额重试，或插入 meta message 续写。见 `vendor/Claude-code-open/src/query.ts:1185-1256`

“self-healing loop” 这个判断成立，但准确说应是：**Claude Code 有一整套多阶段恢复状态机，而不是单一 compact 机制。**

### 4.4 compact boundary 是显式一等消息

- autocompact 成功后，`buildPostCompactMessages()` 生成 post-compact message 序列并 yield。见 `vendor/Claude-code-open/src/query.ts:528-535`
- `QueryEngine` 会把 `compact_boundary` 当作正式系统消息输出给 SDK，并在内部截断 pre-compact message，释放内存。见 `vendor/Claude-code-open/src/QueryEngine.ts:897-942`

这意味着压缩不是“偷偷改数组”，而是**通过边界消息重写活跃上下文视图**。

### 4.5 `compact_boundary` 到底是什么

`compact_boundary`` 可以理解成：**Claude Code 在上下文压缩生效时插入的一条显式系统边界消息。**

它不是普通聊天内容，也不是给模型看的业务文本，而是 Harness 用来声明：

**从这条边界往前，旧上下文已经不再按原样参与后续主循环；从这条边界往后，新的活跃上下文开始生效。**

#### 为什么需要它

当上下文过长时，Claude Code 会做 compaction。但 compaction 不是简单地“删除前面消息”，因为系统还需要同时满足：

- 后续模型请求不能再携带完整旧历史
- SDK / 客户端需要知道压缩刚刚发生
- transcript / resume / replay 需要知道上下文从哪里切换
- 内存里的 pre-compact 历史需要被安全释放

因此系统必须有一个明确的、可编程识别的边界标记。这个标记就是 `compact_boundary`。

#### 它的本质

它本质上是一条 `system message`，其 subtype 是 `compact_boundary`。`QueryEngine.ts` 会专门识别它并转成 SDK 侧系统事件。见 `vendor/Claude-code-open/src/QueryEngine.ts:917-942`

所以它是：

- 系统级控制消息
- 一等事件
- 不是普通 assistant/user 内容
- 也不是 tool_result

更准确地说，它像是“上下文版本切换点”。

#### 它如何产生

在 `query.ts` 中，无论是 proactive autocompact 还是 reactive compact，一旦压缩成功，都会构造 `postCompactMessages` 并逐条 yield：

- proactive path：`vendor/Claude-code-open/src/query.ts:528-535`
- reactive path：`vendor/Claude-code-open/src/query.ts:1148-1151`

这组 post-compact messages 中就包含 `compact_boundary`。因此 Claude Code 的做法不是“静默替换数组”，而是显式发出一个边界消息，让下游知道：

- 压缩已经发生
- 新的活跃上下文已经建立

#### 它解决的根本问题

它真正解决的是“**活跃上下文视图切换**”。

在 `query.ts` 每轮开始时，系统会先做：

- `let messagesForQuery = [...getMessagesAfterCompactBoundary(messages)]`
- 见 `vendor/Claude-code-open/src/query.ts:365`

这说明 `compact_boundary` 的语义并不是“提示一下用户发生过压缩”，而是：

**它定义了当前 active context 的起点。**

换言之：

- summary 负责回答“旧上下文被压缩成了什么”
- `compact_boundary` 负责回答“从哪里开始，系统切换到压缩后的上下文”

#### 为什么说它不是提示，而是状态锚点

在 `QueryEngine.ts` 中，一旦收到 `compact_boundary`，系统会做两件关键事：

- 把这条边界消息作为正式 SDK 系统消息向外输出。见 `vendor/Claude-code-open/src/QueryEngine.ts:935-941`
- 直接裁掉内存里边界之前的消息。见 `vendor/Claude-code-open/src/QueryEngine.ts:922-933`

这意味着它不是日志提示，而是一个真正的状态切换锚点：

- 之前的历史不再是活跃上下文
- 之后的历史才是新的工作集

#### 它与持久化 / 恢复的关系

`QueryEngine.ts` 在写 transcript 时，对 `compact_boundary` 有专门处理：在写入边界前，会先 flush 到 preserved segment 的尾部，避免 resume 时关联链断裂。见 `vendor/Claude-code-open/src/QueryEngine.ts:693-714`

所以 `compact_boundary` 不只是影响当前查询轮次，它还是：

- session transcript 语义的一部分
- resume / replay 正确性的锚点
- 长会话内存与恢复机制的连接点

一句话总结就是：

**`compact_boundary` 是 Claude Code 在上下文压缩发生后插入的一条系统边界消息，用来显式声明“新的活跃上下文从这里开始”，并驱动后续的上下文截断、内存回收、SDK 通知和会话恢复。**

### 4.6 `mutableMessages`、本地 `messages` 与 `compact_boundary` 的关系

这里最容易混淆的是：`mutableMessages` 和 `submitMessage()` 里的本地 `messages` 名字很像，但它们不在同一层。

先给结论：

- `mutableMessages` 是 `QueryEngine` 实例持有的**会话级主状态**
- 本地 `messages` 是 `submitMessage()` 这一轮调用里的**当前工作集快照**
- `compact_boundary` 是两者共同遵循的**上下文切换锚点**
- 一旦收到 `compact_boundary`，两者都会把边界之前的内容裁掉，但服务的层次不同

#### `mutableMessages` 是什么

定义位置：

- `private mutableMessages: Message[]`
- 见 `vendor/Claude-code-open/src/QueryEngine.ts:186`

初始化位置：

- `this.mutableMessages = config.initialMessages ?? []`
- 见 `vendor/Claude-code-open/src/QueryEngine.ts:200-206`

它的语义是：

**这个 `QueryEngine` 对应的一整个 conversation，到当前为止累计下来的内部消息主状态。**

它跨 `submitMessage()` 调用存在，不是一次性局部变量。源码注释也明确说明：

- 一个 `QueryEngine` 对应一个 conversation
- 每次 `submitMessage()` 只是这个 conversation 的一个新 turn

见 `vendor/Claude-code-open/src/QueryEngine.ts:175-183`

所以 `mutableMessages` 更像：

- engine 持有的 conversation 内存态
- 会话级真相源
- 下一次 `submitMessage()` 的起点

#### 本地 `messages` 是什么

在 `submitMessage()` 中，会从 `mutableMessages` 派生一个局部数组：

- `const messages = [...this.mutableMessages]`
- 见 `vendor/Claude-code-open/src/QueryEngine.ts:433-435`

它不是 engine 字段，而是当前这次 `submitMessage()` 内部的工作副本。它的用途包括：

- 作为本轮传给 `query()` 的基础消息集
- 作为 transcript / result 汇总时的本地工作数组
- 随着 query 流式返回消息持续追加

例如后续收到 assistant/user/system message 时，会不断往局部 `messages` 中 push：

- 见 `vendor/Claude-code-open/src/QueryEngine.ts:716`

所以本地 `messages` 更像：

**当前 `submitMessage()` 这一次调用正在处理的工作视图。**

#### 两者的关系

关系可以简化成：

```text
mutableMessages  --拷贝-->  本地 messages
```

然后在本轮执行过程中：

- 新消息通常会同时进入两边
- 但维护目的不同，因此代码分别更新它们

可以理解成：

- `mutableMessages`：长期会话状态
- 本地 `messages`：当前 turn 工作状态

之所以不只保留一个数组，是因为系统需要同时处理：

- conversation 级别长期状态
- 当前 query 的中间态
- transcript 写入与 result 汇总控制

单数组会让这些职责过度耦合。

#### `compact_boundary` 如何作用于两者

在 `submitMessage()` 处理 `system` 消息时，有 `compact_boundary` 专门分支：

- 见 `vendor/Claude-code-open/src/QueryEngine.ts:897-942`

处理逻辑的关键顺序是：

1. 先把 `compact_boundary` push 进 `this.mutableMessages`
2. 如果它真的是 compact boundary，则裁掉边界前的消息
3. 同样把局部 `messages` 也裁掉边界前的部分

对应关键代码：

- `this.mutableMessages.push(message)`：`vendor/Claude-code-open/src/QueryEngine.ts:916`
- 裁 `mutableMessages`：`vendor/Claude-code-open/src/QueryEngine.ts:926-929`
- 裁本地 `messages`：`vendor/Claude-code-open/src/QueryEngine.ts:930-933`

也就是说，一旦边界消息到来：

- `mutableMessages` 切换到 post-compact 会话视图
- 本地 `messages` 切换到 post-compact 当前轮工作视图

#### 为什么两边都要裁

裁 `mutableMessages` 的目的是：

- 让 engine 的长期会话状态也进入 post-compact 视图
- 下一轮不会重新带回 pre-compact 全量历史
- 控制长会话内存占用

源码注释直接指出，这是为了释放 pre-compaction messages 以便 GC。见 `vendor/Claude-code-open/src/QueryEngine.ts:922-925`

裁本地 `messages` 的目的是：

- 让当前这次 `submitMessage()` 后续逻辑立刻使用新的工作集
- transcript / result / SDK 输出基于新的边界后视图继续推进
- 避免当前 turn 内还错误保留 pre-compact 工作集

所以两边都裁，但层级不同：

- `mutableMessages`：切换 engine 长期状态
- 本地 `messages`：切换当前调用工作状态

#### 为什么要先 push boundary，再 splice

顺序上不是先删旧消息再插边界，而是：

1. 先把 `compact_boundary` push 进去
2. 再把它前面的部分删除

这样裁完后，数组最前面就是边界本身，整体结构更像：

```text
compact_boundary
post-compact summary / attachments / preserved messages
recent assistant/user messages
...
```

这很重要，因为后续很多逻辑不是假设“旧消息消失了”，而是明确以这条边界来定义新的 active context 起点。

#### `query.ts` 如何进一步利用这条边界

在更底层的 `query.ts` 中，每轮请求开始前都会先投影：

- `getMessagesAfterCompactBoundary(messages)`
- 见 `vendor/Claude-code-open/src/query.ts:365`

这说明系统在逻辑层面和内存层面都有两道保障：

- 逻辑层：每轮 query 只取最近一次边界之后的消息
- 内存层：`QueryEngine` 直接把边界之前的消息裁掉

前者保证 query 语义正确，后者保证 engine 状态和内存也一起收敛。

#### 一个直观例子

假设压缩前：

```text
mutableMessages = [A, B, C, D, E]
本地 messages = [A, B, C, D, E]
```

压缩后生成：

```text
[compact_boundary, summary, recent1, recent2]
```

处理完边界后，两边都会大致变成：

```text
mutableMessages = [compact_boundary, summary, recent1, recent2, ...]
本地 messages = [compact_boundary, summary, recent1, recent2, ...]
```

之后：

- engine 长期状态从这里继续
- 当前 turn 后续逻辑从这里继续
- 下一轮 query 也只会从这里往后取

一句话总结就是：

**`mutableMessages` 是 `QueryEngine` 持有的会话级主状态，本地 `messages` 是当前 `submitMessage()` 的工作副本；`compact_boundary` 是两者共同遵循的上下文切换锚点，一旦出现，二者都会把边界之前的历史裁掉，从而让系统整体切换到压缩后的活跃上下文视图。**

---

## 5. 可靠性与安全性：权限、重试、fallback 都是 Harness 的组成部分

### 5.1 权限不是外围逻辑，而是工具执行入口的硬门禁

- `QueryEngine` 会先包装 `canUseTool()`，把拒绝记录到 `permissionDenials`，供 SDK 最终结果上报。见 `vendor/Claude-code-open/src/QueryEngine.ts:243-271`
- 真正执行工具前，`toolExecution.ts` 会调用 `resolveHookPermissionDecision(...)`，把 hook / classifier / permission mode / user decision 统一收口成 `permissionDecision`。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:916-932`
- 如果不允许，直接构造 `tool_result(is_error=true)` 回注，而不是执行工具。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:995-1104`
- 如果允许，才进入 `tool.call(...)`。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1206-1223`

这说明 Claude Code 的安全哲学是：**模型永远不能直接触达工具，必须先穿过 permission harness。**

### 5.2 API 重试逻辑是工业级状态机，不只是“失败重试”

- `withRetry()` 是统一重试循环，管理 `maxRetries`、429/529、fast mode cooldown、401 token refresh、stale keep-alive 连接重建等。见 `vendor/Claude-code-open/src/services/api/withRetry.ts:170-188`, `212-253`, `261-323`, `430-516`, `697-767`
- 对 529 是否重试，取决于 `QuerySource` 是否属于前台阻塞场景。见 `vendor/Claude-code-open/src/services/api/withRetry.ts:57-89`
- persistent unattended retry 也是单独支持的。见 `vendor/Claude-code-open/src/services/api/withRetry.ts:91-104`

因此源码体现的不是“简单 retry”，而是**面向交互语义的差异化 retry policy**。

### 5.3 流式失败会自动降级到 non-streaming

- `claude.ts` 在 streaming 失败时，会根据配置决定是否 fallback 到 non-streaming。见 `vendor/Claude-code-open/src/services/api/claude.ts:2464-2570`
- 连 streaming endpoint 404 这种创建阶段失败，也专门兜底到 non-streaming。见 `vendor/Claude-code-open/src/services/api/claude.ts:2607-2689`

这说明 Claude Code 把“用户要拿到结果”看得比“始终坚持 streaming”更重要。

### 5.4 流式本身也有 watchdog 和 stall 监控

- 空闲 watchdog：超过阈值无 chunk 到达则主动 abort stream。见 `vendor/Claude-code-open/src/services/api/claude.ts:1868-1929`
- stall logging：chunk 间隔过长会记录 stall telemetry。见 `vendor/Claude-code-open/src/services/api/claude.ts:1944-1965`, `2366-2380`

这进一步说明 Harness 不是只治理“模型输出内容”，也治理**模型流的活性与网络健康度**。

---

## 6. 真正的设计哲学：集中治理，但不是单文件集中

原概要最后一条说“将如此庞大的逻辑集中在 `QueryEngine.ts` 中”，这与源码有明显出入。

更符合源码的结论是：

- `QueryEngine.ts` 负责 **conversation/session harness**
- `query.ts` 负责 **turn-level control loop**
- `claude.ts` 负责 **provider/API streaming harness**
- `toolExecution.ts` 负责 **tool safety harness**
- `autoCompact.ts` / `compact.ts` 负责 **context recovery harness**

它们共同构成 Claude Code 的核心哲学：

1. **LLM 不是控制器，Harness 才是控制器**  
   模型只负责产生内容块；是否继续、是否执行工具、是否压缩、是否重试，都由外层状态机决定。见 `vendor/Claude-code-open/src/query.ts:307-321`, `1062-1357`, `1360-1408`

2. **一切副作用都要可回注、可恢复、可追踪**  
   tool call 变成 `tool_result`，compact 变成 `compact_boundary`，API retry 变成系统事件，结果最终都进入统一消息流。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1456-1473`; `vendor/Claude-code-open/src/query.ts:528-535`; `vendor/Claude-code-open/src/QueryEngine.ts:917-954`

3. **先保障系统可继续运行，再追求最优上下文保真**  
   microcompact、collapse、autocompact、reactive compact、non-streaming fallback、max-output recovery，都是“先活下来”的工程哲学。见 `vendor/Claude-code-open/src/query.ts:396-543`, `1065-1256`; `vendor/Claude-code-open/src/services/api/claude.ts:2464-2689`

4. **安全和可靠性不依赖模型自觉，而依赖外部硬约束**  
   permission gate、input schema 校验、tool validation、retry policy、watchdog 都在模型外部。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-733`, `916-1104`; `vendor/Claude-code-open/src/services/api/withRetry.ts:170-516`

---

## 7. 对原概要的逐条校正

### 7.1 “QueryEngine 的底层是异步生成器驱动的流式循环”

- 基本正确
- 但应改为：`QueryEngine.submitMessage()`、`query()`、`claude.ts` 三层 async generator 共同完成流式交互

### 7.2 “工具调用循环由 QueryEngine 实现”

- 不准确
- 应改为：`QueryEngine` 负责封装和归一化；真正的 tool-call loop 在 `query.ts`，工具调度在 `services/tools/toolOrchestration.ts`，工具执行和权限校验在 `services/tools/toolExecution.ts`

### 7.3 “识别并处理 `<thinking>` 块”

- 不准确
- 应改为：系统处理的是 Anthropic API 的 `thinking` / `thinking_delta` / `signature_delta` content block

### 7.4 “预留约 13,000 token 用于生成结构化摘要”

- 不准确
- 应改为：有效窗口先扣除最多 20,000 tokens 的 compaction summary 输出预算，再以 13,000 tokens 作为 autocompact 触发缓冲

### 7.5 “将庞大逻辑集中在 QueryEngine.ts 中”

- 不准确
- 应改为：Claude Code 采用的是**分层集中治理**，不是**单文件集中治理**

---

## 8. 用于分享的最终一句话总结

Claude Code 的核心不是“让模型自己工作”，而是用一个分层 Harness 把模型包进严格的控制回路里：`QueryEngine` 管会话，`query.ts` 管状态机，`claude.ts` 管流式 API，`toolExecution.ts` 管副作用与权限，`compact/*` 管上下文自愈。**模型负责生成候选动作，Harness 负责把这些动作变成一个可持续、可恢复、可审计的系统。**
