# Claude-code-open MCP 架构深度解读

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 的 “MCP 架构”。

这里的 MCP 指 [Model Context Protocol] 的客户端实现。对初学者来说，最容易困惑的不是 “MCP 是什么”，而是下面这些更具体的问题：

1. Claude Code 里的 MCP 只是“远程工具调用”吗？
2. MCP 服务器配置从哪里来，优先级怎么定？
3. 一个 MCP server 是什么时候被连接、发现工具、注册到系统里的？
4. 模型输出 `mcp__xxx__yyy` 这样的工具名后，系统如何找到真正的 MCP server 并调用？
5. 为什么代码里既有 `config.ts`，又有 `client.ts`，还有 `useManageMCPConnections.ts`？
6. 为什么 MCP 结果还要做截断、文件持久化、认证缓存、自动重连？

本文基于以下主干代码阅读整理：

- `src/services/mcp/types.ts`
- `src/services/mcp/config.ts`
- `src/services/mcp/client.ts`
- `src/services/mcp/useManageMCPConnections.ts`
- `src/services/mcp/utils.ts`
- `src/services/mcp/mcpStringUtils.ts`
- `src/services/mcp/normalization.ts`
- `src/services/tools/toolExecution.ts`
- `src/commands/mcp/addCommand.ts`

---

## 2. 先给结论

Claude Code 的 MCP 架构，本质上不是“加几个外部工具”这么简单，而是一个完整的 **外部能力接入子系统**。

它可以拆成 4 层：

```text
配置层
  config.ts / types.ts / addCommand.ts

连接与协议层
  client.ts

运行期管理层
  useManageMCPConnections.ts / MCPConnectionManager.tsx

工具集成层
  client.ts 中的 fetchTools/fetchCommands/fetchResources
  toolExecution.ts 中的调用与错误处理
```

每层职责不同：

- 配置层：决定“有哪些 MCP 服务器可以被系统看见”。
- 连接与协议层：决定“如何连上这些服务器并说 MCP 协议”。
- 运行期管理层：决定“如何把连接状态、工具、命令、资源同步到 AppState，并处理重连、启停、刷新”。
- 工具集成层：决定“模型看到的工具名是什么、如何调用、结果如何回流给模型”。

如果只记一句话：

**`config.ts` 管“有哪些 server”，`client.ts` 管“怎么连、怎么调”，`useManageMCPConnections.ts` 管“运行时怎么维护”，`toolExecution.ts` 管“工具执行失败后如何反馈到系统状态”。**

---

## 3. MCP 在整个系统里的位置

参考 `0001-project-arch.md`，Claude Code 的能力来源大致有 4 类：

1. 内置工具
2. Slash commands
3. MCP servers
4. Plugins / Skills / Agents

MCP 的特殊性在于：

- 它不是纯本地模块，而是“外部能力容器”。
- 它既可能是本地进程（`stdio`），也可能是远程服务（`sse` / `http` / `ws`）。
- 它不只提供工具，还可能提供：
  - prompts
  - resources
  - skill 资源
  - list_changed 通知
  - OAuth 认证流

所以 MCP 更像一个“可插拔能力总线”，而不是单一工具。

---

## 4. 核心模块分工

## 4.1 `types.ts`：统一类型中心

`src/services/mcp/types.ts` 定义了这套架构最基础的概念：

- 配置作用域 `ConfigScope`
- 传输类型 `Transport`
- 各类 server config schema
- 连接状态 `MCPServerConnection`
- CLI / UI 序列化结构

最重要的两个类型是：

### A. `ScopedMcpServerConfig`

```ts
type ScopedMcpServerConfig = McpServerConfig & {
  scope: ConfigScope
  pluginSource?: string
}
```

它的意义是：

- `McpServerConfig` 只描述“如何连接”
- `ScopedMcpServerConfig` 还描述“它从哪里来”

这个 `scope` 很关键，因为它决定了：

- 配置优先级
- UI 展示文案
- 权限 / 策略判断
- 工具调用时的来源识别

### B. `MCPServerConnection`

```ts
type MCPServerConnection =
  | ConnectedMCPServer
  | FailedMCPServer
  | NeedsAuthMCPServer
  | PendingMCPServer
  | DisabledMCPServer
```

这说明 Claude Code 不把 MCP server 简化成“连上 / 没连上”两种状态，而是显式建模成状态机：

- `pending`：等待连接或重连中
- `connected`：已连接，可发现工具/命令/资源
- `needs-auth`：连接或调用时发现缺认证
- `failed`：连接失败
- `disabled`：逻辑上禁用，不应发起连接

这也是为什么 `/mcp` 菜单能准确展示运行状态。

---

## 4.2 `config.ts`：配置汇总中心

`src/services/mcp/config.ts` 负责回答一个核心问题：

**系统最终应该连接哪些 MCP servers？**

它做的事情很多，但可以归为 5 类：

1. 读取各来源配置
2. 校验 schema
3. 扩展环境变量
4. 做策略过滤与审批过滤
5. 做跨来源去重与合并

它是 MCP 架构里的“配置编排器”。

---

## 4.3 `client.ts`：协议与执行中心

`src/services/mcp/client.ts` 是 MCP 子系统最核心的文件。

它负责：

- 建立不同 transport 的连接
- 缓存连接
- 拉取 tools / prompts / resources
- 把 MCP tools 映射为 Claude Code 内部 `Tool`
- 执行 MCP tool 调用
- 处理 URL elicitation、401、session expired、large output

如果说 `config.ts` 决定“接谁进来”，那 `client.ts` 决定“接进来之后怎么工作”。

---

## 4.4 `useManageMCPConnections.ts`：运行期状态管理中心

这是 React / AppState 侧的核心。

它负责：

- 启动时把所有 server 放进 `pending`
- 分两阶段连接 Claude Code MCP 和 claude.ai connectors
- 把连接结果批量刷入 `AppState.mcp`
- 注册 `onclose`、`list_changed`、elicitation 等通知处理器
- 处理自动重连、手工重连、启用/禁用

所以它不是协议层，而是 **“MCP 运行期调度器”**。

---

## 4.5 `mcpStringUtils.ts` + `normalization.ts`：命名约定层

MCP 工具最终会变成：

```text
mcp__<normalized_server_name>__<normalized_tool_name>
```

例如：

```text
mcp__github__create_issue
```

这样设计的目的有 3 个：

1. 避免和内置工具重名
2. 让权限规则能精确匹配到“某个 server 的某个 tool”
3. 让模型看到统一可预测的命名空间

这是一个很重要的工程化处理：**把外部协议对象转成内部稳定命名。**

---

## 5. 配置架构：MCP server 从哪里来

## 5.1 支持的配置来源

从代码看，MCP server 可以来自这些来源：

- `user`
- `project`
- `local`
- `dynamic`
- `enterprise`
- `claudeai`
- plugin 派生 server（最终也带 scope 元信息）

其中：

- `project` 对应项目内 `.mcp.json`
- `user` / `local` 对应 Claude 配置文件
- `dynamic` 常见于命令行或运行时注入
- `enterprise` 是托管配置，优先级最高，而且可能具有“独占控制权”
- `claudeai` 是从 claude.ai connector 拉下来的远程配置

---

## 5.2 配置读取流程

可以把 `config.ts` 的读取逻辑理解成下面这个流程：

```text
读取 enterprise
  如果 enterprise MCP 存在 -> 它可独占所有 MCP 配置

否则读取：
  plugin servers
  user servers
  project servers
  local servers
  dynamic servers
  claude.ai servers（在 getAllMcpConfigs 中单独异步并入）
```

关键函数：

- `getProjectMcpConfigsFromCwd()`
- `getMcpConfigsByScope()`
- `getClaudeCodeMcpConfigs()`
- `getAllMcpConfigs()`

其中：

- `getClaudeCodeMcpConfigs()` 只处理 Claude Code 本地侧配置
- `getAllMcpConfigs()` 再把 claude.ai connectors 合并进来

这是一个很好的分层：

- 启动关键路径走“快路径”
- 网络相关的 claude.ai 配置异步叠加

---

## 5.3 配置优先级

从 `getClaudeCodeMcpConfigs()` 的合并顺序可以看出，优先级大致是：

```text
plugin < user < project < local
```

而 `enterprise` 存在时，可以直接排斥其它来源。

这反映出一个很明确的产品决策：

- 越靠近当前用户、当前项目上下文的配置，优先级越高
- 企业托管配置在某些场景下拥有最终控制权

---

## 5.4 项目级 `.mcp.json` 还有审批层

`getProjectMcpServerStatus()` 说明：项目内的 `.mcp.json` 不是天然可信。

它有三种状态：

- `approved`
- `rejected`
- `pending`

只有 `approved` 的 project server 才会进入最终连接集合。

这背后的设计理由很重要：

- `.mcp.json` 是项目仓库文件
- 仓库内容可能来自第三方
- 本地执行 MCP stdio server 可能意味着直接拉起命令进程

因此项目级 MCP 本质上是一个需要审批的执行入口。

---

## 5.5 配置校验与环境变量展开

`parseMcpConfig()` / `parseMcpConfigFromFilePath()` 除了做 Zod schema 校验，还做了两类很实用的增强：

1. 环境变量展开
2. 非致命 warning 收集

例如：

- 缺失环境变量不会直接把整个系统打崩，而是记录 warning
- Windows 下直接用 `npx` 会给出专门提示

这说明这里的配置处理不是“纯解析器”，而是 **带运维经验的健壮配置层**。

---

## 5.6 去重算法：为什么需要“内容去重”

MCP 配置来自多个来源时，最容易出现的问题是：

- 名字不同，但底层连的是同一个 server
- 手工配置和 plugin 配置重复
- 手工配置和 claude.ai connector 重复

这时如果只按 key 去重，会失败。

所以代码实现了：

- `getMcpServerSignature()`
- `dedupPluginMcpServers()`
- `dedupClaudeAiMcpServers()`

核心思路是：

```text
stdio server -> 用 command + args 形成签名
remote server -> 用 url 形成签名
忽略 env / headers 等“附属差异”
```

这是一个很典型的“内容签名去重”算法。

它解决的是：

- 避免重复连接
- 避免同一批工具被重复暴露给模型
- 避免浪费上下文窗口

注意这里还有一个细节：

- 对 CCR 代理 URL，会先通过 `unwrapCcrProxyUrl()` 解出原始 `mcp_url`

也就是说，代码考虑到了“同一个远程 connector 被代理改写 URL”后仍然应该视为同一个 server。

这是非常工程化的处理。

---

## 6. 连接架构：server 是怎么连上的

## 6.1 `connectToServer()` 是连接总入口

`client.ts` 中的 `connectToServer()` 是单个 server 的统一连接入口。

它根据 `config.type` 选择不同 transport：

- `stdio`
- `sse`
- `http`
- `ws`
- `sse-ide`
- `ws-ide`
- `claudeai-proxy`
- `sdk`

可以把它理解成一个 transport factory。

---

## 6.2 各类 transport 的本质差异

### A. `stdio`

本地起子进程，通过标准输入输出通信。

适合：

- 本地文件系统 server
- 本地工具链 server
- 需要直接使用本机环境的 server

### B. `sse`

远程长连接，服务端通过 SSE 推送消息。

### C. `http`

基于 Streamable HTTP transport，适合更标准的远程 MCP 服务。

### D. `ws`

WebSocket 双向连接。

### E. `sdk`

不是传统网络连接，而是特殊的进程内 / SDK 托管通道。

### F. `claudeai-proxy`

本质还是 HTTP，但通过 Claude.ai proxy 打通认证与代理路径。

---

## 6.3 为什么要区分本地 server 和远程 server

`getMcpToolsCommandsAndResources()` 里有一个很关键的调度策略：

- 本地 server 单独一组，并发更低
- 远程 server 单独一组，并发更高

原因很直接：

- `stdio` / `sdk` 连接通常意味着进程启动，成本更高
- 远程连接主要是网络 IO，可以更高并发

这是一个简单但非常有效的调度优化。

代码里甚至专门把原来的“固定批次串行等待”改成了 `pMap` 式并发调度，避免某一个慢 server 卡住整个后续批次。

---

## 6.4 连接状态不是一次性结果，而是缓存对象

`connectToServer()` 是 memoized 的，配套还有：

- `getServerCacheKey()`
- `clearServerCache()`
- `ensureConnectedClient()`

也就是说，系统不会每次调用工具都重新建连接，而是：

1. 先缓存连接对象
2. 调工具前用 `ensureConnectedClient()` 确认连接可用
3. 会话过期、配置变更、手动重连时再清缓存重建

这让 MCP 从“每次都新建连接的 RPC”变成了“有生命周期的长连接资源”。

---

## 7. 发现架构：工具、命令、资源如何进入系统

## 7.1 连接成功后会做三类发现

一个 server 连上后，系统通常会做：

1. `tools/list`
2. `prompts/list`
3. `resources/list`

对应函数：

- `fetchToolsForClient()`
- `fetchCommandsForClient()`
- `fetchResourcesForClient()`

如果开启了 MCP Skills，还会额外从 resources 中派生 skills。

---

## 7.2 MCP tools 如何转成系统内部 `Tool`

这是 `fetchToolsForClient()` 的核心职责。

它会把 MCP tool 描述映射成 Claude Code 内部 `Tool` 对象，并补齐很多系统语义：

- `name`
- `mcpInfo`
- `description()`
- `prompt()`
- `inputJSONSchema`
- `isReadOnly()`
- `isDestructive()`
- `isOpenWorld()`
- `checkPermissions()`
- `call()`
- `userFacingName()`

这一步很重要，因为它说明：

**MCP tool 并不是被“特殊对待”的旁路逻辑，而是被适配成和内置工具统一的抽象。**

这也是整个架构可扩展的关键。

---

## 7.3 为什么工具名要标准化

MCP server 的原始 tool name 不一定满足 Claude Code 内部约束。

所以系统会做两次标准化：

1. server name 用 `normalizeNameForMCP()`
2. tool name 用 `buildMcpToolName()`

形成：

```text
mcp__<server>__<tool>
```

这样做的直接收益：

- 权限规则可以精确匹配到单个 MCP tool
- UI 和 telemetry 有统一命名
- 不会和 `Read`、`Write` 等内置工具冲突

---

## 7.4 prompts 如何转成 slash command

`fetchCommandsForClient()` 会把 MCP prompt 转成内部 `Command`：

```text
mcp__<server>__<prompt>
```

其 `getPromptForCommand()` 内部再调用：

- `ensureConnectedClient()`
- `client.getPrompt(...)`
- `transformResultContent(...)`

也就是说，对系统来说，MCP prompts 被纳入了命令系统，而不只是“资源描述”。

---

## 7.5 resources 为什么还要补两个内置工具

当某个 server 支持 resources 时，系统可能额外注入：

- `ListMcpResourcesTool`
- `ReadMcpResourceTool`

原因是：

- resources 本身不是模型直接可调用的 tool
- 但系统希望模型能够“列出资源”和“读取资源”

于是 Claude Code 在 MCP 原生 resources 能力上，再包装了一层统一工具接口。

这是一个典型的“协议能力 -> 模型可用工具”的适配思路。

---

## 8. 运行期管理：为什么还要有 `useManageMCPConnections`

如果只有 `connectToServer()`，你只能“连一次”。

但真实运行期还需要处理：

- 启动时先把 server 展示出来
- 背景异步连接
- 连接状态变化回写 UI
- 连接断开后自动重连
- tools/prompts/resources 动态刷新
- 用户在 `/mcp` 中手工启停 / 重连

这些都属于运行时状态管理，不适合放在 `client.ts`。

所以有了 `useManageMCPConnections()`。

---

## 8.1 两阶段加载

启动时它采用“两阶段”策略：

### 第一阶段

先加载 Claude Code 本地侧 MCP 配置并开始连接。

特点：

- 快
- 不依赖远程 claude.ai 请求

### 第二阶段

再等待 claude.ai connectors 拉取完成，去重后补充进状态并连接。

这样做的收益是：

- 启动更快
- 本地 MCP 不必等待网络
- claude.ai connector 可以渐进出现

---

## 8.2 先标记 `pending`，再异步落真实结果

它不会等所有 server 连接完才更新 UI，而是先把 server 状态放成：

- `pending`
- 或 `disabled`

然后连接成功后再更新成：

- `connected`
- `needs-auth`
- `failed`

这使得用户可以明确感知系统“正在连接哪些 server”。

这是一个典型的渐进式状态建模。

---

## 8.3 AppState 更新为什么要批处理

`useManageMCPConnections()` 里有：

- `pendingUpdatesRef`
- `flushPendingUpdates()`
- 16ms 的批量刷新窗口

原因是多个 server 的连接结果可能在很短时间内连续返回。

如果每个都直接 `setAppState`，会导致：

- UI 频繁重渲染
- 状态拼接逻辑分散

所以它采用“小窗口批量提交”的办法，把多个 server 更新合并进一次 state update。

这是典型的前端性能优化。

---

## 8.4 自动重连算法

当远程 transport 的 `onclose` 触发时，系统不会立刻宣告失败，而是进入指数退避重连：

```text
第 1 次：1s
第 2 次：2s
第 3 次：4s
...
直到上限
```

关键常量：

- `MAX_RECONNECT_ATTEMPTS = 5`
- `INITIAL_BACKOFF_MS = 1000`
- `MAX_BACKOFF_MS = 30000`

而且重连过程中状态会变成 `pending`，并带上：

- `reconnectAttempt`
- `maxReconnectAttempts`

这让 UI 有机会显示更细的重连信息。

---

## 8.5 动态刷新：MCP 不只是静态加载

如果 server 声明支持：

- `tools.listChanged`
- `prompts.listChanged`
- `resources.listChanged`

Claude Code 会注册通知处理器，在收到通知时：

1. 清除对应缓存
2. 重新拉取最新内容
3. 回写 AppState

这说明 MCP 在这里是“动态能力目录”，而不是启动时静态快照。

---

## 9. 工具调用主流程

## 9.1 从模型输出到定位 MCP server

模型看到的 MCP tool 名一般长这样：

```text
mcp__github__create_issue
```

当它进入工具执行链后，系统通过：

- `mcpInfoFromString()`

解析出：

- `serverName`
- `toolName`

再在当前 `mcpClients` 里按 normalized server name 找到真实连接对象。

这说明工具名不仅是显示名称，还是路由键。

---

## 9.2 `call()` 最终如何执行

MCP tool 的内部 `call()` 大致是：

```text
ensureConnectedClient()
  -> callMCPToolWithUrlElicitationRetry()
    -> callMCPTool()
      -> client.callTool()
      -> processMCPResult()
```

这里有 4 层含义：

1. 保证连接有效
2. 处理 URL elicitation 这种交互型错误
3. 真正调用 MCP SDK
4. 规范化结果并控制上下文体积

这也是 `client.ts` 复杂的根本原因：它同时处理协议、交互、容错、结果治理。

---

## 9.3 URL elicitation 是什么

有些 MCP server 在调用工具时，不是直接返回结果，而是要求用户先打开某个 URL 完成授权或确认。

代码里通过：

- `callMCPToolWithUrlElicitationRetry()`

处理这个场景。

机制是：

1. 捕获 `UrlElicitationRequired` 对应错误码
2. 解析出 elicitation 数据
3. 先跑 hooks，看能否程序化处理
4. 不行再走 UI 或 structured IO
5. 用户完成后重试原 tool call

这相当于在“工具调用”中间嵌入了一个小型人机交互子流程。

---

## 9.4 为什么要单独处理 401 和 session expired

MCP 与普通本地工具不同，它有长期连接与远程认证问题。

所以 `callMCPTool()` 专门处理：

### A. 401 / Unauthorized

抛出 `McpAuthError`，上层在 `toolExecution.ts` 中把该 client 状态改成 `needs-auth`。

这样 `/mcp` UI 就会显示“需要重新授权”。

### B. session expired

检测 404 + JSON-RPC `-32001`，或 HTTP transport 上的 connection closed 特征错误，然后：

1. 清掉 server cache
2. 抛出 `McpSessionExpiredError`
3. 允许上层重新获取 fresh client 再重试

这说明 MCP tool 调用不是一次性 RPC，而是要处理“会话失效”这一类连接级故障。

---

## 10. 结果处理：为什么 MCP 输出要做治理

## 10.1 `transformResultContent()`：协议内容转模型内容

MCP result 里可能出现：

- text
- audio
- image
- resource
- resource_link

Claude Code 会把它们统一转换成模型消息块：

- 文本直接保留
- 图片可能压缩 / 缩放后转 image block
- 音频和二进制 blob 持久化到磁盘，再返回说明文本
- resource / resource_link 也转成可回填上下文的 block

这一步的核心目标是：

**把 MCP 协议结果，转换成 Anthropic Messages API 能消费的内容块。**

---

## 10.2 `processMCPResult()`：大结果治理

MCP server 很容易返回超大结果。

如果不处理，会直接导致：

- 上下文爆炸
- token 成本飙升
- 模型无法继续推理

所以 `processMCPResult()` 做了两件事：

1. 估算结果体积
2. 超阈值时截断或落盘

策略大致是：

- 小结果：直接回填给模型
- 大文本结果：保存到文件，返回“请读取该文件”的指令文本
- 含图片的大结果：优先截断，不走 JSON 持久化

这个设计非常重要，因为它把 MCP 从“协议正确”提升到了“可持续参与 Agent 推理”。

---

## 10.3 为什么要推断 schema

`inferCompactSchema()` 会为结构化结果生成简洁 schema 描述。

意义不是做强校验，而是：

- 给持久化后的结果提供更可理解的格式说明
- 帮助系统生成更友好的“如何读取大结果文件”的提示

这是一种偏产品体验的算法，而不是协议必需项。

---

## 11. 权限与命名：MCP 为什么必须进入统一权限系统

`mcpStringUtils.ts` 里的 `getToolNameForPermissionCheck()` 说明了一个关键设计：

对于 MCP tool，权限匹配应该使用完整的：

```text
mcp__server__tool
```

而不是只看原始 tool 名。

原因很直接：

- 如果某个 MCP tool 也叫 `Write`
- 你不能让它误匹配到内置 `Write` 工具的规则

所以 Claude Code 用命名空间把 MCP 工具和内置工具严格隔离开。

这是权限系统正确性的基础。

---

## 12. 关键数据结构一览

初学者最值得重点记住的结构有 6 个：

### 1. `ScopedMcpServerConfig`

表示“某个 server 的连接方式 + 配置来源”。

### 2. `MCPServerConnection`

表示“某个 server 当前运行状态”。

### 3. `Tool`

MCP tool 被适配后也会变成统一 `Tool`。

### 4. `Command`

MCP prompt 被适配后变成统一 `Command`。

### 5. `ServerResource`

在原始 `Resource` 基础上附加 `server` 字段，便于按 server 聚合。

### 6. `MCPCliState`

用于 CLI / UI 侧展示 MCP clients、configs、tools、resources 的序列化状态。

如果把这 6 个结构串起来，基本就能理解整套 MCP 模块在做什么。

---

## 13. 关键业务流程梳理

## 13.1 流程一：用户添加一个 MCP server

```text
claude mcp add ...
  -> addCommand.ts 解析参数
  -> ensureConfigScope / ensureTransport
  -> addMcpConfig()
  -> Zod 校验 + 策略检查 + 保留名检查
  -> 写入对应配置文件
```

如果是 project scope，还会写入 `.mcp.json`。

这里的关键点是：**“添加配置” 不等于 “立刻连上”**。

连接发生在运行期管理逻辑里。

---

## 13.2 流程二：应用启动后接入 MCP

```text
useManageMCPConnections()
  -> 初始化 pending/disabled 状态
  -> getClaudeCodeMcpConfigs()
  -> 异步连接 Claude Code MCP servers
  -> 再拉 claude.ai connectors
  -> 去重后继续连接
  -> onConnectionAttempt 批量更新 AppState.mcp
```

连接成功后又会继续：

```text
connected
  -> fetch tools
  -> fetch commands/prompts
  -> fetch resources
  -> 注册 onclose / list_changed / elicitation handlers
```

---

## 13.3 流程三：模型调用某个 MCP tool

```text
模型输出 mcp__server__tool
  -> 工具系统定位到对应 Tool
  -> Tool.call()
  -> ensureConnectedClient()
  -> client.callTool()
  -> processMCPResult()
  -> tool result 回填模型
```

如果中途出错，还会分流到：

- needs-auth
- session expired
- URL elicitation
- large output persistence

---

## 14. 这套架构为什么设计得合理

从工程视角看，这套 MCP 架构有几个明显优点：

### 1. 配置与运行期分离

`config.ts` 不负责 UI 状态，`useManageMCPConnections.ts` 不负责 schema 校验。

边界很清晰。

### 2. 协议层与产品层分离

`client.ts` 既依赖 MCP SDK，又把协议结果转成内部抽象，而不是让上层直接操作 SDK 原始对象。

### 3. 外部能力统一适配成内部 Tool / Command

这让 MCP 能自然接入 Query Engine、权限系统、工具执行链、UI。

### 4. 充分考虑真实世界故障

包括：

- 401
- OAuth
- session 过期
- 动态 list_changed
- 大结果
- 重复配置
- 远程连接断开

说明这不是“demo 级 MCP 支持”，而是生产级接入层。

---

## 15. 初学者如何阅读这部分代码

推荐按下面顺序读：

1. `types.ts`
   先搞清楚状态和配置对象长什么样。
2. `config.ts`
   先理解“有哪些 server 会被选进来”。
3. `client.ts`
   再理解“选进来的 server 如何连接、发现能力、调用工具”。
4. `useManageMCPConnections.ts`
   最后理解“这些能力如何挂到 UI 和运行时状态”。
5. `toolExecution.ts`
   再补“工具调用失败后如何反馈到全局状态”。

如果一开始就直接啃 `client.ts`，会很容易迷失在 transport、auth、result processing 细节里。

---

## 16. 最后总结

Claude Code 的 MCP 架构可以概括为一句话：

**它把来源复杂、协议多样、状态动态、可能需要认证的外部能力，统一接入为 Claude Code 内部可治理的 Tool / Command / Resource 体系。**

再拆开一点说：

- `config.ts` 解决“接谁”
- `client.ts` 解决“怎么连、怎么发现、怎么调”
- `useManageMCPConnections.ts` 解决“运行时怎么维护”
- `toolExecution.ts` 解决“调用失败后怎么把状态反馈回系统”

因此，MCP 在 Claude Code 中不是一个边缘功能，而是整个 Agent 扩展能力体系的核心组成部分。

对初学者来说，只要抓住下面这条主线，就能把这部分读通：

```text
配置汇总
  -> 连接建立
  -> 能力发现
  -> Tool/Command 适配
  -> 运行期状态维护
  -> 工具调用与结果治理
```

理解了这条主线，也就理解了 Claude Code 是如何把外部世界接入到 Agent 内核中的。
