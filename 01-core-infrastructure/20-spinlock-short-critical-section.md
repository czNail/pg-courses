# PostgreSQL SpinLock 与极短 shared state 临界区

## 课程定位

前置知识：已经理解 PostgreSQL 是多进程模型，知道 shared memory 中的状态会被多个 backend 同时访问，也已经见过 `PGPROC`、`ProcArray`、ResourceOwner 和 shared memory 初始化的基本边界。

本节唯一主问题：

```text
为什么 PostgreSQL 只允许 spinlock 保护几条指令级的共享字段更新，s_lock.h / spin.h / s_lock.c 如何在 CPU 原子指令、memory barrier、退避和 stuck spinlock PANIC 之间取舍？
```

核心矛盾：shared memory 中有些字段必须用极低成本串行化，例如 WAL 插入位置、wait queue 链表、少量计数器和进程 freelist；但 spinlock 的等待方式本质上是占用 CPU 反复尝试，持锁期间又不能进入复杂错误处理、不能等待 I/O、不能调用可能 `ERROR` 的路径。PostgreSQL 的解法是把 spinlock 压缩成一个非常窄的原语：

```text
只保护几个共享字段的瞬时状态跃迁
  -> acquisition / release 提供编译器与硬件 ordering 边界
  -> 争用时短暂忙等，再退避睡眠
  -> 长时间拿不到锁时 PANIC，因为这通常意味着持锁者死在不允许停留的位置
```

学完后应能判断：

```text
为什么 spinlock 不是轻量版 LWLock；
为什么 spinlock 临界区不能做 I/O、内存分配、等待 latch 或调用可能 ereport(ERROR) 的函数；
为什么 SpinLockAcquire() / SpinLockRelease() 自身承担 memory ordering 责任；
为什么 TAS() 和 TAS_SPIN() 分开；
为什么 s_lock.c 在忙等后会 pg_usleep()，但仍把长时间失败视为 stuck spinlock；
为什么 ERROR-safe cleanup 不能依赖 ResourceOwner 自动释放 spinlock；
如何在代码审查中识别“这个状态应该用 spinlock、atomic、LWLock 还是 condition variable”。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a74`。

## 1. 本节在总主线中的位置

前几节的核心是 shared state 的身份和生命周期：

```text
postmaster 创建 shared memory
  -> backend attach 到同一组共享结构
  -> PGPROC / ProcArray 发布事务状态
  -> ResourceOwner 和 exit callback 清理外部资源
```

从本节开始进入同步原语。顺序很重要：

```text
SpinLock
  -> 保护极短字段更新，不提供睡眠等待语义

LWLock
  -> 保护更大的共享结构，支持 shared/exclusive、等待队列和 wait event

Latch
  -> 让一个进程被异步唤醒

Condition Variable
  -> 把“条件变真”变成可睡眠的谓词等待循环

Barrier
  -> 让多个进程在阶段边界对齐
```

Spinlock 是这一组同步原语的底层边界，不是通用锁。它经常被用来保护其它同步原语内部的小状态，比如 `ConditionVariable` 的 wait list、`LWLock` tranche 注册表、`Barrier` 的计数器。也就是说：

```text
spinlock 不是用来等待复杂条件的；
它常常是实现“更高级等待机制”的短临界区工具。
```

如果把 spinlock 用成普通 mutex，系统会出现几类典型问题：

| 错误用法 | 后果 |
| --- | --- |
| 持锁期间做 I/O 或长循环 | 其它 backend 会持续自旋或退避睡眠，CPU 和 latency 被放大。 |
| 持锁期间可能 `ERROR` | 没有 ResourceOwner 兜底释放，锁可能永久保持 locked。 |
| 持锁期间等待 latch / semaphore / LWLock | 可能形成不可诊断的等待环或 starvation。 |
| 持锁期间访问复杂对象生命周期 | 指针、refcount、invalidation 和 cleanup 边界难以证明。 |
| 用 spinlock 保护长读操作 | 读者越多，竞争越严重，不如 LWLock shared mode 或 atomic snapshot。 |

本节只讲 spinlock 本身。下一节会把问题推进到：

```text
如果临界区不能压缩到几条指令，为什么就需要 LWLock？
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
调用者通过 SpinLockAcquire() 进入一个只包含少量 load/store 的 shared memory 临界区；
底层 S_LOCK() 先用 TAS() 尝试一次原子 test-and-set，失败后进入 s_lock()；
s_lock() 用 TAS_SPIN()、SPIN_DELAY()、perform_spin_delay() 在忙等、CPU pause、随机退避睡眠之间切换；
成功后由 TAS/S_UNLOCK 提供 acquire/release ordering；
调用者必须尽快 SpinLockRelease()，否则其它 backend 长时间拿不到锁会触发 stuck spinlock PANIC。
```

这背后的 tension 是：

```text
shared state 更新必须足够便宜，可以出现在高频热路径；
但越便宜的同步原语，能承载的语义越少。
```

PostgreSQL 对这个 tension 的取舍非常明确：

| 需求 | spinlock 提供什么 | spinlock 不提供什么 |
| --- | --- | --- |
| 互斥 | 一个 CPU 原子 test-and-set 保护极短临界区。 | shared/exclusive 模式、公平队列、死锁检测。 |
| ordering | acquisition / release 阻止编译器和硬件把临界区内存访问移出边界。 | 跨多个对象的事务性一致性。 |
| 等待 | 短忙等，必要时退避睡眠。 | 可取消等待、可中断等待、条件谓词等待。 |
| cleanup | 调用者手动释放。 | ResourceOwner、ERROR abort、自动 unlock。 |
| 诊断 | `WAIT_EVENT_SPIN_DELAY`、stuck spinlock PANIC、perf 自旋热点。 | 像 lock table 那样的 blocker / waiter 图。 |

因此 spinlock 的系统规律是：

```text
它不是“锁住一段逻辑”，而是“让几个共享字段在 CPU ordering 边界内完成一次不可拆分的状态跃迁”。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/spin.h` | 对外 API：`SpinLockInit()`、`SpinLockAcquire()`、`SpinLockRelease()`，以及“few instructions”编码规则。 |
| 2 | `src/include/storage/s_lock.h` | 平台相关实现边界：`slock_t`、`S_LOCK`、`S_UNLOCK`、`TAS`、`TAS_SPIN`、`SPIN_DELAY`、memory barrier 要求。 |
| 3 | `src/backend/storage/lmgr/s_lock.c` | 平台无关等待逻辑：`s_lock()`、`perform_spin_delay()`、`finish_spin_delay()`、`s_lock_stuck()`。 |
| 4 | `src/backend/access/transam/xlog.c` | `insertpos_lck` 如何把 WAL reservation 压缩成几个字段更新。 |
| 5 | `src/backend/storage/lmgr/proc.c` | `freeProcsLock` 如何保护 `PGPROC` freelist，并顺手传播 `spins_per_delay`。 |
| 6 | `src/backend/storage/lmgr/condition_variable.c` | 更高级等待机制内部如何用 spinlock 保护 wait list。 |
| 7 | `src/backend/utils/activity/wait_event_names.txt` | `SPIN_DELAY` wait event 的观测入口。 |

推荐阅读顺序：

```text
先读 spin.h 的注释，确认使用规则
  -> 读 s_lock.h 顶部注释，理解 TAS / barrier / platform contract
  -> 读 s_lock.c，理解争用时不是无限纯忙等
  -> 看 xlog.c 的 ReserveXLogInsertLocation()，观察“几个字段更新”的理想形态
  -> 看 proc.c / condition_variable.c，理解 spinlock 如何服务更大同步结构
```

不要从所有 `SpinLockAcquire()` 调用点开始扫。调用点很多，真正要先建立的模型是：

```text
spinlock 保护的是短状态跃迁，不是业务流程。
```

## 4. 关键数据结构与状态

### `slock_t`: 只是一小块共享内存标志位

`slock_t` 在 `s_lock.h` 中由平台实现决定。常见平台上它可能是 `unsigned char`、`int`、`LONG` 等。课程里不要把它理解成一个复杂对象，它只是：

```text
0:
  unlocked

非 0:
  locked
```

它的语义不来自字段本身，而来自访问它的指令：

```text
slock_t raw byte
  + TAS 原子 test-and-set
  + S_UNLOCK release barrier
  + 调用者极短临界区约束
  = PostgreSQL spinlock 语义
```

这也解释了为什么 `spin.h` 的 API 接收 `volatile slock_t *`，但它的注释又强调从 PostgreSQL 9.5 起，调用者不再需要用 volatile-qualified 指针访问受保护数据。ordering 责任已经下沉到 lock/unlock 宏本身：

```text
调用者不靠 volatile 维持共享字段语义；
SpinLockAcquire/Release 边界负责阻止重排。
```

### `TAS()` 与 `TAS_SPIN()`: 第一次尝试和争用后轮询不同

`s_lock.h` 说明通常 `S_LOCK()` 建立在两个更低层宏上：

| 宏 | 语义 |
| --- | --- |
| `TAS(lock)` | 原子 test-and-set，尝试获得锁，不等待。成功返回 0，失败返回非 0。 |
| `TAS_SPIN(lock)` | 已知发生争用后用于等待循环。默认等于 `TAS()`，部分架构上会先做非原子读，只有看起来空闲时才再次执行昂贵的原子指令。 |

以 x86_64 为例，本地源码中：

```text
TAS()
  -> lock xchgb

TAS_SPIN()
  -> 先读 *lock
  -> 如果看起来已锁，直接返回失败
  -> 如果看起来空闲，再执行 TAS()
```

这背后的成本模型是：

```text
原子 test-and-set 会让 cache line 在 CPU 之间反复失效；
争用时先用普通 load 观察状态，可以减少总线锁和 cache coherency 压力。
```

注意 `s_lock.h` 还提醒：某些平台上 `TAS()` 可能在锁实际空闲时也报告失败。因此调用者不能写“我确定空闲，所以只尝试一次”的逻辑，必须允许 retry。

### `SpinDelayStatus`: 争用等待的本地状态

`SpinDelayStatus` 定义在 `s_lock.h`：

| 字段 | 语义 |
| --- | --- |
| `spins` | 当前这一轮已经忙等多少次。 |
| `delays` | 已经进入睡眠退避多少次。 |
| `cur_delay` | 当前睡眠时间，最初为 0，第一次睡眠后从 1ms 开始随机增长。 |
| `file` / `line` / `func` | stuck spinlock PANIC 时报告调用点。 |

它是 backend-local 栈上状态，不是 shared state。它服务两个目标：

```text
争用短时:
  尽量不进内核，不做昂贵等待。

争用持续:
  让出 CPU，避免所有等待者一直抢占持锁者。
```

### `spins_per_delay`: 每个 backend 的自适应忙等阈值

`s_lock.c` 中有一个 backend-local 静态变量：

```text
static int spins_per_delay = DEFAULT_SPINS_PER_DELAY;
```

`finish_spin_delay()` 根据本次是否进入过睡眠调整它：

```text
没有睡眠就拿到锁:
  增大 spins_per_delay，说明忙等可能划算。

进入过睡眠:
  缓慢减小 spins_per_delay，说明继续忙等可能浪费 CPU。
```

`proc.c` 在 backend 获得或释放 `PGPROC` 时还会把这个估计和 `ProcGlobal->spins_per_delay` 做一次共享传播：

```text
InitProcess()
  -> 持有 ProcGlobal->freeProcsLock
  -> set_spins_per_delay(ProcGlobal->spins_per_delay)

ProcKill()
  -> 持有 ProcGlobal->freeProcsLock
  -> update_spins_per_delay(...)
```

这里有一个很好的边界例子：传播估计值本身也必须非常快，所以源码注释明确说这两个函数是在持有 spinlock 时调用的，必须足够快。

### 受 spinlock 保护的 shared state: 字段组合才是语义

典型例子是 `xlog.c` 中的 `XLogCtlInsert`：

```text
insertpos_lck:
  保护 CurrBytePos 和 PrevBytePos

CurrBytePos:
  已预留 WAL usable byte position 的末尾

PrevBytePos:
  上一条已预留 WAL record 的起点，用于下一条 record 的 xl_prev
```

单独看 `CurrBytePos` 或 `PrevBytePos` 都不是完整语义。它们必须在同一个 spinlock 临界区内一起更新：

```text
startbytepos = CurrBytePos
endbytepos = startbytepos + size
prevbytepos = PrevBytePos
CurrBytePos = endbytepos
PrevBytePos = startbytepos
```

这就是本课要反复强调的规则：

```text
raw field 不是语义；
field + lock boundary + update ordering + lifecycle state 才是语义。
```

## 5. 主流程源码 walkthrough

本节主流程选择 `ReserveXLogInsertLocation()`，因为它展示了 spinlock 最理想的使用方式：

```text
高频热路径
  -> 必须串行化少量字段
  -> 把计算尽量移出临界区
  -> 持锁时只做 load/store
  -> 释放后再做稍重的转换和断言
```

源码位置：

```text
src/backend/access/transam/xlog.c
  -> ReserveXLogInsertLocation()
```

调用背景：

```text
XLogInsertRecord()
  -> 需要为一条 WAL record 预留连续空间
  -> 多个 backend 可以并发构造 WAL 内容
  -> 但“下一条 record 从哪里开始”必须全局串行
```

### 5.1 进入临界区前：先把能算的都算掉

`ReserveXLogInsertLocation()` 先做：

```text
size = MAXALIGN(size)
Assert(size > SizeOfXLogRecord)
```

真正进入 spinlock 前，函数注释已经说明设计意图：

```text
CurrBytePos 只记录 WAL 中可用字节的位置，不包含 page header；
usable byte position 到 XLogRecPtr 的映射可以放在锁外；
持锁时预留 X 字节几乎就是 CurrBytePos += X。
```

这就是 spinlock 临界区设计的第一条原则：

```text
临界区不是从“函数开始”划到“函数结束”；
它只包住必须共同原子观察和更新的几个字段。
```

### 5.2 持有 `insertpos_lck`: 完成一次不可拆分状态跃迁

临界区内的核心动作是：

```text
SpinLockAcquire(&Insert->insertpos_lck);

startbytepos = Insert->CurrBytePos;
endbytepos = startbytepos + size;
prevbytepos = Insert->PrevBytePos;
Insert->CurrBytePos = endbytepos;
Insert->PrevBytePos = startbytepos;

SpinLockRelease(&Insert->insertpos_lck);
```

这几行完成的语义是：

```text
我拿到 [startbytepos, endbytepos) 这段 WAL reservation；
下一位插入者应该从 endbytepos 开始；
下一条 WAL record 的 xl_prev 应该指向我的 startbytepos。
```

为什么这里不能只用 atomic fetch-add 保护 `CurrBytePos`？

```text
因为 PrevBytePos 必须和本次 reservation 的 startbytepos 一起推进；
这不是单字段计数器递增，而是两个字段组成的状态跃迁。
```

为什么这里不需要 LWLock？

```text
临界区没有睡眠、没有队列、没有 shared/exclusive 模式、没有复杂读写；
等待者只需要等几条指令，使用 LWLock 的排队和 semaphore 成本反而过重。
```

### 5.3 释放后：做较重的派生计算

释放 spinlock 后，函数再做：

```text
*StartPos = XLogBytePosToRecPtr(startbytepos);
*EndPos = XLogBytePosToEndRecPtr(endbytepos);
*PrevPtr = XLogBytePosToRecPtr(prevbytepos);

Assert(...)
```

这些转换和断言不需要持锁，因为它们只依赖本 backend 刚刚复制出来的局部变量。这个安排非常关键：

```text
持锁时只更新 shared state；
释放后用 local copy 做派生计算。
```

这是写 PostgreSQL spinlock 临界区时最值得迁移的模式。

### 5.4 争用路径：从 `S_LOCK()` 到 `s_lock()`

如果 `SpinLockAcquire()` 没有立刻成功，宏展开大致是：

```text
SpinLockAcquire()
  -> S_LOCK(lock)
     -> TAS(lock) ? s_lock(lock, file, line, func) : 0
```

第一次 `TAS()` 成功时，完全不进入 `s_lock.c`。这是 hot path 成本控制：

```text
无争用:
  一次平台原子指令 + acquire barrier。

有争用:
  进入平台无关的等待循环。
```

`s_lock()` 的主体是：

```text
init_spin_delay(&delayStatus, file, line, func);

while (TAS_SPIN(lock))
{
    perform_spin_delay(&delayStatus);
}

finish_spin_delay(&delayStatus);
```

`perform_spin_delay()` 每次先执行平台相关的 `SPIN_DELAY()`。在 x86 上通常是 `rep; nop`，在部分 ARM64 上是 `isb`。这些指令不是释放 CPU 的系统调用，而是告诉处理器当前处在 spin-wait loop，减少 pipeline 和超线程资源上的负面影响。

达到 `spins_per_delay` 后，`perform_spin_delay()` 会：

```text
delays++
超过 NUM_DELAYS 时 s_lock_stuck()
第一次睡眠从 1ms 开始
报告 WAIT_EVENT_SPIN_DELAY
pg_usleep(cur_delay)
cur_delay 随机增长，超过 1s 后回到 1ms
清零 spins
```

所以 PostgreSQL 的 spinlock 不是永远纯忙等。它的实际策略是：

```text
短争用:
  忙等，避免进入内核。

长争用:
  睡眠退避，让持锁者获得运行机会。

异常长争用:
  PANIC，因为这通常表示持锁者没有按规则释放锁。
```

## 6. 生命周期 / ownership / cleanup

Spinlock 的生命周期没有 ResourceOwner 那样的“自动资源管理”。它更接近一个嵌在 shared memory 结构里的原始同步字段。

### 谁创建？

创建通常发生在 shared memory 初始化或 DSM 初始化阶段：

```text
SpinLockInit(&XLogCtl->Insert.insertpos_lck)
SpinLockInit(&XLogCtl->info_lck)
SpinLockInit(&ProcGlobal->freeProcsLock)
SpinLockInit(&cv->mutex)
SpinLockInit(&barrier->mutex)
```

`SpinLockInit()` 只是调用 `S_INIT_LOCK()`，默认通常等价于把锁置为 unlocked。它不分配内存，不注册 owner，也不进入全局 lock table。

### 谁持有？

持有者是当前执行到 `SpinLockAcquire()` 并成功完成 `TAS()` 的 backend 或后台进程。PostgreSQL 没有在 `slock_t` 里记录 owner pid，也没有 deadlock detector 能告诉你 blocker 是谁。

这正是 spinlock 必须极短的原因之一：

```text
系统没有为 spinlock 等待维护丰富诊断状态；
它依赖编码规则保证持锁时间短到不需要这种状态。
```

### 谁释放？

调用者必须显式调用 `SpinLockRelease()`。常见写法是：

```text
SpinLockAcquire(&lock);
修改几个字段;
SpinLockRelease(&lock);
```

如果中间有提前返回路径，必须先 release 再 return。比如 `ReserveXLogSwitch()` 在发现已经位于 segment 边界时：

```text
SpinLockAcquire(&Insert->insertpos_lck);
...
if (...)
{
    SpinLockRelease(&Insert->insertpos_lck);
    return false;
}
...
SpinLockRelease(&Insert->insertpos_lck);
```

代码审查时最重要的不是“有没有 acquire/release 成对出现”，而是：

```text
所有 return / goto / ereport 前是否已经 release；
持锁区内是否调用了可能 ERROR 的函数；
持锁区内是否隐藏了内存分配、I/O、锁等待或 callback。
```

### ERROR / abort 时谁兜底？

没有人兜底。

这句话很重要。Spinlock 不挂在 `ResourceOwner` 上，也不会在 transaction abort 中自动释放。`spin.h` 的注释明确说，PostgreSQL 假设持有 spinlock 时不会发生 `CHECK_FOR_INTERRUPTS()`，因此这些函数里不需要 `HOLD_INTERRUPTS()` / `RESUME_INTERRUPTS()`。

这个设计成立的前提是：

```text
持锁区只包含几条不会 ERROR、不会等待、不会长时间运行的指令。
```

如果这个前提被破坏，后果不是普通事务 abort 能清理的资源泄漏，而可能是：

```text
其它 backend 反复尝试拿锁
  -> 进入 SPIN_DELAY
  -> 最终 stuck spinlock detected
  -> PANIC
  -> postmaster 重启共享内存世界
```

### 长期对象如何失效？

Spinlock 本身没有 invalidation 语义。被它保护的 shared structure 生命周期由外层机制决定：

| 外层对象 | 生命周期机制 |
| --- | --- |
| `XLogCtl` | postmaster 创建的主 shared memory，随 postmaster 生命周期存在。 |
| `ProcGlobal` | 主 shared memory 中的 proc 管理结构，server 生命周期存在。 |
| `ConditionVariable` | 嵌在具体 shared state 或 DSM 对象内，由对应对象生命周期决定。 |
| DSM 中的 spinlock | DSM segment attach/detach 生命周期决定，但跨进程指针仍不能直接复用。 |

因此，不要问“spinlock 什么时候 invalidated”。应该问：

```text
包含这个 spinlock 的 shared object 什么时候初始化、可见、停止被访问、最后销毁？
```

## 7. 正确性机制层次

Spinlock 的正确性不是一个层次，而是几个层次叠在一起。

### 7.1 原子性：`TAS()` 把 unlocked 变成 locked

最底层是 CPU 原子 test-and-set。它保证多个 backend 同时尝试时，只有一个能把锁从 unlocked 改成 locked。

在 x86_64 上，本地源码使用带 `lock` 前缀的 `xchgb`。在 ARM、PowerPC、MIPS、Sparc、Windows 等平台上，`s_lock.h` 选择不同实现。课程不要求背每个平台的汇编，但要知道 PostgreSQL 把平台差异收敛到这个 contract：

```text
TAS/TAS_SPIN 成功后，后续 load/store 不能跑到 acquire 之前；
S_UNLOCK 前，临界区内 load/store 不能跑到 release 之后。
```

### 7.2 编译器 barrier：防止编译器“优化掉”临界区边界

`s_lock.h` 顶部注释强调，gcc inline asm 的 clobbers list 应包含 `"memory"`。这不是 CPU 指令本身的互斥，而是告诉编译器：

```text
不要把 shared memory 字段缓存到寄存器并跨过 asm；
不要把临界区内访问重排到 lock/unlock 外面。
```

这解释了 PostgreSQL 9.5 之后的接口变化：调用者不需要把所有受保护字段都写成 volatile 访问，因为 lock/unlock 宏自己承担编译器重排边界。

### 7.3 硬件 memory fence：弱内存序平台的 acquire/release

在 x86 这类较强内存序平台上，很多 ordering 由原子指令和内存模型自然提供。在 PowerPC、Sparc、MIPS、ARM 等弱内存序平台上，`s_lock.h` 必须显式插入 `lwsync`、`membar`、`sync` 或对应 builtin。

如果没有这些 fence，可能出现非常难诊断的问题：

```text
backend A:
  持锁更新 shared field
  release lock

backend B:
  acquire lock
  仍看不到 A 在临界区内的更新
```

所以 spinlock 的 release 不是普通 store 0。它是：

```text
先保证临界区内内存访问已经对其它 CPU 可见
  -> 再把 lock 标志置为 unlocked
```

### 7.4 编码规则：不等待、不 ERROR、不复杂

原子指令和 fence 只能保证内存顺序，不能保证系统不会死锁。PostgreSQL 靠编码规则补上这一层：

```text
持锁区只做简单字段访问；
不调用可能 ereport(ERROR) 的函数；
不调用 CHECK_FOR_INTERRUPTS()；
不做内存分配和 I/O；
不等待其它 lock/latch/semaphore；
不把 callback 或插件代码放进临界区。
```

这是正确性机制的一部分，不是风格建议。

### 7.5 PANIC：把不该发生的长等待升级为系统级失败

`s_lock_stuck()` 在普通 server 构建中调用：

```text
elog(PANIC, "stuck spinlock detected at ...")
```

为什么不是 `ERROR`？

```text
因为 stuck spinlock 意味着 shared memory 同步原语可能已经坏掉；
当前 backend 不能靠事务 abort 恢复全局一致性。
```

PANIC 会导致当前进程退出，并由 postmaster 按崩溃路径处理。这个选择很重，但符合 spinlock 的语义边界：

```text
spinlock 持锁时间应该短到“长时间拿不到”近似等价于 bug、硬件/平台问题或持锁者崩溃。
```

## 8. 错误路径 / 异常路径 / fallback

### 8.1 正常争用 fallback: 从忙等到睡眠退避

`perform_spin_delay()` 的 fallback 不是换成 LWLock，而是调整等待策略：

```text
前 N 次:
  SPIN_DELAY()，仍在用户态忙等。

超过 spins_per_delay:
  pg_usleep(cur_delay)，进入 wait event SPIN_DELAY。

多次 delay:
  cur_delay 随机增长，避免所有 waiter 同步醒来继续抢。

超过 NUM_DELAYS:
  s_lock_stuck()。
```

这说明 PostgreSQL 并不幻想所有 spinlock 都零争用。它真正假设的是：

```text
争用可以发生，但持锁区短到等待者很快能看到锁释放。
```

### 8.2 `TAS()` 假失败: retry 是 contract 的一部分

`s_lock.h` 提醒有些平台可能在锁空闲时也让 `TAS()` 返回失败。例如某些历史架构可能因为中断导致 test-and-set 失败。因此：

```text
S_LOCK() 必须允许 retry；
调用者不能把一次 TAS 失败解释成锁一定被其它 backend 持有。
```

这类 awkwardness 是跨平台数据库内核里必须保留的现实边界。

### 8.3 持锁期间发现错误: 先 release，再 ereport

`lwlock.c` 的 `LWLockNewTrancheId()` 提供了一个小但很重要的模式：

```text
SpinLockAcquire(&LWLockTranches->lock);

if (num_user_defined >= MAX_USER_DEFINED_TRANCHES)
{
    SpinLockRelease(&LWLockTranches->lock);
    ereport(ERROR, ...);
}

更新共享数组;

SpinLockRelease(&LWLockTranches->lock);
```

这不是偶然写法。规则是：

```text
如果错误检查必须依赖受保护 shared state，可以在持锁区检查；
但真正抛 ERROR 前必须释放 spinlock。
```

### 8.4 stuck spinlock: 诊断点是 acquire 调用点，不一定是持锁者

`s_lock_stuck()` 报告的是等待者正在尝试 acquire 的文件、行号和函数。它不一定告诉你谁持有锁。

因此排查 stuck spinlock 时，思路通常是：

```text
从 PANIC 报告的 acquire 点定位是哪把锁
  -> 搜索同一把锁的所有 acquire/release
  -> 检查是否有 return / ereport / goto 路径漏 release
  -> 检查持锁区是否调用了可能长时间运行或等待的函数
  -> 检查平台 TAS / S_UNLOCK 实现和 memory barrier 是否可疑
```

这和 heavyweight lock 的诊断完全不同。你无法期待 `pg_locks` 里出现一条 blocker。

## 9. 成本、资源与跨模块传播

### 成本模型

Spinlock 的无争用成本很低：

```text
一次原子 test-and-set
  + acquire barrier
  + 几个 shared field load/store
  + release barrier
```

但争用成本上升很快：

```text
等待者越多
  -> 同一个 cache line 在 CPU core 之间来回迁移
  -> TAS / TAS_SPIN 失败次数增加
  -> perform_spin_delay() 开始进入 pg_usleep()
  -> latency 变得更离散
```

这就是为什么 PostgreSQL 尽量把 spinlock 用在：

```text
低持锁时间
低共享字段数量
低语义复杂度
可容忍短暂不可公平
```

### 高频热路径例子：WAL reservation

`insertpos_lck` 位于 WAL insert 热路径。`xlog.c` 注释直接说：

```text
ReserveXLogInsertLocation() 是 XLogInsert 中必须跨 backend 串行的性能关键部分；
其它工作尽量并行；
insertpos_lck 在繁忙系统上可能高度争用；
所以锁内代码要尽量短。
```

这类注释读源码时要格外重视。它往往告诉你：

```text
这里的锁不是抽象选择，而是经过热路径成本约束后的结果。
```

### 跨模块传播：spinlock 常常藏在更高级机制内部

几个典型组合：

| 模块 | spinlock 保护什么 | 外层语义 |
| --- | --- | --- |
| `condition_variable.c` | `cv->wakeup` wait list。 | 等待某个谓词变真。 |
| `barrier.c` | `participants`、`arrived`、`phase` 等计数。 | 多进程阶段推进。 |
| `lwlock.c` | `LWLockTranches` 注册表。 | wait event 命名和 LWLock tranche 管理。 |
| `proc.c` | `PGPROC` freelist 与 `spins_per_delay` 估计值。 | backend identity 分配和回收。 |
| `shm_mq.c` / `shm_toc.c` | 小型 DSM 元数据。 | 动态共享内存中的消息队列和目录。 |

这说明一个重要边界：

```text
高级同步原语可以用 spinlock 保护内部元数据；
业务路径不应该用 spinlock 去直接实现长等待协议。
```

### 与 atomic 的边界

有些地方不使用 spinlock，而是用 atomic state。例如 buffer header 的 `BM_LOCKED` 使用 `pg_atomic_fetch_or_u64()` 加 CAS 风格循环，并复用 `SpinDelayStatus` 做等待退避。

这说明：

```text
SpinDelayStatus 是通用的自旋退避工具；
spinlock 是一种具体的互斥协议。
```

如果状态能被一个 atomic word 完整表达，且更新可以用 CAS / fetch-or 完成，atomic 可能比 spinlock 更合适。反过来，如果需要同时更新多个字段并保持组合语义，spinlock 更自然。

### 与 LWLock 的边界

当你发现临界区需要这些能力时，应该考虑 LWLock：

```text
可能等待较久；
需要 shared / exclusive 模式；
需要 wait queue 和唤醒协议；
需要 wait_event 明确表示等待哪类资源；
需要 ERROR-safe release 兜底；
需要保护较大的共享结构遍历。
```

本节结论不是“spinlock 很快，所以优先用”。更准确是：

```text
只有当状态跃迁能被压缩到极短、不可 ERROR、不可等待的字段更新时，spinlock 才成立。
```

## 10. 观测与诊断入口

Spinlock 的可观测性比 LWLock 少很多，因为它不维护 blocker / waiter 图。仍然有几个入口。

### 10.1 `pg_stat_activity`: `SPIN_DELAY`

当 `perform_spin_delay()` 进入 `pg_usleep()` 退避时，会报告：

```text
WAIT_EVENT_SPIN_DELAY
```

`wait_event_names.txt` 中描述为：

```text
Waiting while acquiring a contended spinlock.
```

可以用 SQL 观察：

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event = 'SpinDelay';
```

注意两点：

```text
短暂忙等不会出现在 pg_stat_activity 中；
只有进入 pg_usleep() 退避后才有 wait event。
```

因此，如果完全看不到 `SpinDelay`，不能证明没有 spinlock 争用。它只说明争用没有长到进入可观测睡眠窗口，或者采样没碰到。

### 10.2 日志: stuck spinlock PANIC

典型日志形态是：

```text
PANIC:  stuck spinlock detected at <func>, <file>:<line>
```

这个位置是等待者调用 `SpinLockAcquire()` 的位置。诊断时要把它当成“哪把锁拿不到”的起点，而不是直接断定该函数漏 release。

### 10.3 perf: 忙等热点

短时间争用不会进入 `SPIN_DELAY`，但会在 CPU profile 中出现。常见热点可能包括：

```text
tas / s_lock
perform_spin_delay
平台 pause 指令附近
调用点所在函数，例如 ReserveXLogInsertLocation()
```

如果 perf 显示大量 CPU 消耗在自旋等待，下一步不是立刻调大参数，而是回到源码判断：

```text
是否某个临界区过长；
是否 workload 让某个全局字段成为瓶颈；
是否该状态已经不适合 spinlock，而应拆分、分片、atomic 化或改用 LWLock/队列。
```

### 10.4 gdb 断点

教学环境可以用：

```text
break s_lock
break perform_spin_delay
break s_lock_stuck
```

但生产环境不要随意断在这些函数上。`s_lock()` 是底层同步路径，暂停一个 backend 可能放大其它 backend 的等待。

更温和的源码阅读练习是：

```bash
rg -n "SpinLockAcquire\\(&XLogCtl->Insert.insertpos_lck\\)|SpinLockRelease\\(&XLogCtl->Insert.insertpos_lck\\)" /home/highgo/postgres/src/backend/access/transam/xlog.c
rg -n "SpinLockAcquire\\(&ProcGlobal->freeProcsLock\\)|SpinLockRelease\\(&ProcGlobal->freeProcsLock\\)" /home/highgo/postgres/src/backend/storage/lmgr/proc.c
rg -n "WAIT_EVENT_SPIN_DELAY|perform_spin_delay|s_lock_stuck" /home/highgo/postgres/src
```

## 11. 常见误区

### 误区 1：spinlock 是“更快的 LWLock”

不是。它们解决的问题不同：

```text
SpinLock:
  几条指令的共享字段状态跃迁。

LWLock:
  可能较长的共享结构访问，支持等待队列、唤醒、shared/exclusive、wait event 和 cleanup。
```

如果你需要问“这个 spinlock 等待者怎么排队、公平性如何、ERROR 时怎么释放”，通常说明你需要的已经不是 spinlock。

### 误区 2：持锁区短只是性能建议

不是。持锁区短是 correctness 前提。因为：

```text
没有 ResourceOwner 自动释放；
没有死锁检测；
没有中断检查；
没有 blocker 可观测性；
长时间拿不到锁会 PANIC。
```

### 误区 3：只保护一个字段就一定该用 atomic

不一定。如果是单字段计数器或 bit state，atomic 可能合适；但如果需要和其它字段、生命周期或检查条件组合，即使只看到一个主要字段，也可能需要 lock boundary。

判断标准不是字段数量，而是：

```text
一次状态跃迁能否用一个 atomic operation 表达完整语义？
```

### 误区 4：`volatile` 能替代 lock/unlock ordering

不能。`volatile` 不是跨 CPU 的 acquire/release 同步协议。PostgreSQL 当前接口要求 lock/unlock 宏提供编译器和硬件 ordering，调用者不应该靠 volatile 访问受保护数据来补语义。

### 误区 5：看到 `pg_usleep()` 就说明 spinlock 可以长等

`pg_usleep()` 是争用 fallback，不是鼓励长临界区。它的作用是避免等待者把 CPU 全部烧掉，给持锁者运行机会。进入大量 `SPIN_DELAY` 已经说明 spinlock 争用值得调查。

### 误区 6：stuck spinlock 可以像普通 ERROR 一样恢复

不能。stuck spinlock 指向共享内存同步层面的严重异常，PostgreSQL 用 PANIC 而不是 ERROR。事务 abort 不能可靠释放一个没有 owner tracking 的 spinlock。

## 12. 课堂实验

### 实验 1：画出 `insertpos_lck` 临界区边界

目标：理解 WAL reservation 为什么是 spinlock 的理想形态。

步骤：

```bash
sed -n '1120,1265p' /home/highgo/postgres/src/backend/access/transam/xlog.c
```

回答：

```text
哪些计算发生在 SpinLockAcquire() 前？
哪些字段在临界区内被共同更新？
哪些转换发生在 SpinLockRelease() 后？
如果把 XLogBytePosToRecPtr() 移进临界区，会增加什么成本？
```

预期结论：

```text
锁内只有 CurrBytePos / PrevBytePos 的组合状态跃迁；
WAL physical pointer 转换依赖 local copy，可以移出锁外。
```

### 实验 2：寻找持锁区内的 ERROR 前 release 模式

目标：建立“ERROR 前必须释放 spinlock”的审查肌肉。

步骤：

```bash
sed -n '560,610p' /home/highgo/postgres/src/backend/storage/lmgr/lwlock.c
```

重点观察 `LWLockNewTrancheId()`：

```text
它在持有 LWLockTranches->lock 时检查容量；
但 ereport(ERROR) 前先 SpinLockRelease()。
```

继续搜索：

```bash
rg -n "SpinLockAcquire|ereport\\(ERROR|elog\\(ERROR" /home/highgo/postgres/src/backend/storage/lmgr/lwlock.c
```

回答：

```text
哪些错误检查可以在持锁区内完成？
为什么真正 ERROR 不能在持锁区内抛出？
```

### 实验 3：观察 `SPIN_DELAY` wait event

目标：理解 spinlock 争用的可观测窗口很窄。

在测试实例中运行高并发写入 workload，例如 `pgbench` 写事务，同时另开窗口采样：

```sql
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY 1, 2
ORDER BY count(*) DESC;
```

如果看到 `SpinDelay`，说明至少有 backend 进入了 `perform_spin_delay()` 的睡眠退避阶段。如果看不到，也不能证明没有 spinlock 争用，因为短暂忙等不会稳定暴露在 `pg_stat_activity` 中。

讨论：

```text
为什么 SpinDelay 更像“争用已经长到可被采样”，而不是完整 spinlock 等待统计？
```

### 实验 4：用 `rg` 给 spinlock 调用点分类

目标：区分三类常见用途。

命令：

```bash
rg -n "SpinLockInit|SpinLockAcquire|SpinLockRelease" /home/highgo/postgres/src/backend /home/highgo/postgres/src/include | less
```

尝试把调用点归类：

| 类型 | 例子 | 判断依据 |
| --- | --- | --- |
| shared counter / position | `XLogCtl->Insert.insertpos_lck` | 几个字段共同推进。 |
| freelist / registry | `ProcGlobal->freeProcsLock`、`LWLockTranches->lock` | 短链表或数组元数据更新。 |
| higher-level primitive 内部锁 | `ConditionVariable->mutex`、`Barrier->mutex` | 外层提供等待语义，spinlock 只保护内部队列或计数。 |

最后回答：

```text
哪些调用点如果改成 LWLock 会引入过重成本？
哪些调用点如果扩大临界区会立刻破坏 spinlock 使用规则？
```

### 实验 5：源码改动思考题，不建议在生产实例运行

在教学分支中，尝试把 `ReserveXLogInsertLocation()` 的锁内区域人为扩大，例如把 WAL pointer 转换移进锁内，或在锁内插入短暂 sleep。不要在生产实例运行这类改动。

观察点：

```text
高并发写入吞吐是否下降；
pg_stat_activity 是否更容易看到 SpinDelay；
是否可能触发 stuck spinlock PANIC；
perf 中自旋等待热点是否上升。
```

这个实验的价值不是制造故障，而是验证：

```text
spinlock 的正确使用边界非常窄；
临界区多几行代码，在高并发热路径上就可能变成系统症状。
```

## 13. 讨论题

1. `ReserveXLogInsertLocation()` 为什么不用一个全局 LWLock 保护整个 WAL insert record 构造过程？
2. `CurrBytePos` 看起来像单调递增计数器，为什么不能只用 atomic fetch-add 完成全部语义？
3. 如果某个 spinlock 临界区需要调用 `palloc()`，你会如何重构？
4. 为什么 stuck spinlock 使用 PANIC，而不是 ERROR 或 FATAL？
5. `TAS_SPIN()` 中先做普通 load 再做原子 TAS，会牺牲什么，换来什么？
6. 如果在 `pg_stat_activity` 中看不到 `SpinDelay`，还能用哪些方法判断是否存在 spinlock 热点？
7. 在什么情况下，一个原本由 spinlock 保护的小状态应该演化成 LWLock、condition variable 或 atomic state？
8. 为什么 `ConditionVariable` 内部仍然需要 spinlock，而不是只依赖 latch？

## 14. 本节小结

Spinlock 是 PostgreSQL shared memory 同步体系中最窄、最低层的互斥边界。它的价值不是“功能强”，而是“成本低到可以放在极高频状态跃迁里”。为了获得这个成本，PostgreSQL 放弃了很多高级语义：没有 owner tracking，没有 deadlock detection，没有 ERROR-safe cleanup，没有公平队列，也不能承载长等待。

本节主线可以压缩成：

```text
需要串行化几个共享字段
  -> 用 TAS/S_UNLOCK 建立互斥和 acquire/release ordering
  -> 临界区只做不可 ERROR 的瞬时状态跃迁
  -> 争用时短忙等、再退避睡眠
  -> 长时间拿不到锁视为 stuck spinlock，升级为 PANIC
```

可迁移的系统规律是：

```text
越底层、越便宜的同步原语，越依赖调用者把语义压缩到极小边界；
如果状态变化需要等待、cleanup、可观测 blocker 或复杂生命周期，它就不再是 spinlock 问题。
```

下一节进入 `LWLock`：当 shared state 的临界区无法再压缩到几条指令，PostgreSQL 如何用 state word、tranche、shared/exclusive 模式和等待队列，在成本与可等待语义之间重新取舍。
