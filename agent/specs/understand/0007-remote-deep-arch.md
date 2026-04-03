# Claude-code-open 远程与桥接架构深度解读（专家版）

## 1. 文档目标

本文在 [0001-project-arch.md](./0001-project-arch.md) 和 [0007-remote-arch.md](./0007-remote-arch.md) 的基础上，面向 AI 应用架构师、分布式系统工程师、算法平台专家，深度解读 `vendor/Claude-code-open` 中的“远程与桥接架构”。

相较于 `0007-remote-arch.md`，本文不再停留于“这三种模式分别是什么”，而是聚焦更底层的问题：

1. 这套架构到底把哪些逻辑分布到了本地 UI、远端 Agent、桥接层、服务端控制面？
2. `SDKMessage` 事件流和 `control_request/control_response` 控制流各自扮演什么角色？
3. Bridge 为什么引入 `environment`、`work`、`lease`、`worker_jwt`、`worker_epoch` 这些看似重型的概念？
4. 为什么既存在 env-based bridge，又存在 env-less bridge？
5. 这套系统如何保证远程控制下的消息顺序、一致性、权限竞态、断线恢复和安全边界？
6. 其中哪些设计体现了明显的工业级工程权衡，甚至可以视为关键创新点？

本文主要基于以下代码：

- `src/main.tsx`
- `src/hooks/useRemoteSession.ts`
- `src/hooks/useDirectConnect.ts`
- `src/hooks/useReplBridge.tsx`
- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`
- `src/remote/sdkMessageAdapter.ts`
- `src/remote/remotePermissionBridge.ts`
- `src/bridge/types.ts`
- `src/bridge/bridgeApi.ts`
- `src/bridge/initReplBridge.ts`
- `src/bridge/replBridge.ts`
- `src/bridge/remoteBridgeCore.ts`
- `src/bridge/replBridgeTransport.ts`
- `src/bridge/bridgeMessaging.ts`
- `src/bridge/flushGate.ts`
- `src/bridge/bridgePointer.ts`
- `src/bridge/workSecret.ts`
- `src/bridge/bridgeEnabled.ts`
- `src/bridge/envLessBridgeConfig.ts`
- `src/bridge/bridgePermissionCallbacks.ts`
- `src/bridge/sessionRunner.ts`
- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`
- `src/entrypoints/sdk/controlSchemas.ts`
- `src/entrypoints/agentSdkTypes.ts`
- `src/cli/structuredIO.ts`

---

## 2. 总结先行

如果用系统设计语言描述，Claude Code 的远程与桥接架构本质上是一个：

**“本地交互壳 + 可迁移执行面 + 双通道协议 + 分层恢复机制”的 Agent 控制系统。**

它不是单纯“远程跑模型”，也不是单纯“WebSocket 连一下”。

它解决的是一个更复杂的问题：

**当 Agent 的执行位置、用户交互位置、权限裁决位置、会话控制位置不再重合时，如何仍然让系统表现得像一个连续、可控、可恢复的单体对话系统。**

为了实现这一点，代码中实际上存在三条互补但边界清晰的远程路径：

```text
1. Remote Session
本地终端附着到远端运行中的 Claude Code 会话

2. Bridge / Remote Control
把本地 Claude Code REPL 暴露为远端可调度的执行 worker

3. Direct Connect
本地终端直接连接某个自建 Claude Code server
```

这三条路径共享三个核心思想：

1. **事件流与控制流分离**
   普通消息走 `SDKMessage`，控制动作走 `control_request / control_response`。

2. **UI 与执行环境分离**
   终端 UI 尽量保持本地；执行面既可以是远端 session，也可以是本地 bridge worker。

3. **恢复优先于重建**
   系统优先恢复同一 session / 同一 transport / 同一 environment，而不是简单重开。

---

## 3. 架构全景：三种模式不是三套“功能”，而是三种部署拓扑

## 3.1 Remote Session：远端执行，本地展示与审批

Remote Session 的部署拓扑是：

```text
本地 REPL
  负责：
    - 输入采集
    - 消息渲染
    - 权限审批 UI
    - 中断控制
  通过：
    - Sessions WebSocket 订阅事件
    - HTTP POST 发送用户输入
  连接到：
    - 远端 CCR Session（真正执行对话和工具）
```

这里的本地 CLI 并不执行工具，它更接近一个“远端 agent 的控制终端”。

---

## 3.2 Bridge / Remote Control：本地执行，远端发起和控制

Bridge 的部署拓扑是：

```text
claude.ai / 远端控制端
  负责：
    - 发起 session
    - 派发 work
    - 发送控制请求

Bridge server / session ingress
  负责：
    - environment 注册与调度
    - work lease
    - transport ingress
    - worker_jwt 发放

本地 REPL / Bridge worker
  负责：
    - 真正执行 Claude Code 会话
    - 向远端同步消息
    - 接收远端控制动作
    - 回传结果和权限状态
```

这已经不是“远程聊天”，而是：

**本地 REPL 被包装成了一个受控的、会话化的远程 worker。**

---

## 3.3 Direct Connect：本地附着到另一台 Claude Code 服务

Direct Connect 的拓扑最简单：

```text
本地 REPL
  -> POST /sessions
  -> connect ws_url
  -> 远端 server
```

它不引入 `environment` 和 `work`，说明它是“点对点连接”，不是“通过调度层连接”。

---

## 4. 控制模型：事件流和控制流的双平面架构

理解整套远程架构，最关键的是先理解协议分层。

## 4.1 数据平面：`SDKMessage`

数据平面负责承载对话与执行事件。

来源包括：

- 用户消息
- assistant 消息
- stream event
- result
- system/status
- tool_progress
- compact_boundary

这些消息主要解决：

- 对话内容展示
- 增量流式渲染
- 工具执行进度
- 会话状态感知

在代码层，`src/entrypoints/agentSdkTypes.ts` 统一暴露 SDK 公共消息类型，`src/entrypoints/sdk/coreTypes.ts` 再 re-export 生成的核心 serializable types。

换句话说：

**`SDKMessage` 是“Agent 执行产生了什么”的事实流。**

---

## 4.2 控制平面：`control_request / control_response / control_cancel_request`

控制平面负责承载用户或宿主对 session 行为的干预。

从 `src/entrypoints/sdk/controlSchemas.ts` 可以看到几类关键控制请求：

- `interrupt`
- `can_use_tool`
- `set_permission_mode`
- `set_model`
- `set_max_thinking_tokens`
- 以及 MCP / context / settings 相关控制请求

桥接和远程场景最核心的是前 5 类。

关键结构如下：

```ts
type SDKControlRequest = {
  type: 'control_request'
  request_id: string
  request: SDKControlRequestInner
}

type SDKControlResponse = {
  type: 'control_response'
  response:
    | { subtype: 'success'; request_id: string; response?: Record<string, unknown> }
    | { subtype: 'error'; request_id: string; error: string }
}

type SDKControlCancelRequest = {
  type: 'control_cancel_request'
  request_id: string
}
```

这说明控制协议具备两个非常重要的工业属性：

1. **请求-响应关联性**
   通过 `request_id` 做严格关联。

2. **可取消性**
   `control_cancel_request` 明确允许撤销悬挂中的控制请求，避免 UI 与执行面状态漂移。

因此：

**控制协议不是“附属消息”，而是完整的会话控制 RPC 层。**

---

## 4.3 为什么必须双平面

如果把权限审批、中断、模式切换都塞进普通消息流，会出现三个问题：

1. 普通消息是异步、可延迟、可重放的，不适合承载强时序控制。
2. 权限审批必须带 request-response 语义，否则会出现悬挂请求和重复裁决。
3. 某些控制动作不是“要显示给用户的内容”，而是“要立即驱动运行时状态变化”。

因此，这里采用双平面架构是正确且必要的：

- 数据平面保证内容连续性
- 控制平面保证运行时可控性

这是整套系统的第一层关键设计。

---

## 5. 模式路由：`main.tsx` 是部署拓扑切换器

`src/main.tsx` 在整个系统里承担的是 composition root 的角色。

对于远程与桥接相关路径，它至少负责 4 类分流：

### A. Remote Session 创建与附着

`--remote` 路径：

1. 创建远端 session
2. 切换本地到 remote mode
3. 传入 `remoteSessionConfig`
4. 启动 REPL

### B. Assistant / Viewer attach

`assistant [sessionId]` 路径：

1. 取 OAuth
2. 构造 `RemoteSessionConfig(viewerOnly=true)`
3. 启动 REPL 作为 viewer client

### C. Direct Connect

`connect/open <cc-url>` 路径：

1. `createDirectConnectSession()`
2. 取得 `session_id + ws_url`
3. 启动 REPL

### D. Bridge / Remote Control

`remote-control` 并不通过 Commander 正常执行，而是由更早的 fast-path 接管，最终进入：

- `bridgeMain.ts`
- 或 REPL 内 `initReplBridge()`

这说明：

**Bridge 不是一个普通命令，而是一个会改变 REPL 运行时结构的长期后台连接机制。**

---

## 6. Remote Session 深度设计

## 6.1 模块协作

Remote Session 由四个核心模块构成：

```text
useRemoteSession
  <-> RemoteSessionManager
        <-> SessionsWebSocket
        <-> sendEventToRemoteSession(HTTP)
  <-> sdkMessageAdapter
  <-> remotePermissionBridge
```

职责分工如下：

- `RemoteSessionManager`
  负责会话级控制逻辑

- `SessionsWebSocket`
  负责订阅、心跳、重连、控制消息发送

- `sdkMessageAdapter`
  负责协议消息到 REPL 内部消息的语义映射

- `useRemoteSession`
  负责 UI 集成、权限队列、状态同步和用户操作

---

## 6.2 `RemoteSessionConfig`：远程会话最小能力注入

`RemoteSessionConfig` 的定义体现了 Remote Session 的最小依赖原则：

```ts
type RemoteSessionConfig = {
  sessionId: string
  getAccessToken: () => string
  orgUuid: string
  hasInitialPrompt?: boolean
  viewerOnly?: boolean
}
```

值得注意的不是字段本身，而是几个设计意图：

### A. token 不是值，而是 closure

`getAccessToken` 用闭包而不是字符串，意味着：

- 重连时总能取到最新 token
- OAuth refresh 不需要重建所有上层对象

这是长连接系统的标准做法。

### B. `viewerOnly` 是行为约束位，不是展示标记

`viewerOnly` 会影响：

- Ctrl+C / Escape 是否发送 interrupt
- 60 秒无响应检测是否生效
- 标题是否被本地更新

也就是说它改变的是控制策略，而不仅是 UI。

### C. Remote Session 尽量不携带本地执行能力

config 里没有本地工具、权限策略、工作目录等执行侧信息，说明 Remote Session 有意识地保持“本地终端仅作为远端 session 的 client”。

---

## 6.3 `RemoteSessionManager`：会话控制器

`RemoteSessionManager` 的核心工作可以概括成一个小型会话中枢：

1. `connect()`
   创建 `SessionsWebSocket`

2. `handleMessage()`
   分类处理：
   - 普通 `SDKMessage`
   - `control_request`
   - `control_response`
   - `control_cancel_request`

3. `sendMessage()`
   通过 HTTP POST 向远端 session 发送用户消息

4. `respondToPermissionRequest()`
   将本地审批结果封装为 `control_response`

5. `cancelSession()`
   通过 `control_request{subtype:'interrupt'}` 中断远端

这里的关键点是：

**Remote Session 把“用户输入”和“控制动作”分到了两条不同通道：**

- 用户输入：HTTP 事件写入
- 控制动作：WebSocket 控制消息

这种设计并不常见，但很合理：

- 输入是 append-only 的 session event
- 中断/权限响应是时序敏感的控制信号

---

## 6.4 `SessionsWebSocket`：会话订阅层的状态机

`SessionsWebSocket` 的设计是一套相对克制但完整的连接状态机。

### 状态

```ts
type WebSocketState = 'connecting' | 'connected' | 'closed'
```

### 主要能力

1. fresh token connect
2. ping 保活
3. close code 分类
4. 限次重连
5. force reconnect

### 重连策略

普通断线：

- 最大重连 5 次
- 固定 2 秒回退

特殊 `4001 session not found`：

- 单独预算 3 次
- 随重试次数增加延迟

永久拒绝：

- `4003 unauthorized`
- 不重连，直接 close

这里体现了两个工业级判断：

1. **把业务语义嵌入 close code 解释**
   `4001` 不是简单“会话丢了”，还可能是 compact 窗口期的短暂不一致。

2. **重连预算按错误类型分层**
   不是全局统一 backoff，而是依据错误语义区分。

这在长连接系统里是很成熟的做法。

---

## 6.5 `sdkMessageAdapter`：协议语义适配层

`sdkMessageAdapter` 的意义不是“类型转换”，而是：

**把远端消息语义重新投影到 REPL 的本地认知模型中。**

典型映射：

- `assistant` -> `AssistantMessage`
- `stream_event` -> `StreamEvent`
- `result(error)` -> warning/info system message
- `system.init` -> “remote session initialized”
- `tool_progress` -> 信息型系统消息
- `compact_boundary` -> 压缩边界提示

最关键的是它对 user message 的条件性处理：

```ts
convertSDKMessage(msg, {
  convertToolResults?: boolean,
  convertUserTextMessages?: boolean
})
```

这是因为远程协议中的 user message 有两种截然不同的来源：

1. 本地用户刚刚输入的 prompt 的远端 echo
2. 远端历史回放 / tool_result / viewer-only 显示需要

如果没有这层适配：

- 普通模式会重复显示用户消息
- viewer 模式又会缺失必要信息

这是一种非常典型的“同一协议对象在不同宿主上下文下语义不同”的问题处理方式。

---

## 6.6 `useRemoteSession`：Remote Session 的运行时编排器

`useRemoteSession` 承担了真正的宿主集成职责。

它不是一个简单 Hook，而是 Remote Session 的本地运行时。

### 它维护的关键状态

- `managerRef`
- `sentUUIDsRef`
- `runningTaskIdsRef`
- `isCompactingRef`
- `responseTimeoutRef`
- `toolsRef`

这些状态说明它要同时管理：

- 消息一致性
- 远端任务生命周期
- 会话压缩状态
- 工具元信息引用稳定性
- 无响应检测

### 关键机制 1：echo dedup

本地发送消息后，服务端和 worker 可能分别回显同一 uuid。

因此实现不是简单的：

- “收到一次就删除”

而是：

- 使用 `BoundedUUIDSet(50)` 做短期有界保留
- 命中后不删除

这避免了多源 echo 导致的二次穿透。

### 关键机制 2：远端权限请求本地化

当收到 `can_use_tool` 时：

1. `createSyntheticAssistantMessage()`
2. `findToolByName()` 或 `createToolStub()`
3. 构造 `ToolUseConfirm`
4. 将用户裁决回传为 `RemotePermissionResponse`

这里最妙的地方在于：

**本地 UI 复用了现有权限组件，但远端执行环境并不需要把真实 Tool 实现传回本地。**

系统只需要：

- 可显示的工具名
- 输入参数
- 描述
- 结果回调

这是一种非常“接口导向”的能力复用。

### 关键机制 3：远端任务计数与 loading 管理

远端 `task_started / task_notification` 被本地映射成后台任务计数。

说明系统不仅同步内容，还同步执行状态。

---

## 7. Direct Connect 深度设计

Direct Connect 是三条路径中最接近“经典客户端-服务端连接”的方案。

## 7.1 创建过程

`createDirectConnectSession()` 做的是：

```text
POST {serverUrl}/sessions
  body: { cwd, dangerously_skip_permissions? }
-> validate response schema
-> return {
     config: { serverUrl, sessionId, wsUrl, authToken },
     workDir?
   }
```

这里的关键是：

- 返回结构用 Zod schema 验证
- 连接失败统一抛 `DirectConnectError`

这说明 Direct Connect 是显式面向外部 server 的，不假设对端可靠。

---

## 7.2 `DirectConnectSessionManager`

它负责：

1. 用 `wsUrl` 建 WebSocket
2. 按行拆分 NDJSON
3. 解析并识别：
   - `control_request`
   - 普通 `SDKMessage`
4. 发送：
   - user message
   - permission response
   - interrupt

与 Remote Session 相比，它少了：

- session subscribe API
- close code 分类重连
- 独立发送 HTTP 事件接口
- environment/work dispatch

因此它更像：

**纯粹的协议终端适配层。**

---

## 8. Bridge / Remote Control 深度设计

Bridge 是整个远程体系中设计最复杂、最有工程含量的部分。

其复杂性来源于一个事实：

**它必须把一个原本面向本地交互的 REPL，变成一个可被远端调度、可被远端控制、又不能丢失本地一致性保障的 worker。**

---

## 8.1 Bridge 的领域模型

`src/bridge/types.ts` 定义了桥接层的核心领域对象。

### A. `BridgeConfig`

```ts
type BridgeConfig = {
  dir: string
  machineName: string
  branch: string
  gitRepoUrl: string | null
  maxSessions: number
  spawnMode: 'single-session' | 'worktree' | 'same-dir'
  bridgeId: string
  workerType: string
  environmentId: string
  reuseEnvironmentId?: string
  apiBaseUrl: string
  sessionIngressUrl: string
  ...
}
```

这里可以看出 bridge 不是“连一个 session”这么简单，而是先注册一个“环境实体”。

也就是说，服务端视角下，session 和 environment 是解耦的：

- session：逻辑会话
- environment：可承载该会话的执行环境

### B. `WorkResponse`

```ts
type WorkResponse = {
  id: string
  type: 'work'
  environment_id: string
  state: string
  data: { type: 'session' | 'healthcheck'; id: string }
  secret: string
  created_at: string
}
```

这说明桥接层不是直接“subscribe 某个 session”，而是通过调度系统领取 work。

### C. `WorkSecret`

```ts
type WorkSecret = {
  version: number
  session_ingress_token: string
  api_base_url: string
  sources: ...
  auth: ...
  mcp_config?: ...
  environment_variables?: ...
  use_code_sessions?: boolean
}
```

`WorkSecret` 的作用极其关键：

- 作为当前 work 的短时凭据容器
- 作为 transport 模式选择器
- 作为远端工作环境配置载体

它不是普通 token，而是一个“工作执行上下文胶囊”。

### D. `ReplBridgeTransport`

这是桥接层最重要的抽象接口之一：

```ts
type ReplBridgeTransport = {
  write(message): Promise<void>
  writeBatch(messages): Promise<void>
  close(): void
  isConnectedStatus(): boolean
  getStateLabel(): string
  setOnData(cb): void
  setOnClose(cb): void
  setOnConnect(cb): void
  connect(): void
  getLastSequenceNum(): number
  droppedBatchCount: number
  reportState(state): void
  reportMetadata(metadata): void
  reportDelivery(eventId, status): void
  flush(): Promise<void>
}
```

这个接口说明 bridge 的核心并不是某个具体 WebSocket/SSE，而是：

**在“读流 + 写流 + 生命周期 + 恢复游标 + 状态上报”这几个能力上定义统一 transport 抽象。**

这是后续支持 v1/v2 双栈的关键。

---

## 8.2 Bridge API：环境调度而非直接连接

`src/bridge/bridgeApi.ts` 定义了 bridge 与服务端环境层交互的 API：

- `registerBridgeEnvironment()`
- `pollForWork()`
- `acknowledgeWork()`
- `stopWork()`
- `deregisterEnvironment()`
- `archiveSession()`
- `reconnectSession()`
- `heartbeatWork()`
- `sendPermissionResponseEvent()`

这组接口本身已经说明其本质是：

**一个环境调度协议，而不是单纯会话协议。**

### 关键观察

1. `register -> poll -> ack -> heartbeat -> stop -> deregister`
   这是典型 worker lease lifecycle。

2. `reconnectSession(environmentId, sessionId)`
   说明 session 恢复被设计成显式能力，而不是隐式副作用。

3. `withOAuthRetry()`
   bridge API 将 OAuth 401 refresh 抽象成可注入回调，既避免耦合 REPL 全量模块，又允许 daemon/特殊环境复用。

4. `validateBridgeId()`
   所有 server-provided ID 在进入 URL path 前都要经过 allowlist 验证，防止路径注入与 traversal。

这最后一点很容易被忽略，但其实是明显的工业安全设计。

---

## 8.3 `initReplBridge`：REPL wrapper 与核心隔离

`initReplBridge()` 的职责不是桥接本身，而是：

**从 REPL / bootstrap 世界收集上下文，然后把它压缩成一个纯参数化的 bridge core 输入。**

它处理：

- bridge entitlement gate
- OAuth 和 org policy 检查
- token refresh
- dead-token cross-process backoff
- git 信息
- 会话标题推导
- GrowthBook gate
- `viewerOnly / outboundOnly / perpetual` 等模式参数

这里有两个非常关键的设计约束。

### A. bootstrap 隔离

`replBridge.ts` / `remoteBridgeCore.ts` 不能直接依赖 REPL 重模块树。

原因在注释里写得很清楚：

- 否则会把 command registry / React tree / sessionStorage 等巨大依赖拖进 daemon / SDK bundle

所以它大量采用“依赖注入 + 动态 import”。

这是一种很成熟的 bundle hygiene 设计。

### B. gating 前置

桥接初始化前要先完成：

- entitlement
- OAuth
- organization policy
- version floor

这可以避免：

- 无意义网络请求
- 无限 401 循环
- 用户看到无行动建议的失败

它不只是“体验优化”，也是服务端保护策略。

---

## 8.4 Env-based Bridge 核心状态机：`replBridge.ts`

`replBridge.ts` 是 env-based bridge 的核心。

可以把它理解为一个包含五层子状态机的 orchestrator：

1. environment registration
2. session creation / reuse
3. work polling
4. transport lifecycle
5. teardown / reconnect

### 主流程

```text
registerBridgeEnvironment
  -> tryReconnectInPlace(prior env/session)?
  -> createBridgeSession
  -> writeBridgePointer
  -> startWorkPollLoop
  -> onWorkReceived
      -> decodeWorkSecret
      -> select v1/v2 transport
      -> wire callbacks
      -> initial flush
      -> steady state
  -> reconnect / teardown
```

---

## 8.5 为什么要有 `environment`

很多人第一次看 bridge 会问：

“为什么不直接 session 建好以后让本地 CLI 连上就行？为什么还要 environment 和 work？”

答案是：因为 Bridge 不是简单的 attach，而是远端调度本地执行环境。

引入 environment 后，系统获得了以下能力：

1. **会话和执行环境解耦**
   session 可以在逻辑上存在，而 environment 可以掉线、重连、迁移。

2. **worker 容量建模**
   `maxSessions` 可以让服务端感知 capacity。

3. **work lease / heartbeat**
   worker 失活时可自动回收并重派。

4. **多拓扑支持**
   `single-session / worktree / same-dir` 等 spawn mode 都可建立在 environment 之上。

因此 environment 是 bridge 调度模型成立的前提。

---

## 8.6 `startWorkPollLoop()`：调度心脏

这是整个 env-based bridge 里最有系统设计味道的函数之一。

它承担了四个角色：

1. work dispatcher client
2. lease maintainer
3. failure detector
4. recovery trigger

### 输入接口

它接受一组高层依赖而非闭包耦合内部状态：

- `api`
- `getCredentials`
- `signal`
- `onWorkReceived`
- `onEnvironmentLost`
- `isAtCapacity`
- `capacitySignal`
- `getHeartbeatInfo`
- `onHeartbeatFatal`

这说明它被设计成一个可重用的轮询内核，而不是写死在 REPL 逻辑里。

### 状态逻辑

#### 情况 1：poll 到空结果

- 空闲时用 `poll_interval_ms_not_at_capacity`
- 满载时切换到 heartbeat/低频 poll 模式

#### 情况 2：拿到 work

- decode secret
- ack work
- 如果是 session work，则交给 `onWorkReceived`

#### 情况 3：heartbeat fatal

- transport 失效
- work state 清空
- fast-poll 获取新 token / 新 dispatch

#### 情况 4：404 environment deleted

- 调 `onEnvironmentLost()`
- 尝试 recreate env 并恢复 session

#### 情况 5：fatal 401/403/404/410

- 触发整体 teardown

### 工业级设计点

#### A. “容量满”不是停止，而是降级为心跳模式

这比简单暂停更成熟，因为它仍然保持：

- liveness
- work lease 有效
- 异常唤醒能力

#### B. process suspension detector

代码显式检测 sleep/wake 场景：

- poll error 间隔异常大
- at-capacity sleep 严重超时

这说明系统考虑了：

- 笔记本合盖
- VM suspend
- 进程冻结

而不是只考虑“网络断开”。

#### C. 分层恢复

transport 级错误和 environment 级错误分开处理，不会一刀切 teardown。

这直接提升了会话连续性。

---

## 8.7 Session 恢复策略：不是重试，而是“同一身份恢复”

`replBridge.ts` 最有价值的设计之一，是它对恢复的层次化处理。

### 第一层：transport 级恢复

transport 掉了但 env/session 还活着：

- 先保留 session identity
- 尽量只重建 transport

### 第二层：environment reconnect in place

如果 environment 丢了：

1. 重新 register 同一个 `reuseEnvironmentId`
2. 如果服务端返还的还是原 environment
3. 再调用 `reconnectSession()`

这样可以保住：

- 当前 session id
- 远端页面 URL
- 已同步历史

### 第三层：fresh session fallback

如果 reconnect in place 失败：

- archive old session
- create new session
- 重写 bridge pointer
- 清空旧 session 的 seq/inbound dedup 状态

这里最重要的不是“能恢复”，而是：

**恢复策略严格区分 identity continuity 和 execution continuity。**

能保住 session identity 就优先保住；实在不行才重建。

这是工业远程会话系统里的高级设计。

---

## 8.8 初始历史 flush 与 `FlushGate`

远程系统中常见的一致性问题是：

“会话刚建立时，需要把本地已有历史同步过去；同时用户又可能继续输入新消息。如何避免远端看到历史和新消息交错？”

Claude Code 的做法是 `FlushGate<T>`。

### 状态机

```text
start()
  -> enqueue() 返回 true，消息进入 pending
end()
  -> 结束 flush，返回 pending 供 drain
drop()
  -> 永久丢弃 pending
deactivate()
  -> transport 替换时保留 pending，但先关闭 active
```

### 设计价值

它保证：

1. 初始历史和实时消息不会乱序混发
2. transport replacement 时不会误丢待发消息
3. permanent close 时可显式丢弃，避免悬挂缓冲

这是一个很小但非常工业化的组件。

---

## 8.9 Dedup 设计：`BoundedUUIDSet`

`BoundedUUIDSet` 的实现是：

- Set 做 O(1) membership
- 环形数组做 FIFO eviction
- 容量固定

它同时服务两类去重：

1. **echo dedup**
   本地发出去的消息被远端回显回来

2. **re-delivery dedup**
   transport 切换、replay 或 seq 恢复失败导致服务端重新发旧消息

为什么不用普通 Set？

因为普通 Set 无上界，长会话会泄漏内存。

为什么不用“命中后删除”？

因为同一个 UUID 可能被多次回显或重放。

因此它是一个非常标准、非常合理的流式去重结构。

---

## 8.10 v1 / v2 双栈 transport

Bridge 在 transport 层支持两种不同协议栈：

### v1

- 读：Session-Ingress WebSocket
- 写：Session-Ingress POST
- transport 实现：`HybridTransport`

### v2

- 读：SSETransport
- 写：CCRClient `/worker/*`
- transport 实现：`createV2ReplTransport()`

这不是简单协议切换，而是整个连接模型的升级。

### 关键差异 1：认证

v1：

- 可直接使用 OAuth

v2：

- 必须使用 worker JWT
- 因为 `/worker/*` 需要 session_id claim 和 worker role

### 关键差异 2：恢复游标

v2 使用 SSE sequence number。

`getLastSequenceNum()` 使 transport 可以在切换后继续从上次高水位恢复，而不是从 0 replay 全历史。

### 关键差异 3：epoch 语义

v2 引入 `worker_epoch`，防止多个 worker 版本对同一 session 并发写入。

这已经很像一个分布式单消费者租约机制。

---

## 8.11 `workSecret.ts`：兼容层与 worker 注册

这个文件有三块关键逻辑：

### A. `decodeWorkSecret()`

安全地解 base64url 并校验：

- version
- `session_ingress_token`
- `api_base_url`

### B. `sameSessionId()`

它解决的是 v1 `session_*` 与 infra `cse_*` 的兼容层差异。

做法不是全量解析 tagged ID，而是比较最后一个 `_` 后的 body。

这是一个务实的兼容策略：

- 简单
- 高效
- 对 staging/tag 变体鲁棒

### C. `registerWorker()`

v2 transport 在必要时要先向 `/worker/register` 换取 `worker_epoch`。

这说明 worker 身份不是单纯 token 决定的，还要显式注册并取得 epoch。

---

## 8.12 Bridge 控制流：`bridgeMessaging.ts`

`bridgeMessaging.ts` 是 bridge 体系里的协议中枢。

它包含三类关键逻辑：

### A. ingress 消息解析与路由

`handleIngressMessage()`：

1. parse + normalize control message keys
2. 优先识别 `control_response`
3. 再识别 `control_request`
4. 最后处理 `SDKMessage`
5. 用 `recentPostedUUIDs` / `recentInboundUUIDs` 做双重去重

这个优先级非常重要，因为控制消息不是 SDKMessage 子类型，必须先路由。

### B. server-side control request handling

`handleServerControlRequest()` 处理：

- `initialize`
- `set_model`
- `set_max_thinking_tokens`
- `set_permission_mode`
- `interrupt`

它会立刻返回 `control_response`，否则服务端会在约 10-14 秒后 kill 连接。

这说明控制平面是一个强时序协议，不是 eventual consistency。

### C. `makeResultMessage()`

Bridge teardown 前会显式发一个 result message 给服务端触发归档。

这意味着 session archival 并不单靠 TCP close 推断，而是显式依赖协议消息。

这是正确的设计，因为 close 只能表示 transport 断开，不能安全表示“业务会话已完成”。

---

## 8.13 权限桥：本地与远端权限 UI 的竞态控制

在 bridge 模式下，权限系统变得最复杂。

原因是可能有多个裁决源：

1. 本地 REPL 交互
2. 远端 web/app 控制端
3. SDK host / structured IO

### 关键接口

`BridgePermissionCallbacks`：

```ts
type BridgePermissionCallbacks = {
  sendRequest(...)
  sendResponse(requestId, response)
  cancelRequest(requestId)
  onResponse(requestId, handler): () => void
}
```

这是一层非常漂亮的抽象。

它把权限桥定义成：

- 发送请求
- 发送响应
- 取消请求
- 订阅响应

也就是说，权限桥被当成一个可竞态的异步消息总线，而不是一次性函数调用。

### `useReplBridge.tsx` 中的实现

Hook 初始化后会把 `handle.sendControlRequest/Response/CancelRequest` 封装成 `replBridgePermissionCallbacks`，写进 AppState。

于是交互式权限组件就能：

- 将 `can_use_tool` 推到远端
- 等待任一侧响应
- 在本地或远端已裁决时取消另一侧悬挂请求

### `structuredIO.ts` 的配合

`StructuredIO.injectControlResponse()` 会在 bridge 响应获胜时：

1. resolve pending request
2. 发 `control_cancel_request`

目的是中止另一个仍悬挂中的 `canUseTool` callback，避免重复处理和 orphan handler。

这部分是整套架构里非常值得关注的工业设计：

**权限不是同步阻塞点，而是一个支持多裁决源竞争与取消的异步事务。**

---

## 8.14 `bridgePointer.ts`：跨崩溃恢复的会话锚点

`bridgePointer.ts` 实现了一个很小但非常关键的 crash recovery 子系统。

### 核心思想

- session 创建后立刻写 pointer
- 长会话期间周期性刷新 mtime
- clean shutdown 清理
- crash / kill -9 不清理
- 下次启动可据此 resume

### 为什么用文件 mtime 而不是嵌入 timestamp

注释里已经说得很清楚：

- 定期重写同内容即可刷新“活跃度”
- 与后端滚动 TTL 语义一致

### 为什么还要 worktree-aware 读取

因为 REPL bridge 的 cwd 可能被 worktree 工具改写。

如果只在当前 shell cwd 查 pointer，会丢失真实活跃 session。

因此它实现了：

- 当前 dir fast path
- git worktree fanout search
- 选 freshest pointer

这是非常有实际场景针对性的工程设计。

---

## 9. Env-less Bridge：从“环境调度层”收缩到“直接桥接层”

`src/bridge/remoteBridgeCore.ts` 展示了 bridge 架构的第二阶段演进。

它的核心思想不是“换个 transport”，而是：

**把 env-based bridge 里的 Environments API 调度层直接拿掉。**

## 9.1 env-less 的流程

```text
POST /v1/code/sessions
  -> session.id

POST /v1/code/sessions/{id}/bridge
  -> { worker_jwt, expires_in, api_base_url, worker_epoch }

createV2ReplTransport(...)
  -> SSE + CCRClient

token refresh scheduler
  -> 定期重新 POST /bridge
  -> 刷新 JWT 和 epoch
```

不再需要：

- environment register
- poll work
- ack work
- heartbeatWork
- deregister environment

这说明服务端能力升级后，REPL bridge 可以直接做 OAuth -> worker_jwt 交换，不再依赖调度层中转。

---

## 9.2 为什么 env-less 值得单独存在

它的价值至少有四点：

1. **降低链路复杂度**
   少了一整套 environment/work 生命周期。

2. **降低延迟**
   无需 poll work 再建立 transport。

3. **减少失败面**
   少了 environment deleted / work lease / ack 失败等问题。

4. **更适合 REPL 单会话场景**
   REPL 本身就是交互前台，不一定需要 worker dispatch 模型。

但 env-less 不是完全替代 env-based。

代码明确写了：

- 它只用于 REPL path
- daemon / print 仍走 env-based

原因很现实：

- daemon 更像多 worker / background infra
- env-based 的 dispatch 能力在那里仍有价值

---

## 9.3 env-less 的关键工程约束

### A. JWT 不写入全局环境变量

`createV2ReplTransport()` 支持 `getAuthToken()` per-instance closure。

注释明确指出：如果把 worker JWT 写进 `process.env.CLAUDE_CODE_SESSION_ACCESS_TOKEN`，会被 MCP client 等模块无门槛读取，可能外泄给用户配置的 ws/http MCP server。

这是非常强的安全边界意识。

### B. token refresh scheduler

env-less 没有 `work re-dispatch` 帮它自然刷新 JWT，所以必须主动调度刷新。

这意味着：

- env-based：JWT 刷新更偏被动，依赖 re-dispatch
- env-less：JWT 刷新更偏主动，依赖 scheduler

这两种策略各自匹配不同部署拓扑。

### C. independent version floor

`envLessBridgeConfig.ts` 单独提供 `min_version`，与 v1 bridge floor 分离。

这使得：

- v2 出现 bug 时可单独强制升级
- 不会连带阻断仍可工作的 v1 path

这是 rollout 工程上的细致设计。

---

## 10. 设计约束总表

从代码中可以总结出这套架构显式遵守了多条设计约束。

## 10.1 安全约束

1. bridge 仅对 claude.ai subscriber 开放，依赖 OAuth。
2. 组织策略 `allow_remote_control` / `allow_remote_sessions` 必须通过。
3. 所有 server-provided IDs 必须过 allowlist 校验。
4. worker JWT 不应泄漏到全局 env，防止跨模块意外传播。
5. 权限请求必须可取消，避免双裁决。

## 10.2 一致性约束

1. 初始历史 flush 与实时消息不能交错。
2. 重连后不能把旧会话的 seq / dedup 状态误用于新会话。
3. echo dedup 与 inbound replay dedup 必须分开建模。
4. transport close 不等于 session end，必须显式 result/archive。

## 10.3 包体与模块边界约束

1. bridge core 不能硬依赖 REPL 全量模块树。
2. OAuth refresh、message mapper、title 生成等能力尽量通过参数注入。
3. 动态 import 用于隔离非核心重依赖。

## 10.4 用户体验约束

1. 尽量保持同一 session identity。
2. viewerOnly 模式不得误发送 interrupt。
3. bridge 失败时不能让 REPL 崩溃。
4. 休眠/恢复场景下要尽量自动修复，不要求用户手动重连。

---

## 11. 关键工业级工程设计与创新点

下面列出我认为最值得注意的设计亮点。

## 11.1 双平面协议：内容流与控制流分离

这是整套远程架构最根本的正确性基础。

它避免了：

- 权限审批与内容渲染混淆
- interrupt 被普通消息延迟
- 模式切换缺乏 request-response 保障

在 Agent 系统里，这种双平面协议非常值得借鉴。

---

## 11.2 远程权限桥的“竞态可取消”设计

很多系统会把权限审批实现为简单的 blocking callback。

Claude Code 没这么做。

它把权限审批抽象为：

- 可发送
- 可响应
- 可取消
- 可订阅

这使得本地 REPL、远端 web、SDK host 可以并发参与，而系统仍能保证只有一个最终裁决生效。

这是明显的工业级异步控制设计。

---

## 11.3 分层恢复策略

恢复策略被拆成：

1. transport recover
2. same session recover
3. same environment recover
4. fresh session fallback

这比“统一重连”复杂，但能显著提升会话连续性。

尤其对于 Agent 会话，这种 continuity 非常重要，因为它承载：

- 用户上下文
- UI 链接
- 历史消息一致性

---

## 11.4 `FlushGate + BoundedUUIDSet + lastSequenceNum` 组合拳

这三个小组件共同解决了分布式消息同步中最常见但最烦的问题：

- 顺序
- 去重
- 重放恢复

它们各自职责清晰：

- `FlushGate` 解决顺序
- `BoundedUUIDSet` 解决去重
- `lastSequenceNum` 解决重连后的增量恢复

这是一套非常典型、可迁移到其他 Agent 平台的工程模式。

---

## 11.5 env-based 到 env-less 的演进策略

Claude Code 没有粗暴替换 bridge，而是并存：

- env-based：保守、调度能力强、适合 daemon
- env-less：简化链路、适合 REPL、依赖新服务端能力

并通过：

- 独立 feature gate
- 独立版本门槛
- 独立配置 schema

渐进式 rollout。

这是非常成熟的演进设计。

---

## 11.6 bundle hygiene 与依赖注入

桥接相关核心模块中大量出现的模式：

- “这个依赖不要直接 import，改为参数传入”
- “这个模块用动态 import，否则会把整个 React tree 拖进 bundle”

这说明作者非常清楚：

**在 agent/CLI 系统中，运行时架构设计不仅是控制流设计，也是包体边界设计。**

这点通常只在比较成熟的工业代码里会被如此系统地贯彻。

---

## 12. 给 AI 应用与算法专家的抽象理解

如果从更高层抽象，Claude Code 的这套远程与桥接架构，本质上实现了下面这个目标：

**把“单机交互式 agent”提升为“位置无关的 agent 会话系统”。**

这里的位置无关，具体体现在：

1. 用户不必和执行环境在同一位置。
2. 权限审批不必和工具执行在同一位置。
3. 控制器不必和 session state store 在同一位置。
4. 会话 identity 可以独立于当前 transport 生命周期存在。

这意味着 Claude Code 已经从“本地 CLI 工具”演化成了：

**一个可在本地终端、远端 session、桥接 worker、自建 server 之间自由部署的 Agent runtime。**

对于 AI 应用平台而言，这种架构有很高的参考价值，尤其适合以下场景：

- 多终端协作式 coding agent
- 本地执行、云端控制的高权限工具代理
- 有审批要求的混合人机工作流
- 需要跨断线、跨设备延续的长生命周期 agent session

---

## 13. 建议的阅读顺序

如果你准备继续深挖源码，建议按下面顺序：

1. 先读 `main.tsx` 中 `--remote`、`assistant`、`connect/open`、`remote-control` 的入口分流。
2. 再读 `RemoteSessionManager.ts` + `useRemoteSession.ts`，理解最简单的“远端执行，本地显示”闭环。
3. 然后读 `types.ts` + `bridgeApi.ts`，先建立 environment/work 调度模型。
4. 再读 `replBridge.ts`，重点看：
   - register
   - createSession
   - poll loop
   - onWorkReceived
   - reconnect
   - teardown
5. 然后读 `bridgeMessaging.ts`、`flushGate.ts`、`workSecret.ts`，把一致性细节读透。
6. 最后读 `remoteBridgeCore.ts` 和 `envLessBridgeConfig.ts`，理解这套系统是如何演化到 v2/env-less 的。

---

## 14. 结论

Claude Code 的远程与桥接架构，真正难的不是“远程连接”本身，而是它对以下问题的系统性回答：

1. UI 与执行分离后，如何保持用户体验连续。
2. 控制动作与内容消息分离后，如何保证强时序控制。
3. 多位置权限裁决并存时，如何避免竞态和重复处理。
4. transport、session、environment 失效时，如何按层恢复而不是一刀切重建。
5. 协议、包体、权限、安全、部署拓扑之间，如何同时做出工程上可落地的取舍。

从这个角度看，这套代码不是一个“带远程功能的 CLI”，而是一套：

**具备分布式会话控制、可恢复消息同步、远程权限仲裁与多拓扑部署能力的 Agent 运行时架构。**

这也是它最值得被深度理解的地方。
