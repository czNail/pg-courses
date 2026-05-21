# PostgreSQL ProcArray membership 与事务状态发布

## 课程定位

前置知识：已经理解 `PGPROC` slot 是 backend 进入 shared memory 世界的身份，也知道 `InitProcess()` 只完成低层身份初始化，`InitProcessPhase2()` 才把 backend 加入 `ProcArray`。

本节唯一主问题：

```text
一个 backend 什么时候进入 ProcArray，如何把 XID、SubXID、xmin、vacuum flags 等事务状态发布给其它 backend，为什么这些字段不能只保存在 backend-local 状态里？
```

核心矛盾：事务状态是每个 backend 自己产生的，但 MVCC snapshot、VACUUM horizon、prepared transaction、logical decoding、hot standby feedback、commit ordering 都需要其它进程在正确的时间看到这些状态。PostgreSQL 因此把一部分事务状态从 backend-local 提升为 shared-memory publication，并用 `ProcArrayLock`、`XidGenLock`、dense mirror arrays 和内存屏障约束它们的可见顺序。

学完后应能判断：

```text
PGPROC slot 和 ProcArray membership 有什么区别；
backend 什么时候只有 virtual transaction，什么时候才有真实 XID；
GetNewTransactionId() 为什么必须在释放 XidGenLock 前发布 XID；
GetSnapshotData() 为什么扫描 ProcArray 而不是问每个 backend；
ProcArrayEndTransaction() 为什么必须在释放锁之前清掉 XID；
xmin 和 statusFlags 为什么共同决定 VACUUM horizon；
SubXID cache overflow 为什么不是性能细节，而是 correctness fallback。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节讲的是：

```text
postmaster 启动期预留 PGPROC 池
  -> backend 启动时领取一个 PGPROC slot
  -> 初始化 MyProc / MyProcNumber / latch / semaphore / wait fields
  -> 退出时释放 slot
```

那只是让 backend 有了 shared identity。一个已经有 `MyProc` 的进程，还不等于已经参与事务可见性判断。

本节继续往前走一步：

```text
InitProcessPhase2()
  -> ProcArrayAdd(MyProc)
  -> MyProc 出现在 procArray->pgprocnos[]
  -> ProcGlobal->xids[] / subxidStates[] / statusFlags[] 有了对应 dense entry
  -> 其它 backend 可以把这个 backend 纳入 snapshot / horizon / xact cleanup 判断
```

从这一刻起，backend 的事务状态不再只是本地变量。它必须回答三个全局问题：

| 全局问题 | 需要发布的状态 |
| --- | --- |
| 这个事务是否还在运行？ | `xid`、`subxids`、`subxidStatus`、`latestCompletedXid`。 |
| 哪些旧 tuple 版本还不能删除？ | `xmin`、当前 running XID、replication slot xmin、logical decoding 状态。 |
| 这个 backend 是否应被某些 horizon 计算跳过或特殊对待？ | `statusFlags`，例如 `PROC_IN_VACUUM`、`PROC_IN_LOGICAL_DECODING`、`PROC_AFFECTS_ALL_HORIZONS`。 |

注意本节不展开 snapshot 算法的全部细节，也不讲 ProcArray scan 的扩展性优化。这里的重点是 publication contract：

```text
backend-local transaction state 什么时候变成 shared fact；
shared fact 什么时候被其它 backend 读取；
状态进入和退出 shared fact 集合时有哪些 ordering 约束。
```

下一节会沿着 `GetSnapshotData()` 的 hot path，专门讨论 ProcArray scan 的成本和扩展性。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
backend 拿到 PGPROC 后，通过 ProcArrayAdd() 成为 ProcArray member；
事务开始时先发布 virtual transaction id；
真正需要写入或生成 XID 时，GetNewTransactionId() 在 XidGenLock 下把 top XID 或 SubXID 发布到 MyProc 和 ProcGlobal dense arrays；
建立 snapshot 时，GetSnapshotData() 在 ProcArrayLock shared 下扫描这些 published states；
提交或回滚时，ProcArrayEndTransaction() 在事务已写入 WAL / pg_xact 后、释放锁之前，清除 XID / xmin / SubXID / vacuum flags 并推进 completion state。
```

这背后的 tension 是：

```text
事务状态必须足够全局，才能让其它 backend 构造一致 snapshot 和 cleanup horizon；
但事务状态更新发生在高频路径，不能每次都把所有状态塞进一个巨大的、重锁保护的结构里。
```

PostgreSQL 的答案不是“所有事务状态都放 ProcArray”，也不是“backend 自己保存，别人需要时来问”。它做了一个分层发布：

| 层级 | 保存位置 | 典型访问者 | 设计目的 |
| --- | --- | --- | --- |
| backend-local transaction state | `xact.c` 的 `TransactionStateData`、`CurrentTransactionState` 等 | 当前 backend | 完整表达本地事务生命周期、子事务栈、command id、resource owner。 |
| shared per-backend state | `PGPROC` | 当前 backend 和少量按 proc 定位的读者 | 让其它进程能看到这个 backend 当前是否有 XID、xmin、SubXID、vacuum flags。 |
| dense mirror arrays | `ProcGlobal->xids[]`、`subxidStates[]`、`statusFlags[]` | ProcArray scan hot path | 让 snapshot / horizon 计算用更少 cache miss 扫描大量 backend。 |
| global transaction ordering state | `TransamVariables->latestCompletedXid`、`xactCompletionCount` | snapshot reuse、visibility、horizon 计算 | 维护 commit / abort 与 snapshot-taking 的全局顺序。 |

本节的核心抽象是：

```text
ProcArray 不是“连接数组”，而是事务状态的 publication set。

一个 backend 在 ProcArray 中，不代表它当前一定有 XID；
它只是承诺：如果我产生会影响其它 backend 的事务状态，会按 ProcArray 的规则发布和清除。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/proc.h` | `PGPROC` 中 `vxid`、`xid`、`xmin`、`subxidStatus`、`statusFlags`，以及 `PROC_HDR` 的 dense arrays 约束。 |
| 2 | `src/backend/storage/ipc/procarray.c` | `ProcArrayAdd()`、`ProcArrayEndTransaction()`、`GetSnapshotData()`、`ComputeXidHorizons()` 如何读写 published transaction state。 |
| 3 | `src/backend/access/transam/xact.c` | `StartTransaction()` 何时发布 VXID，commit / abort 何时调用 `ProcArrayEndTransaction()`。 |
| 4 | `src/backend/access/transam/varsup.c` | `GetNewTransactionId()` 如何分配 XID 并发布 top XID / SubXID。 |
| 5 | `src/backend/access/transam/README` | transaction begin/end 与 snapshot 的 interlocking 规则。 |
| 6 | `src/backend/commands/vacuum.c` | lazy VACUUM 如何设置 `PROC_IN_VACUUM`，以及为什么 flags 必须和 xmin 顺序配合。 |
| 7 | `src/backend/access/transam/twophase.c` | prepared transaction 如何用 dummy PGPROC 继续占据 ProcArray。 |
| 8 | `src/backend/replication/walsender.c`、`src/backend/replication/slot.c`、`src/backend/replication/logical/logical.c` | replication slot、hot standby feedback、logical decoding 如何影响 xmin / status flags。 |

推荐阅读顺序：

```text
先读 proc.h 中 PGPROC transaction fields 和 PROC_HDR dense arrays 注释
  -> 读 ProcArrayAdd() 理解 membership 和 pgxactoff
  -> 读 xact.c 的 StartTransaction() 理解 VXID 先于 XID
  -> 读 varsup.c 的 GetNewTransactionId() 理解 XID / SubXID 发布
  -> 读 procarray.c 的 GetSnapshotData() 理解读者如何消费这些状态
  -> 读 ProcArrayEndTransaction() 理解状态如何退出 publication set
  -> 最后读 ComputeXidHorizons() 和 vacuum.c 理解 xmin / statusFlags 的 cleanup 边界
```

不要把 `ProcArray` 只理解为 `pg_stat_activity` 背后的 backend list。`pg_stat_activity` 展示 session 活动，而 `ProcArray` 的核心语义是 MVCC 和 cleanup correctness。

## 4. 关键数据结构与状态

### `ProcArrayStruct`: membership list，不保存完整事务对象

`ProcArrayStruct` 定义在 `src/backend/storage/ipc/procarray.c`。本节关注它的几个字段：

| 字段 | 语义 |
| --- | --- |
| `numProcs` | 当前进入 ProcArray 的 PGPROC 数量。 |
| `maxProcs` | ProcArray 可容纳的最大 membership 数。 |
| `pgprocnos[]` | ProcArray 成员对应的 `ProcGlobal->allProcs[]` 下标。 |
| `replication_slot_xmin` | replication slots 对普通 tuple cleanup 的全局 xmin 约束。 |
| `replication_slot_catalog_xmin` | replication slots 对 catalog cleanup 的全局 xmin 约束。 |

`pgprocnos[]` 保存的是 proc number，不是 `PGPROC *`。这让不同进程都能通过：

```c
PGPROC *proc = &ProcGlobal->allProcs[pgprocno];
```

重新定位目标 backend 的 shared state。

`ProcArrayAdd()` 会让一个 `PGPROC` 加入 `pgprocnos[]`。它同时初始化三组 dense mirror arrays：

```text
ProcGlobal->xids[index]
ProcGlobal->subxidStates[index]
ProcGlobal->statusFlags[index]
```

这三组数组只对已经进入 ProcArray 的 PGPROC 有意义。一个 backend 拿到了 `MyProc`，但还没执行 `ProcArrayAdd()`，它就没有合法的 dense array entry。

### `PGPROC->pgxactoff`: dense arrays 的移动下标

`PGPROC->pgxactoff` 是当前 backend 在 dense arrays 中的 offset：

```text
ProcGlobal->xids[MyProc->pgxactoff]
ProcGlobal->subxidStates[MyProc->pgxactoff]
ProcGlobal->statusFlags[MyProc->pgxactoff]
```

它不是稳定身份。`ProcArrayAdd()` / `ProcArrayRemove()` 会为了保持 `pgprocnos[]` 排序而 `memmove()` dense arrays，并更新后续 procs 的 `pgxactoff`。

因此 `proc.h` 的注释明确了一个重要规则：

```text
只有持有 ProcArrayLock 或 XidGenLock 时，才能安全用 pgxactoff 访问 dense arrays。
```

这条规则非常容易被忽略。`MyProcNumber` 是 slot identity；`pgxactoff` 是 ProcArray scan 优化用的当前 dense offset。二者不是一回事。

### `xid`: top-level transaction 是否正在运行

`PGPROC->xid` 表示当前 backend 正在执行的 top-level transaction XID。无 XID 的事务，比如只读事务、尚未触发 XID 分配的事务，`xid` 为 `InvalidTransactionId`。

这个字段被镜像到：

```text
ProcGlobal->xids[pgxactoff]
```

读者通常扫描 dense `xids[]`，因为 snapshot hot path 关心的是“所有 backend 的 XID 集合”，dense layout 比从 `PGPROC` 逐个跳转更适合 CPU cache。

重要的是：

```text
事务开始不等于 XID 发布。
```

`StartTransaction()` 会发布 VXID：

```text
vxid.procNumber = MyProcNumber
vxid.localTransactionId = GetNextLocalTransactionId()
VirtualXactLockTableInsert(vxid)
MyProc->vxid.lxid = vxid.localTransactionId
```

真正的 XID 由 `GetNewTransactionId()` 延迟分配。只有当事务需要写入、需要永久事务身份、需要进入 clog/subtrans 等路径时，才会发布 `MyProc->xid` 和 `ProcGlobal->xids[]`。

### `SubXID`: 小缓存加 overflow fallback

`PGPROC` 中有两部分 SubXID 状态：

| 字段 | 语义 |
| --- | --- |
| `subxids.xids[]` | 当前 top transaction 下已缓存的 subtransaction XIDs。 |
| `subxidStatus.count` | 当前缓存了多少个 SubXID。 |
| `subxidStatus.overflowed` | 是否超过 `PGPROC_MAX_CACHED_SUBXIDS`，需要读者回退到 `pg_subtrans`。 |

`subxidStatus` 也镜像到：

```text
ProcGlobal->subxidStates[pgxactoff]
```

但实际 SubXID 数组仍在 `PGPROC->subxids.xids[]` 里。读者先看 dense `subxidStates[]` 判断 count / overflow，再用 `pg_read_barrier()` 后从 `PGPROC` 读取 subxid array。

这里的状态组合才是语义：

```text
count = n, overflowed = false:
  前 n 个 subxids 可以从 PGPROC cache 读取。

overflowed = true:
  PGPROC cache 不再完整，读者必须认为可能有更多 subxids，需要查 pg_subtrans 或使用更保守判断。
```

### `xmin`: 当前 backend 对 cleanup horizon 的承诺

`PGPROC->xmin` 表示当前 backend 的活跃 snapshot 需要保留到哪里。

直觉上可以理解为：

```text
如果某个 tuple version 是被 xid >= MyProc->xmin 的事务删除的，
VACUUM 不能仅因为“我看起来不需要”就删除它，因为这个 backend 的 snapshot 可能还需要它。
```

但这个解释必须加上两个边界：

1. `xmin` 与 `xid` 一起参与 horizon 计算。一个事务可能已经有 XID 但还没设置 xmin，`ComputeXidHorizons()` 会同时考虑 `xid` 和 `xmin`。
2. `xmin` 的解释会被 `statusFlags` 改写。lazy VACUUM、logical decoding、hot standby feedback 等路径不按普通 user transaction 的方式影响所有 horizon。

所以本节要记住：

```text
xmin 不是单字段语义；
xmin + xid + statusFlags + databaseId + ProcArrayLock context 才能决定 cleanup boundary。
```

### `statusFlags`: 给 horizon 和特殊事务路径看的小型协议

`statusFlags` 定义在 `proc.h`，既存在于 `PGPROC->statusFlags`，也镜像到 `ProcGlobal->statusFlags[pgxactoff]`。

本节关注这些 flags：

| flag | 语义 |
| --- | --- |
| `PROC_IS_AUTOVACUUM` | 当前 backend 是 autovacuum worker。 |
| `PROC_IN_VACUUM` | 当前正在 lazy VACUUM，horizon 计算可在某些场景跳过它。 |
| `PROC_IN_SAFE_IC` | 当前处于特定安全的 concurrent index build 状态。 |
| `PROC_VACUUM_FOR_WRAPAROUND` | wraparound 防护 vacuum，不应被普通冲突取消策略轻易打断。 |
| `PROC_IN_LOGICAL_DECODING` | logical decoding 以自己的方式管理 xmin。 |
| `PROC_AFFECTS_ALL_HORIZONS` | 该 proc 的 xmin 必须影响所有 database 的 horizon，例如 hot standby feedback。 |

`PROC_VACUUM_STATE_MASK` 表示事务结束时必须清理的一组 flags：

```text
PROC_IN_VACUUM | PROC_IN_SAFE_IC | PROC_VACUUM_FOR_WRAPAROUND
```

这说明 `statusFlags` 不是装饰性状态。它参与 visibility cleanup 的正确性判断，必须和 `xid` / `xmin` 一起发布、一起清除。

## 5. 主流程源码 walkthrough

本节用一条完整时间线串起来：

```text
backend 加入 ProcArray
  -> 开始事务并发布 VXID
  -> 延迟分配并发布 XID
  -> 可能发布 SubXID / xmin / vacuum flags
  -> 其它 backend 构造 snapshot 或计算 horizon
  -> commit / abort 后从 running set 清除事务状态
```

### 5.1 backend 加入 ProcArray: `InitProcessPhase2()` -> `ProcArrayAdd()`

上一节已经看到，普通 backend 在 `InitProcess()` 中领取 `PGPROC` slot。之后 `InitPostgres()` 早期会调用 `InitProcessPhase2()`，再进入：

```c
ProcArrayAdd(MyProc);
```

`ProcArrayAdd()` 在 `src/backend/storage/ipc/procarray.c` 中做四件关键事。

第一，按固定顺序拿锁：

```text
ProcArrayLock exclusive
XidGenLock exclusive
```

为什么两个锁都要拿？`proc.h` 中 `PROC_HDR` 的注释给出原因：

```text
adding/removing ProcArray entry 会移动 dense arrays；
GetNewTransactionId() 在 XidGenLock 下写 dense arrays；
GetSnapshotData() 在 ProcArrayLock 下扫 dense arrays；
因此 add/remove 必须同时排斥两边。
```

第二，把 `pgprocno` 插入 `procArray->pgprocnos[]`。数组按 `PGPROC` 地址顺序排序，目的是让 scan 更有 cache locality。

第三，同步移动 dense arrays：

```text
ProcGlobal->xids[]
ProcGlobal->subxidStates[]
ProcGlobal->statusFlags[]
```

第四，设置当前 proc 的 offset：

```text
proc->pgxactoff = index
ProcGlobal->xids[index] = proc->xid
ProcGlobal->subxidStates[index] = proc->subxidStatus
ProcGlobal->statusFlags[index] = proc->statusFlags
```

这一刻之后，backend 才正式成为事务状态 publication set 的成员。

### 5.2 事务开始: 先发布 VXID，不急着发布 XID

`StartTransaction()` 在 `src/backend/access/transam/xact.c`。它不会立即分配真实 XID，而是先生成 virtual transaction id：

```text
vxid.procNumber = MyProcNumber
vxid.localTransactionId = GetNextLocalTransactionId()
VirtualXactLockTableInsert(vxid)
MyProc->vxid.lxid = vxid.localTransactionId
```

这里有一个很重要的顺序：

```text
先把 VXID lock 放入 lock table
再把 lxid 发布到 MyProc->vxid.lxid
```

这样其它 backend 看到你的 VXID 后，可以用 lock manager 等待它结束。VXID 是“当前 top transaction 的轻量身份”，即使事务没有真实 XID，也能被锁等待、DDL、冲突检测等路径定位。

这解释了一个常见现象：

```sql
BEGIN;
SELECT 1;
```

这个事务已经有 VXID，但通常还没有真实 XID。`txid_current_if_assigned()` 可能返回 NULL，因为它尚未触发 `GetNewTransactionId()`。

### 5.3 真实 XID 发布: `GetNewTransactionId()`

真实 XID 分配在 `src/backend/access/transam/varsup.c` 的 `GetNewTransactionId(bool isSubXact)`。

主流程是：

```text
获取 XidGenLock
  -> 读取并推进 TransamVariables->nextXid
  -> 确保 pg_xact / pg_subtrans 等空间已扩展
  -> 如果是 top-level xact:
       MyProc->xid = xid
       ProcGlobal->xids[MyProc->pgxactoff] = xid
     如果是 subtransaction:
       写入 MyProc->subxids.xids[n]
       write barrier
       更新 MyProc->subxidStatus 和 ProcGlobal->subxidStates[]
       或设置 overflowed
  -> 释放 XidGenLock
```

源码注释明确了关键 correctness rule：

```text
必须在释放 XidGenLock 前把新 XID 存入 shared ProcArray。
```

原因是 `latestCompletedXid` 和 ProcArray running set 之间必须保持一致。否则可能出现：

```text
backend A 分配了较小 XID，但还没发布到 ProcArray
backend B 分配较大 XID 并提交，推进 latestCompletedXid
backend C 构造 snapshot 或计算 horizon
  -> 看到 latestCompletedXid 已越过 A
  -> 又没在 ProcArray 中看到 A
  -> 错误地认为 A 不再运行
```

这会破坏 snapshot 和 cleanup horizon 的基本假设。

所以 `GetNewTransactionId()` 的发布不是统计信息更新，而是 MVCC correctness 边界。

### 5.4 SubXID 发布: cache 完整性靠 barrier 和 overflow flag

当 `isSubXact = true` 时，`GetNewTransactionId()` 不是写 `MyProc->xid`，而是更新 SubXID cache：

```text
nxids = MyProc->subxidStatus.count

如果 nxids < PGPROC_MAX_CACHED_SUBXIDS:
  MyProc->subxids.xids[nxids] = xid
  pg_write_barrier()
  MyProc->subxidStatus.count = ProcGlobal->subxidStates[pgxactoff].count = nxids + 1
否则:
  MyProc->subxidStatus.overflowed = ProcGlobal->subxidStates[pgxactoff].overflowed = true
```

为什么先写数组，再写 count，中间还要 `pg_write_barrier()`？

因为其它 backend 构造 snapshot 时，会先读 `subxidStates[pgxactoff].count`，再根据 count 从 `PGPROC->subxids.xids[]` 拷贝。不能让读者先看到 count 增加，却读到尚未初始化的 SubXID 槽位。

读者侧在 `GetSnapshotData()` 中用：

```text
pg_read_barrier()
memcpy(snapshot->subxip + subcount, proc->subxids.xids, nsubxids * sizeof(TransactionId))
```

与写者侧配对。

当 cache overflow 后，`overflowed = true` 也是共享语义的一部分。它告诉读者：

```text
不要再相信 PGPROC 中的 SubXID cache 是完整集合；
后续判断必须查 pg_subtrans，或者用更保守的 snapshot/subxid 逻辑。
```

### 5.5 snapshot 读取: `GetSnapshotData()` 消费 published state

`GetSnapshotData()` 是 ProcArray 的核心消费者之一。它在 `ProcArrayLock` shared 下执行：

```text
读取 latestCompletedXid
计算 xmax = latestCompletedXid + 1
读取 MyProc->pgxactoff 和自己的 xid
扫描 procArray->pgprocnos[] / ProcGlobal->xids[]
  -> 跳过无 XID backend
  -> 跳过自己
  -> 跳过 xid >= xmax 的事务
  -> 根据 statusFlags 跳过 logical decoding / lazy vacuum 等特殊状态
  -> 把 running xids 收集到 snapshot->xip[]
  -> 按 subxidStatus 拷贝 subxids 或标记 overflow
必要时设置 MyProc->xmin = TransactionXmin = xmin
释放 ProcArrayLock
```

这里有三点尤其重要。

第一，`GetSnapshotData()` 扫描的是 dense arrays：

```text
ProcGlobal->xids[]
ProcGlobal->subxidStates[]
ProcGlobal->statusFlags[]
```

它只有在需要读取 SubXID 数组、databaseId 或 xmin 等 per-proc 信息时，才跳回 `PGPROC`。

第二，读 `xid` 时要“只读一次”。源码使用 `UINT32_ACCESS_ONCE()`，因为 `GetNewTransactionId()` 可以不拿 `ProcArrayLock`，只在 `XidGenLock` 下发布 XID。读者不能假设同一个字段连续读两次一定相同。

第三，`GetSnapshotData()` 可能把当前 snapshot 的 xmin 发布到 `MyProc->xmin`：

```text
if (!TransactionIdIsValid(MyProc->xmin))
    MyProc->xmin = TransactionXmin = xmin;
```

这一步让当前 backend 的 snapshot 需求变成其它 backend 能看到的 cleanup boundary。

### 5.6 horizon 读取: `ComputeXidHorizons()` 同时看 xid、xmin 和 flags

VACUUM、pruning、freeze、catalog cleanup 等路径不只是问“哪些 XID 还 running”，还要问：

```text
哪些旧 tuple version 对任何可能的观察者都不再需要？
```

`ComputeXidHorizons()` 在 `ProcArrayLock` shared 下扫描 ProcArray。它对每个 member 做：

```text
xid = ProcGlobal->xids[index]
xmin = proc->xmin
xmin = older(xmin, xid)

如果 xid/xmin 都无效:
  这个 proc 不影响 horizon
否则:
  更新 oldest_considered_running
  如果 PROC_IN_VACUUM 或 PROC_IN_LOGICAL_DECODING:
    跳过普通 nonremovable horizon
  否则:
    更新 shared_oldest_nonremovable
    如果同库或 affects all horizons:
      更新 data_oldest_nonremovable
```

为什么同时看 `xid` 和 `xmin`？

因为一个 backend 可能已经分配了 XID，但还没建立 snapshot，也就还没设置 `xmin`。如果 horizon 计算只看 `xmin`，它可能错过这个新事务；如果只看 `xid`，又无法表达一个长事务 snapshot 对更老 XID 的保留需求。

为什么要看 `statusFlags`？

因为不同 backend 状态对 cleanup 的含义不同。lazy VACUUM 设置 `PROC_IN_VACUUM` 后，它自己的 snapshot 不应该像普通 user transaction 那样阻止其它 VACUUM 的 tuple cleanup。logical decoding 则通过 slot / decoding state 管理 xmin，不应被普通 per-backend xmin 规则重复解释。

### 5.7 lazy VACUUM 发布 flags: 必须早于 snapshot

`src/backend/commands/vacuum.c` 中 lazy VACUUM 在开始 vacuum relation 后会：

```text
LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE)
MyProc->statusFlags |= PROC_IN_VACUUM
如果是 wraparound vacuum:
  MyProc->statusFlags |= PROC_VACUUM_FOR_WRAPAROUND
ProcGlobal->statusFlags[MyProc->pgxactoff] = MyProc->statusFlags
LWLockRelease(ProcArrayLock)

PushActiveSnapshot(GetTransactionSnapshot())
```

源码注释强调：

```text
PROC_IN_VACUUM 必须在取自己的 snapshot 前设置。
```

否则可能出现短暂状态：

```text
MyProc->xmin 已经对其它 backend 可见
但 statusFlags 还没标记 IN_VACUUM
```

其它 backend 的 horizon 计算可能把这个 lazy VACUUM 当成普通 user transaction，从而得到不符合预期的 cleanup 边界。更严重的是，flags 和 xmin 如果清理顺序不对，horizon 可能看起来倒退。

这再次说明：

```text
raw field 不是语义；
xmin + statusFlags + publication order 才是语义。
```

### 5.8 事务结束: `ProcArrayEndTransaction()` 清除 running state

commit 和 abort 都会在 `xact.c` 中调用：

```c
ProcArrayEndTransaction(MyProc, latestXid);
```

commit 路径中调用点在 `RecordTransactionCommit()` 之后：

```text
RecordTransactionCommit()
  -> WAL / pg_xact 中已经记录事务结论
ProcArrayEndTransaction(MyProc, latestXid)
  -> 让其它 backend 不再把我看作 running
释放资源、锁、本地状态
```

abort 路径也是类似：

```text
RecordTransactionAbort()
  -> pg_xact / WAL 中已经记录 abort
ProcArrayEndTransaction(MyProc, latestXid)
  -> 从 running set 清除
后续 abort cleanup
```

如果事务有真实 XID，`ProcArrayEndTransaction()` 必须获取 `ProcArrayLock` exclusive。它清理：

```text
ProcGlobal->xids[pgxactoff] = InvalidTransactionId
proc->xid = InvalidTransactionId
proc->vxid.lxid = InvalidLocalTransactionId
proc->xmin = InvalidTransactionId
proc->delayChkptFlags = 0
proc->statusFlags 中的 PROC_VACUUM_STATE_MASK
ProcGlobal->statusFlags[pgxactoff]
ProcGlobal->subxidStates[pgxactoff]
proc->subxidStatus
latestCompletedXid
xactCompletionCount
```

如果事务没有真实 XID，它可以不拿 `ProcArrayLock` 去清理 running XID，因为它不会影响 snapshot 的 running-XID 集合，也不会推进 `latestCompletedXid`。但如果它设置过 vacuum state flags，仍需要拿 `ProcArrayLock` exclusive 更新 mirrored `statusFlags[]`。

### 5.9 group XID clearing: high contention commit 的折中

提交很频繁，`ProcArrayEndTransaction()` 又需要 `ProcArrayLock` exclusive。为了降低高并发提交时的锁交接成本，PostgreSQL 有 `ProcArrayGroupClearXid()`。

当 backend 不能立即拿到 `ProcArrayLock` exclusive 时，它会：

```text
把自己加入 ProcGlobal->procArrayGroupFirst 链表
如果自己不是 leader:
  等待 leader 用 semaphore 唤醒
如果自己是 leader:
  获取 ProcArrayLock exclusive
  为链表中的所有 backend 调用 ProcArrayEndTransactionInternal()
  释放锁
  唤醒 followers
```

这不改变语义，只改变执行方式。所有成员仍然在同一个 exclusive lock 持有期间清掉 XID，并推进 completion state。它体现的是一个常见 PostgreSQL 工程取舍：

```text
correctness boundary 不放松；
在 boundary 内批量处理多个 backend，降低 hot path contention。
```

## 6. 生命周期 / ownership / cleanup

把本节状态按生命周期排成一条线：

```text
postmaster startup:
  ProcGlobal dense arrays 和 ProcArrayStruct 被一次性分配。

backend startup:
  InitProcess() 分配 PGPROC，但还没有 ProcArray membership。

database initialization:
  InitProcessPhase2() -> ProcArrayAdd(MyProc)
  backend 成为 ProcArray member。

transaction start:
  StartTransaction() 生成 VXID，发布 MyProc->vxid.lxid。

first XID assignment:
  GetNewTransactionId(false) 发布 MyProc->xid 和 ProcGlobal->xids[]。

subtransaction assignment:
  GetNewTransactionId(true) 发布 subxids 或 overflow flag。

snapshot acquisition:
  GetSnapshotData() 扫描 ProcArray，并可能发布 MyProc->xmin。

special operation:
  lazy VACUUM / logical decoding / walsender 等更新 statusFlags 或 xmin。

transaction end:
  ProcArrayEndTransaction() 清理 XID、VXID、xmin、SubXID、vacuum flags。

backend exit:
  RemoveProcFromArray() 移除 membership；
  ProcKill() 释放 PGPROC slot。
```

ownership 规则可以概括为：

| 状态 | 写者 | 读者 | 保护机制 |
| --- | --- | --- | --- |
| `MyProc->vxid.lxid` | 当前 backend | lock manager、ProcArray 读者、诊断函数 | 发布顺序和 lock table 中 VXID lock。 |
| `MyProc->xid` | 当前 backend | 当前 backend、少量 per-proc 读者 | `XidGenLock` 发布，事务结束时 `ProcArrayLock` 清理。 |
| `ProcGlobal->xids[]` | 当前 backend 或 group clear leader | snapshot / horizon hot path | `XidGenLock` 写入，`ProcArrayLock` 读和清理，atomic fetch/store 约定。 |
| `MyProc->subxids.xids[]` | 当前 backend | snapshot 构造者 | write/read barrier 与 `subxidStatus` 协议。 |
| `subxidStatus` / `ProcGlobal->subxidStates[]` | 当前 backend | snapshot 构造者 | `XidGenLock` 发布，`ProcArrayLock` 清理。 |
| `MyProc->xmin` | 当前 backend | horizon 计算、conflict checks、诊断函数 | `ProcArrayLock` 读上下文，atomic fetch/store 约定。 |
| `statusFlags` / mirrored `statusFlags[]` | 当前 backend 或 cleanup path | snapshot / horizon 计算 | 更新 mirror 时通常持 `ProcArrayLock` exclusive。 |

特别注意：

```text
当前 backend owns 自己的 PGPROC 字段；
但一旦字段成为 ProcArray publication，写入顺序就必须服务其它 backend 的读取协议。
```

本地 owner 并不意味着可以随便写。比如 `SubXID` cache 的 count 不能先增，`PROC_IN_VACUUM` 不能晚于 `xmin` 发布，XID 不能晚于 `XidGenLock` 释放。

## 7. 正确性机制层次

### 7.1 ProcArrayLock: snapshot-taking 与 transaction-end 的互斥

`src/backend/access/transam/README` 中给出一个核心规则：

```text
不允许一个有 XID 的事务在另一个 backend 正在构造 snapshot 时退出 running set。
```

实现方式是：

```text
GetSnapshotData():
  持有 ProcArrayLock shared。

ProcArrayEndTransaction():
  清理 XID 时持有 ProcArrayLock exclusive。
```

这比理论上最小约束更强，但简单可靠。

如果没有这个互斥，可能出现：

```text
B 正在提交
C 正在取 snapshot
A 等 B 的行锁，B 释放锁后 A 提交
C 如果看到 B committed，却没看到 A running
  -> 可能构造出不满足 MVCC 一致性的 snapshot
```

PostgreSQL 选择让 commit/abort 的“退出 running set”与 snapshot scan 严格互斥。

### 7.2 XidGenLock: XID 分配与 XID 发布的顺序

事务开始本身不分配 XID，所以 `StartTransaction()` 不需要进入这个 interlocking。

但一旦 `GetNewTransactionId()` 分配 XID，就必须：

```text
在 XidGenLock 下推进 nextXid；
在释放 XidGenLock 前发布到 ProcArray；
保证所有早于 latestCompletedXid 的 active XID 都能在 ProcArray 中找到。
```

这条规则服务 `ComputeXidHorizons()` 和 snapshot 的共同假设：

```text
如果一个 XID 已经可能影响可见性，它要么还在 ProcArray 的 running set 里，
要么已经有明确完成状态并被 latestCompletedXid 覆盖。
```

### 7.3 Memory barrier: SubXID cache 的局部顺序

SubXID cache 的正确性不是靠大锁完成的。写者在 `GetNewTransactionId(true)` 中：

```text
先写 subxids.xids[n]
pg_write_barrier()
再更新 count
```

读者在 `GetSnapshotData()` 中：

```text
读 count
pg_read_barrier()
memcpy subxids.xids[]
```

这是一个小而关键的 publication protocol。它避免读者看到“count 已经变大，但数组内容还没写好”的中间状态。

### 7.4 Mirrored arrays: 同一语义的双存储

`PGPROC->xid` 与 `ProcGlobal->xids[pgxactoff]` 是同一语义的两个存储位置。

为什么要双存？

| 存储位置 | 优势 |
| --- | --- |
| `PGPROC` 字段 | 当前 backend 查自己状态更便宜，也方便按 proc 定位的路径读取。 |
| dense arrays | 全局扫描更少 indirection，更少 cacheline 浪费，适合 snapshot / horizon hot path。 |

双存储的代价是 coherence contract：

```text
只要 PGPROC 在 ProcArray 中，mirrored values 必须保持一致；
add/remove 会移动 dense arrays，pgxactoff 只在相关锁下安全。
```

### 7.5 WAL / pg_xact 与 ProcArray 清理顺序

commit 路径中，`ProcArrayEndTransaction()` 必须在 `RecordTransactionCommit()` 之后。

顺序是：

```text
先把事务结论写入 WAL / pg_xact
再从 ProcArray running set 清除
```

否则其它 backend 可能不再看到这个事务 running，但又无法从 commit status 中确认它已完成。

abort 路径同理，先记录 abort，再清理 ProcArray running state。

### 7.6 `xactCompletionCount`: snapshot reuse 的变更信号

`ProcArrayEndTransactionInternal()` 在清理 XID 后会递增：

```text
TransamVariables->xactCompletionCount++;
```

`GetSnapshotData()` 的 snapshot reuse 逻辑会用它判断是否可以复用之前的 snapshot 结果。

这说明“事务结束”不只是把 `xids[]` 置空。它还要发布一个全局版本变化信号：

```text
running set 变化了；
依赖 running set 的缓存判断必须失效或重新验证。
```

## 8. 错误路径 / 异常路径 / fallback

### 8.1 ProcArray 已满

`ProcArrayAdd()` 中如果：

```text
arrayP->numProcs >= arrayP->maxProcs
```

会报：

```text
FATAL: sorry, too many clients already
```

源码注释说这通常不该发生，因为 PGPROC slot 数量已经是更早的限制。也就是说，正常情况下会先在 `InitProcess()` 分配 PGPROC slot 时失败。

但 `ProcArrayAdd()` 仍保留防线，因为 ProcArray membership 是独立资源边界。

### 8.2 无 XID transaction 的结束路径

如果 `latestXid` 无效，`ProcArrayEndTransaction()` 不需要用 exclusive `ProcArrayLock` 清理 `xids[]`：

```text
read-only / XID-less transaction:
  不影响 running-XID set
  不推进 latestCompletedXid
  可以清理 vxid.lxid / xmin / delayChkptFlags
```

但如果它设置过 vacuum-related flags：

```text
proc->statusFlags & PROC_VACUUM_STATE_MASK
```

仍要获取 `ProcArrayLock` exclusive，更新 `ProcGlobal->statusFlags[]`。

这个分支说明：

```text
是否需要 ProcArrayLock，不取决于“事务是否结束”，
而取决于它要改变的 shared publication 是否会影响其它 backend 的 snapshot/horizon 判断。
```

### 8.3 SubXID cache overflow

超过 `PGPROC_MAX_CACHED_SUBXIDS` 后，PostgreSQL 不再试图把所有 SubXID 都塞入 `PGPROC`。

它设置：

```text
MyProc->subxidStatus.overflowed = true
ProcGlobal->subxidStates[pgxactoff].overflowed = true
```

fallback 是读者回到 `pg_subtrans` 或使用保守判断。

这不是“性能退化而已”。如果没有 overflow flag，snapshot 读者会误以为 PGPROC cache 是完整 SubXID 集合，从而可能把某些仍 running 的 subtransaction 当成已完成。

### 8.4 prepared transaction: live backend 退出，事务状态不能退出

two-phase commit 会制造一个特别的边界：

```text
backend 可以结束当前本地事务上下文；
但 prepared transaction 的 XID 仍必须被其它 backend 看作 running / unresolved。
```

`twophase.c` 会使用 prepared transaction 对应的 dummy `PGPROC`，并调用 `ProcArrayAdd()` 把它加入 ProcArray。当前 backend 后续用 `ProcArrayClearTransaction()` 清掉自己的 PGPROC 中的事务字段，但 prepared xact 的 PGPROC 继续代表这个事务。

这解释了为什么 ProcArray membership 不是“进程列表”的简单镜像。某些 ProcArray member 可以代表 prepared xact，而不是一个正在执行 SQL 的 live backend。

### 8.5 logical decoding 与 replication slot xmin

logical decoding 和 replication slot 会把 cleanup horizon 的一部分从 backend-local snapshot 扩展成持久复制需求。

典型路径包括：

```text
logical decoding:
  设置 PROC_IN_LOGICAL_DECODING；
  以 decoding / slot 机制管理 xmin。

replication slot:
  ProcArraySetReplicationSlotXmin(xmin, catalog_xmin, ...)
  更新 procArray->replication_slot_xmin
  更新 procArray->replication_slot_catalog_xmin
```

因此 `ComputeXidHorizons()` 不能只扫描 backend 的 `xmin`。它还必须考虑 replication slot 约束。

这也是为什么“谁需要旧数据”不是一个单纯 session 问题：

```text
session snapshot 需要旧 tuple；
logical decoding 需要旧 catalog / WAL interpretation boundary；
replication slot 可以让这个需求在 session 之外持续存在。
```

## 9. 成本、资源与跨模块传播

### 9.1 ProcArray scan 成本随 backend 数增长

`GetSnapshotData()` 和 `ComputeXidHorizons()` 都要扫描 ProcArray members。

大致成本是：

```text
O(number of ProcArray members)
```

这解释了为什么 `max_connections` 不是单纯内存上限。更多 backend 意味着：

```text
更多 PGPROC slots
更多 ProcArray entries
更长 snapshot scan
更大的 dense arrays
更高 ProcArrayLock contention 风险
更多 cacheline traffic
```

PostgreSQL 用 dense arrays、snapshot reuse、group XID clearing 等手段降低成本，但没有改变“全局 snapshot 需要扫描 shared running set”的基本事实。

### 9.2 XID-less transactions 降低 publication 压力

事务开始只发布 VXID，不立即发布 XID。这让很多只读事务避免：

```text
分配 XID
扩展 pg_xact / pg_subtrans
污染 ProcGlobal->xids[]
推进 xid horizon
增加 wraparound 压力
```

这也是为什么只读 workload 下观察 `txid_current_if_assigned()` 很有价值。它能帮助区分：

```text
有 transaction boundary
  vs
真正消耗 XID、影响 running-XID set
```

### 9.3 SubXID overflow 影响 snapshot 和 visibility 判断

大量 savepoint 或深层 PL/pgSQL exception block 可能产生许多 SubXID。

一旦 SubXID cache overflow：

```text
snapshot 不能再完整保存所有 subxids；
visibility 判断可能需要查 pg_subtrans；
某些场景下 snapshot/subtransaction 相关路径会更保守或更慢。
```

所以 SubXID overflow 常见于：

```text
大量 SAVEPOINT / ROLLBACK TO SAVEPOINT
深层 exception handler
批处理框架为每行建立子事务
```

它表面上是 SQL pattern，底层会传播到 ProcArray publication 和 visibility 判断。

### 9.4 VACUUM horizon 受多个模块共同决定

VACUUM 能删除旧 tuple，不只取决于当前表。

它会被这些状态影响：

```text
其它 backend 的 running XID
其它 backend 的 MyProc->xmin
lazy VACUUM / logical decoding flags
databaseId 是否相同
replication slot xmin / catalog_xmin
hot standby feedback
prepared transaction
```

因此 `VACUUM` 看起来“清不掉死元组”，不一定是 vacuum 本身慢。可能是某个 backend 或 slot 通过 ProcArray / replication slot horizon 发布了旧 xmin。

### 9.5 跨模块边界

| 模块 | 与 ProcArray publication 的关系 |
| --- | --- |
| `xact.c` | 事务生命周期主控，开始发布 VXID，结束调用 `ProcArrayEndTransaction()`。 |
| `varsup.c` | XID / SubXID 分配者，负责把 XID 在正确锁序下发布到 ProcArray。 |
| `procarray.c` | membership、snapshot、horizon、transaction-end clearing 的核心实现。 |
| `vacuum.c` | 设置 `PROC_IN_VACUUM` 等 flags，影响 horizon 解释。 |
| `twophase.c` | prepared transaction 用 dummy PGPROC 继续发布 unresolved XID。 |
| `slot.c` / logical decoding | 把 replication / decoding 需求转换成 xmin horizon 约束。 |
| lock manager | VXID lock 和 transactionid lock 让其它 backend 能等待事务结束。 |
| stats / SQL functions | `pg_locks`、`pg_stat_activity`、`txid_current_if_assigned()` 等提供间接观测入口。 |

## 10. 观测与诊断入口

### 10.1 区分 VXID 和 XID

在一个 session 中：

```sql
BEGIN;
SELECT txid_current_if_assigned();
```

通常会返回 NULL，因为事务还没有真实 XID。

再执行：

```sql
CREATE TEMP TABLE procarray_demo(i int);
INSERT INTO procarray_demo VALUES (1);
SELECT txid_current_if_assigned();
```

此时通常会看到真实 XID。对应源码变化是：

```text
写操作触发 GetNewTransactionId(false)
  -> MyProc->xid 被设置
  -> ProcGlobal->xids[MyProc->pgxactoff] 被设置
```

这个实验帮助建立：

```text
transaction start != XID assignment
```

### 10.2 观察长事务 xmin 对 VACUUM 的影响

session A：

```sql
BEGIN;
SELECT * FROM some_table LIMIT 1;
-- 保持事务打开
```

session B：

```sql
UPDATE some_table SET ... WHERE ...;
VACUUM VERBOSE some_table;
```

如果 session A 的 snapshot 仍然需要旧版本，VACUUM 可能无法移除某些 dead tuples。

源码解释路径：

```text
session A GetSnapshotData()
  -> MyProc->xmin = TransactionXmin
session B VACUUM
  -> ComputeXidHorizons()
  -> 扫描 A 的 MyProc->xmin
  -> 得到较老 oldest_nonremovable
  -> 不能删除仍可能被 A snapshot 看到的 tuple version
```

### 10.3 用 gdb 看 ProcArray publication

在 backend 上设置断点：

```gdb
break ProcArrayAdd
break GetNewTransactionId
break GetSnapshotData
break ProcArrayEndTransaction
```

观察：

```gdb
p MyProcNumber
p MyProc
p MyProc->pgxactoff
p MyProc->vxid
p MyProc->xid
p ProcGlobal->xids[MyProc->pgxactoff]
p MyProc->xmin
p MyProc->subxidStatus
p ProcGlobal->subxidStates[MyProc->pgxactoff]
p MyProc->statusFlags
p ProcGlobal->statusFlags[MyProc->pgxactoff]
```

预期时间线：

```text
ProcArrayAdd:
  pgxactoff 开始有效。

StartTransaction 之后:
  vxid.lxid 有效，xid 仍可能无效。

GetNewTransactionId(false):
  MyProc->xid 和 ProcGlobal->xids[] 同步变为新 XID。

GetSnapshotData:
  MyProc->xmin 可能从 Invalid 变为 snapshot xmin。

ProcArrayEndTransaction:
  xid / vxid.lxid / xmin / subxidStatus / vacuum flags 被清理。
```

### 10.4 观察 group XID clearing

高并发短事务提交时，可以在 gdb 或 perf 中关注：

```text
ProcArrayEndTransaction
ProcArrayGroupClearXid
WAIT_EVENT_PROCARRAY_GROUP_UPDATE
ProcArrayLock
```

如果看到 backend 等待 `WAIT_EVENT_PROCARRAY_GROUP_UPDATE`，说明它没有自己拿 `ProcArrayLock` 清理 XID，而是作为 follower 等 leader 批量清理。

这说明 commit hot path 上的瓶颈不是 WAL flush 才可能出现。ProcArray running set 的退出也可能成为同步点。

### 10.5 从 SQL 侧间接定位旧 xmin

常用观察入口：

```sql
SELECT pid, backend_xid, backend_xmin, state, query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL OR backend_xmin IS NOT NULL
ORDER BY backend_xmin NULLS LAST;
```

还可以看 replication slot：

```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin, restart_lsn
FROM pg_replication_slots;
```

解释时要小心：

```text
pg_stat_activity.backend_xmin 是诊断投影，不是 ProcArray 的完整状态；
真正的 cleanup horizon 还要考虑 databaseId、statusFlags、replication slots、prepared xacts 和 recovery state。
```

## 11. 常见误区

### 误区 1: “backend 在 ProcArray 中就一定有 XID”

不对。backend 加入 ProcArray 是 membership；XID 是事务运行中按需发布的状态。

一个 backend 可以：

```text
有 MyProc
有 ProcArray membership
有 VXID
但没有真实 XID
```

### 误区 2: “PGPROC->xid 和 ProcGlobal->xids[] 是重复字段，可以随便读一个”

不对。它们服务不同访问模式，并且有锁和 offset 约束。

`ProcGlobal->xids[]` 适合全局扫描；`PGPROC->xid` 适合按 proc 读取或当前 backend 自查。`pgxactoff` 可能因 add/remove 改变，必须在 `ProcArrayLock` 或 `XidGenLock` 保护下使用。

### 误区 3: “xmin 是我的事务 ID”

不对。`xid` 是当前事务自己的 top-level XID；`xmin` 是当前 backend 的 snapshot 对 cleanup horizon 的保留边界。

一个 backend 可能：

```text
有 xmin 但没有 xid:
  只读事务拿了 snapshot。

有 xid 但暂时没有 xmin:
  写事务刚分配 XID，但还没建立需要发布的 snapshot xmin。
```

### 误区 4: “VACUUM 清不掉数据一定是 VACUUM 慢”

不一定。可能是：

```text
长事务 MyProc->xmin 很老
prepared transaction 保留 XID
replication slot xmin / catalog_xmin 很老
logical decoding 保留 catalog horizon
hot standby feedback 让 walsender 发布跨库 horizon
```

这些都通过 ProcArray 或 slot horizon 传播到 cleanup decision。

### 误区 5: “SubXID overflow 只是 debug 细节”

不对。SubXID overflow 改变 snapshot / visibility 的读取方式。它告诉读者 PGPROC cache 不完整，必须使用 fallback。

大量 savepoint workload 中，SubXID overflow 可能成为真实性能问题和诊断线索。

### 误区 6: “statusFlags 只是标注 backend 类型”

不对。`statusFlags` 参与 horizon 解释。尤其是：

```text
PROC_IN_VACUUM
PROC_IN_LOGICAL_DECODING
PROC_AFFECTS_ALL_HORIZONS
```

这些 flags 会改变一个 backend 的 `xmin` 是否、如何影响不同 cleanup horizon。

### 误区 7: “事务结束后再慢慢清 ProcArray 也可以”

不对。commit / abort 的可见结论与 ProcArray running set 清理有严格顺序：

```text
先记录 commit/abort
再从 ProcArray running set 退出
再释放锁和本地资源
```

否则其它 backend 可能看到一个既不 running、又没有可靠完成状态的事务。

## 12. 课堂实验

### 实验 1: 观察 VXID 先于 XID

目标：验证事务开始不等于真实 XID 发布。

步骤：

```sql
BEGIN;
SELECT txid_current_if_assigned();
```

预期：返回 NULL。

然后执行：

```sql
CREATE TEMP TABLE procarray_xid_demo(i int);
INSERT INTO procarray_xid_demo VALUES (1);
SELECT txid_current_if_assigned();
ROLLBACK;
```

预期：第二次能看到 XID。

源码回扣：

```text
StartTransaction()
  -> 发布 VXID

GetNewTransactionId(false)
  -> 发布 MyProc->xid / ProcGlobal->xids[]
```

### 实验 2: gdb 跟踪 XID 发布顺序

目标：确认 XID 在 `XidGenLock` 持有期间发布到 shared state。

步骤：

```gdb
break GetNewTransactionId
commands
  silent
  printf "GetNewTransactionId: MyProcNumber=%d pgxactoff=%d xid=%u mirrored=%u\n", MyProcNumber, MyProc->pgxactoff, MyProc->xid, ProcGlobal->xids[MyProc->pgxactoff]
  continue
end
```

在 SQL 中执行触发写入的语句。

可进一步在 `GetNewTransactionId()` 的写入位置单步，观察：

```text
MyProc->xid = xid
ProcGlobal->xids[MyProc->pgxactoff] = xid
```

源码回扣：

```text
释放 XidGenLock 前必须完成 shared publication。
```

### 实验 3: 观察 snapshot xmin 对 VACUUM 的影响

准备一张测试表：

```sql
CREATE TABLE procarray_vacuum_demo(id int primary key, payload text);
INSERT INTO procarray_vacuum_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 10000) g;
```

session A：

```sql
BEGIN;
SELECT count(*) FROM procarray_vacuum_demo;
```

session B：

```sql
UPDATE procarray_vacuum_demo SET payload = repeat('y', 100);
VACUUM VERBOSE procarray_vacuum_demo;
```

观察 `VACUUM VERBOSE` 输出，以及：

```sql
SELECT pid, backend_xid, backend_xmin, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

源码回扣：

```text
session A 的 GetSnapshotData() 发布 MyProc->xmin；
session B 的 VACUUM 通过 ComputeXidHorizons() 看到这个 xmin；
旧 tuple version 的 removable boundary 被压低。
```

### 实验 4: 制造 SubXID overflow

目标：观察大量 savepoint 触发 SubXID overflow 的路径。

可以在一个事务中执行大量 savepoint：

```sql
BEGIN;
SAVEPOINT s1;
SAVEPOINT s2;
-- 用脚本生成超过 PGPROC_MAX_CACHED_SUBXIDS 的 SAVEPOINT / 写入
ROLLBACK;
```

更直接的方式是在 gdb 中观察：

```gdb
break GetNewTransactionId if isSubXact
commands
  silent
  printf "subxid count=%u overflowed=%d\n", MyProc->subxidStatus.count, MyProc->subxidStatus.overflowed
  continue
end
```

预期：

```text
count 最多到 PGPROC_MAX_CACHED_SUBXIDS；
之后 overflowed 变为 true。
```

源码回扣：

```text
PGPROC cache 不完整时，用 overflow flag 通知 snapshot/visibility 读者走 fallback。
```

### 实验 5: 观察 lazy VACUUM flags

目标：理解 `PROC_IN_VACUUM` 与 `xmin` 的发布顺序。

在 gdb 中：

```gdb
break vacuum_rel
break GetSnapshotData
commands 1
  silent
  printf "vacuum_rel: statusFlags=%u mirrored=%u pgxactoff=%d\n", MyProc->statusFlags, ProcGlobal->statusFlags[MyProc->pgxactoff], MyProc->pgxactoff
  continue
end
```

执行：

```sql
VACUUM procarray_vacuum_demo;
```

关注：

```text
vacuum.c 在 PushActiveSnapshot(GetTransactionSnapshot()) 之前设置 PROC_IN_VACUUM；
事务结束时 ProcArrayEndTransaction() 清理 PROC_VACUUM_STATE_MASK。
```

## 13. 讨论题

1. 为什么 PostgreSQL 不在 `StartTransaction()` 时立即分配 XID？这对只读事务、wraparound、ProcArray scan 成本分别有什么影响？

2. 如果 `GetNewTransactionId()` 在释放 `XidGenLock` 后才写 `ProcGlobal->xids[]`，可能破坏哪些 snapshot / horizon 假设？

3. 为什么 `ProcArrayEndTransaction()` 要在释放 heavyweight locks 之前执行？如果先释放锁，再清 ProcArray，会出现怎样的可见性窗口？

4. `PGPROC->xid` 和 `ProcGlobal->xids[]` 双存储带来了哪些性能收益？它又引入了哪些 coherence 约束？

5. 为什么 `xmin` 不能只保存在当前 backend 的 local snapshot 结构里？VACUUM 和 pruning 如何依赖它？

6. lazy VACUUM 为什么要在获取 snapshot 前设置 `PROC_IN_VACUUM`？如果顺序反了，horizon 计算可能如何误解它？

7. prepared transaction 为什么需要 dummy PGPROC 继续留在 ProcArray？这说明 ProcArray membership 和 OS process lifecycle 的关系是什么？

8. SubXID cache overflow 后，为什么设置一个 flag 就足够维持 correctness？代价转移到了哪里？

9. 在高并发短事务 workload 中，`ProcArrayGroupClearXid()` 降低了哪类成本？它为什么不能完全消除 ProcArrayLock 作为同步点？

10. 诊断“VACUUM 无法回收 dead tuples”时，为什么只看当前数据库的 active query 不够？

## 14. 本节小结

本节的主线是：

```text
backend-local transaction state
  -> 通过 PGPROC / ProcGlobal dense arrays 发布成 shared fact
  -> 被 snapshot / horizon / vacuum / replication / 2PC 消费
  -> 在 commit / abort / exit 时按严格顺序清除
```

核心结论：

```text
ProcArray 是 PostgreSQL MVCC 的事务状态发布集合。
```

它不是简单的 backend 列表。一个 backend 进入 ProcArray 后，承诺按统一协议发布：

```text
VXID:
  让其它 backend 能等待一个尚未分配真实 XID 的事务。

XID:
  让 snapshot 知道哪些 top-level transactions 正在运行。

SubXID:
  让 snapshot 在子事务存在时仍能判断 running set，overflow 时回退到 pg_subtrans。

xmin:
  让 cleanup horizon 知道哪些旧 tuple version 仍可能被 snapshot 需要。

statusFlags:
  让 horizon 计算正确解释 lazy VACUUM、logical decoding、cross-database feedback 等特殊状态。
```

本节沉淀出的可迁移规律是：

```text
在多进程数据库内核里，一个状态是否应该 shared，不取决于“谁产生它”，
而取决于“谁必须在什么 ordering 下观察它，才能做出正确决定”。
```

PostgreSQL 没有把完整事务对象搬进 shared memory，而是只发布跨 backend correctness 必需的最小状态，并用锁、dense arrays、barrier、flags 和 fallback 拼成可扫描、可清理、可诊断的协议。

下一节将继续沿着这个 publication set 的 hot path，讨论 `GetSnapshotData()` 为什么会成为 ProcArray 扩展性瓶颈，以及 dense arrays、snapshot reuse、group clearing 等优化分别缓解了哪一段成本。
