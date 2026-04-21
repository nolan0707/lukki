# Claude Code Query Harness 可靠性设计梳理

基于 `./specs/out/query_engine.md` 的分层设计梳理，以及 `vendor/Claude-code-open/src/` 中实现代码，这里专门从可靠性视角总结 Claude Code Query Harness 的关键设计。重点关注三条主线：

- 模型访问可靠性
- 上下文管理可靠性
- 工具调度与执行可靠性

整体结论是：

**Claude Code 的可靠性不是一个“重试器”或一个“compact 模块”单独承担的，而是由 `QueryEngine.ts`、`query.ts`、`services/api/*`、`services/tools/*`、`services/compact/*` 共同完成的分层治理。**

---

## 1. 可靠性总览

从实现看，Claude Code 的 Query Harness 可以拆成四层可靠性职责：

1. `QueryEngine.ts`：会话级可靠性  
   负责 transcript 持久化、SDK 输出归一化、compact 边界处理、最终结果收口、错误诊断边界控制。  
   见 `vendor/Claude-code-open/src/QueryEngine.ts:687-1155`

2. `query.ts`：turn 级可靠性状态机  
   负责模型调用后的主循环推进、tool_use/tool_result 闭环、fallback/recovery/abort/stop hooks 协调。  
   见 `vendor/Claude-code-open/src/query.ts:553-1265`, `1360-1565`

3. `services/api/claude.ts` + `services/api/withRetry.ts`：模型访问可靠性  
   负责 API retry、streaming watchdog、stall 检测、streaming -> non-streaming fallback、模型 fallback。  
   见 `vendor/Claude-code-open/src/services/api/claude.ts:1778-2815`，`vendor/Claude-code-open/src/services/api/withRetry.ts:170-516`

4. `services/tools/*` + `services/compact/*`：副作用与上下文可靠性  
   前者保证工具执行顺序、权限、安全和错误回注；后者负责长上下文压缩、边界重建与恢复。  
   见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:19-188`，`vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-1744`，`vendor/Claude-code-open/src/services/compact/*`

这套设计体现的核心原则是：

**所有可恢复故障都不应该直接把会话打断，而应尽量被吸收到统一的消息流和状态机中。**

---

## 2. 模型访问可靠性

模型访问可靠性主要由 `withRetry.ts`、`claude.ts` 和 `query.ts` 协同提供。

### 2.1 Retry 不是“一刀切”，而是按调用源分级

`withRetry.ts` 没有对所有 query source 一视同仁。它区分前台请求和后台请求：

- 前台来源（如 `sdk`、主线程 query、agent 主任务）允许对 529 做 retry
- 后台来源（如总结、建议、标题等）默认不放大重试

这样做的目的不是“更省资源”，而是避免容量雪崩时后台任务放大 3-10 倍 gateway 压力。  
见 `vendor/Claude-code-open/src/services/api/withRetry.ts:57-89`

这说明 Claude Code 的可靠性目标不是盲目“尽量成功”，而是：

**优先保证用户正在等待的主链路成功，同时抑制后台链路对系统稳定性的放大效应。**

### 2.2 连接级自愈：认证失效、陈旧连接、云凭证问题

`withRetry.ts` 在重试前会判断是否需要重建 client，包括：

- 401 / OAuth token revoked
- Bedrock / Vertex 的认证错误
- `ECONNRESET` / `EPIPE` 这类 stale keep-alive socket

对于 stale keep-alive，它甚至会在某些条件下主动禁用 keep-alive，再重建连接。  
见 `vendor/Claude-code-open/src/services/api/withRetry.ts:212-220`

这类设计解决的是“网络连接对象已经不可用，但业务层还没意识到”的问题，属于典型的连接级恢复。

### 2.3 529 过载的处理：有限重试、前后台分流、模型 fallback

对于 529：

- 非前台请求直接放弃，不重试。见 `withRetry.ts:316-324`
- 前台请求允许重试。见 `withRetry.ts:84-89`
- 连续 529 达到阈值后，如果配置了 `fallbackModel`，抛出 `FallbackTriggeredError`。见 `withRetry.ts:326-350`

注意这里并不是在 API 层直接切模型，而是：

- `withRetry.ts` 只发出“需要切模型”的信号
- `query.ts` 捕获这个信号，再做消息状态清理和模型切换

见 `vendor/Claude-code-open/src/query.ts:893-950`

这很关键，因为模型 fallback 不是单纯改一个 `model` 参数，它还涉及：

- 清空本次失败尝试中已收到的 assistant 消息
- 补齐缺失的 `tool_result`
- 清空 toolResults / toolUseBlocks
- 丢弃旧的 streaming executor
- 在需要时剥离 thinking signature，避免新模型不能消费旧模型的签名块

见 `vendor/Claude-code-open/src/query.ts:899-929`

这说明 Claude Code 不是把 fallback 当成“重发一次请求”，而是当成：

**一次受控的状态迁移。**

### 2.4 Streaming 可靠性：watchdog、stall 检测与非流式回退

`claude.ts` 为 streaming 做了两类保护：

1. 被动检测 stall  
   如果 chunk 间隔过长，记录 stall 事件。  
   见 `vendor/Claude-code-open/src/services/api/claude.ts:1944-1965`

2. 主动 watchdog  
   如果长时间没有任何 chunk，到达 idle timeout 后直接 abort stream。  
   见 `vendor/Claude-code-open/src/services/api/claude.ts:1868-1929`

watchdog 解决的不是普通慢请求，而是：

**连接没有明确报错，但流已经不再推进。**

如果 streaming 失败，默认策略不是直接整轮失败，而是回退到 non-streaming：

- 先记录 fallback telemetry
- 再进入 `executeNonStreamingRequest(...)`
- 保持请求上下文和 request_id 链路

见 `vendor/Claude-code-open/src/services/api/claude.ts:2504-2569`

同时它也支持关闭这条 fallback：

- 原因是流式工具执行时，partial stream 可能已经触发了工具
- 若再回退到 non-streaming，同一 `tool_use` 可能再次执行，造成重复副作用

见 `vendor/Claude-code-open/src/services/api/claude.ts:2464-2474`

这体现了一个非常现实的取舍：

**可用性 fallback 不能以破坏副作用幂等性为代价。**

### 2.5 异常流完整性检测：不是收到 200 就算成功

`claude.ts` 不会因为 stream 成功建立就认为调用成功。它还检查：

- 是否收到了 `message_start`
- 是否至少形成过完整 content block
- 是否有 stop_reason

如果 stream 以一种“不完整但不显式报错”的方式结束，也会判定为异常并回退。  
见 `vendor/Claude-code-open/src/services/api/claude.ts:2337-2364`

这类逻辑主要是为了解决代理层 / 网关层返回 200 但 SSE 体不完整的问题。

### 2.6 max_output_tokens 恢复：先隐藏错误，再恢复

`query.ts` 对 `max_output_tokens` 的处理不是立刻把错误抛给上层，而是先 withholding：

- 流内遇到可恢复错误先不 yield
- 让循环先尝试恢复
- 只有恢复失败才真正 surface

见 `vendor/Claude-code-open/src/query.ts:788-823`

恢复策略分两段：

1. 如果当前是默认 capped max output tokens，先直接把上限升高再试一次  
   见 `vendor/Claude-code-open/src/query.ts:1188-1221`

2. 如果仍然截断，则插入一个 meta user message，要求模型“直接续写，不要道歉，不要 recap”  
   见 `vendor/Claude-code-open/src/query.ts:1223-1251`

并且限制恢复次数 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3`。  
见 `vendor/Claude-code-open/src/query.ts:164`, `1223-1255`

这体现的是：

**输出截断被视为可恢复的连续生成问题，而不是普通错误。**

### 2.7 结果回写的可靠性：streaming 消息先 yield，后补 usage/stop_reason

`claude.ts` 在 `content_block_stop` 时就先产出 assistant message，但此时：

- `usage` 还不是最终值
- `stop_reason` 还可能是 `null`

真正的最终 usage / stop_reason 会在后续 `message_delta` 中补齐。  
见 `vendor/Claude-code-open/src/services/api/claude.ts:2213-2237`

而这里特意采用“直接 mutate 已 yield 消息对象”的做法，而不是创建新对象覆盖，原因是 transcript 写入队列持有的是旧对象引用。  
如果替换对象，会导致落盘内容与最终消息不一致。  
见 `vendor/Claude-code-open/src/services/api/claude.ts:2229-2237`

这属于非常典型的“异步流 + 延迟持久化”一致性处理。

---

## 3. 上下文管理可靠性

上下文可靠性由 `autoCompact.ts`、`microCompact.ts`、`compact.ts` 以及 `query.ts` 中的恢复路径共同承担。

### 3.1 上下文管理不是单一 compact，而是多级治理

Claude Code 的上下文治理不是一个动作，而是多个层次：

- `microcompact`：低损耗、预请求削减
- `autoCompact`：基于 token 阈值的主动压缩
- `contextCollapse`：保留更细粒度上下文的 staged collapse
- `reactiveCompact`：收到真实错误后的反应式恢复
- `compact boundary`：compact 之后的持久化和消息链重建

其中 `query.ts` 负责把这些机制编排成一条恢复链。  
见 `vendor/Claude-code-open/src/query.ts:592-647`, `1065-1182`

### 3.2 有效上下文窗口是“模型窗口 - 输出预算 - buffer”

`autoCompact.ts` 不是直接拿模型 context window 做阈值，而是先保留 compact summary 的输出空间，再减 buffer：

- 先根据模型保留最多 20k 作为 summary 输出预算
- 再计算 effective context window
- 再从 effective window 上扣 auto-compact buffer / blocking buffer

见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:28-49`, `62-90`

这说明它考虑的是：

**不是“还能不能发请求”，而是“发请求后还能不能完成一个安全的 compact / continuation”。**

### 3.3 多阈值护栏：warning / error / autocompact / blocking

`calculateTokenWarningState()` 会同时给出：

- warning threshold
- error threshold
- auto-compact threshold
- blocking limit

见 `vendor/Claude-code-open/src/services/compact/autoCompact.ts:93-145`

其中 blocking limit 的作用尤其重要：

- 在 auto-compact 关闭时，防止上下文完全打满
- 保留一定空间给用户显式执行 `/compact`

### 3.4 不是所有情况下都做 proactive 拦截

`query.ts` 在上下文接近极限时，不总是走前置拦截。  
如果启用了：

- `reactiveCompact`
- `contextCollapse`

则会刻意避免提前返回 synthetic prompt-too-long，因为这会让后续基于真实 413 的恢复路径永远得不到触发机会。  
见 `vendor/Claude-code-open/src/query.ts:592-647`

这是一种很成熟的设计：

**恢复逻辑依赖真实故障信号时，不能被更早的“保护性短路”抢跑。**

### 3.5 microcompact：优先低损耗削减，尽量不破坏缓存

`microcompactMessages()` 的优先级是：

1. 先看 time-based microcompact
2. 再看 cached microcompact
3. 不行再交给后续 autocompact

见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:253-292`

cached microcompact 的特点是：

- 不直接改本地 message 内容
- 而是在 API 层发 cache edits / cache reference
- 因此尽量不破坏 prompt cache 命中

见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:295-398`

另外，它只允许主线程运行，避免 forked agents 把自己的 tool_result 注册到全局 cached MC 状态，从而污染主线程上下文。  
见 `vendor/Claude-code-open/src/services/compact/microCompact.ts:272-285`

### 3.6 真正的 overflow 恢复顺序：collapse drain -> reactive compact -> surface

当流中出现被 withholding 的 prompt-too-long 时，`query.ts` 的恢复顺序是：

1. 若开启 context collapse，先 `recoverFromOverflow()`，把 staged collapse 真正提交  
   见 `vendor/Claude-code-open/src/query.ts:1085-1117`

2. 如果还不够，再尝试 `reactiveCompact.tryReactiveCompact(...)`  
   见 `vendor/Claude-code-open/src/query.ts:1119-1165`

3. 仍然失败，再把 withheld error 真正 yield 出来  
   见 `vendor/Claude-code-open/src/query.ts:1168-1182`

这里的逻辑优先级非常有含义：

- collapse drain 更便宜，且能保留更细粒度上下文
- reactive compact 更激进，本质上是更强的压缩 / strip-retry

因此它优先尝试“最小损伤恢复”。

### 3.7 死循环防护：不要让错误恢复链和 hook 互相放大

在 prompt-too-long / media size 错误无法恢复时，`query.ts` 会明确：

- 直接 surface 错误
- 只运行 stop failure hooks
- 不再进入正常 stop hooks

理由在注释里写得很清楚：

如果继续跑 stop hooks，hook 可能注入额外 token，于是形成：

`error -> hook blocking -> retry -> error -> ...`

见 `vendor/Claude-code-open/src/query.ts:1168-1175`, `1258-1264`

这说明其设计非常清楚地区分了：

- “正常收尾路径”
- “失败收尾路径”

### 3.8 compact 边界不是简单摘要，而是带重链元数据的恢复点

`buildPostCompactMessages()` 会把 compact 结果重组为：

- boundary marker
- summary messages
- messagesToKeep
- attachments
- hookResults

见 `vendor/Claude-code-open/src/services/compact/compact.ts:330-337`

更关键的是 `annotateBoundaryWithPreservedSegment()`：

- boundary 上会带 `preservedSegment`
- 指明 preserved tail 的 head / anchor / tail
- loader 据此可以把 compact 之后保留的消息重新接回消息链

见 `vendor/Claude-code-open/src/services/compact/compact.ts:340-367`

这说明 compact 并不是“删除一段历史”那么简单，而是：

**对消息链进行一次带 relink 信息的重写。**

### 3.9 QueryEngine 对 compact 的可靠性补完：先落盘再裁内存

`QueryEngine.ts` 在遇到 compact boundary 时，会先把 boundary 前 preserved tail 对应的消息强制写盘，然后再在内存里裁掉 pre-compact 消息。  
见 `vendor/Claude-code-open/src/QueryEngine.ts:693-715`, `917-933`

原因是：

- 如果还没写盘，桌面端进程在 turn 间被杀掉
- loader 在恢复时就找不到 tailUuid 对应消息
- 最终导致 preserved segment relink 失败，恢复出完整 pre-compact 历史

这类处理非常实用，解决的是“compact 正确，但 crash/restart 之后恢复错误”的问题。

---

## 4. 工具调度与执行可靠性

工具可靠性主要体现在四件事上：

- 调度顺序正确
- `tool_use` / `tool_result` 始终配对
- 权限与 hooks 不破坏主循环
- 工具失败能转化成可继续推理的上下文

### 4.1 follow-up 触发条件依赖真实 block，而不依赖 stop_reason

`query.ts` 明确指出 `stop_reason === 'tool_use'` 并不可靠，因此真正的 follow-up 触发条件是：

- 流中收到 `tool_use` block
- 就把 `needsFollowUp = true`

见 `vendor/Claude-code-open/src/query.ts:553-558`, `826-845`

这体现了典型的可靠性策略：

**不要依赖上游不稳定的摘要信号，要依赖最原始、最可验证的结构化事件。**

### 4.2 调度器不是简单并发，而是语义分批

`toolOrchestration.ts` 不会把所有工具直接 `Promise.all`。  
它先按 `isConcurrencySafe` 切 batch：

- 连续的只读 / 可安全并发工具合成一批
- 非安全工具单独成批

见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:19-82`, `91-116`

如果 `safeParse` 失败或 `isConcurrencySafe` 自身抛错，也会保守降为非并发。  
见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:97-107`

这说明该调度器的第一原则不是吞吐量，而是：

**上下文一致性优先。**

### 4.3 并发执行但顺序提交上下文

在并发 batch 中：

- 工具执行可以并发
- 但 `contextModifier` 不会按完成时间立刻提交
- 而是暂存起来，最后按原 tool_use 顺序回放

见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:30-63`

这是工具可靠性设计里最关键的一点之一。

它保证：

- 读操作并发提升吞吐
- 上下文提交顺序仍然稳定

否则只要两个工具都修改 `ToolUseContext`，就会出现“谁先结束谁先改上下文”的竞态。

### 4.4 in-progress 状态显式维护，便于中断和恢复

无论串行还是并发，tool orchestration 都会维护 `inProgressToolUseIDs`：

- 开始前加入
- 完成后删除

见 `vendor/Claude-code-open/src/services/tools/toolOrchestration.ts:126-149`, `152-188`

这为以下场景提供了基础：

- UI 正确展示运行中工具
- abort 时识别哪些工具未完成
- executor 生成 synthetic tool_result

### 4.5 工具输入可靠性：schema 校验、值校验、内部字段剥离

`toolExecution.ts` 在真正调用工具前有三层输入治理：

1. `inputSchema.safeParse(...)`  
   失败则直接返回 `tool_result(is_error=true)`  
   见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:614-680`

2. `tool.validateInput?.(...)`  
   工具自定义值校验失败同样转成 `tool_result`  
   见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:682-733`

3. defense-in-depth 清洗  
   例如 Bash 主动剥离 `_simulatedSedEdit` 等内部字段  
   见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:756-773`

这说明工具层的错误不是异常流的一部分，而是：

**模型下一轮推理必须能消费的正常观察值。**

### 4.6 hooks 和权限系统被整合到工具执行主路径中

工具调用不是直接 `tool.call()`，而是：

1. `PreToolUse hooks`
2. 权限决策 / `canUseTool`
3. 真正执行 tool
4. `PostToolUse hooks`
5. `PostToolUseFailure hooks`

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:795-1103`, `1397-1588`, `1696-1737`

这里最重要的可靠性价值在于：

- hooks 可以阻止继续执行，但仍然产出配套 `tool_result`
- permission deny 也会产出 `tool_result`
- 即使失败，也有 `PostToolUseFailure` hooks 接管收尾

因此：

**权限系统和 hook 系统不会把 tool loop 打断成不完整状态。**

### 4.7 权限拒绝也要回注为 tool_result

当权限不是 allow 时，系统不会 simply stop。  
它会构造一个 user message，其中包含：

- `tool_result`
- `is_error: true`
- 必要时附带图片或反馈内容

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:995-1103`

如果是 auto-mode classifier 拒绝，还能通过 `PermissionDenied hook` 告诉模型“现在可以重试”。  
见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1073-1100`

这体现的是 agentic 系统里一个很重要的原则：

**权限拒绝不是循环终止信号，而是模型的观察结果。**

### 4.8 backfill 输入时区分“可观察输入”和“真实执行输入”

`toolExecution.ts` 很小心地区分：

- `processedInput`：给 hooks / permissions / telemetry 看的输入
- `callInput`：真正传给 `tool.call()` 的输入

这样做是为了避免：

- path expansion 等派生字段污染 tool 输出
- 改写 transcript 中的 observable input
- 破坏 VCR fixture hash

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:775-793`, `1181-1205`

这是一个很细但很关键的可靠性设计：

**执行时内部规范化，不应改变对模型和 transcript 可见的事实。**

### 4.9 成功与失败都必须生成规范 tool_result

成功时：

- 先统一 map tool result
- 控制结果尺寸
- 再包装成 user message 回注
- 如果工具返回 `contextModifier`，作为附带元数据返回给 orchestration

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1290-1473`

失败时：

- 统一格式化 error
- 生成 `tool_result(is_error=true)`
- 再附加 `PostToolUseFailure hooks`

见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1589-1737`

这保证了无论成功失败，模型总能看到闭环的 `tool_result`。

### 4.10 MCP 特殊失败会回写系统状态，而不仅仅是返回错误

如果工具失败是 `McpAuthError`，系统除了返回错误，还会把对应 MCP client 状态改成 `needs-auth`。  
见 `vendor/Claude-code-open/src/services/tools/toolExecution.ts:1599-1629`

这意味着工具错误不仅影响本轮推理，也会同步修正系统状态，使后续 UI 和操作路径正确反映“需要重新授权”。

### 4.11 中断场景下的 tool_use/tool_result 配对保证

这是 Claude Code 工具可靠性里最强的一条护栏之一。

如果在 streaming 过程中用户中断：

- 若使用 `StreamingToolExecutor`，会继续消费 `getRemainingResults()`
- 让 executor 为未完成 / 已排队工具生成 synthetic `tool_result`
- 若没有 streaming executor，也会遍历 assistantMessages，补齐缺失的 tool_result

见 `vendor/Claude-code-open/src/query.ts:1011-1051`

对应的 fallback 场景也类似：

- streaming fallback 时先 tombstone 旧 assistant 消息
- 丢弃旧 executor
- 避免旧 `tool_use_id` 的结果泄漏到新尝试

见 `vendor/Claude-code-open/src/query.ts:712-740`

模型 fallback 也会先 `yieldMissingToolResultBlocks(...)` 再切模型。  
见 `vendor/Claude-code-open/src/query.ts:899-903`

这说明它非常清楚一个 agent loop 的硬约束：

**不能让 transcript 中出现孤立的 `tool_use`。**

---

## 5. 会话级可靠性：QueryEngine 的外层护栏

虽然可靠性的主战场在 `query.ts` 和底层服务，但 `QueryEngine.ts` 还承担了几个关键的外层护栏。

### 5.1 transcript 持久化与流式最终值一致性

`QueryEngine` 对 transcript 写入做了区分：

- assistant 消息 fire-and-forget
- user / compact boundary 等关键消息同步写入

见 `vendor/Claude-code-open/src/QueryEngine.ts:716-731`

这么做是为了让：

- `claude.ts` 后续 `message_delta` 还能把最终 stop_reason / usage 写回同一对象
- transcript lazy flush 最终落盘的是最终值

### 5.2 compact boundary 的写盘顺序控制

在 compact boundary 到来前，`QueryEngine` 会先补写 preserved tail 相关消息，再记录 boundary。  
见 `vendor/Claude-code-open/src/QueryEngine.ts:693-715`

否则一旦 SDK 子进程在 turn 间重启，恢复流程可能找不到 preserved tail，导致 compact 后消息链错误重建。

### 5.3 compact 后立即释放旧消息，防止长会话内存膨胀

当收到 compact boundary，`QueryEngine` 会：

- 把 `mutableMessages` 截断到 boundary 之后
- 本地 `messages` 也同步裁剪

见 `vendor/Claude-code-open/src/QueryEngine.ts:917-942`

这和 REPL 保留完整滚动历史不同，headless/SDK 场景更关心：

**长会话下内存占用可控。**

### 5.4 最终 result 的收口不是“最后一条消息”，而是显式成功判定

`QueryEngine` 在产出最终 `result` 前，会：

- 只在 `assistant | user` 中找最终语义消息
- 用 `isResultSuccessful(...)` 做显式判定
- 若失败，只返回当前 turn 范围内的错误日志

见 `vendor/Claude-code-open/src/QueryEngine.ts:1051-1118`

这样做避免了：

- progress / attachment 抢占“最后一条消息”
- 整个进程级错误日志污染当前 turn 诊断

---

## 6. 关键设计总结

### 6.1 模型访问可靠性

Claude Code 的模型访问可靠性可以概括为：

- retry 分级，不对所有请求盲重试
- streaming 有 watchdog 和 stall 监控
- streaming 失败优先降级到 non-streaming
- 连续过载会触发模型 fallback
- 输出截断被视为可恢复 continuation 问题

核心思想是：

**请求失败时优先保证状态机还能继续，而不是立刻终止整轮。**

### 6.2 上下文管理可靠性

上下文管理不是“压缩一次就结束”，而是：

- 预防性阈值治理
- 低损耗 microcompact
- 基于真实错误的 reactive recovery
- compact boundary + preserved segment relink
- 持久化与内存裁剪协同

核心思想是：

**上下文满了是可恢复故障，不是必须终止会话的致命故障。**

### 6.3 工具调度与执行可靠性

工具部分最关键的可靠性保证是：

- 调度按语义分批，不是盲并发
- 并发执行但顺序提交上下文
- 每个 `tool_use` 必须配对 `tool_result`
- 权限拒绝、hook 拦截、工具失败都能回注到统一消息流
- 中断 / fallback / 模型切换时也要补齐或清理 tool 轨迹

核心思想是：

**工具系统不能破坏 assistant trajectory 的闭环结构。**

---

## 7. 最值得借鉴的三点

如果把 Claude Code 的可靠性设计抽成最值得迁移的三点，我认为是：

1. **把恢复做成状态机迁移，而不是临时补丁**  
   fallback、compact、abort、tool failure 都不是旁路逻辑，而是主循环的一部分。

2. **把错误也编码成消息流的一部分**  
   `tool_result(is_error)`、withheld API error、api_retry system message、compact boundary，都是模型和 SDK 都能理解的结构化事件。

3. **保证轨迹自洽优先于局部最优**  
   不允许孤儿 `tool_use`，不允许乱序 contextModifier，不允许 compact 后消息链断裂，不允许 transcript 落盘和最终 stop_reason 不一致。

---

## 8. 补充说明

关于 `reactiveCompact`：

- 当前 vendor 目录里未直接展开对应实现文件
- 但 `query.ts` 的调用点已经足够确认其在恢复链中的位置：
  - 流中先 withholding 可恢复错误
  - 完成后先尝试 collapse drain
  - 再尝试 reactive compact
  - 最后才 surface 错误

这部分结论是基于调用点和状态机行为推断，而非直接阅读 `reactiveCompact` 实现文件本体。  
见 `vendor/Claude-code-open/src/query.ts:788-823`, `1065-1182`

---

## 9. 可靠性设计总表

| 主题 | 可靠性问题 | 机制/策略 | 关键行为 | 主要价值 | 关键源码 |
| --- | --- | --- | --- | --- | --- |
| 模型访问 | 529 过载放大 | 前后台分级 retry | 前台请求重试，后台请求快速失败 | 避免容量雪崩时后台任务放大流量 | `services/api/withRetry.ts:57-89`, `316-324` |
| 模型访问 | 连续过载不可恢复 | 模型 fallback 信号 | 连续 529 达阈值后抛 `FallbackTriggeredError` | 不在 API 层偷偷切模型，交给状态机统一处理 | `services/api/withRetry.ts:326-350` |
| 模型访问 | fallback 后状态脏污 | fallback 状态迁移 | 补齐缺失 `tool_result`、清空旧 assistant/tool 状态、重建 executor、剥离 thinking signature | 保证模型切换后消息轨迹仍可继续 | `query.ts:893-950` |
| 模型访问 | stale keep-alive / 连接失效 | 连接级自愈 | 识别 `ECONNRESET`/`EPIPE`，必要时禁用 keep-alive 并重建 client | 修复“连接对象已坏但业务未感知”的问题 | `services/api/withRetry.ts:112-118`, `212-220` |
| 模型访问 | 流式假挂起 | stream watchdog | 长时间无 chunk 主动 abort stream | 防止 streaming 无限挂死 | `services/api/claude.ts:1868-1929` |
| 模型访问 | 流式长停顿 | stall detection | 记录 chunk 间隔过长的 stall | 可观测性增强，便于诊断慢流 | `services/api/claude.ts:1944-1965` |
| 模型访问 | streaming 失败 | streaming -> non-streaming fallback | 流失败后回退到非流式请求 | 保障主流程可继续 | `services/api/claude.ts:2504-2569` |
| 模型访问 | fallback 造成重复工具副作用 | 可关闭 non-streaming fallback | 当流式工具执行存在重复执行风险时允许禁掉回退 | 在可用性与幂等性之间做平衡 | `services/api/claude.ts:2464-2474` |
| 模型访问 | 200 但 SSE 不完整 | 异常流完整性校验 | 无 `message_start` / 无完整 content block / 无 stop_reason 时判异常 | 避免把代理层半残流当成功 | `services/api/claude.ts:2337-2364` |
| 模型访问 | 输出被截断 | max output tokens 恢复 | 先 withholding，再升高上限，再插入 meta continuation message 恢复 | 把输出截断视为可恢复 continuation，而非直接失败 | `query.ts:788-823`, `1185-1255` |
| 模型访问 | streaming 消息先出后补 usage | 直接 mutate 已 yield 消息 | `message_delta` 回写最终 `usage/stop_reason` 到已有消息对象 | 保证 transcript 懒写最终落盘正确值 | `services/api/claude.ts:2213-2237` |
| 上下文管理 | 上下文窗口计算过于理想化 | effective context window | 上下文窗口先扣 summary 输出预算，再扣 buffer | 阈值基于“可安全运行窗口”而非裸窗口 | `services/compact/autoCompact.ts:32-49`, `62-90` |
| 上下文管理 | 上下文逼近极限缺少护栏 | 多阈值控制 | 同时维护 warning / error / autocompact / blocking 四类阈值 | 为提示、自动压缩、硬阻断提供分层策略 | `services/compact/autoCompact.ts:93-145` |
| 上下文管理 | proactive 拦截抢跑恢复链 | 条件性 blocking preempt | 若启用 reactive compact / context collapse，避免过早返回 synthetic PTL | 让真实 413 触发后续恢复逻辑 | `query.ts:592-647` |
| 上下文管理 | 历史太长但不想重压缩 | microcompact | 优先走 time-based / cached microcompact | 以较低损耗削减历史体积 | `services/compact/microCompact.ts:253-292` |
| 上下文管理 | 压缩破坏 prompt cache | cached microcompact | 不改本地 message，只向 API 层发 cache edits | 尽量缩历史但保留 cache 命中 | `services/compact/microCompact.ts:295-398` |
| 上下文管理 | forked agent 污染主线程压缩状态 | 主线程限定 | cached MC 仅允许 main thread 使用 | 防止子 agent 的 tool result 混入主线程状态 | `services/compact/microCompact.ts:272-285` |
| 上下文管理 | prompt too long 恢复路径混乱 | collapse drain -> reactive compact -> surface | 先 drain staged collapse，再 reactive compact，最后才真正报错 | 优先最小损伤恢复，再做激进恢复 | `query.ts:1065-1182` |
| 上下文管理 | 恢复失败后与 hooks 互相放大 | stop hook death spiral 防护 | API error 场景不再进入正常 stop hooks | 避免 `error -> hook -> 更大上下文 -> 再 error` 循环 | `query.ts:1168-1175`, `1258-1264` |
| 上下文管理 | compact 后消息链断裂 | compact boundary + preserved segment | boundary 带 `head/anchor/tail` relink 元数据 | 让 compact 后保留尾段能正确接回消息链 | `services/compact/compact.ts:330-367` |
| 上下文管理 | compact 后 crash/restart 恢复错误 | boundary 前强制写盘 | 先落盘 preserved tail，再记录 boundary | 防止恢复时找不到 tailUuid 导致 relink 失败 | `QueryEngine.ts:693-715` |
| 上下文管理 | 长会话内存膨胀 | compact 后裁剪旧消息 | 收到 compact boundary 后释放 pre-compact history | 让 headless/SDK 长会话内存可控 | `QueryEngine.ts:917-942` |
| 工具调度与执行 | `stop_reason=tool_use` 不可靠 | 依赖真实 tool_use block | 流中收到 tool_use 即设置 `needsFollowUp` | 不依赖上游不稳定摘要信号 | `query.ts:553-558`, `826-845` |
| 工具调度与执行 | 盲并发破坏语义 | 语义分批调度 | 按 `isConcurrencySafe` 切 batch，失败时保守降级 | 保证上下文一致性优先于吞吐 | `services/tools/toolOrchestration.ts:19-116` |
| 工具调度与执行 | 并发执行导致上下文竞态 | 顺序提交 `contextModifier` | 并发跑工具，但延后并按原 tool_use 顺序提交 contextModifier | 保证上下文更新顺序稳定 | `services/tools/toolOrchestration.ts:30-63` |
| 工具调度与执行 | 无法识别未完成工具 | `inProgressToolUseIDs` | 开始执行时加入，完成时删除 | 为 UI、中断恢复、执行跟踪提供稳定状态 | `services/tools/toolOrchestration.ts:126-188` |
| 工具调度与执行 | 模型生成错误工具参数 | schema 校验 | `safeParse` 失败直接转成 `tool_result(is_error=true)` | 让模型下一轮可消费结构化错误反馈 | `services/tools/toolExecution.ts:614-680` |
| 工具调度与执行 | 参数类型对了但语义非法 | 工具级值校验 | `validateInput` 失败同样回注错误 `tool_result` | 保持错误闭环在消息流内部 | `services/tools/toolExecution.ts:682-733` |
| 工具调度与执行 | 模型伪造内部字段 | defense-in-depth 清洗 | 如 Bash 剥离 `_simulatedSedEdit` | 防止内部权限机制被模型直接绕过 | `services/tools/toolExecution.ts:756-773` |
| 工具调度与执行 | hook / permission 与执行割裂 | hooks + permission 融入主路径 | PreToolUse -> permission -> call -> PostToolUse / Failure hooks | 让 hook/权限不打断 tool loop 完整性 | `services/tools/toolExecution.ts:795-1103`, `1397-1737` |
| 工具调度与执行 | 权限拒绝导致循环中断 | deny 也生成 `tool_result` | permission deny 构造成标准错误 tool_result，并可附带重试 hint | 权限拒绝成为模型可观察结果 | `services/tools/toolExecution.ts:995-1103` |
| 工具调度与执行 | 内部 backfill 污染 transcript / VCR | 区分 `processedInput` 与 `callInput` | hooks/permission 看派生输入，`tool.call()` 尽量保留模型原始输入 | 保持可观察事实与内部执行规范化解耦 | `services/tools/toolExecution.ts:775-793`, `1181-1205` |
| 工具调度与执行 | 成功或失败路径格式不一致 | 统一 `tool_result` 回注 | 成功统一 map result，失败统一 format error，并包装为 user message | 保证 assistant trajectory 闭环结构统一 | `services/tools/toolExecution.ts:1290-1473`, `1589-1737` |
| 工具调度与执行 | MCP 认证过期只在本轮报错 | 失败同步修正系统状态 | `McpAuthError` 时把 client 状态改成 `needs-auth` | 让后续 UI 和系统状态保持一致 | `services/tools/toolExecution.ts:1599-1629` |
| 工具调度与执行 | 中断后出现孤儿 `tool_use` | synthetic tool_result 补齐 | abort 时消费剩余 executor 结果，或显式补齐缺失 tool_result | 保证 transcript 中 `tool_use/tool_result` 始终配对 | `query.ts:1011-1051` |
| 工具调度与执行 | fallback 后旧工具结果泄漏 | tombstone + discard 旧 executor | streaming fallback / model fallback 时清理旧尝试残留 | 防止旧 `tool_use_id` 结果污染新尝试 | `query.ts:712-740`, `899-914` |
| 会话级 | transcript 落盘与流式最终值不一致 | 分类型持久化 | assistant fire-and-forget，关键消息同步写盘 | 保证 stop_reason/usage 回写不被阻塞，且顺序正确 | `QueryEngine.ts:716-731` |
| 会话级 | progress/attachment 干扰最终结果提取 | 显式成功判定 | 最终只在 `assistant|user` 中找语义结果，并走 `isResultSuccessful` | 避免把非语义消息误判为结果 | `QueryEngine.ts:1051-1118` |
| 会话级 | 诊断信息跨 turn 污染 | turn-scoped error watermark | 只收集本 turn 之后的 in-memory errors | 提高最终错误诊断信噪比 | `QueryEngine.ts:1058-1115` |
