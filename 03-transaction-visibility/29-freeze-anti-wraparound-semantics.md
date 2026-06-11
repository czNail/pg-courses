# PostgreSQL freeze 的语义与 anti-wraparound

## 课程定位

前置知识：已经理解 heap tuple 的 `xmin` / `xmax` 是事务身份引用，也理解 VACUUM 使用 `OldestXmin` 判断 dead tuple 是否可回收。

本节唯一主问题：

```text
为什么 PostgreSQL 需要把很老的 tuple XID freeze 掉，而不是永远保留原始 xmin/xmax 并在需要时查询 pg_xact？
```

核心矛盾：MVCC 需要长期解释旧 tuple 的事务结果，但 32-bit TransactionId 会循环使用，pg_xact 也不能无限保留所有历史事务状态；如果旧 tuple 永远依赖原始 XID，系统最终无法安全区分“很久以前提交”和“未来/新一轮 XID”。

学完后应能判断：freeze 不是让 tuple “变新”，也不是 VACUUM 删除 dead tuple 的同义词；freeze 的核心是移除旧 XID 对事务结果历史的依赖，并允许 `relfrozenxid` / `relminmxid` 向前推进。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 26 节和第 28 节主要讨论空间回收。

它们回答：

```text
旧版本什么时候可以被删除？
索引 TID 什么时候可以清理？
heap slot 什么时候可以复用？
```

freeze 的问题不同。

一个 tuple 可以是 live。

它不需要删除。

但它的 `xmin` 可能非常老。

如果这个 XID 继续留在 tuple header 里，未来系统还要能解释它。

问题是 PostgreSQL 的 TransactionId 是有限循环空间。

pg_xact 不能保留无限历史。

因此 VACUUM 还有第二个目标：

```text
不是释放空间；
而是把仍然 live 的旧 tuple 从古老 XID 依赖中解放出来。
```

这就是 freeze。

本节只讲 XID / MultiXact freeze 的语义和 anti-wraparound 主线。

第 31 节会继续讲 all-frozen visibility map bit 如何记录页面不再含有需要 freeze 的 tuple。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
vacuum_get_cutoffs() 用 nextXID、OldestXmin、freeze age 和 relfrozenxid 计算 FreezeLimit；
lazy VACUUM 扫描页面时由 heap_prepare_freeze_tuple() 准备 tuple-level freeze plan；
heap_page_prune_and_freeze() 在安全时调用 heap_freeze_prepared_tuples() 应用计划；
VACUUM 最后用扫描结果推进 relfrozenxid / relminmxid，降低 wraparound 风险。
```

本节核心矛盾是：

```text
tuple header 中保留原始事务身份有助于解释历史
  vs
事务 ID 空间和 pg_xact 历史都不能无限增长
```

PostgreSQL 的选择是：

```text
在还能证明旧事务结果安全的时间窗口内，把老 XID 语义压缩成 frozen 状态。
```

freeze 不是 visibility check 的 shortcut。

它是长期存储格式维护。

它把“这个 tuple 的插入事务曾经是某个很老的 XID 并且已经提交”压缩成“这个 tuple 的插入对所有未来正常 snapshot 都是已完成可解释的”。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/commands/vacuum.c` | 阅读 `vacuum_get_cutoffs()`，理解 `OldestXmin`、`FreezeLimit`、`MultiXactCutoff` 和 aggressive VACUUM。 |
| 2 | `src/include/commands/vacuum.h` | 对照 `VacuumCutoffs`、freeze 参数、failsafe 参数。 |
| 3 | `src/backend/access/heap/vacuumlazy.c` | 阅读 `heap_vacuum_rel()` 如何初始化 `NewRelfrozenXid` / `NewRelminMxid`，以及 eager freeze、aggressive、failsafe。 |
| 4 | `src/backend/access/heap/pruneheap.c` | 阅读 `heap_page_prune_and_freeze()`、`heap_page_will_freeze()`、freeze plan 与 WAL。 |
| 5 | `src/backend/access/heap/heapam.c` | 阅读 `heap_prepare_freeze_tuple()`、`heap_freeze_prepared_tuples()`、`heap_tuple_should_freeze()`。 |
| 6 | `src/include/access/htup_details.h` | 对照 tuple header、infomask、HOT bit、frozen 相关标志。 |
| 7 | `src/backend/storage/ipc/procarray.c` | 对照 cleanup horizon 与 freeze cutoff 为什么不能超过 OldestXmin。 |
| 8 | `src/backend/access/transam/multixact.c` | MultiXact cutoff、old members 和 relminmxid 的背景。 |

本节阅读重点不是背 freeze 函数。

要追问：

```text
这个 tuple 是否仍需要原始 XID 才能解释？
这个 relation 是否能安全推进 relfrozenxid？
VACUUM 跳过页面是否会漏掉未冻结 XID？
```

## 4. 一个 runtime 现象先定锚

先观察 relfrozenxid。

```sql
DROP TABLE IF EXISTS freeze_demo;
CREATE TABLE freeze_demo(id int primary key, payload text);

INSERT INTO freeze_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 10000) AS g;

SELECT c.relname,
       c.relfrozenxid,
       age(c.relfrozenxid) AS relfrozen_age
FROM pg_class c
WHERE c.relname = 'freeze_demo';
```

手工执行：

```sql
VACUUM (FREEZE, VERBOSE) freeze_demo;
```

再观察：

```sql
SELECT c.relname,
       c.relfrozenxid,
       age(c.relfrozenxid) AS relfrozen_age
FROM pg_class c
WHERE c.relname = 'freeze_demo';
```

现象是：VACUUM FREEZE 可能推进 `relfrozenxid`，并在 verbose 中报告 frozen tuple / new relfrozenxid。

这不是因为旧 tuple 被删除。

表里数据仍然存在。

变化的是 tuple 对旧 XID 的依赖。

这个实验的关键解释是：

```text
空间回收关注 dead tuple；
freeze 关注 live tuple 中是否还含有过老 XID。
```

如果你只看行数，会以为 VACUUM 什么也没做。

如果你看 `relfrozenxid`、verbose 和 VM all-frozen 计数，才会看到 freeze 的效果。

## 5. `VacuumCutoffs` 建立 freeze 边界

`vacuum_get_cutoffs()` 是本节起点。

它输出 `VacuumCutoffs`。

关键字段可以分成两组。

第一组是 removable / visibility 相关：

```text
OldestXmin
OldestMxact
```

第二组是 freeze 相关：

```text
FreezeLimit
MultiXactCutoff
relfrozenxid
relminmxid
```

`OldestXmin` 来自 `GetOldestNonRemovableTransactionId(rel)`。

它保护仍可能需要旧版本的观察者。

`FreezeLimit` 来自 `nextXID - freeze_min_age`，再被限制不能超过 `OldestXmin`。

这个限制非常关键。

```text
FreezeLimit <= OldestXmin
```

因为 freeze 也不能破坏仍可能运行的事务对 tuple 的解释。

如果某个 XID 还可能被旧 snapshot 视为 running，就不能把相关 tuple 的事务身份提前压缩掉。

`relfrozenxid` 是 pg_class 中 relation 级承诺。

它表达：

```text
这张表中没有比 relfrozenxid 更老的未冻结普通 XID 需要保留。
```

`relminmxid` 对 MultiXact 起类似作用。

aggressive VACUUM 的判断也在这里。

如果 relation 的 `relfrozenxid` 太旧，达到 table freeze age，VACUUM 必须更积极扫描页面，确保至少能推进到 `FreezeLimit`。

## 6. freeze 与 dead tuple cleanup 是两条线

dead tuple cleanup 的问题是：

```text
这个 tuple version 是否还可能被任何观察者需要？
```

freeze 的问题是：

```text
这个仍存在的 tuple 是否还需要一个很老的 XID 才能解释？
```

这两条线经常同时发生。

`heap_page_prune_and_freeze()` 同时负责 pruning 和 freeze。

但它们的目标不同。

一个 dead tuple 可能被删除。

删除后不再需要 freeze 它的 `xmin`。

一个 live tuple 不会被删除。

但它可能需要 freeze。

一个 recently dead tuple 不能删除。

它是否能 freeze 要看 tuple 事务字段和正确性规则。

源码因此准备一个 page-level plan。

```text
pruning plan:
  redirected
  nowdead
  nowunused

freeze plan:
  frozen tuples
  new relfrozenxid
  new relminmxid
```

最后在同一个 critical section 中应用。

这减少 WAL 和 page rewrite 成本。

但概念上仍是两条线。

## 7. 主流程源码 walkthrough

主流程从 VACUUM 开始。

```text
heap_vacuum_rel()
  -> vacuum_get_cutoffs()
  -> vacrel->NewRelfrozenXid = OldestXmin
  -> vacrel->NewRelminMxid = OldestMxact
  -> lazy_scan_heap()
     -> lazy_scan_prune()
        -> heap_page_prune_and_freeze()
           -> prune_freeze_setup()
           -> prune_freeze_plan()
              -> heap_prepare_freeze_tuple()
           -> heap_page_will_freeze()
           -> heap_freeze_prepared_tuples()
           -> log_heap_prune_and_freeze()
  -> vac_update_relstats(... NewRelfrozenXid, NewRelminMxid ...)
```

第一步，`vacuum_get_cutoffs()` 获取 freeze 边界。

它会检查：

```text
freeze_min_age
freeze_table_age
autovacuum_freeze_max_age
multixact_freeze_min_age
multixact_freeze_table_age
```

并计算：

```text
FreezeLimit
MultiXactCutoff
aggressive or normal
```

第二步，`heap_vacuum_rel()` 初始化 relation 级追踪值。

初始：

```text
NewRelfrozenXid = OldestXmin
NewRelminMxid = OldestMxact
```

扫描过程中，如果遇到仍未冻结且更老的 XID，需要把这些值往回保守调整。

第三步，`lazy_scan_heap()` 按页面扫描。

如果页面 all-frozen，普通 VACUUM 可以跳过。

如果 aggressive VACUUM 或禁用 page skipping，则需要更完整扫描。

第四步，`lazy_scan_prune()` 调用 `heap_page_prune_and_freeze()`，并传入：

```text
HEAP_PAGE_PRUNE_FREEZE
HEAP_PAGE_PRUNE_SET_VM
cutoffs
vistest
```

第五步，`prune_freeze_plan()` 扫描页内 tuple。

对于仍有 storage 的 tuple，会检查是否需要 freeze。

需要 freeze 的 tuple 会产生 `HeapTupleFreeze` plan。

第六步，`heap_page_will_freeze()` 决定是否执行 freeze plan。

如果必须 freeze 才能推进 relfrozenxid / relminmxid，它必须执行。

如果不是必须，也可能因为已经要写 WAL 或有 FPI 成本，选择 opportunistic freeze。

第七步，执行阶段调用：

```text
heap_freeze_prepared_tuples()
```

它把准备好的 tuple header 修改应用到 page。

第八步，WAL 记录通过：

```text
log_heap_prune_and_freeze()
```

记录 pruning、freeze、VM bit 和 conflict horizon。

第九步，VACUUM 结束时更新 pg_class。

`vac_update_relstats()` 会写入新的 `relfrozenxid` / `relminmxid`。

如果某些 all-visible page 被跳过，非 aggressive VACUUM 可能不能推进 relation-level freeze horizon。

因为它没有检查那些页面是否仍含旧 XID。

## 8. 生命周期 / ownership / cleanup

一个 XID 在 tuple 中的生命周期可以这样看。

阶段一：插入事务写入 `xmin`。

tuple 需要 `xmin` 判断插入者是否提交。

阶段二：事务提交。

hint bit 可能缓存提交事实。

但 tuple header 仍保留原始 XID。

阶段三：tuple 长期 live。

业务行可能几年不更新。

tuple 仍然携带旧 XID。

阶段四：XID 逐渐变老。

`age(relfrozenxid)` 增长。

系统需要在 wraparound 风险出现前处理。

阶段五：VACUUM 扫描页面。

如果 tuple 中的 XID 早于 `FreezeLimit`，并满足安全条件，生成 freeze plan。

阶段六：freeze 应用。

tuple header 被修改，不再依赖那个旧 XID 的普通历史状态。

阶段七：relation 级 horizon 推进。

如果 VACUUM 能证明 relation 中没有更老未冻结 XID，`relfrozenxid` 向前。

阶段八：旧 pg_xact / multixact 历史可以最终被截断或复用。

这条生命周期强调：

```text
freeze 的对象是 tuple 中的事务身份依赖；
不是 tuple 行本身。
```

ownership 也要分层。

tuple header 属于 heap page。

freeze plan 属于一次 pruning/freezing 调用。

`NewRelfrozenXid` 属于一次 VACUUM relation 运行态。

`relfrozenxid` 属于 pg_class 持久元数据。

pg_xact 截断属于事务日志存储管理。

## 9. 正确性机制层次

第一层是 XID 顺序安全。

`FreezeLimit` 不能超过 `OldestXmin`。

否则可能把仍被旧 snapshot 需要解释的事务身份过早压缩。

第二层是 tuple-level eligibility。

不是所有字段都能随便冻结。

`xmin`、`xmax`、MultiXact、锁和更新语义要分别处理。

第三层是 relation-level completeness。

要推进 `relfrozenxid`，VACUUM 必须知道没有更老未冻结 XID 留在未扫描页面。

跳过 all-visible page 会影响这个证明。

第四层是 aggressive VACUUM。

当 relation 足够老，VACUUM 不能只做 opportunistic cleanup。

它必须扫描足够页面来推进 freeze horizon。

第五层是 WAL。

freeze 修改 tuple header。

它是持久页面修改，必须可 redo。

第六层是 standby conflict horizon。

pruning / freeze WAL 可能携带 conflict xid。

standby recovery 需要用它决定是否和 standby query 冲突。

第七层是 failsafe。

当 wraparound 危险逼近，系统会放弃非必要维护，优先保证 freeze 进展。

这解释了为什么 failsafe 会跳过 index cleanup。

它不是为了减少 bloat。

它是为了保命。

## 10. 错误路径 / 异常路径 / fallback

### `OldestXmin` 被钉住

长事务、prepared transaction、replication slot 或 standby feedback 可能让 `OldestXmin` 很老。

`vacuum_get_cutoffs()` 会发出 warning：

```text
cutoff for removing and freezing tuples is far in the past
```

这种情况下，VACUUM 想 freeze 也受限。

因为安全边界没有前进。

### all-visible page skipping

普通 VACUUM 可能跳过 all-visible page。

这可以节省 I/O。

但如果跳过了可能含有未冻结 XID 的页面，就不能安全推进 `relfrozenxid`。

源码用 `skippedallvis` 影响 final relfrozenxid 更新。

### VACUUM FREEZE

`VACUUM (FREEZE)` 会更积极选择 freeze。

但它仍不意味着可以忽略所有正确性边界。

它不能让未安全的事务身份被过早压缩。

### MultiXact cutoff

`MultiXactCutoff` 和 `relminmxid` 是 parallel problem。

共享锁、行锁和 MultiXact members 也有 wraparound 风险。

如果只关注 XID 而忽略 MultiXact，表仍可能触发 aggressive autovacuum。

### cleanup lock 拿不到

需要 freeze 的 tuple 可能要求 cleanup lock。

如果普通路径拿不到，非 aggressive VACUUM 可能稍后再处理。

aggressive VACUUM 的容忍度低得多。

### failsafe

failsafe 触发后，VACUUM 关闭 cost delay，并绕过 index cleanup / truncation 等非必要工作。

这会让本次 VACUUM 更专注于 freeze 进展。

## 11. 成本、资源与跨模块传播

freeze 的成本不是只有 CPU。

它会修改 heap page。

它会产生 WAL。

它可能导致 full-page image。

它可能让 VACUUM 扫描原本可以跳过的 all-visible pages。

它可能和 pruning 合并，摊薄一次页面修改成本。

主要成本维度：

| 成本来源 | 影响 |
| --- | --- |
| relation size | aggressive VACUUM 需要更广泛扫描。 |
| all-visible but not all-frozen pages | 普通 VACUUM 可能 eager freeze 一部分。 |
| tuple age 分布 | 很多老 tuple 分散在全表，会增加扫描面。 |
| WAL / FPI | freeze 是持久页面修改，需要记录。 |
| cleanup lock | 页面被 pin 或并发访问时 freeze 可能推迟。 |
| replication / standby | conflict horizon 可能导致 standby query conflict。 |
| autovacuum 参数 | freeze_min_age、freeze_table_age、failsafe_age 影响触发时机。 |

跨模块路径：

```text
commands/vacuum.c:
  计算 freeze cutoffs

vacuumlazy.c:
  决定 normal / aggressive / failsafe，追踪 NewRelfrozenXid

pruneheap.c:
  page-level pruning + freezing plan

heapam.c:
  tuple-level freeze preparation and application

visibility map:
  all-frozen bit 记录页面不再含需 freeze 的 tuple

pg_class:
  relfrozenxid / relminmxid 持久化 relation 级证明
```

## 12. 观测与诊断入口

看 relation freeze age：

```sql
SELECT c.oid::regclass AS rel,
       c.relfrozenxid,
       age(c.relfrozenxid) AS xid_age,
       c.relminmxid,
       mxid_age(c.relminmxid) AS mxid_age
FROM pg_class c
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;
```

看 VACUUM verbose：

```sql
VACUUM (VERBOSE) freeze_demo;
```

关注：

```text
new relfrozenxid
new relminmxid
frozen pages
tuples frozen
aggressively vacuuming
automatic aggressive vacuum to prevent wraparound
index scan bypassed by failsafe
```

看 progress mode：

```sql
SELECT pid, phase, mode, heap_blks_scanned, heap_blks_total
FROM pg_stat_progress_vacuum;
```

`mode` 需要对照 `progress.h`。

```text
normal
aggressive
failsafe
```

看阻塞 horizon 的对象：

```sql
SELECT pid, backend_xmin, xact_start, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

看 prepared transaction：

```sql
SELECT gid, prepared, owner, database, transaction
FROM pg_prepared_xacts
ORDER BY prepared;
```

看 replication slot：

```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin,
       restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

源码断点建议：

```text
vacuum_get_cutoffs
heap_vacuum_rel
lazy_scan_prune
heap_page_prune_and_freeze
heap_prepare_freeze_tuple
heap_freeze_prepared_tuples
vac_update_relstats
lazy_check_wraparound_failsafe
```

## 13. 常见误区

误区一：freeze 会删除旧行。

正确理解：freeze 通常作用在仍然 live 的 tuple 上。

它移除旧 XID 依赖，不删除行。

误区二：freeze 只是设置 hint bit。

正确理解：hint bit 缓存事务结果。

freeze 改变 tuple header 中长期事务身份表达。

误区三：`VACUUM FREEZE` 可以无视长事务。

正确理解：freeze cutoff 仍受 `OldestXmin` 约束。

长事务会限制 freeze 边界。

误区四：只看 `n_dead_tup` 就能判断 anti-wraparound 风险。

正确理解：anti-wraparound 主要看 `age(relfrozenxid)` 和 `mxid_age(relminmxid)`。

一张没有 dead tuple 的表也可能需要 freeze。

误区五：all-visible 等于 all-frozen。

正确理解：all-visible 说明所有 tuple 对所有事务可见。

all-frozen 说明页面上不再含需要 freeze 的 tuple。

第 30、31 节会拆开。

误区六：failsafe 是更强的 VACUUM。

正确理解：failsafe 会跳过非必要维护。

它是 emergency mode。

误区七：`relfrozenxid` 推进表示所有 tuple 都被重写。

正确理解：它表示 relation 级证明已更新。

具体页面可能依赖 VM all-frozen、扫描结果和历史 freeze 状态。

## 14. 课堂实验

### 实验一：观察 VACUUM FREEZE 对 relfrozenxid 的影响

```sql
DROP TABLE IF EXISTS freeze_demo;
CREATE TABLE freeze_demo(id int primary key, payload text);

INSERT INTO freeze_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 10000) AS g;

SELECT relfrozenxid, age(relfrozenxid)
FROM pg_class
WHERE oid = 'freeze_demo'::regclass;

VACUUM (FREEZE, VERBOSE) freeze_demo;

SELECT relfrozenxid, age(relfrozenxid)
FROM pg_class
WHERE oid = 'freeze_demo'::regclass;
```

解释：

```text
行仍在。
旧 XID 依赖减少。
relfrozenxid 可能前进。
```

### 实验二：区别 dead cleanup 与 freeze

```sql
DROP TABLE IF EXISTS freeze_live_demo;
CREATE TABLE freeze_live_demo(id int primary key, payload text);

INSERT INTO freeze_live_demo
SELECT g, repeat('y', 100)
FROM generate_series(1, 10000) AS g;

VACUUM (FREEZE, VERBOSE) freeze_live_demo;

SELECT count(*) FROM freeze_live_demo;
```

观察：数据还在。

再对比：

```sql
DELETE FROM freeze_live_demo WHERE id <= 5000;
VACUUM (VERBOSE) freeze_live_demo;
```

解释：

```text
DELETE + VACUUM 关注空间回收。
VACUUM FREEZE 关注旧 XID 依赖。
两者可能同时发生，但不是同一个动作。
```

### 实验三：查看最老 relation

```sql
SELECT oid::regclass AS rel,
       age(relfrozenxid) AS xid_age,
       mxid_age(relminmxid) AS mxid_age
FROM pg_class
WHERE relkind IN ('r', 'm', 't')
ORDER BY xid_age DESC
LIMIT 10;
```

讨论哪些表应该优先关注。

### 实验四：源码断点

```gdb
break vacuum_get_cutoffs
break heap_prepare_freeze_tuple
break heap_freeze_prepared_tuples
break vac_update_relstats
```

执行：

```sql
VACUUM (FREEZE) freeze_demo;
```

观察：

```gdb
print cutoffs->OldestXmin
print cutoffs->FreezeLimit
print vacrel->NewRelfrozenXid
print vacrel->aggressive
```

把 freeze cutoff 和最终 relfrozenxid 对应起来。

## 15. 讨论题

1. 为什么 freeze 不能简单等价为“tuple 已提交”？

2. 如果一张表几乎没有 dead tuple，为什么仍可能触发 anti-wraparound VACUUM？

3. `FreezeLimit <= OldestXmin` 这个不变量保护什么？

4. 为什么普通 VACUUM 跳过 all-visible page 可能影响 `relfrozenxid` 推进？

5. failsafe 为什么会跳过 index cleanup？

6. MultiXact wraparound 风险为什么不能只靠 XID freeze 解决？

7. all-visible 和 all-frozen 分别服务什么成本模型？

## 16. 本节小结

本节把 VACUUM 的第二个目标讲清楚了。

空间回收处理 dead tuple。

freeze 处理 live tuple 中过老的事务身份依赖。

`vacuum_get_cutoffs()` 建立 freeze 安全边界。

`heap_page_prune_and_freeze()` 在 page-level 合并 pruning 和 freeze。

`heap_prepare_freeze_tuple()` 生成 tuple-level freeze plan。

`heap_freeze_prepared_tuples()` 应用持久修改。

`vac_update_relstats()` 推进 relation 级 `relfrozenxid` / `relminmxid`。

本节可迁移规律是：

```text
长期存储不能永远依赖有限编号空间中的历史身份；
系统必须在仍能证明语义安全时，把历史身份压缩成稳定状态。
```

下一节开始讲 visibility map。先看 all-visible bit：为什么它能让 index-only scan 和 VACUUM page skipping 更便宜，又为什么设置和清除都必须非常谨慎。
