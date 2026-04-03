# Claude-code-open 插件与 Skills 架构深度设计解读

## 1. 文档目标

本文不是面向初学者的概览，而是面向 AI 应用架构师、Agent 系统设计者、算法工程专家的深度技术解读。目标是解释：

- 为什么这个系统要把 `Skills` 与 `Plugins` 拆成两个层级；
- 它们如何通过统一抽象接入 Agent Runtime；
- 关键接口、类型与模块边界是什么；
- 运行时如何处理加载、缓存、热更新、权限、信任边界与多来源合并；
- 哪些地方体现了工业级工程设计，而不仅仅是功能堆叠。

参考输入：

- `./specs/understand/0001-project-arch.md`
- `./specs/understand/0005-plugins-skills-arch.md`

主要源码范围：

- `vendor/Claude-code-open/src/skills/*`
- `vendor/Claude-code-open/src/utils/plugins/*`
- `vendor/Claude-code-open/src/tools/SkillTool/*`
- `vendor/Claude-code-open/src/commands.ts`
- `vendor/Claude-code-open/src/types/command.ts`
- `vendor/Claude-code-open/src/Tool.ts`

---

## 2. 总体结论

这套架构的关键思想不是“支持插件”和“支持技能”，而是构造了一套**面向 Agent 的扩展能力中间层**。

它做了四个重要分离：

### 2.1 表达与分发分离

- `Skill` 负责声明“任务应该如何被模型理解和执行”；
- `Plugin` 负责声明“这些任务与相关运行时部件如何被打包、安装、配置、缓存、启停和治理”。

### 2.2 声明与运行分离

- 磁盘上是 `SKILL.md`、`plugin.json`、`hooks.json`、目录树；
- 运行时不直接使用这些文件，而是统一映射为 `Command`、`LoadedPlugin`、`AgentDefinition`、`PluginHookMatcher`、MCP/LSP 配置等中间对象。

### 2.3 快路径与完整路径分离

- 启动和常规查询走 cache-only 路径，尽量快；
- 安装、显式刷新、重新装配走 full-load 路径，保证一致性。

### 2.4 能力暴露与权限控制分离

- `Command` 和 `SkillTool` 负责让能力“可被调用”；
- 权限上下文、deny/allow 规则、hook、enterprise policy 和信任边界负责“能力是否真的可执行”。

这意味着 Claude-code-open 并不是在 CLI 上简单叠加一个插件系统，而是在做一个**可治理的 Agent Capability Plane**。

---

## 3. 架构定位：为什么必须同时存在 Skills 与 Plugins

## 3.1 Skill 是“能力语义单元”，Plugin 是“能力供应单元”

### Skill 的抽象职责

Skill 主要关心：

- 如何描述一个任务；
- 该任务适合什么时候使用；
- 执行时允许哪些工具；
- 是否可被模型主动调用；
- 是否需要 fork 到子 Agent；
- 是否对某些路径上下文敏感。

换句话说，Skill 主要服务于“任务语义层”。

### Plugin 的抽象职责

Plugin 主要关心：

- 如何把一组能力打成包；
- 包从哪里来；
- 如何安装、升级、缓存与清理；
- 是否启用；
- 是否依赖别的插件；
- 是否注入 hooks、MCP、LSP、output styles、agents；
- 是否需要用户配置项；
- 这些配置如何安全存储。

换句话说，Plugin 主要服务于“生态分发与治理层”。

## 3.2 这是一个经典的“双层扩展模型”

如果用更工程化的类比：

```text
Skill  ~= prompt-native procedure / task policy object
Plugin ~= package / extension bundle / lifecycle-managed delivery unit
```

如果没有 Skill：

- 插件作者必须直接面向更重的运行时对象；
- Prompt-level 能力表达无法低成本扩展；
- 模型与用户都缺少一个自然的“任务接口”。

如果没有 Plugin：

- Markdown 技能无法安装分发；
- 缺少版本治理、缓存、依赖和配置；
- 无法把 hooks/MCP/LSP/agents 等一起封装进可控扩展包。

因此两者并不是重复功能，而是两个不同层次的扩展抽象。

---

## 4. 核心抽象图

整个插件与 Skills 子系统可以用下面这张抽象图理解：

```text
磁盘声明层
  SKILL.md / commands/*.md / plugin.json / hooks.json / .mcp.json / .lsp.json

        ↓ 解析与验证

中间表示层
  Command
  LoadedPlugin
  AgentDefinition
  PluginHookMatcher
  McpServerConfig / LspServerConfig

        ↓ 聚合

能力目录层
  getCommands()
  getSkillToolCommands()
  getSlashCommandToolSkills()
  enabled / disabled plugins
  AppState.plugins / AppState.mcp / agentDefinitions

        ↓ 暴露

调用层
  用户 /skills /<command>
  模型 SkillTool 调用
  Hooks 生命周期
  MCP / LSP runtime

        ↓ 约束

治理层
  permissionContext
  enterprise marketplace policy
  dependency demotion
  cache-only vs full-load
  built-in / inline / marketplace source merging
```

这里最关键的是中间表示层。  
没有它，系统会变成“每类扩展文件都有一条专属执行链”，最终不可维护。

---

## 5. 统一中间表示：`Command` 是整个设计的核心

## 5.1 `Command` 并不只是 slash command

在 `src/types/command.ts` 中，`Command` 被定义为：

```ts
export type Command = CommandBase &
  (PromptCommand | LocalCommand | LocalJSXCommand)
```

这意味着在系统里，命令并不是单一行为，而是三类运行时实体的并集：

- `PromptCommand`
- `LocalCommand`
- `LocalJSXCommand`

而 Skill/Plugin Skill/MCP Skill 主要映射为：

- `PromptCommand`

这是一项很关键的建模选择。  
它没有单独定义一个“Skill Runtime Type”，而是把 Skill 直接纳入统一命令系统。

## 5.2 `PromptCommand` 的接口契约

关键字段：

- `type: 'prompt'`
- `progressMessage`
- `contentLength`
- `argNames?`
- `allowedTools?`
- `model?`
- `source`
- `pluginInfo?`
- `hooks?`
- `skillRoot?`
- `context?: 'inline' | 'fork'`
- `agent?`
- `effort?`
- `paths?`
- `getPromptForCommand(args, context): Promise<ContentBlockParam[]>`

从抽象角度看，这个接口同时编码了 5 层信息：

### A. 命名与展示层

- 名字
- 描述
- 参数提示
- `whenToUse`
- `version`
- `userFacingName`

### B. 调度层

- 是否允许用户直接调用
- 是否允许模型调用
- 是普通命令、技能、workflow 还是 MCP skill

### C. 权限层

- `allowedTools`
- `hooks`
- `skillRoot`

### D. 执行策略层

- `context: inline | fork`
- `agent`
- `model`
- `effort`

### E. 内容生成层

- `getPromptForCommand()`

这里最重要的是最后一项。  
项目把 Skill 设计成“调用时动态生成 prompt 的函数对象”，而不是“预加载文本块”。

这带来几个工程收益：

- 参数替换可以晚绑定；
- `${CLAUDE_SKILL_DIR}`、`${CLAUDE_PLUGIN_ROOT}`、`${CLAUDE_SESSION_ID}` 之类变量可以在运行时注入；
- shell 注入执行可以在权限约束下发生；
- fork 技能可以以统一协议进入子 Agent；
- Skill prompt 的真正内容直到需要时才生成，降低启动路径开销。

### 专家点评

这是非常典型的“命令对象 + 延迟求值”模式。  
在 Agent 系统里，这比“把所有技能文本一次性展开进系统 prompt”成熟得多。

---

## 6. `ToolUseContext`：命令与工具如何进入主执行引擎

`src/Tool.ts` 里的 `ToolUseContext` 是另一个关键接口。

它不是单纯的“工具调用上下文”，而是整个 Agent Runtime 的一个最小注入容器，主要包括：

- `options.commands`
- `options.tools`
- `options.agentDefinitions`
- `options.mcpClients`
- `getAppState()`
- `setAppState()`
- `messages`
- `toolPermissionContext`
- 中断、通知、UI、文件历史、归因状态等更新能力

从设计上，`ToolUseContext` 相当于：

```text
命令/工具/技能执行时可见的 runtime envelope
```

它的价值在于：

- Skill 不需要耦合 QueryEngine 内部；
- 命令对象和工具实现都能访问一致的状态面；
- 子 Agent 可以复制或派生它，而不必共享所有可变状态。

这也是 `prepareForkedCommandContext()`、`createGetAppStateWithAllowedTools()` 等辅助逻辑成立的前提。

---

## 7. Skills 的运行时设计

## 7.1 Skills 的本质是“可推理任务单元”

`loadSkillsDir.ts` 中的 frontmatter 解析表明，Skill 并不只是 prompt 内容，而是一种带执行策略的声明式任务单元。

它支持的关键信息包括：

- 描述与适用场景
- 参数签名
- 允许工具列表
- 模型与 effort 覆盖
- 是否用户可调
- 是否模型可调
- 执行上下文
- 绑定 agent
- hooks
- shell 注入策略
- 条件路径匹配

这说明 Skill 是在做“prompt-native orchestration spec”，而不是“Markdown 片段”。

## 7.2 Skill 到 `Command` 的编译过程

编译链条主要由两步组成：

### 第一步：`parseSkillFrontmatterFields()`

负责提取共享字段：

- `displayName`
- `description`
- `allowedTools`
- `argumentHint`
- `argumentNames`
- `whenToUse`
- `version`
- `model`
- `disableModelInvocation`
- `userInvocable`
- `hooks`
- `executionContext`
- `agent`
- `effort`
- `shell`

### 第二步：`createSkillCommand()`

负责把以上字段与：

- `skillName`
- `markdownContent`
- `source`
- `baseDir`
- `loadedFrom`
- `paths`

组装成 `Command`。

这等价于一条“声明文件 -> 运行时命令对象”的编译流水线。

## 7.3 `getPromptForCommand()` 是 Skill 执行核心

Skill 真正被调用时，其 prompt 不是直接返回原文，而是做以下处理：

1. 基于 `baseDir` 添加 skill 根目录说明；
2. 用 `substituteArguments()` 做参数替换；
3. 替换 `${CLAUDE_SKILL_DIR}`；
4. 替换 `${CLAUDE_SESSION_ID}`；
5. 对非 MCP skill 执行 `executeShellCommandsInPrompt()`；
6. 在 shell 注入执行时临时扩展 `alwaysAllowRules.command`；
7. 返回 `ContentBlockParam[]`。

这意味着 Skill 的运行行为是“程序化 prompt 展开”，而非“文本复制”。

### 设计含义

- Skill 可在运行时绑定当前会话；
- Skill 内容可以依赖本地脚本、目录结构和参数；
- Skill 可以声明权限，再由运行时按需提升到上下文中；
- 远端来源的 skill 可以禁用危险能力。

## 7.4 条件技能与动态技能发现

这部分是本架构里很有特色的“上下文感知技能注入”机制。

### 条件技能

带 `paths` frontmatter 的技能在启动阶段不会立即进入无条件命令列表，而是：

- 放入 `conditionalSkills`

后续当文件路径匹配时，通过：

- `activateConditionalSkillsForPaths(filePaths, cwd)`

迁移到：

- `dynamicSkills`

匹配算法使用 `ignore()`，即 gitignore 语义，而不是自制 glob。

### 动态技能目录发现

`discoverSkillDirsForPaths()` 会沿用户操作文件的目录向上查找 `.claude/skills`，但不会重复扫描已发现路径，并且会跳过 gitignored 目录。

之后通过：

- `addSkillDirectories(dirs)`

把新目录中的技能加载到 `dynamicSkills`。

### 这一机制的设计价值

从 Agent 设计角度看，这相当于：

- 根据当前“工作焦点文件”延迟注入局部技能；
- 让技能列表与当前任务局部上下文强耦合；
- 避免系统 prompt / SkillTool listing 被无关技能淹没。

这是一种比“全局技能库平铺”明显更高级的上下文组织方式。

---

## 8. SkillTool：模型如何进入技能系统

## 8.1 SkillTool 的职责不是“列出技能”，而是“受控调用 PromptCommand”

`src/tools/SkillTool/SkillTool.ts` 体现出 SkillTool 的定位：

- 它不是单纯的 discover 工具；
- 它是模型调用 `PromptCommand` 的正式协议入口。

它做的事情包括：

- 验证 skill 名称是否合法；
- 在本地命令与 MCP skill 间查找目标；
- 做权限判定；
- 决定 inline 还是 fork；
- 在 inline 路径里把 slash command 展开成新消息；
- 在 fork 路径里运行子 Agent；
- 对模型返回统一 `tool_result`。

## 8.2 SkillTool 实际调用的是“统一命令目录”

SkillTool 并不直接读目录，而是通过：

- `getAllCommands(context)`

拿到：

- 本地 `getCommands(getProjectRoot())`
- 当前 AppState 中的 MCP skill commands

再进行 `uniqBy([...], 'name')`。

这点非常重要。  
它说明模型看到的技能系统并不是文件系统视图，而是**聚合后的命令目录视图**。

这带来两个好处：

- 本地 skill、plugin skill、MCP skill 被统一对待；
- 模型调用面与用户 slash command 面高度一致。

## 8.3 SkillTool 的输入/输出协议

输入：

```ts
{
  skill: string
  args?: string
}
```

输出是 union：

### inline skill 输出

```ts
{
  success: boolean
  commandName: string
  allowedTools?: string[]
  model?: string
  status?: 'inline'
}
```

### forked skill 输出

```ts
{
  success: boolean
  commandName: string
  status: 'forked'
  agentId: string
  result: string
}
```

这说明 SkillTool 的协议已经把“技能只是 prompt 展开”和“技能是子 Agent 任务”两种执行模式统一在一个 tool 接口里。

从 Agent 平台设计看，这是非常强的抽象收敛。

## 8.4 SkillTool 的权限判定模型

`checkPermissions()` 有几层逻辑：

1. 查询 deny 规则；
2. 处理远程 canonical skill 的特殊 allow；
3. 查询 allow 规则；
4. 如果 skill 只有“安全属性”，自动允许；
5. 否则 ask 用户，并给出添加规则建议。

### 安全属性 allowlist

`SAFE_SKILL_PROPERTIES` 定义了一个白名单：

- 只有这些字段存在非空值时，skill 才被视为“无需显式授权”；
- 任何新增字段默认都不是安全属性；
- 因此新增能力默认趋向保守。

这是一个非常成熟的“安全默认收缩”策略。

### 工程意义

很多系统会做成黑名单：

- 有危险字段才拦；

而这里做成白名单：

- 只有明确评审过的字段才放行。

这对扩展系统非常关键，因为字段集会持续演化。

## 8.5 inline 与 fork 两种执行语义

### inline

- 通过 `processPromptSlashCommand()` 把命令展开为新的消息；
- 通过 `contextModifier()` 改写后续上下文；
- 注入额外 allowed tools；
- 处理模型覆盖与 effort 覆盖；
- 让技能内容进入当前对话主循环。

### fork

- `prepareForkedCommandContext()` 生成 skill 内容；
- 选择 `command.agent` 指定的 agent，或回退到 `general-purpose`；
- 通过 `runAgent()` 运行子 Agent；
- 抽取结果文本；
- 返回 `status: 'forked'`。

这两条路径本质上对应两种 Agent 编排策略：

- inline = 在当前上下文继续推理；
- fork = 开一个 side-agent 在隔离上下文执行。

这使 skill 同时具备：

- prompt 模板能力；
- orchestration 节点能力。

---

## 9. 插件系统的三层模型

`refresh.ts` 的注释已经隐含给出非常清晰的模型，可以形式化为：

## 9.1 Layer 1: Intent Layer

来源：

- settings `enabledPlugins`
- `extraKnownMarketplaces`
- `--plugin-dir`
- managed settings

这是用户或策略声明“想要什么”的层。

## 9.2 Layer 2: Materialization Layer

来源：

- `known_marketplaces.json`
- `~/.claude/plugins/marketplaces/*`
- `~/.claude/plugins/cache/*`
- `installed_plugins.json`

这是把 marketplace 与 plugin 真正物化到本地的层。

## 9.3 Layer 3: Active Runtime Layer

来源：

- `loadAllPlugins*()`
- `getPluginCommands()`
- `getAgentDefinitionsWithOverrides()`
- `loadPluginHooks()`
- `loadPluginMcpServers()`
- `loadPluginLspServers()`
- `AppState.plugins`

这是当前会话实际生效的能力层。

### 专家点评

这三层分离是整个系统最重要的工业化特征之一。  
很多插件系统混淆：

- “配置里声明了”
- “本地已经下载了”
- “当前进程已经生效了”

Claude-code-open 明确把三者拆开，因此：

- 启动可以快；
- 后台安装可以异步；
- 用户可控何时刷新 active runtime；
- 状态不一致时更容易诊断。

---

## 10. 插件主数据结构与接口

## 10.1 `LoadedPlugin`

定义在 `src/types/plugin.ts`。

它不是纯 manifest，而是“插件被系统装配后的运行时描述”。

关键字段按语义分为四组：

### A. 身份组

- `name`
- `source`
- `repository`
- `path`
- `sha`
- `isBuiltin`
- `enabled`

### B. 声明组

- `manifest`

### C. 组件入口组

- `commandsPath`
- `commandsPaths`
- `agentsPath`
- `agentsPaths`
- `skillsPath`
- `skillsPaths`
- `outputStylesPath`
- `outputStylesPaths`

### D. 运行时注入组

- `hooksConfig`
- `mcpServers`
- `lspServers`
- `settings`

### 设计意义

这类对象不是为“文件操作”设计，而是为“运行时装配”设计。  
系统后续各加载器只要拿到 `LoadedPlugin`，就能按自己的部件维度继续消费，不需要重复做 source/path/manifest 的解析。

## 10.2 `PluginManifestSchema`

`src/utils/plugins/schemas.ts` 中把插件声明分成多个子 schema，再合并成 `PluginManifestSchema`。

这是一种模块化 schema 设计：

- Metadata
- Hooks
- Commands
- Agents
- Skills
- OutputStyles
- Channels
- MCP Servers
- LSP Servers
- Settings
- UserConfig

### 工程亮点

顶层 schema 默认 strip unknown top-level fields，而非严格报错。  
但嵌套对象如 `userConfig`、`channels`、`lspServers` 则保持 strict。

这体现出非常成熟的 schema 策略：

- 顶层允许前向兼容和 vendor extension；
- 内层保持强校验，防止作者拼错真正重要的配置。

这比“一刀切 strict”或“一刀切 permissive”都更合理。

## 10.3 `MarketplaceSourceSchema` 与 `PluginSourceSchema`

系统显式区分：

- marketplace 的来源；
- marketplace 中某个 plugin entry 的来源。

前者支持：

- `url`
- `github`
- `git`
- `npm`
- `file`
- `directory`
- `hostPattern`
- `pathPattern`
- `settings`

后者支持：

- 相对路径
- `npm`
- `pip`
- `url`
- `github`
- `git-subdir`

### 为什么要拆开两套 schema

因为 marketplace 是“插件目录”的来源，而 plugin source 是“单个插件”的来源。  
两者生命周期、路径解析上下文与安全边界不同。

这是一个边界非常清晰的设计。

## 10.4 `PluginError`

采用判别联合，不依赖字符串匹配。

这不仅方便 UI 和日志，也意味着：

- 错误有结构化语义；
- 下游系统能按 `type` 做稳定分支；
- 未来新增错误类型时可渐进增强，不需要大规模 grep 修补字符串逻辑。

对于大型扩展生态，这是非常必要的工程化基础。

---

## 11. Plugin Loader：发现、合并、验证、缓存

## 11.1 `loadAllPlugins()` 与 `loadAllPluginsCacheOnly()`

这两个函数不是简单“有没有缓存”的区别，而是两个**不同的读取意图**。

### `loadAllPluginsCacheOnly()`

目标：

- 尽可能快地得到当前缓存中的插件运行时描述；
- 启动与常规请求优先使用；
- 避免不必要的 materialization 工作。

### `loadAllPlugins()`

目标：

- 执行完整刷新语义；
- 安装、刷新、显式同步时使用；
- 能 warm cache-only consumers。

### 为什么不是单函数加参数

从工程管理角度，拆成两个独立入口能避免：

- cache-only 结果意外满足 full-load 调用；
- 刷新逻辑与查询逻辑相互污染；
- memoize 缓存语义变得不清晰。

这是一个典型的“按调用意图分离 API”设计。

## 11.2 `assemblePluginLoadResult()`

这是插件加载核心汇聚点。

流程：

1. 取 `inlinePlugins`；
2. 并行加载 marketplace plugins 与 session-only plugins；
3. 加入 built-in plugins；
4. 合并三种来源；
5. 依赖校验与 demote；
6. 计算 enabled/disabled/errors；
7. 缓存 plugin settings。

这个函数非常像一个 runtime assembly pipeline。

## 11.3 插件多来源合并

插件来源：

- marketplace
- session only
- builtin

设计上，session plugin 允许覆盖已安装插件，但 managed settings 锁定的插件有更强优先级。

这意味着系统做的不只是并集，而是带策略的合并。

本质上是一套：

```text
source precedence + managed override + dependency demotion
```

逻辑。

## 11.4 依赖模型：不是模块图，而是“能力存在性保证”

`dependencyResolver.ts` 注释写得非常清楚：

> dependency semantics are apt-style presence guarantees, not a module graph

这意味着：

- 依赖不是编译链接关系；
- 依赖是“某些 namespaced capabilities 必须存在”。

例如：

- A 插件依赖 B 插件，不表示 A import B；
- 而表示 A 运行时需要 B 提供的 MCP / commands / agents 等能力可见。

### 这为什么重要

在 Agent 平台里，插件耦合更像“能力互补”，而不是“代码链接”。  
这种依赖语义比传统 package dependency 更贴近运行时现实。

## 11.5 `verifyAndDemote()`：负反馈式一致性修复

插件加载后，不满足依赖的 enabled plugin 不会直接使整个系统失败，而是：

- 在 session 内被 demote 为 disabled；
- 记录结构化错误；
- 不写回用户 settings。

并且它通过 fixed-point 迭代处理级联失效：

- A 失效；
- B 依赖 A，因此 B 也失效；
- 直到稳定。

### 专家点评

这是非常成熟的容错策略。  
它优先保证系统可运行，而不是让一次依赖坏掉拖垮全部插件加载。

---

## 12. Marketplace 管理：Intent 与 Materialization 的桥梁

`marketplaceManager.ts` 与 `reconciler.ts` 共同构成 Layer 1 -> Layer 2 的桥梁。

## 12.1 `getDeclaredMarketplaces()`

返回“应该存在什么 marketplace”的声明集合。

其输入包括：

- settings 中的 `extraKnownMarketplaces`
- `--add-dir` 注入的 marketplace
- 官方 marketplace 的隐式声明

特别之处在于：

- 官方 marketplace 可作为 `sourceIsFallback: true` 的隐式声明；
- 如果已经 materialized，不会被无意义 stomp。

这说明系统对“默认官方市场”与“用户显式指定来源”做了策略区分。

## 12.2 `diffMarketplaces()`

比较 declared intent 与 materialized state。

结果分三类：

- `missing`
- `sourceChanged`
- `upToDate`

它还会对本地相对路径进行规范化，以解决 worktree / 项目根解析差异。

### 这里的工程点

diff 不是简单文本比较，而是：

- 带路径规范化；
- 带 fallback source 语义；
- 适配 git worktree；

是一个“声明与物化状态一致性检查器”。

## 12.3 `reconcileMarketplaces()`

负责：

- 对 `missing/sourceChanged` 执行安装或更新；
- 逐个进度回调；
- 失败隔离，不拖垮整体；
- 只做 additive / update，不盲删。

这种策略非常适合扩展生态，因为：

- 删除操作风险更高；
- 保守的加法式 reconcile 更容易稳定运行。

---

## 13. 用户配置与密钥管理：`userConfig` 的工业化实现

这是插件系统里很值得学习的一部分。

## 13.1 `manifest.userConfig` 不是 UI 装饰，而是扩展协议的一部分

在 schema 中，`userConfig` 允许插件声明：

- 类型
- 标题
- 描述
- 默认值
- 是否必填
- 是否 multiple
- 是否 sensitive
- min/max

这意味着插件配置不是“自由 JSON”，而是**声明式配置协议**。

## 13.2 存储分层

`pluginOptionsStorage.ts` 里：

- 非敏感值 -> `settings.json pluginConfigs[pluginId].options`
- 敏感值 -> `secureStorage.pluginSecrets`

读取时：

- 二者 merge；
- secureStorage 优先。

### 这一设计的价值

- 非敏感配置容易 diff、备份、共享；
- 敏感配置不会落明文；
- 调用方只看到统一的 `loadPluginOptions(pluginId)` 结果。

这是典型的“逻辑统一 + 物理分存”设计。

## 13.3 配置如何注入技能/agent/hook/MCP/LSP

系统支持：

- `${user_config.KEY}` 替换

但有严格边界：

- skill/agent content 中只注入非敏感配置；
- 敏感配置主要用于更受控的 runtime config，如 hooks/MCP/LSP env。

这比“一股脑替进 prompt”更安全，也更合理。

## 13.4 为什么这是工业级设计

因为它同时解决了：

- 可配置性
- 类型约束
- 安全存储
- 向不同子系统统一注入
- 缓存与失效

而且实现上保留了 memoize，避免每次 hook 都同步打 keychain。

---

## 14. Plugin Commands / Skills / Agents / Hooks 的部件化加载

## 14.1 `loadPluginCommands.ts`

这个文件本质上是“把插件里的 Markdown 组件编译成 `Command`”。

其加载对象包括两类：

- plugin commands
- plugin skills

二者共享绝大部分实现，但保留语义差异：

- 命名空间命名
- `loadedFrom`
- `progressMessage`
- `CLAUDE_SKILL_DIR` 与 base dir 注入

### 关键机制

- 目录递归扫描由 `walkPluginMarkdown()` 负责；
- `SKILL.md` 的目录优先规则与普通 markdown 文件区分；
- `commandsMetadata` 支持 object-mapping 和 inline content；
- `${CLAUDE_PLUGIN_ROOT}`、`${user_config.X}` 支持运行时替换。

### 专家视角

这套实现让 plugin 既可以：

- 直接交付 markdown 文件；
- 又可以通过 manifest 精准描述命令元数据；
- 还可以 inline content。

也就是说，插件声明面既支持“文件优先”，也支持“manifest 优先”。

## 14.2 `loadPluginAgents.ts`

它把插件 agent markdown 变成 `AgentDefinition`。

支持字段包括：

- tools
- skills
- color
- model
- background
- memory
- isolation
- effort
- maxTurns
- disallowedTools

并且会在 memory 打开时自动补 Read/Edit/Write。

### 关键安全边界

对 plugin agent，以下字段即使在 frontmatter 中出现也会被忽略：

- `permissionMode`
- `hooks`
- `mcpServers`

原因是这些字段会跨越插件安装时的信任边界。

这是一项非常重要的设计约束：

- agent 文件是插件内部局部声明；
- 但 hooks/MCP/权限是平台级强能力；
- 不允许 agent 文件在细粒度上偷偷增加高权限能力。

## 14.3 `loadPluginHooks.ts`

它把插件 hooks 转换为：

- 按 `HookEvent` 分类的 `PluginHookMatcher[]`

再注册到全局 hook 系统。

关键设计点：

- hook matcher 附加 `pluginRoot/pluginName/pluginId`；
- clear 与 register 成对原子切换；
- `pruneRemovedPluginHooks()` 只移除已失效插件 hooks，不提前加载新 hooks。

### 专家点评

这体现了两种不同的刷新语义：

- prune-only：保证移除立即生效；
- full swap：通过 `/reload-plugins` 显式完成。

这种精细控制非常适合长运行会话。

---

## 15. `commands.ts`：用户命令面与模型技能面的统一聚合器

`commands.ts` 是整个能力平面的目录服务。

## 15.1 `loadAllCommands(cwd)`

它并行拉取：

- Skills 目录命令
- plugin skills
- bundled skills
- builtin plugin skills
- plugin commands
- workflows
- 内置命令

形成单一命令集合。

## 15.2 `getCommands(cwd)`

它在聚合结果上再做：

- availability 过滤
- `isEnabled()` 过滤
- 动态 skills 插入

这里的关键点是动态 skills 不是简单 append，而是插到 plugin skills 与 built-in commands 之间，说明命令表是有语义排序的。

## 15.3 SkillTool 与 slash command 使用不同子视图

### `getSkillToolCommands(cwd)`

用于模型调用。  
筛选条件偏向：

- prompt command
- 非 builtin
- 不禁用模型调用
- 必须有足够好的描述或 whenToUse

### `getSlashCommandToolSkills(cwd)`

偏向技能列表展示。  
筛选条件不同，更强调 skills 的语义来源。

### 设计含义

同一底层命令集合，在不同消费方前展示不同切片。  
这是一种典型的“统一目录，按使用者投影”的设计。

---

## 16. 缓存体系：不是一个缓存，而是一组分层缓存

插件与 Skills 子系统并不是只有一个 memoize，而是一整套 cache mesh。

## 16.1 主要缓存层

### A. 数据加载缓存

- `loadAllPlugins()`
- `loadAllPluginsCacheOnly()`
- `getSkillDirCommands()`
- `getPluginCommands()`
- `getPluginSkills()`
- `loadPluginHooks()`

### B. 派生目录缓存

- `loadAllCommands()`
- `getSkillToolCommands()`
- `getSlashCommandToolSkills()`

### C. 选项/密钥缓存

- `loadPluginOptions()`

### D. 文本提示缓存

- `SkillTool/prompt.ts` 里的 prompt cache

### E. 本地 materialization 缓存

- marketplaces cache
- versioned plugin cache
- installed_plugins.json

## 16.2 `clearAllCaches()` 与 `refreshActivePlugins()`

`clearAllCaches()` 会清：

- plugin loader cache
- plugin command/agent/hook/outputstyle cache
- commands cache
- agent definitions cache
- skill prompt cache

但这还不是 runtime reload。  
真正把新状态装回 AppState 的，是：

- `refreshActivePlugins()`

### 设计含义

缓存失效与状态重建是两个不同动作。  
这是成熟系统必须保持的区分。

## 16.3 版本化插件缓存与 orphan cleanup

`cacheUtils.ts` 与 `installedPluginsManager.ts` 表明，插件缓存是 versioned 的，并支持：

- orphan 标记
- 延迟清理
- 重装时移除 stale marker

这意味着系统不是“覆盖式缓存”，而是“版本化可回收缓存”。

从工业化角度看，这对：

- 回滚
- 升级
- 并发会话稳定性
- 安全清理

都更友好。

---

## 17. 子 Agent 执行：fork skill 的工程深意

`prepareForkedCommandContext()` 与 `forkedAgent.ts` 显示出 fork skill 的严谨设计。

## 17.1 fork skill 的输入准备

它会：

1. 调用 `command.getPromptForCommand()`；
2. 把结果内容提取为 skill text；
3. 解析 `allowedTools`；
4. 基于 `createGetAppStateWithAllowedTools()` 构造修改后的 `getAppState`；
5. 从 active agents 中选择目标 agent；
6. 构造 `promptMessages`。

## 17.2 cache-safe params 思维

`forkedAgent.ts` 的注释非常有价值：

- forked agent 希望尽量复用父请求的 prompt cache；
- 因此强调 `system prompt`、`tools`、`model`、`messages prefix`、`thinking config` 一致性；
- 甚至提醒 `maxOutputTokens` 会破坏缓存键。

### 专家点评

这已经不是普通 CLI 工程，而是把“LLM API cache key 的稳定性”纳入编排层设计。  
这体现出对推理成本和系统性能的深度理解。

---

## 18. 信任边界与安全设计

这套架构最强的工业特征之一，就是信任边界非常明确。

## 18.1 来源分级

系统显式区分：

- builtin
- bundled
- plugin
- mcp
- commands_DEPRECATED
- skills
- managed

这不仅用于展示，也用于：

- telemetry 脱敏
- 是否允许 shell 注入
- 权限策略
- 官方 marketplace 标识与红线处理

## 18.2 MCP skill 默认不可信

`createSkillCommand()` 中对 `loadedFrom === 'mcp'` 的技能不执行 inline shell command 注入。

这是一个非常重要的设计决定：

- MCP 是远端接入点；
- 就算 skill 是 markdown，也不应获得本地 shell prompt 拼接能力。

## 18.3 enterprise marketplace policy

`pluginLoader.ts` 在加载 marketplace plugin 时会检查：

- strict allowlist
- blocklist
- source 是否可验证

特别重要的是 fail-closed 逻辑：

- 若 policy 生效且无法解析 marketplace source；
- 则阻止加载，而不是继续。

这防止“配置损坏时静默 fail-open”。

## 18.4 官方 marketplace 伪装防护

`schemas.ts` 中有：

- 官方 marketplace 保留名列表；
- blocked name pattern；
- 非 ASCII 拦截；
- source 校验必须来自官方 GitHub org。

这已经把插件生态中的“品牌仿冒攻击”纳入 schema 层防御。

## 18.5 插件 agent 的能力边界

前文提到，plugin agent 不能声明 `permissionMode/hooks/mcpServers`。

这是另一层防线：  
即使用户装了插件，也不代表插件内部任意一个 agent markdown 就能继续扩权。

---

## 19. 工业级工程设计与关键创新点

下面列出我认为最值得学习的设计。

## 19.1 用 `Command` 做 prompt-native capability IR

这是全局最关键的抽象创新。

效果：

- Skills、plugin commands、plugin skills、MCP skills 收敛为统一中间表示；
- 用户 slash command 与模型 SkillTool 使用同一能力目录；
- 不同来源共享相同的延迟 prompt 生成接口。

这比单独维护 N 套 skill/command/procedure 类型成熟得多。

## 19.2 目录作用域 + 条件激活的技能系统

通过：

- `paths`
- `conditionalSkills`
- `dynamicSkills`
- `discoverSkillDirsForPaths()`

系统把技能从“全局静态表”升级为“随工作上下文激活的局部能力”。

这对 AI coding agent 尤其关键，因为不同子目录的工作方式常常不同。

## 19.3 cache-only / full-load 双通道

这是扩展生态和交互性能之间的核心平衡器。

在 CLI / Agent 系统里，启动延迟直接影响用户体验。  
这套设计避免了“为了插件完整性牺牲所有启动时间”的常见问题。

## 19.4 安全默认的白名单权限模型

SkillTool 的 `SAFE_SKILL_PROPERTIES` 是一个非常漂亮的设计：

- 新属性默认不安全；
- 必须显式评审后才能放进白名单。

这是扩展协议演化时极少见但非常正确的策略。

## 19.5 多层状态分离

把：

- intent
- materialization
- active runtime

三层拆开，是所有长生命周期插件系统应该学习的模式。

## 19.6 结构化错误 + fail-closed policy

插件生态里最怕的就是：

- 错误靠日志字符串；
- policy 失败时静默放行。

这里两者都避免了。

## 19.7 面向 LLM 成本与缓存键的编排意识

fork agent 那套 cache-safe params 设计说明：

- 作者不仅懂软件结构；
- 也在用“LLM 缓存键、thinking config、上下文窗口”来反向约束架构。

这是真正 AI-native 工程设计的表现。

---

## 20. 设计约束与潜在复杂度

再成熟的系统也不是没有代价。

## 20.1 `Command` 的语义负载很重

因为太多能力都收敛到 `Command`，它的含义已经横跨：

- 用户命令
- 模型技能
- plugin skill
- workflow
- MCP prompt skill

这提高了复用，也提高了理解门槛。

## 20.2 缓存很多，必须有严格失效纪律

这里的 memoize 层非常多。  
如果没有像 `clearAllCaches()`、`refreshActivePlugins()` 这样的强约束，很容易出现：

- loader 更新了；
- command directory 还没更新；
- prompt cache 还在旧状态；
- hooks 已被 prune 但没 full swap。

当前代码已经尽量显式管理这个问题，但复杂度是客观存在的。

## 20.3 运行时可组合能力很多，测试面会变宽

组合维度包括：

- source 类型
- managed vs user vs project vs inline
- skill vs command vs agent vs hook
- MCP/LSP 注入
- dynamic skill discovery
- permission rule
- remote mode / bare mode / sync install mode

这意味着单元测试之外，必须依赖强集成测试和灰度保护。

---

## 21. 建议的专家级阅读路径

如果你想从“读懂”走到“能重构或迁移这套架构”，建议按以下顺序。

1. `src/types/command.ts`
   先搞清统一中间表示是什么。
2. `src/Tool.ts`
   再搞清运行时上下文面是什么。
3. `src/skills/loadSkillsDir.ts`
   看 Skill 如何被编译成 Command。
4. `src/tools/SkillTool/SkillTool.ts`
   看模型如何调用 Command。
5. `src/utils/plugins/schemas.ts`
   看 plugin manifest 和 marketplace/source 协议。
6. `src/utils/plugins/pluginLoader.ts`
   看插件如何被发现、合并、demote 和缓存。
7. `src/utils/plugins/loadPluginCommands.ts`
   看 plugin command / plugin skill 如何复用 Command 抽象。
8. `src/utils/plugins/loadPluginAgents.ts`
   看 agent 边界与安全约束。
9. `src/utils/plugins/loadPluginHooks.ts`
   看 lifecycle 注入与热替换。
10. `src/utils/plugins/refresh.ts`
   看 active runtime 如何整体重建。

---

## 22. 最终总结

如果把这套架构压缩成一句话：

**Claude-code-open 把“扩展”做成了 Agent Runtime 的一等公民，并且通过 `Command` 这个中间表示，把 Prompt、Plugin、Tool、Agent、Hook、MCP、LSP 和权限体系连接成一个统一能力平面。**

它的高级之处不在于“支持很多扩展类型”，而在于：

- 抽象收敛得足够好；
- 加载与刷新模型足够清晰；
- 对来源、权限、缓存、配置、信任边界都有显式建模；
- 关键路径里处处体现出对 LLM 运行时特性的理解。

从 AI 应用架构角度看，这不是一个普通的 CLI 插件系统，而是一套：

```text
Prompt-native capability IR
+ lifecycle-managed extension delivery
+ runtime-governed agent orchestration
```

的综合设计。

这正是它最有价值、也最值得学习的地方。
