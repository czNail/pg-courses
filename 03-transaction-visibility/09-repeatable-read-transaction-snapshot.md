# PostgreSQL REPEATABLE READ 事务快照与稳定读成本

## 课程定位

前置知识：

- 已经理解 `SnapshotData` 中 `xmin`、`xmax`、`xip`、`subxip` 的基本含义。
- 已经理解 `GetSnapshotData()` 从 ProcArray 构造 MVCC snapshot。
- 已经理解 SubXID overflow 会让可见性判断退回 `pg_subtrans`。
- 已经知道 heap tuple visibility 最终会进入 `HeapTupleSatisfiesMVCC()`。

本节唯一主问题：

```text
为什么 REPEATABLE READ / SERIALIZABLE 要把第一次 MVCC snapshot 固定到事务结束，并且这个稳定读承诺如何转化为 cleanup horizon 和 SSI 成本？
```

本节围绕的核心矛盾：

- 用户希望一个长事务内多条语句看到同一个数据库版本。
- 系统又希望 VACUUM 尽快回收旧版本，SSI 尽快释放 predicate lock 和 conflict 状态。
- PostgreSQL 选择把事务级 snapshot 复制、注册并持有到事务结束。
- 这个选择给查询稳定性一个清晰边界，但会把 `xmin`、old tuple version、predicate lock 和 serializable conflict 的生命周期拉长。

学完本节后，应能独立判断：

- 为什么 READ COMMITTED 每条语句重新取 snapshot，而 REPEATABLE READ 只在第一次取。
- 为什么 `IsolationUsesXactSnapshot()` 是本节的语义分叉点。
- 为什么事务级 snapshot 必须 `CopySnapshot()` 到 `TopTransactionContext`。
- 为什么 `FirstXactSnapshot` 会进入 `RegisteredSnapshots`。
- 为什么长时间空闲的 RR 事务也可能阻止 cleanup horizon 推进。
- 为什么 SERIALIZABLE 不是更强的 snapshot 字段，而是在稳定 snapshot 外叠加 SSI 状态。
- 为什么 `AtEOXact_Snapshot()` 是释放稳定读承诺的边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开：

- 不重新讲完整 `GetSnapshotData()` 数组采集细节。
- 不展开所有 SSI predicate lock target 类型。
- 不讲 HOT pruning、freezing 和 VACUUM 的完整实现。
- 不讲 READ COMMITTED 下 EvalPlanQual 的全部分支。
- 这些内容只在解释事务级 snapshot 固定成本时出现。

## 1. 本节在总主线中的位置

前几节课已经把可见性拆成两层。第一层是 snapshot 记录创建时仍在运行的事务集合。

第二层是 heap visibility 用 tuple header 对照这个集合。到这里还有一个没有回答的问题：

- snapshot 是每条 SQL 重新取，还是一个事务只取一次？

这个选择不是语法细节。它决定一个事务里第二条 `SELECT` 看到的是现在，还是事务开始读到的那个逻辑时间点。

READ COMMITTED 的运行模型更像：

- 每条 statement 进入 executor 前取一次新的 MVCC snapshot。
- 语句结束后可以释放这个 statement snapshot。
- 下一条语句重新扫描 ProcArray。

REPEATABLE READ 的运行模型不同：

- 第一次需要 transaction snapshot 时取一次。
- 把 snapshot 复制到事务生命周期内存。
- 后续语句继续使用同一个 `CurrentSnapshot`。
- 直到事务结束才释放它对 `xmin` horizon 的影响。

SERIALIZABLE 在这个基础上继续加东西。它不是每条语句取新 snapshot。

它也不是把 `xip` 变成更强的数组。它仍然使用事务级稳定 snapshot。

区别是第一次 snapshot 建立时会进入 SSI。SSI 会创建 `SERIALIZABLEXACT`，记录读写依赖，使用 predicate lock 判断危险结构。

所以本节的线性主线是：

```text
第一次需要 snapshot
  -> IsolationUsesXactSnapshot()
  -> GetSnapshotData() 或 GetSerializableTransactionSnapshot()
  -> CopySnapshot()
  -> FirstXactSnapshot 注册
  -> 后续语句复用 CurrentSnapshot
  -> MyProc->xmin / RegisteredSnapshots 钉住 cleanup horizon
  -> SERIALIZABLE 额外维护 SSI 状态
  -> AtEOXact_Snapshot() 释放
```

这个主线同时解释两个 runtime 现象。第一个现象是稳定读：

- session A 在 REPEATABLE READ 中两次 `SELECT count(*)` 看到相同结果。
- 即使 session B 已经插入并提交。

第二个现象是 cleanup 成本：

- session A 什么也不做，只要事务还没结束，它的 `backend_xmin` 就可能让 VACUUM 不能移除旧 tuple version。
- 如果 isolation 是 SERIALIZABLE，还可能保留 SIREAD lock 和 conflict 状态直到事务安全释放。

这两个现象来自同一个设计选择。不要把它们拆成两个互不相关的机制。

稳定读的代价就是延迟回收。SSI 的代价就是把稳定读的重叠区间变成需要记录和检查的依赖图。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`GetTransactionSnapshot()` 在 `IsolationUsesXactSnapshot()` 为真时，把第一次 MVCC snapshot 复制成 `FirstXactSnapshot` 并注册到 `RegisteredSnapshots`；后续语句复用这个 snapshot 的 `xip/xmin/xmax`，只让命令 ID 随 CCI 推进；事务结束时 `AtEOXact_Snapshot()` 拆除注册，让 `MyProc->xmin` 和 SSI 状态可以释放。
```

这个模型里有三个时间轴。第一条时间轴是用户语义。

- 事务开始。
- 第一条语句读取。
- 其他事务提交新版本。
- 本事务第二条语句仍然读取第一次 snapshot 的世界。
- 事务提交或回滚。

第二条时间轴是 snapshot 状态。

- `FirstSnapshotSet` 初始为 false。
- 第一次调用 `GetTransactionSnapshot()` 后变为 true。
- RR/SERIALIZABLE 下 `CurrentSnapshot` 指向 copied snapshot。
- `FirstXactSnapshot` 保存这个对象。
- `RegisteredSnapshots` 记录它的 `xmin`。
- 后续 `GetTransactionSnapshot()` 直接返回 `CurrentSnapshot`。

第三条时间轴是系统成本。

- `MyProc->xmin` 对外声明本 backend 仍需要老版本。
- `GlobalVis*` 和 VACUUM horizon 不能越过可能仍被这个 snapshot 需要的 XID。
- SERIALIZABLE 的 `MySerializableXact`、predicate lock、conflict edge 也不能随 statement 结束立即消失。

这就是矛盾：

- 稳定读要求 snapshot 语义不随外部提交变化。
- cleanup 要求最老 snapshot 尽快消失。
- SERIALIZABLE correctness 要求重叠读写关系被保留到能判断是否安全。
- 三者无法同时最小化。

PostgreSQL 的选择是：

- 语义优先。
- 稳定 snapshot 生命周期明确绑定事务。
- 通过 `xmin` horizon、snapshot refcount、SSI cleanup 和 safe snapshot 优化降低成本。

这也解释为什么长事务是 MVCC 系统的操作风险。它不是“事务还在运行”这么简单。

它是在对系统声明：

- 我仍然可能读取一个旧时间点。
- 不要移除这个时间点可能需要的旧版本。
- 如果我是 SERIALIZABLE，还要保留与我重叠的读写依赖。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/xact.h` | 先看 `IsolationUsesXactSnapshot()` 和隔离级别分界。 |
| 2 | `src/include/utils/snapshot.h` | 明确 transaction snapshot、registered snapshot 和 `xmin` 字段。 |
| 3 | `src/include/utils/snapmgr.h` | 对照 snapshot manager 暴露给调用者的 API。 |
| 4 | `src/backend/utils/time/snapmgr.c` | 跟 `GetTransactionSnapshot()`、`CopySnapshot()`、`RegisterSnapshot()` 和 `AtEOXact_Snapshot()`。 |
| 5 | `src/backend/storage/ipc/procarray.c` | 看事务级 snapshot 如何影响 `MyProc->xmin` 和 cleanup horizon。 |
| 6 | `src/backend/storage/lmgr/predicate.c` | 看 SERIALIZABLE 在稳定 snapshot 外增加的 SSI 状态。 |
| 7 | `src/backend/access/transam/xact.c` | 核对事务开始、命令计数和事务结束 cleanup 边界。 |
| 8 | `src/backend/access/heap/heapam_visibility.c` | 最后回到 heap visibility，确认稳定 snapshot 被怎样消费。 |

## 4. 可复现 runtime 现象

先看稳定读。准备表：

```sql
DROP TABLE IF EXISTS rr_demo;
CREATE TABLE rr_demo(id int primary key, note text);
INSERT INTO rr_demo VALUES (1, 'old');
```

session A：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM rr_demo;
```

结果是 `1`。session B：

```sql
INSERT INTO rr_demo VALUES (2, 'new');
COMMIT;
```

session A 再执行：

```sql
SELECT count(*) FROM rr_demo;
```

仍然是 `1`。这不是 executor cache。

也不是表扫描没重新执行。源码解释是：

- session A 第一次 `SELECT` 调 `GetTransactionSnapshot()`。
- `IsolationUsesXactSnapshot()` 为真。
- `GetSnapshotData()` 记录当时的 `xmax` 和 running set。
- `CopySnapshot()` 把结果复制到 `TopTransactionContext`。
- 第二次 `SELECT` 调 `GetTransactionSnapshot()` 时直接返回 `CurrentSnapshot`。
- session B 的 XID 如果大于等于 A 的 snapshot `xmax`，对 A 不可见。

把 session A 换成 READ COMMITTED：

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT count(*) FROM rr_demo;
```

session B 插入提交后，session A 第二次 `SELECT` 会看到新行。源码差异只有一个关键分支：

- READ COMMITTED 下 `IsolationUsesXactSnapshot()` 为 false。
- 第二次 `GetTransactionSnapshot()` 会再次调用 `GetSnapshotData()`。

再看 cleanup 现象。session A：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM rr_demo WHERE id = 1;
```

保持事务不结束。session B：

```sql
UPDATE rr_demo SET note = 'dead-to-new-snapshots' WHERE id = 1;
VACUUM rr_demo;
```

此时旧版本可能已经对新 snapshot 不可见。但对 session A 的旧 snapshot 仍可能可见。

因此 VACUUM 不能简单移除它。可以在第三个 session 观察：

```sql
SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY xact_start;
```

这个视图不是完整因果证明。但它能看到一个关键事实：

- 长事务 backend 广告了 `backend_xmin`。
- 这个 xmin 是 cleanup horizon 的重要输入。

如果表足够大，也可以观察：

```sql
SELECT n_dead_tup, vacuum_count, autovacuum_count
FROM pg_stat_user_tables
WHERE relname = 'rr_demo';
```

`n_dead_tup` 是估算，不是精确物理 tuple 数。它只能作为现象入口。

真正解释要回到 `snapmgr.c` 和 `procarray.c`。

## 5. 关键状态与边界

### 5.1 `IsolationUsesXactSnapshot()`

`xact.h` 中的宏把隔离级别分成两类。READ COMMITTED 不使用事务级 snapshot。

REPEATABLE READ 和 SERIALIZABLE 使用事务级 snapshot。这个宏不是一个普通 helper。

它定义了本节的语义分界：

- 为 false：每次需要 transaction snapshot 都可以重新调用 `GetSnapshotData()`。
- 为 true：第一次 snapshot 之后，事务必须继续使用同一个 snapshot。

注意名字里的 `XactSnapshot`。它不是说事务开始时立刻创建 snapshot。

PostgreSQL 通常是 lazy 的。真正创建发生在第一次需要 snapshot 时。

所以这个事实很重要：

- `BEGIN ISOLATION LEVEL REPEATABLE READ;` 本身不一定已经取 snapshot。
- 第一条需要 MVCC snapshot 的语句才会触发。

这可以解释一些看起来奇怪的时间点。事务开始和 snapshot 创建不是同一时刻。

稳定读承诺从第一次 snapshot 开始。

### 5.2 `FirstSnapshotSet`

`FirstSnapshotSet` 是 backend-local static 状态。它表示当前事务是否已经完成第一次 transaction snapshot 建立。

第一次进入 `GetTransactionSnapshot()`：

- 如果它为 false，就进入初始化分支。
- 代码会先 `InvalidateCatalogSnapshot()`。
- 然后根据隔离级别选择 snapshot 获取方式。
- 最后设置 `FirstSnapshotSet = true`。

后续调用：

- 如果是 RR/SERIALIZABLE，直接返回 `CurrentSnapshot`。
- 如果是 READ COMMITTED，重新调用 `GetSnapshotData()`。

所以 `FirstSnapshotSet` 是 snapshot manager 的本地状态机。它不是 shared memory 状态。

别的 backend 看不到它。别的 backend 只能看到它造成的外部效果：

- `MyProc->xmin`
- ProcArray 中的 xmin horizon
- tuple version 是否仍不能回收

### 5.3 `FirstXactSnapshot`

`FirstXactSnapshot` 只在 transaction-snapshot mode 下使用。它保存第一次 snapshot 的私有副本。

这个副本有几个特征：

- 内存分配在 `TopTransactionContext`。
- `copied = true`。
- `regd_count` 会增加。
- 它会被加入 `RegisteredSnapshots`。

为什么不能直接使用 `CurrentSnapshotData`？因为源码注释已经给出原因：

- `GetTransactionSnapshot()` 返回的静态 storage 会被未来调用和 CCI 修改。
- 事务级 snapshot 必须活到事务结束。
- 所以要 `CopySnapshot()`。

`FirstXactSnapshot` 是 snapshot lifetime 的 owner 标记。它不是外部 API。

它帮助 `AtEOXact_Snapshot()` 找到并移除那份事务级引用。

### 5.4 `CopySnapshot()`

`CopySnapshot()` 做的事很具体：

- 计算 `SnapshotData` 加 `xip` 数组所需大小。
- 如果需要，继续加 `subxip` 数组大小。
- 在 `TopTransactionContext` 分配一个连续内存块。
- 复制 `SnapshotData` 本体。
- 复制 `xip`。
- 在非 overflow 或 recovery snapshot 情况下复制 `subxip`。
- 清零 `active_count` 和 `regd_count`。
- 标记 `copied = true`。

这个函数体现了一个边界：

- snapshot 不是只复制一个 struct。
- 它还要复制运行事务数组。

这就是为什么长事务不是零成本。它不仅保留一个指针。

它保留了一份当时的 running set。不过最大成本通常不在这块本地内存。

真正昂贵的是它保存的 `xmin` 语义。这个 `xmin` 会让旧 tuple version 继续被视为可能需要。

### 5.5 `RegisteredSnapshots`

`RegisteredSnapshots` 是 snapmgr 中的 pairing heap。它按 snapshot 的 `xmin` 排序。

目的很实际：

- 快速找到当前 backend 持有的最老 snapshot。
- 当某个 snapshot 被 unregister 或 active stack 为空时，尝试推进 `MyProc->xmin`。

`FirstXactSnapshot` 加入这个 heap。Catalog snapshot、exported snapshot、显式注册的 snapshot 也可能加入。

所以一个 backend 的 `MyProc->xmin` 不是只由 `FirstXactSnapshot` 决定。它由当前仍 active 或 registered 的 snapshot 中最老的 `xmin` 决定。

但是在 RR/SERIALIZABLE 事务里，`FirstXactSnapshot` 通常会成为持续时间最长的那个。

### 5.6 `MyProc->xmin`

`MyProc->xmin` 是 shared memory 中对外发布的 horizon。它告诉其他 backend：

- 我可能仍需要看见不晚于这个边界的 tuple version。

`GetSnapshotData()` 会在合适时设置：

- `MyProc->xmin = TransactionXmin = xmin`

`SnapshotResetXmin()` 会在没有 active snapshot 且 registered heap 允许时清除或推进它。事务结束路径会在 ProcArray 结束事务时清掉相关状态。

`AtEOXact_Snapshot()` 注释说明：

- 如果 `resetXmin` 为真，`ProcArrayEndTransaction()` 已经在它之前重置了 `MyProc->xmin`。
- snapshot manager 不需要再碰 xmin。

这正是 lifecycle 边界。snapshot manager 负责管理本地引用。

ProcArray 负责对外发布和清理 shared state。

### 5.7 `curcid`

`SnapshotData->curcid` 表示当前事务中哪些 command 的修改对这个 snapshot 可见。字段注释说：

- `CID < curcid` 的当前事务修改可见。

这一点在 10 节会详细讲。本节只需要知道：

- RR/SERIALIZABLE 固定的是外部事务集合，也就是 `xmin/xmax/xip/subxip`。
- 同一事务内部，`curcid` 仍会随 `CommandCounterIncrement()` 推进。

这解释一个常见现象：

- RR 事务不会看见别的事务新提交。
- 但会看见自己上一条语句的写入。

这不是矛盾。这是 snapshot 的两个维度：

- external visibility 固定。
- self visibility 随 command id 推进。

## 6. 主流程源码 walkthrough

### 6.1 第一次语句进入 snapshot 获取

普通 SQL 进入 executor 前会 push active snapshot。常见入口包括：

- `postgres.c`
- `pquery.c`
- `PortalStart()`
- `PushActiveSnapshot(GetTransactionSnapshot())`

不同执行路径入口不完全相同。但都会回到 `GetTransactionSnapshot()`。

第一次调用时：

```text
GetTransactionSnapshot()
  -> HistoricSnapshotActive() 检查
  -> !FirstSnapshotSet
  -> InvalidateCatalogSnapshot()
  -> IsInParallelMode() 检查
  -> IsolationUsesXactSnapshot()
```

这里有两个边界。第一，logical decoding 的 historic snapshot 是旁路。

它只能用于 catalog access，不是普通 query snapshot。第二，parallel operation 已经开始后不能再建立新的 transaction snapshot。

因为事务级 snapshot 是 leader 和 worker 需要一致继承的状态。

### 6.2 READ COMMITTED 的旁路

如果 `IsolationUsesXactSnapshot()` 为 false：

```text
CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData)
FirstSnapshotSet = true
return CurrentSnapshot
```

后续再次调用时：

```text
if (IsolationUsesXactSnapshot())
    return CurrentSnapshot

InvalidateCatalogSnapshot()
CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData)
return CurrentSnapshot
```

因为 READ COMMITTED 下条件为 false，所以它会重新取 snapshot。这解释 READ COMMITTED 的 statement snapshot。

每条语句都能看到语句开始前已经提交的事务。旧 statement snapshot 不需要作为事务级承诺持有到结束。

它的 cleanup horizon 压力通常小得多。

### 6.3 RR 第一次 snapshot

如果 `IsolationUsesXactSnapshot()` 为 true 且不是 SERIALIZABLE：

```text
CurrentSnapshot = GetSnapshotData(&CurrentSnapshotData)
CurrentSnapshot = CopySnapshot(CurrentSnapshot)
FirstXactSnapshot = CurrentSnapshot
FirstXactSnapshot->regd_count++
pairingheap_add(&RegisteredSnapshots, &FirstXactSnapshot->ph_node)
FirstSnapshotSet = true
return CurrentSnapshot
```

这里的关键不是“取到了 snapshot”。关键是“复制并注册”。

`GetSnapshotData()` 返回的是静态 `CurrentSnapshotData`。它会被未来调用复用。

RR 需要一个不会被下一次 statement 覆盖的对象。所以必须 `CopySnapshot()`。

注册动作让 snapshot manager 知道：

- 这个 snapshot 的 `xmin` 不能被忽略。
- 即使 active snapshot stack 暂时为空，它仍是事务级承诺。

### 6.4 后续语句复用

RR 后续调用：

```text
GetTransactionSnapshot()
  -> FirstSnapshotSet 已为 true
  -> IsolationUsesXactSnapshot() 为 true
  -> return CurrentSnapshot
```

没有重新扫描 ProcArray。没有把 session B 新提交的 XID 放入一个新 snapshot。

也不会推进 `xmax` 到新值。所以第二条 `SELECT` 对 session B 的新行仍不可见。

注意：

- executor 仍会重新执行 plan。
- buffer manager 仍可能读到包含新 tuple 的 page。
- heap visibility 会对新 tuple 调 `HeapTupleSatisfiesMVCC()`。
- 只是该 tuple 的 `xmin` 对这个旧 snapshot 不可见。

因此稳定读不是“不读取新页”。它是在读取后用旧 snapshot 判断不可见。

### 6.5 SERIALIZABLE 第一次 snapshot

如果 isolation 是 SERIALIZABLE，第一次分支改成：

```text
CurrentSnapshot = GetSerializableTransactionSnapshot(&CurrentSnapshotData)
CurrentSnapshot = CopySnapshot(CurrentSnapshot)
FirstXactSnapshot = CurrentSnapshot
...
```

`GetSerializableTransactionSnapshot()` 在 `predicate.c`。它先断言 `IsolationIsSerializable()`。

如果 recovery 仍在进行，会报错。原因是 hot standby 不能使用 serializable mode 的 SSI 机制。

READ ONLY DEFERRABLE 是特殊优化。如果事务是 `SERIALIZABLE READ ONLY DEFERRABLE`：

- 可以等待一个 safe snapshot。
- 一旦拿到 safe snapshot，就可以避免后续 SSI overhead。

普通 SERIALIZABLE 进入 `GetSerializableTransactionSnapshotInt()`。这个函数会：

- 获取当前 VXID。
- 获取 `SerializableXactHashLock`。
- 创建或复用 `SERIALIZABLEXACT`。
- 在锁保护下调用 `GetSnapshotData()`。
- 初始化 `sxact->xmin = snapshot->xmin`。
- 记录 `lastCommitBeforeSnapshot`。
- 如果是 read-only，登记可能不安全的 concurrent writable transaction。
- 如果是 read-write，增加 `PredXact->WritableSxactCount`。
- 设置 `MySerializableXact`。
- 创建本地 predicate lock hash。

这里的正确性边界是：

- SSI 必须把 snapshot 创建和 serializable transaction 注册放在同一个互斥语义内。
- 否则会漏掉与 snapshot 创建同时发生的 rw-dependency。

所以 Serializable 的成本不是数组更大。它是额外的共享状态、锁、predicate target、conflict edge 和 commit-time 检查。

### 6.6 snapshot 进入 heap visibility

heap scan 最终把 active snapshot 传给 `HeapTupleSatisfiesMVCC()`。这个函数只看 snapshot 内容。

例如：

- 如果 tuple `xmin` 大于等于 snapshot `xmax`，它对该 snapshot 仍在未来。
- 如果 tuple `xmin` 在 `xip` 中，创建它的事务在 snapshot 时仍 running。
- 如果 tuple `xmin` 已提交但在 snapshot 中仍被视为 running，也不能可见。

RR 的稳定读就在这里落地。第二条语句使用同一个 snapshot。

所以判断结果仍然基于第一次语句看到的 `xmax/xip`。

### 6.7 cleanup horizon 的传播

`GetSnapshotData()` 构造 snapshot 时会计算 `xmin`。它还会更新：

- `TransactionXmin`
- `RecentXmin`
- `MyProc->xmin`
- `GlobalVisSharedRels`
- `GlobalVisCatalogRels`
- `GlobalVisDataRels`
- `GlobalVisTempRels`

这说明 snapshot 获取不是只返回一个对象。它还发布了本 backend 的保留需求。

VACUUM、pruning、visibility map、global visibility test 相关路径都会受到 horizon 影响。RR 事务后续语句不重新取 snapshot。

但它已经注册的 `FirstXactSnapshot` 仍在。只要它还在 `RegisteredSnapshots` 中，它的 `xmin` 就可能阻止 `MyProc->xmin` 前进。

其他 backend 在计算全局 oldest non-removable XID 时会看到这个影响。这就是稳定读转化为 cleanup 成本的路径。

### 6.8 事务结束

commit 路径和 abort 路径最终都会进入事务结束 cleanup。与本节相关的是：

```text
CommitTransaction()
  -> AtEOXact_Snapshot(true, ...)
AbortTransaction()
  -> AtEOXact_Snapshot(false, ...)
```

不同分支里 `resetXmin` 参数不同。但共同目标是：

- 移除 `FirstXactSnapshot` 在 `RegisteredSnapshots` 中的引用。
- 清理 exported snapshot 文件记录。
- 检查是否还有未释放的 snapshot。
- 重置 `CurrentSnapshot`、`SecondarySnapshot`、`CatalogSnapshot` 等本地状态。
- 让 `FirstSnapshotSet` 回到 false。

`AtEOXact_Snapshot()` 对 `FirstXactSnapshot` 不调用 `FreeSnapshot()`。原因是它分配在 `TopTransactionContext`。

事务结束时整个 context 会被 reset。这体现了 PostgreSQL 常见的 ownership 分层：

- MemoryContext 管内存释放。
- snapshot manager 管引用和注册关系。
- ProcArray 管 shared `xmin` 发布。
- xact.c 管事务结束调用顺序。

## 7. 生命周期 / ownership / cleanup

谁创建？

- 第一次 `GetTransactionSnapshot()` 创建 transaction snapshot。
- READ COMMITTED 创建 statement snapshot。
- RR/SERIALIZABLE 创建并复制 transaction snapshot。
- SERIALIZABLE 还创建 `SERIALIZABLEXACT`。

谁持有？

- `CurrentSnapshot` 指向当前 transaction snapshot。
- `FirstXactSnapshot` 保存事务级 owner 引用。
- `RegisteredSnapshots` 用 `regd_count` 表示它仍在保护 `xmin`。
- Active snapshot stack 用 `active_count` 表示 executor 正在使用。

谁能访问？

- snapshot 对象本身是 backend-local 内存。
- 其他 backend 不能直接读它的 `xip` 数组。
- 其他 backend 只能通过 `PGPROC->xmin` 看到它对 cleanup horizon 的影响。
- SSI 状态则在 predicate lock shared memory 中被其他 serializable transaction 间接观察。

谁释放？

- 显式 `UnregisterSnapshot()` 释放普通注册引用。
- `PopActiveSnapshot()` 释放 active 引用。
- RR/SERIALIZABLE 的 `FirstXactSnapshot` 由 `AtEOXact_Snapshot()` 拆除。
- 内存由 `TopTransactionContext` 在事务结束时回收。

ERROR 时怎么办？

- 如果 ERROR 导致当前事务 abort，xact abort cleanup 会调用 snapshot manager cleanup。
- active snapshot stack 若没正常弹出，`AtEOXact_Snapshot()` 会发 warning 并重置本地结构。
- exported snapshot 文件删除失败只是 WARNING，因为已经太晚不能 abort 当前 cleanup。

为什么不用 ResourceOwner？

- snapshot 内存主要由 MemoryContext 管。
- snapshot 注册引用由 snapmgr 的 refcount 和 pairing heap 管。
- buffer pin、catcache refcount 这类外部资源才更典型地交给 ResourceOwner。

这节课要记住：

- `FirstXactSnapshot` 的生命周期不是“某条语句结束”。
- 它是“事务结束”。
- 这是 REPEATABLE READ 语义的实现边界。

## 8. 正确性机制层次

第一层是 MVCC snapshot。它回答：

- 哪些外部事务在 snapshot 创建时还在运行。
- 哪些 XID 对我可见或不可见。

它不能回答：

- 是否存在 serializable dangerous structure。
- 是否有 predicate conflict。
- 是否可以回收所有旧版本。

第二层是 command id。RR 固定外部世界。

但当前事务内部仍要按 command 边界推进。`SnapshotSetCommandId()` 会把 `CommandCounterIncrement()` 的结果传播到静态 snapshot。

所以同一事务中：

- 上一条语句插入的行可以被下一条语句看见。
- 当前语句正在产生的新 tuple 不能被同一语句普通扫描反复看见。

第三层是 ProcArray xmin。`MyProc->xmin` 是 cleanup horizon 的共享信号。

它不能表达完整 snapshot。它只表达：

- 不要回收这个 backend 可能还需要的旧版本边界。

第四层是 SSI。SERIALIZABLE 使用同一个稳定 snapshot。

但额外记录：

- rw-conflict in。
- rw-conflict out。
- predicate lock target。
- read-only safe 状态。
- commit sequence。

它保证的不是“更旧或更新的 snapshot”。它保证的是并发结果等价于某个串行顺序。

第五层是事务结束顺序。只有当事务结束 cleanup 执行后：

- 本地 snapshot 引用消失。
- `MyProc->xmin` 可以清空。
- predicate lock 可以释放或转入 finished 状态。
- cleanup horizon 才可能前进。

这些机制共同成立。少任何一层都会破坏本节主问题。

## 9. 异常路径 / fallback

### 9.1 parallel operation 中建立 snapshot

`GetTransactionSnapshot()` 第一次建立 snapshot 时会检查 `IsInParallelMode()`。如果已经在 parallel operation 中，会报错：

- 不能在并行操作期间获取 query snapshot。

Serializable 内部也有类似检查：

- 不能在 parallel operation 已开始后建立 serializable snapshot。

原因不是 parallel scan 本身不能读 snapshot。原因是 transaction snapshot 和 serializable state 必须在 leader 和 worker 之间一致。

parallel worker 使用 leader 序列化传入的 snapshot。不能让 worker 临时创造一个新的事务级 snapshot。

### 9.2 hot standby 上 SERIALIZABLE

`GetSerializableTransactionSnapshot()` 中检查 `RecoveryInProgress()`。如果在 recovery 中尝试 SERIALIZABLE，会报错。

错误提示建议使用 REPEATABLE READ。这说明：

- hot standby 可以提供稳定 snapshot 读。
- 但不能提供完整 SSI predicate lock / conflict tracking。

所以 SERIALIZABLE 的额外语义不是 snapshot 字段能单独表达的。

### 9.3 READ ONLY DEFERRABLE

`SERIALIZABLE READ ONLY DEFERRABLE` 是特殊路径。系统可以等待 safe snapshot。

一旦拿到 safe snapshot：

- 这个 read-only transaction 不会成为危险结构的一部分。
- 可以释放或避免大量 SSI overhead。

这是一种用等待换运行期成本的设计。它没有改变 RR 稳定 snapshot 的基本事实。

它只是把 Serializable 的 conflict 追踪成本降下来。

### 9.4 imported snapshot

`SetTransactionSnapshot()` 允许导入外部 snapshot。它必须修正 `MyProc->xmin`。

`ProcArrayInstallImportedXmin()` 会检查 source transaction 是否仍在运行。如果 source 已经结束，导入失败。

原因很直接：

- 如果没有任何仍在运行的进程或 slot 保护那个 xmin，系统不能保证旧版本还在。

这再次说明：

- snapshot 文件本身不是保留旧版本的魔法。
- 必须有共享 horizon 保护。

### 9.5 exported snapshot 文件泄漏

`AtEOXact_Snapshot()` 会删除 exported snapshot 文件。unlink 失败只报 WARNING。

因为事务已经在结束阶段。这个文件泄漏通常不是 visibility correctness 风险。

真正的 visibility 保护来自运行时 `xmin`。文件只是导入 snapshot 所需的描述。

### 9.6 OOM 与 snapshot copy

`CopySnapshot()` 在 `TopTransactionContext` 分配内存。如果分配失败，会 ERROR。

因为还没建立稳定 snapshot 承诺。事务进入 abort cleanup 后，MemoryContext 会回收已分配状态。

这就是为什么 snapshot copy 放在事务上下文中是合理的。它让 ERROR cleanup 简单可靠。

## 10. 成本、资源与跨模块传播

### 10.1 cleanup horizon 成本

RR 稳定 snapshot 最直接的成本是 `xmin` 被钉住。传播路径：

```text
FirstXactSnapshot->xmin
  -> RegisteredSnapshots 最老项
  -> MyProc->xmin / TransactionXmin
  -> ProcArray horizon computation
  -> GlobalVis* / oldest non-removable XID
  -> VACUUM / pruning / tuple removal 决策
```

这个成本随长事务持续时间扩大。不是随该事务执行 SQL 数量扩大。

一个 idle in transaction 的 RR 会话也能制造压力。如果系统不断 UPDATE/DELETE，同一个旧 snapshot 会让更多 dead tuple 无法物理回收。

表现可能是：

- 表膨胀。
- index 膨胀。
- vacuum 反复扫描但移除少。
- wraparound 风险增加。

### 10.2 ProcArray 与 snapshot 获取成本

READ COMMITTED 每条语句都取 snapshot。RR 后续语句复用 snapshot，减少重复 ProcArray scan。

这看起来是性能优势。但别把它理解成 RR 更轻。

RR 减少的是同一事务内的 snapshot 重建成本。它增加的是旧 snapshot 持有时间。

在短事务里，差异可能不明显。在长事务里，cleanup 成本通常比少扫几次 ProcArray 更重要。

### 10.3 `CopySnapshot()` 本地内存

`CopySnapshot()` 会复制 `xip` 和部分 `subxip`。成本与当时 running XID 数量相关。

如果 backend 数很多，第一次 RR snapshot 复制的数组也可能更大。但这通常不是最主要瓶颈。

本地内存会随事务结束释放。旧版本保留和 VACUUM horizon 才会影响全局。

### 10.4 Serializable shared memory 成本

SERIALIZABLE 额外使用 predicate lock shared memory。成本来源包括：

- `SerializableXactHashLock` 竞争。
- predicate lock hash partition lock。
- target 粒度从 tuple/page/relation 可能提升。
- conflict edge 维护。
- committed serializable transaction 延迟释放。
- `PreCommit_CheckForSerializationFailure()` 检查危险结构。

这些成本随并发 serializable transaction、读集合大小、写集合冲突、索引访问路径和事务持续时间扩大。长 SERIALIZABLE read-only 事务如果不是 safe，可能保留大量 possible unsafe conflict。

### 10.5 replication slot 与 horizon 叠加

snapshot `xmin` 不是唯一 cleanup 限制。`procarray.c` 还考虑 replication slot xmin 和 catalog xmin。

所以生产诊断时不能看到一个长 RR 事务就断定它是唯一原因。需要同时检查：

- `pg_stat_activity.backend_xmin`
- replication slot `xmin` / `catalog_xmin`
- prepared transaction
- autovacuum 进度
- 表级 churn

本节只强调 RR snapshot 是重要来源之一。不要把所有 bloat 都归因于它。

### 10.6 后台进程参与

VACUUM 和 autovacuum 会消费 horizon。checkpointer 不直接解决 snapshot horizon。

walwriter 也不释放 snapshot。startup process 在 hot standby 会维护 KnownAssignedXids。

logical decoding 会用自己的 historic snapshot 和 slot xmin 机制。所以 cleanup 成本不是“等 checkpoint 就好”。

释放稳定读成本的关键通常是：

- 结束长事务。
- 释放 exported/imported snapshot 依赖。
- 推进 replication slot。
- 让 VACUUM 重新获得可回收边界。

## 11. 观测与诊断入口

### 11.1 看稳定读

最直接实验是两 session count。关键不是 count 本身。

关键是把现象回到源码：

- A 第一次 `SELECT` 建立 `FirstXactSnapshot`。
- B 插入提交后 XID 已完成。
- A 第二次 `SELECT` 没有重新调用 `GetSnapshotData()`。
- heap visibility 用旧 `xmax/xip` 判断新 tuple 不可见。

可以用 `EXPLAIN (ANALYZE, BUFFERS)` 证明第二次仍执行扫描。这排除“查询结果缓存”的误解。

### 11.2 看 cleanup horizon

查看长事务：

```sql
SELECT pid, usename, state, backend_xmin, xact_start, now() - xact_start AS age, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY xact_start;
```

解释边界：

- `backend_xmin` 能看到 backend 发布的 xmin。
- 它不显示完整 `xip`。
- 它不显示 `FirstXactSnapshot` 指针。
- 它不告诉你该 xmin 是否是唯一阻塞因素。

再看表级估算：

```sql
SELECT relname, n_live_tup, n_dead_tup, vacuum_count, autovacuum_count
FROM pg_stat_user_tables
WHERE relname = 'rr_demo';
```

解释边界：

- `n_dead_tup` 是统计估算。
- 需要结合 workload、ANALYZE 时机和 VACUUM 日志。

### 11.3 看 SERIALIZABLE predicate locks

Serializable 读事务中可以观察：

```sql
SELECT locktype, mode, relation::regclass, page, tuple, pid
FROM pg_locks
WHERE mode = 'SIReadLock';
```

解释边界：

- `SIReadLock` 不是阻塞普通写的锁。
- 它用于 SSI conflict 检测。
- 它可能被提升到 page 或 relation 粒度。
- 它的存在说明 Serializable 读集合仍需要被 conflict detector 考虑。

### 11.4 看 serialization failure

Serializable 写冲突常见错误：

```text
could not serialize access due to read/write dependencies among transactions
```

源码入口：

- `CheckForSerializableConflictIn()`
- `CheckForSerializableConflictOut()`
- `PreCommit_CheckForSerializationFailure()`

诊断时不要只看报错语句。危险结构可能由更早的读建立。

稳定 snapshot 让整个事务区间都参与判断。

### 11.5 gdb / 断点入口

源码跟读断点建议：

- `GetTransactionSnapshot`
- `CopySnapshot`
- `GetSnapshotData`
- `SnapshotResetXmin`
- `AtEOXact_Snapshot`
- `GetSerializableTransactionSnapshot`
- `PreCommit_CheckForSerializationFailure`

观察变量：

- `FirstSnapshotSet`
- `FirstXactSnapshot`
- `CurrentSnapshot->xmin`
- `CurrentSnapshot->xmax`
- `CurrentSnapshot->curcid`
- `MyProc->xmin`
- `MySerializableXact`

不要在生产系统直接 attach 长时间停住 backend。停住 backend 本身也会延长事务和 snapshot 生命周期。

## 12. 常见误区

误区一：

- “REPEATABLE READ 的 snapshot 在 BEGIN 时创建。”

更准确：

- 第一次需要 transaction snapshot 时创建。

误区二：

- “稳定读说明 executor 没重新扫描表。”

更准确：

- executor 可以重新扫描。
- heap visibility 用旧 snapshot 过滤新版本。

误区三：

- “RR 事务不会看到自己的新写入。”

更准确：

- 外部事务集合固定。
- 当前事务自己的 command visibility 由 `curcid` 推进。

误区四：

- “SERIALIZABLE 就是更严格的 `xip` 数组。”

更准确：

- 它使用稳定 snapshot，加上 SSI predicate lock 和 conflict graph。

误区五：

- “`backend_xmin` 一定就是 bloat 的根因。”

更准确：

- 它是强信号。
- 仍需排查 replication slot、prepared transaction、autovacuum 配置和 workload churn。

误区六：

- “事务结束后 snapshot 内存必须逐个 free。”

更准确：

- `FirstXactSnapshot` 的内存在 `TopTransactionContext` 中。
- `AtEOXact_Snapshot()` 主要拆注册关系。
- 内存随事务 context reset 消失。

## 13. 课堂实验

实验一：稳定读与源码断点。步骤：

1. 在 session A 开启 `BEGIN ISOLATION LEVEL REPEATABLE READ`。
2. 在 `GetTransactionSnapshot()` 和 `CopySnapshot()` 打断点。
3. 执行第一次 `SELECT`，记录 `snapshot->xmin`、`snapshot->xmax`。
4. session B 插入并提交。
5. session A 再执行 `SELECT`，观察不会再次进入 `CopySnapshot()`。
6. 回到 `HeapTupleSatisfiesMVCC()` 看新 tuple 为什么不可见。

实验二：cleanup horizon。步骤：

1. session A 开启 RR 并读取一行后保持事务。
2. session B 对同一表大量 UPDATE。
3. session B 执行 VACUUM。
4. 第三 session 查询 `pg_stat_activity.backend_xmin`。
5. 结束 session A。
6. 再执行 VACUUM，比较 dead tuple 变化和日志。

结论要回源码：

- `FirstXactSnapshot` 注册。
- `MyProc->xmin` 发布。
- 事务结束才释放。

实验三：SERIALIZABLE predicate lock。步骤：

1. session A 开启 `BEGIN ISOLATION LEVEL SERIALIZABLE`。
2. 执行一个范围查询。
3. 查询 `pg_locks` 中 `SIReadLock`。
4. session B 执行可能冲突的写入。
5. 尝试让两个事务以不同顺序提交。
6. 观察是否出现 serialization failure。

源码回看：

- `GetSerializableTransactionSnapshot()`
- `PredicateLockRelation()` / `PredicateLockPage()` / `PredicateLockTID()`
- `PreCommit_CheckForSerializationFailure()`

## 14. 讨论题

1. 为什么 RR 不在 `BEGIN` 时立刻取 snapshot，而是第一次需要时取？
2. 为什么事务级 snapshot 要 `CopySnapshot()`，不能长期指向 `CurrentSnapshotData`？
3. 为什么 `FirstXactSnapshot` 要进入 `RegisteredSnapshots`？
4. `MyProc->xmin` 能表达完整 snapshot 吗？如果不能，它表达了什么？
5. 为什么 RR 能看到自己上一条语句写入，却看不到别人提交的新行？
6. SERIALIZABLE 相比 RR 多出的正确性机制在哪里？
7. `SERIALIZABLE READ ONLY DEFERRABLE` 为什么可以用等待换掉部分 SSI 成本？
8. 诊断 bloat 时，为什么不能只看一个 `backend_xmin` 就下最终结论？

## 15. 本节小结

本节主问题是：

- 为什么 RR / SERIALIZABLE 把第一次 snapshot 固定到事务结束。

核心链路是：

```text
GetTransactionSnapshot()
  -> IsolationUsesXactSnapshot()
  -> GetSnapshotData() 或 GetSerializableTransactionSnapshot()
  -> CopySnapshot()
  -> FirstXactSnapshot
  -> RegisteredSnapshots
  -> 后续语句复用 CurrentSnapshot
  -> AtEOXact_Snapshot()
```

核心状态是：

- `FirstSnapshotSet` 决定是否已经建立第一次 snapshot。
- `FirstXactSnapshot` 保存事务级 snapshot。
- `RegisteredSnapshots` 用 `xmin` 排序管理本 backend 的 snapshot 保留需求。
- `MyProc->xmin` 把本地保留需求发布给 ProcArray。
- `MySerializableXact` 把 SERIALIZABLE 事务接入 SSI。

正确性边界是：

- RR 固定外部事务集合。
- command id 推进当前事务内部可见性。
- Serializable 在稳定 snapshot 外维护 predicate lock 和 conflict graph。

cleanup 边界是：

- statement 结束不释放 `FirstXactSnapshot`。
- 事务结束才由 `AtEOXact_Snapshot()` 拆注册。
- shared `xmin` 的清理由 ProcArray 事务结束路径配合完成。

可观测现象是：

- RR 同一事务内重复查询看不到别人后来提交的新行。
- 长 RR 事务的 `backend_xmin` 可能阻止 VACUUM 回收旧版本。
- SERIALIZABLE 可能出现 `SIReadLock` 和 serialization failure。

可迁移规律：

- MVCC 稳定读不是免费属性。
- 只要系统承诺一个旧时间点仍可读，就必须保留足够状态让这个旧时间点可解释。
- 在 PostgreSQL 中，这个状态从 `SnapshotData` 延伸到 `MyProc->xmin`、VACUUM horizon 和 SSI conflict graph。
