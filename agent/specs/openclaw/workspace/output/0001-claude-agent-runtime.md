# Claude Agent SDK 原生插件实现方案

## 1. 目标与边界

目标是在 OpenClaw 中新增一个基于 `@anthropic-ai/claude-agent-sdk` 的原生 Claude Agent Runtime 插件，形态参考 `extensions/codex` 的 native app-server harness，而不是把 Claude Code 简化成普通 LLM provider 或 CLI backend。

推荐插件 id 为 `claude`，harness id 为 `claude`，provider id 为 `anthropic` 或 `claude` 需要谨慎选择：

- 如果要接管现有 `anthropic/sonnet-*` 模型引用，应注册 harness `claude`，但 provider 仍由现有 Anthropic provider 提供模型和 auth 控制面，运行时通过显式 `agentRuntime.id: "claude"` 选择 Claude harness。
- 如果希望完全插件自包含，可新增 provider id `claude`，暴露 `claude/sonnet-4.6`、`claude/opus-*` 等模型，并由插件声明 synthetic auth。第一阶段更推荐自包含 provider，避免改变现有 `anthropic/*` 默认运行时行为。

核心边界：

- Claude Agent SDK 负责 Claude Code 原生会话、工具执行、内建权限模式、子代理、会话存储和 SDK 消息流。
- OpenClaw 仍负责渠道投递、OpenClaw session/transcript、agent/runtime 选择、auth profile、工具策略、审批 UX、message tool、media 交付、heartbeat、context engine、hooks、fallback 分类和运行控制。
- 插件生产代码只通过 `openclaw/plugin-sdk/*` 与 OpenClaw 交互，不 deep import core。

## 2. 为什么使用 agent harness

Claude Agent SDK 的 `query()` 返回 `AsyncGenerator<SDKMessage>`，并带有 `interrupt()`、`setPermissionMode()`、`setModel()`、`streamInput()`、`mcpServerStatus()`、`close()` 等原生会话控制方法。`Options` 还包含 `resume`、`sessionId`、`permissionMode`、`canUseTool`、`hooks`、`mcpServers`、`settingSources`、`systemPrompt`、`tools`、`agents`、`sandbox`、`pathToClaudeCodeExecutable` 等运行时控制项。

这些语义和 Codex app-server 类似：上游 runtime 不只是一次文本补全，而是拥有完整 agent loop、工具事件、权限请求、会话恢复和本地执行环境。因此应实现 `api.registerAgentHarness(...)`，并按 Codex 的方式把 SDK 消息投影回 `EmbeddedRunAttemptResult`。

不选其他形态的原因：

- Provider plugin 适合普通 HTTP/WS 模型 API，不适合 Claude Code 原生工具、hooks、session resume。
- CLI backend 只适合文本 fallback runner，不承载 OpenClaw 工具桥、事件流、审批和 transcript mirror。
- ACP 已有外部 harness 路径，但这里的目标是 OpenClaw 内嵌原生 runtime，像 Codex 插件一样成为 `agentRuntime.id` 可选项。

## 3. 插件包结构

建议目录：

```text
extensions/claude/
  openclaw.plugin.json
  package.json
  index.ts
  harness.ts
  provider.ts
  provider-catalog.ts
  provider-discovery.ts
  src/agent-sdk/
    config.ts
    client.ts
    run-attempt.ts
    event-projector.ts
    dynamic-tools.ts
    approval-bridge.ts
    session-binding.ts
    transcript-mirror.ts
    compact.ts
    managed-binary.ts
    trajectory.ts
    test-support.ts
```

`package.json` 关键依赖：

- `@anthropic-ai/claude-agent-sdk`
- `zod`
- `@openclaw/plugin-sdk` dev dependency

SDK 文档说明该包会通过可选依赖捆绑平台本地 Claude Code 二进制。如果包管理器跳过 optional dependency，SDK 会报 `Native CLI binary for <platform> not found`，插件应支持配置 `pathToClaudeCodeExecutable` 指向单独安装的 `claude`。

## 4. Manifest 设计

`openclaw.plugin.json` 建议：

```json
{
  "id": "claude",
  "name": "Claude",
  "description": "Claude Agent SDK harness and Claude Code native runtime.",
  "providers": ["claude"],
  "providerDiscoveryEntry": "./provider-discovery.ts",
  "syntheticAuthRefs": ["claude"],
  "nonSecretAuthMarkers": ["claude-agent-sdk"],
  "activation": {
    "onStartup": false,
    "onAgentHarnesses": ["claude"],
    "onProviders": ["claude"]
  },
  "commandAliases": [
    {
      "name": "claude",
      "kind": "runtime-slash",
      "cliCommand": "plugins"
    }
  ],
  "configSchema": {
    "type": "object",
    "additionalProperties": false,
    "properties": {
      "discovery": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "enabled": { "type": "boolean", "default": true },
          "timeoutMs": { "type": "number", "minimum": 1, "default": 2500 }
        }
      },
      "agentSdk": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "pathToClaudeCodeExecutable": { "type": "string" },
          "permissionMode": {
            "type": "string",
            "enum": ["default", "acceptEdits", "bypassPermissions", "plan", "dontAsk", "auto"],
            "default": "default"
          },
          "allowDangerouslySkipPermissions": { "type": "boolean", "default": false },
          "settingSources": {
            "type": "array",
            "items": { "type": "string", "enum": ["user", "project", "local"] },
            "default": ["user", "project", "local"]
          },
          "toolsPreset": {
            "type": "string",
            "enum": ["claude_code", "openclaw_mcp_only", "none"],
            "default": "claude_code"
          },
          "mcpBridge": {
            "type": "string",
            "enum": ["disabled", "sdk", "stdio"],
            "default": "sdk"
          },
          "persistSession": { "type": "boolean", "default": true },
          "startupWarm": { "type": "boolean", "default": false },
          "initializeTimeoutMs": { "type": "number", "minimum": 1, "default": 60000 },
          "requestTimeoutMs": { "type": "number", "minimum": 1, "default": 60000 },
          "turnCompletionIdleTimeoutMs": { "type": "number", "minimum": 1, "default": 60000 },
          "defaultWorkspaceDir": { "type": "string" },
          "debug": { "type": "boolean", "default": false },
          "debugFile": { "type": "string" }
        }
      }
    }
  }
}
```

第一阶段不要让 manifest 改写现有 `anthropic` provider 默认行为。若后续决定让 `anthropic/* + agentRuntime.id: "auto"` 自动选择 Claude harness，需要单独设计 owner review 和迁移策略。

## 5. Runtime 注册入口

`index.ts` 采用 Codex 同款注册模型：

```ts
import { definePluginEntry } from "openclaw/plugin-sdk/plugin-entry";
import { createClaudeAgentSdkHarness } from "./harness.js";
import { buildClaudeProvider } from "./provider.js";

export default definePluginEntry({
  id: "claude",
  name: "Claude",
  description: "Claude Agent SDK harness and Claude Code native runtime.",
  register(api) {
    api.registerAgentHarness(createClaudeAgentSdkHarness({ pluginConfig: api.pluginConfig }));
    api.registerProvider(buildClaudeProvider({ pluginConfig: api.pluginConfig }));
  },
});
```

`register(api)` 必须同步；SDK import、binary resolution、warm startup 都放到 `runAttempt()` 的 lazy import 或 service lifecycle 中，不在入口顶层启动进程。

## 6. AgentHarness 设计

`harness.ts`：

- `id: "claude"`
- `label: "Claude Agent SDK harness"`
- `deliveryDefaults.sourceVisibleReplies = "message_tool"`，与 Codex 保持一致，避免 native runtime 直接绕过 OpenClaw source reply 规则。
- `supports(ctx)` 第一阶段只支持 provider `claude`，priority `100`。
- `runAttempt(params)` lazy import `src/agent-sdk/run-attempt.ts`。
- `compact(params)` 可第一阶段返回 `undefined`，让 OpenClaw 外层保持当前 compaction 策略；第二阶段映射 Claude SDK compact boundary 或 session resume。
- `reset(params)` 清理 `sessionFile -> claude session_id` sidecar binding。
- `dispose()` 关闭 warm query/shared query。

显式 runtime 配置示例：

```json
{
  "agents": {
    "defaults": {
      "model": "claude/sonnet-4.6",
      "agentRuntime": { "id": "claude" }
    }
  }
}
```

## 7. Provider 与模型目录

插件自带 `claude` provider 的职责主要是控制面可见性，不应重复实现普通 Anthropic HTTP provider：

- `resolveSyntheticAuth()` 返回 `claude-agent-sdk` marker，表示 auth 由本机 Claude Code / Claude Agent SDK 管理。
- fallback catalog 包含 `sonnet-4.6`、`opus-*`、`haiku-*` 等当前支持模型别名；具体列表从项目既有测试模型约定和 SDK `supportedModels()` 动态发现组合而来。
- `provider-discovery.ts` 可在 discovery enabled 时用短 timeout 调用 SDK `startup().supportedModels()` 或一次无 prompt initialization；失败回退静态 catalog。
- `resolveThinkingProfile()` 映射 OpenClaw `thinkLevel` 到 SDK `Options.effort` 或 `thinking`。

模型 ref 传入 SDK 时应去掉 provider prefix，只把 Claude 原生 model id 给 `Options.model`。兼容 alias 在 provider 内解决，不放进 core。

## 8. runAttempt 主流程

`runClaudeAgentSdkAttempt(params, { pluginConfig })` 的主体步骤：

1. 解析 live plugin config，得到 SDK options 默认值、workspace、timeout、permission policy。
2. 解析 `sessionAgentId`、agent dir、workspace dir、sandbox context、auth profile、runtimePlan。
3. 构建 OpenClaw 工具集合，按模型能力、图片输入和 runtime plan normalize。
4. 构建 Claude SDK MCP bridge。优先使用 `createSdkMcpServer()` 在同进程暴露 OpenClaw 工具；必要时支持 stdio MCP bridge 作为 fallback。
5. 从 OpenClaw session file 读取 sidecar binding，若存在则设置 `Options.resume = claudeSessionId`；若无则 fresh run。
6. 通过 context engine / bootstrap helper 得到最终 prompt、developer/system instructions 和 history projection。
7. 调用 `resolveAgentHarnessBeforePromptBuildResult(...)` 让 OpenClaw hooks 有机会改写 prompt。
8. 调用 Claude SDK `query({ prompt, options })`，消费 `SDKMessage` 流。
9. 投影 assistant/reasoning/tool/status/result/auth/rate-limit/task/compact 事件到 OpenClaw `onAgentEvent`、reasoning stream、plan update、tool progress。
10. 记录或更新 `sessionFile -> SDK session_id` binding。
11. mirror user/assistant 可见消息到 OpenClaw transcript。
12. 运行 `runAgentHarnessLlmOutputHook(...)`、`runAgentHarnessAgentEndHook(...)`。
13. 返回完整 `EmbeddedRunAttemptResult`。

超时和取消：

- 外层 `params.abortSignal` 触发时调用 `query.interrupt()`，失败时调用 `query.close()`。
- 总 attempt timeout 使用 `AbortController` 注入 `Options.abortController`。
- turn completion idle timeout 用 SDK message 活动刷新；超时后 interrupt 并标记 `idleTimedOut`。
- finally 中必须 `close()` query/warm handle，移除 event listener，flush trajectory。

## 9. SDK Options 映射

建议映射表：

| OpenClaw 输入 | Claude SDK Options |
| --- | --- |
| `params.modelId` | `model` |
| `params.workspaceDir` / resolved spawn workspace | `cwd` |
| `params.timeoutMs` / abort | `abortController` |
| `params.thinkLevel` | `effort` 或 `thinking` |
| plugin config `permissionMode` | `permissionMode` |
| plugin config `allowDangerouslySkipPermissions` | `allowDangerouslySkipPermissions` |
| OpenClaw tool bridge | `mcpServers` + `tools` / `allowedTools` |
| OpenClaw approval bridge | `canUseTool` / hooks `PermissionRequest` |
| session binding | `resume` 或 `sessionId` |
| bootstrap/system prompt | `systemPrompt`，优先 `{ type: "preset", preset: "claude_code", append }` |
| config `settingSources` | `settingSources` |
| auth env/profile bridge | `env` |
| binary override | `pathToClaudeCodeExecutable` |
| sandbox context | `sandbox` 或 permission/tool policy |

默认 `systemPrompt` 建议使用 Claude Code preset 并通过 `append` 注入 OpenClaw developer instructions。若为了 prompt cache 稳定性需要排除动态段，可评估 `excludeDynamicSections: true`，但必须先用测试证明 transcript 字节和 Claude Code 项目说明行为可接受。

## 10. OpenClaw 工具桥

Claude SDK 提供两条工具接入路径：

- `tool()` + `createSdkMcpServer()`：同进程 SDK MCP server。
- `mcpServers` stdio/http/sse：外部 MCP server 配置。

第一阶段应使用 SDK MCP server，避免额外进程：

- 把每个 OpenClaw `AnyAgentTool` 转成 Claude SDK `tool(name, description, zodSchema, handler, annotations)`。
- OpenClaw 工具参数源是 TypeBox/JSON Schema，Claude SDK `tool()` 需要 Zod raw shape。可新增本地 `json-schema -> zod` 的有限转换，或绕过 `tool()` 直接构造 SDK MCP server 需要的 schema 结构；选择前必须读 SDK 类型。
- handler 内调用 OpenClaw tool `prepareArguments()` 和 `execute()`。
- 执行前保留 `wrapToolWithBeforeToolCallHook()`，执行后运行 `createAgentToolResultMiddlewareRunner({ runtime: "claude" })` 和 `runAgentHarnessAfterToolCallHook(...)`。
- 将 OpenClaw `AgentToolResult.content` 映射到 MCP `CallToolResult.content`，文本优先，media artifact 仍交给 OpenClaw telemetry 收集。

工具暴露策略：

- 默认 `toolsPreset: "claude_code"` 时保留 Claude Code 原生文件/shell/edit 工具，只把 OpenClaw 集成工具通过 MCP 暴露给 Claude。
- `openclaw_mcp_only` 可禁用 Claude Code 默认工具，仅暴露 OpenClaw MCP 工具，适合受控环境。
- `none` 只运行 Claude 原生能力，不暴露 OpenClaw 动态工具。

需要类似 Codex 的 `native-first` profile，排除 Claude Code 已原生拥有的 `read`、`write`、`edit`、`apply_patch`、`exec`、`update_plan`，避免重复工具名和行为漂移。

## 11. 审批与权限桥

Claude SDK 的关键权限接口：

- `permissionMode`: `default`、`acceptEdits`、`bypassPermissions`、`plan`、`dontAsk`、`auto`。
- `canUseTool(toolName, input, options) -> PermissionResult`。
- hooks 中的 `PermissionRequest` / `PreToolUse` 事件。

方案：

- 默认不使用 `bypassPermissions`，除非用户显式配置 `allowDangerouslySkipPermissions: true`。
- `canUseTool` 作为 OpenClaw 审批桥主入口，把 Claude 工具名、输入、`toolUseID`、`blockedPath`、`decisionReason` 转成 OpenClaw plugin approval/exec approval 请求。
- 用户 approve 后返回 `{ behavior: "allow", updatedInput, updatedPermissions }`。
- 用户 reject 或 OpenClaw policy block 后返回 `{ behavior: "deny", message, interrupt? }`。
- `suggestions` 可映射为 OpenClaw 后续 permission update，但第一阶段只展示，不自动持久化，避免把 Claude 权限规则误写进 OpenClaw policy。
- 对 OpenClaw MCP 工具，仍由 OpenClaw `before_tool_call` hook 和 trusted tool policy 决策；Claude `canUseTool` 主要处理 Claude 原生工具。

## 12. 事件投影

`event-projector.ts` 消费 `SDKMessage`：

- `assistant`：读取 `message.content` 中 text block，累计 assistant text；提取 `usage`。
- `partial_assistant`：如果启用 `includePartialMessages`，只用于内部 liveness 和可选 reasoning/progress，不直接提交 transcript。
- `result success`：作为 terminal success，记录 `result`、`stop_reason`、`usage`、`modelUsage`、`permission_denials`。
- `result error_*`：作为 terminal error，填充 `promptError` 和 `promptErrorSource: "prompt"`。
- `system` / `status` / `auth_status` / `rate_limit`：投影为 `onAgentEvent` diagnostic/status。
- `tool_progress` / `tool_use_summary` / `local_command_output`：投影为 OpenClaw tool progress stream，注意截断和 redaction。
- `hook_started` / `hook_progress` / `hook_response`：投影为 native hook relay 类事件。
- `compact_boundary`：投影为 compaction start/end 事件，第二阶段接入 context engine maintenance。
- `task_started` / `task_progress` / `task_updated` / `task_notification`：第一阶段只转 agent event，不自动创建 OpenClaw background task；第二阶段评估和 OpenClaw task runtime 对齐。
- `prompt_suggestion`：可转 diagnostic event，不写 transcript。

最终结果用 `classifyAgentHarnessTerminalOutcome(...)` 分类空输出、reasoning-only、planning-only，保持 fallback 语义和 Codex/PI 一致。

## 13. Transcript mirror 与 session binding

Claude SDK 有自己的 session id 和本地 JSONL session store；OpenClaw 仍需要本地 transcript mirror。

sidecar binding：

- key：OpenClaw `sessionFile` 或 `sessionId + sessionKey`。
- value：`claudeSessionId`、`cwd`、`model`、`lastMessageUuid`、`createdAt`、`updatedAt`。
- 存储：优先使用 session file 旁的 plugin-owned binding helper；若要用 `runtime.state.openKeyedStore()`，确认 bundled-only 限制和 TTL 策略。

mirror 规则：

- fresh turn 写入 user prompt mirror，idempotency key `${sdkSessionId}:${userUuid || turnStart}:prompt`。
- terminal assistant 写入最后可见 assistant text，idempotency key `${sdkSessionId}:${assistantUuid}:assistant`。
- reasoning、plan、tool progress 默认不写成普通 assistant transcript，除非 OpenClaw transcript 已有对应结构化表示。
- 写入前运行 `runAgentHarnessBeforeMessageWriteHook(...)`。
- 使用 session write lock，写完发 transcript update。

`reset(params)` 删除 sidecar binding。session 删除、`/new`、`/reset` 都不能留下旧 Claude resume id。

## 14. 多轮与 steer

Claude SDK 支持 `prompt: AsyncIterable<SDKUserMessage>` 和 `query.streamInput(...)`。第一阶段可以按“一次 OpenClaw turn 对应一次 SDK query/resume”实现，复杂度最低。

第二阶段支持 active run steer：

- `setActiveEmbeddedRun(sessionId, handle, sessionKey)`。
- `queueMessage(text)` 通过 `query.streamInput()` 写入 SDK user message。
- `cancel()` 调用 `query.interrupt()`。
- `abort()` 调用 abort controller + `close()`。
- `isStreaming()` 根据 terminal result 状态返回。

如果使用 streaming input，必须确保 OpenClaw 命令如 `/status`、`/unfocus` 不被当普通 prompt 发给 Claude，绑定控制逻辑参考 Codex conversation binding。

## 15. Compaction 方案

第一阶段：

- 不调用 Claude SDK 特定 compaction。
- 依赖 Claude Code 自身上下文管理和 SDK session store。
- OpenClaw context engine 只负责 prompt assembly；mirror transcript 只记录可见消息。

第二阶段：

- 读取 SDK `SDKCompactBoundaryMessage`，投影 `before_compaction` / `after_compaction` hook。
- 如 SDK 暴露可控 compact 命令或 slash command，通过 `supportedCommands()` 探测后实现 `compact(params)`。
- 若 compact 后 SDK session id 不变，只更新 binding metadata；若 fork/resume 生成新 id，显式更新 binding。

不要在 core 添加 Claude-specific compaction 分支。

## 16. Auth 与环境

Claude Agent SDK/Claude Code 通常使用本机 Claude 登录状态。插件应提供三类 auth 运行方式：

- native：继承用户 Claude Code 登录和 settings，`settingSources` 默认包含 user/project/local。
- isolated：设置每个 OpenClaw agent 独立 `CLAUDE_CONFIG_DIR`/HOME 类环境变量；具体变量必须以后续读取 Claude SDK/CLI env docs 为准。
- api-key：从 OpenClaw auth profile 注入 Anthropic API key 到 SDK `env`，仅当 SDK/CLI 支持该路径时启用。

不得在日志、trajectory 或 transcript 中输出真实 token。status 只显示 synthetic marker 和账号/模型可用性摘要。

## 17. 配置模式

建议运行模式：

- `default`：Claude Code 默认权限，OpenClaw 审批桥处理额外请求。
- `guardian`：`permissionMode: "dontAsk"` 或 `"default"` + OpenClaw approvals，sandbox workspace-write，适合远程聊天触发。
- `plan`：`permissionMode: "plan"`，不执行本地修改。
- `yolo`：`permissionMode: "bypassPermissions"` + `allowDangerouslySkipPermissions: true`，必须显式配置和 UI 警示。

和 Codex 不同，Claude SDK 已有较丰富 `PermissionMode`，插件应优先映射 SDK 原生权限，而不是重做一套 app-server mode。

## 18. 测试计划

单元测试：

- manifest contract：`activation.onAgentHarnesses`、provider、synthetic auth、config schema。
- entry 注册：`registerAgentHarness`、`registerProvider` 被调用。
- harness selection：provider `claude` 支持，其他 provider 不支持。
- config parser：permissionMode、settingSources、mcpBridge、binary path、timeout。
- SDK message projector：assistant/result/error/tool/status/rate-limit/auth/task/compact。
- transcript mirror idempotency。
- session binding reset。
- tool bridge：OpenClaw tool success/error/media/message tool telemetry。
- approval bridge：allow/deny/interrupt/updatedInput。
- abort/timeout cleanup：interrupt、close、listener cleanup。

集成测试：

- fake Claude SDK query generator 驱动完整 `runAttempt()`，断言 `EmbeddedRunAttemptResult`。
- fake SDK deferred permission 请求，走 OpenClaw approval bridge。
- fake MCP tool call，验证 OpenClaw hooks 和 middleware 顺序。
- resume binding：第一次保存 session id，第二次传 `Options.resume`。

Live 测试：

- 有 Claude auth 的机器上运行 `OPENCLAW_LIVE_TEST=1 pnpm test extensions/claude`。
- 验证 binary 缺失时错误可行动：提示安装 optional dependency、运行 `pnpm install`、或配置 `pathToClaudeCodeExecutable`。
- 验证 `agentRuntime.id: "claude"` 的实际 harness selection 结构化日志。

## 19. 分阶段交付

阶段 1：最小可运行

- 新增 `extensions/claude` 插件。
- 注册 `claude` provider + `claude` harness。
- `query()` fresh/resume turn。
- assistant/result/error 投影。
- session binding + transcript mirror。
- 基础 config + fake SDK 测试。

阶段 2：OpenClaw 工具与审批

- SDK MCP server 暴露 OpenClaw dynamic tools。
- Claude `canUseTool` 接 OpenClaw approvals。
- tool progress、local command output、permission denial 投影。
- message tool/media/heartbeat telemetry。

阶段 3：运行时质量

- active run steer/cancel。
- warm `startup()` 池。
- model discovery。
- context engine assembly/finalize。
- compact boundary 投影。
- trajectory 和 structured diagnostics。

阶段 4：产品化

- `/claude` 命令和 conversation binding。
- doctor/readiness。
- docs、UI hints、status surface。
- 可选支持现有 `anthropic/* + agentRuntime.id: "claude"`。

## 20. 关键风险

- SDK 类型和消息 union 很宽，projector 必须用窄化函数和 exhaustive tests，不能依赖字符串猜测。
- Claude Code 原生工具和 OpenClaw 工具重名时会导致权限和结果归属混乱，必须默认 native-first 排除。
- `bypassPermissions` 高风险，必须显式配置，不能作为默认。
- SDK session store 和 OpenClaw transcript 是双写关系，必须用 idempotency key 和 reset cleanup 防止重复/错 resume。
- optional native binary 安装受包管理器影响，doctor 需要给出明确修复路径。
- 如果后续让 `anthropic/*` 自动选择 Claude harness，会改变现有用户运行时语义，必须单独走 owner review 和迁移说明。
