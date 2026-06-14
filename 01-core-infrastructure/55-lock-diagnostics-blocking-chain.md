# PostgreSQL lock diagnostics 与 blocking chain
## 课程定位
前置知识：已经理解 `PGPROC`、`ProcArray`、`LOCALLOCK`、`LOCK`、`PROCLOCK`、fast-path relation lock、ResourceOwner、LWLock、latch、deadlock detector 的基本边界。
本节唯一主问题：
```text
如何从 pg_locks、pg_stat_activity 和 wait event 看到的阻塞现象，回到 lock manager 源码里的等待点和 blocking chain？
```
核心矛盾：
```text
线上诊断需要快速回答“谁挡住谁、挡在哪里、该不该介入”；
但 lock manager 的真实状态分散在 backend-local LOCALLOCK、shared LOCK/PROCLOCK、PGPROC wait fields、fast-path slots、wait queue 和 activity stats 中。
```
一句话运行模型：
```text
LockAcquireExtended() 判断冲突后把 PGPROC 挂入 LOCK.waitProcs；
WaitOnLock() 记录 awaitedLock 并进入 ProcSleep()；
ProcSleep() 用 PG_WAIT_LOCK | locktag_type 发布 wait event；
pg_locks 通过 GetLockStatusData() 拷贝 hold/wait 状态；
pg_blocking_pids() 通过 GetBlockerStatusData() 对同一 wait queue 计算 hard blocker 和 soft blocker；
pg_stat_activity 只展示 PGPROC.wait_event_info 的当前等待类别和名称。
```
学完后应能判断：
- `pg_locks.granted=false` 对应哪个 `PGPROC` wait field。
- `pg_locks.waitstart` 为什么可能短暂为空。
- `pg_stat_activity.wait_event_type='Lock'` 为什么只能说明当前正在 heavyweight lock wait。
- `pg_blocking_pids(pid)` 为什么会返回已经在等待队列前方的 waiter。
- fast-path lock 为什么可能出现在 `pg_locks`，却通常不会直接成为 contended lock 的 blocker snapshot。
- blocking chain 如何从 SQL 层递归展开，并回到 `LOCK.waitProcs`、`PROCLOCK.holdMask` 和 conflict table。
- lock timeout、deadlock、cancel、backend exit 分别由哪些 cleanup 路径收尾。
- 哪些信息能直接观测，哪些只能从快照和时间顺序近似推断。
本课基于本地源码：
```text
/home/nail/postgres
branch master
commit 0e1f1ed157e9
```
版本注意：本地源码中没有 `src/backend/utils/activity/pgstat_activity.c`。
`pg_stat_activity` 的 SQL 函数投影在 `src/backend/utils/adt/pgstatfuncs.c`，
wait event 名称解析在 `src/backend/utils/activity/wait_event.c` 和 `wait_event_names.txt`。
这不改变本课主线，但避免按旧资料去找不存在的文件。
## 1. 本节在总主线中的位置
前面几节已经回答三组问题。
第 49 节说明 `LOCKTAG` 如何把 relation、tuple、transactionid、advisory 等对象放进同一套 lock manager。
第 50 节说明 `LockAcquireExtended()` / `LockRelease()` 如何维护 `LOCALLOCK`、fast path、main lock table、wait queue 和 ResourceOwner。
第 51 节说明 weak relation lock 为什么可以先走 fast path，何时必须迁回 main lock table。
本节站在诊断入口反向看同一套状态。
线上你通常不是从源码入口开始。
你先看到的是：
```sql
select pid, wait_event_type, wait_event, state, query
from pg_stat_activity
where wait_event_type = 'Lock';
```
或者：
```sql
select *
from pg_locks
where not granted;
```
再或者：
```sql
select pid, pg_blocking_pids(pid)
from pg_stat_activity
where cardinality(pg_blocking_pids(pid)) > 0;
```
这些入口都不是完整事实。
`pg_stat_activity` 只告诉你 backend 当前发布的 wait event。
`pg_locks` 是 lock manager 状态的一次拷贝。
`pg_blocking_pids()` 是针对一个 blocked PID 的 blocker 计算。
三者必须拼起来，才能接近一条 blocking chain。
本节的主线不是“背系统视图字段”。
本节只追一个时间线：
```text
backend A 持有 relation lock
  -> backend B 请求冲突 lock
  -> B 进入 LOCK.waitProcs
  -> B 在 pg_stat_activity 中显示 Lock wait event
  -> pg_locks 显示 B granted=false
  -> pg_blocking_pids(B) 返回 A
  -> backend C 排在 B 后面
  -> pg_blocking_pids(C) 可能返回 A，也可能返回 B 作为 soft blocker
  -> release、timeout、deadlock 或 cancel 触发 wait queue cleanup
```
核心诊断能力是把这个 runtime 现象回到源码里的几个点：
```text
LockAcquireExtended()
JoinWaitQueue()
WaitOnLock()
ProcSleep()
ProcLockWakeup()
RemoveFromWaitQueue()
GetLockStatusData()
GetBlockerStatusData()
pg_lock_status()
pg_blocking_pids()
pgstat_get_wait_event_type()
pgstat_get_wait_event()
```
## 2. 核心矛盾与一句话运行模型
lock diagnostics 的 tension 可以压缩成一句话：
```text
诊断接口必须便宜、非侵入、能在并发变化中返回；
但 blocking chain 的真实语义依赖锁冲突规则、队列顺序、lock group、fast path 和正在变化的 shared memory。
```
如果诊断函数长期持有 lock partition LWLock，就会反过来干扰正在排队的业务锁。
如果诊断函数完全不持锁读取，就可能拼出不一致的 chain。
PostgreSQL 的选择是：
```text
GetLockStatusData():
  先扫每个 backend 的 fast-path slots；
  再按 partition 顺序拿所有 lock hash partition LWLock；
  拷贝 PROCLOCK / PGPROC wait fields；
  尽快释放 LWLock；
  上层 pg_lock_status() 再格式化成 pg_locks 行。

GetBlockerStatusData(pid):
  持 ProcArrayLock 稳住 PID -> PGPROC 映射；
  拿所有 lock partition LWLock；
  只复制目标 blocked proc 等待的那个 LOCK 上的 PROCLOCK 和 wait queue 前缀；
  上层 pg_blocking_pids() 用 conflictTab 判断 blocker。

pg_stat_activity:
  从 backend status 和 PGPROC.wait_event_info 取当前 wait event；
  不解释 lock table，也不计算 blocker。
```
这三个入口的边界不同。
`pg_locks` 是宽视图。
它适合看某个 locktag 上有哪些 holder 和 waiter。
但它不直接告诉你哪一个 waiter 排在谁前面。
`pg_blocking_pids()` 是窄计算。
它适合从一个 PID 出发问“现在谁挡住我”。
但它不是全局 blocking graph 的物化表。
`pg_stat_activity` 是当前状态投影。
它适合确认 backend 是否真的处在 lock wait。
但 `wait_event='relation'`、`wait_event='transactionid'` 不是对象名，而是 `LOCKTAG` 类型名。
本节要建立的 mental model 是：
```text
pg_stat_activity 说明“这个 backend 正在等哪类等待事件”；
pg_locks 说明“这个 lock manager 快照里，哪些 PROCLOCK 持有或等待哪些 mode”；
pg_blocking_pids() 说明“按当前 conflict table 和 wait queue，目标 PID 被谁硬阻塞或软阻塞”；
三者合并后，才形成可行动的 blocking chain。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/proc.h` | `PGPROC.waitLock`、`waitProcLock`、`waitLockMode`、`waitStart`、`waitStatus`、`wait_event_info`。 |
| 2 | `src/include/storage/lock.h` | `LockInstanceData`、`BlockedProcData`、`BlockedProcsData`、`DeadLockState`。 |
| 3 | `src/backend/storage/lmgr/lock.c` | `LockAcquireExtended()`、`WaitOnLock()`、`RemoveFromWaitQueue()`、`GetLockStatusData()`、`GetBlockerStatusData()`。 |
| 4 | `src/backend/storage/lmgr/proc.c` | `JoinWaitQueue()`、`ProcSleep()`、`ProcLockWakeup()`、`LockErrorCleanup()`。 |
| 5 | `src/backend/storage/lmgr/deadlock.c` | `DeadLockCheck()`、`CheckDeadLock()`、wait queue reorder 和 deadlock report。 |
| 6 | `src/backend/utils/adt/lockfuncs.c` | `pg_lock_status()` 和 `pg_blocking_pids()` 如何把内部快照变成 SQL 行和 PID array。 |
| 7 | `src/backend/utils/adt/pgstatfuncs.c` | `pg_stat_get_activity()` 如何读取 `wait_event_info` 并填充 `wait_event_type` / `wait_event`。 |
| 8 | `src/backend/utils/activity/wait_event.c` | `pgstat_get_wait_event_type()`、`pgstat_get_wait_event()` 如何解析 wait event 编码。 |
| 9 | `src/backend/utils/activity/wait_event_names.txt` | `WaitEventLock` 名称表，必须和 `LockTagTypeNames[]` 保持一致。 |
| 10 | `src/backend/catalog/system_views.sql` | `pg_locks` 和 `pg_stat_activity` 视图如何包装底层 C 函数。 |
推荐阅读顺序：
```text
proc.h 的 wait fields
  -> lock.h 的 LockInstanceData / BlockedProcsData
  -> lock.c 的 LockAcquireExtended() 冲突分支
  -> proc.c 的 JoinWaitQueue()
  -> lock.c 的 WaitOnLock()
  -> proc.c 的 ProcSleep()
  -> lock.c 的 GetLockStatusData()
  -> lockfuncs.c 的 pg_lock_status()
  -> lock.c 的 GetBlockerStatusData()
  -> lockfuncs.c 的 pg_blocking_pids()
  -> pgstatfuncs.c 的 wait_event 投影
```
不要从 `pg_locks` 文档字段开始背。
先建立一条源码链路：
```text
冲突检测
  -> wait queue 入队
  -> wait event 发布
  -> SQL 视图快照
  -> blocker 计算
  -> cleanup / wakeup
```
## 4. 关键数据结构与状态
### 4.1 `PGPROC`：等待中的 backend identity
`PGPROC` 是 blocking chain 的节点。
本节关注的字段在 `proc.h` 的 lock manager 段和 status reporting 段。
关键组合是：
| 字段 | 语义 |
| --- | --- |
| `pid` | SQL 层看到的 backend PID；prepared transaction 的 dummy `PGPROC` 可为 0。 |
| `lockGroupLeader` / `lockGroupMembers` | parallel query lock group 边界；同组成员不互相阻塞。 |
| `waitLock` | 当前正在等待的 `LOCK`；不等待时为 NULL。 |
| `waitProcLock` | 当前等待请求对应的 `PROCLOCK`。 |
| `waitLockMode` | 当前等待的 mode，例如 `AccessExclusiveLock`。 |
| `heldLocks` | 入队时记录自己在同一 lock object 上已持有的 modes。 |
| `waitStart` | lock wait 开始时间，供 `pg_locks.waitstart` 使用。 |
| `waitStatus` | `PROC_WAIT_STATUS_WAITING`、`OK`、`ERROR`。 |
| `wait_event_info` | `pg_stat_activity.wait_event_type` / `wait_event` 的原始编码。 |
这些字段不能单独解释。
`waitLock != NULL` 表示这个 backend 在 heavyweight lock wait queue 中。
`wait_event_info` 表示 backend 当前发布的 wait event，可能是 lock，也可能是 LWLock、IO、Client、Timeout 等。
`pg_locks.granted=false` 来自 `LockInstanceData.waitLockMode != NoLock`，不是来自 `wait_event_info`。
`pg_stat_activity.wait_event_type='Lock'` 来自 `wait_event_info`，不是从 `LOCK.waitProcs` 反查出来。
这两个事实通常同时出现，但不是同一个字段。
### 4.2 `LOCK` 与 `PROCLOCK`：blocking chain 的共享边
`LOCK` 是 lockable object 的 shared entry。
本节关注：
| 字段 | 语义 |
| --- | --- |
| `tag` | `LOCKTAG`，决定 `pg_locks.locktype` 和对象列。 |
| `grantMask` | 当前已授予 modes 的 bitmask。 |
| `waitMask` | 当前等待中的 modes 的 bitmask。 |
| `procLocks` | 所有对该 object 持有或等待的 backend 账本。 |
| `waitProcs` | 正在等待该 object 的 `PGPROC` 队列。 |
| `requested[]` / `granted[]` | mode 维度的请求数和授予数。 |
`PROCLOCK` 是 `LOCK + PGPROC` 的边。
本节关注：
| 字段 | 语义 |
| --- | --- |
| `tag.myLock` | 指向同一个 `LOCK`。 |
| `tag.myProc` | 指向 holder 或 waiter 的 `PGPROC`。 |
| `groupLeader` | lock group leader，用于 parallel query blocker 展示。 |
| `holdMask` | 该 backend 已经持有的 modes。 |
`pg_blocking_pids()` 的 hard block 判断本质是：
```text
blocked_instance.waitLockMode
  -> conflictTab[mode]
  -> 与其它 instance.holdMask 求交
```
soft block 判断本质是：
```text
其它 instance 也在等待冲突 mode
  且该 instance.pid 出现在 blocked proc 之前的 wait queue 前缀里
```
因此 blocking chain 不是只由 holder 决定。
排队在前面的 waiter 也可能阻塞后面的 waiter。
这解释了为什么 `pg_blocking_pids()` 的注释明确区分 hard block 和 soft block。
### 4.3 `LockInstanceData`：`pg_locks` 的内部快照行
`LockInstanceData` 定义在 `lock.h`。
它不是 shared memory 原结构。
它是 `GetLockStatusData()` 或 `GetBlockerStatusData()` 拷贝出的轻量表示。
关键字段：
```text
locktag
holdMask
waitLockMode
vxid
waitStart
pid
leaderPid
fastpath
```
`pg_lock_status()` 对一个 `LockInstanceData` 可能输出多行。
如果 `holdMask` 有多个 mode bit，会逐个输出 `granted=true` 行。
如果 `waitLockMode != NoLock`，会再输出一行 `granted=false`。
所以 `pg_locks` 的“一行”不是 `PROCLOCK` 的一比一完整拷贝。
它是按 lock mode 展开的 SQL 投影。
诊断时要注意：
```text
同一个 pid、locktype、database、relation 可能有多行；
同一个 PROCLOCK 可以既有 granted=true 行，也有 granted=false 行；
granted=false 行的 waitstart 只有在 waitStart 已经写入后才非空。
```
### 4.4 `BlockedProcsData`：`pg_blocking_pids()` 的窄快照
`BlockedProcsData` 也是 C 层临时数据。
它包含三组数组：
| 字段 | 语义 |
| --- | --- |
| `procs` | 每个被检查的 blocked proc。 |
| `locks` | 目标 blocked proc 等待的那个 `LOCK` 上的 `LockInstanceData`。 |
| `waiter_pids` | 在 blocked proc 前面的 wait queue PID 前缀。 |
和 `GetLockStatusData()` 不同，`GetBlockerStatusData(pid)` 只关心目标 PID 或其 lock group 成员正在等待的 lock。
这让 `pg_blocking_pids()` 不需要扫描全局所有 locktag 来计算一个 PID 的 blocker。
但它也意味着这个函数不输出完整 graph。
如果要展开 chain，需要在 SQL 层递归调用：
```text
blocked pid
  -> pg_blocking_pids(blocked pid)
  -> 对每个 blocker 再调用 pg_blocking_pids(blocker)
  -> 直到数组为空或检测到重复 PID
```
### 4.5 wait event 编码：类型和对象类型不是对象身份
`ProcSleep()` 调用 `WaitLatch()` 时使用：
```text
PG_WAIT_LOCK | locallock->tag.lock.locktag_type
```
因此 `pg_stat_activity.wait_event_type` 会解析为 `Lock`。
`wait_event` 会按 `locktag_type` 解析为类似 `relation`、`transactionid`、`tuple`、`virtualxid`、`advisory` 的名称。
它不会告诉你 relation OID 或 transaction ID。
对象 identity 要从 `pg_locks` 的 `database`、`relation`、`transactionid` 等列找。
这条边界非常重要。
`wait_event='relation'` 的含义是：
```text
当前 backend 正在等待 locktag_type 为 LOCKTAG_RELATION 的 heavyweight lock。
```
它不是：
```text
正在等待某个叫 relation 的内部 LWLock；
正在等待所有 relation；
正在等待 relation cache；
已经知道具体表名。
```
## 5. 主流程源码 walkthrough
### 5.1 入口：从上层 lock helper 到 `LockAcquireExtended()`
上层 wrapper 在 `lmgr.c` 构造 `LOCKTAG`。
例如 relation lock 会走 `LockRelationOid()` 或相邻 helper。
transactionid wait 会构造 `LOCKTAG_TRANSACTION`。
advisory lock 会构造 `LOCKTAG_ADVISORY`，并使用 `USER_LOCKMETHOD`。
这些 wrapper 最后进入：
```text
LockAcquireExtended(locktag, lockmode, sessionLock, dontWait, ...)
```
`LockAcquireExtended()` 先处理 local fast path：
```text
找或建 LOCALLOCK
  -> 如果 locallock->nLocks > 0，只增加本地计数
  -> 如果 weak relation lock eligible，尝试 PGPROC fast path
  -> 如果可能和 fast path 冲突，先迁移相关 fast-path locks
  -> 进入 main lock table
```
诊断角度要记住：
fast path 成功时没有 `LOCK` / `PROCLOCK`。
这种 lock 可以出现在 `pg_locks.fastpath=true`。
但真正发生等待时，请求必须进入 main lock table。
因此 blocking chain 的核心等待边在 `LOCK` / `PROCLOCK` / `PGPROC.waitLock` 上。
### 5.2 冲突检测：从 conflict table 到 wait queue
进入 main lock table 后，`LockAcquireExtended()` 做三件事。
第一，拿 lock hash partition 的 `LW_EXCLUSIVE`。
第二，通过 `SetupLockInTable()` 找或建 `LOCK` 和 `PROCLOCK`。
第三，判断是否冲突：
```text
if conflictTab[lockmode] & lock->waitMask:
    found_conflict = true
else:
    found_conflict = LockCheckConflicts(...)
```
这里的顺序很关键。
如果已有等待队列中存在冲突请求，新请求也要考虑队列公平性。
它不只是看当前 holder。
所以 `lock->waitMask` 会让后来的强请求排队，而不是绕过前面的 waiter。
如果没有冲突：
```text
GrantLock(lock, proclock, lockmode)
```
`GrantLock()` 会更新 `LOCK.granted[]`、`LOCK.grantMask` 和 `PROCLOCK.holdMask`。
如果有冲突：
```text
JoinWaitQueue(locallock, lockMethodTable, dontWait)
```
这是 SQL 观测开始和源码 wait point 之间的关键桥。
### 5.3 入队：`JoinWaitQueue()` 写入 `PGPROC` wait fields
`JoinWaitQueue()` 在 `proc.c`。
调用它时，partition LWLock 仍然由 `LockAcquireExtended()` 持有。
入口前 `SetupLockInTable()` 已经把请求计入 `LOCK.requested[]`。
`JoinWaitQueue()` 的任务是决定：
```text
能不能立即授予；
是否因 dontWait 直接失败；
应该插入 wait queue 的哪个位置；
是否发现 early deadlock；
需要写哪些 PGPROC wait fields。
```
它不是简单 append。
如果当前 backend 已经持有某些和前方 waiter 冲突的锁，它可能插到前方 waiter 之前。
这减少一些本来要等 `deadlock_timeout` 才能发现的队列冲突。
真正入队后，核心状态变化是：
```text
LOCK.waitProcs 增加 MyProc.waitLink
LOCK.waitMask |= LOCKBIT_ON(lockmode)
MyProc.heldLocks = myProcHeldLocks
MyProc.waitLock = lock
MyProc.waitProcLock = proclock
MyProc.waitLockMode = lockmode
MyProc.waitStatus = PROC_WAIT_STATUS_WAITING
```
此时 `pg_locks` 已经有机会看到一个 `granted=false` 行。
但 `pg_stat_activity.wait_event` 还不一定已经变成 Lock。
因为 wait event 是后面 `ProcSleep()` 调用 `WaitLatch()` 时发布的。
### 5.4 睡眠包装：`WaitOnLock()` 设置 ERROR cleanup 边界
`LockAcquireExtended()` 如果得到 `PROC_WAIT_STATUS_WAITING`，会释放 partition LWLock，然后调用：
```text
WaitOnLock(locallock, owner)
```
`WaitOnLock()` 做几件诊断相关的事。
第一，发 tracepoint：
```text
TRACE_POSTGRESQL_LOCK_WAIT_START(...)
```
第二，注册错误上下文。
这就是 lock timeout 或 cancel 报错里能看到：
```text
waiting for AccessExclusiveLock on relation ...
```
第三，设置进程标题后缀为 waiting。
第四，设置两个 backend-local 变量：
```text
awaitedLock = locallock
awaitedOwner = owner
```
这两个变量不是 shared memory。
它们让 `LockErrorCleanup()` 能在 ERROR、cancel、die 中知道当前正在等待哪把锁。
`WaitOnLock()` 的注释很重要：
```text
ProcSleep() 返回后不要再做必须的 shared-state cleanup。
授予锁的人或 deadlock cleanup 必须已经把 lock table 状态整理好。
```
原因是 backend 可能在被 grant 后、自己注意到之前收到 cancel。
如果 shared state 还依赖 waiter 自己返回后补写，就会留下不一致状态。
### 5.5 真正等待点：`ProcSleep()` 与 wait event
`ProcSleep()` 是用户在 `pg_stat_activity` 中看到 Lock wait event 的核心等待点。
非 Hot Standby 路径中，它会设置 deadlock timeout 和可选 lock timeout。
然后写入 `waitStart`：
```text
pg_atomic_write_u64(&MyProc->waitStart,
                    get_timeout_start_time(DEADLOCK_TIMEOUT));
```
源码注释明确说，`waitStart` 没有持 lock table partition lock 更新。
所以 `pg_locks` 可能在极短窗口内看到：
```text
granted = false
waitstart is null
```
这不是视图坏了。
这是为了避免给 wait start 时间戳增加额外锁开销。
随后 `ProcSleep()` 循环等待 latch：
```text
WaitLatch(MyLatch,
          WL_LATCH_SET | WL_EXIT_ON_PM_DEATH,
          0,
          PG_WAIT_LOCK | locallock->tag.lock.locktag_type)
```
这行就是从 `pg_stat_activity.wait_event` 回源码的最短路径。
`PG_WAIT_LOCK` 提供 wait event type。
`locktag_type` 提供 wait event name。
因此如果看到：
```text
wait_event_type = Lock
wait_event = transactionid
```
应回到：
```text
ProcSleep()
  -> WaitLatch(..., PG_WAIT_LOCK | LOCKTAG_TRANSACTION)
```
再用 `pg_locks` 找 `transactionid` 列和等待 mode。
### 5.6 唤醒：`ProcLockWakeup()` 先 grant 再 set latch
锁释放或队列重排后，`ProcLockWakeup()` 扫描 `LOCK.waitProcs`。
它检查队首开始的 waiter 是否还被 holder 或前方 waiter 阻塞。
如果可以授予，它调用：
```text
GrantLock(lock, proc->waitProcLock, lockmode)
```
然后再把等待者从队列移除并设置其状态。
这个顺序解释了第 50 节强调的规则：
```text
唤醒前先授予锁，避免被新来的请求抢走。
```
对诊断而言，这意味着被唤醒 backend 还没真正返回用户代码时，shared lock table 已经显示它持有锁。
`ProcSleep()` 返回后只读取 `MyProc->waitStatus`。
如果是 OK，`LockAcquireExtended()` 最后调用 `GrantLockLocal()` 补上本地 owner 账本。
如果是 ERROR，进入 deadlock report。
### 5.7 释放与 cleanup：等待队列如何消失
正常释放路径是：
```text
LockRelease()
  -> UnGrantLock()
  -> CleanUpLock()
  -> ProcLockWakeup()
```
事务结束批量释放会走：
```text
ProcReleaseLocks()
  -> LockErrorCleanup()
  -> LockReleaseAll(DEFAULT_LOCKMETHOD, allLocks)
  -> LockReleaseAll(USER_LOCKMETHOD, false)
```
ERROR / cancel / timeout 等路径的关键是：
```text
LockErrorCleanup()
  -> AbortStrongLockAcquire()
  -> GetAwaitedLock()
  -> 如果 MyProc 仍在 wait queue，RemoveFromWaitQueue()
  -> 如果已经被 grant，GrantAwaitedLock()
  -> ResetAwaitedLock()
```
`RemoveFromWaitQueue()` 会撤销等待请求：
```text
从 LOCK.waitProcs 删除
LOCK.nRequested--
LOCK.requested[lockmode]--
必要时清 LOCK.waitMask bit
清 proc->waitLock / waitProcLock
proc->waitStatus = PROC_WAIT_STATUS_ERROR
CleanUpLock(..., wakeup=true)
```
这解释了为什么 cancel 一个队列前方 waiter 可能唤醒后面的 waiter。
取消者虽然没有释放已持有锁，但它不再 soft-block 后面的请求。
## 6. 观测与诊断入口：从 SQL 回源码
### 6.1 `pg_locks`：`pg_lock_status()` 如何生成行
`pg_locks` 背后的 C 函数是 `pg_lock_status()`。
它在第一次 SRF 调用时收集快照：
```text
mystatus->lockData = GetLockStatusData()
mystatus->predLockData = GetPredicateLockStatusData()
```
本节只关注 regular heavyweight lock。
`GetLockStatusData()` 分两段。
第一段扫描 fast-path slots，逐个 backend 拿 `proc->fpInfoLock`，把 relation / VXID fast-path lock 转成：
```text
fastpath = true
waitLockMode = NoLock
waitStart = 0
```
第二段拿所有 lock hash partition 的 `LW_SHARED`，扫描 `LockMethodProcLockHash`，为每个 `PROCLOCK` 拷贝：
```text
locktag / holdMask / waitLockMode / vxid / pid / leaderPid / waitStart
```
如果 `proc->waitLock == proclock->tag.myLock`，就设置 `waitLockMode = proc->waitLockMode`。
否则是 `NoLock`。
这就是 `granted=false` 行的来源。
`pg_lock_status()` 再按 `LOCKTAG` 类型填充 SQL 列：relation 映射到 `database` / `relation`，tuple 映射到 `page` / `tuple`，transaction 映射到 `transactionid`，object/advisory 映射到 `classid` / `objid` / `objsubid`。
最后用：
```text
GetLockmodeName(lockmethodid, mode)
```
填充 `mode`。
所以 `pg_locks.mode` 是 lock mode 名，不是 wait event 名。
### 6.2 `pg_blocking_pids()`：hard blocker 与 soft blocker
`pg_blocking_pids(blocked_pid)` 背后是：
```text
GetBlockerStatusData(blocked_pid)
  -> pg_blocking_pids() 在 C 层用 conflictTab 计算 blocker
```
`GetBlockerStatusData()` 首先拿 `ProcArrayLock`。
这样 `BackendPidGetProcWithLock(blocked_pid)` 得到的 `PGPROC` 不会立刻消失。
随后拿所有 lock partition LWLock。
如果目标 backend 是 lock group 成员，它会检查同一 lock group 的所有成员。
这就是 parallel query 下 `pg_blocking_pids()` 返回 group leader PID 的原因。
对每个真正 blocked 的 proc，`GetSingleProcBlockerStatusData()` 做两件事。
第一，复制 `blocked_proc->waitLock` 上所有 `PROCLOCK`。
第二，从 `LOCK.waitProcs` 队列开头收集 PID，直到遇到 `blocked_proc` 为止。
上层 `pg_blocking_pids()` 再计算：
```text
conflictMask = conflictTab[blocked_instance->waitLockMode]
```
对同一 lock 上其它 instance：
```text
if conflictMask & instance->holdMask:
    hard block
else if instance->waitLockMode conflicts
        and instance.pid is ahead in wait queue:
    soft block
else:
    not blocker
```
这解释三个常见现象。
第一，一个 blocker 不一定正在运行。
它可能 idle in transaction，但仍持有 `holdMask`。
第二，一个 blocker 不一定已经持有目标 mode。
它可能只是排在前方的 waiter。
第三，同一个 PID 可能重复出现。
源码注释说明 parallel query 和多个 waiter 场景下不会去重。
### 6.3 `pg_stat_activity`：wait event 不是 blocker graph
`pg_stat_activity` 的 wait event 填充在 `pgstatfuncs.c` 的 activity 输出路径。
本地源码读取：
```text
raw_wait_event = UINT32_ACCESS_ONCE(proc->wait_event_info)
wait_event_type = pgstat_get_wait_event_type(raw_wait_event)
wait_event = pgstat_get_wait_event(raw_wait_event)
```
`proc` 是当前 backend 对应的 `PGPROC`。
这条路径不读 `LOCK`。
它不知道 `PROCLOCK.holdMask`。
它也不知道 wait queue 前方是谁。
所以：
```text
pg_stat_activity.wait_event_type='Lock'
```
只应该解释为：
```text
这个 backend 当前正在以 Lock 类型 wait event 睡眠或刚处在相关等待窗口。
```
要知道对象和 blocker，需要回到 `pg_locks` 和 `pg_blocking_pids()`。
## 7. blocking chain walkthrough
### 7.1 构造一条三段 chain
假设有三条连接。
会话 A：
```sql
begin;
lock table t in access share mode;
-- 或者执行长查询，持有 AccessShareLock
```
会话 B：
```sql
begin;
lock table t in access exclusive mode;
-- 等 A
```
会话 C：
```sql
begin;
select * from t;
-- 可能等 B，而不是只等 A
```
源码上的状态可以写成：
```text
A:
  PROCLOCK.holdMask includes AccessShareLock
  PGPROC.waitLock = NULL

B:
  PROCLOCK.holdMask maybe 0
  PGPROC.waitLock = LOCK(t)
  PGPROC.waitLockMode = AccessExclusiveLock
  B 在 LOCK(t).waitProcs 前方

C:
  请求 AccessShareLock
  如果 B 的 AccessExclusiveLock 在 wait queue 前方形成冲突
  C 也进入 LOCK(t).waitProcs
```
诊断查询可能看到：
```text
pg_blocking_pids(B) = {A}
pg_blocking_pids(C) = {B}
```
这里 C 被 B soft-block。
B 还没持有 `AccessExclusiveLock`，但它排在 C 前方。
如果 C 可以绕过 B，B 可能长期被后续弱锁饿死。
所以 wait queue 本身是正确性和公平性的一部分。
### 7.2 从 `pg_locks` 定位同一 lock object
先看等待行，再按同一个 locktag 找 holder：
```sql
select pid, locktype, database, relation, transactionid,
       mode, granted, fastpath, waitstart
from pg_locks
where not granted;

select pid, locktype, database, relation, mode, granted, fastpath
from pg_locks
where locktype = 'relation'
  and relation = '<oid>'::oid
order by granted desc, pid;
```
这一步对应源码：
```text
pg_lock_status()
  -> LockInstanceData.locktag
  -> LockInstanceData.holdMask / waitLockMode
```
不要只按 `relation` 拼。
不同 `locktype` 的对象列不同：`transactionid` 看 `transactionid`，advisory lock 看 `database`、`classid`、`objid`、`objsubid`，relation lock 还要注意 database OID 和临时对象边界。
### 7.3 用 `pg_blocking_pids()` 展开 chain
实用查询可以从 `pg_stat_activity` 中找 `cardinality(pg_blocking_pids(pid)) > 0` 的 PID，再对 blocker 递归调用 `pg_blocking_pids()`。
这个递归只是诊断辅助，不是数据库内部算法。
每次函数调用都会重新取快照，所以高并发下 chain 可能在查询过程中改变。
它适合回答：
```text
当前近似 blocking graph 是什么？
```
不适合证明：
```text
过去几秒内精确发生了什么顺序？
```
如果要看时间顺序，需要结合日志、`waitstart`、应用请求时间、采样或 tracing。
### 7.4 从 wait event 回到 locktag 类型
当 `pg_stat_activity` 显示：
```text
wait_event_type = Lock
wait_event = tuple
```
源码解释是：
```text
ProcSleep()
  -> WaitLatch(..., PG_WAIT_LOCK | LOCKTAG_TUPLE)
```
下一步查同一 PID 的未授予锁：
```sql
select *
from pg_locks
where pid = <pid>
  and not granted;
```
`locktype='tuple'` 时看 `database`、`relation`、`page`、`tuple`。
`locktype='transactionid'` 时看 `transactionid`。
行级 UPDATE 冲突经常显示为等待 `transactionid` lock，因为最终需要等待修改者事务结束，而不是等待某个 tuple latch。
### 7.5 与 deadlock detector 的关系
`pg_blocking_pids()` 计算 blocker，不负责解决 deadlock。
deadlock detector 在等待路径中由 `deadlock_timeout` 触发。
`ProcSleep()` 发现 `got_deadlock_timeout` 后调用：
```text
CheckDeadLock()
  -> DeadLockCheck(MyProc)
```
`DeadLockCheck()` 在 `deadlock.c` 中递归搜索等待图，也基于：
```text
PGPROC.waitLock
PGPROC.waitLockMode
LOCK.procLocks
LOCK.waitProcs
LockMethodData.conflictTab
```
但二者目标不同：`pg_blocking_pids()` 返回诊断 PID array；`DeadLockCheck()` 要么确认无死锁，要么通过 wait queue reorder 解决 soft deadlock，要么报告 hard deadlock。
长 blocking chain 不等于 deadlock；deadlock 是存在环，且无法通过合法重排解决。
## 8. 生命周期 / ownership / cleanup
### 8.1 谁创建
`LOCALLOCK` 由当前 backend 在 `LockMethodLocalHash` 中创建。
`LOCK` 和 `PROCLOCK` 由 `SetupLockInTable()` 在 shared hash table 中创建。
`PGPROC` 是 backend 启动时分配的 shared identity。
`waitLink` 节点嵌在 `PGPROC` 中，不额外分配。
`LockInstanceData` 和 `BlockedProcsData` 是诊断函数调用时 `palloc()` 出来的临时快照。
### 8.2 谁持有
已授予锁的 ownership 分两层。
shared memory 层由 `PROCLOCK.holdMask` 表达某个 backend 持有哪些 mode。
backend-local 层由 `LOCALLOCK.nLocks` 和 `LOCALLOCK.lockOwners[]` 表达本 backend 内部计数和 ResourceOwner 归属。
session-level lock 的 owner 是 NULL。
transaction-level lock 的 owner 是 `CurrentResourceOwner`。
诊断快照不持有 lock manager 对象。
它只复制必要字段。
所以 `pg_locks` 输出行没有延长任何锁的生命周期。
### 8.3 谁释放
正常释放：
```text
LockRelease()
  -> 本地 owner count 下降
  -> 必要时 UnGrantLock()
  -> CleanUpLock()
  -> ProcLockWakeup()
```
事务结束：
```text
ProcReleaseLocks(isCommit)
  -> LockReleaseAll(DEFAULT_LOCKMETHOD, allLocks = !isCommit)
  -> LockReleaseAll(USER_LOCKMETHOD, allLocks = false)
```
ResourceOwner 也会参与零散释放。
子事务 abort 时，会释放子事务 owner 拥有的 locks。
prepared transaction 会把持有锁转移到 dummy `PGPROC`，使 `pg_locks.pid` 可能为 NULL。
这就是 `pg_locks` 中 prepared transaction 锁仍可见但没有普通 backend PID 的原因。
### 8.4 ERROR / timeout / cancel 兜底
等待期间的 cleanup 不能依赖正常返回。
核心兜底是：
```text
LockErrorCleanup()
```
它处理几类窗口：
```text
还在 wait queue:
  RemoveFromWaitQueue()

已经被别人 grant，但当前 backend 尚未返回:
  GrantAwaitedLock()

正在做 strong lock acquire fast-path transfer 边界:
  AbortStrongLockAcquire()
```
`ProcSleep()` 中 lock timeout 和 deadlock timeout 是 timeout 子系统触发的。
`lock_timeout` 会让等待抛 ERROR。
ERROR 展开时 `LockErrorCleanup()` 负责撤销等待队列状态。
deadlock 由 `CheckDeadLock()` 把状态改成 ERROR，再由 `DeadLockReport()` 抛出。
用户 cancel 和 backend die 也依赖相同清理边界。
### 8.5 `waitStart` 的生命周期
`waitStart` 在 `ProcSleep()` 中写。
成功唤醒或移出队列后会清零。
`GetLockStatusData()` 通过 atomic read 拷贝它。
因此：
```text
waitStart 非空:
  说明快照读取时这个 PGPROC 的 waitStart 已经写入且尚未清零。

waitStart 为空但 granted=false:
  可能是刚入队、尚未写 waitStart 的短窗口。

waitStart 消失:
  可能已经被 grant、cancel、deadlock cleanup 或 backend exit 清理。
```
`waitStart` 是诊断时间戳，不是锁请求的全局序列号。
## 9. 正确性机制层次
### 9.1 heavyweight lock 与 LWLock 的边界
heavyweight lock 表达 SQL / object 级互斥语义。
LWLock 保护 lock manager 自己的 shared hash table 和 wait queue。
`LockAcquireExtended()` 修改 `LOCK` / `PROCLOCK` 时持 lock partition LWLock。
`GetLockStatusData()` 拷贝 main lock table 时也按 partition 顺序拿 LWLock。
但 `pg_locks` 里的 lock mode 不是 LWLock mode。
不要把 `AccessExclusiveLock` 理解成某个 shared memory mutex。
它是 lock manager 的逻辑 mode。
### 9.2 conflict table 是 blocking 判断的语义来源
`LockMethodData.conflictTab` 决定 mode 之间是否冲突。
`pg_blocking_pids()` 不靠 lock mode 名字符串判断。
它取：
```text
conflictTab[blocked_instance->waitLockMode]
```
再与其它 backend 的 `holdMask` 或 `waitLockMode` 比较。
这也是 deadlock detector 使用的语义基础。
所以诊断时不要用自写的简化冲突表替代源码。
尤其 advisory lock、object lock、relation lock 虽然共享标准 mode 名，但 locktag namespace 不同。
### 9.3 wait queue 顺序是 correctness 的一部分
等待队列不是纯性能结构。
它影响 fairness、soft blocker 和 deadlock detector 的搜索空间。
`JoinWaitQueue()` 可能把请求插到某个 waiter 前面。
`DeadLockCheck()` 可能通过队列重排解决 soft deadlock。
`ProcLockWakeup()` 从队头扫描，保证不会跳过仍应优先考虑的 waiter。
因此 `pg_blocking_pids()` 必须知道目标 blocked proc 前方有哪些 waiter。
这就是 `GetBlockerStatusData()` 单独保存 `waiter_pids` 前缀的原因。
### 9.4 lock group 边界
parallel query 下多个 backend 可以属于同一个 lock group。
同组成员不互相阻塞。
`pg_blocking_pids()` 输出 blocker 的 group leader PID。
源码判断里会跳过：
```text
instance->leaderPid == blocked_instance->leaderPid
```
这让客户端更容易把 blocker 映射回发起查询的 leader。
但也带来诊断边界：
你看到的 PID 可能不是实际持有 `PROCLOCK` 的 worker PID。
需要结合 `leader_pid`、parallel worker 信息和 `pg_stat_activity` 查询。
### 9.5 fast path 可见性边界
fast-path relation lock 可以在 `pg_locks` 里看到。
`GetLockStatusData()` 会扫描每个 backend 的 fast-path slots。
但 fast-path locks “不能参与冲突”这个前提来自第 51 节：
任何可能被强锁冲突检查需要的 fast-path weak relation locks，会在强锁请求前迁回 main lock table。
所以 `GetSingleProcBlockerStatusData()` 明确可以忽略 fast-path arrays：
```text
当前 blocked proc 等待的 contended lock 必须在 main table；
和它相关的 blocker 也应通过 LOCK.procLocks 体现。
```
这不是说 fast path 不可观测。
而是说 blocking chain 的等待边不从 fast-path slots 推导。
## 10. 错误路径 / 异常路径 / fallback
### 10.1 `dontWait`：不进入睡眠也可能短暂建表
某些调用使用 `dontWait=true`。
`LockAcquireExtended()` 即使在 dontWait 场景也可能调用 `JoinWaitQueue()`。
原因是 `JoinWaitQueue()` 可能发现请求可以插队并立即 grant。
如果确认需要等待且 `dontWait=true`，它返回 ERROR 状态。
调用者随后撤销 `SetupLockInTable()` 已经增加的 requested counts。
诊断上通常看不到长期 `granted=false` 行。
但源码必须正确撤销：
```text
LOCK.nRequested--
LOCK.requested[lockmode]--
必要时删除 holdMask=0 的 PROCLOCK
```
### 10.2 lock table OOM
如果 `SetupLockInTable()` 失败，`LockAcquireExtended()` 根据 `reportMemoryError` 决定抛 ERROR 或返回 `LOCKACQUIRE_NOT_AVAIL`。
典型报错建议增加：
```text
max_locks_per_transaction
```
这不是 waiting chain。
`pg_stat_activity.wait_event_type` 未必显示 Lock。
它是 lock manager shared hash 容量压力。
诊断时不要把所有 lock 问题都解释成阻塞。
### 10.3 deadlock timeout 与 lock timeout
`deadlock_timeout` 不是“超过就报错”。
它是“等待一段时间后运行昂贵 deadlock detection”。
`lock_timeout` 才是用户配置的等待时间上限。
`ProcSleep()` 会同时设置两个 timeout。
如果 deadlock detector 返回 hard deadlock，`DeadLockReport()` 报错。
如果 lock timeout 触发，ERROR 通过 `LockErrorCleanup()` 撤销 wait queue 状态。
如果只是长等待但无死锁，backend 继续睡眠。
所以：
```text
waitstart 很久
```
不等于：
```text
deadlock detector 没有运行
```
它可能运行过并认为没有 hard deadlock。
### 10.4 Hot Standby recovery conflict
`ProcSleep()` 在 Hot Standby 路径会调用 `ResolveRecoveryConflictWithLock()`。
这类等待和 primary 上普通 lock wait 的处理不同。
还可能记录 recovery conflict 日志。
本节不展开 recovery conflict 的全部规则。
诊断边界是：
如果系统处于 standby，lock wait 可能和 WAL replay 需要获得的 AccessExclusiveLock 冲突有关。
这时要结合 recovery conflict 日志、`pg_stat_database_conflicts` 和 replay 状态。
### 10.5 backend exit 和 prepared transaction
普通 backend exit 会释放其事务级锁和进程资源。
prepared transaction 的锁被 dummy `PGPROC` 持有。
`pg_locks.pid` 可能为 NULL，但锁仍然真实阻塞别人。
诊断时看到 blocker PID 为空，需要查：
```sql
select * from pg_prepared_xacts;
```
不能试图 `pg_cancel_backend(NULL)`。
正确动作是业务层决定 `COMMIT PREPARED` 或 `ROLLBACK PREPARED`。
### 10.6 autovacuum blocker
`DeadLockState` 中有 `DS_BLOCKED_BY_AUTOVACUUM`。
`ProcSleep()` 中还有允许中断 autovacuum 的逻辑。
这说明 autovacuum 既是普通 backend，又有运维策略上的特殊处理。
如果 DDL 被 autovacuum 阻塞，系统可能选择 cancel autovacuum 来减少用户可见等待。
诊断时要看 blocker 的 `backend_type`，不要只看 PID。
## 11. 成本、资源与跨模块传播
### 11.1 诊断函数本身的成本
`GetLockStatusData()` 会扫描：
```text
ProcGlobal->allProcCount 个 PGPROC fast-path slots
所有 lock hash partitions
整个 LockMethodProcLockHash
predicate lock snapshot
```
在 backend 数多、fast-path slots 多、lock table 大时，频繁全量查询 `pg_locks` 有成本。
它还会按顺序拿所有 lock partition 的 shared LWLock。
虽然持锁时间只用于拷贝，但高频采样仍会给 lock manager 增加干扰。
`pg_blocking_pids(pid)` 范围更窄，但仍会拿 `ProcArrayLock` 和所有 lock partition LWLock。
在监控系统中对所有 backend 高频调用它，会形成放大。
### 11.2 blocking chain 的规模变量
影响诊断复杂度的变量包括：
| 变量 | 扩张方式 |
| --- | --- |
| `MaxBackends` | `pg_locks` fast-path 扫描和预分配数组按 backend 数扩张。 |
| `NUM_LOCK_PARTITIONS` | 全量快照要按顺序拿所有 partition LWLock。 |
| `LockMethodProcLockHash` entry 数 | `pg_locks` 主表扫描成本随 PROCLOCK 数扩张。 |
| wait queue 长度 | `pg_blocking_pids()` soft blocker 前缀检查随队列前方 waiter 数扩张。 |
| parallel workers | lock group 让一个客户端查询对应多个 PGPROC。 |
| 分区表数量 | 单个 SQL 可持有大量 relation locks，放大 pg_locks 行数。 |
不要在大实例上把下面查询作为 1 秒级全量监控：
```sql
select *
from pg_locks l
join pg_stat_activity a using (pid);
```
更稳妥的策略是：
先用 `pg_stat_activity` 找 `wait_event_type='Lock'` 的少量 PID；
再对这些 PID 调用 `pg_blocking_pids()`；
最后只按相关 locktag 查 `pg_locks`。
### 11.3 与事务和 MVCC 的边界
很多行级冲突最终显示为 `transactionid` lock wait。
这是 lock manager 与 MVCC 的边界。
heap tuple 上看到另一个事务修改痕迹时，当前事务需要等待对方提交或回滚，才能判断可见性和冲突结果。
看到：
```text
locktype = transactionid
mode = ShareLock
granted = false
```
通常应继续查 blocker 事务正在执行什么，而不是只查某张表上的 relation lock。
对象 identity 不总是直接暴露为表名，需要结合 `pg_stat_activity.query`、应用日志和 holder。
### 11.4 与 DDL、invalidation 和后台进程的边界
DDL 常见 pattern 是：
```text
ALTER TABLE 等 AccessExclusiveLock
普通 SELECT 持 AccessShareLock
后续 SELECT 又被 ALTER TABLE soft-block
```
这不是 catalog cache 问题；catalog invalidation 处理元数据新鲜度，relation heavyweight lock 负责 DDL 与并发访问的互斥和等待顺序。
blocking chain 中还可能出现 `autovacuum worker`、parallel worker、logical replication worker、standby startup 相关等待或 prepared transaction dummy `PGPROC`。
因此诊断动作必须先分清 blocker 类型：取消客户端 backend、终止 autovacuum、处理 prepared xact、调整 standby 查询，风险完全不同。
## 12. 课堂实验
### 实验 1：复现 relation lock soft blocker
准备表后开三条连接：
```sql
create table lock_diag_t(id int primary key, v text);
insert into lock_diag_t values (1, 'a');
```
执行顺序：
```sql
-- A
begin; select * from lock_diag_t; -- 保持事务
-- B
begin; alter table lock_diag_t add column c int; -- 等 AccessExclusiveLock
-- C
select * from lock_diag_t; -- 观察是否被 B soft-block
```
诊断时只需要三类查询：
```sql
select pid, wait_event_type, wait_event, state, query from pg_stat_activity;
select pid, locktype, relation::regclass, mode, granted, waitstart from pg_locks;
select pid, pg_blocking_pids(pid) from pg_stat_activity;
```
回源码画出：
```text
B: LockAcquireExtended() -> JoinWaitQueue() -> WaitOnLock() -> ProcSleep()
C: conflictTab[AccessShareLock] 受 waitMask / 前方 AccessExclusiveLock 请求影响
```
### 实验 2：观察 `waitstart` 窗口和 wait event
让一个会话持有锁，另一个会话 `set lock_timeout = '30s'` 后等待，并快速循环采样：
```sql
select now(), pid, wait_event_type, wait_event
from pg_stat_activity
where wait_event_type = 'Lock';

select now(), pid, mode, granted, waitstart
from pg_locks
where not granted;
```
预期：
```text
大多数时候 granted=false 会伴随 waitstart；
极短窗口内 waitstart 可能为空；
wait_event 只显示 locktag 类型，不显示对象 OID。
```
源码位置是：
```text
JoinWaitQueue() 先写 MyProc.waitLock / waitLockMode；
ProcSleep() 后写 MyProc.waitStart，再 WaitLatch(..., PG_WAIT_LOCK | locktag_type)。
```
### 实验 3：源码断点跟 blocker 计算
在测试实例上复现 B 等 A，然后打断点：
```text
break pg_blocking_pids
break GetBlockerStatusData
break GetSingleProcBlockerStatusData
break ProcSleep
```
执行：
```sql
select pg_blocking_pids(<B pid>);
```
观察：
```text
blocked_proc->waitLock
blocked_proc->waitLockMode
theLock->procLocks
theLock->waitProcs
BlockedProcsData.waiter_pids
conflictMask
```
目标是画出：
```text
PGPROC(B)
  -> waitLock
  -> LOCK(t)
  -> procLocks: PROCLOCK(A), PROCLOCK(B)
  -> waitProcs: B, ...
  -> conflictTab[AccessExclusiveLock]
  -> blocker PID array
```
## 13. 常见误区
- 把 wait event 当完整因果：`wait_event_type='Lock'` 不告诉你 blocker、lock mode 或对象 identity，必须结合 `pg_locks` 和 `pg_blocking_pids()`。
- 只找 `granted=true` holder：后方 waiter 可能被前方 waiter soft-block，只看 holder 会漏掉排队顺序。
- 按 `waitstart` 排序当作 wait queue 顺序：真实队列是 `LOCK.waitProcs`，`JoinWaitQueue()` 和 deadlock detector 都可能影响位置。
- 认为 `fastpath=true` 是另一种锁语义：fast path 只是物理表示，冲突等待仍要回 main lock table。
- 把 `transactionid` lock 当成表锁：它等待事务结束，常由 tuple update/delete 冲突引出。
- 看到 blocker PID 就直接 terminate：blocker 可能是长事务、DDL、autovacuum、parallel leader、prepared transaction 或 standby recovery conflict，处理动作不同。
- 认为一次 SQL 递归 chain 是精确历史：`pg_blocking_pids()` 每次调用都是当前快照，只适合近似定位当前链路。
## 14. 讨论题
1. 为什么 `pg_stat_activity.wait_event='relation'` 不能直接告诉你被锁住的是哪张表？
2. 为什么 `pg_blocking_pids()` 要报告排在前方的 waiter，而不仅是已经 granted 的 holder？
3. `pg_locks.granted=false` 但 `waitstart` 为空，源码上可能处在哪个窗口？
4. 为什么 `GetBlockerStatusData()` 可以忽略 fast-path arrays？
5. parallel query lock group 下，为什么 blocker PID 使用 group leader？
6. cancel 一个等待中的 backend 为什么可能唤醒后面的 waiter？
7. 为什么 deadlock detector 和 `pg_blocking_pids()` 都依赖 conflict table，但目标完全不同？
8. 如果一个 blocker 是 prepared transaction，`pg_stat_activity` 为什么可能查不到对应行？
## 15. 本节小结
本节唯一主问题是：
```text
如何从 pg_locks、pg_stat_activity 和 wait event 看到的阻塞现象，回到 lock manager 源码里的等待点和 blocking chain？
```
核心链路是：
```text
LockAcquireExtended()
  -> SetupLockInTable()
  -> conflictTab / waitMask / LockCheckConflicts()
  -> JoinWaitQueue()
  -> PGPROC.waitLock / waitProcLock / waitLockMode
  -> WaitOnLock()
  -> ProcSleep()
  -> WaitLatch(..., PG_WAIT_LOCK | locktag_type)
```
观测链路是：
```text
pg_stat_activity:
  PGPROC.wait_event_info -> wait_event_type / wait_event

pg_locks:
  GetLockStatusData() -> LockInstanceData -> pg_lock_status()

pg_blocking_pids():
  GetBlockerStatusData() -> conflictTab + holdMask + wait queue prefix
```
核心状态边界是：
```text
PGPROC 是等待节点；
LOCK 是 lockable object；
PROCLOCK 是 backend 与 object 的持有/等待边；
LOCALLOCK 是 backend-local owner 账本；
wait_event_info 是活动状态投影；
LockInstanceData / BlockedProcsData 是诊断快照，不是 shared state 本身。
```
cleanup 边界是：
```text
正常 release 由 LockRelease() / LockReleaseAll() 推进；
等待中的 ERROR、cancel、timeout 由 LockErrorCleanup() 和 RemoveFromWaitQueue() 兜底；
grant 必须在唤醒 waiter 前完成；
deadlock detector 可以报 hard deadlock，也可以通过队列重排解决 soft deadlock。
```
观测边界是：
```text
pg_stat_activity 能看到当前 wait event，不能算 blocker；
pg_locks 能看到 held/waited lock modes，不能直接给 wait queue 顺序；
pg_blocking_pids() 能算当前 blocker，不能保存历史；
日志和采样能补时间线，但仍受 timing 和 workload 影响。
```
可迁移规律：
```text
并发诊断接口通常不是原始共享结构的完整镜像。
它们是为了低干扰而构造的瞬时投影。
正确诊断要先问“这个视图从哪个 runtime state 拷贝而来、拷贝时持了什么锁、丢掉了哪些顺序和历史”，再把多个投影合成近似因果链。
```
