# `claude.ts` 差异兼容整理

## 模型差异

| 具体差异 | 兼容处理策略 |
|---|---|
| Thinking / Adaptive Thinking | 不支持就不传；支持 adaptive 就走 `adaptive`；否则走带 `budget_tokens` 的普通 thinking。 |
| Structured Outputs | 仅支持的模型才附加结构化输出格式和对应 beta 配置。 |
| Effort | 仅支持的模型才传 effort；不支持则跳过。 |
| Advisor | 仅支持 advisor 的模型才启用 advisor beta、指令和 server tool。 |
| Tool Search / Deferred Tools | 不支持的模型会移除 ToolSearch 相关能力，并清理相关消息字段；支持时才启用 `defer_loading`。 |
| Fast Mode | 仅支持的模型才启用 `speed: 'fast'`。 |
| 最大输出 Token | 按模型选择默认输出上限和允许上限，并在重试时动态调整。 |
| Sonnet 1M 上下文实验 | 命中实验的模型才追加 1M context 配置。 |
| Prompt Caching | 按模型类型允许单独关闭 small/fast、Sonnet、Opus 的 prompt caching。 |

## Provider 差异

| 具体差异 | 兼容处理策略 |
|---|---|
| Bedrock inference profile | 先解析成实际 backing model，再按真实模型能力处理。 |
| Bedrock beta 参数格式 | 部分 beta 参数不走普通 `betas`，而是写入 extra body。 |
| Cache Editing | 仅 first-party 主线程请求启用。 |
| Opus off-switch | 对非订阅用户的特定 Opus 路径做额外开关限制。 |

