# PostgreSQL predicate lock 的范围表达

## 课程定位

前置知识：已经理解 `rw-conflict` 的方向是 reader -> writer，也知道写路径通过 SIREAD footprint 发现早读者。

本节唯一主问题：

```text
SQL 谓词是逻辑条件，PostgreSQL 为什么把 SIREAD footprint 表达成 relation / page / tuple / index page 这些物理 target？
```

核心矛盾：

```text
SSI 需要知道一个事务读过哪个谓词范围；
但 executor 运行时真正接触的是 heap tuple、heap page、relation、index leaf page 和 access method 回调。
```

学完后应能独立判断：

- `PREDICATELOCKTARGETTAG` 为什么只有四个 32-bit 字段。
- relation / page / tuple 三种 target 如何编码。
- 为什么 index range 通常不是保存 SQL predicate，而是保存 index leaf page footprint。
- 为什么 coarse predicate lock 不破坏正确性，却会增加 false positive。
- 为什么 seq scan 往往产生 relation-level SIREAD。
- 为什么 btree 空索引搜索要拿 relation-level predicate lock 并重试。
- 为什么 page split / combine 需要转移 predicate lock。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 33 节说明了 snapshot isolation 的异常。

第 34 节说明了 `rw-conflict` 和 dangerous structure。

本节回答一个更具体的问题：

```text
writer 怎么知道自己写到了 reader 曾经读过的范围？
```

如果 reader 读的是：

```sql
SELECT *
FROM orders
WHERE customer_id = 42
  AND status = 'open';
```

逻辑谓词是：

```text
customer_id = 42 AND status = open
```

但 PostgreSQL 没有把这段 SQL 表达式原样放进 predicate lock table。

它把读 footprint 映射到实际访问过的对象：

```text
heap relation
heap page
heap tuple
index relation
index leaf page
```

这就是本节的主线。

predicate lock target 不是语义完美的 predicate algebra。

它是 SSI 在 PostgreSQL heap / index AM 上可维护的物理近似。

只要这个近似覆盖真实读范围，正确性就成立。

如果覆盖范围过宽，就会产生 false positive。

false positive 的 runtime 表现不是阻塞。

而是更高的 serialization failure 率。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
SERIALIZABLE 读路径把 SQL 谓词映射为 PREDICATELOCKTARGETTAG；
target 可能是 relation、page 或 tuple；
index range 在 btree 中主要用 leaf page predicate lock 表达；
写路径按 tuple -> page -> relation 检查覆盖 target；
page split、page combine、vacuum 和 access method fallback 负责让 target 不丢失。
```

本节的核心矛盾是：

```text
逻辑谓词越精确，false positive 越少；
但运行时只能以有限 shared memory 和 AM 可见对象维护 footprint。
```

PostgreSQL 的选择是：

```text
用物理 target 保守覆盖逻辑谓词；
允许 coarse lock 和 promotion；
用失败重试处理剩余不确定性。
```

这意味着：

```text
predicate lock 的粒度取决于执行计划和 access method 能力。
```

同一个 SQL，在 seq scan、btree index scan、bitmap scan 或 unsupported index AM 下，SIREAD footprint 可能完全不同。

这也是后续诊断 serialization failure 时必须看执行计划的原因。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/predicate_internals.h` | `PREDICATELOCKTARGETTAG`、target type、relation/page/tuple 宏。 |
| 2 | `src/backend/storage/lmgr/predicate.c` | `PredicateLockRelation()`、`PredicateLockPage()`、`PredicateLockTID()`、promotion 和 child cleanup。 |
| 3 | `src/include/storage/predicate.h` | AM 可调用的 predicate lock 公共 API。 |
| 4 | `src/backend/access/heap/heapam.c` | seq scan relation lock、visible tuple TID lock、heap write conflict target。 |
| 5 | `src/backend/access/index/indexam.c` | `ampredlocks` 不支持时退化为 index relation lock。 |
| 6 | `src/backend/access/nbtree/nbtsearch.c` | btree range scan、空索引 relation lock 与 retry。 |
| 7 | `src/backend/access/nbtree/nbtreadpage.c` | btree leaf page predicate lock。 |
| 8 | `src/backend/access/nbtree/nbtinsert.c` | index insert 对 page target 做 conflict in。 |
| 9 | `src/backend/utils/adt/lockfuncs.c` | `pg_locks` 如何把 target 展示成 relation/page/tuple。 |

推荐阅读顺序：

```text
先读 predicate_internals.h 的 tag 编码
  -> 读 PredicateLockRelation/Page/TID
  -> 读 heapam.c 的 seq scan / tuple read
  -> 读 indexam.c 的 ampredlocks fallback
  -> 读 nbtsearch/nbtreadpage/nbtinsert 的 btree page footprint
  -> 最后读 lockfuncs.c 的 pg_locks 展示
```

本节不要把目标写成抽象层次图就停。

必须回到 runtime：

```text
同一个 SQL 为什么有时留下 relation SIREAD；
有时留下 page SIREAD；
有时留下 tuple SIREAD；
为什么这些差异改变 abort 率。
```

## 4. 从 runtime 现象进入

准备一张表：

```sql
DROP TABLE IF EXISTS pred_demo;

CREATE TABLE pred_demo
(
    id int PRIMARY KEY,
    k int NOT NULL,
    v text
);

INSERT INTO pred_demo
SELECT g, g % 100, repeat('x', 20)
FROM generate_series(1, 10000) AS g;
```

第一轮不建 `k` 索引。

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

SELECT count(*)
FROM pred_demo
WHERE k BETWEEN 10 AND 20;
```

另一个会话观察：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

常见现象是对 heap relation 有 `SIReadLock`。

因为 seq scan 读的是整个 heap relation。

第二轮建索引：

```sql
CREATE INDEX pred_demo_k_idx ON pred_demo(k);
ANALYZE pred_demo;
```

重新开启 SERIALIZABLE 事务执行同样查询。

如果计划走 btree index scan，`pg_locks` 中可能出现 index page 或 heap tuple/page 相关 footprint。

第三轮让另一个 SERIALIZABLE 事务插入：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;

INSERT INTO pred_demo VALUES (20001, 15, 'new');

COMMIT;
```

这个 insert 进入了 A 的谓词范围。

如果两个事务的依赖组合危险，可能出现 serialization failure。

注意：

```text
失败率和 lock target 粒度有关。
```

没有索引时，relation-level footprint 很粗。

它可能让更多无关 insert 也冲突。

有 btree index 时，page-level footprint 通常更窄。

但它仍然不是 SQL predicate 的精确表达式。

它是 index leaf page 范围近似。

## 5. `PREDICATELOCKTARGETTAG` 的编码

核心结构在：

```text
src/include/storage/predicate_internals.h
```

它只有四个字段：

```text
locktag_field1
locktag_field2
locktag_field3
locktag_field4
```

这些字段通过宏解释。

relation target：

```text
field1 = db oid
field2 = rel oid
field3 = InvalidBlockNumber
field4 = InvalidOffsetNumber
```

page target：

```text
field1 = db oid
field2 = rel oid
field3 = block number
field4 = InvalidOffsetNumber
```

tuple target：

```text
field1 = db oid
field2 = rel oid
field3 = block number
field4 = offset number
```

target type 不是单独字段。

它由 `field3` 和 `field4` 是否 invalid 推导：

```text
field4 valid    -> tuple
field3 valid    -> page
otherwise       -> relation
```

这解释了一个重要边界：

```text
predicate lock target 是 relation/page/tuple 形状；
不是任意 SQL expression。
```

对于 index range，PostgreSQL 仍然用 relation/page 形状表达。

区别在于 relation oid 是 index relation。

btree 的 leaf page SIREAD 表示：

```text
这个 serializable scan 读过这个 index leaf page 覆盖的 key range。
```

它不是把 `k BETWEEN 10 AND 20` 存进 lock table。

## 6. target 与 lock 的双层结构

predicate lock table 不是一个简单 set。

它有两层：

```text
PREDICATELOCKTARGET
PREDICATELOCK
```

`PREDICATELOCKTARGET` 表示被锁的对象。

它的 key 是：

```text
PREDICATELOCKTARGETTAG
```

它有一个列表：

```text
predicateLocks
```

列表中每个元素是某个 serializable transaction 在这个 target 上的 SIREAD。

`PREDICATELOCK` 的 key 是：

```text
myTarget + myXact
```

它也挂在两个方向：

- target 的 `predicateLocks` 列表。
- sxact 的 `predicateLocks` 列表。

这样写路径可以从 target 找 reader。

清理路径可以从 transaction 找它持有的所有 locks。

创建入口：

```text
CreatePredicateLock(targettag, targethash, sxact)
```

它会：

- 在 `PredicateLockTargetHash` 中创建或找到 target。
- 在 `PredicateLockHash` 中创建或找到 lock。
- 把 lock 挂到 target list。
- 把 lock 挂到 sxact list。

这里需要同时保护：

```text
SerializablePredicateListLock
PredicateLockHashPartitionLock(hashcode)
```

如果是 parallel mode，还需要 per-sxact predicate list lock。

这个 ownership 解释了为什么 target 和 lock 要分开。

target 是“对象”。

lock 是“某个事务对这个对象的读 footprint”。

## 7. relation target：最粗但最稳的范围

relation target 是最粗粒度。

它表示：

```text
这个事务读过整个 relation 或无法更精确地表达范围。
```

heap seq scan 会用它。

在 `heapam.c` 中，seq scan / sample scan 的初始化路径会：

```text
PredicateLockRelation(relation, snapshot)
```

源码注释说明原因：

```text
heap scan 没有 index range；
为了和新插入 tuple 冲突，需要锁整个 relation。
```

这有两个后果。

正确性后果：

```text
任何插入这个 relation 的 concurrent serializable writer 都能看到这个 reader。
```

性能后果：

```text
即使 insert 的 tuple 不满足原 SQL predicate，也可能因为 relation-level footprint 产生 conflict。
```

这就是 false positive。

relation target 还用于 index AM fallback。

在 `indexam.c`：

```text
if (!(indexRelation->rd_indam->ampredlocks))
    PredicateLockRelation(indexRelation, snapshot);
```

如果某个 index AM 没有声明自己支持 fine-grained predicate locking，PostgreSQL 对整个 index relation 加 SIREAD。

这仍然正确。

但会更容易让任意 index insert 与 reader 冲突。

## 8. page target：btree range 的主要表达

page target 表示：

```text
这个事务读过 relation 的某个 block。
```

在 heap 上，page target 可以作为 tuple target 的父级或 promotion 结果。

在 btree 上，page target 更重要。

btree index scan 会在读取 leaf page 时：

```text
PredicateLockPage(rel, so->currPos.currPage, scan->xs_snapshot)
```

入口位于：

```text
src/backend/access/nbtree/nbtreadpage.c
```

这表达的是：

```text
我扫描过这个 leaf page 覆盖的 key 区间；
如果后续 insert 进入这个 leaf page，可能改变我的 range scan 结果。
```

index insert 路径在 `nbtinsert.c` 中调用：

```text
CheckForSerializableConflictIn(rel, NULL, BufferGetBlockNumber(insertstate.buf))
```

注意参数：

```text
tid = NULL
blkno = insert page
```

这说明 index insert 主要按 page target 触发 conflict。

btree page target 不是“物理页刚好包含我读到的 tuple”这么简单。

它近似代表一个 key range。

因此 page split 必须处理 predicate locks。

如果原 page 分裂成两个 page，而 SIREAD 还留在旧 page 上，新 page 上的 insert 可能看不到 reader。

所以 `predicate.c` 提供：

```text
PredicateLockPageSplit()
PredicateLockPageCombine()
```

btree split 路径会调用 page split 转移逻辑。

这就是 page target 的正确性边界。

## 9. tuple target：已读 tuple 的精确 footprint

tuple target 表示：

```text
这个事务读过某个具体 heap tuple。
```

heap 读取可见 tuple 时会调用：

```text
PredicateLockTID(relation, &(tuple->t_self), snapshot, HeapTupleHeaderGetXmin(tuple->t_data))
```

入口在 `heapam.c` 的 tuple fetch / scan 路径。

`PredicateLockTID()` 有一个关键优化：

```text
如果这个 tuple 是当前事务自己写的，直接返回。
```

原因是当前事务已经有写锁语义。

不需要再用 SIREAD 表达自己读自己写。

`PredicateLockTID()` 还会先快速检查 relation target：

```text
如果当前事务已经有 relation-level SIREAD，就不需要 tuple-level。
```

然后再创建 tuple target。

tuple target 最精确。

但它不能表达“没读到的 gap”。

例如：

```sql
SELECT *
FROM pred_demo
WHERE k = 15;
```

如果当前没有任何 `k = 15` 的 tuple，tuple target 没法表达这个空结果。

必须靠 index range/page 或 relation target 覆盖。

这就是 predicate locking 与普通 tuple locking 的根本差异。

谓词依赖不仅来自“读到了什么”。

还来自“没读到但如果存在就会被读到什么”。

## 10. target 层级与 promotion

target 有父子层级：

```text
tuple -> page -> relation
```

`GetParentPredicateLockTag()` 实现这条链。

relation 没有父级。

page 的父级是 relation。

tuple 的父级是 page。

当事务请求一个 target 时，`PredicateLockAcquire()` 会先检查：

```text
PredicateLockExists(target)
CoarserLockCovers(target)
```

如果当前事务已经有更粗 target，就不再创建更细 target。

如果没有覆盖，就创建 lock。

创建后会调用：

```text
CheckAndPromotePredicateLockRequest()
```

它用 backend-local `LocalPredicateLockHash` 统计 child locks。

如果某个父级下 child 数超过阈值，就提升到更粗 target。

例如：

```text
一个 page 下 tuple locks 太多
  -> promote to page lock

一个 relation 下 page locks 太多
  -> promote to relation lock
```

提升后会删除被覆盖的更细 target locks。

这节要强调：

```text
promotion 是资源保护，不是语义增强。
```

它让 footprint 更粗。

正确性仍然成立。

false positive 会增加。

观测上，你可能看到同一个事务最初有很多 tuple/page SIReadLock，随后变成 relation SIReadLock。

## 11. 空索引和 retry 窗口

btree 中有一个容易忽略的边界。

在 `nbtsearch.c`，如果搜索发现索引完全为空：

```text
_bt_search()
  -> no valid buffer
```

SERIALIZABLE 下会：

```text
PredicateLockRelation(rel, scan->xs_snapshot)
_bt_search() again
```

为什么要重试？

因为第一次 `_bt_search()` 和 `PredicateLockRelation()` 之间有窗口。

在这个窗口里，另一个事务可能插入第一条 index tuple。

如果读者不重试：

```text
读者认为索引为空；
writer 插入；
writer 没看到读者的 SIREAD；
读者也没有覆盖新 tuple 的 range。
```

这会漏掉 rw-conflict。

所以空索引路径必须：

```text
先拿 relation-level predicate lock；
再重新搜索确认是否仍为空。
```

这段代码很适合作为阅读 SSI 的例子。

它体现了 PostgreSQL 不把 predicate lock 当事后装饰。

它必须和 access method 的搜索窗口严密配合。

否则正确性会从一个很小的 race 漏出去。

## 12. page split / combine 与 target 转移

index page 不是静态对象。

btree insert 可能 page split。

vacuum 或 page deletion 可能让 page 被复用。

如果 predicate lock target 只看物理 block number，不处理结构变化，就会出现两类错误。

第一类是漏报。

读者锁了 old page。

split 后新 key range 去了 new page。

writer 插入 new page。

如果 SIREAD 没有复制到 new page，writer 找不到 reader。

第二类是误报扩大。

旧 page 被复用为完全不同 range。

如果旧 SIREAD 无期限留在物理 page 上，后续无关 writer 会不断冲突。

PostgreSQL 提供维护入口：

```text
PredicateLockPageSplit()
PredicateLockPageCombine()
TransferPredicateLocksToHeapRelation()
PageIsPredicateLocked()
```

不同 AM 需要在 split、combine、vacuum、cleanup 时维护 predicate lock 语义。

这解释了为什么 predicate locking 是 access method contract 的一部分。

`ampredlocks` 不是一个性能小开关。

它表示：

```text
这个 AM 能否正确维护自己的 predicate lock 粒度。
```

如果不能，就必须退化到 relation-level，牺牲精度保正确性。

## 13. 生命周期 / ownership / cleanup

target 的生命周期由 shared hash table 管理。

创建：

```text
PredicateLockAcquire()
  -> CreatePredicateLock()
  -> PredicateLockTargetHash
  -> PredicateLockHash
```

持有：

```text
target list 持有所有锁这个 target 的事务；
sxact list 持有这个事务的所有 target。
```

释放：

```text
ReleasePredicateLocks()
  -> ReleaseOneSerializableXact()
  -> 删除 sxact 上的 predicate locks
  -> target list 为空时删除 target
```

promotion 也会删除 target。

当更粗 target 覆盖更细 target 时：

```text
DeleteChildTargetLocks()
RemoveTargetIfNoLongerUsed()
DecrementParentLocks()
```

ERROR / abort：

```text
transaction rollback
  -> sxact 标记 doomed / rolled back
  -> predicate locks 可释放
```

commit：

```text
committed sxact 的 predicate locks 可能继续保留
```

原因同前一节：

```text
仍有 overlapping serializable transaction 可能需要这条读 footprint 完成 conflict 判断。
```

因此 target 不归 SQL 语句所有。

也不归 executor node 所有。

它归 serializable transaction 的 SSI 生命周期所有。

## 14. 正确性、异常与成本

正确性要求：

```text
target 必须覆盖真实读谓词范围。
```

它可以比真实范围大。

不能比真实范围小。

如果太小，会漏掉 rw-conflict。

如果太大，会增加 false positive。

典型异常与 fallback：

| 场景 | PostgreSQL 行为 | 结果 |
| --- | --- | --- |
| heap seq scan | relation-level SIREAD | 正确但粗。 |
| unsupported index AM | index relation-level SIREAD | 正确但粗。 |
| btree range scan | leaf page-level SIREAD | 较细但仍是近似。 |
| 空 btree | relation-level SIREAD 并重试 | 关闭空索引 race。 |
| child locks 太多 | promotion 到 page/relation | 正确但 false positive 增加。 |
| shared memory 不足 | 报 out of shared memory | 需要调参或降低 footprint。 |

成本来自：

- target hash 查找。
- partition LWLock。
- local hash 维护。
- promotion 扫描与 child cleanup。
- `pg_locks` 采样时的全表复制。
- 更粗 target 带来的重试成本。

这些成本随以下因素上升：

- serializable 并发事务数。
- 每事务读取 page / tuple 数。
- 访问计划是否走 seq scan。
- index AM 是否支持 fine-grained predicate lock。
- 参数阈值是否导致频繁 promotion。

## 15. 观测与诊断入口

最直接入口：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple, granted
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

观察规则：

```text
relation 非空、page/tuple 为空
  -> relation target。

relation + page，tuple 为空
  -> page target。

relation + page + tuple
  -> tuple target。
```

注意：

```text
pg_locks 展示的是一瞬间复制出来的 predicate lock 状态；
不是 conflict graph；
也不是 SQL predicate 文本。
```

结合执行计划：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*)
FROM pred_demo
WHERE k BETWEEN 10 AND 20;
```

如果是 seq scan，预期 relation-level footprint。

如果是 index scan，继续看是否有 index page SIREAD。

结合 GUC：

```sql
SHOW max_pred_locks_per_transaction;
SHOW max_pred_locks_per_relation;
SHOW max_pred_locks_per_page;
```

如果看到大量 relation-level SIReadLock，不要马上调参。

先判断它来自：

- 执行计划 seq scan。
- unsupported index AM fallback。
- promotion。
- 空索引 / relation-level race closure。

只有最后一类才可能通过参数减少。

计划和 SQL 形态才是主要因素。

## 16. 常见误区

误区一：

```text
predicate lock 保存的是 SQL WHERE 表达式。
```

不对。

它保存物理 target。

误区二：

```text
tuple-level SIREAD 就足够表达所有读。
```

不对。

空结果和 range gap 需要 index page 或 relation target。

误区三：

```text
relation-level SIReadLock 一定是 bug。
```

不对。

seq scan、fallback、promotion 都可能正确产生 relation target。

误区四：

```text
page target 永远表示 heap page。
```

不对。

relation oid 可以是 heap，也可以是 index。

btree leaf page target 常用于表达 index range。

误区五：

```text
promotion 会破坏 SERIALIZABLE 正确性。
```

不对。

promotion 扩大范围。

它可能增加误 abort，但不会漏掉 conflict。

误区六：

```text
只要有索引，就一定是细粒度 predicate lock。
```

不对。

还要看计划是否走索引，以及 index AM 是否支持 predicate locks。

## 17. 课堂实验

实验一：seq scan 的 relation target。

```sql
DROP TABLE IF EXISTS pred_demo;
CREATE TABLE pred_demo(id int PRIMARY KEY, k int NOT NULL, v text);
INSERT INTO pred_demo SELECT g, g % 100, repeat('x', 20)
FROM generate_series(1, 10000) AS g;
ANALYZE pred_demo;
```

Session A：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SET enable_indexscan = off;
SET enable_bitmapscan = off;
SELECT count(*) FROM pred_demo WHERE k BETWEEN 10 AND 20;
```

Session B：

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

实验二：btree index page target。

```sql
CREATE INDEX pred_demo_k_idx ON pred_demo(k);
ANALYZE pred_demo;
```

Session A：

```sql
ROLLBACK;
BEGIN ISOLATION LEVEL SERIALIZABLE;
SET enable_seqscan = off;
SELECT count(*) FROM pred_demo WHERE k BETWEEN 10 AND 20;
```

Session B 再查 `pg_locks`。

比较 relation/page/tuple 差异。

实验三：range insert。

Session A 保持打开。

Session B：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
INSERT INTO pred_demo VALUES (20001, 15, 'new');
COMMIT;
```

Session A：

```sql
COMMIT;
```

观察是否出现 serialization failure。

实验四：promotion。

降低测试实例的 predicate lock 相关参数需要重启。

如果可以在独立测试实例操作，降低：

```text
max_pred_locks_per_transaction
max_pred_locks_per_relation
max_pred_locks_per_page
```

重复大范围 SERIALIZABLE 读取。

观察 SIReadLock 是否更容易升到 relation level。

实验五：源码断点。

设置断点：

```text
PredicateLockRelation
PredicateLockPage
PredicateLockTID
PredicateLockAcquire
CheckAndPromotePredicateLockRequest
CheckForSerializableConflictIn
```

对照 `EXPLAIN` 的计划节点，记录每个节点实际创建了什么 target。

## 18. 讨论题

1. 为什么 PostgreSQL 不把 SQL predicate expression 放进 lock table？

2. heap seq scan 为什么必须 relation-level，而不是只锁匹配 tuple？

3. btree page target 为什么能近似表达 key range？

4. 空 btree 为什么需要 relation lock 后重新搜索？

5. page split 如果不复制 predicate lock，会漏掉哪类 conflict？

6. promotion 为什么只增加 false positive，不破坏正确性？

7. `ampredlocks` 对 index AM 作者意味着什么责任？

8. `pg_locks` 中 relation/page/tuple 字段如何映射回 `PREDICATELOCKTARGETTAG`？

9. 为什么同一个 SQL 的 serialization failure 率可能随执行计划变化？

10. 对一个高失败率 workload，你会先看计划还是先调 `max_pred_locks_per_transaction`？

## 19. 本节小结

本节的核心结论是：

```text
predicate lock target 是 SQL 谓词在 PostgreSQL runtime 中的物理近似。
```

它不保存 SQL 表达式。

它保存：

- relation。
- page。
- tuple。
- index relation/page。

heap seq scan 通常是 relation-level。

btree range scan 通常靠 leaf page-level。

tuple read 可以产生 tuple-level。

unsupported AM 会退化为 index relation-level。

promotion 会把细粒度 target 合并到更粗 target。

正确性规则很简单：

```text
target 可以保守变粗；
不能漏掉真实读范围。
```

下一节继续讲：

```text
这些 SIREAD lock 为什么不阻塞写，却又必须跨事务结束保留；
内存压力下何时释放、何时合并、何时总结。
```
