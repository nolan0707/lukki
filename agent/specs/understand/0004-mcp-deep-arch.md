# Claude-code-open MCP 深度架构解读

## 1. 文档目标

本文面向 AI 应用架构师、Agent 系统设计者、算法平台工程师，不再停留在 “MCP 是外部工具接入” 这一层，而是深入回答以下问题：

1. Claude Code 为什么把 MCP 设计成一个独立子系统，而不是简单挂接若干 RPC client？
2. 这套 MCP 子系统的接口边界、状态机、数据结构、模块交互机制分别是什么？
3. 它如何处理现实世界中的复杂问题：多来源配置、权限治理、OAuth、动态能力发现、会话失效、结果过大、热更新、插件/claude.ai/SDK 并存？
4. 这套设计里哪些部分体现了工业级工程取舍，哪些部分具有方法论上的创新价值？

本文参考以下已有理解文档：

- `./specs/understand/0001-project-arch.md`
- `./specs/understand/0004-mcp-arch.md`

并基于以下核心代码：

- `vendor/Claude-code-open/src/services/mcp/types.ts`
- `vendor/Claude-code-open/src/services/mcp/config.ts`
- `vendor/Claude-code-open/src/services/mcp/client.ts`
- `vendor/Claude-code-open/src/services/mcp/useManageMCPConnections.ts`
- `vendor/Claude-code-open/src/services/mcp/auth.ts`
- `vendor/Claude-code-open/src/services/mcp/headersHelper.ts`
- `vendor/Claude-code-open/src/services/mcp/claudeai.ts`
- `vendor/Claude-code-open/src/services/mcp/elicitationHandler.ts`
- `vendor/Claude-code-open/src/services/mcp/mcpStringUtils.ts`
- `vendor/Claude-code-open/src/services/mcp/utils.ts`
- `vendor/Claude-code-open/src/services/mcp/InProcessTransport.ts`
- `vendor/Claude-code-open/src/services/mcp/SdkControlTransport.ts`
- `vendor/Claude-code-open/src/services/tools/toolExecution.ts`
- `vendor/Claude-code-open/src/utils/plugins/mcpPluginIntegration.ts`

---

## 2. 总体判断

Claude Code 中的 MCP 不是“协议支持模块”，而是一个完整的 **外部能力接入控制平面**。

它的设计目标不是仅仅满足“能够调用 MCP server”，而是同时满足以下 6 个工程目标：

1. 统一接入多种外部能力源：本地进程、远程服务、claude.ai connector、SDK 内嵌 server、插件派生 server。
2. 把外部协议对象标准化为内部 Agent Runtime 可消费的 `Tool` / `Command` / `Resource` 抽象。
3. 在不可信配置与混合信任域下，保持权限、审批、workspace trust、组织策略的完整约束。
4. 将连接、认证、缓存、重连、动态更新等“长生命周期问题”封装在独立运行时层，不污染 Query Engine 主链路。
5. 允许 MCP 结果无缝进入模型上下文，但又能控制上下文体积、二进制内容、结构化结果的可治理性。
6. 在高扩展性场景下，控制复杂度：多来源配置合并、能力去重、连接并发调度、事件通知刷新、状态批量更新。

因此，Claude Code 的 MCP 架构本质上是：

```text
外部能力接入控制平面
  = 配置编排
  + 连接与认证
  + 能力发现
  + 内部抽象适配
  + 运行期状态维护
  + 错误恢复与上下文治理
```

这比“客户端封装一个 SDK”高了一个系统层级。

---

## 3. 设计原则

从代码结构和实现细节看，这套 MCP 架构遵循了几个很明确的原则。

## 3.1 外部协议对象必须内部化

MCP server 原生暴露的是：

- tools
- prompts
- resources
- notifications
- auth flow

Claude Code 不让这些对象直接穿透到 Query Engine，而是通过适配层转成：

- `Tool`
- `Command`
- `ServerResource`
- `MCPServerConnection`

这意味着：

- 上层不依赖 MCP SDK 的具体类型系统
- 权限系统、UI、telemetry、tool loop 可以复用既有基础设施
- 协议升级的影响被压缩在 `services/mcp/*`

这是典型的“协议对象内聚，业务对象外露”的设计。

## 3.2 配置域与运行域解耦

`config.ts` 解决的是：

- server 应不应该出现
- server 从哪里来
- server 能否被策略允许

`client.ts` / `useManageMCPConnections.ts` 解决的是：

- server 如何连接
- 连接后如何发现能力
- 运行时如何重连与刷新

这是非常关键的分层。否则连接逻辑会被配置优先级、审批、策略、缓存等规则污染。

## 3.3 远程能力必须被当作“不稳定资源”处理

代码没有把远程 MCP 当作稳定函数调用，而是系统性地处理了：

- 401 / token 失效
- OAuth discovery 失败
- 会话过期
- transport 中断
- 大结果污染上下文
- 动态 list_changed
- 重复配置导致的重复连接

这说明设计假设是：

**MCP 是一个半可信、长生命周期、可能失败的分布式依赖。**

这正是工业级 Agent 系统对外部工具的正确建模方式。

---

## 4. 架构分层

Claude Code 的 MCP 子系统可以拆成 6 层。

```text
L1 配置与策略层
  types.ts
  config.ts
  utils/plugins/mcpPluginIntegration.ts
  claudeai.ts

L2 连接与传输层
  client.ts
  InProcessTransport.ts
  SdkControlTransport.ts

L3 认证与请求增强层
  auth.ts
  headersHelper.ts

L4 能力发现与适配层
  client.ts 中 fetchToolsForClient / fetchCommandsForClient / fetchResourcesForClient
  mcpStringUtils.ts
  normalization.ts

L5 运行期控制层
  useManageMCPConnections.ts
  MCPConnectionManager.tsx
  elicitationHandler.ts

L6 Query/Tool 执行集成层
  toolExecution.ts
  tools/MCPTool/*
```

这 6 层之间的数据流是单向收敛的：

```text
配置与策略
  -> 建立连接
  -> 发现能力
  -> 适配成内部对象
  -> 注入运行时状态
  -> 进入 Query Engine 与工具执行链
```

---

## 5. 核心接口与数据结构

## 5.1 配置接口：`McpServerConfig`

`types.ts` 中通过 Zod 定义了统一配置联合类型：

```ts
type McpServerConfig =
  | McpStdioServerConfig
  | McpSSEServerConfig
  | McpSSEIDEServerConfig
  | McpWebSocketIDEServerConfig
  | McpHTTPServerConfig
  | McpWebSocketServerConfig
  | McpSdkServerConfig
  | McpClaudeAIProxyServerConfig
```

这一定义有两个重要意义：

1. 它把 transport 选择上升为显式协议分支，而不是靠若干可选字段隐式推断。
2. 它为连接工厂 `connectToServer()` 提供了标准的 discriminated union 输入。

其中最重要的几类配置：

### `stdio`

```ts
{
  type?: 'stdio'
  command: string
  args: string[]
  env?: Record<string, string>
}
```

### `sse` / `http` / `ws`

```ts
{
  type: 'sse' | 'http' | 'ws'
  url: string
  headers?: Record<string, string>
  headersHelper?: string
  oauth?: {
    clientId?: string
    callbackPort?: number
    authServerMetadataUrl?: string
    xaa?: boolean
  }
}
```

### `sdk`

```ts
{
  type: 'sdk'
  name: string
}
```

### `claudeai-proxy`

```ts
{
  type: 'claudeai-proxy'
  url: string
  id: string
}
```

设计上的关键点：

- 配置结构不仅描述 transport，也显式编码认证能力、动态 header 生成能力、回调端口等运行期要求。
- 这使 `config.ts` 和 `client.ts` 之间的接口是稳定且强类型的。

## 5.2 来源增强配置：`ScopedMcpServerConfig`

```ts
type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  pluginSource?: string
}
```

这个类型非常关键，因为它把“连接配置”扩展为“治理对象”。

新增的两个字段不是附属元数据，而是系统决策所需：

- `scope` 决定优先级、显示标签、策略过滤、权限 telemetry。
- `pluginSource` 用于 channel gate、插件来源识别、跨插件行为控制。

换句话说：

**`McpServerConfig` 是 transport contract，`ScopedMcpServerConfig` 是 control-plane contract。**

## 5.3 运行期连接状态：`MCPServerConnection`

```ts
type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

它不是简单的 “Result<Connected, Error>”，而是一个显式状态机。

### `connected`

携带：

- `client: Client`
- `capabilities`
- `serverInfo`
- `instructions`
- `config`
- `cleanup()`

### `pending`

携带：

- `reconnectAttempt`
- `maxReconnectAttempts`

说明 pending 不是纯展示态，而是自动重连状态机的一部分。

### `needs-auth`

说明 server 本身配置有效，但当前 credential path 不可用。

这个状态与 `failed` 不同。它是可恢复的、用户可操作的状态。

### `disabled`

是逻辑禁用，而不是连接失败。它直接影响配置合并与启动连接计划。

因此 `MCPServerConnection` 的设计价值在于：

- 把错误类型细化成产品可操作状态
- 允许 UI、Query、telemetry、重连逻辑共享同一状态定义

## 5.4 序列化状态：`MCPCliState`

```ts
interface MCPCliState {
  clients: SerializedClient[]
  configs: Record<string, ScopedMcpServerConfig>
  tools: SerializedTool[]
  resources: Record<string, ServerResource[]>
  normalizedNames?: Record<string, string>
}
```

它说明 MCP 子系统被当作一个独立 UI 面板来管理，而不是散落在全局状态中。

这对于：

- `/mcp` 管理界面
- 非交互模式状态回传
- SDK/print 模式展示

都很重要。

---

## 6. 模块交互机制

## 6.1 总体交互图

```text
config.ts
  -> 产出 Record<string, ScopedMcpServerConfig>

useManageMCPConnections.ts
  -> 调 getClaudeCodeMcpConfigs/getAllMcpConfigs
  -> 调 getMcpToolsCommandsAndResources

client.ts
  -> connectToServer
  -> fetchToolsForClient / fetchCommandsForClient / fetchResourcesForClient
  -> 产出 Tool[] / Command[] / ServerResource[]

AppState.mcp
  -> 保存 clients/tools/commands/resources

toolExecution.ts
  -> 运行时根据 mcp__server__tool 定位连接
  -> 执行 MCP tool call
  -> 错误回写 AppState.mcp
```

这里最重要的一点是：

**MCP 子系统与 Query Engine 不是直接耦合，而是通过统一 Tool/Command 集合耦合。**

这让上层只看到“可用能力集合”，看不到 transport 和 auth 细节。

## 6.2 配置编排与运行期调度是异步解耦的

`useManageMCPConnections.ts` 采用两阶段加载：

1. 先加载 Claude Code 本地侧 config
2. 后加载 claude.ai connectors

原因不是代码组织习惯，而是一个明确的系统优化：

- 本地配置读盘快，是启动关键路径
- claude.ai connectors 需要网络请求，不应阻塞本地能力可用

这说明 MCP 配置读取层已经具备了 **性能分层意识**。

## 6.3 动态 server 与静态 server 的统一接入

系统需要同时处理：

- 静态配置 server
- runtime 注入的 `dynamicServers`
- plugin 派生 server
- claude.ai 返回的 connector
- SDK 进程内 server

Claude Code 的做法不是为每类 server 单独写一套能力注入逻辑，而是：

1. 尽可能先变成 `ScopedMcpServerConfig`
2. 进入统一的连接与发现主链路
3. 特殊 transport 再在连接工厂中分支

这是一种标准的“上游异构、下游同构”设计。

---

## 7. 三条核心状态机

MCP 子系统里实际上存在 3 条状态机。

## 7.1 配置状态机

```text
配置出现
  -> schema 校验
  -> env 展开
  -> enterprise/policy 过滤
  -> project 审批过滤
  -> 重复签名去重
  -> 成为可连接 server
```

这里的重点是：

- “配置存在” 不等于 “最终可连接”
- 中间经历了安全、治理、审批、去重等多个过滤阶段

## 7.2 连接状态机

```text
disabled
  <-> pending
  -> connected
  -> needs-auth
  -> failed

connected --onclose--> pending(reconnect)
pending --success--> connected
pending --exhausted--> failed
```

这是 `MCPServerConnection` 与 `useManageMCPConnections.ts` 共同维护的状态机。

它的关键特征是：

- `needs-auth` 被建模成一等状态
- `pending` 承载自动重连元数据
- `disabled` 是逻辑开关，不是错误

## 7.3 单次工具调用状态机

```text
ensureConnectedClient
  -> client.callTool
  -> 结果转换
  -> 大结果治理
  -> 回填模型

异常分支：
  401 -> needs-auth
  session expired -> clear cache -> retry
  URL elicitation -> 人机交互 -> retry
  tool result isError -> McpToolCallError
```

这条状态机的特点是：

- 它不是纯粹的 RPC 调用，而是带交互、重试、状态回写的流程控制器。

---

## 8. 配置控制平面深度分析

## 8.1 多来源配置合并不是 key merge，而是治理 merge

`config.ts` 的复杂度主要不在“读文件”，而在“治理”。

系统同时考虑了：

- enterprise 是否独占
- policy allowlist / denylist
- project `.mcp.json` 是否已审批
- plugin server 是否与手工配置重复
- claude.ai connector 是否与手工配置重复
- server 是否被逻辑禁用

这说明它的真正功能不是 config parsing，而是：

**把多信任域、多来源的 server 候选集收敛为最终执行计划。**

## 8.2 内容签名去重：一个非常关键的工业设计

`getMcpServerSignature()` 的策略是：

- 对 `stdio`，签名来自 `command + args`
- 对 remote server，签名来自 `url`
- 忽略 `env`、`headers`

这背后的设计非常成熟：

### 为什么不能只按 server name 去重

- 插件 server 与手工 server 可能名字不同，但本质上是同一个进程或同一个 URL。

### 为什么忽略 env / headers

- 对“是否代表同一能力源”而言，核心身份是进程命令或 URL。
- headers 更多反映 credential 差异，不应让同一 server 在能力平面重复出现。

### 为什么要处理 CCR 代理 URL unwrap

- 在 remote session 场景中，同一个 vendor URL 可能被重写成代理 URL。
- 如果不解包，就会把同一实际 server 误判为两个不同连接目标。

这是一个典型的“基础设施代理层对上层身份建模造成扰动”的工业问题，而这里已经显式解决。

## 8.3 project server 审批：把配置输入当执行权限入口

`getProjectMcpServerStatus()` 的存在说明项目级 `.mcp.json` 不是普通静态配置。

因为对于 `stdio` server，配置本身就意味着：

- 启动本地命令
- 读取本地环境变量
- 建立网络连接

所以它本质上是一个执行权限入口，而不是“文档配置”。

Claude Code 将其纳入审批层，是非常正确的威胁建模。

## 8.4 plugin-only / managed-only 约束

`isRestrictedToPluginOnly('mcp')` 与 `allowManagedMcpServersOnly` 说明：

- 组织策略可以限制“用户是否能自行注入 MCP”
- 允许与禁止不是简单布尔，而是区分：
  - 谁可以声明 server
  - 谁的 allowlist 生效
  - denylist 是否仍保留用户本地否决权

这是一种成熟的“多租户企业治理”设计。

---

## 9. 连接层与 transport 抽象

## 9.1 `connectToServer()` 是多 transport 工厂

`client.ts` 中的 `connectToServer()` 本质是：

```text
McpServerConfig -> Transport + Client + cleanup + capabilities
```

它统一处理：

- stdio subprocess transport
- SSE transport
- Streamable HTTP transport
- WebSocket transport
- IDE transport
- claude.ai proxy transport
- in-process transport
- SDK control transport

这是 MCP 子系统最关键的“平台适配器”。

## 9.2 本地 / 远程分组调度是实际性能优化

`getMcpToolsCommandsAndResources()` 将连接任务分为：

- local servers
- remote servers

并使用不同并发度。

这反映的不是代码风格，而是对资源瓶颈的建模：

- 本地 server 的瓶颈是进程启动、CPU、文件系统、环境初始化
- 远程 server 的瓶颈主要是网络 IO

把它们分组调度，能够显著减少启动阶段 head-of-line blocking。

## 9.3 memoized 连接缓存是必须的

如果每次 tool call 都重新建 MCP client，会造成：

- OAuth 重复握手
- stream session 丢失
- transport 初始化高成本
- capability 再发现

所以 Claude Code 使用：

- `connectToServer` memoization
- `getServerCacheKey`
- `clearServerCache`
- `ensureConnectedClient`

形成完整的连接生命周期管理。

这意味着 MCP connection 被建模成 **长寿命对象**，而不是纯函数调用。

## 9.4 SDK transport bridge：很有代表性的工业扩展点

`SdkControlTransport.ts` 体现了一个很典型但不常见的设计：

- MCP server 不一定在网络或子进程里
- 它也可以在 SDK 进程内运行
- CLI 侧仍然希望复用同一个 MCP Client / Tool 适配逻辑

于是设计了 control channel bridge：

```text
CLI MCP Client
  -> SdkControlClientTransport
  -> control message
  -> SDK process
  -> SdkControlServerTransport
  -> real MCP server
```

这是一种非常优雅的“协议复用而非逻辑重写”做法。

同理，`InProcessTransport.ts` 则用于完全进程内的 linked transport pair。

这两个模块一起说明：

**Claude Code 的 transport abstraction 足够强，能够把“位置不同的 server”统一到同一 MCP runtime 里。**

---

## 10. 认证子系统深度分析

## 10.1 认证不是附属逻辑，而是 MCP 主路径的一部分

对 MCP 而言，认证不是可选功能，因为：

- 远程 connectors 常常受 OAuth 保护
- 认证失败会直接改变 server 的运行状态
- token 刷新、revocation、step-up scope 都会反馈到连接与工具调用路径

所以 `auth.ts` 实际上是 MCP runtime 的一部分，而不是工具类。

## 10.2 `ClaudeAuthProvider` 是 MCP SDK 与本地凭证系统的桥

它扮演的角色是：

- 对外实现 `OAuthClientProvider`
- 对内封装 secure storage、discovery state、redirect URI、token refresh、step-up scope 持久化

也就是说，它是一个 adapter：

```text
MCP SDK OAuth interface
  <-> Claude Code secure storage / local browser flow / XAA flow
```

这个类的意义在于：MCP SDK 不需要知道 Claude Code 的持久化和 UI 机制，Claude Code 也不需要直接侵入 SDK 内部。

## 10.3 认证路径不是一条，而是三条

系统实际支持至少三种认证路径：

### A. 常规 OAuth 授权码流程

`performMCPOAuthFlow()`：

- 本地起 callback server
- 调 `sdkAuth()`
- 打开浏览器
- 获取 authorization code
- 完成 token exchange

### B. XAA / SEP-990

`performMCPXaaAuth()`：

- 先获取共享 IdP `id_token`
- 再走跨应用 token exchange
- 最终把 token 写回与普通 OAuth 相同的存储位

这是典型的“统一 credential sink，不统一 auth source”设计。

### C. claude.ai proxy bearer 注入

`createClaudeAiProxyFetch()`：

- 为 proxy 请求附加 claude.ai OAuth token
- 遇到 401 时执行强制刷新重试

这不是标准 MCP auth provider，而是针对 claude.ai connector 的专用 auth pipe。

## 10.4 OAuth 细节处理很成熟

`auth.ts` 中有很多非常工业化的细节：

### 1. 2xx 但 body 内为 OAuth error 的兼容处理

`normalizeOAuthErrorBody()` 解决的是像 Slack 这样“HTTP 200 但 JSON body 表示错误”的兼容问题。

这不是协议 happy path，而是真实世界 OAuth 供应商差异的处理。

### 2. non-standard invalid_grant alias 归一化

把：

- `invalid_refresh_token`
- `expired_refresh_token`
- `token_expired`

统一映射到 `invalid_grant`。

这样后续刷新失败分类与凭证失效逻辑才能稳定工作。

### 3. path-aware metadata discovery

`fetchAuthServerMetadata()` 在 RFC 9728/8414 discovery 失败后，还保留 path-aware fallback。

这是为了兼容“元数据挂在路径级别”的旧式服务。

### 4. revocation 兼容策略

`revokeToken()` 先按 RFC 7009 标准做，401 时再 fallback 到 Bearer。

说明系统不是天真地假设 server 合规，而是用兼容机制换取更高实际可用性。

## 10.5 负缓存与 silent skip：非常高价值的启动优化

`client.ts` 中的 `mcp-needs-auth-cache.json` 和 `hasMcpDiscoveryButNoToken()` 很值得单独强调。

它们解决的是一个常见但隐蔽的问题：

- 某些 server 已知需要 auth
- 用户还没有完成授权
- 如果每次启动都去连一次，它们都会稳定 401

结果就是：

- 大量无意义网络请求
- 启动变慢
- print / non-interactive 模式被拖住

Claude Code 的解决方案：

1. 把 “recently needs-auth” 记到本地缓存，带 TTL
2. 如果已经有 discovery state 但没有 token，则直接跳过连接
3. 在 UI 中把它标成 `needs-auth` 并注入 `McpAuthTool`

这是一种非常成熟的 **negative capability caching** 设计。

---

## 11. 请求增强与边界安全

## 11.1 `headersHelper` 是动态 header 扩展点

有些远程 MCP 需要临时 header，不能直接静态写死。

于是引入：

```ts
headersHelper: string
```

运行时执行外部脚本，生成动态 header。

这本身很强大，但也带来风险：

- project/local config 可能来自不可信仓库
- 提前执行 helper 就可能绕过 trust 体系

所以 `headersHelper.ts` 明确加了：

- workspace trust 检查
- 非交互模式豁免
- 执行失败不阻塞连接，只返回 null

这是很经典的“高灵活扩展点 + 最小必要安全护栏”设计。

## 11.2 请求包装层不是装饰器，而是稳定性保障层

`client.ts` 为不同 transport 的 fetch path 增加了多个 wrapper：

- `wrapFetchWithTimeout`
- `wrapFetchWithStepUpDetection`
- `createClaudeAiProxyFetch`

这些 wrapper 不是“代码洁癖”，而是为了解决真实问题：

- stale timeout signal
- 403/step-up 未被持久化
- 401 token 刷新竞争
- headers 在 spread/merge 中丢失

这类实现说明作者不是只读了协议文档，而是把生产故障模式编码进了 transport 层。

---

## 12. 能力发现与内部抽象适配

## 12.1 Claude Code 的关键创新：把 MCP 变成统一 Tool/Command

真正的创新点不在“连上 MCP”，而在于：

**把 MCP 的异构能力收敛进 Claude Code 已有的 Agent Runtime 抽象。**

### 工具适配

`fetchToolsForClient()` 会把 MCP tool 转为内部 `Tool`，并附加：

- 可读写 / destructive / openWorld / searchHint 等语义
- `checkPermissions()` 对接本地权限体系
- `call()` 进入 Claude Code 的工具执行规范
- `userFacingName()` 对接 UI

### prompt 适配

`fetchCommandsForClient()` 把 MCP prompt 转为内部 `Command`。

### resources 适配

resources 不直接变工具，而是通过内置：

- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

让模型以统一工具方式使用资源。

这三种适配使 MCP 能力不再是旁路插件，而是 Agent Runtime 的一等成员。

## 12.2 命名空间设计是权限与可观测性的基础

工具名标准化为：

```text
mcp__<normalized_server>__<normalized_tool>
```

这带来三重价值：

1. 避免与内置工具冲突
2. 权限规则可精确匹配到 server+tool
3. telemetry 可以在 server/tool 维度归因

这是非常小但极重要的设计。很多系统在这一层偷懒，最后权限与日志都做不准。

## 12.3 `mcpInfoFromString()` 让名字成为路由协议

Claude Code 不是把工具名只当 display name，而是把它当 routing key。

工具调用时：

1. `toolExecution.ts` 解析 `mcp__server__tool`
2. 在当前 `mcpClients` 中按 normalized server name 找连接
3. 再调用真实 MCP client

因此，命名规范本身就是一套内部路由协议。

---

## 13. 运行期控制平面

## 13.1 `useManageMCPConnections()` 的角色

这个 hook 不是“拿数据给 UI 展示”，而是 MCP runtime 的控制器。

它负责：

- 初始化 server 状态
- 执行两阶段连接
- 把结果批量写入 `AppState.mcp`
- 注册 `onclose` 与 `list_changed`
- 驱动自动重连
- 提供手工 reconnect / toggle 能力

如果没有这一层，`client.ts` 只能做“连接一次”，无法支撑动态 Agent 运行期。

## 13.2 状态批处理是必要的，不是优化边角

多个 server 连接返回时，如果每个都直接更新 AppState，会导致：

- 高渲染频率
- 状态拼装分裂
- UI 过度抖动

所以系统用了：

- `pendingUpdatesRef`
- 16ms flush window

把 MCP 子系统的状态变更合并提交。

这说明作者已经把 MCP 当成“会产生成批运行事件的控制平面”，而不是零星状态源。

## 13.3 stale plugin client 清理体现了热更新意识

`excludeStalePluginClients()` 与 plugin reload 逻辑一起，解决了一个很典型的问题：

- 插件已经卸载或其 server 配置已变
- 旧连接和旧工具却还留在内存里

系统会：

1. 比较 config hash
2. 找出 stale client
3. 清理连接与缓存
4. 再把新配置作为 pending client 加回

这实际上是一种轻量级的热重配置机制。

## 13.4 自动重连采用指数退避是必要的

远程 transport 的 `onclose` 不直接进入 `failed`，而是：

- 变为 `pending`
- 带 attempt 元数据
- 使用指数退避重连

这说明系统假设：

- 网络中断是常态
- 失败不应立即暴露为最终态
- UI 需要区分“正在恢复”和“已经失败”

这对于 Agent 长任务尤其关键。

---

## 14. 动态通知与增量刷新

## 14.1 `list_changed` 处理非常关键

MCP server 的能力不是静态的。

有些 server 会在运行中：

- 增加新 tools
- 删除 prompts
- 更新 resources

Claude Code 对应做法是：

1. 注册 notification handler
2. 删除对应 memoized cache
3. 重新 fetch
4. 覆盖 AppState 中属于该 server 的能力集合

这一机制非常重要，因为它避免了：

- UI 和真实 server 能力集不一致
- 模型持有过时工具清单
- 资源/skills 目录陈旧

## 14.2 skills 从 resources 派生，是一个更高层的抽象复用

如果打开 MCP skills，系统会从 resource 层派生出 skills。

这说明在 Claude Code 的设计里：

- resource 不只是“内容文件”
- 它也可以成为更高层语义能力的载体

这是非常典型的 Agent 生态设计思路：在协议基础能力上再做语义升阶。

---

## 15. 工具调用路径深度分析

## 15.1 MCP tool call 不是简单的 SDK 透传

调用主链路：

```text
Tool.call()
  -> ensureConnectedClient()
  -> callMCPToolWithUrlElicitationRetry()
  -> callMCPTool()
  -> processMCPResult()
```

每一层解决的是不同问题：

- `ensureConnectedClient`：连接生存性
- `callMCPToolWithUrlElicitationRetry`：交互式错误恢复
- `callMCPTool`：真正的 MCP SDK 调用与错误分类
- `processMCPResult`：上下文治理

这不是 over-engineering，而是对现实问题的必要分层。

## 15.2 URL elicitation：把工具调用扩展为“交互子工作流”

普通工具系统一般假设 tool call 是封闭操作：

```text
input -> tool -> output
```

但 MCP 里出现了 URL elicitation，这意味着：

```text
input -> tool -> 需要用户打开 URL -> 等待确认 -> retry -> output
```

Claude Code 的做法是：

- 将其抽象为可重试状态
- 先交给 hook 程序化处理
- 再交给 REPL/UI 或 structured IO
- 由完成通知推进第二阶段

这说明 MCP tool call 在这里已经升级为一个 **可中断、可恢复、人机协作的微工作流**。

对 AI 应用架构师来说，这是非常值得借鉴的模式。

## 15.3 错误语义分类是调用链稳定性的关键

`callMCPTool()` 区分了：

- 普通 Error
- `McpToolCallError`
- `McpAuthError`
- `McpSessionExpiredError`
- protocol-level `McpError`

并针对不同类型执行：

- 改状态为 `needs-auth`
- 清连接缓存后重试
- telemetry safe 包装
- 把 `_meta` 保留下来

这让上层看到的不再是混乱异常，而是可治理的故障语义。

---

## 16. 结果治理与上下文控制

## 16.1 MCP 结果必须被治理，否则无法稳定参与 Agent 推理

MCP server 可能返回：

- 大文本
- 大 JSON
- 图片
- 音频
- blob
- resource link

如果系统只是把它们原样塞回模型：

- token 暴涨
- context window 失控
- 二进制不可消费
- 图片不可显示

因此 Claude Code 加了一层结果治理。

## 16.2 二进制与多模态结果被转为“可消费代理”

`transformResultContent()` 的策略本质是：

- 文本直接回填
- 图片压缩缩放后转 image block
- 音频/blob 落盘，返回说明文本
- resource/resource_link 转成对模型可读的 text/image block

这是一种非常重要的思想：

**不是所有结果都应该直接传给模型，必要时应先生成一个模型可消费的代理表示。**

## 16.3 大结果持久化是上下文预算控制机制

`processMCPResult()` 在结果过大时会：

- 截断
- 或持久化到文件并返回“如何读取该文件”的说明

这相当于将：

```text
大上下文内联
```

变成：

```text
外部化结果引用 + 延迟读取
```

这是典型的上下文窗口 externalization 策略，对 Agent 系统非常重要。

## 16.4 `inferCompactSchema()` 的作用

它并不是强验证器，而是为结构化结果提供：

- 紧凑 schema 描述
- 更好的 persisted-result 可读性
- 更好的后续读取提示

这体现出 Claude Code 不是只追求“存下来”，而是追求“后续还能被模型高效理解”。

---

## 17. 与 Query/Tool Runtime 的耦合方式

## 17.1 MCP 被耦合在工具抽象层，而非 Query 内核

这是本架构最优秀的地方之一。

Query Engine 并不关心：

- 这个 tool 是否来自 MCP
- 它的 transport 是 SSE 还是 stdio
- 它如何做 OAuth

Query Engine 只关心：

- 当前有哪些 `Tool`
- 调用后能否得到 `tool_result`

这使得 MCP 的复杂性被局部化在 `services/mcp/*` 与 `toolExecution.ts`。

## 17.2 但错误状态会回写 AppState

虽然调用入口统一，MCP 的运行期故障仍然会回写 `AppState.mcp`。

例如：

- `McpAuthError` 会把 client 状态置为 `needs-auth`

这是一种非常合理的“单向抽象 + 双向状态反馈”设计：

- Query 层不感知 MCP 内部复杂性
- UI / runtime 状态仍能准确反映 MCP 生命周期

---

## 18. 关键设计约束

这套架构有一些隐含但非常重要的约束。

## 18.1 同名不可信，必须靠规范化与签名

server name 既用于显示也用于路由，但系统仍然要：

- 规范化 name
- 对连接对象做内容签名

这说明“名字只是逻辑键，不是稳定身份”。

## 18.2 不能假设外部 server 符合规范

代码大量体现了这一点：

- OAuth 200-body-error 兼容
- invalid_grant alias 兼容
- revocation fallback
- path-aware discovery fallback
- headersHelper 失败 soft-fail

这是典型的工业原则：**对外部系统保守假设。**

## 18.3 不能假设配置来源可信

体现为：

- project `.mcp.json` 审批
- workspace trust 检查 headersHelper
- enterprise policy gate
- plugin-only/managed-only 限制

## 18.4 不能假设能力目录静态

体现为：

- `list_changed` 动态刷新
- stale plugin client 清理
- resource/skill cache 失效

## 18.5 不能假设连接稳定

体现为：

- connection cache
- session expired heuristics
- auth negative cache
- auto reconnect backoff

这些约束本质上定义了整个设计空间。

---

## 19. 关键创新点与工业级亮点

下面这些点，是这套 MCP 架构里最值得 AI 平台和 Agent 框架借鉴的部分。

## 19.1 内容签名去重而不是名称去重

这是多来源能力接入系统中的核心技术点。

它解决了：

- 配置源异构
- 名称空间冲突
- 重复能力污染上下文

很多系统做到这里会直接失控，而 Claude Code 用内容签名稳住了。

## 19.2 两阶段加载 + 分组并发调度

把：

- 本地可快速获得的能力
- 远程需要网络的能力

拆成两个阶段，同时把 local/remote transport 分组并发。

这是非常成熟的启动性能工程。

## 19.3 把 MCP 结果外部化成文件引用

这不是简单的“超长就截断”，而是：

- 按内容类型分别处理
- 允许通过文件持久化实现外部化上下文

这对高复杂度 Agent 系统非常关键。

## 19.4 把工具调用扩展为可交互微工作流

URL elicitation 的处理是一个很有代表性的创新：

- tool call 不再被视为同步黑盒
- 而是一个可插入 hook、可等待用户、可重试的状态机

这为未来更复杂的人机协作工具调用奠定了模式。

## 19.5 transport abstraction 足够强，支持进程内 / SDK / 网络 / 子进程统一接入

`InProcessTransport` 与 `SdkControlTransport` 说明：

- 只要 transport 合约统一
- 上层能力发现与工具适配逻辑就可以完全复用

这是非常高质量的架构抽象。

## 19.6 认证负缓存与 discovery-state reuse

这是非常典型的“把线上事故经验编码进控制平面”的设计。

收益非常大：

- 启动更快
- 失败更少
- 用户体验更稳定
- 非交互模式更可控

---

## 20. 对 AI 应用与算法平台的启示

如果把这套设计抽象成通用方法论，对 AI 应用平台至少有 5 个启示。

## 20.1 外部工具不应直接暴露给模型

应该先适配成平台内部统一抽象：

- 带权限
- 带审计
- 带稳定命名
- 带结果治理

## 20.2 工具系统需要控制平面与数据平面分离

- 控制平面：配置、策略、连接、认证、状态
- 数据平面：真正的 tool call 与结果回流

Claude Code 的 MCP 子系统就是一个很好的例子。

## 20.3 工具调用需要显式建模失败语义

401、session expired、step-up、user interaction required，不应都退化成一个 `Error`。

## 20.4 上下文窗口必须被视为稀缺资源

因此：

- 大结果要外部化
- 二进制要代理化
- 结构化结果要摘要化

## 20.5 混合信任域系统必须把配置输入当作攻击面

项目仓库、插件、远程 connector、组织托管配置，不是同一信任级别。

Claude Code 的 MCP 架构已经体现了这一点。

---

## 21. 最终结论

从深层设计上看，Claude Code 的 MCP 架构不是“对 MCP 协议的支持”，而是一个成熟的、面向 Agent Runtime 的外部能力接入平台。

它最核心的价值在于：

1. 把异构外部能力标准化为统一内部抽象。
2. 把外部依赖当作不稳定、半可信、长生命周期资源来治理。
3. 在连接、认证、动态发现、结果治理、权限控制之间建立了清晰而强类型的边界。
4. 通过内容签名去重、两阶段加载、负缓存、状态批处理、transport bridge 等设计，把复杂度控制在可维护范围内。

如果要用一句话概括这套架构：

**Claude Code 把 MCP 从“协议接入”提升成了“Agent 外部能力控制平面”。**

这也是它相较于许多仅做 SDK 封装的 AI 工具系统，更接近工业级 Agent OS 的关键原因。
