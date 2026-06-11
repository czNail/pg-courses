# PostgreSQL 只读事务、safe snapshot 与 DEFERRABLE

## 课程定位

前置知识：已经理解 `SERIALIZABLEXACT`、`rw-conflict`、SIREAD footprint、predicate lock 生命周期，以及 read-only 事务仍可能参与 dangerous structure。

本节唯一主问题：

```text
为什么 SERIALIZABLE READ ONLY DEFERRABLE 可以在开始阶段等待一个 safe snapshot，从而让后续只读执行避开 SSI 冲突检测成本？
```

核心矛盾：

```text
只读事务不会写数据，但它仍可能作为 dangerous structure 的 Tin；
如果系统能在启动时证明它的 snapshot safe，就可以用等待换取后续无 predicate lock / conflict tracking 的执行。
```

学完后应能独立判断：

- read-only serializable 为什么仍可能不 safe。
- safe snapshot 的含义是什么。
- DEFERRABLE 为什么只对 SERIALIZABLE READ ONLY 有意义。
- `GetSafeSnapshot()` 为什么循环获取 snapshot、等待 concurrent r/w transactions 完成并可能重试。
- `possibleUnsafeConflicts` 如何连接 read-only 事务和 active writable 事务。
- `pg_safe_snapshot_blocking_pids()` 能观察什么，不能观察什么。
- 为什么 safe 后可以 `ReleasePredicateLocks(false, true)`。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节已经讲了 writable SERIALIZABLE 的主线：

```text
snapshot
  -> SIREAD footprint
  -> rw-conflict
  -> dangerous structure
  -> commit / abort
```

本节换到只读事务。

只读事务不写。

它不会成为 writer。

但它可以成为 reader。

也就是：

```text
Tin -> Tpivot -> Tout
```

中的 `Tin`。

如果 `Tin` 是 read-only，PostgreSQL 有一个额外优化：

```text
只有当 Tout 在 Tin 获取 snapshot 前已经提交，read-only Tin 才可能参与异常。
```

如果系统能证明某个 read-only snapshot 不可能满足这种危险条件，它就是 safe snapshot。

普通 `SERIALIZABLE READ ONLY` 可以先运行，再在过程中被证明 safe 后释放跟踪。

`SERIALIZABLE READ ONLY DEFERRABLE` 则选择：

```text
启动时等待；
直到拿到 safe snapshot；
之后不再承担 SSI tracking 成本。
```

这是本节唯一主线。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
GetSerializableTransactionSnapshot() 发现 XactReadOnly && XactDeferrable 时进入 GetSafeSnapshot()；
GetSafeSnapshot() 创建 serializable snapshot，并把 concurrent writable sxacts 放入 possibleUnsafeConflicts；
如果仍有可能让该 read-only snapshot unsafe 的事务，就等待它们完成；
若期间被标记 RO_UNSAFE，则释放当前 SSI 状态并重试；
若被标记 RO_SAFE，则调用 ReleasePredicateLocks(false, true) 让后续只读执行避开 predicate lock tracking。
```

本节的核心矛盾是：

```text
立即开始只读事务能降低启动延迟；
但如果 snapshot unsafe，后续需要 SIREAD / conflict tracking；
等待 safe snapshot 会增加启动等待，却能降低长只读事务后续成本和失败风险。
```

这解释了 DEFERRABLE 的适用场景：

- 长只读报表。
- 备份导出。
- 一致性校验。
- 不希望中途 `40001` 的只读任务。

它不适合：

- 短 OLTP 点查。
- 读写事务。
- 非 SERIALIZABLE 隔离级别。
- 对启动延迟极端敏感的请求。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/predicate.c` | `GetSerializableTransactionSnapshot()`、`GetSafeSnapshot()`、`SetPossibleUnsafeConflict()`、`FlagSxactUnsafe()`。 |
| 2 | `src/include/storage/predicate_internals.h` | `possibleUnsafeConflicts`、`SXACT_FLAG_READ_ONLY`、`SXACT_FLAG_DEFERRABLE_WAITING`、`SXACT_FLAG_RO_SAFE`、`SXACT_FLAG_RO_UNSAFE`。 |
| 3 | `src/backend/utils/adt/lockfuncs.c` | `pg_safe_snapshot_blocking_pids()` 如何调用 `GetSafeSnapshotBlockingPids()`。 |
| 4 | `src/backend/storage/lmgr/README-SSI` | read-only optimization 和 DEFERRABLE 的设计理由。 |
| 5 | `src/backend/utils/time/snapmgr.c` | transaction snapshot 如何进入 SERIALIZABLE snapshot 获取。 |
| 6 | `src/backend/access/transam/xact.c` | `XactReadOnly`、`XactDeferrable` 的事务属性边界。 |
| 7 | `doc/src/sgml/mvcc.sgml` | 对 SERIALIZABLE READ ONLY DEFERRABLE 行为的用户可见说明。 |

推荐阅读顺序：

```text
predicate.c 的 GetSerializableTransactionSnapshot()
  -> GetSafeSnapshot()
  -> GetSerializableTransactionSnapshotInt()
  -> possibleUnsafeConflicts 的建立
  -> ReleasePredicateLocks(false, true)
  -> GetSafeSnapshotBlockingPids()
```

不要把 DEFERRABLE 理解成约束延迟检查。

这里讲的是事务属性：

```text
SERIALIZABLE READ ONLY DEFERRABLE
```

它服务 SSI safe snapshot。

## 4. 从 runtime 现象进入

准备表：

```sql
DROP TABLE IF EXISTS safe_demo;

CREATE TABLE safe_demo
(
    id int PRIMARY KEY,
    balance int NOT NULL
);

INSERT INTO safe_demo
SELECT g, 100
FROM generate_series(1, 1000) AS g;
```

Session A 启动一个 writable serializable transaction：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

UPDATE safe_demo
SET balance = balance + 1
WHERE id = 1;

-- 暂不提交。
```

Session B 启动：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;

SELECT sum(balance)
FROM safe_demo;
```

Session B 可能在第一条查询前等待。

这不是普通锁等待。

它在等 safe snapshot。

另一个会话可以观察：

```sql
SELECT pg_safe_snapshot_blocking_pids(<session_b_pid>);
```

如果 B 正在 `GetSafeSnapshot()` 中等待，可能返回阻塞它 safe snapshot 的 PID 列表。

当 Session A：

```sql
COMMIT;
```

或：

```sql
ROLLBACK;
```

B 可能继续，或者发现 snapshot unsafe 后重试获取新的 snapshot。

最终 B 运行时，一旦 snapshot safe，后续只读查询不需要普通 SIREAD tracking。

这就是 DEFERRABLE 的 runtime 现象：

```text
把不确定性集中在 transaction start；
用等待换后续稳定只读执行。
```

## 5. read-only 为什么仍可能 unsafe

只读事务不写。

它不会创建 incoming conflict 给别人。

但它可以读不到某个 concurrent writer 的结果。

所以它可以有：

```text
read-only Tin -> writable Tpivot
```

如果 Tpivot 又有：

```text
Tpivot -> Tout
```

并且 `Tout` 在 Tin 获取 snapshot 前已经提交，就可能出现 anomaly。

README-SSI 对 read-only 优化的核心判断是：

```text
如果 Tin 是 read-only，只有 Tout 在 Tin snapshot 前提交时才可能危险。
```

这让 read-only 可以优化。

但不是所有 read-only 都自动 safe。

一个 read-only transaction 启动时，如果已有 active writable serializable transaction，它们可能成为未来的 pivot 或 related transaction。

PostgreSQL 不能立刻断言 safe。

它需要等待这些 concurrent writable transaction 结束，并看它们是否形成了会让 read-only unsafe 的 conflict out。

这就是 `possibleUnsafeConflicts` 的作用。

它不是已经确认的 rw-conflict。

它是：

```text
这些 active writable transactions 可能让当前 read-only snapshot unsafe。
```

## 6. `GetSerializableTransactionSnapshot()` 的分流

入口在 `predicate.c`：

```text
GetSerializableTransactionSnapshot(Snapshot snapshot)
```

它先确认：

```text
IsolationIsSerializable()
```

如果 recovery 中，会报错。

hot standby 上不能使用 serializable mode。

然后判断：

```text
if (XactReadOnly && XactDeferrable)
    return GetSafeSnapshot(snapshot);
```

否则进入普通：

```text
GetSerializableTransactionSnapshotInt(snapshot, NULL, InvalidPid)
```

这说明 DEFERRABLE 不是全局改变 SERIALIZABLE。

它只是 read-only serializable snapshot 获取的一条特殊路径。

如果事务不是 read-only，`XactDeferrable` 不产生这个效果。

如果隔离级别不是 SERIALIZABLE，也不进入 SSI safe snapshot 逻辑。

这也是 SQL 层常见误区：

```text
DEFERRABLE 不是让所有锁或约束都延迟。
```

在这里，它只意味着：

```text
read-only serializable transaction 愿意等待 safe snapshot。
```

## 7. `GetSafeSnapshot()` 的循环

`GetSafeSnapshot()` 的主体是一个循环。

每轮先调用：

```text
GetSerializableTransactionSnapshotInt(origSnapshot, NULL, InvalidPid)
```

如果返回后：

```text
MySerializableXact == InvalidSerializableXact
```

说明没有 concurrent read/write serializable transactions 需要跟踪。

snapshot safe，直接返回。

否则持有：

```text
SerializableXactHashLock
```

把当前 sxact 标记：

```text
SXACT_FLAG_DEFERRABLE_WAITING
```

然后等待：

```text
possibleUnsafeConflicts 为空
或当前 sxact 被标记 RO_UNSAFE
```

等待用：

```text
ProcWaitForSignal(WAIT_EVENT_SAFE_SNAPSHOT)
```

当 concurrent writable transaction 结束时，`ReleasePredicateLocks()` 会处理 possible unsafe list。

如果它们证明 read-only safe：

```text
SXACT_FLAG_RO_SAFE
```

如果它们证明 unsafe：

```text
SXACT_FLAG_RO_UNSAFE
```

如果 unsafe，`GetSafeSnapshot()` 会：

```text
ReleasePredicateLocks(false, false)
```

然后重新循环，获取新的 snapshot。

如果 safe，循环结束，并调用：

```text
ReleasePredicateLocks(false, true)
```

这表示：

```text
事务仍继续执行；
但当前 read-only snapshot 已 safe，不再需要 SSI tracking。
```

## 8. `possibleUnsafeConflicts` 的 ownership

`SERIALIZABLEXACT` 有：

```text
possibleUnsafeConflicts
```

在 read-only sxact 创建时，`GetSerializableTransactionSnapshotInt()` 会扫描 active serializable transactions。

对每个 concurrent writable sxact：

```text
SetPossibleUnsafeConflict(read_only_sxact, writable_sxact)
```

这用 `RWConflictData` 结构表示一种候选关系。

注意它不是普通 `outConflicts` / `inConflicts`。

它复用了 conflict pool 元素。

但语义是：

```text
writable sxact 可能让 read-only sxact 的 snapshot unsafe。
```

对于 writable sxact，candidate 挂在它的 possible list。

对于 read-only sxact，candidate 也能被遍历。

当 writable sxact 释放时：

```text
ReleasePredicateLocks()
```

会检查：

```text
它是否 commit。
它是否真的写过。
它是否有 conflict out。
它的 earliestOutConflictCommit 是否早于 read-only snapshot 前的 lastCommitBeforeSnapshot。
```

如果满足危险条件：

```text
FlagSxactUnsafe(roXact)
```

否则释放该 possible conflict。

当 read-only sxact 的 possible list 为空：

```text
SXACT_FLAG_RO_SAFE
```

如果它正在 DEFERRABLE waiting：

```text
ProcSendSignal(roXact->pgprocno)
```

这就是 safe snapshot 等待被唤醒的路径。

## 9. safe 后为什么能释放 predicate locks

`ReleasePredicateLocks(false, true)` 中：

```text
isCommit = false
isReadOnlySafe = true
```

这不是 rollback。

也不是 commit。

它表示：

```text
当前事务仍在运行；
但它已经被证明不会成为 dangerous structure 的 read-only 侧；
后续不需要 SIREAD locks 和 conflict tracking。
```

释放后：

```text
MySerializableXact = InvalidSerializableXact
LocalPredicateLockHash = NULL
```

后续只读扫描不会再创建 SIREAD。

这就是 DEFERRABLE 的性能价值。

对于长报表事务，如果不等 safe snapshot，可能在整个报表期间保留大量 SIREAD locks。

这些 locks 不阻塞写。

但会消耗 shared memory，并可能让 writer 记录更多 conflict。

DEFERRABLE 用启动等待换取：

- 后续 scan 更少 SSI overhead。
- 更低 serialization failure 风险。
- 更少 shared predicate lock footprint。

这不是免费优化。

等待时间可能很长。

如果系统持续有 overlapping writable serializable transactions，safe snapshot 可能反复等待和重试。

## 10. imported snapshot 与 parallel 边界

`SetSerializableTransactionSnapshot()` 处理 imported snapshot。

如果当前事务是：

```text
SERIALIZABLE READ ONLY DEFERRABLE
```

并试图 import snapshot，源码会报错：

```text
a snapshot-importing transaction must not be READ ONLY DEFERRABLE
```

原因是：

```text
imported snapshot 已经由别人给定；
当前事务无法等待一个自己证明 safe 的 snapshot。
```

parallel worker 也有特殊边界。

parallel worker 不创建自己的 `SERIALIZABLEXACT`。

leader 的 sxact 会通过 parallel context 共享。

safe release 在 parallel mode 可能 partial。

源码使用：

```text
SXACT_FLAG_PARTIALLY_RELEASED
SavedSerializableXact
AttachSerializableXact()
DetachSerializableXact()
```

保证 worker 不会在 leader 清理后仍引用无效 sxact。

本节不展开 parallel 细节。

但要记住：

```text
safe snapshot 优化也必须尊重 sxact ownership。
```

它不能只把 backend-local 指针清空。

还要确认没有其他执行参与者需要这个对象。

## 11. 观测与诊断入口

观察等待：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event = 'SafeSnapshot';
```

具体 wait event 名称依版本展示可能不同。

源码等待点是：

```text
WAIT_EVENT_SAFE_SNAPSHOT
```

观察 blocking PIDs：

```sql
SELECT pg_safe_snapshot_blocking_pids(<blocked_pid>);
```

`lockfuncs.c` 中：

```text
pg_safe_snapshot_blocking_pids()
  -> GetSafeSnapshotBlockingPids()
```

它只在目标 pid 正在 `GetSafeSnapshot()` 等待时返回候选 blockers。

它不是普通 lock wait blocker。

观察 SIREAD footprint：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, relation, page, tuple;
```

对比：

- 普通 `SERIALIZABLE READ ONLY`。
- `SERIALIZABLE READ ONLY DEFERRABLE`。

如果 DEFERRABLE 已拿到 safe snapshot，后续大查询不应持续制造同样规模的 SIREAD footprint。

源码断点：

```text
GetSerializableTransactionSnapshot
GetSafeSnapshot
GetSerializableTransactionSnapshotInt
SetPossibleUnsafeConflict
FlagSxactUnsafe
ReleasePredicateLocks
GetSafeSnapshotBlockingPids
```

调试字段：

```text
MySerializableXact->flags
possibleUnsafeConflicts
PredXact->WritableSxactCount
SeqNo.lastCommitBeforeSnapshot
SeqNo.earliestOutConflictCommit
```

## 12. 成本、资源与适用边界

DEFERRABLE 的成本是启动等待。

等待取决于：

- concurrent writable serializable transactions 数量。
- 这些事务是否长时间打开。
- 这些事务是否有 conflict out。
- 系统是否持续产生新的 overlapping writable transactions。

收益是后续降低：

- SIREAD lock 创建成本。
- predicate lock shared memory 占用。
- conflict checking 成本。
- read-only transaction 失败风险。

适合：

```text
长只读事务
一致性报表
逻辑导出
审计扫描
可以接受启动等待的后台任务
```

不适合：

```text
短请求
读写事务
需要低 p99 启动延迟的 API
非 SERIALIZABLE workload
```

它也不是替代索引优化。

如果一个只读报表本身很慢，DEFERRABLE 只减少 SSI overhead 和失败风险。

它不让查询计划更快。

它也不会减少 MVCC visibility 或 I/O 成本。

## 13. 常见误区

误区一：

```text
只读事务不会导致 serialization failure。
```

不对。

普通 read-only serializable 仍可能参与 dangerous structure。

误区二：

```text
DEFERRABLE 会让读写事务延迟冲突检查。
```

不对。

这里的 DEFERRABLE 只对 SERIALIZABLE READ ONLY safe snapshot 有意义。

误区三：

```text
safe snapshot 是更新更少的 snapshot。
```

不对。

safe 指的是不会参与 SSI anomaly。

不是数据更新少。

误区四：

```text
等待 safe snapshot 等同于等待普通锁。
```

不对。

它等待 concurrent writable serializable transactions 给出 safe / unsafe 结论。

误区五：

```text
DEFERRABLE 一定更快。
```

不对。

它可能启动等待很久。

它适合长只读任务。

误区六：

```text
拿到 safe snapshot 后还能写。
```

不对。

事务属性是 READ ONLY。

写入不在这个优化语义内。

## 14. 课堂实验

实验一：观察 DEFERRABLE 等待。

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE safe_demo SET balance = balance + 1 WHERE id = 1;
-- 保持事务打开。
```

Session B：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT sum(balance) FROM safe_demo;
```

Session C：

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE query LIKE '%safe_demo%';
```

再执行：

```sql
SELECT pg_safe_snapshot_blocking_pids(<session_b_pid>);
```

提交或回滚 A，观察 B 是否继续。

实验二：比较 SIREAD footprint。

Session B 分别运行：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY;
SELECT count(*) FROM safe_demo;
```

和：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
SELECT count(*) FROM safe_demo;
```

在查询后检查：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, relation, page, tuple;
```

比较 footprint。

实验三：制造 unsafe 重试。

用两个 writable serializable transactions 制造 conflict out。

同时启动 DEFERRABLE read-only。

在 gdb 中观察：

```text
FlagSxactUnsafe
ReleasePredicateLocks(false, false)
GetSafeSnapshot retry
```

这个实验较难稳定手工复现，适合 isolation test。

实验四：import snapshot 边界。

尝试：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE READ ONLY DEFERRABLE;
-- 使用导入 snapshot 的流程。
```

观察错误。

把它映射到：

```text
SetSerializableTransactionSnapshot()
```

实验五：应用策略。

把一个长只读报表分别用：

- `SERIALIZABLE READ ONLY`。
- `SERIALIZABLE READ ONLY DEFERRABLE`。

记录：

- 启动等待。
- 总耗时。
- SIReadLock 数量。
- 是否出现 `40001`。

根据结果判断是否适合启用 DEFERRABLE。

## 15. 讨论题

1. read-only transaction 为什么仍可能是 dangerous structure 的一部分？

2. safe snapshot 到底 safe 在哪里？

3. DEFERRABLE 为什么只适合 SERIALIZABLE READ ONLY？

4. `possibleUnsafeConflicts` 与普通 `rw-conflict` 有什么语义差异？

5. 为什么 unsafe 后要释放当前 SSI 状态并重新获取 snapshot？

6. 为什么 safe 后可以 `ReleasePredicateLocks(false, true)`？

7. `pg_safe_snapshot_blocking_pids()` 和普通 lock blocker 查询有什么差异？

8. imported snapshot 为什么不能用于 READ ONLY DEFERRABLE？

9. DEFERRABLE 对长报表和短 OLTP 点查的收益为什么不同？

10. 如果 DEFERRABLE 等待时间很长，应该优先缩短哪些 writable transactions？

## 16. 本节小结

本节的核心结论是：

```text
DEFERRABLE 用启动等待换取 safe snapshot；
safe snapshot 让 read-only serializable transaction 后续不需要 SSI predicate lock tracking。
```

只读不等于天然 safe。

read-only transaction 仍可能作为 `Tin` 参与 dangerous structure。

PostgreSQL 通过：

- `possibleUnsafeConflicts`。
- `SXACT_FLAG_DEFERRABLE_WAITING`。
- `SXACT_FLAG_RO_SAFE`。
- `SXACT_FLAG_RO_UNSAFE`。
- `GetSafeSnapshot()` 循环。
- `ReleasePredicateLocks(false, true)`。

实现这个优化。

它不改变 MVCC snapshot 的基本可见性。

它改变的是 SSI tracking 成本和失败风险。

到这里，03 目录中从 snapshot、visibility、cleanup 到 SSI 的主线形成闭环：

```text
单事务可见性
  -> 多事务依赖
  -> predicate footprint
  -> SIREAD 生命周期
  -> index 粒度
  -> safe snapshot 优化
```
