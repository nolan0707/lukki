# Claude-code-open 模块化架构深度解读

## 1. 文档目标

本文基于以下已有文档继续向上抽象：

- [0001-project-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0001-project-arch.md)
- [0002-conversation-execution-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0002-conversation-execution-arch.md)
- [0002-conversation-execution-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0002-conversation-execution-deep-arch.md)
- [0003-command-tool-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0003-command-tool-deep-arch.md)
- [0004-mcp-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0004-mcp-deep-arch.md)
- [0005-plugins-skills-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0005-plugins-skills-deep-arch.md)
- [0006-permission-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0006-permission-deep-arch.md)
- [0007-remote-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0007-remote-deep-arch.md)
- [0008-observable-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0008-observable-deep-arch.md)

目标不是重复“有哪些目录”，而是从模块化设计角度回答 5 个问题：

1. 项目由哪些稳定模块组成。
2. 每个模块对外暴露什么接口与能力。
3. 模块之间如何交互，交互靠什么协议和状态结构承接。
4. 模块之间的依赖方向是什么，哪些依赖是刻意控制的。
5. 这套模块化设计中有哪些明显的工业级工程经验。

本文基于对 `vendor/Claude-code-open/src` 关键代码的继续阅读整理，重点覆盖：

- `src/entrypoints/cli.tsx`
- `src/entrypoints/init.ts`
- `src/setup.ts`
- `src/context.ts`
- `src/bootstrap/state.ts`
- `src/state/AppStateStore.ts`
- `src/state/AppState.tsx`
- `src/screens/REPL.tsx`
- `src/QueryEngine.ts`
- `src/query.ts`
- `src/query/deps.ts`
- `src/commands.ts`
- `src/types/command.ts`
- `src/tools.ts`
- `src/Tool.ts`
- `src/services/api/client.ts`
- `src/services/api/claude.ts`
- `src/services/mcp/types.ts`
- `src/utils/plugins/loadPluginCommands.ts`
- `src/remote/RemoteSessionManager.ts`

---

## 2. 一句话结论

`Claude-code-open` 并不是“一个大 REPL 文件 + 一堆工具目录”，而是一个围绕 **Agent Runtime** 构建的模块化平台。它的顶层模块关系可以概括为：

```text
入口与装配层
  entrypoints/* + init.ts + setup.ts

运行壳层
  REPL/UI + AppState + bootstrap/state

执行内核层
  QueryEngine + query.ts + services/api/*

能力注册层
  commands.ts + tools.ts + types/command.ts + Tool.ts

扩展接入层
  MCP + plugins + skills + agents

治理与保障层
  permissions + compact + sandbox + observability

远程与分布式层
  remote/* + bridge/* + server/*

基础设施层
  utils/* + services/* 中的文件系统、网络、认证、配置、持久化能力
```

真正稳定的“模块边界”不是目录边界，而是以下 4 类抽象边界：

1. `Command`：用户控制面能力接口。
2. `Tool` / `ToolUseContext`：模型执行面能力接口。
3. `Message`：跨模块共享的运行时事实载体。
4. `AppState` + `bootstrap/state`：分别承担 React 会话态和进程级基础设施态。

这 4 类抽象让项目在高复杂度下仍然保持了可扩展性。

---

## 3. 模块地图

## 3.1 顶层模块划分

从源码结构和运行时职责看，可以把项目划分为 10 个一级模块：

1. **入口与装配模块**
2. **交互与会话壳模块**
3. **对话执行内核模块**
4. **命令与工具能力模块**
5. **扩展生态模块**
6. **权限与安全治理模块**
7. **上下文与记忆治理模块**
8. **远程与分布式协同模块**
9. **观测与实验模块**
10. **基础设施模块**

其中 1-4 是主路径模块，5-9 是横向扩展与治理模块，10 是所有模块共同依赖的底座。

## 3.2 模块边界的判断标准

本文采用的“模块”不是简单按文件夹切，而是按以下标准识别：

- 是否有稳定的输入输出接口。
- 是否承担相对独立的职责闭包。
- 是否被多个上层场景复用。
- 是否有明显的状态归属。
- 是否存在清晰的依赖方向。

因此：

- `commands.ts` 和 `types/command.ts` 属于同一能力注册模块。
- `QueryEngine.ts`、`query.ts`、`services/api/claude.ts` 属于同一执行内核，但各自是不同子模块。
- `state/*` 与 `bootstrap/state.ts` 不视为一个模块，而是两个状态模块。

---

## 4. 模块总览表

| 模块 | 核心目录/文件 | 主要职责 | 典型对外接口 |
| --- | --- | --- | --- |
| 入口与装配 | `entrypoints/*`, `init.ts`, `setup.ts` | 启动、模式分发、环境装配、初始化基础设施 | `main()`, `init()`, `setup()` |
| 交互与会话壳 | `screens/REPL.tsx`, `state/*`, `context/*` | 终端 UI、会话视图状态、事件交互、输入队列 | React 组件、`useAppState()`, `useSetAppState()` |
| 对话执行内核 | `QueryEngine.ts`, `query.ts`, `query/*`, `services/api/*` | turn 执行、模型采样、工具循环、错误恢复 | `QueryEngine`, `query()`, `queryModelWithStreaming()` |
| 命令与工具能力 | `commands.ts`, `types/command.ts`, `tools.ts`, `Tool.ts`, `tools/*` | 用户入口能力、模型调用能力、协议定义 | `Command`, `Tool`, `getCommands()`, `getTools()` |
| 扩展生态 | `services/mcp/*`, `skills/*`, `utils/plugins/*`, `plugins/*` | 外部能力发现、加载、协议化、合并进运行时 | `MCPServerConnection`, plugin loaders, skill loaders |
| 权限与安全治理 | `utils/permissions/*`, `types/permissions.ts`, `utils/sandbox/*` | 审批、规则、模式切换、OS 级隔离 | `PermissionResult`, `ToolPermissionContext` |
| 上下文与记忆治理 | `context.ts`, `services/compact/*`, `memdir/*`, `services/SessionMemory/*` | 系统上下文、压缩、长时记忆、会话摘要 | `getSystemContext()`, `getUserContext()`, compact APIs |
| 远程与分布式协同 | `remote/*`, `bridge/*`, `server/*`, `tasks/*`, `utils/swarm/*` | 远程会话、桥接、子 agent、任务系统 | `RemoteSessionManager`, bridge APIs, task APIs |
| 观测与实验 | `services/analytics/*`, `utils/telemetry/*`, profiler | 埋点、1P logging、OTel、profiling、flag/config | `logEvent()`, telemetry init APIs |
| 基础设施 | `utils/*`, `services/oauth/*`, `services/api/client.ts` | 认证、配置、网络、文件、持久化、平台适配 | auth/config/fs/network helpers |

---

## 5. 入口与装配模块

## 5.1 职责

入口与装配模块负责把所有可运行能力装配成一次 CLI/SDK/Remote 进程。

对应文件：

- `src/entrypoints/cli.tsx`
- `src/entrypoints/init.ts`
- `src/setup.ts`

## 5.2 对外接口

### `entrypoints/cli.tsx`

对外暴露的是进程入口 `main()`，它承担：

- CLI 参数快路径分发。
- 动态 import 不同模式子入口。
- 避免非必要模块过早求值。

这说明它本质上是一个 **startup dispatcher**，不是业务逻辑模块。

### `entrypoints/init.ts`

核心对外接口：

```ts
init(): Promise<void>
initializeTelemetryAfterTrust(): void
```

它负责初始化：

- 配置系统
- 安全环境变量
- 代理与 mTLS
- 1P logging
- remote managed settings / policy limits 的 loading promise
- scratchpad
- graceful shutdown

它的工程定位是 **进程基础设施初始化器**。

### `setup.ts`

核心对外接口：

```ts
setup(
  cwd,
  permissionMode,
  allowDangerouslySkipPermissions,
  worktreeEnabled,
  ...
): Promise<void>
```

它负责：

- 设置 cwd / projectRoot / session
- 初始化 hooks snapshot
- 创建 worktree / tmux
- 初始化 session memory
- 加载 commands
- 启动与工作目录、git、session 相关的准备逻辑

它的定位是 **会话启动装配器**。

## 5.3 与其他模块的交互

```text
cli.tsx
  -> init.ts
  -> setup.ts
  -> 创建运行模式
  -> 进入 REPL / Remote / Daemon / MCP server / Bridge
```

关键特点：

- 入口模块只装配，不长期持有业务状态。
- 启动路径大量使用动态 import，目的是减少 cold start 代价。
- 初始化顺序经过精心设计，例如先 safe env，再 trust，再 telemetry。

---

## 6. 交互与会话壳模块

## 6.1 职责

这部分对应“用户看见和直接操作的运行壳”。

核心文件：

- `src/screens/REPL.tsx`
- `src/state/AppStateStore.ts`
- `src/state/AppState.tsx`
- `src/context/*`

它承担：

- 终端 UI 渲染
- 输入事件收集
- 命令队列与 prompt 提交
- 会话级 React 状态
- 通知、弹窗、权限请求、任务面板等交互控制

## 6.2 对外接口

### `AppStateStore.ts`

对外暴露的核心接口是 `AppState` 类型和默认状态构造。

`AppState` 是会话壳层的共享状态协议，内容包括：

- `mcp.clients/tools/commands/resources`
- `plugins.enabled/disabled/errors`
- `toolPermissionContext`
- `tasks`
- `notifications`
- `elicitation.queue`
- `agentDefinitions`
- `fileHistory`
- `attribution`

它不是纯 UI state，而是 **会话控制面状态模型**。

### `AppState.tsx`

对外接口：

```ts
AppStateProvider
useAppState(selector)
useSetAppState()
useAppStateStore()
```

工程上非常关键的一点：

- 项目没有引入 Redux/Zustand，而是自己实现了轻量 store。
- React 层通过 `useSyncExternalStore` 订阅切片。
- 非 React 层则通过 `getAppState()/setAppState()` 与执行内核解耦。

这让 UI 层与 runtime 层能共享同一状态模型，但不共享实现方式。

### `REPL.tsx`

对外看它只是一个屏幕组件，实际上它是 **interactive runtime shell**：

- 组装 `commands` / `tools` / `mcpClients`
- 维护 `messages`
- 构造 `ProcessUserInputContext`
- 调用 `query()`
- 处理权限、MCP elicitation、任务视图、远程会话同步

REPL 模块并不直接拥有模型执行逻辑，而是作为壳层编排者消费执行内核。

## 6.3 状态拆分

会话壳层内部又分成两类状态：

### 进程级基础设施状态

由 `bootstrap/state.ts` 承担，服务于：

- sessionId
- telemetry provider
- global cost/token counters
- cwd / projectRoot
- global feature latch
- prompt cache stability 状态

### React 会话视图状态

由 `AppState` 承担，服务于：

- 当前会话可视状态
- 插件/MCP/任务/通知的交互控制
- permission context

这种双状态分层是工业级 CLI/Agent 系统的典型设计：**进程基础设施态与 UI 交互态分离**。

---

## 7. 对话执行内核模块

## 7.1 职责

这是项目最核心的运行时内核。

关键文件：

- `src/QueryEngine.ts`
- `src/query.ts`
- `src/query/deps.ts`
- `src/services/api/claude.ts`
- `src/services/api/client.ts`

## 7.2 子模块划分与接口

### A. `QueryEngine`

定位：

- 会话级执行编排器。

典型职责：

- 聚合 `messages`
- 构造 `ToolUseContext`
- 调用 `query()`
- 把流式执行事件转成上层可消费输出

对外接口形态是一个 class，它让 REPL、SDK、remote 等入口都能共享同一执行壳。

### B. `query.ts`

定位：

- 单个 turn 的执行状态机。

对外接口核心是：

```ts
query(params: QueryParams): AsyncGenerator<...>
```

它负责：

- 调用模型
- 收集 `tool_use`
- 调度工具
- 管理 compact / retry / stop hooks / interrupt

### C. `query/deps.ts`

定位：

- 对 `query()` 的少量关键依赖做显式注入。

接口：

```ts
type QueryDeps = {
  callModel
  microcompact
  autocompact
  uuid
}
```

它体现的是一种很成熟的模块化技巧：**对最关键、最难 mock 的依赖做窄接口注入**，而不是让 `query.ts` 直接强耦合一切实现。

### D. `services/api/claude.ts`

定位：

- 模型请求编译器与流式响应适配器。

它不是单纯 HTTP client，而是负责：

- 把内部消息协议编译成 Anthropic 请求体
- 处理不同 provider 的 beta / header / body 参数
- 处理 streaming/non-streaming fallback
- 维护工具、thinking、cache、output config 等协议正确性

### E. `services/api/client.ts`

定位：

- 多云/多 provider 的实际 API client 工厂。

对外接口：

```ts
getAnthropicClient({...}): Promise<Anthropic>
```

它吸收了 Bedrock、Vertex、Foundry、直连 Anthropic 的差异，是典型的 **provider adapter module**。

## 7.3 内核模块交互链

```text
REPL / SDK / Remote
  -> QueryEngine
  -> query()
  -> claude.ts
  -> client.ts
  -> provider API
  -> tool loop
  -> 返回上层事件流
```

这个分层的核心价值是：

- `QueryEngine` 可以独立于具体 provider 存在。
- `query()` 可以独立于 UI 存在。
- `claude.ts` 可以独立于 REPL 存在。

也就是说，项目把“会话编排”“turn 状态机”“API 协议编译”拆成了 3 层，而不是糊成一个大函数。

---

## 8. 命令与工具能力模块

## 8.1 职责

这个模块提供两套正交能力接口：

- `Command`：面向用户输入面。
- `Tool`：面向模型执行面。

关键文件：

- `src/types/command.ts`
- `src/commands.ts`
- `src/Tool.ts`
- `src/tools.ts`
- `src/tools/*`

## 8.2 `Command` 接口定义

`types/command.ts` 定义了统一的命令协议：

```ts
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)
```

关键对外字段包括：

- `name`
- `description`
- `aliases`
- `availability`
- `isEnabled`
- `source`
- `loadedFrom`
- `immediate`
- `disableModelInvocation`

其中 `PromptCommand` 额外定义：

- `allowedTools`
- `model`
- `hooks`
- `context: 'inline' | 'fork'`
- `getPromptForCommand(args, context)`

这意味着 Command 不是单纯的 slash command 文本配置，而是一个完整的 **用户控制面协议对象**。

## 8.3 `Tool` 接口定义

`Tool.ts` 定义了模型工具协议和运行时上下文：

核心接口对象：

- `Tool`
- `ToolUseContext`
- `ToolPermissionContext`
- `ValidationResult`

`ToolUseContext` 对整个系统非常关键，因为它把工具执行所需依赖统一能力化，包括：

- `options.commands/tools/mcpClients/agentDefinitions`
- `getAppState()` / `setAppState()`
- `messages`
- `abortController`
- `requestPrompt`
- `appendSystemMessage`
- `updateFileHistoryState`
- `updateAttributionState`
- `contentReplacementState`

可以把它理解为：

- `Command` 是声明式用户入口协议。
- `ToolUseContext` 是执行期 capability object。

## 8.4 注册中心

### `commands.ts`

职责：

- 聚合内置 commands
- 聚合 skills
- 聚合 plugin commands
- 聚合 builtin plugin skill commands
- 统一做启用性/可见性治理

对外接口重点是 `getCommands()`。

### `tools.ts`

职责：

- 聚合所有基础工具
- 根据环境和 feature flag 过滤
- 与 MCP tools、plugin/skill tools 进行合并

对外接口重点是：

- `getAllBaseTools()`
- `getToolsForDefaultPreset()`

## 8.5 与执行内核的关系

```text
commands.ts
  -> 提供用户可调用能力目录

tools.ts
  -> 提供模型可调用能力目录

processUserInput
  -> 消费 Command

query.ts / claude.ts / toolExecution.ts
  -> 消费 Tool
```

这是整个项目模块化设计里最值得关注的一点：**用户入口目录和模型能力目录是两套注册中心，但最终都被统一进同一个 Agent Runtime**。

---

## 9. 扩展生态模块

## 9.1 模块组成

扩展生态由 3 个子模块组成：

1. MCP 模块
2. Skills 模块
3. Plugin 模块

它们的共同目标是把外部能力接入 `Command`/`Tool`/`AppState`/`QueryEngine` 所构成的统一运行时。

## 9.2 MCP 模块

关键文件：

- `src/services/mcp/types.ts`
- `src/services/mcp/*`

核心接口：

```ts
type ScopedMcpServerConfig = McpServerConfig & { scope; pluginSource? }

type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

MCP 模块对外暴露的是：

- server config schema
- connection state model
- tools/resources/commands 的适配结果

它本质上是一个 **协议适配与连接管理模块**。

## 9.3 Skills 模块

关键目录：

- `src/skills/*`

它负责把 Markdown skill 转换成 `PromptCommand`，再经由 `SkillTool` 暴露给模型。

因此 Skills 模块不是独立 runtime，而是：

```text
skill markdown
  -> PromptCommand
  -> commands.ts
  -> SkillTool
  -> query/tool loop
```

## 9.4 Plugin 模块

关键目录：

- `src/utils/plugins/*`
- `src/plugins/*`
- `src/services/plugins/*`

如 `loadPluginCommands.ts` 所示，插件会被转换成：

- plugin commands
- plugin skills
- plugin agents
- plugin hooks
- plugin MCP/LSP 配置

也就是说，Plugin 模块不是某种“直接执行的插件容器”，而是一个 **扩展物料装配系统**。

## 9.5 扩展模块的统一落点

所有扩展最终都会落入以下 4 个公共接口面：

1. `Command`
2. `Tool`
3. `AppState.mcp/plugins/agentDefinitions`
4. `settings/hooks/permission context`

这是一种非常成熟的平台化设计：外部扩展先被协议化，再被接入执行链，而不是直接给扩展一个“随便调内部函数”的入口。

---

## 10. 权限与安全治理模块

## 10.1 职责

权限模块负责约束“能力是否可见”和“能力是否可执行”。

关键接口并不集中在一个文件，而是贯穿于：

- `types/permissions.ts`
- `Tool.ts`
- `utils/permissions/*`
- `components/permissions/*`
- `utils/sandbox/*`

## 10.2 核心接口

### `ToolPermissionContext`

由 `Tool.ts` 定义，是工具执行的统一权限上下文：

- `mode`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `additionalWorkingDirectories`
- `shouldAvoidPermissionPrompts`

### `PermissionResult`

由权限模块返回的结构化决策结果，供：

- 工具执行器
- REPL 审批 UI
- remote control callback
- swarm leader/worker 同步

共同消费。

## 10.3 模块交互方式

```text
tools.ts
  -> 过滤工具暴露

toolExecution.ts
  -> 发起权限判定

REPL / RemoteSessionManager / leaderPermissionBridge
  -> 作为审批后端

sandbox-adapter
  -> 提供 OS 级执行约束
```

这个模块的关键工程点在于：它不是 UI 组件，也不是单纯 rule matcher，而是一整套 **策略 + 协议 + 执行隔离** 联动系统。

---

## 11. 上下文与记忆治理模块

## 11.1 模块组成

关键文件：

- `src/context.ts`
- `src/services/compact/*`
- `src/memdir/*`
- `src/services/SessionMemory/*`

## 11.2 对外接口

### `context.ts`

对外提供：

```ts
getSystemContext(): Promise<Record<string, string>>
getUserContext(): Promise<Record<string, string>>
```

它负责把 git 状态、CLAUDE.md、当前日期等内容注入为会话上下文。

### compact 模块

负责提供：

- 自动压缩
- 微压缩
- 压缩后清理

这些接口被 `query.ts` 调用，而不是被 UI 直接调用。

### 记忆模块

负责：

- memory files 检索
- relevant memories 注入
- session memory 总结

## 11.3 交互机制

```text
context.ts
  -> QueryEngine 构造 system/user context

compact/*
  -> query.ts 在 turn 内调用

memdir / SessionMemory
  -> 作为长期上下文数据源输入 QueryEngine / system prompt
```

这说明上下文治理模块在架构上是“执行内核的前后置支持模块”，而不是纯存储模块。

---

## 12. 远程与分布式协同模块

## 12.1 模块组成

关键目录：

- `src/remote/*`
- `src/bridge/*`
- `src/server/*`
- `src/tasks/*`
- `src/utils/swarm/*`

## 12.2 `RemoteSessionManager` 接口

`remote/RemoteSessionManager.ts` 暴露了一个很清晰的远程会话模块接口：

```ts
type RemoteSessionConfig = {
  sessionId
  getAccessToken
  orgUuid
  hasInitialPrompt?
  viewerOnly?
}

type RemoteSessionCallbacks = {
  onMessage
  onPermissionRequest
  onPermissionCancelled
  onConnected
  onDisconnected
  onReconnecting
  onError
}
```

这表明 remote 模块不是直接把远程逻辑塞进 REPL，而是显式建模为：

- 远程消息通道
- 控制请求通道
- 权限响应通道

## 12.3 任务与子 agent

任务系统和 swarm/in-process agent 模块共同提供：

- 背景任务抽象
- 子 agent 生命周期
- leader/worker 权限同步
- foreground/background transcript 切换

它们把“并发 agent”纳入和主线程同构的执行体系，而不是单独再造一套 runtime。

## 12.4 远程模块与本地主路径的关系

```text
Remote / Bridge / Swarm
  -> 不重写 QueryEngine / Tool / Permission 协议
  -> 只改变 transport、审批后端、消息源和执行位置
```

这是这套架构非常强的地方：分布式协同没有破坏本地执行内核的同构性。

---

## 13. 观测与实验模块

## 13.1 模块职责

关键目录：

- `src/services/analytics/*`
- `src/utils/telemetry/*`
- profiler 相关 utils

它负责：

- 事件记录
- 实验与动态配置
- OTel tracing/logging/metrics
- profiler
- query/startup tracing

## 13.2 对外接口

最核心的统一接口是：

```ts
logEvent(...)
```

但从模块化角度更重要的是，它还对其他模块暴露：

- feature flag / dynamic config 读取能力
- query tracing 能力
- startup profiler checkpoints
- session/span 级 tracing

因此它不是旁路埋点模块，而是一个会影响 runtime 行为的控制面模块。

---

## 14. 基础设施模块

## 14.1 范围

基础设施模块分散在：

- `src/utils/*`
- `src/services/oauth/*`
- `src/services/api/client.ts`
- `src/utils/settings/*`
- `src/utils/auth.js`
- `src/utils/config.js`
- `src/utils/sessionStorage.js`

## 14.2 主要子能力

包括：

- 认证与凭证刷新
- 配置加载与热变更
- 文件系统与路径处理
- 会话 transcript 持久化
- git/worktree/tmux
- shell/provider 适配
- 平台差异处理

这些模块通常不直接被 UI 消费，而是被上层模块通过 helper 函数方式调用。

## 14.3 模块化特点

这层的设计特点不是“完全纯净”，而是：

- 尽量做 leaf module
- 让上层模块通过窄接口调用
- 允许一定的全局状态协同

这非常符合 CLI/Agent 工程现实，而不是过度追求教科书式 clean architecture。

---

## 15. 模块间交互主流程

## 15.1 启动装配流程

```text
cli.tsx
  -> init.ts
  -> setup.ts
  -> 构造 bootstrap/state
  -> 创建 AppState
  -> 加载 commands/tools/plugins/skills/MCP
  -> 进入 REPL / Remote / Daemon 模式
```

## 15.2 本地对话执行流程

```text
REPL.tsx
  -> processUserInput
  -> QueryEngine
  -> query.ts
  -> claude.ts
  -> toolExecution/toolOrchestration
  -> 权限/compact/telemetry
  -> AppState + transcript 持久化
```

## 15.3 扩展接入流程

```text
plugin/skill/mcp source
  -> loader / schema parser / connection manager
  -> Command / Tool / MCP state / agentDefinitions
  -> 合并进 AppState 与 ToolUseContext
  -> 被 REPL / QueryEngine / query 消费
```

## 15.4 远程审批流程

```text
remote websocket
  -> RemoteSessionManager
  -> REPL permission UI
  -> control_response
  -> 远端 query/tool loop 继续
```

---

## 16. 模块依赖关系

## 16.1 稳定依赖方向

项目整体依赖方向大致如下：

```text
入口与装配层
  -> 运行壳层
  -> 执行内核层
  -> 扩展生态层
  -> 基础设施层

运行壳层
  -> 执行内核层
  -> 能力注册层
  -> 权限治理层
  -> 远程层

执行内核层
  -> 能力注册层
  -> 权限治理层
  -> 上下文治理层
  -> 基础设施层

扩展生态层
  -> 能力注册层
  -> 基础设施层
  -> 权限治理层
```

最关键的不变量是：

- UI 不直接依赖 provider API 细节。
- 扩展模块不直接侵入 query loop 核心状态机。
- provider client 不依赖 REPL。
- 远程模块尽量复用本地协议，而不是重新定义消息模型。

## 16.2 几个典型的“桥接接口”

项目里真正承担模块桥接责任的接口，主要有以下几类：

### `Command`

桥接：

- skills
- plugins
- slash commands
- local JSX commands

### `Tool`

桥接：

- builtin tools
- MCP tools
- agent tools
- skill tools

### `Message`

桥接：

- UI transcript
- model API 请求
- tool result 回填
- session recovery
- remote message transport

### `AppState`

桥接：

- REPL 组件
- MCP 连接管理
- 插件状态
- 任务系统
- 权限 UI

### `ToolUseContext`

桥接：

- 执行内核
- 工具层
- 命令层
- React 状态层

## 16.3 依赖控制的工程技巧

从代码中可以看到几类有意识的依赖控制手法：

1. **动态 import**  
   大量用于启动优化和 feature DCE。

2. **类型集中定义**  
   如 `types/command.ts`、`Tool.ts`、`types/permissions.ts`，避免协议碎片化。

3. **上下文对象注入**  
   如 `ToolUseContext`、`QueryDeps`，把多依赖聚合成 capability object。

4. **Store 接口而非 React 直接耦合**  
   runtime 层只拿 `getAppState()/setAppState()`。

5. **统一中间表示**  
   扩展能力先收敛到 `Command` / `Tool` / `Message`。

---

## 17. 模块设计约束

## 17.1 不能破坏的模块边界

以下边界是稳定设计，不宜随意打破：

1. 不要让 REPL 直接操作 provider request 细节。
2. 不要让 plugin/skill 直接侵入 `query.ts` 内部状态。
3. 不要让权限系统退化成“工具内部随便判”。
4. 不要让远程模式单独定义一套工具/消息协议。
5. 不要把 `bootstrap/state` 和 `AppState` 混成一个状态容器。

## 17.2 为什么这些约束重要

因为该项目同时面临：

- 本地/远程双运行形态
- 用户/模型双入口
- 多 provider API
- 插件/MCP/skills 多源扩展
- 审批/沙箱/遥测等跨切面治理

如果没有这些边界，系统很快会退化成不可测试、不可恢复、不可扩展的“巨型 REPL”。

---

## 18. 工业级工程亮点

从模块化设计角度，项目里至少有 8 个明显的工业级亮点。

## 18.1 入口层和内核层分离得非常彻底

`cli.tsx` 只做 dispatch，`init.ts` 做 infra init，`setup.ts` 做 session setup，`REPL.tsx` 做交互壳，`QueryEngine/query.ts` 做执行内核。

这使得启动、交互、执行三条责任链清晰分离。

## 18.2 `Command` 与 `Tool` 的双接口体系非常成熟

这不是重复设计，而是明确区分：

- 用户控制面
- 模型执行面

很多 Agent 系统会把两者混成一个万能 action，长期看会严重损害扩展性和安全性。

## 18.3 扩展能力先协议化再接入

无论 skill、plugin 还是 MCP，最终都被编译到统一协议对象，而不是直接暴露内部模块函数。

这是平台级治理思维，而不是脚本拼装思维。

## 18.4 `ToolUseContext` 是一个很强的 runtime capability object

它把执行环境、状态入口、消息、UI prompt、文件历史、追踪状态等都装进一个统一能力对象。

这极大降低了工具实现和宿主运行时之间的耦合成本。

## 18.5 双状态系统很专业

- `bootstrap/state` 管进程级基础设施态。
- `AppState` 管会话壳层态。

这是复杂 CLI/Agent 产品比 demo 工具更成熟的典型标志。

## 18.6 远程模式没有破坏本地协议同构

`RemoteSessionManager` 只是 transport/controller adapter，核心消息、权限、工具协议没有被推翻重做。

这比很多“本地一套、远程一套”系统健壮得多。

## 18.7 执行内核对 provider 差异做了良好封装

`client.ts` 处理 provider 差异，`claude.ts` 处理请求编译，`query.ts` 专注 turn 状态机。

这个切法使模型接入的复杂性被限制在 API 边界，而没有扩散到整条主链。

## 18.8 观测、权限、compact 都是跨切面模块，而非散落逻辑

这说明项目已经进入“平台治理期”，不再是功能堆叠阶段。

---

## 19. 对阅读源码者的建议

如果要从模块化视角继续深入源码，建议按以下顺序：

1. 先读 `entrypoints/cli.tsx`、`entrypoints/init.ts`、`setup.ts`，理解组合根。
2. 再读 `state/AppStateStore.ts`、`bootstrap/state.ts`，理解双状态系统。
3. 再读 `types/command.ts`、`Tool.ts`、`commands.ts`、`tools.ts`，理解双接口体系。
4. 再读 `QueryEngine.ts`、`query.ts`、`services/api/claude.ts`，理解执行内核。
5. 最后分别进入 MCP、plugins、permissions、remote、analytics 这些横向模块。

这样最容易建立稳定的全局心智模型。

---

## 20. 总结

从模块化架构角度看，`Claude-code-open` 的核心设计不是“某个模块特别大”，而是它成功建立了一套统一的运行时协议面：

- 用户入口统一到 `Command`
- 模型能力统一到 `Tool`
- 执行事实统一到 `Message`
- 会话壳状态统一到 `AppState`
- 进程基础设施状态统一到 `bootstrap/state`

围绕这些协议对象，启动、执行、扩展、权限、远程、观测、记忆等模块被组织成了一套相对稳定的依赖结构。  
这也是为什么该项目虽然代码量很大、场景很多，但仍然能维持平台级可扩展性，而不是沦为一组松散功能堆叠。
