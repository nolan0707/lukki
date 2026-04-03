# Claude-code-open 观测与实验架构深度解读（专家版）

## 1. 文档目标

本文在 [0001-project-arch.md](./0001-project-arch.md) 和 [0008-observable-arch.md](./0008-observable-arch.md) 的基础上，进一步从 AI 应用与算法系统专家视角，深度剖析 `vendor/Claude-code-open` 的“观测与实验架构”。

本文不再停留在“有哪些模块”的层面，而是重点回答：

1. 这套观测与实验架构的控制面与数据面如何分层？
2. 为什么同一个项目同时维护 `analytics`、`1P event logging`、`OpenTelemetry telemetry`、`Profiler`、`GrowthBook` 几条系统？
3. 这些模块之间的接口边界、状态管理、调用时序、约束条件分别是什么？
4. 哪些设计明显体现出工业级 agent/runtime 系统的工程经验？
5. 如果要迁移、扩展或复用这套架构，哪些不变量不能破坏？

本文基于以下源码阅读整理：

- `src/services/analytics/index.ts`
- `src/services/analytics/sink.ts`
- `src/services/analytics/growthbook.ts`
- `src/services/analytics/firstPartyEventLogger.ts`
- `src/services/analytics/firstPartyEventLoggingExporter.ts`
- `src/services/analytics/metadata.ts`
- `src/services/analytics/datadog.ts`
- `src/services/analytics/config.ts`
- `src/utils/telemetry/instrumentation.ts`
- `src/utils/telemetry/events.ts`
- `src/utils/telemetry/sessionTracing.ts`
- `src/utils/telemetryAttributes.ts`
- `src/utils/startupProfiler.ts`
- `src/utils/queryProfiler.ts`
- `src/bootstrap/state.ts`
- `src/entrypoints/init.ts`
- `src/main.tsx`
- `src/services/api/logging.ts`
- `src/services/tools/toolExecution.ts`
- `src/components/permissions/hooks.ts`

---

## 2. 一句话结论

这套“观测与实验架构”不是传统意义上的“埋点子系统”，而是一个覆盖运行时控制面、事件数据面、追踪面、性能剖析面和隐私/信任约束面的综合基础设施层。

它有两个最核心的架构特征：

- 观测系统直接参与 runtime decision，而不是只做离线记录。
- 内部分析链路与客户遥测链路被强制分离，但又共享一部分元数据与初始化基础设施。

如果压缩成一张抽象图，可以表示为：

```text
                    控制面
  GrowthBook / Privacy / Trust / Auth / Dynamic Config
            |             |              |
            v             v              v
     +-------------------------------------------+
     |         Runtime Observability Core        |
     |                                           |
     |  analytics/index.ts   telemetry/events.ts |
     |  startupProfiler      queryProfiler       |
     |  sessionTracing       bootstrap/state     |
     +-------------------------------------------+
            |                  |               |
            v                  v               v
     内部产品分析面         客户标准遥测面      本地剖析面
   Datadog / 1P Event     OTLP logs/metrics    Perfetto / report
   Logging / BQ metrics   traces / exporter    files / console
```

换句话说，这个项目把：

- feature flag
- dynamic config
- privacy gate
- trust gate
- event ingestion
- tracing
- profiling
- reliability buffering

统一纳入了 agent runtime 的内核层。

---

## 3. 系统分层：控制面与数据面

专家视角下，这套系统最适合按“控制面 / 数据面”理解。

## 3.1 控制面

控制面负责决定：

- 哪些观测链路允许开启
- 哪些特性是否激活
- 采样率、批量参数、刷新频率是多少
- 是否允许使用 trust-sensitive auth headers
- 当前隐私级别是否压制某些网络流量

核心模块：

- `src/services/analytics/growthbook.ts`
- `src/services/analytics/config.ts`
- `src/utils/privacyLevel.ts`
- `src/utils/telemetryAttributes.ts`
- `src/bootstrap/state.ts`

核心控制信号：

- env var
- local config override
- disk cache
- GrowthBook remote eval
- trust established / not established
- auth available / expired / scoped insufficient

## 3.2 数据面

数据面负责：

- 收集事件
- 富化上下文
- 路由到不同 sink
- 序列化为后端可接受 payload
- 做批量发送、失败重试与 flush

核心模块：

- `src/services/analytics/index.ts`
- `src/services/analytics/sink.ts`
- `src/services/analytics/firstPartyEventLogger.ts`
- `src/services/analytics/firstPartyEventLoggingExporter.ts`
- `src/services/analytics/datadog.ts`
- `src/utils/telemetry/instrumentation.ts`
- `src/utils/telemetry/events.ts`

## 3.3 剖析面

剖析面面向：

- 启动期性能分析
- 单次 query 路径分析
- 交互级 span tracing

核心模块：

- `src/utils/startupProfiler.ts`
- `src/utils/queryProfiler.ts`
- `src/utils/telemetry/sessionTracing.ts`
- `src/utils/telemetry/perfettoTracing.ts`

这个分层很重要，因为它解释了为什么工程上要存在多套“观测”能力而不是一套大而全系统。

---

## 4. 顶层设计原则

从源码可以提炼出 8 条非常稳定的设计原则。

## 4.1 启动路径绝不为了观测阻塞主功能

体现：

- `analytics/index.ts` 的 queued events
- `init.ts` 对 1P logger / telemetry 的动态 import
- GrowthBook 大量提供 `CACHED_MAY_BE_STALE` 读接口
- sink attach 后 `queueMicrotask()` drain

这是 CLI / agent 系统的第一原则：用户输入和模型循环优先，观测后置但尽量保真。

## 4.2 隐私边界先于观测完整性

体现：

- `_PROTO_*` 字段剥离与定向 hoist
- MCP tool name 默认脱敏
- skill / plugin / marketplace 名称分级处理
- privacy level 可全局关闭非必要遥测

这表明作者将“可观测性”放在“隐私合规”之后，而不是反过来。

## 4.3 内部分析与客户遥测必须物理分离

体现：

- 1P event logging 自建 `LoggerProvider`
- 客户 OTLP telemetry 独立 `LoggerProvider` / `TracerProvider`
- `setEventLogger()` 只绑定客户侧 logger
- `firstPartyEventLogger` 不注册为 global provider

这是防止数据泄漏和语义混淆的硬边界。

## 4.4 动态配置是运行时热控制，而不是启动时常量

体现：

- `onGrowthBookRefresh()`
- `reinitialize1PEventLoggingIfConfigChanged()`
- periodic refresh
- auth-change reinit

这意味着“长会话”是架构默认假设，而不是例外。

## 4.5 可靠性问题优先靠本地缓冲解决

体现：

- 事件队列
- failed event JSONL
- next-run retry
- backoff scheduling
- flush hooks

这与 server 常见的“交给 Kafka 再说”不同，符合本地 agent/CLI 的运行环境约束。

## 4.6 类型系统承担一部分数据治理职责

体现：

- `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`
- `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`
- `EnvironmentMetadata` proto-generated typing
- `FirstPartyEventLoggingMetadata`

这说明数据治理不是靠文档提醒，而是试图前移到编译期。

## 4.7 trace 不是辅助功能，而是 agent runtime 结构镜像

`interaction -> llm_request -> tool -> blocked_on_user` 的 span 结构并不是“观测方便起见”的设计，而是对 agent execution graph 的直接映射。

## 4.8 观测参数本身也是实验对象

最典型例子：

- event sampling config 由 GrowthBook 控制
- 1P batch config 由 GrowthBook 控制
- sink kill switch 由 GrowthBook 控制

这意味着该系统不仅“观察业务”，还“观察观测自身”。

---

## 5. 初始化架构：谁先启动，谁后启动，为什么

## 5.1 初始化分三阶段

### 阶段 A：最早期 gate warmup

`src/main.tsx`

关键动作：

- `initializeAnalyticsGates()`

作用：

- 尽早读取 Datadog gate 的 cached 值
- 让早期事件在 sink attach 后能快速按缓存 gate 决策

这一步是轻量同步路径，几乎不引入额外重包。

### 阶段 B：初始化 1P internal logging

`src/entrypoints/init.ts`

关键动作：

- 动态 import `firstPartyEventLogger.js`
- 动态 import `growthbook.js`
- `initialize1PEventLogging()`
- `onGrowthBookRefresh(() => reinitialize1PEventLoggingIfConfigChanged())`

作用：

- 将内部事件日志链接入 runtime
- 但不把 OpenTelemetry sdk-logs 提前塞进 CLI 冷启动关键路径

### 阶段 C：初始化客户 OTLP telemetry

`src/entrypoints/init.ts` -> `setMeterState()` -> `initializeTelemetry()`

作用：

- 建 meter / logger / tracer provider
- 注册全局 event logger
- 初始化 BigQuery metrics exporter
- 初始化 optional traces

## 5.2 为什么 1P event logging 比 telemetry 更早

这是一个非常关键的架构取舍。

原因有三：

1. 内部分析是产品自运维能力，优先级高于客户自定义 telemetry。
2. 1P logging 还被 GrowthBook 依赖，GrowthBook 又承担 feature control。
3. 客户 telemetry 更重、更可选、更依赖用户环境配置。

换句话说：

- GrowthBook 和 1P event logging 更像 runtime control infrastructure。
- OTLP telemetry 更像 extensible export infrastructure。

## 5.3 为什么 telemetry state 存在 `bootstrap/state.ts`

`bootstrap/state.ts` 持有：

- `meter`
- `sessionCounter`
- `loggerProvider`
- `eventLogger`
- `meterProvider`
- `tracerProvider`

并提供接口：

- `setMeter()`
- `getSessionCounter()`
- `setLoggerProvider()`
- `setEventLogger()`
- `setMeterProvider()`
- `setTracerProvider()`

这说明作者将 telemetry provider state 视为全局 runtime state，而不是局部依赖注入对象。

优点：

- 任意业务层都能快速取到 counter / logger
- 无需层层传参

代价：

- 全局状态约束更强
- 初始化顺序必须更严格

这是典型 CLI/agent 项目中的工程性权衡。

---

## 6. `analytics/index.ts`：事件接入面的最小内核

这个文件的职责极其克制：

- 定义 `AnalyticsSink`
- 定义队列
- 提供 `logEvent()` / `logEventAsync()`
- 提供 `attachAnalyticsSink()`
- 提供 `_PROTO_*` 剥离工具

它有三个重要接口。

## 6.1 接口一：`AnalyticsSink`

```ts
export type AnalyticsSink = {
  logEvent: (eventName: string, metadata: LogEventMetadata) => void
  logEventAsync: (
    eventName: string,
    metadata: LogEventMetadata,
  ) => Promise<void>
}
```

设计含义：

- 业务代码与具体 sink 后端解耦
- 支持同步与异步两种契约
- 上层无需感知 Datadog、1P、采样、脱敏等逻辑

## 6.2 接口二：`attachAnalyticsSink()`

语义：

- 幂等 attach
- attach 后 drain 之前排队的事件

约束：

- 若 sink 已存在，则直接 no-op
- drain 通过 `queueMicrotask()` 异步进行

这个幂等性很关键，因为多个入口可能都会尝试初始化 sink。

## 6.3 接口三：`stripProtoFields()`

语义：

- 删除 metadata 中以 `_PROTO_` 开头的字段

设计价值：

- 保证通用 sink 看不到 privileged 字段
- 统一防线，避免每个 sink 各写一份过滤逻辑

这一点很有工业味道，因为它避免了“新增 sink 时忘记过滤”的常见事故。

---

## 7. `analytics/sink.ts`：内部分析路由器

这个模块把统一事件路由到：

- Datadog
- 1P event logging

## 7.1 核心路由算法

伪代码可概括为：

```text
on event:
  sampleResult = shouldSampleEvent(event)
  if sampleResult == 0:
    drop
  metadata' = add sample_rate if needed

  if shouldTrackDatadog():
    trackDatadogEvent(stripProtoFields(metadata'))

  logEventTo1P(metadata')
```

这个顺序不是随意的。

原因：

- 采样要在 fanout 前完成，保证多个 sink 的事件选择一致
- Datadog 先 strip `_PROTO_*`
- 1P 保留 privileged payload

## 7.2 Gate warmup 策略

`initializeAnalyticsGates()` 会提前读取：

- `tengu_log_datadog_events`

并缓存到模块变量 `isDatadogGateEnabled`。

设计意图：

- 让早期事件不必等待完整 GrowthBook init
- 允许使用上次 session 缓存值

这是一个标准的“control plane cache bootstrap”策略。

## 7.3 Sink kill switch

`sink.ts` 还依赖：

- `isSinkKilled('datadog' | 'firstParty')`

其配置来自 GrowthBook dynamic config：

- `tengu_frond_boric`

语义：

- 按 sink 维度熔断
- fail-open：缺失/异常时默认不关闭

这是一种非常实用的生产控制能力。它允许在服务端迅速关闭某一条异常观测链，而不需要发版。

---

## 8. GrowthBook：实验系统与动态配置中心的统一实现

`src/services/analytics/growthbook.ts` 是整个控制面的核心。

## 8.1 它解决的不是单一问题，而是四类问题

1. feature gate 判定
2. dynamic config 下发
3. experiment exposure logging
4. long-running session freshness 与 auth-change correctness

这就是为什么这个文件体量和复杂度都显著高于普通 flag SDK 封装。

## 8.2 核心接口族

### 同步读，允许陈旧

- `getFeatureValue_CACHED_MAY_BE_STALE<T>()`
- `getDynamicConfig_CACHED_MAY_BE_STALE<T>()`
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`

### 异步读，必要时阻塞初始化

- `getFeatureValue_DEPRECATED<T>()`
- `getDynamicConfig_BLOCKS_ON_INIT<T>()`
- `checkGate_CACHED_OR_BLOCKING()`
- `checkSecurityRestrictionGate()`

### 生命周期控制

- `initializeGrowthBook()`
- `refreshGrowthBookAfterAuthChange()`
- `refreshGrowthBookFeatures()`
- `setupPeriodicGrowthBookRefresh()`
- `resetGrowthBook()`
- `onGrowthBookRefresh()`

### override 控制

- `hasGrowthBookEnvOverride()`
- `setGrowthBookConfigOverride()`
- `clearGrowthBookConfigOverrides()`

这组接口已经体现了它不是简单 SDK 包装，而是 runtime control subsystem。

## 8.3 状态机视角

若从状态机角度看，GrowthBook client 至少经历以下状态：

```text
未初始化
  -> 已创建但无 auth
  -> 已创建且正在 init
  -> 已初始化且内存 payload 生效
  -> 周期性 refresh 中
  -> auth 变化后重建中
  -> reset / destroy
```

关键状态变量：

- `client`
- `clientCreatedWithAuth`
- `reinitializingPromise`
- `refreshInterval`
- `remoteEvalFeatureValues`
- `experimentDataByFeature`
- `pendingExposures`
- `loggedExposures`

这实际上构成了一个简化的 client-side control plane state machine。

## 8.4 为什么需要 `remoteEvalFeatureValues`

这可能是该文件最重要的工程性创新点之一。

问题背景：

- 服务端 remote eval 已经给出最终 feature 值。
- 但 SDK 本地仍可能再次评估规则，造成语义偏移。

解决方式：

- `processRemoteEvalPayload()` 解析 payload
- 把 authoritative feature values 提取到 `remoteEvalFeatureValues: Map<string, unknown>`
- 所有关键读取路径优先读这个 Map

这相当于主动接管第三方 SDK 的“最终真值层”。

从架构角度看，这是一个非常成熟的做法：

- 不试图修复所有 SDK 内部行为
- 只在 correctness-sensitive 边界建立自己的 authoritative cache

## 8.5 为什么 `pendingExposures` 与 `loggedExposures` 要同时存在

这是第二个关键工程点。

### `pendingExposures`

解决的问题：

- 某 feature 在 init 之前就被访问
- 当时实验元数据尚未加载

策略：

- 先记 feature 名
- 等 init 成功后统一补曝光

### `loggedExposures`

解决的问题：

- 热路径反复调用 `getFeatureValue_CACHED_MAY_BE_STALE()`

策略：

- session 内 dedup，每个 feature 最多记一次曝光

组合效果：

- 不丢初始化前曝光
- 不刷爆运行中曝光

这是一种对 agent long-lived hot path 非常友好的设计。

## 8.6 `onGrowthBookRefresh()` 的设计语义

这是 GrowthBook 与其他模块交互的主事件接口。

```ts
export function onGrowthBookRefresh(
  listener: GrowthBookRefreshListener,
): () => void
```

语义不是：

- “只有值变化才通知”

而是：

- “每次 refresh 都通知，调用方自己做 change detection”

理由：

- 避免 GrowthBook 层替各订阅者维护 last-seen snapshot
- 让订阅者决定什么变化值得响应

这是典型 observer 模式里的“event-driven, subscriber-owned diffing”。

## 8.7 三类 gate 读取语义的业务意义

### `CACHED_MAY_BE_STALE`

业务含义：

- 启动速度和同步性优先于实时性

适用于：

- sampling config
- kill switch
- 非安全关键、非 entitlement-critical 配置

### `checkSecurityRestrictionGate()`

业务含义：

- auth change 后不能返回过时的宽松结果

策略：

- reinit 时等待
- 优先看 Statsig cache，再看 GrowthBook cache

这体现的是保守安全语义。

### `checkGate_CACHED_OR_BLOCKING()`

业务含义：

- stale true 可接受
- stale false 不可接受

典型场景：

- 用户主动触发功能时，不能因为旧缓存误判成无权限

这非常贴近实际产品决策逻辑。

## 8.8 auth / trust 约束是如何嵌入 GrowthBook 的

初始化 client 时：

- 若 trust 未建立，则不取 auth headers
- 若无 auth，则直接跳过 HTTP init，依赖磁盘缓存

这背后的硬约束是：

- 不能为了观测/实验去触发 `apiKeyHelper` 或其他 trust-sensitive 行为

这说明在此项目里，实验系统必须服从 trust model，而不是反过来。

## 8.9 refresh 策略的工业级细节

### auth change refresh

- destroy + recreate client
- 因为 headers 无法安全热更新

### periodic refresh

- external：6h
- ant：20min

### stale-client guard

无论 init callback 还是 refresh callback，都显式检查：

- 当前回调绑定的 client 是否仍是全局最新 client

这可以防止 reinit race 造成旧 client 回写新状态。

这是非常典型的并发安全写法。

---

## 9. 1P Event Logging：内部事件链的独立日志平面

## 9.1 为什么独立于客户 OTLP logger

`firstPartyEventLogger.ts` 明确指出：

- 内部事件 logging 独立于 customer OTLP telemetry

核心原因：

1. 内部事件不可泄漏到用户自定义后端
2. 客户 telemetry 配置不应影响内部产品分析
3. internal event schema、鉴权、容错策略与 OTLP 标准链路不同

这意味着 1P logging 不是 OTLP telemetry 的一个 exporter，而是一条平行链路。

## 9.2 核心接口

### `logEventTo1P(eventName, metadata)`

职责：

- 业务事件记录
- 内部富化
- 异步 fire-and-forget emit

### `logGrowthBookExperimentTo1P(data)`

职责：

- 记录实验曝光
- 使用专门的 GrowthbookExperimentEvent schema

### `initialize1PEventLogging()`

职责：

- 创建专用 `LoggerProvider`
- 创建 `BatchLogRecordProcessor`
- 装配 `FirstPartyEventLoggingExporter`

### `reinitialize1PEventLoggingIfConfigChanged()`

职责：

- 当 GrowthBook 动态配置变化时热重建 logging pipeline

## 9.3 关键数据结构

### `EventSamplingConfig`

```ts
type EventSamplingConfig = {
  [eventName: string]: {
    sample_rate: number
  }
}
```

它表明采样不是全局统一比例，而是 per-event configurable。

### `BatchConfig`

```ts
type BatchConfig = {
  scheduledDelayMillis?: number
  maxExportBatchSize?: number
  maxQueueSize?: number
  skipAuth?: boolean
  maxAttempts?: number
  path?: string
  baseUrl?: string
}
```

这个结构很重要，因为它说明 1P logging pipeline 几乎整个传输面都可被动态控制。

### `GrowthBookExperimentData`

```ts
type GrowthBookExperimentData = {
  experimentId: string
  variationId: number
  userAttributes?: GrowthBookUserAttributes
  experimentMetadata?: Record<string, unknown>
}
```

它是实验系统与事件日志系统的桥接结构。

## 9.4 采样算法的语义

`shouldSampleEvent(eventName)` 的返回值不是简单 boolean，而是：

- `null`: 不采样控制，默认记录
- `0`: 明确丢弃
- `(0,1)`: 被采中，且 sample_rate 作为元数据写回事件

这比布尔返回值更强，因为它保留了下游估计总量所需的采样率信息。

---

## 10. `FirstPartyEventLoggingExporter`：本地 agent 的可靠投递引擎

这是整个观测架构中工程密度最高的模块之一。

## 10.1 构造参数就是一份传输层 contract

构造器支持：

```ts
constructor(options?: {
  timeout?: number
  maxBatchSize?: number
  skipAuth?: boolean
  batchDelayMs?: number
  baseBackoffDelayMs?: number
  maxBackoffDelayMs?: number
  maxAttempts?: number
  path?: string
  baseUrl?: string
  isKilled?: () => boolean
  schedule?: (fn, delayMs) => () => void
})
```

这意味着：

- exporter 本身是可测试、可注入、可替换调度器的
- 其可靠性策略不是写死在模块内部的黑箱

这在工业代码里很重要，因为可测试性决定了复杂重试逻辑是否可维护。

## 10.2 payload contract

核心 payload：

```ts
type FirstPartyEventLoggingEvent = {
  event_type: 'ClaudeCodeInternalEvent' | 'GrowthbookExperimentEvent'
  event_data: unknown
}

type FirstPartyEventLoggingPayload = {
  events: FirstPartyEventLoggingEvent[]
}
```

这说明 1P 后端不是接收原始 OTel log record，而是接收项目自定义事件信封。

OTel 在这里只是本地 batching + processing 机制，不是最终协议。

## 10.3 失败持久化协议

失败事件写入：

- `1p_failed_events.<sessionId>.<batchUuid>.json`

设计含义：

- session 级隔离
- process-run 级隔离
- append-only JSONL

为什么不是一个全局大文件？

- 避免并发覆盖
- 便于部分恢复
- 便于按 session 定位问题

## 10.4 关键算法 1：append-only 失败缓冲

`appendEventsToFile()` 采取 append-only 策略。

优点：

- 多次失败情况下文件写入更接近原子追加
- 不需要复杂锁
- 对 CLI 退出中断更友好

这是本地 agent 场景下非常合适的 WAL-like 思路。

## 10.5 关键算法 2：首次失败短路剩余 batch

`sendEventsInBatches()` 的逻辑不是“best effort 发完所有 batch”，而是：

- 第一个失败的 batch 之后，剩余 batch 全部直接入失败集合

工业意义：

- endpoint 故障时快速止损
- 避免重复放大错误流量
- 减少 shutdown 前尾部阻塞

## 10.6 关键算法 3：quadratic backoff

```text
delay = base * attempts^2
```

这比指数退避更温和，比线性退避更快拉开重试间隔。

对于本地观测流量，它比标准网络重试策略更适合，因为：

- 流量体量通常较小
- 程序生命周期不一定足够长
- 需要兼顾较快恢复与避免疯狂重试

## 10.7 关键算法 4：成功后立即回捞积压

一旦某次 export 成功：

- `resetBackoff()`
- 若磁盘队列非空，立刻 `retryFailedEvents()`

这说明该系统并不满足于“未来某次慢慢再试”，而是把“endpoint 恢复”当作 drain backlog 的触发器。

## 10.8 auth fallback 策略

发送时优先尝试 auth headers，但有多个降级路径：

- trust 未建立：直接无 auth
- OAuth token 过期：直接无 auth
- 缺少 profile scope：直接无 auth
- 401：重试无 auth

这是一种很成熟的“鉴权可选传输”策略。

核心思想不是“必须强鉴权”，而是：

- 在保证安全模型不被破坏的情况下，尽量让事件被送达

## 10.9 热重建时的 swap 语义

`reinitialize1PEventLoggingIfConfigChanged()` 的步骤是：

1. 暂时置空 logger，阻止并发 emit 到旧 provider
2. flush 旧 provider
3. 重建新 provider
4. 失败则恢复旧 provider
5. 成功后后台 shutdown 旧 provider

这体现出很强的工程成熟度：

- 它不是“关闭再重开”
- 而是带 rollback 的热替换

---

## 11. `metadata.ts`：数据治理与事件模式收敛层

这个模块是被低估的核心模块之一。

它不是简单拼 metadata，而是在做三件事：

1. 统一上下文富化
2. 统一隐私/脱敏策略
3. 统一 1P schema 映射

## 11.1 核心类型

### `EnvContext`

包含：

- 平台、架构、终端、包管理器、运行时
- remote/container/session 信息
- GitHub Actions 信息
- WSL / Linux distro / kernel
- 版本与部署环境

这是环境语义层。

### `ProcessMetrics`

包含：

- uptime
- rss / heap / external / arrayBuffers
- constrainedMemory
- cpuUsage / cpuPercent

这是进程执行层。

### `EventMetadata`

它融合了：

- 模型维度
- 会话维度
- 用户类型维度
- 环境维度
- 进程维度
- agent/team 维度
- 订阅维度
- repo hash 维度

这是全系统事件富化的 canonical internal shape。

### `FirstPartyEventLoggingMetadata`

它再进一步把 `EventMetadata` 变成：

- `env`
- `process`
- `auth`
- `core`
- `additional`

这是面向 1P proto schema 的 canonical transport shape。

## 11.2 类型系统承担的约束

这里有两个特别值得注意的 marker type。

### `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS`

语义：

- 开发者显式承诺这个字符串不含代码、文件路径等敏感内容

### `AnalyticsMetadata_I_VERIFIED_THIS_IS_PII_TAGGED`

语义：

- 该值会被路由到 PII 受控 proto 字段，不进入通用 sink

这是一种“类型注解式的数据治理”。

它不能彻底防止误用，但能显著提高误用门槛并增强代码审查信号。

## 11.3 工具名与工具参数的双层策略

### analytics 侧

- MCP tool 默认脱敏为 `mcp_tool`
- 仅在官方 registry / 特定场景下开放细节

### telemetry 侧

- `extractToolInputForTelemetry()` 只在 `OTEL_LOG_TOOL_DETAILS` 开启时记录
- 且对字符串长度、嵌套深度、collection 项数做截断

这反映出作者对“高价值调试信息”和“高风险高基数字段”之间权衡非常清楚。

## 11.4 proto 对齐约束

`to1PEventFormat()` 中的 `env` 使用 proto-generated `EnvironmentMetadata` 类型，而不是手写对象类型。

这带来的好处是：

- 新增字段若未先更新 proto，会在编译期暴露问题
- 避免 silently dropped fields

这在数据平台工程中是非常关键的设计。

---

## 12. Datadog：受限、高约束、低基数的公共分析 sink

`datadog.ts` 的设计非常保守。

## 12.1 只允许 allowlist 事件

`DATADOG_ALLOWED_EVENTS` 明确列出可发送事件名。

这意味着：

- 并不是所有 analytics event 都进 Datadog
- Datadog 在此更像公共概览/告警 sink，而不是全量内部事件仓库

## 12.2 强调 cardinality reduction

它对以下字段做了显式约束：

- MCP tool 名归一为 `mcp`
- model 名归一为 canonical / `other`
- dev version 截断到 base + date
- status 拆成 `http_status` 和 `http_status_range`

这是非常典型的 Datadog-friendly 设计，因为 Datadog 对高基数字段和 tag 规模很敏感。

## 12.3 tag 与 attribute 分层

它将部分字段放到 `ddtags`：

- 方便检索与 dashboard 查询

其他字段作为 searchable attributes：

- 便于保留信息但不无限扩大 tag cardinality

这是对 Datadog 数据模型非常熟悉的表现。

---

## 13. 客户侧 Telemetry：OTLP 标准遥测管道

`src/utils/telemetry/instrumentation.ts` 是客户遥测的主入口。

## 13.1 它管理三类 provider

- `MeterProvider`
- `LoggerProvider`
- `BasicTracerProvider`

并按 signal 拆分配置：

- metrics exporter
- logs exporter
- traces exporter

这是标准 OTel 结构，但该实现还叠加了几层项目化逻辑。

## 13.2 资源属性构造

资源属性来自：

- base resource：service name/version
- `osDetector`
- `hostDetector`
- `envDetector`
- WSL 特殊字段

随后 merge 成统一 resource。

这保证了：

- internal metrics
- logs
- traces

共享一致的资源语义。

## 13.3 `getOTLPExporterConfig()` 的工程点

这个函数统一处理：

- static headers
- `otelHeadersHelper` 动态 headers
- proxy
- mTLS
- CA cert
- proxy bypass

说明作者把“遥测链路也是生产网络请求”认真对待了，而不是默认裸连。

## 13.4 telemetry 与 BigQuery metrics 是双轨

即使 `CLAUDE_CODE_ENABLE_TELEMETRY` 没开，也可能初始化：

- `BigQueryMetricsExporter`

因为它服务的不是用户自己的可观测平台，而是内部产品/运营指标。

这再次印证该架构是多目标多管道系统。

## 13.5 `setMeter()` 暴露的是“带属性工厂”的 counter

`bootstrap/state.ts` 中：

```ts
type AttributedCounter = {
  add(value: number, additionalAttributes?: Attributes): void
}
```

`setMeter()` 并不直接暴露原始 OTel counter，而是包了一层会自动合并：

- `getTelemetryAttributes()`
- call-site additional attributes

这是一种很实用的封装，保证所有计数器天然带基础归因维度。

---

## 14. Session Tracing：agent execution graph 的观测镜像

`sessionTracing.ts` 应该被理解为：

- 一套面向 agent workflow 的 domain-specific tracing API

而不仅仅是 OTel span helper。

## 14.1 领域建模

span type：

- `interaction`
- `llm_request`
- `tool`
- `tool.blocked_on_user`
- `tool.execution`
- `hook`

这些并不是技术分层，而是 agent runtime 领域事件。

## 14.2 上下文传播策略

它采用：

- `AsyncLocalStorage<SpanContext | undefined>` 管 interaction / tool 上下文
- `activeSpans: Map<string, WeakRef<SpanContext>>`
- `strongSpans: Map<string, SpanContext>`

这说明作者在解决一个非常具体的问题：

- agent 运行中存在大量异步、并行、嵌套子任务
- 不能只依赖 OTel 默认 active span

因此它自己维护了更强的 span registry。

## 14.3 GC 与 orphaned span 治理

`ensureCleanupInterval()` 以 30 分钟 TTL 清理 orphaned span。

这类逻辑通常只会出现在：

- 长生命周期
- 异步复杂
- 有真实内存泄漏经验

的系统中。

换句话说，这不是“预防性优化”，更像是踩坑后的制度化防线。

## 14.4 并发安全：显式 span 句柄优先于“最近一个”

`endLLMRequestSpan(span?, metadata?)` 的注释已经说明：

- 并发请求时若不用显式传 span，会挂错结果

这背后反映的是：

- 该 runtime 真实存在并发 LLM request 场景
- tracing API 必须能够表达“哪个 end 对应哪个 start”

这对多 agent、多 classifier、多 warmup 的 AI runtime 非常重要。

---

## 15. Profiling 子系统：观测架构中的“离线剖析面”

## 15.1 `startupProfiler.ts`

它本质上是：

- sampled production startup telemetry
- detailed local startup forensic report

的双模系统。

关键设计：

- ant 100% sampling，external 低比例 sampling
- 只有 sampled / detailed 模式才记录 checkpoint
- detailed 模式记录 memory snapshot

这使得它既能支撑线上群体趋势分析，又能支撑本地单实例精查。

## 15.2 `queryProfiler.ts`

它的价值在于把 TTFT 分解为：

- pre-request overhead
- network latency

这在 AI 应用里非常关键，因为：

- 用户只感知“首 token 慢”
- 但优化动作完全取决于慢在客户端准备还是后端响应

它实际上是一种“query path phase decomposition profiler”。

---

## 16. 关键调用链：把模块交互放回真实业务里看

## 16.1 权限请求出现

调用点：

- `src/components/permissions/hooks.ts`

交互：

```text
Permission UI
  -> logEvent('tengu_tool_use_show_permission_request', ...)
  -> analytics/index.ts
  -> analytics/sink.ts
  -> Datadog / 1P
```

这里观测服务的是：

- friction analysis
- permission policy tuning
- sandbox 策略评估

## 16.2 工具执行完成

调用点：

- `src/services/tools/toolExecution.ts`

交互：

```text
Tool execution done
  -> logOTelEvent('tool_result', ...)
  -> utils/telemetry/events.ts
  -> global eventLogger
  -> logger provider / OTLP exporter
```

这里观测服务的是：

- 客户或平台级 tool performance tracing
- tool parameter/result envelope analysis

注意这条链不走 `analytics/index.ts`，说明内部分析与客户遥测接口面是明确分叉的。

## 16.3 API 请求完成

调用点：

- `src/services/api/logging.ts`

交互：

```text
API response done
  -> logOTelEvent('api_request', ...)
  -> endLLMRequestSpan(llmSpan, metadata)
```

这里 event 与 trace 是双写关系：

- event 负责结构化 summary
- trace 负责链路关系与时间分布

## 16.4 GrowthBook refresh 触发 1P logger 重建

调用点：

- `init.ts`
- `firstPartyEventLogger.ts`
- `growthbook.ts`

交互：

```text
GrowthBook refresh
  -> refreshed.emit()
  -> onGrowthBookRefresh subscriber
  -> reinitialize1PEventLoggingIfConfigChanged()
  -> hot swap provider
```

这是一个非常典型的控制面驱动数据面热更新的案例。

---

## 17. 设计约束与不变量

如果未来要演进这套系统，下面这些约束最好视为不变量。

## 17.1 不变量一：内部分析链与客户遥测链不可合并

否则会带来：

- 数据泄漏风险
- schema 混淆
- 用户配置影响内部分析 correctness

## 17.2 不变量二：GrowthBook 的同步读接口必须继续存在

原因：

- 启动路径不能为了 flag/config 阻塞
- 多处调用点要求同步语义

若强行统一成 async-only，会大幅破坏 cold start 和调用复杂度。

## 17.3 不变量三：trust gate 必须先于 auth-sensitive telemetry

否则观测系统会绕过产品安全模型。

## 17.4 不变量四：`_PROTO_*` 剥离必须集中处理

若将过滤逻辑分散到各 sink，将极易出现新增 sink 漏过滤的问题。

## 17.5 不变量五：trace API 必须支持并发请求的显式句柄

否则多 agent / 多 request 场景下的 trace correctness 无法保证。

## 17.6 不变量六：exporter 必须支持退出前 flush 与失败补偿

CLI/agent 不是常驻服务，不做这一层会系统性丢尾部数据。

## 17.7 不变量七：metadata 富化与 schema 映射必须集中

若分散到各调用点，将导致：

- 字段不一致
- 隐私策略分叉
- 后端 schema drift

---

## 18. 关键创新性与工业级工程亮点

从架构创新和工业工程经验来看，这套系统至少有 10 个亮点。

## 18.1 用 GrowthBook 同时承担 feature flag、dynamic config、sink control、sampling control

这让控制面高度统一。

## 18.2 针对 remote eval SDK 语义问题，自建 authoritative in-memory value layer

`remoteEvalFeatureValues` 是典型的 correctness patch layer。

## 18.3 用 `pendingExposures + loggedExposures` 同时解决“初始化前不丢曝光”和“热路径不重复曝光”

这是很实用的实验系统设计。

## 18.4 内部 1P logging 采用独立 OTel provider，而不是复用全局 provider

这是安全边界与架构边界都很清晰的实现。

## 18.5 exporter 具备 append-only 本地持久化 + next-run retry 能力

这是一种为 CLI 生命周期量身定制的轻量可靠消息系统。

## 18.6 通过 proto-generated 类型约束 1P schema 对齐

这能显著减少静默字段丢失。

## 18.7 metrics/logs/traces provider state 统一进入 bootstrap 全局状态

这让 agent runtime 任意层都能接入观测，而无需复杂 DI。

## 18.8 `AsyncLocalStorage + WeakRef + TTL cleanup` 管理长会话 trace context

这是非常成熟的异步 tracing 内存治理方式。

## 18.9 profiler 采用 sampled production mode + detailed forensic mode 双模策略

既能看群体趋势，也能做单实例根因定位。

## 18.10 对 Datadog 做了非常严格的 cardinality-aware engineering

这不是“会用 Datadog SDK”级别，而是“理解 Datadog 数据模型成本”的实现。

---

## 19. 对 AI 应用和算法专家的启示

如果从 AI runtime / agent platform 角度看，这套设计有三点非常值得借鉴。

## 19.1 实验系统不能只是“模型 prompt AB test”

在 agent 场景里，实验系统应当能控制：

- prompt 逻辑
- tool policy
- sampling
- transport 参数
- performance 开关
- cache 策略

Claude-code-open 的实现就是这样。

## 19.2 观测必须表达 agent 的结构，而不只是 API 请求

传统应用常按：

- request
- response
- error

建模。

但 agent runtime 更合适按：

- interaction
- llm_request
- tool
- approval wait
- hook

建模。

这一点直接决定 trace 是否有解释力。

## 19.3 对本地 agent 来说，可靠上报比标准协议更重要

OTLP 很重要，但在 CLI 进程中：

- 本地缓冲
- 热重建
- 退出 flush
- trust/auth 约束

往往比协议标准化本身更决定系统是否真正可用。

---

## 20. 最终总结

Claude-code-open 的“观测与实验架构”可以被视为一个完整的 agent runtime control-and-observability substrate。

它不是在业务系统外面挂了几个埋点模块，而是把：

- 控制面：GrowthBook、privacy、trust、auth、dynamic config
- 数据面：analytics、1P logging、Datadog、OTLP telemetry
- 剖析面：startup/query profiler、session tracing、Perfetto
- 可靠性面：队列、缓存、重试、flush、回滚、热重建

统一编织进了运行时。

如果只用一句更专家化的话概括：

```text
这套架构把“实验、观测、性能剖析、隐私治理、传输可靠性”合并成了 agent runtime 的控制基础设施层，
它既产生数据，也决定系统何时、如何、以什么约束条件运行。
```

这正是它区别于普通埋点系统、也区别于普通 feature flag 封装的地方。
