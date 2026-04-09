# Claude-code-open Remote 相关服务端接口梳理

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 中 remote/bridge/direct-connect 相关的服务端接口定义。

目标不是泛泛描述“会调用哪些 API”，而是把下面这些内容系统梳理清楚：

1. 接口按哪几类服务面划分。
2. 每个接口的路径、HTTP 方法、认证方式、关键请求头。
3. 请求体与响应体的关键结构。
4. 这些接口分别由哪个模块调用。
5. 它们在业务流程中的时序位置。
6. 哪些接口属于 Claude Sessions API，哪些属于 Bridge Environments API，哪些属于 CCR v2 Code Session API，哪些属于自建 Direct Connect server。

本文主要基于以下代码：

- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`
- `src/utils/teleport/api.ts`
- `src/utils/teleport.tsx`
- `src/utils/teleport/environments.ts`
- `src/bridge/bridgeApi.ts`
- `src/bridge/createSession.ts`
- `src/bridge/codeSessionApi.ts`
- `src/bridge/workSecret.ts`
- `src/bridge/replBridgeTransport.ts`
- `src/bridge/trustedDevice.ts`
- `src/bridge/inboundAttachments.ts`
- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`
- `src/server/types.ts`

---

## 2. 总览

从代码看，remote 相关服务端接口可以分成 6 组：

```text
A. Sessions API
面向远端 Claude 会话本身

B. Environment Providers API
面向可选执行环境列表与默认云环境

C. Bridge Environments API
面向 Remote Control / bridge worker 调度

D. Code Session API（CCR v2 / env-less bridge）
面向直接桥接的 code session + worker jwt

E. Session Ingress / 文件内容接口
面向会话日志、事件流、附件内容

F. Direct Connect Server API
面向自建 Claude Code server
```

这些接口并不属于同一抽象层：

- `Sessions API` 关注“会话资源”
- `Bridge Environments API` 关注“worker 环境与 work 调度”
- `Code Session API` 关注“直连 worker 凭据发放”
- `Session Ingress` 关注“日志、事件、附件”
- `Direct Connect` 则完全是另一套自托管协议

---

## 3. 认证与公共请求头

## 3.1 OAuth 认证头

大多数 Anthropic 远程接口最终都使用 `getOAuthHeaders(accessToken)`：

```ts
{
  Authorization: `Bearer ${accessToken}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01'
}
```

调用点：

- `src/utils/teleport/api.ts`
- `src/bridge/createSession.ts`
- `src/utils/teleport/environments.ts`
- `src/bridge/codeSessionApi.ts`

---

## 3.2 Sessions/CCR BYOC Beta 头

很多 `Sessions API` 请求还会附带：

```ts
{
  'anthropic-beta': 'ccr-byoc-2025-07-29',
  'x-organization-uuid': orgUUID
}
```

其中：

- `anthropic-beta` 用来进入 Sessions/CCR BYOC 能力面
- `x-organization-uuid` 用于组织级资源作用域

典型调用：

- `GET /v1/sessions`
- `POST /v1/sessions`
- `GET /v1/sessions/{id}`
- `PATCH /v1/sessions/{id}`
- `POST /v1/sessions/{id}/events`
- `POST /v1/sessions/{id}/archive`

---

## 3.3 Bridge Environments Beta 头

`src/bridge/bridgeApi.ts` 里，Bridge 环境层统一带：

```ts
{
  Authorization: `Bearer ${token}`,
  'Content-Type': 'application/json',
  'anthropic-version': '2023-06-01',
  'anthropic-beta': 'environments-2025-11-01',
  'x-environment-runner-version': MACRO.VERSION,
  ...(X-Trusted-Device-Token)
}
```

这里的 token 既可能是：

- OAuth access token
- environment secret
- session ingress token / worker jwt

取决于具体接口。

---

## 3.4 Trusted Device 头

某些 bridge 请求还会在 gate 开启时附带：

```ts
{
  'X-Trusted-Device-Token': trustedDeviceToken
}
```

来源：

- `src/bridge/trustedDevice.ts`

用途：

- 提升 bridge worker 的安全级别
- 配合服务端 elevated auth enforcement

---

## 4. Sessions API 接口族

Sessions API 是 remote session 的主资源面。

## 4.1 `GET /v1/sessions`

### 调用方

- `fetchCodeSessionsFromSessionsAPI()` in `src/utils/teleport/api.ts`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 响应结构

代码中的结构定义：

```ts
type ListSessionsResponse = {
  data: SessionResource[]
  has_more: boolean
  first_id: string | null
  last_id: string | null
}
```

`SessionResource`：

```ts
type SessionResource = {
  type: 'session'
  id: string
  title: string | null
  session_status: 'requires_action' | 'running' | 'idle' | 'archived'
  environment_id: string
  created_at: string
  updated_at: string
  session_context: SessionContext
}
```

### 用途

- 拉取当前用户可见的远程 code sessions 列表
- 为 UI 显示做轻量转换

---

## 4.2 `GET /v1/sessions/{sessionId}`

### 调用方

- `fetchSession()` in `src/utils/teleport/api.ts`
- `getBridgeSession()` in `src/bridge/createSession.ts`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 主要响应字段

`fetchSession()` 按 `SessionResource` 解析。

`getBridgeSession()` 只关心：

```ts
{ environment_id?: string; title?: string }
```

### 用途

- 获取单个 session 的元数据
- 补全 branch/status
- bridge resume 场景下查询 `environment_id`

---

## 4.3 `POST /v1/sessions`

这是最核心的会话创建接口，被多个场景复用。

### 调用方

- `teleportToRemote()` in `src/utils/teleport.tsx`
- `createBridgeSession()` in `src/bridge/createSession.ts`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 请求体 1：普通 remote session 创建

`teleportToRemote()` 发送的典型结构：

```ts
{
  title: string,
  events: Array<{ type: 'event', data: Record<string, unknown> }>,
  session_context: {
    sources: SessionContextSource[],
    outcomes: Outcome[],
    model?: string,
    seed_bundle_file_id?: string,
    reuse_outcome_branches?: boolean,
    github_pr?: { owner: string; repo: string; number: number },
    environment_variables?: Record<string, string>
  },
  environment_id: string
}
```

其中 `events` 里可能预置：

- `control_request{subtype:'set_permission_mode'}`
- 首条 user message

### 请求体 2：bridge session 创建

`createBridgeSession()` 发送的结构：

```ts
{
  title?: string,
  events: Array<{ type: 'event', data: SDKMessage }>,
  session_context: {
    sources: GitSource[],
    outcomes: GitOutcome[],
    model: string
  },
  environment_id: string,
  source: 'remote-control',
  permission_mode?: string
}
```

### 响应

调用方主要依赖：

- `response.data.id`

### 用途

- 创建远程代码会话
- 绑定环境执行上下文
- 在 bridge 模式中，为远端页面建立一个可见 session 实体

---

## 4.4 `GET /v1/sessions/{sessionId}/events`

### 调用方

- `pollRemoteSessionEvents()` in `src/utils/teleport.tsx`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 查询参数

可选：

```ts
{ after_id: string }
```

### 响应结构

代码中的本地约定：

```ts
type EventsResponse = {
  data: unknown[]
  has_more: boolean
  first_id: string | null
  last_id: string | null
}
```

### 客户端过滤规则

代码会忽略：

- `env_manager_log`
- `control_response`

并将带 `session_id` 的事件视为 `SDKMessage`。

### 用途

- 增量轮询 session 事件
- 在非 WebSocket 场景下同步远程 session 的增量更新

---

## 4.5 `POST /v1/sessions/{sessionId}/events`

### 调用方

- `sendEventToRemoteSession()` in `src/utils/teleport/api.ts`
- `bridgeApi.sendPermissionResponseEvent()` in `src/bridge/bridgeApi.ts`

### 认证

两种模式：

#### A. 普通 remote session

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

#### B. bridge permission response

- session token / ingress token
- `anthropic-beta: environments-2025-11-01`

### 请求体 1：发送用户消息

`sendEventToRemoteSession()` 构造：

```ts
{
  events: [
    {
      uuid: string,
      session_id: string,
      type: 'user',
      parent_tool_use_id: null,
      message: {
        role: 'user',
        content: string | ContentBlock[]
      }
    }
  ]
}
```

### 请求体 2：发送 permission control response

`sendPermissionResponseEvent()` 发送：

```ts
{
  events: [
    {
      type: 'control_response',
      response: {
        subtype: 'success',
        request_id: string,
        response: Record<string, unknown>
      }
    }
  ]
}
```

### 用途

- 向已存在 session 追加事件
- remote 模式下追加用户输入
- bridge 模式下回传权限裁决

---

## 4.6 `PATCH /v1/sessions/{sessionId}`

### 调用方

- `updateSessionTitle()` in `src/utils/teleport/api.ts`
- `updateBridgeSessionTitle()` in `src/bridge/createSession.ts`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 请求体

```ts
{ title: string }
```

### 特别说明

`updateBridgeSessionTitle()` 会先把可能的 `cse_*` ID 转成兼容层 `session_*`：

```ts
const compatId = toCompatSessionId(sessionId)
```

说明 compat gateway 只接受 `session_*` 风格 ID。

### 用途

- 同步远程 session 标题
- /rename 后保持 bridge/web 侧标题一致

---

## 4.7 `POST /v1/sessions/{sessionId}/archive`

### 调用方

- `archiveRemoteSession()` in `src/utils/teleport.tsx`
- `archiveBridgeSession()` in `src/bridge/createSession.ts`
- `bridgeApi.archiveSession()` in `src/bridge/bridgeApi.ts`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 请求体

空对象：

```ts
{}
```

### 响应处理

- `bridgeApi.archiveSession()` 将 `409` 视为已归档，属于成功
- `archiveRemoteSession()` 将 `200` / `409` 视为成功

### 用途

- 显式归档 session
- 清理 remote/bridge 会话资源

---

## 5. Environment Providers API 接口族

这组接口用于 remote session 的环境选择，不属于 bridge worker 调度层。

## 5.1 `GET /v1/environment_providers`

### 调用方

- `fetchEnvironments()` in `src/utils/teleport/environments.ts`

### 认证

- OAuth access token
- `x-organization-uuid`

### 响应结构

```ts
type EnvironmentResource = {
  kind: 'anthropic_cloud' | 'byoc' | 'bridge'
  environment_id: string
  name: string
  created_at: string
  state: 'active'
}

type EnvironmentListResponse = {
  environments: EnvironmentResource[]
  has_more: boolean
  first_id: string | null
  last_id: string | null
}
```

### 用途

- 给 `teleportToRemote()` 选择目标 environment
- 支持默认 environment 配置

---

## 5.2 `POST /v1/environment_providers/cloud/create`

### 调用方

- `createDefaultCloudEnvironment()` in `src/utils/teleport/environments.ts`

### 认证

- OAuth access token
- `x-organization-uuid`
- `anthropic-beta: ccr-byoc-2025-07-29`

### 请求体

代码中发送的固定结构：

```ts
{
  name: string,
  kind: 'anthropic_cloud',
  description: '',
  config: {
    environment_type: 'anthropic',
    cwd: '/home/user',
    init_script: null,
    environment: {},
    languages: [
      { name: 'python', version: '3.11' },
      { name: 'node', version: '20' }
    ],
    network_config: {
      allowed_hosts: [],
      allow_default_hosts: true
    }
  }
}
```

### 响应

- `EnvironmentResource`

### 用途

- 用户没有默认云环境时创建一个 `anthropic_cloud` environment

---

## 6. Bridge Environments API 接口族

这组接口只属于 env-based bridge / Remote Control worker 调度层。

## 6.1 `POST /v1/environments/bridge`

### 调用方

- `bridgeApi.registerBridgeEnvironment()`

### 认证

- OAuth access token
- `anthropic-beta: environments-2025-11-01`
- `x-environment-runner-version`
- 可选 `X-Trusted-Device-Token`

### 请求体

```ts
{
  machine_name: string,
  directory: string,
  branch: string,
  git_repo_url: string | null,
  max_sessions: number,
  metadata: { worker_type: string },
  environment_id?: string   // reuseEnvironmentId
}
```

### 响应

```ts
{
  environment_id: string,
  environment_secret: string
}
```

### 用途

- 注册本地 bridge worker environment
- 或在 resume 时重连已有 environment

---

## 6.2 `GET /v1/environments/{environmentId}/work/poll`

### 调用方

- `bridgeApi.pollForWork()`
- 被 `startWorkPollLoop()` 循环调用

### 认证

- `environment_secret`
- `anthropic-beta: environments-2025-11-01`

### 查询参数

可选：

```ts
{ reclaim_older_than_ms: number }
```

### 响应

```ts
WorkResponse | null
```

其中：

```ts
type WorkResponse = {
  id: string
  type: 'work'
  environment_id: string
  state: string
  data: {
    type: 'session' | 'healthcheck'
    id: string
  }
  secret: string
  created_at: string
}
```

### 用途

- 桥接 worker 领取 work item
- session 型 work 到达后触发 transport 建立

---

## 6.3 `POST /v1/environments/{environmentId}/work/{workId}/ack`

### 调用方

- `bridgeApi.acknowledgeWork()`

### 认证

- `session_ingress_token`
- `anthropic-beta: environments-2025-11-01`

### 请求体

空对象：

```ts
{}
```

### 用途

- 显式确认该 work 已被当前 worker 接收
- 防止服务端重复投递

---

## 6.4 `POST /v1/environments/{environmentId}/work/{workId}/stop`

### 调用方

- `bridgeApi.stopWork()`

### 认证

- OAuth access token
- `anthropic-beta: environments-2025-11-01`

### 请求体

```ts
{ force: boolean }
```

### 用途

- 停止当前 work item
- `force=false` 多用于回收并等待重派
- `force=true` 多用于 teardown

---

## 6.5 `POST /v1/environments/{environmentId}/work/{workId}/heartbeat`

### 调用方

- `bridgeApi.heartbeatWork()`
- 被 `startWorkPollLoop()` 的 heartbeat 模式使用

### 认证

- `session_ingress_token`
- `anthropic-beta: environments-2025-11-01`

### 响应结构

```ts
{
  lease_extended: boolean
  state: string
  last_heartbeat: string
  ttl_seconds: number
}
```

客户端实际使用：

```ts
{ lease_extended: boolean; state: string }
```

### 用途

- 延长当前 work lease
- 表明 worker 仍然活着

---

## 6.6 `POST /v1/environments/{environmentId}/bridge/reconnect`

### 调用方

- `bridgeApi.reconnectSession()`

### 认证

- OAuth access token
- `anthropic-beta: environments-2025-11-01`

### 请求体

```ts
{ session_id: string }
```

### 用途

- env-based bridge 在重建 environment 后，尝试把旧 session 重新派回该 environment

---

## 6.7 `DELETE /v1/environments/bridge/{environmentId}`

### 调用方

- `bridgeApi.deregisterEnvironment()`

### 认证

- OAuth access token
- `anthropic-beta: environments-2025-11-01`

### 用途

- clean shutdown 时注销 bridge environment

---

## 7. Code Session API（CCR v2 / env-less bridge）

这组接口属于 env-less bridge，不再经过 environment/work 调度层。

## 7.1 `POST /v1/code/sessions`

### 调用方

- `createCodeSession()` in `src/bridge/codeSessionApi.ts`

### 认证

- OAuth access token
- `anthropic-version: 2023-06-01`

### 请求体

```ts
{
  title: string,
  bridge: {},
  tags?: string[]
}
```

其中 `bridge: {}` 是显式 runner selector，代码注释明确说明：

- 省略 `bridge` 或传 `environment_id: ""` 现在会 400

### 响应期望

代码要求：

```ts
response.data.session.id: string
response.data.session.id.startsWith('cse_')
```

### 用途

- 创建一个 CCR v2 code session
- 为 env-less bridge 提供会话实体

---

## 7.2 `POST /v1/code/sessions/{sessionId}/bridge`

### 调用方

- `fetchRemoteCredentials()` in `src/bridge/codeSessionApi.ts`
- `remoteBridgeCore.ts` 初始化和 token refresh 都会调用

### 认证

- OAuth access token
- `anthropic-version: 2023-06-01`
- 可选 `X-Trusted-Device-Token`

### 请求体

空对象：

```ts
{}
```

### 响应结构

```ts
type RemoteCredentials = {
  worker_jwt: string
  api_base_url: string
  expires_in: number
  worker_epoch: number
}
```

### 语义说明

代码注释明确指出：

- 每次 `/bridge` 调用都会 bump `worker_epoch`
- 这个调用本身就相当于 register worker

### 用途

- 用 OAuth 换取 worker JWT
- 获取 CCR v2 写入和 SSE 读取所需凭据

---

## 7.3 `POST /v1/code/sessions/{sessionId}/worker/register`

### 调用方

- `registerWorker()` in `src/bridge/workSecret.ts`
- 被 `createV2ReplTransport()` 在未提供 `epoch` 时调用

### 认证

- `Authorization: Bearer ${accessToken}`
  这里的 accessToken 实际上是 worker / ingress token
- `anthropic-version: 2023-06-01`

### 请求体

空对象：

```ts
{}
```

### 响应

```ts
{ worker_epoch: number | string }
```

客户端会兼容 protojson 的 string int64 编码。

### 用途

- 显式注册 v2 worker
- 获取 `worker_epoch`

---

## 7.4 `GET /v1/code/sessions/{sessionId}/worker/events/stream`

### 调用方

- `createV2ReplTransport()` 内部构造 SSE URL
- 由 `SSETransport` 消费

### 认证

- worker jwt

### URL 推导

`createV2ReplTransport()` 先拿到 `sessionUrl`：

```ts
${baseUrl}/v1/code/sessions/${sessionId}
```

然后派生：

```ts
${sessionUrl}/worker/events/stream
```

### 用途

- v2 bridge 的读通道
- 承载 session 事件流和控制消息

---

## 7.5 `POST/PUT /worker/*` 系列

代码没有在本文所读文件中直接展开所有 `CCRClient` 的 HTTP 路径实现，但从 `ReplBridgeTransport` 的接口注释可知，至少包括：

- 写事件：`POST /worker/events`
- 上报状态：`PUT /worker state`
- 上报 metadata：`PUT /worker external_metadata`
- 事件投递状态：`POST /worker/events/{id}/delivery`

这些接口由：

- `CCRClient`
- `SSETransport`

在 transport 层内部消费。

### 当前可确定的语义

- `reportState(state)` 用于让后端知道是否 `requires_action`
- `reportMetadata(metadata)` 用于更新 worker 外部元数据
- `reportDelivery(eventId, status)` 用于打点 `processing/processed`

这里的精确请求体未在本文引用代码里完全展开，因此这些字段是基于 transport 注释和接口语义的归纳，不是完整 schema 抄录。

---

## 8. Session Ingress 与日志/附件接口

## 8.1 Session ingress WebSocket

### URL 构造

`buildSdkUrl(apiBaseUrl, sessionId)` in `src/bridge/workSecret.ts`

生成：

```ts
ws(s)://{host}/{v1|v2}/session_ingress/ws/{sessionId}
```

规则：

- localhost 使用 `ws` + `v2`
- 生产环境使用 `wss` + `v1`

### 调用方

- env-based bridge 的 v1 transport

### 用途

- bridge v1 的读通道

---

## 8.2 `GET /v1/sessions/ws/{sessionId}/subscribe?organization_uuid=...`

### 调用方

- `SessionsWebSocket.connect()` in `src/remote/SessionsWebSocket.ts`

### 认证

- WebSocket headers 带 `Authorization: Bearer <oauth token>`
- `anthropic-version: 2023-06-01`

### 用途

- Remote Session 模式下订阅远端 session 的事件流

---

## 8.3 Session logs / Teleport events

`teleportFromSessionsAPI()` 会优先尝试：

- `getTeleportEvents(sessionId, accessToken, orgUUID)`

失败后回退：

- `getSessionLogsViaOAuth(sessionId, accessToken, orgUUID)`

这两个函数定义在：

- `src/services/api/sessionIngress.js`

从当前引用代码能明确得出：

- 它们属于 session ingress / teleport event 读取路径
- 用于拉取会话日志与 transcript hydration

但其精确 URL 在本文展示代码中未完全展开，因此这里只记录其角色，不展开未直接读取到的精确 schema。

---

## 8.4 `PUT` session log append（session ingress）

`src/services/api/sessionIngress.ts` 展示了日志持久化接口族的客户端约束：

- 使用 JWT `session_ingress_token`
- `appendSessionLog()` 最终会 `axios.put(url, entry, ...)`
- 用 `Last-Uuid` header 做 optimistic concurrency

### 关键请求头

```ts
{
  Authorization: `Bearer ${sessionToken}`,
  'Content-Type': 'application/json',
  'Last-Uuid'?: UUID
}
```

### 关键客户端语义

- 409 冲突时会读取 `x-last-uuid`
- 或重新 fetch logs 以 adopt server head

### 用途

- 远端 session transcript 追加写入
- 保证多写者场景下顺序一致性

由于 `url` 是由调用方传入，本文不对其固定路径做超出代码证据的推断。

---

## 8.5 `GET /api/oauth/files/{file_uuid}/content`

### 调用方

- `resolveOne()` in `src/bridge/inboundAttachments.ts`

### 认证

- OAuth access token

### 路径

```ts
${getBridgeBaseUrl()}/api/oauth/files/${encodeURIComponent(file_uuid)}/content
```

### 响应

- 二进制文件内容

### 用途

- 下载远端消息里带的 `file_uuid` 附件
- 保存到本地 `~/.claude/uploads/{sessionId}/`
- 再转成 `@"path"` 注入给本地 REPL

---

## 9. Trusted Device 接口

## 9.1 `POST /api/auth/trusted_devices`

### 调用方

- `enrollTrustedDevice()` in `src/bridge/trustedDevice.ts`

### 认证

- OAuth access token

### 请求体

```ts
{
  display_name: `Claude Code on ${hostname()} · ${process.platform}`
}
```

### 响应结构

代码期望：

```ts
{
  device_token?: string
  device_id?: string
}
```

### 用途

- 在登录后为当前设备注册 trusted device token
- 后续 bridge API 请求可附带 `X-Trusted-Device-Token`

---

## 10. Direct Connect Server API

Direct Connect 使用完全不同的一套服务端接口，不属于 Anthropic Sessions/Bridge API。

## 10.1 `POST {serverUrl}/sessions`

### 调用方

- `createDirectConnectSession()` in `src/server/createDirectConnectSession.ts`

### 认证

- 可选 `Authorization: Bearer ${authToken}`

### 请求体

```ts
{
  cwd: string,
  dangerously_skip_permissions?: boolean
}
```

### 响应结构

由 `connectResponseSchema()` 验证：

```ts
{
  session_id: string,
  ws_url: string,
  work_dir?: string
}
```

### 用途

- 在自建 server 上创建一个 Claude Code 会话
- 获取后续 WebSocket 地址

---

## 10.2 `WebSocket ws_url`

### 调用方

- `DirectConnectSessionManager.connect()`

### 协议

- 文本帧
- 以换行分隔的 JSON 消息

### 入站消息

- `control_request`
- `control_response`
- 各类 `SDKMessage`

### 出站消息 1：用户消息

`sendMessage()` 发送：

```ts
{
  type: 'user',
  message: {
    role: 'user',
    content: RemoteMessageContent
  },
  parent_tool_use_id: null,
  session_id: ''
}
```

### 出站消息 2：权限响应

```ts
{
  type: 'control_response',
  response: {
    subtype: 'success',
    request_id: string,
    response: {
      behavior: 'allow' | 'deny',
      updatedInput?: Record<string, unknown>,
      message?: string
    }
  }
}
```

### 出站消息 3：interrupt

```ts
{
  type: 'control_request',
  request_id: crypto.randomUUID(),
  request: { subtype: 'interrupt' }
}
```

### 用途

- 自建 server 模式下的双向实时会话协议

---

## 11. 接口与模块映射表

| 接口 | 调用模块 | 场景 |
|---|---|---|
| `GET /v1/sessions` | `utils/teleport/api.ts` | 列出远程 sessions |
| `GET /v1/sessions/{id}` | `utils/teleport/api.ts`, `bridge/createSession.ts` | 获取 session 元数据 |
| `POST /v1/sessions` | `utils/teleport.tsx`, `bridge/createSession.ts` | 创建 remote/bridge session |
| `GET /v1/sessions/{id}/events` | `utils/teleport.tsx` | 轮询远程 session 事件 |
| `POST /v1/sessions/{id}/events` | `utils/teleport/api.ts`, `bridge/bridgeApi.ts` | 发送用户消息 / 权限响应 |
| `PATCH /v1/sessions/{id}` | `utils/teleport/api.ts`, `bridge/createSession.ts` | 更新标题 |
| `POST /v1/sessions/{id}/archive` | `utils/teleport.tsx`, `bridge/createSession.ts`, `bridge/bridgeApi.ts` | 归档 session |
| `GET /v1/environment_providers` | `utils/teleport/environments.ts` | 列环境 |
| `POST /v1/environment_providers/cloud/create` | `utils/teleport/environments.ts` | 创建默认云环境 |
| `POST /v1/environments/bridge` | `bridge/bridgeApi.ts` | 注册 bridge environment |
| `GET /v1/environments/{id}/work/poll` | `bridge/bridgeApi.ts` | 轮询 work |
| `POST /v1/environments/{id}/work/{id}/ack` | `bridge/bridgeApi.ts` | ack work |
| `POST /v1/environments/{id}/work/{id}/stop` | `bridge/bridgeApi.ts` | 停止 work |
| `POST /v1/environments/{id}/work/{id}/heartbeat` | `bridge/bridgeApi.ts` | 延长 lease |
| `POST /v1/environments/{id}/bridge/reconnect` | `bridge/bridgeApi.ts` | reconnect session |
| `DELETE /v1/environments/bridge/{id}` | `bridge/bridgeApi.ts` | 注销 environment |
| `POST /v1/code/sessions` | `bridge/codeSessionApi.ts` | env-less 创建 code session |
| `POST /v1/code/sessions/{id}/bridge` | `bridge/codeSessionApi.ts` | 取 worker jwt |
| `POST /v1/code/sessions/{id}/worker/register` | `bridge/workSecret.ts` | 注册 v2 worker |
| `GET /v1/code/sessions/{id}/worker/events/stream` | `bridge/replBridgeTransport.ts` | v2 SSE 读流 |
| `GET /api/oauth/files/{uuid}/content` | `bridge/inboundAttachments.ts` | 下载附件 |
| `POST /api/auth/trusted_devices` | `bridge/trustedDevice.ts` | 注册 trusted device |
| `POST {serverUrl}/sessions` | `server/createDirectConnectSession.ts` | 自建 server 创建 session |
| `WebSocket ws_url` | `server/directConnectManager.ts` | Direct Connect 双向流 |

---

## 12. 关键接口时序

## 12.1 Remote Session

```text
创建或附着 session
  -> GET /v1/sessions/{id}（可选）
  -> WS /v1/sessions/ws/{id}/subscribe
  -> POST /v1/sessions/{id}/events
  -> PATCH /v1/sessions/{id}（标题更新，可选）
  -> POST /v1/sessions/{id}/archive（结束时，可选）
```

---

## 12.2 Env-based Bridge

```text
POST /v1/environments/bridge
  -> POST /v1/sessions
  -> GET /v1/environments/{id}/work/poll
  -> POST /v1/environments/{id}/work/{id}/ack
  -> session ingress / worker transport 建立
  -> POST /.../heartbeat
  -> POST /v1/sessions/{id}/events（权限回传等）
  -> POST /v1/sessions/{id}/archive
  -> DELETE /v1/environments/bridge/{id}
```

---

## 12.3 Env-less Bridge

```text
POST /v1/code/sessions
  -> POST /v1/code/sessions/{id}/bridge
  -> GET /v1/code/sessions/{id}/worker/events/stream
  -> POST/PUT /worker/*
  -> token refresh 时再次 POST /bridge
  -> 结束时 POST /v1/sessions/{compatId}/archive
```

---

## 12.4 Direct Connect

```text
POST {serverUrl}/sessions
  -> connect ws_url
  -> 持续双向发送 SDK/control 消息
```

---

## 13. 结论

从接口定义层面看，Claude Code 的 remote 访问并不是单一 API，而是三套后端面：

1. `Sessions API`
   面向远程会话资源本身

2. `Bridge Environments API`
   面向本地 worker 环境注册、调度、work lease 和恢复

3. `Code Session API`
   面向 env-less bridge 的直接 worker 凭据交换

再加上：

4. `Session Ingress`
   负责日志、事件、附件内容

5. `Direct Connect Server API`
   负责自建 server 的点对点协议

因此，如果要正确理解 remote 架构，必须先分清：

- 哪些接口在操作“session 资源”
- 哪些接口在操作“worker/environment 调度”
- 哪些接口在操作“transport / ingress 通道”

只有把这三层拆开，remote/bridge/direct-connect 的调用链才会真正清晰。
