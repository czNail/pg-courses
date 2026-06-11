# PostgreSQL startup process 恢复状态机与 Hot Standby 一致性边界
## 课程定位
前置知识：已经理解 WAL redo 的基本含义，知道物理复制中 walsender 发送 WAL、walreceiver 接收 WAL、startup process 重放 WAL。
本节唯一主问题：
```text
standby 启动后如何在 archive recovery、streaming recovery、crash recovery 和 consistent state 之间推进，
哪些条件决定数据库何时可以接受只读查询？
```
核心矛盾：standby 希望尽早对外提供只读查询，但它不能在数据目录还可能包含不完整 base backup、缺失 WAL、未建立 primary running-xacts 快照、或 replay 位置未越过安全点时开放连接。
PostgreSQL 的解法不是一个简单的 `in_recovery` 布尔值。
它把启动恢复拆成几层状态：
```text
是否请求 archive / standby recovery
  -> 当前是否已经进入 archive recovery
  -> 当前从哪个 WAL source 读取
  -> redo 是否到达 consistent recovery point
  -> hot standby 快照是否 ready
  -> postmaster 是否允许只读连接
```
学完后应能判断：一个 standby 已经开始 streaming 并不代表可以查询；已经到达 consistent recovery state 也不单独代表 hot standby 可查询；只有一致性边界和 standby snapshot 边界同时满足，postmaster 才会放行只读 backend。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前几节已经把物理复制拆成了：
```text
walsender 发送 WAL
  -> walreceiver 接收并写入 pg_wal
  -> standby 周期性反馈接收、写入、flush、apply 位置
```
这一节进入 standby 本地最关键的推进者：
```text
startup process
```
在 PostgreSQL 中，walreceiver 只负责从 upstream 拉 WAL。
真正决定“下一条 WAL 从哪里读、什么时候等待、什么时候切换来源、什么时候可以开放只读连接”的，是 startup process 内的 recovery 状态机。
主入口是：
```text
postmaster
  -> StartupProcessMain()
     -> StartupXLOG()
        -> InitWalRecovery()
        -> PerformWalRecovery()
        -> FinishWalRecovery()
```
其中 `StartupProcessMain()` 在 `src/backend/postmaster/startup.c`。
`StartupXLOG()` 在 `src/backend/access/transam/xlog.c`。
大部分恢复状态机在 `src/backend/access/transam/xlogrecovery.c`。
Hot Standby 的事务快照状态在 `src/backend/storage/ipc/standby.c` 和 `src/backend/storage/ipc/procarray.c`。
walreceiver 的共享状态和请求入口在 `src/backend/replication/walreceiver.c`、`src/backend/replication/walreceiverfuncs.c` 和 `src/include/replication/walreceiver.h`。
本节不讲 redo 每个 resource manager 如何改页。
本节只追踪一个问题：
```text
startup process 如何把恢复推进到“可接受只读查询”的状态？
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
StartupXLOG() 先根据 pg_control、backup_label、standby.signal / recovery.signal
判断是否需要 crash / archive / standby recovery；
PerformWalRecovery() 反复调用 ReadRecord() 获取下一条 WAL；
ReadRecord() 通过 WaitForWALToBecomeAvailable() 在 archive、pg_wal、stream 之间切换；
每 replay 一条记录后更新 replay LSN、KnownAssignedXids 和一致性状态；
当 minRecoveryPoint 已经到达且 standby snapshot ready 时，
CheckRecoveryConsistency() 设置 SharedHotStandbyActive 并通知 postmaster 允许只读连接。
```
这里的 tension 是：
```text
越早开放 hot standby，越能降低故障切换或只读扩容的停机感；
但开放过早会让查询看到 base backup 尚未完成、running transaction 信息不完整、
AccessExclusiveLock 恢复状态缺失，或者 WAL replay 后续必须撤销的中间状态。
```
因此 PostgreSQL 不是按“walreceiver 已连接”开放查询。
它按两个门槛开放查询：
```text
数据库物理一致：
  reachedConsistency == true

standby MVCC / lock 快照可用：
  standbyState == STANDBY_SNAPSHOT_READY

最终开关：
  SharedHotStandbyActive == true
```
这三个状态不在同一个文件里。
也不由同一个事件一次性设置。
这正是这节课的阅读重点。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/startup.c` | `StartupProcessMain()`、startup process 信号、退出和 promotion signal。 |
| 2 | `src/backend/access/transam/xlog.c` | `StartupXLOG()` 编排恢复、设置 `SharedRecoveryState`、结束恢复、移除 signal 文件。 |
| 3 | `src/backend/access/transam/xlogrecovery.c` | `InitWalRecovery()`、`PerformWalRecovery()`、`ReadRecord()`、`WaitForWALToBecomeAvailable()`、`CheckRecoveryConsistency()`。 |
| 4 | `src/include/access/xlogrecovery.h` | `XLogRecoveryCtlData`、`reachedConsistency`、`StandbyMode`、pause / promotion / replay LSN API。 |
| 5 | `src/include/access/xlogutils.h` | `standbyState` 枚举和 `InHotStandby` 宏。 |
| 6 | `src/backend/storage/ipc/standby.c` | `InitRecoveryTransactionEnvironment()`、`standby_redo()`、running-xacts WAL record 如何推进 standby snapshot。 |
| 7 | `src/backend/storage/ipc/procarray.c` | `ProcArrayInitRecovery()`、`ProcArrayApplyRecoveryInfo()`、`KnownAssignedXids` 状态。 |
| 8 | `src/backend/replication/walreceiverfuncs.c` | `RequestXLogStreaming()` 如何写入 walreceiver 共享状态并请求 postmaster 启动。 |
| 9 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()`、`WalRcvWaitForStartPosition()`、flush 位置和状态视图。 |
| 10 | `src/backend/access/transam/xlogfuncs.c` | `pg_is_in_recovery()`、`pg_last_wal_replay_lsn()`、`pg_promote()` 等诊断入口。 |
推荐阅读顺序不是从 `StartupXLOG()` 顶部一直读到底。
更有效的方式是沿状态转换读：
```text
signal file
  -> ArchiveRecoveryRequested / StandbyModeRequested
  -> InArchiveRecovery / StandbyMode
  -> currentSource
  -> reachedConsistency
  -> standbyState
  -> SharedHotStandbyActive
```
每个状态都回答三个问题：
```text
谁设置它？
谁消费它？
它代表请求、当前事实，还是对其他进程可见的共享事实？
```
## 4. 第一层状态：恢复请求不是恢复事实
`xlogrecovery.c` 中有两组容易混淆的变量。
第一组描述 archive recovery：
```text
ArchiveRecoveryRequested
InArchiveRecovery
```
第二组描述 standby mode：
```text
StandbyModeRequested
StandbyMode
```
它们不是同义词。
`ArchiveRecoveryRequested` 表示启动时发现了恢复请求。
它来自 `standby.signal` 或 `recovery.signal`。
`InArchiveRecovery` 表示当前 recovery 逻辑已经切入 archive recovery。
一个节点可以出现这种状态：
```text
ArchiveRecoveryRequested = true
InArchiveRecovery = false
```
这不是矛盾。
源码注释明确说，这表示当前还在用 `pg_wal` 中已有 WAL 做 crash recovery。
当 replay 走到本地 `pg_wal` 末尾后，才会切入 archive recovery。
`StandbyModeRequested` 表示启动时发现了 `standby.signal`。
`StandbyMode` 表示当前已经进入 standby mode。
同样，二者之间可能存在时间差。
这个时间差主要出现在无法一开始就知道 consistency 边界的场景。
例如没有 `backup_label`，也没有足够的 `minRecoveryPoint` / `backupEndPoint` 信息时，PostgreSQL 会先用本地 `pg_wal` 做 crash recovery。
到达本地 WAL 末尾后，它再把恢复切入 archive / standby recovery。
这解释了为什么课程不能把“存在 standby.signal”直接等同于“已经在 streaming recovery”。
`standby.signal` 只是请求。
实际是否 streaming，要看后续 WAL source 状态机是否进入 `XLOG_FROM_STREAM`。
### signal 文件优先级
`readRecoverySignalFile()` 在 `xlogrecovery.c`。
它检查：
```text
standby.signal
recovery.signal
```
如果两者都存在，`standby.signal` 优先。
因此状态变成：
```text
standby.signal:
  StandbyModeRequested = true
  ArchiveRecoveryRequested = true

recovery.signal:
  StandbyModeRequested = false
  ArchiveRecoveryRequested = true
```
这就是两个文件的语义差异。
`recovery.signal` 是 archive recovery / PITR。
它可以恢复到一个 target，然后结束恢复并成为 primary。
`standby.signal` 是 standby mode。
它会在没有 WAL 时继续等待，除非 promotion 被触发。
### 参数校验的真实边界
`validateRecoveryParameters()` 进一步区分两种请求。
如果是 standby mode，`primary_conninfo` 和 `restore_command` 都可以为空。
源码只发出 WARNING：
```text
specified neither "primary_conninfo" nor "restore_command"
```
这种 standby 会轮询 `pg_wal`，等待外部把 WAL 文件放进来。
如果不是 standby mode，而只是 `recovery.signal`，则必须有 `restore_command`。
否则它无法知道缺失 WAL 时从哪里取 archive，会 FATAL。
这个差异来自目标不同：
```text
standby:
  可以长期等待 WAL

PITR / archive recovery:
  必须按恢复目标推进，缺少恢复来源是配置错误
```
另一个边界是 single-user server。
`StandbyModeRequested && !IsUnderPostmaster` 会 FATAL。
standby 需要 walreceiver、postmaster 信号和 latch 协作，单用户模式无法提供这些进程。
## 5. 第二层状态：crash recovery 与 archive recovery 的切换
`StartupXLOG()` 负责启动恢复总流程。
它先读 `pg_control`，根据 `ControlFile->state` 打日志。
如果上次不是 clean shutdown，还会：
```text
RemoveTempXlogFiles()
SyncDataDirectory()
```
然后调用：
```text
InitWalRecovery(ControlFile, &wasShutdown, &haveBackupLabel, &haveTblspcMap)
```
`InitWalRecovery()` 做几个关键决定：
```text
选择 checkpoint / redo start
识别 backup_label / tablespace_map
识别 standby.signal / recovery.signal
决定是否 InRecovery
决定是否立刻 InArchiveRecovery
决定 minRecoveryPoint / backup end 边界
```
如果存在 `backup_label`，PostgreSQL 知道 base backup 从哪个 checkpoint 开始，通常能立刻进入 archive recovery。
路径大致是：
```text
read_backup_label()
  -> InArchiveRecovery = true
  -> if StandbyModeRequested: EnableStandbyMode()
  -> 读取 backup_label 指向的 checkpoint
```
如果没有 `backup_label`，源码会看 `pg_control`。
当 `ArchiveRecoveryRequested` 为真，并且能从 control file 推断出恢复一致性边界时，也可以立刻进入 archive recovery。
典型条件包括：
```text
ControlFile->minRecoveryPoint 有效
ControlFile->backupEndRequired
ControlFile->backupEndPoint 有效
ControlFile->state == DB_SHUTDOWNED
```
如果 archive recovery 被请求，但暂时无法知道要 replay 到哪里才一致，源码选择先做 crash recovery。
注释里的理由很直接：
```text
先 replay pg_wal 中已有 WAL；
到本地 WAL 末尾再进入 archive recovery。
```
这就是下面状态的含义：
```text
ArchiveRecoveryRequested = true
InArchiveRecovery = false
SharedRecoveryState = RECOVERY_STATE_CRASH
```
它经常发生在“用文件系统快照或不完整外部流程做 base backup”这类场景。
如果你只看 `pg_is_in_recovery()`，它只告诉你 recovery 仍在进行。
它不会告诉你当前是 crash recovery 阶段还是 archive recovery 阶段。
这部分状态主要在 startup process 内部，外部只能从日志和行为推断。
## 6. 第三层状态：StartupXLOG 的主编排
`StartupXLOG()` 不是单纯调用 redo。
在 `PerformWalRecovery()` 前，它先初始化很多子系统。
对本课最重要的是这些步骤：
```text
InitWalRecovery()
  -> 初始化 TransamVariables / CLOG / MultiXact / CommitTs
  -> StartupReplicationSlots()
  -> StartupLogicalDecodingStatus()
  -> restoreTwoPhaseData()
  -> 设置 XLogCtl->SharedRecoveryState
  -> UpdateControlFile()
  -> ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP)
  -> DeleteAllExportedSnapshotFiles()
  -> 初始化 Hot Standby recovery transaction environment
  -> PerformWalRecovery()
```
`SharedRecoveryState` 在 `xlog.c` 的 `XLogCtlData` 中。
它有三种值：
```text
RECOVERY_STATE_CRASH
RECOVERY_STATE_ARCHIVE
RECOVERY_STATE_DONE
```
普通 backend 调用 `RecoveryInProgress()` 时看的是这个共享状态。
`pg_is_in_recovery()` 也是通过 `RecoveryInProgress()` 返回。
这意味着：
```text
pg_is_in_recovery() = true
```
只说明还没进入 `RECOVERY_STATE_DONE`。
它不说明：
```text
是否已经 consistent
是否已经允许 hot standby 查询
当前 WAL 来源是 archive 还是 streaming
walreceiver 是否连接
```
`StartupXLOG()` 在进入 redo 前还会处理 Hot Standby 初始化。
条件是：
```text
ArchiveRecoveryRequested && EnableHotStandby
```
这里用的是 `ArchiveRecoveryRequested`，不是 `InArchiveRecovery`。
原因是 standby snapshot machinery 要在 replay 早期就开始记录 primary 上的事务变化。
如果等到真正切入 archive recovery 再初始化，可能错过本地 `pg_wal` 中已经 replay 的 standby 相关 WAL record。
初始化包括：
```text
InitRecoveryTransactionEnvironment()
ProcArrayInitRecovery(...)
StartupSUBTRANS(...)
```
如果恢复从 shutdown checkpoint 开始，源码知道 primary 当时没有普通运行事务。
于是可以构造一个空的 `RunningTransactionsData`，并调用 `ProcArrayApplyRecoveryInfo()`。
这可能让 `standbyState` 很早就进入 `STANDBY_SNAPSHOT_READY`。
但即使 snapshot ready，也还不能查询。
还需要物理一致性门槛 `reachedConsistency`。
## 7. 第四层状态：standbyState 不是全局 truth
`standbyState` 定义在 `src/include/access/xlogutils.h`：
```text
STANDBY_DISABLED
STANDBY_INITIALIZED
STANDBY_SNAPSHOT_PENDING
STANDBY_SNAPSHOT_READY
```
源码注释强调：
```text
standbyState only valid in the startup process
```
其他进程中它通常是 `STANDBY_DISABLED`。
所以不能在普通 backend 里用 `standbyState` 判断 hot standby 是否可用。
普通进程应看：
```text
HotStandbyActive()
```
它读取的是 `XLogRecoveryCtl->SharedHotStandbyActive`。
这就是第一个重要边界：
```text
standbyState:
  startup process 本地判断 replay 是否有足够 running-xacts 信息

SharedHotStandbyActive:
  其他进程可见的“可以运行 hot standby query”共享事实
```
### STANDBY_INITIALIZED
`InitRecoveryTransactionEnvironment()` 在 `standby.c`。
它初始化 recovery lock hash，注册 startup process 的 shared invalidation backend，并为 startup process 建立一个 virtual transaction lock entry。
最后设置：
```text
standbyState = STANDBY_INITIALIZED
```
这个状态的含义是：
```text
我们已经准备好跟踪 primary 的 running xids 和 AccessExclusiveLocks；
但还没有足够信息构造可用 standby snapshot。
```
此时 redo function 必须开始维护状态。
但 postmaster 仍不能放只读 backend 进来。
### STANDBY_SNAPSHOT_PENDING
`ProcArrayApplyRecoveryInfo()` 处理 `RunningTransactionsData`。
如果 running-xacts 信息里 subxid 溢出，standby 无法完整知道所有 subtransaction。
源码会进入：
```text
STANDBY_SNAPSHOT_PENDING
```
同时记录：
```text
standbySnapshotPendingXmin
procArray->lastOverflowedXid
```
pending 的语义不是“完全不能 replay”。
它是：
```text
redo 可以继续；
KnownAssignedXids 继续维护；
但还不能对外提供可正确解释 MVCC 的 snapshot。
```
后续如果看到非 overflow 的 running-xacts 记录，或者 oldest running xid 已经越过 pending 边界，就可以进入 ready。
### STANDBY_SNAPSHOT_READY
如果 running-xacts 信息完整，`ProcArrayApplyRecoveryInfo()` 会设置：
```text
standbyState = STANDBY_SNAPSHOT_READY
```
这表示 startup process 已经有足够信息维护 standby 上的 `KnownAssignedXids`。
读-only backend 后续拿 snapshot 时，才能把 primary 上未完成事务纳入可见性判断。
但这仍然不是最终放行条件。
它只是 hot standby 的 snapshot 条件。
最终还要由 `CheckRecoveryConsistency()` 同时检查 physical consistency。
## 8. 第五层状态：reachedConsistency 的真正含义
`reachedConsistency` 是 `xlogrecovery.c` 中的全局变量。
注释给出的语义比名字更严格：
```text
system is internally consistent
all WAL has been replayed up to a certain point
there is no trace of later actions on disk
```
它不是“已经追上 primary”。
也不是“walreceiver 已经 streaming”。
它表示当前数据目录已经越过 base backup / min recovery point 的安全边界。
`CheckRecoveryConsistency()` 是设置它的关键函数。
函数开头有一个非常重要的判断：
```text
if (!XLogRecPtrIsValid(minRecoveryPoint))
    return;
```
在 crash recovery 中，`minRecoveryPoint` 无效。
因此 crash recovery 不会设置 `reachedConsistency`。
crash recovery 的一致状态是“replay 完所有本地 WAL 后结束恢复”。
它不会在中途开放 hot standby。
这解释了源码注释：
```text
During crash recovery, we don't reach a consistent state until we've replayed all the WAL.
```
在 archive recovery 中，`minRecoveryPoint` 有效。
`CheckRecoveryConsistency()` 会比较：
```text
minRecoveryPoint <= lastReplayedEndRecPtr
```
并且要求：
```text
!backupEndRequired
```
如果 base backup 还要求 `XLOG_BACKUP_END`，则即使 LSN 看起来越过某些位置，也不能宣布 consistent。
因为 backup 期间的数据目录可能包含需要由后续 WAL 修复的中间状态。
当条件满足后，它会：
```text
XLogCheckInvalidPages()
CheckTablespaceDirectory()
reachedConsistency = true
SendPostmasterSignal(PMSIGNAL_RECOVERY_CONSISTENT)
log "consistent recovery state reached at ..."
```
这一步只是告诉 postmaster：
```text
数据目录已经物理一致。
```
它还没有放行连接。
真正放行还需要下一段。
## 9. 第六层状态：SharedHotStandbyActive 才是查询入口
`CheckRecoveryConsistency()` 的第二个门槛是：
```text
standbyState == STANDBY_SNAPSHOT_READY
!LocalHotStandbyActive
reachedConsistency
IsUnderPostmaster
```
四个条件都满足后，startup process 设置：
```text
XLogRecoveryCtl->SharedHotStandbyActive = true
LocalHotStandbyActive = true
SendPostmasterSignal(PMSIGNAL_BEGIN_HOT_STANDBY)
```
`SharedHotStandbyActive` 在 `XLogRecoveryCtlData` 中，受 `info_lck` spinlock 保护。
普通进程通过 `HotStandbyActive()` 读取它。
`HotStandbyActive()` 本地缓存一旦看到 true，就不再反复读共享内存。
因为 hot standby 只会从 false 变 true，不会在 recovery 期间变回 false。
postmaster 收到 `PMSIGNAL_BEGIN_HOT_STANDBY` 后，会打日志：
```text
database system is ready to accept read-only connections
```
然后把状态更新为 `PM_HOT_STANDBY`，并设置：
```text
connsAllowed = true
```
所以读者要记住这条因果链：
```text
reachedConsistency = true
  不等于允许查询

standbyState = STANDBY_SNAPSHOT_READY
  不等于允许查询

两者都满足
  -> SharedHotStandbyActive = true
  -> PMSIGNAL_BEGIN_HOT_STANDBY
  -> postmaster 允许只读连接
```
如果你看到日志只有：
```text
consistent recovery state reached at ...
```
但还没看到：
```text
database system is ready to accept read-only connections
```
就应该怀疑 standby snapshot 还没 ready，或者 postmaster 还没处理 begin hot standby signal。
## 10. 主流程源码 walkthrough
先看启动入口。
`StartupProcessMain()` 做的是 auxiliary process 初始化、信号处理和 timeout 注册。
然后直接进入：
```text
StartupXLOG()
```
可以把 `StartupXLOG()` 分成三段看。
第一段是恢复前准备：
```text
读取 pg_control
校验数据目录和 pg_wal 结构
如果上次 crash，则清理临时 WAL 并 SyncDataDirectory()
InitWalRecovery()
初始化事务、SLRU、复制槽、逻辑解码、两阶段事务状态
设置 SharedRecoveryState
UpdateControlFile()
准备 Hot Standby tracking
```
第二段是 WAL replay：
```text
PerformWalRecovery()
```
第三段是结束恢复：
```text
FinishWalRecovery()
选择新 timeline
移除 standby.signal / recovery.signal
写 timeline history
写 end-of-recovery 或 checkpoint 相关 WAL
SharedRecoveryState = RECOVERY_STATE_DONE
ShutdownRecoveryTransactionEnvironment()
唤醒 walsender / 请求 checkpoint
```
本节重点是第二段。
`PerformWalRecovery()` 先初始化 replay 位置：
```text
lastReplayedReadRecPtr
lastReplayedEndRecPtr
lastReplayedTLI
replayEndRecPtr
replayEndTLI
recoveryPauseState
```
然后发送：
```text
PMSIGNAL_RECOVERY_STARTED
```
接着马上调用一次：
```text
CheckRecoveryConsistency()
```
这解释了一个看似奇怪的现象：
如果从 shutdown checkpoint 或已经足够一致的位置启动，read-only 连接可能在真正 replay 很少 WAL 后就开放。
因为 consistency 不是“必须 replay 一大段 WAL”。
它是“是否已经越过 minRecoveryPoint，并且 standby snapshot ready”。
随后 `PerformWalRecovery()` 找到第一条要 replay 的 WAL record。
如果 `RedoStartLSN < CheckPointLoc`，它从 redo LSN 开始读。
否则从 checkpoint 后的下一条 record 开始。
主循环是：
```text
for each WAL record:
  ProcessStartupProcInterrupts()
  如果 pause 被请求，调用 recoveryPausesHere(false)
  如果 recovery target 在本 record 前停止，break
  如果 recovery_min_apply_delay 要求延迟，等待
  ApplyWalRecord()
  唤醒 wait-for-LSN 等待者
  如果 recovery target 在本 record 后停止，break
  ReadRecord() 读取下一条 record
```
注意 `ReadRecord()` 在循环尾。
这意味着“缺 WAL”不是 apply 阶段才知道，而是在试图读下一条 record 时进入 WAL source 状态机。
`ApplyWalRecord()` 负责几件和本课直接相关的事：
```text
AdvanceNextFullTransactionIdPastXid(record->xl_xid)
必要时处理 timeline switch
设置 replayEndRecPtr / replayEndTLI
如果 standbyState >= STANDBY_INITIALIZED，则 RecordKnownAssignedTransactionIds()
执行 xlogrecovery_redo() 或 resource manager redo
更新 lastReplayedReadRecPtr / lastReplayedEndRecPtr / lastReplayedTLI
必要时唤醒 walsender / walreceiver apply reply
CheckRecoveryConsistency()
```
所以 hot standby 开放检查不是只在循环结束发生。
它在每条 WAL record apply 后都有机会发生。
这也是为什么 `RUNNING_XACTS` record replay 后，紧接着可能开放只读连接：
```text
standby_redo()
  -> ProcArrayApplyRecoveryInfo()
     -> standbyState = STANDBY_SNAPSHOT_READY
ApplyWalRecord()
  -> CheckRecoveryConsistency()
     -> SharedHotStandbyActive = true
```
这条链路是本节最核心的源码 walkthrough。
## 11. WAL source 状态机：archive、pg_wal、stream 的推进
`ReadRecord()` 调 `XLogPrefetcherReadRecord()`。
当 reader 需要某页 WAL，而当前文件不存在或 stream flush 位置不够时，会进入：
```text
XLogPageRead()
  -> WaitForWALToBecomeAvailable()
```
`WaitForWALToBecomeAvailable()` 的注释直接说：
```text
Standby mode is implemented by a state machine
```
它的核心状态是：
```text
currentSource
lastSourceFailed
```
`currentSource` 的取值是：
```text
XLOG_FROM_ARCHIVE
XLOG_FROM_PG_WAL
XLOG_FROM_STREAM
XLOG_FROM_ANY
```
如果还没有进入 archive recovery：
```text
currentSource = XLOG_FROM_PG_WAL
```
这对应 crash recovery。
如果已经进入 archive recovery，初始会优先：
```text
currentSource = XLOG_FROM_ARCHIVE
```
这里的 archive 分支实际会通过 `XLogFileReadAnyTLI()` 尝试 restore command 或已有文件。
状态机的正常推进可以画成：
```text
archive / pg_wal
  成功:
    读取 WAL page
  失败:
    检查 promotion trigger
    如果不是 standby -> 返回失败
    如果是 standby -> 进入 stream

stream
  成功:
    等 walreceiver flush 到 RecPtr
    从 pg_wal 打开已流入的 segment
  失败:
    shutdown 或 reset walreceiver
    rescan timeline
    sleep wal_retrieve_retry_interval
    回到 archive
```
这说明 streaming recovery 不是独立大循环。
它只是 `WaitForWALToBecomeAvailable()` 在 archive / pg_wal 找不到 WAL 后进入的一个 source state。
### 为什么 promote 要在失败后检查
在 archive / pg_wal source 失败后，源码才检查 promotion trigger。
注释解释了原因：
```text
when you promote, we still finish replaying as much as we can from archive and pg_wal before failover
```
promotion 不是立刻丢弃已在本地的 WAL。
PostgreSQL 仍会尽量 replay 已经可用的 WAL，然后才结束 recovery。
同样，在 stream 等待过程中发现 trigger，也不是马上返回成功。
源码会把它当作 source failure，让状态机回到 archive / pg_wal，再 replay 已经流入本地的 WAL。
这减少了 promotion 时丢失已经接收 WAL 的窗口。
## 12. walreceiver 请求与等待
当 source 进入 `XLOG_FROM_STREAM`，startup process 可能调用：
```text
RequestXLogStreaming(tli, ptr, PrimaryConnInfo, PrimarySlotName, wal_receiver_create_temp_slot)
```
这个函数在 `src/backend/replication/walreceiverfuncs.c`。
它不是网络接收代码。
它只是写入 walreceiver 共享状态：
```text
walrcv->walRcvState
walrcv->receiveStart
walrcv->receiveStartTLI
walrcv->flushedUpto
walrcv->receivedTLI
walrcv->writtenUpto
walrcv->slotname
walrcv->conninfo
```
如果 walreceiver 停止，它设置：
```text
WALRCV_STARTING
SendPostmasterSignal(PMSIGNAL_START_WALRECEIVER)
```
如果 walreceiver 正在等待新指令，它设置：
```text
WALRCV_RESTARTING
SetLatch(walreceiver procLatch)
```
`RequestXLogStreaming()` 还会把请求 LSN 向下对齐到 WAL segment 起点。
源码注释说，这样能避免创建前半段没有 record 的破损 segment。
这也是为什么 `pg_stat_wal_receiver.receive_start_lsn` 可能比 startup process 逻辑上需要的 record LSN 更早。
walreceiver 进程启动后进入 `WalReceiverMain()`。
它连接 upstream，执行 identify system，必要时读取 timeline history，必要时创建 temporary slot，然后调用 walreceiver provider 的 `startstreaming`。
收到 WAL 后：
```text
XLogWalRcvWrite()
  -> 写入 pg_wal
  -> 更新 writtenUpto

XLogWalRcvFlush()
  -> flush 到磁盘
  -> 更新 flushedUpto / latestChunkStart / receivedTLI
  -> WakeupRecovery()
```
startup process 在 `WaitForWALToBecomeAvailable()` 中观察：
```text
flushedUpto = GetWalRcvFlushRecPtr(&latestChunkStart, &receiveTLI)
```
只有：
```text
RecPtr < flushedUpto
receiveTLI == curFileTLI
```
才认为 WAL 已经可读。
所以 walreceiver 的“收到”不是 apply 的充分条件。
startup process 要看到 flush 位置越过需要的 record。
如果还没到，它等待：
```text
WAIT_EVENT_RECOVERY_WAL_STREAM
```
等待前它会做两件重要的事：
```text
WalRcvRequestApplyReply()
KnownAssignedTransactionIdsIdleMaintenance()
```
前者让 upstream 的 `pg_stat_replication` 和 remote_apply 等待者尽快看到最新 apply 位置。
后者在缺 WAL 或等待 WAL 时压缩 / 维护 `KnownAssignedXids`，避免 standby 长时间等待导致数组状态膨胀。
## 13. consistent state 与 hot standby 查询何时开放
把前面的状态合并成一条时间线：
```text
postmaster 启动 startup process
  -> StartupXLOG()
  -> InitWalRecovery()
     -> 发现 standby.signal
     -> ArchiveRecoveryRequested = true
     -> StandbyModeRequested = true
  -> 设置 SharedRecoveryState = CRASH 或 ARCHIVE
  -> 如果 ArchiveRecoveryRequested && hot_standby
     -> InitRecoveryTransactionEnvironment()
     -> standbyState = STANDBY_INITIALIZED
  -> PerformWalRecovery()
     -> ReadRecord()
        -> archive / pg_wal / stream 状态机
     -> ApplyWalRecord()
        -> standby_redo(RUNNING_XACTS)
           -> ProcArrayApplyRecoveryInfo()
              -> standbyState = STANDBY_SNAPSHOT_READY
        -> 更新 lastReplayedEndRecPtr
        -> CheckRecoveryConsistency()
           -> reachedConsistency = true
           -> SharedHotStandbyActive = true
           -> PMSIGNAL_BEGIN_HOT_STANDBY
  -> postmaster
     -> PM_HOT_STANDBY
     -> connsAllowed = true
```
最容易出错的判断是把其中任意一个中间点当成最终点。
下面几个判断都不充分：
```text
standby.signal 存在
walreceiver 正在 streaming
pg_is_in_recovery() = true
pg_last_wal_replay_lsn() 非空
consistent recovery state reached
standbyState ready
```
真正充分的内部条件是：
```text
reachedConsistency
standbyState == STANDBY_SNAPSHOT_READY
SharedHotStandbyActive
postmaster 已处理 PMSIGNAL_BEGIN_HOT_STANDBY
```
外部能直接看到的是最后一层：
```text
能连上 standby 并执行 read-only query
日志出现 "database system is ready to accept read-only connections"
```
外部不能直接看到 `standbyState`。
只能通过日志、断点或间接现象推断。
## 14. pause 的边界
recovery pause 由共享状态：
```text
XLogRecoveryCtl->recoveryPauseState
```
表示。
取值定义在 `xlogrecovery.h`：
```text
RECOVERY_NOT_PAUSED
RECOVERY_PAUSE_REQUESTED
RECOVERY_PAUSED
```
`PerformWalRecovery()` 在每条 WAL record 前检查 pause。
`WaitForWALToBecomeAvailable()` 的长循环末尾也会检查 pause。
但 `recoveryPausesHere()` 开头有一个重要判断：
```text
if (!LocalHotStandbyActive)
    return;
```
这说明：
```text
没有开放 hot standby 之前，不会真正停在 recovery pause。
```
原因很实际。
pause 需要用户连接进来执行 resume 或 promote。
如果还没允许连接，停在 pause 状态只会制造无法恢复的等待。
因此 `recovery_target_action=pause` 在 `hot_standby=off` 时会被 `validateRecoveryParameters()` 改成 shutdown。
如果 pause 期间触发 promotion，`SetPromoteIsTriggered()` 会调用：
```text
SetRecoveryPause(false)
```
也就是说 promotion 会解除 pause。
诊断时要区分：
```text
等待 WAL:
  WAIT_EVENT_RECOVERY_WAL_STREAM
  或 WAIT_EVENT_RECOVERY_RETRIEVE_RETRY_INTERVAL

暂停 replay:
  recoveryPauseState = RECOVERY_PAUSED
  日志 "recovery has paused"
```
二者都可能让 `pg_last_wal_replay_lsn()` 不动。
但原因完全不同。
## 15. promotion 的边界
SQL 函数 `pg_promote()` 在 `xlogfuncs.c`。
它先检查：
```text
RecoveryInProgress()
```
如果不在 recovery，直接 ERROR。
然后创建：
```text
promote signal file
```
并向 postmaster 发 `SIGUSR1`。
postmaster 再通过 startup process 的 `SIGUSR2` handler 设置本地标志。
startup process 在 `CheckForStandbyTrigger()` 中检查：
```text
IsPromoteSignaled()
CheckPromoteSignal()
```
发现 promote 后：
```text
log "received promote request"
RemovePromoteSignalFiles()
ResetPromoteSignaled()
SetPromoteIsTriggered()
```
`SetPromoteIsTriggered()` 设置共享状态：
```text
SharedPromoteIsTriggered = true
```
并解除 recovery pause。
promotion 的执行点和 WAL source 状态机强相关。
在 archive / pg_wal source 失败后，startup process 才检查 trigger 并退出等待。
在 stream 等待中发现 trigger 时，它先把 stream 看作失败，回到本地 WAL source，把已经收到的 WAL 尽量 replay。
之后 `ReadRecord()` 返回 NULL，`PerformWalRecovery()` 离开主循环。
`StartupXLOG()` 进入结束恢复阶段。
如果是 archive recovery，它会：
```text
选择 newTLI
XLogInitNewTimeline()
durable_unlink(standby.signal)
durable_unlink(recovery.signal)
writeTimeLineHistory()
log "archive recovery complete"
```
最后设置：
```text
InRecovery = false
SharedRecoveryState = RECOVERY_STATE_DONE
```
这时 `pg_is_in_recovery()` 才会变成 false。
## 16. 缺 WAL 的边界
缺 WAL 时要分三类看。
第一类是 crash recovery。
如果没有 archive recovery 请求，`ReadRecord()` 读不到下一条 valid record，恢复就结束。
这对应普通 crash recovery：
```text
replay 本地 pg_wal 到末尾
finish recovery
进入 production
```
第二类是 non-standby archive recovery。
如果 `recovery.signal` 请求 PITR，但缺少 WAL 或没达到 recovery target，会报错。
典型错误包括：
```text
recovery ended before configured recovery target was reached
WAL ends before consistent recovery point
WAL ends before end of online backup
```
这是正确性要求。
PITR 不能在缺失目标 WAL 时假装成功。
第三类是 standby mode。
如果 archive / pg_wal / stream 都拿不到 WAL，状态机会：
```text
等待 wal_retrieve_retry_interval
log "waiting for WAL to become available at ..."
回到 archive source 重试
```
如果 `primary_conninfo` 为空，它不会启动 walreceiver。
它仍会轮询 archive / pg_wal。
这对应人工投递 WAL 或共享归档目录的部署方式。
如果 `primary_conninfo` 存在但连接失败，walreceiver 失败后状态机会回到 archive / pg_wal，再 sleep，再重试 stream。
这就是 standby 在缺 WAL 时“不退出”的原因。
它是由 `StandbyMode` 控制的。
不是由 walreceiver 是否存在控制的。
## 17. 生命周期 / ownership / cleanup
恢复状态的 owner 是 startup process。
但不同状态的可见范围不同。
`ArchiveRecoveryRequested`、`InArchiveRecovery`、`StandbyModeRequested`、`StandbyMode` 只在 startup process 中有效。
`currentSource`、`lastSourceFailed`、`flushedUpto` 本地副本也只属于 recovery reader。
`standbyState` 只在 startup process 中有语义。
`XLogRecoveryCtlData` 是共享内存。
它包含：
```text
SharedHotStandbyActive
SharedPromoteIsTriggered
recoveryWakeupLatch
lastReplayedReadRecPtr
lastReplayedEndRecPtr
lastReplayedTLI
replayEndRecPtr
replayEndTLI
recoveryLastXTime
currentChunkStartTime
recoveryPauseState
recoveryNotPausedCV
info_lck
```
`WalRcvData` 也是共享内存。
startup process 写入 `receiveStart` / `receiveStartTLI`。
walreceiver 写入 `writtenUpto` / `flushedUpto` / `receivedTLI`。
postmaster 负责按 `PMSIGNAL_START_WALRECEIVER` 启动 walreceiver。
恢复结束时的 cleanup 也有顺序。
`StartupXLOG()` 在 `SharedRecoveryState = RECOVERY_STATE_DONE` 之后才调用：
```text
ShutdownRecoveryTransactionEnvironment()
```
源码注释说明这是为了避免普通 backend 在 `RecoveryInProgress()` 已经返回 false 后仍依赖 recovery-time `KnownAssignedXids`。
特别是 prepared transactions 需要在 normal operation 的快照中仍然可见。
所以 cleanup 不是“redo 结束立刻释放所有 standby 状态”。
它必须等共享恢复状态切换完、prepared transaction 恢复完，再清理 recovery transaction environment。
## 18. 正确性机制层次
这条链路依赖多层正确性机制。
第一层是 WAL 顺序。
startup process 只按 WAL 顺序推进 `lastReplayedEndRecPtr`。
只有 record redo 成功后，才更新 last replayed 指针。
`pg_last_wal_replay_lsn()` 读取的是这个成功 replay 后的位置。
第二层是 base backup consistency。
`minRecoveryPoint`、`backupEndPoint`、`backupEndRequired` 决定是否已经越过 base backup 安全边界。
`reachedConsistency` 只有在这个边界满足后才会设置。
第三层是 MVCC snapshot。
Hot Standby 需要 `KnownAssignedXids` 反映 primary 上仍未完成的事务。
这个状态由 `RUNNING_XACTS` WAL record 和后续 xid assignment / commit / abort replay 维护。
第四层是 lock 和 invalidation。
`standby.c` 会 replay AccessExclusiveLocks，处理 invalidation，并在 recovery conflict 时取消或等待 standby query。
这保证只读查询不会在 replay DDL、drop database、cleanup buffer pin 等场景下破坏 recovery。
第五层是进程间可见性。
`SharedHotStandbyActive`、`SharedPromoteIsTriggered`、replay LSN 等共享字段通过 spinlock 或 atomic / latch 协作。
普通 backend 不读 startup process 本地变量。
第六层是 postmaster gating。
即使 startup process 设置共享状态，也要通过 `PMSIGNAL_BEGIN_HOT_STANDBY` 让 postmaster 改成 `PM_HOT_STANDBY` 并允许连接。
这个分层可以压缩成一个不变量：
```text
WAL replay position 证明“已经做了什么”；
minRecoveryPoint 证明“物理上可以从这里开始一致”；
standbyState 证明“MVCC 快照可解释”；
SharedHotStandbyActive + postmaster state 才证明“外部 backend 可以进来查询”。
```
## 19. 成本、资源与跨模块传播
startup recovery 状态机的成本主要随 WAL 量、缺 WAL 时间、running-xacts 规模和冲突查询数量扩张。
WAL 量越大，`PerformWalRecovery()` 的循环越长。
每条 record 都可能更新 replay 指针、唤醒 LSN waiters、处理 standby state、触发 resource manager redo。
如果 standby 落后很多，热点通常不在 walreceiver 连接本身，而在 redo IO、buffer 修复、SLRU 扩展、full page image 处理和 recovery conflict。
缺 WAL 时间越长，状态机会在 archive / pg_wal / stream / sleep 之间循环。
这会产生周期性日志：
```text
waiting for WAL to become available at ...
```
同时 startup process 会做 `KnownAssignedTransactionIdsIdleMaintenance()`。
这说明等待不是纯睡眠。
它仍会维护 recovery-time xid 数据结构。
running-xacts 规模越大，`ProcArrayApplyRecoveryInfo()` 初始化 `KnownAssignedXids` 的成本越高。
如果 subxid overflow，standby 可能停在 `STANDBY_SNAPSHOT_PENDING`，等待后续信息把 snapshot 补完整。
这会延迟 hot standby 开放。
只读查询越多，recovery conflict 的成本越明显。
startup process replay 某些 WAL record 时可能需要取消或等待冲突查询。
例如 snapshot conflict、AccessExclusiveLock conflict、buffer pin conflict。
这些路径不改变“何时开放查询”的初始门槛，但会影响开放后 replay 能否继续追上 primary。
跨模块传播路径可以这样记：
```text
walreceiver flush 慢
  -> WaitForWALToBecomeAvailable() 等 RECOVERY_WAL_STREAM
  -> pg_last_wal_replay_lsn 不前进
  -> primary 上 remote_apply 可能等待 apply feedback

RUNNING_XACTS 缺失或 overflow
  -> standbyState 不能 ready
  -> reachedConsistency 可能已 true
  -> postmaster 仍不允许 read-only 连接

read-only query 持有 snapshot 或 buffer pin
  -> recovery conflict
  -> replay 停顿或取消 query
  -> apply LSN 滞后
```
## 20. 观测与诊断入口
诊断 standby 启动恢复，不要只看一个指标。
推荐按这条顺序看：
```text
server log
  -> pg_is_in_recovery()
  -> pg_last_wal_replay_lsn()
  -> pg_stat_wal_receiver
  -> pause / promote 状态
  -> wait event
```
日志是最重要入口。
关键日志包括：
```text
entering standby mode
starting archive recovery
redo starts at ...
consistent recovery state reached at ...
database system is ready to accept read-only connections
waiting for WAL to become available at ...
received promote request
selected new timeline ID: ...
archive recovery complete
```
这些日志对应不同状态。
`entering standby mode` 对应 `StandbyMode` 启用。
`consistent recovery state reached` 对应 `reachedConsistency` 设置。
`database system is ready to accept read-only connections` 对应 postmaster 已经进入 hot standby。
`waiting for WAL to become available` 对应 WAL source 状态机所有来源都暂时失败。
SQL 入口：
```sql
SELECT pg_is_in_recovery();
SELECT pg_last_wal_replay_lsn();
SELECT pg_last_xact_replay_timestamp();
SELECT * FROM pg_stat_wal_receiver;
```
`pg_is_in_recovery()` 来自 `RecoveryInProgress()`。
它只能告诉你 `SharedRecoveryState != RECOVERY_STATE_DONE`。
它不能告诉你是否已经可以查询。
`pg_last_wal_replay_lsn()` 来自 `GetXLogReplayRecPtr()`。
它返回 last successfully replayed end LSN。
如果它不动，可能是：
```text
没有新 WAL
walreceiver 没有 flush 到需要位置
recovery pause
recovery conflict
redo 某条记录耗时
startup process 正在等待 archive restore_command
```
不能直接归因于网络。
`pg_stat_wal_receiver` 只在 walreceiver 进程存在时有行。
它可以看到：
```text
status
receive_start_lsn
receive_start_tli
written_lsn
flushed_lsn
received_tli
last_msg_send_time
last_msg_receipt_time
latest_end_lsn
slot_name
sender_host
sender_port
conninfo
```
如果 `pg_stat_wal_receiver` 为空，不一定代表 recovery 停了。
可能当前 source 在 archive / pg_wal。
也可能 `primary_conninfo` 为空，standby 正在轮询本地 WAL。
也可能 walreceiver 在 timeline 切换或等待重启。
诊断时要把 `pg_stat_wal_receiver.flushed_lsn` 和 `pg_last_wal_replay_lsn()` 分开。
前者是接收并 flush 到本地 WAL 的位置。
后者是 startup process 已成功 replay 的位置。
两者之间的差距是 apply lag。
`pg_last_xact_replay_timestamp()` 只反映最近 replay 的 commit / abort 时间戳。
如果 workload 没有事务提交记录，或者 replay 的 record 不是 commit / abort，它可能不动。
不要把它当成完整 apply progress。
## 21. 一个可复现的 runtime 现象
现象：
```text
standby 日志出现 "consistent recovery state reached at ..."
但客户端仍暂时无法连接执行只读查询。
```
源码解释：
```text
CheckRecoveryConsistency()
  -> reachedConsistency = true
  -> PMSIGNAL_RECOVERY_CONSISTENT

但如果 standbyState != STANDBY_SNAPSHOT_READY:
  不设置 SharedHotStandbyActive
  不发送 PMSIGNAL_BEGIN_HOT_STANDBY
```
最可能的原因是 standby snapshot 还没有完整建立。
例如 startup process 还没 replay 到 `XLOG_RUNNING_XACTS`，或者初始 running-xacts 有 subxid overflow，使 `standbyState` 停在 `STANDBY_SNAPSHOT_PENDING`。
外部可见症状是：
```text
有 consistent recovery state 日志
没有 ready to accept read-only connections 日志
pg_is_in_recovery() 仍为 true
walreceiver 可能正在 streaming
pg_last_wal_replay_lsn() 可能仍在推进
```
这时不要把问题诊断成 walreceiver 没连上。
walreceiver 可以正常 streaming。
缺的是 standby snapshot ready 边界。
源码断点可以放在：
```text
CheckRecoveryConsistency()
ProcArrayApplyRecoveryInfo()
standby_redo()
```
观察：
```text
reachedConsistency
standbyState
XLogRecoveryCtl->SharedHotStandbyActive
```
一旦 `standbyState` 进入 `STANDBY_SNAPSHOT_READY`，下一次 `CheckRecoveryConsistency()` 就会设置 hot standby active。
## 22. 常见误区
误区一：`standby.signal` 只设置 `StandbyModeRequested` 和 `ArchiveRecoveryRequested`，streaming 是否开始还要看 `WaitForWALToBecomeAvailable()` 是否进入 `XLOG_FROM_STREAM`。
误区二：`pg_is_in_recovery()` 只看 `SharedRecoveryState != RECOVERY_STATE_DONE`，不等于 hot standby readiness；同样为 true 时，节点可能已经可读，也可能还没 consistent。
误区三：`consistent recovery state reached` 只对应 `reachedConsistency`，还需要 `standbyState == STANDBY_SNAPSHOT_READY` 和 postmaster 处理 `PMSIGNAL_BEGIN_HOT_STANDBY`。
误区四：`pg_stat_wal_receiver` 为空不一定是故障；startup process 可以从 archive 或 `pg_wal` 读取 WAL，也可以在没有 `primary_conninfo` 时轮询本地 WAL。
误区五：`pg_last_wal_replay_lsn()` 不动不能直接归因于网络；缺 WAL、pause、recovery conflict、redo 慢、restore_command 慢、walreceiver flush 未到位都可能造成同一现象。
误区六：普通 backend 不能用 `standbyState` 判断 hot standby；它只在 startup process 有效，跨进程判断应使用 `HotStandbyActive()` / `SharedHotStandbyActive`。
## 23. 课堂实验
实验一：源码跟读状态转换。在 `/home/nail/postgres` 中给这些函数打断点：
```text
StartupXLOG
InitWalRecovery
PerformWalRecovery
ReadRecord
WaitForWALToBecomeAvailable
CheckRecoveryConsistency
ProcArrayApplyRecoveryInfo
RequestXLogStreaming
```
启动带 `standby.signal` 的 standby，记录每次断点处的：
```text
ArchiveRecoveryRequested
InArchiveRecovery
StandbyModeRequested
StandbyMode
currentSource
reachedConsistency
standbyState
XLogRecoveryCtl->SharedHotStandbyActive
```
目标不是记住值，而是画出请求态、当前恢复态、WAL source、consistency gate、hot standby gate 的顺序。
实验二：观察缺 WAL 的状态机。让 standby 有 `standby.signal`，但暂时关闭 upstream 或不给 `restore_command` 可用 WAL，观察日志：
```text
waiting for WAL to become available at ...
```
再查询：
```sql
SELECT * FROM pg_stat_wal_receiver;
SELECT pg_last_wal_replay_lsn();
```
如果 `pg_stat_wal_receiver` 为空，判断当前是在 archive / pg_wal 轮询，还是 walreceiver 尚未启动。
如果有行，比较 `flushed_lsn` 和 `pg_last_wal_replay_lsn()`。
实验三：区分 consistent 与 read-only ready。在 `CheckRecoveryConsistency()` 内临时加 DEBUG 日志，打印：
```text
reachedConsistency
standbyState
LocalHotStandbyActive
```
不要改变逻辑；重建并启动 standby，观察 `consistent recovery state reached` 和 `database system is ready to accept read-only connections` 之间是否存在间隔。如果存在，回到 `ProcArrayApplyRecoveryInfo()` 看 standby snapshot 是否 pending。
## 24. 讨论题
1. 为什么 `ArchiveRecoveryRequested = true` 时，`InArchiveRecovery` 可以暂时为 false？
2. 为什么 `standby.signal` 同时意味着 archive recovery requested，而 `recovery.signal` 不意味着 standby mode requested？
3. 为什么 crash recovery 中 `CheckRecoveryConsistency()` 不设置 `reachedConsistency`？
4. 为什么 `standbyState == STANDBY_SNAPSHOT_READY` 仍不能单独开放只读连接？
5. 为什么 `SharedHotStandbyActive` 必须是共享状态，而 `standbyState` 只能作为 startup process 本地事实？
6. promotion 为什么先 replay 已经可用的 archive / pg_wal，再结束 recovery？
7. `pg_last_wal_replay_lsn()` 不前进或 `pg_stat_wal_receiver` 为空时，你会按什么顺序排除缺 WAL、pause、conflict、archive restore 和 streaming 问题？
## 25. 本节小结
本节的核心链路是：`StartupProcessMain()` -> `StartupXLOG()` -> `InitWalRecovery()` -> `PerformWalRecovery()` -> `ReadRecord()` -> `WaitForWALToBecomeAvailable()` -> `ApplyWalRecord()` -> `CheckRecoveryConsistency()` -> `PMSIGNAL_BEGIN_HOT_STANDBY`。
核心状态分三类。
请求态是 `ArchiveRecoveryRequested`、`StandbyModeRequested`；startup process 当前事实是 `InArchiveRecovery`、`StandbyMode`、`currentSource`、`standbyState`、`reachedConsistency`；跨进程共享事实是 `SharedRecoveryState`、`SharedHotStandbyActive`、`SharedPromoteIsTriggered`、`lastReplayedEndRecPtr`、`WalRcv->flushedUpto`。
Hot Standby 可查询的关键不变量是：physical consistency reached AND standby snapshot ready AND postmaster has opened read-only connections。
缺 WAL 时，standby 不会把缺失当作立即失败。
它会在 archive、pg_wal、stream 和 sleep 之间循环。
PITR / non-standby archive recovery 则必须达到目标，否则失败。
pause 只能在 hot standby 已经可连接后真正停住。
promotion 会解除 pause，并尽量 replay 已经可用的 WAL 后结束恢复。
诊断时不要把一个指标当成完整因果。
`pg_is_in_recovery()` 只看 recovery 是否结束。
`pg_last_wal_replay_lsn()` 只看 replay 成功位置。
`pg_stat_wal_receiver` 只看 streaming source。
日志中的 `consistent recovery state reached` 和 `ready to accept read-only connections` 分别对应不同门槛。
可迁移的系统规律是：
```text
在恢复型系统中，“已经有输入流”、“已经物理一致”、“已经有可解释的并发可见性状态”、
和“入口已对外开放”通常是四个不同状态；
把它们压成一个布尔值，会导致诊断和设计都失去边界。
```
