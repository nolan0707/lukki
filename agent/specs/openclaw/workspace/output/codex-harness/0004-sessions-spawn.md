# OpenClaw sessions_spawn and sessions_send notes

## Terminology

The tool names are:

- `sessions_spawn`
- `sessions_send`

`session_spwan` is a typo. OpenClaw uses plural `sessions_*`, and `spawn` is spelled normally.

## Short answer

Use `sessions_spawn` when the parent agent needs to delegate a new unit of work to a child run.

Use `sessions_send` when the parent agent already has a target session and wants to send that existing session a message, optionally waiting for a reply.

In other words:

- `sessions_spawn` creates and schedules a new child session/run.
- `sessions_send` sends an inter-session message into an existing session/run context.

## What is an agent?

An OpenClaw agent is a configured, durable runtime profile.

It usually carries an `agentId` and associated defaults such as model, thinking level, tool policy, sandbox/runtime, workspace, and channel behavior. A configured agent is a reusable role or capability boundary, for example `planner`, `reviewer`, or a coding/runtime-specific agent.

Agent configuration is the source of policy for what the agent can do and what other agents it may target. For `sessions_spawn`, relevant policy includes settings such as `subagents.allowAgents`, `subagents.requireAgentId`, spawn depth, and child limits.

## What is a subagent?

A subagent is not a new static configuration profile. It is a runtime child session/run created by `sessions_spawn`.

The child session has lineage metadata such as:

- `spawnedBy`
- `spawnDepth`
- `subagentRole`
- `subagentControlScope`

The subagent registry tracks the child run after it starts. When the run completes, the registry handles completion announcement and cleanup. Depending on spawn mode and cleanup settings, the child session may be retained or deleted/archived.

## sessions_spawn default agent profile

For native subagent spawns, if `sessions_spawn` omits `agentId`, OpenClaw uses the requester agent profile.

The relevant logic is in `src/agents/subagent-spawn.ts`:

```ts
const targetAgentId = requestedAgentId ? normalizeAgentId(requestedAgentId) : requesterAgentId;
```

That means:

- If `agentId` is passed, the child uses that target agent profile, if policy allows it.
- If `agentId` is omitted, the child uses the calling/requester agent profile.
- If `subagents.requireAgentId` is enabled, omitting `agentId` is rejected.

For `runtime: "acp"`, the behavior is different. ACP spawn goes through `src/agents/acp-spawn.ts`; when no target is configured or passed, OpenClaw expects ACP targeting to be resolved from ACP configuration such as `acp.defaultAgent`, otherwise it returns an ACP target configuration error.

## sessions_spawn scheduling flow

The tool entry point is `src/agents/tools/sessions-spawn-tool.ts`.

High-level flow:

1. Parse tool arguments: `task`, `label`, `agentId`, `runtime`, `model`, `thinking`, `thread`, `mode`, `cleanup`, `sandbox`, `context`, attachments, and timeout.
2. Choose runtime:
   - `runtime: "subagent"` uses native OpenClaw subagent flow.
   - `runtime: "acp"` uses ACP harness flow.
3. Reject unsupported channel-delivery parameters. `sessions_spawn` does not accept `target`, `channel`, `to`, `threadId`, `replyTo`, or `transport`; child delivery should happen through `message` or `sessions_send`.
4. For native subagents, call `spawnSubagentDirect`.
5. For ACP, call `spawnAcpDirect`.

Native spawn implementation is in `src/agents/subagent-spawn.ts`.

Native `sessions_spawn` creates a new child session key similar to:

```text
agent:<targetAgentId>:subagent:<uuid>
```

Then it:

1. Resolves requester session and requester agent.
2. Checks spawn depth and max active children.
3. Checks `requireAgentId` and `allowAgents`.
4. Checks sandbox inheritance and `sandbox: "require"`.
5. Resolves child model and thinking defaults.
6. Writes provisional child session metadata.
7. Prepares child context:
   - default isolated context
   - optional `context: "fork"`
   - thread-bound defaults may use fork depending on channel config
8. Optionally creates/binds a channel thread for `thread: true`.
9. Builds the child system prompt and initial child task message.
10. Starts the child run through Gateway `agent` RPC.

The child run is scheduled on the subagent lane:

```ts
lane: AGENT_LANE_SUBAGENT
```

For ordinary background subagent work, delivery is not directly sent to the chat during the child run:

```ts
deliver: false
```

After the Gateway accepts the child run, `sessions_spawn` registers it in the subagent registry and returns immediately:

```json
{
  "status": "accepted",
  "runId": "...",
  "childSessionKey": "agent:<id>:subagent:<uuid>"
}
```

This is why `sessions_spawn` is non-blocking. The parent should not expect the tool call itself to contain the child result.

## Subagent completion and announcement

After `sessions_spawn` starts the child run, `registerSubagentRun` records it in the subagent registry.

The registry:

1. Persists the run record.
2. Starts or ensures lifecycle listeners.
3. Starts a sweeper for recovery and cleanup.
4. Calls `agent.wait` for the child run in the background.
5. Completes the subagent run when the child finishes.
6. Runs announcement and cleanup flow.

Completion delivery is handled by `runSubagentAnnounceFlow`. This posts the child result back to the requester session/channel when `expectsCompletionMessage` is true.

If the parent wants to pause its turn until spawned work reports back, it can use `sessions_yield`. Otherwise the result arrives later through the normal completion announcement path.

## sessions_send scheduling flow

The tool entry point is `src/agents/tools/sessions-send-tool.ts`.

High-level flow:

1. Parse `message` plus either `sessionKey` or `label`.
2. Resolve labels through `sessions.resolve` if needed.
3. Normalize aliases/session IDs into a canonical session key.
4. Enforce visibility policy.
5. Enforce agent-to-agent policy for cross-agent sends.
6. Reject thread-scoped target sessions for inter-agent coordination.
7. Annotate the message as inter-session provenance.
8. Start a Gateway `agent` RPC targeting the existing session.
9. Either return immediately or wait for the target run.

The sent message is wrapped as inter-session data, not as direct end-user text. The receiver sees it with provenance similar to:

```text
[Inter-session message ... isUser=false]
```

This matters because a target agent should treat the message as tool-routed context from another session, not as an instruction directly authored by the human user.

By default, `sessions_send` waits up to `timeoutSeconds` and then reads the latest assistant reply from the target session. With:

```json
{ "timeoutSeconds": 0 }
```

it becomes fire-and-forget.

## Gateway sessions.send versus tool sessions_send

There are two related but different concepts:

- Agent tool: `sessions_send`
- Gateway RPC: `sessions.send`

The agent tool `sessions_send` is for agent-to-agent/session coordination. It calls Gateway `agent` internally, uses inter-session provenance, may wait for `agent.wait`, and may trigger A2A announce/reply behavior.

Gateway `sessions.send` is a control-plane RPC for sending a message into an existing session. Its handler is in `src/gateway/server-methods/sessions.ts`, and it delegates to `chat.send`.

So:

- Use `sessions_send` when an agent is coordinating with another session.
- Use Gateway `sessions.send` when a client/UI/SDK wants to send a normal message to an existing session.

## Choosing between sessions_spawn and sessions_send

Use `sessions_spawn` for:

- New delegated tasks.
- Parallel subtasks.
- Background research or implementation.
- One-shot worker runs.
- A child session with its own transcript.
- A persistent thread-bound child session.
- Running a different configured agent profile via `agentId`.

Use `sessions_send` for:

- Continuing conversation with an existing session.
- Sending follow-up constraints to a previously spawned child.
- Asking an already-running or retained child session a question.
- Cross-agent messaging where both sessions already exist.
- Fire-and-forget nudges or waited replies.

Examples:

```text
"Split this into frontend and backend work."
=> sessions_spawn twice, one child per work unit.
```

```text
"Tell the previous backend child to also check migrations."
=> sessions_send to that child session key.
```

```text
"Ask reviewer agent for feedback on this plan."
=> sessions_spawn with agentId: "reviewer" if this is a new review task.
=> sessions_send if a reviewer session already exists and should continue.
```

```text
"Keep a long-lived thread for this investigation."
=> sessions_spawn with thread: true and mode: "session".
```

## Context behavior

Native `sessions_spawn` supports context modes:

- `context: "isolated"` gives the child a clean task context.
- `context: "fork"` branches the requester transcript into the child session.

For most delegation, isolated is preferable because it keeps child work focused and avoids leaking unnecessary context. Use fork when the child must understand the current conversation or prior tool results.

Thread-bound native subagents may default to fork depending on thread binding config, because a persistent child thread often needs the surrounding session context.

`sessions_send` does not create a new context. It sends a new message into the target session's existing transcript.

## Model and thinking inheritance

For native `sessions_spawn`:

- Explicit `model` on the tool call wins.
- Otherwise subagent model defaults may apply from config.
- Otherwise the child inherits from the requester/target agent behavior.

For thinking:

- Explicit `thinking` on the tool call wins.
- Otherwise configured subagent thinking defaults may apply.
- Otherwise the normal agent default applies.

This is separate from `agentId`. `agentId` chooses the target agent profile; `model` and `thinking` can override parts of the runtime plan for that spawned child run.

## Depth and recursive spawning

OpenClaw limits recursive subagent orchestration.

Default leaf subagents do not get session orchestration tools. When `maxSpawnDepth >= 2`, an intermediate child may receive a constrained set of orchestration tools such as `sessions_spawn`, `subagents`, `sessions_list`, and `sessions_history`, so it can manage its own children.

At the leaf depth, `sessions_spawn` is denied.

This prevents uncontrolled recursive delegation and keeps the parent-child tree bounded.

## Practical recommendation

For multi-agent collaboration, default to `sessions_spawn` for the first assignment of work. It creates a clearly owned work unit with an isolated child session, lifecycle tracking, and completion announcement.

Use `sessions_send` after that when you already know which session should receive a follow-up.

The usual pattern is:

1. Parent uses `sessions_spawn` to create one or more child work units.
2. Parent continues doing non-overlapping work or yields.
3. Child completion announces back to parent.
4. Parent uses `sessions_send` only for follow-up questions, corrections, or continued conversation with a known child session.

## Source files

Key implementation files:

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

Relevant docs:

- `docs/concepts/session-tool.md`
- `docs/tools/subagents.md`
- `docs/gateway/protocol.md`
