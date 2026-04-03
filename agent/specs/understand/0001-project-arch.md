# Claude-code-open 项目架构理解

## 1. 结论概览

`vendor/Claude-code-open` 本质上不是一个“单纯调用 LLM 的 CLI”，而是一个以本地终端为主控面的 AI Agent 平台，核心能力可以概括为：

- 以 Bun + TypeScript 为运行时，React + Ink 构建终端交互界面。
- 以 `main.tsx` 为统一启动入口，按模式切换到 REPL、命令执行、远程会话、直连服务等不同运行路径。
- 以 `QueryEngine` + `services/api/claude.ts` 为主对话引擎，驱动模型请求、工具调用、权限审计、会话持久化。
- 以 `tools/`、`commands/`、`services/mcp/`、`utils/plugins/`、`skills/` 为扩展面，形成“内置工具 + MCP + 插件 + Skills + 子智能体”的统一能力体系。
- 以 `~/.claude`、项目内 `.claude`、会话 `jsonl`、内存目录 `memory/` 为核心持久化面，支持会话恢复、上下文积累、项目记忆、插件缓存和服务端索引。
- 以本地运行为默认形态，同时支持 Remote Session、Bridge、Direct Connect 三种远程协同形态。

从代码结构看，它更接近“本地 Agent OS”，而不是“命令行聊天工具”。

---

## 2. 业务架构

## 2.1 业务目标

项目承载的业务目标可以拆成 6 条主线：

1. 终端内完成 AI 编程协作。
2. 通过工具系统让模型可读写文件、执行命令、搜索、调用 MCP、管理任务。
3. 通过会话、记忆、历史、CLAUDE.md 保持长期上下文。
4. 通过插件、Skills、MCP 让能力可扩展。
5. 通过权限、沙箱、审批体系控制高风险动作。
6. 通过远程桥接和多智能体机制支持跨终端、跨会话、跨工作区协作。

## 2.2 业务能力地图

### A. 会话与对话业务

- 启动 CLI，进入 REPL 或非交互命令模式。
- 维护当前会话消息流、上下文窗口、模型配置、token/cost 使用量。
- 支持历史恢复、摘要压缩、回放、导出、rewind。

核心代码：

- `src/main.tsx`
- `src/screens/REPL.tsx`
- `src/QueryEngine.ts`
- `src/utils/sessionStorage.ts`

### B. Agent 执行业务

- 用户输入被转成模型请求。
- 模型输出普通文本、工具调用、权限请求、任务事件。
- 工具执行结果被回填到消息链继续推理。
- 支持 plan/worktree/任务/子 agent 等流程控制。

核心代码：

- `src/query.ts`
- `src/QueryEngine.ts`
- `src/services/api/claude.ts`
- `src/tools.ts`

### C. 开发工具业务

- 文件读写编辑、grep/glob、bash/powershell、notebook、LSP、web、todo、task、skill、MCP、agent。
- 工具既是运行能力，也是模型系统提示的一部分。

核心代码：

- `src/tools.ts`
- `src/tools/*`

### D. 扩展生态业务

- Slash Commands 提供人机交互入口。
- MCP 提供外部工具与资源接入。
- Plugins 提供命令、agent、hooks、MCP server 等扩展。
- Skills 提供 markdown 驱动的任务模板与约束。

核心代码：

- `src/commands.ts`
- `src/services/mcp/*`
- `src/utils/plugins/pluginLoader.ts`
- `src/skills/loadSkillsDir.ts`

### E. 长期上下文业务

- 全局历史 `history.jsonl`
- 会话 transcript `projects/<project>/<session>.jsonl`
- Auto Memory `memory/` + `MEMORY.md`
- Session Memory 自动总结当前会话
- 项目级 `.claude` 配置、MCP 配置、任务配置

核心代码：

- `src/history.ts`
- `src/utils/sessionStorage.ts`
- `src/memdir/*`
- `src/services/SessionMemory/*`
- `src/utils/config.ts`

### F. 远程协同业务

- Remote Session：CLI 作为本地终端视图，实际对话运行在远端会话。
- Bridge：本地进程与远程环境/会话桥接，支持长连接、会话分发、恢复、超时与多会话。
- Direct Connect：对自建 server 发起会话创建并通过 WS 交互。

核心代码：

- `src/remote/*`
- `src/bridge/*`
- `src/server/*`

## 2.3 业务主流程

### 业务主流程 1：本地交互式编码

```text
用户输入
  -> REPL UI 接收
  -> QueryEngine 组装消息、系统提示、工具清单、MCP/Skills/Memory
  -> services/api/claude.ts 发起流式请求
  -> 模型输出文本 / tool_use
  -> Tool 执行 + 权限校验
  -> tool_result 回填
  -> 持续循环直到 turn 完成
  -> transcript/history/cost/state 持久化
```

### 业务主流程 2：扩展能力加载

```text
启动
  -> 初始化配置与环境变量
  -> 加载内置命令/工具
  -> 读取插件缓存与 marketplace 插件
  -> 扫描 skills 目录
  -> 解析 MCP 配置并建立客户端连接
  -> 合并进 AppState / QueryEngine 可见能力集合
```

### 业务主流程 3：远程协同

```text
本地 CLI
  -> 创建/连接 Remote 或 Direct Connect Session
  -> 通过 WebSocket 接收 SDKMessage / control_request
  -> 本地 UI 展示消息与权限审批
  -> 用户决策回传 control_response
  -> 远端会话继续执行
```

---

## 3. 技术架构

## 3.1 总体分层

```text
表现层
  React + Ink 终端 UI

交互编排层
  main.tsx / REPL.tsx / commands.ts / QueryEngine.ts

能力层
  tools / commands / MCP / plugins / skills / tasks / assistant / bridge

领域服务层
  services/api / services/mcp / services/oauth / services/analytics / services/compact / services/lsp

状态与上下文层
  bootstrap/state.ts / state/AppStateStore.ts / history.ts / sessionStorage.ts

基础设施层
  fs/config/env/auth/network/proxy/sandbox/telemetry/process/platform utils
```

## 3.2 启动架构

`src/main.tsx` 是统一启动编排器，承担以下职责：

- 提前执行启动性能优化与副作用预热。
- 调用 `init()` 完成配置、证书、代理、遥测、LSP 清理钩子、scratchpad 初始化。
- 解析 CLI 参数与运行模式。
- 初始化命令、工具、插件、skills、MCP、remote settings、policy limits。
- 创建 Store 和初始 AppState。
- 启动 REPL、resume、doctor、remote、direct connect 等具体执行路径。

这说明 `main.tsx` 不是简单入口，而是应用内核的 composition root。

## 3.3 UI 与状态架构

项目存在两类状态：

### 全局运行时状态

`src/bootstrap/state.ts`

职责：

- 保存 sessionId、cwd、projectRoot、模型、token/cost、prompt cache、telemetry、agent 颜色、会话 lineage、远程模式标识等全局运行变量。
- 更接近“进程级状态容器”。

特点：

- 覆盖范围大，服务于 API、telemetry、session storage、memory、bridge 等非 React 模块。
- 偏基础设施状态。

### UI/会话视图状态

`src/state/AppStateStore.ts` + `src/state/AppState.tsx`

职责：

- 保存 REPL 展示与交互相关状态：tasks、notifications、mcp clients、plugins、permission context、footer selection、bridge 状态、bagel/tungsten 面板等。
- 通过自定义 `createStore` + `useSyncExternalStore` 驱动 React 渲染。

特点：

- 没有引入 Redux/Zustand，而是使用轻量自建 store。
- `onChangeAppState.ts` 负责状态变化副作用落盘，如 `saveGlobalConfig()`、权限模式同步、env 变量重载。

## 3.4 对话执行架构

`QueryEngine` 是对话生命周期的核心控制器。

职责：

- 聚合消息链、工具列表、命令、MCP、agent、memory、system prompt。
- 包装 `canUseTool`，跟踪权限拒绝。
- 调用底层 `query()` / `services/api/claude.ts` 发起模型请求。
- 处理 tool loop、usage 统计、session transcript 记录、异常归类。

底层 `services/api/claude.ts` 负责：

- 组装 Anthropic Beta Messages API 请求。
- 合并模型能力、thinking/effort/fast mode/cache headers。
- 处理 provider 差异、重试、fallback、流式事件、usage 统计、cache 策略。

因此：

- `QueryEngine` 是业务编排层。
- `services/api/claude.ts` 是模型接入层。

## 3.5 命令与工具架构

### Slash Commands

`src/commands.ts` 聚合 100+ 命令入口。

作用：

- 提供用户显式触发的操作面。
- 涵盖配置、MCP、插件、会话、review、diff、tasks、permissions、theme、usage 等。

### Tools

`src/tools.ts` 聚合 40+ 工具。

特点：

- 工具通过 feature flag、环境变量、权限上下文动态启用。
- 工具不仅决定执行能力，也决定暴露给模型的 tool schema。
- MCP/Skill/Agent/Task 也被纳入统一工具体系。

说明项目的工具架构是“统一调度模型”，而不是内置工具与外部工具分离。

## 3.6 MCP 架构

MCP 是项目最重要的外部扩展协议之一。

### 配置层

`src/services/mcp/config.ts`

- 负责汇总全局配置、项目配置、插件注入配置、企业托管配置、claude.ai connector 配置。
- 做 dedup、启停控制、作用域标记、URL/command 签名归一。

### 连接层

`src/services/mcp/client.ts`

- 支持 `stdio`、SSE、streamable HTTP、WebSocket、SDK control transport。
- 负责 client 连接、tool/resource/prompt 列举、tool call、认证缓存、结果截断与持久化。

### 暴露层

- `MCPTool`
- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

本质上，MCP 在项目中同时扮演：

- 外部工具源
- 外部资源源
- 外部 prompt/skill 源
- 企业连接器体系

## 3.7 插件与 Skills 架构

### Plugins

`src/utils/plugins/pluginLoader.ts`

插件加载支持：

- marketplace 插件
- session inline 插件
- manifest/hook/command/agent 加载
- 版本化缓存、seed cache、zip cache
- 策略校验、来源校验、错误收集

插件不仅能扩展命令，也能扩展 hooks、agents、MCP servers。

### Skills

`src/skills/loadSkillsDir.ts`

Skills 是 markdown 驱动的轻量能力层，支持：

- frontmatter 解析
- 参数替换
- hooks
- allowed-tools
- model/effort/user-invocable
- 从用户目录、项目目录、managed 目录、插件目录、bundled 目录、MCP 目录加载

因此：

- 插件偏“可安装扩展包”
- Skill 偏“可声明的任务模板/工作流提示”

## 3.8 权限与安全架构

权限安全链路贯穿全系统：

- `ToolPermissionContext` 控制工具是否可执行。
- `REPL.tsx` 负责权限请求 UI 和用户审批交互。
- `onChangeAppState.ts` 把 permission mode 同步到外部会话元数据。
- `utils/permissions/*` 提供规则、模式、更新、持久化。
- `utils/sandbox/*` 提供沙箱适配。
- remote/direct-connect/bridge 模式下，权限审批还能通过 control request/response 跨端协同。

说明该项目不是“执行前弹窗”这么简单，而是把权限模式作为核心状态机的一部分。

## 3.9 远程与桥接架构

存在三类远程模式：

### Remote Session

`src/remote/RemoteSessionManager.ts`

- 面向 CCR 远程会话。
- WebSocket 接收 SDKMessage / permission request。
- HTTP 发送用户消息。
- 本地处理审批，再回传远端继续执行。

### Bridge

`src/bridge/bridgeMain.ts`

- 面向常驻桥接环境。
- 维护 environment、work item、session spawn、heartbeat、timeout、reconnect、capacity control。
- 支持多会话、多 worktree、token refresh、长生命周期运行。

### Direct Connect

`src/server/createDirectConnectSession.ts`
- `src/server/directConnectManager.ts`

- 面向自建 server。
- 通过 HTTP 创建 session，通过 WS 走 SDK 格式消息。

这三类模式共享同一套 SDKMessage / control_request 交互模型，说明远程协同是第一等架构能力。

## 3.10 观测与实验架构

项目深度内建可观测性：

- analytics 事件
- 1P event logging
- datadog
- OpenTelemetry meter/tracer/logger provider
- startup profiler / query profiler / session tracing
- GrowthBook feature gate / dynamic config

它们不是外围工具，而是 runtime decision 的一部分，例如：

- 是否启用功能
- 是否使用某种 prompt cache/header
- session memory/remote 功能开关

---

## 4. 数据架构

## 4.1 数据分层

项目数据可分为 6 类：

1. 配置数据
2. 会话数据
3. 历史数据
4. 记忆数据
5. 扩展生态数据
6. 远程服务/索引数据

## 4.2 配置数据

### 全局配置

代码：`src/utils/config.ts`

典型位置：

- `~/.claude.json` 或等价 config home 文件

承载内容：

- 用户偏好
- oauth/account
- theme
- plugin/mcp/global settings
- IDE 配置
- onboarding 状态
- cached bootstrap / growthbook / changelog 等

### 项目配置

位置：

- `.claude/settings.json`
- 项目级 config 映射

承载内容：

- allowedTools
- mcpContextUris
- 项目级 MCP 配置
- trust/onboarding/worktree/session 指针等

## 4.3 会话与消息数据

### Transcript

代码：`src/utils/sessionStorage.ts`

形态：

- `~/.claude/projects/<sanitized-project>/<sessionId>.jsonl`

内容：

- user / assistant / attachment / system 消息
- attribution snapshot
- file history snapshot
- compact boundary
- queue operation 等日志项

特点：

- JSONL 追加写
- 支持 resume
- 子 agent transcript 有独立子目录

### 共享输入历史

代码：`src/history.ts`

形态：

- `~/.claude/history.jsonl`

内容：

- 用户输入历史
- pasted text / image 引用
- project + sessionId 维度索引

用途：

- 上下箭头历史
- ctrl+r 搜索

## 4.4 记忆数据

### Auto Memory

代码：

- `src/memdir/paths.ts`
- `src/memdir/memdir.ts`

路径模型：

- 默认：`~/.claude/projects/<sanitized-git-root>/memory/`
- 入口：`MEMORY.md`
- 日志：`logs/YYYY/MM/YYYY-MM-DD.md`

数据语义：

- 用户长期偏好
- 协作反馈
- 项目背景
- 参考知识

特点：

- 有严格 memory taxonomy
- 入口文件有大小与行数截断
- 可被 system prompt 自动注入

### Session Memory

代码：`src/services/SessionMemory/sessionMemory.ts`

语义：

- 当前会话中的持续总结
- 通过 forked subagent 后台更新
- 与长期 Auto Memory 区分开

## 4.5 扩展数据

### 插件缓存

代码：`src/utils/plugins/pluginLoader.ts`

形态：

- `~/.claude/plugins/cache/<marketplace>/<plugin>/<version>/`

还包含：

- legacy cache
- seed cache
- zip cache

### Skills 数据

来源：

- `~/.claude/skills`
- 项目 `.claude/skills`
- managed 目录
- 插件目录
- bundled skills
- MCP skills

数据本体是 markdown + frontmatter。

### MCP 配置

代码：`src/services/mcp/config.ts`

来源：

- 全局配置
- 项目 `.mcp.json`
- 企业托管 `managed-mcp.json`
- 插件注入
- claude.ai connectors

## 4.6 远程与服务数据

### Direct Connect / Server Session 索引

代码：`src/server/types.ts`

形态：

- `~/.claude/server-sessions.json`

作用：

- 保存 stable session key 到 session metadata 的映射
- server 重启后恢复 detached session

### MCP needs-auth cache

代码：`src/services/mcp/client.ts`

形态：

- `mcp-needs-auth-cache.json`

作用：

- 记录 MCP server 认证失败短期缓存，减少重复噪音

## 4.7 关键数据关系

```text
GlobalConfig
  -> projects[projectKey] -> ProjectConfig
  -> mcpServers / oauth / theme / telemetry cache

ProjectConfig
  -> allowedTools / mcpContextUris / trust / worktree session

Transcript(jsonl)
  -> Message chain
  -> sessionId
  -> agent transcript subtree

History(jsonl)
  -> display
  -> pastedContents
  -> project
  -> sessionId

MemoryDir
  -> MEMORY.md
  -> topic memory files
  -> daily logs
```

---

## 5. 部署架构

## 5.1 默认部署形态：单机本地 CLI

这是最核心形态：

- 进程直接运行在开发者终端。
- 本地访问文件系统、Git、Shell、IDE、MCP stdio 子进程。
- 模型 API 走外部网络。
- 所有状态和持久化主要落在本机 `~/.claude` 和项目目录。

```text
Developer Terminal
  -> Claude Code CLI Process
    -> Local FS / Git / Shell / IDE
    -> Local plugin cache / transcript / memory
    -> External Anthropic API / MCP HTTP services
```

## 5.2 远程会话部署形态

### 模式 A：Remote Session / CCR

- 本地 CLI 主要承担 UI 和审批职责。
- 远程会话运行在服务端环境。
- 消息与权限请求通过 WebSocket + HTTP 在本地与远程之间流转。

适用场景：

- 远程容器
- web 端会话接管
- 跨设备接续

### 模式 B：Bridge 常驻桥

- 本地或某台宿主机上运行 bridge loop。
- bridge 负责注册环境、拉取 work、启动/管理多个 claude 子会话。
- 支持多 session、工作树隔离、超时和心跳。

适用场景：

- 多任务后台代理
- 远程执行节点
- 企业环境编排

### 模式 C：Direct Connect Server

- 外部 server 暴露 `/sessions` 与 WebSocket。
- CLI 发起会话并按 SDK 协议收发消息。

适用场景：

- 私有化宿主
- 自建代理网关
- 受控执行沙箱

## 5.3 部署单元拆分

从代码看，部署单元可抽象为：

1. CLI 主进程
2. 子进程工具执行单元
3. MCP server 单元
4. Bridge worker/session 单元
5. Direct-connect server 单元
6. 外部模型/API 服务单元

说明项目天然是“多进程 + 多连接器”架构，而非单体进程闭环。

## 5.4 外部依赖拓扑

```text
Claude Code CLI
  -> Anthropic API / OAuth / bootstrap API
  -> GrowthBook / telemetry sinks
  -> MCP servers
    -> stdio child process
    -> SSE endpoint
    -> streamable HTTP endpoint
    -> WebSocket endpoint
  -> IDE clients
  -> Local shell / git / filesystem
  -> Remote CCR / direct-connect server
```

## 5.5 部署特征总结

- 默认是 client-heavy 架构，核心逻辑大量运行在本地。
- 远程模式不是替代本地，而是把执行面和展示面分离。
- 没看到仓库内置 Dockerfile、compose、CI workflow，说明当前 vendor 目录更像“源码镜像”而非完整工程交付根目录。
- 部署复杂度主要来自运行模式分化，而不是基础设施模板。

---

## 6. 关键模块关系图

```text
main.tsx
  -> entrypoints/init.ts
  -> commands.ts
  -> tools.ts
  -> createStore(AppState)
  -> screens/REPL.tsx / other launchers

REPL.tsx
  -> QueryEngine / query.ts
  -> permissions UI
  -> hooks / notifications / tasks / IDE / remote session

QueryEngine.ts
  -> system prompt assembly
  -> memory loading
  -> processUserInput
  -> services/api/claude.ts
  -> sessionStorage.ts

services/api/claude.ts
  -> Anthropic SDK
  -> retry / fallback / usage / streaming

tools.ts
  -> built-in tools
  -> MCP tools
  -> Agent/Skill/Task tools

services/mcp/*
  -> config aggregation
  -> transport connection
  -> tool/resource exposure

pluginLoader.ts + loadSkillsDir.ts
  -> extension discovery
  -> command/agent/hook/skill injection

sessionStorage.ts + history.ts + memdir/*
  -> durable context persistence

remote/* + bridge/* + server/*
  -> remote execution and session transport
```

---

## 7. 架构判断

## 7.1 架构风格

项目整体是以下几种风格的组合：

- 本地优先的富客户端架构
- 插件化平台架构
- Agent tool loop 架构
- 事件流 + 状态机驱动架构
- 多运行模式的混合部署架构

## 7.2 架构优势

- 核心能力统一围绕消息链、工具链、权限链展开，主线清晰。
- 扩展点完整，插件/Skills/MCP/子 agent 互相可组合。
- 远程模式与本地模式共享同一套消息协议，复用度高。
- 数据持久化体系比较完整，适合长会话与长期协作。

## 7.3 架构复杂点

- `main.tsx`、`REPL.tsx`、`services/api/claude.ts` 体量很大，承担了过多编排责任。
- feature flag、env、用户类型、远程模式叠加后，分支复杂度高。
- 全局状态 `bootstrap/state.ts` 范围很广，长期可能形成维护负担。
- UI 状态、运行时状态、持久化状态三层交叉较多，理解成本高。

---

## 8. 适合继续深入的阅读顺序

如果要继续深挖，建议顺序如下：

1. `src/main.tsx`
2. `src/screens/REPL.tsx`
3. `src/QueryEngine.ts`
4. `src/services/api/claude.ts`
5. `src/tools.ts` 与重点工具
6. `src/services/mcp/config.ts` + `src/services/mcp/client.ts`
7. `src/utils/plugins/pluginLoader.ts` + `src/skills/loadSkillsDir.ts`
8. `src/utils/sessionStorage.ts` + `src/history.ts` + `src/memdir/*`
9. `src/remote/*` + `src/bridge/*` + `src/server/*`

---

## 9. 本次梳理依据

本次文档基于以下代码主干阅读整理：

- 启动与状态：`src/main.tsx`、`src/entrypoints/init.ts`、`src/bootstrap/state.ts`、`src/state/*`
- 交互执行：`src/screens/REPL.tsx`、`src/QueryEngine.ts`、`src/services/api/claude.ts`
- 工具与命令：`src/tools.ts`、`src/commands.ts`
- 扩展体系：`src/services/mcp/*`、`src/utils/plugins/pluginLoader.ts`、`src/skills/loadSkillsDir.ts`
- 数据持久化：`src/utils/config.ts`、`src/utils/sessionStorage.ts`、`src/history.ts`、`src/memdir/*`、`src/services/SessionMemory/*`
- 远程协同：`src/remote/RemoteSessionManager.ts`、`src/bridge/bridgeMain.ts`、`src/server/*`

同时结合目录规模观察：

- `src/commands` 约 101 个一级子项
- `src/tools` 约 43 个一级子项
- `src/services` 约 36 个一级子项
- `src/components` 约 144 个一级子项

这进一步说明该项目已形成平台级复杂度，而非单一 CLI 工具。
