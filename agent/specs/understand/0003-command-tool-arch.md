# Claude-code-open 命令与工具架构深度解读

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 的“命令与工具架构”。

这里的“命令”和“工具”虽然都能让系统做事，但它们不是一回事：

- 命令（Command）主要面向“人”，是终端里的显式入口。
- 工具（Tool）主要面向“模型”，是 Agent 在推理过程中可调用的执行单元。

初学者最容易混淆的问题通常有这些：

1. `/plan`、`/mcp`、`/review` 这种 slash command 和 `Read`、`Edit`、`Bash` 到底是什么关系？
2. 为什么 Skills 既像命令，又会变成工具？
3. MCP 为什么既能提供工具，也能提供 prompt/command，还能提供 resource？
4. 工具调用是怎么经过权限校验、并发调度、结果回填，最后继续推理的？
5. 为什么 `commands.ts`、`tools.ts`、`Tool.ts`、`query.ts`、`toolExecution.ts` 要拆成这么多层？

本文基于以下主干代码整理：

- `src/commands.ts`
- `src/types/command.ts`
- `src/tools.ts`
- `src/Tool.ts`
- `src/query.ts`
- `src/services/tools/toolOrchestration.ts`
- `src/services/tools/toolExecution.ts`
- `src/services/mcp/client.ts`
- `src/services/mcp/types.ts`
- `src/tools/BashTool/*`
- `src/tools/FileReadTool/*`
- `src/tools/FileEditTool/*`
- `src/tools/SkillTool/*`
- `src/tools/AgentTool/*`

---

## 2. 先给结论

这套架构本质上是在做两层能力编排：

```text
人类操作入口
  Slash Command / Local Command / Prompt Command

模型执行入口
  Tool / MCP Tool / SkillTool / AgentTool
```

可以先记住一句话：

**命令解决“用户想主动触发什么”，工具解决“模型在完成任务时需要调用什么”。**

更具体一点：

- `commands.ts` 负责聚合“用户可见的入口”。
- `tools.ts` 负责聚合“模型可见的能力集合”。
- `Tool.ts` 定义统一工具协议。
- `query.ts` 负责驱动“模型输出 tool_use -> 执行工具 -> 回填 tool_result -> 继续推理”的循环。
- `toolExecution.ts` 和 `toolOrchestration.ts` 负责单次工具调用的权限、执行、并发、结果格式化。
- `services/mcp/client.ts` 把外部 MCP server 暴露的 tool / prompt / resource 适配进本地体系。

所以它不是“命令系统”和“工具系统”各干各的，而是一个统一 Agent 平台里的两种入口层。

---

## 3. 概念模型

## 3.1 命令是什么

命令是给用户直接输入的，例如：

- `/plan`
- `/mcp`
- `/memory`
- `/review`
- `/config`

从类型上看，命令分成 3 类，定义在 `src/types/command.ts`：

### A. `prompt` 命令

特点：

- 本质上不是立即执行本地逻辑。
- 它会生成一段 prompt/message，再交给模型继续处理。
- 常见于 Skills、MCP prompts、某些老命令。

代表字段：

- `progressMessage`
- `contentLength`
- `allowedTools`
- `model`
- `source`
- `getPromptForCommand(args, context)`

### B. `local` 命令

特点：

- 本地执行。
- 返回文本、compact 结果，或者跳过消息。
- 适合纯本地逻辑。

代表字段：

- `supportsNonInteractive`
- `load()`
- `call(args, context)`

### C. `local-jsx` 命令

特点：

- 本地执行，但会渲染 Ink/React UI。
- 适合需要交互式界面的命令。

代表字段：

- `load()`
- `call(onDone, context, args)`

例如 `/plan` 就是一个 `local-jsx` 命令：它要么切换到 plan mode，要么展示当前 plan，要么调用编辑器打开 plan 文件。

## 3.2 工具是什么

工具是模型在对话循环里调用的能力单元，例如：

- `BashTool`
- `FileReadTool`
- `FileEditTool`
- `WebSearchTool`
- `TodoWriteTool`
- `SkillTool`
- `AgentTool`

工具的统一接口定义在 `src/Tool.ts` 的 `Tool` 类型里。每个工具至少要回答这些问题：

- 叫什么名字：`name`
- 输入 schema 是什么：`inputSchema`
- 如何描述自己：`description()`、`prompt()`
- 是否允许并发：`isConcurrencySafe()`
- 是否只读：`isReadOnly()`
- 是否需要权限审批：`checkPermissions()`
- 如何真正执行：`call()`
- 如何把结果转成模型能理解的 `tool_result`：`mapToolResultToToolResultBlockParam()`
- 如何在终端 UI 里展示：`renderToolUseMessage()`、`renderToolResultMessage()`

也就是说，工具不只是一个“函数”，而是一个完整的协议对象。

## 3.3 命令与工具的关系

两者既不同，又会互相转化。

最重要的几种关系：

### 关系 1：命令和工具是两套入口

- 用户直接输入 `/plan`，走命令入口。
- 模型决定读取文件，走工具入口。

### 关系 2：某些命令会生成 prompt，继续进入模型回路

也就是 `prompt` 类型命令。

### 关系 3：`SkillTool` 会把 prompt command 包装成模型可调用工具

这是整个架构里最关键的桥梁之一。

也就是说：

- 对用户来说，Skill 是 `/xxx` 命令。
- 对模型来说，Skill 是 `SkillTool(skill, args)`。

### 关系 4：MCP 既能扩展工具，也能扩展命令，还能扩展资源

所以 MCP 不是“只加工具”。

它会同时扩展：

- `mcp.tools`
- `mcp.commands`
- `mcp.resources`

这也是为什么命令与工具架构必须统一看，而不能分开理解。

---

## 4. 总体分层

从代码职责看，命令与工具架构可以分成 5 层：

```text
入口定义层
  types/command.ts
  Tool.ts

能力聚合层
  commands.ts
  tools.ts

能力扩展层
  services/mcp/*
  skills/*
  utils/plugins/*

执行编排层
  QueryEngine.ts
  query.ts

执行落地层
  services/tools/toolOrchestration.ts
  services/tools/toolExecution.ts
  tools/*/Tool.ts
```

各层职责如下。

## 4.1 入口定义层

这一层回答“一个命令/工具最少应该长什么样”。

- `types/command.ts` 统一定义命令类型。
- `Tool.ts` 统一定义工具类型、工具上下文、工具结果、默认行为。

它们是协议层，不直接做业务。

## 4.2 能力聚合层

这一层回答“当前会话可见的命令和工具有哪些”。

- `commands.ts` 负责聚合内置命令、插件命令、skills、workflow、MCP prompt。
- `tools.ts` 负责聚合内置工具，并和 MCP tools 合并。

这一层的重点是“收集”和“过滤”，不是执行。

## 4.3 能力扩展层

这一层回答“能力从哪里来”。

来源包括：

- 内置代码
- skills 目录
- 插件
- MCP server

这里最关键的设计点是：**扩展能力最终都要被适配成统一的 Command 或 Tool 结构。**

## 4.4 执行编排层

这一层回答“命令与工具最终怎样进入对话回路”。

- `QueryEngine` 从会话视角组织工具、命令、系统提示、MCP、memory。
- `query.ts` 从单次 turn 视角执行模型和工具循环。

## 4.5 执行落地层

这一层回答“某个工具调用具体怎么跑”。

- `toolOrchestration.ts` 管批次和并发。
- `toolExecution.ts` 管单次工具调用生命周期。
- `tools/*` 里的每个工具模块实现自己的校验、权限、执行和结果映射。

---

## 5. 命令架构详解

## 5.1 `commands.ts` 是命令总注册表

`src/commands.ts` 的作用不是“实现命令逻辑”，而是把所有命令源头统一收敛起来。

它的核心职责有 4 个：

1. 注册内置命令。
2. 加载 skills、plugin commands、workflow commands。
3. 根据当前用户状态过滤可用命令。
4. 提供按名称查找、按用途筛选的辅助函数。

### 静态命令集合

`COMMANDS()` 返回内置命令数组。它本身是 memoize 的，并且大量使用 feature flag 和条件 import。

这说明命令层设计有两个目标：

- 启动时不要把所有重模块一次性加载进来。
- 构建产物要能通过 feature flag 做 dead code elimination。

### 动态命令集合

`loadAllCommands(cwd)` 会把这些来源合并：

- bundled skills
- builtin plugin skills
- skill dir commands
- workflow commands
- plugin commands
- plugin skills
- 内置命令

然后 `getCommands(cwd)` 会做第二轮处理：

- 按 `availability` 过滤用户是否有资格看到该命令
- 按 `isEnabled()` 过滤 feature 开关
- 插入运行期动态发现的 skills

这说明命令列表不是固定常量，而是“环境 + 配置 + 扩展 + 当前会话状态”的合成结果。

## 5.2 命令类型为什么这样设计

`Command` 是 `CommandBase` 与三种具体命令类型的联合。

这样做的好处是：

- 共享通用元数据，如 `name`、`description`、`aliases`、`availability`
- 又允许不同命令类型有完全不同的执行方式

例如：

- `prompt` 命令关注 `getPromptForCommand`
- `local` 命令关注 `call`
- `local-jsx` 命令关注 UI 渲染和 `onDone`

这比“所有命令都实现一个统一 `run()`”更合理，因为三类命令的本质差异非常大。

## 5.3 命令查找与筛选

`commands.ts` 里几个辅助函数很关键：

- `findCommand()`
- `getCommand()`
- `hasCommand()`
- `getSkillToolCommands()`
- `getSlashCommandToolSkills()`
- `getMcpSkillCommands()`

这里体现了一个很重要的设计思想：

**“命令列表”不是只有一份，而是会按不同消费场景投影成不同子集。**

例如：

- 用户的 slash command 补全，需要一套命令子集。
- `SkillTool` 需要一套“模型可调用的 prompt commands”子集。
- MCP skill 索引又需要额外把 `AppState.mcp.commands` 里的技能并进来。

换句话说，`commands.ts` 不是单纯的 list，它更像一个命令目录服务。

## 5.4 命令的业务角色

从整体上看，命令层主要承担 3 类业务角色：

### A. 本地控制入口

例如：

- `/plan`
- `/theme`
- `/permissions`
- `/session`

它们主要操作本地状态、配置、UI。

### B. prompt 模板入口

例如 Skills、部分 review 类命令。

它们把用户输入扩展成一段更强约束的 prompt，再交给模型。

### C. 扩展生态承接入口

MCP prompts、plugin commands、workflow commands 都会被适配进命令系统。

所以命令层的本质作用是：

**给用户一个统一、显式、可发现的能力入口。**

---

## 6. 工具架构详解

## 6.1 `Tool.ts` 是工具协议中心

如果说 `commands.ts` 是命令目录，那么 `Tool.ts` 就是工具世界的“接口标准”。

它定义了三个最核心的东西：

- `Tool`
- `ToolUseContext`
- `ToolResult`

## 6.2 `ToolUseContext` 为什么这么大

初学者第一次看到 `ToolUseContext` 往往会觉得字段太多。实际上这正反映了工具在系统里的真实位置。

工具不是在一个“纯函数世界”里运行，而是在一个复杂的 Agent 运行时中运行。它需要访问：

- 全局工具/命令配置：`options.tools`、`options.commands`
- MCP 客户端和资源：`options.mcpClients`、`options.mcpResources`
- 当前消息链：`messages`
- 应用状态：`getAppState()`、`setAppState()`
- 文件缓存：`readFileState`
- 权限和审批状态
- 中断控制：`abortController`
- UI 更新能力：`setToolJSX()`、`addNotification()`
- 任务和子 agent 相关能力
- 内容替换、文件历史、归因等运行期状态

这说明工具不是“附属功能”，而是执行内核的一部分。

## 6.3 `ToolResult` 为什么不只是返回字符串

`ToolResult<T>` 里除了 `data` 以外，还有两个很关键的能力：

- `newMessages`
- `contextModifier`

### `newMessages`

允许工具直接往消息流中插入额外消息。

这类能力常见于：

- Skill 展开
- Agent 执行
- 某些带附件的结果

### `contextModifier`

允许工具在执行完成后修改 `ToolUseContext`。

这个设计很重要，因为某些工具会改变后续工具执行环境，例如：

- 增加新的运行时状态
- 更新消息引用上下文
- 注入额外环境

这说明工具不只是“产生结果”，也可能“改变后续回合的运行上下文”。

## 6.4 `buildTool()` 的作用

`buildTool()` 是所有工具定义的统一入口。

它会自动补上默认行为，例如：

- `isEnabled() -> true`
- `isConcurrencySafe() -> false`
- `isReadOnly() -> false`
- `isDestructive() -> false`
- `checkPermissions() -> allow`
- `toAutoClassifierInput() -> ''`
- `userFacingName() -> name`

这背后的设计思想是：

**工具协议很完整，但不要求每个工具都手工实现所有非关键方法。**

这样有两个好处：

1. 工具定义更简洁。
2. 默认值是 fail-closed 的，例如并发安全默认 false，只读默认 false，更保守。

## 6.5 工具为什么既有“模型描述”，又有“UI 渲染”

`Tool` 同时包含两类方法：

### 模型侧方法

- `description()`
- `prompt()`
- `inputSchema`
- `mapToolResultToToolResultBlockParam()`

它们决定模型如何理解和消费这个工具。

### UI 侧方法

- `renderToolUseMessage()`
- `renderToolResultMessage()`
- `renderToolUseProgressMessage()`
- `renderToolUseErrorMessage()`

它们决定用户在终端里如何看到这个工具。

这说明工具对象本质上是一个“双面适配器”：

- 对上，适配模型协议。
- 对下，适配终端交互体验。

---

## 7. 工具集合是如何装配出来的

## 7.1 `tools.ts` 是内置工具注册中心

`src/tools.ts` 的 `getAllBaseTools()` 是内置工具的源头列表。

它会把这些工具拼起来：

- 文件工具：Read / Edit / Write / NotebookEdit
- Shell 工具：Bash / PowerShell
- 搜索工具：Glob / Grep / WebFetch / WebSearch
- 任务工具：TaskCreate / TaskUpdate / TaskList / TaskStop / TaskOutput
- 流程工具：EnterPlanMode / ExitPlanMode / EnterWorktree / ExitWorktree
- 扩展工具：SkillTool / AgentTool / MCP resource tools / ToolSearch
- 以及各种 feature-gated 工具

这个函数有两个重要特点：

### 特点 1：它是“所有内置工具的唯一真源”

注释明确说它是 source of truth。

这意味着：

- 默认工具 preset 从这里导出。
- 系统提示缓存也依赖它保持稳定顺序。

### 特点 2：它强依赖环境与 feature flag

例如：

- 某些工具只在 ant build 下可见。
- 某些工具只在特定环境变量打开时启用。
- REPL、workflow、LSP、cron 都是条件装载。

所以工具系统天生就是“可裁剪”的。

## 7.2 `getTools()` 做了哪些过滤

`getTools(permissionContext)` 不是简单返回全部工具，而是做了多轮过滤：

1. 处理 simple mode，只保留最小工具集。
2. 去掉特殊工具，如 MCP resource tools、synthetic output。
3. 用 deny rules 过滤掉不可见工具。
4. REPL mode 下隐藏 primitive tools，只暴露 REPL wrapper。
5. 再执行每个工具自己的 `isEnabled()`。

这里有一个很重要的产品设计：

**有些工具不仅“不能执行”，而且连提示词里都不应该让模型看到。**

所以 deny rule 的作用不仅在运行时，也在“工具暴露阶段”就提前生效。

## 7.3 `assembleToolPool()` 为什么要专门存在

它负责把两类工具合并起来：

- 内置工具
- MCP 工具

并做三件事：

1. MCP 工具也按 deny rule 过滤。
2. 内置工具和 MCP 工具分别排序。
3. 按工具名去重，内置工具优先。

这里的排序不是为了好看，而是为了 **prompt cache stability**。

注释里写得很清楚：

- built-in tools 要保持一个连续前缀
- 避免 MCP tool 插进 built-in tool 中间导致系统提示缓存失效

这说明工具集合的顺序不是随意的，它会直接影响成本和性能。

## 7.4 `getMergedTools()` 和 `assembleToolPool()` 的区别

- `assembleToolPool()` 返回“真正给模型看的完整工具池”，并做去重和排序。
- `getMergedTools()` 只是简单拼起来，适合工具搜索阈值计算、token 估算等场景。

这体现出一个细节：不同业务场景对“完整工具集合”的定义并不完全相同。

---

## 8. 一次工具调用的完整业务流程

下面看最关键的主流程：模型发出 `tool_use` 之后，系统怎么把它执行完。

## 8.1 总流程图

```text
模型流式输出 assistant message
  -> query.ts 收集 tool_use blocks
  -> toolOrchestration.ts 按并发安全性分批
  -> toolExecution.ts 执行单个工具
     -> 找到工具定义
     -> 校验输入 schema / validateInput
     -> 权限判断 checkPermissions / canUseTool
     -> 执行 tool.call()
     -> 生成 progress message / user tool_result
     -> 可能返回 newMessages / contextModifier
  -> query.ts 收集 tool_result
  -> 将 tool_result 作为 user message 回填
  -> 进入下一轮模型请求
```

## 8.2 `query.ts` 先负责收集 `tool_use`

`query.ts` 在流式消费模型响应时会做几件事：

- 收集 `assistantMessages`
- 从 assistant content 中提取 `tool_use` blocks
- 把 `needsFollowUp` 置为 true
- 如果启用了 `StreamingToolExecutor`，还会边流式接收边提前执行工具

这里的核心判断是：

- 如果本轮没有 `tool_use`，说明模型已经给出最终回答，可以进入 stop hooks 和结束阶段。
- 如果有 `tool_use`，说明还需要继续下一轮。

因此工具系统本质上是 `query.ts` 的一个子循环。

## 8.3 `toolOrchestration.ts` 负责“分批”

`runTools()` 的关键算法是 `partitionToolCalls()`。

它会把本轮工具调用切成多个 batch，每个 batch 只有两种形式：

1. 单个非并发安全工具
2. 一组连续的并发安全工具

判断依据来自每个工具的 `isConcurrencySafe(input)`。

这个算法非常实用，因为它平衡了两个目标：

- 尽可能并发执行只读工具，提高速度
- 对可能互相影响的工具保持串行，保证正确性

可以把它理解成一个非常轻量的调度器。

### 为什么不是“所有只读工具都全局并发”

因为这里保守地只合并“连续的并发安全工具”。

例如：

```text
Read, Read, Edit, Read
```

会分成：

- `[Read, Read]`
- `[Edit]`
- `[Read]`

这样可以避免 `Edit` 前后的读操作被错误重排。

## 8.4 `toolExecution.ts` 负责单次工具生命周期

`runToolUse()` 所在的 `toolExecution.ts` 才是真正的一次工具调用执行器。

它要解决的问题比“调用 `tool.call()`”复杂得多，至少包括：

1. 根据名字找到工具。
2. 解析并校验输入。
3. 运行 pre-tool hooks。
4. 执行权限判定。
5. 记录 telemetry / tracing。
6. 执行工具本体。
7. 处理 progress 消息。
8. 将结果格式化为标准 `tool_result`。
9. 运行 post-tool hooks。
10. 在出错或拒绝时生成标准化错误消息。

所以 `toolExecution.ts` 是工具世界的“中间件总线”。

## 8.5 权限判定发生在哪

权限不是只在一个地方判定，而是分层判定：

### 第一层：工具集合过滤

在 `tools.ts` 中，deny rule 会直接把工具从可见列表中剔除。

### 第二层：工具输入校验

每个工具的 `validateInput()` 可以先做无副作用检查。

例如：

- `FileReadTool` 会先检查路径、页码范围、二进制文件、设备文件
- `FileEditTool` 会先检查 old/new string 是否合理、文件是否已读、是否被外部修改

### 第三层：工具特定权限检查

每个工具的 `checkPermissions()` 会执行工具级审批逻辑。

例如：

- `FileReadTool` 调 `checkReadPermissionForTool`
- `FileEditTool` 调 `checkWritePermissionForTool`
- `BashTool` 调 `bashToolHasPermission`
- MCP tool 默认返回 `passthrough`，表示继续走更高层审批

### 第四层：统一 `canUseTool`

在 `QueryEngine.submitMessage()` 中，外部注入的 `canUseTool` 会被包装，用于记录权限拒绝信息，再贯穿到执行链中。

所以权限架构不是一刀切，而是“工具特定策略 + 全局审批机制”的组合。

---

## 9. 核心数据结构

## 9.1 `Command`

命令的核心结构在 `src/types/command.ts`。

最值得关注的字段有：

- `type`
- `name`
- `description`
- `aliases`
- `availability`
- `source`
- `loadedFrom`
- `disableModelInvocation`
- `userInvocable`

这些字段共同回答了 4 个问题：

1. 这个命令怎么执行？
2. 谁能看到它？
3. 它来自哪里？
4. 模型能不能调用它？

## 9.2 `PromptCommand`

这是命令与工具架构之间最关键的桥梁结构。

它除了命令基础字段外，还包含：

- `allowedTools`
- `model`
- `hooks`
- `skillRoot`
- `context: 'inline' | 'fork'`
- `agent`
- `effort`
- `paths`
- `getPromptForCommand()`

这说明 Skill 并不只是“一段文本模板”，而是带执行约束的 prompt 任务定义。

## 9.3 `Tool`

`Tool` 是工具协议主结构。

其中最关键的字段分成 5 组：

### 元数据组

- `name`
- `aliases`
- `searchHint`
- `isMcp`
- `mcpInfo`

### 模型协议组

- `description()`
- `prompt()`
- `inputSchema`
- `inputJSONSchema`
- `outputSchema`

### 执行控制组

- `isConcurrencySafe()`
- `isReadOnly()`
- `isDestructive()`
- `checkPermissions()`
- `validateInput()`

### 执行结果组

- `call()`
- `mapToolResultToToolResultBlockParam()`
- `toAutoClassifierInput()`

### UI 展示组

- `renderToolUseMessage()`
- `renderToolResultMessage()`
- `renderToolUseProgressMessage()`

这就是为什么说工具对象不是简单函数，而是一个完整协议。

## 9.4 `ToolUseContext`

这是工具执行时最核心的上下文容器。

可以把它理解为：

```text
一次工具调用所需的一切运行时能力
```

它的内容大致可以归为 6 类：

1. 全局配置：工具、命令、MCP、模型
2. 会话状态：messages、queryTracking、agentId
3. 应用状态：getAppState / setAppState
4. 执行控制：abortController、setInProgressToolUseIDs
5. UI 能力：setToolJSX、notifications、spinner
6. 运行期缓存：readFileState、contentReplacementState、file history

## 9.5 `ToolResult`

`ToolResult<T>` 的意义在于把“工具本体输出”和“回路控制输出”分开：

- `data`：工具业务结果
- `newMessages`：追加消息
- `contextModifier`：修改后续上下文
- `mcpMeta`：MCP 协议元数据透传

这就是工具系统能支持复杂控制流的关键。

## 9.6 MCP 相关结构

`src/services/mcp/types.ts` 定义了 MCP 的几组关键结构：

- `ScopedMcpServerConfig`
- `ConnectedMCPServer`
- `FailedMCPServer`
- `PendingMCPServer`
- `ServerResource`

这些结构说明 MCP 在系统里不是“临时网络调用”，而是长期驻留的连接型能力源。

它既有配置作用域，也有连接状态机。

---

## 10. 几个关键算法与设计技巧

## 10.1 算法一：工具批次划分

位置：`services/tools/toolOrchestration.ts`

核心思想：

- 连续的并发安全工具可以并发。
- 非并发安全工具必须独占一个批次。

这是一个局部顺序保守、全局尽量提速的算法。

它没有引入复杂 DAG 调度器，但已经能解决多数工具执行场景。

## 10.2 算法二：Read-before-Edit 的防陈旧写保护

位置：`tools/FileEditTool/FileEditTool.ts`

`FileEditTool` 里有一套非常关键的保护机制：

1. 编辑前要求该文件已经被 `Read` 过。
2. 会检查文件最后修改时间是否晚于读取时刻。
3. 如果文件被用户或 linter 改动过，必须重新读取。
4. 真正写入前还会再次检查，尽量保持 read-modify-write 原子性。

这套机制解决的是 Agent 系统里非常常见的问题：

**模型根据旧上下文编辑文件，结果把用户刚改过的内容覆盖掉。**

这不是简单的“权限”问题，而是并发一致性问题。

## 10.3 算法三：Read 结果去重

位置：`tools/FileReadTool/FileReadTool.ts`

`FileReadTool` 会把已读文件的内容和时间戳放入 `readFileState`。

如果后续重复读取同一文件、同一区间、并且文件未变化，就返回 `file_unchanged` stub，而不是重复把全文塞回上下文。

这背后解决的是两个成本问题：

1. token 浪费
2. prompt cache 失效

所以它不是单纯的“缓存”，而是针对 Agent 长对话场景的上下文去重优化。

## 10.4 算法四：MCP 名称规范化

位置：`services/mcp/mcpStringUtils.ts`

MCP 工具和命令会被规范化成类似：

```text
mcp__server__tool
```

这样做有几个作用：

1. 避免与内置工具重名。
2. 让权限规则能精确命中。
3. 让不同 server 下的同名工具可共存。

同时，工具对象还保存 `mcpInfo: { serverName, toolName }`，保证显示名、权限名、协议名三者可以分开处理。

这是一种很典型的“外部命名空间适配”技巧。

## 10.5 算法五：Skill 的双重执行模式

位置：`tools/SkillTool/SkillTool.ts`

Skill 有两种运行模式：

### `inline`

把 skill 展开成当前会话里的新消息，继续在当前对话上下文中运行。

### `fork`

把 skill 放到子 agent 里独立执行。

这意味着 Skill 不是固定实现，而是一个“可切换执行边界”的抽象。

这个设计很强，因为它允许：

- 简单技能直接 inline，成本低
- 复杂技能独立 fork，避免污染主线程上下文

## 10.6 算法六：MCP 能力三分法

这个不是代码算法，而是架构算法。

MCP 能力被拆成三类：

- tool
- prompt/command
- resource

对应本地适配层分别变成：

- `MCPTool`
- `Command`
- `ListMcpResourcesTool` / `ReadMcpResourceTool`

这种拆分非常关键，因为三种能力的交互模式完全不同：

- tool 是执行型
- prompt 是生成型
- resource 是读取型

它们不能被粗暴地统一成一种对象。

---

## 11. 代表性模块解读

下面挑几个最能体现设计思想的模块。

## 11.1 `BashTool`：最复杂的执行型工具

`BashTool` 体现了工具协议的大部分能力：

- 输入 schema 很复杂
- 可以判断只读/非只读
- 可以做安全解析和权限匹配
- 支持进度消息
- 支持后台执行
- 支持大输出落盘
- 支持 UI 结果渲染

几个关键点：

### A. 并发安全性来自“命令语义”，不是工具类型

`BashTool.isConcurrencySafe(input)` 内部其实会依据命令语义判断，而不是简单认为“Bash 永远不可并发”。

这很合理，因为：

- `ls`、`cat`、`grep` 之类命令通常只读
- `git commit`、`mv`、`rm` 则明显不是

### B. 结果不一定直接回填全文

如果输出太大，会落到磁盘，再在 `tool_result` 里放一个摘要和路径。

这也是 Agent 系统里很典型的“上下文瘦身”做法。

### C. 工具本身就带任务语义

它支持后台运行、任务 ID、输出路径、完成通知。

说明 Shell 工具在这个系统里已经不只是“执行命令”，而是带任务管理能力。

## 11.2 `FileReadTool`：只读工具的标杆实现

`FileReadTool` 的设计特别能体现“保守、安全、节省 token”的原则。

它做了很多前置校验：

- PDF 页码是否合法
- 路径是否被 deny
- 是否是 UNC 路径
- 是否是二进制文件
- 是否是会挂死的设备文件

而且它明确声明：

- `isConcurrencySafe() -> true`
- `isReadOnly() -> true`

这会直接影响调度器能否并发执行它。

同时它还会触发 skill directory 发现和 conditional skill 激活，说明“读文件”不仅是读内容，也是在帮系统发现新的能力。

## 11.3 `FileEditTool`：一致性保护最强的写工具

`FileEditTool` 的设计核心不是“替换字符串”，而是“在 Agent 场景下安全编辑文件”。

它的关键保护机制包括：

- old_string 与 new_string 不能相同
- 文件必须已读
- 文件未被外部改动
- old_string 必须可定位
- 多匹配时必须 `replace_all`
- notebook 要交给专门工具
- 设置文件要额外校验

然后真正写入时，还会：

- 保留编码和换行风格
- 写前记录 file history
- 更新 read cache
- 通知 LSP 与 VSCode

所以它不是简陋的文本替换器，而是一个带一致性保护、生态联动、可观测性的编辑事务。

## 11.4 `SkillTool`：命令与工具之间的桥

`SkillTool` 是整个架构里最值得理解的模块之一。

它做了三件非常关键的事：

1. 从命令系统里筛出 prompt-based skills。
2. 对模型暴露一个统一的技能执行工具。
3. 按 skill 定义决定 inline 还是 fork 执行。

这使得 Skill 能同时满足：

- 对用户可见：作为命令存在
- 对模型可见：作为工具存在
- 对扩展生态友好：本地 skill、plugin skill、bundled skill、MCP skill 都能纳入统一入口

## 11.5 `AgentTool` / `runAgent.ts`：工具还能再启动一个 Agent

`AgentTool` 说明这里的工具系统不是平面的，而是递归的。

子 agent 执行时会：

- 解析 agent definition
- 初始化 agent 专属 MCP server
- 解析该 agent 可用的工具集
- 创建 subagent context
- 调用同一个 `query()` 执行循环

也就是说，子 agent 并没有绕开主架构，而是复用同一套工具执行框架。

这说明系统设计得足够统一：主线程和子线程只是上下文不同，不是执行内核不同。

## 11.6 MCP resource tools：为什么不把 resource 也做成普通 MCPTool

`ListMcpResourcesTool` 和 `ReadMcpResourceTool` 被单独实现，而不是复用 `MCPTool`。

原因是 resource 的交互范式与 tool 不同：

- 它们是先枚举，再按 URI 读取
- 输入结构固定
- 输出更偏数据访问，不像 action invocation

所以它们被做成两个显式工具：

- 列资源
- 读资源

这对模型更友好，也更容易做权限与 UI 展示。

---

## 12. MCP、Skill、Plugin 是怎样接入命令与工具架构的

## 12.1 MCP 接入方式

MCP 接入是最复杂的扩展路径。

### MCP tool 接入

`fetchToolsForClient()` 会把远端返回的工具列表适配成本地 `Tool` 对象：

- 继承 `MCPTool` 基础定义
- 注入真实名字、描述、schema
- 绑定 `mcpInfo`
- 根据 annotations 设置 `readOnlyHint`、`destructiveHint`、`openWorldHint`
- 配置 `checkPermissions()`
- 实现真实 `call()`

### MCP prompt 接入

`fetchCommandsForClient()` 会把远端 prompts 适配成 `prompt` 命令。

### MCP resource 接入

资源不会被直接灌进工具列表，而是通过 resource tool 间接访问。

因此 MCP 是同时扩展三条能力线。

## 12.2 Skill 接入方式

Skill 有多个来源：

- 本地 skills 目录
- bundled skills
- plugin skills
- MCP skills
- 动态发现 skills

这些 skill 会先被适配成 `Command`，然后：

- 用户可以通过 slash command 使用
- 模型可以通过 `SkillTool` 使用

这就是 skill 的“双重入口”机制。

## 12.3 Plugin 接入方式

插件可以贡献：

- commands
- skills
- agents
- MCP server

因此插件本身并不是一种独立执行协议，而是能力来源。

最终仍然要落进命令系统、工具系统、MCP 系统或 agent 系统。

---

## 13. 典型业务流程梳理

## 13.1 流程一：用户执行 `/plan`

```text
用户输入 /plan
  -> 命令系统匹配到 local-jsx command
  -> 加载 commands/plan 模块
  -> 修改 toolPermissionContext.mode = plan
  -> 必要时继续向模型发送“计划描述”
```

这个流程没有经过工具系统。

说明命令层可以直接操作运行模式。

## 13.2 流程二：模型读取并编辑文件

```text
模型输出 FileReadTool
  -> query.ts 收集 tool_use
  -> toolExecution.ts 执行 FileReadTool
  -> 返回 tool_result 和文件内容
  -> 模型基于读到的内容输出 FileEditTool
  -> FileEditTool 检查 read cache 和 staleness
  -> 写入文件
  -> 返回成功 tool_result
  -> 模型继续回答
```

这里的关键保护是：

- 先读后改
- 改前确认未陈旧

## 13.3 流程三：模型调用 Skill

```text
模型输出 SkillTool(skill=xxx)
  -> SkillTool 从命令目录查找对应 prompt command
  -> 判断是 inline 还是 fork
  -> inline: 生成新消息并回填当前会话
  -> fork: 创建子 agent，单独执行，再汇总结果
```

这里体现的是“命令能力被工具化”。

## 13.4 流程四：模型调用 MCP 工具

```text
启动时连接 MCP server
  -> fetchToolsForClient() 适配成 Tool
  -> assembleToolPool() 合并进当前工具池
  -> 模型输出 mcp__server__tool
  -> toolExecution.ts 执行该适配工具
  -> 内部转发到 MCP client.callTool()
  -> 结果映射为 tool_result
  -> 回填模型继续推理
```

这里体现的是“外部能力被本地协议化”。

---

## 14. 为什么这个架构对初学者难，但其实很合理

初看会觉得模块很多、概念很多，原因是它同时解决了 5 类复杂性：

1. 用户入口复杂性：slash command、local command、prompt command
2. 模型执行复杂性：tool_use、tool_result、并发、权限、重试
3. 扩展来源复杂性：builtin、plugin、skill、MCP、agent
4. 交互形态复杂性：REPL、SDK、remote、subagent
5. 安全与性能复杂性：权限、缓存、压缩、上下文控制

如果把这些能力全部塞进一个“命令执行器”或“工具执行器”，代码会很快失控。

当前架构的优点在于：

- 命令和工具职责边界清楚
- 扩展能力统一适配
- 执行协议标准化
- 子 agent 复用同一套主循环
- 权限与并发控制有明确挂点

它真正复杂的地方，不是“写得绕”，而是它要承载的业务本来就复杂。

---

## 15. 初学者建议的阅读顺序

如果你准备继续深入源码，建议按下面顺序读：

1. `src/types/command.ts`
2. `src/Tool.ts`
3. `src/commands.ts`
4. `src/tools.ts`
5. `src/query.ts`
6. `src/services/tools/toolOrchestration.ts`
7. `src/services/tools/toolExecution.ts`
8. `src/tools/FileReadTool/FileReadTool.ts`
9. `src/tools/FileEditTool/FileEditTool.ts`
10. `src/tools/SkillTool/SkillTool.ts`
11. `src/services/mcp/client.ts`
12. `src/tools/AgentTool/runAgent.ts`

这样读的原因是：

- 先理解协议
- 再理解装配
- 再理解主流程
- 最后看代表性实现

这是理解这套架构成本最低的路径。

---

## 16. 最后总结

“命令与工具架构”的核心不是把能力分成两堆，而是建立一个统一 Agent 平台中的两级入口：

- 命令面向用户，强调显式触发、可发现、可交互。
- 工具面向模型，强调标准协议、可调度、可审批、可组合。

在这之上，Skills、Plugins、MCP、Agents 都不是额外平行体系，而是被适配进这两级入口的扩展来源。

所以从架构本质上说，这个系统做成了三件事：

1. 用 `Command` 统一“人类入口”。
2. 用 `Tool` 统一“模型能力”。
3. 用 `query.ts + toolExecution.ts` 把两者接进同一个 Agent 执行回路。

理解了这一点，后面再看：

- 为什么 skill 可以既是命令又是工具
- 为什么 MCP 能同时提供 tool / prompt / resource
- 为什么子 agent 还能复用同一套执行内核

这些设计就都会变得自然。
