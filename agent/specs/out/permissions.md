# Claude Code Harness 权限机制源码复核

本文以 `agent/vendor/Claude-code-open/src/` 为准，对 `agent/specs/read/permissions.md` 中的描述做源码级重梳理。结论先行：Claude Code 的权限系统确实是 Harness 里的核心治理层，但它不是“统一风险等级表驱动”的单层开关，而是由“工具输入校验 + 通用权限编排 + 工具专属权限逻辑 + 多代理转发审批”组成的组合式框架。

## 1. 权限模式：源码里能确认的，不是“固定六种对外模式”

源码把模式分成“外部可配置模式”和“内部类型全集”两层：

- 对外可配置的 `EXTERNAL_PERMISSION_MODES` 只有 5 个：`acceptEdits`、`bypassPermissions`、`default`、`dontAsk`、`plan`。见 `src/types/permissions.ts:16-24`。
- 内部 `PermissionMode` 额外包含 `auto` 和 `bubble`，但运行时可验证集合 `PERMISSION_MODES` 只会在 `TRANSCRIPT_CLASSIFIER` 打开时追加 `auto`，不会把 `bubble` 暴露为用户可选模式。见 `src/types/permissions.ts:26-38`。
- UI 配置层同样只为 `default`、`plan`、`acceptEdits`、`bypassPermissions`、`dontAsk`，以及特性开关下的 `auto` 提供标题/样式；`bubble` 没有外部配置项。见 `src/utils/permissions/PermissionMode.ts:42-91`、`97-105`。

所以，原文“六种权限模式”并不准确。更严格的说法应是：

- 稳定对外模式是 5 种。
- `auto` 是受 `TRANSCRIPT_CLASSIFIER` 特性控制的附加模式。
- `bubble` 只是内部类型成员，不是面向用户的常规模式。

各模式在权限流中的真实语义如下：

- `default`：没有特殊豁免，最终会把 `passthrough` 收束为 `ask`，进入交互审批。见 `src/utils/permissions/permissions.ts:1299-1310`。
- `plan`：本身不是“禁止一切执行”的硬编码总闸。它仍然走同一套权限框架；但团队协作时需要 leader 的 plan approval 响应才能退出 plan mode。见 `src/utils/permissions/permissions.ts:522-525`、`1268-1279`；`src/hooks/useInboxPoller.ts:156-193`。
- `bypassPermissions`：会在通用权限编排中直接放行，但工具级 `ask` 规则和 `safetyCheck` 仍然可以“免绕过”。见 `src/utils/permissions/permissions.ts:1238-1259`、`1262-1281`。
- `dontAsk`：不是“YOLO 全盘拒绝”的别名，而是在最终阶段把 `ask` 统一改写成 `deny`。见 `src/utils/permissions/permissions.ts:503-517`。
- `acceptEdits`：对文件写类工具，在允许工作目录内可直接放行；对 Bash，只自动放行少量文件系统命令 `mkdir/touch/rm/rmdir/mv/cp/sed`。见 `src/utils/permissions/filesystem.ts:1360-1375`；`src/tools/BashTool/modeValidation.ts:7-15`、`37-49`、`72-115`。
- `auto`：不是简单“自动批准”，而是先走正常权限判断，再由 classifier、allowlist fast path、acceptEdits fast path 等分支做自动决策。见 `src/utils/permissions/permissions.ts:518-590`、`593-686`、`688-876`。

## 2. 决策主链路：不是单一 `QueryEngine -> canUseTool -> checkPermissions`

更准确的执行顺序如下：

1. 工具调用先做 schema 解析与 `validateInput()` 预校验。
   见 `src/services/tools/toolExecution.ts:682-733`。
2. 只有校验通过后，才会进入 `tool.checkPermissions()`。
   见 `src/Tool.ts:489-503`、`src/utils/permissions/permissions.ts:1208-1217`。
3. 通用权限编排函数 `hasPermissionsToUseToolInner()` 先处理整工具级 deny/ask 规则，再调用各工具自己的 `checkPermissions()`，然后再叠加模式/allow 规则，把 `passthrough` 收束为 `ask`。
   见 `src/utils/permissions/permissions.ts:1158-1310`。
4. UI 层的 `useCanUseTool()` 才根据 `allow / deny / ask` 结果，决定是直接通过、直接拒绝，还是进入交互式审批、协调者审批、worker 邮箱审批等后续流程。
   见 `src/hooks/useCanUseTool.tsx:32-180`。
5. `QueryEngine` 只是包裹 `canUseTool` 以记录 denial，真正的权限判定不在 `QueryEngine` 内完成。
   见 `src/QueryEngine.ts:243-270`。

因此，原文把 `QueryEngine` 描述成核心“拦截并引导全部权限管道”的中心点，略有失真。真正的中心在：

- 执行前校验：`toolExecution.ts`
- 通用权限编排：`utils/permissions/permissions.ts`
- 交互与协作审批分发：`hooks/useCanUseTool.tsx`

## 3. 规则系统：确实存在，而且优先级高于很多模式分支

权限规则来源并不只是一份本地配置文件，而是多来源汇总：

- `userSettings`
- `projectSettings`
- `localSettings`
- `flagSettings`
- `policySettings`
- `cliArg`
- `command`
- `session`

见 `src/types/permissions.ts:54-62`，以及规则汇总顺序 `src/utils/permissions/permissions.ts:109-131`、`213-220`。

规则匹配有两层：

- 整工具匹配：`Bash`、`Edit`、`mcp__server` 这类规则，见 `src/utils/permissions/permissions.ts:240-302`。
- 带内容的规则：如 `Bash(git commit:*)`、`Edit(/.claude/**)`，见 `src/utils/permissions/permissions.ts:345-389`。

此外，工具池在暴露给模型前，就会先按 deny 规则过滤整工具，这一点很关键：

- `filterToolsByDenyRules()` 会把被整工具 deny 的内建工具和 MCP 工具直接从工具池里剔除。
  见 `src/tools.ts:253-268`、`345-352`。

这意味着 Claude Code 的治理不只发生在“调用时”，还发生在“提示词中的工具可见性”阶段。

## 4. 不存在统一的 LOW/MEDIUM/HIGH 风险枚举；风险判断是“通用框架 + 工具特化”

源码里没有一个全局 `LOW / MEDIUM / HIGH` 风险等级体系来决定审批路径。实际机制是：

- 通用框架只认识 `allow / deny / ask` 三种行为。见 `src/types/permissions.ts:44`。
- 各工具用自己的 `checkPermissions()` 把业务语义映射成 `PermissionDecision`。
  见 `src/Tool.ts:494-503`。
- 例如：
  - `FileEditTool` / `FileWriteTool` / `NotebookEditTool` 都委托给 `checkWritePermissionForTool()`。
    见 `src/tools/FileEditTool/FileEditTool.ts:125-131`、`src/tools/FileWriteTool/FileWriteTool.ts:135-141`、`src/tools/NotebookEditTool/NotebookEditTool.ts:125-131`。
  - `BashTool` 委托给 `bashToolHasPermission()`。
    见 `src/tools/BashTool/BashTool.tsx:539-540`。

所以更准确的说法不是“系统先做统一风险分级，再决定是否审批”，而是“每个工具通过专属权限逻辑产出 `allow/deny/ask`，再由通用权限框架统一收束”。

## 5. 文件编辑防线：源码强支持，且比概要稿更具体

文件写入类工具共用 `checkWritePermissionForTool()`，其顺序非常明确：

1. 先检查 deny 规则。
   见 `src/utils/permissions/filesystem.ts:1219-1239`。
2. 放行少数内部可编辑路径。
   见 `src/utils/permissions/filesystem.ts:1241-1250`。
3. 对 `.claude/**` 的 session 级 allow 规则做特判。
   见 `src/utils/permissions/filesystem.ts:1252-1300`。
4. 执行安全检查 `checkPathSafetyForAutoEdit()`。
   见 `src/utils/permissions/filesystem.ts:1302-1337`。
5. 再处理 ask 规则。
   见 `src/utils/permissions/filesystem.ts:1340-1358`。
6. 若在 `acceptEdits` 且目标位于允许工作目录内，直接放行。
   见 `src/utils/permissions/filesystem.ts:1360-1375`。
7. 最后才看 allow 规则。
   见 `src/utils/permissions/filesystem.ts:1377-1388`。

其中的安全检查内容包括：

- 敏感文件：`.gitconfig`、`.bashrc`、`.zshrc`、`.mcp.json`、`.claude.json` 等。见 `src/utils/permissions/filesystem.ts:57-68`。
- 敏感目录：`.git`、`.vscode`、`.idea`、`.claude`。见 `src/utils/permissions/filesystem.ts:74-79`。
- Windows 可疑路径模式、Claude 自身配置路径、危险文件路径，都会触发 `safetyCheck`。见 `src/utils/permissions/filesystem.ts:620-665`、`1302-1337`。

这部分与原文“`.gitconfig`、`.bashrc` 等敏感文件被严格保护”是一致的，但实际实现比“黑白名单”更精细，包含 symlink 解析后的多路径检查、工作目录约束和 `.claude` session 级例外。

## 6. Bash 是最重防的一层，但要精确描述，不宜泛化

### 6.1 AST/Tree-sitter 确实参与 Bash 权限治理

`bashToolHasPermission()` 入口首先做 AST-based security parse：

- 通过 `parseCommandRaw()` 和 `parseForSecurityFromAst()` 解析命令。
- 若结果是 `too-complex`，直接进入人工审批路径，而不是继续相信 legacy split。
- 若结果是 `simple`，再做 `checkSemantics()`。

见 `src/tools/BashTool/bashPermissions.ts:1670-1805`。

因此，原文“BashTool 有 AST 级治理”是成立的，但更准确的说法是：

- AST 主要用于命令结构可信度、语义检查、重定向/子命令拆分可信化；
- 不是“一个 AST 审计器独立完成所有危险命令识别”。

### 6.2 Bash 的关键防线是“多段组合”，不只是 AST

源码里 Bash 权限至少包含这些层次：

- AST 解析与 `too-complex` 兜底，见 `bashPermissions.ts:1670-1768`。
- 语义检查 `checkSemantics()`，见 `bashPermissions.ts:1771-1794`。
- sandbox auto-allow 分支，见 `bashPermissions.ts:1829-1843`。
- prompt deny/ask classifier，见 `bashPermissions.ts:1856-1940` 一段。
- 多操作符命令权限检查，见 `bashPermissions.ts:2239-2266`。
- 原始命令重定向路径校验，见 `bashPermissions.ts:2268-2286`。
- `cd + git` 组合防护，防止 bare repo 攻击，见 `bashPermissions.ts:2202-2224`。
- 在 legacy 路径下继续做 command injection / misparsing 检查，见 `bashPermissions.ts:1808-1827`、`2338-2365`。

### 6.3 “sleep 超过 2 秒被拦截”属实，但范围比原文窄

源码不是“沙箱统一拦截所有 sleep > 2s”，而是 `BashTool.validateInput()` 在特定条件下拦截“首个子命令是 `sleep N` 且 `N >= 2`”的前台阻塞命令：

- 模式识别：`detectBlockedSleepPattern()` 只检查第一个子命令是否为整数秒的 `sleep N`，且 `N >= 2`；浮点秒、pipeline、subshell、脚本内 sleep 不在此规则内。
  见 `src/tools/BashTool/BashTool.tsx:322-337`。
- 触发位置：`BashTool.validateInput()` 在 `MONITOR_TOOL` 打开、后台任务未禁用、且 `run_in_background` 未设置时返回校验失败。
  见 `src/tools/BashTool/BashTool.tsx:524-537`。

所以原文这点需要校正为：“Bash 前置校验会拦截一类前台阻塞式 `sleep N` 用法”，而不是“沙箱全面禁止所有超 2 秒 sleep”。

## 7. Auto mode：确实有 classifier，但不是“只有一个 YOLO 分类器”

源码里至少能确认三类与 auto/分类相关的路径：

- 通用 auto mode classifier：`classifyYoloAction()`，输入是转译后的工具动作和会话上下文。见 `src/utils/permissions/permissions.ts:688-699`。
- allowlist fast path：安全工具可跳过 classifier。见 `src/utils/permissions/permissions.ts:658-686`。
- acceptEdits fast path：如果某操作在 `acceptEdits` 下本可直接放行，也可跳过 classifier。见 `src/utils/permissions/permissions.ts:593-649`。
- Bash 还存在独立的 prompt deny/ask/allow classifier 路径，并支持 `pendingClassifierCheck` 的异步审批。见 `src/tools/BashTool/bashPermissions.ts:1755-1767`、`1856-1940`；`src/hooks/useCanUseTool.tsx:126-159`。

因此，原文“自动模式由一个 YOLO 分类器在不同思维层面检查操作合法性”过于简化。源码显示它是：

- 通用 auto classifier
- Bash 专项 classifier
- allowlist / acceptEdits 快速放行
- fail-open / fail-closed / transcript too long 回退逻辑

共同组成的自动审批系统。见 `src/utils/permissions/permissions.ts:518-590`、`593-876`。

## 8. 多代理协作：Mailbox 审批模型是成立的，且有原子 claim 防竞态

这部分原概要基本方向正确，而且源码证据较强。

### 8.1 Worker 无法直接做终态审批，需转发给 leader

- `handleSwarmWorkerPermission()` 在 worker 侧先尝试 classifier 自动放行；失败后创建 permission request，经 mailbox 发给 leader。
  见 `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:26-57`、`59-124`。
- `permissionSync.ts` 顶部注释明确写明了 worker -> leader request、leader -> worker response 的 mailbox 流程。
  见 `src/utils/swarm/permissionSync.ts:1-19`。
- leader 侧 `useInboxPoller()` 会把权限请求装入标准 `ToolUseConfirmQueue`，走和本地相同的审批 UI，再把结果发回 worker。
  见 `src/hooks/useInboxPoller.ts:250-345`。

### 8.2 原子 claim 确实存在

- `createResolveOnce()` 提供 `claim()`，注释明确说明它是“atomically check-and-mark as resolved”。
  见 `src/hooks/toolPermission/PermissionContext.ts:63-93`。
- worker 邮箱审批回调在 `onAllow` / `onReject` / `abort` 三处都先执行 `claim()`，防止多个异步来源同时 resolve。
  见 `src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:67-146`。
- 频道权限转发的设计说明里也写明“First resolver wins via claim()”，表明 claim 语义是整个权限竞争模型的一致约束。
  见 `src/services/mcp/channelPermissions.ts:4-7`。

因此，“Atomic Claim 保证并行执行时不会双重处理同一审批请求”这个结论是成立的。

## 9. 原概要中需要明确修正的几条

### 9.1 “六种权限模式”

不准确。更准确是“5 个外部模式 + 1 个特性开关下的 auto + 1 个内部 bubble 类型成员”。见 `src/types/permissions.ts:16-38`。

### 9.2 “风险分级体系 LOW / MEDIUM / HIGH”

源码中未看到统一的全局风险等级枚举作为权限调度核心。真实机制是每个工具返回 `allow / deny / ask`。见 `src/types/permissions.ts:44`、`src/Tool.ts:494-503`。

### 9.3 “QueryEngine 截获 canUseTool 并引导全部校验”

不够准确。`QueryEngine` 只是包装 `canUseTool` 以记录 denial；真正的权限主链路在 `toolExecution.ts`、`permissions.ts`、`useCanUseTool.tsx`。见 `src/QueryEngine.ts:243-270`。

### 9.4 “partiallySanitizeUnicode() 用于权限检查防 Unicode 走私”

现有源码证据不足。`partiallySanitizeUnicode()` 确实存在，但当前明确可见的调用点是 deep link 解析，不在 Bash 权限检查主链路中。见 `src/utils/sanitization.ts:1-90`、`src/utils/deepLink/parseDeepLink.ts:138-147`。

### 9.5 “2.9 万行 Tool.ts 基类强制执行”

明显与源码不符。`Tool.ts` 是核心抽象，但并非“2.9 万行”；权限强制执行也分散在工具执行器、权限编排器、各工具权限实现与 hooks 层。见 `src/Tool.ts:489-503`、`src/services/tools/toolExecution.ts:682-733`、`src/utils/permissions/permissions.ts:1158-1310`。

## 10. 更贴近源码的设计哲学总结

如果从 Harness 视角总结，Claude Code 的权限设计哲学更接近下面这三句话：

1. **先把工具调用变成结构化对象，再决定能不能做。**
   `validateInput()` 先于 `checkPermissions()`，说明它先追求“输入可信”，再追求“授权可信”。见 `src/services/tools/toolExecution.ts:682-733`、`src/Tool.ts:489-503`。

2. **通用编排层只管规则、模式和流程，真正的危险语义交给工具自己解释。**
   文件编辑走 `checkWritePermissionForTool()`，Bash 走 `bashToolHasPermission()`，不是一把大锁覆盖一切。见 `src/tools/FileEditTool/FileEditTool.ts:125-131`、`src/tools/FileWriteTool/FileWriteTool.ts:135-141`、`src/tools/NotebookEditTool/NotebookEditTool.ts:125-131`、`src/tools/BashTool/BashTool.tsx:539-540`。

3. **审批不是单点，而是一个竞争式决议系统。**
   本地 UI、coordinator、worker mailbox、channel relay、classifier 都可能参与，但最终通过 `claim()` 等机制收敛为单一结果。见 `src/hooks/useCanUseTool.tsx:93-167`、`src/hooks/toolPermission/PermissionContext.ts:63-93`、`src/hooks/toolPermission/handlers/swarmWorkerHandler.ts:67-146`、`src/services/mcp/channelPermissions.ts:4-7`。

这比“怀疑模型，所以加一个权限弹窗”要深得多。它体现的是：Claude Code 并不把安全寄托在模型自觉上，而是把权限判断拆成多个可验证、可竞争、可回退的程序化阶段。
