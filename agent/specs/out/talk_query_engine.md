# 以架构视角，深入源码，逐层拆解 Claude Code 的 Harness 设计哲学

## 1. 开场：今天我们真正要回答的问题是什么

如果只从产品视角看 Claude Code，很容易把它理解成一句话：

“用户输入一个需求，模型边输出边调用工具，最后返回结果。”

但从源码视角看，真正的问题不是“模型怎么回答”，而是：

1. 模型输出 `tool_use` 之后，谁决定什么时候执行工具？
2. 工具执行后，谁保证结果能稳定地回到下一轮上下文？
3. 上下文爆掉时，谁负责自愈，而不是让会话直接崩溃？
4. 权限、重试、fallback、compact，这些复杂逻辑到底挂在哪一层？
5. 为什么 Claude Code 在工程上看起来不像“一个 prompt”，而像“一个操作系统里的调度内核”？

今天这场分享的核心结论会是：

**Claude Code 的核心不在模型，而在 Harness。模型只负责提出候选动作，Harness 负责把这些动作组织成一个可持续、可恢复、可审计的系统。**

---

## 2. 第一张图：整体架构先看全貌

如果直接从 `QueryEngine.ts` 开始读，很容易误以为所有复杂性都塞在这一个文件里。  
但按源码分层看，更准确的结构是这样：

```text
┌──────────────────────────────────────────────┐
│ SDK / Remote Client / UI                     │
│ 消费 SDKMessage 流，渲染结果与状态            │
└──────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────┐
│ QueryEngine.ts                               │
│ 会话级 Harness                               │
│ - submitMessage() 对外异步消息流接口         │
│ - system prompt 拼装                         │
│ - 输入预处理 / transcript 持久化             │
│ - SDK message 归一化 / result 汇总           │
└──────────────────────────────────────────────┘
                      │
                      ▼
┌──────────────────────────────────────────────┐
│ query.ts                                     │
│ turn 级控制循环 / agent 状态机               │
│ - queryLoop()                                │
│ - tool_use -> tool_result 闭环               │
│ - compact / retry / recovery / stop hooks    │
│ - token budget / turn continuation           │
└──────────────────────────────────────────────┘
          │                     │
          │                     │
          ▼                     ▼
┌──────────────────────┐   ┌──────────────────────┐
│ services/api/        │   │ services/tools/      │
│ claude.ts            │   │ toolOrchestration.ts │
│ API / streaming      │   │ + toolExecution.ts   │
│ Harness              │   │ Tool Harness         │
│ - 请求构造           │   │ - 分批调度           │
│ - thinking/effort    │   │ - 权限校验           │
│ - 流式事件解析       │   │ - hook / tool call   │
│ - retry/fallback     │   │ - tool_result /      │
│                      │   │   contextModifier    │
└──────────────────────┘   └──────────────────────┘
          │                     │
          └──────────┬──────────┘
                     ▼
┌──────────────────────────────────────────────┐
│ compact/*                                    │
│ Context Management Harness                   │
│ - microcompact / autocompact                 │
│ - reactive compact / context collapse        │
│ - compact_boundary                           │
└──────────────────────────────────────────────┘
```

这里先抛出第一条结论：

**Claude Code 不是“一个文件里的一坨逻辑”，而是“一个分层 Harness”。**

---

## 3. 第二张图：三层异步生成器，构成一条消息流

很多同学看到 `submitMessage()` 时会觉得它只是一个流式接口。  
但更准确的说法是：Claude Code 用了三层异步生成器，把“模型流 -> agent 循环 -> SDK 输出”串成一条统一消息流。

```text
SDK / 调用方
  |
  | for await (msg of QueryEngine.submitMessage(...))
  v
QueryEngine.submitMessage()
  |
  | for await (message of query(...))
  v
query()
  |
  | for await (message of claude.ts / callModel(...))
  v
claude.ts / queryModelWithStreaming()
  |
  | for await (part of Anthropic Stream)
  v
Anthropic API Stream
```

这三层分别对应三种职责：

### 第 1 层：`claude.ts`

它负责和模型 API 打交道：

- 组装请求参数
- 解析流式事件
- 处理 `thinking` / `text` / `tool_use`
- 处理 streaming fallback、non-streaming fallback、retry

一句话：**它面对的是模型流。**

### 第 2 层：`query.ts`

它负责 turn 级控制循环：

- 接收 assistant 输出
- 判断是否有 `tool_use`
- 执行工具
- 回注 `tool_result`
- 触发 compact / recovery / stop hooks
- 决定本轮是继续还是结束

一句话：**它面对的是 agent 状态机。**

### 第 3 层：`QueryEngine.ts`

它负责对外接口：

- 处理输入
- 拼 system prompt
- 维护会话级消息状态
- 归一化成 SDKMessage
- 最终输出 `result`

一句话：**它面对的是 SDK 调用方。**

所以分享里可以直接给一句口号：

**`claude.ts` 管流，`query.ts` 管循环，`QueryEngine.ts` 管对外接口。**

---

## 4. 第三张图：真正的 Agent Loop 不在 QueryEngine，而在 query.ts

这部分是最容易讲错的地方。

很多分析会把“工具调用循环”直接归到 `QueryEngine.ts`。  
但源码显示，真正的 Think-Act-Observe 循环中枢是在 `query.ts`。

它的核心形态是：

```text
模型输出 assistant
   ↓
如果包含 tool_use
   ↓
执行工具
   ↓
产生 tool_result
   ↓
把 tool_result 回注到上下文
   ↓
再次请求模型
```

这个闭环最关键的点有两个：

### 第一，tool_use 不是副产品，而是循环分支条件

`query.ts` 在流式处理 assistant 输出时，只要发现 `tool_use`，就会把本轮标记成 `needsFollowUp = true`。  
这意味着：

- 本轮不能直接结束
- 必须进入工具执行阶段
- 然后带着工具结果发起下一轮

也就是说，Claude Code 不是“边生成边顺手调个工具”，而是：

**把 `tool_use` 当作状态机中的显式转移条件。**

### 第二，tool_result 是回到模型闭环里的桥

工具执行完以后，结果不会停留在工具层，而是会被封装成 user message 中的 `tool_result` 内容块，再回注到下一轮上下文。

所以：

- `assistant(tool_use)` 是模型提出动作
- `tool_result` 是系统把现实世界执行结果返还给模型

这一进一出，构成了真正的 agentic loop。

这时可以给听众一个定义：

**Claude Code 不是在“调用工具”，而是在“让模型通过 Harness 与现实世界做一轮受控交互”。**

---

## 5. 第四张图：工具执行为什么不是无脑并发

这一层最能体现 Claude Code 的工程质量。

很多系统拿到一批 `tool_use` 后，第一反应是：

```text
Promise.all 跑起来
```

Claude Code 不是这样。它会先做一件更重要的事：

**判断哪些工具可以并发，哪些必须串行。**

### 5.1 它为什么像一个调度器

`toolOrchestration.ts` 的职责不是“执行工具”，而是“调度工具”：

- 先用 `partitionToolCalls()` 按 `isConcurrencySafe(...)` 分批
- 可并发的一批一起跑
- 不可并发的批次严格串行

因此这里的关键不是速度，而是**上下文一致性**。

### 5.2 什么叫“受上下文一致性约束的调度器”

因为工具执行不只是 I/O，还会影响后续上下文：

- 有些工具只是读取信息，适合并发
- 有些工具有副作用，必须保持顺序
- 有些工具还会通过 `contextModifier` 修改 `ToolUseContext`

所以 Claude Code 必须保证：

1. 能并发的才并发
2. 不能并发的严格串行
3. 上下文更新顺序稳定，不能依赖“谁先跑完”

这就是“受上下文一致性约束的调度器”。

### 5.3 真正精妙的点：并发执行，顺序提交

在并发 batch 里，多工具可以同时执行。  
但如果它们返回了 `contextModifier`，系统不会按完成时间立即改写上下文。

而是：

1. 先把 modifier 暂存起来
2. 等整个 batch 完成
3. 再按原始 `tool_use` 顺序依次应用

这一步非常关键，因为它消除了非确定性。

否则你会得到这种不稳定行为：

```text
tool A / tool B 并发
B 先完成，先改 context
A 后完成，再改 context
```

最终上下文就取决于机器快慢，而不是模型原始意图。  
Claude Code 明确不接受这种不确定性。

所以你可以在分享里强调一句：

**Claude Code 允许“执行并发”，但不允许“语义乱序”。**

---

## 6. 第五张图：`tool_result` 和 `contextModifier` 是两条不同状态通道

工具执行完成后，系统的影响不是单一路径，而是分成两路。

### 路径一：`tool_result`

它是给模型看的。

- 会进入消息历史
- 下一轮模型能看到
- 用来构成 agent loop 的观察结果

### 路径二：`contextModifier`

它是给 Harness 自己看的。

- 不进入消息历史
- 模型下一轮看不到
- 只用于更新后续工具执行环境 `ToolUseContext`

这个区分非常重要，因为它揭示了 Claude Code 的一个设计哲学：

**不是所有状态都应该塞进消息历史。**

Claude Code 会按作用对象把状态分流：

- 面向模型的状态，进消息流
- 面向工具层的状态，进运行时上下文

这就是为什么我们说它不像一个“prompt 工程项目”，而像一个“消息总线 + 状态机”的系统。

---

## 7. 第六张图：上下文治理不是一个压缩开关，而是一条 pipeline

如果只从外面观察，很多人会把 compaction 理解成：

“上下文长了，就压缩一下。”

但源码里不是一个单点动作，而是一套多层 context management pipeline。

每次真正发请求前，`query.ts` 都会依次处理：

1. `applyToolResultBudget()`  
   限制 tool result 聚合体积

2. `snipCompactIfNeeded()`  
   裁掉一部分历史

3. `microcompact()`  
   优先清理工具结果等低价值上下文

4. `contextCollapse.applyCollapsesIfNeeded()`  
   做折叠式上下文投影

5. `autocompact()`  
   必要时做摘要压缩，重建 post-compact 视图

所以正确理解不是：

**“超限了就压缩”**

而是：

**“在真正进入 full compact 之前，系统会先尝试多种更低损耗的上下文治理手段。”**

这反映出非常强的工程取向：

**Claude Code 不轻易牺牲上下文保真度，而是分阶段、分层次地做降损处理。**

---

## 8. 第七张图：`compact_boundary` 是上下文版本切换锚点

这里是整个分享里最值得讲透的点之一。

很多人会把 compact 讲成“生成一个 summary”。  
但从 Harness 视角看，更重要的其实不是 summary，而是 `compact_boundary`。

### 8.1 它不是内容，而是结构

`compact_boundary` 是一条 system message。  
它的含义不是“压缩过了，提醒一下”，而是：

**从这条边界开始，新的活跃上下文视图生效。**

### 8.2 它解决的是“上下文视图切换”

压缩前你有完整历史。

压缩后系统不能继续带着完整历史跑，否则压缩没有意义。  
这时就需要一个显式边界告诉系统：

- 前面的历史不再按原样参与后续 query
- 后面的 summary / preserved messages / recent messages 才是新的 active context

### 8.3 它为什么不是可有可无的提示

因为 `QueryEngine.ts` 收到 `compact_boundary` 后，会做两件实质动作：

1. 向 SDK 发出 compact boundary 事件
2. 直接裁掉内存里边界之前的消息

所以它是：

- SDK 可见事件
- 内存裁剪锚点
- transcript / resume 语义边界
- active context 起点

一句话总结：

**summary 定义“旧上下文被压成什么”，`compact_boundary` 定义“从哪里开始切换到新上下文”。**

---

## 9. 第八张图：`mutableMessages`、本地 `messages` 和 `compact_boundary` 的联动

这一段很适合拿来展示源码层面的“状态工程”。

### `mutableMessages`

它是 `QueryEngine` 实例持有的会话级主状态。  
可以把它理解成整个 conversation 在 engine 内部的长期工作内存。

### 本地 `messages`

它是 `submitMessage()` 这一轮调用内部的工作副本。  
可以把它理解成当前 turn 的处理视图。

### 它们为什么要并存

因为系统同时需要：

- 会话级长期状态
- 当前轮次工作状态
- transcript / result 汇总时的局部控制

所以不是一个数组就够，而是：

```text
mutableMessages  --拷贝-->  当前 submitMessage() 的本地 messages
```

### `compact_boundary` 如何作用于它们

一旦 `compact_boundary` 到来：

1. `QueryEngine` 先把边界消息 push 进去
2. 然后把边界前的历史裁掉
3. 这一步同时作用于：
   - `mutableMessages`
   - 当前 turn 的本地 `messages`

这意味着：

- engine 长期状态切换到 post-compact 视图
- 当前 turn 工作状态也同步切换到 post-compact 视图

也正因此，Claude Code 的压缩不是“某次 query 的临时技巧”，而是：

**对整个会话运行状态的结构性重写。**

---

## 10. 第九张图：重试、自愈、fallback，本质上都是 Harness 的恢复机制

很多产品介绍会把这些内容分开讲：

- retry 是网络健壮性
- compact 是上下文管理
- fallback 是模型可用性策略

但从源码视角看，它们其实是一类东西：

**恢复机制。**

Claude Code 在运行中不断面对几种失败：

1. 上下文太长
2. 输出 token 不够
3. 流式请求中断
4. 429 / 529 / 网络瞬断
5. 权限拒绝

它的做法不是把失败原样暴露给用户，而是先尝试恢复：

- prompt-too-long -> context collapse drain / reactive compact
- max_output_tokens -> 提额重试 / 注入续写 meta message
- streaming failure -> fallback 到 non-streaming
- 429 / 529 -> withRetry 差异化重试
- stale connection -> 断开 keep-alive 后重建 client

这里的关键思想是：

**Claude Code 不把异常当作边缘路径，而是把恢复路径当作主设计的一部分。**

这就是 Harness 思维和普通脚本式 agent 的差别。

---

## 11. 第十张图：四类关键状态变化

如果要把全场分享收成一张“状态流总表”，最适合的是这四类：

| 状态项 | 作用对象 | 是否进入消息历史 | 作用 |
| --- | --- | --- | --- |
| `tool_result` | 模型 | 是 | 把工具结果回注给下一轮模型 |
| `contextModifier` | Harness 运行时 | 否 | 更新后续工具执行上下文 |
| `compact_boundary` | 上下文治理 / SDK / 引擎状态 | 是 | 标记新的 active context 起点 |
| `result` | SDK / 外部调用方 | 否 | 交付本轮最终结果 |

这张表非常适合用来收束一个核心观点：

**Claude Code 的状态不是单通道流动的，而是按作用对象被分流到不同载体中。**

换句话说，它不是“把所有信息都塞进消息历史”，而是：

- 对模型有意义的，进消息流
- 对工具层有意义的，进运行时上下文
- 对上下文治理有意义的，进结构边界
- 对外部调用方有意义的，进最终结果

这就是典型的 Harness 设计，不是单纯 prompt 设计。

---

## 12. 最后的设计哲学：Claude Code 为什么长成这样

到这里其实可以把全场收敛成四条设计哲学。

### 哲学一：LLM 不是控制器，Harness 才是控制器

模型可以生成：

- text
- thinking
- tool_use

但真正决定系统怎么运行的是 Harness：

- 工具是否执行
- 上下文如何更新
- 是否需要 compact
- 是否需要 retry
- 是否需要 fallback

所以 Claude Code 的控制权不在模型，而在外层状态机。

### 哲学二：所有副作用都必须被驯化为显式状态变化

Claude Code 不喜欢“隐式发生的事情”。

它会把副作用显式化成：

- `tool_result`
- `contextModifier`
- `compact_boundary`
- `result`

因为只有变成显式状态，才能被记录、恢复、审计和重放。

### 哲学三：恢复不是补丁，而是主流程的一部分

普通 agent 常常把失败路径视为兜底。  
Claude Code 则从一开始就假设：

- 模型会超限
- 网络会抖
- streaming 会挂
- 工具会失败
- 用户会拒绝权限

所以恢复机制不是 later fix，而是 first-class design。

### 哲学四：不要把所有复杂性都压给模型

Claude Code 的工程取向非常明确：

**能在 Harness 层确定的事情，不交给模型拍脑袋。**

比如：

- 权限由外层判
- 并发由调度器判
- compact 由预算与阈值判
- retry / fallback 由策略层判

模型负责生成候选动作，系统负责决定这些动作是否以及如何落地。

---

## 13. 结尾：一句话收束全场

如果要用一句话结束今天这场分享，我会这样说：

**Claude Code 的真正核心，不是“模型会调用工具”，而是“它用一个分层 Harness，把模型输出驯化成了一个可持续、可恢复、可审计的执行系统”。**

再压缩成工程语言就是：

**外层管会话，中层管循环，底层分别治理模型流、工具副作用和上下文恢复。模型给出意图，Harness 保证系统。**
