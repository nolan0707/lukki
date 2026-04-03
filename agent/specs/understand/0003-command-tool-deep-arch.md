# Claude-code-open 命令与工具架构深度解读（专家版）

## 1. 文档定位

本文面向 AI 应用架构师、Agent 系统设计者、算法平台工程师。

相较于 [0001-project-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0001-project-arch.md) 的全局架构综述，以及 [0003-command-tool-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0003-command-tool-arch.md) 的入门级解读，这份文档关注的是更深一层的问题：

1. 命令与工具在系统中的真实抽象边界是什么？
2. 这套架构如何把“人类入口”“模型执行单元”“外部扩展能力”统一进同一个 Agent runtime？
3. 为什么这里不是简单的 function-call 封装，而是一套带权限、调度、缓存、恢复、观测、扩展治理的工业级执行平台？
4. 哪些模块是协议层，哪些是编排层，哪些是控制面，哪些是数据面？
5. 这里有哪些值得 AI 应用系统借鉴的工程创新？

本文基于以下源码深读整理：

- `vendor/Claude-code-open/src/types/command.ts`
- `vendor/Claude-code-open/src/commands.ts`
- `vendor/Claude-code-open/src/Tool.ts`
- `vendor/Claude-code-open/src/tools.ts`
- `vendor/Claude-code-open/src/constants/tools.ts`
- `vendor/Claude-code-open/src/query.ts`
- `vendor/Claude-code-open/src/QueryEngine.ts`
- `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts`
- `vendor/Claude-code-open/src/services/tools/toolExecution.ts`
- `vendor/Claude-code-open/src/services/tools/StreamingToolExecutor.ts`
- `vendor/Claude-code-open/src/services/tools/toolHooks.ts`
- `vendor/Claude-code-open/src/utils/processUserInput/processUserInput.ts`
- `vendor/Claude-code-open/src/utils/processUserInput/processSlashCommand.tsx`
- `vendor/Claude-code-open/src/utils/permissions/permissions.ts`
- `vendor/Claude-code-open/src/services/mcp/client.ts`
- `vendor/Claude-code-open/src/services/mcp/types.ts`
- `vendor/Claude-code-open/src/services/mcp/mcpStringUtils.ts`
- `vendor/Claude-code-open/src/services/mcp/utils.ts`
- `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx`
- `vendor/Claude-code-open/src/tools/FileReadTool/FileReadTool.ts`
- `vendor/Claude-code-open/src/tools/FileEditTool/FileEditTool.ts`
- `vendor/Claude-code-open/src/tools/SkillTool/SkillTool.ts`
- `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts`
- `vendor/Claude-code-open/src/tools/AgentTool/agentToolUtils.ts`

---

## 2. 执行摘要

如果从专家视角给这套架构下定义，它不是“CLI 里的命令系统 + 一些 function calling tools”，而是一个分层清晰的 **Agent 能力虚拟机**：

```text
控制面
  Command
  PermissionMode / Rule
  AppState / Runtime Config
  Plugin / MCP / Skill 注册与治理

数据面
  Tool
  ToolUseContext
  query loop
  tool execution pipeline

扩展面
  PromptCommand
  SkillTool
  MCP Tool / Prompt / Resource
  AgentTool
```

它解决的核心不是“让模型调用函数”，而是下面这组更接近工业生产的问题：

- 如何让能力可组合，又可治理？
- 如何让模型拥有足够强的执行力，但不会失控？
- 如何让外部生态能力接入时，不破坏系统一致性？
- 如何让会话在长时间、多轮、子 agent、MCP、权限审批、hooks、fallback、resume 场景下仍保持可恢复和可观测？

这套设计最值得借鉴的地方有 6 个：

1. **双入口架构**：Command 面向用户，Tool 面向模型，职责边界清晰。
2. **统一协议化扩展**：Plugin、Skill、MCP 最终都被压缩进 Command/Tool 两个统一抽象。
3. **中间件化工具执行管线**：校验、权限、hooks、遥测、结果映射不是散落在工具实现中，而是有统一执行框架。
4. **能力暴露与能力执行分离**：工具的“是否让模型看到”与“是否允许执行”是两层不同控制点。
5. **上下文/缓存稳定性优先**：工具集合排序、读取去重、路径回填、tool_result 裁剪都围绕 prompt-cache 与 transcript-stability 设计。
6. **递归式 Agent Runtime**：子 agent 不绕开主系统，而是复用同一套命令/工具/查询执行机制。

---

## 3. 架构问题的正确切法

要理解命令与工具架构，不能按文件夹切，而要按 4 个核心问题切：

1. 谁来描述能力？
2. 谁来决定暴露哪些能力？
3. 谁来调度和执行能力？
4. 谁来约束能力的安全边界？

对应到代码：

```text
能力描述
  types/command.ts
  Tool.ts

能力暴露
  commands.ts
  tools.ts
  services/mcp/client.ts

能力执行
  processUserInput/*
  QueryEngine.ts
  query.ts
  services/tools/*

能力约束
  utils/permissions/*
  toolHooks.ts
  constants/tools.ts
```

这 4 个问题在系统中被刻意拆开，这是成熟平台化设计的典型特征。

很多系统的 function calling 框架会把“能力描述”“能力暴露”“能力执行”“能力审批”糊在一起，短期看简单，长期必然失控。Claude-code-open 则把这几层拆开，使得：

- 能力可复用
- 执行可观测
- 风险可治理
- 扩展可插拔

---

## 4. 顶层抽象：Command 与 Tool 的正交分工

## 4.1 Command 不是 Tool 的别名

在很多 Agent 框架里，开发者会把 `/review`、`/plan`、`/mcp` 这种人类输入入口也看成工具，导致系统抽象混乱。这里并没有这么做。

### `Command` 的职责

`Command` 表示“用户显式触发的入口”。

它解决的是：

- 可发现性
- 交互意图切换
- 模式切换
- 本地 UI 操作
- prompt 模板化入口

### `Tool` 的职责

`Tool` 表示“模型推理过程中可调用的执行单元”。

它解决的是：

- 可执行能力
- 模型协议暴露
- 输入输出约束
- 权限审批
- 运行期状态交互

这两者是正交的，而不是包含关系。

## 4.2 为什么要双入口

这是这套架构的第一个关键设计。

如果只保留一种统一入口，会出现两个问题：

### 问题 1：用户触发和模型触发的控制模式完全不同

- 用户入口强调 discoverability 和 explicit intent。
- 模型入口强调 schema correctness 和 iterative execution。

### 问题 2：两者的安全模型完全不同

- 用户输入 `/plan` 是显式授权行为。
- 模型发起 `FileEditTool` 是代理行为，必须经过额外控制。

因此 Command/Tool 分离不是语法风格问题，而是 **控制流与信任边界的分离**。

## 4.3 PromptCommand 是命令与工具之间的桥梁

`PromptCommand` 是整个架构中最有意思的抽象。

它同时具备：

- 命令属性：用户可通过 `/xxx` 触发
- Prompt 生成属性：`getPromptForCommand()`
- Tool 可包装属性：可被 `SkillTool` 重新暴露给模型

这意味着：

```text
PromptCommand
  = 人类可触发的 prompt 入口
  = 模型可间接调用的能力模板
```

这正是 Skill 能够同时服务于用户和模型的根本原因。

---

## 5. 命令协议设计

## 5.1 `Command` 联合类型的工程意义

`src/types/command.ts` 中：

```ts
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

这不是普通的类型体操，而是在做一件很重要的事：

**把“统一注册接口”与“异质执行语义”同时保留下来。**

### 统一部分 `CommandBase`

包括：

- `name`
- `description`
- `aliases`
- `availability`
- `source`
- `loadedFrom`
- `disableModelInvocation`
- `userInvocable`
- `isEnabled`
- `isHidden`
- `userFacingName`

这些字段主要服务于：

- 注册
- 展示
- 筛选
- 权限与来源标记

### 差异化部分

- `PromptCommand` 负责 prompt expansion
- `LocalCommand` 负责本地纯逻辑执行
- `LocalJSXCommand` 负责本地 UI 交互执行

这种设计避免了一个常见反模式：为了“统一”，强行要求所有命令实现同一个 `run()`，导致类型宽泛、调用方满地 if/else。

## 5.2 `availability` 与 `isEnabled` 的二层可见性治理

`types/command.ts` 把“这个命令对谁可见”和“这个命令当前是否启用”拆成两层：

- `availability`
- `isEnabled()`

`commands.ts` 中的 `meetsAvailabilityRequirement()` 再负责运行时筛选。

这个分层很重要：

### `availability`

回答的是：

- 这个命令是否适用于当前认证/服务环境？

例如：

- 只对 `claude-ai`
- 只对 `console`

这属于 **能力资格约束**。

### `isEnabled()`

回答的是：

- 在当前环境、feature flag、平台条件下是否打开？

这属于 **能力发布约束**。

这两层被分开之后，产品治理、灰度发布、环境隔离都会更容易。

## 5.3 `source` / `loadedFrom` 的双重来源标记

这里有两个经常容易混淆的字段：

- `source`
- `loadedFrom`

它们不完全等价。

### `source`

更偏“业务来源语义”：

- builtin
- plugin
- bundled
- mcp

### `loadedFrom`

更偏“装载通道/归属语义”：

- skills
- plugin
- bundled
- mcp
- commands_DEPRECATED

这两个字段一起使用，解决的是平台生态治理中的一个关键问题：

**能力从哪里来，不只是显示问题，而是信任等级、可见性、模型可调用性、统计归因的基础元数据。**

例如：

- `SkillTool` 在筛选 skill 时就依赖这些字段。
- telemetry 也会根据 pluginInfo / source 打标签。

---

## 6. 命令聚合架构：`commands.ts`

## 6.1 它本质上是命令目录服务，而不是注册表常量

`commands.ts` 表面上像一个大数组，实际上它更像一个 **命令目录服务**。

原因有三：

1. 它不只返回内置命令，还动态合并 skills / plugins / workflows / MCP prompts。
2. 它会根据用户状态、feature flag、动态发现的 skills 做运行时筛选。
3. 它会为不同消费方提供不同视角的命令子集。

因此 `getCommands(cwd)` 的语义不是“拿到全部命令”，而是：

**拿到当前运行时上下文中，对当前消费方可见的命令集合。**

## 6.2 多级来源合并流程

核心流程：

```text
COMMANDS()
  -> getSkills()
  -> getPluginCommands()
  -> getWorkflowCommands()
  -> loadAllCommands(cwd)
  -> getCommands(cwd)
```

其中：

- `COMMANDS()` 是内置命令基础集
- `getSkills()` 又会继续读取 skill dir commands、plugin skills、bundled skills、builtin plugin skills
- `loadAllCommands()` 把各来源做一次粗合并
- `getCommands()` 再做 availability / isEnabled / dynamic skills 注入

这是非常典型的“两阶段聚合”：

- 第一阶段：能力装载
- 第二阶段：运行时过滤

## 6.3 `getSkillToolCommands()` 与 `getSlashCommandToolSkills()`

这是命令系统中非常关键的两个投影函数。

### `getSkillToolCommands()`

语义：

- 为 `SkillTool` 提供“模型可间接调用的 prompt commands 子集”

筛选条件：

- `cmd.type === 'prompt'`
- `!cmd.disableModelInvocation`
- `cmd.source !== 'builtin'`
- 同时需要满足一套描述/来源约束

### `getSlashCommandToolSkills()`

语义：

- 为 slash command 层提供“技能型命令子集”

这两个函数说明：

**命令系统不只有一个全局视图，而是会被不同模块按需投影。**

这是典型的平台目录化设计，而不是简单数组遍历。

## 6.4 动态 skills 插入位置的工程含义

`getCommands()` 会把运行期发现的 `dynamicSkills` 插入“plugin skills 后、builtin commands 前”。

这不是无关紧要的顺序细节，而是 UI/优先级治理的一部分：

- 运行时发现的技能应该比系统命令更靠前
- 但不应该破坏已有 plugin / skill 生态的分组语义

这种“插入位置也是协议”的设计，很像成熟 IDE/编辑器对 action palette 的治理方式。

---

## 7. 工具协议设计：`Tool.ts`

## 7.1 `Tool` 是多面体接口，不是函数签名

`Tool` 抽象同时携带 5 类语义：

1. 模型协议语义
2. 运行时执行语义
3. UI 展示语义
4. 安全治理语义
5. 观测与缓存稳定性语义

这也是为什么 `Tool` 接口字段很多，但并不臃肿，因为每个字段都对某个系统层负责。

## 7.2 `Tool` 接口按职责可分为 8 组

### A. 身份与命名

- `name`
- `aliases`
- `userFacingName()`
- `mcpInfo`

### B. 模型描述

- `description()`
- `prompt()`
- `inputSchema`
- `inputJSONSchema`
- `outputSchema`

### C. 调度控制

- `isConcurrencySafe()`
- `isReadOnly()`
- `isDestructive()`
- `interruptBehavior()`

### D. 安全与权限

- `validateInput()`
- `checkPermissions()`
- `preparePermissionMatcher()`
- `getPath()`

### E. 执行

- `call()`
- `inputsEquivalent()`
- `toAutoClassifierInput()`

### F. 结果映射

- `mapToolResultToToolResultBlockParam()`
- `isResultTruncated()`
- `extractSearchText()`

### G. UI 呈现

- `renderToolUseMessage()`
- `renderToolUseProgressMessage()`
- `renderToolResultMessage()`
- `renderToolUseRejectedMessage()`
- `renderToolUseErrorMessage()`
- `renderGroupedToolUse()`

### H. 暴露策略

- `isEnabled()`
- `isOpenWorld()`
- `shouldDefer`
- `alwaysLoad`
- `isMcp`
- `isLsp`

这套接口设计说明它不是“工具执行 API”，而是 **工具运行时描述对象**。

## 7.3 `ToolUseContext` 是 runtime capability object

`ToolUseContext` 的最佳理解方式不是“参数包”，而是：

**一次工具调用的 capability object。**

它注入的不是普通数据，而是运行期能力：

- 状态读取能力：`getAppState`
- 状态修改能力：`setAppState`
- UI 更新能力：`setToolJSX`
- 生命周期控制能力：`abortController`
- 消息上下文访问能力：`messages`
- 会话基础设施访问能力：`setAppStateForTasks`
- 内容预算、文件历史、归因等能力

这种对象设计非常像 capability-based runtime，而不是传统 service locator。

## 7.4 `ToolResult` 为什么允许 `newMessages` 和 `contextModifier`

这是这套工具架构与简单 function calling 最大的差异之一。

### `newMessages`

允许工具把额外消息注入消息流。

这使工具不只是返回结构化值，还能：

- 扩展上下文
- 插入附件
- 追加系统提示
- 在 forked/skill/agent 场景下回流结果

### `contextModifier`

允许工具执行后修改未来工具调用的上下文。

这意味着工具不仅是“纯执行节点”，还是 **状态转移节点**。

对于复杂 Agent runtime，这一点非常关键。

## 7.5 `buildTool()` 的深层意义

`buildTool()` 并不只是减少模板代码。

它真正做的是：

- 为所有工具提供统一的安全默认值
- 让工具实现者只关注差异化逻辑
- 把工具协议的“可选实现”收束为“完整对象”

尤其是这些默认值：

- `isConcurrencySafe -> false`
- `isReadOnly -> false`
- `checkPermissions -> allow`

这体现了非常明确的设计态度：

- 调度默认保守
- 并发默认关闭
- 权限机制允许由统一上层覆盖

这是成熟平台设计常见的做法：**默认保守，局部放开。**

---

## 8. 工具装配架构：`tools.ts`

## 8.1 `getAllBaseTools()` 是系统能力真源

`getAllBaseTools()` 的注释明确说明它是 source of truth。

这意味着两件事：

1. 它不只是给运行时使用，还影响 system prompt 缓存稳定性。
2. 它的顺序、存在性、feature-gating 都具有协议含义。

在很多项目里，“工具列表”只是运行时一个数组。这里不是。这里的工具列表是：

- prompt surface 的一部分
- cache key 的一部分
- feature gate 的一部分

因此它必须稳定、可推导、集中治理。

## 8.2 能力暴露与能力执行分离

`getTools(permissionContext)` 会在模型看见工具之前做过滤：

- simple mode 过滤
- deny rule 过滤
- REPL 模式隐藏 primitive tools
- `isEnabled()` 过滤

注意，这里只是“暴露控制”，不是“执行控制”。

这很重要，因为：

- 某个工具可以被系统拥有，但不暴露给当前模型
- 某个工具即使暴露了，执行时仍要再次过权限判定

这说明系统明确区分了：

### 暴露权限

模型是否知道这个工具存在

### 执行权限

模型即使知道它，是否能真正调用

这在安全系统里是非常重要的分层。

## 8.3 `assembleToolPool()` 的 cache-stability 设计

这里的关键不是合并，而是合并后的排序与去重。

它做了几件事情：

1. 内置工具先过滤
2. MCP 工具按 deny rules 再过滤
3. built-in tools 排序
4. allowed MCP tools 排序
5. 拼接后 `uniqBy(name)`，让内置工具优先

注释明确指出排序不是 UX 需求，而是为了：

- 保持 built-in tools 作为连续前缀
- 避免 MCP tools 插进 built-in tools 中间
- 降低系统 prompt cache key 抖动

这是一种非常少见但非常高级的 LLM 工程设计：

**把“工具集合顺序稳定性”显式提升为缓存策略的一部分。**

---

## 9. 工具执行管线

## 9.1 工具执行不是一次函数调用，而是一条流水线

从 `query.ts` 到 `toolOrchestration.ts` 再到 `toolExecution.ts`，完整执行链是：

```text
assistant stream
  -> 收集 tool_use
  -> 批次划分
  -> 单工具执行
     -> schema validation
     -> semantic validation
     -> pre-tool hooks
     -> permission decision
     -> telemetry/span
     -> tool.call
     -> result mapping
     -> post-tool hooks
     -> message/context updates
  -> tool_result 回填
  -> 下一轮 query
```

这是一条标准化的数据面流水线。

## 9.2 `toolOrchestration.ts`：轻量调度器

`partitionToolCalls()` 是关键算法。

它把 `tool_use` 列表划成若干批，每批要么：

- 是单个非并发安全工具
- 要么是连续的并发安全工具序列

这类算法的特征是：

- 不做全局重排
- 不做复杂依赖分析
- 只利用工具显式暴露的并发安全信息

工业上这类设计很常见，因为：

- 全局 DAG 调度理论上更优
- 但在 Agent 场景里可解释性差，错误面大
- 局部保守的批次划分足以获得大部分收益

这里的思想可以总结为：

**顺序一致性优先于最大吞吐，吞吐优化只在已知安全区域进行。**

## 9.3 `StreamingToolExecutor`：流式工具执行器

这是这套架构的一个显著工程亮点。

传统模式：

- 模型先完整输出 assistant
- 再执行所有工具

这里增加了 streaming path：

- 工具在 `tool_use` block 流式到达时即可进入队列
- 并发安全工具可以尽早执行
- 结果按原始到达顺序有序产出

### 它解决的是什么问题

它解决的是 agentic latency，而不是单次 API latency。

当模型边生成边吐出多个工具调用时，系统不必等全部输出完成后再执行，而是：

- 先启动能启动的工具
- 并用 `pendingProgress`、`results`、`contextModifiers` 做局部缓存

### 关键约束

它同时保持了两个性质：

1. **执行可并发**
2. **结果可按到达顺序发射**

这是通过 `TrackedTool.status`、`pendingProgress`、`results`、`canExecuteTool()` 共同实现的。

### 工程亮点

- 有 sibling abort controller
- 对 streaming fallback 有 discard 逻辑
- 对 interrupt behavior 有 cancel/block 语义
- 对 Bash tool error 有并行 sibling kill 机制

这已经非常接近成熟 workflow engine 的执行器设计。

## 9.4 `toolExecution.ts`：中间件化单工具执行器

`runToolUse()` 真正的价值不是“执行工具”，而是：

**把一次工具调用标准化为一组中间件阶段。**

关键阶段如下。

### 阶段 1：工具解析

- 从当前可见工具集中找
- 找不到时尝试 alias / deprecated fallback

这保证了 transcript/resume/backward-compatibility。

### 阶段 2：schema validation

- `tool.inputSchema.safeParse`

如果失败，会额外检查 deferred tool schema 是否根本没发给模型，并给出 `ToolSearch` 修复提示。

这点非常高级，因为它不是简单报错，而是把错误映射回“工具发现机制”的根因。

### 阶段 3：semantic validation

- `tool.validateInput()`

这层负责工具语义验证，而不是结构验证。

### 阶段 4：pre-tool hooks

- `runPreToolUseHooks()`

这层可以：

- 改写输入
- 直接做权限决策
- 附加上下文
- 阻止继续执行

### 阶段 5：permission resolution

- `resolveHookPermissionDecision()`
- `canUseTool()`

这是统一权限决策阶段。

### 阶段 6：tool.call

只在前面所有阶段都通过后，才真正触发工具逻辑。

### 阶段 7：tool result mapping

- `mapToolResultToToolResultBlockParam`
- `processToolResultBlock`

这里会做：

- 大输出持久化
- 结构化输出到 Claude-compatible `tool_result`
- 结果裁剪与持久化记录

### 阶段 8：post-tool hooks

- `runPostToolUseHooks()`
- `runPostToolUseFailureHooks()`

这使工具调用结束后仍能被平台层介入。

这个设计本质上把工具执行变成了：

```text
input -> middleware pipeline -> effect -> normalized result
```

它不是 ad-hoc 逻辑，而是统一执行框架。

---

## 10. 权限架构：不是一个 if，而是一套决策系统

## 10.1 权限有三个控制点

### 控制点 1：工具可见性

`tools.ts` 中 deny rules 会把工具直接从可见列表剔除。

### 控制点 2：工具语义级权限

每个工具的 `checkPermissions()` 决定该工具如何参与权限体系。

### 控制点 3：统一权限决策

`hasPermissionsToUseTool()` / `hasPermissionsToUseToolInner()` 把规则、模式、hooks、classifier、UI prompt 串起来。

这三层叠加在一起，说明权限不是局部实现，而是平台能力。

## 10.2 权限对象模型

`ToolPermissionContext` 里包含：

- `mode`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `additionalWorkingDirectories`
- `isBypassPermissionsModeAvailable`
- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`
- `prePlanMode`

这说明权限上下文不是简单布尔开关，而是一个小型策略机。

## 10.3 规则模型的工业化设计

`permissions.ts` 中有几个关键点：

### 工具级规则匹配

- `toolMatchesRule()`
- `getDenyRuleForTool()`
- `toolAlwaysAllowedRule()`

### 内容级规则匹配

- `getRuleByContentsForTool()`
- `preparePermissionMatcher()`

后者很重要。它说明权限规则不仅能匹配工具名，还能匹配工具内容。

例如 Bash：

- `Bash(git *)`
- `Bash(prefix:*)`

这使权限系统从“能力级控制”升级为“调用级控制”。

## 10.4 权限决策来源是可解释的

`PermissionDecisionReason` 会记录决策来自：

- rule
- hook
- classifier
- mode
- workingDir
- permissionPromptTool
- sandboxOverride

这使系统具备很强的 explainability。

很多工具平台只能告诉你“拒绝了”。这里可以告诉你“因为哪条规则、哪个 hook、哪个 classifier、哪个 mode 拒绝了”。

对于工业系统，这是非常关键的运维能力。

## 10.5 headless agent 也不是直接 auto-deny

`runPermissionRequestHooksForHeadlessAgent()` 表明：

- 在无法弹权限框的 headless/async agent 中
- 仍然会先给 hooks 一次介入机会
- hooks 可允许或拒绝
- 最后才 fallback 到 auto-deny

这说明权限系统不仅兼容 REPL 交互，也兼容后台 agent。

这是一种平台化兼容设计，而不是只针对前台终端体验。

---

## 11. Hooks 架构：工具调用的可编程中间层

## 11.1 hooks 的位置非常关键

工具 hooks 并不是零散附加逻辑，而是被放在：

- 工具执行前
- 工具执行失败后
- 工具执行成功后

三个位置。

对应：

- PreToolUse
- PostToolUse
- PostToolUseFailure

## 11.2 hook 可以做什么

从 `toolHooks.ts` 看，hook 能做的事情包括：

- 产出 progress / attachment message
- 追加上下文
- 阻止后续继续
- 修改 MCP tool output
- 触发 blocking error
- 参与 permission decision

这意味着 hooks 不是“日志监听器”，而是真正的执行链扩展点。

## 11.3 为什么这是工业级设计

因为这让平台方可以在不侵入具体工具实现的情况下，注入：

- 安全策略
- 合规策略
- 诊断逻辑
- 额外观测
- 上下文增强

这非常像 web 框架里的 middleware，或者 service mesh 的 sidecar 语义。

对大型 AI 平台，这是关键能力。

---

## 12. 输入处理架构：命令如何进入运行时

## 12.1 `processUserInput` 不是纯 parser，而是输入路由器

`processUserInput()` 的作用不是把文本变成消息那么简单。

它实际上负责：

1. 输入类型归一化
2. slash command 路由
3. 文本 prompt 处理
4. pasted content / attachment 注入
5. user prompt submit hooks
6. 返回“是否需要进入 query”这一控制决策

因此它更像一个 **input router**。

## 12.2 `ProcessUserInputBaseResult` 是控制面数据结构

它不是普通返回值，而是一个“下一步怎么执行”的描述：

- `messages`
- `shouldQuery`
- `allowedTools`
- `model`
- `effort`
- `resultText`
- `nextInput`
- `submitNextInput`

这个结构很重要，因为它让输入处理层与 query 执行层解耦：

- 输入层只描述应该发生什么
- `QueryEngine` 决定是否真的进入模型执行

## 12.3 `processSlashCommand.tsx` 的关键设计

这个模块体现出命令系统并不只是用户 UI 菜单，而是可以直接进入 Agent runtime。

它支持几类命令路径：

### 本地命令路径

- 直接执行 local / local-jsx

### prompt command 路径

- 展开 prompt
- 生成消息
- 回到正常 query 执行

### forked slash command 路径

- 对 `context: 'fork'` 的 prompt command
- 直接生成一个 subagent 执行流程

这点很关键，因为它说明：

**slash command 并不是 query loop 外的旁路，而是 query loop 的上游控制器。**

## 12.4 `executeForkedSlashCommand()` 的创新点

这里有一个非常工业化的设计：

- assistant mode 下，forked slash command 可以 fire-and-forget
- 后台 subagent 结果会重新作为 hidden meta prompt 回到主队列

这相当于把“命令触发的后台任务”重新编织进主会话控制流。

这是一种很强的 async orchestration 设计，值得注意。

---

## 13. 查询层如何消费工具系统

## 13.1 `QueryEngine` 的角色

`QueryEngine` 不是工具执行器，而是会话级 orchestrator。

在命令与工具架构中的职责是：

- 接收 `processUserInput()` 的结果
- 组装本轮 `ToolUseContext`
- 注入 `tools`、`commands`、`mcpClients`、`agentDefinitions`
- 包装 `canUseTool`
- 调用 `query()`

它是控制面与执行面之间的接缝层。

## 13.2 `query.ts` 的角色

`query.ts` 才是单轮执行内核。

它负责：

- 维护消息状态机
- 管理 compaction / collapse / recovery
- 流式拉取模型输出
- 收集 `tool_use`
- 调度工具执行
- 把 `tool_result` 回填成新的用户消息
- 直到本轮结束

因此从系统结构上看：

```text
processUserInput
  -> QueryEngine
  -> query
  -> tool pipeline
```

这是典型的入口路由、会话编排、回路执行三级分工。

---

## 14. MCP 接入架构：外部能力如何协议化

## 14.1 MCP 在系统中的真实定位

MCP 不是“远程工具调用插件”，而是一个外部能力总线。

它带来三类外部能力：

- tools
- prompts
- resources

系统并没有强行把这三种东西揉成一个统一对象，而是分别适配到不同本地抽象：

- tools -> `Tool`
- prompts -> `Command`
- resources -> resource tools

这是一种非常干净的设计。

## 14.2 `fetchToolsForClient()`：MCP tool 适配器

这个函数做的事情极其关键：

1. 向 MCP server 请求 `tools/list`
2. 清洗返回值
3. 生成 fully-qualified name
4. 基于 `MCPTool` 模板生成本地 `Tool` 对象
5. 把 server annotations 翻译成：
   - `isReadOnly`
   - `isConcurrencySafe`
   - `isDestructive`
   - `isOpenWorld`
6. 注入 `inputJSONSchema`
7. 构造 `checkPermissions()` 和 `call()`

它本质上是一个 **远程能力到本地运行时协议的编译器**。

## 14.3 MCP 命名空间设计

`mcpStringUtils.ts` 里最核心的是：

- `buildMcpToolName(serverName, toolName)`
- `getToolNameForPermissionCheck()`
- `mcpInfoFromString()`

这里使用：

```text
mcp__<normalized_server>__<normalized_tool>
```

这是非常关键的工业设计，原因有三：

1. 避免和本地 builtin tools 冲突
2. 让 permission rules 可稳定匹配
3. 保证 transcript 和 telemetry 中的外部能力可识别

此外，skip-prefix mode 又允许 SDK MCP 工具覆盖 builtin 名称，这说明系统在“隔离性”和“替代性”之间做了高级兼容。

## 14.4 MCP resource 为什么是显式工具

`ListMcpResourcesTool` 和 `ReadMcpResourceTool` 被单独建模，而不是把 `resources/read` 也纳入普通 `MCPTool`。

这是合理的，因为 resource 的交互范式不同：

- 先 discover
- 再按 URI fetch

把它做成显式工具后，模型会更容易理解资源访问流程，而且权限、UI、结果裁剪也更容易控制。

---

## 15. Skill 与 Agent：两种高阶复合能力

## 15.1 `SkillTool` 的本质

`SkillTool` 不是普通工具，而是一个 **命令虚拟化工具**。

它的工作是：

1. 从命令系统中筛出 prompt-based commands
2. 校验 skill 的可调用性
3. 按定义决定 inline 还是 fork
4. 把结果重新注入当前会话

因此 SkillTool 相当于在工具层之上，又引入了一层“prompt-program execution”。

## 15.2 inline vs fork：上下文边界控制

这套设计最有价值的地方是 skill execution boundary 可配置：

### inline

- 技能内容直接展开到当前会话
- 成本低
- 上下文共享
- 污染主上下文的风险也更高

### fork

- 技能在独立 subagent 里运行
- 有自己的 token budget 和执行上下文
- 对主线程更安全

这本质上是在给 Prompt-based capability 增加“上下文隔离级别”。

从 AI 系统设计角度看，这是非常值得借鉴的。

## 15.3 `AgentTool`：系统是递归的

`AgentTool` / `runAgent.ts` 展示了这套系统真正成熟的一面：

- 子 agent 没有自己的一套孤立执行器
- 它复用同一个 query/tool framework
- 只是在工具集、MCP 客户端、上下文、权限模式上做了特化

这意味着：

**主 agent 与子 agent 是同构的，只是 capability set 不同。**

这是非常漂亮的 runtime 设计。

## 15.4 agent tool resolution 体现了平台治理

`agentToolUtils.ts` 中：

- `filterToolsForAgent()`
- `resolveAgentTools()`

非常关键。

这里通过：

- `ALL_AGENT_DISALLOWED_TOOLS`
- `ASYNC_AGENT_ALLOWED_TOOLS`
- `CUSTOM_AGENT_DISALLOWED_TOOLS`
- `IN_PROCESS_TEAMMATE_ALLOWED_TOOLS`

对 agent 可用工具做了白名单/黑名单治理。

这意味着子 agent 不是拿到主线程的完整能力集，而是拿到 **受约束的 capability subset**。

这就是多 agent 系统能够工业化运行的基础。

---

## 16. 关键设计约束

下面列出这套架构中最重要的设计约束，这些约束直接塑造了代码形态。

## 16.1 Prompt Cache Stability

受影响模块：

- `tools.ts`
- `query.ts`
- `toolExecution.ts`
- `FileReadTool`

体现：

- 工具集合排序稳定
- 内置工具前缀连续
- file path 回填不能污染 transcript
- 重复读取去重
- 大输出持久化而不是直接回填上下文

这是典型的 LLM serving 成本约束反向塑造架构。

## 16.2 Transcript Determinism

受影响模块：

- `toolExecution.ts`
- `processSlashCommand.tsx`
- MCP 命名与结果映射

体现：

- alias fallback
- 回填 observable input 但不污染实际 call input
- 恢复原始 file_path，避免 VCR fixture hash 改变
- 结果 message 使用稳定格式

这是为了 resume、回放、测试 fixture、sidechain transcript 一致性服务。

## 16.3 Safety Before Convenience

受影响模块：

- `FileEditTool`
- `permissions.ts`
- `BashTool`
- `tools.ts`

体现：

- 默认不并发
- 写前必须 read
- 文件外部修改需重读
- deny rule 在暴露层就生效
- headless agent 没 UI 时也不能绕过权限

## 16.4 Capability Subsetting

受影响模块：

- `agentToolUtils.ts`
- `constants/tools.ts`
- `tools.ts`

体现：

- 主线程、subagent、async agent、in-process teammate 工具集不同
- REPL 模式下 primitive tools 隐藏
- simple mode 下只暴露最小工具集

这说明系统能力不是静态全集，而是上下文化的能力投影。

## 16.5 Extensibility Without Trusting Extensions

受影响模块：

- `commands.ts`
- `services/mcp/client.ts`
- `SkillTool.ts`
- 权限系统

体现：

- Plugin / MCP / Skill 可扩展
- 但都必须转译到本地统一协议
- 且再经过本地权限、过滤、可见性治理

这是一种“开放接入，封闭执行”的平台原则。

---

## 17. 关键创新点与工业级工程亮点

下面总结这套架构最值得借鉴的地方。

## 17.1 把 Command 和 Tool 做成两级入口，而不是一个万能 action

这让：

- 用户交互层
- 模型执行层

可以独立演化。

这是极少数 Agent CLI 框架会认真做的事情。

## 17.2 PromptCommand + SkillTool 的双向桥接

这相当于把“prompt 模板”提升成可组合能力对象：

- 人能调用
- 模型也能调用
- 还能配置 inline/fork

这是 AI 原生应用里很强的模式。

## 17.3 StreamingToolExecutor 的低延迟 Agent 编排

这不是简单并发，而是：

- 流式接收
- 即时调度
- 顺序产出
- 支持 fallback/discard/abort

在 Agent latency 和 correctness 之间做了高质量平衡。

## 17.4 权限系统的分层决策模型

这里把：

- rule
- mode
- hook
- classifier
- interactive prompt

统一进一个可解释权限体系，属于非常成熟的 agent safety engineering。

## 17.5 把 cache stability / transcript determinism 作为一等设计目标

这点在大多数产品代码里不会被系统性设计，而这里只要看：

- 工具排序
- 读取去重
- path backfill 保护
- result persistence

就能看出这是一套面向长生命周期 Agent 运行时的设计。

## 17.6 子 agent 与主 agent 同构

这是架构成熟度非常高的体现。

很多多 agent 系统会为子 agent 另写一套简化执行框架，导致行为不一致。这里没有。这里让子 agent 复用同一套 query/tool pipeline，只是在 capability 和 context 上受限。

这显著降低了系统复杂度和一致性风险。

---

## 18. 对 AI 应用系统设计的启发

如果把 Claude-code-open 的命令与工具架构抽象成通用模式，可以得到如下设计原则。

## 18.1 不要把“用户入口”和“模型入口”混成一层

应分成：

- human command plane
- model tool plane

## 18.2 工具必须是协议对象，而不是裸函数

至少需要显式建模：

- schema
- permission
- concurrency
- interruptibility
- result mapping
- observability

## 18.3 扩展能力必须先本地协议化，再接入执行链

不要让 plugin / MCP / remote extension 直接进执行层。

## 18.4 工具执行必须经过统一中间件流水线

不要让每个工具自己各管各的权限、校验、日志和错误格式。

## 18.5 多 agent 系统必须做 capability subsetting

不要默认把主线程全部能力下放给子 agent。

## 18.6 LLM 工程必须把“缓存稳定性”和“转录稳定性”前置为架构约束

否则系统一复杂，成本和恢复能力都会失控。

---

## 19. 建议的源码阅读路线

如果目标是完全掌握这套架构，建议按下面顺序读：

1. `src/types/command.ts`
2. `src/Tool.ts`
3. `src/commands.ts`
4. `src/tools.ts`
5. `src/utils/processUserInput/processUserInput.ts`
6. `src/utils/processUserInput/processSlashCommand.tsx`
7. `src/query.ts`
8. `src/services/tools/toolOrchestration.ts`
9. `src/services/tools/StreamingToolExecutor.ts`
10. `src/services/tools/toolExecution.ts`
11. `src/utils/permissions/permissions.ts`
12. `src/services/mcp/client.ts`
13. `src/tools/SkillTool/SkillTool.ts`
14. `src/tools/AgentTool/runAgent.ts`
15. `src/tools/AgentTool/agentToolUtils.ts`
16. `src/tools/FileReadTool/FileReadTool.ts`
17. `src/tools/FileEditTool/FileEditTool.ts`
18. `src/tools/BashTool/BashTool.tsx`

理由很简单：

- 先读协议
- 再读聚合
- 再读输入/查询编排
- 再读执行中间件
- 最后看代表性复杂工具

这是理解成本最低的路径。

---

## 20. 总结

Claude-code-open 的“命令与工具架构”真正有价值的地方，不在于它有很多命令、很多工具，而在于它把 Agent 能力系统做成了一个可扩展、可治理、可观测、可恢复的统一运行时。

从系统设计角度看，它完成了四件关键事情：

1. 用 `Command` 定义人类控制面。
2. 用 `Tool` 定义模型执行面。
3. 用 `query.ts + toolExecution.ts` 建立统一的数据面执行管线。
4. 用权限、hooks、MCP 适配、agent tool subsetting 建立完整的治理层。

因此，这个架构最准确的评价不是“一个做得比较复杂的 tool calling 系统”，而是：

**一个面向真实开发工作流、长生命周期会话、多 agent 协作和外部生态接入的工业级 Agent Runtime。**

如果只站在“函数调用”视角看它，会低估它。

如果从“控制面 + 数据面 + 治理面”的平台视角看它，就能理解为什么这里的设计细节如此多，而且大多是合理且必要的。
