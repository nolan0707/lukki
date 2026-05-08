# Claude Agent Runtime 插件详细设计

## 1. 背景

本文基于 `specs/workspace/output/0001-claude-agent-runtime.md`，把“基于 Claude Agent SDK 的 OpenClaw 原生插件”细化为可实施的工程设计。目标插件参考 `extensions/codex` 的实现机制，将 Claude Agent SDK 接入为 OpenClaw 的 trusted agent harness。

设计结论：

- 插件 id：`claude`
- harness id：`claude`
- provider id：`claude`
- SDK 依赖：`@anthropic-ai/claude-agent-sdk`
- 主要入口：`api.registerAgentHarness(...)`
- 辅助入口：`api.registerProvider(...)`

该插件不改变现有 `anthropic/*` 默认 provider 路径。用户必须显式使用 `claude/*` 模型引用，或显式配置 `agentRuntime.id: "claude"`。如需让 `anthropic/*` 自动选择 Claude harness，应作为独立迁移项目处理。

## 2. 设计目标

功能目标：

- 使用 Claude Agent SDK 的 `query()` 执行一次 OpenClaw embedded agent attempt。
- 支持 Claude SDK session resume，并把 Claude session id 绑定到 OpenClaw session。
- 把 SDK 消息流投影为 OpenClaw assistant text、reasoning/progress、tool progress、usage、diagnostic event 和 terminal result。
- 通过 SDK MCP server 将 OpenClaw 中定义的 agent tools 暴露给 Claude，并完整执行 OpenClaw tool lifecycle。
- 将 Claude 原生工具权限请求桥接到 OpenClaw 审批体系。
- 支持 OpenClaw agent harness 相关 hook，包括 prompt、LLM input/output、tool call、message write、compaction、agent finalize/end 等生命周期。
- 镜像用户可见消息到 OpenClaw transcript，保持渠道历史、搜索和 session 管理可用。
- 支持 abort、timeout、idle timeout、cleanup 和 reset。

非目标：

- 不接管现有 Anthropic HTTP provider。
- 不在 core 中加入 Claude-specific 分支。
- 不把 Claude Code 作为 CLI backend 实现。
- 不默认启用 `bypassPermissions`。
- 不让插件启动时 eager import 或启动 Claude SDK 进程。
- 不把 Claude SDK session store 当成 OpenClaw canonical transcript。

## 3. 架构总览

```text
OpenClaw embedded run
  -> provider/model/auth/runtime plan resolution
  -> select AgentHarness(id="claude")
  -> extensions/claude/harness.ts
  -> src/agent-sdk/run-attempt.ts
     -> config/auth/workspace/session binding
     -> OpenClaw tool bridge as Claude SDK MCP server
     -> Claude SDK query({ prompt, options })
     -> SDKMessage projector
     -> transcript mirror
     -> EmbeddedRunAttemptResult
```

核心分工：

| 层 | 责任 |
| --- | --- |
| OpenClaw core | runtime 选择、session、channel delivery、tool policy、hooks、fallback、model/auth resolution |
| `extensions/claude/index.ts` | 同步注册 provider 和 harness |
| `extensions/claude/harness.ts` | 声明 harness id、支持条件、lazy 调用 run/compact/reset/dispose |
| `src/agent-sdk/run-attempt.ts` | 一次 Claude SDK turn 的编排 |
| `src/agent-sdk/event-projector.ts` | SDKMessage 到 OpenClaw result/event 的投影 |
| `src/agent-sdk/dynamic-tools.ts` | OpenClaw 工具到 SDK MCP server 的桥 |
| `src/agent-sdk/approval-bridge.ts` | Claude 原生权限请求到 OpenClaw 审批 |
| `src/agent-sdk/session-binding.ts` | OpenClaw session 和 Claude session id 的绑定 |
| `src/agent-sdk/transcript-mirror.ts` | 可见 transcript 镜像 |

## 4. 文件布局

建议新增：

```text
extensions/claude/
  openclaw.plugin.json
  package.json
  tsconfig.json
  index.ts
  harness.ts
  provider.ts
  provider-catalog.ts
  provider-discovery.ts
  src/
    agent-sdk/
      approval-bridge.ts
      auth-bridge.ts
      config.ts
      dynamic-tool-profile.ts
      dynamic-tools.ts
      event-projector.ts
      managed-binary.ts
      message-readers.ts
      run-attempt.ts
      session-binding.ts
      shared-query.ts
      transcript-mirror.ts
      usage.ts
      test-support.ts
```

完整目标态包含 `/claude` 命令与 conversation binding；它们可以和 runtime 文件分开实现，但不作为可省略能力处理：

```text
extensions/claude/src/
  commands.ts
  command-handlers.ts
  conversation-binding.ts
  conversation-control.ts
```

## 5. Package 设计

`extensions/claude/package.json`：

```json
{
  "name": "@openclaw/claude",
  "version": "2026.5.6",
  "description": "OpenClaw Claude Agent SDK harness and Claude Code runtime plugin",
  "type": "module",
  "dependencies": {
    "@anthropic-ai/claude-agent-sdk": "^0.0.0",
    "zod": "^4.4.3"
  },
  "devDependencies": {
    "@openclaw/plugin-sdk": "workspace:*"
  },
  "openclaw": {
    "extensions": ["./index.ts"],
    "install": {
      "npmSpec": "@openclaw/claude",
      "defaultChoice": "npm",
      "minHostVersion": ">=2026.5.6"
    },
    "compat": {
      "pluginApi": ">=2026.5.6"
    },
    "build": {
      "openclawVersion": "2026.5.6"
    }
  }
}
```

注意：

- 真实 `@anthropic-ai/claude-agent-sdk` 版本在实现时按 lockfile 和 SDK docs 确认。
- SDK optional native binary 属于插件 runtime dependency，不放到 root，除非 OpenClaw package dist 自身要 import 它。
- 如果发布包提供 built JS，应补 `runtimeExtensions`，遵守插件 entrypoint 规则。

## 6. Manifest 设计

`extensions/claude/openclaw.plugin.json` 负责控制面事实。

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

Manifest 不声明 runtime entry，不执行 SDK。`activation.onAgentHarnesses` 用于当配置明确需要 `claude` harness 时加载插件。`providers: ["claude"]` 让模型选择、status 和 setup 能看到该 provider。

## 7. Config 类型与默认值

`src/agent-sdk/config.ts`：

```ts
export type ClaudePermissionMode =
  | "default"
  | "acceptEdits"
  | "bypassPermissions"
  | "plan"
  | "dontAsk"
  | "auto";

export type ClaudeToolsPreset = "claude_code" | "openclaw_mcp_only" | "none";
export type ClaudeMcpBridgeMode = "disabled" | "sdk" | "stdio";
export type ClaudeSettingSource = "user" | "project" | "local";

export type ClaudePluginConfig = {
  discovery?: {
    enabled?: boolean;
    timeoutMs?: number;
  };
  agentSdk?: {
    pathToClaudeCodeExecutable?: string;
    permissionMode?: ClaudePermissionMode;
    allowDangerouslySkipPermissions?: boolean;
    settingSources?: ClaudeSettingSource[];
    toolsPreset?: ClaudeToolsPreset;
    mcpBridge?: ClaudeMcpBridgeMode;
    persistSession?: boolean;
    startupWarm?: boolean;
    initializeTimeoutMs?: number;
    requestTimeoutMs?: number;
    turnCompletionIdleTimeoutMs?: number;
    defaultWorkspaceDir?: string;
    debug?: boolean;
    debugFile?: string;
  };
};

export type ResolvedClaudeAgentSdkConfig = {
  discoveryEnabled: boolean;
  discoveryTimeoutMs: number;
  permissionMode: ClaudePermissionMode;
  allowDangerouslySkipPermissions: boolean;
  settingSources: ClaudeSettingSource[];
  toolsPreset: ClaudeToolsPreset;
  mcpBridge: ClaudeMcpBridgeMode;
  persistSession: boolean;
  startupWarm: boolean;
  initializeTimeoutMs: number;
  requestTimeoutMs: number;
  turnCompletionIdleTimeoutMs: number;
  pathToClaudeCodeExecutable?: string;
  defaultWorkspaceDir?: string;
  debug: boolean;
  debugFile?: string;
};
```

解析规则：

- 不可信 plugin config 使用 zod `.strict()` 解析，失败回退默认值并记录 diagnostic。
- `bypassPermissions` 只有在 `allowDangerouslySkipPermissions === true` 时允许；否则降级为 `default` 并发 warning。
- `settingSources` 默认 `["user", "project", "local"]`，但测试可覆盖为 `[]`。
- `mcpBridge` 默认 `sdk`。
- timeout 全部最小 100ms clamp。

## 8. Runtime entry

`index.ts`：

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

实现约束：

- `register()` 不 `await`。
- 不在 top-level import `@anthropic-ai/claude-agent-sdk`，避免 discovery/full load 时立即解析 optional binary。
- 重 SDK import 放在 `run-attempt.ts`、`provider-discovery.ts` 的 lazy import 内。
- 如果需要 live config，参考 Codex 的 `resolveLivePluginConfigObject(...)` 模式，在运行时读取当前 config。

## 9. Harness 设计

`harness.ts`：

```ts
import type { AgentHarness } from "openclaw/plugin-sdk/agent-harness-runtime";

const DEFAULT_CLAUDE_HARNESS_PROVIDER_IDS = new Set(["claude"]);

export function createClaudeAgentSdkHarness(options?: {
  id?: string;
  label?: string;
  providerIds?: Iterable<string>;
  pluginConfig?: unknown;
}): AgentHarness {
  const providerIds = new Set(
    [...(options?.providerIds ?? DEFAULT_CLAUDE_HARNESS_PROVIDER_IDS)].map((id) =>
      id.trim().toLowerCase(),
    ),
  );

  return {
    id: options?.id ?? "claude",
    label: options?.label ?? "Claude Agent SDK harness",
    deliveryDefaults: {
      sourceVisibleReplies: "message_tool",
    },
    supports(ctx) {
      const provider = ctx.provider.trim().toLowerCase();
      if (providerIds.has(provider)) {
        return { supported: true, priority: 100 };
      }
      return {
        supported: false,
        reason: `provider is not one of: ${[...providerIds].toSorted().join(", ")}`,
      };
    },
    async runAttempt(params) {
      const { runClaudeAgentSdkAttempt } = await import("./src/agent-sdk/run-attempt.js");
      return runClaudeAgentSdkAttempt(params, { pluginConfig: options?.pluginConfig });
    },
    async compact(params) {
      const { maybeCompactClaudeAgentSdkSession } = await import("./src/agent-sdk/compact.js");
      return maybeCompactClaudeAgentSdkSession(params, { pluginConfig: options?.pluginConfig });
    },
    async reset(params) {
      if (params.sessionFile) {
        const { clearClaudeAgentSdkBinding } = await import("./src/agent-sdk/session-binding.js");
        await clearClaudeAgentSdkBinding(params.sessionFile);
      }
    },
    async dispose() {
      const { closeSharedClaudeAgentSdkQueries } = await import("./src/agent-sdk/shared-query.js");
      await closeSharedClaudeAgentSdkQueries();
    },
  };
}
```

`compact()` 必须接入 Claude SDK 可用的 compaction/session 能力；如果 SDK 当前没有显式 compact API，则至少消费 `SDKCompactBoundaryMessage` 并完整触发 OpenClaw compaction hooks，返回清晰的 unsupported/unchanged 结果，而不是静默 no-op。

## 10. Provider 设计

`provider.ts` 让 `claude/*` 模型对 OpenClaw 可见。它不负责普通 Anthropic HTTP 推理。

职责：

- 暴露 provider id `claude`。
- 提供 fallback model catalog。
- 提供 synthetic auth marker。
- 把 `claude/sonnet-4.6` 等模型 ref 解析为 SDK `Options.model` 能接受的 id。
- 把 OpenClaw thinking level 映射到 SDK `effort` / `thinking`。

模型目录第一版：

```ts
export const CLAUDE_FALLBACK_MODELS = [
  {
    id: "sonnet-4.6",
    label: "Claude Sonnet 4.6",
    aliases: ["sonnet", "claude-sonnet-4.6"],
    contextWindow: undefined,
    supportsTools: true,
    supportsImages: true,
  },
  {
    id: "opus",
    label: "Claude Opus",
    aliases: ["opus"],
    supportsTools: true,
    supportsImages: true,
  },
  {
    id: "haiku",
    label: "Claude Haiku",
    aliases: ["haiku"],
    supportsTools: true,
    supportsImages: true,
  },
];
```

动态发现：

- `provider-discovery.ts` 在 `discovery.enabled !== false` 时尝试调用 SDK `startup({ initializeTimeoutMs })`。
- 成功后读取 `warm.query(...).supportedModels()` 不合适，因为 `WarmQuery.query()` 只能调用一次。更稳妥的实现是直接 `query({ prompt, options }).supportedModels()` 或使用 SDK 初始化 API 后立即 `close()`，具体以实际 SDK 类型验证为准。
- discovery timeout 到期或 SDK binary/auth 不可用时，返回 fallback catalog，不阻断插件注册。

## 11. runAttempt 状态机

`runClaudeAgentSdkAttempt()` 使用显式状态，便于 cleanup 和测试。

```text
init
  -> resolve_config
  -> resolve_workspace
  -> resolve_auth
  -> load_binding
  -> build_tools
  -> build_prompt
  -> start_query
  -> consume_stream
  -> mirror_transcript
  -> finalize_context
  -> return_result
  -> cleanup
```

失败来源映射：

| 来源 | promptErrorSource | replayMetadata |
| --- | --- | --- |
| config/auth/workspace precheck | `precheck` | replay safe |
| before_agent_run hook block | `hook:before_agent_run` | replay safe |
| SDK query start failure | `prompt` | replay safe if no SDK session id emitted |
| SDK stream error after tool/message side effect | `prompt` | replay unsafe if telemetry says side effects |
| context finalize/compaction cleanup failure | `compaction` | do not replay fresh prompt |

## 12. runAttempt 参数解析

使用 SDK helpers：

- `resolveAttemptSpawnWorkspaceDir(...)`
- `resolveSandboxContext(...)`
- `resolveSessionAgentIds(...)`
- `resolveModelAuthMode(...)`
- `resolveBootstrapContextForRun(...)`
- `resolveAgentHarnessBeforePromptBuildResult(...)`
- `normalizeAgentRuntimeTools(...)`
- `buildEmbeddedAttemptToolRunContext(...)`

需要准备的本地变量：

```ts
type ClaudeAttemptRuntime = {
  runAbortController: AbortController;
  sessionAgentId: string;
  sandboxSessionKey?: string;
  effectiveWorkspace: string;
  agentDir: string;
  binding?: ClaudeAgentSdkSessionBinding;
  sdkSessionId?: string;
  promptText: string;
  developerInstructions: string;
  mcpServerName?: string;
  query?: import("@anthropic-ai/claude-agent-sdk").Query;
  projector: ClaudeAgentSdkEventProjector;
  toolBridge?: ClaudeAgentSdkToolBridge;
};
```

不要在类型定义顶层 import SDK runtime。需要类型时使用 `import type` 或本地 adapter 类型，生产代码中 SDK lazy import。

## 13. SDK query 构造

核心调用：

```ts
const sdk = await import("@anthropic-ai/claude-agent-sdk");
const query = sdk.query({
  prompt: promptBuild.prompt,
  options: buildClaudeSdkOptions({
    params,
    config,
    cwd: effectiveWorkspace,
    developerInstructions: promptBuild.developerInstructions,
    binding,
    toolBridge,
    abortController: runAbortController,
  }),
});
```

`buildClaudeSdkOptions()` 映射：

```ts
type BuildClaudeSdkOptionsInput = {
  params: EmbeddedRunAttemptParams;
  config: ResolvedClaudeAgentSdkConfig;
  cwd: string;
  developerInstructions: string;
  binding?: ClaudeAgentSdkSessionBinding;
  toolBridge?: ClaudeAgentSdkToolBridge;
  abortController: AbortController;
};
```

输出规则：

- `model = params.modelId`
- `cwd = effectiveWorkspace`
- `abortController = runAbortController`
- `permissionMode = config.permissionMode`
- `allowDangerouslySkipPermissions = config.allowDangerouslySkipPermissions`，仅当 permissionMode 是 `bypassPermissions`
- `persistSession = config.persistSession`
- `resume = binding?.claudeSessionId`，当 binding workspace/model 策略允许 resume
- `settingSources = config.settingSources`
- `pathToClaudeCodeExecutable = config.pathToClaudeCodeExecutable`
- `debug = config.debug`
- `debugFile = config.debugFile`
- `systemPrompt = { type: "preset", preset: "claude_code", append: developerInstructions }`
- `mcpServers = toolBridge?.mcpServers`
- `tools = resolveClaudeToolsOption(config.toolsPreset, toolBridge)`
- `canUseTool = createClaudePermissionBridge(...)`
- `stderr = data => embeddedAgentLog.debug(...)`，带 redaction 和 tail limit

`resume` 兼容策略：

- binding 中 `cwd` 与当前 effective workspace 不一致时不 resume，改 fresh run 并更新 binding。
- binding 中 `model` 与当前 `params.modelId` 不一致时默认不 resume，避免 Claude session 内模型切换行为不确定；如 SDK 文档/类型证明 `setModel()` 对 resume session 稳定，再由插件本地策略放开。
- binding 缺失时不设置 `sessionId`，让 SDK 生成；从首个 SDK message 读取 `session_id` 后保存。

## 14. Prompt 与 history 投影

Claude SDK 原生会话拥有自己的历史和压缩状态，OpenClaw context engine 拥有 OpenClaw 侧 transcript mirror、摘要、索引和上下文维护状态。两者是双轨上下文：

```text
Claude SDK session = 原生执行权威上下文
OpenClaw transcript mirror = OpenClaw 可见历史和 context engine 输入
OpenClaw context engine = prompt assembly / finalize / maintenance
```

Claude harness 不能把 OpenClaw transcript 当成 Claude 原生 session 的替代品，也不能在 resume turn 中重放完整 OpenClaw transcript。OpenClaw context engine 的输出只作为本轮 prompt projection 或 system/developer append 注入。

History projection 策略：

- Fresh run：把 OpenClaw 当前 user prompt 作为 SDK `prompt`。
- Resume run：仍把当前 user prompt 作为 SDK `prompt`，依赖 SDK `resume` 找到 Claude session 历史。
- 如果 OpenClaw context engine 启用，调用 `assembleHarnessContextEngine(...)`，把 assembly 的 user-visible context 合并进 prompt。
- developer/system instructions 通过 `systemPrompt.append` 注入。
- 不把完整 OpenClaw transcript 每轮重放给 SDK，避免和 Claude session store 双历史叠加。
- context engine 生成的摘要、facts、外部记忆或新增关键上下文可以进入 prompt；完整历史不能进入 resume prompt。

Fresh run 的 prompt 模板：

```text
<context from OpenClaw context engine if present>

<current user prompt>
```

Resume run 的 prompt 模板：

```text
<current user prompt>
```

当 context engine 需要注入新 facts 时，通过 `before_prompt_build` hook 和 assembly result 控制。必须避免在 resume turn 每次塞入完整历史。

Run 前的 assembly 只回答一个问题：本轮 Claude SDK query 除了当前用户 prompt 之外，还需要哪些 OpenClaw 可控上下文。它不改变 Claude SDK session store。

## 15. OpenClaw 工具桥详细设计

Claude harness 必须支持 OpenClaw 中定义的 agent tools，而不只是少量动态工具。这里的“OpenClaw tools”指 core 和插件通过 OpenClaw runtime 为当前 attempt 准备出的 `AnyAgentTool[]`，包括 message、session/subagent、browser、cron、media、gateway、heartbeat，以及当前工具策略允许的 coding tools。

`dynamic-tools.ts` 输出：

```ts
export type ClaudeAgentSdkToolBridge = {
  mcpServers: Record<string, unknown>;
  allowedTools: string[];
  telemetry: ClaudeToolTelemetry;
  close(): Promise<void> | void;
};

export type ClaudeToolTelemetry = {
  didSendViaMessagingTool: boolean;
  messagingToolSentTexts: string[];
  messagingToolSentMediaUrls: string[];
  messagingToolSentTargets: MessagingToolSend[];
  heartbeatToolResponse?: HeartbeatToolResponse;
  toolMediaUrls: string[];
  toolAudioAsVoice: boolean;
  successfulCronAdds?: number;
};
```

构建步骤：

1. 调用 `createOpenClawCodingTools(...)` 获取 OpenClaw tool。
2. 合并 attempt 已准备的 tool 集合和 provider/runtime plan 允许的工具，保持 OpenClaw 侧工具选择为权威来源。
3. 调用 `normalizeAgentRuntimeTools(...)` 和 `params.runtimePlan?.tools.normalize(...)`。
4. 对每个 tool 包一层 `wrapToolWithBeforeToolCallHook(...)`。
5. 转成 Claude SDK tool definition 或 MCP server tool。
6. handler 执行 OpenClaw tool、运行中间件、触发 after-tool hook 并收集 telemetry。

工具暴露策略：

- 默认使用 `native-first`：Claude Code 原生文件/shell/edit 工具保持原生，OpenClaw 暴露消息、session、subagent、browser、cron、media、gateway、heartbeat 等集成工具。
- `openclaw_mcp_only`：关闭 Claude Code 默认工具，只暴露 OpenClaw 工具集合，适合完全由 OpenClaw policy 控制的部署。
- `claude_code`：保留 Claude Code preset，同时暴露所有不冲突的 OpenClaw 工具。
- `none`：只允许 Claude 原生工具，不暴露 OpenClaw tool；该模式仅用于诊断，默认配置不能选择它。

OpenClaw tool 执行上下文：

- `tool.execute(callId, preparedArgs, signal)` 使用 composed abort signal，包含 attempt abort、SDK tool call abort 和 per-tool timeout。
- `buildEmbeddedAttemptToolRunContext(...)` 生成的上下文必须传递给工具工厂，保证 message tool、session tool、subagent tool、browser/gateway tool 拿到一致的 run/session/channel 信息。
- tool allowlist、sandbox、runtimePlan tool normalization、schema normalization 都以 OpenClaw 准备结果为准，Claude 侧不能重新解释 OpenClaw policy。
- 如果 Claude SDK MCP bridge 只支持文本结果，media 结果仍必须通过 OpenClaw telemetry 回到统一 media delivery path，不能丢弃。

OpenClaw tool 到 MCP result 映射：

| OpenClaw result content | MCP content |
| --- | --- |
| `{ type: "text", text }` | `{ type: "text", text }` |
| image/media URL | 文本摘要 + telemetry media URL |
| structured details | JSON stringify 后文本，限制长度 |
| error result | `isError: true` 或失败文本 |

需要保留的 hook 顺序：

```text
Claude calls MCP tool
  -> OpenClaw before_tool_call hook via wrapper
  -> hook may block / require approval / rewrite args
  -> tool.prepareArguments()
  -> tool.execute()
  -> agent tool result middleware(runtime="claude")
  -> collect telemetry
  -> runAgentHarnessAfterToolCallHook()
  -> return MCP CallToolResult
```

工具名冲突：

- `read`, `write`, `edit`, `apply_patch`, `exec`, `process`, `update_plan`, `bash`, `grep`, `glob` 默认不暴露给 Claude MCP bridge。
- 如果 `toolsPreset === "openclaw_mcp_only"`，可暴露这些 OpenClaw 工具，但同时 `tools` 不使用 Claude Code preset。
- 如果 SDK/MCP server 不允许重复 tool name，启动前检测并 fail fast，错误包含冲突列表。
- 对被 native-first 排除的 OpenClaw 工具，必须记录 diagnostic，包括 tool name、排除原因和可用配置开关。

## 16. JSON Schema 到 Zod 策略

Claude SDK `tool()` 接收 Zod raw shape。OpenClaw tools 使用 TypeBox/JSON Schema。

实现优先级：

1. 检查 Claude SDK 是否允许 `createSdkMcpServer()` 接收非 `tool()` 构造的 JSON Schema tool definition。
2. 如果必须使用 `tool()`，实现局部 JSON Schema 转 Zod：
   - `type: "string"` -> `z.string()`
   - `type: "number"` / `integer` -> `z.number()` / `z.number().int()`
   - `type: "boolean"` -> `z.boolean()`
   - `type: "array"` -> `z.array(item)`
   - `type: "object"` -> `z.object(shape)`，默认 `.passthrough()` 仅当 schema 允许 additionalProperties
   - `enum` -> `z.enum([...])` 或 `z.union([z.literal(...)])`
   - unknown/unsupported -> `z.unknown()`
3. 转换失败时该工具不暴露，并发 structured diagnostic。

完整目标态必须覆盖 OpenClaw 当前工具 schema 的实际形态。对无法映射的 JSON Schema，插件不能默默降级为无参数工具；必须 fail closed 或跳过该 tool 并把原因写入 tool diagnostics。

## 17. 权限与审批桥

`approval-bridge.ts`：

```ts
export function createClaudeCanUseTool(params: {
  attempt: EmbeddedRunAttemptParams;
  config: ResolvedClaudeAgentSdkConfig;
  signal: AbortSignal;
}): CanUseTool {
  return async (toolName, input, options) => {
    return resolveClaudeToolPermission({
      toolName,
      input,
      toolUseId: options.toolUseID,
      blockedPath: options.blockedPath,
      decisionReason: options.decisionReason,
      agentId: options.agentID,
      suggestions: options.suggestions,
      attempt: params.attempt,
      signal: AbortSignal.any([params.signal, options.signal]),
    });
  };
}
```

决策流程：

```text
Claude native tool permission request
  -> classify tool as read/write/exec/network/unknown
  -> consult OpenClaw trusted tool policy if applicable
  -> if policy allows: return allow
  -> if policy blocks: return deny
  -> if approval callback exists: ask user
  -> approve: return allow
  -> reject/timeout: return deny
```

返回映射：

| OpenClaw decision | Claude PermissionResult |
| --- | --- |
| allow | `{ behavior: "allow", toolUseID }` |
| allow with rewritten args | `{ behavior: "allow", updatedInput, toolUseID }` |
| deny | `{ behavior: "deny", message, toolUseID }` |
| deny terminal | `{ behavior: "deny", message, interrupt: true, toolUseID }` |

安全默认：

- 无审批通道时，write/exec 类请求 deny。
- read-only 类请求是否 allow 由 `permissionMode` 和 OpenClaw sandbox 决定。
- `bypassPermissions` 必须由用户显式打开 `allowDangerouslySkipPermissions`。

## 18. OpenClaw hook 支持策略

Claude harness 必须支持 OpenClaw 中定义的 agent harness hook 生命周期。Claude SDK `Options.hooks` 只作为 native relay 入口，不替代 OpenClaw hooks。OpenClaw hook 仍由 `openclaw/plugin-sdk/agent-harness-runtime` 暴露的 helper 执行，确保 Claude harness、Codex harness 和 PI harness 的可观察行为一致。

必须覆盖的 OpenClaw hook：

| OpenClaw hook helper | 触发点 | Claude harness 行为 |
| --- | --- | --- |
| `resolveAgentHarnessBeforePromptBuildResult(...)` | SDK query 前，prompt/system prompt 定稿前 | 允许 hook 改写 prompt、developer instructions 和 history projection |
| `runAgentHarnessLlmInputHook(...)` | 调用 `query()` 前 | 记录 provider/model/system prompt/user prompt/history/images/tool count |
| `runAgentHarnessLlmOutputHook(...)` | terminal result 前 | 记录 assistant texts、last assistant、usage、resolved ref、harness id |
| `runAgentHarnessBeforeCompactionHook(...)` | SDK compact boundary start 或插件 compact start | 传入当前 mirrored messages 和 session context |
| `runAgentHarnessAfterCompactionHook(...)` | SDK compact boundary end 或插件 compact end | 传入 compacted count、messages 和 session context |
| `runAgentHarnessBeforeMessageWriteHook(...)` | transcript mirror 写入前 | 允许 hook 修改或阻止 mirror message |
| `runAgentHarnessAfterToolCallHook(...)` | OpenClaw MCP tool 执行完成后 | 记录 tool args、result/error、duration、run/session context |
| `runAgentHarnessBeforeAgentFinalizeHook(...)` | Claude Stop/SessionEnd 或 terminal result 前 | 允许插件请求最终化前处理；不能重入 SDK query，除非设计显式支持 |
| `runAgentHarnessAgentEndHook(...)` | attempt 完成或失败 | 记录 messages、success、error、duration |

SDK hook 到 OpenClaw hook 的映射：

| Claude SDK hook/event | OpenClaw 投影 |
| --- | --- |
| `PreToolUse` | native tool start diagnostic；必要时进入 approval bridge |
| `PostToolUse` | native tool result diagnostic；不走 OpenClaw tool middleware，因为这是 Claude 原生工具 |
| `PostToolUseFailure` | native tool error diagnostic；更新 replay side-effect 判断 |
| `PostToolBatch` | tool batch summary event；刷新 liveness |
| `Notification` | diagnostic event |
| `UserPromptSubmit` | prompt submit event；不写 transcript，transcript 由 mirror 层负责 |
| `SessionStart` | lifecycle start，保存 SDK session id |
| `SessionEnd` | lifecycle terminal，触发 before-finalize 和 agent-end |
| `Stop` | before-agent-finalize 投影 |
| `SubagentStart` / `SubagentStop` | subagent diagnostic event |
| `PreCompact` | before-compaction hook |
| `PermissionRequest` | approval bridge fallback |
| `TaskCompleted` | task diagnostic event |

每个 SDK hook 返回必须保守：

- 不在 hook 中改写 Claude 原生工具参数，除非 OpenClaw approval 明确返回 `updatedInput`。
- 不在 hook 中写 transcript。
- hook timeout 后按 deny 或 no-op，不能卡住 agent turn。

OpenClaw hook 错误处理：

- blocking hook 返回 block 时，Claude query 不能启动，result 使用 `promptErrorSource: "hook:before_agent_run"` 或对应阶段错误。
- tool hook block/approval 发生在 OpenClaw MCP tool wrapper 内，必须返回 Claude 可理解的 tool error，而不是让 SDK stream 崩溃。
- message write hook block 时，不写 mirror transcript，但 SDK canonical session 不回滚；result 中记录 diagnostic，避免误判为完整 mirror 成功。
- before-finalize hook 如果请求追加模型轮次，Claude harness 只能在有明确 SDK streaming/input 能力时执行；否则返回 unsupported diagnostic 并保持原 terminal result。

## 19. Event projector 设计

`event-projector.ts` 保持状态聚合：

```ts
export class ClaudeAgentSdkEventProjector {
  private sdkSessionId: string | undefined;
  private assistantTexts: string[] = [];
  private toolMetas: Array<{ toolName: string; meta?: string }> = [];
  private promptError: unknown = null;
  private promptErrorSource: EmbeddedRunAttemptResult["promptErrorSource"] = null;
  private usage: NormalizedUsage | undefined;
  private completed = false;
  private aborted = false;

  async handle(message: SDKMessage): Promise<void>;
  markTimedOut(message: string): void;
  buildResult(input: BuildResultInput): EmbeddedRunAttemptResult;
}
```

消息处理：

| SDK message | 行为 |
| --- | --- |
| `assistant` | 提取 text blocks，更新 `assistantTexts`、`lastAssistant`、usage |
| `partial_assistant` | 刷新 liveness；可选发 reasoning/progress，不进 transcript |
| `result.success` | terminal success，保存 result text、stop reason、usage |
| `result.error_*` | terminal error，保存 errors |
| `system` | 保存 init/session 信息，发 lifecycle event |
| `status` | 发 status event |
| `tool_progress` | 发 tool event，更新 tool meta |
| `tool_use_summary` | 发 tool summary event |
| `local_command_output` | 发 command output event，截断/redact |
| `hook_started/progress/response` | 发 native hook event |
| `auth_status` | 发 auth diagnostic，不能包含 secret |
| `task_*` | 发 task diagnostic event |
| `compact_boundary` | 发 compaction event |
| `rate_limit` | 缓存/发 rate limit event |
| `prompt_suggestion` | 发 diagnostic |

Assistant text 提取：

- 只取 `message.content` 中 `type === "text"` 的 block。
- 多个 assistant message 按顺序累计。
- terminal `result.result` 仅作为 fallback assistant text；如果已有 assistant text，不重复追加。

Result 构造：

- `messagesSnapshot` 至少包含当前 user prompt 和 terminal assistant。
- `agentHarnessResultClassification` 使用 `classifyAgentHarnessTerminalOutcome(...)`。
- `attemptUsage` 使用 `normalizeUsage(...)` 从 SDK usage/modelUsage 转换。
- `replayMetadata.hadPotentialSideEffects` 来自 tool telemetry、permission approval、native tool summary。

## 20. Usage 映射

`usage.ts` 将 SDK usage 转成 OpenClaw `NormalizedUsage`。

输入来源：

- assistant `BetaMessage.usage`
- result `usage`
- result `modelUsage`

优先级：

1. result `usage`
2. assistant 最新 `message.usage`
3. modelUsage 汇总

映射字段：

- input tokens
- output tokens
- cache read/write tokens，如 SDK usage 包含
- total tokens

无法映射的字段保留在 diagnostic/trajectory，不塞进 `NormalizedUsage` 的未知字段。

## 21. Session binding 设计

`session-binding.ts`：

```ts
export type ClaudeAgentSdkSessionBinding = {
  version: 1;
  claudeSessionId: string;
  sessionFile: string;
  cwd: string;
  modelId: string;
  harnessId: "claude";
  createdAt: string;
  updatedAt: string;
  lastMessageUuid?: string;
};
```

存储位置建议：

```text
<sessionFile>.claude-agent-sdk.json
```

理由：

- 和 Codex `sessionFile` 绑定同类，reset/delete 容易清理。
- 不依赖 bundled-only plugin state。
- 便于测试和人工排查。

安全读写：

- 只允许 session file sibling 路径。
- JSON 最大字节数限制，例如 64KB。
- 写入使用 temp file + rename。
- binding 中不存 secret。
- 解析失败时忽略并重建，不阻断 fresh run；同时发 diagnostic。

Resume 判定：

```ts
export function shouldResumeClaudeSession(binding, ctx): boolean {
  if (!binding) return false;
  if (binding.harnessId !== "claude") return false;
  if (binding.cwd !== ctx.cwd) return false;
  if (ctx.config.persistSession === false) return false;
  if (binding.modelId !== ctx.modelId) return false;
  return true;
}
```

## 22. Transcript mirror 设计

`transcript-mirror.ts` 负责把 Claude SDK turn 的用户可见消息写入 OpenClaw session transcript。

输入：

```ts
type MirrorClaudeTranscriptInput = {
  params: EmbeddedRunAttemptParams;
  binding: ClaudeAgentSdkSessionBinding;
  sdkSessionId: string;
  userPrompt: string;
  assistantText?: string;
  userMessageUuid?: string;
  assistantMessageUuid?: string;
};
```

规则：

- user mirror identity：`${sdkSessionId}:${userMessageUuid ?? "prompt"}:user`
- assistant mirror identity：`${sdkSessionId}:${assistantMessageUuid ?? "assistant"}:assistant`
- 空 assistant 不写 assistant message。
- `NO_REPLY` 仍写 transcript，但 delivery suppression 由 runtimePlan 处理。
- 写入前调用 `runAgentHarnessBeforeMessageWriteHook(...)`。
- 使用 `acquireSessionWriteLock(...)` 保护 session file。
- 写入后 `emitSessionTranscriptUpdate(...)`。

Transcript content：

- user：原始 prompt，必要时标记来源为 `claude-agent-sdk`。
- assistant：最终可见 assistant text。
- tool progress 不写普通 transcript。
- permission denial 如果是 terminal error，可作为 assistant-visible error 由外层 reply 处理，不在 mirror 中制造额外 assistant。

## 23. Abort 与 timeout

三类终止：

| 来源 | 动作 | 结果字段 |
| --- | --- | --- |
| 外部 abortSignal | `query.interrupt()`，失败后 `query.close()` | `aborted: true`, `externalAbort: true` |
| attempt timeout | abort controller abort + interrupt | `timedOut: true`, `promptErrorSource: "prompt"` |
| idle timeout | interrupt + close | `idleTimedOut: true`, `promptErrorSource: "prompt"` |

实现模式：

```ts
const runAbortController = new AbortController();
const abortFromUpstream = () => runAbortController.abort("upstream-abort");
params.abortSignal?.addEventListener("abort", abortFromUpstream, { once: true });

const timeout = setTimeout(() => {
  timedOut = true;
  runAbortController.abort("timeout");
}, params.timeoutMs);
```

Cleanup 顺序：

1. clear timeout/idle timers。
2. remove abort listener。
3. unregister active embedded run。
4. close query/warm handle。
5. close MCP bridge。
6. flush trajectory。
7. release locks。

所有 cleanup 用 `runAgentCleanupStep(...)`，单个 cleanup 失败只记录 warning，不掩盖主结果。

## 24. Active run steer

Claude harness 必须支持 OpenClaw active run steer/cancel。实现优先使用 Claude SDK `streamInput()`；如果 SDK 当前 query 模式不支持 post-start input，则 harness 要 fail closed，并在注册/runtime status 中标记 steer unsupported，而不是把 steer 文本当普通后续 channel reply 丢失。

Active handle：

```ts
const handle = {
  kind: "embedded" as const,
  queueMessage: async (text: string) => {
    if (!query || completed) return false;
    await query.streamInput(toClaudeUserMessageStream(text));
    return true;
  },
  isStreaming: () => !completed,
  isCompacting: () => projector.isCompacting(),
  cancel: () => void query?.interrupt(),
  abort: () => runAbortController.abort("aborted"),
};
```

关键限制：

- 只在 `prompt` 为 async iterable 或 SDK 支持 post-start `streamInput()` 时启用。
- OpenClaw slash commands 不能进入 Claude prompt。
- 同一 session 只允许一个 active Claude query。
- steer 输入要进入 transcript mirror，避免 SDK 有历史但 OpenClaw 看不到。

## 25. Warm startup

Claude SDK `startup()` 可预热 CLI 子进程，返回 `WarmQuery`。

`startupWarm` 是完整设计中的可配置性能优化，默认关闭，原因：

- `WarmQuery.query()` 每个 handle 只能调用一次。
- OpenClaw attempt 的 prompt、cwd、mcpServers、permission bridge 都是请求时才确定。
- 预热池 key 很复杂，错误复用可能导致 workspace/auth/tool 泄露。

启用时按严格 key 复用：

```ts
type WarmQueryKey = {
  cwd: string;
  modelId: string;
  permissionMode: string;
  settingSources: string[];
  executable?: string;
  toolProfileFingerprint: string;
  authEpoch: string;
};
```

只预热无 prompt、无 request-scoped MCP server 的基础进程。若 tool bridge 是 per-run SDK MCP instance，不应复用 warm query，除非 SDK 支持 query-time替换 MCP servers。

## 26. Auth bridge

`auth-bridge.ts` 必须把 OpenClaw auth profile、host env 和 Claude SDK native auth 三者归一成 SDK `Options.env` 与 diagnostic 状态。

模式：

```ts
export type ClaudeAuthMode =
  | { kind: "native" }
  | { kind: "isolated"; configDir: string; homeDir: string }
  | { kind: "api-key"; env: Record<string, string> };
```

解析顺序：

1. 用户显式 auth profile。
2. plugin config auth mode。
3. host env / Claude Code native login。

实现前必须验证的事实：

- Claude SDK/CLI 支持哪些 env var。
- API key env name 是否为 `ANTHROPIC_API_KEY`。
- Claude config dir 是否可通过 env 覆盖。

在未验证前，文档和 doctor 只能说“native Claude Code auth required”，不要承诺 isolated/api-key 可用。

## 27. Binary resolution 与 doctor

SDK 文档说明 optional dependency 可能缺失并抛 `Native CLI binary for <platform> not found`。插件应捕获并转换为可行动错误。

错误分类：

| 错误 | 提示 |
| --- | --- |
| native binary missing | 重新安装插件依赖，确认 optional dependencies 未被禁用，或配置 `pathToClaudeCodeExecutable` |
| executable ENOENT | 检查配置路径或安装 Claude Code |
| auth missing | 运行 Claude Code 登录流程或配置 supported auth profile |
| permission unavailable | 调整 `permissionMode` 或 OpenClaw approval config |
| model unsupported | 运行 discovery 或选择 `claude/sonnet-4.6` |

`managed-binary.ts` 只做错误归一和配置提示，不自己扫描 node_modules 深层路径，除非 SDK 类型要求。

## 28. Compaction

Claude harness 必须把 Claude SDK compaction 事件接入 OpenClaw hook 生命周期，同时保持双轨压缩边界：

- Claude SDK 原生 compaction 只压缩 Claude session 内部上下文。
- OpenClaw context engine compaction/maintenance 只压缩 OpenClaw 侧 transcript mirror、摘要、索引和 context state。
- 两边通过 harness 事件、hooks、diagnostics 和 prompt projection 同步，不互相直接写入内部存储。

若 SDK 提供显式 compact/slash command/session API，`compact()` 使用该 API；若 SDK 只在 query 流中发出 `SDKCompactBoundaryMessage`，则 `compact()` 返回明确的 unsupported 结果，但事件投影仍必须完整运行 before/after compaction hook。

Event projector 处理 `SDKCompactBoundaryMessage`：

- boundary start：调用 `runAgentHarnessBeforeCompactionHook(...)`。
- boundary end：调用 `runAgentHarnessAfterCompactionHook(...)`。
- 记录 `compactionCount`。

`compact(params)` 实现顺序：

1. 读取 `sessionFile -> claudeSessionId` binding。
2. 用 SDK `getSessionInfo()` 验证 session 存在。
3. 探测 `query.supportedCommands()` 或 SDK session API 是否支持 compact。
4. 支持时触发 SDK compact，并围绕调用运行 OpenClaw before/after compaction hooks。
5. 不支持时返回 `{ compacted: false, reason: "unsupported" }` 风格结果，并发 diagnostic。
6. compact 后如 SDK session id 变化，更新 binding；不变化则只更新 `updatedAt` 和 compaction metadata。

不要把 OpenClaw transcript 压缩结果写回 Claude SDK session，除非 SDK 提供官方接口。

OpenClaw context engine 压缩路径：

1. 用户或 runtime 触发 OpenClaw compact/maintenance。
2. Claude harness 读取 OpenClaw mirrored transcript 和 context engine state。
3. 调用 OpenClaw context engine maintenance/compaction，生成 OpenClaw 侧摘要、facts、索引或裁剪状态。
4. 不修改 Claude SDK session store。
5. 下一次 Claude run 的 assembly 只注入 context engine 认为必要的摘要/facts。

原生 Claude compact 路径：

1. Claude SDK 在 query 中自动 compact，或插件通过 SDK compact API 触发。
2. Projector 收到 compact boundary。
3. Harness 触发 OpenClaw before/after compaction hooks。
4. 更新 binding metadata、diagnostics、trajectory。
5. OpenClaw transcript mirror 保持用户可见历史，不跟随原生 compact 删除或重写。

显式 compact 命令的结果必须区分两种状态：

- `openclawContextCompacted: true/false`
- `nativeClaudeSessionCompacted: true/false`

如果 Claude SDK 不支持显式 native compact，但 OpenClaw context engine compact 成功，应返回类似“OpenClaw context compacted; Claude native session unchanged”的结构化结果。

## 29. Context engine

OpenClaw context engine 是 Claude harness 中的 OpenClaw 侧上下文管理层。它由三个动作组成：prompt assembly、finalize、maintenance。

集成点：

- pre-run：`bootstrapHarnessContextEngine(...)`
- prompt assembly：`assembleHarnessContextEngine(...)`
- post-run：`finalizeHarnessContextEngineTurn(...)`
- maintenance：`runHarnessContextEngineMaintenance`

### 29.1 Prompt assembly

Assembly 发生在 Claude SDK `query()` 前。

目标：决定本轮要额外提供给 Claude 的 OpenClaw 可控上下文。

输入：

- 当前用户 prompt。
- OpenClaw mirrored transcript。
- context engine 已维护的摘要、facts、索引或 memory state。
- bootstrap/workspace context。
- 当前 provider/model、token budget、tool names、capability metadata。
- session/channel/run metadata。

输出：

- 本轮最终 prompt projection。
- system/developer instruction addition。
- assembly 后的 messages 或 context facts。
- prePromptMessageCount。
- prompt cache metadata。

Claude harness 使用方式：

- Fresh run 可以把 assembly 生成的上下文和当前 prompt 一起交给 Claude。
- Resume run 不能注入完整 transcript；只能注入新增且必要的 facts/summary。
- Assembly 输出进入 `resolveAgentHarnessBeforePromptBuildResult(...)`，允许 OpenClaw hooks 再做最终改写。

### 29.2 Finalize

Finalize 发生在 Claude SDK terminal result、transcript mirror 和 event projection 之后。

目标：让 OpenClaw context engine 吸收本轮 OpenClaw 可见结果，更新自己的上下文状态。

输入：

- 本轮是否成功、是否 abort/timeout、是否 promptError。
- mirror 后的 messagesSnapshot。
- assistantTexts、tool telemetry、usage、prompt cache。
- sessionId/sessionKey/sessionFile、workspace、agent id。
- compaction metadata。

输出/副作用：

- 更新 OpenClaw 侧摘要或 facts。
- 记录哪些 transcript message 已进入 context state。
- 更新 token budget/cache metadata。
- 标记失败 turn 是否应排除在长期上下文之外。
- 触发必要的 context engine maintenance。

Claude harness 使用方式：

- 只用 OpenClaw mirrored transcript 和 result finalize，不读取或修改 Claude SDK 内部 session JSONL。
- 如果 Claude 原生 compact 在本轮发生，finalize 只记录 compact metadata，不试图反推出 Claude 内部压缩后的上下文。

### 29.3 Maintenance

Maintenance 是 OpenClaw 侧长期维护动作，可在 run 后、compact 后或后台任务中执行。

目标：保持 OpenClaw context state 可控、紧凑、可检索。

可能动作：

- 压缩 OpenClaw transcript 摘要。
- 清理过旧或重复 context records。
- 更新 memory/index。
- 合并重复 facts。
- 修复损坏状态。
- 维护 token budget 和 prompt cache metadata。

Claude harness 使用方式：

- Maintenance 不修改 Claude SDK session store。
- Maintenance 产物只在后续 assembly 中作为摘要/facts 注入。
- 如果 native Claude session 和 OpenClaw context state 对同一历史有不同压缩结果，以 Claude session 为原生执行权威，以 OpenClaw context state 为 OpenClaw 控制面权威。

Context engine 策略：

- Fresh run 使用 context engine assembly。
- Resume run 不自动注入完整历史，只注入新增 contextual facts。
- finalization 使用 mirror transcript 后的 messagesSnapshot。
- maintenance 可以压缩 OpenClaw context state，但不删除 OpenClaw 用户可见 transcript，除非 OpenClaw session 管理层有明确策略。

风险：

- Claude SDK session store 已有历史，OpenClaw context engine 再注入历史会重复。
- prompt cache 稳定性依赖 deterministic context ordering。

测试必须覆盖 fresh/resume 两种 projection。

## 30. Delivery 与 silent payload

Harness result 只返回文本和 telemetry，不直接调用 channel send。

规则：

- `deliveryDefaults.sourceVisibleReplies = "message_tool"`。
- 如果 OpenClaw message tool 已发送，则 `didSendViaMessagingTool = true`，外层避免重复 source reply。
- `NO_REPLY` 由 `params.runtimePlan?.delivery.isSilentPayload(...)` 判断。
- tool media URL 走 OpenClaw 统一 media delivery path。

这保持和 PI/Codex 一致。

## 31. Trajectory 与 diagnostics

`trajectory.ts` 可参考 Codex，并记录完整 runtime 生命周期的结构化事件：

- `session.started`
- `prompt.submitted`
- `sdk.message`，只记录 type/subtype，不记录完整 secret-bearing payload
- `tool.called`
- `tool.completed`
- `permission.requested`
- `hook.started`
- `hook.completed`
- `compaction.started`
- `compaction.completed`
- `session.ended`

Redaction：

- env、headers、auth profile、tool args 中可疑 key 一律 redacted。
- SDK stderr 只保留 tail，限制长度。
- 文件路径按现有 redaction helper 处理。

Diagnostic event：

- SDK unavailable
- auth unavailable
- model discovery failed
- tool schema skipped
- permission denied
- resume binding stale

## 32. Error taxonomy

本地错误类型：

```ts
export type ClaudeAgentSdkErrorCode =
  | "sdk_unavailable"
  | "binary_missing"
  | "auth_unavailable"
  | "model_unavailable"
  | "permission_denied"
  | "tool_schema_unsupported"
  | "query_timeout"
  | "query_aborted"
  | "stream_error"
  | "binding_invalid";
```

`formatClaudeAgentSdkError(error)` 输出用户可读消息。`promptError` 存储 message，不存原始 error object，trajectory 存 normalized error。

重试策略：

- binary/auth/model/config 错误不 retry。
- SDK stream transient error 可由外层模型 fallback 分类处理，但 selected plugin harness failure 不自动回 PI。
- tool execution error 返回 MCP tool error 给 Claude，不终止整个 attempt，除非 hook/policy 要求 terminal。

## 33. Security 模型

默认安全：

- `permissionMode: "default"`。
- `bypassPermissions` 需要双开关。
- MCP bridge 默认只暴露 native-first 后的 OpenClaw 集成工具。
- 无 approval callback 时 write/exec native permission deny。
- 不保存 secret 到 binding、transcript、trajectory。
- 不自动继承 arbitrary local Claude project settings，除非 `settingSources` 包含对应 source。

隔离：

- native auth 可继承 host Claude Code 状态。
- isolated mode 必须按 agent dir 分离 config/home/cache。
- 不在 config 中存真实 token，使用 OpenClaw credentials/auth profile。

## 34. 测试设计

单元测试文件建议：

```text
extensions/claude/index.test.ts
extensions/claude/harness.test.ts
extensions/claude/provider.test.ts
extensions/claude/src/agent-sdk/config.test.ts
extensions/claude/src/agent-sdk/event-projector.test.ts
extensions/claude/src/agent-sdk/dynamic-tools.test.ts
extensions/claude/src/agent-sdk/approval-bridge.test.ts
extensions/claude/src/agent-sdk/session-binding.test.ts
extensions/claude/src/agent-sdk/transcript-mirror.test.ts
extensions/claude/src/agent-sdk/run-attempt.test.ts
```

Fake SDK：

```ts
export type FakeClaudeQueryScript = Array<
  | { type: "message"; message: unknown }
  | { type: "throw"; error: Error }
  | { type: "wait"; ms: number }
>;
```

`test-support.ts` 注入 SDK factory，避免 module mock 重 SDK barrel。

覆盖矩阵：

| 场景 | 断言 |
| --- | --- |
| assistant + success result | assistantTexts、messagesSnapshot、usage |
| error result | promptError、success false |
| SDK throws before session id | replay safe |
| SDK throws after message tool send | replay unsafe |
| abortSignal | interrupt called、externalAbort true |
| timeout | timedOut true、close called |
| stale binding cwd mismatch | fresh run，不传 resume |
| valid binding | options.resume set |
| MCP tool success | result returned、after_tool hook |
| MCP tool error | MCP error content、telemetry no success |
| before_prompt_build hook | prompt/developer instructions rewrite reflected in SDK options |
| LLM input/output hooks | provider/model/prompt/assistant/usage emitted once |
| before_message_write hook | mirror message can be rewritten or blocked |
| compaction hooks | SDK compact boundary triggers before/after compaction hooks |
| before_agent_finalize hook | Stop/SessionEnd path runs finalize hook before agent_end |
| agent_end hook | success/error/duration/messages emitted on all terminal paths |
| permission allow | SDK allow |
| permission deny | SDK deny |
| bypass without allowDangerously | downgraded default |

Contract tests：

- manifest includes `activation.onAgentHarnesses: ["claude"]`。
- package dependency owner check expects SDK dependency in plugin package。
- no extension prod imports from `src/**`。

## 35. Live 验证

本地 targeted：

```bash
corepack pnpm test extensions/claude/src/agent-sdk/run-attempt.test.ts
corepack pnpm test extensions/claude
corepack pnpm exec oxfmt --check --threads=1 extensions/claude
```

Changed gate：

```bash
corepack pnpm check:changed
```

Live：

```bash
OPENCLAW_LIVE_TEST=1 corepack pnpm test extensions/claude
```

Live 前提：

- host 有 Claude Code auth。
- optional native SDK binary installed，或配置 `pathToClaudeCodeExecutable`。
- 测试 workspace 是临时目录，不使用真实用户 repo。
- 不开启 `bypassPermissions`，除非测试只在隔离临时目录内运行。

## 36. 完整实现清单

完整方案一次性包含以下能力，不按阶段拆分：

- 插件包、manifest、entry、harness、provider、provider discovery。
- Claude SDK config parser、auth bridge、binary/readiness diagnostics。
- `runAttempt()` fresh/resume 执行、SDK query lifecycle、abort/timeout/idle timeout。
- OpenClaw tool bridge：当前 attempt 准备出的 OpenClaw tools 必须可被 Claude 调用，并完整执行 before/after tool hooks、tool result middleware、telemetry 和 media delivery 归集。
- OpenClaw hook bridge：prompt build、LLM input/output、tool call、message write、compaction、before-finalize、agent-end 全部接入。
- Claude native SDK hooks 投影：tool、permission、notification、session、compaction、task、stop 等事件进入 OpenClaw diagnostic/lifecycle。
- Approval bridge：Claude 原生工具权限请求进入 OpenClaw approval/policy。
- Session binding、transcript mirror、context engine assemble/finalize。
- Active run steer/cancel、compaction handling、trajectory、structured diagnostics。
- `/claude` command、conversation binding、doctor/status/docs/UI hints。
- Fake SDK 测试、contract tests、targeted tests、live tests。

## 37. 验收标准

完整验收标准：

- `claude` 插件可以被发现并注册 harness/provider，且不会 eager import 或启动 Claude SDK。
- 显式 `agentRuntime.id: "claude"` 的 run 使用 Claude harness。
- fake SDK 和 live SDK 路径都能完成 fresh/resume turn。
- OpenClaw 当前 attempt 的 tools 能通过 Claude SDK MCP bridge 执行；before_tool_call、after_tool_call、tool result middleware、message/media/heartbeat telemetry 均有测试覆盖。
- OpenClaw harness hooks 全部覆盖：prompt build、LLM input/output、tool call、message write、compaction、before-finalize、agent-end。
- Claude 原生权限请求进入 OpenClaw approval bridge，allow/deny/timeout/updated input 都有确定映射。
- valid binding 触发 resume，reset/delete 清理 binding，cwd/model mismatch 不错 resume。
- Transcript mirror 与 Claude session 不重复、不错写，message write hook 可阻止或改写 mirror。
- abort/timeout/idle timeout 不泄漏 query/MCP resources。
- Claude SDK binary/auth/model 错误有清晰 doctor/status 提示。
- 不触碰 core 的 Claude-specific provider/runtime 分支。
- Targeted tests、contract tests 和 live test 通过。

## 38. 实现前必须确认的问题

- Claude SDK `createSdkMcpServer()` 是否能直接接受 JSON Schema，还是必须使用 `tool()` + Zod raw shape。
- Claude SDK isolated auth/config 的官方环境变量是什么。
- `supportedModels()` 是否能在无 prompt 的 query 上稳定调用，或必须 startup/query 初始化后调用。
- `SDKCompactBoundaryMessage` 的 exact lifecycle 是否足够实现 OpenClaw compact hooks。
- Claude `PermissionUpdate` 是否应映射到 OpenClaw 持久 permission policy。
- 是否要让 `anthropic/*` 在 `agentRuntime.id: "auto"` 下被 Claude harness claim。

这些问题必须在实现对应模块前通过 SDK 类型/source 或官方文档确认；无法确认时该模块应 fail closed，并在 doctor/status 中给出明确诊断。
