# OpenClaw 工具执行流程与并行支持结论

## 结论

OpenClaw 的工具执行支持并行，但并行调度不在 OpenClaw 自己实现，而在嵌入的 `@mariozechner/pi-agent-core` / `@mariozechner/pi-coding-agent` agent loop 中实现。

OpenClaw 负责：

- 组装内置工具、OpenClaw 工具、channel 工具、plugin 工具。
- 按配置、provider、agent、group、sandbox、subagent、owner 权限过滤工具。
- 包装 before-tool-call hook、AbortSignal、schema normalization。
- 把工具转换成 pi SDK 的 `ToolDefinition` 后作为 `customTools` 传给 `createAgentSession()`。

实际同一轮模型返回多个 tool call 时，由 pi SDK agent loop 负责调度执行。OpenClaw 本仓库未发现把 pi 的工具执行模式改为串行的配置，也没有实现自己的多工具执行调度器。

因此判断：

- 服务端执行的 OpenClaw 工具：支持并行，依赖 pi SDK 默认行为。
- OpenClaw 自身：不是并行调度器，只提供工具定义和 `execute` 回调。
- client hosted tools：OpenClaw 服务端不真正执行，只返回 pending 并委托客户端执行。

## 依赖版本

OpenClaw 当前使用的 pi 依赖版本：

- `@mariozechner/pi-agent-core`: `0.71.1`，见 `package.json:1659`。
- `@mariozechner/pi-coding-agent`: `0.71.1`，见 `package.json:1661`。

## 工具执行流程

### 1. 构建原始工具列表

入口在 agent run attempt：

- `src/agents/pi-embedded-runner/run/attempt.ts:861` 到 `src/agents/pi-embedded-runner/run/attempt.ts:935`：根据当前 run 参数调用 `createOpenClawCodingTools()`，再应用 `toolsAllow`。
- `src/agents/pi-tools.ts:265`：`createOpenClawCodingTools()` 是 OpenClaw 工具组装主函数。

工具来源包括：

- pi 基础 coding tools：`src/agents/pi-tools.ts:500` 调 `createCodingTools(workspaceRoot)`。
- 自定义 `read/write/edit` 与 sandbox 版本：`src/agents/pi-tools.ts:501` 到 `src/agents/pi-tools.ts:543`。
- `exec` / `process`：`src/agents/pi-tools.ts:547` 到 `src/agents/pi-tools.ts:593`。
- `apply_patch`：`src/agents/pi-tools.ts:594` 到 `src/agents/pi-tools.ts:604`。
- channel tools：`src/agents/pi-tools.ts:679` 到 `src/agents/pi-tools.ts:680`。
- OpenClaw 自有工具：`src/agents/pi-tools.ts:681` 到 `src/agents/pi-tools.ts:727`。
- plugin tools：`src/agents/pi-tools.ts:620` 到 `src/agents/pi-tools.ts:649`。

### 2. 策略过滤、schema 处理和执行包装

工具列表组装后继续经过多层处理：

- 消息 provider 过滤：`src/agents/pi-tools.ts:752` 到 `src/agents/pi-tools.ts:755`。
- 模型/provider 策略过滤：`src/agents/pi-tools.ts:757` 到 `src/agents/pi-tools.ts:765`。
- owner-only 权限过滤：`src/agents/pi-tools.ts:767` 到 `src/agents/pi-tools.ts:773`。
- profile/global/agent/group/sandbox/subagent policy pipeline：`src/agents/pi-tools.ts:774` 到 `src/agents/pi-tools.ts:797`。
- schema normalization：`src/agents/pi-tools.ts:798` 到 `src/agents/pi-tools.ts:808`。
- before-tool-call hook 包装：`src/agents/pi-tools.ts:809` 到 `src/agents/pi-tools.ts:818`。
- AbortSignal 包装：`src/agents/pi-tools.ts:820` 到 `src/agents/pi-tools.ts:823`；具体包装实现见 `src/agents/pi-tools.abort.ts:48` 到 `src/agents/pi-tools.abort.ts:72`。

### 3. runtime/provider 层再次规范化工具

在 attempt 中，工具会按 runtime plan 或 provider runtime 再规范化：

- `src/agents/pi-embedded-runner/run/attempt.ts:1041` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1051`：调用 `normalizeAgentRuntimeTools()`。
- `src/agents/runtime-plan/tools.ts:35` 到 `src/agents/runtime-plan/tools.ts:52`：如果有 runtime plan，则走 `runtimePlan.tools.normalize()`；否则走 `normalizeProviderToolSchemas()`。

bundle MCP / LSP 工具会在后续加入：

- `src/agents/pi-embedded-runner/run/attempt.ts:1058` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1074`：materialize bundle MCP tools。
- `src/agents/pi-embedded-runner/run/attempt.ts:1075` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1086`：创建 bundle LSP runtime。
- `src/agents/pi-embedded-runner/run/attempt.ts:1087` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1109`：应用最终工具策略并合并到 `effectiveTools`。

### 4. 转换为 pi SDK `ToolDefinition`

OpenClaw 不直接把自己的 `AgentTool` 交给 pi，而是适配成 pi-coding-agent 的 `ToolDefinition`：

- `src/agents/pi-embedded-runner/tool-split.ts:4` 到 `src/agents/pi-embedded-runner/tool-split.ts:5`：注释说明 OpenClaw 始终通过 `customTools` 传工具。
- `src/agents/pi-embedded-runner/tool-split.ts:8` 到 `src/agents/pi-embedded-runner/tool-split.ts:15`：`splitSdkTools()` 返回 `customTools: toToolDefinitions(tools)`。
- `src/agents/pi-tool-definition-adapter.ts:220` 到 `src/agents/pi-tool-definition-adapter.ts:283`：`toToolDefinitions()` 为每个工具生成 `execute` 回调。

执行回调内部流程：

- `src/agents/pi-tool-definition-adapter.ts:228`：定义 `execute`。
- `src/agents/pi-tool-definition-adapter.ts:233` 到 `src/agents/pi-tool-definition-adapter.ts:247`：运行 before-tool-call hook，并允许 hook 修改参数或阻止调用。
- `src/agents/pi-tool-definition-adapter.ts:249`：调用真实 OpenClaw tool 的 `tool.execute(toolCallId, executeParams, signal, onUpdate)`。
- `src/agents/pi-tool-definition-adapter.ts:250` 到 `src/agents/pi-tool-definition-adapter.ts:254`：规范化工具结果。
- `src/agents/pi-tool-definition-adapter.ts:255` 到 `src/agents/pi-tool-definition-adapter.ts:279`：捕获错误并转换成工具错误结果。

### 5. 创建 pi session 并交给 SDK 执行 agent loop

OpenClaw 创建 session 时把工具传给 pi SDK：

- `src/agents/pi-embedded-runner/run/attempt.ts:1503` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1506`：从 `effectiveTools` 生成 `customTools`。
- `src/agents/pi-embedded-runner/run/attempt.ts:1545` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1559`：把 client hosted tools 也转换为 `ToolDefinition`。
- `src/agents/pi-embedded-runner/run/attempt.ts:1561`：合并为 `allCustomTools`。
- `src/agents/pi-embedded-runner/run/attempt.ts:1565` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1567`：生成 session tool allowlist。
- `src/agents/pi-embedded-runner/run/attempt.ts:1569` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1587`：创建 pi session。
- `src/agents/pi-embedded-runner/run/attempt.ts:1581`：传 `tools: sessionToolAllowlist`。
- `src/agents/pi-embedded-runner/run/attempt.ts:1582`：传 `customTools: allCustomTools`。

文档也说明 agent loop 由 SDK 负责：

- `docs/pi.md:230` 到 `docs/pi.md:236`：`session.prompt(...)` 后，SDK 处理发送给 LLM、执行工具调用、流式响应。
- `docs/pi.md:254` 到 `docs/pi.md:286`：说明 `AgentTool` 到 `ToolDefinition` 的适配，以及所有工具通过 `customTools` 传入。

## 并行支持分析

### OpenClaw 仓库内证据

OpenClaw 自身没有实现“多个 tool call 的执行循环”。它只实现每个工具的 `execute` 回调，并把工具交给 pi SDK：

- `src/agents/pi-tool-definition-adapter.ts:228` 到 `src/agents/pi-tool-definition-adapter.ts:280`：这里只定义单个工具调用如何执行、如何处理 hook/错误/结果。
- `src/agents/pi-embedded-runner/run/attempt.ts:1569` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1587`：创建 pi session 并传入工具。

仓库内也未发现 OpenClaw 设置 pi 的工具执行模式：

- 搜索 `executionMode` 的结果主要是 hook/context-engine/外部 action，与 agent tool call 调度无关。
- 搜索 `toolExecution` 只命中监控、测试和 tool result 命名，不是 pi agent loop 的执行模式配置。

这意味着是否并行由 pi SDK 默认 agent loop 行为决定。

### 上游 pi SDK 行为

上游 `badlogic/pi-mono` 的 `@mariozechner/pi-agent-core` 支持工具执行模式：

- `toolExecution` 类型支持 `"parallel" | "sequential"`。
- 默认值是 `"parallel"`。
- parallel 模式下，tool calls 会先顺序准备，然后并发执行。

对应上游源码索引位置：

- `packages/agent/src/agent.ts:203`
- `packages/agent/src/agent-loop.ts:346` 到 `packages/agent/src/agent-loop.ts:351`

参考索引：https://deepwiki.com/badlogic/pi-mono/3-%40mariozechnerpi-agent-core

结合 OpenClaw 没有覆盖 `toolExecution` 配置，OpenClaw 当前服务端工具执行应采用 pi SDK 默认的 parallel 模式。

## extra-params.ts 中 parallel_tool_calls 的作用

`src/agents/pi-embedded-runner/extra-params.ts` 里的 `parallel_tool_calls` 是传给 OpenAI/GPT 兼容 API 的请求 payload 参数。它控制的是“模型是否允许在一次响应中生成多个 tool calls”，不是 OpenClaw 本地的工具执行调度开关。

这和前面的 pi SDK 并行执行是两层概念：

- `parallel_tool_calls`：provider/model 请求参数，影响模型是否可以一次返回多个 tool calls。
- pi SDK `toolExecution`：agent loop 收到多个 tool calls 后，如何执行这些工具调用。

### 参数来源与归一化

OpenClaw 支持 snake case 和 camel case 两种配置写法：

- `parallel_tool_calls`
- `parallelToolCalls`

归一化逻辑：

- `src/agents/pi-embedded-runner/extra-params.ts:86` 到 `src/agents/pi-embedded-runner/extra-params.ts:95`：`resolveExtraParams()` 从 defaults、model config、agent params 中读取别名，并统一写回 `merged.parallel_tool_calls`，删除 `parallelToolCalls`。
- `src/agents/pi-embedded-runner/extra-params.ts:357` 到 `src/agents/pi-embedded-runner/extra-params.ts:377`：`resolveAliasedParamValue()` 按来源顺序解析 snake/camel alias，后面的来源覆盖前面的来源。

### GPT-5 默认注入

对 OpenAI GPT-5 系列，OpenClaw 默认启用 `parallel_tool_calls`：

- `src/agents/pi-embedded-runner/extra-params.ts:226` 到 `src/agents/pi-embedded-runner/extra-params.ts:234`：只有 provider 是 `openai` 或 `openai-codex`，且 model id 匹配 `gpt-5...`，才应用默认 GPT runtime params。
- `src/agents/pi-embedded-runner/extra-params.ts:236` 到 `src/agents/pi-embedded-runner/extra-params.ts:248`：如果用户没有显式设置 `parallel_tool_calls` 或 `parallelToolCalls`，则默认写入 `merged.parallel_tool_calls = true`。

对应测试：

- `src/agents/pi-embedded-runner-extraparams.test.ts:1903` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1917`：OpenAI Responses payload 默认带 `parallel_tool_calls: true`。
- `src/agents/pi-embedded-runner-extraparams.test.ts:1919` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1932`：OpenAI Codex Responses payload 默认带 `parallel_tool_calls: true`。

### 请求 payload 注入位置

`parallel_tool_calls` 最终不是作为 OpenClaw 内部状态使用，而是通过 streamFn wrapper 写入 provider request payload：

- `src/agents/pi-embedded-runner/extra-params.ts:379` 到 `src/agents/pi-embedded-runner/extra-params.ts:394`：`createParallelToolCallsWrapper()` 包装 `streamFn`，如果目标 API 支持，就通过 `streamWithPayloadPatch()` 写入 `payloadObj.parallel_tool_calls = enabled`。
- `src/agents/pi-embedded-runner/extra-params.ts:601` 到 `src/agents/pi-embedded-runner/extra-params.ts:611`：从 effective extra params / override 中读取 `parallel_tool_calls` 或 `parallelToolCalls`，值为 boolean 时安装 wrapper。
- `src/agents/pi-embedded-runner/extra-params.ts:613` 到 `src/agents/pi-embedded-runner/extra-params.ts:619`：值为 `null` 时抑制注入；其他非 boolean 值会 warn 并跳过。

请求参数应用到 agent 的位置：

- `src/agents/pi-embedded-runner/run/attempt.ts:1851` 到 `src/agents/pi-embedded-runner/run/attempt.ts:1866`：run attempt 调用 `applyExtraParamsToAgent()`，把配置、stream params、runtime plan 准备好的 extra params 应用到当前 `activeSession.agent`。

### 支持的 provider API 范围

OpenClaw 只向 GPT/OpenAI 兼容的少数 API 注入该字段：

- `src/agents/provider-api-families.ts:1` 到 `src/agents/provider-api-families.ts:6`：支持 `openai-completions`、`openai-responses`、`openai-codex-responses`、`azure-openai-responses`。
- `src/agents/provider-api-families.ts:8` 到 `src/agents/provider-api-families.ts:10`：`supportsGptParallelToolCallsPayload()` 判断目标 `model.api` 是否支持该 payload patch。

对应测试：

- `src/agents/pi-embedded-runner-extraparams.test.ts:1101` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1127`：显式配置后注入到 `openai-responses`。
- `src/agents/pi-embedded-runner-extraparams.test.ts:1129` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1155`：显式配置后注入到 `openai-codex-responses`。
- `src/agents/pi-embedded-runner-extraparams.test.ts:1185` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1211`：显式配置后注入到 `azure-openai-responses`。
- `src/agents/pi-embedded-runner-extraparams.test.ts:1213` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1237`：不支持的 API，例如 `anthropic-messages`，不会注入 `parallel_tool_calls`。

### 覆盖规则

`parallel_tool_calls` 可以被运行时覆盖：

- `src/agents/pi-embedded-runner-extraparams.test.ts:1240` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1267`：runtime override 的 `parallelToolCalls: false` 会覆盖配置中的 `parallel_tool_calls: true`，最终 payload 是 `false`。
- `src/agents/pi-embedded-runner-extraparams.test.ts:1270` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1297`：runtime override 的 `parallelToolCalls: null` 会抑制继承来的注入，最终 payload 不包含 `parallel_tool_calls`。
- `src/agents/pi-embedded-runner-extraparams.test.ts:1300` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:1328`：非法值如字符串 `"false"` 会被忽略，并记录 warning。
- `src/agents/pi-embedded-runner-extraparams.test.ts:2302` 到 `src/agents/pi-embedded-runner-extraparams.test.ts:2324`：transport hook 返回的 patch 也可以把 `parallel_tool_calls` 注入到请求 payload。

### 实际效果

实际语义：

- `true`：告诉 OpenAI/GPT 兼容 provider，模型可以在同一响应中产生多个 tool calls。
- `false`：告诉 provider 不要并行 tool calls，通常限制模型一次只发起一个工具调用序列。
- `null`：OpenClaw 内部约定，表示不要注入该字段。

重要边界：

- 它不会直接让 OpenClaw 并行执行工具。
- 它也不会改变 `session-tool-result-state.ts` 的 pending toolResult 配对逻辑。
- 它只改变 provider 请求 payload；provider 是否遵守、如何返回多个 tool calls，取决于 provider API 和模型行为。

## 事件处理不是执行调度

OpenClaw 有 tool execution event 订阅处理，但这是观察、展示、记录层，不是 executor：

- `src/agents/pi-embedded-subscribe.handlers.ts:23` 到 `src/agents/pi-embedded-subscribe.handlers.ts:73`：事件处理器通过 `pendingEventChain` 排队处理部分事件。
- `src/agents/pi-embedded-subscribe.handlers.ts:92` 到 `src/agents/pi-embedded-subscribe.handlers.ts:109`：接收 `tool_execution_start`、`tool_execution_update`、`tool_execution_end` 事件。
- `src/agents/pi-embedded-subscribe.handlers.tools.ts:640` 到 `src/agents/pi-embedded-subscribe.handlers.tools.ts:728`：处理工具开始事件，发出 UI/agent event，记录 pending message send。
- `src/agents/pi-embedded-subscribe.handlers.tools.ts:809` 到 `src/agents/pi-embedded-subscribe.handlers.tools.ts:880`：处理工具结束事件，更新状态和结果。

这里的事件顺序控制只影响订阅处理和状态更新，不决定工具是否串行执行。

## session-tool-result-state.ts 的作用

`src/agents/session-tool-result-state.ts` 是工具调用流程里的 transcript 持久化状态机。它不执行工具，也不调度并行；它只记录“assistant 已经写入 session 的 tool call 中，哪些还没有 matching `toolResult`”。

核心状态是一个 `Map<toolCallId, toolName>`：

- `src/agents/session-tool-result-state.ts:16` 到 `src/agents/session-tool-result-state.ts:18`：`createPendingToolCallState()` 创建 pending map。
- `src/agents/session-tool-result-state.ts:29` 到 `src/agents/session-tool-result-state.ts:33`：`trackToolCalls()` 记录 assistant message 里的 tool calls。
- `src/agents/session-tool-result-state.ts:34`：`getPendingIds()` 用于调试、测试和外部清理检查。
- `src/agents/session-tool-result-state.ts:35` 到 `src/agents/session-tool-result-state.ts:38`：决定何时需要 flush pending tool calls。

它被 `installSessionToolResultGuard()` 使用：

- `src/agents/session-tool-result-guard.ts:268` 到 `src/agents/session-tool-result-guard.ts:313`：安装 guard，保存原始 `appendMessage`，创建 `pendingState`。
- `src/agents/session-tool-result-guard.ts:474` 到 `src/agents/session-tool-result-guard.ts:475`：monkey-patch `sessionManager.appendMessage`，所有 transcript 写入都先经过 guard。

### assistant tool call 写入时：登记 pending

当写入 assistant message 时，guard 会提取其中的 tool calls：

- `src/agents/session-tool-result-guard.ts:423` 到 `src/agents/session-tool-result-guard.ts:433`：只有非 `aborted` / 非 `error` 的 assistant message 才提取 tool calls。
- `src/agents/session-tool-result-guard.ts:467` 到 `src/agents/session-tool-result-guard.ts:469`：如果有 tool calls，则调用 `pendingState.trackToolCalls(toolCalls)`。

这里的目的不是执行工具，而是让后续持久化层知道 transcript 中已经有 assistant tool call，需要等待对应的 `toolResult`。

### toolResult 写入时：匹配并消账

当写入 `toolResult` 时：

- `src/agents/session-tool-result-guard.ts:397` 到 `src/agents/session-tool-result-guard.ts:403`：提取 `toolResult` id，从 pending map 里读取 toolName，然后删除对应 pending id。
- `src/agents/session-tool-result-guard.ts:238` 到 `src/agents/session-tool-result-guard.ts:264`：如果 `toolResult.toolName` 为空，会用 pending 里的 toolName 回填。
- `src/agents/session-tool-result-guard.ts:404` 到 `src/agents/session-tool-result-guard.ts:420`：对 tool result 做持久化前处理，包括大小截断、hook、details cap，然后写入原始 session。

对应测试：

- `src/agents/session-tool-result-guard.test.ts:156` 到 `src/agents/session-tool-result-guard.test.ts:170`：matching `toolResult` 到达时不会插入 synthetic result。
- `src/agents/session-tool-result-guard.test.ts:197` 起：空 toolName 会从 pending tool call 回填。

### toolResult 缺失时：生成 synthetic result 或清理 pending

如果 assistant tool call 已写入，但后续没有 matching `toolResult`，下一条非 toolResult message 到来前会触发 flush：

- `src/agents/session-tool-result-state.ts:35`：assistant message 被 sanitize/drop 时，只要有 pending 就需要 flush。
- `src/agents/session-tool-result-state.ts:36` 到 `src/agents/session-tool-result-state.ts:37`：非 toolResult 写入前，如果仍有 pending，就需要 flush。
- `src/agents/session-tool-result-state.ts:38`：新 tool calls 到来但旧 pending 仍存在时，先 flush 旧 pending。
- `src/agents/session-tool-result-guard.ts:350` 到 `src/agents/session-tool-result-guard.ts:374`：`flushPendingToolResults()` 默认会为每个 pending id 写入 synthetic error `toolResult`，然后清空 pending map。
- `src/agents/session-tool-result-guard.ts:435` 到 `src/agents/session-tool-result-guard.ts:447`：在写入非 toolResult 或新 tool calls 前检查是否需要 flush。

对应测试：

- `src/agents/session-tool-result-guard.test.ts:80` 到 `src/agents/session-tool-result-guard.test.ts:102`：pending tool call 后写入非 tool message，会插入 synthetic `toolResult`。
- `src/agents/session-tool-result-guard.test.ts:104` 到 `src/agents/session-tool-result-guard.test.ts:123`：可以显式 flush，且可自定义 synthetic result 文本。
- `src/agents/session-tool-result-guard.test.ts:137` 到 `src/agents/session-tool-result-guard.test.ts:153`：禁用 synthetic results 时，用户中断会清空 pending。
- `src/agents/session-tool-result-guard.test.ts:369` 到 `src/agents/session-tool-result-guard.test.ts:390`：禁用 synthetic results 时，新 tool calls 会替换旧 pending ids。

### 与工具执行并行的关系

`session-tool-result-state.ts` 不参与并行执行决策。并行执行发生在 pi SDK agent loop；该文件只在 `sessionManager.appendMessage` 的持久化边界工作。

它对并行工具调用的价值是：同一 assistant turn 可能产生多个 tool calls，pending map 可以同时记录多个 id。每个 `toolResult` 到达时按 id 消账；如果某个 id 永远没有结果，则在 transcript 边界 flush synthetic result 或清空 pending，避免后续 provider replay 出现 orphaned `tool_use` / 缺失 `tool_result` 造成 API 400。

## client hosted tools 例外

client tools 被转换成 `ToolDefinition`，但服务端不会真实执行：

- `src/agents/pi-tool-definition-adapter.ts:317` 到 `src/agents/pi-tool-definition-adapter.ts:318`：注释说明 client tools 被拦截，返回 pending。
- `src/agents/pi-tool-definition-adapter.ts:331` 到 `src/agents/pi-tool-definition-adapter.ts:365`：执行 before hook 后，返回 `status: "pending"` 和 `message: "Tool execution delegated to client"`。
- `src/agents/pi-embedded-runner/types.ts:147` 到 `src/agents/pi-embedded-runner/types.ts:154`：run meta 中可返回 `stopReason: "tool_calls"` 和 `pendingToolCalls`。

因此 client hosted tools 的并行性取决于客户端如何消费 `pendingToolCalls`，不是 OpenClaw 服务端工具执行并行性的证据。

## 最终判断

OpenClaw 服务端工具执行链路支持并行。OpenClaw 不自行调度多个工具，而是把所有可执行工具作为 `customTools` 交给 pi SDK；当前未覆盖 pi 的默认 `toolExecution`，因此沿用 pi SDK 默认 parallel 模式。同一 assistant turn 中多个工具调用可以并发执行；单个工具调用内部仍按 OpenClaw wrapper 的 before hook、真实 `tool.execute`、结果规范化、错误处理顺序执行。
