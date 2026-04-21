# 以架构视角，深入源码，逐层拆解 Claude Code 的 Tools Harness 设计哲学

基于 `vendor/Claude-code-open/src/` 源码，`./specs/read/tools.md` 中有几条判断需要校正。最关键的结论是：

- Claude Code 的工具系统不是由某个“超大 Tool.ts 底座文件”单点承载，而是由 `Tool.ts` 的工具契约、`tools.ts` 的工具注册/拼装、`query.ts` 的回路调度、`services/tools/*` 的执行器，以及各工具目录下的具体实现共同组成。
- 它的核心哲学也不是“相信模型会守规矩”，而是把工具调用拆成一条确定性的治理链：`schema 校验 -> 业务校验 -> hook -> 权限决策 -> 工具执行 -> 结果回注/持久化`。这条链的主入口在 `services/tools/toolExecution.ts:599-1493`，工具批调度在 `services/tools/toolOrchestration.ts:19-188`，回到主 agent loop 则在 `query.ts:1363-1405`。

---

## 1. 先纠偏：旧稿中哪些说法与源码不符

### 1.1 不存在一个“约 29000 行的 `src/tools/Tool.ts`”

当前仓库里真正存在的是：

- `src/Tool.ts`：定义工具契约 `Tool` / `ToolDef` / `buildTool()`，并不接近 29000 行。`buildTool()` 在这里给工具补默认行为，形成统一契约。见 `vendor/Claude-code-open/src/Tool.ts:362-716`, `vendor/Claude-code-open/src/Tool.ts:721-792`
- `src/tools.ts`：内建工具注册表、按权限过滤、与 MCP 工具池合并。见 `vendor/Claude-code-open/src/tools.ts:193-250`, `vendor/Claude-code-open/src/tools.ts:271-367`

更准确的说法是：**Claude Code 用统一的工具类型系统和注册表来约束工具，而不是把所有逻辑塞进一个单体文件里。**

### 1.2 “40 到 50 多个自包含工具模块”只能算大致量感，不能当作固定事实

源码树当前确实有较多工具目录：`vendor/Claude-code-open/src/tools/` 下一级目录数为 `42`。但最终给模型看到的工具集合不是一个固定数字，而是：

1. 先由 `getAllBaseTools()` 生成“当前环境下可能出现的内建工具全集”
2. 再经 `getTools()` 按环境、模式、deny 规则裁剪
3. 最后通过 `assembleToolPool()` 与 MCP 工具合并

对应源码：

- 基础内建工具枚举：`vendor/Claude-code-open/src/tools.ts:193-250`
- deny 规则预过滤：`vendor/Claude-code-open/src/tools.ts:262-269`
- 获取当前可用内建工具：`vendor/Claude-code-open/src/tools.ts:271-327`
- 合并 MCP 工具池：`vendor/Claude-code-open/src/tools.ts:329-367`

所以适合分享里的表述是：**Claude Code 有一个“几十个内建工具 + feature gate + MCP 动态扩展”的工具池，而不是写死的固定数量。**

### 1.3 “工具之间不存在共享易变状态”不准确

源码恰恰说明：工具执行期共享一组集中托管的上下文对象，例如：

- `ToolUseContext.messages`
- `ToolUseContext.readFileState`
- `ToolUseContext.contentReplacementState`
- `ToolUseContext.getAppState()/setAppState()`
- `ToolUseContext.setInProgressToolUseIDs`

见 `vendor/Claude-code-open/src/Tool.ts:144-288`。

因此，更准确的结论是：**工具模块边界是清晰的，但运行期状态并非“完全隔离”，而是通过 `ToolUseContext` 统一托管和传递。**

### 1.4 “AgentTool 让系统不需要特殊编排层”也需要收敛

`AgentTool` 确实把“派生子代理”包装成了普通工具调用，这一点是成立的。`AgentTool` 本身就是一个标准工具，定义了 `inputSchema`、`outputSchema`、`call()` 等。见 `vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:196-250`。

但它并不是“没有额外编排”。子代理背后仍然有一整套专门逻辑：

- `runAgent()` 负责建立子代理上下文、拼装工具池、接管 query loop。见 `vendor/Claude-code-open/src/tools/AgentTool/runAgent.ts:248-280`
- `spawnMultiAgent.ts` 负责 teammate/tmux/in-process 多代理启动。见 `vendor/Claude-code-open/src/tools/shared/spawnMultiAgent.ts:1-240`

所以更准确的表述是：**Claude Code 把 sub-agent 暴露成一个普通工具接口，但内部仍有专门的 agent orchestration 实现；只是这个编排层被封装在工具系统内部，而不是暴露成另一套外部模型协议。**

---

## 2. 工具 Harness 的真实分层

如果要给技术分享一个干净的主叙事，最适合把 Tools Harness 拆成 5 层：

```text
Tool Contract Layer
  Tool.ts / buildTool()

Tool Registry Layer
  tools.ts / getTools() / assembleToolPool()

Tool Loop Layer
  query.ts 识别 tool_use 并触发 runTools()

Tool Execution Layer
  toolOrchestration.ts / toolExecution.ts / toolHooks.ts

Tool Implementation Layer
  tools/BashTool/*, AgentTool/*, SkillTool/*, MCPTool/* ...
```

### 2.1 契约层：所有工具都服从统一接口

`Tool.ts` 定义了工具系统的核心契约。一个工具至少要能提供：

- `inputSchema`
- `description()`
- `prompt()`
- `checkPermissions()`
- `call()`
- `mapToolResultToToolResultBlockParam()`

见 `vendor/Claude-code-open/src/Tool.ts:362-716`。

随后 `buildTool()` 统一补默认值，而且默认值是明显的 fail-closed 倾向：

- `isConcurrencySafe` 默认 `false`
- `isReadOnly` 默认 `false`
- `isDestructive` 默认 `false`
- `toAutoClassifierInput` 默认空字符串

见 `vendor/Claude-code-open/src/Tool.ts:743-792`。

这就是“Schema/Contract first”的第一层含义：**工具不是一堆 ad-hoc 函数，而是先进入统一契约，再谈实现。**

### 2.2 注册层：工具先被裁剪，再进入模型视野

`tools.ts` 不是简单的导出列表，它承担三件事：

1. 定义“可能出现的内建工具全集” `getAllBaseTools()`。见 `vendor/Claude-code-open/src/tools.ts:193-250`
2. 按 deny 规则在“展示给模型之前”就过滤工具。见 `vendor/Claude-code-open/src/tools.ts:262-269`
3. 把内建工具与 MCP 工具按名称合并并去重。见 `vendor/Claude-code-open/src/tools.ts:345-367`

尤其这一点很关键：**Claude Code 不只在“调用时”做权限判断，还会在“模型看到工具列表之前”做一次工具面裁剪。** `filterToolsByDenyRules()` 的注释就明确写了：MCP 前缀 deny 规则会在模型看到工具前把整组工具去掉。见 `vendor/Claude-code-open/src/tools.ts:253-260`。

### 2.3 回路层：真正触发工具执行的是 `query.ts`

当模型在 assistant message 中产出 `tool_use` block 后，`query.ts` 会进入工具执行阶段：

- 入口：`runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)`，见 `vendor/Claude-code-open/src/query.ts:1380-1383`
- 执行过程中产生的新消息会被立刻 `yield`，并且被规范化后回写到 `toolResults`，供下一轮模型调用使用。见 `vendor/Claude-code-open/src/query.ts:1384-1405`

所以，Claude Code 的工具系统不是“工具自己跑完再统一回填”，而是**工具执行被嵌进 agent loop，作为 query loop 的一个阶段。**

### 2.4 执行层：`runTools()` 先做批次划分，再决定并发或串行

`toolOrchestration.ts` 的职责很明确：

- `partitionToolCalls()` 按 `tool.isConcurrencySafe(parsedInput)` 把一批 `tool_use` 切成若干批次。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:84-116`
- 并发安全批次走 `runToolsConcurrently()`。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:152-177`
- 非并发安全批次走 `runToolsSerially()`。见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:118-150`

这里旧稿“只读工具支持并行、写入工具强制顺序执行”是大方向正确，但源码里的判据不是“只读”这一个标签，而是 `isConcurrencySafe()`。例如 `BashTool` 就是把 `isConcurrencySafe(input)` 定义成“只有在它被判定为 read-only 时才允许并发”。见 `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:434-441`。

因此更准确的表述是：**Claude Code 的并发控制是工具自声明的，并以 `isConcurrencySafe` 为总开关；在当前实现里，很多只读工具会因此进入并发批次，而有副作用的工具默认串行。**

---

## 3. 工具执行生命周期：源码里的真实顺序

工具执行主链在 `services/tools/toolExecution.ts`，旧稿的生命周期描述基本方向是对的，但可以写得更严谨。

### 3.1 第一步：Zod 结构校验

执行入口首先做：

- `tool.inputSchema.safeParse(input)`

如果失败，会直接构造一个 `tool_result` 错误块返回给模型，不进入后续权限和执行流程。见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-680`。

这对应分享里可以强调的一点：**模型的工具输入不是“拿来就跑”，而是先过结构化 schema 边界。**

### 3.2 第二步：工具自己的 `validateInput()`

通过 schema 后，还会执行工具级的业务校验：

- 调用点：`vendor/Claude-code-open/src/services/tools/toolExecution.ts:682-733`

例如 `BashTool.validateInput()` 就会拦截前导且阻塞性的 `sleep >= 2s`，并引导改用后台任务或 Monitor tool。见 `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:317-337`, `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:524-537`。

这说明 `validateInput()` 不是多余层，它承载的是**“结构合法，但业务上仍不该允许”的治理。**

### 3.3 第三步：PreToolUse hooks

在进入权限决策前，会先执行 `runPreToolUseHooks()`：

- 调用点：`vendor/Claude-code-open/src/services/tools/toolExecution.ts:795-863`
- hook 解析与 stop/deny/updatedInput 语义：`vendor/Claude-code-open/src/services/tools/toolHooks.ts:435-520`

PreToolUse hook 可以做到：

- 注入附加消息
- 修改输入
- 直接给出 `allow/deny/ask`
- 阻止后续 continuation

这意味着 hook 不是“外围插件点”，而是**主执行链上的一级参与者。**

### 3.4 第四步：权限决策

权限决策是通过 `resolveHookPermissionDecision()` 把 hook 决策和标准权限系统整合起来：

- 调用点：`vendor/Claude-code-open/src/services/tools/toolExecution.ts:916-932`
- 具体逻辑：`vendor/Claude-code-open/src/services/tools/toolHooks.ts:321-433`

这里的一个关键设计是：

- hook 的 `allow` 不能自动绕过 deny/ask 规则
- 如果工具要求用户交互，或者 `requireCanUseTool` 为真，仍然会回到 `canUseTool()` 主路径

也就是说，**Claude Code 允许 hook 参与权限，但不让 hook 轻易绕过统一权限边界。**

### 3.5 第五步：真正执行工具 `tool.call()`

只有权限结果为 `allow`，才会进入：

- `tool.call(...)`

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1206-1223`。

执行完成后，系统会：

- 映射 tool result
- 统计结果大小
- 打日志/遥测
- 回注 `tool_result`

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1290-1479`。

### 3.6 第六步：PostToolUse hooks

工具成功后还会继续跑 `runPostToolUseHooks()`：

- 调用点：`vendor/Claude-code-open/src/services/tools/toolExecution.ts:1397-1493`
- hook 实现：`vendor/Claude-code-open/src/services/tools/toolHooks.ts:39-191`

工具失败时则有专门的 `runPostToolUseFailureHooks()`，见 `vendor/Claude-code-open/src/services/tools/toolHooks.ts:193-319`。

因此完整生命周期更准确地写成：

```text
safeParse
-> validateInput
-> PreToolUse hooks
-> permission resolution
-> tool.call
-> map/persist tool_result
-> PostToolUse / PostToolUseFailure hooks
-> 回注下一轮 query
```

---

## 4. 安全治理：Claude Code 的 Harness 到底“硬”在哪里

### 4.1 权限不是一个 if，而是多层决策链

从 `permissions.ts` 看，权限系统不是简单黑白名单，而是把这些因素串在一起：

- 当前模式，如 `default / acceptEdits / dontAsk / plan / auto`
- 工具自己的 `checkPermissions()`
- 路径安全检查
- allow/deny/ask 规则
- auto mode classifier
- headless/async agent 是否允许弹权限框

尤其在 `auto` 模式下，`permissions.ts` 会：

1. 先处理 `dontAsk`
2. 再进入 `auto` classifier 分支
3. 先判断是否是不可被 classifier 自动批准的 safety check
4. 再判断是否可以用 `acceptEdits` 快路径放行
5. 再判断是否命中 safe-tool allowlist
6. 最后才真正跑 classifier

见 `vendor/Claude-code-open/src/utils/permissions/permissions.ts:503-760`。

所以更准确的说法是：**Claude Code 的权限治理是“规则优先 + 安全检查优先 + classifier 补充判断”，而不是单纯依赖分类器。**

### 4.2 Bash 的权限链是最重的一层

`bashToolHasPermission()` 本身就是一个大型治理函数，核心逻辑在：

- `vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1663-2557`

它至少串了这些步骤：

1. AST 安全解析 / tree-sitter shadow
2. `too-complex` fail-closed
3. 语义检查 `checkSemantics`
4. sandbox auto-allow
5. exact match 规则
6. Bash prompt deny/ask classifier
7. 操作符级检查（pipe、redirect 等）
8. legacy misparse 安全检查
9. 子命令拆分与 compound command 检查
10. `cd + git` 安全检查
11. 子命令级权限决策
12. 原始命令上的路径/重定向检查
13. 汇总建议规则并返回 `allow / ask / deny`

这与旧稿“BashTool 内置了大量验证逻辑”方向一致，但准确说法应是：**Bash 的 Harness 不是单一的 sandbox，而是一条由 AST、语义、路径、规则、classifier 共同组成的 permission pipeline。**

### 4.3 “AST 分析识别危险指令”是对的，但需要说清楚它做什么、不做什么

`bashToolHasPermission()` 一上来就先走 AST 安全解析：

- `parseCommandRaw()` + `parseForSecurityFromAst()`。见 `vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1670-1695`

然后：

- 如果 AST 结果是 `too-complex`，直接走 `ask`。见 `vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1741-1768`
- 如果 AST 结果是 `simple`，还会执行 `checkSemantics()`。见 `vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1771-1794`

这层真正的哲学是：**先确认“我能不能可靠理解命令结构”，理解不了就 fail-closed。**

### 4.4 `rm -rf /` 这类灾难性删除是明确被专项处理的

这条旧稿是正确的，而且源码证据很直接：

- `checkDangerousRemovalPaths()` 明确说是为防止 `rm -rf /` 这类灾难性删除。见 `vendor/Claude-code-open/src/tools/BashTool/pathValidation.ts:65-108`
- 命中危险路径后会强制 `ask`，并明确标注“cannot be auto-allowed by permission rules”。见 `vendor/Claude-code-open/src/tools/BashTool/pathValidation.ts:88-99`

因此分享里完全可以保留“Claude Code 会对灾难性删除做专项硬拦截”。

### 4.5 对敏感文件与目录的自动编辑保护是明确存在的

旧稿提到 `.bashrc`、`.gitconfig` 等敏感文件，这点有充足源码支持。

危险文件与目录清单：

- `DANGEROUS_FILES`：`.gitconfig`, `.gitmodules`, `.bashrc`, `.bash_profile`, `.zshrc`, `.zprofile`, `.profile`, `.ripgreprc`, `.mcp.json`, `.claude.json`。见 `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:53-68`
- `DANGEROUS_DIRECTORIES`：`.git`, `.vscode`, `.idea`, `.claude`。见 `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:70-79`

自动编辑安全检查：

- `checkPathSafetyForAutoEdit()` 会同时检查原路径和 symlink-resolved 路径。见 `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:604-665`
- 一旦命中敏感文件，会返回 `safe: false`。见 `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:652-660`

因此更准确的结论是：**Claude Code 不只是“防命令危险”，也明确防“模型悄悄改启动脚本、IDE 配置、Claude 自身配置”等持久化控制点。**

### 4.6 Unicode 与路径绕过也被单独考虑了

旧稿“Unicode 规范化/防走私攻击”这个方向是对的，但实现位置需要准确化：

- Bash 命令侧，`validateUnicodeWhitespace()` 会拦截可能造成 parser differential 的 Unicode whitespace。见 `vendor/Claude-code-open/src/tools/BashTool/bashSecurity.ts:1894-1917`
- 路径侧，`normalizeCaseForComparison()` 用于防大小写混淆绕过，例如 `.cLauDe/Settings.locaL.json`。见 `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:81-92`
- Windows 花式路径绕过（ADS、8.3、长路径前缀、尾随点/空格、DOS 设备名、UNC）在 `hasSuspiciousWindowsPathPattern()` 中集中处理。见 `vendor/Claude-code-open/src/utils/permissions/filesystem.ts:490-600`

所以更准确的表达是：**Claude Code 不是泛泛地“做 Unicode 正则净化”，而是针对命令解析差异和路径规范化绕过分别做防护。**

### 4.7 “拦截超过 2 秒的 sleep”是事实，但它属于响应性治理，不是沙箱本体

源码里：

- `detectBlockedSleepPattern()` 会识别前导 `sleep N`
- 当 `N >= 2` 且不是后台运行时，`validateInput()` 直接阻止

见 `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:317-337`, `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:524-533`。

但它的提示语很明确，是让用户改用后台任务或 Monitor tool。这更像**响应性/资源占用治理**，不是“Bash sandbox 的核心安全边界”。

### 4.8 “专门拦截 fork bomb、TTY 注入、history 操纵”当前版本没有足够直接证据

我在当前源码里能确认的是：

- shell/AST 级危险语义检查
- 路径与重定向检查
- Unicode whitespace / parser differential 检查
- 敏感文件/目录保护
- compound command 与 `cd + git` 等绕过场景防护

但没有找到一组可直接支撑“专门拦截 fork bomb / TTY 注入 / history 操纵”的同等级源码入口。因此分享里不宜把这些说成已证实事实。

---

## 5. MCP 与扩展机制：工具系统如何保持开放

### 5.1 MCP 工具会被动态包装成内部 Tool

MCP 集成不是外挂，而是直接融入 Tool Harness。

`fetchToolsForClient()` 会：

1. 调 MCP `tools/list`
2. 对返回工具做 Unicode 清理
3. 把远端工具映射成内部 `Tool`
4. 继承 MCP annotations，如 `readOnlyHint`、`destructiveHint`、`openWorldHint`

见 `vendor/Claude-code-open/src/services/mcp/client.ts:1743-1835`。

尤其关键的是：

- `name` 会被转换成 `mcp__<server>__<tool>` 这样的 fully qualified name。见 `vendor/Claude-code-open/src/services/mcp/client.ts:1767-1775`
- `isConcurrencySafe()` / `isReadOnly()` / `isDestructive()` 直接从 MCP annotations 派生。见 `vendor/Claude-code-open/src/services/mcp/client.ts:1795-1806`
- `checkPermissions()` 仍然接入 Claude Code 的统一权限机制。见 `vendor/Claude-code-open/src/services/mcp/client.ts:1814-1831`

这就是分享里可以强调的一点：**在 Claude Code 里，MCP 不是第二套工具框架，而是被“收编”为统一 Tool 抽象。**

### 5.2 工具池合并也是统一做的

MCP 工具不是随意追加，而是通过 `assembleToolPool()` 与内建工具统一合并、排序、去重。见 `vendor/Claude-code-open/src/tools.ts:329-367`。

这意味着 Claude Code 的设计哲学是：

- 外部能力可以很多样
- 但进入模型之前，必须先被变成统一的 Tool contract

### 5.3 SkillTool 说明“能力封装”也被当成工具

`SkillTool` 本身也是通过 `buildTool()` 构建的标准工具。见 `vendor/Claude-code-open/src/tools/SkillTool/SkillTool.ts:331-344`

它在 `validateInput()` 中会：

- 规范化 skill 名称
- 检查 skill 是否存在
- 检查是否允许模型调用
- 检查是否为 prompt-based skill

见 `vendor/Claude-code-open/src/tools/SkillTool/SkillTool.ts:354-430`。

这说明 Claude Code 的“工具”概念并不局限于 shell/file/network 操作，而是把**可被模型调用的能力封装**都纳入统一 Harness。

---

## 6. 从源码反推 Claude Code 的 Tools Harness 设计哲学

### 6.1 第一原则：不信任模型输出，先过契约边界

表现为：

- 每个工具必须声明 `inputSchema`
- 执行前先做 `safeParse`
- 再做业务级 `validateInput`

源码位置：

- `vendor/Claude-code-open/src/Tool.ts:362-716`
- `vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-733`

对应哲学可以总结为：**模型产出的 tool input 只是候选，不是事实。**

### 6.2 第二原则：权限是主链，而不是附加层

表现为：

- hook 决策要和统一权限系统合流，而不是旁路
- deny 规则会在“模型看到工具前”就裁掉工具
- 运行时仍会做 `canUseTool()` / `checkPermissions()`

源码位置：

- `vendor/Claude-code-open/src/tools.ts:262-327`
- `vendor/Claude-code-open/src/services/tools/toolHooks.ts:321-433`
- `vendor/Claude-code-open/src/utils/permissions/permissions.ts:503-760`

对应哲学是：**权限治理不依赖模型遵守提示词，而是依赖确定性的工程边界。**

### 6.3 第三原则：先做 fail-closed 的可理解性判断，再谈自动化

最典型体现在 Bash：

- 先问“命令结构是否可被可靠理解”
- 复杂、模糊、易差异解析的命令直接 `ask`
- 只有能解释清楚的命令才进入更细颗粒的自动判定

源码位置：

- `vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1670-1794`
- `vendor/Claude-code-open/src/tools/BashTool/bashSecurity.ts:1894-1917`

对应哲学是：**自动化不是默认权利，可解释性才是自动放行的前提。**

### 6.4 第四原则：开放扩展，但必须先归一到内部 Tool 抽象

MCP、Skill、Agent 都没有绕开 Tool contract，而是被装配进同一套执行链：

- MCP：`vendor/Claude-code-open/src/services/mcp/client.ts:1743-1835`
- Skill：`vendor/Claude-code-open/src/tools/SkillTool/SkillTool.ts:331-430`
- Agent：`vendor/Claude-code-open/src/tools/AgentTool/AgentTool.tsx:196-250`

这对应的设计哲学是：**Claude Code 对外是开放的，但对内必须是统一的。**

### 6.5 第五原则：把“快”和“安全”都变成可编排属性

Claude Code 没有简单地“全部串行求稳”，而是让工具显式声明：

- 是否并发安全 `isConcurrencySafe`
- 是否只读 `isReadOnly`
- 是否 destructive `isDestructive`

见 `vendor/Claude-code-open/src/Tool.ts:401-406`, `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:19-188`。

这意味着它追求的不是“绝对保守”，而是**在契约约束下的可控并发与可控自动化。**

---

## 7. 面向分享可直接使用的最终表述

如果要把 Claude Code 的 Tools Harness 压缩成一句话，可以这样说：

> Claude Code 并不是把工具调用交给模型“自由发挥”，而是先用统一 Tool contract 把能力封装起来，再通过 query loop、权限系统、hook、结果持久化和并发调度，把每一次 tool_use 变成一条可验证、可中断、可审计的确定性执行链。

如果要进一步提炼成 4 个关键词：

- `Contract-first`
- `Fail-closed`
- `Policy-centered`
- `Unified extensibility`

如果要给《以架构视角，深入源码，进行逐层拆解，呈现 Claude Code 的 Harness 设计哲学》做一页结论页，推荐直接用下面这段：

> Claude Code 的 Tools Harness，本质上是一套“把模型行为降格为工程对象”的治理系统。模型只能提出工具调用意图；真正决定它能否执行、如何执行、执行结果如何回注上下文的，是一条由 schema、hook、permission、orchestration 和 storage 组成的确定性流水线。这就是它高代理能力背后的工程底座。
