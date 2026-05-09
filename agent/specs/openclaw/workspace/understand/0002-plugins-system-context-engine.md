# Codex Harness 与 ContextEngine 交互机制分析

## 1. 结论

以 `extensions/codex` 为例，ContextEngine 在 Harness 中承担的是“OpenClaw 可见上下文的装配、维护和旁路索引”职责；Codex app-server 仍然拥有自己的 canonical thread、native history、native tool trace 和 native compaction。

因此需要分清两件事：

- Codex 普通 turn 可以调用 ContextEngine。实现上已经调用了 `bootstrap`、`assemble`、`afterTurn`/`ingestBatch`、`maintain`，但调用结果会被投影成 Codex app-server 能接受的 prompt/developer instructions。
- Codex 上下文压缩不能用通用 ContextEngine/PI compaction 替代 Codex native compaction。Codex 的真实历史在 app-server thread 中，OpenClaw session file 只是 mirror 和绑定载体；只压缩 OpenClaw mirror 不会压缩 Codex thread。

当前实现的正确形态是：如果非 legacy ContextEngine 声明 `info.ownsCompaction === true`，Codex compaction 可以先调用它的 `compact()`，但随后仍会尝试执行 Codex native `thread/compact/start`，并把 native 结果作为 `codexNativeCompaction` 记录在 details 中。也就是说 ContextEngine 可以参与 Codex 压缩，但不能成为 Codex 压缩的唯一事实源。

## 2. 相关入口

### 2.1 Codex harness 注册

`extensions/codex/harness.ts` 返回一个 `AgentHarness`：

- `id: "codex"`。
- `supports()` 默认支持 provider `codex`。
- `runAttempt()` 懒加载 `extensions/codex/src/app-server/run-attempt.ts`。
- `compact()` 懒加载 `extensions/codex/src/app-server/compact.ts`。
- `reset()` 清除 OpenClaw session file 中的 Codex binding。
- `dispose()` 清理 shared Codex app-server client。

这个 Harness 被 core 的 `src/agents/harness/selection.ts` 选中。`runAgentHarnessAttempt()` 先根据 provider/model/runtime policy 选择 harness，再通过 v2 lifecycle 包装执行。OpenAI 官方 provider 在默认策略下也可能被路由到 `codex` harness。

### 2.2 ContextEngine 契约

`src/context-engine/types.ts` 定义了 ContextEngine 的主要方法：

- `bootstrap()`：初始化 engine state，必要时导入历史。
- `assemble()`：按 token budget 装配要发给模型的消息。
- `afterTurn()` / `ingestBatch()` / `ingest()`：turn 完成后持久化新上下文。
- `maintain()`：做 transcript rewrite 或后台维护。
- `compact()`：压缩上下文。
- `info.ownsCompaction`：声明 engine 是否拥有自己的 compaction 生命周期。

`src/agents/harness/context-engine-lifecycle.ts` 把这套契约封装成 Harness 可复用 helper：

- `bootstrapHarnessContextEngine()`
- `assembleHarnessContextEngine()`
- `finalizeHarnessContextEngineTurn()`
- `runHarnessContextEngineMaintenance()`
- `isActiveHarnessContextEngine()`

其中 `isActiveHarnessContextEngine()` 明确排除 `info.id === "legacy"`。legacy engine 是旧 PI 行为的包装，不应影响插件 harness 的普通上下文投影。

## 3. 普通 Codex turn 如何调用 ContextEngine

普通 Codex turn 的主体在 `extensions/codex/src/app-server/run-attempt.ts`。

### 3.1 读取 Codex mirror history

Codex app-server thread 是 canonical。OpenClaw 需要一个可见 transcript 供 UI、session 管理、ContextEngine、hook 和 channel 逻辑使用。

`runCodexAppServerAttempt()` 先读取 OpenClaw session file：

```ts
let historyMessages = (await readMirroredSessionHistoryMessages(params.sessionFile)) ?? [];
```

读取逻辑在 `extensions/codex/src/app-server/session-history.ts`：

- 用 `@mariozechner/pi-coding-agent` 的 `parseSessionEntries()` 解析 JSONL。
- 校验第一条是 session header。
- 调用 `migrateSessionEntries()`。
- 用 `buildSessionContext()` 生成 `AgentMessage[]`。

这说明 ContextEngine 看到的是 OpenClaw mirror transcript，不是 Codex app-server 内部 thread store 的完整原生结构。

### 3.2 Bootstrap 阶段

如果 `params.contextEngine` 是 active 非 legacy engine，Codex 会调用：

```ts
await bootstrapHarnessContextEngine({
  contextEngine: activeContextEngine,
  sessionId: params.sessionId,
  sessionKey: sandboxSessionKey,
  sessionFile: params.sessionFile,
  runtimeContext: buildHarnessContextEngineRuntimeContext(...),
  runMaintenance: runHarnessContextEngineMaintenance,
});
```

行为：

1. 只有已有 session file 且 engine 有 `bootstrap` 或 `maintain` 时才运行。
2. 先调用 `contextEngine.bootstrap()`。
3. 再调用 maintenance，reason 为 `bootstrap`。
4. 如果 bootstrap 改写了 session transcript，Codex 会重新读取 mirrored history。

相关测试在 `extensions/codex/src/app-server/run-attempt.context-engine.test.ts`，覆盖了 bootstrap mutation 后重新 assemble 的场景。

### 3.3 Assemble 阶段

Codex 不把 ContextEngine 返回的 `AgentMessage[]` 直接当作 Codex thread history 注入。原因是 Codex app-server 拥有自己的 thread/resume 机制，OpenClaw 不能伪造或重写它的 native thread records。

实际流程：

```text
historyMessages
  -> assembleHarnessContextEngine()
  -> contextEngine.assemble()
  -> projectContextEngineAssemblyForCodex()
  -> promptText + developerInstructionAddition
  -> thread/start or thread/resume developerInstructions
  -> turn/start input
```

`extensions/codex/src/app-server/context-engine-projection.ts` 做投影：

- `assembled.systemPromptAddition` 变成 Codex `developerInstructions` 的追加段。
- `assembled.messages` 被渲染进当前 user prompt 前面的 `<conversation_context>` 块。
- 当前用户 prompt 如果已经在 assembled messages 末尾，会被去重。
- 非文本内容只保留占位，例如 `[image omitted]`、`tool call ... [input omitted]`、`tool result ... [content omitted]`。
- 渲染总量限制为 `MAX_RENDERED_CONTEXT_CHARS = 24000`，单 text part 限制为 `MAX_TEXT_PART_CHARS = 6000`。

这是一种安全投影，而不是 transcript 替换：

- Codex app-server thread 的历史仍由 `thread/start` / `thread/resume` 管理。
- ContextEngine 装配出的历史只是“quoted reference data”。
- 投影明确加了安全提示：把上下文当参考数据，不当新指令。

### 3.4 Turn 完成后的 mirror 与 afterTurn

Codex turn 完成后，`run-attempt.ts` 先调用 `mirrorTranscriptBestEffort()`，内部使用 `extensions/codex/src/app-server/transcript-mirror.ts` 写 OpenClaw session transcript。

mirror 的特点：

- 只写 `user` 和 `assistant` 消息。
- 使用 `idempotencyKey` 去重。
- 走 `runAgentHarnessBeforeMessageWriteHook()`。
- 写完后 emit session transcript update。

随后如果 active ContextEngine 存在，会重新读取 mirror history，并调用：

```ts
await finalizeHarnessContextEngineTurn({
  contextEngine: activeContextEngine,
  messagesSnapshot: finalMessages,
  prePromptMessageCount,
  runtimeContext: buildHarnessContextEngineRuntimeContextFromUsage(...),
});
```

`finalizeHarnessContextEngineTurn()` 的策略：

1. 如果 engine 有 `afterTurn()`，用完整 conversation snapshot 调用它。
2. 否则把新增消息交给 `ingestBatch()` 或逐条 `ingest()`。
3. 只有非 prompt error、非 abort、非 yield abort 且 finalization 成功时，才做 reason 为 `turn` 的 maintenance。

所以普通 Codex turn 中，ContextEngine 负责 OpenClaw mirror 维度的上下文生命周期；Codex native thread 继续由 app-server 自己维护。

## 4. Codex 压缩的真实执行路径

OpenClaw compaction 请求会经过 core 的 harness selection。`src/agents/harness/selection.ts` 的 `maybeCompactAgentHarnessSession()` 会选择当前 session pinned 的 harness，并调用 harness 的 `compact()`。

对于 Codex harness，`extensions/codex/harness.ts` 调到：

```ts
maybeCompactCodexAppServerSession(params, { pluginConfig })
```

实现位于 `extensions/codex/src/app-server/compact.ts`。

### 4.1 默认：Codex native thread compaction

没有 active owning ContextEngine 时，Codex 只走 `compactCodexNativeThread()`：

1. 读取 OpenClaw session file 中的 Codex app-server binding。
2. 校验 `threadId` 存在。
3. 校验 auth profile 是否匹配。
4. 启动 Codex app-server client。
5. 发送 JSON-RPC request：

```ts
client.request("thread/compact/start", { threadId: binding.threadId })
```

6. 等待 Codex notification：

- `thread/compacted`
- 或 `item/completed` 且对应 compaction item

7. 返回 `backend: "codex-app-server"`、`threadId`、`turnId`、`itemId` 等 details。

Codex app-server 还会在 normal turn 期间发出 `contextCompaction` item。`extensions/codex/src/app-server/event-projector.ts` 会把它映射到 OpenClaw compaction hook 和 agent event：

- `runAgentHarnessBeforeCompactionHook()`
- `runAgentHarnessAfterCompactionHook()`
- `stream: "compaction"`，`backend: "codex-app-server"`

这说明 Codex native compaction 是 app-server thread 级能力，不只是 OpenClaw transcript summary。

### 4.2 Owning ContextEngine：参与但不替代 native compaction

`compact.ts` 有一个特殊分支：

```ts
if (activeContextEngine?.info.ownsCompaction) {
  primary = await activeContextEngine.compact(...)
  if (primary?.ok && primary.compacted) {
    await runHarnessContextEngineMaintenance(...)
  }
  const nativeResult = await compactCodexNativeThread(params, options)
  return merge(primary, nativeResult)
}
```

注意这里不是“ContextEngine 成功就跳过 Codex native compaction”，而是：

- ContextEngine compaction 是 primary result。
- Codex native compaction 仍然执行。
- native status 被合并进 `details.codexNativeCompaction`。
- 如果 ContextEngine 失败，仍尝试 native compaction，并把错误放进 `details.contextEngineCompaction`。
- 如果 native compaction 失败但 ContextEngine 成功，整体仍可返回 ContextEngine 成功，同时记录 native 失败原因。

这正是 Codex 场景里的关键边界：ContextEngine 可以维护自己的索引/摘要/外部 store，但 Codex app-server thread 仍需要自己的 native compaction。

## 5. 为什么 Codex 上下文压缩不能只调用 ContextEngine

### 5.1 Codex 的 canonical history 不在 OpenClaw session file

Codex app-server 的 thread 是真实模型上下文来源。OpenClaw session file 是 mirror：

- 用于 UI、OpenClaw session 列表、ContextEngine、hooks、delivery、搜索和切换。
- 只镜像用户/助手可见消息。
- 不完整表示 Codex native tool trace、reasoning、thread metadata、compaction item、provider-side state。

如果只调用 ContextEngine/PI compaction，压缩的是 OpenClaw mirror 或 ContextEngine store；Codex 下一次 `thread/resume` 仍会加载未压缩的 app-server thread。

结果：OpenClaw 以为上下文变小了，Codex 实际 prompt 仍然超预算或继续携带旧历史。

### 5.2 Legacy ContextEngine 实际代理的是 PI runtime compaction

`src/context-engine/legacy.ts` 的 `compact()` 调用 `delegateCompactionToRuntime()`。

`src/context-engine/delegate.ts` 再调用 `compactEmbeddedPiSessionDirect()`，这是内置 PI compaction path。它会构造 PI compaction LLM session、读取 OpenClaw session transcript、生成摘要/修剪 artifacts。

这个路径对 Codex 有两个问题：

1. 它不知道 Codex app-server thread 的内部历史。
2. 它不会发送 `thread/compact/start` 给 Codex app-server。

所以在 Codex harness compaction 中，把 legacy ContextEngine 当成压缩实现会产生错觉：OpenClaw mirror 被摘要了，但 Codex native thread 没被摘要。

当前 `isActiveHarnessContextEngine()` 排除 legacy，就是为了避免插件 harness 把 legacy PI 行为误当成通用可替换上下文引擎。

### 5.3 Codex thread resume 依赖 app-server 维护的工具和线程状态

`extensions/codex/src/app-server/thread-lifecycle.ts` 的 `startOrResumeThread()` 会读取 Codex binding，尝试 `thread/resume`，并传入：

- `threadId`
- model/provider/auth profile
- dynamic tools fingerprint
- developer instructions
- `persistExtendedHistory: true`

Codex app-server 需要维护自己的 native tool/event/history 结构。OpenClaw mirror 不能安全重写这些结构，也不能用 ContextEngine assemble 结果替代 native thread records。

如果用 ContextEngine 压缩结果直接重写 OpenClaw session file，再让 Codex resume 原 thread，会出现两个历史源分叉：

- OpenClaw mirror：已经被摘要或删除旧 entry。
- Codex thread：仍然有原始旧 entry 和 native trace。

后续 `afterTurn()` 又会从 Codex 输出重新 mirror，ContextEngine 看到的历史可能和 Codex 实际使用的历史长期不一致。

### 5.4 Compaction hook 和事件语义会错位

Codex native compaction 会通过 app-server events 暴露 `contextCompaction` item。OpenClaw 在 `event-projector.ts` 中把它映射到 `before_compaction` / `after_compaction` hook 和 `compaction` stream。

如果只调用 ContextEngine：

- Codex app-server 不会产生 native compaction item。
- Codex thread 内不会有对应 compaction event。
- OpenClaw hook 看到的压缩和 Codex 实际 thread 压缩不是同一件事。

这会破坏调试、trajectory、UI 进度、plugin hook 和后续诊断。

### 5.5 可能触发递归或错误 runtime 边界

ContextEngine 的 `compact()` 是通用契约。第三方 engine 可以选择自己实现，也可以调用 `delegateCompactionToRuntime()` 复用 OpenClaw built-in compaction。

在 Codex harness 中如果无条件调用 ContextEngine compact，会把“插件 harness 压缩”重新带回 “PI embedded runner compaction” 边界。即使不形成直接递归，也会产生错误 runtime：

- 压缩模型、工具、settings manager、PI session manager 都来自 PI compaction path。
- `compact.ts` 中有注释说明 PI compaction LLM session “不是 user-facing agent session，没有关联 context engine”，因此故意不传 `contextEngineInfo` 给 PI auto-compaction guard。
- Codex app-server 的 binding、thread state、native compaction timeout、auth profile mismatch 等 Codex 专属约束都不会被 PI path 正确处理。

换句话说，ContextEngine compact 是“上下文系统的公共接口”，不是“任意 harness 的 native 压缩替代实现”。

## 6. 如果 Codex 压缩只调用 ContextEngine，会出现什么问题

### 6.1 压缩无效

最直接的问题是 token pressure 不会在 Codex 真实 thread 中下降。用户触发 `/compact` 后，OpenClaw 可能返回成功，但下一次 Codex `turn/start` 仍基于未压缩 app-server thread。

表现可能是：

- 继续 overflow。
- Codex app-server 自动再次触发 native compaction。
- `/status` 或 OpenClaw mirror 估算显示变小，但 Codex provider 实际请求仍很大。

### 6.2 历史分叉

ContextEngine 或 PI compaction 改写的是 OpenClaw session transcript；Codex native thread 没同步改写。

后续 turn 会形成双重事实源：

- OpenClaw mirror 中是摘要后的旧历史。
- Codex thread 中是完整旧历史。
- 新 assistant 输出再 mirror 回 OpenClaw 时，ContextEngine 只能看到投影后的外部结果，无法知道 Codex 在 native thread 中实际用了哪些旧工具结果和 native messages。

这种分叉会让调试“模型为什么这样回答”变得困难，也会影响 ContextEngine 的检索、citation、maintenance 和 transcript rewrite 决策。

### 6.3 可能丢失 Codex native tool 语义

Codex native shell/edit/apply_patch/update_plan 等工具不是通过 OpenClaw dynamic tool bridge 执行的。它们的完整 event 和 thread items 由 app-server 管理。

OpenClaw mirror 只保留足够产品使用的可见消息，不是完整 native trace。用 ContextEngine 结果替代 native compaction，可能会让摘要缺失：

- 原生工具调用顺序。
- 原生工具参数和结果。
- app-server 对 contextCompaction item 的内部引用。
- Codex 自己用于 resume/review/steer 的结构化 thread state。

### 6.4 Hook、trajectory、UI 事件不可信

如果 OpenClaw 报告 ContextEngine compaction 成功，但 Codex app-server 没有完成 native compaction：

- `compaction` stream 不能证明 Codex thread 已压缩。
- `before_compaction` / `after_compaction` hook 和 Codex native event 不再对应。
- trajectory 中记录的 prompt/mirror 和 Codex 实际 thread 可能不一致。
- 运维看到的 details 缺少 `backend: "codex-app-server"`、`threadId`、`turnId`、`itemId` 证据。

### 6.5 Auth/profile/binding 校验被绕过

Codex native compaction 会校验：

- session file 里是否有 `threadId` binding。
- 请求 auth profile 是否和 binding 匹配。
- app-server 是否能启动并访问对应账号。
- `thread/compact/start` 是否实际完成。

通用 ContextEngine/PI compaction 不具备这些 Codex app-server 约束。只调用 ContextEngine 会让错误会话、旧 binding、auth profile mismatch 等问题在 OpenClaw 层被掩盖。

## 7. 当前实现的安全边界

当前 Codex 实现采用三层边界：

### 7.1 Turn 上下文：ContextEngine 只做投影

普通 turn 中，ContextEngine assemble 的结果被放入当前 prompt 的 `<conversation_context>`，并且显式标记为 quoted reference data。它不重写 Codex native thread。

### 7.2 Turn 后维护：ContextEngine 只基于 mirror

Codex 完成后先 mirror transcript，再调用 ContextEngine `afterTurn()`/`ingestBatch()` 和 `maintain()`。这保证 ContextEngine 看到的是 OpenClaw 产品层 transcript，而不是半完成的 app-server event stream。

### 7.3 压缩：Codex native 必须参与

`extensions/codex/src/app-server/compact.ts` 的核心策略是：

- 非 owning ContextEngine：只跑 Codex native compaction。
- owning ContextEngine：先跑 ContextEngine compact，再跑 Codex native compaction，并合并结果。
- ContextEngine 失败：仍尝试 Codex native compaction，并报告 primary failure。
- Codex native 失败：不伪装为 native 成功，写入 details。

这个策略承认两个事实源：

- ContextEngine 可能拥有外部 store / lossless memory / summary lifecycle。
- Codex app-server 必须压缩自己的 native thread。

## 8. 代码级调用图

### 8.1 普通 turn

```text
src/agents/harness/selection.ts
  -> runAgentHarnessAttempt()
  -> selectAgentHarnessDecision()
  -> Codex AgentHarness.runAttempt()

extensions/codex/harness.ts
  -> runCodexAppServerAttempt()

extensions/codex/src/app-server/run-attempt.ts
  -> readMirroredSessionHistoryMessages()
  -> isActiveHarnessContextEngine()
  -> bootstrapHarnessContextEngine()
  -> assembleHarnessContextEngine()
  -> projectContextEngineAssemblyForCodex()
  -> resolveAgentHarnessBeforePromptBuildResult()
  -> startOrResumeThread()
      -> thread/start or thread/resume
  -> turn/start
  -> mirrorCodexAppServerTranscript()
  -> finalizeHarnessContextEngineTurn()
      -> contextEngine.afterTurn()
      -> or contextEngine.ingestBatch()/ingest()
      -> runHarnessContextEngineMaintenance(reason: "turn")
```

### 8.2 手动或预算触发压缩

```text
core compaction command/path
  -> maybeCompactAgentHarnessSession()
  -> selectAgentHarness()
  -> Codex AgentHarness.compact()

extensions/codex/harness.ts
  -> maybeCompactCodexAppServerSession()

extensions/codex/src/app-server/compact.ts
  -> if activeContextEngine?.info.ownsCompaction
       -> contextEngine.compact()
       -> runHarnessContextEngineMaintenance(reason: "compaction")
       -> compactCodexNativeThread()
       -> merge ContextEngine result + codexNativeCompaction status
     else
       -> compactCodexNativeThread()

compactCodexNativeThread()
  -> readCodexAppServerBinding()
  -> client.request("thread/compact/start", { threadId })
  -> wait for thread/compacted or item/completed
```

## 9. 设计判断

Codex harness 的 ContextEngine 集成不是“把 Codex 接入 OpenClaw 后统一改用 OpenClaw 上下文系统”，而是“双层上下文”：

- OpenClaw 层：session transcript mirror、ContextEngine、memory、hooks、UI、delivery、subagent/session 管理。
- Codex 层：app-server thread、native tools、native compaction、resume/steer/review、provider-side execution trace。

ContextEngine 可以优化 OpenClaw 层上下文，也可以通过 `ownsCompaction` 管理自己的存储和摘要。但它不能单独压缩 Codex 层，因为它没有 Codex app-server thread 的完整所有权和协议能力。

因此，Codex 中“上下文压缩不能调用 ContextEngine”的准确表述应是：

> Codex 压缩不能只调用通用 ContextEngine，尤其不能调用 legacy/PI delegate compaction 来替代 Codex native compaction；只有声明 `ownsCompaction` 的非 legacy ContextEngine 可以作为 primary/sidecar 参与，且仍必须同步执行或至少记录 Codex native compaction 结果。

这也是当前 `compact.ts` 的实现意图。

## 10. 相关源码索引

- `extensions/codex/harness.ts`：Codex AgentHarness 注册。
- `extensions/codex/src/app-server/run-attempt.ts`：普通 Codex turn、ContextEngine bootstrap/assemble/finalize。
- `extensions/codex/src/app-server/context-engine-projection.ts`：ContextEngine assembled messages 到 Codex prompt 的投影。
- `extensions/codex/src/app-server/compact.ts`：Codex native compaction 与 owning ContextEngine compaction 合并策略。
- `extensions/codex/src/app-server/session-history.ts`：从 OpenClaw session file 读取 Codex mirror history。
- `extensions/codex/src/app-server/transcript-mirror.ts`：Codex 输出写回 OpenClaw transcript。
- `extensions/codex/src/app-server/event-projector.ts`：Codex native compaction item 到 OpenClaw hook/event 的投影。
- `extensions/codex/src/app-server/thread-lifecycle.ts`：Codex thread start/resume/turn/start 参数。
- `src/agents/harness/selection.ts`：Harness 选择和 `maybeCompactAgentHarnessSession()`。
- `src/agents/harness/context-engine-lifecycle.ts`：Harness ContextEngine lifecycle helper。
- `src/context-engine/types.ts`：ContextEngine 契约。
- `src/context-engine/legacy.ts`：legacy ContextEngine 对 PI compaction 的兼容包装。
- `src/context-engine/delegate.ts`：`delegateCompactionToRuntime()` 到 PI compaction runtime 的桥接。
- `src/agents/pi-embedded-runner/compact.ts`：PI embedded runner compaction 实现。
