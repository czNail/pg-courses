# PostgreSQL 同步复制反馈与 commit 等待边界
## 课程定位
前置知识：已经知道 physical streaming replication 的基本角色：primary 上 `walsender` 发送 WAL，standby 上 `walreceiver` 接收并写入 WAL，startup process 按 LSN 顺序 replay WAL。
本节唯一主问题：
```text
主库提交事务时为什么可能等待 standby 的 write、flush 或 apply 反馈，
synchronous_commit 和 synchronous standby 选择如何影响事务可见延迟？
```
核心矛盾：primary 希望尽快向客户端确认 commit；用户又可能要求这个 commit 在一个或多个 standby 上达到更强的边界后才确认。
本节不把同步复制讲成“开关”。它是一组边界叠加：本地 WAL 是否 flush、standby 是否 write、standby 是否 flush、standby 是否 apply、primary backend 是否离开 ProcArray、客户端是否收到 COMMIT 成功。
学完后应能判断：`synchronous_commit=off/local/remote_write/on/remote_apply` 分别在哪里停；`SyncRepWaitForLSN()` 等待哪个 LSN；`SyncRepReleaseWaiters()` 如何用 standby 反馈唤醒队列；`FIRST` 和 `ANY` 如何改变释放条件；为什么同步复制会延迟 primary 本地可见性。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前几节的 physical streaming 主线可以压缩成：
```text
replication connection handshake
  -> walsender 读取并发送 primary WAL
  -> walreceiver 接收、写入、flush standby WAL
  -> startup process replay WAL
```
本节只盯住 commit 这一刻。
主链路是：
```text
backend 写 commit record
  -> primary 本地 WAL flush / pg_xact 标记
  -> 如果需要同步复制，进入 SyncRepWaitForLSN()
  -> standby 反馈 write / flush / apply
  -> walsender 调用 SyncRepReleaseWaiters()
  -> backend 退出等待，继续事务收尾
```
这条链路里有两个时间流。
backend 的时间流：产生 WAL、flush 本地 WAL、标记 pg_xact、入同步复制等待、被唤醒、离开 ProcArray、释放锁、向客户端确认。
standby 的时间流：接收 WAL、写入 WAL 文件、fsync WAL、startup process replay commit record、walreceiver 发 status update、walsender 更新 shared state。
本节后面所有状态、边界和诊断都回到这两个时间流。
## 2. 一句话运行模型
一句话运行模型：
```text
提交 backend 把自己的 commit LSN 放进按 LSN 排序的同步复制等待队列；
walsender 收到 standby status update 后更新 WalSnd.write/flush/apply；
SyncRepReleaseWaiters() 根据 synchronous_standby_names 选择当前同步 standby，
计算每种 wait mode 能释放到的 LSN，并唤醒 waitLSN 已满足的 backend。
```
这里的 tension 是：
```text
commit ack 越早，primary latency 越低；
commit ack 越晚，ack 表达的跨节点 durability / visibility 语义越强。
```
PostgreSQL 没有让 standby 理解每个 primary 事务的同步策略。
standby 只报告自己的位置。
primary 自己解释这些位置：
```text
这个事务是否需要等
等 write 还是 flush 还是 apply
等哪些 standby
等几个 standby
等待取消或配置降级时如何收尾
```
这让策略集中在 primary，也让 primary commit path 和 `walsender` shared state、`PGPROC` wait queue、GUC reload、standby reply 节奏耦合在一起。
## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/xact.h` | `SyncCommitLevel` 和 `SYNCHRONOUS_COMMIT_ON`。 |
| 2 | `src/backend/access/transam/xact.c` | `RecordTransactionCommit()` 的本地 flush、pg_xact 标记、`SyncRepWaitForLSN()` 调用位置。 |
| 3 | `src/backend/replication/syncrep.c` | `SyncRepWaitForLSN()`、队列、standby 选择、`SyncRepReleaseWaiters()`。 |
| 4 | `src/include/replication/syncrep.h` | wait mode、sync rep state、`SyncRepConfigData`、`SyncRepStandbyData`。 |
| 5 | `src/include/replication/walsender_private.h` | `WalSnd`、`WalSndCtlData`、`SyncRepQueue[]`。 |
| 6 | `src/backend/replication/walsender.c` | standby reply 解析、`WalSnd.write/flush/apply` 推进、`pg_stat_replication`。 |
| 7 | `src/backend/replication/walreceiver.c` | standby write / flush / apply 如何组成 status update。 |
| 8 | `src/backend/access/transam/xlogrecovery.c` | startup process replay 何时请求 apply reply。 |
| 9 | `src/backend/utils/activity/wait_event_names.txt` | `SyncRep` wait event 的诊断含义。 |
阅读源码时不要从单文件顶部线性读。
建议按这条问题链读：
```text
commit 为什么会等
  -> wait state 放在哪里
  -> standby 反馈如何进入 WalSnd
  -> 哪些 WalSnd 被选成 synchronous
  -> release LSN 如何唤醒队列头部 backend
  -> cancel / disconnect / GUC 降级如何避免永久卡住
```
## 4. 关键状态与边界
### 4.1 `synchronous_commit` 是 ack 等级
`src/include/access/xact.h` 的 `SyncCommitLevel` 有五档：
```text
SYNCHRONOUS_COMMIT_OFF
SYNCHRONOUS_COMMIT_LOCAL_FLUSH
SYNCHRONOUS_COMMIT_REMOTE_WRITE
SYNCHRONOUS_COMMIT_REMOTE_FLUSH
SYNCHRONOUS_COMMIT_REMOTE_APPLY
```
用户常用 GUC 值在 `guc_tables.c` 中映射为：

| GUC 值 | C 枚举 | commit ack 边界 |
| --- | --- | --- |
| `off` | `SYNCHRONOUS_COMMIT_OFF` | 不等本地 WAL flush，也不等 standby。 |
| `local` | `SYNCHRONOUS_COMMIT_LOCAL_FLUSH` | 等 primary 本地 WAL flush，不等 standby。 |
| `remote_write` | `SYNCHRONOUS_COMMIT_REMOTE_WRITE` | 等 primary 本地 flush，再等同步 standby write。 |
| `on` | `SYNCHRONOUS_COMMIT_REMOTE_FLUSH` | 等 primary 本地 flush，再等同步 standby flush。 |
| `remote_apply` | `SYNCHRONOUS_COMMIT_REMOTE_APPLY` | 等 primary 本地 flush，再等同步 standby apply。 |
`SYNCHRONOUS_COMMIT_ON` 定义为 `SYNCHRONOUS_COMMIT_REMOTE_FLUSH`。
所以用户说的 `on`，在内部 wait mode 上是 remote flush。
`syncrep.h` 中 `SyncRepRequested()` 是：
```text
max_wal_senders > 0 && synchronous_commit > SYNCHRONOUS_COMMIT_LOCAL_FLUSH
```
因此只有 `remote_write`、`on`、`remote_apply` 可能进入同步复制等待。
`off` 和 `local` 都不会进入 `SyncRepWaitForLSN()` 的等待队列。
### 4.2 GUC 到 wait queue 的映射
`syncrep.c` 的 `assign_synchronous_commit()` 设置静态变量 `SyncRepWaitMode`：
```text
remote_write  -> SYNC_REP_WAIT_WRITE
remote_flush  -> SYNC_REP_WAIT_FLUSH
remote_apply  -> SYNC_REP_WAIT_APPLY
其它           -> SYNC_REP_NO_WAIT
```
用户侧 `on` 映射到 `SYNCHRONOUS_COMMIT_REMOTE_FLUSH`，所以最终进入 `SYNC_REP_WAIT_FLUSH`。
`WalSndCtlData` 有三条等待队列：
```text
SyncRepQueue[SYNC_REP_WAIT_WRITE]
SyncRepQueue[SYNC_REP_WAIT_FLUSH]
SyncRepQueue[SYNC_REP_WAIT_APPLY]
```
每条队列都按 `PGPROC.waitLSN` 递增排序。
队列头是最早可以释放的 LSN。
### 4.3 `PGPROC` 保存等待者状态
`src/include/storage/proc.h` 中同步复制等待相关字段是：
```text
waitLSN
syncRepState
syncRepLinks
procLatch
```
字段组合语义如下：

| 字段 | 语义 |
| --- | --- |
| `waitLSN` | backend 正在等待的 commit LSN。 |
| `syncRepState` | `NOT_WAITING`、`WAITING`、`WAIT_COMPLETE`。 |
| `syncRepLinks` | backend 在某条 `SyncRepQueue` 上的链表节点。 |
| `procLatch` | walsender 满足条件后唤醒 backend。 |
raw `waitLSN` 不是语义。
只有同时满足下面条件，backend 才是真的在同步复制等待：
```text
syncRepState == SYNC_REP_WAITING
syncRepLinks 挂在某条 SyncRepQueue
waitLSN 是有效 LSN
backend 在 WaitLatch(... WAIT_EVENT_SYNC_REP)
```
状态写入者也有边界。
owning backend 设置：
```text
NOT_WAITING -> WAITING
WAIT_COMPLETE -> NOT_WAITING
cancel 时删除队列节点
```
walsender 设置：
```text
WAITING -> WAIT_COMPLETE
删除队列节点
SetLatch(procLatch)
```
### 4.4 `WalSnd` 保存 standby 反馈
`src/include/replication/walsender_private.h` 的 `WalSnd` 中，本节关注：
```text
sentPtr
write
flush
apply
writeLag
flushLag
applyLag
sync_standby_priority
replyTime
state
mutex
```
三个位点的含义：

| 字段 | standby 报告的边界 |
| --- | --- |
| `write` | standby 已经写入 WAL 文件。 |
| `flush` | standby 已经 flush WAL 到持久介质。 |
| `apply` | standby startup process 已经 replay 到该 LSN。 |
这些字段由 `walsender.c` 的 `ProcessStandbyReplyMessage()` 更新。
standby 不知道 primary 正在等哪个事务。
standby 只发送自己的三个位点。
primary 根据 wait mode 和 synchronous standby 选择规则解释这些位点。
### 4.5 `WalSndCtlData` 是 shared state 根
`WalSndCtlData` 中本节关注：
```text
SyncRepQueue[NUM_SYNC_REP_WAIT_MODE]
lsn[NUM_SYNC_REP_WAIT_MODE]
sync_standbys_status
walsnds[]
```
`lsn[]` 是每个 wait mode 已经可以释放到的全局 LSN。
语义是：
```text
lsn[WRITE]  当前同步 standby 集合满足 write 的 release LSN
lsn[FLUSH]  当前同步 standby 集合满足 flush 的 release LSN
lsn[APPLY]  当前同步 standby 集合满足 apply 的 release LSN
```
“同步 standby 集合”不是固定列表。
它由 `synchronous_standby_names`、当前 active walsenders、standby priority、quorum 规则和有效 flush LSN 共同决定。
`sync_standbys_status` 包含：
```text
SYNC_STANDBY_INIT
SYNC_STANDBY_DEFINED
```
等待 backend 不能安全 reload config。
所以 checkpointer 用 `SyncRepUpdateSyncStandbysDefined()` 维护这个状态，避免配置清空后 backend 继续按旧认知入队并永久等待。
### 4.6 `SyncRepConfigData` 表达选择规则
`SyncRepConfigData` 是 `synchronous_standby_names` 的 flat 解析结果：
```text
num_sync
syncrep_method
nmembers
member_names[]
```
`syncrep_method` 有两类：
```text
SYNC_REP_PRIORITY  -- FIRST / priority-based
SYNC_REP_QUORUM    -- ANY / quorum-based
```
`member_names[]` 是配置候选名单，不是当前同步 standby。
当前候选还必须满足：
```text
walsender pid 非 0
state 是 STREAMING 或 STOPPING
sync_standby_priority 非 0
flush LSN 有效
```
### 4.7 standby 侧状态不是 primary 等待状态
`walreceiver.c` 中 standby 侧有：
```text
LogstreamResult.Write
LogstreamResult.Flush
WalRcv->writtenUpto
WalRcv->flushedUpto
GetXLogReplayRecPtr()
```
这些状态服务 standby 本地 recovery 和 status reply。
它们不是 primary 上的 `SyncRepQueue`。
primary 只有收到 reply 后，才会更新对应 `WalSnd` slot。
所以：
```text
standby 已经 flush
  不等于
primary 已经释放等待 backend
```
中间还有 walreceiver 发送、网络传输、walsender 读取、`ProcessStandbyReplyMessage()`、`SyncRepReleaseWaiters()` 和 latch 调度。
## 5. 主流程源码 walkthrough
### 5.1 commit path 的关键位置
`CommitTransaction()` 在提交后段执行：
```text
HOLD_INTERRUPTS()
AtEOXact_RelationMap(...)
s->state = TRANS_COMMIT
RecordTransactionCommit()
ProcArrayEndTransaction()
post-commit cleanup
RESUME_INTERRUPTS()
```
关键点是：
```text
RecordTransactionCommit() 发生在 ProcArrayEndTransaction() 之前
```
如果同步复制等待发生，它会卡在这两者之间。
这时事务已经进入本地提交路径，但 backend 还在 ProcArray 中，锁也还没释放。
### 5.2 `RecordTransactionCommit()` 写 commit record
事务有 XID 时，`RecordTransactionCommit()` 会调用：
```text
XactLogCommitRecord(...)
```
该函数写 commit WAL record，并更新 `XactLastRecEnd`。
如果事务没有分配 XID，通常不会写 commit record。
后面是否等待同步复制的判断是：
```text
if (wrote_xlog && markXidCommitted)
    SyncRepWaitForLSN(XactLastRecEnd, true);
```
所以本节讨论的是同时满足：
```text
事务写了 WAL
事务有真正要标记 committed 的 XID
```
只读事务、无 XID 轻量事务、只涉及 temp / unlogged 的事务，不会因为配置了同步复制就凭空等待 standby。
### 5.3 本地提交先于同步复制等待
`RecordTransactionCommit()` 的本地 durable 分支：
```text
if ((wrote_xlog && markXidCommitted &&
     synchronous_commit > SYNCHRONOUS_COMMIT_OFF) ||
    forceSyncCommit || nrels > 0)
{
    XLogFlush(XactLastRecEnd);
    if (markXidCommitted)
        TransactionIdCommitTree(...);
}
else
{
    XLogSetAsyncXactLSN(XactLastRecEnd);
    if (markXidCommitted)
        TransactionIdAsyncCommitTree(..., XactLastRecEnd);
}
```
结论：
```text
synchronous_commit=off:
  commit 可以不等 primary 本地 WAL flush。
synchronous_commit=local 或更高:
  先等 primary 本地 XLogFlush，再标记 pg_xact committed。
remote_write/on/remote_apply:
  在本地 flush 和 pg_xact 标记之后，再进入 SyncRepWaitForLSN()。
```
`SyncRepWaitForLSN()` 不是让本地事务变成 committed 的函数。
它是在本地提交之后，推迟客户端确认和 ProcArray 退出。
### 5.4 commit record 为 remote apply 埋反馈请求
`XactLogCommitRecord()` 中：
```text
if (synchronous_commit >= SYNCHRONOUS_COMMIT_REMOTE_APPLY)
    xl_xinfo.xinfo |= XACT_COMPLETION_APPLY_FEEDBACK;
```
这个 bit 的目的很具体：
```text
standby replay 这个 commit record 后，应尽快让 walreceiver 发 apply reply。
```
standby 的 `xact_redo_commit()` 看到 `XactCompletionApplyFeedback(parsed->xinfo)` 后调用：
```text
XLogRequestWalReceiverReply()
```
recovery 主循环在 `ApplyWalRecord()` 后发现 `doRequestWalReceiverReply`，再调用：
```text
WalRcvRequestApplyReply()
```
walreceiver latch 被唤醒后执行：
```text
XLogWalRcvSendReply(false, false, true)
```
这解释了为什么 `remote_apply` 可以被 commit record replay 及时触发，而不是只能等 `wal_receiver_status_interval` 周期。
### 5.5 `SyncRepWaitForLSN()` fast path
`SyncRepWaitForLSN(lsn, commit)` 首先检查：
```text
!SyncRepRequested()
```
如果当前 `synchronous_commit` 不高于 local flush，直接返回。
它还检查 `WalSndCtl->sync_standbys_status`。
如果 checkpointer 已经初始化该状态，并确认没有 synchronous standby 定义，直接返回。
这是 commit hot path 上的 fast path。
没有同步复制需求时，不能每次 commit 都拿 `SyncRepLock` 或扫描 walsender。
### 5.6 `commit=false` 时 apply 等待被降到 flush
`SyncRepWaitForLSN()` 里：
```text
if (commit)
    mode = SyncRepWaitMode;
else
    mode = Min(SyncRepWaitMode, SYNC_REP_WAIT_FLUSH);
```
原因是源码注释里的边界：
```text
only commit records provide apply feedback
```
非 commit LSN 即使配置 `remote_apply`，也只等 remote flush。
本节主线是 commit，所以主要走 `commit=true`。
但这个 cap 是理解 apply feedback 边界的关键。
### 5.7 入队前必须二次检查
慢路径拿锁：
```text
LWLockAcquire(SyncRepLock, LW_EXCLUSIVE)
```
然后检查：
```text
WalSndCtl->sync_standbys_status
WalSndCtl->lsn[mode]
SyncStandbysDefined()
```
如果：
```text
lsn <= WalSndCtl->lsn[mode]
```
说明 standby reply 已经推进过 release LSN，不需要入队。
这避免 race：
```text
backend 判断需要等
  -> walsender 已经释放到该 LSN
  -> backend 如果盲目入队，会错过唤醒
```
只有仍需要等待时才设置：
```text
MyProc->waitLSN = lsn
MyProc->syncRepState = SYNC_REP_WAITING
SyncRepQueueInsert(mode)
```
### 5.8 队列按 LSN 排序
`SyncRepQueueInsert()` 从队尾向前找插入点。
常见 commit LSN 递增，新 waiter 接近追加到尾部。
队列不变量是：
```text
head: 最小 waitLSN
tail: 最大 waitLSN
```
`SyncRepWakeQueue(false, mode)` 从头扫描。
遇到第一个：
```text
WalSndCtl->lsn[mode] < proc->waitLSN
```
就停止。
所以 standby 每次反馈不需要全队列扫描。
### 5.9 等待 loop 不能普通 ERROR
backend 入队后等待：
```text
WaitLatch(MyLatch,
          WL_LATCH_SET | WL_POSTMASTER_DEATH,
          -1,
          WAIT_EVENT_SYNC_REP)
```
源码注释强调：此时 commit 已经本地发生。
如果这时对客户端报 ERROR 或 FATAL，客户端可能误以为事务 abort。
因此 `QueryCancelPending` 的处理是：
```text
QueryCancelPending = false
WARNING
SyncRepCancelWait()
break
```
`ProcDiePending` 的处理也是 WARNING、取消等待，然后让进程在 cleanup 后死亡。
取消的是“等待 standby 确认”，不是“取消已经本地提交的事务”。
### 5.10 standby write 推进
standby 收到 WAL data message 后，`walreceiver.c` 调用：
```text
XLogWalRcvWrite(buf, nbytes, recptr, tli)
```
写入 WAL segment 后推进：
```text
LogstreamResult.Write = recptr
WalRcv->writtenUpto = LogstreamResult.Write
```
然后唤醒 standby 本地等待者：
```text
WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_WRITE, LogstreamResult.Write)
```
这不是 primary sync rep queue 的唤醒。
primary 要看到 write，必须等 walreceiver 发 status update。
### 5.11 standby flush 推进
`XLogWalRcvFlush()` 在 `LogstreamResult.Flush < LogstreamResult.Write` 时：
```text
issue_xlog_fsync(...)
LogstreamResult.Flush = LogstreamResult.Write
WalRcv->flushedUpto = LogstreamResult.Flush
WakeupRecovery()
XLogWalRcvSendReply(false, false, false)
```
这个 flush 边界对应 `synchronous_commit=on` 的主要远端要求。
但 flush 只说明 standby WAL durable。
它不说明 startup process 已经 replay。
standby 查询是否可见，要看 apply/replay。
### 5.12 apply 位置来自 startup process
walreceiver 发送 reply 时构造：
```text
writePtr = LogstreamResult.Write
flushPtr = LogstreamResult.Flush
applyPtr = GetXLogReplayRecPtr(NULL)
```
`GetXLogReplayRecPtr()` 读取：
```text
XLogRecoveryCtl->lastReplayedEndRecPtr
```
该值由 startup process 在 `ApplyWalRecord()` 成功 redo 后更新。
所以 apply 的 owner 是 startup process。
walreceiver 只是把 startup process 的 replay 位置带回 primary。
### 5.13 apply reply 的链路
remote apply 的反馈链可以写成：
```text
primary commit record carries XACT_COMPLETION_APPLY_FEEDBACK
  -> standby xact_redo_commit()
  -> XLogRequestWalReceiverReply()
  -> recovery loop calls WalRcvRequestApplyReply()
  -> walreceiver wakes
  -> XLogWalRcvSendReply(... checkApply=true)
  -> status update carries applyPtr
```
这条链路只在 commit record 提供 apply feedback 语义时成立。
普通 WAL record 的 replay 不等价于 commit 可见。
### 5.14 walsender 解析 standby reply
primary 上 `ProcessStandbyReplyMessage()` 读取：
```text
writePtr
flushPtr
applyPtr
replyTime
replyRequested
```
然后在 `MyWalSnd->mutex` 下更新：
```text
walsnd->write = writePtr
walsnd->flush = flushPtr
walsnd->apply = applyPtr
walsnd->writeLag = writeLag
walsnd->flushLag = flushLag
walsnd->applyLag = applyLag
walsnd->replyTime = replyTime
```
更新后，如果不是 cascading walsender：
```text
SyncRepReleaseWaiters()
```
`pg_stat_replication` 的 `write_lsn`、`flush_lsn`、`replay_lsn`、lag 和 `reply_time` 都来自这些 shared fields。
### 5.15 release 前先判断 walsender 资格
`SyncRepReleaseWaiters()` 快速退出条件包括：
```text
sync_standby_priority == 0
state 不是 STREAMING 或 STOPPING
flush LSN 无效
```
这说明 standby 发来 write 还不够。
它必须至少是 synchronous standby 名单候选，并且有有效 flush 位置。
有些复制连接能出现在 `pg_stat_replication`，但不会释放 sync commit。
### 5.16 选择当前同步 standby
`SyncRepReleaseWaiters()` 拿 `SyncRepLock` 后调用：
```text
SyncRepGetSyncRecPtr(&writePtr, &flushPtr, &applyPtr, &am_sync)
```
内部先调用：
```text
SyncRepGetCandidateStandbys(&sync_standbys)
```
候选必须：
```text
pid active
state STREAMING 或 STOPPING
sync_standby_priority 非 0
flush LSN valid
```
如果候选数量少于 `SyncRepConfig->num_sync`，返回 false。
如果当前 walsender 不属于候选集合，也不会由它释放等待者。
### 5.17 priority 模式取最慢 selected standby
`FIRST n (...)` 使用 `SYNC_REP_PRIORITY`。
`SyncRepGetCandidateStandbys()` 在 priority 模式下按 priority 排序，只保留前 `num_sync` 个。
`SyncRepGetOldestSyncRecPtr()` 对这些 selected standby 取：
```text
writePtr = min(selected.write)
flushPtr = min(selected.flush)
applyPtr = min(selected.apply)
```
所以 priority 模式被 selected standby 中最慢的位置限制。
如果最高优先级 standby 断开，下一个 priority standby 可以接替。
但它必须已经是有效候选，并推进到足够 LSN。
### 5.18 quorum 模式取第 N 快位置
`ANY n (...)` 使用 `SYNC_REP_QUORUM`。
所有匹配名单的候选都进入集合。
`SyncRepGetNthLatestSyncRecPtr()` 分别对三组 LSN 降序排序：
```text
write_array
flush_array
apply_array
```
然后取：
```text
array[nth - 1]
```
所以 `ANY 2 (s1, s2, s3)` 等候选中第 2 快的 write / flush / apply 位置。
它不是固定绑定某两个 standby。
### 5.19 release LSN 先推进再唤醒
计算完成后：
```text
WalSndCtl->lsn[WRITE] = writePtr
SyncRepWakeQueue(false, WRITE)
WalSndCtl->lsn[FLUSH] = flushPtr
SyncRepWakeQueue(false, FLUSH)
WalSndCtl->lsn[APPLY] = applyPtr
SyncRepWakeQueue(false, APPLY)
```
顺序不能反。
等待者被唤醒后可能立即继续运行。
如果 release LSN 还没更新，会造成状态解释不一致。
### 5.20 `SyncRepWakeQueue()` 的跨进程协议
`SyncRepWakeQueue()` 对每个可释放 backend 执行：
```text
dlist_delete_thoroughly(&proc->syncRepLinks)
pg_write_barrier()
proc->syncRepState = SYNC_REP_WAIT_COMPLETE
SetLatch(&proc->procLatch)
```
barrier 的原因是等待 backend 不拿 `SyncRepLock` 读取 `syncRepState`。
walsender 必须保证 backend 看到：
```text
先从队列删除
再看到 WAIT_COMPLETE
```
backend 退出 loop 后：
```text
pg_read_barrier()
Assert(syncRepLinks detached)
syncRepState = NOT_WAITING
waitLSN = InvalidXLogRecPtr
```
这是本节最小的 shared-memory 状态机。
## 6. 同步 standby 选择边界
### 6.1 名字匹配靠 `application_name`
`SyncRepGetStandbyPriority()` 用当前 walsender 的 `application_name` 匹配 `synchronous_standby_names`。
也支持 wildcard `*`。
常见配置错误：
```text
standby 连接正常
pg_stat_replication 有行
application_name 不匹配
sync_priority = 0
sync_state = async
commit 不把它当同步 standby
```
如果配置 `*`，新的 standby 可能自动成为候选，也可能改变 commit latency。
### 6.2 cascading walsender 不参与
`SyncRepGetStandbyPriority()` 中：
```text
if (am_cascading_walsender)
    return 0;
```
本节讨论的是 primary 等直接连接 standby 的反馈。
级联下游的可见性不被 primary 本次 sync commit 直接保证。
### 6.3 `sync_priority > 0` 不是充分条件
`sync_standby_priority != 0` 只说明 standby 在候选名单内。
它还要：
```text
pid active
state STREAMING 或 STOPPING
flush LSN valid
priority 模式下排进前 num_sync
quorum 模式下贡献到第 N 快 LSN
```
诊断时要同时看 `sync_state`、`state`、`write_lsn`、`flush_lsn`、`replay_lsn`、`reply_time`。
### 6.4 `STOPPING` 仍可能释放
candidate 条件允许：
```text
WALSNDSTATE_STREAMING
WALSNDSTATE_STOPPING
```
连接关闭边缘，如果 walsender 已经收到足够反馈，仍可释放等待者。
但它不能确认未来没有收到的 LSN。
如果候选不足，后续等待会继续卡住。
### 6.5 candidate 必须有有效 flush
即使等待模式是 `remote_write`，候选 standby 也要求 flush LSN valid。
flush valid 是“这个连接是可参与同步复制决策的 streaming standby”的基本门槛。
没有有效 flush 的连接可能能显示在统计视图中，但不能释放 sync commit。
## 7. durability 与 visibility 边界
### 7.1 local 边界
`synchronous_commit=off` 时，commit 可在 WAL writer 后续 flush 前确认。
postmaster crash 可能丢最后一段 async commit。
`local` 及更高等级在 `RecordTransactionCommit()` 中执行 `XLogFlush(XactLastRecEnd)`，给出 primary 本地 crash safety。
但 `local` 不等待 standby。
如果之后发生 failover，新主是否包含该事务取决于 standby 是否已经收到并保留足够 WAL。
### 7.2 remote write 边界
`remote_write` 等待：
```text
write_lsn >= commit LSN
```
standby 已经把 WAL bytes 写入本地 WAL 文件。
它比“primary 已发送”更强。
但它没有要求 standby fsync。
如果 standby OS 或存储在 flush 前崩溃，这段 WAL 可能丢失。
### 7.3 remote flush / `on` 边界
`on` 等待：
```text
flush_lsn >= commit LSN
```
standby 已经 fsync WAL。
这给出同步 standby 上的 WAL durability。
但 startup process 可能尚未 replay 到 commit record。
所以 `on` 不保证 standby 查询已经可见。
### 7.4 remote apply 边界
`remote_apply` 等待：
```text
replay_lsn >= commit LSN
```
standby startup process 已经 replay commit record。
在 Hot Standby 已经 consistent 且新查询获取新 snapshot 的情况下，后续只读查询可以看到该事务。
但已经存在的 standby snapshot 不会因为 remote apply 返回而改变。
remote apply 保证的是 commit record 已进入 standby 已重放历史，而不是刷新所有已打开查询。
### 7.5 primary 本地可见性也被推迟
`RecordTransactionCommit()` 注释说：
```text
at this stage we have marked clog,
but still show as running in the procarray and continue to hold locks.
```
同步复制等待发生在：
```text
TransactionIdCommitTree()
  之后
ProcArrayEndTransaction()
  之前
```
因此 primary 上：
```text
pg_xact 可能已经 committed
但其它 backend 的 snapshot 仍会把该 XID 视为 running
锁也还没有释放
```
所以慢 standby 不只拖慢客户端 COMMIT 返回。
它也会拖慢 primary 上其它事务看到这个提交和获得相关锁。
### 7.6 ack 不是全局线性化
`remote_apply` 返回不代表所有 standby 都可见。
真实语义是：
```text
当前 synchronous standby selection 规则要求的 standby 数量
已经对这个 commit LSN 达到 apply 边界
```
`ANY 1 (s1, s2, s3)` 只要求一个候选达到目标。
`FIRST 1 (s1, s2)` 通常等 s1，s1 不合格时才可能由 s2 接替。
## 8. 生命周期、异常路径与 fallback
### 8.1 等待记录的 owner
同步复制等待没有单独对象。
它复用提交 backend 的 `PGPROC`。
创建者是提交 backend。
正常释放者是 walsender。
异常释放者可能是 owning backend、proc exit cleanup 或 checkpointer。
`SyncRepCleanupAtProcExit()` 会在 backend exit 时删除残留队列节点。
它清理的是 shared wait queue，不是回滚事务。
### 8.2 没有同步复制需求时 fast return
条件：
```text
synchronous_commit <= local
max_wal_senders == 0
sync_standbys_status 显示没有 synchronous standby 定义
```
结果：
```text
SyncRepWaitForLSN() 不入队
commit latency 只包含本地提交需要的部分
```
这是配置语义，不是异常降级。
### 8.3 配置了同步 standby 但候选不足
如果 `synchronous_standby_names` 非空，但没有足够 candidate standby：
```text
backend 可以入队
SyncRepReleaseWaiters() 因 got_recptr=false 不释放
```
PostgreSQL 不会自动异步降级。
能改变结果的事件是：
```text
standby 连接并 catch up
候选 standby 接替
synchronous_standby_names 被清空或降低要求
backend wait 被 cancel / terminate
postmaster death
```
### 8.4 standby 断开
当前 synchronous standby 断开后，对应 walsender 不再提供未来反馈。
priority 模式下，下一个 priority standby 可能接替。
接替发生在 `SyncRepGetCandidateStandbys()` 每次重新扫描 `WalSndCtl->walsnds[]` 时。
如果没有足够候选，等待者继续等。
已经满足 release LSN 的等待者可能在 `STOPPING` walsender 边缘被释放。
### 8.5 query cancel
等待期间收到 cancel：
```text
QueryCancelPending = false
WARNING "canceling wait for synchronous replication due to user request"
SyncRepCancelWait()
```
客户端看到的是 warning，不是事务回滚。
事务已经本地提交，后续会离开 ProcArray 并释放锁。
### 8.6 terminate connection
等待期间收到 terminate：
```text
WARNING "canceling the wait for synchronous replication and terminating connection"
whereToSendOutput = DestNone
SyncRepCancelWait()
```
`ProcDiePending` 不会被清掉。
进程会在 commit cleanup 后死亡。
这同时避免两个错误：不能告诉客户端事务 abort，也不能假装同步复制已经成功。
### 8.7 postmaster death
`WaitLatch()` 包含 `WL_POSTMASTER_DEATH`。
postmaster 死亡时：
```text
ProcDiePending = true
whereToSendOutput = DestNone
SyncRepCancelWait()
```
walsender 也会退出，继续等待 standby ack 没有意义。
### 8.8 配置清空唤醒所有队列
`synchronous_standby_names` 变空时，checkpointer 调用 `SyncRepUpdateSyncStandbysDefined()`：
```text
SyncRepWakeQueue(true, WRITE)
SyncRepWakeQueue(true, FLUSH)
SyncRepWakeQueue(true, APPLY)
WalSndCtl->sync_standbys_status = SYNC_STANDBY_INIT
```
`all=true` 不看 LSN。
这不是 standby 达到目标，而是用户取消了等待语义。
### 8.9 reload 改变 release 条件
walsender 处理 SIGHUP 时：
```text
ProcessConfigFile(PGC_SIGHUP)
SyncRepInitConfig()
SyncRepReleaseWaiters()
```
如果降低 `num_sync` 或改变名单让已有反馈满足新规则，等待者可能被释放。
如果提高要求，未来 commit 可能等待更久。
## 9. 成本、资源与跨模块传播
同步 commit 的额外 latency 来自：
```text
primary 本地 XLogFlush
WAL sender 发送和网络传输
standby WAL write
standby WAL fsync
standby replay
standby status reply
walsender 处理 reply
SyncRepLock 和队列唤醒
backend 被调度继续运行
```
`remote_write` 不强制等 standby fsync 和 replay。
`on` 不强制等 replay。
`remote_apply` 把 startup process replay 延迟也放进 primary commit path。
队列成本随等待 backend 数增长。
`SyncRepQueueInsert()` 通常接近尾插，但乱序时会从尾向前扫描。
`SyncRepWakeQueue()` 从头释放所有已满足 backend，一次大 feedback 可能唤醒很多 backend。
quorum 模式还有额外排序成本。
`SyncRepGetNthLatestSyncRecPtr()` 会为 write / flush / apply 分别分配数组并 `qsort()`，成本随候选 standby 数增长。
真正的主导延迟通常仍在网络、standby WAL fsync 和 replay。
`remote_apply` 会把 standby recovery 资源传播到 primary：
```text
standby 数据盘慢
standby WAL 盘慢
standby replay CPU 慢
Hot Standby conflict 延迟 recovery
recovery paused
网络 reply 延迟
```
因为等待发生在 `ProcArrayEndTransaction()` 前，慢 standby 还会延迟 primary 本地锁释放和可见性。
这会把复制问题扩散成 primary 上的并发等待问题。
## 10. 观测与诊断入口
### 10.1 primary 上先看谁在等
核心查询：
```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event = 'SyncRep';
```
`wait_event_type='IPC'` 的 `SyncRep` 对应：
```text
WaitLatch(... WAIT_EVENT_SYNC_REP)
```
也就是 backend 等 standby confirmation。
同名 `SyncRep` 也可能作为 LWLock wait event 出现。
那表示竞争 `SyncRepLock`，不是等待 standby ack。
所以诊断时必须同时看 `wait_event_type`。
### 10.2 primary 上看 standby 分层
`pg_stat_replication` 暴露：
```text
sent_lsn
write_lsn
flush_lsn
replay_lsn
write_lag
flush_lag
replay_lag
sync_priority
sync_state
reply_time
```
分层判断：
```text
sent_lsn 不动:
  primary WAL sender 或 WAL flush/source 卡住。
write_lsn 落后 sent_lsn:
  网络、walreceiver receive、standby WAL write 可能慢。
flush_lsn 落后 write_lsn:
  standby WAL fsync 慢。
replay_lsn 落后 flush_lsn:
  startup process replay 慢、pause、conflict 或 redo 成本高。
sync_state = async:
  不释放 synchronous commit。
sync_state = potential:
  priority 模式候补，不是当前 selected。
sync_state = sync:
  priority 模式当前 selected。
sync_state = quorum:
  quorum 模式候选，是否满足取决于第 N 快 LSN。
```
### 10.3 standby 上看 walreceiver
standby 的 `pg_stat_wal_receiver` 暴露：
```text
written_lsn
flushed_lsn
latest_end_lsn
last_msg_send_time
last_msg_receipt_time
sender_host
sender_port
```
它能说明 standby 是否还在接收 primary WAL。
它不能直接说明 primary 上哪个 backend 在 sync rep queue。
需要和 primary 的 `pg_stat_activity`、`pg_stat_replication` 对照。
### 10.4 standby 上看 replay
standby 上：
```sql
SELECT pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp();
```
如果 `pg_stat_wal_receiver.flushed_lsn` 前进，但 `pg_last_wal_replay_lsn()` 不动，卡点在 replay。
这对 `remote_apply` 至关重要。
如果只配置 `on`，replay 落后不一定阻塞 primary commit。
### 10.5 现象到源码的映射

| 现象 | 更可能的卡点 | 回到源码 |
| --- | --- | --- |
| backend `wait_event_type=IPC` 且 `wait_event=SyncRep` | 正在等 standby ack | `SyncRepWaitForLSN()` wait loop |
| `write_lsn` 未到 commit LSN | standby 未报告 write | `XLogWalRcvWrite()`、`XLogWalRcvSendReply()` |
| `write_lsn` 到了，`flush_lsn` 未到 | standby fsync 慢 | `XLogWalRcvFlush()` |
| `flush_lsn` 到了，`replay_lsn` 未到，GUC 是 `remote_apply` | standby replay 慢 | `ApplyWalRecord()`、`xact_redo_commit()` |
| `sync_priority=0` | 名字不匹配或级联 walsender | `SyncRepGetStandbyPriority()` |
| `sync_state=potential` | priority 模式候补 | `SyncRepGetCandidateStandbys()` priority sort |
| `sync_state=quorum` | quorum 候选 | `SyncRepGetNthLatestSyncRecPtr()` |
| 所有 standby 断开但 commit 仍等 | 配置仍要求同步 | `SyncRepReleaseWaiters()` 候选不足 |
### 10.6 gdb / 临时 instrumentation
课堂调试可以看：
```text
p MyProc->waitLSN
p MyProc->syncRepState
p WalSndCtl->lsn[0]
p WalSndCtl->lsn[1]
p WalSndCtl->lsn[2]
p MyWalSnd->write
p MyWalSnd->flush
p MyWalSnd->apply
```
临时日志点：
```text
SyncRepWaitForLSN(): lsn, mode
ProcessStandbyReplyMessage(): writePtr, flushPtr, applyPtr
SyncRepReleaseWaiters(): computed writePtr, flushPtr, applyPtr
SyncRepWakeQueue(): proc->waitLSN, mode
```
这些适合课堂实验，不应保留在产品代码中。
### 10.7 日志和进程标题
`SyncRepWaitForLSN()` 在 `update_process_title` 开启时设置：
```text
waiting for %X/%08X
```
`SyncRepReleaseWaiters()` 有 DEBUG3 日志，显示释放到的 write / flush / apply LSN。
standby 变为 synchronous standby 时会 LOG：
```text
standby "..." is now a synchronous standby with priority ...
standby "..." is now a candidate for quorum synchronous standby
```
这些日志有助于确认 standby selection 变化。
默认日志级别不一定显示 DEBUG 细节。
### 10.8 观测盲区
`pg_stat_replication` 看不到：
```text
每个等待 backend 的 waitLSN
每个 backend 等待的 wait mode
SyncRepQueue 的完整长度和排序
某次 commit 被哪一次 standby reply 释放
```
这些只能通过 wait event、LSN 差距、日志、gdb、临时 instrumentation 和业务 commit latency 近似推理。
不要把单个统计字段解释成完整因果。
## 11. 常见误区
1. 误以为 `synchronous_commit=on` 等 standby apply。`on` 对应 remote flush，不是 replay。
2. 误以为 `remote_write` 已经 standby crash-safe。它只等 standby write，不等 standby fsync。
3. 误以为 primary 上事务在 sync rep wait 期间已对其它事务完全可见。实际上 backend 仍在 ProcArray 中并持锁。
4. 误以为 standby 连接存在就能释放同步提交。它还要匹配名字、状态、priority/quorum 规则，并有有效 flush LSN。
5. 误以为 cancel COMMIT 等待会回滚事务。源码只发 WARNING，因为事务已经本地提交。
6. 误以为 lag 字段是每个事务的精确等待时间。`write_lag` / `flush_lag` / `replay_lag` 是基于采样和 reply 的近似值。
7. 误以为 quorum 固定等某几个 standby。`ANY n` 等的是候选中第 N 快位置，具体 standby 可以变。
8. 误以为清空 `synchronous_standby_names` 和 standby 达到 LSN 是同一种释放。前者取消等待语义，后者满足等待语义。
## 12. 课堂实验
### 12.1 实验一：观察 `on` 等 flush
primary 配置：
```text
synchronous_standby_names = 'FIRST 1 (s1)'
```
standby 连接设置：
```text
application_name=s1
```
primary 会话：
```sql
SET synchronous_commit = on;
CREATE TABLE sync_rep_demo(id int primary key);
INSERT INTO sync_rep_demo VALUES (1);
```
观察：
```sql
SELECT application_name, state, sent_lsn, write_lsn, flush_lsn,
       replay_lsn, sync_priority, sync_state
FROM pg_stat_replication;
```
源码对应：
```text
RecordTransactionCommit()
  -> XLogFlush()
  -> TransactionIdCommitTree()
  -> SyncRepWaitForLSN(..., commit=true)
  -> mode = SYNC_REP_WAIT_FLUSH
```
预期：COMMIT 返回需要 selected standby 的 `flush_lsn` 达到 commit LSN；`replay_lsn` 可以短暂落后。
### 12.2 实验二：让 `remote_apply` 卡在 replay
standby 暂停 replay：
```sql
SELECT pg_wal_replay_pause();
```
primary 会话：
```sql
BEGIN;
SET LOCAL synchronous_commit = remote_apply;
INSERT INTO sync_rep_demo VALUES (2);
COMMIT;
```
另一个 primary 会话观察：
```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event = 'SyncRep';
SELECT application_name, write_lsn, flush_lsn, replay_lsn, sync_state
FROM pg_stat_replication;
```
预期：
```text
write_lsn / flush_lsn 可能前进
replay_lsn 停住
COMMIT backend 等待 SyncRep
```
恢复：
```sql
SELECT pg_wal_replay_resume();
```
源码对应：`remote_apply -> SYNC_REP_WAIT_APPLY`，replay paused 导致 `GetXLogReplayRecPtr()` 不前进，`WalSndCtl->lsn[APPLY]` 不足以唤醒队列。
### 12.3 实验三：验证 cancel 不回滚事务
在 standby replay 暂停时，primary 会话 A：
```sql
BEGIN;
SET LOCAL synchronous_commit = remote_apply;
INSERT INTO sync_rep_demo VALUES (3);
COMMIT;
```
另一个会话取消 A：
```sql
SELECT pg_cancel_backend(<pid_of_A>);
```
观察 A 收到 WARNING。
恢复 standby replay 后，在 primary 查询：
```sql
SELECT * FROM sync_rep_demo WHERE id = 3;
```
源码对应：
```text
QueryCancelPending
  -> WARNING
  -> SyncRepCancelWait()
  -> 不抛 ERROR
```
重点：取消的是同步复制等待，不是撤销本地 commit。
### 12.4 实验四：priority 接替
配置：
```text
synchronous_standby_names = 'FIRST 1 (s1, s2)'
```
两个 standby 分别设置 `application_name=s1` 和 `application_name=s2`。
观察：
```sql
SELECT application_name, sync_priority, sync_state
FROM pg_stat_replication
ORDER BY sync_priority;
```
停止 s1，观察 s2 是否从 `potential` 变成 `sync`。
源码对应：
```text
SyncRepGetCandidateStandbys()
  -> sort by standby_priority
  -> keep first num_sync
```
重点：接替发生在 release 计算时，不是事务入队时固定绑定某个 standby。
### 12.5 实验五：quorum 第 N 快 LSN
配置：
```text
synchronous_standby_names = 'ANY 2 (s1, s2, s3)'
```
制造一个慢 standby。
观察：
```sql
SELECT application_name, write_lsn, flush_lsn, replay_lsn, sync_state
FROM pg_stat_replication
ORDER BY application_name;
```
预期：三个候选都可能显示 `quorum`；commit 等第 2 快的目标 LSN；最慢那个不一定阻塞。
源码对应：
```text
SyncRepGetNthLatestSyncRecPtr()
  -> qsort write_array
  -> qsort flush_array
  -> qsort apply_array
  -> take nth - 1
```
### 12.6 实验六：源码断点
建议断点：
```text
RecordTransactionCommit
SyncRepWaitForLSN
ProcessStandbyReplyMessage
SyncRepReleaseWaiters
SyncRepWakeQueue
XLogWalRcvSendReply
xact_redo_commit
WalRcvRequestApplyReply
```
画出状态表：

| 时刻 | primary backend | walsender | standby walreceiver | standby startup |
| --- | --- | --- | --- | --- |
| commit record inserted | `XactLastRecEnd` valid | not relevant | no data yet | no replay |
| local flush done | `TransactionIdCommitTree` done | may send WAL | no data yet | no replay |
| wait entered | `PGPROC.waitLSN` set | waits for reply | maybe receiving | maybe idle |
| standby write | waiting | no release until reply | `LogstreamResult.Write` advanced | maybe not replayed |
| standby flush | waiting | receives flush reply | `LogstreamResult.Flush` advanced | woken |
| standby apply | waiting if remote_apply | receives apply reply | sends applyPtr | `lastReplayedEndRecPtr` advanced |
| release | `WAIT_COMPLETE` | `SetLatch` | done | done |
| local visibility | `ProcArrayEndTransaction` | not involved | not involved | not involved |
## 13. 讨论题
1. 为什么 `SyncRepWaitForLSN()` 必须在本地 commit record flush 和 pg_xact 标记之后，而不是之前？
2. 为什么等待期间收到 cancel 只能 WARNING，而不能 ERROR？
3. `remote_write`、`on`、`remote_apply` 分别牺牲哪些 latency，换来哪些 durability 或 visibility 语义？
4. 为什么 `remote_apply` 需要 commit record 上的 `XACT_COMPLETION_APPLY_FEEDBACK`？
5. `ANY 2 (s1, s2, s3)` 下，为什么 `sync_state=quorum` 不代表三个 standby 都必须达到 commit LSN？
6. 如果 primary 上大量 backend 等待 `SyncRep`，但 `pg_stat_replication.flush_lsn` 已经追上，下一步应该检查 replay、sync_state、还是 wait_event_type？
7. 为什么清空 `synchronous_standby_names` 必须在 `SyncRepUpdateSyncStandbysDefined()` 中唤醒所有队列？
8. 在 priority 模式下，为什么一个 `sync_priority > 0` 的 standby 仍可能不释放等待者？
## 14. 本节小结
本节核心链路：
```text
RecordTransactionCommit()
  -> XLogFlush() / TransactionIdCommitTree()
  -> SyncRepWaitForLSN(XactLastRecEnd, true)
  -> standby status update carries write / flush / apply
  -> ProcessStandbyReplyMessage()
  -> SyncRepReleaseWaiters()
  -> SyncRepWakeQueue()
  -> backend exits wait
  -> ProcArrayEndTransaction()
```
核心状态分三层：
```text
backend PGPROC:
  waitLSN, syncRepState, syncRepLinks
primary WalSnd:
  per-standby write, flush, apply, priority, replyTime
WalSndCtl:
  release lsn per wait mode, wait queues, sync_standbys_status
```
standby 只负责报告位置。
primary 负责解释这些位置是否满足当前事务的同步提交语义。
`synchronous_commit` 决定等 write、flush 还是 apply。
`synchronous_standby_names` 决定哪些 standby、多少 standby 的反馈能释放等待。
priority 模式等待 selected standby 中最慢位置。
quorum 模式等待候选中第 N 快位置。
durability 和 visibility 边界必须分开：
```text
remote_write:
  standby 已写 WAL，不保证 standby fsync 或 replay。
on / remote flush:
  standby 已 flush WAL，不保证 standby 查询可见。
remote_apply:
  standby 已 replay commit record，后续 standby snapshot 可见，但旧 snapshot 不会变化。
```
primary 本地也有可见性延迟：
```text
sync rep wait 发生在 pg_xact commit 之后，
但在 ProcArrayEndTransaction 和锁释放之前。
```
异常路径的基本规律：
```text
事务已经本地提交后，不能再把等待失败伪装成事务 abort。
```
所以 cancel / terminate / postmaster death 都是取消等待并发 warning 或断开，不是回滚事务。
配置清空则通过 `SyncRepUpdateSyncStandbysDefined()` 唤醒所有等待者，因为等待语义本身被取消。
诊断时先问：
```text
当前等待的是哪个边界？
这个边界由哪个进程推进？
反馈字段在哪里写入？
等待者在哪里被唤醒？
取消等待后事务是否已经越过不可回退点？
```
可迁移规律：
```text
跨节点 commit ack 不是一个布尔状态；
它是本地持久化、远端接收、远端持久化、远端可见、客户端确认、本地 ProcArray 退出
这些状态按源码顺序组合出来的边界。
```
