# OpenClaw 中的 sessions_spawn 与 sessions_send

## 术语

OpenClaw 里实际的工具名是：

- `sessions_spawn`
- `sessions_send`

`session_spwan` 是拼写错误。仓库中使用复数形式 `sessions_*`，并且 `spawn` 按正常英文拼写。

## 简短结论

当父 agent 需要把一个新的工作单元委派给子运行时，用 `sessions_spawn`。

当父 agent 已经知道一个目标 session，并希望给这个已有 session 发送一条消息、可选等待回复时，用 `sessions_send`。

换句话说：

- `sessions_spawn`：创建并调度一个新的 child session/run。
- `sessions_send`：向已有 session/run 上下文发送一条 inter-session 消息。

## agent 是什么

OpenClaw 里的 agent 是一个配置化、可复用、相对长期存在的运行档案。

它通常包含 `agentId`，以及模型、thinking 级别、工具策略、sandbox/runtime、workspace、channel 行为等默认配置。一个配置好的 agent 更像是一个角色或能力边界，例如 `planner`、`reviewer`，或者某个特定代码/运行时 agent。

agent 配置也是策略来源。对于 `sessions_spawn` 来说，相关策略包括 `subagents.allowAgents`、`subagents.requireAgentId`、spawn 深度限制、每个 agent 的最大 child 数量等。

## subagent 是什么

subagent 不是一个新的静态配置档案。它是 `sessions_spawn` 在运行时创建出来的 child session/run。

child session 会记录父子关系和控制元数据，例如：

- `spawnedBy`
- `spawnDepth`
- `subagentRole`
- `subagentControlScope`

子运行启动后，会由 subagent registry 继续跟踪。运行结束后，registry 负责 completion announce 和 cleanup。根据 spawn mode 与 cleanup 设置，child session 可能被保留，也可能被删除/归档。

## sessions_spawn 未指定 agentId 时使用谁的档案

对 native subagent spawn 来说，如果 `sessions_spawn` 没有传 `agentId`，OpenClaw 使用调用者，也就是 requester agent 的档案。

关键逻辑在 `src/agents/subagent-spawn.ts`：

```ts
const targetAgentId = requestedAgentId ? normalizeAgentId(requestedAgentId) : requesterAgentId;
```

含义是：

- 传了 `agentId`：child 使用指定的目标 agent 档案，前提是策略允许。
- 没传 `agentId`：child 使用当前调用/requester agent 的档案。
- 如果启用了 `subagents.requireAgentId`，不传 `agentId` 会被拒绝。

对于 `runtime: "acp"`，路径不同。ACP spawn 走 `src/agents/acp-spawn.ts`。如果没有传目标，也没有通过 ACP 配置解析到目标，例如 `acp.defaultAgent`，OpenClaw 会返回 ACP target 配置错误。

## sessions_spawn 的调度流程

工具入口在 `src/agents/tools/sessions-spawn-tool.ts`。

高层流程：

1. 解析工具参数：`task`、`label`、`agentId`、`runtime`、`model`、`thinking`、`thread`、`mode`、`cleanup`、`sandbox`、`context`、attachments 和 timeout。
2. 选择 runtime：
   - `runtime: "subagent"` 使用 OpenClaw native subagent 流程。
   - `runtime: "acp"` 使用 ACP harness 流程。
3. 拒绝 channel-delivery 参数。`sessions_spawn` 不接受 `target`、`channel`、`to`、`threadId`、`replyTo`、`transport`；child 如需发送消息，应使用 `message` 或 `sessions_send`。
4. native subagent 调用 `spawnSubagentDirect`。
5. ACP 调用 `spawnAcpDirect`。

native spawn 的实现位于 `src/agents/subagent-spawn.ts`。

native `sessions_spawn` 会创建一个新的 child session key，形态类似：

```text
agent:<targetAgentId>:subagent:<uuid>
```

然后它会：

1. 解析 requester session 和 requester agent。
2. 检查 spawn depth 和 max active children。
3. 检查 `requireAgentId` 与 `allowAgents`。
4. 检查 sandbox 继承以及 `sandbox: "require"`。
5. 解析 child 的 model 和 thinking 默认值。
6. 写入 provisional child session 元数据。
7. 准备 child context：
   - 默认 isolated context
   - 可选 `context: "fork"`
   - thread-bound 场景可能根据 channel 配置默认使用 fork
8. 如果 `thread: true`，尝试创建或绑定 channel thread。
9. 构造 child system prompt 和初始 child task message。
10. 通过 Gateway `agent` RPC 启动 child run。

child run 会被放到 subagent lane 上调度：

```ts
lane: AGENT_LANE_SUBAGENT
```

普通后台 subagent 运行期间不会直接向聊天面投递消息：

```ts
deliver: false
```

Gateway 接受 child run 后，`sessions_spawn` 会把它注册到 subagent registry，并立即返回：

```json
{
  "status": "accepted",
  "runId": "...",
  "childSessionKey": "agent:<id>:subagent:<uuid>"
}
```

这就是为什么 `sessions_spawn` 是非阻塞的。父 agent 不应该期待工具调用本身直接包含 child 的最终结果。

## subagent 完成与结果通知

`sessions_spawn` 启动 child run 后，`registerSubagentRun` 会把它记录到 subagent registry。

registry 会：

1. 持久化 run record。
2. 启动或确认 lifecycle listener。
3. 启动 sweeper，用于恢复和清理。
4. 在后台对 child run 调用 `agent.wait`。
5. child 完成后标记 subagent run 完成。
6. 进入 announce/cleanup flow。

结果通知由 `runSubagentAnnounceFlow` 处理。当 `expectsCompletionMessage` 为 true 时，它会把 child 的结果发回 requester session/channel。

如果父 agent 希望暂停当前 turn，直到 spawned work 回来，可以使用 `sessions_yield`。否则结果会稍后通过正常 completion announcement 路径回来。

## sessions_send 的调度流程

工具入口在 `src/agents/tools/sessions-send-tool.ts`。

高层流程：

1. 解析 `message`，以及 `sessionKey` 或 `label`。
2. 如有需要，通过 `sessions.resolve` 把 label 解析成 session key。
3. 将 alias/session ID 归一化为 canonical session key。
4. 执行 visibility policy 检查。
5. 对跨 agent 发送执行 agent-to-agent policy 检查。
6. 拒绝把 thread-scoped session 作为 inter-agent coordination 目标。
7. 给消息加上 inter-session provenance。
8. 启动 Gateway `agent` RPC，目标是已有 session。
9. 根据 `timeoutSeconds` 立即返回或等待目标 run。

发送出去的消息会被标记为 inter-session data，而不是直接的终端用户文本。接收侧会看到类似这样的 provenance：

```text
[Inter-session message ... isUser=false]
```

这点很重要：目标 agent 应把它当作来自另一个 session 的工具路由上下文，而不是人类用户直接写下的指令。

默认情况下，`sessions_send` 会等待最多 `timeoutSeconds`，然后读取目标 session 最新的 assistant 回复。如果传：

```json
{ "timeoutSeconds": 0 }
```

它就变成 fire-and-forget。

## Gateway sessions.send 与工具 sessions_send 的区别

这里有两个相关但不同的概念：

- agent 工具：`sessions_send`
- Gateway RPC：`sessions.send`

agent 工具 `sessions_send` 用于 agent 到 agent/session 的协同。它内部调用 Gateway `agent`，使用 inter-session provenance，可能等待 `agent.wait`，并可能触发 A2A announce/reply 行为。

Gateway `sessions.send` 是控制面 RPC，用于客户端、UI、SDK 向已有 session 发送普通消息。它的 handler 在 `src/gateway/server-methods/sessions.ts`，内部会委托给 `chat.send`。

所以：

- agent 协调另一个 session 时，用 `sessions_send`。
- 客户端/UI/SDK 给已有 session 发送普通消息时，用 Gateway `sessions.send`。

## 如何在 sessions_spawn 和 sessions_send 之间选择

适合用 `sessions_spawn` 的场景：

- 新的委派任务。
- 并行子任务。
- 后台调研或实现。
- 一次性 worker run。
- 需要一个拥有独立 transcript 的 child session。
- 需要 persistent thread-bound child session。
- 需要通过 `agentId` 使用另一个已配置 agent 档案。

适合用 `sessions_send` 的场景：

- 继续和已有 session 对话。
- 给之前 spawn 出来的 child 补充约束。
- 询问一个仍在运行或被保留的 child session。
- 两个已存在 session 之间跨 agent 发送消息。
- fire-and-forget 提醒，或等待一次回复。

例子：

```text
"把这个任务拆成前端和后端两部分。"
=> 调用两次 sessions_spawn，每个 child 负责一个工作单元。
```

```text
"告诉刚才的后端 child 顺便检查 migrations。"
=> 对那个 child session key 调用 sessions_send。
```

```text
"请 reviewer agent 审查这个方案。"
=> 如果是新审查任务，用 sessions_spawn 并指定 agentId: "reviewer"。
=> 如果 reviewer session 已存在且应该延续上下文，用 sessions_send。
```

```text
"为这个调查保留一个长期 thread。"
=> 使用 sessions_spawn，传 thread: true 和 mode: "session"。
```

## context 行为

native `sessions_spawn` 支持 context mode：

- `context: "isolated"` 给 child 一个干净的任务上下文。
- `context: "fork"` 把 requester 当前 transcript 分支到 child session。

大多数委派任务更适合 isolated，因为它让 child 工作更聚焦，也避免泄露不必要的上下文。只有当 child 必须理解当前对话或之前的工具结果时，才使用 fork。

thread-bound native subagent 可能根据配置默认使用 fork，因为长期 child thread 往往需要周围 session 上下文。

`sessions_send` 不会创建新上下文。它只是向目标 session 的已有 transcript 追加一条新消息。

## model 和 thinking 继承

对 native `sessions_spawn` 来说：

- 工具调用里显式传入的 `model` 优先。
- 否则可能使用配置里的 subagent model default。
- 再否则使用 requester/target agent 的默认行为。

thinking 也是类似：

- 工具调用里显式传入的 `thinking` 优先。
- 否则可能使用配置里的 subagent thinking default。
- 再否则使用普通 agent 默认值。

这和 `agentId` 是两件事。`agentId` 决定目标 agent 档案；`model` 和 `thinking` 可以覆盖这个 spawned child run 的部分运行计划。

## 深度和递归 spawn

OpenClaw 会限制递归 subagent orchestration。

默认 leaf subagent 不会获得 session orchestration 工具。当 `maxSpawnDepth >= 2` 时，中间层 child 可能获得受限的一组 orchestration 工具，例如 `sessions_spawn`、`subagents`、`sessions_list`、`sessions_history`，让它可以管理自己的 children。

到 leaf depth 后，`sessions_spawn` 会被拒绝。

这可以避免失控的递归委派，并保持 parent-child tree 有边界。

## 实践建议

做多 agent 协同时，第一次分配工作默认用 `sessions_spawn`。它会创建一个边界清晰的工作单元，拥有 isolated child session、生命周期跟踪和 completion announcement。

之后如果已经知道要找哪个 session 继续沟通，再用 `sessions_send`。

常见模式是：

1. parent 用 `sessions_spawn` 创建一个或多个 child work unit。
2. parent 继续处理不重叠的工作，或者调用 `sessions_yield` 等结果。
3. child 完成后 announce 回 parent。
4. parent 只在需要追问、修正或延续已知 child session 对话时使用 `sessions_send`。

## 关键源码

核心实现文件：

- `src/agents/tools/sessions-spawn-tool.ts`
- `src/agents/subagent-spawn.ts`
- `src/agents/subagent-registry.ts`
- `src/agents/subagent-registry-run-manager.ts`
- `src/agents/subagent-registry-lifecycle.ts`
- `src/agents/subagent-announce.ts`
- `src/agents/tools/sessions-send-tool.ts`
- `src/agents/run-wait.ts`
- `src/gateway/server-methods/sessions.ts`
- `src/gateway/server-methods/agent.ts`

相关文档：

- `docs/concepts/session-tool.md`
- `docs/tools/subagents.md`
- `docs/gateway/protocol.md`
