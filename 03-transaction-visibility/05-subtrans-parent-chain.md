# PostgreSQL Subtrans 父事务链

## 课程定位

前置知识：

- 读者已经知道 PostgreSQL tuple header 会记录 `xmin` 和 `xmax`。
- 读者已经知道 `pg_xact`/CLOG 记录事务最终状态。
- 读者已经知道 snapshot 用 `xmin`、`xmax`、`xip` 描述可见性边界。
- 读者不需要先掌握全部 savepoint 实现。

本节唯一主问题：

当 tuple header 里保存的是一个子事务 XID 时，PostgreSQL 如何把这个子事务追溯到顶层事务，

并用顶层事务的最终命运决定 tuple version 是否可见？

本节核心矛盾：

子事务要能局部提交、局部回滚，但 MVCC 读者需要一个全局稳定的事务结果。

这个矛盾不能靠“把子事务当普通事务提交”解决。`RELEASE SAVEPOINT` 只表示子事务的效果并入父事务。

如果父事务最终 abort，已经 release 的子事务写入仍必须整体消失。

本节围绕一个状态展开：

`pg_subtrans` 中的 `subxid -> parent xid` 父事务链。

学完后应能独立判断：

- 一个 subxid 什么时候需要查 `pg_subtrans`。
- `pg_subtrans` 保存的是 parent，不是 commit result。
- `SUB_COMMITTED` 为什么不是普通 committed。
- `PGPROC` subxid cache overflow 后为什么 visibility 会变慢。
- crash、abort、checkpoint、truncate 对父链语义的边界。

本课基于本地源码：

- 源码目录：`/home/highgo/postgres`
- 分支：`master`
- 提交：`bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`

本节不是 `06-subxid-overflow-visibility-cost.md`。overflow 会出现，

但这里只解释父链为什么存在以及如何保证正确性。下一节再专门展开 overflow 的成本模型。

## 1. 本节在总主线中的位置

前几节已经建立了三个事实。第一，事务只有在需要写入可见性相关状态时才分配 XID。

第二，`pg_xact` 是事务结果判定的中心。第三，abort 路径必须让未完成或失败的写入不能被误认为 committed。

本节把问题推进到子事务。用户看到的是：

- `SAVEPOINT s`
- `ROLLBACK TO s`
- `RELEASE SAVEPOINT s`
- PL/pgSQL exception block
- 内核内部的 `BeginInternalSubTransaction()`

内核看到的是：

- 一棵事务状态栈。
- 多个可能写入 tuple header 的 XID。
- 一个顶层事务最终 commit 或 abort 的结果。

核心 runtime 现象：

一个子事务插入 tuple 后被 `RELEASE SAVEPOINT`，另一个 backend 不能仅凭这个 subxid 已经“子提交”就看到 tuple。

只有顶层事务 commit 后，这个 tuple 才能对符合 snapshot 边界的读者可见。

如果顶层事务 abort，这个 tuple 必须被解释为 aborted。

这就是 `pg_subtrans` 的位置：

它不回答“这个 XID committed 了吗”。它回答“这个 subxid 应该向哪个 parent 继续追问”。

因此本节的阅读主线是：

```text
tuple header 中出现 subxid
  -> snapshot/ProcArray 无法直接完整判断
  -> pg_subtrans 找 parent/top xid
  -> pg_xact 判断最终状态
  -> heap visibility 返回可见或不可见
```

注意这里有两个不同问题。问题一：

这个子事务属于哪个顶层事务？问题二：

这个顶层事务最终 committed 还是 aborted？`pg_subtrans` 只解决问题一。

`pg_xact` 解决问题二。把这两个问题混在一起，

是阅读 subtrans 代码最常见的误区。

## 2. 核心矛盾与一句话运行模型

核心矛盾可以压缩成一句话：

子事务的局部边界需要被保留给 rollback，但 MVCC 的外部可见性必须服从顶层事务的最终结果。

为什么不能把子事务 commit 写成普通 committed？因为子事务 commit 不是 durability 和 visibility 的终点。

考虑这个过程：

```sql
BEGIN;
SAVEPOINT s;
INSERT INTO t VALUES (1);
RELEASE SAVEPOINT s;
ROLLBACK;
```

`RELEASE SAVEPOINT s` 后，子事务的修改已经并入父事务。

但顶层 `ROLLBACK` 后，这个插入不能存在。

如果子事务在 `RELEASE` 时被写成普通 committed，其他 backend 后续看到 tuple 的 `xmin = subxid` 时，

就可能把它误判成可见。PostgreSQL 的做法是：

- 子事务分配 XID 时记录父链。
- 子事务 release 时不把 CLOG 写成最终 committed。
- 顶层 commit 时一次性写整棵已提交子事务树的最终状态。
- 跨 CLOG page 时先写 `SUB_COMMITTED` 作为过渡状态。
- 读者遇到 `SUB_COMMITTED` 或 snapshot overflow 时沿父链追到 top xid。
- 最终仍由 `pg_xact` 和 snapshot 决定 tuple 可见性。

一句话运行模型：

`pg_subtrans` 是一张临时父指针表，把 subxid 映射到 parent xid；

`pg_xact` 是结果表，把 xid 映射到 committed/aborted/subcommitted；

visibility 在信息不完整时先追父链，再问结果表。这句话里有三个边界。

第一，父链是临时的。`subtrans.c` 明确说它不需要 crash 后保持完整历史。

第二，父链是单向的。child 能找 parent，

parent 不能从 `pg_subtrans` 枚举 children。第三，父链不是 WAL 语义。

`pg_subtrans` 没有自己的 WAL redo。恢复时通过 startup 初始化、xact assignment WAL、prepared transaction 等路径补足当前仍需要的父链信息。

这也是为什么它适合做 fallback lookup，不适合做最终事务结果。

本节的核心不变量：

- parent XID 必须早于 child XID 分配。
- subxid 进入 tuple/WAL 等可被外部追问的位置前，父链必须已经可补足。
- 一个 valid parent 不能被另一个 valid parent 覆盖。
- `SUB_COMMITTED` 必须继续追 parent，不能直接等价于 committed。
- 只要 XID 早于 `TransactionXmin`，系统可以不再保证完整父链。

这些不变量共同保证：

读者即使只拿到 tuple header 里的 subxid，也能在必要时回到顶层事务语义。

## 3. 核心文件分工与阅读顺序

本节重点源码文件如下。

| 文件 | 本节阅读目的 |
| --- | --- |
| `src/backend/access/transam/subtrans.c` | `pg_subtrans` SLRU 的 parent 写入、读取、追顶层、启动、checkpoint、truncate |
| `src/include/access/subtrans.h` | 对外暴露的最小接口，说明模块只提供 parent/topmost 查询 |
| `src/backend/access/transam/xact.c` | 子事务状态栈、XID 分配、subcommit 合并、subabort cleanup、顶层 commit/abort |
| `src/backend/access/transam/clog.c` | CLOG status tree 更新，尤其 `SUB_COMMITTED` 过渡状态 |
| `src/backend/access/transam/transam.c` | `TransactionIdDidCommit()` 和 `TransactionIdDidAbort()` 如何遇到 `SUB_COMMITTED` |
| `src/backend/storage/ipc/procarray.c` | running XID 判定、snapshot 采集、subxid cache overflow 后回查父链 |
| `src/include/access/xact.h` | WAL record 中 subxact assignment、subxact callback、xact record 数据结构边界 |

辅助文件也会短暂出现。它们不是本节主线，

但能帮助定位 runtime 入口：

| 文件 | 只读原因 |
| --- | --- |
| `src/backend/access/transam/varsup.c` | `GetNewTransactionId()` 把 XID 放入 `PGPROC` subxid cache |
| `src/include/storage/proc.h` | `PGPROC_MAX_CACHED_SUBXIDS`、`XidCacheStatus`、`PGPROC->subxids` |
| `src/backend/utils/time/snapmgr.c` | `XidInMVCCSnapshot()` 是 heap MVCC 可见性调用的父链入口 |
| `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesMVCC()` 如何调用 snapshot 判定 |

推荐阅读顺序不是按文件名排序。先读状态边界：

1. `src/include/access/subtrans.h`
2. `src/backend/access/transam/subtrans.c`
3. `src/include/storage/proc.h` 中 `XidCacheStatus`
4. `src/include/utils/snapshot.h` 中 `SnapshotData`

再读创建与发布：

1. `xact.c` 的 `AssignTransactionId()`
2. `varsup.c` 的 `GetNewTransactionId()`
3. `subtrans.c` 的 `SubTransSetParent()`

再读正常结束：

1. `xact.c` 的 `CommitSubTransaction()`
2. `xact.c` 的 `AtSubCommit_childXids()`
3. `xact.c` 的 `RecordTransactionCommit()`
4. `transam.c` 的 `TransactionIdCommitTree()`
5. `clog.c` 的 `TransactionIdSetTreeStatus()`

再读读者路径：

1. `heapam_visibility.c` 的 `HeapTupleSatisfiesMVCC()`
2. `snapmgr.c` 的 `XidInMVCCSnapshot()`
3. `procarray.c` 的 `TransactionIdIsInProgress()`
4. `transam.c` 的 `TransactionIdDidCommit()`
5. `subtrans.c` 的 `SubTransGetTopmostTransaction()`

最后读 cleanup 和 fallback：

1. `xact.c` 的 `AbortSubTransaction()`
2. `xact.c` 的 `CleanupSubTransaction()`
3. `procarray.c` 的 `XidCacheRemoveRunningXids()`
4. `subtrans.c` 的 `StartupSUBTRANS()`
5. `subtrans.c` 的 `TruncateSUBTRANS()`

读源码时要刻意区分四条链。第一条是 transaction state stack。

它是 backend-local。第二条是 `PGPROC` 的 running XID cache。

它是 shared memory。第三条是 `pg_subtrans` 的 parent chain。

它是 SLRU 管理的共享状态和文件状态。第四条是 `pg_xact` 的 result status。

它才是最终事务结果。源码有历史痕迹。

例如 `CommitSubTransaction()` 里有注释：

早期版本会在子事务提交时写 subcommit；当前实现只在顶层事务 commit 或 abort 的整体状态更新中需要时写。

这不是无关注释。它解释了为什么本节不能把 `RELEASE SAVEPOINT` 理解成 CLOG 层面的提交。

## 4. 关键数据结构与状态

### 4.1 `TransactionStateData`：backend-local 子事务栈

`TransactionStateData` 定义在 `xact.c` 内部。它不是公开 ABI。

本节只关心这些语义字段：

- `fullTransactionId`
- `subTransactionId`
- `parent`
- `nestingLevel`
- `childXids`
- `nChildXids`
- `curTransactionContext`
- `curTransactionOwner`
- `state`
- `blockState`

`subTransactionId` 是 backend-local 的嵌套层编号。它不是 tuple header 里的 XID。

顶层事务通常是 `TopSubTransactionId`。子事务从 2 开始递增。

`fullTransactionId` 才是可能写入 tuple header 的永久事务身份。子事务并不在创建 savepoint 时立刻分配 XID。

只有它执行需要 XID 的操作时才分配。`parent` 指向父 `TransactionStateData`。

这是 backend-local 指针。其他 backend 不能访问。

它不能跨进程解释。因此需要 `pg_subtrans` 把一部分 parent 关系转成 shared 可查询状态。

`childXids` 存的是已经 subcommit 并并入当前事务的子事务 XID。它不是所有子事务。

abort 掉的子事务不会成为顶层 commit 的 committed children。`childXids` 的生命周期在 `TopTransactionContext` 里。

这让顶层 commit 可以拿到整棵已提交子事务树。也让 abort 路径必须显式清理，避免数组泄漏。

### 4.2 `PGPROC` subxid cache：快速 running 判定

`src/include/storage/proc.h` 里定义：

```c
#define PGPROC_MAX_CACHED_SUBXIDS 64
typedef struct XidCacheStatus
{
    uint8 count;
    bool overflowed;
} XidCacheStatus;
```

每个 backend 最多在 `PGPROC->subxids.xids[]` 中公开 64 个非 aborted 子事务 XID。同时，`ProcGlobal->subxidStates[]` 保存 `subxidStatus` 的密集数组镜像。

这个 cache 的语义是：

- `count` 表示当前可内联公开的 subxid 数量。
- `overflowed = false` 表示这个 backend 的 subxid 集合完整地放在 cache 中。
- `overflowed = true` 表示至少有一个 subxid 没放进去。

这不是 visibility 结果。它只是 running set 的加速结构。

当所有 backend 都没有 overflow 时，读者找不到某个 subxid 就可以认为它不在运行。

当任意相关 backend overflow 时，找不到 subxid 不再说明它不在运行。

此时必须回查 `pg_subtrans`。

### 4.3 `pg_subtrans`：SLRU 中的 parent 指针表

`subtrans.c` 的核心模型非常小。每个 XID 对应一个 `TransactionId` 宽度的 entry。

这个 entry 保存 immediate parent XID。顶层事务没有 parent。

普通子事务有 immediate parent。代码层面：

- `SUBTRANS_XACTS_PER_PAGE = BLCKSZ / sizeof(TransactionId)`
- `TransactionIdToPage(xid)` 找 SLRU page。
- `TransactionIdToEntry(xid)` 找 page 内 entry。
- `SubTransSetParent(xid, parent)` 写 parent。
- `SubTransGetParent(xid)` 读 immediate parent。
- `SubTransGetTopmostTransaction(xid)` 沿 parent 链向上追。

`pg_subtrans` 只能从 child 走到 parent。它不能从 parent 枚举 children。

这是一个重要设计取舍。读者常常需要回答：

“这个 tuple header 里的 subxid 属于哪个 top xid？”很少需要回答：

“这个 top xid 下面有哪些 children？”顶层 commit 需要 children 列表，

但那由 backend-local `childXids` 在事务生命周期内维护，不是从 `pg_subtrans` 反查。

### 4.4 `pg_xact`：结果状态，不是父链

`src/include/access/clog.h` 定义了 CLOG status：

- `TRANSACTION_STATUS_IN_PROGRESS`
- `TRANSACTION_STATUS_COMMITTED`
- `TRANSACTION_STATUS_ABORTED`
- `TRANSACTION_STATUS_SUB_COMMITTED`

`SUB_COMMITTED` 的注释很关键。它表示：

一个子事务已经提交到父事务，但父事务尚未提交或 abort。

它不是对外最终 committed。所以 `TransactionIdDidCommit(subxid)` 遇到 `SUB_COMMITTED` 时，

会调用 `SubTransGetParent()` 继续问 parent。`TransactionIdDidAbort(subxid)` 遇到 `SUB_COMMITTED` 时，

也会继续问 parent。最终答案来自父链尽头的 top xid。

这就是 `pg_xact` 和 `pg_subtrans` 的组合语义：

`status + parent chain = final interpretation`单独看 `status` 不够。

单独看 parent 也不够。

### 4.5 `SnapshotData`：读者是否需要父链的开关

`SnapshotData` 中和本节相关的字段：

- `xmin`
- `xmax`
- `xip`
- `xcnt`
- `subxip`
- `subxcnt`
- `suboverflowed`
- `takenDuringRecovery`

普通 MVCC snapshot 中：

- `xip[]` 存 running top-level XID。
- `subxip[]` 尽量存 running subxid。
- `suboverflowed = false` 表示 `subxip[]` 完整。
- `suboverflowed = true` 表示不能相信 `subxip[]` 已列出全部 subxid。

`suboverflowed` 是 `XidInMVCCSnapshot()` 是否查 `pg_subtrans` 的关键开关。如果没有 overflow，

查 `subxip[]` 和 `xip[]` 就够了。如果 overflow，

`XidInMVCCSnapshot()` 会先把 xid 转成 topmost xid，再查 `xip[]`。

### 4.6 Recovery 中的 KnownAssignedXids

hot standby 上没有普通 backend 正在分配本地 XID。recovery 需要根据 WAL 重建 running XID 信息。

`xact.c` 会写 `XLOG_XACT_ASSIGNMENT` 记录，里面包含：

- `xtop`
- `nsubxacts`
- `xsub[]`

`procarray.c` 的 `ProcArrayApplyXidAssignment()` 会处理这类 WAL。恢复中它把每个 subxid 的 parent 直接写成 top xid。

这和正常执行时记录 immediate parent 不完全一样。但它是正确的。

原因是恢复里 aborted subtransactions 已经通过 abort WAL 明确处理。读者只需要从 subxid 回到 top-level result。

这说明一个重要边界：

`pg_subtrans` 的稳定语义是“能把 subxid 归属到 top-level outcome”。它的当前实现路径可以是 immediate parent，

也可以在 recovery 中直接指向 top xid。

## 5. 主流程源码 walkthrough

本节主流程选择一个具体对象：

一个由子事务插入的 tuple version。我们跟踪它从子事务分配 XID，

到被另一个 backend 做可见性判断，再到顶层事务 commit 或 abort。

### 5.1 创建 savepoint：只有本地栈变化

SQL 层执行：

```sql
BEGIN;
SAVEPOINT s;
```

此时通常还没有给子事务分配永久 XID。`xact.c` 中的状态变化大致是：

```text
DefineSavepoint()
  -> PushTransaction()
     -> 新建 TransactionStateData
     -> 分配 backend-local subTransactionId
     -> parent 指向上一层 TransactionState
     -> state = TRANS_DEFAULT
     -> blockState = TBLOCK_SUBBEGIN
```

随后主循环推进 transaction command：

```text
CommitTransactionCommand()
  -> StartSubTransaction()
     -> AtSubStart_Memory()
     -> AtSubStart_ResourceOwner()
     -> AfterTriggerBeginSubXact()
     -> state = TRANS_INPROGRESS
     -> CallSubXactCallbacks(SUBXACT_EVENT_START_SUB)
```

这里的关键点：

`PushTransaction()` 让 cleanup 可以处理半初始化子事务。源码注释明确要求：

从栈节点创建后，`AbortSubTransaction()` 和 `CleanupSubTransaction()` 就必须能处理它。

这解释了为什么 `AbortSubTransaction()` 里有很多“如果 resource owner 已存在”的条件。到这一步还没有 `pg_subtrans`。

还没有 tuple header XID。也还没有 `PGPROC` subxid cache。

### 5.2 子事务第一次写入：分配永久 XID

当子事务执行需要 XID 的操作，例如 INSERT，

内核会走到 XID 分配。主入口在 `xact.c`：

```text
AssignTransactionId(s)
  -> 如果 parent 还没有 XID，先给 parent 链分配 XID
  -> GetNewTransactionId(isSubXact)
  -> 如果是子事务，SubTransSetParent(child, parent)
  -> XactLockTableInsert(child)
  -> 可能写 XLOG_XACT_ASSIGNMENT
```

这里有第一个强不变量：

父事务必须先于子事务分配 XID。`AssignTransactionId()` 为了避免深递归栈溢出，

会先把未分配 XID 的 parent 链收集到数组里，再从上往下分配。

这保证 child XID 大于 parent XID。`SubTransSetParent()` 也 assert：

- `parent` 必须 valid。
- `xid` 必须 follow `parent`。

这不是美观要求。`SubTransGetTopmostTransaction()` 依赖 parent XID 单调向旧方向移动。

否则损坏的 parent chain 可能形成环，导致无限循环或错误可见性。

### 5.3 `GetNewTransactionId()`：先进入 running set

`GetNewTransactionId(isSubXact)` 在 `varsup.c`。它做几件本节关心的事：

```text
GetNewTransactionId(isSubXact)
  -> 持有 XidGenLock
  -> 取 TransamVariables->nextXid
  -> ExtendCLOG(xid)
  -> ExtendCommitTs(xid)
  -> ExtendSUBTRANS(xid)
  -> advance nextXid
  -> 如果 top-level:
       MyProc->xid = xid
       ProcGlobal->xids[pgxactoff] = xid
     如果 subxact:
       如果 PGPROC subxid cache 有空间:
         MyProc->subxids.xids[count] = xid
         write barrier
         count++
       否则:
         overflowed = true
  -> 释放 XidGenLock
```

`ExtendSUBTRANS(xid)` 只在新 page 的第一个 XID 需要实际清零。大多数 XID 分配不会触碰新的 subtrans page。

这里有一个容易读错的顺序。`GetNewTransactionId()` 会先把 subxid 放进 `PGPROC` cache，

然后 `AssignTransactionId()` 返回后立即调用 `SubTransSetParent()`。如果 cache 已满，

`GetNewTransactionId()` 只设置 overflow flag，此时别的 backend 不能靠 cache 直接看到这个 subxid。

源码注释解释了这个短窗口：

父链写入前，新 XID 还不会出现在 tuple header 或其他外部共享存储中。期间拿到的 snapshot 会包含 parent XID。

所以之后如果真的需要追问这个 subxid，`pg_subtrans` 已经能提供 parent。

不要把这个顺序简化成“subtrans 先于 PGPROC”。更准确的说法是：

subxid 进入可被 tuple visibility 追问的世界之前，parent chain 必须可用。

### 5.4 `SubTransSetParent()`：写入父指针

`SubTransSetParent(xid, parent)` 的动作很直接：

```text
SubTransSetParent(xid, parent)
  -> pageno = TransactionIdToPage(xid)
  -> entryno = TransactionIdToEntry(xid)
  -> 获取 SubTrans SLRU bank LWLock exclusive
  -> SimpleLruReadPage(..., writeOK = true)
  -> 找到 page_buffer[slotno][entryno]
  -> 如果 entry 不是 parent:
       assert entry 是 InvalidTransactionId
       写 parent
       标记 page_dirty
  -> 释放 LWLock
```

这个函数允许重复设置成同一个 parent。但不允许把一个 valid parent 改成另一个 valid parent。

这条规则保护的是父链不可变性。一旦 tuple header 或 WAL 中出现 subxid，

系统必须能长期用同一条 parent chain 解释它。`SubTransSetParent()` 不写 WAL。

它只把 `pg_subtrans` SLRU page 标 dirty。这符合本节模型：

父链只需要覆盖当前仍可能被问到的事务，不是 crash 后完整历史账本。

### 5.5 子事务 release：合并到父事务，不写最终结果

执行：

```sql
RELEASE SAVEPOINT s;
```

主路径在 `CommitSubTransaction()`。核心流程：

```text
CommitSubTransaction()
  -> CallSubXactCallbacks(PRE_COMMIT_SUB)
  -> AtEOSubXact_Parallel(true)
  -> state = TRANS_COMMIT
  -> CommandCounterIncrement()
  -> 如果子事务有 XID:
       AtSubCommit_childXids()
  -> 各子系统 AtEOSubXact(..., true)
  -> 删除子事务 XID lock
  -> resource owner 释放/转移资源
  -> AtSubCommit_Memory()
  -> PopTransaction()
```

关键注释在 `CommandCounterIncrement()` 后面。当前版本不会在这里把子事务写成 CLOG subcommit。

它只在顶层事务 commit 或 abort 的整体状态更新中需要时处理。`AtSubCommit_childXids()` 的语义是：

把当前子事务自己的 XID，以及它已经 subcommit 的 children，

复制到 parent 的 `childXids` 数组。数组顺序依赖 XID 分配顺序：

父先于子，早开始且早 subcommit 的 child XID 也更早。

这个顺序让后续按 CLOG page 成批更新更自然。`RELEASE SAVEPOINT` 后，

子事务自己的 `ResourceOwner` 被释放或把资源转给 parent。内存 context 如果为空可以直接删。

但子事务的写入语义没有独立完成。它被并入 parent。

如果 parent 后续 abort，它也 abort。

### 5.6 顶层 commit：一次性决定整棵已提交子事务树

顶层事务 commit 时进入 `RecordTransactionCommit()`。本节只保留和 subtrans 相关的主线：

```text
RecordTransactionCommit()
  -> xid = GetTopTransactionIdIfAny()
  -> nchildren = xactGetCommittedChildren(&children)
  -> XactLogCommitRecord(..., nchildren, children, ...)
  -> 同步提交时 XLogFlush(XactLastRecEnd)
  -> TransactionIdCommitTree(xid, nchildren, children)
  -> 异步提交时 TransactionIdAsyncCommitTree(..., lsn)
  -> latestXid = TransactionIdLatest(xid, nchildren, children)
```

`xactGetCommittedChildren()` 返回 `CurrentTransactionState->childXids`。这个数组来自每次 `CommitSubTransaction()` 的合并。

所以顶层 commit 不需要从 `pg_subtrans` 反查 children。它已经在 backend-local state 里持有 committed children 列表。

这就是为什么 `pg_subtrans` 可以只做 child-to-parent。

### 5.7 `TransactionIdSetTreeStatus()`：`SUB_COMMITTED` 的真正位置

`TransactionIdCommitTree()` 在 `transam.c`。它调用 `clog.c` 的 `TransactionIdSetTreeStatus()`。

这个函数处理一个困难：

顶层 XID 和所有 subxid 可能分布在多个 CLOG page。单个 CLOG page 更新可以在一个 SLRU bank lock 下完成。

多 page 更新不是天然原子。PostgreSQL 的做法是三阶段：

```text
如果 xid 和所有 subxid 在同一个 CLOG page:
  -> 一次性把 top xid 和 subxids 写成 committed
如果跨 CLOG page:
  -> 先把不在 top xid page 上的 subxids 写成 SUB_COMMITTED
  -> 再把 top xid 和同页 subxids 写成 COMMITTED
  -> 最后把前面那些 subxids 写成 COMMITTED
```

为什么这能保持读者看到的语义？因为读者遇到 `SUB_COMMITTED` 不会返回 committed。

它会沿 `pg_subtrans` 找 parent。如果 parent 还没 committed，

读者不会把子事务当最终 committed。如果 parent 已 committed，

读者可以得到 committed。因此跨 page 的非原子更新被 `SUB_COMMITTED + parent chain` 转换成对外原子语义。

这就是本节最核心的正确性连接。`SUB_COMMITTED` 不是一个业务状态。

它是 CLOG 多页更新时的中间状态，需要 `pg_subtrans` 帮它回到 top-level commit decision。

### 5.8 可见性读路径：从 tuple header 的 subxid 回到 top xid

普通 SELECT 的 heap visibility 主入口是 `HeapTupleSatisfiesMVCC()`。当 tuple 的 `xmin` 没有 hint bit 表示 committed 时，

大致逻辑是：

```text
HeapTupleSatisfiesMVCC(tuple, snapshot)
  -> 如果 xmin 是当前事务:
       用 command id 判断自可见性
  -> else if XidInMVCCSnapshot(xmin, snapshot):
       插入事务对这个 snapshot 仍 in progress
       tuple 不可见
  -> else if TransactionIdDidCommit(xmin):
       可以设置 HEAP_XMIN_COMMITTED hint bit
  -> else:
       aborted/crashed
       tuple 不可见
```

`XidInMVCCSnapshot()` 是本节的关键。如果 snapshot 没有 subxid overflow：

```text
XidInMVCCSnapshot(xid, snapshot)
  -> range check: xid < xmin => false
  -> range check: xid >= xmax => true
  -> 查 snapshot->subxip[]
  -> 查 snapshot->xip[]
```

如果 snapshot 有 subxid overflow：

```text
XidInMVCCSnapshot(xid, snapshot)
  -> range check
  -> xid = SubTransGetTopmostTransaction(xid)
  -> 如果 top xid < xmin => false
  -> 查 snapshot->xip[]
```

这一步说明：

`pg_subtrans` 不只服务 CLOG 的 `SUB_COMMITTED`。它也服务 snapshot 信息不完整时的 running 判定。

当 `subxip[]` 不能完整保存所有 running subxid，系统只能把待查 subxid 归并到 top xid，

再问 top xid 是否在 snapshot 的 running top-level 集合里。

### 5.9 `SubTransGetTopmostTransaction()`：允许“有限撒谎”

`SubTransGetTopmostTransaction(xid)` 会循环：

```text
previousXid = xid
parentXid = xid
while parentXid valid:
  previousXid = parentXid
  if parentXid < TransactionXmin:
    break
  parentXid = SubTransGetParent(parentXid)
  if parentXid 不早于 previousXid:
    ERROR
return previousXid
```

函数注释说它可能返回 intermediate subtransaction，而不是真正 topmost parent。

这看起来危险，但在它的调用语境里是可以接受的。

原因是：

系统不能查早于 `TransactionXmin` 的 `pg_subtrans`。那些父链可能已经被截断。

但可见性只关心：

这个 top-level outcome 是否仍 running，或是否属于当前 snapshot 的 running set。

早于 `TransactionXmin` 的 XID 不可能仍需要作为当前 running parent 被精确追踪。因此返回一个足够老的 intermediate xid，

在这些调用中等价于返回更老的 top xid。这不是通用 API 语义。

不要拿 `SubTransGetTopmostTransaction()` 去做需要完整树形结构的工具。它是 visibility/running 判定的内部 helper。

### 5.10 `TransactionIdDidCommit()`：`SUB_COMMITTED` 继续追问 parent

`transam.c` 的 `TransactionIdDidCommit()`：

```text
TransactionIdDidCommit(xid)
  -> status = TransactionLogFetch(xid)
  -> 如果 status == COMMITTED:
       return true
  -> 如果 status == SUB_COMMITTED:
       如果 xid < TransactionXmin:
         return false
       parent = SubTransGetParent(xid)
       如果 parent invalid:
         WARNING
         return false
       return TransactionIdDidCommit(parent)
  -> return false
```

`TransactionIdDidAbort()` 是对称的：

遇到 `SUB_COMMITTED` 继续问 parent。但如果父链不可用，

它返回 true。这两个函数体现了 fallback 策略：

如果一个 subcommitted XID 的 parent 缺失，系统宁愿保守地不把它当 committed。

对可见性来说，错误地认为未提交比错误地认为已提交安全。

### 5.11 `TransactionIdIsInProgress()`：PGPROC cache 失败后的父链慢路

`procarray.c` 的 `TransactionIdIsInProgress()` 用于当前 running 判定。它不是 MVCC snapshot 的主入口，

但很多非 snapshot 场景会用它。流程大致是：

```text
TransactionIdIsInProgress(xid)
  -> xid < RecentXmin => false
  -> cachedXidIsNotInProgress 命中 => false
  -> 当前事务或当前子事务 => true
  -> 持 ProcArrayLock shared
  -> 查每个 backend 的 top xid
  -> 查每个 backend 的 cached subxids
  -> 如果某 backend overflowed，保存它的 top xid
  -> recovery 下查 KnownAssignedXids
  -> 释放 ProcArrayLock
  -> 如果没有 overflow candidate => false
  -> 如果 xid 已 aborted => false
  -> topxid = SubTransGetTopmostTransaction(xid)
  -> 如果 topxid 在 candidate top xid 列表 => true
  -> false
```

这个流程解释了 `pg_subtrans` 的 hot path 边界。没有 overflow 时，

`pg_subtrans` 不在普通 running 判定路径上。有 overflow 时，

它才成为“找不到 subxid 但不能相信找不到”的 fallback。

## 6. 生命周期 / ownership / cleanup

### 6.1 谁创建 parent chain？

父链创建发生在子事务分配永久 XID 后。主入口：

- `AssignTransactionId()` 判断 `isSubXact`。
- `GetNewTransactionId(true)` 分配 child xid。
- `SubTransSetParent(child, parent)` 写入 `pg_subtrans`。

父事务如果还没有 XID，`AssignTransactionId()` 会先给父链分配。

这保证 parent xid 早于 child xid。

### 6.2 谁持有 parent chain？

`pg_subtrans` 由 SLRU 管理。它的内存页在 shared memory buffer 中。

它的文件在数据目录的 `pg_subtrans` 下。访问需要对应 SLRU bank 的 LWLock。

没有某个 backend “拥有”一个 parent entry。写入者只是当前分配 child xid 的 backend。

读者可以是：

- heap visibility 路径。
- `TransactionIdIsInProgress()`。
- `TransactionIdDidCommit()`。
- `TransactionIdDidAbort()`。
- standby recovery 相关路径。
- 部分 lock/predicate 逻辑的 top xid 归并路径。

### 6.3 谁释放 parent chain？

单个 parent entry 没有逐项释放。释放粒度是 SLRU segment/page 的 truncation。

`TruncateSUBTRANS(oldestXact)` 会删除早于包含 `oldestXact` 的 segment。这里的 `oldestXact` 是所有 running transaction 的最老 `TransactionXmin` 边界。

只要某个 running snapshot 仍可能需要查较老 XID 的 parent，截断就不能越过它。

`SubTransGetParent()` assert：

不能问早于 `TransactionXmin` 的东西。这就是 truncation 与 lookup 的边界。

### 6.4 Top-level commit 如何收尾？

顶层 commit 收尾有两部分。第一部分是事务结果：

- `RecordTransactionCommit()` 收集 `childXids`。
- WAL commit record 记录 children。
- 同步提交时先 flush WAL。
- `TransactionIdCommitTree()` 更新 `pg_xact`。
- 异步提交时 `TransactionIdAsyncCommitTree()` 带上 commit LSN。

第二部分是 running set：

顶层事务结束后会从 `PGPROC` 中清理 top xid 和 subxid cache。这个清理在 top-level transaction cleanup 路径中完成，

不依赖逐项删除 `pg_subtrans`。`pg_subtrans` 只保留到 horizon 允许截断。

### 6.5 Subtransaction abort 如何收尾？

子事务 abort 走：

```text
AbortSubTransaction()
  -> HOLD_INTERRUPTS()
  -> AtSubAbort_Memory()
  -> AtSubAbort_ResourceOwner()
  -> LWLockReleaseAll()
  -> LockErrorCleanup()
  -> state = TRANS_ABORT
  -> RecordTransactionAbort(true)
  -> AtSubAbort_childXids()
  -> 各子系统 abort cleanup
  -> RESUME_INTERRUPTS()
CleanupSubTransaction()
  -> AtSubCleanup_Portals()
  -> ResourceOwnerDelete()
  -> AtSubCleanup_Memory()
  -> PopTransaction()
```

`RecordTransactionAbort(true)` 会：

- 写 abort record。
- 调用 `TransactionIdAbortTree(xid, nchildren, children)`。
- 将本子事务和它已提交的 children 标为 aborted。
- 调用 `XidCacheRemoveRunningXids()` 从当前 backend 的 subxid cache 中移除失败 XID。

这里的语义很重要。一个子事务 abort 后，

即使它的 parent 还在运行，这个子事务的 tuple 也不能继续被当作 running parent 的一部分。

因此 `TransactionIdIsInProgress()` 的慢路在追父链前会先检查 `TransactionIdDidAbort(xid)`。否则它可能看到 parent 还在运行，

误判已经失败的 child 也在运行。

### 6.6 MemoryContext 与 ResourceOwner 的分工

子事务的内存生命周期：

- `AtSubStart_Memory()` 创建 child `CurTransactionContext`。
- subcommit 时切回 parent context。
- 如果 child context 空，可以删除。
- subabort cleanup 时删除 child context。
- 顶层 cleanup 时 reset `TopTransactionContext`。

子事务的外部资源生命周期：

- `AtSubStart_ResourceOwner()` 创建 child resource owner。
- subcommit 时资源释放或转移到 parent。
- subabort 时资源释放。
- cleanup 时删除 resource owner。

这两者都不是 `pg_subtrans` 的 owner。`pg_subtrans` 是 shared SLRU 状态。

不能用 MemoryContext 或 ResourceOwner 的生命周期解释它。

### 6.7 Startup 和 crash 后的边界

`StartupSUBTRANS(oldestActiveXID)` 会把当前 active page 范围清零。源码注释说得很直接：

`pg_subtrans` 不需要 crash 后保持有效。crash 后未完成事务按 abort 处理。

仍需要保留的 prepared transaction 由 startup/prescan 路径重新建立必要边界。这和 `pg_xact` 完全不同。

`pg_xact` 必须参与 crash recovery。`pg_subtrans` 只服务当前仍可能被追问的父链。

### 6.8 Checkpoint 的角色

`CheckPointSUBTRANS()` 会写脏的 SUBTRANS page。但注释强调：

这不是 correctness 所必需。它只是提高由 checkpoint process 而不是普通 backend 写脏页的概率。

这也是为什么 `SUBTRANSShmemRequest()` 设置 `sync_handler = SYNC_HANDLER_NONE`。不要把 checkpoint 写 `pg_subtrans` 理解成 WAL crash safety。

它不是。

## 7. 正确性机制层次

### 7.1 第一层：XID 分配顺序

父事务先分配 XID。子事务后分配 XID。

这保证 parent chain 单调向更老 XID 移动。`SubTransGetTopmostTransaction()` 用这个性质防止环。

如果 parent 不早于 child，它会 `elog(ERROR)`。

这属于结构正确性。它不直接回答 visibility。

### 7.2 第二层：running set 发布

`PGPROC->xid` 和 `PGPROC->subxids` 是 running set 的快速路径。写入 subxid cache 时使用 write barrier。

读者在 `procarray.c` 中先读 count，再用 read barrier，

再读数组元素。这保证读者不会看到 count 已增加但数组元素未初始化的中间状态。

这属于 shared memory 并发正确性。它不保证事务已经提交。

### 7.3 第三层：parent chain fallback

当 `PGPROC` subxid cache 不完整，或者 CLOG 看到 `SUB_COMMITTED`，

系统需要 parent chain。`pg_subtrans` 保证 child 能找到 parent。

它不保证 parent committed。它也不保证能枚举 children。

这属于归属正确性。

### 7.4 第四层：CLOG result

`pg_xact` 才记录最终状态。`COMMITTED` 和 `ABORTED` 是最终结果。

`SUB_COMMITTED` 是过渡状态。`IN_PROGRESS` 是全零初始状态，

通常还要结合 ProcArray/running 判定解释。这属于事务结果正确性。

### 7.5 第五层：WAL-before-CLOG

顶层 commit 写 WAL commit record。同步提交时先 flush WAL，

再把 CLOG 写成 committed。异步提交时 CLOG update 带上 commit record LSN，

避免 CLOG committed 先安全落盘而 WAL 不安全。这属于 crash safety。

`pg_subtrans` 不提供 crash safety。

### 7.6 第六层：snapshot 边界

`GetSnapshotData()` 在 `ProcArrayLock` 下收集 top-level XID 和可缓存 subxid。它设置：

- `snapshot->xmin`
- `snapshot->xmax`
- `snapshot->xip`
- `snapshot->subxip`
- `snapshot->suboverflowed`

`XidInMVCCSnapshot()` 使用这些字段，而不是每次都扫描当前 ProcArray。

这保证一个 snapshot 内 running set 判断稳定。

### 7.7 第七层：tuple hint bit

heap visibility 可以在 tuple header 上设置 hint bit。例如 `HEAP_XMIN_COMMITTED`。

hint bit 是结果缓存。它不是父链来源。

如果一个 subxid 的结果需要沿 parent chain 才能解释，hint bit 只能在结果已经能被正确判断后写入。

### 7.8 这些层不能互相替代

`PGPROC` cache 不能替代 `pg_xact`。它只表示 running。

`pg_xact` 不能替代 `pg_subtrans`。遇到 subxid 时它可能只有 `SUB_COMMITTED`。

`pg_subtrans` 不能替代 WAL。它 crash 后不保证完整。

snapshot 不能替代 `pg_subtrans`。snapshot 可能 `suboverflowed`。

hint bit 不能替代任何一个基础机制。它只是加速后续 tuple 判断。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 Subxid cache overflow

最重要的 fallback 是 `PGPROC` subxid cache overflow。每个 backend 只能内联 64 个 subxid。

超过后：

- `MyProc->subxidStatus.overflowed = true`
- `ProcGlobal->subxidStates[pgxactoff].overflowed = true`
- 后续 snapshot 标记 `suboverflowed`
- 后续 running 判断可能需要 `pg_subtrans`

这不是错误。这是有意设计的退化路径。

系统用小的 shared memory cache 覆盖常见路径，用 `pg_subtrans` 覆盖极端子事务数量。

### 8.2 `SUB_COMMITTED` parent 缺失

`TransactionIdDidCommit()` 遇到 `SUB_COMMITTED` 时需要 parent。如果 xid 早于 `TransactionXmin`，

它不能安全查 `pg_subtrans`。此时 `DidCommit()` 返回 false。

`DidAbort()` 返回 true。如果 `SubTransGetParent()` 返回 invalid，

代码会打 WARNING：

```text
no pg_subtrans entry for subcommitted XID %u
```

然后采取保守解释。这通常不该在正常路径出现。

但 prepared transaction、startup 边界和父链不完整窗口使源码不能只写 assert。

### 8.3 损坏的 parent chain

`SubTransGetTopmostTransaction()` 要求 parent xid 早于 child xid。如果发现 parent 不早于 previous xid，

会报 ERROR：

```text
pg_subtrans contains invalid entry
```

这是防止无限循环和错误可见性的硬保护。它说明 `pg_subtrans` 虽然不是 crash safety 日志，

但当前运行期父链一旦被使用，结构必须自洽。

### 8.4 子事务 abort 但 parent 仍 running

这是容易错的 case。子事务 abort 后，

parent 可能还在运行。如果 running 判定直接沿父链找 top xid，

会把已经 abort 的 child 当成 running。所以 `TransactionIdIsInProgress()` 的 slow path 先做：

```text
if TransactionIdDidAbort(xid):
  return false
```

然后才 `SubTransGetTopmostTransaction(xid)`。这一步把“child 独立失败”从“parent 仍运行”中分离出来。

### 8.5 子事务创建或结束过程中 ERROR

`xact.c` 的 transaction block state machine 对子事务异常很保守。如果在子事务内部 ERROR：

- `AbortSubTransaction()` abort 当前子事务。
- 状态进入 persistent `SUBABORT`。
- 直到用户执行 `ROLLBACK TO` 或结束事务块。

如果在创建子事务或结束子事务过程中 ERROR：

- 先清理破损子事务。
- 再 abort parent。

这是因为 half-started 或 half-ended 子事务无法安全继续作为 parent 的一部分。

### 8.6 OOM 与 committed children 数组

`AtSubCommit_childXids()` 可能需要扩展 parent 的 `childXids` 数组。如果超过 `MaxAllocSize / sizeof(TransactionId)`，

会报：

```text
maximum number of committed subtransactions exceeded
```

这不是普通内存泄漏。它是顶层 commit 必须持有完整 committed children 列表的结构限制。

没有这个列表，顶层 commit 无法一次性把整棵已提交子事务树写入 WAL/CLOG。

### 8.7 并行模式下不能新分配 XID

`AssignTransactionId()` 和 `GetNewTransactionId()` 都防止在 parallel operation 中分配新 XID。原因是 worker 的事务状态在 parallel operation 开始时同步。

中途新增 XID 会让 leader/worker 的事务身份不一致。这和 `pg_subtrans` 不是同一个机制，

但它保护的是同一条父链语义：

不能让某个执行者写入别人无法解释的 subxid。

### 8.8 CLOG group update fallback

`clog.c` 对小事务树有 group status update 优化。但它要求：

- 所有 XID 在同一 CLOG page。
- subxid 数量不超过阈值。
- `MyProc` 中的 subxid cache 与待更新数组一致。

不满足就回退为普通 SLRU bank lock 更新。这影响性能，

不改变语义。语义仍由 `TransactionIdSetTreeStatus()` 的 `SUB_COMMITTED` 过渡保证。

## 9. 成本、资源与跨模块传播

### 9.1 正常路径成本

少量子事务的常见路径很便宜。写入成本：

- 分配 child XID。
- 写 `PGPROC->subxids` 一个数组元素。
- 写 `pg_subtrans` 一个 parent entry。

读路径成本：

- snapshot 里有完整 `subxip[]`。
- `XidInMVCCSnapshot()` 做 range check。
- 查 `subxip[]` 和 `xip[]`。
- 不访问 `pg_subtrans`。

这种情况下，`pg_subtrans` 主要是备用解释表。

### 9.2 子事务数量扩大后的成本

成本随几个变量扩张：

- 一个顶层事务内 subxid 数量。
- 活跃 backend 数。
- snapshot 持有时间。
- `pg_subtrans` SLRU buffer 命中率。
- CLOG page 跨越数量。

当 subxid 超过 64，`PGPROC` cache overflow。

此后其他 backend 的 snapshot 可能 `suboverflowed`。每个在 `[snapshot->xmin, snapshot->xmax)` 范围内的可疑 subxid，

都可能需要：

```text
SubTransGetTopmostTransaction()
  -> SubTransGetParent()
  -> SLRU page lookup
  -> 可能多级 parent chain
```

这是下一节的主问题。本节只记住：

父链把 correctness 保存下来，但也把 overflow 后的成本传播到 visibility hot path。

### 9.3 CLOG 更新放大

顶层 commit 时，`nChildXids` 越多，

CLOG 更新越大。如果 XID 跨多个 CLOG page，

`TransactionIdSetTreeStatus()` 需要多批更新。跨 page commit 还会出现 `SUB_COMMITTED` 过渡。

因此一个大量 savepoint 的事务，哪怕每个子事务只写很少数据，

也会在 commit 时集中支付状态更新成本。

### 9.4 `childXids` 的本地内存成本

每个已 subcommit 且有 XID 的子事务，都需要进入 parent 的 `childXids`。

数组在 `TopTransactionContext` 中增长。它会在顶层事务结束后释放。

长事务加大量子事务会带来本地内存增长。这不是 `work_mem`。

也不是 shared memory。它属于事务本地状态。

### 9.5 `pg_subtrans` SLRU 成本

`pg_subtrans` 是 SLRU。相关资源包括：

- shared SLRU buffers。
- `SubtransBuffer` I/O wait。
- `SubtransSLRU` LWLock wait。
- `pg_subtrans` 文件页读写。
- checkpoint 写 dirty pages。

`subtransaction_buffers` 控制 buffer 数。默认 `0` 表示 auto-tune。

当前源码中 auto-tune 大致按 shared buffers 规模估计，并在上限内限制。

增加 buffer 只能减少 SLRU miss 或 I/O。它不能消除 parent chain lookup 本身。

### 9.6 XID 消耗与 wraparound 压力

每个分配了永久 XID 的子事务都会消耗 XID。大量 savepoint 或 PL/pgSQL exception block 可能快速推进 `nextXid`。

这会间接影响：

- `pg_xact` 扩展。
- `pg_subtrans` 扩展。
- autovacuum anti-wraparound 压力。
- snapshot horizon。
- clog/subtrans truncation 节奏。

所以子事务不是“纯本地控制流”。只要它分配了 XID，

它就进入全局事务 ID 空间。

### 9.7 跨模块传播边界

本节至少连接这些模块：

- `xact.c`：子事务栈和事务结束协议。
- `varsup.c`：XID 分配和 `PGPROC` 发布。
- `subtrans.c`：父链存储。
- `clog.c`：事务结果状态。
- `procarray.c`：running set 和 snapshot。
- `snapmgr.c`：MVCC snapshot 判断。
- `heapam_visibility.c`：tuple 可见性。
- `xlog.c`/recovery：startup、checkpoint、WAL assignment。

其中只有部分模块直接修改 `pg_subtrans`。但很多模块依赖父链语义。

这是 PostgreSQL 事务系统常见的工程形态：

状态很小，语义跨模块组合。

## 10. 观测与诊断入口

### 10.1 能直接看到什么

可以直接看到：

- backend 当前缓存了多少 subxid。
- backend subxid cache 是否 overflow。
- `pg_stat_slru` 中 `subtransaction` 的 SLRU 读写统计。
- backend wait event 是否在 `SubtransBuffer` 或 `SubtransSLRU`。
- `pg_locks` 中 transactionid lock。
- `pg_stat_activity.backend_xid` 和 `backend_xmin`。

当前版本提供：

```sql
SELECT *
FROM pg_stat_get_backend_subxact(backend_id);
```

返回字段包括：

- `subxact_count`
- `subxact_overflowed`

可以结合：

```sql
SELECT *
FROM pg_stat_get_backend_idset() AS bid
CROSS JOIN LATERAL pg_stat_get_backend_subxact(bid);
```

这能看到每个 backend 的 subxid cache 状态。`pg_stat_slru` 可以看：

```sql
SELECT *
FROM pg_stat_slru
WHERE name = 'subtransaction';
```

关注字段：

- `blks_zeroed`
- `blks_hit`
- `blks_read`
- `blks_written`
- `flushes`
- `truncates`

这些是 instance 累计统计。它们不能告诉你哪条 query 导致某次 lookup。

### 10.2 只能推断什么

以下状态通常只能推断：

- 某个 tuple header 的 XID 是否是 subxid。
- 某次 `XidInMVCCSnapshot()` 是否走了 `SubTransGetTopmostTransaction()`。
- 某个 snapshot 的 `suboverflowed` 是否影响了具体 tuple 判断。
- 某条 parent chain 的长度。
- 某次 CLOG `SUB_COMMITTED` 过渡是否被读者撞见。

这些需要源码断点、debug log、perf 或注入计数器。SQL 视图不能直接给出完整因果。

### 10.3 几乎不可见什么

几乎不可见：

- `pg_subtrans` 某个 entry 的 parent 值。
- `SubTransGetParent()` 每次读取哪个 page。
- `SubTransGetTopmostTransaction()` 返回 intermediate 还是 true top xid。
- `childXids` 数组的当前内容。
- CLOG 多 page commit 中短暂 `SUB_COMMITTED` 的瞬间。

这些不是稳定 SQL API。不要把 `pg_subtrans` 文件当作可直接解析的诊断接口。

### 10.4 一个诊断闭环

看到现象：

某业务大量使用 savepoint 或 PL/pgSQL exception block。在并发 SELECT 下，

CPU 增加，`pg_stat_slru` 的 `subtransaction` 读增加，

部分 backend wait event 出现 `SubtransSLRU`。解释路径：

大量子事务让某些 backend 的 `PGPROC` subxid cache overflow。snapshot 标记 `suboverflowed`。

heap visibility 对 in-range tuple XID 调用 `XidInMVCCSnapshot()`。因为 subxip 不完整，

它调用 `SubTransGetTopmostTransaction()`。该调用访问 `pg_subtrans` SLRU。

回到源码：

- `GetNewTransactionId()` 设置 overflow flag。
- `GetSnapshotData()` 设置 `snapshot->suboverflowed`。
- `XidInMVCCSnapshot()` 调用 `SubTransGetTopmostTransaction()`。
- `SubTransGetParent()` 读 SLRU。

这就是本节的 runtime truth：

subtrans 父链平时隐藏在事务管理后面，但一旦 snapshot 不能完整携带 subxid，

它会进入 visibility hot path。

### 10.5 用 gdb 断点看父链

适合源码实验的断点：

```gdb
break SubTransSetParent
break SubTransGetTopmostTransaction
break TransactionIdSetTreeStatus
break XidInMVCCSnapshot
break TransactionIdDidCommit
```

观察重点：

- `SubTransSetParent()` 的 `xid` 和 `parent`。
- `TransactionIdSetTreeStatus()` 的 `nsubxids`。
- `XidInMVCCSnapshot()` 的 `snapshot->suboverflowed`。
- `SubTransGetTopmostTransaction()` 的返回值。

不要只看函数是否被调用。要把调用和 SQL 里的 savepoint 数量、snapshot 时间、tuple XID 联系起来。

### 10.6 perf 视角

如果怀疑 subtrans 成本，`perf` 或火焰图上可能看到：

- `XidInMVCCSnapshot`
- `SubTransGetTopmostTransaction`
- `SubTransGetParent`
- `SimpleLruReadPage_ReadOnly`
- `TransactionLogFetch`
- `HeapTupleSatisfiesMVCC`

但 perf 只能说明 CPU 时间在哪里。它不能自动证明根因是 subxid overflow。

需要结合：

- `pg_stat_get_backend_subxact()`
- `pg_stat_slru`
- workload 中 savepoint/exception 使用量
- 长事务和 snapshot 持有情况

### 10.7 日志与 wait event 粒度

`SubtransBuffer` 表示等待 subtransaction SLRU buffer I/O。`SubtransSLRU` 表示等待访问 subtransaction SLRU cache。

这两个 wait event 是 backend 当前等待状态。它们不是累计因果。

如果采样时看到，说明当前确实卡在 subtrans SLRU 相关路径。

如果没看到，也不能说明没有 subtrans 成本。

CPU 上的 parent chain lookup 不一定表现为 wait。

## 11. 常见误区

### 11.1 把 `RELEASE SAVEPOINT` 当成独立 commit

`RELEASE SAVEPOINT` 只把子事务状态并入父事务。它不是对外可见性的最终点。

顶层事务 abort 后，已经 release 的子事务写入仍会消失。

### 11.2 把 `SUB_COMMITTED` 当成 committed

`SUB_COMMITTED` 不能直接返回 committed。它必须沿 `pg_subtrans` 找 parent。

最终答案来自 parent/top xid。

### 11.3 以为 `pg_subtrans` 保存完整事务树

`pg_subtrans` 只保存 child 到 parent。它不保存 parent 到 children。

顶层 commit 的 children 列表来自 `xact.c` 的 `childXids`。

### 11.4 以为 `pg_subtrans` 是 WAL 保护的历史表

`pg_subtrans` 没有自己的 WAL redo。startup 会清零当前 active page。

它只需要覆盖当前仍可能被查询的父链。

### 11.5 只看 `pg_xact` 不看 ProcArray

判断 running 不能只看 CLOG。CLOG 初始状态是 in-progress。

事务是否仍运行还要看 ProcArray。子事务 overflow 时还要看 `pg_subtrans`。

### 11.6 以为增加 `subtransaction_buffers` 能修复所有问题

增加 buffer 可能减少 SLRU read/write。它不能减少子事务数量。

它不能让 `PGPROC` cache 不 overflow。它也不能消除 `XidInMVCCSnapshot()` 的 parent chain 逻辑。

### 11.7 忽略 abort 子事务的独立失败语义

子事务 abort 后，parent 可能还在运行。

不能因为 parent running 就认为 child running。`TransactionIdIsInProgress()` slow path 先检查 child 是否 aborted，

就是为了解决这个边界。

### 11.8 把 `SubTransactionId` 和 `TransactionId` 混用

`SubTransactionId` 是 backend-local nesting id。`TransactionId` 是全局 XID。

tuple header 存的是 `TransactionId`。`pg_subtrans` 也按 `TransactionId` 索引。

## 12. 课堂实验

实验 1：观察 subxid cache overflow。一个会话在事务中通过 PL/pgSQL exception block 或大量 savepoint 执行写操作，另一个会话查询 `pg_stat_get_backend_subxact()`。

重点看 `subxact_count` 是否停在 64，以及 `subxact_overflowed` 是否变成 true。回源码解释 `GetNewTransactionId(true)` 填 `PGPROC` subxid cache，超过 `PGPROC_MAX_CACHED_SUBXIDS` 后设置 overflow。

实验 2：断点确认父链写入。断 `AssignTransactionId`、`GetNewTransactionId`、`SubTransSetParent`，执行 `BEGIN; SAVEPOINT; INSERT; RELEASE SAVEPOINT; COMMIT;`。

观察 parent XID 先分配，child XID 后分配，并调用 `SubTransSetParent(child, parent)`。再断 `AtSubCommit_childXids` 和 `TransactionIdSetTreeStatus`，确认 release savepoint 只是把 child 合并到 parent 的 children 集合，最终结果仍由 top-level commit 写入。

实验 3：观察 SLRU 诊断边界。用大量写入型子事务制造 `pg_subtrans` 压力，比较 `pg_stat_slru where name='subtransaction'` 和 wait event。

没有 wait event 不等于没有 lookup；有 SLRU read 增长也不能单独证明某条 query 是根因。诊断要把 workload 的 savepoint/exception 模式、snapshot overflow、`SubTransGetParent()` 调用链放在一起看。

## 13. 讨论题

1. 为什么 `RELEASE SAVEPOINT` 不能把子事务 XID 直接写成 `COMMITTED`？
2. `pg_subtrans` 为什么只保存 child-to-parent，而不保存 parent-to-children？
3. `TransactionIdDidCommit()` 遇到 `SUB_COMMITTED` 时为什么必须递归问 parent？
4. `PGPROC_MAX_CACHED_SUBXIDS` 只有 64 时，为什么 correctness 仍能成立？
5. 子事务 abort 后 parent 仍 running，`TransactionIdIsInProgress()` 为什么要先检查 child 是否 aborted？
6. `StartupSUBTRANS()` 清零当前 active page，为什么不破坏 crash recovery correctness？
7. `subtransaction_buffers` 增大后，哪些成本可能下降，哪些成本不会下降？
8. 如果你在 `pg_stat_activity` 看到 `SubtransSLRU`，还需要哪些证据才能判断业务中过量 savepoint 是根因？

## 14. 本节小结

本节唯一主问题是：

tuple header 中保存 subxid 时，PostgreSQL 如何追溯到顶层事务并决定最终可见性。

核心链路是：

```text
子事务分配 XID
  -> SubTransSetParent(child, parent)
  -> 子事务 release 合并 childXids
  -> 顶层 commit/abort 写 pg_xact tree
  -> 读者遇到 subxid 时必要情况下查 pg_subtrans
  -> 再用 pg_xact/snapshot 得到最终结果
```

核心状态和边界：

- `TransactionStateData` 是 backend-local 子事务栈。
- `PGPROC->subxids` 是 shared memory running cache。
- `pg_subtrans` 是 subxid 到 parent 的临时 SLRU。
- `pg_xact` 是事务结果状态。
- `SnapshotData.suboverflowed` 决定读者是否需要 parent chain fallback。

ownership 与 cleanup：

- 子事务栈节点和 `childXids` 属于事务本地内存。
- 子事务资源由 `ResourceOwner` 转移或释放。
- `pg_subtrans` entry 没有逐项释放。
- SLRU truncation 根据 oldest running horizon 清理旧 segment。
- crash 后 `pg_subtrans` 不作为完整历史恢复。

错误路径：

- subxid cache overflow 触发 `pg_subtrans` 慢路。
- parent chain 缺失时对 committed 采取保守解释。
- 损坏的 parent chain 会 ERROR。
- 子事务 abort 会立即把失败 XID 从 running subxid cache 中移除。

可观测性：

- `pg_stat_get_backend_subxact()` 能看到 cache count 和 overflow。
- `pg_stat_slru` 能看到 `subtransaction` SLRU 累计统计。
- `SubtransBuffer` 和 `SubtransSLRU` wait event 能看到等待。
- 单个 parent entry 和单次 visibility fallback 通常不可直接从 SQL 看到。

可迁移规律：

PostgreSQL 常用小而快的 shared memory cache 覆盖 hot path，再用可截断的共享辅助结构保存 correctness fallback。

这种结构的代价是：

正常路径很便宜，但一旦 cache 信息不完整，

成本会传播到原本看似无关的 visibility hot path。判断这类问题时，

不要只问“状态在哪里”。还要问：

- 这个状态是否完整？
- 不完整时 fallback 查哪里？
- fallback 的生命周期边界是什么？
- fallback 是否参与 crash safety？
- fallback 的成本会传播到哪个 hot path？

这些问题会在下一节 `SubXID overflow 与可见性成本` 中继续展开。
