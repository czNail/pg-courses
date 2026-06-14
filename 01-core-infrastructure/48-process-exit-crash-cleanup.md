# PostgreSQL process exit crash cleanup
## 课程定位
前置知识：已经理解 `MemoryContext`、`ResourceOwner`、`PGPROC`、`ProcArray`、latch、postmaster 监督模型，以及 backend 通过 `ERROR` / `FATAL` 离开当前执行路径。
本节唯一主问题：
```text
普通退出、FATAL、postmaster death 和 crash restart 分别由谁清理 backend-local 与 shared state？
```
核心矛盾：
```text
可信退出时，backend 自己最了解本地状态和它发布到 shared memory 的状态，应该由它精细清理。
不可信退出时，backend 的栈、锁和 shared memory 写入都可能已经损坏，系统不能再相信 per-backend cleanup。
```
本节结论先放在开头：
| 路径 | backend-local state | shared state | postmaster 做什么 |
| --- | --- | --- | --- |
| 普通退出 | backend 走 `proc_exit()` callback，OS 最终回收进程内存和 fd。 | backend 自己通过 `before_shmem_exit` / `on_shmem_exit` 撤销。 | wait child，释放 `PMChild` slot，确认 child clean detach。 |
| `FATAL` | `ereport(FATAL)` 进入 `proc_exit(1)`，仍运行 callback。 | 仍由 backend 清理；exit code 1 不算 crash。 | 当作非 crash child exit。 |
| postmaster death | child 检测 supervisor 消失后停止服务并退出。 | cleanup 只是 best effort；没有 postmaster 继续维持实例。 | postmaster 已经不存在；下一次启动重建 shared memory 并 recovery。 |
| crash restart | 崩溃进程的本地资源由 OS 回收；其它 child `quickdie()` 后 `_exit(2)`。 | 不修补旧 shared memory；整体重建。 | kill remaining children，清旧 IPC，重建 shared memory，启动 startup process。 |
学完后应能判断：
```text
为什么 FATAL 不是 crash restart。
为什么 quickdie() 故意不运行 proc_exit callbacks。
为什么 postmaster 不帮崩溃 backend 调用 ProcKill()。
为什么 PMChildFlags 是 child clean detach 的死手开关。
为什么 RemoveProcFromArray() 必须早于 ProcKill()。
为什么 postmaster death detection 是 fail-stop，不是 cleanup。
为什么 crash restart 同时涉及 shared memory、WAL recovery 和文件系统扫描。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面的基础设施课分别回答了三个局部问题。
`MemoryContext` 回答：
```text
backend-local 内存树什么时候 reset / delete？
```
`ResourceOwner` 回答：
```text
buffer pin、lock、snapshot、文件句柄等外部资源如何在 ERROR / abort 中兜底释放？
```
`PGPROC` / `ProcArray` 回答：
```text
backend 如何把事务 running state 发布给其它 backend，又如何在事务结束时撤销？
```
本节把这些边界放到 process exit 和 crash restart 中。
问题不再是单个事务结束，而是一个 OS process 退出时，哪些状态由它自己撤销，哪些由 postmaster 监督，哪些在 crash 后不允许精细修复。
这个问题的关键不在“有没有资源要释放”。
关键在：
```text
cleanup 的前提是 cleanup 代码和被清理状态仍然可信。
```
普通退出和 `FATAL` 满足这个前提。
`SIGQUIT` / crash path 不满足这个前提。
postmaster death path 甚至失去了 supervisor。
因此本节主线是：
```text
可信 backend 自清理；
不可信 backend 不清理；
postmaster 只监督和重建，不在旧 shared memory 中做手术。
```
这也是它和第 19 课的区别。
第 19 课聚焦 `PGPROC` / `ProcArray` stale state。
本节把视角提升到 process、postmaster、shared memory segment、WAL recovery 和文件系统状态的分工。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
普通退出和 FATAL 由 backend 调用 proc_exit()，先运行 shmem_exit() 的 before_shmem_exit、DSM cleanup、on_shmem_exit，再运行 on_proc_exit；如果 child 异常死亡或未 clean detach，postmaster 把它视为 crash，通知其它 child quickdie，等它们全部退出后释放旧 IPC、重建 shared memory，并让 startup process 通过 WAL recovery 恢复持久状态。
```
这里有两个不能同时完全满足的目标。
目标一：
```text
尽量精细地释放资源，避免 stale PGPROC、ProcArray entry、锁、replication slot、DSM mapping、temp file 等状态影响其它 backend。
```
目标二：
```text
一旦 shared memory 可能损坏，就不要相信任何依赖 shared memory 的精细 cleanup。
```
PostgreSQL 的解法是按退出类型切换策略。
普通退出：
```text
backend 活着
  -> C 栈可执行
  -> shared memory 可信
  -> backend 按 callback 顺序清理
```
`FATAL`：
```text
backend 不能继续服务当前 session
  -> 不回 command loop
  -> 仍调用 proc_exit(1)
  -> cleanup 与普通退出基本相同
```
`SIGQUIT` / crash：
```text
shared memory 可能不可信
  -> 不跑 proc_exit callback
  -> 直接 _exit(2)
  -> postmaster 触发全局 crash restart
```
postmaster death：
```text
supervisor 不存在
  -> child 不能继续作为正常 server process
  -> wait path 检测后退出
  -> 下一次 postmaster 启动重新建立实例级状态
```
所以本节最重要的 mental model 是：
```text
不是所有 cleanup 都是释放旧对象。
有时正确 cleanup 是停止相信旧对象，并整体重建。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/ipc/ipc.c` | `proc_exit()`、`proc_exit_prepare()`、`shmem_exit()`、`before_shmem_exit()`、`on_shmem_exit()`、`on_proc_exit()` 的分层和 LIFO 顺序。 |
| 2 | `src/include/storage/ipc.h` | exit callback API、`PG_ENSURE_ERROR_CLEANUP`、`proc_exit_inprogress`、`shmem_exit_inprogress`。 |
| 3 | `src/backend/storage/lmgr/proc.c` | `InitProcess()`、`InitProcessPhase2()` 注册 `ProcKill` 和 `RemoveProcFromArray`；`ProcKill()` 归还 `PGPROC`。 |
| 4 | `src/backend/storage/ipc/procarray.c` | `ProcArrayEndTransaction()`、`ProcArrayRemove()` 清理 running XID、xmin、SubXID、status flags 和 dense arrays。 |
| 5 | `src/backend/utils/init/postinit.c` | `ShutdownPostgres()` 作为 `before_shmem_exit` callback，兜底 `AbortOutOfAnyTransaction()`。 |
| 6 | `src/backend/tcop/postgres.c` | `die()`、`ProcessInterrupts()`、`quickdie()` 区分 `SIGTERM` 的 `FATAL` cleanup 与 `SIGQUIT` 的 `_exit(2)`。 |
| 7 | `src/backend/postmaster/postmaster.c` | `CleanupBackend()`、`HandleChildCrash()`、`ResetShmemAllocator()`、`CreateSharedMemoryAndSemaphores()` 的 crash restart 路径。 |
| 8 | `src/backend/postmaster/pmchild.c` | `ReleasePostmasterChildSlot()` 如何把 child clean detach 结果反馈给 postmaster。 |
| 9 | `src/backend/storage/ipc/pmsignal.c` | `RegisterPostmasterChildActive()`、`MarkPostmasterChildInactive()`、`PMChildFlags` 死手开关。 |
| 10 | `src/include/storage/pmsignal.h` | `PostmasterIsAlive()` fast path / slow path。 |
| 11 | `src/backend/storage/ipc/latch.c`、`waiteventset.c` | `WL_POSTMASTER_DEATH` 与 `WL_EXIT_ON_PM_DEATH`。 |
| 12 | `src/backend/storage/file/fd.c`、`reinit.c` | backend exit 的 temp cleanup、crash restart 的 temp cleanup、unlogged relation reset。 |
| 13 | `src/backend/storage/ipc/ipci.c`、`shmem.c` | shared memory request / init / reset。 |
| 14 | `src/backend/access/transam/xlog.c` | crash recovery 中 `ResetUnloggedRelations()`、exported snapshot 删除、WAL recovery。 |
推荐阅读顺序：
```text
ipc.c
  -> proc.c
  -> postinit.c
  -> postgres.c
  -> pmsignal.c / pmchild.c
  -> postmaster.c
  -> shmem.c / ipci.c / xlog.c / fd.c
```
不要从 `postmaster.c` 开始把 postmaster 理解成总清洁工。
正常退出时，postmaster 主要做监督和 `PMChild` bookkeeping。
真正清理 `PGPROC`、`ProcArray`、DSM、replication slot、temp file 的，是 child 自己的 exit callback。
## 4. 关键状态与责任边界
### 4.1 backend-local state
backend-local state 只存在于当前进程地址空间。
典型状态：
| 状态 | owner | 正常 cleanup | crash 后 |
| --- | --- | --- | --- |
| `MemoryContext` tree | backend process | 事务 abort、portal cleanup、context reset/delete。 | OS 回收地址空间。 |
| `ResourceOwner` tree | backend process | `AbortCurrentTransaction()`、`AbortOutOfAnyTransaction()`、`ResourceOwnerRelease()`。 | 本地内存由 OS 回收，但 shared resources 可能 stale。 |
| `CurrentMemoryContext` / `CurrentResourceOwner` / `ActivePortal` | backend process | ERROR/FATAL 路径恢复或清理。 | 进程消失，无恢复意义。 |
| local catcache / relcache / plancache | backend process | transaction end、invalidation、process exit。 | OS 回收。 |
| VFD table、本地 fd、socket | backend process | `fd.c`、`pqcomm.c` callback close；OS 兜底关闭 fd。 | OS 关闭 fd，但 PostgreSQL 逻辑 cleanup 可能缺失。 |
backend-local memory 不需要 postmaster 释放。
但 backend-local 指针经常是清理 shared state 的入口。
例如 `MyProc`、`CurrentResourceOwner`、`MyReplicationSlot`、DSM mapping list。
这就是普通退出必须趁 process 还活着运行 PostgreSQL cleanup 的原因。
### 4.2 shared state
shared state 会被其它 backend 观察。
典型状态：
| shared state | 正常 owner | 清理入口 | stale 后果 |
| --- | --- | --- | --- |
| `PGPROC` slot | 当前 backend | `ProcKill()` | slot 复用、latch、semaphore、lock wait identity 出错。 |
| ProcArray membership | 当前 backend | `RemoveProcFromArray()` | snapshot、xmin horizon、VACUUM、lock wait 看到 dead process。 |
| running XID / SubXID / xmin | 当前事务 | `ProcArrayEndTransaction()` | MVCC visibility 和 cleanup horizon 错误。 |
| `PMChildFlags` | child + postmaster | `MarkPostmasterChildInactive()` | postmaster 无法确认 clean detach。 |
| ProcSignal slot | 当前 backend | `CleanupProcSignalState()` | barrier、conflict、cancel 信号可能指向旧 pid。 |
| sinval state | 当前 backend | `CleanupInvalidationState()` | invalidation queue 保留 dead reader。 |
| shared buffer pins / locks | ResourceOwner / lock manager | transaction abort 和 ResourceOwner cleanup。 | waiters 或 recovery 被 dead holder 卡住。 |
| replication slot active state | 当前 backend | `ReplicationSlotShmemExit()` | active pid、xmin、restart_lsn 保持占用。 |
| DSM mappings | 当前 backend | `dsm_backend_shutdown()` | parallel / logical / DSA 状态无法 detach。 |
正常退出时，shared state cleanup 是 backend 自己的责任。
postmaster 只确认 child 已经把 `PMChildFlags` 退回可回收状态。
如果没有，postmaster 不会逐项修 shared memory。
它会把整件事升级为 crash。
### 4.3 postmaster-local state
postmaster 也有自己的本地状态。
这些状态不在 shared memory 中。
| 状态 | 作用 |
| --- | --- |
| `PMChild` | postmaster 对 child process 的记录。 |
| `ActiveChildList` | wait child 后查找对应 child。 |
| `pmState` | postmaster state machine。 |
| `FatalError` | 是否已经进入 crash restart cycle。 |
| background worker restart state | 控制 crash recovery 后 worker 是否可重启。 |
普通退出需要两边动作：
```text
backend:
  清理自己拥有的 shared state。
postmaster:
  wait child，释放 PMChild entry，检查 child 是否 clean detach。
```
### 4.4 文件系统状态
文件系统状态跨 process 生命周期。
典型状态：
| 文件系统状态 | 正常 cleanup | crash cleanup |
| --- | --- | --- |
| transaction-local temp files | `AtEOXact_Files()`、ResourceOwner。 | `RemovePgTempFiles()` 扫描 `pgsql_tmp`。 |
| interXact temp files | `BeforeShmemExit_Files()`。 | `RemovePgTempFiles()`。 |
| temporary relation files | namespace/storage cleanup。 | startup / crash cleanup 扫描移除。 |
| unlogged relation main fork | 正常运行不 WAL-log 数据。 | recovery 中按 init fork reset。 |
| exported snapshot files | transaction / snapshot cleanup。 | recovery 中 `DeleteAllExportedSnapshotFiles()`。 |
这说明 crash cleanup 不是单一机制。
shared memory 靠重建。
持久状态靠 WAL recovery。
临时和 unlogged 文件靠文件系统规则。
## 5. 四类退出路径对照
### 5.1 普通退出
普通退出典型入口：
```text
客户端正常断开
后台进程完成任务后 proc_exit(0)
postmaster 要求优雅关闭时 child 在安全点退出
startup/auth 早期还没触 shared memory 时直接退出
```
主链路：
```text
PostgresMain()
  -> proc_exit(0)
     -> proc_exit_prepare(0)
        -> proc_exit_inprogress = true
        -> 清 pending interrupt flags
        -> 清 error_context_stack / debug_query_string
        -> shmem_exit(0)
           -> LWLockReleaseAll()
           -> before_shmem_exit callbacks
           -> dsm_backend_shutdown()
           -> on_shmem_exit callbacks
        -> on_proc_exit callbacks
     -> exit(0)
postmaster wait()
  -> CleanupBackend()
  -> ReleasePostmasterChildSlot()
```
`before_shmem_exit` 适合仍需要系统大部分能力的 cleanup。
例如事务 abort、temp relation、replication slot、pgstat、logical worker 状态。
`on_shmem_exit` 适合低层 shared memory identity teardown。
例如 ProcArray membership、`PGPROC`、ProcSignal、sinval、backend status。
`on_proc_exit` 更靠后。
它适合不依赖 shared memory 的 process-local 收尾，例如 socket close、disconnect log、smgr shutdown。
### 5.2 FATAL
`FATAL` 表示当前 process 不能继续服务。
它不表示 shared memory 已经损坏。
典型链路：
```text
die(SIGTERM)
  -> ProcDiePending = true
  -> SetLatch(MyLatch)
CHECK_FOR_INTERRUPTS()
  -> ProcessInterrupts()
     -> LockErrorCleanup()
     -> ereport(FATAL, ...)
        -> proc_exit(1)
```
`postmaster.c` 的 `CleanupBackend()` 把 exit status 0 和 1 都当作非 crash。
但还有一个条件：
```text
ReleasePostmasterChildSlot() 必须确认 child 已 clean detach。
```
所以：
```text
FATAL + clean detach:
  单 backend 退出。
FATAL-looking exit + 未 clean detach:
  postmaster 仍会 treat as crash。
```
### 5.3 postmaster death
postmaster death 是 supervisor 消失。
child 不能继续提供正常服务。
Unix-like 平台的基础机制：
```text
postmaster 创建 pipe
postmaster 持有 write end
child 持有 read end
postmaster 死亡后 read end 看到 EOF/HUP
```
wait API 把这个机制接入：
```text
WL_EXIT_ON_PM_DEATH:
  检测到 postmaster death 时直接 proc_exit(1)。
WL_POSTMASTER_DEATH:
  把事件返回给调用者，由调用者决定如何退出或降级。
```
`WaitLatch()` 断言 postmaster-managed caller 必须处理 postmaster death。
显式检查路径是：
```text
PostmasterIsAlive()
  -> fast path 看 postmaster_possibly_dead
  -> slow path 读 postmaster_alive_fds[WATCH]
```
重要边界：
```text
postmaster death detection 是 fail-stop 机制，不是 shared state cleanup 机制。
```
child 可能通过 `proc_exit(1)` 尽力清理。
但没有 postmaster 继续监督，也没有 postmaster 在本轮中重启 startup process。
下一次实例启动会重新建立 shared memory 并 recovery。
### 5.4 crash restart
crash restart 的入口有两类。
第一类是不可信 exit：
```text
SIGSEGV
SIGABRT
exit code 2
手工 SIGQUIT 某个 backend
其它非 0/1 exit status
```
第二类是未 clean detach：
```text
ReleasePostmasterChildSlot()
  -> MarkPostmasterChildSlotUnassigned()
  -> 返回 false
  -> child failed to clean itself up
```
child 开始使用 shared memory 前调用：
```text
RegisterPostmasterChildActive()
  -> PMChildFlags[slot] = PM_CHILD_ACTIVE
  -> on_shmem_exit(MarkPostmasterChildInactive)
```
正常 shmem exit 末尾：
```text
MarkPostmasterChildInactive()
  -> PMChildFlags[slot] = PM_CHILD_ASSIGNED
```
postmaster 回收 child slot 时只有看到 `ASSIGNED` 才相信 cleanup 完成。
如果 slot 仍是 `ACTIVE` / `WALSENDER`，postmaster 不知道 shared state 清到了哪一步。
它只能进入 crash restart。
## 6. 主流程源码 walkthrough
### 6.1 初始化时决定退出顺序
退出顺序在 backend 初始化时通过 callback 注册顺序决定。
核心链路：
```text
InitProcess()
  -> RegisterPostmasterChildActive()
     -> on_shmem_exit(MarkPostmasterChildInactive)
  -> 从 ProcGlobal freelist 取 PGPROC
  -> 初始化 MyProc 字段、latch、semaphore、wait event storage
  -> on_shmem_exit(ProcKill)
InitProcessPhase2()
  -> ProcArrayAdd(MyProc)
  -> on_shmem_exit(RemoveProcFromArray)
```
`ipc.c` 中 callback 是 LIFO。
因此实际退出顺序是：
```text
RemoveProcFromArray()
  -> ProcArrayRemove(MyProc, InvalidTransactionId)
ProcKill()
  -> SyncRepCleanupAtProcExit()
  -> LWLockReleaseAll()
  -> WaitLSNCleanup()
  -> ConditionVariableCancelSleep()
  -> SwitchBackToLocalLatch()
  -> DisownLatch(&MyProc->procLatch)
  -> detach lock group
  -> MyProc = NULL
  -> proc->pid = 0
  -> PGPROC slot 回到 freelist
MarkPostmasterChildInactive()
  -> PMChildFlags[slot] = PM_CHILD_ASSIGNED
```
`RemoveProcFromArray()` 必须早于 `ProcKill()`。
因为 `ProcArrayRemove()` 仍需要有效 `MyProc` 和 `pgxactoff`。
`ProcKill()` 必须早于 `MarkPostmasterChildInactive()`。
因为 postmaster 只能在低层 shared cleanup 完成后，才相信 child clean detach。
### 6.2 `proc_exit()` 做什么
`proc_exit(code)` 先检查当前进程确实是 PostgreSQL backend 自己。
然后进入：
```text
proc_exit_prepare(code)
```
`proc_exit_prepare()` 做四件关键事。
第一，承诺退出：
```text
proc_exit_inprogress = true
```
这让后续 `ereport()` 不再回到 command loop。
第二，清理 pending interrupt：
```text
InterruptPending = false
ProcDiePending = false
QueryCancelPending = false
InterruptHoldoffCount = 1
CritSectionCount = 0
```
退出过程中不能再被 cancel/die 打断。
第三，清空 error context：
```text
error_context_stack = NULL
debug_query_string = NULL
```
exit callback 可能发生在事务已经 abort、context 已拆、catalog/cache 状态不完整时。
继续调用 error context callback 反而可能二次出错。
第四，按顺序运行：
```text
shmem_exit(code)
on_proc_exit callbacks
exit(code)
```
### 6.3 `shmem_exit()` 的三段
`shmem_exit()` 是 process exit cleanup 的中心。
伪调用链：
```text
shmem_exit(code)
  -> shmem_exit_inprogress = true
  -> LWLockReleaseAll()
  -> before_shmem_exit callbacks
  -> dsm_backend_shutdown()
  -> on_shmem_exit callbacks
  -> shmem_exit_inprogress = false
```
先 `LWLockReleaseAll()` 是为了避免带锁进入 cleanup。
有些 callback 会访问 shared state。
有些 LWLock 甚至可能来自 DSM。
如果先 detach DSM 再处理 held LWLocks，held lock tracking 可能指向已经 detach 的地址。
`before_shmem_exit` callback 可以依赖系统大部分能力仍可用。
典型例子：
```text
ShutdownPostgres()
ReplicationSlotShmemExit()
BeforeShmemExit_Files()
Async_UnlistenOnExit()
RemoveTempRelationsCallback()
pgstat_shutdown_hook()
ParallelWorkerShutdown()
logical worker cleanup
```
`on_shmem_exit` callback 接近低层 shared identity teardown。
典型例子：
```text
RemoveProcFromArray()
ProcKill()
CleanupProcSignalState()
CleanupInvalidationState()
AtProcExit_Buffers()
pgstat_beshutdown_hook()
MarkPostmasterChildInactive()
```
`dsm_backend_shutdown()` 被硬编码在两者之间。
它没有简单注册成一个 `on_shmem_exit` callback，是为了让 DSM detach callback 有自己的逐个调用和错误隔离逻辑。
### 6.4 `ShutdownPostgres()` 兜底 active transaction
backend exit 时不一定 idle。
可能处于：
```text
idle in transaction
正在执行 query
ERROR 后的 aborted transaction block
subtransaction 中
ROLLBACK 过程中再次出错
```
`postinit.c` 很早注册：
```text
before_shmem_exit(ShutdownPostgres, 0)
```
`ShutdownPostgres()` 做：
```text
AbortOutOfAnyTransaction()
LockReleaseAll(USER_LOCKMETHOD, true)
```
`AbortOutOfAnyTransaction()` 会循环清掉 top transaction 和 subtransaction 状态，直到回到默认事务块状态。
这解释了为什么 `FATAL` 能清理事务。
`FATAL` 不回 command loop，但仍通过 `proc_exit()` 触发 `ShutdownPostgres()`。
只要进程和 shared memory 可信，事务 abort、ProcArray 清理、ResourceOwner 释放和 lock 释放仍能完成。
### 6.5 transaction end 与 process exit 的交界
`AbortTransaction()` 的关键顺序：
```text
RecordTransactionAbort()
  -> ProcArrayEndTransaction(MyProc, latestXid)
  -> ResourceOwnerRelease(BEFORE_LOCKS)
  -> AtEOXact_Buffers(false)
  -> AtEOXact_Inval(false)
  -> ResourceOwnerRelease(LOCKS)
  -> ResourceOwnerRelease(AFTER_LOCKS)
  -> AtEOXact_Files(false)
```
`CommitTransaction()` 中对应的是：
```text
RecordTransactionCommit()
  -> ProcArrayEndTransaction(MyProc, latestXid)
  -> post-commit cleanup
```
这说明事务 running state 清理早于 process identity teardown。
三类清理不要混淆：
| 函数 | 清理什么 | 何时发生 |
| --- | --- | --- |
| `ProcArrayEndTransaction()` | 当前 transaction 的 running XID / xmin / SubXID / vacuum flags。 | commit / abort。 |
| `ProcArrayRemove()` | 当前 process 的 ProcArray membership。 | process shmem exit。 |
| `ProcKill()` | 当前 process 对 `PGPROC` slot、latch、semaphore、lock group 的 ownership。 | ProcArray removal 之后。 |
如果 backend exit 时还有 active transaction，`ShutdownPostgres()` 先把事务 abort 到确定边界。
然后 `RemoveProcFromArray()` 移除 process membership。
最后 `ProcKill()` 归还 `PGPROC`。
### 6.6 postmaster 处理正常 child exit
postmaster wait 到 backend 退出后进入：
```text
CleanupBackend(bp, exitstatus)
```
初判：
```text
exit status 0:
  正常
exit status 1:
  FATAL，但认为 child 自己做了 cleanup
其它:
  crashed = true
```
随后：
```text
ReleasePostmasterChildSlot(bp)
  -> MarkPostmasterChildSlotUnassigned(child_slot)
```
只有 slot 是 `PM_CHILD_ASSIGNED` 时，`MarkPostmasterChildSlotUnassigned()` 返回 true。
这表示 child 已运行 `MarkPostmasterChildInactive()`。
如果返回 false，即使 exit status 看起来正常，也会被当成 crash。
这个设计避免 postmaster 误信一个没有 clean detach 的 child。
### 6.7 crash child 触发 restart
`CleanupBackend()` 判定 crash 后：
```text
HandleChildCrash(pid, exitstatus, procname)
```
它不清理 `PGPROC`。
它做：
```text
LogChildExit()
LOG: terminating any other active server processes
HandleFatalError(PMQUIT_FOR_CRASH, true)
```
其它 backend 收到 `SIGQUIT` 后运行 `quickdie()`。
`quickdie()` 的关键语义：
```text
不运行 proc_exit()
不运行 atexit()
不清理 transaction
直接 _exit(2)
```
原因有两个：
```text
signal handler 中运行复杂 cleanup 不安全。
shared memory may be corrupted。
```
这就是 crash restart 和 `FATAL` 的分水岭。
`FATAL` 是“我不能继续”。
crash restart 是“旧 shared memory 可能不能信”。
### 6.8 postmaster 重建 shared memory
postmaster 不会在第一个 child crash 后立刻重建。
它先等所有非 syslogger child 退出。
当 `FatalError && pmState == PM_NO_CHILDREN`：
```text
LOG: all server processes terminated; reinitializing
if remove_temp_files_after_crash:
  RemovePgTempFiles()
ResetBackgroundWorkerCrashTimes()
shmem_exit(1)
LocalProcessControlFile(true)
ResetShmemAllocator()
ShmemCallRequestCallbacks()
CreateSharedMemoryAndSemaphores()
UpdatePMState(PM_STARTUP)
maybe_start_io_workers()
StartChildProcess(B_STARTUP)
```
这里的 `shmem_exit(1)` 是 postmaster 释放旧 shared memory / semaphore 等 IPC resources。
它不是替崩溃 backend 跑 `ProcKill()`。
`ResetShmemAllocator()` 把 shared memory request state 回到初始状态，但保留已注册 callbacks。
`CreateSharedMemoryAndSemaphores()` 重新创建 segment、初始化 allocator、初始化各子系统 shared state、启动 DSM postmaster 设施，并调用 `shmem_startup_hook`。
startup process 随后用 WAL recovery 恢复持久状态。
`xlog.c` 还会处理：
```text
ResetUnloggedRelations(UNLOGGED_RELATION_CLEANUP)
DeleteAllExportedSnapshotFiles()
ResetUnloggedRelations(UNLOGGED_RELATION_INIT)
```
## 7. 生命周期 / ownership / cleanup
### 7.1 创建阶段
简化生命周期：
```text
postmaster:
  分配 PMChild slot
  fork/exec child
child:
  on_exit_reset()
  InitProcess()
    -> RegisterPostmasterChildActive()
    -> 取 PGPROC
    -> on_shmem_exit(ProcKill)
  InitProcessPhase2()
    -> ProcArrayAdd(MyProc)
    -> on_shmem_exit(RemoveProcFromArray)
  InitPostgres()
    -> before_shmem_exit(ShutdownPostgres)
    -> 初始化 catalog/cache/portal/temp file 等
```
`on_exit_reset()` 很重要。
child 不能继承 postmaster 的 exit callbacks。
否则 child 退出时可能关闭 postmaster listen socket、删除 lockfile 或释放 postmaster IPC。
### 7.2 持有阶段
同一个 backend 同时持有多类资源：
| 资源 | owner 表达 | 退出时 cleanup |
| --- | --- | --- |
| backend-local 内存 | `MemoryContext` tree | transaction cleanup / OS。 |
| transaction resources | `ResourceOwner` tree | `AbortOutOfAnyTransaction()`。 |
| `PGPROC` slot | `MyProc` / `MyProcNumber` | `ProcKill()`。 |
| ProcArray membership | `proc->pgxactoff` / dense arrays | `ProcArrayRemove()`。 |
| postmaster child liveness | `PMChildFlags[slot]` | `MarkPostmasterChildInactive()`。 |
| temp file VFDs | VFD cache + flags | `AtEOXact_Files()` / `BeforeShmemExit_Files()`。 |
| DSM callbacks | DSM backend-local list | `dsm_backend_shutdown()`。 |
| replication slot active ownership | shared slot fields + `MyReplicationSlot` | `ReplicationSlotShmemExit()`。 |
`ProcKill()` 不是总 cleanup。
它只是低层 process identity 的释放。
### 7.3 释放阶段
正常释放顺序：
```text
事务语义收尾
  -> AbortOutOfAnyTransaction()
     -> ProcArrayEndTransaction()
     -> ResourceOwnerRelease()
     -> locks / buffers / files / snapshots cleanup
高层 process exit cleanup
  -> before_shmem_exit callbacks
DSM cleanup
  -> dsm_backend_shutdown()
低层 shared identity cleanup
  -> RemoveProcFromArray()
  -> ProcKill()
  -> ProcSignal / sinval / pgstat cleanup
  -> MarkPostmasterChildInactive()
process-local cleanup
  -> on_proc_exit callbacks
  -> exit()
postmaster local cleanup
  -> wait()
  -> ReleasePostmasterChildSlot()
```
越早的 cleanup 越能依赖完整系统。
越晚的 cleanup 越接近 shared memory identity teardown。
## 8. 正确性机制层次
### 8.1 LIFO callback
`ipc.c` 的三个 callback list 都是 LIFO。
后初始化的对象通常依赖先初始化的对象。
退出时反向拆除。
这保证晚注册的 `RemoveProcFromArray()` 早于早注册的 `ProcKill()` 执行。
### 8.2 ProcArrayLock / XidGenLock
`ProcArrayRemove()` 持有：
```text
ProcArrayLock exclusive
XidGenLock exclusive
```
它维护：
```text
ProcArrayStruct->pgprocnos[]
ProcGlobal->xids[]
ProcGlobal->subxidStates[]
ProcGlobal->statusFlags[]
PGPROC->pgxactoff
latestCompletedXid / xactCompletionCount for 2PC removal
```
普通 backend exit 调用 `ProcArrayRemove(MyProc, InvalidTransactionId)`。
这要求 live XID 已经被事务层清理。
### 8.3 PMChildFlags
`PMChildFlags` 是 postmaster 判断 child 是否 clean detach 的 conservative signal。
状态含义：
| 状态 | 含义 |
| --- | --- |
| `PM_CHILD_ASSIGNED` | slot 已分配，但 child 不再 active 使用 shared memory。 |
| `PM_CHILD_ACTIVE` | child 正在使用 shared memory。 |
| `PM_CHILD_WALSENDER` | active wal sender。 |
| `PM_CHILD_UNUSED` | postmaster slot 空闲。 |
child 正常进入 shared memory 前置 `ACTIVE`。
child 正常 shmem exit 末尾退回 `ASSIGNED`。
postmaster 只有看到 `ASSIGNED` 才相信 cleanup 完成。
这是 fail-safe：
```text
不能证明 clean，就按不 clean 处理。
```
### 8.4 postmaster death pipe
postmaster death pipe 是 liveness 机制。
它避免 child 在 supervisor 死亡后长期运行。
`WL_EXIT_ON_PM_DEATH` 把这个约束放进常见 wait path。
长期 vacuum、archiver、walwriter、condition variable wait、socket wait 都需要处理这个事件。
### 8.5 WAL recovery
shared memory reset 不能恢复磁盘状态。
磁盘状态依赖 WAL 和 startup process。
例子：
| 状态 | 恢复机制 |
| --- | --- |
| committed transaction | WAL / pg_xact 让 redo 后仍可见。 |
| in-progress-at-crash transaction | recovery 通过 WAL / clog / storage rules 处理。 |
| unlogged relation | 不 WAL-log 数据页，必须按 init fork reset。 |
| temp files | 不属于 WAL 恢复范围，由 `RemovePgTempFiles()` 扫描。 |
## 9. 错误路径 / 异常路径 / fallback
### 9.1 exit callback 再次报错
`proc_exit_prepare()` 设置 `proc_exit_inprogress = true`。
一旦进入这个状态，`ereport()` 不应把控制流送回 main loop。
callback 执行前，callback index 先递减。
如果 callback 中再次 `ERROR` 或 `FATAL`，同一个 callback 不会无限重复调用。
但这只是防止递归。
它不表示 callback 可以执行复杂、高风险逻辑。
exit callback 应尽量短，不执行用户代码，不依赖容易失效的高层状态。
### 9.2 `exit()` 与 `_exit()`
`ipc.c` 注册 `atexit_callback()`。
如果代码错误地直接调用 `exit()`，`atexit` 仍会尝试：
```text
proc_exit_prepare(-1)
```
但 `_exit()` 不运行 `atexit`。
这种情况下，`PMChildFlags` 通常不会退回 `ASSIGNED`。
postmaster 会通过死手开关把它当作 failed cleanup。
### 9.3 `quickdie()` 为什么不清理
`quickdie()` 运行在 `SIGQUIT` handler。
它面对两个风险：
```text
signal handler 中调用复杂 cleanup 不安全。
shared memory 可能已经被 crash backend 破坏。
```
所以它只做极少动作：
```text
阻塞嵌套 SIGQUIT
HOLD_INTERRUPTS()
必要时给 client 发 warning
_exit(2)
```
这不是偷懒。
这是 crash path 的正确性边界。
### 9.4 `restart_after_crash=off`
如果 `restart_after_crash` 关闭，postmaster 不会自动恢复到可连接状态。
如果 startup process 自己失败，postmaster 也不会不断重试。
它会等 children 退出后退出 postmaster。
所以 crash restart 是 policy 控制的恢复路径，不是无条件行为。
### 9.5 startup packet 早期退出
`backend_startup.c` 处理 startup packet 早期有特殊边界。
此时 backend 还没严肃使用 shared memory。
代码会检查：
```text
check_on_shmem_exit_lists_are_empty()
```
如果已经注册 `before_shmem_exit` / `on_shmem_exit`，说明可能已有 shared state 需要 undo。
此时就不能再把退出当作“直接丢掉进程即可”。
### 9.6 2PC 边界
prepared transaction 不是普通 backend exit。
准备成功后，事务状态会转移到 prepared transaction 的 shared state / dummy proc 语义。
`ProcArrayClearTransaction()` 只清当前 backend 的 transaction fields。
`COMMIT PREPARED` / `ROLLBACK PREPARED` 才让 prepared transaction 真正离开 running set。
因此不能根据“backend 不在了”推断所有事务语义都结束了。
## 10. 成本、资源与跨模块传播
普通 exit 成本通常不是 hot path，但连接 churn 高时会显现。
主要成本：
| 成本 | 扩张变量 | 说明 |
| --- | --- | --- |
| ProcArray remove | `numProcs` | dense arrays `memmove`，并更新后续 `pgxactoff`。 |
| ResourceOwner release | 持有资源数 | buffer pin、lock、snapshot、file、catcache ref 越多，abort/exit 越重。 |
| temp file cleanup | VFD 数和文件数 | transaction-local 和 interXact temp files 需要 close/unlink。 |
| DSM callbacks | DSM segment / callback 数 | parallel query、logical apply、DSA 等路径放大。 |
| stats / logging | 配置 | `log_disconnections`、pgstat shutdown hook 增加动作。 |
crash restart 是实例级成本。
它影响：
```text
所有 active sessions 断开
backend-local caches 全部丢失
shared memory 整体重建
background workers 重新启动
startup process WAL recovery
temp files 扫描删除
unlogged relations reset
prepared transaction prescan
```
主要扩张变量：
| 成本 | 扩张变量 |
| --- | --- |
| WAL recovery | crash 前未 checkpoint 的 WAL 量、I/O 带宽、redo 复杂度。 |
| temp cleanup | tablespace 数、`pgsql_tmp` 文件数量、目录规模。 |
| unlogged reset | unlogged relation 数量和大小。 |
| shmem init | shared_buffers、MaxBackends、扩展 shared memory request。 |
| client 恢复 | 连接数、应用重连策略、statement/cache 重建。 |
跨模块传播：
| 模块 | 贡献 |
| --- | --- |
| `xact.c` | active transaction 先 commit/abort 到确定状态。 |
| `procarray.c` | 撤销 running transaction 和 process membership。 |
| `proc.c` | 释放 `PGPROC`、latch、semaphore、lock group identity。 |
| `resowner.c` / lock manager | 释放 locks、pins、snapshots、files。 |
| `fd.c` | 清理 temp files。 |
| `slot.c` | 释放 replication slot active ownership。 |
| `dsm.c` | detach dynamic shared memory。 |
| `pmsignal.c` / `pmchild.c` | 让 postmaster 判断 clean detach。 |
| `xlog.c` | crash recovery 和 unlogged reset。 |
可迁移规律：
```text
process exit 是资源 ownership 图的逆向遍历，不是一个模块内部函数。
```
## 11. 观测与诊断入口
普通退出和 `FATAL` 可见现象：
```text
server log 中的 FATAL / disconnection log
pg_stat_activity 中 backend 消失
pg_locks 中相关 locks 消失
客户端收到 connection terminated / administrator command 等错误
postmaster 不输出 all server processes terminated; reinitializing
```
crash restart 可见现象：
```text
LOG: server process ... was terminated by signal ...
LOG: terminating any other active server processes
客户端看到 terminating connection because of crash of another server process
LOG: all server processes terminated; reinitializing
startup/recovery 相关日志
```
postmaster death 可见现象：
```text
客户端连接断开
child 在 wait path 检测 postmaster death 后退出
没有原 postmaster 输出后续 crash restart 日志
下一次启动有新的 postmaster 日志和 recovery 日志
```
SQL 看不到的状态：
```text
某个 on_shmem_exit callback 执行到了第几个。
PMChildFlags 当前值。
RemoveProcFromArray() 与 ProcKill() 之间的短窗口。
quickdie 是否跳过了某个具体 callback。
postmaster 因 exit status 还是 PMChildFlags false 判定 crash。
```
这些需要 server log、gdb、tracepoints、debug logging 或源码跟读。
普通 / FATAL exit 的 gdb 断点：
```gdb
break proc_exit
break shmem_exit
break ShutdownPostgres
break RemoveProcFromArray
break ProcKill
break MarkPostmasterChildInactive
break CleanupBackend
```
期望顺序：
```text
backend:
  proc_exit
  shmem_exit
  ShutdownPostgres
  RemoveProcFromArray
  ProcKill
  MarkPostmasterChildInactive
postmaster:
  CleanupBackend
  ReleasePostmasterChildSlot
```
crash restart 的 gdb 断点：
```gdb
break HandleChildCrash
break quickdie
break ResetShmemAllocator
break CreateSharedMemoryAndSemaphores
break StartupXLOG
```
日志判断边界：
```text
只有 FATAL / terminating connection:
  通常是单 backend 退出。
terminating any other active server processes + all server processes terminated; reinitializing:
  crash restart cycle。
```
`pg_postmaster_start_time()` 不能单独判断 backend crash restart。
因为 backend crash restart 时 postmaster process 可能仍然活着。
postmaster start time 不变，并不表示没有发生过 crash restart。
## 12. 常见误区
误区一：
```text
FATAL 就是 PostgreSQL crash。
```
纠正：`FATAL` 是 process 退出级错误；只要 `proc_exit(1)` clean detach，postmaster 不重启实例。
误区二：
```text
postmaster 会清理崩溃 backend 的 PGPROC。
```
纠正：postmaster 不在旧 shared memory 中精细修复；它触发 crash restart。
误区三：
```text
OS 会回收进程资源，所以 PostgreSQL 不需要 exit callback。
```
纠正：OS 只能回收本地内存和 fd；ProcArray、PGPROC、locks、slots、DSM、sinval 等 shared state 需要 PostgreSQL 撤销。
误区四：
```text
quickdie 是更快的 proc_exit。
```
纠正：`quickdie()` 故意绕开 `proc_exit()`，因为 shared memory 可能不可信。
误区五：
```text
postmaster death 和 backend crash restart 是同一件事。
```
纠正：backend crash restart 需要 postmaster 活着监督重建；postmaster death 是 supervisor 消失，child 应 fail-stop。
误区六：
```text
ProcKill() 等于事务 abort。
```
纠正：事务 abort 在 `xact.c` 完成，核心是 `ProcArrayEndTransaction()`；`ProcKill()` 是 process identity teardown。
误区七：
```text
RemovePgTempFiles() 是普通退出的主要 temp cleanup。
```
纠正：普通退出靠 ResourceOwner、`AtEOXact_Files()`、`BeforeShmemExit_Files()`；`RemovePgTempFiles()` 是 startup / crash cleanup 的实例级扫描。
## 13. 课堂实验
### 实验一：观察 `FATAL` 仍然 clean detach
目标：
```text
证明 pg_terminate_backend() 触发单 backend FATAL，不触发 crash restart。
```
会话 A：
```sql
BEGIN;
CREATE TEMP TABLE t_exit(a int);
INSERT INTO t_exit VALUES (1);
SELECT pg_backend_pid();
```
会话 B：
```sql
SELECT pid, state, backend_xid, backend_xmin
FROM pg_stat_activity
WHERE pid = <pid>;
SELECT locktype, mode, granted
FROM pg_locks
WHERE pid = <pid>;
```
会话 B 终止 A：
```sql
SELECT pg_terminate_backend(<pid>);
```
期望：
```text
会话 A 收到 FATAL。
该 pid 从 pg_stat_activity 和 pg_locks 消失。
日志没有 all server processes terminated; reinitializing。
```
源码回扣：
```text
die()
  -> ProcessInterrupts()
  -> ereport(FATAL)
  -> proc_exit(1)
  -> ShutdownPostgres()
  -> RemoveProcFromArray()
  -> ProcKill()
  -> MarkPostmasterChildInactive()
```
### 实验二：验证 callback LIFO 顺序
断点：
```gdb
break proc_exit
break shmem_exit
break ShutdownPostgres
break RemoveProcFromArray
break ProcKill
break MarkPostmasterChildInactive
```
触发：
```sql
SELECT pg_terminate_backend(<debug_backend_pid>);
```
记录：
```text
每个断点所在 pid。
MyProc 何时仍非 NULL。
proc->pid 何时置 0。
PMChildFlags 何时退回 assigned。
```
思考：
```text
如果 MarkPostmasterChildInactive() 早于 ProcKill()，postmaster 会误信什么？
如果 ProcKill() 早于 RemoveProcFromArray()，ProcArrayRemove() 还依赖哪些字段？
```
### 实验三：触发 crash restart
警告：
```text
只在一次性开发实例上做，不要在共享环境或生产环境做。
```
会话 A：
```sql
SELECT pg_backend_pid();
```
OS shell：
```bash
kill -SEGV <pid>
```
或：
```bash
kill -QUIT <pid>
```
期望日志：
```text
server process ... was terminated by signal ...
terminating any other active server processes
all server processes terminated; reinitializing
database system was interrupted
```
源码回扣：
```text
CleanupBackend()
  -> HandleChildCrash()
  -> HandleFatalError(PMQUIT_FOR_CRASH, true)
  -> other children quickdie()
  -> _exit(2)
  -> postmaster waits PM_NO_CHILDREN
  -> RemovePgTempFiles()
  -> shmem_exit(1)
  -> ResetShmemAllocator()
  -> CreateSharedMemoryAndSemaphores()
  -> startup process recovery
```
### 实验四：源码跟读 postmaster death
跟读入口：
```text
InitPostmasterDeathWatchHandle()
PostmasterIsAlive()
WaitLatch()
AddWaitEventToSet()
WaitEventSetWait()
```
画出状态：
```text
postmaster_alive_fds[POSTMASTER_FD_OWN]
postmaster_alive_fds[POSTMASTER_FD_WATCH]
WL_POSTMASTER_DEATH
WL_EXIT_ON_PM_DEATH
postmaster_possibly_dead
```
核心问题：
```text
为什么 WaitLatch() 要断言 postmaster-managed caller 必须处理 postmaster death？
为什么 WL_EXIT_ON_PM_DEATH 可以直接 proc_exit(1)，而 WL_POSTMASTER_DEATH 要返回给调用者？
```
## 14. 讨论题
1. 为什么 `FATAL` exit code 1 不触发 postmaster crash restart？
2. 如果 backend 直接 `_exit(0)`，postmaster 为什么仍可能把它当 crash？
3. 为什么 `PMChildFlags` 要由 child 在 `on_shmem_exit` 中退回 `ASSIGNED`，而不是由 postmaster wait 后直接置位？
4. `RemoveProcFromArray()` 和 `ProcKill()` 顺序反过来会破坏哪些不变量？
5. 为什么 `quickdie()` 不应该尝试 `AbortOutOfAnyTransaction()`？
6. postmaster death 时，child 即使能 `proc_exit(1)`，为什么也不能说集群完成了正常 cleanup？
7. crash restart 中哪些状态靠 shared memory 重建，哪些靠 WAL recovery，哪些靠文件系统扫描？
8. 为什么 `pg_postmaster_start_time()` 不能单独判断是否发生过 backend crash restart？
## 15. 本节小结
本节唯一主问题是：
```text
普通退出、FATAL、postmaster death 和 crash restart 分别由谁清理 backend-local 与 shared state？
```
普通退出由 backend 自己清理。
```text
proc_exit()
  -> shmem_exit()
     -> before_shmem_exit
     -> dsm_backend_shutdown
     -> on_shmem_exit
  -> on_proc_exit
  -> exit
```
`FATAL` 仍是可信 cleanup path。
```text
ereport(FATAL)
  -> proc_exit(1)
  -> callback cleanup
```
postmaster death 是 fail-stop，不是 cleanup。
child 通过 `PostmasterIsAlive()`、`WL_POSTMASTER_DEATH`、`WL_EXIT_ON_PM_DEATH` 发现 supervisor 消失后退出。
crash restart 不修旧 shared memory。
```text
child crash or failed clean detach
  -> postmaster HandleChildCrash()
  -> signal other children quickdie
  -> children _exit(2), no callbacks
  -> wait PM_NO_CHILDREN
  -> RemovePgTempFiles()
  -> shmem_exit(1)
  -> ResetShmemAllocator()
  -> CreateSharedMemoryAndSemaphores()
  -> startup process WAL recovery
```
本节可迁移规律：
```text
只要状态跨进程可见，就不能只问谁释放内存；
必须问谁能在什么可信边界下撤销它对其它进程的语义。
```
