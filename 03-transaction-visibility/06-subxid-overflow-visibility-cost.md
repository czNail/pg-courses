# PostgreSQL SubXID overflow 与可见性成本

## 课程定位

前置知识：

- 已经理解 PostgreSQL MVCC tuple header 中 `xmin`、`xmax`、hint bit 的基本含义。
- 已经理解普通 snapshot 的 `xmin`、`xmax`、`xip` 表示“哪些 XID 在快照时仍在运行”。
- 已经知道 `SAVEPOINT`、PL/pgSQL exception block、nested transaction 在 PostgreSQL 内部会产生 subtransaction。
- 已经能在源码里区分 backend-local state、shared memory state 和 SLRU-backed state。

本节唯一主问题：

```text
当一个顶层事务产生超过 `PGPROC` 可缓存数量的 SubXID 时，PostgreSQL 如何仍然判断 tuple 对某个 MVCC snapshot 是否可见，并且这个正确性 fallback 会把成本传播到哪些 hot path？
```

本节围绕的核心矛盾：

- 可见性判断希望在 tuple hot path 上只做便宜的本地数组判断。
- 但一个事务可以产生很多子事务，系统不能为每个 backend 在 shared memory 中无限保存完整 SubXID 集合。
- PostgreSQL 选择在 `PGPROC` 中缓存有限数量的 SubXID，overflow 后退回 `pg_subtrans` 追溯 parent chain。
- 这个选择保住了 correctness 和 shared memory 上界，但把额外 CPU、SLRU lookup、锁和 cache miss 成本延迟传播到 snapshot 与 tuple visibility。

学完本节后，应能独立判断：

- `suboverflowed` 出现时，快照并没有失效，只是从“完整枚举子事务”退化为“需要映射到顶层事务”。
- `PGPROC->subxids` 的 64 槽不是子事务数量上限，而是 shared-memory fast path 的缓存上限。
- `pg_subtrans` 不是 commit log，也不回答一个 XID 是否提交，只回答 subtransaction 的 parent 是谁。
- 一次 SubXID overflow 的成本不只由制造 overflow 的 backend 承担，读它影响过的 tuple 的 backend 也会承担。
- 常规 SQL 视图很难直接看到 overflow，很多诊断只能从 workload、profile、断点或源码计数器推断。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节课通常会把 MVCC 讲成一个快照集合问题：

- snapshot 记录哪些事务在创建快照时仍未结束。
- heap tuple 的 `xmin` 和 `xmax` 对照 snapshot 得到可见性。
- hint bit 把部分 commit/abort 状态缓存到 tuple header 上，减少后续查询。

这套模型在没有子事务，或者子事务数量很少时很顺。难点从这里开始：

- PostgreSQL 支持 savepoint。
- PL/pgSQL exception block 会隐式使用 subtransaction。
- 每个子事务如果执行了需要 XID 的写操作，就可能得到自己的 XID。
- tuple header 存的是实际修改 tuple 的 XID。
- 这个 XID 可能是顶层事务 XID，也可能是某个 SubXID。

因此可见性判断不能只问：

- 这个顶层 XID 是否在 snapshot 中？

还必须问：

- 如果 tuple 的 `xmin` 或 `xmax` 是 SubXID，它属于哪个顶层事务？
- 这个顶层事务在我的 snapshot 中是否仍在运行？
- 如果子事务已经单独 abort，是否应该把它看作不再运行？

PostgreSQL 处理这个问题的 fast path 是：

- 每个 backend 在 `PGPROC` 中公开当前顶层事务的 XID。
- 同时公开最多 `PGPROC_MAX_CACHED_SUBXIDS` 个 still-running SubXID。
- snapshot 创建时复制这些 SubXID 到 `snapshot->subxip`。
- tuple visibility 检查时先用数组搜索回答“它是否还在运行”。

这节课只讨论一个退化点：

- 当前顶层事务产生超过 64 个缓存槽能容纳的 SubXID。
- `PGPROC` 不再能完整公开子事务集合。
- snapshot 上的 `suboverflowed` 被设置。
- 后续判断需要查询 `pg_subtrans`。

这个退化点值得单独成课，因为它同时影响：

- 可见性 correctness。
- snapshot 获取成本。
- heap scan tuple visibility 成本。
- vacuum 判断 still-running transaction 的成本。
- hot standby 的 KnownAssignedXids 处理。
- 事务 abort 和 cleanup 的资源释放。

本节不展开的内容：

- 不系统讲整个 transaction block state machine。
- 不讲 SSI predicate lock。
- 不讲 MultiXact 的 locker/updater 语义。
- 不讲 logical decoding 的完整 reorder buffer。
- 只在它们影响 SubXID overflow 主链路时提及边界。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
SubXID overflow 不是子事务失败，而是 `PGPROC` 子事务枚举信息不完整；快照把这个不完整性保存为 `suboverflowed`，tuple 可见性在需要时通过 `pg_subtrans` 把 SubXID 映射到顶层 XID，再用顶层 XID 与 snapshot 比较。
```

核心矛盾有三层。第一层：shared memory 上界 vs 任意多 savepoint。

- `PGPROC` 是 shared memory 中的 per-process 状态。
- 所有 backend 获取 snapshot 时都会扫描 ProcArray。
- 如果每个 backend 的 `PGPROC` 都保存无限 SubXID，shared memory 会失去上界。
- 如果保存大量可变长度数组，snapshot 获取会更慢，内存 locality 也更差。
- PostgreSQL 固定缓存 64 个 SubXID。
- 这使 `PGPROC` 大小可控，也使 common case 的 snapshot copy 便宜。

第二层：tuple hot path 成本 vs overflow correctness。

- heap scan 的每个候选 tuple 都可能调用 `HeapTupleSatisfiesMVCC()`。
- 这个路径必须极度便宜。
- 非 overflow 时，`XidInMVCCSnapshot()` 主要做范围判断和 `xip/subxip` 数组搜索。
- overflow 后，仅凭 `subxip` 不能证明某个 SubXID 不在运行。
- 系统必须能找出 SubXID 所属的顶层事务。
- 这个映射由 `pg_subtrans` 提供。

第三层：快照稳定性 vs 事务状态继续变化。

- snapshot 创建后，它表达的是“创建时的运行集合”。
- 之后事务可能 commit 或 abort。
- `HeapTupleSatisfiesMVCC()` 不会为了一个旧 snapshot 去重新读取最新真实运行状态。
- 它对 snapshot 中仍运行的 XID 继续按 running 处理。
- 这保持了 MVCC snapshot 的稳定性。
- 但当 snapshot overflow，它保存的是“不完整的子事务集合”。
- 后续必须在同一个 snapshot 语义下补足 SubXID 到 top XID 的映射。

关键结论：

- `suboverflowed` 是一种信息完整性标志，不是错误状态。
- `pg_subtrans` 是 correctness fallback，不是性能优化。
- overflow 成本通常在读路径显现，而不一定在制造 savepoint 的语句上显现。

本节主问题可以压缩为一个判断：

- 如果 `snapshot->suboverflowed == false`，一个 XID 不在 `subxip` 且不在 `xip`，通常就不是该 snapshot 下的 running XID。
- 如果 `snapshot->suboverflowed == true`，一个 XID 不在 `subxip` 不能说明它不 running；需要先把它映射到 top XID。

## 3. 核心文件分工与阅读顺序

推荐阅读顺序不是按文件名排序。先读状态定义，再读写入点，再读 snapshot，再读 visibility。

### 3.1 `src/include/storage/proc.h`

先看它，因为它定义了 shared memory fast path 的容量边界。核心状态：

- `PGPROC_MAX_CACHED_SUBXIDS`
- `XidCacheStatus`
- `XidCache`
- `PGPROC->xid`
- `PGPROC->xmin`
- `PGPROC->subxidStatus`
- `PGPROC->subxids`

源码事实：

- `PGPROC_MAX_CACHED_SUBXIDS` 是 64。
- 每个 backend 最多公开 64 个当前顶层事务下的 non-aborted subtransactions。
- `XidCacheStatus.count` 是当前缓存的 SubXID 数。
- `XidCacheStatus.overflowed` 表示有至少一个 SubXID 没放进 cache。
- `PGPROC->subxidStatus` 会 mirror 到 `ProcGlobal->subxidStates[i]`。

阅读重点：

- 不要把 `count` 当成总 SubXID 数。
- overflow 后 `count` 仍然最多 64。
- `overflowed == true` 才表示“子事务集合不完整”。

### 3.2 `src/backend/access/transam/varsup.c`

SubXID cache 的写入点在这个文件里。核心函数：

- `GetNewTransactionId(bool isSubXact)`

阅读重点：

- 分配 XID 时持有 `XidGenLock`。
- 对顶层事务，写 `MyProc->xid` 和 `ProcGlobal->xids[pgxactoff]`。
- 对子事务，如果 cache 未满，写 `MyProc->subxids.xids[nxids]`。
- 写数组后用 `pg_write_barrier()`，再增加 `count`。
- 如果 cache 已满，设置 `MyProc->subxidStatus.overflowed` 和 mirror 状态。

这段代码解释了两个不变量：

- 读者必须先读 count，再通过 read barrier 读数组内容。
- overflow flag 一旦在事务期间被设置，其他 backend 不能假设 SubXID 集合完整。

### 3.3 `src/backend/access/transam/xact.c`

这是 transaction state stack 和 subtransaction 生命周期的主文件。核心函数：

- `AssignTransactionId()`
- `TransactionIdIsCurrentTransactionId()`
- `AtSubCommit_childXids()`
- `RecordTransactionAbort()`
- `AtSubAbort_childXids()`
- `StartSubTransaction()`
- `CommitSubTransaction()`
- `AbortSubTransaction()`
- `CleanupSubTransaction()`

阅读重点：

- 子事务先确保 parent 有 XID。
- `AssignTransactionId()` 调 `GetNewTransactionId(isSubXact)`。
- 对子事务调用 `SubTransSetParent(child, parent)` 写 parent link。
- subcommit 不是独立 commit 到 pg_xact，而是把 child XID 合并到 parent 的 `childXids`。
- subabort 会记录 abort，并把失败的 XID 从 PGPROC running-child cache 中移除。
- `TransactionIdIsCurrentTransactionId()` 只回答“是不是当前 backend 自己的事务或已 subcommit child”。

### 3.4 `src/backend/access/transam/subtrans.c`

这是 `pg_subtrans` 的实现。核心函数：

- `SubTransSetParent()`
- `SubTransGetParent()`
- `SubTransGetTopmostTransaction()`
- `StartupSUBTRANS()`
- `ExtendSUBTRANS()`
- `TruncateSUBTRANS()`
- `CheckPointSUBTRANS()`

阅读重点：

- `pg_subtrans` 保存 child XID 到 immediate parent XID 的映射。
- 它是 SLRU-backed。
- 它没有 WAL interaction。
- crash 后不要求保留全部历史。
- startup 会把当前 active page 范围清零。
- 只需要记住 currently-open transactions 的 parent 信息。

最重要的边界：

- `pg_subtrans` 不告诉你这个 XID committed 还是 aborted。
- committed/aborted 由 `pg_xact` 负责。
- `pg_subtrans` 只用于从 SubXID 追溯到 top-level XID。

### 3.5 `src/backend/storage/ipc/procarray.c`

这是 snapshot 采集和 transaction-in-progress 判断的核心。核心函数：

- `GetSnapshotData()`
- `TransactionIdIsInProgress()`
- `ProcArrayEndTransaction()`
- `ProcArrayEndTransactionInternal()`
- `XidCacheRemoveRunningXids()`

阅读重点：

- `GetSnapshotData()` 持 `ProcArrayLock` shared 扫描 running backends。
- 它复制 top-level XID 到 `snapshot->xip`。
- 如果没有 overflow，它复制 SubXID 到 `snapshot->subxip`。
- 如果发现任意 backend 的 `subxidStates[i].overflowed`，就设置 `snapshot->suboverflowed`。
- `TransactionIdIsInProgress()` 在 overflow 情况下也会退回 `pg_subtrans`。
- 事务结束时在 `ProcArrayLock` exclusive 下清空 advertised XID 和 SubXID 状态。

### 3.6 `src/backend/utils/time/snapmgr.c`

核心函数：

- `XidInMVCCSnapshot()`

阅读重点：

- 它是 heap visibility 问“这个 XID 在我的 snapshot 中是否 still in progress”的函数。
- 非 overflow 时搜索 `subxip` 和 `xip`。
- overflow 时调用 `SubTransGetTopmostTransaction()`。
- 然后用 top-level XID 搜索 `xip`。

这个函数是本节的可见性成本放大器。

### 3.7 `src/backend/access/heap/heapam_visibility.c`

核心函数：

- `HeapTupleSatisfiesMVCC()`
- `HeapTupleSatisfiesVacuumHorizon()`
- `HeapTupleSatisfiesMVCCBatch()`

阅读重点：

- `HeapTupleSatisfiesMVCC()` 对 `xmin` 和 `xmax` 都可能调用 `XidInMVCCSnapshot()`。
- 它会先处理“是不是我自己的事务”。
- 对当前 snapshot 仍 running 的 XID，不会尝试用最新真实状态修正 hint bit。
- vacuum 相关路径更多使用 `TransactionIdIsInProgress()`，它也受 overflow 影响。

### 3.8 `src/include/access/xact.h` 与 `src/include/access/subtrans.h`

这些 header 给出外部边界。`xact.h` 暴露：

- `GetTopTransactionId()`
- `GetCurrentTransactionId()`
- `GetCurrentSubTransactionId()`
- `TransactionIdIsCurrentTransactionId()`
- transaction command 边界函数。

`subtrans.h` 暴露：

- `SubTransSetParent()`
- `SubTransGetParent()`
- `SubTransGetTopmostTransaction()`
- `StartupSUBTRANS()`
- `ExtendSUBTRANS()`
- `TruncateSUBTRANS()`

阅读 header 的价值：

- 看哪些能力是跨模块依赖。
- 看哪些能力没有暴露。
- 例如没有 API 能从 `pg_subtrans` 直接列出某个 parent 的全部 children。
- 这说明 parent chain 是单向的：child -> parent，而不是 parent -> children。

## 4. 关键数据结构与状态

### 4.1 `PGPROC` 中的 SubXID cache

`PGPROC` 是 shared memory 中每个 backend 的公开事务状态。本节关注这组字段：

- `PGPROC->xid`
- `PGPROC->xmin`
- `PGPROC->subxidStatus`
- `PGPROC->subxids`
- `ProcGlobal->xids[pgxactoff]`
- `ProcGlobal->subxidStates[pgxactoff]`

语义组合：

- `xid` 表示该 backend 当前顶层事务的 XID。
- `xmin` 表示该 backend 当前 snapshot 需要保留的下界。
- `subxidStatus.count` 表示 `subxids.xids[]` 中有效项数量。
- `subxidStatus.overflowed` 表示 `subxids.xids[]` 不完整。
- `ProcGlobal` mirror 用于更紧凑地扫描 ProcArray。

状态类型：

- `PGPROC` 是 static shared memory。
- `ProcGlobal` 也是 shared memory。
- `subxids.xids[]` 的实际数组在 `PGPROC` 中。
- 其他 backend 可以在持 `ProcArrayLock` 时读它。

容量边界：

- `PGPROC_MAX_CACHED_SUBXIDS == 64`。
- 每个 backend 当前顶层事务最多公开 64 个 SubXID。
- 第 65 个及之后的 SubXID 不会写入 `subxids.xids[]`。
- 取而代之的是设置 overflow flag。

注意 raw field 不是语义：

- `count == 64` 不能单独说明 overflow。
- `overflowed == true` 才说明“可能存在未缓存 SubXID”。
- `subxids.xids[i]` 只代表一部分 running SubXID。
- `xid + subxids + overflowed + ProcArrayLock` 才构成可解释状态。

### 4.2 `SnapshotData` 中的 `xip/subxip/suboverflowed`

`SnapshotData` 是 backend-local snapshot state。与本节相关字段：

- `xmin`
- `xmax`
- `xip`
- `xcnt`
- `subxip`
- `subxcnt`
- `suboverflowed`
- `takenDuringRecovery`
- `curcid`

普通 MVCC snapshot 语义：

- `xmin` 之前的 XID 不在运行。
- `xmax` 及之后的 XID 对该 snapshot 不可见。
- `xip[]` 保存 snapshot 创建时仍在运行的 top-level XID。
- `subxip[]` 保存 snapshot 创建时能完整枚举的 running SubXID。
- `suboverflowed` 表示 `subxip[]` 不完整。

状态边界：

- `SnapshotData` 本身是 backend-local。
- `xip` 和 `subxip` 通常由当前 backend 分配和复用。
- snapshot 创建后表示一个稳定视角。
- 它不是 shared memory truth 的实时 view。

非 overflow 判断：

- 如果 `xid < snapshot->xmin`，不 running。
- 如果 `xid >= snapshot->xmax`，running。
- 如果 `xid` 在 `subxip[]`，running。
- 如果 `xid` 在 `xip[]`，running。
- 否则不 running。

overflow 判断：

- 如果 `snapshot->suboverflowed == true`，不能相信 `subxip[]` 的缺席。
- 需要把候选 XID 通过 `pg_subtrans` 映射到 top-level XID。
- 再搜索 `xip[]`。

### 4.3 `pg_subtrans` 状态

`pg_subtrans` 是 SLRU-backed parent map。它保存：

- key：child transaction ID。
- value：immediate parent transaction ID。

它不保存：

- commit 状态。
- abort 状态。
- tuple visibility。
- parent 到 children 的反向索引。
- 长期历史。

关键性质：

- parent XID 总是先于 child XID 分配。
- `SubTransSetParent()` 断言 `xid` follows `parent`。
- `SubTransGetTopmostTransaction()` 沿 child -> parent chain 向上走。
- 如果走到 `TransactionXmin` 之前，可能返回中间 parent。
- 这在可见性判断中是可接受的，因为太老的 XID 已经不需要精确 parent。

生命周期边界：

- 启动时当前 active page 范围被清零。
- XID 分配推进到新 page 时 `ExtendSUBTRANS()` 清零页面。
- checkpoint 可调用 `TruncateSUBTRANS()` 删除不再需要的旧段。
- `CheckPointSUBTRANS()` 刷 dirty page 主要是为了减少 backend 后续写脏页概率，不是 crash correctness 必需。

资源边界：

- `subtransaction_buffers` 控制 `pg_subtrans` 的 SLRU buffer 数。
- 基线里 sample config 注释为 `subtransaction_buffers = 0`，表示自动配置。
- auto-tune 大致按 shared buffers 估算，并有上限。
- 这个 GUC 改变的是 fallback 的缓存命中概率，不改变 64 个 PGPROC cache 槽。

### 4.4 transaction state stack 中的 child XID

`xact.c` 维护 backend-local transaction stack。每个 `TransactionState` 关心：

- 当前层级是否有 `fullTransactionId`。
- 当前层级的 `subTransactionId`。
- parent 指针。
- `childXids` 数组。
- `nChildXids` 和 `maxChildXids`。
- ResourceOwner 和 MemoryContext。

subcommit 时：

- 子事务的 XID 不会立刻作为独立提交对外完成。
- `AtSubCommit_childXids()` 把自己的 XID 和已 subcommit children 合并到 parent 的 `childXids`。
- 顶层 commit 时统一记录 transaction tree 的 commit。

subabort 时：

- `RecordTransactionAbort(true)` 写 abort record，并把 abort 标记写入 `pg_xact`。
- 对 subxact，`XidCacheRemoveRunningXids()` 会立即从 PGPROC child cache 中移除已失败 XID。
- `AtSubAbort_childXids()` 释放本地 child XID 数组。

这解释了为什么 `PGPROC->subxids` 注释说是 non-aborted subtransactions。

### 4.5 tuple header 中的 XID 与本节关系

heap tuple header 不知道它保存的是 top XID 还是 SubXID。它只保存：

- `xmin`：插入该版本的 transaction ID。
- `xmax`：删除、更新或锁定该版本的 transaction ID 或 MultiXactId。
- hint bits：部分 commit/abort cache。

因此 heap 可见性必须处理：

- `xmin` 是当前事务 XID。
- `xmin` 是当前 backend 的 subcommitted child XID。
- `xmin` 是其他 backend 的 running top XID。
- `xmin` 是其他 backend 的 running SubXID。
- `xmin` 是已经 abort 的 SubXID。
- `xmin` 是已经 commit 的 SubXID，但 top-level commit 对当前 snapshot 不可见。

SubXID overflow 的影响出现在第四种和第六种。

## 5. 主流程源码 walkthrough

本节主流程从一个 runtime 场景开始。场景：

- session A 开启事务。
- session A 通过 PL/pgSQL exception block 或大量 savepoint 生成超过 64 个写入型子事务。
- session A 保持事务不提交。
- session B 获取 snapshot 并扫描 session A 修改过的表。
- session B 的 heap visibility 遇到某些 tuple 的 `xmin` 或 `xmax` 是 session A 的 SubXID。

### 5.1 子事务开始：先建立本地 stack

`SAVEPOINT` 不是马上把所有事务资源都建完。`xact.c` 里有一个历史痕迹：

- `DefineSavepoint` 更像是把 transaction block 状态推入子事务开始状态。
- 真正初始化由 `StartSubTransaction()` 在命令边界完成。

`StartSubTransaction()` 做的事情：

- 设置当前 `TransactionState` 为 `TRANS_START`。
- 调 `AtSubStart_Memory()` 创建子事务内存上下文。
- 调 `AtSubStart_ResourceOwner()` 创建子事务 ResourceOwner。
- 调 `AfterTriggerBeginSubXact()`。
- 切到 `TRANS_INPROGRESS`。
- 调 subxact callbacks。

此时不一定已经有 XID。PostgreSQL 延迟分配 XID：

- 只读子事务可能永远没有 XID。
- 只有执行需要 XID 的操作时，才调用 `GetCurrentTransactionId()` 或等价路径。
- 这避免把纯控制流 savepoint 都变成全局 XID 压力。

### 5.2 子事务分配 XID：parent 必须先有 XID

入口在 `AssignTransactionId()`。关键顺序：

```text
AssignTransactionId(s)
  -> 如果 s 是子事务且 parent 没有 XID
       先沿 parent chain 向上补齐 parent XID
  -> GetNewTransactionId(isSubXact)
       分配 FullTransactionId
       更新 PGPROC top XID 或 SubXID cache
  -> 如果是子事务
       SubTransSetParent(child_xid, parent_xid)
  -> XactLockTableInsert(child_or_top_xid)
  -> 必要时记录 XLOG_XACT_ASSIGNMENT
```

为什么 parent 必须先有 XID？

- `pg_subtrans` parent chain 依赖 child XID follows parent XID。
- `SubTransSetParent()` 断言 child follows parent。
- `SubTransGetTopmostTransaction()` 也依赖 parent 总是更老，避免循环。
- snapshot range check 也利用了 parent 和 child 的 XID 顺序关系。

这不是风格问题，是 correctness 边界。

### 5.3 `GetNewTransactionId()`：写入 PGPROC 或设置 overflow

`GetNewTransactionId(bool isSubXact)` 持有 `XidGenLock`。它先做全局 XID 分配相关工作：

- 检查 wraparound 限制。
- 扩展 `pg_xact`。
- 扩展 commit timestamp。
- 扩展 `pg_subtrans`。
- 推进 `TransamVariables->nextXid`。

然后写 ProcArray 可见状态。顶层事务：

```text
MyProc->xid = xid
ProcGlobal->xids[MyProc->pgxactoff] = xid
```

子事务：

```text
nxids = MyProc->subxidStatus.count
if nxids < PGPROC_MAX_CACHED_SUBXIDS:
    MyProc->subxids.xids[nxids] = xid
    pg_write_barrier()
    MyProc->subxidStatus.count = nxids + 1
    ProcGlobal->subxidStates[pgxactoff].count = nxids + 1
else:
    MyProc->subxidStatus.overflowed = true
    ProcGlobal->subxidStates[pgxactoff].overflowed = true
```

内存顺序边界：

- 写数组元素必须发生在增加 count 之前。
- 读者会先读取 count，再通过 `pg_read_barrier()` 读取数组。
- 否则其他 backend 可能看到 count 已增加，但数组槽位还没初始化。

overflow 边界：

- 第 65 个 SubXID 不会进入 `PGPROC->subxids.xids[]`。
- 它仍然有 XID。
- 它仍然会在 `pg_subtrans` 中写 parent。
- 它仍然可以修改 tuple。
- 它仍然受顶层事务 commit/abort 约束。
- 只是其他 backend 不能靠 PGPROC cache 完整枚举它。

源码注释里有一个微妙 race window：

- 当 cache 已满时，新 SubXID 不会出现在 PGPROC child array。
- `SubTransSetParent()` 在 `AssignTransactionId()` 中随后执行。
- 在 parent link 写入之前，新 XID 还不能通过 `pg_subtrans` 找到 parent。
- PostgreSQL 认为这可接受，因为此时还没有人有理由查询这个刚分配的 XID 的状态。
- 同时这期间创建的 snapshot 会包含 parent XID，之后仍能得出正确答案。

这个细节说明：

- 正确性不是靠“任何时刻所有信息都完整”。
- 正确性靠状态发布顺序、调用者可见边界和后续 fallback 共同成立。

### 5.4 写 parent link：`SubTransSetParent()`

子事务拿到 XID 后，`AssignTransactionId()` 调：

```text
SubTransSetParent(child_xid, parent_xid)
```

`SubTransSetParent()` 做的事情：

- 用 child XID 计算 SLRU page 和 entry。
- 获取对应 bank lock 的 exclusive 模式。
- 读取或创建 page。
- 找到 entry。
- 如果 entry 不是 parent，断言它原本是 invalid。
- 写入 parent。
- 标记 page dirty。
- 释放 lock。

写入的是 immediate parent。如果有多层 savepoint：

- child 只指向直接父事务。
- 直接父事务可能还是子事务。
- `SubTransGetTopmostTransaction()` 会循环向上走。

这解释了一个成本点：

- overflow fallback 的成本不只是一次 SLRU lookup。
- 嵌套很深时，可能沿 parent chain 多次 lookup。
- 虽然多数 workload 的 nesting depth 不大，但 PL/pgSQL 递归 exception 可能制造深链。

### 5.5 session B 获取 snapshot：`GetSnapshotData()`

session B 进入查询，获取 MVCC snapshot。主入口是 `GetSnapshotData(snapshot)`。

普通非 recovery 路径：

```text
GetSnapshotData(snapshot)
  -> 确保 snapshot->xip 和 snapshot->subxip 已分配
  -> LWLockAcquire(ProcArrayLock, LW_SHARED)
  -> 读取 latestCompletedXid，计算 xmax
  -> 初始化 xmin = xmax
  -> 扫描 ProcArray
       跳过无 XID backend
       跳过自己
       跳过 xid >= xmax
       跳过 lazy vacuum / logical decoding 特殊状态
       把 top-level xid 加入 snapshot->xip
       如果尚未发现 overflow:
           如果该 proc overflowed:
               suboverflowed = true
           else:
               复制该 proc 的 cached SubXID 到 snapshot->subxip
  -> 设置 MyProc->xmin / TransactionXmin
  -> LWLockRelease(ProcArrayLock)
  -> 填 snapshot->xmin/xmax/xcnt/subxcnt/suboverflowed
```

当 session A 已 overflow：

- B 扫到 A 的 `ProcGlobal->subxidStates[pgxactoff].overflowed == true`。
- B 设置本地 `snapshot->suboverflowed = true`。
- B 仍然把 A 的 top-level `xid` 加入 `snapshot->xip`。
- B 不再认为 `snapshot->subxip` 是完整子事务集合。

注意 `suboverflowed` 的传播方式：

- 它不是 per-backend 标志。
- snapshot 上只有一个 bool。
- 只要任何 relevant backend overflow，整个 snapshot 的子事务集合都被标记为不完整。

为什么这么设计？

- 可见性检查面对一个 tuple XID 时，tuple header 不带“来自哪个 backend”的信息。
- 如果 snapshot 中存在任意 overflowed backend，某个不在 `subxip` 的 SubXID 可能属于它。
- 因此不能按 backend 局部判断。

### 5.6 session B 扫 tuple：`HeapTupleSatisfiesMVCC()`

heap scan 进入 `HeapTupleSatisfiesMVCC()`。对 `xmin` 的核心判断可以压缩为：

```text
if xmin hint bit says invalid:
    invisible
if xmin is current transaction:
    use cmin/curcid rules
else if XidInMVCCSnapshot(xmin, snapshot):
    invisible, treat inserter as still running
else if TransactionIdDidCommit(xmin):
    set committed hint bit if allowed
else:
    set invalid hint bit and invisible
```

对 `xmax` 也有类似问题：

```text
if xmax invalid or lock-only:
    visible
if xmax is current transaction:
    use cmax/curcid rules
else if XidInMVCCSnapshot(xmax, snapshot):
    visible, treat deleter/updater as still running
else if TransactionIdDidCommit(xmax):
    invisible
else:
    xmax aborted, visible
```

本节只抓住一点：

- `XidInMVCCSnapshot()` 是 tuple hot path。
- 一次 heap scan 可能调用很多次。
- 如果 snapshot overflow，它可能触发 `pg_subtrans` lookup。

### 5.7 非 overflow 的 `XidInMVCCSnapshot()`

`XidInMVCCSnapshot(xid, snapshot)` 先做范围判断：

- `xid < snapshot->xmin`：不 running。
- `xid >= snapshot->xmax`：running。

范围判断非常关键：

- 大量 tuple 的 XID 会被这里直接过滤。
- 只有落在 `[xmin, xmax)` 的 XID 才需要数组搜索或 fallback。

普通非 recovery 且未 overflow：

```text
if !snapshot->suboverflowed:
    if xid in snapshot->subxip:
        return true
    if xid in snapshot->xip:
        return true
    return false
```

这里的成本主要是：

- 一次 `subxip` 搜索。
- 一次 `xip` 搜索。
- `pg_lfind32()` 是适合小数组的线性查找优化。
- 不需要 shared lock。
- 不需要 SLRU。

### 5.8 overflow 的 `XidInMVCCSnapshot()`

普通非 recovery 且 overflow：

```text
if snapshot->suboverflowed:
    xid = SubTransGetTopmostTransaction(xid)
    if xid < snapshot->xmin:
        return false
    if xid in snapshot->xip:
        return true
    return false
```

这里发生了语义切换：

- 非 overflow 时直接问候选 XID 是否在 `subxip/xip` 中。
- overflow 时先把候选 XID 规范化到 top-level XID。
- 然后只搜索 `xip`。

为什么 overflow 后不搜索 `subxip`？

- `subxip` 已经不完整。
- 即使候选 XID 不在 `subxip`，也不能证明它不是 running SubXID。
- 统一映射到 top-level XID 后，`xip` 仍然可靠。

为什么 `xip` 可靠？

- overflow 只影响 SubXID 缓存。
- top-level XID 仍然在 ProcArray 中公开。
- snapshot 创建时仍然收集了 relevant top-level XID。

### 5.9 `SubTransGetTopmostTransaction()` 的语义边界

`SubTransGetTopmostTransaction(xid)` 从 child 向 parent 走。主逻辑：

- 初始 `parentXid = xid`。
- 循环读取 `SubTransGetParent(parentXid)`。
- 保存上一个有效 XID。
- 如果 parent 早于 `TransactionXmin`，停止。
- 如果 parent 不 precedes child，认为结构异常并退出。
- 返回最后一个可接受的 parent。

源码注释明确承认：

- 因为不能回看早于 `TransactionXmin` 的 `pg_subtrans`，它可能返回中间 subtransaction，而不是真正 topmost transaction。

为什么这仍然正确？

- 可见性判断只关心 topmost parent 是否仍 running，或是否在当前 snapshot 的 running 集合里。
- 如果链条已经早于 `TransactionXmin`，那些 XID 对当前判断来说已经足够老。
- 返回中间 parent 不会把一个必须可见的 tuple 错判成不可见。

这个点很重要：

- PostgreSQL 在这里接受“足够精确”，不是追求绝对历史重建。
- `pg_subtrans` 的生命周期也因此可以短。

### 5.10 abort 子事务：从 running cache 中移除

如果一个子事务 abort，它不能继续被其他 backend 当成 running child。路径：

```text
AbortSubTransaction()
  -> RecordTransactionAbort(true)
       -> XactLogAbortRecord(...)
       -> TransactionIdAbortTree(xid, children)
       -> XidCacheRemoveRunningXids(xid, children, latestXid)
  -> AtSubAbort_childXids()
  -> subtransaction callbacks and resource cleanup
  -> CleanupSubTransaction()
```

`XidCacheRemoveRunningXids()`：

- 持 `ProcArrayLock` exclusive。
- 在 `MyProc->subxids.xids[]` 中删除指定 XID 和 children。
- 同步减少 `MyProc->subxidStatus.count` 和 `ProcGlobal->subxidStates[].count`。
- 如果找不到且未 overflow，发 warning。
- 如果已经 overflow，找不到是允许的，因为该 XID 可能根本没缓存过。
- 推进 `latestCompletedXid` 和 `xactCompletionCount`。

这里的 correctness 目标：

- aborted subtransaction 不应继续被当成 running。
- 对已经 overflow 的事务，不能要求每个 abort child 都能从 cache 找到。
- `pg_xact` abort 状态会在慢路径中帮助判断。

### 5.11 顶层 commit 或 abort：清空 PGPROC advertised state

顶层事务结束时，先完成 commit/abort 状态记录，再退出 ProcArray running 集合。commit 关键点：

- `RecordTransactionCommit()` 取得 committed children。
- 写 commit WAL record。
- 更新 commit timestamp。
- 标记 transaction tree committed。
- 可能等待同步复制。
- 返回 latestXid。

之后事务收尾会调用 `ProcArrayEndTransaction(MyProc, latestXid)`。`ProcArrayEndTransactionInternal()`：

- 持 `ProcArrayLock` exclusive。
- 清 `ProcGlobal->xids[pgxactoff]`。
- 清 `proc->xid`。
- 清 `proc->vxid.lxid`。
- 清 `proc->xmin`。
- 清 vacuum 状态 flags。
- 清 `ProcGlobal->subxidStates[pgxactoff].count`。
- 清 `ProcGlobal->subxidStates[pgxactoff].overflowed`。
- 清 `proc->subxidStatus.count`。
- 清 `proc->subxidStatus.overflowed`。
- 推进 `latestCompletedXid` 和 `xactCompletionCount`。

注意顺序：

- 事务 commit/abort 必须先在 `pg_xact` 等状态中可判定。
- 然后才能从 ProcArray 的 running 集合退出。
- 否则 concurrent snapshot 可能错过一个仍需判断的 XID。

## 6. 生命周期 / ownership / cleanup

### 6.1 谁创建 SubXID？

创建者：

- 当前 backend。
- 当前 transaction state stack 中的子事务层级。

触发时机：

- 子事务执行需要 XID 的操作。
- 常见是写 heap tuple、更新系统 catalog、某些锁或 WAL 相关路径要求 XID。

不是每个 savepoint 都创建 XID。

- `SAVEPOINT s; RELEASE s;` 如果没有需要 XID 的操作，可能不产生 SubXID。
- PL/pgSQL exception block 常常产生 SubXID，因为异常处理需要子事务边界。
- 只有生成真实 transaction ID 的子事务才进入本节链路。

### 6.2 谁持有 SubXID？

有四类 owner，要分清。backend-local transaction stack：

- `TransactionState.fullTransactionId` 持有当前层级 XID。
- parent stack 持有已 subcommit 的 `childXids`。
- `TopTransactionContext` 管这些数组的内存生命周期。

shared memory advertised state：

- `PGPROC->subxids.xids[]` 暴露最多 64 个 running child XID。
- `PGPROC->subxidStatus` 暴露 count 和 overflow。
- `ProcGlobal->subxidStates[]` 提供扫描友好的 mirror。

SLRU parent map：

- `pg_subtrans` 持有 child -> immediate parent。
- 它不归单个 backend 私有。
- 它是全局 fallback 状态。

tuple header：

- heap tuple 的 `xmin/xmax` 可以持有 SubXID。
- 它不持有 parent 信息。
- 它需要 snapshot 或 `pg_subtrans` 来解释。

### 6.3 谁释放或失效？

本地内存：

- subcommit 时，child 的 `childXids` 合并到 parent 后释放 child 数组。
- subabort 时，`AtSubAbort_childXids()` 释放当前子事务 child array。
- 顶层事务 cleanup 时，`TopTransactionContext` reset。

shared memory：

- subabort 时，`XidCacheRemoveRunningXids()` 从 PGPROC child cache 移除 failed SubXID。
- 顶层事务结束时，`ProcArrayEndTransactionInternal()` 清空 top XID 和 SubXID status。
- backend 退出 ProcArray 时也会清理 mirror 状态。

SLRU：

- `pg_subtrans` entry 不按单个子事务立即删除。
- 它通过 page/segment 生命周期回收。
- checkpoint 和 horizon 计算之后可 `TruncateSUBTRANS(oldestXact)`。
- startup 会重置 active pages。

tuple header：

- `xmin/xmax` 不因为事务结束而被释放。
- hint bit 可能在后续访问中被设置。
- VACUUM/pruning 可能最终移除死 tuple version。
- 这取决于 visibility horizon，不由 SubXID cleanup 直接决定。

### 6.4 ERROR / abort 时谁兜底？

ERROR 进入 abort path。子事务 abort 路径必须处理半初始化状态：

- `AbortSubTransaction()` 先 `AtSubAbort_Memory()` 和 `AtSubAbort_ResourceOwner()`。
- 然后释放 LWLock、buffer lock、condition variable sleep、lock wait、timeouts 等。
- 如果 `curTransactionOwner` 还没建立，后续清理要能跳过。
- 如果 XID 还没分配，`RecordTransactionAbort(true)` 直接返回 invalid。

这解释了为什么代码里经常检查：

- `FullTransactionIdIsValid(...)`
- `s->curTransactionOwner`
- `s->childXids != NULL`

子事务 abort 有两个可见性相关兜底：

- 写 `pg_xact` abort 状态。
- 从 PGPROC running child cache 移除能找到的 failed XID。

如果 cache 已 overflow：

- abort 的 SubXID 可能不在 cache 中。
- `XidCacheRemoveRunningXids()` 不把找不到当硬错误。
- 慢路径会用 `TransactionIdDidAbort(xid)` 避免把 already-failed subtransaction 当 running。

### 6.5 prepared transaction 边界

本节不展开 two-phase commit。只保留一个边界：

- prepared transaction 也可能有 transaction tree。
- 它不等同于普通 backend 的 live PGPROC state。
- 本节讨论的是当前 running backend 的 SubXID cache overflow 和 snapshot visibility。
- 读 prepared transaction 相关路径时，不要把普通 backend cleanup 顺序直接套过去。

## 7. 正确性机制层次

SubXID overflow 的正确性不是一个机制保证的。它至少叠了七层。

### 7.1 XID 分配顺序

保证：

- parent XID 先于 child XID。
- child XID follows parent XID。

依赖：

- `AssignTransactionId()` 先补 parent XID。
- `GetNewTransactionId()` 使用全局 XID 分配。
- `SubTransSetParent()` 断言 parent 先于 child。

不能保证：

- 它不保证 parent 已提交。
- 它不保证 child 可见。
- 它只提供 parent chain 可遍历的顺序基础。

### 7.2 ProcArray advertised running set

保证：

- running top-level XID 能被 snapshot 看到。
- 未 overflow 时，running child XID 能被 snapshot 完整复制。
- overflow 时，snapshot 能知道子事务集合不完整。

依赖：

- `ProcArrayLock`。
- `ProcGlobal->xids`。
- `ProcGlobal->subxidStates`。
- `PGPROC->subxids`。

不能保证：

- 它不保存无限 SubXID。
- overflow 后不保证 `subxids.xids[]` 完整。
- 它也不保存 commit/abort 状态。

### 7.3 memory barrier

保证：

- 子事务 XID 数组元素对读者可见后，读者才会看到增加后的 count。
- 读者在复制数组前用 read barrier 避免读未初始化槽位。

依赖：

- `pg_write_barrier()`。
- `pg_read_barrier()`。
- 按源码注释约定读写 `PGPROC->subxids`。

不能保证：

- barrier 不提供 transaction semantics。
- barrier 不替代 `ProcArrayLock`。
- barrier 不说明 XID committed/aborted。

### 7.4 snapshot 稳定性

保证：

- `SnapshotData` 创建后代表一个稳定视角。
- 对该 snapshot 来说，snapshot 中 running 的 XID 继续按 running 处理。
- 后续真实 commit 不改变这个 snapshot 的判断。

依赖：

- `GetSnapshotData()` 采集 `xip/subxip/suboverflowed`。
- isolation level 决定 snapshot 生命周期。
- snapshot manager 维护 active/registered refcount。

不能保证：

- snapshot 不是当前系统真实状态。
- `suboverflowed` 不是让 snapshot 失效。
- 它只是改变后续判断算法。

### 7.5 `pg_subtrans` parent map

保证：

- 给定一个 still-relevant SubXID，可以追溯 immediate parent。
- 进一步可追溯到 top-level 或足够老的 parent。

依赖：

- `SubTransSetParent()`。
- `SubTransGetParent()`。
- `SubTransGetTopmostTransaction()`。
- SLRU bank lock。
- `TransactionXmin` 边界。

不能保证：

- 不保证 crash 后完整历史。
- 不保证能列出 parent 的 children。
- 不回答 commit/abort。

### 7.6 `pg_xact` commit/abort 状态

保证：

- 判断 XID 是否 committed 或 aborted。
- abort 子事务能被标记，不再必须等顶层事务结束。

依赖：

- `TransactionIdDidCommit()`。
- `TransactionIdDidAbort()`。
- `TransactionIdAbortTree()`。
- `TransactionIdCommitTree()` 或 async commit tree。

不能保证：

- 它不知道子事务 parent。
- 对 overflow 的 parent 映射仍需 `pg_subtrans`。

### 7.7 heap visibility rules

保证：

- 对 `xmin/xmax` 组合给出 MVCC 可见性。
- 区分当前事务、自身 command id、snapshot running set、commit/abort 状态。

依赖：

- `HeapTupleSatisfiesMVCC()`。
- `TransactionIdIsCurrentTransactionId()`。
- `XidInMVCCSnapshot()`。
- hint bits。
- `TransactionIdDidCommit/Abort()`。

不能保证：

- 它不负责维护 ProcArray。
- 它不负责分配 SubXID。
- 它只是消费这些状态。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 fallback 一：PGPROC cache overflow

正常路径：

- 子事务 XID 放入 `PGPROC->subxids`。
- snapshot 复制它。
- visibility 通过 `subxip` 直接判断。

异常或退化：

- 子事务数量超过 64。
- `PGPROC` 设置 overflow flag。
- 后续 snapshot 设置 `suboverflowed`。

fallback：

- visibility 用 `SubTransGetTopmostTransaction()`。
- 把候选 XID 映射到 top-level XID。
- 搜索 `snapshot->xip`。

调用者看到：

- SQL 语义不变。
- 查询可能变慢。
- 普通系统视图不直接显示 `suboverflowed`。

### 8.2 fallback 二：`TransactionIdIsInProgress()` slow path

`TransactionIdIsInProgress()` 用于一些非 snapshot visibility 路径。例如 vacuum 相关判断和等待逻辑。

它的 fast path：

- `xid < RecentXmin`，直接 not running。
- 最近已知 not in progress cache 命中。
- 当前事务命中。
- `xid > latestCompletedXid`，认为 running。
- ProcArray main XID 命中。
- cached child XID 命中。
- 如果没有 overflowed cache，not running。

overflow slow path：

- 扫 ProcArray 时收集 overflowed backend 的 top XID。
- 释放 `ProcArrayLock`。
- 先查 `TransactionIdDidAbort(xid)`。
- 再 `SubTransGetTopmostTransaction(xid)`。
- 如果 top XID 在收集的 running top XID 中，则 running。

为什么先查 abort？

- 一个 already-failed subtransaction 的 parent 可能仍 running。
- 但该 child 自己已经不 running。
- 如果不先查 abort，可能错把 failed child 当成 running。

### 8.3 fallback 三：`pg_subtrans` 不能回看太老

`SubTransGetParent()` 断言不能询问早于 `TransactionXmin` 的东西。`SubTransGetTopmostTransaction()` 走 parent chain 时：

- 如果 parent 早于 `TransactionXmin`，停止。
- 返回上一个可用 XID。

这不是精确历史查询。这是 bounded-lifetime metadata。

正确性理由：

- 早于 `TransactionXmin` 的 XID 不需要被当作当前 running 子事务精确追溯。
- 可见性只要不把仍可能影响当前 snapshot 的事务误判即可。

### 8.4 fallback 四：recovery / hot standby 形态

hot standby snapshot 形态不同。`GetSnapshotData()` 在 recovery 中：

- 从 `KnownAssignedXids` 取得 XID。
- 因为不总能区分 top-level 和 subxact，很多 XID 存入 `subxip`。
- `xip` 可能为空。
- 如果 KnownAssigned overflow 或 `lastOverflowedXid` 边界命中，也会设置 suboverflowed。

`XidInMVCCSnapshot()` 在 recovery snapshot 中：

- overflow 时也先 `SubTransGetTopmostTransaction()`。
- 然后搜索 `subxip`。

本节不展开 recovery 的完整 KnownAssignedXids。只记住：

- overflow 的模式在 primary 和 standby 都存在。
- standby 还需要从 WAL assignment record 重建 running XID 信息。
- `XLOG_XACT_ASSIGNMENT` 的记录频率也与 `PGPROC_MAX_CACHED_SUBXIDS` 有关。

### 8.5 ERROR cleanup 不是只清内存

子事务 abort 要清：

- LWLock。
- buffer lock。
- WAL insertion state。
- wait state。
- condition variable sleep。
- lock manager wait。
- timeout。
- ResourceOwner 管的资源。
- local cache invalidation。
- snapshot state。
- child XID arrays。
- PGPROC child cache。

不要把 cleanup 简化成“MemoryContext reset”。MemoryContext 只管内存。

ResourceOwner 管外部资源。ProcArray state 管其他 backend 如何看你。

`pg_subtrans` 管 parent map fallback。`pg_xact` 管 commit/abort truth。

## 9. 成本、资源与跨模块传播

### 9.1 成本变量一：子事务数量

关键阈值：

- 64 个 cached SubXID。

成本变化：

- 0 到 64：主要是 PGPROC cache 写入和 snapshot copy。
- 超过 64：设置 overflow flag。
- 之后新增 SubXID 不再增加 PGPROC array copy 成本。
- 但会增加 `pg_subtrans` parent map 写入和后续 fallback 可能性。

重要反直觉：

- 制造第 65 个 SubXID 的语句不一定慢很多。
- 慢可能出现在另一个 backend 之后扫描受影响 tuple 时。

### 9.2 成本变量二：snapshot 中 backend 数

`GetSnapshotData()` 扫 ProcArray。成本随：

- active backend 数。
- assigned XID backend 数。
- 每个 backend cached SubXID 数。
- 是否存在任意 overflow。

非 overflow：

- snapshot 需要复制每个 relevant backend 的 cached SubXID。
- 总复制量最多约 `active_xid_backends * 64`。

overflow：

- 一旦 `suboverflowed` 被置 true，后续复制 SubXID 没意义。
- snapshot 本身可能少复制一些 `subxip`。
- 但这个节省会在后续 tuple visibility 中以 `pg_subtrans` lookup 的形式还回来。

### 9.3 成本变量三：tuple 数

`HeapTupleSatisfiesMVCC()` 是按 tuple 执行。overflow 影响是否放大，取决于：

- 扫描 tuple 数。
- 这些 tuple 的 `xmin/xmax` 是否落在 `[snapshot->xmin, snapshot->xmax)`。
- 这些 XID 是否有 hint bit。
- 是否命中 range check。
- 是否需要查 `XidInMVCCSnapshot()`。
- 是否真的触发 `SubTransGetTopmostTransaction()`。

大表顺序扫描时：

- 如果大量 tuple 由 overflowed transaction 的 SubXID 修改。
- 并且 snapshot 处于能看到这些 XID 范围的位置。
- 则 per-tuple fallback 可能成为 CPU 和 SLRU 热点。

索引扫描时：

- tuple 数少，影响可能小。
- 但如果返回行都集中在 overflowed SubXID 修改的版本上，单行成本仍可能高。

### 9.4 成本变量四：parent chain 深度

`SubTransGetTopmostTransaction()` 不是哈希查找。它沿 child -> parent 链。

成本随：

- nesting depth。
- SLRU page locality。
- `subtransaction_buffers` 命中率。
- 是否跨多个 SLRU page。

常见 savepoint pattern 可能是宽而不深：

- 同一个顶层事务下很多 sibling subtransaction。
- chain depth 不大。

递归 exception pattern 可能更深：

- 每层 function 都建立 exception block。
- child parent chain 更长。

诊断时要区分：

- SubXID 总数导致 overflow。
- nesting depth 导致 topmost 追溯更贵。

### 9.5 成本变量五：SLRU buffer 与锁

`pg_subtrans` 使用 SLRU。相关成本：

- `SimpleLruReadPage_ReadOnly()`。
- bank lock。
- page miss 时读文件。
- dirty page evict 时写文件。
- `CheckPointSUBTRANS()` 可提前刷脏页。

如果 `subtransaction_buffers` 太小：

- `SubTransGetParent()` 命中率下降。
- overflow visibility 可能更多访问 `pg_subtrans` 文件。
- 并发 backend 可能在 SLRU bank lock 上排队。

但调大它不能避免 overflow。它只能降低 overflow fallback 的 SLRU 代价。

### 9.6 成本变量六：hint bit 状态

hint bit 可以让 visibility 少走一些事务状态判断。但对 MVCC snapshot：

- 如果 snapshot 认为某 XID still running，即使真实世界已经 committed，也不能按 committed 处理。
- `HeapTupleSatisfiesMVCC()` 明确避免为 snapshot 中 running 的事务读取最新 ProcArray 状态来设置 hint bit。

因此：

- overflow snapshot 下，一些 tuple 仍需要 `XidInMVCCSnapshot()`。
- hint bit 能减少 `TransactionIdDidCommit()` 或 `TransactionIdDidAbort()` 后续成本。
- 但不能消除 snapshot running-set 判断。

### 9.7 资源传播路径

SubXID overflow 的资源传播可以这样画：

```text
PL/pgSQL exception / SAVEPOINT pattern
  -> many assigned SubXIDs
  -> PGPROC 64-slot cache overflow
  -> snapshot->suboverflowed
  -> XidInMVCCSnapshot() uses pg_subtrans
  -> SLRU lookup / parent-chain walk
  -> per-tuple visibility CPU cost
  -> possible SLRU lock / IO pressure
  -> vacuum / pruning horizon decisions may also pay slow path
```

跨模块边界：

- PL/pgSQL 或 SQL pattern 制造 subtransaction。
- `xact.c` 管 transaction stack 和 subcommit/subabort。
- `varsup.c` 分配 XID 并发布到 PGPROC。
- `procarray.c` 采集 snapshot 与运行事务状态。
- `snapmgr.c` 将 snapshot overflow 转成 parent lookup。
- `subtrans.c` 提供 parent map。
- `heapam_visibility.c` 在 tuple hot path 消费判断。
- `pg_xact` 提供 commit/abort truth。
- checkpointer 间接参与 `pg_subtrans` dirty page 刷写与 truncate 周期。

### 9.8 后台进程参与状态推进

本节涉及 shared state，所以要说明后台进程。直接参与：

- 普通 backend 分配 XID、写 `pg_subtrans`、更新 PGPROC。
- 普通 backend 获取 snapshot、执行 visibility check。

间接参与：

- checkpointer 调 `CheckPointSUBTRANS()`，并在 checkpoint 周期中可能触发 `TruncateSUBTRANS()`。
- startup process 在恢复启动阶段调用 `StartupSUBTRANS()`。
- walwriter 可能刷 abort/assignment WAL，但 `pg_subtrans` 本身不依赖 WAL 保留历史。
- autovacuum backend 使用 visibility horizon 和 in-progress 判断，可能受 overflow slow path 影响。
- standby recovery 相关进程维护 KnownAssignedXids，并受 xact assignment WAL 影响。

不要误解：

- checkpointer 不帮你“解决 overflow”。
- autovacuum 不清理当前 running transaction 的 SubXID cache。
- backend 退出顶层事务才清 PGPROC overflow 状态。

## 10. 观测与诊断入口

### 10.1 能直接观测什么

SQL 层能直接看到的很少。可直接观测：

- 当前事务状态：`pg_stat_activity.state`。
- 等待事件：可能看到 SLRU、LWLock、IO 等等待。
- 大量子事务 workload 的 SQL pattern。
- 表膨胀、vacuum 滞后、长事务 xmin 等外围现象。
- `pg_stat_slru` 中 `Subtrans` 相关统计，取决于版本和视图字段。

可通过日志看到：

- 长事务。
- autovacuum 日志中的 removable horizon 受限。
- wraparound warning。
- lock wait。

但这些都不是 `suboverflowed` 的直接计数器。

### 10.2 只能推断什么

通常只能推断：

- 某个 backend 的 PGPROC SubXID cache 是否 overflow。
- 某个 snapshot 的 `suboverflowed` 是否为 true。
- 某次 heap scan 是否因为 `SubTransGetTopmostTransaction()` 变慢。
- `pg_subtrans` page lookup 是否由 visibility fallback 驱动。

推断来源：

- workload 中大量 savepoint 或 PL/pgSQL exception。
- profile 中 `SubTransGetTopmostTransaction`、`SubTransGetParent`、`XidInMVCCSnapshot` 占比。
- `pg_stat_slru` 中 Subtrans hit/read/write 异常。
- gdb 断点观察 `snapshot->suboverflowed`。
- 临时源码计数器。

### 10.3 几乎不可见什么

几乎不可通过普通 SQL 直接看到：

- 某个 backend 当前 `PGPROC->subxidStatus.count`。
- 某个 backend 当前 `PGPROC->subxidStatus.overflowed`。
- 某个 snapshot 的 `suboverflowed`。
- 某个 tuple 的 `xmin` 是否是 overflow 后未缓存的 SubXID。
- 某次 `XidInMVCCSnapshot()` 具体走了几层 parent chain。

需要：

- gdb。
- perf。
- DTrace/SystemTap/eBPF。
- 临时源码 instrumentation。
- isolation test。

### 10.4 一个具体 runtime truth

本节锚定的 runtime truth：

```text
当一个 running transaction 产生超过 64 个 assigned SubXID 后，其他 backend 获取的 MVCC snapshot 会把 `suboverflowed` 标记为 true；之后对落在 snapshot XID 范围内的 tuple XID 做可见性判断时，可能调用 `SubTransGetTopmostTransaction()`。
```

验证路径：

- SQL 制造超过 64 个 assigned SubXID。
- 在另一个 session 获取 snapshot。
- 用断点或临时日志确认 `GetSnapshotData()` 设置 `suboverflowed`。
- 扫描由这些 SubXID 修改的 tuple。
- 用断点或 perf 确认 `XidInMVCCSnapshot()` 进入 overflow 分支。

### 10.5 perf 诊断建议

如果怀疑 SubXID overflow 可见性成本：

- 先确认 workload 是否大量使用 exception/savepoint。
- 用 `perf top` 或 flamegraph 看函数占比。
- 重点看：
  - `XidInMVCCSnapshot`
  - `SubTransGetTopmostTransaction`
  - `SubTransGetParent`
  - `SimpleLruReadPage_ReadOnly`
  - `HeapTupleSatisfiesMVCC`
  - `HeapTupleSatisfiesVacuumHorizon`
- 同时看 `pg_stat_slru` 的 Subtrans 读写和命中。

判断边界：

- 如果大头在 `HeapTupleSatisfiesMVCC()`，不等于一定是 SubXID overflow。
- 可能是 hint bit 未设置。
- 可能是 snapshot 范围很宽。
- 可能是 tuple 很多。
- 需要看到 `SubTrans*` 函数或断点证据。

### 10.6 gdb 断点入口

适合课堂源码跟读的断点：

- `GetNewTransactionId`
- `SubTransSetParent`
- `GetSnapshotData`
- `XidInMVCCSnapshot`
- `SubTransGetTopmostTransaction`
- `XidCacheRemoveRunningXids`
- `ProcArrayEndTransactionInternal`

建议观察变量：

- `isSubXact`
- `MyProc->subxidStatus.count`
- `MyProc->subxidStatus.overflowed`
- `ProcGlobal->subxidStates[MyProc->pgxactoff]`
- `snapshot->suboverflowed`
- `snapshot->subxcnt`
- `snapshot->xcnt`
- 传入 `XidInMVCCSnapshot()` 的 `xid`
- `SubTransGetTopmostTransaction()` 返回值。

### 10.7 统计粒度边界

`pg_stat_activity`：

- backend 当前活动和等待。
- 不显示 SubXID cache。

`pg_stat_slru`：

- SLRU 层累计统计。
- 可能显示 Subtrans 读、写、hit、flush 等。
- 不能证明每次读来自 visibility fallback。

`EXPLAIN (ANALYZE, BUFFERS)`：

- 能看到 query 时间和 buffer 行为。
- 不能直接显示 `XidInMVCCSnapshot()` 调用了 `pg_subtrans`。

`perf`：

- 能看到 CPU 函数占比。
- 需要结合 SQL pattern 解释。

gdb：

- 能直接看状态。
- 会改变时序，不适合量化性能。

临时源码计数器：

- 最清晰。
- 但只适合实验，不应留在产品代码。

## 11. 常见误区

### 误区一：把 64 当成子事务数量上限

不是。64 是 `PGPROC` cached SubXID 上限。

超过 64 后：

- 子事务仍可继续产生。
- XID 仍会分配。
- parent link 仍写入 `pg_subtrans`。
- 事务仍可提交或回滚。
- 只是 fast path 信息不完整。

### 误区二：认为 overflow 会让 snapshot 不正确

不会。overflow 让 snapshot 不能完整枚举 SubXID。

它通过 `suboverflowed` 把这个事实带到可见性判断。后续用 `pg_subtrans` 补足映射。

正确性不变，成本变高。

### 误区三：把 `pg_subtrans` 当成 commit log

错误。`pg_subtrans` 保存 child -> parent。

commit/abort 状态在 `pg_xact`。visibility 常常需要两者配合：

- 先问 snapshot running set。
- overflow 时用 subtrans 找 top。
- 需要真实完成状态时问 pg_xact。

### 误区四：认为 `subxidStatus.count` 是总子事务数

不是。它只是 cache 中的有效槽位数。

overflow 后总子事务数可能远大于 count。如果 `overflowed == true`，`count` 只能说明“前面有一些还缓存着”。

### 误区五：认为只有写入方付成本

不对。制造 overflow 的 backend 付出：

- XID 分配。
- `pg_subtrans` 写入。
- transaction stack 管理。

其他 backend 可能付出：

- snapshot `suboverflowed`。
- tuple visibility fallback。
- SLRU lookup。
- vacuum in-progress slow path。

SubXID overflow 是跨 backend 的成本传播。

### 误区六：只看 `pg_stat_activity` 诊断

`pg_stat_activity` 不能告诉你：

- 是否 overflow。
- 哪个 snapshot overflow。
- 可见性是否查了 `pg_subtrans`。

它只能提供外围线索。真正定位需要：

- workload pattern。
- `pg_stat_slru`。
- perf。
- gdb。
- 或临时 instrumentation。

## 12. 课堂实验

实验一：复现 overflow 并跟到 `XidInMVCCSnapshot()`。一个会话在事务中制造超过 64 个写入型子事务并保持打开；另一个会话建立 snapshot 后扫描相关 tuple。

断点放在 `GetNewTransactionId`、`GetSnapshotData`、`XidInMVCCSnapshot`、`SubTransGetTopmostTransaction`。观察 `MyProc->subxidStatus.count` 到 64 后 overflow，snapshot `suboverflowed` 为 true，并在 tuple visibility 中回查 parent chain。

如果没有进入 fallback，检查 tuple XID 是否落在 `[xmin, xmax)`，以及 hint bit 是否已经绕过了事务状态查询。实验二：对比 profile。

case A 制造 10 个写入型子事务，case B 制造 100 个，并让另一个会话全表扫描。用 `perf record -g` 关注 `HeapTupleSatisfiesMVCC`、`XidInMVCCSnapshot`、`SubTransGetTopmostTransaction`、`SimpleLruReadPage_ReadOnly`。

预期 case B 更可能出现 `SubTrans*` 成本，但比例受表大小、range check、SLRU 命中和 CPU cache 影响，不能当固定常数。实验三：临时计数器。

只在本地实验分支给 `XidInMVCCSnapshot()` 的 overflow 分支和 `GetSnapshotData()` 设置 `suboverflowed` 的位置加 DEBUG 计数。用 10 个和 100 个子事务对比计数，实验后删除补丁。

SQL 看不到 snapshot 内部 bool，也看不到每个 tuple 是否走 subtrans；这个实验训练的是“现象 -> 断点确认状态 -> 回到源码解释”。

## 13. 讨论题

1. 为什么 PostgreSQL 不把所有 SubXID 都放进 `PGPROC`，而是固定缓存 64 个并允许 overflow？
2. `snapshot->suboverflowed == true` 时，为什么不能因为某个 XID 不在 `snapshot->subxip` 中就判断它不 running？
3. `pg_subtrans` 为什么只保存 child -> parent，而不保存 parent -> children？
4. `SubTransGetTopmostTransaction()` 为什么允许在 `TransactionXmin` 边界返回中间 parent？这个“近似”依赖什么正确性前提？
5. 子事务 abort 时，为什么 `XidCacheRemoveRunningXids()` 在 overflow 后找不到某个 SubXID 不能作为硬错误？
6. 如果一个 workload 使用大量 PL/pgSQL exception block，为什么慢查询可能出现在另一个只读 session，而不是 exception block 所在 session？
7. `subtransaction_buffers` 能缓解什么成本？它不能改变什么语义或阈值？
8. 你会用哪些证据区分“heap visibility 普遍贵”和“SubXID overflow fallback 贵”？

## 14. 本节小结

本节唯一主问题是：

- `PGPROC` 只能缓存 64 个 SubXID 时，PostgreSQL 如何保持 MVCC visibility 正确，并把成本传播到哪里。

核心链路：

- 子事务需要 XID 时进入 `AssignTransactionId()`。
- `GetNewTransactionId()` 分配 XID 并更新 `PGPROC` SubXID cache。
- 前 64 个 SubXID 进入 `PGPROC->subxids`。
- 第 65 个之后设置 `subxidStatus.overflowed`。
- `SubTransSetParent()` 写入 child -> parent。
- `GetSnapshotData()` 看到任意 overflow 后设置 `snapshot->suboverflowed`。
- `HeapTupleSatisfiesMVCC()` 调 `XidInMVCCSnapshot()`。
- overflow snapshot 下，`XidInMVCCSnapshot()` 用 `SubTransGetTopmostTransaction()` 映射到 top XID。
- 然后用 top XID 搜索 `snapshot->xip`。

核心状态和边界：

- `PGPROC->subxids` 是 shared-memory fast path，不是完整历史。
- `SnapshotData->suboverflowed` 是信息不完整标志，不是错误。
- `pg_subtrans` 是 parent map，不是 commit log。
- `pg_xact` 才回答 commit/abort。
- tuple header 只保存 XID，不保存 parent。

ownership / cleanup：

- transaction stack 持有本地 `TransactionState` 和 `childXids`。
- `TopTransactionContext` 管本地 XID arrays 的内存。
- `ResourceOwner` 管子事务资源。
- `PGPROC` 暴露 running top XID 和 cached SubXID。
- subabort 从 PGPROC cache 移除 failed child。
- 顶层事务结束清空 advertised XID、count 和 overflow。
- `pg_subtrans` 通过 SLRU page lifecycle、startup 和 checkpoint/truncate 回收。

错误路径和 fallback：

- SubXID cache overflow 后不报错。
- snapshot 记录 overflow。
- visibility 退回 `pg_subtrans`。
- `TransactionIdIsInProgress()` 也有 overflow slow path。
- abort 子事务先查 `pg_xact`，避免把 failed child 因 parent running 而误判为 running。

成本模型：

- common case 成本是数组复制和数组搜索。
- overflow 后成本可能变成 per-tuple parent-chain lookup。
- 成本随 tuple 数、active backend 数、SubXID 数、nesting depth、SLRU 命中率和 hint bit 状态变化。
- 制造 overflow 的 session 不一定是最慢的 session。
- 成本可能传播到读查询、vacuum、pruning、hot standby replay 相关路径。

观测边界：

- 普通 SQL 很难直接看到 `suboverflowed`。
- `pg_stat_activity` 只能提供外围状态。
- `pg_stat_slru` 能提供 Subtrans 层累计线索。
- perf 能看到函数级成本。
- gdb 或临时源码计数器才能直接确认状态变化。

可迁移系统规律：

- 一个 bounded fast path 通常需要一个 correctness fallback。
- fast path 的容量上限不是业务语义上限。
- fallback 的成本常常不是由触发者单独承担，而是在共享状态被其他 hot path 消费时扩散。
- 诊断这种问题时，必须同时问“信息在哪里变得不完整”和“哪个后续路径为补全信息付费”。

仍需标注的推断边界：

- 性能影响是 workload-dependent。
- parent chain 成本取决于 nesting depth。
- SLRU 成本取决于 buffer 配置、命中率和 IO。
- profile 结果取决于硬件、版本、编译选项和表数据形态。
- 本课基于 commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`，其他版本的函数拆分和观测视图可能不同，但核心矛盾稳定。
