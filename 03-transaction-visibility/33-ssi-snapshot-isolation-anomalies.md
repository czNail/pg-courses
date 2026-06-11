# PostgreSQL SSI 与 snapshot isolation 异常

## 课程定位

前置知识：已经理解 `SnapshotData`、事务级 snapshot、tuple visibility、cleanup horizon、HOT / VACUUM 与 long transaction 对旧版本保留的影响。

本节唯一主问题：

```text
为什么一个事务级 snapshot 已经能保证本事务内重复读取稳定，PostgreSQL 的 SERIALIZABLE 仍然不能只停在 snapshot isolation，而必须额外追踪 SSI 状态？
```

核心矛盾：

```text
snapshot isolation 让读不阻塞写、写不阻塞读，读者能得到稳定旧世界；
但两个事务各自基于稳定旧世界做写入时，最终结果可能不等价于任何串行顺序。
```

学完后应能独立判断：

- `REPEATABLE READ` 为什么可以出现 write skew。
- `SERIALIZABLE` 为什么不是简单地把 snapshot 取得更早或保存更久。
- SSI 为什么关注 `rw-conflict`，而不是把所有读写都变成阻塞锁。
- `SERIALIZABLEXACT` 为什么是 snapshot 之外的事务级状态。
- 为什么一个正常提交路径也可能因为串行化失败而报错。
- 为什么重试是 SERIALIZABLE 应用层协议的一部分。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 predicate lock 的完整目标表达。

本节也不展开 dangerous structure 的完整判定。

这些会在后续几节拆开。

这里先把最重要的问题压实：

```text
事务级 snapshot 解决的是单个读者的稳定性；
SSI 解决的是一组并发事务结果是否能排成串行顺序。
```

## 1. 本节在总主线中的位置

前面 32 节已经把 MVCC 的局部世界讲完：

```text
XID 分配
  -> pg_xact 状态
  -> SnapshotData
  -> heap tuple visibility
  -> update chain / HOT
  -> row lock / MultiXact
  -> cleanup horizon / pruning / vacuum / freeze
```

这些机制共同回答一个问题：

```text
当前 snapshot 是否应该看见这个 tuple version？
```

第 33 节开始进入另一个问题：

```text
多个事务都基于各自合法 snapshot 做出的写入，最终组合结果是否合法？
```

这不是 tuple visibility 的局部问题。

一个事务看到的每一行都可能符合 MVCC。

它提交时的每个行级锁冲突也可能都处理正确。

但两个事务合起来仍可能破坏业务约束。

最典型的例子是 write skew。

两个事务读同一组行。

两个事务都看到约束仍成立。

两个事务分别更新不同的行。

单独看，每个事务都没有写写冲突。

一起看，约束被破坏。

PostgreSQL 的 `REPEATABLE READ` 是 snapshot isolation。

它提供事务级稳定 snapshot。

它不尝试保证所有并发事务结果都可串行化。

PostgreSQL 的 `SERIALIZABLE` 在 snapshot isolation 上叠加 SSI。

它仍然让普通读不阻塞普通写。

但它记录读写依赖。

当依赖形状可能形成无法串行化的结果时，它选择回滚其中一个事务。

从本节开始，主线从：

```text
tuple version 对当前 snapshot 是否可见
```

转向：

```text
当前事务的读写依赖是否让整组事务失去串行化解释
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
SERIALIZABLE 事务在获取 snapshot 时创建 SERIALIZABLEXACT；
读路径通过 SIREAD predicate lock 或 MVCC 信息记录“我读过什么旧世界”；
写路径通过 CheckForSerializableConflictIn() / CheckForSerializableConflictOut() 发现读写依赖；
提交路径通过 PreCommit_CheckForSerializationFailure() 阻止 dangerous structure 变成已提交事实。
```

本节的核心矛盾是：

```text
读写互不阻塞带来高并发；
但不阻塞意味着系统必须事后识别哪些并发读写组合不能串行化。
```

PostgreSQL 没有把 SERIALIZABLE 做成严格两阶段锁。

它没有让普通 `SELECT` 阻塞所有可能写入的事务。

它也没有让所有写入等待所有读者结束。

它选择了 SSI：

```text
先允许并发执行；
同时记录会影响串行顺序的 rw-conflict；
只在冲突图出现危险结构时中止事务。
```

这让读路径仍接近 snapshot isolation 的性能模型。

代价是：

- 需要额外 shared memory 追踪 `SERIALIZABLEXACT`。
- 需要 SIREAD lock 表达读过的谓词范围。
- 需要在 heap / index AM 的读写路径上插入 SSI 检查。
- 需要应用层接受 `40001 serialization_failure` 并重试。

本节只建立第一层模型：

```text
稳定 snapshot 不是串行化证明。
```

后续课程会分别讲：

```text
第 34 节：rw-conflict 与 dangerous structure。
第 35 节：predicate lock target 如何表达谓词依赖。
第 36 节：SIREAD lock 为什么不阻塞写但要保留。
第 37 节：索引访问方法如何影响 predicate lock 粒度。
第 38 节：safe snapshot / DEFERRABLE 如何让只读事务避开后续检测成本。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/storage/lmgr/README-SSI` | 先读 SSI 为什么只追踪 `rw-conflict`，以及 write skew 为什么是 snapshot isolation 的核心异常。 |
| 2 | `src/backend/storage/lmgr/predicate.c` | 看 SERIALIZABLE snapshot 获取、SIREAD、rw-conflict、提交检查如何串起来。 |
| 3 | `src/include/storage/predicate_internals.h` | 看 `SERIALIZABLEXACT`、`RWConflictData`、`PREDICATELOCK` 的真实状态边界。 |
| 4 | `src/include/storage/predicate.h` | 看 predicate locking 对 heap / index AM 暴露的公共入口。 |
| 5 | `src/backend/utils/time/snapmgr.c` | 看普通 transaction snapshot 如何进入 serializable snapshot 包装。 |
| 6 | `src/backend/access/heap/heapam.c` | 看 heap 读写路径在哪里调用 SSI 检查。 |
| 7 | `src/backend/access/index/indexam.c` | 看不支持细粒度 predicate lock 的 index AM 为什么退化成 index relation lock。 |
| 8 | `src/backend/access/nbtree/nbtsearch.c` | 看 btree 空索引和 range scan 为什么要补 relation / page predicate lock。 |
| 9 | `src/backend/access/nbtree/nbtinsert.c` | 看 index insert 如何触发 page 级 conflict in。 |
| 10 | `src/backend/utils/adt/lockfuncs.c` | 看 `pg_locks` 如何展示 SIREAD predicate lock，以及 safe snapshot 等待如何观测。 |

推荐阅读顺序：

```text
先读 README-SSI 的异常模型
  -> 读 predicate_internals.h 的 SERIALIZABLEXACT
  -> 读 predicate.c 的 GetSerializableTransactionSnapshot()
  -> 读 heapam.c 的读写调用点
  -> 读 predicate.c 的 conflict in/out
  -> 最后读 PreCommit_CheckForSerializationFailure()
```

不要从 `predicate.c` 顶部一路读到底。

这个文件把 shared memory 初始化、predicate lock、conflict graph、safe snapshot、2PC、SLRU summary 都放在一起。

本节只追一条线：

```text
SERIALIZABLE transaction
  -> snapshot
  -> read footprint
  -> write creates rw-conflict
  -> commit may abort
```

## 4. 一个 runtime 现象先定锚

先用一个简单的值班表。

业务约束是：

```text
每个科室任意时刻至少要有一名医生 on call。
```

建表：

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
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT count(*)
FROM on_call
WHERE dept = 'icu'
  AND on_call;

UPDATE on_call
SET on_call = false
WHERE doctor = 'alice';
```

Session B 同时执行：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;

SELECT count(*)
FROM on_call
WHERE dept = 'icu'
  AND on_call;

UPDATE on_call
SET on_call = false
WHERE doctor = 'bob';
```

两个事务都看到 count = 2。

两个事务都只更新自己的行。

没有更新同一行。

两个事务都可以提交。

最终：

```sql
SELECT *
FROM on_call
WHERE dept = 'icu'
ORDER BY doctor;
```

结果是：

```text
alice false
bob   false
```

业务约束被破坏。

这个结果不能对应任何串行顺序。

如果 A 先串行执行，B 在执行 `SELECT count(*)` 时应该看到只剩一名医生。

如果 B 先串行执行，A 也应该看到只剩一名医生。

但实际并发中，两边都基于旧 snapshot 做判断。

这就是 write skew。

现在把两个事务改成：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
```

同样交错执行。

通常其中一个 `COMMIT` 会报：

```text
ERROR: could not serialize access due to read/write dependencies among transactions
```

SQLSTATE 是 `40001`。

这个错误不是锁等待超时。

不是 deadlock。

不是唯一约束冲突。

它是 SSI 发现这组依赖不能安全提交，于是回滚一个事务。

这个 runtime 现象有三个关键点：

- `REPEATABLE READ` 的事务级 snapshot 很稳定，但仍允许异常。
- `SERIALIZABLE` 不是让两个 `SELECT` 阻塞两个 `UPDATE`。
- `SERIALIZABLE` 通过失败一个事务保护串行化语义。

本节后面所有源码都回到这个现象。

## 5. 为什么事务级 snapshot 不够

事务级 snapshot 的承诺是：

```text
同一个事务内，普通 MVCC 读使用同一个事务 snapshot；
snapshot 创建之后提交的事务，对这个事务不可见；
snapshot 创建时仍 running 的事务，对这个事务不可见；
本事务自己的写入按 command id 和 self visibility 规则处理。
```

这对单个事务是稳定的。

但它没有承诺：

```text
所有并发事务的最终结果都能按某个串行顺序解释。
```

在 write skew 里，两个事务各自的读取都正确。

A 读取时，B 的更新还不可见。

B 读取时，A 的更新还不可见。

两个事务更新的是不同 tuple。

普通 row lock 冲突不会把它们挡住。

MVCC 也不会在 commit 时重新执行 predicate。

这就是 snapshot isolation 的边界。

它保存的是读者视角。

它不保存“这个读者基于哪个谓词做过决策”。

例如：

```sql
SELECT count(*)
FROM on_call
WHERE dept = 'icu'
  AND on_call;
```

这条语句的业务含义是一个谓词范围：

```text
dept = 'icu' AND on_call = true 的所有医生集合。
```

但普通 MVCC snapshot 只会让 executor 判断当前能看到哪些 tuple。

如果另一个事务稍后把某个 tuple 从这个范围移走，普通 snapshot 不会把这件事记录成跨事务依赖。

如果另一个事务插入一个新 tuple 进入这个范围，普通 snapshot 也不会记住“我当时读过这个范围但没看到它”。

所以 `SnapshotData` 的边界是：

```text
xmin / xmax / xip / subxip 能回答 XID 是否在这个 snapshot 看来 running。
```

它不能回答：

```text
哪个并发写入改变了我依赖过的 predicate 结果。
```

SSI 补的正是这个缺口。

## 6. `SERIALIZABLEXACT` 是 snapshot 之外的事务级事实

`SnapshotData` 是 backend-local 的读视角。

`SERIALIZABLEXACT` 是 shared memory 中的 serializable transaction 事实。

它定义在：

```text
src/include/storage/predicate_internals.h
```

本节关心这些字段：

| 字段 | 语义 |
| --- | --- |
| `vxid` | 没有真实 XID 时也能标识一个正在执行的事务。 |
| `topXid` | 当前 serializable 事务真正分配 XID 后的顶层 XID。 |
| `xmin` | 该事务 snapshot 的 `xmin`，参与 SSI cleanup 边界。 |
| `outConflicts` | 本事务读不到某个并发写入时形成的 outgoing rw-conflict。 |
| `inConflicts` | 其它事务读不到本事务写入时形成的 incoming rw-conflict。 |
| `predicateLocks` | 本事务持有的 SIREAD lock 列表。 |
| `possibleUnsafeConflicts` | read-only / deferrable safe snapshot 判断使用的候选冲突。 |
| `prepareSeqNo` / `commitSeqNo` | SSI 用来近似判断提交先后和 dangerous structure。 |
| `SeqNo.lastCommitBeforeSnapshot` | snapshot 创建前最后一个 serializable commit 序号。 |
| `SeqNo.earliestOutConflictCommit` | 提交后保留的 conflict out 摘要。 |
| `flags` | committed、prepared、doomed、read-only、RO-safe 等状态。 |

这些字段组合起来回答：

```text
这个 serializable transaction 在冲突图里处于什么位置？
它是否已经提交？
它是否可能成为 pivot？
它的 predicate locks 还需要保留多久？
它能否作为 read-only safe snapshot 优化退出 SSI 跟踪？
```

`GetSerializableTransactionSnapshot()` 是入口。

它位于：

```text
src/backend/storage/lmgr/predicate.c
```

普通 SERIALIZABLE 事务会进入：

```text
GetSerializableTransactionSnapshot()
  -> GetSerializableTransactionSnapshotInt()
     -> CreatePredXact()
     -> GetSnapshotData()
     -> 初始化 SERIALIZABLEXACT
     -> CreateLocalPredicateLockHash()
```

注意顺序。

先创建 `SERIALIZABLEXACT`。

再拿 `GetSnapshotData()`。

然后把 snapshot 的 `xmin` 记录到 `sxact->xmin`。

这说明 SSI 没有替换 MVCC snapshot。

它是在 MVCC snapshot 旁边建立一套依赖追踪状态。

如果一个 read-only serializable 事务发现当前没有 active writable serializable transaction，它甚至可以不进入完整 predicate tracking。

这也是后面 safe snapshot 的基础。

但 write skew 里的两个事务都是 writable。

它们都会有 `SERIALIZABLEXACT`。

它们都会持有 predicate locks。

它们也都会参与 conflict graph。

## 7. read footprint：SIREAD lock 不是普通锁

`predicate.c` 文件顶部明确说明：

```text
SIREAD locks never block anything.
```

课程里不要把 SIREAD 理解成 `SELECT ... FOR SHARE`。

SIREAD 是一个冲突检测标记。

它的作用是：

```text
如果后来有 concurrent serializable writer 修改了我读过的对象或范围，
writer 可以通过这个标记发现 rw-conflict。
```

在 write skew 里，读路径通常会通过 heap 或 index 记录 footprint。

如果是 seq scan：

```text
heap_beginscan()
  -> PredicateLockRelation(relation, snapshot)
```

`heapam.c` 对 seq scan 和 sample scan 的注释说明：

```text
heap scan 没有更细粒度的 predicate range；
所以 serializable seq scan 要对整个 relation 取 predicate lock。
```

如果是 index scan：

```text
index_beginscan_internal()
  -> 如果 index AM 没有 ampredlocks
     -> PredicateLockRelation(indexRelation, snapshot)
```

btree 支持更细粒度。

btree 扫描会在 leaf page 上：

```text
_bt_readpage()
  -> PredicateLockPage(rel, so->currPos.currPage, scan->xs_snapshot)
```

如果 btree 搜索发现索引还没有 root page，源码会先拿 relation-level predicate lock 再重新搜索。

这是为了关掉一个窄窗口：

```text
读者认为索引空；
writer 在读者加 SIREAD 前插入；
writer 看不到读者的 predicate lock；
读者也没有扫描到 writer 的 tuple。
```

SIREAD lock 的目标可以是：

- heap relation。
- heap page。
- heap tuple。
- index relation。
- index page。

第 35 和第 37 节会细讲目标表达和粒度。

本节只抓住一点：

```text
SERIALIZABLE 读路径必须留下 footprint；
否则后续写入无法知道自己改变了谁曾经读过的旧世界。
```

## 8. write path：写入怎样把异常变成可检测冲突

读路径留下 SIREAD footprint。

写路径负责检查这些 footprint。

heap 写入路径会调用：

```text
CheckForSerializableConflictIn(relation, tid, blkno)
```

在 `heapam.c` 中，典型写入口包括：

- insert。
- delete。
- update。
- relation 级 truncate / rewrite 相关路径。

`CheckForSerializableConflictIn()` 的问题是：

```text
我正在写某个 tuple / page / relation；
有没有别的 serializable transaction 之前读过这个目标或覆盖它的上级目标？
```

源码检查顺序是：

```text
tuple target
  -> page target
  -> relation target
```

这是为了避免 granularity promotion 期间漏掉冲突。

如果找到别的事务的 SIREAD lock：

```text
FlagRWConflict(reader_sxact, writer_sxact)
```

在 write skew 里：

```text
A 读过 bob 或 icu/on_call 范围；
B 更新 bob；
于是形成 A -> B 的 rw-conflict。

B 读过 alice 或 icu/on_call 范围；
A 更新 alice；
于是形成 B -> A 的 rw-conflict。
```

方向很重要。

这里的箭头不是“谁先执行”。

它表示：

```text
reader 读到的世界没有包含 writer 的写入；
所以在可串行化解释中，reader 必须排在 writer 前面。
```

如果 A 必须排在 B 前面，同时 B 必须排在 A 前面，就出现环。

SSI 不一定直接维护完整环。

它维护足以发现异常的 dangerous structure。

下一节会展开这个判断。

本节只需要知道：

```text
write skew 的错误结果不是靠行级写写冲突发现；
它靠读路径 SIREAD footprint 和写路径 conflict in 检查发现。
```

## 9. read path 也会发现 conflict out

写路径可以通过 SIREAD 发现：

```text
别人读过我现在要写的目标。
```

读路径也可以通过 tuple MVCC 信息发现另一类冲突：

```text
我正在读一个 tuple；
这个 tuple 的创建者或删除者是 concurrent serializable transaction；
我没有看到它的效果；
所以我对那个 writer 有 rw-conflict out。
```

入口在 `heapam.c`：

```text
HeapCheckForSerializableConflictOut()
```

它被调用在 heap tuple 读取路径。

它先用：

```text
HeapTupleSatisfiesVacuum(tuple, TransactionXmin, buffer)
```

判断 tuple 的状态。

然后选出可能冲突的 XID：

- tuple 对当前 snapshot 不可见时，可能是 inserter 的 `xmin`。
- tuple 可见但被并发删除或更新时，可能是 updater / deleter 的 XID。

接着：

```text
SubTransGetTopmostTransaction()
CheckForSerializableConflictOut(relation, xid, snapshot)
```

`CheckForSerializableConflictOut()` 会去 `SerializableXidHash` 找这个 XID 对应的 `SERIALIZABLEXACT`。

如果找到，并且两个事务确实并发，就记录：

```text
MySerializableXact -> other_sxact
```

这条路径说明：

```text
SSI 不是只靠 predicate lock 表。
```

早写晚读的情况，读者可以从 heap tuple 的 MVCC 头部推导出冲突。

晚写早读的情况，writer 需要从 SIREAD lock 找到早读者。

两条路径合起来覆盖 read/write dependency。

这也是 PostgreSQL 实现里很容易读偏的地方。

如果只看 `PredicateLockRelation()` / `PredicateLockPage()`，会以为 SSI 只是 lock table。

如果只看 `HeapCheckForSerializableConflictOut()`，会漏掉 phantom / range insert 场景。

正确模型是：

```text
MVCC tuple header 提供一部分已发生写入的证据；
SIREAD predicate lock 提供一部分读过范围的证据；
rw-conflict list 把证据压缩成事务依赖。
```

## 10. 提交路径为什么还会失败

许多工程师第一次遇到 `serialization_failure` 时，会疑惑：

```text
前面的 UPDATE 已经成功；
为什么 COMMIT 才失败？
```

原因是 SSI 不是只在语句执行点做局部判断。

它要判断一组事务依赖是否形成危险结构。

冲突可能在读时记录。

也可能在写时记录。

也可能只有当某个事务准备提交时，提交顺序才足以判定它必须失败。

提交检查入口是：

```text
PreCommit_CheckForSerializationFailure()
```

它在 `predicate.c` 中。

如果当前事务没有 `MySerializableXact`，直接返回。

否则它持有：

```text
SerializableXactHashLock
```

然后检查当前事务是否已经被别人标记为 `DOOMED`。

如果被标记，就报 `serialization_failure`。

接着它遍历：

```text
MySerializableXact->inConflicts
```

寻找会让当前事务成为 dangerous structure 远端或近端的形状。

源码注释说明：

```text
如果当前事务是 conflict out 一侧并要提交；
而 pivot 和 in 侧仍可能形成结构；
则通常标记 pivot 死亡，保证重试能前进。
```

最终，当前事务会设置：

```text
prepareSeqNo = ++PredXact->LastSxactCommitSeqNo
SXACT_FLAG_PREPARED
```

这里的 `PREPARED` 不只服务两阶段提交。

普通提交过程中也会短暂进入 prepared 状态。

SSI 用 `prepareSeqNo` / `commitSeqNo` 近似提交先后。

所以 commit 失败不是补丁式特例。

它是 SSI 运行模型的正常出口之一。

## 11. 生命周期 / ownership / cleanup

`SERIALIZABLEXACT` 的生命周期从 snapshot 开始。

它不一定从事务一开始就存在。

在 PostgreSQL 中，普通事务可以先没有 XID。

SERIALIZABLE 事务也可以在真正获取 snapshot 前还没有 SSI 状态。

创建阶段：

```text
GetSerializableTransactionSnapshotInt()
  -> CreatePredXact()
  -> 初始化 sxact
  -> MySerializableXact = sxact
  -> CreateLocalPredicateLockHash()
```

读路径持有：

```text
PREDICATELOCK
```

这些 lock 挂在两个方向：

- target 的 `predicateLocks` 列表。
- sxact 的 `predicateLocks` 列表。

写路径持有：

```text
RWConflictData
```

这些 conflict 也挂在两个方向：

- reader 的 `outConflicts`。
- writer 的 `inConflicts`。

释放阶段在：

```text
ReleasePredicateLocks(isCommit, isReadOnlySafe)
```

如果事务回滚：

```text
SXACT_FLAG_DOOMED
SXACT_FLAG_ROLLED_BACK
```

它的 predicate locks 可以尽快释放。

因为失败事务不会再参与已提交结果。

如果事务提交：

```text
SXACT_FLAG_COMMITTED
commitSeqNo = ++PredXact->LastSxactCommitSeqNo
```

它的 SIREAD locks 不一定马上释放。

原因是：

```text
一个已提交事务仍可能和仍在运行的重叠事务形成 dangerous structure。
```

这就是 SIREAD lock 与普通 heavyweight lock 最大的区别之一。

普通锁通常在事务结束释放。

SIREAD lock 可能跨过提交点保留。

清理时机由：

```text
PredXact->SxactGlobalXmin
finishedBefore
FinishedSerializableTransactions
ClearOldPredicateLocks()
```

共同决定。

本节先不深入清理算法。

第 36 节会专门讲 SIREAD 生命周期和内存压力。

这里要记住：

```text
SERIALIZABLEXACT 的 owner 是 predicate lock manager；
backend 只通过 MySerializableXact 持有当前事务引用；
提交后对象可能继续在 shared memory 中服务其它事务的冲突判断。
```

## 12. 正确性机制层次

SSI 不是取代 MVCC。

它叠在 MVCC 上。

分层如下：

| 层次 | 解决的问题 | 本节边界 |
| --- | --- | --- |
| MVCC snapshot | 当前事务能看到哪些 tuple version | `SnapshotData`、`HeapTupleSatisfiesMVCC()` |
| row lock / update conflict | 两个事务是否修改同一行或同一 MultiXact 资源 | `heap_update()`、row lock modes |
| SIREAD predicate lock | 某个 serializable 事务读过哪些 tuple / page / relation / index range | `PredicateLockRelation/Page/TID()` |
| rw-conflict graph | 并发读写之间的串行顺序依赖 | `RWConflictData`、`outConflicts`、`inConflicts` |
| dangerous structure 检测 | 依赖组合是否可能形成不可串行化结果 | `FlagRWConflict()`、`PreCommit_CheckForSerializationFailure()` |

这几层的关系不能混淆。

MVCC 可以说：

```text
这个 tuple 对 A 不可见。
```

row lock 可以说：

```text
两个事务同时更新同一行，必须等待或失败。
```

SIREAD 可以说：

```text
A 读过这个范围，B 后来写入这个范围时要记录依赖。
```

rw-conflict 可以说：

```text
A 的读视角把 B 排在自己之后。
```

dangerous structure 可以说：

```text
这些依赖如果都提交，会让结果没有串行顺序。
```

工程诊断时，不要看到 `SIREAD` 就以为是阻塞锁。

也不要看到 `serialization_failure` 就以为某行被同时更新。

write skew 的关键恰恰是：

```text
没有同一行写写冲突；
但有跨谓词的读写依赖。
```

## 13. 错误路径 / 异常路径 / fallback

本节至少要认识四类异常出口。第一类是正常 SSI 失败：

```text
ERROR: could not serialize access due to read/write dependencies among transactions
SQLSTATE 40001
```

它可能来自 conflict out checking、conflict in checking、commit attempt 或 old committed summary conflict。源码里的 internal reason code 能帮助定位，普通客户端通常只需要按 `40001` 重试整个事务。

第二类是 shared memory 不足。SIREAD lock 表和 conflict pool 都在 postmaster 启动时按参数预留。如果无法创建 predicate lock 或 conflict 元素，可能报：

```text
out of shared memory
```

hint 通常指向：

```text
max_pred_locks_per_transaction
```

第三类是 granularity promotion。细粒度 SIREAD lock 太多时，系统会把多个 tuple/page lock 合并成更粗 target。正确性不变，false positive 增加，表现是 `40001` 变多而不是查询被阻塞。

第四类是 old committed summary。为了限制内存，老的 committed serializable transaction 会被总结到 serial SLRU；之后冲突判断可能只知道“存在过危险摘要”，无法保留完整图，因此会保守失败。

这些 fallback 都是 correctness over precision，不是 bug。

## 14. 成本、资源与跨模块传播

SSI 的成本来自四个方向：

```text
读路径：
  SERIALIZABLE read 可能创建 SIREAD。
  seq scan 常见 relation-level。
  btree index scan 常见 leaf page-level。
  heap visible tuple read 可能创建 tuple-level。

写路径：
  heap write 按 tuple -> page -> relation 检查。
  index insert 还要检查 index page target。

shared memory：
  SERIALIZABLEXACT / PREDICATELOCKTARGET / PREDICATELOCK / RWConflictData 都是共享资源。

abort / retry：
  SERIALIZABLE 用 40001 保护正确性，应用必须重试整个事务。
```

跨模块传播如下：

```text
executor 选择 seq scan 或 index scan
  -> heap / index AM 选择 predicate lock 粒度
  -> predicate lock 粒度影响 false positive
  -> false positive 影响 serialization failure 率
  -> retry 影响业务延迟和吞吐
```

所以 SSI 不是只属于 lock manager。

它和 planner、index AM、heap visibility、transaction manager、application retry 都有关。同一业务逻辑在 SERIALIZABLE 下换执行计划，可能改变失败率；原因不是 SQL 语义变了，而是 predicate lock footprint 变了。

## 15. 观测与诊断入口

第一组入口是错误本身。

```text
SQLSTATE 40001
ERROR: could not serialize access due to read/write dependencies among transactions
```

客户端至少记录事务内 SQL、隔离级别、read only / deferrable 属性、失败点和重试结果。

第二组入口是 `pg_locks`。`lockfuncs.c` 会把 predicate lock target 展示成 relation/page/tuple。

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple, granted
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

`SIReadLock granted=true` 不表示 writer 被它阻塞；它表示一个可能用于未来冲突检测的 read footprint。

```sql
SELECT pid, state, xact_start, backend_xmin, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type = 'client backend'
ORDER BY xact_start NULLS LAST;
```

write skew 通常不是长时间互相等待，而是在写入或提交边界失败。源码断点优先放在：

```text
GetSerializableTransactionSnapshot
PredicateLockRelation
PredicateLockPage
PredicateLockTID
CheckForSerializableConflictIn
CheckForSerializableConflictOut
FlagRWConflict
PreCommit_CheckForSerializationFailure
ReleasePredicateLocks
```

调试时先看 `MySerializableXact`、`topXid`、`xmin`、`outConflicts`、`inConflicts`、`flags`，不要一开始展开整个 `predicate.c`。

## 16. 常见误区

- `SERIALIZABLE` 不是所有读都加锁阻塞写；PostgreSQL 的 SSI 保留非阻塞读写模型，SIREAD 是检测标记。
- `REPEATABLE READ` 不是 `SERIALIZABLE`；它能让事务内读稳定，但不能阻止 write skew。
- 唯一索引点查只能减少一部分 footprint，不能消除所有跨行约束依赖。
- `serialization_failure` 是 SERIALIZABLE 的预期出口，不是数据库损坏信号。
- `pg_locks` 里的 `SIReadLock` 不能用来判断谁阻塞谁，它只能说明 read footprint。
- read-only 事务仍可能参与 dangerous structure，只是后续有 safe snapshot 优化。

## 17. 课堂实验

实验一：比较 `REPEATABLE READ` 与 `SERIALIZABLE` 的 write skew。

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

第一轮两个会话都使用：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM on_call WHERE dept = 'icu' AND on_call;
```

然后 A 更新 `alice`，B 更新 `bob`，两边提交，观察约束被破坏。

第二轮重置数据，两个会话都使用：

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE dept = 'icu' AND on_call;
```

按同样交错更新和提交，观察其中一个事务失败。

实验二：观察 SIREAD。

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT count(*) FROM on_call WHERE dept = 'icu' AND on_call;
```

```sql
SELECT pid, locktype, mode, relation::regclass, page, tuple, granted
FROM pg_locks
WHERE mode = 'SIReadLock'
ORDER BY pid, locktype, relation, page, tuple;
```

比较有索引和无索引时的 locktype / page / tuple。

实验三：观察失败点。把两个会话的 `UPDATE` 和 `COMMIT` 分开执行，记录失败发生在第二个 `UPDATE`、第一个 `COMMIT` 还是第二个 `COMMIT`，再映射回：

```text
CheckForSerializableConflictIn()
CheckForSerializableConflictOut()
PreCommit_CheckForSerializationFailure()
```

实验四：增加重试。客户端必须重试整个事务：

```text
begin
  run transaction
commit
if SQLSTATE == 40001
  rollback if needed
  retry whole transaction with backoff
```

只重试最后一条语句不够，因为事务内读到的 snapshot 和依赖判断都必须重建。

## 18. 讨论题

1. 为什么 write skew 不能靠行级锁自动发现？
2. 如果把 `SELECT count(*) ...` 改成 `SELECT ... FOR UPDATE`，结果和成本分别会怎样变化？
3. 为什么 SIREAD lock 需要覆盖“没有读到的范围”，而不仅是读到的 tuple？
4. 为什么 PostgreSQL 选择 abort/retry，而不是让所有 serializable 读阻塞所有可能写？
5. `REPEATABLE READ` 能保证哪些稳定性？不能保证哪些全局性质？
6. 如果一个 workload 在 SERIALIZABLE 下失败率很高，应该先看 SQL 逻辑、索引计划、事务长度，还是先调大 predicate lock 参数？
7. 只读事务为什么仍可能需要 SSI 状态？
8. 为什么 `SERIALIZABLEXACT` 可能在事务提交后仍然存在？
9. 应用层如果只重试失败语句，不重试整个事务，会留下什么错误假设？
10. 为什么说 SSI 是 `runtime -> reusable abstraction` 的典型例子？

## 19. 本节小结

本节只建立一个核心判断：

```text
事务级 snapshot 解决单个事务读视角稳定；
SERIALIZABLE 还必须解决多个并发事务结果能否串行解释。
```

write skew 说明了差异。

两个事务都读到合法 snapshot，两个事务都更新不同 tuple，普通写写冲突不会出现，结果仍可能不可串行化。

PostgreSQL 的答案是 SSI，它在 snapshot isolation 上增加：

- `SERIALIZABLEXACT`。
- SIREAD predicate locks。
- `rw-conflict` 列表。
- dangerous structure 检测。
- commit-time serialization failure。

`SERIALIZABLE` 的失败不是异常 bug，而是正确性机制的出口；能正确使用 SERIALIZABLE 的系统，必须把 `40001` 作为事务级重试信号。

下一节继续拆开：

```text
rw-conflict 为什么足够重要；
dangerous structure 为什么是 SSI 的核心判定形状；
PreCommit_CheckForSerializationFailure() 到底在保护什么。
```
