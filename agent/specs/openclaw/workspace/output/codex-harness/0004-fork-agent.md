# OpenClaw 中的 Forked Agent 上下文

## 摘要

OpenClaw 当前是在 OpenClaw session 层实现 forked agent context。
代码中没有一个独立的 `ForkedAgent` runtime class。创建 forked child 的方式是通过
native subagent，并传入 `context: "fork"`：

```ts
sessions_spawn({
  runtime: "subagent",
  context: "fork",
  task: "...",
});
```

这会在 child run 启动前，把 requester 的 OpenClaw session transcript fork 成
一个 child session。这个 fork 是否能进一步映射到 provider-native 或
harness-native 的 thread fork，取决于具体 runtime 路径。

## 当前 Native Subagent 语义

对 native subagent 来说，`context: "fork"` 的含义是：

1. 解析 requester 的 session store entry。
2. 确认 requester transcript 可用。
3. 估算 parent context 大小；如果过大则跳过 fork。
4. 读取 parent JSONL transcript。
5. 将 active branch 复制到新的 child transcript 文件。
6. 写入 child session metadata，包括新的 `sessionId`、`sessionFile`，并标记
   `forkedFromParent: true`。
7. 基于 child session 启动 child run。

重要限制：

- `context: "fork"` 只支持 `runtime: "subagent"`。
- `runtime: "acp"` 会拒绝 `context: "fork"`。
- 当前不支持 cross-agent fork；requester agent 和 target agent 必须相同。
- 如果 parent context 超过 fork 阈值，OpenClaw 会回退到 isolated context，并在
  spawn note 中报告这个 fallback。

## Codex Harness 路径

bundled Codex app-server harness 会引入第二层状态：

```text
OpenClaw session transcript
Codex app-server native thread
```

当前 Codex harness 下的 `context: "fork"` 行为是：

```text
parent OpenClaw transcript
  -> child OpenClaw transcript
  -> assembled/projected context
  -> Codex thread/start or thread/resume
  -> turn/start
```

OpenClaw 当前不会调用 Codex `thread/fork`。Codex 集成在正常 run startup 中只使用
`thread/start`、`thread/resume` 和 `turn/start`。OpenClaw fork 后的 context 会被
投影到 Codex prompt 中，通常渲染成 `OpenClaw assembled context for this turn`
上下文块。

因此：

- OpenClaw 的 child transcript 会被 fork。
- Codex native thread 不会被 OpenClaw fork。
- Codex 通过 prompt projection 看到 fork 后的 OpenClaw context。
- Codex 仍然拥有并管理自己的 native thread history 和 compaction 行为。

## ACP / External Harness 路径

ACP harness session 不支持通过 `context: "fork"` fork。

下面的调用会被明确拒绝：

```ts
sessions_spawn({
  runtime: "acp",
  context: "fork",
  task: "...",
});
```

ACP harness session 是运行在 host 上的 external runtime。它的 spawn API 不携带
`context` 字段，OpenClaw 将 `runtime: "acp"` 视为 isolated harness work，并使用
其独立的 delivery 和 binding model。

## 可能的 Codex Native Thread Fork

未来可以将 Codex native `thread/fork` 作为 best-effort enhancement 接入。

期望流程：

```text
parent OpenClaw transcript
  -> child OpenClaw transcript

parent Codex thread binding
  -> Codex thread/fork
  -> child Codex thread binding

child turn/start runs on child Codex thread
```

实现草图：

1. 仍然先执行现有的 OpenClaw transcript fork，这是 canonical first step。
2. Codex harness 为 child session 启动 run 时，检查 child transcript header 中的
   `parentSession`。
3. 从 `<parentSession>.codex-app-server.json` 读取 parent Codex binding。
4. 如果 parent binding 存在且兼容，则调用 Codex `thread/fork`。
5. 将返回的 forked thread id 写入 child session 的 Codex binding。
6. 在 forked Codex thread 上运行 `turn/start`。
7. 如果任何步骤不支持或失败，则回退到当前的 `thread/start + projected context`
   路径。

这应该做成 feature-gated 或透明 fallback 的增强，因为 OpenClaw 当前的 typed
protocol layer 还没有接入 Codex `thread/fork` request contract。

风险和待确认问题：

- 必须从 upstream Codex app-server protocol/types 确认 `thread/fork` 的准确
  request shape。
- forked Codex thread 可能在第一个 forked/resumed turn 上不接受新的
  developer instructions。OpenClaw 依赖 developer instructions 注入 workspace
  bootstrap、context-engine additions 和 runtime guidance。
- 如果 native thread fork 成功，OpenClaw 应避免再把同一段完整 mirrored history
  投影进 prompt，否则 child 可能看到重复 context。
- 使用 parent Codex thread 作为 fork source 前，必须确认 dynamic tools、cwd、
  auth profile、model provider 和 sandbox settings 兼容。

## Native PI Agent 路径

native PI agent 路径是当前最干净的 forked agent 语义实现。

在 PI 路径中，没有额外的 Codex native thread state：

```text
parent OpenClaw/PI transcript JSONL
  -> fork active branch
  -> child OpenClaw/PI transcript JSONL
  -> PI runner assembles context directly from child transcript
```

这意味着 fork 后的 transcript 就是 runtime 的真实 history source。不存在第二套
harness thread 需要 fork 或 reconciliation。

如果目标是“child agent 继承 parent conversation branch 并独立继续运行”，PI 路径
是当前最直接、语义最完整的路径。

## KV Cache 影响

OpenClaw transcript fork 不会复制模型 KV cache。

Native PI 路径：

- fork JSONL transcript。
- 为 child request 重新 assemble prompt context。
- 发起一次新的 model request。
- 不会继承 in-flight 或之前已经构建好的 model KV cache。
- 如果 provider 支持 prompt cache，并且 prompt prefix 足够 byte-stable，则可能
  受益于 provider prompt caching。

当前 Codex harness 路径：

- 同样不会通过 OpenClaw 复制 KV cache。
- 使用投影后的 OpenClaw context 作为 Codex prompt。
- 任何 cache reuse 都依赖 Codex/app-server/OpenAI backend prompt caching。
- projection text 可能改变 byte layout，因此不保证 cache reuse。

潜在 Codex native `thread/fork` 路径：

- 如果 Codex app-server 和 backend 将 forked thread 映射到可复用的内部 cache
  state，则可能提供更强的 native thread continuity。
- OpenClaw 仍然不会直接操作 KV cache。
- cache 行为仍是 Codex/backend contract，不是 OpenClaw guarantee。

实践结论：

```text
OpenClaw transcript fork: yes
model KV cache fork/share: no
provider/backend prompt-cache reuse: possible, runtime-dependent
Codex native thread fork: possible future enhancement, not current behavior
```
