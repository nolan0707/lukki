# Codex 插件调用 OpenClaw 工具的交互流程

## 1. 总览

Codex 插件调用 OpenClaw 中注册工具的机制，核心是 Codex app-server 的 `dynamicTools` 能力。OpenClaw 在每次 Codex harness turn 启动前，把当前 agent run 可用的 OpenClaw 工具物化为一组 `AnyAgentTool`，再转换成 Codex app-server 可理解的 dynamic tool spec。Codex 模型在执行过程中如果选择调用这些工具，app-server 会通过 JSON-RPC `item/tool/call` request 回调 OpenClaw，OpenClaw 再按工具名找到对应 `AnyAgentTool` 并执行。

这条链路的关键源码：

- `extensions/codex/src/app-server/run-attempt.ts`：构建工具、传给 thread/start、处理 `item/tool/call`。
- `extensions/codex/src/app-server/dynamic-tools.ts`：把 `AnyAgentTool` 转为 Codex dynamic tool spec，并执行工具调用。
- `extensions/codex/src/app-server/dynamic-tool-profile.ts`：Codex 专用工具过滤策略。
- `extensions/codex/src/app-server/thread-lifecycle.ts`：把 `dynamicTools` 放进 `thread/start`，并计算 fingerprint。
- `src/agents/pi-tools.ts`：`createOpenClawCodingTools()`，OpenClaw agent 工具总装配入口。
- `src/agents/runtime-plan/tools.ts`：按 runtime plan 或 provider 规则规范化工具 schema。

## 2. 交互链路图

```text
OpenClaw auto reply / embedded run
  -> Codex harness runAttempt()
  -> runCodexAppServerAttempt()
  -> buildDynamicTools()
      -> createOpenClawCodingTools()
      -> applyCodexDynamicToolProfile()
      -> filterToolsForVisionInputs()
      -> filterCodexDynamicToolsForAllowlist()
      -> normalizeAgentRuntimeTools()
  -> createCodexDynamicToolBridge(tools)
      -> bridge.specs = [{ name, description, inputSchema }]
      -> bridge.handleToolCall()
  -> startOrResumeThread(dynamicTools: bridge.specs)
      -> thread/start sends dynamicTools to Codex app-server
  -> Codex model decides to call a dynamic tool
  -> Codex app-server sends JSON-RPC request item/tool/call
  -> run-attempt request handler validates threadId/turnId
  -> handleDynamicToolCallWithTimeout()
  -> toolBridge.handleToolCall()
      -> tool.prepareArguments?()
      -> tool.execute(callId, args, signal)
      -> tool result middleware / extensions / telemetry
      -> convert OpenClaw result content to Codex contentItems
  -> JSON-RPC response returned to Codex app-server
  -> Codex continues turn with tool result
  -> EventProjector includes tool telemetry in EmbeddedRunAttemptResult
```

## 3. 工具构建阶段

### 3.1 入口：`buildDynamicTools()`

`runCodexAppServerAttempt()` 早期会调用 `buildDynamicTools()`：

```ts
const tools = await buildDynamicTools({
  params,
  resolvedWorkspace,
  effectiveWorkspace,
  sandboxSessionKey,
  sandbox,
  runAbortController,
  sessionAgentId,
  pluginConfig,
  onYieldDetected: () => {
    yieldDetected = true;
  },
});
```

如果 `params.disableTools` 为 true，或当前模型不支持 tools，`buildDynamicTools()` 直接返回空数组。这意味着 Codex thread 会以无 OpenClaw dynamic tools 的方式启动。

### 3.2 复用 OpenClaw 工具总装配入口

Codex 插件不是自己重新实现一套工具 registry。它动态 import SDK facade：

```ts
const createOpenClawCodingTools =
  openClawCodingToolsFactoryForTests ??
  (await import("openclaw/plugin-sdk/agent-harness")).createOpenClawCodingTools;
```

然后调用 `createOpenClawCodingTools()` 生成当前 run 的工具集合。传入的上下文非常完整，包括：

- agent/session/run：`agentId`、`sessionId`、`runId`、`sessionKey`、`runSessionKey`。
- workspace/sandbox：`workspaceDir`、`spawnWorkspaceDir`、`sandbox`、exec override。
- message routing：`messageProvider`、`agentAccountId`、`messageTo`、`messageThreadId`、group 信息、Slack thread 信息、reply-to mode。
- requester：`senderId`、`senderName`、`senderUsername`、`senderE164`、`senderIsOwner`。
- model：`modelProvider`、`modelId`、`modelApi`、`modelCompat`、context window、auth mode、vision capability。
- policy：`toolsAllow` 映射为 `runtimeToolAllowlist`，`disableMessageTool`、`forceMessageTool`、heartbeat tool 开关、subagent binding。
- abort：`abortSignal`。
- `onYield`：`sessions_yield` 工具调用后会标记 yield，并 abort 当前 Codex run。

因此 Codex 插件拿到的是“这次 OpenClaw agent run 本来就应该拥有的工具集合”，包括 core 工具、channel 工具、plugin tools、message tool、session/subagent 工具、media/browser/gateway/cron/heartbeat 等，而不是 Codex 插件私有工具。

### 3.3 `createOpenClawCodingTools()` 内部装配

`src/agents/pi-tools.ts` 中的 `createOpenClawCodingTools()` 会组合多类工具：

- 基础 coding tools：read/write/edit 等。
- shell tools：`apply_patch`、`exec`、`process`。
- channel-defined agent tools。
- OpenClaw tools：message、sessions、subagents、cron、browser、nodes、gateway、media、web、heartbeat 等。
- plugin tools：通过 OpenClaw plugin tool 机制 materialize。

随后它会经过共享 policy pipeline：

- message provider 过滤。
- model provider policy。
- owner-only tool policy。
- profile/provider/global/agent/group/sandbox/subagent tool policy。
- tool schema normalization。
- before tool call hook wrapper。
- abort signal wrapper。
- deferred follow-up description 处理。

这一步是 Codex 能复用 OpenClaw 工具安全模型的关键：Codex 插件拿到工具前，OpenClaw 已经按当前 agent、sender、channel、sandbox、profile、provider 等维度应用过权限和 schema 规则。

## 4. Codex 专用工具过滤

### 4.1 native-first 默认策略

`extensions/codex/src/app-server/dynamic-tool-profile.ts` 定义 Codex 专用 profile：

```ts
export const CODEX_NATIVE_FIRST_DYNAMIC_TOOL_EXCLUDES = [
  "read",
  "write",
  "edit",
  "apply_patch",
  "exec",
  "process",
  "update_plan",
] as const;
```

默认 `codexDynamicToolsProfile` 是 `native-first`，所以这些工具会从 OpenClaw dynamic tools 中排除。原因是 Codex app-server 自身已有原生文件读写、shell、patch、plan 等工具。默认让 Codex 使用 native 工具处理本地代码工作，OpenClaw dynamic tools 主要保留 OpenClaw 集成能力。

如果配置 `codexDynamicToolsProfile: "openclaw-compat"`，则不会应用这组默认排除，只保留用户显式 `codexDynamicToolsExclude` 过滤。

### 4.2 名称归一和额外排除

过滤前会归一工具名：

- `bash` 归一为 `exec`。
- `apply-patch` 归一为 `apply_patch`。
- 其他名称 trim + lowercase。

`codexDynamicToolsExclude` 可以继续排除工具。例如排除 `message` 可以阻止 Codex 通过 OpenClaw message tool 发送可见回复；排除 `cron` 可以阻止 Codex 创建计划任务。

### 4.3 视觉输入过滤

`buildDynamicTools()` 会检查：

- 当前模型是否有 vision capability。
- 本次 inbound 是否包含 images。

然后通过 `filterToolsForVisionInputs()` 过滤和图片输入相关的工具，避免无视觉能力模型看到不适用工具。

### 4.4 `toolsAllow` allowlist

如果 embedded run params 带 `toolsAllow`，Codex 插件会在 profile 和 vision 过滤后再执行：

```ts
filterCodexDynamicToolsForAllowlist(visionFilteredTools, params.toolsAllow)
```

这个 allowlist 同样使用 Codex 工具名归一逻辑。它是 runtime 层的显式收窄机制。

### 4.5 runtime/provider schema normalization

最后调用：

```ts
normalizeAgentRuntimeTools({
  runtimePlan: params.runtimePlan,
  tools: filteredTools,
  provider: params.provider,
  config: params.config,
  workspaceDir: input.effectiveWorkspace,
  env: process.env,
  modelId: params.modelId,
  modelApi: params.model.api,
  model: params.model,
});
```

如果已有 `AgentRuntimePlan`，用 runtime plan 归一化；否则走 provider tool schema normalization。这样交给 Codex app-server 的 schema 已经是当前 provider/runtime 可接受的形态。

## 5. 转换为 Codex dynamicTools

### 5.1 创建 bridge

`runCodexAppServerAttempt()` 构建完工具后创建 bridge：

```ts
const toolBridge = createCodexDynamicToolBridge({
  tools,
  signal: runAbortController.signal,
  hookContext: {
    agentId: sessionAgentId,
    config: params.config,
    sessionId: params.sessionId,
    sessionKey: sandboxSessionKey,
    runId: params.runId,
  },
});
```

`createCodexDynamicToolBridge()` 会：

- 确保每个工具都包了 before-tool-call hook。若工具尚未 wrapped，则调用 `wrapToolWithBeforeToolCallHook()`。
- 建立 `toolMap: Map<tool.name, tool>`。
- 初始化 telemetry。
- 创建 tool result middleware runner。
- 创建 legacy Codex app-server tool result extension runner。

### 5.2 dynamic tool spec 结构

bridge 暴露给 Codex app-server 的 spec 是：

```ts
{
  name: tool.name,
  description: tool.description,
  inputSchema: toJsonValue(tool.parameters),
}
```

这组 spec 保存在 `toolBridge.specs`。OpenClaw 不把 `execute` 函数发给 Codex；Codex app-server 只拿到名称、描述、输入 JSON schema。真正执行仍由 OpenClaw 的 request handler 在本进程内完成。

### 5.3 传给 Codex app-server

`startOrResumeThread()` 收到 `dynamicTools: toolBridge.specs`。

新建 thread 时，`buildThreadStartParams()` 把它写进 `thread/start`：

```ts
{
  model,
  cwd,
  approvalPolicy,
  approvalsReviewer,
  sandbox,
  serviceName: "OpenClaw",
  developerInstructions,
  dynamicTools,
  experimentalRawEvents: true,
  persistExtendedHistory: true,
}
```

resume thread 时不重新发送 `dynamicTools`，所以 thread lifecycle 会用 dynamic tools fingerprint 判断旧 binding 是否兼容。

## 6. dynamicTools fingerprint 与 thread 复用

Codex app-server 的 dynamic tools 是 thread 级输入。OpenClaw 因此在 `thread-lifecycle.ts` 中计算 fingerprint：

- fingerprint 基于 tool spec。
- 排序稳定。
- 忽略 `description` 字段。
- 对 JSON value 做稳定化。

binding 文件中记录 `dynamicToolsFingerprint`。后续启动时：

- 如果 binding 存在且 fingerprint 兼容，优先 `thread/resume`。
- 如果 fingerprint 不兼容，通常清除 binding 并新建 thread。
- 如果之前有 dynamic tools、这次变成空工具，OpenClaw 会保留旧 binding，但启动一个 transient no-tool thread，避免破坏原绑定。

这保证 Codex thread 内看到的 dynamic tool schema 和 OpenClaw 当前能处理的工具调用一致，避免旧 thread 继续调用已经不存在或 schema 已变化的工具。

## 7. Codex 发起工具调用

### 7.1 JSON-RPC request

Codex app-server 在模型决定调用 dynamic tool 时，会向 OpenClaw client 发 request：

```text
method: "item/tool/call"
params: {
  threadId,
  turnId,
  callId,
  tool,
  arguments
}
```

`extensions/codex/src/app-server/client.ts` 的 `CodexAppServerClient` 负责解析 app-server stdout 中的 JSON-RPC message。如果 message 有 `id` 和 `method`，它会调用已注册的 server request handlers，并把 handler 返回值作为 JSON-RPC response 写回 app-server stdin。

### 7.2 request handler 校验

`run-attempt.ts` 在 turn 启动后注册 request handler。对 `item/tool/call` 的处理逻辑：

1. 先递增 active request 计数并清掉 turn completion idle timer。
2. 如果 `turnId` 尚未建立，返回 undefined。
3. 解析 request params：`readDynamicToolCallParams(request.params)`。
4. 校验 `call.threadId === thread.threadId` 且 `call.turnId === turnId`。
5. 记录 trajectory `tool.call`。
6. 根据 `toolProgressDetail` 推断 tool meta 和 sanitized args。
7. 发出 OpenClaw agent event：`stream: "tool"`、`phase: "start"`。
8. 调用 `handleDynamicToolCallWithTimeout()`。
9. 记录 trajectory `tool.result`。
10. 发出 `stream: "tool"`、`phase: "result"`，包含 sanitized result。
11. 将 `CodexDynamicToolCallResponse` 作为 JSON-RPC response 返回给 app-server。

如果 request method 不是 `item/tool/call`，但属于审批 request，则走 approval bridge；如果是 elicitation 或 user input，也走对应 bridge。未知 request 返回 undefined，最终 client 默认 fallback 会拒绝审批或给空对象。

## 8. 工具执行阶段

### 8.1 超时与 abort

`handleDynamicToolCallWithTimeout()` 包装一次工具调用：

- 如果 run signal 已 abort，直接返回失败响应。
- 创建单独的 `AbortController`。
- 监听 run abort，abort 工具执行并返回 `"OpenClaw dynamic tool call aborted."`。
- 设置 timeout，最大使用 `CODEX_DYNAMIC_TOOL_TIMEOUT_MS`，当前值为 30s。
- timeout 时：
  - 格式化 timeout details。
  - abort 工具执行。
  - 记录 trajectory `tool.timeout`。
  - 写 warn log。
  - 返回失败 dynamic tool response。
- 工具调用、abort promise、timeout promise 三者 race。

因此 Codex app-server 不会无限等待 OpenClaw 动态工具。超时也会被作为普通失败 tool result 返回给 Codex，让模型可以继续处理。

### 8.2 `toolBridge.handleToolCall()`

真正执行在 `dynamic-tools.ts`：

```ts
const tool = toolMap.get(call.tool);
if (!tool) {
  return {
    contentItems: [{ type: "inputText", text: `Unknown OpenClaw tool: ${call.tool}` }],
    success: false,
  };
}
const args = jsonObjectToRecord(call.arguments);
const preparedArgs = tool.prepareArguments ? tool.prepareArguments(args) : args;
const rawResult = await tool.execute(call.callId, preparedArgs, signal);
```

调用语义：

- tool lookup 使用 Codex request 中的 `tool` 名称精确匹配 `AnyAgentTool.name`。
- arguments 必须是 JSON object；非 object 会变成 `{}`。
- 如果工具实现了 `prepareArguments()`，先做参数准备/校验/补全。
- 执行函数是 OpenClaw 标准 `AnyAgentTool.execute(callId, args, signal)`。
- `signal` 是 run abort signal 和 per-call timeout signal 的组合。

未知工具不会抛出，而是返回 `success: false` 的 Codex tool response。

### 8.3 before/after tool hooks

Codex bridge 会确保工具包裹 before tool call hook：

- `createOpenClawCodingTools()` 通常已经在 `pi-tools.ts` 里调用 `wrapToolWithBeforeToolCallHook()`。
- bridge 再用 `isToolWrappedWithBeforeToolCallHook()` 检查，未包裹才补包。

工具执行后，bridge 异步调用：

```ts
runAgentHarnessAfterToolCallHook({
  toolName,
  toolCallId,
  runId,
  agentId,
  sessionId,
  sessionKey,
  startArgs,
  result 或 error,
  startedAt,
});
```

因此 Codex dynamic tool 调用同样进入 OpenClaw agent harness tool hook 体系。

### 8.4 tool result middleware 与 legacy extension

工具原始结果出来后会经过两层处理：

1. `createAgentToolResultMiddlewareRunner({ runtime: "codex", ... })`
   - 调用当前 OpenClaw runtime 的 tool result middleware。
   - 传入 threadId、turnId、toolCallId、toolName、args、isError、result。

2. `createCodexAppServerToolResultExtensionRunner(...)`
   - 兼容 legacy Codex app-server tool result extension。
   - 同样按 thread/turn/tool 维度处理 result。

最终用于返回给 Codex 的是 extension runner 处理后的 `result`。

### 8.5 错误判定

`isToolResultError()` 不只看异常，也看 tool result details：

- `details.timedOut === true` 视为错误。
- `details.exitCode` 为非 0 视为错误。
- `details.status` 非空且不是 `0`、`ok`、`success`、`completed`、`recorded`、`running` 时视为错误。

如果工具执行抛错，bridge 捕获异常，发 after tool call hook，并返回：

```ts
{
  contentItems: [{ type: "inputText", text: error.message }],
  success: false
}
```

## 9. 结果转换回 Codex

OpenClaw `AgentToolResult` 的 `content` 会被转换成 Codex app-server dynamic tool response：

- OpenClaw text content：

```ts
{ type: "text", text }
```

转换为：

```ts
{ type: "inputText", text }
```

- OpenClaw image content：

```ts
{ type: "image", mimeType, data }
```

转换为：

```ts
{ type: "inputImage", imageUrl: `data:${mimeType};base64,${data}` }
```

最终 response 形态：

```ts
{
  contentItems: [...],
  success: true | false
}
```

这个 response 通过 JSON-RPC 返回给 Codex app-server。Codex app-server 再把工具结果纳入当前 turn，模型继续推理或生成最终回复。

## 10. 工具 telemetry 如何影响 OpenClaw 主流程

`createCodexDynamicToolBridge()` 维护 telemetry：

- `didSendViaMessagingTool`
- `messagingToolSentTexts`
- `messagingToolSentMediaUrls`
- `messagingToolSentTargets`
- `heartbeatToolResponse`
- `toolMediaUrls`
- `toolAudioAsVoice`
- `successfulCronAdds`

`collectToolTelemetry()` 只在成功工具结果上收集：

- 如果工具是 `cron` 且 action 是 `add`，增加 successful cron count。
- 如果工具是 heartbeat response tool，规范化 heartbeat response。
- 如果工具结果包含 media artifact，提取并按 trust 规则过滤 media URL。
- 如果工具是 message tool 的 send action，记录已经发送的文本、媒体和目标。

turn 完成时，`CodexAppServerEventProjector.buildResult()` 会把这些 telemetry 放进 `EmbeddedRunAttemptResult`：

- 如果 message tool 已发送可见回复，OpenClaw 可以避免重复自动投递。
- media URL 和 audio-as-voice 会参与后续媒体投递逻辑。
- heartbeat response 会参与 heartbeat wake 的结构化结果处理。
- replay metadata 会把 message tool side effect 标记为非 replay-safe。

这就是 Codex 插件能和 OpenClaw “消息是否已发送”“是否有媒体结果”“heartbeat 是否已响应”等主流程状态保持一致的原因。

## 11. 与可见回复的关系

Codex harness 的 delivery default 是：

```ts
deliveryDefaults.sourceVisibleReplies = "message_tool"
```

同时 `buildDynamicTools()` 会按 run params 设置：

- `disableMessageTool`
- `forceMessageTool`
- `forceMessageTool: params.sourceReplyDeliveryMode === "message_tool_only"`
- `requireExplicitMessageTarget`

因此在 Codex runtime 中，面向源聊天的可见回复通常应通过 OpenClaw message tool 发送，而不是只依赖 Codex 最终 assistant text。这样回复可以复用 OpenClaw channel adapter、thread routing、media handling、reply-to 策略和去重逻辑。

如果模型没有调用 message tool，projector 仍会收集最后 assistant text，OpenClaw 可按普通 embedded result 做 fallback/final delivery。

## 12. `sessions_yield` 的特殊行为

`createOpenClawCodingTools()` 接收 `onYield` callback。Codex 插件传入的 callback 会：

- 设置 `yieldDetected = true`。
- 发出 `codex_app_server.tool` event，工具名为 `sessions_yield`。
- 调用 `runAbortController.abort("sessions_yield")`。

这表示 Codex 调用 `sessions_yield` 时，当前 run 会主动让出。最终 result 会带 `yieldDetected`，供 OpenClaw session/subagent 调度逻辑识别。

## 13. 安全边界

Codex 调用 OpenClaw 工具时有几层边界：

- 工具 materialization 前继承 OpenClaw agent/channel/sender/group/sandbox/profile/provider policy。
- 默认 `native-first` 排除文件和 shell 类 OpenClaw 工具，避免和 Codex 原生执行能力重复。
- `toolsAllow` 和 `codexDynamicToolsExclude` 可进一步收窄。
- tool schema 会按 runtime/provider 规范化。
- 每次 tool call 校验 `threadId` 和 `turnId`。
- 每次 tool call 使用 run abort signal 和 30s timeout。
- 未知工具返回失败，不会穿透执行。
- app-server 未处理的审批请求默认 decline。
- message tool side effect 会进入 telemetry，并影响 replay safety。

需要注意的是，Codex 原生工具和 OpenClaw dynamic tools 是两套不同执行面。`appServer.mode: yolo` 会影响 Codex 原生审批/sandbox，OpenClaw dynamic tools 仍按 OpenClaw 自己的工具策略和参数上下文执行。

## 14. 端到端时序

### 14.1 启动前

1. OpenClaw 选择 Codex harness。
2. `runCodexAppServerAttempt()` 解析 workspace、sandbox、agent/session、auth、plugin config。
3. `buildDynamicTools()` 调用 `createOpenClawCodingTools()`。
4. OpenClaw 工具经过通用 policy pipeline、Codex profile、vision/allowlist、runtime schema normalization。
5. `createCodexDynamicToolBridge()` 建立 tool map 和 dynamic tool specs。
6. `startOrResumeThread()` 基于 specs fingerprint 决定 resume 还是 start。
7. `thread/start` 把 `dynamicTools` 发送给 Codex app-server。

### 14.2 调用中

1. Codex app-server 执行 `turn/start`。
2. Codex 模型选择某个 dynamic tool。
3. app-server 发送 JSON-RPC `item/tool/call`。
4. OpenClaw request handler 校验 thread/turn。
5. OpenClaw 发 `tool start` event 和 trajectory。
6. `handleDynamicToolCallWithTimeout()` 包装 timeout/abort。
7. `toolBridge.handleToolCall()` 执行 `AnyAgentTool`。
8. tool result middleware 和 extension runner 处理结果。
9. OpenClaw 收集 telemetry。
10. OpenClaw 发 `tool result` event 和 trajectory。
11. JSON-RPC response 返回 app-server。

### 14.3 完成后

1. Codex app-server 继续 turn，直到 `turn/completed`。
2. projector 生成 `EmbeddedRunAttemptResult`。
3. telemetry 被写入 result。
4. transcript mirror 写入 OpenClaw session。
5. OpenClaw 根据 message tool side effect、assistant text、media、heartbeat 等信息完成最终投递和状态更新。

## 15. 一句话结论

Codex 插件调用 OpenClaw 工具的本质是：OpenClaw 在 Codex thread 启动前把当前 run 的授权工具集编译成 Codex `dynamicTools` schema；Codex app-server 只负责让模型看到并请求这些工具；真正的工具查找、权限后执行、hook、middleware、telemetry、超时和结果转换都留在 OpenClaw 进程内完成。
