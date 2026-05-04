# `claude.ts` 模型访问可靠性策略

## 请求可靠性

| 策略类别 | 触发场景 | 处理策略 | 目的 |
|---|---|---|---|
| 统一重试入口 | 流式请求和非流式 fallback 请求 | 两条路径都通过 `withRetry` 驱动，请求重试、模型 fallback 信号和重试上下文统一收口。 | 避免重试逻辑分散，保证故障处理一致。 |
| 手动接管 SDK 重试 | 主流式请求创建 client 时 | 显式关闭 client 自带自动重试，改由本地策略统一控制。 | 避免 SDK 与上层重试叠加，减少重复请求和状态不一致。 |
| 流式空闲 watchdog | 流建立后长时间没有 chunk | 启动定时器；先 warning，再主动中止流并释放资源。 | 防止 SSE 连接静默卡死。 |
| 流式 stall 监控 | chunk 间隔过长但尚未完全卡死 | 记录 stall 次数和累计 stall 时间，但不中断当前流。 | 监控退化链路，保留继续成功的机会。 |
| 流式协议完整性校验 | 收到 `content_block_delta/stop` 时找不到对应 block，或 block 类型不匹配 | 直接抛错并记录 streaming error。 | 避免把损坏的流拼成错误消息。 |
| 模型 fallback 信号上抛 | `withRetry` 触发 `FallbackTriggeredError` | 不在 `claude.ts` 内吞掉，继续抛给 `query.ts` 执行真正的换模型重试。 | 保证模型切换真的发生，而不是只显示错误提示。 |

## 降级与 Fallback

| 策略类别 | 触发场景 | 处理策略 | 目的 |
|---|---|---|---|
| 非流式 fallback 超时上限 | streaming 失败后切 non-streaming | 非流式 fallback 请求附带独立 timeout，上限默认 120s/300s，可环境变量覆盖。 | 防止 fallback 本身挂死。 |
| 无有效事件兜底 | 流结束但没有 `message_start`，或没有完整内容块且也没有 `stop_reason` | 视为代理/网关异常，触发 non-streaming fallback。 | 处理 200 但不是有效 SSE、半截流等异常。 |
| 404 流创建兜底 | streaming endpoint 创建阶段返回 404 | 不把它当成最终失败，而是直接切到 non-streaming fallback。 | 兼容只支持非流式的网关实现。 |
| streaming -> non-streaming 自动降级 | 流式消费过程中抛错 | 默认记录错误并切换为 non-streaming fallback。 | 优先拿到有效结果，而不是直接失败。 |
| 可关闭自动降级 | 已知流式 fallback 可能导致重复 tool 执行的环境 | 支持通过环境变量/特性开关禁用 non-streaming fallback，让错误直接向上抛。 | 在正确性比可用性更重要的场景避免重复副作用。 |
| 529 连续错误预算继承 | streaming 因 529 失败后切 non-streaming | 把 streaming 的一次 529 计入后续 non-streaming 的连续 529 预算。 | 保持“多少次过载后切模型”的策略一致。 |
| 请求参数非流式裁剪 | fallback 到 non-streaming 且 `max_tokens` 过大 | 对 non-streaming 参数做 capped 调整，并同步修正 thinking budget。 | 满足 API 约束，避免 fallback 再因参数非法失败。 |

## 错误响应

| 策略类别 | 触发场景 | 处理策略 | 目的 |
|---|---|---|---|
| 用户中断与超时区分 | 捕获 `APIUserAbortError` 时 | 若 `signal.aborted` 为真，视为用户中断；否则改写为连接超时错误。 | 区分用户操作和链路故障，避免误判。 |
| 用户中断静默返回 | 用户主动取消请求 | 不产出 assistant error message，交给上层中断逻辑处理。 | 避免把用户取消显示成模型故障。 |
| stop reason 错误翻译 | `stop_reason` 为 `max_tokens` 或 `model_context_window_exceeded` | 产出统一的 assistant API error message，映射到 `max_output_tokens` 恢复链路。 | 让上层复用同一套“续写/恢复”逻辑。 |
| refusal 显式转消息 | 模型拒答类 stop reason | 生成可展示的 refusal message。 | 让拒答成为结构化结果，而不是隐式失败。 |
| 资源释放保障 | 流结束、fallback 失败、用户中断、生成器被提前 `.return()` | 统一调用 `releaseStreamResources()` / `cleanupStream()`，取消 stream 和 response body。 | 防止 socket/TLS/native buffer 泄漏。 |
| fallback 成本补记 | 走 non-streaming fallback 成功返回 | 在 `finally` 中补记 usage、stopReason 和 cost。 | 防止 fallback 路径遗漏计费与统计。 |
| request id 关联 | streaming / fallback / error 路径 | 同时记录 server request id 与 client request id，必要时从 error header/body 里回填。 | 便于把失败、重试、fallback 串成同一条调用链。 |
| 配额状态提取 | 成功响应头或 APIError | 从 headers / error 中提取 quota 状态。 | 在限额和过载场景下保留额外诊断信息。 |
