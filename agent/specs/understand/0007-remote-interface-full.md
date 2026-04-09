# Claude-code-open Remote 相关服务端接口梳理（扩展版）

## 1. 文档目标

本文是在 [0007-remote-interface.md](./0007-remote-interface.md) 基础上的扩展版。

上一版只覆盖了 remote/bridge/direct-connect 主链路直接使用的服务端接口；这一版进一步把“虽不属于主控制面，但会显著影响 remote 能否工作”的外围接口也纳入进来。

因此本文覆盖 4 层接口：

1. **主链路接口**
   Sessions API、Bridge Environments API、Code Session API、Direct Connect

2. **环境与资源准备接口**
   Environment Providers、GitHub 访问预检、bundle seed、Files API

3. **认证与会话前置接口**
   OAuth token exchange、refresh、profile、roles、API key、trusted device

4. **历史/传输辅助接口**
   Session Ingress transcript、teleport-events、附件内容下载

本文主要基于以下代码：

- `src/utils/teleport/api.ts`
- `src/utils/teleport.tsx`
- `src/utils/teleport/environments.ts`
- `src/utils/background/remote/preconditions.ts`
- `src/services/api/sessionIngress.ts`
- `src/services/api/filesApi.ts`
- `src/services/oauth/client.ts`
- `src/services/oauth/getOauthProfile.ts`
- `src/bridge/bridgeApi.ts`
- `src/bridge/createSession.ts`
- `src/bridge/codeSessionApi.ts`
- `src/bridge/trustedDevice.ts`
- `src/bridge/inboundAttachments.ts`
- `src/remote/SessionsWebSocket.ts`
- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`

---

## 2. 接口分层全景

从 remote 相关代码看，服务端接口可分成 9 组：

```text
A. OAuth / 账户接口
B. Sessions API
C. Environment Providers API
D. Bridge Environments API
E. CCR v2 Code Session API
F. Session Ingress / Teleport Events
G. Files API / Bundle Seed
H. GitHub 远程访问预检接口
I. Direct Connect Server API
```

其中：

- `B/C/D/E/I` 是业务主链路
- `A/F/G/H` 是强相关支撑链路

如果从依赖方向看，大致是：

```text
OAuth
  -> Sessions / Environments / Code Sessions / GitHub precheck / Files / Trusted Device

Sessions / Code Sessions
  -> Session Ingress / Teleport Events / Attachments

Bridge Environments
  -> Work poll / heartbeat / reconnect / permission event send

Files API
  -> 供 teleport bundle 和附件下载使用
```

---

## 3. 主链路接口回顾

这部分是对上一版的压缩回顾，完整定义见 [0007-remote-interface.md](./0007-remote-interface.md)。

### A. Remote Session 主链路

- `POST /v1/sessions`
- `GET /v1/sessions/{id}`
- `GET /v1/sessions/{id}/events`
- `POST /v1/sessions/{id}/events`
- `PATCH /v1/sessions/{id}`
- `POST /v1/sessions/{id}/archive`
- `GET /v1/sessions/ws/{id}/subscribe?organization_uuid=...`

### B. Env-based Bridge 主链路

- `POST /v1/environments/bridge`
- `GET /v1/environments/{id}/work/poll`
- `POST /v1/environments/{id}/work/{id}/ack`
- `POST /v1/environments/{id}/work/{id}/stop`
- `POST /v1/environments/{id}/work/{id}/heartbeat`
- `POST /v1/environments/{id}/bridge/reconnect`
- `DELETE /v1/environments/bridge/{id}`

### C. Env-less Bridge 主链路

- `POST /v1/code/sessions`
- `POST /v1/code/sessions/{id}/bridge`
- `POST /v1/code/sessions/{id}/worker/register`
- `GET /v1/code/sessions/{id}/worker/events/stream`

### D. Direct Connect 主链路

- `POST {serverUrl}/sessions`
- `WebSocket ws_url`

---

## 4. OAuth / 账户接口

这些接口不属于 remote 主控制面，但 remote/bridge 是否可用，强依赖它们。

## 4.1 Token Exchange：`POST {TOKEN_URL}`

### 调用方

- `exchangeCodeForTokens()` in `src/services/oauth/client.ts`

### 请求体

授权码交换：

```ts
{
  grant_type: 'authorization_code',
  code: authorizationCode,
  redirect_uri: string,
  client_id: CLIENT_ID,
  code_verifier: string,
  state: string,
  expires_in?: number
}
```

### 响应

代码按 `OAuthTokenExchangeResponse` 消费，关键字段包括：

- `access_token`
- `refresh_token`
- `expires_in`
- `scope`
- `account`
- `organization`

### 用途

- 用户首次登录后，获取 OAuth access/refresh token

---

## 4.2 Token Refresh：`POST {TOKEN_URL}`

### 调用方

- `refreshOAuthToken()` in `src/services/oauth/client.ts`
- 间接被 `checkAndRefreshOAuthTokenIfNeeded()` 触发

### 请求体

```ts
{
  grant_type: 'refresh_token',
  refresh_token: string,
  client_id: CLIENT_ID,
  scope: string
}
```

### 用途

- 在 remote / bridge 会话长期运行时续签 access token

### 与 remote 的关系

以下 remote 子系统都依赖 refresh 后的 token：

- `SessionsWebSocket`
- `sendEventToRemoteSession()`
- `bridgeApi.withOAuthRetry()`
- `fetchRemoteCredentials()`
- GitHub 远程预检
- Files API 上传/下载

---

## 4.3 Profile：`GET /api/oauth/profile`

### 调用方

- `getOauthProfileFromOauthToken()` in `src/services/oauth/getOauthProfile.ts`

### 认证

- `Authorization: Bearer ${accessToken}`

### 用途

- 获取 OAuth 账户 profile
- 填充 org/account 相关信息
- 支撑 remote/bridge entitlement 判断

### 备注

`bridgeEnabled.ts` 的注释明确指出：

- `organizationUuid` 依赖 profile 相关信息
- 没有 profile scope 会影响 Remote Control gate 判断

---

## 4.4 API Key Profile：`GET /api/claude_cli_profile`

### 调用方

- `getOauthProfileFromApiKey()` in `src/services/oauth/getOauthProfile.ts`

### 认证

- `x-api-key`
- `anthropic-beta: OAUTH_BETA_HEADER`

### 查询参数

```ts
{ account_uuid: string }
```

### 用途

- 用 API key 补查 CLI profile
- 不属于 remote 主链路，但与身份补全相关

---

## 4.5 Roles：`GET {ROLES_URL}`

### 调用方

- `fetchAndStoreUserRoles()` in `src/services/oauth/client.ts`

### 认证

- `Authorization: Bearer ${accessToken}`

### 用途

- 获取 organization/workspace 角色
- 为 remote/bridge 权限边界提供账户维度信息

---

## 4.6 API Key 创建：`POST {API_KEY_URL}`

### 调用方

- `createAndStoreApiKey()` in `src/services/oauth/client.ts`

### 认证

- `Authorization: Bearer ${accessToken}`

### 用途

- 创建 API key
- 不直接属于 remote 主链路，但与 auth bootstrap 相关

---

## 5. Sessions API 扩展定义

这一节补充上一版未详细展开的字段。

## 5.1 `SessionContext`

`src/utils/teleport/api.ts` 中本地定义：

```ts
type SessionContext = {
  sources: SessionContextSource[]
  cwd: string
  outcomes: Outcome[] | null
  custom_system_prompt: string | null
  append_system_prompt: string | null
  model: string | null
  seed_bundle_file_id?: string
  github_pr?: { owner: string; repo: string; number: number }
  reuse_outcome_branches?: boolean
}
```

这说明 `POST /v1/sessions` 在 remote 创建时，并不只是创建一个空会话，还可承载：

- Git source
- Git outcome branch
- bundle seed
- PR 关联
- 模型
- 系统提示

因此 `Sessions API` 在系统里兼具：

- 会话资源管理
- 远端执行环境初始化参数传递

---

## 5.2 `POST /v1/sessions` 的两类增强参数

### A. `seed_bundle_file_id`

用于让远端环境以 bundle 的形式启动本地工作树快照。

### B. 预置 `events`

常见包括：

- `control_request{subtype:'set_permission_mode'}`
- 第一条 `user` message

也就是说，会话创建时就可以嵌入初始控制状态和初始提示，而不是等 session ready 后再补发。

---

## 6. Environment Providers API 扩展

## 6.1 `GET /v1/environment_providers`

### 用途再解释

此接口返回的不是 bridge worker 环境，而是 remote session 可以选择落在哪种 provider 上：

- `anthropic_cloud`
- `byoc`
- `bridge`

所以它是“会话执行环境选择器”，不是“worker 调度接口”。

---

## 7. Bridge Environments API 扩展

## 7.1 `POST /v1/environments/bridge`

### 请求字段语义

```ts
{
  machine_name,
  directory,
  branch,
  git_repo_url,
  max_sessions,
  metadata: { worker_type },
  environment_id?   // reuse
}
```

这些字段分别用于：

- `machine_name`
  人类可识别的执行端标识

- `directory`
  当前工作目录

- `branch` / `git_repo_url`
  让服务端知道桥接环境背后的仓库语义

- `max_sessions`
  为多会话能力做 capacity 建模

- `worker_type`
  区分 REPL / assistant / cowork 等来源

- `environment_id`
  作为 idempotent re-registration / resume 的锚点

---

## 7.2 `GET /v1/environments/{id}/work/poll`

### `WorkSecret`

从 `src/bridge/types.ts` 可知，`secret` 解码后至少包含：

```ts
{
  version: number,
  session_ingress_token: string,
  api_base_url: string,
  sources: ...,
  auth: ...,
  claude_code_args?: ...,
  mcp_config?: ...,
  environment_variables?: ...,
  use_code_sessions?: boolean
}
```

因此 `work/poll` 的响应，不只是“给你一个 token”，而是给你一个完整的工作执行胶囊。

---

## 8. CCR v2 Code Session API 扩展

## 8.1 `POST /v1/code/sessions/{id}/bridge`

### 响应结构语义

```ts
{
  worker_jwt: string,
  api_base_url: string,
  expires_in: number,
  worker_epoch: number
}
```

每个字段的作用：

- `worker_jwt`
  v2 worker 的真正运行 token

- `api_base_url`
  transport 实际连接用的 API base

- `expires_in`
  用于客户端设置 proactive refresh scheduler

- `worker_epoch`
  用于 worker 注册代际控制，防止旧 worker 继续写

### 重要语义

代码注释明确指出：

- 每次 `/bridge` 都会 bump `worker_epoch`
- 因此 refresh path 不能重复并发调用，否则会把自己顶掉

这是 env-less bridge 设计里的一个重要约束。

---

## 8.2 `GET /v1/code/sessions/{id}/teleport-events`

### 调用方

- `getTeleportEvents()` in `src/services/api/sessionIngress.ts`

### 认证

- OAuth access token
- `x-organization-uuid`

### 查询参数

```ts
{
  limit: number,   // 默认客户端传 1000
  cursor?: string
}
```

### 响应结构

```ts
type TeleportEventsResponse = {
  data: Array<{
    event_id: string
    event_type: string
    is_compaction: boolean
    payload: Entry | null
    created_at: string
  }>
  next_cursor?: string
}
```

### 用途

- 以 CCR v2 Sessions API 方式拉取 transcript / teleport 历史
- 逐步取代旧的 session-ingress transcript 拉取

### 客户端语义

- `payload` 就是 `Entry`
- `payload === null` 直接跳过
- `next_cursor` 为空表示结束
- 404 第 0 页会回退到旧 session ingress 路径

---

## 9. Session Ingress 接口族

这一组接口属于“会话历史与顺序持久化”层。

## 9.1 `GET /v1/session_ingress/session/{sessionId}`

### 调用方

- `getSessionLogsViaOAuth()` in `src/services/api/sessionIngress.ts`

### 认证

- OAuth access token
- `x-organization-uuid`

### 用途

- 旧路径下按 OAuth 获取 transcript hydration 数据
- teleport 恢复时的 fallback

---

## 9.2 Session log append URL（由调用方提供）

`appendSessionLog(sessionId, entry, url)` 最终会对调用方提供的 `url` 发：

```ts
PUT {url}
Authorization: Bearer {session_ingress_token}
Content-Type: application/json
Last-Uuid?: UUID
Body: TranscriptMessage
```

### 客户端约束

- 每个 session 串行写
- 409 时使用 `x-last-uuid` 或重拉日志做修复
- 401 直接视为 token 失效

### 用途

- transcript 持久化
- session ingress optimistic concurrency

因为 URL 是外部传入，本文不额外推断固定路径，只记录协议行为。

---

## 10. Files API / Bundle Seed

这组接口与 remote 主链路关系很强，因为 `seed_bundle_file_id` 直接影响远端容器如何初始化代码。

## 10.1 `POST /v1/files`

### 调用方

- `uploadFile()` in `src/services/api/filesApi.ts`
- 被 `createAndUploadGitBundle()` 调用

### 认证

- OAuth token
- `anthropic-version: 2023-06-01`
- `anthropic-beta: files-api-2025-04-14,oauth-2025-04-20`

### 请求体

multipart/form-data，至少包括：

- `file`
- `purpose=user_data`

### 响应

客户端只强依赖：

```ts
response.data.id
```

### 用途

- 上传 git bundle
- 作为 `seed_bundle_file_id` 注入 `SessionContext`

---

## 10.2 `GET /v1/files/{fileId}/content`

### 调用方

- `downloadFile()` in `src/services/api/filesApi.ts`

### 认证

- OAuth token
- `anthropic-version: 2023-06-01`
- `anthropic-beta: files-api-2025-04-14,oauth-2025-04-20`

### 用途

- 下载 public files content
- 用于 session startup 附件下载等场景

---

## 10.3 `GET /v1/files`

### 调用方

- `filesApi.ts` 中用于查询文件列表的逻辑

### 用途

- 代码注释说明支持 `after_created_at` 查询
- 用于查找 session 相关上传文件

本文未展开其完整 schema，因为当前任务中 remote 主链路对它的依赖弱于上传/下载。

---

## 10.4 Git Bundle 链路

`createAndUploadGitBundle()` 的接口链路是：

```text
本地 git bundle create
  -> POST /v1/files
  -> 返回 file_id
  -> POST /v1/sessions 时把 file_id 放进 seed_bundle_file_id
```

这意味着 Files API 在 remote 中不是可有可无的附件能力，而是：

**远端执行环境初始化旁路。**

---

## 11. GitHub 远程访问预检接口

这组接口不直接创建 session，但会影响是否能走 GitHub clone，而不是退回 bundle 模式。

## 11.1 `GET /api/oauth/organizations/{orgUUID}/code/repos/{owner}/{repo}`

### 调用方

- `checkGithubAppInstalled()` in `src/utils/background/remote/preconditions.ts`

### 认证

- OAuth access token
- `x-organization-uuid`

### 响应结构

客户端期望：

```ts
{
  repo: {
    name: string,
    owner: { login: string },
    default_branch: string
  },
  status: {
    app_installed: boolean,
    relay_enabled: boolean
  } | null
}
```

### 用途

- 判断 GitHub App 是否安装在当前 repo 上
- remote 创建时决定能否直接走 GitHub source

---

## 11.2 `GET /api/oauth/organizations/{orgUUID}/sync/github/auth`

### 调用方

- `checkGithubTokenSynced()` in `src/utils/background/remote/preconditions.ts`

### 认证

- OAuth access token
- `x-organization-uuid`

### 响应判定

客户端只看：

```ts
response.data?.is_authenticated === true
```

### 用途

- 判断用户是否通过 `/web-setup` 同步了 GitHub token
- 作为 GitHub App 未安装时的第二层访问手段

---

## 12. Trusted Device 接口扩展

## 12.1 `POST /api/auth/trusted_devices`

### 调用方

- `enrollTrustedDevice()` in `src/bridge/trustedDevice.ts`

### 认证

- OAuth access token

### 响应

```ts
{
  device_token?: string,
  device_id?: string
}
```

### 用途

- 注册当前设备
- 将 token 持久化到 secure storage
- 供 bridge 请求头 `X-Trusted-Device-Token` 使用

### 约束

代码注释指出：

- enrollment 受 session freshness 限制
- 必须在刚登录后尽快执行
- 不能作为 bridge 403 后的 lazy fallback

---

## 13. Attachments 接口扩展

## 13.1 `GET /api/oauth/files/{file_uuid}/content`

### 调用方

- `resolveOne()` in `src/bridge/inboundAttachments.ts`

### 认证

- OAuth access token

### 用途

- 下载 web composer 上传的 inbound 附件
- 写入本地 `~/.claude/uploads/{sessionId}/`
- 转成 `@"path"` 注入 REPL 输入

### 与 Files API 的区别

它不是 `public /v1/files/{id}/content`，而是 OAuth 文件内容接口，适用于 bridge inbound attachment 流程。

---

## 14. Direct Connect 接口扩展

Direct Connect 主链路很简单，但协议层有两个值得单独写清的点。

## 14.1 创建接口：`POST {serverUrl}/sessions`

### 请求体

```ts
{
  cwd: string,
  dangerously_skip_permissions?: true
}
```

### 响应体

```ts
{
  session_id: string,
  ws_url: string,
  work_dir?: string
}
```

### 客户端处理

- 用 `connectResponseSchema()` 做严格验证
- `work_dir` 存在时，会把本地 cwd 切过去

---

## 14.2 WebSocket 协议

### 入站

- `control_request`
- `control_response`
- `keep_alive`
- `SDKMessage`

### 出站

- user message
- permission response
- interrupt request

### 备注

Direct Connect 的 WebSocket 数据是以换行切开的 JSON 文本，而不是浏览器标准的单消息单事件语义；客户端会自行 `split('\n')` 逐行解析。

---

## 15. 接口依赖图

```text
OAuth Token
  -> /api/oauth/profile
  -> /v1/sessions/*
  -> /v1/environment_providers*
  -> /v1/environments/bridge*
  -> /v1/code/sessions*
  -> /api/oauth/organizations/{org}/code/repos/*
  -> /api/oauth/organizations/{org}/sync/github/auth
  -> /v1/files*
  -> /api/auth/trusted_devices
  -> /api/oauth/files/{uuid}/content

Remote Session
  -> /v1/sessions
  -> /v1/sessions/{id}
  -> /v1/sessions/{id}/events
  -> /v1/sessions/ws/{id}/subscribe
  -> /v1/sessions/{id}/archive

Env-based Bridge
  -> /v1/environments/bridge
  -> /v1/environments/{id}/work/poll
  -> /v1/environments/{id}/work/{id}/ack
  -> /v1/environments/{id}/work/{id}/heartbeat
  -> /v1/environments/{id}/bridge/reconnect
  -> /v1/sessions/{id}/events
  -> /v1/sessions/{id}/archive

Env-less Bridge
  -> /v1/code/sessions
  -> /v1/code/sessions/{id}/bridge
  -> /v1/code/sessions/{id}/worker/register
  -> /v1/code/sessions/{id}/worker/events/stream

Teleport / Resume
  -> /v1/code/sessions/{id}/teleport-events
  -> /v1/session_ingress/session/{id}

Bundle Seed
  -> /v1/files
  -> seed_bundle_file_id -> /v1/sessions

Direct Connect
  -> {serverUrl}/sessions
  -> ws_url
```

---

## 16. 与上一版相比补充了什么

本扩展版新增覆盖：

1. OAuth token exchange / refresh / profile / roles / API key 接口
2. GitHub App 安装检查与 token sync 检查接口
3. `teleport-events` 与旧 `session_ingress` transcript 拉取接口
4. Files API 上传/下载与 bundle seed 链路
5. trusted device enrollment 接口
6. inbound attachment 的 OAuth 文件内容接口

换句话说，上一版回答的是：

**“remote 主流程会直接打哪些 API？”**

这一版回答的是：

**“为了让 remote 主流程真正跑起来，系统外围还依赖哪些服务端接口？”**

---

## 17. 结论

如果只看 `remote/bridge/direct-connect` 主链路，很容易误以为这套系统只依赖：

- Sessions API
- Bridge API
- Code Sessions API

但从完整代码看，真正支撑 remote 体系落地的接口面明显更宽：

1. **认证层**
   OAuth token/profile/roles/trusted-device

2. **环境准备层**
   environment providers、GitHub 访问预检

3. **代码种子层**
   Files API、bundle upload、seed_bundle_file_id

4. **历史回放层**
   teleport-events、session ingress transcript

5. **主控制层**
   sessions / bridge / code sessions / direct connect

因此，Claude Code 的 remote 能力在服务端接口层面，本质上不是“一个 API”，而是：

**以会话资源管理为中心、以 worker 调度和代码种子注入为补充、以 OAuth 和 transcript 基础设施为底座的一组复合接口系统。**
