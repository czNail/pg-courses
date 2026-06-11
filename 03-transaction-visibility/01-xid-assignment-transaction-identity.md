# PostgreSQL XID 分配与事务身份边界

## 课程定位

前置知识：已经理解 backend 进程、`PGPROC`、`ProcArray`、基础 MVCC、heap tuple header 的 `xmin` / `xmax`，以及 `MemoryContext` 和 `ResourceOwner` 分别负责本地内存与外部资源 cleanup。

本节唯一主问题：

```text
一个已经开始执行 SQL 的事务，什么时候才真正拥有会写入 tuple header、进入 ProcArray running set、并参与 MVCC 可见性判断的 XID？
```

本节围绕的核心矛盾：

```text
事务从一开始就需要可等待、可取消、可被 lock manager 识别的身份；
但只有真正可能改变 MVCC 可见性的事务，才应该消耗全局 XID、影响 ProcArray hot path、推进 wraparound 压力，并让其它 backend 把它当作 running XID。
```

学完后应能判断：

- 为什么 `BEGIN` 后 `pg_current_xact_id_if_assigned()` 可以返回 `NULL`。
- 为什么 `pg_current_xact_id()` 是会改变状态的观测函数。
- 为什么只读事务可以从开始到结束都没有真实 XID。
- 为什么 `INSERT` / `UPDATE` / `DELETE` 必须触发 XID 分配。
- 为什么 `GetNewTransactionId()` 不只是 `nextXid++`。
- 为什么 commit / abort 后必须先让事务结果可解释，再从 ProcArray 清除 running 身份。
- 为什么 `virtualxid`、`backend_xid`、`transactionid` lock 表示不同身份边界。

本课基于本地源码：

```text
/home/highgo/postgres
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```

本节不展开 `pg_xact` 状态机、完整 snapshot 字段语义、heap visibility 分支和 SubXID overflow 的全部成本模型；这些分别进入后续课程。

## 1. 本节在总主线中的位置

事务与可见性这一组课的目标是解释一个 tuple version 为什么可见、不可见、可锁定、可更新或可回收。本节站在最前面，先回答 tuple header 中 `xmin` / `xmax` 的事务身份来自哪里。

一个 SQL 事务开始后，backend 内部已经有事务栈、command id、timestamp、memory context、resource owner 和 VXID。但这些状态不都等价于真实 XID。

其它 backend 不能直接访问当前 backend 的 C 栈和 `CurrentTransactionState` 指针。因此 PostgreSQL 必须定义哪些状态要发布到 shared memory，什么时候发布，谁能读取，事务结束或 ERROR 时如何撤销发布。

最小只读故事：

```sql
BEGIN;
SELECT 1;
COMMIT;
```

这个事务有生命周期。它有 `VirtualTransactionId`。

它可以被某些等待路径等待。它可以出现在 `pg_locks` 的 `virtualxid` 相关信息中。

但它通常不会分配真实 XID。它不会把 XID 写进 heap tuple header。

最小写入故事：

```sql
BEGIN;
INSERT INTO t VALUES (1);
COMMIT;
```

这次 `heap_insert()` 需要把当前事务 ID 写入新 tuple 的 `xmin`。于是调用链必须把事务从“只有 VXID”升级为“拥有真实 XID”。

从此以后，snapshot、visibility、CLOG、SubTrans、transactionid lock、WAL replay 和 VACUUM horizon 都可能依赖这个 XID。本节要建立的模型是：

```text
事务已开始
  不等于 已分配真实 XID
  不等于 已写 tuple header
  不等于 已提交或回滚
  不等于 已从 ProcArray running set 消失
```

这些是不同时间点。很多 MVCC bug 和诊断误判，都来自把这些时间点合并成一个“事务状态”。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
StartTransaction() 为顶层事务创建 VXID 并发布到 MyProc；
事务保持无 XID，直到某条路径调用强制分配 accessor；
AssignTransactionId() 维护父子事务顺序、SubTrans 父链、XID lock 和 WAL assignment 周边状态；
GetNewTransactionId() 在 XidGenLock 下取号、准备 CLOG/SUBTRANS、推进 nextXid，并把 XID 发布到 MyProc 和 ProcGlobal；
写 heap tuple 时把 XID 写入 tuple header；
commit/abort 先写事务结果，再调用 ProcArrayEndTransaction() 清除 running 身份。
```

PostgreSQL 拆出三层事务身份：

| 层次 | 状态 | 生命周期 | 是否参与 MVCC tuple visibility |
| --- | --- | --- | --- |
| backend identity | `MyProc`、`MyProcNumber` | backend 进程进入 shared memory 到退出 | 否 |
| virtual transaction identity | `procNumber + localTransactionId` | 顶层事务开始到结束 | 否 |
| real transaction identity | `FullTransactionId` / `TransactionId` | XID 分配后到结果被清理或冻结解释 | 是 |

这三个层次解决不同问题。`PGPROC` 让其它进程能定位当前 backend。

VXID 让系统能等待一个事务生命周期，即使它没有真实 XID。真实 XID 让 tuple header、ProcArray、`pg_xact`、`pg_subtrans` 和 snapshot 有共同语言。

关键边界是：

```text
XID assignment 是 publication boundary。
在它之前，事务身份主要服务当前 backend 和 lock wait。
在它之后，事务身份成为 MVCC、CLOG、SubTrans、ProcArray、WAL、VACUUM horizon 共同依赖的共享事实。
```

因此本节不把 XID 分配看成局部 helper。它是 transaction identity 从本地生命周期进入全局可见性世界的瞬间。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xact.c` | `StartTransaction()` 创建 VXID；`GetTopTransactionId()` / `GetCurrentTransactionId()` 触发分配；`AssignTransactionId()` 维护父子事务和周边状态；commit/abort 清理。 |
| 2 | `src/backend/access/transam/varsup.c` | `GetNewTransactionId()` 在 `XidGenLock` 下分配 `FullTransactionId`、扩展状态存储并发布到 `PGPROC` / `ProcGlobal`。 |
| 3 | `src/include/access/xact.h` | 对外事务 ID accessor，尤其是强制分配与 `IfAny` accessor 的语义差异。 |
| 4 | `src/include/access/transam.h` | `TransactionId` 特殊值、`FullTransactionId`、`TransamVariablesData`、`nextXid`、`latestCompletedXid`。 |
| 5 | `src/include/storage/proc.h` | `PGPROC->vxid`、`xid`、`xmin`、`subxidStatus`、`subxids`，以及 `PROC_HDR` dense mirror arrays。 |
| 6 | `src/backend/storage/lmgr/proc.c` | backend 初始化 `MyProc->xid` / `vxid`，进入 ProcArray，退出时兜底移除。 |
| 7 | `src/backend/storage/ipc/procarray.c` | `ProcArrayAdd()`、`ProcArrayEndTransaction()`、`TransactionIdIsInProgress()`、`XidCacheRemoveRunningXids()`。 |
| 8 | `src/backend/access/transam/transam.c` | `TransactionIdDidCommit()` / `TransactionIdDidAbort()` 通过 `pg_xact` 和 `pg_subtrans` 解释 XID 结果。 |

辅助阅读：

| 文件 | 用途 |
| --- | --- |
| `src/backend/access/heap/heapam.c` | `heap_insert()` 用 `GetCurrentTransactionId()` 证明写 tuple 前必须拿 XID。 |
| `src/backend/utils/adt/xid8funcs.c` | `pg_current_xact_id()` 与 `pg_current_xact_id_if_assigned()` 的观测边界。 |
| `src/include/storage/lock.h`、`src/backend/storage/lmgr/lock.c` | VXID lock 与 transactionid lock 的等待语义。 |

推荐跟读顺序：

```text
xact.c: StartTransaction()
  -> xact.c: GetTopTransactionId() / GetCurrentTransactionId()
  -> xact.c: AssignTransactionId()
  -> varsup.c: GetNewTransactionId()
  -> heapam.c: heap_insert()
  -> xact.c: RecordTransactionCommit() / RecordTransactionAbort()
  -> procarray.c: ProcArrayEndTransaction()
  -> procarray.c: TransactionIdIsInProgress()
  -> transam.c: TransactionIdDidCommit() / TransactionIdDidAbort()
```

阅读时关注状态跨边界，而不是背函数名。状态流向是：

```text
backend-local transaction state
  -> MyProc fields
  -> ProcGlobal dense arrays
  -> tuple header
  -> WAL / pg_xact / pg_subtrans
  -> snapshot / visibility / cleanup consumers
```

## 4. 关键数据结构与状态

### 4.1 `TransactionStateData`

`TransactionStateData` 定义在 `xact.c` 内部。它表达当前 backend 内部事务栈，不是 shared-memory public contract。

本节关注字段：

| 字段 | 语义 |
| --- | --- |
| `fullTransactionId` | 当前事务层级的 `FullTransactionId`，未分配时 invalid。 |
| `subTransactionId` | backend-local 的子事务层级 ID。 |
| `state` / `blockState` | 低层与高层事务状态。 |
| `curTransactionContext` | 当前事务层级内存生命周期。 |
| `curTransactionOwner` | 当前事务层级资源 owner。 |
| `childXids` / `nChildXids` | 已 subcommit 的 child XID，顶层结束时统一写结果。 |
| `didLogXid` | 当前事务 XID 是否已出现在 WAL 记录中。 |
| `topXidLogged` | 子事务场景中 top XID 是否已按 logical decoding 要求记录。 |
| `parent` | 子事务栈回到父事务的指针。 |

它回答的是当前 backend 内部事务生命周期问题。它不能让其它 backend 判断一个 XID 是否 running。

其它 backend 需要读 `PGPROC`、`ProcGlobal`、`pg_xact` 和 `pg_subtrans`。

### 4.2 `TransactionId` 与 `FullTransactionId`

`transam.h` 中的特殊 XID：

| 值 | 名称 | 语义 |
| --- | --- | --- |
| `0` | `InvalidTransactionId` | 未分配或无效。 |
| `1` | `BootstrapTransactionId` | bootstrap 特殊事务。 |
| `2` | `FrozenTransactionId` | frozen tuple 使用的特殊表示。 |
| `3` | `FirstNormalTransactionId` | 第一个普通事务 ID。 |

tuple header 中保存 32 bit `TransactionId`。运行时全局分配使用 `FullTransactionId`：

```text
FullTransactionId = epoch + 32 bit xid
```

`XidFromFullTransactionId()` 取低 32 bit。`FullTransactionIdAdvance()` 负责跨过低 32 bit 上看起来像 special XID 的值。

要区分：

```text
tuple header 持久保存的是 32 bit XID；
内存中全局顺序和 wraparound 判断需要 FullTransactionId；
从 full xid 回到 32 bit xid 必须依赖 horizon 和 wraparound 保护。
```

### 4.3 `TransamVariablesData`

`TransamVariablesData` 在 shared memory 中，`varsup.c` 通过 `TransamVariables` 指针访问。本节关注字段：

| 字段 | 保护 | 语义 |
| --- | --- | --- |
| `nextXid` | `XidGenLock` | 下一个要分配的 `FullTransactionId`。 |
| `oldestXid` | `XidGenLock` | cluster-wide 最老 `datfrozenxid`。 |
| `xidVacLimit` | `XidGenLock` | 到达后开始推动 autovacuum。 |
| `xidWarnLimit` | `XidGenLock` | 到达后发出 wraparound warning。 |
| `xidStopLimit` | `XidGenLock` | 到达后拒绝继续分配普通 XID。 |
| `xidWrapLimit` | `XidGenLock` | wraparound 灾难边界，用于保护和报错。 |
| `latestCompletedXid` | `ProcArrayLock` | 最新完成的事务 XID。 |
| `xactCompletionCount` | `ProcArrayLock` | 带 XID 事务完成计数，用于 snapshot reuse。 |
| `oldestClogXid` | `XactTruncationLock` | 仍可安全查询 CLOG 的最老 XID。 |

`nextXid` 只是 raw field。真正语义是：

```text
nextXid + XidGenLock + CLOG/SUBTRANS readiness + ProcArray publication + wraparound limits
```

这就是为什么 `GetNewTransactionId()` 不能被理解为简单计数器。

### 4.4 `PGPROC->vxid`

`PGPROC` 中的 VXID 拆成：

```text
vxid.procNumber
vxid.lxid
```

`procNumber` 是当前 `PGPROC` 在 `ProcGlobal->allProcs[]` 中的编号。`lxid` 是当前顶层事务的 local transaction id。

`lock.h` 中的 `VirtualTransactionId` 也是这两个部分。它的语义：

- 短期唯一。
- 不写入磁盘。
- 不参与 tuple visibility。
- 可被 lock manager 用来等待事务生命周期结束。
- 可覆盖无真实 XID 的事务。

`StartTransaction()` 会先 `VirtualXactLockTableInsert(vxid)`，再把 `MyProc->vxid.lxid` 设为有效值。VXID lock 可以证明：

```text
一个事务可以被等待，但仍没有真实 XID。
```

### 4.5 `PGPROC->xid` 与 `ProcGlobal->xids[]`

`PGPROC->xid` 表示当前 top-level transaction 的真实 XID。没有分配时为 `InvalidTransactionId`。

它镜像到：

```text
ProcGlobal->xids[MyProc->pgxactoff]
```

两份状态服务不同访问模式。`PGPROC` 适合当前 backend 自己读写，或按 proc 定位的少量读者。

`ProcGlobal->xids[]` 是 dense array，适合 snapshot / horizon hot path 扫描很多 backend。`proc.h` 中 `PROC_HDR` 注释强调：

- dense arrays 只包含已经进入 ProcArray 的 `PGPROC`。
- `pgxactoff` 会在 `ProcArrayAdd()` / `ProcArrayRemove()` 后变化。
- 只有持有 `ProcArrayLock` 或 `XidGenLock` 时，才能安全用 `pgxactoff` 访问 dense arrays。

`GetNewTransactionId()` 为顶层事务发布 XID 时写：

```text
MyProc->xid = xid
ProcGlobal->xids[MyProc->pgxactoff] = xid
```

写入发生在 `XidGenLock` 仍持有期间。这保证一个已分配且可能写入 tuple header 的 XID，不会被 concurrent snapshot 漏掉。

### 4.6 `subxids` 与 `pg_subtrans`

`PGPROC` 会缓存最多 `PGPROC_MAX_CACHED_SUBXIDS` 个 SubXID。本基线源码中值为 `64`。

相关字段：

| 字段 | 语义 |
| --- | --- |
| `PGPROC->subxids.xids[]` | 当前 top transaction 下 cached SubXID。 |
| `PGPROC->subxidStatus.count` | 当前缓存数量。 |
| `PGPROC->subxidStatus.overflowed` | 是否已有 SubXID 未放进 cache。 |
| `ProcGlobal->subxidStates[]` | dense mirror，用于快速扫描 count / overflow。 |

未 overflow 时，读者可以相信：

```text
某个 XID 既不是任何 top XID，也不在任何 SubXID cache 中，它就不是 running SubXID。
```

overflow 后，这个结论不再成立。读者必须查 `pg_subtrans`：

```text
SubXID -> parent/top-level XID -> 判断 top-level XID 是否 running
```

所以 SubXID overflow 是 correctness fallback，不只是性能优化缺失。

### 4.7 VXID lock 与 XID lock

`AssignTransactionId()` 分配真实 XID 后会调用 `XactLockTableInsert(xid)`。这让其它 backend 能等待这个真实 XID 完成。

对比：

| 身份 | 创建时机 | 主要用途 | 是否写入 tuple header |
| --- | --- | --- | --- |
| VXID lock | 顶层事务开始时 | 等待事务生命周期结束 | 否 |
| XID lock | 真实 XID 分配后 | 等待 XID 结果确定 | 可能 |

只读事务通常只有 VXID lock。写事务拥有 VXID lock，也拥有 XID lock。

`pg_locks.virtualxid` 和 `pg_locks.transactionid` 不能混为一谈。

### 4.8 `pg_xact` 与 `pg_subtrans`

`pg_xact` 记录事务结果。`pg_subtrans` 记录 SubXID 到 parent 的关系。

它们不分配身份。但分配身份前必须准备它们的存储边界。

`GetNewTransactionId()` 在推进 `nextXid` 前调用：

```text
ExtendCLOG(xid)
ExtendCommitTs(xid)
ExtendSUBTRANS(xid)
```

这保证 XID 一旦被发布，后续 commit/abort 和 SubXID parent lookup 都有地方写或查。

## 5. 主流程源码 walkthrough

### 5.1 backend 进入 ProcArray，但还没有事务 XID

`proc.c` 初始化 `MyProc` 时设置：

```text
MyProc->xid = InvalidTransactionId
MyProc->xmin = InvalidTransactionId
MyProc->vxid.procNumber = MyProcNumber
MyProc->vxid.lxid = InvalidLocalTransactionId
```

`InitProcessPhase2()` 随后调用 `ProcArrayAdd(MyProc)`。这说明 backend 已经成为 ProcArray member。

但 ProcArray membership 不等于当前事务已经有真实 XID。一个 backend 可以在 ProcArray 中，`ProcGlobal->xids[pgxactoff]` 仍是 invalid。

### 5.2 `StartTransaction()`: 创建 VXID，不创建 XID

`xact.c` 的 `StartTransaction()` 与本节相关的顺序是：

```text
CurrentTransactionState = &TopTransactionStateData
s->fullTransactionId = InvalidFullTransactionId
AtStart_Memory()
AtStart_ResourceOwner()
vxid.procNumber = MyProcNumber
vxid.localTransactionId = GetNextLocalTransactionId()
VirtualXactLockTableInsert(vxid)
MyProc->vxid.lxid = vxid.localTransactionId
s->state = TRANS_INPROGRESS
```

这里没有调用 `GetNewTransactionId()`。也没有写 `MyProc->xid`。

事务已开始，但真实 XID 未分配。`xid8funcs.c` 中的 SQL 函数正好展示了这个差异：

| 函数 | 内部 accessor | 未分配时行为 |
| --- | --- | --- |
| `pg_current_xact_id()` | `GetTopFullTransactionId()` | 强制分配并返回。 |
| `pg_current_xact_id_if_assigned()` | `GetTopFullTransactionIdIfAny()` | 返回 `NULL`。 |

### 5.3 只读事务路径：有 VXID，无 XID

普通只读事务可以从头到尾不分配真实 XID。路径概括：

```text
StartTransaction()
  -> 创建 transaction memory / resource owner
  -> 创建并发布 VXID
  -> MyProc->xid 仍 invalid
SELECT
  -> 可能使用 snapshot / command id
  -> 不写 tuple header
  -> 不调用强制分配 XID 的 accessor
CommitTransaction()
  -> RecordTransactionCommit()
     -> GetTopTransactionIdIfAny()
     -> 无 XID，不写普通 COMMIT record
  -> ProcArrayEndTransaction(MyProc, InvalidTransactionId)
     -> 清除 VXID / xmin / flags
```

`RecordTransactionCommit()` 明确处理无 XID commit。没有 XID 的事务不能、也不想写普通 commit record。

但是无 XID 不等于什么都没发生。源码仍保留 standby invalidation 的特殊 WAL 记录路径。

正确说法是：

```text
无 XID 表示没有产生需要由真实 XID 控制 MVCC tuple visibility 的变化。
```

### 5.4 第一次需要真实身份：强制分配 accessor

`GetCurrentTransactionId()` 的语义是：

```text
如果当前事务层级没有 XID，就调用 AssignTransactionId(s)；
然后返回当前事务层级的 32 bit XID。
```

`GetTopTransactionId()` 对顶层事务做同样事情。`IfAny` 版本不会分配。

常见 accessor 差异：

| accessor | 未分配时行为 | 典型用途 |
| --- | --- | --- |
| `GetTopTransactionId()` | 强制分配顶层 XID | 需要顶层真实身份。 |
| `GetTopTransactionIdIfAny()` | 返回 invalid | commit/abort 判断是否需要写结果。 |
| `GetCurrentTransactionId()` | 强制分配当前层级 XID | heap insert/update/delete。 |
| `GetCurrentTransactionIdIfAny()` | 返回 invalid | abort 或检查路径避免无意义分配。 |
| `GetTopFullTransactionId()` | 强制分配并返回 xid8 | `pg_current_xact_id()`。 |
| `GetTopFullTransactionIdIfAny()` | 不分配 | `pg_current_xact_id_if_assigned()`。 |

### 5.5 heap insert 如何触发 XID 分配

`heapam.c` 的 `heap_insert()` 开始就执行：

```text
TransactionId xid = GetCurrentTransactionId()
```

随后 `heap_prepare_insert()` 把它写入 tuple header：

```text
HeapTupleHeaderSetXmin(tup->t_data, xid)
```

所以写 tuple header 前，真实 XID 必须存在。从这个瞬间开始，事务身份不再只是内存状态。

它进入持久数据页。后续 visibility code 必须能判断这个 XID 是 running、committed、aborted，还是 sub-committed 需要追 parent。

### 5.6 `AssignTransactionId()`: 分配前后的协调层

`AssignTransactionId()` 不直接自增 `nextXid`。它负责维护分配前后的不变量。

主路径：

```text
AssignTransactionId(s)
  -> 确认当前事务层级 TRANS_INPROGRESS
  -> 禁止 parallel mode / parallel worker 中分配新 XID
  -> 如果是子事务，确保所有未分配 XID 的父事务先分配
  -> 如果 logical WAL 需要，准备记录 top XID assignment
  -> GetNewTransactionId(isSubXact)
  -> top-level: 设置 XactTopFullTransactionId
  -> subxact: SubTransSetParent(child, parent)
  -> top-level: RegisterPredicateLockingXid()
  -> XactLockTableInsert(xid)
  -> subxact + standby/logical: 必要时写 XLOG_XACT_ASSIGNMENT
```

子事务必须保证 parent XID 早于 child XID。源码避免深递归：先收集未分配父事务，再从最高父级往下分配。

分配后，子事务会写 `pg_subtrans` parent link。真实 XID 还会插入 transaction lock table。

这说明 `AssignTransactionId()` 是事务身份升级的协调层。它把本地事务栈、SubTrans、predicate locking、lock manager 和 WAL assignment 连接起来。

### 5.7 `GetNewTransactionId()`: 全局取号与发布

`varsup.c` 的 `GetNewTransactionId(bool isSubXact)` 是核心取号点。主路径：

```text
GetNewTransactionId(isSubXact)
  -> 禁止 parallel mode 分配
  -> bootstrap 特例
  -> recovery 中禁止分配
  -> LWLockAcquire(XidGenLock, LW_EXCLUSIVE)
  -> full_xid = TransamVariables->nextXid
  -> xid = XidFromFullTransactionId(full_xid)
  -> 检查 xidVacLimit / xidWarnLimit / xidStopLimit
  -> ExtendCLOG(xid)
  -> ExtendCommitTs(xid)
  -> ExtendSUBTRANS(xid)
  -> FullTransactionIdAdvance(&TransamVariables->nextXid)
  -> top-level: 发布 MyProc->xid 和 ProcGlobal->xids[]
  -> subxact: 写入 MyProc->subxids[] 或设置 overflowed
  -> LWLockRelease(XidGenLock)
  -> return full_xid
```

关键不变量：

- 不能在 CLOG 准备好之前推进 `nextXid`。
- 不能在释放 `XidGenLock` 后才发布 top-level XID。
- 不能让读者看到 count 已增加但 SubXID 数组槽还没初始化。
- overflow 后必须让读者走 `pg_subtrans` fallback。

所以分配 XID 的真实语义是：

```text
预留全局顺序 + 准备事务结果/父链存储 + 发布 running 身份。
```

### 5.8 top-level XID 发布

顶层事务分配成功后：

```text
MyProc->xid = xid
ProcGlobal->xids[MyProc->pgxactoff] = xid
```

从这一刻起：

- `TransactionIdIsInProgress(xid)` 可以通过 `ProcGlobal->xids[]` 找到它。
- `GetSnapshotData()` 可以把它纳入 running XID 集合。
- `ComputeXidHorizons()` 会把它和 `xmin` 一起考虑。
- `BackendXidGetPid()` 可以通过 main XID 找到 backend pid。

这就是真实 XID 成为共享事实的时刻。

### 5.9 SubXID 发布

子事务分配成功后，`GetNewTransactionId()` 处理 SubXID cache。未满时：

```text
MyProc->subxids.xids[nxids] = xid
pg_write_barrier()
MyProc->subxidStatus.count = substat->count = nxids + 1
```

已满时：

```text
MyProc->subxidStatus.overflowed = substat->overflowed = true
```

读者在 `TransactionIdIsInProgress()` 中先读 count，再用 read barrier 读数组元素。overflow 后，读者不能用“不在 cache 中”证明 SubXID 不 running。

它必须回 `pg_subtrans` 查 parent。

### 5.10 commit: 先写结果，再清除 running 身份

`CommitTransaction()` 的关键顺序：

```text
RecordTransactionCommit()
  -> GetTopTransactionIdIfAny()
  -> 如果无 XID，走无 XID分支
  -> 如果有 XID，写 commit WAL record
  -> 根据 synchronous_commit / WAL 状态 flush 或 async commit
  -> TransactionIdCommitTree() 或 TransactionIdAsyncCommitTree()
  -> 计算 latestXid
ProcArrayEndTransaction(MyProc, latestXid)
  -> 清除 ProcGlobal->xids[] 和 MyProc->xid
  -> 清除 vxid.lxid / xmin / delayChkptFlags
  -> 清除 SubXID cache
  -> MaintainLatestCompletedXid(latestXid)
  -> xactCompletionCount++
```

源码注释要求 `ProcArrayEndTransaction()` 在 `RecordTransactionCommit()` 之后、释放锁之前。这样其它 backend 看到 XID 不再 running 时，已经能从事务结果路径解释它。

注意一个中间状态：

```text
事务已经在 pg_xact 中标记 committed，但仍在 ProcArray 中显示为 running，并仍持有锁。
```

这是合法的。running set 与 commit result 是不同层次，按顺序收敛，而不是原子同时变化。

### 5.11 abort: 无 XID 直接结束，有 XID 写 abort 结果

`RecordTransactionAbort(bool isSubXact)` 先调用：

```text
GetCurrentTransactionIdIfAny()
```

如果没有 XID：

- 不写 abort record。
- 不标记 CLOG。
- 顶层事务会重置 `XactLastRecEnd`。
- 返回 `InvalidTransactionId`。

如果有 XID：

- 检查它没有已经 committed。
- 写 abort WAL record。
- `TransactionIdAbortTree(xid, children)`。
- 子事务 abort 时调用 `XidCacheRemoveRunningXids()`。
- 返回 `latestXid`。

顶层 abort 随后调用 `ProcArrayEndTransaction(MyProc, latestXid)`。同样遵守先结果、后清除 running 身份。

### 5.12 running 判断

`procarray.c` 的 `TransactionIdIsInProgress(xid)` 展示了 XID 发布后的消费方式。主逻辑：

```text
1. 早于 RecentXmin，直接认为不 running。
2. 当前事务或当前子事务，本地判断。
3. 扫描 ProcGlobal->xids[] 查 top-level XID。
4. 扫描 PGPROC subxid cache 查 cached SubXID。
5. hot standby 下查 KnownAssignedXids。
6. 如果存在 overflow，回 pg_subtrans 查 topmost parent，再判断 parent 是否 running。
```

这里可以看到 SubXID overflow 的成本来源。快路径是 shared memory array check。

慢路径是 `pg_subtrans` parent lookup。

### 5.13 完成结果判断

当一个 XID 不再 running，visibility code 需要知道它 committed 还是 aborted。`transam.c` 提供：

```text
TransactionIdDidCommit()
TransactionIdDidAbort()
TransactionIdCommitTree()
TransactionIdAsyncCommitTree()
TransactionIdAbortTree()
```

`TransactionIdDidCommit()` / `TransactionIdDidAbort()` 通过 `TransactionLogFetch()` 查 `pg_xact`。如果看到 `TRANSACTION_STATUS_SUB_COMMITTED`，还要回 `pg_subtrans` 找 parent，再递归判断 parent。

这引出下一节：

```text
本节解释 XID 何时出现；
下一节解释 XID 不再 running 后，pg_xact 如何表达 committed / aborted / sub-committed。
```

## 6. 生命周期 / ownership / cleanup

### 6.1 谁创建

backend identity 由 `proc.c` 创建。`InitProcess()` 领取 `PGPROC`，设置 `MyProc` 和 `MyProcNumber`。

ProcArray membership 由 `InitProcessPhase2()` 调用 `ProcArrayAdd(MyProc)` 建立。VXID 由 `StartTransaction()` 创建：

```text
GetNextLocalTransactionId()
VirtualXactLockTableInsert(vxid)
MyProc->vxid.lxid = lxid
```

真实 XID 由强制分配 accessor 触发：

```text
GetTopTransactionId()
GetCurrentTransactionId()
GetTopFullTransactionId()
GetCurrentFullTransactionId()
```

实际取号由 `AssignTransactionId()` 调用 `GetNewTransactionId()` 完成。

### 6.2 谁持有

不同状态有不同 owner：

| 状态 | owner | 生命周期 |
| --- | --- | --- |
| `TransactionStateData.fullTransactionId` | 当前 backend | 当前事务层级。 |
| `XactTopFullTransactionId` | 当前 backend | 顶层事务。 |
| `MyProc->xid` | 当前 backend 写，其它 backend 读 | 分配 XID 到事务结束。 |
| `ProcGlobal->xids[]` | 当前 backend 写，ProcArray 扫描者读 | 与 `MyProc->xid` 同步维护。 |
| tuple header `xmin` / `xmax` | heap page | tuple version 生命周期。 |
| `pg_xact` status | transaction log storage | 到可截断或 frozen 解释后。 |
| `pg_subtrans` parent | subtrans storage | 到不再需要查父链。 |
| XID lock | lock manager / resource owner | 事务结束释放。 |

不要把 MemoryContext 或 ResourceOwner 当作 XID owner。XID 一旦分配，不能因为 ERROR 或 abort 还给 `nextXid`。

abort 只能把它标记为 aborted，并清理 running publication。

### 6.3 commit cleanup

正常 commit 的关键顺序：

```text
RecordTransactionCommit()
  -> 写 WAL / pg_xact
  -> 得到 latestXid
ProcArrayEndTransaction()
  -> 清除 running publication
  -> 推进 latestCompletedXid
  -> xactCompletionCount++
post-commit cleanup
  -> ResourceOwnerRelease(...)
  -> release locks
  -> AtEOXact_* callbacks
  -> memory cleanup
```

`CommitTransaction()` 注释强调：先释放对其它 backend 可见的资源，再释放 locks，最后释放 backend-local 资源。这保证等待者看到事务完成后，不会遇到结果、锁和资源状态互相矛盾。

### 6.4 abort cleanup

顶层 abort 的关键顺序：

```text
AbortTransaction()
  -> AtAbort_* / AtEOXact_* 回滚子系统状态
  -> RecordTransactionAbort(false)
  -> ProcArrayEndTransaction(MyProc, latestXid)
  -> ResourceOwnerRelease(..., isCommit=false)
  -> lock cleanup
  -> memory cleanup
```

`TransactionAbortContext` 在启动时预留，保证 OOM 等错误后仍有空间完成 abort cleanup。这说明 ERROR cleanup 不能假设普通内存分配仍然可靠。

### 6.5 子事务 commit

`CommitSubTransaction()` 不会把 SubXID 立刻标成普通 committed。顶层 commit/abort 统一处理整个 transaction tree。

子事务 commit 时主要做：

- `CommandCounterIncrement()`。
- `AtSubCommit_childXids()` 把 child XID 归并到 parent。
- 释放当前子事务的 XID lock。
- 把可继承资源转移到父事务 owner。
- 清理当前层级 memory / callback / snapshot 状态。

所以 subcommit 表示局部成功，等待顶层最终结果。它不是一个对其它事务独立可见的 commit。

### 6.6 子事务 abort

`AbortSubTransaction()` 调用 `RecordTransactionAbort(true)`。如果子事务有 XID，会写 abort 结果。

随后 `XidCacheRemoveRunningXids()` 把失败 SubXID 从当前 backend 的 running child cache 中移除。该函数持有 `ProcArrayLock` exclusive，并推进 `latestCompletedXid` 与 `xactCompletionCount`。

如果 SubXID cache 已 overflow，找不到某个 SubXID 不一定是严重错误。正确性由 `pg_subtrans` fallback 维持。

### 6.7 backend exit 兜底

`proc.c` 注册：

```text
on_shmem_exit(RemoveProcFromArray, 0)
```

`RemoveProcFromArray()` 调用 `ProcArrayRemove(MyProc, InvalidTransactionId)`。要区分：

| 路径 | 语义 |
| --- | --- |
| `ProcArrayEndTransaction()` | 当前事务结束，backend 仍可处理下一事务。 |
| `ProcArrayRemove()` | backend 离开 ProcArray membership，通常是进程退出或 2PC 相关路径。 |

进程退出是 membership cleanup。事务结束是 running transaction publication cleanup。

## 7. 正确性机制层次

### 7.1 延迟分配是语义边界

延迟分配 XID 不只是性能优化。它定义了事务身份从本地生命周期进入 MVCC 世界的条件。

只读事务不写 tuple header，不需要真实 XID 控制 tuple visibility。VXID 解决等待事务生命周期的问题。

真实 XID 解决 tuple version 出生、死亡和结果判定的问题。

### 7.2 `XidGenLock` 串起取号协议

`XidGenLock` 保护 `nextXid`，也把取号、状态存储准备和 ProcArray publication 串成协议。在锁内完成：

```text
读取 nextXid
检查 wrap limits
扩展 CLOG / commit_ts / SUBTRANS
推进 nextXid
写 MyProc->xid 或 SubXID cache
写 ProcGlobal dense arrays
```

如果把 `nextXid` 自增和 ProcArray 发布拆开，snapshot 可能漏掉已经分配、即将写入 tuple header 的 XID。

### 7.3 CLOG / SUBTRANS readiness

`ExtendCLOG()` 必须早于 `nextXid` 推进。`ExtendSUBTRANS()` 必须早于后续 SubXID parent 查询需求。

`AssignTransactionId()` 还要求 SubTrans parent 信息在必要时早于 XID 被其它存储解释。共同目标是：

```text
别人一旦看到 XID，就总能沿规定路径解释它。
```

### 7.4 ProcArray publication 与 snapshot

真实 XID 发布到 `ProcGlobal->xids[]` 后，snapshot 构造者可以把它纳入 running set。事务结束时，`ProcArrayEndTransaction()` 持 `ProcArrayLock` 清除。

原因是不能在别人拿 snapshot 时，让一个已分配且未完成的 XID 从 running set 中消失。`latestCompletedXid` 和 `xactCompletionCount` 也在这个边界下推进。

### 7.5 WAL / pg_xact result

commit 路径有 XID 时需要写 commit WAL record，并按同步级别 flush 或 async commit，再写 `pg_xact` 结果。abort 路径通常不强制 flush abort WAL，因为 crash 后默认可把未提交事务视为 aborted。

这里的边界：

```text
commit 的可见性结果不能丢；
abort 结果丢失通常不会导致未提交数据被当作 committed。
```

更细的 WAL-before-CLOG 顺序留到本目录第 3 课。

### 7.6 XID lock 与 VXID lock

VXID lock 等事务生命周期。XID lock 等真实 XID 结果。

`VirtualXactLock()` 还要处理 VXID 对应事务可能已经 prepared、转换到 2PC XID 的情况。这说明等待语义必须跨越 VXID-only、ordinary XID、prepared transaction。

### 7.7 正确性不是 raw field

`MyProc->xid` 单独不是语义。`MyProc->xid + ProcGlobal->xids[] + XidGenLock + ProcArrayLock + pg_xact result + cleanup order` 才是语义。

同理，`backend_xid` 为 NULL 不能说明没有事务。它只能说明当前 backend 没有已发布的 top-level real XID。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 parallel mode 禁止新分配

`AssignTransactionId()` 和 `GetNewTransactionId()` 都禁止 parallel mode 中分配新 XID。原因是 parallel operation 开始时，worker 已经同步了事务状态。

之后突然分配新 XID 会让 leader / worker 对事务身份、WAL、resource ownership 的理解分裂。`heap_prepare_insert()` 也禁止 parallel worker insert tuple。

### 8.2 recovery 中禁止普通分配

`GetNewTransactionId()` 在 recovery 中 ERROR。standby 不能像 primary 一样取新 XID。

它通过 WAL 中的 running xacts、KnownAssignedXids 和 assignment records 重建 primary 上的 running XID 事实。主路径是：

```text
RecordKnownAssignedTransactionIds()
ProcArrayApplyXidAssignment()
KnownAssignedXidsAdd()
KnownAssignedXidsRemoveTree()
```

本节只保留边界：

```text
primary 是分配 XID；
standby 是从 WAL 重建哪些 XID 应被视作 running。
```

### 8.3 wraparound protection

`GetNewTransactionId()` 在分配前检查：

```text
xidVacLimit
xidWarnLimit
xidStopLimit
xidWrapLimit
```

到 `xidVacLimit` 后，会按节流请求 autovacuum launcher。到 `xidWarnLimit` 后，发 warning。

到 `xidStopLimit` 后，普通 multiuser 模式拒绝继续分配 XID。报错 hint 会指向 database-wide VACUUM、旧 prepared transactions、stale replication slots。

这说明 XID 分配压力会传播到维护任务、2PC 和 replication slot。

### 8.4 无 XID commit 的 invalidation fallback

无 XID 事务不写普通 commit record。但如果存在 standby 需要处理的 invalidation message，`RecordTransactionCommit()` 可以写特殊 standby invalidation record。

源码注释承认这些场景有历史复杂性。这提醒我们不要把实现理想化。

PostgreSQL 会为了兼容性和运行边界保留 awkward path。

### 8.5 async commit

允许 asynchronous commit 时，commit 路径会调用：

```text
XLogSetAsyncXactLSN(XactLastRecEnd)
TransactionIdAsyncCommitTree(xid, children, XactLastRecEnd)
```

XID 分配、commit WAL insert、WAL flush、`pg_xact` result、ProcArray 清除是不同层次。不要把“有 XID”理解成“已 durable”。

不要把“不在 ProcArray running set”理解成“不需要查 `pg_xact`”。

### 8.6 SubXID overflow

超过 `PGPROC_MAX_CACHED_SUBXIDS` 后，SubXID cache 设置 overflow flag。`TransactionIdIsInProgress()` 必须回查 `pg_subtrans`：

```text
保存可能有 overflow child 的 top XIDs
释放 ProcArrayLock
检查目标 xid 是否已 aborted
SubTransGetTopmostTransaction(xid)
判断 topxid 是否仍 running
```

这条 fallback 维持 correctness，但增加 CPU、SLRU 和 cache miss 成本。

### 8.7 cleanup 可重复

`XidCacheRemoveRunningXids()` 注释提到，`AbortSubTransaction()` 中发生错误时可能重复尝试移除同一 SubXID。因此找不到 SubXID 时，在某些情况下只 warning。

异常路径的目标是状态收敛，不是扩大故障。

### 8.8 bootstrap 特例

bootstrap processing 中，`GetNewTransactionId()` 返回 `BootstrapTransactionId`，并直接写 `MyProc->xid` 与 `ProcGlobal->xids[]`。这是初始化特例。

不要把它推广到普通事务路径。

## 9. 成本、资源与跨模块传播

### 9.1 XID 消耗

真实 XID 是有限的 32 bit 空间。每次分配都会推进 `nextXid`。

XID 不能因为 abort 而回收。误用 `pg_current_xact_id()` 会把原本无 XID 的只读事务变成 XID 消耗源。

传播路径：

```text
更多 XID 分配
  -> 更快接近 xidVacLimit / xidWarnLimit
  -> autovacuum pressure 上升
  -> frozen horizon 维护压力上升
  -> 极端情况下 xidStopLimit 拒绝分配
```

### 9.2 `XidGenLock` contention

每个真实 XID 分配都要拿 `XidGenLock` exclusive。高并发写事务、频繁 savepoint、PL/pgSQL exception block、误用强制 XID 函数，都会增加竞争。

但不要把所有写性能问题都归因到 `XidGenLock`。瓶颈可能迁移到 WAL insertion、buffer extension、CLOG group update、lock manager、ProcArrayEndTransaction、index insert、fsync 或 replication wait。

### 9.3 CLOG / SUBTRANS 扩展

`GetNewTransactionId()` 在新 CLOG 页边界扩展 CLOG。一页覆盖很多事务，所以不是每次分配都昂贵。

但扩展发生在 `XidGenLock` 持有期间，边界时刻可能放大延迟。SubXID-heavy workload 还会增加 SUBTRANS 压力。

### 9.4 ProcArray 扫描

XID 发布后，`ProcGlobal->xids[]` 会被 snapshot、horizon 和 running 判断扫描。成本随 backend 数、活跃写事务数、snapshot 获取频率和事务完成频率增长。

dense arrays 是为了减少扫描大量 backend 时的 cache miss。但它仍然是 shared memory hot path。

### 9.5 SubXID overflow 成本

未 overflow 时，running 判断可以通过 top XID 和 cached SubXID 快速完成。overflow 后，读者必须保守，可能访问 `pg_subtrans`。

可见性判断成本从 tuple 本地分支扩散到事务系统和 SLRU。

### 9.6 WAL / standby / logical 传播

SubXID assignment 在 standby info active 或 logical decoding 相关场景中需要 WAL 记录。传播路径：

```text
SubXID assignment
  -> unreportedXids[]
  -> XLOG_XACT_ASSIGNMENT
  -> standby KnownAssignedXids / pg_subtrans reconstruction
  -> hot standby snapshot correctness
  -> logical decoding transaction assembly
```

XID 分配影响 primary，也影响 replay 和 decoding。

### 9.7 cleanup horizon

`ComputeXidHorizons()` 同时考虑 backend 的 `xmin` 和 `xid`。源码注释强调：

```text
一个事务可能有 xmin 但还没有 xid；
也可能有 xid，但 xmin 还没设置。
```

因此 horizon 计算必须看二者的 older 值。这正是本节的抽象：事务身份不是单字段。

### 9.8 后台进程

| 进程 | 相关性 |
| --- | --- |
| autovacuum launcher / worker | XID 接近限制时被请求；推进 frozen horizon。 |
| walwriter | async commit / abort LSN 后续 flush。 |
| checkpointer | commit critical section 通过 `delayChkptFlags` 约束 checkpoint。 |
| startup process | standby recovery 中用 KnownAssignedXids 重建 running XID。 |
| walsender | hot standby feedback、slot xmin、logical decoding 影响 cleanup horizon。 |
| logical workers | decoding 需要 XID / SubXID assignment 信息归并事务。 |

## 10. 观测与诊断入口

### 10.1 直接可见

| 入口 | 粒度 | 能看到什么 |
| --- | --- | --- |
| `pg_current_xact_id_if_assigned()` | 当前事务 | 是否已有真实 XID，不强制分配。 |
| `pg_current_xact_id()` | 当前事务 | 强制分配并返回真实 XID。 |
| `pg_stat_activity.backend_xid` | backend 当前状态 | 当前 top-level XID，如果已分配。 |
| `pg_stat_activity.backend_xmin` | backend 当前状态 | 当前 advertised xmin。 |
| `pg_locks.virtualxid` | lock table 当前状态 | VXID lock，事务生命周期可被等待。 |
| `pg_locks.transactionid` | lock table 当前状态 | 真实 XID lock 或等待。 |
| wraparound warning 日志 | database / instance | XID 分配接近危险边界。 |
| `pg_control_checkpoint()` 类入口 | control state | `next_xid`、`oldest_xid` 等全局状态。 |

`backend_xid` 为 NULL 不代表没有事务。它只代表当前事务没有已发布的 top-level real XID。

### 10.2 只能推断或断点观察

这些状态通常不能通过 SQL 完整看到：

- `TransactionStateData.fullTransactionId`。
- `XactTopFullTransactionId`。
- `MyProc->subxids[]` 的完整内容。
- `ProcGlobal->subxidStates[]` 的瞬时一致视图。
- `unreportedXids[]`。
- `TopTransactionStateData.didLogXid`。
- `GetNewTransactionId()` 内部刚取号未发布的瞬间。

需要 gdb、tracepoint、临时日志或 profiling。生产诊断中不要依赖 backend-local 指针、`pgxactoff` 瞬时值、SubXID cache 数组顺序或 snapshot scan 的精确 interleaving。

### 10.3 SQL 观测：强制与非强制函数

会话 A：

```sql
BEGIN;
SELECT pg_current_xact_id_if_assigned();
SELECT pg_current_xact_id();
SELECT pg_current_xact_id_if_assigned();
```

预期：

```text
第一次为 NULL；
pg_current_xact_id() 返回 xid8；
第二次返回同一个 xid8。
```

源码对应：

```text
pg_current_xact_id()
  -> GetTopFullTransactionId()
  -> AssignTransactionId()
  -> GetNewTransactionId()
```

### 10.4 SQL 观测：`backend_xid`

会话 A：

```sql
BEGIN;
SELECT pg_backend_pid();
SELECT pg_sleep(60);
```

会话 B：

```sql
SELECT pid, backend_xid, backend_xmin, state
FROM pg_stat_activity
WHERE pid = <会话A pid>;
```

预期：`backend_xid` 可能为 NULL。会话 A 再执行：

```sql
SELECT pg_current_xact_id();
SELECT pg_sleep(60);
```

会话 B 再查，`backend_xid` 应出现。

### 10.5 SQL 观测：VXID 与 transactionid lock

会话 A 只读 sleep 时，通常能在 `pg_locks` 看到 `virtualxid`，但不一定有 `transactionid`。写入后再 sleep：

```sql
BEGIN;
CREATE TEMP TABLE xid_demo(i int);
INSERT INTO xid_demo VALUES (1);
SELECT pg_sleep(60);
```

会话 B：

```sql
SELECT locktype, virtualxid, transactionid, mode, granted
FROM pg_locks
WHERE pid = <会话A pid>
ORDER BY locktype, mode;
```

判断时记住：

```text
virtualxid 表示事务生命周期；
transactionid 表示真实 XID。
```

### 10.6 gdb / profiling

建议断点：

```text
StartTransaction
GetTopFullTransactionId
GetCurrentTransactionId
AssignTransactionId
GetNewTransactionId
heap_insert
RecordTransactionCommit
RecordTransactionAbort
ProcArrayEndTransaction
TransactionIdIsInProgress
```

建议观察：

```text
CurrentTransactionState->fullTransactionId
XactTopFullTransactionId
MyProc->vxid.lxid
MyProc->xid
ProcGlobal->xids[MyProc->pgxactoff]
MyProc->subxidStatus
TransamVariables->nextXid
TransamVariables->latestCompletedXid
```

怀疑 CPU 成本时，看 `GetNewTransactionId()`、`LWLockAcquire()`、`ExtendCLOG()`、`ExtendSUBTRANS()`、`TransactionIdIsInProgress()`、`SubTransGetTopmostTransaction()`、`ProcArrayEndTransaction()`。SQL 视图能证明状态边界，不一定能完成性能归因。

## 11. 常见误区

误区一：`BEGIN` 一定分配 XID。实际：`BEGIN` 创建事务生命周期和 VXID；真实 XID 延迟到需要时分配。

误区二：`backend_xid` 为 NULL 说明没有事务。实际：可能有事务，只是尚未分配真实 XID。

误区三：`virtualxid` 是 XID 的另一种显示格式。实际：VXID 是 `procNumber + localTransactionId`，短期唯一，不写磁盘，不参与 tuple visibility。

误区四：XID 分配只是 `nextXid++`。实际：它还涉及 wraparound 防线、CLOG/SUBTRANS 扩展、ProcArray publication、SubXID cache、XID lock 和 WAL assignment。

误区五：abort 可以把 XID 还回去。实际：XID 一旦分配就消耗掉；abort 只能写 aborted 结果并清理 running publication。

误区六：SubXID cache overflow 只是少了一个优化。实际：overflow 改变 running 判断路径，读者必须回查 `pg_subtrans`。

误区七：事务从 ProcArray 消失后就不需要 CLOG。实际：从 running set 消失后，visibility code 更需要 `pg_xact` 判断 committed / aborted。

误区八：`pg_current_xact_id()` 是无副作用观测函数。实际：它会在未分配时强制分配 XID；无副作用观测应使用 `pg_current_xact_id_if_assigned()`。

## 12. 课堂实验

### 实验 1：证明事务开始不等于 XID 分配

目标：观察 VXID-only 事务，并用 SQL 函数触发真实 XID 分配。会话 A：

```sql
BEGIN;
SELECT pg_backend_pid() AS pid;
SELECT pg_current_xact_id_if_assigned() AS before_assignment;
```

预期：`before_assignment` 为 NULL。会话 B：

```sql
SELECT pid, backend_xid, backend_xmin, state
FROM pg_stat_activity
WHERE pid = <会话A pid>;
```

预期：`backend_xid` 可能为 NULL。会话 A：

```sql
SELECT pg_current_xact_id() AS forced_xid;
SELECT pg_current_xact_id_if_assigned() AS after_assignment;
```

会话 B 再查 `pg_stat_activity`。预期：`backend_xid` 出现，`forced_xid` 与 `after_assignment` 相同。

源码回看：`xid8funcs.c: pg_current_xact_id()` -> `xact.c: GetTopFullTransactionId()` -> `AssignTransactionId()` -> `varsup.c: GetNewTransactionId()`。

### 实验 2：观察写 tuple 前的 XID 分配

目标：把 heap insert 与 XID assignment 连接起来。准备：

```sql
CREATE TABLE xid_assignment_demo(id int);
```

会话 A：

```sql
BEGIN;
SELECT pg_current_xact_id_if_assigned();
INSERT INTO xid_assignment_demo VALUES (1);
SELECT pg_current_xact_id_if_assigned();
SELECT pg_sleep(60);
```

预期：`INSERT` 前为 NULL，`INSERT` 后有 XID。会话 B：

```sql
SELECT pid, backend_xid, backend_xmin, wait_event_type, wait_event
FROM pg_stat_activity
WHERE query LIKE '%pg_sleep(60)%';
```

可选观察 lock：

```sql
SELECT locktype, virtualxid, transactionid, mode, granted
FROM pg_locks
WHERE pid = <会话A pid>
ORDER BY locktype, mode;
```

源码回看：`heapam.c: heap_insert()` -> `GetCurrentTransactionId()` -> `heap_prepare_insert()` -> `HeapTupleHeaderSetXmin()`。

### 实验 3：断点画出 identity 状态迁移

目标：用断点验证 VXID、XID、ProcArray publication、commit cleanup 的顺序。建议 debug build，低并发本地实例。

断点：

```text
StartTransaction
AssignTransactionId
GetNewTransactionId
heap_insert
RecordTransactionCommit
ProcArrayEndTransaction
```

测试 SQL：

```sql
BEGIN;
INSERT INTO xid_assignment_demo VALUES (2);
COMMIT;
```

每个断点记录：

```text
MyProc->vxid.lxid
MyProc->xid
ProcGlobal->xids[MyProc->pgxactoff]
CurrentTransactionState->fullTransactionId
XactTopFullTransactionId
TransamVariables->nextXid
```

预期状态迁移：

| 时间点 | VXID | top XID | `ProcGlobal->xids[]` | tuple header | `pg_xact` result |
| --- | --- | --- | --- | --- | --- |
| `StartTransaction()` 后 | 有 | 无 | invalid | 无 | 无 |
| `GetNewTransactionId()` 返回后 | 有 | 有 | 已发布 | 尚未写入 | 未完成 |
| `heap_prepare_insert()` 后 | 有 | 有 | 已发布 | `xmin = xid` | 未完成 |
| `RecordTransactionCommit()` 后 | 有 | 有 | 仍 running | `xmin = xid` | committed / async committed |
| `ProcArrayEndTransaction()` 后 | 清除 | 本地 cleanup | invalid | `xmin = xid` | committed |

## 13. 讨论题

1. 为什么 PostgreSQL 不在 `BEGIN` 时就给每个事务分配真实 XID？
2. 如果 `GetNewTransactionId()` 先释放 `XidGenLock`，再写 `ProcGlobal->xids[]`，snapshot 可能出现什么错误？
3. 为什么 `pg_current_xact_id_if_assigned()` 适合观测，而 `pg_current_xact_id()` 会改变系统状态？
4. 一个事务已经写了 tuple header 中的 `xmin`，但还没有从 ProcArray 清除。另一个 backend 应该如何判断这个 tuple 是否可见？
5. 子事务为什么必须先确保父事务有 XID？如果 child XID 早于 parent XID，会破坏哪些判断？
6. SubXID cache overflow 后，为什么不能简单认为“不在 cache 里就是不 running”？
7. commit 路径为什么要先写 WAL / `pg_xact`，再调用 `ProcArrayEndTransaction()`？
8. `backend_xid`、`backend_xmin`、`virtualxid` lock、`transactionid` lock 分别能诊断什么，不能诊断什么？

## 14. 本节小结

本节主链路：

```text
StartTransaction()
  -> 创建 VXID，不分配真实 XID
GetCurrentTransactionId() / GetTopTransactionId()
  -> AssignTransactionId()
  -> GetNewTransactionId()
  -> 发布 MyProc->xid / ProcGlobal->xids[]
heap insert/update/delete
  -> 把 XID 写入 tuple header
RecordTransactionCommit/Abort()
  -> 写 WAL / pg_xact 结果
ProcArrayEndTransaction()
  -> 清除 running publication
```

核心边界：

- VXID 是事务生命周期身份，可等待，不参与 tuple visibility。
- XID / FullTransactionId 是真实事务身份，分配后进入全局顺序，可能写入 tuple header。
- `PGPROC` / `ProcGlobal` 是 running XID publication set。
- `pg_xact` 是 completed result。
- `pg_subtrans` 是 SubXID 到 parent 的 fallback chain。

ownership / cleanup：

- `TransactionStateData` 管本地事务栈。
- `ResourceOwner` 管锁、pin、snapshot 等资源。
- ProcArray publication 管其它 backend 是否认为该 XID running。
- `pg_xact` 管最终结果。
- abort 不能归还 XID，只能标记结果并清理 publication。

错误和 fallback：

- parallel mode / recovery 中禁止普通 XID 分配。
- wraparound limit 在分配前阻断。
- 无 XID commit/abort 不写普通事务结果。
- SubXID overflow 后回查 `pg_subtrans`。
- async commit 把 WAL flush 与 `pg_xact` 可见性边界拆开。

观测结论：

- `pg_current_xact_id_if_assigned()` 可以无副作用观察当前事务是否已有 XID。
- `pg_current_xact_id()` 会强制分配。
- `pg_stat_activity.backend_xid` 为空不代表没有事务。
- `pg_locks.virtualxid` 和 `transactionid` lock 表示不同身份层次。
- SubXID cache、`XactTopFullTransactionId`、`unreportedXids` 等需要源码断点或 profiling 才能可靠观察。

可迁移系统规律：

```text
内核系统中的 identity 往往不是单一字段。
一个对象可能先拥有本地生命周期身份，再拥有可等待身份，最后才拥有持久可见性身份。
只有状态跨过 publication boundary 后，其他并发参与者才能把它当作系统事实。
正确性来自“何时发布、发布前准备了什么、谁能读、何时撤销发布、撤销前结果是否可解释”这一整套协议，而不是来自某个 raw field。
```

下一节会沿着本节产生的真实 XID，进入 `pg_xact` / CLOG 状态机：一个 XID 不再 running 后，可见性代码如何判断它 committed、aborted，还是 sub-committed 需要继续追 parent。
