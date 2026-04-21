# 以架构视角，深入源码，逐层拆解 Claude Code 的 Subagent / Harness 设计

## 先给结论

`Claude Code` 并不是通过一个单独的“子代理子系统”来实现复杂协作，而是把“派生代理”拆进现有运行时：

1. `AgentTool` 把“启动一个代理”做成普通工具调用，子代理因此成为工具体系的一部分，而不是额外的调度总线。源码入口见：
   - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:196-250`
   - `vendor/Claude-code-open/src/tools/AgentTool/prompt.ts:202-212`
2. 真正的运行 Harness 由几层共同组成：
   - `AgentTool` 负责选型、建上下文、决定同步/异步、创建隔离环境
   - `runAgent` 负责组装 prompt、权限模式、工具池、MCP、hooks、transcript、生命周期清理
   - `query` / task framework 负责把异步结果重新送回主循环
   - `coordinator mode` 与 `team/swarm` 再在此基础上叠加并行协作和信箱通信
3. 原概要里把 `AgentTool` 子代理、`coordinator worker`、`team/swarm teammate` 混成了一个系统。源码显示这三者有关联，但不是同一层机制。

---

## 一、核心触发机制：Subagent 首先是一个 Tool

原稿“AgentTool 与平坦架构”的判断基本成立，而且这是源码里最清晰的一层。

- `AgentTool` 通过 `buildTool(...)` 注册为标准工具，名字就是 `AGENT_TOOL_NAME`，调用入口在 `call(...)`。见：
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:196-250`
- 它的输入 schema 就是“描述任务、prompt、subagent_type、model、background、team、mode、isolation、cwd”等普通工具参数，而不是特殊 DSL。见：
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:81-138`
- 工具提示语直接写明：`Launch a new agent to handle complex, multi-step tasks autonomously.`，并且把 agent 列表、可用工具、fork 行为都作为工具 prompt 的一部分暴露给模型。见：
  - `vendor/Claude-code-open/src/tools/AgentTool/prompt.ts:66-76`
  - `vendor/Claude-code-open/src/tools/AgentTool/prompt.ts:190-218`

这意味着“派生代理”在架构上不是中心调度器私有能力，而是被压平到工具层。主代理是否委派，本质上是一次普通 tool call。

---

## 二、真正的 Harness：`AgentTool.call()` 只负责起飞，`runAgent()` 才是运行时核心

### 1. `AgentTool.call()` 负责做哪些事

`AgentTool.call()` 不是直接“跑完子代理”，而是在进入 `runAgent()` 前完成几件关键准备工作：

- 解析 agent 类型、model、permission mode 和 tool pool
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:567-578`
- 提前生成 agent id
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:579-580`
- 视情况创建 worktree 隔离环境
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:582-593`
- 构造 fork path 所需的上下文与 system prompt 复用逻辑
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:595-636`
- 决定是同步执行还是注册为异步后台任务
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:555-567`
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:686-765`
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:766-858`
- 为异步和同步两种路径都注入 `AsyncLocalStorage` agent context
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:714-752`
  - `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:769-785`

所以从职责上说，`AgentTool` 更像“spawn harness”，`runAgent` 才是“execution harness”。

### 2. `runAgent()` 负责哪些运行时问题

`runAgent()` 是最核心的子代理生命周期函数。它做的事情远比“调用一次模型”多：

- 组装 agent id、transcript 子目录、trace 注册
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:347-359`
- 决定 fork context 如何进入子代理
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:368-379`
- 读取/裁剪 userContext 与 systemContext，例如对 Explore/Plan 精简 `CLAUDE.md` 和 `gitStatus`
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:380-410`
- 重写权限模式与 `shouldAvoidPermissionPrompts`
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:412-479`
- 解析 agent 的工具集，并和 agent 专属 MCP tools 合并
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:500-518`
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:648-665`
- 运行 `SubagentStart` hooks，并把 hooks 追加上下文插入首轮消息
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:530-555`
- 注册 frontmatter hooks，并把 `Stop` 转成 `SubagentStop`
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:557-575`
- 预加载 agent skills
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:577-645`
- 用 `createSubagentContext(...)` 创建隔离的 `ToolUseContext`
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:697-719`
- 保存 sidechain transcript 和 agent metadata，供 resume / background / UI 使用
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:732-745`
- 最终驱动 `query(...)` 执行，并在 `finally` 里做资源清理
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:747-859`

换句话说，Claude Code 的 Harness 设计哲学不是“造一个更大的 orchestrator”，而是把上下文、权限、工具、hooks、MCP、transcript、cache、cleanup 都收束到一个可复用的 agent runtime 里。

---

## 三、并行与编排：Coordinator Mode 是独立叠加层，不等同于普通 Subagent

原稿里“协调者与工作者模式”是对的，但需要收窄：这是 `coordinator mode` 的系统提示和约束，不是所有 `AgentTool` 子代理的通用行为。

### 1. Coordinator Mode 的角色定义

`getCoordinatorSystemPrompt()` 明确把当前主代理定义为 coordinator：

- “You are a coordinator”
- 负责拆任务、指挥 worker、综合结果、对用户输出
- 使用 `AgentTool` 启 worker，`SendMessage` 续跑 worker，`TaskStopTool` 停 worker

源码：
- `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:111-140`

### 2. 标准协作流程直接写在 coordinator prompt 里

源码明确给了四阶段：

- `Research`
- `Synthesis`
- `Implementation`
- `Verification`

并强调 coordinator 必须自己读懂研究结果后再写实施 prompt，不能说“based on your findings”。见：

- `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:198-227`
- `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:251-270`

### 3. “并行是超能力”是 coordinator prompt 的原文，不是分析者总结

原稿这点有源码支撑，而且几乎可以原样讲：

- `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:211-219`

其中核心句就是：

- workers are async
- launch independent workers concurrently
- make multiple tool calls in a single message

### 4. coordinator worker 的工具面是被收窄的

coordinator 模式下，主线程不是全工具代理，而是只保留输出与 worker 管理相关工具：

- `AGENT_TOOL_NAME`
- `TASK_STOP_TOOL_NAME`
- `SEND_MESSAGE_TOOL_NAME`
- `SYNTHETIC_OUTPUT_TOOL_NAME`

见：
- `vendor/Claude-code-open/src/constants/tools.ts:104-112`

因此，coordinator mode 的设计哲学不是“主代理更强”，而是“主代理更窄、更像调度器”。

---

## 四、通信机制：有两套，不要混讲

原稿里“XML 消息通信”说得过于统一。源码显示至少要拆成两类：

### 1. 主循环收到的异步任务通知：`<task-notification>`

后台 agent 完成后，会被编码成 `<task-notification>`，再塞回 pending notification queue：

- 任务通知 XML tag 定义
  - `vendor/Claude-code-open/src/constants/xml.ts:27-38`
- LocalAgentTask 生成 `<task-notification>`
  - `vendor/Claude-code-open/src/tasks/LocalAgentTask/LocalAgentTask.tsx:224-261`
- 通用 task framework 也会生成同类通知
  - `vendor/Claude-code-open/src/utils/task/framework.ts:271-290`
- `query()` 会按 `agentId` 作用域消费这些通知；主线程只消费自己的，subagent 只消费发给自己的
  - `vendor/Claude-code-open/src/query.ts:1560-1578`
  - `vendor/Claude-code-open/src/query.ts:1630-1643`
- coordinator prompt 也明确把 worker 结果描述为 user-role message 中的 `<task-notification>`
  - `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:142-165`

所以，`<task-notification>` 的准确定位是：**后台任务完成后的回流协议**，不是“代理之间通用消息总线”。

### 2. team/swarm 里的 teammate mailbox：底层是 JSON / inbox 文件，显示层才转 XML

swarm teammate 的邮箱是文件式 inbox：

- 路径：`~/.claude/teams/{team}/inboxes/{agent}.json`
- 写入时用 lock 避免并发冲突

见：
- `vendor/Claude-code-open/src/utils/teammateMailbox.ts:1-8`
- `vendor/Claude-code-open/src/utils/teammateMailbox.ts:56-66`
- `vendor/Claude-code-open/src/utils/teammateMailbox.ts:134-191`

消息展示给 UI 时才格式化成 `<teammate-message ...>` XML：

- `vendor/Claude-code-open/src/utils/teammateMailbox.ts:370-389`
- `vendor/Claude-code-open/src/constants/xml.ts:51-52`
- `vendor/Claude-code-open/src/components/messages/UserTeammateMessage.tsx:25-47`

因此更准确的表述是：

- **运行时通信载体**：mailbox JSON + inbox 文件
- **渲染层协议**：`<teammate-message>` XML

而不是“一切代理之间都通过 XML 通信”。

---

## 五、隔离机制：AsyncLocalStorage 是事实存在的，但“不同进程”只覆盖 teammate/swarm 一部分

原稿关于隔离的方向是对的，但要区分对象。

### 1. 普通 subagent 的隔离：`AsyncLocalStorage` + cloned `ToolUseContext`

`agentContext.ts` 直接说明：

- Subagents run in-process
- 背景并发时不能靠共享 AppState
- 用 `AsyncLocalStorage` 隔离 agent identity

见：
- `vendor/Claude-code-open/src/utils/agentContext.ts:1-22`
- `vendor/Claude-code-open/src/utils/agentContext.ts:32-54`
- `vendor/Claude-code-open/src/utils/agentContext.ts:93-110`

真正的运行态隔离在 `createSubagentContext(...)`：

- 默认克隆 `readFileState`
- 新建 `AbortController`
- 默认把 `setAppState` 变成 no-op
- 克隆 `contentReplacementState`
- 新建独立 query tracking chain

见：
- `vendor/Claude-code-open/src/utils/forkedAgent.ts:345-462`

### 2. in-process teammate 的隔离：另一套 `AsyncLocalStorage`

team/swarm 下的 in-process teammate 使用独立的 `TeammateContext`：

- `vendor/Claude-code-open/src/utils/teammateContext.ts:1-14`
- `vendor/Claude-code-open/src/utils/teammateContext.ts:16-41`
- `vendor/Claude-code-open/src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx:1-10`

### 3. “不同进程 / tmux / iTerm2 面板”不是普通 subagent 的默认形态

源码说明：

- 普通 `AgentTool` 子代理默认走 in-process runtime
- tmux / split-pane / swarm teammate 是 `spawnMultiAgent.ts` 那一套 teammate spawn 机制

见：
- `vendor/Claude-code-open/src/tools/shared/spawnMultiAgent.ts:1-4`
- `vendor/Claude-code-open/src/tools/shared/spawnMultiAgent.ts:188-259`
- `vendor/Claude-code-open/src/tools/shared/spawnMultiAgent.ts:300-304`

所以更准确的表述是：**Claude Code 既支持 in-process 隔离，也支持 team/swarm 的外部 pane/process 形态，但后者不是普通 subagent 的默认实现。**

---

## 六、共享暂存区：原稿里的 `tengu_scratch` 基本成立，但它属于 coordinator/permission 体系，不是 subagent 专属

原稿提到的 `tengu_scratch` 在源码里确实存在，不过准确名称是 `scratchpad`，并且它是“会话级 scratchpad 目录”，不是“SubAgent 私有 scratch”。

- 开关：`tengu_scratch`
  - `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:294-300`
- 目录格式：`/tmp/claude-{uid}/{sanitized-cwd}/{sessionId}/scratchpad`
  - `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:372-406`
- session init 时会创建该目录
  - `vendor/Claude-code-open/src/entrypoints/init.ts:202-209`
- coordinator mode 会把 `scratchpadDir` 注入 worker 上下文，并明确告诉 worker 这里可无权限读写，用于 cross-worker durable knowledge
  - `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:19-27`
  - `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:80-108`
- `QueryEngine` 把 scratchpadDir 传给 coordinator user context
  - `vendor/Claude-code-open/src/QueryEngine.ts:100-117`

所以可以讲为：**Claude Code 在 coordinator 模式下提供了一个会话级 scratchpad，作为 worker 间共享持久中间知识的落盘区。**

---

## 七、Prompt Cache：这不是“可能共享”，而是被明确设计成一等目标

原稿“缓存重用”是正确的，而且源码证据很强。

### 1. fork 子代理明确以 cache hit 为设计目标

`forkedAgent.ts` 开宗明义：

- forked agents 要共享与 parent 完全一致的 cache-critical params
- cache key 包含 system prompt、tools、model、messages prefix、thinking config

见：
- `vendor/Claude-code-open/src/utils/forkedAgent.ts:1-9`
- `vendor/Claude-code-open/src/utils/forkedAgent.ts:46-68`

### 2. AgentTool 的 fork path 会尽力保持 prefix byte-identical

包括：

- fork child 继承 parent system prompt
- 继承 parent exact tools
- 继承 parent context messages
- 特别说明这是为了 cache-identical prefix

见：
- `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:603-633`

### 3. `runAgent()` 里也明确继承 thinking config / tools 以保持 cache hit

- `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:666-695`
- `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:721-730`

### 4. AgentTool prompt 直接教育模型“fork 很便宜，因为 share your cache”

- `vendor/Claude-code-open/src/tools/AgentTool/prompt.ts:80-96`

因此这部分可以升格成一个分享重点：**Claude Code 的子代理设计不是单纯“能并行”，而是“并行时仍然强约束前缀一致性，尽量复用 prompt cache”。**

---

## 八、安全治理：需要拆成两类权限流

原稿里的“信箱审批模式”只说对了一部分。

### 1. 普通 `AgentTool` 子代理并不总是走 mailbox

普通 subagent 的权限治理主要靠：

- agent 级别工具过滤
  - `vendor/Claude-code-open/src/tools/AgentTool/agentToolUtils.ts:70-115`
- permission mode 重写与 `shouldAvoidPermissionPrompts`
  - `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:412-463`
- async agent 工具白名单
  - `vendor/Claude-code-open/src/constants/tools.ts:52-71`

也就是说，普通 subagent 的主防线是**工具面收窄 + 权限模式约束**。

### 2. mailbox 审批主要属于 swarm teammate / coordinator worker

这条链路在源码里很完整：

- `permissionSync.ts` 注释直接写明：worker 把 permission request 发给 leader mailbox，leader 决策后再回 worker mailbox
  - `vendor/Claude-code-open/src/utils/swarm/permissionSync.ts:1-19`
- 工具权限请求经 mailbox 发送
  - `vendor/Claude-code-open/src/utils/swarm/permissionSync.ts:669-783`
- sandbox 网络权限也经 mailbox 发送
  - `vendor/Claude-code-open/src/utils/swarm/permissionSync.ts:796-928`
- REPL 里 worker 若需网络权限，会转发给 leader，再等待响应
  - `vendor/Claude-code-open/src/screens/REPL.tsx:2216-2248`
- leader 端收到后，会在 UI 中展示审批，并把结果经 mailbox 发回 worker
  - `vendor/Claude-code-open/src/screens/REPL.tsx:4672-4710`
- in-process teammate 的 `canUseTool` 也优先走 leader dialog，bridge 不可用时回退到 mailbox
  - `vendor/Claude-code-open/src/utils/swarm/inProcessRunner.ts:116-127`
  - `vendor/Claude-code-open/src/utils/swarm/inProcessRunner.ts:195-220`

所以更精确的表述应为：**mailbox approval pattern 是 swarm/coordinator worker 的权限同步方案，而不是全部 subagent 的统一审批架构。**

### 3. “Atomic Claim” 不适合放在这篇讲稿里当主结论

我没有在 subagent/mailbox 主链路里找到“同一审批任务被多个 worker 原子申领”的明确实现。源码里大量 `atomic` 更多是：

- inbox / permission 文件写锁
- task notification 去重
- 文件原子写

最接近的是：

- inbox 写入加锁
  - `vendor/Claude-code-open/src/utils/teammateMailbox.ts:129-191`
- pending permission 文件写入加锁
  - `vendor/Claude-code-open/src/utils/swarm/permissionSync.ts:210-249`
- task notification 的 notified flag 原子去重
  - `vendor/Claude-code-open/src/tasks/LocalAgentTask/LocalAgentTask.tsx:224-240`

因此不建议继续保留“Atomic Claim 是 mailbox 审批核心机制”这种说法，除非另行补充 task-claim 子系统分析。

---

## 九、自主性约束：原稿这部分基本准确，且有非常直接的源码证据

### 1. 子代理默认禁用若干关键工具

`ALL_AGENT_DISALLOWED_TOOLS` 明确禁用了：

- `ASK_USER_QUESTION_TOOL_NAME`
- `EXIT_PLAN_MODE_V2_TOOL_NAME`
- `ENTER_PLAN_MODE_TOOL_NAME`
- 非 ant 用户下的 `AGENT_TOOL_NAME`
- `TASK_STOP_TOOL_NAME`

见：
- `vendor/Claude-code-open/src/constants/tools.ts:36-46`

这直接支撑原稿里“子代理通常不能 AskUserQuestion / ExitPlanMode”。

### 2. async agent 进一步收窄工具面

- `vendor/Claude-code-open/src/constants/tools.ts:55-71`
- `vendor/Claude-code-open/src/tools/AgentTool/agentToolUtils.ts:100-113`

### 3. Prompt 明确禁止“委派理解”

这点在两个 prompt 体系里都写得很狠：

- AgentTool 常规 prompt：
  - `vendor/Claude-code-open/src/tools/AgentTool/prompt.ts:99-113`
- coordinator prompt：
  - `vendor/Claude-code-open/src/coordinator/coordinatorMode.ts:253-259`

其中“Never delegate understanding”基本可以作为分享中的设计金句。

### 4. fork child 还有更强的行为约束

fork boilerplate 明确要求：

- 你不是主代理
- 不要再 spawn sub-agents
- 不要对话、不要提问
- 直接用工具
- 最后一次性汇报

见：
- `vendor/Claude-code-open/src/tools/AgentTool/forkSubagent.ts:171-198`

这说明 Claude Code 不是简单地“把一个代理再复制一份”，而是会对不同派生形态施加不同强度的行为收束。

---

## 十、对原概要的修正总结

以下判断建议在分享时按源码修正：

1. “SubAgent 的实现机制就是 Harness”
   - 可保留，但要说明 Harness 不是单一模块，而是 `AgentTool + runAgent + query/task + coordinator/swarm` 的组合。
2. “AgentTool 是核心触发机制”
   - 正确。
3. “Coordinator-Worker 是 Claude Code 的标准编排模式”
   - 只对 `coordinator mode` 成立，不适用于所有普通 subagent。
4. “代理之间通过 XML 消息通信”
   - 不准确。更准确是：`task-notification` 和 UI 展示层用了 XML；swarm mailbox 的底层载体是 JSON + inbox 文件。
5. “每个子代理用 AsyncLocalStorage 或独立进程隔离”
   - 需要拆开说。普通 subagent 默认是 in-process + ALS；独立 pane/process 主要是 teammate/swarm。
6. “tengu_scratch 是代理共享暂存区”
   - 基本成立，但它是会话级 scratchpad，主要在 coordinator 模式下暴露给 workers。
7. “Prompt cache 共享”
   - 正确，而且是明确设计目标。
8. “Mailbox approval 是所有 worker 的统一安全治理”
   - 过宽。它主要是 swarm/coordinator teammate 的权限同步机制。
9. “Atomic Claim 是该机制关键”
   - 证据不足，建议删除或降级表述。

---

## 可以作为分享收束页的总括

如果要把 Claude Code 的 Harness 设计哲学压缩成一句话，可以这样讲：

**它没有把“多代理”做成一个笨重的新框架，而是把代理派生压平为工具调用，再用统一的 agent runtime 去承接上下文隔离、权限约束、工具裁剪、prompt cache 复用、异步通知和 team mailbox 协作。**

从源码看，这套设计最有价值的不是“能起很多 agent”，而是：

- 派生入口统一：`AgentTool`
- 运行时统一：`runAgent`
- 回流协议统一：`<task-notification>`
- 权限治理按场景分层：普通 subagent 用工具/权限模式收窄，swarm teammate 用 leader/mailbox 协调
- 成本优化前置：fork path 从一开始就围绕 cache hit 设计

这才是 Claude Code Subagent Harness 真正值得讲的地方。
