# PostgreSQL lock wait queue 与 deadlock detector
## 课程定位
前置知识：已经理解 `LOCKTAG`、lock method、`LOCK`、`PROCLOCK`、`LOCALLOCK`、fast path relation lock、ResourceOwner 和 LWLock 分区保护。
本节唯一主问题：
```text
等待队列、soft edge、deadlock timeout 和 detector 如何判断是否必须 ERROR？
```
核心矛盾：
```text
每次 lock wait 都立刻全局死锁检测会把正常短等待变成昂贵慢路径；
但一旦等待图真的形成环，系统又不能让事务永久睡眠；
同时，wait queue 的到达顺序既要提供可预测的 grant 顺序，也会制造可被重排消除的 soft edge。
```
一句话运行模型：
```text
PostgreSQL 先把等待者编码成 LOCK.waitProcs 中的 PGPROC；
等待超过 deadlock_timeout 后才用当前 lock table 构造 hard edge 和 soft edge；
如果 soft edge 可以通过 wait queue reorder 消环，就继续等待；
如果所有可行 reorder 都不能消环，当前检测者从队列移除并由 DeadLockReport() 抛 ERROR。
```
学完后应能独立判断：
- 为什么 `deadlock_timeout` 不是最长等待时间。
- 为什么 `pg_blocking_pids()` 可能返回排在前面的等待者，而不只是已持锁者。
- 为什么 `ProcLockWakeup()` 不能简单唤醒所有与已授予锁不冲突的 waiter。
- 为什么 deadlock detector 要区分 hard edge 和 soft edge。
- 为什么 soft deadlock 可以不 ERROR，而 hard deadlock 必须中断一个等待请求。
- 为什么 detector 不扫描 relation fast-path array 仍然不会漏掉真实死锁。
- 为什么被唤醒的 waiter 不再自己重新 grant shared lock state。
本课基于本地源码：
```text
/home/nail/postgres
commit 0e1f1ed157e9
```
本节只讲 heavyweight lock wait queue 和 deadlock detector。
LWLock 等待、condition variable、predicate lock 的冲突检测不在本节展开。
## 1. 本节在总主线中的位置
49 节建立了 `LOCKTAG`、lock method table 和 `LOCK` / `PROCLOCK` / `LOCALLOCK` 的基本状态边界。
50 节跟过 `LockAcquireExtended()` 和 `LockRelease()` 的 acquire / release 生命周期。
51 节解释 relation fast path 为什么能绕开 main lock table，又何时必须迁回。
本节接住 50 节留下的等待问题。
一把 heavyweight lock 不能立即 grant 时，系统不只是把 backend 挂起。
它必须同时维护四件事：
- shared lock table 中的 requested / granted 计数。
- `LOCK.waitProcs` 中的等待队列顺序。
- `PGPROC` 上可被其它 backend 读取的等待状态。
- 一个延迟触发的 deadlock detector。
这四件事合起来回答本节主问题。
`wait queue` 不是一个普通 FIFO。
它是 lock manager correctness 的一部分。
排在前面的 waiter 即使还没拿到锁，也可能阻止后面的 waiter 被唤醒。
这种阻塞不是已经持有锁造成的。
PostgreSQL 把它称为 soft edge。
hard edge 来自已经授予的冲突锁。
soft edge 来自同一个 wait queue 中排在前面的冲突请求。
detector 的核心判断是：
```text
如果环只靠 hard edge 形成，或者 soft edge 无法找到一致 reorder 消除：
  必须 ERROR。
如果环中存在可反转的 soft edge，并且重排后所有相关等待图无环：
  reorder wait queue，不 ERROR。
```
这个设计保留了两个目标。
短等待不付全局图搜索成本。
真正死锁最终一定由某个等待者打破。
本节的 runtime truth 是：
```text
一个 backend 在 pg_stat_activity 中 wait_event='Lock'；
pg_locks 显示 granted=false；
超过 deadlock_timeout 后，日志可能出现 still waiting、avoided deadlock 或 deadlock detected；
源码上对应 ProcSleep() 中的 timeout、CheckDeadLock() 和 DeadLockCheck()。
```
## 2. 核心矛盾与一句话运行模型
先把等待路径压缩成一条时间线：
```text
LockAcquireExtended()
  -> conflict check 发现不能 grant
  -> JoinWaitQueue()
  -> MyProc 进入 lock->waitProcs
  -> WaitOnLock()
  -> ProcSleep()
  -> deadlock_timeout 到期后 CheckDeadLock()
  -> DeadLockCheck()
  -> DS_NO_DEADLOCK / DS_SOFT_DEADLOCK / DS_HARD_DEADLOCK
```
这条链的第一个关键选择是延迟检测。
绝大多数 lock wait 是正常排队。
holder 很快提交或释放锁。
如果每次入队都扫描所有 lock partitions 和 wait queues，正常 workload 会为罕见死锁付固定成本。
所以 `ProcSleep()` 先睡。
它只设置 `DEADLOCK_TIMEOUT`。
timeout 到期后才把“我等太久了”转化为“现在值得检查等待图”。
第二个关键选择是让 wait queue 本身表达部分 dependency。
`ProcLockWakeup()` 唤醒 waiter 时看两个条件：
```text
request 不与已经 grant 的锁冲突；
request 不与前面尚未可唤醒 waiter 的 request 冲突。
```
第二个条件就是 soft edge 的来源。
如果队列中 B 在 A 前面，且 A 的请求与 B 的请求冲突，那么 A 不能绕过 B。
哪怕 B 还没有持有锁，A 也在等待 B 的队列优先级。
第三个关键选择是 detector 优先尝试 reorder。
soft edge 与 hard edge 不同。
hard edge 表示 blocker 已经持有冲突锁。
不能通过移动等待队列改变这个事实。
soft edge 只表示队列顺序。
如果把 A 移到 B 前面能消除等待图中的环，系统可以避免 abort。
第四个关键选择是只让一个请求失败。
`DeadLockReport()` 抛出的 ERROR 不是“杀掉所有事务”。
它取消当前 start-point 的 lock request。
事务 abort 后释放它已经持有的 locks，等待图自然断开。
这也是为什么 `DeadLockCheck()` 只需要确认涉及当前等待者的环。
不涉及当前 start-point 的环应该由环上的进程自己检测并打破。
一句话模型可以写得更机械：
```text
wait queue 先保存真实等待顺序；
deadlock timeout 只是启动一次昂贵检查；
detector 把已授予冲突变成 hard edge，把队列前序冲突变成 soft edge；
soft edge 能重排就重排，不能重排才 ERROR。
```
这也是本节所有源码阅读的主轴。
## 3. 核心源码与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/README` | deadlock algorithm、hard edge、soft edge、wait queue reorder 的设计解释。 |
| 2 | `src/include/storage/lock.h` | `LOCK.waitProcs`、`waitMask`、`DeadLockState`、`LockInstanceData`。 |
| 3 | `src/include/storage/proc.h` | `PGPROC.waitLock`、`waitProcLock`、`waitLockMode`、`waitStatus`、`waitStart`。 |
| 4 | `src/backend/storage/lmgr/lock.c` | `LockAcquireExtended()`、`WaitOnLock()`、`RemoveFromWaitQueue()`、`GetBlockerStatusData()`。 |
| 5 | `src/backend/storage/lmgr/proc.c` | `JoinWaitQueue()`、`ProcSleep()`、`ProcLockWakeup()`、`CheckDeadLock()`、`LockErrorCleanup()`。 |
| 6 | `src/backend/storage/lmgr/deadlock.c` | `DeadLockCheck()`、`FindLockCycle()`、`ExpandConstraints()`、`TopoSort()`、`DeadLockReport()`。 |
| 7 | `src/backend/utils/misc/timeout.c` | timeout framework 如何启停 `DEADLOCK_TIMEOUT` 和 `LOCK_TIMEOUT`。 |
| 8 | `src/backend/utils/init/postinit.c` | `RegisterTimeout(DEADLOCK_TIMEOUT, CheckDeadLockAlert)` 的注册位置。 |
| 9 | `src/backend/tcop/postgres.c` | signal / interrupt 路径如何把 deadlock timeout 转成 ProcSleep 可处理的 latch wakeup。 |
| 10 | `src/backend/utils/adt/lockfuncs.c` | `pg_locks` 和 `pg_blocking_pids()` 的观测入口。 |
推荐阅读顺序不是文件名顺序。
先读 `lock.h` / `proc.h` 的状态字段。
再读 `JoinWaitQueue()` 如何写入这些字段。
然后读 `ProcSleep()` 如何把等待转换成 timeout-driven detector。
最后读 `deadlock.c` 的图搜索和重排。
不要先从 `DeadLockCheckRecurse()` 递归细节开始。
如果还不知道 `soft edge` 从哪里来，递归只会像一个抽象图算法。
本节要始终把算法压回这个事实：
```text
edge 来自 shared lock table 和 wait queue 的真实状态。
```
## 4. 关键结构与状态边界
### 4.1 `LOCK`：对象级等待队列
`LOCK` 是每个 lockable object 的 shared state。
本节最关键的字段不是 tag，而是这些运行时计数和队列：
```c
LOCKMASK waitMask;
dclist_head waitProcs;
int requested[MAX_LOCKMODES];
int granted[MAX_LOCKMODES];
```
`requested[]` 包含已经 grant 的请求，也包含正在等待的请求。
`granted[]` 只包含已经授予的请求。
`waitMask` 是 `requested[mode] > granted[mode]` 的压缩表达。
`waitProcs` 链接的是等待中的 `PGPROC`，不是 `PROCLOCK`。
这意味着等待队列的节点是 backend 身份。
但每个等待节点仍通过 `PGPROC.waitProcLock` 指回它在这个 lock object 上的 `PROCLOCK`。
`LOCK` 只在 main lock table 中存在。
fast-path relation lock 不创建 `LOCK`。
但一旦可能等待，相关状态必须在 main table。
这就是上一节说 detector 不扫 fast path 仍然正确的原因。
可能参与等待图的请求已经转入 shared `LOCK` / `PROCLOCK`。
### 4.2 `PROCLOCK`：backend 对对象的 shared interest
`PROCLOCK` 表达一个 backend 或 lock group 对一个 `LOCK` 的 shared interest。
本节关注两个事实：
- `holdMask` 表示已经授予的 lock modes。
- 即使 `holdMask == 0`，正在等待的 backend 也可能有 `PROCLOCK`。
hard edge 的判断来自 `holdMask`。
如果 A 正在等某 mode，而 B 的 `PROCLOCK.holdMask` 中有冲突 mode，A -> B 是 hard edge。
等待中的 `PROCLOCK` 也在 `LOCK.procLocks` 链表里。
这让观测和 detector 可以从同一个 `LOCK` 找到所有 holder 和 waiter。
### 4.3 `PGPROC`：等待图节点
`PGPROC` 是 wait queue 的节点。
本节最关键字段是：
```c
LOCK       *waitLock;
dlist_node waitLink;
PROCLOCK   *waitProcLock;
LOCKMODE    waitLockMode;
LOCKMASK    heldLocks;
ProcWaitStatus waitStatus;
pg_atomic_uint64 waitStart;
```
`waitLock == NULL` 表示当前没有在 heavyweight lock 上等待。
`waitLink` 表示当前节点在 `waitLock->waitProcs` 中的位置。
`waitProcLock` 指向这次等待对应的 `PROCLOCK`。
`waitLockMode` 是请求强度。
`heldLocks` 保存本 backend 在同一 lock object 上已经持有的模式。
`waitStatus` 是 waiter 与 grantor / detector 的通信结果。
`waitStart` 用于 `pg_locks.waitstart` 和等待统计。
字段组合才是语义。
只看 `waitStatus` 不够。
只看 `waitLock` 也不够。
真正等待状态是：
```text
waitLock != NULL
  + waitLink attached
  + waitProcLock 指向目标 PROCLOCK
  + waitStatus == PROC_WAIT_STATUS_WAITING
```
### 4.4 `LOCALLOCK`：本地 ownership 与 cleanup 锚点
`LOCALLOCK` 是 backend-local。
等待期间它提供三个关键锚点：
- `tag` 说明正在等哪个 lock object 和 mode。
- `lock` / `proclock` 指向 shared state。
- `hashcode` 找回对应 partition LWLock。
`WaitOnLock()` 把当前 `LOCALLOCK` 记录到 backend-local 的 awaited state。
`LockErrorCleanup()` 依赖这个指针在 ERROR、cancel 或 die interrupt 后退出 wait queue。
所以 `LOCALLOCK` 不是 wait queue 节点。
它是 cleanup 能回到正确 shared lock partition 的索引。
### 4.5 `DeadLockState`
`lock.h` 中的返回状态是 detector 和 `ProcSleep()` 的协议：
```c
DS_NO_DEADLOCK
DS_SOFT_DEADLOCK
DS_HARD_DEADLOCK
DS_BLOCKED_BY_AUTOVACUUM
```
`DS_NO_DEADLOCK` 表示这次检查没有发现需要处理的环。
`DS_SOFT_DEADLOCK` 表示通过 wait queue reorder 避免了 deadlock。
`DS_HARD_DEADLOCK` 表示没有可行 reorder，当前请求必须失败。
`DS_BLOCKED_BY_AUTOVACUUM` 表示没有 hard deadlock，但可以尝试取消直接阻塞自己的 autovacuum worker。
`DS_NOT_YET_CHECKED` 只在 `ProcSleep()` 本地使用。
它区分“还没到 deadlock_timeout”和“已经检查过但还要继续等”。
### 4.6 `EDGE`、`WAIT_ORDER`、`DEADLOCK_INFO`
`deadlock.c` 内部有三组 workspace。
`EDGE` 表示 waits-for graph 中的一条边。
其中 `waiter` 和 `blocker` 存的是 lock group leader。
这让并行查询的 lock group 作为整体参与检测。
`WAIT_ORDER` 表示某个 `LOCK` 的候选 wait queue 顺序。
detector 不会一边搜索一边真实修改队列。
它先在 lookaside table 中构造候选顺序。
`DEADLOCK_INFO` 是错误报告用的快照。
源码特意把 `LOCKTAG`、`LOCKMODE`、pid 拷贝出来。
原因是 `DeadLockReport()` 在释放所有 lock partition LWLocks 后才打印错误。
不能在那时再信任 shared pointer。
### 4.7 timeout 状态
`proc.c` 中有一个 backend-local signal flag：
```c
static volatile sig_atomic_t got_deadlock_timeout;
```
`CheckDeadLockAlert()` 在 signal handler 中只做很少的事：
```text
got_deadlock_timeout = true
SetLatch(MyLatch)
```
真正的 detector 不在 signal handler 里跑。
`ProcSleep()` 被 latch 唤醒后，在普通执行上下文调用 `CheckDeadLock()`。
这条边界非常重要。
deadlock detector 要拿所有 lock partition LWLocks。
它不能在异步信号处理函数里直接做。
## 5. 主流程 walkthrough：从冲突到 wait queue
入口仍然是 `LockAcquireExtended()`。
前面 fast path、local re-entry、shared hash table setup 已经在 50 和 51 节讲过。
本节从 main lock table 中发现冲突开始。
简化调用链是：
```text
LockAcquireExtended()
  -> SetupLockInTable()
  -> LockCheckConflicts()
  -> JoinWaitQueue()
  -> WaitOnLock()
  -> ProcSleep()
```
在进入 `JoinWaitQueue()` 前，`LOCK` 和 `PROCLOCK` 已经存在。
请求计数也已经体现出“有人想要这把锁”。
这解释了为什么等待中的请求会影响后续请求。
它不是进程私有意愿。
它已经进入 shared lock object 的 requested state。
### 5.1 先看 waitMask
`LockAcquireExtended()` 的冲突判断先看 `waitMask`。
如果目标 mode 与 `waitMask` 中已有等待请求冲突，新请求必须进入等待逻辑。
这一步让“前序等待者”具有队列优先权。
否则 later compatible-with-holder 的请求可能不断绕过 earlier exclusive waiter。
之后才调用 `LockCheckConflicts()` 检查已经持有的 locks。
顺序体现了两个不同问题：
- 已经持有的锁是否阻塞我。
- 已经排队的冲突请求是否应该先于我。
`waitMask` 不是 deadlock detector 的完整图。
它是 hot path 上的快速保守信号。
真正的 edge 还要在 `JoinWaitQueue()` 和 `deadlock.c` 中结合具体队列顺序推导。
### 5.2 `JoinWaitQueue()` 的 priority insertion
`JoinWaitQueue()` 不总是把自己放到队尾。
它先看自己在同一个 object 上已经持有哪些锁。
如果自己已持有的锁会阻塞某个前序 waiter，就尝试插到那个 waiter 前面。
源码注释说这不是必须的。
即使不做，deadlock detector 之后也能重排。
但提前处理可以避免一次 `deadlock_timeout` 延迟。
这段逻辑还会检查一个 two-way early deadlock。
形态是：
```text
我已经持有的锁阻塞前序 waiter；
前序 waiter 已持有的锁也阻塞我；
```
这时 `RememberSimpleDeadLock()` 先记录错误细节。
`JoinWaitQueue()` 返回 `PROC_WAIT_STATUS_ERROR`。
调用者会走 `DeadLockReport()`。
这是少数不等 `deadlock_timeout` 的 deadlock 路径。
### 5.3 `dontWait` 与 wait queue
`dontWait` 的请求也会进入 `JoinWaitQueue()`。
原因是 `JoinWaitQueue()` 可能发现特殊情况：插队点之前没有冲突，且已持锁状态也允许 grant。
这样即使调用者说不要阻塞，也可能立即成功。
如果确认真的需要睡眠，`dontWait` 返回 `PROC_WAIT_STATUS_ERROR`。
这不是 deadlock。
`LockAcquireExtended()` 会清理 shared request 计数和临时 `PROCLOCK`，返回 `LOCKACQUIRE_NOT_AVAIL`。
因此看到 `PROC_WAIT_STATUS_ERROR` 不能直接解释为死锁。
必须结合路径：
- `dontWait=true` 是非阻塞失败。
- early deadlock 或 detector hard deadlock 才会 `DeadLockReport()`。
### 5.4 写入 `PGPROC`
真正入队时，`JoinWaitQueue()` 做三件事。
第一，把 `MyProc->waitLink` 插入 `lock->waitProcs`。
第二，设置 `lock->waitMask` 对应 bit。
第三，填充 `MyProc` 等待字段：
```text
heldLocks = myProcHeldLocks
waitLock = lock
waitProcLock = proclock
waitLockMode = lockmode
waitStatus = PROC_WAIT_STATUS_WAITING
```
从这一刻开始，其它 backend 能把当前 backend 当作等待图节点。
`pg_locks` 能看到 `granted=false`。
`pg_blocking_pids()` 能找到它等待的 `LOCK`。
deadlock detector 能沿着它的 outgoing edges 继续搜索。
### 5.5 释放 partition LWLock 后睡眠
`LockAcquireExtended()` 在 `PROC_WAIT_STATUS_WAITING` 后释放当前 lock partition LWLock。
然后调用 `WaitOnLock()`。
这一点很关键。
heavyweight lock wait 可以长时间睡眠。
不能持有保护 lock table 的 LWLock 睡眠。
等待者能安全睡眠，是因为它的等待状态已经完整写入 shared memory。
之后 grantor、detector 或 cleanup 都能找到它。
## 6. 主流程 walkthrough：`WaitOnLock()` 与 `ProcSleep()`
`WaitOnLock()` 是 shared lock state 和可中断睡眠之间的边界。
它先设置 error context。
这样 lock timeout 或 deadlock error 能报告“waiting for X on Y”。
然后设置进程标题后缀为 waiting。
再把当前 `LOCALLOCK` 存为 awaited lock。
简化模型：
```text
awaitedLock = locallock
awaitedOwner = owner
ProcSleep(locallock)
awaitedLock = NULL
```
注释里有本节最重要的 cleanup 规则：
```text
ProcSleep() 返回后，不应再做必要的 shared-state cleanup。
```
原因是等待中可能收到 cancel 或 die interrupt。
也可能刚被 grant 后还没来得及运行用户态代码就被 interrupt。
因此 grantor 必须在唤醒前把 shared state 完全改好。
失败 cleanup 必须能由 `LockErrorCleanup()` 独立完成。
### 6.1 timeout 安装
`ProcSleep()` 进入时先确认 `awaitedLock` 已设置。
然后初始化：
```text
deadlock_state = DS_NOT_YET_CHECKED
got_deadlock_timeout = false
```
非 Hot Standby 路径会启用 `DEADLOCK_TIMEOUT`。
如果 `LockTimeout > 0`，同时启用 `LOCK_TIMEOUT`。
源码使用 `enable_timeouts()` 一次设置两个 timeout，减少重复取时间成本。
`waitStart` 使用 deadlock timeout 的 start time。
这解释了 `pg_locks.waitstart` 的一个边界：
它是在不持 lock partition LWLock 的情况下写入的。
源码承认很短时间内可能看到 `granted=false` 但 `waitstart` 仍是 NULL。
这不是语义矛盾。
只是观测采样边界。
### 6.2 latch loop
普通路径的等待是：
```text
WaitLatch(MyLatch, WL_LATCH_SET | WL_EXIT_ON_PM_DEATH, 0, wait_event)
ResetLatch(MyLatch)
if got_deadlock_timeout:
  deadlock_state = CheckDeadLock()
CHECK_FOR_INTERRUPTS()
read MyProc->waitStatus once
```
latch 被 set 不代表锁已经可用。
很多事件都会 set latch。
所以循环必须读取 `MyProc->waitStatus`。
`waitStatus` 才是 grantor 或 detector 传来的结果。
这也是为什么 `ProcWakeup()` 在 `SetLatch()` 前设置 `waitStatus`。
### 6.3 `deadlock_timeout` 的真实语义
`deadlock_timeout` 到期只做一件事：
让当前等待者运行一次 `CheckDeadLock()`。
它不是最长等待时间。
如果结果是 `DS_NO_DEADLOCK`，backend 继续睡。
如果结果是 `DS_SOFT_DEADLOCK`，队列已经被重排，backend 继续按新顺序等待或被唤醒。
如果结果是 `DS_HARD_DEADLOCK`，`CheckDeadLock()` 已把当前 backend 从 wait queue 移除并设置 `waitStatus=ERROR`。
`ProcSleep()` 看到 error 后返回。
真正的 ERROR 由上层 `DeadLockReport()` 抛出。
`lock_timeout` 是另一回事。
`LOCK_TIMEOUT` handler 触发 SIGINT。
之后 `ProcessInterrupts()` 报 lock timeout。
这会走 `LockErrorCleanup()`，但错误码和语义不是 deadlock detected。
### 6.4 log 和 stats
超过 `deadlock_timeout` 后，如果 `log_lock_waits` 打开，`ProcSleep()` 会打印等待日志。
可能的日志包括：
- still waiting。
- acquired after waiting。
- avoided deadlock by rearranging queue order。
- detected deadlock while waiting。
等待统计通过 `pgstat_count_lock_waits()` 更新。
它只在已经运行过 deadlock timeout 检查、并且最终成功拿到锁时累计等待。
这意味着短于 `deadlock_timeout` 的等待不会进入这组 lock wait time 统计。
诊断时不要把统计值理解成全部 lock wait 总量。
## 7. 主流程 walkthrough：release 与 `ProcLockWakeup()`
释放路径在 `LockRelease()`、`CleanUpLock()` 和 `ProcLockWakeup()` 之间完成。
本节只看唤醒规则。
`ProcLockWakeup()` 扫描 `lock->waitProcs`。
它维护一个本地变量：
```text
aheadRequests
```
含义是前面那些尚未被唤醒的 waiter 的请求模式集合。
对当前 waiter，唤醒条件是：
```text
conflictTab[waitLockMode] & aheadRequests == 0
and LockCheckConflicts(...) == false
```
第一个条件保护队列顺序。
第二个条件保护已经授予的 holder 状态。
如果两个条件都满足，`ProcLockWakeup()` 会先调用 `GrantLock()`。
然后调用 `ProcWakeup()`。
`ProcWakeup()` 做这些事：
```text
从 waitProcs 删除 waitLink
waitLock = NULL
waitProcLock = NULL
waitStatus = PROC_WAIT_STATUS_OK
waitStart = 0
SetLatch(procLatch)
```
注意顺序。
grant 发生在唤醒之前。
waiter 醒来后不再重新竞争 lock table。
它只需要在本地调用 `GrantLockLocal()` 更新 `LOCALLOCK` 和 ResourceOwner。
这避免一个 race：
```text
如果先唤醒 waiter，再让 waiter 自己 grant；
新的请求可能插入并观察到不完整状态。
```
因此 heavyweight lock 的 wait 模型不是 LWLock retry 模型。
它是 grantor-completes-shared-state 模型。
## 8. Deadlock detector walkthrough
`CheckDeadLock()` 是 `ProcSleep()` 在 deadlock timeout 后调用的入口。
它先按 partition number 顺序拿下所有 lock partition LWLocks。
这有两个目的。
第一，得到一个全局一致的 lock table / wait queue 快照。
第二，避免多个 partition LWLock 之间形成 LWLock deadlock。
拿锁后先检查自己是否已经被唤醒。
如果 `MyProc->waitLink` 已 detached，说明 grantor 已处理完 shared state。
这时返回 `DS_NO_DEADLOCK`。
否则调用 `DeadLockCheck(MyProc)`。
### 8.1 `DeadLockCheck()` 的三类结果
`DeadLockCheck()` 初始化 workspace：
```text
nCurConstraints = 0
nPossibleConstraints = 0
nWaitOrders = 0
blocking_autovacuum_proc = NULL
```
然后调用 `DeadLockCheckRecurse(proc)` 搜索是否存在可行 configuration。
如果递归报告无解，返回 `DS_HARD_DEADLOCK`。
如果存在 wait queue reorder，真实修改相关 `LOCK.waitProcs`。
修改后对每个被重排的 lock 调用 `ProcLockWakeup()`。
如果重排发生，返回 `DS_SOFT_DEADLOCK`。
如果没有重排但发现直接阻塞自己的 autovacuum worker，返回 `DS_BLOCKED_BY_AUTOVACUUM`。
否则返回 `DS_NO_DEADLOCK`。
### 8.2 hard edge
`FindLockCycleRecurseMember()` 先扫描目标 `LOCK.procLocks`。
它拿当前请求 mode 的 conflict mask。
对每个 `PROCLOCK.holdMask`，如果有冲突 mode，就产生 hard edge。
形态是：
```text
checkProc waits for proc because proc already holds conflicting lock.
```
hard edge 不能通过 wait queue reorder 消除。
holder 已经拿到锁。
除非 holder 自己释放或 abort，等待者不能越过它。
如果搜索沿 hard edge 回到 start point，且环里没有 soft edge，detector 必须返回 hard deadlock。
### 8.3 soft edge
扫描 hard edge 之后，源码再扫描 wait queue。
顺序不能反。
如果同一个 backend 同时 hard-block 和 soft-block 当前请求，应该归为 hard edge。
soft edge 来自排在当前请求之前的冲突 waiter。
形态是：
```text
checkProc waits for earlierProc because earlierProc is ahead in waitProcs
and earlierProc's requested mode conflicts with checkProc's requested mode.
```
这条边存在的原因不是锁已被授予。
它来自 `ProcLockWakeup()` 的队列规则。
只要 earlierProc 还不能唤醒，later conflicting waiter 就不能绕过它。
所以 soft edge 是真实等待关系。
只是它可能通过改变 wait queue 顺序消除。
### 8.4 `FindLockCycle()`
`FindLockCycle()` 从一个 start proc 出发。
它递归沿 hard edge 和 soft edge 搜索。
如果路径回到 start proc，发现一个需要处理的环。
如果路径回到其它已经访问过的 proc，源码把它视为“不涉及 start point 的环”。
当前检测者不负责解决那个环。
这和 README 的论证一致：
取消当前请求无法保证打破不含自己的环。
环上的其它等待者会在自己的 timeout 中处理。
`FindLockCycle()` 在发现环时还收集环中的 soft edges。
如果 soft edge list 为空，这是 hard deadlock。
如果不为空，说明存在潜在 reorder 机会。
### 8.5 `TestConfiguration()`
`TestConfiguration()` 把当前尝试的 soft-edge reversal constraints 展开成候选 wait queue 顺序。
然后检查这个 configuration 是否仍有环。
返回值含义：
```text
0: 无死锁
-1: hard deadlock 或 constraints 自相矛盾
>0: 发现 soft cycle，并返回可尝试的 soft edges
```
它不只检查原始 start proc。
还检查当前 constraints 中涉及的 waiter 和 blocker。
原因是重排一个队列可能消除原环，同时在相关节点上制造新环。
detector 必须证明候选 configuration 没有制造新的死锁。
### 8.6 `ExpandConstraints()` 与 `TopoSort()`
每个要反转的 soft edge 可以理解成一个 partial order：
```text
waiter 必须排在 blocker 前面
```
`ExpandConstraints()` 按 lock 聚合 constraints。
对每个受影响的 `LOCK.waitProcs` 调用 `TopoSort()`。
`TopoSort()` 的目标不是随便排序。
它要满足 partial order，同时尽量保持原队列顺序。
这是 fairness 和最小扰动的折中。
如果 constraints 互相矛盾，topological sort 失败。
失败意味着这组 soft-edge reversal 不能形成一致队列。
递归会尝试其它 soft edge。
### 8.7 真实 reorder
一旦找到可行 configuration，`DeadLockCheck()` 才修改 shared wait queue。
它对每个 `WAIT_ORDER`：
```text
dclist_init(waitQueue)
按候选顺序重新 push PGPROC.waitLink
ProcLockWakeup(lock)
```
重排后立刻调用 `ProcLockWakeup()`。
因为某些 waiter 可能因为顺序改变变得可唤醒。
这就是 soft deadlock 的运行时现象：
系统没有 ERROR。
但 wait queue 顺序变了。
如果 `log_lock_waits` 打开，日志可能出现 avoided deadlock。
## 9. detector 如何判断必须 ERROR
本节主问题可以在这里收束。
PostgreSQL 只有在下面路径判断必须 ERROR。
### 9.1 入队时的 immediate two-way deadlock
`JoinWaitQueue()` 发现：
```text
我持有的锁阻塞前序 waiter；
前序 waiter 已持有的锁阻塞我。
```
它调用 `RememberSimpleDeadLock()` 保存两个等待边。
然后返回 `PROC_WAIT_STATUS_ERROR`。
`LockAcquireExtended()` 在非 `dontWait` 路径调用 `DeadLockReport()`。
这条路径不等 `deadlock_timeout`。
因为环已经在当前局部队列中明确成立。
### 9.2 timeout 后的 hard deadlock
更常见路径是：
```text
ProcSleep()
  -> got_deadlock_timeout
  -> CheckDeadLock()
  -> DeadLockCheck(MyProc)
  -> DS_HARD_DEADLOCK
```
`CheckDeadLock()` 收到 `DS_HARD_DEADLOCK` 后调用 `RemoveFromWaitQueue(MyProc, ...)`。
`RemoveFromWaitQueue()` 做四类 cleanup：
- 从 `LOCK.waitProcs` 删除当前 `PGPROC`。
- 回滚 `LOCK.requested[]` 和 `nRequested`。
- 必要时清理 `waitMask`。
- 设置 `waitStatus = PROC_WAIT_STATUS_ERROR`。
然后 `CleanUpLock()` 可能删除无 holder 的 `PROCLOCK` 或 `LOCK`，并唤醒因当前 waiter 消失而可唤醒的后续 waiter。
`ProcSleep()` 循环读到 `PROC_WAIT_STATUS_ERROR` 后返回。
`LockAcquireExtended()` 调用 `DeadLockReport()`。
`DeadLockReport()` 抛：
```text
ERROR: deadlock detected
```
### 9.3 soft deadlock 不 ERROR
`DS_SOFT_DEADLOCK` 表示 detector 找到了可行重排。
这时 shared wait queue 已经被修改。
`ProcSleep()` 继续等待或因为 `ProcLockWakeup()` 已 grant 而退出。
如果开启日志，会看到 avoided deadlock。
这类情况不应该理解成“发生过 deadlock error”。
它是 detector 成功避免 abort 的路径。
### 9.4 no deadlock 不 ERROR
`DS_NO_DEADLOCK` 后继续睡眠。
这可能只是正常长等待。
也可能是等待图中存在不含当前 start point 的环。
后者由那个环上的等待者自己负责。
这不是漏检。
取消当前 start point 不一定能解决不含自己的环。
### 9.5 autovacuum cancellation 不是 deadlock ERROR
`DS_BLOCKED_BY_AUTOVACUUM` 表示当前请求被 autovacuum worker 直接 hard-block。
`ProcSleep()` 会尝试给 autovacuum worker 发 SIGINT。
前提是它不是 wraparound-protection vacuum。
这条路径实现的是 autovacuum 的低锁优先级。
它不是 deadlock detected error。
如果 autovacuum 退出并释放锁，当前请求可以继续成功。
### 9.6 lock timeout 也不是 deadlock ERROR
`lock_timeout` 到期走 SIGINT 和 interrupt processing。
它会通过 `LockErrorCleanup()` 离开 wait queue。
用户看到的是 lock timeout 相关错误。
不要把它和 `deadlock_timeout` 混淆。
一个是取消等待。
一个是触发一次检查。
## 10. 生命周期 / ownership / cleanup
### 10.1 谁创建等待状态
`LockAcquireExtended()` 在 main lock table 中创建或复用 `LOCK` / `PROCLOCK`。
`JoinWaitQueue()` 创建等待状态。
等待状态不是单独对象。
它由这些字段共同表达：
```text
LOCK.requested / waitMask
LOCK.waitProcs
PROCLOCK with possible holdMask == 0
PGPROC.waitLock / waitProcLock / waitLockMode / waitStatus
LOCALLOCK awaited pointer
```
### 10.2 谁持有等待状态
等待状态由当前 backend 发起。
但一旦入队，它是 shared lock manager 状态。
holder release、deadlock detector、lock timeout cleanup、transaction abort 都可能修改它。
这也是为什么等待期间不能只依赖 backend-local 变量。
`PGPROC` 是其它 backend 能安全定位 waiter 的 shared identity。
### 10.3 谁释放等待状态
正常 grant 时，释放等待状态的是 grantor。
`ProcLockWakeup()` 调 `GrantLock()` 后，`ProcWakeup()` 从 wait queue 删除 waiter 并设置 OK。
deadlock hard failure 时，释放等待状态的是当前等待者自己运行的 `CheckDeadLock()`。
它调用 `RemoveFromWaitQueue()`。
cancel、lock timeout、die interrupt 时，释放等待状态的是 `LockErrorCleanup()`。
它用 `awaitedLock` 找回 partition。
如果还在队列中，就调用 `RemoveFromWaitQueue()`。
如果已经被 grantor 移出队列且 `waitStatus == OK`，它调用 `GrantAwaitedLock()` 补本地状态。
### 10.4 ResourceOwner 的边界
ResourceOwner 不拥有 wait queue 节点。
它拥有成功 grant 后的 local lock ownership。
等待期间，`awaitedOwner` 只是为了 grant 成功后能把本地 ownership 记到正确 owner。
如果在等待中 ERROR，shared cleanup 由 `LockErrorCleanup()` 做。
事务 abort 之后，已持有的 locks 再由 `ProcReleaseLocks()` / `LockReleaseAll()` 释放。
不要把这两层混在一起。
### 10.5 detector workspace 生命周期
`InitDeadLockChecking()` 在 backend 启动时把 detector workspace 分配在 `TopMemoryContext`。
这些数组大小按 `MaxBackends` 估算。
包括 `visitedProcs`、`deadlockDetails`、`waitOrders`、`waitOrderProcs`、`curConstraints`、`possibleConstraints`。
运行 detector 时不再临时 palloc 大块 workspace。
原因很直接：
detector 运行时持有所有 lock partition LWLocks。
如果这时因为内存分配失败不能检测死锁，系统反而失去保证。
## 11. 正确性机制层次
| 机制 | 保证 | 不能保证 |
| --- | --- | --- |
| lock partition LWLock | 单个 partition 内 `LOCK` / `PROCLOCK` / wait queue 一致性。 | 长时间等待语义。 |
| all partition LWLocks | detector 看到全局一致等待图。 | 低成本 hot path。 |
| `LOCK.waitProcs` 顺序 | grant 顺序和 soft edge 来源。 | 已持锁冲突的消失。 |
| `PROCLOCK.holdMask` | hard edge 来源。 | backend-local 重入次数。 |
| `PGPROC.waitStatus` | grantor / detector 到 waiter 的结果传递。 | 单独表达完整等待状态。 |
| latch | 唤醒睡眠中的 backend。 | 锁已经可用。 |
| ResourceOwner | 成功持锁后的 ownership cleanup。 | wait queue 自动退出。 |
| timeout framework | 延迟运行 detector 或触发 lock timeout。 | `deadlock_timeout` 后一定失败。 |
### 11.1 delayed check 不会漏死锁
README 给出的关键论证是：
等待图中的环是在最后一条 edge 加入时形成的。
形成这条 edge 的 backend 一定是某个等待者。
它之后会在自己的 `deadlock_timeout` 到期时运行 detector。
所以延迟检查不会漏掉环。
它只推迟发现时间。
这就是 `deadlock_timeout` 的 trade-off。
值太小会增加检测成本和日志噪音。
值太大则延迟真正死锁的 ERROR。
### 11.2 soft edge 为什么是 correctness，不只是公平
如果 `ProcLockWakeup()` 允许 later conflicting waiter 越过 earlier waiter，某些 workload 会饿死强锁请求。
例如一个 `AccessExclusiveLock` 等在队列前面，后面不断进入 `AccessShareLock`。
如果后者都能绕过前者，DDL 可能长期无法执行。
因此前序冲突 waiter 必须被视为 blocker。
deadlock detector 再在必要时有控制地反转这个关系。
这比普通 FIFO 更强，也比无序唤醒更安全。
### 11.3 fast path 为什么不进 detector
relation fast path 只覆盖不会彼此等待的弱 relation locks。
一旦有强锁或等待风险，相关状态会进入 main lock table。
deadlock detector 依赖的是 `LOCK.waitProcs` 和 `PROCLOCK.holdMask`。
fast-path array 中没有 wait queue。
因此 detector 不扫描 fast path 不是遗漏。
它依赖前置 invariant：
```text
可能参与等待图的 heavyweight lock request 必须已经有 main table 表示。
```
### 11.4 lock group
并行查询让一个事务可能对应多个 backend。
`deadlock.c` 用 lock group leader 代表整个 group。
同一 group 内的普通 heavyweight locks 不互相阻塞。
否则 leader 和 workers 很容易自死锁。
这让 `FindLockCycleRecurse()` 和 `TopoSort()` 都必须处理 group members。
本节不展开并行执行，但诊断时要知道：
`leaderPid` 和 lock group 会影响 blocker 解释。
## 12. 错误路径 / 异常路径 / fallback
### 12.1 out of shared memory
如果 `SetupLockInTable()` 不能创建 `LOCK` 或 `PROCLOCK`，`LockAcquireExtended()` 报 out of shared memory。
错误提示通常指向 `max_locks_per_transaction`。
这发生在等待之前。
没有 wait queue，也没有 deadlock detector。
### 12.2 `dontWait`
`dontWait=true` 失败会清理 shared request state 并返回 `LOCKACQUIRE_NOT_AVAIL`。
上层可能把它转换为 SQL 层错误，也可能走条件分支。
它不是 deadlock。
### 12.3 cancel / lock timeout / die interrupt
等待期间可以处理中断。
`LockErrorCleanup()` 是兜底。
它必须处理两个 race：
- 当前 backend 仍在 wait queue。
- grantor 已经把它移出 wait queue 并 grant，但它还没更新本地 `LOCALLOCK`。
第二种情况下必须调用 `GrantAwaitedLock()`。
否则 shared table 已经认为它持锁，本地 ResourceOwner 却不知道，后续 cleanup 会丢失 ownership。
### 12.4 hard deadlock
hard deadlock 路径不直接在 detector 中 ereport。
`CheckDeadLock()` 只把当前请求移出队列并设置 `waitStatus=ERROR`。
`ProcSleep()` 返回后，`LockAcquireExtended()` 调 `DeadLockReport()`。
这样错误报告可以在不持所有 partition LWLocks 的情况下构造。
### 12.5 soft deadlock
soft deadlock 是 fallback 成功。
它把 abort 降级为 queue reorder。
这条路径仍有成本。
detector 拿所有 partitions，搜索图，topological sort，真实重排队列，再运行 wakeup。
但它保留了事务。
### 12.6 Hot Standby recovery conflict
Hot Standby 中普通 backend 的弱锁可能阻塞 Startup process replay `AccessExclusiveLock`。
`ProcSleep()` 在 Hot Standby 路径会调用 recovery conflict 处理，而不是普通 `WaitLatch` detector 流程。
这里通常不是用户事务之间的 deadlock。
它是恢复进度和查询稳定性的冲突。
本节只把它作为异常边界。
### 12.7 autovacuum cancellation
detector 被复用来识别直接阻塞自己的 autovacuum worker。
如果不是防 wraparound vacuum，当前 backend 可以发 SIGINT。
这个行为跨过了 lock manager 与 autovacuum 的模块边界。
它体现的是 operational priority，而不是 waits-for graph 必然有环。
## 13. 成本、资源与跨模块传播
### 13.1 hot path 成本
无等待 acquire 的 hot path 只需要目标 lock partition。
等待路径会更重。
进入 wait queue 要修改 `LOCK`、`PROCLOCK`、`PGPROC`，并可能设置 timeout。
如果等待短于 `deadlock_timeout`，通常不会跑 detector。
这就是延迟检测的成本控制。
### 13.2 detector 成本
`CheckDeadLock()` 要拿所有 lock partitions 的 exclusive LWLock。
这会短暂阻塞其它 lock acquire / release 对 lock table 的访问。
`FindLockCycle()` 要扫描相关 `LOCK.procLocks` 和 `LOCK.waitProcs`。
`TopoSort()` 在涉及的 wait queue 上做候选排序。
workspace 大小随 `MaxBackends` 扩张。
实际搜索成本取决于等待图规模、队列长度和 soft-edge 尝试数量。
这也是为什么 `deadlock_timeout` 不宜过小。
### 13.3 wait queue 长度放大
单个 hot relation 上的强锁等待会让很多 later 请求排队。
`ProcLockWakeup()` release 时要扫描 wait queue。
`pg_blocking_pids()` 也要读取同一对象的 holders 和前序 waiters。
队列越长，诊断和唤醒成本越高。
如果 workload 频繁制造强锁等待，成本不只在 blocker 上。
它会扩散到所有后续 lock requests。
### 13.4 跨模块传播
`postinit.c` 把 `DEADLOCK_TIMEOUT` 注册到 `CheckDeadLockAlert()`，`timeout.c` 负责统一启停 timeout。
`LOCK_TIMEOUT` 的 indicator 必须保留，否则 timeout 触发的 SIGINT 可能被误报为普通 cancel。
`DeadLockReport()` 增加数据库级 deadlock 计数，`pgstat_count_lock_waits()` 更新 lock 类型维度的等待统计。
`log_lock_waits` 也使用 `deadlock_timeout` 作为长等待阈值。
hard deadlock ERROR 通常导致事务 abort，再由 ResourceOwner cleanup 释放已持有 locks 并唤醒其它 wait queue。
`GetBlockerStatusData()` 收集 holder `PROCLOCK` 和前序 waiter pids，并明确忽略 fast-path arrays。
autovacuum cancellation 还会触达 ProcArray 和 signal delivery。
所以 lock wait 的资源传播不是单个 blocker 的局部问题。
## 14. 观测与诊断入口
### 14.1 `pg_locks`
等待中的 heavyweight lock 会显示：
```text
granted = false
waitstart = 等待开始时间或短暂 NULL
mode = 请求 mode
locktype / relation / transactionid / virtualxid = locktag 解码
```
`granted=false` 说明请求已经进入 shared lock manager。
它不说明 blocker 一定是 holder。
前序 waiter 也可能是 soft blocker。
`waitstart` 是原子字段，采样上可能短暂为空。
不要把这个短暂 NULL 当成没有等待。
### 14.2 `pg_stat_activity`
等待时常见：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```
`wait_event='Lock'` 说明当前睡在 heavyweight lock wait。
它不告诉你 hard edge 还是 soft edge。
也不告诉你 detector 是否已经运行。
需要结合 `pg_locks`、`pg_blocking_pids()` 和日志。
### 14.3 `pg_blocking_pids()`
`pg_blocking_pids(pid)` 返回直接阻塞目标 backend 的 pids。
它会包含：
- 已持有冲突锁的 hard blockers。
- 同一 wait queue 中排在前面的冲突 waiters。
这正是 soft edge 对诊断层的体现。
如果看到 blocker 自己也 `granted=false`，不要惊讶。
它可能是前序 waiter。
### 14.4 server log
打开 `log_lock_waits` 并调低 `deadlock_timeout` 后，等待超过阈值可能看到：
```text
process ... still waiting for ... after ...
process ... acquired ... after ...
process ... avoided deadlock ... by rearranging queue order ...
process ... detected deadlock ...
```
日志的 waiters / holders 列表来自 `GetLockHoldersAndWaiters()`。
它是当时采样。
高并发下不要把日志中的 holder 列表当作完整历史因果。
### 14.5 deadlock ERROR detail
`DeadLockReport()` 的 client detail 形态是：
```text
Process A waits for MODE on LOCKTAG; blocked by process B.
Process B waits for MODE on LOCKTAG; blocked by process A.
```
server log 还会附带相关 query strings。
这些 detail 来自 detector 在持锁期间保存的 `DEADLOCK_INFO`。
所以它比事后查询 `pg_locks` 更可靠。
事后 `pg_locks` 可能已经因为 abort cleanup 改变。
### 14.6 `pg_stat_database.deadlocks`
`DeadLockReport()` 调用 `pgstat_report_deadlock()`。
因此真正 hard deadlock ERROR 会增加数据库级 `deadlocks` 计数。
soft deadlock 和 lock timeout 不会增加这个计数。
它只能看到累计 hard deadlock，看不到被 reorder 避免的 soft deadlock。
### 14.7 `pg_stat_lock`
当前本地源码有 `pg_stat_lock`。
`wait_time` 来自 `pgstat_count_lock_waits()`，按 lock tag type 聚合。
它更适合看超过 deadlock timeout 的长等待趋势，不适合还原所有 lock wait。
### 14.8 gdb 断点入口
建议断点集中在等待、检查、唤醒和失败四个边界：
```gdb
break JoinWaitQueue
break ProcSleep
break CheckDeadLock
break ProcLockWakeup
break DeadLockReport
```
等待 backend 上看 `MyProc->waitLock`、`waitProcLock`、`waitLockMode`、`waitStatus` 和 `locallock`。
释放 backend 上看 `lock->waitMask`、`nRequested` 和 `nGranted`。
detector 中看 `nWaitOrders`、`nCurConstraints` 和 `nDeadlockDetails`。
## 15. 课堂实验
### 实验 1：复现 hard deadlock ERROR
准备：
```sql
DROP TABLE IF EXISTS dl_a;
DROP TABLE IF EXISTS dl_b;
CREATE TABLE dl_a(id int primary key);
CREATE TABLE dl_b(id int primary key);
INSERT INTO dl_a VALUES (1);
INSERT INTO dl_b VALUES (1);
```
Session A：
```sql
BEGIN;
UPDATE dl_a SET id = id WHERE id = 1;
```
Session B：
```sql
BEGIN;
UPDATE dl_b SET id = id WHERE id = 1;
```
Session A：
```sql
UPDATE dl_b SET id = id WHERE id = 1;
```
Session B：
```sql
UPDATE dl_a SET id = id WHERE id = 1;
```
预期：
```text
一个 session 在 deadlock_timeout 后报 ERROR: deadlock detected。
另一个 session 因对方 abort 释放锁而继续。
```
观察：
```sql
SELECT pid, locktype, relation::regclass, transactionid, mode, granted, waitstart
FROM pg_locks
WHERE pid IN (...);
```
源码回看：
```text
ProcSleep()
  -> CheckDeadLock()
  -> DeadLockCheck()
  -> DeadLockReport()
```
### 实验 2：观察 soft blocker
准备：
```sql
DROP TABLE IF EXISTS queue_demo;
CREATE TABLE queue_demo(id int);
```
Session A：
```sql
BEGIN;
LOCK TABLE queue_demo IN ACCESS SHARE MODE;
```
Session B：
```sql
BEGIN;
LOCK TABLE queue_demo IN ACCESS EXCLUSIVE MODE;
```
Session C：
```sql
BEGIN;
LOCK TABLE queue_demo IN ACCESS SHARE MODE;
```
Session C 的 `AccessShareLock` 与 Session A 的 `AccessShareLock` 兼容。
但它可能仍然等待。
原因是 Session B 的 `AccessExclusiveLock` 排在 wait queue 前面。
观察：
```sql
SELECT a.pid, a.wait_event_type, a.wait_event, pg_blocking_pids(a.pid) AS blockers
FROM pg_stat_activity a
WHERE a.wait_event_type = 'Lock';
```
再看：
```sql
SELECT pid, mode, granted, waitstart
FROM pg_locks
WHERE relation = 'queue_demo'::regclass
ORDER BY granted DESC, waitstart NULLS FIRST, pid;
```
源码解释：
```text
ProcLockWakeup()
  -> aheadRequests contains AccessExclusiveLock
  -> later AccessShareLock cannot bypass it
```
这个实验证明：
```text
blocker 不一定已经持锁；
前序冲突 waiter 也是 blocker。
```
### 实验 3：观察 `deadlock_timeout` 不是上限
Session A：
```sql
BEGIN;
LOCK TABLE queue_demo IN ACCESS EXCLUSIVE MODE;
```
Session B：
```sql
SET deadlock_timeout = '200ms';
SET log_lock_waits = on;
BEGIN;
LOCK TABLE queue_demo IN ACCESS SHARE MODE;
```
让 Session B 等待数秒。
预期：
```text
Session B 不会在 200ms 后失败；
日志会报告 still waiting；
直到 Session A commit，Session B 才成功。
```
Session A：
```sql
COMMIT;
```
源码解释：
```text
deadlock_timeout
  -> CheckDeadLock()
  -> DS_NO_DEADLOCK
  -> ProcSleep() continues
```
### 实验 4：断点观察 hard edge 与 soft edge
建议用 debug build。
断点：
```gdb
break FindLockCycleRecurseMember
break ProcLockWakeup
```
在实验 1 中观察 hard edge。
关注 `PROCLOCK.holdMask` 与 `conflictMask` 的匹配。
在实验 2 中观察 soft edge。
关注扫描 `LOCK.waitProcs` 时，目标 proc 前面的 waiter request。
需要画出的图：
```text
hard edge:
  waiter -> holder
soft edge:
  later waiter -> earlier conflicting waiter
```
## 16. 常见误区
误区 1：`deadlock_timeout` 是最长锁等待时间。
正确理解：它只是运行 detector 和长等待日志的阈值。
真正取消等待的是 `lock_timeout`、statement timeout、cancel 或 deadlock ERROR。
误区 2：只有已持锁 backend 才能阻塞我。
正确理解：排在前面的冲突 waiter 也能 soft-block 我。
`pg_blocking_pids()` 会返回这类前序 waiter。
误区 3：soft deadlock 是不严重的 deadlock。
正确理解：soft deadlock 是可以通过 wait queue reorder 消除的等待图环。
如果 reorder 无解，它仍会变成 hard deadlock ERROR。
误区 4：detector 每次都扫描所有 backend 的所有状态。
正确理解：它拿所有 lock partitions 得到一致图，但实际 edge 来自相关 `LOCK.procLocks` 和 `LOCK.waitProcs`。
fast-path array 不在图中。
误区 5：waiter 醒来后重新尝试拿锁。
正确理解：`ProcLockWakeup()` 在唤醒前已经 `GrantLock()`。
waiter 只补本地 ownership。
误区 6：`granted=false` 就一定能从 `pg_locks` 直接看出 holder。
正确理解：`pg_locks` 是 lock state snapshot。
直接 blocker 需要结合 `pg_blocking_pids()` 或 server log。
误区 7：deadlock ERROR 的 victims 总是最后加入等待的人。
正确理解：最后加入者一定能检测到环，但实际哪个等待者先 timeout 并报错取决于时序。
误区 8：soft edge reorder 会任意打乱队列。
正确理解：`TopoSort()` 满足 constraints，同时尽量保留原始顺序。
它只在必要范围内改变 wait queue。
## 17. 讨论题
1. 为什么 `ProcLockWakeup()` 必须把前序冲突 waiter 计入 `aheadRequests`？
2. 如果每次入队都立即跑 `DeadLockCheck()`，会改善哪些延迟，又会制造哪些扩展性问题？
3. 为什么 hard edge 不能通过 wait queue reorder 解决？
4. `DeadLockCheck()` 为什么要拿所有 lock partition LWLocks，而不是只拿当前 lock 的 partition？
5. `LockErrorCleanup()` 为什么要处理“已经被 grant 但本地还没记账”的 race？
6. `pg_blocking_pids()` 返回一个 `granted=false` 的 pid 时，你应该如何解释？
7. 为什么 detector 不负责解决不包含当前 start-point 的环？
8. autovacuum cancellation 为什么放在 deadlock detector 附近，而不是单独做一个后台扫描器？
## 18. 本节小结
本节核心链路：
```text
LockAcquireExtended()
  -> JoinWaitQueue()
  -> PGPROC.wait* fields
  -> WaitOnLock()
  -> ProcSleep()
  -> DEADLOCK_TIMEOUT
  -> CheckDeadLock()
  -> DeadLockCheck()
  -> queue reorder or RemoveFromWaitQueue()
  -> ProcLockWakeup() / DeadLockReport()
```
核心状态：
```text
LOCK.waitProcs:
  wait queue 顺序，也是 soft edge 来源。
PROCLOCK.holdMask:
  hard edge 来源。
PGPROC.waitLock / waitProcLock / waitLockMode / waitStatus:
  backend 在等待图中的节点状态。
LOCALLOCK + awaitedLock:
  ERROR cleanup 和本地 ownership 补账入口。
```
核心判断：
```text
hard edge cycle with no workable soft-edge reversal:
  ERROR。
soft edge cycle with workable wait queue reorder:
  reorder，不 ERROR。
no cycle involving current start point:
  继续等待。
direct autovacuum blocker:
  可尝试 cancel autovacuum，不等于 deadlock ERROR。
```
ownership 与 cleanup：
- grantor 在唤醒前完成 shared `GrantLock()`。
- waiter 醒来后只做 `GrantLockLocal()`。
- hard deadlock 用 `RemoveFromWaitQueue()` 移除当前请求。
- cancel / timeout / die 用 `LockErrorCleanup()` 兜底。
- transaction abort 再由 ResourceOwner / `LockReleaseAll()` 释放已经持有的 locks。
正确性规律：
```text
等待图不是单独维护的数据结构；
它从 lock table holder state 和 wait queue ordering 推导出来。
```
成本规律：
```text
延迟 detector 把常见短等待留在 cheap path；
真正长等待才支付全 partition snapshot、图搜索和可能的 topological sort。
```
观测规律：
- `pg_locks.granted=false` 能看到等待请求。
- `pg_blocking_pids()` 能暴露 hard blocker 和 soft blocker。
- server log 和 deadlock ERROR detail 比事后查询更接近 detector 当时看到的图。
- `pg_stat_database.deadlocks` 只累计 hard deadlock ERROR。
- soft deadlock reorder 通常只能从日志、断点或源码 instrumentation 观察。
可迁移 mental model：
```text
当一个系统把公平队列也纳入 correctness，队列顺序本身就会成为 dependency graph 的一部分；
如果某些 dependency 只来自顺序而非已占有资源，detector 可以先尝试重排；
只有无法通过合法重排消除的环，才需要牺牲一个请求。
```
判断边界：
```text
deadlock_timeout 的最佳值依赖 workload、锁等待分布和日志需求；
soft edge 是否出现依赖 wait queue 时序；
observability 是采样，不是完整历史；不同 PostgreSQL 版本的统计视图可能不同，但 hard/soft edge 模型是当前源码的核心不变量。
```
