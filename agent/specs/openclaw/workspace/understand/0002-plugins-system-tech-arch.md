# OpenClaw 插件系统技术架构理解

## 1. 总览

OpenClaw 的插件系统是项目的能力扩展中枢。它把模型 Provider、消息 Channel、CLI backend、Agent tool、HTTP route、Gateway method、Hook、Service、Memory、Media、Web search/fetch、Speech、Realtime、QA runner、Skill bundle 等能力统一收敛到一个插件控制面和运行时注册表里。

代码实现主要分布在：

- `src/plugins/`：插件发现、manifest 解析、启用策略、installed index、metadata snapshot、activation planner、loader、runtime registry、契约测试。
- `src/plugin-sdk/`：插件作者使用的公开 SDK subpath 和入口 helper。
- `extensions/`：内置插件。内置插件也按第三方插件边界组织，只通过 `openclaw/plugin-sdk/*` 和自身 `api.ts` / `runtime-api.ts` 暴露能力。
- `docs/plugins/*`、`docs/tools/plugin.md`：插件架构、manifest、SDK、安装管理文档。

核心设计不是“启动时 import 所有扩展”，而是分成两个平面：

- 控制面：只读 manifest、package metadata、installed index、bundle metadata，完成发现、配置校验、owner lookup、setup hint、activation plan、模型目录等冷路径判断。
- 运行面：真正加载插件 entry module，创建 `OpenClawPluginApi`，调用 `register(api)`，把能力写入 `PluginRegistry`，再由 Gateway、Agent、Channel、Tools 等消费。

这个分层支撑了两个目标：配置和诊断尽量不执行插件代码；请求热路径尽量只加载当前 provider/channel/tool 相关插件。

## 2. 插件形态

OpenClaw 当前识别两大格式。

### 2.1 Native OpenClaw plugin

Native 插件必须有 `openclaw.plugin.json`，并在 `package.json#openclaw.extensions` 或候选入口中提供 runtime entry。典型结构：

```text
extensions/openai/
  openclaw.plugin.json
  package.json
  index.ts
  api.ts
  runtime-api.ts
  src/*
```

Native 插件的 manifest 用于控制面，entry module 用于运行面。

典型 provider 插件入口是 `extensions/openai/index.ts`：

- `definePluginEntry(...)` 定义插件。
- `register(api)` 中注册 provider、CLI backend、memory embedding、image generation、realtime transcription、realtime voice、speech、media understanding、video generation。
- manifest `extensions/openai/openclaw.plugin.json` 提前声明 `providers`、`modelSupport`、`modelCatalog`、`providerEndpoints`、`providerRequest` 等元数据。

### 2.2 Bundle plugin

兼容 Codex、Claude、Cursor bundle 布局：

- Codex：`.codex-plugin/plugin.json`
- Claude：`.claude-plugin/plugin.json` 或默认 component layout
- Cursor：`.cursor-plugin/plugin.json`

解析实现位于 `src/plugins/bundle-manifest.ts`。Bundle 记录会映射出 skills、settings、commands、agents、output styles、LSP、MCP、hooks 等能力，但不会走 native `register(api)` 的全部运行时能力模型。`src/plugins/loader.ts` 对 `record.format === "bundle"` 有单独路径：记录能力、发出 unsupported capability warning，然后直接进入 registry record。

## 3. 控制面：Manifest-first

### 3.1 Manifest 文件

`src/plugins/manifest.ts` 定义 `PluginManifest`。`loadPluginManifest()` 读取 `openclaw.plugin.json`，特点：

- 文件名固定为 `openclaw.plugin.json`。
- 最大读取 `MAX_PLUGIN_MANIFEST_BYTES = 256 * 1024`。
- 通过 `openRootFileSync()` 做插件根目录边界检查、hardlink 检查、最大字节数限制。
- JSON 解析使用 `parseJsonWithJson5Fallback()`。
- 必须包含 `id` 和对象形式 `configSchema`。
- 解析结果有 LRU cache，但 key 绑定文件 stat、路径和安全参数。

Manifest 的核心字段包括：

- 身份和配置：`id`、`name`、`description`、`version`、`configSchema`、`uiHints`。
- 启用策略：`enabledByDefault`、`enabledByDefaultOnPlatforms`、`legacyPluginIds`、`autoEnableWhenConfiguredProviders`、`kind`。
- 能力所有权：`providers`、`channels`、`cliBackends`、`contracts.*`。
- 控制面 provider 元数据：`modelSupport`、`modelCatalog`、`modelPricing`、`modelIdNormalization`、`providerEndpoints`、`providerRequest`。
- setup/onboarding：`setup.providers`、`providerAuthChoices`、`providerAuthEnvVars`、`channelEnvVars`、`syntheticAuthRefs`、`nonSecretAuthMarkers`。
- 加载规划：`activation.onStartup`、`onProviders`、`onChannels`、`onCommands`、`onRoutes`、`onCapabilities`、`onConfigPaths`。
- 其他：`qaRunners`、`skills`、`channelConfigs`、`toolMetadata`、generation provider metadata、config contracts。

关键原则：manifest 不声明代码 entrypoint，不承担 runtime 行为注册；它是便宜、可静态检查的控制面事实。

### 3.2 Discovery

发现入口是 `discoverOpenClawPlugins()`，位于 `src/plugins/discovery.ts`。

来源包括：

- `plugins.load.paths` 配置路径，origin 为 `config`。
- workspace 下的 `extensions` 根，origin 为 `workspace`。
- 全局/managed plugin roots，origin 为 `global`。
- OpenClaw bundled plugin root，origin 为 `bundled`。
- source overlay 和 packaged bundled load path alias。

安全检查包括：

- entry source 不能 escape plugin root。
- 非 Windows 平台检查 stat。
- world-writable 路径阻断。
- 非 bundled 插件要求 owner uid 是当前进程 uid 或 root。
- bundled 路径在可修复时会收紧过宽权限。

发现结果只是 `PluginCandidate[]` 和 diagnostics。候选包含 `idHint`、`source`、`setupSource`、`rootDir`、`origin`、`format`、`bundleFormat`、package manifest、dependencies、bundled manifest 等。

### 3.3 Manifest registry

`loadPluginManifestRegistry()` 位于 `src/plugins/manifest-registry.ts`，负责把 discovery candidate 转为 `PluginManifestRecord`：

- 加载 native manifest 或 bundle manifest。
- 校验 host version、重复 id、路径归属、package metadata。
- 合并 package channel metadata 到 `channelConfigs`。
- 记录 source、rootDir、manifestPath、config schema、contracts、providers、channels、activation、setup、dependency metadata。
- 对同一物理插件根按 origin rank 去重，`config > workspace > global > bundled`。

`PluginManifestRecord` 是控制面的核心数据结构，后续 metadata snapshot、activation planner、config schema、startup plan 都围绕它工作。

### 3.4 Installed plugin index 和 persisted registry

安装插件不是 Gateway 启动时做。安装/更新/doctor 负责修复依赖和生成 installed index。相关文件：

- `src/plugins/installed-plugin-index.ts`
- `src/plugins/installed-plugin-index-store.ts`
- `src/plugins/plugin-registry-snapshot.ts`
- `src/plugins/plugin-registry-contributions.ts`

`loadPluginRegistrySnapshotWithMetadata()` 优先读取 persisted index，但会在以下情况回退 derived index：

- policy hash 与当前 config 不匹配。
- persisted source/manifest/setup source 缺失。
- bundled root 指向另一个 tree。
- manifest/package 文件 hash 或 signature stale。
- index 缺少可恢复的 install records。
- `OPENCLAW_DISABLE_PERSISTED_PLUGIN_REGISTRY` 禁用。

这让 startup 可以快读索引，又不会把 stale registry 当真。

### 3.5 Plugin metadata snapshot

`loadPluginMetadataSnapshot()` 位于 `src/plugins/plugin-metadata-snapshot.ts`。它组合：

- installed index 或 registry snapshot。
- manifest registry。
- diagnostics。
- `normalizePluginId`。
- `byPluginId`。
- owner maps。
- metrics。
- config/control-plane fingerprint。

Owner maps 记录：

- `channels`
- `channelConfigs`
- `providers`
- `modelCatalogProviders`
- `cliBackends`
- `setupProviders`
- `commandAliases`
- `contracts`

Gateway startup 使用当前 config 构造一个 snapshot。`isPluginMetadataSnapshotCompatible()` 通过 policy hash、config fingerprint、workspaceDir、index fingerprint 判断是否可复用。

这个 snapshot 是元数据快照，不包含已加载模块、provider SDK、runtime exports 或运行时 registry。它的边界是“当前 config + 当前 plugin inventory”的不可变事实。

## 4. 启用策略和 Activation planning

### 4.1 Config normalization

插件启用逻辑在 `src/plugins/config-state.ts` 及 `src/plugins/config-activation-shared.ts`。

核心输入：

- `plugins.enabled`
- `plugins.allow` / `plugins.deny`
- `plugins.entries.<id>.enabled`
- `plugins.entries.<id>.config`
- `plugins.slots.*`
- bundled manifest `enabledByDefault`
- platform default
- channel config 对 bundled channel 的显式启用
- auto-enabled provider reason

`normalizePluginId()` 支持 legacy/built-in alias，例如 `openai-codex -> openai`。

Vitest 下还有测试默认：如果没有显式插件配置，`plugins.enabled=false` 且 memory slot 默认 none，避免测试隐式加载重插件。

### 4.2 Slot 策略

`kind` 支持独占插件类型，例如 `memory`、`context-engine`。`resolveMemorySlotDecision()` 决定 memory 插件是否被 slot 选中。loader 对 bundled memory 插件有 fast path：如果 slot 策略已经保证不会启用，避免 import 重 memory runtime。

### 4.3 Activation planner

`src/plugins/activation-planner.ts` 提供 manifest 驱动的加载计划。

触发类型：

- command
- provider
- agentHarness
- channel
- route
- capability

命中原因分两类：

- 显式 hint：`activation-provider-hint`、`activation-channel-hint`、`activation-command-hint`、`activation-route-hint`、`activation-capability-hint`、`activation-agent-harness-hint`。
- manifest ownership fallback：`manifest-provider-owner`、`manifest-channel-owner`、`manifest-command-alias`、`manifest-setup-provider-owner`、`manifest-tool-contract`、`manifest-hook-owner`。

输出是 `PluginActivationPlan`，包含 `pluginIds`、entries、diagnostics。它的作用是把“当前要处理某个 provider/channel/command/tool”转成最小 plugin id 集合，避免 broad load。

## 5. 运行面：Loader

插件运行面入口是 `loadOpenClawPlugins()`，位于 `src/plugins/loader.ts`。

### 5.1 Load options

`PluginLoadOptions` 支持：

- `config`、`activationSourceConfig`、`autoEnabledReasons`
- `workspaceDir`、`env`
- Gateway core handlers/method names
- runtime options
- plugin SDK resolution preference
- cache 开关
- `mode: "full" | "validate"`
- `onlyPluginIds`
- setup-only channel 相关选项
- `preferSetupRuntimeForChannelPlugins`
- `preferBuiltPluginArtifacts`
- `toolDiscovery`
- `activate`
- `loadModules`
- 外部传入 `manifestRegistry`

`resolvePluginLoadCacheContext()` 会规范化配置、installed records、onlyPluginIds，并构造 cache key。cache key 覆盖 workspace、plugin config/trust list、activation metadata、installs、env、runtime subagent mode、SDK resolution、toolDiscovery、loadModules、activate 等。

### 5.2 Runtime 懒初始化

loader 不会一开始创建完整 runtime。它先创建一个 `Proxy<PluginRuntime>`：

- 第一次访问 runtime 属性时才加载 `src/plugins/runtime/index.ts`。
- `createPluginRuntime()` 内部又用 lazy runtime module 加载 TTS、media understanding、model auth 等重模块。
- 插件 API 拿到的是按 plugin id 包装过的 runtime，支持 plugin state keyed store 和 subagent scope。

这保证 discovery、validate、disabled plugin 路径不提前拉起全部依赖树。

### 5.3 Module loader

`createPluginModuleLoader()` 使用 `createPluginModuleLoaderCache()` 和 `getCachedPluginModuleLoader()`：

- 为每个 module path 构造 jiti/native loader。
- 注入 SDK alias map，让插件 import `openclaw/plugin-sdk/*` 能解析到当前 host SDK。
- 优先 native module load，TypeScript source 是 fallback/dev path。
- 对 bundled source checkout 可通过 `preferBuiltPluginArtifacts` 优先 `dist-runtime` / `dist` 产物。

`runPluginRegisterSync()` 强制 `register(api)` 必须同步。如果返回 Promise，会抛出 `plugin register must be synchronous`。异步工作应注册 service/hook/route，在运行时事件里执行。

### 5.4 Registration plan

`resolvePluginRegistrationPlan()` 把 loader 意图转成内部行为标志：

| mode | 使用场景 | 入口 | 注册行为 |
| --- | --- | --- | --- |
| `full` | 正常运行时激活 | runtime entry | 注册全部能力和全局副作用 |
| `discovery` | 非激活 registry snapshot | runtime entry | 可注册能力快照，但不激活全局副作用 |
| `setup-only` | disabled/unconfigured channel 的 setup surface | setup entry | 只注册 channel setup |
| `setup-runtime` | 配置通道延迟 full load，但 setup 需要轻量 runtime | setup entry + runtime setter | setup-safe runtime/channel 面 |
| `tool-discovery` | 工具发现 | runtime entry | 能力注册，但 suppressed runtime channel |
| `cli-metadata` | CLI root help/parse-time metadata | cli metadata entry | 只注册 CLI descriptors |

`src/plugins/registry.ts` 通过 `resolvePluginRegistrationCapabilities()` 进一步把 mode 解码成：

- `capabilityHandlers`：`full`、`discovery`、`tool-discovery` 为 true。
- `runtimeChannel`：除 `setup-only`、`tool-discovery` 外为 true。

这样注册表不在各个 handler 里散落字符串判断。

### 5.5 Loader 主流程

`loadOpenClawPlugins()` 的主流程：

1. 解析 onlyPluginIds，空 scope 直接返回空 registry。
2. 解析 load cache context。
3. 尝试复用 active registry 或 loader cache。
4. 必要时清理已激活插件 runtime state。
5. 创建 lazy runtime proxy 和 `createPluginRegistry(...)`。
6. discovery 或使用传入 manifest registry。
7. 加载 manifest registry，push diagnostics。
8. 对 candidate 排序、去重。
9. 对每个 candidate：
   - 查 manifest record。
   - 检查 onlyPluginIds。
   - 计算 activation state 和 enable state。
   - 建立 `PluginRecord`。
   - 判断 registration plan；无 plan 则 disabled。
   - bundle 插件走 bundle path。
   - memory slot fast path。
   - 校验 manifest config schema。
   - `loadModules=false` 时只写 manifest snapshot metadata。
   - 安全打开 entry source，防 root escape。
   - import module。
   - setup entry 特殊处理。
   - resolve default/module/register/activate export。
   - 检查 export id 和 manifest id 一致。
   - 处理 kind mismatch warning。
   - full activation only 注册 reload/node/security audit。
   - validate mode 只记录，不调用 register。
   - 构造 api，调用 `register(api)`。
   - 若 snapshot load，恢复全局注册状态，避免污染进程。
   - 错误时 rollback plugin global side effects 和 registry snapshot。
10. activate 时调用 `setActivePluginRegistry()` 和 `initializeGlobalHookRunner()`。
11. 如果 `throwOnLoadError`，有 error plugin 则抛 `PluginLoadFailureError`。

## 6. Runtime registry

### 6.1 Registry 数据结构

`src/plugins/registry-types.ts` 定义 `PluginRegistry`。主要集合：

- `plugins`
- `tools`
- `hooks` / `typedHooks`
- `channels` / `channelSetups`
- `providers`
- `cliBackends`
- `speechProviders`
- `realtimeTranscriptionProviders`
- `realtimeVoiceProviders`
- `mediaUnderstandingProviders`
- `imageGenerationProviders`
- `videoGenerationProviders`
- `musicGenerationProviders`
- `webFetchProviders`
- `webSearchProviders`
- `migrationProviders`
- `memoryEmbeddingProviders`
- `agentHarnesses`
- `gatewayHandlers`
- `httpRoutes`
- `cliRegistrars`
- `services`
- `gatewayDiscoveryServices`
- `commands`
- host hooks：session extensions、trusted tool policies、tool metadata、Control UI descriptors、runtime lifecycles、agent event subscriptions、scheduler jobs
- `diagnostics`

`PluginRecord` 是每个插件的运行时摘要，记录 status、origin、enabled、activation source、已注册能力 ids、config schema、contracts、dependency status、failure phase 等。

### 6.2 createPluginRegistry

`createPluginRegistry()` 位于 `src/plugins/registry.ts`。它返回：

- `registry`
- `createApi(record, params)`
- 注册 helper
- rollback side effects
- diagnostics helper

它负责所有注册约束：

- tool 必须在 manifest `contracts.tools` 声明。
- channel id 不能重复，runtime channel 和 setup channel 分开。
- provider id 不能重复。
- Gateway method 不能覆盖 core method，保留 admin namespace 会 scope coercion。
- HTTP route 做 path/match/auth overlap 检查。
- bundled-only 能力限制，例如 Codex app-server extension factory、agent tool result middleware、trusted tool policy。
- memory capability 只能由 memory kind 插件注册，或按 contracts 声明 memory embedding provider。
- `registerHook()` 可同时写 registry 和内部 hook 系统，非 activate 或 hooks disabled 时不写全局副作用。
- typed hook 受 `allowPromptInjection`、`allowConversationAccess`、timeout policy 约束。

### 6.3 Active registry

active registry 由 `src/plugins/runtime.ts` / `src/plugins/active-runtime-registry.ts` 等文件管理。它是进程当前运行时插件状态：

- `setActivePluginRegistry()` 替换 active registry。
- registry 替换时清理 host hook runtime、commands、interactive handlers、memory/global state 等。
- `ensurePluginRegistryLoaded()` 位于 `src/plugins/runtime/runtime-registry-loader.ts`，按 scope 加载：
  - `configured-channels`
  - `channels`
  - `all`
- 对 scoped request，可通过 onlyPluginIds 或 onlyChannelIds 只加载当前需要的 owner 插件。

运行时热路径应优先使用已有 prepared facts 或 active registry，避免重新 broad discovery。

## 7. Plugin SDK

`src/plugin-sdk/` 是插件和核心之间的公开契约。原则是窄 subpath，而不是大而全 barrel。导入形态：

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { defineChannelPluginEntry } from "openclaw/plugin-sdk/channel-core";
```

### 7.1 definePluginEntry

`src/plugin-sdk/plugin-entry.ts` 的 `definePluginEntry()` 用于普通插件：

- provider
- tool
- command
- service
- hook
- memory
- context-engine

返回标准 entry object：

- `id`
- `name`
- `description`
- `kind`
- `configSchema`
- `reload`
- `nodeHostCommands`
- `securityAuditCollectors`
- `register(api)`

`configSchema` 支持 lazy getter，并通过 `createCachedLazyValueGetter()` memoize。

### 7.2 Provider helper

`src/plugin-sdk/provider-entry.ts` 的 `defineSingleProviderPluginEntry()` 是单 provider 简化 helper：

- 自动构造 API key auth method。
- 支持 env vars、wizard setup、catalog/staticCatalog。
- 最终仍调用 `definePluginEntry()`，并在 `register(api)` 中 `api.registerProvider(...)`。

多能力 vendor 插件通常直接用 `definePluginEntry()`，例如 OpenAI。

### 7.3 Channel helper

`defineChannelPluginEntry()` 实现在 `src/plugin-sdk/core.ts`，并通过 `src/plugin-sdk/channel-core.ts` 重新导出。

它会：

- 包装 channel plugin。
- 在非 `cli-metadata` / `tool-discovery` 路径注册 channel。
- 调用 `setRuntime(api.runtime)`。
- 在 `discovery` 和 `full` 注册 CLI metadata。
- 只有 `full` 执行 `registerFull(api)`。

`defineSetupPluginEntry()` 返回 `{ plugin }`，用于轻量 setup entry。

Bundled channel 常用更强的 `defineBundledChannelEntry()` / `defineBundledChannelSetupEntry()`，位于 `src/plugin-sdk/channel-entry-contract.ts`。它允许通过 specifier/exportName 延迟加载轻量 public artifacts，例如：

- channel plugin artifact
- runtime setter
- account inspect
- secret contract

`extensions/discord/index.ts` 是典型例子：entry 只声明 artifact specifier，`registerFull(api)` 只注册 Discord subagent hooks。

### 7.4 OpenClawPluginApi

`src/plugins/api-builder.ts` 构造 `OpenClawPluginApi`。API 上的注册方法覆盖：

- capability：`registerProvider`、`registerCliBackend`、`registerChannel`、speech、realtime、media understanding、image/music/video generation、web fetch/search、agent harness。
- tools/commands：`registerTool`、`registerCommand`。
- infrastructure：`registerHook`、`registerHttpRoute`、`registerGatewayMethod`、`registerGatewayDiscoveryService`、`registerCli`、`registerNodeCliFeature`、`registerService`、`registerInteractiveHandler`。
- host hooks：session extension、next-turn injection、trusted tool policy、tool metadata、Control UI descriptor、runtime lifecycle、agent event subscription、run context、scheduler job。
- memory：memory capability、prompt section/supplement、corpus supplement、flush plan、runtime、embedding provider。

`api.registrationMode` 是插件判断当前加载语境的公开字段。SDK helper 会帮常见插件处理 mode，手写 entry 则必须自己避免 discovery/setup 阶段启动 socket、worker、subprocess、网络客户端。

## 8. Channel plugin 架构

Channel plugin 的边界是：核心拥有通用消息 turn、session、shared message tool、delivery pipeline；插件拥有平台语义和平台 API。

典型 channel plugin 提供：

- `ChannelPlugin` 对象。
- config/schema/setup/status/probe/doctor。
- security、pairing、groups、threading、outbound。
- inbound monitor。
- message action adapter。
- runtime setter，注入 `PluginRuntime`。

`src/plugin-sdk/core.ts` 提供：

- `createChannelPluginBase()`
- `createChatChannelPlugin()`
- DM security helper
- pairing helper
- threading resolver
- outbound adapter helper

Channel 插件不需要为发送/编辑/反应都注册单独 agent tool。核心 shared `message` tool 在运行时调用 channel plugin 的 scoped action discovery 和 execution。插件负责 `describeMessageTool(...)`、action schema fragment、media source params、实际 API 调用。

`extensions/discord/openclaw.plugin.json` 声明：

- `id: "discord"`
- `activation.onStartup: false`
- `channels: ["discord"]`
- `channelEnvVars.discord = ["DISCORD_BOT_TOKEN"]`
- 空 plugin config schema

`extensions/discord/index.ts` 使用 `defineBundledChannelEntry()`，将 heavy runtime 分散在 `channel-plugin-api.js`、`runtime-setter-api.js`、`account-inspect-api.js` 等 narrow artifact 中。这是 channel 插件 lazy surface 的典型形态。

## 9. Provider plugin 架构

Provider 插件把厂商能力注册为多个 capability provider，而不是让核心硬编码厂商。

OpenAI 插件例子：

- manifest 声明 `providers: ["openai", "openai-codex"]`。
- `modelSupport.modelPrefixes` 用于 `gpt-`、`o1`、`o3`、`o4` 等 shorthand model 激活。
- `modelCatalog` 提供静态模型目录。
- `providerEndpoints` 将 `api.openai.com`、`chatgpt.com`、Azure OpenAI host suffix 映射到 endpoint class。
- `providerRequest.providers` 声明 OpenAI family。
- runtime 注册 text provider、Codex CLI backend、memory embedding、image/realtime/speech/media/video generation。

Provider 分层：

- 插件本地拥有 auth、catalog、onboarding、厂商 API 调用、特殊模型处理。
- 核心拥有通用 provider loop、fallback、auth profile store、model catalog 聚合、agent runtime dispatch。
- SDK 提供 family-level helper，例如 provider tool schema compat、request/transport/replay policy 复用。

Manifest 里 `contracts.*Providers` 是静态能力所有权快照，运行时 `api.register*Provider()` 是实际执行能力注册。二者要保持一致，契约测试负责守住边界。

## 10. Hook、Service、HTTP、CLI 和 Host Hook

插件不只注册 provider/channel。

### 10.1 Hooks

`api.registerHook(events, handler, opts)` 支持 legacy/internal hook。`api.on(hookName, handler, opts)` 支持 typed host hook。相关类型在 `src/plugins/hook-types.ts`、`src/plugins/types.ts`。

Legacy `before_agent_start` 仍兼容，但是旧模式。更推荐：

- `before_model_resolve`：模型/provider override。
- `before_prompt_build`：prompt mutation。
- phase hooks / typed hooks：更明确的生命周期能力。

`allowPromptInjection=false` 会阻止 prompt-mutating hooks 和 next-turn injection。

### 10.2 Service

`api.registerService(service)` 注册后台服务。服务只应在 full runtime 下启动。注册表记录 `origin`、`trustedOfficialInstall`、source、rootDir。Gateway startup/shutdown 负责启动和停止服务。

### 10.3 HTTP route 和 Gateway method

插件可注册：

- `api.registerHttpRoute({ path, handler, auth, match, handleUpgrade })`
- `api.registerGatewayMethod(name, handler, { scope })`

HTTP route 必须声明 auth 为 `gateway` 或 `plugin`。route overlap 不允许 auth 冲突。Gateway method 不能覆盖 core method；`config.*`、`exec.approvals.*`、`wizard.*`、`update.*` 等 reserved core admin namespace 会强制 `operator.admin`。

### 10.4 CLI

`api.registerCli(registrar, opts)` 支持：

- `commands`
- `descriptors`
- `parentPath`

`descriptors` 是 parse-time metadata，让 root command help/parse tree 可以知道插件命令，但真实 CLI module 保持 lazy。`api.registerNodeCliFeature()` 是 `parentPath: ["nodes"]` 的包装。

## 11. 配置校验和 Schema 合并

配置校验依赖 manifest schema，不依赖 runtime export schema。

`src/plugins/config-schema.ts` 提供：

- `buildJsonPluginConfigSchema()`
- `buildPluginConfigSchema()`
- `emptyPluginConfigSchema()`

Runtime helper 可以从 Zod schema 生成 JSON Schema，但 manifest 仍是控制面 source of truth。loader 在每个 enabled plugin 加载前调用 `validatePluginConfig()`：

- empty object schema 只允许空对象。
- 非空 schema 走 `validateJsonSchemaValue({ applyDefaults: true })`。
- 校验失败记录 plugin error，failure phase 为 `validation`。

`channelConfigs` 和 plugin config schema 会被 config/schema、Control UI、doctor/setup 等读取，从而做到“不执行插件代码也能提示配置形状”。

旧配置修复原则：runtime path 不做隐式 legacy migration。旧 config/store 兼容修复应放在 doctor/fix 或 setup migration。

## 12. 缓存与懒加载边界

插件系统有多层缓存，但职责不同：

- Manifest load LRU：只缓存单个 manifest 文件解析，绑定 stat 和安全参数。
- Persisted installed index：安装/doctor 生成，startup 可读，但 stale 时回退。
- Metadata snapshot：Gateway 当前 config/plugin inventory 的一次性控制面快照。
- Loader cache：按 load options cache key 缓存 runtime registry。
- Module loader cache：jiti/native loader 缓存。
- Plugin runtime lazy values：TTS、media understanding、model auth、generation runtime 等延迟加载。

禁止的方向是用 scattered cache 掩盖请求热路径反复 discovery。正确方向是把 provider/channel/tool owner 等 prepared facts 提前解析，并通过 `PluginMetadataSnapshot`、`PluginLookUpTable`、activation plan、onlyPluginIds、active runtime registry 传递。

## 13. 安全和所有权边界

插件系统的安全边界体现在多层。

### 13.1 文件系统边界

- Manifest、entry、setup entry 必须在 plugin root 内。
- 非 bundled 插件检查 owner uid。
- world-writable 路径阻断。
- managed npm/git/clawhub 安装只在 install/update/doctor 运行包管理器，Gateway runtime 不自动安装依赖。

### 13.2 SDK 边界

- 插件生产代码应 import `openclaw/plugin-sdk/*`，不要 deep import `src/**`。
- Bundled 插件也按第三方插件边界对待。
- 核心需要 bundled helper 时，应通过插件 `api.ts` / `runtime-api.ts` 或推广成通用 SDK subpath。
- 新 SDK surface 要同步 docs、exports、entrypoints、API baseline、contract tests。

### 13.3 Runtime 注册边界

- tool 必须声明 `contracts.tools`。
- agent tool result middleware、trusted tool policy 等是 bundled/trusted 能力。
- memory capability 必须由 memory plugin 或 contracts 声明。
- command/gateway/http/channel/provider id 冲突会产生 error diagnostic。
- async register 被拒绝，避免注册顺序和 side effects 不可控。

## 14. 契约测试

插件系统测试密集，体现了架构边界。

代表性测试路径：

- `src/plugins/contracts/loader.contract.test.ts`
- `src/plugins/contracts/registry.contract.test.ts`
- `src/plugins/contracts/shape.contract.test.ts`
- `src/plugins/contracts/plugin-sdk-subpaths.test.ts`
- `src/plugins/contracts/plugin-sdk-package-contract-guardrails.test.ts`
- `src/plugins/contracts/extension-package-project-boundaries.test.ts`
- `src/plugins/contracts/extension-runtime-dependencies.contract.test.ts`
- `src/plugins/contracts/package-manifest.contract.test.ts`
- `src/plugins/contracts/plugin-registration.*.contract.test.ts`
- `src/plugins/contracts/runtime-import-side-effects.contract.test.ts`
- `src/plugins/contracts/config-boundary-guard.test.ts`

这些测试不只是单元测试，还守住：

- bundled 插件不 deep import core internals。
- SDK export map 和 baseline 不漂移。
- plugin manifest/package metadata 合法。
- provider/channel/tool contracts 与 runtime registration 一致。
- lazy import 和 runtime side effects 不退化。
- dependency ownership 符合“插件 runtime deps 留在插件”的边界。

## 15. 典型链路图

```text
Config + env + workspace
        |
        v
plugins.load.paths / workspace / global / bundled / installed index
        |
        v
discoverOpenClawPlugins()
        |
        v
loadPluginManifestRegistry()
        |
        +-----------------------------+
        | 控制面                       |
        | PluginMetadataSnapshot       |
        | owner maps                   |
        | PluginLookUpTable            |
        | activation planner           |
        | config/schema/setup/catalog  |
        +-----------------------------+
        |
        v
loadOpenClawPlugins()
        |
        v
resolve registration plan
        |
        +-------------------+---------------------+
        | setup-only        | discovery/validate  | full
        v                   v                     v
setup entry only       non-activating registry    runtime entry
channel setup          capability snapshot        register(api)
                                                |
                                                v
                                      PluginRegistry
                                                |
                  +-----------------------------+-----------------------------+
                  v                             v                             v
           Gateway methods/routes        Channels/message tool         Agent/providers/tools
```

## 16. 代表性源码索引

- 插件架构文档：`docs/plugins/architecture.md`
- 插件 manifest 文档：`docs/plugins/manifest.md`
- 插件 SDK 文档：`docs/plugins/sdk-overview.md`
- 插件入口文档：`docs/plugins/sdk-entrypoints.md`
- 插件安装管理文档：`docs/tools/plugin.md`
- 插件边界说明：`src/plugins/AGENTS.md`
- SDK 边界说明：`src/plugin-sdk/AGENTS.md`
- 内置插件边界说明：`extensions/AGENTS.md`
- Manifest 类型和读取：`src/plugins/manifest.ts`
- Discovery：`src/plugins/discovery.ts`
- Manifest registry：`src/plugins/manifest-registry.ts`
- Metadata snapshot：`src/plugins/plugin-metadata-snapshot.ts`
- Activation planner：`src/plugins/activation-planner.ts`
- Enable/config state：`src/plugins/config-state.ts`
- Loader：`src/plugins/loader.ts`
- Registry：`src/plugins/registry.ts`
- Registry types：`src/plugins/registry-types.ts`
- Runtime registry loader：`src/plugins/runtime/runtime-registry-loader.ts`
- Plugin runtime：`src/plugins/runtime/index.ts`
- API builder：`src/plugins/api-builder.ts`
- Native entry helper：`src/plugin-sdk/plugin-entry.ts`
- Channel SDK helper：`src/plugin-sdk/core.ts`
- Channel entry contract：`src/plugin-sdk/channel-entry-contract.ts`
- Provider entry helper：`src/plugin-sdk/provider-entry.ts`
- OpenAI provider 插件：`extensions/openai/index.ts`
- OpenAI manifest：`extensions/openai/openclaw.plugin.json`
- Discord channel 插件：`extensions/discord/index.ts`
- Discord manifest：`extensions/discord/openclaw.plugin.json`
- Discord public API：`extensions/discord/api.ts`
- Discord runtime API：`extensions/discord/runtime-api.ts`

## 17. 关键理解

OpenClaw 插件机制的核心不是一个简单的 `plugins.forEach(import)`，而是一套 manifest-first、capability-oriented、runtime-lazy 的控制面和运行面体系。

最重要的实现机制有五个：

1. `openclaw.plugin.json` 是控制面事实源，支撑不执行代码的发现、配置、setup、owner lookup 和 activation。
2. `PluginMetadataSnapshot` / owner maps / activation planner 把“谁拥有哪个能力”提前计算出来，避免热路径 broad discovery。
3. `loadOpenClawPlugins()` 根据 registration plan 精细选择 `full`、`discovery`、`setup-only`、`setup-runtime`、`tool-discovery`、`cli-metadata` 行为。
4. `createPluginRegistry()` 是所有 runtime 能力注册和边界检查的统一关口。
5. `src/plugin-sdk/` 用窄 subpath 提供公开契约，保证 bundled 插件和第三方插件走同一边界。

因此，新增或修改插件能力时，正确路径通常是：

- 先在 manifest 声明静态 ownership、config schema、activation/setup metadata。
- 再通过 SDK entry helper 在 `register(api)` 中注册 runtime 能力。
- 如果核心需要新能力，优先添加通用、窄、文档化 SDK seam。
- 避免在核心热路径写死 bundled plugin id 或反复重新加载 broad registry。
- 用 contract tests 同步锁住 manifest、runtime registration、SDK exports 和 dependency ownership。
