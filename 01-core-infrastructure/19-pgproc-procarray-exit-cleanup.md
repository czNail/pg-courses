# PostgreSQL 事务结束、backend 退出与 stale state 清理

## 课程定位

前置知识：已经理解 `PGPROC` 是 backend 在 shared memory 中的身份，`ProcArray` 是事务状态 publication set，snapshot 和 cleanup horizon 都依赖 ProcArray 中的 XID、SubXID、xmin 与状态 flag。

本节唯一主问题：

```text
commit、abort、FATAL exit 和 postmaster cleanup 如何把 PGPROC / ProcArray 状态恢复到可复用边界，哪些状态必须先对其它 backend 不再可见，哪些资源仍要交给后续 cleanup 阶段？
```

核心矛盾：backend 退出时必须尽快从其它 backend 的并发视野中消失，否则 snapshot、VACUUM、lock wait、replication feedback 都可能被 stale state 卡住；但它又不能过早释放锁、PGPROC、latch、semaphore、buffer pin、DSM、临时文件等资源，否则其它 backend 可能在“事务状态还没定案”时观察到不完整清理。PostgreSQL 的解法不是一个“大 cleanup 函数”，而是把清理拆成几层严格排序的边界：

```text
先把事务结果写入 WAL / pg_xact
  -> 在 ProcArray 中宣布 XID 不再 running
  -> 释放对其它 backend 可见的资源
  -> 释放 heavyweight locks
  -> 清理 backend-local 状态
  -> backend 退出时从 ProcArray 移除 PGPROC
  -> 最后归还 PGPROC 槽位、latch、semaphore 等低层身份资源
```

学完后应能判断：

```text
为什么 ProcArrayEndTransaction() 必须在释放锁之前执行；
为什么 ProcArrayEndTransaction() 不是 ProcKill()；
为什么有 XID 的事务结束必须拿 ProcArrayLock exclusive；
为什么读写事务 commit/abort 要推进 latestCompletedXid 和 xactCompletionCount；
为什么 read-only 事务结束可以少走 ProcArrayLock；
为什么 2PC 不能直接清掉 running state，而要先转移到 dummy PGPROC；
FATAL exit 为什么先走 before_shmem_exit，再走 on_shmem_exit；
backend 崩溃时 postmaster 为什么不尝试精细修复单个 PGPROC，而是重置共享内存；
哪些 stale state 会影响 snapshot / VACUUM / lock wait / backend slot reuse；
如何用 SQL、日志和 gdb 观察这些清理边界。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

前几节的主线是从进入系统到被其它 backend 观察：

```text
InitProcess()
  -> 获得 PGPROC
InitProcessPhase2()
  -> ProcArrayAdd(MyProc)
GetNewTransactionId()
  -> 把 XID 发布到 ProcGlobal->xids[]
GetSnapshotData()
  -> 扫描 ProcArray
ComputeXidHorizons()
  -> 汇总 xmin / slot xmin / recovery 状态
```

本节看反向生命周期：

```text
事务结束或 backend 退出
  -> 哪些字段必须先从 ProcArray 视野中撤掉
  -> 哪些资源必须在锁还没释放前完成可见性传播
  -> 哪些清理只能在 process exit callback 中兜底
  -> PGPROC 什么时候才可以重新放回 freelist
```

这不是“回收内存”的问题，而是 shared state 的可见性顺序问题。一个 stale `PGPROC` entry 可能造成完全不同类型的症状：

| stale state | 典型后果 |
| --- | --- |
| `ProcGlobal->xids[]` 未清理 | snapshot 继续认为事务 running，visibility 与 `xmax` 边界错误。 |
| `MyProc->xmin` 未清理 | VACUUM cleanup horizon 被钉住，old tuple version 无法移除。 |
| `subxidStatus` 未清理 | snapshot / subtransaction 查询需要走更保守路径。 |
| `statusFlags` 未清理 | VACUUM / logical decoding horizon 分类错误。 |
| lock wait state 未清理 | lock manager 可能仍认为 backend 在等待或持有锁。 |
| lock group membership 未清理 | leader / member PGPROC 回收顺序错误。 |
| `pid` / latch / semaphore 未清理 | 新 backend 复用 PGPROC 时可能继承旧 wait state。 |

因此本节的主线是：

```text
清理顺序本身就是 correctness 机制。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
事务 commit / abort 先把结果持久化到 WAL / pg_xact，再通过 ProcArrayEndTransaction() 在 ProcArrayLock exclusive 下清除 running XID、xmin、SubXID cache 和相关状态 flag；随后事务层按 ResourceOwner 三阶段释放资源和锁；backend 进程真正退出时，proc_exit() 先运行 before_shmem_exit 做需要事务系统仍可用的清理，再运行 on_shmem_exit 移除 ProcArray entry、释放低层 shared memory 资源，最后 ProcKill() 归还 PGPROC。
```

这背后的 tension 是：

```text
其它 backend 需要尽快停止把我当作 running transaction；
但锁、buffer pin、catalog invalidation、文件删除、prepared transaction、replication slot、DSM 等资源必须在正确阶段释放。
```

PostgreSQL 把这个问题拆成三种不同的“结束”：

| 结束类型 | 语义 | 代表函数 |
| --- | --- | --- |
| transaction end | 当前事务不再 running，事务结果已经写入 pg_xact。 | `ProcArrayEndTransaction()` |
| backend shmem exit | backend 将离开共享内存世界，低层 shared state 要归还。 | `shmem_exit()`、`RemoveProcFromArray()`、`ProcKill()` |
| postmaster crash recovery | 单个 child 没有可信地清理共享内存，整套 shared memory 需要重置。 | `CleanupBackend()`、`HandleChildCrash()`、`ResetShmemAllocator()` |

不要把它们混为一谈。

`ProcArrayEndTransaction()` 的含义不是“backend 退出”。它只说：

```text
这个 transaction 的 XID 不再属于 running set。
```

`ProcKill()` 的含义也不是“事务结束”。它说：

```text
这个 OS process 不再拥有这个 PGPROC 槽位。
```

postmaster 的 crash path 更不是“帮 backend 调用 ProcKill”。它的原则是：

```text
如果 child 以不可信方式死亡，共享内存可能已经损坏；
postmaster 不在旧 shared memory 里做精细手术，而是终止其它 child 并重建 shared memory。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xact.c` | `CommitTransaction()`、`AbortTransaction()`、`CleanupTransaction()`、`AbortOutOfAnyTransaction()` 的事务结束顺序。 |
| 2 | `src/backend/storage/ipc/procarray.c` | `ProcArrayEndTransaction()`、`ProcArrayEndTransactionInternal()`、`ProcArrayGroupClearXid()`、`ProcArrayClearTransaction()`、`ProcArrayRemove()` 如何清理 publication state。 |
| 3 | `src/backend/storage/lmgr/proc.c` | `InitProcess()` / `InitProcessPhase2()` 注册退出 callback，`LockErrorCleanup()`、`ProcReleaseLocks()`、`RemoveProcFromArray()`、`ProcKill()` 如何处理 lock wait 与 PGPROC 回收。 |
| 4 | `src/backend/storage/ipc/ipc.c` | `proc_exit()`、`proc_exit_prepare()`、`shmem_exit()`、`before_shmem_exit()`、`on_shmem_exit()` 的 callback 分层。 |
| 5 | `src/backend/utils/init/postinit.c` | `ShutdownPostgres()` 如何在 backend 退出时保证 active transaction 被 abort。 |
| 6 | `src/backend/tcop/postgres.c` | `ProcessInterrupts()`、`die()`、`quickdie()` 在 SIGTERM、FATAL、SIGQUIT 下如何进入不同清理路径。 |
| 7 | `src/backend/postmaster/postmaster.c` | `CleanupBackend()`、`HandleChildCrash()`、crash recovery 中 postmaster 为什么重建 shared memory。 |
| 8 | `src/backend/access/transam/twophase.c` | prepared transaction 如何用 dummy `PGPROC` 延续 ProcArray / lock 状态。 |
| 9 | `src/backend/utils/time/snapmgr.c` | `AtEOXact_Snapshot()` 与 `SnapshotResetXmin()` 为什么在普通 commit 中不抢 `ProcArrayEndTransaction()` 的职责。 |
| 10 | `src/include/storage/proc.h` | `PGPROC`、`PROC_HDR`、dense arrays、freelist、`procArrayGroup*` 字段的状态边界。 |
| 11 | `src/backend/access/transam/README` | transaction end 与 snapshot-taking 的 interlocking 证明。 |

推荐阅读顺序：

```text
先读 xact.c 的 CommitTransaction() / AbortTransaction()
  -> 找到 ProcArrayEndTransaction() 在事务结束中的位置
  -> 跳到 procarray.c 看清 XID / xmin / SubXID / flags 的原子清理边界
  -> 回到 xact.c 看 ResourceOwner 三阶段释放为什么在它之后
  -> 读 proc.c 和 ipc.c 理解 process exit callback 如何兜底
  -> 最后读 postmaster.c，区分正常 FATAL exit 与不可信 crash reset
```

不要从 `ProcKill()` 开始理解事务结束。那样容易误以为“释放 PGPROC 就等于事务完成”。在 PostgreSQL 中，事务对其它 backend 的语义完成点通常早于 PGPROC 槽位回收。

## 4. 关键数据结构与状态

### `PGPROC`: 进程身份，不等于当前事务

`PGPROC` 同时服务多条边界：

```text
lock manager wait identity
ProcArray membership identity
snapshot / xmin publication identity
latch / semaphore wait target
sync replication wait target
lock group membership
process-local MyProc pointer
```

因此 `PGPROC` 的回收不能只看“事务是否结束”。同一个 backend 可以连续执行很多事务，`PGPROC` 在整个 backend 生命周期中保持存在；只有 backend 进程退出时，`ProcKill()` 才把它放回 freelist。

关键字段组合：

| 字段 | 清理语义 |
| --- | --- |
| `xid` | 当前 top-level XID。事务结束时清成 invalid。 |
| `vxid` | virtual transaction identity。事务结束或 PGPROC 退出时清理。 |
| `xmin` | active / registered snapshot 对 cleanup horizon 的需求。事务结束或 snapshot reset 时清理。 |
| `subxidStatus` | subtransaction cache 状态。事务结束时必须和 dense array 一起清理。 |
| `statusFlags` | vacuum / logical decoding / horizon 相关状态。事务结束时清理 vacuum state mask。 |
| `delayChkptFlags` | checkpoint delay 相关状态。abort/transaction end 要清掉。 |
| `pid` | OS process 是否仍拥有该 slot。`ProcKill()` 置 0。 |
| `procLatch` / `sem` | 其它 backend 唤醒或等待本 backend 的低层对象。PGPROC 回收前要 disown / reset。 |

本课要反复强调：

```text
事务状态清理发生在 PGPROC 仍然有效时；
PGPROC 回收发生在事务和低层 shared state 都不再依赖它之后。
```

### `PROC_HDR` dense arrays: ProcArray scan 的真实热路径

`src/include/storage/proc.h` 中说明，一些 `PGPROC` 字段被镜像到更紧凑的数组：

```text
ProcGlobal->xids[]
ProcGlobal->subxidStates[]
ProcGlobal->statusFlags[]
ProcArrayStruct->pgprocnos[]
```

这些数组只包含已经 `ProcArrayAdd()` 的 backend / prepared transaction entry。它们和 `PGPROC` 的关系是：

```text
PGPROC 是固定槽位，可以有未使用洞；
dense arrays 是 ProcArray scan 的紧凑视图；
PGPROC->pgxactoff 是进入 dense arrays 的偏移；
ProcArrayAdd() / ProcArrayRemove() 会改变其它 proc 的 pgxactoff。
```

所以清理 dense arrays 有两个约束：

```text
访问 pgxactoff 必须持有 ProcArrayLock 或 XidGenLock；
只要 PGPROC 还在 ProcArray 中，PGPROC 字段和 dense arrays 必须语义一致。
```

这解释了为什么 `ProcArrayEndTransactionInternal()` 同时改：

```text
ProcGlobal->xids[pgxactoff]
proc->xid
proc->vxid.lxid
proc->xmin
ProcGlobal->subxidStates[pgxactoff]
proc->subxidStatus
ProcGlobal->statusFlags[pgxactoff]
proc->statusFlags
```

它不是重复劳动，而是在维护两个访问模式：

```text
单 backend 快速看自己的 PGPROC；
全局扫描快速看 dense arrays。
```

### `latestCompletedXid` 与 `xactCompletionCount`

事务结束不是只把 `xid` 清成 invalid。对于有 XID 的事务，`ProcArrayEndTransactionInternal()` 还会推进：

```text
TransamVariables->latestCompletedXid
TransamVariables->xactCompletionCount
```

这两者服务 snapshot 可复用和 correctness：

```text
latestCompletedXid
  -> GetSnapshotData() 可以用 latestCompletedXid + 1 作为 snapshot xmax 的候选边界

xactCompletionCount
  -> snapshot cache 逻辑能知道自上次 snapshot 以来是否有事务完成
```

因此这两个更新必须和“退出 running set”处于同一个 ProcArray interlock 中。否则 snapshot 可能看到一个互相矛盾的世界：

```text
某个 XID 不在 ProcArray running set 中
但 latestCompletedXid / completion counter 还没反映完成
```

### `procArrayGroup*`: commit storm 下的批量清 XID

很多 backend 同时提交时，如果每个 backend 都独占 `ProcArrayLock` 清自己的 XID，会在 commit hot path 上形成严重锁交接成本。`ProcArrayGroupClearXid()` 用 `PGPROC` 中的 group clear 字段把多个提交者串成一组：

```text
第一个进入者成为 leader
  -> 拿 ProcArrayLock exclusive
  -> 代表链表中的多个 PGPROC 调 ProcArrayEndTransactionInternal()
  -> 放锁后唤醒 follower

后续进入者成为 follower
  -> 把自己挂到 procArrayGroupFirst
  -> 等待 leader 清理自己的 XID
```

这是一种很典型的 PostgreSQL hot path 折中：

```text
正确性仍由同一个 exclusive ProcArrayLock 区间保证；
性能通过把多次 lock handoff 合并成一次完成。
```

### exit callback lists: backend 退出不是单点函数

`src/backend/storage/ipc/ipc.c` 中有三类 callback：

| callback | 运行阶段 | 典型用途 |
| --- | --- | --- |
| `before_shmem_exit` | `shmem_exit()` 早期 | 需要事务系统、catalog、shared state 仍尽量可用的清理，例如 abort active transaction、temp relation cleanup。 |
| `on_shmem_exit` | `before_shmem_exit` 和 DSM cleanup 之后 | 低层 shared memory 资源释放，例如从 ProcArray 移除、释放 PGPROC、清 buffer pin、清 proc signal state。 |
| `on_proc_exit` | shmem cleanup 完成后 | 不依赖 shared memory 的 process-local cleanup，例如关闭 socket、日志、smgr shutdown。 |

`InitProcess()` 注册：

```text
on_shmem_exit(ProcKill, 0)
```

`InitProcessPhase2()` 注册：

```text
on_shmem_exit(RemoveProcFromArray, 0)
```

由于 callback 以后进先出的方式执行，而 `RemoveProcFromArray` 在 `ProcKill` 之后注册，所以 backend 退出时顺序是：

```text
RemoveProcFromArray()
  -> ProcArrayRemove(MyProc, InvalidTransactionId)

ProcKill()
  -> 释放 / 归还 PGPROC 槽位
```

这正是“先从 ProcArray 消失，再归还 PGPROC”的顺序。

## 5. 主流程源码 walkthrough

### 5.1 正常 commit: 先 durable commit，再从 running set 消失

主链路在 `CommitTransaction()`：

```text
CommitTransaction()
  -> HOLD_INTERRUPTS()
  -> RecordTransactionCommit()
       把 commit 结果写到 WAL / pg_xact
  -> ProcArrayEndTransaction(MyProc, latestXid)
       向其它 backend 宣布本 XID 不再 running
  -> ResourceOwnerRelease(BEFORE_LOCKS)
       释放 buffer pin、relcache ref 等锁前资源
  -> AtEOXact_Inval(true)
       发送 catalog / relcache invalidation
  -> ResourceOwnerRelease(LOCKS)
       释放 heavyweight locks
  -> ResourceOwnerRelease(AFTER_LOCKS)
  -> smgrDoPendingDeletes(true)
  -> AtCommit_Notify()
  -> backend-local cleanup
```

源码注释明确强调：

```text
ProcArrayEndTransaction 必须在释放锁之前执行；
同时必须在 RecordTransactionCommit 之后执行。
```

原因是 commit 后其它 backend 可能马上被锁唤醒。如果锁先释放，而 XID 仍在 ProcArray 中，等待者可能看到：

```text
锁已经可获得
但阻塞自己的事务仍显示 running
```

反过来，如果在 WAL / pg_xact 记录 commit 之前就从 ProcArray 清除 XID，snapshot taker 可能看到：

```text
这个事务不 running
但查询事务状态时还没有可靠 commit / abort 结果
```

所以 commit 的外部可见顺序是：

```text
durable result
  -> running set removal
  -> lock release / waiters continue
```

这条顺序是本课最重要的不变量。

### 5.2 `ProcArrayEndTransaction()`: 有 XID 和无 XID 的分叉

`ProcArrayEndTransaction()` 先看 `latestXid` 是否 valid。

有 XID：

```text
ProcArrayEndTransaction()
  -> 需要 ProcArrayLock exclusive
  -> 若能立即拿锁，直接 ProcArrayEndTransactionInternal()
  -> 若不能，进入 ProcArrayGroupClearXid()
```

无 XID：

```text
ProcArrayEndTransaction()
  -> 不需要修改 running XID set
  -> 不需要推进 latestCompletedXid
  -> 清 vxid.lxid / xmin / delayChkptFlags
  -> 如有 vacuum state flag，短暂拿 ProcArrayLock 清 dense statusFlags
```

这个分叉解释了读写事务和纯读事务结束成本不同：

```text
有 XID 的事务结束会改变其它 backend 的 snapshot correctness；
无 XID 的事务结束通常只影响 xmin horizon 的保守估计。
```

源码 README 中的 interlocking 规则是：

```text
任何有 XID 的事务不能在别人取 snapshot 的中途退出 running set。
```

`GetSnapshotData()` 持 `ProcArrayLock` shared，多个 snapshot taker 可以并发；`ProcArrayEndTransaction()` 持 exclusive，和 snapshot-taking 互斥。

### 5.3 `ProcArrayEndTransactionInternal()`: 同一个锁区间内清 publication state

内部清理的语义可以压缩成：

```text
在 ProcArrayLock exclusive 下:
  清 dense xids[]
  清 PGPROC->xid
  清 VXID local id
  清 PGPROC->xmin
  清 delay checkpoint flags
  清 vacuum 状态 flag
  清 dense subxidStates[]
  清 PGPROC->subxidStatus
  推进 latestCompletedXid
  增加 xactCompletionCount
```

这里有两个容易漏掉的点。

第一，`xmin` 和 `xid` 必须一起看：

```text
xid 表示我这个事务本身是否 running；
xmin 表示我是否仍持有旧 snapshot 会阻止 cleanup。
```

普通事务结束时，二者都不应继续发布给其它 backend。

第二，`statusFlags` 的清理不是 cosmetic：

```text
PROC_IN_VACUUM
PROC_IN_LOGICAL_DECODING
PROC_AFFECTS_ALL_HORIZONS
```

这些 flag 会影响 horizon 计算。如果 backend 已经结束事务但 vacuum state flag 仍留在 dense array 中，后续 `ComputeXidHorizons()` 可能按错误 observer 类型解释这个 slot。

### 5.4 abort: 先脱离不安全等待，再记录 abort，再清 ProcArray

`AbortTransaction()` 的早期动作比 commit 更像 emergency cleanup：

```text
AbortTransaction()
  -> HOLD_INTERRUPTS()
  -> AtAbort_Memory()
  -> AtAbort_ResourceOwner()
  -> LWLockReleaseAll()
  -> WaitLSNCleanup()
  -> pgstat wait / progress cleanup
  -> UnlockBuffers()
  -> XLogResetInsertion()
  -> ConditionVariableCancelSleep()
  -> LockErrorCleanup()
  -> reschedule_timeouts()
  -> sigprocmask(... UnBlockSig ...)
```

这些动作发生在记录 abort 之前，目标是把 backend 从危险的半等待状态中拉出来：

```text
不能带着 LWLock 进入复杂 abort cleanup；
不能仍挂在 lock wait queue 上再去申请其它锁；
不能保留 buffer content lock；
不能让 condition variable / wait event 继续描述旧等待。
```

之后才进入事务 abort 主线：

```text
AbortTransaction()
  -> 清理 trigger / portal / pending sync / notify / relation map 等
  -> RecordTransactionAbort(false)
  -> ProcArrayEndTransaction(MyProc, latestXid)
  -> ResourceOwnerRelease(BEFORE_LOCKS)
  -> AtEOXact_Inval(false)
  -> ResourceOwnerRelease(LOCKS)
  -> ResourceOwnerRelease(AFTER_LOCKS)
  -> smgrDoPendingDeletes(false)
  -> backend-local cleanup
```

这里和 commit 的共同点更重要：

```text
RecordTransactionAbort()
  -> ProcArrayEndTransaction()
  -> lock release
```

abort 也必须先让事务状态有结果，再从 running set 中消失，最后才释放锁。

### 5.5 `CleanupTransaction()`: abort 之后的 backend-local 收尾

`AbortTransaction()` 结束后，事务状态保持 `TRANS_ABORT`，直到 `CleanupTransaction()`：

```text
CleanupTransaction()
  -> AtCleanup_Portals()
  -> AtEOXact_Snapshot(false, true)
  -> ResourceOwnerDelete()
  -> AtCleanup_Memory()
  -> 清 TransactionState 字段
  -> s->state = TRANS_DEFAULT
```

这解释了 abort 清理分两段的原因：

```text
AbortTransaction()
  -> 对外宣布事务不再 running，并释放共享资源 / locks

CleanupTransaction()
  -> 清理 portal、snapshot、resource owner、memory context 等 backend-local 状态
```

不要把 `CleanupTransaction()` 误读成 ProcArray 清理点。对其它 backend 来说，XID 的 running state 已经在 `AbortTransaction()` 中通过 `ProcArrayEndTransaction()` 消失。

### 5.6 subtransaction: 释放子事务资源，但不拥有独立 VXID

subtransaction 不拥有自己的 VXID，它共享 top transaction 的 VXID。子事务 commit / abort 的清理重点不同：

```text
subtransaction commit
  -> 子事务锁和资源转移给父事务
  -> 删除 subtransaction XID lock
  -> 清理子事务 memory / resource owner

subtransaction abort
  -> 释放子事务持有的资源
  -> 清理等待状态、buffer lock、LWLock
  -> 回到父事务或继续 abort 父事务
```

顶层事务的 `ProcArrayEndTransaction()` 仍然是最终从 running set 消失的主要边界。SubXID cache 在顶层事务结束时一起清理，或者在运行中通过 overflow flag 告诉 snapshot taker 不要完全信任本地缓存。

### 5.7 prepared transaction: 从当前 backend 转移到 dummy PGPROC

2PC 是本节最容易暴露“PGPROC 不等于 backend process”的路径。

`PrepareTransaction()` 中有一个关键顺序：

```text
EndPrepare(gxact)
  -> PostPrepare_Locks(fxid)
       把 locks 转移到 prepared transaction 的 dummy PGPROC
  -> ProcArrayClearTransaction(MyProc)
       清掉当前 backend 自己的 XID / VXID / xmin / SubXID cache
  -> PostPrepare_* cleanup
  -> PostPrepare_Twophase()
       允许其它 backend 后续 COMMIT PREPARED / ROLLBACK PREPARED
```

为什么不是直接 `ProcArrayEndTransaction(MyProc)`？

因为 prepared transaction 还没有 commit 或 abort。它必须继续被系统视为“可能影响 visibility / locks 的事务状态”。做法是：

```text
原 backend 的 MyProc 不再代表这个事务；
prepared transaction 的 dummy PGPROC 继续留在 ProcArray 和 lock manager 视野中。
```

`ProcArrayClearTransaction()` 的注释说得很关键：

```text
我们没有报告该事务 XID 不再 running；
因为 gxact 对应的 ProcArray entry 仍会让它显示为 running。
```

后续 `FinishPreparedTransaction()` 中：

```text
RecordTransactionCommitPrepared() 或 RecordTransactionAbortPrepared()
  -> ProcArrayRemove(dummy proc, latestXid)
  -> post commit / abort callbacks
  -> 释放 prepared transaction 持有的锁
```

这条路径对应同一个外部顺序：

```text
先写事务结果
  -> 从 ProcArray 移除 prepared XID
  -> 再释放锁
```

只是执行者已经不是原来的 backend。

### 5.8 backend 正常退出 / FATAL exit: `proc_exit()` 的分层清理

普通连接结束、`ereport(FATAL)`、管理员终止 backend 等路径最终通常进入 `proc_exit()`：

```text
proc_exit(code)
  -> proc_exit_prepare(code)
       设置 proc_exit_inprogress
       清 pending interrupt flags
       清 error_context_stack / debug_query_string
       shmem_exit(code)
       on_proc_exit callbacks
  -> exit(code)
```

`shmem_exit()` 的顺序是：

```text
LWLockReleaseAll()
  -> before_shmem_exit callbacks
  -> dsm_backend_shutdown()
  -> on_shmem_exit callbacks
```

其中 `ShutdownPostgres()` 是重要的 `before_shmem_exit` callback：

```text
ShutdownPostgres()
  -> AbortOutOfAnyTransaction()
  -> LockReleaseAll(USER_LOCKMETHOD, true)
```

这保证即使 backend 因 FATAL 退出，只要还能安全运行 callback，它也会先尝试：

```text
abort active transaction
  -> ProcArrayEndTransaction()
  -> ResourceOwner / lock cleanup
```

随后进入 `on_shmem_exit` 阶段：

```text
RemoveProcFromArray()
  -> ProcArrayRemove(MyProc, InvalidTransactionId)

ProcKill()
  -> SyncRepCleanupAtProcExit()
  -> LWLockReleaseAll()
  -> WaitLSNCleanup()
  -> ConditionVariableCancelSleep()
  -> 退出 lock group
  -> SwitchBackToLocalLatch()
  -> pgstat_reset_wait_event_storage()
  -> MyProc = NULL
  -> MyProcNumber = INVALID_PROC_NUMBER
  -> DisownLatch()
  -> proc->pid = 0
  -> proc->vxid invalid
  -> 归还 PGPROC 到 freelist
```

这里有一个特别关键的注册顺序：

```text
InitProcess()
  -> on_shmem_exit(ProcKill)

InitProcessPhase2()
  -> ProcArrayAdd(MyProc)
  -> on_shmem_exit(RemoveProcFromArray)
```

callback 后进先出，所以退出时：

```text
RemoveProcFromArray 先执行
ProcKill 后执行
```

这避免了一个危险状态：

```text
PGPROC 已被放回 freelist
但 ProcArray dense arrays 里还保留它的 pgprocno
```

### 5.9 `ProcArrayRemove()`: 从 ProcArray membership 中移除 PGPROC

`ProcArrayRemove()` 用于两类情况：

```text
backend exit:
  ProcArrayRemove(MyProc, InvalidTransactionId)

prepared xact finish:
  ProcArrayRemove(dummyProc, latestXid)
```

它会在 `ProcArrayLock` 和 `XidGenLock` 下：

```text
定位 proc->pgxactoff
必要时推进 latestCompletedXid / xactCompletionCount
清 dense xids / subxidStates / statusFlags
从 pgprocnos[] 中 memmove 删除该 entry
递减 numProcs
修正后续 PGPROC 的 pgxactoff
```

这里 `XidGenLock` 的参与不是偶然。ProcArray membership 和 XID 分配之间有 interlock：系统必须维持“已分配并可能 running 的 XID 能被 snapshot / horizon 看到”这个约束。删除 membership 时也要避免扫描者看到半更新的 dense array。

### 5.10 ugly crash: postmaster 不做单 backend 精细清理

如果 child 正常 `exit(0)` 或 FATAL `exit(1)`，postmaster 认为 child 自己已经运行了 cleanup callback，可以从 active child list 中移除。

如果 exit status 不可信，或者 child attached shared memory 但没有 clean detach，`CleanupBackend()` 会把它当作 crash：

```text
CleanupBackend()
  -> 判断 exitstatus 是否异常
  -> ReleasePostmasterChildSlot()
       如果发现 child 没有 clean detach，也视为 crash
  -> HandleChildCrash()
```

`HandleChildCrash()` 的策略是：

```text
记录日志
  -> 进入 FatalError 状态
  -> 通知其它 server processes quickdie
  -> 等所有 child 退出
  -> reinitialize shared memory
  -> 启动 crash recovery
```

`postmaster.c` 文件头部解释了为什么 postmaster 自己尽量不碰 shared memory：

```text
postmaster 不是 PGPROC array 的成员；
它不能参与 lock-manager operations；
保持 postmaster 远离 shared memory，才能在 backend crash 后可靠地重置系统。
```

所以对“postmaster cleanup”的正确理解是：

```text
正常 FATAL:
  backend 自己通过 proc_exit callback 清 ProcArray / PGPROC，postmaster 只回收本地 child slot。

不可信 crash:
  postmaster 不尝试修复那个 backend 留下的 PGPROC / locks；
  它认为 shared memory 可能损坏，终止其它进程并重建整套 shared memory。
```

这也是 `quickdie()` 为什么直接 `_exit(2)` 而不跑 callback 的原因。SIGQUIT 通常意味着 shared memory 可能已经不可信，继续做“优雅 cleanup”反而可能死锁或破坏更多状态。

## 6. 生命周期 / ownership / cleanup

可以把本主题的 ownership 分成四层：

| 层次 | owner | 创建 | 正常清理 | 异常兜底 |
| --- | --- | --- | --- | --- |
| transaction publication state | 当前事务 | XID 分配 / snapshot 获取 | `ProcArrayEndTransaction()` | `AbortOutOfAnyTransaction()` 触发 abort 后清理 |
| transaction resources | `TopTransactionResourceOwner` | `StartTransaction()` | `ResourceOwnerRelease()` 三阶段 | abort path / cleanup path |
| backend ProcArray membership | backend process | `InitProcessPhase2()` | `RemoveProcFromArray()` | FATAL exit callback；crash 则 shared memory reset |
| PGPROC slot ownership | backend process | `InitProcess()` | `ProcKill()` | FATAL exit callback；crash 则 postmaster reset |

典型生命周期：

```text
backend fork
  -> InitProcess()
       分配 PGPROC，注册 ProcKill
  -> InitProcessPhase2()
       ProcArrayAdd(MyProc)，注册 RemoveProcFromArray
  -> 多次 StartTransaction / CommitTransaction / AbortTransaction
       每个事务结束只清 transaction publication state
  -> proc_exit()
       RemoveProcFromArray
       ProcKill
```

注意：

```text
一个 backend 可以执行很多事务；
一次事务结束不归还 PGPROC；
一次 backend 退出必须先确保没有 active transaction；
Prepared transaction 会把部分 ownership 转移给 dummy PGPROC。
```

### commit ownership

commit 中的 ownership 转移可以这样看：

```text
WAL / pg_xact 持有事务结果
  -> ProcArray 不再持有 running state
  -> lock manager 释放事务锁
  -> ResourceOwner 释放外部资源引用
  -> MemoryContext 清 backend-local 内存
```

commit 之后如果 post-commit cleanup 出错，已经太晚，不能再 abort。源码注释也强调：

```text
ProcArrayEndTransaction() 之后的 cleanup 应该是 noncritical resource releasing。
```

这对扩展和内核改动非常重要：不要把可能决定事务成败的动作塞到 “post-commit cleanup” 之后。

### abort ownership

abort 中的 ownership 目标是回到可继续服务的 backend idle 状态：

```text
取消 wait / buffer lock / LWLock / condition variable
  -> 写 abort 状态
  -> 清 ProcArray transaction publication
  -> release resources and locks
  -> cleanup portals / snapshots / memory
```

abort path 不能假定当前 backend 状态整洁。它必须能处理：

```text
正在等 lock 时 ERROR；
持有 LWLock 时 ERROR；
正在构造 WAL record 时 ERROR；
active snapshot / portal / buffer pin 未按普通路径释放；
子事务嵌套中 ERROR；
parallel worker / leader 关系仍存在。
```

### backend exit ownership

backend exit 的关键不是事务语义，而是 shared memory 身份回收：

```text
before_shmem_exit:
  还可以做高层 cleanup，尤其是 abort active transaction。

on_shmem_exit:
  释放低层 shared state，ProcArrayRemove / ProcKill 属于这一层。

on_proc_exit:
  shared memory 已经离开，做 process-local cleanup。
```

这个分层给了系统一个降级策略：

```text
如果高层 cleanup 已经失败或重新进入，低层 callback 仍有机会执行；
如果进程以不可信方式消失，postmaster 放弃单点 cleanup，转向 crash restart。
```

## 7. 正确性机制层次

### 7.1 ProcArrayLock: running set 与 snapshot 的互斥边界

`src/backend/access/transam/README` 给出的核心规则是：

```text
不允许有 XID 的事务在另一个 backend 正在构造 snapshot 时退出 running set。
```

实现方式：

```text
GetSnapshotData()
  -> ProcArrayLock shared

ProcArrayEndTransaction()
  -> ProcArrayLock exclusive
```

这个规则比理论最小条件更强，但它简单、稳定，并且支撑 `latestCompletedXid + 1` 作为 snapshot `xmax` 的计算。

### 7.2 WAL / pg_xact before ProcArray clear

事务从 running set 中消失后，其它 backend 会去判断它是 committed 还是 aborted。因此清 ProcArray 之前必须先让事务结果可查询：

```text
commit:
  RecordTransactionCommit()
  -> ProcArrayEndTransaction()

abort:
  RecordTransactionAbort()
  -> ProcArrayEndTransaction()

prepared finish:
  RecordTransactionCommitPrepared() / RecordTransactionAbortPrepared()
  -> ProcArrayRemove(dummyProc, latestXid)
```

如果顺序反过来，snapshot 或 tuple visibility 可能看到：

```text
事务不 running
但状态查询还没有可靠答案
```

### 7.3 ProcArray clear before lock release

锁释放会让等待者继续执行。等待者继续执行时应该看到阻塞事务已经从 running set 消失，并且事务状态已经可查询。

因此：

```text
ProcArrayEndTransaction()
  -> ResourceOwnerRelease(LOCKS)
```

而不是：

```text
ResourceOwnerRelease(LOCKS)
  -> ProcArrayEndTransaction()
```

2PC 完成也是同样模式：

```text
ProcArrayRemove(dummyProc, latestXid)
  -> post commit / abort callbacks
  -> release locks
```

### 7.4 ResourceOwner 三阶段释放顺序

事务结束后，`ResourceOwnerRelease()` 分三阶段：

```text
RESOURCE_RELEASE_BEFORE_LOCKS
RESOURCE_RELEASE_LOCKS
RESOURCE_RELEASE_AFTER_LOCKS
```

本节只关注它和 ProcArray 的交界：

```text
ProcArrayEndTransaction()
  -> BEFORE_LOCKS
  -> invalidation / multixact 等
  -> LOCKS
  -> AFTER_LOCKS
```

在 commit 中，catalog invalidation 要在释放 locks 前完成，因为等待 relation lock 的 backend 被唤醒后应该先知道 catalog 变化。

### 7.5 Interrupt holdoff: cleanup 期间不要再次被打断

`CommitTransaction()` 和 `AbortTransaction()` 都会 `HOLD_INTERRUPTS()`。这不是为了性能，而是为了避免 cleanup 过程中再次进入 signal / ERROR 路径，造成半清理状态。

`proc_exit_prepare()` 也会设置：

```text
proc_exit_inprogress = true
InterruptPending = false
ProcDiePending = false
QueryCancelPending = false
InterruptHoldoffCount = 1
CritSectionCount = 0
```

意思是：

```text
现在已经决定退出；
不要再把后续 ereport / interrupt 带回普通执行循环；
cleanup callback 要按退出语义继续推进。
```

### 7.6 Postmaster isolation: crash 后重建，而不是修补

postmaster 的正确性机制是“隔离自己”：

```text
不加入 PGPROC array；
不参与 lock manager；
尽量不触碰 backend shared memory；
child crash 后杀其它 children 并重建 shared memory。
```

这牺牲了局部恢复能力，换取 postmaster 能在 shared memory 已损坏时仍保持可靠。对数据库系统来说，这是一个经典边界：

```text
进程内 cleanup 可以精细；
进程异常死亡后的 shared memory 不能假定仍自洽。
```

## 8. 错误路径 / 异常路径 / fallback

### ERROR during lock wait

当 backend 正在等待 heavyweight lock 时，`MyProc` 可能挂在 wait queue 上。ERROR / cancel / die 进入 abort 前必须先：

```text
LockErrorCleanup()
  -> AbortStrongLockAcquire()
  -> 关闭 deadlock / lock timeout
  -> 如仍在 wait queue，RemoveFromWaitQueue()
  -> 如锁已被授予，GrantAwaitedLock()
  -> ResetAwaitedLock()
```

否则后续再次等待 lock 或释放 lock 时，lock manager 会看到不一致 wait state。

### abort while holding LWLock / buffer lock

abort 早期会：

```text
LWLockReleaseAll()
UnlockBuffers()
ConditionVariableCancelSleep()
```

这类资源不适合等到普通 ResourceOwner 三阶段，因为 abort cleanup 自己可能需要重新进入 shared structures，带着低层锁清理容易死锁。

### FATAL inside backend

`ereport(FATAL)` 最终会调用 `proc_exit(1)`。只要进程仍能执行 callback，清理路径是：

```text
proc_exit()
  -> shmem_exit()
       before_shmem_exit: ShutdownPostgres -> AbortOutOfAnyTransaction()
       on_shmem_exit: RemoveProcFromArray / ProcKill / other shmem cleanup
  -> on_proc_exit
  -> exit(1)
```

因此 FATAL 不等于“跳过 cleanup”。它跳过的是普通命令循环，但仍尽量运行退出 callback。

### SIGTERM / administrator termination

`die()` signal handler 设置：

```text
InterruptPending = true
ProcDiePending = true
```

之后 `ProcessInterrupts()` 在安全点处理，通常抛出 FATAL。进入 FATAL 后仍走 `proc_exit()` cleanup。为了避免打断 cleanup，`die()` 会检查 `proc_exit_inprogress`。

### SIGQUIT / quickdie

`quickdie()` 是不同路径。它的原则是：

```text
不要运行 proc_exit() 或 atexit() callback；
shared memory 可能已经损坏；
直接 _exit(2)，让 postmaster 进入 crash reset。
```

这就是 PostgreSQL 区分“可清理失败”和“不可信崩溃”的边界。

### postmaster child cleanup failure

`CleanupBackend()` 中即使 exit status 看似正常，也会检查 child 是否 clean detach。如果 `ReleasePostmasterChildSlot()` 发现 child 没有正确清理，仍会当 crash 处理：

```text
child failed to clean itself up
  -> crashed = true
  -> HandleChildCrash()
```

这避免 postmaster 在旧 shared memory 上继续运行，而那里可能还留着 stale PGPROC / semaphore / lock state。

### prepared transaction finish failure

`FinishPreparedTransaction()` 在记录 commit / abort 并 `ProcArrayRemove()` 之后，会把 `gxact->valid = false`，防止失败后其它 backend 再尝试完成同一个 prepared transaction。prepared xact 是跨 backend ownership，必须显式管理“谁还能完成它”。

### extension / callback failure

`shmem_exit()` 和 `proc_exit_prepare()` 在调用 callback 时会先递减 callback index，再执行 callback。这样如果 callback 中 `ereport(ERROR)` 或 `ereport(FATAL)`，重新进入 cleanup 时不会无限重复调用同一个 callback。

这说明 exit cleanup 的目标不是“所有 callback 都绝对成功”，而是：

```text
尽最大努力推进剩余 cleanup；
避免 re-entry 造成无限循环；
如果不能信任 shared memory，则交给 crash restart。
```

## 9. 成本、资源与跨模块传播

### ProcArrayLock 是 transaction end 的高频共享锁

每个有 XID 的事务结束都要从 running set 中消失。高并发 commit workload 下，这会集中到：

```text
ProcArrayLock exclusive
ProcArrayEndTransactionInternal()
latestCompletedXid update
xactCompletionCount update
```

`ProcArrayGroupClearXid()` 通过 group clear 降低 lock handoff 成本，但没有改变正确性边界：

```text
仍然必须在 exclusive ProcArrayLock 下批量清 XID。
```

可观测现象：

```text
大量小事务并发提交时，可能看到 WAIT_EVENT_PROCARRAY_GROUP_UPDATE；
这通常说明许多 backend 正在等待 group leader 统一清 XID。
```

### ProcArrayRemove 会移动 dense arrays

backend exit 不是 hot path，但 `ProcArrayRemove()` 会 `memmove` dense arrays 并修正后续 `pgxactoff`：

```text
删除一个 ProcArray member
  -> 移动 pgprocnos[]
  -> 移动 xids[]
  -> 移动 subxidStates[]
  -> 移动 statusFlags[]
  -> 修正后续 allProcs[procno].pgxactoff
```

这解释了为什么 `pgxactoff` 不能被无锁长期缓存。它是当前 dense array 的偏移，不是稳定 ID。

### ResourceOwner 与 ProcArray 边界

ResourceOwner 释放的是“当前 backend 持有的资源引用”。ProcArray 释放的是“其它 backend 对当前事务状态的观察入口”。

顺序必须是：

```text
ProcArrayEndTransaction()
  -> ResourceOwnerRelease()
```

如果反过来，资源释放可能唤醒其它 backend，而它们看到的事务 running state 仍是旧的。

### invalidation 与 lock waiters

commit 中：

```text
AtEOXact_RelationCache(true)
AtEOXact_TypeCache()
AtEOXact_Inval(true)
ResourceOwnerRelease(LOCKS)
```

这保证等待某个 relation lock 的 backend 被唤醒后，已经能收到必要的 invalidation。这里的 correctness 不只在 ProcArray，也跨到了 relcache / catcache / sinval。

### replication / logical worker exit

replication workers、walsender、logical workers 都可能注册自己的 `before_shmem_exit` 或 `on_shmem_exit` callback。它们遵守同一分层：

```text
需要事务或 catalog 的清理放 before_shmem_exit；
释放 shared slot / worker state 放 on_shmem_exit；
进程本地连接关闭放 on_proc_exit 或普通退出路径。
```

本课不展开 replication slot 清理细节，但诊断 FATAL exit 后的 stale slot / stale xmin 时，要同时检查：

```text
backend 是否还活着；
slot 是否仍 active；
ProcArray 中是否仍有相关 xmin；
postmaster 是否进入 crash recovery 而非普通 FATAL cleanup。
```

### background worker 与 auxiliary process

不是所有进程都会进入 ProcArray。

```text
普通 backend / full backend worker:
  InitProcess()
  InitProcessPhase2()
  有 PGPROC，也在 ProcArray。

auxiliary process:
  InitAuxiliaryProcess()
  有 PGPROC 用于 LWLock 等 shared memory wait；
  通常不在 ProcArray，不运行普通事务。
```

因此 auxiliary process 的 `AuxiliaryProcKill()` 是 `ProcKill()` 的简化版：

```text
释放 LWLock
取消 condition variable sleep
切回 local latch
清 MyProc / MyProcNumber
pid = 0
不把 PGPROC 放回普通 freelist
```

理解这个差异能避免把所有 `PGPROC` 都误认为 ProcArray member。

## 10. 观测与诊断入口

### SQL 观察 backend 是否仍发布事务状态

常用入口：

```sql
select pid, backend_xid, backend_xmin, state, wait_event_type, wait_event, query
from pg_stat_activity
where backend_type = 'client backend';
```

判断方向：

```text
backend_xid 不为空
  -> 该 backend 当前发布了 top-level XID

backend_xmin 不为空
  -> 该 backend 仍持有 snapshot 或类似 xmin 需求，可能钉住 VACUUM horizon

state = 'idle in transaction'
  -> 事务还没结束，ProcArrayEndTransaction() 未发生

pid 消失
  -> backend 已退出；如果是正常/FATAL exit，应已执行 ProcArrayRemove / ProcKill
```

`pg_stat_activity` 是采样视图，不是锁保护下的完整 ProcArray dump。它适合观测症状，不适合证明没有竞态。

### SQL 观察锁是否仍由事务持有

```sql
select pid, locktype, mode, granted, relation::regclass, transactionid, virtualtransaction
from pg_locks
order by pid nulls last, granted, locktype;
```

诊断问题：

```text
事务已经从 pg_stat_activity 消失，但锁仍在？
  -> 检查是否是 prepared transaction。

pid 为空或 virtualtransaction 特殊？
  -> 可能来自 prepared transaction 或非普通 backend。
```

prepared transaction：

```sql
select gid, prepared, owner, database, transaction
from pg_prepared_xacts;
```

2PC 的关键观测规律：

```text
原 backend 可以退出；
prepared transaction 的锁和 ProcArray 状态仍可能存在；
需要 COMMIT PREPARED 或 ROLLBACK PREPARED 才真正释放。
```

### 观察 exit / crash 日志

正常管理员终止 backend 常见路径：

```text
SIGTERM
  -> ProcDiePending
  -> FATAL
  -> proc_exit(1)
  -> child exit status 1
  -> postmaster 不做 crash restart
```

不可信 crash 常见日志方向：

```text
server process was terminated by signal ...
terminating any other active server processes
all server processes terminated; reinitializing
database system was interrupted; last known up at ...
```

如果看到 reinitializing，说明这不是“某个 backend 的 ProcKill 没跑完”那么简单，而是 postmaster 已经认为 shared memory 不能继续信任。

### gdb 断点练习

可以在测试实例上设置断点：

```text
break ProcArrayEndTransaction
break ProcArrayEndTransactionInternal
break ProcArrayGroupClearXid
break ProcArrayRemove
break ProcKill
break ShutdownPostgres
break AbortOutOfAnyTransaction
```

观察顺序：

```text
COMMIT;
  -> ProcArrayEndTransaction
  -> ResourceOwnerRelease locks

select pg_terminate_backend(pid);
  -> ProcessInterrupts
  -> FATAL
  -> proc_exit
  -> ShutdownPostgres
  -> RemoveProcFromArray
  -> ProcKill

PREPARE TRANSACTION;
  -> PostPrepare_Locks
  -> ProcArrayClearTransaction

COMMIT PREPARED;
  -> ProcArrayRemove(dummy proc, latestXid)
```

gdb 中可以重点打印：

```text
MyProc
MyProc->xid
MyProc->xmin
MyProc->subxidStatus.count
MyProc->statusFlags
MyProc->pgxactoff
procArray->numProcs
TransamVariables->latestCompletedXid
TransamVariables->xactCompletionCount
```

注意 `pgxactoff` 只在持锁上下文中稳定。不要在任意断点长期保存它再解释 dense arrays。

### wait event

高并发 commit 下可能看到：

```text
wait_event = 'ProcArrayGroupUpdate'
```

它对应 `WAIT_EVENT_PROCARRAY_GROUP_UPDATE`，表示 follower 正在等 group clear leader 清理自己的 XID。

这不是“事务没提交成功”，而是：

```text
commit path 正在等待 ProcArray running set 清理完成。
```

## 11. 常见误区

### 误区 1：commit 后释放锁就是事务结束点

更准确的顺序是：

```text
RecordTransactionCommit()
  -> ProcArrayEndTransaction()
  -> release locks
```

锁释放很重要，但 snapshot correctness 的 running set 边界在 `ProcArrayEndTransaction()`。

### 误区 2：`ProcKill()` 会清当前事务

`ProcKill()` 负责释放 PGPROC 身份。正常路径下 active transaction 应该已经由 `ShutdownPostgres()` / `AbortOutOfAnyTransaction()` 或普通事务结束路径清掉。把事务语义塞给 `ProcKill()` 会太晚，也太低层。

### 误区 3：backend 退出就一定没有残留事务影响

prepared transaction 是反例：

```text
原 backend 退出
  -> prepared xact 的 dummy PGPROC 仍可持有 locks / ProcArray 状态
```

需要检查 `pg_prepared_xacts`，不能只看 `pg_stat_activity`。

### 误区 4：postmaster 会帮崩溃 backend 清理 PGPROC

postmaster 的原则恰好相反：

```text
不可信 crash 后不在旧 shared memory 中做精细修复；
终止其它进程并重建 shared memory。
```

正常 FATAL exit 的清理是在 child 自己的 `proc_exit()` callback 中完成的。

### 误区 5：`pgxactoff` 是稳定身份

`pgxactoff` 是 dense arrays 的当前偏移。`ProcArrayAdd()` / `ProcArrayRemove()` 会改变它。稳定身份是 `PGPROC` 槽位和 proc number，dense array 偏移是扫描优化。

### 误区 6：read-only transaction end 完全不用清理 ProcArray 相关状态

无 XID 事务结束可以不互斥 snapshot-taking，因为它不改变 running XID set。但它仍可能需要清：

```text
vxid.lxid
xmin
delayChkptFlags
vacuum state flags
```

尤其 `xmin` 和 vacuum flags 会影响 cleanup horizon。

### 误区 7：`AtEOXact_Snapshot()` 总是负责清 `MyProc->xmin`

普通 commit 路径中，`ProcArrayEndTransaction()` 已经先清 `MyProc->xmin`。`snapmgr.c` 中也说明，在正常 commit 处理中不需要 `AtEOXact_Snapshot()` 再处理这个字段。不同路径下是否传 `resetXmin` 取决于前面是否已经通过 ProcArray transaction end 清掉。

## 12. 课堂实验

### 实验 1：观察 idle in transaction 如何保留 `backend_xmin`

会话 A：

```sql
begin;
select * from pg_class limit 1;
select pg_backend_pid();
```

会话 B：

```sql
select pid, backend_xid, backend_xmin, state, query
from pg_stat_activity
where pid = <会话A pid>;
```

然后在会话 A：

```sql
commit;
```

再次在会话 B 查询。

观察目标：

```text
事务结束前，backend 可能发布 backend_xmin；
commit 后，ProcArrayEndTransaction() 清理 xmin / xid；
pg_stat_activity 中对应字段消失或变化。
```

讨论：

```text
如果会话 A 是纯读事务，backend_xid 可能为空；
但 backend_xmin 仍可能存在，因为 snapshot 仍会钉住 cleanup horizon。
```

### 实验 2：终止 backend 与 FATAL cleanup

会话 A：

```sql
begin;
select pg_backend_pid();
select pg_sleep(300);
```

会话 B：

```sql
select pg_terminate_backend(<会话A pid>);
```

观察：

```sql
select pid, state, backend_xid, backend_xmin
from pg_stat_activity
where pid = <会话A pid>;

select *
from pg_locks
where pid = <会话A pid>;
```

预期：

```text
backend 被 FATAL 终止；
child 运行 proc_exit cleanup；
active transaction 被 abort；
ProcArray entry 和 PGPROC ownership 被清理；
pg_stat_activity / pg_locks 不再显示该 pid 的普通事务状态。
```

如果日志中出现 crash restart 相关信息，说明走的是不可信 crash path，不是普通 `pg_terminate_backend()` 清理。

### 实验 3：prepared transaction 不是 backend 残留

准备实例需要开启 `max_prepared_transactions`。

会话 A：

```sql
begin;
create table if not exists t_2pc_demo(id int primary key);
insert into t_2pc_demo values (1);
prepare transaction 'demo_gid';
```

会话 B：

```sql
select * from pg_prepared_xacts where gid = 'demo_gid';

select locktype, mode, granted, relation::regclass, transactionid, virtualtransaction, pid
from pg_locks
where transactionid = (
  select transaction from pg_prepared_xacts where gid = 'demo_gid'
);
```

然后：

```sql
rollback prepared 'demo_gid';
```

观察目标：

```text
原会话不再需要保持连接；
prepared transaction 仍能持有事务状态和锁；
真正完成时才 ProcArrayRemove(dummy proc, latestXid) 并释放锁。
```

### 实验 4：高并发小事务与 group XID clear

用 `pgbench` 运行大量短事务：

```bash
pgbench -c 64 -j 8 -T 60 -S
```

同时观察：

```sql
select wait_event_type, wait_event, count(*)
from pg_stat_activity
where wait_event is not null
group by wait_event_type, wait_event
order by count(*) desc;
```

如果 workload 和硬件足够触发 commit-side contention，可能看到：

```text
ProcArrayGroupUpdate
```

解释：

```text
多个提交者正在通过 ProcArrayGroupClearXid() 合并 ProcArrayLock exclusive 清理。
```

### 实验 5：gdb 验证 callback 顺序

测试环境中：

```text
break ShutdownPostgres
break AbortOutOfAnyTransaction
break RemoveProcFromArray
break ProcKill
```

对一个正在事务中的 backend 发起终止：

```sql
select pg_terminate_backend(<pid>);
```

观察调用顺序应符合：

```text
ShutdownPostgres
  -> AbortOutOfAnyTransaction
  -> RemoveProcFromArray
  -> ProcKill
```

如果直接制造 SIGQUIT crash，预期不应该看到普通 backend 走完整 `proc_exit()` callback，而是 postmaster 进入 crash reset 流程。

## 13. 讨论题

1. 为什么 commit path 中 `ProcArrayEndTransaction()` 必须在 `ResourceOwnerRelease(LOCKS)` 之前，而不是之后？

2. 如果一个 backend 清掉了 `ProcGlobal->xids[]`，但还没有推进 `latestCompletedXid`，`GetSnapshotData()` 可能看到什么不一致状态？

3. 为什么 `ProcArrayGroupClearXid()` 可以改善高并发 commit 性能，却不能降低 ProcArrayLock 的正确性要求？

4. 为什么 prepared transaction 要把 locks 转移到 dummy `PGPROC`，而不是让原 backend 持续存在？

5. 为什么 postmaster 不加入 PGPROC array？如果 postmaster 参与 lock manager，会如何影响 crash recovery 可信边界？

6. 对一个 VACUUM 被旧 xmin 卡住的问题，如何区分普通 idle transaction、prepared transaction、replication slot 和 backend stale state？

7. 如果扩展需要在 backend 退出时清理 shared memory 状态，应选择 `before_shmem_exit`、`on_shmem_exit` 还是 `on_proc_exit`？判断标准是什么？

8. 为什么 `quickdie()` 不运行 `proc_exit()` callback？这和 shared memory corruption 假设有什么关系？

## 14. 本节小结

本节把 PGPROC / ProcArray 主线补完整：

```text
InitProcess()
  -> backend 获得 shared memory 身份

InitProcessPhase2()
  -> backend 进入 ProcArray，被其它 backend 扫描

GetNewTransactionId()
  -> XID 发布到 running set

GetSnapshotData() / ComputeXidHorizons()
  -> 其它 backend 消费这些状态

ProcArrayEndTransaction()
  -> 当前事务从 running set 中消失

ResourceOwnerRelease()
  -> 释放事务资源和 locks

RemoveProcFromArray()
  -> backend 退出 ProcArray membership

ProcKill()
  -> backend 归还 PGPROC 槽位

postmaster crash reset
  -> 不可信死亡时重建 shared memory，而不是修补单个 PGPROC
```

最重要的可迁移规律是：

```text
shared-state cleanup 的核心不是“最后都释放掉”；
而是先撤掉会被并发观察者用来做 correctness 判断的 publication state，
再释放会唤醒或影响其它 backend 的资源，
最后回收本进程身份和 backend-local 状态。
```

在 PostgreSQL 中，这个规律具体表现为：

```text
WAL / pg_xact result before ProcArray clear;
ProcArray clear before lock release;
transaction cleanup before PGPROC reuse;
child-owned cleanup before postmaster child slot cleanup;
untrusted crash leads to shared memory reset, not local repair.
```

到这里，`PGPROC / ProcArray 与 backend 状态` 这组课程形成了完整闭环：

```text
获得身份
  -> 进入 ProcArray
  -> 发布事务状态
  -> 被 snapshot / horizon 扫描
  -> 事务结束时撤销 publication
  -> backend 退出时回收身份
```

下一组基础设施可以进入同步原语：`SpinLock / LWLock / Latch / Condition Variable / Barrier`。在那里会继续看到同一个系统规律，只是关注点从“状态是否可见”切换到“等待、唤醒和互斥边界如何组合”。
