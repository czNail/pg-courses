# PostgreSQL SIREAD lock 生命周期与内存压力

## 课程定位

前置知识：已经理解 SIREAD lock 是 predicate lock footprint，不阻塞写；也理解 `rw-conflict` 通过 reader -> writer 边进入 `SERIALIZABLEXACT` 的冲突列表。

本节唯一主问题：

```text
SIREAD lock 既然不阻塞写，为什么不能在语句结束或事务提交时立即释放？
```

核心矛盾：

```text
SIREAD 只是检测标记，越早释放越省内存；
但已提交 reader 仍可能和仍在运行的 writer / reader 形成 dangerous structure，过早释放会漏掉 rw-conflict。
```

学完后应能独立判断：

- SIREAD 与 heavyweight lock 的生命周期差异。
- 为什么 committed serializable transaction 可能继续保留 predicate locks。
- `SxactGlobalXmin` 为什么决定旧 SSI 状态何时可清理。
- read-only safe transaction 为什么能提前退出 SSI 跟踪。
- rollback 为什么比 commit 更容易清理。
- predicate lock promotion 与 old committed summary 分别解决哪类内存压力。
- `pg_locks` 中 SIReadLock 为什么不能按普通锁语义解释。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 35 节讲的是 footprint 怎么表达。

本节讲 footprint 活多久。

普通锁的直觉是：

```text
事务结束
  -> 锁释放
```

SIREAD 不能简单套这个模型。

它不保护物理资源互斥。

它保护的是：

```text
未来某个写入是否应该与过去某个读取形成 rw-conflict。
```

过去的读取事务即使已经提交，也可能仍是 dangerous structure 的一端。

只要还有 overlapping serializable transaction 没结束，系统可能还需要这段读 footprint。

所以 SIREAD 生命周期不是：

```text
statement lifetime
transaction lifetime
```

而是：

```text
SSI overlap lifetime
```

这也是为什么 predicate lock manager 不能直接复用普通 lock manager。

普通 lock manager 的 session / transaction ownership 不适合“提交后仍保留”的 SIREAD。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
SERIALIZABLE 事务获取 snapshot 时创建 SERIALIZABLEXACT；
读路径创建 SIREAD locks 并挂到 sxact；
提交时 ReleasePredicateLocks() 标记 sxact committed，但可能把它放入 finished list 继续保留 locks；
当 SxactGlobalXmin 前进且不再有 overlapping serializable transaction 需要它时，ClearOldPredicateLocks() / ReleaseOneSerializableXact() 才真正释放；
read-only safe 或 rollback 可以更早清理。
```

本节的核心矛盾是：

```text
内存希望及时回收；
correctness 要求读 footprint 活到所有可能依赖它的事务结束。
```

PostgreSQL 的答案分四层：

- rollback 事务尽快释放。
- read-only safe 事务提前释放。
- committed writable 事务保留到 overlap 边界。
- 老 committed 事务在必要时总结到 SLRU，限制 RAM 增长。

这不是单纯的垃圾回收。

这是 correctness-aware cleanup。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/predicate_internals.h` | `SERIALIZABLEXACT` 的 `finishedBefore`、`xmin`、flags、`predicateLocks`。 |
| 2 | `src/backend/storage/lmgr/predicate.c` | `ReleasePredicateLocks()`、`ReleaseOneSerializableXact()`、`ClearOldPredicateLocks()`、`SetNewSxactGlobalXmin()`。 |
| 3 | `src/backend/storage/lmgr/predicate.c` | `SerialAdd()`、`SerialGetMinConflictCommitSeqNo()`、old committed summary。 |
| 4 | `src/include/storage/predicate.h` | release / conflict API 的外部边界。 |
| 5 | `src/backend/access/transam/xact.c` | transaction end 如何进入 predicate lock release。 |
| 6 | `src/backend/utils/adt/lockfuncs.c` | `pg_locks` 如何从 predicate lock manager 复制 SIReadLock。 |
| 7 | `src/backend/storage/lmgr/README-SSI` | 为什么 SIREAD 需要跨 commit 保留，以及 PostgreSQL 为什么不用普通锁结构。 |

推荐阅读顺序：

```text
先读 SERIALIZABLEXACT 字段
  -> 读 ReleasePredicateLocks()
  -> 读 SetNewSxactGlobalXmin()
  -> 读 ClearOldPredicateLocks()
  -> 读 ReleaseOneSerializableXact()
  -> 最后读 SerialAdd() 的 summary fallback
```

本节不要把 cleanup 写成后台整理。

它是 SSI correctness 的一部分。

## 4. 从 runtime 现象进入

准备表：

```sql
DROP TABLE IF EXISTS siread_life;

CREATE TABLE siread_life
(
    id int PRIMARY KEY,
    k int NOT NULL,
    payload text
);

INSERT INTO siread_life
SELECT g, g % 10, repeat('x', 50)
FROM generate_series(1, 1000) AS g;
```

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT count(*)
FROM siread_life
WHERE k = 3;

UPDATE siread_life
SET payload = payload
WHERE id = 3;
```

让 A 成为 writable serializable transaction。

Session B 在 A 提交前启动：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT count(*)
FROM siread_life
WHERE k = 4;
```

现在 A 提交：

```sql
COMMIT;
```

此时 A 的 backend 已经结束事务。

但 A 的 SSI 状态不一定马上完全消失。

原因是 B 与 A overlap。

如果后续冲突判断需要知道 A 曾经读过或写过什么，系统仍可能需要 A 的 `SERIALIZABLEXACT`、conflict summary 或 SIREAD footprint。

用 `pg_locks` 观察：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, relation, page, tuple;
```

不要把结果机械理解为“哪个 pid 正在持有普通锁”。

SIREAD lock 是 predicate lock manager 的状态快照。

它可能对应 active transaction。

也可能对应已经 finished 但尚未 cleanup 的 serializable transaction。

这就是本节 runtime 入口：

```text
事务结束不等于 SSI footprint 立即结束。
```

## 5. SIREAD 为什么不阻塞写

SIREAD 的目标不是互斥。

它不让 writer 等待 reader。

writer 看到 SIREAD 时做的是：

```text
CheckForSerializableConflictIn()
  -> CheckTargetForConflictsIn()
  -> FlagRWConflict(reader, writer)
```

如果 dangerous structure 已经形成，writer 可能被错误中止。

如果没有，writer 继续执行。

所以 SIREAD 的运行语义是：

```text
记录依赖；
必要时失败；
不阻塞资源访问。
```

这与 heavyweight lock 完全不同。

heavyweight lock 的核心状态是：

```text
granted / waiting
```

SIREAD 的核心状态是：

```text
reader footprint / conflict detection
```

这也是为什么：

```sql
SELECT *
FROM pg_locks
WHERE mode = 'SIReadLock';
```

看到 `granted = true` 不表示有人被挡住。

它只是说明 footprint 存在。

writer 可以继续写。

但写入可能让某个事务未来或立即报 `40001`。

## 6. 为什么 commit 后还要保留

考虑三事务结构：

```text
Tin -> Tpivot -> Tout
```

如果 `Tout` 已经提交，系统仍然需要判断 `Tin` 和 `Tpivot` 后续是否会让结构变成真实异常。

因此 committed transaction 不能简单删除所有信息。

`ReleasePredicateLocks(isCommit=true, isReadOnlySafe=false)` 中，提交路径会：

```text
SXACT_FLAG_COMMITTED
commitSeqNo = ++PredXact->LastSxactCommitSeqNo
```

如果事务 commit 且仍持有 predicate locks，它可能被加入：

```text
FinishedSerializableTransactions
```

随后等待 cleanup。

字段 `finishedBefore` 记录：

```text
事务结束时 nextXid 的近似边界。
```

它用于判断：

```text
当所有仍活跃 serializable transaction 的 xmin 都超过这个边界后；
这个 finished sxact 不再能和它们 overlap；
可以释放。
```

这就是：

```text
finishedBefore 与 SxactGlobalXmin 比较
```

的意义。

不是“提交后多久释放”。

而是“是否还有可能重叠的 serializable transaction”。

## 7. rollback 为什么更容易清理

rollback 的语义不同。

失败事务不会进入成功提交结果。

它的读写依赖不需要继续保护一个已提交 serial order。

因此 `ReleasePredicateLocks(isCommit=false, ...)` 会标记：

```text
SXACT_FLAG_DOOMED
SXACT_FLAG_ROLLED_BACK
```

并更积极地释放：

```text
predicate locks
rw-conflicts
SERIALIZABLEXACT
local predicate lock hash
```

这也是 SSI 失败的一个重要成本边界。

失败事务需要释放 footprint。

其他事务看到 doomed transaction 时通常可以忽略它。

例如 conflict 检查中会跳过：

```text
SxactIsDoomed(sxact)
```

因为它不会提交成图中的有效节点。

这解释了为什么一旦发现某个 transaction 必须失败，PostgreSQL 会设置 `DOOMED`。

后续路径再次碰到它，可以快速报错或忽略。

## 8. read-only safe 为什么可以提前释放

read-only transaction 没有写入。

它仍可能成为 dangerous structure 的 `Tin`。

但 PostgreSQL 有一个优化：

```text
如果能证明 read-only snapshot 是 safe；
这个事务不会再成为 anomaly 的一部分；
它可以释放 predicate locks 并退出 SSI tracking。
```

入口仍在 `ReleasePredicateLocks()`。

当参数：

```text
isReadOnlySafe = true
```

时，函数不是普通事务结束释放。

它表示：

```text
事务仍在运行；
但已经不需要后续 SSI conflict tracking。
```

非 parallel 模式下，可以完全释放。

parallel 模式下要考虑 leader / worker 共享同一个 `SERIALIZABLEXACT`。

源码用：

```text
SXACT_FLAG_PARTIALLY_RELEASED
SavedSerializableXact
```

处理 partial release。

这说明生命周期并不只跟 SQL transaction 边界绑定。

它还跟：

- read-only safe 证明。
- parallel workers 是否仍引用 sxact。
- 当前 backend-local `MySerializableXact` 是否已清空。

有关。

第 38 节会专门展开 safe snapshot。

本节只强调：

```text
read-only safe 是释放 SIREAD 的 correctness shortcut。
```

## 9. `SxactGlobalXmin` 是 cleanup 边界

`PredXactListData` 中有：

```text
SxactGlobalXmin
SxactGlobalXminCount
```

它们表示：

```text
当前 active serializable transactions 中最老 snapshot xmin；
有多少 active sxact 使用这个 xmin。
```

创建 serializable snapshot 时：

```text
GetSerializableTransactionSnapshotInt()
  -> sxact->xmin = snapshot->xmin
  -> 维护 PredXact->SxactGlobalXmin / count
```

释放时：

```text
ReleasePredicateLocks()
  -> 如果当前 sxact->xmin 等于 SxactGlobalXmin
     -> SxactGlobalXminCount--
     -> count 到 0 时 SetNewSxactGlobalXmin()
```

当 global xmin 前进，可能触发：

```text
ClearOldPredicateLocks()
```

清理条件的直觉是：

```text
如果某个 finished sxact 结束时的边界已经早于所有 active serializable snapshot；
它不再和任何 active sxact overlap；
它的 SIREAD 和 conflict 信息可以释放或总结。
```

这与 MVCC cleanup horizon 类似。

但它不是同一个对象。

MVCC cleanup horizon 保护 tuple version。

`SxactGlobalXmin` 保护 SSI conflict 判断所需的 transaction footprint。

两者都围绕“还有谁可能观察旧世界”。

但服务的状态不同。

## 10. 内存压力：promotion 与 summary 是两件事

SIREAD 内存压力有两类。

第一类是 lock target 太多。

例如一个事务读了大量 tuple。

解决方式是：

```text
promotion
```

多个 tuple target 可以提升到 page。

多个 page target 可以提升到 relation。

相关入口：

```text
CheckAndPromotePredicateLockRequest()
MaxPredicateChildLocks()
DeleteChildTargetLocks()
```

promotion 降低 lock 数。

代价是 footprint 变粗，false positive 增加。

第二类是 committed serializable transaction 太多。

即使 predicate locks 已经尽量合并，仍可能有大量 finished sxact 和 conflict summary 需要保留。

解决方式包括：

```text
ClearOldPredicateLocks()
SummarizeOldestCommittedSxact()
SerialAdd()
SerialGetMinConflictCommitSeqNo()
```

old committed summary 会把部分信息写入 serial SLRU。

之后再遇到 old committed xid，只能做更保守判断。

这可能增加失败。

但能避免 shared memory 无界增长。

所以：

```text
promotion 处理 lock granularity；
summary 处理 old committed transaction retention。
```

不要把它们混为一谈。

## 11. ERROR / fallback 边界

常见异常之一：

```text
out of shared memory
```

可能来自创建 predicate lock target 或 lock 时。

hint 可能建议增加：

```text
max_pred_locks_per_transaction
```

另一个异常是 conflict pool 不足：

```text
not enough elements in RWConflictPool to record a read/write conflict
```

这说明冲突边本身耗尽。

第三类不是 ERROR，而是行为退化：

```text
fine-grained locks promoted to relation-level
old committed sxact summarized
```

它们不破坏正确性。

但会提高 false positive。

第四类是 parallel query partial release。

read-only safe 事务在 parallel 模式下不能简单释放 `SERIALIZABLEXACT`。

因为 worker 可能仍持有引用。

源码通过 partial flag 和 leader final cleanup 处理。

第五类是 two-phase commit。

prepared transaction 可能长期停留。

它的 `SERIALIZABLEXACT` 和 predicate locks 也需要跨普通 backend 生命周期保存。

这让 cleanup 更保守。

本节只需要建立边界：

```text
SSI cleanup 只在证明不会漏掉 dangerous structure 后发生。
```

## 12. 观测与诊断入口

观察 SIReadLock：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, relation, page, tuple;
```

观察长事务：

```sql
SELECT pid, state, xact_start, backend_xmin, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

观察参数：

```sql
SHOW max_pred_locks_per_transaction;
SHOW max_pred_locks_per_relation;
SHOW max_pred_locks_per_page;
SHOW serializable_buffers;
```

观察失败：

```text
SQLSTATE 40001
out of shared memory
not enough elements in RWConflictPool
```

源码断点：

```text
ReleasePredicateLocks
ReleasePredicateLocksLocal
ReleaseOneSerializableXact
ClearOldPredicateLocks
SetNewSxactGlobalXmin
SummarizeOldestCommittedSxact
SerialAdd
```

调试时看：

```text
MySerializableXact->flags
MySerializableXact->xmin
MySerializableXact->finishedBefore
PredXact->SxactGlobalXmin
PredXact->SxactGlobalXminCount
PredXact->WritableSxactCount
```

如果 serialization failure 增多，先判断：

- 是否有长 serializable transaction。
- 是否有大量 seq scan。
- 是否有 relation-level promotion。
- 是否有 prepared transaction。
- 是否应用未重试导致并发堆积。

不要只盯 `pg_locks` 中的数量。

数量只是表象。

生命周期和粒度才是原因。

## 13. 常见误区

误区一：

```text
SIREAD 不阻塞，所以可以语句结束释放。
```

不对。

它服务未来 conflict 检测。

语句结束并不代表 overlapping transaction 已结束。

误区二：

```text
COMMIT 后所有 predicate lock 都应消失。
```

不对。

committed serializable transaction 可能继续影响 active transaction 的 dangerous structure 判断。

误区三：

```text
看到 SIReadLock 就说明有人被锁住。
```

不对。

SIREAD 是检测标记。

误区四：

```text
promotion 和 summary 是同一个机制。
```

不对。

promotion 合并 target 粒度。

summary 压缩 old committed transaction 信息。

误区五：

```text
read-only 事务一定不需要 SIREAD。
```

不对。

只有 safe read-only 才能退出。

普通 read-only serializable 仍可能参与 dangerous structure。

误区六：

```text
调大参数总能解决 SERIALIZABLE 失败。
```

不对。

参数只能减少部分资源型退化。

真实冲突仍必须失败。

## 14. 课堂实验

实验一：观察 SIREAD 不阻塞写。

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM siread_life WHERE k = 3;
```

Session B：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
INSERT INTO siread_life VALUES (20001, 3, 'new');
COMMIT;
```

观察 B 是否等待 A。

再提交 A，观察是否出现 serialization failure。

实验二：观察 footprint 粒度。

分别执行：

```sql
SET enable_seqscan = on;
SET enable_indexscan = off;
```

和：

```sql
CREATE INDEX siread_life_k_idx ON siread_life(k);
SET enable_seqscan = off;
```

比较 `pg_locks` 中 SIReadLock 粒度。

实验三：观察长事务影响。

保持一个 SERIALIZABLE 事务打开。

另起多个会话执行短 SERIALIZABLE 事务读写同一表。

观察：

- `pg_locks` 中 SIReadLock 数量。
- `pg_stat_activity` 中最老 `xact_start`。
- serialization failure 率。

实验四：源码断点。

设置：

```text
break ReleasePredicateLocks
break ClearOldPredicateLocks
break SetNewSxactGlobalXmin
```

提交不同类型事务：

- rollback。
- read-only commit。
- read-write commit。
- read-only deferrable。

比较 flags 和 cleanup 路径。

实验五：promotion 压力。

在测试实例调低 predicate lock 阈值。

执行大范围 SERIALIZABLE 查询。

观察 tuple/page SIReadLock 是否提升到 relation SIReadLock。

记录失败率变化。

## 15. 讨论题

1. 为什么 SIREAD lock 不阻塞写，仍然能保护 serializable correctness？

2. committed reader 为什么仍可能需要保留 footprint？

3. rollback 事务为什么可以更早释放 SSI 状态？

4. `SxactGlobalXmin` 和 MVCC cleanup horizon 有什么相似点和差异？

5. promotion 为什么是内存保护，而不是正确性证明？

6. old committed summary 为什么可能增加 false positive？

7. parallel read-only safe release 为什么需要 partial release？

8. prepared transaction 会怎样拖慢 SSI cleanup？

9. `pg_locks` 中 SIReadLock 的 pid 字段为什么不能按普通锁直觉解释？

10. 一个生产系统 SERIALIZABLE 失败率升高时，如何区分真实业务冲突与 footprint 变粗？

## 16. 本节小结

本节的核心结论是：

```text
SIREAD lock 的生命周期由 SSI overlap correctness 决定，不由语句或普通事务锁语义决定。
```

SIREAD 不阻塞写。

它记录读 footprint。

writer 看到它时记录 rw-conflict。

如果 dangerous structure 可能成立，事务失败。

因为 committed transaction 仍可能是冲突结构的一端，SIREAD 和 `SERIALIZABLEXACT` 可能在 commit 后继续保留。

cleanup 依赖：

- `SxactGlobalXmin`。
- `finishedBefore`。
- finished list。
- read-only safe。
- rollback。
- old committed summary。

内存压力通过两类机制处理：

- promotion 合并 lock target。
- summary 压缩 old committed transaction。

下一节继续讲：

```text
索引访问方法如何决定 predicate lock 粒度；
为什么 btree、heap scan、unsupported AM 会产生不同 false positive 和 abort 行为。
```
