# OpenClaw 技术架构理解

## 1. 项目定位

OpenClaw 是一个多渠道 AI Gateway。它把聊天渠道、控制端、节点设备、Agent 运行时、模型 Provider、工具、自动化任务和插件能力统一接入到一个常驻 Gateway 进程中。

核心目标不是单一聊天机器人，而是一个可扩展的个人/团队 AI 控制面：

- 多渠道消息入口：Telegram、WhatsApp、Discord、Slack、Signal、iMessage、Matrix、WebChat 等。
- 多客户端控制面：CLI、Control UI、macOS/iOS/Android/headless node。
- 多模型/多运行时：原生 Provider、CLI backend、PI embedded agent、Codex/OpenCode/ACP 等 harness。
- 插件化扩展：Provider、Channel、Tool、HTTP route、Gateway method、CLI command、Service、Hook、Memory、Media capability。
- 长驻服务能力：Gateway health、sessions、cron、heartbeat、pairing、approvals、logs、diagnostics、OpenAI-compatible HTTP API。

项目当前是 TypeScript ESM 主体，Node 22+ 运行，pnpm workspace 管理根包、`ui`、`packages/*` 和 `extensions/*`。发布入口是 `openclaw.mjs`，源码入口是 `src/entry.ts`，构建后运行 `dist/entry.js`。

## 2. 仓库分层

### 2.1 顶层模块

- `src/`：OpenClaw 核心。包含 CLI、Gateway、Agent、Channel 核心、Plugin loader、Config、Sessions、Tools、Media、Memory、TUI、Web server 等。
- `extensions/`：内置插件。每个插件按第三方插件边界组织，通过 `openclaw/plugin-sdk/*` 接入核心。
- `src/plugin-sdk/`：插件公共 SDK 源码，是核心和插件之间的公开契约。
- `packages/`：可独立发布或复用的包，例如 `packages/plugin-sdk`、`packages/plugin-package-contract`、`packages/memory-host-sdk`。
- `ui/`：Control UI，Vite + Lit 构建，Gateway 同端口静态托管。
- `apps/`：平台客户端，包括 Android、iOS、macOS、shared OpenClawKit 等。
- `docs/`：产品/架构/配置/插件/渠道/工具文档。
- `scripts/`：构建、测试、生成、发布、架构检查、插件清单等工程脚本。

### 2.2 核心源码子域

- `src/cli/`：命令行入口、Commander 命令注册、gateway fast path、容器/profile/env/bootstrap。
- `src/gateway/`：Gateway HTTP/WebSocket 服务、协议、鉴权、方法处理、节点会话、健康检查、Control UI、OpenAI-compatible API。
- `src/agents/`：Agent runtime plan、PI embedded runner、CLI backend bridge、sandbox、skills、tool wiring、auth profiles。
- `src/auto-reply/`：从入站消息到 Agent turn 的编排层，处理指令、队列、模型选择、流式回复、fallback、session 更新。
- `src/channels/`：渠道核心抽象，负责消息 turn、allowlist、mention gating、session、shared message tool、插件渠道 registry。
- `src/plugins/`：插件发现、manifest、启用策略、loader、registry、runtime state、activation planning、hook/service/http/CLI 注册。
- `src/config/`：JSON5 配置读写、schema 校验、runtime snapshot、热重载、last-known-good、SecretRef/env/include 处理。
- `src/gateway/protocol/`：TypeBox 协议 schema，生成 JSON Schema 和 Swift models。
- `src/media*`、`src/*-generation`、`src/*-search`、`src/tts`、`src/talk`：媒体、搜索、语音、实时会话等 capability 核心或路由层。

## 3. 启动链路

### 3.1 包入口

`openclaw.mjs` 是 npm bin。它负责：

- 校验 Node 版本至少 22.12。
- 区分源码 checkout 和打包产物。
- 管理 Node compile cache。
- 对根 help 做 fast path。
- 动态导入 `dist/entry.js` 或 `dist/entry.mjs`。

源码入口 `src/entry.ts` 继续做运行时 bootstrap：

- 安装 warning filter 和环境规范化。
- 处理 profile、container target、Windows argv。
- 设置进程 title 和 OpenClaw exec marker。
- 对 root help/version 做 fast path。
- 最终动态导入 `src/cli/run-main.ts` 的 `runCli()`。

### 3.2 CLI 编排

`src/cli/run-main.ts` 是 CLI 主调度层：

- 对 `openclaw gateway` 有单独 fast path，避免加载完整命令树，提高 Gateway 启动速度。
- 按需加载 dotenv、proxy dispatcher、Crestodian、plugin command metadata。
- 构造 Commander program，命令注册入口在 `src/cli/program/build-program.ts` 和 `src/cli/program/command-registry.ts`。

普通短命令走 CLI action；`gateway` 命令进入常驻服务启动路径。

## 4. Gateway 架构

Gateway 是 OpenClaw 的中心进程。文档和源码一致体现出几个核心原则：

- 单个长驻 Gateway 进程拥有消息渠道连接和控制面。
- HTTP、WebSocket、Control UI、OpenAI-compatible API、hooks、tools invoke 共用一个主端口，默认 `127.0.0.1:18789`。
- 控制客户端和节点都通过同一个 WebSocket server 连接，但节点在 handshake 中声明 `role: "node"` 和能力。
- Gateway 对外暴露 typed RPC methods 和 server-push events。

### 4.1 Gateway 启动阶段

`src/gateway/server.ts` 是懒加载壳，真正实现位于 `src/gateway/server.impl.ts`。`startGatewayServer()` 的主要启动顺序：

1. 读取 Gateway startup config snapshot。
2. 激活 SecretRef/runtime secrets。
3. 准备 auth、diagnostics、restart policy、control UI origin seed。
4. 设置 runtime config snapshot。
5. 构造 plugin metadata snapshot 和 `PluginLookUpTable`。
6. 解析 runtime bind/auth/TLS/control UI/OpenAI API 配置。
7. 创建 channel manager、readiness checker、HTTP server、WebSocket server、dedupe、chat run state 等 runtime state。
8. 启动 early runtime、插件、channels、cron、health/tick loop、post-ready maintenance。
9. 注册 shutdown/close prelude，清理插件、渠道、cron、MCP、rate limiter、WebSocket、HTTP server。

这个启动路径强调“先元数据、后运行时”：能用 manifest/lookup table 决策时，不加载重的插件 runtime。

### 4.2 Gateway 协议

协议定义在 `src/gateway/protocol/schema/*`，汇总出口是 `src/gateway/protocol/schema.ts`。顶层 frame 来自 `src/gateway/protocol/schema/frames.ts`：

- 连接首帧：`connect` params，包含协议版本、client identity、role、caps、commands、device signature、auth。
- 成功响应：`hello-ok`，包含 server connId、features.methods/events、snapshot、auth role/scopes、policy。
- 请求：`{ type: "req", id, method, params }`
- 响应：`{ type: "res", id, ok, payload | error }`
- 事件：`{ type: "event", event, payload, seq?, stateVersion? }`

Gateway method/event 清单的基础定义在 `src/gateway/server-methods-list.ts`。基础 methods 覆盖 health、config、plugins、sessions、agents、tools、channels、nodes、cron、secrets、message、send、agent、chat 等，channel plugin 还可以贡献额外 Gateway methods。

### 4.3 HTTP 和 Web Surface

Gateway 同时提供：

- WebSocket RPC/control plane。
- Control UI 静态资源，默认根路径 `/`，可配置 `gateway.controlUi.basePath`。
- OpenAI-compatible endpoints：`/v1/models`、`/v1/embeddings`、`/v1/chat/completions`、`/v1/responses`。
- `/tools/invoke` 等工具调用 HTTP API。
- hooks/webhooks 和插件 HTTP routes。

Control UI 位于 `ui/`，使用 Vite + Lit，构建产物由 Gateway 从 `dist/control-ui` 托管。

## 5. 配置系统

配置默认文件是 `~/.openclaw/openclaw.json`，格式为 JSON5。源码主要在 `src/config/io.ts`、`src/config/types.openclaw.ts` 和各 `types.*.ts`。

配置系统的关键设计：

- `OpenClawConfig` 是总配置类型，覆盖 auth、env、gateway、agents、models、channels、plugins、tools、skills、cron、hooks、memory、mcp、proxy 等。
- 读取时区分 source config、resolved source config 和 runtime config。
- 支持 `$include`、`${ENV}` 替换、SecretRef、runtime defaults。
- 写入使用 atomic replace，并记录 audit。
- schema 严格校验，未知键默认拒绝。
- Gateway 启动成功后会维护 last-known-good，但不会自动恢复坏配置；修复通过 doctor/fix。
- Gateway 热重载基于 runtime snapshot，成功 reload 原子替换，不成功则保留当前运行配置。
- 插件和渠道 schema 会通过 manifest registry 合并到配置校验和 Control UI schema。

## 6. Agent 执行架构

Agent 执行入口不是单一文件，而是由消息路由、reply dispatcher、runtime plan、model fallback、runner 共同组成。

### 6.1 入站消息到 Agent Turn

典型渠道入站路径：

1. Channel plugin 接收平台事件。
2. 插件把平台事件归一化为 channel turn。
3. `src/channels/turn/kernel.ts` 负责 admission、record、history、reply pipeline、durable delivery。
4. `src/auto-reply/reply/get-reply-run.ts` 构造 prompt/session/context/directives/queue 状态。
5. `src/auto-reply/reply/agent-runner-execution.ts` 的 `runAgentTurnWithFallback()` 执行模型选择、fallback、runtime 分派、流式回调和最终 payload 处理。

### 6.2 Runtime 分派

`runAgentTurnWithFallback()` 会根据 provider/model/session runtime override 判断运行方式：

- CLI provider：调用 `runCliAgent()`，面向 Claude CLI、Codex CLI、OpenCode、Gemini CLI 等 CLI backend。
- Embedded/PI runtime：调用 `runEmbeddedPiAgent()`，接入 `@mariozechner/pi-agent-core` 相关能力。
- Agent harness：通过 `resolveAgentHarnessPolicy()` 和 runtime override 选择更底层 executor。

运行时会传入：

- session id/key/file、workspace、agent id。
- provider/model/think level/auth profile。
- 来源渠道、账号、sender、group/thread 信息。
- images、media order、tools policy、sandbox session key。
- block streaming、reasoning stream、tool events、approval events、plan updates、command output 等回调。

### 6.3 Model fallback

模型选择遵循文档和代码中的分层：

- primary model。
- configured fallbacks。
- provider 内部 auth/profile failover。
- transient/classified error 后的 fallback candidate persistence。

用户显式 `/model` 选择是 strict override；自动 fallback 会记录 `modelOverrideSource: "auto"`，避免覆盖用户选择。

## 7. Channel 架构

`src/channels/**` 是核心渠道实现；插件作者不直接 import 这里，而通过 SDK subpath 使用 channel contract。

核心 channel 层负责：

- session key、conversation binding、thread binding。
- allowlist、pairing、group mention gating、command gating。
- shared message tool host。
- inbound turn lifecycle。
- outbound reply pipeline。
- durable delivery 和 receipt。
- channel plugin registry。

Channel plugin 负责：

- 平台 SDK/API 接入。
- 平台事件解析。
- 平台发送、编辑、反应、附件、thread/topic 语义。
- channel config schema、setup、status、probe、doctor。
- scoped message tool action discovery 和 execution。

Telegram 是典型例子：

- `extensions/telegram/openclaw.plugin.json` 声明 id、activation、channels、env vars、config schema。
- `extensions/telegram/index.ts` 用 `defineBundledChannelEntry()` 暴露轻量入口，引用 channel plugin、secret contract、runtime setter、account inspect 等 artifact。
- `extensions/telegram/src/channel.ts` 通过 SDK 组合 `createChatChannelPlugin()`、`createChannelMessageAdapterFromOutbound()`、directory/status/setup/security/probe/audit/message actions。
- 重发送逻辑、轮询/webhook、Bot API、thread/topic、inline buttons 等保留在插件内部。

这体现了项目的所有权边界：核心只拥有通用 channel contract，平台行为留在插件。

## 8. 插件系统总览

插件系统是项目的核心扩展机制，主要在 `src/plugins/` 和 `src/plugin-sdk/`。

### 8.1 插件四层

1. Manifest + discovery：发现 `openclaw.plugin.json`，来自 bundled plugins、configured paths、workspace/global plugin roots、installed index。
2. Enablement + validation：根据 config、默认启用、平台、slot、禁用/阻塞策略决定有效状态。
3. Runtime loading：加载 native plugin module，调用 `register(api)` 写入 registry。
4. Surface consumption：Gateway、Agent、Channel、CLI、HTTP、Tools、Hooks、Services 等读取 registry。

### 8.2 Manifest-first 控制面

`src/plugins/manifest.ts` 定义 manifest 元数据，包括：

- `providers`、`channels`、`modelSupport`、`modelCatalog`。
- `activation` hints。
- `setup.providers`、env vars、auth evidence。
- channel configs、secret paths、dangerous config flags。
- provider endpoint/request/model normalization/pricing metadata。

Gateway 启动会构建 `PluginMetadataSnapshot` 和 `PluginLookUpTable`。`src/plugins/plugin-lookup-table.ts` 将 installed index、manifest registry、owner maps 和 startup plan 组合为一次性 lookup table，避免热路径重复 broad discovery。

### 8.3 Runtime registry

`src/plugins/runtime.ts` 管理 active plugin registry：

- 全局 active registry 及 version。
- HTTP route registry 和 channel registry 的 pin/release。
- registry 替换时清理 host hook runtime。
- agent event bridge。
- imported plugin id 跟踪。

`src/plugins/loader.ts` 负责实际加载，包含：

- discovery/context fingerprint/cache。
- native module/Jiti 加载。
- manifest validation。
- activation、onlyPluginIds、validate/full mode。
- channel setup runtime、bundled artifacts、runtime cache。
- plugin API 构建和注册。

### 8.4 Plugin SDK

插件入口通常使用：

- `definePluginEntry()`：普通 provider/tool/service/hook 插件。
- `defineBundledChannelEntry()` / channel SDK helpers：渠道插件。
- provider/channel/core/runtime/config 等大量窄 subpath。

`src/plugins/api-builder.ts` 说明 `OpenClawPluginApi` 是注入式注册对象。插件可注册：

- Provider：text inference、CLI backend、speech、realtime、media understanding、image/music/video generation、web fetch/search。
- Channel：消息渠道。
- Tools/commands。
- Gateway methods/HTTP routes/discovery service。
- Hooks/services/runtime lifecycle。
- Session extension、next-turn injection、trusted tool policy、tool metadata、Control UI descriptor、agent event subscription。
- Memory capability、prompt/corpus supplement、embedding provider。

OpenAI 插件是 provider 插件例子：

- `extensions/openai/openclaw.plugin.json` 声明 provider ids、model support、catalog、endpoint、request family。
- `extensions/openai/index.ts` 用 `definePluginEntry()` 注册 openai/openai-codex provider、Codex CLI backend、memory embedding、image/realtime/speech/media/video generation。

## 9. Provider 和能力路由

Provider 能力被插件注册到 registry 后，由 Agent runtime 和工具层消费。项目把 provider 相关行为拆为：

- auth/profile resolution。
- model catalog/normalization/pricing。
- transport/replay/tool schema compatibility。
- runtime auth preparation。
- dynamic model/provider fallback。
- system prompt contribution。

共享行为通过 SDK family helper 复用，vendor-specific auth/catalog/onboarding 保留在插件内。这样可以避免核心把某个 provider 的规则硬编码进通用路径。

媒体能力同样插件化：image/music/video generation、media understanding、speech/realtime/web search/fetch 都是 capability provider，而不是核心写死单个服务商。

## 10. Sessions、队列和状态

OpenClaw 的运行状态围绕 session 展开：

- 消息渠道把会话映射为 session key。
- session store 记录模型 override、origin/delivery context、history、binding、compaction、usage 等。
- reply queue 处理活跃 run、steer、stop、followup、heartbeat、cron 等并发语义。
- Gateway 通过 `sessions.*` methods 和 events 暴露列表、订阅、preview、send、abort、patch、compact、reset/delete。

Agent 执行中会通过 `emitAgentEvent()` 发出 lifecycle/assistant/tool/item/plan/approval/command_output 等事件，Gateway 再投射给 Control UI/TUI/WebSocket 客户端或渠道 reply pipeline。

## 11. 节点与设备模型

Gateway WebSocket 不只服务控制客户端，也服务 node：

- node 在 `connect` 中声明 role、caps、commands、permissions、device signature。
- Pairing 是 device-based，approved device 拿 device token。
- node 可暴露 camera、screen、canvas、location、system.run 等能力。
- Gateway methods 包含 `node.*`、`device.*`、`node.invoke`、pending queue 和 result ack。

这让手机/桌面/headless 节点成为 Gateway 后面的能力执行端，而不是每个客户端直接访问本地资源。

## 12. 安全模型

项目安全边界主要体现在：

- Gateway auth 默认启用：token/password/trusted-proxy/Tailscale identity。
- 非 loopback bind 必须显式满足 auth/origin 要求。
- WebSocket handshake 必须先 connect，设备签名和 pairing 参与信任建立。
- DM/group 入口有 pairing、allowlist、mention gating。
- SecretRef 和 `~/.openclaw/credentials/` 避免把密钥直接散落到 config。
- Exec approvals、sandbox policy、elevated mode、node invoke policy 分层控制命令执行。
- 插件边界禁止 bundled plugin 直接 deep import 核心内部；通过 SDK 契约接入。
- Config strict schema 和 doctor/fix 承担迁移/修复，runtime 不做隐式 legacy 迁移。

## 13. 构建、测试和质量门禁

主要工具链：

- TypeScript ESM，`tsgo` lanes 做类型检查。
- `tsdown`/自定义 `scripts/build-all.mjs` 构建。
- Vitest 通过仓库 wrapper 运行。
- `oxfmt` 格式化，`oxlint` lint。
- TypeBox schema 生成 Gateway protocol JSON Schema 和 Swift models。
- 架构检查包括 import cycles、plugin SDK exports、config docs/schema、plugin inventory、dependency ownership、extension boundary 等。

常用脚本来源：

- `pnpm build`：完整构建。
- `pnpm check:changed`：按 changed lanes 选择验证。
- `pnpm test <path-or-filter>`：目标测试。
- `pnpm tsgo:*`：类型检查 lanes。
- `pnpm format:check` / `pnpm exec oxfmt --check --threads=1 <files>`：格式化验证。

## 14. 关键架构原则

### 14.1 Gateway 是控制面，插件是能力边界

核心保持 extension-agnostic。核心消费 manifest、SDK contract、registry、capability provider，不把具体 bundled plugin id、平台 API 或 vendor auth 规则写进热路径。

### 14.2 Manifest-first，runtime-lazy

配置校验、setup、owner lookup、activation planning、模型目录等尽量从 manifest 和 metadata snapshot 得到答案。真正执行时才加载插件 runtime。

### 14.3 Prepared facts 优先

请求热路径应携带已解析的 provider/channel/session/runtime facts，避免在 reply/tool/outbound/media 路径反复 broad discovery 或 registry materialization。

### 14.4 通用 contract 在核心，平台语义在插件

核心拥有 shared message tool、session、delivery pipeline、Gateway RPC、provider loop；插件拥有平台/厂商特殊行为。

### 14.5 修复旧状态走 doctor/fix

启动和运行热路径保持 canonical contract。旧配置、旧 store shape、迁移修复通过 doctor/fix 或 setup migration，而不是 runtime fallback。

## 15. 技术架构图

```text
                          +----------------------+
                          | openclaw.mjs / CLI   |
                          | src/entry.ts         |
                          +----------+-----------+
                                     |
                                     v
                          +----------------------+
                          | Gateway process      |
                          | HTTP + WebSocket     |
                          +----+------+-----+----+
                               |      |     |
             +-----------------+      |     +-------------------+
             v                        v                         v
   +-------------------+    +-------------------+     +-------------------+
   | Control clients   |    | Node devices      |     | HTTP surfaces     |
   | CLI / UI / TUI    |    | iOS/Android/etc.  |     | /v1/* /tools/etc. |
   +-------------------+    +-------------------+     +-------------------+
                               ^
                               |
 +-----------------------------+--------------------------------+
 |                                                              |
 v                                                              v
+--------------------+      +--------------------+      +--------------------+
| Channel plugins    | ---> | Channel turn core  | ---> | Auto reply / Agent |
| Telegram/Slack/... |      | session/delivery   |      | runtime/fallback   |
+--------------------+      +--------------------+      +---------+----------+
                                                                 |
                                                                 v
                                                       +--------------------+
                                                       | Provider plugins   |
                                                       | models/media/tools |
                                                       +--------------------+

 Supporting plane:
 - Config/runtime snapshots
 - Plugin manifest metadata + lookup table
 - Plugin runtime registry
 - Sessions/store/queue
 - Secrets/approvals/sandbox
 - Diagnostics/logging/health
```

## 16. 代表性源码索引

- CLI bin wrapper：`openclaw.mjs`
- 源码入口：`src/entry.ts`
- CLI 主调度：`src/cli/run-main.ts`
- CLI program：`src/cli/program/build-program.ts`
- Gateway lazy shell：`src/gateway/server.ts`
- Gateway 实现：`src/gateway/server.impl.ts`
- Gateway method/event 清单：`src/gateway/server-methods-list.ts`
- Gateway frame schema：`src/gateway/protocol/schema/frames.ts`
- Config IO：`src/config/io.ts`
- Config 总类型：`src/config/types.openclaw.ts`
- Channel turn kernel：`src/channels/turn/kernel.ts`
- Reply run 构造：`src/auto-reply/reply/get-reply-run.ts`
- Agent fallback 执行：`src/auto-reply/reply/agent-runner-execution.ts`
- Agent runtime plan types：`src/agents/runtime-plan/types.ts`
- Plugin loader：`src/plugins/loader.ts`
- Plugin runtime registry：`src/plugins/runtime.ts`
- Plugin lookup table：`src/plugins/plugin-lookup-table.ts`
- Plugin manifest 类型：`src/plugins/manifest.ts`
- Plugin API builder：`src/plugins/api-builder.ts`
- Plugin SDK entry helper：`src/plugin-sdk/plugin-entry.ts`
- Telegram 插件入口：`extensions/telegram/index.ts`
- Telegram channel 实现：`extensions/telegram/src/channel.ts`
- OpenAI provider 插件入口：`extensions/openai/index.ts`
- Control UI 包：`ui/package.json`

## 17. 后续阅读建议

本文件覆盖项目整体技术架构。若继续拆分专题，建议按以下顺序深入：

1. 插件系统：`src/plugins/loader.ts`、`src/plugins/registry.ts`、`src/plugin-sdk/*`、`docs/plugins/architecture.md`。
2. Gateway 协议和方法：`src/gateway/protocol/schema/*`、`src/gateway/server-methods/*`。
3. Agent loop：`src/auto-reply/reply/*`、`src/agents/pi-embedded-runner/*`。
4. Channel SDK：`src/channels/plugins/*`、`src/channels/turn/*`、`extensions/telegram/src/channel.ts`。
5. Config/schema：`src/config/*`、`docs/gateway/configuration-reference.md`。
