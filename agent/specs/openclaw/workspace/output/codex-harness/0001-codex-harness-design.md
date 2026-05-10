# Codex Harness 设计说明

本文以 `extensions/codex` 为样本，梳理 OpenClaw Codex 插件的实现机制、运行链路、关键状态、扩展边界和方案取舍。这里的 Codex 指 native Codex app-server 路径，不是 ACP/acpx 的 `runtime: "acp", agentId: "codex"` 路径。

## 1. 定位

`extensions/codex` 是一个 native OpenClaw 插件。它把 Codex app-server 接入为 OpenClaw 的 agent harness，同时注册 provider/catalog、media understanding、migration provider、`/codex` command、conversation binding hook 和 native hook relay。

核心边界是：

- Codex app-server 拥有原生模型循环、native thread、native shell/patch/MCP 工具、native compaction 和内部 OpenAI 请求构造。
- OpenClaw 仍拥有 channel routing、session file、agent/runtime selection、auth profile store、OpenClaw dynamic tools、message tool、approvals、context engine、transcript mirror、trajectory、diagnostics 和最终 channel delivery。
- Codex 插件是二者之间的 adapter：把 OpenClaw embedded run 映射成 Codex `thread/start|resume` + `turn/start`，再把 app-server notifications 和 server requests 投影回 OpenClaw runtime。

主要源码入口：

- `extensions/codex/openclaw.plugin.json`
- `extensions/codex/package.json`
- `extensions/codex/index.ts`
- `extensions/codex/harness.ts`
- `extensions/codex/provider.ts`
- `extensions/codex/provider-catalog.ts`
- `extensions/codex/provider-discovery.ts`
- `extensions/codex/media-understanding-provider.ts`
- `extensions/codex/src/app-server/run-attempt.ts`
- `extensions/codex/src/app-server/thread-lifecycle.ts`
- `extensions/codex/src/app-server/client.ts`
- `extensions/codex/src/app-server/shared-client.ts`
- `extensions/codex/src/app-server/auth-bridge.ts`
- `extensions/codex/src/app-server/dynamic-tools.ts`
- `extensions/codex/src/app-server/approval-bridge.ts`
- `extensions/codex/src/app-server/native-hook-relay.ts`
- `extensions/codex/src/commands.ts`
- `extensions/codex/src/command-handlers.ts`
- `extensions/codex/src/conversation-binding.ts`
- `extensions/codex/src/conversation-control.ts`

相关用户文档和运行约束：

- `docs/plugins/codex-harness.md`
- `docs/tools/acp-agents.md`
- `docs/tools/acp-agents-setup.md`
- `docs/providers/openai.md`
- `docs/plugins/sdk-agent-harness.md`

## 2. 插件形态和控制面

### 2.1 Manifest

`extensions/codex/openclaw.plugin.json` 是不执行插件代码即可读取的控制面事实。关键声明包括：

- `id: "codex"`：插件 canonical id。
- `providers: ["codex"]`：插件拥有 virtual Codex provider。
- `contracts.mediaUnderstandingProviders: ["codex"]`：注册 Codex image understanding provider。
- `contracts.migrationProviders: ["codex"]`：注册 Codex migration provider。
- `providerDiscoveryEntry: "./provider-discovery.ts"`：轻量 provider discovery 入口，用于 model catalog/control plane。
- `syntheticAuthRefs: ["codex"]` 和 `nonSecretAuthMarkers: ["codex-app-server"]`：Codex provider 使用 synthetic auth marker，而不是把真实 secret 暴露成 model provider API key。
- `activation.onStartup: false`：不靠启动时副作用变可用。
- `activation.onAgentHarnesses: ["codex"]`：当 runtime/harness selection 需要 `codex` 时激活。
- `commandAliases` 中注册 `codex` runtime slash command。

配置 schema 分四块：

- `codexDynamicToolsProfile` / `codexDynamicToolsExclude`：控制 OpenClaw dynamic tools 暴露给 Codex 的工具集。
- `discovery`：控制 app-server model discovery。
- `computerUse`：控制 Codex Computer Use marketplace/plugin/MCP server 准备逻辑。
- `appServer`：控制 transport、command、args、url、headers、timeout、approval policy、sandbox、reviewer、service tier、default workspace。

这些字段的设计遵循插件边界：控制面只声明静态或可廉价解析的事实，真正 runtime 行为留给 `index.ts` 注册的插件代码。

### 2.2 Package

`extensions/codex/package.json` 发布为 `@openclaw/codex`，`openclaw.extensions` 指向 `./index.ts`。运行时依赖里最关键的是：

- `@openai/codex`：managed Codex app-server binary 来源。
- `ws`：WebSocket transport。
- `zod`：plugin config runtime parser。

因此本地源码插件和发布后的 npm 插件共享同一套 entry 与 runtime 文件。

## 3. Runtime 注册入口

`extensions/codex/index.ts` 使用 `definePluginEntry()` 注册插件，并在 `register(api)` 中挂载能力：

1. `api.registerAgentHarness(createCodexAppServerAgentHarness(...))`
2. `api.registerProvider(buildCodexProvider(...))`
3. `api.registerMediaUnderstandingProvider(buildCodexMediaUnderstandingProvider(...))`
4. `api.registerMigrationProvider(buildCodexMigrationProvider())`
5. `api.registerCommand(createCodexCommand(...))`
6. `api.on("inbound_claim", handleCodexConversationInboundClaim(...))`
7. `api.onConversationBindingResolved?.(handleCodexConversationBindingResolved)`

实现细节：入口会构造 `resolveCurrentPluginConfig()`，优先通过 `api.runtime.config.current()` 读取 live config，再回退到 `api.pluginConfig`。这保证 `/codex` conversation binding、inbound claim 等长生命周期逻辑不会固定在插件初次加载时的配置快照上。

## 4. Agent Harness

`extensions/codex/harness.ts` 返回 `AgentHarness`：

- `id` 默认为 `codex`。
- `label` 默认为 `Codex agent harness`。
- `deliveryDefaults.sourceVisibleReplies = "message_tool"`，使 Codex 源聊天 turn 默认通过 OpenClaw message tool 产生可见回复。
- `supports(ctx)` 默认只支持 provider id `codex`，命中时 priority `100`。
- `runAttempt(params)` 懒加载 `src/app-server/run-attempt.ts`，执行一次 Codex app-server turn。
- `compact(params)` 懒加载 `src/app-server/compact.ts`，把 compaction 委托给 Codex app-server。
- `reset(params)` 清理当前 session file 旁边的 Codex binding 文件。
- `dispose()` 清理 shared app-server client。

需要注意：`harness.ts` 本身只声明 provider 支持，不负责 OpenAI 官方模型默认选 Codex runtime 的策略。`openai/gpt-*` 走 Codex runtime 的路由由 core harness selection 与 OpenAI routing 实现；插件只提供可被选中的 harness 能力。

## 5. Provider 和模型目录

Codex provider 是 virtual provider。它不是传统 HTTP provider，而是让 OpenClaw model catalog 能看到 Codex app-server 可用模型，并给 runtime selection/auth/profile 提供统一 surface。

### 5.1 Provider plugin

`extensions/codex/provider.ts` 的 `buildCodexProvider()` 返回 `ProviderPlugin`：

- `id: "codex"`，`label: "Codex"`。
- `auth` 是 custom wizard，`run()` 返回空 profiles 和默认模型 `codex/gpt-5.5` 形态的默认 ref。
- `catalog.order = "late"`：catalog 尽量晚解析。
- `staticCatalog` 使用 fallback model list。
- `resolveDynamicModel(modelId)` 可按 model id 生成 runtime model definition。
- `resolveSyntheticAuth()` 返回 `{ apiKey: "codex-app-server", source: "codex-app-server", mode: "token" }`。
- `resolveThinkingProfile()` 暴露 reasoning 档位。
- `resolveSystemPromptContribution()` 接入 Codex/GPT prompt overlay。
- `isModernModelRef()` 标记现代 Codex 模型。

`extensions/codex/provider-catalog.ts` 定义 fallback catalog：

- `gpt-5.5`
- `gpt-5.4-mini`
- `gpt-5.2`

这些 fallback 模型默认带 text/image input、reasoning efforts、较大的 context/max token 配置和 `openai-codex-responses` API family。

### 5.2 Live model discovery

`buildCodexProviderCatalog()` 的策略：

1. 读取 `plugins.entries.codex.config.discovery`。
2. 解析 app-server start options。
3. 如果 discovery 未禁用，且不是测试环境下的默认跳过模式，则调用 `listCodexAppServerModels()`。
4. 分页请求 app-server `model/list`，过滤 hidden model。
5. 失败或超时时 debug log，然后回退 fallback catalog。

`provider-discovery.ts` 提供轻量 discovery entry，让控制面在需要 model list 时不必加载完整 plugin runtime。

## 6. App-server 连接层

### 6.1 Config 解析

`extensions/codex/src/app-server/config.ts` 负责把 plugin config 和 env 解析成 runtime options。

默认值：

- transport: `stdio`
- command: `codex`
- args: `["app-server", "--listen", "stdio://"]`
- request timeout: `60000`
- turn completion idle timeout: `60000`
- mode: `yolo`

`mode` 是便捷 preset：

- `yolo` -> `approvalPolicy: "never"`、`sandbox: "danger-full-access"`、`approvalsReviewer: "user"`
- `guardian` -> `approvalPolicy: "on-request"`、`sandbox: "workspace-write"`、`approvalsReviewer: "auto_review"`

也可以直接覆盖 `approvalPolicy`、`sandbox`、`approvalsReviewer`。`websocket` transport 必须配置 `appServer.url`，可附加 `authToken` 和 headers。

### 6.2 Managed binary

`extensions/codex/src/app-server/managed-binary.ts` 只处理 `stdio` 且 `commandSource === "managed"` 的情况。它会在插件根、父级目录、dist 结构和 `@openai/codex` package bin 中查找 `codex` 或 `codex.cmd`。

找不到 managed binary 时，错误会提示：

- 重新安装或更新 OpenClaw。
- 源码 checkout 下运行 install。
- 配置 `plugins.entries.codex.config.appServer.command` 或 `OPENCLAW_CODEX_APP_SERVER_BIN` 使用自定义 Codex binary。

### 6.3 Transport

`client.ts` 抽象 JSON-RPC client；底层 transport 有两种：

- `transport-stdio.ts`：通过 `spawn()` 启动 app-server，本地 stdio JSON line 通信。
- `transport-websocket.ts`：通过 `ws` 连接远端 app-server，WebSocket frame 被转换成和 stdio 一样的 line stream。

stdio transport 细节：

- Windows 下使用 `openclaw/plugin-sdk/windows-spawn` 解析可执行文件。
- 会合并 base env 与 options.env。
- 支持 `clearEnv` 删除指定环境变量。
- 非 Windows 使用 detached process group，关闭时优先 kill process group。

WebSocket transport 细节：

- `Authorization: Bearer <authToken>` 由 config 注入。
- initialize 可能在 socket open 前写入，因此 transport 会缓存 pending frames，open 后再发送。

### 6.4 JSON-RPC client

`CodexAppServerClient` 的职责：

- `initialize()` 发送 `initialize`，clientInfo 为 OpenClaw，并开启 `experimentalApi`。
- 校验 app-server 版本满足最低版本，版本从 userAgent 的 leading product 里解析。
- `request()` 管理 request id、pending map、timeout、abort signal。
- `notify()` 发送 notification。
- `addRequestHandler()` 处理 app-server 发给 OpenClaw 的 server request，例如 dynamic tool call、approval request、token refresh、elicitation。
- `addNotificationHandler()` 处理 app-server notification。
- 解析 stdout JSON line；如果遇到多行 JSON，会在 bounded buffer 内尝试拼接。
- stderr tail 仅保留 bounded preview，并做 token/api key 等 redaction。
- close 时拒绝所有 pending requests，并关闭 transport。

默认 server request response 是 fail-closed：

- 未注册的 `item/tool/call` 返回失败文本。
- approval request 默认 decline。
- permissions request 默认空权限。
- elicitation request 默认 decline。

## 7. Shared client、隔离和认证

### 7.1 Shared client

`extensions/codex/src/app-server/shared-client.ts` 用 `globalThis` 保存 shared client state：

- `client`
- `promise`
- `key`

key 来自 app-server start options、auth profile id、agent dir 等。key 不同会清理旧 client。这样同一 agent/auth/app-server 配置下，多次 model discovery、command 和 run attempt 可以复用一个 app-server 进程。

也有 `createIsolatedCodexAppServerClient()` 用于一次性任务，例如 media understanding，避免污染 shared client。

### 7.2 每 agent 的 Codex home

`auth-bridge.ts` 会为本地 stdio app-server 设置隔离环境：

- `CODEX_HOME = <agentDir>/codex-home`
- `HOME = <agentDir>/codex-home/home`

这样 app-server 不会默认读取操作者个人 `~/.codex`、native thread、skills、plugins、hooks 或 `$HOME/.agents/skills`。这是 native Codex runtime 和 OpenClaw agent 隔离的关键。

### 7.3 Auth profile bridge

认证优先级大致为：

1. OpenClaw 传入的 `authProfileId`。
2. 按 agent/config 中 `openai-codex` provider 的 auth profile order 解析出的 profile。
3. app-server 隔离 home 中已有账号态。
4. stdio 本地启动时，如果 app-server 仍需要 OpenAI auth，尝试 `CODEX_API_KEY` / `OPENAI_API_KEY`。

OpenClaw auth profile 可转成 app-server `account/login/start` 参数：

- API key profile -> `{ type: "apiKey", apiKey }`
- OAuth/token profile -> `{ type: "chatgptAuthTokens", accessToken, chatgptAccountId, chatgptPlanType }`

如果当前选择的是 ChatGPT/Codex subscription-style profile，插件会清理 child env 中的 `CODEX_API_KEY` / `OPENAI_API_KEY`，避免 inherited env key 意外改变 app-server 账号和计费路径。

app-server 如需刷新 ChatGPT token，会发 `account/chatgptAuthTokens/refresh` server request；`run-attempt.ts` 将其路由到 `refreshCodexAppServerAuthTokens()`。

## 8. Embedded run 主链路

一次 Codex harness turn 的主体在 `extensions/codex/src/app-server/run-attempt.ts` 的 `runCodexAppServerAttempt()`。

### 8.1 入口准备

主要准备步骤：

1. 读取 plugin config，解析 app-server runtime options。
2. 解析 workspaceDir，并创建目录。
3. 根据 OpenClaw sandbox context 计算 effective workspace。
4. 建立 `AbortController`，监听上游 abort。
5. 解析 session/agent id 和 agent dir。
6. 读取已有 Codex binding 文件。
7. 解析 startup auth profile id。
8. 构造 dynamic tools。
9. 读取 mirrored session history。
10. 如启用 context engine，则 bootstrap/assemble context engine。
11. 构造 developer instructions 和当前 prompt text。
12. 运行 `before_prompt_build` 类 hook，允许 OpenClaw 插件改写 prompt/developer instructions。
13. 创建 trajectory recorder。

### 8.2 Developer instructions

`thread-lifecycle.ts` 的 `buildDeveloperInstructions()` 会拼接：

- OpenClaw runtime 固定说明：运行在 OpenClaw 内、使用 OpenClaw dynamic tools 处理 messaging/cron/sessions/media/gateway/nodes 等集成。
- Codex/GPT prompt overlay。
- `params.extraSystemPrompt`。
- skills snapshot prompt。

`run-attempt.ts` 还会额外把 OpenClaw bootstrap/persona 文件投影到 developer instructions：

- Codex 自己处理 `AGENTS.md` project doc。
- OpenClaw 将 `SOUL.md`、`IDENTITY.md`、`USER.md`、`TOOLS.md`、`BOOTSTRAP.md`、`MEMORY.md`、`HEARTBEAT.md` 等按稳定顺序转入 developer instructions。

如果 context engine active，OpenClaw 会把 assembled context 投影为：

- 当前 turn prompt text。
- developer instruction addition。
- pre-prompt message count。

Codex 仍拥有 canonical native thread history，OpenClaw 的 history 主要用于 mirror、context projection 和未来切换 runtime 时的上下文补偿。

### 8.3 Thread start/resume

`thread-lifecycle.ts` 的 `startOrResumeThread()` 决定是否复用 app-server thread。

本地 binding 文件路径是：

```text
<sessionFile>.codex-app-server.json
```

binding schema 包含：

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

复用策略：

- 如果 binding 有 `threadId` 且 dynamic tools fingerprint 兼容，则调用 app-server `thread/resume`。
- 如果 fingerprint 不兼容，通常清理 binding 并新建 thread。
- 特殊情况：上次有工具、这次 dynamic tools 为空，会启动 transient no-tool thread，但保留原 binding。
- resume 失败且不是 app-server connection closed，会 warn 后清理 binding 并新建 thread。
- connection closed 会向上抛出，由 startup retry 逻辑重启 shared client。

新建 thread 调用 `thread/start`，resume 调用 `thread/resume`。参数包括：

- `model`
- 可选 `modelProvider`
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
- 可选 native hook relay config。

### 8.4 Turn start

thread ready 后，`run-attempt.ts` 调用 app-server `turn/start`。`buildTurnStartParams()` 构造：

- `threadId`
- `input`：文本 prompt + inbound images。
- `cwd`
- `approvalPolicy`
- `approvalsReviewer`
- `sandboxPolicy`
- `model`
- `serviceTier`
- `effort`
- `collaborationMode`

reasoning effort 有兼容处理：现代 Codex 模型 `gpt-5.5`、`gpt-5.4`、`gpt-5.4-mini`、`gpt-5.2` 不接受 legacy `minimal`，因此 `minimal` 会映射为 `low`。

heartbeat turn 会通过 `collaborationMode.settings.developer_instructions` 注入 heartbeat-specific overlay；普通 chat turn 不携带 heartbeat 哲学。

### 8.5 Notification projection

`CodexAppServerEventProjector` 把 app-server notifications 投影成 OpenClaw run result：

- `item/agentMessage/delta`：累计 assistant text。
- `item/reasoning/summaryTextDelta`、`item/reasoning/textDelta`：累计 reasoning text。
- `item/plan/delta`、`turn/plan/updated`：累计 plan。
- `item/started`、`item/completed`：维护 item lifecycle 和 tool progress。
- `item/commandExecution/outputDelta`、`item/fileChange/outputDelta`：投影 native bash/apply_patch output progress。
- `item/autoApprovalReview/*`：投影 guardian review 事件。
- `hook/started`、`hook/completed`：投影 Codex native hook event。
- `thread/tokenUsage/updated`：更新 usage。
- `account/rateLimits/updated`：缓存 rate limit。
- `turn/completed`：记录 terminal turn。
- `error`：记录 prompt error，retryable error 除外。

最终 `buildResult()` 输出 `EmbeddedRunAttemptResult`，包含：

- `assistantTexts`
- `messagesSnapshot`
- `lastAssistant`
- `attemptUsage`
- tool telemetry
- replay metadata
- item lifecycle
- prompt error/abort/timeout 状态

OpenClaw 会 mirror transcript、触发 `llm_output` 和 `agent_end` hook，并在 context engine active 时 finalize context engine turn。

## 9. Dynamic tools 桥接

### 9.1 工具构造

`run-attempt.ts` 的 `buildDynamicTools()` 负责从 OpenClaw runtime 生成 Codex 可见工具：

1. 如果 `disableTools` 或模型不支持 tools，返回空。
2. 调用 `createOpenClawCodingTools()` 创建 OpenClaw coding tools。
3. 传入当前 channel、message target、sender、session、sandbox、workspace、agent、model、auth mode、heartbeat 等上下文。
4. 默认启用 message tool/heartbeat tool 的特殊策略。
5. 调用 `applyCodexDynamicToolProfile()` 排除 Codex native 已有工具。
6. 图片输入场景下调用 `filterToolsForVisionInputs()`。
7. 如果 `toolsAllow` 存在，再按 allowlist 过滤。
8. 调用 `normalizeAgentRuntimeTools()` 让 runtime plan 和 provider/model policy 继续规范化工具集。

默认 `native-first` profile 排除：

- `read`
- `write`
- `edit`
- `apply_patch`
- `exec`
- `process`
- `update_plan`

原因是这些能力 Codex app-server 已经有 native 实现。`openclaw-compat` profile 会保留更多 OpenClaw 兼容工具，适合测试和迁移场景。

### 9.2 工具 spec

`dynamic-tools.ts` 的 `createCodexDynamicToolBridge()` 把每个 OpenClaw tool 转为 Codex dynamic tool spec：

```ts
{
  name: tool.name,
  description: tool.description,
  inputSchema: toJsonValue(tool.parameters),
}
```

Codex app-server 只拿到 schema；真实 `execute()` 函数留在 OpenClaw 进程里。

### 9.3 工具调用

Codex app-server 发 `item/tool/call` request 后，OpenClaw：

1. 校验 request 对应当前 `threadId` 和 `turnId`。
2. 记录 trajectory `tool.call`。
3. 发 OpenClaw agent event `tool:start`。
4. 调用 `handleDynamicToolCallWithTimeout()`。
5. bridge 查找 tool map。
6. 运行 `prepareArguments()`。
7. 执行 `tool.execute(callId, args, signal)`。
8. 应用 tool-result middleware 和 Codex app-server legacy extension runner。
9. 收集 telemetry。
10. 触发 `after_tool_call` hook。
11. 把 OpenClaw tool result content 转成 Codex `contentItems`。
12. 返回 `{ success, contentItems }` 给 app-server。

动态工具有 30 秒 per-tool-call watchdog。超时会 abort tool signal，并返回失败文本；这和整次 app-server turn timeout、turn completion idle timeout 是不同层级。

## 10. Approvals 和 native hooks

### 10.1 App-server approval bridge

Codex native tool 或权限路径可能向 OpenClaw 发 server request：

- `item/commandExecution/requestApproval`
- `item/fileChange/requestApproval`
- `item/permissions/requestApproval`

`approval-bridge.ts` 将这些请求转成 OpenClaw plugin approval：

1. 确认 request 属于当前 thread/turn。
2. 构造 approval title/description/severity/toolName。
3. 清理命令预览中的 ANSI/control/invisible 字符，并限制长度。
4. 调用 `requestPluginApproval()`。
5. 等待 `waitForPluginApprovalDecision()`。
6. 将 OpenClaw 决策映射回 Codex app-server 期望的 response。

决策映射：

- command approval：`accept`、`acceptForSession`、`decline`、`cancel` 或 app-server 可用 amendment decision。
- file change：`accept`、`acceptForSession`、`decline`、`cancel`。
- permissions：approved 时返回 requested permissions 和 `scope: "turn" | "session"`；拒绝时返回空权限。

失败、不可用或 run abort 都 fail closed。

### 10.2 Native hook relay

`native-hook-relay.ts` 把 OpenClaw plugin hook relay 注入为 Codex app-server per-thread config。支持事件：

- `pre_tool_use` -> Codex `PreToolUse`
- `post_tool_use` -> Codex `PostToolUse`
- `permission_request` -> Codex `PermissionRequest`
- `before_agent_finalize` -> Codex `Stop`

`run-attempt.ts` 会注册 relay，并把 `buildCodexNativeHookRelayConfig()` 的结果放进 thread start/resume 的 `config` 字段。

默认策略：

- 如果 `approvalPolicy === "never"`，包含全部 relay events。
- 如果 app-server approval mode 启用，默认排除 `permission_request`，避免在 Codex guardian review 前暴露 stale permission request；真实升级由 app-server approval bridge 处理。

这层 relay 支持 observe/block/revise 等 OpenClaw plugin 兼容行为，但 v1 不支持改写 Codex-native tool arguments，也不让 OpenClaw 直接编辑 Codex native transcript。

## 11. Active run 控制、steer 和 timeout

`run-attempt.ts` 在 turn start 后注册 active embedded run handle：

- `queueMessage()` -> Codex `turn/steer`
- `isStreaming()`
- `isCompacting()`
- `cancel()` / `abort()`

steering queue 支持两种行为：

- 默认 `all`：短 debounce 后把多条 queued texts 合并为一个 `turn/steer`。
- `one-at-a-time`：逐条发送。

如果当前 app-server 正在等待 `item/tool/requestUserInput`，下一条 queued message 会优先作为 user input answer，而不是作为普通 steer 文本。

Timeout 分三层：

- `params.timeoutMs`：整次 run 外层 timeout。
- `turnCompletionIdleTimeoutMs`：OpenClaw 回复了 turn-scoped app-server request 后，等待 `turn/completed` 的短 idle watchdog。
- `turnTerminalIdleTimeoutMs`：长时间没有 terminal event 的 stuck-turn backstop。

abort 时会 best-effort 调用 `turn/interrupt`，释放 OpenClaw session lane。

## 12. Session binding 和 transcript mirror

OpenClaw session file 和 Codex native thread 通过旁路 binding 文件关联：

```text
<sessionFile>.codex-app-server.json
```

binding 由 `session-binding.ts` 读写。它让后续 OpenClaw turn 可以 resume 同一个 Codex thread，同时保留 model/provider/auth/policy/service tier/dynamic tool fingerprint。

OpenClaw 不把 Codex native thread 当作自己的 canonical transcript。它只维护 mirror：

- 用户 prompt。
- Codex reasoning/plan 的轻量记录。
- 最终 assistant text。
- 工具 telemetry 和 OpenClaw-owned tool results。

mirror 用于：

- OpenClaw history/search。
- context engine projection。
- `/new`、`/reset`、runtime 切换后的上下文恢复。
- trajectory/diagnostics。

Codex native compaction 由 app-server 处理，OpenClaw 只观察 compaction lifecycle；当前不获得详细 kept/dropped list 或 byte-for-byte internal prompt。

## 13. `/codex` 命令和 conversation binding

`commands.ts` 注册 `/codex` runtime command。它的 guidance 明确要求：如果用户要 bind/control/thread/resume/steer/stop Codex，优先 native `/codex ...`，只有用户明确要求 ACP/acpx 时才走 ACP。

`command-handlers.ts` 支持的主要子命令：

- `status`
- `models`
- `threads`
- `resume`
- `bind`
- `detach` / `unbind`
- `binding`
- `stop`
- `steer`
- `model`
- `fast`
- `permissions`
- `compact`
- `review`
- `diagnostics`
- `computer-use`
- `mcp`
- `skills`
- `account`

命令通过 `command-rpc.ts` 访问 app-server control methods，例如：

- `account/read`
- `account/rateLimits/read`
- `model/list`
- `thread/list`
- `thread/resume`
- `thread/compact/start`
- `review/start`
- `mcpServerStatus/list`
- `skills/list`
- `feedback/upload`

### 13.1 Conversation binding

`conversation-binding.ts` 处理 `/codex bind` 和 inbound claim：

- `startCodexConversationThread()` 可以创建新 Codex thread 或 attach existing thread。
- 绑定数据由 `conversation-binding-data.ts` 写入 plugin conversation binding。
- inbound message 命中 binding 后，`handleCodexConversationInboundClaim()` 拦截普通 agent 流程，把消息直接送给 bound Codex thread。
- 同一个 session file 使用 in-memory queue 串行执行 bound turns，避免一个 Codex thread 并发 turn。

bound conversation turn 当前与 harness run 有区别：

- 不暴露 OpenClaw dynamic tools。
- approval request fail closed，提示使用 Codex harness 或 explicit ACP。
- 主要目标是把聊天会话绑定到 Codex thread，支持普通回复、图片转发、stop/steer/model/fast/permissions 控制。

### 13.2 Bound turn input

`conversation-turn-input.ts` 会把 inbound channel metadata 中的图片转为 Codex input：

- 本地路径或 `file://` -> `{ type: "localImage", path }`
- 远端 URL -> `{ type: "image", url }`
- 文本 -> `{ type: "text", text, text_elements: [] }`

`conversation-turn-collector.ts` 监听当前 thread/turn 的 app-server notifications，累计 assistant text，在 `turn/completed` 后返回 channel reply。

### 13.3 Conversation control

`conversation-control.ts` 维护 active turn map：

- `/codex stop` -> `turn/interrupt`
- `/codex steer <message>` -> `turn/steer`
- `/codex model <model>` -> `thread/resume` with model override，并更新 binding。
- `/codex fast on|off` -> 更新 binding service tier。
- `/codex permissions yolo|default` -> 更新 binding approvalPolicy/sandbox。

这些控制只影响 bound conversation 或对应 session file 的 Codex thread binding。

## 14. Media understanding provider

`media-understanding-provider.ts` 注册 Codex image provider：

1. 解析模型 id，不能为空。
2. 创建 isolated app-server client。
3. 调用 `model/list` 确认模型存在且支持 image。
4. 新建 ephemeral thread：
   - `approvalPolicy: "on-request"`
   - `sandbox: "read-only"`
   - `dynamicTools: []`
   - `persistExtendedHistory: false`
   - developer instructions 限定为 image understanding worker。
5. 调用 `turn/start`，input 为 prompt + data URL images。
6. 拒绝所有 tool/file/permission/elicitation approval request。
7. 收集 assistant text，返回 `{ text, model }`。
8. 关闭 isolated client。

这条路径复用 Codex app-server 模型能力，但不会给图片描述 turn 工具或持久历史。

## 15. Computer Use 准备

Codex Computer Use 逻辑在 `src/app-server/computer-use.ts`。本文只概括它和 harness 的关系：

- `run-attempt.ts` 在 thread start 前调用 `ensureCodexComputerUse()`。
- 配置来自 `plugins.entries.codex.config.computerUse` 或对应 env。
- 它通过 app-server marketplace/plugin/MCP APIs 检查或安装 Codex Computer Use plugin。
- 最终目标是确保 app-server 侧 MCP server 可用；OpenClaw 不直接实现桌面控制。

因此 Computer Use 是 app-server native plugin/MCP preparation，不是 OpenClaw dynamic tool。

## 16. Migration provider

`src/migration/provider.ts` 注册 Codex migration provider：

- `detect()` 查找 Codex source state。
- `plan()` 构造迁移计划。
- `apply()` 执行迁移。

描述是：inventory and promote Codex CLI skills，同时保持 Codex native plugins/hooks 显式。它用于把操作者已有 Codex CLI 资产有选择地迁移/提升到 OpenClaw 管理范围，而不是让隔离 app-server 直接读取个人 Codex home。

## 17. Native Codex 与 ACP Codex 的区别

Native Codex app-server 路径：

- command: `/codex ...`
- runtime: `agentRuntime.id: "codex"`
- plugin: `extensions/codex`
- thread: Codex app-server native thread。
- OpenClaw embedded run 直接经 `runCodexAppServerAttempt()` 执行。
- OpenClaw dynamic tools 通过 app-server `dynamicTools` server request 回调。

ACP Codex 路径：

- command: `/acp ...`
- runtime: `runtime: "acp", agentId: "codex"`
- plugin: `extensions/acpx`
- thread/session: ACP runtime session。
- 适合显式测试 ACP/acpx 或外部 harness runtime。

用户说“绑定这个聊天到 Codex”“查看 Codex threads”“resume Codex thread”时，应使用 native `/codex`。用户明确说“通过 ACP/acpx 跑 Codex”时才使用 ACP。

## 18. 失败策略

Codex 插件整体偏 fail-closed：

- app-server 版本过旧或无法解析版本：拒绝运行。
- managed binary 找不到：报错并提示安装或配置自定义 binary。
- model discovery 失败：仅 catalog 回退 fallback，不代表 run 一定可用。
- app-server approval route 不可用：拒绝。
- dynamic tool 不存在：返回 failed tool response。
- dynamic tool 超时：返回 failed tool response 并 abort tool signal。
- bound conversation 的 dynamic tools/approval：默认拒绝。
- forced `agentRuntime.id: "codex"` 失败：不静默回退 PI。

这种策略避免把 native Codex 权限、账号、线程和 OpenClaw session 状态混在不明确的降级路径里。

## 19. 关键扩展点

可扩展或可调整的主要点：

- 新 app-server control method：添加到 `capabilities.ts` / `command-rpc.ts`，再接入 command handler。
- 新 notification 投影：扩展 `event-projector.ts` 和对应 tests。
- 新 app-server request handler：在 `run-attempt.ts` request handler 分支处理。
- Dynamic tool result middleware：通过 OpenClaw plugin hook/middleware，在 OpenClaw tool result 返回 Codex 前处理。
- Native hook relay event：需要 Codex app-server 支持对应 native hook，并明确加入 v1/vNext support contract。
- Provider catalog fallback：更新 `provider-catalog.ts` 和相关 provider tests。
- Config schema：同步 `openclaw.plugin.json`、`config.ts` parser、UI hints 和 docs。

不应做的扩展：

- 不从 `extensions/codex` 深 import core internals；生产代码走 `openclaw/plugin-sdk/*`。
- 不在 core 写 Codex 特定 runtime 迁移或恢复策略；owner-specific 行为留在 Codex 插件。
- 不用 scattered cache 掩盖 request-time broad discovery；应把 runtime fact 提前准备并传入。
- 不假设 Codex native thread 可被 OpenClaw 直接改写；除非 app-server 暴露正式 API。

## 20. 验证面

源码中已有高价值测试覆盖：

- provider/catalog：`extensions/codex/provider.test.ts`
- manifest：`extensions/codex/src/manifest.test.ts`
- commands：`extensions/codex/src/commands.test.ts`、`extensions/codex/src/command-rpc.test.ts`
- conversation binding/control：`extensions/codex/src/conversation-binding.test.ts`、`extensions/codex/src/conversation-control.test.ts`
- app-server client/transport：`extensions/codex/src/app-server/client.test.ts`、`transport-stdio.test.ts`、`transport-websocket.test.ts`
- auth bridge：`extensions/codex/src/app-server/auth-bridge.test.ts`
- run attempt：`extensions/codex/src/app-server/run-attempt.test.ts`
- dynamic tools：`extensions/codex/src/app-server/dynamic-tools.test.ts`
- approval/native hooks：`approval-bridge.test.ts`、`native-hook-relay.test.ts`
- thread lifecycle/session binding：`thread-lifecycle.test.ts`、`session-binding.test.ts`
- trajectory/transcript/context-engine：`trajectory.test.ts`、`transcript-mirror.test.ts`、`context-engine-projection.test.ts`
- media understanding：`extensions/codex/media-understanding-provider.test.ts`
- migration：`extensions/codex/src/migration/provider.test.ts`

Live proof 对应 docs：

- `docs/help/testing-live.md`
- `docs/help/testing.md`

典型 live lane 是 Codex app-server harness smoke，目标是证明 gateway agent turn 经 native Codex harness 执行、线程可续用、图片/MCP/guardian/subagent 等关键探针按配置通过。

## 21. 总结

Codex 插件的方案不是“把 OpenClaw 的 provider 请求转发给 Codex 模型 API”，而是“把 OpenClaw embedded agent runtime 嵌入到 Codex app-server 线程系统里”。它的设计重点是：

- 用 OpenClaw plugin manifest 暴露控制面事实。
- 用 agent harness 选择 native Codex runtime。
- 用 shared/isolated app-server client 管理本地或远端 Codex app-server。
- 用 auth bridge 保持每 agent 隔离和 OpenClaw auth profile 兼容。
- 用 thread binding 连接 OpenClaw session file 与 Codex native thread。
- 用 dynamic tools bridge 把 OpenClaw 工具留在 OpenClaw 进程执行。
- 用 notification projector 把 Codex native lifecycle 投影回 OpenClaw run result。
- 用 approval bridge 和 native hook relay 在 v1 合同内连接权限和插件 hook。
- 用 transcript mirror/context projection 在不改写 Codex canonical history 的前提下保留 OpenClaw 会话能力。

这让 OpenClaw 可以复用 Codex app-server 的 native coding runtime，同时保留 OpenClaw 的渠道、工具、审批、会话和插件生态边界。
