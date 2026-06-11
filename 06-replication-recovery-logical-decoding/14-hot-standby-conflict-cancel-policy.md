# PostgreSQL Hot Standby conflict 类型与取消策略
## 课程定位
前置知识：已经理解 physical streaming、startup process redo 状态机，以及 Hot Standby snapshot 如何由 running-xacts WAL record 和 KnownAssignedXids 支撑。
本节唯一主问题：
```text
REDO 需要清理 tuple、访问关系文件、推进锁或删除数据库时，
为什么 standby 查询会被取消，
max_standby_streaming_delay 和 feedback 分别牺牲什么？
```
核心矛盾：standby 想同时承担两个角色。
一边，它是恢复系统，必须按 WAL 顺序重放 primary 已经提交的物理事实。
另一边，它又是只读查询系统，查询希望持有 snapshot、relation lock、buffer pin、临时文件和数据库连接直到自己结束。
这两个目标不能总是同时满足。
如果 redo 被查询挡住，standby replay lag 会增长。
如果 redo 强行推进，查询可能继续读取 primary 上已经不存在、已经重写、已经删除或正在被 cleanup 的状态。
PostgreSQL 的解法不是让 standby 查询“读旧页”或“跳过 WAL”。
它把冲突压缩成几类 runtime 事实：
```text
snapshot conflict:
  查询的 xmin 太老，可能需要看到将被 redo 移除的 tuple version。

lock conflict:
  startup process 必须在 standby 上重放 primary 的 AccessExclusiveLock。

buffer pin conflict:
  redo 要 cleanup page，但某个 backend 仍 pin 着该 shared buffer。

tablespace/database conflict:
  redo 要删除文件系统对象，standby 仍有人可能使用它。
```
学完后应能判断：
```text
为什么 standby 取消查询不是 MVCC bug，而是 recovery ordering 的结果；
为什么 snapshotConflictHorizon 是 WAL 里携带的 cleanup cutoff；
为什么 max_standby_streaming_delay 牺牲的是 replay freshness；
为什么 hot_standby_feedback 牺牲的是 primary 的清理边界；
为什么 feedback 不能解决 AccessExclusiveLock、DROP DATABASE、DROP TABLESPACE 和 buffer pin；
如何从日志、wait event 和 pg_stat_database_conflicts 回到源码路径。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面三节已经建立：
```text
walsender / walreceiver 传输 WAL
  -> startup process 读取并 replay WAL
  -> running-xacts record 让 standby 能建立只读 snapshot
```
现在进入 Hot Standby 最容易误判的一层：
```text
只读查询已经被允许执行
  -> redo 后续遇到一个必须立即推进的物理状态
  -> standby 查询占着一个会让 redo 无法安全推进的资源
  -> startup process 等一段时间
  -> 到达 delay 截止点后取消查询或终止连接
```
本节不讲 primary 上 VACUUM 为什么判断 tuple 可移除。
那属于 heap pruning、vacuum 和 visibility horizon 的主题。
本节只讲 standby 收到相关 WAL 后，如何把“primary 已经做出的 cleanup / DDL / drop 决定”转成 standby 本地的等待、取消和诊断信号。
主线是一条线性链路：
```text
primary 写 WAL 记录
  -> WAL 记录携带 cleanup horizon 或 AccessExclusiveLock / drop 信息
  -> startup process replay 前检查 standby 查询是否挡路
  -> 如果挡路，按 max_standby_*_delay 等待
  -> 超时后通过 PROCSIG_RECOVERY_CONFLICT 通知 backend
  -> backend 在 CHECK_FOR_INTERRUPTS() 中 ERROR 或 FATAL
  -> pg_stat_database_conflicts 和日志留下可观测痕迹
```
这一节的重点不是背 conflict enum。
重点是看清楚每类 conflict 背后被保护的真实对象：
```text
tuple version
relation lock / relation file state
shared buffer page memory
database directory
tablespace directory / temp files
```
这些对象不在同一层。
所以它们不能用同一种“等待事务提交”的模型解释。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
Hot Standby 允许读查询持有 snapshot、lock 和 pin；
startup process 作为 WAL replay 的唯一推进者，必须按 primary 的 commit 顺序应用 cleanup、lock、drop 和 page rewrite；
当查询持有的 runtime 状态让 redo 无法安全推进时，
startup process 最多等到 GetXLogReceiptTime() + max_standby_*_delay，
之后给冲突 backend 发送 recovery conflict 信号，由 backend 自己抛 ERROR 或 FATAL 释放资源。
```
这句话包含三个边界。
第一，取消不是 primary 直接取消 standby 查询。
primary 只写 WAL。
standby 的 startup process 在 replay 某条 WAL 时发现本地查询挡住了恢复。
第二，取消不是立即发生的。
多数路径会先等 virtual transaction、lock 或 buffer pin 消失。
能等多久由 WAL receipt time 和 `max_standby_archive_delay` / `max_standby_streaming_delay` 决定。
第三，取消不是总能用 ERROR 收尾。
如果 backend idle in transaction、处在 subtransaction、冲突数据库正在被 drop，或者协议状态不能安全只取消当前语句，PostgreSQL 会 FATAL 终止连接。
这个设计牺牲了 standby 查询的持续性，换来两个不变量：
```text
redo 不跳过已经提交的 WAL；
standby 查询不继续依赖 primary 已经物理移除或逻辑排他的状态。
```
这里的 tension 可以写成：
```text
standby query availability
  vs
recovery progress and physical consistency
```
`max_standby_streaming_delay` 调整的是这一组 tension。
`hot_standby_feedback` 调整的是另一组 tension：
```text
standby query survival
  vs
primary vacuum / pruning / catalog cleanup freedom
```
二者经常一起讨论，但它们作用点完全不同。
`max_standby_streaming_delay` 是 standby replay 遇到冲突后“等多久再取消”。
`hot_standby_feedback` 是 standby 提前告诉 primary“别清理我可能还要看的版本”。
前者牺牲 standby apply freshness。
后者牺牲 primary cleanup horizon。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/standby.h` | `RecoveryConflictReason`、`ResolveRecoveryConflictWith*` 入口、standby rmgr 对外语义。 |
| 2 | `src/backend/storage/ipc/standby.c` | delay 截止时间、VXID 等待、snapshot/lock/database/tablespace/buffer pin 冲突处理。 |
| 3 | `src/backend/storage/ipc/procarray.c` | `GetConflictingVirtualXIDs()` 如何用 `PGPROC->xmin` 找老 snapshot，以及 conflict signal 如何写入 `pendingRecoveryConflicts`。 |
| 4 | `src/backend/tcop/postgres.c` | backend 收到 `PROCSIG_RECOVERY_CONFLICT` 后如何 ERROR / FATAL，并更新 conflict stats。 |
| 5 | `src/backend/access/heap/heapam_xlog.c` | heap prune / freeze WAL replay 如何先处理 `snapshotConflictHorizon` 再改页。 |
| 6 | `src/backend/access/heap/pruneheap.c` | primary 如何在 pruning 时计算并 WAL 记录 `conflict_xid`。 |
| 7 | `src/backend/storage/lmgr/lock.c` | primary 如何 WAL 记录 AccessExclusiveLock，standby 如何用 startup process 代理持锁。 |
| 8 | `src/backend/storage/lmgr/proc.c` | startup process 等 relation lock 时如何调用 `ResolveRecoveryConflictWithLock()`。 |
| 9 | `src/backend/storage/buffer/bufmgr.c` | `LockBufferForCleanup()` 如何把 cleanup lock 等待转成 buffer pin conflict。 |
| 10 | `src/backend/commands/dbcommands.c` | `XLOG_DBASE_DROP` replay 如何终止连接并删除数据库目录。 |
| 11 | `src/backend/commands/tablespace.c` | `XLOG_TBLSPC_DROP` replay 如何处理 standby temp file users。 |
| 12 | `src/backend/access/transam/xlogrecovery.c` | `GetXLogReceiptTime()` 和 streamed/archive WAL source 如何决定 delay 参数。 |
| 13 | `src/backend/replication/walreceiver.c` | `hot_standby_feedback` 如何从 standby 发出 xmin / catalog_xmin。 |
| 14 | `src/backend/replication/walsender.c` | primary 端如何把反馈 xmin 变成 walsender 或 physical slot 的 horizon。 |
| 15 | `src/backend/utils/activity/pgstat_database.c` | `pg_stat_database_conflicts` 的计数来源和粒度。 |
推荐阅读顺序不是“把 `standby.c` 从头读到尾”。
更有效的是沿一个冲突走完整：
```text
WAL replay 入口
  -> conflict target 选择
  -> wait policy
  -> signal delivery
  -> backend interrupt processing
  -> stats / log observation
```
本节会用 snapshot conflict 做主 walkthrough。
然后把 lock、database、tablespace 和 buffer pin 放回同一模型。
## 4. 关键状态：不是一个 conflict flag 就能解释全部
### 4.1 `RecoveryConflictReason`
`standby.h` 中的 `RecoveryConflictReason` 是 backend 收到信号后解释错误原因的枚举。
本节关注这些值：
```text
RECOVERY_CONFLICT_DATABASE
RECOVERY_CONFLICT_TABLESPACE
RECOVERY_CONFLICT_LOCK
RECOVERY_CONFLICT_SNAPSHOT
RECOVERY_CONFLICT_BUFFERPIN
RECOVERY_CONFLICT_STARTUP_DEADLOCK
RECOVERY_CONFLICT_BUFFERPIN_DEADLOCK
```
它不是“发生过哪些冲突”的全局日志。
它是一个待处理原因，被写入目标 backend 的 `PGPROC->pendingRecoveryConflicts`。
真正的动作发生在目标 backend 下一次处理中断时。
`procarray.c` 的 signal 路径做两件事：
```text
pg_atomic_fetch_or_u32(&proc->pendingRecoveryConflicts, 1 << reason)
SendProcSignal(pid, PROCSIG_RECOVERY_CONFLICT, procNumber)
```
`procsignal.c` 收到 `PROCSIG_RECOVERY_CONFLICT` 后只把 `InterruptPending` 置起。
真正抛错在 `postgres.c` 的 `ProcessRecoveryConflictInterrupts()`。
这解释了一个运行时现象：
```text
startup process 已经发了 conflict signal；
目标查询不一定在同一微秒立刻退出；
它要到可处理中断的位置才会 ERROR 或 FATAL。
```
### 4.2 `snapshotConflictHorizon`
`snapshotConflictHorizon` 是 snapshot conflict 的核心状态。
它的语义不是“当前 standby 最老 snapshot”。
它是 primary 在生成 WAL 时记录的 cleanup cutoff：
```text
这次物理移除、TID 删除、page reuse 或 VM all-visible 设置，
可能影响到 XID <= snapshotConflictHorizon 仍被视为运行中的 snapshot。
```
在 heap prune 路径中，primary 侧 `pruneheap.c` 通过 `HeapTupleHeaderAdvanceConflictHorizon()` 推进 `latest_xid_removed`。
随后 `log_heap_prune_and_freeze()` 在 `conflict_xid` 有效时设置 `XLHP_HAS_CONFLICT_HORIZON`，并把这个 TransactionId 写进 WAL。
standby 侧 `heapam_xlog.c` 的 `heap_xlog_prune_freeze()` 在真正改页前读出这个值。
如果 `InHotStandby`，它调用：
```text
ResolveRecoveryConflictWithSnapshot(snapshot_conflict_horizon,
                                    isCatalogRel,
                                    rlocator)
```
这个顺序很关键。
不是先改页再取消查询。
而是：
```text
先确保没有可能需要旧 tuple version 的 snapshot
  -> 再 replay page pruning / freezing / visibility map change
```
`InvalidTransactionId` 在这里有特殊语义：
```text
这条 WAL 记录明确不需要 snapshot conflict。
```
注意它和 `GetConflictingVirtualXIDs(InvalidTransactionId, ...)` 的语义相反。
在 `GetConflictingVirtualXIDs()` 中，invalid limit 表示 caller 想杀掉所有相关 active VXID。
课程里要避免把 raw value 当语义。
同一个 raw value 在不同函数边界里表达不同约定。
### 4.3 `PGPROC->xmin`
standby 查询是否挡住 cleanup，最后落到 `PGPROC->xmin`。
`GetConflictingVirtualXIDs(limitXmin, dbOid)` 在 `procarray.c` 中扫描 ProcArray。
对每个 backend：
```text
如果 dbOid 有效，只看同数据库 backend；
取 proc->xmin；
如果 limitXmin 无效，认为它冲突；
如果 proc->xmin 有效且不大于 limitXmin，认为它冲突；
把对应 VirtualTransactionId 放入返回数组。
```
这不是按 query text 判断。
也不是按 relation lock 判断。
它只问一个 snapshot horizon 问题：
```text
这个 backend 的 snapshot 是否可能还认为将被移除的 tuple version 可见？
```
这就是长事务、长游标、慢查询会触发 snapshot conflict 的原因。
### 4.4 `VirtualTransactionId`
startup process 不直接等待 PID。
它等待 `VirtualTransactionId` 消失。
`ResolveRecoveryConflictWithVirtualXIDs()` 对每个 VXID 调用 `VirtualXactLock(vxid, false)`。
返回 false 表示目标 virtual transaction 仍存在。
只要 VXID 消失，说明目标事务或 backend 已经释放了相关事务级状态。
这比 PID 更适合表达：
```text
我不关心是不是同一个 OS process；
我关心挡住 recovery 的那个事务生命周期是否结束。
```
如果等待超过 standby delay，startup process 才调用 `SignalRecoveryConflictWithVirtualXID()`。
### 4.5 `XLogReceiptTime` 和 WAL source
`max_standby_streaming_delay` 和 `max_standby_archive_delay` 的选择来自 `GetXLogReceiptTime()`。
`standby.c` 的 `GetStandbyLimitTime()` 做：
```text
GetXLogReceiptTime(&rtime, &fromStream)

if fromStream:
  limit = rtime + max_standby_streaming_delay
else:
  limit = rtime + max_standby_archive_delay
```
`-1` 表示 wait forever。
`0` 表示一旦 conflict 阻塞 recovery，就几乎不等。
这里最容易误解的是：
```text
delay 不是单条查询从开始执行起能跑多久。
```
在 streaming recovery 中，`xlogrecovery.c` 的注释说明：
当 startup process 跟得上 walreceiver 的最新 chunk 时，`XLogReceiptTime` 会推进。
当 standby 已经落后时，`XLogReceiptTime` 不再推进，给冲突查询的宽限时间会减少。
因此一个 standby 已经落后 30 秒时，即使 `max_standby_streaming_delay = 30s`，后续冲突也可能立刻取消。
这个参数限制的是：
```text
一段已收到 WAL 可以因为 standby 查询被阻塞多久。
```
不是：
```text
每个查询 guaranteed runtime。
```
## 5. 主流程源码 walkthrough：tuple cleanup 触发 snapshot conflict
先看 primary 侧。
一个 heap page pruning 或 VACUUM cleanup 发现某些 tuple version 已经可以物理移除。
在 `pruneheap.c` 中，pruning 遍历 HOT chain。
遇到 `HEAPTUPLE_DEAD` 时调用：
```text
HeapTupleHeaderAdvanceConflictHorizon(htup,
                                      &prstate->latest_xid_removed)
```
这个函数在 `heapam.c` 中从 tuple header 中取 `xmin`、update/delete `xmax` 和可能的 `xvac`。
它只把 committed XID 纳入 horizon。
原因是 standby 只需要保护“某个合法 snapshot 可能还会看到”的版本。
aborted tuple 不需要保护。
`latest_xid_removed` 最终传给：
```text
log_heap_prune_and_freeze(..., conflict_xid, ...)
```
`log_heap_prune_and_freeze()` 在 WAL record 中设置：
```text
XLHP_HAS_CONFLICT_HORIZON
XLHP_IS_CATALOG_REL
XLHP_CLEANUP_LOCK
```
其中 `XLHP_HAS_CONFLICT_HORIZON` 表示 main data 后面跟着一个 unaligned `TransactionId`。
`XLHP_IS_CATALOG_REL` 用于后续 logical slot invalidation 边界。
`XLHP_CLEANUP_LOCK` 表示 redo 需要 cleanup lock 语义。
接着 WAL 到达 standby。
startup process 在 `heapam_xlog.c` 进入 `heap_xlog_prune_freeze()`。
这条函数的时间顺序是：
```text
XLogRecGetBlockTag(record, 0, &rlocator, NULL, &blkno)
读 xl_heap_prune
如果有 XLHP_HAS_CONFLICT_HORIZON:
  memcpy 出 snapshot_conflict_horizon
  if InHotStandby:
    ResolveRecoveryConflictWithSnapshot(...)
然后才 XLogReadBufferForRedoExtended(...)
然后才 heap_page_prune_execute() / heap_execute_freeze_tuple() / VM bit 更新
```
这就是 recovery conflict 的核心原则：
```text
冲突处理发生在 redo 修改 shared buffer 之前。
```
`ResolveRecoveryConflictWithSnapshot()` 的内部流程：
```text
if snapshotConflictHorizon invalid:
  return

backends = GetConflictingVirtualXIDs(snapshotConflictHorizon, locator.dbOid)

ResolveRecoveryConflictWithVirtualXIDs(backends,
                                       RECOVERY_CONFLICT_SNAPSHOT,
                                       WAIT_EVENT_RECOVERY_CONFLICT_SNAPSHOT,
                                       true)

if logical decoding enabled and catalog relation:
  InvalidateObsoleteReplicationSlots(...)
```
这里有两个细节。
第一，`locator.dbOid` 会把普通 relation 的冲突限制在同一个 database。
共享 catalog relation 则可能影响所有数据库。
第二，logical decoding slot 的 invalidation 在 snapshot conflict 等待之后执行。
这个路径不是本节主问题，但它解释了为什么 `pg_stat_database_conflicts` 中还有 `confl_active_logicalslot`。
接着进入 `ResolveRecoveryConflictWithVirtualXIDs()`。
它对每个 VXID：
```text
standbyWait_us = 1000
while !VirtualXactLock(vxid, false):
  if WaitExceedsMaxStandbyDelay(wait_event):
    SignalRecoveryConflictWithVirtualXID(vxid, RECOVERY_CONFLICT_SNAPSHOT)
    if signaled:
      pg_usleep(5000)
  如果开启日志或 process title，按 deadlock_timeout 报告等待
```
`WaitExceedsMaxStandbyDelay()` 做三件事：
```text
CHECK_FOR_INTERRUPTS()
ltime = GetStandbyLimitTime()
如果当前时间 >= ltime，返回 true
否则 pg_usleep() 一小段时间，并报告 wait event
```
等待从 1ms 指数增长，最多 1s。
这不是精密调度器。
它是 startup process 在冲突期间避免 busy wait 的退避循环。
如果目标查询自己结束，VXID 消失，startup process 继续 replay。
如果目标查询不结束，到达 limit time 后，startup process 发信号。
目标 backend 收到 `PROCSIG_RECOVERY_CONFLICT`。
`procsignal.c` 调用 `HandleRecoveryConflictInterrupt()`。
它看到 `pendingRecoveryConflicts` 非零，就设置 `InterruptPending`。
下一次 `CHECK_FOR_INTERRUPTS()` 进入 `postgres.c` 的 `ProcessRecoveryConflictInterrupts()`。
它逐个清理 pending bit，并调用 `ProcessRecoveryConflictInterrupt(reason)`。
对于 `RECOVERY_CONFLICT_SNAPSHOT`，逻辑是：
```text
如果 backend 已经不在 transaction 中，忽略；
否则 report_recovery_conflict(RECOVERY_CONFLICT_SNAPSHOT)
```
`report_recovery_conflict()` 决定 ERROR 还是 FATAL。
普通顶层查询通常是 ERROR：
```text
canceling statement due to conflict with recovery
DETAIL: User query might have needed to see row versions that must be removed.
```
如果处在 subtransaction，或者 backend 正在等待 client input，ERROR 不一定能释放挡住 recovery 的资源。
这时会 FATAL：
```text
terminating connection due to conflict with recovery
```
这条 walkthrough 已经回答了第一类问题：
```text
REDO 清理 tuple 时为什么取消 standby 查询？
因为 WAL 已经表示 primary 物理移除了某些版本；
standby 上 `xmin <= snapshotConflictHorizon` 的查询还可能需要这些版本；
redo 不能跳过这条 WAL，也不能让查询继续依赖即将被修改的页；
所以只能等查询释放 snapshot，或取消它。
```
## 6. 第二条主线：AccessExclusiveLock WAL 与 relation file 状态
snapshot conflict 保护 tuple version。
lock conflict 保护 relation 级逻辑对象和关系文件状态。
primary 上执行 `DROP TABLE`、`TRUNCATE`、`ALTER TABLE`、`VACUUM FULL`、某些 rewrite 或 DDL 时，会拿 `AccessExclusiveLock`。
standby 查询通常会拿 `AccessShareLock`。
这两者冲突。
如果 standby 不重放 primary 的 AccessExclusiveLock，查询可能在 DDL 已经开始 replay 后继续访问一个被重写、截断或删除的 relation file。
PostgreSQL 只把可能和 standby 只读查询冲突的锁写入 WAL。
`lock.c` 的 `LockAcquireExtended()` 中：
```text
if lockmode >= AccessExclusiveLock
   && locktag is LOCKTAG_RELATION
   && !RecoveryInProgress()
   && XLogStandbyInfoActive():
  LogAccessExclusiveLockPrepare()
  log_lock = true
```
锁真正拿到后：
```text
LogAccessExclusiveLock(dbOid, relOid)
```
`LogAccessExclusiveLockPrepare()` 强制分配 transaction id。
它这样做有两个原因。
第一，事务结束时必须有 commit/abort WAL record，让 standby 知道何时释放这个 recovery lock。
第二，checkpoint 采集当前 AccessExclusiveLock 时不能看到 `InvalidTransactionId`。
standby replay 这类 WAL record 时，入口在 `standby.c` 的 `standby_redo()`：
```text
XLOG_STANDBY_LOCK:
  for each xl_standby_lock:
    StandbyAcquireAccessExclusiveLock(xid, dbOid, relOid)
```
`StandbyAcquireAccessExclusiveLock()` 不代表 startup process 在执行原始 DDL。
它是 primary 事务的代理锁：
```text
startup process 用 session lock 持有 AccessExclusiveLock；
RecoveryLockHash 按 xid/dbOid/relOid 去重；
RecoveryLockXidHash 让 commit/abort replay 时能按 xid 释放。
```
为什么用 session lock？
startup process 不是一个普通 SQL transaction owner。
session lock 避免依赖 ResourceOwner 生命周期。
释放在 redo commit/abort 时由 `StandbyReleaseLockTree()` 完成。
如果 standby query 已经持有 `AccessShareLock`，startup process 申请 `AccessExclusiveLock` 会进 lock wait。
这时 `proc.c` 的 `ProcSleep()` 在 `InHotStandby` 下调用：
```text
ResolveRecoveryConflictWithLock(locallock->tag.lock, maybe_log_conflict)
```
`ResolveRecoveryConflictWithLock()` 的策略与 snapshot conflict 相同但实现不同：
```text
算 GetStandbyLimitTime()
如果已经到期：
  GetLockConflicts(locktag, AccessExclusiveLock, NULL)
  ResolveRecoveryConflictWithVirtualXIDs(... RECOVERY_CONFLICT_LOCK ...)
否则：
  设置 STANDBY_LOCK_TIMEOUT 和 STANDBY_DEADLOCK_TIMEOUT
  ProcWaitForSignal(PG_WAIT_LOCK | locktag_type)
```
它还处理 startup process 和 standby backend 的 deadlock。
例如 standby 查询持有 buffer pin 后又等待某个 relation lock，而 startup process 又在等这个 buffer pin 或 lock。
普通 lock deadlock detector 不完整覆盖 buffer pin，所以 Hot Standby 有额外的 conservative 检查。
这一条链路回答“REDO 需要访问关系文件、推进锁时为什么取消查询”：
```text
primary 已经在 WAL 中记录了 AccessExclusiveLock；
standby 必须在相同顺序上阻止新的 relation readers；
已有 readers 可能挡住 startup process 获取这个 lock；
超过 delay 后，startup process 取消这些 readers；
否则后续 redo 可能在 relation file 被读者继续访问时 replay DDL / rewrite / truncate / drop 语义。
```
要注意：
```text
AccessExclusiveLock WAL 本身不修改 relation file。
它建立的是后续 relation file redo 的并发边界。
```
这也是为什么本节把它称为“访问关系文件状态”的保护层，而不是单纯“锁冲突”。
## 7. 第三条主线：DROP DATABASE 不等普通 VXID
`DROP DATABASE` 的冲突比 relation lock 更粗。
database 被删除时，不只是一个 relation 不可访问。
整个数据库目录、buffer cache、smgr fd、replication slot 和连接身份都要被清理。
standby replay 入口在 `dbcommands.c` 的 `dbase_redo()`。
遇到 `XLOG_DBASE_DROP`：
```text
if InHotStandby:
  LockSharedObjectForSession(DatabaseRelationId, db_id, 0, AccessExclusiveLock)
  ResolveRecoveryConflictWithDatabase(db_id)

ReplicationSlotsDropDBSlots(db_id)
DropDatabaseBuffers(db_id)
ForgetDatabaseSyncRequests(db_id)
XLogDropDatabase(db_id)
WaitForProcSignalBarrier(PROCSIGNAL_BARRIER_SMGRRELEASE)
for each tablespace:
  rmtree(database path)

if InHotStandby:
  UnlockSharedObjectForSession(...)
```
`ResolveRecoveryConflictWithDatabase()` 不调用 `ResolveRecoveryConflictWithVirtualXIDs()`。
源码注释明确指出原因：
```text
VXID 只覆盖 active transaction；
idle session 也会挡住 DROP DATABASE。
```
所以它循环：
```text
while CountDBBackends(dbid) > 0:
  SignalRecoveryConflictWithDatabase(dbid, RECOVERY_CONFLICT_DATABASE)
  pg_usleep(10000)
```
目标 backend 在 `postgres.c` 中收到 `RECOVERY_CONFLICT_DATABASE` 后直接 FATAL：
```text
terminating connection due to conflict with recovery
DETAIL: User was connected to a database that must be dropped.
```
这里没有“取消当前 statement 后继续留在同一个 database”这个选项。
database identity 本身要消失。
继续保持连接没有可恢复语义。
这也解释了 `pg_stat_database_conflicts` 的一个细节。
`pgstat_database.c` 对 `RECOVERY_CONFLICT_DATABASE` 不增加计数。
注释说数据库信息会随 drop 消失，因此没有意义计数。
诊断时不能只看 `confl_*` 总和判断是否发生过 database drop conflict。
日志和客户端 FATAL 才是主要证据。
## 8. 第四条主线：DROP TABLESPACE 与临时文件
tablespace drop 的对象不是单个查询 snapshot。
它是文件系统目录。
primary 上 `DROP TABLESPACE` 能成功，意味着当时没有 permanent objects 留在该 tablespace。
但 standby 上只读查询可能正在用这个 tablespace 写临时文件。
redo 入口在 `tablespace.c` 的 `tblspc_redo()`。
遇到 `XLOG_TBLSPC_DROP`：
```text
WaitForProcSignalBarrier(PROCSIGNAL_BARRIER_SMGRRELEASE)
if !destroy_tablespace_directories(ts_id, true):
  ResolveRecoveryConflictWithTablespace(ts_id)
  retry destroy_tablespace_directories(ts_id, true)
  if still fail:
    LOG directories could not be removed
```
`ResolveRecoveryConflictWithTablespace()` 的实现很粗：
```text
temp_file_users = GetConflictingVirtualXIDs(InvalidTransactionId, InvalidOid)
ResolveRecoveryConflictWithVirtualXIDs(temp_file_users,
                                       RECOVERY_CONFLICT_TABLESPACE,
                                       WAIT_EVENT_RECOVERY_CONFLICT_TABLESPACE,
                                       true)
```
为什么是 `InvalidTransactionId, InvalidOid`？
因为 startup process 没有便宜可靠的 shared state 能精确列出“谁正在这个 tablespace 里用 temp file”。
代码注释提到可以通过扫描 temp filename 找 pid 再转 VXID，但当前实现没有这么做。
因此它选择广泛取消 active transactions。
这会产生 false positive。
某些被取消的查询可能实际上没用那个 tablespace。
这是 PostgreSQL 在稀有路径上的工程取舍：
```text
不为罕见 DROP TABLESPACE replay 引入昂贵的 per-temp-file 精确索引；
用保守取消换取 redo 能继续推进。
```
如果重试删除目录仍失败，redo 不抛 ERROR 崩掉实例。
它只 LOG。
原因是文件权限、手工目录残留等 standby 本地问题不应该让 recovery 永远卡在同一条 WAL 上。
这是一条 fallback 边界：
```text
能通过取消查询清理的 temp file，尽量清理；
清理不了的本地残留，记录日志并继续 replay。
```
## 9. 第五条主线：buffer pin conflict
buffer pin conflict 保护的是 shared buffer 中一页的内存安全。
它不是 MVCC snapshot。
也不是 heavyweight relation lock。
当 redo 要删除 line pointer、compact page、cleanup index/heap page 或做其他需要 cleanup lock 的修改时，需要确认没有别的 backend 仍 pin 着这个 buffer。
`bufmgr.c` 的 `LockBufferForCleanup()` 说明这个协议：
```text
删除 disk page 上的 item 前，caller 必须持有 exclusive buffer lock；
并且观察到没有其他 backend 持有 pin；
否则别的 backend 可能手里还有指向 page item 的指针。
```
在普通 primary 上，等待 pin 消失即可。
在 Hot Standby 的 startup process 中，等待 pin 会阻塞 WAL replay。
`LockBufferForCleanup()` 在 `InHotStandby` 时做：
```text
SetStartupBufferPinWaitBufId(buffer - 1)
ResolveRecoveryConflictWithBufferPin()
SetStartupBufferPinWaitBufId(-1)
```
`ResolveRecoveryConflictWithBufferPin()` 的策略：
```text
ltime = GetStandbyLimitTime()
如果当前已经到期:
  SendRecoveryConflictWithBufferPin(RECOVERY_CONFLICT_BUFFERPIN)
否则:
  设置 STANDBY_TIMEOUT
  设置 STANDBY_DEADLOCK_TIMEOUT
  ProcWaitForSignal(WAIT_EVENT_BUFFER_CLEANUP)
如果 delay timeout:
  广播 RECOVERY_CONFLICT_BUFFERPIN
如果 deadlock timeout:
  广播 RECOVERY_CONFLICT_BUFFERPIN_DEADLOCK
```
这里的广播是故意的。
`SendRecoveryConflictWithBufferPin()` 调用：
```text
SignalRecoveryConflictWithDatabase(InvalidOid, reason)
```
它给所有 backend 发信号。
每个 backend 在 `postgres.c` 自己检查：
```text
HoldingBufferPinThatDelaysRecovery()
```
这个函数读 startup process 发布的 buffer id，然后看自己是否对该 buffer 有 private refcount。
大部分 backend 是无辜的，会直接 return。
真正持 pin 的 backend 抛 conflict error。
为什么不精确找 pin holder？
因为 buffer pin 是 backend-local refcount 加 shared buffer refcount 的组合，不像 ProcArray xmin 或 heavyweight lock 那样有一个容易扫描的“谁持有这个 pin”的全局索引。
为了 redo 进度，PostgreSQL 选择：
```text
广播信号
  -> backend 自查
  -> 只让真实 pin holder 退出
```
buffer pin conflict 的 error detail 是：
```text
User was holding shared buffer pin for too long.
```
如果涉及 deadlock，则 detail 是 buffer deadlock。
这类冲突常见于 standby 上长时间 cursor、慢 sequential scan、某些 extension 或 executor 路径长时间持 pin。
但 `pg_stat_database_conflicts.confl_bufferpin` 只能告诉你发生过这类取消。
它不能告诉你是哪张表、哪一页、哪个 query plan 节点持 pin。
需要结合日志、`pg_stat_activity`、query duration、wait event、必要时断点或 perf 分析。
## 10. wait policy：两个 max_standby 参数到底牺牲什么
`max_standby_archive_delay` 和 `max_standby_streaming_delay` 在 `standby.c` 中是毫秒整数，默认都是 30s。
GUC 定义在 `guc_parameters.dat`：
```text
max_standby_archive_delay:
  archived WAL replay 冲突时，取消查询前最大 delay；-1 表示永久等待。

max_standby_streaming_delay:
  streamed WAL replay 冲突时，取消查询前最大 delay；-1 表示永久等待。
```
选择哪一个不是由配置文件中是否写了 `primary_conninfo` 决定。
而是由 startup process 当前 replay 的 WAL chunk 来源决定。
`GetStandbyLimitTime()` 通过 `GetXLogReceiptTime()` 拿到：
```text
rtime
fromStream
```
`fromStream` 为 true 用 streaming delay。
否则用 archive delay。
这两个参数的共同代价是：
```text
允许 standby replay 被查询挡住。
```
直接表现是：
```text
pg_last_wal_replay_lsn() 落后；
replay_lag / apply lag 增大；
promotion 时可能少 replay 已收到但未应用的 WAL；
只读查询看到的状态更新更慢；
级联 standby 下游也会继续落后。
```
它不牺牲 primary 的空间。
primary 不会因为你调大 `max_standby_streaming_delay` 就保留更多 dead tuple。
primary 已经生成了 cleanup WAL。
standby 只是在本地晚一点应用。
调大 delay 适合的场景：
```text
standby 主要服务长报表；
可接受 replay lag；
故障切换时宁可先保护查询体验；
业务知道 standby freshness 不是强约束。
```
调小 delay 或设为 0 适合的场景：
```text
standby 主要用于低延迟只读；
需要快速接近 primary；
failover freshness 更重要；
长查询应该被拆分、迁移或跑在专用分析副本。
```
`-1` 的含义非常重。
它不是“给查询更多时间”这么简单。
它表示 recovery 可以无限期等冲突资源释放。
一个 idle in transaction 或挂住的 cursor 可能让 standby 永远不 replay 后续 WAL。
这会影响监控、promotion 预期、级联复制和磁盘保留策略。
因此 `-1` 通常只适合明确隔离的分析 standby。
还有一个常见误区：
```text
max_standby_streaming_delay = 30s 不代表每个查询最多跑 30s 后才会被取消。
```
如果 WAL chunk 30 秒前已经收到，只是 standby 一直落后，那么到达冲突记录时 limit time 可能已经过去。
这时查询会马上被取消。
参数约束的是：
```text
replay 对已收到 WAL 的容忍滞后。
```
不是：
```text
query runtime SLA。
```
## 11. hot_standby_feedback：减少 snapshot conflict 的另一种牺牲
`hot_standby_feedback` 作用在 conflict 发生之前。
walreceiver 周期性给 upstream 发送 standby 的 visibility horizon。
入口在 `walreceiver.c` 的 `XLogWalRcvSendHSFeedback()`。
它有几个边界：
```text
如果 Hot Standby 还没 active，不发送有效 feedback；
如果 hot_standby_feedback 关闭，发送 InvalidTransactionId 清掉 primary 端 xmin；
发送频率受 wal_receiver_status_interval 控制；
有效时调用 GetReplicationHorizons(&xmin, &catalog_xmin)。
```
消息内容包括：
```text
timestamp
xmin
xmin_epoch
catalog_xmin
catalog_xmin_epoch
```
primary 端 `walsender.c` 处理 feedback。
如果反馈值有效，它把这个 xmin 放进：
```text
MyProc->xmin
```
或者在使用 replication slot 时调用：
```text
PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)
```
源码注释说得很直接：
```text
这会让 GetSnapshotData() / ComputeXidHorizons() 把 standby 的 xmin 纳入考虑，
从而推迟 dead rows 移除，避免 standby cleanup conflicts。
```
所以 feedback 牺牲的是 primary 的清理自由度。
具体表现是：
```text
VACUUM 不能移除某些 dead tuple；
HOT pruning 和 index cleanup 空间回收变慢；
catalog_xmin 滞留会保护 catalog 旧版本；
表和索引 bloat 增长；
pg_xact / pg_subtrans 等旧事务状态保留压力可能增大；
高写入 primary 上 autovacuum 做更多无效工作。
```
如果没有 replication slot，walsender 的 xmin 是会话级、非持久的。
连接断开或 feedback 未及时到达时，primary 可能已经清理了 standby 后续查询需要的版本。
`procarray.c` 的 horizon 注释也说明：
没有 slot 持久保护时，数据只在 walsender 持续运行期间被保护；
Hot Standby 通过取消需要已移除数据的查询来兜底 correctness。
如果使用 physical replication slot，xmin 可以通过 slot 更稳定地参与 primary horizon。
但这也意味着主库 bloat 风险更稳定、更持久。
feedback 能减少的主要是 snapshot cleanup conflict。
它不能解决：
```text
AccessExclusiveLock WAL replay conflict；
DROP DATABASE；
DROP TABLESPACE；
buffer pin conflict；
startup deadlock；
standby 本地 temp file / cursor 持有；
primary 已经在 feedback 生效前清理掉的版本。
```
因此 feedback 不是“让 standby 查询不再被取消”的总开关。
它只是把一部分取消风险，从 standby replay 时刻，转移成 primary cleanup horizon 的保留成本。
这就是两个旋钮的根本差异：
```text
max_standby_*_delay:
  已经收到 cleanup / DDL WAL 后，在 standby 本地等多久。
  牺牲 standby replay freshness。

hot_standby_feedback:
  在 primary 生成 cleanup WAL 前，阻止 primary 太早清理。
  牺牲 primary vacuum / pruning / catalog cleanup。
```
## 12. 生命周期 / ownership / cleanup
### snapshot conflict 的 ownership
`snapshotConflictHorizon` 由 primary cleanup 路径计算。
它被写进 WAL record。
standby startup process 只消费它。
冲突目标由 standby 本地 ProcArray 扫描得出。
被取消 backend 通过事务 abort 释放 snapshot、lock、pin、portal 等资源。
如果 ERROR 不足以释放资源，则 FATAL 释放整个 session。
### lock conflict 的 ownership
primary 持有 AccessExclusiveLock。
WAL 记录这个事实。
standby startup process 作为代理用 session lock 持有 AccessExclusiveLock。
`RecoveryLockHash` 和 `RecoveryLockXidHash` 记录 xid 到锁的映射。
commit/abort redo 调用 `StandbyReleaseLockTree()` 释放。
end of recovery 或 shutdown 调用 `StandbyReleaseAllLocks()` 清理。
如果 crash recovery 中有未完成事务没有 commit/abort record，`StandbyReleaseOldLocks()` 会按 old xid 清理非 prepared transaction 的旧锁。
### database conflict 的 ownership
database 连接属于各 backend。
database 目录删除属于 startup process redo。
`ResolveRecoveryConflictWithDatabase()` 不等待普通 VXID，而是按 database id signal 所有连接。
backend 只能 FATAL。
数据库目录、buffers、smgr fd、sync requests 和 db-specific slots 由 redo 路径逐步清理。
### tablespace conflict 的 ownership
tablespace 目录属于文件系统状态。
standby temp file 由各 backend 创建和清理。
redo 尝试删除目录失败后，取消可能使用 temp file 的 active transactions。
如果取消后仍删不掉，只 LOG 并继续。
### buffer pin conflict 的 ownership
pin 是 backend-local refcount 与 shared buffer refcount 的组合。
startup process 只能发布自己等待的 buffer id。
所有 backend 收信号后自查是否持有该 pin。
真实持有者 ERROR/FATAL 后，ResourceOwner 和 buffer manager cleanup 释放 pin。
## 13. 正确性机制层次
| 机制 | 保证什么 | 不能保证什么 |
| --- | --- | --- |
| WAL order | standby 以 primary 的提交物理事实顺序推进 | 不能让旧查询永远依赖被移除状态 |
| MVCC snapshot | 查询看到哪个 tuple version | 不能阻止 redo 删除已经 WAL-logged 的 dead version |
| `snapshotConflictHorizon` | cleanup WAL 对 standby snapshot 的 cutoff | 不是当前全局 oldest xmin |
| ProcArray scan | 找出可能冲突的 VXID | 不能解释 query text 或 plan node |
| heavyweight lock | relation 级 AccessExclusive 与 AccessShare 互斥 | 不能保护 buffer page 内存指针 |
| buffer pin | 保护 backend 正在引用的 page memory | 不能表达 MVCC visibility |
| PROCSIG | 异步通知 backend 自己抛错 | 不能保证目标 backend 立即退出 |
| `max_standby_*_delay` | 限制 recovery 等待本地查询的时长 | 不能阻止 primary 清理 |
| `hot_standby_feedback` | 推迟 primary cleanup horizon | 不能解决 DDL/drop/buffer pin |
这张表背后的规律是：
```text
PostgreSQL 不用一个统一锁解决所有 recovery conflict。
它按资源层次选择最小足够的保护机制。
```
tuple version 用 snapshot horizon。
relation file state 用 AccessExclusiveLock。
page memory 用 buffer pin。
database / tablespace 目录用进程终止和文件系统重试。
这种分层让 hot path 保持局部化，但也让诊断必须先判断冲突属于哪一层。
## 14. 错误路径 / 异常路径 / fallback
### 14.1 ERROR 升级为 FATAL
`report_recovery_conflict()` 的目标是释放挡住 recovery 的资源。
如果能通过 abort 当前 statement / transaction 释放，抛 ERROR。
如果不能，抛 FATAL。
典型 FATAL：
```text
RECOVERY_CONFLICT_DATABASE:
  database 要被删除，连接身份失效。

idle in transaction:
  backend 正在等 client input，不在安全执行 command 的中间。

subtransaction:
  父事务可能仍持有 snapshot / locks / temp file。
```
这解释了为什么有时客户端看到的是 statement canceled，有时连接直接断开。
二者都来自同一类 recovery conflict，但 cleanup 边界不同。
### 14.2 startup deadlock
Hot Standby deadlock 有两种。
一种是 startup process 等 relation lock，而持锁 backend 又在等 startup process 释放的锁。
`ResolveRecoveryConflictWithLock()` 在 `deadlock_timeout` 后给 blockers 发送 `RECOVERY_CONFLICT_STARTUP_DEADLOCK`。
目标 backend 如果正在等 lock，会触发 deadlock check。
另一种是 buffer pin deadlock。
startup process 等 buffer pin。
某个 backend 持有这个 pin，又在等一个只能由 startup process replay commit/abort 后释放的 lock。
普通 deadlock detector 不跟踪 buffer pin。
所以 `RECOVERY_CONFLICT_BUFFERPIN_DEADLOCK` 是保守自查。
源码也承认这里可能 false positive，但实践中概率低，没必要为 buffer pin 建完整 deadlock graph。
### 14.3 tablespace 删除失败后继续
`tblspc_redo()` 取消冲突后会重试删除目录。
如果仍失败，记录 LOG 并继续。
这不是忽略 correctness。
因为 primary 的永久对象已经不存在。
剩下的是 standby 本地目录残留、权限或 temp file cleanup 问题。
让 recovery 永远卡住这条 WAL 的运维成本更高。
### 14.4 feedback 迟到
feedback 是异步的。
它受 `wal_receiver_status_interval`、网络、walsender 处理、slot 状态和 primary 当前 vacuum 进度影响。
如果 primary 已经生成 cleanup WAL，feedback 不能撤销那条 WAL。
standby 只能在 replay 时取消查询。
所以“开启 feedback 仍然发生 snapshot conflict”并不矛盾。
可能原因包括：
```text
feedback 生效前 primary 已经 cleanup；
walreceiver 尚未进入 HotStandbyActive；
没有 slot 且连接中断；
上游级联节点没有继续传递有效 horizon；
冲突不是 snapshot cleanup，而是 lock / drop / buffer pin。
```
## 15. 成本、资源与跨模块传播
### standby delay 的成本
调大 `max_standby_streaming_delay` 的直接成本是 replay lag。
它会传播到：
```text
读请求 freshness；
promotion 后可用的最新 replay LSN；
级联 standby 的下游延迟；
监控系统的 lag 告警；
WAL 保留策略，因为下游 replay 更慢。
```
如果 standby 也有 replication slot 或 archive retention，lag 会进一步变成磁盘压力。
但根因仍是 standby 本地 replay 没有推进。
### feedback 的成本
开启 `hot_standby_feedback` 的直接成本在 primary。
长查询 standby 会把 xmin 反馈给 primary。
primary 的 VACUUM、HOT pruning、index cleanup、catalog cleanup 都会更保守。
成本随这些变量扩大：
```text
standby 最长查询时长；
primary 写入 / 更新 / 删除量；
表和索引大小；
autovacuum 周期；
是否有 physical slot 持久保留 xmin；
是否涉及 catalog_xmin；
级联复制链路长度。
```
反馈保护的是“不要生成会让 standby 老 snapshot 失败的 cleanup”。
它不是“减少 WAL”。
相反，primary 因为 bloat 可能产生更多 I/O 和更长 VACUUM。
### lock conflict 的成本
AccessExclusiveLock conflict 的成本取决于 relation DDL 和 standby reader 的重叠。
一个长查询拿着 AccessShareLock，会挡住 startup process replay AccessExclusiveLock。
后续 WAL 即使和这张表无关，也不能越过当前记录乱序 replay。
所以局部 DDL 可以放大全局 apply lag。
### buffer pin 的成本
buffer pin conflict 的成本不随事务数简单扩张。
它取决于执行器和访问方法持 pin 的时间。
大多数 pin 很短。
问题通常出现在慢 I/O、长游标、扩展代码、某些 executor 节点或查询卡在持 pin 状态下。
因为诊断视图不直接暴露 pin holder 的页号到 SQL，定位成本比 snapshot conflict 高。
### ProcArray scan 成本
snapshot 和 tablespace conflict 都会扫描 ProcArray。
`GetConflictingVirtualXIDs()` 的成本随 backend 数增长。
这是冲突慢路径，不是普通查询 hot path。
但在高连接 standby 上，大量冲突会放大 startup process 的 CPU 和信号成本。
## 16. 观测与诊断入口
### 16.1 客户端错误
普通 statement cancel：
```text
ERROR: canceling statement due to conflict with recovery
DETAIL: User query might have needed to see row versions that must be removed.
```
lock conflict：
```text
DETAIL: User was holding a relation lock for too long.
```
buffer pin：
```text
DETAIL: User was holding shared buffer pin for too long.
```
tablespace：
```text
DETAIL: User was or might have been using tablespace that must be dropped.
```
database drop：
```text
FATAL: terminating connection due to conflict with recovery
DETAIL: User was connected to a database that must be dropped.
```
这些 detail 文案在 `postgres.c` 的 `errdetail_recovery_conflict()` 中。
### 16.2 server log
`log_recovery_conflict_waits` 控制 startup process 是否记录长时间等待 recovery conflict。
超过 `deadlock_timeout` 后，`standby.c` 的 `LogRecoveryConflict()` 会记录：
```text
recovery still waiting after ...: recovery conflict on snapshot
recovery finished waiting after ...: recovery conflict on snapshot
```
如果能构造出冲突 backend 列表，detail log 会带 PID。
lock 和 buffer pin 路径也会在对应等待超过 `deadlock_timeout` 时记录。
这类日志说明 startup process 被冲突挡住过。
它不是每次取消的完整审计日志。
真正的 ERROR/FATAL 是否进日志，还取决于 `log_min_error_statement`、连接日志和客户端行为。
### 16.3 `pg_stat_database_conflicts`
视图定义在 `system_views.sql`：
```text
datid
datname
confl_tablespace
confl_lock
confl_snapshot
confl_bufferpin
confl_deadlock
confl_active_logicalslot
stats_reset
```
计数在 `pgstat_database.c` 的 `pgstat_report_recovery_conflict()` 中增加。
粒度是 database 累计计数。
它能回答：
```text
这个 standby 上哪类 conflict 被 backend 报告过？
```
它不能回答：
```text
哪条 WAL record 触发？
哪个 relation / block？
哪个 exact query？
等待了多久？
是否是 feedback 迟到？
```
`RECOVERY_CONFLICT_DATABASE` 不计数。
因为 database 被 drop 后统计项也没有稳定归属。
### 16.4 wait events
startup process 等 snapshot conflict 时会报告：
```text
RecoveryConflictSnapshot
```
DROP TABLESPACE conflict：
```text
RecoveryConflictTablespace
```
buffer cleanup 等待：
```text
BufferCleanup
```
lock conflict 走 lock wait event：
```text
Lock / relation locktag type
```
这些 wait event 表示 startup process 或 backend 当下在等什么。
它们不是持久历史。
如果冲突已经取消并继续 replay，wait event 就消失。
### 16.5 replication lag 与 replay LSN
诊断 standby conflict 必须同时看 replay 是否被挡住：
```text
select pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
select now() - pg_last_xact_replay_timestamp();
```
如果 receive 前进而 replay 不前进，同时日志出现 recovery conflict wait，就说明 WAL 已经到了本机，但 redo 被本地查询挡住。
如果 receive 本身不前进，那是传输或上游问题，不是 Hot Standby conflict 本身。
### 16.6 primary bloat 侧证据
开启 feedback 后，应在 primary 侧看：
```text
pg_stat_activity 中 walsender 的 backend_xmin
pg_replication_slots 的 xmin / catalog_xmin
表的 n_dead_tup、vacuum 次数、膨胀估计
autovacuum 日志
```
如果 standby conflict 减少但 primary bloat 增长，这正是 feedback 的预期 trade-off。
不要把它诊断成单纯 autovacuum 不工作。
## 17. 常见误区
误区一：
```text
max_standby_streaming_delay 是查询最大执行时间。
```
它不是。
它是以 WAL receipt time 为起点的 recovery wait budget。
standby 已经落后时，新遇到的冲突可能没有剩余时间。
误区二：
```text
hot_standby_feedback 开启后 standby 查询不会被取消。
```
feedback 只影响 primary cleanup horizon，主要减少 snapshot cleanup conflict。
它不能解决 lock、drop、tablespace、buffer pin，也不能撤销已经生成的 cleanup WAL。
误区三：
```text
snapshot conflict 表示 standby snapshot 算错了。
```
不是。
snapshot 是正确的。
问题是 primary 已经合法移除了该 snapshot 可能需要的旧版本。
standby 必须在继续 redo 和保留查询之间选一个。
误区四：
```text
confl_snapshot 增加就一定是某张业务表 VACUUM 太激进。
```
可能是 heap pruning、index deletion、page reuse、catalog relation cleanup，也可能是 feedback 迟到。
需要结合 WAL 位置、日志和 primary vacuum 行为判断。
误区五：
```text
buffer pin conflict 可以从 pg_locks 找到 holder。
```
找不到。
buffer pin 不是 heavyweight lock。
只能通过 backend 自查、日志、查询形态、wait event、断点或 profiling 缩小范围。
误区六：
```text
DROP DATABASE conflict 会出现在 pg_stat_database_conflicts。
```
源码不会计数 database conflict。
因为 database 被删除后统计归属不稳定。
要看 FATAL 日志和 `dbase_redo()` 路径。
## 18. 课堂实验
### 实验一：snapshot conflict
目标：看到 `snapshotConflictHorizon -> VXID -> ERROR` 的链路。
准备一主一备，备库开启 Hot Standby。
备库 session A：
```text
begin;
select count(*) from t;
-- 保持事务不提交，必要时用游标或 pg_sleep 延长 snapshot 生命周期
```
主库：
```text
delete from t where ...;
vacuum t;
```
备库观察：
```text
select * from pg_stat_database_conflicts where datname = current_database();
select pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn();
```
把 `max_standby_streaming_delay` 调小，再重复。
预期现象：
```text
备库长事务被 cancel；
confl_snapshot 增加；
receive LSN 可能领先 replay LSN；
日志可能出现 recovery conflict on snapshot。
```
源码回读：
```text
pruneheap.c: log_heap_prune_and_freeze()
heapam_xlog.c: heap_xlog_prune_freeze()
standby.c: ResolveRecoveryConflictWithSnapshot()
procarray.c: GetConflictingVirtualXIDs()
postgres.c: report_recovery_conflict()
```
### 实验二：AccessExclusiveLock conflict
目标：看到 relation lock 如何挡住 redo。
备库 session A：
```text
begin;
select * from t, pg_sleep(60);
```
主库 session B：
```text
alter table t add column c int;
```
观察备库：
```text
select wait_event_type, wait_event, state, query
from pg_stat_activity
where backend_type = 'startup' or query like '%pg_sleep%';

select confl_lock
from pg_stat_database_conflicts
where datname = current_database();
```
预期现象：
```text
startup process 可能等待 relation lock；
超过 delay 后，备库查询被 cancel；
confl_lock 增加。
```
源码回读：
```text
lock.c: LogAccessExclusiveLockPrepare()
lock.c: LogAccessExclusiveLock()
standby.c: standby_redo()
standby.c: StandbyAcquireAccessExclusiveLock()
proc.c: ResolveRecoveryConflictWithLock()
```
### 实验三：feedback 的 bloat 取舍
目标：比较“取消查询”和“primary 保留 dead tuples”的取舍。
设置两轮实验。
第一轮关闭 `hot_standby_feedback`，运行长 standby 查询，同时主库大量 update/delete + vacuum。
第二轮开启 `hot_standby_feedback`，保持相同 workload。
比较：
```text
备库 pg_stat_database_conflicts.confl_snapshot
主库 pg_stat_activity 中 walsender backend_xmin
主库 pg_replication_slots.xmin / catalog_xmin
主库目标表 n_dead_tup 和 vacuum 日志
```
预期：
```text
关闭 feedback 时，standby 更容易 cancel；
开启 feedback 时，standby cancel 减少，但 primary cleanup horizon 后退，dead tuple 保留更久。
```
源码回读：
```text
walreceiver.c: XLogWalRcvSendHSFeedback()
procarray.c: GetReplicationHorizons()
walsender.c: ProcessStandbyHSFeedbackMessage()
procarray.c: ComputeXidHorizons()
```
## 19. 讨论题
1. 为什么 snapshot conflict 不能靠 standby 自己保留旧 page 版本解决？
2. 为什么 `InvalidTransactionId` 在 `ResolveRecoveryConflictWithSnapshot()` 和 `GetConflictingVirtualXIDs()` 中语义不同？
3. 为什么 AccessExclusiveLock 要写 WAL，而 AccessShareLock 不需要写 WAL 给 standby？
4. 为什么 DROP DATABASE 必须 FATAL，而不是只取消当前 statement？
5. 为什么 buffer pin conflict 要广播给所有 backend 自查，而不是 startup process 精确找到 holder？
6. 如果 standby 已经落后 5 分钟，`max_standby_streaming_delay = 30s` 对新冲突意味着什么？
7. 开启 `hot_standby_feedback` 后，primary 的哪几个指标应该一起观察？
8. 为什么 `pg_stat_database_conflicts` 不能单独作为完整因果证据？
## 20. 本节小结
本节的主链路是：
```text
primary 做 cleanup / DDL / drop
  -> WAL 携带 snapshotConflictHorizon、AccessExclusiveLock 或 drop record
  -> standby startup process 按 WAL 顺序 replay
  -> redo 前发现本地查询持有 snapshot / lock / pin / database connection / temp file
  -> startup process 最多等到 WAL receipt time + max_standby_*_delay
  -> 超时后发送 recovery conflict signal
  -> backend 在 interrupt path 中 ERROR 或 FATAL
  -> stats 和日志留下 coarse-grained 证据
```
核心状态不是一个字段。
它是这些状态的组合：
```text
WAL record 中的 cleanup horizon 或 lock/drop 信息
standby ProcArray 中 backend 的 xmin / VXID
lock table 中 AccessShareLock 与 startup process 代理 AccessExclusiveLock
shared buffer pin refcount
database / tablespace 文件系统状态
XLogReceiptTime 与 WAL source
pendingRecoveryConflicts 与 backend interrupt handling
```
`max_standby_streaming_delay` 和 `max_standby_archive_delay` 牺牲的是 standby replay freshness。
它们让查询多活一会儿，但 WAL apply lag 会增长。
`hot_standby_feedback` 牺牲的是 primary cleanup freedom。
它让 primary 少生成会伤害 standby 老 snapshot 的 cleanup WAL，但 dead tuple、index bloat 和 catalog horizon 压力会转移到 primary。
因此，面对 Hot Standby 查询取消，正确诊断顺序是：
```text
先判断 conflict 类型；
再判断它保护的是 tuple version、relation lock、buffer pin、database 还是 tablespace；
再看 replay lag 和 WAL source；
最后决定调 delay、开 feedback、拆查询、隔离分析 standby，还是减少 primary DDL / cleanup 与 standby 长查询重叠。
```
可迁移的系统规律是：
```text
一个恢复系统一旦同时提供读服务，就必须为“历史读状态”和“日志推进状态”建立冲突裁决。
裁决可以选择等、取消本地读、或让上游延迟回收；
但不可能既无限保留读体验，又无限保持 replay 新鲜，还不让 primary 付出空间成本。
```
