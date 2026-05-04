# 基于 SDK query 的用户任务级并行开发规格

## 1. 目标

实现用户任务级并行：多个用户任务可以在任意时间提交，每个任务独立运行一个 SDK `query()` 分支；任务完成后，系统自动把该任务分支产生的增量 transcript 合并回主会话。

本规格用于指导开发，不是方案讨论稿。推荐落地路径是“方案三：扩展 SDK control 协议”，同时保留另外两种方案作为边界和备选。

核心约束：

1. `SessionActor` 只负责会话管理，不调用 `query()` 发起 turn。
2. 主会话 transcript 必须保持单写者和线性 `parentUuid` 链。
3. 用户任务不按读/写分类；任务可能包含读、写、命令执行、MCP 调用等任意 agentic 行为。
4. 并行层只负责分支执行、事件归属、取消、增量 transcript 合并。
5. 任务完成后的合并对象是任务分支真实增量 transcript，不是合成 user message，也不是摘要。
6. 工具是否允许执行由现有 `permissionMode`、`canUseTool`、hooks、sandbox、MCP 权限系统决定。
7. 文件级副作用冲突不在本会话合并层解决。

## 2. 源码依据

开发前需要理解以下 Claude Code 源码事实。

### 2.1 `QueryEngine` 是会话级对象

`vendor/Claude-code-open/src/QueryEngine.ts` 的语义是：

```text
One QueryEngine per conversation.
Each submitMessage() call starts a new turn within the same conversation.
State persists across turns.
```

结论：

- 不能让多个用户任务并发调用同一个 `QueryEngine.submitMessage()`。
- `mutableMessages`、read file cache、usage、permission denials 等状态不能被多个任务并发写。

### 2.2 `query()` 是一个完整 agentic turn

`vendor/Claude-code-open/src/query.ts` 内部会循环执行：

```text
model stream
  -> assistant/tool_use
  -> tool execution
  -> tool_result user message
  -> maybe next model call
```

结论：

- 一个用户任务的并行单位应是一个完整 `query()` loop。
- 不要把多个用户任务塞进同一个 `query()` loop 并发调度。

### 2.3 transcript 是线性 `parentUuid` 链

`vendor/Claude-code-open/src/utils/sessionStorage.ts` 中：

- `recordTranscript()` 会过滤已记录消息，并假设已记录消息通常是前缀。
- `Project.insertMessageChain()` 会按顺序设置 `parentUuid`。
- `loadTranscriptFile()` / resume 依赖 `parentUuid` 链还原会话。
- compact boundary 会特殊处理 `parentUuid` / `logicalParentUuid`。

结论：

- 主会话 transcript 必须单写。
- worker 不能直接写主 transcript。
- 任务完成后必须通过主会话 merge lock 重写并追加增量。

### 2.4 已有 fork / sidechain 思路

可参考：

- `vendor/Claude-code-open/src/tasks/LocalMainSessionTask.ts`
- `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts`
- `vendor/Claude-code-open/src/utils/forkedAgent.ts`
- `vendor/Claude-code-open/src/commands/branch/branch.ts`

这些实现证明 Claude Code 已有“复制当前上下文，独立运行分支，写 sidechain / fork transcript”的模式。

### 2.5 SDK control 通道已存在

可参考：

- `vendor/Claude-code-open/src/entrypoints/sdk/controlSchemas.ts`
- `vendor/Claude-code-open/src/entrypoints/sdk/controlTypes.ts`
- `vendor/Claude-code-open/src/cli/structuredIO.ts`
- `vendor/Claude-code-open/src/cli/print.ts`

`print.ts` 已有大量 `control_request` 分支，例如 `interrupt`、`set_model`、`mcp_status`、`rewind_files`、`stop_task`。方案三应沿用这个通道新增 `merge_session_delta`。

## 3. 推荐架构

```text
Client/API
  -> TaskScheduler.submit(req)
      -> SessionActor.forkForTask(req)
      -> TaskWorkerPool.run(snapshot, prompt)
          -> SDK query({ resume, resumeSessionAt, forkSession, sessionId })
          -> EventMux.emit(task_event)
      -> SessionActor.withMergeLock(...)
          -> mergeSessionDelta(control request)
          -> update currentHead
```

组件职责：

| 组件 | 职责 | 禁止事项 |
| --- | --- | --- |
| `SessionActor` | 管理 `sessionId`、`currentHeadUuid`、fork snapshot、merge lock、主会话 mutation 顺序 | 不调用 `query()` |
| `TurnRunner` | 串行执行 foreground turn，调用 SDK `query()` 或 `streamInput()` | 不并发执行同一主会话 turn |
| `TaskScheduler` | 在线接收任意时间到达的任务，立即返回 `TaskHandle`，异步派发 worker 和自动 merge | 不按读/写分类 |
| `TaskWorkerPool` | 每个任务独立 `query()`，独立 `sessionId` 和 `AbortController` | 不写主 transcript |
| `EventMux` | 包装 SDK event，附加 `taskId`、`branchSessionId`、`seq` | 不丢失事件归属 |
| `mergeSessionDelta` | 读取 branch delta，重写并追加到 target session | 不生成合成 user message |

## 4. 数据模型

### 4.1 任务请求

```ts
type ParallelTaskRequest = {
  id?: string
  prompt: string
  cwd: string
}
```

注意：这里没有 `kind: readonly | write`。调度层不分类，权限和工具副作用由现有系统处理。

### 4.2 任务句柄

```ts
type TaskHandle = {
  taskId: string
  done: Promise<TaskResult>
  cancel(): void
}
```

`done` 的语义：

- worker 执行完成。
- 自动 merge 已完成或明确失败。
- 因此 `await handle.done` 后，主会话要么已经包含任务增量，要么返回可诊断错误。

### 4.3 fork 快照

```ts
type TaskForkSnapshot = {
  taskId: string
  branchSessionId: string
  baseSessionId: string
  baseHeadUuid: string | null
  cwd: string
  permissionMode: PermissionMode
  allowedTools: string[]
  disallowedTools: string[]
  sessionStore?: SessionStore
}
```

`baseHeadUuid` 必须是创建任务分支时主会话的当前 head。任务可能稍后才完成，但合并时仍要保留这个 origin。

### 4.4 任务结果

```ts
type TaskResult = {
  taskId: string
  branchSessionId: string | null
  baseHeadUuid: string | null
  ok: boolean
  resultMessage?: SDKResultMessage
  merge?: {
    mergedCount: number
    skippedCount: number
    newHeadUuid: string | null
  }
  error?: string
}
```

推荐让 branch delta 由 `mergeSessionDelta` 从 branch transcript 中读取，而不是只依赖 SDK stream 中收集到的消息。原因是 transcript reader 能覆盖恢复、崩溃补偿、content replacement、compact boundary 等持久化语义。

## 5. 执行流程

### 5.1 在线任务提交

任务可以在任意时间到达，不要求一批任务同时提交。

```text
submit(task)
  -> allocate taskId
  -> return TaskHandle immediately
  -> async:
       snapshot = SessionActor.forkForTask(task)
       result = TaskWorkerPool.run(snapshot, task.prompt)
       merge = SessionActor.withMergeLock(currentHead => mergeSessionDelta(...))
       resolve TaskHandle.done
```

后到达的任务基于提交派发时的 `currentHeadUuid` fork。已经运行的任务继续使用自己的 fork 上下文，不会被主会话后续 turn 修改。

### 5.2 worker 执行

每个任务一个独立 SDK `query()`：

```ts
const q = query({
  prompt: taskPrompt,
  options: {
    cwd: snapshot.cwd,
    sessionId: snapshot.branchSessionId,
    resume: snapshot.baseSessionId,
    resumeSessionAt: snapshot.baseHeadUuid ?? undefined,
    forkSession: true,
    abortController,
    permissionMode: snapshot.permissionMode,
    allowedTools: snapshot.allowedTools,
    disallowedTools: snapshot.disallowedTools,
    sessionStore: snapshot.sessionStore,
    persistSession: true,
  },
})
```

关键点：

- `resume + resumeSessionAt + forkSession` 表示从主会话指定 head fork。
- `sessionId` 是 branch session ID，不是主 session ID。
- `persistSession: true`，否则完成后无法可靠读取 branch transcript delta。
- `continue` 禁止用于 task worker，因为它依赖“最近会话”隐式选择。

### 5.3 事件流

worker 输出事件时必须包装任务归属：

```ts
type TaskEvent = {
  type: 'task_event'
  taskId: string
  branchSessionId: string
  baseSessionId: string
  baseHeadUuid: string | null
  seq: number
  message: SDKMessage
}
```

要求：

- 每个 task 内部 `seq` 单调递增。
- 不同 task 的事件允许交错。
- permission request、tool progress、result 都必须能路由回对应 task。
- cancel task 只 abort 对应 task 的 `AbortController`。

### 5.4 自动合并

任务完成后自动合并：

```text
branch transcript
  -> read delta after baseHeadUuid
  -> validate delta
  -> rewrite sessionId to targetSessionId
  -> rewrite parentUuid from targetParentUuid
  -> attach mergedFrom metadata
  -> append to target transcript
  -> return newHeadUuid
```

不是：

```text
append SDKUserMessage { shouldQuery: false, content: "任务结果摘要..." }
```

也不是：

```text
append synthetic user message with full result text
```

合并后主 transcript 应保留真实的 assistant/tool_use/tool_result 轨迹，后续 resume 和模型上下文能看到任务真实执行过程。

## 6. 三种实现方案

### 6.1 方案一：宿主侧 SessionStore 镜像

宿主系统通过 `sessionStore` 记录 branch transcript，自行维护主会话 transcript。

优点：

- 不改 `vendor/Claude-code-open/src`。
- 完全基于公开 SDK `query()`。

缺点：

- Claude Code 本地 JSONL 不会自动包含合并后的 delta。
- 原生 `resume` 看不到宿主侧合并结果。
- 宿主侧要自己实现 parentUuid、compact boundary、tool_use/tool_result 校验。

适用：

- 产品已有独立会话存储。
- Claude Code 只作为执行器。

### 6.2 方案二：同仓内部 transcript writer

在同仓内直接新增内部 helper，复用 `sessionStorage.ts` 写 transcript。

优点：

- 改动小，适合原型。
- 合并后的内容能进入 Claude Code 本地 transcript。

缺点：

- 不是公开 SDK 能力。
- 容易受 `getSessionId()`、`getCwd()` 等全局状态影响。

适用：

- 并行调度器和 Claude Code 源码在同一仓内。
- 先验证 transcript delta merge 语义。

### 6.3 方案三：扩展 SDK control 协议

推荐长期实现。SDK 外部编排仍用 `query()` 执行分支，合并通过 SDK control request 交给 Claude Code CLI 内部完成。

优点：

- 仍以 `query()` 作为任务执行入口。
- transcript 读写复用 Claude Code 内部实现。
- 合并后的主 session 能被原生 `resume` 读取。
- 可以稳定封装为 SDK 能力。

需要修改：

```text
vendor/Claude-code-open/src/entrypoints/sdk/controlSchemas.ts
vendor/Claude-code-open/src/entrypoints/sdk/controlTypes.ts
vendor/Claude-code-open/src/entrypoints/agentSdkTypes.ts
vendor/Claude-code-open/src/cli/print.ts
vendor/Claude-code-open/src/utils/sessionStorage.ts
```

## 7. 方案三实现规格

### 7.1 新增 SDK control schema

在 `entrypoints/sdk/controlSchemas.ts` 增加：

```ts
export const SDKControlMergeSessionDeltaRequestSchema = lazySchema(() =>
  z.object({
    subtype: z.literal('merge_session_delta'),
    source_session_id: z.string().uuid(),
    target_session_id: z.string().uuid(),
    source_base_head_uuid: z.string().uuid().nullable(),
    target_parent_uuid: z.string().uuid().nullable(),
    task_id: z.string(),
    include_system_messages: z.boolean().optional(),
  }),
)

export const SDKControlMergeSessionDeltaResponseSchema = lazySchema(() =>
  z.object({
    merged_count: z.number(),
    skipped_count: z.number(),
    new_head_uuid: z.string().uuid().nullable(),
  }),
)
```

并把 `SDKControlMergeSessionDeltaRequestSchema()` 加入 `SDKControlRequestInnerSchema` union。

### 7.2 新增 SDK 类型

在 `entrypoints/sdk/controlTypes.ts` 生成或补充：

```ts
export type SDKControlMergeSessionDeltaRequest = {
  subtype: 'merge_session_delta'
  source_session_id: string
  target_session_id: string
  source_base_head_uuid: string | null
  target_parent_uuid: string | null
  task_id: string
  include_system_messages?: boolean
}

export type SDKControlMergeSessionDeltaResponse = {
  merged_count: number
  skipped_count: number
  new_head_uuid: string | null
}
```

### 7.3 新增 session storage helper

在 `utils/sessionStorage.ts` 新增两个内部能力：

```ts
export async function readSessionDelta(params: {
  sourceSessionId: UUID
  baseHeadUuid: UUID | null
  includeSystemMessages: boolean
}): Promise<Message[]> {
  const sourceFile = getTranscriptPathForSession(params.sourceSessionId)
  const { messages } = await loadTranscriptFile(sourceFile, {
    keepAllLeaves: true,
  })

  const baseIndex = params.baseHeadUuid
    ? messages.findIndex(message => message.uuid === params.baseHeadUuid)
    : -1

  if (params.baseHeadUuid && baseIndex < 0) {
    throw new Error(`Base head not found: ${params.baseHeadUuid}`)
  }

  return messages
    .slice(baseIndex + 1)
    .filter(message =>
      isMergeableTranscriptMessage(message, params.includeSystemMessages),
    )
}
```

```ts
export async function appendMergedTranscriptDelta(params: {
  targetSessionId: UUID
  targetParentUuid: UUID | null
  sourceSessionId: UUID
  sourceBaseHeadUuid: UUID | null
  taskId: string
  messages: Message[]
}): Promise<UUID | null> {
  validateTranscriptDelta(params.messages)

  const rewritten = params.messages.map(message => ({
    ...message,
    sessionId: params.targetSessionId,
    isSidechain: false,
    mergedFrom: {
      kind: 'parallel-task-delta',
      taskId: params.taskId,
      sourceSessionId: params.sourceSessionId,
      sourceBaseHeadUuid: params.sourceBaseHeadUuid,
      originalUuid: message.uuid,
    },
  }))

  const project = getProjectForSession(params.targetSessionId)
  await project.insertMessageChain(
    rewritten,
    false,
    undefined,
    params.targetParentUuid,
  )

  clearSessionMessagesCache(params.targetSessionId)

  const last = rewritten.findLast(isChainParticipant)
  return (last?.uuid as UUID | undefined) ?? params.targetParentUuid
}
```

实现要求：

- `getProjectForSession(targetSessionId)` 必须显式定位目标 session 文件。
- 不要依赖当前进程全局 `getSessionId()` 决定写入目标。
- 如果复用现有 `insertMessageChain()`，需要确保它盖章时使用 target session，而不是当前 session。
- 合并后清理 session message cache，避免后续 `recordTranscript()` 使用旧消息集合。
- `validateTranscriptDelta()` 至少校验 `tool_use` 与 `tool_result` 配对、UUID 不重复、compact boundary 处理合法。

### 7.4 新增 `mergeSessionDeltaFromTranscript`

在 `utils/sessionStorage.ts` 或临近模块组合读取和写入：

```ts
export async function mergeSessionDeltaFromTranscript(params: {
  sourceSessionId: UUID
  targetSessionId: UUID
  sourceBaseHeadUuid: UUID | null
  targetParentUuid: UUID | null
  taskId: string
  includeSystemMessages: boolean
}): Promise<{
  mergedCount: number
  skippedCount: number
  newHeadUuid: UUID | null
}> {
  const delta = await readSessionDelta({
    sourceSessionId: params.sourceSessionId,
    baseHeadUuid: params.sourceBaseHeadUuid,
    includeSystemMessages: params.includeSystemMessages,
  })

  const newHeadUuid = await appendMergedTranscriptDelta({
    targetSessionId: params.targetSessionId,
    targetParentUuid: params.targetParentUuid,
    sourceSessionId: params.sourceSessionId,
    sourceBaseHeadUuid: params.sourceBaseHeadUuid,
    taskId: params.taskId,
    messages: delta,
  })

  return {
    mergedCount: delta.length,
    skippedCount: 0,
    newHeadUuid,
  }
}
```

### 7.5 `print.ts` 处理 control request

在 `cli/print.ts` 的 `if (message.type === 'control_request')` 分支中增加：

```ts
} else if (message.request.subtype === 'merge_session_delta') {
  try {
    const result = await mergeSessionDeltaFromTranscript({
      sourceSessionId: message.request.source_session_id as UUID,
      targetSessionId: message.request.target_session_id as UUID,
      sourceBaseHeadUuid: message.request.source_base_head_uuid as UUID | null,
      targetParentUuid: message.request.target_parent_uuid as UUID | null,
      taskId: message.request.task_id,
      includeSystemMessages: message.request.include_system_messages ?? true,
    })

    sendControlResponseSuccess(message, {
      merged_count: result.mergedCount,
      skipped_count: result.skippedCount,
      new_head_uuid: result.newHeadUuid,
    })
  } catch (error) {
    sendControlResponseError(message, errorMessage(error))
  }
```

### 7.6 SDK wrapper 暴露能力

在 SDK 层暴露：

```ts
export async function mergeSessionDelta(params: {
  sourceSessionId: string
  targetSessionId: string
  sourceBaseHeadUuid: string | null
  targetParentUuid: string | null
  taskId: string
}): Promise<{
  mergedCount: number
  skippedCount: number
  newHeadUuid: string | null
}> {
  const response = await sendControlRequest({
    subtype: 'merge_session_delta',
    source_session_id: params.sourceSessionId,
    target_session_id: params.targetSessionId,
    source_base_head_uuid: params.sourceBaseHeadUuid,
    target_parent_uuid: params.targetParentUuid,
    task_id: params.taskId,
    include_system_messages: true,
  })

  return {
    mergedCount: response.merged_count,
    skippedCount: response.skipped_count,
    newHeadUuid: response.new_head_uuid,
  }
}
```

具体放置位置取决于现有 SDK transport 封装。如果 `Query` 实例已经持有 structured IO control 通道，也可以设计成：

```ts
await queryHandle.mergeSessionDelta(params)
```

但推荐先做独立函数，便于 `TaskScheduler` 在 worker 结束后调用。

## 8. 编排层完整示例

下面是宿主调度层的参考实现。代码强调职责边界，不是可直接复制的最终实现。

```ts
import {
  query,
  mergeSessionDelta,
  type PermissionMode,
  type SDKMessage,
  type SDKResultMessage,
  type SessionStore,
} from '@anthropic-ai/claude-agent-sdk'
import { randomUUID } from 'node:crypto'

type UUID = string

type ParallelTaskRequest = {
  id?: string
  prompt: string
  cwd: string
}

type TaskForkSnapshot = {
  taskId: UUID
  branchSessionId: UUID
  baseSessionId: UUID
  baseHeadUuid: UUID | null
  cwd: string
  permissionMode: PermissionMode
  allowedTools: string[]
  disallowedTools: string[]
  sessionStore?: SessionStore
}

type TaskResult = {
  taskId: UUID
  branchSessionId: UUID | null
  baseHeadUuid: UUID | null
  ok: boolean
  resultMessage?: SDKResultMessage
  merge?: {
    mergedCount: number
    skippedCount: number
    newHeadUuid: UUID | null
  }
  error?: string
}

type TaskHandle = {
  taskId: UUID
  done: Promise<TaskResult>
  cancel(): void
}

class SessionActor {
  private queue: Promise<unknown> = Promise.resolve()

  constructor(
    public readonly sessionId: UUID,
    private headUuid: UUID | null,
    public readonly cwd: string,
    public readonly sessionStore?: SessionStore,
  ) {}

  currentHead(): UUID | null {
    return this.headUuid
  }

  setHead(headUuid: UUID | null): void {
    this.headUuid = headUuid
  }

  forkForTask(req: ParallelTaskRequest, taskId: UUID): TaskForkSnapshot {
    return {
      taskId,
      branchSessionId: randomUUID(),
      baseSessionId: this.sessionId,
      baseHeadUuid: this.headUuid,
      cwd: req.cwd,
      permissionMode: 'default',
      allowedTools: [],
      disallowedTools: [],
      sessionStore: this.sessionStore,
    }
  }

  withMergeLock<T>(fn: (currentHead: UUID | null) => Promise<T>): Promise<T> {
    const next = this.queue.then(async () => fn(this.headUuid))
    this.queue = next.catch(() => {})
    return next
  }

  observeMainTurnMessage(message: SDKMessage): void {
    if ('uuid' in message && message.session_id === this.sessionId) {
      this.headUuid = message.uuid
    }
  }
}

class TurnRunner {
  constructor(private readonly sessionActor: SessionActor) {}

  async runForegroundTurn(prompt: string): Promise<void> {
    const hasHistory = this.sessionActor.currentHead() !== null
    const q = query({
      prompt,
      options: {
        cwd: this.sessionActor.cwd,
        sessionStore: this.sessionActor.sessionStore,
        ...(hasHistory
          ? { resume: this.sessionActor.sessionId }
          : { sessionId: this.sessionActor.sessionId }),
      },
    })

    for await (const message of q) {
      this.sessionActor.observeMainTurnMessage(message)
    }
  }
}

class EventMux {
  emitTaskEvent(event: {
    taskId: UUID
    branchSessionId: UUID
    baseSessionId: UUID
    baseHeadUuid: UUID | null
    seq: number
    message: SDKMessage
  }): void {
    void event
  }
}

class TaskWorkerPool {
  private readonly running = new Map<UUID, AbortController>()

  constructor(private readonly eventMux: EventMux) {}

  async run(snapshot: TaskForkSnapshot, prompt: string): Promise<TaskResult> {
    const abortController = new AbortController()
    this.running.set(snapshot.taskId, abortController)

    let seq = 0
    let resultMessage: SDKResultMessage | undefined

    try {
      const q = query({
        prompt,
        options: {
          cwd: snapshot.cwd,
          sessionId: snapshot.branchSessionId,
          resume: snapshot.baseSessionId,
          resumeSessionAt: snapshot.baseHeadUuid ?? undefined,
          forkSession: true,
          abortController,
          permissionMode: snapshot.permissionMode,
          allowedTools: snapshot.allowedTools,
          disallowedTools: snapshot.disallowedTools,
          sessionStore: snapshot.sessionStore,
          persistSession: true,
        },
      })

      for await (const message of q) {
        this.eventMux.emitTaskEvent({
          taskId: snapshot.taskId,
          branchSessionId: snapshot.branchSessionId,
          baseSessionId: snapshot.baseSessionId,
          baseHeadUuid: snapshot.baseHeadUuid,
          seq: seq++,
          message,
        })

        if (message.type === 'result') {
          resultMessage = message
        }
      }

      return {
        taskId: snapshot.taskId,
        branchSessionId: snapshot.branchSessionId,
        baseHeadUuid: snapshot.baseHeadUuid,
        ok: resultMessage !== undefined && !resultMessage.is_error,
        resultMessage,
      }
    } catch (error) {
      return {
        taskId: snapshot.taskId,
        branchSessionId: snapshot.branchSessionId,
        baseHeadUuid: snapshot.baseHeadUuid,
        ok: false,
        error: error instanceof Error ? error.message : String(error),
      }
    } finally {
      this.running.delete(snapshot.taskId)
    }
  }

  cancel(taskId: UUID): void {
    this.running.get(taskId)?.abort()
  }
}

class TaskScheduler {
  private readonly cancelled = new Set<UUID>()

  constructor(
    private readonly sessionActor: SessionActor,
    private readonly workerPool: TaskWorkerPool,
  ) {}

  submit(req: ParallelTaskRequest): TaskHandle {
    const taskId = req.id ?? randomUUID()

    const done = new Promise<TaskResult>((resolve, reject) => {
      queueMicrotask(() => {
        const snapshot = this.sessionActor.forkForTask(req, taskId)
        this.runForkAndMerge(snapshot, req.prompt).then(resolve, reject)
      })
    })

    return {
      taskId,
      done,
      cancel: () => this.cancel(taskId),
    }
  }

  private async runForkAndMerge(
    snapshot: TaskForkSnapshot,
    prompt: string,
  ): Promise<TaskResult> {
    if (this.cancelled.has(snapshot.taskId)) {
      return {
        taskId: snapshot.taskId,
        branchSessionId: null,
        baseHeadUuid: snapshot.baseHeadUuid,
        ok: false,
        error: 'cancelled',
      }
    }

    const result = await this.workerPool.run(snapshot, prompt)

    if (!result.branchSessionId) {
      return result
    }

    const merge = await this.sessionActor.withMergeLock(async currentHead => {
      const response = await mergeSessionDelta({
        sourceSessionId: snapshot.branchSessionId,
        targetSessionId: snapshot.baseSessionId,
        sourceBaseHeadUuid: snapshot.baseHeadUuid,
        targetParentUuid: currentHead,
        taskId: snapshot.taskId,
      })

      this.sessionActor.setHead(response.newHeadUuid)
      return response
    })

    return {
      ...result,
      merge,
    }
  }

  private cancel(taskId: UUID): void {
    this.cancelled.add(taskId)
    this.workerPool.cancel(taskId)
  }
}
```

## 9. 并发与一致性规则

### 9.1 主会话单写

所有会改变主会话 head 的操作必须串行：

- foreground user turn 结束后更新 head。
- task delta merge 后更新 head。
- compact、clear、resume 等会话级操作。

### 9.2 worker 不写主会话

worker 只能写：

- branch session transcript。
- sidechain transcript。
- task event log。

worker 不能写：

- 主 session transcript。
- 主会话 `currentHeadUuid`。
- 主会话 mutable message array。

### 9.3 base head 过期

如果任务 fork 于 H0，但完成时主会话已经到 H1，默认策略是把 delta 追加到 H1 后：

```text
H0 -> H1 -> task_delta_1 -> task_delta_2
```

同时每条合并后的消息保留：

```ts
mergedFrom: {
  kind: 'parallel-task-delta',
  taskId,
  sourceSessionId,
  sourceBaseHeadUuid,
  originalUuid,
}
```

后续可以基于这些 metadata 做 rebase、审计或自动修正。

### 9.4 权限隔离

每个 task 应有独立 permission context。权限 decision key 至少包含：

```text
sessionId / taskId / toolName / normalizedInput / scope
```

“本 session 允许”不能无条件扩散到所有并行任务。

### 9.5 取消

取消语义：

- `cancel(taskId)` 只 abort 对应 worker。
- foreground turn 的 interrupt 不应默认中断所有 task。
- session shutdown 可以级联 abort 所有关联 task。
- cancelled task 不应合并不完整 delta，除非后续明确支持 partial merge。

## 10. 持久化与恢复

### 10.1 主 transcript

主 transcript 记录：

- foreground turn 产生的真实消息。
- 已合并的 task delta rewritten copy。
- compact boundary。
- 必要的 system/status message。

### 10.2 branch transcript

branch transcript 记录任务原始轨迹：

- task prompt。
- assistant/tool_use/tool_result。
- result。
- 错误或中断状态。

branch transcript 不删除，作为审计和重试来源。

### 10.3 task metadata

建议持久化：

```ts
type TaskMetadata = {
  taskId: string
  status: 'queued' | 'running' | 'completed' | 'merge_failed' | 'cancelled'
  baseSessionId: string
  branchSessionId: string
  baseHeadUuid: string | null
  mergedHeadUuid?: string | null
  error?: string
  createdAt: number
  completedAt?: number
  mergedAt?: number
}
```

### 10.4 恢复策略

主会话 resume：

- 读取 target transcript 线性链。
- 已合并 delta 自然出现在主上下文。
- 对 `completed` 但 `merge_failed` 的任务重新入 merge queue。
- 对 `running` 但进程已消失的任务标记 orphaned 或尝试恢复 branch session。

## 11. 测试计划

### 11.1 单元测试

覆盖：

- `readSessionDelta()` 能从 base head 后提取 delta。
- base head 不存在时报错。
- `appendMergedTranscriptDelta()` 正确重写 target `sessionId`。
- `appendMergedTranscriptDelta()` 从传入 `targetParentUuid` 开始串联 `parentUuid`。
- `validateTranscriptDelta()` 拒绝不配对的 `tool_use` / `tool_result`。
- 合并后清理 session message cache。

### 11.2 集成测试

覆盖：

1. 两个任务同时从 H0 fork，A 先完成，B 后完成，主 transcript 仍为单链。
2. 任务运行期间 foreground turn 推进到 H1，任务完成后 delta 追加到 H1 后。
3. 任务执行工具调用，合并后主 transcript 保留 assistant tool_use 和 user tool_result。
4. branch transcript 保留原始 sessionId，target transcript 保留 rewritten delta。
5. `resume targetSessionId` 能读到合并后的任务增量。
6. `cancel(taskId)` 只影响该 task，不影响其他 task 或 foreground turn。
7. `merge_session_delta` control request 失败时返回 control error，`TaskHandle.done` 返回 `merge_failed` 信息。

### 11.3 并发压力测试

覆盖：

- 同一主 session 下连续提交 20 个任务。
- 任务以随机顺序完成。
- 每次 merge 后 target transcript 只有一个 leaf。
- 没有两个 merged delta 使用同一个 `parentUuid` 形成分叉。

## 12. 开发里程碑

### 阶段一：最小可用

目标：

- 每个 task 独立 `query()` 分支。
- 事件带 task 归属。
- worker 完成后通过 `merge_session_delta` 自动合并。
- 主 transcript 保持单链。

交付：

- `merge_session_delta` control schema/type。
- `readSessionDelta()`。
- `appendMergedTranscriptDelta()`。
- `TaskScheduler` / `TaskWorkerPool` / `EventMux` 最小实现。
- 单元测试和两个任务并发集成测试。

### 阶段二：恢复与补偿

目标：

- task metadata 持久化。
- 崩溃后恢复未 merge 的 completed task。
- branch transcript 审计和重试。

交付：

- task state store。
- resume 补偿流程。
- merge failed retry。

### 阶段三：rebase 与自动修正

目标：

- 检测 base head 过期。
- 对强上下文依赖任务触发 rebase 或修正 delta。
- 支持策略化 merge ordering。

交付：

- merge policy。
- rebase 检测。
- 自动修正任务或重新执行。

## 13. 验收标准

开发完成后必须满足：

1. 同一主会话可以并行运行多个用户任务。
2. `SessionActor` 没有调用 `query()`。
3. 每个 worker 使用独立 `sessionId`、`AbortController`、权限上下文。
4. 任务完成后自动合并真实增量 transcript。
5. 合并不是合成 user message。
6. 主 transcript 始终是线性 `parentUuid` 链。
7. target session resume 后能看到已合并 task delta。
8. 任意任务失败或 merge 失败不会破坏主 transcript。
9. 任务不按读/写分类；调度层不基于工具副作用做分流。
10. 关键路径有单元测试、集成测试和并发压力测试。

