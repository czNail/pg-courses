# PostgreSQL logical decoding snapshot build 与一致性点

## 课程定位

前置知识：已经理解 replication slot 会保留 WAL 和 `catalog_xmin`，知道 logical decoding 不是 redo，而是把 WAL 中的物理变更重组为事务级 logical change stream。 本节唯一主问题：
```text
创建 logical slot 时为什么不能立刻解码所有变更，
snapbuild 如何从 WAL 中收集 running transactions、catalog xmin 和一致性点？
```

核心矛盾：logical decoding 希望从某个 LSN 开始连续输出事务，但 WAL 里已经存在正在运行的事务、可能缺失事务开头的 heap record，也可能需要历史 catalog 版本解释 tuple。 如果创建 slot 后立刻输出“当前 WAL 之后的所有 record”，会破坏两个不变量。 第一个不变量是事务完整性：
```text
不能输出只看到了后半段的事务。
```

第二个不变量是 catalog 可见性：
```text
解码某条 WAL record 时，catalog lookup 必须使用该 record 产生时可见的 catalog 状态。
```

PostgreSQL 的选择不是在创建 slot 时阻塞所有写事务，也不是把 primary 当前 ProcArray 复制进 slot 文件。 它让 `SnapBuild` 顺序读取 WAL 中的 `XLOG_RUNNING_XACTS` 记录。 这些记录携带 `xl_running_xacts`，描述某个 WAL 时刻的 `nextXid`、`oldestRunningXid` 和 running XID 集合。 `SnapBuildProcessRunningXacts()` 依次推动状态：
```text
SNAPBUILD_START
  -> SNAPBUILD_BUILDING_SNAPSHOT
  -> SNAPBUILD_FULL_SNAPSHOT
  -> SNAPBUILD_CONSISTENT
```

到 `SNAPBUILD_FULL_SNAPSHOT` 后，后续新事务已经有足够 catalog snapshot 信息，可以开始收集部分 change。 但只有到 `SNAPBUILD_CONSISTENT`，所有旧事务边界都已经过去，slot 才有一个外部可见的 consistent point。 学完后应能判断：
```text
为什么 FULL_SNAPSHOT 不是可以对外输出的点；
为什么不能直接使用 xl_running_xacts->xids 快速构造完整 snapshot；
为什么 catalog_xmin 要先保护再逐步发布；
为什么 export snapshot 需要 full snapshot 且只能在 consistent 后；
为什么 persistent slot 要等 startpoint 找到后才持久化；
为什么重启后可以从 serialized snapbuild state 恢复，但 slot creation 不能随便使用旧 snapshot。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

前面几节已经建立了三个事实。 第一，logical slot 不是普通进度游标。 它同时保护：
```text
restart_lsn  -> 旧 WAL 不能太早删除
catalog_xmin -> 旧 catalog tuple 不能太早 vacuum 掉
confirmed_flush -> 客户端已经确认消费到哪里
```

第二，reorder buffer 解决的是 WAL record 顺序和事务提交顺序之间的错位。 它可以把 interleaved WAL 变成提交顺序输出。 第三，logical decoding 输出用户表变更时，经常需要读取系统 catalog。 比如 tuple 的 relation、列定义、toast、rewrite、类型信息、publication 过滤和 output plugin 回调都可能触发 catalog lookup。 本节把这三个事实连接起来：
```text
slot 创建
  -> 先保护 catalog horizon 和 WAL
  -> 顺序读 WAL 找到可解码起点
  -> 构造 historic catalog snapshot
  -> 得到 consistent point
  -> 再把 slot 变成持久对象
```

这节不讲 output plugin 协议。 也不讲 reorder buffer spill 的完整策略。 它们只在主链路中作为边界出现。 本节真正要解释的是：
```text
为什么 slot 创建返回的 LSN 不是“当前 WAL 末尾”，
而是 snapbuild 证明从这里以后不会丢旧事务前半段、也能解释 catalog 的 consistent point。
```

## 2. 核心矛盾与一个容易误判的 runtime 现象

考虑这个场景。 会话 A：
```sql
BEGIN;
CREATE TABLE sb_demo(id int primary key, note text);
INSERT INTO sb_demo VALUES (1, 'before slot');
-- 暂不提交
```

会话 B 创建 logical slot：
```sql
SELECT * FROM pg_create_logical_replication_slot('s1', 'test_decoding');
```

会话 A 再提交：
```sql
COMMIT;
```

直觉上可能会认为：
```text
slot 是在事务 A 提交前创建的；
所以事务 A 的 commit 在 slot 创建之后；
logical decoding 应该输出它。
```

真实边界更细。 事务 A 的某些 WAL record 可能已经在 slot 创建前写入。 比如 heap insert record 已经存在，commit record 还没出现。 如果 decoder 从创建时刻立刻输出 commit，就可能只看见事务后半段。 它没有完整 change set。 它也可能缺少事务开始后发生的 catalog 变更对应的 historic snapshot。 所以 `DecodingContextFindStartpoint()` 必须从 slot 的 `restart_lsn` 开始向前读。 它读到 `SnapBuild` 进入 `SNAPBUILD_CONSISTENT` 才停。 这个停下来的位置写入：
```text
slot->data.confirmed_flush = ctx->reader->EndRecPtr
```

SQL 函数返回的 `lsn` 或 replication protocol 返回的 `consistent_point`，就是这个位置。 它表达的不是：
```text
slot 创建命令开始执行时的 WAL 位置。
```

而是：
```text
从这个 LSN 之后提交的事务，decoder 已经有足够历史状态完整解释。
```

这是本节的 runtime truth。

## 3. 核心文件分工与阅读顺序

阅读顺序按 runtime 推进，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/replication/walsender.c` | replication protocol `CREATE_REPLICATION_SLOT ... LOGICAL` 的 slot 创建、snapshot 选项、`ReplicationSlotPersist()` 时机。 |
| 2 | `src/backend/replication/slotfuncs.c` | SQL 函数 `pg_create_logical_replication_slot()` 的同类路径。 |
| 3 | `src/backend/replication/logical/logical.c` | `CreateInitDecodingContext()`、`StartupDecodingContext()`、`DecodingContextFindStartpoint()`、`LogicalIncreaseXminForSlot()`。 |
| 4 | `src/include/replication/snapbuild.h` | `SnapBuildState` 真实状态名和对外入口。 |
| 5 | `src/include/replication/snapbuild_internal.h` | `struct SnapBuild` 内部字段：`xmin`、`xmax`、`start_decoding_at`、`next_phase_at`、`committed`、`catchange`。 |
| 6 | `src/backend/replication/logical/snapbuild.c` | 状态机、historic snapshot、snapshot export、serialization / restore。 |
| 7 | `src/backend/replication/logical/decode.c` | `standby_decode()` 处理 `XLOG_RUNNING_XACTS`，heap/xact record 如何被状态 gate 掉。 |
| 8 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferSetBaseSnapshot()`、`ReorderBufferGetOldestXmin()`、`ReorderBufferSetRestartPoint()`。 |
| 9 | `src/backend/storage/ipc/standby.c` | `LogStandbySnapshot()` 和 `LogCurrentRunningXacts()` 如何写 `xl_running_xacts`。 |
| 10 | `src/include/storage/standbydefs.h` | WAL record 里的 `xl_running_xacts` 布局。 |
| 11 | `src/backend/storage/ipc/procarray.c` | `GetRunningTransactionData()`、`GetOldestSafeDecodingTransactionId()`、slot xmin horizon 的 ProcArray 边界。 |
| 12 | `src/backend/access/transam/xlogreader.c` | `XLogBeginRead()`、`XLogReadRecord()` 的读位置语义。 |

主阅读线：
```text
ReplicationSlotCreate()
  -> CreateInitDecodingContext()
     -> StartupDecodingContext()
        -> AllocateSnapshotBuilder()
  -> DecodingContextFindStartpoint()
     -> XLogBeginRead(slot->data.restart_lsn)
     -> XLogReadRecord()
     -> LogicalDecodingProcessRecord()
        -> standby_decode()
           -> SnapBuildProcessRunningXacts()
  -> slot->data.confirmed_flush = reader->EndRecPtr
  -> SnapBuildExportSnapshot() / SnapBuildInitialSnapshot()
  -> ReplicationSlotPersist()
```

这条线里每一步都改变一个长期状态。 不要把它读成“创建上下文后读几条 WAL”。 它实际是在证明一个新的解码起点是安全的。

## 4. 关键状态：SnapBuild 不是普通 MVCC snapshot

`SnapBuild` 是 decoding backend 的 backend-local 状态。 它不在 shared memory。 它由 `AllocateSnapshotBuilder()` 分配在 `"snapshot builder context"` 里。 它被 `LogicalDecodingContext` 持有。 `ReorderBuffer` 通过 snapshot refcount 接收它构造出的 historic snapshot。 最先要记住的是状态枚举。 当前源码里的真实名字是：
```text
SNAPBUILD_START = -1
SNAPBUILD_BUILDING_SNAPSHOT = 0
SNAPBUILD_FULL_SNAPSHOT = 1
SNAPBUILD_CONSISTENT = 2
```

不要写成抽象的 `BUILDING`、`FULL`、`READY`。 这些名字和 `contrib/pg_logicalinspect` 的状态描述也有对应关系。 `SNAPBUILD_START`：
```text
还没有足够 running-xacts 信息；
heap change 和 xact commit 都不能对外形成有意义的 decoding 边界。
```

`SNAPBUILD_BUILDING_SNAPSHOT`：
```text
已经看到一个可用的 running-xacts record；
正在等待当时已经运行的事务结束；
期间会记录后续 commit 对 catalog snapshot 的影响。
```

`SNAPBUILD_FULL_SNAPSHOT`：
```text
已经有足够信息解码从此以后开始的事务；
可以给后续事务设置 base snapshot；
但仍可能有旧事务在运行，不能对外宣布一致性点。
```

`SNAPBUILD_CONSISTENT`：
```text
所有影响起点安全性的旧事务都已经越过边界；
从 start_decoding_at 之后提交的事务可以交付；
slot 创建可以返回 consistent point。
```

`struct SnapBuild` 中几个字段要组合理解。 `state` 是状态机位置。 `xmin` 表示所有小于它的事务已经提交或 abort。 `xmax` 表示所有大于等于它的事务在当前 historic snapshot 语义里不是已知 committed。 这组 `xmin` / `xmax` 不是普通 `SnapshotData` 的 running-xid 模型。 snapbuild 的 historic snapshot 是反向组织的。 它把 `[xmin, xmax)` 中需要被当作 committed 的 catalog-changing transaction 存在 `committed.xip`。 没有出现在这个数组里的事务，在 historic snapshot 里会被当作 abort 或 in-progress。 这样做的原因是 catalog-changing transaction 通常少于普通用户数据事务。 复制一整份 running set 成本更高。 `start_decoding_at` 是另一个关键字段。 它表示：
```text
不要解码结束 LSN 小于这个位置的事务内容。
```

`SnapBuildXactNeedsSkip(builder, ptr)` 的判断就是：
```text
ptr < builder->start_decoding_at
```

这意味着 consistent point 不只是 `state == SNAPBUILD_CONSISTENT`。 还必须知道哪些事务由于起点边界需要跳过。 `next_phase_at` 表示下一阶段转换需要等待的 XID cutoff。 它在 `SNAPBUILD_START` 中通常还没有意义。 进入 `SNAPBUILD_BUILDING_SNAPSHOT` 时，它保存第一条 running-xacts record 的 `nextXid`。 进入 `SNAPBUILD_FULL_SNAPSHOT` 时，它变成下一条等待边界的 `running->nextXid`。 进入 `SNAPBUILD_CONSISTENT` 后，它回到 `InvalidTransactionId`。 `initial_xmin_horizon` 是从 slot 创建时的安全 horizon 来的。 如果某条 running-xacts record 的 `oldestRunningXid` 早于它，snapbuild 不能使用。 因为相关 catalog row 可能已经被 vacuum 或 prune。 这个字段把 slot 的 `catalog_xmin` 保护和 WAL 中的 historical snapshot 连接起来。 `building_full_snapshot` 表示这次是否需要可导出或可直接使用的完整普通 snapshot。 `CREATE_REPLICATION_SLOT ... SNAPSHOT 'export'` 和 SQL 创建接口通常需要 full snapshot。 只用于后续 decoding 时，catalog-only snapshot 就够了。 `in_slot_creation` 很容易被忽略。 slot 创建时它为 true。 这会阻止 `SnapBuildFindSnapshot()` 直接使用已存在的 serialized snapbuild state。 原因是创建新 slot 需要找到自己的 decoding startpoint。 如果随便恢复某个旧 snapshot，可能得到一个任意 restart LSN，而不是新 slot 能保证完整事务边界的位置。 `committed.includes_all_transactions` 也很关键。 在一致性建立前，如果需要 full snapshot，snapbuild 会记录所有提交事务，而不只是 catalog-changing 事务。 一旦进入正常 decoding，只记录 catalog-changing 事务。 此后就不能再把它导出成“普通数据表可用”的 initial slot snapshot。 因此 `SnapBuildInitialSnapshot()` 会检查：
```text
builder->state == SNAPBUILD_CONSISTENT
builder->committed.includes_all_transactions
```

raw field 不是语义。 `xmin`、`xmax`、`committed.xip`、`start_decoding_at`、`next_phase_at` 和 `state` 一起才定义当前 decoder 的安全边界。

## 5. running-xacts record 从哪里来

`SnapBuild` 不是直接扫描 live `PGPROC`。 它在 WAL 流里消费 `XLOG_RUNNING_XACTS`。 这条 record 由 `standby.c` 里的 `LogStandbySnapshot()` 产生。 名字来自 Hot Standby，但 logical decoding 也复用它。 主链路是：
```text
LogStandbySnapshot()
  -> GetRunningTransactionLocks()
  -> LogAccessExclusiveLocks()
  -> GetRunningTransactionData()
  -> LogCurrentRunningXacts()
     -> XLogInsert(RM_STANDBY_ID, XLOG_RUNNING_XACTS)
```

`GetRunningTransactionData()` 在 `procarray.c` 中。 它和普通 `GetSnapshotData()` 相似，但目标不同。 它会收集所有已经分配 XID 的 PGPROC。 它包括 VACUUM 过程中可能分配的 XID。 它也包括 prepared transaction 的 dummy PGPROC。 它返回 `RunningTransactionsData`。 这个结构是内存结构，字段包括：
```text
xcnt
subxcnt
subxid_status
nextXid
oldestRunningXid
oldestDatabaseRunningXid
latestCompletedXid
xids
```

WAL 里的结构是 `xl_running_xacts`，定义在 `src/include/storage/standbydefs.h`。 它是连续的 WAL payload：
```text
xcnt
subxcnt
subxid_overflow
nextXid
oldestRunningXid
latestCompletedXid
xids[]
```

注意 `RunningTransactionsData` 和 `xl_running_xacts` 不是同一个结构。 源码保留两个结构，是因为 WAL record 需要连续布局。 `LogCurrentRunningXacts()` 把内存结构转换成 WAL 结构。 它注册 header，再注册 `(xcnt + subxcnt)` 个 TransactionId 数组。 它用 `XLOG_MARK_UNIMPORTANT` 标记这类 record。 这表示它不应该为了耐久性触发额外 checkpoint 或归档压力。 但它仍然会调用 `XLogSetAsyncXactLSN(recptr)`。 目的是提醒 walwriter 不要太晚刷出这条 record。 这个 record 对 standby 可读和 logical decoding startpoint 都很重要。 `LogStandbySnapshot()` 里有一个逻辑解码特有的锁边界。 `GetRunningTransactionData()` 会拿 `ProcArrayLock` 和 `XidGenLock`。 普通 Hot Standby 场景可以在插入 WAL 前释放 `ProcArrayLock`。 因为 standby replay 时可以用 CLOG 重新检查 commit status。 logical decoding 启用时不能这样做。 源码注释说明，如果先释放锁，CLOG 对 historic snapshot 来说可能已经处在“未来”。 这会出现一个 transaction 被记录在 running-xacts 中，但它的 commit WAL 又排在 running-xacts 之前的矛盾。 因此 logical decoding 启用时，`LogStandbySnapshot()` 把 `ProcArrayLock` 保持到 `LogCurrentRunningXacts()` 插入 WAL 后再释放。 这个细节解释了一个核心问题：
```text
logical decoding 不能靠“读 WAL 时查当前 CLOG”补足历史事务状态。
```

它必须依赖 WAL 顺序中的 running-xacts record。

## 6. 创建 logical slot 的第一半：先保护历史

Replication protocol 路径在 `walsender.c` 的 `CreateReplicationSlot()`。 SQL 函数路径在 `slotfuncs.c` 的 `pg_create_logical_replication_slot()`。 两条路径最终都会调用 `CreateInitDecodingContext()`。 创建 permanent logical slot 时，不是一开始就创建 persistent slot。 源码先调用：
```text
ReplicationSlotCreate(name, true, RS_EPHEMERAL, ...)
```

temporary slot 则用 `RS_TEMPORARY`。 `RS_EPHEMERAL` 的意义是：
```text
初始化过程中出错时，这个 slot 可以被清理；
只有等 consistent point 找到后，才通过 ReplicationSlotPersist() 变成 RS_PERSISTENT。
```

这不是工程洁癖。 如果还没找到 consistent point 就持久化，crash 后会留下一个看似存在、但没有可靠 `confirmed_flush` 和 snapshot 边界的 logical slot。 `CreateInitDecodingContext()` 先做 basic check。 它要求当前已经 acquire 了 `MyReplicationSlot`。 它拒绝 physical slot。 它检查 slot 所属 database 必须是当前 database。 它拒绝在已经执行写操作的 transaction 中创建 logical slot。 然后把 output plugin 名称写入 slot。 如果调用者没有传入 `restart_lsn`，它会调用 `ReplicationSlotReserveWal()`。 这一步给 slot 一个 WAL 保留起点。 但这还不是 consistent point。 接下来是 `catalog_xmin` 的初始保护。 `CreateInitDecodingContext()` 同时拿：
```text
ReplicationSlotControlLock
ProcArrayLock
```

顺序是先 `ReplicationSlotControlLock`，再 `ProcArrayLock`。 然后调用：
```text
GetOldestSafeDecodingTransactionId(!need_full_snapshot)
```

得到 `xmin_horizon`。 它随后在 slot 上设置：
```text
slot->effective_catalog_xmin = xmin_horizon
slot->data.catalog_xmin = xmin_horizon
```

如果需要 full snapshot，还会设置：
```text
slot->effective_xmin = xmin_horizon
```

然后调用：
```text
ReplicationSlotsComputeRequiredXmin(true)
```

这一步把 slot 的保护要求发布到全局 xmin 计算里。 为什么要这么早做？ 因为接下来 `DecodingContextFindStartpoint()` 可能要读一段历史 WAL。 在这段时间里 VACUUM 不能把需要用来解释 historic catalog snapshot 的 catalog tuple 删掉。 所以 slot 必须先保护历史，再去找 startpoint。 这里有一个 crash-safety 细节。 初始 `catalog_xmin` 设置后，`CreateInitDecodingContext()` 会：
```text
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```

但 permanent slot 仍然是 ephemeral。 也就是说：
```text
slot state file 可以保存初始化中的 xmin；
slot 的 persistency 还没变成 RS_PERSISTENT。
```

这个组合允许初始化过程中用 slot 保护资源。 同时避免失败后留下正式 slot。

## 7. `StartupDecodingContext()` 只搭舞台，不证明 startpoint

`CreateInitDecodingContext()` 后半段调用：
```text
StartupDecodingContext(NIL,
                       restart_lsn,
                       xmin_horizon,
                       need_full_snapshot,
                       false,
                       true,
                       xl_routine,
                       prepare_write,
                       do_write,
                       update_progress)
```

最后的 `in_create` 参数为 true。 这会传给 `AllocateSnapshotBuilder()` 的 `in_slot_creation`。 `StartupDecodingContext()` 做几件事。 第一，创建 `"Logical decoding context"` memory context。 第二，如果不是 fast-forward，加载 output plugin。 第三，如果不在 transaction block 中，把当前 backend 标记为：
```text
PROC_IN_LOGICAL_DECODING
```

这样 ProcArray 计算 xmin horizon 时不需要把这个 backend 当作普通事务单独扫描。 因为它的 xmin 保护由 replication slot 表达。 第四，分配 `XLogReaderState`。 它调用：
```text
XLogReaderAllocate(wal_segment_size, NULL, xl_routine, ctx)
```

`xl_routine` 由调用方决定。 walsender 使用 `logical_read_xlog_page` 和 `WalSndSegmentOpen`。 SQL 函数路径使用 `read_local_xlog_page` 和 `wal_segment_open`。 第五，分配 reorder buffer：
```text
ctx->reorder = ReorderBufferAllocate()
```

第六，分配 snapshot builder：
```text
ctx->snapshot_builder =
  AllocateSnapshotBuilder(ctx->reorder,
                          xmin_horizon,
                          start_lsn,
                          need_full_snapshot,
                          in_create,
                          slot->data.two_phase_at)
```

此时 `SnapBuild` 的 state 仍是 `SNAPBUILD_START`。 `StartupDecodingContext()` 不会读 WAL。 它不会调用 `SnapBuildProcessRunningXacts()`。 它也不会把 slot 变 persistent。 它只是把 decoder 所需的状态对象、callback 和 ownership 关系连起来。 真正证明 startpoint 的工作在下一步。

## 8. `DecodingContextFindStartpoint()`：读 WAL 直到 consistent

`DecodingContextFindStartpoint()` 的逻辑很短，但语义很重。 它先定位 WAL reader：
```text
XLogBeginRead(ctx->reader, slot->data.restart_lsn)
```

`xlogreader.c` 中 `XLogBeginRead()` 会把：
```text
EndRecPtr = 起始位置
ReadRecPtr = InvalidXLogRecPtr
```

后续每次 `XLogReadRecord()` 成功，reader 会更新：
```text
ReadRecPtr = 当前 record 起点
EndRecPtr  = 当前 record 结束后一字节
```

然后 `DecodingContextFindStartpoint()` 进入循环：
```text
for (;;)
  record = XLogReadRecord(ctx->reader, &err)
  LogicalDecodingProcessRecord(ctx, ctx->reader)
  if (DecodingContextReady(ctx))
    break
```

`DecodingContextReady()` 只是检查：
```text
SnapBuildCurrentState(ctx->snapshot_builder) == SNAPBUILD_CONSISTENT
```

所以循环停止条件完全由 `SnapBuild` 状态机决定。 `LogicalDecodingProcessRecord()` 在 `decode.c` 中。 它会先检查当前 WAL record 是否有 top-level XID。 如果有，就调用：
```text
ReorderBufferAssignChild(ctx->reorder, topxid, xid, buf.origptr)
```

然后根据 rmgr 分发到对应 decode 函数。 对本节最重要的是 `standby_decode()`。 它处理 `RM_STANDBY_ID` 的 `XLOG_RUNNING_XACTS`：
```text
xl_running_xacts *running = XLogRecGetData(r)
SnapBuildProcessRunningXacts(builder, buf->origptr, running)
ReorderBufferAbortOld(ctx->reorder, running->oldestRunningXid)
```

这里的 `buf->origptr` 是 running-xacts record 的起点 LSN。 `running` 是 WAL record 中的 `xl_running_xacts` payload。 `ReorderBufferAbortOld()` 用 `oldestRunningXid` 清掉已经不可能再提交的旧事务状态。 如果 WAL 读取失败，`DecodingContextFindStartpoint()` 会报：
```text
could not find logical decoding starting point
```

如果读页面 callback 等待新 WAL，它可能阻塞。 这就是创建 logical slot 有时会等待 long transaction 的原因。 它不是在等待 output plugin。 它是在等待 WAL 中出现足以推进 `SnapBuild` 状态的 running-xacts 记录和事务结束边界。 循环结束后，函数设置：
```text
slot->data.confirmed_flush = ctx->reader->EndRecPtr
```

如果 slot 启用了 two-phase，还会设置：
```text
slot->data.two_phase_at = ctx->reader->EndRecPtr
```

注意此处没有调用 `ReplicationSlotPersist()`。 它只是在内存 slot data 中记录 consistent point。 持久化时机在外层。

## 9. `SnapBuildProcessRunningXacts()` 的主流程

`SnapBuildProcessRunningXacts()` 是本节核心函数。 它的入口参数是：
```text
builder
lsn
xl_running_xacts *running
```

函数先判断是否已经 consistent。 如果还没 consistent：
```text
if (!SnapBuildFindSnapshot(builder, lsn, running))
  return;
```

如果已经 consistent：
```text
SnapBuildSerialize(builder, lsn)
```

这个分支说明 running-xacts record 有两个用途。 一致性建立前，它用于推动状态机。 一致性建立后，它用于周期性序列化 snapbuild state，并推进 slot 的 xmin / restart_lsn 候选。 之后函数更新：
```text
builder->xmin = running->oldestRunningXid
```

但它不会用 running-xacts 直接提高 `builder->xmax`。 源码注释强调：
```text
consistent 之后只需要在 catalog-changing transaction commit 时提高 xmax。
```

因为 logical decoding 只通过 historic snapshot 读取 catalog。 普通用户表 tuple 的数据已经在 WAL 中。 接下来调用：
```text
SnapBuildPurgeOlderTxn(builder)
```

它从 `committed.xip` 和 `catchange.xip` 中删除小于 `builder->xmin` 的事务。 这些事务对后续 historic snapshot 不再需要。 然后计算要发布给 slot 的 xmin。 它先问 reorder buffer：
```text
xmin = ReorderBufferGetOldestXmin(builder->reorder)
```

如果当前没有任何 base snapshot，就使用：
```text
xmin = running->oldestRunningXid
```

然后调用：
```text
LogicalIncreaseXminForSlot(lsn, xmin)
```

这一步不是马上把 `catalog_xmin` 推进到共享全局。 它可能只是设置 candidate。 只有客户端确认消费到相应 LSN，`LogicalConfirmReceivedLocation()` 才能真正把候选写入 slot。 最后处理 `restart_lsn`。 如果还没到 `SNAPBUILD_CONSISTENT`，函数直接 return。 因为还不知道可重启的 serialized snapshot 位置。 到 consistent 后，它看 reorder buffer 中最老还在进行的事务：
```text
txn = ReorderBufferGetOldestTXN(builder->reorder)
```

如果存在并且 `txn->restart_decoding_lsn` 有效，就调用：
```text
LogicalIncreaseRestartDecodingForSlot(lsn, txn->restart_decoding_lsn)
```

如果没有 in-progress txn，并且 `last_serialized_snapshot` 有效，就用 serialized snapshot 的 LSN 推进 restart decoding。 这解释了 `restart_lsn` 的本质：
```text
它不是最新消费位置；
它是 crash/restart 后重新解码仍能重建未完成事务和 historic snapshot 的最早 WAL 位置。
```

## 10. `SnapBuildFindSnapshot()`：三条建快照路径

`SnapBuildFindSnapshot()` 是 `SnapBuildProcessRunningXacts()` 在未 consistent 时的 helper。 它在注释里明确列出三种路径。 第一种是最简单路径：
```text
running->oldestRunningXid == running->nextXid
```

这表示记录产生时没有 running transaction。 此时可以直接进入：
```text
SNAPBUILD_CONSISTENT
```

函数会设置：
```text
builder->start_decoding_at = lsn + 1
builder->xmin = running->nextXid
builder->xmax = running->nextXid
builder->next_phase_at = InvalidTransactionId
```

并打日志：
```text
logical decoding found consistent point at ...
There are no running transactions.
```

这个路径说明 consistent point 可以很快找到。 但它依赖一个强条件：
```text
没有任何 running transaction。
```

第二种路径是 restore serialized snapshot。 条件是：
```text
!builder->building_full_snapshot
!builder->in_slot_creation
SnapBuildRestore(builder, lsn)
```

这只适用于普通 decoding 重启或后续 decoding context。 不适用于新建 slot。 也不适用于需要 full snapshot 的创建场景。 原因有两个。 新建 slot 不能把旧 serialized snapshot 的位置当作自己的 startpoint。 full snapshot 需要能读取普通表的 snapshot，而 serialized snapbuild state 主要保存 catalog snapshot 语义。 第三种路径是增量构建。 这是 long transaction 场景最常见的路径。 它有三个阶段。 阶段一：`SNAPBUILD_START` 遇到有 running xacts 的 record。 函数执行：
```text
builder->state = SNAPBUILD_BUILDING_SNAPSHOT
builder->next_phase_at = running->nextXid
builder->xmin = running->nextXid
builder->xmax = running->nextXid
SnapBuildWaitSnapshot(running, running->nextXid)
```

这不是说 running-xacts record 里的 xids 都被直接放进 snapshot。 源码注释明确说这样做有问题。 因为记录生成和 commit WAL 插入之间存在 race。 被标记为 running 的事务，可能已经在 WAL 顺序上提交。 如果直接按 `xids[]` 构造 snapshot，就可能把已经提交的事务错误地当作 running。 所以 snapbuild 用 `nextXid` 作为 cutoff。 它等待所有小于这个 cutoff 的事务结束。 阶段二：`SNAPBUILD_BUILDING_SNAPSHOT` 等到：
```text
builder->next_phase_at <= running->oldestRunningXid
```

这表示第一阶段看到的那些旧事务都结束了。 此时切到：
```text
SNAPBUILD_FULL_SNAPSHOT
```

并把：
```text
builder->next_phase_at = running->nextXid
```

这里日志叫：
```text
logical decoding found initial consistent point at ...
Waiting for transactions ... older than ...
```

名字容易误导。 这是内部 initial consistent point，不是 slot 对外返回的 final consistent point。 真正对外的状态仍要等到 `SNAPBUILD_CONSISTENT`。 阶段三：`SNAPBUILD_FULL_SNAPSHOT` 等到：
```text
builder->next_phase_at <= running->oldestRunningXid
```

这表示 FULL 阶段刚开始时仍在运行的事务也都结束了。 函数切到：
```text
SNAPBUILD_CONSISTENT
```

并设置：
```text
builder->next_phase_at = InvalidTransactionId
```

然后打日志：
```text
logical decoding found consistent point at ...
There are no old transactions anymore.
```

这才是 `DecodingContextFindStartpoint()` 可以停止的点。

## 11. 为什么 `SNAPBUILD_FULL_SNAPSHOT` 仍不能对外输出

`SNAPBUILD_FULL_SNAPSHOT` 的名字很容易让人误解。 它不是“slot 已经 ready”。 它的精确定义是：
```text
从这个状态之后开始的事务，
decoder 有足够信息给它们建立 base snapshot。
```

但此时可能还有事务在 FULL 切换前就已经开始。 这些事务的 change 可能在 WAL 中跨过状态边界。 如果它们之后提交，decoder 可能没有完整 change set。 所以 `SnapBuildProcessChange()` 有一道过滤：
```text
if builder->state < SNAPBUILD_FULL_SNAPSHOT:
  return false

if builder->state < SNAPBUILD_CONSISTENT &&
   xid < builder->next_phase_at:
  return false
```

第一条避免在没有 snapshot 前收集数据变更。 第二条避免在 FULL 但未 consistent 时收集旧事务。 对可以收集的事务，`SnapBuildProcessChange()` 会检查 reorder buffer 是否已有 base snapshot。 如果没有，就构造或复用 `builder->snapshot`，然后：
```text
ReorderBufferSetBaseSnapshot(builder->reorder, xid, lsn, builder->snapshot)
```

这让事务后续在 commit 输出时有 historic catalog snapshot。 但 commit 输出仍要受 `start_decoding_at` 和 consistent 边界影响。 在 `DecodeCommit()` 中，`SnapBuildCommitTxn()` 先更新 snapshot builder 的事务知识。 随后 `DecodeTXNNeedSkip()` 检查：
```text
SnapBuildXactNeedsSkip(ctx->snapshot_builder, buf->origptr)
database filter
origin filter
fast_forward
```

如果当前 commit record 的起点 LSN 小于 `start_decoding_at`，就跳过。 reorder buffer 也有自己的 boundary。 `ReorderBufferCommit()` 找不到事务状态时，直接返回。 如果事务没有 `base_snapshot`，说明它没有需要 decode 的 database change。 它会 cleanup，而不会调用 output plugin。 因此 FULL 阶段的意义是“开始安全收集新事务”。 CONSISTENT 阶段的意义是“外部可以从这里开始消费”。 把这两个状态合并，会导致 partial transaction 泄露。

## 12. catalog snapshot 为什么是“反向”的

普通 MVCC snapshot 的核心是 running XID set。 `xmin` 之前都已完成。 `xmax` 之后都不可见。 `xip` 里的是生成 snapshot 时仍在运行的事务。 snapbuild 的 historic snapshot 不同。 它只用于读取 catalog。 它更关心：
```text
在 WAL record 产生的历史时刻，哪些 catalog-changing transaction 已经提交。
```

因此 `SnapBuildBuildSnapshot()` 构造 `SNAPSHOT_HISTORIC_MVCC`。 它把 `builder->committed.xip` 复制到 `snapshot->xip`。 这些 XID 在 historic snapshot 中被当作 committed。 注释明确说：
```text
SnapshotData 的 xip / subxip 在这里被重新解释。
```

这样做的成本模型更适合 logical decoding。 大多数事务只改用户表。 decoder 解码用户表 tuple 时，tuple image 已经在 WAL record 里。 它不需要用 MVCC 去扫描用户表旧版本。 它需要的是 catalog lookup。 只有修改 catalog 的事务会影响 lookup 结果。 所以 `committed.xip` 可以只保存 catalog-changing transaction。 在 slot creation 且需要 full snapshot 时例外。 这时要导出或使用一个普通 MVCC snapshot。 `SnapBuildInitialSnapshot()` 会把 snapbuild 的反向 snapshot 转成普通 snapshot。 它遍历：
```text
xid in [snap->xmin, snap->xmax)
```

如果 XID 不在 committed set 中，就把它放入普通 snapshot 的 `xip`，表示 in-progress。 这个过程可能很贵。 所以它只在创建 slot 且用户要求 `SNAPSHOT 'export'` 或 `SNAPSHOT 'use'` 等场景发生。 它还检查 `GetOldestSafeDecodingTransactionId(false)`。 如果当前安全 XID 已经超过 `snap->xmin`，就报错。 这防止导出一个已经不受 slot horizon 保护的 snapshot。

## 13. catalog xmin 如何发布到 slot

创建 slot 时，`CreateInitDecodingContext()` 先设置初始 `catalog_xmin`。 之后 `SnapBuildProcessRunningXacts()` 会持续推进。 推进不是直接写 `slot->data.catalog_xmin`。 它调用：
```text
LogicalIncreaseXminForSlot(current_lsn, xmin)
```

这个函数在 `logical.c` 中。 它比较新 xmin 和当前 `slot->data.catalog_xmin`。 如果新值不更靠后，就忽略。 如果 `current_lsn <= slot->data.confirmed_flush`，说明客户端已经确认到这个 WAL 位置。 它可以把 candidate 标记为可立即应用。 否则，如果当前没有 pending candidate，就设置：
```text
slot->candidate_catalog_xmin = xmin
slot->candidate_xmin_lsn = current_lsn
```

真正应用发生在：
```text
LogicalConfirmReceivedLocation(lsn)
```

当确认 LSN 覆盖 `candidate_xmin_lsn` 后，函数会：
```text
slot->data.catalog_xmin = slot->candidate_catalog_xmin
clear candidate
ReplicationSlotMarkDirty()
ReplicationSlotSave()
slot->effective_catalog_xmin = slot->data.catalog_xmin
ReplicationSlotsComputeRequiredXmin(false)
```

顺序很重要。 源码注释强调：
```text
先把新的 xmin 写入 slot state file；
再更新 effective_catalog_xmin；
否则 crash 后可能忘记 catalog tuple 已经允许被清理。
```

`catalog_xmin` 是持久语义。 `effective_catalog_xmin` 是当前进程内全局 xmin 计算的输入。 candidate 是 WAL 消费确认前的暂存语义。 三者不能混用。 这个设计避免了过早释放 catalog 版本。 也避免了每读到一个 running-xacts record 就 fsync slot 文件。 catalog horizon 的发布被客户端确认节奏限速。 这就是逻辑复制消费者慢时，publisher 上 catalog bloat 可能被 slot 保留的原因。

## 14. restart_lsn 和 serialized snapshot

`SnapBuildSerialize()` 只在 `SNAPBUILD_CONSISTENT` 后有意义。 它把当前 `SnapBuild` 状态写到：
```text
pg_logical/snapshots/%X-%X.snap
```

文件名来自 LSN。 内容是 `SnapBuildOnDisk` 加上两个可变数组：
```text
committed.xip
catchange.xip
```

写入过程是 crash-safe 的。 它先写临时文件：
```text
%X-%X.snap.<pid>.tmp
```

写完后 fsync 文件。 再 fsync 目录。 然后 rename 到正式文件名。 最后 fsync 正式文件和目录。 如果另一个 backend 已经写了同一个 LSN 的 snapshot，它不会覆盖。 它会重复 fsync 并记住这个位置。 这避免了多 decoder 并发时用额外锁协调。 `SnapBuildRestore()` 则从指定 LSN 的 snapshot 文件恢复。 它会校验 magic、version 和 checksum。 它只接受：
```text
ondisk.builder.state >= SNAPBUILD_CONSISTENT
ondisk.builder.xmin >= builder->initial_xmin_horizon
```

恢复成功后，会重新构造 `builder->snapshot`，并调用：
```text
ReorderBufferSetRestartPoint(builder->reorder, lsn)
```

slot 创建时不会走这个捷径。 因为 `builder->in_slot_creation` 为 true。 但普通 decoding 重启可以利用它。 这解释了两个行为。 第一，第一次创建 slot 可能等 long transaction。 第二，已经使用过的 slot 重启后不一定从零开始等。 它可以从保存过的 consistent snapbuild state 继续。 `restart_lsn` 的推进和 serialized snapshot 相关。 当 `SnapBuildProcessRunningXacts()` 已经 consistent 时，它会看 reorder buffer 的 oldest transaction。 如果有事务仍在进行，slot 的 restart decoding 位置必须足够早，能重新读取它的 change。 如果没有事务进行，就可以用 `last_serialized_snapshot`。 也就是说：
```text
restart_lsn 保留 WAL；
serialized snapshot 保留 catalog snapshot builder 状态；
二者一起决定 crash 后能否重新构造 decoder 现场。
```

## 15. export snapshot 的边界

Replication protocol 创建 logical slot 支持 snapshot action。 `walsender.c` 中默认是：
```text
CRS_EXPORT_SNAPSHOT
```

也可以选择：
```text
CRS_USE_SNAPSHOT
CRS_NOEXPORT_SNAPSHOT
```

export 和 use 都需要：
```text
need_full_snapshot = true
```

`SNAPSHOT 'export'` 不能在 transaction block 里调用。 因为 `SnapBuildExportSnapshot()` 自己会启动一个 transaction。 它调用：
```text
StartTransactionCommand()
XactIsoLevel = XACT_REPEATABLE_READ
XactReadOnly = true
SnapBuildInitialSnapshot()
ExportSnapshot()
```

它还保存并恢复 `CurrentResourceOwner`。 原因是导出 snapshot 需要一个仍然打开的 transaction。 导入方会检查源事务是否仍然存在，从而确认 xmin horizon 仍被保护。 `SnapBuildClearExportedSnapshot()` 在清理时会 abort 这个临时 transaction。 `SnapBuildResetExportedSnapshotState()` 用于 transaction abort 边界重置静态状态。 `SNAPSHOT 'use'` 则要求调用者已经在 transaction block 中。 它还要求：
```text
REPEATABLE READ
read-only
还没有执行过任何 query
不能在 subtransaction
```

成功后调用：
```text
snap = SnapBuildInitialSnapshot(ctx->snapshot_builder)
RestoreTransactionSnapshot(snap, MyProc)
```

两种模式都必须等 `DecodingContextFindStartpoint()` 完成。 因为 `SnapBuildInitialSnapshot()` 要求：
```text
builder->state == SNAPBUILD_CONSISTENT
builder->building_full_snapshot
builder->committed.includes_all_transactions
MyProc->xmin 还没有有效值
```

这就是 export snapshot 的边界。 它不是创建 slot 时顺手拿当前 snapshot。 它是 snapbuild 先从 WAL 证明了 consistent point，再把 historic 信息转换成普通 MVCC snapshot。

## 16. slot persist 的准确时机

`ReplicationSlotCreate()` 创建 persistent logical slot 时先使用 `RS_EPHEMERAL`。 这一步已经会创建 slot on disk。 但 persistency 不是 `RS_PERSISTENT`。 它还会把 slot 标记为 in use，并把当前 backend 设为 `active_proc`。 如果初始化过程中 ERROR，slot release / cleanup 路径可以丢弃 ephemeral slot。 在 replication protocol 路径中，顺序是：
```text
ReplicationSlotCreate(... RS_EPHEMERAL ...)
CreateInitDecodingContext()
DecodingContextFindStartpoint()
SnapBuildExportSnapshot() / SnapBuildInitialSnapshot() / noexport
FreeDecodingContext()
if !temporary:
  ReplicationSlotPersist()
return slot_name, consistent_point, snapshot_name, output_plugin
```

在 SQL 函数路径中，顺序是：
```text
create_logical_replication_slot()
  -> ReplicationSlotCreate(... RS_EPHEMERAL ...)
  -> CreateInitDecodingContext()
  -> DecodingContextFindStartpoint()
  -> FreeDecodingContext()
return slot_name, confirmed_flush
if !temporary:
  ReplicationSlotPersist()
ReplicationSlotRelease()
```

`ReplicationSlotPersist()` 做的事很小但关键：
```text
slot->data.persistency = RS_PERSISTENT
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```

它保证 crash 后 slot 仍存在。 所以 slot 持久化发生在：
```text
consistent point 已经找到之后。
```

不是在 `ReplicationSlotCreate()` 刚分配 slot 时。 也不是在 `CreateInitDecodingContext()` 刚保护 `catalog_xmin` 时。 这条边界能防止半初始化 slot 留在系统中。

## 17. 错误路径和异常路径

第一类异常是 long transaction。 如果 `SnapBuildFindSnapshot()` 在 `SNAPBUILD_START` 看到 running xacts，它会进入 `SNAPBUILD_BUILDING_SNAPSHOT`。 然后调用：
```text
SnapBuildWaitSnapshot(running, running->nextXid)
```

它遍历 `running->xids`。 对小于等于 cutoff 的 XID 调用：
```text
XactLockTableWait(xid, NULL, NULL, XLTW_None)
```

如果发现要等的是当前事务：
```text
TransactionIdIsCurrentTransactionId(xid)
```

它会 `elog(ERROR, "waiting for ourselves")`。 这避免无限等待。 等待结束后，如果不在 recovery 中，它会主动调用：
```text
LogStandbySnapshot()
```

目的是尽快写出新的 running-xacts record。 否则 decoder 可能要等 bgwriter 或 checkpoint 之后才看到下一条推进状态的 record。 第二类异常是安全 horizon 太旧。 如果：
```text
running->oldestRunningXid < builder->initial_xmin_horizon
```

snapbuild 会跳过这条 snapshot。 它打 DEBUG1 日志，说明 xmin horizon 太低。 然后仍然可能调用 `SnapBuildWaitSnapshot()` 等旧事务结束并促使新 record 出现。 这样做的原因是：
```text
旧 running-xacts 记录引用的 catalog 版本可能已经不受保护。
```

第三类异常是 WAL 读取失败。 `DecodingContextFindStartpoint()` 如果 `XLogReadRecord()` 返回 err，会报：
```text
could not find logical decoding starting point: ...
```

如果没有 record，也会报：
```text
could not find logical decoding starting point
```

这通常意味着所需 WAL 不可用、WAL 损坏、timeline/segment 边界异常，或读页面 callback 无法提供数据。 第四类异常是 snapshot file 损坏。 `SnapBuildRestoreSnapshot()` 校验：
```text
magic
version
checksum
```

不匹配会报 `DATA_CORRUPTED`。 如果 `missing_ok` 为 true 且文件不存在，它返回 false。 这让普通 decoding 可以退回到继续读 WAL 找状态。 但坏文件不能静默忽略。 第五类异常是 snapshot export 失败。 `SnapBuildInitialSnapshot()` 会拒绝：
```text
已有 active / registered snapshot
state 未到 CONSISTENT
includes_all_transactions 为 false
MyProc->xmin 已经有效
oldest safe xid 超过 snap->xmin
```

这些检查都在防止导出一个普通 MVCC 无法保证的数据 snapshot。 第六类异常是 serialization I/O 失败。 `SnapBuildSerialize()` 写临时文件、fsync、rename、再 fsync。 任何写、fsync、close、rename 失败都会 ERROR。 这通常不会破坏已有 snapshot。 因为正式文件通过 atomic rename 发布。 失败后后续 decoding 可以重试序列化。

## 18. 成本、资源与跨模块传播

创建 logical slot 的等待成本主要来自事务年龄，不是 WAL 量本身。 如果有长事务跨过 slot 创建时刻，snapbuild 必须等它结束或等到可证明旧事务不再影响 consistent point。 这会体现在：
```text
CREATE_REPLICATION_SLOT 或 pg_create_logical_replication_slot() 长时间不返回。
```

第二个成本是 ProcArray 与 XidGenLock。 `GetRunningTransactionData()` 要收集运行事务。 它拿 `ProcArrayLock` 和 `XidGenLock`。 logical decoding 启用时，`LogStandbySnapshot()` 会把 `ProcArrayLock` 保持到 running-xacts WAL 记录插入后。 这个路径不在每个事务 hot path 上频繁执行。 但在 backend 很多、事务很多、checkpoint 或主动 log standby snapshot 较频繁时，它仍然是全局状态扫描和锁持有。 第三个成本是 catalog horizon 保留。 slot 的 `catalog_xmin` 会阻止 vacuum 清理过新的 catalog tuple。 如果 logical consumer 很慢，candidate xmin 迟迟不能确认，`effective_catalog_xmin` 也不能推进。 这会传播到：
```text
pg_class / pg_attribute / pg_depend 等 catalog bloat
old xid horizon
autovacuum 清理效果
clog / multixact 等历史保留压力
```

第四个成本是 reorder buffer 中 snapshot pin。 `SnapBuildProcessChange()` 把 base snapshot 交给事务。 `ReorderBufferGetOldestXmin()` 会返回最老 base snapshot 的 xmin。 只要 reorder buffer 中还有事务持有旧 base snapshot，slot 的 catalog xmin 就不能越过它。 第五个成本是 snapbuild serialization。 `SnapBuildSerialize()` 同步写文件并 fsync。 源码里有 TODO 提到这里的同步 fsync 可能有可见开销。 它的收益是重启后不必从更早的 WAL 重新构建 snapshot。 所以这是典型的：
```text
运行期同步 I/O
  vs
重启后恢复成本和 WAL 保留窗口
```

第六个成本是 output plugin 无关成本。 创建 slot 找 startpoint 时即使不输出 change，也要读 WAL、处理 running-xacts、维护 reorder buffer 的最小状态。 所以把创建 slot 慢归因到插件回调，通常是错的。 插件 startup callback 会被调用，但 consistent point 的等待主要由 snapbuild 和 WAL 状态决定。 跨模块传播可以压缩为：
```text
ProcArray 提供 running transaction 事实
WAL 固化这些事实
SnapBuild 把事实转成 historic catalog snapshot
ReorderBuffer 持有事务与 base snapshot
ReplicationSlot 发布 restart_lsn 和 catalog_xmin
Vacuum / checkpoint / wal retention 被 slot horizon 反向约束
```

## 19. 观测与诊断入口

最直接的 SQL 入口是 `pg_replication_slots`。 创建成功后观察：
```sql
SELECT slot_name,
       slot_type,
       database,
       restart_lsn,
       confirmed_flush_lsn,
       catalog_xmin,
       active
FROM pg_replication_slots
WHERE slot_name = 's1';
```

能看到：
```text
restart_lsn
confirmed_flush_lsn
catalog_xmin
```

看不到：
```text
SnapBuild.state
next_phase_at
committed.xip
catchange.xip
candidate_catalog_xmin
```

`candidate_catalog_xmin` 是 shared memory slot 内部字段，不是普通 SQL 视图字段。 第二个入口是 server log。 提高 logical decoding 相关日志级别后，可能看到：
```text
logical decoding found initial starting point at ...
Waiting for transactions ... older than ...

logical decoding found initial consistent point at ...
Waiting for transactions ... older than ...

logical decoding found consistent point at ...
There are no old transactions anymore.
```

这些日志分别对应：
```text
START -> BUILDING_SNAPSHOT
BUILDING_SNAPSHOT -> FULL_SNAPSHOT
FULL_SNAPSHOT -> CONSISTENT
```

或者 no-running-xacts 的直接 consistent 路径。 第三个入口是 wait。 如果创建 slot 卡住，可以查：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query LIKE '%replication_slot%'
   OR backend_type = 'walsender';
```

等待旧事务结束时，底层使用 `XactLockTableWait()`。 wait event 可能表现为 transactionid lock 相关等待。 具体显示取决于路径和版本。 第四个入口是长事务排查。 可以查：
```sql
SELECT pid, backend_xid, backend_xmin, xact_start, state, query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL
ORDER BY xact_start;
```

这只能近似解释。 `SnapBuild` 等的是 WAL 中某条 running-xacts record 的 cutoff。 当前 live backend 状态可能已经和那条 record 的历史状态不同。 但长事务仍然是最常见原因。 第五个入口是 `pg_logicalinspect`。 当前源码 `snapbuild.h` 注释要求 `get_snapbuild_state_desc()` 跟随 `SnapBuildState` 更新。 安装 extension 后，可以检查 `pg_logical/snapshots` 中 serialized snapbuild state。 它适合验证重启后是否有可恢复 snapshot。 但它只能看已经序列化到磁盘的 state。 创建 slot 正在等待时的 backend-local `SnapBuild` 通常看不到。 第六个入口是断点。 适合源码实验的断点：
```text
CreateInitDecodingContext
DecodingContextFindStartpoint
standby_decode
SnapBuildProcessRunningXacts
SnapBuildFindSnapshot
SnapBuildWaitSnapshot
LogicalIncreaseXminForSlot
LogicalConfirmReceivedLocation
ReplicationSlotPersist
```

观察变量：
```text
builder->state
builder->next_phase_at
builder->start_decoding_at
builder->xmin
builder->xmax
running->nextXid
running->oldestRunningXid
slot->data.confirmed_flush
slot->data.catalog_xmin
slot->candidate_catalog_xmin
```

第七个入口是 WAL rmgr。 用 `pg_waldump` 可以看到 standby rmgr 的 running xacts record。 它能帮助确认 running-xacts record 是否稀疏，或者是否迟迟没有推进到新的 `oldestRunningXid`。 但 `pg_waldump` 不能直接告诉你 `SnapBuild` 是否已经 consistent。 那是 decoder 消费 WAL 后的本地状态。 诊断时要区分三类状态。 能直接看到：
```text
slot confirmed_flush_lsn
slot restart_lsn
slot catalog_xmin
server log 中的 consistent point
WAL 中的 running-xacts record
```

只能推断：
```text
slot 创建等待的具体 cutoff
哪个事务阻挡 state transition
candidate xmin 何时能发布
```

几乎不可见：
```text
backend-local committed.xip / catchange.xip
当前未序列化的 SnapBuild snapshot refcount
某个 in-progress ReorderBufferTXN 的 base_snapshot_lsn
```

不要把 `pg_replication_slots.confirmed_flush_lsn` 解释成 decoder 当前读 WAL 的位置。 创建过程中它还没设置。 正常消费过程中它表示客户端确认位置。 `restart_lsn` 才是 WAL 保留和重新解码的下界。

## 20. 常见误区

误区一：
```text
创建 slot 的时间点就是 logical stream 起点。
```

不是。 stream 起点是 `DecodingContextFindStartpoint()` 找到的 `confirmed_flush`。 它可能晚于创建命令开始时的 WAL 位置。 误区二：
```text
SNAPBUILD_FULL_SNAPSHOT 表示 slot 已经 ready。
```

不是。 FULL 表示可以开始收集后续新事务的 change。 对外 ready 是 `SNAPBUILD_CONSISTENT`。 误区三：
```text
running-xacts record 里的 xids 可以直接当作 snapshot xip。
```

不能。 生成 record 和 commit WAL 插入之间有 race。 snapbuild 使用 `nextXid` 和 `oldestRunningXid` 的阶段边界，而不是直接信任 xids 数组构造完整输出点。 误区四：
```text
catalog_xmin 一旦计算出来就可以立刻释放旧 catalog。
```

不能。 `LogicalIncreaseXminForSlot()` 可能只是设置 candidate。 必须等客户端确认覆盖对应 LSN，`LogicalConfirmReceivedLocation()` 持久化 slot 后，`effective_catalog_xmin` 才能推进。 误区五：
```text
serialized snapbuild snapshot 可以用于任何 slot 创建。
```

不能。 新建 slot 的 `in_slot_creation` 会禁用这条捷径。 否则会绕过为新 slot 找 startpoint 的证明过程。 误区六：
```text
logical decoding 可以用当前 catalog 解释历史 WAL。
```

不能。 DDL/DML 混合事务、catalog invalidation、`XLOG_HEAP2_NEW_CID` 和 historic snapshot 都是在防止这个错误。 误区七：
```text
slot persist 和 slot create 是同一个时刻。
```

不是。 persistent logical slot 创建时先是 `RS_EPHEMERAL`。 只有 consistent point 找到并处理 snapshot action 后，才调用 `ReplicationSlotPersist()`。

## 21. 课堂实验

实验一：观察 long transaction 阻塞 slot consistent point。 步骤：
```sql
-- session 1
BEGIN;
CREATE TABLE sb_wait(id int);
INSERT INTO sb_wait VALUES (1);
```

另一个会话创建 slot：
```sql
SELECT * FROM pg_create_logical_replication_slot('sb_wait_slot', 'test_decoding');
```

如果环境中 running-xacts record 和状态边界触发得合适，slot 创建可能等待。 在源码断点上观察：
```text
SnapBuildFindSnapshot
SnapBuildWaitSnapshot
builder->state
builder->next_phase_at
running->oldestRunningXid
running->nextXid
```

提交 session 1：
```sql
COMMIT;
```

再观察 slot 创建返回的 LSN。 将它和事务开始时间区分开。 实验二：验证 consistent point 与后续输出边界。 创建 slot 后执行：
```sql
CREATE TABLE sb_out(id int primary key, note text);
INSERT INTO sb_out VALUES (1, 'after consistent');
```

读取变更：
```sql
SELECT * FROM pg_logical_slot_get_changes('sb_wait_slot', NULL, NULL);
```

关注输出事务的 commit 是否都在 slot 返回 LSN 之后。 再查：
```sql
SELECT restart_lsn, confirmed_flush_lsn, catalog_xmin
FROM pg_replication_slots
WHERE slot_name = 'sb_wait_slot';
```

解释：
```text
confirmed_flush_lsn 是消费确认边界；
restart_lsn 是重新解码下界；
catalog_xmin 是 historic catalog 保护边界。
```

实验三：源码跟读 `catalog_xmin` 发布。 在以下函数加断点或 DEBUG 日志：
```text
LogicalIncreaseXminForSlot
LogicalConfirmReceivedLocation
ReplicationSlotSave
ReplicationSlotsComputeRequiredXmin
```

关注变量：
```text
current_lsn
xmin
slot->data.confirmed_flush
slot->candidate_xmin_lsn
slot->candidate_catalog_xmin
slot->data.catalog_xmin
slot->effective_catalog_xmin
```

目标不是记住某次运行的数字。 目标是画出这条状态链：
```text
running-xacts record
  -> oldest base snapshot xmin
  -> candidate catalog xmin
  -> client confirmed_flush 覆盖
  -> slot state file 持久化
  -> effective_catalog_xmin 发布
```

实验四：验证 slot persist 时机。 在 `ReplicationSlotCreate()`、`DecodingContextFindStartpoint()`、`ReplicationSlotPersist()` 断点。 创建非 temporary logical slot。 观察：
```text
slot->data.persistency
slot->data.restart_lsn
slot->data.confirmed_flush
slot->data.catalog_xmin
```

预期：
```text
刚创建 logical slot 时 persistency 是 RS_EPHEMERAL；
FindStartpoint 后 confirmed_flush 有效；
ReplicationSlotPersist 后 persistency 才变 RS_PERSISTENT。
```

实验五：观察 snapbuild serialized snapshot。 让 logical slot 消费一段 WAL 后，检查：
```text
pg_logical/snapshots/
```

如果安装了 `pg_logicalinspect`，查看 snapshot state。 然后重启实例，再启动 decoding。 在 `SnapBuildRestore()` 断点观察是否能从保存的 consistent snapshot 恢复。 注意：
```text
新建 slot 不应依赖这条 restore 捷径；
已有 slot 的 restart 才可能使用它。
```

## 22. 讨论题

1. 为什么 `SNAPBUILD_FULL_SNAPSHOT` 后可以开始收集部分 change，却仍然不能把 slot 创建返回给客户端？

2. `SnapBuildFindSnapshot()` 为什么不用 `xl_running_xacts->xids` 直接构造完整 snapshot，而要用两次 `nextXid` / `oldestRunningXid` 边界推进？

3. `CreateInitDecodingContext()` 为什么要在找 startpoint 前设置 `slot->data.catalog_xmin` 和 `effective_catalog_xmin`？

4. `LogicalIncreaseXminForSlot()` 为什么把新 xmin 放进 candidate，而不是每次 running-xacts record 都直接 `ReplicationSlotSave()`？

5. `SnapBuildInitialSnapshot()` 为什么要求 `committed.includes_all_transactions`，而普通 decoding 只需要 catalog-changing transaction？

6. 如果创建 slot 时 crash 发生在 `DecodingContextFindStartpoint()` 之前，为什么 persistent slot 不应该留下？

7. 已有 slot 重启时为什么可以使用 serialized snapbuild snapshot，而新建 slot 时 `in_slot_creation` 要避免使用它？

8. 在诊断 slot 创建慢时，哪些信息能从 SQL 视图直接看到，哪些必须从 log、WAL dump 或断点推断？

## 23. 本节小结

创建 logical slot 不能立刻解码所有变更，因为 WAL 起点附近可能存在已经开始但尚未结束的事务。 decoder 如果只从创建时刻读，会丢事务前半段。 同时，历史 WAL 的 logical 解释需要历史 catalog snapshot，而不是当前 catalog。 PostgreSQL 用 `SnapBuild` 解决这个问题。 `LogStandbySnapshot()` 把 running transaction 信息写成 `XLOG_RUNNING_XACTS`。 `standby_decode()` 读到 `xl_running_xacts` 后调用 `SnapBuildProcessRunningXacts()`。 状态从 `SNAPBUILD_START` 推进到 `SNAPBUILD_BUILDING_SNAPSHOT`，再到 `SNAPBUILD_FULL_SNAPSHOT`，最后到 `SNAPBUILD_CONSISTENT`。 `SNAPBUILD_FULL_SNAPSHOT` 只表示新事务可以被完整收集。 `SNAPBUILD_CONSISTENT` 才表示旧事务边界已经越过，slot 可以得到 consistent point。 `CreateInitDecodingContext()` 先保护 `catalog_xmin`，再创建 `LogicalDecodingContext`、`ReorderBuffer` 和 `SnapBuild`。 `DecodingContextFindStartpoint()` 从 `slot->data.restart_lsn` 读 WAL，直到 `DecodingContextReady()` 为 true。 然后把 `ctx->reader->EndRecPtr` 写成 `slot->data.confirmed_flush`。 `catalog_xmin` 的后续推进通过 candidate 机制延迟发布。 只有客户端确认到对应 LSN 后，`LogicalConfirmReceivedLocation()` 才持久化新 `catalog_xmin`，再更新 `effective_catalog_xmin`。 slot 持久化也有明确边界。 permanent logical slot 创建时先是 `RS_EPHEMERAL`。 只有 consistent point 找到、snapshot action 完成后，才调用 `ReplicationSlotPersist()`。 异常路径包括 long transaction 等待、xmin horizon 太旧、WAL 读取失败、serialized snapshot 损坏、export snapshot 检查失败和 serialization I/O 失败。 诊断时要把可见状态和不可见状态分开。 SQL 能看到 `restart_lsn`、`confirmed_flush_lsn`、`catalog_xmin`。 日志能看到 state transition。 `pg_logicalinspect` 和断点才能看到部分 snapbuild 内部状态。 本节可迁移的系统规律是：
```text
当一个系统要从 append-only log 中恢复高层语义时，
不能只保存“读到哪里”；
还必须保存足以解释该位置之后所有事件的历史上下文，
并把“开始收集”、“可以解释”、“可以对外承诺”拆成不同状态。
```

logical decoding 的 snapbuild 正是这个规律在 PostgreSQL 中的实现。
