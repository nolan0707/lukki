# 云侧 Agent 调用端侧工具的开销优化设计

## 1. 文档目标

在 [0010-client-server-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0010-client-server-arch.md) 中，我们已经给出了从“本地 Agent OS”向“端云协同 Agent 平台”演进的总体方向。

其中一个最关键的工程问题是：

**当云侧 agent 需要频繁访问端侧文件、命令、IDE、桌面或本地工作区资源时，如何避免细粒度跨端调用导致的巨大网络开销、时延抖动和交互退化。**

这个问题如果处理不好，会直接导致：

1. 云侧执行看似先进，但实际体验比本地更慢。
2. 工具调用链被网络往返放大，吞吐急剧下降。
3. 权限审批、上下文构建、文件读取等高频操作都变成系统瓶颈。
4. agent 为了弥补信息不足而发起更多调用，形成恶性循环。

本文目标是提出一套可落地的优化设计，回答：

1. 问题的本质是什么。
2. 应该采用什么总体架构模式解决。
3. 如何通过批处理、缓存、快照、自治执行减少跨端调用次数。
4. 协议层、运行时层、工具层、权限层需要做哪些改造。
5. 如何分阶段落地，而不是一次性大重构。

---

## 2. 一句话结论

解决“云侧 agent 频繁调用端侧工具导致网络开销过高”的正确方向，不是单纯优化 RPC，而是：

**把端侧从“被动工具服务器”升级为“带缓存、带索引、可批处理、可短时自治的本地执行代理”。**

更具体地说，应同时做 5 件事：

1. **减少调用次数**  
   从细粒度 tool RPC 改成粗粒度 task/plan 下发。

2. **减少重复取数**  
   用端侧缓存、工作区快照、索引和版本戳避免反复回源。

3. **减少往返轮次**  
   用批处理、流水线和端侧局部自治，把 N 次交互收敛成 1 次回合。

4. **减少上传体积**  
   结果分层返回，优先摘要和结构化信息，按需拉全量。

5. **减少不必要的端侧依赖**  
   把可云化的能力留在云端，把必须本地执行的能力明确收敛为 local-bound tools。

如果压缩成一张图，可以表示为：

```text
旧模式
  Cloud Agent
    -> read_file
    -> grep
    -> ls
    -> read_file
    -> bash
    -> read_file
    -> ...
  Edge Client

新模式
  Cloud Planner
    -> 下发 plan / task bundle
  Edge Executor
    -> 本地批量读取
    -> 本地搜索
    -> 本地缓存命中
    -> 本地连续执行工具
    -> 回传摘要 / 差异 / 结构化结果
```

---

## 3. 问题本质

## 3.1 真正的问题不是“网络慢”，而是“调用粒度错了”

如果云侧 agent 仍沿用本地 runtime 的细粒度工具习惯，例如：

- 读一个文件
- 再决定读下一个
- 再 grep 一次
- 再看 git status
- 再读一个 diff

那么每个小动作都会跨端一次。

这会把原本在本地几毫秒完成的一串操作，放大成：

- 多次序列化/反序列化
- 多次网络往返
- 多次权限检查
- 多次队列调度
- 多次消息写回 transcript

因此瓶颈首先是**交互粒度**，其次才是网络栈。

## 3.2 这是 Agent 运行时问题，不是单个工具问题

该问题不能只靠优化某个 `read_file` 工具解决，因为高开销来自整条执行链：

```text
云侧推理
  -> 决定调用工具
  -> 发送请求到端侧
  -> 端侧执行
  -> 结果回云侧
  -> 云侧再推理
  -> 再决定下一次工具调用
```

也就是说，问题本质上是：

- **思考和执行被过度交替**
- **每次交替都跨越网络边界**

## 3.3 端云协同下的最坏模式

最坏情况是下面这种“网络回声式 agent”：

1. 云侧每次只拿到一点点信息。
2. 因信息不够，又发起下一次本地调用。
3. 本地每次只返回原始片段，不做结构提炼。
4. 云侧继续追问。
5. 最终形成几十次甚至上百次小工具调用。

这种模式会导致：

- 高延迟
- 高 token 消耗
- transcript 膨胀
- 权限提示频繁
- 用户体验断裂

---

## 4. 设计原则

针对这个问题，建议坚持 8 条原则。

## 4.1 优先减少调用次数，而不是优先压缩单次延迟

最有效的优化通常不是把单次 RPC 从 120ms 降到 80ms，而是把 30 次 RPC 降到 3 次。

## 4.2 优先把“读”和“分析”前移到端侧

端侧更接近真实工作区，适合：

- 批量读取
- 搜索
- 索引
- 版本判断
- 初步摘要

云侧应更多承担高层规划和复杂推理。

## 4.3 工具调用要从 RPC 模型升级为 Capability + Task 模型

不是“我调用这个函数”，而是“我把一个执行目标交给端侧能力代理完成”。

## 4.4 优先结构化摘要，而不是原始全量回传

很多云侧推理只需要：

- 文件名
- 命中位置
- 错误摘要
- diff 摘要
- symbol 列表

不需要每次都拿全文。

## 4.5 本地强绑定能力必须在端侧自治

如：

- filesystem
- shell
- IDE/LSP
- git workspace
- 桌面自动化

这些能力应端侧连续执行，而不是让云侧逐步遥控每一步。

## 4.6 权限边界不因优化而被绕过

批处理和自治不能成为跳过审批的理由。  
必须保留：

- 审批策略分级
- 可审计执行计划
- 本地最终 veto

## 4.7 结果要带版本戳

没有版本戳，就无法安全使用缓存和快照。  
端侧返回的任何高频信息都应尽量附带：

- file hash
- workspace version
- git HEAD
- index version
- snapshot id

## 4.8 协议要支持退化模式

如果：

- 端侧没有索引
- 端侧缓存失效
- 端侧自治器不可用

系统仍应退回到传统逐次调用模式，而不是整体失效。

---

## 5. 总体解法：四层优化模型

建议把优化方案拆成 4 层。

## 5.1 Layer 1: 工具粒度优化

目标：

- 把多个小工具调用合并成一个粗粒度调用。

典型方式：

- `read_files()` 替代多次 `read_file()`
- `workspace_search()` 替代多次 `grep/glob/ls`
- `collect_context()` 替代多个状态读取

## 5.2 Layer 2: 端侧缓存与快照

目标：

- 避免同一 turn、同一任务甚至同一 session 里重复读取同样内容。

典型方式：

- 文件缓存
- 搜索结果缓存
- workspace snapshot
- git 状态快照
- LSP/symbol 索引

## 5.3 Layer 3: 端侧局部自治执行

目标：

- 把一串本地相关操作在端侧连续执行，减少云端往返。

典型方式：

- local executor
- plan bundle
- subgoal execution
- bounded autonomy

## 5.4 Layer 4: 云侧规划与分叉控制

目标：

- 让云侧只负责高层规划、关键决策和异常处理。

典型方式：

- plan-first
- branch-point callback
- confidence-based escalation

这 4 层一起作用，才能真正解决问题。

---

## 6. 协议层改造建议

## 6.1 从 Tool RPC 升级为 Edge Capability Protocol

未来不应让云侧直接暴露“调用端侧 `read_file`”这种低层语义，而应引入统一的端侧能力协议。

例如：

```ts
type EdgeCapabilityRequest =
  | { type: 'collect_workspace_context'; ... }
  | { type: 'batch_read'; paths: string[]; ... }
  | { type: 'workspace_search'; queries: SearchQuery[]; ... }
  | { type: 'execute_plan_bundle'; plan: EdgePlanBundle; ... }
  | { type: 'apply_edit_bundle'; edits: EditOperation[]; ... }
  | { type: 'run_validation_bundle'; checks: ValidationTask[]; ... }
```

这类协议的核心思想是：

- 抽象执行目标
- 而不是暴露过细的内部工具 API

## 6.2 结果协议要分层返回

建议每个端侧响应支持 3 级结果表达：

```ts
type EdgeResult =
  | { level: 'summary'; summary: string; refs?: Ref[] }
  | { level: 'structured'; data: Record<string, unknown>; refs?: Ref[] }
  | { level: 'full'; payload: unknown }
```

默认优先返回：

- summary
- structured

只有当云侧明确要求时，才拉 full payload。

## 6.3 引入版本戳与快照 ID

建议所有高频读取类响应都附带：

```ts
type ResultMeta = {
  workspaceVersion: string
  snapshotId?: string
  fileVersions?: Record<string, string>
  gitHead?: string
  indexVersion?: string
}
```

这样云侧才能：

- 判断结果是否可复用
- 避免重复请求
- 识别本地状态是否已变化

## 6.4 引入计划包协议

```ts
type EdgePlanBundle = {
  goal: string
  steps: EdgeStep[]
  autonomyPolicy: {
    maxSteps: number
    maxDurationMs: number
    requireApprovalFor: string[]
    stopOnUncertainty: boolean
  }
}
```

它允许云侧把多步意图一次下发给端侧，由端侧在受限边界内连续执行。

---

## 7. 批处理设计

## 7.1 文件读取批处理

不要：

```text
read_file(a.ts)
read_file(b.ts)
read_file(c.ts)
```

而应：

```text
batch_read([a.ts, b.ts, c.ts], with_metadata=true, chunk_policy=adaptive)
```

### 关键能力

- 去重路径
- 按目录聚合
- 自动裁剪大文件
- 统一附带 hash / size / language / lastModified

## 7.2 搜索批处理

云侧通常会在一轮推理里想做多个搜索，应允许一次请求内表达多个查询：

```ts
type WorkspaceSearchRequest = {
  queries: Array<
    | { kind: 'grep'; q: string }
    | { kind: 'glob'; pattern: string }
    | { kind: 'symbol'; name: string }
  >
}
```

这样端侧可以：

- 共享目录遍历
- 共享索引
- 合并 IO

## 7.3 编辑批处理

不要把修改拆成多个小 edit RPC。  
应允许端侧一次性接收 edit bundle：

- 多文件 patch
- 应用顺序
- 冲突检查
- dry-run 摘要

并支持：

- 先预览
- 后审批
- 再实际写入

## 7.4 校验批处理

对本地验证类能力，可把：

- lint
- test
- typecheck
- targeted commands

一次性下发，端侧自己调度执行，并只回传：

- 失败摘要
- 关键日志片段
- 成功/失败矩阵

---

## 8. 端侧缓存与工作区快照

## 8.1 三类缓存

建议至少建立三层缓存。

### A. 短期请求缓存

作用范围：

- 同一 turn / 同一任务

缓存内容：

- 文件内容
- 搜索命中
- 目录树
- git status

### B. 会话缓存

作用范围：

- 同一 session

缓存内容：

- 常读文件
- symbol index
- 最近命令结果
- diagnostics

### C. 工作区快照

作用范围：

- workspace 版本级

缓存内容：

- 文件树摘要
- git HEAD / dirty set
- 文件 hash map
- search index

## 8.2 工作区快照的作用

工作区快照并不是为了替代文件系统，而是为了让云侧先知道：

- 这个工作区长什么样
- 哪些文件最近变了
- 哪些索引可用
- 哪些结果可以直接复用

这样云侧很多决策可以基于 snapshot 做，而不必每次回源本地。

## 8.3 缓存失效策略

缓存最怕错用，因此必须有清晰失效纪律：

1. 文件写入后，相关路径缓存立即失效。
2. git HEAD 变化后，依赖 HEAD 的上下文缓存失效。
3. 目录结构变化后，文件树和搜索缓存失效。
4. LSP/index 重建后，symbol cache 升级版本号。

## 8.4 返回“引用”而非总是返回“内容”

端侧可先返回：

- file ref
- line range
- snippet hash
- search hit ref

只有当云侧确认需要全文时，再二次拉取。

这会显著减少上行带宽。

---

## 9. 端侧局部自治执行

## 9.1 为什么必须有端侧自治器

如果没有端侧自治器，云侧仍会维持这种模式：

```text
读一点
想一下
再读一点
再想一下
```

哪怕做了批处理和缓存，也只能缓解，不能根治。

端侧自治器的目标是：

**让端侧在一个受限边界内连续完成若干本地动作，只在关键分叉点回调云侧。**

## 9.2 推荐角色划分

### 云侧 Agent

负责：

- 高层规划
- 复杂推理
- 目标分解
- 关键分叉决策
- 最终结果整合

### 端侧 Executor

负责：

- 连续执行本地工具
- 本地缓存命中
- 本地索引查询
- 初步摘要
- 计划内的有限自治

## 9.3 自治边界

端侧自治不应无限制。建议限制：

- 最大连续步数
- 最大执行时长
- 最大写操作数量
- 高风险动作必须停下来请求审批
- 不确定性超过阈值时回调云侧

## 9.4 三类端侧自治任务

### A. 上下文收集型

如：

- 批量读文件
- 收集 workspace 摘要
- 搜索和定位相关代码

这类最适合自治。

### B. 受控编辑型

如：

- 按 edit bundle 应用 patch
- 回传 dry-run diff

适合受限自治，但通常需要审批。

### C. 验证执行型

如：

- 本地跑测试
- 类型检查
- lint

这类也很适合自治，因为往往是成组执行。

---

## 10. 结果压缩与摘要策略

## 10.1 结果分层

不要默认把端侧工具全量输出回传到云侧。  
建议分为：

1. **摘要层**  
   适合大多数决策。

2. **结构化层**  
   适合程序化判断。

3. **全量层**  
   只在必要时拉取。

## 10.2 常见场景的返回策略

### 文件读取

- 默认返回文件摘要 + 关键片段 + 元数据
- 仅在云侧要求时返回全文

### 搜索

- 默认返回命中列表 + 少量上下文
- 不返回整个大结果集

### 测试

- 默认返回失败摘要和首个关键堆栈
- 成功用统计结果即可

### shell 输出

- 默认截断
- 附带 tail/head
- 大输出按引用存储，按需回拉

## 10.3 结构化优先于文本

云侧推理如果只消费长文本，很容易再次要求更多读取。  
因此端侧应尽量返回结构化字段，例如：

- `matchedFiles`
- `errorCount`
- `failedTests`
- `symbols`
- `changedPaths`

这能减少云侧二次追问。

---

## 11. 权限与安全如何配合优化

## 11.1 批处理不能绕过权限

如果一个批处理请求里包含：

- 只读步骤
- 写步骤
- 网络步骤

则端侧必须按步骤风险分组，而不是整体一次放行。

## 11.2 计划级审批优于步骤级审批

相比每一步都询问用户，更好的方式是：

- 先展示一个 plan bundle
- 用户一次批准本轮计划
- 端侧在范围内自治执行

这样既减少审批次数，也减少网络往返。

## 11.3 本地最终 veto 不变

无论云侧多智能，端侧仍要保留：

- 对高风险动作的最终否决权
- 对本地资源访问的强制控制

---

## 12. 推荐的工具分类法

为支持优化，未来建议把工具分成三类。

## 12.1 `cloud-native tools`

可完全在云端执行，例如：

- 云知识库检索
- 云端代码分析
- 组织级搜索
- 远程工作区任务

## 12.2 `local-bound tools`

必须在端侧执行，例如：

- 本地文件编辑
- 本地 shell
- 本地 git workspace
- IDE/LSP
- 桌面自动化

## 12.3 `hybrid tools`

可优先使用云侧或索引结果，必要时回源端侧，例如：

- workspace search
- symbol lookup
- memory retrieval
- diagnostics

这套分类非常重要，因为它直接决定是否需要跨端调用。

---

## 13. 落地路线

## 13.1 第一阶段：先做批处理接口

优先实现：

- `batch_read`
- `workspace_search`
- `apply_edit_bundle`
- `run_validation_bundle`

这是投入最小、收益最快的阶段。

## 13.2 第二阶段：引入端侧缓存与快照

实现：

- workspace snapshot
- file hash/version
- 会话级搜索缓存
- git 状态快照

这会显著减少重复取数。

## 13.3 第三阶段：引入端侧 executor

实现：

- `execute_plan_bundle`
- bounded autonomy
- 关键分叉回调

这是从“工具代理”走向“本地执行代理”的关键一步。

## 13.4 第四阶段：云侧 planner 模式化

让云侧 agent 优先输出：

- 计划
- 子目标
- 端侧执行包

而不是继续逐步发低层工具指令。

---

## 14. 风险与权衡

## 14.1 风险一：端侧自治过强，行为不可控

解决方式：

- 强限制自治边界
- 保留审批
- 做完整审计日志

## 14.2 风险二：缓存导致读到陈旧状态

解决方式：

- 版本戳
- 明确失效纪律
- 关键读回源校验

## 14.3 风险三：批处理请求过大

解决方式：

- 请求分片
- 自适应 chunk
- 设置 payload 上限

## 14.4 风险四：协议过重，导致实现复杂度上升

解决方式：

- 先从 2-3 个最高频能力做起
- 保留兼容旧 RPC 模式
- 逐步替换

---

## 15. 最终建议

如果只给一条最重要的建议，那就是：

**不要让云侧 agent 直接频繁遥控端侧工具，而要让端侧升级为一个可缓存、可批处理、可短时自治的本地执行代理。**

具体落地优先级建议如下：

1. 先把高频只读调用改造成批处理接口。
2. 再补工作区快照和缓存。
3. 再引入 plan bundle 和端侧 executor。
4. 最后让云侧 agent 从“逐步遥控者”转成“高层规划者”。

一句话总结：

**优化跨端工具调用的本质，不是把网络做得更快，而是让跨端边界只承载高价值决策，而不是承载每一个低层动作。**
