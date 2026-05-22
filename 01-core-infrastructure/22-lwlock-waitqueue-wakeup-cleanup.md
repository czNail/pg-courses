# PostgreSQL LWLock 等待队列、唤醒协议与 ERROR-safe release

## 课程定位

前置知识：已经理解上一节 `LWLock` 的 atomic `state` 位布局，知道 shared / exclusive holder 如何压缩在一个 `pg_atomic_uint32` 中；已经理解 `PGPROC` 是 backend 在 shared memory 中的进程身份，并且每个 backend 有一个可被其它进程唤醒的 process semaphore。

本节唯一主问题：

```text
LWLockAcquire() 为什么必须“先尝试、入队、再尝试、再睡眠”，PGPROC.lwWaiting、process semaphore、LW_FLAG_WAKE_IN_PROGRESS 和 held_lwlocks 如何共同避免 missed wakeup、重复唤醒和 ERROR 后遗留锁？
```

核心矛盾：LWLock 的 fast path 希望非常轻，拿不到锁时才进入等待队列；但“决定要睡眠”和“真正把自己放进等待队列”不是原子动作。如果 holder 在这段窗口里释放锁并检查等待队列，waiter 可能永远睡过去。PostgreSQL 的解法不是让 release 直接授予锁，而是建立一个可重试的等待协议：

```text
先用 atomic state 尝试获取；
失败后，在 wait-list lock 保护下把 MyProc 挂进 lock->waiters；
入队后立刻再次尝试获取；
如果第二次成功，就把自己从队列撤掉；
如果仍失败，才睡在 MyProc->sem 上；
被唤醒后只表示“可以重试”，不表示已经持有锁；
真正成功持锁后记录到 held_lwlocks，ERROR 时由 LWLockReleaseAll() 兜底释放。
```

学完后应能独立判断：

```text
为什么 LWLockAcquire() 不能“失败后直接 sleep”；
为什么 wakeup 只把 waiter 从队列移出并 post semaphore，而不把锁所有权转交给它；
为什么被唤醒后还要回到 LWLockAttemptLock()；
为什么 lwWaiting 有 NOT_WAITING / WAITING / PENDING_WAKEUP 三态；
为什么 LW_FLAG_WAKE_IN_PROGRESS 能减少重复唤醒，却不是一个“锁已经交给某人”的标志；
为什么持有 LWLock 时要 HOLD_INTERRUPTS()，并把成功 acquire 的锁记录到 held_lwlocks；
为什么 AbortTransaction() 要尽早调用 LWLockReleaseAll()；
为什么 LWLock wait event 表示“等待过 LWLock”，不保证下一刻已经拿到锁。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节回答了第一个问题：

```text
一个 LWLock 如何用 atomic state 表达 shared holder count、exclusive sentinel 和 wait flags？
```

这解释了无冲突 acquire 为什么可以很快：

```text
LWLockAttemptLock()
  -> 读 lock->state
  -> 构造 desired_state
  -> CAS 成功则成为 holder
  -> CAS 看到冲突则返回 mustwait=true
```

但 `mustwait=true` 之后还有一个更危险的问题：

```text
看到锁被占用，不等于之后一定会有人唤醒我。
```

考虑一个错误实现：

```text
waiter:
  if (LWLockAttemptLock(lock) fails)
      PGSemaphoreLock(MyProc->sem);

holder:
  LWLockRelease(lock);
  if (waiters list is non-empty)
      wake waiters;
```

如果时间线变成：

```text
T1 waiter 看到锁被 holder 持有；
T2 holder 释放锁，检查 waiters list，发现没人；
T3 waiter 才开始 PGSemaphoreLock()；
```

waiter 会错过唯一一次唤醒。这就是 `lwlock.c` 文件头注释说的 race：

```text
we're now stuck waiting inside the OS
```

所以本节接着回答 LWLock 的第二层问题：

```text
拿不到锁后，如何把“我需要被唤醒”这件事可靠地发布给 holder？
```

本节只讲 `LWLockAcquire()` 的等待、唤醒和 cleanup。`LWLockWaitForVar()`、`LWLockUpdateVar()`、WAL insert 等“等待变量变化”的特殊路径会被作为旁路理解，不作为主线展开。Latch 和 Condition Variable 会在后续课程单独讲。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
LWLockAcquire() 不把 wakeup 当作“授予锁”，而把 wakeup 当作“允许重试”；
为了避免 missed wakeup，waiter 必须先进入 lock->waiters，再第二次检查 state；
为了避免重复唤醒，release 通过 LW_FLAG_WAKE_IN_PROGRESS 表达“已有一批 waiter 被移出队列等待运行”；
为了避免 ERROR 泄漏锁，成功 acquire 后必须记录到 backend-local held_lwlocks，abort 路径统一 release。
```

这个模型里有三个不变量：

```text
不变量 1:
  一个真正准备睡眠的 backend，必须已经在 lock->waiters 中，
  或者已经被 release 路径移出队列并即将收到 semaphore post。

不变量 2:
  被唤醒的 backend 还没有获得锁。
  它只能重新执行 LWLockAttemptLock()，和新来的 fast-path backend 竞争。

不变量 3:
  一个 backend 成功持有 LWLock 后，必须能在 ERROR cleanup 中找到并释放它。
```

这三个不变量分别对应本节的三个关键状态：

| 状态 | 所在位置 | 解决的问题 |
| --- | --- | --- |
| `lock->waiters` | shared memory 中的 `LWLock` | release 能找到需要唤醒的 backend。 |
| `PGPROC.lwWaiting` | shared memory 中的 per-backend `PGPROC` | waiter 与 waker 之间确认自己是否仍在队列中、是否正在被唤醒。 |
| `held_lwlocks[]` | backend-local 静态数组 | ERROR 后能释放已经持有的 LWLock。 |

注意这三类状态跨越了两个世界：

```text
shared state:
  lock->state
  lock->waiters
  PGPROC.lwWaiting
  PGPROC.lwWaitMode
  PGPROC.lwWaitLink
  PGPROC.sem

backend-local state:
  num_held_lwlocks
  held_lwlocks[]
```

这正是 LWLock 难读的原因：它不是单纯的锁算法，而是一个跨进程等待协议加错误恢复协议。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/lwlock.c` | `LWLockAcquire()` 的四阶段等待协议、`LWLockWakeup()`、`LWLockRelease()`、`LWLockReleaseAll()`。 |
| 2 | `src/include/storage/lwlock.h` | `LWLockWaitState` 三态、`LWLock` waiters 链表。 |
| 3 | `src/include/storage/proc.h` | `PGPROC.sem`、`lwWaiting`、`lwWaitMode`、`lwWaitLink`。 |
| 4 | `src/backend/storage/lmgr/proc.c` | backend 获取或复用 `PGPROC` 时如何初始化 `lwWaiting` 与 reset semaphore。 |
| 5 | `src/include/storage/pg_sema.h` | PostgreSQL 对 process semaphore 的抽象语义：计数、lock、unlock、trylock。 |
| 6 | `src/backend/access/transam/xact.c` | `AbortTransaction()` 为什么尽早 `LWLockReleaseAll()`。 |
| 7 | `doc/src/sgml/monitoring.sgml` | LWLock wait probe 与 wait event 的观测边界。 |

推荐阅读顺序：

```text
先读 lwlock.c 顶部四阶段注释
  -> 精读 LWLockAcquire()
  -> 回头看 LWLockQueueSelf() / LWLockDequeueSelf()
  -> 再读 LWLockRelease() / LWLockWakeup()
  -> 最后读 held_lwlocks 与 LWLockReleaseAll()
```

不要先从 `PGSemaphoreLock()` 的平台实现开始读。semaphore 在本节只是“睡眠和唤醒”的承载物，正确性来自 LWLock 自己维护的 wait-list 与 `lwWaiting` 状态。

## 4. 关键数据结构与状态

### `LWLockWaitState`: waiter 与 waker 的握手状态

`src/include/storage/lwlock.h` 中定义了三个状态：

```text
LW_WS_NOT_WAITING:
  backend 当前不在 LWLock 等待协议里，或者已经被唤醒完成。

LW_WS_WAITING:
  backend 在某个 LWLock 的 waiters 链表中。

LW_WS_PENDING_WAKEUP:
  backend 已经被 waker 从 waiters 链表移出，
  但 semaphore post 还没完成。
```

`PENDING_WAKEUP` 很容易被忽略，但它是避免链表损坏的关键状态。

想象一个 waker 正在做：

```text
proclist_delete(&lock->waiters, waiter)
  -> waiter->lwWaiting = PENDING_WAKEUP
  -> 稍后 waiter->lwWaiting = NOT_WAITING
  -> PGSemaphoreUnlock(waiter->sem)
```

如果 waiter 同时因为二次尝试成功而调用 `LWLockDequeueSelf()`，它需要知道：

```text
我还在 waiters 链表中吗？
还是已经被别人移出了，正在等一次安排好的 wakeup？
```

`lwWaiting` 就是这个判断的共享状态。

### `PGPROC`: 每个 backend 的等待槽

`src/include/storage/proc.h` 中与本节相关的字段：

```text
PGSemaphore sem:
  backend 私有的 process semaphore，其地址保存在 shared PGPROC 中，
  其它 backend 可以通过 PGSemaphoreUnlock() 唤醒它。

uint8 lwWaiting:
  当前 LWLock 等待状态。

uint8 lwWaitMode:
  等待的模式，可能是 LW_SHARED、LW_EXCLUSIVE、LW_WAIT_UNTIL_FREE。

proclist_node lwWaitLink:
  把这个 PGPROC 挂入某个 LWLock waiters 链表的节点。
```

这些字段在 `proc.c` 中初始化：

```text
MyProc->lwWaiting = LW_WS_NOT_WAITING;
MyProc->lwWaitMode = 0;
PGSemaphoreReset(MyProc->sem);
```

这里有两个边界：

```text
一个 backend 同一时刻只能等待一个 LWLock 或 buffer content lock；
准备等待前 semaphore 必须处于干净状态，否则旧 wakeup 计数会污染新的等待。
```

### `lock->waiters`: 不是 owner 队列，而是 retry 队列

`LWLock` 的 `waiters` 链表保存的是 `PGPROC` 编号，不是普通 C 指针。跨进程 shared memory 中不能随意存放 backend-local 地址，PostgreSQL 用 `ProcNumber` 和 `GetPGProcByNumber()` 找回 shared `PGPROC`。

这个队列的语义是：

```text
这里的人需要在锁释放后被叫醒重试。
```

不是：

```text
队头的人已经被承诺下一次获得锁。
```

这个区别非常重要。`LWLockWakeup()` 会选择一批 waiter 从队列中移出，并 post 它们的 semaphore；它并不修改 low bits holder count，不会把 shared count 或 exclusive sentinel 直接转交给 waiter。

### `LW_FLAG_HAS_WAITERS`: release 是否需要看队列

入队时 `LWLockQueueSelf()` 会设置：

```text
pg_atomic_fetch_or_u32(&lock->state, LW_FLAG_HAS_WAITERS);
```

release 端看到：

```text
oldstate has LW_FLAG_HAS_WAITERS
  and not LW_FLAG_WAKE_IN_PROGRESS
  and no holder remains
```

才会调用 `LWLockWakeup()`。

`HAS_WAITERS` 是一个优化和协议提示：

```text
没有这个 bit，release 不必抢 wait-list lock；
有这个 bit，表示 waiters 链表中可能有人，需要在合适时机检查。
```

它不是“队列一定非空”的绝对真理，因为 dequeue self 和 wakeup 可能正在并发更新它。因此代码在持有 wait-list lock 后会重新看 `proclist_is_empty()`。

### `LW_FLAG_WAKE_IN_PROGRESS`: 抑制重复唤醒

`LW_FLAG_WAKE_IN_PROGRESS` 表示：

```text
已经有一批 waiter 被移出 waiters 队列，正在等待调度运行；
在它们重新进入 LWLockAttemptLock() 之前，release 不要不断唤醒更多 waiter。
```

为什么需要它？

```text
LWLockRelease() 释放 holder count 后，锁可能马上又被其它 backend 抢走；
如果每次释放都唤醒一批 waiter，而上一批 waiter 还没运行，
系统可能制造大量无效 wakeup 和上下文切换。
```

`WAKE_IN_PROGRESS` 不是所有权标志。它不保证被唤醒者会拿到锁，只是在 release 侧表达：

```text
先等已经叫醒的人跑一下。
```

被唤醒的 waiter 在准备重试时会清掉它：

```text
pg_atomic_fetch_and_u32(&lock->state, ~LW_FLAG_WAKE_IN_PROGRESS);
```

这让下一轮 release 可以继续唤醒等待者。

### `held_lwlocks[]`: ERROR cleanup 的 backend-local ledger

`lwlock.c` 中维护：

```text
#define MAX_SIMUL_LWLOCKS 200

static int num_held_lwlocks = 0;
static LWLockHandle held_lwlocks[MAX_SIMUL_LWLOCKS];
```

每次 `LWLockAcquire()` 成功后都会追加：

```text
held_lwlocks[num_held_lwlocks].lock = lock;
held_lwlocks[num_held_lwlocks++].mode = mode;
```

每次 `LWLockRelease()` 会反向查找并移除对应记录。

这个数组不在 shared memory 中。它回答的问题是：

```text
当前 backend 自己持有哪些 LWLock？
如果 ereport(ERROR) 长跳走了，应该释放哪些？
```

它不回答：

```text
谁持有某个 LWLock？
```

生产环境里通常不能直接从 `held_lwlocks[]` 观测全局 owner；这也是诊断 LWLock 等待时经常只能看到 tranche 名，而不能直接看到 holder 的原因之一。

## 5. 主流程源码 walkthrough

本节用一个 exclusive waiter 等待某个被占用的 LWLock 作为主线。shared waiter 逻辑相同，只是 `LWLockAttemptLock()` 的冲突条件不同。

### 阶段 0: 调用者进入 `LWLockAcquire()`

入口：

```text
LWLockAcquire(lock, mode)
```

函数先做几件事：

```text
检查 mode 必须是 LW_SHARED 或 LW_EXCLUSIVE；
检查 held_lwlocks[] 是否还有空间；
HOLD_INTERRUPTS()；
进入 acquire loop。
```

`HOLD_INTERRUPTS()` 的意义不是“禁止所有信号到达”，而是推迟 cancel / die interrupt 的处理，让 backend 不会在持有 LWLock 的 shared memory 临界区中间被 ERROR 跳走。

注释说得很直接：

```text
cancel/die interrupts are held off until lock release
```

这和 `held_lwlocks[]` 是一组机制：

```text
正常路径:
  acquire 成功 -> 记录 held_lwlocks[] -> release 移除记录 -> RESUME_INTERRUPTS()

异常路径:
  ERROR cleanup -> LWLockReleaseAll() 根据 held_lwlocks[] 释放
```

### 阶段 1: 第一次尝试

主循环第一步：

```text
mustwait = LWLockAttemptLock(lock, mode);

if (!mustwait)
    break;
```

如果锁刚好空闲，fast path 成功，直接跳出循环。此时还没有碰 waiters 链表。

如果失败，此时 backend 只知道：

```text
我刚刚观察到冲突。
```

它还不能睡，因为 holder 并不知道它需要被唤醒。

### 阶段 2: 入队发布“我需要被唤醒”

失败后调用：

```text
LWLockQueueSelf(lock, mode);
```

`LWLockQueueSelf()` 做的状态变化：

```text
检查 MyProc 存在；
检查 MyProc->lwWaiting == LW_WS_NOT_WAITING；
获取 LWLock wait-list lock；
设置 lock->state 的 LW_FLAG_HAS_WAITERS；
MyProc->lwWaiting = LW_WS_WAITING；
MyProc->lwWaitMode = mode；
把 MyProcNumber 挂到 lock->waiters 队尾；
释放 wait-list lock。
```

这一步的本质是把 backend-local 的意图发布为 shared memory 中的事实：

```text
release 端现在能看见我了。
```

这里用的是 LWLock 自己的 wait-list lock，也就是 `LW_FLAG_LOCKED`。它只保护 `lock->waiters` 链表操作，不阻止别的 backend 在无冲突情况下通过 atomic state 获取该 LWLock。

### 阶段 3: 入队后第二次尝试

入队完成后，`LWLockAcquire()` 立刻再次调用：

```text
mustwait = LWLockAttemptLock(lock, mode);
```

这一步就是避免 missed wakeup 的核心。

如果第二次成功，说明在阶段 1 和阶段 2 之间，原 holder 可能已经释放了锁，或者竞争状态已经变化。此时当前 backend 已经拿到锁，但它还在 waiters 队列里，所以必须：

```text
LWLockDequeueSelf(lock);
break;
```

如果不 dequeue，会留下一个已经持锁的 backend 作为假 waiter，后续 release 会尝试唤醒它，甚至污染队列状态。

如果第二次仍失败，才可以睡眠。因为这时已经满足：

```text
我在 waiters 队列里；
我入队后又确认锁仍冲突；
当前或后续 holder release 时一定能看到 waiters 标记或队列状态。
```

这就是文件头注释里的四阶段协议：

```text
Phase 1: Try atomically.
Phase 2: Add ourselves to waitqueue.
Phase 3: Try again, and remove ourselves if successful.
Phase 4: Sleep until wake-up, then go back to Phase 1.
```

### 阶段 4: 睡在 process semaphore 上

真正睡眠发生在：

```text
for (;;)
{
    PGSemaphoreLock(proc->sem);
    if (proc->lwWaiting == LW_WS_NOT_WAITING)
        break;
    extraWaits++;
}
```

这里有一个细节：

```text
semaphore 被 post 了，不一定表示这次 post 来自 LWLockRelease()。
```

进程 semaphore 是计数语义，可能有额外 wakeup。PostgreSQL 用 `lwWaiting` 作为二次确认：

```text
只有当 lwWaiting 变成 NOT_WAITING，才认为 LWLock 等待完成；
否则把这次 semaphore 返回计入 extraWaits，继续等。
```

醒来后做两件事：

```text
清掉 LW_FLAG_WAKE_IN_PROGRESS；
结束 wait event 统计与 trace；
回到外层循环，再次 LWLockAttemptLock()。
```

注意：

```text
wait-done 不等于 acquire-done。
```

`doc/src/sgml/monitoring.sgml` 对 DTrace probe 的描述也强调了这一点：`lwlock-wait-done` 表示 server process 已经从等待中释放，但并不表示它已经持有锁。

### 阶段 5: 成功后记录 holder

跳出循环后：

```text
held_lwlocks[num_held_lwlocks].lock = lock;
held_lwlocks[num_held_lwlocks++].mode = mode;
```

从这一刻开始，如果后续代码抛 ERROR，`AbortTransaction()` 或其它异常清理路径就能通过 `LWLockReleaseAll()` 找到它。

这个顺序很重要：

```text
在成功 acquire 之前不能记录；
成功 acquire 之后必须尽快记录。
```

否则要么 cleanup 错放未持有的锁，要么 ERROR 时遗留真实持有的锁。

## 6. release 与 wakeup 主流程

### `LWLockRelease()`: 先释放 holder，再决定是否唤醒

`LWLockRelease(lock)` 先在 `held_lwlocks[]` 中反向查找对应锁：

```text
for (i = num_held_lwlocks; --i >= 0;)
    if (lock == held_lwlocks[i].lock)
        break;
```

找不到会：

```text
elog(ERROR, "lock %s is not held", T_NAME(lock));
```

找到后移除本地记录，再根据 mode 更新 atomic state：

```text
exclusive:
  pg_atomic_sub_fetch_u32(&lock->state, LW_VAL_EXCLUSIVE)

shared:
  pg_atomic_sub_fetch_u32(&lock->state, LW_VAL_SHARED)
```

release 的关键判断：

```text
if ((oldstate & LW_FLAG_HAS_WAITERS) &&
    !(oldstate & LW_FLAG_WAKE_IN_PROGRESS) &&
    (oldstate & LW_LOCK_MASK) == 0)
    LWLockWakeup(lock);
```

含义是：

```text
可能有人等；
没有一批 wakeup 正在路上；
当前 release 后已经没有 holder。
```

只有这三个条件同时成立，才值得进入 `LWLockWakeup()`。

### 为什么 release 不直接授予锁

代码注释解释了一个很实际的性能选择：

```text
如果 release 直接把锁交给 waiter，
两个或多个进程竞争同一个短临界区时，每次 acquire 都可能强制进程切换。
```

LWLock 保护的通常是短 shared memory 临界区。让当前 CPU 上正在运行的 backend 有机会继续 acquire / release 多次，往往比严格队列转交更高效。

所以 PostgreSQL 选择：

```text
release 唤醒 waiter；
waiter 被调度后重新竞争；
新来的 backend 也可能先拿到锁。
```

这不是严格公平锁。它换取的是短临界区高吞吐。

### `LWLockWakeup()`: 选择一批有机会成功的 waiter

`LWLockWakeup()` 的主要步骤：

```text
初始化临时 wakeup 链表；
获取 wait-list lock；
从 lock->waiters 遍历 waiter；
把可唤醒的 waiter 从 lock->waiters 移到本地 wakeup 链表；
设置 waiter->lwWaiting = LW_WS_PENDING_WAKEUP；
计算是否需要设置 LW_FLAG_WAKE_IN_PROGRESS；
释放 wait-list lock；
逐个设置 waiter->lwWaiting = LW_WS_NOT_WAITING 并 PGSemaphoreUnlock()。
```

它对 shared / exclusive waiter 的选择规则：

```text
如果先遇到 shared waiter，可以继续唤醒后续 shared waiter；
一旦遇到 exclusive waiter，唤醒它后停止；
如果已经唤醒过某个真正要 acquire 的 waiter，后续 exclusive waiter 不再唤醒；
LW_WAIT_UNTIL_FREE waiter 只等待锁释放，不自动重试 acquire。
```

这对应 reader-writer lock 的自然语义：

```text
多个 shared waiter 可能同时成功；
exclusive waiter 只能独占，所以叫醒一个就够。
```

### 为什么先移到本地 wakeup 链表，再 post semaphore

`LWLockWakeup()` 不是在持有 wait-list lock 时直接 `PGSemaphoreUnlock()`。它先把 waiter 从共享队列移到本地 `wakeup` 链表，更新 flags，释放 wait-list lock，然后再 post。

原因是：

```text
wait-list lock 只应保护很短的链表和 flag 操作；
唤醒进程可能触发调度，不应扩大持有 wait-list lock 的窗口；
被唤醒者可能立刻参与新的 LWLock 等待协议，必须先保证旧链表状态已经干净。
```

在真正 post 前有一个 memory barrier：

```text
pg_write_barrier();
waiter->lwWaiting = LW_WS_NOT_WAITING;
PGSemaphoreUnlock(waiter->sem);
```

注释解释了它要保证：

```text
把 waiter 从链表移除这件事，必须先于 NOT_WAITING 对目标 backend 可见；
否则目标 backend 可能因为其它原因醒来并开始等待新锁，
旧 lwWaitLink 还没从旧队列完全摘干净，就被复用到新队列，链表会损坏。
```

这是本节非常重要的系统规律：

```text
跨进程等待协议里，“唤醒”通常不是最后一个共享状态变化；
必须先发布队列状态已经一致，再允许对方继续运行。
```

## 7. `LWLockDequeueSelf()` 的两个分支

`LWLockDequeueSelf()` 发生在两类场景：

```text
入队后第二次尝试成功；
或者 wait-for-var 等旁路发现条件已经不需要等待。
```

它先获取 wait-list lock，然后看：

```text
on_waitlist = MyProc->lwWaiting == LW_WS_WAITING;
```

### 分支 A: 我还在队列中

如果 `on_waitlist=true`：

```text
proclist_delete(&lock->waiters, MyProcNumber, lwWaitLink);
如果队列空了，清 LW_FLAG_HAS_WAITERS；
释放 wait-list lock；
MyProc->lwWaiting = LW_WS_NOT_WAITING；
```

这是最直观的撤销入队。

### 分支 B: 我已经被别人移出队列

如果 `on_waitlist=false`，说明另一个 backend 的 `LWLockWakeup()` 已经把当前 backend 从 waiters 队列移出，并设置了 `PENDING_WAKEUP`，但当前 backend 同时走到了 dequeue path。

此时不能简单地把 `lwWaiting` 改成 `NOT_WAITING` 后返回。因为 waker 仍然计划 post 一次 semaphore，如果不吸收这次唤醒，后续新的等待可能被旧 semaphore 计数污染。

所以代码做：

```text
清 LW_FLAG_WAKE_IN_PROGRESS；
循环 PGSemaphoreLock(MyProc->sem)，直到 lwWaiting == NOT_WAITING；
记录 extraWaits；
把额外吸收的 wakeup 再 PGSemaphoreUnlock() 补回。
```

这个分支说明：

```text
waiter 和 waker 的竞态不是 bug，而是协议的一部分。
```

`LW_WS_PENDING_WAKEUP` 存在的价值就在这里。

## 8. 生命周期 / ownership / cleanup

### 正常生命周期

一个普通 `LWLockAcquire()` / `LWLockRelease()` 生命周期：

```text
caller
  -> LWLockAcquire(lock, mode)
     -> HOLD_INTERRUPTS()
     -> 尝试 acquire
     -> 必要时入队、睡眠、被唤醒、重试
     -> 成功后记录 held_lwlocks[]
  -> 访问受保护 shared memory
  -> LWLockRelease(lock)
     -> 从 held_lwlocks[] 移除
     -> atomic 减 holder count
     -> 必要时 LWLockWakeup()
     -> RESUME_INTERRUPTS()
```

owner 不是写在 `LWLock` 结构里的稳定字段。正常构建下，`LWLock` 不记录 holder identity。`LOCK_DEBUG` 下有 `owner`，但它是 debug 支持，不是生产语义边界。

### ERROR cleanup 生命周期

事务 abort 入口 `AbortTransaction()` 中很早就做：

```text
HOLD_INTERRUPTS();
AtAbort_Memory();
AtAbort_ResourceOwner();
LWLockReleaseAll();
```

注释说明：

```text
Release any LW locks we might be holding as quickly as possible.
Regular locks, however, must be held till we finish aborting.
Releasing LW locks is critical since we might try to grab them again while cleaning up.
```

也就是说：

```text
heavyweight locks:
  是事务语义的一部分，abort 期间还需要维持一段时间。

LWLocks:
  只是 shared memory 内部结构保护。
  ERROR 后继续持有会阻塞其它 backend，甚至让 cleanup 自己再次 acquire 时死锁。
```

`LWLockReleaseAll()` 的实现：

```text
while (num_held_lwlocks > 0)
{
    HOLD_INTERRUPTS();
    LWLockRelease(held_lwlocks[num_held_lwlocks - 1].lock);
}
```

这里额外 `HOLD_INTERRUPTS()` 是为了匹配 `LWLockRelease()` 内部的 `RESUME_INTERRUPTS()`，避免 error recovery 已经设置好的 interrupt holdoff 被逐个 release 错误地降到负值。

### 为什么 `held_lwlocks[]` 不走 ResourceOwner

从源码形态看，LWLock 使用的是 `lwlock.c` 内的静态数组，而不是常规 `ResourceOwner` 资源表。原因可以从职责推断：

```text
LWLock 是非常底层、非常热的同步原语；
每次 acquire/release 都进入通用资源所有者机制会增加 hot path 成本；
ERROR cleanup 只需要知道当前 backend 持有哪些 LWLock，并且数量通常很少。
```

这也带来一个边界：

```text
MAX_SIMUL_LWLOCKS 是硬限制；
超过会 elog(ERROR, "too many LWLocks taken")。
```

如果某段代码需要同时持有大量 LWLock，本身就应该重新审视设计。

## 9. 正确性机制层次

### 第一层: atomic state 解决“是否能成为 holder”

上一节讲过：

```text
LWLockAttemptLock()
  -> shared: 没有 exclusive sentinel 即可加 shared count
  -> exclusive: 没有任何 holder 即可写 exclusive sentinel
```

这层只回答：

```text
现在能不能拿锁？
```

它不解决：

```text
现在不能拿锁时，未来谁来叫醒我？
```

### 第二层: waiters 链表解决“release 能找到谁”

`LWLockQueueSelf()` 在 wait-list lock 下挂入 `lock->waiters`，并设置 `HAS_WAITERS`。

这层回答：

```text
如果锁释放了，release 端应该去哪里找等待者？
```

### 第三层: 二次尝试解决 missed wakeup

入队后第二次 `LWLockAttemptLock()` 回答：

```text
我入队期间锁是否已经变得可用？
```

如果可用，立即拿锁并撤销排队。

如果不可用，则睡眠是安全的，因为 holder release 时不会再看不到我。

### 第四层: `lwWaiting` 解决 waiter/waker 竞态

`lwWaiting` 不只是“是否等待”的布尔值。三态表达了：

```text
NOT_WAITING:
  不在旧等待协议里。

WAITING:
  还在 waiters 链表里。

PENDING_WAKEUP:
  已离开 waiters 链表，但 semaphore post 还没完成。
```

它让 `LWLockDequeueSelf()`、`LWLockWakeup()` 和 `PGSemaphoreLock()` 的返回之间可以安全握手。

### 第五层: semaphore 只负责阻塞和计数

`PGSemaphore` 的接口只有：

```text
PGSemaphoreLock()
PGSemaphoreUnlock()
PGSemaphoreTryLock()
```

它不懂 LWLock，也不懂 shared / exclusive。LWLock 代码必须在 semaphore 之上维护：

```text
我是否真的被本次 LWLock release 唤醒；
额外 wakeup 是否需要吸收或补回；
被唤醒后是否还要重试。
```

所以不要把 `PGSemaphoreUnlock(waiter->sem)` 理解为：

```text
把 LWLock 交给 waiter。
```

它只是：

```text
让 waiter 从 OS wait 中回来，继续执行协议。
```

### 第六层: interrupt holdoff 与 `held_lwlocks[]` 解决 ERROR 后遗留锁

最后一层是异常控制流。

PostgreSQL 的 ERROR 不是普通返回值，而可能通过长跳转离开当前调用栈。如果 shared memory 临界区中间发生 ERROR，普通 C 栈上的 `finally` 风格 cleanup 不存在。

因此 LWLock 必须：

```text
持锁期间推迟 cancel/die interrupt；
成功持锁后登记到 backend-local held_lwlocks[]；
abort 时尽早 LWLockReleaseAll()。
```

这层解决：

```text
错误路径不应该永久占用 shared memory 同步原语。
```

## 10. 错误路径 / 异常路径 / fallback

### 没有 `PGPROC` 却要等待

`LWLockQueueSelf()` 中：

```text
if (MyProc == NULL)
    elog(PANIC, "cannot wait without a PGPROC structure");
```

没有 `PGPROC` 就没有 shared wait slot，也没有 process semaphore。此时不能进入等待协议。

`LWLockAcquire()` 也有：

```text
Assert(!(proc == NULL && IsUnderPostmaster));
```

这说明 shared memory 初始化早期可以有特殊路径，但正常 backend 不能在没有 `PGPROC` 的情况下等待。

### 正在等一个 LWLock，又试图排队等另一个

`LWLockQueueSelf()` 中：

```text
if (MyProc->lwWaiting != LW_WS_NOT_WAITING)
    elog(PANIC, "queueing for lock while waiting on another one");
```

一个 `PGPROC` 只有一个 `lwWaitLink` 和一个 `lwWaiting` 槽。嵌套等待两个 LWLock 会让 wait-list 链表节点无从区分。

这不是普通 ERROR，而是 PANIC 级别的“不变量破坏”。

### 持有锁过多

`LWLockAcquire()` 与旁路 acquire 都会检查：

```text
if (num_held_lwlocks >= MAX_SIMUL_LWLOCKS)
    elog(ERROR, "too many LWLocks taken");
```

这个 ERROR 发生在真正 acquire 之前，所以不会新增一个未登记的 holder。

这也提醒代码设计：

```text
LWLock 不适合用作大量对象级长期持有的锁集合。
```

### release 未持有的锁

`LWLockRelease()` 找不到对应记录时：

```text
elog(ERROR, "lock %s is not held", T_NAME(lock));
```

这是 backend-local ledger 发现调用者破坏 acquire/release 配对。

### semaphore 虚假或额外唤醒

等待循环不信任一次 `PGSemaphoreLock()` 返回：

```text
if (proc->lwWaiting == LW_WS_NOT_WAITING)
    break;
extraWaits++;
```

最后：

```text
while (extraWaits-- > 0)
    PGSemaphoreUnlock(proc->sem);
```

这保持 process semaphore 的计数语义不被 LWLock 等待循环吃掉。

### `LWLockAcquireOrWait()` 与 `LW_WAIT_UNTIL_FREE`

`LWLockAcquireOrWait()` 使用几乎相同的双尝试协议，但语义不同：

```text
如果锁立即可用，就 acquire 并返回 true；
如果锁不可用，就等待它释放，但不 acquire，返回 false。
```

典型使用是 `WALWriteLock`：某个 backend 持锁 flush WAL 时，可能顺便把其它 backend 的 commit record 也 flush 了。其它 backend 只需要等 flush 完成，然后重新检查自己的 LSN 是否已经 durable，不一定还要拿锁。

`LW_WAIT_UNTIL_FREE` waiter 会被放到队头：

```text
proclist_push_head(&lock->waiters, MyProcNumber, lwWaitLink);
```

这是一条旁路，但仍然复用：

```text
入队
  -> 二次检查
  -> semaphore sleep
  -> lwWaiting 握手
```

### `LWLockWaitForVar()` 与变量变化

`LWLockWaitForVar()` 也复用同类协议，但等待条件变成：

```text
锁释放，或者 valptr 的值不再等于 oldval。
```

它说明 LWLock 等待队列还能支撑“锁持有者更新某个变量后唤醒等待者”的场景。但这不是本节主线，记住边界即可：

```text
等待谓词或变量变化时，仍然要防 missed wakeup；
只是二次检查的对象从 lock state 扩展到了 variable value。
```

## 11. 成本、资源与跨模块传播

### fast path 成本

无冲突 acquire/release 的主要成本：

```text
atomic CAS 或 atomic subtract；
backend-local held_lwlocks[] 追加 / 删除；
HOLD_INTERRUPTS / RESUME_INTERRUPTS 计数更新；
trace hook 条件判断。
```

不会进入 wait-list lock，也不会系统调用 semaphore。

### contention path 成本

冲突路径会增加：

```text
wait-list lock 的 atomic fetch_or 与短 spin；
proclist 链表插入 / 删除；
pg_stat wait event 上报；
DTrace probe；
PGSemaphoreLock() 进入 OS wait；
PGSemaphoreUnlock() 唤醒其它进程；
被唤醒后再次 CAS 竞争。
```

这意味着：

```text
LWLock 本身支持睡眠，但不意味着可以把长耗时操作放在 LWLock 临界区。
```

临界区越长，waiter 越容易进入 semaphore path，系统成本从 CPU cache line 争用扩展到进程调度。

### `WAKE_IN_PROGRESS` 的成本控制

没有 `WAKE_IN_PROGRESS` 时，可能出现：

```text
holder A release，唤醒 waiter W1；
W1 尚未被调度；
新 backend B 获取并释放锁，又唤醒 W2；
新 backend C 获取并释放锁，又唤醒 W3；
```

一批 waiter 都醒来后可能继续失败，制造调度风暴。

`WAKE_IN_PROGRESS` 把 release 端行为改成：

```text
已经有人被叫醒了，先让它们跑；
等它们醒来并清掉 flag 后，再考虑下一批。
```

这是 throughput 优先的设计，不是严格 FIFO 公平。

### 与 buffer manager 的关系

Buffer mapping lock 是典型 LWLock 使用者：

```text
BufTableLookup():
  caller 至少持有 shared BufferMapping partition lock。

BufTableInsert() / BufTableDelete():
  caller 持有 exclusive BufferMapping partition lock。
```

如果 mapping lock 等待严重，`pg_stat_activity` 可能看到：

```sql
wait_event_type = 'LWLock'
wait_event = 'BufferMapping'
```

这表示某个 buffer mapping partition lock 出现竞争。它不告诉你：

```text
具体是哪一个 BufferTag；
holder 是哪个 backend；
waiter 醒来后是否已经拿到锁。
```

### 与 ProcArray 的关系

`ProcArrayLock` 保护全局事务可见性相关状态。snapshot 获取、事务结束、xmin horizon 计算都会触碰它。

当很多 backend 同时读写 ProcArray 时，你可能看到：

```sql
wait_event_type = 'LWLock'
wait_event = 'ProcArray'
```

源码层的解释仍然回到本节模型：

```text
waiter 没有立即拿到 ProcArrayLock；
它进入对应 LWLock waiters 队列；
release 端唤醒它；
它重新竞争 ProcArrayLock；
成功后才进入受保护临界区。
```

### 与 heavyweight lock manager 的关系

PostgreSQL 文档和 `lwlock.c` 注释都强调：

```text
User-level locking should be done with the full lock manager,
which depends on LWLocks to protect its shared state.
```

heavyweight lock manager 自己的共享 hash table 也要用 LWLock partition 保护。于是等待链可能呈现两层：

```text
SQL 层等待 relation lock:
  wait_event_type = Lock

lock manager 内部 hash partition 竞争:
  wait_event_type = LWLock
  wait_event = LockManager
```

诊断时必须区分：

```text
业务语义锁等待；
内核共享结构保护锁竞争。
```

## 12. 观测与诊断入口

### SQL 观测

最常用入口：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock';
```

你能看到：

```text
backend 当前正在等某类 LWLock tranche。
```

你不能直接看到：

```text
它在 waiters 队列的第几位；
它的 lwWaiting 是 WAITING 还是 PENDING_WAKEUP；
哪个 backend 持有目标 LWLock；
被唤醒后是否能立即 acquire 成功。
```

如果版本提供 `pg_wait_events`，可以查 wait event 描述：

```sql
SELECT type, name, description
FROM pg_wait_events
WHERE type = 'LWLock'
ORDER BY name;
```

### trace probe

文档列出的 LWLock probe 包括：

```text
lwlock-acquire
lwlock-release
lwlock-wait-start
lwlock-wait-done
lwlock-condacquire
lwlock-condacquire-fail
```

特别注意：

```text
lwlock-wait-done:
  表示 wait 被释放，不表示已经 acquire。
```

如果你用 DTrace / SystemTap / eBPF 观测，需要把：

```text
wait-done -> 后续 acquire
```

作为两个事件关联，而不是把 wait-done 直接计为锁持有开始。

### `LOCK_DEBUG` 与 `trace_lwlocks`

如果编译时定义 `LOCK_DEBUG`，配置项 `trace_lwlocks` 可以输出 lightweight lock 使用信息。

`lwlock.c` 中 debug 输出会打印类似：

```text
excl
shared
haswaiters
waiters
waking
```

这对源码实验很有帮助，但不是生产默认能力。

### gdb 观察点

源码调试可以在这些函数设断点：

```text
LWLockAcquire
LWLockQueueSelf
LWLockDequeueSelf
LWLockWakeup
LWLockRelease
LWLockReleaseAll
```

有价值的观察对象：

```text
lock->state
lock->waiters
MyProc->lwWaiting
MyProc->lwWaitMode
num_held_lwlocks
held_lwlocks[0..num_held_lwlocks-1]
```

如果要解码 `state`，回到上一节的位布局：

```text
HAS_WAITERS
WAKE_IN_PROGRESS
LOCKED
shared count
exclusive sentinel
```

### perf 观察

LWLock 竞争严重时，CPU 侧可能看到：

```text
LWLockAttemptLock
LWLockWaitListLock
perform_spin_delay
LWLockWakeup
PGSemaphoreLock
PGSemaphoreUnlock
```

如果热点主要在 `LWLockAttemptLock()`，说明大量 backend 在 fast-path CAS 上竞争。

如果热点扩展到 semaphore 与 scheduler，说明临界区足够长或竞争足够重，等待已经进入 sleep / wake path。

## 13. 常见误区

### 误区 1: “LWLock 被唤醒就说明已经拿到锁”

不对。wakeup 只意味着：

```text
你可以重新尝试了。
```

真正持锁必须看到后续 `LWLockAttemptLock()` 成功，并记录到 `held_lwlocks[]`。

### 误区 2: “waiters 队列是严格 FIFO owner handoff”

不对。`LWLockWakeup()` 会按队列选择可唤醒者，但不直接转交 ownership。新来的 backend 可能在被唤醒者调度前先拿到锁。

这是一种短临界区吞吐优先的选择。

### 误区 3: “`LW_FLAG_HAS_WAITERS` 表示队列一定非空”

不严谨。它表示 release 端应该考虑 waiters。真正链表是否为空要在 wait-list lock 下检查。

### 误区 4: “`LW_FLAG_WAKE_IN_PROGRESS` 表示某个 backend 已经拥有锁”

不对。它只表示已有 waiter 被移出队列并安排唤醒，release 端暂时不要重复唤醒更多人。

### 误区 5: “process semaphore 能表达所有等待语义”

不对。semaphore 只负责阻塞和计数。LWLock 语义靠 `lock->state`、`lock->waiters`、`PGPROC.lwWaiting` 和重试循环共同建立。

### 误区 6: “ERROR 会自动释放所有 C 层资源”

不对。PostgreSQL 的 ERROR 会跳出普通调用栈。LWLock 必须显式记录在 `held_lwlocks[]`，并由 `LWLockReleaseAll()` 清理。

### 误区 7: “LWLock 可以保护长时间操作，因为它能睡眠”

不对。LWLock 能睡眠是为了避免长时间 busy wait，不是鼓励长临界区。长临界区会把成本推到 wait queue、semaphore、进程调度和全局吞吐下降。

## 14. 课堂实验

### 实验 1: 画出 missed wakeup 时间线

目标：理解为什么必须“入队后再尝试”。

步骤：

1. 打开 `src/backend/storage/lmgr/lwlock.c` 顶部注释。
2. 手写一个错误协议：

```text
AttemptLock fails -> sleep
Release -> if waiters non-empty wake
```

3. 写出三步 interleaving：

```text
waiter 看到冲突；
holder release 但队列为空；
waiter 开始 sleep。
```

4. 再把 PostgreSQL 的四阶段协议套上去，说明为什么这条 interleaving 不会导致永久睡眠。

验收问题：

```text
入队前释放会怎样？
入队后释放会怎样？
入队后第二次尝试成功又会怎样？
```

### 实验 2: 跟踪 `lwWaiting` 三态

目标：理解 `WAITING` 和 `PENDING_WAKEUP` 的区别。

步骤：

1. 在源码中阅读：

```text
LWLockQueueSelf()
LWLockDequeueSelf()
LWLockWakeup()
```

2. 建表记录每个函数会写什么：

| 函数 | 写入状态 | 语义 |
| --- | --- | --- |
| `LWLockQueueSelf()` | `LW_WS_WAITING` | 已在队列。 |
| `LWLockWakeup()` | `LW_WS_PENDING_WAKEUP` | 已出队，待 post。 |
| `LWLockWakeup()` post 前 | `LW_WS_NOT_WAITING` | 允许目标 backend 认为等待完成。 |
| `LWLockDequeueSelf()` | `LW_WS_NOT_WAITING` | 自己撤销排队。 |

3. 解释为什么没有 `PENDING_WAKEUP` 会让 `LWLockDequeueSelf()` 难以判断自己是否还在链表中。

### 实验 3: 观察 `pg_stat_activity` 中的 LWLock wait

目标：把 runtime 现象和源码协议连接起来。

步骤：

1. 找一个容易制造 LWLock 竞争的环境，例如高并发小事务、热点 buffer 或 ProcArray 压力。
2. 在另一个会话执行：

```sql
SELECT pid, wait_event_type, wait_event, state
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock'
ORDER BY pid;
```

3. 对一个 `wait_event`，回答：

```text
它对应哪个 tranche？
它能否告诉你具体 holder？
它能否证明这个 backend 已经进入 semaphore sleep？
它能否证明 wait-done 后立刻 acquire 成功？
```

预期结论：

```text
SQL wait event 是入口，不是完整状态 dump。
必须结合源码协议理解它能看见和看不见什么。
```

### 实验 4: 用 gdb 看 `held_lwlocks[]`

目标：理解 ERROR-safe release 的本地账本。

步骤：

1. 在测试环境给 `LWLockAcquire()` 和 `LWLockRelease()` 设断点。
2. 让一个 backend 执行会触发 shared memory 访问的 SQL。
3. 在断点处观察：

```text
p num_held_lwlocks
p held_lwlocks[0]
p lock->state
p MyProc->lwWaiting
```

4. 找到成功 acquire 后 `num_held_lwlocks` 增加、release 后减少的时刻。

延伸问题：

```text
如果 acquire 成功后发生 ERROR，AbortTransaction() 靠什么知道要释放这个 LWLock？
```

### 实验 5: 阅读 `LWLockAcquireOrWait()` 的旁路语义

目标：理解同一等待协议如何服务“不一定要获得锁”的场景。

步骤：

1. 阅读 `LWLockAcquireOrWait()`。
2. 找出它和 `LWLockAcquire()` 相同的部分：

```text
第一次尝试；
入队；
第二次尝试；
semaphore wait；
extraWaits 修正。
```

3. 找出它不同的部分：

```text
等待模式使用 LW_WAIT_UNTIL_FREE；
等待后可能返回 false，表示没有 acquire。
```

验收问题：

```text
为什么 WAL flush 等场景只需要等锁释放，而不一定要拿锁？
```

## 15. 讨论题

1. 如果 `LWLockAcquire()` 失败后直接睡在 semaphore 上，不入队也不二次尝试，最小 missed wakeup 时间线是什么？
2. `LWLockWakeup()` 为什么不在持有 wait-list lock 时直接 post semaphore？
3. 为什么 `lwlock-wait-done` probe 不能被当成 “lock acquired” 事件？
4. `WAKE_IN_PROGRESS` 牺牲了什么公平性，换来了什么成本收益？
5. 为什么 `LWLockReleaseAll()` 要在 `AbortTransaction()` 中尽早发生，而 heavyweight lock 不能同样立刻全部释放？
6. 如果某段代码经常触发 `too many LWLocks taken`，这更像是配置问题、workload 问题，还是内核设计问题？
7. 对一个 `BufferMapping` LWLock wait，SQL 层能观测到哪些事实，哪些只能通过源码、perf 或 gdb 推断？
8. 如果一个 backend 在 `PENDING_WAKEUP` 状态下又准备进入新的 LWLock 等待，为什么必须先等旧 wakeup 协议完成？

## 16. 本节小结

本节的主线是：

```text
LWLock 的慢路径不是“拿不到锁就睡眠”，而是一个跨进程的可靠等待协议。
```

关键结论：

```text
1. missed wakeup 来自“观察到冲突”和“发布等待意图”之间的竞态。
2. PostgreSQL 用“先尝试、入队、再尝试、再睡眠”关闭这个竞态窗口。
3. waiters 队列保存的是需要被唤醒重试的 backend，不是即将获得 ownership 的严格队列。
4. lwWaiting 三态让 waiter 和 waker 在出队、post semaphore、撤销排队之间安全握手。
5. WAKE_IN_PROGRESS 抑制重复唤醒，降低调度风暴，但不保证公平转交。
6. process semaphore 只承载阻塞和唤醒，LWLock 语义来自 state、waiters、PGPROC 和 retry loop。
7. held_lwlocks[] 与 LWLockReleaseAll() 让 ERROR 后不会遗留 shared memory 内部锁。
```

可迁移的系统规律：

```text
任何“先检查条件，再决定睡眠”的并发系统，都必须让等待意图和条件复查形成一个闭环。
可靠等待不是 sleep primitive 自己提供的，而是由：
  条件状态
  wait queue
  waiter visible state
  wakeup ordering
  retry loop
  cleanup ledger
共同构成。
```

下一节会从 LWLock 转向 `Latch`：当等待对象不再是“某个共享内存锁释放”，而是“进程级异步事件、信号、socket readiness 或 timeout”时，PostgreSQL 如何把唤醒抽象提升到 backend 自己的 latch。
