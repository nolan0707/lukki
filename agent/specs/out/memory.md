# 以源码为准，重构 Claude Code 的 Memory / Harness 设计分析

基于 `vendor/Claude-code-open/src/` 的源码，原概要里“Claude Code 通过三层级联记忆架构解决上下文熵增”的说法并不准确。更贴近源码的表述是：

- `CLAUDE.md` 体系负责**稳定指令注入**，它是 instruction memory，不是 `MEMORY.md` 的静态层替身。
- `auto-memory` / `agent memory` / `team memory` 负责**跨会话持久化记忆**，其入口文件是 `MEMORY.md`，但真正的内容通常保存在按主题拆分的 memory files 中。
- `microcompact` / `autocompact` / `context collapse` 负责**当前会话上下文压缩**，这是另一条独立子系统，不等于 persistent memory。

换句话说，Claude Code 不是把“所有记忆”收敛成单一三层结构，而是把**指令、持久化事实、会话上下文**拆成了三种不同治理对象，并分别交给不同 Harness 机制处理。

## 1. `CLAUDE.md` 是“指令记忆链”，不是 `MEMORY.md` 的一层

源码明确写出了 `CLAUDE.md` 的加载优先级与发现规则：

- 加载顺序分为 `Managed -> User -> Project -> Local` 四类，项目内会向上遍历目录，并同时检查 `CLAUDE.md`、`.claude/CLAUDE.md`、`.claude/rules/*.md`。这是一条**instruction loading chain**，不是用户概要里所谓“Structural Project Memory 只由 CLAUDE.md 构成”的简化版。见 `vendor/Claude-code-open/src/utils/claudemd.ts:1-26`。
- `CLAUDE.md` 支持 `@include`，且只接受文本类文件、限制最大 include 深度、会做循环引用去重。见 `vendor/Claude-code-open/src/utils/claudemd.ts:18-26`, `vendor/Claude-code-open/src/utils/claudemd.ts:336-399`, `vendor/Claude-code-open/src/utils/claudemd.ts:448-535`, `vendor/Claude-code-open/src/utils/claudemd.ts:537-685`。
- REPL 启动时会主动加载这些 memory/instruction files 到 `readFileState`，并记录哪些内容经过了 frontmatter/comment stripping 或 `MEMORY.md` truncation。见 `vendor/Claude-code-open/src/screens/REPL.tsx:3797-3816`。
- 系统提示词构建时会把 memory section 作为动态 section 注入。见 `vendor/Claude-code-open/src/constants/prompts.ts:491-506`。

因此，更准确的说法应当是：`CLAUDE.md` 体系是 Claude Code 的**稳定指令层**，作用是给模型持续施加行为约束和项目规则；它与 auto-memory 共享“memory files”这一装载通道，但语义上并不是同一种记忆。

## 2. 持久化 Memory 的真实结构：`MEMORY.md` 是入口索引，不是全部内容

### 2.1 `MEMORY.md` 的定位

- auto-memory 的入口文件名固定为 `MEMORY.md`，并有明确的行数/体积上限：`200` 行、`25_000` bytes。见 `vendor/Claude-code-open/src/memdir/memdir.ts:34-39`。
- `truncateEntrypointContent()` 会在超限时裁剪 `MEMORY.md`，并附加 warning，说明它是一个**受限入口索引**。见 `vendor/Claude-code-open/src/memdir/memdir.ts:49-103`。
- `buildMemoryLines()` 与 extraction prompt 都把 `MEMORY.md` 定义为“index, not a memory”，要求每条只保留一行 hook，真实内容写进独立 topic file。见 `vendor/Claude-code-open/src/memdir/memdir.ts:219-234`, `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:68-82`。
- `getMemoryFiles()` 在 auto-memory 开启时，只会自动把 `getAutoMemEntrypoint()` 指向的 `MEMORY.md` 注入上下文；topic files 并不会像 `CLAUDE.md` 链那样被全量自动注入。见 `vendor/Claude-code-open/src/utils/claudemd.ts:979-992`。

所以，源码支持“`MEMORY.md` 是轻量索引入口”这一判断，但**不支持**“第二层 memory 本体就是一个由 `MEMORY.md` 驱动的全量外部索引层”这种过度抽象。更准确地说，它是：

- 自动注入：`MEMORY.md`
- 按需读取：topic memory files
- 写入目标：topic file + `MEMORY.md` pointer


### 2.2 持久化 memory 的类型边界

Claude Code 并没有把“架构、代码模式、文件结构”鼓励写入 memory；相反，它显式禁止这么做：

- memory taxonomy 只有四类：`user`、`feedback`、`project`、`reference`。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:14-31`。
- 源码明确写明：`Code patterns, conventions, architecture, file paths, or project structure` 不应该保存进 memory，因为这些内容可从当前项目状态推导。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:4-7`, `vendor/Claude-code-open/src/memdir/memoryTypes.ts:183-195`。

这点和原概要有明显出入。原文把 `code-patterns.md`、`decisions.md` 一类内容视为 memory 主题文件的典型例子，但源码的设计哲学恰恰是在**限制 memory 的语义范围**：只保存“不能从 repo 当前状态推导出来”的协作信息。

### 2.3 auto-memory、team memory、agent memory 是三条变体

- auto-memory 的根目录默认在 `~/.claude/projects/<sanitized-git-root>/memory/`，也支持 env/settings override。见 `vendor/Claude-code-open/src/memdir/paths.ts:79-90`, `vendor/Claude-code-open/src/memdir/paths.ts:198-235`。
- team memory 是另一套目录与 `MEMORY.md`，在 combined prompt 中与 private auto-memory 并存。见 `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:96-153`, `vendor/Claude-code-open/src/utils/claudemd.ts:994-1007`。
- agent memory 也复用同一套 `MEMORY.md + topic files` 机制，只是 scope 分成 `user/project/local`。见 `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:12-18`, `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:46-64`, `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:131-177`。

## 3. Harness 对 memory 的核心约束：限制保存内容，而不是鼓励“记住一切”

`buildMemoryLines()` 给出的系统指令说明了 Harness 的真实设计哲学：

- memory 是 “persistent, file-based memory system”，目标是跨会话累积用户画像、协作偏好、非代码事实。见 `vendor/Claude-code-open/src/memdir/memdir.ts:236-255`。
- 若用户要求记住/忘记某事，agent 应立即保存/删除相应 memory。见 `vendor/Claude-code-open/src/memdir/memdir.ts:241-244`。
- 保存 memory 是一个**两步过程**：先写 topic file，再把 pointer 加进 `MEMORY.md`。见 `vendor/Claude-code-open/src/memdir/memdir.ts:219-234`。

但要注意，源码只证明了“这是 prompt 级写作纪律”，**没有看到一个显式的事务性 Harness 保证**去强制“topic file 写入成功后才允许更新 `MEMORY.md`”。原概要中的“只有 FileWrite 真正成功后才更新索引”更像设计推断，而不是这里能直接落到代码行号上的机制。

源码里真正能确认的，是另外两种治理：

- 写权限被严格收窄。memory extraction / autoDream fork 只能用 `Read/Grep/Glob`、只读 Bash，以及**仅限 memoryDir 内**的 `Edit/Write`。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:166-221`。
- 若主 agent 已经在本轮直接写过 memory，后台 extraction 会跳过，避免重复写入。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:112-148`, `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:345-359`。

因此，源码能支持的更谨慎结论是：Claude Code 的 Harness 对 memory 的治理重点是**写入边界控制、重复写入互斥、内容类型约束**，而不是显式实现了一个“索引更新事务”。

## 4. “怀疑性检索”是源码明确存在的

这部分原概要基本成立，而且源码证据很强：

- `MEMORY_DRIFT_CAVEAT` 明确要求：memory 可能过期，使用前必须读取当前文件/资源重新验证；若 memory 与当前观察冲突，应以当前事实为准，并更新或删除 stale memory。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:197-202`。
- `## When to access memories` 明确要求：用户显式要求 recall/check/remember 时必须访问 memory；若用户要求 ignore memory，则要把 `MEMORY.md` 当成空的。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:205-222`。
- `## Before recommending from memory` 进一步要求：凡是 memory 中提到具体 file/function/flag，都必须先检查文件是否存在、grep 确认符号是否仍存在。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:224-256`。

这说明 Claude Code 的 Harness 并不把 memory 当作 authoritative truth，而是把它当作**时间戳化的线索**。这是整个 memory 设计最关键的哲学之一。

## 5. 后台记忆维护不是一个机制，而是两个异步子系统

### 5.1 `extractMemories`: turn-end 的轻量抽取器

`extractMemories` 会在 query loop 正常完成后，经由 `handleStopHooks()` fire-and-forget 触发。见 `vendor/Claude-code-open/src/query/stopHooks.ts:133-156`。

它的特征是：

- 运行在 forked agent 中，继承主对话上下文和 prompt cache。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:1-14`, `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:415-427`。
- 只分析最近新增的消息，并且 prompt 明确禁止再去 grep 代码、读源码、跑 git 做验证。见 `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:29-43`。
- 工具预算有限，最大 `5` turns。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:421-427`。
- 若写出了 memory file，会生成 `memory_saved` 系统消息反馈。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:463-496`, `vendor/Claude-code-open/src/utils/messages.ts:4465`。

这说明 Claude Code 不是只靠主 agent 主动维护 memory，还在 turn 结束后挂了一个**补写型后台抽取器**。

### 5.2 `autoDream`: 跨 session 的 consolidation

`autoDream` 才是更接近“睡眠整合”的那套机制：

- 入口注释直接写明 gate 顺序：`Time -> Sessions -> Lock`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:1-12`。
- 默认触发阈值正是 `24` 小时和 `5` 个 session。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:58-66`。
- 实际执行时会先读 `lastConsolidatedAt`，再数自上次以来被触碰过的 sessions，最后争抢 `.consolidate-lock`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:130-190`, `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:25-124`。
- 真正的 consolidation prompt 分四阶段：`Orient`、`Gather recent signal`、`Consolidate`、`Prune and index`。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:26-64`。
- prompt 明确要求把相对时间改写成绝对时间，并把 `MEMORY.md` 维持在 `200` 行以内且约 `25KB` 以下，每行约 `150` 字符。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:48-60`。

所以，原概要中的“三门系统 + 四阶段整合逻辑”是基本成立的，但应当归因给 `autoDream`，而不是整个 memory subsystem 的统一运行方式。

## 6. KAIROS 模式下，新增记忆不是直接写 `MEMORY.md`

这是原概要完全没有提到，但对 Harness 设计非常重要的一点：

- `buildAssistantDailyLogPrompt()` 明确说明，在 assistant-mode / KAIROS 下，新记忆应 append 到按日期命名的 daily log，而不是实时整理 `MEMORY.md`。见 `vendor/Claude-code-open/src/memdir/memdir.ts:318-369`。
- `getAutoMemDailyLogPath()` 给出了路径模式：`<autoMemPath>/logs/YYYY/MM/YYYY-MM-DD.md`。见 `vendor/Claude-code-open/src/memdir/paths.ts:237-251`。
- 同一处注释明确写道：nightly `/dream` 会把这些 logs 蒸馏成 topic files + `MEMORY.md`。见 `vendor/Claude-code-open/src/memdir/paths.ts:237-245`, `vendor/Claude-code-open/src/memdir/memdir.ts:321-325`。

这意味着 Claude Code 的 memory 设计不是静态的；在 KAIROS 模式下，它会切换成**append-only log -> nightly distillation** 的两段式架构。

## 7. 上下文压缩确实存在，但它是“会话治理”，不是“持久化记忆”

原概要把 context compaction 直接并入 memory 体系，这会混淆两个层面。源码里它们是相邻但独立的：

- query pipeline 的顺序是：`snip` -> `microcompact` -> `context collapse` -> `autocompact`。见 `vendor/Claude-code-open/src/query.ts:396-467`。
- `autocompact` 的阈值是 `effectiveContextWindow - 13_000`，其中会为 compact summary 预留最多 `20_000` output tokens。见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:28-49`, `vendor/Claude-code-open/src/services/compact/autoCompact.ts:62-91`。
- 若连续 compact 失败 `3` 次，会触发 circuit breaker，停止继续重试。见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:67-70`, `vendor/Claude-code-open/src/services/compact/autoCompact.ts:257-265`。
- `microcompact` 主要清理的是旧 tool results，压缩目标是当前消息数组，不是 memory files。见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:40-50`, `vendor/Claude-code-open/src/services/compact/microCompact.ts:253-293`。
- 模型上下文窗口默认是 `200k`，支持 `1M` 上下文的模型才会升级；所以“如 200K 或 1M 的 token 窗口”这一点是成立的，但属于 `context management`，不是 memory file 机制。见 `vendor/Claude-code-open/src/utils/context.ts:8-13`, `vendor/Claude-code-open/src/utils/context.ts:51-98`。

因此，更准确的定位是：Claude Code 用 compaction 解决**当前会话窗口压力**，用 memory 解决**跨会话持续性**。二者协同，但不应混成一个 subsystem。

## 8. 结论：Claude Code 的 Harness 设计哲学是什么

从源码看，Claude Code 围绕 “memory” 的 Harness 哲学可以概括为四句话：

1. **把不同类型的“记忆”拆开治理。**  
   `CLAUDE.md` 管规则，`MEMORY.md + topic files` 管跨会话事实，`compact` 管当前对话窗口；三者有协同，但边界清晰。主要证据见 `vendor/Claude-code-open/src/utils/claudemd.ts:1-26`, `vendor/Claude-code-open/src/utils/claudemd.ts:979-1007`, `vendor/Claude-code-open/src/query.ts:396-467`。

2. **只保存 repo 当前状态无法推导的东西。**  
   用户偏好、协作反馈、项目背景、外部系统指针属于 memory；代码模式、架构、文件路径不属于。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:4-7`, `vendor/Claude-code-open/src/memdir/memoryTypes.ts:183-195`。

3. **memory 不是事实源，而是带时间属性的线索。**  
   使用前要验证，冲突时相信当前状态，并回写修正 stale memory。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:197-202`, `vendor/Claude-code-open/src/memdir/memoryTypes.ts:245-256`。

4. **记忆维护异步化、后台化。**  
   短周期靠 `extractMemories` 补写，长周期靠 `autoDream` 整合；KAIROS 模式下甚至先写 daily log，再由 `/dream` 蒸馏。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:391-496`, `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:122-271`, `vendor/Claude-code-open/src/memdir/memdir.ts:318-369`。

如果把这套设计放回 Harness 视角，一个更准确的总结是：

> Claude Code 并不是在做“把所有历史都压进一个分层记忆体”，而是在做“把不同生命周期、不同可信度、不同用途的信息分发到不同治理回路里”：稳定规则走 `CLAUDE.md`，可演化事实走 memory files，超长会话压力走 compaction，跨 session 演化则交给 extraction 和 dream。

## 补充：auto-memory、team memory、agent memory 三条变体

虽然三者都复用“文件型持久化记忆”这一基本范式，但它们的设计意图和实现边界并不相同：

- auto-memory 面向**主线程 Claude 与当前用户在当前项目中的长期协作关系**。
- team memory 面向**同一项目内多个协作者之间的共享项目记忆**。
- agent memory 面向**某个 agent 类型自身的长期专业化经验积累**。

这三条线共享 `MEMORY.md + topic files` 的总体形态，但在目录布局、prompt 注入方式、后台维护方式、同步语义上明显分化。

### auto-memory：主线程助手的私有项目记忆

设计意图：

- 让 Claude 在同一个项目里跨会话记住用户画像、协作偏好、项目中的非代码背景、外部系统指针等“无法从 repo 当前状态直接推导”的信息。
- 作用对象是“当前用户 + 当前项目 + 主线程助手”，不是团队共享空间，也不是某个专门 agent 的私有知识库。

实现特征：

- 默认目录为 `~/.claude/projects/<sanitized-git-root>/memory/`，支持 env / settings override。见 `vendor/Claude-code-open/src/memdir/paths.ts:79-90`, `vendor/Claude-code-open/src/memdir/paths.ts:198-235`。
- 会进入主线程系统提示词：`loadMemoryPrompt()` 在 auto-memory 开启时构造 memory 行为规则，并作为 `systemPromptSection('memory', ...)` 注入。见 `vendor/Claude-code-open/src/memdir/memdir.ts:419-507`, `vendor/Claude-code-open/src/constants/prompts.ts:491-506`。
- `getMemoryFiles()` 会自动把 auto-memory 的 `MEMORY.md` 入口文件注入上下文。见 `vendor/Claude-code-open/src/utils/claudemd.ts:979-992`。
- 带有最完整的后台维护链路：turn-end 的 `extractMemories` 补写，跨 session 的 `autoDream` consolidation，以及 KAIROS 模式下的 daily log -> nightly dream 蒸馏。见 `vendor/Claude-code-open/src/query/stopHooks.ts:141-156`, `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:391-496`, `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:122-271`, `vendor/Claude-code-open/src/memdir/memdir.ts:318-369`。

因此，auto-memory 可以理解成 Claude Code 的**主助手长期记忆层**。

### team memory：项目级共享记忆

设计意图：

- 把一部分原本只属于“当前用户与 Claude”的记忆，提升为“整个项目协作者都应该共享的非代码知识”。
- 典型内容包括项目背景、团队级反馈规则、外部系统入口等，而不是个人沟通偏好或敏感信息。

实现特征：

- team memory 不是独立于 auto-memory 的平行根系统，而是 auto-memory 的子目录：`<autoMemPath>/team/`。见 `vendor/Claude-code-open/src/memdir/teamMemPaths.ts:80-94`。
- `isTeamMemoryEnabled()` 先依赖 `isAutoMemoryEnabled()`，再看 feature gate，说明 team memory 在产品语义上是 auto-memory 之上的扩展层。见 `vendor/Claude-code-open/src/memdir/teamMemPaths.ts:66-78`。
- combined prompt 会同时暴露 private dir 与 team dir，并将每种 memory type 的 scope 指导嵌入到 type block 中，同时显式要求 team memory 不得保存 secrets。见 `vendor/Claude-code-open/src/memdir/teamMemPrompts.ts:17-100`。
- `getMemoryFiles()` 会同时把 private auto-memory 的 `MEMORY.md` 和 team memory 的 `MEMORY.md` 注入上下文，所以主线程拿到的是“双索引”。见 `vendor/Claude-code-open/src/utils/claudemd.ts:979-1007`。

team memory 最关键的实现差异，在于它多了远端同步子系统：

- 服务目标是 repo-scoped shared memory，服务端通过 `/api/claude_code/team_memory?repo=...` 提供 GET/PUT。见 `vendor/Claude-code-open/src/services/teamMemorySync/index.ts:1-25`, `vendor/Claude-code-open/src/services/teamMemorySync/index.ts:163-167`。
- 同步语义是：pull 时 server wins per-key；push 时按 checksum 做 delta upload，服务端 upsert；本地删除不会传播到服务器，下次 pull 会被恢复。见 `vendor/Claude-code-open/src/services/teamMemorySync/index.ts:14-20`, `vendor/Claude-code-open/src/services/teamMemorySync/index.ts:100-118`。
- 启动时先 initial pull，再启动 watcher；目录变化经由 `fs.watch + debounce` 触发 push。见 `vendor/Claude-code-open/src/services/teamMemorySync/watcher.ts:1-8`, `vendor/Claude-code-open/src/services/teamMemorySync/watcher.ts:129-145`, `vendor/Claude-code-open/src/services/teamMemorySync/watcher.ts:231-305`。
- 为了防止共享目录写入带来的攻击面，team memory 路径校验明显更重，专门处理 null byte、URL-encoded traversal、Unicode normalization、symlink escape。见 `vendor/Claude-code-open/src/memdir/teamMemPaths.ts:17-64`, `vendor/Claude-code-open/src/memdir/teamMemPaths.ts:96-171`, `vendor/Claude-code-open/src/memdir/teamMemPaths.ts:222-256`。

因此，team memory 的本质是**共享型、可同步、带更强安全边界的项目记忆层**。

### agent memory：agent 类型专属的长期记忆

设计意图：

- 不是让主线程 Claude 记住“当前用户和项目”，而是让某个 agent 类型在未来任务里积累自己的专业经验。
- 例如 reviewer、architect、test-runner 这类 agent，可以有自己独立的长期记忆目录，用来保存它在这一角色上的稳定经验。

实现特征：

- agent memory 的作用域围绕 `agentType + scope` 组织，而不是围绕“主线程用户/团队共享”组织。见 `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:12-18`, `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:46-64`。
- scope 分为三种：
  - `user`: `~/.claude/agent-memory/<agentType>/`
  - `project`: `.claude/agent-memory/<agentType>/`
  - `local`: `.claude/agent-memory-local/<agentType>/`
  见 `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:46-64`。
- agent memory 不走主线程 `getMemoryFiles()` 的全局注入链，而是通过 `loadAgentMemoryPrompt()` 把该 agent 的 memory prompt 与其 `MEMORY.md` 内容直接内联进 agent 的 system prompt。见 `vendor/Claude-code-open/src/memdir/memdir.ts:268-316`, `vendor/Claude-code-open/src/tools/AgentTool/agentMemory.ts:131-177`。

这点非常关键：agent memory 更像是 agent 的**角色化长期经验缓存**，而不是主线程系统级 memory 子系统的一部分。源码注释甚至明确写道，`buildMemoryPrompt()` 这一分支是给 agent memory 用的，因为 agent memory “has no getClaudeMds() equivalent”。见 `vendor/Claude-code-open/src/memdir/memdir.ts:268-272`。

从当前源码能看到的范围内，agent memory 也没有像 auto-memory 那样绑定一整套 `extractMemories` / `autoDream` 后台维护链路。它更偏向于让 agent 自己在执行中读写其专属 memory，而不是由主线程后台守护系统统一整理。

### 三者的设计意图与实现区别总结

可以把三者压缩成下表：

| 维度 | auto-memory | team memory | agent memory |
| --- | --- | --- | --- |
| 设计意图 | 主线程 Claude 记住“当前用户 + 当前项目”的长期协作信息 | 团队共享“这个项目里代码之外的重要事实” | 某类 agent 记住“这个角色长期有用的经验” |
| 主键 | 项目 | 项目 + 团队共享 | agentType + scope |
| 默认目录 | `~/.claude/projects/<repo>/memory/` | `<autoMemPath>/team/` | `~/.claude/agent-memory/` / `.claude/agent-memory/` / `.claude/agent-memory-local/` |
| Prompt 注入 | 主线程 `loadMemoryPrompt()` | 主线程 combined prompt | agent 自己的 `loadAgentMemoryPrompt()` |
| 内容注入 | `getMemoryFiles()` 自动注入 `MEMORY.md` | `getMemoryFiles()` 自动注入 private/team 两个 `MEMORY.md` | 直接内联到该 agent 的 system prompt |
| 后台维护 | `extractMemories` + `autoDream` + KAIROS 日志蒸馏 | 共享主线程 memory 使用方式，额外增加远端 sync | 主要依赖 agent 自主管理 |
| 协作模型 | 单用户长期协作 | 多用户共享协作 | 特定 agent 的角色持续学习 |
| 安全重点 | 普通 memory 写入边界 | secret 防护 + 共享目录路径安全 + sync 冲突 | scope 与目录隔离 |

## 补充：为什么说 Harness 对 memory 的治理重点是写入边界控制、重复写入互斥、内容类型约束，而不是“索引更新事务”

前文提到，Claude Code 的 Harness 并没有在源码里显式实现一个“topic file 写成功后再原子提交 `MEMORY.md` 索引”的事务层。更准确的说法是：它把 memory 写入治理拆成了三个更现实的控制面，分别是**写入边界控制、重复写入互斥、内容类型约束**。

### 1. 写入边界控制：先限制“能做什么”，而不是先保证“事务成功”

Claude Code 对 memory 写入的第一层治理不是事务提交，而是工具权限收缩。

`extractMemories` 和 `autoDream` 共用 `createAutoMemCanUseTool(memoryDir)`，其规则非常严格：

- `Read/Grep/Glob` 全放行，因为它们天然只读。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:184-191`。
- `Bash` 只允许通过 `isReadOnly()` 判定的只读命令，任何可写 shell 命令都会被拒绝。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:193-204`。
- `Edit/Write` 只允许目标路径落在 memory 目录内部。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:206-215`。
- 其他工具都会被 deny，并带回明确错误信息。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:217-220`。

这意味着后台 memory agent 根本没有能力去改源码仓库中的普通文件、运行会修改状态的 shell、或写出 memory 目录之外。换句话说，Harness 先把 memory 写入缩进一个很小的“沙箱”。

对应的 extraction prompt 也再次把边界说死：

- 只允许 `Read/Grep/Glob`
- 只允许只读 Bash
- `FILE_EDIT/FILE_WRITE` 仅限 memory directory
- 其他工具如 MCP、Agent、可写 Bash 全部拒绝  
  见 `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:35-43`。

因此，这里的设计哲学不是“任何 memory 写入都必须经过一个强一致事务层”，而是“memory 写入首先必须被限制在受控边界里”。在 agent 架构里，这是更基础、也更实际的治理点。

### 2. 重复写入互斥：避免主线程和后台抽取器同时写同一段 memory

第二层治理是“谁来写”的互斥，而不是“索引如何原子提交”的互斥。

`hasMemoryWritesSince()` 会扫描从上次 cursor 之后的 assistant 消息，查找是否已经出现面向 auto-memory 路径的 `Write/Edit tool_use`。如果主 agent 已经在这一段会话里写过 memory，那么后台 `extractMemories` 就会直接跳过。见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:112-148`。

源码注释写得非常明确：

- 主 agent 的 prompt 本身已经具备完整的 memory 保存能力
- 一旦主 agent 已写，forked extraction 就是冗余的
- 因此 `runExtraction` 会跳过，并把 cursor 前移
- 这样主 agent 和后台 extraction agent 就在“每一轮”上实现了 mutually exclusive  
  见 `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:116-120`, `vendor/Claude-code-open/src/services/extractMemories/extractMemories.ts:345-359`。

这说明 Claude Code 处理的不是数据库意义上的并发事务冲突，而是多 agent 体系下常见的**重复落盘**问题：主线程已经记了，后台就别再记一遍。

同时，extraction prompt 还会预注入 existing memory manifest，并显式要求：

- 写之前先检查现有 memory files
- 优先 update 现有文件而不是新建 duplicate
- 不要写重复 memory  
  见 `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:29-43`, `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:55-82`。

因此，这一层的真实机制是：

- **执行来源互斥**：主线程写过则后台跳过
- **语义级去重**：更新已有 memory，避免重复文件和重复 pointer

它控制的是“写入来源竞争”和“内容重复”，不是 `MEMORY.md` 与 topic file 的两阶段提交。

### 3. 内容类型约束：最重要的是限制“什么值得记”

第三层治理是最有设计哲学意味的一层：Claude Code 并不鼓励“只要有用就记住”，而是先把 memory 的语义空间压得很窄。

源码在 `memoryTypes.ts` 中显式规定：

- memory 只有四类：`user`、`feedback`、`project`、`reference`。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:14-31`。
- 这些 memory 的共同特征是：它们保存的是**不能从当前项目状态推导出来**的信息。见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:1-12`。

同时，`WHAT_NOT_TO_SAVE_SECTION` 明确排除了下列内容：

- code patterns
- conventions
- architecture
- file paths
- git history
- debugging fix recipes
- 已经写进 `CLAUDE.md` 的内容
- 当前会话中的临时任务信息  
  见 `vendor/Claude-code-open/src/memdir/memoryTypes.ts:183-195`。

这里非常关键。Claude Code 并没有把 memory 设计成一个“所有长期信息都能塞进去的外部脑”，而是把它严格限制为协作语义层：

- 用户是谁
- 用户怎么要求你做事
- 项目里代码之外的背景事实
- 外部系统入口

而代码模式、架构、路径、历史这些内容，Harness 认为都应该从 repo 当前状态、`CLAUDE.md` 或 `git` 里重新推导，而不是存进 memory。

这实际上是在从源头防止 memory 污染：如果什么都能记，memory 会迅速退化成一个失真、过期、冗余的信息堆；如果只允许记四类“高价值、不可推导”的事实，memory 才能长期维持可用。

### 4. 为什么说这里没有显式的“索引更新事务”

当前源码能明确看到的，是 prompt 级两步写入纪律：

- Step 1：写 topic file
- Step 2：把 pointer 写进 `MEMORY.md`
- `MEMORY.md` 是 index，不是 memory 正文  
  见 `vendor/Claude-code-open/src/services/extractMemories/prompts.ts:68-82`, `vendor/Claude-code-open/src/memdir/memdir.ts:219-234`。

但源码里并没有看到一个显式事务协调器来保证：

- topic file 成功后，索引才可提交
- 任一步失败时自动回滚
- `topic file + MEMORY.md` 具备原子提交语义
- 或者有两阶段提交/提交日志/rollback hook 这类机制

因此，更符合源码事实的表述是：

- 这是一套**prompt 纪律 + 工具边界 + 来源互斥 + 类型约束**的 memory 治理系统
- 而不是一套带强一致事务语义的 memory storage engine

### 5. 从 Harness 设计哲学看，这三种控制为什么比事务更优先

如果从 Harness 视角总结，Claude Code 优先解决的是三类更高频、更真实的问题：

1. agent 会不会写到不该写的地方  
   这由工具权限和路径边界控制解决。

2. 主线程和后台子 agent 会不会对同一段会话重复写 memory  
   这由 `hasMemoryWritesSince()` 驱动的来源互斥和 existing-manifest 去重解决。

3. agent 会不会把不该长期保存的内容塞进 memory  
   这由四类 taxonomy 和 `WHAT_NOT_TO_SAVE_SECTION` 约束解决。

相比之下，“`MEMORY.md` 索引更新是否具备强事务语义”反而不是这套系统首先关心的问题。因为在 agent memory 场景下，真正危险的往往不是一次 index 漏更新，而是：

- 写入越界
- 多个 agent 重复写入
- 把 code/architecture/task state 之类不该记的东西写进 memory

所以 Claude Code 的 Harness 更像是在把 memory 当成一个**受约束的 agent 写作系统**来治理，而不是把它当成一个**强一致数据库子系统**来治理。

## 补充：autoDream 的实现机制

`autoDream` 可以理解为 Claude Code 的“后台记忆整合器”。它不是主线程对话的一部分，也不是每轮都跑的 memory 抽取器，而是一个**低频、异步、跨 session 的 consolidation 流程**。从源码上看，它的实现方式更像是：在主线程 turn 结束时做轻量门控检查，一旦满足条件，就启动一个受限权限的 forked subagent，专门去整理 memory。

### 1. 触发入口：挂在 stopHooks 之后的 fire-and-forget 调用

`autoDream` 的触发点并不是独立线程或 cron，而是 query loop 的 stop-hook：

- `handleStopHooks()` 在非 bare 模式且当前不是 subagent 时，会 fire-and-forget 调用 `executeAutoDream(...)`。见 `vendor/Claude-code-open/src/query/stopHooks.ts:133-156`。
- `executeAutoDream()` 本身只是一个薄入口，真正逻辑来自 `initAutoDream()` 里初始化出的 closure-scoped `runner`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:118-125`, `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:315-324`。

这意味着 `autoDream` 的运行模式是：

- 每轮结束后做一次 cheap gate check
- 如果条件不满足，立即退出
- 只有在满足整合条件时，才真正起一个 forked agent 去执行 consolidation

源码注释甚至明确写了：启用时每轮额外成本大约只是一次 GrowthBook cache read 和一次 `stat`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:315-318`。

### 2. 开关体系：先判断当前 session 是否允许 dream

`autoDream` 不是默认无条件运行，而是先通过一层总体 gate：

- 若 `getKairosActive()` 为真，则不走这套 `autoDream` 路径。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:95-100`。
- 若当前是 remote mode，则不运行。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:95-100`。
- 若 auto-memory 未开启，则不运行。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:95-100`。
- 最后还要通过 `isAutoDreamEnabled()`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:95-100`。

而 `isAutoDreamEnabled()` 的优先级是：

- 如果 `settings.json` 中显式设置了 `autoDreamEnabled`，优先采用用户设置
- 否则 fallback 到 GrowthBook 的 `tengu_onyx_plover.enabled`  
  见 `vendor/Claude-code-open/src/services/autoDream/config.ts:8-20`。

这说明 `autoDream` 是一个**用户可显式关闭/覆盖的后台能力**，而不是不可控的内建常驻逻辑。

### 3. 调度机制：典型的“三门系统”

`autoDream` 的核心调度逻辑在 `runAutoDream()` 中，源码开头直接写明了 gate 顺序：

1. Time gate
2. Session gate
3. Lock gate  
   见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:1-12`。

默认阈值为：

- `minHours = 24`
- `minSessions = 5`  
  见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:58-66`。

这些阈值还能从 GrowthBook 的 `tengu_onyx_plover` 动态读取，并逐字段校验类型合法性。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:68-93`。

三门的具体实现如下。

#### 3.1 Time gate

- 通过 `readLastConsolidatedAt()` 读取上次 consolidation 时间。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:130-139`, `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:25-35`。
- 若距离现在不足 `minHours`，直接返回。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:140-141`。

#### 3.2 Scan throttle + Session gate

即便 time gate 已通过，`autoDream` 还会加一层 scan throttle：

- 如果距离上次 session scan 不足 10 分钟，则先跳过，防止时间门通过后每轮都重新扫描 transcripts 目录。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:54-57`, `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:143-151`。

随后进入 session gate：

- 通过 `listSessionsTouchedSince(lastAt)` 找出自上次 consolidation 以来被触碰过的 session。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:153-162`, `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:110-124`。
- 显式排除当前 session，因为当前 session 的 mtime 天然是新的。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:163-166`。
- 若 session 数不足 `minSessions`，直接跳过。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:166-171`。

所以它不是“24 小时一到就 dream”，而是“24 小时之后，且累积了足够多值得整理的会话，才 dream”。

#### 3.3 Lock gate

最后一门是锁。它防止多个进程同时跑 consolidation。

- `tryAcquireConsolidationLock()` 会尝试获取 `.consolidate-lock`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:173-190`, `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:38-84`。
- 如果锁已被 live PID 持有，直接返回 `null`，本轮跳过。见 `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:60-68`。

### 4. 锁文件的双重语义：既是互斥锁，也是 lastConsolidatedAt

`autoDream` 的锁设计很有代表性：`.consolidate-lock` 同时承担两种职责：

- **mtime** 表示 `lastConsolidatedAt`
- **文件内容** 存当前 holder 的 PID  
  见 `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:1-5`, `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:25-35`。

获取锁的过程是：

- 若已有 lock，则读取 mtime 和 PID
- 若 mtime 很新且 PID 仍存活，则视为其他进程正在 consolidation，直接退出
- 否则尝试 reclaim
- 写入当前 PID 后，再读回校验是否自己赢得了 race  
  见 `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:46-84`。

这种设计的好处是：

- 不需要单独维护“上次整合时间”和“当前锁状态”两份状态
- crash 后可以通过 PID 是否仍活着判断是否可 reclaim
- 同一个文件即可支持“读取上次整合时间”和“进程互斥”

### 5. 真正执行 consolidation 的不是主线程，而是 forked agent

门控全部通过后，`autoDream` 不会在当前线程里自己扫描和编辑 memory，而是调用 `runForkedAgent(...)` 启动一个 forked subagent。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:224-233`。

这个 forked agent 有两个关键特征。

#### 5.1 共享 cache-safe 上下文

`autoDream` 调用 `createCacheSafeParams(context)`，把主线程当前的：

- `systemPrompt`
- `userContext`
- `systemContext`
- `toolUseContext`
- `forkContextMessages`  
  作为 cache-safe 参数传给 forked agent。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:224-227`, `vendor/Claude-code-open/src/utils/forkedAgent.ts:46-68`, `vendor/Claude-code-open/src/utils/forkedAgent.ts:131-141`。

这意味着 dream subagent 会尽量复用主线程的 prompt cache，而不是完全从零起一个隔离环境。

#### 5.2 工具权限被严格收窄

`autoDream` 传入的 `canUseTool` 是 `createAutoMemCanUseTool(memoryRoot)`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:224-228`。

因此 dream agent 只能：

- 读 memory / transcripts
- 使用 grep/glob
- 运行只读 shell
- 在 memory 目录内部 edit/write

它不能顺手去修改代码仓库中的普通源码文件，也不能跑写状态的 shell。也就是说，即便是“后台整合器”，Harness 仍然把它关在 memory 沙箱里。

### 6. dream prompt：四阶段流程主要由自然语言协议驱动

`autoDream` 的核心行为并不是由一套硬编码状态机实现，而是主要由 `buildConsolidationPrompt()` 生成的 prompt 驱动。

prompt 明确划分四阶段：

- `Phase 1 — Orient`
- `Phase 2 — Gather recent signal`
- `Phase 3 — Consolidate`
- `Phase 4 — Prune and index`  
  见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:26-64`。

各阶段要求如下：

- `Orient`：先 `ls` memory 目录、读 `MEMORY.md`、浏览已有 topic files，必要时查看 `logs/` 或 `sessions/`。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:26-32`。
- `Gather`：优先看 daily logs，其次看 drifted memories，最后才在 transcripts 里做窄 grep。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:33-42`。
- `Consolidate`：把值得保留的内容写回 memory files，要求把相对时间转成绝对时间，并删除被证伪的旧事实。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:44-52`。
- `Prune`：更新 `MEMORY.md`，要求其保持在 `200` 行和约 `25KB` 以内，每条约 `150` 字符，并移除 stale pointers、解决冲突。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:53-60`。

因此，所谓“autoDream 的四阶段”不是代码里的阶段机，而是**代码负责调度和约束，具体整合策略由 prompt 协议交给 dream agent 执行**。

### 7. 输入材料：memory 目录、daily logs、session transcripts 三种来源

`autoDream` 并不是只基于已有 `MEMORY.md` 做整理，它会同时暴露多个输入源给 dream agent：

- 当前 memory root。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:211-222`。
- project 的 transcript 目录。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:19-23`。
- 若存在，优先读取 `logs/YYYY/MM/YYYY-MM-DD.md`。见 `vendor/Claude-code-open/src/services/autoDream/consolidationPrompt.ts:35-40`。
- 额外上下文中还会附上“自上次 consolidation 以来的 session IDs 列表”。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:216-222`。

这保证了 autoDream 不是“只在索引文件上修修补补”，而是真正具备跨 session 汇总、回看日志、修正 drift 的信息来源。

### 8. DreamTask：让后台 dream 可见，但不参与决策

一旦 autoDream 触发，会注册一个 `DreamTask`，把原本 invisible 的后台 consolidation fork 显示到 UI 里。见 `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:1-4`, `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:200-208`。

`DreamTask` 记录的状态包括：

- 当前 phase：`starting` 或 `updating`
- 正在 review 的 session 数
- 已触碰的文件路径
- 最近的 assistant turns 文本
- `abortController`
- `priorMtime`，供 kill 时回滚锁  
  见 `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:20-41`。

要注意的是，它并不解析 prompt 里的四阶段；phase 只有一个简单近似：

- 初始是 `starting`
- 一旦观察到第一个 `Edit/Write` tool_use，就切成 `updating`  
  见 `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:20-23`, `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:76-103`。

也就是说，四阶段是 dream 的“认知流程”，DreamTask 只是用于 UI 暴露的轻量可视化层。

### 9. 进度回传：通过 `onMessage` 旁路观察 forked agent

`runForkedAgent(...)` 支持 `onMessage` 回调，而 `autoDream` 把它接到了 `makeDreamProgressWatcher(...)` 上。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:224-233`, `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:281-313`。

这个 watcher 会：

- 只处理 assistant message
- 聚合 text blocks
- 折叠 tool uses 为数量
- 若 tool_use 是 `FILE_EDIT` 或 `FILE_WRITE`，就提取 `file_path`
- 再调用 `addDreamTurn(...)` 更新 DreamTask UI  
  见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:286-312`, `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:76-103`。

这使得主线程可以看到 dream 大致在做什么，而不需要把 dream 过程直接暴露成主对话的一部分。

### 10. 成功、失败、中止：都有明确的回滚路径

autoDream 成功时：

- `completeDreamTask(...)` 把 DreamTask 标记为完成。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:235`, `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:106-120`。
- 如果确实有文件被触碰，还会向主 transcript 追加一条 `memory_saved` 风格的系统消息，只是 verb 改为 `Improved`。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:236-248`。

失败时：

- 标记 task failed
- 调用 `rollbackConsolidationLock(priorMtime)`，把锁的 mtime 倒回去，让未来还能再次尝试  
  见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:258-271`, `vendor/Claude-code-open/src/services/autoDream/consolidationLock.ts:91-107`。

用户手动 kill DreamTask 时：

- `abortController.abort()`
- 然后同样回滚 lock mtime  
  见 `vendor/Claude-code-open/src/tasks/DreamTask/DreamTask.ts:136-156`。

这说明 autoDream 不是“失败了就静默放弃”的 best-effort 脚本，而是认真处理了**失败后是否还能重试**这一调度问题。

### 11. 一句话总结 autoDream 的实现机制

如果把整套实现压缩成一句话，可以说：

> `autoDream` 不是一个常驻后台线程，而是一个在每轮 stop-hook 后做轻量门控检查、在满足“时间 + session 数 + 锁”条件时启动的、只能操作 memory 沙箱的 forked consolidation subagent；它通过四阶段 dream prompt 整合 memory、logs 与 transcripts，并用锁文件、DreamTask 和失败回滚来保证整个异步流程可控。

## 补充：KAIROS 模式

KAIROS 模式不是 Claude Code 中某一个单独功能点，而是一种更高阶的运行态。按源码看，它更接近一个“assistant mode 的总开关”：一旦激活，Claude Code 会把当前 session 从普通的 request/response REPL，切换成一个**长期驻留、自主推进、持续在线的 assistant runtime**。

### 1. KAIROS 的底层形态：一个全局 session 运行态位

KAIROS 在 bootstrap state 中的底层表示非常简单：

- `STATE.kairosActive`
- `getKairosActive()`
- `setKairosActive()`  
  见 `vendor/Claude-code-open/src/bootstrap/state.ts:72-79`, `vendor/Claude-code-open/src/bootstrap/state.ts:1085-1091`。

但这个布尔位的意义很重，因为多个子系统都用它切换行为：

- memory prompt 是否改成 assistant-mode daily log
- autoDream 是否停用
- Bash / PowerShell 是否自动后台化
- fast mode 在 SDK 环境里是否例外放开
- UI 是否进入 brief / assistant 视图

因此，`kairosActive` 不是一个简单的 UI flag，而是一个会影响多个模块分支的**session capability latch**。

### 2. 激活路径：通过 assistant mode 进入 KAIROS

KAIROS 的激活逻辑在 `main.tsx` 中完成。

关键流程是：

- 若 `feature('KAIROS')` 打开，并且 `assistantModule.isAssistantMode()` 为真
- 当前进程不是被 leader 派生出的 teammate
- 目录已经通过 trust
- 再通过 `kairosGate.isKairosEnabled()` entitlement 检查
- 成功后把 `brief` 视图打开，并调用 `setKairosActive(true)`
- 同时预初始化一个 in-process assistant team  
  见 `vendor/Claude-code-open/src/main.tsx:1048-1088`。

另外，若使用 `--assistant`，则会先 `markAssistantForced()`，跳过重复 entitlement 检查。见 `vendor/Claude-code-open/src/main.tsx:1050-1057`。

这说明 KAIROS 在产品语义上并不是“普通会话中的一个按钮”，而是 assistant daemon / assistant session 的专用运行模式。

### 3. KAIROS 与 proactive 的关系：不是同义词，但高度复用其运行时

源码中大量地方使用 `feature('PROACTIVE') || feature('KAIROS')` 联合判断，说明 KAIROS 复用了 proactive runtime 的很多机制，例如 tick、pause/resume、context-blocked 管理和 SleepTool。见 `vendor/Claude-code-open/src/utils/systemPrompt.ts:18-26`, `vendor/Claude-code-open/src/tools.ts:25-28`。

在系统提示词层面：

- 当 proactive 打开时，会追加 `# Proactive Mode` 段落，明确要求模型主动推进、响应周期性 `<tick>`、在无事可做时调用 `Sleep`。见 `vendor/Claude-code-open/src/main.tsx:2197-2204`。
- 若当前 session 同时处于 KAIROS assistant mode，还会再追加 assistant-specific system prompt addendum。见 `vendor/Claude-code-open/src/main.tsx:2206-2209`。

因此，更贴近源码的说法是：

- proactive 是一种通用的自主运行底座
- KAIROS 在此基础上叠加 assistant mode 的特定语义

也就是说，KAIROS 不是“等于 proactive”，而是把 proactive 机制吸收进了自己的 assistant runtime。

### 4. KAIROS 对 memory 的最大改造：从实时维护 `MEMORY.md` 切换到 daily log

KAIROS 模式下，memory 机制会发生最明显的变化。

普通 auto-memory 下，模型被提示去：

- 写 topic file
- 更新 `MEMORY.md` 作为索引

但在 KAIROS active 时，`loadMemoryPrompt()` 会优先返回 `buildAssistantDailyLogPrompt()`，并覆盖掉 team memory / 普通 auto-memory 的提示词分支。见 `vendor/Claude-code-open/src/memdir/memdir.ts:419-438`。

这个 assistant-mode prompt 明确规定：

- session 是长生命周期的
- 新记忆应当**append** 到按日期命名的 daily log 文件
- log 是 append-only，不应重写或重组
- nightly `/dream` 再把 logs 蒸馏成 topic files + `MEMORY.md`
- `MEMORY.md` 仍然作为 distilled index 自动注入上下文，但不应在本轮直接维护  
  见 `vendor/Claude-code-open/src/memdir/memdir.ts:318-369`。

而 `getAutoMemDailyLogPath()` 的注释也再次确认：

- assistant mode 下不是实时维护 `MEMORY.md`
- 而是持续写 daily log
- 再由单独的 nightly `/dream` 技能做 distillation  
  见 `vendor/Claude-code-open/src/memdir/paths.ts:237-251`。

这意味着 KAIROS 把 memory 从“在线索引维护模式”切换成了“日志采集 + 低频蒸馏模式”。

### 5. KAIROS 与 autoDream 的关系：assistant mode 会停用普通 autoDream

普通模式下，Claude Code 有 `autoDream` 这套后台 consolidation 机制；但 KAIROS active 时，这条路径会被显式关闭：

- `isGateOpen()` 中直接写了 `if (getKairosActive()) return false`，注释说明 KAIROS mode uses disk-skill dream。见 `vendor/Claude-code-open/src/services/autoDream/autoDream.ts:95-100`。

这说明在设计上：

- 普通模式：使用 `autoDream` 这个 forked subagent 做 consolidation
- KAIROS 模式：改用 assistant-mode 专属的 disk/log-based `/dream` 流程

因此，KAIROS 不是在普通 autoDream 上再叠一层，而是**替换掉了 memory consolidation 的工作流**。

### 6. KAIROS 对工具运行时的改变：主线程必须保持响应性

在 assistant mode 下，主线程的角色更像 coordinator，而不是可以长时间阻塞等待子任务结束的前台执行器。

这一点在 `BashTool` 中体现得很明显：

- 若 `feature('KAIROS')` 打开
- 且 `getKairosActive()` 为真
- 且当前是主线程
- 且用户未显式要求 `run_in_background`

则超过 assistant-mode blocking budget 后，会自动把命令转入后台继续执行。见 `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:973-983`。

源码注释直接写明其目的：

- assistant mode 下主 agent 应保持 responsive
- 应持续协调而不是一直等待 shell 完成  
  见 `vendor/Claude-code-open/src/tools/BashTool/BashTool.tsx:973-976`。

这体现出 KAIROS 的一个核心运行时假设：

- 主线程是持续协调者
- 长任务应该后台化、并发化
- 主线程应继续 orchestrate，而不是同步阻塞

### 7. KAIROS 与 brief mode：默认切到 assistant/brief 交互视图

KAIROS 激活后，`main.tsx` 会直接把 `brief` 视图打开。见 `vendor/Claude-code-open/src/main.tsx:1077-1081`。

这说明 assistant mode 的默认用户界面不是普通 transcript-heavy 模式，而是更偏“状态简报 / checkpoint / assistant overlay”风格的视图。

从源码里也能看到，多个组件都用：

- `feature('KAIROS') || feature('KAIROS_BRIEF')`
- `getKairosActive()`  

来切换 spinner、prompt input、message layout 等 UI 表现。相关引用可见：

- `vendor/Claude-code-open/src/components/messages/UserPromptMessage.tsx`
- `vendor/Claude-code-open/src/components/Spinner.tsx`
- `vendor/Claude-code-open/src/components/PromptInput/PromptInput.tsx`

因此，KAIROS 既是运行时模式切换，也是交互视图模式切换。

### 8. KAIROS 与 channels：assistant mode 可以接入外部消息通道

bootstrap state 中专门维护了：

- `allowedChannels`
- `hasDevChannels`  
  见 `vendor/Claude-code-open/src/bootstrap/state.ts:208-217`。

而 `main.tsx` 在 `feature('KAIROS') || feature('KAIROS_CHANNELS')` 下，会解析：

- `--channels`
- `--dangerously-load-development-channels`  

并把结果写入 state。源码搜索结果显示相关逻辑集中在 `vendor/Claude-code-open/src/main.tsx:1642-1717` 一段。

同时，channels 自身还受两层 gate 控制：

- 总开关 `isChannelsEnabled()` 读取 `tengu_harbor`
- allowlist `getChannelAllowlist()` 读取 `tengu_harbor_ledger`  
  见 `vendor/Claude-code-open/src/services/mcp/channelAllowlist.ts:37-53`。

这说明 KAIROS 不是单纯的本地自主模式，它还被设计为一个**可接入外部消息 / 通知源的 assistant runtime**。

### 9. KAIROS 与 scheduled tasks / cron：更适合长期驻留型任务

在 cron / scheduled task 体系中，assistant mode 也是一个显式分支。

`cronScheduler` 的注释明确写到：

- 当 `assistantMode` 为真时，scheduler 可以绕过部分 `isLoading` gate
- assistant mode 下安装时就可能存在 scheduled tasks
- 不应等待普通 loader 再手动启用  
  见 `vendor/Claude-code-open/src/utils/cronScheduler.ts:67-75`。

而 `useScheduledTasks()` 中同样保留了 `assistantMode?: boolean` 参数，并说明：

- assistant mode 不再依赖普通用户输入驱动
- system-generated prompts / scheduled tasks 会在 turn 之间排队执行  
  见 `vendor/Claude-code-open/src/hooks/useScheduledTasks.ts:18-28`, `vendor/Claude-code-open/src/hooks/useScheduledTasks.ts:40-120`。

这说明 KAIROS 所假设的 session 生命周期更长，适合处理：

- periodic ticks
- scheduled tasks
- later-priority system prompts

也就是更接近“在线值守助手”。

### 10. KAIROS 与 fast mode / SDK：assistant daemon 拥有例外待遇

`fastMode.ts` 中有一条非常能说明 KAIROS 身份的逻辑：

- Agent SDK 的非交互 session 默认不开放 fast mode
- 但 assistant daemon mode 是例外
- 原因是 `kairosActive` 会在这个检查运行前被置位  
  见 `vendor/Claude-code-open/src/utils/fastMode.ts:96-109`。

这说明在系统设计者看来，KAIROS 并不被视为普通第三方 SDK client，而是更接近 first-party orchestration 体系的一部分，因此运行时能力策略会更宽。

### 11. 一句话总结 KAIROS 模式

如果把 KAIROS 的源码实现压缩成一句话，可以说：

> KAIROS 模式是 Claude Code 的 assistant runtime：它把当前 session 从普通 REPL 切换成一个长期驻留、带 proactive tick、append-only daily memory log、自动后台化长任务、支持 channels 与 scheduled tasks 的 assistant mode。

它不是一个单点功能，而是一种全局运行态；一旦 `setKairosActive(true)`，多个原本独立的子系统都会切换到更适合“长期自主助手”的执行分支。

## 补充：上下文压缩的实现机制

Claude Code 的上下文压缩并不是单一算法，而是一条分层流水线。按源码看，至少并行存在四种机制：

- `snip`：裁掉历史中不再值得继续保留的片段
- `microcompact`：优先清空旧工具结果
- `context collapse`：把部分历史投影为折叠后的摘要视图
- `autocompact` / `reactive compact`：在接近窗口上限或 API 已报 too long 时做正式压缩

主查询循环中的执行顺序是：

1. `snip`
2. `microcompact`
3. `context collapse`
4. `autocompact`  
   见 `vendor/Claude-code-open/src/query.ts:396-467`。

这意味着 Claude Code 并不是“等到上下文爆掉后再一次性总结”，而是先尝试廉价、小粒度、损伤更小的压缩，再进入昂贵的正式 compact。

### 1. 压缩入口：发生在每轮 query 的前半段

在 `query.ts` 中，模型真正发请求前，会先对 `messagesForQuery` 做一轮上下文治理：

- 先 `snipCompactIfNeeded(messagesForQuery)`
- 再 `deps.microcompact(...)`
- 再 `contextCollapse.applyCollapsesIfNeeded(...)`
- 最后 `deps.autocompact(...)`  
  见 `vendor/Claude-code-open/src/query.ts:396-467`。

因此，上下文压缩不是后台日志整理，而是每次 API 调用前的**prompt shaping pipeline**。

### 2. `snip`：第一层、最轻量的历史裁剪

`snip` 位于流水线的第一层，因为它最轻，并且不会像正式 compact 那样把长历史重写成摘要。

在 query loop 中：

- `snipCompactIfNeeded()` 返回新的 `messages`
- 同时返回 `tokensFreed`
- 若需要还会返回一个 `boundaryMessage`  
  见 `vendor/Claude-code-open/src/query.ts:400-409`。

源码注释说明了它为什么先于 autocompact 执行：

- `snipTokensFreed` 会传给后续 autocompact
- 因为一些 surviving message 的 usage 仍然反映的是 pre-snip 上下文
- 单靠 `tokenCountWithEstimation()` 看不到 snip 真正节省的 token  
  见 `vendor/Claude-code-open/src/query.ts:396-399`。

所以 `snip` 的角色并不是总结，而是：

- 先删明显低价值或可裁切的历史片段
- 再把“删掉了多少”反馈给后面的 autocompact 决策

它更像一个前置瘦身器。

### 3. `microcompact`：优先清空旧工具结果

`microcompact` 的设计目标非常明确：它主要处理旧的 tool results，而不是总结整段对话。

在 `microCompact.ts` 中，可被 compact 的工具集合包括：

- `FILE_READ`
- shell tools
- `GREP`
- `GLOB`
- `WEB_SEARCH`
- `WEB_FETCH`
- `FILE_EDIT`
- `FILE_WRITE`  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:40-50`。

这说明它压缩的主要对象是：

- token 量大
- 但随着时间推移语义价值迅速下降
- 又经常来自工具输出的内容

主入口 `microcompactMessages(...)` 的总体逻辑是：

- 先尝试 time-based microcompact
- 再尝试 cached microcompact
- 若条件不满足则 no-op 返回原 messages  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:253-293`。

#### 3.1 time-based microcompact

源码注释说明：

- 若距离上次 assistant message 已经过久，说明 prompt cache 很可能已经冷掉
- 这时旧 tool result 即使保留，发请求时也会整体重写
- 不如先把这些低价值大块内容清掉  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:261-269`。

因此，time-based microcompact 是基于“缓存已冷”的判断，主动回收旧工具输出。

#### 3.2 cached microcompact

若环境支持 cache editing，`microcompact` 不会直接修改本地 message 内容，而是：

- 跟踪可删除的 tool result
- 生成 `cache_edits`
- 在 API 层应用这些 edits  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:295-304`。

这条路径的设计目标是：**删除工具结果，同时尽量不破坏 prompt cache 命中**。

#### 3.3 `microcompact_boundary`

发生 microcompact 后，系统会生成 `microcompact_boundary` 消息：

- `subtype: 'microcompact_boundary'`
- 记录 `preTokens`
- 记录 `tokensSaved`
- 记录被 compact 的 tool ids  
  见 `vendor/Claude-code-open/src/utils/messages.ts:4557-4583`。

这说明系统不会默默删内容，而是显式保留一条结构化边界消息来标记“这一段上下文已做过 microcompact”。

### 4. `context collapse`：把完整历史投影成折叠视图

`context collapse` 与 `microcompact`、`autocompact` 的机制不同。它并不是直接在当前 message array 中插入一个 compact summary 再替换整段前缀。

源码注释明确写到：

- collapse 是 read-time projection
- summary messages 不直接存在于 REPL array 中
- 它们存在于 collapse store 中
- 每轮进入时，`projectView()` 会重放 commit log  
  见 `vendor/Claude-code-open/src/query.ts:428-447`。

它的设计目标是：

- 尽量保留 REPL 的完整历史
- 但给模型呈现一个折叠后的上下文视图
- 如果 collapse 已经把 token 压到 autocompact 阈值以下，就不必再做正式 summary compact  
  见 `vendor/Claude-code-open/src/query.ts:428-431`。

因此，`context collapse` 试图保住更多粒度信息，避免过早把历史完全拍平成一个摘要块。

### 5. `autocompact`：正式的会话总结压缩

`autocompact` 是上下文压缩链中的重型机制。它不是简单删除，而是真正触发正式的 compact 流程。

主入口是 `autoCompactIfNeeded(...)`。见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:241-350`。

#### 5.1 触发阈值

它首先会检查：

- 是否 `DISABLE_COMPACT`
- 是否已经触发 circuit breaker
- 是否 `shouldAutoCompact(...)`  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:253-277`。

而 `shouldAutoCompact()` 的核心阈值来自：

- `effectiveContextWindow = contextWindow - reservedSummaryOutput`
- `autocompactThreshold = effectiveContextWindow - 13_000`  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:28-49`, `vendor/Claude-code-open/src/services/compact/autoCompact.ts:62-91`。

这里的 `13,000` 就是 autocompact buffer。

#### 5.2 先尝试 session memory compaction

在正式 `compactConversation(...)` 之前，autocompact 会先尝试更轻的一条路径：

- `trySessionMemoryCompaction(...)`  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:287-309`。

如果 session memory compaction 成功：

- 会直接返回 compacted 结果
- 运行 `runPostCompactCleanup(querySource)`
- 标记 post-compaction 状态  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:293-309`。

这说明 autocompact 本身也不是单一路径，而是先试一个更便宜的 session-memory 方案，再 fallback 到正式 compact。

#### 5.3 正式 `compactConversation(...)`

若 session-memory 方案不可用，就会调用：

- `compactConversation(messages, toolUseContext, cacheSafeParams, ...)`  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:312-333`。

成功后会：

- reset `lastSummarizedMessageId`
- 调 `runPostCompactCleanup(querySource)`
- 返回 `CompactionResult`  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:323-333`。

随后在 `query.ts` 中：

- 记录 compact 的 token 与 usage 指标
- 重置 tracking（`turnId` / `turnCounter` / `consecutiveFailures`）
- 调用 `buildPostCompactMessages(compactionResult)`
- 将生成的 post-compact messages yield 到上层
- 并用它们替换 `messagesForQuery`  
  见 `vendor/Claude-code-open/src/query.ts:470-543`。

因此，正式 compact 的本质是：

- 把旧上下文前缀总结掉
- 生成一套新的 post-compact message 集合
- 后续 query 基于这套新上下文继续运行

### 6. `compact_boundary`：压缩后的结构化锚点

正式 compact 完成后，系统会生成一条 `compact_boundary` 系统消息：

- `subtype: 'compact_boundary'`
- `content: 'Conversation compacted'`
- 附带 `compactMetadata`，包括 `trigger`、`preTokens`、`messagesSummarized` 等  
  见 `vendor/Claude-code-open/src/utils/messages.ts:4530-4555`。

它不是单纯给用户看的提示，而是后续多个逻辑的结构化锚点：

- `findLastCompactBoundaryIndex()` 会从后向前查找最近一次 boundary。见 `vendor/Claude-code-open/src/utils/messages.ts:4614-4629`。
- `getMessagesAfterCompactBoundary()` 会把消息裁到最后一次 compact 之后，并在需要时叠加 snip 视图。见 `vendor/Claude-code-open/src/utils/messages.ts:4631-4655`。

因此，`compact_boundary` 是整个“压缩后如何恢复、如何截取有效上下文”的关键锚点，而不仅仅是一个 UI 文案。

### 7. `reactive compact`：API 已报 too long 时的兜底压缩

除 proactive autocompact 外，`query.ts` 还保留了 reactive 路径：

- 如果流式返回中出现 withheld prompt-too-long 或 media-size 错误
- 且 `reactiveCompact` 可用
- 就调用 `tryReactiveCompact(...)`
- 成功后同样构造 `postCompactMessages` 并 retry  
  见 `vendor/Claude-code-open/src/query.ts:1119-1148`。

这条路径的意义在于：

- 即便 proactive 阶段未及时 compact
- API 已明确告诉你“上下文太长”
- 系统仍保留最后一层自动恢复能力

因此，Claude Code 的压缩不是只有阈值触发，还包括**错误驱动的兜底压缩**。

### 8. circuit breaker：防止压缩死循环

`autocompact` 有显式的失败计数与熔断机制：

- 最多允许 `3` 次连续失败
- 超过后，本 session 后续会跳过未来自动压缩尝试  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:257-265`, `vendor/Claude-code-open/src/services/compact/autoCompact.ts:338-349`。

源码注释给出的动机也很直接：

- 某些 irrecoverably over-limit 的 session
- 若不熔断，会在每轮都不断重试 compact
- 白白消耗大量 API 调用  
  见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:257-263`。

这说明 Claude Code 不仅治理上下文本身，也治理“压缩机制失控”这一二阶问题。

### 9. 一句话总结上下文压缩的实现机制

如果把整条链路压缩成一句话，可以说：

> Claude Code 的上下文压缩不是一次性的“总结生成”，而是一条分层、渐进、带结构化边界消息的 prompt shaping 流水线：先删低价值工具输出，再折叠部分历史，再在必要时执行正式 compact，并在 API 明确报 too long 时保留 reactive 兜底。

它背后的核心设计取舍是：

- 能不总结就不总结
- 能先清工具垃圾就先清工具垃圾
- 能保留粒度化历史就尽量保留
- 真逼近窗口上限时，再进入正式 compact
- compact 后必须留下结构化 boundary，供后续截取、恢复和继续运行使用

## 补充：cached microcompact 如何做到“删除工具结果同时尽量不破坏 prompt cache”

`cached microcompact` 的核心思路是：**不直接修改本地消息数组中的旧 `tool_result` 内容**，而是把“删除哪些工具结果”的意图编码为 API 层的控制块，并配合 `cache_reference` / `cache_edits` 机制在服务端缓存前缀内部完成删除。这样既能减少后续上下文中的有效 token，又尽量不打碎已建立的 prompt cache 前缀。

源码对此写得非常直接：

- cached MC 的关键差异是“**Does NOT modify local message content**”
- `cache_reference` 和 `cache_edits` 都是在 **API layer** 注入的  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:296-303`。

### 1. 为什么不能直接在本地把旧工具结果删掉

如果像 time-based microcompact 那样，直接把旧 `tool_result.content` 改成占位文本，那么传给模型的 `messages` 前缀就变了。

而 Anthropic prompt cache 的 cache-safe 参数本身就依赖于：

- system prompt
- tools
- model
- messages prefix
- thinking config  
  见 `vendor/Claude-code-open/src/utils/forkedAgent.ts:46-68`。

因此，一旦直接修改历史消息内容：

- 前缀不再与之前的请求一致
- 已有 prompt cache 可能整体失效
- 即使省下了部分 token，也可能被 cache miss 抵消

所以 cached microcompact 要解决的核心矛盾是：

- 想逻辑上“删掉”旧工具结果
- 但又不想物理上重写整段消息前缀

### 2. cached microcompact 的基本策略：messages 原样返回，只携带“待删除工具集合”

在 `cachedMicrocompactPath(...)` 中，逻辑大致是：

1. 收集所有可 compact 的 `tool_use_id`
2. 在 user message 中登记对应的 `tool_result`
3. 通过 cached-MC 状态机决定哪些旧 tool result 该删
4. 生成 `cache_edits` block
5. 返回时 **messages 保持不变**
6. 只把 `pendingCacheEdits` 附在结果中，留给 API 层处理  
   见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:305-399`。

最关键的行为就在这里：

- `Return messages unchanged - cache_reference and cache_edits are added at API layer`  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:369-394`。

这保证了：

- REPL 里的本地消息历史不被直接改写
- cache key 相关的共享前缀尽量保持稳定
- 删除动作被推迟到真正的 API 请求层表达

### 3. API 层如何表达删除：`cache_reference` + `cache_edits`

真正把“删除工具结果”编码进请求结构的，是 `addCacheBreakpoints(...)`。见 `vendor/Claude-code-open/src/services/api/claude.ts:3062-3210`。

#### 3.1 `cache_edits` 结构

API 层为 cached MC 定义的删除块结构是：

- `type: 'cache_edits'`
- `edits: [{ type: 'delete', cache_reference: string }]`  
  见 `vendor/Claude-code-open/src/services/api/claude.ts:3052-3060`。

这说明 cached MC 的删除不是“把某个本地 block 内容改成空”，而是：

- 针对某个 `cache_reference`
- 发出一个 delete 指令

也就是说，删除动作是**面向缓存引用**的，而不是面向本地 message 重写的。

#### 3.2 `cache_reference` 的注入

随后，API 层会给符合条件的旧 `tool_result` block 加上：

- `cache_reference: block.tool_use_id`  
  见 `vendor/Claude-code-open/src/services/api/claude.ts:3164-3207`。

这里的约束也很细：

- 只给**最后一个 `cache_control` marker 之前**的 `tool_result` 注入 `cache_reference`
- 并且采用 strict “before” 而不是 “before or on”
- 以避免 `cache_edits` 插入后引起边界错位  
  见 `vendor/Claude-code-open/src/services/api/claude.ts:3180-3186`。

这意味着 cached microcompact 能工作，依赖的是这一对机制：

- 先给历史工具结果稳定命名（`cache_reference`）
- 再在新请求中发出 `cache_edits(delete cache_reference=...)`

服务端据此知道：这些工具结果虽然原本存在于缓存前缀中，但这次请求里逻辑上可以删除。

### 4. 为什么说它是“尽量不破坏”而不是“完全不影响” prompt cache

cached MC 的目标不是完全不改变请求语义，而是**避免通过直接改写历史正文去破坏共享前缀**。

源码注释用词是：

- `uses cache editing API to remove tool results without invalidating the cached prefix`  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:296-303`。

这句话更准确的理解是：

- 尽量保留已建立的 shared prefix
- 不要因为改写 `tool_result.content` 而让整段历史 cache miss
- 把变化压缩在 `cache_reference` / `cache_edits` 这种 API 控制结构里

因此，它实现的是“最大化保住前缀复用”，而不是“什么都不变”。

### 5. `pinCacheEdits`：让删除语义在后续请求中保持稳定位置

仅仅这次请求发送 `cache_edits` 还不够。若下一次请求把同样的删除块插到不同位置，前缀结构仍然会漂移，从而影响 cache 稳定性。

为了解决这个问题，API 层在把新的 `cache_edits` 插入**最后一个 user message**后，会调用：

- `pinCacheEdits(i, newCacheEdits)`  
  见 `vendor/Claude-code-open/src/services/api/claude.ts:3141-3158`。

在之后的请求中，API 层会先把所有之前 pin 住的 `cache_edits` **重新插回原始位置**：

- `Re-insert all previously-pinned cache_edits at their original positions`  
  见 `vendor/Claude-code-open/src/services/api/claude.ts:3127-3139`。

这一步的作用是：

- 让“删除哪些旧工具结果”也成为前缀结构的一部分
- 并且在每次请求中都稳定复现
- 避免因为 `cache_edits` 漂移而造成新的 cache miss

因此，`pinCacheEdits` 本质上是在维护“删除语义的位置稳定性”。

### 6. 去重：避免重复删除同一个 `cache_reference`

API 层还专门维护了：

- `seenDeleteRefs`
- `deduplicateEdits(...)`  
  见 `vendor/Claude-code-open/src/services/api/claude.ts:3112-3125`。

它的目的不是简单代码整洁，而是进一步保证：

- 同一个 `cache_reference` 不会在多个 `cache_edits` block 里重复删除
- 避免请求结构中出现冗余、漂移或重复控制语义

这也是“尽量不破坏 prompt cache”的一部分，因为越稳定、越去重的请求结构越利于前缀复用。

### 7. 为什么 boundary message 要等 API 返回后才发

cached microcompact 不像 time-based microcompact 那样直接在本地改写内容，因此客户端在发请求前其实并不知道“这次真正删除了多少 token”。

于是系统会：

- 在发请求前记录 `baselineCacheDeletedTokens`
- 等 API 返回后，从 assistant usage 中取出累计的 `cache_deleted_input_tokens`
- 再减去 baseline，得到这一次 cached MC 真正删掉的 token 数
- 最后才生成 `microcompact_boundary`  
  见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:369-394`, `vendor/Claude-code-open/src/query.ts:866-892`。

这说明 cached MC 的删除效果并不是客户端本地估出来的，而是以服务端真实返回的 usage 字段为准。

### 8. 一句话总结 cached microcompact 的实现机制

如果把这套机制压缩成一句话，可以说：

> cached microcompact 不是“真的把历史工具结果从本地消息里删掉”，而是保持本地消息前缀基本不变，只把“这些旧工具结果现在逻辑上可忽略”的意图编码成 `cache_reference + cache_edits`，再借助 API 层的固定位置重放与服务端 cache editing，在尽量保住 prompt cache 的前提下实现逻辑删除。

## 补充：`cache_reference` 和 `cache_edits` 的结构定义

从源码看，`cache_edits` 有明确的结构定义，而 `cache_reference` 并不是一个独立的顶层 block type，而是 API 组包阶段动态补到 `tool_result` block 上的字段。

### 1. `cache_edits` 的结构定义

`cache_edits` 定义在 `vendor/Claude-code-open/src/services/api/claude.ts:3052-3055`：

```ts
type CachedMCEditsBlock = {
  type: 'cache_edits'
  edits: { type: 'delete'; cache_reference: string }[]
}
```

也就是说，它本身就是一个独立 block，实际形态可以写成：

```ts
{
  type: 'cache_edits',
  edits: [
    { type: 'delete', cache_reference: 'toolu_...' }
  ]
}
```

这里的语义非常直接：

- `type: 'cache_edits'` 表示这是一个缓存编辑控制块
- `edits` 是一组编辑指令
- 当前源码里看到的编辑类型是 `delete`
- `cache_reference` 是要删除的缓存对象标识

### 2. pinned `cache_edits` 的结构定义

源码紧接着还定义了 pinned 版本，见 `vendor/Claude-code-open/src/services/api/claude.ts:3057-3060`：

```ts
type CachedMCPinnedEdits = {
  userMessageIndex: number
  block: CachedMCEditsBlock
}
```

这说明系统内部并不只是临时生成一个 `cache_edits` block，还会把它和某个 `user message` 的索引绑定起来。它的作用是：

- 记录这个 `cache_edits` block 应该插在哪条 `user message` 后面
- 确保后续请求继续把同一个 block 重插回同一位置
- 保持请求前缀结构稳定，避免因为控制块位置漂移而影响 prompt cache

### 3. `cache_reference` 的实际结构

`cache_reference` 本身在这一段源码里没有看到单独的 type alias；它的实现方式是：在 API 层遍历 `tool_result` block 时，动态补上一个字符串字段。实现位置见 `vendor/Claude-code-open/src/services/api/claude.ts:3164-3203`，核心赋值是：

```ts
{
  ...block,
  cache_reference: block.tool_use_id,
}
```

所以从实际数据结构上看，`cache_reference` 的形态是：

```ts
{
  type: 'tool_result',
  tool_use_id: 'toolu_...',
  content: ...,
  cache_reference: 'toolu_...'
}
```

也就是说：

- `cache_reference` 挂在 `tool_result` block 上
- 类型是 `string`
- 值直接取对应的 `tool_use_id`

因此，它不是“另外生成的一份对象”，而是对已有 `tool_result` 的扩展标记。

### 4. 两者的配合关系

`cache_reference` 和 `cache_edits` 的关系可以压缩成一句话：

- `tool_result.cache_reference` 负责声明“这个可缓存对象叫什么”
- `cache_edits.edits[].cache_reference` 用这个名字去引用并删除它

也就是：

```ts
// 被标记的旧 tool_result
{
  type: 'tool_result',
  tool_use_id: 'toolu_123',
  cache_reference: 'toolu_123',
  content: ...
}

// 删除它的控制块
{
  type: 'cache_edits',
  edits: [
    { type: 'delete', cache_reference: 'toolu_123' }
  ]
}
```

这一设计说明 `cache_edits` 并不是“重写内容”，而是“按引用删除”。

### 5. 插入位置上的实现细节

`cache_edits` block 不是随便插在消息数组任意位置。API 层会结合 `insertBlockAfterToolResults(...)` 来把它稳定插入到 user message content array 中合适的位置，见 `vendor/Claude-code-open/src/utils/contentArray.ts:1-21` 与 `vendor/Claude-code-open/src/services/api/claude.ts:3127-3143`。

它的设计目标是：

- 让 `cache_edits` 出现在可被服务端识别的位置
- 让同一条删除语义在后续请求中反复出现在相同结构位置
- 尽量减少对 prompt cache 前缀形状的扰动

### 6. 一句话总结

如果只用一句话概括结构定义：

> `cache_edits` 是一个显式的控制 block，结构为 `{ type: 'cache_edits', edits: [{ type: 'delete', cache_reference: string }] }`；`cache_reference` 则是附着在旧 `tool_result` 上的字符串标识，实际值直接复用该结果的 `tool_use_id`。
