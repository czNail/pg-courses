# PostgreSQL Barrier 与多进程阶段推进

## 课程定位

前置知识：已经理解 `SpinLock` 只保护极短共享字段更新，`Latch` 是进程级唤醒 bit，`ConditionVariable` 把等待同一 predicate 的 `PGPROC` 组织成共享 wait list。

本节唯一主问题：

```text
并行 hash join 这类多阶段算法为什么需要所有参与者在阶段边界对齐，
BarrierArriveAndWait()、BarrierAttach()、BarrierDetach()
如何用 phase、participants、arrived 和 condition variable
支持 static party 与 dynamic party？
```

核心矛盾：并行算法希望多个 backend 尽量独立推进，减少中心化锁竞争；但某些阶段又存在严格的全局边界：

```text
hash table 还没分配完，其他进程不能开始插入 tuple；
inner relation 还没全部 hash 完，probe 阶段不能读取不完整状态；
batch repartition 还没完成，任何人都不能按旧 nbatch / bucket 布局继续访问；
最后一个参与者离开后，才允许释放 shared memory。
```

如果只用一个 `ConditionVariable`，它只能表达“某个条件可能变了，请醒来重查”。它不知道：

```text
这一轮到底应该等多少参与者；
谁已经到达了阶段边界；
新加入者应该从算法的哪个阶段开始；
最后一个到达者是否应该推进 phase；
某个参与者 detach 后，等待人数是否减少到足以放行；
某个串行小任务应由哪个 backend 执行。
```

`Barrier` 的抽象就是在 `ConditionVariable` 之上增加“阶段推进”的共享状态：

```text
phase:
  当前算法阶段，也可以理解成 shared program counter。

participants:
  当前 attached 参与者数量，或者 static party 的固定人数。

arrived:
  当前 phase 已经到达 barrier 的参与者数量。

condition_variable:
  phase 尚未推进时，等待者睡在这里。
```

学完后应能独立判断：

```text
为什么多阶段并行算法不能只靠一把 LWLock 或一个 ConditionVariable；
为什么 dynamic barrier 要让 BarrierAttach() 返回当前 phase；
为什么 BarrierArriveAndWait() 的 true 返回值可以用作“选一个人干串行活”；
为什么 BarrierDetach() 可能推进 phase；
为什么 BarrierPhase() 在 attached 前提下可以无锁读取；
为什么 parallel hash join 不允许在可能再次 wait 的 barrier 阶段向上层 emit tuple；
为什么 batch barrier 要用 ArriveAndDetachExceptLast() 做 wait-free election；
为什么 late joiner 必须用 phase 同步本地状态机，而不是从头执行。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节 `ConditionVariable` 的模型是：

```text
调用方维护 predicate；
ConditionVariable 维护 wait list；
Signal / Broadcast 只是让 waiter 重查 predicate。
```

这能解决：

```text
buffer I/O 完成了吗？
checkpoint 到我等待的轮次了吗？
replication slot 是否不再 active？
```

但它没有“群体阶段”的概念。假设 4 个 backend 一起做一个三阶段算法：

```text
阶段 A:
  每个人产生局部结果。

阶段 B:
  必须读完所有人的阶段 A 输出。

阶段 C:
  必须读完阶段 B 的全局整理结果。
```

每个 backend 都可以用 CV 等待“阶段 A 完成”，但它们还需要一个共享计数：

```text
这一轮有几个人参与？
已经到了几个人？
最后一个到的人如何推进到下一阶段？
如果有人退出，这一轮等待人数如何变化？
```

这就是 `Barrier` 的位置：

```text
ConditionVariable:
  解决“睡在哪里、怎么被唤醒”。

Barrier:
  解决“一组参与者什么时候共同跨过一个阶段边界”。
```

在 PostgreSQL 中，最能体现这个抽象的调用方是 Parallel Hash Join。

`nodeHashjoin.c` 的文件注释直接把 parallel-aware hash join 描述成两个状态机：

```text
per-backend state machine:
  每个 backend 的本地执行位置。

shared state machine:
  多个 backend 共享的阶段位置，由 Barrier 管理。
```

这句话是读本节的钥匙。`Barrier` 不只是一个等待工具，它把多个 backend 的本地程序计数器压到一个共享 `phase` 上。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Barrier 是一个带 phase 计数器的 ConditionVariable；
参与者 attach 后进入 participants 集合；
每次 ArriveAndWait() 把 arrived 加一；
当 arrived == participants 时，最后到达者把 arrived 清零、phase 加一，
Broadcast 唤醒其他等待者；
没等到 phase 推进的人睡在 barrier->condition_variable 上；
dynamic party 中 detach 会减少 participants，并可能让等待者足够放行。
```

它解决的不是“共享字段互斥访问”。字段更新仍然由内部 spinlock 保护：

```text
Barrier.mutex:
  保护 phase / participants / arrived / elected 的短临界区。
```

它解决的是“算法时间”的一致性：

```text
backend A 不能在 phase 2 的 shared memory layout 上读；
backend B 还在按 phase 1 的 layout 写。
```

为了让这句话落地，可以先看一个最小 static party 模型：

```c
perform_a();
BarrierArriveAndWait(&barrier, WAIT_EVENT_SOMETHING);

perform_b();
BarrierArriveAndWait(&barrier, WAIT_EVENT_SOMETHING);

perform_c();
BarrierArriveAndWait(&barrier, WAIT_EVENT_SOMETHING);
```

如果 `BarrierInit(&barrier, 4)`，每轮都等 4 个参与者。

Parallel Hash Join 更接近 dynamic party：

```c
phase = BarrierAttach(&barrier);

switch (phase)
{
    case PHASE_ALLOCATE:
        ...
    case PHASE_LOAD:
        ...
    case PHASE_PROBE:
        ...
}

BarrierDetach(&barrier);
```

dynamic party 的核心是：

```text
参与者可能晚到；
晚到者不能从算法开头盲目执行；
它必须读取当前 phase，把本地状态机跳到正确位置。
```

这也是为什么 `BarrierAttach()` 返回 `phase`，而不是只返回“attach 成功”。

## 3. 核心文件分工与阅读顺序

本节建议按下面顺序读源码。

| 文件 | 作用 | 本节关注点 |
| --- | --- | --- |
| `src/include/storage/barrier.h` | `Barrier` 结构与 API 声明 | `phase` / `participants` / `arrived` / `elected` / `static_party` |
| `src/backend/storage/ipc/barrier.c` | Barrier 状态转移实现 | arrive、wait、attach、detach、phase advance、election |
| `src/include/executor/hashjoin.h` | Parallel Hash Join shared state | `build_barrier`、`grow_*_barrier`、`batch_barrier` 与 phase 常量 |
| `src/backend/executor/nodeHashjoin.c` | Hash join 状态机入口与 DSM 初始化 | shared state machine 为什么依赖 barrier |
| `src/backend/executor/nodeHash.c` | parallel hash build、growth、batch probe | dynamic barrier 的真实调用链 |
| `src/backend/utils/activity/wait_event_names.txt` | wait event 名称 | pg_stat_activity 里看到的 barrier 等待点 |

阅读顺序不要从 `barrier.c` 顶部注释读完就停。更好的顺序是：

```text
1. 先看 barrier.h 的字段组合；
2. 再看 BarrierArriveAndWait() 如何推进 phase；
3. 再看 BarrierDetachImpl() 为什么 detach 也可能推进 phase；
4. 然后进入 ParallelHashJoinState，看哪些算法阶段被 barrier 表示；
5. 最后跟一条 MultiExecParallelHash() / ExecParallelHashJoinNewBatch() 主流程。
```

本节只讲 IPC barrier，不讲 CPU memory barrier。PostgreSQL 源码里也有 `pg_memory_barrier()` 这一类“内存顺序屏障”，它解决的是 CPU / compiler 重排序问题；本节的 `Barrier` 是多进程阶段同步 primitive，名字相近但层次完全不同。

## 4. 关键数据结构与状态

`Barrier` 在 `src/include/storage/barrier.h` 中定义：

```c
typedef struct Barrier
{
    slock_t     mutex;
    int         phase;
    int         participants;
    int         arrived;
    int         elected;
    bool        static_party;
    ConditionVariable condition_variable;
} Barrier;
```

不要把这些字段拆开孤立理解。它们的语义来自组合。

### 4.1 phase：共享程序计数器

`phase` 是 barrier 的核心语义字段。

在 static party 中，调用方通常不需要显式读取 phase：

```text
每个参与者都按同一段代码顺序执行；
每次 ArriveAndWait() 返回，就说明大家都跨过了同一个边界；
本地程序计数器自然知道下一步该做什么。
```

在 dynamic party 中，phase 必须暴露给调用方：

```text
新参与者 attach 时，算法可能已经推进到中间阶段；
它需要根据当前 phase 跳过已经完成的步骤。
```

Parallel Hash Join 正是 dynamic party：

```text
worker 可能晚于 leader 进入某个 batch；
某个 batch 可能已经完成 allocate / load；
晚到者必须从 BarrierAttach() 返回的 phase 开始 switch。
```

### 4.2 participants：这一轮要等多少人

`participants` 表示当前 attached party 大小。

static party：

```text
BarrierInit(&barrier, n):
  participants = n
  static_party = true
  后续不允许 Attach / Detach。
```

dynamic party：

```text
BarrierInit(&barrier, 0):
  participants = 0
  static_party = false
  BarrierAttach() 增加 participants
  BarrierDetach() 减少 participants
```

Parallel Hash Join 的 barrier 基本都以 dynamic party 初始化：

```c
BarrierInit(&pstate->build_barrier, 0);
BarrierInit(&pstate->grow_batches_barrier, 0);
BarrierInit(&pstate->grow_buckets_barrier, 0);
BarrierInit(&shared->batch_barrier, 0);
```

原因是参与者不是在 barrier 初始化时一次性固定的。worker 进入某个阶段、某个 batch、某个 growth cycle 的时间不同。

### 4.3 arrived：当前 phase 已经到了几个人

`arrived` 只对当前 phase 有意义：

```text
每个 ArriveAndWait():
  arrived++

如果 arrived == participants:
  arrived = 0
  phase++
  唤醒等待者
```

所以 `arrived` 不能离开 `phase` 单独解释。

一个常见误读是：

```text
arrived 很大，说明整体已经做了很多工作。
```

正确理解是：

```text
arrived 只说明当前 phase 的 barrier 边界已有多少 attached participants 抵达；
phase 推进后它会归零。
```

### 4.4 elected：每个 phase 只选一个执行串行小任务

`BarrierArriveAndWait()` 返回 `bool`：

```text
true:
  当前 backend 被选中。

false:
  当前 backend 只是普通参与者。
```

这不是“最后一个永远特殊”的 API 花样，而是并行算法非常常见的需求：

```text
所有人都要等某个阶段完成；
但阶段边界上有一小段串行工作只能做一次。
```

Parallel Hash Join 的例子：

```text
PHJ_BUILD_ELECT:
  选一个 backend 初始化 shared batch state 和 batch 0 hash table。

PHJ_GROW_BATCHES_ELECT:
  选一个 backend 分配新一代 batches。

PHJ_GROW_BATCHES_DECIDE:
  选一个 backend 判断是否继续增长 batch。

PHJ_BATCH_ELECT:
  选一个 backend 为某个 batch 分配 hash table。
```

通常最后到达并推进 phase 的 backend 返回 `true`。但如果 phase 是因为某个参与者 detach 而推进，最后到达者可能不存在于等待集合里，于是 `elected` 字段允许被唤醒者中有一个补位成为 elected participant。

### 4.5 condition_variable：等待 phase 改变

`Barrier` 内嵌一个 `ConditionVariable`：

```text
等待者睡眠条件:
  barrier->phase 仍是我进入等待时看到的 start_phase。

唤醒条件:
  最后到达者或 detach 者推进了 phase，
  对 condition_variable 做 Broadcast。
```

`ConditionVariable` 不知道 phase 语义。它只提供：

```text
PrepareToSleep();
Sleep(wait_event_info);
Broadcast();
CancelSleep();
```

`Barrier` 在它之上定义：

```text
start_phase / next_phase；
arrived / participants；
phase advance；
election；
dynamic detach。
```

## 5. 主流程源码 walkthrough：BarrierArriveAndWait()

先看 `src/backend/storage/ipc/barrier.c` 的核心入口。

### 5.1 进入 barrier：记录当前 phase

`BarrierArriveAndWait()` 一开始在 spinlock 下读取当前阶段：

```c
SpinLockAcquire(&barrier->mutex);
start_phase = barrier->phase;
next_phase = start_phase + 1;
++barrier->arrived;
```

这里的时间线是：

```text
我到达当前 phase 的边界；
我希望等到 phase 从 start_phase 变成 next_phase；
我把 arrived 加一，表示自己已到达。
```

为什么需要 `start_phase`？

因为后面睡眠时不应该等待一个泛泛条件：

```text
等 barrier 被唤醒。
```

而应该等待具体条件：

```text
等 phase 不再是我进入时看到的 start_phase。
```

这就是 barrier 比 CV 多出来的状态含义。

### 5.2 最后一个到达者：推进 phase

如果当前到达者让 `arrived == participants`：

```c
if (barrier->arrived == barrier->participants)
{
    release = true;
    barrier->arrived = 0;
    barrier->phase = next_phase;
    barrier->elected = next_phase;
}
```

状态变化可以画成：

```text
phase = P
participants = N
arrived = N - 1

当前 backend 到达:
  arrived = N
  arrived == participants

推进:
  arrived = 0
  phase = P + 1
  elected = P + 1
```

随后释放 spinlock，再唤醒其他等待者：

```c
if (release)
{
    ConditionVariableBroadcast(&barrier->condition_variable);
    return true;
}
```

注意顺序：

```text
先在 spinlock 下改变 phase；
释放 spinlock；
再 Broadcast。
```

等待者醒来后重新读取 `phase`，看到阶段已经推进。

### 5.3 不是最后一个：睡到 phase 改变

如果不是最后一个到达者：

```c
ConditionVariablePrepareToSleep(&barrier->condition_variable);
for (;;)
{
    SpinLockAcquire(&barrier->mutex);
    release = barrier->phase == next_phase;
    ...
    SpinLockRelease(&barrier->mutex);

    if (release)
        break;

    ConditionVariableSleep(&barrier->condition_variable, wait_event_info);
}
ConditionVariableCancelSleep();
```

这段有几个重点。

第一，等待者不是睡到“有人 Broadcast”：

```text
Broadcast 只是唤醒提示；
真实条件是 phase == next_phase。
```

第二，`ConditionVariableSleep()` 允许 spurious wakeup：

```text
醒来不代表 phase 已推进；
必须回到 spinlock 下重新检查 phase。
```

第三，注释中有一个非常重要的不变量：

```text
phase 只能是 start_phase 或 next_phase。
```

为什么不会跳到更远的 phase？

因为当前 backend 已经 attached，而且它还没参与下一轮 barrier。只要它不再次 `ArriveAndWait()`，其他参与者就无法越过下一轮需要它参与的边界。

这条不变量支撑了 `BarrierPhase()` 的无锁读取，也支撑了 parallel hash join 用 phase 做 shared program counter。

### 5.4 detach 推进时的补位 election

等待循环里有一段容易被忽略：

```c
if (release && barrier->elected != next_phase)
{
    barrier->elected = barrier->phase;
    elected = true;
}
```

通常 phase advance 由最后一个 `ArriveAndWait()` 的 backend 完成，它会设置：

```text
elected = next_phase
return true
```

但 dynamic party 中，某个参与者可能 detach：

```text
participants 减少；
原来等待 N 人，现在只需要 N-1 人；
已经 arrived 的人数可能足够推进 phase。
```

这种情况下推进 phase 的 backend 不一定是一个正常等待返回者。于是醒来的等待者中需要选一个返回 `true`，继续执行“每个 phase 只做一次”的串行任务。

这就是 `elected` 字段的意义。

## 6. Attach / Detach：dynamic party 的关键

static barrier 很容易理解：初始化时固定人数，每轮等满。

PostgreSQL 的复杂场景主要来自 dynamic barrier：

```text
参与者可以晚到；
参与者可以提前离开；
离开可能改变“这一轮是否已经等够人”。
```

### 6.1 BarrierAttach()

`BarrierAttach()` 很短：

```c
Assert(!barrier->static_party);

SpinLockAcquire(&barrier->mutex);
++barrier->participants;
phase = barrier->phase;
SpinLockRelease(&barrier->mutex);

return phase;
```

它同时完成两件事：

```text
1. 把当前 backend 纳入后续阶段的 participants；
2. 返回当前 phase，让调用方同步本地状态机。
```

为什么不能只做第 1 件事？

因为 dynamic party 中 attach 不是“从算法开头加入”。比如某个 worker 进入一个 hash join batch 时，batch 可能已经处在：

```text
PHJ_BATCH_ELECT:
  还没人分配 hash table。

PHJ_BATCH_ALLOCATE:
  有人正在分配。

PHJ_BATCH_LOAD:
  正在并行加载 inner tuples。

PHJ_BATCH_PROBE:
  已经可以 probe。

PHJ_BATCH_FREE:
  这个 batch 已经完成。
```

所以调用方通常写成：

```c
switch (BarrierAttach(batch_barrier))
{
    case PHJ_BATCH_ELECT:
        ...
    case PHJ_BATCH_ALLOCATE:
        ...
    case PHJ_BATCH_LOAD:
        ...
    case PHJ_BATCH_PROBE:
        ...
}
```

这不是 C 语言技巧，而是 dynamic barrier 的使用协议：attach 返回的是 shared state machine 的当前位置。

### 6.2 BarrierDetach()

`BarrierDetach()` 调用内部的 `BarrierDetachImpl(barrier, false)`。

核心逻辑：

```c
--barrier->participants;

if ((arrive || barrier->participants > 0) &&
    barrier->arrived == barrier->participants)
{
    release = true;
    barrier->arrived = 0;
    ++barrier->phase;
}

last = barrier->participants == 0;
```

这段代码回答了一个很实际的问题：

```text
如果大家正在等某个 backend，
但这个 backend 退出了参与集合，
等待者是否还要继续等它？
```

答案是不应该。detach 减少 `participants` 后，如果已到达人数已经等于新的参与者数，就可以推进 phase。

时间线：

```text
phase = P
participants = 3
arrived = 2

第三个参与者不再参与，调用 Detach:
  participants = 2
  arrived == participants

BarrierDetachImpl:
  arrived = 0
  phase = P + 1
  Broadcast
```

所以 `BarrierDetach()` 不是普通引用计数 decrement。它可能改变算法阶段。

### 6.3 BarrierArriveAndDetach()

`BarrierArriveAndDetach()` 表示：

```text
我到达了当前 phase 的边界；
但下一轮我不再参与。
```

它调用：

```c
BarrierDetachImpl(barrier, true);
```

其中 `arrive = true` 允许一种情况：

```text
如果我 detach 后没有其他参与者了，
这次 arrive 仍然可以推进 phase。
```

Parallel Hash Join 用它表达：

```text
我完成了这个 shared object 的使用；
如果我是最后一个离开的参与者，
我负责把 phase 推到 FREE 并释放资源。
```

### 6.4 BarrierArriveAndDetachExceptLast()

这个 API 比名字更重要。

它表示：

```text
大家都到达边界；
除了最后一个到达者继续 attached 进入下一阶段，
其他人 detach。
```

代码逻辑：

```c
if (barrier->participants > 1)
{
    --barrier->participants;
    return false;
}

++barrier->phase;
return true;
```

它用于 wait-free election：

```text
不能再让所有人等待；
但需要选出最后一个人继续做单人阶段。
```

Parallel Hash Join 在 `PHJ_BATCH_PROBE` 之后用它选出一个 backend 做 unmatched scan：

```text
probe 阶段会向上层 emit tuple；
emit tuple 后不能保证所有参与者仍然同时活跃在这个节点；
所以不能在这里再用普通 ArriveAndWait() 等大家；
只能让除最后一个之外的人 detach，最后一个独自进入 scan 阶段。
```

这条边界是本节最重要的工程判断之一：

```text
barrier 可以同步阶段，但不能魔法般保证调用方还会回来。
```

## 7. Parallel Hash Join 的 shared state machine

`src/include/executor/hashjoin.h` 里，Parallel Hash Join 的共享状态包含多个 barrier：

```c
typedef struct ParallelHashJoinState
{
    ...
    Barrier build_barrier;
    Barrier grow_batches_barrier;
    Barrier grow_buckets_barrier;
    ...
} ParallelHashJoinState;
```

每个 batch 也有自己的 barrier：

```c
typedef struct ParallelHashJoinBatch
{
    dsa_pointer buckets;
    Barrier     batch_barrier;
    ...
} ParallelHashJoinBatch;
```

这些 barrier 分别表达不同层次的 shared program counter。

### 7.1 build_barrier：整体 build 阶段

`build_barrier` 的 phase 常量：

```c
#define PHJ_BUILD_ELECT       0
#define PHJ_BUILD_ALLOCATE    1
#define PHJ_BUILD_HASH_INNER  2
#define PHJ_BUILD_HASH_OUTER  3
#define PHJ_BUILD_RUN         4
#define PHJ_BUILD_FREE        5
```

语义：

```text
ELECT:
  选一个 backend 做 shared 初始化。

ALLOCATE:
  等初始化完成。

HASH_INNER:
  所有人并行扫描 inner relation 并插入 shared hash table / batch tuplestore。

HASH_OUTER:
  多 batch 场景下，预先 partition outer relation。

RUN:
  build 完成，可以 probe。

FREE:
  所有参与者都 detach，shared batch memory 可以释放。
```

### 7.2 grow_batches_barrier 与 grow_buckets_barrier

hash build 过程中，某个 backend 可能发现：

```text
batch 0 的内存预算快爆了，需要增加 nbatch；
hash bucket 装载因子太高，需要增加 nbuckets。
```

这时不能让一个 backend 自己偷偷改变布局。所有参与者必须停在同一个阶段边界，共同完成重新分配、repartition、reinsert。

`grow_batches_barrier` 的 phase 是循环使用的：

```c
#define PHJ_GROW_BATCHES_ELECT        0
#define PHJ_GROW_BATCHES_REALLOCATE   1
#define PHJ_GROW_BATCHES_REPARTITION  2
#define PHJ_GROW_BATCHES_DECIDE       3
#define PHJ_GROW_BATCHES_FINISH       4
#define PHJ_GROW_BATCHES_PHASE(n)     ((n) % 5)
```

为什么要取模？

因为增长可能发生多轮：

```text
第一次 repartition 后仍然太大；
第二次继续增加 batch；
每一轮都复用同一个 Barrier 的 phase counter。
```

`phase` 自身仍然单调递增，但调用方用 `% 5` 把它映射到一轮增长子状态。

### 7.3 batch_barrier：每个 batch 的 probe 生命周期

每个 batch 的 phase：

```c
#define PHJ_BATCH_ELECT      0
#define PHJ_BATCH_ALLOCATE   1
#define PHJ_BATCH_LOAD       2
#define PHJ_BATCH_PROBE      3
#define PHJ_BATCH_SCAN       4
#define PHJ_BATCH_FREE       5
```

一个 batch 的状态故事：

```text
ELECT:
  选一个 backend 分配该 batch 的 hash table。

ALLOCATE:
  等分配完成。

LOAD:
  所有人从 shared tuplestore 并行加载 inner tuples。

PROBE:
  该 batch 可以 probe 并向上层 emit tuple。

SCAN:
  right / full / right anti join 场景下，选一个 backend 扫描 unmatched inner tuples。

FREE:
  最后一个参与者释放 chunks / buckets。
```

这里的关键不是阶段名，而是每个阶段是否允许等待。

```text
ELECT / ALLOCATE / LOAD:
  还没向上层 emit tuple，可以安全 ArriveAndWait。

PROBE:
  会向上层返回 tuple，控制权可能离开这个节点；
  不能再假设所有参与者都会同时回来等待。

SCAN / FREE:
  通过 detach 系列 API wait-free 推进。
```

## 8. 主流程源码 walkthrough：从 parallel hash build 到 probe

下面跟一条真实主线。重点不是背 hash join，而是看 barrier 如何承载阶段推进。

### 8.1 DSM 初始化：barrier 从 0 participants 开始

`ExecHashJoinInitializeDSM()` 在 parallel context 的 DSM 中分配 `ParallelHashJoinState`：

```c
pstate = shm_toc_allocate(pcxt->toc, sizeof(ParallelHashJoinState));
...
pstate->nparticipants = pcxt->nworkers + 1;
LWLockInitialize(&pstate->lock, LWTRANCHE_PARALLEL_HASH_JOIN);
BarrierInit(&pstate->build_barrier, 0);
BarrierInit(&pstate->grow_batches_barrier, 0);
BarrierInit(&pstate->grow_buckets_barrier, 0);
```

注意 `pstate->nparticipants` 和 `Barrier.participants` 不是同一个语义：

```text
pstate->nparticipants:
  计划层面这次 parallel hash join 预期有多少参与者，
  用于内存预算、batch 数估算等。

Barrier.participants:
  当前真的 attached 到某个 barrier 的参与者数量，
  是 runtime 动态集合。
```

这就是为什么 barrier 初始化为 0。并不是 PostgreSQL 不知道 worker 数，而是算法允许参与者在不同时间 attach 到不同 barrier。

### 8.2 创建 hash table：先 attach build_barrier

`ExecHashTableCreate()` 中，如果是 parallel hash：

```c
build_barrier = &pstate->build_barrier;
BarrierAttach(build_barrier);

if (BarrierPhase(build_barrier) == PHJ_BUILD_ELECT &&
    BarrierArriveAndWait(build_barrier, WAIT_EVENT_HASH_BUILD_ELECT))
{
    pstate->nbatch = nbatch;
    pstate->space_allowed = space_allowed;
    pstate->growth = PHJ_GROWTH_OK;

    ExecParallelHashJoinSetUpBatches(hashtable, nbatch);
    pstate->nbuckets = nbuckets;
    ExecParallelHashTableAlloc(hashtable, 0);
}
```

这段体现了两个 barrier 用法。

第一，attach 后先读 phase：

```text
如果已经不是 PHJ_BUILD_ELECT，说明别的 backend 已经完成或正在完成初始化；
当前 backend 不应重复初始化。
```

第二，`ArriveAndWait()` 返回 true 的人执行串行初始化：

```text
shared batch state 只能分配一次；
batch 0 hash table 只能分配一次；
但所有人都需要等这个初始化边界过去。
```

这个场景如果只用 CV，会很容易退化成手写一套：

```text
bool initialized;
int arrived;
int participants;
wait list;
elected flag;
phase number;
```

`Barrier` 把这组状态收拢成一个 primitive。

### 8.3 MultiExecParallelHash()：按 build phase 跳到正确步骤

`MultiExecParallelHash()` 的开头非常典型：

```c
build_barrier = &pstate->build_barrier;
Assert(BarrierPhase(build_barrier) >= PHJ_BUILD_ALLOCATE);

switch (BarrierPhase(build_barrier))
{
    case PHJ_BUILD_ALLOCATE:
        BarrierArriveAndWait(build_barrier,
                             WAIT_EVENT_HASH_BUILD_ALLOCATE);
        pg_fallthrough;

    case PHJ_BUILD_HASH_INNER:
        ...
}
```

为什么这里是 `switch (BarrierPhase(...))`？

因为当前 backend 到这里时，其他 backend 可能已经推进过一些阶段。它不能假设自己从 `PHJ_BUILD_ALLOCATE` 开始。

`pg_fallthrough` 也不是代码风格偶然：

```text
如果我刚完成 PHJ_BUILD_ALLOCATE，就自然进入 HASH_INNER；
如果我 attach 时已经处在 HASH_INNER，也直接执行 HASH_INNER；
如果未来某个 phase 已经更晚，则 switch 会跳过更早阶段。
```

这就是 shared program counter 和 local program counter 对齐的代码形态。

### 8.4 HASH_INNER：所有人并行工作，然后 barrier 对齐

在 `PHJ_BUILD_HASH_INNER` 阶段，所有参与者扫描 inner plan 并插入 shared hash table 或写入 batch tuplestore。

期间可能触发增长：

```c
if (PHJ_GROW_BATCHES_PHASE(BarrierAttach(&pstate->grow_batches_barrier)) !=
    PHJ_GROW_BATCHES_ELECT)
    ExecParallelHashIncreaseNumBatches(hashtable);

if (PHJ_GROW_BUCKETS_PHASE(BarrierAttach(&pstate->grow_buckets_barrier)) !=
    PHJ_GROW_BUCKETS_ELECT)
    ExecParallelHashIncreaseNumBuckets(hashtable);
```

这说明一个 backend 可能在增长操作已经进行到一半时加入：

```text
它 attach 后读取 grow barrier 的当前 phase；
如果不是 ELECT，说明已有增长周期正在进行；
它必须立即参与完成这个周期，才能安全访问 batches / buckets。
```

完成 inner build 后：

```c
for (i = 0; i < hashtable->nbatch; ++i)
    sts_end_write(hashtable->batches[i].inner_tuples);

ExecParallelHashMergeCounters(hashtable);

BarrierDetach(&pstate->grow_buckets_barrier);
BarrierDetach(&pstate->grow_batches_barrier);

if (BarrierArriveAndWait(build_barrier,
                         WAIT_EVENT_HASH_BUILD_HASH_INNER))
{
    pstate->growth = PHJ_GROWTH_DISABLED;
}
```

这里有两层同步：

```text
grow_*_barrier:
  build 期间临时参与的增长子状态机；
  inner build 完成时 detach。

build_barrier:
  整体 build 阶段边界；
  所有人必须完成 inner relation hashing 和 shared tuplestore flush 后，
  才能进入下一阶段。
```

被 elected 的 backend 把 `pstate->growth` 置为 `PHJ_GROWTH_DISABLED`，表示 batches 已固定。这个动作必须发生在所有人完成 hashing 之后，否则仍可能有人按旧 growth 状态修改布局。

### 8.5 batch probe：late joiner 根据 batch phase 跳转

`ExecParallelHashJoinNewBatch()` 会寻找未完成的 batch，并 attach 到该 batch 的 barrier：

```c
switch (BarrierAttach(batch_barrier))
{
    case PHJ_BATCH_ELECT:
        if (BarrierArriveAndWait(batch_barrier,
                                 WAIT_EVENT_HASH_BATCH_ELECT))
            ExecParallelHashTableAlloc(hashtable, batchno);
        pg_fallthrough;

    case PHJ_BATCH_ALLOCATE:
        BarrierArriveAndWait(batch_barrier,
                             WAIT_EVENT_HASH_BATCH_ALLOCATE);
        pg_fallthrough;

    case PHJ_BATCH_LOAD:
        ...
        BarrierArriveAndWait(batch_barrier,
                             WAIT_EVENT_HASH_BATCH_LOAD);
        pg_fallthrough;

    case PHJ_BATCH_PROBE:
        ExecParallelHashTableSetCurrentBatch(hashtable, batchno);
        sts_begin_parallel_scan(...);
        return true;
}
```

这条链路很适合作为 barrier 阅读练习。

`PHJ_BATCH_ELECT`：

```text
所有 attach 到这个 batch 的参与者到达 barrier；
只有一个 true 返回者分配 hash table。
```

`PHJ_BATCH_ALLOCATE`：

```text
其他参与者等待分配完成。
```

`PHJ_BATCH_LOAD`：

```text
所有参与者并行从 shared tuplestore 加载 inner tuples；
再用 barrier 等所有人加载完成。
```

`PHJ_BATCH_PROBE`：

```text
hash table 已完整；
当前 backend 可以返回 true，让上层开始 probe 并 emit tuple。
```

注意这里 `return true` 之前没有 detach：

```text
backend 仍 attached 到 batch_barrier；
这样 batch 的 shared chunks / buckets 不能被别人释放。
```

但进入 `PHJ_BATCH_PROBE` 后，不能再用普通 `ArriveAndWait()` 做下一阶段同步，因为当前 backend 可能向上层返回 tuple，甚至被 Limit、错误、取消、join node 生命周期等控制流影响。

## 9. Phase 边界、ownership 与 cleanup

Barrier 自身不拥有业务资源。它只拥有：

```text
内部状态字段；
一个 ConditionVariable wait list。
```

业务资源 ownership 在调用方。

Parallel Hash Join 中的典型资源包括：

```text
DSM 中的 ParallelHashJoinState；
DSA 中的 batches 数组；
每个 batch 的 buckets；
tuple chunks；
SharedTuplestore 临时文件；
backend-local batch accessors。
```

Barrier 的作用是定义“什么时候释放这些资源是安全的”。

### 9.1 build_barrier 的最后释放

`ExecHashTableDetach()` 中：

```c
if (pstate && BarrierPhase(&pstate->build_barrier) == PHJ_BUILD_RUN)
{
    ...
    if (BarrierArriveAndDetach(&pstate->build_barrier))
    {
        Assert(BarrierPhase(&pstate->build_barrier) == PHJ_BUILD_FREE);

        if (DsaPointerIsValid(pstate->batches))
        {
            dsa_free(hashtable->area, pstate->batches);
            pstate->batches = InvalidDsaPointer;
        }
    }
}
```

语义：

```text
每个参与者完成 probe 后 arrive-and-detach；
如果我是最后一个 detach 的参与者，
build_barrier 推进到 PHJ_BUILD_FREE；
这时 late joiner 会看到太晚，不能再访问 batch shared memory；
我可以释放 pstate->batches。
```

这里的 `true` 返回值就是资源 cleanup ownership 的转交。

### 9.2 batch_barrier 的最后释放

`ExecHashTableDetachBatch()` 中，当前 batch 可能还在 `PHJ_BATCH_PROBE` 或已经到 `PHJ_BATCH_SCAN`。

如果还在 probe：

```c
attached = BarrierArriveAndDetachExceptLast(&batch->batch_barrier);
```

如果当前 backend 不是最后一个：

```text
它 detach；
这个 batch 对它来说完成；
它不能释放 shared buckets，因为还有别人 attached。
```

如果它是最后一个：

```text
phase 推进到 PHJ_BATCH_SCAN；
它继续 attached，独自执行 unmatched scan 或跳过；
随后调用 BarrierArriveAndDetach() 进入 PHJ_BATCH_FREE；
如果返回 true，释放 batch chunks / buckets。
```

这条链路的本质：

```text
probe 阶段之后，不能再等待大家；
但还必须保证只有最后一个人释放资源。
```

`ArriveAndDetachExceptLast()` 刚好表达这个边界。

### 9.3 ERROR / cancel 下的边界

Barrier 本身不像 LWLock 那样有一个全局 `held_lwlocks` cleanup 栈。调用方必须在 executor 清理路径中 detach。

Parallel Hash Join 的清理路径会关闭 shared tuplestore scan / write，并通过 detach 推进资源生命周期：

```text
ExecHashTableDetachBatch():
  当前 batch 的 barrier detach；
  必要时释放 batch chunks / buckets。

ExecHashTableDetach():
  build_barrier arrive-and-detach；
  最后一个释放 batches 数组。

ExecHashJoinReInitializeDSM():
  rescan 前先 detach 旧 hash table；
  删除 shared fileset；
  重新初始化 build_barrier。
```

所以读 barrier 调用方时，一定要找：

```text
attach 在哪里？
正常 detach 在哪里？
提前结束 / rescan / abort 时是否也能 detach？
最后一个 detach 是否负责 free？
```

只看 `BarrierArriveAndWait()` 很容易错过真正的资源生命周期。

## 10. 正确性机制层次

Barrier 的正确性来自几层机制叠加。

### 10.1 内部字段由 spinlock 串行化

`phase`、`participants`、`arrived`、`elected` 的写入都在 `barrier->mutex` 下完成。

这些临界区非常短：

```text
读写几个 int；
判断是否 release；
不做内存分配；
不做 I/O；
不调用 sleep。
```

这符合前面 spinlock 课程的边界。

### 10.2 等待由 ConditionVariable 承载

等待者不会持有 spinlock 睡眠。

流程是：

```text
spinlock 下检查 phase；
释放 spinlock；
ConditionVariableSleep();
醒来后重新抢 spinlock 检查 phase。
```

这继承了 CV 的规则：

```text
允许 spurious wakeup；
Sleep 返回只代表“应该重查”；
真实条件仍是 phase 是否推进。
```

### 10.3 phase 防止 late joiner 误执行旧阶段

dynamic barrier 中，`BarrierAttach()` 返回当前 `phase`。

调用方必须把它当作共享状态机位置：

```text
如果 phase 已经过了 allocate，不要重复 allocate；
如果 phase 已经过了 load，不要重复 load；
如果 phase 已到 free，不要再访问已释放资源。
```

这不是性能优化，而是 correctness。

### 10.4 participants / arrived 保证群体边界

`participants` 和 `arrived` 的组合保证：

```text
除非所有 attached participants 到达或 detach，
phase 不会推进。
```

它同时允许 dynamic party：

```text
如果某人 detach，participants 降低；
等待集合变小，可能立即满足 arrived == participants；
phase 可以推进。
```

### 10.5 elected 保证串行任务只做一次

很多阶段边界需要“选一个人”：

```text
分配 shared batches；
分配 batch hash table；
判断是否继续增长；
释放 shared memory。
```

`BarrierArriveAndWait()` 的 bool 返回值让调用方不必额外维护一次性 flag。

但要注意：

```text
elected 只保证这一轮 barrier 返回 true 的 backend 至多一个；
它不自动保护被 elected backend 访问的业务状态。
业务状态仍要靠调用方协议、LWLock、phase 边界或单线程阶段保证。
```

### 10.6 WaitEvent 让等待可观测

`BarrierArriveAndWait()` 接收 `wait_event_info`：

```c
BarrierArriveAndWait(build_barrier,
                     WAIT_EVENT_HASH_BUILD_HASH_INNER);
```

实际等待通过 `ConditionVariableSleep()` 和 `WaitLatch()`，但 `pg_stat_activity` 看到的是调用方传入的业务等待点：

```text
HashBuildHashInner
HashBatchLoad
HashGrowBatchesRepartition
```

这对诊断很重要。用户不需要看到“某个 CV 在睡”，而要看到“Parallel Hash 正在等哪个阶段”。

## 11. 错误路径 / 异常路径 / fallback

### 11.1 不能在会 emit tuple 的阶段后继续等待

`nodeHashjoin.c` 的注释明确说，为避免死锁：

```text
never wait for any barrier unless it is known that all other backends
attached to it are actively executing the node or have finished.
```

翻译成执行器语境：

```text
如果某个 backend 已经向上层返回 tuple，
它可能不会马上回到这个节点；
甚至可能因为上层 Limit 已满足、查询取消、错误路径而不再执行到下一次 wait。
```

因此 Parallel Hash Join 规定：

```text
PHJ_BUILD_RUN:
  可以 emit tuple，但之后 build_barrier 不再做普通等待；
  用 BarrierArriveAndDetach() 进入 FREE。

PHJ_BATCH_PROBE:
  可以 emit tuple，但之后 batch_barrier 不再做普通等待；
  用 ArriveAndDetachExceptLast() 做 wait-free election。
```

这是 barrier 使用中的关键异常边界。

### 11.2 skip_unmatched：提前结束 probe 的正确性处理

如果一个 backend 在 `PHJ_BATCH_PROBE` 阶段没有扫完整个 outer batch 就结束：

```text
full / right join 的 unmatched inner tuple 标记可能不完整；
不能再让某个 backend 扫描并输出 unmatched tuples。
```

`ExecHashTableDetachBatch()` 中会设置：

```c
batch->skip_unmatched = true;
```

注释说明这个 flag 可能被多个 backend 写：

```text
但只会在 PHJ_BATCH_SCAN 阶段读取；
phase 边界保证读发生在所有 probe 写之后。
```

这是一个非常典型的 barrier pattern：

```text
某些共享字段不靠单独锁保护；
它们的可见和解释依赖 phase 边界。
```

### 11.3 资源释放由最后一个 detach 承担

如果没有 `BarrierArriveAndDetach()` 的 true 返回值，释放 shared memory 会很麻烦：

```text
释放早了:
  其他 attached backend 还可能访问。

释放晚了:
  DSM / DSA 资源泄漏或等待外层粗暴回收。

另设 refcount:
  又要重新实现 participants / detach / wakeup 逻辑。
```

Barrier 把它统一成：

```text
participants 降到 0 的那个 backend 返回 true；
调用方在这个分支里 free。
```

### 11.4 rescan 重新初始化

`ExecHashJoinReInitializeDSM()` 在重新扫描前会：

```text
detach 当前 hash table / batch；
删除 shared fileset；
重新 BarrierInit(&pstate->build_barrier, 0)。
```

这说明 barrier 的 `phase` 不是永久 epoch，也不跨 executor 生命周期复用语义。它属于某一轮 shared algorithm instance。

如果调用方想复用同一块 shared memory，必须明确重置 barrier，并确保旧参与者已经离开。

## 12. 成本、资源与跨模块传播

Barrier 的成本来自两个层面。

第一，状态推进本身很便宜：

```text
一次 spinlock；
几个 int 更新；
必要时一次 ConditionVariableBroadcast。
```

第二，等待代价取决于算法阶段：

```text
如果所有参与者进度接近，barrier 基本只是短暂对齐；
如果某个 worker 被调度延迟、I/O 慢、数据倾斜严重，
其他 worker 会在 barrier 等待，CPU 并行度下降。
```

Parallel Hash Join 中常见等待点：

```text
WAIT_EVENT_HASH_BUILD_ELECT
WAIT_EVENT_HASH_BUILD_ALLOCATE
WAIT_EVENT_HASH_BUILD_HASH_INNER
WAIT_EVENT_HASH_BUILD_HASH_OUTER
WAIT_EVENT_HASH_BATCH_ELECT
WAIT_EVENT_HASH_BATCH_ALLOCATE
WAIT_EVENT_HASH_BATCH_LOAD
WAIT_EVENT_HASH_GROW_BATCHES_REPARTITION
WAIT_EVENT_HASH_GROW_BUCKETS_REINSERT
```

这些等待不是“锁竞争”那么简单。它们可能来自：

```text
数据倾斜:
  某些 worker 分到更多 tuple 或更大的 batch。

I/O:
  shared tuplestore spill / load 不均衡。

内存预算:
  hash_mem 不足触发更多 batch growth。

调度:
  worker 数、CPU 数、OS scheduling 影响到达 barrier 的时间。

执行器控制流:
  上层节点消费 tuple 的方式影响 probe 阶段 detach 时机。
```

跨模块传播也很明显：

```text
planner:
  决定是否使用 Parallel Hash Join、worker 数和初始 batch / bucket 估算。

executor:
  用 barrier 管理 shared state machine。

DSM / DSA:
  存放 barrier 和 hash join shared objects。

SharedTuplestore:
  batch spill / load 的共享数据通道。

pg_stat_activity:
  暴露 barrier 等待阶段。
```

因此诊断 barrier 等待时，不能只盯着 `barrier.c`。它通常只是症状汇聚点。

## 13. 观测与诊断入口

### 13.1 用 EXPLAIN 确认 Parallel Hash Join

先确认计划真的使用了 parallel-aware hash join：

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT ...
```

观察计划中是否有：

```text
Parallel Hash Join
Parallel Hash
Workers Planned / Workers Launched
Batches
Buckets
Memory Usage
```

如果计划是普通 Hash Join 或每个 worker 自己建 hash table，本文 barrier 主线不适用。

### 13.2 用 pg_stat_activity 看等待阶段

在查询运行时观察：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event LIKE 'Hash%';
```

可能看到：

```text
HashBuildHashInner:
  等其他参与者完成 inner relation hashing。

HashBatchLoad:
  等同一 batch 的其他参与者完成加载。

HashGrowBatchesRepartition:
  等 repartition 完成。

HashGrowBucketsReinsert:
  等 reinsert 完成。
```

解释规则：

```text
看到 Hash* wait event，不等于 barrier 本身慢；
它说明某个 phase 的最慢参与者还没到达。
```

### 13.3 gdb 断点观察 phase 推进

可以在开发环境中打断点：

```gdb
break BarrierArriveAndWait
break BarrierDetachImpl
commands
  silent
  printf "barrier=%p phase=%d participants=%d arrived=%d\n", barrier, barrier->phase, barrier->participants, barrier->arrived
  continue
end
```

观察重点：

```text
arrived 如何从 0 增到 participants；
最后一个到达者如何把 phase 加一；
detach 是否导致 phase advance；
同一个 grow barrier 的 phase 是否多轮递增。
```

### 13.4 源码实验：人为放慢某个阶段

在本地开发库里，可以临时在某个 elected 分支或某个 worker 路径中插入 sleep，例如：

```c
if (BarrierArriveAndWait(batch_barrier,
                         WAIT_EVENT_HASH_BATCH_ELECT))
{
    pg_usleep(5 * 1000000L);
    ExecParallelHashTableAlloc(hashtable, batchno);
}
```

然后观察其他 worker 的 wait event 是否停在 `HashBatchElect` 或 `HashBatchAllocate`。

实验结束要删除 sleep。这个实验的目标不是改功能，而是建立：

```text
phase 中最慢的参与者决定其他人的等待时间。
```

## 14. 常见误区

误区一：把 `Barrier` 当成 CPU memory barrier。

```text
IPC Barrier:
  多进程阶段同步，有 participants / arrived / phase。

Memory barrier:
  控制 CPU / compiler 内存访问重排序。
```

本节讲的是前者。

误区二：认为 `BarrierArriveAndWait()` 返回 true 一定代表“我是最后一个到达的人”。

更准确的说法是：

```text
通常最后到达者返回 true；
但如果 phase 因 detach 推进，某个被唤醒者也可能被 elected 返回 true。
```

误区三：认为 `BarrierPhase()` 无锁读取在任何时候都安全。

源码注释的前提是：

```text
caller must be attached。
```

未 attach 的 backend 不能假设 phase 不会在自己脚下变化。

误区四：认为 dynamic barrier 的 participants 等于计划 worker 数。

实际是：

```text
participants 是当前 attached 到这个 barrier 的 runtime 集合；
worker 数只是 parallel plan 的资源背景。
```

误区五：认为所有 barrier 阶段都可以 `ArriveAndWait()`。

Parallel Hash Join 明确避免在可能 emit tuple 后等待：

```text
PHJ_BUILD_RUN / PHJ_BATCH_PROBE 之后用 detach 系列 API 推进，
避免等待不会再回来的参与者。
```

误区六：只看 `barrier.c`，不看调用方 cleanup。

`barrier.c` 不知道业务资源是什么。真正的安全释放必须在调用方通过 phase / last-detach 分支实现。

## 15. 课堂实验

### 实验 1：手动画出 BarrierArriveAndWait() 状态表

给定：

```text
phase = 7
participants = 3
arrived = 0
```

三个 backend 依次调用 `BarrierArriveAndWait()`。

记录每一步：

```text
调用者
start_phase
arrived
phase
是否 release
是否 Broadcast
返回值
```

然后改成第三个 backend 在到达前调用 `BarrierDetach()`，再重新推演。

目标：理解 detach 为什么可能推进 phase。

### 实验 2：跟踪 batch_barrier 的 late joiner

阅读：

```text
src/backend/executor/nodeHashjoin.c
  ExecParallelHashJoinNewBatch()
```

回答：

```text
如果 BarrierAttach(batch_barrier) 返回 PHJ_BATCH_LOAD，
当前 backend 会跳过哪些阶段？
它会参与哪些工作？
它何时开始 probe？
```

目标：理解 dynamic phase 如何同步本地状态机。

### 实验 3：解释 wait-free election

阅读：

```text
src/backend/executor/nodeHash.c
  ExecParallelPrepHashTableForUnmatched()
  ExecHashTableDetachBatch()
```

回答：

```text
为什么 PHJ_BATCH_PROBE 后不能再 ArriveAndWait？
BarrierArriveAndDetachExceptLast() 如何选出唯一继续 scan 的 backend？
非最后一个 backend 做了哪些 cleanup？
```

目标：理解 barrier 的“可等待阶段”和“不可等待阶段”边界。

### 实验 4：构造 Parallel Hash Join 观测

准备两张较大的表，并调低单次 hash 内存，让执行更容易进入 multi-batch：

```sql
SET max_parallel_workers_per_gather = 4;
SET enable_parallel_hash = on;
SET work_mem = '4MB';

EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
SELECT count(*)
FROM big_a a
JOIN big_b b ON a.k = b.k;
```

并在另一个 session 中观察：

```sql
SELECT pid, wait_event_type, wait_event, state
FROM pg_stat_activity
WHERE wait_event LIKE 'Hash%';
```

目标：把 `Hash*` wait event 回扣到 `BarrierArriveAndWait()` 的 `wait_event_info`。

### 实验 5：源码断点观察 grow barrier 循环 phase

在调试构建中对下面函数打断点：

```text
ExecParallelHashIncreaseNumBatches
BarrierArriveAndWait
```

观察：

```text
grow_batches_barrier 的 phase 单调递增；
PHJ_GROW_BATCHES_PHASE(phase) 用取模映射到循环子阶段；
每一轮增长中哪些阶段选一个 backend，哪些阶段所有人参与。
```

目标：理解 `phase` 既是 generation，也是调用方状态机输入。

## 16. 讨论题

1. 为什么 `Barrier` 不直接保存“当前阶段的业务对象是否已经准备好”？

2. 如果 `BarrierAttach()` 不返回当前 phase，Parallel Hash Join 的 late joiner 需要额外维护哪些状态？

3. 为什么 `BarrierPhase()` 的注释强调 caller must be attached？如果不 attached，会出现什么竞态？

4. 为什么 `PHJ_BATCH_PROBE` 可以向上层 emit tuple 后，不能再用普通 barrier wait？

5. `BarrierArriveAndWait()` 的 true 返回值和 “leader / worker” 身份是否有关？为什么 PostgreSQL 选择任意 participant election？

6. Parallel Hash Join 中哪些共享字段依赖 phase 边界解释，而不是每次读写都单独上锁？

7. 如果某个 worker 长时间停在 `HashBuildHashInner`，你会从数据倾斜、I/O、worker 调度、内存预算四个方向分别如何排查？

## 17. 本节小结

`Barrier` 是 PostgreSQL 在 `ConditionVariable` 之上构建的多进程阶段同步 primitive。它不保护任意业务数据，也不替代 LWLock；它维护的是：

```text
谁参与这一轮；
谁已经到达；
当前算法推进到哪个 phase；
phase 推进后叫醒哪些等待者；
哪个参与者被选中执行一次性串行工作。
```

本节最重要的源码规律是：

```text
phase 是 shared program counter；
participants / arrived 是推进这个 program counter 的 runtime party 集合；
ConditionVariable 是等待 phase 改变的睡眠机制；
detach 是 dynamic party 正确性的核心，不是普通 cleanup 细节。
```

Parallel Hash Join 展示了这个规律的完整形态：

```text
build_barrier:
  对齐整体 build 生命周期。

grow_batches_barrier / grow_buckets_barrier:
  在 hash build 中临时对齐 shared layout 变更。

batch_barrier:
  对齐每个 batch 的 allocate / load / probe / scan / free。
```

最后要记住一个边界：

```text
Barrier 只适合参与者会按协议到达或 detach 的阶段。
一旦调用方可能向上层 emit tuple、被取消或不再返回这个节点，
就不能再设计“大家回来一起 wait”的协议，
必须用 detach / last participant / phase free 这类 wait-free 结束路径。
```

这条规律会在下一节“同步原语组合”中继续出现：PostgreSQL 很少孤立使用某一种 primitive，真正的内核设计通常是把 spinlock、condition variable、barrier、LWLock、latch 和业务状态边界组合成一个可恢复、可诊断的 runtime 协议。
