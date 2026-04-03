# Claude-code-open 远程与桥接架构深度解读

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 中的“远程与桥接架构”。

这里最容易混淆的，不是“项目支持远程”这件事，而是下面几个更具体的问题：

1. `--remote`、`assistant`、`remote-control`、`connect/open` 到底有什么区别？
2. 为什么代码里同时存在 `src/remote/*`、`src/bridge/*`、`src/server/*` 三套实现？
3. 什么是 Remote Session，什么是 Bridge，什么又是 Direct Connect？
4. 哪些场景下，模型运行在远端；哪些场景下，只是 UI 远程连接；哪些场景下，是本地环境暴露给远端控制？
5. 权限请求、用户消息、结果消息、会话恢复、断线重连，分别由哪一层处理？
6. 为什么桥接层里会出现 `environment`、`work item`、`session ingress token`、`worker epoch` 这些看起来很“基础设施”的概念？

本文基于以下主干代码阅读整理：

- `src/main.tsx`
- `src/remote/RemoteSessionManager.ts`
- `src/remote/SessionsWebSocket.ts`
- `src/remote/sdkMessageAdapter.ts`
- `src/remote/remotePermissionBridge.ts`
- `src/hooks/useRemoteSession.ts`
- `src/bridge/replBridge.ts`
- `src/bridge/initReplBridge.ts`
- `src/bridge/bridgeMessaging.ts`
- `src/bridge/replBridgeTransport.ts`
- `src/bridge/types.ts`
- `src/bridge/createSession.ts`
- `src/bridge/workSecret.ts`
- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`

---

## 2. 先给结论

Claude Code 的“远程能力”并不是一套机制，而是三套职责不同的机制：

```text
A. Remote Session
本地 CLI 连接一个已经在云端运行的 Claude Code 会话

B. Bridge / Remote Control
把“你的本地开发环境”注册成一个可被 claude.ai 远程驱动的执行端

C. Direct Connect
本地 CLI 直接连接某个自建 Claude Code server
```

如果只记一句话：

**`remote/` 解决“我去连接远端会话”，`bridge/` 解决“我把本地环境暴露给远端控制”，`server/` 解决“我去连接另一台 Claude Code 服务端”。**

三者都复用了同一类消息协议思路：

- 统一用 `SDKMessage` 表示对话流中的事件
- 用 `control_request / control_response` 表示控制面交互
- 用本地 Hook 把远端协议事件翻译成 REPL 内部 `Message`
- 用本地 UI 继续承担展示、权限审批、输入采集

所以可以把它理解成：

**Claude Code 把“终端 UI”和“Agent 执行环境”解耦了。**

---

## 3. 三种模式分别解决什么问题

## 3.1 Remote Session：本地看远端，Agent 在远端跑

典型入口：

- `claude --remote "task"`
- `claude --teleport <session-id>`
- `claude assistant [sessionId]`

核心特点：

- 真正执行对话与工具调用的是远端 CCR 会话
- 本地 CLI 主要负责：
  - 展示消息
  - 发送用户输入
  - 展示并处理权限审批
  - 断线重连
- 因为执行环境不在本地，所以本地通常不加载完整工具集，只做协议适配

对应代码：

- `src/remote/*`
- `src/hooks/useRemoteSession.ts`

---

## 3.2 Bridge / Remote Control：本地机器变成远端可调度执行端

典型入口：

- `claude remote-control`
- REPL 内自动/手动开启 `/remote-control`
- `useReplBridge` 自动拉起桥接

核心特点：

- Agent 的实际执行仍发生在“这台本地机器”的 Claude Code REPL
- 但远端 claude.ai/code 可以给这台机器派发会话与控制请求
- 本地机器需要先向服务端注册一个 `environment`
- 服务端再把某个 `session` 的 `work item` 分发给这个 environment
- 本地桥接层拿到 work 后建立 ingress 通道，把消息双向同步

这本质上是：

**把本地 Claude Code 进程包装成一个可被服务端调度的 worker。**

对应代码：

- `src/bridge/*`
- `src/hooks/useReplBridge.tsx`

---

## 3.3 Direct Connect：本地 CLI 连另一台 Claude Code server

典型入口：

- `claude connect/open <cc-url>`

核心特点：

- 对端不是 Anthropic 的 Remote Session 服务，而是某个自建 server
- 本地先 `POST /sessions` 创建会话
- 再连 server 返回的 `ws_url`
- 双方直接走流式 JSON / SDKMessage 协议

对应代码：

- `src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`

---

## 4. 总体关系图

```text
模式 1：Remote Session
本地 REPL
  -> RemoteSessionManager
  -> SessionsWebSocket + HTTP POST
  -> 远端 CCR Session

模式 2：Bridge
本地 REPL
  -> initReplBridge / initBridgeCore
  -> 注册 environment
  -> poll work
  -> 建 ingress transport
  -> claude.ai/code 从远端控制本地环境

模式 3：Direct Connect
本地 REPL
  -> createDirectConnectSession
  -> DirectConnectSessionManager
  -> 自建 server
```

其中最关键的架构思想是：

1. UI 永远尽量保留在本地终端。
2. 执行面可以在远端，也可以在本地，只要协议统一即可。
3. 控制面与消息面分离。
4. 断线恢复不是“重新开会话”，而是尽量做“恢复同一会话”。

---

## 5. 入口与模式分流

`src/main.tsx` 是模式路由中心。

它至少把远程相关路径分成三类：

### A. Direct Connect

`main.tsx` 在 `claude connect/open <cc-url>` 路径里调用：

- `createDirectConnectSession()`
- 启动 REPL
- 把 `directConnectConfig` 传入 REPL

这说明 Direct Connect 在产品层是“一种特殊 REPL 模式”。

### B. Assistant / Viewer 模式

`main.tsx` 在 `assistant [sessionId]` 路径里：

- 准备 OAuth
- `createRemoteSessionConfig(..., viewerOnly = true)`
- 启动 REPL 并注入 `remoteSessionConfig`

这里的 `viewerOnly` 非常关键，表示：

- 本地只是附着查看
- 本地不应该打断远端执行
- 标题更新、超时检测等行为要收敛

### C. Remote Session 模式

`main.tsx` 在 `--remote` 路径里：

- 先创建远端 session
- 再进入本地 TUI
- REPL 后续通过 `remoteSessionConfig` 驱动远端会话

这说明 `--remote` 不是“打印一个链接就结束”，而是可进入完整本地终端交互。

### D. Bridge 模式

`main.tsx` 对 `remote-control` 命令只做帮助注册，真实执行走 `bridgeMain.ts` / `initReplBridge.ts` 这一套快速路径。

这说明桥接是一个更底层、更偏“运行时基础设施”的模式，不能简单理解成普通命令。

---

## 6. Remote Session 架构

## 6.1 核心分工

Remote Session 主要由四层组成：

```text
RemoteSessionManager
  会话级编排

SessionsWebSocket
  订阅、心跳、重连

sdkMessageAdapter
  SDKMessage -> REPL Message

useRemoteSession
  React/UI 集成、权限队列、加载态、断线处理
```

---

## 6.2 `RemoteSessionManager`：会话控制器

`src/remote/RemoteSessionManager.ts` 的职责很清晰：

1. 创建并持有 `SessionsWebSocket`
2. 收到消息后区分：
   - 普通 `SDKMessage`
   - `control_request`
   - `control_response`
   - `control_cancel_request`
3. 为权限请求维护 `pendingPermissionRequests`
4. 发送用户消息
5. 回传权限审批结果
6. 发起 interrupt

这里最重要的设计点是：

**Remote Session 并没有把权限判断放在本地执行，而是把“远端发起的权限请求”映射成本地 UI 的审批行为。**

也就是：

```text
远端 Agent 发现需要权限
  -> 发送 control_request(can_use_tool)
  -> 本地 UI 弹权限确认
  -> 用户 allow/deny
  -> 本地回 control_response
  -> 远端继续执行
```

这是一种很典型的“执行在远端，授权在人侧终端”的架构。

---

## 6.3 `SessionsWebSocket`：订阅与重连器

`src/remote/SessionsWebSocket.ts` 专门处理会话订阅。

关键职责：

1. 拼装 `/v1/sessions/ws/{sessionId}/subscribe`
2. 每次连接前取最新 access token
3. 维护连接状态 `connecting / connected / closed`
4. 定时发 ping 保活
5. 针对关闭码决定是否重连

关键算法/策略有三条：

### A. 限次重连

- 普通断线最多重连 5 次
- 每次延迟 2 秒

这是典型的有限恢复策略，避免无限重连。

### B. 特殊处理 `4001 session not found`

代码里明确写了：会话 compact 等过程中，服务端可能短暂认为 session 不存在。

因此它不是立刻失败，而是给一个有限重试窗口：

- 最多 3 次
- 逐次递增延迟

这属于“识别瞬时错误，避免误判成永久失败”。

### C. 永久错误立即停止

例如：

- `4003 unauthorized`

这种错误不做重连，直接 `onClose`。

这说明它在网络层就区分了：

- 暂时性故障
- 业务级永久拒绝

---

## 6.4 `sdkMessageAdapter`：协议翻译层

`src/remote/sdkMessageAdapter.ts` 的本质，是把远端 SDK 协议翻译成本地 REPL 能显示的内部消息。

例如：

- `assistant` -> `AssistantMessage`
- `stream_event` -> `StreamEvent`
- `result` -> `SystemMessage`
- `tool_progress` -> 信息型系统消息
- `compact_boundary` -> 会话压缩边界消息

这个文件的重要性在于：

**远端协议消息不直接等于本地 UI 消息。**

两者之间需要适配层，原因包括：

1. UI 结构不同
2. 有些消息要显示，有些只用于状态控制
3. 同一协议消息在不同模式下处理不同

典型例子是用户消息：

- 普通 Remote Session 下，用户输入本地已经知道了，WS 回声通常忽略
- `viewerOnly` 或历史回放场景下，用户文本和 `tool_result` 又必须转换出来展示

这就是 `convertToolResults`、`convertUserTextMessages` 两个开关存在的原因。

---

## 6.5 `useRemoteSession`：远程会话的 UI 运行时

`src/hooks/useRemoteSession.ts` 是真正把远端会话接进 REPL 的地方。

它主要做 6 件事：

1. 创建 `RemoteSessionManager`
2. 处理消息流并更新 `messages`
3. 处理权限请求并构造本地 `ToolUseConfirm`
4. 做 echo 去重
5. 管理 loading / compacting / reconnecting 等状态
6. 在需要时发送 interrupt

其中有几个很值得初学者注意的设计点。

### A. Echo 去重

本地发送的用户消息，服务端可能会通过 WS 再回一遍。

为避免 UI 显示两次，代码维护了 `BoundedUUIDSet` 风格的 `sentUUIDsRef`。

算法本质：

- 发送前记住 uuid
- 收到同 uuid 的 user message 时丢弃
- 不在第一次命中后删除，而是做有界保留

原因是：

- 同一个消息可能被 server 和 worker 各回显一次
- “命中一次就删”会让第二次回显漏网

### B. 远端权限请求本地化

远端发来的 `can_use_tool` 请求会被包装成：

- synthetic assistant message
- 本地 Tool 对象或 stub Tool
- REPL 的权限确认队列项

这样 UI 层完全可以复用本地工具审批界面。

### C. 未知工具兜底

远端可能有本地没有加载的工具，例如某个远端 MCP 工具。

这时 `createToolStub()` 会创建一个最小 Tool 占位对象，使权限 UI 仍可正常展示。

这是一个很实用的兼容设计：

**权限系统依赖的是“工具元信息可展示”，而不是“工具可在本地执行”。**

---

## 7. Bridge / Remote Control 架构

Bridge 是整套远程架构里最复杂的一部分。

如果 Remote Session 是“本地连到远端 Agent”，那 Bridge 就是：

**把本地 REPL 变成远端服务可调度的 Agent 执行端。**

---

## 7.1 先理解几个核心概念

`src/bridge/types.ts` 里定义了桥接层最重要的领域模型。

### A. `BridgeConfig`

描述本地环境如何注册为远端可见环境：

- `dir`
- `machineName`
- `branch`
- `gitRepoUrl`
- `workerType`
- `environmentId`
- `reuseEnvironmentId`
- `apiBaseUrl`
- `sessionIngressUrl`

其中最重要的是：

- `environmentId`：环境实例身份
- `workerType`：告诉服务端这是什么来源的 worker
- `reuseEnvironmentId`：重连时尽量复用原环境

### B. `WorkResponse`

服务端派发给 bridge 的工作单元。

包含：

- `environment_id`
- `data.type`
- `data.id`
- `secret`

其中 `data.id` 在 session 类型 work 下就是会话 ID。

### C. `WorkSecret`

这是桥接层的关键数据结构。

里面至少有：

- `session_ingress_token`
- `api_base_url`
- `use_code_sessions`

可以把它理解成：

**服务端给当前 work 的一次性执行凭据和传输配置包。**

### D. `SessionHandle`

描述本地实际跑起来的一个会话执行句柄，包括：

- 生命周期 `done`
- kill / forceKill
- 当前活动摘要
- access token
- stderr ring buffer

它是桥接层把“远程 session”映射到“本地进程会话”的抽象。

---

## 7.2 `initReplBridge` 与 `initBridgeCore`

桥接初始化分成两层：

### A. `initReplBridge`

REPL 专属包装层，负责读取本地上下文：

- 当前 cwd
- session id
- git 信息
- OAuth
- 标题生成
- feature gate / policy gate

### B. `initBridgeCore`

真正的桥接内核，不依赖 REPL 全量模块，只接收参数。

这种拆分很重要，因为桥接逻辑还要给 daemon / SDK 等非 REPL 场景复用。

也就是说：

**`initReplBridge` 负责“从本地应用取上下文”，`initBridgeCore` 负责“按协议完成桥接”。**

---

## 7.3 Bridge 主流程

桥接主流程可以概括成：

```text
本地 REPL 开启 bridge
  -> registerBridgeEnvironment()
  -> createBridgeSession()
  -> 启动 work poll loop
  -> 服务端派发 work
  -> decodeWorkSecret()
  -> acknowledgeWork()
  -> 建立 transport
  -> 本地 REPL 与远端 session 双向同步
  -> 断线则尽量恢复同一 environment / session
  -> teardown 时 stopWork + archiveSession + deregisterEnvironment
```

这里的难点不在“建立连接”，而在“服务端调度 + work lease + transport 切换 + 会话恢复”。

---

## 7.4 `replBridge.ts`：桥接核心状态机

`src/bridge/replBridge.ts` 是整套桥接架构的核心。

它做了几件关键事情。

### A. 注册 environment

启动时先调用 `registerBridgeEnvironment()`。

可以理解为：

“我这台本地机器已经上线，服务端可以把工作派给我了。”

### B. 创建 session

然后调用 `createSession()`，也就是 `createBridgeSession()`。

这个 session 是远端可见的会话实体，会出现在 claude.ai 侧。

### C. 启动 poll loop

桥接不会傻等 WebSocket 被动推送，而是主动轮询是否有新 work。

这就是 `startWorkPollLoop()` 的职责。

### D. 收到 work 后建立 transport

收到 `work` 以后：

1. 解码 `secret`
2. 校验 session 是否属于当前 bridge
3. 保存当前 `workId` / `ingressToken`
4. 根据 `use_code_sessions` 选择 v1 或 v2 transport

### E. 维护长生命周期恢复

桥接非常强调“尽量恢复同一会话”，所以代码里大量逻辑都在做：

- env 丢了，尝试重注册同一 env
- session 丢了，尝试 reconnect 同一 session
- 实在不行，再创建新 session

这比“断线就重开”复杂得多，但用户体验会好很多。

---

## 7.5 Work Poll Loop：桥接的调度心脏

`startWorkPollLoop()` 是理解 Bridge 的关键。

它不是单纯轮询，而是一个带有恢复、心跳、容量控制的状态循环。

### 它做什么

1. 调 `pollForWork()`
2. 没 work 时按不同节奏 sleep
3. 有 work 时 decode secret、ack、回调 `onWorkReceived`
4. 容量满时切换到 heartbeat 模式
5. 遇到环境丢失时尝试重建
6. 遇到 fatal error 时终止整个桥接

### 为什么需要“容量模式”

REPL bridge 是单 session 的。

只要 transport 存在，就说明当前桥已经在服务一个会话，不能再接新的活。

所以代码里有 `isAtCapacity()`。

容量满时并不是完全停掉轮询，而是：

- 低频轮询 / 心跳保活
- 一旦 transport 丢失，立刻唤醒轮询，抢回 work

这是一个典型的“忙时低频保活，空闲时快速探测”的调度算法。

### 为什么还要 heartbeat

因为 work item 有 lease TTL。

如果桥接端长时间不 heartbeat，服务端会认为 worker 失活，把 work 重新派发。

因此 heartbeat 的作用是：

- 延长 work lease
- 证明当前 transport 仍然活着

### 环境丢失恢复策略

代码中对 `404 environment deleted` 的处理很讲究：

1. 优先尝试重新注册原 environment 并 reconnect 原 session
2. 失败后才 archive 旧 session，创建新 session

这就是 bridge 最核心的恢复策略。

---

## 7.6 v1 / v2 Transport 双栈

`src/bridge/replBridgeTransport.ts` 定义了统一的 `ReplBridgeTransport` 抽象。

它背后有两种实现：

### A. v1

- 读：`HybridTransport` WebSocket
- 写：Session-Ingress POST

### B. v2

- 读：`SSETransport`
- 写：`CCRClient` 到 `/worker/*`

Bridge 代码通过统一接口屏蔽了二者差异，例如：

- `write`
- `writeBatch`
- `setOnData`
- `setOnClose`
- `getLastSequenceNum`

这是一种标准的“传输层多态”设计。

初学者需要特别注意：

**v1/v2 的差异不只是底层协议不同，连认证方式也不同。**

- v1 可用 OAuth
- v2 必须使用 `session_ingress_token` 这种 JWT

原因在 `workSecret.ts` 和 `replBridgeTransport.ts` 里写得很清楚：

- v2 要校验 session_id claim
- OAuth token 不带这个 claim

---

## 7.7 `bridgeMessaging.ts`：桥接消息处理公共层

这个文件很像 Remote Session 里的 `sdkMessageAdapter`，但定位不同。

它解决三类问题：

### A. ingress 消息路由

`handleIngressMessage()` 会区分：

- `control_response`
- `control_request`
- 普通 `SDKMessage`

然后决定：

- 回调权限结果
- 处理服务端控制请求
- 把真正的 inbound user message 送回 REPL

### B. echo / 重放去重

桥接层使用 `BoundedUUIDSet` 做两类去重：

- `recentPostedUUIDs`：过滤自己刚发出去又被回显的消息
- `recentInboundUUIDs`：过滤服务端因重连或 replay 重新送来的入站消息

这个结构是一个固定容量的环形缓冲 + Set：

- 查重 O(1)
- 内存上界固定
- 老数据自动淘汰

它是本文里最典型的“小而关键的工程算法”。

### C. 服务端控制请求处理

`handleServerControlRequest()` 负责处理：

- `initialize`
- `set_model`
- `set_max_thinking_tokens`
- `set_permission_mode`
- `interrupt`

这个设计说明：

**桥接不仅传消息，还允许远端对本地 REPL 的执行参数做控制。**

---

## 7.8 Bridge 里的几个关键算法

### A. 同会话判断：`sameSessionId()`

`workSecret.ts` 里的 `sameSessionId()` 会忽略 tag 前缀，只比较底层 UUID 部分。

原因是：

- v1 API 可能给出 `session_*`
- infra/work queue 里可能是 `cse_*`

本质是同一个 session，只是不同兼容层的 ID 外壳不同。

这是一个很典型的“协议兼容层 ID 归一化”算法。

### B. 初始历史 flush + flush gate

桥接建立后，可能要把当前 REPL 历史同步到远端。

为了避免：

- 历史消息还没刷完
- 新消息又插队发到远端

代码设计了 `FlushGate`：

- flush 开始时先 gate
- 新消息先排队
- flush 完成后统一 drain

这是保证消息顺序一致性的关键机制。

### C. SSE sequence number 继承

v2 transport 会记录 `lastSequenceNum`。

transport 切换或重连时，把这个高水位传给新 transport，避免服务端从 0 重放整段历史。

这本质上是流处理里的 cursor / offset 恢复。

### D. 分层恢复策略

桥接并不是遇错就 teardown，而是按层级恢复：

1. transport 断了 -> 先重连 transport
2. transport 恢复不了 -> 尝试 reconnect environment + session
3. env/session 也恢复不了 -> 再 teardown / 新建 session

这就是典型的分层容错设计。

---

## 8. Direct Connect 架构

Direct Connect 比 Bridge 简单很多。

## 8.1 创建阶段

`createDirectConnectSession()` 做三件事：

1. `POST ${serverUrl}/sessions`
2. 校验返回结构 `session_id / ws_url / work_dir`
3. 生成 `DirectConnectConfig`

所以 Direct Connect 没有 environment/work 这一层，直接就是：

**创建 session -> 连 ws。**

---

## 8.2 运行阶段

`DirectConnectSessionManager` 负责：

1. 建 WebSocket
2. 解析每一行 JSON
3. 识别 `control_request`
4. 转发普通 `SDKMessage`
5. 发送用户消息、权限响应、interrupt

它和 `RemoteSessionManager` 很像，但少了：

- Sessions API 那套订阅语义
- poll work
- environment 管理
- 复杂重连策略

所以可以把 Direct Connect 看成：

**一个更轻量、更直接的远程协议适配器。**

---

## 9. 关键数据结构总表

### A. `RemoteSessionConfig`

用途：定义如何连接一个远端 CCR session。

关键字段：

- `sessionId`
- `getAccessToken`
- `orgUuid`
- `hasInitialPrompt`
- `viewerOnly`

### B. `RemotePermissionResponse`

用途：把本地审批结果回传给远端。

两种结果：

- `allow + updatedInput`
- `deny + message`

### C. `BridgeConfig`

用途：注册 bridge environment。

### D. `WorkResponse`

用途：服务端派发给 bridge 的 work 单。

### E. `WorkSecret`

用途：承载当前 work 的 ingress token、API base URL、v2 开关等。

### F. `ReplBridgeTransport`

用途：统一 v1/v2 传输抽象。

### G. `BoundedUUIDSet`

用途：固定内存上界的 UUID 去重结构。

---

## 10. 给初学者的理解路径

如果你第一次读这部分代码，建议按下面顺序：

1. 先读 `src/main.tsx` 里 Remote / Assistant / Connect / Remote Control 的入口分流，先建立“有三种模式”的全局图。
2. 再读 `src/remote/RemoteSessionManager.ts` 和 `src/hooks/useRemoteSession.ts`，理解“本地 UI 连接远端会话”最小闭环。
3. 然后读 `src/bridge/types.ts`，先把 `environment / work / secret / session` 这些名词搞清楚。
4. 再读 `src/bridge/replBridge.ts`，重点盯住：注册环境、创建 session、poll work、建 transport、teardown。
5. 最后读 `src/bridge/bridgeMessaging.ts` 和 `src/bridge/replBridgeTransport.ts`，理解消息路由、去重和 v1/v2 双栈抽象。

如果读到某处开始混乱，可以反复问自己一个问题：

**现在这段代码是在解决“谁在执行”、还是“谁在展示”、还是“谁在授权”、还是“谁在恢复连接”？**

只要把这四件事分开，整套远程与桥接架构就会清晰很多。

---

## 11. 最后总结

Claude Code 的远程与桥接设计，体现了一个很成熟的工程思路：

1. 把 UI、执行、控制、授权拆成不同层。
2. 用统一消息协议贯穿不同运行位置。
3. 用桥接层把“本地开发环境”抽象成可调度远端 worker。
4. 用去重、flush gate、sequence number、heartbeat、reconnect 等机制，解决长连接系统最麻烦的一批一致性问题。

因此，这套架构的重点从来不是“能不能远程”，而是：

**当执行位置和展示位置分离之后，系统如何仍然保证消息顺序、权限交互、会话连续性和错误恢复。**

这正是 `remote/`、`bridge/`、`server/` 三套代码共同回答的问题。
