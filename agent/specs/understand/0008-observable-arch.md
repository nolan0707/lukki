# Claude-code-open 观测与实验架构深度解读

## 1. 文档目标

本文聚焦 `vendor/Claude-code-open` 中“观测与实验架构”的实现。

这里的“观测”不是狭义的埋点系统，而是一整套会直接参与运行时决策的基础设施，包括：

- analytics 事件上报
- 1P event logging
- Datadog 日志
- OpenTelemetry metrics / logs / traces
- startup profiler / query profiler / session tracing
- GrowthBook feature gate / dynamic config

本文重点回答下面几个问题：

1. 为什么这个项目把“观测”和“实验”做成核心架构，而不是外围插件？
2. 事件、指标、日志、trace、feature gate 在代码里分别由谁负责？
3. 一次启动、一次 API 请求、一次工具执行，观测链路是怎么串起来的？
4. GrowthBook 为什么既像 feature flag 系统，又像动态配置中心？
5. 这套设计里有哪些关键数据结构和关键算法，值得初学者重点理解？

本文基于以下主干代码阅读整理：

- `src/services/analytics/*`
- `src/utils/telemetry/*`
- `src/utils/startupProfiler.ts`
- `src/utils/queryProfiler.ts`
- `src/entrypoints/init.ts`
- `src/main.tsx`
- `src/services/api/logging.ts`
- `src/services/tools/toolExecution.ts`
- `src/components/permissions/hooks.ts`

---

## 2. 先给结论

这套架构本质上是一个“运行时决策 + 多通道观测”的统一系统。

可以先把它理解成 4 层：

```text
实验与开关层
  GrowthBook
  feature gate / dynamic config / kill switch

观测接入层
  services/analytics/index.ts
  utils/telemetry/events.ts
  startupProfiler / queryProfiler / sessionTracing

观测路由层
  analytics/sink.ts
  firstPartyEventLogger.ts
  utils/telemetry/instrumentation.ts

观测后端层
  Datadog
  1P event logging endpoint
  OTLP exporters / BigQuery / Perfetto
```

其中最关键的设计思想有 3 个：

### 2.1 观测不是“事后记录”，而是“运行时能力”

GrowthBook 不只是决定“是否做实验”，还决定：

- 是否启用 Datadog sink
- 事件采样率是多少
- 1P 批量上报参数是多少
- 是否启用 enhanced telemetry
- 某些业务功能 gate 是否允许进入

也就是说，配置系统本身直接影响程序行为。

### 2.2 内部分析与客户遥测是两条不同管道

项目明确把两类数据分开：

- 内部产品分析：`src/services/analytics/*`
- 客户 OTLP 遥测：`src/utils/telemetry/*`

这不是重复建设，而是安全边界和产品目标不同：

- 内部分析关注产品使用、实验曝光、权限交互、启动性能等。
- 客户遥测关注用户自己接入的 OpenTelemetry 后端。

### 2.3 这是一套“默认不阻塞主流程，但尽量不丢数据”的系统

它大量使用以下策略平衡性能和可靠性：

- sink 未挂载前先入队
- 动态 import 延迟加载重型 OTel 依赖
- 优先读磁盘缓存，后台刷新远端配置
- 批量发送、失败落盘、退避重试
- `beforeExit` / `exit` 强制 flush
- 对高成本观测做采样

所以它不是“简单发 HTTP 埋点”，而是一套比较成熟的 agent runtime telemetry architecture。

---

## 3. 总体模块分工

## 3.1 `src/services/analytics/*`：内部分析与实验控制中心

这部分负责：

- 通用 analytics API：`logEvent()`
- sink 挂载与事件路由
- Datadog 事件日志
- 1P event logging
- GrowthBook feature gate / dynamic config
- 元数据富化与格式转换

可以把它理解成：

```text
给产品自己看的观测系统
```

核心文件：

- `src/services/analytics/index.ts`
- `src/services/analytics/sink.ts`
- `src/services/analytics/datadog.ts`
- `src/services/analytics/firstPartyEventLogger.ts`
- `src/services/analytics/firstPartyEventLoggingExporter.ts`
- `src/services/analytics/growthbook.ts`
- `src/services/analytics/metadata.ts`

## 3.2 `src/utils/telemetry/*`：客户侧 OpenTelemetry 管道

这部分负责：

- meter / logger / tracer provider 初始化
- OTLP exporter 配置
- BigQuery metrics exporter
- session tracing
- OTel event logging
- Perfetto tracing

可以把它理解成：

```text
给用户或平台接入标准遥测后端的系统
```

核心文件：

- `src/utils/telemetry/instrumentation.ts`
- `src/utils/telemetry/events.ts`
- `src/utils/telemetry/sessionTracing.ts`
- `src/utils/telemetry/perfettoTracing.ts`

## 3.3 profiler 模块：性能剖析的独立旁路

这部分不直接属于 analytics 或 OTel，但和观测体系强相关：

- `src/utils/startupProfiler.ts`
- `src/utils/queryProfiler.ts`

它们的目标不是持续生产指标，而是帮助开发者定位慢在哪里。

## 3.4 初始化入口：把整套系统接进 runtime

初始化主要分两段：

- `src/main.tsx`
  负责早期 analytics gate 初始化
- `src/entrypoints/init.ts`
  负责异步初始化 1P event logging 和客户侧 telemetry

这说明作者很在意启动性能，故意把重型模块拆成多阶段加载。

---

## 4. 为什么“观测与实验”是第一等架构能力

`0001-project-arch.md` 里已经点出关键事实：这些系统“不是外围工具，而是 runtime decision 的一部分”。

这句话非常重要。

在很多项目里：

- 埋点系统只负责记日志
- feature flag 只负责开关页面按钮

但在这里：

- `GrowthBook` 直接影响 sink 是否启用、采样率、批大小、功能权限。
- `sessionTracing` 直接包裹用户交互、LLM 请求、工具调用。
- `startupProfiler` 直接嵌在 `main.tsx` 和 `init.ts` 的关键路径上。
- `analytics metadata` 会携带会话、agent、环境、订阅、repo hash 等上下文，影响后续分析与归因。

也就是说，这套架构不是“业务完成后顺手打个点”，而是：

```text
业务执行时，系统一边决策，一边观测，一边把结果反馈回下一轮决策
```

这就是 agent 系统和普通 CLI 的一个重要区别。

---

## 5. 启动时序：观测与实验系统是怎么启动的

先看主流程：

```text
main.tsx
  -> initializeAnalyticsGates()
  -> init()
      -> 动态 import firstPartyEventLogger + growthbook
      -> initialize1PEventLogging()
      -> onGrowthBookRefresh(() => reinitialize1PEventLoggingIfConfigChanged())
      -> 之后按需 initializeTelemetry()
```

## 5.1 `main.tsx`：先把轻量 gate 读起来

`src/main.tsx` 在很早阶段调用：

- `initializeAnalyticsGates()`

它只做一件事：

- 用 `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()` 读取 Datadog gate 的缓存值。

这是一个典型的“启动快于准确”的设计：

- 早期事件可以先用上次缓存值决策，避免 cold start 时丢数据。
- 后续再由 GrowthBook 正式初始化和刷新。

## 5.2 `init.ts`：异步初始化 1P event logging

`src/entrypoints/init.ts` 里有非常关键的一段：

- 动态 import `firstPartyEventLogger.js`
- 动态 import `growthbook.js`
- 调 `initialize1PEventLogging()`
- 注册 `onGrowthBookRefresh()`，配置变更时重建 1P logger provider

这里体现了两个架构意图：

### 意图 A：避免启动时立即加载重型 OTel SDK

注释里明确写了：

- 延迟加载 OpenTelemetry sdk-logs / resources

原因很现实：

- 这些包很重
- CLI 首屏性能敏感

### 意图 B：动态配置不是“只读一次”

1P logger 的 batch 配置来自 GrowthBook 动态配置：

- `tengu_1p_event_batch_config`

所以项目不是“启动时读一次配置就算了”，而是支持：

- 运行中配置刷新
- logger provider 热重建

这对长生命周期 session 很重要。

## 5.3 `initializeTelemetry()`：客户 OTLP 遥测按需初始化

`init.ts` 的 `setMeterState()` 会延迟 import：

- `src/utils/telemetry/instrumentation.ts`

然后调用：

- `initializeTelemetry()`

它负责：

- 初始化 meter provider
- 按配置初始化 logger provider
- 按配置初始化 tracer provider
- 注册 cleanup / shutdown / flush 逻辑

所以内部分析和客户遥测虽然都用到 OTel，但它们是两条独立初始化链路。

---

## 6. 核心业务流程一：analytics 事件是如何被记录和路由的

## 6.1 统一入口：`logEvent()`

文件：`src/services/analytics/index.ts`

对业务代码来说，最常见的调用方式就是：

```ts
logEvent('tengu_xxx', { ...metadata })
```

这个模块刻意保持“无依赖”，原因是：

- 避免 import cycle
- 保证所有模块都可以安全调用

它的关键机制有两个。

### 机制 A：sink 未挂载时先排队

内部维护：

- `eventQueue: QueuedEvent[]`
- `sink: AnalyticsSink | null`

如果调用 `logEvent()` 时 sink 还没 attach：

- 事件先进入 `eventQueue`

等 `attachAnalyticsSink()` 执行后：

- 异步 drain 队列

这解决了一个非常实际的问题：

- 启动早期很多模块已经想记事件
- 但真正后端 sink 可能还没初始化好

### 机制 B：异步 drain，避免启动路径被阻塞

`attachAnalyticsSink()` 用 `queueMicrotask()` 排空队列，而不是同步 drain。

好处是：

- sink 挂上后马上可用
- 不把历史事件回放塞进当前启动关键路径

这是典型的 CLI 启动优化手法。

## 6.2 sink 路由：`analytics/sink.ts`

这个文件把统一事件分发到两个后端：

- Datadog
- 1P event logging

关键逻辑：

1. 根据 `shouldSampleEvent()` 决定是否采样
2. 如果启用了 Datadog，则发送到 Datadog
3. 总是尝试发到 1P event logging
4. 对 Datadog 先做 `_PROTO_*` 字段剥离

这里有两个关键点。

### 点 A：采样发生在 sink 层，而不是调用点

这意味着业务代码不需要关心采样策略。

调用方只负责：

- 记录事件名
- 带上业务元数据

采样策略由运行时统一控制，且可通过 GrowthBook 动态下发。

### 点 B：不同 sink 的隐私边界不同

`_PROTO_*` 字段表示：

- 这些值允许进入带 PII 访问控制的 proto 字段
- 但不能进入通用可访问后端

因此：

- Datadog 前必须 `stripProtoFields()`
- 1P exporter 可识别并提取这些字段

这是一种“数据分级后再路由”的设计。

---

## 7. 核心业务流程二：1P event logging 为什么要单独做一条 OTel 日志链

## 7.1 `firstPartyEventLogger.ts`：内部事件发射器

这个模块做的事可以概括为：

```text
业务事件
  -> 富化元数据
  -> 组装 OTel log record
  -> 交给内部专用 LoggerProvider
```

它没有复用全局 logger provider，而是自己创建了一套：

- `LoggerProvider`
- `BatchLogRecordProcessor`
- `FirstPartyEventLoggingExporter`

原因非常重要：

- 客户 telemetry 不能混进内部事件
- 内部事件也不能跑去用户配置的 OTLP endpoint

所以这里是“同样使用 OTel SDK，但 provider 独立”。

## 7.2 事件数据是怎么富化的

调用 `logEventTo1P()` 时，系统会进一步调用：

- `getEventMetadata()`
- `getCoreUserData(true)`
- `getOrCreateUserID()`

最终产出的 attributes 大致包括：

- `event_name`
- `event_id`
- `core_metadata`
- `user_metadata`
- `event_metadata`
- `user_id`

这一步相当于把一条简单埋点变成“可分析事件对象”。

## 7.3 GrowthBook 实验曝光也走 1P 管道

`logGrowthBookExperimentTo1P()` 会单独发 `growthbook_experiment` 事件。

也就是说，实验曝光没有只停留在内存或 console，而是被正式纳入内部事件日志体系。

这使得后续可以分析：

- 某个实验分流到了哪些用户
- 该实验分流与后续行为、错误、转化之间的关系

## 7.4 `FirstPartyEventLoggingExporter`：可靠性是重点

这个 exporter 是整个观测架构里最值得初学者精读的模块之一。

它实现的不是“发个 POST”这么简单，而是一整套可靠投递策略：

- 批量发送
- 大批拆小批
- 失败落盘
- 追加写 JSONL
- 退避重试
- 下次运行继续补发
- 401 自动降级为无鉴权重试
- 成功后立即回捞磁盘积压事件

### 关键算法 1：分批发送 + 首次失败短路

`sendEventsInBatches()` 会把事件按 `maxBatchSize` 切成多个 batch。

发送策略不是“全部都试一遍”，而是：

- 前面的 batch 成功则继续
- 一旦某个 batch 失败，后续 batch 直接视为失败，不再继续打接口

这样做的假设是：

- 如果当前 endpoint 已经坏了，继续 POST 后续 batch 没有意义

这是一种很实用的“快速止损”算法。

### 关键算法 2：二次退避

`scheduleBackoffRetry()` 用的是：

```text
delay = baseBackoffDelayMs * attempts^2
```

也就是 quadratic backoff，而不是简单线性增长。

优点：

- 前几次重试比较快
- 多次失败后增长速度更明显
- 比指数退避更温和，也更符合这种本地缓冲重试场景

### 关键算法 3：落盘重试与跨进程恢复

失败事件会写入：

- `~/.claude/.../telemetry/1p_failed_events.<session>.<uuid>.json`

下次 exporter 初始化时会扫描旧文件并后台重试。

这意味着：

- 即使进程退出，事件也不一定丢
- 即使当前 batch rebuild / provider rebuild，也能继续恢复

对于 CLI/agent 这种“进程可能随时结束”的程序，这是很重要的设计。

### 关键算法 4：热重建时先 flush 再 swap

当 GrowthBook 刷新导致 batch config 变化时，
`reinitialize1PEventLoggingIfConfigChanged()` 会：

1. 先把 logger 置空，阻止并发新写入
2. `forceFlush()` 老 provider
3. 重建 provider / logger
4. 后台关闭老 provider

这是一种典型的“短暂丢少量事件，换取管道一致性”的工程折中。

---

## 8. 核心业务流程三：GrowthBook 如何同时承担 feature flag 与动态配置中心

## 8.1 先理解它的定位

在这个项目里，GrowthBook 不只是返回 `true/false`。

它支持三类读取模式：

1. 布尔 gate
2. 任意 feature value
3. JSON dynamic config

因此它既是：

- feature flag 系统

也是：

- 远程动态配置中心

## 8.2 关键状态结构

文件：`src/services/analytics/growthbook.ts`

这里维护了几个非常关键的运行时状态：

- `experimentDataByFeature: Map<string, StoredExperimentData>`
- `remoteEvalFeatureValues: Map<string, unknown>`
- `pendingExposures: Set<string>`
- `loggedExposures: Set<string>`
- `reinitializingPromise: Promise<unknown> | null`

这几个结构几乎就是整个实验系统的核心。

### `remoteEvalFeatureValues`

含义：

- 保存服务端 remote eval 后的最终 feature value

用途：

- 避开 SDK 本地再求值问题
- 给同步读取路径提供当前进程内最准确的值

### `experimentDataByFeature`

含义：

- 保存 feature 对应的实验信息，如 experimentId / variationId

用途：

- 在 feature 被真正访问时，再补记 exposure event

### `pendingExposures`

含义：

- 某个 feature 在初始化前就被读取了，但当时还没有实验信息

用途：

- 等 init 完成后再补曝光

### `loggedExposures`

含义：

- 本 session 已经记过曝光的 feature

用途：

- 避免高频热路径反复上报同一个曝光

这是一种“延迟曝光 + 会话去重”的组合策略。

## 8.3 为什么要自己维护 `remoteEvalFeatureValues`

注释里明确提到一个背景：

- 远端返回的是预计算好的 remote eval 结果
- SDK 某些逻辑仍会尝试本地重新求值
- `setForcedFeatures()` 也不可靠

于是代码采取了一个很务实的 workaround：

1. 读远端 payload
2. 把每个 feature 的 authoritative value 缓存到 `remoteEvalFeatureValues`
3. 后续读值优先读这个 Map

这说明作者的设计哲学非常务实：

- 不迷信第三方 SDK 的抽象
- 真正影响 correctness 的地方自己接管

## 8.4 为什么有这么多“CACHED_MAY_BE_STALE”接口

这类命名是理解整个架构的关键。

代表函数：

- `getFeatureValue_CACHED_MAY_BE_STALE()`
- `getDynamicConfig_CACHED_MAY_BE_STALE()`
- `checkStatsigFeatureGate_CACHED_MAY_BE_STALE()`

这些接口的设计目标是：

- 启动快
- 同步可读
- 允许短暂陈旧

它们会按优先级读取：

1. env override
2. 本地 config override
3. 进程内 `remoteEvalFeatureValues`
4. 磁盘缓存 `cachedGrowthBookFeatures`
5. defaultValue

这个优先级非常合理：

- 测试覆盖最强的 env override 优先级最高
- 当前进程内 fresh 值优于磁盘缓存
- 最后才回落默认值

## 8.5 三种 gate 读取策略

GrowthBook 这里不是只有一个 API，而是按风险等级提供三种语义。

### 策略 A：`getFeatureValue_CACHED_MAY_BE_STALE`

适用于：

- 启动关键路径
- 可以容忍缓存陈旧
- 需要同步返回

例子：

- event sampling config
- sink kill switch
- 常规 UI / 体验类开关

### 策略 B：`checkSecurityRestrictionGate`

适用于：

- 安全相关 gate
- auth 切换后必须等待 reinit 完成

它会：

- 如果正在 reinitialize，就等待
- 先看 Statsig cache
- 再看 GrowthBook cache

这是一种“宁可保守也别放过安全限制”的策略。

### 策略 C：`checkGate_CACHED_OR_BLOCKING`

适用于：

- 用户主动触发的 entitlement gate
- 缓存里是 `false` 时不能轻易误伤用户

它的策略是：

- 缓存已是 `true`，直接放行
- 缓存是 `false` 或缺失，则阻塞等待 fresh 值

这是典型的“对 stale true 宽容，对 stale false 严格纠偏”的业务语义设计。

## 8.6 刷新机制

GrowthBook 有两类刷新。

### 刷新 A：鉴权状态变化后重建

函数：

- `refreshGrowthBookAfterAuthChange()`

原因：

- auth headers 变了，老 client 不能安全复用

动作：

- `resetGrowthBook()`
- 重新初始化 client
- 发出 refresh signal

### 刷新 B：周期性轻量刷新

函数：

- `refreshGrowthBookFeatures()`
- `setupPeriodicGrowthBookRefresh()`

特点：

- 不重建 client
- 只重新拉 features
- external 默认 6 小时一刷
- ant 默认 20 分钟一刷

这样既兼顾长会话 freshness，也不把刷新成本放大。

---

## 9. OpenTelemetry 侧的设计：为什么要把 meter / log / trace 分开

## 9.1 `initializeTelemetry()` 的基本职责

文件：`src/utils/telemetry/instrumentation.ts`

它会根据环境变量决定启用哪些 exporter，并分别初始化：

- `MeterProvider`
- `LoggerProvider`
- `TracerProvider`

这符合 OpenTelemetry 标准模型，但这里还有几层项目化设计。

## 9.2 exporter 是按 signal 分开配置的

代码分别处理：

- `getOtlpReaders()` for metrics
- `getOtlpLogExporters()` for logs
- `getOtlpTraceExporters()` for traces

这意味着：

- 指标、日志、链路可以分别启停
- 协议也可以不同

例如：

- metrics 用 OTLP protobuf
- logs 用 console
- traces 直接关闭

对复杂部署场景很友好。

## 9.3 动态 import exporter：控制包体与启动成本

这里大量使用：

- 按协议懒加载 exporter
- gRPC exporter 进一步延迟 import

原因很明确：

- 静态 import 会把所有协议实现都打进启动路径
- 很多用户只会用一种协议

这是一个非常实用的性能优化点。

## 9.4 内部指标和客户 telemetry 共存

即使用户没有开启 `CLAUDE_CODE_ENABLE_TELEMETRY`，系统仍可能初始化：

- 内部 BigQuery metrics exporter

因为项目把：

- 内部运营/产品指标
- 用户自定义 OTLP 遥测

分成了两类目标。

所以这里 again 体现的是“双通道观测”设计，而不是单一 telemetry pipeline。

## 9.5 shutdown / flush 被认真设计过

`initializeTelemetry()` 和 `flushTelemetry()` 都显式处理：

- `meterProvider.forceFlush()`
- `loggerProvider.forceFlush()`
- `tracerProvider.forceFlush()`
- timeout 保护

这说明作者非常清楚：

- CLI 程序不是常驻服务
- 不显式 flush，很多批量数据会在退出时丢失

因此 cleanup 是这个架构不可缺的组成部分。

---

## 10. Session Tracing：把一次交互拆成 interaction / llm_request / tool 这些 span

## 10.1 它想解决什么问题

文件：`src/utils/telemetry/sessionTracing.ts`

普通 metrics 只能回答：

- 平均多久
- 总共多少次

但 agent 真正复杂的是链路结构，例如：

- 一次用户交互里发了几次 LLM 请求？
- 哪个工具卡在用户审批？
- TTFT 慢是请求前准备慢，还是网络慢？

这类问题必须靠 trace / span 才能看清。

## 10.2 span 模型

这里定义了几个关键 span type：

- `interaction`
- `llm_request`
- `tool`
- `tool.blocked_on_user`
- `tool.execution`
- `hook`

可以把层级理解成：

```text
interaction
  -> llm_request
  -> tool
      -> tool.blocked_on_user
      -> tool.execution
```

这和 agent 的真实运行结构高度一致。

## 10.3 为什么要用 `AsyncLocalStorage`

它维护了：

- `interactionContext`
- `toolContext`

目的不是为了“存个变量方便”，而是为了：

- 在异步调用链中保留父 span 上下文
- 让后续 LLM / tool span 自动挂到正确父节点下

这是 Node.js / Bun 里实现 trace context 传播的常见做法。

## 10.4 为什么同时有 `activeSpans` 和 `strongSpans`

这里有一套很细的内存管理设计：

- `activeSpans: Map<string, WeakRef<SpanContext>>`
- `strongSpans: Map<string, SpanContext>`

原因是：

- 某些 span 依赖 ALS 保持强引用
- 某些 span 不在 ALS 里，必须手动强引用
- 又不能无上限泄漏内存

因此系统引入：

- 弱引用追踪
- 30 分钟 TTL 清理定时器

这是一个很典型的“长会话异步 trace 内存治理”设计。

## 10.5 为什么 `endLLMRequestSpan(span, metadata)` 强调显式传 span

注释说得很清楚：

- 并发 LLM 请求场景下，如果只按“最近一个 llm span”匹配，会把响应挂错 span

所以新接口要求：

- start 时拿到 span
- end 时尽量显式传回这个 span

这其实是在修复并发 trace 里最经典的一类 bug：

- request A 的结束信息记到了 request B 上

对 agent 而言，这类问题非常常见，因为 warmup / classifier / 主请求都可能并行。

---

## 11. profiler：为什么还需要 startupProfiler 和 queryProfiler

## 11.1 `startupProfiler.ts`

这个模块的目标是回答：

- 启动慢到底慢在哪个阶段？

它通过 `profileCheckpoint(name)` 在关键启动路径打时间点，然后：

- sampled 模式下记录 `tengu_startup_perf`
- detailed 模式下输出完整报告到文件

关键点：

- ant 100% 采样
- external 低比例采样
- 只在 sampled 或 detailed 模式下才启用，避免常态成本

这是一个典型的“低成本线上统计 + 开发者深度模式”双模设计。

## 11.2 `queryProfiler.ts`

这个模块更接近“单次 query 剖析器”。

它关注：

- 上下文加载
- microcompact
- autocompact
- tool schema 构建
- client 创建
- API request 发出
- first chunk 到达
- tool execution

最终能回答一个很实用的问题：

```text
TTFT 慢，到底是请求前准备慢，还是网络慢？
```

它甚至专门计算：

- pre-request overhead
- network latency

对性能分析非常有帮助。

---

## 12. 三条典型业务流：把观测与实验放回真实场景里看

## 12.1 场景一：权限弹窗出现时

在 `src/components/permissions/hooks.ts` 中，
`usePermissionRequestLogging()` 会在权限请求展示时调用：

- `logEvent('tengu_tool_use_show_permission_request', ...)`

它会记录：

- messageID
- toolName
- 是否 MCP
- decisionReasonType
- sandbox 是否开启

这说明权限系统并不是黑箱。

产品可以分析：

- 哪些工具最常触发审批
- 哪类原因最常导致用户被打断
- sandbox 策略是否导致过多 friction

## 12.2 场景二：工具执行成功后

在 `src/services/tools/toolExecution.ts` 中，
工具执行完成后会调用：

- `logOTelEvent('tool_result', ...)`

带上：

- tool_name
- success
- duration_ms
- tool_parameters
- tool_input
- tool_result_size_bytes
- decision_source / decision_type

这条链路属于客户 OTLP telemetry，而不是内部 analytics。

它适合回答：

- 某类工具是否经常超时
- 某些决策路径是否导致结果偏大
- MCP 工具范围是否异常

## 12.3 场景三：一次 API 请求结束后

在 `src/services/api/logging.ts` 中，
请求完成会：

1. `logOTelEvent('api_request', ...)`
2. `endLLMRequestSpan(llmSpan, metadata)`

metadata 里包括：

- token 使用
- cache token
- cost
- duration
- model output
- thinking output
- hasToolCall
- ttftMs

于是你能同时得到两类视角：

- event 视角：这次请求是什么
- trace 视角：它位于哪条交互链路里、耗时构成如何

这就是 event + trace 组合的价值。

---

## 13. 核心数据结构梳理

下面列出理解这套架构最重要的数据结构。

## 13.1 `QueuedEvent`

文件：`src/services/analytics/index.ts`

```ts
type QueuedEvent = {
  eventName: string
  metadata: LogEventMetadata
  async: boolean
}
```

作用：

- 承接 sink attach 之前的事件

它体现的是“先收下，再找后端”的设计。

## 13.2 `AnalyticsSink`

```ts
type AnalyticsSink = {
  logEvent: (eventName, metadata) => void
  logEventAsync: (eventName, metadata) => Promise<void>
}
```

作用：

- 把接入层和后端路由层解耦

这让 `index.ts` 可以完全不知道 Datadog / 1P 的实现细节。

## 13.3 `EventSamplingConfig`

文件：`src/services/analytics/firstPartyEventLogger.ts`

```ts
type EventSamplingConfig = {
  [eventName: string]: {
    sample_rate: number
  }
}
```

作用：

- 控制不同事件的采样率

特点：

- 动态配置驱动
- 不同事件可独立采样

## 13.4 `GrowthBookUserAttributes`

文件：`src/services/analytics/growthbook.ts`

这是实验定向的用户画像结构，包含：

- device / session
- platform
- organization / account
- subscriptionType
- rateLimitTier
- email
- appVersion

作用：

- 支持服务端做实验定向
- 支持曝光日志带上可解释上下文

## 13.5 `StoredExperimentData`

```ts
type StoredExperimentData = {
  experimentId: string
  variationId: number
  inExperiment?: boolean
  hashAttribute?: string
  hashValue?: string
}
```

作用：

- 把“feature 值”与“实验分流信息”关联起来

这是后续 exposure logging 的依据。

## 13.6 `EventMetadata`

文件：`src/services/analytics/metadata.ts`

这是内部分析的核心元数据结构之一，包含：

- model
- sessionId
- userType
- envContext
- clientType
- processMetrics
- agentId / parentSessionId / agentType / teamName
- subscriptionType
- repo remote hash

它本质上是：

```text
一条事件的上下文快照
```

## 13.7 `FirstPartyEventLoggingMetadata`

同样在 `metadata.ts` 中。

它把 `EventMetadata` 进一步转换为：

- `env`
- `process`
- `auth`
- `core`
- `additional`

这是 1P event logging 最终写入 proto / BigQuery 前的中间结构。

## 13.8 `SpanContext`

文件：`src/utils/telemetry/sessionTracing.ts`

```ts
interface SpanContext {
  span: Span
  startTime: number
  attributes: Record<string, string | number | boolean>
  ended?: boolean
  perfettoSpanId?: string
}
```

作用：

- 把 OTel span 和本地管理信息绑在一起

它是 session tracing 的核心运行时对象。

---

## 14. 关键算法与机制总结

这里把最值得初学者记住的机制再收敛一下。

## 14.1 事件排队与延迟挂载

目的：

- 解决“调用点比 sink 初始化更早”的问题

核心思想：

- 先缓存，后异步回放

## 14.2 多级配置读取优先级

GrowthBook 读取顺序大致是：

```text
env override
  > local config override
  > in-memory remote eval cache
  > disk cache
  > default
```

这是一个非常实用的配置优先级模型。

## 14.3 曝光延迟记录与去重

策略：

- feature 访问时才记 exposure
- init 前访问先放到 `pendingExposures`
- session 内靠 `loggedExposures` 去重

优点：

- 少记无意义曝光
- 不会在热路径刷爆曝光事件

## 14.4 失败落盘 + 跨进程补偿

这解决的是：

- CLI 随时退出
- 网络波动
- endpoint 暂时不可用

这是 1P event logging 最重要的可靠性保证。

## 14.5 tracing 上下文传播

靠：

- `AsyncLocalStorage`
- `activeSpans`
- `strongSpans`

把异步 agent 执行流串成树。

## 14.6 profiler 双模式

模式：

- sampled 线上统计
- detailed 本地深挖

这是一种成本和分析深度平衡得很好的设计。

---

## 15. 初学者应该怎么读这部分源码

推荐按下面顺序阅读。

### 第一步：先看入口，不要一上来啃 exporter

先读：

- `src/services/analytics/index.ts`
- `src/services/analytics/sink.ts`

目标：

- 理解“事件从哪里进、往哪里走”

### 第二步：再看 GrowthBook

重点读：

- `src/services/analytics/growthbook.ts`

目标：

- 理解为什么项目会有这么多 `CACHED_MAY_BE_STALE`
- 理解实验曝光、缓存、刷新、重建这几个动作

### 第三步：再看 1P event logging

重点读：

- `src/services/analytics/firstPartyEventLogger.ts`
- `src/services/analytics/firstPartyEventLoggingExporter.ts`

目标：

- 理解内部事件为什么要有独立 provider
- 理解失败落盘和退避重试

### 第四步：再看 telemetry tracing

重点读：

- `src/utils/telemetry/instrumentation.ts`
- `src/utils/telemetry/sessionTracing.ts`

目标：

- 理解 metrics / logs / traces 是怎么接入的
- 理解 interaction -> llm_request -> tool 的 span 模型

### 第五步：最后回到业务调用点

去看：

- `src/components/permissions/hooks.ts`
- `src/services/tools/toolExecution.ts`
- `src/services/api/logging.ts`

目标：

- 看清这套基础设施如何真正嵌进业务流

---

## 16. 最终总结

Claude-code-open 的“观测与实验架构”有三个最值得记住的特点。

### 16.1 它不是单一埋点系统，而是多条观测管道协作

至少同时存在：

- 内部 analytics
- 1P event logging
- Datadog
- 客户 OTLP telemetry
- tracing / profiler

### 16.2 它不是静态配置系统，而是运行时决策系统

GrowthBook 不只是“做实验”，还负责：

- gate
- dynamic config
- sink kill switch
- 采样率
- 批处理参数

### 16.3 它不是“尽力而为”的玩具实现，而是面向长会话 agent 的工程化实现

体现在：

- 启动早期事件排队
- 运行中配置刷新
- trace 上下文传播
- 批量投递与失败落盘
- 跨进程补偿重试
- 退出前 flush

如果只用一句话概括：

```text
这套架构把“观测、实验、性能剖析、运行时配置”合并成了 agent runtime 的基础设施层，
它既帮助开发者看清系统，也直接参与系统如何工作。
```
