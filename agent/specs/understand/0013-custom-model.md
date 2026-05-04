# Claude-code-open 自定义模型流程分析

本文基于 `vendor/Claude-code-open` 源码阅读，梳理 Claude Code Open 如何选择自定义模型、如何映射到不同 API Provider，以及 SDK/headless 模式如何把模型参数传入实际 API 调用。

> 说明：当前环境没有暴露可用的 GitNexus MCP 资源或 `gitnexus_*` 工具。本次只新增理解文档，没有修改项目代码符号。

## 结论概览

`Claude-code-open` 的自定义模型能力不是独立的 Provider 插件机制，而是贯穿在三层中：

1. 模型选择层：解析 `/model`、SDK control、`--model`、`ANTHROPIC_MODEL`、settings。
2. 模型映射层：解析别名、默认族模型、三方 provider 模型 ID、`modelOverrides`。
3. API 调用层：通过 `@anthropic-ai/sdk` 或三方 Anthropic SDK wrapper 发起 `beta.messages.create()` 请求。

最直接的接入方式：

```bash
export ANTHROPIC_API_KEY=sk-...
export ANTHROPIC_BASE_URL=https://your-compatible-endpoint.example.com
export ANTHROPIC_MODEL=your-custom-model

claude -p "hello"
```

或显式指定：

```bash
claude -p "hello" --model your-custom-model
```

SDK 使用时，核心也是把 `model` 传给 SDK options，或通过环境变量继承给 SDK 子进程：

```ts
import { query } from '@anthropic-ai/claude-code'

process.env.ANTHROPIC_API_KEY = 'sk-...'
process.env.ANTHROPIC_BASE_URL = 'https://your-compatible-endpoint.example.com'

for await (const message of query({
  prompt: 'hello',
  options: {
    model: 'your-custom-model',
  },
})) {
  console.log(message)
}
```

## 关键源码位置

| 文件 | 作用 |
| --- | --- |
| `src/utils/model/model.ts` | 模型优先级、默认模型、别名解析、自定义模型透传 |
| `src/utils/model/modelStrings.ts` | provider 模型 ID 初始化、settings `modelOverrides` 覆盖 |
| `src/utils/model/configs.ts` | 内置 Claude 模型在 firstParty/Bedrock/Vertex/Foundry 的 ID 映射 |
| `src/utils/model/providers.ts` | API provider 选择 |
| `src/utils/model/modelAllowlist.ts` | `availableModels` allowlist 校验 |
| `src/utils/model/modelOptions.ts` | `/model` 选择器模型列表和自定义模型展示 |
| `src/utils/model/validateModel.ts` | 通过一次最小 API 调用验证模型是否可用 |
| `src/main.tsx` | CLI 参数解析，处理 `--model`、`--fallback-model` |
| `src/cli/print.ts` | headless/SDK 模式主循环，处理 SDK control `set_model` |
| `src/QueryEngine.ts` | SDK/headless query 生命周期，解析 `userSpecifiedModel` |
| `src/query.ts` | agentic query loop，把当前模型传给 API 层 |
| `src/services/api/client.ts` | 创建 Anthropic/Bedrock/Vertex/Foundry client |
| `src/services/api/claude.ts` | 构造 `beta.messages.create()` 请求并发送 |
| `src/entrypoints/agentSdkTypes.ts` | SDK public type facade |
| `src/entrypoints/sdk/controlSchemas.ts` | SDK control protocol schema，如 `initialize`、`set_model` |

## 模型选择优先级

核心逻辑在 `src/utils/model/model.ts`。

`getUserSpecifiedModelSetting()` 的优先级：

1. 会话内覆盖：`getMainLoopModelOverride()`，包括 `/model`、SDK control `set_model`、配置变更等。
2. 启动覆盖：`--model` 最终也会写入 `setMainLoopModelOverride()`。
3. 环境变量：`ANTHROPIC_MODEL`。
4. settings：用户或项目配置里的 `model`。

`getMainLoopModel()` 在没有显式模型时，会回退到 `getDefaultMainLoopModel()`。

```ts
export function getMainLoopModel(): ModelName {
  const model = getUserSpecifiedModelSetting()
  if (model !== undefined && model !== null) {
    return parseUserSpecifiedModel(model)
  }
  return getDefaultMainLoopModel()
}
```

### 自定义模型为什么能工作

`parseUserSpecifiedModel()` 只特殊处理内置 alias：

- `sonnet`
- `opus`
- `haiku`
- `best`
- `sonnet[1m]`
- `opus[1m]`
- `opusplan`

如果传入不是这些 alias，也不是需要特殊 remap 的 legacy Opus，函数会保留原始字符串并返回。

源码注释明确说明：

```ts
// Preserve original case for custom model names (e.g., Azure Foundry deployment IDs)
```

这意味着 `your-custom-model`、Azure Foundry deployment ID、Bedrock inference profile ARN、代理服务模型名都可以作为字符串透传到 API 调用层。

### `[1m]` 后缀

`[1m]` 是客户端层面的上下文窗口标记。发 API 前会调用：

```ts
export function normalizeModelStringForAPI(model: string): string {
  return model.replace(/\[(1|2)m\]/gi, '')
}
```

所以 `claude-sonnet-4-6[1m]` 最终请求里的 `model` 是 `claude-sonnet-4-6`，上下文能力通过 betas/配置控制，而不是模型名后缀直接发给 API。

## 默认模型与环境变量覆盖

默认模型函数在 `model.ts`：

- `getDefaultOpusModel()`
- `getDefaultSonnetModel()`
- `getDefaultHaikuModel()`

可通过环境变量覆盖：

```bash
export ANTHROPIC_DEFAULT_OPUS_MODEL=your-opus-like-model
export ANTHROPIC_DEFAULT_SONNET_MODEL=your-sonnet-like-model
export ANTHROPIC_DEFAULT_HAIKU_MODEL=your-haiku-like-model
```

对应 `/model` 选择器展示文案也支持：

```bash
export ANTHROPIC_DEFAULT_SONNET_MODEL_NAME="My Sonnet"
export ANTHROPIC_DEFAULT_SONNET_MODEL_DESCRIPTION="Custom Sonnet model"
```

如果只想额外展示一个自定义模型选项，可用：

```bash
export ANTHROPIC_CUSTOM_MODEL_OPTION=your-custom-model
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="My Model"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="Custom proxy model"
```

这部分在 `src/utils/model/modelOptions.ts` 中处理。

## Settings 配置

settings schema 在 `src/utils/settings/types.ts`，与模型相关的配置包括：

```json
{
  "model": "your-custom-model",
  "availableModels": ["sonnet", "opus", "your-custom-model"],
  "modelOverrides": {
    "claude-sonnet-4-6": "your-provider-specific-model-id-or-arn"
  }
}
```

### `model`

覆盖默认模型。可放 full model ID，也可放 alias。

### `availableModels`

企业或托管环境可限制用户能选的模型。

支持：

- family alias：`sonnet`、`opus`、`haiku`
- version prefix：`opus-4-5`、`claude-opus-4-5`
- full model ID：`claude-opus-4-5-20251101`
- 自定义模型 ID：如 `your-custom-model`

如果设置了 allowlist，但没有包含自定义模型，`getUserSpecifiedModelSetting()` 会忽略该模型，`validateModel()` 也会返回不可用。

### `modelOverrides`

用于把 canonical first-party model ID 映射为 provider-specific model ID。

例如：

```json
{
  "modelOverrides": {
    "claude-sonnet-4-6": "arn:aws:bedrock:us-east-1:123456789012:inference-profile/my-sonnet"
  }
}
```

这适合 Bedrock inference profile、自建代理或企业托管环境中“用户选择 Claude 语义模型，但实际调用私有部署 ID”的场景。

## API Provider 选择

Provider 选择在 `src/utils/model/providers.ts`：

```ts
export type APIProvider = 'firstParty' | 'bedrock' | 'vertex' | 'foundry'
```

环境变量优先级：

1. `CLAUDE_CODE_USE_BEDROCK`
2. `CLAUDE_CODE_USE_VERTEX`
3. `CLAUDE_CODE_USE_FOUNDRY`
4. 默认 `firstParty`

示例：

```bash
export CLAUDE_CODE_USE_BEDROCK=1
export AWS_REGION=us-east-1
```

```bash
export CLAUDE_CODE_USE_VERTEX=1
export ANTHROPIC_VERTEX_PROJECT_ID=your-project-id
export CLOUD_ML_REGION=us-central1
```

```bash
export CLAUDE_CODE_USE_FOUNDRY=1
export ANTHROPIC_FOUNDRY_API_KEY=...
```

`src/utils/model/configs.ts` 定义了同一个 Claude 模型在不同 provider 下的 ID：

```ts
export const CLAUDE_SONNET_4_6_CONFIG = {
  firstParty: 'claude-sonnet-4-6',
  bedrock: 'us.anthropic.claude-sonnet-4-6',
  vertex: 'claude-sonnet-4-6',
  foundry: 'claude-sonnet-4-6',
}
```

`src/utils/model/modelStrings.ts` 会根据当前 provider 初始化模型字符串，并叠加 settings `modelOverrides`。

## API Client 创建

核心在 `src/services/api/client.ts` 的 `getAnthropicClient()`。

不同 provider 使用不同 SDK：

- firstParty：`new Anthropic(...)`
- Bedrock：`new AnthropicBedrock(...)`
- Vertex：`new AnthropicVertex(...)`
- Foundry：`new AnthropicFoundry(...)`

firstParty client 主要使用：

```ts
const clientConfig = {
  apiKey: isClaudeAISubscriber() ? null : apiKey || getAnthropicApiKey(),
  authToken: isClaudeAISubscriber()
    ? getClaudeAIOAuthTokens()?.accessToken
    : undefined,
  defaultHeaders,
  maxRetries,
  timeout,
  fetchOptions,
}
```

`ANTHROPIC_BASE_URL` 在仓库内被大量用于判断是否是 first-party Anthropic host，比如是否启用某些 first-party-only beta。`getAnthropicClient()` firstParty 分支没有显式写：

```ts
baseURL: process.env.ANTHROPIC_BASE_URL
```

因此这里应理解为依赖底层 `@anthropic-ai/sdk` 对 `ANTHROPIC_BASE_URL` 的环境变量支持，或由运行环境提前注入。README 也明确给出：

```bash
export ANTHROPIC_BASE_URL=https://your-proxy.example.com
export ANTHROPIC_MODEL=claude-opus-4-6
```

## 实际 API 请求

API 请求构造在 `src/services/api/claude.ts`。

流式主路径：

```ts
const result = await anthropic.beta.messages
  .create(
    { ...params, stream: true },
    { signal, headers }
  )
  .withResponse()
```

`paramsFromContext()` 返回的核心字段：

```ts
{
  model: normalizeModelStringForAPI(options.model),
  messages,
  system,
  tools: allTools,
  tool_choice: options.toolChoice,
  betas: betasParams,
  metadata: getAPIMetadata(),
  max_tokens: maxOutputTokens,
  thinking,
  temperature,
  context_management,
  output_config,
  speed,
}
```

非流式 fallback 也是同一个 API：

```ts
await anthropic.beta.messages.create({
  ...adjustedParams,
  model: normalizeModelStringForAPI(adjustedParams.model),
})
```

因此无论是 CLI、SDK、子 Agent，最终自定义模型都会落实到 `beta.messages.create({ model: ... })`。

## CLI 调用链路

非交互模式示例：

```bash
ANTHROPIC_API_KEY=sk-... \
ANTHROPIC_BASE_URL=https://your-compatible-endpoint.example.com \
bun run src/entrypoints/cli.tsx -p "hello" --model your-custom-model
```

主要调用链：

```text
src/main.tsx
  -> parse CLI options.model
  -> setMainLoopModelOverride(effectiveModel)
  -> runHeadless({ userSpecifiedModel: effectiveModel })

src/cli/print.ts
  -> runHeadless()
  -> runHeadlessStreaming()
  -> ask({ userSpecifiedModel: activeUserSpecifiedModel })

src/QueryEngine.ts
  -> initialMainLoopModel = parseUserSpecifiedModel(userSpecifiedModel)
  -> processUserInput()
  -> query({ toolUseContext.options.mainLoopModel })

src/query.ts
  -> currentModel = toolUseContext.options.mainLoopModel
  -> deps.callModel({ options: { model: currentModel } })

src/services/api/claude.ts
  -> queryModelWithStreaming()
  -> getAnthropicClient({ model: options.model })
  -> anthropic.beta.messages.create({ model: normalizeModelStringForAPI(options.model), ... })
```

## SDK/headless 调用链路

`src/entrypoints/agentSdkTypes.ts` 暴露 SDK 类型和函数签名，但在这个 vendor 源码中，public SDK 函数是 stub：

```ts
export function query(): Query {
  throw new Error('query is not implemented in the SDK')
}
```

这说明该目录包含 CLI/runtime 侧实现和 SDK 类型定义，但没有完整 npm SDK runtime 实现。实际 SDK package 可能在构建产物或另一个包中实现。

不过 runtime 侧已经能看出 SDK 调用如何进入系统：

1. SDK 子进程或 stream-json 进入 `src/cli/print.ts` 的 `runHeadless()`。
2. 初始化阶段通过 SDK control `initialize` 注入系统提示词、agents、hooks、json schema 等。
3. 模型来自启动 options 的 `userSpecifiedModel`，或者后续 control `set_model`。
4. 最终进入 `QueryEngine`，与 CLI 一样调用 `parseUserSpecifiedModel()`。

### SDK 初始化 control

Schema 在 `src/entrypoints/sdk/controlSchemas.ts`：

```ts
export const SDKControlInitializeRequestSchema = z.object({
  subtype: z.literal('initialize'),
  hooks: z.record(...).optional(),
  sdkMcpServers: z.array(z.string()).optional(),
  jsonSchema: z.record(z.string(), z.unknown()).optional(),
  systemPrompt: z.string().optional(),
  appendSystemPrompt: z.string().optional(),
  agents: z.record(z.string(), AgentDefinitionSchema()).optional(),
  promptSuggestions: z.boolean().optional(),
  agentProgressSummaries: z.boolean().optional(),
})
```

初始化响应会返回模型列表：

```ts
{
  commands,
  agents,
  output_style,
  available_output_styles,
  models,
  account,
  pid,
  fast_mode_state,
}
```

`models` 来自 `getModelOptions()`，会包含 `ANTHROPIC_CUSTOM_MODEL_OPTION` 和当前已选自定义模型。

### SDK 动态切模型

Schema：

```ts
export const SDKControlSetModelRequestSchema = z.object({
  subtype: z.literal('set_model'),
  model: z.string().optional(),
})
```

处理逻辑在 `src/cli/print.ts`：

```ts
const requestedModel = message.request.model ?? 'default'
const model =
  requestedModel === 'default'
    ? getDefaultMainLoopModel()
    : requestedModel
activeUserSpecifiedModel = model
setMainLoopModelOverride(model)
notifySessionMetadataChanged({ model })
```

后续 turn 会使用新的 `activeUserSpecifiedModel`。

## SDK 示例

基于 `agentSdkTypes.ts` 暴露的签名，典型 one-shot 调用形态如下：

```ts
import { query } from '@anthropic-ai/claude-code'

for await (const message of query({
  prompt: '分析当前项目',
  options: {
    model: 'your-custom-model',
    maxTurns: 3,
  },
})) {
  console.log(message)
}
```

如果使用自定义 endpoint：

```ts
import { query } from '@anthropic-ai/claude-code'

process.env.ANTHROPIC_API_KEY = 'sk-...'
process.env.ANTHROPIC_BASE_URL = 'https://your-compatible-endpoint.example.com'

for await (const message of query({
  prompt: 'hello',
  options: {
    model: 'your-custom-model',
  },
})) {
  console.log(message)
}
```

如果 SDK wrapper 支持给子进程传 env，应优先显式传：

```ts
{
  env: {
    ANTHROPIC_API_KEY: 'sk-...',
    ANTHROPIC_BASE_URL: 'https://your-compatible-endpoint.example.com',
    ANTHROPIC_MODEL: 'your-custom-model',
  }
}
```

具体字段名需要以实际 SDK runtime package 为准，因为当前 vendor 中 `query()` 只是类型 facade。

## 推荐接入方案

### 方案一：兼容 Anthropic Messages API 的代理

适用：LiteLLM、内部网关、自建兼容 API。

```bash
export ANTHROPIC_API_KEY=sk-...
export ANTHROPIC_BASE_URL=https://proxy.example.com
export ANTHROPIC_MODEL=your-custom-model
```

优点：

- 最少改动。
- CLI 和 SDK 都能通过环境变量继承。
- 自定义模型字符串会原样透传。

注意：

- 某些 first-party-only beta/tool 字段代理可能不支持。代码通过 `isFirstPartyAnthropicBaseUrl()` 对部分能力做了保护，但仍需代理兼容 Anthropic beta messages API。
- 流式接口如果返回 404，代码会尝试 fallback 到非流式请求。

### 方案二：使用 `--model` 或 SDK `options.model`

适用：单次任务或每个 SDK request 使用不同模型。

```bash
claude -p "hello" --model your-custom-model
```

```ts
query({
  prompt: 'hello',
  options: { model: 'your-custom-model' },
})
```

优点：

- 优先级高于 `ANTHROPIC_MODEL` 和 settings。
- 明确、局部、便于测试。

### 方案三：settings `modelOverrides`

适用：企业托管、Bedrock inference profile、Vertex/Foundry deployment、希望用户仍选择 `sonnet`/`opus` 语义模型的场景。

```json
{
  "model": "sonnet",
  "modelOverrides": {
    "claude-sonnet-4-6": "your-provider-specific-model-id-or-arn"
  }
}
```

优点：

- 保留 Claude Code 内部模型族语义。
- `/model` 选择器仍可显示正常模型名。
- 可以集中管理 provider-specific ID。

### 方案四：自定义 `/model` 选项展示

适用：希望用户在 `/model` 中看到自定义模型。

```bash
export ANTHROPIC_CUSTOM_MODEL_OPTION=your-custom-model
export ANTHROPIC_CUSTOM_MODEL_OPTION_NAME="My Model"
export ANTHROPIC_CUSTOM_MODEL_OPTION_DESCRIPTION="Internal model via proxy"
```

## 常见问题

### 1. 自定义模型是否必须在内置列表中？

不需要。只要没有被 `availableModels` 禁止，`parseUserSpecifiedModel()` 会原样返回未知模型字符串。

### 2. `ANTHROPIC_MODEL` 和 `--model` 谁优先？

`--model` 更高。实际实现中 `--model` 会进入 `setMainLoopModelOverride()`，高于 `ANTHROPIC_MODEL`。

### 3. `/model` 能否切换到自定义模型？

可以。`/model` 选择器会自动追加当前已选但不在列表里的 custom model；也可以通过 `ANTHROPIC_CUSTOM_MODEL_OPTION` 主动加入选项。

### 4. SDK 是否能动态切模型？

runtime 支持 SDK control `set_model`。处理后会更新 `activeUserSpecifiedModel` 和 `setMainLoopModelOverride()`，后续 turn 生效。

### 5. 自定义 endpoint 是否一定可用？

需要 endpoint 兼容 Anthropic Messages API，特别是：

- `/v1/messages` 或 SDK 对应 endpoint
- streaming
- beta messages 参数
- tools/tool_choice/tool schema
- thinking/output_config/betas 等扩展字段的容忍或支持

如果代理不支持某些 beta 字段，可能需要在环境或代码层关闭相关能力。

## 数据流摘要

```text
用户配置
  ├─ --model
  ├─ ANTHROPIC_MODEL
  ├─ settings.model
  ├─ SDK options.model
  └─ SDK control set_model

模型解析
  ├─ getUserSpecifiedModelSetting()
  ├─ isModelAllowed()
  ├─ parseUserSpecifiedModel()
  ├─ getModelStrings()
  └─ modelOverrides

Query 执行
  ├─ QueryEngine.submitMessage()
  ├─ query()
  └─ deps.callModel()

API 层
  ├─ queryModelWithStreaming()
  ├─ getAnthropicClient()
  ├─ normalizeModelStringForAPI()
  └─ anthropic.beta.messages.create({ model, ... })
```

## 最小验证命令

CLI：

```bash
ANTHROPIC_API_KEY=sk-... \
ANTHROPIC_BASE_URL=https://your-compatible-endpoint.example.com \
ANTHROPIC_MODEL=your-custom-model \
claude -p "Say hi"
```

显式模型：

```bash
ANTHROPIC_API_KEY=sk-... \
ANTHROPIC_BASE_URL=https://your-compatible-endpoint.example.com \
claude -p "Say hi" --model your-custom-model
```

SDK：

```ts
import { query } from '@anthropic-ai/claude-code'

process.env.ANTHROPIC_API_KEY = 'sk-...'
process.env.ANTHROPIC_BASE_URL = 'https://your-compatible-endpoint.example.com'

for await (const message of query({
  prompt: 'Say hi',
  options: { model: 'your-custom-model' },
})) {
  console.log(JSON.stringify(message))
}
```
