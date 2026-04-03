# Claude-code-open 权限与安全架构深度解读

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 的“权限与安全架构”。

目标不是简单回答“它怎么弹确认框”，而是帮助初学者真正理解下面这些问题：

1. 为什么这个项目的权限控制不只是一个“是否允许执行工具”的布尔判断？
2. 一次工具调用到底会经过哪些安全关卡？
3. `PermissionMode`、权限规则、路径校验、审批弹窗、沙箱之间是什么关系？
4. 为什么既有应用层权限判断，又有操作系统层沙箱？
5. 远程会话、bridge、多 agent/swarm 场景下，权限是如何继续成立的？

本文基于以下主干代码阅读整理：

- `src/utils/permissions/*`
- `src/hooks/useCanUseTool.tsx`
- `src/hooks/toolPermission/*`
- `src/components/permissions/*`
- `src/utils/sandbox/sandbox-adapter.ts`
- `src/screens/REPL.tsx`
- `src/QueryEngine.ts`
- `src/remote/RemoteSessionManager.ts`
- `src/state/AppStateStore.ts`
- `src/Tool.ts`

---

## 2. 先给结论

这套权限与安全架构，本质上是一个“多层防线”设计：

```text
模型准备调用工具
  -> QueryEngine / useCanUseTool 进入权限判定
  -> 规则层：deny / ask / allow
  -> 工具层：每个工具自己的 checkPermissions()
  -> 路径层：工作目录、敏感文件、Windows/UNC/符号链接安全检查
  -> 模式层：default / plan / acceptEdits / auto / bypassPermissions / dontAsk
  -> 交互层：本地 UI、bridge、channel、远程审批、swarm leader 审批
  -> 执行层：真正执行工具
  -> 运行时隔离层：sandbox 限制文件和网络
```

如果只记一句话：

**Claude-code-open 不是“先问用户再执行”的单点权限模型，而是“规则判定 + 安全校验 + 模式机 + 审批编排 + OS 沙箱”共同组成的纵深防御体系。**

---

## 3. 这套架构到底在解决什么问题

从业务目标看，这个项目必须同时满足两件事：

1. Agent 要足够自动化，不能每做一步都打断用户。
2. Agent 又不能因为“自动化”而获得不受控制的本地执行权。

所以这里的安全问题不是单一问题，而是至少有 5 类：

1. 工具级风险  
   例如 `Bash`、`PowerShell`、`WebFetch`、文件写入、本地 agent 启动。

2. 路径级风险  
   例如编辑 `.git`、`.claude/settings.json`、shell 配置、UNC 路径、符号链接绕过。

3. 模式级风险  
   例如 `acceptEdits`、`bypassPermissions`、`auto` 这些模式切换后，权限语义会整体变化。

4. 分布式审批风险  
   例如远程 CCR、bridge、手机 channel、swarm worker 并不是都在本地终端里点确认。

5. 执行隔离风险  
   就算应用层“误放行”，也不能让高风险命令直接无限制读写文件或访问网络。

因此这套系统采用了分层方案：

- 应用层负责“这次操作在逻辑上是否应被允许”。
- 沙箱层负责“即使执行了，也只能在受限环境里执行”。

这就是它的核心设计思想。

---

## 4. 总体分层

可以把权限与安全架构拆成 6 层：

```text
第 1 层：权限配置与模式层
  PermissionMode / PermissionRule / PermissionUpdate / permissionSetup

第 2 层：统一权限决策层
  permissions.ts / hasPermissionsToUseTool()

第 3 层：资源安全校验层
  filesystem.ts / pathValidation.ts

第 4 层：交互审批编排层
  useCanUseTool.tsx / PermissionContext.ts / interactiveHandler.ts

第 5 层：权限展示层
  components/permissions/*

第 6 层：运行时隔离层
  sandbox-adapter.ts / REPL sandboxAskCallback
```

各层职责如下：

- 第 1 层定义“规则和模式是什么”。
- 第 2 层定义“一次工具调用如何综合这些信息得到 allow / ask / deny”。
- 第 3 层补充“文件路径是否安全”。
- 第 4 层负责 ask 之后如何真正拿到审批结果。
- 第 5 层负责把不同工具的审批内容渲染给用户。
- 第 6 层负责 OS 级别的文件和网络限制。

---

## 5. 最核心的入口：`hasPermissionsToUseTool`

核心文件：

- `src/utils/permissions/permissions.ts`
- `src/hooks/useCanUseTool.tsx`

### 5.1 两个关键入口的分工

这里有两个最重要的函数：

1. `hasPermissionsToUseTool()`  
   这是“纯决策引擎”，输入是工具、参数、上下文，输出是：
   - `allow`
   - `ask`
   - `deny`

2. `useCanUseTool()`  
   这是“交互编排器”，它拿到 `hasPermissionsToUseTool()` 的结果后：
   - 如果是 `allow`，直接通过
   - 如果是 `deny`，直接拒绝
   - 如果是 `ask`，就进入交互审批流程

所以：

- `permissions.ts` 决定“应不应该问”
- `useCanUseTool.tsx` 决定“怎么问、问谁、谁先答算数”

这是一个很重要的职责拆分。

### 5.2 `PermissionDecision` 是整个系统的统一语言

核心类型在 `src/types/permissions.ts`：

```ts
type PermissionDecision =
  | { behavior: 'allow', updatedInput?, decisionReason? }
  | { behavior: 'ask', message, updatedInput?, suggestions?, decisionReason? }
  | { behavior: 'deny', message, decisionReason }
```

它有几个设计亮点：

1. 不只是 allow / deny，还有 `ask`
   说明“审批”被设计成一等公民，而不是 deny 的附属分支。

2. 可以返回 `updatedInput`
   说明用户审批后，不一定只是点“允许”，还可能顺手修改参数。

3. 可以返回 `suggestions`
   说明审批界面不只是确认，还会建议“把这类操作加入 session/user/project 权限规则”。

4. 带 `decisionReason`
   说明系统不仅给结论，还保留“为什么这么决定”的因果链。

这使整个权限系统具备可解释性。

---

## 6. 核心数据结构

## 6.1 `ToolPermissionContext`

定义在 `src/Tool.ts`。

这是权限运行时上下文，几乎所有权限判断都围绕它展开。

关键字段：

- `mode`
- `additionalWorkingDirectories`
- `alwaysAllowRules`
- `alwaysDenyRules`
- `alwaysAskRules`
- `isBypassPermissionsModeAvailable`
- `isAutoModeAvailable`
- `strippedDangerousRules`
- `shouldAvoidPermissionPrompts`
- `awaitAutomatedChecksBeforeDialog`
- `prePlanMode`

可以这样理解：

- `mode` 决定全局策略。
- 三组 rules 决定细粒度策略。
- `additionalWorkingDirectories` 决定工作区边界。
- 其余字段负责适配 plan、auto、headless、bypass 等特殊场景。

这说明作者没有把权限状态散落在各处，而是集中在一个明确的上下文对象里。

## 6.2 `PermissionRule`

定义在 `src/types/permissions.ts`：

```ts
type PermissionRule = {
  source: PermissionRuleSource
  ruleBehavior: 'allow' | 'deny' | 'ask'
  ruleValue: {
    toolName: string
    ruleContent?: string
  }
}
```

它表达的是：

- 对哪个工具生效：`toolName`
- 是否限定内容：`ruleContent`
- 行为是什么：`allow/deny/ask`
- 规则来自哪里：`source`

例如：

- `Bash`  
  整个 Bash 工具规则

- `Bash(git status)`  
  针对具体命令

- `Read(/foo/**)`  
  针对文件路径模式

- `WebFetch(domain:example.com)`  
  针对域名

这个设计的关键点是：

**规则不是一个统一字符串到处手写匹配，而是先解析成结构化对象，再进入各自工具的匹配逻辑。**

## 6.3 `PermissionUpdate`

定义在 `src/types/permissions.ts`。

它不是“规则本身”，而是“对权限配置进行变更的操作”。

支持：

- `addRules`
- `replaceRules`
- `removeRules`
- `setMode`
- `addDirectories`
- `removeDirectories`

这意味着系统区分：

- 静态权限状态
- 导致状态变化的事件

这是一种典型的“命令式更新”设计。好处是：

- UI 可以把用户选择转成更新操作
- 更新既可以落盘，也可以只作用于 session
- 同一更新逻辑可以复用在本地 UI、SDK、bridge、remote、swarm 中

## 6.4 `PermissionMode`

核心模式包括：

- `default`
- `plan`
- `acceptEdits`
- `bypassPermissions`
- `dontAsk`
- `auto`

语义上可以粗略理解为：

- `default`：标准审批模式
- `plan`：偏规划，不鼓励直接执行
- `acceptEdits`：工作区内编辑自动放行
- `bypassPermissions`：极危险，基本绕过审批
- `dontAsk`：不弹框，直接拒绝 ask 类操作
- `auto`：不直接问用户，而是先让分类器判断

这里最值得注意的是：  
**mode 不是简单 UI 状态，而是权限判定流程中的一级输入。**

---

## 7. 权限规则系统是怎么工作的

## 7.1 规则来源是分层的

规则来源 `PermissionRuleSource` 包括：

- `userSettings`
- `projectSettings`
- `localSettings`
- `flagSettings`
- `policySettings`
- `cliArg`
- `command`
- `session`

这说明权限不是单一配置文件控制，而是多来源叠加。

你可以把它理解成：

```text
组织策略
  + 用户设置
  + 项目设置
  + 本地设置
  + CLI 参数
  + 命令期间临时规则
  + 当前会话临时规则
```

其中：

- `policySettings` 偏组织治理
- `project/local/user` 偏配置持久化
- `session/command/cliArg` 偏临时态

## 7.2 规则如何解析

核心文件：`src/utils/permissions/permissionRuleParser.ts`

规则字符串支持两类：

1. 只有工具名  
   例如 `Bash`

2. 工具名 + 内容  
   例如 `Bash(git status)`

解析函数：

- `permissionRuleValueFromString()`
- `permissionRuleValueToString()`

这里做了两个重要处理：

1. 处理转义括号  
   避免 `Bash(python -c "print(1)")` 这种内容被错误切分。

2. 兼容 legacy tool name  
   老规则名会映射到新工具名，保证历史配置不失效。

这属于典型的“配置兼容层”设计。

## 7.3 Shell 规则匹配算法

核心文件：`src/utils/permissions/shellRuleMatching.ts`

Shell 规则支持三种形式：

1. 精确匹配  
   例如 `git status`

2. 前缀匹配  
   例如 `git:*`

3. 通配符匹配  
   例如 `git *`

内部做法是：

- 先把规则解析成 `exact / prefix / wildcard`
- 再按规则类型去匹配命令
- wildcard 会被转成正则

其中有一个很值得学习的小细节：

- `git *` 在只有一个未转义 `*` 且位于末尾时，会把“后续参数部分”视为可选  
- 所以它既能匹配 `git add .`，也能匹配裸 `git`

这说明作者并不是只做语法匹配，而是在尽量让规则语义符合用户直觉。

---

## 8. 主业务流程：一次工具调用如何穿过权限系统

下面是最重要的一条链路。

```text
模型输出 tool_use
  -> QueryEngine 调用 canUseTool
  -> useCanUseTool()
  -> hasPermissionsToUseTool()
     -> deny 规则
     -> ask 规则
     -> tool.checkPermissions()
     -> 安全检查
     -> mode 判断
     -> allow 规则
     -> ask / deny / allow
  -> 若 ask
     -> useCanUseTool 进入审批编排
     -> 本地 UI / bridge / channel / swarm / remote
     -> 拿到结果后返回
  -> 工具执行
  -> 若是 Bash 且启用 sandbox
     -> 进入 OS 层文件/网络限制
```

下面拆开讲。

## 8.1 第一步：规则层先拦截

`hasPermissionsToUseToolInner()` 的前几个步骤是：

1. 整个工具是否被 deny
2. 整个工具是否被 ask
3. 调用工具自己的 `checkPermissions()`

这里体现出一个原则：

**先做全局规则判断，再做工具专属判断。**

这能保证：

- 统一配置有最高优先级
- 工具只负责自己最懂的细粒度逻辑

## 8.2 第二步：工具自身的权限逻辑

每个工具都可以实现 `checkPermissions()`。

例如文件类工具会进入 `filesystem.ts`，Shell 类工具会做命令细分，返回：

- `allow`
- `deny`
- `ask`
- `passthrough`

这里的 `passthrough` 很关键。

它表示：

- 我这个工具自己没有明确允许或拒绝
- 交回统一权限引擎继续判定

这就是“工具自治 + 总控编排”的结合点。

## 8.3 第三步：安全检查比模式更早

`permissions.ts` 和 `filesystem.ts` 有一个非常重要的设计：

**敏感安全检查发生在 bypass/acceptEdits/allow rule 之前。**

例如：

- `.claude/settings.json`
- `.git`
- `.vscode`
- `.bashrc`
- 可疑 Windows 路径
- UNC 路径

这些会返回 `decisionReason.type === 'safetyCheck'`。

这类结果在代码里被标成“bypass-immune”，也就是：

- 即使是 `bypassPermissions`
- 即使是 `acceptEdits`
- 即使 hooks 说 allow

也不能直接越过这道关。

这是整个安全架构里最重要的防穿透设计之一。

## 8.4 第四步：模式层决定默认行为

当规则和工具都没有最终决定时，才进入 mode 逻辑。

典型行为：

- `bypassPermissions`：直接 allow
- `acceptEdits`：工作区内写操作 allow
- `dontAsk`：ask 转 deny
- `auto`：ask 不直接给用户，而是走分类器

所以 mode 不是“附加选项”，而是权限系统里的策略切换器。

## 8.5 第五步：allow 规则兜底放行

如果前面都没挡住，再检查 `alwaysAllowRules`。

这一步在 `hasPermissionsToUseToolInner()` 中位于 bypass 之后、最终 ask 之前。

也就是说，规则优先级大致是：

```text
deny / ask / safety
  > bypass logic
  > allow
  > 默认 ask
```

这个顺序保证了：

- 危险规则不能被“宽松模式”悄悄覆盖
- allow 规则也不是无限大权

---

## 9. 文件权限与路径安全架构

核心文件：

- `src/utils/permissions/filesystem.ts`
- `src/utils/permissions/pathValidation.ts`

这是整个系统最复杂、也最容易被低估的部分。

## 9.1 为什么文件权限比工具权限复杂

例如“允许 Edit”并不等于“所有文件都能改”。

还要回答：

1. 路径是否在工作区内？
2. 是否是敏感路径？
3. 是否有符号链接绕过？
4. 是否是 Windows 可疑路径格式？
5. 是否是 UNC 网络路径？
6. 是否是内部系统目录，应该自动允许？

所以文件权限实际上是：

**工具权限 + 路径权限 + 安全策略**

## 9.2 工作目录是第一道天然边界

`ToolPermissionContext.additionalWorkingDirectories` 和 `getOriginalCwd()` 一起定义了可操作工作区。

`pathInAllowedWorkingPath()` 的逻辑是：

- 收集原始工作目录和额外目录
- 解析路径的不同真实形态
- 对符号链接、macOS `/private/tmp` 这类路径差异做归一化
- 确认目标路径是否处于允许目录内

这说明“工作目录”不是简单字符串前缀比较，而是考虑真实文件系统语义后的判断。

## 9.3 读写权限的主流程

### 写入流程：`checkWritePermissionForTool()`

主要步骤：

1. 先看 edit deny 规则
2. 允许内部可编辑路径
3. 对 `.claude/**` 的 session 级例外做特殊处理
4. 做综合安全检查 `checkPathSafetyForAutoEdit()`
5. 看 ask 规则
6. `acceptEdits + 工作区内` 自动允许
7. 看 allow 规则
8. 否则 ask

### 读取流程：`checkReadPermissionForTool()`

主要步骤：

1. UNC / Windows 可疑路径检查
2. read deny 规则
3. read ask 规则
4. 如果写权限允许，则读权限隐含允许
5. 工作区内允许读
6. 内部可读路径允许
7. allow 规则
8. 否则 ask

初学者要特别注意：

**读取和写入虽然共用很多能力，但策略并不完全相同。**

例如：

- 读操作更宽松
- “能写就能读”
- 但 read deny/read ask 仍然优先于 edit allow

这点在设计上非常严谨。

## 9.4 内部路径白名单

### 自动允许写入

`checkEditableInternalPath()` 允许：

- 当前 session 的 plan 文件
- scratchpad
- job 目录
- agent memory
- auto memory
- 项目 `.claude/launch.json`

### 自动允许读取

`checkReadableInternalPath()` 允许：

- session memory
- project dir 下的历史数据
- plan 文件
- tool result storage
- scratchpad
- project temp dir
- agent/auto memory
- tasks/teams 目录
- bundled skill references

这体现的是一个重要原则：

**系统自己生成、自己消费的内部资产，不应该反复打扰用户审批。**

这属于“可信内部通道”。

## 9.5 敏感路径安全检查

`checkPathSafetyForAutoEdit()` 是文件安全的关键算法。

它会检查：

1. 可疑 Windows 路径  
   包括 ADS、8.3 短文件名、长路径前缀、尾部点空格、DOS 设备名、连续 `...`

2. Claude 自己的配置文件  
   例如 `.claude/settings.json`

3. 敏感文件与目录  
   例如：
   - `.git`
   - `.vscode`
   - `.idea`
   - `.claude`
   - `.bashrc`
   - `.gitconfig`
   - `.mcp.json`

而且它检查的不是单一路径，而是：

- 原始路径
- 经过符号链接解析后的路径链

这就是为了防止：

- 表面上写的是普通文件
- 实际 symlink 到敏感文件

这是一个典型的反绕过设计。

## 9.6 路径模式匹配算法

`matchingRuleForInput()` 使用 `ignore` 库把 Read/Edit 规则当作路径模式来匹配。

设计特点：

1. 支持不同 root 语义  
   - `//path` 表示绝对路径
   - `/path` 相对 settings root
   - `~/path` 相对 home
   - 相对路径相对 cwd

2. 支持 `/**` 目录递归语义

3. 不同设置源会映射到不同 root

这意味着文件规则并不是简单 glob，而是带“来源上下文”的模式系统。

---

## 10. 自动模式 `auto` 的设计

核心文件：

- `src/utils/permissions/permissions.ts`
- `src/utils/permissions/permissionSetup.ts`

`auto` 是整个系统最有代表性的“安全与自动化平衡”设计。

## 10.1 `auto` 不是“全部自动允许”

很多初学者会误以为 `auto` = “不再问用户”。

实际不是。

`auto` 的语义是：

- 当结果本来是 `ask`
- 不立刻弹用户确认框
- 而是优先交给分类器 `classifyYoloAction()`
- 分类器决定 allow / block

所以它是“用机器审批替代人工审批”，不是“取消审批”。

## 10.2 为什么进入 auto 前要剥离危险规则

核心函数：`stripDangerousPermissionsForAutoMode()`

它会移除那些会绕过分类器的 allow 规则，例如：

- `Bash(*)`
- `Bash(python:*)`
- 危险的 `PowerShell(*)`
- `Agent(*)`

因为如果这些规则不移除，系统会在规则层直接 allow，分类器根本来不及判断。

这就是 auto 模式最关键的安全设计：

**先剥掉会绕过分类器的规则，再启用分类器。**

被剥掉的规则会暂存到 `strippedDangerousRules`，退出 auto 时再恢复。

## 10.3 auto 模式的快速路径

在 `permissions.ts` 中，进入分类器前还有两种“快路径”：

1. 如果某操作在 `acceptEdits` 下本来就会允许  
   则不必浪费一次分类器调用。

2. 某些 allowlisted tool 被认为足够安全  
   可直接跳过分类器。

这是为了控制时延和成本。

## 10.4 auto 模式的失败处理

如果分类器：

- 上下文太长
- 模型不可用
- 连续拒绝过多

系统不会无限死撑，而是：

- 回退到人工审批
- 或在 headless 场景直接中止 agent

同时还维护 `denialTracking`：

- `consecutiveDenials`
- `totalDenials`

超过阈值后，系统会强制停止“自动拒绝循环”，避免 agent 一直在危险动作上空转。

这是一种很成熟的 fail-safe 设计。

---

## 11. 模式切换状态机

核心函数：`transitionPermissionMode()`

这部分非常值得初学者学习，因为它不是简单改个字段，而是“切模式时同步处理副作用”。

模式切换会联动：

- plan 进入/退出
- auto 激活/关闭
- dangerous rules 剥离/恢复
- `prePlanMode` 存储与恢复

### 11.1 plan 模式

`prepareContextForPlanMode()` 会：

- 记录进入 plan 前的模式到 `prePlanMode`
- 在满足条件时让 plan 期间仍然使用 auto 语义

这样退出 plan 时能恢复到原模式，而不是粗暴回到 default。

### 11.2 bypass killswitch / auto gate

系统还支持动态关闭高风险模式：

- `checkAndDisableBypassPermissionsIfNeeded()`
- `checkAndDisableAutoModeIfNeeded()`

也就是说：

- 某些组织策略或远程 gate 变化后
- 当前会话里的 bypass/auto 也可能被强制失效

这说明权限模式不是单机静态配置，而是可被远端策略治理的。

---

## 12. ask 之后到底发生什么：交互审批编排

核心文件：

- `src/hooks/useCanUseTool.tsx`
- `src/hooks/toolPermission/PermissionContext.ts`
- `src/hooks/toolPermission/handlers/interactiveHandler.ts`
- `src/components/permissions/PermissionRequest.tsx`

## 12.1 `useCanUseTool()` 是 ask 流程总控

当 `hasPermissionsToUseTool()` 返回 `ask` 后，`useCanUseTool()` 会做三类事：

1. 先看是否有自动化审批路径  
   例如 coordinator hooks、classifier、swarm leader。

2. 如果都没有解决，进入交互审批  
   `handleInteractivePermission()`

3. 等审批结果出来，再 resolve 当前工具调用

这里最大的设计亮点是：

**审批本身是异步竞态系统。**

因为可能同时存在：

- 本地终端用户点击允许
- 远程 bridge 用户点击允许
- 手机 channel 回复 yes
- hook 自动批准
- classifier 自动批准

谁先成功，谁就赢。

## 12.2 `PermissionContext` 把审批过程对象化

`createPermissionContext()` 提供：

- 日志记录
- 权限更新持久化
- 构造 allow/deny 结果
- hook 执行
- classifier 尝试
- 取消与中止逻辑

这相当于把“审批过程中的公共动作”收敛成一个上下文对象，避免这些逻辑散落在 UI 组件里。

## 12.3 `createResolveOnce()` 是关键并发算法

审批竞态里最重要的算法不是分类器，而是 `createResolveOnce()`。

它提供：

- `claim()`
- `isResolved()`
- `resolve()`

核心意义：

- 多个异步审批源可能同时返回
- 系统必须保证最终只采用一个结果

否则会出现：

- 本地点了允许
- bridge 又回来一个拒绝
- 状态被覆盖

`claim()` 的原子“抢占”语义就是为了解决这个竞态问题。

这是这套交互权限系统里最关键的并发控制点。

## 12.4 UI 不是一个弹框，而是一组工具专属审批组件

`PermissionRequest.tsx` 会根据工具类型分发到不同组件：

- `BashPermissionRequest`
- `FileEditPermissionRequest`
- `FileWritePermissionRequest`
- `FilesystemPermissionRequest`
- `PowerShellPermissionRequest`
- `WebFetchPermissionRequest`
- `SkillPermissionRequest`
- `EnterPlanModePermissionRequest`
- `ExitPlanModePermissionRequest`

这说明审批 UI 不只是“允许/拒绝”，而是尽量针对不同风险给出专业上下文：

- Bash 看命令
- 文件编辑看 diff
- WebFetch 看域名
- plan 模式看计划内容

这本质上是在提高用户审批质量。

---

## 13. 远程、bridge、channel、swarm 如何参与审批

## 13.1 bridge 审批

`interactiveHandler.ts` 会把权限请求转发到 bridge：

```text
本地 ask
  -> bridgeCallbacks.sendRequest()
  -> 远端 UI 显示审批
  -> control_response 返回
  -> claim() 抢占成功则生效
```

`useReplBridge.tsx` 则负责建立这些 callback。

这意味着本地终端不是唯一审批者。

## 13.2 remote session 审批

`RemoteSessionManager.ts` 处理：

- `control_request`
- `control_response`
- `control_cancel_request`

也就是说，远程会话权限请求在协议层被显式建模，而不是混在普通消息流里。

这是远程权限系统能稳定工作的关键。

## 13.3 channel 审批

`interactiveHandler.ts` 还支持把 ask 转发给 Telegram/iMessage 等 channel。

本质是：

- 通过 channel MCP 把审批请求发送到外部消息渠道
- 用户在外部渠道回复
- 再回到统一的 resolve-once 竞态中

## 13.4 swarm worker 审批

当 agent 运行在 swarm worker 上时：

- worker 先尝试 classifier
- 不行就通过 mailbox 发给 leader
- leader 回答后 worker 继续

这使得子 agent 不需要各自拥有独立交互 UI，而是把高风险审批上收给 leader。

这是一种典型的“分布式权限集中审批”设计。

---

## 14. 沙箱架构：为什么有了权限系统还需要 sandbox

核心文件：`src/utils/sandbox/sandbox-adapter.ts`

这是另一个很关键的问题。

答案是：

**应用层权限判断解决“该不该做”，沙箱解决“即使做了也只能做多少”。**

## 14.1 沙箱与权限规则如何打通

`convertToSandboxRuntimeConfig()` 会把 Claude Code 的权限配置转换为 sandbox-runtime 配置：

- `Edit` allow/deny 规则 -> `allowWrite` / `denyWrite`
- `Read` deny 规则 -> `denyRead`
- `WebFetch(domain:...)` -> 网络 allowlist
- 额外工作目录 -> sandbox 写目录

这说明：

- 应用层规则和 OS 沙箱并不是两套完全割裂的系统
- Claude Code 会把高层意图下沉到运行时隔离层

## 14.2 沙箱中的默认保护

沙箱默认还会额外保护：

- settings 文件
- `.claude/skills`
- 裸 git repo 伪造路径
- 当前 cwd 与 originalCwd 差异场景
- Claude temp 目录和内部工作目录

这部分很多并不是普通用户手写规则，而是系统的内建防御。

## 14.3 网络审批

REPL 中的 `sandboxAskCallback` 负责网络访问审批。

当某 host 不在允许列表里：

- 本地 UI 会弹出 sandbox 网络审批
- 若是 swarm worker，会转发给 leader
- 若 bridge 已连接，也会同步给远程审批方

所以 sandbox 的网络权限和工具权限一样，也被接入了统一的审批基础设施。

---

## 15. 关键算法与设计亮点总结

## 15.1 规则优先级算法

核心思想：

```text
deny / ask / safety
  优先于
mode 宽松逻辑
  优先于
allow
  优先于
默认 ask
```

这保证了“更保守”的规则能压住“更宽松”的规则。

## 15.2 符号链接与多路径形态校验

很多路径判断不是只看输入字符串，而是看：

- 原始路径
- realpath
- 父目录真实路径
- macOS `/tmp` 与 `/private/tmp` 兼容路径

这是防止路径绕过的关键。

## 15.3 auto 模式危险规则剥离算法

进入 auto 前：

- 扫描 allow 规则
- 找出会绕过分类器的危险规则
- 从 context 中移除
- 存到 `strippedDangerousRules`
- 退出 auto 时恢复

这本质上是一个“模式切换时做安全归一化”的算法。

## 15.4 审批多方竞态的 resolve-once 算法

审批来源可能很多，但最终决策只能有一个。

通过 `claim()` 机制实现：

- 先抢到者生效
- 后到者无效

这是并发审批系统的核心。

## 15.5 deny tracking 与 fail-safe

自动模式下不是无限自动化，而是：

- 记录被拒绝次数
- 超限后回退人工审批或直接 abort

这防止 agent 在危险动作上陷入死循环。

---

## 16. 初学者最容易混淆的几个点

## 16.1 “规则”和“模式”不是一回事

- 规则是细粒度的：允许哪个工具、哪个命令、哪个路径。
- 模式是全局策略：默认怎么解释这些结果。

## 16.2 “allow rule” 不是最大权限

allow 规则仍然可能被：

- deny 规则
- ask 规则
- safetyCheck
- 管理策略

压住。

## 16.3 `auto` 不是“无权限”

`auto` 仍然有审批，只是审批者从“人”变成了“分类器”，而且还带有各种兜底回退机制。

## 16.4 “文件写入权限”不等于“路径安全”

即使工具是 FileEdit，也还要检查：

- 工作区范围
- 敏感目录
- UNC
- Windows 特殊路径
- 符号链接

## 16.5 沙箱不是 UI 功能

沙箱不是“弹个提示”的表现层能力，而是实际约束进程文件系统和网络能力的运行时隔离层。

---

## 17. 架构价值总结

这套权限与安全架构的价值，可以概括成 4 点：

1. 可扩展  
   新工具只要实现 `checkPermissions()`，就能接入统一权限体系。

2. 可解释  
   `decisionReason` 让每次 allow/ask/deny 都有因果来源。

3. 可治理  
   支持 policy settings、gate、killswitch、远程审批、session 临时规则。

4. 可防御  
   既有应用层权限，又有 OS 层 sandbox，还有路径安全与自动模式防绕过。

从工程设计角度看，这不是“给工具前面加个确认框”，而是一个相当完整的本地 Agent 安全控制框架。

---

## 18. 推荐阅读顺序

如果你是第一次读这套代码，建议按下面顺序继续：

1. `src/types/permissions.ts`  
   先把核心类型看懂。

2. `src/utils/permissions/permissions.ts`  
   看总决策流程。

3. `src/utils/permissions/filesystem.ts`  
   看文件权限和敏感路径逻辑。

4. `src/hooks/useCanUseTool.tsx`  
   看 ask 之后怎么编排审批。

5. `src/hooks/toolPermission/interactiveHandler.ts`  
   看多审批源竞态。

6. `src/utils/sandbox/sandbox-adapter.ts`  
   看规则怎样下沉到 OS 沙箱。

如果要再进一步理解：

- 看 `src/utils/permissions/permissionSetup.ts` 理解 mode 状态机
- 看 `src/remote/RemoteSessionManager.ts` 理解远程审批协议
- 看 `src/screens/REPL.tsx` 理解本地 UI 如何承接权限请求

---

## 19. 一句话总结

`vendor/Claude-code-open` 的权限与安全架构，本质上是一个围绕 Agent 工具调用构建的“多层次安全执行框架”：上层用规则、模式和审批决定是否允许，中层用路径与资源校验防止绕过，下层用 sandbox 限制真正的系统能力。
