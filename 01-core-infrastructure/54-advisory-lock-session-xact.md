# PostgreSQL advisory lock session / transaction boundary
## 课程定位
前置知识：已经理解 `LOCKTAG`、`LOCK`、`PROCLOCK`、`LOCALLOCK`、`PGPROC`、`ResourceOwner`、事务结束 cleanup 和 `pg_locks` 的基本含义。
本节唯一主问题：
```text
session-level 与 transaction-level advisory lock 的 owner、释放边界和诊断方式有什么不同？
```
核心矛盾：
```text
advisory lock 要给应用一个简单的整数 key 互斥工具；
内核却必须同时支持跨事务的 session hold 和随事务自动释放的 xact hold；
两种 hold 又必须在同一把 LOCKTAG 上互相冲突、可等待、可死锁检测、可诊断。
```
一句话运行模型：
```text
lockfuncs.c 把 SQL key 编码成 LOCKTAG_ADVISORY；
session 函数用 LockAcquire(..., sessionLock=true, ...)，owner 记为 NULL；
xact 函数用 LockAcquire(..., sessionLock=false, ...)，owner 记为 CurrentResourceOwner；
共享 lock table 只看 LOCKTAG 和 lock mode；
释放边界由 LOCALLOCK.lockOwners[] 中的 owner 切分。
```
学完后应能判断：
- 为什么 `pg_advisory_lock()` 在 `ROLLBACK` 后仍然持有。
- 为什么 `pg_advisory_xact_lock()` 不能显式 unlock。
- 为什么同一个 key 的 session lock 和 xact lock 会互相阻塞。
- 为什么同一 backend 重复拿同一 advisory lock 会立即成功。
- 为什么 `pg_locks` 能看到 advisory lock，却不能直接告诉你 owner 是 session 还是 xact。
- 为什么 `pg_advisory_unlock_all()` 只释放 session-level advisory locks。
- 为什么 `DISCARD ALL` 和 backend shutdown 需要额外释放 USER locks。
- 为什么 `PREPARE TRANSACTION` 遇到同一对象同时有 session 和 xact hold 会报错。
本课基于本地源码：
```text
/home/nail/postgres
branch: master
commit: 0e1f1ed157e9
```
本节不是 advisory lock 使用教程。
它只追一个内核边界：同一个 `LOCKTAG_ADVISORY`，如何被 `LOCALLOCKOWNER.owner` 切成 session hold 和 transaction hold。
## 1. 本节在总主线中的位置
第 49 节说明 `LOCKTAG` 和 lock method table 如何把不同对象压进同一套锁管理器。
第 50 节说明 `LockAcquireExtended()`、`LockRelease()`、wait queue、deadlock detector 和 `ResourceOwner` 如何协作。
第 51 节说明 relation lock fast path 为什么是物理表示优化，而不是新语义。
advisory lock 站在这些机制之上。
它没有自己的 wait queue。
它没有自己的死锁检测器。
它没有自己的 shared memory hash table。
它只是使用 `LOCKTAG_ADVISORY` 和 `USER_LOCKMETHOD` 进入通用 heavyweight lock manager。
但 advisory lock 有一个特殊之处：用户可以选择 owner 边界。
普通 relation lock 通常属于当前事务。
advisory lock 的 SQL API 则把这个选择暴露出来。
- `pg_advisory_lock*` 表示 session-level。
- `pg_advisory_xact_lock*` 表示 transaction-level。
- `pg_try_advisory_lock*` 表示 session-level 非阻塞尝试。
- `pg_try_advisory_xact_lock*` 表示 transaction-level 非阻塞尝试。
- `pg_advisory_unlock*` 只释放 session-level。
- `pg_advisory_unlock_all()` 释放本 backend 的 session-level advisory locks。
所以本节不是重新讲 conflict table。
`ShareLock` 和 `ExclusiveLock` 的兼容性仍然来自 `LockConflicts[]`。
本节也不重新讲 relation fast path；advisory lock 不走 relation fast path。
本节要看的状态是：
```text
LOCALLOCK.lockOwners[]
  -> owner == NULL
  -> owner == CurrentResourceOwner
  -> nLocks
  -> LockReleaseAll(USER_LOCKMETHOD, allLocks)
```
这个状态不在 `pg_locks` 中直接展示。
SQL 视图能看到某个 backend 持有 advisory lock。
但 session/xact 边界需要结合函数调用、事务状态、释放行为和源码 owner 规则推断。
## 2. 核心矛盾与一句话运行模型
advisory lock 的表面 API 很简单：
```sql
SELECT pg_advisory_lock(42);
SELECT pg_advisory_unlock(42);
SELECT pg_advisory_xact_lock(42);
```
内核问题却不是“怎么存一个整数 key”。
真正的问题是：同一个 key 必须在同一套冲突规则中互斥，但释放边界可以是 session，也可以是 transaction。
如果 session-level 和 xact-level 使用两套 lock table，同一个 key 在两套表里不能自然互斥，deadlock detector 也要同时理解两套等待图。
如果完全只依赖 shared `PROCLOCK`，又无法区分释放边界。
`PROCLOCK` 的 key 是 `LOCK *myLock + PGPROC *myProc`，它回答“这个 backend 对这个 lockable object 持有哪些 mode”，不回答“这个 hold 应该在 transaction end 释放，还是 session end 释放”。
PostgreSQL 的解法是两层分离。
共享层只表达冲突事实。
本地层表达 ownership 边界。
共享层包括：
- `LOCKTAG_ADVISORY`
- `USER_LOCKMETHOD`
- `LOCK`
- `PROCLOCK`
- `PGPROC` wait state
- `LockConflicts[]`
本地层包括：
- `LOCALLOCK`
- `LOCALLOCKOWNER`
- `ResourceOwner`
- `owner == NULL`
- `CurrentResourceOwner`
所以一句话运行模型可以压成：
```text
LOCKTAG 决定和谁冲突；
LOCKMODE 决定怎么冲突；
LOCALLOCKOWNER.owner 决定何时释放。
```
这个分层解释了三个现象。
第一，session 和 xact advisory lock 会互相阻塞，因为它们进入同一个 `LOCKTAG_ADVISORY`。
第二，xact advisory lock 自动释放，因为它被 `CurrentResourceOwner` 记住，事务结束时 release owner。
第三，`pg_locks` 看不出 owner 边界，因为 `pg_locks` 从 shared lock data 投影，而 session/xact owner 是 backend-local `LOCALLOCK` 细节。
## 3. 核心源码文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/adt/lockfuncs.c` | SQL 函数如何把 int8 / int4 key 编成 `LOCKTAG_ADVISORY`，以及如何传 `sessionLock`。 |
| 2 | `src/include/storage/locktag.h` | `LOCKTAG_ADVISORY`、`USER_LOCKMETHOD`、`SET_LOCKTAG_ADVISORY()` 的 key 布局。 |
| 3 | `src/include/storage/lockdefs.h` | advisory lock 使用标准 `ShareLock` / `ExclusiveLock` mode。 |
| 4 | `src/include/storage/lock.h` | `LOCALLOCK`、`LOCALLOCKOWNER`、`LockAcquire()`、`LockReleaseAll()` 接口。 |
| 5 | `src/backend/storage/lmgr/lock.c` | `LockAcquireExtended()` 如何选择 owner，`GrantLockLocal()` 如何记账，`LockReleaseAll()` 如何区分 session / xact。 |
| 6 | `src/backend/storage/lmgr/proc.c` | `ProcReleaseLocks()` 在 top-level transaction end 释放 xact advisory locks。 |
| 7 | `src/backend/utils/resowner/resowner.c` | `RESOURCE_RELEASE_LOCKS` 阶段如何调用 `ProcReleaseLocks()` 或 retail release subtransaction locks。 |
| 8 | `src/backend/utils/init/postinit.c` | `ShutdownPostgres()` 为什么要显式 `LockReleaseAll(USER_LOCKMETHOD, true)`。 |
| 9 | `src/backend/commands/discard.c` | `DISCARD ALL` 为什么释放 session-level advisory locks。 |
| 10 | `src/backend/access/transam/xact.c` | commit / abort / subtransaction end 何时进入 `ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)`。 |
| 11 | `src/backend/utils/adt/lockfuncs.c` | `pg_lock_status()` 如何把 `LOCKTAG_ADVISORY` 投影到 `pg_locks` 列。 |
| 12 | `src/backend/storage/lmgr/lock.c` | `AtPrepare_Locks()` / `PostPrepare_Locks()` 的 2PC 边界。 |
推荐阅读顺序：
```text
lockfuncs.c advisory SQL wrappers
  -> locktag.h 的 SET_LOCKTAG_ADVISORY
  -> lock.c 的 LockAcquireExtended(owner 选择)
  -> lock.c 的 GrantLockLocal()
  -> resowner.c 的 ResourceOwnerRelease(... LOCKS ...)
  -> proc.c 的 ProcReleaseLocks()
  -> lock.c 的 LockReleaseAll(USER_LOCKMETHOD, false/true)
  -> lockfuncs.c 的 pg_lock_status()
  -> lock.c 的 AtPrepare_Locks()
```
不要从 `pg_locks` 视图开始读；`pg_locks` 是结果投影。
本节主语义在 owner 记账和 release 边界。
也不要从应用场景开始扩展；leader election、migration guard、job queue 等场景都不是本节核心。
## 4. 关键数据结构与状态
### 4.1 `LOCKTAG_ADVISORY`：应用 key 进入 lock manager 的对象身份
`lockfuncs.c` 中 advisory SQL wrapper 使用两个本地宏。
简化后是：
```text
SET_LOCKTAG_INT64(tag, key64)
  -> field1 = MyDatabaseId
  -> field2 = high 32 bits
  -> field3 = low 32 bits
  -> field4 = 1
  -> type = LOCKTAG_ADVISORY
  -> method = USER_LOCKMETHOD
SET_LOCKTAG_INT32(tag, key1, key2)
  -> field1 = MyDatabaseId
  -> field2 = key1
  -> field3 = key2
  -> field4 = 2
  -> type = LOCKTAG_ADVISORY
  -> method = USER_LOCKMETHOD
```
这里有四个语义。
第一，advisory lock 默认是 database-local，因为 `field1 = MyDatabaseId`。
第二，`int8` key 和两个 `int4` key 不共享 namespace，因为 `field4` 用 `1` 和 `2` 区分。
第三，advisory lock 使用 `LOCKTAG_ADVISORY`，在 `pg_locks.locktype` 中显示为 `advisory`。
第四，advisory lock 使用 `USER_LOCKMETHOD`，和 DEFAULT lock method 的 relation、transactionid、object lock 分开。
`locktag_lockmethodid` 是 `LOCKTAG` 的一部分。
同样的四个 field 值，如果 method 不同，就不是同一个 lockable object。
### 4.2 `LOCKMODE`：shared / exclusive advisory lock 的冲突强度
advisory lock 没有专属 mode。
SQL API 只暴露两种常用强度：
- shared advisory lock 使用 `ShareLock`。
- exclusive advisory lock 使用 `ExclusiveLock`。
这意味着 conflict table 仍然是通用 `LockConflicts[]`。
真正判断发生在 `LockCheckConflicts()`，输入包括 `lockMethodTable->conflictTab[requested_mode]`、`LOCK.grantMask`、`PROCLOCK.holdMask` 和 lock group membership。
shared advisory lock 之间兼容。
exclusive advisory lock 和同 key 的 shared / exclusive advisory lock 冲突。
session 与 xact 不是 mode，也不是 lock type；它们只是 local owner 边界。
### 4.3 `LOCALLOCK`：当前 backend 的本地账本
`LOCALLOCK` 存在于 backend-local hash `LockMethodLocalHash`。
key 是：
```text
LOCKTAG + LOCKMODE
```
同一个 advisory key 的 `ShareLock` 和 `ExclusiveLock` 是两个 `LOCALLOCK` entry。
同一个 advisory key 的 session-level 和 transaction-level hold 如果 mode 相同，则共享同一个 `LOCALLOCK` entry。
`LOCALLOCK` 的关键字段是：
| 字段 | 本节语义 |
| --- | --- |
| `tag.lock` | `LOCKTAG_ADVISORY` 和 `USER_LOCKMETHOD`。 |
| `tag.mode` | `ShareLock` 或 `ExclusiveLock`。 |
| `lock` | shared `LOCK` 指针。advisory lock 正常走 main lock table。 |
| `proclock` | shared `PROCLOCK` 指针。 |
| `nLocks` | 本 backend 对该 `(LOCKTAG, mode)` 的总持有次数。 |
| `lockOwners` | 按 owner 切分的持有次数。 |
| `numLockOwners` | `lockOwners` 当前有效项数。 |
`LOCALLOCK.nLocks` 是本 backend 总次数，不是 shared lock table 中的全局 holder 数。
`PROCLOCK.holdMask` 只需要表达该 backend 是否持有某个 mode。
重复获取同一 lock mode 不需要重复修改 shared hash table，重复次数必须留在 `LOCALLOCK` 中。
### 4.4 `LOCALLOCKOWNER`：本节最关键字段
`LOCALLOCKOWNER` 的字段很少：
```text
owner
nLocks
```
`owner` 的语义是：
```text
owner == NULL:
  session-level hold
owner == CurrentResourceOwner:
  transaction-level hold
```
这不是内存 ownership，而是释放边界。
`nLocks` 表示这个 owner 对同一 `LOCALLOCK` 的重复持有次数。
例如同一个 backend 执行：
```sql
SELECT pg_advisory_lock(7);
SELECT pg_advisory_lock(7);
```
会形成：
```text
LOCALLOCK.nLocks = 2
lockOwners[owner=NULL].nLocks = 2
```
必须执行两次 `pg_advisory_unlock(7)` 才真正释放 shared lock。
再比如：
```sql
BEGIN;
SELECT pg_advisory_lock(7);
SELECT pg_advisory_xact_lock(7);
COMMIT;
```
同一 backend 已持有同 key session lock，xact lock 请求会本地立即成功。
commit 后 xact owner 被释放，session owner 仍保留。
这不是绕过冲突规则，而是“自己已经持有”在 lock manager 中被视为可重入。
### 4.5 `ResourceOwner`：transaction-level cleanup 的索引
`ResourceOwner` 不保存 shared lock 本体。
它保存的是 `LOCALLOCK *` 的 lossy cache。
`GrantLockLocal()` 中只有在 `owner != NULL` 时才调用 `ResourceOwnerRememberLock(owner, locallock)`。
所以：
```text
transaction-level advisory lock:
  进入 ResourceOwner
session-level advisory lock:
  不进入 ResourceOwner
```
这解释了为什么事务 abort 不能释放 session lock。
abort cleanup 走 `ResourceOwnerRelease()`，session lock 根本不在 `ResourceOwner` 的 lock cache 里。
但 session lock 仍然在 `LOCALLOCK.lockOwners` 里，由 explicit unlock、`LockReleaseSession()`、`LockReleaseAll(..., true)` 或 backend shutdown 清理。
### 4.6 `LOCK` 与 `PROCLOCK`：共享冲突事实
`LOCK` 和 `PROCLOCK` 不知道 session/xact owner 的细节。
`LOCK` 表示某个 `LOCKTAG_ADVISORY` 的全局状态，包含 `grantMask`、`waitMask`、`procLocks`、`waitProcs`、`requested[]`、`granted[]`。
`PROCLOCK` 表示某个 `PGPROC` 对某个 `LOCK` 的状态，包含 `holdMask`、`releaseMask`、`lockLink`、`procLink`。
从 shared lock table 看：
```text
backend A 持有 advisory key 7 的 ExclusiveLock
```
至于这个 hold 是 session-level 还是 xact-level，shared table 不直接表达。
这不是信息遗漏，而是设计分层。
冲突判断不需要知道释放边界，释放时才需要 local owner。
### 4.7 `pg_locks` 投影：能看到对象，不能看到 local owner
`pg_lock_status()` 调用 `GetLockStatusData()` 把 shared lock state 取出来。
对 `LOCKTAG_ADVISORY`，投影规则和 object-like lock 类似：
| `LOCKTAG` field | `pg_locks` 列 |
| --- | --- |
| `field1` | `database` |
| `field2` | `classid` |
| `field3` | `objid` |
| `field4` | `objsubid` |
所以 advisory lock 在 `pg_locks` 里常见形态是：
```text
locktype = advisory
database = 当前数据库 OID
classid  = key part 1
objid    = key part 2
objsubid = 1 或 2
mode     = ShareLock 或 ExclusiveLock
granted  = true/false
```
`objsubid = 1` 表示 SQL 使用 `int8` key family。
`objsubid = 2` 表示 SQL 使用两个 `int4` key family。
但 `pg_locks` 没有 `session_level` 或 `resource_owner` 列。
如果想判断释放边界，必须做 runtime 实验或结合当前事务行为。
## 5. 主流程源码 walkthrough
### 5.1 session-level acquire：`pg_advisory_lock(int8)`
入口在 `lockfuncs.c`。
主链路是：
```text
pg_advisory_lock_int8()
  -> SET_LOCKTAG_INT64(tag, key)
  -> LockAcquire(&tag, ExclusiveLock, true, false)
  -> LockAcquireExtended(..., sessionLock=true, dontWait=false, ...)
```
SQL 参数先被读成 `int64 key`，`SET_LOCKTAG_INT64()` 生成 `LOCKTAG_ADVISORY`。
`LockAcquire()` 传入 `ExclusiveLock`、`sessionLock=true`、`dontWait=false`。
`dontWait=false` 表示冲突时会等待。
进入 `LockAcquireExtended()` 后，第一条关键分支是 owner 选择：
```text
if (sessionLock)
  owner = NULL;
else
  owner = CurrentResourceOwner;
```
session-level advisory lock 的 owner 不是某个全局 Session 对象，而是 `NULL`。
后面的 release 逻辑必须和这个约定完全一致。
然后构造 `LOCALLOCKTAG`：
```text
localtag.lock = *locktag
localtag.mode = lockmode
```
`LockMethodLocalHash` 查找或创建 `LOCALLOCK`。
如果这个 backend 已经持有同一 `(LOCKTAG, mode)`，`locallock->nLocks > 0`，通常可以直接 `GrantLockLocal()` 增加本地计数，不再次进入 shared conflict path。
这解释了文档现象：同一 session 已经持有某 advisory lock 时，后续同 key 请求总是成功。
如果本地没有 hold，才进入 shared lock table。
advisory lock 不符合 relation fast path 条件，所以会通过 main lock table 建立 `LOCK` 和 `PROCLOCK`。
共享层授予成功后调用 `GrantLockLocal(locallock, owner)`。
对 session-level 来说，`owner = NULL`。
`GrantLockLocal()` 增加 `locallock->nLocks`，在 `lockOwners[]` 中找到 `owner == NULL` 的 entry 并增加计数。
因为 `owner == NULL`，它不会调用 `ResourceOwnerRememberLock()`。
这就是 session-level 的核心：它有 shared lock holder，有 local hold count，但没有 ResourceOwner 绑定。
### 5.2 transaction-level acquire：`pg_advisory_xact_lock(int8)`
transaction-level 入口同样在 `lockfuncs.c`。
主链路是：
```text
pg_advisory_xact_lock_int8()
  -> SET_LOCKTAG_INT64(tag, key)
  -> LockAcquire(&tag, ExclusiveLock, false, false)
  -> LockAcquireExtended(..., sessionLock=false, dontWait=false, ...)
```
和 session-level 的唯一关键参数差异是 `sessionLock=false`。
进入 `LockAcquireExtended()` 后，`owner = CurrentResourceOwner`。
之后的 shared conflict path 相同：同一个 `LOCKTAG_ADVISORY`、同一个 `USER_LOCKMETHOD`、同一个 `ExclusiveLock`、同一个 `LockConflicts[]`。
差异只在 `GrantLockLocal()`。
对 transaction-level 来说，`owner != NULL`，所以 `GrantLockLocal()` 会调用 `ResourceOwnerRememberLock(owner, locallock)`。
从此这个 transaction owner 能在 cleanup 阶段找到这把 lock。
如果当前事务 abort，`ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)` 会释放它。
如果当前事务 commit，top-level 事务也会释放它。
如果当前是 subtransaction，commit 会转移给 parent owner，abort 会 retail release。
这个行为来自 `resowner.c` 的 `ResourceOwnerReleaseInternal()`：
```text
top-level:
  ProcReleaseLocks(isCommit)
subtransaction commit:
  LockReassignCurrentOwner(...)
subtransaction abort:
  LockReleaseCurrentOwner(...)
```
所以 xact-level advisory lock 的生命周期不是当前 SQL 语句，也不是当前 portal，而是 `ResourceOwner` 树上的事务生命周期。
### 5.3 shared / exclusive variants：只改 lock mode
shared advisory SQL 函数使用 `ShareLock`。
exclusive advisory SQL 函数使用 `ExclusiveLock`。
```text
pg_advisory_lock_shared_int8()
  -> LockAcquire(&tag, ShareLock, true, false)
pg_advisory_xact_lock_shared_int8()
  -> LockAcquire(&tag, ShareLock, false, false)
```
两个维度彼此独立。
第一个维度是 mode：`ShareLock` 或 `ExclusiveLock`。
第二个维度是 owner boundary：`sessionLock=true` 或 `sessionLock=false`。
不要把 shared lock 误解成 session lock，也不要把 exclusive lock 误解成 transaction lock。
### 5.4 try variants：只改 `dontWait`
`pg_try_advisory_lock*` 和 `pg_try_advisory_xact_lock*` 的差异是 `dontWait=true`。
```text
pg_try_advisory_lock_int8()
  -> LockAcquire(&tag, ExclusiveLock, true, true)
pg_try_advisory_xact_lock_int8()
  -> LockAcquire(&tag, ExclusiveLock, false, true)
```
如果发现冲突，`LockAcquire()` 返回 `LOCKACQUIRE_NOT_AVAIL`，SQL wrapper 返回 `false`。
try 不改变 owner，不改变释放边界，只改变等待行为。
如果自己已经持有兼容或同 mode hold，请求通常会本地成功。
### 5.5 explicit unlock：只针对 session-level
`pg_advisory_unlock_int8()` 的链路是：
```text
pg_advisory_unlock_int8()
  -> SET_LOCKTAG_INT64(tag, key)
  -> LockRelease(&tag, ExclusiveLock, true)
```
`sessionLock=true` 是关键。
`LockRelease()` 会使用同样的 owner 选择规则：
```text
if (sessionLock)
  owner = NULL;
else
  owner = CurrentResourceOwner;
```
SQL 暴露的 unlock 函数只调用 `sessionLock=true`。
所以它只释放 `owner == NULL` 的 hold。
如果当前 backend 只持有 transaction-level advisory lock，调用 `pg_advisory_unlock()` 不会释放它；它会找不到 session owner 的 hold，并返回 false 或产生 warning 路径。
这不是权限问题，而是 owner 不匹配。
transaction-level advisory lock 没有 SQL unlock 函数。
它只能随 transaction / subtransaction / prepared transaction 边界释放。
### 5.6 `pg_advisory_unlock_all()`：session-level 批量释放
`pg_advisory_unlock_all()` 的实现是 `LockReleaseSession(USER_LOCKMETHOD)`。
`LockReleaseSession()` 扫描 `LockMethodLocalHash`，只处理 `LOCALLOCK_LOCKMETHOD(*locallock) == USER_LOCKMETHOD` 的 entry，然后调用 `ReleaseLockIfHeld(locallock, true)`。
这里的 `true` 仍然表示 session owner。
所以 `pg_advisory_unlock_all()` 不是“释放所有 advisory locks”，而是释放当前 backend 持有的所有 session-level advisory locks。
transaction-level advisory locks 仍由事务 cleanup 处理。
### 5.7 transaction end：top-level commit / abort
top-level transaction 的锁释放从 `xact.c` 进入 `ResourceOwnerRelease()`：
```text
ResourceOwnerRelease(TopTransactionResourceOwner,
                     RESOURCE_RELEASE_LOCKS,
                     isCommit,
                     true)
```
`resowner.c` 在 top-level 且 owner 是 `TopTransactionResourceOwner` 时调用 `ProcReleaseLocks(isCommit)`。
`ProcReleaseLocks()` 做三件事：
```text
LockErrorCleanup()
LockReleaseAll(DEFAULT_LOCKMETHOD, !isCommit)
LockReleaseAll(USER_LOCKMETHOD, false)
```
第一步如果当前 backend 正在 wait queue 中因为 ERROR 或 cancel 退出等待，先清理 wait state。
第二步处理 standard locks。
第三步是本节核心：无论 commit 还是 abort，`USER_LOCKMETHOD` 都传 `allLocks=false`。
含义是：
```text
释放 transaction-level advisory locks；
保留 session-level advisory locks。
```
这解释了四个现象：
- `COMMIT` 后 xact advisory lock 消失。
- `ROLLBACK` 后 xact advisory lock 消失。
- `COMMIT` 后 session advisory lock 仍在。
- `ROLLBACK` 后 session advisory lock 仍在。
### 5.8 `LockReleaseAll(USER_LOCKMETHOD, false)` 如何保留 session hold
`LockReleaseAll()` 扫描本 backend 的 `LOCALLOCK` table。
当 `allLocks=false` 时，它不会简单删除 entry，而是扫描 `lockOwners[]`。
如果发现 `owner == NULL` 的 session entry，就把它移到 array 位置 0；同时对非 NULL owner 调用 `ResourceOwnerForgetLock()`。
然后判断：
```text
如果 session owner 仍有 nLocks:
  locallock->nLocks = session nLocks
  locallock->numLockOwners = 1
  保留 LOCALLOCK
  不释放 shared LOCK/PROCLOCK
否则:
  删除 LOCALLOCK
  释放 shared hold
```
如果同一 backend 同一 key 同一 mode 同时有 session hold 和 xact hold，transaction end 只减少本地 owner 计数。
shared `PROCLOCK` 仍然表示这个 backend 持有 lock。
只有最后一个 local hold 消失，才会 `UnGrantLock()`。
### 5.9 backend shutdown 与 `DISCARD ALL`
session-level advisory lock 不随事务结束释放，但 backend 退出时不能把 shared lock table 留脏。
`postinit.c` 的 `ShutdownPostgres()` 先调用 `AbortOutOfAnyTransaction()`，再显式调用：
```text
LockReleaseAll(USER_LOCKMETHOD, true)
```
这里 `allLocks=true`，表示释放当前进程持有的所有 USER locks，包括 session-level advisory locks。
这也解释了 `ProcKill()` 的 assert：到释放 `PGPROC` shared state 前，锁应该已经被上层 shutdown cleanup 释放掉。
`DISCARD ALL` 在 `commands/discard.c` 中也调用：
```text
LockReleaseAll(USER_LOCKMETHOD, true)
```
这对连接池很关键。
session-level advisory lock 是 backend session 状态；连接复用时，如果不清理 session 状态，下一个逻辑请求可能继承同一个 backend 的 advisory lock。
`DISCARD ALL` 不能在 transaction block 中执行，所以它不是事务内清理 xact advisory lock 的工具。
### 5.10 subtransaction commit / abort
subtransaction commit 中，`ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS, true, false)` 不释放普通 locks，而是调用 `LockReassignCurrentOwner(locks, nlocks)` 把当前 subtransaction owner 的 lock hold 转移给 parent resource owner。
```sql
BEGIN;
SAVEPOINT s;
SELECT pg_advisory_xact_lock(100);
RELEASE SAVEPOINT s;
-- lock 仍然持有，直到 top-level COMMIT/ROLLBACK
```
subtransaction abort 中，`ResourceOwnerRelease(... false, false)` 调用 `LockReleaseCurrentOwner(locks, nlocks)`。
```sql
BEGIN;
SAVEPOINT s;
SELECT pg_advisory_xact_lock(100);
ROLLBACK TO SAVEPOINT s;
-- 如果 hold 只属于这个 subxact，则释放
```
session-level advisory lock 不绑定 `ResourceOwner`，因此 savepoint rollback 不会释放它。
### 5.11 wait path：session/xact 不影响等待图结构
如果 advisory lock 发生冲突，进入普通 heavyweight lock wait path：
```text
LockAcquireExtended()
  -> SetupLockInTable()
  -> LockCheckConflicts()
  -> JoinWaitQueue()
  -> WaitOnLock()
  -> ProcSleep()
```
等待状态写在 `PGPROC` 中，包括 `waitLock`、`waitProcLock`、`waitLockMode`、`waitStart`、`waitStatus`。
deadlock detector 只需要 `LOCK`、`PROCLOCK`、`PGPROC`、wait queue 和 conflict matrix。
它不需要理解 advisory key 的业务含义，也不需要理解 session/xact owner。
owner 边界只影响释放，不影响冲突图。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建
advisory lock 的 shared state 在第一次需要进入 main lock table 时创建。
创建入口是 `LockAcquireExtended()`。
它通过 `SetupLockInTable()` 找到或创建 shared `LOCK` 和 shared `PROCLOCK`，同时在 backend-local `LockMethodLocalHash` 中找到或创建 `LOCALLOCK` 和 `LOCALLOCKOWNER` entry。
SQL key 本身不需要 catalog entry。
advisory lock 不写系统表、不写 WAL，是运行时 shared memory 状态。
### 6.2 谁持有
从全局冲突角度看，holder 是 `PGPROC`，因为 `PROCLOCK.tag.myProc` 指向持有者。
从本 backend cleanup 角度看，holder 是 `LOCALLOCKOWNER.owner`。
两者要一起看。
同一个 backend 可以有：
```text
shared PROCLOCK:
  holdMask includes ExclusiveLock
local LOCALLOCK:
  owner=NULL, nLocks=1
  owner=TopTransactionResourceOwner, nLocks=1
```
这表示 shared 层只看到一个 backend 持有 `ExclusiveLock`，local 层知道其中一份 hold 是 session，另一份 hold 是 transaction。
### 6.3 谁释放 session-level advisory lock
session-level advisory lock 的释放入口有四类：
- 显式 unlock：`pg_advisory_unlock()`、`pg_advisory_unlock_shared()`。
- 显式批量 unlock：`pg_advisory_unlock_all()`。
- session reset：`DISCARD ALL`。
- backend shutdown：`ShutdownPostgres()` 调用 `LockReleaseAll(USER_LOCKMETHOD, true)`。
这些入口共同点是：它们最终必须以 session owner 为目标，或者使用 `allLocks=true`。
普通 transaction end 不释放 session-level advisory lock。
### 6.4 谁释放 transaction-level advisory lock
transaction-level advisory lock 的释放入口有三类：
- top-level transaction end：`ProcReleaseLocks()` 调用 `LockReleaseAll(USER_LOCKMETHOD, false)`。
- subtransaction abort：`ResourceOwnerRelease(... false, false)` 调用 `LockReleaseCurrentOwner(...)`。
- prepared transaction finish：`lock_twophase_postcommit()` / `lock_twophase_postabort()` 调用 `LockRefindAndRelease(...)`。
没有 SQL unlock 函数。
`pg_advisory_unlock()` 不释放 xact owner。
### 6.5 ERROR / abort 时谁兜底
如果 ERROR 发生在等待期间，必须先离开 wait queue。
`ProcReleaseLocks()` 开头调用 `LockErrorCleanup()`，`postgres.c` 的若干错误恢复路径也会调用它。
这个函数处理的是 wait state，不是释放所有 advisory lock 的函数。
等 wait queue 清理完成后，transaction abort 继续走 `ResourceOwnerRelease()`。
对 xact advisory lock，最终释放在 `RESOURCE_RELEASE_LOCKS` 阶段。
对 session advisory lock，abort 不释放。
如果 ERROR 发生在 `pg_advisory_unlock()` 之后，unlock 已经生效，因为 session-level unlock 不服从事务回滚。
### 6.6 长期对象如何失效
advisory lock 没有 catalog invalidation、relcache invalidation 或 MVCC visibility。
它的“失效”只有释放。
释放后如果没有 holder 和 waiter，`CleanUpLock()` 可以回收 shared `LOCK` / `PROCLOCK` entry。
如果仍有 waiters，release path 会调用 `ProcLockWakeup()` 尝试授予等待者。
普通 session-level advisory lock 是 backend 运行态，不要期待 crash recovery 重放。
prepared transaction 里的 xact lock 是 2PC 特例。
### 6.7 2PC 边界
`PREPARE TRANSACTION` 会调用 lock manager 的 prepare 路径。
`AtPrepare_Locks()` 明确忽略 session-level locks 和 VXID locks，但会为 transaction-level locks 写 2PC state。
`PostPrepare_Locks()` 把这些 lock 从当前 backend 转交给 prepared transaction 的 dummy `PGPROC`。
如果同一个 lockable object 同时有 session-level 和 transaction-level hold，`CheckForSessionAndXactLocks()` 会报错。
错误语义是：
```text
cannot PREPARE while holding both session-level and transaction-level locks on the same object
```
源码解释了原因：当前 `PROCLOCK` 不能同时安全表达“同一 backend 的 session hold 还留在 backend、transaction hold 转给 dummy proc”。
为了避免在 `PostPrepare_Locks()` 中需要额外 `PROCLOCK` 且可能 OOM，系统选择提前 ERROR。
## 7. 正确性机制层次
advisory lock 的正确性由几层机制组合出来。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| 对象身份 | `LOCKTAG_ADVISORY` | 同 database、同 key family、同 key 才冲突。 | 不保证应用真的按约定使用 key。 |
| 冲突规则 | `USER_LOCKMETHOD` + `LockConflicts[]` | shared / exclusive 兼容性。 | 不表达释放边界。 |
| 共享状态 | `LOCK` / `PROCLOCK` | 所有 backend 对 holder / waiter 有一致视图。 | 不记录 session/xact owner。 |
| 本地 ownership | `LOCALLOCKOWNER.owner` | 同 backend 内区分 session hold 和 xact hold。 | 其他 backend 不能直接诊断。 |
| 事务 cleanup | `ResourceOwner` | xact-level hold 在 abort / commit / subxact 边界释放或转移。 | 不管理 session-level hold。 |
| 等待 cleanup | `LockErrorCleanup()` | ERROR / cancel 后从 wait queue 撤出。 | 不替代正常 release。 |
| 进程 cleanup | `ShutdownPostgres()` | backend 退出前释放 USER session locks。 | 不让 session lock 具备 crash 持久性。 |
| 2PC | `AtPrepare_Locks()` / `PostPrepare_Locks()` | xact locks 可随 prepared xact 保留。 | 同对象 session+xact 混持不能 prepare。 |
关键不变量有五个。
```text
同一 advisory key 的 session/xact hold 进入同一个 shared LOCK。
session/xact 差异只在 local owner 和 cleanup 路径。
owner == NULL 永远表示 session-level hold。
ResourceOwner 只记 owner != NULL 的 lock hold。
pg_locks 是 shared state 投影，不是 LOCALLOCKOWNER 投影。
```
这些不变量解释了大部分线上误判。
例如，看到 `pg_locks` 中 `locktype='advisory'` 不足以判断它是否会在 rollback 后消失；需要看它是如何获取的，或做 rollback 观测。
### 7.1 为什么不用事务 ID 直接表达 xact advisory lock
一种看似简单的设计是把 transaction-level advisory lock 挂到 XID 上。
PostgreSQL 没这么做，因为 heavyweight lock manager 已经有 `ResourceOwner`。
事务、子事务、portal 和 ERROR cleanup 都围绕 ResourceOwner 释放外部资源。
如果 advisory lock 另起一套 XID cleanup，就会复制 subtransaction 转移、ERROR cleanup 和 2PC 边界。
当前实现把 xact-level advisory lock 当作普通 lock resource。
差异只在 `USER_LOCKMETHOD` 和 `LOCKTAG_ADVISORY`。
### 7.2 为什么 `pg_locks` 不暴露 session/xact
`pg_locks` 的主要数据来自 shared lock manager。
shared `LOCK` / `PROCLOCK` 不保存 local owner。
要暴露 session/xact，需要跨 backend 读取每个 backend 的 `LOCALLOCK.lockOwners`。
这会引入三个问题。
第一，`LOCALLOCK` 是 backend-local 内存，不是 shared memory public contract。
第二，其他 backend 不能安全直接解引用目标 backend 的 local 指针。
第三，即使复制出来，owner 指针也只在本进程有意义。
所以 `pg_locks` 暴露的是稳定共享事实：谁持有或等待哪个 locktag 的哪个 mode。
它不暴露本地释放边界。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 `dontWait` 冲突
try advisory 函数传 `dontWait=true`。
如果冲突，`LockAcquireExtended()` 不进入睡眠等待，而是返回 `LOCKACQUIRE_NOT_AVAIL`，SQL wrapper 返回 `false`。
此时不会建立成功 hold。
如果已经创建了中间 shared request 状态，lock manager 必须撤销 request counts。
对调用者来说，`false` 表示没有拿到锁，不需要 unlock，也不会在事务结束时释放不存在的 hold。
### 8.2 等待期间 ERROR / cancel / deadlock
阻塞 advisory lock 和 relation lock 走同一条等待路径。
等待期间可能发生 query cancel、statement timeout、lock timeout、deadlock detected 或 backend termination。
这时必须从 `LOCK.waitProcs` 移除当前 `PGPROC`，并撤销等待中的 request / proclock 状态。
`LockErrorCleanup()` 是关键兜底。
它处理的是“等待失败后的 lock manager 状态一致性”。
如果错误发生在 lock 已授予之后，则后续按 owner release。
### 8.3 `LockRelease()` owner 不匹配
`LockRelease()` 用 `sessionLock` 决定要释放哪个 owner。
如果当前 backend 只持有 transaction-level advisory lock，却调用 `pg_advisory_unlock()`，第二步会尝试释放 session owner。
由于本 backend 没有 `owner == NULL` 的 hold，它不会释放 xact hold。
正确诊断要查：
```text
pg_advisory_unlock() 的返回值；
当前事务结束后 pg_locks 是否消失；
LOCALLOCK owner 规则。
```
### 8.4 事务 rollback 不回滚 session unlock
session-level unlock 本身不服从事务语义。
```sql
SELECT pg_advisory_lock(1);
BEGIN;
SELECT pg_advisory_unlock(1);
ROLLBACK;
```
rollback 不会恢复这把 session lock。
原因是 unlock 调用 `LockRelease(..., sessionLock=true)`，直接修改 lock manager 运行态。
这里没有 WAL、MVCC undo 或事务回滚日志。
### 8.5 lock table shared memory 耗尽
advisory locks 和 regular locks 使用同一套 lock shared memory pool。
大小受 `max_locks_per_transaction`、连接数和 prepared transaction 等因素影响。
如果大量不同 advisory keys 被持有，可能耗尽 lock table。
典型错误指向：
```text
out of shared memory
You might need to increase "max_locks_per_transaction".
```
这不是 advisory lock key 太大，而是不同 lock objects 太多。
重复获取同一 key 主要增加 local count，不线性增加 shared `LOCK` objects。
### 8.6 同对象 session+xact 混持下的 PREPARE
`PREPARE TRANSACTION` 前会调用 `CheckForSessionAndXactLocks()`。
它按 `LOCKTAG` 而不是 `LOCALLOCKTAG` 建临时表，因为 `LOCALLOCKTAG` 包含 mode。
同一 advisory key 的 shared 和 exclusive mode 可能是两个 local entries，但 prepare 风险是同一个 lockable object 上既有 session hold 又有 xact hold。
如果同一 `LOCKTAG` 同时有：
```text
sessLock = true
xactLock = true
```
则 ERROR。
这是 `PROCLOCK` 迁移到 prepared transaction dummy proc 的实现边界。
### 8.7 backend crash
普通 session-level advisory lock 不持久化。
backend crash 后 postmaster 会清理该 backend 的 shared resources。
用户不应该把 session advisory lock 当成 crash-safe lease。
如果需要 crash 后可恢复的 ownership，应该写业务表、使用事务状态或外部协调系统。
## 9. 成本、资源与跨模块传播
### 9.1 shared lock table 成本
每个不同 advisory lock object 可能需要一个 `LOCK`、一个或多个 `PROCLOCK`、backend-local `LOCALLOCK` 和 `LOCALLOCKOWNER` array entry。
不同 key 的数量越多，shared lock hash 压力越大。
同一 key 被同一 backend 重复获取，主要增加 local `nLocks`。
同一 key 被多个 backend 获取 shared lock，会增加不同 backend 的 `PROCLOCK`。
高频短期 xact advisory lock 如果 key 分散，会制造 lock partition LWLock contention。
它不会走 relation fast path。
### 9.2 等待与 deadlock 成本
advisory lock 等待进入普通 heavyweight wait queue。
等待成本包括修改 shared lock hash、设置 `PGPROC` wait fields、latch sleep / wakeup、deadlock timeout 后运行 deadlock detector。
如果应用用 advisory locks 表达复杂业务图，可能制造真实 deadlock。
deadlock detector 不懂业务含义，只看 lock wait graph。
业务层必须保证 key ordering。
### 9.3 ResourceOwner 成本
xact-level advisory locks 会进入 `ResourceOwner` lock cache。
`ResourceOwnerRememberLock()` 最多缓存一定数量的 `LOCALLOCK *`。
超过阈值后变成 overflow，cleanup 时可能退化为扫描整个 `LockMethodLocalHash`。
所以大量 xact-level advisory locks 的 cleanup 成本不只是 shared lock release，还包括 ResourceOwner 释放路径的索引退化。
### 9.4 连接池传播
session-level advisory lock 的状态绑定 backend。
连接池复用 backend 时，状态可能传播给下一个逻辑请求。
如果业务在 transaction pooling 模式下使用 session-level advisory lock，会产生边界错觉：客户端以为 session 结束了，PostgreSQL backend 实际没结束。
因此 transaction pooling 下应优先使用 xact-level advisory lock，或确保池化器执行可靠 reset。
`DISCARD ALL` 能释放 USER locks，但成本较高，并且不能在事务块内执行。
### 9.5 与相邻模块的连接
advisory lock 至少连接六个模块。
- lock manager 提供 `LOCKTAG`、mode conflict、wait queue、deadlock detection 和 shared hash。
- ResourceOwner 提供 xact-level cleanup 和 subtransaction 转移。
- transaction manager 决定 top-level commit/abort/subcommit/subabort 何时释放 owner。
- process lifecycle 通过 `ShutdownPostgres()` 和 `ProcKill()` 决定 backend exit 前必须释放 shared locks。
- SQL observability 通过 `lockfuncs.c` 和 `pg_locks` 把 shared state 投影给用户。
- 2PC 通过 `AtPrepare_Locks()` 和 `PostPrepare_Locks()` 决定 xact locks 能否跨 session 变成 prepared xact 的状态。
### 9.6 不涉及 WAL 与 MVCC 的原因
普通 advisory lock 不写 WAL，不参与 MVCC visibility，也不保护 tuple version。
它只是 lock manager 里的运行时同步对象。
如果 advisory lock 引发 bloat，通常是业务持锁期间延长事务时间，那是 workload pattern 的副作用，不是 advisory lock shared state 的直接属性。
## 10. 观测与诊断入口
### 10.1 最小 runtime truth
本节锚定的 runtime truth 是：
```text
同一个 backend 中，
session-level advisory lock 在 ROLLBACK 后仍留在 pg_locks；
transaction-level advisory lock 在 ROLLBACK 后消失；
但两者在持有期间都显示为 locktype='advisory'。
```
ROLLBACK 后 session lock 仍在，是因为 `owner == NULL`、`ResourceOwner` 不管理、`ProcReleaseLocks()` 对 `USER_LOCKMETHOD` 使用 `allLocks=false`。
ROLLBACK 后 xact lock 消失，是因为 `owner == CurrentResourceOwner`，`ResourceOwnerRelease(... LOCKS ...)` 会释放它。
`pg_locks` 都显示 `advisory`，是因为共享 `LOCKTAG` 相同，且 `pg_locks` 不投影 `LOCALLOCKOWNER`。
### 10.2 `pg_locks` 查询模板
基础查询：
```sql
SELECT locktype,
       database,
       classid,
       objid,
       objsubid,
       virtualtransaction,
       pid,
       mode,
       granted,
       fastpath,
       waitstart
FROM pg_locks
WHERE locktype = 'advisory'
ORDER BY pid NULLS LAST, granted DESC, mode, classid, objid, objsubid;
```
解释列时要注意：
- `classid` 是 advisory key part 1。
- `objid` 是 advisory key part 2。
- `objsubid` 是 key family。
- `database` 是 `MyDatabaseId`。
- `mode` 是 `ShareLock` 或 `ExclusiveLock`。
- `granted=false` 表示等待中。
- `waitstart` 是等待开始时间。
- `fastpath` 对 advisory lock 通常不是核心指标。
`objsubid=1` 时，key 来自 `int8`。
`objsubid=2` 时，key 来自两个 `int4`。
如果要从 `int8` 还原 key，需要把 `classid` 和 `objid` 当作高低 32 bit，并注意 signed / unsigned 显示差异。
### 10.3 判断 session-level 还是 xact-level
`pg_locks` 不能直接判断 owner。
可用方法是行为验证。
方法一：
```sql
BEGIN;
SELECT pg_advisory_lock(9001);
ROLLBACK;
SELECT * FROM pg_locks WHERE locktype = 'advisory';
```
如果 lock 仍在，说明它是 session-level hold。
方法二：
```sql
BEGIN;
SELECT pg_advisory_xact_lock(9002);
ROLLBACK;
SELECT * FROM pg_locks WHERE locktype = 'advisory';
```
如果 lock 消失，说明它是 xact-level hold。
方法三：
```sql
SELECT pg_advisory_unlock_all();
```
观察 session-level locks 是否消失。
注意不要在事务块中用 `DISCARD ALL`。
### 10.4 观察等待者
构造两个 session。
Session A：
```sql
SELECT pg_advisory_lock(42);
```
Session B：
```sql
SELECT pg_advisory_xact_lock(42);
```
Session B 会等待。
查询：
```sql
SELECT pid, locktype, mode, granted, waitstart
FROM pg_locks
WHERE locktype = 'advisory';
```
你会看到同一 key 上一行 granted=true，一行 granted=false。
这说明 session-level holder 和 xact-level waiter 进入同一共享等待图。
再查：
```sql
SELECT pg_blocking_pids(<blocked_pid>);
```
可以看到 blocker pid。
`pg_blocking_pids()` 依赖 lock manager 的 conflict graph，不需要知道 advisory key 的业务含义。
### 10.5 wait event 与日志
阻塞 advisory lock 时，`pg_stat_activity.wait_event_type` 通常是 `Lock`。
wait event 只说明当前在等待 lock manager，不说明 holder 是 session-level 还是 xact-level，不说明 key 的业务含义，也不说明是否存在应用层顺序反转。
如果怀疑 advisory lock wait 过长，可以设置：
```sql
SET lock_timeout = '5s';
SET deadlock_timeout = '1s';
```
`lock_timeout` 会取消等待，取消后 `LockErrorCleanup()` 清理 wait state。
如果配置了 `log_lock_waits`，日志会帮助定位等待对象，但日志中的 locktag 字段仍然需要按 advisory mapping 解码。
### 10.6 prepared transaction 诊断
如果 xact-level advisory lock 被 `PREPARE TRANSACTION` 保留，它会转移到 dummy `PGPROC`。
这时 `pg_locks.pid` 可能为空。
应结合：
```sql
SELECT * FROM pg_prepared_xacts;
```
如果 advisory lock 的 `pid` 为空且系统存在 prepared transactions，不要误判为 orphan lock。
它可能是 prepared xact 持有。
释放边界是 `COMMIT PREPARED` 或 `ROLLBACK PREPARED`。
### 10.7 能直接看到、只能推断、看不到
能直接看到：
- `locktype='advisory'`
- key fields
- `mode`
- holder / waiter pid
- `granted`
- `waitstart`
- blocker pid
- prepared xact 是否存在
只能推断：
- session-level 还是 xact-level。
- 应用层 key 含义。
- 是否重复获取了同一 key 多次。
- holder 是否会在当前事务结束释放。
基本看不到：
- 目标 backend 的 `LOCALLOCK.lockOwners[]`。
- `owner == NULL` 或具体 `ResourceOwner` 指针。
- 同一 backend 内某 key 的 session hold count 和 xact hold count。
所以 `pg_locks` 是 shared state 视图，不是 backend-local owner dump。
## 11. 常见误区
### 误区一：`pg_advisory_unlock_all()` 会释放 xact advisory lock
不会。它调用 `LockReleaseSession(USER_LOCKMETHOD)`，目标是 session owner，xact owner 等事务结束释放。
### 误区二：`ROLLBACK` 会撤销 session-level `pg_advisory_lock()`
不会。session-level acquire 的 owner 是 `NULL`，它不进入 `ResourceOwner`。
### 误区三：`pg_locks` 能告诉我这是 session lock 还是 xact lock
不能直接告诉。`pg_locks` 显示 shared lock state，session/xact owner 在 backend-local `LOCALLOCKOWNER` 中。
### 误区四：同一 backend 再次获取自己持有的 lock 应该等待
不会。同一 backend 的重复请求会增加 local count。
### 误区五：advisory lock 是 crash-safe 业务租约
普通 session advisory lock 不是。backend 退出会释放，server crash 后不会保留普通 session lock。
### 误区六：advisory lock 不会影响 lock table
会。它使用 shared lock manager，大量不同 advisory keys 会消耗 lock shared memory。
### 误区七：`ShareLock` 表示“共享到事务外”
不是。`ShareLock` 是 lock mode，session / transaction 是 owner boundary。
### 误区八：`objsubid` 是业务子对象编号
对 advisory lock 来说不是。它是 key family 标记，`1` 表示 int8，`2` 表示两个 int4。
## 12. 课堂实验
### 实验一：ROLLBACK 边界
目标：看到 session-level 和 xact-level advisory lock 的释放差异。
Session A 执行：
```sql
BEGIN;
SELECT pg_advisory_lock(101);
ROLLBACK;
SELECT locktype, classid, objid, objsubid, mode, granted
FROM pg_locks
WHERE locktype = 'advisory';
```
预期：lock 仍然存在。
源码解释：
```text
pg_advisory_lock_int8()
  -> LockAcquire(... sessionLock=true ...)
  -> owner = NULL
  -> GrantLockLocal(... owner=NULL ...)
  -> ROLLBACK
  -> LockReleaseAll(USER_LOCKMETHOD, false)
  -> 保留 owner=NULL
```
清理：
```sql
SELECT pg_advisory_unlock_all();
```
再执行：
```sql
BEGIN;
SELECT pg_advisory_xact_lock(102);
ROLLBACK;
SELECT locktype, classid, objid, objsubid, mode, granted
FROM pg_locks
WHERE locktype = 'advisory';
```
预期：lock 消失。
源码解释：
```text
pg_advisory_xact_lock_int8()
  -> LockAcquire(... sessionLock=false ...)
  -> owner = CurrentResourceOwner
  -> ResourceOwnerRememberLock()
  -> ROLLBACK
  -> ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)
  -> ProcReleaseLocks(false)
  -> LockReleaseAll(USER_LOCKMETHOD, false)
```
### 实验二：同 key session holder 阻塞 xact waiter
目标：证明 session/xact 进入同一冲突图。
Session A：
```sql
SELECT pg_advisory_lock(201);
```
Session B：
```sql
BEGIN;
SELECT pg_advisory_xact_lock(201);
```
Session C：
```sql
SELECT pid, locktype, classid, objid, objsubid, mode, granted, waitstart
FROM pg_locks
WHERE locktype = 'advisory'
ORDER BY granted DESC, pid;
```
预期：同一 key 上有 holder 和 waiter，waiter 的 `granted=false`。
再查：
```sql
SELECT pid, pg_blocking_pids(pid)
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```
源码解释：两个 SQL wrapper 生成同一个 `LOCKTAG_ADVISORY`，session/xact 只影响 `LOCALLOCKOWNER`，`LockCheckConflicts()` 只看 shared lock state。
释放：
```sql
-- Session A
SELECT pg_advisory_unlock(201);
-- Session B
ROLLBACK;
```
### 实验三：同 backend 混合 session 和 xact owner
目标：观察同 key 同 mode 同 backend 的混合 owner。
```sql
SELECT pg_advisory_lock(301);
BEGIN;
SELECT pg_advisory_xact_lock(301);
COMMIT;
SELECT locktype, classid, objid, objsubid, mode, granted
FROM pg_locks
WHERE locktype = 'advisory';
```
预期：`COMMIT` 后 lock 仍存在。
源码解释：同一 `LOCALLOCK` 有 `owner=NULL` 和 `owner=CurrentResourceOwner`；commit 释放 xact owner，session owner 仍有 `nLocks`，shared `PROCLOCK` 不能释放。
再执行：
```sql
SELECT pg_advisory_unlock(301);
```
预期：lock 消失。
### 实验四：subtransaction 边界
Case A：
```sql
BEGIN;
SAVEPOINT s;
SELECT pg_advisory_xact_lock(401);
RELEASE SAVEPOINT s;
SELECT count(*) FROM pg_locks WHERE locktype = 'advisory';
ROLLBACK;
```
预期：`RELEASE SAVEPOINT` 后仍持有，top-level `ROLLBACK` 后释放。
源码解释：subcommit 调用 `LockReassignCurrentOwner()`。
Case B：
```sql
BEGIN;
SAVEPOINT s;
SELECT pg_advisory_xact_lock(402);
ROLLBACK TO SAVEPOINT s;
SELECT count(*) FROM pg_locks WHERE locktype = 'advisory';
ROLLBACK;
```
预期：`ROLLBACK TO SAVEPOINT` 后释放。
源码解释：subabort 调用 `LockReleaseCurrentOwner()`。
### 实验五：2PC 混合 owner 报错
前提：
```sql
ALTER SYSTEM SET max_prepared_transactions = 10;
-- 重启后生效。
```
实验：
```sql
BEGIN;
SELECT pg_advisory_lock(501);
SELECT pg_advisory_xact_lock(501);
PREPARE TRANSACTION 'adv-mixed';
```
预期：
```text
ERROR: cannot PREPARE while holding both session-level and transaction-level locks on the same object
```
源码解释：
```text
AtPrepare_Locks()
  -> CheckForSessionAndXactLocks()
  -> 按 LOCKTAG 聚合
  -> sessLock && xactLock
  -> ERROR
```
清理：
```sql
ROLLBACK;
SELECT pg_advisory_unlock_all();
```
### 实验六：gdb 断点
建议断点：
```text
lockfuncs.c:
  pg_advisory_lock_int8
  pg_advisory_xact_lock_int8
lock.c:
  LockAcquireExtended
  GrantLockLocal
  LockReleaseAll
  ReleaseLockIfHeld
proc.c:
  ProcReleaseLocks
resowner.c:
  ResourceOwnerReleaseInternal
```
观察变量：
```text
sessionLock
owner
locktag->locktag_type
locktag->locktag_lockmethodid
locallock->nLocks
locallock->numLockOwners
locallock->lockOwners[i].owner
locallock->lockOwners[i].nLocks
allLocks
```
断点问题：
- `pg_advisory_lock(1)` 时 `owner` 是否为 `NULL`？
- `pg_advisory_xact_lock(1)` 时 `owner` 是否为 `CurrentResourceOwner`？
- `ROLLBACK` 时 `LockReleaseAll(USER_LOCKMETHOD, false)` 是否保留 `owner == NULL`？
- `pg_advisory_unlock_all()` 是否调用 `LockReleaseSession(USER_LOCKMETHOD)`？
## 13. 讨论题
1. 为什么 advisory lock 要使用 `USER_LOCKMETHOD`，而不是 DEFAULT lock method 中的 object lock？
2. 如果 `pg_locks` 不显示 session/xact owner，线上如何判断某个 advisory lock 是否会在 rollback 后释放？
3. 为什么 `owner == NULL` 可以安全表示 session-level hold？这个约定需要哪些 release 路径共同遵守？
4. 为什么 `ProcReleaseLocks()` 对 `USER_LOCKMETHOD` 在 commit 和 abort 时都传 `allLocks=false`？
5. 同一 backend 同一 key 同一 mode 同时有 session hold 和 xact hold 时，commit 为什么不能释放 shared `PROCLOCK`？
6. `PREPARE TRANSACTION` 为什么不能处理同一对象上的 session+xact 混合 hold？
7. 如果应用在连接池 transaction pooling 模式下使用 session-level advisory lock，会出现什么状态泄漏？
8. 大量不同 advisory keys 会把成本推到哪些结构和锁上？
9. `pg_advisory_unlock_all()` 为什么不应该被理解成事务内 cleanup API？
10. 如果要做 crash-safe lease，为什么 advisory lock 不够？
## 14. 本节小结
本节的核心链路是：
```text
SQL advisory function
  -> SET_LOCKTAG_ADVISORY
  -> LockAcquire(sessionLock)
  -> owner = NULL 或 CurrentResourceOwner
  -> GrantLockLocal()
  -> shared LOCK/PROCLOCK 表达冲突
  -> LOCALLOCKOWNER 表达释放边界
  -> ResourceOwner / LockReleaseAll / ShutdownPostgres 清理
```
核心状态是 `LOCALLOCKOWNER.owner`。
`owner == NULL` 表示 session-level。
`owner != NULL` 表示 transaction-level，并由 `ResourceOwner` 管理。
共享 lock table 不直接区分 session/xact，它只表达同一 `LOCKTAG_ADVISORY` 上的 holder、waiter 和 mode。
释放边界分三类。
transaction end 调用 `LockReleaseAll(USER_LOCKMETHOD, false)`，释放 xact owner，保留 session owner。
explicit session unlock 调用 `LockRelease(..., sessionLock=true)` 或 `LockReleaseSession(USER_LOCKMETHOD)`，释放 `owner == NULL`。
backend shutdown 或 `DISCARD ALL` 调用 `LockReleaseAll(USER_LOCKMETHOD, true)`，释放当前 backend 的所有 USER locks。
诊断时要记住：
```text
pg_locks 能看到 advisory lock shared state；
pg_locks 不能直接看到 LOCALLOCKOWNER。
```
能直接观测的是 locktype、key fields、pid、mode、granted、waitstart。
session/xact 边界通常要通过 rollback/commit 行为、应用调用路径或断点推断。
本节可迁移的系统规律是：
```text
一个共享同步对象的冲突语义和释放 owner 可以分层设计。
共享层负责全局可见性和等待图；
本地 owner 层负责生命周期边界。
```
这个规律不仅适用于 advisory lock，也适用于理解 PostgreSQL 中许多“同一 shared object，不同 cleanup boundary”的资源管理设计。
