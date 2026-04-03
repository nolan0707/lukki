# Claude-code-open 端云协同演进架构分析

## 1. 文档目标

当前 `Claude-code-open` 更像一个“本地 Agent OS”平台：

- 用户交互壳在本地
- 执行内核大多在本地
- 工具与权限决策大多在本地
- 长时上下文、会话、观测也大量偏本地化

如果未来要从“本地单机 Agent OS”演进到“端-云协同 Agent 平台”，核心问题不再是“能不能远程连一下”，而是：

1. 哪些模块天然应该留在端侧。
2. 哪些模块适合服务化、平台化，迁移到云端。
3. 哪些模块必须拆成 client/server 双层。
4. 如何在不破坏现有 Agent Runtime 协议的前提下完成演进。
5. 哪些设计红线不能碰，否则会损坏安全性、可控性和工程可维护性。

本文主要参考：

- [0001-project-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0001-project-arch.md)
- [0002-conversation-execution-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0002-conversation-execution-deep-arch.md)
- [0006-permission-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0006-permission-deep-arch.md)
- [0007-remote-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0007-remote-deep-arch.md)
- [0008-observable-deep-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0008-observable-deep-arch.md)
- [0009-modular-arch.md](/Users/lukki/codes/github/nolan0707/lukki/agent/specs/understand/0009-modular-arch.md)

---

## 2. 先给结论

从未来端云协同演进角度看，最合理的目标形态不是“把本地架构整体搬上云”，而是把系统拆成三层：

```text
端侧 Client Runtime
  本地 UI / 本地审批 / 本地沙箱 / 本地工具代理 / 本地短期状态

云侧 Agent Control Plane
  会话管理 / 任务调度 / 远程执行编排 / 插件分发 / 策略控制 / 观测分析

云侧 Agent Execution Plane
  云端 Query Runtime / 云端 Tool Runtime / 云工作区 / 云内存与索引 / 批处理任务
```

更具体地说：

### 应优先留在本地的模块

- 交互壳与终端 UI
- 本地权限审批 UI
- 本地沙箱与 OS 资源控制
- 强依赖本地文件系统/IDE/终端/桌面的工具
- 本地工作区感知、git 感知和设备级状态
- 高敏感凭据的最后一跳使用

### 应优先迁到云端的模块

- 会话控制面
- 任务/子 agent 调度
- 长时记忆、知识索引、检索增强
- 插件市场、配置下发、策略治理
- 远程工作区执行
- 观测分析、实验配置、全局运营数据面

### 最适合做双层拆分的模块

- 对话执行内核
- 命令与工具注册
- 权限系统
- MCP/插件/skills 接入层
- 上下文与记忆治理
- 远程协同模块

也就是说，未来演进的关键词不是“迁移”，而是：

- **分层**
- **代理化**
- **控制面/执行面分离**
- **端侧资源代理 + 云侧执行编排**

---

## 3. 设计判据：什么应该在本地，什么应该在云端

要判断模块放在哪里，建议用 6 个判据。

## 3.1 本地性判据

如果模块强依赖以下资源，则优先放在本地：

- 本地文件系统
- 本地 shell / tmux / PowerShell
- 本地 IDE / LSP / 编辑器
- 本地桌面 / 浏览器 / 剪贴板 / 通知
- 本地 git 工作区真实状态
- 用户实时审批与人机交互

这类能力的共同特征是：

- 延迟要求高
- 依赖设备环境
- 脱离端侧就无法正确执行

## 3.2 多租户平台化判据

如果模块天然是组织级或平台级能力，则优先放在云端：

- 统一策略管理
- 会话与任务编排
- 运营分析与实验
- 插件市场与分发
- 团队知识、组织记忆、共享索引
- 账号、权限、租户、计费

这类能力的共同特征是：

- 需要跨设备、跨会话共享
- 需要全局视角
- 不依赖单台机器的真实执行环境

## 3.3 数据重力判据

如果数据天然存在于某一侧，就要尽量让计算靠近数据：

- 本地代码仓库、本地未提交修改、本地 shell 历史，本地化执行更优。
- 组织级文档库、团队记忆、跨项目索引、插件仓库，云端化更优。

## 3.4 安全边界判据

如果模块需要作为“最终风险闸门”，应尽量留在本地：

- 本地文件写入授权
- 本地 shell/network 放行
- 桌面自动化审批
- 沙箱策略最终落地

因为云端即使做了逻辑授权，也不能替代端侧 OS 级控制。

## 3.5 弹性与成本判据

如果模块存在显著的并发/吞吐/批处理需求，更适合云端：

- 多 agent 并行调度
- 批量索引与 embedding
- 长时任务
- 共享缓存
- 跨会话聚合分析

## 3.6 一致性与可恢复性判据

如果模块天然属于“单设备瞬时状态”，更适合本地：

- 当前输入框状态
- 本地交互队列
- 临时 UI 弹窗
- 当前终端/窗口焦点状态

如果属于“跨设备/跨时间连续状态”，更适合云端：

- 会话元数据
- 任务状态
- agent 运行图
- 团队共享 memory

---

## 4. 推荐的未来端云协同分层

## 4.1 目标拓扑

未来更合理的拓扑不是单一的“thin client + full server”，而是：

```text
Client
  Local Interaction Shell
  Local Approval & Sandbox
  Local Resource Adapters
  Local Transcript Cache

Cloud Control Plane
  Session Service
  Task Scheduler
  Policy Service
  Plugin/Marketplace Service
  Observability & Experiment Service
  Team Memory / Index Service

Cloud Execution Plane
  Remote Query Runtime
  Remote Tool Runtime
  Workspace Runtime
  Background Agents
```

## 4.2 为什么要三层而不是两层

如果只分 client/server 两层，容易把以下两类完全不同的职责混在一起：

- 控制面：会话、策略、调度、观测、插件治理
- 执行面：真正跑 QueryEngine、工具、workspace、后台任务

这两者的扩缩容模型、故障模型、数据一致性要求都不同。

因此建议显式拆成：

- **Client Runtime**
- **Cloud Control Plane**
- **Cloud Execution Plane**

---

## 5. 模块放置建议总表

| 模块 | 当前形态 | 推荐归属 | 结论 |
| --- | --- | --- | --- |
| 入口与装配 | 本地 CLI 启动 | 本地 | 保留本地 |
| 交互与会话壳 | 本地 REPL/UI | 本地 | 保留本地 |
| 对话执行内核 | 主要本地 | 双层拆分 | 本地保留轻内核，云端承载远程执行内核 |
| 命令与工具注册 | 本地静态+动态装配 | 双层拆分 | 注册目录可双端共享，执行按工具类型分流 |
| 扩展生态 | 本地加载为主 | 双层拆分 | 发现/分发上云，执行适配分端 |
| 权限与安全治理 | 本地主导 | 双层拆分 | 策略上云，最终裁决与隔离留端 |
| 上下文与记忆治理 | 本地为主 | 双层拆分 | 本地短期上下文，云端长时记忆/索引 |
| 远程与分布式协同 | 已有雏形 | 云控 + 双端 | 大部分增强到云控层 |
| 观测与实验 | 混合 | 双层拆分 | 控制面和分析面上云，隐私缓冲与设备指标保留本地 |
| 基础设施 | 本地工具化 | 分裂 | 设备相关留端，平台基础设施上云 |

---

## 6. 适合保留在本地的模块

## 6.1 交互与会话壳模块

对应：

- `screens/REPL.tsx`
- `state/*`
- `context/*` 中与 UI 交互直接耦合的部分

### 为什么必须在本地

1. 这是用户体验入口，终端输入输出必须本地实时反馈。
2. 权限审批、弹窗、消息选择器、任务视图都依赖本地交互状态。
3. 即便执行迁到云端，用户仍然需要本地控制终端。

### 可上云的子部分

- transcript 同步
- 任务状态镜像
- 会话 title / metadata

### 结论

该模块整体应保留本地，只做“云端状态镜像”和“远程事件订阅”增强，不应迁移成云 UI 主导。

---

## 6.2 本地权限审批 UI 与最终风险闸门

对应：

- `components/permissions/*`
- `hooks/useCanUseTool.tsx`
- `REPL.tsx` 中审批交互部分

### 为什么要留在本地

1. 真正风险动作发生在本机资源上。
2. 用户需要在本地上下文中看到实际文件 diff、命令内容、工作目录。
3. 云端可做预判，但不能取代端侧最终“允许/拒绝”。

### 适合上云的部分

- 组织策略
- 默认规则
- classifier 模型
- 风险评分策略

### 结论

权限系统应演进成：

- 云端策略控制面
- 本地最终裁决与审批 UI

不能把最终授权完全迁到云端。

---

## 6.3 本地沙箱与资源访问控制

对应：

- `utils/sandbox/*`
- 本地 shell / filesystem / desktop / browser 等工具的 OS 侧约束

### 为什么必须留在本地

1. 沙箱本质上是操作系统执行边界，不是逻辑规则。
2. 本地文件写入、网络访问、子进程创建，最终都发生在设备侧。
3. 云端即使能做策略判断，也无法替代端侧强制执行。

### 结论

该模块必须留在本地，且在未来端云协同时重要性更高，而不是更低。

---

## 6.4 强本地依赖工具

包括但不限于：

- 文件读写编辑工具
- Bash/PowerShell 工具
- git/worktree/tmux 相关工具
- 本地 IDE/LSP 工具
- 桌面/浏览器/剪贴板/通知类工具

### 为什么保留本地

这些工具的价值恰恰来自“就地作用于本地开发环境”。

如果将其迁到云端，会出现 3 个问题：

1. 语义失真：云端看不到真实本地工作区和终端态。
2. 交互变慢：本地读写变成远程代理往返。
3. 安全边界变差：对用户来说，云端代替本地写文件更不可控。

### 结论

这些工具应本地保留，但要抽象成“本地资源代理接口”，以便云端 runtime 能通过受控协议调用它们。

---

## 6.5 设备态与临时态

包括：

- 输入缓冲
- 当前选中的消息
- 终端/窗口焦点
- 本地通知状态
- 临时 clipboard/paste 状态
- 本机 shell provider 解析结果

这些状态天然应留在端侧，不适合服务化。

---

## 7. 适合迁到云端的模块

## 7.1 会话控制面

对应当前分散在：

- `remote/*`
- `bridge/*`
- 一部分 `bootstrap/state.ts`
- 一部分 session metadata / session storage 逻辑

### 适合上云的原因

1. 会话天然是跨设备连续对象。
2. 断线重连、恢复、会话转移、viewer-only 附着都需要中心化管理。
3. 多端共享和后台会话天然需要服务端 session service。

### 云端化后职责

- session registry
- session lifecycle
- transcript authoritative store
- reconnect token / lease / worker binding
- multi-device attach

### 结论

会话控制面非常适合迁到云端，而且应优先迁移。

---

## 7.2 任务调度与多 agent 编排

对应：

- `tasks/*`
- `utils/swarm/*`
- `AgentTool` 背景运行部分
- `remote`/`bridge` 中对任务的支撑逻辑

### 适合上云的原因

1. 多 agent 并行执行天然需要统一调度器。
2. 后台任务与长任务不应绑定某个本地终端生命周期。
3. 任务状态、重试、超时、配额、优先级都更适合服务端控制。

### 云端化后收益

- 跨设备继续任务
- 异步批处理
- 统一 worker 池
- 长任务可靠恢复

### 结论

任务编排是第二优先级的云端化对象。

---

## 7.3 长时记忆、知识索引与共享检索

对应：

- `memdir/*`
- `services/SessionMemory/*`
- `services/extractMemories/*`
- 团队 memory sync 相关能力

### 适合上云的原因

1. 组织记忆、团队共享知识天然是跨设备、跨项目资产。
2. embedding、索引构建、检索聚合、去重压缩都更适合云端。
3. 长期来看，本地 memory 文件不适合承载大规模知识图谱。

### 推荐拆分

- 本地保留 session-local short memory cache
- 云端承载 organization/project/team/session 的 long memory service

### 结论

这是非常适合云端化的平台能力。

---

## 7.4 插件市场、扩展分发与集中治理

对应：

- `utils/plugins/*`
- `services/plugins/*`
- `plugins/*`
- marketplace 相关逻辑

### 适合上云的原因

1. 插件发现、版本管理、签名校验、策略控制天然是平台能力。
2. 企业级环境需要统一插件 allowlist / denylist。
3. 插件安装统计、分发策略、灰度升级更适合服务化。

### 哪些仍需在本地

- 插件本地 materialization
- 本地命令/工具适配
- 本地插件执行时的资源访问

### 结论

插件系统应拆成：

- 云端扩展控制面
- 本地扩展运行时

---

## 7.5 观测分析与实验控制面

对应：

- `services/analytics/*`
- `utils/telemetry/*`
- GrowthBook / 1P logging / OTel / profiler 的控制部分

### 适合上云的原因

1. feature flag、动态配置、采样率、实验分桶都需要全局视角。
2. 聚合分析、运营报表、异常检测、任务画像本来就属于云端分析面。
3. 端侧只适合做 buffering、隐私裁剪、轻量埋点，不适合承载分析系统本身。

### 结论

观测系统应变成：

- 端侧采集与隐私缓冲
- 云端配置与分析平台

---

## 7.6 云工作区与远程执行平面

如果未来希望真正承载“云端 agent 执行”，则以下能力也应迁到云端：

- 远程 QueryEngine runtime
- 云端工具 runtime
- 云工作区 filesystem / git workspace
- 后台长期执行任务

### 适用场景

- 云仓库分析
- PR review
- 后台大规模 refactor
- 多 agent 批处理
- 组织级知识库对接

### 结论

这不是所有场景都必须，但对“平台化 Agent”是重要演进方向。

---

## 8. 适合做双层拆分的模块

## 8.1 对话执行内核

对应：

- `QueryEngine.ts`
- `query.ts`
- `services/api/claude.ts`

### 为什么不能简单全迁

因为它既有纯逻辑执行部分，也有强本地耦合部分。

### 更合理的拆法

#### 本地保留

- 轻量 QueryEngine facade
- 本地 transcript cache
- 本地 tool proxy client
- 本地 interrupt / approval binding

#### 云端承载

- 远程 turn runtime
- 远程模型调用
- 远程上下文治理
- 远程后台 agent loop

### 未来形态

`QueryEngine` 变成一个抽象接口，有两种实现：

- `LocalQueryEngine`
- `RemoteQueryEngine`

这会是最健康的演进方式。

---

## 8.2 命令与工具能力模块

对应：

- `commands.ts`
- `tools.ts`
- `Tool.ts`
- `types/command.ts`

### 推荐拆法

#### 协议与目录层

可双端共享：

- `Command` schema
- `Tool` schema
- 工具元数据
- 命令元数据

#### 执行层

按资源位置分流：

- 本地工具由 client runtime 执行
- 云端工具由 server execution runtime 执行

### 关键原则

未来不应再把“工具”简单分成 builtin/cloud 两类，而应分成：

- `local-bound tools`
- `cloud-executable tools`
- `hybrid tools`

---

## 8.3 权限系统

### 推荐拆法

#### 云端

- 组织策略
- 默认规则
- 风险分类器
- 审计规则模板

#### 本地

- 设备态权限上下文
- 实时审批 UI
- 最终 allow/deny
- sandbox 强制执行

### 关键红线

云端只能做 policy recommendation 和 policy enforcement intent，不能取代端侧最终资源裁决。

---

## 8.4 MCP / Plugins / Skills

### MCP

- server registry、认证配置、企业 allowlist 可上云
- 本地连接器、端侧资源访问型 MCP server 仍需本地

### Plugins

- 市场、分发、签名、版本治理上云
- 本地 runtime adapter 保留

### Skills

- 共享 skill catalog 可上云
- 与本地工作区强相关的 skill 执行仍需端侧或端侧代理

---

## 8.5 上下文与记忆治理

### 推荐拆法

#### 本地

- 当前工作区即时上下文
- git status snapshot
- 当前 session 附件与消息
- 本地 CLAUDE.md 和即时 memory

#### 云端

- 长期 memory store
- 组织/团队 knowledge graph
- 跨会话 session summary
- embedding / indexing / retrieval

### 原则

短期、敏感、设备相关上下文在本地。  
长期、共享、可索引上下文在云端。

---

## 9. 不建议直接迁到云端的能力

## 9.1 本地文件编辑与命令执行

不建议将“直接对用户本地仓库执行写操作”迁到云端主导，因为：

1. 信任模型会急剧恶化。
2. 延迟和失败模式更复杂。
3. 审批体验更差。
4. 很难处理本地未保存缓冲区、IDE 状态和 shell 副作用。

正确做法是：

- 云端提出执行意图
- 本地代理执行
- 本地沙箱和审批兜底

## 9.2 终端主交互

终端交互若迁到云端主导，会导致：

- 输入回显不稳定
- 审批交互延迟高
- 本地设备感知差

因此 REPL 应坚持本地壳模式。

## 9.3 安全控制最终闭环

任何能影响本机资源的动作，其最终闭环都不应迁走。

---

## 10. 推荐的端云协同职责划分

## 10.1 Client 侧职责

建议端侧保留以下职责：

- 本地 REPL / UI / IDE 集成
- 本地权限审批与交互
- 本地沙箱
- 本地资源代理
- 本地 transcript cache
- 本地即时上下文采集
- 本地设备级 observability buffer

## 10.2 Server Control Plane 职责

建议云端控制面承担：

- 用户/组织/租户/权限策略
- session registry
- task scheduler
- worker orchestration
- plugin marketplace / skill catalog
- global config / experiments
- auditing / analytics / billing

## 10.3 Server Execution Plane 职责

建议云端执行面承担：

- 远程 QueryEngine runtime
- 远程 tool runtime
- 云工作区执行
- 后台 agent
- 长时 memory retrieval
- 大规模并行任务

---

## 11. 演进路线建议

## 11.1 第一阶段：控制面先上云

优先云端化：

- session service
- task service
- plugin/skill catalog
- policy/config service
- observability backend

此阶段本地仍是主执行面。

### 原因

这是收益最大、改动风险最小的路径。

---

## 11.2 第二阶段：执行内核双实现

引入：

- `LocalQueryEngine`
- `RemoteQueryEngine`

同时把工具分成：

- local-bound
- cloud-executable
- hybrid

这时本地和云端都可以承载 turn runtime。

---

## 11.3 第三阶段：本地资源代理化

将本地强依赖能力抽象成稳定代理协议，例如：

- local_fs
- local_shell
- local_git
- local_ide
- local_desktop

让远程 runtime 可以通过受控协议调用它们。

这一步是从“本地 Agent OS”走向“端云协同 Agent OS”的真正关键。

---

## 11.4 第四阶段：长时记忆与多 agent 平台化

最终把：

- 长时 memory
- shared retrieval
- agent graph
- background workflows
- enterprise policy

统一收敛成平台能力。

---

## 12. 迁移过程中的关键红线

## 12.1 不要破坏 `Command` / `Tool` / `Message` 三大协议面

未来演进应复用这些稳定协议，而不是重写一套 client/server RPC 语义。

## 12.2 不要让云端越过本地最终安全边界

任何本地资源操作都必须允许端侧最终 veto。

## 12.3 不要把 AppState 直接服务端化

`AppState` 本质上是端侧会话壳状态，不应被当作服务端 authoritative state。

## 12.4 不要把所有工具都统一迁云

工具必须按资源绑定类型分类，而不是按“是否先进”分类。

## 12.5 不要忽视断网/弱网模式

端云协同系统必须保留：

- 本地退化执行
- 云端断开后的局部可用性
- transcript 与任务状态恢复能力

---

## 13. 最终建议

如果从架构演进价值和实现风险综合排序，建议优先级如下：

1. 先把会话、任务、插件/策略、观测这些控制面能力云端化。
2. 再把执行内核抽象为本地/远程双实现。
3. 再把本地资源能力代理化，而不是直接云端接管。
4. 最后再构建真正的云端执行平面和长时记忆平台。

一句话概括：

**未来不应该把 Claude-code-open 从“本地 Agent OS”改造成“纯云 Agent”，而应该把它演进成“端侧交互与资源主权 + 云侧控制与计算放大”的协同 Agent 平台。**

这条路线既保留了 Claude Code 现有架构最强的部分：

- 本地可控性
- 工具真实环境执行
- 权限与沙箱闭环
- 交互即时性

又能获得平台化的核心收益：

- 任务调度
- 多 agent 并行
- 组织级记忆
- 统一治理
- 远程执行与跨设备连续性

这才是从“本地 Agent OS”走向“端云协同 Agent OS”的合理演进方向。
