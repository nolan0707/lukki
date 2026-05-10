# ContextEngine 与 Codex Harness 架构边界详解

本文从架构师视角梳理 RuntimePlan、ContextEngine、Codex harness、Codex native thread、OpenClaw mirror、binding 文件和 compaction 的职责边界。重点回答这些问题：

- RuntimePlan 如何约定 ContextEngine 的调用？
- ContextEngine 的 `bootstrap`、`assemble`、`afterTurn`、`maintain`、`compact` 分别负责什么？
- Codex native thread 和 OpenClaw mirror 是什么关系？
- ContextEngine compact 和 Codex native compaction 各自压缩什么？
- 为什么 ContextEngine 不应直接改 Codex native thread？
- Codex harness 如果直接实现 V2，应如何安排 ContextEngine 生命周期？

## 1. 核心结论

RuntimePlan 当前不直接调用 ContextEngine，也不拥有 ContextEngine lifecycle。RuntimePlan 的职责是提供 provider/model/auth/prompt/tools/transcript/delivery/outcome/transport/observability 等 prepared runtime facts；ContextEngine 通过 `EmbeddedRunAttemptParams.contextEngine` 单独传入 harness，由 harness 在合适的生命周期阶段调用。

Codex harness 下存在两套会话事实：

- Codex app-server 的 canonical native thread。
- OpenClaw 的 session transcript / mirror / ContextEngine store。

这两套状态不是同一个东西。Codex native thread 只能通过 Codex app-server API 影响，例如 `thread/start`、`thread/resume`、`turn/start`、`thread/compact/start`。ContextEngine 处理的是 OpenClaw 可观测、可维护的 session/mirror/context store；它不能直接删除、重排、覆盖 Codex native thread 内的历史 item。

因此压缩也不是单一动作：

- Codex native thread 超限：触发 Codex native compaction。
- OpenClaw mirror / ContextEngine store / projection 超限：触发 ContextEngine compact / maintenance。
- Codex harness 中如果 ContextEngine 声明 `info.ownsCompaction === true`，当前实现会先压 OpenClaw context/mirror，再跑 Codex native compaction；两者是并列双层压缩，不是把 ContextEngine 摘要交给 Codex native compaction 使用。

## 2. 术语和对象

### 2.1 RuntimePlan

RuntimePlan 是一次 agent runtime 执行前准备出的事实集合。它不是一个执行器，也不是 ContextEngine coordinator。

它约定的事实包括：

- `resolvedRef`：解析后的 provider/model/runtime 引用。
- `auth`：auth profile forwarding、provider auth 选择等。
- `prompt`：provider/system prompt contribution、text transforms。
- `tools`：工具 schema normalize、诊断、prepared metadata snapshot。
- `transcript`：transcript policy。
- `delivery`：visible reply、silent payload、followup fallback route。
- `outcome`：结果分类。
- `transport`：extra params。
- `observability`：resolvedRef、provider、modelId、harnessId、authProfileId 等观测字段。

RuntimePlan 与 ContextEngine 的关系是：RuntimePlan 给 ContextEngine 调用提供输入事实，例如工具集合、模型 id、prompt overlay、auth profile、observability；但 ContextEngine 的调用时机仍由 harness lifecycle 决定。

### 2.2 ContextEngine

ContextEngine 是 OpenClaw 的上下文管理插件接口。它围绕 OpenClaw session 运行，输入通常是：

- `sessionId`
- `sessionKey`
- `sessionFile`
- OpenClaw `AgentMessage[]`
- token budget / current token estimate
- runtimeContext

它可以维护索引、摘要、长期记忆、检索上下文、transcript rewrite 请求等。它不是下游 runtime 的 native thread editor。

### 2.3 Codex native thread

Codex native thread 是 Codex app-server 内部的 canonical 会话线程。它由 `thread/start` 创建，由 `thread/resume` 续用，由 `turn/start` 添加 turn。里面包含 Codex 自己的 user/assistant item、reasoning/plan item、native shell/apply_patch/MCP 工具记录、approval 状态、native compaction 状态等。

OpenClaw 不直接读写这条 thread 的内部数据库。OpenClaw 只能通过 app-server 暴露的正式 API 和 notifications 与它交互。

### 2.4 OpenClaw mirror

OpenClaw mirror 是 OpenClaw 对 Codex turn 的可观测投影。它用于：

- channel history
- search
- `/new` / `/reset`
- future runtime switching
- ContextEngine assemble / afterTurn
- trajectory / diagnostics

mirror 不是 Codex canonical history。它只包含 app-server notification 和 OpenClaw adapter 能观察到的信息，例如用户 prompt、最终 assistant text、部分 reasoning/plan、tool telemetry 等。

### 2.5 Binding 文件

Codex binding 文件是 OpenClaw session file 到 Codex native thread 的映射。

路径：

```text
<sessionFile>.codex-app-server.json
```

映射关系：

```text
OpenClaw session file -> Codex app-server threadId
```

典型字段：

- `schemaVersion`
- `threadId`
- `sessionFile`
- `cwd`
- `authProfileId`
- `model`
- `modelProvider`
- `approvalPolicy`
- `sandbox`
- `serviceTier`
- `dynamicToolsFingerprint`
- `createdAt`
- `updatedAt`

它不是所有 harness 的通用机制。PI/default harness 不需要外部 native thread binding；Codex 需要，因为 canonical thread 在 Codex app-server；ACP/acpx 走 ACP session/task/binding metadata，而不是 Codex 的 sidecar 文件。

## 3. ContextEngine 方法职责

ContextEngine 拆分多个方法，是为了让不同阶段的副作用、失败语义和数据所有权清晰。

### 3.1 bootstrap

职责：初始化 engine 对某个 OpenClaw session 的认识。

典型用途：

- 首次接管已有 session file。
- 导入历史 transcript。
- 建立 engine 私有索引或 metadata。
- 做会话级兼容初始化。

边界：

- 不负责组装当前模型 prompt。
- 不负责写入当前 turn 的结果。
- 失败不应让整条历史会话完全不可用。

### 3.2 assemble

职责：在模型调用前，基于 OpenClaw messages、当前 prompt、token budget、available tools、model 等事实，返回本次模型应该看到的上下文。

典型用途：

- 挑选保留哪些历史消息。
- 注入检索到的长期记忆。
- 插入 summary。
- 根据 token budget 裁剪上下文。
- 根据 available tools 对 prompt guidance 做对齐。

边界：

- 它是 pre-turn read/assembly 阶段。
- 不提交当前 turn 结果。
- 不改写 Codex native thread。
- 对 Codex harness，assemble 结果还要经过 Codex-specific projection，不能原样当作 native history 替换。

### 3.3 afterTurn

职责：一次 turn 完成后，基于完整 turn snapshot 做 post-turn 生命周期处理。

典型用途：

- 持久化本轮新消息。
- 更新检索索引。
- 根据 `prePromptMessageCount` 找出本轮新增内容。
- 记录 prompt cache / token usage 观测。
- 触发 engine 自己的后处理。

边界：

- 它处理已完成事实。
- 如果 engine 实现了 `afterTurn`，harness helper 优先调用它。
- 如果没有 `afterTurn`，helper fallback 到 `ingestBatch` 或逐条 `ingest`。
- 不应做无 runtime 授权的 transcript surgery；如需 rewrite，应通过 runtime-owned helper。

### 3.4 maintain

职责：维护 OpenClaw session transcript / ContextEngine store。

触发原因通常是：

- `bootstrap`
- `turn`
- `compaction`

典型用途：

- 执行 deferred compaction debt。
- 修复索引漂移。
- 合并或清理 engine state。
- 通过 `runtimeContext.rewriteTranscriptEntries()` 请求安全 transcript rewrite。

边界：

- 它不是模型调用前的上下文组装；那是 `assemble`。
- 它不是当前 turn ingestion 的主入口；那是 `afterTurn` 或 `ingestBatch` / `ingest`。
- 它维护 OpenClaw-owned transcript/context，不改 Codex canonical native thread。

### 3.5 compact

职责：压缩 OpenClaw session/context，以减少 token usage 或维护上下文预算。

典型用途：

- 生成摘要。
- prune old turns。
- rewrite/rotate OpenClaw transcript。
- 更新 ContextEngine checkpoint。

边界：

- 压缩对象是 OpenClaw session/mirror/context store。
- 对 Codex harness，它不会把摘要写回 Codex native thread。
- 如果 downstream runtime 另有 canonical thread，例如 Codex，仍需要 downstream native compaction 处理那一层状态。

## 4. RuntimePlan 和 ContextEngine 的约定方式

RuntimePlan 不直接包含 `contextEngine` 字段。合理的约定方式是：

```text
RuntimePlan:
  准备运行事实

Harness:
  在生命周期中消费 RuntimePlan 和 params.contextEngine
  调 ContextEngine helper
```

### 4.1 prepare 阶段应消费的 RuntimePlan facts

如果 Codex harness 直接实现 AgentHarnessV2，`prepare()` 应固定这些事实：

- `runtimePlan.auth.forwardedAuthProfileId`
- `runtimePlan.prompt.resolveSystemPromptContribution(...)`
- `runtimePlan.tools.normalize(...)`
- `runtimePlan.observability.resolvedRef`
- `runtimePlan.observability.harnessId`
- `runtimePlan.transcript.resolvePolicy(...)`

然后用这些事实调用 ContextEngine：

- `bootstrapHarnessContextEngine(...)`
- `assembleHarnessContextEngine(...)`

### 4.2 start/send 不应重算 ContextEngine 输入

`start()` 应只使用 prepared facts 启动 app-server、auth bridge、start/resume thread。

`send()` 应只执行 `turn/start`、notification loop、tool/approval request loop、timeout/abort/steer。不要在 hot path 里重新 broad discovery provider/tools/context facts。

### 4.3 resolveOutcome 阶段 finalize ContextEngine

`resolveOutcome()` 或等价的 post-turn 阶段应：

- 基于 final mirror/messages snapshot 调 `finalizeHarnessContextEngineTurn(...)`。
- 用 usage 构造 `buildHarnessContextEngineRuntimeContextFromUsage(...)`。
- 成功时触发 `maintain(reason="turn")`。
- 使用 `runtimePlan.observability` 记录 lifecycle 输出。

## 5. Codex Harness 中 ContextEngine 的当前调用

当前 Codex harness 仍注册 V1 `AgentHarness`，由 core 适配到 V2 lifecycle 执行。但内部 `runCodexAppServerAttempt()` 已经按阶段组织了 ContextEngine。

当前时序：

```text
runCodexAppServerAttempt()
  -> resolve workspace/sandbox/auth/runtime facts
  -> build dynamic tools
  -> read mirrored session history
  -> bootstrapHarnessContextEngine()
  -> assembleHarnessContextEngine()
  -> projectContextEngineAssemblyForCodex()
  -> thread/start or thread/resume
  -> turn/start
  -> collect notifications/server requests
  -> mirror transcript
  -> finalizeHarnessContextEngineTurn()
```

其中 `projectContextEngineAssemblyForCodex(...)` 是关键适配点：ContextEngine 返回通用 assembled messages 和 system prompt addition；Codex harness 再把它投影成 Codex app-server 可接受的：

- `promptText`
- `developerInstructionAddition`
- `prePromptMessageCount`

原因是 Codex app-server 拥有 native thread，OpenClaw 不能把 assembled messages 当作 replacement history 写入 Codex thread。

## 6. Codex native thread 的产生和续用

Codex native thread 在 app-server 收到 `thread/start` 时产生。

### 6.1 普通 Codex harness turn

第一次某个 OpenClaw session 选择 Codex harness 时：

```text
OpenClaw agent turn
  -> select codex harness
  -> runCodexAppServerAttempt()
  -> startOrResumeThread()
  -> no binding
  -> app-server thread/start
  -> response.thread.id
  -> write <sessionFile>.codex-app-server.json
```

后续同一个 session：

```text
read <sessionFile>.codex-app-server.json
  -> threadId
  -> app-server thread/resume
```

如果 resume 失败、binding 无效、dynamic tools fingerprint 不兼容，Codex harness 可能清理 binding 并新建 thread。

### 6.2 `/codex bind` / `/codex resume`

`/codex bind` 未指定 thread id 时会 `thread/start` 创建新 Codex native thread。

`/codex resume <threadId>` 或 bind 指定 thread id 时会 `thread/resume` attach 到已有 native thread。

然后同样写 binding 文件，把 OpenClaw session file 映射到 Codex thread id。

## 7. 不改 native thread 的含义和依据

“不改 native thread”不是说 OpenClaw 不影响 Codex。OpenClaw 会通过 app-server API 影响当前 turn 输入和 thread 配置。

允许：

- `thread/start`
- `thread/resume`
- `turn/start`
- `turn/steer`
- `turn/interrupt`
- `thread/compact/start`
- 通过 `developerInstructions` / prompt projection 影响当前 turn。
- 通过 dynamic tools 让 Codex 请求 OpenClaw 执行工具。

不允许：

- 直接删除 Codex native thread 里的历史 item。
- 直接重排 Codex native thread。
- 把 ContextEngine compact 后的 OpenClaw transcript 覆盖回 Codex thread。
- 改写 Codex native shell/apply_patch/MCP tool result。
- 修改 Codex native compaction retained/dropped set。

依据是能力和合同边界：

- Codex app-server owns native thread history、native tool continuation、native compaction。
- OpenClaw 当前只拥有 mirror 和 adapter-level projection。
- ContextEngine 接口没有 Codex thread item id、native retained context、native compaction summary、replace-context API 等安全改写所需的参数。

## 8. OpenClaw Mirror 如何影响 Codex Native Thread

OpenClaw mirror 修改后不会直接影响已有 Codex native thread 历史。它只影响后续 turn 输入。

路径如下：

```text
OpenClaw mirror / ContextEngine store 被 compact 或维护
  -> 下一次 Codex harness turn
  -> read mirrored session history
  -> ContextEngine assemble()
  -> projectContextEngineAssemblyForCodex()
  -> turn/start 当前输入
  -> Codex 把这次输入作为新 turn 写入 native thread
```

所以 mirror 对 Codex 的影响是 append/projection 影响，不是 mutation 影响。

## 9. Assemble 与 Codex 内部上下文的冲突风险

存在重复风险。

如果 Codex native thread 已经有：

```text
A
B
C
D
```

而 ContextEngine compact 后在下一次 turn 里注入：

```text
Summary of A/B/C/D
```

那么 Codex 可能同时看到原文 A/B/C/D 和摘要，产生重复上下文。

当前实现降低风险的方式是：

- 不把 ContextEngine assembled messages 当作 native thread replacement。
- 通过 Codex-specific projection 注入当前 turn。
- 只有 active non-legacy ContextEngine 才参与 plugin harness assemble。
- dynamic tool fingerprint 不兼容时新建 thread 或 transient no-tool thread。
- turn 后以 OpenClaw mirror 可观测事实 finalize ContextEngine。

但这不能完全消除重复。根因是 OpenClaw 当前不知道 Codex native compaction 具体保留/删除了哪些 item，也没有 replace-context API。

更严谨的策略应按 thread 状态区分：

```text
resume existing Codex native thread:
  默认不注入完整会话摘要。
  只注入外部记忆、长期偏好、检索型补充、或明确缺失的上下文。

new Codex native thread / binding lost / runtime switch recovery:
  可以用 OpenClaw mirror projection 恢复历史。
```

## 10. ContextEngine Compact 何时被调用

当前实际调用点主要有四类。

### 10.1 PI/default runner timeout recovery

当 LLM 超时且 prompt token 使用率较高时，PI runner 会调用 `contextEngine.compact(...)`，然后重试 prompt。

目的：避免高上下文压力导致反复 timeout。

### 10.2 PI/default runner context overflow recovery

当 promptError 或 assistant error 被识别为 context overflow 时，PI runner 会调用 `contextEngine.compact(...)`。成功后 adopt compacted transcript，再运行 `runContextEngineMaintenance(reason="compaction")`。

这是 OpenClaw-owned model loop 的运行中溢出恢复路径。

### 10.3 Queued/manual compaction

queued compaction lifecycle 会在 manual 或 budget-oriented compaction 触发时调用 `contextEngine.compact(...)`。

典型来源：

- 用户手动 compaction。
- 自动/队列化 compaction。
- session maintenance 触发。

成功后可能做 transcript rotation、checkpoint，并运行 maintenance。

### 10.4 CLI turn 前预算压缩

CLI compaction lifecycle 会根据 session token snapshot 或 preemptive estimate 判断是否超预算，必要时调用 `contextEngine.compact(...)`。

成功后同样运行 `runContextEngineMaintenance(reason="compaction")`。

### 10.5 Codex harness compact 特例

Codex harness 的 compaction 实现不同：

```text
if active ContextEngine exists and info.ownsCompaction:
  contextEngine.compact(...)
  runHarnessContextEngineMaintenance(reason="compaction")
  codex thread/compact/start
else:
  codex thread/compact/start
```

也就是说，普通 Codex compaction 默认是 Codex native compaction；只有 ContextEngine 声明自己 owns compaction 时，才先压 OpenClaw context/mirror。

## 11. 两种 Codex 压缩路径的差异

### 11.1 直接 Codex native compaction

对象：

- Codex app-server canonical native thread。

效果：

- 缩短或整理 Codex 自己下次模型调用会看到的 native thread。
- 保留 Codex 对 native tools、approval、MCP、compaction 状态的一致性。
- OpenClaw mirror / ContextEngine store 不一定变小。
- 下一次 ContextEngine assemble 仍可能从较大的 mirror 投影较多历史。

适合：

- Codex thread 自己超上下文。
- `/codex compact`。
- ContextEngine 不声明 `ownsCompaction`。

### 11.2 ContextEngine compact + Codex native compaction

对象：

- 第一层：OpenClaw mirror / ContextEngine store / session transcript。
- 第二层：Codex app-server canonical native thread。

效果：

- OpenClaw 后续 projection 变短或更摘要化。
- Codex native thread 也被 app-server 自己压缩。
- 成本更高，失败组合更复杂。
- 两边摘要可能不完全一致。

重要边界：ContextEngine compact 的产物不会作为输入改变当次 Codex native compaction。当前实现只是：

```text
contextEngine.compact()
  -> 更新 OpenClaw context/mirror
  -> maintenance
  -> app-server thread/compact/start({ threadId })
```

Codex native compaction 不消费 ContextEngine 的 `summary`、`tokensAfter`、`firstKeptEntryId` 或 rewritten transcript。

## 12. 单 Turn 中上下文溢出时应调用谁

取决于谁拥有该层上下文。

### 12.1 OpenClaw-owned model loop

PI/default harness 中，OpenClaw 构造 prompt 和 canonical transcript。上下文溢出时应调用 ContextEngine compact，然后重试。

### 12.2 Codex-owned native thread

Codex harness 中，如果溢出来自 Codex native thread，应优先走 Codex native compaction。ContextEngine 不应直接改 Codex thread。

如果溢出来自 OpenClaw 额外投影的 context，则应减少 OpenClaw projection，可能调用 ContextEngine compact 或调整 projection 策略。

### 12.3 推荐决策

```text
Codex canonical native thread 超限
  -> Codex native compaction

OpenClaw mirror / projection 超限
  -> ContextEngine compact / maintenance

两边都可能造成压力
  -> ContextEngine ownsCompaction 时先压 OpenClaw，再跑 Codex native compaction
```

## 13. 对 Codex Harness V2 的建议

如果 Codex harness 直接实现 AgentHarnessV2，应把 ContextEngine 生命周期显式拆到 V2 阶段。

### 13.1 prepare

职责：

- 读取 RuntimePlan facts。
- 解析 workspace/sandbox/auth/profile。
- 构建并 normalize dynamic tools。
- 读取 mirror history。
- 调 `bootstrapHarnessContextEngine(...)`。
- 调 `assembleHarnessContextEngine(...)`。
- 调 `projectContextEngineAssemblyForCodex(...)`。
- 保存 `promptText`、`developerInstructions`、`prePromptMessageCount`、tool fingerprint。

### 13.2 start

职责：

- 获取 app-server client。
- 做 auth bridge。
- ensure Computer Use。
- 注册 native hook relay。
- `thread/resume` 或 `thread/start`。
- 写 binding。

不应重新调用 ContextEngine assemble。

### 13.3 send

职责：

- `turn/start`。
- 处理 notifications。
- 处理 dynamic tool calls。
- 处理 approval/user input/elicitation。
- 管理 timeout、abort、steer。

不应 broad rediscover tools/provider/context。

### 13.4 resolveOutcome

职责：

- mirror transcript。
- 调 `finalizeHarnessContextEngineTurn(...)`。
- 运行 `llm_output` / `agent_end` 等 lifecycle hooks。
- 使用 RuntimePlan observability 输出诊断事实。

### 13.5 cleanup

职责：

- unregister notification/request handlers。
- unregister native hook relay。
- flush trajectory。
- clear active embedded run。
- cancel pending input/steering。
- remove abort listeners。

不应关闭 shared app-server client；shared client 仍应由 key 变化或 harness `dispose()` 管理。

## 14. 架构原则

这套设计应遵循以下原则：

1. Prepared runtime facts 早生成、晚消费；不要在 hot path 重新 broad discovery。
2. RuntimePlan 不拥有 ContextEngine lifecycle；harness 拥有调用时机。
3. ContextEngine 管 OpenClaw session/mirror/context store。
4. Codex app-server 管 Codex canonical native thread。
5. Mirror 影响 Codex 的方式是后续 turn projection，不是历史 mutation。
6. Codex native compaction 和 ContextEngine compact 是双层压缩，不是上下游摘要传递。
7. 只有 app-server 暴露明确 replace-context / retained-context / lineage API 后，OpenClaw 才能更强地对齐 ContextEngine 与 Codex native history。

## 15. 未来改进方向

当前最大风险是重复上下文和摘要漂移。可考虑的改进：

- Codex projection 策略区分 existing native thread resume 与 new thread recovery。
- Existing thread resume 时，默认只注入外部/长期/检索型上下文，不注入完整 mirror 摘要。
- ContextEngine assemble 增加 context source/lineage metadata，标记内容来自 mirror、external memory、retrieval、summary。
- Codex app-server 暴露 native compaction summary、retained item ids、dropped item ids。
- Codex app-server 暴露 safe replace-context 或 context overlay API。
- Binding 文件记录上次 ContextEngine projection watermark，避免重复注入同一段 mirror summary。

在这些能力出现前，最稳妥的边界仍是：Codex native thread 由 Codex 压缩，OpenClaw mirror/context 由 ContextEngine 压缩，两者通过后续 turn projection 松耦合。
