# PostgreSQL rw-conflict 与 dangerous structure

## 课程定位

前置知识：已经理解 snapshot isolation 可以出现 write skew，也知道 SERIALIZABLE 事务会创建 `SERIALIZABLEXACT`，读路径留下 SIREAD footprint，写路径可能触发 SSI conflict 检查。

本节唯一主问题：

```text
PostgreSQL 为什么只追踪并发事务之间的 rw-conflict，并用 dangerous structure 判断是否必须中止事务？
```

核心矛盾：

```text
完整依赖图能更准确地证明是否存在环；
但在 OLTP runtime 中维护完整事务图成本太高，且很多依赖天然随提交顺序成立。
```

学完后应能独立判断：

- `rw-conflict out` 和 `rw-conflict in` 的方向。
- 为什么 `wr` / `ww` 依赖通常不需要 SSI 额外追踪。
- dangerous structure 的 `Tin -> Tpivot -> Tout` 为什么足以成为报警形状。
- 为什么 `Tout` 的提交顺序影响是否必须失败。
- 为什么有些事务在写入阶段失败，有些在提交阶段失败。
- 为什么 PostgreSQL 可能标记 pivot 为 `DOOMED`，而不是总是中止当前事务。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节从 write skew 进入 SSI。

它说明了：

```text
稳定 snapshot 不等于可串行化结果。
```

本节继续向内走一步。

问题从：

```text
为什么需要 SSI？
```

变成：

```text
SSI 具体追踪什么状态，为什么这些状态足以在 runtime 里阻止异常？
```

PostgreSQL 没有维护一个完整的理论依赖图。

它没有把每个 tuple 的每次 read / write 都转成持久边。

它没有在 commit 时做一次全图环检测。

它只追踪关键的并发读写依赖：

```text
rw-conflict
```

在源码里，这些边挂在：

```text
SERIALIZABLEXACT->outConflicts
SERIALIZABLEXACT->inConflicts
```

每条边由 `RWConflictData` 表示。

边的方向是：

```text
reader -> writer
```

含义是：

```text
reader 的 snapshot 没有包含 writer 的写入；
所以在任何串行解释中，reader 必须排在 writer 前面。
```

本节聚焦这条边如何产生、如何保存、如何触发失败。

第 35 节再讲读 footprint 如何落到 relation/page/tuple target。

第 36 节再讲这些状态为什么要跨事务结束保留。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
当读者没有看到并发 writer 的效果时，PostgreSQL 记录 reader -> writer 的 rw-conflict；
每次新增 conflict 都调用 FlagRWConflict() 检查是否形成 dangerous structure；
提交时 PreCommit_CheckForSerializationFailure() 再检查未完全暴露的结构；
一旦发现可能无法串行化，就报 40001 或标记某个 sxact 为 DOOMED。
```

本节的核心矛盾是：

```text
系统必须足够早地阻止不可串行化提交；
又不能为每个事务维护昂贵的完整依赖图和全局环检测。
```

SSI 的取舍是：

```text
只追踪并发 rw-conflict；
利用 snapshot isolation 异常一定包含两个相邻 rw-conflict 的性质；
在冲突记录和提交边界上做保守失败。
```

这带来一个 runtime 特征：

```text
失败点不固定。
```

同一类业务异常，可能在：

- 读 tuple 时失败。
- 写 tuple 时失败。
- index insert 时失败。
- `COMMIT` 时失败。

取决于哪条 conflict 先被发现，以及哪个事务先进入 prepared / committed 状态。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/README-SSI` | dangerous structure 的设计理由和 `rw-conflict` 方向。 |
| 2 | `src/include/storage/predicate_internals.h` | `SERIALIZABLEXACT`、`RWConflictData`、`SerCommitSeqNo`、flags。 |
| 3 | `src/backend/storage/lmgr/predicate.c` | `SetRWConflict()`、`FlagRWConflict()`、`OnConflict_CheckForSerializationFailure()`、`PreCommit_CheckForSerializationFailure()`。 |
| 4 | `src/backend/access/heap/heapam.c` | `HeapCheckForSerializableConflictOut()` 如何从 MVCC tuple 状态发现 conflict out。 |
| 5 | `src/include/storage/predicate.h` | conflict in/out 对 table/index AM 暴露的入口。 |
| 6 | `src/backend/access/nbtree/nbtinsert.c` | index insert 如何形成 page 级 conflict in。 |
| 7 | `src/backend/utils/adt/lockfuncs.c` | 用 `pg_locks` 看 SIREAD，但不能直接看到完整 conflict graph。 |

推荐阅读顺序：

```text
README-SSI 的 Tin/Tpivot/Tout
  -> predicate_internals.h 的 SERIALIZABLEXACT
  -> predicate.c 的 RWConflictExists/SetRWConflict
  -> predicate.c 的 FlagRWConflict
  -> predicate.c 的 OnConflict_CheckForSerializationFailure
  -> predicate.c 的 PreCommit_CheckForSerializationFailure
```

不要把本节写成函数清单。

主线只有一条：

```text
一条 rw-conflict 边进入图
  -> 图上是否已经存在另一条相关边
  -> 是否可能出现 dangerous structure
  -> 谁必须失败
```

## 4. 从 runtime 现象进入

继续使用值班表。

这次直接使用 SERIALIZABLE：

```sql
DROP TABLE IF EXISTS on_call;

CREATE TABLE on_call
(
    doctor text PRIMARY KEY,
    dept text NOT NULL,
    on_call boolean NOT NULL
);

INSERT INTO on_call VALUES
('alice', 'icu', true),
('bob',   'icu', true);
```

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE dept = 'icu' AND on_call;
UPDATE on_call SET on_call = false WHERE doctor = 'alice';
```

Session B：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE dept = 'icu' AND on_call;
UPDATE on_call SET on_call = false WHERE doctor = 'bob';
```

然后交错提交。

你会看到其中一个事务失败。

把这个现象翻译成 dependency：

```text
A 的 snapshot 没看到 B 之后要做的写入。
B 写入 bob 后，A -> B 成为候选顺序。

B 的 snapshot 没看到 A 之后要做的写入。
A 写入 alice 后，B -> A 成为候选顺序。
```

如果两条边都被接受并提交，图中有环。

环表示：

```text
A 必须排在 B 前；
B 又必须排在 A 前。
```

没有合法串行顺序。

PostgreSQL 不一定真的等到完整环出现才行动。

它看的是 dangerous structure。

对于 write skew 两事务场景，结构很紧凑。

对于三事务场景，更典型：

```text
Tin -> Tpivot -> Tout
```

如果 `Tout` 已经先提交，继续让另外两边都提交就危险。

这就是后面源码判断的核心。

## 5. `rw-conflict` 的方向

`RWConflictData` 定义在 `predicate_internals.h`：

```text
sxactOut
sxactIn
```

命名容易让人读慢。

把它映射成课程模型：

```text
sxactOut = reader
sxactIn  = writer
```

一条边表示：

```text
reader -> writer
```

为什么是这个方向？

因为 reader 的 snapshot 没看到 writer 的写入。

在任何串行解释里，reader 都必须排在 writer 之前。

所以 reader 对 writer 有 outgoing conflict。

writer 接收 incoming conflict。

源码中：

```text
SetRWConflict(reader, writer)
```

会把同一个 `RWConflictData` 挂入：

```text
reader->outConflicts
writer->inConflicts
```

这个双向挂载很重要。

当一个新 conflict 进入时，系统可能需要看：

- writer 是否已经有 conflict out。
- reader 是否已经有 conflict in。
- 某个 related transaction 是否已经 committed / prepared。
- read-only 优化是否允许忽略。

如果只保存单向列表，提交检查要么很贵，要么需要全局扫描。

PostgreSQL 选择在事务对象上保存两端列表。

这是 SSI 的状态局部性：

```text
冲突图不是独立大图；
它嵌在 SERIALIZABLEXACT 对象的 in/out 链表里。
```

## 6. conflict out：读路径从 MVCC 看到 writer

读路径的关键入口在 `heapam.c`：

```text
HeapCheckForSerializableConflictOut()
```

它解决的问题是：

```text
我正在读一个 tuple；
这个 tuple 的 MVCC 状态是否证明有并发 writer 的效果没有被我看到？
```

调用者必须持有 buffer shared lock。

因为函数可能为了 visibility 判断设置 hint bit。

它先用：

```text
HeapTupleSatisfiesVacuum(tuple, TransactionXmin, buffer)
```

粗分 tuple 状态。

然后根据 `visible` 参数选择 XID：

```text
不可见的 live / inserting tuple
  -> 看 xmin。

可见但被并发删除或更新
  -> 看 update xid / delete xid。
```

之后：

```text
SubTransGetTopmostTransaction(xid)
CheckForSerializableConflictOut(relation, xid, snapshot)
```

`CheckForSerializableConflictOut()` 会：

- 确认当前 relation / snapshot 需要 serialization read 检查。
- 忽略自己的 top XID。
- 在 `SerializableXidHash` 中找 writer 的 `SERIALIZABLEXACT`。
- 确认两个事务并发。
- 去重已有 conflict。
- 调用 `FlagRWConflict(MySerializableXact, sxact)`。

这里的 `MySerializableXact` 是 reader。

`sxact` 是 writer。

所以这条边是：

```text
reader -> writer
```

如果 writer 已经被总结到 serial SLRU，源码会通过 `SerialGetMinConflictCommitSeqNo()` 走 summary fallback。

这时精度下降。

但正确性仍然保守。

## 7. conflict in：写路径从 SIREAD 看到 reader

写路径的关键入口在 `predicate.c`：

```text
CheckForSerializableConflictIn(relation, tid, blkno)
```

它解决的问题是：

```text
我正在写某个目标；
有没有别的 serializable reader 之前用 SIREAD 标记过这个目标或覆盖它的粗粒度目标？
```

写入前先检查：

```text
SerializationNeededForWrite(relation)
```

如果当前没有 relevant serializable 事务，或 relation 不参与 SSI，就快速返回。

接着检查当前事务是否已经被标记为 doomed。

如果是，直接报 serialization failure。

然后设置：

```text
MyXactDidWrite = true
```

这对 read-only 优化很重要。

随后按顺序检查：

```text
tuple
page
relation
```

每个 target 进入：

```text
CheckTargetForConflictsIn()
```

它扫描目标上的 predicate locks。

如果发现别的事务的 SIREAD：

```text
FlagRWConflict(reader_sxact, MySerializableXact)
```

这里的 reader 是持有 SIREAD 的事务。

当前事务是 writer。

所以仍然是：

```text
reader -> writer
```

这说明 conflict in/out 不是两种边。

它们是从不同 runtime 入口发现同一种边。

读路径发现：

```text
我读到了一个被并发 writer 影响的 tuple 状态。
```

写路径发现：

```text
我写到了别人已经读过的 predicate target。
```

两者最终都汇入 `FlagRWConflict()`。

## 8. 为什么不追踪所有依赖

README-SSI 解释了三类依赖：

- `wr-dependency`：读到了别人写出的结果。
- `ww-dependency`：两个写入之间的版本顺序。
- `rw-conflict`：读者没有看到并发 writer 的结果。

`wr` 和 `ww` 依赖通常与 commit order 一致。

例如你读到了某个已提交版本，说明 writer 已经先完成。

例如两个事务更新同一行，row lock / update conflict 会强制处理顺序。

这些依赖通常不需要 SSI 额外维护。

真正危险的是并发事务之间的 `rw-conflict`。

它表达的是：

```text
读者基于一个不包含 writer 的旧世界做了决策。
```

snapshot isolation 的异常循环一定包含相邻的 `rw-conflict` 边。

所以 PostgreSQL 可以不维护完整图。

它只需要盯住：

```text
Tin -> Tpivot -> Tout
```

两条相邻 rw-conflict。

如果 `Tout` 已经先提交，危险结构可能成为真实异常。

这就是 SSI 的核心降维。

完整图检测更精确。

但它需要更多状态、更多同步和更复杂 cleanup。

PostgreSQL 选择：

```text
少量 false positive
  换取低得多的 runtime 开销。
```

这也是为什么 SERIALIZABLE 可能回滚一个最终不一定真的形成环的事务。

它保守。

但保守方向是正确性。

## 9. `FlagRWConflict()` 是汇合点

`FlagRWConflict(reader, writer)` 是本节最重要的源码入口之一。

它做两件事：

```text
OnConflict_CheckForSerializationFailure(reader, writer)
SetRWConflict(reader, writer)
```

顺序不能反过来理解。

先检查是否新增这条边会触发失败。

再真正写入边。

如果 reader 或 writer 是 `OldCommittedSxact`，不是普通链表边，而是设置 summary flag：

```text
SXACT_FLAG_SUMMARY_CONFLICT_IN
SXACT_FLAG_SUMMARY_CONFLICT_OUT
```

普通情况才进入：

```text
SetRWConflict(reader, writer)
```

`SetRWConflict()` 从 `RWConflictPool` 取一个元素。

如果 pool 空了，会报：

```text
not enough elements in RWConflictPool to record a read/write conflict
```

这说明 rw-conflict 不是无限记录。

它是 postmaster 启动时按配置和最大并发规模预留的 shared memory 资源。

在高冲突 workload 中，冲突记录本身就是资源压力。

但这不是调优第一入口。

第一入口仍然是缩短事务、减少范围读、改善索引计划、降低不必要的 SERIALIZABLE 并发交错。

## 10. dangerous structure 的源码判断

`OnConflict_CheckForSerializationFailure(reader, writer)` 的注释直接给出结构：

```text
Tin ------> Tpivot ------> Tout
      rw             rw
```

函数检查的不是一个抽象图对象。

它围绕当前新增边：

```text
reader -> writer
```

判断新增后是否让 reader 或 writer 成为危险结构的一部分。

典型判断包括：

第一种：

```text
writer 已经 committed；
writer 还有 conflict out；
```

现在新增：

```text
reader -> writer
```

就形成：

```text
reader -> writer -> T2
```

如果 writer 已提交且 out 侧提交顺序满足条件，reader 需要失败。

第二种：

```text
writer 有 outConflicts 指向 prepared / committed 事务。
```

需要比较：

```text
prepareSeqNo
commitSeqNo
lastCommitBeforeSnapshot
```

第三种：

```text
reader 已经有 inConflicts；
writer 是 prepared；
```

需要避免 prepared writer 已经无法被中止的情况。

这个函数看起来分支很多。

但主线一直是：

```text
新增 reader -> writer 后；
writer 是否成为 pivot；
reader 是否已经有更早的 in 边；
out 侧事务是否已经足够早提交；
如果危险，当前事务或 writer 谁还能被安全中止。
```

这也是课程阅读时应该跟的路径。

不要把每个 flag 当孤立枚举背。

把它们都放回 dangerous structure。

## 11. commit-time 检查为什么仍然需要

冲突记录时已经检查过。

为什么提交还要检查？

因为有些危险结构只有在提交顺序变清楚后才能判断。

`PreCommit_CheckForSerializationFailure()` 处理的重点是：

```text
当前事务准备提交；
它可能是某个 pivot 的 out 侧；
如果让它提交，会不会让另一个 pivot 必须失败？
```

源码遍历：

```text
MySerializableXact->inConflicts
```

把其中每条 near conflict 当作：

```text
nearConflict->sxactOut -> MySerializableXact
```

然后再看 nearConflict 的 `sxactOut` 是否有自己的 `inConflicts`。

如果发现：

```text
farConflict->sxactOut == MySerializableXact
```

或某个未提交、非 read-only、未 doomed 的 far 侧事务存在，就可能需要中止 pivot。

为什么通常标记 pivot，而不是总是中止当前提交者？

源码注释给出工程理由：

```text
当前事务正在提交写入；
让它提交可以保证重试更可能前进；
如果中止 far conflict，重试可能立刻撞到同样结构。
```

但 prepared transaction 是例外。

prepared pivot 不能再随意中止。

这种情况下当前事务可能只能自杀。

这就是 `PREPARED` flag、`prepareSeqNo` 和 2PC 状态在 SSI 中必须出现的原因。

## 12. 生命周期与 cleanup 对 dangerous structure 的影响

`rw-conflict` 不是事务结束就一定删除。

因为 committed 事务仍可能作为 dangerous structure 的一端。

释放入口：

```text
ReleasePredicateLocks(isCommit, isReadOnlySafe)
```

提交时：

```text
SXACT_FLAG_COMMITTED
commitSeqNo = ++PredXact->LastSxactCommitSeqNo
```

如果当前事务有 conflict out 到已提交事务，会设置：

```text
SXACT_FLAG_CONFLICT_OUT
SeqNo.earliestOutConflictCommit
```

回滚时：

```text
SXACT_FLAG_DOOMED
SXACT_FLAG_ROLLED_BACK
```

它对别人的 dangerous structure 贡献可以被忽略。

read-only 事务有额外优化。

如果它被证明 safe，可以提前释放 predicate locks 和 conflict。

但普通 committed writable transaction 可能进入 finished list。

它会一直保留到不再有 overlapping serializable transaction 需要它。

这解释了为什么：

```text
pg_locks 里可能看到已经提交事务遗留的 SIREAD 信息。
```

也解释了为什么 cleanup 与 `SxactGlobalXmin` 有关。

这些细节第 36 节会展开。

本节只要抓住：

```text
dangerous structure 是跨事务结束边界的判断；
所以 conflict 和 predicate lock 的生命周期不能等同于普通锁。
```

## 13. 观测与诊断入口

第一入口是错误码。

捕获：

```text
SQLSTATE 40001
```

不要只匹配错误文本。

不同路径的 internal detail 可能不同。

第二入口是 `pg_locks`。

它只能看 SIREAD footprint。

看不到 `outConflicts` / `inConflicts` 链表。

查询：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple, granted
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

第三入口是 `pg_stat_activity`。

对 dangerous structure 诊断，重点不是等待事件，而是事务时间窗口：

```sql
SELECT pid, state, xact_start, backend_xmin, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;
```

长事务会扩大 overlap 窗口。

overlap 窗口越大，rw-conflict 组合越多。

第四入口是断点。

推荐断点：

```text
SetRWConflict
FlagRWConflict
OnConflict_CheckForSerializationFailure
PreCommit_CheckForSerializationFailure
ReleasePredicateLocks
```

每次命中看：

```text
reader->topXid
writer->topXid
reader->flags
writer->flags
reader->outConflicts
writer->inConflicts
prepareSeqNo
commitSeqNo
```

第五入口是 isolation test。

PostgreSQL 源码树中 `src/test/isolation/specs` 有 serializable 相关测试。

它们比手工 psql 更适合固定交错。

## 14. 常见误区

误区一：

```text
rw-conflict 表示两个事务同时写同一行。
```

不对。

那是写写冲突或 update conflict。

rw-conflict 表示读者的 snapshot 没看到 writer 的写入。

误区二：

```text
outConflicts 是别人指向我的边。
```

不对。

`outConflicts` 是从当前事务出去的边。

当前事务是 reader。

误区三：

```text
只要有 dangerous structure，就一定存在真实环。
```

不一定。

SSI 允许少量 false positive。

这换来不维护完整图。

误区四：

```text
COMMIT 失败说明 COMMIT 本身写坏了什么。
```

不对。

COMMIT 只是让提交顺序变成事实。

SSI 在这个边界上做最后检查。

误区五：

```text
调大 predicate lock 参数可以消除所有 serialization failure。
```

不对。

参数只能降低一部分因 coarse promotion 带来的 false positive。

真实业务依赖冲突仍会失败。

误区六：

```text
read-only 事务不会出现在 conflict graph。
```

不对。

read-only 可以有 conflict out。

只是 PostgreSQL 有 safe snapshot 优化。

## 15. 课堂实验

实验一：固定两事务 write skew。

使用上一节的 `on_call` 表。

两个事务都用：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

交错执行：

```text
A: SELECT count...
B: SELECT count...
A: UPDATE alice...
B: UPDATE bob...
A: COMMIT;
B: COMMIT;
```

记录哪个事务失败。

再反过来：

```text
B: COMMIT;
A: COMMIT;
```

比较失败点。

实验二：把 `UPDATE` 提前。

交错：

```text
A: SELECT count...
A: UPDATE alice...
B: SELECT count...
B: UPDATE bob...
```

观察失败是在 B 的读、B 的写，还是某个提交点。

把结果映射到：

```text
HeapCheckForSerializableConflictOut()
CheckForSerializableConflictIn()
PreCommit_CheckForSerializationFailure()
```

实验三：使用 `pg_locks` 看 footprint。

在两个事务 `SELECT` 后查询：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

如果执行计划是 seq scan，通常会看到 relation 级 footprint。

加索引并调整条件后，观察 footprint 是否变化。

实验四：用 gdb 看 edge。

断点：

```text
break FlagRWConflict
break SetRWConflict
break PreCommit_CheckForSerializationFailure
```

命中时打印：

```text
reader->topXid
writer->topXid
reader->flags
writer->flags
```

不要试图从 `pg_locks` 还原完整 conflict graph。

那张图只在内存链表里。

实验五：验证应用重试。

用脚本并发执行值班事务。

如果遇到 `40001`，重试整个事务。

比较：

- 不重试时失败率。
- 重试一次时成功率。
- 固定等待与指数退避的差异。

记录平均事务耗时。

这能把 SSI correctness 成本落到业务延迟。

## 16. 讨论题

1. 为什么 `rw-conflict` 的方向是 reader -> writer？

2. 如果只追踪 writer -> reader，会在哪些源码判断上变复杂？

3. 为什么 snapshot isolation 的异常一定包含相邻 `rw-conflict`？

4. 为什么 PostgreSQL 不在每次 COMMIT 做完整图环检测？

5. 为什么 `Tout` 的提交顺序会影响是否必须失败？

6. 什么情况下标记 pivot 比中止当前事务更有利于重试前进？

7. prepared transaction 为什么让失败选择变复杂？

8. `pg_locks` 为什么看不到 `outConflicts` / `inConflicts`？

9. 如果一个事务读很多范围但写很少，可能在哪些路径上制造 conflict？

10. 如果一个事务写很多范围但读很少，可能在哪些路径上触发 conflict in？

## 17. 本节小结

本节把 SSI 从现象推进到状态。

核心结论是：

```text
PostgreSQL 的 SERIALIZABLE 不维护完整依赖图；
它追踪并发事务之间的 rw-conflict；
并用 dangerous structure 做保守失败判断。
```

`rw-conflict` 的方向是：

```text
reader -> writer
```

它表示 reader 的 snapshot 没有包含 writer 的写入。

读路径通过 MVCC tuple 状态发现 conflict out。

写路径通过 SIREAD predicate lock 发现 conflict in。

两条路径都汇入：

```text
FlagRWConflict()
```

提交路径再通过：

```text
PreCommit_CheckForSerializationFailure()
```

保护提交顺序边界。

下一节继续讲：

```text
SIREAD predicate lock 的 target 到底如何表达 relation/page/tuple/index range；
为什么 coarse target 不破坏正确性，却会提高 false positive 和 abort 率。
```
