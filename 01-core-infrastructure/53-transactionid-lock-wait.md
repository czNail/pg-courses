# PostgreSQL transactionid lock wait
## 课程定位
前置知识：已经理解 `LOCKTAG`、heavyweight lock、`PGPROC` wait state、deadlock detector、heap tuple header、MVCC visibility 和 `ResourceOwner` 的基本边界。
本节唯一主问题：
```text
tuple update 冲突为什么常表现为等待 transactionid lock，而不是等待 heap tuple 本身？
```
核心矛盾：
```text
冲突发生在一个具体 heap tuple 上；
但等待真正需要知道的是另一个事务最终 commit 还是 abort。
heap tuple 是页内物理记录，不能承载长时间等待队列和事务结局；
transactionid lock 是事务生命周期对象，可以被 lock manager 等待、唤醒、诊断和死锁检测。
```
一句话运行模型：
```text
heap_update() / heap_delete() / heap_lock_tuple() 先从 tuple header 读出 xmax 或 MultiXactId；
如果 xmax 指向仍在运行且冲突的事务，就先用 tuple lock 只建立本 tuple 的排队优先级；
真正睡眠时调用 XactLockTableWait() 或 MultiXactIdWait()；
后者最终等待一个或多个 LOCKTAG_TRANSACTION 的 ShareLock；
holder 事务结束时释放自己持有的 transactionid ExclusiveLock，等待者醒来后重新检查 tuple header。
```
学完后应能判断：
- 为什么 `pg_stat_activity.wait_event = 'transactionid'` 仍然可能是行级更新冲突。
- 为什么 `pg_locks.locktype = 'transactionid'` 不表示用户显式锁了一个事务对象。
- 为什么 heap tuple header 中的 `xmax` 是冲突线索，而不是等待队列。
- 为什么等待醒来后必须重新读 buffer 并检查 `xmax`、`infomask` 和 `t_ctid`。
- 为什么 MultiXact 场景会等待多个成员事务，而不是等待 MultiXactId 自身。
- 为什么 `SELECT ... FOR UPDATE NOWAIT` / `SKIP LOCKED` 会走条件等待路径。
- 为什么 `LOCKTAG_TUPLE` 存在，但它不是 tuple update 冲突的主要等待对象。
- 为什么 deadlock detector 能处理这种行级冲突。
本课基于本地源码：
```text
/home/nail/postgres
branch: master
commit: 0e1f1ed157e9
```
本节不讲 SQL 级行锁语法全集，只围绕一个 runtime 现象展开：两个事务更新同一行时，第二个事务常在 `pg_stat_activity` 中显示 `wait_event_type='Lock'`、`wait_event='transactionid'`。
源码里确实有 `LOCKTAG_TUPLE` 和 `LockTuple()`；本节要区分的是哪类状态承载长等待，哪类状态只是页内事实和排队辅助。
## 1. 本节在总主线中的位置
第 49 节解释了 `LOCKTAG` 如何把 relation、tuple、transactionid、advisory key 等对象压进统一 lock manager。
第 50 节解释了 `LockAcquireExtended()`、`LockRelease()`、`LOCALLOCK`、`LOCK`、`PROCLOCK` 和 `ResourceOwner` 的 acquire / release 生命周期。
第 51 节解释了 relation fast path 只是物理表示优化，等待和强协调必须回到 main lock table。
第 52 节解释了 wait queue、soft edge 和 deadlock detector。
本节插在 52 和 54 之间。
它把前面几节的通用 lock manager 带回 heap tuple update 现场。
在用户眼里，冲突对象是一行。
在 heap 页上，冲突线索是 tuple header 的 `xmax`、`infomask` 和 `t_ctid`。
在 lock manager 里，长等待对象经常是 `LOCKTAG_TRANSACTION`。
这三个视角都正确，但层次不同。
看到 `transactionid` wait 后转去查显式事务锁，或者看到 `LOCKTAG_TUPLE` 后要求所有 row-level wait 都显示为 tuple，都会误诊。
源码不是这样分层的：
`LOCKTAG_TUPLE` 的主要作用是“排队优先级”和避免多个 waiter 在等待前后抢写同一个 tuple header。
`LOCKTAG_TRANSACTION` 的作用是等待一个事务结束。
tuple header 的作用是记录谁修改或锁住过这个 tuple，以及这种状态是 lock-only、update、delete 还是 MultiXact。
所以本节连接三条边界：
```text
heapam.c:
  解释 tuple header 状态，决定是否需要等待、等待谁、醒来后如何重查。
lmgr.c / lock.c / proc.c:
  负责 transactionid lock 的等待、唤醒、timeout、deadlock 和 cleanup。
xact.c / multixact.c:
  负责事务 XID lock 生命周期，以及 MultiXact member 列表的事务集合语义。
```
## 2. 核心矛盾与一句话运行模型
先把用户现象压缩成一条最小时间线：
```text
Session A:
  BEGIN;
  UPDATE t SET v = v + 1 WHERE id = 1;
  -- 不提交

Session B:
  BEGIN;
  UPDATE t SET v = v + 1 WHERE id = 1;
  -- 阻塞
```
从 SQL 看，B 等的是同一行。
从 heap 看，A 已经把旧 tuple 的 `xmax` 改成自己的 XID，并可能把 `t_ctid` 指向新版本。
从 lock manager 看，A 在取得 XID 时已经通过 `XactLockTableInsert()` 持有该 XID 的 `ExclusiveLock`。
B 看到旧 tuple 的 `xmax = A.xid` 还在运行。
B 不能只靠读 tuple header 判断结果。
因为 A 可能 commit。
也可能 abort。
还可能是 subtransaction abort 后 top transaction 仍然运行。
也可能旧 `xmax` 是一个 MultiXactId，里面包含多个 locker 和一个 updater。
B 真正需要等待的不是“tuple 这块内存可用”。
B 需要等待的是：
```text
A.xid 的事务结局稳定。
```
因此睡眠对象自然是 transactionid。
tuple 本身不适合作为长等待对象：它在 buffer page 内，content lock 不能跨事务持有；tuple header 只存状态，不存 wait queue；等待期间 `xmax`、`infomask`、`t_ctid` 都可能变化；冲突结果还取决于 pg_xact、ProcArray、SubTrans 和 lock manager 推进的事务结局。
让 row update 等待 transactionid lock，也让行级冲突复用已有 wait queue、timeout、cleanup 和 deadlock detection。
所以本节一句话答案是：
```text
tuple header 告诉你“该等哪个事务”；
transactionid lock 承载“等到这个事务结束”的长等待；
tuple lock 只在必要时建立本 tuple 上多个 waiter 的排队优先级。
```
注意这里有三个名字很像但语义不同的东西。
`heap tuple header` 是页内记录。
`LOCKTAG_TUPLE` 是 heavyweight lock manager 中的 tuple lock tag。
`LOCKTAG_TRANSACTION` 是 heavyweight lock manager 中的 transactionid lock tag。
tuple update 冲突常见路径会同时涉及三者。
但最终让 backend 睡眠的 wait event 常常是 transactionid。
这不是 PostgreSQL 忽略了 tuple。
这是 PostgreSQL 把“物理冲突位置”和“等待完成条件”分开了。
本节的可迁移抽象是：
```text
等待对象应该绑定到状态完成条件，而不一定绑定到冲突被发现的位置。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/heapam.c` | `heap_update()`、`heap_delete()`、`heap_lock_tuple()` 如何从 `xmax` 判断冲突、调用 tuple lock 和 XID wait。 |
| 2 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesUpdate()` 如何返回 `TM_BeingModified`、`TM_Updated`、`TM_Deleted` 等状态。 |
| 3 | `src/backend/storage/lmgr/lmgr.c` | `LockTuple()`、`ConditionalLockTuple()`、`XactLockTableWait()`、`ConditionalXactLockTableWait()`。 |
| 4 | `src/backend/storage/lmgr/lock.c` | transactionid lock 仍走通用 `LockAcquireExtended()`、wait queue、release 和 timeout 路径。 |
| 5 | `src/backend/storage/lmgr/proc.c` | `ProcSleep()`、`ProcLockWakeup()`、`LockErrorCleanup()` 如何处理等待、唤醒和 ERROR cleanup。 |
| 6 | `src/backend/access/transam/xact.c` | XID 分配时 `XactLockTableInsert()`，事务结束时 `XactLockTableDelete()`。 |
| 7 | `src/backend/access/transam/multixact.c` | MultiXact 成员、offset/member storage、oldest member 边界。 |
| 8 | `src/include/storage/locktag.h` | `SET_LOCKTAG_TRANSACTION()` 和 `SET_LOCKTAG_TUPLE()` 的 key 布局。 |
| 9 | `src/include/storage/lockdefs.h` | `ShareLock`、`ExclusiveLock` 等 mode 的编号和冲突规则输入。 |
| 10 | `src/include/access/htup_details.h` | `t_xmax`、`HEAP_XMAX_IS_MULTI`、`HEAP_XMAX_LOCK_ONLY`、hint bits。 |
| 11 | `src/include/access/tableam.h` | `TM_Result` 和 `TM_FailureData` 的 table AM contract。 |
| 12 | `src/include/nodes/lockoptions.h` | `LockTupleMode` 和 `LockWaitPolicy` 的 SQL 等待策略边界。 |
推荐阅读顺序：
```text
heapam_visibility.c 的 HeapTupleSatisfiesUpdate()
  -> tableam.h 的 TM_Result / TM_FailureData
  -> heapam.c 的 heap_update() 等待分支
  -> heapam.c 的 heap_lock_tuple() NOWAIT / SKIP LOCKED 分支
  -> lmgr.c 的 XactLockTableWait()
  -> xact.c 的 XactLockTableInsert/Delete 生命周期
  -> heapam.c 的 MultiXactIdWait() / Do_MultiXactIdWait()
  -> lock.c / proc.c 的普通 heavyweight wait path
```
不要从 `pg_locks` 视图开始读。
视图展示的是等待对象投影，不是冲突被发现的位置。
也不要先从 `LOCKTAG_TUPLE` 开始读。
否则容易把 tuple lock 理解成长等待主语。
本节源码阅读要一直问：
```text
当前状态回答的是“谁改了 tuple”；
还是“谁的事务结局还不确定”；
还是“多个 waiter 在同一个 tuple 上谁排在前面”？
```
## 4. 关键数据结构与状态
### 4.1 heap tuple header：冲突线索在页内
heap tuple header 的相关字段和 bit 在 `htup_details.h`。
本节只需要最小集合：
```text
t_xmin:
  插入该 tuple version 的事务。
t_xmax:
  删除、更新或锁住该 tuple version 的事务 XID，或 MultiXactId。
t_ctid:
  当前 tuple 自身位置，或 update 后指向后继版本。
t_infomask:
  HEAP_XMAX_INVALID、HEAP_XMAX_COMMITTED、HEAP_XMAX_IS_MULTI、
  HEAP_XMAX_LOCK_ONLY、HEAP_XMAX_EXCL_LOCK 等状态 bit。
t_infomask2:
  HEAP_KEYS_UPDATED 等补充状态。
```
`t_xmax` 不是“删除事务”这么简单。
它可能表示 update、delete、lock-only、MultiXact 成员集合，或 abort 后留下的无效痕迹。
所以本节诊断的第一条底线是：
```text
raw xmax 只是指针；
xmax + infomask + tuple visibility + transaction status 才是语义。
```
等待路径通常从 `HeapTupleSatisfiesUpdate()` 返回 `TM_BeingModified` 开始。
这不是说 tuple 内存正在被别的 backend 改写，而是表示：
```text
对当前 update/delete/lock 操作来说，
tuple header 指向的另一个事务或 MultiXact 仍在运行，
其结果会影响本操作能否继续。
```
这就是 transactionid wait 的入口。
### 4.2 `TM_Result`：table AM 层的冲突协议
`tableam.h` 中的 `TM_Result` 是 table AM 与 executor 的协议。
本节关注这些值：
```text
TM_Ok             操作可以继续。
TM_Invisible      tuple 对当前 command 不可见。
TM_SelfModified   当前事务自己已经修改过该 tuple。
TM_Updated        其它已提交事务更新了该 tuple。
TM_Deleted        其它已提交事务删除了该 tuple。
TM_BeingModified  其它事务仍在修改或锁住该 tuple，需要等待或返回 would-block。
TM_WouldBlock     NOWAIT / SKIP LOCKED 等条件路径无法等待。
```
`TM_BeingModified` 是本节主入口，它把 heap visibility 判断转化成等待决策。
失败时，`TM_FailureData` 会把 `ctid`、`xmax` 和必要的 `cmax` 返回给上层。
这解释了为什么等待醒来后不能直接继续写：
```text
等待只是让事务结局稳定；
heap_update() 仍要重新判断 TM_Ok、TM_Updated 还是 TM_Deleted。
```
### 4.3 `LOCKTAG_TRANSACTION`：等待事务结束的对象身份
`lmgr.c` 中的 transactionid wait 使用通用 heavyweight lock。
核心模式可以简化为：
```c
SET_LOCKTAG_TRANSACTION(tag, xid);
LockAcquire(&tag, ShareLock, false, false);
LockRelease(&tag, ShareLock, false);
```
被等待事务在取得 XID 时持有：
```c
SET_LOCKTAG_TRANSACTION(tag, xid);
LockAcquire(&tag, ExclusiveLock, false, false);
```
这就是 `transactionid` wait 的直接原因。
等待者请求同一个 locktag 的 `ShareLock`。
运行中的事务持有该 locktag 的 `ExclusiveLock`。
`ShareLock` 与 `ExclusiveLock` 冲突。
等待者进入通用 lock wait queue。
主事务结束时通过 `ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)` -> `ProcReleaseLocks()` -> `LockReleaseAll(DEFAULT_LOCKMETHOD, false)` 隐式释放非 session locks；子事务 XID lock 则由 `XactLockTableDelete()` 显式释放。
释放 `ExclusiveLock` 后，等待者被授予 `ShareLock`，随即释放它。
注意等待者不是为了长期持有 transactionid lock。
它只是借这个 lock 完成“等到事务结束”的同步。
所以 `XactLockTableWait()` 的语义不是“锁住事务”。
它的语义是：
```text
等到指定 XID 不再 in progress。
```
### 4.4 `LOCKTAG_TUPLE`：排队辅助，不是主等待结局
`lmgr.c` 也提供 `LockTuple()`、`ConditionalLockTuple()` 和 `UnlockTuple()`。
`LockTuple()` 使用：
```text
database OID
relation OID
block number
offset number
LOCKTAG_TUPLE
```
源码注释明确提醒：
```text
tuple-level lock 以不直观方式使用；
不能为每个 tuple 在 shared memory 中长期保留一把锁；
使用前要看 heap_lock_tuple。
```
在 `heap_update()`、`heap_delete()` 和 `heap_lock_tuple()` 的等待分支中，常见顺序是：
```text
释放 buffer content lock；
必要时获取 tuple lock，建立本 tuple 的排队优先级；
调用 XactLockTableWait() 或 MultiXactIdWait() 睡眠；
重新获取 buffer content lock；
检查 xmax 是否变化；
必要时 goto restart。
```
tuple lock 的作用是让同一 tuple 上的多个等待者形成顺序，避免它们在事务结束后同时争抢 header。
它不告诉你前一个事务 commit 还是 abort，因此不能替代 transactionid wait。
### 4.5 `MultiXactId`：多个 locker 的集合，不是一个事务
当多个事务同时对一个 tuple 持有兼容行锁时，tuple header 不能只存一个 XID。
PostgreSQL 用 MultiXact 记录成员集合。
成员包含 `xid` 和 `status`，`status` 表达 `ForKeyShare`、`ForShare`、`ForNoKeyUpdate`、`ForUpdate`、`NoKeyUpdate`、`Update` 等模式。
`HEAP_XMAX_IS_MULTI` 表示 `t_xmax` 要按 MultiXactId 解释。
MultiXact 本身不是会 commit 或 abort 的事务。
它是成员事务集合的编号。
所以等待 MultiXact 的实际动作是等待成员事务。
`heapam.c` 的 `Do_MultiXactIdWait()` 会遍历成员。
对冲突成员调用 `ConditionalXactLockTableWait(memxid, ...)` 或：
```text
XactLockTableWait(memxid, rel, ctid, oper)
```
这说明 MultiXact wait 最终仍会落回 transactionid lock wait。
`pg_locks` 中可能看到多个 transactionid wait。
这不是多把行锁被混淆。
它反映的是一个 tuple `xmax` 指向多个事务成员。
### 4.6 `LockTupleMode` 与 `LockWaitPolicy`
`LockTupleMode` 来自 `lockoptions.h`。
常见模式是：
```text
LockTupleKeyShare
LockTupleShare
LockTupleNoKeyExclusive
LockTupleExclusive
```
`heap_update()` 会根据是否修改 key columns 选择 `LockTupleNoKeyExclusive` 或 `LockTupleExclusive`。
`heap_lock_tuple()` 则对应 `SELECT FOR KEY SHARE`、`FOR SHARE`、`FOR NO KEY UPDATE`、`FOR UPDATE`。
`LockWaitPolicy` 决定不能立即获得冲突状态时怎么办：
```text
LockWaitBlock  正常阻塞等待。
LockWaitSkip   条件路径，失败时返回 TM_WouldBlock，用于 SKIP LOCKED。
LockWaitError  条件路径，失败时报错，用于 NOWAIT。
```
这三种策略不会改变核心事实：
```text
真正需要等待事务结局时，仍然围绕 XID 或 MultiXact 成员 XID。
```
不同之处只是：阻塞路径进入 `XactLockTableWait()`，跳过路径进入条件等待并返回 would-block，报错路径条件等待失败后抛 `could not obtain lock on row`。
### 4.7 `PGPROC` wait state：诊断看到的是等待对象
当 `XactLockTableWait()` 走进 `LockAcquire()` 并发生冲突时，通用 lock manager 会设置 `PGPROC` 的等待状态。
等待对象是 `LOCKTAG_TRANSACTION`。
所以 SQL 观测层看到：
```text
wait_event_type = 'Lock'
wait_event      = 'transactionid'
```
`pg_locks` 中等待项的 `locktype` 也是：
```text
transactionid
```
这并不否认冲突由 heap tuple 触发。
观测层显示的是当前 backend 正在等待的 heavyweight lock tag，不是完整因果链；完整解释必须再回到 heap tuple header：
```text
哪个 tuple 的 xmax 指向这个 xid？
这个 xid 对应哪个 blocker？
blocker 在执行哪条 SQL？
醒来后当前 tuple 会变成可继续、已更新还是已删除？
```
## 5. 主流程源码 walkthrough
### 5.1 事务先取得自己的 XID lock
transactionid wait 的生命周期从被等待事务开始。
事务或子事务取得真实 XID 时，会调用 `XactLockTableInsert()`。
主线在 `xact.c` 和 `lmgr.c` 之间：
```text
GetNewTransactionId()
  -> XactLockTableInsert(xid)
     -> SET_LOCKTAG_TRANSACTION(tag, xid)
     -> LockAcquire(&tag, ExclusiveLock, false, false)
```
这个 `ExclusiveLock` 是事务仍在运行的同步信号，不是用户显式锁。
主事务结束时通过 `ResourceOwnerRelease(... RESOURCE_RELEASE_LOCKS ...)` -> `ProcReleaseLocks()` -> `LockReleaseAll(DEFAULT_LOCKMETHOD, false)` 释放；子事务 XID lock 才由 `XactLockTableDelete()` 显式释放。
释放后等待者被唤醒。
holder 是目标事务自身，waiter 是需要判断目标事务结果的其它 backend。
纯读事务可能没有 XID；但能更新 tuple 的事务必然取得 XID，因此 tuple update/delete 冲突总能从 tuple header 找到可等待的事务身份。
### 5.2 第一个 updater 写入 tuple header
Session A 更新一行时，`heap_update()` 会在旧 tuple 上写入状态。
简化主线：
```text
heap_update()
  -> 获取 buffer 并锁住 page content
  -> HeapTupleSatisfiesUpdate()
  -> 计算 new_xmax / new_infomask
  -> START_CRIT_SECTION()
  -> HeapTupleHeaderSetXmax(oldtup, new_xmax)
  -> 设置 HEAP_KEYS_UPDATED、LOCK_ONLY 等 infomask
  -> 设置 oldtup.t_ctid 指向新 tuple version
  -> MarkBufferDirty()
  -> XLogInsert()
  -> END_CRIT_SECTION()
```
关键是旧 tuple 现在包含：
```text
xmax = A.xid 或包含 A.xid 的 MultiXactId
t_ctid = 后继版本位置
infomask = update / lock / multi / key-update 等语义 bit
```
对后来者来说，tuple header 是发现冲突的入口，但不是等待队列，也不是事务结局。
A 未提交时，B 不能仅凭 header 判断旧版本最终是否被更新；A abort 时 B 可以继续，A commit 时 B 要么跟随后继版本，要么返回并发更新冲突。
### 5.3 第二个 updater 看到 `TM_BeingModified`
Session B 进入 `heap_update()`。
它读取同一旧 tuple。
`HeapTupleSatisfiesUpdate()` 发现 `xmax` 指向仍在运行的事务。
结果是：
```text
TM_BeingModified
```
`heap_update()` 的等待分支会在释放 buffer content lock 前复制页内状态：
```text
xwait = HeapTupleHeaderGetRawXmax(oldtup.t_data)
infomask = oldtup.t_data->t_infomask
```
接下来 B 判断是否真的需要等待：
```text
如果 xmax 是 MultiXact:
  判断其中成员是否与本次 lock mode 冲突。
如果 xmax 是自己的事务:
  可能是 self-modified 或锁升级边界。
如果只是 key-share locker 且本次 update 不改 key:
  可以继续，但要保留 locker 信息。
否则:
  需要等待 xwait 结束。
```
不冲突的锁模式可以共存；一旦需要知道另一个事务是否提交，等待对象就转向 XID。
### 5.4 等待前先获取 tuple lock
真正睡眠前，`heap_update()` 常见路径会先调用：
```text
heap_acquire_tuplock(relation, &oldtup.t_self, *lockmode,
                     LockWaitBlock, &have_tuple_lock)
```
这些函数最终使用 `LOCKTAG_TUPLE`，目的不是等待 A commit，而是建立 B 在这个 tuple 上的排队位置。
源码注释的核心语义是：
```text
LockTuple 会在本 backend 成为该 tuple 下一位时释放等待；
如果后续必须 restart，保留 tuple lock 可以让本 backend 继续站在队头。
```
tuple lock 把“同一 tuple 上多个等待者的顺序”交给 lock manager，但不能给出 A 的 commit/abort 结果，所以它是排队辅助，不是事务结局同步。
### 5.5 正式睡眠：`XactLockTableWait()`
当 B 确认需要等待普通 XID 时，调用：
```text
XactLockTableWait(xwait, relation, &oldtup.t_self, XLTW_Update)
```
`XactLockTableWait()` 的主循环是：
```text
for (;;)
{
    SET_LOCKTAG_TRANSACTION(tag, xid);
    LockAcquire(&tag, ShareLock, false, false);
    LockRelease(&tag, ShareLock, false);
    if (!TransactionIdIsInProgress(xid))
        break;
    xid = SubTransGetTopmostTransaction(xid);
}
```
请求 `ShareLock` 只是等待手段，成功后立即释放。
函数在返回前还检查 `TransactionIdIsInProgress(xid)`，因为子事务 XID lock 可能在子事务结束时释放；如果 top transaction 仍运行，就转向 `SubTransGetTopmostTransaction()`。
少见情况下，调用者可能先在 ProcArray 看到事务、但目标事务尚未完成 lock table 注册，循环会短暂 sleep 后重试。
因此 `XactLockTableWait()` 的语义不是“锁获取成功就结束”，而是：
```text
确认目标 XID 对 visibility 来说不再 in progress。
```
### 5.6 MultiXact 等待最终也是 XID 等待
如果 tuple header 的 `xmax` 是 MultiXactId，路径是：
```text
MultiXactIdWait()
  -> Do_MultiXactIdWait()
  -> GetMultiXactIdMembers()
  -> 对冲突成员调用 XactLockTableWait()
```
条件路径则是：
```text
ConditionalMultiXactIdWait()
  -> ConditionalXactLockTableWait()
```
MultiXact wait 结合成员 `status` 判断冲突。
key-share locker 不一定阻塞 no-key update，多个 shared locker 可以共存；但成员模式与当前请求冲突时，等待对象仍是成员 XID。
所以 `pg_locks` 不一定显示 MultiXact 等待，一个行锁等待也可能需要等待多个事务结束。
### 5.7 醒来后必须重新锁 buffer 并 recheck
`XactLockTableWait()` 返回后，B 重新获取 buffer exclusive content lock。
然后检查：
```text
xmax_infomask_changed(oldtup.t_data->t_infomask, infomask)
HeapTupleHeaderGetRawXmax(oldtup.t_data) 是否仍等于 xwait
```
如果状态变化，`heap_update()` 会跳回 restart label：
```text
B 睡眠期间没有持有 buffer content lock。
tuple header 可能已经被别的 backend 改写。
```
如果 `xmax` 仍然一致，B 调用 `UpdateXmaxHintBits()`，再根据结果决定：
```text
如果前一个 xmax abort 或 lock-only:
  can_continue = true，B 可以继续更新。
如果前一个事务 commit 且 t_ctid 指向后继:
  result = TM_Updated。
如果前一个事务 commit 且 tuple 被删除:
  result = TM_Deleted。
```
transactionid wait 只解决“结局稳定”；能不能写 tuple，必须回到 heap header 和 visibility 重新判断。
### 5.8 `heap_lock_tuple()` 的 NOWAIT / SKIP LOCKED 分支
`SELECT ... FOR UPDATE` 等路径进入 `heap_lock_tuple()`。
它也可能从 tuple header 读到 `TM_BeingModified`。
不同的是它显式支持 `LockWaitPolicy`。
`LockWaitBlock` 调用 `XactLockTableWait(xwait, relation, &tuple->t_self, XLTW_Lock)`。
`LockWaitSkip` 调用条件等待，失败时返回 `TM_WouldBlock`。
`LockWaitError` 调用条件等待，失败时报 `ERRCODE_LOCK_NOT_AVAILABLE`。
对 MultiXact 也是同样结构，只是先走 `ConditionalMultiXactIdWait()`。
`NOWAIT` 和 `SKIP LOCKED` 不是不看 tuple header，而是不进入阻塞等待；条件 XID wait 不能立即完成时，返回错误或跳过。
### 5.9 deadlock detector 如何看见行级冲突
行级 update 冲突最终进入 heavyweight lock manager。
等待者在 `LOCKTAG_TRANSACTION` 上请求 `ShareLock`。
被等待事务持有同一 `LOCKTAG_TRANSACTION` 的 `ExclusiveLock`。
所以通用 deadlock detector 可以把它建成等待边。
典型死锁：
```text
Session A 更新 row 1；
Session B 更新 row 2；
Session A 再更新 row 2，等待 B.xid；
Session B 再更新 row 1，等待 A.xid。
```
检测器不需要理解 heap page、tuple header 或 update chain。
它只看到：
```text
A waits for LOCKTAG_TRANSACTION(B.xid)
B waits for LOCKTAG_TRANSACTION(A.xid)
```
这正是把长等待绑定到 transactionid 的另一个收益。
行级冲突进入统一等待图。
deadlock timeout 到期后，已有的 `ProcSleep()` / `DeadLockCheck()` 路径可以处理它。
## 6. 生命周期 / ownership / cleanup
### 6.1 transactionid lock 谁创建
事务取得 XID 时创建自己的 transactionid lock hold。
路径在 `xact.c` 调用 `XactLockTableInsert()`。
持有模式是 `ExclusiveLock`。
owner 是事务自身在当前 backend 中的 lock ownership。
这把锁不是用户语句显式创建。
它是事务系统内部身份的一部分。
任何可能把 XID 写入 tuple header 的事务都需要这个等待锚点。
### 6.2 transactionid lock 谁持有
运行中的事务持有自己 XID 上的 `ExclusiveLock`。
子事务也会取得自己的 XID lock。
但子事务结束时会释放自己的 XID lock。
这就是 `XactLockTableWait()` 要处理 subtransaction 的原因。
如果等待的是子事务 XID，且子事务 lock 已经消失，但 top transaction 仍在运行，函数必须继续等待 topmost transaction。
否则 visibility 可能过早判断。
### 6.3 transactionid lock 谁释放
commit 或 abort 结束路径会释放 transactionid lock。
释放发生在事务状态已经足以让等待者最终判断之后。
等待者醒来再读 pg_xact / hint bits / tuple header。
它不会从 lock release 本身直接得知 commit 或 abort。
lock release 只表示：
```text
这个 XID 不再运行；
可以去查询真实事务状态。
```
这也是为什么 `XactLockTableWait()` 返回后，heap 路径还要调用 `UpdateXmaxHintBits()` 或 `TransactionIdDidAbort()`。
### 6.4 tuple lock 谁创建、谁释放
等待者在需要排队时创建 tuple lock hold。
它通常在等待前获取。
如果等待后需要 restart，可能继续持有以保持队头位置。
当本次 heap 操作完成、失败或返回并发冲突时，调用 `UnlockTupleTuplock()` 释放。
ERROR 路径由 lock manager 的 cleanup 机制兜底。
但 heap 正常路径也会显式释放 `have_tuple_lock`。
tuple lock 的生命周期比事务短。
它服务于一次 tuple 操作。
不要把它当成 SQL 行锁的持久事实。
SQL 行锁的持久事实主要落在 tuple header 的 `xmax` / MultiXact 中。
### 6.5 tuple header 状态谁持有
tuple header 在 heap page 中。
它的并发访问由 buffer content lock 保护。
修改 tuple header 的关键区间通常还会进入 critical section 并写 WAL。
但是 buffer content lock 只保护内存页的一次读写。
它不会跨事务持有。
tuple header 的长期语义由 XID 状态、MultiXact 成员和 hint bits 解释。
这说明：
```text
buffer lock 管物理并发；
transactionid lock 管等待事务结束；
MVCC visibility 管哪个版本可见；
tuple header 管长期版本状态。
```
### 6.6 ERROR / abort 谁兜底
等待期间如果发生 cancel、timeout、deadlock ERROR 或 backend die，通用 lock manager 负责把 `PGPROC` 从 wait queue 中移除。
`proc.c` 的 `LockErrorCleanup()` 依赖当前等待的 `LOCALLOCK` 和 `PGPROC` wait fields 找回 shared lock partition。
然后撤销等待请求、清理 `waitLock`、更新 wait queue，并唤醒可能被队列顺序阻塞的后继者。
heap 层的 tuple lock hold 也要通过正常或异常 cleanup 释放。
事务 abort 会释放本事务持有的 transactionid lock、relation locks、tuple locks 和其它 `ResourceOwner` 资源。
旧 tuple header 中的 `xmax` 不会在 abort 时被同步清除。
后续访问通过 pg_xact 状态和 hint bits 判断 `xmax` invalid。
这是 MVCC 的常见设计：
```text
事务结束释放等待锚点；
页内状态可以延迟清理或标 hint；
可见性判断必须容忍旧痕迹。
```
## 7. 正确性机制层次
### 7.1 MVCC 与 lock manager
MVCC 判断回答：
```text
当前 snapshot 下哪个 tuple version 可见？
```
它不负责睡眠；`HeapTupleSatisfiesUpdate()` 可以返回 `TM_BeingModified`，真正等待交给 lock manager。
反过来，transactionid lock wait 返回后，lock manager 也不知道目标事务更新了哪个 tuple；`TM_Updated`、`TM_Deleted`、`TM_Ok` 仍由 heap 层重新判断。
### 7.2 buffer、tuple lock 与 recheck
heap 读写 tuple header 时需要 buffer content lock，但一旦要等待另一个事务，必须释放 buffer lock。
释放后就必须接受 tuple header 可能变化，因此醒来后 recheck 是正确性必需。
`LOCKTAG_TUPLE` 只让多个 waiter 在同一 tuple 上形成顺序；真正的行锁持久状态仍在 tuple header。
### 7.3 MultiXact 与 deadlock
MultiXact 的正确性目标是：
```text
允许兼容的多个 row lock 共存；
遇到冲突操作时只等待相关成员；
更新 tuple header 时保留仍然有效的 surviving lockers。
```
因此 `heap_update()` 在等待 MultiXact 后不一定认为 MultiXact 全部结束，可能还要保留本事务或其它兼容 locker。
一旦 row conflict 转化为 transactionid lock wait，deadlock detector 不需要 heapam 知识，只依赖：
```text
LOCK / PROCLOCK / PGPROC.waitLock / wait queue / conflict table
```
### 7.4 WAL / crash safety 与等待分离
更新 tuple header 和插入新 tuple 需要 WAL；transactionid lock wait 只保证并发等待，不保证 crash safety。
crash 后恢复依赖 WAL、pg_xact 和 heap redo。
不要把“等待到事务结束”理解为“数据页一定已经刷盘”；commit 可见性、WAL flush 和 heap page flush 是不同层次。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 NOWAIT / SKIP LOCKED
`LockWaitError` 会尝试条件 tuple lock 或条件 transactionid wait。
如果 `ConditionalXactLockTableWait()` 返回 false，就抛：
```text
could not obtain lock on row in relation ...
```
`SKIP LOCKED` 对应 `LockWaitSkip`。
条件等待失败时返回 `TM_WouldBlock`。
上层 executor 跳过该 tuple，继续扫描。
两者都可能短暂检查 lock table，但不加入长时间等待；NOWAIT 错误不是 deadlock，SKIP LOCKED 跳过的 tuple 也不一定最终被更新。
### 8.2 lock timeout / statement timeout
普通阻塞路径会进入 `ProcSleep()`。
`lock_timeout`、`statement_timeout` 或 cancel 可中断等待。
中断后 lock manager 必须清理 wait queue。
heap 层收到 ERROR 后事务通常 abort。
如果事务 abort，它持有的 transactionid lock 会释放。
其它等待者会继续推进。
这条路径说明：
```text
等待失败不等于 blocker 结束；
只是当前 waiter 放弃了本次请求。
```
### 8.3 deadlock
两个事务互相更新对方持有的 row 时，等待图通常表现为 transactionid lock 环。
deadlock detector 超过 `deadlock_timeout` 后运行。
如果检测到不可通过 wait queue reorder 消除的环，一个事务会收到 deadlock ERROR。
错误上下文可能来自 `XactLockTableWaitErrorCb()`：
```text
while updating tuple (block, offset) in relation "..."
```
这正好说明诊断要同时看两个层次。
wait event 是 transactionid。
error context 指回 heap tuple。
### 8.4 subtransaction
如果 tuple header 记录的是子事务 XID，等待者不能只等子事务 lock 消失就结束。
`XactLockTableWait()` 在发现 XID 仍被认为 in progress 时，会用 `SubTransGetTopmostTransaction()` 转向 topmost XID。
这是 transactionid wait 比简单 lock acquire 更强的地方：它服务 tuple visibility。
### 8.5 MultiXact member disappear
MultiXact 等待期间，部分成员可能结束，部分成员仍活着。
`Do_MultiXactIdWait()` 会统计 remaining 成员。
heap 更新路径需要保留 surviving lockers。
这就是 `locker_remains`、`checked_lockers` 等状态存在的原因。
### 8.6 restart loop
等待期间 tuple header 可能变化。
醒来后如果 `xmax` 或 `infomask` 与睡前不同，heap 路径会 `goto` restart label。
这是正常 fallback；高并发热点行会放大重试成本，但没有 recheck 会破坏 correctness。
## 9. 成本、资源与跨模块传播
### 9.1 hot row 把并发压成事务等待队列
同一行被大量事务更新时，吞吐不再随 backend 数线性扩展。
后续 updater 会等待前一个 updater 的 transactionid。
等待链的长度随竞争 backend 数增加。
每个事务持锁时间越长，后继等待时间越长。
成本主要来自：
```text
事务持有时间
lock table wait queue
context switch / latch wakeup
重复 recheck tuple header
可能的 deadlock detection
```
这类问题通常不是 CPU 算不动。
它是 serialization point。
### 9.2 transactionid lock table 成本
每个等待要进入 heavyweight lock manager。
会涉及：
```text
LOCKTAG hash
lock partition LWLock
LOCK / PROCLOCK
PGPROC wait state
timeout framework
deadlock detector slow path
```
相比直接读 tuple header，这很重。
但它只发生在冲突路径。
无冲突 update 不需要等待 transactionid lock。
这是 PostgreSQL 接受的取舍：
```text
常见无冲突路径保持页内状态和轻量检查；
真正长等待才进入通用 heavyweight lock manager。
```
### 9.3 MultiXact 成本
MultiXact 让兼容行锁可以共存。
代价是：
```text
需要创建或扩展 MultiXactId；
需要访问 multixact offset/member storage；
等待时要遍历成员；
VACUUM 需要考虑 multixact age；
tuple freeze 需要处理 xmax MultiXact。
```
热点 `SELECT FOR SHARE` / `FOR KEY SHARE` 与 update 混合时，MultiXact 可能成为重要成本来源。
诊断时要同时看 row lock wait 和 multixact age。
### 9.4 长事务传播
blocker 的长事务不只阻塞一个 waiter。
它还可能：
```text
延迟 vacuum cleanup；
拉低 xmin horizon；
增加 bloat；
阻塞更多 transactionid wait；
延迟 hot row 队列；
放大 deadlock 检测概率。
```
所以处理 transactionid wait 不能只杀当前 waiter。
通常要定位 blocker 的事务年龄、SQL、应用状态和持有行。
### 9.5 tuple lock 不是免费
`LOCKTAG_TUPLE` 进入 heavyweight lock manager。
它没有 fast path。
但 PostgreSQL 不为每个 tuple 常驻 lock object。
只有发生需要排队的场景才用。
这避免了按 tuple 数量膨胀 shared memory。
代价是源码路径不直观：
持久行锁状态在 tuple header。
短期排队在 tuple lock。
事务结局等待在 transactionid lock。
## 10. 观测与诊断入口
### 10.1 最小 SQL 现象
准备：
```sql
CREATE TABLE tx_wait_demo(id int primary key, v int);
INSERT INTO tx_wait_demo VALUES (1, 0);
```
Session A：
```sql
BEGIN;
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 1;
```
Session B：
```sql
BEGIN;
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 1;
```
Session C 观察：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE datname = current_database()
ORDER BY pid;
```
常见结果：
```text
Session B:
  wait_event_type = Lock
  wait_event = transactionid
```
这就是本节 runtime truth。
SQL 冲突点是同一行。
等待对象是 transactionid。
### 10.2 `pg_locks`
观察等待项：
```sql
SELECT pid, locktype, mode, transactionid, relation, page, tuple, granted
FROM pg_locks
WHERE locktype IN ('transactionid', 'tuple', 'relation')
ORDER BY pid, locktype, granted;
```
常见可见事实：
```text
blocker 持有 transactionid ExclusiveLock，granted=true。
waiter 等待同一个 transactionid 的 ShareLock，granted=false。
可能还能看到 relation RowExclusiveLock。
tuple lock 是否出现取决于具体路径和时机。
```
不要要求 `pg_locks` 一定显示 tuple lock。
长等待主对象是 transactionid。
tuple lock 可能是短暂排队辅助。
### 10.3 blocker 查询
使用：
```sql
SELECT blocked.pid AS blocked_pid,
       blocker.pid AS blocker_pid,
       blocked.wait_event,
       blocker.state,
       blocker.query
FROM pg_stat_activity blocked
JOIN LATERAL unnest(pg_blocking_pids(blocked.pid)) AS b(pid) ON true
JOIN pg_stat_activity blocker ON blocker.pid = b.pid
WHERE blocked.wait_event_type = 'Lock';
```
`pg_blocking_pids()` 基于 lock manager 当前等待图。
它能告诉你当前 blocking backend。
它不能直接告诉你哪个 heap tuple header 的 `xmax` 指向 blocker。
要定位行，通常结合应用 SQL、主键条件、`ctid` 实验或日志上下文。
### 10.4 server log
设置：
```sql
SET deadlock_timeout = '200ms';
SET log_lock_waits = on;
```
长等待日志可能显示：
```text
process ... still waiting for ShareLock on transaction ... after ... ms
```
如果最终死锁，错误上下文可能显示：
```text
while updating tuple (block,offset) in relation "tx_wait_demo"
```
这两个片段合起来正是本节主题。
等待对象是 transactionid。
业务冲突位置是 tuple。
### 10.5 能看到什么
能直接看到：
```text
当前 backend 的 wait_event。
pg_locks 中 waiting/granted transactionid locks。
blocking pid。
server log 中等待时长和 lock tag。
deadlock error context 中的 tuple block/offset。
```
只能推断：
```text
哪个 SQL 条件命中了同一行。
tuple header 当前 xmax 是否正是 blocker xid。
MultiXact 中哪些成员仍存活。
醒来后会继续、跳过还是报并发更新。
```
几乎不可见或需要源码/扩展/调试：
```text
heap_update() restart 次数。
等待前后 xmax_infomask_changed() 的真实变化。
Do_MultiXactIdWait() 遍历了哪些成员。
tuple lock 排队瞬间的先后顺序。
```
### 10.6 gdb / 源码断点
源码跟读可设置断点：
```text
heap_update
HeapTupleSatisfiesUpdate
heap_acquire_tuplock
XactLockTableWait
ConditionalXactLockTableWait
MultiXactIdWait
Do_MultiXactIdWait
ProcSleep
ProcLockWakeup
ProcReleaseLocks
LockReleaseAll
```
断点观察顺序：
```text
B 在 HeapTupleSatisfiesUpdate() 得到 TM_BeingModified；
B 复制 xwait 和 infomask；
B 调用 heap_acquire_tuplock()；
B 进入 XactLockTableWait(xwait)；
A commit 时经 ProcReleaseLocks() / LockReleaseAll() 释放 transactionid lock；
B 醒来后重新锁 buffer 并检查 xmax 是否变化。
```
不要在持有 buffer content lock 时单步太久。
调试会改变并发时序。
实验只用于理解状态流，不要把时间测量当性能结论。
## 11. 常见误区
### 误区 1：看到 transactionid wait 就不是行锁问题
错误。
行级 update 冲突常见表现正是 transactionid wait。
因为等待完成条件是另一个事务结束。
诊断时要从 transactionid wait 反推 blocker，再回到 SQL 和 tuple。
### 误区 2：tuple lock 才是 row lock 的全部实现
错误。
PostgreSQL 行锁的长期状态主要在 tuple header 的 `xmax` / MultiXact。
`LOCKTAG_TUPLE` 是辅助排队机制。
transactionid lock 是等待事务结局的机制。
三者组合才是 row-level conflict 的完整实现。
### 误区 3：`xmax` 有值表示 tuple 已删除
错误。
`xmax` 可能是 updater、deleter、locker 或 MultiXact。
还可能是 abort 后留下的无效痕迹。
必须结合 `infomask`、hint bits、pg_xact 和 visibility 解释。
### 误区 4：等待 transactionid lock 返回后就可以直接更新
错误。
等待返回只表示目标 XID 不再 in progress。
heap 路径必须重新检查 tuple header。
如果前一个事务 commit 并更新了 tuple，当前操作可能返回 `TM_Updated`。
如果前一个事务 abort 或只是 lock-only，当前操作才可能继续。
### 误区 5：MultiXact wait 是等待一个 MultiXact 对象结束
错误。
MultiXactId 是成员集合编号。
真正结束的是成员事务。
等待路径会遍历成员并等待冲突成员 XID。
### 误区 6：deadlock detector 需要理解 heap tuple
错误。
一旦等待进入 transactionid lock，deadlock detector 只看 lock manager 等待图。
heap tuple 信息只用于错误上下文和醒来后的语义判断。
### 误区 7：`pg_locks` 能完整解释行锁因果
错误。
`pg_locks` 展示 lock manager 状态。
它不展示完整 tuple header、HOT chain、MultiXact 成员细节和 heap restart 次数。
它是入口，不是完整因果模型。
## 12. 课堂实验
### 实验 1：复现 update/update 的 transactionid wait
目标：看到“行级冲突 -> transactionid wait”。
步骤：
```sql
DROP TABLE IF EXISTS tx_wait_demo;
CREATE TABLE tx_wait_demo(id int primary key, v int);
INSERT INTO tx_wait_demo VALUES (1, 0);
```
Session A：
```sql
BEGIN;
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 1;
SELECT pg_backend_pid();
```
Session B：
```sql
BEGIN;
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 1;
```
Session C：
```sql
SELECT a.pid, a.wait_event_type, a.wait_event, l.locktype,
       l.mode, l.transactionid, l.granted
FROM pg_stat_activity a
LEFT JOIN pg_locks l ON l.pid = a.pid
WHERE a.datname = current_database()
ORDER BY a.pid, l.granted, l.locktype;
```
解释要求：
```text
指出 B 等待的是 transactionid ShareLock。
指出 A 持有同一 transactionid 的 ExclusiveLock。
说明冲突仍然来自同一 heap tuple。
```
继续：
```sql
-- Session A
COMMIT;
```
观察 B 继续完成。
如果 A 改成 `ROLLBACK`，B 会继续更新旧版本。
如果 A commit，B 可能基于新状态继续 EvalPlanQual 或返回并发更新结果，具体取决于隔离级别和执行路径。
### 实验 2：NOWAIT 与 SKIP LOCKED
目标：比较阻塞等待和条件等待。
Session A：
```sql
BEGIN;
SELECT * FROM tx_wait_demo WHERE id = 1 FOR UPDATE;
```
Session B：
```sql
SELECT * FROM tx_wait_demo WHERE id = 1 FOR UPDATE NOWAIT;
```
期望：
```text
立即报 could not obtain lock on row。
```
Session B 改为：
```sql
SELECT * FROM tx_wait_demo WHERE id = 1 FOR UPDATE SKIP LOCKED;
```
期望：
```text
返回空结果而不是阻塞。
```
源码对应：
```text
LockWaitError -> ConditionalXactLockTableWait() 失败后 ERROR。
LockWaitSkip  -> ConditionalXactLockTableWait() 失败后 TM_WouldBlock。
```
### 实验 3：deadlock 的 transactionid wait 图
目标：证明行级 deadlock 进入通用 lock manager。
准备两行：
```sql
TRUNCATE tx_wait_demo;
INSERT INTO tx_wait_demo VALUES (1, 0), (2, 0);
```
Session A：
```sql
BEGIN;
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 1;
```
Session B：
```sql
BEGIN;
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 2;
```
Session A：
```sql
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 2;
```
Session B：
```sql
UPDATE tx_wait_demo SET v = v + 1 WHERE id = 1;
```
观察：
```text
一个事务等待另一个事务的 transactionid。
deadlock_timeout 后，其中一个事务报 deadlock detected。
```
讨论：
```text
deadlock detector 没有扫描 heap page。
它只需要 transactionid lock wait graph。
```
### 实验 4：源码断点画状态图
目标：把 SQL 现象回到源码。
断点：
```text
heap_update
HeapTupleSatisfiesUpdate
heap_acquire_tuplock
XactLockTableWait
ProcSleep
ProcReleaseLocks
LockReleaseAll
```
记录表：
```text
时间点
函数
xwait
infomask
wait_event
pg_locks locktype/mode/granted
是否持有 tuple lock
醒来后 xmax 是否变化
```
要求画出：
```text
tuple header 发现冲突
  -> tuple lock 建立排队
  -> transactionid wait 睡眠
  -> blocker 释放 XID lock
  -> waiter recheck tuple header
```
### 实验 5：MultiXact 路径观察
目标：理解多个 locker 为什么仍等待成员 XID。
可以用多个 session 对同一行执行兼容的 `FOR SHARE` 或 `FOR KEY SHARE`。
再用另一个 session 尝试 `UPDATE`。
观察 `pg_locks` 和 wait event。
源码跟读：
```text
DoesMultiXactIdConflict()
MultiXactIdWait()
Do_MultiXactIdWait()
XactLockTableWait(memxid, ...)
```
讨论：
```text
为什么 MultiXactId 不是一个会结束的事务？
为什么等待要落到 member xid？
```
## 13. 讨论题
1. 如果 PostgreSQL 把等待队列直接挂在 heap tuple header 上，会遇到哪些 shared memory、buffer、WAL 和 crash recovery 问题？
2. 为什么等待 transactionid lock 返回后，`heap_update()` 仍必须检查 `xmax` 和 `infomask` 是否变化？
3. `LOCKTAG_TUPLE` 已经存在，为什么还需要 `XactLockTableWait()`？
4. `xmax` 是 MultiXactId 时，为什么不能等待 MultiXactId 本身“提交”？
5. `pg_locks` 中只看到 transactionid wait 时，如何定位到可能的业务行冲突？
6. `NOWAIT` 和 `SKIP LOCKED` 在源码上改变了什么？没有改变什么？
7. 行级 deadlock 为什么能被通用 heavyweight lock detector 发现？
8. 高并发热点行的主要成本是 CPU、buffer lock、lock manager，还是事务持有时间？不同 workload 下答案如何变化？
## 14. 本节小结
本节唯一主问题是：
```text
tuple update 冲突为什么常表现为等待 transactionid lock，而不是等待 heap tuple 本身？
```
答案是：
```text
冲突在 tuple 上被发现；
但等待完成条件是另一个事务结束；
因此长等待对象绑定到 transactionid。
```
核心链路：
```text
heap_update() / heap_delete() / heap_lock_tuple()
  -> HeapTupleSatisfiesUpdate() 返回 TM_BeingModified
  -> 从 tuple header 复制 xwait / infomask
  -> 必要时 heap_acquire_tuplock() 建立 tuple 排队优先级
  -> XactLockTableWait() 或 MultiXactIdWait()
  -> LockAcquire(LOCKTAG_TRANSACTION, ShareLock)
  -> blocker 事务结束释放 ExclusiveLock
  -> waiter 醒来，重新检查 tuple header
```
核心状态边界：
```text
tuple header:
  记录 `xmax`、MultiXact、lock-only、update/delete、ctid 链。
LOCKTAG_TUPLE:
  辅助同一 tuple 上 waiter 排队。
LOCKTAG_TRANSACTION:
  等待事务结局。
MultiXact:
  把多个 locker/updater 成员编码进 tuple xmax。
PGPROC / LOCK / PROCLOCK:
  承载真正 wait queue、wakeup、deadlock detection 和诊断投影。
```
ownership 与 cleanup：
```text
事务取得 XID 时持有自己的 transactionid ExclusiveLock；
事务结束释放；
waiter 请求 ShareLock 并在成功后立即释放；
tuple lock 由等待者短期持有，正常返回或 ERROR cleanup 释放；
tuple header 的 abort 痕迹由后续 hint bit、visibility 和 pruning 处理。
```
正确性来自分层组合。
MVCC 决定版本可见性。
buffer lock 保护页内读写。
transactionid lock 等待事务结束。
tuple lock 建立局部排队。
MultiXact 表达多个行锁成员。
deadlock detector 处理 lock manager 等待图。
没有任何单一字段能解释全部语义。
诊断时能直接看到的是 wait event、`pg_locks`、blocking pid 和日志。
只能推断的是具体 tuple header、MultiXact 成员和醒来后的 heap 结果。
本节可迁移规律是：
```text
系统中“冲突发生的位置”和“等待完成条件”可以不同。
好的等待对象应该绑定到完成条件，并复用已有的等待、唤醒、清理和死锁检测基础设施。
```
下一节 advisory lock 会换一个方向看同一套 lock manager：
用户自定义 key 如何在 session owner 和 transaction owner 之间切换释放边界。
