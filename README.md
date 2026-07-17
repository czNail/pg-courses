# PostgreSQL 内核课程

面向有 PostgreSQL 使用经验、准备进入源码级内核研发、问题定位和性能诊断的工程师的中文课程系列。

每节课围绕一个核心内核矛盾展开，沿"问题 -> 状态 -> 主流程 -> 边界 -> 异常 -> 诊断"形成可线性阅读的推理链。不是源码百科或 API 清单。

## 目录

| 目录 | 简介 |
| --- | --- |
| [01-core-infrastructure](01-core-infrastructure/README.md) | 内核基础设施：Shmem、MemoryContext、ResourceOwner、PGPROC、同步原语、DSM、并行上下文、进程生命周期、重量级锁、Catalog/Cache |
| [02-storage-persistence](02-storage-persistence/README.md) | 存储与持久化：Buffer Manager、WAL、Storage Manager、Checkpointer、HeapAM DML/HOT、B-tree、Index AM API |
| [03-transaction-visibility](03-transaction-visibility/README.md) | 事务与可见性：XID/CLOG/Subtrans、Snapshot、Heap Tuple Visibility、Row Lock/MultiXact、VACUUM/Freeze、SSI、Autovacuum |
| [04-executor-observability](04-executor-observability/README.md) | Executor 与可观测性：Executor 生命周期、PlanState/Slot/ExprContext、执行节点、EXPLAIN ANALYZE、pg_stat、Wait Event、协议/Plan Cache、表达式/FMGR/JIT、分区路由 |
| [05-planner-optimizer-statistics](05-planner-optimizer-statistics/README.md) | Planner / Optimizer / Statistics：Parser/Analyzer/Rewrite、Planner 主流程、统计信息、选择率、Cost Model、RelOptInfo/Path、Join Search、Upper Planning、DDL、分区 |
| [06-replication-recovery-logical-decoding](06-replication-recovery-logical-decoding/README.md) | 复制、恢复与逻辑解码：Physical Streaming、Replication Slot、Recovery/Hot Standby、Timeline、Reorder Buffer、Logical Decoding、Output Plugin、Subscription、延迟诊断 |

## 课程写作标准

参见 [00-course-writing-standard.md](00-course-writing-standard.md)。
