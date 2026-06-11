# PostgreSQL Hot Standby snapshot visibility 与 KnownAssignedXids
## 课程定位
前置知识：已经理解 PostgreSQL MVCC snapshot 的 `xmin` / `xmax` / `xip` 基本语义，知道 WAL replay 由 startup process 顺序执行，也知道 standby 上用户连接只能执行只读事务。
本节唯一主问题：
```text
standby 上没有本地写事务时，只读查询如何获得可用 snapshot，
running-xacts WAL record 和 known-assigned XID 如何让 recovery 中的 MVCC 可见性成立？
```
核心矛盾：只读 standby backend 自己不会分配写事务 XID，也看不到 primary backend 的 `PGPROC` 数组；但 heap tuple 的 `xmin` / `xmax` 来自主库事务，MVCC 可见性仍必须按主库在某个 WAL replay 点的事务运行状态解释。
PostgreSQL 的选择不是让 standby 查询去询问 primary，也不是把 primary 的 ProcArray 原样复制过来。
它把 primary 的事务运行集合编码进 WAL，并在 standby startup process replay WAL 时维护一份 shared-memory `KnownAssignedXids`。
只读 backend 调用 `GetSnapshotData()` 时，在 recovery 分支把 `KnownAssignedXids` 拷入 snapshot。
随后 `HeapTupleSatisfiesMVCC()` 仍使用普通 snapshot 判定，只是 `XidInMVCCSnapshot()` 根据 `snapshot->takenDuringRecovery` 改变对 `xip` / `subxip` 的解释。
学完后应能判断：
```text
consistent recovery state 与 STANDBY_SNAPSHOT_READY 为什么不是同一件事；
running-xacts WAL record 为什么必须存在；
RecordKnownAssignedTransactionIds() 为什么要补齐没直接出现在 WAL 中的 XID；
GetSnapshotData() 在 standby 上为什么不扫描本地写事务；
replay 与只读查询并发时，哪些由 snapshot 保证，哪些会退化为 recovery conflict。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面几节已经从 replication 连接、WAL sender / receiver、slot 和 WAL 保留边界进入 recovery 语境。
本节开始回答 standby 可读的第一个 correctness 问题：
```text
WAL 已经能到达 standby
  -> startup process 能顺序 replay WAL
  -> postmaster 何时允许只读连接
  -> 只读连接如何拿到能解释 primary XID 的 MVCC snapshot
```
这里不要先跳到冲突取消。
冲突取消回答的是：
```text
replay 需要推进，而旧只读 snapshot 或 buffer pin 阻挡了 replay，怎么办？
```
本节主线回答的是更基础的问题：
```text
在没有本地写事务参与的 standby 上，snapshot 本身从哪里来？
```
如果没有这层状态，冲突取消也无从谈起。
因为 standby 查询甚至无法判断一个 tuple 的 `xmin` 是否属于“snapshot 生成时仍在运行的 primary 事务”。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
primary 在 checkpoint 附近写出 XLOG_RUNNING_XACTS；
standby startup process 从 checkpoint 开始 replay，先进入 STANDBY_INITIALIZED，
边 replay 边用 RecordKnownAssignedTransactionIds() 记录观察到或推断出的主库 XID，
遇到 running-xacts record 后用 ProcArrayApplyRecoveryInfo() 初始化或修正 KnownAssignedXids，
进入 STANDBY_SNAPSHOT_READY 后，postmaster 才能允许只读连接；
只读 backend 的 GetSnapshotData() 从 KnownAssignedXids 构造 recovery snapshot。
```
这条链路把三个时间点连在一起：
```text
primary 取 running-xacts snapshot 的时间点；
standby replay 到某个 WAL record 的时间点；
只读 backend 调用 GetSnapshotData() 的时间点。
```
MVCC correctness 的关键是不让这三个时间点互相冒充。
primary 上事务是否运行，是通过 WAL record 传下来的历史事实。
standby 上当前有哪些只读 backend，是本地 ProcArray 里的运行事实。
只读查询需要的是前者，而不是后者。
这也是 `KnownAssignedXids` 存在的原因。
它不是普通 backend 的事务数组。
它是 startup process 根据 WAL 维护出来的“在当前 replay 点，primary 上必须被当作仍在运行的 XID 集合”。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/xlogutils.h` | `HotStandbyState`、`standbyState`、`InHotStandby` 的状态边界。 |
| 2 | `src/backend/access/transam/xlog.c` | recovery 初始化、shutdown checkpoint 的 empty running-xacts、checkpoint 前 `LogStandbySnapshot()`。 |
| 3 | `src/backend/access/transam/xlogrecovery.c` | WAL replay 主循环、`RecordKnownAssignedTransactionIds()` 调用点、`CheckRecoveryConsistency()`。 |
| 4 | `src/backend/storage/ipc/standby.c` | `InitRecoveryTransactionEnvironment()`、`standby_redo()`、`LogStandbySnapshot()`、recovery conflict 边界。 |
| 5 | `src/include/storage/standby.h` | `RunningTransactionsData` 与 `xl_running_xacts` 相关字段语义。 |
| 6 | `src/backend/storage/ipc/procarray.c` | `KnownAssignedXids`、`ProcArrayApplyRecoveryInfo()`、`RecordKnownAssignedTransactionIds()`、`GetSnapshotData()` recovery 分支。 |
| 7 | `src/include/storage/procarray.h` | recovery 事务跟踪接口声明。 |
| 8 | `src/backend/access/transam/xact.c` | commit / abort / xid assignment redo 如何移除或折叠 known-assigned XID。 |
| 9 | `src/backend/utils/time/snapmgr.c` | `XidInMVCCSnapshot()` 对 recovery snapshot 的解释。 |
| 10 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesMVCC()` 如何使用 snapshot 判断 tuple 可见性。 |
推荐阅读时不要从 `standby.c` 的冲突处理一路读到底。
本节的主链路是：
```text
primary 写 running-xacts WAL
  -> standby 初始化 recovery transaction environment
  -> replay 每条带 xl_xid 的 WAL record 时记录 known-assigned XID
  -> replay XLOG_RUNNING_XACTS 时建立可用 snapshot 状态
  -> postmaster 允许 read-only connection
  -> backend GetSnapshotData()
  -> heapam visibility 使用 recovery snapshot
```
这个顺序比按文件名排序更接近 runtime。
## 4. 先看 runtime 现象
一个可复现现象：
```text
primary:
  BEGIN;
  INSERT INTO hs_demo VALUES (1);
  -- 不提交，保持事务打开
standby:
  SELECT * FROM hs_demo;
```
如果 standby 已经 replay 到插入 tuple 的 WAL，但还没有 replay 到该事务的 commit WAL，standby 查询不能看到这行。
它不能说“standby 上没有写事务，所以没有 running XID，tuple 应该可见”。
它也不能直接访问 primary 的 backend 进程。
它能做的判断是：
```text
tuple xmin = primary transaction xid X
当前 recovery snapshot 认为 X 仍在运行
所以 HeapTupleSatisfiesMVCC() 返回不可见
```
当 primary 提交并且 standby replay 到 commit record 后，新 snapshot 才能看到这行。
旧 snapshot 仍应保持旧语义。
这正是 MVCC 的要求：
```text
snapshot 看到的是生成时刻的 running set；
不是每次读 tuple 时重新查询最新提交状态。
```
因此本节的 runtime truth 是：
```text
standby 只读查询的可见性，取决于 startup process 已经 replay 到的 WAL 历史，
以及 GetSnapshotData() 从 KnownAssignedXids 复制出的 running set。
```
## 5. 两个不要混淆的 readiness
Hot standby 对外可读需要两个 readiness。
第一类是 storage 一致性：
```text
recovery 已经达到 consistent recovery state。
```
`xlogrecovery.c` 的 `CheckRecoveryConsistency()` 使用 `minRecoveryPoint`、`backupEndPoint`、`lastReplayedEndRecPtr` 等状态判断数据文件是否已经到达安全一致点。
满足后会设置 `reachedConsistency = true`，并发送 `PMSIGNAL_RECOVERY_CONSISTENT`。
日志中常见：
```text
consistent recovery state reached at ...
```
第二类是 MVCC snapshot 可用性：
```text
standbyState == STANDBY_SNAPSHOT_READY。
```
这表示 standby 已经拥有一份足以解释 primary running XID 的 recovery snapshot 起点。
`CheckRecoveryConsistency()` 只有在两者都满足时才会打开 hot standby：
```text
standbyState == STANDBY_SNAPSHOT_READY
  && reachedConsistency
  && !LocalHotStandbyActive
  -> SharedHotStandbyActive = true
  -> SendPostmasterSignal(PMSIGNAL_BEGIN_HOT_STANDBY)
```
postmaster 收到 `PMSIGNAL_BEGIN_HOT_STANDBY` 后才会进入 `PM_HOT_STANDBY`，设置 `connsAllowed = true`，并记录：
```text
database system is ready to accept read-only connections
```
所以：
```text
consistent state 说明数据页可作为一致恢复状态使用；
snapshot ready 说明 MVCC running set 可用于只读查询。
```
这两个条件任何一个缺失，都不能安全接受普通只读查询。
## 6. `standbyState` 是 startup process 的本地状态
`src/include/access/xlogutils.h` 定义：
```text
STANDBY_DISABLED
STANDBY_INITIALIZED
STANDBY_SNAPSHOT_PENDING
STANDBY_SNAPSHOT_READY
```
源码注释强调一个细节：
```text
standbyState 只在 startup process 中可靠；
其他连接 shared memory 的进程不能只看这个变量判断 Hot Standby 是否 active。
```
普通 backend 应该通过 `HotStandbyActive()` 或 `RecoveryInProgress()` 这样的共享状态接口理解运行环境。
本节仍然要读 `standbyState`，因为它控制 startup process 如何维护 recovery-time transaction tracking。
状态含义可以压缩成：
| 状态 | 含义 |
| --- | --- |
| `STANDBY_DISABLED` | 没有启用 recovery-time transaction tracking。 |
| `STANDBY_INITIALIZED` | 已初始化锁、invalidation、known-assigned 基础环境，但还不能给查询 snapshot。 |
| `STANDBY_SNAPSHOT_PENDING` | 已有部分 running-xacts 信息，但 subxid overflow 让 snapshot 尚不完整。 |
| `STANDBY_SNAPSHOT_READY` | 已拥有完整或可保守解释的 primary running set，可以生成 recovery snapshot。 |
`InHotStandby` 宏是：
```text
standbyState >= STANDBY_SNAPSHOT_PENDING
```
但这不是“用户可连接”的完整条件。
用户可连接还要等 recovery consistency 与 postmaster 状态推进。
## 7. 为什么不能直接扫描 standby ProcArray
普通 primary 上 `GetSnapshotData()` 的核心动作是扫描 ProcArray：
```text
看每个 backend 的 xid；
跳过没有 xid 的 backend；
收集其他 running top-level xid 到 xip；
尽量收集 subxid 到 subxip；
计算 xmin / xmax。
```
standby 上只读 backend 不会产生修改数据的事务 XID。
即使它们出现在本地 ProcArray，也不是 heap tuple 的 `xmin` / `xmax` 来源。
standby 正在 replay 的 heap tuple 来自主库 WAL。
这些 tuple 的事务身份是 primary 上的 XID。
因此 standby snapshot 必须回答：
```text
在当前 replay 点，哪些 primary XID 要被当作 still running？
```
这个集合不在本地只读 backend 的 `PGPROC` 中。
它由 startup process 从 WAL 推导并维护。
于是 `GetSnapshotData()` 在 recovery 分支走不同路径：
```text
if (!snapshot->takenDuringRecovery):
  scan local ProcArray PGPROC xids
else:
  get XIDs from KnownAssignedXids
```
这就是本节所有机制的中心。
## 8. primary 为什么要写 running-xacts WAL record
standby 从 checkpoint 开始恢复。
如果 checkpoint 是 shutdown checkpoint，主库在该点没有普通 running transaction。
这种情况 standby 可以伪造一条 empty running-xacts 信息，只需要考虑 prepared transaction。
但更常见的是 online checkpoint。
online checkpoint 时 primary 上可能有事务已经运行很久。
standby 从 checkpoint 之后开始 replay 时，可能会先看到这些事务的后续 WAL，却看不到它们开始时的全部状态。
如果没有额外信息，standby 无法知道：
```text
checkpoint 时哪些 XID 已经在运行；
这些 XID 何时应该被放进初始 recovery snapshot；
哪些 xid gap 是已分配但还没在 WAL 中直接出现。
```
`standby.c` 的注释把问题说得很直接：
```text
initial snapshot must contain all running xids and all current AccessExclusiveLocks
```
这些信息不能通过一个轻量锁同时精确取得。
因此 primary 在 checkpoint 附近分块记录：
```text
当前 AccessExclusiveLock
当前 RunningTransactionsData
```
`xlog.c` 的 checkpoint 路径在非 shutdown checkpoint 且 `XLogStandbyInfoActive()` 时调用：
```text
LogStandbySnapshot()
```
而 `XLogStandbyInfoActive()` 是：
```text
wal_level >= WAL_LEVEL_REPLICA
```
这也是为什么 hot standby 所需信息依赖 `wal_level = replica` 或更高。
## 9. `RunningTransactionsData` 不是普通 Snapshot
`src/include/storage/standby.h` 定义 `RunningTransactionsData`。
本节关注字段：
| 字段 | 语义 |
| --- | --- |
| `xcnt` | `xids[]` 中 top-level XID 数量。 |
| `subxcnt` | `xids[]` 中 subtransaction XID 数量。 |
| `subxid_status` | subxid 是否完整、缺失，或已经可从 subtrans 解释。 |
| `nextXid` | primary `nextXid`，用于推进 standby 的 XID 分配边界。 |
| `oldestRunningXid` | primary 上最老 still-running XID，不是 vacuum 的 oldestXmin。 |
| `latestCompletedXid` | 用于建立 snapshot 的 `xmax` 基准。 |
| `xids` | running top/sub XID 数组。 |
头文件注释明确说：
```text
This has nothing at all to do with visibility on this server.
```
意思不是“它和 visibility 无关”。
意思是它不是当前服务器本地查询的普通 snapshot。
它是 primary running transaction state 的 WAL 表达。
standby replay 后，会把它转成 recovery-time shared state。
这个转换发生在：
```text
standby_redo()
  -> XLOG_RUNNING_XACTS
  -> ProcArrayApplyRecoveryInfo(&running)
```
## 10. `LogStandbySnapshot()` 的写入顺序
`src/backend/storage/ipc/standby.c` 的 `LogStandbySnapshot()` 做两件事。
第一，记录 AccessExclusive locks：
```text
GetRunningTransactionLocks(&nlocks)
if nlocks > 0:
  LogAccessExclusiveLocks(nlocks, locks)
```
第二，记录 running transaction：
```text
running = GetRunningTransactionData()
recptr = LogCurrentRunningXacts(running)
```
`LogCurrentRunningXacts()` 组装 `xl_running_xacts`，并插入：
```text
XLogInsert(RM_STANDBY_ID, XLOG_RUNNING_XACTS)
```
这条 record 标记为 `XLOG_MARK_UNIMPORTANT`，避免为了这个辅助信息触发过多 checkpoint / archiving 压力。
但它对 hot standby correctness 很重要。
`LogStandbySnapshot()` 注释里还有一个关键 race：
```text
primary 取得 running-xacts 信息和写 WAL record 之间存在窗口；
有些事务可能在窗口内提交；
standby apply running-xacts 时必须重新检查 CLOG，忽略已经提交或回滚的 XID。
```
这就是 `ProcArrayApplyRecoveryInfo()` 里会调用 `TransactionIdDidCommit()` / `TransactionIdDidAbort()` 过滤的原因。
它不是多余的防御。
它是 running-xacts record 生成方式的 correctness 补偿。
## 11. 主流程源码 walkthrough：hot standby 初始化入口
recovery 初始化路径在 `xlog.c`。
当满足：
```text
ArchiveRecoveryRequested && EnableHotStandby
```
系统会初始化 Hot Standby transaction tracking：
```text
InitRecoveryTransactionEnvironment()
ProcArrayInitRecovery(...)
StartupSUBTRANS(oldestActiveXID)
```
`InitRecoveryTransactionEnvironment()` 在 `standby.c`。
它做的不是创建一个普通 SQL transaction。
它给 startup process 建立 recovery-time 能用的锁和 invalidation 环境：
```text
创建 RecoveryLockHash / RecoveryLockXidHash；
SharedInvalBackendInit(true)；
为 startup process 放入一个 permanent virtual transaction lock entry；
standbyState = STANDBY_INITIALIZED。
```
这个 permanent vxid 是为了让 recovery replay 的 AccessExclusiveLock 能与 standby 查询的 AccessShareLock 形成冲突关系。
但本节只把它作为边界。
我们的主线仍是 XID visibility。
`ProcArrayInitRecovery()` 设置 `latestObservedXid`。
它告诉 recovery 后续：
```text
从这个 XID 边界开始，遇到 WAL record 中的 XID 时要 gaplessly 初始化 subtrans，并维护 known-assigned 状态。
```
## 12. shutdown checkpoint 的 fast path
如果从 shutdown checkpoint 开始恢复，primary 在 checkpoint 时没有普通 running transaction。
`xlog.c` 会构造一份 synthetic `RunningTransactionsData`：
```text
running.xcnt = prepared transaction count
running.subxcnt = 0
running.subxid_status = SUBXIDS_IN_SUBTRANS
running.nextXid = checkpoint.nextXid
running.oldestRunningXid = oldestActiveXID
running.latestCompletedXid = checkpoint.nextXid - 1
running.xids = prepared transaction xids
ProcArrayApplyRecoveryInfo(&running)
```
这可以直接把 standby 推向 `STANDBY_SNAPSHOT_READY`。
它不是绕过机制。
它是用“shutdown checkpoint 上没有普通 running transaction”这个更强事实，替代后续等待 `XLOG_RUNNING_XACTS`。
replay 过程中再次看到 shutdown checkpoint 时，也会用同样方式构造 empty running-xacts 状态。
这条 fast path 对诊断很有用：
```text
从干净停机后的 base backup 启动 standby，snapshot ready 通常不需要等很久；
从 online checkpoint 启动，则必须等待相关 running-xacts WAL 信息。
```
## 13. replay 每条 WAL record 时先记录 XID
`xlogrecovery.c` 的 WAL replay 主路径在调用 rmgr redo 前做一件事：
```text
if (standbyState >= STANDBY_INITIALIZED &&
    TransactionIdIsValid(record->xl_xid))
  RecordKnownAssignedTransactionIds(record->xl_xid);
```
这一步发生在具体 rmgr redo 之前。
原因是 redo 期间可能需要依赖 recovery-time transaction tracking。
`RecordKnownAssignedTransactionIds()` 在 `procarray.c`。
它的核心语义是：
```text
看到 WAL record 上的 xid X；
如果 X 大于 latestObservedXid；
则认为 (latestObservedXid, X] 之间所有 XID 都已经被 primary 分配；
必要时 ExtendSUBTRANS()；
在 snapshot machinery 已经 ready/pending 后，把这段 XID 加入 KnownAssignedXids；
推进 latestObservedXid；
推进 standby 的 nextXid 边界。
```
为什么要补齐 gap？
因为 XID 在 primary 上顺序分配。
不是每个已分配 XID 都一定马上出现在 WAL record 的 `xl_xid` 字段中。
如果 standby 只记录自己直接看到的 XID，就会把“已分配但尚未出现”的事务误认为不运行。
对 MVCC 来说，这会让未提交 tuple 过早可见。
因此 known-assigned 的含义不是“我在 WAL 中见过这些 XID”。
更准确是：
```text
根据 WAL 中观察到的最大 XID 和 XID 顺序分配规则，
这些 XID 必须被视为已经分配，并且在完成记录到达前不能被当作完成。
```
## 14. 为什么 `STANDBY_INITIALIZED` 时只推进观察边界
`RecordKnownAssignedTransactionIds()` 有一个容易误读的分支：
```text
if (standbyState <= STANDBY_INITIALIZED)
{
  latestObservedXid = xid;
  return;
}
```
这不是说 `STANDBY_INITIALIZED` 阶段 XID 不重要。
相反，它们很重要。
但此时 initial recovery snapshot 尚未建立。
系统还不能让查询拿 snapshot。
在这段时间，startup process 需要做两件基础工作：
```text
gaplessly ExtendSUBTRANS()
推进 latestObservedXid
```
真正把 XID 放入 `KnownAssignedXids`，要等 running-xacts 信息应用后。
`LogStandbySnapshot()` 注释解释了这个设计：
```text
从 checkpoint 起点开始累计变化；
稍后看到 running-xacts record 时，把 primary 上的 running set 和已经 replay 的变化重新组装。
```
这是一种分阶段重建。
它避免在信息不完整时给只读查询暴露 snapshot。
## 15. `standby_redo()` 如何消费 running-xacts record
`standby.c` 的 `standby_redo()` 处理 `RM_STANDBY_ID`。
对于 `XLOG_RUNNING_XACTS`，它把 WAL payload 转成内存中的 `RunningTransactionsData`：
```text
running.xcnt = xlrec->xcnt
running.subxcnt = xlrec->subxcnt
running.subxid_status =
  xlrec->subxid_overflow ? SUBXIDS_MISSING : SUBXIDS_IN_ARRAY
running.nextXid = xlrec->nextXid
running.latestCompletedXid = xlrec->latestCompletedXid
running.oldestRunningXid = xlrec->oldestRunningXid
running.xids = xlrec->xids
ProcArrayApplyRecoveryInfo(&running)
```
注意这里没有把 WAL 里的 bytes 直接当作 snapshot 给用户 backend。
startup process 仍要把它合并进 ProcArray recovery state。
这个合并必须处理：
```text
stale KnownAssignedXids；
nextXid 推进；
AccessExclusiveLock stale release；
initial snapshot 是否 overflow；
running-xacts record 中已经在 CLOG 里完成的 XID；
subtrans 初始化；
latestCompletedXid 维护。
```
所以 `XLOG_RUNNING_XACTS` 是输入，不是最终 snapshot。
最终 query snapshot 仍由 `GetSnapshotData()` 在用户 backend 中生成。
## 16. `ProcArrayApplyRecoveryInfo()` 第一阶段：清旧边界
`ProcArrayApplyRecoveryInfo()` 一进来先做：
```text
ExpireOldKnownAssignedTransactionIds(running->oldestRunningXid)
```
`ExpireOldKnownAssignedTransactionIds()` 的含义是：
```text
running->oldestRunningXid 之前的普通事务不可能仍在运行；
把 KnownAssignedXids 中更老的条目标记为完成；
维护 latestCompletedXid；
递增 xactCompletionCount；
必要时清理 lastOverflowedXid。
```
这一步不是只服务初始化。
后续每次 replay `XLOG_RUNNING_XACTS`，它都能清理可能因为 FATAL、缺失 abort record 或历史窗口残留的 known-assigned XID。
源码注释说：
```text
primary backend somehow disappear before it can write an abort record
```
这种 XID 留在 `KnownAssignedXids` 里并不会让错误数据可见。
它们被当作 running，而 running transaction 的修改本来就不可见。
真正的问题是数组可能无限膨胀。
定期 running-xacts record 提供了 pruning 边界。
接着函数推进 standby 的 transaction machinery：
```text
advanceNextXid = running->nextXid - 1
AdvanceNextFullTransactionIdPastXid(advanceNextXid)
StandbyReleaseOldLocks(running->oldestRunningXid)
```
这样 XID 边界和 lock 边界一起推进到当前 WAL 历史。
## 17. `ProcArrayApplyRecoveryInfo()` 第二阶段：处理 pending
如果 `standbyState == STANDBY_SNAPSHOT_READY`，函数完成 pruning 和 nextXid 推进后即可返回。
因为 query snapshot 已经可用，后续 running-xacts record 主要用于修剪和推进边界。
如果 `standbyState == STANDBY_SNAPSHOT_PENDING`，说明之前 initial running-xacts snapshot 因 subxid overflow 不完整。
此时有两种出路：
```text
新的 running-xacts record 不再 overflow；
或者 running->oldestRunningXid 已经超过 standbySnapshotPendingXmin。
```
第一种情况，系统丢弃已收集的 known-assigned 状态，回到 `STANDBY_INITIALIZED`，用新的完整 snapshot 重新初始化。
第二种情况，虽然没有拿到完整 subxid 列表，但那些缺失 subxid 所属的旧事务已经不再可能影响新 snapshot。
于是可以进入 `STANDBY_SNAPSHOT_READY`。
如果两者都不满足，就继续等待。
日志中可能看到类似：
```text
recovery snapshot waiting for non-overflowed snapshot or until oldest active xid ...
```
这条日志解释了 standby 为什么已经在 replay WAL，却仍暂时不能开放只读查询。
## 18. `ProcArrayApplyRecoveryInfo()` 第三阶段：建立 KnownAssignedXids
当状态是 `STANDBY_INITIALIZED`，函数会拿 `ProcArrayLock` exclusive。
然后为 running-xacts record 中的 XID 做过滤：
```text
for xid in running->xids:
  if TransactionIdDidCommit(xid) || TransactionIdDidAbort(xid):
    continue
  xids[nxids++] = xid
```
这个过滤对应前面提到的 primary 取 snapshot 与写 WAL 之间的 race。
接着排序：
```text
qsort(xids, nxids, sizeof(TransactionId), xidLogicalComparator)
```
然后逐个加入：
```text
KnownAssignedXidsAdd(xids[i], xids[i], true)
```
这里 `exclusive_lock = true`，因为已经持有 `ProcArrayLock` exclusive。
加入完成后，函数补齐 `SUBTRANS` 到 `running->nextXid - 1`。
最后根据 subxid 状态决定：
```text
SUBXIDS_MISSING:
  standbyState = STANDBY_SNAPSHOT_PENDING
  standbySnapshotPendingXmin = latestObservedXid
  procArray->lastOverflowedXid = latestObservedXid
SUBXIDS_IN_SUBTRANS:
  standbyState = STANDBY_SNAPSHOT_READY
  procArray->lastOverflowedXid = latestObservedXid
SUBXIDS_IN_ARRAY:
  standbyState = STANDBY_SNAPSHOT_READY
  procArray->lastOverflowedXid = InvalidTransactionId
```
再调用：
```text
MaintainLatestCompletedXidRecovery(running->latestCompletedXid)
```
至此，startup process 拥有了当前 replay 点可用的 primary running set。
## 19. `KnownAssignedXids` 的结构语义
`KnownAssignedXids` 在 `procarray.c` 中是 shared memory 数组。
初始化时只有 `EnableHotStandby` 才请求：
```text
KnownAssignedXids
KnownAssignedXidsValid
```
容量为：
```text
TOTAL_MAX_CACHED_SUBXIDS =
  (PGPROC_MAX_CACHED_SUBXIDS + 1) * PROCARRAY_MAXPROCS
```
`ProcArrayStruct` 里配套字段包括：
```text
maxKnownAssignedXids
numKnownAssignedXids
tailKnownAssignedXids
headKnownAssignedXids
lastOverflowedXid
```
数组设计是：
```text
KnownAssignedXids[] 按 TransactionIdPrecedes 顺序保存 XID；
KnownAssignedXidsValid[] 标记条目是否仍有效；
tail <= i < head 是当前使用区间；
删除时只把 valid 标记置 false；
必要时再 compress。
```
这样做是为了让几类操作有可控成本：
| 操作 | 成本与锁 |
| --- | --- |
| startup process append 新 XID | 通常 O(1)，无锁或已有 exclusive lock。 |
| 删除完成事务 | O(logS) search，加 exclusive `ProcArrayLock`。 |
| snapshot 复制 | O(S)，shared `ProcArrayLock`。 |
| 查某个 XID 是否 running | O(logS)，shared `ProcArrayLock`。 |
| compress gaps | O(S)，exclusive `ProcArrayLock`。 |
这里的 `S` 是 `tail..head` 的跨度，不一定等于有效 XID 数 `N`。
所以源码会用启发式压缩，让 `S` 不长期远大于 `N`。
## 20. 为什么 add 可以不拿 exclusive lock
`KnownAssignedXidsAdd()` 有一个非直觉设计：
```text
startup process 追加 XID 时通常不持有 ProcArrayLock。
```
这是安全的，因为只有 startup process 会修改 head/tail 和数组内容。
并发读者是只读 backend。
它们在 `GetSnapshotData()` 或 `TransactionIdIsInProgress()` 中持有 shared `ProcArrayLock`，并使用 memory barrier 与 writer 配对。
写入顺序是：
```text
写 KnownAssignedXids[head..]
写 KnownAssignedXidsValid[head..]
pg_write_barrier()
推进 headKnownAssignedXids
```
读取端：
```text
读取 headKnownAssignedXids
pg_read_barrier()
扫描 tail..head
```
这样读者不会先看到新的 head，却看不到对应数组内容。
删除和 compress 必须拿 exclusive lock。
因为删除会改变 valid 标记、tail、numKnownAssignedXids，必须与 snapshot 复制互斥。
这层锁和 barrier 的组合是本节重要边界：
```text
MVCC 语义来自 KnownAssignedXids；
内存一致性来自 ProcArrayLock + barriers；
二者不能混为一谈。
```
## 21. commit / abort replay 如何移除 XID
`xact.c` 的 commit redo 在 standby mode 下大致是：
```text
RecordKnownAssignedTransactionIds(max_xid)
TransactionIdAsyncCommitTree(...)
ExpireTreeKnownAssignedTransactionIds(xid, nsubxacts, subxacts, max_xid)
ProcessCommittedInvalidationMessages(...)
StandbyReleaseLockTree(...)
```
abort redo 类似：
```text
RecordKnownAssignedTransactionIds(max_xid)
TransactionIdAbortTree(...)
ExpireTreeKnownAssignedTransactionIds(xid, nsubxacts, subxacts, max_xid)
StandbyReleaseLockTree(...)
```
这里顺序很关键。
源码注释强调：
```text
We must mark clog before we update the ProcArray.
```
先把 transaction status 写入 pg_xact / CLOG，再从 `KnownAssignedXids` 移除。
这样只读 backend 不会遇到“snapshot 不再认为 XID running，但 CLOG 还没有完成状态”的中间语义。
`ExpireTreeKnownAssignedTransactionIds()` 内部持有 `ProcArrayLock` exclusive，移除 top XID 和 subxids，然后：
```text
MaintainLatestCompletedXidRecovery(max_xid)
TransamVariables->xactCompletionCount++
```
`xactCompletionCount` 对 `GetSnapshotDataReuse()` 很重要。
只要有完成事务改变 running set，复用旧 snapshot 的快速路径就会失效。
## 22. xid assignment record 如何处理 subxid
primary 每个 top-level transaction 内分配一定数量 subxid 后，会写 `XLOG_XACT_ASSIGNMENT`。
`xact.c` 的注释说明原因：
```text
限制 standby 为 in-progress XID 保留的 shared memory；
避免 KnownAssignedXids 和 snapshot 无界增长。
```
redo 时：
```text
xact_redo()
  -> XLOG_XACT_ASSIGNMENT
  -> ProcArrayApplyXidAssignment(xtop, nsubxacts, xsub)
```
`ProcArrayApplyXidAssignment()` 先调用：
```text
RecordKnownAssignedTransactionIds(max_xid)
```
然后更新：
```text
SubTransSetParent(subxid, topxid)
```
如果状态已经超过 `STANDBY_INITIALIZED`，它会把这些 subxid 从 `KnownAssignedXids` 移除，并推进：
```text
procArray->lastOverflowedXid = max(max, lastOverflowedXid)
```
这样做的后果是：
```text
snapshot 里不一定保存所有 subxid；
需要时通过 pg_subtrans 找 topmost transaction；
snapshot->suboverflowed 会告诉 XidInMVCCSnapshot() 走慢路径。
```
这也是为什么 standby snapshot 会把所有 known-assigned XID 放到 `subxip`。
在 recovery 中，系统经常不知道一个 XID 是 top-level 还是 subxact。
但对 `XidInMVCCSnapshot()` 来说，只要它能判断“这个 XID 或它的 top-level parent 是否在 snapshot running set 中”，可见性就成立。
## 23. 只读查询如何调用 `GetSnapshotData()`
用户执行普通 `SELECT` 时，最终会通过 snapshot manager 取得 MVCC snapshot。
本节只追 `procarray.c` 的核心函数：
```text
GetSnapshotData(Snapshot snapshot)
```
它先为 `snapshot->xip` 和 `snapshot->subxip` 分配足够空间。
注释明确说：
```text
Snapshot is same size whether or not we are in recovery
```
然后拿 shared `ProcArrayLock`。
在锁内，它计算：
```text
latest_completed = TransamVariables->latestCompletedXid
xmax = latest_completed + 1
xmin = xmax
snapshot->takenDuringRecovery = RecoveryInProgress()
```
非 recovery 分支扫描本地 PGPROC。
recovery 分支则进入本节重点：
```text
subcount = KnownAssignedXidsGetAndSetXmin(snapshot->subxip, &xmin, xmax)
if (TransactionIdPrecedesOrEquals(xmin, procArray->lastOverflowedXid))
  suboverflowed = true
```
它没有把 XID 放进 `snapshot->xip`。
它把 known-assigned 全部放进 `snapshot->subxip`。
随后设置：
```text
snapshot->xmin = xmin
snapshot->xmax = xmax
snapshot->xcnt = count       -- recovery 下通常为 0
snapshot->subxcnt = subcount
snapshot->suboverflowed = suboverflowed
snapshot->takenDuringRecovery = true
```
并在需要时把 `MyProc->xmin` 设置为当前 snapshot 的 `xmin`。
这让 vacuum / conflict 判断能知道本地只读 backend 正持有多老的 snapshot。
## 24. `KnownAssignedXidsGetAndSetXmin()` 的边界
`KnownAssignedXidsGetAndSetXmin()` 在 shared lock 下扫描 `tail..head`。
它做三件事：
```text
跳过 invalid gap；
用第一个有效 XID 降低 *xmin；
把 < xmax 的 knownXid 拷到输出数组；
遇到 >= xmax 的 XID 就停止。
```
为什么可以遇到 `>= xmax` 就停止？
因为数组按 XID 顺序排序。
而 `xmax = latestCompletedXid + 1` 是 snapshot 的上界。
MVCC 规则中：
```text
任何 xid >= xmax 都被认为 still running；
不必出现在 snapshot array 中。
```
这也是 snapshot 结构的经典三段式：
```text
xid < xmin:
  finished
xmin <= xid < xmax:
  查 xip/subxip 判断是否 running
xid >= xmax:
  running
```
在 standby 上，这个三段式仍成立。
不同的是中间集合来自 `KnownAssignedXids`，不是本地写 backend。
## 25. `XidInMVCCSnapshot()` 如何解释 recovery snapshot
`src/backend/utils/time/snapmgr.c` 的 `XidInMVCCSnapshot()` 先做通用快速判断：
```text
if xid < snapshot->xmin:
  return false
if xid >= snapshot->xmax:
  return true
```
然后看：
```text
snapshot->takenDuringRecovery
```
普通 snapshot：
```text
subxip 保存 subxid；
xip 保存 top-level xid；
overflow 时先用 pg_subtrans 找 top-level，再查 xip。
```
recovery snapshot：
```text
所有 known-assigned XID 都放在 subxip；
xip 为空；
如果 suboverflowed，先用 SubTransGetTopmostTransaction()；
然后只查 subxip。
```
这和 `GetSnapshotData()` recovery 分支互相配套。
不要把 `subxip` 名字理解成“这里只能放 subtransaction”。
在 recovery snapshot 中，`subxip` 是一个更大的 running-XID array。
字段名字来自普通 snapshot 结构，但语义由：
```text
snapshot->takenDuringRecovery
snapshot->suboverflowed
GetSnapshotData() 写入方式
XidInMVCCSnapshot() 读取方式
```
共同决定。
raw field 不是语义。
## 26. heap 可见性如何回到普通 MVCC
`HeapTupleSatisfiesMVCC()` 不需要知道 running-xacts WAL record。
它只面对 tuple header 和 snapshot。
插入者 `xmin` 的核心判断是：
```text
if xmin is current transaction:
  用 command id 判断
else if XidInMVCCSnapshot(xmin, snapshot):
  return false
else if TransactionIdDidCommit(xmin):
  tuple inserted by committed transaction
else:
  tuple invisible
```
删除者 `xmax` 的核心判断类似：
```text
if XidInMVCCSnapshot(xmax, snapshot):
  deleter still running, tuple still visible
else if TransactionIdDidCommit(xmax):
  tuple deleted
else:
  deleter aborted, tuple visible
```
这解释了本节开头的现象。
standby 查询看不到未提交 primary insert，不是因为 standby 有本地写事务。
而是因为：
```text
tuple xmin = X
GetSnapshotData() 从 KnownAssignedXids 拷出了 X
XidInMVCCSnapshot(X, snapshot) = true
HeapTupleSatisfiesMVCC() 把插入者视为 still running
```
commit replay 后，新 snapshot 中 X 被移除。
再读 tuple 时：
```text
XidInMVCCSnapshot(X, new_snapshot) = false
TransactionIdDidCommit(X) = true
tuple visible
```
旧 snapshot 仍保持原判断。
这就是 WAL replay 与 MVCC snapshot 的结合点。
## 27. replay 与只读查询并发的正确性边界
startup process 持续 replay WAL。
只读 backend 同时执行查询。
二者共享：
```text
ProcArrayLock
KnownAssignedXids
TransamVariables->latestCompletedXid
TransamVariables->xactCompletionCount
procArray->lastOverflowedXid
MyProc->xmin
```
并发规则可以分层理解：
```text
KnownAssignedXids append:
  startup process 单 writer，barrier 保证读者看见完整条目。
KnownAssignedXids remove/compress:
  startup process 持 ProcArrayLock exclusive。
GetSnapshotData:
  backend 持 ProcArrayLock shared，复制出稳定 snapshot。
HeapTupleSatisfiesMVCC:
  使用已经复制出的 snapshot，不反复读取 KnownAssignedXids。
```
所以 snapshot 一旦生成，它的 running set 对该查询稳定。
startup process 后续 replay commit record，不会改变旧 snapshot 的 `subxip` 数组。
这就是 MVCC 的基本隔离边界。
但这不等于 replay 永远不受查询影响。
某些 WAL record 的 replay 需要删除旧 row version、清理 page、获取 AccessExclusiveLock 或处理 buffer pin。
如果本地只读 backend 的 snapshot 或 pin 阻挡 replay，系统会走 recovery conflict 机制。
那是本节边界，不是 snapshot 构造主线。
## 28. Hot Standby 启用条件
源码中有几层条件容易混淆。
第一，primary 必须产生 standby 所需 WAL 信息：
```text
XLogStandbyInfoActive() == (wal_level >= WAL_LEVEL_REPLICA)
```
这使 checkpoint 能写 running-xacts WAL record，也使 AccessExclusiveLock 等信息进入 WAL。
第二，standby 必须处于 archive recovery / standby recovery，并启用：
```text
ArchiveRecoveryRequested && EnableHotStandby
```
`EnableHotStandby` 对应 `hot_standby` 配置。
第三，startup process 必须达到：
```text
reachedConsistency == true
standbyState == STANDBY_SNAPSHOT_READY
```
第四，postmaster 必须收到信号并允许连接：
```text
PMSIGNAL_BEGIN_HOT_STANDBY
pmState = PM_HOT_STANDBY
connsAllowed = true
```
诊断时要逐层问：
```text
primary 是否生成了足够 WAL 信息？
standby 是否启用了 hot_standby？
recovery 是否已经 consistent？
recovery snapshot 是否 ready？
postmaster 是否已经开放只读连接？
```
不要只看 `pg_is_in_recovery()`。
它说明当前仍在 recovery，不说明 snapshot ready 的历史过程。
## 29. ERROR / cleanup / shutdown 边界
`KnownAssignedXids` 的 owner 是 shared memory。
修改者是 startup process。
只读 backend 只能在受控路径下读取。
创建位置在 ProcArray shared memory 初始化阶段。
启用 hot standby 后，recovery transaction environment 由：
```text
InitRecoveryTransactionEnvironment()
```
建立。
结束或切换到正常运行时：
```text
ShutdownRecoveryTransactionEnvironment()
```
会做：
```text
ExpireAllKnownAssignedTransactionIds()
StandbyReleaseAllLocks()
hash_destroy(RecoveryLockHash)
hash_destroy(RecoveryLockXidHash)
VirtualXactLockTableCleanup()
```
`ExpireAllKnownAssignedTransactionIds()` 持有 `ProcArrayLock` exclusive，清空 known-assigned，并把 `latestCompletedXid` 调整到 `nextXid - 1`。
这不是普通用户事务 abort cleanup。
它是 recovery transaction tracking 的整体关闭。
`GetSnapshotData()` 中 snapshot arrays 使用 `malloc()` 长期复用。
如果第一次分配失败，会 `ereport(ERROR, out of memory)`。
普通 snapshot 注册、active snapshot 和 ResourceOwner cleanup 属于 snapshot manager 主线，不是本节重点。
本节只需要记住：
```text
KnownAssignedXids 的生命周期跟 shared memory / recovery 状态绑定；
query snapshot 是 backend-local copy；
ERROR 后 query snapshot cleanup 不会回滚 startup process 已 replay 的 WAL 状态。
```
## 30. 异常路径一：initial snapshot overflow
如果 `XLOG_RUNNING_XACTS` 表示 subxid overflow：
```text
running->subxid_status == SUBXIDS_MISSING
```
`ProcArrayApplyRecoveryInfo()` 不会立刻开放 snapshot。
它进入：
```text
STANDBY_SNAPSHOT_PENDING
standbySnapshotPendingXmin = latestObservedXid
procArray->lastOverflowedXid = latestObservedXid
```
含义是：
```text
当前知道一部分 running XID；
但缺失的 subxid 可能影响 visibility；
不能让普通只读查询拿这个 snapshot。
```
后续 running-xacts record 如果不再 overflow，系统可以 reset 并重新建立。
如果一直 overflow，但 `oldestRunningXid` 已经超过 pending 边界，也可以认为缺失信息不再影响后续 snapshot。
这个 fallback 体现了 PostgreSQL 常见风格：
```text
信息不完整时不猜；
要么等到完整信息；
要么等到不完整信息落出可见性相关范围。
```
## 31. 异常路径二：completion record 带来未观察 subxid
commit / abort redo 在 standby mode 下会再次调用：
```text
RecordKnownAssignedTransactionIds(max_xid)
```
源码注释很强烈：
```text
This is confusing and it is easy to think this call is irrelevant,
which has happened three times in development already. Leave it in.
```
原因是 transaction completion record 可能包含此前未完全观察到的 subtransaction。
如果只依赖主 replay loop 对 `record->xl_xid` 的处理，就可能漏掉 completion record 中更大的 subxid。
因此 commit / abort redo 必须先补齐到 `max_xid`，再标记 CLOG，再从 `KnownAssignedXids` 移除。
这保证：
```text
所有可能影响 visibility 的 XID 都经历过 assignment tracking；
完成状态写入顺序早于 ProcArray recovery state 移除。
```
## 32. 异常路径三：KnownAssignedXids 空间压力
`KnownAssignedXidsAdd()` 如果发现 head 后空间不足，会先：
```text
KnownAssignedXidsCompress(KAX_NO_SPACE, ...)
```
如果压缩后仍放不下：
```text
elog(ERROR, "too many KnownAssignedXids")
```
这不是普通 workload 下应该出现的可恢复慢路径。
设计上 primary 会定期写 `XLOG_XACT_ASSIGNMENT`，把 subxid 信息转入 `pg_subtrans` 并从 known-assigned 中移除。
同时 running-xacts record 会 prune stale XID。
如果仍溢出，说明 standby 无法维护正确 recovery snapshot，只能报错。
诊断时要关注：
```text
primary 上极端 subtransaction workload；
WAL apply 是否长时间落后；
running-xacts record 是否能被 replay；
是否存在大量 prepared transaction 或异常长事务；
debug 日志中的 KnownAssignedXids display。
```
## 33. 异常路径四：recovery conflict 不是 snapshot 构造失败
`standby.c` 中有大量 `ResolveRecoveryConflict*()`。
本节只标边界。
snapshot conflict 的入口是：
```text
ResolveRecoveryConflictWithSnapshot(snapshotConflictHorizon, ...)
```
它的语义是：
```text
某个 WAL replay 操作需要消除可能看到 XID <= horizon 仍 running 的 standby snapshot。
```
系统会找出冲突 backend 的 virtual XID，并等待或发信号取消。
等待事件包括：
```text
RecoveryConflictSnapshot
RecoveryConflictTablespace
RecoveryConflictBufferPin
```
`pg_stat_database_conflicts` 暴露：
```text
confl_snapshot
confl_lock
confl_bufferpin
confl_tablespace
confl_deadlock
```
这些指标说明：
```text
replay 与只读查询发生了资源或可见性冲突。
```
它们不说明：
```text
GetSnapshotData() 没能生成 snapshot。
```
把这两件事混起来，会把“snapshot visibility 机制”误诊成“冲突取消策略”。
## 34. 成本模型
hot standby snapshot 的成本主要来自四处：
| 成本点 | 扩张方式 |
| --- | --- |
| `GetSnapshotData()` 复制 `KnownAssignedXids` | O(S)，S 是 `headKnownAssignedXids - tailKnownAssignedXids`，gap 多时大于有效 XID 数。 |
| completion replay 删除 XID | `KnownAssignedXidsSearch()` 是 O(logS)，但需要 exclusive `ProcArrayLock`。 |
| subxid overflow | `XidInMVCCSnapshot()` 可能走 `SubTransGetTopmostTransaction()`，引入 SLRU 和额外分支。 |
| WAL 信息量 | `XLOG_RUNNING_XACTS`、`XLOG_XACT_ASSIGNMENT` 是 hot standby / logical correctness 的信息成本。 |
高 replay 速率和大量只读 snapshot 并发时，exclusive 删除 / compress 与 shared snapshot 获取会形成 contention。
成本传播链路是：primary subxid pattern -> WAL assignment records -> startup process 维护 `KnownAssignedXids` -> standby `GetSnapshotData()` copy -> heap visibility slow path。
## 35. 可观测与不可观测
能直接看见的：
| 入口 | 能看见什么 |
| --- | --- |
| server log | `consistent recovery state reached`、`database system is ready to accept read-only connections`、debug 下 `recovery snapshots are now enabled`。 |
| `SHOW in_hot_standby` | 当前会话是否处于 hot standby recovery 语境。 |
| `SELECT pg_is_in_recovery()` | 当前实例是否仍在 recovery。 |
| `pg_stat_recovery` | replay LSN、replay end、pause state 等 recovery 进度。 |
| `pg_stat_wal_receiver` | 接收端连接、written / flushed LSN、上游信息。 |
| `pg_stat_database_conflicts` | 已发生的 snapshot / lock / buffer pin 等冲突计数。 |
| `pg_stat_activity.wait_event` | `RecoveryWalStream`、`RecoveryConflictSnapshot` 等等待点。 |
只能间接推断的：
```text
当前 KnownAssignedXids 的具体内容；
standbyState 从 INITIALIZED 到 READY 的精确瞬间；
某个 query snapshot 的 subxip 内容；
lastOverflowedXid 对某次 tuple visibility slow path 的贡献。
```
源码调试能看见，但 SQL 层没有直接视图。
几乎不可见或不应强行归因的：
```text
primary 取 running-xacts snapshot 与写 WAL record 的 race 中具体哪些 XID 被过滤；
某次 hint bit 设置是否因为 recovery snapshot 而延后；
barrier 级别的数组可见性。
```
这些需要 gdb、trace log、断点或临时 instrumentation。
不要把 `pg_stat_*` 当成完整因果图。
## 36. 源码断点诊断入口
如果要源码跟踪，按进程分三组断点：
| 进程 | 断点 |
| --- | --- |
| primary | `LogStandbySnapshot`、`GetRunningTransactionData`、`LogCurrentRunningXacts`。 |
| standby startup process | `InitRecoveryTransactionEnvironment`、`ProcArrayInitRecovery`、`RecordKnownAssignedTransactionIds`、`standby_redo`、`ProcArrayApplyRecoveryInfo`、`ExpireOldKnownAssignedTransactionIds`、`ExpireTreeKnownAssignedTransactionIds`、`ProcArrayApplyXidAssignment`。 |
| standby read-only backend | `GetSnapshotData`、`KnownAssignedXidsGetAndSetXmin`、`XidInMVCCSnapshot`、`HeapTupleSatisfiesMVCC`。 |
观察重点：`ProcArrayApplyRecoveryInfo()` 看 `standbyState` 与 running-xacts 边界；`RecordKnownAssignedTransactionIds()` 看 `xid`、`latestObservedXid` 和是否进入 `KnownAssignedXidsAdd()`；`GetSnapshotData()` 看 `takenDuringRecovery`、`xmax`、`subcount`、`suboverflowed`。
轻量 instrumentation 应放在 startup process 路径打印 `KnownAssignedXids` 的 num/tail/head、`standbyState` transition 和 running-xacts record LSN；不要在 `HeapTupleSatisfiesMVCC()` 热路径大量打日志。
## 37. 课堂实验一：未提交 primary insert 在 standby 不可见
环境：
```text
primary / standby physical streaming replication
wal_level = replica
hot_standby = on
```
primary：
```sql
CREATE TABLE hs_demo(id int primary key);
BEGIN;
INSERT INTO hs_demo VALUES (1);
SELECT txid_current();
```
保持事务不提交。
standby：
```sql
SELECT pg_is_in_recovery();
SHOW in_hot_standby;
SELECT * FROM hs_demo;
```
预期：
```text
如果 DDL 已 replay，查询能访问表；
未提交 insert 不可见。
```
然后 primary：
```sql
COMMIT;
```
standby 等 replay 追上后：
```sql
SELECT * FROM hs_demo;
```
预期看到 `(1)`。
源码解释：
```text
commit 前 snapshot->subxip 包含插入事务 XID，XidInMVCCSnapshot() 为 true；
commit record replay 后 ExpireTreeKnownAssignedTransactionIds() 移除该 XID；
新 snapshot 不再包含它，且 CLOG 显示 committed。
```
## 38. 课堂实验二：观察 readiness 日志
重启 standby 或从 base backup 启动 standby。
调高日志级别：
```text
log_min_messages = debug1
```
观察日志顺序：
```text
initializing for hot standby
consistent recovery state reached at ...
recovery snapshots are now enabled
database system is ready to accept read-only connections
```
不同场景下顺序可能交错，但判断标准不变：
```text
consistent state 是数据恢复一致性；
recovery snapshots are now enabled 是 standbyState 到 READY；
ready to accept read-only connections 是 postmaster 允许连接。
```
如果长期看不到 snapshot enabled，优先怀疑：
```text
standby 尚未 replay 到 running-xacts record；
initial running-xacts snapshot overflow 且 pending 条件未满足；
WAL 获取或 replay 被阻塞。
```
## 39. 常见误区
| 误区 | 修正 |
| --- | --- |
| standby 没有本地写事务，所以 snapshot 里不需要 running XID。 | tuple 的 `xmin` / `xmax` 来自主库事务，standby snapshot 必须解释 primary XID。 |
| consistent recovery state reached 就等于可以接受只读查询。 | 还要 `STANDBY_SNAPSHOT_READY`，并由 postmaster 处理 `PMSIGNAL_BEGIN_HOT_STANDBY`。 |
| running-xacts record 就是用户 backend 使用的 snapshot。 | 它是 WAL 输入；standby replay 后合并进 `KnownAssignedXids`，用户 backend 再生成自己的 `SnapshotData`。 |
| `KnownAssignedXids` 只保存 WAL 中直接出现过的 XID。 | `RecordKnownAssignedTransactionIds()` 会根据 XID 顺序分配规则补齐未直接观察到的 gap。 |
| snapshot conflict 表示 snapshot 构造错了。 | conflict 表示 replay 需要推进，而某些 standby backend 的旧 snapshot 或 pin 阻挡了 replay。 |
| `subxip` 在 recovery snapshot 中只保存 subtransaction。 | recovery snapshot 把 known-assigned top XID 和 subxid 都放进 `subxip`，并依赖 `takenDuringRecovery` 改变解释方式。 |
## 40. 讨论题
1. 为什么 standby 不能通过扫描本地只读 backend 的 ProcArray 来构造 MVCC snapshot？
2. 如果 primary 在取 running-xacts snapshot 后、写 `XLOG_RUNNING_XACTS` 前提交了某事务，standby 如何避免把它错误保留为 running？
3. `RecordKnownAssignedTransactionIds()` 为什么要把 `(latestObservedXid, xid]` 都加入 known-assigned，而不是只加入当前 WAL record 的 `xl_xid`？
4. 为什么 commit / abort redo 必须先更新 CLOG，再调用 `ExpireTreeKnownAssignedTransactionIds()`？
5. `STANDBY_SNAPSHOT_PENDING` 为什么不能直接开放只读查询？
6. `GetSnapshotData()` recovery 分支为什么把 XID 放进 `subxip` 而不是 `xip`？
7. `pg_stat_database_conflicts.confl_snapshot` 能说明什么，不能说明什么？
8. 如果一个 standby 长期不能进入 hot standby ready，应按哪些层次排查？
## 41. 本节小结
本节主链路是：
```text
primary checkpoint 前后写 running-xacts WAL
  -> standby startup process 初始化 recovery transaction tracking
  -> replay 每条带 XID 的 WAL 时维护 latestObservedXid 和 KnownAssignedXids
  -> replay XLOG_RUNNING_XACTS 时建立或修正 primary running set
  -> standbyState 进入 STANDBY_SNAPSHOT_READY
  -> postmaster 在 recovery consistent 后允许只读连接
  -> backend GetSnapshotData() 从 KnownAssignedXids 生成 recovery snapshot
  -> HeapTupleSatisfiesMVCC() 通过 XidInMVCCSnapshot() 解释 primary XID
```
核心状态边界是：
```text
standbyState:
  startup process 的 recovery snapshot readiness。
KnownAssignedXids:
  shared memory 中 primary running XID 的 recovery-time 表达。
SnapshotData:
  用户 backend 的 backend-local copy。
tuple hint / CLOG:
  具体 XID 完成状态的持久来源。
```
正确性不是单个字段保证的。
它来自：
```text
WAL 顺序
running-xacts record
XID 顺序分配
ProcArrayLock 与 memory barrier
CLOG-before-ProcArray 更新顺序
snapshot 的 backend-local 稳定副本
```
异常路径的规律是：
```text
信息不完整时不开放 snapshot；
缺失 subxid 时 pending；
completion record 可能补齐未观察 XID；
stale known-assigned 用 running-xacts prune；
replay 与旧 snapshot 冲突时走 recovery conflict，而不是破坏 MVCC。
```
可迁移的 mental model：
```text
在异步复制系统里，read-only 节点的 snapshot 不是“本地没有 writer”这个事实的结果；
它是上游事务时间线被编码、replay、合并并复制到本地查询上下文后的结果。
```
诊断时必须分清：
```text
WAL 是否到达；
replay 是否推进；
recovery 是否 consistent；
snapshot state 是否 ready；
query snapshot 是否太旧；
replay conflict 是否在取消查询。
```
这些层次相邻，但不是同一个问题。
