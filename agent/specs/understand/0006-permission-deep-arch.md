# Claude-code-open 权限与安全架构深度设计解读

## 1. 文档定位

本文不是入门版架构说明，而是面向 AI 应用架构师、算法专家、Agent 系统设计者的深度设计解读。

参考文档：

- `./specs/understand/0001-project-arch.md`
- `./specs/understand/0006-permission-arch.md`

本文目标是从“工业级 Agent 安全控制框架”的角度，系统分析 `vendor/Claude-code-open` 权限与安全架构相关模块的：

1. 抽象边界与接口定义
2. 关键数据结构与状态组织
3. 模块间交互协议与控制流
4. 规则系统、模式系统、路径系统、审批系统、沙箱系统之间的耦合方式
5. 并发控制、失败处理、可观测性、策略治理等工业级工程设计
6. 对 AI Agent 平台设计有参考价值的创新点与工程价值

本文基于以下代码主链阅读整理：

- `src/types/permissions.ts`
- `src/Tool.ts`
- `src/utils/permissions/*`
- `src/hooks/useCanUseTool.tsx`
- `src/hooks/toolPermission/*`
- `src/components/permissions/*`
- `src/utils/sandbox/sandbox-adapter.ts`
- `src/screens/REPL.tsx`
- `src/state/AppStateStore.ts`
- `src/QueryEngine.ts`
- `src/remote/RemoteSessionManager.ts`
- `src/hooks/useReplBridge.tsx`

---

## 2. 总体结论

从系统设计角度看，Claude-code-open 的权限与安全架构不是一个“审批弹窗模块”，而是一个围绕 Agent 工具执行构建的**多层控制面**：

```text
策略面
  PermissionRule / PermissionMode / Policy / Gate / Settings

决策面
  hasPermissionsToUseTool() / tool.checkPermissions()

资源安全面
  filesystem.ts / pathValidation.ts / matchingRuleForInput()

审批编排面
  useCanUseTool() / PermissionContext / interactiveHandler

分布式协同面
  bridge / remote / channel / swarm worker-leader relay

隔离执行面
  sandbox-adapter.ts / sandbox-runtime

观测治理面
  permissionLogging / analytics / OTel / denialTracking / killswitch
```

这套设计的本质不是单点授权，而是：

**把 Agent 执行中的“能力开放”建模成一个可治理、可解释、可降级、可隔离、可观测的控制系统。**

---

## 3. 设计问题域建模

如果从 AI 应用与算法系统角度抽象，这套架构解决的是一个典型的 Agent Control Problem：

### 3.1 目标

系统希望最大化：

- 自动化程度
- 执行效率
- 用户交互体验

同时最小化：

- 越权执行
- 敏感路径修改
- 网络或 shell 风险扩散
- 分布式审批竞态错误
- headless agent 的失控运行

### 3.2 本质冲突

这类系统始终存在几个矛盾：

1. 自动化 vs 可控性
2. 通用工具能力 vs 细粒度约束
3. 多通道审批 vs 决策唯一性
4. 高层策略表达 vs 底层 OS 约束能力
5. 用户体验 vs fail-closed 安全策略

Claude-code-open 的核心价值，在于它没有用单一机制硬扛这些矛盾，而是按层拆解：

- 规则解决表达问题
- mode 解决策略切换问题
- path check 解决资源安全问题
- interactive handler 解决异步审批问题
- sandbox 解决执行隔离问题
- telemetry / denial tracking 解决治理问题

这就是它的系统性。

---

## 4. 抽象模型

## 4.1 权限系统的输入输出抽象

从纯函数角度，可以把核心权限引擎写成：

```ts
PermissionDecision = F(
  Tool,
  ToolInput,
  ToolPermissionContext,
  ToolUseContext,
  ConversationState,
  RuntimePolicy
)
```

其中：

- `Tool` 描述能力类型
- `ToolInput` 描述具体动作
- `ToolPermissionContext` 描述权限状态
- `ToolUseContext` 描述执行环境
- `ConversationState` 描述消息链和 agent 行为上下文
- `RuntimePolicy` 来自 settings / policy / feature gate / remote state

输出统一收敛为：

- `allow`
- `ask`
- `deny`

这是整个架构最重要的稳定抽象。

## 4.2 决策分层抽象

实际实现中，这个函数不是一个单体，而是分成 3 层：

```text
全局权限引擎
  permissions.ts

工具局部权限引擎
  tool.checkPermissions()

资源与环境安全引擎
  filesystem.ts / pathValidation.ts / sandbox-adapter.ts
```

这种分层的含义是：

- “全局策略”与“工具局部策略”不在一个层面混写
- “逻辑授权”与“资源安全”不在一个层面混写
- “授权”与“执行隔离”不在一个层面混写

这正是复杂 Agent 系统要保持可维护性的前提。

---

## 5. 核心接口定义

## 5.1 `PermissionDecision` / `PermissionResult`

`src/types/permissions.ts`

这是整套权限系统最关键的协议类型。

### `PermissionDecision`

```ts
type PermissionDecision =
  | PermissionAllowDecision
  | PermissionAskDecision
  | PermissionDenyDecision
```

### `PermissionResult`

```ts
type PermissionResult =
  | PermissionDecision
  | { behavior: 'passthrough', ... }
```

这里最重要的设计点不是 union 本身，而是语义分层：

- `PermissionDecision` 是全局决策终态
- `PermissionResult` 是中间层返回值，允许 `passthrough`

这意味着：

1. 工具可以不直接做最终裁决
2. 工具只需要声明“我这里有没有特别的安全意见”
3. 全局引擎保留最终仲裁权

这是很成熟的可插拔权限协议设计。

### `ask` 作为一等结果

传统 ACL 系统常常只有 allow/deny，而这里把 `ask` 单独建模为 first-class result。

其意义在于：

- 它允许人类审批介入成为正式控制节点
- 它允许“先自动检查，再人工兜底”的分级策略
- 它允许 UI/remote/channel/swarm 等审批后端共享同一种协议

这使系统天然支持“人机协同控制”。

## 5.2 `PermissionDecisionReason`

这套系统最容易被忽略、但最有工业价值的接口之一，就是 `decisionReason`。

主要类型包括：

- `rule`
- `mode`
- `subcommandResults`
- `permissionPromptTool`
- `hook`
- `asyncAgent`
- `sandboxOverride`
- `classifier`
- `workingDir`
- `safetyCheck`
- `other`

这相当于给每一次权限决策附带了一个 explainability payload。

它的工程价值非常大：

1. 让 UI 能生成高质量解释文本
2. 让 telemetry 能区分真正的原因来源
3. 让后续工具循环或模型反馈可理解“为什么失败”
4. 让开发者排查复杂模式交互问题

对 AI 系统而言，这类“结构化拒绝原因”非常关键，因为它可以作为模型后续自我修正的上下文。

## 5.3 `ToolPermissionContext`

`src/Tool.ts`

这是运行时权限状态的聚合接口。

```ts
type ToolPermissionContext = {
  mode: PermissionMode
  additionalWorkingDirectories: Map<string, AdditionalWorkingDirectory>
  alwaysAllowRules: ToolPermissionRulesBySource
  alwaysDenyRules: ToolPermissionRulesBySource
  alwaysAskRules: ToolPermissionRulesBySource
  isBypassPermissionsModeAvailable: boolean
  isAutoModeAvailable?: boolean
  strippedDangerousRules?: ToolPermissionRulesBySource
  shouldAvoidPermissionPrompts?: boolean
  awaitAutomatedChecksBeforeDialog?: boolean
  prePlanMode?: PermissionMode
}
```

可以把这个类型理解成一个小型权限状态机的完整快照。

值得注意的设计：

### 1. `DeepImmutable`

它被定义为 `DeepImmutable`，意味着：

- 读路径上默认禁止随意原地突变
- 写路径必须通过 update/apply 机制

这对于复杂并发 UI/agent 系统是非常重要的约束。

### 2. 三类规则分桶

- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`

这不是简单的“规则列表 + behavior 字段”，而是运行时已按行为预分桶。

这是一种用空间换时间、用结构降低复杂度的做法：

- 查询快
- 优先级清晰
- UI 管理也更简单

### 3. 状态字段显式承载模式切换副作用

例如：

- `strippedDangerousRules`
- `prePlanMode`

这些字段说明 mode transition 不是纯展示切换，而是会引入上下文变换和可逆状态。

### 4. 行为约束字段

- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`

这两个字段本质上不是“权限规则”，而是“审批后端能力约束”。

它们把“当前执行环境是否具备交互能力”建模进权限上下文，这是非常工业化的设计。

## 5.4 `ToolUseContext`

`ToolUseContext` 不是权限模块专属接口，但它决定了权限系统的执行边界。

关键字段包括：

- `abortController`
- `getAppState()` / `setAppState()`
- `setAppStateForTasks`
- `handleElicitation`
- `addNotification`
- `messages`
- `agentId`
- `agentType`
- `toolDecisions`
- `localDenialTracking`

从权限架构角度，它的意义是：

1. 权限系统不是孤立纯函数，而是可感知会话状态、消息历史、agent 身份的控制器
2. 权限系统拥有“中止当前 agent”、“发通知”、“持久化决策”、“读写全局状态”的能力
3. 这使它不仅能做授权，还能做执行流程控制

因此可以说：

**ToolPermissionContext 描述“允许什么”，ToolUseContext 描述“系统此刻能怎么处理这个允许/拒绝”。**

## 5.5 `CanUseToolFn`

`src/hooks/useCanUseTool.tsx`

```ts
type CanUseToolFn = (
  tool,
  input,
  toolUseContext,
  assistantMessage,
  toolUseID,
  forceDecision?
) => Promise<PermissionDecision>
```

这个接口是整个权限系统的对外统一入口。

它的设计价值在于：

1. QueryEngine 只依赖一个统一权限回调，不需要感知 REPL/UI 细节
2. SDK、print、REPL、remote、subagent 都可以复用同一权限决策协议
3. `forceDecision` 允许上层把某个外部已知决策注入进流程

这相当于把“权限判断 + 审批编排”做成了一个平台级 capability middleware。

---

## 6. 关键模块职责与交互机制

## 6.1 `permissions.ts`：统一权限决策内核

这是主控制内核，负责：

1. 读取当前规则集
2. 调用工具级 `checkPermissions()`
3. 执行全局模式逻辑
4. 处理 auto/dontAsk/headless 等高层策略
5. 维护 denial tracking
6. 在必要时把 ask 转为 deny 或 classifier decision

这个模块的核心特点是：

- 统一仲裁
- 工具自治
- 模式收口
- 风险兜底

### 设计约束

它内部非常强调决策顺序，原因是顺序本身就是安全语义的一部分。

例如：

- deny 必须先于 allow
- safetyCheck 必须先于 bypass
- dontAsk 必须发生在 ask 结果最终返回前
- auto 只对 ask 生效，不能篡改显式 deny

这不是代码实现细节，而是安全不变量。

## 6.2 `tool.checkPermissions()`：工具局部安全语义层

统一引擎并不知道每个工具的细粒度风险表达。

例如：

- Bash 需要分析命令结构
- FileEdit 需要分析目标路径
- WebFetch 需要分析 domain
- Agent 需要分析 agent type

因此工具通过 `checkPermissions()` 暴露自己的专属语义。

这层的价值在于：

- 全局引擎保持通用
- 工具风险建模留在最懂该工具的模块中
- 新工具可以低成本接入统一权限框架

这相当于 policy engine + domain-specific checker 的双层结构。

## 6.3 `PermissionContext`：审批上下文对象

`src/hooks/toolPermission/PermissionContext.ts`

它是 ask 路径的 orchestration object。

其职责包括：

- `logDecision`
- `persistPermissions`
- `resolveIfAborted`
- `cancelAndAbort`
- `tryClassifier`
- `runHooks`
- `buildAllow` / `buildDeny`
- `handleUserAllow`
- `handleHookAllow`
- queue 操作

从架构上看，它把 ask 阶段的“跨模块公共操作”收束为一个上下文对象。

否则这些逻辑会散落在：

- useCanUseTool
- 各种 handler
- UI 组件
- bridge callback
- swarm callback

这是一种很成熟的 orchestration context 模式。

## 6.4 `useCanUseTool()`：ask 路径调度器

它不是单纯的 React hook，而是权限系统的运行时调度器。

它做的事情可以概括为：

```text
hasPermissionsToUseTool()
  -> allow: 直接结束
  -> deny: 记录 + 返回
  -> ask:
       coordinator automated checks
       -> swarm worker path
       -> speculative classifier path
       -> interactive path
```

它把 ask 阶段拆成多个 resolver pipeline。

关键设计点：

### 1. 自动化 resolver 前置

如果 `awaitAutomatedChecksBeforeDialog` 为真，先运行：

- hooks
- classifier

只有它们都不能解决，才打断用户。

这是一种典型的“最小打扰原则”。

### 2. speculative classifier

对于 Bash，在主 agent 上还支持投机式 classifier race：

- 给 classifier 一个短时间窗口
- 如果高置信命中，直接 auto-approve
- 否则再显示 dialog

这是明显的工业级 UX 优化设计：

- 减少弹窗频率
- 不阻塞用户过久
- 保持最终行为一致

### 3. abort-aware

整个流程几乎每个异步分支后都检查 `resolveIfAborted()`。

这保证：

- 用户已取消后不会再弹旧审批框
- 后到达的 classifier/hook/bridge 结果不会污染已终止的流程

这是典型的 async cancellation hygiene。

## 6.5 `interactiveHandler.ts`：多审批源竞态协调器

这是整个系统在并发与分布式交互上最精妙的部分之一。

它同时支持：

- 本地 UI 审批
- bridge 审批
- channel 审批
- hooks/background classifier 审批

而且要求：

- 只接受一个最终结果
- 其余结果无害化
- 能在任意一侧先响应时清理另一侧挂起状态

### `createResolveOnce()` 的价值

这是 ask 协调的关键同步原语。

其语义不是普通“只 resolve 一次”，而是：

- `claim()` 先占坑
- 后续异步回调先看自己是否已输掉竞态

这比简单 `if (!resolved)` 更强，因为它覆盖了：

- 检查与 resolve 之间的异步窗口
- await 期间的竞态

这是工业级异步仲裁设计，而不是普通前端状态管理技巧。

---

## 7. 决策优先级与安全不变量

## 7.1 决策顺序是安全策略本身

这套系统里最关键的不是“有没有规则”，而是**决策顺序严格编码了安全优先级**。

以 `hasPermissionsToUseToolInner()` 和 `checkWritePermissionForTool()` 为例，可以抽象成：

```text
显式 deny
  > 显式 ask
  > 工具级 deny/ask
  > 安全检查
  > 模式快速路径
  > allow
  > 默认 ask
```

如果顺序错一位，安全语义就会变化。

例如：

- 如果先 allow 再 safetyCheck，敏感文件可被宽规则绕过
- 如果先 bypass 再 tool.ask(rule)，内容级 ask 规则失效
- 如果先 acceptEdits 再 path safety，工作区内敏感文件会被静默修改

所以顺序是强约束，不是重构可随意改变的实现细节。

## 7.2 关键安全不变量

可提炼出几条架构级 invariants：

1. `deny` 优先于 `allow`
2. `safetyCheck` 优先于 mode fast path
3. `ask` 不是 deny 的文案分支，而是独立控制分支
4. headless 环境不能依赖 UI 审批
5. 自动分类器不能被 allow 规则短路
6. 应用层放行不应取消 OS 层隔离
7. 审批结果必须单写入、不可双生效

这几条不变量构成了整个系统稳定运行的基础。

---

## 8. 文件与路径安全子系统

## 8.1 为什么路径安全要独立成子系统

对很多 Agent 平台来说，文件权限常常被简化为：

- 是否在 cwd 下
- 是否有 allow list

Claude-code-open 明显认为这远远不够。

路径子系统要解决的是：

1. 路径语义归一化
2. symlink 绕过
3. OS 路径差异
4. Windows 特殊路径攻击面
5. UNC / 网络路径
6. 内部系统路径白名单
7. sandbox allowlist 与应用层 working dir 的语义对齐

因此它把这部分抽成独立子系统，而不是工具附属函数。

## 8.2 `matchingRuleForInput()` 的模式系统

文件类规则使用 `ignore` 风格模式系统，而不是 shell rule matcher。

其关键设计点是“pattern root”：

- `//path`：文件系统根
- `/path`：settings root
- `~/path`：home root
- 相对路径：null root，按 cwd 处理

这意味着同一条规则字符串的解释依赖 source。

从工程上看，这是一个很成熟的设计：

- 用户表达自然
- 不同配置源能保留自己的相对路径语义
- 最终都归一到统一匹配流程

## 8.3 `checkPathSafetyForAutoEdit()` 的威胁模型

这个函数本质上是一个 threat-model-driven validator。

它不是在问“路径格式是否正确”，而是在问：

- 这个路径是否可能作为权限绕过载体？

覆盖面包括：

- NTFS ADS
- 8.3 short names
- long path prefix
- trailing dots/spaces
- DOS device names
- `...` path confusion
- UNC
- Claude 配置目录
- shell 配置
- Git / IDE 配置
- symlink 目标链

这说明作者的安全设计不是抽象 ACL，而是明显带有 AppSec 实战经验。

## 8.4 `pathValidation.ts` 的接口价值

这个模块把文件权限校验提炼成更通用的路径接口：

- `isPathAllowed()`
- `validateGlobPattern()`
- `ResolvedPathCheckResult`

这使得：

- 工具不必都走完整 FileRead/FileWrite permission path
- 任何需要做路径前置检查的模块都能复用同一套安全判定

从架构上看，这是把“工具权限”进一步抽象成“资源操作权限”。

---

## 9. 模式系统与状态迁移

## 9.1 `PermissionMode` 不是 UI 状态，而是控制策略状态

模式系统承担的是全局策略变换职责。

主要模式：

- `default`
- `plan`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `auto`

其中最关键的不是模式枚举本身，而是：

- 不同模式会改变 `ask` 的解释方式
- 不同模式会改变工作区内文件写入语义
- 不同模式会影响是否允许 classifier 介入
- 模式切换本身会触发上下文重写

## 9.2 `transitionPermissionMode()` 的工程意义

这个函数是整套模式机的 central transition coordinator。

它统一处理：

- 计划模式进入/退出
- auto mode 激活/关闭
- dangerous rules strip / restore
- `prePlanMode` 记录和恢复

这意味着系统显式避免了“各处 set mode，各处补副作用”的散乱实现。

这是工业系统里非常典型、非常必要的 central transition point。

## 9.3 `stripDangerousPermissionsForAutoMode()`

这是本系统最值得注意的创新性工程设计之一。

其核心思想是：

> auto mode 的安全性不只取决于 classifier 本身，还取决于 classifier 之前的决策路径是否会被 allow rule 短路。

因此进入 auto 前要：

1. 扫描 allow rules
2. 识别会绕过 classifier 的危险规则
3. 把这些规则从 active context 中剥离
4. 在退出 auto 时恢复

这本质上是一个**mode-specific normalization transform**。

很多系统只在“执行时”做 classifier，但忽略了“进入 classifier 前的 allow graph”。

Claude-code-open 在这一点上明显更成熟。

## 9.4 killswitch 与 gate

系统支持：

- `checkAndDisableBypassPermissionsIfNeeded()`
- `checkAndDisableAutoModeIfNeeded()`

这说明权限控制不是完全本地自治的，而是支持：

- 远端策略动态失效
- 组织级 gate 控制
- 会话内实时踢出高风险模式

这是典型的 enterprise-grade governance 设计。

---

## 10. 自动审批链与算法控制

## 10.1 多种“自动 resolver”的串联结构

对 ask 结果，系统不是直接弹框，而是按环境选择以下 resolver：

1. hooks
2. local classifier
3. coordinator worker automated path
4. swarm worker -> leader relay
5. speculative classifier
6. interactive UI / bridge / channel

这实际上形成了一个 layered resolver pipeline。

抽象上可以写成：

```text
AskDecision
  -> AutomatedResolvers*
  -> HumanResolvers*
  -> FinalDecision
```

其中 AutomatedResolvers 是尽量减少人类中断的优化层，人类审批是最终兜底层。

## 10.2 auto 模式 classifier 与 bash allow classifier 的差异

系统中至少存在两类 classifier 角色：

1. `auto` 模式的 yolo/action classifier  
   处理“高层 ask 是否应被自动批准”

2. Bash allow classifier / speculative classifier  
   处理命令级 prompt-rule 匹配，减少本地 UI 打断

这说明 classifier 并不是单点模型，而是被嵌入到不同阶段，承担不同职责。

从 AI 系统设计角度，这是一种“多阶段机器审核”架构，而不是单个模型接管所有权限。

## 10.3 denial tracking

`DenialTrackingState` 用于记录：

- `consecutiveDenials`
- `totalDenials`

其设计价值在于：

1. 防止 classifier 不断拒绝但 agent 仍然重试
2. 把“被机器反复否决”识别成一个单独异常态
3. 在合适时机回退到人工审查或直接终止

这实际上是一个“Agent 安全熔断器”。

很多 Agent 平台会做 token budget、tool budget，但不会做 safety denial budget。这里是明显更成熟的一步。

---

## 11. 分布式审批与多通道控制

## 11.1 统一协议：`control_request` / `control_response`

远程权限请求没有走 ad hoc 消息格式，而是明确建模为 control protocol：

- `control_request`
- `control_response`
- `control_cancel_request`

这使得：

- 远程 UI
- bridge
- remote session

都能共享同一类审批协议。

这在工业设计上非常重要，因为权限请求本质上是控制面协议，不应与普通聊天消息混杂。

## 11.2 bridge callback 抽象

`BridgePermissionCallbacks` 把桥接审批抽象成：

- `sendRequest`
- `sendResponse`
- `cancelRequest`
- `onResponse`

这是一个典型的 transport abstraction。

意味着 interactive handler 不关心：

- WS 还是 HTTP
- remote UI 长什么样
- 远程侧如何展示权限请求

只关心 request/response/cancel 语义。

## 11.3 swarm worker -> leader 模型

swarm worker 场景下，权限系统采用：

```text
worker local classifier
  -> if unresolved
     send mailbox request to leader
  -> leader returns decision
  -> worker continues or aborts
```

这是一个非常典型的 distributed control-plane 模型：

- 执行平面分布式
- 审批平面集中式

对于多 agent 系统，这是一种很合理的治理方案。

## 11.4 channel approval

channel approval 的意义不在“能不能用 Telegram 审批”，而在于它证明了这套权限系统后端是可插拔的。

只要一个外部通道能满足：

- 接收请求
- 返回 allow/deny
- 可携带必要的 request id

它就能接入统一 ask pipeline。

这说明整个系统的 ask 后端并没有被 UI 框死。

---

## 12. 沙箱系统：从逻辑授权到 OS 执行约束

## 12.1 `sandbox-adapter.ts` 的真实角色

这个模块的角色不是“封装第三方库”，而是：

**把 Claude Code 高层权限语义编译成 sandbox-runtime 可执行约束。**

这是一层语义映射器。

其职责包括：

1. 解释 Claude 风格路径规则
2. 合并 settings、policy、permission rules
3. 下推为：
   - `allowWrite`
   - `denyWrite`
   - `allowRead`
   - `denyRead`
   - 网络 allow/deny 域名
4. 注入系统内建防护
5. 处理不同 settings source 的 root 语义

这相当于一个 policy compiler。

## 12.2 关键设计：双重防线

应用层允许与 sandbox 允许不是同一层含义：

- 应用层决定 Claude “是否应该发起该动作”
- sandbox 决定即使它发起了，进程“实际上能触达什么”

这意味着：

- 如果应用层漏掉一个边界，OS 侧仍可能拦住
- 如果 sandbox 配置过宽，应用层仍会先 ask/deny

这是典型纵深防御。

## 12.3 policy compiler 的工业价值

`convertToSandboxRuntimeConfig()` 的价值在于：

- 把高层语义映射到底层隔离，不需要手写双份策略
- 使用户配置在应用层与执行层保持一致
- 降低策略漂移风险

这是工业系统里非常关键的一点：

> 高层 policy 与底层 enforcement 如果各管各的，最终一定会漂移。

Claude-code-open 显然在努力避免这种漂移。

## 12.4 sandox write allowlist 与应用层 working dir 的协调

`pathValidation.ts` 中专门处理了：

- 工作区路径
- sandbox write allowlist

两者的优先级与交互语义。

例如：

- 在 working dir 内，不能因为 sandbox 默认包含 `.` 就绕过 `acceptEdits` 语义
- 在 working dir 外，如果 sandbox 明确允许某目录，则可减少无意义 prompt

这说明作者意识到“高层语义”和“低层可写目录”并不是完全同构的。

这类细节是工业级系统与演示级系统的重要区别。

---

## 13. 可观测性与治理设计

## 13.1 `permissionLogging.ts`

权限决策不是黑箱。所有 approve/reject 都走统一日志入口：

- analytics event
- OTel event
- code-edit counter
- `toolUseContext.toolDecisions`

这意味着系统对权限不是“只做控制”，而是“做可观测控制”。

### 工程价值

1. 可分析用户到底被什么打断
2. 可分析 classifier、hook、user 哪种批准路径最常见
3. 可分析代码编辑工具的风险行为
4. 可对组织策略调整提供数据支撑

## 13.2 tool decision persistence

`toolUseContext.toolDecisions` 会记录：

- source
- decision
- timestamp

这等于在工具执行上下文里附加了一层“权限审计轨迹”。

这为后续：

- 工具结果解释
- agent 自我修正
- 远程排障

提供了基础。

## 13.3 denial and notification

auto mode deny 不只是拒绝，它还会：

- 记入 auto mode denial
- 推送 UI notification
- 更新 denial tracking

这说明系统不是把 deny 当作静态结束，而是把 deny 看作后续治理与 UX 调整的重要信号。

---

## 14. 关键设计约束与重构红线

如果要演进这套系统，有几条约束几乎不能破坏：

## 14.1 不要破坏决策顺序

规则、mode、safety、allow 的顺序属于安全语义，不可随意重排。

## 14.2 不要把 ask 降级成 UI 分支

ask 是协议级状态，不是前端表现层状态。否则 remote/bridge/swarm/channel 无法共享。

## 14.3 不要把 `decisionReason` 退化为字符串

结构化 reason 是 explainability、telemetry、远程协议、模型反馈的共同基础。

## 14.4 不要把模式切换散落到各处

`transitionPermissionMode()` 这种 central transition point 必须保留，否则副作用会很快发散。

## 14.5 不要把 sandbox 当成“额外特性”

sandbox 是 enforcement layer，不是可有可无的体验增强功能。

## 14.6 不要取消 resolve-once 竞态控制

一旦多通道审批失去单写入保障，就会出现极难排查的双生效问题。

---

## 15. 关键创新性与工业级工程价值

从 AI Agent 平台设计角度，这套架构至少有 8 个值得借鉴的点。

## 15.1 `ask` 被正式建模为协议状态

这使人类审批成为系统控制面的一部分，而不是外部 UI hack。

## 15.2 classifier 前的规则剥离

`stripDangerousPermissionsForAutoMode()` 很有代表性。它不是“上个分类器”这么简单，而是保证分类器前的 control graph 不会绕过它。

这是非常成熟的 AI safety engineering 思维。

## 15.3 审批后端可插拔

本地 UI、bridge、remote、channel、swarm leader 都共享 ask 协议。

这使审批系统天然支持多终端、多会话和分布式控制。

## 15.4 结构化拒绝原因

`PermissionDecisionReason` 的设计非常适合：

- 解释性
- 观测
- 模型反馈
- 审计

这比简单错误字符串强得多。

## 15.5 安全熔断器

`denialTracking` 把“反复被分类器拒绝”识别为一种系统异常态，而不是普通拒绝。

这很适合高自主 Agent。

## 15.6 逻辑策略与 OS 隔离联动

把高层 permission rules 编译成 sandbox config，是非常实用的 policy-to-enforcement 闭环设计。

## 15.7 路径威胁模型非常完整

Windows/UNC/symlink/.claude/.git/IDE config 等边界都被明确考虑，明显超出普通 CLI 工具的安全成熟度。

## 15.8 异步竞态控制规范化

`createResolveOnce()` 虽然代码小，但它把多审批源的最终一致性问题压成了一个稳定原语，工程价值很高。

---

## 16. 对 AI Agent 平台设计的启示

如果把这套设计迁移到更广义的 AI Agent 平台，可以得到几条重要原则：

1. 不要把工具权限只做成 allow/deny ACL
   需要显式的人类审批状态。

2. 不要把自动审批和人工审批混成一个黑箱
   应该拆成 resolver pipeline。

3. 不要只在工具层做控制
   还要有资源层与 OS 层控制。

4. 不要让 explainability 只存在于日志文本
   需要结构化 reason schema。

5. 不要只做功能正确
   还要做并发正确、模式正确、治理正确。

6. 不要把安全视为静态配置
   gate、killswitch、remote policy、动态 mode rollback 都很重要。

---

## 17. 总结

Claude-code-open 的权限与安全架构，已经明显超出了“CLI 工具的权限确认机制”这个范畴。

它更像一个 Agent 平台级控制系统，具备以下特征：

- 有统一权限协议
- 有工具级与全局级双层决策
- 有显式模式状态机
- 有高保真路径安全模型
- 有多审批源分布式竞态协调
- 有自动审批与人工审批的组合控制
- 有逻辑授权到 OS 隔离的闭环
- 有完善的可观测与治理能力

如果从 AI 应用和算法系统设计角度评价，这套架构最大的价值不是“安全功能很多”，而是它把 Agent 权限问题系统化为了一个可演化的工程框架。

一句话总结：

**这不是一个 permission dialog system，而是一个面向本地 Agent OS 的、策略驱动的、可解释的、分布式可审批的、安全执行控制架构。**
