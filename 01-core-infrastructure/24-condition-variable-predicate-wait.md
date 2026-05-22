# PostgreSQL ConditionVariable 与谓词等待循环

## 课程定位

前置知识：已经理解 `SpinLock` 只保护极短共享字段更新，`LWLock` 解决 shared / exclusive 互斥与等待队列，`Latch` 解决进程级异步唤醒和 `WaitEventSet` 统一等待点。

本节唯一主问题：

```text
为什么等待“某个条件变真”不能只暴露一个 latch，
ConditionVariableSleep() / Signal() / Broadcast()
如何用 PGPROC.cvWaitLink、spinlock 保护的 wait list 和 spurious wakeup 规则，
让 buffer I/O、checkpoint、replication slot 等路径等待共享状态变化？
```

核心矛盾：很多 PostgreSQL 等待不是“等某个锁可用”，也不是“等本进程被任意事件叫醒”，而是“等某个共享谓词发生变化”：

```text
buffer 上的 BM_IO_IN_PROGRESS 清掉了吗？
checkpoint request 真的开始了吗？
checkpoint 是否已经完成到我发起的那一轮？
replication slot 是否已经不再被别的 backend 使用？
parallel scan 需要的 page number 是否已经产生？
```

这些谓词的真实状态分散在各模块自己的共享结构里。一个裸 `Latch` 只能叫醒进程，不能回答“该检查哪个条件、谁在等、signal 应该叫醒一个还是所有人”。如果每个模块都自己维护一套 wait list，又会重复踩 missed wakeup、ERROR cleanup、spurious wakeup、跨进程指针安全等坑。

PostgreSQL 的解法是把问题拆成两层：

```text
谓词状态:
  由调用方维护，例如 BufferDesc.state、CheckpointerShmem 计数器、
  ReplicationSlot.active_proc。

等待协议:
  ConditionVariable 只维护“哪些 PGPROC 需要在谓词可能变化时被唤醒”，
  真正 sleep 仍然落到 MyLatch / WaitLatch()。
```

学完后应能独立判断：

```text
为什么 ConditionVariable 不保存 predicate value；
为什么等待方必须写成 prepare -> while (!predicate) sleep -> cancel；
为什么 ConditionVariableSleep() 返回不代表条件已经满足；
为什么 signal / broadcast 只是“让 waiter 重新检查”，不是转交资源所有权；
为什么 PGPROC.cvWaitLink 让 CV 可以放进 DSM，而不是保存进程私有指针；
为什么 ConditionVariableCancelSleep() 是 exit / abort cleanup 的一部分；
为什么 Broadcast() 要用 sentinel 避免和立即 requeue 的 waiter 无限缠绕；
为什么 pg_stat_activity 看到的是 WAIT_EVENT_BUFFER_IO / CHECKPOINT_DONE 等业务等待点，
而不是一个泛泛的 ConditionVariable。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节讲 `Latch` 时，我们得到一个基础模型：

```text
业务状态先改变；
SetLatch() 只是唤醒 owner；
owner 醒来后重新检查自己关心的条件。
```

这个模型适合后台主循环：

```text
bgwriter / walwriter / checkpointer 主循环:
  ResetLatch();
  检查 signal flag、共享状态、周期任务；
  没事就 WaitLatch();
```

但还有一类更结构化的等待：

```text
多个 backend 可能同时等待同一个共享对象状态变化；
状态变化后可能只需要叫醒一个 waiter，也可能要叫醒所有 waiter；
等待方需要一个 wait_event 名字出现在 pg_stat_activity；
ERROR / proc exit 时必须从等待队列摘掉自己；
等待对象可能在 DSM 中，不能保存 backend 私有地址。
```

这就是 `ConditionVariable` 的位置。

它不是 `LWLock` 的替代品：

```text
LWLock:
  保护共享状态的互斥访问，等待目标是“锁状态允许获取”。

ConditionVariable:
  不保护谓词本身，等待目标是“某个调用方定义的 predicate 可能变化了”。
```

它也不是 `Latch` 的替代品：

```text
Latch:
  per-process wakeup bit，任何唤醒原因都可能让等待者醒来。

ConditionVariable:
  shared wait list，把等待同一个 predicate 的 PGPROC 组织起来，
  最终仍然通过 SetLatch(&proc->procLatch) 唤醒对应进程。
```

所以本节连接了前三节同步原语：

```text
SpinLock:
  CV 内部用它保护 wait list 的几条指令级操作。

Latch:
  CV 唤醒等待者时设置等待者的 procLatch。

ConditionVariable:
  在调用方 predicate 和进程 latch 之间增加“谁在等这个 predicate”的共享队列。
```

下一节 `Barrier` 会继续往上抽象：它把“一组进程等待同一阶段推进”建立在 `ConditionVariable` 之上。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ConditionVariable 是一个不保存 predicate 的共享等待队列；
等待方先把自己的 PGPROC.cvWaitLink 挂到 cv->wakeup，
再在调用方锁保护下检查 predicate；
如果 predicate 仍不满足，就 WaitLatch(MyLatch)；
signal / broadcast 从 cv->wakeup 移出 PGPROC 并 SetLatch(&proc->procLatch)；
等待方醒来后只知道“应该重查 predicate”，最后必须 CancelSleep() 摘队。
```

这个模型有三个核心不变量。

第一，谓词不在 CV 里。

```text
ConditionVariable 不知道:
  BM_IO_IN_PROGRESS 是什么；
  ckpt_started / ckpt_done 怎么比较；
  replication slot active_proc 何时有效；
  parallel bitmap scan 是否初始化完成。

ConditionVariable 只知道:
  哪些 PGPROC 当前准备在这个 CV 上被唤醒。
```

因此 `ConditionVariableSleep()` 不能返回“条件满足”。它最多返回：

```text
有人 signal / broadcast 过我；
或者 timeout 到了；
或者 latch 因别的原因被 set 过；
或者中断处理导致当前 CV sleep target 变化。
```

第二，等待必须是谓词循环。

正确形态是：

```c
ConditionVariablePrepareToSleep(cv);
while (!predicate_is_true())
    ConditionVariableSleep(cv, wait_event_info);
ConditionVariableCancelSleep();
```

这段代码的关键不是 `Sleep()`，而是 `while (!predicate)`。

```text
被唤醒:
  只表示 predicate 可能变化。

真正退出:
  必须由调用方重新读取共享状态后判断。
```

第三，prepare 必须早于可能错过的状态检查。

错误形态是：

```text
检查 predicate 不满足；
准备 sleep；
真正 sleep。
```

如果状态在“检查不满足”和“准备 sleep”之间变成满足并完成 signal，等待者可能挂到队列后再也没人叫醒。正确顺序是：

```text
先把自己挂到 CV wait list；
再检查 predicate；
如果已经满足，CancelSleep()；
如果仍不满足，Sleep()。
```

这个顺序会带来一个看似奇怪的结果：

```text
可能最终根本不睡，但也要先 prepare。
```

这不是浪费，而是关闭 missed wakeup 窗口的代价。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/condition_variable.h` | `ConditionVariable` 的字段、API contract、spurious wakeup 规则。 |
| 2 | `src/backend/storage/lmgr/condition_variable.c` | `PrepareToSleep()`、`Sleep()`、`CancelSleep()`、`Signal()`、`Broadcast()` 的队列协议。 |
| 3 | `src/include/storage/proc.h` | `PGPROC.cvWaitLink`，等待队列不保存普通指针的原因。 |
| 4 | `src/include/storage/proclist.h` | 以 `ProcNumber` / intrusive link 组织 shared memory 等待队列。 |
| 5 | `src/backend/storage/buffer/bufmgr.c` | `WaitIO()` / `TerminateBufferIO()`：buffer I/O 等待的主 walkthrough。 |
| 6 | `src/backend/storage/buffer/buf_init.c`、`src/include/storage/buf_internals.h` | `BufferIOCVArray` 的 shared memory 初始化和按 buffer id 映射。 |
| 7 | `src/backend/postmaster/checkpointer.c` | `RequestCheckpoint()` 等待 checkpoint start / done 的计数器谓词。 |
| 8 | `src/backend/replication/slot.c` | replication slot active state 等待与 invalidation 路径。 |
| 9 | `src/backend/replication/walsender.c` | 只 prepare CV、实际用 `WaitEventSetWait()` 等 socket 的混合等待。 |
| 10 | `src/backend/utils/activity/wait_event_names.txt` | `BUFFER_IO`、`CHECKPOINT_START`、`CHECKPOINT_DONE`、`REPLICATION_SLOT_DROP` 等可观测等待点。 |

推荐阅读顺序不是从 `condition_variable.c` 顶部一路读到底，而是先建立 contract：

```text
header 中的用法约束
  -> ConditionVariable 结构
  -> WaitIO() 这种真实谓词循环
  -> Sleep() 内部如何区分 signal 与 spurious latch wakeup
  -> Signal/Broadcast 如何只移出队列并 SetLatch()
  -> cleanup 为什么挂在 ProcKill / Abort 路径上
```

本节不展开 `proclist` 的完整实现，只需要记住一点：

```text
CV wait list 存的是 PGPROC 的 proc number 和 PGPROC 内部 link，
不是 malloc 出来的 waiter node，也不是 backend-local 指针。
```

这让 `ConditionVariable` 可以安全放在 shared memory 和 DSM 中。

## 4. 关键数据结构与状态

### 4.1 `ConditionVariable`

`src/include/storage/condition_variable.h` 中的结构很小：

```c
typedef struct
{
    slock_t      mutex;
    proclist_head wakeup;
} ConditionVariable;
```

两个字段分别对应两件事：

```text
mutex:
  一个 spinlock，只保护 wakeup list 的插入、删除、检查。
  它不保护调用方 predicate。

wakeup:
  一个 PGPROC 队列，保存“当前准备在这个 CV 上被唤醒的进程”。
```

因此不要把 `cv->mutex` 理解成条件变量传统意义上的“associated mutex”。在 PostgreSQL 这里：

```text
调用方 predicate 的锁:
  由调用模块自己选择，例如 BufferDesc header spinlock、
  CheckpointerShmem->ckpt_lck、ReplicationSlot mutex / LWLock。

CV 内部 mutex:
  只保证 wait list 不被并发破坏。
```

这条边界非常重要。`ConditionVariable` 本身不能让 predicate 读取变得正确；它只负责让等待者不丢唤醒。

### 4.2 `ConditionVariableMinimallyPadded`

同一个头文件里还有：

```c
#define CV_MINIMAL_SIZE (sizeof(ConditionVariable) <= 16 ? 16 : 32)
typedef union ConditionVariableMinimallyPadded
{
    ConditionVariable cv;
    char        pad[CV_MINIMAL_SIZE];
} ConditionVariableMinimallyPadded;
```

这个 padding 的用处在 buffer I/O 路径里最明显。PostgreSQL 为每个 shared buffer 准备一个 CV：

```text
BufferIOCVArray:
  NBuffers 个 ConditionVariableMinimallyPadded。
```

如果很多 buffer 的 CV 紧密挤在同一 cache line，不同 buffer 的 I/O wait / wake 会互相制造 cache line bouncing。padding 不能消除共享写，但能降低数组型 CV 在热点路径里的伪共享风险。

### 4.3 `PGPROC.cvWaitLink`

`src/include/storage/proc.h` 中：

```c
proclist_node cvWaitLink;
```

这是每个 backend 在 shared memory 中的 CV 等待队列节点。它带来两个限制：

```text
每个 PGPROC 同一时间只能有一个 active CV wait link；
如果要准备等待另一个 CV，必须先取消之前的 sleep target。
```

`condition_variable.c` 用一个 backend-local 静态变量记录当前目标：

```c
static ConditionVariable *cv_sleep_target = NULL;
```

这形成了一个配对关系：

```text
cv_sleep_target:
  backend-local，说明我当前准备在哪个 CV 上睡。

MyProc->cvWaitLink:
  shared memory link，说明我在那个 CV 的 wakeup list 里的位置。
```

为什么不为每次等待分配一个 waiter node？

```text
无需内存分配:
  signal / broadcast 可能出现在不适合复杂分配的路径。

DSM safe:
  CV 结构中不保存 backend 私有地址。

cleanup 简单:
  proc exit / abort 只需要调用 ConditionVariableCancelSleep()，
  就能根据 cv_sleep_target 找到需要摘掉的队列。
```

代价是一个 backend 不能同时把自己挂在多个 CV 上。`walsender.c` 的混合等待展示了这一点：它会根据当前等待目标选择一个 CV `PrepareToSleep()`，实际 socket 等待仍走 `WaitEventSetWait()`。

### 4.4 `MyLatch` 与 `procLatch`

CV 不自己实现 OS sleep。等待方最终还是睡在 `MyLatch` 上：

```text
ConditionVariableSleep()
  -> WaitLatch(MyLatch, WL_LATCH_SET | WL_EXIT_ON_PM_DEATH, ..., wait_event_info)
```

唤醒方也不是直接操作 semaphore：

```text
ConditionVariableSignal()
  -> 从 cv->wakeup pop 一个 PGPROC
  -> SetLatch(&proc->procLatch)

ConditionVariableBroadcast()
  -> 逐个 pop PGPROC
  -> SetLatch(&proc->procLatch)
```

这让 CV 继承了上一节 `Latch` 的性质：

```text
signal / broadcast 是 sticky wakeup；
被唤醒者需要 ResetLatch()；
postmaster death 和 wait event reporting 统一走 WaitLatch()；
其它 SetLatch(MyLatch) 也可能造成 spurious wakeup。
```

### 4.5 调用方 predicate

本节最容易被忽略的状态，是“不在 CV 里”的状态。

几个典型 predicate：

| 使用场景 | CV 所在位置 | predicate 所在位置 | 退出条件 |
| --- | --- | --- | --- |
| Buffer I/O | `BufferIOCVArray[buf_id]` | `BufferDesc.state` | `!(state & BM_IO_IN_PROGRESS)` |
| Checkpoint start | `CheckpointerShmem->start_cv` | `CheckpointerShmem->ckpt_started` | `new_started != old_started` |
| Checkpoint done | `CheckpointerShmem->done_cv` | `CheckpointerShmem->ckpt_done` / `ckpt_failed` | `new_done - new_started >= 0` |
| Replication slot acquire / drop | `ReplicationSlot.active_cv` | `ReplicationSlot.active_proc` | slot 不再由别人持有 |
| AIO completion | `PgAioHandle.cv` | AIO handle state / generation | I/O 已完成或 handle recycled |

这一点决定了所有正确性分析都要同时看两把“锁”：

```text
CV mutex:
  我是否在 wait list 中？

predicate lock:
  我看到的共享状态是否可靠？
```

只看其中一边，都会误判。

## 5. 主流程源码 walkthrough：等待 buffer I/O 完成

本节用 `src/backend/storage/buffer/bufmgr.c` 的 `WaitIO()` 作为主流程。它是一个非常典型的 predicate wait：

```text
等待对象:
  某个 shared buffer。

谓词:
  BufferDesc.state 中 BM_IO_IN_PROGRESS 清掉。

等待队列:
  BufferDescriptorGetIOCV(buf) 对应的 ConditionVariable。

唤醒者:
  完成 I/O 的 backend / AIO completion 路径调用 TerminateBufferIO()。
```

### 5.1 每个 buffer 一个 CV

`src/backend/storage/buffer/buf_init.c` 请求 shared memory：

```text
Buffer IO Condition Variables:
  NBuffers * sizeof(ConditionVariableMinimallyPadded)
```

初始化 buffer pool 时：

```c
for (int i = 0; i < NBuffers; i++)
{
    BufferDesc *buf = GetBufferDescriptor(i);
    ...
    ConditionVariableInit(BufferDescriptorGetIOCV(buf));
}
```

映射函数在 `src/include/storage/buf_internals.h`：

```c
static inline ConditionVariable *
BufferDescriptorGetIOCV(const BufferDesc *bdesc)
{
    return &(BufferIOCVArray[bdesc->buf_id]).cv;
}
```

这说明 buffer I/O CV 的生命周期和 shared buffer pool 一样长：

```text
不是每次 I/O 临时创建；
不是每个 backend 私有；
不是挂在 BufferDesc 结构体内部；
而是用 buf_id 映射到一组 shared CV。
```

这样做的好处是：

```text
任何 backend 只要知道 BufferDesc，就能找到同一个等待队列；
CV 不随某次 I/O 开始 / 结束分配释放；
TerminateBufferIO() 不需要知道具体有哪些 backend 在等。
```

### 5.2 进入等待前：先准备入队

`WaitIO()` 开头：

```c
static void
WaitIO(BufferDesc *buf)
{
    ConditionVariable *cv = BufferDescriptorGetIOCV(buf);

    Assert(!pgaio_have_staged());

    ConditionVariablePrepareToSleep(cv);
    for (;;)
    {
        uint64      buf_state;
        PgAioWaitRef iow;
        ...
```

注意顺序：

```text
先 ConditionVariablePrepareToSleep(cv)；
再进入循环读取 BM_IO_IN_PROGRESS。
```

这正是防 missed wakeup 的关键。

假设相反，先读状态再入队：

```text
T1 waiter 读到 BM_IO_IN_PROGRESS=true；
T2 I/O owner 完成 I/O，清 BM_IO_IN_PROGRESS，Broadcast；
T3 waiter 才入队并 Sleep；
```

这个 waiter 就可能错过 broadcast。当前实现先入队，所以时间线变成：

```text
T1 waiter 入队；
T2 waiter 读 predicate；
T3 I/O owner 完成 I/O，Broadcast；
T4 waiter 只要还在队列中，就会被 SetLatch；
```

或者：

```text
T1 waiter 入队；
T2 I/O owner 完成 I/O，Broadcast，把 waiter 移出队列并 SetLatch；
T3 waiter 读取 predicate，看到 BM_IO_IN_PROGRESS 已清，直接退出；
T4 CancelSleep() 发现自己已经被 signal 过，清理 cv_sleep_target。
```

两种都安全。

### 5.3 在 predicate lock 下读取状态

循环中：

```c
buf_state = LockBufHdr(buf);
iow = buf->io_wref;
UnlockBufHdr(buf);

if (!(buf_state & BM_IO_IN_PROGRESS))
    break;
```

这里 `LockBufHdr()` 是 buffer header 的 spinlock，不是 CV 的 spinlock。

它保护的是：

```text
BufferDesc.state;
buf->io_wref;
BM_IO_IN_PROGRESS / BM_VALID / BM_DIRTY 等 flag 组合。
```

CV 的 `cv->mutex` 没有参与读取这些字段。也就是说：

```text
能不能退出等待:
  由 buffer header 状态决定。

能不能被正确唤醒:
  由 CV wait list 决定。
```

两个锁保护两个不同维度，不能混用。

### 5.4 同步 I/O 与 AIO 的分支

如果 buffer 上有有效 AIO wait reference：

```c
if (pgaio_wref_valid(&iow))
{
    pgaio_wref_wait(&iow);
    ConditionVariablePrepareToSleep(cv);
    continue;
}
```

这段注释很有信息量：AIO 子系统内部也会用 condition variable，可能把当前 backend 从 BufferDesc 的 CV 上移走。于是返回后重新 `PrepareToSleep(cv)`。

这里体现了 `PGPROC.cvWaitLink` 的限制：

```text
一个 backend 只有一个 cvWaitLink；
等待 AIO 内部 CV 可能覆盖之前准备的 BufferDesc CV；
回到 BufferDesc predicate loop 时必须重新挂回去。
```

如果没有 AIO wait reference，就等待 BufferDesc 的 CV：

```c
ConditionVariableSleep(cv, WAIT_EVENT_BUFFER_IO);
```

注意 wait event 是 `WAIT_EVENT_BUFFER_IO`。这就是为什么 `pg_stat_activity` 里能看到：

```text
wait_event_type = 'IPC'
wait_event      = 'BufferIO'
```

而不是一个没有业务含义的 “ConditionVariable”。

### 5.5 完成 I/O 时：先改变 predicate，再 broadcast

I/O 完成路径在 `TerminateBufferIO()`：

```c
buf_state = LockBufHdr(buf);

Assert(buf_state & BM_IO_IN_PROGRESS);
unset_flag_bits |= BM_IO_IN_PROGRESS;
...
buf_state = UnlockBufHdrExt(buf, buf_state,
                            set_flag_bits, unset_flag_bits,
                            refcount_change);

if (forget_owner)
    ResourceOwnerForgetBufferIO(...);

ConditionVariableBroadcast(BufferDescriptorGetIOCV(buf));
```

顺序是：

```text
1. 在 buffer header lock 下清 BM_IO_IN_PROGRESS；
2. 释放 buffer header lock；
3. 如有需要忘记 ResourceOwner 中的 buffer I/O ownership；
4. Broadcast buffer 对应的 CV。
```

这符合 CV 的基本 contract：

```text
先让 predicate 变成等待者能看到的新状态；
再唤醒等待者重查 predicate。
```

如果先 broadcast 再清 flag，等待者可能醒来后仍看到 `BM_IO_IN_PROGRESS=true`，再次入睡。虽然 spurious wakeup 允许这种情况，但它会增加延迟；更严重的是，如果之后清 flag 没有再广播，等待者可能等到别的唤醒才继续。

### 5.6 退出等待：必须 cancel

`WaitIO()` 末尾：

```c
ConditionVariableCancelSleep();
```

这一步有两个作用：

```text
如果还在 cv->wakeup:
  从 wait list 摘掉 MyProc->cvWaitLink。

如果已经被 signal / broadcast 移出:
  记录 signaled=true，清空 cv_sleep_target。
```

对调用者来说，不管是哪种情况，离开 predicate loop 后都不能继续占着 wait list 位置。

如果忘记 cancel，会留下两个问题：

```text
后续 signal / broadcast 可能错误唤醒一个已经不再等待该 predicate 的 backend；
同一个 backend 下次等待别的 CV 时，cv_sleep_target / cvWaitLink 状态会被迫修正，
但中间窗口会造成额外唤醒和诊断混乱。
```

所以 `CancelSleep()` 不是可选的“礼貌清理”，而是等待协议的一部分。

## 6. `ConditionVariableSleep()` 内部：为什么醒来还要重新入队

`ConditionVariableSleep()` 是 `ConditionVariableTimedSleep(cv, -1, wait_event_info)` 的薄封装。关键逻辑分三段。

### 6.1 没 prepare 过：先 prepare，然后立即返回

`ConditionVariableTimedSleep()` 开头：

```c
if (cv_sleep_target != cv)
{
    ConditionVariablePrepareToSleep(cv);
    return false;
}
```

这解释了头文件里那段性能建议：

```text
如果大概率第一次检查 predicate 就满足:
  可以不提前 PrepareToSleep()，避免队列操作。

如果大概率要等:
  提前 PrepareToSleep()，避免第一次 Sleep() 只做 prepare 后返回，
  让 predicate 多检查一次。
```

但有一个硬约束：

```text
如果显式调用 PrepareToSleep(cv)，必须在 Prepare 和 Sleep 之间检查 predicate。
```

否则你只是提前入队，没有利用它关闭 missed wakeup 窗口。

### 6.2 真正睡眠：WaitLatch(MyLatch)

如果已经准备好，函数会设置等待事件：

```text
无 timeout:
  WL_LATCH_SET | WL_EXIT_ON_PM_DEATH

有 timeout:
  WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH
```

然后循环：

```c
(void) WaitLatch(MyLatch, wait_events, cur_timeout, wait_event_info);
ResetLatch(MyLatch);
```

这里有几个边界：

```text
CV sleep 是 interruptible 的:
  WaitLatch() 返回后会 CHECK_FOR_INTERRUPTS()。

CV sleep 会响应 postmaster death:
  postmaster child 不会永远睡在一个内部 predicate 上。

CV sleep 的 wait event 由调用方传入:
  诊断时看到的是业务等待点。
```

这也是 CV 和 LWLock wait 的差异之一。`condition_variable.c` 文件头直接说明：

```text
Waits for condition variables can be interrupted, unlike LWLock waits.
```

也就是说 CV 更适合等待“外部状态推进”，而不是保护必须短时间完成的内部互斥。

### 6.3 被 signal 过：返回前重新入队

`WaitLatch()` 返回并 `ResetLatch()` 后，CV 会检查自己是否还在 wait list：

```c
SpinLockAcquire(&cv->mutex);
if (!proclist_contains(&cv->wakeup, MyProcNumber, cvWaitLink))
{
    done = true;
    proclist_push_tail(&cv->wakeup, MyProcNumber, cvWaitLink);
}
SpinLockRelease(&cv->mutex);
```

这一段是 PostgreSQL CV 最值得仔细理解的地方。

如果 `Signal()` / `Broadcast()` 唤醒了当前 backend，它们会先把这个 PGPROC 从 `cv->wakeup` 中移出，再 `SetLatch(&proc->procLatch)`。因此：

```text
醒来后发现自己不在 wait list:
  说明自己被这个 CV signal / broadcast 过。
```

但它马上又把自己 push 回队尾。为什么？

因为 `ConditionVariableSleep()` 返回给调用者后，调用者还要重新检查 predicate：

```c
while (!predicate_is_true())
    ConditionVariableSleep(cv, wait_event_info);
```

在这段“返回调用者 -> 调用者读取 predicate -> 可能再次 Sleep”的窗口里，predicate 可能又变化一次并 broadcast。如果等待者不先重新入队，就可能错过那次变化。

所以 CV 的策略是：

```text
只要当前等待循环还没由调用方 CancelSleep() 结束，
Sleep() 每次被 signal 返回前都重新把自己挂回 wait list。
```

这让等待方始终处于两种安全状态之一：

```text
正在检查 predicate:
  仍然在 CV wait list 中，可以接收下一次 signal。

正在 WaitLatch():
  也在 CV wait list 中，可以接收下一次 signal。
```

真正退出等待循环时，才由 `ConditionVariableCancelSleep()` 摘队。

### 6.4 spurious wakeup 的处理

如果 latch 是因为别的原因被 set，比如 signal handler、socket wait 组合、其它内部通知，当前 PGPROC 仍在 `cv->wakeup` 中：

```text
proclist_contains(...) == true
```

这种情况下 CV 会继续等待，而不是马上返回，尽量避免“明显的 spurious return”。

但它仍然承认 spurious wakeup：

```text
CHECK_FOR_INTERRUPTS() 可能运行代码；
中断处理逻辑可能等待另一个 CV；
cv_sleep_target 可能变化；
timeout 可能到达；
其它 latch set 也可能使 WaitLatch() 返回。
```

所以 API contract 不能承诺“Sleep 返回一定是 signal”。它只能要求调用方写谓词循环。

## 7. `Signal()` 与 `Broadcast()`：唤醒一个，还是唤醒一批

### 7.1 `ConditionVariableSignal()`

`Signal()` 很短：

```c
PGPROC *proc = NULL;

SpinLockAcquire(&cv->mutex);
if (!proclist_is_empty(&cv->wakeup))
    proc = proclist_pop_head_node(&cv->wakeup, cvWaitLink);
SpinLockRelease(&cv->mutex);

if (proc != NULL)
    SetLatch(&proc->procLatch);
```

它只做三件事：

```text
从 wait list 头部拿出一个 PGPROC；
释放 CV spinlock；
SetLatch() 唤醒该进程。
```

它不做这些事：

```text
不检查 predicate；
不保证被唤醒者会退出等待；
不转交任何锁或资源；
不返回“是否真的让某个业务动作完成”。
```

文件注释还提醒，不要轻易让 `Signal()` 返回一个“是否唤醒成功”的 flag。因为 list 里可能存在 broadcast sentinel，或者被唤醒者醒来后又马上 requeue，这个布尔值很容易被误用成业务语义。

适合 `Signal()` 的场景通常是：

```text
状态变化一次最多只需要一个等待者继续推进；
多个等待者都会抢同一个后续资源，叫醒全部只会制造惊群；
谓词循环能容忍被唤醒者发现条件仍不满足。
```

并行索引构建里 worker 完成后 `ConditionVariableSignal(&workersdonecv)` 就是这类模式：leader 只需要醒来重查“还有多少 worker 没结束”。

### 7.2 `ConditionVariableBroadcast()`

`Broadcast()` 要复杂得多，因为它要保证：

```text
调用时已经在 wait list 里的 waiter 都会被唤醒；
但 awakened waiter 可能立刻重新入队；
不能因为 waiter requeue 而无限 pop；
也不能长时间持有 CV spinlock 做 SetLatch()。
```

因此它用了一个 sentinel 技巧。

大致流程：

```text
如果当前 backend 已经在别的 CV 上 prepared:
  CancelSleep()，释放自己的 cvWaitLink。

获取 cv->mutex:
  如果队列为空，结束；
  如果只有一个 entry，pop 它；
  如果还有更多 entry，把自己的 cvWaitLink 插到队尾作为 sentinel。

释放 cv->mutex:
  SetLatch(first waiter)。

while sentinel 仍在队列中:
  pop 队头；
  检查 sentinel 是否仍存在；
  释放 lock；
  如果 pop 的不是自己 sentinel，就 SetLatch()。
```

sentinel 的意义是：

```text
Broadcast 只承诺唤醒“调用开始时已经在队列里的那些 waiter”。
新加入或醒来后重新加入的 waiter 排在 sentinel 后面，不属于这一轮必须唤醒的范围。
```

如果另一个 concurrent `Signal()` 把 sentinel 移走怎么办？

源码注释的处理很务实：

```text
因为 CV waiters 的插入和移出保持顺序；
如果 sentinel 被别人移走，说明 sentinel 前面的 waiter 都已经被处理；
当前 Broadcast 最多额外唤醒一个后来的 waiter；
额外唤醒比丢唤醒更可接受。
```

这正是 CV 设计中反复出现的取舍：

```text
允许少量 spurious wakeup；
不允许 missed wakeup。
```

### 7.3 为什么 SetLatch() 放在释放 CV spinlock 之后

`Signal()` / `Broadcast()` 都是先从队列移出 PGPROC，再释放 `cv->mutex`，最后 `SetLatch()`。

这样做有两个原因：

```text
spinlock 持有区必须极短；
SetLatch() 可能触达平台 wakeup 原语，不应该在 CV spinlock 内执行。
```

队列语义仍然安全，因为：

```text
从队列移出 PGPROC 就已经发布了“你被 signal 了”这个事实；
等待者醒来后会通过“不在 wait list”判断自己被 signal；
SetLatch() 只是让它尽快从 WaitLatch() 返回。
```

如果 `SetLatch()` 发生在 waiter 尚未真正进入 `WaitLatch()` 前，也没问题。上一节讲过 latch 是 sticky 的：`procLatch.is_set` 会保留这次唤醒。

## 8. 对照 walkthrough：checkpoint 与 replication slot

Buffer I/O 说明了 per-object CV。接下来用两个对照场景说明 CV 的通用性：同一个等待协议，predicate 完全不同。

### 8.1 Checkpoint：等待计数器推进

`src/backend/postmaster/checkpointer.c` 的 `RequestCheckpoint()` 如果带 `CHECKPOINT_WAIT`，会等待两件事：

```text
新 checkpoint 开始；
该 checkpoint 完成。
```

共享状态在 `CheckpointerShmem`：

```text
ckpt_started:
  checkpoint 启动计数。

ckpt_done:
  checkpoint 完成计数。

ckpt_failed:
  checkpoint 失败计数。

start_cv / done_cv:
  等待 start / done 计数变化的 CV。
```

等待 start：

```c
ConditionVariablePrepareToSleep(&CheckpointerShmem->start_cv);
for (;;)
{
    SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    new_started = CheckpointerShmem->ckpt_started;
    SpinLockRelease(&CheckpointerShmem->ckpt_lck);

    if (new_started != old_started)
        break;

    ConditionVariableSleep(&CheckpointerShmem->start_cv,
                           WAIT_EVENT_CHECKPOINT_START);
}
ConditionVariableCancelSleep();
```

等待 done：

```c
ConditionVariablePrepareToSleep(&CheckpointerShmem->done_cv);
for (;;)
{
    SpinLockAcquire(&CheckpointerShmem->ckpt_lck);
    new_done = CheckpointerShmem->ckpt_done;
    new_failed = CheckpointerShmem->ckpt_failed;
    SpinLockRelease(&CheckpointerShmem->ckpt_lck);

    if (new_done - new_started >= 0)
        break;

    ConditionVariableSleep(&CheckpointerShmem->done_cv,
                           WAIT_EVENT_CHECKPOINT_DONE);
}
ConditionVariableCancelSleep();
```

这里 predicate 不再是一个 flag，而是一组计数器的比较：

```text
start predicate:
  ckpt_started 从 old_started 推进到新值。

done predicate:
  ckpt_done 在 modulo sense 上追到 new_started。

error predicate:
  ckpt_failed 是否变化。
```

CV 仍然不理解这些语义。它只负责让等待者在 `start_cv` / `done_cv` 被 broadcast 后醒来重读计数器。

诊断时也能看到两个不同 wait event：

```text
CHECKPOINT_START:
  Waiting for a checkpoint to start.

CHECKPOINT_DONE:
  Waiting for a checkpoint to complete.
```

同一个 `ConditionVariableSleep()`，因为调用方传入不同 `wait_event_info`，在 `pg_stat_activity` 中呈现为不同业务等待。

### 8.2 Replication slot：等待 active owner 消失

`src/backend/replication/slot.c` 里，每个 slot 有 `active_cv`。等待者关心的 predicate 是：

```text
slot->active_proc 是否仍然指向某个使用该 slot 的 backend？
```

获取 slot 时，如果 slot 已被别的进程占用：

```c
if (!nowait)
    ConditionVariablePrepareToSleep(&s->active_cv);

SpinLockAcquire(&s->mutex);
if (s->active_proc == INVALID_PROC_NUMBER)
    s->active_proc = MyProcNumber;
active_proc = s->active_proc;
SpinLockRelease(&s->mutex);
...
if (active_proc != MyProcNumber)
{
    if (!nowait)
    {
        ConditionVariableSleep(&s->active_cv,
                               WAIT_EVENT_REPLICATION_SLOT_DROP);
        ConditionVariableCancelSleep();
        goto retry;
    }
    ...
}
else if (!nowait)
    ConditionVariableCancelSleep();
```

这里再次看到同样顺序：

```text
先 prepare active_cv；
再在 slot mutex 下检查 / 尝试设置 active_proc；
如果发现别人持有，则 sleep；
醒来后 goto retry，从头重新查找并获取 slot。
```

为什么醒来后不是直接拥有 slot？

```text
ConditionVariableSignal/Broadcast 没有转交 ownership；
slot 释放和另一个 backend 抢占之间可能有竞争；
slot 还可能被 invalidation、drop、sync 逻辑改变语义。
```

所以 replication slot 路径的返回语义非常保守：

```text
被 active_cv 唤醒:
  只说明 active_proc 可能变化；
  必须 goto retry，在正确锁保护下重新判断。
```

slot invalidation 路径也类似。它可能需要终止当前 owner，然后等待 slot 被释放：

```text
PrepareToSleep(&s->active_cv);
释放 ReplicationSlotControlLock；
向 owner 发 SIGTERM 或 recovery conflict；
ConditionVariableSleep(&s->active_cv, WAIT_EVENT_REPLICATION_SLOT_DROP);
重新获取 ReplicationSlotControlLock；
continue 从头判断。
```

这里的 CV 不仅是性能工具，也是 correctness 工具：

```text
它允许 invalidation 在释放全局锁后等待 owner 变化；
同时不会错过 owner 刚好释放 slot 的通知。
```

### 8.3 Walsender：只借用 CV wait list，不调用 CV sleep

`src/backend/replication/walsender.c` 的 `WalSndWait()` 是一个有趣旁路。walsender 要同时等待：

```text
socket readiness；
timeout；
postmaster death；
WAL flush / replay / confirmation 的共享状态变化。
```

它实际睡眠走 `WaitEventSetWait(FeBeWaitSet, ...)`，因为还要等 socket。但它会先：

```c
ConditionVariablePrepareToSleep(&WalSndCtl->wal_flush_cv);
```

或：

```c
ConditionVariablePrepareToSleep(&WalSndCtl->wal_replay_cv);
```

唤醒方 `WalSndWakeup()`：

```c
if (physical)
    ConditionVariableBroadcast(&WalSndCtl->wal_flush_cv);

if (logical)
    ConditionVariableBroadcast(&WalSndCtl->wal_replay_cv);
```

注释明确说明：

```text
walsender 只把自己加入 CV waitlist；
并不调用 ConditionVariableSleep()；
Broadcast() 最终 SetLatch()，
而 FeBeWaitSet 里也包含 latch，因此 WaitEventSetWait() 会返回。
```

这说明 `ConditionVariable` 的本质不是“睡眠函数”，而是：

```text
一个 shared wait list + SetLatch wakeup protocol。
```

当调用方有更复杂的等待集合时，可以只借用 wait list 部分，把实际 sleep 放到自己的 `WaitEventSet` 中。

## 9. 生命周期 / ownership / cleanup

### 9.1 CV 对象谁创建

不同模块用不同生命周期创建 CV：

| CV | 创建位置 | 生命周期 |
| --- | --- | --- |
| `BufferIOCVArray` | shared buffer 初始化 | 与 shared buffer pool 一样长 |
| `CheckpointerShmem->start_cv/done_cv` | checkpointer shared memory 初始化 | 与 postmaster shared memory 一样长 |
| `ReplicationSlot.active_cv` | replication slot 初始化 | 与 slot entry 一样长 |
| `PgAioHandle.cv` | AIO shared state 初始化 | 与 AIO handle pool 一样长 |
| parallel scan / parallel index build CV | DSM 或 parallel shared state 初始化 | 与 parallel operation 一样长 |

初始化统一调用：

```c
ConditionVariableInit(cv);
```

它只做：

```text
SpinLockInit(&cv->mutex);
proclist_init(&cv->wakeup);
```

没有 backend ownership，没有动态内存，没有 OS handle。

### 9.2 等待状态谁持有

等待状态跨两个位置：

```text
CV 对象:
  cv->wakeup 中有 MyProcNumber / cvWaitLink。

当前 backend:
  cv_sleep_target 指向这个 CV。
```

这个等待状态由调用 `ConditionVariablePrepareToSleep()` 的 backend 持有。它必须在离开等待循环时调用：

```c
ConditionVariableCancelSleep();
```

如果等待者被 signal 过，`CancelSleep()` 返回 `true`；如果只是自己退出，返回 `false`。大多数调用方不关心这个返回值，因为真正语义来自 predicate。

### 9.3 ERROR / abort 谁兜底

PostgreSQL 在多个退出路径调用 `ConditionVariableCancelSleep()`：

```text
ProcKill():
  普通 backend 退出时取消 pending CV sleep。

AuxiliaryProcKill():
  bgwriter、checkpointer 等 auxiliary process 退出时取消 pending CV sleep。

AbortTransaction() / xact cleanup:
  事务 abort 清理未完成的 CV sleep。

部分后台进程主循环:
  退出或异常路径显式 CancelSleep()。
```

`src/backend/storage/lmgr/proc.c` 中，释放 `PGPROC` 前会：

```c
LWLockReleaseAll();
WaitLSNCleanup();
ConditionVariableCancelSleep();
...
SwitchBackToLocalLatch();
...
DisownLatch(&proc->procLatch);
```

顺序含义是：

```text
先从 CV wait list 摘掉当前 PGPROC；
再放弃 shared latch / PGPROC ownership；
避免别的 backend 后续从 CV 队列拿到一个正在释放的 procLatch。
```

这就是为什么 `CancelSleep()` 必须能在“什么都没准备”的情况下安全返回。它的注释写得很直接：

```text
Do nothing if nothing is pending;
this allows this function to be called during transaction abort.
```

### 9.4 CV 自身谁释放

大多数核心 CV 位于固定 shared memory 中，不单独释放。DSM 场景下，CV 随 DSM segment / parallel shared state 生命周期一起释放。

释放 CV 前的责任在调用方：

```text
确保没有 backend 仍可能等待这个 CV；
或上层并行框架 / DSM detach 已经保证参与者退出；
或 ERROR cleanup 会先 CancelSleep()。
```

CV 不维护 refcount。它没有能力判断“这个 CV 对象是否还能被等待”。因此把 CV 放在短生命周期内存中时，必须由模块自己的 owner 协议保证安全。

## 10. 正确性机制层次

### 10.1 防 missed wakeup：prepare-before-check

本节最重要的不变量：

```text
可能睡眠的 backend，在检查 predicate 前，必须已经能被 signal / broadcast 找到。
```

对应代码形态：

```c
ConditionVariablePrepareToSleep(cv);
while (!predicate())
    ConditionVariableSleep(cv, wait_event_info);
ConditionVariableCancelSleep();
```

这个模式和 LWLock 上一节的“入队后再尝试获取”是同一类思想：

```text
先发布“如果状态变化请叫醒我”；
再检查状态是否已经变化。
```

差异是：

```text
LWLock 的 predicate 是锁 state，由 LWLock 自己理解；
ConditionVariable 的 predicate 由调用方理解，CV 只管理 waiter visibility。
```

### 10.2 防资源转交误解：wakeup != ownership

`Signal()` 从队列拿出一个 waiter，`Broadcast()` 拿出一批 waiter，但它们都不转交业务资源。

因此等待方必须假设：

```text
我醒来时，资源可能仍不可用；
别的 backend 可能抢先改变了 predicate；
唤醒可能来自 unrelated latch set；
我必须重新拿调用方锁并重读状态。
```

这正是所有调用方使用 `while` 而不是 `if` 的原因。

错误写法：

```c
if (!predicate())
    ConditionVariableSleep(cv, WAIT_EVENT_X);
/* assume predicate is true */
```

正确写法：

```c
while (!predicate())
    ConditionVariableSleep(cv, WAIT_EVENT_X);
```

### 10.3 防并发队列破坏：CV spinlock 只保护 list

`cv->mutex` 保护：

```text
proclist_push_tail();
proclist_pop_head_node();
proclist_delete();
proclist_contains();
sentinel 插入与检测。
```

它不保护：

```text
BufferDesc.state；
ReplicationSlot.active_proc；
CheckpointerShmem 计数器；
业务对象生命周期；
调用方错误码。
```

这避免了一个常见设计错误：

```text
把 ConditionVariable 当作“带锁条件变量”，以为拿了 cv->mutex 就能安全检查条件。
```

PostgreSQL 的 CV 更接近：

```text
一个为 shared memory / backend process 设计的 wait queue primitive。
```

### 10.4 防队列永久残留：CancelSleep cleanup

`ConditionVariableCancelSleep()` 的核心：

```c
if (proclist_contains(&cv->wakeup, MyProcNumber, cvWaitLink))
    proclist_delete(&cv->wakeup, MyProcNumber, cvWaitLink);
else
    signaled = true;

cv_sleep_target = NULL;
```

如果当前 PGPROC 还在队列中，它被摘掉。

如果不在队列中，说明它已经被 signal / broadcast 移出。这时不用再删，但必须清空 `cv_sleep_target`。

这个函数必须是 idempotent-like 的：

```text
没有 pending CV sleep:
  直接返回 false。

有 pending sleep:
  清理队列或确认已被 signal。
```

这让 abort / proc exit 可以不精确知道当前栈帧是否正在 CV wait。

### 10.5 防 broadcast 无限循环：sentinel

`Broadcast()` 最大的问题不是“如何叫醒所有人”，而是：

```text
被叫醒的人会立刻重新入队。
```

如果 naive 地写：

```text
while (!list_empty)
  pop waiter;
  SetLatch(waiter);
```

那么一个醒来后马上发现 predicate 仍不满足的 waiter 会重新进入队尾，broadcast 可能一直追着它跑。

sentinel 把“这一轮要唤醒的人”定界：

```text
sentinel 之前:
  broadcast 开始时已经在队列里的 waiter，应该被唤醒。

sentinel 之后:
  后来加入或重新加入的 waiter，不属于本轮必须唤醒范围。
```

这是一种很典型的并发工程取舍：

```text
不追求严格无 spurious；
追求有界完成和不丢入口时已存在的 waiter。
```

## 11. 错误路径 / 异常路径 / fallback

### 11.1 中断与 ERROR

`ConditionVariableTimedSleep()` 在 `WaitLatch()` 返回后会：

```c
CHECK_FOR_INTERRUPTS();
if (cv != cv_sleep_target)
    done = true;
```

这意味着等待期间可以响应 query cancel、shutdown 等中断。中断处理过程中如果等待了另一个 CV，当前 `cv_sleep_target` 可能变化，于是本次 sleep 选择 spuriously return。

这也是为什么上层必须有 cleanup：

```text
正常退出等待循环:
  ConditionVariableCancelSleep()。

ERROR / abort:
  transaction / proc cleanup 调用 ConditionVariableCancelSleep()。
```

### 11.2 timeout

`ConditionVariableTimedSleep()` 支持毫秒 timeout：

```text
返回 true:
  timeout expires。

返回 false:
  被 signal / broadcast 或其它原因唤醒。
```

内部用 `instr_time` 记录起始时间。发生 spurious wakeup 后，会重新计算剩余 timeout：

```text
cur_timeout = timeout - elapsed_ms;
```

这样 timeout 表示整个等待调用的总预算，而不是每次 spurious wakeup 后重新给满。

典型使用包括：

```text
WAL summarizer 等待 summary file；
recovery pause 等待恢复继续；
procsignal barrier 等待其它进程响应；
AIO / 测试模块中的有限等待。
```

诊断时要注意：

```text
TimedSleep 返回 timeout，不代表 predicate 永远不会满足；
它只说明这次等待预算耗尽，调用方需要决定重试、报错还是走 fallback。
```

### 11.3 postmaster death

CV sleep 使用 `WL_EXIT_ON_PM_DEATH`。这继承了 `WaitLatch()` 的安全策略：

```text
postmaster child 不能在内部 CV wait 中忽略 postmaster death；
postmaster 死亡时进程应按 PostgreSQL 的退出协议结束。
```

因此不要把 CV sleep 当成一个普通 OS condition variable wait。它嵌入了 PostgreSQL 进程生命周期语义。

### 11.4 nowait / no-sleep fallback

有些调用方在等待之前有 `nowait` 选项。例如 replication slot acquire：

```text
nowait=false:
  PrepareToSleep -> 检查 active_proc -> Sleep -> retry。

nowait=true:
  不挂 CV；
  发现 active_proc 被别人占用就 ERROR。
```

这说明 CV 是阻塞等待协议，不是 predicate 检查本身。调用方可以选择不等，但仍要在同一把业务锁下读取 predicate。

### 11.5 AIO 混合路径

Buffer I/O 的 `WaitIO()` 在看到 `PgAioWaitRef` 时，会转而等待 AIO 子系统：

```text
pgaio_wref_wait(&iow);
ConditionVariablePrepareToSleep(cv);
continue;
```

这是一个很好的异常路径例子：

```text
同一层业务 predicate 是 BM_IO_IN_PROGRESS；
等待实现可以是 AIO wait ref，也可以是 BufferDesc CV；
AIO 内部也可能使用 CV，因此回到原 predicate loop 前必须重新 prepare 原 CV。
```

从系统设计角度看，这说明：

```text
ConditionVariable 不是排他的等待机制；
它可以和 AIO、WaitEventSet、socket wait 组合，
只要调用方重新维护好“我当前在哪个 CV wait list 上”。
```

## 12. 成本、资源与跨模块传播

### 12.1 hot path 成本

一次 `ConditionVariablePrepareToSleep()` 的成本主要是：

```text
获取一个 spinlock；
把 MyProcNumber 通过 cvWaitLink push 到 proclist 尾部；
释放 spinlock。
```

一次 `ConditionVariableSignal()`：

```text
获取 spinlock；
pop 一个 PGPROC；
释放 spinlock；
SetLatch(procLatch)。
```

一次 `Broadcast()`：

```text
可能多次获取 / 释放 spinlock；
为每个 waiter SetLatch()；
用 sentinel 限定本轮范围。
```

所以选择 `Signal()` 还是 `Broadcast()` 不是风格问题，而是成本和语义问题：

```text
一个状态变化只可能释放一个等待者:
  Signal() 更合适。

一个状态变化可能让所有等待者的 predicate 都满足:
  Broadcast() 更合适。

不确定:
  倾向 Broadcast() 可以避免等待者遗留，但可能造成惊群。
```

Buffer I/O 完成用 `Broadcast()`，因为多个 backend 可能都在等同一个 buffer 的 I/O 完成；I/O 完成后所有等待者都可以继续重查。

worker done CV 常用 `Signal()`，因为 leader 只需要被叫醒检查计数，不需要唤醒所有等待者。

### 12.2 shared memory 与 DSM 安全

CV 使用 `proclist` 和 `ProcNumber`，而不是普通指针。这样它可以放在：

```text
main shared memory；
dynamic shared memory；
parallel operation shared state；
extension / test module 的 shared state。
```

这解释了头文件注释：

```text
do not use pointers internally,
so that they are safe to use within DSMs.
```

但这不表示 CV 对象生命周期自动安全。DSM 释放前仍要保证参与者已经不再等待，或上层 cleanup 会先 cancel。

### 12.3 wait event 传播

CV 等待时调用方传入 `wait_event_info`。因此同一个底层实现会传播成不同观测点：

```text
WAIT_EVENT_BUFFER_IO:
  Waiting for buffer I/O to complete.

WAIT_EVENT_CHECKPOINT_START:
  Waiting for a checkpoint to start.

WAIT_EVENT_CHECKPOINT_DONE:
  Waiting for a checkpoint to complete.

WAIT_EVENT_REPLICATION_SLOT_DROP:
  Waiting for a replication slot to become inactive so it can be dropped.

WAIT_EVENT_AIO_IO_COMPLETION:
  Waiting for another process to complete IO.
```

这是 PostgreSQL wait event 设计中很值得学习的一点：

```text
底层 primitive 不泄露为观测语义；
观测语义由调用点命名。
```

看到 `BufferIO` 时，诊断方向应该是 buffer I/O owner、storage latency、AIO / sync I/O、buffer state，而不是盯着 `condition_variable.c` 本身。

### 12.4 跨模块传播

ConditionVariable 作为基础设施，被多个上层模块复用：

```text
buffer manager:
  等 buffer I/O 完成。

checkpointer:
  等 checkpoint started / done 计数推进。

replication slot:
  等 active owner 释放 slot。

walsender:
  用 CV wait list 做 selective wakeup，再结合 socket WaitEventSet。

parallel executor / index build:
  等 worker 完成或共享扫描状态推进。

barrier:
  在下一节中，阶段同步会直接内嵌一个 ConditionVariable。
```

这个传播路径说明 CV 的抽象边界很窄：

```text
它没有定义任何业务状态；
正因为如此，它能被很多模块复用。
```

## 13. 观测与诊断入口

### 13.1 从 `pg_stat_activity` 看 CV 等待

常用查询：

```sql
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY wait_event_type, wait_event, pid;
```

你可能看到：

```text
IPC / BufferIO
IPC / CheckpointStart
IPC / CheckpointDone
IPC / ReplicationSlotDrop
IPC / BtreePage
IPC / ParallelBitmapScan
IPC / AioIoCompletion
```

这些很多都可能由 `ConditionVariableSleep()` 报告，但诊断入口应从 wait event 的业务名字开始。

例如：

```text
BufferIO:
  谁持有该 buffer 的 I/O ownership？
  是否底层存储慢？
  是否有大量 backend 等同一个 relation block？
  是否 AIO path / sync path 存在异常？

CheckpointDone:
  checkpoint 是否正在写大量 dirty buffers？
  checkpointer 日志是否显示 sync 慢？
  checkpoint_timeout / max_wal_size / checkpoint_completion_target 是否合理？

ReplicationSlotDrop:
  slot 是否被 walsender / logical decoding / subscription worker 使用？
  DROP_REPLICATION_SLOT WAIT 是否在等 active owner 退出？
```

### 13.2 用 gdb 观察队列与 predicate

调试 `BufferIO` 等待时，可以在等待 backend 上看：

```gdb
break WaitIO
break ConditionVariableSleep
break ConditionVariableBroadcast
```

关注状态：

```text
cv_sleep_target:
  当前 backend 准备等待哪个 CV。

MyProc->cvWaitLink:
  是否挂在某个 CV wait list。

buf->state:
  BM_IO_IN_PROGRESS 是否仍在。

buf->io_wref:
  是否走 AIO wait reference。
```

在 I/O owner 上可以断：

```gdb
break TerminateBufferIO
```

确认顺序：

```text
清 BM_IO_IN_PROGRESS；
ResourceOwnerForgetBufferIO；
ConditionVariableBroadcast(BufferDescriptorGetIOCV(buf))。
```

### 13.3 日志与 wait event 对齐

checkpoint 场景中，可以结合：

```sql
CHECKPOINT;
SELECT pid, backend_type, wait_event_type, wait_event
FROM pg_stat_activity
WHERE wait_event IN ('CheckpointStart', 'CheckpointDone');
```

以及 `log_checkpoints` 输出观察：

```text
等待 CheckpointStart:
  checkpoint request 已发出，但 checkpointer 尚未推进 ckpt_started。

等待 CheckpointDone:
  checkpoint 已开始，但 ckpt_done 尚未追上该轮 started counter。
```

源码对应：

```text
RequestCheckpoint()
  -> start_cv predicate: ckpt_started != old_started
  -> done_cv predicate: ckpt_done - new_started >= 0
```

### 13.4 不能直接从 SQL 看到的东西

SQL 看不到：

```text
某个 CV wait list 里具体有哪些 PGPROC；
某次 wakeup 是 Signal 还是 Broadcast；
某次 WaitLatch 返回是否来自 spurious latch set；
某个 waiter 是否刚被 signal 后又重新入队。
```

这些只能通过：

```text
gdb；
tracepoint / probe；
临时日志；
perf + callgraph；
源码断点；
wait event 与业务状态交叉推断。
```

所以排查 CV 等待时，不要试图从 `pg_stat_activity` 直接还原 wait list。它只能告诉你“当前栈正在等什么业务事件”。

## 14. 常见误区

### 14.1 “ConditionVariable 保存了条件”

错误。CV 不保存条件，只保存 waiters。

真正条件在调用方共享状态里，并由调用方锁保护。

### 14.2 “Sleep 返回就说明条件满足”

错误。Sleep 返回只说明应该重查条件，甚至可能只是 spurious wakeup。

正确心智模型：

```text
Sleep 返回:
  请检查 predicate。

predicate 为真:
  可以退出等待。
```

### 14.3 “Signal 会把资源给某个 waiter”

错误。Signal 只 pop 一个 waiter 并 SetLatch。资源竞争仍由调用方锁和 predicate 决定。

### 14.4 “CV mutex 可以保护业务状态”

错误。`cv->mutex` 只保护 `cv->wakeup`。业务状态要用业务模块自己的 spinlock、LWLock、atomic 或其它可见性机制。

### 14.5 “只要有 latch，就不需要 CV”

裸 latch 缺少“谁在等这个 predicate”的共享队列。

如果只有一个固定进程等待某个事件，latch 可能足够；如果多个 backend 会等待同一共享对象状态变化，就需要 CV 或更高层等待结构。

### 14.6 “spurious wakeup 是 bug”

不是。PostgreSQL CV 明确允许 spurious wakeup，并用 predicate loop 吸收它。

真正的 bug 是：

```text
用 if 代替 while；
醒来后不重查 predicate；
退出循环忘记 CancelSleep()；
检查 predicate 前没有先 prepare，从而打开 missed wakeup 窗口。
```

### 14.7 “Broadcast 必须唤醒之后加入的 waiter”

不是。`Broadcast()` 保证唤醒调用开始时已经在队列里的 waiters。调用期间新加入或重新加入的 waiters 通常不属于这一轮。

这正是 sentinel 的意义：定义本轮边界。

### 14.8 “看到 ReplicationSlotDrop 就一定是 DROP SLOT”

不一定。`WAIT_EVENT_REPLICATION_SLOT_DROP` 的描述是等待 slot 变 inactive，以便 drop；但相关路径还可能出现在 invalidation / conflict handling 中。要结合 backend type、query、slot 状态和日志判断。

## 15. 课堂实验

### 实验 1：源码走读 `WaitIO()` 的 missed wakeup 防线

目标：解释为什么 `ConditionVariablePrepareToSleep(cv)` 必须在读取 `BM_IO_IN_PROGRESS` 前。

步骤：

```text
1. 打开 src/backend/storage/buffer/bufmgr.c。
2. 找到 WaitIO()。
3. 标出三个状态点：
   - ConditionVariablePrepareToSleep(cv)
   - LockBufHdr(buf) 读取 BM_IO_IN_PROGRESS
   - ConditionVariableSleep(cv, WAIT_EVENT_BUFFER_IO)
4. 写出一个错误顺序的时间线：
   先读 flag，后入队，I/O owner 在中间 Broadcast。
5. 再写出当前顺序为什么不会丢唤醒。
```

验收问题：

```text
如果 WaitIO() 在 PrepareToSleep 之后发现 BM_IO_IN_PROGRESS 已经清掉，
为什么仍然必须调用 ConditionVariableCancelSleep()？
```

### 实验 2：观察 checkpoint wait event

目标：把 `CHECKPOINT_WAIT` 的 SQL 可见等待和 `start_cv` / `done_cv` 源码谓词对应起来。

步骤：

```sql
-- session A
CHECKPOINT;

-- session B
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event IN ('CheckpointStart', 'CheckpointDone');
```

在负载较轻的系统中等待可能很短。可以结合大写入负载、较慢存储或 gdb 断点放大窗口：

```gdb
break RequestCheckpoint
break ConditionVariableSleep
```

源码对照：

```text
WAIT_EVENT_CHECKPOINT_START:
  ckpt_started != old_started。

WAIT_EVENT_CHECKPOINT_DONE:
  ckpt_done - new_started >= 0。
```

验收问题：

```text
为什么 done wait 不能只等“某个 checkpoint done 事件”，
而必须比较 ckpt_done 和本次 new_started？
```

### 实验 3：用 gdb 看 CV signal 后重新入队

目标：理解 `ConditionVariableSleep()` 返回前为什么重新 `proclist_push_tail()`。

步骤：

```gdb
break ConditionVariableTimedSleep
break ConditionVariableSignal
break ConditionVariableBroadcast
```

挑一个容易触发的 CV 使用点，例如测试模块、checkpoint 或 buffer I/O。观察：

```text
1. waiter prepare 后进入 cv->wakeup。
2. signal / broadcast pop 出 waiter。
3. waiter WaitLatch 返回后发现自己不在 list。
4. waiter 重新 push_tail。
5. 调用方检查 predicate。
6. 调用方最终 CancelSleep。
```

验收问题：

```text
如果第 4 步不重新入队，predicate 检查期间发生第二次状态变化，
为什么可能丢掉下一次唤醒？
```

### 实验 4：阅读 walsender 的混合等待

目标：理解 CV wait list 和实际 sleep 点可以分离。

步骤：

```text
1. 打开 src/backend/replication/walsender.c。
2. 找到 WalSndWait()。
3. 标出 ConditionVariablePrepareToSleep(&WalSndCtl->wal_flush_cv)
   或 wal_replay_cv 的分支。
4. 标出 WaitEventSetWait(FeBeWaitSet, ...)。
5. 找到 WalSndWakeup() 里的 ConditionVariableBroadcast()。
```

验收问题：

```text
为什么 walsender 不直接调用 ConditionVariableSleep()？
为什么 Broadcast() 仍然能让 WaitEventSetWait() 返回？
```

### 实验 5：比较 Signal 与 Broadcast 的适用场景

目标：从源码调用点判断唤醒一个还是唤醒全部。

步骤：

```bash
rg -n "ConditionVariableSignal|ConditionVariableBroadcast" /home/highgo/postgres/src/backend /home/highgo/postgres/src/include
```

选三个调用点，回答：

```text
这个 predicate 改变后，理论上有几个 waiter 能继续？
叫醒全部是否会造成惊群？
叫醒一个是否可能遗漏应该继续的 waiter？
调用方是否有 while predicate loop 吸收 spurious wakeup？
```

推荐对比：

```text
TerminateBufferIO() -> Broadcast:
  I/O 完成后所有等待同一 buffer 的 backend 都可能继续。

parallel index worker done -> Signal:
  leader 醒来重查 worker done count 即可。

WalSndWakeup() -> Broadcast:
  WAL flush/replay 位置推进后，多数 walsender 都需要重查。
```

## 16. 讨论题

1. 为什么 PostgreSQL 的 `ConditionVariable` 不把 predicate 和 mutex 绑定在一起，而是让调用方自己维护业务锁？

2. `ConditionVariableSleep()` 被 signal 后返回前重新入队，这看起来会让 wait list 中保留“正在检查 predicate 的进程”。这个设计解决了什么 race？它的代价是什么？

3. `Broadcast()` 为什么不简单循环到 `cv->wakeup` 为空？sentinel 定义的“本轮广播边界”对性能和正确性分别有什么影响？

4. 如果你在一个新共享结构上等待状态变化，什么时候应该用 `Latch`，什么时候应该用 `ConditionVariable`，什么时候应该用 `LWLock` 或更高层锁？

5. 为什么 `pg_stat_activity.wait_event='BufferIO'` 不能直接说明问题在 `condition_variable.c`？你会继续查看哪些业务状态？

6. 如果某个 CV 等待路径忘记 `ConditionVariableCancelSleep()`，最可能出现哪些症状？为什么这类 bug 不一定马上崩溃？

7. replication slot 等待被唤醒后为什么要 `goto retry`，而不是直接认为 slot 已经可用？

8. `ConditionVariableTimedSleep()` 允许 timeout，但核心用法仍推荐 predicate loop。timeout 在正确性中应该扮演主路径还是 fallback？为什么？

## 17. 本节小结

本节的核心规律：

```text
ConditionVariable 不保存条件；
它保存“哪些 PGPROC 需要在条件可能变化时被叫醒”。
```

因此 PostgreSQL 的 CV 正确使用必须同时满足四件事：

```text
1. 调用方用自己的锁维护 predicate。
2. 等待方先 PrepareToSleep，再检查 predicate。
3. Sleep 返回后只重查 predicate，不假设资源已获得。
4. 退出等待循环或异常退出时 CancelSleep。
```

从源码看，`ConditionVariable` 把几个底层机制组合得很窄：

```text
SpinLock:
  保护 cv->wakeup 的极短 list 操作。

PGPROC.cvWaitLink:
  让 wait list 在 shared memory / DSM 中安全表达 backend 身份。

Latch:
  让 signal / broadcast 最终变成进程级可中断唤醒。

wait_event_info:
  让同一个底层等待协议在 pg_stat_activity 中呈现为业务等待点。
```

Buffer I/O、checkpoint、replication slot 的共同点不是业务相似，而是并发形态相同：

```text
状态由某个模块推进；
多个 backend 可能等待状态变化；
唤醒只表示“该重新检查了”；
真正语义由 predicate loop 决定。
```

这是本节可以迁移出去的系统规律：

```text
把“条件是否满足”和“等待者如何被唤醒”分离；
让状态检查留在业务锁下，
让唤醒协议只负责不丢等待者。
```

下一节 `Barrier` 会在这个基础上再加一层：当等待目标不再是单个 predicate，而是一组参与者共同推进 phase 时，PostgreSQL 如何用 `ConditionVariable`、计数器和 generation 组织阶段同步。
