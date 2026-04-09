# Remote Session，Bridge Environments，Code Session 之间的关系

本文聚焦一个容易混淆的问题：

`Remote Session`、`Bridge Environments`、`Code Session` 到底分别是什么，它们处在系统的哪一层，彼此之间是什么关系？

结论先行：

- `Remote Session` 是“远程会话语义层”
- `Bridge Environments` 是“env-based 的 worker 调度层”
- `Code Session` 是“env-less 的 worker 接入层”

换句话说，`Remote Session` 定义“用户看到的远程会话是什么”，而 `Bridge Environments` 和 `Code Session` 定义“实际执行这个远程会话的 worker 怎么接进来”。

---

## 1. 三者不在同一抽象层

很多初学者第一次看这套代码时，会误以为：

- `Remote Session` 是一种功能
- `Bridge Environments` 是另一种功能
- `Code Session` 又是第三种并列功能

这其实不准确。

更合理的理解方式是分层：

```text
用户 / 本地 CLI
  |
  | 输入、展示、审批
  v
Remote Session 语义层
  | 负责：session 生命周期、消息流、control_request/control_response、archive
  | 接口：/v1/sessions/*, /v1/sessions/ws/*
  |
  +-------------------------------+
  |                               |
  | 两种把 worker 接进来的方式     |
  v                               v
Bridge Environments           Code Session
(env-based)                  (env-less)
  |                               |
  | 负责：worker 调度              | 负责：worker 凭据直发
  | 接口：/v1/environments/*      | 接口：/v1/code/sessions/*
  |                               |
  +---------------+---------------+
                  |
                  v
        Worker Transport / Ingress
        SSE / WS / worker JWT / session ingress token
                  |
                  v
            实际运行的 Claude Code worker
```

这张图表达了三个关键事实：

1. `Remote Session` 是上层的产品语义。
2. `Bridge Environments` 和 `Code Session` 是下层的 worker 接入机制。
3. 两种 worker 接入机制最终都要服务于同一个 session 模型。

---

## 2. `Remote Session` 是什么

`Remote Session` 的本质是：

“Agent 在远端执行，本地 CLI 只负责连接、展示、输入和审批。”

所以它关注的是：

- 如何创建或附着到一个 session
- 如何把用户输入送到远端
- 如何接收远端返回的 `SDKMessage`
- 如何处理 `control_request`
- 如何把本地审批结果回传成 `control_response`
- 如何归档 session

对应接口主要是：

- `POST /v1/sessions`
- `GET /v1/sessions/{id}`
- `POST /v1/sessions/{id}/events`
- `GET /v1/sessions/{id}/events`
- `PATCH /v1/sessions/{id}`
- `POST /v1/sessions/{id}/archive`
- `GET /v1/sessions/ws/{id}/subscribe?organization_uuid=...`

对应源码主要是：

- [SessionsWebSocket.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/remote/SessionsWebSocket.ts)
- [RemoteSessionManager.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/remote/RemoteSessionManager.ts)

这里最重要的一点是：

`Remote Session` 本身并不负责“worker 是怎么被调度起来的”。它描述的是会话语义，而不是 worker orchestration。

---

## 3. `Bridge Environments` 是什么

`Bridge Environments` 是传统的 `remote-control` 方案。

它解决的问题不是“session 是什么”，而是：

“如何把本地机器注册成一个可被服务端调度的执行环境，并让服务端把工作派给它。”

它的主流程是：

1. 本地 bridge 向服务端注册一个 environment
2. 服务端把某个 session work 派发到这个 environment
3. 本地 bridge 轮询拿到 work
4. 本地 bridge ack 这份 work
5. 本地 bridge 拉起实际的 Claude Code worker
6. worker 通过 ingress/transport 开始处理 session

核心接口是：

- `POST /v1/environments/bridge`
- `GET /v1/environments/{id}/work/poll`
- `POST /v1/environments/{id}/work/{id}/ack`
- `POST /v1/environments/{id}/work/{id}/stop`
- `POST /v1/environments/{id}/work/{id}/heartbeat`
- `POST /v1/environments/{id}/bridge/reconnect`
- `DELETE /v1/environments/bridge/{id}`

对应源码主要是：

- [bridgeApi.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/bridgeApi.ts)
- [replBridge.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/replBridge.ts)
- [types.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/types.ts)

这里的中心对象是：

- `environment_id`
- `environment_secret`
- `work`
- `work lease`
- `session_ingress_token`

因此 `Bridge Environments` 的本质是一个 worker 调度面，而不是用户直接感知的 session 面。

---

## 4. `Code Session` 是什么

`Code Session` 是较新的 env-less bridge 方案。

它的核心思想是：

“如果目标只是把 bridge 接到 worker 层，就不一定需要 Environments API 这层 poll/ack/heartbeat 调度面。”

也就是说，`Code Session` 试图绕过传统的 `Bridge Environments` 调度层，直接完成：

1. 创建一个 code session
2. 为这个 code session 换取 worker JWT
3. 直接连上 `/worker/*` 传输层

核心接口是：

- `POST /v1/code/sessions`
- `POST /v1/code/sessions/{id}/bridge`
- `POST /v1/code/sessions/{id}/worker/register`
- `GET /v1/code/sessions/{id}/worker/events/stream`

对应源码主要是：

- [codeSessionApi.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/codeSessionApi.ts)
- [remoteBridgeCore.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/remoteBridgeCore.ts)
- [replBridgeTransport.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/replBridgeTransport.ts)

[remoteBridgeCore.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/remoteBridgeCore.ts) 文件开头已经把这个意图写得很清楚：

- env-less 不是“换一种 transport”
- env-less 是“去掉 Environments API 调度层”

因此：

- `Bridge Environments` 和 `Code Session` 的关系是“老路径 vs 新路径”
- 它们都服务于 bridge，但 worker 接入方式不同

---

## 5. 三条主链路的统一时序图

### 5.1 纯 `Remote Session`

```text
本地 CLI/UI
  -> POST /v1/sessions                创建 session
  -> WS /v1/sessions/ws/{id}/subscribe 订阅远端事件
  -> POST /v1/sessions/{id}/events    发送用户输入
  <- WS SDKMessage / control_request  收到输出与控制请求
  -> POST /v1/sessions/{id}/events    回传 control_response
  -> POST /v1/sessions/{id}/archive   结束归档
```

特点：

- 没有本地 worker 调度
- 没有 `Environments API`
- 没有 `Code Session API`
- 本地只是远端 session client

---

### 5.2 `Bridge Environments` 路径

```text
本地 Bridge
  -> POST /v1/environments/bridge              注册 environment
  <- environment_id, environment_secret

服务端调度
  -> POST /v1/sessions                         创建一个要执行的 session
  -> 把 session 变成 work 绑定到 environment

本地 Bridge
  -> GET /v1/environments/{env}/work/poll      轮询 work
  <- work(sessionId, secret, lease...)

  -> POST /v1/environments/{env}/work/{work}/ack
  -> 用 work secret/session token 建立 worker transport
  -> 本地拉起 Claude Code worker

运行中
  <-> worker transport / session ingress       双向消息流
  -> POST /v1/environments/{env}/work/{work}/heartbeat
  -> POST /v1/sessions/{id}/events             回传权限响应等
  -> POST /v1/sessions/{id}/archive            会话结束
  -> DELETE /v1/environments/bridge/{env}      注销 environment
```

这个链路说明：

- `session` 仍然是用户感知的主对象
- `environment` 只是调度和承载 worker 的对象
- 因此 env-based bridge 是“worker orchestration + session compatibility”双层叠加

---

### 5.3 `Code Session` 路径

```text
本地 Bridge
  -> POST /v1/code/sessions                    创建 code session
  <- cse_* session id

  -> POST /v1/code/sessions/{id}/bridge        直接换 worker_jwt
  <- worker_jwt, api_base_url, worker_epoch

  -> GET /v1/code/sessions/{id}/worker/events/stream
  -> 建立 SSE / CCR v2 transport
  -> 本地拉起 worker 或直接绑定 transport

运行中
  <-> /v1/code/sessions/{id}/worker/*          worker 专用收发
  -> token 快过期时再次 POST /bridge           刷新 jwt + epoch
  -> 结束时 POST /v1/sessions/{compatId}/archive
```

这个链路说明：

- 不再需要 environment register/poll/ack/heartbeat
- `/bridge` 直接承担 worker 凭据发放职责
- 但 session 兼容层没有消失

这就是为什么 `Code Session` 看起来是新接口族，但系统里仍然保留 `/v1/sessions/*`。

---

## 6. 三者之间的精确关系

可以把三者的关系总结成下面四句话：

1. `Remote Session` 是用户会话层，不是 worker 调度层。
2. `Bridge Environments` 是旧的 worker 调度方案。
3. `Code Session` 是新的 worker 直连方案。
4. `Code Session` 替代的是 `Bridge Environments` 的一部分职责，不是替代 `Remote Session` 本身。

也就是说：

```text
Remote Session != Bridge Environments != Code Session
```

而是：

```text
Remote Session = 会话层
Bridge Environments / Code Session = 两种 worker 接入层
```

---

## 7. 为什么会看到 `/v1/code/sessions/*` 和 `/v1/sessions/*` 同时存在

这是整套设计里非常工业化的一点。

两套接口并存，是因为它们承担不同职责：

- `/v1/code/sessions/*` 负责 worker 接入、worker JWT、worker stream
- `/v1/sessions/*` 负责通用 session 语义、事件兼容、标题更新、归档

所以在 bridge 路径里，会出现这样的组合：

- 启动 worker 时使用 `/v1/code/sessions/*`
- 归档 session 时继续使用 `/v1/sessions/{compatId}/archive`

这说明新执行面没有强行把旧 session 语义一起推翻，而是采用了“新执行面 + 旧兼容面”的渐进式演进。

相关兼容逻辑可见：

- [sessionIdCompat.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/sessionIdCompat.ts)
- [remoteBridgeCore.ts](/Users/lukki/codes/github/nolan0707/lukki/agent/vendor/Claude-code-open/src/bridge/remoteBridgeCore.ts)

---

## 8. 从系统设计角度看，为什么要这样拆

把这三者拆开，有几个工程上的好处。

### 8.1 会话语义和执行接入解耦

如果 session 层直接绑定某一种 worker 调度方式，那么后续任何执行面演进都会影响用户会话模型。

而当前设计允许：

- 上层继续保持 `session`
- 下层从 `environment-based` 演进到 `code-session-based`

这降低了迁移成本。

### 8.2 兼容旧系统

`/v1/sessions/*` 已经承载了大量现有语义：

- session 生命周期
- events
- archive
- title update
- 客户端展示逻辑

如果完全替换，代价很大。保留 compat 层可以让新链路逐步落地。

### 8.3 让 bridge 能从“调度系统”退化为“连接系统”

传统 env-based bridge 更像一个 worker scheduler client。

而 env-less code session 让 bridge 更接近：

- 创建执行上下文
- 获取 worker 凭据
- 建立 transport

这减少了状态机复杂度，也减少了 lease / poll / reclaim 这类调度问题。

---

## 9. 一句话记忆法

如果只记一条，可以记成：

```text
Remote Session = 会话是什么
Bridge Environments = 老的 worker 怎么接入
Code Session = 新的 worker 怎么接入
```

或者更口语一点：

```text
用户想开一个远程会话
  -> 系统需要一个 session
  -> 还需要把某个 worker 接进去
  -> 接 worker 可以走 Environments，也可以走 Code Session
```

这就是三者最本质的关系。
