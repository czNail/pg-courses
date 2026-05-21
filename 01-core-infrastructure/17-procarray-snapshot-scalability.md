# PostgreSQL Snapshot 获取与 ProcArray scan 扩展性

## 课程定位

前置知识：已经理解 `PGPROC` 是 backend 在 shared memory 中的身份，也理解 `ProcArray` 是事务状态 publication set，知道 XID、SubXID、xmin、status flags 什么时候被发布和清除。

本节唯一主问题：

```text
为什么一个普通 snapshot 需要扫描全局 backend 状态，GetSnapshotData() 如何在 visibility correctness 与 MaxBackends 扩展性之间折中？
```

核心矛盾：MVCC snapshot 必须给当前查询一个稳定的“哪些事务在我看来仍在运行”的答案；但这个答案来自所有 backend 的共享事务状态，而获取 snapshot 又发生在查询入口、catalog 访问、索引构建、portal、并行查询等高频路径上。PostgreSQL 不能为了正确性每次都做昂贵的全局精确计算，也不能为了性能跳过会影响可见性的运行中事务。因此 `GetSnapshotData()` 用 `ProcArrayLock`、dense arrays、`latestCompletedXid`、`xactCompletionCount`、SubXID cache 和 GlobalVis 近似边界，在正确性和扩展性之间做了一组很具体的工程折中。

学完后应能判断：

```text
为什么 snapshot 必须包含 xmin / xmax / xip / subxip；
GetSnapshotData() 为什么要持有 ProcArrayLock shared；
xmax 为什么来自 latestCompletedXid + 1；
为什么 snapshot scan 跳过自己、跳过 xid >= xmax、跳过 lazy VACUUM / logical decoding；
SubXID overflow 如何把一次数组查找退化成 pg_subtrans fallback；
xactCompletionCount 为什么能复用静态 snapshot；
为什么 GetSnapshotData() 不再计算精确 oldest xmin；
MaxBackends、活跃 XID 数、SubXID 数和事务结束竞争如何放大 snapshot hot path 成本。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节讲的是事务状态如何发布：

```text
backend 加入 ProcArray
  -> 事务需要时分配 XID
  -> XID / SubXID / xmin / statusFlags 进入 PGPROC 和 dense arrays
  -> commit / abort 时清除 shared transaction state
```

本节换到读者视角：

```text
一个查询开始执行
  -> 需要一个 MVCC snapshot
  -> 从 ProcArray 读取所有可能影响可见性的 running XID
  -> 把全局 mutable state 压缩成本 backend 可反复使用的 SnapshotData
  -> tuple visibility 后续只查本地 snapshot，尽量不再碰 ProcArray
```

这里的关键转变是：

```text
ProcArray 是全局、会变的 shared state；
SnapshotData 是当前查询看到的稳定局部视图。
```

`GetSnapshotData()` 的价值就在这条边界上。它在一个短暂的同步窗口里读取全局运行中事务集，然后把结果变成 `xmin`、`xmax`、`xip[]`、`subxip[]`。后续 `HeapTupleSatisfiesMVCC()` 判断每一行 tuple 是否可见时，主要调用 `XidInMVCCSnapshot()` 查这个本地 snapshot，而不是每个 tuple 都重新扫描 `ProcArray`。

如果把这个边界去掉，tuple visibility 会在每个 tuple 上访问高竞争 shared state。那会把一次 seq scan、index scan、join、sort 前过滤的成本扩大到不可接受。

本节不深入 VACUUM horizon 的所有分类，那是下一节的主线。本节只讲 `GetSnapshotData()` 为了不拖慢 snapshot hot path，对 GlobalVis 做的近似更新，以及它为什么不在这里做完整 `ComputeXidHorizons()`。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
snapmgr.c 在查询需要 MVCC 视图时调用 GetSnapshotData()；
GetSnapshotData() 在 ProcArrayLock shared 下读取 latestCompletedXid、ProcGlobal->xids[]、subxidStates[]、statusFlags[] 和必要的 PGPROC subxid cache；
它构造 xmin / xmax / xip / subxip，安装 MyProc->xmin，并记录 snapXactCompletionCount；
后续 tuple visibility 用 XidInMVCCSnapshot() 在本地 snapshot 上判断 xid 是否仍在运行。
```

这背后的 tension 是：

```text
visibility correctness 要求 snapshot 看到一个一致的 running-xid 集合；
scalability 要求 snapshot 获取不能随 MaxBackends 和 SubXID 数无限制恶化。
```

PostgreSQL 的折中不是单一技巧，而是一组边界：

| 折中点 | 正确性收益 | 性能收益 |
| --- | --- | --- |
| `ProcArrayLock` shared vs transaction end exclusive | snapshot 构造期间没有 XID 从 running set 消失 | 多个 backend 可以并发获取 snapshot |
| `latestCompletedXid + 1` 作为 `xmax` | 给“新于 snapshot 的事务”一个统一上界 | 避免读取 `nextXid` 并追踪每个新事务 |
| dense `ProcGlobal->xids[]` | 扫描时拿到所有运行中 top XID | 减少 `PGPROC *` 跳转和 cache miss |
| 跳过 `xid >= xmax` | 这些事务按规则天然不可见 | 不必写入 `xip[]`，也不必处理它们的 SubXID |
| `xactCompletionCount` 复用 | 没有带 XID 的事务结束时 snapshot 内容不变 | 避免重复重建静态 snapshot 数组 |
| SubXID cache + overflow | 小事务快，大事务正确 | 正常情况数组复制，极端情况回退 `pg_subtrans` |
| GlobalVis 近似边界 | 不破坏 cleanup correctness | 避免 snapshot hot path 扫描频繁变化的 `xmin` |

本节要建立的系统规律是：

```text
高频读路径上的全局一致性，通常不是靠“每次读完整真相”实现；
而是靠一个短同步窗口冻结必要状态，再把后续判断转移到本地、不可变或近似可验证的数据结构上。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | `GetTransactionSnapshot()`、`GetLatestSnapshot()`、`GetCatalogSnapshot()` 如何选择和缓存 snapshot。 |
| 2 | `src/include/utils/snapshot.h` | `SnapshotData` 中 `xmin`、`xmax`、`xip`、`subxip`、`suboverflowed`、`snapXactCompletionCount` 的语义。 |
| 3 | `src/backend/storage/ipc/procarray.c` | `GetSnapshotData()` 主流程、`GetSnapshotDataReuse()`、GlobalVis 近似更新。 |
| 4 | `src/backend/utils/time/snapmgr.c` | `XidInMVCCSnapshot()` 如何消费 snapshot，SubXID overflow 如何 fallback。 |
| 5 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesMVCC()` 为什么依赖 snapshot，而不是每次查全局事务状态。 |
| 6 | `src/backend/access/transam/README` | transaction end 与 snapshot-taking 的 interlocking 正确性说明。 |
| 7 | `src/include/storage/proc.h` | `PGPROC` / `PROC_HDR` dense arrays、`pgxactoff`、SubXID cache 的结构边界。 |
| 8 | `src/backend/access/transam/varsup.c` | `GetNewTransactionId()` 如何保证新 XID 在释放 `XidGenLock` 前进入 ProcArray。 |

推荐阅读顺序：

```text
先从 snapmgr.c 的 GetTransactionSnapshot() 看什么时候需要 snapshot
  -> 看 snapshot.h 理解 SnapshotData 是什么本地承诺
  -> 进入 procarray.c 的 GetSnapshotData() 主循环
  -> 回到 snapmgr.c 的 XidInMVCCSnapshot() 看 snapshot 如何被消费
  -> 最后读 transam/README 的 interlocking 证明
```

不要从 `HeapTupleSatisfiesMVCC()` 的每个分支开始读。tuple visibility 分支很多，容易把主问题淹没。本节的主线是：

```text
全局 running transactions
  -> 本地 SnapshotData
  -> tuple visibility 的 xid membership check
```

## 4. 关键数据结构与状态

### `SnapshotData`: 一个查询的 MVCC 视图压缩结果

`SnapshotData` 定义在 `src/include/utils/snapshot.h`。普通 MVCC snapshot 的核心字段是：

| 字段 | 语义 |
| --- | --- |
| `xmin` | 所有 `xid < xmin` 对当前 snapshot 来说都不是 in-progress。它们要么已提交，要么已回滚，可以查事务状态或 hint bit。 |
| `xmax` | 所有 `xid >= xmax` 对当前 snapshot 来说都当作 in-progress，也就是当前 snapshot 不可见。 |
| `xip[]` / `xcnt` | `xmin <= xid < xmax` 范围内，仍在运行的 top-level XID 集合。 |
| `subxip[]` / `subxcnt` | 已知仍在运行的 SubXID 集合。正常运行时是 SubXID；recovery snapshot 中也可能存放所有 known assigned XID。 |
| `suboverflowed` | 是否因为 SubXID cache overflow，无法完整记录所有 SubXID。 |
| `takenDuringRecovery` | snapshot 是否在 hot standby recovery 中获取。recovery 下 running XID 来源不是 primary backend 的 `PGPROC`。 |
| `curcid` | 当前事务内 command id 边界，用来区分本事务内早于或晚于当前命令的修改。 |
| `active_count` / `regd_count` | snapshot manager 的生命周期引用计数。 |
| `snapXactCompletionCount` | 创建该 snapshot 时的事务完成计数，用于判断是否可以复用静态 snapshot 内容。 |

`snapshot.h` 对 MVCC snapshot 的语义非常精炼：

```text
一个 MVCC snapshot 可以看到所有早于 xmax 的事务效果，
但要排除 xip[] / subxip[] 中仍被该 snapshot 认为 running 的事务。
```

换成判断规则就是：

```text
xid < xmin:
  不在运行中。

xid >= xmax:
  在当前 snapshot 看来还没完成。

xmin <= xid < xmax:
  查 xip[] / subxip[]；如果命中，就认为仍在运行。
```

这个数据结构的关键不在字段多，而在它把全局可变状态变成了本地稳定状态。`HeapTupleSatisfiesMVCC()` 后续处理一个 heap tuple 时，只需要拿 tuple header 里的 `xmin/xmax` 去问当前 snapshot。

### `CurrentSnapshotData`、`SecondarySnapshotData`、`CatalogSnapshotData`

`snapmgr.c` 里有三组静态 MVCC snapshot：

```c
static SnapshotData CurrentSnapshotData = {SNAPSHOT_MVCC};
static SnapshotData SecondarySnapshotData = {SNAPSHOT_MVCC};
static SnapshotData CatalogSnapshotData = {SNAPSHOT_MVCC};
```

它们解决的是 snapshot 获取本身的内存成本：

```text
GetSnapshotData() 第一次为 snapshot->xip / subxip 分配最大容量；
之后因为 MaxBackends 不会在运行时变化，可以复用数组；
如果需要长生命周期 snapshot，snapmgr.c 再 CopySnapshot() 到 TopTransactionContext。
```

这解释了 `GetSnapshotData()` 中那个看起来有点奇怪的注释：

```text
最好不要传入非静态分配的 SnapshotData。
```

原因不是语义不能支持，而是实现利用了静态 snapshot 复用数组，避免在 `ProcArrayLock` 持有期间或每次查询入口反复 malloc/free。

### `ProcGlobal->xids[]`: snapshot scan 的主输入

上一节已经讲过，`ProcArrayAdd()` 会为进入 ProcArray 的 `PGPROC` 建立 dense array entry：

```text
ProcGlobal->xids[pgxactoff]
ProcGlobal->subxidStates[pgxactoff]
ProcGlobal->statusFlags[pgxactoff]
```

`GetSnapshotData()` 扫描的是 `ProcGlobal->xids[]`，不是先拿 `PGPROC *` 再读 `proc->xid`。这样做的原因很直接：

```text
snapshot scan 是按 backend 数量线性扫描；
dense array 比散落的 PGPROC 字段更容易被 CPU cache 预取；
只在需要读取 SubXID 数组或 databaseId 等字段时才跳回 PGPROC。
```

这也是为什么 `PGPROC->pgxactoff` 不是稳定身份，而是 dense arrays 的当前下标。它只在持有 `ProcArrayLock` 或 `XidGenLock` 的上下文里安全。

### `latestCompletedXid` 与 `xactCompletionCount`

`TransamVariables` 在 `src/include/access/transam.h` 中定义，其中两个字段直接服务 snapshot：

| 字段 | 语义 |
| --- | --- |
| `latestCompletedXid` | 已完成的最大 FullTransactionId。`GetSnapshotData()` 用它加一得到 `xmax`。 |
| `xactCompletionCount` | 自 server 启动以来，带 XID 的 top-level transaction 完成次数。用于判断 snapshot 内容是否可能变化。 |

`ProcArrayEndTransaction()` 在持有 `ProcArrayLock` exclusive 时清除 XID、推进 `latestCompletedXid`，并递增 `xactCompletionCount`。于是 `GetSnapshotData()` 在持有 `ProcArrayLock` shared 时可以依赖：

```text
如果 xactCompletionCount 没变，
则没有带 XID 的事务从 running set 中退出；
当前静态 snapshot 的 xip/subxip 内容可以复用。
```

注意这个复用只覆盖“snapshot 内容”。`curcid`、refcount、`MyProc->xmin` 等仍需要更新。

### `TransactionXmin`、`RecentXmin` 与 `MyProc->xmin`

`GetSnapshotData()` 除了返回 `SnapshotData`，还更新当前 backend 的全局变量：

| 状态 | 语义 |
| --- | --- |
| `MyProc->xmin` | 当前 backend 对其它 backend 发布的 oldest active snapshot xmin。 |
| `TransactionXmin` | 当前事务内最老 snapshot 的 xmin，和 `MyProc->xmin` 对应。 |
| `RecentXmin` | 最近一次 snapshot 的 `xmin`，表示更老的 XID 已知不再 running。 |

这说明 `GetSnapshotData()` 有两个输出方向：

```text
给当前查询:
  返回 SnapshotData。

给其它 backend:
  安装 MyProc->xmin，告诉 VACUUM / horizon 计算我可能还需要哪些旧版本。
```

本节的重点是前者，但不要忘记后者。否则你会误以为 snapshot 只是本地对象，而忽略它对全局 cleanup 边界的反向影响。

### SubXID cache 与 overflow

每个 `PGPROC` 有有限的 SubXID cache。`GetSnapshotData()` 尽量复制运行中事务的 SubXID：

```text
subxidStates[pgxactoff].overflowed == false:
  读取 count，并从 PGPROC->subxids.xids[] 复制。

overflowed == true:
  当前 snapshot 的 suboverflowed 置 true。
```

`suboverflowed` 的含义不是“丢失可见性正确性”。它表示：

```text
本地 snapshot 中没有完整 SubXID 集合；
判断某个 xid 是否属于运行中子事务时，需要通过 pg_subtrans 找 top-level parent，再查 xip[]。
```

这是一种典型 fallback：

```text
常见 case:
  小 SubXID 数，内存数组复制和查找。

极端 case:
  子事务过多，牺牲后续判断成本，保持正确性。
```

## 5. 主流程源码 walkthrough

### 5.1 查询入口：`snapmgr.c` 选择 snapshot 策略

普通查询通常从 `GetTransactionSnapshot()` 进入：

```text
GetTransactionSnapshot()
  -> 如果是逻辑解码 historic snapshot，返回 HistoricSnapshot
  -> 如果是当前事务第一次获取 snapshot
       -> 根据隔离级别决定是否保存 transaction snapshot
       -> 调用 GetSnapshotData(&CurrentSnapshotData)
  -> 如果是 repeatable read / serializable
       -> 返回第一次保存的 CurrentSnapshot
  -> 否则
       -> InvalidateCatalogSnapshot()
       -> 重新调用 GetSnapshotData(&CurrentSnapshotData)
```

这对应 SQL 层的隔离级别差异：

| 隔离级别 | snapshot 生命周期 |
| --- | --- |
| Read Committed | 每条语句可以获取新的 snapshot。 |
| Repeatable Read | 一个事务第一次 snapshot 保存到事务结束。 |
| Serializable | 在 transaction snapshot 外还需要 predicate locking 相关处理。 |

`GetLatestSnapshot()` 用 `SecondarySnapshotData` 获取“当前时刻最新 snapshot”。`GetCatalogSnapshot()` 用 `CatalogSnapshotData` 缓存 catalog scan snapshot，并通过 catalog invalidation 决定是否丢弃。

这里先记住一个实际诊断点：

```text
同一个 SQL 事务内看到几次 GetSnapshotData()，
取决于隔离级别、catalog snapshot invalidation、是否需要 latest snapshot，
不是简单等于“每条 SQL 一次”。
```

### 5.2 第一次进入 `GetSnapshotData()`: 先准备本地数组

`GetSnapshotData()` 开头先确保 `snapshot->xip` 和 `snapshot->subxip` 已分配：

```text
snapshot->xip:
  容量为 GetMaxSnapshotXidCount()，也就是 procArray->maxProcs。

snapshot->subxip:
  容量为 GetMaxSnapshotSubxidCount()，也就是 TOTAL_MAX_CACHED_SUBXIDS。
```

代码注释解释了为什么不按 `numProcs` 精确分配：

```text
numProcs 需要在 ProcArrayLock 下读取；
malloc 最好不要放在锁内；
maxProcs 运行期不变，所以第一次多分配一些，后续复用。
```

这就是第一处 scalability 折中：

```text
用可能过大的内存容量，换取锁内更短路径和后续零分配。
```

如果 malloc 失败，函数直接 `ereport(ERROR)`。这类 ERROR 发生在拿锁之前，因此不会留下 `ProcArrayLock` 泄漏。

### 5.3 持有 `ProcArrayLock` shared

随后进入核心同步窗口：

```text
LWLockAcquire(ProcArrayLock, LW_SHARED);
```

`transam/README` 给出的正确性规则是：

```text
GetSnapshotData() 获取 ProcArrayLock shared；
ProcArrayEndTransaction() 清除 XID 时获取 ProcArrayLock exclusive；
因此 snapshot 构造期间，不允许带 XID 的事务退出 running set。
```

这个规则比理论上最低要求更强，但实现简单，并且支持 `latestCompletedXid + 1` 作为 `xmax`。

注意 shared lock 的性能含义：

```text
多个 backend 可以同时拿 snapshot；
但它们会阻塞正在提交/回滚并需要清除 XID 的 backend；
反过来，大量提交也会让 snapshot 获取等待 exclusive holder 释放。
```

所以 snapshot 扩展性问题常常表现为：

```text
大量 backend 高频短事务提交
  -> ProcArrayLock exclusive 清 XID 压力上升
  -> 查询入口 GetSnapshotData() shared scan 也变多
  -> shared/exclusive 互相影响，延迟尾部变长
```

### 5.4 尝试复用：`GetSnapshotDataReuse()`

拿到锁后，`GetSnapshotData()` 先调用：

```c
if (GetSnapshotDataReuse(snapshot))
{
    LWLockRelease(ProcArrayLock);
    return snapshot;
}
```

复用条件是：

```text
snapshot->snapXactCompletionCount != 0
且
TransamVariables->xactCompletionCount 没变。
```

为什么这足够？

```text
snapshot 内容只依赖带 XID 的 running transactions；
带 XID 的事务退出 running set 时，ProcArrayEndTransaction() 会在 ProcArrayLock exclusive 下递增 xactCompletionCount；
如果计数没变，重建 xip/subxip 也会得到同样内容。
```

复用时仍然要做几件事：

```text
如果 MyProc->xmin 无效:
  重新安装 MyProc->xmin = TransactionXmin = snapshot->xmin。

更新 RecentXmin。
更新 snapshot->curcid。
重置 active_count / regd_count / copied。
```

这说明复用不是“直接返回旧指针”。它复用的是 visibility 内容，仍会刷新生命周期和当前命令边界。

### 5.5 读取 snapshot 上界：`xmax = latestCompletedXid + 1`

不能复用时，主流程读取：

```text
latest_completed = TransamVariables->latestCompletedXid
oldestxid = TransamVariables->oldestXid
curXactCompletionCount = TransamVariables->xactCompletionCount
```

随后：

```text
xmax = latestCompletedXid + 1
xmin = xmax
```

`xmax` 的语义是：

```text
所有 xid >= xmax 的事务，对当前 snapshot 来说都不可见，视为 still running。
```

为什么可以用 `latestCompletedXid + 1`？

因为在 `ProcArrayLock` shared 持有期间：

```text
没有事务能在我们构造 snapshot 时从 ProcArray running set 中清除；
ProcArrayEndTransaction() 推进 latestCompletedXid 需要 exclusive lock；
因此 latestCompletedXid 给出了当前 snapshot 已完成事务的安全上界。
```

如果一个事务在我们读到 `latestCompletedXid` 之后才获得更大的 XID，它必然 `>= xmax`，snapshot 规则会把它当作 running 或未来事务处理，不需要列入 `xip[]`。

### 5.6 把自己的 XID 纳入 `xmin`，但不放入 `xip[]`

主循环前有一段特殊处理：

```text
mypgxactoff = MyProc->pgxactoff
myxid = ProcGlobal->xids[mypgxactoff]

如果 myxid 是 normal 且早于 xmin:
  xmin = myxid
```

但后面的扫描会跳过自己：

```text
if (pgxactoff == mypgxactoff)
    continue;
```

这两个动作看似矛盾，其实是两个不同语义：

```text
自己的 XID 需要影响 xmin:
  因为自己当前事务也可能需要防止 cleanup horizon 前进过头。

自己的 XID 不放入 xip[]:
  因为本事务自己的可见性由 TransactionIdIsCurrentTransactionId()
  和 curcid 规则处理，不通过 XidInMVCCSnapshot() 报告为 running。
```

`XidInMVCCSnapshot()` 的注释也明确说：`GetSnapshotData()` 从不把本 backend 的 top XID 或 SubXID 放进 snapshot。

### 5.7 普通 primary 模式：扫描 dense arrays

非 recovery 下，核心循环是：

```text
for pgxactoff in [0, numProcs)
  xid = UINT32_ACCESS_ONCE(ProcGlobal->xids[pgxactoff])

  if xid invalid:
    continue

  if pgxactoff 是自己:
    continue

  if xid >= xmax:
    continue

  statusFlags = ProcGlobal->statusFlags[pgxactoff]
  if PROC_IN_LOGICAL_DECODING or PROC_IN_VACUUM:
    continue

  xmin = min(xmin, xid)
  snapshot->xip[count++] = xid

  如果 subxid 未 overflow:
    复制该 backend 的 SubXID cache
```

每个 `continue` 都是一个性能和语义边界。

#### 跳过无 XID backend

只读事务或尚未分配 XID 的事务不会进入 snapshot 的 running XID 列表：

```text
没有 XID 的事务不会修改普通 heap tuple；
也不会出现在 tuple header 的 xmin/xmax 中；
因此不需要出现在 xip[]。
```

这也是延迟分配 XID 对 snapshot 扩展性的帮助：大量纯读事务不会膨胀 `xip[]` 内容。

#### 跳过 `xid >= xmax`

这种事务不需要放入 `xip[]`，因为 `XidInMVCCSnapshot()` 的第一层范围判断已经规定：

```text
xid >= snapshot->xmax:
  return true
```

也就是它在当前 snapshot 看来仍在运行。把它写入 `xip[]` 只会增加数组长度，不改变判断结果。

#### 跳过 lazy VACUUM 和 logical decoding

源码注释写明：

```text
Skip over backends doing logical decoding which manages xmin separately
and ones running LAZY VACUUM.
```

这不是说它们“不重要”。而是它们对 visibility / cleanup horizon 的影响不按普通 user transaction 的 `xip[]` 规则表达：

```text
lazy VACUUM:
  不应该让自己的特殊事务状态污染普通 snapshot running set。

logical decoding:
  通过 slot xmin / catalog xmin 等路径管理保留边界。
```

这再次体现一个原则：

```text
raw xid 是否存在，不等于它应以普通事务身份进入 snapshot。
statusFlags 改写字段语义。
```

### 5.8 SubXID 复制与内存屏障

如果还没有 overflow，`GetSnapshotData()` 会处理 SubXID：

```text
if subxidStates[pgxactoff].overflowed:
  suboverflowed = true
else:
  nsubxids = subxidStates[pgxactoff].count
  pg_read_barrier()
  memcpy(snapshot->subxip + subcount,
         proc->subxids.xids,
         nsubxids * sizeof(TransactionId))
```

这里的几个细节都很重要。

第一，读取 `nsubxids` 只读一次。`transam/README` 强调，`GetNewTransactionId()` 发布 XID / SubXID 时允许不拿 `ProcArrayLock`，因此读者必须把 XID 读取当成并发可变值处理，不能多次读取后假设一致。

第二，SubXID 可以并发增加，但不能从 cache 中并发删除。即使复制时另一个 backend 新增了 SubXID，那个新 SubXID 必然晚于当前 `xmax`，不需要被当前 snapshot 记录。

第三，`pg_read_barrier()` 和 `GetNewTransactionId()` 中的写屏障配对，确保看到 count 后能看到对应数组内容。

第四，一旦任何 backend overflow，当前 snapshot 只需要设置一个整体 `suboverflowed`：

```text
snapshot->suboverflowed = true
```

后续 `XidInMVCCSnapshot()` 会走更保守路径。

### 5.9 Hot standby 模式：KnownAssignedXids

如果 `snapshot->takenDuringRecovery` 为 true，`GetSnapshotData()` 不扫描 primary backend 的 running XID。原因在 `procarray.c` 文件头注释里：

```text
hot standby 上的 PGPROC array 表示 standby 上运行查询的进程；
这些进程不是 primary 上正在产生 XID 的事务。
```

standby snapshot 的 running XID 来源是 `KnownAssignedXids`：

```text
subcount = KnownAssignedXidsGetAndSetXmin(snapshot->subxip, &xmin, xmax)
```

并且 recovery 模式下把所有 XID 放进 `subxip[]`：

```text
因为 standby 不一定知道哪些是 top-level XID，哪些是 SubXID；
subxip[] 容量更大；
XidInMVCCSnapshot() 会根据 takenDuringRecovery 使用不同查找规则。
```

这是本节的一个边界提醒：

```text
普通 snapshot 的抽象稳定；
但 running XID 的来源依运行模式不同。
```

本节主线是 primary 上普通 backend 的 ProcArray scan；standby 只作为重要 fallback 入口认识即可。

### 5.10 安装 `MyProc->xmin` 并释放锁

扫描结束后，仍在锁内读取 replication slot xmin：

```text
replication_slot_xmin = procArray->replication_slot_xmin
replication_slot_catalog_xmin = procArray->replication_slot_catalog_xmin
```

然后：

```text
if (!TransactionIdIsValid(MyProc->xmin))
    MyProc->xmin = TransactionXmin = xmin;

LWLockRelease(ProcArrayLock);
```

为什么只在 `MyProc->xmin` 无效时设置？

因为一个事务内可能已经有更老的 snapshot 注册或活跃。`MyProc->xmin` 必须代表当前 backend 仍持有的最老 snapshot，而不是每次获取新 snapshot 都无条件前移。

后续 `SnapshotResetXmin()` 会在 active / registered snapshots 释放时，根据 `RegisteredSnapshots` pairing heap 判断是否能清掉或前推 `MyProc->xmin`。

### 5.11 锁外更新 GlobalVis 近似边界

释放 `ProcArrayLock` 后，`GetSnapshotData()` 更新 `GlobalVisSharedRels`、`GlobalVisCatalogRels`、`GlobalVisDataRels`、`GlobalVisTempRels` 的近似边界。

`procarray.c` 文件头注释说明了为什么这里不做精确 oldest xmin：

```text
GetSnapshotData() 是性能关键路径；
精确 horizon 需要看频繁变化的 xmins；
频繁读取其它 backend 的 xmin 会造成 cacheline ping-pong；
因此 snapshot 构造只维护两个近似边界：
  definitely_needed
  maybe_needed
真正需要精确判断时，由 GlobalVisTest* fallback 到 ComputeXidHorizons()。
```

这是 PostgreSQL v14 之后的重要设计取向：

```text
snapshot 内容只需要 running XID 集合；
cleanup horizon 需要 xmin，但不应把精确 xmin 计算强塞进每次 snapshot hot path。
```

### 5.12 填充 SnapshotData 并返回

最后设置：

```text
RecentXmin = xmin

snapshot->xmin = xmin
snapshot->xmax = xmax
snapshot->xcnt = count
snapshot->subxcnt = subcount
snapshot->suboverflowed = suboverflowed
snapshot->snapXactCompletionCount = curXactCompletionCount
snapshot->curcid = GetCurrentCommandId(false)
snapshot->active_count = 0
snapshot->regd_count = 0
snapshot->copied = false
```

从这里开始，当前查询拿到的是一个本地 snapshot。

后续如果调用者要长期持有：

```text
PushActiveSnapshot()
  -> 必要时 CopySnapshot()
  -> active_count++

RegisterSnapshot()
  -> CopySnapshot()
  -> regd_count++
  -> 加入 RegisteredSnapshots pairing heap
```

如果调用者不注册也不压栈，`HeapTupleSatisfiesMVCC()` 在 assert 构建下会提醒：

```text
snapshot->regd_count > 0 || snapshot->active_count > 0
```

## 6. 生命周期 / ownership / cleanup

### 6.1 snapshot 的创建者

普通 snapshot 的创建者不是 executor，而是 snapshot manager：

```text
GetTransactionSnapshot()
GetLatestSnapshot()
GetCatalogSnapshot()
SetTransactionSnapshot()
```

executor、table access method、index scan 等消费者拿到的是已经构造好的 `Snapshot` 指针。

这是一条非常重要的 ownership 边界：

```text
GetSnapshotData() 构造可见性视图；
snapmgr.c 管 snapshot 生命周期；
heapam_visibility.c 消费 snapshot；
ResourceOwner 兜底释放注册过的 snapshot。
```

不要把 `GetSnapshotData()` 当作通用 allocator。它返回的静态 snapshot 可能被下一次调用覆盖。

### 6.2 active snapshot 与 registered snapshot

snapshot 可以通过两种方式被延长生命周期：

| 方式 | 典型用途 | cleanup |
| --- | --- | --- |
| `PushActiveSnapshot()` | executor 当前正在使用的 snapshot stack | `PopActiveSnapshot()`，subtransaction abort 时 `AtSubAbort_Snapshot()` |
| `RegisterSnapshot()` | portal、cursor、SPI、长期引用 | `UnregisterSnapshot()`，ResourceOwner release fallback |

这两类引用最终都会影响 `MyProc->xmin`：

```text
只要还有活跃或注册 snapshot，
当前 backend 就可能需要旧版本 tuple；
MyProc->xmin 不能随意清掉。
```

`RegisteredSnapshots` 是按 `xmin` 排序的 pairing heap。这样当没有 active snapshot 时，`SnapshotResetXmin()` 可以快速找到最老 registered snapshot。

### 6.3 清理路径

典型清理路径：

```text
PopActiveSnapshot()
  -> active_count--
  -> 如果没有 active / registered 引用，FreeSnapshot()
  -> SnapshotResetXmin()

UnregisterSnapshot()
  -> ResourceOwnerForget()
  -> regd_count--
  -> 从 RegisteredSnapshots 移除
  -> 可能 FreeSnapshot()
  -> SnapshotResetXmin()

AtEOXact_Snapshot()
  -> 移除 FirstXactSnapshot
  -> 清理 exported snapshots
  -> InvalidateCatalogSnapshot()
  -> reset CurrentSnapshot / SecondarySnapshot / FirstSnapshotSet
```

`AtEOXact_Snapshot()` 的注释还强调：

```text
正常 commit 处理中，ProcArrayEndTransaction() 已经重置 MyProc->xmin；
因此 AtEOXact_Snapshot() 通常不需要再处理 xmin。
```

这串顺序连接了上一节事务结束清理：

```text
transaction end 清除 ProcArray transaction state
  -> snapshot manager 清理本 backend snapshot 引用
  -> TopTransactionContext 回收 copied snapshot 内存
```

### 6.4 catalog snapshot 的特殊生命周期

`GetCatalogSnapshot()` 会缓存 catalog snapshot，但 catalog invalidation 会让它失效：

```text
InvalidateCatalogSnapshot()
  -> 从 RegisteredSnapshots 移除 CatalogSnapshot
  -> CatalogSnapshot = NULL
  -> SnapshotResetXmin()
```

`InvalidateCatalogSnapshotConditionally()` 在即将等待客户端输入时，如果 catalog snapshot 是唯一 snapshot，会主动丢弃，避免空闲连接因为 catalog snapshot 让 xmin horizon 长时间停住。

这解释了一类线上现象：

```text
不是所有 idle backend 都会持有 xmin；
持有 snapshot 的 idle in transaction、open cursor、exported snapshot、
或未释放 catalog snapshot 才会影响 cleanup horizon。
```

## 7. 正确性机制层次

### 7.1 running set 不能在 snapshot 构造中消失

`transam/README` 的核心例子是：

```text
B 修改一行，A 被 B 阻塞；
B 正在 commit；
C 正在获取 snapshot；
B 释放锁后 A 继续并 commit。
```

如果 C 的 snapshot 看到 B 仍在运行，却没看到 A 仍在运行，就可能看到不一致的 tuple version 组合。

PostgreSQL 采用的规则是：

```text
snapshot-taking 与带 XID 的 transaction end 严格互斥。
```

实现：

```text
GetSnapshotData():
  ProcArrayLock shared

ProcArrayEndTransaction():
  ProcArrayLock exclusive while clearing ProcGlobal->xids[]
```

这个规则保证：

```text
从读取 latestCompletedXid 到填完 xip[]，
没有事务能从 running set 退出。
```

### 7.2 XID 进入 ProcArray 的 ordering

事务开始本身不需要 XID。但一旦分配 XID，`GetNewTransactionId()` 必须在释放 `XidGenLock` 前发布到 shared ProcArray。

否则会出现：

```text
事务 T1 分配较小 XID，但尚未发布；
事务 T2 分配较大 XID，提交并推进 latestCompletedXid；
snapshot 看到 latestCompletedXid 已越过 T1，却在 ProcArray 看不到 T1；
于是 T1 被错误地认为完成。
```

所以 snapshot correctness 同时依赖两个方向：

```text
XID enter running set:
  GetNewTransactionId() + XidGenLock ordering。

XID leave running set:
  ProcArrayEndTransaction() + ProcArrayLock exclusive。
```

### 7.3 本地 snapshot 消费规则

`XidInMVCCSnapshot()` 先做范围判断：

```text
if xid < snapshot->xmin:
  return false

if xid >= snapshot->xmax:
  return true
```

只有中间范围才查数组。

正常模式下：

```text
if !suboverflowed:
  先查 subxip[]
else:
  xid = SubTransGetTopmostTransaction(xid)
  再查 xip[]

查 xip[]
```

recovery 模式下：

```text
所有 known assigned XID 主要放在 subxip[]；
按 takenDuringRecovery 使用另一套查找规则。
```

这也是为什么 `SnapshotData` 的 `xmin/xmax` 不是辅助字段，而是性能关键字段。大多数 tuple 的 XID 可以被范围判断快速排除，不需要扫描 `xip[]`。

### 7.4 tuple visibility 避免反复查 ProcArray

`HeapTupleSatisfiesMVCC()` 的注释说明了一个关键选择：

```text
如果 tuple 的插入或删除事务在当前 snapshot 看来仍在运行，
即使它实际上已经提交或回滚，也不立即查最新全局事务状态来设置 hint bit。
```

原因是：

```text
每个 tuple 都查 TransactionIdIsInProgress 或 ProcArray 会产生高 contention；
而对当前 snapshot 来说，结果不会改变；
hint bit 可以留给未来更新的 snapshot 来设置。
```

这正是 snapshot 抽象的意义：

```text
一次昂贵但受控的全局读取，
换取后续大量 tuple visibility 判断的本地稳定读取。
```

## 8. 错误路径 / 异常路径 / fallback

### 8.1 内存分配失败

`GetSnapshotData()` 第一次使用某个静态 snapshot 时会 `malloc()` `xip` 和 `subxip` 数组。失败时：

```text
ereport(ERROR, "out of memory")
```

这发生在获取 `ProcArrayLock` 之前，因此错误路径简单，不需要释放锁。错误会交给 PostgreSQL 的 ERROR / ResourceOwner / MemoryContext 机制处理。

### 8.2 SubXID overflow

SubXID cache 容量有限。overflow 后，snapshot 不再拥有完整 SubXID 列表：

```text
snapshot->suboverflowed = true
```

后续 `XidInMVCCSnapshot()` 必须：

```text
SubTransGetTopmostTransaction(xid)
  -> 找到 top-level XID
  -> 查 xip[]
```

成本变化：

```text
未 overflow:
  内存数组查找，通常很快。

overflow:
  可能访问 pg_subtrans SLRU，带来额外锁、缓存和 I/O 风险。
```

这就是为什么大量 savepoint、PL/pgSQL exception、深层子事务会让看似无关的查询变慢：它不只影响产生子事务的 backend，也会污染其它 backend 的 snapshot 判断路径。

### 8.3 Hot standby snapshot

standby 上的 snapshot 使用 `KnownAssignedXids`。如果 KnownAssignedXids overflow 或 `lastOverflowedXid` 影响当前 xmin：

```text
suboverflowed = true
```

这里的 fallback 语义和普通 SubXID overflow 类似：

```text
不能完整表达所有 running xid；
后续判断必须更保守。
```

### 8.4 imported / exported snapshot

`SetTransactionSnapshot()` 导入 snapshot 时，即使最终要使用外部 snapshot，也会先调用：

```text
CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData);
```

原因有两个：

```text
确保 CurrentSnapshotData 的 XID 数组已经分配；
更新 GlobalVis* 状态。
```

随后它会调用：

```text
ProcArrayInstallImportedXmin()
或
ProcArrayInstallRestoredXmin()
```

这一步必须验证 source transaction 仍在运行，并原子安装 `MyProc->xmin`。否则导入一个已经失效的旧 snapshot 会让系统 xmin horizon 倒退。

### 8.5 parallel mode 限制

`GetTransactionSnapshot()` 和 `GetLatestSnapshot()` 在 parallel mode 下有保护：

```text
cannot take query snapshot during a parallel operation
cannot update SecondarySnapshot during a parallel operation
```

并行查询会在 worker 启动时共享 snapshot。执行过程中随意更新 snapshot 或 command id，可能造成 leader 和 worker 可见性不一致。

### 8.6 snapshot 泄漏

`AtEOXact_Snapshot()` 在 commit 时会检查：

```text
registered snapshots seem to remain after cleanup
snapshot %p still active
```

这些 warning 不是普通用户常见错误，但对扩展开发、executor 改造、portal 生命周期问题非常有用。它们说明某处没有正确 `PopActiveSnapshot()` 或 `UnregisterSnapshot()`。

## 9. 成本、资源与跨模块传播

### 9.1 成本模型

一次普通 `GetSnapshotData()` 的主要成本可以粗略拆成：

```text
锁成本:
  获取 ProcArrayLock shared，可能等待正在提交/回滚的 exclusive holder。

扫描成本:
  O(procArray->numProcs)，至少读取 dense xids[]。

写入成本:
  O(active top-level XID count)，把符合条件的 xid 写入 xip[]。

SubXID 成本:
  O(cached SubXID count)，直到遇到 overflow。

缓存一致性成本:
  读取其它 backend 频繁变化字段可能导致 cacheline 流量。
```

`MaxBackends` 影响的是容量和最坏扫描边界；`numProcs` 影响实际扫描长度；真正写入 `xip[]` 的数量取决于当前有多少 backend 拥有 normal XID 且小于 `xmax`。

因此：

```text
1000 个连接但多数 idle / read-only / no XID:
  仍有 scan 成本，但 xip[] 写入少。

1000 个连接且多数活跃写事务:
  scan 成本、xip[] 写入、commit exclusive lock 竞争都会上升。
```

### 9.2 为什么 dense arrays 重要

如果 `GetSnapshotData()` 每次都遍历 `PGPROC` 并读取散落字段：

```text
PGPROC 很大；
字段分散在不同 cacheline；
每个 backend 的 PGPROC 还承载 lock、latch、wait event、replication 等状态。
```

dense arrays 把 snapshot scan 最常用字段压缩到连续内存：

```text
ProcGlobal->xids[]
ProcGlobal->subxidStates[]
ProcGlobal->statusFlags[]
```

这不是改变算法复杂度，而是改善常数和 cache behavior。对 hot path 来说，常数就是系统行为。

### 9.3 为什么不扫描精确 `xmin`

`transam/README` 明确说：

```text
GetSnapshotData() 性能关键；
snapshot 内容只依赖其它 backend 的 xid，不依赖它们的 xmin；
backend 的 xmin 比 xid 变化更频繁；
在 GetSnapshotData() 中读取所有 xmins 会造成不必要的 cacheline ping-pong。
```

所以现在的设计是：

```text
GetSnapshotData():
  构造 snapshot 必需内容，并更新 GlobalVis 近似边界。

ComputeXidHorizons():
  在真正需要精确 cleanup horizon 时扫描 xid / xmin / statusFlags。
```

这是第 18 节会继续展开的问题。

### 9.4 与事务提交路径的传播

`ProcArrayEndTransaction()` 对 snapshot 路径有两个影响：

```text
清除 ProcGlobal->xids[]:
  让后续 snapshot 不再把该事务视为 running。

递增 xactCompletionCount:
  让已有静态 snapshot 下次 GetSnapshotData() 不能错误复用。
```

为了降低多个 backend 同时提交时的竞争，还有：

```text
ProcArrayGroupClearXid()
```

它让一个 leader 持有 `ProcArrayLock` exclusive，一次清掉多个 backend 的 XID。这减少 context switch 和锁竞争，但没有改变正确性边界：

```text
running set 的退出仍在 ProcArrayLock exclusive 下发生。
```

### 9.5 与 tuple visibility 的传播

`HeapTupleSatisfiesMVCC()` 的主要输入来自：

```text
tuple header:
  xmin / xmax / infomask / command id

snapshot:
  xmin / xmax / xip / subxip / curcid

pg_xact / hint bits:
  事务最终提交或回滚状态
```

`GetSnapshotData()` 不直接判断 tuple。它只决定“哪些 XID 在当前视图中仍然 running”。真正可见性还要结合 tuple header 和 commit status。

因此不要把 `xip[]` 理解为“不可见 tuple 列表”。它只是 running transaction membership set。

### 9.6 与 VACUUM / pruning 的传播

`GetSnapshotData()` 安装 `MyProc->xmin`，这会影响其它 backend 的 cleanup horizon。长事务、长 cursor、导出 snapshot 会使 `MyProc->xmin` 长时间保持较旧值，进而影响：

```text
VACUUM 可移除哪些 dead tuple；
HOT pruning 能否剪掉旧版本；
pg_subtrans / pg_xact 截断边界；
replication feedback 和 logical decoding 保留边界。
```

但 `GetSnapshotData()` 不在 hot path 中做完整 VACUUM horizon 计算。它只维护近似 GlobalVis 边界，把精确计算留给更低频、更合适的路径。

## 10. 观测与诊断入口

### 10.1 SQL 观察 snapshot 边界

在一个会话中：

```sql
BEGIN;
SELECT pg_current_snapshot();
SELECT txid_current_if_assigned();
```

在另一个会话中做写事务：

```sql
BEGIN;
CREATE TABLE IF NOT EXISTS snap_demo(id int);
INSERT INTO snap_demo VALUES (1);
SELECT txid_current();
```

回到第一个会话：

```sql
SELECT pg_current_snapshot();
```

可观察点：

```text
Read Committed 下，不同语句可能获取不同 snapshot；
Repeatable Read 下，第一次 transaction snapshot 会被复用；
只读事务在未分配 XID 时，txid_current_if_assigned() 可以返回 NULL。
```

注意 SQL 层展示的 snapshot 是序列化后的视图，不会直接显示所有内部字段，比如 `snapXactCompletionCount`、`takenDuringRecovery`、active/refcount。

### 10.2 gdb 断点

适合本节的断点：

```gdb
break GetTransactionSnapshot
break GetSnapshotData
break GetSnapshotDataReuse
break XidInMVCCSnapshot
break ProcArrayEndTransaction
break ProcArrayGroupClearXid
```

在 `GetSnapshotData()` 内可观察：

```gdb
print procArray->numProcs
print TransamVariables->latestCompletedXid
print TransamVariables->xactCompletionCount
print MyProc->xmin
print snapshot->xmin
print snapshot->xmax
print snapshot->xcnt
print snapshot->subxcnt
print snapshot->suboverflowed
```

如果要看扫描效果：

```gdb
print ProcGlobal->xids[0]@procArray->numProcs
print ProcGlobal->statusFlags[0]@procArray->numProcs
```

调试时要意识到：

```text
GetSnapshotData() 是高频函数；
断在这里会显著改变并发 timing。
```

### 10.3 观察锁等待

如果怀疑 snapshot 和提交路径围绕 `ProcArrayLock` 竞争，可以从几个方向观察：

```text
pg_stat_activity:
  wait_event_type / wait_event 是否出现 LWLock 相关等待。

perf:
  GetSnapshotData、ProcArrayEndTransaction、LWLockAcquire、XidInMVCCSnapshot 是否出现在热点。

gdb:
  多 backend 是否卡在 ProcArrayLock shared / exclusive。
```

`ProcArrayLock` 的具体 wait event 名称会随版本和构建细节变化，诊断时不要只靠一个固定字符串。更稳的办法是结合 stack、perf symbol 和 workload 时序。

### 10.4 观察 SubXID overflow 影响

可以构造大量 savepoint 或 PL/pgSQL exception：

```sql
BEGIN;
-- 在客户端循环执行大量 SAVEPOINT / INSERT / RELEASE 或 ROLLBACK TO
SELECT txid_current();
```

另一个会话反复执行查询并用 perf / gdb 观察：

```text
GetSnapshotData()
XidInMVCCSnapshot()
SubTransGetTopmostTransaction()
```

如果看到 `SubTransGetTopmostTransaction()` 变热，说明 snapshot 的 SubXID fast path 已经失效，后续 visibility 判断需要查 `pg_subtrans`。

### 10.5 观察长事务对 xmin 的影响

会话 A：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT pg_current_snapshot();
-- 保持事务不结束
```

会话 B：

```sql
SELECT pid, backend_xmin, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL;
```

现象：

```text
会话 A 的 backend_xmin 会保持在较旧位置；
VACUUM 和 pruning 相关边界可能被它拖住。
```

回到源码解释：

```text
GetSnapshotData() 安装 MyProc->xmin；
Repeatable Read 的 FirstXactSnapshot 注册到事务结束；
AtEOXact_Snapshot() / ProcArrayEndTransaction() 才清理。
```

## 11. 常见误区

### 误区 1：snapshot 就是当前所有连接列表

不是。snapshot 只关心影响 MVCC 可见性的 running XID 集合。

```text
无 XID backend:
  可能在 ProcArray 中，但不进入 xip[]。

idle session:
  有 PGPROC，但未必有 snapshot，也未必有 backend_xmin。

lazy VACUUM / logical decoding:
  即使有 XID，也可能不按普通事务方式进入 xip[]。
```

### 误区 2：`xmin` 越新越好，所以每次 snapshot 都可以前推 `MyProc->xmin`

错误。一个 backend 可能同时持有多个 snapshot。`MyProc->xmin` 必须代表仍在使用的最老 snapshot。

这就是 `RegisteredSnapshots` pairing heap 和 `ActiveSnapshot` stack 存在的原因。

### 误区 3：`xmax` 等于当前 nextXid

普通 `GetSnapshotData()` 使用的是：

```text
xmax = latestCompletedXid + 1
```

它不是简单读取 `nextXid`。这个选择依赖 transaction end 与 snapshot-taking 的 interlocking。

### 误区 4：SubXID overflow 只是产生子事务的那个 backend 的问题

不是。一个 backend overflow 后，其它 backend 的 snapshot 可能设置 `suboverflowed`，后续 visibility 判断需要 `pg_subtrans` fallback。

影响会从一个事务传播到其它查询的 tuple visibility 路径。

### 误区 5：`GetSnapshotData()` 应该计算最精确的全局 xmin

这在正确性上可行，但在 hot path 上不划算。当前源码明确选择：

```text
snapshot 内容依赖 running XID；
精确 cleanup horizon 交给 ComputeXidHorizons()；
GetSnapshotData() 只维护 GlobalVis 近似边界。
```

### 误区 6：`xactCompletionCount` 没变表示系统没有事务活动

不准确。它只表示：

```text
没有带 XID 的 top-level transaction 完成。
```

无 XID 的只读事务可以开始和结束；当前 backend 的 `curcid` 也可能变化；snapshot lifecycle refcount 也要重置。

### 误区 7：tuple visibility 会实时反映最新 commit 状态

MVCC snapshot 的目标恰恰是“不实时变化”。如果一个事务在 snapshot 看来 running，即使它物理上刚刚 commit，当前 snapshot 仍按 running 处理。后续更新的 snapshot 才会看到它完成。

## 12. 课堂实验

### 实验 1：Read Committed 与 Repeatable Read 的 snapshot 差异

目标：观察 `GetTransactionSnapshot()` 何时复用 snapshot。

步骤：

```sql
-- 会话 A
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT pg_current_snapshot();

-- 会话 B
BEGIN;
INSERT INTO snap_demo VALUES (100);
COMMIT;

-- 会话 A
SELECT pg_current_snapshot();
COMMIT;
```

然后换成：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT pg_current_snapshot();
-- 会话 B 提交写入
SELECT pg_current_snapshot();
COMMIT;
```

源码对应：

```text
Read Committed:
  FirstSnapshotSet 后仍可再次 GetSnapshotData(&CurrentSnapshotData)。

Repeatable Read:
  IsolationUsesXactSnapshot() 返回 true，复用 FirstXactSnapshot。
```

### 实验 2：无 XID 事务不进入 `xip[]`

目标：理解事务开始不等于 XID 分配。

步骤：

```sql
BEGIN;
SELECT txid_current_if_assigned();
SELECT pg_current_snapshot();
```

然后：

```sql
CREATE TEMP TABLE xid_force(id int);
INSERT INTO xid_force VALUES (1);
SELECT txid_current_if_assigned();
SELECT pg_current_snapshot();
ROLLBACK;
```

预期：

```text
只读阶段可能没有真实 XID；
写入后分配 XID，可能影响其它 backend 的 snapshot scan。
```

源码对应：

```text
GetNewTransactionId()
  -> MyProc->xid
  -> ProcGlobal->xids[MyProc->pgxactoff]

GetSnapshotData()
  -> 扫描 ProcGlobal->xids[]
```

### 实验 3：长事务持有 `backend_xmin`

目标：观察 snapshot 如何反向影响 cleanup horizon。

步骤：

```sql
-- 会话 A
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT pg_current_snapshot();

-- 会话 B
SELECT pid, backend_xmin, state
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL;
```

源码对应：

```text
GetSnapshotData()
  -> MyProc->xmin = TransactionXmin = xmin

AtEOXact_Snapshot() / ProcArrayEndTransaction()
  -> 清理 snapshot 与 xmin
```

思考：

```text
为什么单纯 idle session 通常不显示 backend_xmin？
为什么 idle in transaction 更危险？
```

### 实验 4：SubXID overflow 诊断

目标：观察大量子事务对 snapshot 消费路径的影响。

步骤建议：

```text
写一个脚本在一个事务中创建大量 savepoint；
另一个会话持续执行需要 MVCC 判断的查询；
用 perf top 或 gdb 观察 XidInMVCCSnapshot() 和 SubTransGetTopmostTransaction()。
```

源码对应：

```text
GetSnapshotData()
  -> subxidStates[pgxactoff].overflowed
  -> snapshot->suboverflowed = true

XidInMVCCSnapshot()
  -> SubTransGetTopmostTransaction()
```

### 实验 5：观察静态 snapshot 数组复用

目标：理解 `snapXactCompletionCount`。

gdb 步骤：

```gdb
break GetSnapshotDataReuse
commands
  silent
  print snapshot->snapXactCompletionCount
  print TransamVariables->xactCompletionCount
  continue
end
```

在低写入压力下反复执行简单 SELECT，观察复用条件；再在另一个会话持续提交写事务，观察计数变化后复用失败。

源码对应：

```text
ProcArrayEndTransaction()
  -> TransamVariables->xactCompletionCount++

GetSnapshotDataReuse()
  -> 比较 snapshot->snapXactCompletionCount
```

### 实验 6：catalog snapshot invalidation

目标：理解 catalog snapshot 不是普通 query snapshot 的简单副本。

步骤：

```sql
-- 会话 A
BEGIN;
SELECT * FROM pg_class WHERE relname = 'snap_demo';

-- 会话 B
CREATE TABLE snap_demo_catalog_inval(id int);

-- 会话 A
SELECT * FROM pg_class WHERE relname = 'snap_demo_catalog_inval';
COMMIT;
```

结合 gdb：

```gdb
break GetCatalogSnapshot
break InvalidateCatalogSnapshot
```

观察 catalog snapshot 何时被缓存、何时被 invalidation 丢弃。

## 13. 讨论题

1. 为什么 `GetSnapshotData()` 可以用 `latestCompletedXid + 1` 作为 `xmax`，而不直接读取 `nextXid`？

2. 如果 `GetSnapshotData()` 不持有 `ProcArrayLock`，只用原子读取 `xids[]`，会破坏哪个 commit-ordering 不变量？

3. 为什么 snapshot 不把当前 backend 自己的 XID 放进 `xip[]`，但又要用自己的 XID 更新 `xmin`？

4. `xactCompletionCount` 能复用 snapshot 内容，为什么仍然需要更新 `curcid` 和 refcount 字段？

5. 如果一个 workload 有大量连接但大部分是无 XID 只读事务，`GetSnapshotData()` 的成本主要来自哪里？如果大部分是写事务，又会增加哪些成本？

6. 为什么 SubXID overflow 会影响其它 backend 的查询，而不只是影响产生大量 savepoint 的事务本身？

7. `GetSnapshotData()` 不精确扫描所有 `xmin`，为什么仍然不会破坏 VACUUM correctness？

8. `HeapTupleSatisfiesMVCC()` 为什么不在发现事务“按当前 snapshot running”时再去查一次最新全局事务状态？

9. catalog snapshot 为什么可以缓存？又为什么有些 catalog 访问必须强制刷新？

10. 如果让每个 backend 维护自己的 snapshot cache，不扫描 ProcArray，哪些跨事务可见性规则会丢失？

## 14. 本节小结

本节的主问题是：

```text
为什么普通 snapshot 要扫描全局 backend 状态，
以及 GetSnapshotData() 如何让这件事在 MaxBackends 增大时仍尽量可控。
```

核心答案：

```text
MVCC visibility 需要一个稳定的 running transaction 集合；
这个集合只能从 ProcArray 发布的全局事务状态中获得；
GetSnapshotData() 在 ProcArrayLock shared 下把全局状态压缩成本地 SnapshotData；
后续 tuple visibility 使用本地 snapshot，避免每个 tuple 访问高竞争 shared state。
```

本节最重要的状态转换是：

```text
ProcGlobal->xids[] / subxidStates[] / statusFlags[]
  -> GetSnapshotData()
  -> SnapshotData xmin / xmax / xip / subxip / suboverflowed
  -> XidInMVCCSnapshot()
  -> HeapTupleSatisfiesMVCC()
```

本节最重要的 correctness 边界是：

```text
GetSnapshotData() 持 ProcArrayLock shared；
带 XID 的事务结束清 XID 时持 ProcArrayLock exclusive；
因此 snapshot 构造期间，running set 不会有事务消失。
```

本节最重要的 scalability 边界是：

```text
GetSnapshotData() 仍然是 O(numProcs) scan；
但它用 dense arrays、静态数组复用、xactCompletionCount、SubXID cache、
跳过无关 XID、GlobalVis 近似边界来降低热路径常数和避免额外全局计算。
```

最后沉淀成一个可迁移规律：

```text
当系统必须从全局 mutable state 中得到一致视图时，
不要把每次局部判断都绑定到全局状态；
更可扩展的做法是用短同步窗口捕获必要事实，
把后续高频判断转化为本地稳定结构上的 membership / range check，
并为极端 case 设计明确的保守 fallback。
```

下一节将沿着 `GetSnapshotData()` 留下的 `xmin` 和 GlobalVis 线索继续展开：系统如何从各 backend 的 `xmin`、replication slot、prepared transaction 和 vacuum flags 推导 cleanup horizon，决定哪些 tuple、clog、subtrans 还不能回收。
