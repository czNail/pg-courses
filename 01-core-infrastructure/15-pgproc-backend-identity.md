# PostgreSQL PGPROC slot 与 backend identity 生命周期

## 课程定位

前置知识：已经理解 main shared memory 的启动期 sizing / init / attach 生命周期，也理解 `MemoryContext` 管 backend-local 内存、`ResourceOwner` 管事务资源 cleanup。

本节唯一主问题：

```text
为什么每个 backend 必须先获得一个 PGPROC 槽位，才能参与 shared memory 中的锁、等待、事务状态发布和退出清理？
```

核心矛盾：PostgreSQL 的 backend 是独立进程。进程间不能共享普通 C 栈、本地指针或线程对象；但锁等待、快照可见性、取消信号、同步复制等待、checkpoint delay、wait event、退出清理都需要一个其它进程能识别、能唤醒、能扫描、能回收的身份。于是系统必须给每个参与 shared memory 的进程分配一个稳定的 shared-memory identity，也就是 `PGPROC`。

学完后应能判断：什么时候一个进程只有 `MyProc` 但还没进入 `ProcArray`；为什么 `InitProcess()` 必须早于 LWLock / lock manager / buffer manager 的常规使用；为什么 `ProcArrayAdd()` 被拆到 `InitProcessPhase2()`；为什么退出时必须先从 `ProcArray` 消失，再释放 PGPROC slot；以及 `too many clients already`、wait event、`pg_locks`、gdb 中的 `MyProc` 分别能说明什么。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

前几课已经建立了三层基础设施：

```text
shared memory:
  哪些状态能被所有 backend 看到，以及这些状态如何在启动期定容。

MemoryContext:
  一个 backend 内部的内存对象如何按生命周期批量回收。

ResourceOwner:
  buffer pin、lock、snapshot、file、cache ref 等外部资源如何在 ERROR / abort 时兜底释放。
```

本节进入另一个基础问题：

```text
一个 OS process 什么时候成为 PostgreSQL shared memory 世界里的一个 backend？
```

答案不是“fork 出来就算”。fork 只创建了一个操作系统进程。PostgreSQL 还需要给它分配：

```text
MyProc:
  当前进程在 shared memory 中的 PGPROC 指针。

MyProcNumber:
  当前进程在 ProcGlobal->allProcs[] 中的稳定编号。

shared latch / semaphore:
  其它 backend 可以用来唤醒它的等待对象。

wait / lock / xact fields:
  其它 backend 可以扫描或修改的共享状态。
```

这一步由 `InitProcess()` 完成。后续 `InitPostgres()` 里调用 `InitProcessPhase2()`，才把这个 `PGPROC` 加入 `ProcArray`，让它成为事务可见性和全局 backend scan 的一员。

本节只讲这个 identity 的生命周期：

```text
启动期预留 PGPROC slots
  -> backend 启动时领取一个 slot
  -> 初始化 shared latch / semaphore / wait fields
  -> 加入 ProcArray
  -> 正常或异常退出时从 ProcArray 移除
  -> 释放 PGPROC slot 供后续 backend 复用
```

下一节会沿着已经加入 `ProcArray` 的 backend，继续讲 XID、SubXID、xmin、status flags 如何发布给其它 backend。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
postmaster 启动期按 MaxBackends 等上限一次性创建 PGPROC 池；
backend 启动时从对应 freelist 领取一个 PGPROC，建立 MyProc / MyProcNumber / shared latch / semaphore / wait fields；
InitPostgres 早期再把 MyProc 加入 ProcArray；
进程退出时 on_shmem_exit 先从 ProcArray 移除，再由 ProcKill 释放等待状态、shared latch 和 PGPROC slot。
```

这背后的 tension 是：

```text
shared memory 里的身份必须稳定、可被其它进程直接引用
  vs
backend 进程不断创建和退出，slot 必须能安全复用
```

如果没有 `PGPROC`，很多机制会失去锚点：

| 机制 | 为什么需要 PGPROC |
| --- | --- |
| LWLock / buffer content lock 等等待 | wait queue 里不能放 backend-local 指针；需要放可跨进程解释的 `ProcNumber` / `PGPROC`。 |
| heavyweight lock manager | 锁等待、死锁检测、PROCLOCK 链表都要知道“哪个 backend 持有或等待”。 |
| latch / semaphore 唤醒 | 其它进程需要一个共享的唤醒目标。 |
| wait event | `pg_stat_activity.wait_event` 需要把当前等待状态写到共享位置。 |
| transaction visibility | XID、SubXID、xmin、vacuum flags 要从当前 backend 发布给 snapshot / VACUUM 相关扫描者。 |
| cancellation / recovery conflict / barriers | procsignal 和 recovery conflict 需要定位目标 backend。 |
| process exit cleanup | 进程退出时必须把 shared memory 中仍指向它的低层状态清掉。 |

所以 `PGPROC` 不是“连接信息结构体”。它是 PostgreSQL 进程模型里最小的 shared identity。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/proc.h` | `PGPROC`、`PROC_HDR`、`MyProc`、`ProcGlobal`、`MyProcNumber` 的状态边界。 |
| 2 | `src/backend/storage/lmgr/proc.c` | `ProcGlobalShmemRequest()`、`ProcGlobalShmemInit()`、`InitProcess()`、`InitProcessPhase2()`、`InitAuxiliaryProcess()`、`ProcKill()`。 |
| 3 | `src/backend/storage/ipc/procarray.c` | `ProcArrayAdd()` / `ProcArrayRemove()` 如何把 PGPROC 加入或移出全局扫描数组，并维护 dense mirror arrays。 |
| 4 | `src/backend/tcop/backend_startup.c` | 普通客户端 backend 在认证前后什么时候调用 `InitProcess()`。 |
| 5 | `src/backend/tcop/postgres.c` | standalone backend 和普通 backend 如何在 `PostgresMain()` 中 `BaseInit()`、生成 cancel key、进入 `InitPostgres()`。 |
| 6 | `src/backend/utils/init/postinit.c` | `BaseInit()` 为什么要求已有 `MyProc`，`InitPostgres()` 为什么一开始调用 `InitProcessPhase2()`。 |
| 7 | `src/backend/postmaster/auxprocess.c` | auxiliary process 如何只拿 PGPROC、初始化 LWLock / shared memory 访问，但不走完整 `InitPostgres()`。 |
| 8 | `src/backend/storage/ipc/ipc.c` | `shmem_exit()`、`on_shmem_exit()` 的 callback 顺序，解释为什么 RemoveProcFromArray 会早于 ProcKill。 |
| 9 | `src/backend/access/transam/varsup.c` | `GetNewTransactionId()` 如何依赖 `MyProc->pgxactoff` 把 XID 写入 `ProcGlobal->xids[]`。 |
| 10 | `src/backend/access/transam/README` | ProcArray / XID / snapshot 的 correctness 规则。本节只读 identity 相关边界，后续课程再展开 snapshot。 |

推荐阅读顺序：

```text
先读 proc.h 的 PGPROC / PROC_HDR 注释
  -> 看 ProcGlobalShmemRequest/Init 如何预留 slot 池
  -> 看 InitProcess 如何从 freelist 领取 MyProc
  -> 看 InitProcessPhase2 如何 ProcArrayAdd
  -> 看 shmem_exit 如何按 LIFO 执行 RemoveProcFromArray 和 ProcKill
```

不要一开始就读 `GetSnapshotData()`。它是下一组问题：已经有了 `ProcArray` membership 后，如何扫描事务状态。

## 4. 关键数据结构与状态

### `PGPROC`: shared identity，不是 backend-local session object

`PGPROC` 定义在 `src/include/storage/proc.h`。每个参与 shared memory 的 backend 都有一个 `PGPROC`，保存在 shared memory 的 `ProcGlobal->allProcs[]` 中。

本节只关注影响 identity 生命周期的字段组合：

| 字段组 | 语义 |
| --- | --- |
| `pid`、`backendType` | 当前 slot 是否被真实进程占用，以及这个进程类型是什么。prepared transaction dummy PGPROC 的 `pid` 为 0。 |
| `databaseId`、`roleId`、`tempNamespaceId` | backend 连接数据库、认证和临时 schema 之后才逐步填充，不是 `InitProcess()` 刚完成就都有值。 |
| `vxid.procNumber`、`vxid.lxid` | `procNumber` 是 slot identity；`lxid` 是当前顶层事务的本地事务号。 |
| `xid`、`xmin`、`subxidStatus`、`subxids` | 当前事务状态发布入口。具体可见性语义留到后续 ProcArray / snapshot 课程。 |
| `procLatch`、`sem` | 当前 backend 可被其它进程唤醒的 shared wait primitive。 |
| `lwWaiting`、`lwWaitMode`、`lwWaitLink` | LWLock 等待队列状态。 |
| `waitLock`、`waitLink`、`waitProcLock`、`waitStatus`、`myProcLocks[]` | heavyweight lock manager 的持锁 / 等待状态。 |
| `wait_event_info` | wait event 写入位置，供 stats 视图读取。 |
| `procArrayGroupMember`、`procArrayGroupNext` | commit/abort 时 group XID clearing 使用的队列状态。 |
| `clogGroupMember`、`clogGroupNext` | group transaction status update 使用的队列状态。 |

这里要特别注意：

```text
PGPROC 是 shared memory 对象；
MyProc 是当前进程指向自己 PGPROC 的 backend-local 全局变量；
MyProcNumber 是当前 PGPROC 在 ProcGlobal->allProcs[] 中的编号。
```

其它 backend 可以通过 `ProcNumber` 找到你的 `PGPROC`：

```c
#define GetPGProcByNumber(n) (&ProcGlobal->allProcs[(n)])
#define GetNumberFromPGProc(proc) ((proc) - &ProcGlobal->allProcs[0])
```

这也是为什么 wait queue、lock group、replication slot、async notification 等地方经常保存 `MyProcNumber`，而不是保存一个 backend-local 指针。

### `PROC_HDR`: 全局 PGPROC 池和 freelists

`PROC_HDR` 也在 `proc.h`，全局变量是 `ProcGlobal`。它不是某一个 backend 的状态，而是整个 cluster 的进程表头：

| 字段 | 语义 |
| --- | --- |
| `allProcs` | 所有真实进程 PGPROC 的数组，不包括 prepared transaction dummy PGPROC。 |
| `xids`、`subxidStates`、`statusFlags` | 从 PGPROC mirror 出来的 dense arrays，只对已进入 ProcArray 的 PGPROC 有意义。 |
| `freeProcsLock` | 保护 PGPROC freelists 的 spinlock。不能用 LWLock，因为 LWLock 等待本身依赖已有 PGPROC。 |
| `freeProcs` | 普通 client backend slot 池。 |
| `autovacFreeProcs` | autovacuum worker 和 special worker slot 池。 |
| `bgworkerFreeProcs` | background worker slot 池。 |
| `walsenderFreeProcs` | WAL sender slot 池。 |
| `procArrayGroupFirst` | ProcArray group XID clear 队列头。 |
| `clogGroupFirst` | CLOG group update 队列头。 |
| `walwriterProc`、`checkpointerProc` | 某些 auxiliary process 的当前 ProcNumber。 |

`freeProcsLock` 使用 spinlock 是一个很好的系统边界例子：

```text
分配 PGPROC 是进入 LWLock 世界之前的动作；
因此不能用 LWLock 保护 PGPROC freelist；
只能用更底层、短临界区的 spinlock。
```

### PGPROC 池的固定分区

`ProcGlobalShmemRequest()` 按上限一次性预留所有 `PGPROC`：

```text
TotalProcs = MaxBackends + NUM_AUXILIARY_PROCS + max_prepared_xacts
```

源码注释把消费者分成六类：

```text
1. normal backends
2. autovacuum workers and special workers
3. background workers
4. walsenders
5. auxiliary processes
6. prepared transactions
```

这些 slot 不是运行时随意互相借用的。`ProcGlobalShmemInit()` 会把不同区间挂到不同 freelist：

```text
freeProcs:
  normal backend

autovacFreeProcs:
  autovacuum worker / special worker

bgworkerFreeProcs:
  background worker

walsenderFreeProcs:
  WAL sender

AuxiliaryProcs:
  fixed array, 通过 pid == 0 判断空闲，不走普通 freelist

PreparedXactProcs:
  prepared transaction dummy PGPROC，由 twophase.c 管理
```

这解释了一个常见现象：`max_connections` 不是唯一会影响 `PGPROC` 数量的配置。`max_worker_processes`、`max_wal_senders`、`autovacuum_worker_slots`、`max_prepared_transactions` 和 auxiliary process 上限也会进入 shared memory sizing。

### `pgxactoff`: ProcArray dense arrays 的索引，不是永久身份

`PGPROC->pgxactoff` 指向 `ProcGlobal->xids[]`、`subxidStates[]`、`statusFlags[]` 中对应的 dense array 位置。

但它有一个重要限制：

```text
PGPROC slot number 是稳定 identity；
pgxactoff 会随着 ProcArrayAdd / ProcArrayRemove 后的数组 memmove 改变。
```

`proc.h` 注释明确说，只有持有 `ProcArrayLock` 或 `XidGenLock` 时，才能安全用 `pgxactoff` 访问 dense arrays。原因是 `ProcArrayAdd()` / `ProcArrayRemove()` 会移动 dense arrays，并调整后续 PGPROC 的 `pgxactoff`。

这也是 PostgreSQL 同时保留两套状态的原因：

```text
PGPROC 字段:
  适合当前 backend 或单个 backend 的直接访问，局部性好。

ProcGlobal dense arrays:
  适合 ProcArray scan，减少跨 cacheline 跳转。
```

raw field 不是语义。`proc->xid`、`ProcGlobal->xids[proc->pgxactoff]`、`ProcArrayLock` / `XidGenLock`、backend 是否已经 `ProcArrayAdd()`，合在一起才构成“这个 XID 正在运行”的可观察语义。

## 5. 主流程源码 walkthrough

### 5.1 启动期：一次性创建 PGPROC 池

`ProcGlobalShmemRequest()` 在 shared memory request 阶段声明三类核心对象：

```text
"PGPROC structures":
  TotalProcs 个 PGPROC
  TotalProcs 个 mirrored xids
  TotalProcs 个 mirrored subxidStates
  TotalProcs 个 mirrored statusFlags

"Fast-Path Lock Array":
  每个 backend 的 fast-path lock bitmap 和 relation oid slots

"Proc Header":
  PROC_HDR 本身，也就是 ProcGlobal
```

随后 `ProcGlobalShmemInit()` 初始化全局进程表：

```text
ProcGlobalShmemInit()
  -> 初始化 ProcGlobal->freeProcsLock 和各 freelist
  -> 把 shared memory 切成 PGPROC[]、xids[]、subxidStates[]、statusFlags[]
  -> 为每个真实进程 PGPROC 初始化 semaphore、shared latch、fast-path lock arrays
  -> 把不同区间的 PGPROC 放入不同 freelist
  -> 初始化 myProcLocks[]、lockGroupMembers、group update atomic fields
  -> 设置 AuxiliaryProcs 和 PreparedXactProcs 指针
```

这里有两个设计点值得停一下：

第一，semaphore 在 postmaster 启动期就全部创建。源码注释解释过原因：如果等 backend 启动时再创建 semaphore，系统可能在负载高时才暴露内核 semaphore 上限不足，导致线上突然无法接收新连接。启动期一次性创建可以早失败。

第二，prepared transaction 也有 dummy PGPROC。它不是一个真实 OS process，但 prepared transaction 需要继续表现为持锁和事务仍可被观察，因此也要复用 `PGPROC` 这个 shared identity 抽象。这个点后续 2PC 课程会展开。

### 5.2 普通 backend：先认证相关初始化，再领取 PGPROC

普通客户端连接从 `backend_startup.c` 进入：

```text
BackendStartup()
  -> BackendInitialize()
     -> 初始化 libpq、读取 startup packet、认证
     -> 这段代码刻意不依赖 shared memory
  -> InitProcess()
     -> 创建当前 backend 的 PGPROC identity
  -> MemoryContextSwitchTo(TopMemoryContext)
  -> PostgresMain()
```

`BackendInitialize()` 的注释强调：这段认证前后逻辑不依赖 shared memory。在 `EXEC_BACKEND` 下，进程可能已经物理 attach 到 shared memory，但大多数本地 shared pointers 还没有接好。

真正进入 shared memory backend identity 的入口是 `InitProcess()`：

```text
InitProcess()
  -> 确认 ProcGlobal 已初始化，且 MyProc 还不存在
  -> RegisterPostmasterChildActive()
  -> 按 backend type 选择 freelist
  -> SpinLockAcquire(ProcGlobal->freeProcsLock)
  -> 从 freelist pop 一个 PGPROC
  -> 设置 MyProc 和 MyProcNumber
  -> 初始化 pid / backendType / vxid / xid / xmin / wait fields / lock fields
  -> OwnLatch(&MyProc->procLatch)
  -> SwitchToSharedLatch()
  -> pgstat_set_wait_event_storage(&MyProc->wait_event_info)
  -> PGSemaphoreReset(MyProc->sem)
  -> on_shmem_exit(ProcKill, 0)
  -> InitLWLockAccess()
  -> InitDeadLockChecking()
  -> EXEC_BACKEND 下 AttachSharedMemoryStructs()
```

这里的顺序非常紧：

| 步骤 | 为什么在这里 |
| --- | --- |
| pop PGPROC 前用 `freeProcsLock` | 还没有 PGPROC，不能用 LWLock 等待。 |
| `MyProcNumber = GetNumberFromPGProc(MyProc)` | 后续 wait queue、stats、slot ownership 都需要稳定 ProcNumber。 |
| 初始化 `pid`、`backendType`、`vxid.procNumber` | 其它进程可能通过 PGPROC 判断 slot 是否被占用、属于什么进程、如何引用它。 |
| `OwnLatch()` + `SwitchToSharedLatch()` | 之后 signal handler / wait path 使用的是当前 backend 的 shared latch。 |
| `pgstat_set_wait_event_storage()` | 之后 wait event 才能写入 shared memory，被观测到。 |
| `on_shmem_exit(ProcKill)` | 一旦 backend 拿到 PGPROC，就必须有退出兜底路径。 |
| `InitLWLockAccess()` | 现在已经有 PGPROC，可以参与 LWLock wait。 |

如果对应 freelist 为空，`InitProcess()` 会 FATAL：

```text
普通 backend:
  sorry, too many clients already

WAL sender:
  number of requested standby connections exceeds "max_wal_senders"
```

这不是普通 SQL ERROR。backend 此时无法建立完整 shared-memory identity，只能退出当前连接进程。

### 5.3 从 `MyProc` 到 `ProcArray`: 第二阶段公开身份

`InitProcess()` 完成后，当前 backend 已经能参与 LWLock、wait event、low-level shared memory 访问。但它还没有进入 `ProcArray`。

进入 `ProcArray` 的入口在 `postinit.c`：

```text
InitPostgres()
  -> InitProcessPhase2()
     -> ProcArrayAdd(MyProc)
     -> on_shmem_exit(RemoveProcFromArray, 0)
  -> pgstat_beinit()
  -> pgstat_bestart_initial()
  -> SharedInvalBackendInit(false)
  -> ProcSignalInit(...)
  -> 后续数据库、role、catalog cache、事务系统初始化
```

`InitPostgres()` 的注释很直接：

```text
Once I have done this, I am visible to other backends!
```

这就是两阶段初始化的核心：

```text
InitProcess:
  我有了 shared-memory identity，可以等待、被唤醒、拿 LWLock、做低层初始化。

InitProcessPhase2:
  我进入 ProcArray，成为其它 backend snapshot / xact horizon / backend scan 的一员。
```

为什么不在 `InitProcess()` 里立刻 `ProcArrayAdd()`？

源码注释给出两个约束：

```text
1. 我们不能在创建 PGPROC 之前获取 LWLock。
2. EXEC_BACKEND 下，ProcArrayAdd 要等 AttachSharedMemoryStructs() 之后才工作。
```

所以 PostgreSQL 选择拆成：

```text
InitProcess()
  先建立能使用 LWLock 和 attach shared structures 的最小身份。

InitProcessPhase2()
  shared pointers 和更高层初始化条件满足后，再加入 ProcArray。
```

### 5.4 `ProcArrayAdd()`: 把 PGPROC 放入全局扫描数组

`ProcArrayAdd()` 的核心动作：

```text
ProcArrayAdd(proc)
  -> LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE)
  -> LWLockAcquire(XidGenLock, LW_EXCLUSIVE)
  -> 按 PGPROC number 排序插入 procArray->pgprocnos[]
  -> memmove dense arrays: xids[] / subxidStates[] / statusFlags[]
  -> proc->pgxactoff = 插入位置
  -> dense arrays 初始化为当前 PGPROC 的 xid / subxidStatus / statusFlags
  -> arrayP->numProcs++
  -> 调整后续 PGPROC 的 pgxactoff
  -> 释放 XidGenLock
  -> 释放 ProcArrayLock
```

这里同时持有 `ProcArrayLock` 和 `XidGenLock`。`proc.h` 说明了原因：dense arrays 被 `GetNewTransactionId()` 和 `GetSnapshotData()` 等路径访问；添加或删除 ProcArray entry 会改变 `pgxactoff`，必须阻止并发 reader / writer 在数组移动时看到不一致位置。

排序插入不是语义要求，而是 hot path 优化。ProcArray scan 很频繁，让 `pgprocnos[]` 按 PGPROC 地址顺序排列，可以改善访问局部性。

本节只需要记住：

```text
ProcArray membership 是 PGPROC identity 的公开阶段；
进入 ProcArray 后，dense arrays 和 pgxactoff 的一致性由 ProcArrayLock / XidGenLock 边界维护。
```

### 5.5 transaction XID 发布：依赖已经进入 ProcArray

当事务后来真正分配 XID，`varsup.c` 的 `GetNewTransactionId()` 会在持有 `XidGenLock` 时写：

```text
MyProc->xid = xid
ProcGlobal->xids[MyProc->pgxactoff] = xid
```

这要求：

```text
MyProc 已存在；
MyProc 已经 ProcArrayAdd；
MyProc->pgxactoff 当前有效；
调用者持有 XidGenLock。
```

`access/transam/README` 解释了 correctness 规则：新 XID 必须在释放 `XidGenLock` 前写入 shared ProcArray，否则其它 backend 可能分配并提交更晚的 XID，让 `latestCompletedXid` 越过一个尚未出现在 ProcArray 中的旧 XID，破坏 snapshot / horizon 计算。

这部分的完整推理属于下一节和 snapshot 课程。本节只抓住 identity 层结论：

```text
没有 PGPROC slot，就没有 MyProc->pgxactoff；
没有 ProcArray membership，就不能把当前事务发布到全局运行事务集合里。
```

### 5.6 auxiliary process：有 PGPROC，但不是完整 backend

auxiliary process 走 `AuxiliaryProcessMainCommon()`：

```text
AuxiliaryProcessMainCommon()
  -> InitAuxiliaryProcess()
  -> BaseInit()
  -> ProcSignalInit(NULL, 0)
  -> 后续进入 checkpointer / bgwriter / walwriter / startup 等主函数
```

`InitAuxiliaryProcess()` 和 `InitProcess()` 很像，但有几个关键差异：

| 普通 backend | auxiliary process |
| --- | --- |
| 从 `freeProcs` / `bgworkerFreeProcs` / `walsenderFreeProcs` 等 freelist pop。 | 在固定 `AuxiliaryProcs[]` 中找 `pid == 0` 的 slot。 |
| 初始化 `vxid.procNumber = MyProcNumber`。 | 基础路径设置 `vxid.procNumber = INVALID_PROC_NUMBER`。 |
| 注册 `ProcKill`。 | 注册 `AuxiliaryProcKill`。 |
| 初始化 deadlock checker。 | 不初始化 deadlock checker。 |
| 后续 `InitPostgres()` 会 `ProcArrayAdd()`。 | 通常不走完整 `InitPostgres()`，也不加入普通 ProcArray。 |

`proc.c` 注释说得很清楚：auxiliary process 需要一个“real enough”的 `MyProc` 来等待 LWLock，但它们不是完整 backend，不能运行普通事务，也不参与普通 heavyweight lock 等待模型。

因此本节要区分两个层次：

```text
有 PGPROC:
  可以作为 shared memory wait / latch / LWLock participant。

进入 ProcArray:
  成为事务可见性和 backend scan 的普通成员。
```

不是所有有 PGPROC 的进程都等价于普通 SQL backend。

## 6. 生命周期 / ownership / cleanup

### 谁创建？

postmaster 或 standalone backend 在 shared memory 初始化阶段创建整个 PGPROC 池：

```text
ProcGlobalShmemRequest()
  声明 "PGPROC structures" 和 "Proc Header"

ProcGlobalShmemInit()
  初始化 ProcGlobal、PGPROC[]、dense arrays、semaphore、shared latch、freelists
```

单个 backend 进程不 malloc 自己的 PGPROC。它只是从 shared memory 池里领取一个已经预留好的 slot。

### 谁持有？

运行中的 backend 通过本进程全局变量持有自己的 slot：

```text
MyProc:
  指向当前进程的 PGPROC。

MyProcNumber:
  当前 PGPROC 在 ProcGlobal->allProcs[] 里的编号。
```

其它 backend 不能持有这个 backend-local 指针本身，但可以通过 shared memory 中的 `ProcNumber` / `PGPROC` 字段观察或唤醒它。例如 LWLock wait list 存 `ProcNumber`，lock manager 的 wait queue 挂 `PGPROC` 链接，synchronous replication queue 使用 `syncRepLinks`。

### 谁释放？

领取 PGPROC 后，`InitProcess()` 注册：

```text
on_shmem_exit(ProcKill, 0)
```

加入 ProcArray 后，`InitProcessPhase2()` 再注册：

```text
on_shmem_exit(RemoveProcFromArray, 0)
```

`shmem_exit()` 调用 `on_shmem_exit` callbacks 时按后进先出执行。因此对于普通已经完成 `InitPostgres()` 的 backend，退出顺序是：

```text
RemoveProcFromArray()
  -> ProcArrayRemove(MyProc, InvalidTransactionId)
  -> 先让其它 backend 不再从 ProcArray 看到它

ProcKill()
  -> 清理 sync rep / LWLock / WaitLSN / condition variable / lock group
  -> 切回 local latch
  -> 停止 wait event reporting
  -> MyProc = NULL
  -> MyProcNumber = INVALID_PROC_NUMBER
  -> DisownLatch()
  -> proc->pid = 0
  -> proc->vxid = invalid
  -> 把 PGPROC 放回对应 freelist
```

这个顺序非常重要：

```text
先从 ProcArray 消失，再释放 PGPROC slot。
```

否则新 backend 可能复用同一个 PGPROC，而旧 backend 的 ProcArray entry 还没移除，扫描者会把新旧状态混在一起。

### ERROR / abort 时谁兜底？

普通 SQL `ERROR` 不会释放 PGPROC。它只 abort 当前事务，backend 继续服务下一条命令。

PGPROC 的释放属于进程退出路径：

```text
FATAL / proc_exit / postmaster child exit:
  shmem_exit()
    -> before_shmem_exit callbacks
    -> dsm_backend_shutdown()
    -> on_shmem_exit callbacks
       -> RemoveProcFromArray
       -> ProcKill
```

如果 backend 在已经拿到 PGPROC 后异常退出，`ProcKill()` 是低层兜底。它会再次 `LWLockReleaseAll()`，取消 condition variable sleep，切回 local latch，把 slot 标为可复用。

如果进程不是正常 `proc_exit()`，postmaster 还有 child active / crash detection 机制处理更坏的死亡方式。本节只需要知道：`InitProcess()` 一旦成功，就必须注册 `ProcKill()`，因为 shared memory 里已经有其它进程可能引用这个 identity。

## 7. 正确性机制层次

### 层次一：freelist 分配用 spinlock

`ProcGlobal->freeProcsLock` 保护各类 PGPROC freelist。它必须是 spinlock，因为：

```text
LWLock 等待需要 MyProc；
而分配 MyProc 之前还没有 PGPROC；
所以不能用 LWLock 保护 PGPROC freelist。
```

这个锁只保护很短的 pop / push 临界区，不在 hot query path 上长期持有。

### 层次二：ProcArray membership 用 ProcArrayLock + XidGenLock

`ProcArrayAdd()` / `ProcArrayRemove()` 同时持有：

```text
ProcArrayLock:
  保护 procArray->pgprocnos[] 和 snapshot scan 相关结构。

XidGenLock:
  保护 XID 分配与 dense xids[] 位置的一致性边界。
```

这不是为了“分配 PGPROC”，而是为了维护：

```text
pgprocnos[] 排序数组
pgxactoff
ProcGlobal->xids[]
ProcGlobal->subxidStates[]
ProcGlobal->statusFlags[]
```

在 add/remove 时不会被 snapshot 或 XID 分配路径看到半移动状态。

### 层次三：XID 发布依赖 XidGenLock 和 memory barrier

`GetNewTransactionId()` 在持有 `XidGenLock` 时把新 XID 写到：

```text
MyProc->xid
ProcGlobal->xids[MyProc->pgxactoff]
```

对于 subxid cache，还使用 `pg_write_barrier()`，避免其它 backend 看到“count 已增加但数组元素还没填好”的中间状态。

本节不展开 snapshot 证明，只保留结论：

```text
PGPROC slot 提供发布位置；
ProcArray membership 提供扫描入口；
XidGenLock / ProcArrayLock 提供 ordering 边界。
```

### 层次四：exit callback 的 LIFO 顺序

`on_shmem_exit()` callback 后注册的先执行：

```text
InitProcess()
  注册 ProcKill

InitProcessPhase2()
  注册 RemoveProcFromArray

退出:
  先 RemoveProcFromArray
  后 ProcKill
```

这个顺序服务一个不变量：

```text
只要 PGPROC 仍在 ProcArray 中，就不能把它放回 freelist 给新 backend 复用。
```

## 8. 错误路径 / 异常路径 / fallback

### PGPROC slot 不够

`InitProcess()` 从对应 freelist pop 失败时，会 FATAL：

```text
普通 backend:
  sorry, too many clients already

WAL sender:
  number of requested standby connections exceeds "max_wal_senders"
```

这时 backend 还没有完整 shared-memory identity，不能进入普通 SQL 错误恢复路径。

诊断时要看具体进程类型：

| 现象 | 可能对应的 slot 池 |
| --- | --- |
| 普通连接报 too many clients | `freeProcs`，通常由 `max_connections` 限制。 |
| 复制连接超过上限 | `walsenderFreeProcs`，看 `max_wal_senders`。 |
| background worker 起不来 | `bgworkerFreeProcs`，看 `max_worker_processes`。 |
| autovacuum worker 起不来 | `autovacFreeProcs`，看 autovacuum worker / special worker 相关上限。 |

### `ProcArrayAdd()` 理论上没空间

`ProcArrayAdd()` 也检查 `arrayP->numProcs >= arrayP->maxProcs`。源码注释说这通常不应发生，因为 PGPROC 固定供应应该先限制住进程数量。这里仍然 FATAL，是为了防御 ProcArray 状态和 PGPROC 池不一致。

### backend 在 phase2 前退出

如果 backend `InitProcess()` 成功，但还没 `InitProcessPhase2()` 就 FATAL：

```text
已注册:
  ProcKill

未注册:
  RemoveProcFromArray
```

退出时只需要释放 PGPROC slot，不需要从 ProcArray 移除，因为它从未对其它 backend 作为 ProcArray member 可见。

这正是两阶段注册 cleanup callback 的意义：cleanup 范围和已经完成的生命周期阶段一致。

### auxiliary process cleanup

auxiliary process 注册的是 `AuxiliaryProcKill()`。它不会把 slot 放回 freelist，只是：

```text
LWLockReleaseAll()
ConditionVariableCancelSleep()
SwitchBackToLocalLatch()
pgstat_reset_wait_event_storage()
MyProc = NULL
MyProcNumber = INVALID_PROC_NUMBER
pid = 0
vxid = invalid
```

因为 auxiliary slots 是固定数组，通过 `pid == 0` 判断空闲。

## 9. 成本、资源与跨模块传播

### 启动期资源成本

PGPROC 池在启动期按上限定容。相关配置会影响 shared memory：

```text
max_connections
autovacuum_worker_slots
max_worker_processes
max_wal_senders
max_prepared_transactions
NUM_AUXILIARY_PROCS
FastPathLockGroupsPerBackend
```

成本不只是 `sizeof(PGPROC)`：

```text
每个真实进程 PGPROC:
  semaphore
  shared latch
  fast-path lock bits / relation oid arrays
  myProcLocks[] heads
  wait fields

每个 ProcArray member:
  dense xids[] / subxidStates[] / statusFlags[] entry
```

所以提高 backend 上限会同时放大 shared memory、semaphore、lock table 相关压力，也会放大某些 O(N backends) scan 的上界。

### hot path 成本

PGPROC 分配和释放不是每条 SQL 的 hot path；它发生在 backend startup / shutdown。

真正的 hot path 是：

```text
ProcArray scan:
  GetSnapshotData()
  ComputeXidHorizons()

wait / lock path:
  LWLock wait queue
  heavyweight lock wait queue
  deadlock checking

commit / abort:
  ProcArrayEndTransaction()
  group XID clearing
```

本节不展开这些路径，但要看到为什么 `PGPROC` 结构里放了很多看似不相干的字段：它是多个 hot path 共用的 per-backend shared identity。

### 跨模块传播

`MyProc` 一旦建立，会被很多模块当作“当前 backend 的 shared handle”：

| 模块 | 传播方式 |
| --- | --- |
| LWLock | wait queue 使用 `MyProcNumber`，唤醒使用 `MyProc->sem` / latch 相关状态。 |
| lock manager | `PROCLOCK`、`waitLock`、`myProcLocks[]` 把锁持有者和等待者挂回 PGPROC。 |
| stats | wait event storage 指向 `MyProc->wait_event_info`。 |
| transaction | `MyProc->xid`、`xmin`、`subxids` 发布事务状态。 |
| snapmgr | active snapshot 变化会更新 `MyProc->xmin`。 |
| replication | sync replication queue 使用 `syncRepLinks`、`waitLSN`、`syncRepState`。 |
| procsignal | cancel key、barrier、recovery conflict 等需要能定位 backend。 |
| async notification / replication slot 等 | 常用 `MyProcNumber` 标记 slot owner 或 listener。 |

这也是为什么 PGPROC cleanup 不能只是“free 一个结构体”。它要先把多个模块的共享队列、等待状态和 observable state 退干净。

## 10. 观测与诊断入口

### SQL 入口

`PGPROC` 本身没有直接暴露成一张系统表，但可以从这些视图间接观察：

```sql
select pid, backend_type, datname, usename, state, wait_event_type, wait_event
from pg_stat_activity
order by pid;
```

能看到：

```text
pid / backend_type:
  来自 backend identity 和 stats 层。

wait_event:
  依赖 MyProc->wait_event_info 的 wait event storage。
```

观察 VXID / lock identity：

```sql
select pid, locktype, virtualxid, transactionid, mode, granted
from pg_locks
where pid is not null
order by pid, locktype;
```

这里看到的是 lock manager 和 ProcArray / VXID 语义的投影，不是完整 PGPROC dump。

### 日志入口

当 PGPROC slot 不够时，典型日志 / client error：

```text
FATAL: sorry, too many clients already
```

WAL sender slot 不够时：

```text
FATAL: number of requested standby connections exceeds "max_wal_senders" (...)
```

这类错误发生在 backend identity 建立阶段，不能按普通 query ERROR 来理解。

### gdb 入口

调试单个 backend 时，常见检查：

```gdb
p MyProc
p MyProcNumber
p *MyProc
p ProcGlobal->allProcCount
p ProcGlobal->freeProcs
p ProcGlobal->xids[MyProc->pgxactoff]
p MyProc->xid
p MyProc->xmin
p MyProc->wait_event_info
```

如果怀疑退出清理顺序，可以在这些函数上断点：

```gdb
b InitProcess
b InitProcessPhase2
b ProcArrayAdd
b RemoveProcFromArray
b ProcKill
b shmem_exit
```

预期顺序：

```text
启动:
  InitProcess
  InitProcessPhase2
  ProcArrayAdd

退出:
  shmem_exit
  RemoveProcFromArray
  ProcKill
```

### 源码 trace 入口

用 `rg` 快速定位当前课程主链路：

```bash
rg -n "InitProcess\\(|InitProcessPhase2\\(|ProcArrayAdd\\(|ProcKill\\(|RemoveProcFromArray" /home/highgo/postgres/src
```

用 `rg` 看哪些模块依赖 `MyProcNumber`：

```bash
rg -n "MyProcNumber" /home/highgo/postgres/src/backend /home/highgo/postgres/src/include
```

这能帮助你建立直觉：`MyProcNumber` 是 PostgreSQL shared memory 里的“进程编号”，远比 connection pid 更适合做数组索引和 wait queue 链接。

## 11. 常见误区

### 误区一：PGPROC 等于客户端连接

不对。普通客户端 backend 有 PGPROC，但 background worker、WAL sender、autovacuum worker、auxiliary process、prepared transaction dummy entry 也可能有 PGPROC。PGPROC 是 shared-memory process identity，不是 libpq connection object。

### 误区二：有 PGPROC 就一定在 ProcArray 里

不对。`InitProcess()` 后已经有 `MyProc`，但 `InitProcessPhase2()` 之前还没进入 ProcArray。auxiliary process 通常有 PGPROC 但不走普通 ProcArray membership。

### 误区三：`pid` 是唯一可靠身份

`pid` 用来识别 OS process，但 shared memory 内部大量结构使用 `ProcNumber` / `PGPROC`。`pid` 会随进程退出失效，slot 会复用。判断 shared memory ownership 时要看 lifecycle 和 cleanup，不要只看 pid。

### 误区四：PGPROC slot 可以运行时扩容

传统 main shared memory 里不行。PGPROC 池在启动期按配置上限定容，slot 数不足时只能拒绝新进程。要动态扩展通常意味着 DSM 或完全不同的注册机制，但普通 backend identity 不走这条路。

### 误区五：退出时只要把 `pid = 0` 就够了

不够。退出必须先退出 ProcArray，清理 wait queue / sync rep / LWLock / condition variable / lock group / latch / wait event storage，再把 slot 标为空闲。否则其它 backend 可能扫描到 stale transaction state，或者唤醒一个已经不拥有该 latch 的进程。

## 12. 课堂实验

### 实验一：跟踪一个普通 backend 的 identity 建立

目标：在 gdb 中看到 `InitProcess()` 与 `InitProcessPhase2()` 的分界。

步骤：

```bash
gdb -p <new-backend-pid>
```

建议配合 `PreAuthDelay` 或 `PostAuthDelay` 给自己留 attach 时间。

断点：

```gdb
b InitProcess
b InitProcessPhase2
b ProcArrayAdd
continue
```

观察：

```gdb
p MyProc
p MyProcNumber
p MyProc->pid
p MyProc->vxid
p MyProc->pgxactoff
```

预期：

```text
InitProcess 刚完成:
  MyProc 非空
  MyProcNumber 有效
  vxid.procNumber 已设置
  但还没有通过 InitProcessPhase2 进入 ProcArray

ProcArrayAdd 后:
  MyProc->pgxactoff 对应 dense arrays 中的位置
  当前 backend 对其它 backend 可见
```

### 实验二：观察 PGPROC slot 上限

目标：把连接数打到 `max_connections` 上限，观察错误发生在哪个生命周期阶段。

步骤：

```sql
show max_connections;
```

开多个连接直到超过上限。

预期错误：

```text
FATAL: sorry, too many clients already
```

回到源码解释：

```text
InitProcess()
  -> 选择 ProcGlobal->freeProcs
  -> freelist empty
  -> ereport(FATAL, ERRCODE_TOO_MANY_CONNECTIONS)
```

这个错误不是 SQL 执行期资源泄漏，而是 shared-memory backend identity slot 已经耗尽。

### 实验三：观察退出 callback 顺序

目标：确认 `RemoveProcFromArray` 先于 `ProcKill`。

gdb 断点：

```gdb
b shmem_exit
b RemoveProcFromArray
b ProcKill
continue
```

关闭该 backend 连接。

预期顺序：

```text
shmem_exit
RemoveProcFromArray
ProcKill
```

回到源码解释：

```text
InitProcess 先注册 ProcKill；
InitProcessPhase2 后注册 RemoveProcFromArray；
on_shmem_exit 按 LIFO 执行；
因此 RemoveProcFromArray 先执行。
```

### 实验四：区分 pid、MyProcNumber 和 VXID

SQL 观察：

```sql
select pg_backend_pid();

begin;
select pid, locktype, virtualxid, mode, granted
from pg_locks
where pid = pg_backend_pid()
order by locktype, mode;
rollback;
```

gdb 对同一 backend 观察：

```gdb
p MyProcPid
p MyProcNumber
p MyProc->vxid
```

思考：

```text
pid:
  OS process identity。

MyProcNumber:
  shared memory PGPROC slot identity。

vxid:
  ProcNumber + local transaction id，服务事务和 lock 可见性。
```

## 13. 讨论题

1. 为什么 `ProcGlobal->freeProcsLock` 不能换成 LWLock？如果这么做，backend 启动会卡在哪个自举依赖上？
2. 如果 `InitProcess()` 直接调用 `ProcArrayAdd()`，`EXEC_BACKEND` 和 shared memory attach 顺序会遇到什么问题？
3. 为什么退出时必须先 `ProcArrayRemove()`，再 `ProcKill()` 把 PGPROC 放回 freelist？
4. `PGPROC->pid`、`MyProcNumber`、`vxid.procNumber` 三者分别服务什么场景？哪个可以作为数组索引？
5. 为什么 `pgxactoff` 不能当作稳定 backend identity？
6. prepared transaction 没有真实进程，为什么还需要 dummy PGPROC？
7. 如果一个 backend 在 `InitProcess()` 后、`InitProcessPhase2()` 前 FATAL，应该执行哪些 cleanup？哪些 cleanup 不应该执行？

## 14. 本节小结

本节的可迁移系统规律：

```text
多进程 shared memory 系统必须先建立可被其它进程引用的 shared identity，
再让进程参与等待、唤醒、全局扫描和退出清理；
identity 的公开范围要分阶段推进，cleanup 也必须按相反顺序收回。
```

在 PostgreSQL 中，这个规律具体落到：

```text
ProcGlobalShmemInit:
  启动期创建固定 PGPROC 池。

InitProcess:
  当前进程领取 PGPROC，建立 MyProc / MyProcNumber / shared latch / wait state。

InitProcessPhase2:
  把 MyProc 加入 ProcArray，让其它 backend 能扫描到它。

ProcArrayAdd / Remove:
  维护 pgprocnos[]、pgxactoff 和 dense mirror arrays。

ProcKill:
  释放 wait / latch / lock group 等低层状态，把 slot 放回 freelist。
```

一句话带走：

```text
PGPROC 是 backend 进入 PostgreSQL shared memory 世界的身份证；
ProcArray membership 是这张身份证对事务可见性系统的公开登记。
```

下一节可以继续追问：一个已经进入 ProcArray 的 backend，如何把 XID、SubXID、xmin 和 status flags 发布给其它 backend，并让 snapshot / VACUUM / recovery 读到一致语义？
