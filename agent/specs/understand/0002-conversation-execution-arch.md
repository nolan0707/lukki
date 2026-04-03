# Claude-code-open 对话执行架构深度解读

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 的“对话执行架构”。

目标不是泛泛介绍“它会调用模型”，而是回答下面几个初学者最容易困惑的问题：

1. 用户输入后，代码到底从哪里开始处理？
2. 什么情况下会直接执行本地命令，什么情况下会进入模型查询？
3. 一次完整的 turn 是如何在“模型输出 -> 工具执行 -> 工具结果回填 -> 再次调用模型”之间循环的？
4. 为什么这里会有 `QueryEngine`、`query.ts`、`services/api/claude.ts` 三层？
5. 消息、工具上下文、权限、压缩、恢复这些能力分别在哪一层生效？

本文基于以下主干代码阅读整理：

- `src/QueryEngine.ts`
- `src/query.ts`
- `src/services/api/claude.ts`
- `src/services/tools/*`
- `src/utils/processUserInput/*`
- `src/utils/messages.ts`
- `src/Tool.ts`
- `src/query/*`

---

## 2. 先给结论

这套对话执行架构可以理解成一个 4 层流水线：

```text
输入接入层
  processUserInput / processTextPrompt

会话编排层
  QueryEngine

单次 turn 执行层
  query.ts

模型与工具运行层
  services/api/claude.ts
  services/tools/*
```

每层职责不同：

- `processUserInput`：把“用户输入”翻译成系统可理解的消息和动作。
- `QueryEngine`：站在会话级别，组织系统提示、上下文、工具、持久化、结果输出。
- `query.ts`：站在单次 turn 级别，负责循环执行“模型 -> 工具 -> 再模型”。
- `services/api/claude.ts` 和 `services/tools/*`：真正和模型 API、工具调用打交道。

如果只记一句话：

`QueryEngine` 管“这次会话怎么问”，`query.ts` 管“这一轮怎么跑”，`claude.ts` 管“怎么请求模型”，`toolExecution.ts` 管“怎么执行工具”。

---

## 3. 为什么要分成这么多层

很多初学者第一眼会觉得：

- 为什么不直接在 `REPL.tsx` 里调用模型？
- 为什么不把工具执行写在 `claude.ts` 里？

原因是这个项目不是“问一次答一次”的简单聊天程序，而是 Agent 型系统。它有这些复杂特征：

- 一次用户提问可能触发多轮模型调用。
- 模型输出里可能包含多个 `tool_use`。
- 工具执行可能并发，也可能串行。
- 工具执行前后还要跑 hooks、权限审批、日志、观测。
- 中途可能发生 autocompact、reactive compact、fallback、resume。
- 同一套执行链要支持 REPL、SDK、remote、subagent。

如果都写在一个函数里，复杂度会失控。所以代码把复杂度按层切开：

- 会话级复杂度交给 `QueryEngine`
- 单次查询循环交给 `query.ts`
- 模型接入复杂度交给 `claude.ts`
- 工具执行复杂度交给 `toolExecution.ts` / `toolOrchestration.ts`

这是一个很典型的“编排层 + 执行层”分离设计。

---

## 4. 整体执行流程

先看从用户输入到 turn 结束的全景图：

```text
用户输入
  -> processUserInput
  -> QueryEngine.submitMessage()
  -> 构建 systemPrompt / userContext / systemContext / ToolUseContext
  -> query()
     -> 预处理消息
     -> token/compact/context-collapse 处理
     -> 调 services/api/claude.ts 流式请求模型
     -> 收到 assistant 文本 / tool_use
     -> 调 services/tools/* 执行工具
     -> 生成 tool_result user message
     -> 如果还有 tool_use，则继续下一轮
     -> 如果没有 tool_use，则进入 stop hooks / budget / 结束
  -> QueryEngine 持久化 transcript / 产出 SDK result
```

这里最关键的是：**一次用户提交，不一定只发起一次模型请求**。

只要模型产生了 `tool_use`，`query.ts` 就会继续下一轮，把工具结果喂回模型，直到模型不再请求工具。

所以整个系统本质上是一个“循环式 Agent 执行器”。

---

## 5. 输入接入层：用户输入是怎么被变成消息的

## 5.1 入口：`processUserInput`

文件：`src/utils/processUserInput/processUserInput.ts`

它负责做三件事：

1. 识别输入类型。
2. 决定这次输入是“本地处理”还是“进入模型查询”。
3. 生成标准化消息数组。

输出结构 `ProcessUserInputBaseResult` 很关键：

```ts
{
  messages: Message[]
  shouldQuery: boolean
  allowedTools?: string[]
  model?: string
  effort?: EffortValue
  resultText?: string
}
```

这里最重要的字段是：

- `messages`：本次输入生成的消息
- `shouldQuery`：是否进入模型查询

也就是说，输入处理层本身就可以“截流”。

## 5.2 两类输入结果

### 情况 A：本地处理，不调用模型

例如：

- slash command 本地执行
- 某些命令直接修改配置
- 某些命令只产生本地输出

此时：

- `shouldQuery = false`
- `QueryEngine` 不会进入 `query()`
- 直接把本地结果作为输出返回

### 情况 B：进入模型查询

此时：

- 会生成一个 `UserMessage`
- 可能附带 `AttachmentMessage`
- 然后交给 `QueryEngine` 进入后续对话执行

## 5.3 文本输入如何转成 `UserMessage`

文件：`src/utils/processUserInput/processTextPrompt.ts`

核心动作：

- 生成 `promptId`
- 记录 telemetry
- 处理图片/附件
- 调 `createUserMessage()`

如果包含图片，最终消息内容可能是：

```text
text block + image blocks + attachment messages
```

如果是纯文本，就是一个普通 `UserMessage`。

这一步本质上是在做：

**把终端输入翻译成模型能理解的内容块数组。**

---

## 6. 会话编排层：`QueryEngine`

## 6.1 `QueryEngine` 的定位

文件：`src/QueryEngine.ts`

`QueryEngine` 是“会话级编排器”，不是底层执行器。

它负责：

- 保存本会话的 `mutableMessages`
- 保存累计 usage、permission denials、文件读取状态
- 构造系统提示和上下文
- 调用 `query()`
- 处理 transcript 持久化
- 把内部消息翻译成 SDK 可消费输出

可以把它理解成：

```text
一个会话 = 一个 QueryEngine 实例
一次 submitMessage = 这个会话中的一轮用户输入
```

## 6.2 `QueryEngineConfig`

`QueryEngineConfig` 是理解架构的第一关键结构。

它告诉你一次对话执行真正依赖什么：

- `cwd`
- `tools`
- `commands`
- `mcpClients`
- `agents`
- `canUseTool`
- `getAppState` / `setAppState`
- `initialMessages`
- `readFileCache`
- `customSystemPrompt`
- `thinkingConfig`
- `maxTurns`
- `maxBudgetUsd`
- `jsonSchema`

这说明 `QueryEngine` 并不自己“拥有所有资源”，它依赖外部注入。

这有两个好处：

1. 更容易复用到 REPL / SDK / Remote 等不同入口。
2. 更容易测试，因为依赖可以替换。

## 6.3 `submitMessage()` 干了什么

`submitMessage()` 是对话执行的主入口。

它的大致步骤如下：

### 第一步：处理用户输入

调用 `processUserInput()`：

- 解析 slash command
- 生成 `UserMessage` / `AttachmentMessage`
- 决定 `shouldQuery`

### 第二步：立即持久化用户消息

如果要查询模型，会先把用户消息写入 transcript。

这样即使 API 调用没回来，会话也仍然可 resume。

这是一个非常实用的设计点：

- 不是“响应完成后再记录”
- 而是“用户提交成功就先记录”

避免中途崩溃后会话丢失。

### 第三步：装配系统提示和上下文

通过 `fetchSystemPromptParts()` 获取：

- `defaultSystemPrompt`
- `userContext`
- `systemContext`

然后再组合：

- `customSystemPrompt`
- `appendSystemPrompt`
- memory mechanics prompt

形成最终 `systemPrompt`。

### 第四步：构造 `ToolUseContext`

`ToolUseContext` 是整个执行链最重要的数据结构之一，后面会单独解释。

### 第五步：加载技能与插件缓存

在真正查询之前，`QueryEngine` 还会加载：

- slash command tool skills
- 插件缓存

并产出 `system init message`。

### 第六步：调用 `query()`

把当前消息、system prompt、contexts、tools、权限函数等交给 `query()`。

### 第七步：消费 `query()` 产出的流式消息

`query()` 会 yield：

- assistant message
- user tool_result message
- compact boundary
- progress
- tool summary
- tombstone

`QueryEngine` 一边消费，一边：

- 写 transcript
- 累计 usage
- 输出 SDK 结果

## 6.4 `QueryEngine` 的设计价值

它解决的是“会话级别的稳定性”问题：

- 保证输入先落盘
- 保证初始消息回放
- 保证 usage 累计
- 保证结构化输出和权限拒绝可追踪
- 保证本地命令和模型查询共存

如果没有 `QueryEngine`，这些逻辑就会被散落在 REPL 和 query loop 里。

---

## 7. 单次 turn 的核心：`query.ts`

## 7.1 `query.ts` 的定位

文件：`src/query.ts`

`query.ts` 是“单个 turn 的执行循环”。

它不关心：

- UI 怎么显示
- transcript 怎么组织成完整会话
- 本次提交来自 REPL 还是 SDK

它只关心：

- 当前消息如何送给模型
- 返回后是否要执行工具
- 工具执行后是否要继续追问模型
- 中途如何压缩、恢复、fallback、停止

可以把它理解成：

```text
一个 turn 内部的小状态机
```

## 7.2 `QueryParams`

`QueryParams` 表示执行一轮 query 所需要的最小输入：

- `messages`
- `systemPrompt`
- `userContext`
- `systemContext`
- `canUseTool`
- `toolUseContext`
- `fallbackModel`
- `querySource`
- `maxTurns`
- `taskBudget`
- `deps`

其中 `deps` 很关键。

## 7.3 为什么有 `deps`

文件：`src/query/deps.ts`

`query.ts` 没有把底层能力写死，而是通过 `QueryDeps` 注入：

- `callModel`
- `microcompact`
- `autocompact`
- `uuid`

这是一种轻量依赖反转设计。

好处是：

- 测试时可以替换底层依赖
- `query.ts` 更像“纯编排器”
- 降低和外部 I/O 的耦合

## 7.4 `State`：query loop 的状态机

`query.ts` 内部有一个很重要的 `State` 结构：

```ts
type State = {
  messages
  toolUseContext
  autoCompactTracking
  maxOutputTokensRecoveryCount
  hasAttemptedReactiveCompact
  maxOutputTokensOverride
  pendingToolUseSummary
  stopHookActive
  turnCount
  transition
}
```

这是 query loop 的核心状态。

它表示：

- 当前传给模型的消息是什么
- 当前工具上下文是什么
- 是否做过 compact
- 是否做过 reactive compact
- 是否处于 stop hook 之后的重试
- 是否处于 max output token 恢复阶段

可以理解为：

```text
State = 当前这轮 query 的执行现场
```

每次 `continue` 重试，都不是重新从零开始，而是基于新的 `State` 继续。

---

## 8. query loop 的关键业务流程

下面按执行顺序拆开。

## 8.1 第 0 步：预处理上下文

在真正调模型之前，`query.ts` 会对消息做一系列处理：

1. `getMessagesAfterCompactBoundary(messages)`
2. `applyToolResultBudget()`
3. `snipCompactIfNeeded()`
4. `microcompact()`
5. `contextCollapse.applyCollapsesIfNeeded()`
6. `autocompact()`

### 这一步在解决什么问题？

LLM 最大问题不是“不会执行”，而是“上下文会爆”。

所以在 query loop 开头，系统会先做上下文治理：

- 太大的 tool result 要替换成瘦身版本
- 太长历史要 snip
- 可缓存部分要 microcompact
- 可折叠上下文要 collapse
- 再不行就 autocompact 成摘要

也就是说：

**模型调用之前，系统会先尝试把上下文打磨成“可发送状态”。**

## 8.2 第 1 步：准备本轮调用状态

接着会初始化：

- `assistantMessages = []`
- `toolResults = []`
- `toolUseBlocks = []`
- `needsFollowUp = false`

含义很直观：

- 本轮模型输出的 assistant 消息有哪些
- 本轮工具结果有哪些
- 本轮要求执行的工具块有哪些
- 本轮结束后是否还需要继续跟进

## 8.3 第 2 步：调用模型

`query.ts` 通过 `deps.callModel()` 调用 `services/api/claude.ts` 的流式接口。

传入内容包括：

- 预处理后的 `messagesForQuery`
- `fullSystemPrompt`
- `thinkingConfig`
- `tools`
- 当前模型
- fallback model
- 权限上下文 getter
- querySource
- agents
- fastMode / effort / taskBudget 等运行选项

这一步真正开始和模型对话。

## 8.4 第 3 步：消费流式输出

在 `for await` 中，`query.ts` 会不断接收模型产出的消息。

这里主要做四类事：

### A. 把 assistant message 收集起来

每条 assistant 输出都会进入 `assistantMessages`。

### B. 识别 `tool_use`

如果 assistant message 中包含 `tool_use block`：

- 把 block 放进 `toolUseBlocks`
- 设置 `needsFollowUp = true`

这说明本轮不是普通回答，而是进入工具调用分支。

### C. 流式执行工具

如果开启了 `StreamingToolExecutor`：

- 工具不必等整条 assistant 消息完全结束才执行
- 可边流边启动工具

这是这个架构里比较高级的一点：**模型和工具执行是部分重叠的**。

### D. 处理 withheld error

某些错误不会立即 yield 给上层，而是先“暂存”：

- prompt too long
- media size error
- max_output_tokens

原因是这些错误可能可恢复。

先隐藏，后续如果恢复失败再真正暴露给上层。

这是一个典型的“错误恢复优先”设计。

---

## 9. 模型接入层：`services/api/claude.ts`

## 9.1 它负责什么

`services/api/claude.ts` 是模型接入层。

它负责：

- 组装 Anthropic 请求
- 处理多 provider
- 处理 streaming / non-streaming fallback
- 处理 usage 累计
- 处理 prompt caching / cache edits
- 处理 retries / 529 / fallback model
- 产出标准化 `AssistantMessage`

注意：

- 它不做会话编排
- 不做工具执行编排
- 不关心 transcript

它只是负责“把请求稳定地送到模型，再把结果稳定地拿回来”。

## 9.2 为什么 `claude.ts` 还这么大

因为模型接入并不简单。它要同时处理：

- Anthropic 直连
- Bedrock
- Vertex
- Foundry
- OAuth / API key
- streaming stall watchdog
- non-streaming fallback
- token/cost/useage 统计
- cache headers / beta headers / thinking

所以它虽然叫 API 层，但实际上是“模型通信中台”。

## 9.3 关键算法 1：流式失败回退到非流式

在 streaming 过程中如果出现某些错误：

- 网络问题
- watchdog 超时
- 某些网关不支持 streaming

系统会：

1. 记录 streaming fallback 事件
2. 调用 `executeNonStreamingRequest()`
3. 生成一个正常的 `AssistantMessage`
4. 继续上层流程

这说明上层 `query.ts` 不需要关心“底层到底是流式还是非流式”，因为 `claude.ts` 已经把差异屏蔽掉了。

## 9.4 关键算法 2：usage 是“覆盖式更新 + 跨消息累计”

`claude.ts` 有两个关键函数：

- `updateUsage()`
- `accumulateUsage()`

为什么要两个？

### `updateUsage()`

处理单次流式请求内部的 usage 更新。

Anthropic 流式事件中的 usage 是累计值，不是增量值，所以：

- 不是简单相加
- 而是“用新的累计值覆盖旧值”

### `accumulateUsage()`

处理多轮 assistant 消息之间的 usage 汇总。

也就是：

- 一轮 query 内部，用 `updateUsage`
- 整个 turn 跨多次 assistant/tool loop，用 `accumulateUsage`

这是一个非常容易被误写的地方。

---

## 10. 工具执行架构

## 10.1 两层工具执行

工具执行分成两层：

### 第一层：编排层

文件：`src/services/tools/toolOrchestration.ts`

负责：

- 把多个 `tool_use` 分批
- 判断哪些可以并发
- 组织串行或并发执行

### 第二层：单工具执行层

文件：`src/services/tools/toolExecution.ts`

负责：

- 找到工具定义
- 输入 schema 校验
- 自定义业务校验
- hooks
- 权限审批
- 调用 `tool.call()`
- 生成 `tool_result`

所以：

- `toolOrchestration.ts` 负责“调度”
- `toolExecution.ts` 负责“执行”

## 10.2 `partitionToolCalls()`：并发分批算法

这是工具执行里一个很关键的算法。

目标：

- 连续只读工具可以并发执行
- 有副作用的工具必须串行执行

做法：

1. 对每个 `tool_use` 找到对应工具定义。
2. 用 `tool.isConcurrencySafe(input)` 判断是否可并发。
3. 把连续的并发安全工具归为一个 batch。
4. 非并发安全工具单独成 batch。

效果：

```text
[read, grep, web_fetch, edit, read, read]
  ->
[
  [read, grep, web_fetch],   // 并发 batch
  [edit],                    // 串行 batch
  [read, read]               // 并发 batch
]
```

这是一种非常实用的吞吐优化：

- 保持安全
- 也尽量并发

## 10.3 `StreamingToolExecutor`：边流边执行

文件：`src/services/tools/StreamingToolExecutor.ts`

这是一个更高级的优化层。

普通模式下，工具执行要等模型整轮 streaming 结束后再开始。

而 `StreamingToolExecutor` 允许：

- 一旦流里出现 `tool_use`
- 就尽早开始执行

它内部会维护每个工具的状态：

- `queued`
- `executing`
- `completed`
- `yielded`

并且处理：

- 并发安全工具并行
- 非并发工具独占执行
- progress 立即输出
- tool result 保持原始接收顺序

这说明架构设计者很明确地在追求：

**降低工具等待导致的整轮 turn 延迟。**

## 10.4 为什么 Bash 失败会中止兄弟工具

`StreamingToolExecutor` 里有个很有意思的策略：

- 如果某个 Bash tool 失败，会 abort 同批其他兄弟工具
- 但其他工具类型失败，不一定 abort 兄弟工具

原因是 Bash 经常有依赖链，比如：

```text
mkdir -> cd -> build -> deploy
```

前一步失败，后一步继续执行可能没有意义。

而像：

- FileRead
- WebFetch
- MCP 查询

往往相对独立。

这体现了一个重要设计理念：

**并发策略是按工具语义定制的，不是一刀切。**

---

## 11. 单工具执行的关键步骤

文件：`src/services/tools/toolExecution.ts`

一个工具调用会经历下面这些步骤。

## 11.1 找工具

通过 `findToolByName()` 在当前可用工具集合中查找。

找不到时：

- 直接生成错误型 `tool_result`
- 不会中断整个 turn

这体现了“单工具失败不等于整个 Agent 崩溃”。

## 11.2 输入 schema 校验

先做 `tool.inputSchema.safeParse(input)`。

如果失败：

- 生成 `InputValidationError`
- 回传给模型

这是因为模型生成参数经常不完全合法。

所以客户端要做第一层防线。

## 11.3 工具级业务校验

再调用 `tool.validateInput?.()`

这和 schema 校验不同：

- schema 校验解决“类型和结构”
- `validateInput` 解决“业务语义”

比如：

- 路径是否允许
- 参数组合是否合法
- 值域是否合理

## 11.4 Hooks

执行 `PreToolUse hooks`。

hooks 可以：

- 添加消息
- 修改输入
- 给出权限建议
- 阻止继续执行

所以 hooks 实际上是工具执行的“切面层”。

## 11.5 权限决策

通过 `resolveHookPermissionDecision()` 和 `canUseTool()` 走权限链。

决策可能是：

- `allow`
- `deny`
- `ask`

如果不是 `allow`：

- 会生成一个 `tool_result` 错误消息
- 这个错误消息再被回送给模型

注意这点很关键：

**权限拒绝不是抛异常结束，而是作为对话内容返回给模型。**

这样模型有机会：

- 换一种做法
- 向用户解释为什么需要权限
- 继续处理其他任务

这就是 Agent 架构相比普通函数调用的不同。

## 11.6 真正执行工具

只有前面全部通过后，才会执行：

```ts
tool.call(...)
```

执行期间：

- 可以上报 progress
- 可以记录 tracing / telemetry
- 可以输出 structured output attachment

最后再把工具结果映射成 `tool_result block`。

---

## 12. 消息模型：执行链里最核心的数据结构

虽然 vendor 镜像里没有直接展开 `src/types/message.ts` 文件，但从 `utils/messages.ts`、`types/logs.ts` 和所有调用侧可以反推出消息体系。

## 12.1 消息大类

执行链里的消息主要有 5 类：

### 1. `UserMessage`

表示：

- 用户输入
- 工具结果
- 中断说明
- 某些系统注入的 meta 用户消息

关键字段：

- `message.role = 'user'`
- `message.content`
- `isMeta`
- `toolUseResult`
- `uuid`

### 2. `AssistantMessage`

表示模型输出。

关键字段：

- `message.id`
- `message.model`
- `message.stop_reason`
- `message.usage`
- `message.content`
- `isApiErrorMessage`

### 3. `AttachmentMessage`

表示不会直接发给模型正文，但会参与执行链的附加信息。

例如：

- hook 输出
- structured output
- 额外上下文

### 4. `ProgressMessage`

表示执行过程中的 UI 进度。

例如：

- bash progress
- hook progress

### 5. `SystemMessage`

表示系统边界事件。

例如：

- compact boundary
- local command
- warning/info

## 12.2 为什么 `tool_result` 是 `UserMessage`

这点初学者最容易迷惑：

工具结果为什么不是 `SystemMessage`？

因为从模型 API 视角看，工具结果属于下一条“用户消息”中的 `tool_result block`。

也就是说：

```text
assistant: 我想调用工具
user: 这是工具结果
assistant: 我继续推理
```

所以客户端内部必须把工具结果包装成 `UserMessage`。

## 12.3 `createUserMessage()` 和 `createAssistantMessage()`

文件：`src/utils/messages.ts`

它们是消息体系的构造工厂。

意义在于：

- 统一 UUID / timestamp
- 统一 content 结构
- 统一 synthetic/meta 标识

这样整个执行链不需要到处手写消息对象。

## 12.4 `normalizeMessages()`：一块一消息

为了很多后续处理方便，系统会把多 block 的消息拆开。

例如：

```text
一个 assistant message 包含 3 个 content block
```

会被拆成多个 normalized message。

这样做的好处：

- 便于 UI 排序
- 便于 progress / hook / tool_result 对齐
- 便于构建查找索引

本质上是把“粗粒度消息”变成“细粒度操作单元”。

## 12.5 `normalizeMessagesForAPI()`：API 前的最终清洗

这是消息层最重要的算法之一。

它会在发 API 前做：

- 附件重排
- virtual message 过滤
- 合并连续 user message
- 过滤不该继续发给 API 的 synthetic error
- strip 多余 document/image block
- 修正 tool reference
- 规范 assistant tool input

为什么要单独有这一层？

因为“UI 里能显示的消息”不等于“适合发给 API 的消息”。

系统必须在真正发 API 前进行一次“净化”。

## 12.6 `ensureToolResultPairing()`：保证 tool_use/tool_result 成对

这是执行链里非常关键的防御性算法。

它做两件事：

1. 缺少 `tool_result` 时，补 synthetic error result。
2. 多余的 orphaned `tool_result` 时，剔除掉。

为什么必须做？

因为 Anthropic 的消息协议对 `tool_use` / `tool_result` 配对要求非常严格。

一旦不成对，就会被 API 拒绝。

所以客户端宁可：

- 自动修复
- 或在 strict mode 直接抛错

也不能让错误消息链继续流入模型。

这体现了这个架构很强的一点：

**在协议边界做强一致性保护。**

---

## 13. `ToolUseContext`：工具执行的运行时上下文

文件：`src/Tool.ts`

这是整个对话执行架构里第二个必须理解的核心结构。

## 13.1 它是什么

`ToolUseContext` 可以理解成：

```text
工具执行期间的运行时世界
```

里面包含：

- 当前可用 tools / commands / mcpClients
- abortController
- readFileState
- getAppState / setAppState
- handleElicitation
- addNotification
- appendSystemMessage
- requestPrompt
- updateFileHistoryState
- updateAttributionState
- messages
- agentId / agentType
- queryTracking
- contentReplacementState

## 13.2 为什么不只传工具 input

如果只给工具传 input，它只能做纯函数式执行。

但这个项目里的工具不是纯函数，它们需要：

- 看当前消息
- 读权限模式
- 写 UI 状态
- 发通知
- 记录文件历史
- 中断自己或兄弟工具
- 调用 MCP / prompt / hooks

所以工具需要完整运行时上下文。

这也是 Agent 工具系统和普通函数库的差异。

## 13.3 `queryTracking`

`ToolUseContext` 里有个小但重要的字段：

```ts
queryTracking = { chainId, depth }
```

作用：

- 把同一条 query 链串起来
- 便于 telemetry 追踪多轮 tool loop

对于调试很重要，因为一次用户请求可能会触发很多 API 请求和工具调用。

---

## 14. 关键恢复算法

这个架构最有技术含量的部分，不是“调用模型”，而是“出问题后怎么恢复”。

## 14.1 Prompt Too Long 恢复

恢复顺序大致是：

1. 先试 `contextCollapse` drain
2. 再试 `reactiveCompact`
3. 再不行才真正把错误暴露给用户

这个顺序体现了设计取舍：

- 先尝试保留更多原始上下文
- 不行再退化成摘要

## 14.2 Max Output Tokens 恢复

恢复策略有两级：

### 第一级：升高 `maxOutputTokens`

如果当前是默认上限，先升级到更高上限重试。

### 第二级：插入恢复消息继续生成

如果还不够，就给模型加一条 meta 用户消息：

```text
输出被截断了，请直接从中断处继续
```

然后再跑下一轮。

这其实是一种“协议内 continuation”算法。

## 14.3 Streaming 失败恢复

如果 streaming 中断：

- 有些情况切到 non-streaming fallback
- 有些情况传播为可恢复错误
- 有些情况直接中止并补全缺失 tool_result

这样做是为了避免两类问题：

1. 用户看不到任何结果。
2. tool_use 已经发出了，但 tool_result 没跟上，消息链损坏。

## 14.4 中断恢复

如果用户中断：

- `query.ts` 会先消费剩余工具执行器结果
- 必要时合成错误型 `tool_result`
- 再生成用户中断消息

设计目标不是“立刻停下就完了”，而是：

**停下时也要把消息链补成合法状态。**

---

## 15. stop hooks 和 turn 收尾

当本轮没有新的 `tool_use` 之后，并不代表马上结束。

还会进入 `handleStopHooks()`。

文件：`src/query/stopHooks.ts`

它负责：

- 执行 Stop hooks
- PromptSuggestion
- extract memories
- auto dream
- task completed hook
- teammate idle hook

也就是说，turn 的“结束”本身也是一个可扩展阶段。

这是一个非常好的架构点：

- 把收尾逻辑从主 query loop 主路径里分离
- 让“turn 完成后”变成正式生命周期节点

---

## 16. token budget 算法

文件：`src/query/tokenBudget.ts`

这也是一个很适合初学者学习的小算法。

## 16.1 它要解决什么问题

某些 turn 很长，不能无限继续。

系统要决定：

- 还值不值得继续追问模型
- 是否已经进入收益递减

## 16.2 核心判断

`checkTokenBudget()` 会看：

- 当前 turn tokens 占预算百分比
- 连续 continuation 次数
- 最近两次 continuation 增量是否太小

如果：

- 预算还没接近上限
- 而且还没有明显收益递减

就继续。

否则停止。

这不是一个复杂算法，但非常实用：

- 不是只看“是否超预算”
- 而是看“继续是否还有意义”

---

## 17. 这套架构最值得学习的设计点

## 17.1 会话级与 turn 级分离

- `QueryEngine` 管会话
- `query.ts` 管 turn

这让复杂逻辑边界更清晰。

## 17.2 协议边界防御

- `normalizeMessagesForAPI()`
- `ensureToolResultPairing()`

这些逻辑确保发给模型的消息永远尽量合法。

## 17.3 工具执行可并发但不盲并发

- 用 `isConcurrencySafe()` 分批
- 用 `StreamingToolExecutor` 提升吞吐
- 又保留语义安全

这是非常成熟的工程化取舍。

## 17.4 错误优先恢复而不是直接失败

无论是：

- prompt too long
- max output tokens
- streaming fallback
- user interrupt

系统都优先尝试恢复消息链和执行链，而不是直接崩掉。

## 17.5 所有能力都围绕消息链工作

这个项目最核心的抽象不是“函数调用”，而是“消息流”。

工具、权限、hooks、compact、memory、resume，最终都落到消息链上。

理解了这一点，就理解了这套架构为什么长这样。

---

## 18. 初学者建议的阅读顺序

如果你要自己顺着代码继续读，建议按这个顺序：

1. `src/utils/processUserInput/processUserInput.ts`
2. `src/utils/processUserInput/processTextPrompt.ts`
3. `src/QueryEngine.ts`
4. `src/query.ts`
5. `src/services/api/claude.ts`
6. `src/services/tools/toolOrchestration.ts`
7. `src/services/tools/toolExecution.ts`
8. `src/services/tools/StreamingToolExecutor.ts`
9. `src/utils/messages.ts`
10. `src/Tool.ts`

阅读策略建议：

- 第一次只看主流程，不要一开始追每个 feature flag。
- 第二次只看消息结构。
- 第三次只看工具执行与权限链。
- 第四次再看 compact、fallback、budget 这些恢复逻辑。

---

## 19. 一句话总结

Claude-code-open 的“对话执行架构”本质上是一个**消息驱动的多轮 Agent 执行循环**：

- 输入先被翻译成消息
- `QueryEngine` 负责会话编排
- `query.ts` 负责单轮循环状态机
- `claude.ts` 负责模型通信
- `toolExecution.ts` 负责工具执行
- 所有恢复、压缩、权限、hook 都围绕消息链进行

如果把它看成“聊天程序”，会觉得复杂。

如果把它看成“一个能持续运行、能自我修复、能调用外部能力的 Agent Runtime”，这个架构就非常合理。
