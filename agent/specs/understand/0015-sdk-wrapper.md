# SDK query wrapper 与 stream-json/NDJSON 通信机制

## 1. 结论

TypeScript Agent SDK 暴露的 `query()` 不是直接在 SDK 进程里调用模型 API。它的运行边界分两层：

```text
SDK query({ prompt, options })
  -> 启动 Claude Code CLI 子进程
  -> 通过 stdin/stdout 做 stream-json / NDJSON 通信
  -> CLI headless 模式接收用户消息和 control 消息
  -> QueryEngine.submitMessage()
  -> src/query.ts 内部 agent loop
  -> 模型流式响应、工具执行、权限请求、MCP、hooks、result
  -> 每个 SDKMessage 作为一行 JSON 写回 SDK host
```

因此需要区分两个 `query()`：

1. **SDK wrapper `query()`**：用户 import 的 `@anthropic-ai/claude-agent-sdk` API。它负责子进程生命周期、stdin/stdout 协议、AsyncGenerator 包装和控制方法。
2. **CLI 内部 `query()`**：`vendor/Claude-code-open/src/query.ts` 里的 agent 主循环。它负责模型调用、工具调用、compact、stop hooks、重试和终止条件。

`vendor/Claude-code-open/src/entrypoints/agentSdkTypes.ts` 中的 `query()` 只是类型 stub，源码里会 `throw new Error('query is not implemented in the SDK')`。真正 npm SDK wrapper 的实现不在这个 TypeScript 源文件中；当前 vendor 源码主要展示 CLI 子进程内部如何处理 SDK 协议和 agent loop。

## 2. 关键源码位置

| 文件 | 作用 |
| --- | --- |
| `specs/reference/claude_code_sdk.md` | SDK API 参考，定义 `query()`、`Query`、`SDKMessage`、`SDKUserMessage` |
| `vendor/Claude-code-open/src/entrypoints/agentSdkTypes.ts` | SDK 对外类型和方法签名 stub |
| `vendor/Claude-code-open/src/cli/print.ts` | CLI headless / `--print` / `stream-json` 主入口 |
| `vendor/Claude-code-open/src/cli/structuredIO.ts` | stdin/stdout 上的 SDK NDJSON 协议实现 |
| `vendor/Claude-code-open/src/cli/ndjsonSafeStringify.ts` | 一行一条 JSON 的安全序列化 |
| `vendor/Claude-code-open/src/QueryEngine.ts` | SDK/headless 会话包装层，把内部消息转成 `SDKMessage` |
| `vendor/Claude-code-open/src/query.ts` | 真正的 agent query loop |
| `vendor/Claude-code-open/src/query/deps.ts` | `query()` 的生产依赖，模型调用为 `queryModelWithStreaming` |

## 3. stream-json 与 NDJSON

`stream-json` 是 CLI 的输出格式名；实际传输 framing 是 NDJSON。

NDJSON 的规则是：

```text
一行 = 一个完整 JSON 对象
换行符 \n = 消息边界
```

示例：

```text
{"type":"system","subtype":"init","session_id":"..."}\n
{"type":"assistant","message":{"role":"assistant","content":[...]}}\n
{"type":"result","subtype":"success","result":"..."}\n
```

它不是一个整体 JSON 数组，也不是用逗号连接的 JSON stream。接收方必须按行切分，每行单独 `JSON.parse()`。

### 3.1 输出方向

CLI 向 SDK host 写 stdout：

```text
Claude CLI stdout -> SDK host reader -> AsyncGenerator<SDKMessage>
```

`StructuredIO.write()` 会执行：

```ts
writeToStdout(ndjsonSafeStringify(message) + '\n')
```

也就是说，任何 `StdoutMessage` 最终都会序列化为一行 JSON。

`ndjsonSafeStringify()` 会额外转义 `U+2028` 和 `U+2029`。这两个字符是合法 JSON 字符，但某些 JavaScript 分行逻辑会把它们当 line terminator，导致一条 JSON 被错误切断。转义后仍是等价 JSON，但不会破坏按行切分。

### 3.2 输入方向

SDK host 向 CLI 写 stdin：

```text
SDK host writer -> Claude CLI stdin -> StructuredIO.read()
```

`StructuredIO.read()` 的处理流程：

```text
读取任意大小的字符串块
  -> 追加到 content 缓冲区
  -> 查找 \n
  -> 切出一行
  -> JSON.parse(line)
  -> normalizeControlMessageKeys()
  -> 按 type 分发
```

如果 `prompt` 是普通字符串，CLI 侧会被包装为一条 `SDKUserMessage`：

```json
{
  "type": "user",
  "session_id": "",
  "message": {
    "role": "user",
    "content": "用户提示"
  },
  "parent_tool_use_id": null
}
```

如果 `prompt` 是 `AsyncIterable<SDKUserMessage>`，SDK wrapper 会持续把用户消息写入 stdin。CLI 不需要等输入流结束才开始工作；它会边读边 enqueue。

## 4. 协议消息类型

同一条 NDJSON 通道里混合两类消息：

1. **事件流消息**：返回给 SDK 用户的 `SDKMessage`。
2. **控制协议消息**：SDK host 与 CLI 子进程之间的 RPC 往返。

### 4.1 事件流消息

常见事件：

```text
system/init
assistant
user replay
stream_event
tool_use_summary
status
task_started / task_progress / task_notification
result
```

`result` 是一个用户 turn 的终止消息，但不是整个进程一定退出的唯一条件。在流式输入模式下，子进程可以继续等待后续 user message 或 control request。

### 4.2 控制协议消息

控制请求形态：

```json
{
  "type": "control_request",
  "request_id": "uuid",
  "request": {
    "subtype": "can_use_tool",
    "tool_name": "Bash",
    "input": {},
    "tool_use_id": "toolu_..."
  }
}
```

控制响应形态：

```json
{
  "type": "control_response",
  "response": {
    "request_id": "uuid",
    "subtype": "success",
    "response": {
      "behavior": "allow",
      "toolUseID": "toolu_..."
    }
  }
}
```

`request_id` 是配对键。`StructuredIO.sendRequest()` 发送请求时会把 pending request 存入 `Map<requestId, PendingRequest>`；stdin reader 收到 `control_response` 后查表 resolve 或 reject 对应 Promise。

常见 control subtype：

| subtype | 方向 | 作用 |
| --- | --- | --- |
| `initialize` | SDK -> CLI | 初始化 SDK 会话能力、MCP server、prompt suggestion 等 |
| `interrupt` | SDK -> CLI | abort 当前 turn |
| `end_session` | SDK -> CLI | 结束输入和会话 |
| `set_permission_mode` | SDK -> CLI | 修改权限模式 |
| `set_model` | SDK -> CLI | 修改主循环模型 |
| `set_max_thinking_tokens` | SDK -> CLI | 修改 thinking 配置 |
| `mcp_status` | SDK -> CLI | 查询 MCP server 状态 |
| `mcp_message` | 双向 | SDK MCP server 与 CLI MCP client 间转发 JSON-RPC |
| `mcp_set_servers` | SDK -> CLI | 动态设置 MCP server |
| `mcp_reconnect` / `mcp_toggle` | SDK -> CLI | 重连或启停 MCP server |
| `rewind_files` | SDK -> CLI | 文件 checkpoint 回滚 |
| `seed_read_state` | SDK -> CLI | 注入已读文件状态，帮助 Edit 权限和 diff |
| `stop_task` | SDK -> CLI | 停止后台任务 |
| `hook_callback` | CLI -> SDK | 让 SDK host 执行 hook callback |
| `elicitation` | CLI -> SDK | MCP elicitation / 用户澄清 |
| `can_use_tool` | CLI -> SDK | 工具权限请求 |
| `control_cancel_request` | CLI -> SDK | 取消已发出的 control request |

## 5. CLI 侧并发模型

`print.ts` 在 headless SDK 模式里有两个并发循环：

```text
stdin loop
  -> for await structuredIO.structuredInput
  -> 处理 control_request / control_response
  -> user message 入队
  -> 触发 run()

runner loop
  -> drain command queue
  -> ask()
  -> QueryEngine.submitMessage()
  -> 内部 query()
  -> 输出 SDKMessage 到 structuredIO.outbound
```

`StructuredIO.outbound` 是统一输出队列。`sendRequest()` 和 `print.ts` 都只向这个队列 enqueue，由 drain loop 写 stdout。这样可以避免 `control_request`、模型事件、result 之间发生不可控插队。

用户消息不会直接调用模型。它先进入 `messageQueueManager`，然后 `run()` 从队列取出 prompt，调用 `ask()`。

## 6. 从用户 prompt 到 result 的完整路径

```text
SDK host
  -> stdin: SDKUserMessage NDJSON

StructuredIO.read()
  -> processLine()
  -> type=user

print.ts stdin loop
  -> 去重 uuid
  -> resolveAndPrepend()
  -> enqueue({ mode: 'prompt', value, uuid })
  -> run()

runHeadlessStreaming.run()
  -> drainCommandQueue()
  -> ask({ prompt, tools, mcpClients, canUseTool, ... })

QueryEngine.submitMessage()
  -> processUserInput()
  -> 构建 systemPrompt / userContext / ToolUseContext
  -> yield system init
  -> for await internal query(...)

src/query.ts queryLoop()
  -> compact / microcompact / context collapse
  -> queryModelWithStreaming()
  -> yield assistant stream
  -> 收集 tool_use
  -> runTools() 或 StreamingToolExecutor
  -> yield tool_result
  -> 如果需要 follow-up，下一轮模型调用
  -> 没有 follow-up 时 stop hooks / budget / result

QueryEngine
  -> normalize internal Message 为 SDKMessage
  -> 统计 usage/cost/turn count
  -> yield SDKResultMessage

print.ts
  -> structuredIO.write(message)
  -> stdout: 每条 SDKMessage 一行 JSON

SDK host
  -> 按行 parse stdout
  -> AsyncGenerator yield 给用户
```

## 7. 工具权限的往返

工具权限是理解 SDK wrapper 的关键例子。

当模型输出 `tool_use`，内部 `query.ts` 会执行工具。执行前会调用 `canUseTool`。在 SDK/headless 模式下，`canUseTool` 来自 `StructuredIO.createCanUseTool()`。

流程：

```text
模型输出 tool_use
  -> query.ts 执行工具前调用 canUseTool
  -> StructuredIO.createCanUseTool()
  -> 先检查本地权限规则和 hooks
  -> 如果需要 SDK host 决策：
       stdout 写 control_request can_use_tool
       pendingRequests[request_id] = Promise
  -> SDK host 展示权限 UI 或自动决策
  -> stdin 写 control_response
  -> StructuredIO.processLine() resolve Promise
  -> 工具执行继续或被拒绝
  -> tool_result 作为 user message 回到 query loop
```

这里同一条 stdout 既会输出 assistant 消息，也会输出 `control_request`。SDK wrapper 必须能区分：

- `assistant` / `result` 等是要 yield 给用户的事件。
- `control_request` 是 SDK wrapper 自己要处理的内部 RPC。

## 8. MCP SDK server 的转发

SDK 可以通过 `createSdkMcpServer()` 在 host 进程内定义 MCP server。CLI 子进程不能直接调用 host 进程内函数，因此也通过 control 协议桥接。

典型路径：

```text
模型调用 MCP tool
  -> CLI 内部 MCP client 需要发 JSON-RPC
  -> StructuredIO.sendMcpMessage()
  -> stdout: control_request { subtype: "mcp_message", server_name, message }
  -> SDK host 收到后投递给本进程 MCP server
  -> SDK host stdin 回 control_response { mcp_response }
  -> CLI 继续 MCP tool result
```

这解释了为什么 SDK wrapper 不能只是“读 stdout 并 yield”：它还必须维护一套双向 control RPC。

## 9. `startup()` / WarmQuery 的意义

文档中的 `startup()` 会提前启动 CLI 子进程并完成初始化握手。之后调用 `warm.query(prompt)` 时，只需要把 prompt 写入已经就绪的 stdin。

它优化的是 SDK wrapper 层成本：

```text
普通 query:
  spawn CLI
  load settings/plugins/MCP
  initialize
  write prompt
  first token

startup + warm.query:
  spawn CLI
  load settings/plugins/MCP
  initialize
  等 prompt
  write prompt
  first token
```

内部 agent loop 没有因为 warm 而变简单；只是子进程启动和初始化不再压在首个 prompt 的关键路径上。

## 10. 实现上的注意点

1. **必须按行解析 stdout**  
   不要尝试用一次 `JSON.parse(buffer)` 解析整个 stdout。`stream-json` 是 NDJSON。

2. **必须处理半包**  
   stdin/stdout 的 chunk 边界不等于消息边界。需要缓冲到 `\n` 才能 parse。

3. **不要丢 control_request**  
   `control_request` 不一定要暴露给用户，但 SDK wrapper 必须处理，否则权限、MCP、hooks、elicitation 会挂起。

4. **control_response 必须携带正确 request_id**  
   CLI 侧依赖 `request_id` resolve pending Promise。错配会导致请求一直 pending 或被当作 orphan response。

5. **一条 query 可能包含多轮模型调用**  
   用户看到的是一个 `query()`，内部可能是：

   ```text
   assistant tool_use -> tool_result -> assistant tool_use -> tool_result -> assistant final
   ```

6. **`result` 是 turn 结果，不等同于所有后台活动都结束**  
   `print.ts` 对后台任务有 hold-back 和 task notification 逻辑。SDK consumer 应按 `origin`、task event、session state 区分主 prompt 结果和后台任务结果。

7. **中断是 control 协议，不是杀进程优先**  
   `interrupt()` 通常应发送 control request，让 CLI abort 当前 `AbortController` 并产出可恢复的终止事件。直接 kill 子进程会绕过 transcript flush 和 cleanup。

8. **`Query` 方法大多是 control_request 包装**  
   `setModel()`、`setPermissionMode()`、`mcpServerStatus()`、`rewindFiles()`、`stopTask()` 等都可以理解为向子进程发送特定 control request，然后等待 control response。

## 11. 简化时序图

```text
SDK host                         Claude CLI subprocess
   |                                      |
   | stdin: user message                  |
   |------------------------------------->|
   |                                      | enqueue prompt
   |                                      | build QueryEngine context
   | stdout: system/init                  |
   |<-------------------------------------|
   |                                      | call model
   | stdout: assistant tool_use           |
   |<-------------------------------------|
   | stdout: control_request can_use_tool |
   |<-------------------------------------|
   | stdin: control_response allow        |
   |------------------------------------->|
   |                                      | execute tool
   | stdout: user tool_result replay      |
   |<-------------------------------------|
   |                                      | call model again
   | stdout: assistant final text         |
   |<-------------------------------------|
   | stdout: result success               |
   |<-------------------------------------|
```

## 12. 对本项目实现 SDK wrapper 的启示

如果本项目要自己实现或替换 SDK wrapper，最低可行职责不是“spawn 后把 stdout yield 出来”，而是：

1. 启动和管理 Claude CLI 子进程。
2. 将用户 prompt / streaming input 编码为 NDJSON 写 stdin。
3. 从 stdout 按行解析 NDJSON。
4. 区分 SDK event 与 control message。
5. 为 control request 实现 request/response 路由。
6. 支持 permission、MCP、hook、elicitation、interrupt、end_session。
7. 将普通 SDK event 包装成 `AsyncGenerator<SDKMessage>`。
8. 在 `close()` / `interrupt()` / 进程退出时处理 pending Promise 和资源清理。

这也是 `query()` 能表现成一个普通 async generator，但内部仍能支持权限弹窗、MCP in-process server、动态设置、流式输入和多轮工具调用的核心原因。
