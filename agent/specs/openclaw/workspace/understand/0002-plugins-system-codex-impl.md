# Codex 插件功能与实现机制理解

## 1. 总览

`extensions/codex` 是 OpenClaw 内置的 Codex 原生运行时插件。它不是只提供一个 `codex/*` 模型 provider，而是把 Codex app-server 接入为 OpenClaw 的一个 agent harness，同时补齐 provider/catalog、媒体理解、迁移、运行时 slash command、会话绑定、动态工具桥接、审批桥接、auth 桥接和 native hook relay。

从系统边界看，Codex 插件把“模型推理和原生 Codex 线程执行”交给 Codex app-server，但 OpenClaw 仍然拥有这些主流程能力：

- 聊天渠道、会话文件、会话选择、conversation binding。
- provider/model/auth/profile 解析和默认运行时选择。
- OpenClaw 动态工具、消息发送工具、心跳工具、媒体投递、cron、session/subagent/browser/gateway 等工具能力。
- 用户审批、工具进度、plan/reasoning/assistant event 回投。
- 可见 transcript mirror、context engine、bootstrap prompt、skill snapshot。
- fallback 策略、错误呈现、active run 控制、abort/interrupt。

核心实现入口：

- `extensions/codex/openclaw.plugin.json`：插件 manifest，声明 provider、contracts、activation、config schema、命令 alias。
- `extensions/codex/package.json`：发布包和运行时依赖，包含 `@openai/codex` managed app-server。
- `extensions/codex/index.ts`：runtime entry，调用 plugin API 注册各类能力。
- `extensions/codex/harness.ts`：注册 `codex` agent harness。
- `extensions/codex/src/app-server/run-attempt.ts`：一次 Codex app-server turn 的主体执行逻辑。
- `extensions/codex/src/app-server/thread-lifecycle.ts`：Codex thread start/resume、turn/start 参数构造。
- `extensions/codex/src/app-server/client.ts`：Codex app-server JSON-RPC client。
- `extensions/codex/provider.ts`、`extensions/codex/provider-catalog.ts`、`extensions/codex/provider-discovery.ts`：Codex provider 和模型目录。
- `extensions/codex/src/commands.ts`、`extensions/codex/src/command-handlers.ts`、`extensions/codex/src/conversation-binding.ts`、`extensions/codex/src/conversation-control.ts`：`/codex` 命令和绑定式 Codex 会话。
- `extensions/codex/media-understanding-provider.ts`：Codex image understanding provider。
- `extensions/codex/src/migration/provider.ts`：Codex CLI state/skills 迁移 provider。

## 2. Manifest 与包形态

### 2.1 Manifest 声明的控制面事实

`extensions/codex/openclaw.plugin.json` 中的关键字段：

- `id: "codex"`、`name: "Codex"`：插件身份。
- `providers: ["codex"]`：声明 Codex provider id。
- `contracts.mediaUnderstandingProviders: ["codex"]`：声明图片理解能力。
- `contracts.migrationProviders: ["codex"]`：声明迁移能力。
- `providerDiscoveryEntry: "./provider-discovery.ts"`：控制面可在不加载完整 runtime entry 的情况下发现 Codex provider/catalog。
- `syntheticAuthRefs: ["codex"]`、`nonSecretAuthMarkers: ["codex-app-server"]`：Codex provider 使用 synthetic auth marker，不直接暴露真实 secret。
- `activation.onStartup: false`：默认不在启动时无条件加载。
- `activation.onAgentHarnesses: ["codex"]`：当运行时需要 `codex` harness 时激活插件。
- `commandAliases`：把 `codex` 注册成 runtime slash command，CLI command 指向 `plugins`。

配置 schema 覆盖几组能力：

- `codexDynamicToolsProfile`：`native-first` 或 `openclaw-compat`，默认 `native-first`。
- `codexDynamicToolsExclude`：额外排除的 OpenClaw 动态工具名。
- `discovery.enabled`、`discovery.timeoutMs`：Codex app-server 模型发现开关和超时。
- `computerUse.*`：Codex Computer Use 插件和 MCP server 检测/安装配置。
- `appServer.*`：app-server 启动、transport、RPC timeout、审批策略、sandbox、reviewer、service tier、默认 workspace。

这些字段先进入插件控制面，用于配置 UI、doctor、provider discovery、activation planning；真正执行 Codex turn 时再由 runtime entry 读取当前 live plugin config。

### 2.2 package 依赖和发布身份

`extensions/codex/package.json` 声明：

- package 名称 `@openclaw/codex`。
- `openclaw.extensions: ["./index.ts"]`，作为 native OpenClaw plugin 入口。
- 依赖 `@openai/codex`，用于 managed Codex app-server binary。
- 依赖 `@mariozechner/pi-coding-agent`、`zod`、`ws`、`ajv` 等运行时支持。
- `openclaw.install` 和 `openclaw.compatibility` 描述插件安装/兼容元数据。

因此源码 checkout 和发布包都能通过插件发现机制加载同一个 runtime entry。

## 3. Runtime 注册入口

`extensions/codex/index.ts` 使用 `definePluginEntry()` 定义插件，并在 `register(api)` 中注册六类能力：

- `api.registerAgentHarness(createCodexAppServerAgentHarness(...))`
- `api.registerProvider(buildCodexProvider(...))`
- `api.registerMediaUnderstandingProvider(buildCodexMediaUnderstandingProvider(...))`
- `api.registerMigrationProvider(buildCodexMigrationProvider())`
- `api.registerCommand(createCodexCommand(...))`
- `api.on("inbound_claim", handleCodexConversationInboundClaim(...))`
- `api.onConversationBindingResolved?.(handleCodexConversationBindingResolved)`

这里有一个重要实现细节：`index.ts` 不是只使用启动时传入的 `api.pluginConfig`，还通过 `resolveLivePluginConfigObject(api, "codex")` 读取 live runtime config。这样 `/codex` 会话绑定和 inbound claim 可以跟随当前配置，而不是固定在插件初次加载时的配置快照。

## 4. 功能特性

### 4.1 Codex agent harness

`extensions/codex/harness.ts` 注册 `id: "codex"` 的 `AgentHarness`。

它的能力边界：

- `supports(ctx)` 默认只支持 provider id `codex`，返回 priority `100`。
- `deliveryDefaults.sourceVisibleReplies = "message_tool"`，即源聊天可见回复默认走 OpenClaw message tool。
- `runAttempt(params)` 懒加载 `extensions/codex/src/app-server/run-attempt.ts`，把一次 OpenClaw embedded run 转成一次 Codex app-server turn。
- `compact(params)` 懒加载 `compact.ts`，把 OpenClaw compaction 请求交给 Codex app-server。
- `reset(params)` 清除当前 OpenClaw session file 对应的 Codex app-server binding。
- `dispose()` 清理 shared Codex app-server client。

注意：OpenAI provider 默认转 Codex runtime 的逻辑不在 `harness.ts`，而在 core 的 harness policy 和 OpenAI routing 中实现。

### 4.2 Codex provider 与模型目录

`extensions/codex/provider.ts` 和 `provider-catalog.ts` 把 Codex 暴露成 provider：

- provider id 为 `codex`。
- auth method 是 custom wizard，实际 `resolveSyntheticAuth()` 返回 `codex-app-server` marker。
- fallback catalog 包含 `gpt-5.5`、`gpt-5.4-mini`、`gpt-5.2` 等模型。
- `buildCodexProviderCatalog()` 在 discovery 启用时通过 app-server `model/list` 拉取真实模型，失败则回退 fallback catalog。
- `resolveDynamicModel()` 可基于 fallback 模型能力创建动态模型条目。
- `resolveThinkingProfile()` 为现代模型提供 `off/minimal/low/medium/high/xhigh` 等 reasoning 档位。
- `resolveSystemPromptContribution()` 复用 GPT-5 prompt overlay 规则。

`extensions/codex/provider-discovery.ts` 是轻量 provider discovery entry。它能在控制面运行 catalog 发现，而不需要完整加载 Codex 插件 entry。

### 4.3 OpenAI 官方 provider 的默认 Codex runtime

Codex 插件还服务于 `openai/gpt-*` 这类用户常用模型引用。相关 core 实现在：

- `src/agents/harness/selection.ts`
- `src/agents/openai-codex-routing.ts`
- `src/auto-reply/reply/get-reply-run.ts`
- `src/auto-reply/reply/agent-runner-execution.ts`

关键策略：

- 当 provider 是 `openai` 且 base URL 是官方 OpenAI endpoint 时，隐式 PI runtime 会被策略转换为 `codex` runtime。
- 显式 `agentRuntime.id: "pi"` 会被尊重，不强制转 Codex。
- `OPENCLAW_AGENT_RUNTIME`、agent config、defaults、session pinned harness 会参与最终选择。
- `openai-codex` legacy provider 也会转到 Codex runtime。
- 对 `openai + codex runtime`，auth profile provider 接受列表会变成 `["openai-codex"]`，以匹配 Codex app-server 的原生 auth 形态。
- 插件 harness 一旦选中并执行失败，不会在同一 turn 自动回退到 PI。只有 `auto` 策略没有找到支持的 plugin harness 时才会选择 built-in PI。

因此用户推荐配置可以是 `agents.defaults.model: "openai/gpt-5.5"` 加 `agentRuntime.id: "codex"`，而不是使用 legacy `openai-codex/gpt-*` 或 `codex/gpt-*` 作为主要配置面。

### 4.4 Codex app-server transport

`extensions/codex/src/app-server/client.ts` 实现 JSON-RPC client。启动方式由 `config.ts` 解析：

- 默认 transport 是 `stdio`。
- 默认命令为 managed `codex app-server --listen stdio://`。
- 也支持 `websocket`，但必须提供 `appServer.url`。
- `OPENCLAW_CODEX_APP_SERVER_BIN` 和 `OPENCLAW_CODEX_APP_SERVER_ARGS` 可覆盖 binary 和 args。
- `appServer.mode` 默认 `yolo`，对应 `approvalPolicy: "never"`、`sandbox: "danger-full-access"`、`approvalsReviewer: "user"`。
- `guardian` 模式对应 `approvalPolicy: "on-request"`、`sandbox: "workspace-write"`、`approvalsReviewer: "auto_review"`。
- RPC request timeout 默认 60s，turn completion idle timeout 默认 60s。

managed binary 解析位于 `extensions/codex/src/app-server/managed-binary.ts`：

- 只有 `stdio + commandSource: "managed"` 会解析 managed binary。
- 在插件根、父目录、发布 dist 结构、`@openai/codex` package bin 中查找 `codex`/`codex.cmd`。
- 找不到时提示重新安装/更新 OpenClaw，或配置 `plugins.entries.codex.config.appServer.command` / `OPENCLAW_CODEX_APP_SERVER_BIN`。

`client.initialize()` 会发送 `initialize` 请求，带上 OpenClaw clientInfo 和 `experimentalApi: true`，并校验 Codex app-server 最低版本。

### 4.5 app-server 进程共享与隔离

`extensions/codex/src/app-server/shared-client.ts` 维护进程级 shared client：

- 使用 `globalThis` 上的 `openclaw.codexAppServerClientState` 保存 client/promise/key。
- key 由 transport、command、args、url、headers、env、clearEnv、authProfileId、agentDir 计算。
- key 改变会清理旧 client。
- 初始化时先解析 managed binary，再调用 auth bridge 注入隔离环境和登录信息。

`extensions/codex/src/app-server/auth-bridge.ts` 做两件事：

- 为 stdio Codex app-server 设置每个 agent 独立的 `CODEX_HOME` 和 `HOME`，目录位于 agent dir 下的 `codex-home` 和 `codex-home/home`。
- 根据 auth profile 生成 app-server `account/login/start` 参数。支持 API key 和 ChatGPT OAuth token；当使用 native ChatGPT profile 时，会清理继承的 `CODEX_API_KEY` / `OPENAI_API_KEY`，避免环境变量覆盖账号态。

如果没有显式 auth profile，stdio 场景会尝试从 app-server spawn env 中读取 `CODEX_API_KEY` 或 `OPENAI_API_KEY` 作为 fallback，但只有 app-server 当前需要 OpenAI auth 时才执行登录。

### 4.6 动态工具桥接

Codex app-server 有原生 shell/edit/apply_patch/update_plan 等工具；OpenClaw 也有自己的工具系统。Codex 插件用动态工具桥接二者：

- `run-attempt.ts` 调用 `createOpenClawCodingTools()` 创建 OpenClaw agent 工具集合。
- `dynamic-tool-profile.ts` 默认 `native-first`，排除 `read`、`write`、`edit`、`apply_patch`、`exec`、`process`、`update_plan`，避免和 Codex 原生工具重复。
- `codexDynamicToolsExclude` 可继续按名称排除工具，支持 `bash -> exec`、`apply-patch -> apply_patch` alias。
- `buildDynamicTools()` 会继续按模型能力、图片输入、tool allowlist、runtime plan 进行过滤和 normalize。
- `createCodexDynamicToolBridge()` 把 OpenClaw tool 转成 Codex `dynamicTools` spec：`name`、`description`、`inputSchema`。
- app-server 发起 `item/tool/call` request 时，bridge 调用对应 OpenClaw tool 的 `prepareArguments()` 和 `execute()`。
- 工具结果经过 OpenClaw before/after tool hook、tool result middleware、legacy Codex app-server tool result extension。
- telemetry 会记录 message tool 是否发送、发送文本/媒体、heartbeat 响应、cron add、tool media URL、audio-as-voice 等。
- 单次动态工具调用默认 30s timeout。超时会返回失败 tool response 给 Codex，并标记 timed out during tool execution。

这意味着 Codex 原生工具负责本地代码编辑和 shell 能力，OpenClaw 动态工具负责 OpenClaw 集成面，例如聊天消息发送、会话、媒体、cron、browser、subagent、gateway、heartbeat 等。

### 4.7 审批桥接和用户输入

Codex app-server 会通过 JSON-RPC request 向 OpenClaw 请求审批或用户输入：

- `item/commandExecution/requestApproval`
- `item/fileChange/requestApproval`
- `item/permissions/requestApproval`
- 其他包含 `requestApproval` 的 native request
- `item/tool/requestUserInput`
- `mcpServer/elicitation/request`

`run-attempt.ts` 的 request handler 分发到：

- `handleApprovalRequest()`：把 Codex 原生审批接到 OpenClaw 审批 callback/策略。
- `handleCodexAppServerElicitationRequest()`：处理 MCP elicitation。
- user input bridge：处理 app-server tool user input。
- 默认 client fallback 会 decline 审批、返回空答案或 decline elicitation，防止没有 handler 时误授权。

`appServer.mode: guardian` 会启用更保守的审批和 workspace-write sandbox；`yolo` 默认给 Codex 原生侧 full access，但 OpenClaw 动态工具仍受 OpenClaw 工具策略和 sandbox 上下文约束。

### 4.8 事件投影和 transcript mirror

Codex app-server 的通知由 `extensions/codex/src/app-server/event-projector.ts` 投影回 OpenClaw：

- `item/agentMessage/delta`：收集 assistant 文本，但中间 delta 不直接作为可见回复发出，最终选择最后 assistant item。
- `item/reasoning/summaryTextDelta`、`item/reasoning/textDelta`：转成 OpenClaw reasoning stream。
- `item/plan/delta`、`turn/plan/updated`：转成 OpenClaw plan update。
- `item/started`、`item/completed`：转成 item lifecycle、tool progress、tool result summary/output。
- `item/commandExecution/outputDelta`、`item/fileChange/outputDelta`：转成 shell/apply_patch 工具输出流，带截断保护。
- `item/autoApprovalReview/*`：转成 guardian event。
- `hook/started`、`hook/completed`：转成 native hook event。
- `thread/tokenUsage/updated`：归一化 token usage。
- `account/rateLimits/updated`：缓存 rate limit 信息。
- `turn/completed`：决定最终 turn 状态、错误、assistant text、usage、item lifecycle 和 replay metadata。

结果对象符合 OpenClaw embedded runner 的 `EmbeddedRunAttemptResult`，包含：

- `assistantTexts`、`lastAssistant`。
- `messagesSnapshot`。
- `didSendViaMessagingTool` 和 message tool telemetry。
- `heartbeatToolResponse`。
- `toolMediaUrls`、`toolAudioAsVoice`。
- `attemptUsage`。
- `promptError`、`aborted`、`timedOut`、`yieldDetected`。
- `agentHarnessResultClassification`。

`extensions/codex/src/app-server/transcript-mirror.ts` 把 Codex canonical thread 的关键消息镜像到 OpenClaw session transcript：

- 只 mirror `user` / `assistant` 角色。
- 每条 mirror message 带稳定 `mirrorIdentity`，例如 `${turnId}:prompt`、`${turnId}:assistant`。
- idempotency key 由 `idempotencyScope + mirrorIdentity` 组成，避免 retry 或跨 turn snapshot 重放造成重复写入。
- 写入前运行 `runAgentHarnessBeforeMessageWriteHook()`。
- 通过 session write lock 保护 session file，并发写安全。
- 写完触发 `emitSessionTranscriptUpdate()`。

因此 Codex thread 是原生执行的权威历史，而 OpenClaw transcript 是可见 UI、搜索、后续运行时切换和 session 管理所需的镜像。

### 4.9 Native hook relay

`run-attempt.ts` 会根据 app-server approval policy 创建 native hook relay：

- 使用 `registerNativeHookRelay()` 注册 provider `codex` 的 relay。
- relay id 由 agent/session 维度派生，带 TTL 和 abort signal。
- Codex app-server native hook 通过 command/request 回到 OpenClaw hook 体系。
- projector 接收 `hook/started`、`hook/completed` 通知并转成 OpenClaw agent event。

文档声明的 native hook 事件包括 prompt build、compaction、LLM input/output、tool call、message write、agent finalize/end 等。实现中 OpenClaw 也会在关键阶段主动运行 `runAgentHarnessBeforePromptBuildResult`、`runAgentHarnessLlmInputHook`、`runAgentHarnessLlmOutputHook`、`runAgentHarnessAgentEndHook`、compaction hooks、message write hook 等。

### 4.10 Codex Computer Use

`extensions/codex/src/app-server/computer-use.ts` 负责 Codex Computer Use 集成：

- `resolveCodexComputerUseConfig()` 读取 plugin config 和 env。
- 默认 disabled；如果配置 autoInstall、marketplace source/path/name，或 env 开关，则可自动 enabled。
- `readCodexComputerUseStatus()` 只检查状态。
- `ensureCodexComputerUse()` 在每次 Codex harness 启动 thread 前执行。如果启用但未 ready，会失败关闭本次 turn；如果 `autoInstall` 为 true 且安全条件满足，会尝试安装。
- `installCodexComputerUse()` 对应 `/codex computer-use install`。
- 检查流程包括启用 Codex plugins experimental feature、定位 marketplace、确认 `computer-use` plugin installed/enabled、确认 MCP server 可用和 tool 列表。
- 远程 marketplace install 会被认为不安全或不支持自动安装。

OpenClaw 只负责准备和检查 Computer Use plugin；真正的 MCP 调用和桌面控制仍由 Codex app-server 及其 plugin 处理。

### 4.11 媒体理解 provider

`extensions/codex/media-understanding-provider.ts` 注册 `id: "codex"` 的 image understanding provider：

- capability 为 `image`。
- 默认 image model 来自 fallback Codex model 中支持 image 的模型。
- `describeImages()` 会创建 isolated Codex app-server client，避免污染 shared conversation client。
- 先调用 `model/list` 确认目标模型存在且支持 image。
- 创建 ephemeral thread，`approvalPolicy: "on-request"`、`sandbox: "read-only"`、`dynamicTools: []`、`persistExtendedHistory: false`。
- developer instructions 明确它是 bounded image-understanding worker，不调用工具、不改文件、不追问。
- 发送 `turn/start`，输入是 text prompt 加 data URL images。
- app-server 如果请求审批、permissions 或 MCP elicitation，provider handler 一律 decline。
- collector 监听 assistant delta 和 `turn/completed`，返回最终描述文本。

这是一个独立、受限的 Codex app-server 使用场景，和主 agent harness thread 不共享 thread。

### 4.12 迁移 provider

`extensions/codex/src/migration/provider.ts` 注册 migration provider：

- `detect(ctx)` 调用 `discoverCodexSource()` 探测 Codex state。
- `plan(ctx)` 构造 Codex 迁移计划。
- `apply(ctx, plan)` 应用迁移。
- 描述是“Inventory and promote Codex CLI skills while keeping Codex native plugins and hooks explicit.”

它的定位是把 Codex CLI skills 等状态纳入 OpenClaw 可管理迁移流程，但 Codex native plugins/hooks 仍保持显式处理，避免隐藏兼容迁移影响 runtime。

## 5. Codex 与 OpenClaw 主回复流程

### 5.1 总调用链

一次普通聊天回复使用 Codex runtime 的主链路如下：

```text
用户消息 / channel inbound
  -> OpenClaw auto reply routing
  -> get-reply-run 解析 agent、model、provider、auth profile、agent harness policy
  -> agent-runner-execution 构造 embedded run params
  -> runEmbeddedPiAgent()
  -> selectAgentHarness()
  -> runAgentHarnessAttempt()
  -> Codex harness runAttempt()
  -> runCodexAppServerAttempt()
  -> Codex app-server thread/start 或 thread/resume
  -> Codex app-server turn/start
  -> notifications / requests 双向桥接
  -> EventProjector buildResult()
  -> mirror Codex transcript 到 OpenClaw session
  -> OpenClaw 根据 result 做可见回复、媒体投递、状态更新
```

关键点是 `runEmbeddedPiAgent()` 名称仍然带 PI 历史，但真实执行 backend 已经通过 `src/agents/pi-embedded-runner/run/backend.ts` 转到 `runAgentHarnessAttempt()`，再由 harness selection 选择 `pi` 或插件 harness。

### 5.2 provider/model/auth 解析阶段

`src/auto-reply/reply/get-reply-run.ts` 会：

- 从 agent config、defaults、message override 中解析模型引用。
- 根据 provider/model/config/session 计算 `agentHarnessPolicy`。
- 当 `openai + official base URL` 或 legacy `openai-codex` 命中 Codex runtime 默认策略时，把 runtime policy 解析成 `codex`。
- 通过 `listOpenAIAuthProfileProvidersForAgentRuntime()` 改变 auth profile 接受 provider 列表。
- 继续走 OpenClaw 的 auth profile 解析、上下文、session、tool policy 等主流程。

`src/auto-reply/reply/agent-runner-execution.ts` 在构造 embedded params 时：

- 根据 `agentRuntimeOverride` 或 harness policy 设置 `requestedAgentHarnessId` / `agentHarnessId`。
- 对 PI runtime 的特殊 OpenAI route 调用 `resolveOpenAIRuntimeProviderForPi()`；Codex runtime 不走这个 PI provider 重写。
- 把 message channel、sender、thread、reply delivery mode、tool callbacks、approval callbacks、images、workspace、sessionFile、runtimePlan 等完整传给 embedded runner。

### 5.3 harness selection

`src/agents/harness/selection.ts` 的规则：

- session pinned harness 优先。
- 显式 `agentRuntime.id` 优先。
- `OPENCLAW_AGENT_RUNTIME` 可覆盖。
- explicit plugin runtime 找不到时 fail closed。
- explicit plugin runtime 执行失败时不 fallback 到 PI。
- `auto` 会询问各 plugin harness 的 `supports(ctx)`，按 priority 选择；没有匹配才 fallback 到 built-in PI。
- `pi` 明确选择 built-in PI。

Codex harness 的 `supports()` 默认只接受 provider `codex`。OpenAI 官方 provider 默认 Codex runtime 是通过 policy 把 runtime 指向 `codex`，不是让 Codex harness 对所有 `openai` provider 自报支持。

### 5.4 Codex runAttempt 启动阶段

`runCodexAppServerAttempt()` 的启动阶段做这些准备：

- 读取 live plugin config，解析 app-server runtime options。
- 解析 workspace 和 sandbox context，准备 `sandboxSessionKey`。
- 创建 AbortController 并绑定上游 abort signal。
- 解析 agent dir、auth profile、startup binding。
- 处理 context engine bootstrap 和 prompt projection。
- 创建 OpenClaw dynamic tools 和 Codex dynamic tool bridge。
- 读取已有 session mirror history，必要时投影到 Codex prompt。
- 生成 developer instructions：OpenClaw runtime 说明、message tool 说明、prompt overlay、extra system prompt、skills snapshot、bootstrap context。
- 运行 before prompt build hook。
- 注册 native hook relay。
- 获取 shared Codex app-server client。
- 调用 `ensureCodexComputerUse()`。
- 调用 `startOrResumeThread()`。

app-server client 获取时还会：

- 解析 managed binary。
- 注入 per-agent `CODEX_HOME` / `HOME`。
- 解析和应用 Codex app-server auth profile。
- 发送 initialize 并校验版本。

### 5.5 thread start/resume

`extensions/codex/src/app-server/thread-lifecycle.ts` 管理 Codex thread lifecycle。

Binding 文件路径：

```text
<OpenClaw session file>.codex-app-server.json
```

binding schema 记录：

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

`startOrResumeThread()` 的逻辑：

- 计算当前 dynamic tools fingerprint。
- 读取已有 binding。
- 如果 binding 存在且 dynamic tools fingerprint 兼容，优先 `thread/resume`。
- resume 成功后更新 binding。
- resume 失败如果是 connection closed，抛出让上层重试 client；其他失败则 warn、清 binding、start 新 thread。
- 如果 dynamic tools 从非空变为空，会保留已有 binding 但启动 transient no-tool thread，避免破坏旧绑定。
- 其他 fingerprint 不兼容时清 binding 并新建 thread。

`thread/start` 参数包含：

- bare `model`，例如 `gpt-5.5`。
- 可选 `modelProvider`；native Codex auth profile 下 `openai` / `openai-codex` 会归一为 undefined，让 app-server 使用原生账号态。
- `cwd`
- `approvalPolicy`
- `approvalsReviewer`
- `sandbox`
- `serviceTier`
- `serviceName: "OpenClaw"`
- `developerInstructions`
- `dynamicTools`
- `experimentalRawEvents: true`
- `persistExtendedHistory: true`

`turn/start` 参数包含：

- `threadId`
- text + image data URL input
- `cwd`
- `approvalPolicy`
- turn 级 `sandboxPolicy`
- `model`
- `serviceTier`
- reasoning `effort`
- `collaborationMode`

heartbeat turn 会额外把 heartbeat-specific developer instruction 注入 Codex collaboration mode。

### 5.6 turn/start 后的双向 RPC

`runCodexAppServerAttempt()` 发送 `turn/start` 后进入双向桥接状态：

- 记录 `turnId`。
- 创建 user input bridge。
- 创建 `CodexAppServerEventProjector`。
- 回放 turn/start 前已收到的 pending notifications。
- 通过 `setActiveEmbeddedRun()` 暴露 active run handle，支持 queue message、streaming 判断、compacting 判断、cancel/abort。
- 安装 terminal idle watch、overall timeout、turn completion idle watch。
- 上游 abort 时 best-effort 发送 `turn/interrupt`。

notification handler 负责：

- 过滤当前 thread/turn 的通知。
- 将 Codex delta、tool、plan、reasoning、usage、hook、guardian、turn completed 等事件交给 projector。
- 遇到 retryable error 时解除 completion watch，非 retryable error 时触发完成等待。
- 收到 `turn/completed` 后 resolve completion。

request handler 负责：

- `account/chatgptAuthTokens/refresh`：刷新 OAuth token。
- `mcpServer/elicitation/request`：转给 OpenClaw elicitation handler。
- `item/tool/requestUserInput`：转给 user input bridge。
- 审批 request：转给 approval bridge。
- `item/tool/call`：调用 OpenClaw dynamic tool bridge。

OpenClaw 响应完 turn-scoped request 后，会启动 completion watch，等待 app-server 后续 `turn/completed`；如果静默超时，会 interrupt 并释放 lane。

### 5.7 完成阶段

turn 完成或超时后：

- projector 生成 `EmbeddedRunAttemptResult`。
- 记录 trajectory。
- 调用 `mirrorCodexAppServerTranscript()` 写 OpenClaw session mirror。
- 如果有 terminal assistant text，触发 assistant text event。
- 运行 lifecycle end/error event。
- finalize context engine。
- 运行 LLM output hook 和 agent end hook。
- 返回 result 给 embedded runner。

`finally` 中会：

- 补发 lifecycle error backstop。
- flush trajectory。
- flush steering queue。
- cancel user input。
- 清理 timers。
- 卸载 request/notification handlers。
- unregister native hook relay。
- 移除 abort listener。
- clear active embedded run。

## 6. `/codex` 命令和 conversation binding

`extensions/codex/src/commands.ts` 注册 reserved runtime slash command `/codex`。handler 懒加载 `command-handlers.ts`。

主要子命令：

- `/codex status`：读取 app-server 状态探针。
- `/codex models`：列出 Codex app-server 模型。
- `/codex threads [filter]`：列出 Codex threads。
- `/codex resume <thread-id>`：把当前 OpenClaw session 绑定到已有 Codex thread。
- `/codex bind [thread-id] [--cwd <path>] [--model <model>] [--provider <provider>]`：创建或附加 Codex thread，并请求 OpenClaw conversation binding。
- `/codex detach` / `/codex unbind`：解除 conversation binding 并清理 binding 文件。
- `/codex binding`：显示当前绑定详情。
- `/codex stop`：interrupt 当前 active Codex turn。
- `/codex steer <message>`：向当前 active Codex turn 发送 steer input。
- `/codex model <model>`：修改绑定 thread 的模型。
- `/codex fast [on|off|status]`：切换 service tier `fast` / `flex`。
- `/codex permissions [default|yolo|status]`：切换绑定 thread 的审批/sandbox 策略。
- `/codex compact`：调用 app-server `thread/compact/start`。
- `/codex review`：调用 app-server `review/start`。
- `/codex diagnostics ...`：上传诊断反馈，有确认、冷却和 scope 限制。
- `/codex computer-use status|install ...`：检查或安装 Computer Use plugin。
- `/codex mcp`：列出 MCP server status。
- `/codex skills`：列出 Codex skills。
- `/codex account`：读取账号和 rate limits。

### 6.1 bind 模式和普通 agent harness 模式的区别

普通 agent harness 模式：

- OpenClaw auto reply 主流程决定何时调用 Codex。
- Codex thread 与 OpenClaw session file 通过 `.codex-app-server.json` 隐式绑定。
- 回复结果仍走 embedded runner result，再由 OpenClaw 做投递。

conversation binding 模式：

- `/codex bind` 通过 `ctx.requestConversationBinding()` 把当前聊天会话显式绑定到 Codex thread。
- 后续 inbound message 会被 `handleCodexConversationInboundClaim()` claim，不再进入普通 auto reply agent flow。
- prompt 来自 inbound event 的 `bodyForAgent` 或 `content`。
- `enqueueBoundTurn()` 保证同一 session file 的 bound turn 串行执行。
- bound turn 直接通过 app-server `turn/start` 运行，并用 `createCodexConversationTurnCollector()` 收集回复。
- `trackCodexConversationActiveTurn()` 记录 active turn，供 `/codex stop` 和 `/codex steer` 控制。
- 如果 conversation binding 被拒绝，`handleCodexConversationBindingResolved()` 会清除 Codex binding 文件。

这使 Codex 可以作为“原生线程被绑定到某个 OpenClaw 聊天”的交互模式存在，而不只是模型 provider。

## 7. 错误、超时和失败语义

Codex 插件显式处理多类失败：

- app-server initialize/version 不满足：直接报错，提示升级或移除 custom command override。
- managed binary 找不到：提示重新安装/更新或配置 custom binary。
- startup connection closed：`run-attempt.ts` 最多重试 3 次，重试前清理 shared client。
- `thread/resume` 非连接关闭失败：清 binding 并 start 新 thread。
- `turn/start` 失败：记录 lifecycle、trajectory、LLM output、agent end hook 后抛错；rate limit 错误会格式化为更友好的 usage-limit 信息。
- dynamic tool call 超时：返回失败 tool response 给 Codex，并标记工具超时。
- app-server server request 没有 handler：默认拒绝审批、拒绝 elicitation、tool call 返回失败，防止 silent allow。
- terminal idle timeout / overall timeout：mark timeout，best-effort interrupt。
- Computer Use 启用但未 ready：在 thread start 前失败，不让 Codex 在缺失能力状态下继续。
- plugin harness 已选中后失败：core 不自动降级 PI，避免同一用户请求在不同 runtime 下重复执行副作用。

## 8. 关键设计取舍

### 8.1 Codex owns native execution, OpenClaw owns product workflow

Codex 插件让 Codex app-server 维护原生 thread、resume、compaction、原生工具和 raw event，但 OpenClaw 仍维护聊天产品工作流。这个分工避免把 OpenClaw 变成 Codex 的薄 wrapper：

- OpenClaw 的 channel/message/session/tool/approval 体系继续统一适配 Discord、Telegram、CLI、Gateway 等入口。
- Codex 的原生执行能力不被 PI 抽象压扁，可以保留 native thread、native hook、native tools、Codex Computer Use。

### 8.2 Manifest-first + runtime lazy load

Codex manifest 声明 provider、contracts、activation、config schema，让控制面可静态发现。真正 heavy 的 app-server client、dynamic tools、Computer Use、provider live catalog 都在 runtime 需要时懒加载或 best-effort 执行。

### 8.3 Native-first dynamic tool profile

默认排除文件和 shell 类 OpenClaw 工具，交给 Codex 原生实现；保留 OpenClaw 集成工具。这减少工具重复和 schema 冲突，也让 Codex 使用自身最擅长的代码编辑执行路径。

### 8.4 Transcript mirror 而非 ownership 反转

Codex thread 是 canonical。OpenClaw transcript 只 mirror 足够的信息用于 UI、搜索、session 管理和运行时切换。mirror 通过 stable idempotency key 避免重复写入。

### 8.5 Explicit fail-closed

一旦用户或策略选择 Codex runtime，OpenClaw 不在失败后自动改跑 PI。这是为了避免工具、消息发送、代码修改等副作用在不同 runtime 下重复执行。

## 9. 源码索引

- Manifest：`extensions/codex/openclaw.plugin.json`
- Package metadata：`extensions/codex/package.json`
- Runtime entry：`extensions/codex/index.ts`
- Agent harness：`extensions/codex/harness.ts`
- Codex provider：`extensions/codex/provider.ts`
- Provider catalog：`extensions/codex/provider-catalog.ts`
- Provider discovery：`extensions/codex/provider-discovery.ts`
- Prompt overlay：`extensions/codex/prompt-overlay.ts`
- Main app-server attempt：`extensions/codex/src/app-server/run-attempt.ts`
- Thread lifecycle：`extensions/codex/src/app-server/thread-lifecycle.ts`
- App-server config：`extensions/codex/src/app-server/config.ts`
- Shared client：`extensions/codex/src/app-server/shared-client.ts`
- JSON-RPC client：`extensions/codex/src/app-server/client.ts`
- Managed binary lookup：`extensions/codex/src/app-server/managed-binary.ts`
- Auth bridge：`extensions/codex/src/app-server/auth-bridge.ts`
- Dynamic tools bridge：`extensions/codex/src/app-server/dynamic-tools.ts`
- Dynamic tools profile：`extensions/codex/src/app-server/dynamic-tool-profile.ts`
- Event projector：`extensions/codex/src/app-server/event-projector.ts`
- Session binding：`extensions/codex/src/app-server/session-binding.ts`
- Transcript mirror：`extensions/codex/src/app-server/transcript-mirror.ts`
- Computer Use：`extensions/codex/src/app-server/computer-use.ts`
- Media understanding：`extensions/codex/media-understanding-provider.ts`
- `/codex` command entry：`extensions/codex/src/commands.ts`
- `/codex` command handlers：`extensions/codex/src/command-handlers.ts`
- Conversation binding：`extensions/codex/src/conversation-binding.ts`
- Conversation control：`extensions/codex/src/conversation-control.ts`
- Migration provider：`extensions/codex/src/migration/provider.ts`
- Core harness registry：`src/agents/harness/registry.ts`
- Core harness selection：`src/agents/harness/selection.ts`
- OpenAI/Codex routing：`src/agents/openai-codex-routing.ts`
- Reply setup：`src/auto-reply/reply/get-reply-run.ts`
- Reply execution：`src/auto-reply/reply/agent-runner-execution.ts`
- Embedded runner backend handoff：`src/agents/pi-embedded-runner/run/backend.ts`

## 10. 一句话结论

Codex 插件是 OpenClaw 插件系统中“深运行时集成”的代表：manifest 让它在控制面以 provider/contract/command/harness 被发现，runtime entry 把它注册进 OpenClaw 主流程，而 app-server bridge 把 Codex 原生线程执行、工具调用、审批、事件流和 transcript mirror 全部映射回 OpenClaw 的统一 agent harness 协议。
