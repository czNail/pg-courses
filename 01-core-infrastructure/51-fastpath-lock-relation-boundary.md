# PostgreSQL relation fast path lock 与 main lock table 边界
## 课程定位
前置知识：已经理解 `PGPROC`、`LOCALLOCK`、`LOCK`、`PROCLOCK`、ResourceOwner、LWLock 和 heavyweight lock 的基本边界。
本节唯一主问题：
```text
relation lock fast path 如何降低轻量访问成本，什么时候必须转入 main lock table？
```
核心矛盾：
```text
普通 DML 高频拿 relation lock，绝大多数弱锁彼此不冲突；
如果每次都进入 shared lock hash table，会把无冲突访问变成 lock partition LWLock 争用；
但 DDL、显式强锁、deadlock detection、2PC 和 pg_locks 又必须看到完整锁语义。
```
一句话答案：
```text
PostgreSQL 把常见弱 relation lock 临时记录在每个 backend 的 PGPROC fast-path array；
只要出现可能冲突的强 relation lock、slot 容量边界、2PC、等待图或 cleanup 需要 main table 状态，
就把状态迁回 LOCK / PROCLOCK。
```
学完后应能判断：
- 为什么 `AccessShareLock`、`RowShareLock`、`RowExclusiveLock` 可以走 fast path。
- 为什么 `ShareLock` 及以上强锁必须阻止新的 fast path，并迁移已有 fast-path locks。
- 为什么 `ShareUpdateExclusiveLock` 既不走 fast path，也通常不触发 relation fast-path transfer。
- 为什么 `pg_locks.fastpath=true` 只是物理表示，不是不同的锁语义。
- 为什么 slot 满只是 fallback，不是 correctness failure。
- 为什么 deadlock detector 不扫描 fast-path array 仍然成立。
- 为什么 prepared transaction 不能保留 relation fast-path lock。
本课基于本地源码：
```text
/home/nail/postgres
commit 0e1f1ed157e
```
本课只讲 relation fast path boundary。
同一源码里还有 VXID fast path，但 VXID 解决的是 virtual transaction 等待和 deadlock detection 边界，不是本节主问题。
## 1. 本节在总主线中的位置
前面几节已经建立了四个基础边界。
`PGPROC` 是 backend 在 shared memory 中的身份锚点。
`LWLock` 保护 shared memory 数据结构的短临界区。
`ResourceOwner` 兜底释放 lock、pin、snapshot 等外部资源。
heavyweight lock manager 用 `LOCKTAG` 表达逻辑对象互斥关系，并在必要时进入 wait queue 和 deadlock detector。
relation fast path 正好处在这些边界中间。
它仍然属于 heavyweight lock manager。
它没有改变 relation lock 的兼容性规则。
它只改变无冲突弱 relation lock 的物理记录位置。
普通 main lock table 路径是：
```text
LockAcquireExtended()
  -> LockMethodLocalHash
  -> LockHashPartitionLock
  -> LockMethodLockHash
  -> LockMethodProcLockHash
  -> conflict check / wait queue / deadlock detector
```
fast path 成功时是：
```text
LockAcquireExtended()
  -> LockMethodLocalHash
  -> MyProc->fpInfoLock
  -> MyProc->fpRelId[] + MyProc->fpLockBits[]
  -> GrantLockLocal()
```
这是一种 PostgreSQL 常见优化形态：
```text
语义仍由原抽象定义；
常见无冲突路径绕开共享全局结构；
一旦需要全局协调，就迁回原抽象。
```
所以本课不是 fast path 函数清单。
本课只跟一条时间线：
```text
relation lock acquire
  -> 判断是否 eligible
  -> 成功时写 PGPROC fast-path slot
  -> 强锁或边界出现时 transfer 到 main table
  -> release / abort / prepare 按真实物理位置收尾
```
## 2. 核心矛盾与运行模型
弱 relation lock 的典型 workload 是短查询和 DML。
很多 backend 可以同时持有同一个 relation 的 `AccessShareLock`。
`AccessShareLock` 和 `RowExclusiveLock` 这类锁在普通 DML 之间也通常不形成等待。
如果每次都更新 shared lock hash table，就会在同一个 lock hash partition 上制造热点。
但 main lock table 不能被完全绕开。
DDL 需要等待已经存在的弱锁。
显式 `LOCK TABLE` 可能形成冲突。
deadlock detector 只理解 wait queue 和 `PROCLOCK`。
`PREPARE TRANSACTION` 需要把锁持久化到 2PC state。
`pg_locks` 需要能解释当前系统锁状态。
因此运行模型是：
```text
weak relation lock fast path:
  backend-local LOCALLOCK 记录 ownership；
  PGPROC fast-path array 发布轻量 shared state；
  不创建 LOCK / PROCLOCK。
strong relation lock boundary:
  先增加 strong-lock partition counter；
  阻止同 partition 新 weak lock 走 fast path；
  扫描所有 PGPROC；
  把目标 relation 的 fast-path weak locks 转入 LOCK / PROCLOCK。
slow path:
  slot 满、strong counter 非零、transfer、prepare、等待、deadlock detection
  都回到 main lock table。
```
这背后的取舍是：
| 需求 | fast path 的选择 |
| --- | --- |
| 高频弱锁 | 不碰 lock hash partition。 |
| 冲突判断 | 只允许不会彼此等待的弱锁直接进入。 |
| 强锁出现 | 迁移相关 fast-path locks。 |
| deadlock detection | 等待发生前必须进入 main table。 |
| 2PC | PREPARE 前必须拥有 `PROCLOCK`。 |
| 诊断 | `pg_locks` 同时扫描 fast path 和 main table。 |
要带走的抽象是：
```text
fast path 不是独立锁语义；
它是 main abstraction 的缓存化物理表示。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/README` | fast path locking 的动机、strong counter、memory ordering 和 deadlock 边界。 |
| 2 | `src/include/storage/proc.h` | `PGPROC` 的 `fpInfoLock`、`fpLockBits`、`fpRelId`、slot 常量。 |
| 3 | `src/backend/utils/init/postinit.c` | `InitializeFastPathLocks()` 按 `max_locks_per_transaction` 决定 group 数。 |
| 4 | `src/backend/storage/lmgr/proc.c` | fast-path array shared memory sizing 和挂接到每个 `PGPROC`。 |
| 5 | `src/backend/storage/lmgr/lock.c` | eligible、grant、ungrant、transfer、strong counter、fallback 主逻辑。 |
| 6 | `src/include/storage/lock.h` | `LOCALLOCK.holdsStrongLockCount` 和 `LockInstanceData.fastpath`。 |
| 7 | `src/backend/utils/adt/lockfuncs.c` | `pg_locks.fastpath` 输出。 |
| 8 | `src/backend/utils/activity/pgstat_lock.c` | `pg_stat_lock.fastpath_exceeded` 统计。 |
| 9 | `src/test/regress/sql/stats.sql` | partitions 触发 fast path slot exceeded 的回归测试。 |
推荐读法：
```text
先读 lmgr/README 的 Fast Path Locking
  -> 看 proc.h 的 PGPROC fast-path 字段
  -> 看 postinit.c / proc.c 的 slot 数和 shared memory 布局
  -> 精读 LockAcquireExtended() 的 fast path 分支
  -> 再读 FastPathGrantRelationLock() / FastPathTransferRelationLocks()
  -> 最后读 GetLockStatusData() 和 lockfuncs.c
```
不要从所有 `LockRelationOid()` 调用点开始扫。
调用点太多，且业务含义不同。
先建立底层边界：
```text
同一个 relation lock 请求，什么时候只留下 PGPROC fast-path 表示；
什么时候必须拥有 main lock table 中的 LOCK / PROCLOCK。
```
## 4. 关键数据结构与状态
### 4.1 `PGPROC` fast-path fields
`src/include/storage/proc.h` 中的相关字段是：
```text
fpInfoLock:
  保护这个 PGPROC 的 fast-path 状态。
fpLockBits:
  每个 group 一个 uint64，按 slot 存放 lockmode bits。
fpRelId:
  每个 slot 对应一个 relation OID。
fpVXIDLock / fpLocalTransactionId:
  VXID fast path 使用，本课不展开。
```
`fpLockBits` 和 `fpRelId` 不是 `PGPROC` 内嵌固定数组。
`proc.c` 申请 `"Fast-Path Lock Array"` shared memory。
初始化时给每个 `PGPROC` 分配一段：
```text
PGPROC.fpLockBits -> 本 backend 的 bit area
PGPROC.fpRelId    -> 本 backend 的 relid slot area
```
这块状态在 shared memory 中。
其它 backend 可以在持有目标 backend 的 `fpInfoLock` 后扫描它。
所以 fast path 不是完全 backend-local。
更准确地说：
```text
ownership 是当前 backend 的；
可迁移状态发布在 PGPROC 附属 shared memory；
并发访问由每个 PGPROC 自己的 fpInfoLock 串行化。
```
### 4.2 group、slot 与 bit layout
本地源码常量：
```text
FP_LOCK_SLOTS_PER_GROUP = 16
FP_LOCK_GROUPS_PER_BACKEND_MAX = 1024
FAST_PATH_BITS_PER_SLOT = 3
FAST_PATH_LOCKNUMBER_OFFSET = 1
```
每个 group 有 16 个 slot。
每个 slot 存一个 relation OID。
每个 slot 有 3 个 lockmode bit，对应 lock number 1 到 3：
```text
1: AccessShareLock
2: RowShareLock
3: RowExclusiveLock
```
这就是 relation fast path 的物理边界。
一个 slot 可以同时记录同一 relation 上多个弱 lock modes。
但完全相同的 `(locktag, lockmode)` 再次 acquire 不会重新写 slot。
`LockAcquireExtended()` 会先发现 `LOCALLOCK.nLocks > 0`，只增加本地计数。
### 4.3 `FastPathLocalUseCounts`
`lock.c` 中有 backend-private 静态数组：
```text
FastPathLocalUseCounts[group]
```
它记录本 backend 认为某个 group 使用了多少 slot。
源码注释强调：
```text
它可能高于真实值；
但不能低于真实值。
```
原因是本 backend 只能自己获取自己的 fast-path lock。
但其它 backend 可以在强锁 transfer 时清掉本 backend 的 fast-path bit。
本 backend 不一定马上修正本地计数。
这个字段不是 correctness 状态。
它是热路径提示：
```text
count 偏高:
  少尝试 fast path，提前走 main table。
真实空间不足:
  FastPathGrantRelationLock() 扫描后返回 false，再走 main table。
```
### 4.4 `FastPathStrongRelationLocks`
全局 strong-lock 边界由 `FastPathStrongRelationLocks` 表达：
```text
FAST_PATH_STRONG_LOCK_HASH_PARTITIONS = 1024
FastPathStrongRelationLocks:
  mutex
  count[1024]
```
它不是每 relation 精确表。
它是 1024 个 partition counter。
强锁路径 bump 一个 counter。
弱锁 fast path acquire 读取同一个 counter。
语义是：
```text
count[partition] == 0:
  这个 strong partition 当前没有强锁屏障，弱锁可以尝试 fast path。
count[partition] != 0:
  新弱锁不能走 fast path，必须走 main table。
```
它允许 false positive。
无关 relation 如果落到同一个 partition，也可能让弱锁走慢路径。
但它不允许 false negative。
只要可能有强锁扫描窗口，新弱锁不能偷偷进入 fast path。
### 4.5 `LOCALLOCK` 仍然是 ownership 锚点
fast path 没有绕过 `LOCALLOCK`。
`LockAcquireExtended()` 仍然先在 `LockMethodLocalHash` 中创建或找到 `LOCALLOCK`。
fast path 成功后会设置：
```text
locallock->lock = NULL
locallock->proclock = NULL
GrantLockLocal(locallock, owner)
```
这里的 `NULL` 是物理位置标记。
它表示 main lock table 中暂时没有当前 lock 的 `LOCK` / `PROCLOCK` 指针。
它不表示没有锁。
fast-path relation lock 的语义要组合看：
```text
LOCALLOCK.nLocks > 0
  + lock/proclock == NULL
  + EligibleForRelationFastPath(locktag, mode)
  = 当前可能由 relation fast path 表示
```
## 5. 哪些 relation lock 可以走 fast path
`EligibleForRelationFastPath(locktag, mode)` 的条件是：
- lock method 是 `DEFAULT_LOCKMETHOD`。
- locktag type 是 `LOCKTAG_RELATION`。
- locktag database oid 等于 `MyDatabaseId`。
- `MyDatabaseId != InvalidOid`。
- lockmode 小于 `ShareUpdateExclusiveLock`。
也就是当前数据库中的普通 relation 弱锁。
可走 fast path 的常见模式：
```text
AccessShareLock:
  SELECT 读 relation。
RowShareLock:
  SELECT FOR UPDATE/SHARE 等路径可能使用。
RowExclusiveLock:
  INSERT / UPDATE / DELETE 常见 relation lock。
```
不能走 fast path 的常见模式：
```text
ShareUpdateExclusiveLock:
  自冲突，需要 main table 表达同模式冲突；
  但不和前三类弱锁冲突，因此不触发 relation fast-path transfer。
ShareLock
ShareRowExclusiveLock
ExclusiveLock
AccessExclusiveLock:
  会和弱 relation lock 冲突，属于 strong boundary。
shared relation:
  locktag database oid 是 InvalidOid。
非 relation locktag:
  transactionid、tuple、object、advisory 等不属于 relation fast path。
```
`ConflictsWithRelationFastPath(locktag, mode)` 只覆盖：
```text
DEFAULT_LOCKMETHOD
LOCKTAG_RELATION
database oid != InvalidOid
mode > ShareUpdateExclusiveLock
```
所以 `ShareUpdateExclusiveLock` 是边界值。
它自己不走 fast path。
它也不需要驱逐 `AccessShareLock`、`RowShareLock`、`RowExclusiveLock`。
## 6. 主流程源码 walkthrough：弱锁 fast path acquire
### 6.1 入口和本地状态
统一入口是：
```text
LockAcquireExtended(locktag, lockmode, sessionLock, dontWait, ...)
```
入口先校验 lock method 和 mode。
然后决定 owner：
```text
sessionLock:
  owner = NULL
transaction-level lock:
  owner = CurrentResourceOwner
```
接着构造 `LOCALLOCKTAG` 并查 `LockMethodLocalHash`。
如果是新 `LOCALLOCK`，初始化：
```text
lock = NULL
proclock = NULL
hashcode = LockTagHashCode(locktag)
nLocks = 0
holdsStrongLockCount = false
lockCleared = false
lockOwners = TopMemoryContext allocation
```
如果 `locallock->nLocks > 0`，直接：
```text
GrantLockLocal(locallock, owner)
return already-held status
```
这一步说明 fast path 跳过 shared main table，不跳过本地 ownership。
### 6.2 eligibility 和本地 group 容量
只有 `locallock->nLocks == 0` 才进入真实 acquire。
弱锁分支先判断：
```text
EligibleForRelationFastPath(locktag, lockmode)
```
然后判断本地 group 是否看起来未满：
```text
FastPathLocalUseCounts[FAST_PATH_REL_GROUP(relid)] < FP_LOCK_SLOTS_PER_GROUP
```
如果本地估计 group 已满，直接放弃 fast path。
源码会增加：
```text
pgstat_count_lock_fastpath_exceeded(LOCKTAG_RELATION)
```
注意这个统计只覆盖 slot limit reached。
它不统计 strong counter 非零导致的 fallback。
### 6.3 `fpInfoLock` 和 strong counter
如果 group 可能有空间，弱锁 acquire 拿：
```text
LWLockAcquire(&MyProc->fpInfoLock, LW_EXCLUSIVE)
```
然后检查：
```text
fasthashcode = FastPathStrongLockHashPartition(hashcode)
if FastPathStrongRelationLocks->count[fasthashcode] != 0:
  acquired = false
else:
  acquired = FastPathGrantRelationLock(relid, lockmode)
```
为什么必须持有 `fpInfoLock` 后读 strong counter？
`lmgr/README` 给出的关键不变量是：
```text
LWLock acquisition 是 memory sequencing point；
弱锁持有自己的 fpInfoLock 后看到 strong counter 为 0；
如果这个 0 是 stale，则强锁扫描者还没拿到这个 backend 的 fpInfoLock；
它之后扫描时会看到弱锁写入的 fast-path slot。
```
这段 ordering 是 fast path 正确性的核心。
弱锁看 counter。
强锁扫 per-backend array。
双方通过每个 backend 的 `fpInfoLock` 建立同步。
### 6.4 写入 slot
`FastPathGrantRelationLock(relid, lockmode)` 扫描目标 group 的 16 个 slot。
它先找已有 relid slot：
```text
fpRelId[f] == relid:
  设置对应 lockmode bit
  return true
```
如果没有已有 relid slot，但找到空 slot：
```text
fpRelId[unused_slot] = relid
设置 lockmode bit
FastPathLocalUseCounts[group]++
return true
```
如果没有空间：
```text
return false
```
成功时没有创建 `LOCK`。
没有创建 `PROCLOCK`。
没有进入 wait queue。
没有 deadlock detector。
性能收益就来自这里。
### 6.5 返回给调用者
fast path 成功后：
```text
locallock->lock = NULL
locallock->proclock = NULL
GrantLockLocal(locallock, owner)
return LOCKACQUIRE_OK
```
`GrantLockLocal()` 会增加本地计数并把 lock 记入 ResourceOwner。
所以 abort 或 owner release 仍然能找到它。
状态分工是：
```text
ResourceOwner / LOCALLOCK:
  本 backend ownership 和 cleanup。
PGPROC fast-path array:
  其它 backend 必要时能发现和迁移。
main lock table:
  冲突、等待、死锁、2PC 和完整锁图。
```
## 7. 主流程源码 walkthrough：强锁触发 transfer
现在看 `AccessExclusiveLock` 这类强 relation lock。
它满足：
```text
ConflictsWithRelationFastPath(locktag, lockmode)
```
所以进入 main lock table 前，必须先执行 strong-lock 协议。
### 7.1 `BeginStrongLockAcquire()`
强锁路径计算：
```text
fasthashcode = FastPathStrongLockHashPartition(hashcode)
```
然后调用：
```text
BeginStrongLockAcquire(locallock, fasthashcode)
```
它在 spinlock 下做：
```text
FastPathStrongRelationLocks->count[fasthashcode]++
locallock->holdsStrongLockCount = true
StrongLockInProgress = locallock
```
这个 counter 不是强锁授予状态。
它是迁移窗口屏障：
```text
从现在起，同 partition 新 weak lock 不能继续写 fast-path slot；
它们必须走 main lock table。
```
`StrongLockInProgress` 是 ERROR cleanup 锚点。
如果后续 transfer 或 main table acquire 失败，`AbortStrongLockAcquire()` 能回滚 counter。
### 7.2 扫描所有 `PGPROC`
随后调用：
```text
FastPathTransferRelationLocks(lockMethodTable, locktag, hashcode)
```
它遍历：
```text
ProcGlobal->allProcCount
GetPGProcByNumber(i)
```
对每个 `proc` 拿：
```text
LWLockAcquire(&proc->fpInfoLock, LW_EXCLUSIVE)
```
然后过滤：
```text
proc->databaseId != locktag->locktag_field1:
  跳过
proc->fpLockBits[group] == 0:
  跳过
```
再扫描该 group 的 16 个 slot：
```text
proc->fpRelId[f] == relid
  + FAST_PATH_GET_BITS(proc, f) != 0
  = 找到目标 relation 的 fast-path locks
```
prepared transaction 的 dummy `PGPROC` 不在这个扫描范围。
因为 PREPARE 前 fast-path locks 必须已经转入 main table。
### 7.3 建立 `LOCK` / `PROCLOCK`
找到目标 slot 后，transfer 拿目标 locktag 的 partition lock：
```text
partitionLock = LockHashPartitionLock(hashcode)
LWLockAcquire(partitionLock, LW_EXCLUSIVE)
```
对 slot 中每个已设置 lockmode bit：
```text
proclock = SetupLockInTable(lockMethodTable, proc, locktag, hashcode, lockmode)
GrantLock(proclock->tag.myLock, proclock, lockmode)
FAST_PATH_CLEAR_LOCKMODE(proc, f, lockmode)
```
这就是从 PGPROC fast path 到 main table 的迁移。
之后强锁 acquire 进入普通路径：
```text
SetupLockInTable()
LockCheckConflicts()
GrantLock() 或 JoinWaitQueue()
WaitOnLock() if needed
GrantLockLocal()
FinishStrongLockAcquire()
```
deadlock detector 不扫描 fast-path array 的原因也在这里：
```text
凡是可能参与等待图的 relation lock，在等待发生前已经被迁移到 main table。
```
### 7.4 transfer 失败
`FastPathTransferRelationLocks()` 可能因为 shared lock table 内存不足返回 false。
强锁路径会：
```text
AbortStrongLockAcquire()
RemoveLocalLock(locallock) if nLocks == 0
locallockp = NULL
ERROR out of shared memory 或返回 LOCKACQUIRE_NOT_AVAIL
```
这说明 strong counter increment 不是事务锁状态。
它是迁移窗口标记。
中途失败必须回滚，否则同 partition 弱锁会长期失去 fast path。
## 8. fallback 到 main lock table 的边界
relation weak lock 也可能走 main lock table。
主要 fallback 有四类。
第一类是 fast-path group slot 满。
每个 group 只有 16 个 slot。
如果同一 backend 访问很多 relation，且这些 relation hash 到同一 group，就会触发：
```text
FastPathLocalUseCounts[group] >= FP_LOCK_SLOTS_PER_GROUP
```
然后走 main table，并增加 `fastpath_exceeded` 统计。
第二类是 strong counter 非零。
弱锁 eligible，group 也未满，但：
```text
FastPathStrongRelationLocks->count[fasthashcode] != 0
```
这时弱锁不能走 fast path。
它可能真的和目标 relation 强锁相关。
也可能只是同 strong partition 的无关 relation 导致 false positive。
两者都走 main table。
第三类是实际 slot 扫描失败。
`FastPathLocalUseCounts` 只是提示。
真正写入由 `FastPathGrantRelationLock()` 扫描确认。
返回 false 时继续走 main table。
第四类是上层需要 main table 语义。
例如 PREPARE、等待、deadlock detection、强锁冲突、释放时发现已被迁移。
进入 main table 后状态回到标准 heavyweight lock 模型：
```text
LOCK:
  每个 locktag 的共享锁对象。
PROCLOCK:
  每个 backend 对这个 locktag 的持有/等待关系。
LOCALLOCK:
  当前 backend 的本地计数和 owner 映射。
```
fast path 的态度是：
```text
常见无冲突路径尽量便宜；
一旦需要完整语义，不坚持留在快路径。
```
## 9. 生命周期 / ownership / cleanup
### 9.1 创建
slot storage 在 shared memory 初始化阶段创建。
相关路径：
```text
InitializeFastPathLocks()
  -> 根据 max_locks_per_transaction 计算 FastPathLockGroupsPerBackend
ProcGlobalShmemInit()
  -> CalculateFastPathLockShmemSize()
  -> ShmemRequestStruct("Fast-Path Lock Array")
  -> 给每个 PGPROC 设置 fpLockBits / fpRelId 指针
```
默认 `max_locks_per_transaction = 128` 时：
```text
FastPathLockGroupsPerBackend = 8
FP_LOCK_SLOTS_PER_GROUP = 16
每 backend relation fast-path slots = 128
```
但这不是一个完全自由复用的 128 slot 池。
它是 8 个 group，每组 16 个 slot。
单个 group 满会触发 fallback。
### 9.2 持有
逻辑持有者仍然是 backend / transaction / session。
本地 ownership 由这些状态表达：
```text
LOCALLOCK.nLocks
LOCALLOCK.lockOwners
ResourceOwner
```
fast-path shared state 由这些状态表达：
```text
MyProc->fpRelId[f]
MyProc->fpLockBits[group]
```
两者必须组合理解。
只有 `LOCALLOCK` 没有 fast-path bit，可能说明锁已被 transfer。
只有 fast-path bit 没有本地 ownership，对当前 backend 来说就是不完整状态。
### 9.3 正常释放
当 `locallock->nLocks` 递减到 0，释放路径先尝试 fast release：
```text
if EligibleForRelationFastPath(locktag, lockmode)
  && FastPathLocalUseCounts[group] > 0:
    LWLockAcquire(&MyProc->fpInfoLock, LW_EXCLUSIVE)
    released = FastPathUnGrantRelationLock(relid, lockmode)
    LWLockRelease(&MyProc->fpInfoLock)
```
`FastPathUnGrantRelationLock()` 会：
```text
清除匹配 relid + lockmode bit；
重算这个 group 的 FastPathLocalUseCounts[group]；
返回是否真的释放了 fast-path bit。
```
如果成功：
```text
RemoveLocalLock(locallock)
return true
```
### 9.4 释放时已被迁移
释放时可能找不到 fast-path bit。
原因是其它 backend 的强锁已经把它迁移到 main table。
这时释放路径补查：
```text
LockMethodLockHash
LockMethodProcLockHash
```
然后执行普通：
```text
UnGrantLock()
CleanUpLock()
```
这条路径解释了一个重要边界：
```text
locallock->lock/proclock == NULL 只是缺少缓存指针；
不是没有锁。
```
### 9.5 事务结束与 abort
fast path 成功后仍然 `GrantLockLocal()`。
所以 transaction end、abort、ResourceOwner release 会扫描到这个 lock。
`LockReleaseAll()` 遇到 `lock/proclock == NULL` 的 eligible fast-path relation lock，会先尝试 `FastPathUnGrantRelationLock()`。
如果已被迁移，再走 main table 释放。
ResourceOwner 不理解 `fpLockBits` 的内部布局。
它只管理 `LOCALLOCK` ownership。
真正释放时由 lock manager 判断物理位置。
### 9.6 PREPARE TRANSACTION
prepared transaction 不能保留 relation fast-path lock。
原因是 prepared transaction 的锁要持久化到 2PC state，并在 dummy `PGPROC` 上继续表示。
fast-path arrays 属于活 backend runtime 状态，不适合作为 prepared lock 长期表示。
`AtPrepare_Locks()` 中如果发现：
```text
locallock->proclock == NULL
```
会调用：
```text
FastPathGetRelationLockEntry(locallock)
```
它会把 fast-path lock 转成 main table `PROCLOCK`，或补查已经被其它 backend transfer 的 `PROCLOCK`。
## 10. 正确性机制层次
第一层是 heavyweight lock conflict table。
fast path 只是利用前三类弱锁之间不会形成需要等待的冲突。
lock mode 的语义仍然来自 lock method。
第二层是 `fpInfoLock`。
当前 backend acquire/release 自己的 fast-path lock 时拿它。
其它 backend transfer 时也拿目标 backend 的 `fpInfoLock`。
它提供互斥和 memory ordering。
第三层是 strong counter。
`FastPathStrongRelationLocks->count[]` 不表示某个强锁已授予。
它表示某个 strong partition 中存在强锁 acquire/hold 屏障，新弱锁不能继续写 fast-path slot。
第四层是 main lock table。
等待、冲突、死锁检测、2PC 持久化和完整锁图都在那里表达。
deadlock detector 不扫描 fast path，是因为等待发生前相关状态已迁移。
第五层是 ResourceOwner。
它保证 abort cleanup 能找到本 backend 持有的 lock。
但它不负责解释 fast-path bits。
ResourceOwner 和 lock manager 的边界是：
```text
ResourceOwner:
  管 ownership 生命周期。
lock manager:
  管 fast-path bit 或 main table 的实际释放动作。
```
## 11. 错误路径 / 异常路径
### 11.1 shared lock table OOM
fast path 写预分配 slot，不为 `LOCK` / `PROCLOCK` 分配 shared hash entry。
但 fallback 和 transfer 需要 main table entry。
如果 `SetupLockInTable()` 返回 NULL，路径会：
```text
AbortStrongLockAcquire()
LWLockRelease(partitionLock)
RemoveLocalLock() if needed
ERROR out of shared memory
```
错误 hint 通常指向 `max_locks_per_transaction`。
这说明问题发生在 main lock table capacity，不是 fast-path bit 本身。
### 11.2 strong acquire 中途失败
`BeginStrongLockAcquire()` 之后必须保证失败时 counter 回滚。
源码用：
```text
locallock->holdsStrongLockCount = true
StrongLockInProgress = locallock
```
兜底。
失败时 `AbortStrongLockAcquire()` decrement counter。
main table 状态完全建立后才 `FinishStrongLockAcquire()`。
### 11.3 dontWait / deadlock
main table 路径中，如果 `dontWait` 不能等待，或 join wait queue 前发现错误，会撤销未完成状态：
```text
AbortStrongLockAcquire()
删除未持有的 PROCLOCK
回退 LOCK requested counters
RemoveLocalLock() if nLocks == 0
return LOCKACQUIRE_NOT_AVAIL
```
如果进入等待后 deadlock detector 报错，`DeadLockReport()` 不返回。
此时相关锁状态已经在 main table。
### 11.4 `pg_locks` 快照不完全一致
`GetLockStatusData()` 先逐个 backend 扫 fast-path arrays。
它一次只拿一个 `proc->fpInfoLock`。
随后再拿所有 lock hash partition 的 shared LWLock，复制 main table。
源码承认 fast-path 部分可能不是全系统完全一致的瞬时图。
这通常可接受。
因为会形成阻塞关系的状态必须在 main table。
### 11.5 本 backend 滞后感知 transfer
其它 backend 可以把当前 backend 的 fast-path lock 迁移到 main table。
当前 backend 的 `FastPathLocalUseCounts` 可能仍偏高。
当前 backend 的 `LOCALLOCK.lock/proclock` 可能仍是 NULL。
这不是损坏。
释放或 prepare 时会补查 main table。
## 12. 成本、资源与跨模块传播
fast path 省掉的成本主要是：
```text
LockHashPartitionLock；
LockMethodLockHash 查找/创建；
LockMethodProcLockHash 查找/创建；
main table conflict scan；
wait queue 相关状态维护。
```
它仍然要付出：
```text
LOCALLOCK hash；
MyProc->fpInfoLock；
strong counter 读；
group 内最多 16 slot 扫描。
```
所以它不是零成本。
它把高争用 shared hash 成本替换成 per-backend 小数组成本。
成本随 backend 数扩张时，弱锁 acquire 仍然接近小常数。
但强锁 transfer 会扫描 `ProcGlobal->allProcCount`。
这说明系统把便宜留给高频 DML，把偶发成本留给 DDL / explicit strong lock。
成本随 relation 数扩张时，slot 压力会上升。
典型场景是大量 partitions、复杂 view、继承树、catalog 访问和单事务访问许多 relation。
源码回归测试用 partition 表触发 `fastpath_exceeded`。
strong counter 的 1024 partition 设计允许 false positive。
无关 relation 的强锁可能让同 partition 弱锁走慢路径。
这是保守近似：
```text
允许误判为慢路径；
不允许误判为安全快路径。
```
shared memory 成本随这些变量扩张：
```text
TotalProcs
FastPathLockGroupsPerBackend
FP_LOCK_SLOTS_PER_GROUP
```
`FastPathLockGroupsPerBackend` 根据 `max_locks_per_transaction` 计算，最高 cap 到 1024。
默认 `max_locks_per_transaction=128` 时是 8 groups。
跨模块边界：
- ResourceOwner 管 lock ownership release，不直接管理 fast-path bits。
- `PGPROC` 发布 backend identity 和 fast-path shared state。
- `ProcGlobal->allProcCount` 决定强锁 transfer 扫描范围。
- `LWLock` 中 `fpInfoLock` 保护 per-backend fast-path state，`LockHashPartitionLock` 保护 main table。
- stats 只记录 `fastpath_exceeded`，不记录 strong counter fallback 或 transfer 次数。
- 2PC 要求 PREPARE 前把 fast-path lock 转入 main table。
## 13. 观测与诊断入口
### 13.1 `pg_locks.fastpath`
`pg_locks` 的 `fastpath` 列来自：
```text
lockfuncs.c
  -> pg_lock_status()
  -> GetLockStatusData()
  -> LockInstanceData.fastpath
```
`GetLockStatusData()` 先扫描 fast-path arrays。
对 relation fast-path slot，它构造：
```text
LOCKTAG_RELATION(databaseId, relid)
holdMask = lockbits << FAST_PATH_LOCKNUMBER_OFFSET
waitLockMode = NoLock
fastpath = true
waitStart = 0
```
随后扫描 main lock table。
main table 复制出的 entry：
```text
fastpath = false
```
所以 `pg_locks.fastpath=true` 只表示：
```text
当前采样来自 PGPROC fast-path array。
```
它不表示锁更弱、不受事务控制、不会被强锁等待或不需要释放。
### 13.2 `pg_stat_lock.fastpath_exceeded`
`pg_stat_lock.fastpath_exceeded` 来自 `pgstat_count_lock_fastpath_exceeded()`。
源码注释说明它只统计：
```text
fast-path slot limit reached
```
它能说明某类 locktag 因 slot limit 进入慢路径的次数增加。
它不能说明 strong counter fallback、transfer 次数、具体 relation、具体 query 或是否发生等待。
### 13.3 wait event
fast-path 成功的弱 relation lock 不会等待 main table。
如果发生 `wait_event_type = 'Lock'`，说明已经进入 main table。
可查：
```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```
在 `pg_locks` 中，等待行通常是：
```text
granted = false
fastpath = false
```
因为 wait queue 属于 main table。
### 13.4 gdb / instrumentation 入口
建议断点：
```text
LockAcquireExtended
FastPathGrantRelationLock
BeginStrongLockAcquire
FastPathTransferRelationLocks
FastPathUnGrantRelationLock
FastPathGetRelationLockEntry
GetLockStatusData
```
建议观察：
```text
locktag->locktag_field1
locktag->locktag_field2
lockmode
locallock->lock
locallock->proclock
FastPathLocalUseCounts[group]
MyProc->fpRelId[f]
FAST_PATH_GET_BITS(MyProc, f)
FastPathStrongRelationLocks->count[fasthashcode]
```
### 13.5 可见性边界
能直接看到：
- `pg_locks.fastpath`
- `pg_locks.granted`
- `pg_locks.mode`
- `pg_stat_activity.wait_event`
- `pg_stat_lock.fastpath_exceeded`
只能推断：
- 某次 fallback 是否由 strong counter 非零导致。
- 某个 `fastpath=false` 弱锁是否曾经从 fast path transfer 而来。
- strong counter false positive 的影响。
- group 内 slot 分布。
普通 SQL 看不到：
- `fpLockBits` 原始 bit。
- `FastPathLocalUseCounts`。
- `FastPathStrongRelationLocks->count[]`。
- transfer 次数。
- 强锁扫描了多少 `PGPROC`。
这些需要源码日志、gdb、perf 或临时 instrumentation。
## 14. 常见误区
误区一：`fastpath=true` 的 lock 不是真正的 heavyweight lock。
正确理解：它仍然是 heavyweight relation lock 语义，只是当前物理表示在 `PGPROC` fast-path array。
误区二：fast path 完全 backend-local。
正确理解：`LOCALLOCK` 是 backend-local，但 `fpRelId` / `fpLockBits` 在 shared memory 中，其它 backend 可持有 `fpInfoLock` 扫描。
误区三：`fastpath=false` 说明这把锁一定没有走过 fast path。
正确理解：它只说明当前采样来自 main table；它可能一开始就在 main table，也可能后来被 transfer。
误区四：默认 128 slots 是线性池。
正确理解：默认是 8 groups * 16 slots，单个 group 满就 fallback。
误区五：strong counter 精确到 relation。
正确理解：它是 1024-way partition counter，允许无关 relation 的 false positive slow path。
误区六：deadlock detector 漏扫 fast path 是缺陷。
正确理解：可能参与等待图的 relation lock 在等待前已迁移到 main table。
## 15. 课堂实验
### 实验 1：观察弱锁从 fast path 到 main table
Session A：
```sql
CREATE TABLE fp_demo(id int);
BEGIN;
SELECT * FROM fp_demo;
```
Session C：
```sql
SELECT pid, mode, granted, fastpath, relation::regclass
FROM pg_locks
WHERE relation = 'fp_demo'::regclass
ORDER BY pid, mode;
```
常见现象：
```text
Session A 的 AccessShareLock 可能 fastpath=true。
```
Session B：
```sql
BEGIN;
LOCK TABLE fp_demo IN ACCESS EXCLUSIVE MODE;
```
Session C 再查：
```sql
SELECT pid, mode, granted, fastpath, relation::regclass
FROM pg_locks
WHERE relation = 'fp_demo'::regclass
ORDER BY granted DESC, mode;
```
预期解释：
```text
Session B 的强锁触发 FastPathTransferRelationLocks()；
Session A 的 AccessShareLock 被迁移到 main table；
Session B 在 main table wait queue 中等待。
```
源码回看：
```text
LockAcquireExtended()
  -> ConflictsWithRelationFastPath()
  -> BeginStrongLockAcquire()
  -> FastPathTransferRelationLocks()
```
### 实验 2：用 partitions 触发 `fastpath_exceeded`
目标是看到访问大量 relation 导致 fast-path slot capacity fallback。
可参考本地源码：
```text
src/test/regress/sql/stats.sql
```
核心形态：
```sql
CREATE TABLE part_test (id int) PARTITION BY RANGE (id);
SELECT pg_stat_reset_shared('lock');
-- 创建数量超过 max_locks_per_transaction 的 partitions
SELECT count(*) FROM part_test;
SELECT pg_stat_force_next_flush();
SELECT fastpath_exceeded
FROM pg_stat_lock
WHERE locktype = 'relation';
```
解释：
```text
查询需要对许多 partitions 拿 relation lock；
超过 fast-path slot 边界后，部分 relation lock 走 main lock table；
pg_stat_lock.fastpath_exceeded 增加。
```
### 实验 3：断点画状态转移
断点：
```text
break FastPathGrantRelationLock
break BeginStrongLockAcquire
break FastPathTransferRelationLocks
break FastPathUnGrantRelationLock
break FastPathGetRelationLockEntry
```
需要画出的状态：
```text
LOCALLOCK:
  nLocks 0 -> 1
  lock/proclock NULL
PGPROC fast path:
  fpRelId slot empty -> relid
  lockmode bit 0 -> 1
strong transfer:
  strong counter 0 -> 1
  fast-path bit 1 -> 0
  LOCK/PROCLOCK absent -> present
release:
  fast-path bit cleared 或 main table counters decremented
  LOCALLOCK removed
```
## 16. 讨论题
1. 为什么 `AccessShareLock` 可以走 relation fast path，而 `AccessExclusiveLock` 不可以？
2. `ShareUpdateExclusiveLock` 为什么不走 fast path，但也不触发 `ConflictsWithRelationFastPath()`？
3. 如果 strong counter 精确到 relation，会减少哪些 false positive？又会增加哪些热路径成本？
4. 为什么强锁获取前要先增加 strong counter，再扫描所有 `PGPROC` fast-path arrays？
5. `LOCALLOCK.lock == NULL` 在 fast path 语境下表示什么？为什么不能据此判断锁不存在？
6. 为什么 deadlock detector 不需要理解 `fpRelId` 和 `fpLockBits`？
7. `pg_stat_lock.fastpath_exceeded` 增加能证明什么？不能证明什么？
8. prepared transaction 为什么必须把 relation fast-path lock 转入 main lock table？
## 17. 本节小结
本节核心链路：
```text
weak relation lock acquire
  -> LOCALLOCK 记录本地 ownership
  -> eligible、slot 未满、strong counter 为 0
  -> 写 MyProc->fpRelId / fpLockBits
  -> 不创建 LOCK / PROCLOCK
strong relation lock acquire
  -> strong counter++
  -> 扫描所有 PGPROC fast-path arrays
  -> 把目标 relation 的 fast-path weak locks 转入 main lock table
  -> 再进入普通 conflict / wait / deadlock 路径
```
核心状态：
```text
PGPROC fast-path array:
  shared memory 中的轻量物理表示。
LOCALLOCK / ResourceOwner:
  本 backend ownership 和 cleanup 锚点。
FastPathStrongRelationLocks:
  coarse-grained strong-lock 屏障，允许 false positive fallback。
main lock table:
  冲突、等待、deadlock detection、2PC 和完整诊断图。
```
cleanup 规则：
```text
正常释放先尝试 FastPathUnGrantRelationLock()；
如果已被迁移，就补查 main table 后释放；
abort / transaction end 通过 ResourceOwner / LockReleaseAll() 兜底；
PREPARE 前必须 FastPathGetRelationLockEntry()。
```
观测边界：
```text
pg_locks.fastpath=true:
  当前采样来自 PGPROC fast-path array。
pg_locks.fastpath=false:
  当前采样来自 main lock table，不保留是否曾经 fast path 的历史。
pg_stat_lock.fastpath_exceeded:
  只统计 slot limit exceeded。
wait_event='Lock':
  说明等待已经进入 main lock table。
```
可迁移系统规律：
```text
高频无冲突路径可以使用更便宜的局部物理表示；
但这种表示必须有明确迁移边界；
一旦系统需要全局排序、等待图、持久化或诊断一致性，就必须回到完整抽象。
```
本节判断仍然依赖 workload、硬件和版本。
slot 数受 `max_locks_per_transaction` 和当前源码实现影响。
收益取决于弱锁和强锁比例、backend 数、relation 数、partition 数、DDL 频率和 CPU cache contention。
不要把某一次 `fastpath=true` 或 `fastpath_exceeded` 计数变化解释成完整因果。
正确诊断要把 SQL 形态、relation 数、锁模式、`pg_locks` 快照、wait event 和源码边界放在同一条链路里。
