Claude Code 的 Memory（记忆）系统是其核心架构中极其精妙的部分，其设计初衷是解决 “上下文熵增”（Context Entropy） 问题，即防止长会话中因信息冗余导致的模型幻觉或逻辑矛盾。
根据源码拆解和架构分析，Claude Code 的 Memory 系统主要通过以下机制实现：
1. 三层级联记忆架构 (Three-Layer Memory Architecture)
Claude Code 放弃了传统的“全量上下文填充”模式，构建了一个结构化的三层体系：
第一层：In-Context Memory（活跃记忆）：这是目前加载在模型上下文窗口中的实时内容，包括对话历史、最近的工具输出和正在编辑的文件。系统将其视为“临时草稿本”，处理即时推理任务，但不存储持久知识。
第二层：External File Memory（外部文件记忆/索引层）：这是该系统的核心，由一个名为 MEMORY.md 的轻量级文件驱动。它不直接存储详细信息，而是作为 “指针索引”，每行仅约 150 个字符，指向分布在磁盘上的具体主题文件（如 decisions.md 或 code-patterns.md）。
第三层：Structural Project Memory（项目结构记忆/静态层）：由 CLAUDE.md 文件组成，存放稳定的项目背景、编码标准、架构目标和测试指令。它充当“项目宪法”，在每个会话开始时首先被读取，提供持续一致的行为约束。
2. “自愈”与严格写入纪律 (Self-Healing & Strict Write Discipline)
Memory 系统具有 “自愈” 能力，这意味着 Agent 负责主动维护自己记忆的准确性：
读后写纪律 (Strict Write Discipline)：Agent 只有在文件写入操作（FileWrite）真正成功后，才被允许更新记忆索引。这确保了记忆库中永远不会被失败的尝试、错误的猜测或回滚的操作所污染。
怀疑性检索 (Skeptical Retrieval)：Harness（治理框架）指令 Agent 将记忆视为 “提示（Hint）”而非“真理”。在执行高风险操作前，Agent 必须通过工具（如 Grep 或 FileRead）重新验证代码库的实时状态，以确保其基于当前事实而非陈旧记忆采取行动。
3. 后台整合机制：autoDream (Memory Consolidation)
Claude Code 包含一个代号为 KAIROS 的后台守护进程，其中最关键的任务是执行 autoDream 过程，类似于人类在睡眠中巩固记忆：
触发条件（三门系统）：仅在满足距离上次“梦境”超过 24 小时、已进行至少 5 个会话且成功获取全局锁时才会启动。
四阶段整合逻辑：
Orient（定向）：扫描目录并熟悉现有记忆。
Gather（收集）：从最近的日志和转录中提取新信号。
Consolidate（整合）：合并观察结果、消除矛盾、将相对时间转换为绝对时间。
Prune（修剪）：强制将 MEMORY.md 保持在 200 行以内或 25KB 以下，移除过时指针。
4. 上下文压缩管理 (Context Compaction)
为了在有限的 Token 窗口（如 200K 或 1M）内实现无限会话，Memory 系统与 Query 引擎配合进行多级压缩：
MicroCompact（微压缩）：在本地直接编辑或修剪旧的工具输出内容（如清理冗长的 Grep 结果），不产生 API 成本。
AutoCompact（自动压缩）：当 Token 计数接近上限并触及预留的 13,000 Token 缓冲带 时触发。系统会调用 LLM 生成一段结构化的会话摘要来替换冗长的历史消息。
回路断路器：如果连续三次压缩失败，系统会强制停止，以防止产生无限循环或天价账单。
通过这种 “索引化、怀疑化、后台化” 的设计，Claude Code 的 Memory 系统成功地在极长的开发周期内保持了 Agent 的连贯性和智力，成为了工业级 Agent 设计的典型范式。
