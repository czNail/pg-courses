# PostgreSQL 同步原语选择与组合边界

## 课程定位

前置知识：已经理解 PostgreSQL 中 `SpinLock`、`LWLock`、`Latch`、`ConditionVariable` 和 `Barrier` 各自解决的问题，也已经看过它们在 shared memory、`PGPROC`、wait event 和 ERROR cleanup 中的基本形态。

本节唯一主问题：

```text
面对一个新的共享状态等待点，如何判断应该使用 spinlock、LWLock、latch、
condition variable 还是 barrier，PostgreSQL 为什么经常把它们分层组合成
“短临界区修改状态、长等待交给 latch/CV、阶段推进交给 barrier”的模式？
```

核心矛盾：内核代码既要保证共享状态变化对其它 backend 可见、可等待、可诊断，又不能把所有事情塞进一把大锁里。

```text
只用 spinlock:
  状态修改很快，但不能睡眠、不能做 I/O、不能执行复杂逻辑。

只用 LWLock:
  可以保护结构化共享状态，但把长等待放在锁内会扩大阻塞面。

只用 latch:
  可以可靠唤醒进程，但不知道业务 predicate，也不知道谁在等同一条件。

只用 ConditionVariable:
  可以等待 predicate 变化，但不保护 predicate 本身，也没有阶段人数语义。

只用 Barrier:
  可以推进多进程阶段，但只适合一组参与者共同跨越阶段边界。
```

PostgreSQL 的常见答案不是“选一个万能原语”，而是分层：

```text
短临界区:
  用 spinlock、atomic 或 LWLock 修改共享状态。

长等待:
  释放业务锁后，用 latch / condition variable 睡眠。

群体阶段:
  用 barrier 记录 phase、participants、arrived，再用 CV/latch 实现等待。

退出兜底:
  用 ResourceOwner、CancelSleep、ReleaseAll、detach API 清理本 backend 留下的 ownership。
```

学完后应能独立判断：

```text
什么时候只需要 spinlock；
什么时候需要 LWLock；
什么时候 latch 足够；
什么时候应该在 latch 上再加 ConditionVariable；
什么时候等待点已经是阶段边界，必须使用 Barrier；
为什么“先改状态，再唤醒”等价于一个协议，而不是两行随意代码；
为什么 wait event 名字属于业务等待点，不属于底层同步原语；
为什么一个新等待点最危险的 bug 往往不是锁没加，而是 predicate、wait list、
cleanup 和唤醒顺序没有形成闭环。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a74`。

## 1. 本节在总主线中的位置

前六节分别拆开了五种同步原语：

```text
SpinLock:
  极短共享字段临界区。

LWLock:
  shared / exclusive 互斥、等待队列和 wait event 命名。

Latch:
  进程级可靠唤醒 bit，统一 signal、postmaster death、timeout、socket readiness。

ConditionVariable:
  等待调用方定义的 predicate 变化，内部通过 PGPROC wait list + procLatch 唤醒。

Barrier:
  多进程阶段推进，在 CV 之上增加 phase / participants / arrived。
```

这些课分别回答了“一个原语内部怎么工作”。本节换一个角度：

```text
当你正在写一个新的共享状态等待点时，
你如何决定该引入哪一层原语？
```

这个问题在 PostgreSQL 内核里非常实际。很多代码路径看起来像一串不同原语的混搭：

```text
Buffer I/O:
  BufferDesc.state 用 atomic / header lock 修改；
  buffer mapping table 用 LWLock 保护；
  BM_IO_IN_PROGRESS 的长等待用 ConditionVariable；
  CV 最终用 SetLatch 唤醒 waiter；
  I/O ownership 用 ResourceOwner 兜底。

Checkpoint request:
  ckpt_started / ckpt_done 用 spinlock 保护；
  请求 checkpointer 用 procLatch；
  等 checkpoint start/done 用 ConditionVariable；
  失败用 ckpt_failed 计数回传。

ProcSignalBarrier:
  全局 generation 用 atomic；
  每个 slot 的 PID / signal flags 用 spinlock；
  操作系统信号只负责触发 CHECK_FOR_INTERRUPTS；
  等所有 backend 吸收 barrier 用 ConditionVariable；
  signal handler 最后 SetLatch(MyLatch)。

Parallel Hash Join:
  算法阶段用 Barrier；
  Barrier 内部用 spinlock + ConditionVariable；
  shared hash/table/batch 状态还会用 LWLock、atomic、DSM/DSA 等机制。
```

这不是风格混乱，而是 PostgreSQL 对不同问题边界的明确切分。

本节主线会沿一个判断链展开：

```text
共享状态是什么？
  -> 是否需要互斥读写？
  -> 是否会长时间等待？
  -> 等待的是进程事件、predicate 变化，还是阶段推进？
  -> 谁负责唤醒？
  -> ERROR / exit 时谁负责撤销等待或释放 ownership？
  -> pg_stat_activity 应该看到哪个 wait event？
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
PostgreSQL 把同步问题拆成“状态保护、等待挂队、进程唤醒、阶段推进、退出清理”几层；
spinlock / atomic / LWLock 只让状态转换短暂、可见且互斥；
latch / CV / barrier 让等待离开临界区；
ResourceOwner 和 cancel/detach API 让 ERROR 后不会留下半个等待者或半个 owner。
```

最重要的判断是：你正在保护的是“字段”，还是“时间”？

字段保护问题：

```text
我要把一个共享字段从 A 改到 B；
其它 backend 不能看见半更新状态；
临界区很短；
更新期间不需要 sleep、I/O、ERROR。
```

这类问题属于 `SpinLock`、atomic 或 `LWLock`。

等待时间问题：

```text
我要等另一个 backend 将来做完某件事；
等待可能持续毫秒、秒甚至更久；
等待期间不能占住保护共享状态的短锁；
等待必须可被中断，最好能在 pg_stat_activity 中诊断。
```

这类问题不能留在 spinlock 或普通临界区里。它需要 `Latch`、`ConditionVariable` 或 `Barrier`。

进一步区分等待类型：

```text
等本进程被某类异步事件叫醒:
  用 Latch。

等某个共享 predicate 可能变真:
  用 ConditionVariable，predicate 仍由调用方维护。

等一组参与者都到达某个阶段边界:
  用 Barrier。
```

因此，一个新等待点的设计可以用下面的伪代码做 sanity check：

```c
/* 1. 用短锁读取或修改 predicate 状态。 */
lock_short_state();
if (predicate_is_true())
{
    unlock_short_state();
    return;
}
record_that_i_am_waiting_if_needed();
unlock_short_state();

/* 2. 长等待不能拿着短锁。 */
wait_with_latch_or_cv_or_barrier();

/* 3. 醒来后重新检查状态，不相信一次唤醒就是成功。 */
goto retry;
```

这里的关键不是某个 API 名称，而是不变量：

```text
状态变化必须有保护；
等待必须不持有不该持有的短锁；
唤醒必须发生在状态变化之后；
醒来必须重查状态；
退出必须摘掉等待者或释放 ownership。
```

## 3. 核心文件分工与阅读顺序

建议阅读顺序不是按文件名排序，而是按“从底层原语到组合案例”推进。

| 顺序 | 文件 | 阅读重点 |
| --- | --- | --- |
| 1 | `src/include/storage/spin.h`、`src/include/storage/s_lock.h` | spinlock 的使用边界：只保护极短共享字段更新 |
| 2 | `src/include/storage/lwlock.h`、`src/backend/storage/lmgr/lwlock.c` | `LWLock` state、wait queue、tranche、`held_lwlocks` 和 ERROR-safe release |
| 3 | `src/include/storage/latch.h`、`src/backend/storage/ipc/latch.c` | `SetLatch()` / `WaitLatch()` 作为进程级唤醒层 |
| 4 | `src/include/storage/condition_variable.h`、`src/backend/storage/lmgr/condition_variable.c` | CV 如何用 spinlock 保护 wait list，再用 procLatch 唤醒 |
| 5 | `src/include/storage/barrier.h`、`src/backend/storage/ipc/barrier.c` | Barrier 如何在 CV 上加 phase / participants / arrived |
| 6 | `src/include/storage/buf_internals.h`、`src/backend/storage/buffer/bufmgr.c` | Buffer I/O 如何组合 header state、LWLock、CV、ResourceOwner |
| 7 | `src/backend/postmaster/checkpointer.c` | checkpoint request 如何组合 spinlock、latch、CV |
| 8 | `src/backend/storage/ipc/procsignal.c` | 全局 ProcSignalBarrier 如何组合 atomic、signal、latch、CV |
| 9 | `src/backend/executor/nodeHash.c`、`src/backend/executor/nodeHashjoin.c`、`src/include/executor/hashjoin.h` | 并行 hash join 如何把 Barrier 作为 shared state machine |

本节会重点 walkthrough 三条路径：

```text
Buffer I/O:
  最典型的“短状态修改 + 长 predicate 等待”。

ProcSignalBarrier:
  最典型的“全局 generation + 每进程确认 + CV 等待确认”。

Parallel Hash Join:
  最典型的“predicate 等待不够，必须升级为阶段同步”。
```

## 4. 关键数据结构与状态

### 4.1 同步原语本身的状态边界

`SpinLock` 的语义边界最小：

```text
它不是资源 owner；
不是等待队列；
不是可诊断 wait event；
不是错误恢复机制；
只是几条指令级共享字段更新的互斥边界。
```

`LWLock` 的状态边界更大：

```text
保护一段共享结构；
支持 shared / exclusive；
可以让 backend 等待锁状态；
会进入 pg_stat_activity 的 LWLock wait event；
持有记录在 backend-local held_lwlocks 中，ERROR 后可释放。
```

`Latch` 的状态边界是 per-process wakeup bit：

```text
Latch 不知道业务 predicate；
它只保证 SetLatch() 之后 WaitLatch() 不会永久错过唤醒；
等待方必须自己 ResetLatch() 并重新检查业务状态。
```

`ConditionVariable` 的状态边界是共享 wait list：

```c
typedef struct
{
    slock_t       mutex;
    proclist_head wakeup;
} ConditionVariable;
```

它仍然不保存业务 predicate。业务状态在调用方结构中，例如：

```text
BufferDesc.state:
  BM_IO_IN_PROGRESS / BM_VALID / BM_DIRTY。

CheckpointerShmem:
  ckpt_started / ckpt_done / ckpt_failed。

ReplicationSlot:
  active_proc / invalidated / xmin / restart_lsn。
```

`Barrier` 的状态边界是阶段计数：

```c
typedef struct Barrier
{
    slock_t           mutex;
    int               phase;
    int               participants;
    int               arrived;
    int               elected;
    bool              static_party;
    ConditionVariable condition_variable;
} Barrier;
```

它仍然不是业务数据本身。它只表达：

```text
这一组参与者现在处于哪个 phase；
本 phase 有多少人需要到达；
已经到了多少人；
谁被选中执行串行小任务。
```

### 4.2 Buffer I/O 的组合状态

`BufferDesc` 是一个很好的组合案例。`buf_internals.h` 把 buffer state 压到一个 64-bit 变量中：

```text
refcount;
usage_count;
flags;
content lock state。
```

与本节相关的 flag：

```text
BM_VALID:
  page 内容有效。

BM_DIRTY:
  page 需要写回。

BM_IO_IN_PROGRESS:
  有 backend 正在对这个 buffer 执行读或写 I/O。

BM_IO_ERROR:
  上一次 I/O 失败。
```

每个 shared buffer 还有一个 I/O condition variable：

```c
extern PGDLLIMPORT ConditionVariableMinimallyPadded *BufferIOCVArray;

static inline ConditionVariable *
BufferDescriptorGetIOCV(const BufferDesc *bdesc)
{
    return &BufferIOCVArray[bdesc->buf_id].cv;
}
```

这里的分层非常清晰：

```text
BufferDesc.state:
  predicate 所在地。

LockBufHdr() / UnlockBufHdrExt():
  短临界区，修改 BM_IO_IN_PROGRESS 等状态。

BufferDescriptorGetIOCV(buf):
  等待“这个 buffer 的 I/O 状态可能变化”。

WaitLatch(MyLatch):
  CV 底层真正睡眠的位置。

ResourceOwnerRememberBufferIO():
  记录“我当前拥有这个 in-progress buffer I/O”，ERROR 时兜底。
```

### 4.3 Checkpointer 的组合状态

`checkpointer.c` 的共享状态中有一个 spinlock 和两个 CV：

```text
ckpt_lck:
  保护 ckpt_* 字段。

ckpt_started:
  checkpoint 开始时递增。

ckpt_done:
  checkpoint 完成时推进到对应 started。

ckpt_failed:
  checkpoint 失败时递增。

start_cv:
  ckpt_started 变化时 broadcast。

done_cv:
  ckpt_done 变化时 broadcast。
```

请求方的动作拆成三层：

```text
修改请求字段:
  持有 ckpt_lck。

叫醒 checkpointer:
  SetLatch(&checkpointerProc->procLatch)。

等待同步 checkpoint 完成:
  在 start_cv / done_cv 上做 predicate loop。
```

这说明一个事实：

```text
同一个业务动作可能同时需要 latch 和 CV。
```

`Latch` 负责叫醒 checkpointer 这个进程；`ConditionVariable` 负责把多个等待者挂到“checkpoint started/done 这个 predicate 变化”上。

### 4.4 ProcSignalBarrier 的组合状态

`procsignal.c` 中每个 slot 包含：

```c
typedef struct
{
    pg_atomic_uint32 pss_pid;
    int             pss_cancel_key_len;
    uint8           pss_cancel_key[MAX_CANCEL_KEY_LENGTH];
    volatile sig_atomic_t pss_signalFlags[NUM_PROCSIGNALS];
    slock_t         pss_mutex;

    pg_atomic_uint64 pss_barrierGeneration;
    pg_atomic_uint32 pss_barrierCheckMask;
    ConditionVariable pss_barrierCV;
} ProcSignalSlot;
```

这里名字里也有 `Barrier`，但它不是 `storage/barrier.h` 的 `Barrier`。它的目标不是让一组参与者同步进入下一算法阶段，而是：

```text
发起者发布一个 generation；
每个 backend 在安全点吸收 barrier 请求；
吸收后更新自己的 pss_barrierGeneration；
发起者等待所有 slot 的 generation 都追上。
```

它组合了：

```text
atomic:
  generation 和 check mask。

spinlock:
  slot PID、cancel key、signal flags 的短更新。

OS signal:
  触发目标 backend 注意到 PROCSIG_BARRIER。

latch:
  signal handler 最后 SetLatch(MyLatch)，打断 WaitLatch。

ConditionVariable:
  WaitForProcSignalBarrier() 等某个 slot 的 generation 追上。
```

这个案例提醒我们：

```text
“barrier”这个词在不同上下文中可能表示不同抽象。
CPU memory barrier、ProcSignalBarrier generation、storage Barrier phase
不能混为一谈。
```

## 5. 选择同步原语的判断表

面对一个新的共享状态等待点，可以从下面的问题开始。

| 问题 | 倾向原语 | 典型信号 |
| --- | --- | --- |
| 只是更新几个共享标志或计数？ | spinlock 或 atomic | 临界区只有几条指令，不能 sleep |
| 需要保护一段共享结构，读多写少或需要 exclusive 修改？ | LWLock | 有 shared / exclusive 模式，有明确 critical section |
| 只是要叫醒某个进程重新检查自己的工作队列或 signal flag？ | Latch | 目标是一个进程，不是一个业务 predicate wait list |
| 多个进程等待同一个业务条件可能变化？ | ConditionVariable | 需要 signal / broadcast，醒来后重查 predicate |
| 多个参与者必须共同跨过阶段边界？ | Barrier | 有 phase、participants、arrived、late joiner 或 last detacher |
| ERROR 后必须自动释放持有资源？ | ResourceOwner 或专门 cleanup | `Remember` / `Forget`、`CancelSleep`、`ReleaseAll`、`Detach` |

更细一点，可以这样判断。

第一问：临界区里会不会 sleep、I/O、palloc、elog(ERROR) 或等待其它 backend？

```text
会:
  不应该用 spinlock 保护整段逻辑。

不会，且只改少量字段:
  spinlock / atomic 可能合适。
```

第二问：等待目标是“锁可用”，还是“业务条件变真”？

```text
锁可用:
  LWLock 或 heavyweight lock。

业务条件变真:
  ConditionVariable 或 Latch。
```

第三问：这个条件是否属于某个单一进程的主循环？

```text
是:
  通常 SetLatch(target->procLatch) 足够。

否，多个 backend 可能等待同一个对象状态:
  通常需要 ConditionVariable。
```

第四问：等待是否需要统计“这一轮有几个人到达”？

```text
不需要:
  ConditionVariable predicate loop 足够。

需要:
  这是 Barrier 的领域。
```

第五问：谁在 ERROR / cancel / backend exit 时清理？

```text
锁:
  是否进入 held_lwlocks 或 lock manager resource owner？

CV sleep:
  是否能 ConditionVariableCancelSleep()？

Buffer I/O:
  是否 ResourceOwnerRememberBufferIO()？

Barrier participation:
  是否有明确 detach 路径？
```

很多坏设计都不是第一眼就错，而是漏了最后一问。

## 6. 主流程源码 walkthrough：Buffer I/O 的分层组合

`StartSharedBufferIO()` / `WaitIO()` / `TerminateBufferIO()` 是本节最重要的源码 walkthrough。它展示了 PostgreSQL 为什么不把 I/O 等待放进 buffer header lock 或 mapping LWLock。

### 6.1 查找 buffer 身份用 LWLock

查找 buffer table 时，代码先根据 `BufferTag` 得到 partition lock：

```c
newHash = BufTableHashCode(&newTag);
newPartitionLock = BufMappingPartitionLock(newHash);

LWLockAcquire(newPartitionLock, LW_SHARED);
buf_id = BufTableLookup(&newTag, newHash);
LWLockRelease(newPartitionLock);
```

这里的 `LWLock` 保护的是 mapping table 结构：

```text
tag -> buffer id 的哈希表是否被并发修改；
查找时允许多个 shared reader；
插入、删除、替换 tag 时需要 exclusive。
```

它不是用来等待磁盘 I/O 的。因为磁盘 I/O 可能很久，把 partition lock 持有到 I/O 完成会阻塞其它 backend 查找完全无关的 buffer tag。

### 6.2 开始 I/O 用短状态修改

`StartSharedBufferIO()` 的核心循环先锁住 buffer header：

```c
buf_state = LockBufHdr(buf);

if (!(buf_state & BM_IO_IN_PROGRESS))
    break;
```

如果发现已有 I/O：

```text
有 async wait reference:
  返回 BUFFER_IO_IN_PROGRESS，让调用方异步等待。

不想等待:
  返回 BUFFER_IO_IN_PROGRESS。

必须等待:
  释放 buffer header，再 WaitIO(buf)。
```

注意这个顺序：

```text
先看 BM_IO_IN_PROGRESS；
如果需要等，先 UnlockBufHdr(buf)；
再等待。
```

这就是“短临界区修改状态、长等待交给 CV”的基本形状。`WaitIO()` 不能在 `LockBufHdr()` 期间调用，否则完成 I/O 的 backend 也可能需要同一个 header lock 来清掉 `BM_IO_IN_PROGRESS`，等待者会把完成者堵住。

当没有 I/O 且确实需要发起 I/O 时：

```c
UnlockBufHdrExt(buf, buf_state,
                BM_IO_IN_PROGRESS, 0,
                0);

ResourceOwnerRememberBufferIO(CurrentResourceOwner,
                              BufferDescriptorGetBuffer(buf));
```

这里有两个含义：

```text
BM_IO_IN_PROGRESS:
  对其它 backend 发布“这个 buffer 正在 I/O”。

ResourceOwnerRememberBufferIO:
  对本 backend 记录“我负责结束这个 I/O”。
```

如果发起 I/O 后抛 ERROR，不能只靠内存释放。`BM_IO_IN_PROGRESS` 是 shared state，必须被清掉，否则其它 backend 会一直等待这个 buffer I/O。ResourceOwner 正是为了这种外部资源 ownership 兜底。

### 6.3 等 I/O 用 ConditionVariable

`WaitIO()` 使用 buffer 对应的 CV：

```c
ConditionVariable *cv = BufferDescriptorGetIOCV(buf);

ConditionVariablePrepareToSleep(cv);
for (;;)
{
    buf_state = LockBufHdr(buf);
    iow = buf->io_wref;
    UnlockBufHdr(buf);

    if (!(buf_state & BM_IO_IN_PROGRESS))
        break;

    if (pgaio_wref_valid(&iow))
    {
        pgaio_wref_wait(&iow);
        ConditionVariablePrepareToSleep(cv);
        continue;
    }

    ConditionVariableSleep(cv, WAIT_EVENT_BUFFER_IO);
}
ConditionVariableCancelSleep();
```

这个循环有四个正确性点。

第一，predicate 是 `BM_IO_IN_PROGRESS`，不在 CV 里。

```text
CV 只保存等待者；
退出条件必须重新读取 BufferDesc.state。
```

第二，读取 predicate 时短暂锁 header。

```text
这个检查是正确性边界，不是诊断用的随便读。
```

第三，等待时已经释放 header。

```text
完成者必须能进入 TerminateBufferIO() 清状态并 broadcast。
```

第四，wait event 是业务语义：

```text
WAIT_EVENT_BUFFER_IO
```

`pg_stat_activity` 应该告诉你 backend 正在等 buffer I/O，而不是只显示“ConditionVariable”。

### 6.4 完成 I/O 先改状态，再唤醒

`TerminateBufferIO()` 的核心顺序：

```c
buf_state = LockBufHdr(buf);

Assert(buf_state & BM_IO_IN_PROGRESS);
unset_flag_bits |= BM_IO_IN_PROGRESS;

buf_state = UnlockBufHdrExt(buf, buf_state,
                            set_flag_bits, unset_flag_bits,
                            refcount_change);

if (forget_owner)
    ResourceOwnerForgetBufferIO(CurrentResourceOwner,
                                BufferDescriptorGetBuffer(buf));

ConditionVariableBroadcast(BufferDescriptorGetIOCV(buf));
```

这条顺序非常重要：

```text
1. 持 header lock 清掉 BM_IO_IN_PROGRESS，并设置 BM_VALID 或 BM_IO_ERROR。
2. 释放 header lock。
3. Forget ResourceOwner 中的 I/O ownership。
4. Broadcast 这个 buffer 的 I/O CV。
```

如果先 broadcast 再清状态，等待者会醒来、重查、发现 `BM_IO_IN_PROGRESS` 还在，于是继续睡。虽然 spurious wakeup 本身可接受，但如果没有后续唤醒，就可能造成延迟甚至 missed wakeup 风险。

PostgreSQL 的惯用协议是：

```text
先让 predicate 真的变化；
再唤醒等待者重查 predicate。
```

## 7. 主流程源码 walkthrough：Checkpoint request 的 latch + CV

`RequestCheckpoint()` 展示了另一种组合：一个操作既要叫醒目标后台进程，又要让调用者等待一个可比较的完成条件。

### 7.1 请求字段由 spinlock 保护

请求方先记录旧计数，并设置请求标志：

```c
SpinLockAcquire(&CheckpointerShmem->ckpt_lck);

old_failed = CheckpointerShmem->ckpt_failed;
old_started = CheckpointerShmem->ckpt_started;
CheckpointerShmem->ckpt_flags |= (flags | CHECKPOINT_REQUESTED);

SpinLockRelease(&CheckpointerShmem->ckpt_lck);
```

这里使用 spinlock 是因为：

```text
只是读写几个共享字段；
临界区短；
不等待 checkpointer；
不做 I/O。
```

### 7.2 叫醒 checkpointer 用 latch

接着通过 `PGPROC.procLatch` 唤醒 checkpointer：

```c
SetLatch(&GetPGProcByNumber(checkpointerProc)->procLatch);
```

这一步不是等待 checkpoint 完成。它只是告诉 checkpointer：

```text
你的主循环应该醒来，重新检查 CheckpointerShmem->ckpt_flags。
```

如果 checkpointer 还没启动，代码会重试一段时间；如果调用方没有要求等待，则失败唤醒只是 LOG。原因是共享请求状态已经写入，即使这次 `SetLatch()` 没有命中，checkpointer 启动后仍然可以看到请求。

### 7.3 等开始和完成用两个 CV

如果调用方传入 `CHECKPOINT_WAIT`，就进入两个 predicate loop。

等待 checkpoint start：

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

等待 checkpoint done：

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

这里为什么不是一个 latch？

```text
因为多个 backend 可以同时等待 checkpoint start/done；
它们等待的是共享计数器变化；
唤醒后必须分别比较自己记录的 old_started / new_started。
```

为什么不是一个 barrier？

```text
因为请求方不是一组要共同到达阶段边界的参与者；
它们只是等待 checkpointer 推进一个全局计数。
```

所以 `ConditionVariable` 正合适。

## 8. 主流程源码 walkthrough：ProcSignalBarrier 的确认协议

`ProcSignalBarrier` 不是 `storage/barrier.h` 的 `Barrier`，但它是“组合边界”的好例子：它需要全局确认，却不需要参与者共同停在同一个算法阶段。

### 8.1 发起者发布 generation

`EmitProcSignalBarrier()` 先给所有 slot 设置 check bit：

```c
pg_atomic_fetch_or_u32(&slot->pss_barrierCheckMask, flagbit);
```

再推进全局 generation：

```c
generation =
    pg_atomic_add_fetch_u64(&ProcSignal->psh_barrierGeneration, 1);
```

源码注释强调这些 atomic 操作有 full barrier semantics。这里要表达的是：

```text
调用者在发 barrier 前完成的共享状态修改，
应该在目标 backend 吸收 barrier 时可见。
```

然后它遍历 slot，设置 `PROCSIG_BARRIER` 并发送 `SIGUSR1`：

```c
SpinLockAcquire(&slot->pss_mutex);
slot->pss_signalFlags[PROCSIG_BARRIER] = true;
SpinLockRelease(&slot->pss_mutex);
kill(pid, SIGUSR1);
```

这里的 spinlock 只保护 slot flags 的短更新，真正的跨进程通知由 OS signal 和 latch 处理。

### 8.2 signal handler 只设置 pending，并 SetLatch

`procsignal_sigusr1_handler()` 检查到 `PROCSIG_BARRIER` 后：

```c
HandleProcSignalBarrierInterrupt();
...
SetLatch(MyLatch);
```

`HandleProcSignalBarrierInterrupt()` 只做轻量动作：

```c
InterruptPending = true;
ProcSignalBarrierPending = true;
```

它不在 signal handler 中读取 64-bit atomic generation，也不做复杂处理。真正工作推迟到 `CHECK_FOR_INTERRUPTS()` 调用的 `ProcessProcSignalBarrier()`。

这是一条通用规则：

```text
signal handler 只做 async-signal-safe 的最小状态标记；
复杂共享状态处理放到普通执行上下文；
latch 负责把正在 WaitLatch 的 backend 叫醒。
```

### 8.3 backend 吸收 barrier 并广播自己的 CV

`ProcessProcSignalBarrier()` 读取本地和全局 generation：

```c
local_gen = pg_atomic_read_u64(&MyProcSignalSlot->pss_barrierGeneration);
shared_gen = pg_atomic_read_u64(&ProcSignal->psh_barrierGeneration);
```

如果需要处理，它先用 atomic exchange 清掉 check mask：

```c
flags = pg_atomic_exchange_u32(&MyProcSignalSlot->pss_barrierCheckMask, 0);
```

源码注释解释了顺序：必须先清 bit，再尝试处理。如果先处理再清 bit，新的 barrier 可能在窗口里进来，然后被错误清掉。

处理成功后：

```c
pg_atomic_write_u64(&MyProcSignalSlot->pss_barrierGeneration, shared_gen);
ConditionVariableBroadcast(&MyProcSignalSlot->pss_barrierCV);
```

这和 buffer I/O 的顺序相同：

```text
先更新 predicate:
  pss_barrierGeneration >= generation。

再唤醒等待者:
  Broadcast pss_barrierCV。
```

如果处理失败或 ERROR：

```text
ResetProcSignalBarrierBits(flags);
ProcSignalBarrierPending = true;
InterruptPending = true;
```

这使 backend 之后还会重试，不会假装已经吸收 barrier。

### 8.4 发起者等待所有 slot

`WaitForProcSignalBarrier()` 遍历所有 slot：

```c
oldval = pg_atomic_read_u64(&slot->pss_barrierGeneration);
while (oldval < generation)
{
    if (ConditionVariableTimedSleep(&slot->pss_barrierCV,
                                    5000,
                                    WAIT_EVENT_PROC_SIGNAL_BARRIER))
        ereport(LOG,
                (errmsg("still waiting for backend with PID %d to accept ProcSignalBarrier",
                        (int) pg_atomic_read_u32(&slot->pss_pid))));
    oldval = pg_atomic_read_u64(&slot->pss_barrierGeneration);
}
ConditionVariableCancelSleep();
```

这里为什么用 CV 而不是 `BarrierArriveAndWait()`？

```text
因为目标 backend 并不是同一段算法的 active participants；
有些 backend 可能退出，有些可能刚启动；
发起者要的是“每个 slot 都确认 generation 至少达到 X”；
不需要所有人共同停在一个 phase 边界。
```

这就是选择边界：有 generation confirmation，不等于需要 `Barrier` 抽象。

## 9. 阶段推进为什么需要 Barrier：Parallel Hash Join

现在看 `Barrier` 真正适合的场景。Parallel Hash Join 中，多个 backend 共同建立 hash table，阶段之间有严格依赖。

`hashjoin.h` 定义 build phases：

```c
#define PHJ_BUILD_ELECT          0
#define PHJ_BUILD_ALLOCATE       1
#define PHJ_BUILD_HASH_INNER     2
#define PHJ_BUILD_HASH_OUTER     3
#define PHJ_BUILD_RUN            4
#define PHJ_BUILD_FREE           5
```

`ParallelHashJoinState` 中有：

```c
LWLock  lock;
Barrier build_barrier;
Barrier grow_batches_barrier;
Barrier grow_buckets_barrier;
```

为什么这里不够用 CV？

```text
CV 可以让等待者重查“build_phase 是否变了”；
但还需要知道本 phase 有多少 attached backend；
谁已经到达；
最后一个到达者什么时候推进 phase；
late joiner 进入时应该从哪个 phase 开始；
最后一个 detacher 何时释放 shared batch。
```

这些都是 `Barrier` 的字段语义：

```text
phase:
  shared program counter。

participants:
  当前 attached backend 数。

arrived:
  本 phase 已到达人数。

elected:
  本 phase 选出的串行工作者。
```

在 `ExecHashTableCreate()` 中，parallel hash 初始化时 attach build barrier，并在 `PHJ_BUILD_ELECT` 阶段选一个 backend 分配共享结构：

```c
BarrierAttach(build_barrier);

if (BarrierPhase(build_barrier) == PHJ_BUILD_ELECT &&
    BarrierArriveAndWait(build_barrier, WAIT_EVENT_HASH_BUILD_ELECT))
{
    /* elected backend initializes shared hash table state */
}
```

在 `MultiExecParallelHash()` 中，backend 会根据当前 barrier phase fall through：

```c
switch (BarrierPhase(build_barrier))
{
    case PHJ_BUILD_ALLOCATE:
        BarrierArriveAndWait(build_barrier,
                             WAIT_EVENT_HASH_BUILD_ALLOCATE);

    case PHJ_BUILD_HASH_INNER:
        ...
        if (BarrierArriveAndWait(build_barrier,
                                 WAIT_EVENT_HASH_BUILD_HASH_INNER))
            ...
}
```

这个 fall through 是 dynamic party 的核心。late joiner 不能从头假装自己在 `PHJ_BUILD_ELECT`，它必须先读共享 `phase`，把本地程序计数器同步到当前阶段。

到 `PHJ_BUILD_RUN` 之后，`nodeHashjoin.c` 的注释还强调一个约束：

```text
在还可能有人等待某个 barrier 的阶段，不能向上层返回 tuple，
否则该 backend 可能长时间不再执行本节点，导致其它 attached backend 永久等待。
```

这说明 `Barrier` 不是“高级 CV”那么简单。它要求参与者遵守阶段协议：

```text
attach 后必须继续参与需要它参与的 phase；
不会再参与时必须 detach；
可能等待的 barrier 阶段不能把控制权交还给不保证及时回来的上层。
```

## 10. 生命周期 / ownership / cleanup

同步原语选择必须同时回答 ownership 问题。

### 10.1 SpinLock

创建：

```text
通常作为 shared struct 内字段初始化，例如 SpinLockInit(&state->mutex)。
```

持有：

```text
当前 CPU 执行流短暂持有。
```

释放：

```text
必须在同一条短路径显式 SpinLockRelease()。
```

ERROR 兜底：

```text
不要让 spinlock 持有期间可能 ERROR。
```

因此 spinlock 临界区里应避免：

```text
palloc；
ereport(ERROR)；
I/O；
WaitLatch / ConditionVariableSleep；
LWLockAcquire；
复杂函数调用。
```

### 10.2 LWLock

创建：

```text
静态 MainLWLockArray、named tranche 或嵌入 shared struct 的 LWLock。
```

持有：

```text
backend-local held_lwlocks 记录持有情况。
```

释放：

```text
正常路径 LWLockRelease()；
异常路径 LWLockReleaseAll()。
```

边界：

```text
LWLock 可以等待锁可用，但不适合被当成业务 predicate 的长睡眠机制。
```

### 10.3 Latch

创建：

```text
local latch 或 PGPROC.procLatch；
多数 backend/auxiliary process 使用 MyLatch。
```

持有：

```text
owner process 等待自己的 latch；
其它 process 可以 SetLatch()。
```

释放：

```text
Latch 不是资源所有权；它是可重复 set/reset 的 wakeup bit。
```

边界：

```text
SetLatch() 不携带业务状态；
业务状态必须先写好，等待方醒来再检查。
```

### 10.4 ConditionVariable

创建：

```text
通常嵌入 shared memory 或 DSM 中，ConditionVariableInit()。
```

持有：

```text
等待者把 MyProcNumber 挂入 cv->wakeup；
PGPROC.cvWaitLink 是每个 backend 的唯一 CV wait link。
```

释放：

```text
退出 predicate loop 后必须 ConditionVariableCancelSleep()。
```

ERROR 兜底：

```text
CancelSleep() 可以在 abort cleanup 中调用；
它能从当前 prepared CV wait list 摘掉自己。
```

边界：

```text
CV 不拥有 predicate；
predicate 的锁、生命周期、内存位置由调用模块负责。
```

### 10.5 Barrier

创建：

```text
BarrierInit(barrier, participants)；
participants > 0 是 static party；
participants == 0 是 dynamic party。
```

持有：

```text
dynamic party 需要 BarrierAttach()；
attached 后 phase 不会在没有本 backend 参与的情况下越过下一阶段。
```

释放：

```text
BarrierDetach()；
BarrierArriveAndDetach()；
BarrierArriveAndDetachExceptLast()。
```

ERROR 兜底：

```text
调用方必须有明确 detach 路径。
```

边界：

```text
Barrier 只管理阶段；
阶段内资源的创建、销毁、异常处理仍由调用方管理。
```

## 11. 正确性机制层次

### 11.1 先改 predicate，再唤醒

这是最重要的层次。

Buffer I/O：

```text
清 BM_IO_IN_PROGRESS / 设置 BM_VALID 或 BM_IO_ERROR；
Broadcast BufferIOCV。
```

Checkpoint：

```text
推进 ckpt_started 或 ckpt_done；
Broadcast start_cv 或 done_cv。
```

ProcSignalBarrier：

```text
写 pss_barrierGeneration；
Broadcast pss_barrierCV。
```

如果反过来做，等待者可能醒来后看不到新状态，然后重新睡下。CV 允许 spurious wakeup，但不允许你丢掉真正的 predicate 变化。

### 11.2 等待方必须重查 predicate

`ConditionVariableSleep()` 返回不代表条件满足。

可能原因包括：

```text
被 Signal / Broadcast；
被其它 latch set；
timeout；
CHECK_FOR_INTERRUPTS 中断路径改变了当前 CV sleep target；
spurious wakeup。
```

所以正确形式永远是：

```c
ConditionVariablePrepareToSleep(cv);
while (!predicate_is_true())
    ConditionVariableSleep(cv, wait_event);
ConditionVariableCancelSleep();
```

### 11.3 长等待必须离开短锁

`WaitIO()` 在检查 `BM_IO_IN_PROGRESS` 时短暂 `LockBufHdr()`，但等待前释放。

`RequestCheckpoint()` 在读取 `ckpt_started` / `ckpt_done` 时短暂持有 `ckpt_lck`，但 `ConditionVariableSleep()` 前释放。

`BarrierArriveAndWait()` 在更新 `arrived` / `phase` 时持有 `barrier->mutex`，但等待 phase 推进时睡在 CV 上。

这形成统一结构：

```text
short lock:
  只保护状态观察/转换。

wait primitive:
  在锁外承接长等待。
```

### 11.4 唤醒不是 ownership 转移

`SetLatch()` 只是 wakeup bit。

`ConditionVariableSignal()` 只是从 wait list 摘一个 PGPROC 并 set latch。

`ConditionVariableBroadcast()` 只是让等待者重查 predicate。

`BarrierArriveAndWait()` 返回 true 也只是一个 election 信号，真正能做什么取决于调用方阶段协议。

因此不要把“被唤醒”理解成：

```text
资源已经归我；
条件一定满足；
我可以不加锁读取所有共享字段；
其它 backend 不会同时操作。
```

### 11.5 wait event 应该描述业务等待

PostgreSQL 的 wait event 命名通常给诊断者看业务语义：

```text
WAIT_EVENT_BUFFER_IO:
  等 buffer I/O 完成。

WAIT_EVENT_CHECKPOINT_START:
  等 checkpoint 开始。

WAIT_EVENT_CHECKPOINT_DONE:
  等 checkpoint 完成。

WAIT_EVENT_PROC_SIGNAL_BARRIER:
  等其它 backend 吸收 ProcSignalBarrier。

WAIT_EVENT_HASH_BUILD_HASH_INNER:
  等 parallel hash build inner phase。
```

这比暴露“正在 CV sleep”更有价值。因为同样的 CV API 可以服务完全不同的业务 predicate。

## 12. 错误路径 / 异常路径 / fallback

### 12.1 Buffer I/O 失败

如果 I/O 失败，`TerminateBufferIO()` 会设置 `BM_IO_ERROR`，清掉 `BM_IO_IN_PROGRESS`，并 broadcast。

这保证：

```text
等待者不会永远卡在 WAIT_EVENT_BUFFER_IO；
醒来后能看到 I/O 不再进行；
后续路径可以根据 BM_IO_ERROR 或 page validity 决定报错、重试或处理。
```

如果 backend 在持有 buffer I/O ownership 时 ERROR，ResourceOwner 的 buffer I/O release callback 会参与兜底。核心原则是：

```text
shared in-progress bit 不能依赖正常路径才清理。
```

### 12.2 Checkpointer 不运行或 checkpoint 失败

`RequestCheckpoint()` 中，若 checkpointer 没有运行：

```text
如果调用方要求 CHECKPOINT_WAIT:
  达到重试上限后 ERROR。

如果不要求等待:
  LOG 后返回。
```

若 checkpoint 失败：

```text
ckpt_failed 递增；
等待方发现 new_failed != old_failed；
报 ERROR 并提示查看 server log。
```

这里的异常状态仍然通过同一套 predicate loop 传播。

### 12.3 ProcSignalBarrier 吸收失败

`ProcessProcSignalBarrier()` 对 barrier processing 包了 `PG_TRY()`。

如果某个 barrier type 处理失败或 ERROR：

```text
ResetProcSignalBarrierBits(flags);
ProcSignalBarrierPending = true;
InterruptPending = true;
```

这样 backend 不会提前写入新的 `pss_barrierGeneration`。发起者的 `WaitForProcSignalBarrier()` 仍会等待，必要时每 5 秒 LOG 一次仍在等待哪个 PID。

这体现了 generation protocol 的正确性：

```text
只有真正吸收 barrier 后，才能发布 generation 已追上。
```

### 12.4 Barrier participant 退出

`Barrier` 的动态参与者必须 detach。否则：

```text
participants 中还算着这个 backend；
其它 backend 可能永远等不到 arrived == participants；
阶段无法推进。
```

Parallel Hash Join 因此有很多显式 detach 路径：

```text
BarrierArriveAndDetach(&build_barrier);
BarrierArriveAndDetachExceptLast(&batch_barrier);
BarrierDetach(&grow_batches_barrier);
BarrierDetach(&grow_buckets_barrier);
```

新设计中只要使用 dynamic barrier，就必须同时设计：

```text
正常完成如何 detach；
提前退出如何 detach；
最后一个 detacher 是否负责释放 shared resource；
detach 是否可能推进 phase 并唤醒其它等待者。
```

## 13. 成本、资源与跨模块传播

### 13.1 SpinLock 成本

spinlock 成本主要来自：

```text
cache line 争用；
CPU 自旋；
stuck spinlock 风险；
临界区代码膨胀导致其它 backend 空转。
```

它适合更新：

```text
小计数器；
flag；
单个 slot 状态；
wait list 链接。
```

不适合：

```text
扫描大数组；
执行 I/O；
分配内存；
等待另一个 backend。
```

### 13.2 LWLock 成本

LWLock 成本来自：

```text
atomic state 修改；
等待队列操作；
process semaphore sleep/wakeup；
tranche 上的竞争和 cache line bouncing。
```

它适合保护共享结构，但不适合把无关长等待串行化。

Buffer mapping partition lock 就是典型例子：

```text
查找 tag 时拿；
查完立刻放；
磁盘 I/O 不在 mapping LWLock 内等待。
```

### 13.3 Latch 成本

`SetLatch()` 需要跨进程唤醒，可能涉及 signal、pipe、eventfd 或平台相关等待设施。源码注释多次强调：

```text
先写业务状态，再 SetLatch；
如果 latch 已经 set，SetLatch 会尽快返回；
不要用高频 latch storm 替代局部队列或条件合并。
```

### 13.4 ConditionVariable 成本

CV 成本来自：

```text
wait list spinlock；
proclist 操作；
SetLatch(waiter->procLatch)；
spurious wakeup 后重复检查 predicate。
```

它的收益是：

```text
不用每个模块自己发明 wait list；
wait event 可以挂在业务等待上；
ERROR cleanup 有统一 CancelSleep 入口；
DSM 中不用保存 backend 私有指针。
```

### 13.5 Barrier 成本

Barrier 成本来自：

```text
每个 phase 边界都需要参与者到达；
慢参与者拖住所有人；
dynamic attach/detach 需要维护 participants；
错误的 phase 协议可能导致全组等待。
```

它只在算法确实存在阶段边界时值得使用。

如果只是“某个字段变了叫醒别人”，用 Barrier 会过重，并且容易引入不必要的 participant lifecycle 问题。

## 14. 观测与诊断入口

### 14.1 pg_stat_activity

常见等待可从 `pg_stat_activity` 看：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
ORDER BY pid;
```

本节相关 wait event：

```text
BufferIO:
  wait_event 通常显示 BufferIO，对应 WAIT_EVENT_BUFFER_IO。

CheckpointStart / CheckpointDone:
  对应 WAIT_EVENT_CHECKPOINT_START / WAIT_EVENT_CHECKPOINT_DONE。

ProcSignalBarrier:
  对应 WAIT_EVENT_PROC_SIGNAL_BARRIER。

Hash build / grow / batch:
  HASH_BUILD_ELECT、HASH_BUILD_HASH_INNER、
  HASH_GROW_BATCHES_REPARTITION、HASH_BATCH_LOAD 等。
```

诊断时不要只问“用了哪个原语”，而要问：

```text
它在等哪个业务 predicate？
predicate 由哪个共享字段表达？
谁应该改变这个字段？
改变字段后谁负责唤醒？
等待者醒来后是否能继续推进？
```

### 14.2 日志

ProcSignalBarrier 等待超过 5 秒会 LOG：

```text
still waiting for backend with PID ... to accept ProcSignalBarrier
```

checkpoint 失败会在请求方报：

```text
checkpoint request failed
```

并提示查看 server log。诊断时应把等待事件和 server log 放在一起看：

```text
pg_stat_activity:
  谁在等？

server log:
  推进者是否失败？
```

### 14.3 gdb 断点

可以设置这些断点观察状态转换：

```gdb
break StartSharedBufferIO
break WaitIO
break TerminateBufferIO
break ConditionVariableSleep
break ConditionVariableBroadcast
break RequestCheckpoint
break EmitProcSignalBarrier
break ProcessProcSignalBarrier
break BarrierArriveAndWait
```

观察 Buffer I/O 时重点看：

```gdb
p/x buf->state.value
p buf->buf_id
p buf->io_wref
```

观察 Barrier 时重点看：

```gdb
p barrier->phase
p barrier->participants
p barrier->arrived
p barrier->elected
```

观察 ProcSignalBarrier 时重点看：

```gdb
p pg_atomic_read_u64(&ProcSignal->psh_barrierGeneration)
p pg_atomic_read_u64(&slot->pss_barrierGeneration)
p pg_atomic_read_u32(&slot->pss_barrierCheckMask)
```

### 14.4 perf 和采样

如果 CPU 消耗集中在 spinlock acquire 或 atomic CAS 附近，通常说明：

```text
临界区过热；
共享 cache line 被频繁抢占；
应该检查是否需要分区、padding、local aggregation 或减少共享字段更新频率。
```

如果大量 backend 在 `BufferIO`、`CheckpointDone`、`ProcSignalBarrier` 等 wait event 上等待，通常说明：

```text
瓶颈不在 CV 本身；
应该追踪 predicate 的推进者为什么慢。
```

## 15. 常见误区

误区一：把 spinlock 当成“小号 LWLock”。

```text
spinlock 不是可睡眠锁，也没有 ERROR cleanup。
临界区必须极短。
```

误区二：用 latch 替代 predicate。

```text
Latch 只表示“醒来看看”。
业务条件必须保存在共享状态里。
```

误区三：`ConditionVariableSleep()` 返回就认为条件满足。

```text
CV 允许 spurious wakeup。
必须 while 循环重查 predicate。
```

误区四：把 `ConditionVariableBroadcast()` 放在状态变化之前。

```text
正确协议是先改 predicate，再唤醒。
```

误区五：看到“barrier”就想到 `Barrier`。

```text
CPU memory barrier、ProcSignalBarrier generation、storage Barrier phase 是不同概念。
只有“多参与者共同跨阶段”才需要 storage Barrier。
```

误区六：忘记 cleanup。

```text
CV 要 CancelSleep；
LWLock 要 release；
Buffer I/O 要 ResourceOwnerForget 或异常 release；
dynamic Barrier 要 detach。
```

误区七：wait event 设计成底层原语名。

```text
诊断者关心的是 BufferIO、CheckpointDone、HashBuildHashInner，
不是“正在等待一个 CV”。
```

## 16. 课堂实验

### 实验 1：观察 Buffer I/O 等待点

目标：把 `WAIT_EVENT_BUFFER_IO` 和 `BM_IO_IN_PROGRESS` 的源码关系连起来。

步骤：

```text
1. 在一个较大的表上制造冷缓存顺序扫描或随机读。
2. 同时查询 pg_stat_activity 中 wait_event = 'BufferIO' 的 backend。
3. 用 gdb 给 WaitIO() 和 TerminateBufferIO() 下断点。
4. 在 WaitIO() 中观察 buffer state 是否包含 BM_IO_IN_PROGRESS。
5. 在 TerminateBufferIO() 中观察清 flag 后 Broadcast 的顺序。
```

思考：

```text
如果 WaitIO() 持有 buffer header lock 睡眠，完成 I/O 的 backend 会遇到什么问题？
```

### 实验 2：观察 checkpoint start/done 两个 predicate

目标：理解同一个请求为什么同时需要 latch 和 CV。

步骤：

```text
1. 开启一个会话执行 CHECKPOINT。
2. 给 RequestCheckpoint()、ConditionVariableSleep()、ConditionVariableBroadcast() 下断点。
3. 观察请求方设置 ckpt_flags 后如何 SetLatch(checkpointer)。
4. 观察请求方如何分别等待 start_cv 和 done_cv。
5. 查询 pg_stat_activity，看等待事件是否是 CheckpointStart 或 CheckpointDone。
```

思考：

```text
为什么请求方不能只 WaitLatch(MyLatch) 等 checkpoint 完成？
```

### 实验 3：观察 ProcSignalBarrier generation

目标：区分 ProcSignalBarrier 和 storage Barrier。

步骤：

```text
1. 找一个会触发 ProcSignalBarrier 的路径，例如全局状态变更相关操作。
2. 给 EmitProcSignalBarrier()、ProcessProcSignalBarrier()、
   WaitForProcSignalBarrier() 下断点。
3. 观察 psh_barrierGeneration 和每个 slot 的 pss_barrierGeneration。
4. 观察 signal handler 只设置 pending，并在末尾 SetLatch(MyLatch)。
```

思考：

```text
为什么 WaitForProcSignalBarrier() 遍历每个 slot 的 generation，
而不是让所有 backend 调 BarrierArriveAndWait()？
```

### 实验 4：观察 Parallel Hash Join 的 phase

目标：判断什么时候 CV 不够，必须使用 Barrier。

步骤：

```text
1. 开启并行 hash join：
   SET max_parallel_workers_per_gather = 4;
   SET enable_hashjoin = on;
2. 构造足够大的 join，让 plan 出现 Parallel Hash。
3. 给 BarrierAttach()、BarrierArriveAndWait()、
   MultiExecParallelHash() 下断点。
4. 观察 build_barrier 的 phase / participants / arrived。
5. 查询 pg_stat_activity 中 HASH_BUILD_* 或 HASH_BATCH_* wait event。
```

思考：

```text
late joiner 为什么必须先看 BarrierAttach() 返回的 phase？
```

### 实验 5：设计一个新等待点

假设你要在 shared memory 中维护一个后台任务队列，多个 backend 可以提交任务，一个 worker 消费任务，提交者可以选择等待任务完成。

请回答：

```text
任务队列结构用什么保护？
worker 如何被叫醒？
提交者等待任务完成用 latch 还是 CV？
如果任务分多个全局阶段，是否需要 Barrier？
ERROR 后提交者从哪里摘掉自己的等待状态？
pg_stat_activity 的 wait_event 应该叫什么？
```

## 17. 讨论题

1. 为什么 `BM_IO_IN_PROGRESS` 适合用 CV 等待，而不是用 LWLock 一直锁住 buffer header 到 I/O 完成？

2. `RequestCheckpoint()` 为什么要先修改 `ckpt_flags`，再 `SetLatch()` checkpointer？

3. `ConditionVariableBroadcast()` 为什么不能替代 Barrier 的 `participants` / `arrived` 计数？

4. ProcSignalBarrier 的 generation confirmation 和 `Barrier.phase` 有什么共同点和根本区别？

5. 如果一个 backend attach 了 dynamic Barrier，但随后把控制权返回给上层很久不回来，可能造成什么问题？

6. 设计 wait event 时，为什么应使用业务语义而不是底层原语名？

7. 为什么很多同步 bug 表现为“偶发卡住”，而不是稳定崩溃？

## 18. 本节小结

本节的可迁移规律：

```text
同步原语不是按“强弱”选择，而是按语义边界组合。
```

可以把 PostgreSQL 的选择压缩成一张心智图：

```text
几个共享字段的极短修改:
  spinlock / atomic。

共享结构的互斥读写:
  LWLock。

叫醒某个进程重查自己的状态:
  Latch。

等待某个共享 predicate 可能变化:
  ConditionVariable + 调用方 predicate loop。

等待一组参与者共同跨过阶段:
  Barrier。

任何跨 ERROR 边界的 ownership:
  ResourceOwner 或专门 cleanup / detach 协议。
```

源码里的组合通常遵循同一条线：

```text
短锁内改变状态；
释放短锁；
唤醒等待者；
等待者醒来重查 predicate；
异常路径清理本 backend 留下的 wait/ownership/participation。
```

读 PostgreSQL 同步代码时，不要只问“这里用了哪把锁”。更好的问题是：

```text
predicate 在哪里？
谁保护它？
谁改变它？
谁等待它？
谁唤醒等待者？
醒来后如何验证？
ERROR 后如何撤销？
这个等待在 pg_stat_activity 中叫什么？
```

能连续回答这八个问题，基本就能判断一个新共享状态等待点该使用哪种同步原语，以及它们应该如何组合。
