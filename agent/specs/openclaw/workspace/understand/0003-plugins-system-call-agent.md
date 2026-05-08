# Codex 插件调度 OpenClaw Agent 机制分析

## 1. 总览

Codex 插件支持调度 OpenClaw 中定义的 agent，并不是通过 Codex 插件直接读取 `agents.list[]` 后自行启动 agent，而是复用 OpenClaw 标准工具系统。Codex 插件在 Codex app-server 的 `dynamicTools` 中暴露 `agents_list`、`sessions_spawn`、`subagents` 等工具；当 Codex 模型调用这些工具时，OpenClaw 在本进程内执行对应 `AnyAgentTool.execute()`，再进入 core 的 subagent 调度流程。

核心调用链：

```text
Codex turn 启动
  -> extensions/codex/src/app-server/run-attempt.ts
  -> buildDynamicTools()
  -> createOpenClawCodingTools()
  -> createOpenClawTools()
  -> 注入 agents_list / sessions_spawn / subagents
  -> dynamicTools 传给 Codex app-server thread/start
  -> Codex 模型调用 sessions_spawn
  -> app-server 发 item/tool/call 给 OpenClaw
  -> Codex dynamic tool bridge 执行 AnyAgentTool.execute()
  -> sessions_spawn 调用 spawnSubagentDirect()
  -> OpenClaw 解析目标 agent、权限、会话、workspace、模型
  -> callGateway({ method: "agent", ... })
  -> 子 agent run 在 subagent lane 启动
  -> 子 run 进入标准 agent command / embedded runner
  -> selectAgentHarness() 按 target agent/provider/model 选择 harness
  -> 如果解析为 codex runtime，则再次进入 extensions/codex/harness.ts
  -> 子 agent 也作为独立 Codex app-server run 执行
```

关键源码：

- `extensions/codex/src/app-server/run-attempt.ts`：Codex turn 构建 dynamic tools，并处理 `item/tool/call`。
- `extensions/codex/src/app-server/dynamic-tools.ts`：把 OpenClaw `AnyAgentTool` 转为 Codex dynamic tool，并桥接执行。
- `src/agents/pi-tools.ts`：`createOpenClawCodingTools()`，OpenClaw agent 工具总装配入口。
- `src/agents/openclaw-tools.ts`：创建 `agents_list`、`sessions_spawn`、`subagents` 等 OpenClaw 工具。
- `src/agents/tools/agents-list-tool.ts`：列出当前 requester 可 target 的 OpenClaw agent。
- `src/agents/tools/sessions-spawn-tool.ts`：`sessions_spawn` 工具入口。
- `src/agents/tools/subagents-tool.ts`：管理已 spawn 的子 agent。
- `src/agents/subagent-spawn.ts`：原生 OpenClaw subagent 调度实现。
- `src/agents/subagent-target-policy.ts`：跨 agent 调度 allowlist 规则。
- `src/agents/command/attempt-execution.ts`：gateway `agent` 命令进入 embedded agent runner 的路径。
- `src/agents/pi-embedded-runner/run.ts`：session lane/global lane 入队，以及 harness selection。
- `src/agents/harness/selection.ts`：根据 runtime policy、provider、model 选择 Codex/Pi 等 agent harness。
- `extensions/codex/harness.ts`：Codex agent harness 注册与 `runAttempt` 入口。
- `src/gateway/server-lanes.ts` / `src/config/agent-limits.ts` / `src/process/command-queue.ts`：subagent 并发 lane 配置与队列执行。
- `docs/tools/subagents.md`：用户文档层的 subagent 行为说明。

## 2. Codex 如何获得 agent 调度工具

### 2.1 dynamic tools 构建入口

Codex harness 的一次 turn 会进入 `runCodexAppServerAttempt()`。该函数在 app-server thread 启动前调用 `buildDynamicTools()`：

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

`buildDynamicTools()` 会通过 SDK facade 动态加载 OpenClaw 工具构建入口：

```ts
const createOpenClawCodingTools =
  openClawCodingToolsFactoryForTests ??
  (await import("openclaw/plugin-sdk/agent-harness")).createOpenClawCodingTools;
```

这意味着 Codex 插件不会绕过 OpenClaw 工具系统，也不会维护一套独立 agent 调度 registry。它拿到的是当前 OpenClaw run 经过标准 policy 处理后的工具集合。

### 2.2 传入工具构建的上下文

Codex 插件调用 `createOpenClawCodingTools()` 时，会传入当前 run 的关键上下文：

- `agentId`：当前 requester agent id。
- `sessionKey` / `runSessionKey` / `sessionId` / `runId`。
- `workspaceDir` / `spawnWorkspaceDir`。
- `sandbox` 和 exec override。
- `messageProvider`、`agentAccountId`、`messageTo`、`messageThreadId`。
- group 信息：`groupId`、`groupChannel`、`groupSpace`。
- requester 信息：`senderId`、`senderName`、`senderUsername`、`senderE164`、`senderIsOwner`。
- `allowGatewaySubagentBinding`。
- model 信息：`modelProvider`、`modelId`、`modelApi`、`modelCompat`、vision capability。
- tool policy 输入：`toolsAllow`、`disableMessageTool`、`forceMessageTool`、heartbeat tool 开关。
- `onYield` callback。

这些上下文决定 `sessions_spawn` 能否出现、能 target 哪些 agent、是否能创建 thread-bound session、子 agent workspace 如何继承、完成结果如何回到 requester。

### 2.3 OpenClaw 工具装配

`src/agents/pi-tools.ts` 的 `createOpenClawCodingTools()` 会组合多类工具，再进入共享工具 policy pipeline。与 agent 调度直接相关的工具由 `src/agents/openclaw-tools.ts` 创建：

- `agents_list`：列出可 target 的 OpenClaw agent ids。
- `sessions_spawn`：启动原生 subagent 或 ACP runtime。
- `sessions_yield`：结束当前 turn，等待子任务结果进入后续消息。
- `subagents`：列出、kill、steer 已 spawn 的子 agent。
- `sessions_list` / `sessions_history` / `sessions_send`：跨 session 查看和通信辅助。

这些工具是否能暴露给 Codex，取决于当前有效工具策略。`src/agents/tool-catalog.ts` 中 `sessions_spawn`、`subagents`、`agents_list` 属于 `sessions` 类工具，通常在 `coding` profile 下可用；`messaging` profile 默认不暴露 `sessions_spawn`。

## 3. Codex 看到的是 dynamic tool schema

`extensions/codex/src/app-server/dynamic-tools.ts` 的 `createCodexDynamicToolBridge()` 会把 OpenClaw 工具转成 Codex app-server dynamic tool spec：

```ts
{
  name: tool.name,
  description: tool.description,
  inputSchema: toJsonValue(tool.parameters),
}
```

Codex app-server 只拿到工具名、描述和 JSON schema。真实 JS 执行函数不传给 app-server，而是留在 OpenClaw 进程内的 `toolMap`：

```ts
const toolMap = new Map(tools.map((tool) => [tool.name, tool]));
```

所以 Codex 模型调用 `sessions_spawn` 时，本质上是请求 OpenClaw 执行名为 `sessions_spawn` 的 `AnyAgentTool`。

## 4. agent 发现：`agents_list`

Codex 如果需要知道可以调度哪些 OpenClaw agent，可以调用 `agents_list`。

实现位于 `src/agents/tools/agents-list-tool.ts`。执行流程：

1. 读取 runtime config。
2. 解析当前 requester session key。
3. 得到 requester agent id。
4. 读取 requester agent 的 `subagents.allowAgents`。
5. 如果 requester agent 没配置，读取 `agents.defaults.subagents.allowAgents`。
6. 读取 `agents.list[]` 中已配置的 agent id 和 name。
7. 调用 `resolveSubagentAllowedTargetIds()` 得到当前 requester 可 target 的 agent ids。
8. 返回每个 agent 的 id、name、是否 configured、有效模型和 runtime metadata。

默认策略在 `src/agents/subagent-target-policy.ts`：

- 如果没有配置 `allowAgents`，默认只能 target requester agent 自己。
- `allowAgents: ["research"]` 允许 target `research`。
- `allowAgents: ["*"]` 允许 target 所有已配置 agent。

这让 Codex 可以通过工具结果了解“当前上下文允许我调度哪些 OpenClaw config agent”。

## 5. agent 调度入口：`sessions_spawn`

### 5.1 工具参数

`sessions_spawn` 由 `src/agents/tools/sessions-spawn-tool.ts` 创建。主要参数：

- `task`：子 agent 任务描述，必填。
- `label`：人类可读标签。
- `runtime`：`"subagent"` 或 `"acp"`，默认 `"subagent"`。
- `agentId`：目标 OpenClaw agent id。
- `model`：本次子 run 模型覆盖。
- `thinking`：本次子 run thinking 覆盖。
- `cwd`：工作目录提示。
- `runTimeoutSeconds` / `timeoutSeconds`：运行超时。
- `thread`：是否请求 channel thread binding。
- `mode`：`"run"` 或 `"session"`。
- `cleanup`：`"delete"` 或 `"keep"`。
- `sandbox`：`"inherit"` 或 `"require"`。
- `context`：`"isolated"` 或 `"fork"`。
- `lightContext`：轻量 bootstrap context。
- `attachments` / `attachAs`：传给子 agent 的附件。

`sessions_spawn` 明确拒绝这些 channel delivery 参数：

- `target`
- `transport`
- `channel`
- `to`
- `threadId`
- `thread_id`
- `replyTo`
- `reply_to`

这是为了避免把“调度 agent”工具误用成“发送消息”工具。需要投递消息时应使用 `message` 或 `sessions_send`。

### 5.2 runtime 分支

`sessions_spawn` 支持两类 runtime：

- `runtime: "subagent"`：调度 OpenClaw config 中定义的 agent，即 `agents.list[]` / defaults 管理的 agent。
- `runtime: "acp"`：调度外部 ACP harness，如 Claude Code、Gemini、OpenCode，或显式 Codex ACP/acpx。

本文关注的是 OpenClaw 中定义的 agent，对应 `runtime: "subagent"`。此分支调用：

```ts
spawnSubagentDirect(...)
```

ACP runtime 走 `spawnAcpDirect()`，不使用 `agents_list` 的 allowlist 语义。

## 6. `spawnSubagentDirect()` 调度流程

`src/agents/subagent-spawn.ts` 的 `spawnSubagentDirect()` 是真正启动 OpenClaw 子 agent 的核心。

### 6.1 参数校验

首先校验显式传入的 `agentId`：

- 必须符合 `[a-z0-9][a-z0-9_-]{0,63}`。
- 非法 id 直接返回 error。
- 这样可以避免错误字符串被 normalize 成 ghost agent id，进而创建错误 workspace 或 session。

然后解析：

- `requestThreadBinding = params.thread === true`
- `sandboxMode = params.sandbox === "require" ? "require" : "inherit"`
- `spawnMode`：显式 `mode` 优先；如果请求 thread binding 且未指定 mode，默认 `"session"`；否则默认 `"run"`。

如果 `mode: "session"` 但没有 `thread: true`，直接失败，因为持久 session 模式要求 channel thread binding。

### 6.2 requester 与 target agent 解析

`spawnSubagentDirect()` 会解析当前 requester：

1. 从 config 读取 main session alias。
2. 将工具上下文中的 `agentSessionKey` 转成内部 session key。
3. 从 requester session key 解析 requester agent id。
4. 如果工具上下文提供 `requesterAgentIdOverride`，优先使用 override。

然后处理 target：

- 如果 `sessions_spawn.agentId` 存在，target agent id 是该值 normalize 后的结果。
- 如果没有传 `agentId`，target agent id 默认是 requester agent id。

如果配置了：

```ts
subagents.requireAgentId = true
```

则必须显式传 `agentId`，否则返回 forbidden，并提示使用 `agents_list` 查看允许的 agent ids。

### 6.3 跨 agent allowlist

target policy 调用 `resolveSubagentTargetPolicy()`：

```ts
const targetPolicy = resolveSubagentTargetPolicy({
  requesterAgentId,
  targetAgentId,
  requestedAgentId,
  allowAgents:
    resolveAgentConfig(cfg, requesterAgentId)?.subagents?.allowAgents ??
    cfg?.agents?.defaults?.subagents?.allowAgents,
});
```

规则：

- 未显式传 `agentId` 且 target 是 requester 自己，允许。
- 显式 target 或跨 agent target 需要命中 allowlist。
- `allowAgents: ["*"]` 允许所有已配置 agent。
- 否则返回 `agentId is not allowed for sessions_spawn`。

这意味着 Codex 不能任意调度 OpenClaw config 里所有 agent，必须经过 requester agent 的 subagent policy。

### 6.4 深度与并发限制

`spawnSubagentDirect()` 会检查：

- 当前 caller depth：`getSubagentDepthFromSessionStore(requesterInternalKey, { cfg })`
- 最大深度：`agents.defaults.subagents.maxSpawnDepth`
- 当前 requester 活跃 child 数：`countActiveRunsForSession(requesterInternalKey)`
- 最大 child 数：`agents.defaults.subagents.maxChildrenPerAgent`

如果超过深度或并发限制，返回 forbidden。

另外，OpenClaw 对子 agent 的工具策略也会根据深度收紧。`src/agents/pi-tools.policy.ts` 中的策略：

- orchestrator subagent 可以继续使用 `sessions_spawn`、`subagents`、`sessions_list`、`sessions_history`。
- leaf subagent 会被 deny `sessions_spawn`、`subagents`、`sessions_list`、`sessions_history`。
- 子 agent 始终 deny 一些高风险或主 agent 协调工具，例如 `gateway`、`agents_list`、`session_status`、`cron`、`sessions_send`。

需要区分两类“并发限制”：

- `maxChildrenPerAgent` 是父 session 维度的 active child 数限制，防止单个父 agent 无限扇出。
- `agents.defaults.subagents.maxConcurrent` 是全局 `subagent` lane 的执行并发上限，默认 8。

前者决定“一个 requester 当前最多能挂多少个子 agent”，后者决定“gateway 进程内最多同时跑多少个 subagent run”。因此 OpenClaw 支持的是有界并行，而不是无限并行。

### 6.5 sandbox 限制

调度前会解析 requester runtime 和 child runtime sandbox 状态：

- sandboxed requester 不能 spawn unsandboxed child。
- `sandbox: "require"` 要求 target child runtime 是 sandboxed。
- 不满足时返回 forbidden。

这保证 Codex 在 sandboxed OpenClaw run 中不能通过子 agent 绕过 sandbox。

### 6.6 子 session key 和目标 agent 配置

通过后会生成 child session key：

```text
agent:<targetAgentId>:subagent:<uuid>
```

然后解析目标 agent：

- `resolveAgentDir(cfg, targetAgentId)`
- `resolveAgentConfig(cfg, targetAgentId)`
- `resolveSubagentModelAndThinkingPlan(...)`

模型和 thinking 的优先级包括：

- `sessions_spawn.model` / `sessions_spawn.thinking`
- target agent 的 subagent 配置
- `agents.defaults.subagents.*`
- caller 当前模型/思考等级的继承策略

如果模型解析失败，spawn 返回 error。

## 7. 子 agent 会话准备

### 7.1 初始 session patch

OpenClaw 会先写 child session 记录，包含：

- `spawnDepth`
- `subagentRole`
- `subagentControlScope`
- model / provider override
- thinking level

之后再写 lineage：

- `spawnedBy`
- `spawnedWorkspaceDir`

### 7.2 context 模式

`sessions_spawn.context` 支持：

- `isolated`：默认模式，创建干净子 session。
- `fork`：从 requester transcript 分叉。

`prepareSubagentSessionContext()` 处理 fork。当前实现中，`context: "fork"` 要求 target agent 与 requester agent 相同；跨 agent spawn 通常要用 `isolated`。

thread-bound spawns 如果没有显式 context，会按 channel thread binding policy 决定默认 context；非 thread spawn 默认 `isolated`。

### 7.3 workspace 继承

Codex 插件在构建工具时传入 `spawnWorkspaceDir`。`sessions_spawn` 再通过 `mapToolContextToSpawnedRunMetadata()` 和 `resolveSpawnedWorkspaceInheritance()` 解析最终子 workspace。

关键规则：

- 同 agent spawn 可以继承 caller 的真实 workspace。
- 跨 agent spawn 会忽略 caller inherited workspace，让 target agent 自己解析 workspace。
- 这避免 `main` agent 调度 `research` agent 时把 `main` 的 workspace 错带过去。

### 7.4 attachments

如果 `sessions_spawn` 带 attachments，会调用 `materializeSubagentAttachments()` 写入目标 agent 的附件目录，并把附件提示追加到 child system prompt。

相关安全点：

- mount path hint 会 sanitize。
- sessions transcript repair 会对 `sessions_spawn` attachment payload 做 redaction，避免敏感内容留在 transcript 工具调用块里。

### 7.5 context engine 准备

子 agent 启动前会调用 context engine 的 `prepareSubagentSpawn?.()`，传入 parent/child session key、context mode、session file、TTL 等信息。如果准备失败，会清理 provisional child session。

## 8. 真正启动子 agent run

准备完成后，`spawnSubagentDirect()` 通过 gateway 调用启动 child agent：

```ts
callSubagentGateway({
  method: "agent",
  params: {
    message: childTaskMessage,
    sessionKey: childSessionKey,
    channel: childSessionOrigin?.channel,
    to: childSessionOrigin?.to,
    accountId: childSessionOrigin?.accountId,
    threadId,
    idempotencyKey: childIdem,
    deliver,
    lane: AGENT_LANE_SUBAGENT,
    cleanupBundleMcpOnRunEnd: spawnMode !== "session",
    extraSystemPrompt: childSystemPrompt,
    thinking: thinkingOverride,
    timeout: runTimeoutSeconds,
    label,
    ...
  },
});
```

几个关键参数：

- `sessionKey`：新生成的 child session key，决定 target agent id 和 session 归属。
- `lane: AGENT_LANE_SUBAGENT`：子 agent 专用 lane。
- `extraSystemPrompt`：包含任务、父子 session、调度约束、可用命令指导、插件命令 guidance。
- `deliver`：通常为 false，表示 child run 不直接向外部聊天投递，而是完成后走 announce/handoff。
- `idempotencyKey`：避免重复启动。
- `cleanupBundleMcpOnRunEnd`：run 模式结束后清理 bundle MCP runtime。
- `thinking` / `timeout` / `label`：来自解析后的计划。

`callSubagentGateway()` 内部会根据方法需要选择 scope。对于 `method: "agent"`，它保持 write 级 least-privilege，而不是强行 admin scope，避免把 caller 当成 owner 并暴露 owner-only 工具。

## 9. run 注册与完成回传

启动成功后，`spawnSubagentDirect()` 调用 `registerSubagentRun()`，记录：

- `runId`
- `childSessionKey`
- `controllerSessionKey`
- `requesterSessionKey`
- requester origin
- requester display key
- `task`
- `cleanup`
- `label`
- `model`
- `agentDir`
- `workspaceDir`
- `runTimeoutSeconds`
- `expectsCompletionMessage`
- `spawnMode`
- attachments 信息

随后：

- 如果有 `subagent_spawned` hook，执行 lifecycle hook。
- 发出 session lifecycle event。
- 返回 `{ status: "accepted", childSessionKey, runId, mode, note, ... }` 给 `sessions_spawn`。
- `sessions_spawn` 把结果作为 JSON tool result 返回给 Codex app-server。

子 agent 完成后，OpenClaw 的 subagent registry / announce 流程会把结果回传给 requester session。Codex 插件不负责轮询子 agent，也不直接发送 child result；它只通过 OpenClaw 工具 result 告知 Codex spawn 已接受。

## 10. 子 run 如何重新进入 Codex 插件

`spawnSubagentDirect()` 启动子 agent 时调用的是 gateway 标准 `agent` 方法，而不是直接调用某个插件函数。因此子 run 会重新经过 OpenClaw 主流程：

```text
callSubagentGateway({ method: "agent", params: { sessionKey, lane: "subagent", ... } })
  -> src/agents/agent-command.ts
  -> src/agents/command/attempt-execution.ts
  -> runEmbeddedPiAgent(...)
  -> src/agents/pi-embedded-runner/run.ts
  -> selectAgentHarness(...)
  -> extensions/codex/harness.ts
  -> runCodexAppServerAttempt(...)
```

`src/agents/command/attempt-execution.ts` 在进入 embedded runner 时会把 target agent 的运行上下文传下去：

- `agentId: params.sessionAgentId`
- `sessionKey: params.sessionKey`
- `provider: embeddedPiProvider`
- `model: params.modelOverride`
- `agentHarnessId: sessionPinnedAgentHarnessId`
- `lane: params.opts.lane`
- `spawnedBy: params.spawnedBy`

这些参数让子 run 的执行方式由 target agent 的配置、当前模型/provider、session pinning 和 runtime policy 共同决定。

随后 `src/agents/pi-embedded-runner/run.ts` 调用 `selectAgentHarness()`。`src/agents/harness/selection.ts` 中的 `resolveAgentHarnessPolicy()` 会按以下来源解析 runtime：

1. 环境变量 `OPENCLAW_AGENT_RUNTIME`。
2. target agent 自己的 runtime 配置。
3. `agents.defaults` runtime 配置。
4. provider/model 的隐式默认策略。

当 target agent/provider/model 解析为 Codex runtime 时，harness selection 会选中 Codex 插件注册的 harness。`extensions/codex/harness.ts` 中 `createCodexAppServerAgentHarness()` 返回 id 为 `codex` 的 `AgentHarness`，其 `runAttempt` 会动态导入并执行：

```ts
runCodexAppServerAttempt(params, { pluginConfig })
```

所以“父 Codex 调 OpenClaw subagent，子 run 又调用到 Codex 插件”的本质是递归走了一遍 OpenClaw 标准 agent runtime pipeline：

```text
父 Codex run
  -> Codex dynamic tool: sessions_spawn
  -> OpenClaw core: spawnSubagentDirect()
  -> Gateway: agent command
  -> OpenClaw embedded runner: selectAgentHarness()
  -> Codex plugin harness
  -> 子 Codex run
```

Codex 插件不特殊识别“这是 subagent”；它只是在子 run 的 harness selection 阶段，作为符合 target runtime policy 的 agent harness 被选中。

## 11. subagent 之间如何保证并行

OpenClaw 的 subagent 并行由三层机制组成。

第一，`sessions_spawn` 是非阻塞工具。`spawnSubagentDirect()` 只等待 gateway 接受并启动子 run，随后注册 run 并返回 `{ status: "accepted", runId, childSessionKey }`。它不会等待子 agent 完成。因此父 Codex run 可以连续调用多次 `sessions_spawn`，一次派发多个子任务；之后可以调用 `sessions_yield` 让出当前 turn，等待完成事件回到 requester session。

第二，每个子 agent 拥有独立 session lane。`src/agents/pi-embedded-runner/run.ts` 会先计算：

```ts
const sessionLane = resolveSessionLane(params.sessionKey?.trim() || params.sessionId);
```

`resolveSessionLane()` 会把 session key 转成 `session:<key>`。而原生 subagent 的 child session key 形如：

```text
agent:<targetAgentId>:subagent:<uuid>
```

因此多个 child run 的 session lane 分别是：

```text
session:agent:<targetAgentId>:subagent:<uuid-1>
session:agent:<targetAgentId>:subagent:<uuid-2>
session:agent:<targetAgentId>:subagent:<uuid-3>
```

session lane 保证“同一个 session 内串行”，但不同 child session key 会进入不同 session lane，因此不会互相占用同一个 session 锁。

第三，所有 subagent 再进入共享的全局 `subagent` lane。`spawnSubagentDirect()` 启动 child agent 时显式传入：

```ts
lane: AGENT_LANE_SUBAGENT
```

`AGENT_LANE_SUBAGENT` 对应 `CommandLane.Subagent`，即字符串 `"subagent"`。`src/agents/pi-embedded-runner/run.ts` 会把它解析成 global lane：

```ts
const globalLane = resolveGlobalLane(params.lane);
```

最终执行时是双层入队：

```text
enqueueSession(() => enqueueGlobal(actualRun))
```

也就是说，每个 run 先受自己的 session lane 保护，再受全局 lane 的总并发控制。

全局 `subagent` lane 的并发由 gateway 启动时应用：

```ts
setCommandLaneConcurrency(CommandLane.Subagent, resolveSubagentMaxConcurrent(cfg));
```

`resolveSubagentMaxConcurrent()` 读取：

```text
agents.defaults.subagents.maxConcurrent
```

如果未配置，则使用 `DEFAULT_SUBAGENT_MAX_CONCURRENT = 8`。底层 `src/process/command-queue.ts` 的 drain 逻辑会在：

```ts
state.activeTaskIds.size < state.maxConcurrent
```

时持续从队列取任务启动。因此默认最多 8 个 subagent run 可以在同一个 gateway 进程内并行执行；超过 8 个会排队等待。

这套设计的并行语义是：

- 同一个子 session 内仍然串行，避免 transcript/session file 竞争。
- 不同子 session 可以并行，避免多个子任务互相阻塞。
- 所有 subagent 总量受 `agents.defaults.subagents.maxConcurrent` 有界控制。
- 单个父 session 的扇出受 `maxChildrenPerAgent` 控制。
- 嵌套层级受 `maxSpawnDepth` 控制，默认 depth 1 的子 agent 不能再 spawn 子 agent。

因此 OpenClaw 保证的是“有界并行 + session 隔离 + 全局限流”。如果把 `agents.defaults.subagents.maxConcurrent` 配成 1，那么多个子 agent 仍会被接受，但实际执行会在全局 `subagent` lane 上串行；如果保持默认 8 或显式调高，则多个 child session 能真正同时运行。

## 12. `subagents` 工具的控制能力

`src/agents/tools/subagents-tool.ts` 提供对已 spawn 子 agent 的控制：

- `subagents(action: "list")`
- `subagents(action: "kill", target: "...")`
- `subagents(action: "steer", target: "...", message: "...")`

它会：

1. 解析当前 requester session 的 controller。
2. 读取该 controller 控制范围内的子 agent runs。
3. list 时返回 active/recent 列表。
4. kill 时按 target 解析并终止 run。
5. steer 时向 active child run 发送 steering message。

因此 Codex 可以先用 `sessions_spawn` 调度 agent，再用 `subagents` 管理正在运行的子 agent。

## 13. `sessions_yield` 与 Codex 编排模式

`sessions_yield` 用于“派发子任务后让出当前 turn”。

Codex 插件在 `buildDynamicTools()` 中给 `createOpenClawCodingTools()` 传入 `onYield`：

- 设置 `yieldDetected = true`。
- 发出 `codex_app_server.tool` event，工具名为 `sessions_yield`。
- 调用 `runAbortController.abort("sessions_yield")`。

这让 Codex 可以形成以下编排：

```text
1. 调用 sessions_spawn 派发一个或多个子 agent。
2. 调用 sessions_yield 结束当前 Codex turn。
3. 等子 agent 完成后，OpenClaw 将完成事件/结果带回 requester session。
4. 后续 Codex turn 再基于结果继续处理。
```

### 13.1 如何促使 LLM 调用 `sessions_yield`

当前系统不能从模型层面“保证”LLM 一定会主动调用 `sessions_yield`。LLM 工具调用本质上仍是模型决策；OpenClaw 能做的是通过工具协议、工具返回值、运行时中断和完成回传机制，把正确路径设计成最自然、最低摩擦的路径。

现有保障分为几层。

第一层是工具描述。`sessions_yield` 的 tool description 明确写明：

```text
End your current turn. Use after spawning subagents to receive their results as the next message.
```

这让模型在工具选择阶段知道：它不是用来查询状态的工具，而是“spawn 后等待完成事件”的控制工具。

第二层是 `sessions_spawn` 的 accepted note。原生 subagent run 被接受后，OpenClaw 返回的 note 会明确要求：

- auto-announce 是 push-based。
- spawn children 之后不要调用 `sessions_list`、`sessions_history`、`exec sleep` 或任何 polling 工具。
- 跟踪 expected child session keys。
- 如果必要 child completion 还没到，调用 `sessions_yield` 结束当前 turn，等待 completion events 作为 user messages 到达。
- 只有所有 expected completions 到达后才发送最终答案。
- 如果 completion 在最终答案之后才到达，只回复 `NO_REPLY`。

这层提示非常关键：它把“等待子任务完成”从模型可能会误解的轮询问题，改写成“yield 当前 turn，等 OpenClaw 主动推送 completion”的协议问题。

第三层是 Codex 插件运行时回调。Codex 在创建 OpenClaw tools 时传入 `onYield` callback；一旦模型真的调用 `sessions_yield`，Codex 插件会：

```text
设置 yieldDetected = true
发出 codex_app_server.tool 事件
runAbortController.abort("sessions_yield")
```

也就是说，`sessions_yield` 一旦被调用，就不是普通 JSON tool result 后继续生成，而是变成一次干净的 turn yield。父 Codex run 会停止当前 turn，避免继续生成没有等到子结果的总结。

第四层是完成回传不依赖模型轮询。即使父模型没有调用 `sessions_yield`，已 spawn 的 subagent 仍然会继续运行；完成后由 OpenClaw subagent registry / announce 流程把结果送回 requester session。也就是说，`sessions_yield` 影响的是父 run 是否在当前编排点主动等待，而不是 child result 是否会丢失。

### 13.2 无法完全保证的边界

如果 LLM 调用 `sessions_spawn` 后没有调用 `sessions_yield`，OpenClaw 当前不会强行把它改写为 yield。可能出现的行为是：

```text
LLM spawn child
  -> sessions_spawn 返回 accepted
  -> LLM 忽略 note，继续直接回答用户
  -> child 后台继续执行
  -> child completion 后续再 announce 到 requester session
```

这种情况下，系统仍保持数据一致性和完成回传，但父 agent 的“当前答案”可能没有整合子 agent 结果。后续 completion 到达时，如果父 agent 已经给出 final answer，accepted note 要求它只回复 `NO_REPLY`，避免重复或迟到总结。

因此，当前实现的保证边界是：

- 能保证 `sessions_spawn` 非阻塞接受后 child run 继续执行。
- 能保证 `sessions_yield` 被调用后当前 Codex/OpenClaw turn 会 yield。
- 能保证 child completion 通过 announce/handoff 回到 requester session。
- 不能保证 LLM 一定主动调用 `sessions_yield`。
- 不能保证 LLM 不会在 child completion 到达前过早总结，除非运行时增加硬性策略。

### 13.3 如果需要硬保证，可以如何增强

如果产品目标是“只要本 turn spawn 了必要 subagent，就必须等待完成后再总结”，仅靠 prompt 和 tool description 不够，需要 runtime 级别约束。可选方案包括：

1. 在 `sessions_spawn` 返回结果中增加结构化元信息，例如 `nextRecommendedTool: "sessions_yield"`、`requiredCompletions`、`mustYield: true`。这仍是软约束，但比纯文本 note 更容易被工具策略或 harness 检测。
2. 在 tool loop / harness 层检测“本 turn 已 spawn active subagent，但没有调用 `sessions_yield` 且准备输出 final answer”，自动追加一次强提醒或阻止 final answer。
3. 增加 `autoYieldAfterSpawn` 配置：当 run 模式 subagent 被接受且没有 thread-bound direct delivery 时，runtime 在当前工具回合结束后自动 yield。
4. 增加组合工具，例如 `sessions_spawn_many_and_yield`，把“派发多个子任务 + yield 等待完成”做成一个原子编排工具，减少模型漏调 `sessions_yield` 的空间。
5. 增加显式 `subagents_wait` 工具，用于等待一组 `runId` / `childSessionKey` 完成；但它必须有严格 timeout、取消传播和最大等待限制，否则会把父 session/tool call 长时间占住，并削弱当前 background-task 架构的优势。

从当前架构看，最符合 OpenClaw 现有 subagent 模型的是第 2 或第 3 种：保留 `sessions_spawn` 非阻塞和 push-based announce，同时在 harness/runtime 里对“spawn 后直接 final”做策略约束。直接把 `sessions_spawn` 改成同步等待，不符合现有设计，也会让并行扇出退化成串行等待。

## 14. thread-bound 子 agent session

`sessions_spawn` 支持 `thread: true` 和 `mode: "session"`。这用于在支持 thread binding 的 channel 中创建持久子 agent 会话。

流程：

1. `sessions_spawn` 参数中请求 `thread: true`。
2. `spawnSubagentDirect()` 通过 `ensureThreadBindingForSubagentSpawn()` 调用 `subagent_spawning` hook。
3. channel/plugin hook 创建或绑定 thread。
4. child session origin 合并 thread delivery origin。
5. gateway `agent` 调用可设置 `deliver: true`，让初始 child run 直接进入绑定 thread。
6. 后续 thread 内消息可以继续路由到同一 child session。

如果当前 channel 不支持 thread binding，或 hook 不可用：

- `thread: true` 会失败。
- `mode: "session"` 也会失败并提示使用 `mode: "run"` 或 `sessions_send`。

## 15. Codex 插件自身的责任边界

Codex 插件在“调度 OpenClaw agent”中承担的是桥接责任：

- 把 `sessions_spawn`、`agents_list`、`subagents` 等 OpenClaw 工具暴露为 Codex dynamic tools。
- 把 Codex app-server 的 `item/tool/call` request 转成 OpenClaw `AnyAgentTool.execute()`。
- 给工具执行传入当前 Codex run 的 agent/session/channel/workspace/sandbox 上下文。
- 收集 tool telemetry，并把 tool result 返回给 Codex app-server。

Codex 插件不承担：

- target agent allowlist 决策。
- subagent depth / concurrency 限制。
- child session key 生成和 session store 写入。
- model/thinking 解析。
- workspace 继承策略。
- gateway `agent` 调用。
- lifecycle hook。
- 子 agent 完成 announce/handoff。
- subagent 的实际并发调度、session lane、全局 `subagent` lane 限流。

这些都属于 OpenClaw core 的 subagent 系统。

## 16. 安全边界

Codex 调度 OpenClaw agent 时受到多层约束：

- 工具可见性：当前 run 必须有效暴露 `sessions_spawn`。
- 工具 policy：profile、agent、group、sandbox、subagent depth policy 都可能移除 `sessions_spawn`。
- target allowlist：默认只能 spawn requester 自己，跨 agent 需要 `subagents.allowAgents`。
- `requireAgentId`：可强制显式指定 target。
- id 格式校验：非法 `agentId` 直接失败。
- depth 限制：`maxSpawnDepth` 阻止无限递归。
- child 数限制：`maxChildrenPerAgent` 控制并发。
- 全局 subagent lane 限制：`agents.defaults.subagents.maxConcurrent` 控制同时执行的子 run 数。
- sandbox 限制：sandboxed requester 不能 spawn unsandboxed child。
- leaf policy：leaf subagent 不再拥有 `sessions_spawn`。
- delivery 参数拒绝：`sessions_spawn` 不接受 channel/to/threadId 等投递参数。
- context fork 限制：跨 agent fork 不允许，避免把一个 agent transcript 直接分叉给另一个 agent。
- gateway scope：子 agent 启动保持最小必要 scope。
- Codex tool call 校验：Codex app-server 的 `item/tool/call` 必须匹配当前 thread/turn。

## 17. 与 ACP runtime 的区别

`sessions_spawn(runtime: "subagent")` 和 `sessions_spawn(runtime: "acp")` 是两条不同路径：

- `subagent`：调度 OpenClaw config agent，target 来自 `agents.list[]` / defaults，并受 `agents_list`、`subagents.allowAgents` 管理。
- `acp`：调度外部 ACP harness，target 是 ACP agent/harness id，受 ACP 配置和 ACP availability 管理。

系统 prompt 和 docs 都强调：ACP harness ids 不应通过 `agents_list` / `subagents` 来理解。你问的“OpenClaw 中定义的 agent”对应 `runtime: "subagent"`。

## 18. 结论

Codex 插件支持调度 OpenClaw agent 的实现方式，是把 OpenClaw 的 agent 调度工具作为 Codex app-server dynamic tools 暴露出去。Codex 模型可以调用 `agents_list` 查看允许 target 的 agent，调用 `sessions_spawn` 创建 `agent:<targetAgentId>:subagent:<uuid>` 子 agent run，调用 `subagents` 管理已创建的子 agent。

子 agent 启动后不会由 Codex 插件私有调度，而是重新进入 OpenClaw gateway 的标准 `agent` 命令路径，再由 embedded runner 按 target agent/provider/model 选择 harness。如果选择结果是 Codex runtime，子 run 会再次进入 `extensions/codex/harness.ts`，作为一个独立 Codex app-server run 执行。

并行性由 OpenClaw core 保证：`sessions_spawn` 非阻塞返回；每个子 agent 使用独立 `session:<childSessionKey>` lane；所有子 agent 共享专用全局 `subagent` lane；`agents.defaults.subagents.maxConcurrent` 默认 8，控制实际同时执行的子 run 数。Codex 插件只负责把工具调用桥接进 OpenClaw 工具系统，不负责 child session、harness selection、并发 lane、生命周期 hook 或完成 announce。
