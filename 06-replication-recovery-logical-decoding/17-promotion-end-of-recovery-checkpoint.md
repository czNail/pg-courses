# PostgreSQL promotion、end-of-recovery checkpoint 与新历史起点
## 课程定位
前置知识：已经理解 startup process 如何做 WAL replay，知道 Hot Standby 依赖 `SharedRecoveryState`、`SharedHotStandbyActive`、walreceiver 和 postmaster 状态机协作。
本节唯一主问题：
```text
standby promotion 时 startup process 如何结束 recovery、写 end-of-recovery record/checkpoint，
并把只读恢复实例切换成可写 primary？
```
核心矛盾：promotion 要尽快让业务恢复写入能力；但 standby 正在读旧 timeline 的 WAL，walreceiver 可能仍在写入同一段 WAL，control file 仍表示 recovery，普通 backend 仍不能插入 WAL，级联 standby 还需要知道新历史从哪里分叉。
PostgreSQL 的解法不是“把一个 in_recovery 布尔值改成 false”。
它把 promotion 拆成一条严格顺序：
```text
promotion trigger
  -> startup process 停止 replay 和 walreceiver
  -> 选出新 timeline
  -> 写 timeline history
  -> 初始化 WAL insertion 边界
  -> 写 XLOG_END_OF_RECOVERY 或 end-of-recovery checkpoint
  -> 更新 control file 和 SharedRecoveryState
  -> startup process 退出
  -> postmaster 切到 PM_RUN 并允许普通可写连接
```
学完后应能判断：为什么 promotion 必须先停 walreceiver；为什么新 primary 不能继续写旧 timeline；为什么 `pg_promote(wait => true)` 返回不等于“checkpoint 已经完成”；为什么 `pg_is_in_recovery()` 变为 false 的真正边界是 `RECOVERY_STATE_DONE`。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
上一节 timeline 课程解释了：
```text
同一个 WAL segment name 里的 TLI 不是装饰字段；
promotion 会让历史分叉；
.history 文件记录 parent timeline、switchpoint 和分叉原因。
```
这一节把这个抽象落到 startup process 的 runtime。
standby 已经处在 recovery 中。
它可能处于 `PM_RECOVERY`。
它也可能已经进入 `PM_HOT_STANDBY`，正在接受只读查询。
walreceiver 可能正在从上游读取 WAL。
startup process 可能正在：
```text
ReadRecord()
  -> WaitForWALToBecomeAvailable()
  -> ApplyWalRecord()
```
promotion 的问题是：
```text
在这一套 recovery machinery 仍然活着时，
如何安全地停止消费旧历史，
然后让本节点从一个确定的 LSN 开始写自己的新历史？
```
本节不再讲每条 WAL record 如何 redo。
本节也不展开 follow timeline 和级联 standby 如何追新 timeline。
这些是第 18 课的问题。
本节只追踪 promotion 的本地结束恢复链路。
最短调用主线是：
```text
pg_promote()
  -> postmaster SIGUSR1
  -> postmaster 检查 promote 文件并 SIGUSR2 startup process
  -> StartupProcTriggerHandler()
  -> CheckForStandbyTrigger()
  -> PerformWalRecovery() 退出 replay loop
  -> FinishWalRecovery()
  -> StartupXLOG() end-of-recovery actions
  -> SharedRecoveryState = RECOVERY_STATE_DONE
  -> startup process proc_exit(0)
  -> postmaster UpdatePMState(PM_RUN)
```
这条链路连接了 SQL 函数、文件系统信号、postmaster 状态机、WAL recovery、timeline、control file、checkpoint 和普通 backend 写入权限。
如果只读某一个文件，很容易误判 promotion 的完成边界。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
promotion 请求先被物化为 PGDATA 下的 promote 文件和 startup process 的 SIGUSR2；
startup process 在 recovery wait/replay 边界看到请求后设置 SharedPromoteIsTriggered，
停止 walreceiver，关闭 standby mode，重新确认 end-of-log，
选择新 timeline 并写 .history 文件，
再用 XLOG_END_OF_RECOVERY 或 end-of-recovery checkpoint 标记分叉点，
最后在 ControlFileLock 下把 pg_control 状态改为 DB_IN_PRODUCTION，
并把 XLogCtl->SharedRecoveryState 改为 RECOVERY_STATE_DONE。
```
这段话里有三个不同层次。
第一层是 promotion 请求：
```text
promote file
postmaster signal
startup process local flag
XLogRecoveryCtl->SharedPromoteIsTriggered
```
第二层是 WAL 历史切换：
```text
last replayed record
endOfLog
newTLI
ThisTimeLineID
PrevTimeLineID
timeline history file
XLOG_END_OF_RECOVERY / XLOG_CHECKPOINT_SHUTDOWN
```
第三层是实例角色切换：
```text
ControlFile->state = DB_IN_PRODUCTION
XLogCtl->SharedRecoveryState = RECOVERY_STATE_DONE
postmaster pmState = PM_RUN
connsAllowed = true
```
这三个层次不能互相替代。
`.history` 文件出现，不代表普通 backend 已经可以写。
walreceiver 退出，不代表 control file 已经进入 production。
`pg_is_in_recovery()` 为 false，不保证后续强制 checkpoint 已经结束。
promotion 的 correctness 来自这三层按顺序收敛。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xlogfuncs.c` | `pg_promote()` 如何创建 promote 文件、通知 postmaster、等待 `RecoveryInProgress()` 变 false。 |
| 2 | `src/backend/postmaster/postmaster.c` | `process_pm_pmsignal()` 如何把 promote 文件转成 `SIGUSR2`，startup 退出后如何进入 `PM_RUN`。 |
| 3 | `src/backend/postmaster/startup.c` | `StartupProcessMain()`、`StartupProcTriggerHandler()`、`promote_signaled`。 |
| 4 | `src/backend/access/transam/xlogrecovery.c` | `CheckForStandbyTrigger()`、`PromoteIsTriggered()`、`FinishWalRecovery()`、`ShutdownWalRecovery()`。 |
| 5 | `src/backend/access/transam/xlog.c` | `StartupXLOG()` end-of-recovery 主流程、`PerformRecoveryXLogAction()`、`CreateEndOfRecoveryRecord()`、`CreateCheckPoint()`、`RecoveryInProgress()`。 |
| 6 | `src/backend/access/transam/timeline.c` | `writeTimeLineHistory()` 如何写新 timeline history 并通知归档。 |
| 7 | `src/backend/replication/walreceiverfuncs.c` | `ShutdownWalRcv()` 如何停止 walreceiver 并等待 `WALRCV_STOPPED`。 |
| 8 | `src/include/access/xlogrecovery.h` | `XLogRecoveryCtlData`、`EndOfWalRecoveryInfo`、promotion/replay 共享状态。 |
| 9 | `src/include/access/xlog.h` | `RecoveryState`、signal 文件名、checkpoint flag、WAL/recovery 外部入口。 |
推荐阅读顺序按状态推进，而不是按文件名。
先看 `pg_promote()` 的外部触发。
再看 postmaster 如何把文件信号转给 startup process。
然后跳到 `CheckForStandbyTrigger()`，确认请求何时被 startup process 接受。
最后读 `StartupXLOG()` 的后半段。
那里才是真正把 recovery 变成 primary 的地方。
## 4. 关键状态：请求、历史和角色不是同一个东西
本节最容易混淆的状态有三组。
第一组是 promotion 请求状态。
`PROMOTE_SIGNAL_FILE` 定义在 `src/include/access/xlog.h`，文件名是：
```text
promote
```
`pg_promote()` 创建这个文件。
postmaster 看到后向 startup process 发送 `SIGUSR2`。
startup process 的 `StartupProcTriggerHandler()` 只做一件很小的事：
```text
promote_signaled = true
```
这只是信号处理器安全的本地标志。
真正把请求发布给其他进程的是 `SetPromoteIsTriggered()`。
它写：
```text
XLogRecoveryCtl->SharedPromoteIsTriggered = true
```
这解释了 `pg_stat_get_recovery()` 中 `promote_triggered` 的来源。
第二组是 timeline 状态。
`XLogCtl->InsertTimeLineID` 是接下来要写 WAL 的 timeline。
`XLogCtl->PrevTimeLineID` 是本次分叉前的 timeline。
`xl_end_of_recovery` record 中也有：
```text
ThisTimeLineID
PrevTimeLineID
```
`CheckPoint` record 中也有同名字段。
这些字段的语义不是“当前文件名里看到的 TLI”。
它们表示：
```text
这条 WAL record 属于哪个新 timeline；
如果它开始了 timeline switch，它从哪个旧 timeline 分叉。
```
第三组是实例角色状态。
`ControlFile->state` 是 pg_control 中的 durable cluster state。
promotion 结束时它必须变成：
```text
DB_IN_PRODUCTION
```
`XLogCtl->SharedRecoveryState` 是共享内存里给普通进程读的 recovery state。
promotion 结束时它必须变成：
```text
RECOVERY_STATE_DONE
```
`postmaster` 的 `pmState` 是进程管理状态。
promotion 完成后 startup process 正常退出，postmaster 把它切成：
```text
PM_RUN
```
这三者各自服务不同读者。
control file 服务 crash 后的下一次启动。
`SharedRecoveryState` 服务当前共享内存里的 backend。
`pmState` 服务 postmaster 是否接受普通连接和启动哪些后台进程。
## 5. promotion trigger：从 SQL 请求到 startup process 本地事实
外部最常用入口是：
```sql
select pg_promote(wait => true, wait_seconds => 60);
```
源码入口在 `xlogfuncs.c` 的 `pg_promote()`。
函数第一步检查：
```text
RecoveryInProgress()
```
如果当前不是 recovery，就报错：
```text
recovery is not in progress
```
这不是权限或 timeline 错误。
这是语义错误：primary 已经不需要 promotion。
通过检查后，`pg_promote()` 创建 `PROMOTE_SIGNAL_FILE`。
然后它向 `PostmasterPid` 发送 `SIGUSR1`。
这里故意不直接 signal startup process。
原因是 startup process 是 postmaster 管理的特殊子进程。
postmaster 才知道当前是否存在 `StartupPMChild`，以及状态是否处于：
```text
PM_STARTUP
PM_RECOVERY
PM_HOT_STANDBY
```
postmaster 在 `process_pm_pmsignal()` 里检查 promote 文件。
满足条件时，它执行：
```text
signal_child(StartupPMChild, SIGUSR2)
```
注释特别说明：
```text
Leave the promote signal file in place and let the Startup process do the unlink.
```
这让请求的最终接受者是 startup process。
startup process 的 `SIGUSR2` handler 在 `startup.c`。
它只设置 `promote_signaled`。
真正处理发生在 `xlogrecovery.c`。
`CheckForStandbyTrigger()` 检查：
```text
IsPromoteSignaled()
CheckPromoteSignal()
```
两个条件都满足时，它打日志：
```text
received promote request
```
然后执行：
```text
RemovePromoteSignalFiles()
ResetPromoteSignaled()
SetPromoteIsTriggered()
```
`SetPromoteIsTriggered()` 还会调用：
```text
SetRecoveryPause(false)
```
这解释了一个重要边界。
如果 recovery 正处于 pause 状态，promotion 不是继续保持 paused。
promotion 请求会解除 pause，让 recovery 走向结束。
所以 `pg_wal_replay_pause()` 和 `pg_promote()` 不是两个互不干扰的开关。
promotion 有更高优先级。
### pg_promote(wait) 等待的是什么
`pg_promote(wait => true)` 创建文件并 signal postmaster 后，进入循环。
循环里它反复检查：
```text
RecoveryInProgress()
```
如果变成 false，就返回 true。
等待使用 `WAIT_EVENT_PROMOTE`，每 100ms timeout 一次。
如果超过 `wait_seconds`，它发出 WARNING 并返回 false。
这意味着：
```text
pg_promote(wait => true) 返回 true:
  说明本 session 观察到 SharedRecoveryState 已经 DONE。

pg_promote(wait => false) 返回 true:
  只说明请求已经发起。

pg_promote(wait => true) 返回 false:
  说明等待窗口内没有观察到 recovery 完成；
  不等于 promotion 一定失败。
```
诊断时不要把 `pg_promote()` 的返回值当成完整的 end-of-recovery 进度条。
它没有告诉你 walreceiver 是否已停。
没有告诉你 `.history` 是否已归档。
也没有告诉你后续强制 checkpoint 是否完成。
## 6. startup process 在哪里停止 replay
promotion 请求不在任意 C 语句中断 recovery。
startup process 只在明确的 wait/replay 边界检查它。
典型位置包括：
```text
PerformWalRecovery() 主循环
recoveryPausesHere()
WaitForWALToBecomeAvailable()
WaitForWALToBecomeAvailable() 等待 stream/archive/pg_wal 时
```
`PerformWalRecovery()` 每处理一条 record 周期性调用：
```text
ProcessStartupProcInterrupts()
```
在需要等待 WAL、等待 pause、或 recovery target 到达时，也会检查 trigger。
这种设计避免在 redo routine 的任意中间状态强行切断。
redo 正在修改 buffer、SLRU 或 shared state 时，不能突然开始写新 timeline。
promotion 只能在 startup process 已经回到 recovery 控制面时推进。
当 `CheckForStandbyTrigger()` 返回 true，后续主循环会停止等待旧 WAL。
如果已经达到 recovery target，并且 `recovery_target_action = promote`，也会结束 recovery。
但要注意：
```text
外部 promote trigger 会设置 SharedPromoteIsTriggered；
recovery target action promote 只是让 recovery 在目标点结束。
```
这会影响后面 `PerformRecoveryXLogAction()` 选择写轻量 `XLOG_END_OF_RECOVERY` 还是等待完整 checkpoint。
本课重点是 standby promotion trigger。
## 7. FinishWalRecovery：先停 walreceiver，再决定 end-of-log
`StartupXLOG()` 在 `PerformWalRecovery()` 返回后调用：
```text
FinishWalRecovery()
```
这是 promotion 的第一个关键收束点。
函数开头马上调用：
```text
XLogShutdownWalRcv()
```
源码注释解释得很直接：
```text
Kill WAL receiver, if it's still running,
before we continue to write the startup checkpoint and aborted-contrecord records.
```
原因是 walreceiver 仍然属于旧角色。
它从 upstream 接收旧 timeline 的 WAL。
如果它还活着，而 startup process 开始写 end-of-recovery record、checkpoint 或 overwrite-contrecord，那么两个写入者会竞争 `pg_wal` 中同一段历史。
promotion 的第一条不变量是：
```text
本节点成为 WAL writer 之前，旧 upstream 的 walreceiver 必须停止。
```
`XLogShutdownWalRcv()` 是 `xlog.c` 中的薄 wrapper。
它调用 `ShutdownWalRcv()`，然后 `ResetInstallXLogFileSegmentActive()`。
真正停止逻辑在 `walreceiverfuncs.c`。
`ShutdownWalRcv()` 读 `WalRcv->walRcvState`。
如果状态是：
```text
WALRCV_STOPPED
```
不需要做事。
如果是：
```text
WALRCV_STARTING
```
它直接改成 `WALRCV_STOPPED`。
如果是：
```text
WALRCV_CONNECTING
WALRCV_STREAMING
WALRCV_WAITING
WALRCV_RESTARTING
```
它改成 `WALRCV_STOPPING`，记录 pid，并向 walreceiver 发送 `SIGTERM`。
最后它在 `walRcvStoppedCV` 上等待：
```text
while (WalRcvRunning())
  ConditionVariableSleep(..., WAIT_EVENT_WAL_RECEIVER_EXIT)
```
walreceiver 退出时在 `WalRcvDie()` 中设置：
```text
walRcvState = WALRCV_STOPPED
pid = 0
procno = INVALID_PROC_NUMBER
```
并 broadcast condition variable。
这个等待不是性能细节。
它是角色切换的 correctness 边界。
只有确认 walreceiver 不会继续写旧 WAL，startup process 才能把本地 WAL insertion 打开。
## 8. FinishWalRecovery：关闭 standby mode 和重新确认 end-of-log
停 walreceiver 后，`FinishWalRecovery()` 还会调用：
```text
ShutDownSlotSync()
```
slot sync worker 只在 standby 上有意义。
promotion 后它不能继续从旧 upstream 拉 failover slots。
然后代码断言：
```text
Assert(!WalRcvStreaming())
```
再设置：
```text
StandbyMode = false
```
注释强调顺序：
```text
standby mode must be turned off after killing WAL receiver
```
原因是关闭 standby mode 会改变 WAL source 状态机的含义。
如果先关 standby mode，walreceiver 仍可能在短窗口内继续作为 streaming source 写入。
这会让 end-of-recovery 边界变得含糊。
接下来 `FinishWalRecovery()` 要回答一个精确问题：
```text
新 primary 应该从哪个 LSN 开始写新 WAL？
```
它不能简单使用最后一次 `lastReplayedEndRecPtr`。
WAL 末尾可能存在：
```text
一个完整 record 后的 partial page
一个不完整 continuation record
一个已经读到但不能作为有效历史延续的尾部
```
所以函数重新定位到最后一条有效或已应用 record：
```text
lastRec = XLogRecoveryCtl->lastReplayedReadRecPtr
lastRecTLI = XLogRecoveryCtl->lastReplayedTLI
XLogPrefetcherBeginRead(xlogprefetcher, lastRec)
ReadRecord(..., lastRecTLI)
endOfLog = xlogreader->EndRecPtr
```
同时记录：
```text
result->endOfLogTLI = xlogreader->seg.ws_tli
```
`endOfLogTLI` 可能和 `lastRecTLI` 不同。
原因是 segment 文件名里的 TLI 可能已经来自更高 timeline，而 record 自身属于旧 timeline。
这一点是 timeline 课程的延续。
promotion 要尊重 record 的历史语义，也要尊重文件边界。
如果当前是 archive recovery，`FinishWalRecovery()` 会设置：
```text
InArchiveRecovery = false
```
并关闭已经打开的 read file。
它还会复制最后一个 partial block 到 `EndOfWalRecoveryInfo`。
后续 `StartupXLOG()` 用这段 bytes 初始化 WAL buffer，保证新 WAL 从 `EndOfLog` 后面连续写入。
`EndOfWalRecoveryInfo` 最后携带这些状态返回：
```text
lastRec
lastRecTLI
endOfLog
endOfLogTLI
lastPageBeginPtr
lastPage
abortedRecPtr
missingContrecPtr
recoveryStopReason
standby_signal_file_found
recovery_signal_file_found
```
这不是普通返回值集合。
它是 end-of-recovery 后半段的临时契约。
`FinishWalRecovery()` 负责确定旧历史的末端。
`StartupXLOG()` 负责从这个末端开始建立新历史。
## 9. 一致性检查：不能在不完整 backup 边界 promotion
`StartupXLOG()` 拿到 `EndOfLog` 后，先做恢复完整性检查。
关键条件是：
```text
InRecovery &&
(EndOfLog < LocalMinRecoveryPoint ||
 XLogRecPtrIsValid(ControlFile->backupStartPoint))
```
如果 archive recovery 请求存在，或者 control file 表示 backup end 仍然 required，就可能 FATAL：
```text
WAL ends before end of online backup
WAL ends before consistent recovery point
```
这一步回答一个容易被忽略的问题：
```text
promotion 请求能不能强行把一个尚未达到一致性的 standby 提升成 primary？
```
答案是不能。
promotion 是结束 recovery。
但它不能跳过恢复的一致性下限。
如果 base backup 还没越过 end-of-backup record 或 minRecoveryPoint，数据目录不是一个可以写入的新 primary 起点。
这一步发生在 timeline 切换和 signal 文件删除之前。
如果这里失败，节点仍然不应被当成新 primary。
这也解释了为什么运维诊断要把 promotion failure 和 WAL 缺失联系起来。
如果日志里有 “WAL ends before consistent recovery point”，问题不是 `pg_promote()` 本身。
问题是 recovery 没有可用 WAL 到达一致边界。
## 10. 选择新 timeline：为什么不能继续写旧 TLI
promotion 的下一步是决定 `newTLI`。
`StartupXLOG()` 先设：
```text
newTLI = endOfRecoveryInfo->lastRecTLI
```
这是 crash recovery 的普通情况。
如果只是崩溃后重启，系统可以继续原 timeline。
但如果 `ArchiveRecoveryRequested` 为 true，代码总是选择新 timeline：
```text
newTLI = findNewestTimeLine(recoveryTargetTLI) + 1
```
并打日志：
```text
selected new timeline ID: %u
```
这里的理由有两层。
第一，如果 recovery 停在旧 WAL 中间，显然会产生新历史。
第二，即使 replay 到旧 WAL 的末尾，直接修改当前最后一个 segment 也会和已经归档的旧 segment 冲突。
很多 `archive_command` 会拒绝覆盖已有归档文件。
新 timeline 让新 primary 的 WAL 文件名不同，从而避免覆盖旧历史。
然后调用：
```text
XLogInitNewTimeline(EndOfLogTLI, EndOfLog, newTLI)
```
它为新 timeline 准备可写 segment。
在这个点之后，本地已经准备好从 `EndOfLog` 继续写新历史。
但系统还没有完全离开 recovery。
接下来要移除 signal 文件并写 `.history`。
## 11. 移除 recovery/standby signal 文件的时机
promotion 后不能让下一次启动再次进入 recovery。
因此 `StartupXLOG()` 会删除启动时发现的 signal 文件：
```text
standby.signal
recovery.signal
```
对应代码是：
```text
if (endOfRecoveryInfo->standby_signal_file_found)
  durable_unlink(STANDBY_SIGNAL_FILE, FATAL)

if (endOfRecoveryInfo->recovery_signal_file_found)
  durable_unlink(RECOVERY_SIGNAL_FILE, FATAL)
```
注意这不是删除 `promote` 文件。
`promote` 文件在 `CheckForStandbyTrigger()` 接受请求时已经由 startup process 删除。
这里删除的是 recovery mode 的启动请求。
`durable_unlink()` 的使用也很重要。
如果只是普通 `unlink()` 后崩溃，下一次启动可能还会在文件系统持久化边界上看到旧 signal 文件。
end-of-recovery 的目标是让角色切换可重启。
所以 signal 文件的移除属于 durable state transition。
移除顺序在写 `.history` 之前。
这降低了 crash 后误入 standby/recovery 的风险。
但它并不单独代表 promotion 完成。
如果后续写 history 或 EOR record 失败，startup process 会失败，postmaster 会按 startup crash 处理。
## 12. 写 timeline history：让分叉点对外可见
删除 signal 文件后，`StartupXLOG()` 调用：
```text
writeTimeLineHistory(newTLI, recoveryTargetTLI,
                     EndOfLog, endOfRecoveryInfo->recoveryStopReason)
```
实现位于 `timeline.c`。
它会先写临时文件：
```text
pg_wal/xlogtemp.<pid>
```
如果 parent timeline 有历史文件，它会复制 parent history。
然后追加一行：
```text
parentTLI    switchpoint    reason
```
最后 fsync、close、`durable_rename()` 成目标 `.history` 文件。
如果开启 WAL archiving，它会调用：
```text
XLogArchiveNotify(histfname)
```
这一步的语义是：
```text
新 timeline 已经有了公开的历史声明。
级联 standby 或未来 PITR 可以知道新历史从哪个 LSN 分叉。
```
它不是 WAL record。
它是 WAL 文件命名空间外的一份历史索引。
源码注释承认这里有一个窗口。
history 文件一旦出现，其他 standby 可能认为新 timeline 已经存在。
如果随后在真正切换到新 timeline 之前崩溃，会出现短暂不一致。
所以 `StartupXLOG()` 在写 history 后尽量少做事，尽快写 end-of-recovery record。
这就是课程里要保留的 awkwardness。
系统不能原子地同时发布 `.history` 文件、WAL record、control file 和 postmaster 状态。
它只能通过顺序和 crash recovery 规则缩小窗口。
## 13. 设置 InsertTimeLineID / PrevTimeLineID：共享 WAL insertion 边界
history 文件写完后，`StartupXLOG()` 更新 shared xlog state：
```text
SpinLockAcquire(&XLogCtl->info_lck)
XLogCtl->InsertTimeLineID = newTLI
XLogCtl->PrevTimeLineID = endOfRecoveryInfo->lastRecTLI
SpinLockRelease(&XLogCtl->info_lck)
```
`InsertTimeLineID` 是接下来写 WAL 的 timeline。
`PrevTimeLineID` 是本次分叉前的 timeline。
这两个字段随后被 `CreateEndOfRecoveryRecord()` 或 `CreateCheckPoint()` 读走。
所以它们必须在写 EOR record/checkpoint 之前设置。
如果 WAL 末尾有 incomplete continuation record，`StartupXLOG()` 还会处理：
```text
missingContrecPtr
abortedRecPtr
```
如果需要，它会把 `EndOfLog` 推到 `missingContrecPtr`。
当发生 timeline switch 时，源码断言不应有这种 missing continuation 情况。
因为新 timeline 只复制旧 WAL 到最后一个完整 record 结束位置。
这部分逻辑服务一个不变量：
```text
新 primary 写出的第一条 WAL 不能把旧 timeline 尾部的 broken record 当成有效前缀。
```
随后 `StartupXLOG()` 用 `EndOfLog` 初始化 WAL insertion shared state：
```text
Insert->PrevBytePos = XLogRecPtrToBytePos(lastRec)
Insert->CurrBytePos = XLogRecPtrToBytePos(EndOfLog)
```
如果最后一页是 partial block，它把旧页面有效部分复制到 WAL buffer，并把剩余部分清零。
这保证新 WAL 不是从一个未初始化 buffer 开始。
这里还没有允许普通 backend 写 WAL。
只是 startup process 建好了本地和 shared WAL insertion 起点。
## 14. 从 recovery 到 local WAL writer
接下来 `StartupXLOG()` 更新写入位置：
```text
LogwrtResult.Write = EndOfLog
LogwrtResult.Flush = EndOfLog
XLogCtl->logInsertResult = EndOfLog
XLogCtl->logWriteResult = EndOfLog
XLogCtl->logFlushResult = EndOfLog
XLogCtl->LogwrtRqst.Write = EndOfLog
XLogCtl->LogwrtRqst.Flush = EndOfLog
```
然后预创建 WAL 文件：
```text
PreallocXlogFiles(EndOfLog, newTLI)
```
此后源码写了一句很重要的本地状态：
```text
InRecovery = false
```
这个变量是 startup process 本地事实。
它不是普通 backend 的 `pg_is_in_recovery()` 来源。
普通 backend 看的是 `XLogCtl->SharedRecoveryState`。
所以课程要区分：
```text
startup process 自己已经离开 replay path
  !=
其他进程已经可以写 WAL
```
`StartupXLOG()` 接着初始化 `latestCompletedXid`、启动或补齐 SLRU、恢复 prepared transactions。
然后调用：
```text
ShutdownWalRecovery()
```
它释放 xlogreader、xlogprefetcher、关闭剩余 WAL read file，并清理 archive restore 临时文件。
这一步是 recovery reader 的 cleanup。
它发生在启用本地 WAL insert 之前。
随后 startup process 调用：
```text
LocalSetXLogInsertAllowed()
```
这只让当前 startup process 能写 WAL。
普通 backend 此时仍不能写。
如果之前发现 broken continuation record，startup process 会先写：
```text
CreateOverwriteContrecordRecord(abortedRecPtr, missingContrecPtr, newTLI)
```
之后更新 `full_page_writes` 并写 `XLOG_FPW_CHANGE`。
这些都是在真正发布 `RECOVERY_STATE_DONE` 前由 startup process 收尾的 WAL。
## 15. end-of-recovery record 与 end-of-recovery checkpoint
源码里的名字容易造成误读。
很多文档和日志会把结束恢复称为 end-of-recovery checkpoint。
但 master 当前 promotion trigger 路径有一个重要优化：
```text
真正的 promotion 不等待完整 checkpoint；
它先写轻量 XLOG_END_OF_RECOVERY record；
之后再请求一个普通 checkpoint。
```
选择发生在：
```text
PerformRecoveryXLogAction()
```
条件是：
```text
ArchiveRecoveryRequested &&
IsUnderPostmaster &&
PromoteIsTriggered()
```
满足时：
```text
promoted = true
CreateEndOfRecoveryRecord()
```
不满足时：
```text
RequestCheckpoint(CHECKPOINT_END_OF_RECOVERY |
                  CHECKPOINT_FAST |
                  CHECKPOINT_WAIT)
```
所以同样是“结束恢复”，有两条路径。
standby 被 `pg_promote()` 触发时，走 `XLOG_END_OF_RECOVERY`。
PITR 到 recovery target 后结束，或 standalone 场景，可能走 end-of-recovery checkpoint。
这不是语义差异。
两者都在 WAL 中建立“旧 recovery 历史结束，新 production 历史开始”的 durable 边界。
差异是 I/O 策略。
promotion 追求快速恢复写能力。
完整 checkpoint 可能很慢。
如果 checkpointer 正在做 time-smoothed restartpoint，强行等待它收尾再 checkpoint 会拖长 failover。
所以 promotion 先写一个很小的 record：
```text
XLOG_END_OF_RECOVERY
```
随后在完全离开 recovery 后请求：
```text
RequestCheckpoint(CHECKPOINT_FORCE)
```
这解释了：
```text
pg_promote(wait => true) 返回 true
```
通常不表示后续 force checkpoint 已经完成。
它表示 recovery state 已经 DONE。
如果系统随后立刻崩溃，crash recovery 会从 control file 和 EOR record 重新建立正确 timeline 边界。
## 16. CreateEndOfRecoveryRecord：轻量 promotion 标记
`CreateEndOfRecoveryRecord()` 在 `xlog.c`。
它首先检查：
```text
if (!RecoveryInProgress())
  elog(ERROR, "can only be used to end recovery")
```
注意这里仍然要求共享 recovery state 未 DONE。
promotion 的 EOR record 是在对外宣布 production 之前写的。
函数构造：
```text
xl_end_of_recovery xlrec
```
字段包括：
```text
end_time
wal_level
ThisTimeLineID
PrevTimeLineID
```
`ThisTimeLineID` 和 `PrevTimeLineID` 从 `XLogCtl` 读取。
读取时持有 `WALInsertLockAcquireExclusive()`。
随后进入 critical section：
```text
XLogBeginInsert()
XLogRegisterData(&xlrec, sizeof(xl_end_of_recovery))
recptr = XLogInsert(RM_XLOG_ID, XLOG_END_OF_RECOVERY)
XLogFlush(recptr)
```
`XLogFlush(recptr)` 是 durable 边界。
如果 EOR record 没有 flush，就不能把 recovery 结束发布给其他进程。
flush 后它更新 pg_control：
```text
ControlFile->minRecoveryPoint = recptr
ControlFile->minRecoveryPointTLI = xlrec.ThisTimeLineID
ControlFile->data_checksum_version = XLogCtl->data_checksum_version
UpdateControlFile()
```
这个 `minRecoveryPoint` 更新很关键。
promotion 后第一个普通 checkpoint 可能还没完成。
如果此时崩溃，下一次启动不能只按旧 checkpoint redo。
它必须至少 replay 到 EOR record。
`minRecoveryPoint` 给 crash recovery 一个安全下限。
所以 EOR record 不是“少写 checkpoint 的偷懒”。
它是把 timeline switch 的最低恢复点 durable 地写进 WAL 和 pg_control。
## 17. end-of-recovery checkpoint 路径：同样用 shutdown checkpoint 表达 timeline switch
如果不走轻量 EOR record，`PerformRecoveryXLogAction()` 请求 checkpoint：
```text
CHECKPOINT_END_OF_RECOVERY
CHECKPOINT_FAST
CHECKPOINT_WAIT
```
`CreateCheckPoint()` 把 `CHECKPOINT_END_OF_RECOVERY` 当成 shutdown checkpoint：
```text
if (flags & (CHECKPOINT_IS_SHUTDOWN | CHECKPOINT_END_OF_RECOVERY))
  shutdown = true
```
注释说明：
```text
An end-of-recovery checkpoint is really a shutdown checkpoint,
just issued at a different time.
```
为什么用 shutdown checkpoint？
因为 timeline switch 只允许出现在 shutdown checkpoint 或 end-of-recovery record 这类边界上。
`CreateCheckPoint()` 写 checkpoint record 时设置：
```text
checkPoint.ThisTimeLineID = XLogCtl->InsertTimeLineID
if (flags & CHECKPOINT_END_OF_RECOVERY)
  checkPoint.PrevTimeLineID = XLogCtl->PrevTimeLineID
else
  checkPoint.PrevTimeLineID = checkPoint.ThisTimeLineID
```
这和 EOR record 的字段语义一致。
redo 端在 `xlogrecovery.c` 看到 `XLOG_CHECKPOINT_SHUTDOWN` 时，会从 checkpoint record 中读出：
```text
newReplayTLI = checkPoint.ThisTimeLineID
prevReplayTLI = checkPoint.PrevTimeLineID
```
如果看到 `XLOG_END_OF_RECOVERY`，则从 `xl_end_of_recovery` 中读同样语义的字段。
然后调用：
```text
checkTimeLineSwitch(...)
```
所以 timeline switch 的 redo 语义被统一到两类 WAL record：
```text
shutdown checkpoint
end-of-recovery record
```
在线 checkpoint 不允许改变 TLI。
这就是 `ThisTimeLineID` / `PrevTimeLineID` 的正确性边界。
它们不是给人看日志的附属信息。
它们让 redo 能验证：
```text
这一条记录是否在预期 timeline 上；
这次 timeline switch 是否从正确 parent 发生。
```
## 18. control file 和 SharedRecoveryState：真正对外解除 recovery
写完 EOR record 或 end-of-recovery checkpoint 后，`StartupXLOG()` 还没有立刻让普通 backend 写。
它还会：
```text
XLogReportParameters()
CleanupAfterArchiveRecovery()
CompleteCommitTsInitialization()
UpdateLogicalDecodingStatusEndOfRecovery()
```
然后才进入最关键的状态发布。
源码注释说得很清楚：
```text
Now allow backends to write WAL and update the control file status in consequence.
```
发布在 `ControlFileLock` 下完成：
```text
LWLockAcquire(ControlFileLock, LW_EXCLUSIVE)
ControlFile->state = DB_IN_PRODUCTION

SpinLockAcquire(&XLogCtl->info_lck)
ControlFile->data_checksum_version = XLogCtl->data_checksum_version
XLogCtl->SharedRecoveryState = RECOVERY_STATE_DONE
SpinLockRelease(&XLogCtl->info_lck)

UpdateControlFile()
LWLockRelease(ControlFileLock)
```
这个顺序是本节最重要的边界。
普通 backend 通过 `RecoveryInProgress()` 判断能否写。
`RecoveryInProgress()` 读：
```text
XLogCtl->SharedRecoveryState != RECOVERY_STATE_DONE
```
`XLogInsertAllowed()` 在第一次看到 recovery 已结束后，把本地缓存设为 true。
后续 `XLogBeginInsert()` 就不再报：
```text
cannot make new WAL entries during recovery
```
但 shared recovery state 不能在 control file 仍旧表示 recovery 时先改。
否则其他 backend 可能开始写 WAL，而 shared memory 里的 control file 还不是 `DB_IN_PRODUCTION`。
源码因此在 `ControlFileLock` 下同时更新 control file state 和 shared state。
注释承认仍有一个小窗口：
```text
backend 可以写 WAL；
on-disk control file 刚好还没完全反映 DB_IN_PRODUCTION。
```
但 shared memory 里的 control file 和 `SharedRecoveryState` 在锁保护下保持一致。
这就是 current runtime 的一致性边界。
## 19. ShutdownRecoveryTransactionEnvironment：为什么在 DONE 之后清理 Hot Standby 环境
如果 standby 曾经启用 Hot Standby，startup process 初始化过 recovery transaction environment。
promotion 后要清理它：
```text
ShutdownRecoveryTransactionEnvironment()
```
但源码把这一步放在 `SharedRecoveryState = RECOVERY_STATE_DONE` 之后。
注释解释了原因。
普通 session 构造 snapshot 时会先看 `RecoveryInProgress()`。
如果 recovery 仍然 true，它可能依赖 KnownAssignedXids。
如果先清理 KnownAssignedXids，再让 `RecoveryInProgress()` 变 false，中间窗口里的 backend 可能拿到错误边界。
尤其 prepared 2PC transaction 仍需要包含进 snapshot。
因此顺序是：
```text
RecoverPreparedTransactions()
  -> SharedRecoveryState = DONE
  -> ShutdownRecoveryTransactionEnvironment()
```
这体现一个常见内核规律：
```text
cleanup 不能只按“谁不再需要”判断；
必须按“其他并发读者何时不再会按旧语义访问它”判断。
```
promotion 的 cleanup 也是状态发布协议的一部分。
不是最后顺手释放内存。
## 20. startup process 退出与 postmaster 切换到 PM_RUN
`StartupXLOG()` 完成后返回 `StartupProcessMain()`。
`StartupProcessMain()` 调用：
```text
proc_exit(0)
```
退出码 0 告诉 postmaster：
```text
recovery completed successfully
```
postmaster 在 reaper 逻辑中发现 `StartupPMChild` 退出。
如果退出码是 0，它执行：
```text
StartupStatus = STARTUP_NOT_RUNNING
FatalError = false
AbortStartTime = 0
ReachedNormalRunning = true
UpdatePMState(PM_RUN)
connsAllowed = true
StartWorkerNeeded = true
```
然后打日志：
```text
database system is ready to accept connections
```
注意 Hot Standby 已经开放过只读连接时，之前也有一条日志：
```text
database system is ready to accept read-only connections
```
promotion 完成后看到的是没有 `read-only` 的版本。
这条日志来自 postmaster，不来自 startup process。
这就是进程管理层面的角色切换。
`maybe_start_bgworkers()` 之后会按 `PM_RUN` 启动只在正常运行时需要的进程。
例如：
```text
WAL writer
autovacuum launcher
BgWorkerStart_RecoveryFinished 的 background workers
```
checkpointer 和 bgwriter 在 recovery 期间已经可以运行。
walreceiver 不会在 `PM_RUN` 中再次启动。
`WalSndWakeup(true, true)` 还会唤醒 walsender。
如果这个 standby 原先有级联 standby 连接，walsender 需要意识到本节点已 promotion，并让下游处理 timeline 变化。
## 21. 可写 primary 的真正条件
现在可以把“切换成可写 primary”压缩成四个条件。
第一，旧 WAL source 停止：
```text
walreceiver stopped
StandbyMode = false
InArchiveRecovery = false
```
第二，新历史已经声明：
```text
new timeline selected
.history written
InsertTimeLineID / PrevTimeLineID set
EOR record or end-of-recovery checkpoint flushed
```
第三，实例状态已经发布：
```text
ControlFile->state = DB_IN_PRODUCTION
SharedRecoveryState = RECOVERY_STATE_DONE
```
第四，postmaster 已进入正常运行：
```text
startup process exit 0
pmState = PM_RUN
connsAllowed = true
```
只有四层都收敛，外部才应把节点当成新的 primary。
如果只看一个信号，会误判。
例如：
```text
promote 文件存在:
  请求已发起，不代表已接受。

received promote request:
  startup process 已接受，不代表 WAL 边界已写。

selected new timeline ID:
  已选 TLI，不代表 SharedRecoveryState 已 DONE。

pg_is_in_recovery() = false:
  当前进程看到 recovery 已 DONE，不代表 postmaster 后台任务都已完成启动。

database system is ready to accept connections:
  postmaster 已进入 PM_RUN，是对外连接层最接近完成的日志。
```
生产诊断通常要把这些日志按时间排序。
promotion 是一个小状态机，不是单个事件。
## 22. 异常路径一：walreceiver 停不下来
`ShutdownWalRcv()` 可能等待 walreceiver 退出。
等待事件是：
```text
WAIT_EVENT_WAL_RECEIVER_EXIT
```
如果 walreceiver 正在系统调用、网络关闭或写盘路径上，它需要先响应 `SIGTERM` 并走 `WalRcvDie()`。
startup process 在这段时间不会继续写 EOR record。
从外部看，可能出现：
```text
pg_promote(wait => true) 尚未返回；
pg_stat_get_recovery().promote_triggered = true；
pg_is_in_recovery() 仍然 true；
walreceiver 相关状态正在退出。
```
这不是矛盾。
promotion 已经被接受，但还没有越过“旧 WAL writer 已停止”的边界。
如果这里长时间卡住，诊断重点不应先看 timeline。
应先看 walreceiver 进程是否仍在、wait event、系统 I/O、网络断开和 postmaster 日志。
## 23. 异常路径二：history 文件或 control file 持久化失败
`writeTimeLineHistory()` 可能在 create、write、fsync、close 或 rename 失败。
这些路径会 ERROR。
因为 startup process 是 auxiliary process，关键失败会导致 startup 失败，postmaster 进入 crash/restart 处理。
`CreateEndOfRecoveryRecord()` 在 critical section 中写 WAL 并更新 control file。
如果在这段 critical section 出现严重 I/O 问题，错误可能升级到 PANIC。
这不是过度保守。
promotion 发布新历史时，最怕的是：
```text
一部分 durable state 说已经 promotion；
另一部分 durable state 说仍在旧 recovery。
```
因此写 history、写 WAL record、更新 pg_control 都采用 fsync/critical section/lock 边界。
`UpdateControlFile()` 的失败不是普通业务 ERROR。
它关系到 crash 后系统从哪个状态继续。
如果日志里出现 history 或 control file 写失败，不要把节点手工标记为 primary。
应先确认它下一次启动能否从 pg_control 和 WAL 中一致地恢复。
## 24. 异常路径三：promotion 期间崩溃
promotion 期间崩溃最需要分阶段理解。
如果崩溃发生在 `CheckForStandbyTrigger()` 接受请求之前，promote 文件可能仍在。
下一次 postmaster 启动时会清理旧 promote signal 文件，避免误触发过期请求。
`postmaster.c` 启动早期有 `RemovePromoteSignalFiles()` 的防竞态处理。
如果崩溃发生在 signal 文件删除后、history 写入前，下一次启动会按 pg_control 和 WAL 状态继续恢复。
如果崩溃发生在 history 写入后、EOR record 前，archive 里可能短暂看见新 `.history`。
源码通过“写 history 后尽量少做事”缩小窗口。
如果崩溃发生在 EOR record flush 后、后续 force checkpoint 前，pg_control 的 `minRecoveryPoint` 已经指向 EOR record。
下一次启动会 replay 到这个点，识别 timeline switch。
如果崩溃发生在 `SharedRecoveryState = DONE` 后，control file 已经进入 `DB_IN_PRODUCTION`。
下一次启动走普通 crash recovery。
这四种情况说明：
```text
promotion crash safety 不是靠一个大事务；
而是靠每个阶段留下足够的 durable breadcrumb，
让下一次 startup process 可以继续判断。
```
## 25. 成本、资源与跨模块传播
promotion 的 hot path 目标是低延迟，但仍有几个不可省略的成本。
第一是 `ShutdownWalRcv()`。
如果 walreceiver 正在 flush 接收的 WAL 或退出网络循环，startup process 必须等到 `WALRCV_STOPPED`。
第二是 `.history` 文件。
它很小，但需要 write、fsync、durable rename，并可能通知 archiver。
第三是 `CreateEndOfRecoveryRecord()` 的 `XLogFlush(recptr)`。
这个 flush 是 promotion 的 durable WAL 边界。
第四是非 promotion 路径的 end-of-recovery checkpoint。
`CHECKPOINT_END_OF_RECOVERY | CHECKPOINT_WAIT` 要写脏页、SLRU 和 checkpoint record，成本远高于轻量 EOR record。
第五是 promotion 后的 force checkpoint。
`RequestCheckpoint(CHECKPOINT_FORCE)` 不阻塞 `pg_promote(wait)` 到 checkpoint 完成，但会给刚成为 primary 的节点带来短期 I/O 压力。
跨模块传播也在这几步展开。
replication 层停止 walreceiver、唤醒 walsender，让下游有机会处理 timeline 变化。
archive 层归档新的 `.history`，避免旧 timeline 被覆盖。
checkpoint 层先用 EOR record 快速离开 recovery，再做安全性更强的后续 checkpoint。
logical decoding 层通过 `UpdateLogicalDecodingStatusEndOfRecovery()` 更新 shared status。
postmaster 层把 `PM_RECOVERY` 或 `PM_HOT_STANDBY` 推到 `PM_RUN`，后台进程启动条件随之变化。
promotion 因此不是 xlog 模块内部事件。
它把复制拓扑、归档、checkpoint、后台进程生命周期和 SQL 连接语义一起推进。
## 26. 观测与诊断入口
最直接的 SQL 入口是：
```sql
select pg_is_in_recovery();
select * from pg_stat_get_recovery();
select pg_promote(true, 60);
```
`pg_is_in_recovery()` 调用 `RecoveryInProgress()`，本质上看 `XLogCtl->SharedRecoveryState` 是否已经 `RECOVERY_STATE_DONE`。
`pg_stat_get_recovery()` 只在 recovery 中返回一行。
其中 `promote_triggered` 来自 `XLogRecoveryCtl->SharedPromoteIsTriggered`。
如果它为 true，但 `pg_is_in_recovery()` 仍为 true，说明 startup process 已接受 promotion，但还没完成结束恢复。
`pg_promote(wait => false)` 返回 true 只表示请求已发起。
`pg_promote(wait => true)` 返回 true 表示等待期间观察到 recovery DONE。
返回 false 表示等待超时，不等于 promotion 必然失败。
日志要按顺序读：
```text
received promote request
selected new timeline ID: N
archive recovery complete
database system is ready to accept connections
```
如果之前已经开放 Hot Standby，会更早看到 `database system is ready to accept read-only connections`。
文件系统入口包括：
```text
standby.signal / recovery.signal 是否已移除
pg_wal/0000000N.history 是否存在
```
`pg_controldata` 能看 on-disk control file state。
但诊断时必须区分 shared memory 当前事实、on-disk 下次启动事实、日志历史事件和 SQL 当前观察。
## 27. pg_promote 卡住时怎么拆
如果 `select pg_promote(true, 60);` 超时，先不要直接判断 timeline 损坏。
先看 `pg_is_in_recovery()`。
如果已经 false，说明当前连接观察到的 promotion 已完成，问题可能只是调用等待窗口或业务连接切换。
如果仍 true，再看 `pg_stat_get_recovery()`。
`promote_triggered = false` 表示请求还没被 startup process 接受，重点检查 promote 文件、postmaster `SIGUSR1`、`StartupPMChild` 是否存在。
`promote_triggered = true` 表示请求已经进入 startup process，下一步按日志分段。
有 `received promote request` 但没有 `selected new timeline ID`，重点看 replay loop 退出、walreceiver shutdown 和一致性检查。
有 `selected new timeline ID` 但没有 `archive recovery complete`，重点看 `XLogInitNewTimeline()`、signal 文件删除和 `.history` 写入。
有 `archive recovery complete` 但没有 `database system is ready to accept connections`，重点看 EOR record/checkpoint、control file 更新、startup process exit 0 和 postmaster reaper。
如果 walreceiver 仍在，关注 `WAIT_EVENT_WAL_RECEIVER_EXIT`。
如果 EOR record 已写但后续 checkpoint 仍在跑，短期 I/O 压力通常来自 promotion 后的 `CHECKPOINT_FORCE`，不是 recovery 尚未结束。
诊断顺序可以压缩为：
```text
请求是否被接受
  -> 旧 WAL source 是否关闭
  -> 新 timeline 是否 durable
  -> SharedRecoveryState 是否 DONE
  -> postmaster 是否 PM_RUN
```
## 28. 常见误区
`promotion 就是删除 standby.signal` 是错的。
signal 文件移除只防止下次启动再进 recovery；真正可写边界是 EOR record/checkpoint、control file 和 `SharedRecoveryState`。
`walreceiver 停止后节点就是 primary` 也是错的。
walreceiver 停止只说明旧 WAL source 消失，还要写新 timeline marker 并发布 DONE。
`pg_promote(wait => true) 等待 checkpoint 完成` 不符合当前 promotion trigger 路径。
它等待 recovery DONE，后续 force checkpoint 可能仍在推进。
`ThisTimeLineID` 可以单独解释 timeline 也不对。
在 timeline switch record 中，`PrevTimeLineID` 是 redo 验证分叉合法性的必要字段。
`pg_is_in_recovery()` 能解释所有卡点也不对。
它只能回答 `SharedRecoveryState` 是否 DONE，看不到 walreceiver shutdown、history fsync 或 postmaster reaper 的细分阶段。
`history 文件归档完成才允许写` 也不准确。
本地 promotion 需要写 history 并通知归档，但 `pg_promote(wait)` 不等待所有下游都拿到 history。
## 29. 课堂实验
实验一：源码断点跟读。
按顺序在 `pg_promote()`、`process_pm_pmsignal()`、`StartupProcTriggerHandler()`、`CheckForStandbyTrigger()`、`FinishWalRecovery()`、`ShutdownWalRcv()`、`PerformRecoveryXLogAction()`、`CreateEndOfRecoveryRecord()` 和 postmaster reaper 的 startup exit 0 分支设置断点。
每个断点记录 `promote_signaled`、`SharedPromoteIsTriggered`、`WalRcvState`、`InsertTimeLineID`、`PrevTimeLineID`、`ControlFile->state`、`SharedRecoveryState` 和 `pmState`。
目标是画出 local state、shared memory state、durable control file state、postmaster private state 的转换图。
实验二：SQL 与日志对照。
在 standby 上执行：
```sql
select pg_is_in_recovery();
select * from pg_stat_get_recovery();
select pg_promote(true, 60);
select pg_is_in_recovery();
```
把结果和日志中的 `received promote request`、`selected new timeline ID`、`archive recovery complete`、`database system is ready to accept connections` 对齐。
再检查 `standby.signal` 是否消失、`pg_wal/*.history` 是否出现。
实验三：区分 EOR record 和后续 checkpoint。
开启 `log_checkpoints = on` 后 promotion。
如果 `pg_promote(wait => true)` 已返回、业务已经能写，但随后日志中仍有 checkpoint I/O，这对应 `StartupXLOG()` 末尾的 `RequestCheckpoint(CHECKPOINT_FORCE)`。
不要把后续 checkpoint I/O 误判为 promotion 未完成。
## 30. 讨论题
1. 为什么 promotion 必须先 `ShutdownWalRcv()`，而不是先写 `XLOG_END_OF_RECOVERY`？
2. 为什么 archive recovery 结束即使 replay 到旧 WAL 末尾，也要选择新 timeline？
3. `ThisTimeLineID` 和 `PrevTimeLineID` 为什么同时出现在 shutdown checkpoint 和 end-of-recovery record 中？
4. `pg_promote(wait => true)` 为什么不能承诺后续 force checkpoint 已完成？
5. `SharedRecoveryState = RECOVERY_STATE_DONE` 为什么要和 `ControlFile->state = DB_IN_PRODUCTION` 放在同一段 `ControlFileLock` 保护下？
6. 如果看到 `selected new timeline ID` 但没有 `database system is ready to accept connections`，应该优先检查哪几段源码边界？
7. 为什么 `ShutdownRecoveryTransactionEnvironment()` 要放在 recovery DONE 之后？
8. 如果 promotion 期间 crash，哪些 durable artifact 能帮助下一次 startup process 判断恢复边界？
## 31. 本节小结
promotion 的核心链路是：
```text
promote request
  -> startup process 接受 trigger
  -> 停 walreceiver
  -> 确认 end-of-log
  -> 选新 timeline
  -> 删除 recovery/standby signal
  -> 写 timeline history
  -> 写 EOR record 或 end-of-recovery checkpoint
  -> control file 进入 DB_IN_PRODUCTION
  -> SharedRecoveryState 进入 RECOVERY_STATE_DONE
  -> postmaster 进入 PM_RUN
```
核心状态分三层。
promotion 请求层回答是否有人要求结束 recovery。
timeline 层回答新 WAL 历史从哪个 LSN、哪个 parent timeline 分叉。
实例角色层回答普通 backend 是否可以把本节点当成 primary 写 WAL。
ownership 和 cleanup 的关键顺序是：
```text
先停旧 WAL source；
再释放 recovery reader；
先发布 DONE；
再清理 Hot Standby recovery transaction environment。
```
错误路径的重点不是把所有失败都吞掉。
恰恰相反，history、WAL marker、control file 的持久化失败必须让 startup 失败或 panic。
因为半发布的新 primary 比 promotion 失败更危险。
观测上，`pg_is_in_recovery()` 只能看 `SharedRecoveryState`。
`pg_promote()` 只能发起请求并可选等待 recovery DONE。
日志、`.history`、walreceiver 状态、pg_control 和 postmaster ready 日志必须合起来解释。
可迁移的系统规律是：
```text
角色切换不是改一个角色字段；
它是停止旧 owner、建立新 durable history、发布 shared runtime state、
再让进程管理层放行新 workload 的线性协议。
```
这个规律同样适用于复制、failover、logical slot ownership 和后台进程接管问题。
判断边界时始终回到四个问题：旧写入者是否真的停了，新历史是否 durable，共享状态是否对并发读者可见，postmaster 是否已经把连接语义切过去。
