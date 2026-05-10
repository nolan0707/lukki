# OpenClaw 同一 session 内多条用户消息的并行处理分析

## 结论

OpenClaw 不支持同一个 `sessionKey` 下并行启动多个独立 agent turn。同一 session 同一时间只能有一个 active reply run。后续用户消息会按队列策略处理：默认尝试注入当前 active run 的下一个 runtime/model boundary；不能注入或配置为 followup/collect 时，进入 followup queue，在当前 run 结束后再处理。

因此，准确表述是：

- 不同 session 可以并行处理。
- 同一个 session 内不会并行跑多个 agent turn。
- 同一个 session 的后续用户消息可以在 active run 忙时被接收、去重、排队、合并，或在支持 steering 的 runtime 中注入当前 run。

## 关键证据

### 1. active run 注册表按 sessionKey 加锁

`src/auto-reply/reply/reply-run-registry.ts:196` 的 `createReplyOperation(...)` 创建 reply operation 时要求 canonical `sessionKey` 和 `sessionId`。如果 `replyRunState.activeRunsByKey` 已经有相同 `sessionKey`，会抛出 `ReplyRunAlreadyActiveError`：

```ts
if (replyRunState.activeRunsByKey.has(sessionKey)) {
  throw new ReplyRunAlreadyActiveError(sessionKey);
}
```

这说明同一 `sessionKey` 下不能同时注册两个 active reply operation。这个锁是判断“同一 session 是否能并行跑多个 agent turn”的核心证据。

### 2. 默认队列模式是 steer

`src/auto-reply/reply/queue/settings.ts:23` 的 `resolveQueueSettings(...)` 会按以下优先级解析队列模式：

- inline mode
- session entry 的 `queueMode`
- channel 级配置
- 全局 `messages.queue.mode`
- 默认值

默认值来自 `defaultQueueModeForChannel(...)`，当前直接返回 `"steer"`。也就是说，未显式配置时，忙时消息默认走 steering 语义。

### 3. active run streaming 时先尝试注入当前 run

`src/auto-reply/reply/agent-runner.ts:1042` 在 `effectiveShouldSteer && isStreaming` 时调用 `queueEmbeddedPiMessage(...)`：

```ts
const steered = queueEmbeddedPiMessage(steerSessionId, followupRun.prompt, {
  steeringMode: resolvePiSteeringModeForQueueMode(resolvedQueue.mode),
  ...(resolvedQueue.debounceMs !== undefined ? { debounceMs: resolvedQueue.debounceMs } : {}),
});
if (steered && !effectiveShouldFollowup) {
  await touchActiveSessionEntry();
  typing.cleanup();
  return undefined;
}
```

这条路径不是启动第二个 agent run，而是把后续用户消息交给当前 active runtime 的 steering 队列。成功 steer 且不需要 backlog/followup 时，当前 inbound 处理直接返回，不再创建新的 reply operation。

### 4. 不能或不应立即运行时进入 followup queue

`src/auto-reply/reply/agent-runner.ts:1082` 在 `activeRunQueueAction === "enqueue-followup"` 时调用 `enqueueFollowupRun(...)`，然后根据 active run 是否仍存活决定是否立即 drain：

```ts
enqueueFollowupRun(queueKey, followupRun, resolvedQueue, "message-id", queuedRunFollowupTurn, false);
const queuedBehindActiveRun = isRunActive?.() === true;
if (!queuedBehindActiveRun) {
  scheduleFollowupDrain(queueKey, queuedRunFollowupTurn);
}
```

如果当前 run 仍在运行，消息留在队列中；如果当前 run 已结束，则立即调度 drain。这里仍然保持同一 session 的串行执行。

### 5. followup queue 是按 key 维护的共享队列

`src/auto-reply/reply/queue/state.ts:24` 使用全局 map 维护 `FOLLOWUP_QUEUES`。队列状态包括：

- `items`
- `draining`
- `lastEnqueuedAt`
- `mode`
- `debounceMs`
- `cap`
- `dropPolicy`
- `lastRun`

`src/auto-reply/reply/queue/enqueue.ts:59` 的 `enqueueFollowupRun(...)` 负责将 busy-session 后续消息加入队列，并支持按 message id 或 prompt 去重。队列 keyed by `queueKey`，通常是 `sessionKey ?? sessionIdFinal`，见 `src/auto-reply/reply/get-reply-run.ts:913` 附近。

### 6. collect/followup drain 是串行消费

`src/auto-reply/reply/queue/drain.ts:170` 的 `scheduleFollowupDrain(...)` 用 `beginQueueDrain(...)` 标记队列正在 drain，然后在 async loop 中消费：

- `collect` 模式：等待 debounce，按兼容性/授权上下文合并多个 queued messages 为一个 followup turn。
- 非 `collect` 模式：按队列顺序逐个 drain。
- drain 完成后清理队列；未完成则递归调度继续 drain。

这说明 queued followup 是排队串行处理，不是并行展开。

## 文档语义

`docs/concepts/queue-steering.md:10` 明确说明：当消息在 session run 正在 streaming 时到达，OpenClaw 可以把消息发送进 active runtime，而不是为同一 session 启动另一个 run。

`docs/concepts/queue-steering.md:17` 说明 steering 不会打断已在执行的 tool call。Pi 在 model boundary 检查 queued steering messages：

1. assistant 请求 tool calls。
2. Pi 执行当前 assistant message 的 tool-call batch。
3. Pi 发出 turn end event。
4. Pi drain queued steering messages。
5. Pi 在下一次 LLM call 前把这些消息追加为 user messages。

`docs/concepts/queue-steering.md:48` 的模式表给出各模式语义：

- `steer`: 默认。将 queued steering messages 在下一个 runtime boundary 一起注入。
- `queue`: legacy serialized steering，一条一条注入。
- `steer-backlog`: 同 `steer`，但还保留 later followup。
- `followup`: 不 steer 当前 run，之后再跑 queued messages。
- `collect`: 不 steer 当前 run，active run 结束后 debounce 并合并兼容消息。
- `interrupt`: abort 当前 run，然后启动最新消息。

## 运行时差异

### Pi runtime

Pi 有内部 steering queue。默认 `steer` 模式下，active run 执行工具期间收到的消息会累积，并在下一个 model boundary 作为用户消息进入上下文。它不会中断当前工具调用，也不会创建第二个同 session run。

### Native Codex app-server harness

文档说明 native Codex app-server harness 使用 `turn/steer`。`steer` 会在 quiet window 后把收集到的用户输入按到达顺序批量发送为一个 `turn/steer` 请求；`queue` 则保持 legacy serialized shape，发送多个单独的 `turn/steer` 请求。

Codex review 和 manual compaction turns 拒绝 same-turn steering；当 runtime 不接受 steering 且模式允许 fallback 时，OpenClaw 回退到 followup queue。

## 配置影响

同一 session 忙时消息的行为由 `messages.queue` 和 session-level queue override 影响。关键配置包括：

- `mode`: `steer | queue | steer-backlog | followup | collect | interrupt`
- `debounceMs`: followup/collect debounce；native Codex `steer` quiet window 也使用它
- `cap`: queue cap
- `drop`: queue 满时 drop policy
- `byChannel`: channel-specific queue mode
- `debounceMsByChannel`: channel-specific debounce

默认 `mode` 是 `steer`，默认 `debounceMs` 是 `500`，默认 `cap` 是 `20`，默认 `dropPolicy` 是 `summarize`，见 `src/auto-reply/reply/queue/state.ts:16` 和 `src/auto-reply/reply/queue/settings.ts:23`。

## 边界情况

### active run 不 streaming

`steer` 只在 `isStreaming` 时先尝试注入。如果 active run 存在但不能 steering，`resolveActiveRunQueueAction(...)` 会让 `steer`/`queue` 路径进入 `enqueue-followup`，见 `src/auto-reply/reply/queue-policy.ts:9`。

### interrupt 模式

`interrupt` 是例外语义：它不会并行处理，而是 abort 当前 active run，然后等待当前 active run 结束，再运行新消息。对应逻辑在 `src/auto-reply/reply/get-reply-run-queue.ts:23`：`queueMode === "interrupt"` 时先 abort，再 `waitForActiveRunEnd(...)`。

### heartbeat

当 session 已 active 且 inbound 是 heartbeat，`resolveActiveRunQueueAction(...)` 返回 `drop`，避免 heartbeat 在 busy session 中排队制造额外 agent work。

### 多用户群聊

同一 channel/session 中不同用户的消息仍然进入同一个 session 队列或 steering 流。collect 模式会按授权相关字段拆分兼容组，避免把不同执行授权上下文的消息错误合并。相关逻辑见 `src/auto-reply/reply/queue/drain.ts:70` 的 `resolveFollowupAuthorizationKey(...)`。

## 用户可见行为

从用户视角看，可能出现几种情况：

- agent 正在处理 A，用户又发 B；默认情况下 B 可能被注入 A 的当前 run，下一次模型决策会看到 B。
- 如果 runtime 不能 steering，B 会被排队，A 完成后再处理。
- 如果短时间内多条消息进入 `collect` 队列，OpenClaw 可能把它们合成一个 followup prompt，标题类似 `[Queued messages while agent was busy]`。
- 如果配置为 `interrupt`，后来的消息会中止当前 run 并替代它。

这些都不是“同 session 并行多个 agent turn”。

## 对问题的直接回答

“同一个 session 的多个用户消息在 OpenClaw 中支持并行处理吗？”

答案：不支持同 session 多个 agent turn 并行。OpenClaw 通过 `replyRunRegistry` 保证同一 `sessionKey` 单 active run。忙时消息默认走 `steer`，即在 runtime boundary 注入当前 run；不能注入或配置为 followup/collect 时进入 per-session followup queue，按顺序或合并后继续处理。

