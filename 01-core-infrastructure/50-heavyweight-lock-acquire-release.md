# PostgreSQL heavyweight lock acquire/release
## 课程定位
前置知识：已经理解 `PGPROC` 是 backend 在 shared memory 中的身份，`LWLock` 保护共享结构，`ResourceOwner` 负责事务级资源 cleanup。
本节唯一主问题：
```text
LockAcquireExtended() / LockRelease() 如何在 fast path、main lock table、resource owner 和 wait queue 之间推进？
```
核心矛盾：普通 DML 的 relation lock 必须非常便宜，最好不碰全局 lock table；但一旦出现强锁、等待、死锁检测、事务 abort 或 SQL 观测，同一把逻辑锁又必须进入所有 backend 都能一致解释的共享状态。
一句话运行模型：
```text
LOCALLOCK 是当前 backend 的本地账本；
fast path 是少量弱 relation lock 的 per-PGPROC 快速记录；
main lock table 是冲突、等待、唤醒、死锁检测和 pg_locks 的共享事实；
ResourceOwner 让事务、子事务和 ERROR cleanup 能找到应该释放或转移的 LOCALLOCK。
```
学完后应能判断：
- 一次 acquire 为什么可能完全不创建 `LOCK` / `PROCLOCK`。
- 什么时候 fast-path relation lock 必须转入 main lock table。
- 为什么 `ProcLockWakeup()` 在唤醒 waiter 前先授予锁。
- `LockRelease()` 释放的是本 owner 的一次 local hold，还是共享 lock table 中最后一个 hold。
- `ERROR`、`lock_timeout`、deadlock、subtransaction abort 分别由哪条路径收尾。
- `pg_locks.fastpath`、`granted=false`、`waitstart`、`pg_blocking_pids()` 各自能说明什么。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
前面课程已经建立了三层基础设施。
`PGPROC` 给每个 backend 一个 shared-memory identity。
`LWLock` 保护共享数据结构的短临界区。
`ResourceOwner` 负责事务、子事务、Portal 生命周期中的外部资源 cleanup。
heavyweight lock 把这三层组合起来，但它本身不是一个普通互斥原语。
它表达 SQL 层和内核对象层的逻辑冲突。
典型例子：
- `SELECT` 获取 `AccessShareLock`。
- `INSERT` / `UPDATE` / `DELETE` 获取 `RowExclusiveLock`。
- `ALTER TABLE` / `DROP TABLE` 获取 `AccessExclusiveLock`。
- tuple update 冲突可能等待 `transactionid` lock。
- advisory lock 使用 `USER_LOCKMETHOD`。
这些锁的共同点是等待可能很长，需要 deadlock detection，事务结束要自动释放，用户还需要通过 `pg_locks`、日志和 wait event 诊断。
这和底层同步原语不同。
`SpinLock` 保护几条指令级字段更新。
`LWLock` 保护 shared memory 数据结构访问。
`Latch` 承载睡眠和唤醒。
`Heavyweight lock` 表达事务/对象级并发语义，并把等待者纳入死锁检测。
本节不重新展开 `LOCKTAG` 如何表示 relation、tuple、transactionid、advisory 等对象。
本节只追一条生命周期：
```text
lmgr.c 构造 LOCKTAG
  -> LockAcquireExtended()
  -> LOCALLOCK 计数
  -> fast path 或 main lock table
  -> wait queue / ProcSleep
  -> ResourceOwner 记账
  -> LockRelease()
  -> local owner count 下降
  -> fast-path release 或 UnGrantLock
  -> ProcLockWakeup
  -> LOCALLOCK cleanup
```
## 2. 核心矛盾与运行模型
本节 tension 可以压缩成一句话：
```text
无冲突弱锁要低成本本地推进；一旦可能冲突，所有参与者必须进入同一个可等待、可唤醒、可诊断、可 cleanup 的共享事实。
```
如果所有 relation lock 都进入 main lock table，短查询会反复 hash、反复拿 partition LWLock，并为热 relation 放大 cache miss 和 contention。
如果所有弱锁都只存在 backend-local 状态，强锁请求者、wait queue、deadlock detector、`pg_locks` 和 abort cleanup 都无法得到一致事实。
PostgreSQL 的实际选择是四层状态协作。
| 层次 | 位置 | 作用 |
| --- | --- | --- |
| `LOCALLOCK` | backend-local hash | 当前 backend 对某个 locktag/mode 的 local hold count、owner count、shared entry 缓存。 |
| fast path | `PGPROC.fpLockBits` / `fpRelId` | 少量弱 relation lock 的快速记录，避开 main lock table。 |
| main lock table | shared hash: `LOCK` / `PROCLOCK` | 冲突判断、等待队列、唤醒、死锁检测、`pg_locks` 的主要事实来源。 |
| `ResourceOwner` | backend-local owner tree | 事务/子事务/Portal cleanup 时找到哪些 `LOCALLOCK` 要释放或转移。 |
四层不是重复缓存。
`LOCALLOCK` 回答“我这个 backend 对这把锁拿了几次，分别归哪个 owner”。
fast path 回答“我是否持有一个当前不需要进入 main table 的弱 relation lock”。
main lock table 回答“哪些 backend 持有或等待同一 locktag，队列顺序是什么”。
`ResourceOwner` 回答“当前生命周期结束时，哪些 local holds 应该释放或转移”。
理解本课时要先接受一个事实：
```text
main lock table 不是所有持锁状态的唯一入口。
```
fast-path relation lock 可能没有 `LOCK` / `PROCLOCK`。
已经转入 main table 的 lock 仍然需要 `LOCALLOCK` 记录 local count 和 owner。
等待中的 lock 已经增加 `LOCK.requested[]`，但 `LOCALLOCK.nLocks` 还没增加。
所以一把 heavyweight lock 的语义来自组合：
```text
LOCKTAG + lockmode
  + LOCALLOCK.nLocks / lockOwners[]
  + fast-path slot 或 LOCK/PROCLOCK
  + PGPROC wait fields
  + ResourceOwner 生命周期
  + partition LWLock 保护边界
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/README` | heavyweight lock、fast path、wait queue、deadlock detector 的设计边界。 |
| 2 | `src/include/storage/lockdefs.h` | 标准 lock mode 编号和名称。 |
| 3 | `src/include/storage/locktag.h` | `LOCKTAG` 如何表示 relation、tuple、transactionid、advisory 等对象。 |
| 4 | `src/include/storage/lock.h` | `LOCK`、`PROCLOCK`、`LOCALLOCK`、`LockInstanceData`、`DeadLockState`。 |
| 5 | `src/include/storage/proc.h` | `PGPROC` 的 wait fields、fast-path slots、`myProcLocks[]`、`waitStart`。 |
| 6 | `src/backend/storage/lmgr/lmgr.c` | 上层 relation/page/tuple/xact/advisory lock helper 如何调用入口。 |
| 7 | `src/backend/storage/lmgr/lock.c` | acquire/release 主线、fast path transfer、shared hash 操作。 |
| 8 | `src/backend/storage/lmgr/proc.c` | `JoinWaitQueue()`、`ProcSleep()`、`ProcLockWakeup()`、`LockErrorCleanup()`。 |
| 9 | `src/backend/storage/lmgr/deadlock.c` | `DeadLockCheck()` 如何处理 soft/hard deadlock。 |
| 10 | `src/backend/utils/resowner/resowner.c` | `RESOURCE_RELEASE_LOCKS` 阶段如何释放或转移锁。 |
| 11 | `src/backend/utils/adt/lockfuncs.c` | `pg_locks`、`pg_blocking_pids()` 如何构造 SQL 观测结果。 |
| 12 | `src/backend/catalog/system_views.sql` | `pg_locks`、`pg_stat_lock` 视图定义。 |
推荐阅读顺序：
```text
lock.h 的 LOCK / PROCLOCK / LOCALLOCK
  -> lock.c 的 LockAcquireExtended()
  -> lock.c 的 LockRelease() / LockReleaseAll()
  -> proc.c 的 wait queue 和 cleanup
  -> resowner.c 的锁释放阶段
  -> lockfuncs.c 的 pg_locks 观测边界
```
不要从 `deadlock.c` 开始读。
死锁检测是等待路径的 fallback，本节主线是 acquire/release 如何推进状态。
## 4. 关键数据结构与状态
### 4.1 `LOCKTAG` 与 lock method
`LOCKTAG` 是 lock manager 的 hash key。
它包含四个整数域、一个 `locktag_type`、一个 `locktag_lockmethodid`。
relation lock 常见构造是 `SET_LOCKTAG_RELATION(tag, dbid, relid)`。
tuple lock 常见构造是 `SET_LOCKTAG_TUPLE(tag, dbid, relid, blocknum, offnum)`。
transactionid lock 常见构造是 `SET_LOCKTAG_TRANSACTION(tag, xid)`。
advisory lock 使用 `SET_LOCKTAG_ADVISORY()`，并落到 `USER_LOCKMETHOD`。
`LockMethodData` 给出 lock mode conflict table。
当前 DEFAULT 和 USER lock method 使用同一组标准 mode：
- `AccessShareLock`
- `RowShareLock`
- `RowExclusiveLock`
- `ShareUpdateExclusiveLock`
- `ShareLock`
- `ShareRowExclusiveLock`
- `ExclusiveLock`
- `AccessExclusiveLock`
冲突规则不应该从函数名猜。
`LockCheckConflicts()` 实际读取 `lockMethodTable->conflictTab[lockmode]`，再结合 `LOCK.grantMask`、`PROCLOCK.holdMask` 和 lock group membership 判断。
### 4.2 `LOCK`
`LOCK` 存在于 shared hash table `LockMethodLockHash`。
一条 entry 对应一个 `LOCKTAG`。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `tag` | 这个 shared entry 代表哪个逻辑对象。 |
| `grantMask` | 当前已授予 lock modes 的 bitmask。 |
| `waitMask` | 当前等待中的 lock modes 的 bitmask。 |
| `procLocks` | 关联的 `PROCLOCK`，包括 holder 和 waiter。 |
| `waitProcs` | 真正睡眠等待该 lock 的 `PGPROC` 队列。 |
| `requested[]` / `nRequested` | 已请求数量，包括已 grant 和等待中。 |
| `granted[]` / `nGranted` | 已授予数量。 |
两个不变量贯穿全课：
```text
0 <= granted[i] <= requested[i]
0 <= nGranted <= nRequested
```
等待者入队前，`SetupLockInTable()` 已经增加 `requested[]`。
所以等待中的请求已经是 shared fact。
如果后续 `dontWait`、deadlock 或 cancel 放弃等待，必须撤销 request counts。
### 4.3 `PROCLOCK`
`PROCLOCK` 存在于 shared hash table `LockMethodProcLockHash`。
key 是 `LOCK *myLock + PGPROC *myProc`。
它回答的问题是：
```text
某个 backend 对某个 lockable object 持有或等待哪些 mode？
```
关键字段：
| 字段 | 语义 |
| --- | --- |
| `tag.myLock` | 指向 shared `LOCK`。 |
| `tag.myProc` | 指向 holder/waiter 的 `PGPROC`。 |
| `groupLeader` | parallel lock group 的 leader。 |
| `holdMask` | 已授予给这个 backend 的 modes。 |
| `releaseMask` | `LockReleaseAll()` 批量释放时的临时工作区。 |
| `lockLink` | 挂到 `LOCK.procLocks`。 |
| `procLink` | 挂到 `PGPROC.myProcLocks[partition]`。 |
`PROCLOCK` 可能存在但 `holdMask == 0`。
这通常表示该 backend 已经 request 了 lock，但还在等待。
如果等待失败或 `dontWait` 返回，必须删除这种没有 hold 的 `PROCLOCK`。
### 4.4 `LOCALLOCK`
`LOCALLOCK` 存在于 backend-local hash `LockMethodLocalHash`。
key 是 `LOCKTAG + LOCKMODE`。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `hashcode` | `LOCKTAG` hash，避免反复计算并定位 partition。 |
| `lock` / `proclock` | 已进入 main table 时指向 shared entry；fast path 时可以为 `NULL`。 |
| `nLocks` | 当前 backend 对这个 locktag/mode 的总 local hold count。 |
| `lockOwners[]` | 按 `ResourceOwner` 或 session 统计的 local hold count。 |
| `holdsStrongLockCount` | 本 backend 是否增加了 strong fast-path counter，需要 cleanup。 |
| `lockCleared` | relation lock acquire 后是否已经吸收相关 sinval message。 |
shared table 不记录同一 backend 重复获取同一 lockmode 的次数。
例如同一事务中两段代码都获取同一 relation 的 `AccessShareLock`，shared `PROCLOCK.holdMask` 只需要一个 bit。
本 backend 内部必须知道拿了几次、归哪个 owner、什么时候最后一次释放。
这就是 `LOCALLOCK` 的职责。
### 4.5 `PGPROC` 等待字段
`PGPROC` 中和 heavyweight lock 相关的字段包括：
| 字段 | 语义 |
| --- | --- |
| `waitLock` | 当前正在等待的 `LOCK`，未等待时为 `NULL`。 |
| `waitProcLock` | 当前等待对应的 `PROCLOCK`。 |
| `waitLockMode` | 正在等待的 mode。 |
| `heldLocks` | 等待同一 `LOCK` 时自己已持有的 mode mask。 |
| `waitLink` | 挂入 `LOCK.waitProcs` 的链表节点。 |
| `waitStatus` | `WAITING`、`OK`、`ERROR`。 |
| `waitStart` | 等待开始时间，供 `pg_locks.waitstart` 使用。 |
| `myProcLocks[]` | 按 partition 分开的本 backend `PROCLOCK` 链表。 |
| `procLatch` | 被 holder 或 timeout handler 唤醒的 latch。 |
这些字段在 shared memory 中，其他 backend 会在 partition LWLock 下读写。
等待时序简化为：
```text
JoinWaitQueue()
  -> link MyProc into LOCK.waitProcs
  -> set waitLock / waitProcLock / waitLockMode
  -> waitStatus = WAITING
ProcLockWakeup()
  -> GrantLock(lock, proc->waitProcLock, mode)
  -> ProcWakeup(proc, OK)
  -> SetLatch(proc->procLatch)
```
关键不变量：
```text
被唤醒前，授予方已经把 shared lock table 改成“waiter 已持有锁”。
```
waiter 从 `ProcSleep()` 返回后不能再补写 shared state。
它只更新自己的 `LOCALLOCK`。
### 4.6 fast path 与 strong counter
fast path 只覆盖特定弱 relation lock。
当前 eligibility：
```text
DEFAULT_LOCKMETHOD
LOCKTAG_RELATION
locktag_field1 == MyDatabaseId
MyDatabaseId != InvalidOid
mode < ShareUpdateExclusiveLock
```
也就是 `AccessShareLock`、`RowShareLock`、`RowExclusiveLock`。
状态在 `PGPROC` 的 `fpInfoLock`、`fpLockBits`、`fpRelId` 中。
每个 group 有 `FP_LOCK_SLOTS_PER_GROUP` 个 slot，当前为 16。
本地 `FastPathLocalUseCounts[group]` 记录当前 backend 认为该 group 用了多少 slot。
强锁请求不能看不见这些弱锁。
因此有全局 strong counter：
```text
FastPathStrongRelationLocks->count[1024]
```
强锁 acquire 时先 `BeginStrongLockAcquire()` 增加 counter，再 `FastPathTransferRelationLocks()` 扫描所有 `PGPROC` 的 fast-path slots，把匹配 relation 的弱锁转入 main lock table。
transfer 后，强锁的冲突检查和 deadlock detection 只看 main lock table。
这解释了一个常见现象：
```text
原本 pg_locks.fastpath = true 的 AccessShareLock，
在另一个 session 请求 AccessExclusiveLock 后，
可能变成 fastpath = false。
```
不是 holder 主动重拿，而是强锁请求者做了 transfer。
### 4.7 ResourceOwner lock cache
`ResourceOwner` 对 lock 使用特殊接口：
```text
ResourceOwnerRememberLock(owner, locallock)
ResourceOwnerForgetLock(owner, locallock)
```
它不是普通 `ResourceOwnerDesc` 资源。
原因是 lock cleanup 有特殊语义：
- top-level transaction 结束时批量 `ProcReleaseLocks()`。
- subtransaction commit 时不释放，而是转移到父 owner。
- subtransaction abort 时只释放当前 owner 的 locks。
- lock list 是 lossy cache，超过 `MAX_RESOWNER_LOCKS` 后改为扫描本地 lock hash。
`LOCALLOCK.lockOwners[]` 从 lock 找 owner。
`ResourceOwner.locks[]` 从 owner 找 lock。
两者都只在当前 backend 内有效。
shared lock table 不知道 ResourceOwner。
## 5. `LockAcquireExtended()` 主流程
典型入口来自 `lmgr.c`：
```text
LockRelationOid(relid, lockmode)
  -> SetLocktagRelationOid(&tag, relid)
  -> LockAcquireExtended(&tag, lockmode, false, false, true, &locallock, false)
  -> AcceptInvalidationMessages()
  -> MarkLockClear(locallock)
```
`LockRelation()`、`LockTuple()`、`XactLockTableWait()`、advisory lock SQL 函数也进入同一套 lock manager。
差别只是 `LOCKTAG`、mode、`sessionLock`、`dontWait` 和上层 post-acquire 工作不同。
### 5.1 参数检查与 owner 选择
`LockAcquireExtended()` 先检查 lock method 和 lock mode 是否有效。
recovery 中某些 database object lock mode 也会被拒绝。
然后选择 owner：
```text
if sessionLock:
  owner = NULL
else:
  owner = CurrentResourceOwner
```
`owner == NULL` 表示 session-level lock。
`owner != NULL` 表示 transaction/subtransaction/portal 级 lock，需要 ResourceOwner 跟踪。
shared table 不区分这两类。
区分发生在 `LOCALLOCK.lockOwners[]`。
### 5.2 创建或复用 LOCALLOCK
源码构造 `LOCALLOCKTAG`：
```text
localtag.lock = *locktag
localtag.mode = lockmode
```
然后在 `LockMethodLocalHash` 中 `HASH_ENTER`。
新 entry 初始化：
- `lock = NULL`
- `proclock = NULL`
- `hashcode = LockTagHashCode(locktag)`
- `nLocks = 0`
- `holdsStrongLockCount = false`
- `lockCleared = false`
- `lockOwners` 分配在 `TopMemoryContext`
如果 entry 已存在，先确保 `lockOwners[]` 有空间。
### 5.3 已经持有的快速本地路径
如果 `locallock->nLocks > 0`，说明当前 backend 已经持有同一 `LOCKTAG + LOCKMODE`。
这时不访问 shared memory。
只调用：
```text
GrantLockLocal(locallock, owner)
```
它增加 `locallock->nLocks`，更新对应 owner 的 `nLocks`，必要时 `ResourceOwnerRememberLock()`。
返回值可能是 `LOCKACQUIRE_ALREADY_HELD` 或 `LOCKACQUIRE_ALREADY_CLEAR`。
这条路径解释了为什么同一事务中反复打开同一 relation 不会反复进入 shared lock table。
### 5.4 fast path 尝试
如果满足 `EligibleForRelationFastPath()`，则尝试 fast path。
流程：
```text
if FastPathLocalUseCounts[group] < FP_LOCK_SLOTS_PER_GROUP:
  LWLockAcquire(MyProc->fpInfoLock, exclusive)
  if FastPathStrongRelationLocks->count[fasthashcode] == 0:
      FastPathGrantRelationLock(relid, lockmode)
  LWLockRelease(MyProc->fpInfoLock)
```
成功后：
```text
locallock->lock = NULL
locallock->proclock = NULL
GrantLockLocal(locallock, owner)
return LOCKACQUIRE_OK
```
fast path 成功意味着：
```text
main lock table 没有这把锁的 LOCK/PROCLOCK；
PGPROC fast-path slot + LOCALLOCK 共同表示持有。
```
为什么读取 strong counter 时要拿 `fpInfoLock`？
因为强锁请求者 transfer 时也会拿每个 backend 的 `fpInfoLock`。
弱锁请求者在持有自己的 `fpInfoLock` 时看到 strong counter 为 0，可以保证要么当前没有 transfer，要么强锁请求者稍后会看到刚放进去的 slot。
### 5.5 fast path 失败的原因
fast path 失败不等于 acquire 失败。
它只是转入 main lock table。
常见原因：
- 不是 relation lock。
- relation 是 shared relation 或不属于当前 database。
- mode 不在弱锁范围。
- 当前 group 的 fast-path slot 已满。
- strong counter 非 0。
如果 slot limit exceeded，会调用：
```text
pgstat_count_lock_fastpath_exceeded(locktag_type)
```
这能在 `pg_stat_lock.fastpath_exceeded` 中看到累计值。
### 5.6 强锁请求与 transfer
如果 `ConflictsWithRelationFastPath(locktag, lockmode)` 为 true，说明本次请求可能与 fast-path 弱锁冲突。
流程：
```text
BeginStrongLockAcquire(locallock, fasthashcode)
FastPathTransferRelationLocks(lockMethodTable, locktag, hashcode)
```
`BeginStrongLockAcquire()` 增加强锁 counter，并设置 `StrongLockInProgress = locallock`。
如果后续 allocation 或 transfer 失败，`AbortStrongLockAcquire()` 能撤销 counter。
`FastPathTransferRelationLocks()` 会扫描 `ProcGlobal->allProcs`。
对每个 backend，它拿 `proc->fpInfoLock`，检查 database/group/relid，找到匹配 slot 后拿目标 lock partition LWLock，调用 `SetupLockInTable()` 和 `GrantLock()`，再清 fast-path bit。
transfer 的目的不是优化。
它是 correctness 边界：
```text
deadlock detector 不扫描 fast path；任何可能参与等待和死锁的锁，必须先进入 main lock table。
```
### 5.7 进入 main lock table
如果没有 fast path，就进入 shared hash。
流程：
```text
partitionLock = LockHashPartitionLock(hashcode)
LWLockAcquire(partitionLock, LW_EXCLUSIVE)
proclock = SetupLockInTable(lockMethodTable, MyProc, locktag, hashcode, lockmode)
```
`SetupLockInTable()` 查找或创建 `LOCK`，新建时初始化 masks、lists 和 counts。
然后查找或创建 `PROCLOCK`，新建时设置 `groupLeader`、`holdMask = 0`、`releaseMask = 0`，并挂入 `LOCK.procLocks` 与 `PGPROC.myProcLocks[partition]`。
最后它增加：
```text
lock->nRequested++
lock->requested[lockmode]++
```
注意，此时还没有 grant。
这是等待路径 cleanup 必须撤销的 request fact。
### 5.8 冲突检查
源码先看 waiters：
```text
if conflictTab[lockmode] & lock->waitMask:
  found_conflict = true
else:
  found_conflict = LockCheckConflicts(...)
```
先看 `waitMask` 是为了尊重已有 wait queue。
即使当前 holder 不冲突，前面已经有等待者时，新请求也不能随意越过与自己冲突的 waiter。
`LockCheckConflicts()` 再比较 requested mode 与 `lock->grantMask`。
如果有冲突，还会减掉自己已经持有的 locks。
同一 backend 不阻塞自己。
同一 parallel lock group 的成员通常也不互相阻塞，relation extension lock 是例外。
### 5.9 无冲突 grant
无冲突时：
```text
GrantLock(lock, proclock, lockmode)
waitResult = PROC_WAIT_STATUS_OK
```
`GrantLock()` 修改 shared table：
```text
lock->nGranted++
lock->granted[lockmode]++
lock->grantMask |= LOCKBIT_ON(lockmode)
if granted[lockmode] == requested[lockmode]:
    lock->waitMask &= LOCKBIT_OFF(lockmode)
proclock->holdMask |= LOCKBIT_ON(lockmode)
```
shared state 已经表示该 backend 持有锁。
local state 在函数后段统一通过 `GrantLockLocal()` 更新。
### 5.10 有冲突入队
有冲突时：
```text
waitResult = JoinWaitQueue(locallock, lockMethodTable, dontWait)
```
`JoinWaitQueue()` 不保证一定等待。
它可能返回 `OK`、`WAITING` 或 `ERROR`。
队列规则：
- 通常放到队尾。
- 如果自己已经持有的 locks 会阻塞前面某个 waiter，则插到该 waiter 前。
- 如果插到前面后不冲突，则直接 grant。
- 如果发现双向等待，则记录 simple deadlock 并返回 error。
- 如果 `dontWait` 且确实需要睡眠，则返回 error。
真正等待时它会设置：
```text
lock->waitMask |= LOCKBIT_ON(lockmode)
MyProc->heldLocks = myProcHeldLocks
MyProc->waitLock = lock
MyProc->waitProcLock = proclock
MyProc->waitLockMode = lockmode
MyProc->waitStatus = WAITING
```
并把 `MyProc->waitLink` 挂入 `LOCK.waitProcs`。
### 5.11 `dontWait` 失败 cleanup
如果 `JoinWaitQueue()` 返回 error，`LockAcquireExtended()` 必须撤销前面已经做的共享修改。
主要步骤：
```text
AbortStrongLockAcquire()
if proclock->holdMask == 0:
  delete proclock from lock/proc lists and hash
lock->nRequested--
lock->requested[lockmode]--
release partition LWLock
RemoveLocalLock(locallock) if nLocks == 0
return LOCKACQUIRE_NOT_AVAIL if dontWait
```
如果不是 `dontWait`，则进入 `DeadLockReport()`，该函数不返回。
### 5.12 `WaitOnLock()` 与 `ProcSleep()`
真正等待时，`LockAcquireExtended()` 释放 partition LWLock，然后调用：
```text
WaitOnLock(locallock, owner)
```
`WaitOnLock()` 设置 error context、进程标题，并记录：
```text
awaitedLock = locallock
awaitedOwner = owner
```
这两个变量让 `LockErrorCleanup()` 能在 ERROR/cancel/die 时收尾。
`ProcSleep()` 启用 `DEADLOCK_TIMEOUT`，必要时启用 `LOCK_TIMEOUT`，并写 `MyProc->waitStart`。
等待点是：
```text
WaitLatch(MyLatch, WL_LATCH_SET | WL_EXIT_ON_PM_DEATH, 0,
          PG_WAIT_LOCK | locktag_type)
```
这就是 `pg_stat_activity.wait_event_type = 'Lock'` 的来源。
`deadlock_timeout` 到期后，`CheckDeadLock()` 拿所有 lock partitions，运行 `DeadLockCheck()`。
可能结果：
- `DS_NO_DEADLOCK`: 继续等待。
- `DS_SOFT_DEADLOCK`: 重排 wait queue 并尝试唤醒。
- `DS_HARD_DEADLOCK`: `RemoveFromWaitQueue()` 后报 deadlock error。
- `DS_BLOCKED_BY_AUTOVACUUM`: 可能 cancel 非 wraparound autovacuum。
### 5.13 被授予后的返回
release 方唤醒 waiter 时会先：
```text
GrantLock(lock, proc->waitProcLock, mode)
ProcWakeup(proc, OK)
```
`ProcWakeup()` 从 wait queue 移除 `PGPROC`，清 `waitLock` / `waitProcLock`，设置 `waitStatus = OK`，清 `waitStart`，再 `SetLatch()`。
因此 `ProcSleep()` 返回后，shared lock table 已经完成更新。
`LockAcquireExtended()` 只做本地工作：
```text
Assert(proclock->holdMask has lockmode)
GrantLockLocal(locallock, owner)
FinishStrongLockAcquire()
maybe LogAccessExclusiveLock()
return LOCKACQUIRE_OK
```
源码注释强调，不要在 `ProcSleep()` 返回后补做 shared-state cleanup。
因为 waiter 可能在授予后、返回前被 cancel/die 打断。
### 5.14 relation lock 的 invalidation 边界
`lmgr.c` 的 relation helper 在 acquire 成功后通常做：
```text
AcceptInvalidationMessages()
MarkLockClear(locallock)
```
`lockCleared` 表示已经吸收其他 session 在本次 acquire 前产生的相关 sinval message。
它不是 lock conflict 语义。
`LockRelease()` 在最后一个 local hold 释放前会把它重置为 false。
## 6. `LockRelease()` 主流程
`LockRelease()` 的核心判断是：
```text
这次 release 只是减少本 backend 的 local owner count，还是要真正修改 fast-path slot 或 main lock table 并唤醒 waiter？
```
### 6.1 查找 LOCALLOCK 与 owner
入口：
```text
LockRelease(locktag, lockmode, sessionLock)
```
它用同样的 `LOCALLOCKTAG` 在 `LockMethodLocalHash` 中查找。
找不到或 `nLocks <= 0` 时，输出 WARNING 并返回 false。
然后按 acquire 同样规则选择 owner：
```text
if sessionLock:
  owner = NULL
else:
  owner = CurrentResourceOwner
```
它扫描 `locallock->lockOwners[]`，只减少匹配 owner 的计数。
如果找不到 owner，也输出 WARNING 并返回 false。
这说明一个子事务不能随便释放父 owner 的 lock。
### 6.2 local count 未归零
减少 owner count 后：
```text
locallock->nLocks--
```
如果 `locallock->nLocks > 0`，release 完成。
它不触碰 fast path，不触碰 main lock table，不唤醒 waiter。
重复 acquire 的前 N-1 次 release 都是 backend-local。
最后一次 release 才影响全局状态。
### 6.3 fast-path release
当 local count 归零时，先重置：
```text
locallock->lockCleared = false
```
如果 lock eligible for fast path，且本地 group use count 非零，尝试：
```text
LWLockAcquire(MyProc->fpInfoLock, exclusive)
released = FastPathUnGrantRelationLock(relid, lockmode)
LWLockRelease(MyProc->fpInfoLock)
```
成功则：
```text
RemoveLocalLock(locallock)
return true
```
失败并不一定是错误。
另一个 backend 请求强锁时可能已经把这个 fast-path lock transfer 到 main lock table。
### 6.4 main table release
如果 fast-path release 没成功，进入 main table。
流程：
```text
partitionLock = LockHashPartitionLock(locallock->hashcode)
LWLockAcquire(partitionLock, LW_EXCLUSIVE)
```
通常 `locallock->lock` / `proclock` 已缓存。
如果原本 fast path 后被 transfer，这两个指针可能是 `NULL`，源码会重新查 `LOCK` 和 `PROCLOCK`。
确认当前 `PROCLOCK.holdMask` 包含要释放的 mode 后，调用：
```text
wakeupNeeded = UnGrantLock(lock, lockmode, proclock, lockMethodTable)
CleanUpLock(lock, proclock, lockMethodTable, hashcode, wakeupNeeded)
```
`UnGrantLock()` 同时减少 requested 和 granted：
```text
lock->nRequested--
lock->requested[lockmode]--
lock->nGranted--
lock->granted[lockmode]--
proclock->holdMask &= ~LOCKBIT_ON(lockmode)
```
如果 released mode 与 `lock->waitMask` 有冲突，返回 `wakeupNeeded = true`。
它不是只看 `granted[released_mode] == 0`。
因为剩下的 granted locks 可能属于某个 waiter 自己，不会阻塞自己。
### 6.5 cleanup 与 wakeup
`CleanUpLock()` 做三件事。
如果 `proclock->holdMask == 0`，从 `LOCK.procLocks`、`PGPROC.myProcLocks[partition]` 和 PROCLOCK hash 中删除它。
如果 `lock->nRequested == 0`，从 LOCK hash 中删除 `LOCK`。
如果仍有 request 且 `wakeupNeeded`，调用：
```text
ProcLockWakeup(lockMethodTable, lock)
```
`ProcLockWakeup()` 扫描 wait queue。
对每个 waiter，它检查：
```text
不与前面尚未唤醒 waiter 的请求冲突
且不与当前 granted locks 冲突
```
满足则先 `GrantLock()`，再 `ProcWakeup()`。
这和 LWLock 不同。
heavyweight lock 的 wakeup 不是“允许重试”，而是“shared table 已经表示你持有锁”。
### 6.6 最后移除 LOCALLOCK
main table release 后：
```text
LWLockRelease(partitionLock)
RemoveLocalLock(locallock)
```
`RemoveLocalLock()` 会 forget 剩余 owner、释放 `lockOwners`、撤销 `holdsStrongLockCount`、从 local hash 删除 entry。
正常显式 release 前，owner list 应该已经清空。
但 cleanup 路径可能从不同方向到达，所以这里仍然保守处理。
## 7. 生命周期 / ownership / cleanup
backend 启动时，`InitLockManagerAccess()` 创建 backend-local `LockMethodLocalHash`。
shared memory 初始化时，`LockManagerShmemRequest()` 请求 `LOCK hash`、`PROCLOCK hash` 和 `Fast Path Strong Relation Lock Data`，`LockManagerShmemInit()` 初始化 strong counter mutex。
单次 acquire 时，`LockAcquireExtended()` 创建或复用 `LOCALLOCK`。
fast path 成功时，不创建 shared `LOCK` / `PROCLOCK`。
main table 路径通过 `SetupLockInTable()` 创建或复用 `LOCK` / `PROCLOCK`。
等待时不创建单独 wait object，而是复用 `MyProc->waitLink` 挂入 `LOCK.waitProcs`。
逻辑持有者有三层：
- SQL 语义上是 session-level 或 transaction-level lock。
- 当前 backend local 中是 `LOCALLOCK.nLocks` 和 `lockOwners[]`。
- shared lock manager 中是 fast-path slot 或 `PROCLOCK.holdMask`。
正常显式释放走 `LockRelease()`。
事务结束 bulk release 走：
```text
ResourceOwnerRelease(..., RESOURCE_RELEASE_LOCKS, ...)
```
top-level transaction 时，`resowner.c` 在 `TopTransactionResourceOwner` 上调用：
```text
ProcReleaseLocks(isCommit)
ReleasePredicateLocks(isCommit, false)
```
`ProcReleaseLocks()` 做：
```text
LockErrorCleanup()
LockReleaseAll(DEFAULT_LOCKMETHOD, !isCommit)
LockReleaseAll(USER_LOCKMETHOD, false)
```
含义：
- main transaction commit 释放 DEFAULT method 的非 session locks。
- main transaction abort 释放 DEFAULT method 的所有 locks，包括 DEFAULT session locks。
- USER lock method 只释放 transaction-level advisory locks。
- session-level advisory locks 在事务结束时保留。
子事务 commit 不释放锁。
它调用 `LockReassignCurrentOwner()` 把当前 owner 的 locks 转移到 parent owner。
子事务 abort 调用 `LockReleaseCurrentOwner()`，只释放当前 owner 的 local holds。
如果父事务也持有同一 lock，shared table 不一定释放。
只有当前 backend 对同一 `LOCKTAG + mode` 的 `LOCALLOCK.nLocks` 归零时，才会真正 `UnGrantLock()`。
等待中的 ERROR/cancel/die 最危险。
`WaitOnLock()` 设置 `awaitedLock` 和 `awaitedOwner`。
abort 路径调用 `LockErrorCleanup()`。
如果 `MyProc->waitLink` 还在 wait queue，`RemoveFromWaitQueue()` 会退出队列并 undo request counts。
如果 waitLink 已 detached 且 `waitStatus == OK`，说明授予方已经把 shared table 改成我持有锁。
这时 `GrantAwaitedLock()` 必须补上 `LOCALLOCK` / `ResourceOwner` 记账，让后续 ResourceOwner release 能释放它。
强锁 acquire 的错误窗口由 `StrongLockInProgress` 保护。
`BeginStrongLockAcquire()` 增加 strong counter 后，如果 transfer 或 shared hash allocation 失败，`AbortStrongLockAcquire()` 会撤销 counter。
否则后续弱 relation locks 会长期无法使用 fast path。
backend exit 前，正常事务 cleanup 应已释放 locks。
`ProcKill()` 在 assert build 中检查 `MyProc->myProcLocks[]` 应为空，这是一条防线，不是主要释放路径。
## 8. 正确性机制层次
heavyweight lock correctness 由多个机制分层保证。
| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `LockConflicts[]` | mode 兼容性统一定义。 | 不知道 owner 生命周期。 |
| partition LWLock | `LOCK` / `PROCLOCK` / wait fields 的 shared memory 修改互斥。 | 不表达 SQL 冲突语义。 |
| `LOCALLOCK` | 当前 backend 重复 acquire 和 owner count。 | 其他 backend 看不到。 |
| fast path | 高频弱 relation lock 避开 main table。 | 不参与 deadlock detector。 |
| strong counter + transfer | 强锁能发现并迁移 fast-path 弱锁。 | 不消除强锁等待成本。 |
| wait queue | 阻塞请求的 arrival order 和 soft block。 | 不自动解决所有 deadlock。 |
| deadlock detector | 延迟发现 hard/soft deadlock，并可重排队列。 | 不在每次 wait 前运行。 |
| `ResourceOwner` | abort/subtransaction cleanup 能找到本 backend lock。 | shared table 不记录 ResourceOwner。 |
| latch/wait event | 睡眠、唤醒、可观测等待点。 | latch set 本身不等于锁语义。 |
| invalidation | relation lock 后建立 relcache freshness 边界。 | 不是互斥机制。 |
特别要区分 heavyweight lock 和 LWLock。
LWLock 保护 lock manager 自己的 shared data structure。
Heavyweight lock 是 lock manager 暴露给 SQL/内核模块的逻辑锁。
LWLock wait 没有 deadlock detection。
Heavyweight lock wait 有 wait queue、deadlock detector、ResourceOwner cleanup 和 SQL 观测。
也要区分 heavyweight lock 和 MVCC snapshot。
MVCC 决定读到哪个 tuple version。
Heavyweight lock 决定 DDL/DML/对象操作是否可以并发进入同一语义区。
`AccessShareLock` 不决定 SELECT 的 tuple visibility。
它防止 concurrent `DROP TABLE` / `ALTER TABLE` 破坏 relation 语义。
## 9. 错误路径 / 异常路径 / fallback
fast path slot exceeded 时，源码不报错。
它增加 `pgstat_count_lock_fastpath_exceeded(locktag_type)`，然后进入 main lock table。
正确性不受影响，代价是更多 partition LWLock 和 shared hash 操作。
lock table shared memory 不足时，`SetupLockInTable()` 返回 NULL。
默认 `reportMemoryError=true` 会报 `out of shared memory`，提示增加 `max_locks_per_transaction`。
如果调用者传 `reportMemoryError=false`，返回 `LOCKACQUIRE_NOT_AVAIL`。
源码注释提醒不要和 `dontWait=true` 组合，因为同一个返回值也表示“锁现在不可得”。
`dontWait=true` 仍然调用 `JoinWaitQueue()`。
因为队列插入规则可能发现本请求可以立即 grant。
只有确认必须等待时，才返回 error，并由 `LockAcquireExtended()` undo request counts 和临时 `PROCLOCK`。
`lock_timeout` 或 query cancel 发生在 `ProcSleep()` 的 `CHECK_FOR_INTERRUPTS()`。
ordinary return path 可能被 longjmp 跳过。
关键 cleanup 因此放在 `LockErrorCleanup()`。
hard deadlock 发生时，`CheckDeadLock()` 在持有所有 lock partitions 后调用 `DeadLockCheck()`。
如果不可通过 wait queue 重排解决，`RemoveFromWaitQueue()` 会把当前 waiter 从队列移除，设置 `waitStatus = ERROR`，随后 `DeadLockReport()` 报 `ERROR`。
soft deadlock 可解决时，deadlock detector 会重新排列 wait queue，然后调用 `ProcLockWakeup()`。
这可能避免 abort，但如果 `log_lock_waits` 打开，日志会出现 avoided deadlock。
hot standby/recovery conflict 是旁路。
AccessExclusiveLock 可能通过 WAL 传播给 standby。
`ProcSleep()` 在 hot standby 分支会调用 `ResolveRecoveryConflictWithLock()`，必要时用 `GetLockConflicts()` 和日志说明冲突。
释放时如果 fast-path slot 找不到，不一定是 corruption。
可能另一个 backend 请求强锁时已经 transfer。
`LockRelease()` 会重新查 main table 中的 `LOCK` / `PROCLOCK` 再释放。
## 10. 成本、资源与跨模块传播
heavyweight lock 成本随这些变量扩张：
| 变量 | 成本传播 |
| --- | --- |
| backend 数 | fast-path transfer 扫描 `ProcGlobal->allProcs`；deadlock workspace 和 waits-for graph 随上限扩张。 |
| 热 relation 数 | 如果不能走 fast path，会集中竞争某个 lock partition。 |
| 每事务 relation 数 | `LOCALLOCK`、fast-path slots、lock table entries 增加。 |
| 分区表数量 | 单条 SQL 可能拿很多 relation locks，导致 fastpath exceeded 或 lock table 压力。 |
| wait queue 长度 | `ProcLockWakeup()` 扫描 waiters；deadlock detector 处理 soft edges。 |
| lock group 成员数 | `LockCheckConflicts()` 处理 group leader 和 group members。 |
| `max_locks_per_transaction` | 增加 shared lock table 容量，也增加 shared memory footprint。 |
fast path 优化普通 DML 的弱 relation lock。
一旦有强锁请求，成本从本 backend 局部变成全局协调：
```text
strong counter increment
scan all PGPROC fast-path slots
transfer matching locks
main table conflict check
maybe wait
```
所以 DDL 等强锁在高连接数系统上会放大 lock manager 开销。
没有 fast path 时，瓶颈常在 `LockHashPartitionLock(hashcode)`。
有 fast path 后，普通 DML 减少该 partition 压力，但新成本可能迁移到 per-backend `fpInfoLock`、strong counter mutex、`ProcGlobal->allProcs` 扫描和 `pg_locks` 全局快照。
shared lock table sizing 近似来自：
```text
NLOCKENTS() = max_locks_per_xact * (MaxBackends + max_prepared_xacts)
```
真实压力还取决于 workload：
- 一个事务锁了多少对象。
- 是否有大量 partitioned table。
- 是否有 prepared transaction。
- 是否大量使用 advisory locks。
- 是否 DDL/DML 混合导致 fast-path locks 频繁 transfer。
相邻模块边界：
| 模块 | 连接点 |
| --- | --- |
| relcache / sinval | relation lock 成功后 `AcceptInvalidationMessages()`、`MarkLockClear()`。 |
| transaction manager | `xact.c` 在 commit/abort/subxact 阶段调用 ResourceOwner release。 |
| ProcArray / VXID | transactionid / virtualxid lock 用于等待事务结束；`GetLockConflicts()` 返回 VXID。 |
| hot standby | AccessExclusiveLock WAL 和 recovery conflict。 |
| autovacuum | deadlock detector 可返回 `DS_BLOCKED_BY_AUTOVACUUM`。 |
| stats / monitoring | `pg_locks`、`pg_stat_activity`、`pg_stat_lock`、`log_lock_waits`。 |
| parallel query | lock group leader 影响冲突判断和 blocking PID 输出。 |
lock manager 状态主要由前台 backend 推进。
没有后台进程定期整理 lock table。
状态推进发生在 acquire/release、等待超时后的 deadlock check、事务 cleanup 和 recovery conflict 路径中。
## 11. 观测与诊断入口
`pg_locks` 定义为：
```sql
CREATE VIEW pg_locks AS
    SELECT * FROM pg_lock_status() AS L;
```
`pg_lock_status()` 调用 `GetLockStatusData()` 和 `GetPredicateLockStatusData()`。
regular lock 输出包括 `locktype`、对象字段、`virtualtransaction`、`pid`、`mode`、`granted`、`fastpath`、`waitstart`。
解释边界：
- `granted=true` 表示 held mode。
- `granted=false` 表示该 PGPROC 正在等待 `waitLockMode`。
- `fastpath=true` 表示 held lock 来自 `PGPROC` fast-path slot。
- `waitstart` 来自 `PGPROC.waitStart`，等待刚开始的极短窗口可能为 NULL。
`GetLockStatusData()` 先扫描每个 backend 的 fast-path arrays。
然后按 partition order 拿所有 lock partition shared LWLocks，复制 main table。
fast-path 部分可能是 fuzzy snapshot。
main table 部分在持有所有 partition shared locks 时自洽。
`pg_locks` 适合看当前形态，不是完整因果历史。
`pg_blocking_pids(pid)` 调用 `GetBlockerStatusData()`。
它报告两类 blocker：
```text
hard block:
  已持有与 blocked request 冲突的 lock。
soft block:
  同一 wait queue 中排在 blocked backend 前面，
  且请求 mode 与 blocked request 冲突的 waiter。
```
这比自己 join `pg_locks` 只找 holder 更接近 lock manager 规则。
parallel query 中返回的是 lock group leader PID。
`pg_stat_activity.wait_event_type = 'Lock'` 表示当前卡在 heavyweight lock wait。
`wait_event` 来自 `PG_WAIT_LOCK | locktag_type`，例如 relation、transactionid、tuple、advisory。
它不告诉你 holder、队列顺序、soft blocker 或是否刚发生 queue rearrangement。
本地源码还有 `pg_stat_lock`：
```sql
SELECT locktype, waits, wait_time, fastpath_exceeded, stats_reset
FROM pg_stat_lock;
```
`waits` / `wait_time` 只在等待超过 `deadlock_timeout` 并最终成功 acquire 时累计。
`fastpath_exceeded` 记录 fast-path slot limit 导致无法走 fast path 的次数。
它适合看长期趋势，不适合解释某一刻的所有等待。
`log_lock_waits` 打开后，等待超过 `deadlock_timeout` 会输出 still waiting、acquired after、detected deadlock 或 avoided deadlock by rearranging queue order。
日志 detail 包含 holder 和 wait queue。
短等待不会出现。
gdb 跟读可设断点：
```gdb
break LockAcquireExtended
break FastPathGrantRelationLock
break FastPathTransferRelationLocks
break SetupLockInTable
break JoinWaitQueue
break ProcSleep
break ProcLockWakeup
break LockRelease
break LockErrorCleanup
```
重点观察：
```gdb
p *locallock
p locallock->lock
p locallock->proclock
p *MyProc
p MyProc->waitLock
p MyProc->waitLockMode
p MyProc->waitStatus
```
能直接看到的状态：
- `pg_locks` 当前 holder/waiter。
- `pg_locks.fastpath`。
- `pg_locks.waitstart`。
- `pg_blocking_pids()` blocker。
- `pg_stat_activity.wait_event`。
- `pg_stat_lock` 累计 waits / wait_time / fastpath_exceeded。
- `log_lock_waits` 的长等待日志。
只能推断或需要 gdb 的状态：
- 一把 fast-path lock 是否刚被 transfer。
- `LOCALLOCK` owner count。
- `StrongLockInProgress` 的短暂窗口。
- `lockCleared` 状态。
- lock table partition LWLock contention 的真实 CPU 成本。
## 12. 常见误区
误区 1：main lock table 没有 row 就说明没锁。
fast-path relation lock 可能没有 `LOCK` / `PROCLOCK`。
`pg_locks` 会扫描 `PGPROC` fast-path arrays 补出来，但 gdb 只看 `LockMethodLockHash` 会漏。
误区 2：`fastpath=true` 表示这把锁不真实。
不是。
它只是存储位置不同。
强锁请求出现时，它会被 transfer 并参与冲突判断。
误区 3：唤醒 waiter 后 waiter 自己再拿锁。
heavyweight lock 不是 LWLock retry 模型。
`ProcLockWakeup()` 在 `SetLatch()` 前已经 `GrantLock()`。
误区 4：ResourceOwner 释放锁就是扫描 shared lock table。
ResourceOwner 持有的是 `LOCALLOCK` 指针 cache。
top-level bulk release、subtransaction release 和 reassign 各有不同路径。
误区 5：`deadlock_timeout` 是最长等待时间。
不是。
它只是多久后运行 deadlock detector 和长等待日志的阈值。
真正限制等待的是 `lock_timeout` 或 cancel。
误区 6：holder 才是唯一 blocker。
wait queue 中排在前面的冲突 waiter 也可能是 soft blocker。
`pg_blocking_pids()` 会报告这类情况。
误区 7：session lock 一律事务结束释放。
DEFAULT method 的某些 session locks 在 abort 中会被 `LockReleaseAll(DEFAULT, true)` 释放。
session-level advisory locks 属于 USER method，事务结束不会释放。
误区 8：`wait_event='Lock'` 足够做性能归因。
wait event 只说明当前等待点。
holder、队列、fast-path transfer、lock table 容量、deadlock detector 行为都要结合其他入口。
## 13. 课堂实验
### 实验 1：观察 fast-path relation lock 被强锁 transfer
准备：
```sql
DROP TABLE IF EXISTS hw_lock_demo;
CREATE TABLE hw_lock_demo(id int primary key);
INSERT INTO hw_lock_demo VALUES (1);
```
Session A：
```sql
BEGIN;
SELECT * FROM hw_lock_demo;
SELECT locktype, relation::regclass, mode, granted, fastpath
FROM pg_locks
WHERE pid = pg_backend_pid()
  AND relation = 'hw_lock_demo'::regclass;
```
预期看到 `AccessShareLock granted=true fastpath=true`。
Session B：
```sql
BEGIN;
LOCK TABLE hw_lock_demo IN ACCESS EXCLUSIVE MODE;
```
Session B 会等待。
回到 Session A：
```sql
SELECT pid, mode, granted, fastpath, waitstart
FROM pg_locks
WHERE relation = 'hw_lock_demo'::regclass
ORDER BY granted DESC, pid;
```
Session A 的 `AccessShareLock` 可能已经 `fastpath=false`，Session B 的 `AccessExclusiveLock` 为 `granted=false`。
源码解释：
```text
Session B LockAcquireExtended()
  -> ConflictsWithRelationFastPath()
  -> BeginStrongLockAcquire()
  -> FastPathTransferRelationLocks()
  -> SetupLockInTable() / GrantLock() for Session A
  -> JoinWaitQueue()
```
### 实验 2：区分 hard blocker 和 soft blocker
Session A：
```sql
BEGIN;
LOCK TABLE hw_lock_demo IN ACCESS SHARE MODE;
```
Session B：
```sql
BEGIN;
LOCK TABLE hw_lock_demo IN ACCESS EXCLUSIVE MODE;
```
Session C：
```sql
BEGIN;
LOCK TABLE hw_lock_demo IN ACCESS SHARE MODE;
```
Session C 可能等待，不是因为 A 的 `AccessShareLock` 阻塞 `AccessShareLock`，而是 B 的 `AccessExclusiveLock` 已经排在 wait queue 前面。
观察：
```sql
SELECT a.pid, a.wait_event_type, a.wait_event, pg_blocking_pids(a.pid) AS blockers
FROM pg_stat_activity a
WHERE a.datname = current_database()
  AND a.wait_event_type = 'Lock';
```
再看：
```sql
SELECT pid, mode, granted, fastpath, waitstart
FROM pg_locks
WHERE relation = 'hw_lock_demo'::regclass
ORDER BY granted DESC, waitstart NULLS FIRST, pid;
```
源码解释：
```text
ProcLockWakeup()
  -> later waiter 不能越过前面与它冲突的 un-wakable waiter
pg_blocking_pids()
  -> 报告 hard blockers 和 preceding conflicting waiters
```
### 实验 3：子事务 abort 释放当前 owner 的锁
Session A：
```sql
BEGIN;
SAVEPOINT s1;
LOCK TABLE hw_lock_demo IN ACCESS EXCLUSIVE MODE;
SELECT mode, granted
FROM pg_locks
WHERE pid = pg_backend_pid()
  AND relation = 'hw_lock_demo'::regclass;
```
然后：
```sql
ROLLBACK TO SAVEPOINT s1;
SELECT mode, granted
FROM pg_locks
WHERE pid = pg_backend_pid()
  AND relation = 'hw_lock_demo'::regclass;
```
预期：子事务中获取的 `AccessExclusiveLock` 消失。
源码解释：
```text
AbortSubTransaction()
  -> ResourceOwnerRelease(curTransactionOwner, RESOURCE_RELEASE_LOCKS, false, false)
  -> LockReleaseCurrentOwner()
  -> ReleaseLockIfHeld()
  -> LockRelease()
```
变体：
```sql
BEGIN;
SAVEPOINT s1;
LOCK TABLE hw_lock_demo IN ACCESS EXCLUSIVE MODE;
RELEASE SAVEPOINT s1;
SELECT mode, granted
FROM pg_locks
WHERE pid = pg_backend_pid()
  AND relation = 'hw_lock_demo'::regclass;
```
预期：lock 仍存在，直到父事务结束。
### 实验 4：源码断点跟一条等待路径
建议断点：
```gdb
break LockAcquireExtended
break SetupLockInTable
break JoinWaitQueue
break ProcSleep
break ProcLockWakeup
break LockRelease
```
在等待 backend 观察：
```gdb
p locallock->nLocks
p *locallock->lock
p *locallock->proclock
p MyProc->waitLock
p MyProc->waitLockMode
p MyProc->waitStatus
```
在释放 backend 观察：
```gdb
p lock->waitMask
p lock->nGranted
p lock->nRequested
```
画出这条链：
```text
requested++ before wait
  -> waitProcs link
  -> holder release
  -> UnGrantLock
  -> ProcLockWakeup
  -> GrantLock for waiter
  -> ProcWakeup
  -> waiter GrantLockLocal
```
## 14. 讨论题
1. 为什么 `LOCALLOCK` 必须记录重复 acquire count，而 shared `LOCK` / `PROCLOCK` 不记录同一 backend 的重复次数？
2. fast path 为什么只覆盖弱 relation lock，而不覆盖 tuple lock、transactionid lock 或 advisory lock？
3. 强锁请求为什么要先增加 strong counter，再 transfer 其他 backend 的 fast-path locks？
4. 如果 `ProcLockWakeup()` 先唤醒 waiter，再由 waiter 自己 `GrantLock()`，会出现什么 race？
5. `LockErrorCleanup()` 为什么在发现 waitLink 已经 detached 且 waitStatus 为 OK 时，要调用 `GrantAwaitedLock()`？
6. 子事务 commit 为什么不能释放它获取的 locks，而要 reassign 给父 owner？
7. `pg_locks.granted=false` 和 `pg_stat_activity.wait_event='Lock'` 分别表达什么？哪一个能告诉你 wait queue 前序 waiter？
8. `deadlock_timeout` 为什么不在每次 wait 前立即运行 deadlock detector？这样会漏掉 deadlock 吗？
9. 如果一个系统 `pg_stat_lock.fastpath_exceeded` 很高，你会先怀疑 workload 的哪些形态？
10. 为什么 `pg_locks.fastpath=true` 的锁仍然可以阻塞 DDL？
## 15. 本节小结
本节核心链路：
```text
LockAcquireExtended()
  -> LOCALLOCK
  -> fast path or main lock table
  -> conflict check
  -> wait queue / ProcSleep
  -> GrantLockLocal / ResourceOwnerRememberLock
LockRelease()
  -> owner count
  -> local count
  -> fast-path release or UnGrantLock
  -> CleanUpLock
  -> ProcLockWakeup
  -> RemoveLocalLock
```
核心状态和边界：
- `LOCALLOCK` 是当前 backend 的重复 acquire 和 owner 账本。
- fast path 是弱 relation lock 的 `PGPROC` 快速记录。
- `LOCK` 是 lockable object 的 shared state。
- `PROCLOCK` 是 backend 与 `LOCK` 的 shared 关系。
- `PGPROC.wait*` 字段表示当前等待。
- `ResourceOwner` 让事务/subtransaction cleanup 能找到 `LOCALLOCK`。
cleanup 规则：
- session lock 用 `owner = NULL`。
- transaction lock 用 `CurrentResourceOwner`。
- top-level transaction release 走 `ProcReleaseLocks()`。
- subtransaction commit 走 `LockReassignCurrentOwner()`。
- subtransaction abort 走 `LockReleaseCurrentOwner()`。
- wait 中断由 `LockErrorCleanup()` 收尾。
错误路径规则：
- fast path slot 满则退回 main table。
- shared hash OOM 默认 ERROR，特殊调用可返回 `LOCKACQUIRE_NOT_AVAIL`。
- `dontWait` 必须 undo request counts 和临时 `PROCLOCK`。
- hard deadlock 会 `RemoveFromWaitQueue()` 并 `DeadLockReport()`。
- 强锁 acquire 失败必须 `AbortStrongLockAcquire()`。
- 被授予后发生 ERROR，`GrantAwaitedLock()` 让 ResourceOwner 后续能释放。
观测边界：
- `pg_locks` 看当前 holder/waiter 和 fastpath 状态。
- `pg_blocking_pids()` 能表达 hard blocker 与 soft blocker。
- `pg_stat_activity.wait_event` 看当前等待点。
- `pg_stat_lock` 看累计长等待和 fastpath exceeded。
- `log_lock_waits` 看超过 `deadlock_timeout` 的长等待细节。
- `LOCALLOCK` owner count、strong acquire 短窗口、`lockCleared` 主要靠 gdb 或 instrumentation。
可迁移 mental model：
```text
高频无冲突路径可以被局部化；
但任何可能产生等待、唤醒、死锁、cleanup 或用户观测的状态，
最终必须进入一个共享、可排序、可恢复的事实表。
```
对 PostgreSQL heavyweight lock 来说，这个事实表就是 main lock table。
fast path 是优化，不是语义替代。
`ResourceOwner` 是 cleanup 索引，不是 shared ownership。
wait queue 是授予顺序的一部分，不只是睡眠队列。
线上诊断要同时回答：
```text
这把锁现在在哪里记录？
谁的 LOCALLOCK/ResourceOwner 认为自己持有它？
main lock table 是否已有 holder/waiter 顺序？
ERROR/abort 时哪条 cleanup 路径会收尾？
```
