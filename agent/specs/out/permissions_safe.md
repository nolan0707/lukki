# Claude Code Open 权限、安全与隐私设计分析

本文以 `vendor/Claude-code-open/src/` 源码为准，重写 `./specs/read/permissions.md` 的概要分析。若概要与源码不一致，以本文和所引源码为准。

## 1. 总体结论

Claude Code Open 的安全设计不是单一的“是否允许”开关，而是分成几层串联执行：

1. 工具输入校验与工具内安全检查。
2. 通用权限管线：`deny -> ask -> tool-specific -> bypass/allow -> prompt/deny/classifier`。
3. 文件系统和 Bash 的专项防护。
4. 异步 Agent、多 Agent、远程会话、MCP 的额外授权桥接。
5. 隐私与数据最小化：日志脱敏、子进程环境变量清洗、本地秘密扫描。

核心入口与类型定义：

- 权限模式、规则、决策原因：`src/types/permissions.ts:16-44`, `src/types/permissions.ts:67-145`, `src/types/permissions.ts:171-324`, `src/types/permissions.ts:427-441`
- 通用权限判定主流程：`src/utils/permissions/permissions.ts:473-955`, `src/utils/permissions/permissions.ts:1060-1319`
- 工具执行时真正调用权限检查：`src/services/tools/toolExecution.ts:916-1138`
- 工具接口中区分 `validateInput` 与 `checkPermissions`：`src/Tool.ts:483-523`

## 2. 权限模式：源码中的实际定义

`./specs/read/permissions.md` 里写了“六种权限模式”，这和源码不完全一致。

源码中：

- 对外可见的模式是 `acceptEdits / bypassPermissions / default / dontAsk / plan`：`src/types/permissions.ts:16-24`
- 内部模式额外包含 `auto` 与 `bubble`：`src/types/permissions.ts:26-38`
- 其中 `auto` 只有在 `TRANSCRIPT_CLASSIFIER` 特性开启时才进入运行时可选集合：`src/types/permissions.ts:33-38`

含义可以从实现直接归纳：

- `default`：默认交互模式，遇到需要审批的工具调用会进入权限提示。
- `plan`：计划模式；本质上通过模式和专用工具控制执行边界，退出计划模式需要单独授权：`src/components/permissions/PermissionRequest.tsx:47-81`
- `acceptEdits`：对“编辑类动作”做快速放行，而不是全局放行。对 Bash，仅自动放行特定文件系统命令：`src/tools/BashTool/modeValidation.ts:7-21`, `src/tools/BashTool/modeValidation.ts:37-50`, `src/tools/BashTool/modeValidation.ts:111-115`
- `dontAsk`：不是“全盘 YOLO”，而是把最终 `ask` 结果转换成 `deny`：`src/utils/permissions/permissions.ts:503-517`
- `bypassPermissions`：跳过大部分权限提示，但仍受 deny 规则、内容级 ask 规则、安全检查约束：`src/utils/permissions/permissions.ts:1238-1260`, `src/utils/permissions/permissions.ts:1262-1281`
- `auto`：不是“自动批准所有操作”，而是把需要审批的行为交给分类器判断，允许被拦截、失败关闭或回退到人工审批：`src/utils/permissions/permissions.ts:518-927`
- `bubble`：主要用于 Agent 场景，表示权限提示可以冒泡到可显示 UI 的上层线程，而不是本地弹不出来：`src/tools/AgentTool/runAgent.ts:436-463`

## 3. 权限规则模型与来源

权限规则由“行为 + 工具名 + 可选内容模式”组成：

- 规则结构：`toolName + ruleContent?`：`src/types/permissions.ts:67-79`
- 支持的行为：`allow / deny / ask`：`src/types/permissions.ts:44`
- 规则来源：`userSettings / projectSettings / localSettings / flagSettings / policySettings / cliArg / command / session`：`src/types/permissions.ts:54-63`

设置源的顺序很重要，注释明确说明“后者覆盖前者”：

- `SETTING_SOURCES` 顺序：`userSettings -> projectSettings -> localSettings -> flagSettings -> policySettings`：`src/utils/settings/constants.ts:3-22`

规则加载与合并：

- 磁盘规则来自所有启用设置源；若开启 `allowManagedPermissionRulesOnly`，则只接受 `policySettings`：`src/utils/permissions/permissionsLoader.ts:27-44`, `src/utils/permissions/permissionsLoader.ts:116-145`
- 初始上下文通过 `applyPermissionRulesToPermissionContext` 做 additive 合并：`src/utils/permissions/permissions.ts:1405-1414`
- 动态同步设置变更时，会先清空磁盘来源再 replace，避免陈旧规则残留：`src/utils/permissions/permissions.ts:1416-1471`

一个重要的企业管控点：

- 若 `allowManagedPermissionRulesOnly=true`，普通用户来源的“总是允许”选项会被隐藏：`src/utils/permissions/permissionsLoader.ts:27-44`

## 4. 通用权限判定管线

真正的权限执行链路在 `hasPermissionsToUseTool` / `hasPermissionsToUseToolInner`：

- 入口：`src/utils/permissions/permissions.ts:473-955`
- 内层顺序：`src/utils/permissions/permissions.ts:1158-1319`

核心顺序是：

1. 整个工具是否被 deny：`src/utils/permissions/permissions.ts:1169-1181`
2. 整个工具是否被 ask：`src/utils/permissions/permissions.ts:1183-1206`
3. 调用工具自己的 `checkPermissions`：`src/utils/permissions/permissions.ts:1208-1227`
4. 若工具要求必须交互，则即便 bypass 也保留 ask：`src/utils/permissions/permissions.ts:1230-1236`
5. 内容级 ask 规则优先于 bypass：`src/utils/permissions/permissions.ts:1238-1250`
6. `safetyCheck` 类型的安全检查优先于 bypass：`src/utils/permissions/permissions.ts:1252-1260`
7. 只有到这一步，`bypassPermissions` 才真正放行：`src/utils/permissions/permissions.ts:1262-1281`
8. 否则再看 entire-tool allow 规则：`src/utils/permissions/permissions.ts:1283-1297`
9. 剩余 `passthrough` 才被转成 `ask`：`src/utils/permissions/permissions.ts:1299-1318`

工具执行层会在真正调用工具前统一走这条链：

- `toolExecution.ts` 在执行前先 `resolveHookPermissionDecision` / `canUseTool`，然后再执行或返回拒绝结果：`src/services/tools/toolExecution.ts:916-1104`

## 5. `validateInput` 与 `checkPermissions` 的职责分离

工具接口本身已明确区分两类检查：

- `validateInput`：输入合法性、预检、安全约束：`src/Tool.ts:483-493`
- `checkPermissions`：是否需要用户授权：`src/Tool.ts:494-503`

这意味着很多安全能力不是“弹窗审批”而是更早拦截。例如：

- FileWrite 会先检查 team memory 秘密、deny 规则、UNC 风险、是否先读后写、文件是否被外部修改：`src/tools/FileWriteTool/FileWriteTool.ts:153-221`
- FileEdit 会先检查 team memory 秘密、deny 规则、UNC 风险、文件大小、文件存在性、old/new string 合法性：`src/tools/FileEditTool/FileEditTool.ts:137-260`

## 6. 文件系统权限与敏感路径防护

### 6.1 敏感文件/目录清单

源码直接定义了危险文件与目录：

- 危险文件：`.gitconfig`, `.gitmodules`, shell rc, `.mcp.json`, `.claude.json`：`src/utils/permissions/filesystem.ts:54-68`
- 危险目录：`.git`, `.vscode`, `.idea`, `.claude`：`src/utils/permissions/filesystem.ts:70-79`

同时，路径比较统一做大小写归一化，防止在大小写不敏感文件系统上绕过：`src/utils/permissions/filesystem.ts:81-92`

### 6.2 写权限判定的真实顺序

文件写入权限的关键路径在 `checkWritePermissionForTool` 所依赖的逻辑中，摘出关键阶段：

- 内部可编辑路径（plan 文件、scratchpad、agent memory、job dirs）先放行：`src/utils/permissions/filesystem.ts:1241-1250`
- 只允许“session 级”的 `.claude/**` 规则绕过 `.claude` 安全拦截，防止永久授予过宽权限：`src/utils/permissions/filesystem.ts:1252-1300`
- 然后才做综合安全检查：`src/utils/permissions/filesystem.ts:1302-1338`

这个顺序很关键：并不是任何 allow 规则都能越过 `.claude/`、`.git/`、shell 配置等安全边界。

### 6.3 安全检查内容

`checkPathSafetyForAutoEdit()` 同时检查原始路径和 symlink 解析路径：

- 防 Windows 可疑路径模式：`src/utils/permissions/filesystem.ts:620-639`
- 防 Claude 自身配置文件被无提示修改：`src/utils/permissions/filesystem.ts:641-649`
- 防敏感文件直接编辑：`src/utils/permissions/filesystem.ts:652-660`
- 显式说明会检查 symlink 解析路径，避免通过符号链接绕过：`src/utils/permissions/filesystem.ts:614-665`

此外，路径校验辅助逻辑也强调：

- 写操作的安全校验发生在 working directory 判断之前，防止被 `acceptEdits` 或工作目录范围绕过：`src/utils/permissions/pathValidation.ts:164-210`

## 7. Bash / PowerShell 的安全设计

### 7.1 `acceptEdits` 不是 Bash 全放开

在 `acceptEdits` 模式下，Bash 仅对有限文件系统命令做模式级自动放行：

- `mkdir / touch / rm / rmdir / mv / cp / sed`：`src/tools/BashTool/modeValidation.ts:7-21`
- 自动放行逻辑：`src/tools/BashTool/modeValidation.ts:37-50`

### 7.2 只读命令允许列表

Bash 的“只读安全命令”不是靠简单字符串匹配，而是显式 allowlist + flag 解析：

- allowlist 获取：`src/tools/BashTool/readOnlyValidation.ts:1201-1215`
- `xargs` 目标命令白名单与安全约束：`src/tools/BashTool/readOnlyValidation.ts:1217-1239`
- 统一 flag 解析验证：`src/tools/BashTool/readOnlyValidation.ts:1241-1304`
- 针对 `git ls-remote` 还额外防止 URL/SSH/变量注入导致外联：`src/tools/BashTool/readOnlyValidation.ts:1306-1325`

### 7.3 破坏性动作提示只是提示，不是强制规则

源码中有“破坏性命令警告”，但它明确只是信息提示，不影响权限决策：

- `destructiveCommandWarning.ts` 顶部注释：`src/tools/BashTool/destructiveCommandWarning.ts:1-5`
- 例如识别 `git reset --hard`、`git push --force`、`rm -rf`、`terraform destroy`：`src/tools/BashTool/destructiveCommandWarning.ts:12-102`

### 7.4 Auto mode 下对危险 shell allow 规则做削减

源码对 auto mode 的一个关键安全设计是：会主动剥离会绕过分类器的危险 allow 规则。

- Bash 全量 allow 或解释器前缀 allow 视为危险：`src/utils/permissions/permissionSetup.ts:84-147`
- PowerShell 同理，特别防 `iex` / `start-process` / `pwsh` 等：`src/utils/permissions/permissionSetup.ts:149-233`
- `Agent(*)` 也被视为危险，因为会绕过对子 Agent 提示词的分类器审查：`src/utils/permissions/permissionSetup.ts:235-245`
- 进入 auto mode 时会 strip，退出 auto mode 再 restore：`src/utils/permissions/permissionSetup.ts:505-579`, `src/utils/permissions/permissionSetup.ts:597-646`

## 8. Auto mode 的真实安全模型

源码中的 `auto` 不是“自动批准模式”，而是“分类器替代部分人工审批”：

- UI 文案直接说明 Claude 会检查 risky actions 和 prompt injection，且可能出错，建议只在隔离环境使用：`src/components/AutoModeOptInDialog.tsx:9-10`

实现上：

- 只有 `behavior === ask` 的动作才进入 auto mode 分类器路径：`src/utils/permissions/permissions.ts:518-525`
- 不可分类器批准的 `safetyCheck` 仍然保留人工审批，异步无 UI 场景则直接拒绝：`src/utils/permissions/permissions.ts:526-548`
- 一部分安全工具可直接跳过分类器 API 调用（safe allowlist）：`src/utils/permissions/classifierDecision.ts:50-98`
- 分类器不可用时，支持 fail-closed 或回退人工审批：`src/utils/permissions/permissions.ts:843-876`
- 连续/累计拒绝过多，会强制回退人工复核；headless 场景则中止 Agent：`src/utils/permissions/permissions.ts:878-1058`

所以，auto mode 的设计目标是“减少人工审批频率”，不是“弱化安全边界”。

## 9. 人工授权机制

### 9.1 交互式权限弹窗

权限请求组件按工具类型分发：

- Bash、PowerShell、FileEdit、FileWrite、NotebookEdit、WebFetch、Skill、AskUserQuestion、Plan Mode 进出等都有专门权限 UI：`src/components/permissions/PermissionRequest.tsx:47-81`

用户交互后可以：

- 仅本次允许
- 应用建议规则后允许
- 对 Bash 保存特定 prefix allow
- 拒绝并给反馈

可见于 Bash 权限请求实现：`src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx:337-425`

### 9.2 “总是允许”不是任意持久化

源码中“总是允许”的持久化粒度受严格限制：

- Bash prefix allow 可落到 `localSettings`：`src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx:348-357`
- classifier reviewed 规则只落 `session`：`src/components/permissions/BashPermissionRequest/BashPermissionRequest.tsx:368-377`
- `.claude/**` 绕过安全拦截也只接受 `session` 级规则：`src/utils/permissions/filesystem.ts:1252-1300`

### 9.3 网络越沙箱授权

当 Bash 想访问沙箱外网络时，会出现单独的沙箱网络权限对话框：

- UI 允许：本次允许 / 持久允许 / 拒绝：`src/components/permissions/SandboxPermissionRequest.tsx:24-50`, `src/components/permissions/SandboxPermissionRequest.tsx:77-161`
- 文案明确区分“Allow unsandboxed fallback” 与 “Strict sandbox mode”：`src/components/sandbox/SandboxOverridesTab.tsx:162-178`

### 9.4 非交互/SDK 权限提示工具

SDK 模式下，可以指定一个 MCP 工具作为 permission prompt tool：

- CLI 参数：`--permission-prompt-tool`：`src/main.tsx:976-1000`
- `createCanUseToolWithPermissionPrompt()` 将这个 MCP 工具包装成权限决策器：`src/cli/print.ts:4145-4248`

这说明 Claude Code 的授权面板不仅可以是终端 UI，也可以被宿主集成接管。

## 10. Agent 的工具集合与权限隔离

### 10.1 全局工具池

基础工具池在 `src/tools.ts` 统一声明：

- 所有 base tools 来源：`src/tools.ts:173-250`

### 10.2 Agent 的默认裁剪规则

Agent 工具过滤规则来源于 `constants/tools.ts` 与 `agentToolUtils.ts`：

- 所有 Agent 默认禁用：`TaskOutput / ExitPlanMode / EnterPlanMode / AskUserQuestion / TaskStop / Workflow`，外部版还禁用 Agent 本身：`src/constants/tools.ts:36-50`
- 异步 Agent 只允许一组有限工具：读、搜、Web、Shell、Edit/Write、Skill、Worktree 等：`src/constants/tools.ts:52-71`
- 协调器模式只允许 agent 管理与输出工具：`src/constants/tools.ts:104-112`
- 实际过滤逻辑：`src/tools/AgentTool/agentToolUtils.ts:70-116`

### 10.3 内建 Agent 的差异

源码内建了多种不同安全姿态的 Agent：

- `Explore`：显式 read-only，禁用 `Agent / ExitPlanMode / Edit / Write / NotebookEdit`：`src/tools/AgentTool/built-in/exploreAgent.ts:24-57`, `src/tools/AgentTool/built-in/exploreAgent.ts:64-82`
- `Plan`：同样 read-only，禁用编辑类工具：`src/tools/AgentTool/built-in/planAgent.ts:21-70`, `src/tools/AgentTool/built-in/planAgent.ts:73-91`
- `verification`：后台运行，但系统提醒中声明“项目目录不可编辑，tmp 仅限临时脚本”：`src/tools/AgentTool/built-in/verificationAgent.ts:131-152`

### 10.4 Agent 权限不继承父线程全部 allow

`runAgent()` 里有一个非常重要的隔离点：

- 若 `allowedTools` 被明确传入，会用它重建 agent 的 `session` allow 规则
- 仅保留 SDK 级 `cliArg` allow，不继承父线程的 session allow

实现：`src/tools/AgentTool/runAgent.ts:465-479`

源码注释写得很清楚：这是为了防止“parent approvals don't leak through”。

### 10.5 Agent 本身的权限判定

AgentTool 并不是总要审批：

- 非 auto mode 下，`AgentTool.checkPermissions()` 直接 `allow`：`src/tools/AgentTool/AgentTool.tsx:1281-1297`
- auto mode 下才把子 Agent 生成交给分类器/上层权限链继续处理：同上

这也解释了为什么 `Agent(*)` allow 规则在 auto mode 里会被当作危险规则剥离。

## 11. 多 Agent 与人工授权转发

多 Agent 协作不是“每个 worker 独立弹窗”，而是 leader 汇总审批。

### 11.1 邮箱式权限同步

`permissionSync.ts` 明确描述了 worker -> leader -> worker 的邮箱式流程：

- 设计说明：`src/utils/swarm/permissionSync.ts:1-19`
- worker 写入 pending permission request：`src/utils/swarm/permissionSync.ts:46-85`, `src/utils/swarm/permissionSync.ts:209-250`
- leader 通过邮箱回写审批结果：`src/utils/swarm/permissionSync.ts:760-783`
- sandbox 网络权限同样走 mailbox：`src/utils/swarm/permissionSync.ts:785-928`

这和 `./specs/read/permissions.md` 提到的“信箱审批”方向一致，但源码里实现为明确的 mailbox 消息和文件/锁机制，不是抽象概念。

### 11.2 远程会话权限桥

远程 CCR 会话的权限请求也被显式桥接：

- `RemoteSessionManager` 接收 `control_request`，把 `can_use_tool` 交给本地回调：`src/remote/RemoteSessionManager.ts:64-85`, `src/remote/RemoteSessionManager.ts:146-214`
- 用户决策后再发送 `control_response`：`src/remote/RemoteSessionManager.ts:245-282`

这说明远程场景的权限并没有被省略，而是通过协议桥接回本地审批。

## 12. 沙箱与 bypass 模式的防护

### 12.1 bypass 不是无限制

`--dangerously-skip-permissions` 有启动期安全门：

- root/sudo 环境禁止：`src/setup.ts:395-414`
- ant 内部版要求运行在 Docker/bwrap/沙箱内且无互联网：`src/setup.ts:416-441`
- 启动后若检测到 bypass mode，还会异步再校验并可能关闭：`src/main.tsx:2655-2659`

### 12.2 沙箱仍然有硬编码 deny

沙箱适配器不是单纯照搬权限规则，而是附加操作系统级 deny：

- 始终禁止写 settings 文件与 managed settings 目录：`src/utils/sandbox/sandbox-adapter.ts:230-245`
- 补充禁止写 `.claude/skills`：`src/utils/sandbox/sandbox-adapter.ts:247-255`
- 针对 bare git repo 伪造 / `core.fsmonitor` 类逃逸场景做特殊处理：`src/utils/sandbox/sandbox-adapter.ts:257-280`
- 额外工作目录和 settings 里的 fs allow/deny 会被并入 sandbox 配置：`src/utils/sandbox/sandbox-adapter.ts:290-348`

### 12.3 网络访问是单独授权面

SDK/宿主桥接网络越沙箱权限时，使用了一个 synthetic tool name，把网络访问当成普通权限请求发送给宿主：

- `createSandboxAskCallback()`：`src/cli/structuredIO.ts:724-753`

这说明网络权限是独立于“文件/命令权限”的另一条授权链路。

## 13. 隐私与敏感数据处理

### 13.1 分析日志默认脱敏

源码对分析元数据有显式“不得包含代码或文件路径”的开发约束：

- 类型标记：`src/services/analytics/metadata.ts:44-57`

具体策略：

- MCP 工具名默认脱敏为 `mcp_tool`：`src/services/analytics/metadata.ts:59-77`
- 只有官方 MCP、claude.ai proxy、local-agent 等场景才允许记录 MCP 细节：`src/services/analytics/metadata.ts:90-167`
- 工具参数日志默认关闭，仅 `OTEL_LOG_TOOL_DETAILS=1` 才打开：`src/services/analytics/metadata.ts:79-88`, `src/services/tools/toolExecution.ts:1134-1157`

### 13.2 文件操作分析使用哈希

文件读写编辑分析不会直接上传路径与内容：

- 路径使用截断 SHA256：`src/utils/fileOperationAnalytics.ts:5-16`
- 内容仅做 SHA256，且限制最大 100KB：`src/utils/fileOperationAnalytics.ts:18-33`
- 实际上报的是 hash 而非明文：`src/utils/fileOperationAnalytics.ts:37-70`

### 13.3 子进程环境变量清洗

为了防止 prompt injection 借 Bash/MCP/LSP/Hook 子进程窃取密钥，源码支持对子进程 env 做敏感变量剥离：

- 设计目的：`src/utils/subprocessEnv.ts:3-14`
- 被清洗的包括 Anthropic/OAuth/OTLP/AWS/GCP/Azure/GitHub Actions OIDC 等：`src/utils/subprocessEnv.ts:15-53`
- 作用范围包括 Bash、shell snapshot、MCP stdio、LSP、hooks：`src/utils/subprocessEnv.ts:55-99`

### 13.4 team memory 本地秘密扫描

当写入 team memory 时，Claude Code 试图保证秘密不离开本机：

- `checkTeamMemSecrets()` 在 FileWrite/FileEdit 的 `validateInput` 中调用：`src/services/teamMemorySync/teamMemSecretGuard.ts:4-13`, `src/tools/FileWriteTool/FileWriteTool.ts:153-160`, `src/tools/FileEditTool/FileEditTool.ts:137-147`
- secret scanner 明确声明“before upload so secrets never leave the user's machine”：`src/services/teamMemorySync/secretScanner.ts:1-19`
- 规则来自高置信度 gitleaks 子集：`src/services/teamMemorySync/secretScanner.ts:39-224`

### 13.5 `--bare` 最小暴露模式

CLI 的 `--bare` 会跳过 hooks、LSP、plugin sync、auto-memory、keychain 读取、CLAUDE.md 自动发现等：

- 参数说明：`src/main.tsx:976-1006`

这是一种很明确的数据最小化/环境最小化开关。

## 14. 动作分级：源码里的真实含义

如果把“动作分级”理解为统一执行级别枚举，源码并没有一个全局的 `LOW/MEDIUM/HIGH/CRITICAL` 强制策略引擎。

源码里实际存在三类“分级”：

1. 强制执行级别：`allow / ask / deny`，以及 `mode/rule/safetyCheck/classifier` 等决策原因：`src/types/permissions.ts:44`, `src/types/permissions.ts:171-324`
2. 解释性风险等级：`LOW / MEDIUM / HIGH`，由 permission explainer 生成，服务于 UI 展示：`src/utils/permissions/permissionExplainer.ts:14-33`, `src/utils/permissions/permissionExplainer.ts:43-84`, `src/components/permissions/PermissionExplanation.tsx:41-59`
3. 信息性破坏性提示：例如 destructive bash warning，只提示、不参与决策：`src/tools/BashTool/destructiveCommandWarning.ts:1-5`

因此，`./specs/read/permissions.md` 中“LOW/MEDIUM/HIGH 决定审批逻辑”的说法并不准确。真正决定审批逻辑的是权限管线和工具自身安全检查，不是 `riskLevel`。

## 15. 与现有概要的主要偏差

对照 `./specs/read/permissions.md`，需要修正的点包括：

- “六种权限模式”不准确。源码外显模式是 5 个，`auto` 是受特性控制的内部扩展模式，`bubble` 也是内部模式：`src/types/permissions.ts:16-38`
- `dontAsk` 不是“YOLO”，而是 ask->deny：`src/utils/permissions/permissions.ts:503-517`
- “风险分级决定审批逻辑”不准确。真正执行的是 `allow/ask/deny + decisionReason`，`LOW/MEDIUM/HIGH` 主要用于解释 UI：`src/types/permissions.ts:239-324`, `src/utils/permissions/permissionExplainer.ts:14-33`
- “Bash AST 审计 + sleep > 2 秒一律拦截”这类表述过度概括。源码确实有大量 Bash 安全校验与 allowlist/flag 解析，但当前证据更明确地体现在 read-only 校验、路径校验、模式校验、destructive warning、sandbox 约束上，而不是一个单独的“统一 AST 审计入口”足以概括全部逻辑。
- “多 Agent 信箱审批”方向正确，但源码实现是 mailbox + request/response + 文件锁/目录结构，而不是仅抽象模式：`src/utils/swarm/permissionSync.ts:1-19`, `src/utils/swarm/permissionSync.ts:209-250`, `src/utils/swarm/permissionSync.ts:760-928`

## 16. 最终判断

从源码看，Claude Code Open 的安全模型有几个鲜明特点：

- 安全边界前移：很多风险在 `validateInput` 或 `safetyCheck` 阶段就被拦下，不等用户点按钮。
- 模式不是万能钥匙：即便 `bypassPermissions` 也不能绕过 deny、内容 ask 和敏感路径安全检查。
- Auto mode 是“分类器辅助审批”，不是“自动全放行”。
- Agent 权限是收缩而不是继承，尤其异步 Agent 和多 Agent 场景有额外隔离。
- 隐私保护不是一句口号，源码中确实落了日志脱敏、哈希、子进程环境清洗、本地 secrets 扫描。

如果要用一句话概括：Claude Code Open 的权限系统更接近“分层强制执行 + 受控自动化 + 可桥接人工授权”的治理框架，而不是简单的工具白名单。
