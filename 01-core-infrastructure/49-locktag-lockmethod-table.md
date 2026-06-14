# PostgreSQL LOCKTAG 与 lock method table
## 课程定位
前置知识：已经理解 `PGPROC` / `ProcArray`、`ResourceOwner`、`LWLock`、latch、condition variable、事务提交/回滚，以及 relation / tuple / XID 的基本运行含义。
本节唯一主问题：
```text
LOCKTAG、lock method table 和 lock mode 如何把 relation、page、tuple、transactionid、advisory lock 映射到统一锁管理器？
```
核心矛盾：
```text
上层需要锁住很多不同对象：relation、page、tuple、transactionid、virtualxid、catalog object、advisory key。
底层又不能为每类对象复制一套 shared hash table、wait queue、deadlock detector、ResourceOwner cleanup 和 pg_locks 视图。
```
PostgreSQL 的解法是三层分工：
```text
LOCKTAG:
  统一表达“锁住哪个对象”。
lock method table:
  统一表达“这类锁方法有哪些 mode 以及 mode 冲突规则”。
lock mode:
  在同一个对象上表达请求强度，例如 AccessShareLock、ShareLock、ExclusiveLock。
```
学完后应能判断：
```text
为什么 relation lock 和 advisory lock 能进入同一套 lock table；
为什么 locktype 是对象类型，而 mode 才是锁强度；
为什么 transactionid lock 等待的是事务结束，而不是某个 heap tuple；
为什么 advisory lock 显示在 pg_locks 的 classid / objid / objsubid 列；
为什么 object lock 不会自动和 relation lock 冲突；
为什么 deadlock detector 不需要理解 relation/page/tuple/advisory 的业务语义；
为什么 ERROR cleanup 必须同时处理 PGPROC wait state、LOCALLOCK 和 shared PROCLOCK。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面几组基础设施课回答的是：
```text
shared memory 如何建立；
backend 如何发布 PGPROC 状态；
ResourceOwner 如何兜底释放外部资源；
LWLock / latch / condition variable 如何保护和等待共享状态。
```
重量级锁管理器把这些能力组合到 SQL 可感知对象上。
它保护的不是某个 C 结构体字段。
它保护的是逻辑对象上的并发语义：
```text
DDL 不能和普通查询随意并发；
一个事务结束前，别人不能安全判断它留下的 tuple 结果；
唯一约束 speculative insertion 决策前，别人需要等待；
用户可以用 advisory key 构造应用级互斥。
```
这些对象的 identity 来源完全不同。
relation 来自 `database OID + relation OID`。
page 来自 `database OID + relation OID + block number`。
tuple 来自 `database OID + relation OID + block number + offset number`。
transactionid 来自 `TransactionId`。
advisory lock 来自用户传入的 `int8` 或两个 `int4`。
如果每类对象单独实现锁表，死锁检测和 cleanup 会碎裂。
如果所有对象压成无类型整数，又无法诊断，也无法避免不同 namespace 误撞。
`LOCKTAG` 正是中间层：
```text
它保留对象类型；
它把不同对象压进固定 16 字节 key；
它让底层只按 key、method、mode 和 conflict bitmask 运行。
```
因此本节连接三个边界：
```text
上层 wrapper:
  把业务对象编码成 LOCKTAG。
通用 lock.c:
  用 method table 和 mode 执行统一锁状态机。
观测层 lockfuncs.c:
  把 LOCKTAG 再解码回 pg_locks 列。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
lmgr.c 或 lockfuncs.c 用 SET_LOCKTAG_* 构造 LOCKTAG；
LOCKTAG 自带 locktag_type 和 locktag_lockmethodid；
LockAcquireExtended() 根据 lockmethodid 取得 LockMethodData；
用 LOCKTAG hash 定位 shared LOCK；
用 LOCK + PGPROC 定位 PROCLOCK；
用 lock mode 查 conflictTab 判断兼容性；
不兼容时把 PGPROC 挂到 LOCK.waitProcs；
释放时按同一张 conflict table 唤醒可授予的等待者；
pg_locks 再按 locktag_type 把 field1..field4 解释成 SQL 列。
```
这条链路有四个不变量。
第一，`LOCKTAG` 定义对象身份。
第二，`LOCKMODE` 定义请求强度。
第三，`LockMethodData` 定义 mode 之间的冲突规则。
第四，`LOCK` / `PROCLOCK` / `LOCALLOCK` 定义 runtime ownership。
不要把这些层混在一起。
`LOCKTAG_RELATION` 不是 `AccessShareLock`。
`AccessShareLock` 也不是 relation 专用。
同名 lock mode 可用于 relation、transactionid、advisory lock。
真正差异来自上层如何解释对象，以及调用点选择哪个 mode。
当前源码中有两个 lock method：
```text
DEFAULT_LOCKMETHOD = 1
USER_LOCKMETHOD    = 2
```
二者当前使用同一套标准 mode 和同一张 `LockConflicts[]`。
但 `locktag_lockmethodid` 仍是 `LOCKTAG` 的一部分。
所以 DEFAULT 和 USER 即使 field 值相同，也不是同一个 lockable object。
这个设计接受的复杂性是：
```text
LOCKTAG 有类型；
field1..field4 的含义随 locktag_type 改变；
pg_locks 和 DescribeLockTag() 必须按 type 反向解释。
```
它换来的收益是：
```text
一套 shared hash table；
一套 wait queue；
一套 ResourceOwner cleanup；
一套 deadlock detector；
一套 SQL 观测入口。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/locktag.h` | `LOCKTAG` 布局、`LockTagType`、`DEFAULT_LOCKMETHOD`、`USER_LOCKMETHOD`、`SET_LOCKTAG_*`。 |
| 2 | `src/include/storage/lockdefs.h` | `LOCKMODE` 编号，`AccessShareLock` 到 `AccessExclusiveLock`。 |
| 3 | `src/include/storage/lock.h` | `LockMethodData`、`LOCK`、`PROCLOCK`、`LOCALLOCK`、`LockInstanceData`。 |
| 4 | `src/backend/storage/lmgr/lock.c` | `LockConflicts[]`、method table、`LockAcquireExtended()`、`LockRelease()`、`GetLockStatusData()`。 |
| 5 | `src/backend/storage/lmgr/lmgr.c` | relation/page/tuple/xid/object wrapper 如何构造 `LOCKTAG`。 |
| 6 | `src/backend/storage/lmgr/proc.c` | `JoinWaitQueue()`、`ProcSleep()`、`ProcLockWakeup()`、`LockErrorCleanup()`。 |
| 7 | `src/backend/storage/lmgr/deadlock.c` | deadlock detector 如何只依赖 `LOCK`、`PGPROC`、wait queue 和 conflict edge。 |
| 8 | `src/backend/utils/adt/lockfuncs.c` | `pg_lock_status()`、`pg_blocking_pids()`、advisory lock SQL 函数。 |
| 9 | `src/backend/utils/activity/pgstat_lock.c` | `pg_stat_lock` 的 waits、wait_time、fastpath_exceeded。 |
| 10 | `src/backend/storage/lmgr/README` | regular lock manager、fast path、wait queue、deadlock detector 的设计说明。 |
推荐阅读顺序：
```text
locktag.h
  -> lockdefs.h
  -> lock.h
  -> lock.c 顶部 conflict table
  -> lmgr.c 的 SET_LOCKTAG_* wrapper
  -> LockAcquireExtended() 的 method/table/conflict 主链路
  -> LockRelease() / LockReleaseAll() cleanup
  -> lockfuncs.c 的 pg_locks 投影
  -> proc.c / deadlock.c 的等待图边界
```
## 4. 关键数据结构与状态
### 4.1 `LOCKTAG`：16 字节 typed key
`LOCKTAG` 在 `locktag.h` 中被刻意设计成 16 字节且无 padding。
字段是：
```text
locktag_field1       uint32
locktag_field2       uint32
locktag_field3       uint32
locktag_field4       uint16
locktag_type         uint8
locktag_lockmethodid uint8
```
`field1` 到 `field4` 没有独立语义。
它们必须和 `locktag_type` 一起解释。
典型映射：
| type | field1 | field2 | field3 | field4 | method |
| --- | --- | --- | --- | --- | --- |
| `LOCKTAG_RELATION` | database OID | relation OID | 0 | 0 | DEFAULT |
| `LOCKTAG_RELATION_EXTEND` | database OID | relation OID | 0 | 0 | DEFAULT |
| `LOCKTAG_DATABASE_FROZEN_IDS` | database OID | 0 | 0 | 0 | DEFAULT |
| `LOCKTAG_PAGE` | database OID | relation OID | block number | 0 | DEFAULT |
| `LOCKTAG_TUPLE` | database OID | relation OID | block number | offset number | DEFAULT |
| `LOCKTAG_TRANSACTION` | transaction ID | 0 | 0 | 0 | DEFAULT |
| `LOCKTAG_VIRTUALTRANSACTION` | proc number | local transaction ID | 0 | 0 | DEFAULT |
| `LOCKTAG_SPECULATIVE_TOKEN` | transaction ID | token | 0 | 0 | DEFAULT |
| `LOCKTAG_OBJECT` | database OID | class OID | object OID | sub ID | DEFAULT |
| `LOCKTAG_ADVISORY` | database OID | key part 1 | key part 2 | key type | USER |
| `LOCKTAG_APPLY_TRANSACTION` | database OID | subscription OID | transaction ID | object ID | DEFAULT |
因此：
```text
LOCKTAG_RELATION.field1       = database OID
LOCKTAG_TRANSACTION.field1    = transaction ID
LOCKTAG_ADVISORY.field1       = database OID
```
只打印 field 值而不打印 type，会丢失语义。
`locktag_lockmethodid` 也是 key 的一部分。
这让 advisory lock 和内部 lock 分离在不同 method namespace。
### 4.2 `LockTagType`：对象类型边界
当前枚举包括 relation、extend、frozenid、page、tuple、transactionid、virtualxid、spectoken、object、userlock、advisory、applytransaction。
`lockfuncs.c` 中的 `LockTagTypeNames[]` 必须和枚举顺序一致。
源码用 `StaticAssertDecl` 检查数组长度。
新增一个 lock tag type 时，至少要同步：
```text
locktag.h 的 enum；
lockfuncs.c 的 LockTagTypeNames[]；
DescribeLockTag()；
pg_locks 文档；
wait event LOCK 名称；
必要的 SET_LOCKTAG_* 宏和上层 wrapper。
```
这说明 `LOCKTAG` 不是私有小结构。
它跨越 runtime、诊断、错误报告和文档。
### 4.3 `LOCKMODE`：请求强度
`lockdefs.h` 定义标准 mode：
```text
NoLock = 0
AccessShareLock = 1
RowShareLock = 2
RowExclusiveLock = 3
ShareUpdateExclusiveLock = 4
ShareLock = 5
ShareRowExclusiveLock = 6
ExclusiveLock = 7
AccessExclusiveLock = 8
```
`NoLock` 不是有效锁模式。
它是“不请求锁”的标记。
`LOCKMASK` 用 `1 << lockmode` 表达 mode 集合。
判断冲突时不是简单比较数值大小。
真正规则来自：
```text
lockMethodTable->conflictTab[requested_mode]
```
### 4.4 `LockMethodData`：冲突规则表
`LockMethodData` 包含：
```text
numLockModes
conflictTab
lockModeNames
trace_flag
```
`lock.c` 顶部的 `LockConflicts[]` 是标准冲突矩阵。
例子：
```text
AccessShareLock:
  只与 AccessExclusiveLock 冲突。
RowExclusiveLock:
  与 ShareLock、ShareRowExclusiveLock、ExclusiveLock、AccessExclusiveLock 冲突。
AccessExclusiveLock:
  与所有标准 mode 冲突，包括自己。
```
当前 `default_lockmethod` 和 `user_lockmethod` 都引用 `LockConflicts[]`。
抽象边界是：
```text
LOCKTAG 指向 lock method；
lock method 指向 conflict table；
LockAcquireExtended() 不硬编码某个对象的冲突矩阵。
```
### 4.5 `LOCK`：每个 lockable object 的 shared state
`LOCK` 的 key 是 `LOCKTAG`。
它保存在 shared lock hash 中。
关键字段：
```text
grantMask:
  当前已授予的 mode 集合。
waitMask:
  当前有人等待的 mode 集合。
procLocks:
  所有相关 PROCLOCK。
waitProcs:
  正在等待这个 LOCK 的 PGPROC 队列。
requested[] / nRequested:
  已请求数量，包含 granted 和 waiting。
granted[] / nGranted:
  已授予数量。
```
这些计数按 backend / proc 粒度统计。
同一 backend 多次重入同一个 object/mode，不会在 shared `granted[]` 中重复很多次。
重入计数在 `LOCALLOCK`。
### 4.6 `PROCLOCK`：backend 对 object 的 shared interest
`PROCLOCKTAG` 是：
```text
LOCK *myLock
PGPROC *myProc
```
这里使用指针作为 key 是允许的。
因为 `PROCLOCK` 不会比对应 `LOCK` 或 `PGPROC` 活得更久。
`PROCLOCK` 关键字段：
```text
groupLeader:
  parallel lock group 代表。
holdMask:
  这个 proc 已持有的 mode 集合。
releaseMask:
  LockReleaseAll() 的工作区。
lockLink:
  挂到 LOCK.procLocks。
procLink:
  挂到 PGPROC.myProcLocks[partition]。
```
这提供两个方向：
```text
从 LOCK 找所有 holder / waiter；
从 PGPROC 找本 backend 需要释放的所有锁。
```
`holdMask == 0` 的 `PROCLOCK` 也可能存在。
典型情况是这个 backend 正在等待，或失败路径尚未 cleanup。
### 4.7 `LOCALLOCK`：backend-local owner 和重入计数
每个 backend 有 `LockMethodLocalHash`。
key 是：
```text
LOCKTAG + LOCKMODE
```
关键字段：
```text
hashcode:
  LOCKTAG hash 缓存。
lock / proclock:
  shared LOCK / PROCLOCK 指针；fast path 时可能为 NULL。
nLocks:
  当前 backend 的总持有次数。
lockOwners:
  每个 ResourceOwner 或 session owner 的持有次数。
holdsStrongLockCount:
  strong relation lock fast path gate 是否已增加。
lockCleared:
  relation/object lock 相关 sinval 是否已吸收。
```
shared 层回答：
```text
其它 backend 是否应该被阻塞？
```
local 层回答：
```text
当前 backend 还持有几次？
属于 session 还是事务 ResourceOwner？
ERROR / abort 时释放哪些？
```
当 `nLocks == 0` 时，`LOCALLOCK.lock` 和 `LOCALLOCK.proclock` 指针可能不可信。
源码专门警告失败的获取尝试可能留下这样的 local entry。
### 4.8 `LockInstanceData`：观测快照
`LockInstanceData` 是 `GetLockStatusData()` 复制给 `lockfuncs.c` 的结构。
它包含：
```text
LOCKTAG locktag
LOCKMASK holdMask
LOCKMODE waitLockMode
VirtualTransactionId vxid
TimestampTz waitStart
pid
leaderPid
fastpath
```
它不是 live pointer。
`pg_locks` 格式化的是这份快照。
因此 `pg_locks` 适合诊断当前近似状态，不适合当完整历史。
## 5. 主流程源码 walkthrough
### 5.1 relation：上层只构造 tag
典型入口：
```text
LockRelationOid(relid, lockmode)
  -> SetLocktagRelationOid(&tag, relid)
     -> shared relation 使用 InvalidOid 作为 dbid
     -> 普通 relation 使用 MyDatabaseId
     -> SET_LOCKTAG_RELATION(tag, dbid, relid)
  -> LockAcquireExtended(&tag, lockmode, false, false, true, &locallock, false)
  -> AcceptInvalidationMessages()
  -> MarkLockClear(locallock)
```
relation wrapper 没有自己的等待队列。
它只负责把对象身份放进 `LOCKTAG_RELATION`。
`LockRelationId()` 从 `LockRelId` 取 dbId / relId。
`LockRelation()` 从 `Relation.rd_lockInfo.lockRelId` 取 identity。
`RelationInitLockInfo()` 在 relcache descriptor 创建时填 `rd_lockInfo`。
这就是 relcache 与 lock manager 的连接点。
### 5.2 page 与 tuple：更细粒度 tag，同一状态机
page lock：
```text
LockPage(relation, blkno, mode)
  -> SET_LOCKTAG_PAGE(dbid, relid, blkno)
  -> LockAcquire()
```
tuple lock：
```text
LockTuple(relation, tid, mode)
  -> SET_LOCKTAG_TUPLE(dbid, relid, block, offset)
  -> LockAcquire()
```
`lmgr.h` 注释提醒：
```text
LockTuple: see heap_lock_tuple before assuming you understand this.
```
原因是 tuple 并发控制不只靠 heavyweight tuple lock。
MVCC tuple header、MultiXact、buffer pin、transactionid wait 和 tuple lock tag 是不同层。
`LOCKTAG_TUPLE` 只是在需要时创建一个 heavyweight lock object。
当 `nRequested == 0`，shared `LOCK` 可以被回收。
### 5.3 transactionid：等待事务结束
事务拿到 XID 后插入事务锁：
```text
XactLockTableInsert(xid)
  -> SET_LOCKTAG_TRANSACTION(tag, xid)
  -> LockAcquire(&tag, ExclusiveLock, false, false)
```
等待某事务结束：
```text
XactLockTableWait(xid, rel, ctid, oper)
  -> SET_LOCKTAG_TRANSACTION(tag, xid)
  -> LockAcquire(&tag, ShareLock, false, false)
  -> LockRelease(&tag, ShareLock, false)
  -> TransactionIdIsInProgress(xid) 检查
  -> 必要时追到 topmost transaction 再等
```
这解释了很多行更新冲突的现象：
```text
locktype = transactionid
mode = ShareLock
granted = false
```
等待者不是直接等 heap tuple。
它在等持有 `transactionid ExclusiveLock` 的事务结束。
### 5.4 virtualxid：等待 backend 当前顶层事务
`VirtualTransactionId` 是：
```text
procNumber + localTransactionId
```
它是短期身份，不能长期写盘。
`LOCKTAG_VIRTUALTRANSACTION` 把两者放入 field1 / field2。
`WaitForLockersMultiple()` 的模式是：
```text
先对目标 LOCKTAG 调 GetLockConflicts()；
得到当前冲突 holder 的 VXID 列表；
再逐个 VirtualXactLock(vxid, true) 等 holder 结束。
```
它不直接获取目标对象 lock。
它只等待采样时已经存在的 holder。
之后新来的冲突 holder 不会自动被包含。
### 5.5 advisory lock：用户 key 进入 USER method
`lockfuncs.c` 中 advisory 函数构造 tag。
`int8` key：
```text
SET_LOCKTAG_INT64(tag, key64)
  -> SET_LOCKTAG_ADVISORY(MyDatabaseId,
                          high 32 bits,
                          low 32 bits,
                          1)
```
两个 `int4` key：
```text
SET_LOCKTAG_INT32(tag, key1, key2)
  -> SET_LOCKTAG_ADVISORY(MyDatabaseId, key1, key2, 2)
```
`field4` 区分 key 形式。
这避免 `int8` 和 `(int4, int4)` namespace 混撞。
session-level exclusive advisory lock：
```text
pg_advisory_lock_int8()
  -> SET_LOCKTAG_INT64()
  -> LockAcquire(&tag, ExclusiveLock, true, false)
```
transaction-level advisory lock：
```text
pg_advisory_xact_lock_int8()
  -> SET_LOCKTAG_INT64()
  -> LockAcquire(&tag, ExclusiveLock, false, false)
```
try-lock：
```text
pg_try_advisory_lock_int8()
  -> LockAcquire(&tag, ExclusiveLock, true, true)
  -> res != LOCKACQUIRE_NOT_AVAIL
```
`LOCKTAG_ADVISORY` 使用 `USER_LOCKMETHOD`。
### 5.6 object lock：不是 relation lock 的别名
database-local object：
```text
LockDatabaseObject(classid, objid, objsubid, mode)
  -> SET_LOCKTAG_OBJECT(MyDatabaseId, classid, objid, objsubid)
  -> LockAcquire()
  -> AcceptInvalidationMessages()
```
shared object：
```text
LockSharedObject(classid, objid, objsubid, mode)
  -> SET_LOCKTAG_OBJECT(InvalidOid, classid, objid, objsubid)
  -> LockAcquire()
  -> AcceptInvalidationMessages()
```
源码注释明确说，不要用它锁 relation。
原因是：
```text
LOCKTAG_OBJECT(dbid, pg_class_oid, relid, 0)
LOCKTAG_RELATION(dbid, relid, 0, 0)
```
这是两个不同 key。
底层不会根据 catalog 含义把它们合并。
### 5.7 `LockAcquireExtended()`：method table 进入状态机
入口先校验：
```text
lockmethodid = locktag->locktag_lockmethodid
lockmethodid 必须在 LockMethods[] 范围内
lockmode 必须在 1..lockMethodTable->numLockModes 范围内
```
然后确定 owner：
```text
sessionLock == true:
  owner = NULL
sessionLock == false:
  owner = CurrentResourceOwner
```
这解释 advisory lock 生命周期。
同一个 advisory key：
```text
pg_advisory_lock:
  LOCALLOCKOWNER.owner = NULL
pg_advisory_xact_lock:
  LOCALLOCKOWNER.owner = CurrentResourceOwner
```
shared `LOCK` / `PROCLOCK` 不区分 session 和 transaction。
这个差异在 `LOCALLOCK.lockOwners[]`。
### 5.8 local fast return：重入不再碰 shared table
`LockAcquireExtended()` 构造 local key：
```text
localtag.lock = *locktag
localtag.mode = lockmode
```
如果 `locallock->nLocks > 0`：
```text
GrantLockLocal(locallock, owner)
return LOCKACQUIRE_ALREADY_HELD 或 LOCKACQUIRE_ALREADY_CLEAR
```
这个路径不访问 shared lock hash。
原因是同一个 backend 不会被自己的同一个 object/mode 阻塞。
`LOCKACQUIRE_ALREADY_CLEAR` 让 relation/object wrapper 跳过重复 invalidation 吸收。
### 5.9 main lock table：LOCKTAG hash 到 partition
不能本地返回时进入 main table：
```text
hashcode = LockTagHashCode(locktag)
partitionLock = LockHashPartitionLock(hashcode)
LWLockAcquire(partitionLock, LW_EXCLUSIVE)
SetupLockInTable(lockMethodTable, MyProc, locktag, hashcode, lockmode)
```
`SetupLockInTable()` 做：
```text
找或创建 LOCK；
找或创建 PROCLOCK；
增加 lock->nRequested；
增加 lock->requested[lockmode]。
```
request count 在真正 grant 前增加。
因为 waiting 也是 request。
`PROCLOCK` hash 被设计成和对应 `LOCK` 落在同一 partition。
一个 partition LWLock 因此保护：
```text
LOCK entry；
相关 PROCLOCK entries；
LOCK.procLocks；
LOCK.waitProcs；
PGPROC 中等待该 LOCK 的 wait fields。
```
### 5.10 conflict check：对象无关，mode 表驱动
核心判断是：
```text
if (lockMethodTable->conflictTab[lockmode] & lock->waitMask)
    found_conflict = true;
else
    found_conflict = LockCheckConflicts(lockMethodTable, lockmode, lock, proclock);
```
先看等待者，是为了避免和前面等待请求冲突时插队。
再看已授予锁。
`LockCheckConflicts()` 先用：
```text
conflictMask & lock->grantMask
```
如果没有交集，直接 grant。
如果有交集，还要扣除本 backend 自己的锁和同一 parallel lock group 的锁。
`LOCKTAG_RELATION_EXTEND` 是少数特殊例外。
relation extension lock 在 group members 之间也冲突。
### 5.11 wait queue：PGPROC 进入 LOCK.waitProcs
冲突时：
```text
JoinWaitQueue(locallock, lockMethodTable, dontWait)
```
它设置：
```text
MyProc->heldLocks
MyProc->waitLock
MyProc->waitProcLock
MyProc->waitLockMode
MyProc->waitStatus = PROC_WAIT_STATUS_WAITING
lock->waitMask |= LOCKBIT_ON(lockmode)
```
并把 `MyProc->waitLink` 插入 `lock->waitProcs`。
通常插入队尾。
如果当前进程已持有的锁会阻塞前面的 waiter，源码可能把它插到第一个受影响 waiter 前。
这可以避免某些场景等到 deadlock timeout 才重排。
`dontWait=true` 且确实需要等待时，返回 `PROC_WAIT_STATUS_ERROR`。
调用者撤销 request/proclock 改动，再返回 `LOCKACQUIRE_NOT_AVAIL`。
### 5.12 sleep 与 grant：唤醒者先完成 shared state
等待路径：
```text
WaitOnLock()
  -> awaitedLock = locallock
  -> awaitedOwner = owner
  -> ProcSleep()
```
`ProcSleep()` 通过 latch 等待：
```text
WaitLatch(MyLatch,
          WL_LATCH_SET | WL_EXIT_ON_PM_DEATH,
          0,
          PG_WAIT_LOCK | locktag_type)
```
wait event 的 lock 名称来自 `locktag_type`。
释放者路径：
```text
LockRelease()
  -> UnGrantLock()
  -> CleanUpLock()
  -> ProcLockWakeup()
     -> LockCheckConflicts()
     -> GrantLock()
     -> ProcWakeup()
```
重点：
```text
唤醒者在 SetLatch 前已经 GrantLock()；
等待者醒来后只补本地 GrantLockLocal()。
```
这样避免等待者醒来后再和新请求者竞争 shared state。
### 5.13 `pg_locks`：把 tag 解码回 SQL 列
`pg_lock_status()` 第一次调用时采集：
```text
GetLockStatusData()
GetPredicateLockStatusData()
```
`GetLockStatusData()` 先扫描 fast path arrays，再扫描 main lock table。
main table 扫描时按 partition 顺序拿 shared LWLocks，复制 `LockInstanceData` 后释放。
`lockfuncs.c` 按 `locktag_type` 填列：
```text
RELATION / EXTEND:
  database, relation
PAGE:
  database, relation, page
TUPLE:
  database, relation, page, tuple
TRANSACTION:
  transactionid
VIRTUALTRANSACTION:
  virtualxid
OBJECT / USERLOCK / ADVISORY:
  database, classid, objid, objsubid
```
advisory lock 不是 catalog object。
它只是复用 object-like 列显示 field2 / field3 / field4。
`mode` 来自：
```text
GetLockmodeName(lockmethodid, mode)
```
`locktype` 来自：
```text
LockTagTypeNames[locktag_type]
```
诊断时要同时看：
```text
locktype；
database/relation/page/tuple/transactionid/classid/objid/objsubid；
mode；
granted；
fastpath；
pid / virtualtransaction。
```
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建
`LOCKTAG` 通常是栈上变量。
上层 wrapper 用 `SET_LOCKTAG_*` 宏填满字段。
创建者包括：
```text
lmgr.c:
  relation、page、tuple、transactionid、speculative token、object、apply transaction。
lockfuncs.c:
  advisory lock SQL 函数。
其它模块:
  构造 LOCKTAG 后传给 WaitForLockers()、LockAcquire() 或诊断 helper。
```
### 6.2 谁持有
shared 层持有状态：
```text
LOCK.granted[]
LOCK.grantMask
PROCLOCK.holdMask
PGPROC.myProcLocks[partition]
```
local 层持有状态：
```text
LOCALLOCK.nLocks
LOCALLOCK.lockOwners[]
ResourceOwner 的 lock cache
```
session-level lock 用 `owner = NULL`。
transaction-level lock 用 `owner = CurrentResourceOwner`。
### 6.3 谁释放
单个显式释放：
```text
LockRelease(locktag, lockmode, sessionLock)
  -> 找 LOCALLOCK
  -> 找对应 LOCALLOCKOWNER
  -> 减 owner count
  -> 减 locallock->nLocks
  -> 最后一次本地持有时释放 shared state
  -> UnGrantLock()
  -> CleanUpLock()
  -> RemoveLocalLock()
```
`CleanUpLock()` 负责：
```text
holdMask == 0 时删除 PROCLOCK；
nRequested == 0 时删除 LOCK；
仍有等待者且可能可授予时 ProcLockWakeup()。
```
### 6.4 transaction end
`ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)` 在顶层事务调用：
```text
ProcReleaseLocks(isCommit)
ReleasePredicateLocks(...)
```
`ProcReleaseLocks()` 做：
```text
LockErrorCleanup()
LockReleaseAll(DEFAULT_LOCKMETHOD, !isCommit)
LockReleaseAll(USER_LOCKMETHOD, false)
```
含义：
```text
commit:
  释放 DEFAULT 的非 session locks；
  释放 USER 的 transaction-level advisory locks；
  保留 session-level advisory locks。
abort:
  DEFAULT 释放所有 locks；
  USER 仍只释放 transaction-level advisory locks；
  session-level advisory locks 保留到显式 unlock 或 backend exit。
```
子事务 commit：
```text
LockReassignCurrentOwner()
```
子事务 abort：
```text
LockReleaseCurrentOwner()
```
如果 ResourceOwner lock cache 溢出，release 会扫描整个 local hash。
这是 correctness 优先于局部成本。
### 6.5 wait 中 ERROR cleanup
等待开始前：
```text
awaitedLock = locallock
awaitedOwner = owner
```
ERROR / cancel / lock_timeout 可能让控制流不走普通返回。
`LockErrorCleanup()` 兜底：
```text
AbortStrongLockAcquire()
关 deadlock / lock timeout
拿 partition LWLock
如果 MyProc 仍在 wait queue:
  RemoveFromWaitQueue()
否则如果 MyProc->waitStatus == OK:
  GrantAwaitedLock()
ResetAwaitedLock()
```
`GrantAwaitedLock()` 很关键。
可能别人已经授予锁并把本进程移出队列，但本进程还没来得及更新 `LOCALLOCK`。
cleanup 必须补齐 local ownership，后续 ResourceOwner release 才能释放 shared hold。
### 6.6 PREPARE TRANSACTION
`AtPrepare_Locks()` 把 transaction-level locks 写入 2PC state。
它忽略：
```text
session-level locks；
VXID locks。
```
fast path locks 会先转入 main table。
`PostPrepare_Locks()` 把 `PROCLOCK` ownership 从当前 `MyProc` 转移给 prepared transaction 的 dummy `PGPROC`。
不能直接改 `proclock->tag.myProc`。
因为它是 hash key。
源码使用 `hash_update_hash_key()`。
如果同一 `LOCKTAG` 同时有 session-level 和 transaction-level holds，PREPARE 会提前 ERROR。
这在 advisory lock 中可能发生。
## 7. 正确性机制层次
`LOCKTAG` 只提供对象身份。
它不保护内存，也不合并业务相关对象。
`LOCKTAG_OBJECT(dbid, pg_class, relid, 0)` 和 `LOCKTAG_RELATION(dbid, relid)` 是两把锁。
`LockMethodData` 只提供 mode 冲突规则。
它不知道 relation 名称、tuple 可见性、advisory key 的业务意义。
partition `LWLock` 只保护 lock manager shared memory。
它不是用户级 heavyweight lock。
`ResourceOwner` 管 transaction-level lock cleanup。
MemoryContext 回收不了 shared `LOCK` / `PROCLOCK` ownership。
relation/object lock 成功后还要吸收 invalidation。
`lockCleared` 是 backend-local 状态，用于避免重复处理已吸收的 sinval。
relation `AccessExclusiveLock` 在 hot standby 相关场景可能写 WAL standby lock record。
这不是所有 heavyweight lock 都 WAL logged。
deadlock detector 只看 waits-for graph。
它的 edge 是：
```text
waiter PGPROC -> blocker PGPROC via LOCK
```
对象类型只影响诊断显示和上层为什么选择这把锁。
## 8. 错误路径 / 异常路径 / fallback
非法 method 或 mode：
```text
lockmethodid 不在 LockMethods[]；
lockmode 不在 1..numLockModes。
```
结果是 `elog(ERROR)`，通常说明调用方构造了坏 tag 或坏 mode。
recovery 限制：
```text
RecoveryInProgress() 且非 recovery backend；
LOCKTAG_OBJECT 或 LOCKTAG_RELATION；
mode > RowExclusiveLock。
```
会报“cannot acquire lock mode ... while recovery is in progress”。
shared memory 不足：
```text
SetupLockInTable() 创建 LOCK 或 PROCLOCK 失败。
```
常见提示是增加 `max_locks_per_transaction`。
如果创建 `LOCK` 成功但 `PROCLOCK` 失败，且没有其它 request，源码会删除刚创建的 `LOCK`，避免永久泄漏。
`dontWait` 路径：
```text
try lock 仍会进入 LockAcquireExtended()；
如果确实需要等待，撤销 request/proclock 改动；
返回 LOCKACQUIRE_NOT_AVAIL。
```
它不是完全不碰 shared table。
deadlock 路径：
```text
ProcSleep() 先睡；
deadlock_timeout 后 CheckDeadLock()；
CheckDeadLock() 锁住所有 partitions；
DeadLockCheck(MyProc) 找 waits-for cycle；
hard deadlock 时 RemoveFromWaitQueue()；
之后 DeadLockReport() ERROR。
```
lock timeout / cancel 路径：
```text
ERROR 可能发生在 ProcSleep() 内；
LockErrorCleanup() 必须摘 wait queue 或补记已授予 lock；
ResourceOwner release 再释放本事务持有的锁。
```
strong relation lock fast path gate：
```text
BeginStrongLockAcquire()
  -> FastPathStrongRelationLocks count++
失败或 ERROR:
  -> AbortStrongLockAcquire()
  -> count--
```
否则会长期阻止对应 fast path partition。
`pg_locks` 快照边界：
```text
fast path 部分不是严格全局同一时刻；
main table 部分通过锁所有 partitions 得到自洽快照；
视图优先短时间持锁，而不是提供审计历史。
```
## 9. 成本、资源与跨模块传播
lock table 容量来自：
```text
max_locks_per_xact * (MaxBackends + max_prepared_xacts)
```
一个活跃 lockable object 至少需要一个 `LOCK`。
每个 holder/waiter backend 对该 object 需要一个 `PROCLOCK`。
资源压力随这些因素增长：
```text
并发 backend 数；
每事务访问 relation 数；
advisory key 基数；
等待同一对象的 backend 数；
prepared transaction 持有锁数量；
DDL / maintenance 同时触达的对象数量。
```
contention 主要来自 hot `LOCKTAG` 所在 partition。
同一热表的频繁 weak relation locks 会竞争同一个 partition LWLock。
relation fast path 是为了降低这类热路径成本。
fast path 不改变语义。
可能参与冲突或 deadlock 的锁必须转入 main lock table。
`pg_locks` 本身也有观测成本：
```text
扫描所有 backend fast path arrays；
锁所有 lock partitions；
扫描 PROCLOCK hash；
复制状态到本地内存。
```
deadlock detector 成本更高。
它按顺序锁所有 partitions。
所以 PostgreSQL 用 optimistic waiting：先等 `deadlock_timeout`，再检测。
跨模块传播包括：
```text
heap/MVCC:
  tuple update conflict 常表现为 transactionid wait。
relcache/syscache:
  relation/object lock 后要处理 invalidation。
WAL/hot standby:
  relation AccessExclusiveLock 可能写 standby lock 信息。
autovacuum:
  deadlock detector 可识别 autovacuum blocker 并触发取消。
parallel query:
  lock group 让组内进程多数情况下不互相阻塞。
stats:
  pg_stat_lock 按 locktag_type 聚合等待统计。
```
## 10. 观测与诊断入口
### 10.1 `pg_locks`
基本查询：
```sql
SELECT locktype, database, relation, page, tuple,
       virtualxid, transactionid, classid, objid, objsubid,
       virtualtransaction, pid, mode, granted, fastpath, waitstart
FROM pg_locks
ORDER BY granted, locktype, pid;
```
解读：
```text
locktype:
  LOCKTAG type。
mode:
  LOCKMODE 名称。
granted:
  true 来自 holdMask，false 来自 waitLockMode。
fastpath:
  是否从 fast path 表示采集。
waitstart:
  PGPROC.waitStart 快照；刚开始等待的短窗口可能为 NULL。
```
`locktype = tuple` 不等于所有 row lock 都可见。
很多行冲突会表现为 `transactionid` wait。
### 10.2 `pg_blocking_pids()`
`pg_blocking_pids(pid)` 使用 `GetBlockerStatusData()`。
它考虑：
```text
hard block:
  已持有冲突 mode 的 holder。
soft block:
  wait queue 中排在你前面且请求冲突 mode 的 waiter。
```
所以它比手写 join `pg_locks` 更贴近 wait queue 语义。
但它仍是瞬时快照。
### 10.3 wait event
等待 heavyweight lock 时：
```text
PG_WAIT_LOCK | locktag_type
```
查询：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```
`wait_event` 反映 lock tag type。
它不显示 mode。
mode 要看 `pg_locks` 中 `granted = false` 的行。
### 10.4 `pg_stat_lock`
当前源码提供 `pg_stat_lock`。
底层 `pg_stat_get_lock()` 输出：
```text
locktype
waits
wait_time
fastpath_exceeded
stats_reset
```
查询：
```sql
SELECT *
FROM pg_stat_lock
ORDER BY wait_time DESC;
```
`pg_stat_lock` 是聚合统计。
`pg_locks` 是当前实例快照。
二者不能互相替代。
### 10.5 server log 与 error context
`log_lock_waits` 可记录：
```text
still waiting；
acquired after wait；
deadlock detected；
soft deadlock avoided by queue rearrangement。
```
`DescribeLockTag()` 生成对象描述。
它通常打印 OID 或 raw field。
它不查 catalog 名称，因为死锁报告路径不能再拿系统表锁。
### 10.6 gdb 断点
建议断点：
```text
LockAcquireExtended
SetupLockInTable
LockCheckConflicts
JoinWaitQueue
WaitOnLock
LockRelease
CleanUpLock
GetLockStatusData
pg_lock_status
```
观察变量：
```text
*locktag
locktag->locktag_type
locktag->locktag_lockmethodid
lockmode
lockMethodTable->conflictTab[lockmode]
lock->grantMask
lock->waitMask
proclock->holdMask
locallock->nLocks
MyProc->waitLockMode
MyProc->waitStatus
```
### 10.7 观测边界
能直接看到：
```text
pg_locks 的 locktype / mode / granted / pid / fastpath / waitstart；
pg_stat_activity 的 Lock wait event；
pg_blocking_pids() 的 blocker pid；
pg_stat_lock 的聚合等待次数和时间。
```
只能推断：
```text
某次 SQL 为什么选择这个 mode；
transactionid wait 背后的具体 tuple；
soft blocker 对等待时间贡献；
fast path 转入 main table 的瞬间。
```
通常看不到：
```text
LOCALLOCK 每次重入计数变化；
ResourceOwner lock cache 溢出历史；
已经释放并回收的 LOCK / PROCLOCK；
短暂 fast path lock 的完整时间线。
```
## 11. 常见误区
误区一：`locktype` 是对象类型，`mode` 才是请求强度。
误区二：advisory lock 走同一套 `LockAcquire()` / `LockRelease()`、wait queue 和 deadlock detector。
误区三：transactionid wait 等待事务结束，具体 tuple 需要结合 SQL、日志和堆访问路径推断。
误区四：object lock 不会自动和 relation lock 冲突，tag type 不同就是不同 lockable object。
误区五：`fastpath=false` 是表示路径，不是 mode。
误区六：`pg_locks` 是当前快照，短锁可能完全看不到。
## 12. 课堂实验
### 12.1 relation lock
会话 A：
```sql
BEGIN;
LOCK TABLE pg_class IN ACCESS EXCLUSIVE MODE;
```
会话 B：
```sql
SELECT count(*) FROM pg_class;
```
会话 C：
```sql
SELECT locktype, relation::regclass, mode, granted, pid, waitstart
FROM pg_locks
WHERE locktype = 'relation'
  AND relation = 'pg_class'::regclass
ORDER BY granted, pid;
```
回到源码：
```text
LockRelationOid()
  -> SET_LOCKTAG_RELATION()
  -> LockAcquireExtended()
  -> conflictTab[AccessShareLock] 与 grantMask 中 AccessExclusiveLock 冲突。
```
### 12.2 transactionid wait
会话 A：
```sql
CREATE TEMP TABLE t_locktag(id int primary key, v int);
INSERT INTO t_locktag VALUES (1, 10);
BEGIN;
UPDATE t_locktag SET v = 11 WHERE id = 1;
```
会话 B：
```sql
BEGIN;
UPDATE t_locktag SET v = 12 WHERE id = 1;
```
会话 C：
```sql
SELECT locktype, transactionid, mode, granted, pid
FROM pg_locks
WHERE locktype IN ('transactionid', 'tuple')
ORDER BY locktype, granted, pid;
```
回到源码：
```text
XactLockTableInsert()
  -> SET_LOCKTAG_TRANSACTION()
  -> ExclusiveLock
XactLockTableWait()
  -> SET_LOCKTAG_TRANSACTION()
  -> ShareLock
```
### 12.3 advisory key 编码
会话 A：
```sql
SELECT pg_advisory_lock(42);
```
会话 B：
```sql
SELECT pg_try_advisory_lock(42);
```
会话 C：
```sql
SELECT locktype, database, classid, objid, objsubid,
       mode, granted, pid
FROM pg_locks
WHERE locktype = 'advisory'
ORDER BY granted, pid;
```
观察：
```text
locktype = advisory；
int8 key 拆到 classid / objid；
objsubid = 1；
try 失败不会长期留下 waiting row。
```
### 12.4 断点跟读
设置断点：
```text
b LockAcquireExtended
b SetupLockInTable
b LockCheckConflicts
b JoinWaitQueue
b ProcLockWakeup
b pg_lock_status
```
分别执行：
```sql
LOCK TABLE pg_class IN SHARE MODE;
SELECT pg_advisory_xact_lock(1, 2);
```
比较：
```text
locktag_type；
locktag_lockmethodid；
lockmode；
conflictTab；
LOCK.grantMask；
PROCLOCK.holdMask；
LOCALLOCK.nLocks。
```
## 13. 讨论题
1. 为什么 `LOCKTAG` 要包含 `locktag_type`，而不是把所有对象压成一个无类型整数？
2. DEFAULT 和 USER 当前使用同一张 conflict table，为什么仍保留 lock method table？
3. 为什么 `LOCKTAG_OBJECT` 不能替代 `LOCKTAG_RELATION`？
4. 为什么 session-level 与 transaction-level ownership 留在 `LOCALLOCK`，而不是拆成两个 shared `PROCLOCK`？
5. wait 中 ERROR 后，为什么 `LockErrorCleanup()` 可能需要 `GrantAwaitedLock()`？
6. advisory lock 显示在 `classid` / `objid` / `objsubid`，这是否说明它是 catalog object？
7. 新增一种 `LOCKTAG` 时，除了宏，还要同步哪些观测和诊断入口？
8. deadlock detector 为什么可以不理解 relation/page/tuple/advisory 的业务语义？
## 14. 本节小结
本节主链路：
```text
具体对象
  -> SET_LOCKTAG_* 编码成 16 字节 typed key
  -> locktag_lockmethodid 选择 LockMethodData
  -> lock mode 查 conflictTab
  -> LOCKTAG hash 定位 shared LOCK
  -> LOCK + PGPROC 定位 PROCLOCK
  -> LOCALLOCK 记录本 backend owner 和重入计数
  -> wait queue、deadlock detector、pg_locks 复用同一套状态。
```
核心边界：
```text
LOCKTAG:
  对象身份。
LOCKMODE:
  请求强度。
LockMethodData:
  冲突规则。
LOCK:
  每个对象的 shared runtime state。
PROCLOCK:
  某 backend 对某对象的 shared interest。
LOCALLOCK:
  backend-local owner、重入计数和 cleanup 边界。
```
cleanup 规律：
```text
transaction-level lock 交给 ResourceOwner；
session-level advisory lock 跨事务保留；
wait 中 ERROR 由 LockErrorCleanup() 摘队或补记授予；
计数归零后 CleanUpLock() 回收 PROCLOCK / LOCK 并唤醒等待者。
```
观测规律：
```text
pg_locks 看当前实例；
pg_blocking_pids() 看 hard / soft blocker；
pg_stat_activity 看 Lock wait event；
pg_stat_lock 看按 locktype 聚合的等待统计；
server log 和 DescribeLockTag() 给 OID / raw field 级上下文。
```
可迁移 mental model：
```text
可扩展锁管理器通常先把异构对象规范化成 typed key，
再把请求强度规范化成 table-driven conflict rule，
最后把 ownership、wait、deadlock 和 cleanup 统一在少数 shared state 上。
```
判断边界：
```text
pg_locks 是快照，不是历史；
mode 名称不是完整业务语义；
transactionid wait 不直接告诉你 tuple；
object lock 不会自动和 relation lock 合并；
fast path 是表示优化，不是另一套锁规则；
性能判断依赖 workload、backend 数、对象基数、等待时间和版本实现。
```
