# Claude-code-open 插件与 Skills 架构深度解读

## 1. 结论概览

如果只看表面，`Plugins` 和 `Skills` 都像“扩展 Claude 能力”的机制，但它们在架构里的定位并不相同：

- `Skills` 更像“基于 Markdown 的任务模板 / prompt 命令系统”。
- `Plugins` 更像“可安装扩展包容器”，它不仅能提供 Skills，还能提供 commands、agents、hooks、MCP servers、LSP servers、output styles、userConfig 等一整组能力。
- 二者最终会被统一映射到 Agent 可理解的能力面中：
  - 一部分通过 `/xxx` 命令暴露给用户；
  - 一部分通过 `SkillTool` 暴露给模型；
  - 一部分通过 `hooks`、`MCP`、`LSP`、`AgentDefinition` 注入运行时。

因此，**Skills 是一种能力表达形式，Plugins 是一种能力分发与治理载体**。  
可以把它们理解成：

```text
Skill = 一条可执行的“任务说明书”
Plugin = 一个可安装的“扩展包”，里面可以装很多说明书和更多运行部件
```

---

## 2. 为什么要同时有 Plugins 和 Skills

这是初学者最容易困惑的地方。

### 2.1 只有 Skills 不够

如果系统只有 Skills，那么它很适合做：

- Prompt 模板
- 任务规范
- 限制允许工具
- 引导模型按某种方式工作

但它不适合做：

- 插件安装与卸载
- 版本化缓存
- 市场分发
- 用户启停开关
- 依赖校验
- hooks 生命周期扩展
- MCP/LSP 服务接入

也就是说，Skill 擅长“描述行为”，不擅长“分发生命周期与治理”。

### 2.2 只有 Plugins 也不够

如果系统只有 Plugin Manifest 和代码逻辑，而没有 Skill 这种 Markdown 形态，那么：

- 插件作者要写更重的结构；
- prompt 规范无法自然复用；
- 用户与模型都难以把扩展理解成“可调用的任务”；
- 很多轻量扩展场景会变得过度工程化。

因此项目把两者拆开：

- `Skills` 负责低门槛表达能力；
- `Plugins` 负责打包、分发、版本、启停、依赖、市场与运行时整合。

这个拆分很像：

```text
函数/脚本  vs  包管理器
页面组件   vs  npm 包
Prompt 模板 vs  插件生态
```

---

## 3. 模块分层与职责

围绕插件与 Skills，核心模块可以分成 6 层。

### 3.1 声明层：Markdown + plugin.json

#### Skills 声明

Skills 主要来自：

- 用户目录 `~/.claude/skills`
- 项目目录 `.claude/skills`
- 额外目录 `--add-dir/.../.claude/skills`
- legacy `commands/`
- bundled skills
- plugin 内部的 `skills/`
- MCP 动态生成的 skills

核心加载器：

- `vendor/Claude-code-open/src/skills/loadSkillsDir.ts`

Skill 文件本质是：

- 目录式 `skill-name/SKILL.md`
- 或 legacy 下的 markdown command 文件

#### Plugin 声明

Plugin 的静态声明核心是 `plugin.json`。

关键 schema：

- `vendor/Claude-code-open/src/utils/plugins/schemas.ts`

它定义了插件能声明哪些部件：

- `commands`
- `agents`
- `skills`
- `hooks`
- `mcpServers`
- `lspServers`
- `outputStyles`
- `userConfig`
- `dependencies`

所以 plugin.json 本质上是插件能力清单。

### 3.2 发现与加载层

#### Skills 加载

核心入口：

- `getSkillDirCommands(cwd)` in `src/skills/loadSkillsDir.ts`

职责：

- 扫描多个来源目录
- 解析 frontmatter
- 构造成统一 `Command`
- 去重
- 管理 conditional skills
- 管理动态发现 skills

#### Plugins 加载

核心入口：

- `loadAllPluginsCacheOnly()`
- `loadAllPlugins()`
- `assemblePluginLoadResult()`

代码：

- `vendor/Claude-code-open/src/utils/plugins/pluginLoader.ts`

职责：

- 读取 settings 中启用的插件声明
- 从 marketplace / session inline / builtin 三类来源收集插件
- 合并冲突
- 依赖校验与 demote
- 产出 `enabled/disabled/errors`

### 3.3 能力转换层

这一层非常关键。系统不会直接把“文件”交给模型，而是把它们**转换为统一运行时对象**。

#### Skill -> Command

核心函数：

- `parseSkillFrontmatterFields()`
- `createSkillCommand()`

都在：

- `vendor/Claude-code-open/src/skills/loadSkillsDir.ts`

转换结果是统一的 `Command` 对象，类型通常为 `type: 'prompt'`。

#### Plugin 内的 commands / skills -> Command

核心文件：

- `vendor/Claude-code-open/src/utils/plugins/loadPluginCommands.ts`

这里做了两件事：

- plugin `commands/` 转成 `Command`
- plugin `skills/` 也转成 `Command`

所以从运行时角度，**Skill 和 slash command 最终落点是同一个抽象：`Command`**。  
区别只在来源、命名空间、默认语义和部分字段。

#### Plugin agents -> AgentDefinition

核心文件：

- `vendor/Claude-code-open/src/utils/plugins/loadPluginAgents.ts`

作用：

- 读取 agent markdown
- 解析 frontmatter
- 生成 `AgentDefinition`

#### Plugin hooks -> PluginHookMatcher

核心文件：

- `vendor/Claude-code-open/src/utils/plugins/loadPluginHooks.ts`

作用：

- 把 plugin hook 配置转换为按事件分类的 matcher
- 注册到全局 hook 回调表

### 3.4 聚合层

统一命令入口：

- `vendor/Claude-code-open/src/commands.ts`

关键逻辑：

- `getSkills()`
- `loadAllCommands(cwd)`
- `getCommands(cwd)`

它会把以下来源合并：

- bundled skills
- builtin plugin skills
- skill dir commands
- workflow commands
- plugin commands
- plugin skills
- 内建命令
- 动态发现 skills

也就是说，对用户看到的 `/xxx` 世界而言，所有来源最后汇聚到同一张命令表。

### 3.5 工具暴露层

统一工具入口：

- `vendor/Claude-code-open/src/tools.ts`

这里把 `SkillTool` 注册进基础工具集。  
也就是说，模型并不是“直接扫描技能目录”，而是通过工具协议来调用 skill。

相关文件：

- `vendor/Claude-code-open/src/tools/SkillTool/SkillTool.ts`

### 3.6 运行时刷新与状态同步层

几个关键模块：

- `src/utils/plugins/refresh.ts`
- `src/utils/plugins/reconciler.ts`
- `src/services/plugins/PluginInstallationManager.ts`
- `src/utils/plugins/installedPluginsManager.ts`

它们负责：

- marketplace 物化
- 插件缓存
- 版本化安装记录
- 运行中刷新 active plugin components
- AppState 中的 enabled/disabled/errors/commands 更新

---

## 4. 核心数据结构

这一部分是理解架构的关键。

## 4.1 `Command`

虽然 `Command` 类型定义不在本次主题核心文件里，但插件与 Skills 架构几乎都围绕它转。

在这里，`Command` 承担三类角色：

- 用户可输入的 slash command
- 模型可调用的 prompt skill
- MCP skill 的统一承载体

典型重要字段：

- `type: 'prompt'`
- `name`
- `description`
- `allowedTools`
- `argNames`
- `whenToUse`
- `model`
- `disableModelInvocation`
- `userInvocable`
- `source`
- `loadedFrom`
- `hooks`
- `context`
- `agent`
- `getPromptForCommand()`

其中最关键的是 `getPromptForCommand()`。  
因为这个架构不是把 Skill 当作“预先展开的静态文本”，而是把它建模成**可在调用时动态生成 prompt 的命令对象**。

这带来几个好处：

- 参数替换可以延迟到调用时；
- 环境变量替换可以延迟到调用时；
- shell 注入执行可以延迟到调用时；
- 可以根据调用上下文修改权限与 AppState；
- 可支持 inline/fork 两种执行方式。

## 4.2 `LoadedPlugin`

定义文件：

- `vendor/Claude-code-open/src/types/plugin.ts`

这是插件被加载进系统后的统一结构。

关键字段：

- `name`
- `manifest`
- `path`
- `source`
- `repository`
- `enabled`
- `isBuiltin`
- `commandsPath` / `commandsPaths`
- `agentsPath` / `agentsPaths`
- `skillsPath` / `skillsPaths`
- `outputStylesPath` / `outputStylesPaths`
- `hooksConfig`
- `mcpServers`
- `lspServers`
- `settings`

可以把它理解为：

```text
LoadedPlugin = plugin.json + 解析后的各组件入口 + 运行时启用状态
```

它不是“原始 manifest”，而是“可运行插件实例的描述对象”。

## 4.3 `BuiltinPluginDefinition`

定义文件：

- `vendor/Claude-code-open/src/types/plugin.ts`
- 注册逻辑在 `vendor/Claude-code-open/src/plugins/builtinPlugins.ts`

它表示“随 CLI 一起发布、但仍按插件方式治理”的插件。

这类插件和 bundled skills 的区别很重要：

- bundled skills 更像内置技能库；
- built-in plugins 更像官方预装但可启停的扩展包。

## 4.4 `PluginError`

定义文件：

- `vendor/Claude-code-open/src/types/plugin.ts`

这是插件系统的重要设计点。  
项目没有只用字符串错误，而是定义了具备上下文字段的判别联合类型。

例如：

- `plugin-not-found`
- `manifest-validation-error`
- `marketplace-load-failed`
- `component-load-failed`
- `mcp-config-invalid`
- `hook-load-failed`
- `marketplace-blocked-by-policy`

意义在于：

- UI 能做更精确展示；
- 日志和诊断更稳定；
- 后续新增错误类型时不会破坏字符串匹配逻辑。

## 4.5 `PluginManifest`

schema 定义在：

- `vendor/Claude-code-open/src/utils/plugins/schemas.ts`

从架构角度，它有三个重点：

### A. 元数据

- 名称、版本、作者、仓库、关键词、依赖

### B. 组件声明

- `commands`
- `agents`
- `skills`
- `hooks`
- `mcpServers`
- `lspServers`
- `outputStyles`

### C. 配置声明

- `userConfig`

这说明插件架构不仅是“发现文件”，还把“用户配置接口”作为一等公民纳入 manifest。

---

## 5. Skills 架构深度解读

## 5.1 Skill 的本质：Markdown 驱动的可执行 Prompt

Skill 的核心不是“文档”，而是“可解析、可参数化、可带权限约束、可延迟执行的 prompt 程序”。

Skill 解析的关键 frontmatter 字段包括：

- `description`
- `allowed-tools`
- `arguments`
- `argument-hint`
- `when_to_use`
- `version`
- `model`
- `disable-model-invocation`
- `user-invocable`
- `hooks`
- `context`
- `agent`
- `effort`
- `shell`
- `paths`

这说明一个 Skill 同时描述了：

- 给谁用
- 什么时候用
- 可以用哪些工具
- 是否允许用户直接 `/skill`
- 是否能被模型主动调用
- 应该 inline 执行还是 fork 到子 agent
- 在哪些文件路径触达后才激活

所以 Skill 不是单纯 prompt 文本，而是一个“带元数据的执行单元”。

## 5.2 为什么 Skill 会被转换为 `Command`

这是整个设计最聪明的地方之一。

如果单独设计一套 `Skill` 类型，会出现：

- slash command 一套执行链
- skill 一套执行链
- plugin skill 再一套执行链
- MCP skill 再一套执行链

复杂度会迅速爆炸。

项目直接把它们统一成 `Command`，于是：

- 用户输入 `/foo`
- 模型通过 `SkillTool` 调用 `foo`
- plugin skill `foo`
- MCP skill `foo`

最终都能复用同一套命令模型和 prompt 生成逻辑。

这属于典型的“统一中间表示”设计。

## 5.3 Skill 的加载来源与优先级

`getSkillDirCommands(cwd)` 会并行加载：

- managed skills
- user skills
- project skills
- additional dirs skills
- legacy commands

随后做：

- 扁平化合并
- 按 realpath 去重
- 条件 skill 拆分

这里有两个重要设计点。

### A. 用 `realpath` 去重，而不是仅按名称去重

函数：

- `getFileIdentity()`

目的：

- 解决软链接重复加载
- 解决父目录重叠扫描
- 避免同一个文件经不同路径被加载两次

这是一个很工程化、也很务实的设计。

### B. `paths` 条件激活机制

带 `paths` frontmatter 的 skill 不会一开始就全部暴露，而是先进入：

- `conditionalSkills`

当用户读写的文件路径匹配时，才通过：

- `activateConditionalSkillsForPaths()`

把 skill 挪到：

- `dynamicSkills`

这相当于做了一层“按工作上下文懒激活”的技能系统。

好处：

- 降低无关技能噪音
- 避免模型看到太多不相关技能
- 使技能更接近“文件上下文触发规则”

## 5.4 动态技能发现

另一个很有意思的机制是：

- `discoverSkillDirsForPaths()`
- `addSkillDirectories()`
- `getDynamicSkills()`

这套机制会在文件操作过程中，沿文件路径向上寻找嵌套的 `.claude/skills` 目录，并动态加入运行时。

它体现了项目对“局部上下文”的重视：

- 根目录技能是全局的；
- 子目录技能是局部的；
- 离当前文件越近的技能优先级越高。

本质上，这是把“目录作用域”引入了技能系统。

## 5.5 Skill 调用时的关键处理

Skill 真正执行时，不是直接返回原文，而是经过一系列运行时加工：

### A. 参数替换

- `substituteArguments()`

### B. 特殊变量替换

- `${CLAUDE_SKILL_DIR}`
- `${CLAUDE_SESSION_ID}`

插件 skill 还会替换：

- `${CLAUDE_PLUGIN_ROOT}`
- `${user_config.X}`

### C. shell 命令注入执行

- `executeShellCommandsInPrompt()`

这允许 Skill Markdown 中嵌入 shell 执行结果，再把结果拼进 prompt。

### D. 权限放大为临时 alwaysAllowRules

运行 shell 注入时，会把 `allowedTools` 注入到临时的 permission context 中，使 skill 的可用工具边界与声明一致。

### E. 对 MCP skills 做信任收缩

源码明确禁止对 `loadedFrom === 'mcp'` 的 skill 执行 inline shell 注入，因为它们是远程不可信来源。

这个边界说明作者并没有把所有 skills 视为同等可信。

---

## 6. Plugin 架构深度解读

## 6.1 Plugin 的本质：扩展包装配与治理系统

Plugin 不是一个“单文件扩展”，而是一个**包含多个能力部件的装配单元**。

一个插件可以同时贡献：

- prompt commands
- skills
- agents
- hooks
- MCP servers
- LSP servers
- output styles
- plugin settings

所以 plugin 系统解决的是“如何把外部能力安全地、可治理地接到 Agent Runtime 上”。

## 6.2 Plugin 的三类来源

插件加载最终会汇总三类来源：

### A. marketplace plugins

来自 settings 中的 `plugin@marketplace`

### B. session-only plugins

来自：

- `--plugin-dir`
- SDK inline plugins

### C. built-in plugins

随 CLI 提供，但仍能启停。

在 `assemblePluginLoadResult()` 中，这三类来源会被一起合并。

## 6.3 插件加载的三层模型

`refresh.ts` 的注释把这个系统概括得很好，可以总结为：

### Layer 1: intent

用户 / 项目 / 管理策略在 settings 里声明“我想启用哪些插件、哪些市场”。

### Layer 2: materialization

`reconcileMarketplaces()` 把 marketplace 声明真正物化到本地缓存目录与 `known_marketplaces.json`。

### Layer 3: active components

`refreshActivePlugins()` 把插件里的 commands、agents、hooks、MCP、LSP 刷进当前运行会话。

这是理解插件系统最重要的认知模型。

很多初学者会混淆：

- “settings 里写了”不等于“插件已经安装到了本地”
- “本地缓存里有了”也不等于“当前会话已经切换到新插件状态”

必须区分这三层。

## 6.4 `loadAllPlugins()` 与 `loadAllPluginsCacheOnly()`

这两个函数经常让人迷惑。

### `loadAllPlugins()`

偏“刷新型 / 完整型”加载：

- 可以触发更完整的加载流程
- 用于显式刷新与安装场景

### `loadAllPluginsCacheOnly()`

偏“启动快路径 / 只读缓存”加载：

- 尽量不做重型工作
- 启动时优先读缓存
- 适合查询当前已装插件能力

项目通过这两个入口，把“启动性能”和“加载准确性”分离开来。

这是一个典型的双通道设计：

```text
快路径：启动时先尽量读缓存
慢路径：需要刷新时走完整加载
```

## 6.5 插件组件是如何拆开加载的

Plugin 不会被一次性展开成一个巨对象，而是按部件加载。

### commands / skills

文件：

- `src/utils/plugins/loadPluginCommands.ts`

这里同时负责：

- `getPluginCommands()`
- `getPluginSkills()`

它们都依赖 `loadAllPluginsCacheOnly()` 返回的 enabled plugin 列表，再去读各自目录。

### agents

文件：

- `src/utils/plugins/loadPluginAgents.ts`

### hooks

文件：

- `src/utils/plugins/loadPluginHooks.ts`

### MCP / LSP

在插件刷新阶段再做整合：

- `src/utils/plugins/mcpPluginIntegration.ts`
- `src/utils/plugins/lspPluginIntegration.ts`

这种拆分的优点是：

- 避免所有组件都在启动时同步重载
- 不同消费方只加载自己关心的部分
- 更容易单独清缓存与热更新

## 6.6 Plugin command 与 plugin skill 的区别

在实现上，两者都走 `loadPluginCommands.ts`，都会变成 `Command`。

但它们仍有语义区别：

### plugin command

- 来自 plugin 的 `commands/`
- 更偏 slash command
- `progressMessage` 往往是 `running`

### plugin skill

- 来自 plugin 的 `skills/`
- 更偏技能 / prompt 模板
- `loadedFrom: 'plugin'`
- `progressMessage` 更偏 `loading`
- 会注入 `Base directory for this skill`
- 支持 `${CLAUDE_SKILL_DIR}`

这说明：**它们共享抽象，但保留来源语义。**

## 6.7 插件 hooks 机制

插件可以通过 hooks 在多个生命周期点挂接逻辑。

`loadPluginHooks()` 的过程是：

1. 加载所有 enabled plugins
2. 提取 `hooksConfig`
3. 把 hook 配置转成带 plugin 上下文的 matcher
4. 按事件聚合
5. 原子式 clear + register

这套设计的重点有两个：

### A. hook matcher 附带 plugin 上下文

例如：

- `pluginRoot`
- `pluginName`
- `pluginId`

这样 hook 在运行时知道自己来自哪个插件。

### B. clear + register 是原子切换

源码特意避免“先清掉旧 hooks，但新 hooks 还没装回去”的中间空窗状态。  
这说明作者踩过热更新时 hook 丢失的坑，并修正成更安全的替换过程。

## 6.8 插件 agent 的安全边界

`loadPluginAgents.ts` 里有一个很重要的安全策略：

对于 plugin agent，以下 frontmatter 即使写了也会被忽略：

- `permissionMode`
- `hooks`
- `mcpServers`

原因很明确：

- plugin 是第三方扩展；
- agent 文件如果可以再偷偷声明 hooks / MCP / 权限，就会绕过安装时的信任边界。

这说明项目把“manifest 安装时信任”和“agent 文件运行时信任”分开了。

这是一个很值得学习的设计细节。

---

## 7. 启动与运行时关键业务流程

下面把对初学者最重要的几条链路串起来。

## 7.1 启动时：插件与 Skills 如何进入系统

### 主流程

```text
main.tsx 启动
  -> initBuiltinPlugins()
  -> initBundledSkills()
  -> initializeVersionedPlugins()
  -> 后续命令/工具需要时调用 getCommands() / getTools()
  -> getCommands() 触发技能、插件命令、插件技能的聚合
  -> getTools() 把 SkillTool 等注册给模型
```

这里有两个重点：

- 插件和 skills 并不是都在启动时一次性“完全加载”；
- 很多组件是延迟到真正需要时才读取。

这是一种“启动轻、运行时按需展开”的策略。

## 7.2 用户执行 `/skills` 或输入 `/某个技能`

### `/skills`

- 只是打开技能菜单 UI；
- 命令列表来自 `context.options.commands`；
- 这些 commands 已经是聚合后的统一结果。

### `/某个技能`

```text
用户输入 /xxx
  -> commands.ts 提供聚合命令表
  -> 找到对应 Command
  -> 调用 Command.getPromptForCommand()
  -> 参数/变量/shell 注入处理
  -> 生成最终 prompt
  -> 交给主对话执行链
```

## 7.3 模型主动调用 Skill

### 主流程

```text
模型看到 SkillTool
  -> SkillTool.getAllCommands()
  -> 拿到本地 commands + MCP skills
  -> 按名字找到 Command
  -> 根据 Command.context 决定 inline 还是 fork
  -> 执行 getPromptForCommand()
  -> 结果并回主会话，或在子 agent 中完成
```

这里能看出一个关键事实：

**Skill 对模型来说不是目录扫描结果，而是工具系统中的一个受控调用目标。**

## 7.4 插件市场后台安装

后台安装管理器：

- `PluginInstallationManager.ts`

主流程：

```text
启动后后台检查 declared marketplaces
  -> diffMarketplaces()
  -> reconcileMarketplaces()
  -> marketplace 安装/更新
  -> 若有新 marketplace 安装成功
       -> refreshActivePlugins()
  -> 若只是 marketplace 更新
       -> 标记 needsRefresh，等待用户 /reload-plugins
```

这是一个兼顾体验的设计：

- 新装成功时尽量自动刷新，避免“装了但不可用”；
- 更新时不强行打断用户当前会话，而是提示刷新。

## 7.5 `/reload-plugins` 如何真正生效

核心函数：

- `refreshActivePlugins()`

流程：

```text
清缓存
  -> loadAllPlugins() 完整重载
  -> getPluginCommands()
  -> getAgentDefinitionsWithOverrides()
  -> loadPluginMcpServers()
  -> loadPluginLspServers()
  -> 更新 AppState.plugins / agentDefinitions
  -> pluginReconnectKey + 1
  -> reinitializeLspServerManager()
  -> loadPluginHooks()
```

所以 `/reload-plugins` 并不是简单“再扫一次目录”，而是一次**对运行时能力面的整体重建**。

---

## 8. 关键算法与机制

这部分不是传统算法题意义上的算法，而是构成系统稳定性的关键机制。

## 8.1 基于 `realpath` 的技能去重

用途：

- 避免软链接重复
- 避免重叠目录重复
- 保证同一 skill 文件只加载一次

核心思想：

- 不按名字去重
- 不按相对路径去重
- 按文件真实身份去重

这是比“字符串去重”更可靠的工程做法。

## 8.2 条件技能激活算法

用途：

- 只在相关文件被触达时激活技能

流程：

```text
启动时：
  带 paths 的 skill -> conditionalSkills

文件操作时：
  activateConditionalSkillsForPaths(filePaths, cwd)
    -> 将 filePath 转为相对 cwd
    -> 用 ignore() 按 gitignore 语义匹配
    -> 命中则移到 dynamicSkills
```

优点：

- 技能暴露更精准
- 降低提示词污染
- 让技能与代码上下文建立显式关联

## 8.3 深度优先的动态技能优先级

`discoverSkillDirsForPaths()` 和 `addSkillDirectories()` 会优先使用更深层目录的技能定义。

规则：

- 路径更深，说明离当前文件更近；
- 更近的局部规则应覆盖更远的全局规则。

这其实是一种“目录作用域覆盖算法”。

## 8.4 插件三源合并

在 `assemblePluginLoadResult()` 中，插件来自：

- marketplace
- session inline
- builtin

再经过：

- merge
- dependency verify
- demote

这体现的不是算法复杂度，而是“组合策略”：

- 多来源输入
- 统一结构
- 冲突治理
- 依赖失败自动降级

## 8.5 cache-only / full-load 双加载通道

这是插件系统里最重要的性能策略之一。

思路：

- 启动阶段优先 cache-only；
- 显式刷新与安装阶段再 full-load。

收益：

- 常规启动更快；
- 插件市场生态仍可保持完整功能；
- 避免所有用户都为少数刷新场景付成本。

## 8.6 Hook 的原子替换

不是单纯“清空再重建”，而是：

- 保留旧值直到新值准备好
- 在交换点一次性切换

这避免了热更新期间某些 hook 事件丢失。

这是很典型的运行时热更新一致性设计。

---

## 9. 设计亮点与取舍

## 9.1 统一抽象：把 Skill、Plugin Skill、MCP Skill 都落到 `Command`

优点：

- 复用调用链
- 降低类型数量
- slash command 与 skill 能共享执行模型

代价：

- `Command` 语义被拉大；
- 阅读源码时不容易第一眼看出“这是 slash command 还是 skill”。

## 9.2 轻表达 + 重治理分离

优点：

- Skill 编写门槛低
- Plugin 治理能力强
- 两者可组合

代价：

- 初学者会困惑“我该写 skill 还是 plugin？”

我的理解是：

- 只想写任务模板，用 skill；
- 需要安装、市场、hooks、MCP、agents、多组件分发，用 plugin。

## 9.3 延迟加载很多，但不是完全懒到不可控

项目没有把所有东西都提前展开，也没有把一切都推到最后一刻。

而是做了平衡：

- bundled skills 启动同步注册
- plugin 能力大多按需加载
- marketplace 安装后台进行
- 显式刷新时执行完整重建

这说明作者非常在意：

- CLI 启动速度
- 会话连续性
- 大型扩展生态下的可维护性

## 9.4 对信任边界有明确区分

例如：

- MCP skill 不允许执行 inline shell prompt 注入
- plugin agent 忽略 `permissionMode/hooks/mcpServers`
- enterprise policy 会拦 marketplace source

这些点说明系统不是“能扩展就全放开”，而是一直在做来源分级信任。

---

## 10. 初学者应该如何理解这套架构

我建议按下面的心智模型来记。

## 10.1 先记住一句话

```text
Skills 负责“怎么做”
Plugins 负责“怎么装、怎么管、怎么接进系统”
```

## 10.2 再记住统一落点

对于 prompt 类能力，最后几乎都会落到：

- `Command`

对于模型调用，最后通过：

- `SkillTool`

对于插件整体治理，最后通过：

- `LoadedPlugin`
- `loadAllPlugins*()`
- `refreshActivePlugins()`

## 10.3 最后记住三条主线

### 主线 1：声明

- `SKILL.md`
- `plugin.json`

### 主线 2：加载

- `loadSkillsDir.ts`
- `pluginLoader.ts`
- `loadPluginCommands.ts`
- `loadPluginAgents.ts`
- `loadPluginHooks.ts`

### 主线 3：运行

- `commands.ts`
- `tools.ts`
- `SkillTool.ts`
- `refresh.ts`

如果把这三条线看清楚，这套架构就不再乱。

---

## 11. 推荐阅读顺序

如果你要继续深入源码，建议按下面顺序读：

1. `vendor/Claude-code-open/src/skills/loadSkillsDir.ts`
   先理解 Skill 如何被转成 `Command`。
2. `vendor/Claude-code-open/src/utils/plugins/schemas.ts`
   再理解 plugin.json 能声明什么。
3. `vendor/Claude-code-open/src/utils/plugins/pluginLoader.ts`
   看插件如何被发现、合并和缓存。
4. `vendor/Claude-code-open/src/utils/plugins/loadPluginCommands.ts`
   看 plugin command / skill 如何复用 `Command` 抽象。
5. `vendor/Claude-code-open/src/tools/SkillTool/SkillTool.ts`
   看模型如何真正调用技能。
6. `vendor/Claude-code-open/src/utils/plugins/refresh.ts`
   看运行时刷新如何把插件能力重新接到会话里。

---

## 12. 最终总结

“插件与 Skills 架构”的核心思想，不是简单地把扩展塞进 CLI，而是构建了一套**统一能力抽象 + 分层加载 + 运行时治理 + 信任边界控制**的系统。

其关键设计可以浓缩为四句话：

- 用 `Skill` 解决轻量 prompt 能力表达；
- 用 `Plugin` 解决安装、分发、依赖、配置和多组件整合；
- 用 `Command` 作为 prompt 类能力的统一中间表示；
- 用 `SkillTool`、`refreshActivePlugins()`、`loadAllPlugins*()` 把这些能力接入 Agent 运行时。

所以，这套架构真正强的地方不是“能扩展”，而是：

**它把扩展做成了一个可组合、可治理、可热更新、可控信任边界的 Agent 平台能力层。**
