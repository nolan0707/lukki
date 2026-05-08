# OpenClaw Plugin SDK 接口能力理解

## 1. 阅读范围

本次梳理聚焦三块源码：

- `src/plugin-sdk/`：插件作者可导入的 SDK subpath、entry helper、runtime helper facade。
- `src/plugins/`：插件发现、manifest、loader、registry、metadata snapshot、host hooks、runtime 注入。
- `src/plugin-state/`：插件持久状态 KV store，以及它如何通过 `api.runtime.state.openKeyedStore()` 暴露。

相关公开文档：

- `docs/plugins/sdk-overview.md`
- `docs/plugins/sdk-entrypoints.md`
- `docs/plugins/sdk-runtime.md`
- `docs/plugins/architecture.md`
- `docs/plugins/manifest.md`
- `docs/plugins/building-plugins.md`
- `docs/plugins/hooks.md`

结论：Plugin SDK 不是一个独立外部应用 SDK，而是 native plugin 和 OpenClaw core 之间的进程内契约。插件通过 `openclaw/plugin-sdk/*` 声明入口、注册能力，并在运行时使用注入的 `OpenClawPluginApi` / `api.runtime` 调用 host 能力。真正的插件运行不隔离，和 Gateway 同进程、同信任边界。

## 2. SDK 的核心形态

### 2.1 入口 helper

插件 entry module 默认导出一个 plugin definition。SDK 提供三类常用 helper：

- `definePluginEntry(...)`：通用插件入口。适合 provider、tool、hook、service、HTTP route、CLI、memory、media 等非 channel 插件。实现见 `src/plugin-sdk/plugin-entry.ts`。
- `defineSingleProviderPluginEntry(...)`：单 provider 快速入口。封装 `definePluginEntry`，帮 provider 构建 API key auth method 和 catalog。实现见 `src/plugin-sdk/provider-entry.ts`。
- `defineChannelPluginEntry(...)` / `defineSetupPluginEntry(...)`：channel 插件入口。封装 channel 注册、runtime 注入、CLI metadata 和 full runtime 分段。实现见 `src/plugin-sdk/core.ts` 与 `src/plugin-sdk/channel-entry-contract.ts`。

入口运行时会收到 `OpenClawPluginApi`。该类型定义在 `src/plugins/types.ts`，是插件能力注册的主契约。

### 2.2 Registration mode

`api.registrationMode` 是插件入口必须尊重的运行模式：

- `full`：Gateway 正常启动，允许注册完整能力和启动长生命周期 side effect。
- `discovery`：只读能力发现，允许注册 capability 和静态 CLI metadata，但不应启动 socket、worker、client、listener。
- `tool-discovery`：工具能力发现，跳过 channel runtime hydration。
- `setup-only`：轻量 channel setup entry。
- `setup-runtime`：setup 流程需要轻量 runtime channel entry。
- `cli-metadata`：CLI 根 help / parse-time metadata 收集。

`src/plugins/registry.ts` 用 `resolvePluginRegistrationCapabilities()` 把 mode 归一成能力布尔值。`full`、`discovery`、`tool-discovery` 都允许 broad registry 写入；`setup-only` 和 `tool-discovery` 抑制 runtime channel 注册。

### 2.3 Manifest-first 控制面

每个 native plugin 需要 `openclaw.plugin.json`。Manifest 是控制面事实，不是 runtime 行为：

- 声明 `id`、`configSchema`、`uiHints`。
- 声明 `providers`、`channels`、`cliBackends`、`contracts.*` 等所有权。
- 声明 `activation`、`setup`、auth env vars、model catalog、provider request、channel configs 等静态元数据。

`src/plugins/plugin-metadata-snapshot.ts` 会构造 `PluginMetadataSnapshot`，包括 installed index、manifest registry、owner maps、diagnostics、normalizer 和 metrics。它不包含已加载模块或 runtime exports。Gateway startup 和很多冷路径用 snapshot/lookup table 做 owner lookup，避免反复 broad runtime load。

## 3. `OpenClawPluginApi` 可注册能力

### 3.1 Capability 类扩展

插件可以注册以下一等能力：

- 文本模型 provider：`api.registerProvider(...)`
- 低层 agent executor：`api.registerAgentHarness(...)`
- 本地 CLI inference backend：`api.registerCliBackend(...)`
- 消息 channel：`api.registerChannel(...)`
- Speech provider：`api.registerSpeechProvider(...)`
- Realtime transcription：`api.registerRealtimeTranscriptionProvider(...)`
- Realtime voice：`api.registerRealtimeVoiceProvider(...)`
- Media understanding：`api.registerMediaUnderstandingProvider(...)`
- Image generation：`api.registerImageGenerationProvider(...)`
- Video generation：`api.registerVideoGenerationProvider(...)`
- Music generation：`api.registerMusicGenerationProvider(...)`
- Web fetch：`api.registerWebFetchProvider(...)`
- Web search：`api.registerWebSearchProvider(...)`
- Migration provider：`api.registerMigrationProvider(...)`
- Context engine：`api.registerContextEngine(...)`
- Compaction provider：`api.registerCompactionProvider(...)`
- Memory capability：`api.registerMemoryCapability(...)`
- Memory embedding provider：`api.registerMemoryEmbeddingProvider(...)`
- Detached task runtime：`api.registerDetachedTaskRuntime(...)`

这些注册最终写入 `PluginRegistry`，类型在 `src/plugins/registry-types.ts`。registry 会记录 pluginId、source、rootDir、provider/tool/channel id，并做重复所有权和 contract 校验。

### 3.2 Tool / command / hook 类扩展

插件可以扩展 Agent 和消息执行链：

- Agent tool：`api.registerTool(tool, opts?)`
- 自定义文本命令：`api.registerCommand(def)`
- 内部 hook：`api.registerHook(events, handler, opts?)`
- typed plugin hook：`api.on(hookName, handler, opts?)`
- Tool metadata：`api.registerToolMetadata(...)`
- Tool-result middleware：`api.registerAgentToolResultMiddleware(...)`
- Trusted tool policy：`api.registerTrustedToolPolicy(...)`

Hook 覆盖面很广，主要包括：

- Agent turn：`before_model_resolve`、`agent_turn_prepare`、`before_prompt_build`、`before_agent_run`、`before_agent_reply`、`before_agent_finalize`、`agent_end`。
- Tool：`before_tool_call`、`after_tool_call`、`tool_result_persist`、`before_message_write`。
- Message：`inbound_claim`、`message_received`、`message_sending`、`message_sent`、`before_dispatch`、`reply_dispatch`。
- Session / compaction / subagent / lifecycle：`session_start`、`before_compaction`、`subagent_spawning`、`gateway_start`、`gateway_stop`、`before_install` 等。

`before_tool_call`、`before_agent_run`、`before_agent_reply`、`message_sending` 等支持决策结果，可以 block、rewrite、cancel、require approval 或 short-circuit。

### 3.3 Gateway / CLI / service 类扩展

插件可以扩展 host 表面：

- Gateway HTTP route：`api.registerHttpRoute(params)`
- Gateway RPC method：`api.registerGatewayMethod(name, handler, opts?)`
- Gateway discovery advertiser：`api.registerGatewayDiscoveryService(service)`
- CLI command registrar：`api.registerCli(registrar, opts?)`
- Node CLI feature：`api.registerNodeCliFeature(registrar, opts?)`
- Node host command：`api.registerNodeHostCommand(command)`
- Node invoke policy：`api.registerNodeInvokePolicy(policy)`
- Background service：`api.registerService(service)`
- Interactive handler：`api.registerInteractiveHandler(registration)`
- Reload hook：`api.registerReload(registration)`
- Security audit collector：`api.registerSecurityAuditCollector(collector)`
- Hosted media resolver：`api.registerHostedMediaResolver(resolver)`

Gateway method 有保留命名空间限制：`config.*`、`exec.approvals.*`、`wizard.*`、`update.*` 始终归为 `operator.admin`，插件不能用更窄 scope 降权。

CLI 支持 `descriptors` 作为 parse-time metadata，让 root help 能显示插件命令，同时真实 CLI 模块保持 lazy load。

### 3.4 Workflow / host state 类扩展

SDK 暴露了一组 host lifecycle seam：

- Session extension：`api.registerSessionExtension(...)`
- Next-turn injection：`api.enqueueNextTurnInjection(...)`
- Control UI descriptor：`api.registerControlUiDescriptor(...)`
- Runtime lifecycle cleanup：`api.registerRuntimeLifecycle(...)`
- Agent event subscription：`api.registerAgentEventSubscription(...)`
- Per-run JSON context：`api.setRunContext(...)` / `api.getRunContext(...)` / `api.clearRunContext(...)`
- Session scheduler job：`api.registerSessionSchedulerJob(...)`
- Conversation binding resolved handler：`api.onConversationBindingResolved(...)`

这些能力适合 approval workflow、background monitor、setup wizard、workspace policy、UI companion plugin 等。它们不是任意全局状态入口，而是按 pluginId/namespace/runId/sessionKey 绑定，由 host 负责清理和投影。

## 4. `api.runtime` 可调用能力

`api.runtime` 是 host 注入给 trusted native plugin 的运行时 helper，类型见 `src/plugins/runtime/types-core.ts`、`src/plugins/runtime/types-channel.ts`、`src/plugins/runtime/types.ts`。

主要 namespace：

- `runtime.config`：读取当前 runtime config snapshot，或通过 `mutateConfigFile` / `replaceConfigFile` 持久化配置。旧的 `loadConfig` / `writeConfigFile` 仍在但已 deprecated。
- `runtime.agent`：解析 agent dir/workspace/identity/thinking/timeout，确保 workspace，运行 embedded agent，读写 session store。
- `runtime.subagent`：启动、等待、读取、删除插件创建的 subagent session。model override 需要 operator opt-in。
- `runtime.nodes`：列出 paired nodes，调用 node-host command。
- `runtime.system`：system event、heartbeat、command timeout、native dependency hint。
- `runtime.media`：MIME、web media、图片 metadata、resize 等基础媒体工具。
- `runtime.tts`、`runtime.stt`、`runtime.mediaUnderstanding`：调用共享语音和媒体理解 runtime。
- `runtime.imageGeneration`、`runtime.videoGeneration`、`runtime.musicGeneration`：调用共享生成能力并列出 runtime provider。
- `runtime.webSearch`：列出和调用 web search provider。
- `runtime.events`：订阅 agent event 和 session transcript update。
- `runtime.logging`：子 logger 和 verbose 判断。
- `runtime.state`：解析 state dir，打开 keyed plugin state store。
- `runtime.tasks` / `runtime.taskFlow`：task run、Task Flow、managed flow runtime。
- `runtime.modelAuth`：按 model/provider 解析 auth 或 runtime auth。
- `runtime.channel`：channel 插件专用 helper，覆盖 chunking、reply dispatch、routing、pairing、media、activity、session、mentions、reactions、groups、debounce、commands、outbound adapter、channel turn、thread bindings、runtime contexts。

重要限制：`runtime.state.openKeyedStore()` 在 `src/plugins/registry.ts` 中通过 proxy 加了 origin 检查，当前 release 只允许 bundled plugin 使用；非 bundled plugin 调用会抛错。

## 5. Plugin state 能力

`src/plugin-state/` 提供 SQLite-backed keyed store。公开类型：

```ts
type PluginStateKeyedStore<T> = {
  register(key, value, opts?)
  registerIfAbsent(key, value, opts?)
  lookup(key)
  consume(key)
  delete(key)
  entries()
  clear()
}
```

打开参数：

- `namespace`
- `maxEntries`
- `defaultTtlMs`

约束来自 `src/plugin-state/plugin-state-store.ts` 和 `src/plugin-state/plugin-state-store.sqlite.ts`：

- namespace 必须是安全 path segment，最长 128 bytes。
- key 最长 512 bytes。
- value 必须是 JSON-serializable plain value，不能有循环引用、symbol key、getter/setter、非枚举属性、稀疏数组。
- JSON 最大深度 64。
- 单 value 最大 64KB。
- `ttlMs` 必须为正整数。
- 同一 pluginId + namespace reopen 时 `maxEntries` / `defaultTtlMs` 必须一致。
- 超过 `maxEntries` 会按 namespace 淘汰旧 entry。
- `pluginId` 以 `core:` 开头保留给 core consumer。

所以 plugin-state 适合短小、插件私有、可 TTL 的工作流状态，例如 approval token、临时映射、去重 key、小型 checkpoint。不适合大文件、长期数据库、跨插件共享状态、secret store 或高吞吐队列。

## 6. 可以扩展什么，如何扩展

### 6.1 新模型 provider

可扩展。方式：

1. 在 manifest 声明 `providers`、`configSchema`、auth/setup/model metadata、activation hint。
2. 用 `definePluginEntry` 或 `defineSingleProviderPluginEntry` 创建入口。
3. 在 `register(api)` 中调用 `api.registerProvider(...)`。
4. 按需注册 CLI backend、media/speech/generation/web capability。
5. 将 provider-owned config、auth、catalog、request compat、thinking policy、replay policy 放在 provider plugin 内，不让 core hardcode vendor id。

局限：如果需要的是 OpenClaw 尚不存在的 capability contract，不能只靠插件私有 API 完成一等集成。需要先在 core 定义 typed capability，再通过 SDK 暴露。

### 6.2 新消息渠道

可扩展。方式：

1. Manifest 声明 `channels`、`channelConfigs`、setup metadata、activation hint。
2. 用 `defineChannelPluginEntry` 注册 `ChannelPlugin`。
3. 在 channel adapter 中实现入站接收、outbound send/edit/react/poll、目录、pairing、mention/group/thread 行为。
4. 用 `api.runtime.channel.*` 复用核心 turn、reply、session、pairing、media、reaction、command helper。
5. 对 setup-only 场景提供 `setupEntry`，避免 disabled/unconfigured 时加载重 runtime。

局限：core 仍拥有 shared `message` tool、prompt wiring、session/thread bookkeeping 和 dispatch host。channel 插件只能通过 channel contract 暴露 action/capability/schema contribution，不能直接重写 core message tool 的总体语义。

### 6.3 新 Agent tool

可扩展。方式：

1. Manifest `contracts.tools` 声明 tool name。
2. 需要按 config/env/auth 控制可用性时，补充 `toolMetadata` 和 activation。
3. Runtime 里 `api.registerTool(...)` 注册 required 或 optional tool。
4. 需要审批、参数改写、阻断时，用 `before_tool_call` hook。

局限：tool metadata 只能装饰本插件声明的 tool。registry 会检查 `contracts.tools`，不能用 metadata 改写其他插件或 core tool。

### 6.4 新 hook / workflow / prompt 扩展

可扩展。方式：

1. 用 `api.on(...)` 注册 typed hook。
2. 对 prompt 注入使用 `agent_turn_prepare`、`before_prompt_build`、`heartbeat_prompt_contribution` 或 `enqueueNextTurnInjection`。
3. 对工作流状态使用 session extension、run context、session scheduler job、agent event subscription。

局限：

- `plugins.entries.<id>.hooks.allowPromptInjection=false` 会禁用 prompt-mutating hook 和 next-turn injection。
- hook 有 priority 和 timeout，不能无限阻塞调用方。
- `before_agent_start` 仍支持但属于兼容旧 seam，新插件应使用更细分的 phase hook。

### 6.5 新 Gateway / CLI / Control UI 表面

可扩展。方式：

- Gateway HTTP：`api.registerHttpRoute(...)`
- Gateway RPC：`api.registerGatewayMethod(...)`
- CLI：`api.registerCli(...)`，优先提供 `descriptors` 保持 lazy load。
- Node 功能：`api.registerNodeCliFeature(...)`、`api.registerNodeHostCommand(...)`、`api.runtime.nodes.invoke(...)`。
- Control UI contribution：`api.registerControlUiDescriptor(...)`

局限：

- Gateway core admin namespace 保留。
- Control UI descriptor 当前是描述符贡献，不等于任意注入前端代码。
- Node command 仍受 node pairing、allowlist、plugin node-invoke policy 约束。

### 6.6 新媒体 / 搜索 / 语音 capability provider

可扩展。方式：

- 注册对应 provider：speech、realtime transcription、realtime voice、media understanding、image/video/music generation、web fetch、web search。
- Manifest 中用 `contracts.*Providers` 和 provider metadata 提供静态 auth/config availability signals。
- 消费方通过 `api.runtime.*` 调用共享 core runtime，不直接 import vendor plugin。

局限：fallback、选择策略、工具 schema、delivery semantics 通常由 core capability 层拥有。插件实现 provider 适配，不应绕过共享 runtime 自造一套 vendor-specific 调度。

### 6.7 Memory / context engine

可扩展，但有 slot 约束：

- `kind: "memory"` 的插件可以注册 `registerMemoryCapability(...)`。
- memory prompt/corpus supplement 是 additive，多个插件可并存。
- context-engine 是 exclusive slot。
- memory embedding provider 可以多实现并存，但非 memory 插件必须在 manifest `contracts.memoryEmbeddingProviders` 声明。

局限：独占 slot 只能有一个 active owner。未被 slot 选中的 dual-kind memory plugin 会跳过相关 registration。

## 7. 不能扩展或受限的点

### 7.1 插件不能安全隔离执行

Native plugin in-process 运行，不是 sandbox。加载后可以执行任意进程内代码，bug 可 crash Gateway，恶意插件等同 Gateway 进程内任意代码执行。因此：

- 外部 native plugin 需要 allowlist/install/load path 信任。
- `plugins.allow` 信任的是 plugin id，不是 source provenance。
- workspace plugin 可 shadow bundled plugin id，这是开发能力也是风险。

### 7.2 外部插件不能使用所有 trusted seam

这些 seam 当前是 bundled/trusted-only：

- `api.registerAgentToolResultMiddleware(...)`：非 bundled 会产生 diagnostic error；还必须声明 `contracts.agentToolResultMiddleware`。
- `api.registerTrustedToolPolicy(...)`：非 bundled 会产生 diagnostic error。
- `api.runtime.state.openKeyedStore(...)`：非 bundled 调用会抛错。
- `api.registerCodexAppServerExtensionFactory(...)`：文档和类型说明为 bundled-only，且需要 manifest contract。

外部插件应优先使用普通 hook、tool、command、service、session extension 等稳定 seam。

### 7.3 插件不能绕过 manifest ownership

很多 runtime 注册要求 manifest 先声明：

- tool 必须在 `contracts.tools` 中声明。
- tool metadata 必须对应本插件声明的 tool。
- agent tool result middleware 必须声明目标 runtime contract。
- memory embedding provider 对非 memory 插件要求 manifest contract。
- activation、setup、provider/channel owner lookup 依赖 manifest 控制面。

这意味着“只在 register(api) 里偷偷注册能力”不是完整做法；配置校验、startup auto-enable、CLI/help、Control UI/schema、provider catalog 等冷路径可能看不到该能力。

### 7.4 插件不能拥有 core 的通用策略

插件是 ownership boundary，capability 是 core contract。插件可以实现 provider/channel/tool，但不应：

- 在 core 中 hardcode 自己的 vendor id、channel id、默认值、migrations、recovery policy。
- 让 channel 直接 import vendor provider 实现。
- 通过 private SDK facade 暴露只服务单一 bundled plugin 的便利 API。
- 在请求热路径调用 broad loader 来重新发现已知 provider/channel/tool。

如果多个插件需要同类能力，应该先定义 generic capability 或 family-level SDK seam，再让插件实现。

### 7.5 插件不能任意修改配置和运行时状态

配置写入必须通过 `runtime.config.mutateConfigFile(...)` 或 `replaceConfigFile(...)`，并显式声明 `afterWrite` 策略。旧的 `loadConfig` / `writeConfigFile` deprecated，repo 内 bundled plugin 受架构 guard 限制。

run context、session extension、session scheduler job 都按 pluginId/namespace/session/run 归属。插件不能用这些接口直接写入别的插件 namespace。

### 7.6 插件不能替代完整加载生命周期

Manifest `activation` 只是 planner hint，不是 lifecycle hook。真正 runtime 行为仍必须通过 `register(api)`。同样，`discovery` 可能 evaluate trusted plugin entry，但要求轻量、无 live side effect；side effect 应只在 `full` 中启动。

### 7.7 SDK subpath 不是全部稳定公共 API

`docs/plugins/architecture.md` 明确：capability registration 是方向，但外部插件兼容性仍以文档标注的 narrow contract 为准。生成的 SDK subpath 很多，部分是 bundled-plugin maintenance 或 deprecated compatibility facade。新插件应：

- 从具体、窄的 `openclaw/plugin-sdk/*` subpath import。
- 避免 provider/channel-branded convenience seam。
- 不 import `src/**`、`src/plugin-sdk-internal/**` 或其他插件内部源码。

## 8. 扩展设计建议

判断一个新需求放在哪里：

- 如果是 vendor API、auth、catalog、transport patch：放 vendor plugin。
- 如果是 channel 平台语义、消息 target、thread、pairing、发送行为：放 channel plugin。
- 如果是多个 vendor/channel 都会使用的能力：先定义 core capability contract，再通过 SDK 注册 provider/consumer。
- 如果只是 operator automation：优先用 hook/plugin command/service，不要扩 core。
- 如果需要 prompt 注入：使用 phase-specific hook，并尊重 `allowPromptInjection`。
- 如果需要持久小状态：bundled plugin 可用 plugin-state；外部插件当前应使用自身文件/服务或等待该 seam 开放。
- 如果需要在 setup/config/doctor 冷路径可见：必须补 manifest 元数据，不能只写 runtime register。

## 9. 关键源码索引

- `src/plugins/types.ts`：`OpenClawPluginApi`、provider/channel/tool/hook/service 类型。
- `src/plugins/registry.ts`：registry 写入、重复校验、bundled-only seam 检查、runtime proxy。
- `src/plugins/registry-types.ts`：`PluginRegistry` 和各类 registration record。
- `src/plugins/loader.ts`：discovery 到 runtime load 的主流程。
- `src/plugins/plugin-metadata-snapshot.ts`：metadata snapshot 构建。
- `src/plugins/plugin-metadata-snapshot.types.ts`：snapshot 结构。
- `src/plugins/plugin-registry-contributions.ts`：manifest contribution owner lookup。
- `src/plugins/host-hooks.ts`：session extension、trusted policy、run context、scheduler job 等 host hook 类型。
- `src/plugins/runtime/types-core.ts`：`api.runtime` core helper。
- `src/plugins/runtime/types-channel.ts`：channel runtime helper。
- `src/plugin-sdk/plugin-entry.ts`：`definePluginEntry`。
- `src/plugin-sdk/provider-entry.ts`：`defineSingleProviderPluginEntry`。
- `src/plugin-sdk/core.ts`：channel/core SDK facade。
- `src/plugin-sdk/channel-entry-contract.ts`：bundled channel entry contract。
- `src/plugin-state/plugin-state-store.ts`：plugin state validation 和 keyed store facade。
- `src/plugin-state/plugin-state-store.sqlite.ts`：plugin state SQLite schema、TTL、maxEntries、64KB value 限制。
