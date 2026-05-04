# Claude-code-open Agent 任务 Token 消耗统计流程

## 1. 文档目标

本文深度阅读 `vendor/Claude-code-open` 中与 token usage、cost、SDK result 输出相关的代码，回答两个问题：

1. 一次 agent 任务执行过程中，token 消耗是如何从 Anthropic API streaming 事件流进入本地统计系统的。
2. SDK 调用方应该从哪里获取最终消耗的 token 数。

本文只做静态代码梳理，不修改源码。

> 说明：当前工作区没有暴露可用的 GitNexus MCP resource/tool，因此本文基于本地源码阅读完成。

---

## 2. 一句话结论

Claude Code 的 token 消耗统计有两层：

1. **API 调用层统计**
   - 入口在 `src/services/api/claude.ts`
   - 从 streaming 事件 `message_start` 和 `message_delta` 中读取 Anthropic SDK 返回的 `usage`
   - 在 `message_delta` 里拿到最终 usage 后，写回 assistant message，并调用 `addToTotalSessionCost(...)` 更新全局 cost/model usage

2. **SDK 结果层统计**
   - 入口在 `src/QueryEngine.ts`
   - SDK 层监听 query 产出的 `stream_event`
   - 每次 `message_start/message_delta` 更新当前 message usage
   - 到 `message_stop` 时把当前 message usage 累加到 `this.totalUsage`
   - 最终 SDK `result` 消息返回 `usage`、`modelUsage`、`total_cost_usd`

SDK 使用方最直接的获取方式是消费 `query(...)` 的最终 `result` 消息：

```ts
for await (const msg of query({ prompt, options })) {
  if (msg.type === "result") {
    console.log(msg.usage)
    console.log(msg.modelUsage)
    console.log(msg.total_cost_usd)
  }
}
```

---

## 3. 关键文件

| 文件 | 作用 |
|---|---|
| `src/services/api/claude.ts` | 调用 Anthropic SDK，处理 streaming events，提取 usage，计算 cost |
| `src/cost-tracker.ts` | 将 usage 按模型累加为全局 session cost/model usage |
| `src/bootstrap/state.ts` | 保存全局 cost、modelUsage、total input/output/cache tokens |
| `src/QueryEngine.ts` | SDK/headless 查询执行器，累加 SDK result 的 `usage` |
| `src/entrypoints/sdk/coreSchemas.ts` | SDK result message schema，声明 `usage`、`modelUsage`、`total_cost_usd` |
| `src/services/api/emptyUsage.ts` | 零值 usage 对象 |
| `src/utils/tokens.ts` | usage 到 token count 的辅助计算 |
| `src/tasks/LocalAgentTask/LocalAgentTask.tsx` | 本地/后台 subagent 任务进度 token 统计 |
| `src/tools/AgentTool/AgentTool.tsx` | AgentTool 执行时把 subagent progress/notification usage 发给 SDK |

---

## 4. API 调用层：usage 从哪里来

核心逻辑在 `src/services/api/claude.ts` 的 `queryModelWithStreaming(...)`。

### 4.1 创建 streaming 请求

代码通过 Anthropic SDK 发起流式请求：

```ts
const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    { signal, headers },
  )
  .withResponse()
```

后续 `for await (const part of stream)` 消费 `BetaRawMessageStreamEvent`。

### 4.2 `message_start` 初始化 usage

当收到 `message_start`：

```ts
case 'message_start': {
  partialMessage = part.message
  ttftMs = Date.now() - start
  usage = updateUsage(usage, part.message?.usage)
  break
}
```

这里通常能拿到输入侧 token：

- `input_tokens`
- `cache_creation_input_tokens`
- `cache_read_input_tokens`

注意：这时 `output_tokens` 往往还不是最终值。

### 4.3 `content_block_stop` 先 yield assistant message

当一个 content block 结束时，代码会构造 `AssistantMessage` 并 yield 给上层：

```ts
const m: AssistantMessage = {
  message: {
    ...partialMessage,
    content: normalizeContentFromAPI(...),
  },
  type: 'assistant',
  ...
}
newMessages.push(m)
yield m
```

此时 assistant message 的 `usage` 可能还不是最终 usage，因为最终 `output_tokens` 和 `stop_reason` 通常在之后的 `message_delta` 才到。

### 4.4 `message_delta` 更新最终 usage

当收到 `message_delta`：

```ts
case 'message_delta': {
  usage = updateUsage(usage, part.usage)
  stopReason = part.delta.stop_reason

  const lastMsg = newMessages.at(-1)
  if (lastMsg) {
    lastMsg.message.usage = usage
    lastMsg.message.stop_reason = stopReason
  }

  const costUSDForPart = calculateUSDCost(resolvedModel, usage)
  costUSD += addToTotalSessionCost(
    costUSDForPart,
    usage,
    options.model,
  )
  break
}
```

这里有三个重要动作：

1. 用 `part.usage` 合并最终 token usage。
2. 直接 mutation 最后一个 assistant message 的 `message.usage` 和 `stop_reason`。
3. 用最终 usage 计算 cost，并写入全局 session cost。

源码注释特别强调不能替换整个 `message` 对象，而是要直接改属性，因为 transcript 写入队列持有的是旧对象引用。

### 4.5 `updateUsage(...)` 的合并规则

`updateUsage(...)` 位于 `src/services/api/claude.ts`，注释说明 Anthropic streaming API 返回的是累计 usage，不是增量 delta。

合并规则大致是：

```ts
input_tokens:
  partUsage.input_tokens !== null && partUsage.input_tokens > 0
    ? partUsage.input_tokens
    : usage.input_tokens

cache_creation_input_tokens:
  partUsage.cache_creation_input_tokens !== null &&
  partUsage.cache_creation_input_tokens > 0
    ? partUsage.cache_creation_input_tokens
    : usage.cache_creation_input_tokens

cache_read_input_tokens:
  partUsage.cache_read_input_tokens !== null &&
  partUsage.cache_read_input_tokens > 0
    ? partUsage.cache_read_input_tokens
    : usage.cache_read_input_tokens

output_tokens:
  partUsage.output_tokens ?? usage.output_tokens
```

为什么 input/cache 字段要用 `> 0` 保护：

- input/cache token 通常在 `message_start` 设置，且整次响应内保持不变。
- `message_delta` 可能显式返回 0。
- 如果无脑覆盖，会把真实 input/cache usage 覆盖成 0。

---

## 5. Cost Tracker：全局 token/cost 如何累计

`src/cost-tracker.ts` 中的关键函数是：

```ts
export function addToTotalSessionCost(
  cost: number,
  usage: Usage,
  model: string,
): number
```

内部先调用：

```ts
const modelUsage = addToTotalModelUsage(cost, usage, model)
addToTotalCostState(cost, modelUsage, model)
```

`addToTotalModelUsage(...)` 会按模型累加：

```ts
modelUsage.inputTokens += usage.input_tokens
modelUsage.outputTokens += usage.output_tokens
modelUsage.cacheReadInputTokens += usage.cache_read_input_tokens ?? 0
modelUsage.cacheCreationInputTokens += usage.cache_creation_input_tokens ?? 0
modelUsage.webSearchRequests += usage.server_tool_use?.web_search_requests ?? 0
modelUsage.costUSD += cost
```

这些数据最终存入 `src/bootstrap/state.ts` 的全局状态：

```ts
STATE.modelUsage[model] = modelUsage
STATE.totalCostUSD += cost
```

全局读取函数包括：

```ts
getTotalInputTokens()
getTotalOutputTokens()
getTotalCacheReadInputTokens()
getTotalCacheCreationInputTokens()
getTotalCostUSD()
getModelUsage()
```

这些函数主要服务于 CLI status line、统计展示、SDK result 的 `modelUsage` 和 `total_cost_usd`。

---

## 6. SDK 层：result.usage 如何累计

SDK/headless 执行核心在 `src/QueryEngine.ts`。

### 6.1 QueryEngine 持有 totalUsage

`QueryEngine` 初始化时：

```ts
private totalUsage: NonNullableUsage

constructor(config: QueryEngineConfig) {
  ...
  this.totalUsage = EMPTY_USAGE
}
```

这里的 `this.totalUsage` 是 SDK 会话内累计 usage。

### 6.2 监听 query 产出的 stream_event

`QueryEngine.submitMessage(...)` 内部调用底层 `query(...)`：

```ts
for await (const message of query({
  messages,
  systemPrompt,
  userContext,
  systemContext,
  canUseTool: wrappedCanUseTool,
  toolUseContext: processUserInputContext,
  fallbackModel,
  querySource: 'sdk',
  maxTurns,
  taskBudget,
})) {
  ...
}
```

当收到 `stream_event`：

```ts
case 'stream_event':
  if (message.event.type === 'message_start') {
    currentMessageUsage = EMPTY_USAGE
    currentMessageUsage = updateUsage(
      currentMessageUsage,
      message.event.message.usage,
    )
  }

  if (message.event.type === 'message_delta') {
    currentMessageUsage = updateUsage(
      currentMessageUsage,
      message.event.usage,
    )
  }

  if (message.event.type === 'message_stop') {
    this.totalUsage = accumulateUsage(
      this.totalUsage,
      currentMessageUsage,
    )
  }
```

也就是说：

- `currentMessageUsage` 是当前单次模型响应的 usage。
- `this.totalUsage` 是当前 SDK 查询过程累计的 usage。
- 累加点不是 `message_delta`，而是 `message_stop`。

### 6.3 accumulateUsage 的规则

`accumulateUsage(...)` 位于 `src/services/api/claude.ts`：

```ts
input_tokens:
  totalUsage.input_tokens + messageUsage.input_tokens

cache_creation_input_tokens:
  totalUsage.cache_creation_input_tokens +
  messageUsage.cache_creation_input_tokens

cache_read_input_tokens:
  totalUsage.cache_read_input_tokens +
  messageUsage.cache_read_input_tokens

output_tokens:
  totalUsage.output_tokens + messageUsage.output_tokens
```

同时会累加 server tool use：

```ts
server_tool_use: {
  web_search_requests:
    totalUsage.server_tool_use.web_search_requests +
    messageUsage.server_tool_use.web_search_requests,
  web_fetch_requests:
    totalUsage.server_tool_use.web_fetch_requests +
    messageUsage.server_tool_use.web_fetch_requests,
}
```

`service_tier`、`inference_geo`、`iterations`、`speed` 这类字段使用最近一次 message 的值。

---

## 7. SDK result 返回哪些统计字段

`src/QueryEngine.ts` 在成功、失败、预算超限、max turns 等路径都会 yield `type: 'result'` 的消息。

成功路径示例：

```ts
yield {
  type: 'result',
  subtype: 'success',
  is_error: isApiError,
  duration_ms: Date.now() - startTime,
  duration_api_ms: getTotalAPIDuration(),
  num_turns: turnCount,
  result: textResult,
  stop_reason: lastStopReason,
  session_id: getSessionId(),
  total_cost_usd: getTotalCost(),
  usage: this.totalUsage,
  modelUsage: getModelUsage(),
  permission_denials: this.permissionDenials,
  structured_output: structuredOutputFromTool,
  uuid: randomUUID(),
}
```

SDK schema 在 `src/entrypoints/sdk/coreSchemas.ts` 中声明：

```ts
export const SDKResultSuccessSchema = lazySchema(() =>
  z.object({
    type: z.literal('result'),
    subtype: z.literal('success'),
    ...
    total_cost_usd: z.number(),
    usage: NonNullableUsagePlaceholder(),
    modelUsage: z.record(z.string(), ModelUsageSchema()),
    ...
  }),
)
```

错误 result 也同样包含：

- `total_cost_usd`
- `usage`
- `modelUsage`

因此 SDK 调用方不需要自己监听 `message_delta`，通常只需要读取最终 `result` 消息。

---

## 8. SDK 如何获取消耗 token 数

### 8.1 获取总 token

SDK 消费方式：

```ts
import { query } from "@anthropic-ai/claude-code"

for await (const msg of query({
  prompt: "分析当前项目的执行流程",
  options: {
    cwd: process.cwd(),
  },
})) {
  if (msg.type !== "result") continue

  const usage = msg.usage as {
    input_tokens: number
    output_tokens: number
    cache_creation_input_tokens?: number
    cache_read_input_tokens?: number
  }

  const totalTokens =
    usage.input_tokens +
    (usage.cache_creation_input_tokens ?? 0) +
    (usage.cache_read_input_tokens ?? 0) +
    usage.output_tokens

  console.log({
    totalTokens,
    inputTokens: usage.input_tokens,
    outputTokens: usage.output_tokens,
    cacheCreationInputTokens: usage.cache_creation_input_tokens ?? 0,
    cacheReadInputTokens: usage.cache_read_input_tokens ?? 0,
    totalCostUsd: msg.total_cost_usd,
    modelUsage: msg.modelUsage,
  })
}
```

### 8.2 推荐读取字段

如果只关心本次 SDK query 的总 token：

```ts
msg.usage
```

如果需要按模型拆分：

```ts
msg.modelUsage
```

如果需要美元成本：

```ts
msg.total_cost_usd
```

### 8.3 usage 字段公式

`src/utils/tokens.ts` 给出了 context/token count 的通用公式：

```ts
export function getTokenCountFromUsage(usage: Usage): number {
  return (
    usage.input_tokens +
    (usage.cache_creation_input_tokens ?? 0) +
    (usage.cache_read_input_tokens ?? 0) +
    usage.output_tokens
  )
}
```

SDK result 的 `usage` 可按同样方式计算总 token。

---

## 9. modelUsage 与 usage 的区别

### 9.1 `usage`

`msg.usage` 是 Anthropic usage 风格，字段是 snake_case：

```ts
{
  input_tokens: number
  cache_creation_input_tokens: number
  cache_read_input_tokens: number
  output_tokens: number
  server_tool_use: {
    web_search_requests: number
    web_fetch_requests: number
  }
  service_tier: string
  cache_creation: {
    ephemeral_1h_input_tokens: number
    ephemeral_5m_input_tokens: number
  }
  inference_geo: string
  iterations: unknown[]
  speed: string
}
```

它表示 SDK 查询过程累计 usage。

### 9.2 `modelUsage`

`msg.modelUsage` 是 Claude Code 自己整理的 per-model usage，字段是 camelCase：

```ts
{
  [modelName: string]: {
    inputTokens: number
    outputTokens: number
    cacheReadInputTokens: number
    cacheCreationInputTokens: number
    webSearchRequests: number
    costUSD: number
    contextWindow: number
    maxOutputTokens: number
  }
}
```

它来自 `cost-tracker.ts` 和 `bootstrap/state.ts` 的全局统计，更适合做：

- 按模型聚合
- 成本展示
- 模型维度的 token 分布分析

---

## 10. Subagent / AgentTool 任务进度统计

除了主 SDK result，Claude Code 还有一套 subagent 任务进度 usage。

这套逻辑在 `src/tasks/LocalAgentTask/LocalAgentTask.tsx`：

```ts
export type ProgressTracker = {
  toolUseCount: number
  latestInputTokens: number
  cumulativeOutputTokens: number
  recentActivities: ToolActivity[]
}
```

更新逻辑：

```ts
const usage = message.message.usage

tracker.latestInputTokens =
  usage.input_tokens +
  (usage.cache_creation_input_tokens ?? 0) +
  (usage.cache_read_input_tokens ?? 0)

tracker.cumulativeOutputTokens += usage.output_tokens
```

总 token：

```ts
export function getTokenCountFromTracker(tracker: ProgressTracker): number {
  return tracker.latestInputTokens + tracker.cumulativeOutputTokens
}
```

这里和 SDK result 的累计方式不同，原因在源码注释里写得很明确：

- Claude API 的 `input_tokens` 是按 turn 累计上下文计算的。
- 如果 subagent 每轮都把 input_tokens 相加，会重复计算历史上下文。
- 所以 subagent 进度统计保留最新 input tokens，累加每轮 output tokens。

`src/tools/AgentTool/AgentTool.tsx` 会把这些统计发成 SDK task 事件：

```ts
enqueueSdkEvent({
  type: 'system',
  subtype: 'task_notification',
  task_id: foregroundTaskId,
  ...
  usage: {
    total_tokens: progress.tokenCount,
    tool_uses: progress.toolUseCount,
    duration_ms: Date.now() - agentStartTime,
  },
})
```

SDK schema 中对应：

- `system/subtype: task_progress`
- `system/subtype: task_notification`

字段形态：

```ts
usage: {
  total_tokens: number
  tool_uses: number
  duration_ms: number
}
```

因此：

- 如果你关心整个 SDK query 的最终消耗，看最终 `result.usage`。
- 如果你关心某个后台/前台 subagent 任务的任务级进度，看 `task_progress` 或 `task_notification` 的 `usage.total_tokens`。

---

## 11. 完整链路图

```text
Anthropic SDK streaming response
  |
  | message_start
  |   usage = updateUsage(EMPTY_USAGE, part.message.usage)
  |
  | content_block_stop
  |   yield assistant message
  |   此时 usage/stop_reason 可能还不是最终值
  |
  | message_delta
  |   usage = updateUsage(usage, part.usage)
  |   lastMsg.message.usage = usage
  |   lastMsg.message.stop_reason = stopReason
  |   cost = calculateUSDCost(model, usage)
  |   addToTotalSessionCost(cost, usage, model)
  |
  | message_stop
  v

query(...) yields stream_event
  |
  v

QueryEngine
  |
  | message_start:
  |   currentMessageUsage = updateUsage(EMPTY_USAGE, event.message.usage)
  |
  | message_delta:
  |   currentMessageUsage = updateUsage(currentMessageUsage, event.usage)
  |
  | message_stop:
  |   totalUsage = accumulateUsage(totalUsage, currentMessageUsage)
  |
  v

SDK result message
  |
  | usage: totalUsage
  | modelUsage: getModelUsage()
  | total_cost_usd: getTotalCost()
  v

SDK consumer
```

---

## 12. 使用建议

### 12.1 只需要总 token

读取最终 `result.usage`，按以下公式计算：

```ts
total =
  input_tokens +
  cache_creation_input_tokens +
  cache_read_input_tokens +
  output_tokens
```

### 12.2 需要按模型统计

读取最终 `result.modelUsage`：

```ts
for (const [model, usage] of Object.entries(msg.modelUsage)) {
  const total =
    usage.inputTokens +
    usage.cacheCreationInputTokens +
    usage.cacheReadInputTokens +
    usage.outputTokens
}
```

### 12.3 需要成本

读取：

```ts
msg.total_cost_usd
```

或按模型读取：

```ts
msg.modelUsage[model].costUSD
```

### 12.4 需要 subagent 任务进度

监听 SDK system message：

```ts
if (
  msg.type === "system" &&
  (msg.subtype === "task_progress" ||
    msg.subtype === "task_notification")
) {
  console.log(msg.usage?.total_tokens)
}
```

---

## 13. 注意事项

1. `message_delta` 中的 usage 是累计值，不是增量值。
2. `message_start` 通常提供 input/cache token，`message_delta` 才提供最终 output token。
3. SDK result 的 `usage` 是 SDK 查询过程累计 usage，不是单个 assistant message 的 usage。
4. `modelUsage` 来自全局 cost tracker，适合按模型和成本维度分析。
5. subagent 的 `task_progress/task_notification.usage.total_tokens` 是任务进度口径，不等同于主 SDK result 的累计口径。
6. 对 cache token 要显式计入，否则会低估上下文/token 消耗。

