# PostgreSQL 索引访问方法与 predicate lock 粒度

## 课程定位

前置知识：已经理解 predicate lock target 是 relation/page/tuple 的物理近似，也知道 promotion 会把细粒度 footprint 合并成更粗粒度。

本节唯一主问题：

```text
为什么同一条 SERIALIZABLE 查询，换一个执行计划或索引访问方法，就可能改变 SIREAD 粒度、false positive 和 serialization failure 率？
```

核心矛盾：

```text
SQL 谓词是逻辑范围；
但 PostgreSQL 只能通过 heap scan、index AM 和 page/tuple 访问路径来近似表达这个范围。
```

学完后应能独立判断：

- heap seq scan 为什么只能 relation-level。
- btree 为什么可以用 leaf page-level 表达 range footprint。
- unsupported index AM 为什么退化成 index relation-level。
- bitmap scan 为什么是 index footprint 与 heap tuple footprint 的组合。
- index insert 为什么要检查 page-level predicate conflict。
- page split 为什么是 predicate lock 正确性问题。
- 为什么优化执行计划可能降低 SERIALIZABLE 失败率，但不能改变正确性目标。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 35 节讲 target。

第 36 节讲生命周期。

本节把 target 放回 access method。

很多 SERIALIZABLE 失败不是因为业务逻辑突然更危险。

而是因为 runtime footprint 变粗了。

典型变化包括：

```text
index scan 变成 seq scan；
btree scan 变成 bitmap heap scan；
支持 predicate lock 的 AM 换成不支持的 AM；
统计信息变化导致 plan 选择更大范围；
predicate lock promotion 把 page locks 合并成 relation lock。
```

这些变化不会让 PostgreSQL 变得不正确。

但会让更多事务被认为可能冲突。

所以本节的问题不是：

```text
哪种索引更快？
```

而是：

```text
哪种访问路径能更精确地表达读过的谓词范围？
```

这也是 SERIALIZABLE 诊断必须看 `EXPLAIN` 的原因。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
heap seq scan 在 SERIALIZABLE 下对 heap relation 取 SIREAD；
btree index scan 在相关 leaf pages 上取 SIREAD；
不支持 fine-grained predicate lock 的 index AM 在 index relation 上取 SIREAD；
heap tuple read 仍可能取 TID SIREAD；
index insert / heap write 通过 CheckForSerializableConflictIn() 对对应 target 检查读者。
```

本节的核心矛盾是：

```text
越靠近真实 key range，误 abort 越少；
但 AM 必须证明自己能在 split、vacuum、empty index、page reuse 等场景下不丢 predicate footprint。
```

因此 PostgreSQL 采用一个保守规则：

```text
AM 不能维护细粒度 footprint，就退化成 relation-level。
```

正确性优先。

性能和失败率通过更好的计划、更合适的索引和更细 AM 支持改善。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/heapam.c` | seq scan relation lock、heap tuple TID lock、heap conflict out。 |
| 2 | `src/backend/access/index/indexam.c` | `ampredlocks` fallback：AM 不支持时锁整个 index relation。 |
| 3 | `src/backend/access/nbtree/nbtsearch.c` | btree range scan 搜索、空索引 relation lock 与 retry。 |
| 4 | `src/backend/access/nbtree/nbtreadpage.c` | leaf page 上的 `PredicateLockPage()`。 |
| 5 | `src/backend/access/nbtree/nbtinsert.c` | index tuple insert 对 page target 做 `CheckForSerializableConflictIn()`。 |
| 6 | `src/include/access/amapi.h` | index AM capability 中的 `ampredlocks`。 |
| 7 | `src/backend/storage/lmgr/predicate.c` | target 创建、promotion、page split / combine。 |
| 8 | `src/backend/storage/lmgr/README-SSI` | 各类 index AM predicate locking 的设计说明。 |
| 9 | `doc/src/sgml/indexam.sgml` | AM 对 predicate lock 支持的文档边界。 |

推荐阅读顺序：

```text
heapam.c 的 seq scan relation lock
  -> indexam.c 的 ampredlocks fallback
  -> nbtreadpage.c 的 leaf page PredicateLockPage
  -> nbtinsert.c 的 CheckForSerializableConflictIn
  -> README-SSI 的 AM 粒度说明
```

不要孤立读 btree。

要对比：

```text
heap 无 range 结构
btree 有 leaf key order
unsupported AM 只能保守覆盖
```

## 4. 从 runtime 现象进入

准备数据：

```sql
DROP TABLE IF EXISTS idx_pred_demo;

CREATE TABLE idx_pred_demo
(
    id bigint PRIMARY KEY,
    account_id int NOT NULL,
    status text NOT NULL,
    payload text
);

INSERT INTO idx_pred_demo
SELECT g,
       g % 1000,
       CASE WHEN g % 5 = 0 THEN 'open' ELSE 'closed' END,
       repeat('x', 20)
FROM generate_series(1, 50000) AS g;
```

查询：

```sql
SELECT count(*)
FROM idx_pred_demo
WHERE account_id BETWEEN 10 AND 20
  AND status = 'open';
```

第一轮不建 `(account_id, status)` 索引。

SERIALIZABLE 下执行，常见是 seq scan 或较粗路径。

`pg_locks` 可能显示 heap relation-level SIReadLock。

第二轮创建索引：

```sql
CREATE INDEX idx_pred_demo_account_status_idx
ON idx_pred_demo(account_id, status);

ANALYZE idx_pred_demo;
```

同样 SERIALIZABLE 查询，如果走 btree index scan，footprint 可能变成 index page-level，加上必要的 heap tuple locks。

第三轮执行 concurrent insert：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

INSERT INTO idx_pred_demo
VALUES (900001, 15, 'open', 'new');

COMMIT;
```

这个 insert 进入查询范围。

它应该能与早读者形成 conflict。

如果插入 `account_id = 900`，理论上不在范围。

但如果早读者只有 relation-level footprint，也可能产生更粗的冲突。

这就是本节现象：

```text
同一逻辑谓词；
访问路径决定 footprint 粒度；
footprint 粒度决定 false positive 的边界。
```

## 5. heap seq scan：为什么只能 relation-level

heap 本身没有按谓词排序。

seq scan 访问的是整个 relation 的 heap pages。

它可以在 executor 层过滤：

```text
account_id BETWEEN 10 AND 20
status = open
```

但 heap AM 不知道一个未来插入的 tuple 是否会满足这个 SQL predicate。

如果只对已返回 tuple 加 SIREAD，就会漏掉 phantom：

```text
reader 扫描时没有某个 tuple；
writer 后来插入一个满足 predicate 的 tuple；
writer 找不到 reader footprint；
rw-conflict 漏掉。
```

因此 `heapam.c` 在 seq scan / sample scan 下：

```text
PredicateLockRelation(relation, snapshot)
```

源码注释说明：

```text
在 heap scan 中没有更细粒度的 range 可以锁。
```

这会覆盖整个 heap relation。

正确性是安全的。

但插入同表任意位置都更容易被认为冲突。

所以 SERIALIZABLE workload 中，减少不必要 seq scan 不是只为了 I/O。

它还可能降低 false positive。

## 6. heap tuple read：TID lock 的角色

即使使用 index scan，最终通常还要访问 heap tuple。

当 heap tuple 对当前 snapshot 可见时，读取路径可能调用：

```text
PredicateLockTID(relation, &(tuple->t_self), snapshot, tuple_xmin)
```

它表达：

```text
当前事务读过这个具体 heap tuple version 所在的 TID。
```

如果后续 writer 更新或删除这个 tuple，写路径通过：

```text
CheckForSerializableConflictIn(relation, tid, block)
```

能检查 tuple/page/relation target。

TID lock 对已存在 tuple 很精确。

但它不能覆盖 gap。

如果查询结果为空，或者未来 insert 进入范围，TID lock 没法表达。

所以 index page SIREAD 与 heap TID SIREAD 需要配合。

一个 btree range scan 可能同时留下：

- index leaf page footprint，覆盖未来插入。
- heap tuple footprint，覆盖已读 tuple 的更新或删除。

这就是为什么只看 heap 或只看 index 都不完整。

## 7. `ampredlocks` fallback

index AM 能不能细粒度支持 predicate lock，由 AM capability 表达。

在 `indexam.c` 的 scan 初始化中：

```text
if (!(indexRelation->rd_indam->ampredlocks))
    PredicateLockRelation(indexRelation, snapshot);
```

含义是：

```text
如果 AM 不负责细粒度 predicate lock；
任何 SERIALIZABLE index scan 都对整个 index relation 取 SIREAD。
```

这不是偷懒。

这是 access method contract。

一个 AM 如果宣称支持 fine-grained predicate lock，就必须保证：

- scan 读到的范围有 target。
- insert 能检查对应 target。
- split / overflow / combine 不丢 target。
- vacuum / page reuse 不破坏 target 语义。
- empty search 不留下 race。

如果做不到，退化成 index relation-level 是正确选择。

代价是：

```text
该 index 上任何 concurrent insert 更容易与 reader 冲突。
```

这就是为什么 AM 支持粒度不是单纯优化。

它影响 SERIALIZABLE false positive 的上限。

## 8. btree leaf page：range footprint 的主路径

btree 有 key order。

range scan 会定位到 leaf page，然后顺序读取相关 leaf pages。

在 `nbtreadpage.c`：

```text
PredicateLockPage(rel, so->currPos.currPage, scan->xs_snapshot)
```

这表示：

```text
当前 scan 读取了这个 leaf page 覆盖的 key range。
```

如果未来 insert 进入这个 page，`nbtinsert.c` 会调用：

```text
CheckForSerializableConflictIn(rel, NULL, BufferGetBlockNumber(insertstate.buf))
```

由于 `tid = NULL`，检查的是：

```text
page target
relation target
```

这样 reader 的 page SIREAD 能与 writer 的 page insert 对上。

为什么不是 index tuple-level？

因为 range phantom 关心的是 gap。

新插入的 index tuple 在 reader 扫描时不存在。

reader 不可能锁住一个尚不存在的 index tuple。

leaf page 是 btree 能维护的实际范围单位。

这个单位不是完美 SQL range。

一个 page 可能覆盖比查询更宽的 key 区间。

但它比整个 index relation 更细。

这就是 btree 在 SERIALIZABLE 下比 heap seq scan 更少误 abort 的核心原因之一。

## 9. 空 btree：relation lock 与重搜

btree 空索引路径很特殊。

`nbtsearch.c` 中，如果 `_bt_search()` 找不到有效 buffer，说明索引可能完全为空。

SERIALIZABLE 下会：

```text
PredicateLockRelation(rel, scan->xs_snapshot)
_bt_search(...) again
```

为什么不直接返回空结果？

因为存在窗口：

```text
reader 第一次搜索：索引为空
writer 插入第一条记录
reader 才加 relation SIREAD
```

如果不重搜，writer 看不到 reader 的 footprint。

reader 也没有覆盖第一条记录所在 page。

所以必须：

```text
拿 relation-level predicate lock
再重搜一次
```

如果重搜后仍为空，relation lock 覆盖未来第一条插入。

如果重搜后不为空，就回到正常 leaf page 逻辑。

这个例子说明：

```text
predicate lock 粒度不是只看数据结构；
还要关掉 search 与 lock acquisition 之间的并发窗口。
```

## 10. page split：粒度正确性的维护成本

btree leaf page 不是稳定 range。

insert 可能 split。

split 后，一个旧 page 的 key range 分到两个 page。

如果 reader 曾经锁过 old page，而 split 后 writer 插入 new page：

```text
new page 如果没有继承 SIREAD；
writer 就看不到 reader。
```

所以 btree split 路径会调用：

```text
PredicateLockPageSplit(rel, oldblkno, newblkno)
```

它把 old page 上的 predicate locks 转移或复制到新 target。

同理，page combine / deletion 等路径也需要维护。

predicate lock manager 提供：

```text
PredicateLockPageCombine()
TransferPredicateLocksToHeapRelation()
PageIsPredicateLocked()
```

这些入口说明：

```text
细粒度 predicate lock 的成本不只在 scan；
还在 AM 结构变化时维护 footprint 不丢失。
```

一个 AM 如果不能可靠维护这些状态，就不应该宣称 fine-grained support。

## 11. bitmap scan 的组合边界

bitmap scan 容易被误解。

它先扫描 index 建 bitmap。

再访问 heap pages。

README-SSI 对 heap locking 的说明指出：

```text
bitmap scan 已经扫描 index 并锁了覆盖 predicate 的 index pages；
但仍需要对匹配 heap tuples 加锁。
```

所以 bitmap scan 的 footprint 往往是组合：

```text
index side:
  page/range footprint 覆盖 future insert。

heap side:
  tuple footprint 覆盖已读 tuple 的 update/delete。
```

如果 bitmap heap scan 因 lossy bitmap 访问更多 heap pages，heap tuple footprint 与 executor recheck 也会影响实际读 footprint。

本节不展开 bitmap 内部。

只保留诊断规则：

```text
bitmap scan 不是简单等价于 index scan；
它可能留下 index predicate locks 和 heap predicate locks 的组合。
```

这会影响 `pg_locks` 观察结果。

不要看到 heap tuple SIReadLock 就以为没有 index range footprint。

也不要看到 index page SIReadLock 就以为 heap tuple 更新不会冲突。

## 12. 正确性机制与误 abort

正确性要求：

```text
任何会改变 reader 结果的 writer，都必须能发现 reader footprint。
```

这意味着：

```text
footprint 可以更粗；
不能更窄。
```

执行计划影响的是误 abort，而不是 correctness 目标。

同一查询：

```text
seq scan
  -> heap relation SIREAD
  -> 很宽。

btree index scan
  -> index leaf page SIREAD + heap tuple SIREAD
  -> 较窄。

unsupported AM
  -> index relation SIREAD
  -> 可能很宽。

promotion
  -> page/relation SIREAD
  -> 随内存压力变宽。
```

误 abort 的机制：

```text
writer 写到了粗 footprint；
实际 SQL predicate 可能无关；
但系统无法从 coarse target 证明无关；
于是记录 rw-conflict；
后续 dangerous structure 可能让事务失败。
```

这不是错误。

这是保守近似。

优化方向是：

- 改善索引支持，让 footprint 更接近谓词。
- 避免不必要 seq scan。
- 降低长事务 overlap。
- 避免一次事务读过过大范围。
- 合理设置 predicate lock 参数，减少过早 promotion。

## 13. 成本、资源与跨模块传播

索引粒度成本会跨模块传播。

planner 选择 plan：

```text
seq scan / index scan / bitmap scan
```

access method 创建 footprint：

```text
relation / page / tuple
```

predicate lock manager 维护状态：

```text
target hash / lock hash / local hash / promotion
```

writer 检查 conflict：

```text
heap write / index insert
```

transaction manager 处理失败：

```text
40001 / retry
```

因此一个看似 planner 层的小变化，会在 SERIALIZABLE 下放大成业务延迟变化。

例如统计信息变旧导致 seq scan。

查询仍然正确。

但 relation-level SIREAD 变多。

insert workload 的 conflict in 变多。

serialization failure 变多。

应用重试增多。

平均延迟上升。

这条链路必须完整诊断。

## 14. 观测与诊断入口

先看 plan：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*)
FROM idx_pred_demo
WHERE account_id BETWEEN 10 AND 20
  AND status = 'open';
```

再看 SIReadLock：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

再看失败：

```text
SQLSTATE 40001
```

源码断点：

```text
heap_beginscan
PredicateLockRelation
PredicateLockPage
PredicateLockTID
index_beginscan_internal
_bt_readpage
_bt_first
_bt_doinsert
CheckForSerializableConflictIn
```

调试顺序：

```text
先确认 plan。
再确认哪个 AM 入口创建 SIREAD。
再确认 writer 写入哪个 target。
最后看 FlagRWConflict 是否进入 dangerous structure。
```

配置入口：

```sql
SHOW random_page_cost;
SHOW cpu_tuple_cost;
SHOW enable_seqscan;
SHOW max_pred_locks_per_transaction;
SHOW max_pred_locks_per_relation;
SHOW max_pred_locks_per_page;
```

不要在生产里随意禁用 plan。

实验可以用 GUC 固定 plan。

诊断要回到统计信息、索引设计和事务范围。

## 15. 常见误区

误区一：

```text
索引只影响性能，不影响 SERIALIZABLE 失败率。
```

不对。

索引路径影响 predicate lock 粒度。

误区二：

```text
btree page SIReadLock 精确等于 SQL 范围。
```

不对。

它是 leaf page 覆盖范围的近似。

误区三：

```text
unsupported AM 不能用于 SERIALIZABLE。
```

不对。

可以用。

但会退化到 index relation-level footprint。

误区四：

```text
seq scan 下只锁匹配 tuple 就够了。
```

不对。

未来插入的 matching tuple 不存在，无法被 tuple lock 覆盖。

误区五：

```text
page split 是索引结构问题，和 SSI 无关。
```

不对。

page split 会改变 page target 对应的 key range。

必须转移 predicate locks。

误区六：

```text
调计划能消除所有 serialization failure。
```

不对。

调计划只能减少 false positive。

真实 dangerous structure 仍然必须失败。

## 16. 课堂实验

实验一：seq scan 与 index scan 对比。

```sql
DROP TABLE IF EXISTS idx_pred_demo;
CREATE TABLE idx_pred_demo(id bigint PRIMARY KEY, account_id int, status text, payload text);
INSERT INTO idx_pred_demo
SELECT g, g % 1000, CASE WHEN g % 5 = 0 THEN 'open' ELSE 'closed' END, repeat('x', 20)
FROM generate_series(1, 50000) AS g;
ANALYZE idx_pred_demo;
```

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SET enable_indexscan = off;
SET enable_bitmapscan = off;
SELECT count(*) FROM idx_pred_demo
WHERE account_id BETWEEN 10 AND 20 AND status = 'open';
```

Session B 查 `pg_locks`。

记录 relation/page/tuple。

实验二：创建 btree。

```sql
CREATE INDEX idx_pred_demo_account_status_idx
ON idx_pred_demo(account_id, status);
ANALYZE idx_pred_demo;
```

Session A：

```sql
ROLLBACK;
BEGIN ISOLATION LEVEL SERIALIZABLE;
SET enable_seqscan = off;
SELECT count(*) FROM idx_pred_demo
WHERE account_id BETWEEN 10 AND 20 AND status = 'open';
```

再次查 `pg_locks`。

比较 footprint。

实验三：range insert。

Session B：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
INSERT INTO idx_pred_demo VALUES (900001, 15, 'open', 'new');
COMMIT;
```

Session A：

```sql
COMMIT;
```

观察失败点。

实验四：无关 insert。

重复实验三，但插入：

```sql
INSERT INTO idx_pred_demo VALUES (900002, 900, 'closed', 'new');
```

比较 seq scan footprint 与 btree footprint 下的失败概率。

实验五：源码断点。

设置：

```text
break PredicateLockRelation
break PredicateLockPage
break PredicateLockTID
break CheckForSerializableConflictIn
```

在 seq scan、index scan、bitmap scan 下分别运行查询。

记录：

```text
relation oid
block number
offset number
```

把它映射回 `pg_locks` 输出。

## 17. 讨论题

1. heap seq scan 为什么不能只锁匹配 tuple？

2. btree leaf page 为什么能表达 range，但不是完美 range？

3. index AM 如果声明 `ampredlocks`，必须维护哪些边界？

4. 为什么 empty btree search 要 relation lock 后重搜？

5. bitmap scan 的 index footprint 和 heap footprint 分别覆盖什么？

6. page split 丢 predicate lock 会造成什么 anomaly？

7. 调整索引为什么可能降低 `40001`，但不能替代事务重试？

8. 什么时候应该先改 SQL / 索引，而不是先调 predicate lock 参数？

9. relation-level SIReadLock 在哪些场景是预期结果？

10. 如何用 `EXPLAIN` 与 `pg_locks` 共同定位 false positive 来源？

## 18. 本节小结

本节的核心结论是：

```text
SERIALIZABLE 的读 footprint 由实际访问路径决定。
```

heap seq scan 通常只能 relation-level。

btree index scan 可以用 leaf page-level 表达 range。

unsupported index AM 退化成 index relation-level。

bitmap scan 是 index footprint 与 heap tuple footprint 的组合。

index insert 必须检查 page / relation target。

page split 必须维护 predicate lock 转移。

执行计划不改变 SQL 语义。

但它改变 SSI footprint。

footprint 变粗，false positive 和 retry 成本就可能上升。

下一节进入本章 SSI 小单元的最后一课：

```text
只读事务、safe snapshot 与 DEFERRABLE；
为什么有些只读事务可以用等待换取后续 SSI 检测成本降低。
```
