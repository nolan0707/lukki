# 以架构视角，深入源码，逐层拆解 Claude Code 的 Bash Harness 设计哲学

## 结论先行

`bash_sandbox.md` 现有概要把三件事混成了一层：

1. `utils/bash/*` 的 AST/语义分析层：负责判断“这条命令能不能被可靠理解”，不是运行时沙箱。源码自己就明确写了：`This is NOT a sandbox.`（`vendor/Claude-code-open/src/utils/bash/ast.ts:1-19`）
2. `BashTool` 的权限治理层：负责 `Zod` 结构校验、额外输入校验、模式判定、规则匹配、路径/危险命令审查，决定 `allow / ask / deny`。（`vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-687`，`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1663-1843`）
3. `sandbox-runtime` 运行时隔离层：真正把命令包进 OS 级沙箱，限制文件系统和网络。（`vendor/Claude-code-open/src/utils/Shell.ts:259-273`，`vendor/Claude-code-open/src/utils/sandbox/sandbox-adapter.ts:222-380`）

因此，更准确的说法不是“BashTool 内部有一个 AST 沙箱”，而是“Claude Code 用 AST 让权限判断 fail-closed，用权限系统决定是否放行，再由 sandbox-runtime 做最终隔离”。

补充一个量级感：仅 `utils/bash/*.ts` 与 `tools/BashTool/*.ts` 合计就有 `22,987` 行代码，确实是一个重型治理子系统，而不是几条正则。（`wc -l vendor/Claude-code-open/src/utils/bash/*.ts vendor/Claude-code-open/src/tools/BashTool/*.ts`）

---

## 一、第一层：命令理解层，不是沙箱，而是 fail-closed 的 AST 安全分析

### 1.1 tree-sitter 是 Bash Harness 的核心底座之一

- `parseCommandRaw()` 通过 tree-sitter 解析 Bash；当模块未加载、特性关闭、命令过长时返回 `null`；当解析超时/资源耗尽时返回 `PARSE_ABORTED`，要求调用方按 fail-closed 处理。（`vendor/Claude-code-open/src/utils/bash/parser.ts:19-20, 46-54, 93-135`）
- `parseForSecurity()` / `parseForSecurityFromAst()` 是 Bash 权限判断前的核心安全入口。（`vendor/Claude-code-open/src/utils/bash/ast.ts:381-460`）
- 这层的设计目标不是“阻止执行”，而是回答“能否可靠提取可信的 argv 结构”。源码原文：`This is NOT a sandbox. It does not prevent dangerous commands from running.`（`vendor/Claude-code-open/src/utils/bash/ast.ts:15-18`）

### 1.2 设计哲学：只理解白名单语法，超出理解能力就要求人工确认

- `ast.ts` 文件头直接说明：它不再依赖 shell-quote + 手工字符扫描，而是“用 tree-sitter 解析，再以显式 allowlist 的 node type 白名单做遍历”；任何未显式允许的结构都会被判为 `too-complex`，进入正常权限询问流。（`vendor/Claude-code-open/src/utils/bash/ast.ts:2-19`）
- `parseForSecurityFromAst()` 先做一轮 parser differential 预检查，发现控制字符、Unicode 空白、反斜杠转义空白、zsh 动态目录语法、zsh equals expansion、带引号的大括号扩展混淆等情况，直接判为 `too-complex`。（`vendor/Claude-code-open/src/utils/bash/ast.ts:404-437`）
- 如果 parser 自己 abort，也不是回退成“当没看见”，而是直接 `too-complex -> ask`。（`vendor/Claude-code-open/src/utils/bash/ast.ts:444-456`）

### 1.3 AST 之后还有语义层，不只看“能不能 parse”

- `checkSemantics()` 会在 AST 已经判定为 `simple` 后，再检查语义级危险包装器和内建命令。（`vendor/Claude-code-open/src/utils/bash/ast.ts:2209-2325`）
- 它会剥掉 `time`、`nohup`、`timeout`、`nice`、`env` 这些包装器，再检查真正的内部命令，避免 `timeout 10 rm -rf /`、`nohup eval ...` 之类包装绕过。（`vendor/Claude-code-open/src/utils/bash/ast.ts:2215-2325`）
- 它显式拉黑一批“逃逸 argv 抽象”的 shell builtin，例如 `eval`、`source`、`.`、`exec`、`command`、`builtin`、`fc`、`coproc`、`trap`、`enable`、`hash`、`bind`、`complete`、`compgen`、`alias` 等。（`vendor/Claude-code-open/src/utils/bash/ast.ts:2080-2130`）

### 1.4 这层能防什么

- 不是简单关键字匹配，而是结构级识别。
- 能处理“包装器包裹真实命令”“复合命令中真实危险子命令”“parser differential”“过于复杂的命令结构”。
- 对于无法可靠理解的命令，不会乐观放过，而是降级为 `ask`。

---

## 二、第二层：权限治理层，真正决定 allow / ask / deny

### 2.1 先过 schema，再过 validateInput，再进权限系统

- 工具执行入口先用 `tool.inputSchema.safeParse(input)` 做 `Zod` 校验；失败直接返回 `InputValidationError`。（`vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-680`）
- 之后调用每个工具自己的 `validateInput()`；对 Bash 来说，`BashTool.validateInput()` 目前最明确的一条策略是拦截前导 `sleep N` 且 `N >= 2` 的阻塞式等待。（`vendor/Claude-code-open/src/services/tools/toolExecution.ts:682-687`，`vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:317-337, 524-537`）
- 只有这些都通过后，才会进入 `checkPermissions()`。（`vendor/Claude-code-open/src/Tool.ts:495-500`，`vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:539-541`）

### 2.2 Bash 权限主流程：AST 结果直接参与 permission decision

- `bashToolHasPermission()` 是 Bash 权限判定主入口。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1663-1843`）
- 它先做 AST 安全解析：`parseCommandRaw()` + `parseForSecurityFromAst()`。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1670-1699`）
- AST 结果如果是 `too-complex`，会优先检查 deny 规则；没有命中 deny 时，直接返回 `ask`。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1741-1768`）
- AST 结果如果是 `simple`，还会跑 `checkSemantics()`；语义层失败同样进入 `ask` 或 `deny`。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1771-1794`）
- 只有 tree-sitter 不可用或被 gate 关闭，才回落到 legacy shell-quote 路径。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1808-1827`）

### 2.3 运行模式不是“三层 + 六种”那样简单

以源码为准：

- 外部可见模式是 5 种：`acceptEdits`、`bypassPermissions`、`default`、`dontAsk`、`plan`。（`vendor/Claude-code-open/src/types/permissions.ts:16-22`）
- 内部运行时模式会在开启 `TRANSCRIPT_CLASSIFIER` 时额外加入 `auto`；类型层还有 `bubble`，但它不是用户可配置的常规模式。（`vendor/Claude-code-open/src/types/permissions.ts:26-38`）
- `PermissionMode.ts` 给这些模式配了 title / color / external 映射。（`vendor/Claude-code-open/src/utils/permissions/PermissionMode.ts:42-91`）
- `acceptEdits` 模式下，`mkdir`、`touch`、`rm`、`rmdir`、`mv`、`cp`、`sed` 这类文件系统操作可自动放行。（`vendor/Claude-code-open/src/tools/BashTool/modeValidation.ts:7-20, 37-50, 72-109`）

所以，现有概要里的“三层权限模型 + 六种权限模式”并不准确。更准确的表述是：

- 行为结果层：`allow / ask / deny`
- 风险说明层：`LOW / MEDIUM / HIGH`
- 模式层：5 个外部模式，外加特性门控下的 `auto`

### 2.4 `LOW / MEDIUM / HIGH` 不是 Bash 独占的执行门，而是权限解释器输出

- 风险级别定义在 `permissionExplainer.ts`，是用来解释命令风险的元信息。（`vendor/Claude-code-open/src/utils/permissions/permissionExplainer.ts:14-33, 43-84`）
- 它不是 BashTool 主流程里唯一的“风险分级裁决器”；真正的放行逻辑仍然是 `allow / ask / deny`。

### 2.5 sandbox 开启后，Bash 还能被“自动允许”，但前提是规则链先没拦住

- 若 sandbox 已启用，且 `autoAllowBashIfSandboxed` 打开，`checkSandboxAutoAllow()` 会在显式 deny/ask 规则之后执行；无明确规则时可直接 `allow`，理由是 `Auto-allowed with sandbox`。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:1829-1843, 1349-1358`）
- 这说明 Claude Code 的思路不是“凡 Bash 必高危”，而是“若运行时沙箱足够强，且无规则冲突，则允许自动执行”。

---

## 三、第三层：运行时沙箱层，真正的物理隔离在这里

### 3.1 BashTool 最终是通过 SandboxManager 包装命令执行

- `runShellCommand()` 在 `shouldUseSandbox(input)` 为真时，会调用 `SandboxManager.wrapWithSandbox()` 包装命令。（`vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:881-898`，`vendor/Claude-code-open/src/utils/Shell.ts:259-273`）
- `shouldUseSandbox()` 会考虑：
  - sandbox 是否启用
  - 是否显式设置 `dangerouslyDisableSandbox`
  - 策略是否允许 unsandboxed command
  - 命令是否命中 `sandbox.excludedCommands`（注意源码明确说这不是 security boundary，而只是用户便利特性）。（`vendor/Claude-code-open/src/tools/BashTool/shouldUseSandbox.ts:18-20, 130-152`）

### 3.2 sandbox-adapter 才是真正拼装文件系统/网络隔离规则的地方

- `sandbox-adapter.ts` 会构造 `allowWrite / denyWrite / allowRead / denyRead / allowedDomains / deniedDomains`，最后交给 sandbox-runtime。（`vendor/Claude-code-open/src/utils/sandbox/sandbox-adapter.ts:222-380`）
- 它会默认阻止写入 Claude settings 文件，防止借配置文件逃逸。（`vendor/Claude-code-open/src/utils/sandbox/sandbox-adapter.ts:230-245`）
- 它还会显式阻止写入 `.claude/skills`，因为 skills 与 commands/agents 一样具备高权限能力。（`vendor/Claude-code-open/src/utils/sandbox/sandbox-adapter.ts:247-255`）
- 它还针对 bare git repo 注入做了额外保护，避免后续 unsandboxed git 被恶意 hooks 利用。（`vendor/Claude-code-open/src/utils/sandbox/sandbox-adapter.ts:257-279`）

这才是“物理边界”层。

---

## 四、路径与文件安全：这部分确实很重，但不只属于 Bash

### 4.1 敏感文件保护是明确存在的

- `DANGEROUS_FILES` 明确包含：`.gitconfig`、`.gitmodules`、`.bashrc`、`.bash_profile`、`.zshrc`、`.zprofile`、`.profile`、`.ripgreprc`、`.mcp.json`、`.claude.json`。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:53-68`）
- `DANGEROUS_DIRECTORIES` 明确包含：`.git`、`.vscode`、`.idea`、`.claude`。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:70-79`）
- `checkPathSafetyForAutoEdit()` 会在 auto-edit 场景同时检查原始路径和 symlink-resolved 路径；一旦命中 Claude config 或敏感文件，直接判为 unsafe。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:620-665`）
- 而且这道安全检查刻意放在 allow 规则之前，避免用户误把敏感文件纳入自动编辑范围。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:1302-1306`）

### 4.2 路径遍历、大小写绕过、Windows 路径花活都考虑到了

- `normalizeCaseForComparison()` 统一把路径转为小写，防止大小写混合绕过，例如 `.cLauDe/Settings.locaL.json`。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:82-91`）
- `pathInWorkingPath()` 在比较路径是否位于允许工作目录内时，会做大小写归一，并通过 `containsPathTraversal(relative)` 拦截 `..` 穿越。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:709-744`）
- `isScratchpadPath()` 等函数在做 startsWith 判定前，也先 `normalize()` 以消除 `../../../` 绕过。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:409-423`）
- `hasSuspiciousWindowsPathPattern()` 会额外拦截 ADS、8.3 短文件名、长路径前缀、尾随点/空格、DOS 设备名、连续三个点、UNC/WebDAV 风险路径等。（`vendor/Claude-code-open/src/utils/permissions/filesystem.ts:490-602`）

### 4.3 Bash 自身也对路径危险操作做了专项检查

- `checkPathConstraints()` 会检查重定向目标、process substitution、路径型命令参数。（`vendor/Claude-code-open/src/tools/BashTool/pathValidation.ts:1005-1105`）
- 其中 process substitution（如 `>(cmd)` / `<(cmd)`）会直接要求人工批准。（`vendor/Claude-code-open/src/tools/BashTool/pathValidation.ts:1021-1037`）
- 对 `rm` / `rmdir`，还会走 `checkDangerousRemovalPaths()`；根目录、盘符根、家目录、根目录一级子目录、通配删除等都会触发危险删除保护。（`vendor/Claude-code-open/src/tools/BashTool/pathValidation.ts:728-737`，`vendor/Claude-code-open/src/utils/permissions/pathValidation.ts:331-360`）

所以，概要里“严禁 `rm -rf /`”这个方向是对的，但实现位置更准确地说是在“路径危险操作检查”而不是“单独的 Bash sandbox 黑名单”。

---

## 五、哪些说法与源码不符，或者证据不足

### 5.1 `partiallySanitizeUnicode()` 存在，但不在 Bash 沙箱主链路

- `partiallySanitizeUnicode()` 确实存在，负责对 Unicode 做 NFKC 归一和隐藏字符清理。（`vendor/Claude-code-open/src/utils/sanitization.ts:1-65`）
- 但我在 BashTool 主链路里没有看到它被直接接入 `BashTool.validateInput()`、`bashToolHasPermission()`、`checkPathConstraints()` 或 `SandboxManager.wrapWithSandbox()`。
- Bash 命令防 Unicode/parser differential 的主路径，实际在 `parseForSecurityFromAst()` 对 `UNICODE_WHITESPACE_RE` 等模式的预检查。（`vendor/Claude-code-open/src/utils/bash/ast.ts:404-437`）

因此，不能把“Bash 安全依赖 partiallySanitizeUnicode”写成事实。

### 5.2 “专门拦截 fork bomb、TTY 注入、history 操纵”目前没有找到同等级源码证据

- 我在 `tools/BashTool/*`、`utils/bash/*`、`utils/permissions/*` 中没有找到与 fork bomb 显式匹配的实现。
- 也没有找到能支持“专门拦截 TTY 注入 / history 操纵”的直接 Bash 权限逻辑。
- 能找到的是：
  - 对 `trap`、`bind`、`complete`、`compgen`、`hash`、`enable` 等 shell 级逃逸/持久化相关 builtin 的语义拦截。（`vendor/Claude-code-open/src/utils/bash/ast.ts:2086-2128`）
  - 对 `sudo`、`doas`、`pkexec` 这类提权包装器，不允许把它们变成宽泛的 prefix suggestion；这是权限规则建议层的保守设计，不等于“无条件禁止执行 sudo”。（`vendor/Claude-code-open/src/tools/BashTool/bashPermissions.ts:190-226`）

所以分享里最好表述为：**源码能证实其对 shell builtin 逃逸、提权包装器建议、路径危险删除、process substitution、复杂语法差异做了很多硬化；但“fork bomb / TTY 注入 / history 操纵专项拦截”当前版本没有直接证据。**

### 5.3 `sleep >= 2s` 的限制是存在的，但语义上更像“阻塞预算治理”

- `detectBlockedSleepPattern()` 明确只抓“前导且独立”的 `sleep N`，并且只在 `N >= 2` 时阻止。（`vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:317-337`）
- 这个限制挂在 `validateInput()`，给出的建议是“放后台”或“用 Monitor tool”，所以它更像响应性/资源治理，而不是 Bash AST 沙箱的一部分。（`vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:524-533`）

---

## 六、可以在分享中使用的最终表述

可以把 Claude Code 的 Bash Harness 总结为三层协同：

### 6.1 第 1 层：可理解性优先的 AST 审查

- 先问“我能否可靠理解这条命令？”
- 能理解，才继续做精细权限匹配和路径分析。
- 不能理解，就 fail-closed，进入人工确认。

### 6.2 第 2 层：权限系统作为策略中枢

- 先做 `Zod` 结构校验和业务级 `validateInput`
- 再按模式、规则、路径、危险操作、语义分析结果决定 `allow / ask / deny`
- sandbox 开启后还能结合 `autoAllowBashIfSandboxed` 做自动放行

### 6.3 第 3 层：sandbox-runtime 作为最终隔离边界

- 即使权限链允许，命令仍默认在 OS 级沙箱里执行
- 文件系统和网络访问由 sandbox 配置决定
- Claude settings、skills、bare git repo 等高价值资产被额外保护

---

## 七、面向分享的设计哲学提炼

如果要用一句话概括 Claude Code 的 Harness 设计哲学，可以这样说：

> Claude Code 并不把安全寄托在模型“别做坏事”上，而是把 Bash 执行拆成“先理解、再裁决、后隔离”三层：AST 分析负责 fail-closed 地理解命令，权限系统负责基于上下文做策略判定，sandbox-runtime 负责提供最后的物理隔离。

再进一步压缩成三个关键词：

- `Fail-closed parsing`
- `Policy-first permissioning`
- `Runtime sandbox isolation`

