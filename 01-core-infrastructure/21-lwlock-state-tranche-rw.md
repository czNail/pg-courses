# PostgreSQL LWLock state、tranche 与共享读写互斥

## 课程定位

前置知识：已经理解 PostgreSQL 多进程共享内存模型，知道 `PGPROC`、`ProcArray`、shared buffer mapping table 等结构会被多个 backend 同时访问；已经读过上一节 SpinLock，明白 spinlock 只适合保护几条指令级的 shared state 更新。

本节唯一主问题：

```text
为什么很多 shared memory 结构不能只靠 spinlock，而需要支持 shared / exclusive 模式、wait event 命名和 cache-line padding 的 LWLock，LWLockAttemptLock() 如何用一个 atomic state 表达读者数、独占持有者和等待标志？
```

核心矛盾：很多共享结构既需要高频读取，又需要偶发修改。读者之间本来可以并发，如果用 spinlock 或单一 mutex 串行化所有访问，会把“没有语义冲突的读”也变成 cache line 争用；但如果把读者计数、独占持有者、等待者标记、内部 wait-list mutex 分散成多个字段，又会让 fast path 变重，甚至重新回到“为了拿一个 shared lock 先抢一个 spinlock”的旧问题。PostgreSQL 的解法是：

```text
把 LWLock 的核心状态压进一个 pg_atomic_uint32 state
  -> 低位表达 shared holder 数或 exclusive sentinel
  -> 高位表达 has waiters / wake in progress / wait-list locked
  -> shared acquire 在没有 exclusive holder 时只做 atomic CAS 加一
  -> exclusive acquire 在没有任何 holder 时写入 sentinel
  -> tranche 只负责身份、命名和观测，不进入互斥判定
```

学完后应能判断：

```text
为什么 LWLock 不是 heavyweight lock，也不是“可以睡眠的 spinlock”；
为什么 shared buffer mapping、ProcArray、lock manager partition 这类结构需要 shared / exclusive；
为什么 LWLock 的 fast path 不能先抢内部 spinlock；
为什么 LWLockAttemptLock() 总是执行 compare-and-exchange，即使已经判断锁不空闲；
为什么 LWLockPadded 要把常用 LWLock 隔离到 cache line；
为什么 wait_event 看到的是 tranche 名，而不是某个 C 指针；
为什么本节只分析 atomic state，完整睡眠队列和 ERROR-safe release 要放到下一节。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节的 spinlock 给出一个非常窄的边界：

```text
只保护几个共享字段的瞬时状态跃迁；
持锁期间不能做 I/O、不能等待、不能走复杂 ERROR 路径。
```

这个边界很快会遇到现实压力。以 buffer mapping table 为例：

```text
多数 ReadBuffer 路径只想查一个 BufferTag 是否已经在 shared buffers 中；
多个 backend 查不同或相同 tag 时，读者之间没有必要互斥；
但插入、删除 mapping 时又必须排斥读者，防止读到半更新状态。
```

这类结构需要的不是 spinlock，而是 reader-writer style 的短时共享内存锁：

```text
shared mode:
  多个读者可以同时进入。

exclusive mode:
  修改者独占进入，排斥所有读者和其它修改者。
```

PostgreSQL 把这个角色交给 `LWLock`。它处在同步原语谱系的中间：

| 原语 | 适合保护什么 | 等待方式 | 本组课程位置 |
| --- | --- | --- | --- |
| SpinLock | 几个共享字段的瞬时更新。 | 忙等 + 短退避。 | 已讲。 |
| LWLock | shared memory 结构的一段短访问，支持读写模式。 | fast path atomic；慢路径进入等待队列并睡眠。 | 本节与下一节。 |
| Latch | 一个进程等待异步事件。 | 等待 latch / socket / timeout。 | 后续。 |
| Condition Variable | 等待某个共享谓词变真。 | 谓词循环 + latch/sem 唤醒。 | 后续。 |
| Barrier | 多个进程等待阶段推进。 | 计数 + condition variable。 | 后续。 |

本节只回答 LWLock 的第一层问题：

```text
一个 LWLock 如何用一个 atomic state 支持 shared / exclusive acquire？
```

下一节再回答：

```text
拿不到锁时，为什么 LWLockAcquire() 必须先尝试、入队、再尝试、再睡眠？
```

这两个问题故意拆开。否则容易把 `LWLockAttemptLock()` 的 fast path 和 `PGPROC.lwWaiting`、process semaphore、`LW_FLAG_WAKE_IN_PROGRESS` 的 missed wakeup 协议混在一起，读者会看见很多机制，却抓不住 state 模型。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
LWLockAcquire(lock, mode) 先调用 LWLockAttemptLock()；
LWLockAttemptLock() 读取 lock->state，根据 mode 构造 desired_state；
shared 模式只要求没有 exclusive sentinel，成功后 shared holder 计数加 1；
exclusive 模式要求没有 shared holder、没有 exclusive sentinel，成功后写入 exclusive sentinel；
atomic compare-and-exchange 同时完成“检查旧状态”和“发布新状态”，并作为 acquisition memory barrier；
如果 state 显示冲突，函数不睡眠，只返回 mustwait=true，把等待问题交还给 LWLockAcquire() 慢路径。
```

这背后的 tension 是：

```text
读多写少的共享结构需要让读者并发；
但读锁 acquire 自身不能重到每次都抢一个内部锁。
```

`src/backend/storage/lmgr/lwlock.c` 顶部注释直接记录了这段演化：旧实现更像 straight-forward reader-writer lock，内部状态由 spinlock 保护；问题是很多 workload 会非常频繁地拿 shared lock，结果大家在一个“显然是独占的 spinlock”上自旋，即使目标 LWLock 实际上没有 exclusive holder。当前实现改为：

```text
single atomic state
  + compare-and-exchange
  + shared holder count
  + exclusive sentinel
  + high-bit flags
```

因此，LWLock 的核心不是“一个结构体里有等待队列”。

更准确的模型是：

```text
fast path:
  用 atomic state 判断读写冲突，并在同一个 CAS 中登记自己成为 holder。

slow path:
  fast path 失败后，才使用 waiters 链表、PGPROC 和 semaphore 睡眠。
```

本节聚焦 fast path，因为这是理解 LWLock 成本边界的关键。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/lwlock.h` | `LWLock` 结构、`LWLockMode`、`LWLockPadded`、builtin tranche ID 边界。 |
| 2 | `src/backend/storage/lmgr/lwlock.c` | `state` 位布局、`LWLockAttemptLock()`、初始化、tranche 名称、wait event 上报入口。 |
| 3 | `src/include/storage/lwlocklist.h` | 预定义 LWLock 与内置 tranche 列表，理解 wait_event 名称来自哪里。 |
| 4 | `src/backend/utils/activity/wait_event_names.txt` | `WaitEventLWLock` 文档源，确认 `ProcArray`、`WALWrite`、`BufferMapping` 等观测名。 |
| 5 | `src/backend/storage/buffer/buf_table.c` | buffer mapping table 的 shared lookup / exclusive insert/delete 边界。 |
| 6 | `src/include/storage/buf_internals.h` | `BufMappingPartitionLock()` 如何把 hash partition 映射到一组 padded LWLock。 |
| 7 | `src/backend/storage/ipc/procarray.c` | `ProcArrayLock` 如何用 exclusive mode 修改数组成员，用 shared mode 支撑 snapshot / horizon 读取。 |

推荐阅读顺序：

```text
先读 lwlock.h 的结构体和 mode
  -> 回到 lwlock.c 顶部注释，理解为什么移除内部 spinlock fast path
  -> 读 state 位布局宏
  -> 精读 LWLockAttemptLock()
  -> 再看 lwlocklist.h 与 wait_event_names.txt，建立 tranche / 观测名模型
  -> 最后用 BufferMapping 或 ProcArray 的调用点验证 shared / exclusive 的使用边界
```

不要一开始就从所有 `LWLockAcquire()` 调用点扫起。调用点太多，且它们保护的业务状态各不相同。本节要先建立一个可迁移的判断：

```text
只要读者之间不冲突，且临界区又比 spinlock 能承受的几条指令更长，
就应该问：这里是不是 LWLock shared/exclusive 的形状？
```

## 4. 关键数据结构与状态

### `LWLock`: 语义集中在 `state`

`LWLock` 定义在 `src/include/storage/lwlock.h`：

```text
tranche:
  这个锁属于哪个命名类别，用于 wait event、debug 和扩展注册。

state:
  32 位 atomic 状态，是 acquire/release fast path 的核心。

waiters:
  等待中的 PGPROC 链表。慢路径使用，本节只讲它和 state flags 的边界。
```

重要的是：`tranche` 不参与互斥判定。两个不同 LWLock 可以属于同一个 tranche，例如 128 个 `BufferMapping` partition lock 的 wait event 都显示为 `BufferMapping`。互斥语义来自具体 `LWLock *` 的 `state`，观测语义来自 `tranche`。

这解释了一个常见诊断现象：

```text
pg_stat_activity.wait_event = 'BufferMapping'
```

它说明 backend 正在等某个 buffer mapping partition lock，但不直接告诉你是哪一个 partition、哪个 BufferTag、哪个 backend 持有。

### `state`: 一个 32 位 word 里的多重语义

`lwlock.c` 中的核心宏：

| 位或值 | 含义 |
| --- | --- |
| `LW_FLAG_HAS_WAITERS` | waiters 链表中可能有人。 |
| `LW_FLAG_WAKE_IN_PROGRESS` | 已经有人被移出队列等待调度，先抑制重复唤醒。 |
| `LW_FLAG_LOCKED` | LWLock 自己的 wait list mutex 被持有。 |
| `LW_VAL_SHARED` | 一个 shared holder 的计数增量。 |
| `LW_VAL_EXCLUSIVE` | exclusive holder 的 sentinel，值为 `MAX_BACKENDS + 1`。 |
| `LW_SHARED_MASK` | 从 state 中取 shared holder 数。 |
| `LW_LOCK_MASK` | 从 state 中取“是否已有 holder”。 |

可以把 `state` 看成两段：

```text
高位 flags:
  HAS_WAITERS | WAKE_IN_PROGRESS | LOCKED

低位 lock count:
  0
  shared holder count: 1..MAX_BACKENDS
  exclusive sentinel: MAX_BACKENDS + 1
```

`StaticAssertDecl` 保证这些值不会互相重叠：

```text
MAX_BACKENDS + 1 必须是 2 的幂；
MAX_BACKENDS 不能覆盖高位 flags；
exclusive sentinel 不能覆盖高位 flags。
```

这类断言不是装饰。它们保证一个 `uint32` 可以同时承载：

```text
读写互斥状态
  + 等待者存在性
  + 唤醒状态
  + wait-list 内部互斥
```

### shared holder 与 exclusive sentinel

shared acquire 的冲突规则：

```text
只要没有 LW_VAL_EXCLUSIVE，就可以进入；
进入时 state += LW_VAL_SHARED。
```

这意味着多个 shared holder 可以并发叠加：

```text
state low bits = 3
  -> 当前有 3 个 shared holder
```

exclusive acquire 的冲突规则：

```text
只有 (state & LW_LOCK_MASK) == 0 时才能进入；
进入时 state += LW_VAL_EXCLUSIVE。
```

因此 exclusive holder 不和任何 shared holder 共存，也不和另一个 exclusive holder 共存。

用状态图表示：

```text
unlocked:
  low bits = 0

shared held:
  low bits = N, 1 <= N <= MAX_BACKENDS

exclusive held:
  low bits = MAX_BACKENDS + 1
```

高位 flags 可以和这些 holder 状态同时存在。例如：

```text
LW_FLAG_HAS_WAITERS | 2
  -> 有等待者标记，且当前有 2 个 shared holder。
```

### `LWLockPadded`: cache line 是状态边界的一部分

`LWLockPadded` 把一个 `LWLock` padding 到 `PG_CACHE_LINE_SIZE`：

```text
typedef union LWLockPadded
{
    LWLock lock;
    char   pad[LWLOCK_PADDED_SIZE];
} LWLockPadded;
```

这不是为了美观，也不是为了 ABI 整齐。LWLock 的 `state` 是高频修改的 atomic word。如果多个热 LWLock 落在同一个 cache line 上，即使它们保护完全不同的结构，也会出现 false sharing：

```text
backend A 修改 lock[0].state
backend B 修改 lock[1].state
如果二者在同一 cache line
  -> cache coherency 仍会让这条 line 在 CPU 间反复迁移
```

PostgreSQL 对主 LWLock 数组尤其在意这一点，因为数组里的锁数量不大，但个别锁非常热，例如 `ProcArrayLock`、`WALWriteLock`、`BufferMapping` partitions、`LockManager` partitions。

### tranche: 不是锁粒度，而是命名与分类

`lwlock.c` 注释把 tranche 分成三类：

| 类别 | 来源 | 例子 |
| --- | --- | --- |
| individually-named locks | `PG_LWLOCK()` | `ProcArray`、`WALWrite`、`ControlFile`。 |
| built-in lock groups | `PG_LWLOCKTRANCHE()` | `BufferMapping`、`LockManager`、`WALInsert`。 |
| user-defined tranches | extension 或 core 动态注册 | 扩展通过 `RequestNamedLWLockTranche()` 或 `LWLockNewTrancheId()` 注册。 |

tranche 的主要作用：

```text
初始化时给 LWLock 赋予类别；
wait event 上报时把类别 ID 转成名称；
让一组同类锁共享诊断名。
```

它不解决这些问题：

```text
不告诉你哪个 backend 持有锁；
不表达 SQL-level lock 对象；
不参与 deadlock detector；
不替代业务层的 partition/hash/tag。
```

## 5. 主流程源码 walkthrough

本节主流程只跟 `LWLockAttemptLock()`，因为它是 `state` 模型最集中、最可验证的一段源码。

### 5.1 入口：`LWLockAcquire()`

调用者一般从这里进入：

```text
LWLockAcquire(lock, mode)
  -> HOLD_INTERRUPTS()
  -> LWLockAttemptLock(lock, mode)
  -> 成功：记录到 held_lwlocks，返回
  -> 失败：进入等待队列慢路径
```

`HOLD_INTERRUPTS()` 和 `held_lwlocks` 在本节不是主角，但要先记住两个边界：

```text
LWLock 持有期间 cancel/die interrupts 被延后；
成功 acquire 后，当前 backend 会在 backend-local held_lwlocks[] 中记录这把锁。
```

这和 spinlock 已经不同：

```text
spinlock 没有 ResourceOwner 式兜底；
LWLock 至少有 backend-local held_lwlocks[]，ERROR cleanup 可以调用 LWLockReleaseAll()。
```

不过本节仍先停在 fast path。

### 5.2 第一步：读取旧 state

`LWLockAttemptLock()` 开头：

```text
old_state = pg_atomic_read_u32(&lock->state);
```

这次读取只是建立候选旧值。真正决定成功与否的是后面的 compare-and-exchange。

原因是任何时刻都可能有其它 backend 并发修改同一个 `state`：

```text
backend A 读到 state = 0；
backend B 先一步 shared acquire，把 state 改成 1；
backend A 的 CAS 不能再基于 0 成功。
```

因此 `old_state` 只是“我看到的版本”，不是锁状态的稳定事实。

### 5.3 exclusive mode：只有完全空闲才能进入

当 `mode == LW_EXCLUSIVE`：

```text
lock_free = (old_state & LW_LOCK_MASK) == 0;
if (lock_free)
    desired_state += LW_VAL_EXCLUSIVE;
```

判断只看 low bits 中是否已有 holder：

```text
没有 shared holder
  + 没有 exclusive sentinel
  = 可以拿 exclusive
```

高位 flags 不等于 holder。即使 `LW_FLAG_HAS_WAITERS` 已经设置，只要当前没有 holder，exclusive acquire 在 state 层面仍可能成功。是否公平、是否唤醒队列，是后续慢路径策略的问题，不属于 `LWLockAttemptLock()` 的职责。

这点很重要：

```text
LWLockAttemptLock() 只回答“当前 state 是否冲突”；
它不负责全局公平性，也不直接睡眠。
```

### 5.4 shared mode：只要没有 exclusive holder 就能进入

当 `mode == LW_SHARED`：

```text
lock_free = (old_state & LW_VAL_EXCLUSIVE) == 0;
if (lock_free)
    desired_state += LW_VAL_SHARED;
```

shared 模式不要求 low bits 为 0。它只要求没有 exclusive sentinel：

```text
state low bits = 0:
  第一个 reader 进入，变成 1。

state low bits = 5:
  第六个 reader 进入，变成 6。

state low bits = LW_VAL_EXCLUSIVE:
  writer 正在持有，reader 必须等待。
```

这就是 LWLock 相对于 spinlock 的关键收益：

```text
多个读者不需要在一个互斥锁上排队；
它们只是在同一个 atomic counter 上 CAS 加一。
```

当然，CAS 加一仍然会修改同一个 cache line，所以 shared acquire 不是免费的。它只是避免了“读者之间语义上没有冲突，却被独占 spinlock 完全串行化”。

### 5.5 CAS：检查和发布合并为一次原子状态跃迁

核心代码：

```text
if (pg_atomic_compare_exchange_u32(&lock->state,
                                   &old_state, desired_state))
{
    if (lock_free)
        return false;
    else
        return true;
}
```

这里有一个容易漏掉的细节：即使 `lock_free == false`，代码也把 `desired_state` 设成 `old_state` 并执行 CAS。

源码注释给出原因：

```text
always swap in the value
  -> doubles as a memory barrier
```

这不是为了改变 state，而是为了保持 acquire 路径的 memory ordering 语义一致。实现者可以想象两种分支：

```text
看起来可拿锁:
  CAS(old_state -> old_state + holder)

看起来不可拿锁:
  CAS(old_state -> old_state)
```

第二种 CAS 不改变值，但仍然是一个同步边界。注释也说明，曾经可以更聪明地只在可拿锁时 CAS，但 benchmark 没证明有收益。

这个选择体现了 PostgreSQL 常见风格：

```text
hot path 上的复杂优化必须证明收益；
否则宁愿保留较简单、ordering 明确的实现。
```

### 5.6 CAS 失败：不是锁冲突，而是 state 版本变了

如果 `pg_atomic_compare_exchange_u32()` 返回 false，说明：

```text
lock->state 当前值不等于 old_state；
CAS 已经把新的当前值写回 old_state；
循环基于新 old_state 重新计算 desired_state。
```

这和“拿不到锁”不同。

```text
CAS 失败:
  有并发 state 变化，重新判断。

CAS 成功且 lock_free=false:
  state 稳定地显示有冲突，返回 mustwait=true。

CAS 成功且 lock_free=true:
  已经把自己登记为 holder，返回 mustwait=false。
```

这三种结果是读 `LWLockAttemptLock()` 时最重要的分叉。

### 5.7 一个完整状态故事：两个读者和一个写者

假设初始 `state = 0`。

第一个 reader：

```text
old_state = 0
mode = LW_SHARED
没有 exclusive sentinel
desired_state = 1
CAS(0 -> 1) 成功
返回 mustwait=false
```

第二个 reader：

```text
old_state = 1
mode = LW_SHARED
仍然没有 exclusive sentinel
desired_state = 2
CAS(1 -> 2) 成功
返回 mustwait=false
```

writer 到来：

```text
old_state = 2
mode = LW_EXCLUSIVE
(old_state & LW_LOCK_MASK) != 0
desired_state = old_state
CAS(2 -> 2) 成功
返回 mustwait=true
```

writer 不会在 `LWLockAttemptLock()` 中睡眠。它只是把“当前不能拿锁”这个事实返回给 `LWLockAcquire()`。下一节才讨论 writer 如何入队、如何避免 missed wakeup。

两个 reader 释放：

```text
LWLockRelease()
  -> shared holder 每次 pg_atomic_sub_fetch_u32(..., LW_VAL_SHARED)
  -> 第二个 reader 释放后 low bits 变成 0
  -> 如果有 waiters 且没有 wake in progress，才进入 wakeup 逻辑
```

到这里，writer 才有机会被唤醒后重新跑 `LWLockAttemptLock()`，并把 state 从 0 改成 `LW_VAL_EXCLUSIVE`。

### 5.8 concrete case：BufferMapping partition lock

`src/backend/storage/buffer/buf_table.c` 把锁边界写得很清楚：

```text
BufTableLookup():
  caller must hold at least share lock on BufMappingLock for tag's partition

BufTableInsert():
  caller must hold exclusive lock on BufMappingLock for tag's partition

BufTableDelete():
  caller must hold exclusive lock on BufMappingLock for tag's partition
```

`src/include/storage/buf_internals.h` 中：

```text
BufMappingPartitionLock(hashcode)
  -> &MainLWLockArray[BUFFER_MAPPING_LWLOCK_OFFSET + partition].lock
```

这是一条很适合课堂跟读的链路：

```text
BufferTag
  -> BufTableHashCode()
  -> BufTableHashPartition()
  -> BufMappingPartitionLock()
  -> LWLockAcquire(..., LW_SHARED 或 LW_EXCLUSIVE)
  -> BufTableLookup / Insert / Delete
```

它说明 LWLock 的 shared/exclusive 不是抽象练习，而是直接服务 shared hash table 的读写边界：

```text
lookup 是共享读；
insert/delete 是独占写；
partitioning 降低不同 hash bucket 之间的锁竞争；
LWLockPadded 降低不同 partition lock 之间的 false sharing。
```

## 6. 生命周期 / ownership / cleanup

### 谁创建？

主 LWLock 数组由 shared memory 初始化路径创建。`LWLockShmemRequest()` 申请两块共享状态：

```text
LWLock tranches
Main LWLock array
```

`LWLockShmemInit()` 初始化：

```text
预定义 individual LWLocks
  -> 例如 ProcArrayLock、WALWriteLock

BufferMapping partitions
  -> NUM_BUFFER_PARTITIONS 个

LockManager partitions
  -> NUM_LOCK_PARTITIONS 个

PredicateLockManager partitions
  -> NUM_PREDICATELOCK_PARTITIONS 个

通过 RequestNamedLWLockTranche() 预先申请的 user-defined locks
```

每个锁最终调用：

```text
LWLockInitialize(lock, tranche_id)
  -> pg_atomic_init_u32(&lock->state, 0)
  -> lock->tranche = tranche_id
  -> proclist_init(&lock->waiters)
```

所以一个新 LWLock 的初始语义是：

```text
没有 holder；
没有 waiters；
属于某个 tranche；
等待队列为空。
```

### 谁持有？

持有者不是写在共享 `LWLock` 结构里的普通字段中。正常构建下，持有者语义来自两处：

```text
shared state:
  lock->state 低位表示 holder 数或 exclusive sentinel。

backend-local state:
  当前 backend 的 held_lwlocks[] 记录自己持有哪些 LWLock、以什么 mode 持有。
```

这意味着：

```text
其它 backend 能从 state 推断“有几个 shared holder / 是否有 exclusive holder”；
但不能从普通生产构建中直接问“谁持有这个 LWLock”。
```

`LOCK_DEBUG` 下有额外 `owner` 和 `nwaiters`，但不能把 debug 字段当作生产诊断模型。

### 谁释放？

正常路径由持有者调用：

```text
LWLockRelease(lock)
```

释放时先从当前 backend 的 `held_lwlocks[]` 中找到对应条目，确定 mode，然后：

```text
exclusive:
  state -= LW_VAL_EXCLUSIVE

shared:
  state -= LW_VAL_SHARED
```

如果释放后：

```text
有 HAS_WAITERS
没有 WAKE_IN_PROGRESS
低位 holder 已经为 0
```

才进入 `LWLockWakeup()`。这保证“唤醒等待者”不在每次 release 上无条件发生。

### ERROR / abort 时谁兜底？

LWLock 和 spinlock 的一个关键差异是：

```text
LWLock 成功 acquire 后会记录在 held_lwlocks[]；
ERROR cleanup 可以调用 LWLockReleaseAll() 释放当前 backend 仍持有的 LWLock。
```

`LWLockReleaseAll()` 循环释放所有 held locks。注释强调它用于 `ereport(ERROR)` 后清理，并且不直接改 `InterruptHoldoffCount` 的整体语义，因为错误恢复路径已经设置过合适的 interrupt holdoff 状态。

这不等于“持 LWLock 时可以随便 ERROR”。更准确的边界是：

```text
LWLock 比 spinlock 有更强的 ERROR cleanup 兜底；
但持锁期间仍应保持短、明确、可证明，不应跨越不可控 I/O 或复杂上层调用。
```

很多源码会在可能 `ERROR` 前主动释放 LWLock，或者把会报错的检查放到 acquire 之前。这是因为 cleanup 能避免锁永久泄漏，但不能自动恢复所有共享结构的半更新语义。

### tranche 生命周期

builtin tranche 名称编译进 `BuiltinTrancheNames[]`，来自 `lwlocklist.h`。

用户定义 tranche 有两条路径：

```text
RequestNamedLWLockTranche()
  -> shared_preload_libraries 的 shmem_request_hook 阶段请求
  -> 锁数组放入 MainLWLockArray

LWLockNewTrancheId()
  -> 运行时分配 tranche ID
  -> 调用者自己初始化具体 LWLock
```

`LWLockTranches` 是 shared memory 结构，内部 `num_user_defined` 和名称数组由一个 spinlock 保护。这里又体现了分层：

```text
spinlock:
  保护 tranche 注册表中几个字段的瞬时更新。

LWLock:
  作为被注册、被命名、被观测的共享结构互斥原语。
```

## 7. 正确性机制层次

### 层次一：atomic CAS 保证单个 state 跃迁不可拆分

`LWLockAttemptLock()` 的核心正确性来自：

```text
pg_atomic_compare_exchange_u32()
```

它保证：

```text
只有当 state 仍等于我看到的 old_state 时，才写入 desired_state；
如果别人已经改变 state，我必须基于新 state 重算。
```

这收住了典型竞态：

```text
两个 writer 同时看到 state = 0
  -> 只有一个 CAS(0 -> EXCLUSIVE) 成功
  -> 另一个 CAS 失败后看到新 state，再返回 mustwait=true
```

### 层次二：low bits 表达读写兼容矩阵

LWLock 的读写兼容矩阵很小：

| 已有状态 | 新 shared | 新 exclusive |
| --- | --- | --- |
| unlocked | 允许 | 允许 |
| shared holders | 允许 | 等待 |
| exclusive holder | 等待 | 等待 |

`LWLockAttemptLock()` 没有显式写这个表，而是把它压成两个判断：

```text
shared:
  old_state & LW_VAL_EXCLUSIVE == 0

exclusive:
  old_state & LW_LOCK_MASK == 0
```

这就是“field + flag + lifecycle state + lock context 才是语义”的具体例子。单看一个 `uint32` 数字没有意义，必须结合 bit mask 和 mode 才能解释。

### 层次三：memory barrier 边界

源码注释明确说 CAS doubles as a memory barrier。对于调用者，这意味着：

```text
成功 acquire 后，临界区内读取共享结构，不应被重排到 acquire 之前；
release 时的 atomic decrement 与后续 wakeup 逻辑共同形成释放边界。
```

LWLock 不是只保护 C 语言层面的变量赋值顺序。它必须在多 CPU、多进程共享内存中建立 ordering 边界。

这也是为什么不能把 LWLockAcquire 替换成普通 `if (state == 0) state = 1`：

```text
缺少原子性；
缺少跨 CPU 可见性顺序；
缺少等待者状态协调；
缺少 wait event 和 cleanup 语义。
```

### 层次四：cache-line padding 降低伪共享

正确性保证锁是对的；性能边界保证锁能用在 hot path。

`LWLOCK_PADDED_SIZE = PG_CACHE_LINE_SIZE` 服务的是：

```text
减少相邻 LWLock 的 state 落在同一 cache line；
降低独立锁之间的 cache invalidation 传播；
让 partitioned LWLock 真正接近 partitioned，而不是在硬件层又粘回一起。
```

这个机制不改变语义，但会强烈影响 scalability。

### 层次五：tranche 把内部锁变成可观测事件

`LWLockReportWaitStart(lock)`：

```text
pgstat_report_wait_start(PG_WAIT_LWLOCK | lock->tranche)
```

这让 `pg_stat_activity` 可以显示：

```text
wait_event_type = 'LWLock'
wait_event      = 'ProcArray'
```

或：

```text
wait_event_type = 'LWLock'
wait_event      = 'BufferMapping'
```

观测名不是为了 correctness，但它决定了线上问题能否被定位到足够小的模块边界。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 `LWLockAttemptLock()` 失败不是异常

`LWLockAttemptLock()` 返回 `true` 的语义是：

```text
somebody else has the lock，需要等待。
```

这不是 ERROR，也不是 PANIC。它只是把控制权交回调用者：

```text
LWLockAcquire()
  -> 入队并睡眠

LWLockConditionalAcquire()
  -> RESUME_INTERRUPTS()
  -> 返回 false

LWLockAcquireOrWait()
  -> 等锁变空闲，但未必最终持有它
```

因此同一个 atomic state fast path 支撑了三类上层语义：

| 上层 API | 拿不到时怎么办 |
| --- | --- |
| `LWLockAcquire()` | 等待，直到拿到。 |
| `LWLockConditionalAcquire()` | 不等待，返回 false。 |
| `LWLockAcquireOrWait()` | 等它释放；如果别人已经完成目标工作，可以不持锁返回。 |

本节重点是：这三者都先通过 `LWLockAttemptLock()` 做同一套冲突判断。

### 8.2 没有 `PGPROC` 时不能等待

`LWLockAcquire()` 中有断言：

```text
Assert(!(proc == NULL && IsUnderPostmaster));
```

慢路径 `LWLockQueueSelf()` 更直接：

```text
if (MyProc == NULL)
    elog(PANIC, "cannot wait without a PGPROC structure");
```

原因是 LWLock 慢路径等待依赖：

```text
MyProc
  -> lwWaiting
  -> lwWaitMode
  -> lwWaitLink
  -> sem
```

没有 `PGPROC` 就没有可入队、可唤醒的身份。早期 shared memory 初始化阶段可以初始化 LWLock，也可以在没有竞争的情况下使用某些锁路径，但不能进入真正等待。

### 8.3 太多同时持有的 LWLock

当前 backend 用固定大小数组记录 held LWLocks：

```text
MAX_SIMUL_LWLOCKS = 200
held_lwlocks[MAX_SIMUL_LWLOCKS]
```

如果超过：

```text
elog(ERROR, "too many LWLocks taken")
```

这不是 arbitrary limit，而是一个设计约束：

```text
LWLock 应保护短临界区；
正常路径不应积累大量同时持有的 LWLock；
如果代码需要持有几百把 LWLock，通常说明锁层次或算法边界有问题。
```

### 8.4 wait event 名称可能退化

文档提醒：扩展添加的 LWLock wait event 名称不一定在所有 server process 中可用，有时会显示为 generic 的 extension 类名称，而不是扩展注册的名字。

这来自 tranche 名称的生命周期和本地缓存边界：

```text
tranche 名称在 shared memory 中注册；
backend 有 LocalNumUserDefinedTranches 本地缓存；
不同进程看到名称的时机可能不同。
```

因此诊断扩展 LWLock 时，不能假设 wait_event 名称总是和扩展代码里的字符串完全一致。

### 8.5 本节刻意不展开的异常路径

以下内容属于下一节：

```text
入队后锁刚好释放，如何避免 missed wakeup；
LW_FLAG_WAKE_IN_PROGRESS 为什么存在；
PGPROC.lwWaiting 的 WAITING / PENDING_WAKEUP / NOT_WAITING 如何转换；
process semaphore 多余唤醒如何归还；
ERROR 后 LWLockReleaseAll() 如何和 held_lwlocks 配合。
```

它们都很重要，但它们解决的是等待队列协议，不是本节唯一主问题。

## 9. 成本、资源与跨模块传播

### shared acquire 的成本不是零

当前实现避免了“先抢内部 spinlock 再改读者计数”，但 shared acquire 仍然要：

```text
读 lock->state
计算 desired_state
执行 atomic compare-and-exchange
在成功后修改同一个 cache line
```

在低竞争下，这很便宜。在大量 CPU 同时频繁拿同一把 LWLock shared mode 时，cache line 仍然会在 CPU 间迁移。

所以 LWLock 的性能规律不是：

```text
shared mode 可以无限扩展。
```

而是：

```text
shared mode 避免读者被语义上不必要地互斥；
但所有读者仍然共同修改 holder count，热点锁仍会形成 cache coherency 成本。
```

这解释了为什么 PostgreSQL 还会把一些结构 partition：

```text
BufferMapping:
  NUM_BUFFER_PARTITIONS = 128

LockManager:
  NUM_LOCK_PARTITIONS = 16

PredicateLockManager:
  NUM_PREDICATELOCK_PARTITIONS = 16
```

### exclusive acquire 的成本来自等待读者清空

exclusive holder 必须等所有 shared holder 离开：

```text
state low bits 从 N 逐步减到 0
writer 才能 CAS(0 -> LW_VAL_EXCLUSIVE)
```

如果 workload 是：

```text
大量 backend 高频 shared acquire/release
偶发 writer 需要 exclusive
```

writer latency 可能被 shared 流量放大。后续等待队列协议会缓解重复唤醒和 missed wakeup，但不能改变读写互斥的基本事实。

### tranche 粒度影响诊断聚合

`BufferMapping` 是一组 partition lock 的 tranche。线上看到：

```sql
SELECT wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock'
GROUP BY 1;
```

如果出现很多 `BufferMapping`，只能先说明：

```text
buffer mapping table 的某些 partition 有争用。
```

不能直接说明：

```text
所有 partition 都一样热；
某个具体 block 是根因；
持锁者是谁。
```

这类诊断要继续结合：

```text
workload 的 relation/block 访问模式；
shared_buffers 命中与淘汰行为；
perf 中 lwlock / buffer manager 热点；
gdb 或 tracepoint 采样；
必要时自定义 instrumentation。
```

### LWLock 是很多上层模块的基础设施

常见跨模块传播：

| 模块 | LWLock 角色 |
| --- | --- |
| ProcArray | 保护 backend 事务状态数组，snapshot 和事务结束都依赖它。 |
| shared buffers | 保护 BufferTag 到 buffer id 的 mapping table。 |
| lock manager | 保护 heavyweight lock shared hash partitions。 |
| WAL | `WALWriteLock`、`WALBufMappingLock`、`WALInsert` tranche 保护 WAL 写入与 buffer 状态。 |
| replication slot | 保护 slot 分配、控制状态和 I/O 状态。 |
| parallel query | 保护 parallel hash、append、shared tuplestore、DSA 等共享执行状态。 |

因此 LWLock 争用不是一个“锁模块问题”就能结束。它通常是上层模块共享状态设计、workload 访问模式和硬件 cache coherency 共同作用的结果。

## 10. 观测与诊断入口

### SQL：看等待类别和 tranche 名

最直接入口：

```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock';
```

典型结果：

```text
wait_event_type = LWLock
wait_event      = ProcArray
```

或：

```text
wait_event_type = LWLock
wait_event      = BufferMapping
```

解释边界：

```text
能看到：
  等的是哪类 LWLock tranche。

看不到：
  具体 lock pointer；
  holder backend；
  shared holder 数；
  等待队列顺序；
  业务对象 tag。
```

### SQL：连接 `pg_wait_events`

本地源码文档中推荐了 `pg_wait_events`。可以用：

```sql
SELECT a.pid, a.wait_event, w.description
FROM pg_stat_activity a
JOIN pg_wait_events w
  ON a.wait_event_type = w.type
 AND a.wait_event = w.name
WHERE a.wait_event_type = 'LWLock';
```

这能把 `ProcArray`、`WALWrite`、`BufferMapping` 解释成更接近模块语义的描述。

### perf：看 CPU 是否花在 lwlock atomic / spin delay

如果现象是 CPU 高、吞吐下降，而 `pg_stat_activity` 中 LWLock 等待不明显，需要区分：

```text
backend 是否真的睡在 LWLock wait event；
还是大量 CPU 花在 fast path CAS、短暂 spinning、cache line 迁移上。
```

可用方向：

```bash
perf top -p <pid>
perf record -g -p <pid>
perf report
```

关注符号：

```text
LWLockAttemptLock
LWLockAcquire
LWLockRelease
LWLockWaitListLock
perform_spin_delay
pg_atomic_compare_exchange_u32
```

如果热点集中在 `LWLockAttemptLock()` 或原子指令附近，说明问题可能还没进入长睡眠，而是在 fast path cache line 竞争。

### gdb：直接观察 state 位

实验环境中可以对一个 backend 下断点：

```gdb
break LWLockAttemptLock
commands
  print lock
  print mode
  print lock->tranche
  print lock->state.value
  continue
end
```

具体 atomic 类型字段名可能随平台和编译选项变化；如果不能直接 `lock->state.value`，可以先：

```gdb
ptype lock->state
```

然后按实际结构读取。

观察重点：

```text
mode = LW_SHARED 时，state 低位如何递增；
mode = LW_EXCLUSIVE 时，低位非 0 如何导致 mustwait；
高位 flags 是否和 holder bits 同时出现。
```

### 源码 tracepoint / DTrace probes

`lwlock.c` 中有 tracepoint：

```text
TRACE_POSTGRESQL_LWLOCK_ACQUIRE
TRACE_POSTGRESQL_LWLOCK_RELEASE
TRACE_POSTGRESQL_LWLOCK_WAIT_START
TRACE_POSTGRESQL_LWLOCK_WAIT_DONE
TRACE_POSTGRESQL_LWLOCK_CONDACQUIRE
TRACE_POSTGRESQL_LWLOCK_CONDACQUIRE_FAIL
```

它们传递 tranche 名和 mode，适合在支持相关探针的构建和平台上做低侵入采样。

### `LOCK_DEBUG` / `LWLOCK_STATS`

源码里还有调试构建选项：

```text
LOCK_DEBUG:
  Trace_lwlocks、owner、nwaiters。

LWLOCK_STATS:
  sh_acquire_count、ex_acquire_count、block_count、spin_delay_count。
```

这些不是生产默认观测能力，但很适合课堂实验或本地复现：

```text
同一个 workload 下，比较 shared acquire 数、exclusive acquire 数、block 数；
验证某个 wait_event 是否只是偶发睡眠，还是大量 acquire 都集中在同一 tranche。
```

## 11. 常见误区

### 误区一：LWLock 是 lightweight，所以可以保护很长逻辑

`lightweight` 是相对于 heavyweight lock manager 的 SQL-visible lock、deadlock detector 和复杂对象语义而言的。它不是说：

```text
可以持有 LWLock 做任意长时间工作。
```

LWLock 仍应保护短共享内存访问。持锁时间越长，shared/exclusive 兼容矩阵越容易把上层 workload 放大成系统等待。

### 误区二：shared mode 没有成本

shared mode 允许读者并发，但每个 reader 仍要修改 holder count。热点 shared lock 在多核上仍然会产生 cache line 迁移。

正确表述是：

```text
shared mode 避免不必要的读者互斥；
不是消除所有读者同步成本。
```

### 误区三：tranche 就是具体锁

tranche 是分类和名字。多个 LWLock 可以属于同一 tranche。

```text
wait_event = BufferMapping
```

不等于只有一把 `BufferMappingLock`，而是表示正在等待某个 buffer mapping partition 的 LWLock。

### 误区四：state 高位 flags 表示锁被业务持有

`LW_FLAG_LOCKED` 这个名字容易误导。它不是“业务层 LWLock 已被持有”，而是：

```text
LWLock wait list 自己的 mutex 被持有。
```

业务层 holder 在 low bits：

```text
shared count
exclusive sentinel
```

### 误区五：看到 LWLock 等待就等于 PostgreSQL 锁等待

SQL 里的 `Lock` wait event 和 `LWLock` wait event 不同：

```text
Lock:
  heavyweight lock，通常是 SQL-visible 对象或事务锁。

LWLock:
  internal lightweight lock，通常保护 shared memory data structure。
```

`wait_event_type='LWLock'` 一般不能用 `pg_locks` 直接找到 blocker。

### 误区六：LWLock 可以替代所有 atomic

有些状态只需要单个 atomic counter 或 flag，不需要等待队列和 shared/exclusive 语义。用 LWLock 会增加不必要的 acquire/release 成本和潜在等待。

选择顺序应从语义出发：

```text
一个字段的无锁计数:
  atomic

几个字段的极短一致更新:
  spinlock

共享结构短读写访问:
  LWLock

等待某个条件:
  condition variable / latch
```

## 12. 课堂实验

### 实验一：手工解码 `LWLockAttemptLock()` 的 state

目标：确认 shared / exclusive 的兼容矩阵如何落到 bit 判断上。

步骤：

1. 打开 `src/backend/storage/lmgr/lwlock.c`。
2. 找到这些宏：

```text
LW_FLAG_HAS_WAITERS
LW_FLAG_WAKE_IN_PROGRESS
LW_FLAG_LOCKED
LW_VAL_EXCLUSIVE
LW_VAL_SHARED
LW_SHARED_MASK
LW_LOCK_MASK
```

3. 代入几个 state：

```text
state = 0
state = 1
state = 3
state = LW_VAL_EXCLUSIVE
state = LW_FLAG_HAS_WAITERS | 2
```

4. 分别判断 `LW_SHARED` 和 `LW_EXCLUSIVE` 下 `lock_free` 的值。

预期结论：

```text
shared 只被 exclusive sentinel 阻塞；
exclusive 被任何 holder 阻塞；
HAS_WAITERS 不等价于 holder。
```

### 实验二：跟读 BufferMapping 的 shared / exclusive 边界

目标：理解 LWLock 为什么适合保护 shared hash table。

步骤：

1. 在 `/home/highgo/postgres` 中搜索：

```bash
rg -n "BufMappingPartitionLock|BufTableLookup|BufTableInsert|BufTableDelete" src/backend src/include
```

2. 阅读：

```text
src/include/storage/buf_internals.h
src/backend/storage/buffer/buf_table.c
src/backend/storage/buffer/bufmgr.c
```

3. 记录每条路径使用的 mode：

```text
lookup:
  至少 shared lock

insert/delete:
  exclusive lock
```

4. 回答：

```text
如果这里用 spinlock，会把哪些本可并发的读者串行化？
如果这里用单个全局 LWLock，不做 partition，会把什么 workload 放大？
```

### 实验三：观察 LWLock wait event

目标：把源码 tranche 名和 SQL wait event 对上。

在有并发压力的测试库中运行：

```sql
SELECT wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock'
GROUP BY wait_event
ORDER BY count(*) DESC, wait_event;
```

再连接说明表：

```sql
SELECT a.wait_event, w.description, count(*)
FROM pg_stat_activity a
JOIN pg_wait_events w
  ON a.wait_event_type = w.type
 AND a.wait_event = w.name
WHERE a.wait_event_type = 'LWLock'
GROUP BY a.wait_event, w.description
ORDER BY count(*) DESC;
```

分析要求：

```text
看到 ProcArray:
  回到 procarray.c，看 snapshot / transaction end 是否相关。

看到 BufferMapping:
  回到 buf_table.c / bufmgr.c，看 buffer lookup / allocation 是否相关。

看到 WALWrite:
  回到 xlog.c，看 WAL flush / commit latency 是否相关。
```

不要直接把 wait_event 当根因。它是入口，不是结论。

### 实验四：比较 `LWLockAcquire()` 与 `LWLockConditionalAcquire()`

目标：理解同一个 `LWLockAttemptLock()` fast path 如何支撑不同上层语义。

步骤：

1. 阅读 `LWLockAcquire()`：

```text
失败后入队并等待。
```

2. 阅读 `LWLockConditionalAcquire()`：

```text
失败后 RESUME_INTERRUPTS() 并返回 false。
```

3. 在源码中搜索调用点：

```bash
rg -n "LWLockConditionalAcquire" src/backend src/include
```

4. 选择 `ProcArrayEndTransaction()` 中的调用，解释为什么它可以：

```text
能立即拿 ProcArrayLock:
  直接清理 XID。

不能立即拿:
  走 group XID clearing。
```

结论：

```text
LWLockAttemptLock() 只判断 state；
是否等待、绕行、合并工作，是上层 API 和模块策略。
```

### 实验五：用 perf 区分睡眠等待和 fast path 竞争

目标：避免把所有 LWLock 问题都理解成“睡在 wait_event 上”。

步骤：

1. 在压测时查看 SQL 等待：

```sql
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY 1, 2
ORDER BY 3 DESC;
```

2. 同时对热点 backend 采样：

```bash
perf top -p <pid>
```

3. 对比两种现象：

```text
pg_stat_activity 中大量 LWLock wait_event:
  backend 多数时间可能已经睡眠。

perf 中 LWLockAttemptLock / atomic / spin delay 热:
  backend 可能在 CPU 上反复竞争 cache line。
```

这两者的优化方向不同：

```text
睡眠等待:
  找持锁时间、唤醒路径、上层长临界区。

fast path 竞争:
  找热点 state、partition 是否不足、读路径是否过于频繁。
```

## 13. 讨论题

1. 为什么 PostgreSQL 不把 LWLock 的 shared holder count、exclusive flag 和 has-waiters flag 拆成三个字段？

2. `LWLockAttemptLock()` 在 `lock_free == false` 时仍执行 CAS(old_state -> old_state)，这个选择在性能和 ordering 上分别有什么含义？

3. 如果一个共享结构读多写少，但读路径每次都要持锁扫描很长链表，使用 LWLock shared mode 是否一定合适？

4. `BufferMapping` 使用 partitioned LWLock。partitioning 降低了什么竞争？又没有降低什么竞争？

5. 为什么 `wait_event='ProcArray'` 不能直接告诉你 blocker backend 是谁？如果线上看到大量 `ProcArray`，你会先检查哪些 workload 或源码路径？

6. `LW_FLAG_LOCKED` 为什么不是业务锁持有状态？如果把它误读成 holder，会导致哪些诊断错误？

7. 扩展注册 user-defined tranche 时，为什么 tranche name 是观测接口的一部分？命名不清楚会给线上诊断带来什么成本？

8. 如果把某个 spinlock 临界区改成 LWLock，除了“可以睡眠”外，还引入了哪些额外状态和 cleanup 责任？

## 14. 本节小结

本节把 LWLock 的第一层抽象压缩成一个模型：

```text
LWLock 是 shared memory 短临界区的 reader-writer 同步原语；
它的 fast path 不靠内部 spinlock，而靠一个 pg_atomic_uint32 state；
state 低位表达 shared holder count 或 exclusive sentinel；
state 高位表达 waiters / wake-in-progress / wait-list mutex flags；
LWLockAttemptLock() 用 CAS 同时完成冲突检查、状态发布和 memory ordering；
tranche 给锁分类和命名，让内部等待能出现在 pg_stat_activity wait_event 中；
LWLockPadded 把高频 atomic state 放到 cache-line 友好的布局里。
```

可迁移系统规律：

```text
当共享结构需要“读者并发、写者排他”时，锁设计的关键不是给它加一个更大的 mutex，
而是把兼容矩阵压进足够便宜、足够可观测、且能与等待协议组合的状态机。
```

也要保留边界：

```text
shared mode 不是无成本；
tranche 不是具体锁实例；
wait_event 不是 blocker 图；
atomic state 只解决 acquire/release fast path，不单独解决 missed wakeup。
```

下一节会沿着 `LWLockAcquire()` 的失败路径继续：

```text
为什么拿不到 LWLock 时，必须先尝试、入队、再尝试、再睡眠？
PGPROC.lwWaiting、process semaphore、LW_FLAG_WAKE_IN_PROGRESS 和 held_lwlocks
如何一起避免 missed wakeup、重复唤醒和 ERROR 后遗留锁？
```
