# PostgreSQL Latch 与进程级异步唤醒

## 课程定位

前置知识：已经理解 `SpinLock` 只适合极短共享字段更新，也理解 `LWLock` 通过 `PGPROC`、等待队列和 process semaphore 解决“等待某个共享锁可用”的问题。

本节唯一主问题：

```text
为什么后台进程不能靠周期性 pg_usleep() 轮询信号和共享标志，
SetLatch() / ResetLatch() / WaitLatch() 以及 PGPROC.procLatch
如何把 signal、postmaster death、timeout 和 socket readiness
统一成可靠可观测的等待点？
```

核心矛盾：PostgreSQL 有大量后台进程和内部等待循环，它们既要在没有工作时长期睡眠，降低 CPU wakeup；又要在信号、其它 backend 通知、postmaster 死亡、超时、socket 可读写等事件到来时快速醒来。简单的 `pg_usleep()` 加全局 flag 不能同时满足这两点：

```text
睡太短:
  响应快，但空闲系统持续被定时唤醒。

睡太长:
  省 CPU，但配置重载、shutdown、bgwriter notification、WAL receiver socket
  等事件可能被延迟处理。

只依赖 signal interrupt sleep:
  不可移植，而且存在 signal 已到达但进程尚未进入 sleep 的窗口。
```

PostgreSQL 的解法是把“有事要处理”拆成两层：

```text
业务状态:
  例如 ConfigReloadPending、ShutdownRequestPending、buffer allocation notification、
  WAL receiver socket readability、postmaster death。

唤醒状态:
  一个 Latch 的 is_set / maybe_sleeping / owner_pid，
  加 WaitEventSet 对操作系统等待原语的封装。
```

学完后应能独立判断：

```text
为什么 signal handler 中通常是“设置 flag，然后 SetLatch(MyLatch)”；
为什么等待循环常见写法是 ResetLatch() -> 检查工作 -> WaitLatch()；
为什么 SetLatch() 不是业务消息队列，只是唤醒提示；
为什么 shared latch 只能由 owner 等待，却能被其它进程 SetLatch()；
为什么 PGPROC.procLatch 比随手放一个 shared latch 更适合进程级通知；
为什么 WaitLatch() 要强制 postmaster child 处理 postmaster death；
为什么 timeout 是 fallback，不应该成为后台进程推进状态的唯一方式；
为什么 pg_stat_activity 看到的是等待点名字，不是“谁唤醒了我”。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节讲 `LWLock` 等待队列时，等待对象非常明确：

```text
我要等某个 LWLock 的 state 允许 shared / exclusive acquire。
```

因此 `LWLockAcquire()` 可以把 waiter 挂到 `lock->waiters`，release 时由 `LWLockWakeup()` 唤醒。

但 PostgreSQL 内部还有另一类等待：

```text
我不是在等一个锁；
我是在等“可能有各种事情发生”。
```

典型例子是 background writer：

```text
后台空闲时要睡；
有 buffer allocation 时可能需要被唤醒；
达到 bgwriter_delay 时要做周期性工作；
postmaster 死亡时要退出；
收到 shutdown / config reload signal 时要处理 flag。
```

如果每种事情都发明一套等待原语，后台进程主循环会变成：

```text
sleep 一会儿；
检查 signal flag；
检查 shared memory flag；
检查 socket；
检查 postmaster；
检查 timeout；
继续 sleep。
```

这既浪费 CPU，也容易错过事件。Latch 的位置就在这里：

```text
LWLock:
  等一个共享锁状态变化。

Latch:
  等“本进程需要重新检查自己关心的一组条件”。
```

本节只围绕进程级异步唤醒展开。`ConditionVariable` 会在下一节继续讲：它会把“等待某个 predicate 变化”的 higher-level 协议建立在 `MyLatch` 之上。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Latch 是一个可跨信号处理器和进程设置的 sticky wakeup bit；
调用方先把真正的业务状态写到共享内存或 backend-local flag，
再 SetLatch() 唤醒 owner；
owner 在循环中 ResetLatch()、检查业务条件，
如果没有工作，就通过 WaitLatch() / WaitEventSetWait() 同时等待 latch、timeout、
postmaster death 和 socket readiness。
```

这里有四个关键点。

第一，`Latch` 不是消息队列。

```text
SetLatch() 只表达:
  你应该醒来重新检查条件。

SetLatch() 不表达:
  具体是哪件事发生；
  发生了几次；
  是否仍然需要做；
  是否已经获得某个资源。
```

第二，`Latch` 是 sticky 的。

```text
一旦 is_set=true，WaitLatch(WL_LATCH_SET) 会立刻返回；
只有 owner 调用 ResetLatch() 后，下一次 WaitLatch() 才可能睡眠。
```

第三，`ResetLatch()` 的位置是 correctness 的一部分。

正确模式通常是：

```text
for (;;)
{
    ResetLatch();
    if (work to do)
        do_work();
    WaitLatch(...);
}
```

这样即使别人刚刚 `SetLatch()`，等待者也会在 reset 后立刻检查业务状态；只要业务状态先于 `SetLatch()` 发布，就不会依赖 latch 自身携带消息。

第四，`WaitLatch()` 是 `WaitEventSet` 的薄封装。

```text
WaitLatch()
  -> 使用进程级 LatchWaitSet
  -> 可动态启用 / 禁用 WL_LATCH_SET
  -> 可切换 WL_EXIT_ON_PM_DEATH / WL_POSTMASTER_DEATH
  -> 调用 WaitEventSetWait()
  -> 返回 WL_LATCH_SET / WL_TIMEOUT / WL_POSTMASTER_DEATH 等 bitmask
```

所以 Latch 的本质不是一个独立睡眠系统，而是 PostgreSQL 把异步通知折叠进统一 wait event 基础设施的入口。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/latch.h` | `Latch` 字段、local/shared latch 语义、正确等待模式。 |
| 2 | `src/backend/storage/ipc/latch.c` | `InitLatch()`、`InitSharedLatch()`、`OwnLatch()`、`WaitLatch()`、`SetLatch()`、`ResetLatch()`。 |
| 3 | `src/include/storage/waiteventset.h` | `WL_LATCH_SET`、`WL_TIMEOUT`、`WL_POSTMASTER_DEATH`、`WL_SOCKET_*` 等事件 bit。 |
| 4 | `src/backend/storage/ipc/waiteventset.c` | self-pipe / signalfd / kqueue / Windows event 的平台封装，以及 `maybe_sleeping` 防 missed wakeup。 |
| 5 | `src/backend/utils/init/miscinit.c` | `MyLatch` 从 local latch 切到 `PGPROC.procLatch`，再在退出时切回 local latch。 |
| 6 | `src/backend/storage/lmgr/proc.c` | `PGPROC.procLatch` 初始化、ownership、进程退出时 disown。 |
| 7 | `src/backend/postmaster/interrupt.c` | signal handler 如何设置 pending flag 并 `SetLatch(MyLatch)`。 |
| 8 | `src/backend/postmaster/bgwriter.c` | 后台主循环如何用 `ResetLatch()` / `WaitLatch()` 组合周期性工作和异步唤醒。 |
| 9 | `src/backend/storage/buffer/freelist.c` | 普通 backend 如何通过 `PGPROC.procLatch` 唤醒 bgwriter。 |
| 10 | `src/backend/utils/activity/wait_event_names.txt` 与 `doc/src/sgml/monitoring.sgml` | 用户能从 `pg_stat_activity` 看到哪些等待点，哪些只能推断。 |

推荐阅读顺序：

```text
先读 latch.h 顶部注释，理解为什么不用 pg_usleep() + flag；
再读 Latch 结构体字段；
然后读 ResetLatch() / SetLatch()；
再进入 WaitLatch() 和 WaitEventSetWait()；
最后拿 bgwriter 主循环验证这个协议如何落地。
```

不要从 `epoll_wait()` 或 `poll()` 分支开始读。平台等待原语只是实现细节，本课的抽象边界是：

```text
业务状态先发布；
Latch 只负责可靠唤醒；
WaitEventSet 负责把多个可等待事件统一交给操作系统。
```

## 4. 关键数据结构与状态

### `Latch`: 一个有 owner 的 sticky wakeup bit

`src/include/storage/latch.h` 中的 `Latch` 包含这几个关键字段：

```text
is_set:
  latch 是否已被设置。
  为 true 时，等待 WL_LATCH_SET 的 WaitLatch() 应立即返回。

maybe_sleeping:
  owner 是否正处于“准备或正在进入 OS wait primitive”的窗口。
  setter 用它判断是否需要执行真正的跨进程唤醒动作。

is_shared:
  local latch 还是 shared latch。

owner_pid:
  当前可以 WaitLatch() / ResetLatch() 这个 latch 的进程 pid。

event:
  Windows 下使用的事件对象。
```

不要把 `is_set` 单独理解成语义。`Latch` 的语义来自组合：

```text
is_set:
  是否有未消费的 wakeup。

maybe_sleeping:
  setter 是否需要触碰 OS wakeup primitive。

owner_pid:
  应该唤醒哪个进程。

caller-side business flag:
  醒来后究竟要做什么。
```

### local latch 与 shared latch

PostgreSQL 有两种 latch：

```text
local latch:
  InitLatch() 初始化；
  属于当前进程；
  主要用于同一进程 signal handler 唤醒主循环。

shared latch:
  InitSharedLatch() 初始化在 shared memory 中；
  初始没有 owner；
  owner 调用 OwnLatch() 后才能 WaitLatch() / ResetLatch()；
  其它进程可以 SetLatch() 唤醒 owner。
```

`src/include/storage/latch.h` 明确给出约束：

```text
只有 owner 可以 wait / reset 一个 latch；
shared latch 可以被其它进程 set。
```

这和 LWLock 的等待队列不同。`SetLatch()` 不需要把 caller 挂到某个队列，也不需要转移资源所有权。它只是把 owner 从等待中叫醒。

### `MyLatch`: signal handler 可以无条件使用的当前进程 latch

`src/backend/utils/init/globals.c` 对 `MyLatch` 的注释非常关键：

```text
当前进程没有 PGPROC 时:
  MyLatch 指向 backend-local 的 LocalLatchData。

当前进程有 PGPROC 后:
  MyLatch 指向 MyProc->procLatch。
```

这个设计让 signal handler 可以写成：

```text
flag = true;
SetLatch(MyLatch);
```

而不必在信号处理器里判断：

```text
MyProc 是否已经存在？
procLatch 是否已经被 OwnLatch()？
当前进程是否已经进入 shared memory 阶段？
```

`src/backend/utils/init/miscinit.c` 中的切换路径是：

```text
InitProcessLocalLatch()
  -> MyLatch = &LocalLatchData
  -> InitLatch(MyLatch)

InitProcess() / InitAuxiliaryProcess()
  -> OwnLatch(&MyProc->procLatch)
  -> SwitchToSharedLatch()
     -> MyLatch = &MyProc->procLatch
     -> 如果 FeBeWaitSet 存在，ModifyWaitEvent() 指向新的 latch
     -> SetLatch(MyLatch)

ProcKill() / AuxiliaryProcKill()
  -> SwitchBackToLocalLatch()
     -> MyLatch = &LocalLatchData
     -> 如果 FeBeWaitSet 存在，ModifyWaitEvent() 指回 local latch
     -> SetLatch(MyLatch)
  -> DisownLatch(&proc->procLatch)
```

这里的 `SetLatch(MyLatch)` 看起来多余，但它是在切换指针后保守唤醒自己：

```text
旧 latch 可能已经 set；
新 latch 被 set 后，等待循环至少会重新检查一次条件。
```

### `PGPROC.procLatch`: 进程级跨进程通知入口

`PGPROC.procLatch` 是 shared memory 中每个 backend / auxiliary process 的进程级 latch。它的价值在于：

```text
其它进程可以通过 PGPROC 找到你；
可以 SetLatch(&proc->procLatch)；
signal handler 也统一 SetLatch(MyLatch)；
procLatch 永远嵌在 PGPROC 里，不会像普通 palloc 对象那样被释放。
```

`src/backend/storage/lmgr/proc.c` 在 `ProcGlobalShmemInit()` 阶段为 `PGPROC` 初始化 `procLatch`，在 backend 或 auxiliary process 拿到 `PGPROC` 后：

```text
OwnLatch(&MyProc->procLatch);
SwitchToSharedLatch();
```

进程退出时：

```text
SwitchBackToLocalLatch();
DisownLatch(&proc->procLatch);
proc->pid = 0;
```

这也是为什么很多地方宁愿用 `PGPROC.procLatch`，而不是临时创建一个 shared latch。`latch.h` 顶部注释直接提醒：

```text
process latch 通常优于 ad-hoc shared latch，
因为 generic signal handlers 只会 SetLatch(MyLatch)。
```

### `WaitEventSet`: 多事件等待的统一承载

`src/include/storage/waiteventset.h` 定义了可等待事件：

| 事件 bit | 含义 |
| --- | --- |
| `WL_LATCH_SET` | latch 被设置。 |
| `WL_TIMEOUT` | `WaitLatch()` 层面的超时参数。 |
| `WL_POSTMASTER_DEATH` | postmaster 死亡，作为返回 bit 交给 caller。 |
| `WL_EXIT_ON_PM_DEATH` | postmaster 死亡时直接退出。 |
| `WL_SOCKET_READABLE` | socket 可读。 |
| `WL_SOCKET_WRITEABLE` | socket 可写。 |
| `WL_SOCKET_CONNECTED` | Windows 下 socket connect 完成；非 Windows 映射到 writable。 |
| `WL_SOCKET_CLOSED` | socket 被 peer 关闭。 |
| `WL_SOCKET_ACCEPT` | server socket 可 accept；非 Windows 映射到 readable。 |

`WaitLatch()` 是常用入口，但它不是唯一入口：

```text
WaitLatch():
  使用进程级静态 LatchWaitSet，最多等待 latch + postmaster death。

WaitLatchOrSocket():
  临时创建 WaitEventSet，同时等待 latch、socket、postmaster death、timeout。

直接使用 WaitEventSet:
  长生命周期 event set，适合频繁等待多个 socket / latch 的路径。
```

`latch.h` 也提醒：

```text
频繁使用 latch + 多个额外事件时，长期存在的 WaitEventSet 通常比
反复 WaitLatchOrSocket() 更高效。
```

## 5. 主流程源码 walkthrough：一次 signal 唤醒后台主循环

本节主线选择最常见的后台进程模式：

```text
后台进程在 WaitLatch(MyLatch, WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH, ...)
中睡眠；
signal handler 设置 pending flag 并 SetLatch(MyLatch)；
主循环醒来，ResetLatch()，处理 pending flag，再决定是否继续睡。
```

### Step 1: 后台进程初始化 `MyLatch`

进程早期会先拥有 local latch：

```text
InitProcessLocalLatch()
  -> MyLatch = &LocalLatchData
  -> InitLatch(MyLatch)
```

`InitLatch()` 做的事情很少：

```text
latch->is_set = false;
latch->maybe_sleeping = false;
latch->owner_pid = MyProcPid;
latch->is_shared = false;
```

当进程拿到 `PGPROC` 后，例如 `InitProcess()` 或 `InitAuxiliaryProcess()`：

```text
OwnLatch(&MyProc->procLatch)
  -> 检查 latch 是 shared
  -> 检查 owner_pid == 0
  -> owner_pid = MyProcPid

SwitchToSharedLatch()
  -> MyLatch = &MyProc->procLatch
  -> 更新 FeBeWaitSet 中的 latch 指针
  -> SetLatch(MyLatch)
```

从这个时刻开始：

```text
signal handler:
  SetLatch(MyLatch)

其它 backend:
  SetLatch(&target_proc->procLatch)

后台主循环:
  WaitLatch(MyLatch, ...)
```

三者都汇合到同一个 per-process wakeup object 上。

### Step 2: 主循环在工作前清掉旧 wakeup

以 `src/backend/postmaster/bgwriter.c` 为例，主循环开头是：

```text
ResetLatch(MyLatch);
ProcessMainLoopInterrupts();
can_hibernate = BgBufferSync(&wb_context);
...
rc = WaitLatch(MyLatch,
               WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
               BgWriterDelay,
               WAIT_EVENT_BGWRITER_MAIN);
```

这个顺序很重要：

```text
ResetLatch()
  -> 清掉上一轮已经消费过的 wakeup。

ProcessMainLoopInterrupts()
  -> 检查 ConfigReloadPending、ShutdownRequestPending 等 signal flag。

BgBufferSync()
  -> 做本轮真正工作。

WaitLatch()
  -> 如果这期间没有新事件，就睡；
  -> 如果这期间有人 SetLatch()，立即返回。
```

`ResetLatch()` 内部不是简单赋值。它还带一个 memory barrier：

```text
latch->is_set = false;
pg_memory_barrier();
```

注释说明了原因：

```text
必须确保 is_set=false 的写入先对外可见，
然后本进程再检查 flag variables。
否则 concurrent SetLatch() 可能误以为不需要 signal，
而等待者又没有看到它想通知的业务 flag。
```

换句话说，`ResetLatch()` 和“检查业务条件”是一组协议：

```text
ResetLatch();
检查业务状态;
如果没有工作，WaitLatch();
```

不能随意把业务条件检查放在 `WaitLatch()` 之后、`ResetLatch()` 之前。

### Step 3: signal handler 发布业务 flag 并设置 latch

`src/backend/postmaster/interrupt.c` 中有两个典型 signal handler：

```text
SignalHandlerForConfigReload()
  -> ConfigReloadPending = true
  -> SetLatch(MyLatch)

SignalHandlerForShutdownRequest()
  -> ShutdownRequestPending = true
  -> SetLatch(MyLatch)
```

这就是 Latch 的标准用法：

```text
先写业务状态；
再 SetLatch()。
```

如果只 `SetLatch()` 不写业务状态，主循环醒来也不知道要做什么。

如果只写业务状态不 `SetLatch()`，主循环可能正在无限或长时间 sleep，响应只能依赖 timeout。

### Step 4: `SetLatch()` 设置 sticky bit，并按需唤醒 owner

`src/backend/storage/ipc/latch.c` 中 `SetLatch()` 的关键顺序是：

```text
pg_memory_barrier();

if (latch->is_set)
    return;

latch->is_set = true;

pg_memory_barrier();
if (!latch->maybe_sleeping)
    return;

owner_pid = latch->owner_pid;
if (owner_pid == 0)
    return;
else if (owner_pid == MyProcPid)
    WakeupMyProc();
else
    WakeupOtherProc(owner_pid);
```

这段逻辑回答了三个问题。

第一，为什么 `SetLatch()` 已经 set 时直接返回？

```text
Latch 是 sticky wakeup bit。
如果 is_set 已经为 true，owner 的下一次 WaitLatch(WL_LATCH_SET) 本来就会返回；
重复 signal 通常只会增加成本，不增加语义。
```

第二，为什么要看 `maybe_sleeping`？

```text
如果 owner 不在或不准备进入 OS wait primitive，
只设置 is_set 就够了；
下一次 WaitLatch() 会先检查 is_set。

只有 owner 可能已经在内核等待中，
才需要 signal / self-pipe / Windows event 把它叫醒。
```

第三，为什么 `SetLatch()` 可以在 signal handler 和 critical section 中调用？

源码注释强调：

```text
SetLatch() 不能依赖会抛 ERROR 的路径；
它要尽量快速、安全，即使 OS wakeup 失败也可能只能静默忽略。
```

这也是为什么它不做复杂业务逻辑、不分配内存、不持有 heavyweight state。

### Step 5: `WaitLatch()` 进入 `WaitEventSetWait()`

`WaitLatch()` 本身很薄：

```text
Assert(postmaster child 必须处理 postmaster death);

if (!(wakeEvents & WL_LATCH_SET))
    latch = NULL;

ModifyWaitEvent(LatchWaitSet, LatchWaitSetLatchPos, WL_LATCH_SET, latch);

if (IsUnderPostmaster)
    ModifyWaitEvent(LatchWaitSet, LatchWaitSetPostmasterDeathPos,
                    WL_EXIT_ON_PM_DEATH 或 WL_POSTMASTER_DEATH,
                    NULL);

rc = WaitEventSetWait(LatchWaitSet,
                      timeout or -1,
                      &event,
                      1,
                      wait_event_info);

return WL_TIMEOUT 或 event.events;
```

`InitializeLatchWaitSet()` 创建的 `LatchWaitSet` 有两个槽位：

```text
slot 0:
  WL_LATCH_SET，默认关联 MyLatch。

slot 1:
  postmaster death，只有 IsUnderPostmaster 时添加；
  每次 WaitLatch() 可切换 WL_EXIT_ON_PM_DEATH / WL_POSTMASTER_DEATH。
```

这解释了为什么 `WaitLatch()` 参数里总能看到：

```text
WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH
```

后台进程在 postmaster 管理下运行时，不能假装 postmaster death 不存在。否则 postmaster 已死，子进程可能继续持有 shared memory 状态，系统恢复边界会变差。

### Step 6: `WaitEventSetWait()` 关闭 missed wakeup 窗口

`WaitEventSetWait()` 的核心不是 `epoll_wait()` 本身，而是进入 OS wait 前的这一段：

```text
if (set->latch && !set->latch->is_set)
{
    set->latch->maybe_sleeping = true;
    pg_memory_barrier();
}

if (set->latch && set->latch->is_set)
{
    返回 WL_LATCH_SET;
    set->latch->maybe_sleeping = false;
}

rc = WaitEventSetWaitBlock(...);

if (set->latch && set->latch->maybe_sleeping)
    set->latch->maybe_sleeping = false;
```

这个“设置 maybe_sleeping 后再检查 is_set”的结构，和上一节 `LWLockAcquire()` 的“入队后再尝试”是同一种并发形状：

```text
等待者必须先发布“我可能要睡了”；
再重新检查事件是否已经发生；
如果事件已经发生，就不能进入 OS sleep。
```

错误模式是：

```text
if (!latch->is_set)
    poll(...);
```

竞态窗口：

```text
T1 waiter 看到 is_set=false；
T2 setter SetLatch()，看到 maybe_sleeping=false，只设置 is_set=true，不 signal；
T3 waiter 进入 poll()，但没有 fd readiness；
T4 waiter 睡过头，直到 timeout 或其它事件。
```

正确模式是：

```text
T1 waiter 看到 is_set=false；
T2 waiter 设置 maybe_sleeping=true；
T3 waiter memory barrier 后重读 is_set；
T4 setter 如果此时 SetLatch()，会看到 maybe_sleeping=true 并执行 wakeup；
T5 waiter 要么看到 is_set=true 直接返回，要么进入 OS wait 并被 wakeup。
```

这就是 Latch 作为“可靠 pg_usleep replacement”的核心。

### Step 7: 平台层如何真正唤醒等待原语

`src/backend/storage/ipc/waiteventset.c` 根据平台选择等待原语：

```text
Linux epoll:
  通常使用 signalfd 接收 SIGURG。

poll:
  使用 self-pipe trick。

kqueue:
  使用 EVFILT_SIGNAL 等待 SIGURG。

Windows:
  使用 Windows event。
```

文件头注释解释了为什么不能只靠 signal：

```text
有些平台 signal 不会中断 sleep；
即使会中断，signal 如果刚好在 poll/select 之前到达，
也不能阻止随后进入 sleep。
```

self-pipe 的思想是：

```text
signal handler 或 WakeupMyProc() 往 pipe 写一个字节；
等待者在 poll/epoll 中等待 pipe readable；
即使 signal 发生在进入 poll 之前，pipe 中的字节仍然可读。
```

PostgreSQL 把这个细节藏在：

```text
WakeupMyProc()
WakeupOtherProc(pid)
drain()
WaitEventSetWaitBlock()
```

调用者不需要知道当前平台用 self-pipe、signalfd、kqueue 还是 Windows event。

### Step 8: 主循环醒来后重新解释业务状态

`WaitLatch()` 返回后，caller 不应该把返回值理解成“具体工作已经被交付”。例如 bgwriter：

```text
rc = WaitLatch(...);

if (rc == WL_TIMEOUT && can_hibernate && prev_hibernate)
    进入 hibernation 逻辑；

下一轮 loop:
  ResetLatch(MyLatch);
  ProcessMainLoopInterrupts();
  BgBufferSync(...);
```

它并不依赖 `WL_LATCH_SET` 本身携带“哪个 backend 分配了 buffer”。真正状态在其它地方：

```text
StrategyControl->bgwprocno
numBufferAllocs
ConfigReloadPending
ShutdownRequestPending
postmaster death fd
```

Latch 返回只是说：

```text
不要继续睡了；
重新进入主循环；
按自己的业务规则检查状态。
```

## 6. 跨进程唤醒 walkthrough：普通 backend 唤醒 bgwriter

上一条主流程从 signal handler 出发。这一节再看一个跨进程的真实路径：buffer 分配触发 bgwriter wakeup。

### Step 1: bgwriter 声明“下次 buffer allocation 时叫我”

`src/backend/postmaster/bgwriter.c` 中 hibernation 分支：

```text
if (rc == WL_TIMEOUT && can_hibernate && prev_hibernate)
{
    StrategyNotifyBgWriter(MyProcNumber);

    WaitLatch(MyLatch,
              WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
              BgWriterDelay * HIBERNATE_FACTOR,
              WAIT_EVENT_BGWRITER_HIBERNATE);

    StrategyNotifyBgWriter(-1);
}
```

这表达的是：

```text
当前系统看起来空闲；
bgwriter 可以睡久一点；
但如果普通 backend 开始分配 buffer，请通过我的 PGPROC.procLatch 叫醒我。
```

### Step 2: 普通 backend 分配 buffer 时读取通知槽

`src/backend/storage/buffer/freelist.c` 中：

```text
bgwprocno = INT_ACCESS_ONCE(StrategyControl->bgwprocno);
if (bgwprocno != -1)
{
    StrategyControl->bgwprocno = -1;
    SetLatch(&GetPGProcByNumber(bgwprocno)->procLatch);
}
```

源码注释承认这里没有拿 ProcArrayLock：

```text
如果 bgwriter 在错误时刻死亡或 PGPROC 被复用，
可能 set 到错误进程的 latch；
但 PGPROC->procLatch 不会被释放，
最坏只是额外唤醒某个进程。
```

这是一种很 PostgreSQL 的工程取舍：

```text
目标不是精确投递消息；
目标是低成本地提示“系统不再空闲，你该醒来检查了”。
```

因为真正的状态不是 latch，而是 buffer allocation 计数与 bgwriter 自己下一轮观察到的共享状态。

### Step 3: 为什么这不是 missed wakeup？

这个路径里确实存在 races，但它们被业务语义吸收：

```text
backend 可能在 bgwriter 设置 bgwprocno 前分配 buffer；
bgwriter 可能还是进入 hibernation；
timeout 会兜底醒来。
```

源码注释也说这不是 correctness-critical。它只是避免空闲系统刚恢复活动时 bgwriter 睡得过久：

```text
not critical;
try to reduce odds;
mitigate by not hibernating forever.
```

这说明 Latch 不能自动把所有竞态变成强一致通知。它提供的是：

```text
可靠唤醒 owner 的机制。
```

是否需要强一致，取决于业务状态和调用方协议。bgwriter hibernation 是优化路径，所以允许 timeout fallback。

## 7. `WaitLatchOrSocket()`：把 socket readiness 放进同一个等待点

很多 replication、libpq helper、logical worker 路径既要等网络，又要处理 signal / shutdown / postmaster death。典型需求是：

```text
等 socket readable；
但如果收到 SIGTERM，要退出；
如果 postmaster 死亡，要退出；
如果超时，要做 keepalive 或重试；
如果有 latch set，要处理 pending flag。
```

`WaitLatchOrSocket()` 做的事是：

```text
CreateWaitEventSet(CurrentResourceOwner, 3);

if (wakeEvents & WL_LATCH_SET)
    AddWaitEventToSet(set, WL_LATCH_SET, ..., latch, NULL);

if (wakeEvents & WL_POSTMASTER_DEATH)
    AddWaitEventToSet(set, WL_POSTMASTER_DEATH, ...);

if (wakeEvents & WL_EXIT_ON_PM_DEATH)
    AddWaitEventToSet(set, WL_EXIT_ON_PM_DEATH, ...);

if (wakeEvents & WL_SOCKET_MASK)
    AddWaitEventToSet(set, socket_events, sock, NULL, NULL);

WaitEventSetWait(set, timeout, &event, 1, wait_event_info);
FreeWaitEventSet(set);
```

返回值会合并：

```text
WL_LATCH_SET
WL_POSTMASTER_DEATH
WL_SOCKET_MASK
WL_TIMEOUT
```

这里再次体现了 Latch 的定位：

```text
Latch 不是替代 socket wait；
它是“当我正在等 socket 时，也能被进程内外异步事件打断”的机制。
```

频繁等待多个 fd 的地方，应该直接使用长生命周期 `WaitEventSet`，避免每次临时创建和销毁。

## 8. 生命周期 / ownership / cleanup

### local latch 生命周期

local latch 属于单个进程：

```text
InitProcessLocalLatch()
  -> 初始化 LocalLatchData
  -> MyLatch 指向它

进程获得 PGPROC 前:
  signal handler 可以 SetLatch(MyLatch)

进程退出释放 PGPROC 后:
  SwitchBackToLocalLatch()
  signal handler 仍然可以 SetLatch(MyLatch)
```

local latch 的核心价值是：

```text
MyLatch 永远可用。
```

这降低了 signal handler 的复杂性。

### shared latch 生命周期

shared latch 必须先放在 shared memory 中并初始化：

```text
InitSharedLatch(latch)
  -> is_set=false
  -> maybe_sleeping=false
  -> owner_pid=0
  -> is_shared=true
```

等待者在使用前：

```text
OwnLatch(latch)
  -> owner_pid = MyProcPid
```

退出或不再等待：

```text
DisownLatch(latch)
  -> owner_pid = 0
```

`OwnLatch()` 没有锁保护，只做 sanity check。源码注释明确说：

```text
如果两个进程可能同时 own 同一个 latch，
caller 必须自己提供 interlock。
```

所以 shared latch 的 ownership 是调用方协议的一部分，不能把它当成自动互斥对象。

### `PGPROC.procLatch` 生命周期

`PGPROC.procLatch` 是 shared latch 的特殊实例：

```text
创建:
  postmaster 初始化 PGPROC 数组时 InitSharedLatch(&proc->procLatch)。

owner:
  对应 backend / auxiliary process 在 InitProcess() / InitAuxiliaryProcess() 中 OwnLatch()。

使用:
  MyLatch 指向它；
  其它 backend 可通过 PGPROC 编号 SetLatch(&proc->procLatch)。

cleanup:
  进程退出时 SwitchBackToLocalLatch()；
  pgstat wait event storage reset；
  DisownLatch(&proc->procLatch)；
  PGPROC 标记为空闲。
```

这里的 cleanup 顺序不能随便换：

```text
先把 MyLatch 切回 local latch；
再清 MyProc / disown shared latch。
```

否则 signal handler 可能在退出过程中访问一个已经不再归本进程 owned 的 shared latch。

### WaitEventSet 生命周期

`WaitLatch()` 使用静态 `LatchWaitSet`：

```text
InitializeLatchWaitSet()
  -> CreateWaitEventSet(NULL, 2)
  -> 添加 WL_LATCH_SET
  -> 如果在 postmaster 下，添加 postmaster death event
```

`WaitLatchOrSocket()` 每次临时创建：

```text
CreateWaitEventSet(CurrentResourceOwner, 3)
...
FreeWaitEventSet(set)
```

直接使用 `WaitEventSet` 时，可以把它挂到 `ResourceOwner`：

```text
CreateWaitEventSet(CurrentResourceOwner, nevents)
```

这样 ERROR cleanup 时能由 resowner 释放，避免 fd / kernel object 泄漏。

## 9. 正确性机制层次

### 层次 1: 业务状态先于 latch

Latch 正确使用的第一条规则：

```text
先更新业务状态；
再 SetLatch()。
```

例如 signal handler：

```text
ShutdownRequestPending = true;
SetLatch(MyLatch);
```

跨进程通知：

```text
写 shared memory 状态；
SetLatch(&target_proc->procLatch);
```

等待者醒来后检查的是业务状态，不是 latch 本身。

### 层次 2: `ResetLatch()` 先于检查条件

`latch.h` 推荐模式：

```text
for (;;)
{
    ResetLatch();
    if (work to do)
        DoStuff();
    WaitLatch(...);
}
```

为什么不是：

```text
if (work to do)
    DoStuff();
ResetLatch();
WaitLatch(...);
```

因为如果别人刚好在检查条件之后、`ResetLatch()` 之前设置业务状态并 `SetLatch()`：

```text
T1 waiter 检查发现没工作；
T2 setter 写业务状态并 SetLatch()；
T3 waiter ResetLatch() 清掉这次 wakeup；
T4 waiter WaitLatch() 睡眠；
```

这就把 wakeup 消费掉了。把 `ResetLatch()` 放在检查条件之前，可以让 setter 的业务状态在检查时被看见。

### 层次 3: `maybe_sleeping` 关闭 wait 前窗口

`WaitEventSetWait()` 在真正阻塞前：

```text
set->latch->maybe_sleeping = true;
pg_memory_barrier();
重新检查 is_set;
```

`SetLatch()` 在设置 `is_set=true` 后：

```text
if (maybe_sleeping)
    WakeupMyProc() or WakeupOtherProc();
```

这个组合保证：

```text
如果 setter 发生在 waiter 准备睡眠之后，
setter 会触发 OS wakeup；

如果 setter 发生在 waiter 准备睡眠之前，
is_set=true 会让 WaitEventSetWait() 直接返回。
```

### 层次 4: OS wait primitive 只负责阻塞与返回

`poll` / `epoll` / `kqueue` / Windows event 不知道 PostgreSQL 的业务状态。它们只负责：

```text
fd readable / writable；
signal fd readable；
self-pipe readable；
postmaster alive pipe HUP；
Windows event signaled。
```

PostgreSQL 在其上提供：

```text
WaitEventSetWait()
  -> 转换为 WaitEvent
  -> 上报 pgstat wait event
  -> 处理 timeout
  -> 处理 postmaster death
  -> 返回给 caller
```

### 层次 5: wait event 观测只描述等待点

`WaitEventSetWait()` 会：

```text
pgstat_report_wait_start(wait_event_info);
...
pgstat_report_wait_end();
```

因此 `pg_stat_activity` 能看到：

```text
wait_event_type = 'Activity'
wait_event = 'BgWriterMain'
```

或：

```text
wait_event_type = 'Client'
wait_event = 'ClientRead'
```

但它看不到：

```text
latch 是谁 set 的；
业务 flag 是什么；
SetLatch() 发生了几次；
WaitLatch() 返回后 caller 做了什么。
```

这就是诊断边界。

## 10. 错误路径 / 异常路径 / fallback

### postmaster death

`WaitLatch()` 对 postmaster child 有断言：

```text
Assert(!IsUnderPostmaster ||
       wakeEvents 包含 WL_EXIT_ON_PM_DEATH 或 WL_POSTMASTER_DEATH);
```

这不是风格问题，而是生存边界：

```text
postmaster 是 shared memory 生命周期和子进程监管的中心；
postmaster 死亡后，普通子进程不能继续像正常系统一样长期运行。
```

两种模式语义不同：

```text
WL_EXIT_ON_PM_DEATH:
  等待层直接退出，适合大多数后台进程。

WL_POSTMASTER_DEATH:
  WaitLatch() 返回该 bit，由 caller 自己决定如何处理。
```

### timeout fallback

`WaitLatch()` 支持 `WL_TIMEOUT`，但 `latch.h` 注释提醒：

```text
timeouts should be avoided when possible, as they incur extra overhead.
```

这句话不是说不能用 timeout，而是说不要把 timeout 当成主要通知机制。

合理用法：

```text
周期性后台工作:
  bgwriter_delay、wal_writer_delay。

容忍 missed optimization wakeup 的 fallback:
  bgwriter hibernation。

网络 keepalive / retry:
  replication wait loop。
```

危险用法：

```text
业务状态只能靠每 N ms poll 发现；
SetLatch() 缺失或调用顺序错误；
低延迟要求却依赖长 timeout。
```

### `SetLatch()` 中不能抛复杂错误

`SetLatch()` 注释强调：

```text
可能被 signal handler 或 critical section 调用；
throwing an error is not a good idea。
```

因此 Unix 下唤醒其它进程时，如果底层 signal / pipe 操作遇到部分错误，很多路径只能静默忽略。系统可靠性不是靠每次 OS wakeup 都成功，而是靠：

```text
is_set 是 sticky；
业务状态仍然存在；
timeout / 后续事件可能兜底；
postmaster 监管进程生命周期。
```

这也意味着：

```text
SetLatch() 不是可靠消息投递 API。
```

它是低成本 wakeup API。

### owner 变化与 stale PGPROC

`SetLatch()` 读取 `owner_pid` 时，源码注释承认了一个现实：

```text
pid_t 原子性不是 C 标准保证；
shared latch 可能正在 own / disown；
可能 signal 到错误 pid。
```

实践中 PostgreSQL 依赖：

```text
pid_t 通常 fits in 32-bit 并可原子读写；
错误 signal 到 PostgreSQL 进程通常只是额外 SIGURG；
backend 能容忍多余 SIGUSR1 / SIGURG 类唤醒；
关键路径应有业务状态和 timeout fallback。
```

`freelist.c` 唤醒 bgwriter 的注释也体现了同一取舍：

```text
procLatch 不释放；
最坏 set 到错误进程；
只是额外 wakeup，不破坏 correctness。
```

### signal handler 中忘记 `SetLatch()`

这是最常见的错误模式：

```text
handler:
  flag = true;
  // 忘了 SetLatch(MyLatch)
```

结果：

```text
如果进程正在 WaitLatch() 且没有 timeout，可能永远不处理 flag；
如果有 timeout，只能等下一次超时；
如果恰好有其它事件唤醒，问题表现为偶发延迟。
```

`latch.h` 特别提醒：

```text
signals will not interrupt latch wait primitive by themselves on some platforms;
meant-to-terminate handlers must call SetLatch().
```

### `ResetLatch()` 位置错误

另一个常见错误：

```text
WaitLatch(...);
if (work to do)
    DoStuff();
ResetLatch();
```

`latch.h` 明确说：

```text
不能把 asynchronous event 检查放在 WaitLatch() 和 ResetLatch() 之间。
```

因为这会创建：

```text
检查条件 -> reset 清掉新 wakeup -> sleep
```

的 missed wakeup 窗口。

## 11. 成本、资源与跨模块传播

### `SetLatch()` 的成本不是零

`SetLatch()` 的 cheap path：

```text
is_set 已经为 true
  -> 直接返回。

maybe_sleeping 为 false
  -> 设置 is_set 后返回。
```

expensive path：

```text
maybe_sleeping 为 true
  -> signal / self-pipe / signalfd / Windows event
  -> 可能触发跨进程调度。
```

因此高频路径通常会避免无意义地连续 `SetLatch()`。例如 `src/backend/storage/ipc/shm_mq.c` 注释中就提到：

```text
SetLatch() calls are quite expensive.
```

这也是 sticky bit 的价值：

```text
如果 receiver 尚未 ResetLatch()，
重复 SetLatch() 可以被快速合并。
```

### `WaitLatch()` 与 `WaitEventSet` 的成本

`WaitLatch()` 使用进程级静态 `LatchWaitSet`，适合主循环：

```text
长期存在；
只在每次 wait 前 ModifyWaitEvent()；
避免反复分配内核对象。
```

`WaitLatchOrSocket()` 每次：

```text
CreateWaitEventSet()
AddWaitEventToSet()
WaitEventSetWait()
FreeWaitEventSet()
```

所以 `latch.h` 说：

```text
频繁使用时，long lived WaitEventSet 更高效。
```

### 与 signal 子系统的边界

signal handler 不做复杂工作：

```text
设置 pending flag；
SetLatch(MyLatch)；
返回。
```

真正处理发生在主循环：

```text
ProcessMainLoopInterrupts()
  -> 读取 pending flag
  -> ProcessConfigFile()
  -> proc_exit()
  -> ProcessLogMemoryContextInterrupt()
```

这个分层让 PostgreSQL 避免在 signal handler 里做不安全操作。

### 与后台进程调度的边界

很多后台进程的 main loop 形态类似：

```text
ResetLatch(MyLatch);
处理 pending interrupts；
做一轮可用工作；
WaitLatch(MyLatch,
          WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
          delay,
          对应后台进程的 main-loop wait event);
```

例如：

```text
bgwriter:
  WAIT_EVENT_BGWRITER_MAIN / BGWRITER_HIBERNATE。

walwriter:
  WAIT_EVENT_WAL_WRITER_MAIN。

checkpointer:
  WAIT_EVENT_CHECKPOINTER_MAIN。

archiver:
  WAIT_EVENT_ARCHIVER_MAIN。

logical launcher / apply worker:
  LOGICAL_LAUNCHER_MAIN / LOGICAL_APPLY_MAIN。
```

这让后台进程的“无工作等待”在用户视角里统一归类为 `Activity`。

### 与 socket / client wait 的边界

普通 backend 等客户端输入通常不是 `Activity`，而是 `ClientRead`。这是 wait event 分类语义：

```text
Activity:
  后台进程主循环没工作。

Client:
  等前端 socket。

IPC:
  等其它 server process 互动。

Timeout:
  等纯 timeout 到期。
```

Latch 可能参与这些等待，但 wait_event_type 不一定叫 `Latch`。用户看到的是等待场景，而不是底层原语。

## 12. 观测与诊断入口

### SQL: 看后台进程是否睡在主循环

可以观察当前等待点：

```sql
SELECT pid, backend_type, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY backend_type, pid;
```

常见结果：

```text
background writer:
  wait_event_type = Activity
  wait_event      = BgWriterMain 或 BgWriterHibernate

checkpointer:
  wait_event_type = Activity
  wait_event      = CheckpointerMain

walwriter:
  wait_event_type = Activity
  wait_event      = WalWriterMain

client backend:
  wait_event_type = Client
  wait_event      = ClientRead
```

解释边界：

```text
能看到:
  进程当前报出的等待点。

看不到:
  latch.is_set；
  maybe_sleeping；
  谁调用了 SetLatch()；
  被唤醒后处理了哪个 flag；
  一个 wakeup 是否被多个业务状态合并。
```

### 源码断点: 看 signal handler 到主循环

如果用 gdb 观察配置重载路径，可设断点：

```text
SignalHandlerForConfigReload
SetLatch
WaitEventSetWait
ProcessMainLoopInterrupts
```

观察重点不是停在 `SetLatch()` 里看复杂业务，而是看：

```text
ConfigReloadPending:
  signal handler 中置 true；
  主循环醒来后被 ProcessMainLoopInterrupts() 消费。

MyLatch:
  早期是 local latch；
  InitProcess 后是 MyProc->procLatch。
```

### perf / strace: 看 timeout polling 与 latch wakeup

在空闲系统中，后台进程通常应长期睡眠在：

```text
epoll_wait
poll
kevent
WaitForMultipleObjects
```

如果某个后台进程异常高频醒来，可以从两边查：

```text
谁在高频 SetLatch()？
WaitLatch() 是否使用了过短 timeout？
是否把 latch 当消息计数器导致重复唤醒？
业务 loop 是否每次醒来都 ResetLatch() 后又立即 SetLatch()？
```

### 日志与配置

Latch 本身通常不会打日志。诊断要结合：

```text
log_min_messages / debug 日志；
pg_stat_activity wait events；
后台进程特定统计视图；
gdb / perf；
源码断点或 tracepoint。
```

不要期待日志里出现：

```text
process X set latch of process Y
```

PostgreSQL 没有把 latch wakeup 作为默认可审计事件记录。

## 13. 常见误区

误区一：`SetLatch()` 就等于发送了一条消息。

正确理解：

```text
SetLatch() 只设置 sticky wakeup bit 并唤醒 owner；
消息内容必须在其它状态中。
```

误区二：`WaitLatch()` 返回 `WL_LATCH_SET` 就知道发生了什么。

正确理解：

```text
它只说明 latch 被 set；
caller 必须重新检查所有相关业务条件。
```

误区三：先检查条件，再 `ResetLatch()`，再 `WaitLatch()` 也可以。

正确理解：

```text
这会把检查条件和清 wakeup 之间的 SetLatch() 丢掉。
ResetLatch() 必须放在检查条件之前，或者使用 latch.h 描述的第二种安全模式。
```

误区四：signal 自然会打断等待，所以 signal handler 不需要 `SetLatch()`。

正确理解：

```text
这既不可移植，也有 signal-before-sleep 竞态。
如果 signal handler 的目标是让主循环尽快处理 flag，就要 SetLatch(MyLatch)。
```

误区五：`maybe_sleeping` 是 owner 是否一定睡着。

正确理解：

```text
maybe_sleeping 只是“可能正在进入或已经处于 wait primitive”的提示；
它允许 setter 保守地执行 wakeup。
```

误区六：shared latch 能自动处理多个 owner。

正确理解：

```text
shared latch 只能有一个 owner；
OwnLatch() 没有并发互斥；
如果 owner 可能竞争，caller 必须加 interlock。
```

误区七：`PGPROC.procLatch` 被 set 到错误进程一定是严重 bug。

正确理解：

```text
某些优化路径接受 stale proc number 导致的 extra wakeup；
前提是 procLatch 不释放，且业务正确性不依赖精确消息投递。
```

误区八：看到 `Activity/BgWriterMain` 就说明 bgwriter 卡住了。

正确理解：

```text
这通常只是 bgwriter 没有工作，在主循环等待；
要结合 buffer allocation、checkpoint、I/O、系统负载判断。
```

## 14. 课堂实验

### 实验 1: 从源码验证正确等待模式

目标：找出 PostgreSQL 后台进程主循环的 latch 模式。

步骤：

```bash
cd /home/highgo/postgres
rg -n "ResetLatch\\(MyLatch\\)|WaitLatch\\(MyLatch" src/backend/postmaster src/backend/replication
```

观察：

```text
bgwriter.c
walwriter.c
checkpointer.c
pgarch.c
autovacuum.c
walsender.c
walreceiver.c
logical launcher / worker
```

思考：

```text
哪些路径在 WaitLatch() 前 ResetLatch()？
哪些路径在 WaitLatch() 返回后 ResetLatch()？
它们是否符合 latch.h 注释中两种安全模式之一？
```

### 实验 2: 观察后台进程 wait event

目标：理解 Latch 等待在 SQL 视角下如何呈现。

步骤：

```sql
SELECT pid, backend_type, state, wait_event_type, wait_event
FROM pg_stat_activity
WHERE backend_type IN
      ('background writer', 'checkpointer', 'walwriter',
       'autovacuum launcher', 'archiver')
ORDER BY backend_type;
```

预期：

```text
空闲系统中，多个后台进程会显示 Activity 类型的 main loop wait event。
```

思考：

```text
为什么 wait_event_type 不是 Latch？
为什么这不能告诉你谁调用了 SetLatch()？
```

### 实验 3: 跟踪 SIGHUP 唤醒

目标：把 signal handler、latch 和主循环连起来。

步骤：

```bash
cd /home/highgo/postgres
rg -n "SignalHandlerForConfigReload|ConfigReloadPending|ProcessMainLoopInterrupts|SetLatch\\(MyLatch\\)" src/backend
```

阅读路径：

```text
SignalHandlerForConfigReload()
  -> ConfigReloadPending = true
  -> SetLatch(MyLatch)
  -> WaitLatch() 返回
  -> ProcessMainLoopInterrupts()
  -> ProcessConfigFile(PGC_SIGHUP)
```

思考：

```text
如果 handler 只设置 ConfigReloadPending，但不 SetLatch(MyLatch)，
后台进程何时才会 reload 配置？
```

### 实验 4: 阅读 bgwriter hibernation 的弱通知协议

目标：理解 Latch 不是强消息队列。

步骤：

```bash
cd /home/highgo/postgres
sed -n '300,345p' src/backend/postmaster/bgwriter.c
sed -n '210,235p' src/backend/storage/buffer/freelist.c
```

重点看注释：

```text
可能错过 buffer allocation；
可能 set 到错误 procLatch；
但 timeout fallback 和业务语义让它只是优化问题。
```

思考：

```text
如果这是事务提交确认路径，能否接受这种弱通知？
为什么 bgwriter 可以接受？
```

### 实验 5: 比较 `WaitLatch()` 和 `WaitLatchOrSocket()`

目标：理解 socket readiness 如何被合并进同一个等待模型。

步骤：

```bash
cd /home/highgo/postgres
rg -n "WaitLatchOrSocket\\(" src/backend src/include
```

选择一个 replication 或 libpq helper 路径，回答：

```text
它等待哪个 socket 事件？
它是否同时等待 WL_LATCH_SET？
它如何处理 WL_TIMEOUT？
它如何处理 postmaster death？
```

## 15. 讨论题

1. 为什么 `SetLatch()` 设计成 sticky bit，而不是每次 set 都累加一个 counter？

2. 如果一个后台进程的业务条件非常多，为什么仍然通常只需要一个 `MyLatch`？

3. `ResetLatch()` 放在循环顶部会不会清掉“别人刚刚通知我的事件”？为什么业务状态检查能弥补这一点？

4. 哪些路径可以接受 timeout 作为 fallback？哪些路径必须有精确的状态队列或锁协议？

5. 为什么 `PGPROC.procLatch` 能容忍偶尔 extra wakeup，但不能容忍指向已释放内存？

6. 如果你在扩展中写一个 background worker，什么情况下应该用 `MyLatch`，什么情况下需要自建 shared latch 或 condition variable？

7. 为什么 `pg_stat_activity.wait_event='BgWriterMain'` 不能直接解释成性能瓶颈？

8. `maybe_sleeping` 与上一节 `LWLockAcquire()` 的“入队后再尝试”解决的是同一类问题吗？它们的不同点是什么？

## 16. 本节小结

本节的核心问题是：

```text
PostgreSQL 如何让后台进程在无工作时安静睡眠，
同时在 signal、其它进程通知、postmaster death、timeout 和 socket readiness
到来时可靠醒来？
```

答案不是“用一个更好的 sleep 函数”，而是一个分层协议：

```text
业务状态:
  由调用方写入 flag、shared memory、socket state 或进程状态。

Latch:
  一个 sticky wakeup bit，负责把 owner 叫醒重新检查业务状态。

WaitEventSet:
  把 latch、postmaster death、timeout 和 socket readiness 统一交给平台等待原语。

wait event:
  把当前等待点暴露给 pg_stat_activity，但不暴露 wakeup 来源和业务消息。
```

最重要的不变量：

```text
先发布业务状态，再 SetLatch()；
等待者 ResetLatch() 后检查业务状态，再 WaitLatch()；
WaitEventSetWait() 在真正 sleep 前设置 maybe_sleeping 并重新检查 is_set；
WaitLatch() 返回后必须重新解释业务状态，而不是把 latch 当消息。
```

可迁移的系统规律：

```text
异步唤醒机制不要承载业务语义。
可靠设计通常把“状态在哪里”和“如何醒来检查状态”拆开：
状态由业务模块维护，唤醒原语只保证等待者不会睡过一个已经发布的检查机会。
```

下一节会在这个基础上进入 `ConditionVariable`：当多个 backend 不只是要“醒来检查”，而是要围绕某个 predicate 形成等待队列和广播唤醒时，PostgreSQL 如何在 `MyLatch` 之上构造更高层的条件等待协议。
