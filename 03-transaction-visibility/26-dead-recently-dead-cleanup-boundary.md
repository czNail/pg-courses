# PostgreSQL dead tuple、recently dead 与可回收边界

## 课程定位

前置知识：已经理解 MVCC snapshot、heap tuple header 的 `xmin` / `xmax`、CLOG/pg_xact 事务结果，以及 cleanup horizon 为什么不能等同于当前查询的可见性。

本节唯一主问题：

```text
一个已经被提交事务删除的旧 tuple version，什么时候只是 recently dead，什么时候才真正 dead 并允许被物理回收？
```

核心矛盾：空间回收希望尽快删除旧版本，但系统必须证明没有任何仍受保护的观察者可能需要这个版本；如果证明不出来，VACUUM 和 pruning 都只能保守地留下它。

学完后应能判断：`DELETE` 已提交、当前查询不可见、`HEAPTUPLE_RECENTLY_DEAD`、`HEAPTUPLE_DEAD`、`LP_DEAD`、`LP_UNUSED` 这些状态分别回答什么问题，为什么它们不能混用。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面课程已经建立了三层判断。

第一层是事务结果。

```text
xmin / xmax 指向哪个事务
  -> pg_xact / hint bit 告诉我们这个事务是否 commit
```

第二层是当前 snapshot 可见性。

```text
HeapTupleSatisfiesMVCC()
  -> 回答这个 tuple version 是否应该被当前扫描返回
```

第三层是 cleanup horizon。

```text
GetOldestNonRemovableTransactionId()
  -> 回答系统里是否还可能有人需要旧版本
```

本节把这三层落到一个具体状态机。

一个旧 tuple version 先被 UPDATE 或 DELETE 产生。

删除事务提交后，它对新 snapshot 可能已经不可见。

但它未必立刻可回收。

VACUUM 要问的问题更保守：

```text
有没有任何仍可能存活的 snapshot、prepared transaction、slot 或 standby horizon 需要它？
```

如果答案不能确定为否，它就是 recently dead。

如果答案确定为否，它才是 dead。

后续第 27 节会继续追问：当一个页上出现 dead tuple，heap page pruning 如何把 HOT chain 缩短。

第 28 节会继续追问：VACUUM 为什么必须把 heap scan、index cleanup 和 heap cleanup 分成多个阶段。

所以本节只讲一个边界：

```text
tuple-level vacuum visibility classification
```

不要把本节误读成完整 VACUUM 课程。

完整 VACUUM 还包括索引 TID 清理、visibility map、freeze、FSM、truncate、parallel vacuum 和 autovacuum 调度。

这些都不是本节唯一主问题。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
HeapTupleSatisfiesVacuumHorizon() 先根据 xmin/xmax 的事务结果判断 tuple 的 vacuum 视角状态；
如果删除事务已提交但仍可能被某些观察者需要，它返回 HEAPTUPLE_RECENTLY_DEAD 和 dead_after；
HeapTupleSatisfiesVacuum() 再用 OldestXmin 比较 dead_after，只有 dead_after < OldestXmin 时才升级为 HEAPTUPLE_DEAD。
```

本节的核心矛盾是：

```text
局部可见性已经证明当前读者不需要旧版本
  vs
物理回收必须证明所有受保护读者都不需要旧版本
```

这导致 PostgreSQL 把“不可见”和“可回收”拆开。

`HeapTupleSatisfiesMVCC()` 不负责释放空间。

它返回“当前 snapshot 是否看见”。

`HeapTupleSatisfiesVacuum()` 不负责返回行。

它返回“VACUUM 是否可以把 tuple 视为 removable”。

两个函数会读取同一批 tuple header 字段。

但它们的问题不同。

因此它们的结论也不能互相替代。

本节需要牢记一条主线：

```text
DELETE committed
  -> tuple 对新 snapshot 不可见
  -> deletion xid 仍可能 >= OldestXmin
  -> HEAPTUPLE_RECENTLY_DEAD
  -> horizon 前移
  -> deletion xid < OldestXmin
  -> HEAPTUPLE_DEAD
  -> page pruning / VACUUM 才能进一步改 line pointer
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/heapam_visibility.c` | 本节主文件。阅读 `HeapTupleSatisfiesVacuum()`、`HeapTupleSatisfiesVacuumHorizon()`、`HeapTupleIsSurelyDead()`。 |
| 2 | `src/include/access/htup_details.h` | 对照 tuple header、`t_infomask`、`t_infomask2`、`t_ctid`、hint bit 与 HOT 标记。 |
| 3 | `src/include/access/heapam.h` | 对照 `HTSV_Result` 枚举语义。 |
| 4 | `src/backend/commands/vacuum.c` | 阅读 `vacuum_get_cutoffs()` 如何生成 `OldestXmin`、`FreezeLimit`、`OldestMxact`。 |
| 5 | `src/backend/storage/ipc/procarray.c` | 阅读 `GetOldestNonRemovableTransactionId()` 与 `ComputeXidHorizons()`。 |
| 6 | `src/backend/access/heap/pruneheap.c` | 看 `heap_prune_satisfies_vacuum()` 如何消费 `RECENTLY_DEAD` 与 `GlobalVisState`。 |
| 7 | `src/backend/access/heap/vacuumlazy.c` | 看 `lazy_scan_prune()` 和 `lazy_scan_noprune()` 如何统计 live、recently dead、dead。 |
| 8 | `src/include/storage/itemid.h` | 区分 tuple status 和 line pointer status：`LP_NORMAL`、`LP_DEAD`、`LP_UNUSED`。 |
| 9 | `src/include/commands/vacuum.h` | 对照 `VacuumCutoffs` 字段语义，避免把 `OldestXmin` 当成 snapshot xmin。 |

阅读顺序不要从 VACUUM 总流程开始。

先读 `HeapTupleSatisfiesVacuumHorizon()`。

再读 `OldestXmin` 从哪里来。

最后读 pruning 和 VACUUM 如何使用这个结果。

本节源码锚点：

```text
/home/highgo/postgres/src/backend/access/heap/heapam_visibility.c
/home/highgo/postgres/src/backend/access/heap/pruneheap.c
/home/highgo/postgres/src/backend/access/heap/vacuumlazy.c
/home/highgo/postgres/src/backend/commands/vacuum.c
/home/highgo/postgres/src/backend/storage/ipc/procarray.c
```

## 4. 一个 runtime 现象先定锚

先构造一个最小现象。

Session A：

```sql
DROP TABLE IF EXISTS htsv_demo;
CREATE TABLE htsv_demo(id int primary key, payload text);
INSERT INTO htsv_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 50000) AS g;

BEGIN;
SELECT count(*) FROM htsv_demo;
-- 保持事务打开。
```

Session B：

```sql
DELETE FROM htsv_demo WHERE id <= 30000;
VACUUM (VERBOSE) htsv_demo;
```

你会看到几个并存的事实。

第一，Session B 中新的查询已经看不到被删掉的行。

第二，`DELETE` 事务已经提交。

第三，`VACUUM VERBOSE` 可能报告仍有 dead but not yet removable 的 tuple。

第四，`pg_stat_activity.backend_xmin` 中能看到 Session A 把 horizon 钉住。

这说明：

```text
删除事务提交
  !=
tuple 立刻可物理删除
```

再让 Session A：

```sql
COMMIT;
```

Session B 再执行：

```sql
VACUUM (VERBOSE) htsv_demo;
```

这一次旧版本更可能被移除。

这里不要纠结 `n_dead_tup` 的即时精确值。

统计视图是估算和异步刷新的。

本节关心的是因果关系：

```text
long snapshot
  -> OldestXmin 无法前进
  -> deletion xid 不能证明早于全局安全边界
  -> recently dead 保留
  -> long snapshot 结束
  -> horizon 前移
  -> dead tuple 可以被回收
```

这就是 `HEAPTUPLE_RECENTLY_DEAD` 的运行时意义。

## 5. 关键数据结构与状态

### `HTSV_Result`

`HTSV_Result` 不是 SQL 层 visible / invisible 的重命名。

它是 VACUUM 视角下的 tuple 状态分类。

核心状态可以这样理解：

| 状态 | 语义 |
| --- | --- |
| `HEAPTUPLE_LIVE` | 插入事务已提交，且没有真正删除它的已提交事务。 |
| `HEAPTUPLE_DEAD` | 对所有需要保护的观察者都不再可见，可以被物理清理。 |
| `HEAPTUPLE_RECENTLY_DEAD` | 删除事务已提交，但删除还不够老，仍可能被旧 snapshot 需要。 |
| `HEAPTUPLE_INSERT_IN_PROGRESS` | 插入事务还在进行，不能当成可清理。 |
| `HEAPTUPLE_DELETE_IN_PROGRESS` | 删除或更新事务还在进行，不能当成已死。 |

这个枚举刻意没有直接对应 SQL visibility。

一个 tuple 可以对当前查询不可见。

但对 VACUUM 是 `RECENTLY_DEAD`。

一个 tuple 的插入事务 abort。

它从未对任何事务可见，可以直接是 `DEAD`。

一个 tuple 的 `xmax` 只是锁。

它在 VACUUM 视角仍然是 `LIVE`。

### `dead_after`

`HeapTupleSatisfiesVacuumHorizon()` 有一个输出参数：

```text
TransactionId *dead_after
```

当函数返回 `HEAPTUPLE_RECENTLY_DEAD` 时，`dead_after` 保存需要和 horizon 比较的事务 ID。

普通删除场景下，它通常是删除事务的 `xmax`。

MultiXact 更新场景下，它会取真正 updater 的 XID。

这比直接返回布尔值更重要。

因为同一个 tuple 的“是否 removable”可能要和不同 horizon 比较。

例如 VACUUM 初始 `OldestXmin`、on-access pruning 的 `GlobalVisState`、assertion 或特殊 snapshot 都可能需要不同视角。

源码因此先拆成两步：

```text
HeapTupleSatisfiesVacuumHorizon()
  -> 判断 tuple 本身的事务状态
  -> 返回 RECENTLY_DEAD 和 dead_after

HeapTupleSatisfiesVacuum()
  -> 用 OldestXmin 决定是否升级为 DEAD
```

### `OldestXmin`

`OldestXmin` 是 VACUUM 可使用的删除安全边界。

它来自：

```text
vacuum_get_cutoffs()
  -> GetOldestNonRemovableTransactionId(rel)
     -> ComputeXidHorizons()
```

它不是当前查询 snapshot 的 `xmin`。

它也不是系统中最小已提交事务。

它是 relation-aware 的 conservative horizon。

普通数据表、catalog、shared relation、temporary relation 可能使用不同 horizon。

因此同一个 XID 是否 removable，要看 relation kind。

本节关注普通 heap table。

### tuple header 与 line pointer 的分层

tuple header 记录事务身份和 tuple 内部标志。

line pointer 记录页内 item slot 状态。

两层不能混在一起。

| 层次 | 例子 | 回答的问题 |
| --- | --- | --- |
| tuple header | `xmin`、`xmax`、hint bit、HOT bit | 这个 tuple version 的事务语义是什么。 |
| vacuum status | `HTSV_Result` | VACUUM 当前能否把它视为 live、recently dead 或 dead。 |
| line pointer | `LP_NORMAL`、`LP_REDIRECT`、`LP_DEAD`、`LP_UNUSED` | 页内 slot 是否仍有 tuple storage、是否被索引引用、是否可复用。 |

`HEAPTUPLE_DEAD` 不等于 `LP_DEAD`。

`LP_DEAD` 不等于 `LP_UNUSED`。

这两个区别后面会反复出现。

## 6. 从 tuple 事务状态到 vacuum 状态

`HeapTupleSatisfiesVacuumHorizon()` 的第一问是插入事务是否提交。

如果 `xmin` 被标记 invalid，tuple 从未真正存在于 MVCC 世界。

它可以直接 `HEAPTUPLE_DEAD`。

如果 `xmin` 是当前事务，并且当前事务还没有完成插入语义，它返回 insert in progress。

如果 `xmin` 仍在 running，它返回 insert in progress。

如果 `xmin` 已提交，函数才进入第二问：

```text
删除事务 xmax 怎么样？
```

如果 `xmax` invalid，tuple 仍 live。

如果 `xmax` 只是锁，tuple 仍 live。

锁会占用 `xmax` 字段。

但它不代表 tuple 被删除。

这就是误判高发点。

当 `xmax` 是真正删除或更新事务时，函数继续判断：

```text
xmax 还在进行中:
  -> HEAPTUPLE_DELETE_IN_PROGRESS

xmax abort:
  -> HEAPTUPLE_LIVE

xmax commit:
  -> HEAPTUPLE_RECENTLY_DEAD + dead_after = xmax
```

注意最后一步还不是 `DEAD`。

提交只证明删除发生了。

提交不能证明所有旧 snapshot 都不需要删除前的版本。

`HeapTupleSatisfiesVacuum()` 之后才做：

```text
if res == HEAPTUPLE_RECENTLY_DEAD:
    if dead_after < OldestXmin:
        res = HEAPTUPLE_DEAD
```

这就是本节的核心分界。

`RECENTLY_DEAD` 是已删除但不能清。

`DEAD` 是已删除且安全可清。

## 7. 主流程源码 walkthrough

主流程从一次 VACUUM 扫描普通 heap 页开始。

```text
vacuum_rel()
  -> heap_vacuum_rel()
     -> vacuum_get_cutoffs()
     -> lazy_scan_heap()
        -> lazy_scan_prune()
           -> heap_page_prune_and_freeze()
              -> heap_prune_satisfies_vacuum()
                 -> HeapTupleSatisfiesVacuumHorizon()
                 -> compare OldestXmin / GlobalVisState
```

第一步，`vacuum_get_cutoffs()` 计算 `OldestXmin`。

它调用 `GetOldestNonRemovableTransactionId(rel)`。

这个函数会根据 relation kind 选择 horizon。

普通数据表通常使用 data horizon。

catalog 还要考虑 `catalog_xmin`。

temporary relation 可以更激进，因为只有当前 backend 的修改相关。

第二步，`heap_vacuum_rel()` 把 cutoffs 存进 `LVRelState`。

```text
vacrel->cutoffs.OldestXmin
vacrel->cutoffs.FreezeLimit
vacrel->cutoffs.OldestMxact
vacrel->vistest
```

其中 `OldestXmin` 负责本节的 recently dead / dead 边界。

`FreezeLimit` 会在第 29 节展开。

第三步，`lazy_scan_heap()` 按 block 扫描 heap。

如果拿到 cleanup lock，它走 `lazy_scan_prune()`。

如果拿不到 cleanup lock，它可能走 `lazy_scan_noprune()` 做较弱处理。

这两个路径都会关心 tuple 是否 live、recently dead、dead。

区别是：有 cleanup lock 才能真正修改 HOT chain 和 page layout。

第四步，`heap_page_prune_and_freeze()` 扫描页内 line pointer。

它不会在一开始就直接修改页面。

它先计划：

```text
哪些 item 要 redirect
哪些 item 要 nowdead
哪些 item 要 nowunused
哪些 tuple 要 freeze
是否可以设置 VM bit
```

这样可以把会发生 I/O 或事务状态查询的逻辑放在 critical section 之外。

第五步，`heap_prune_satisfies_vacuum()` 调用 `HeapTupleSatisfiesVacuumHorizon()`。

如果结果不是 `RECENTLY_DEAD`，直接返回。

如果结果是 `RECENTLY_DEAD`，它先看 VACUUM cutoffs。

```text
dead_after < OldestXmin
  -> HEAPTUPLE_DEAD
```

如果 cutoffs 不足以证明，还会用 `GlobalVisTestIsRemovableXid()`。

这允许 on-access pruning 和 VACUUM 使用近似但安全的 global visibility 状态。

第六步，pruning 根据结果决定 line pointer。

一个 dead tuple 可能变成：

```text
LP_REDIRECT
LP_DEAD
LP_UNUSED
```

取决于它在 HOT chain 中的位置、是否有索引引用、VACUUM 是否允许立刻 mark unused。

本节不展开 HOT chain 的全部细节。

这里先记住：

```text
HTSV_Result 是 tuple-level judgment；
line pointer change 是 page-level action。
```

## 8. 生命周期 / ownership / cleanup

旧 tuple version 的生命周期可以分成七个阶段。

阶段一：插入。

```text
xmin = inserting xid
xmax invalid
line pointer = LP_NORMAL
```

如果插入事务 abort，这个 tuple 后续可被视为 dead。

它从未被其他事务可见过。

阶段二：成为可见版本。

插入事务提交后，hint bit 可能被设置为 `HEAP_XMIN_COMMITTED`。

tuple 是 live。

阶段三：被 UPDATE 或 DELETE 取代。

旧版本的 `xmax` 被设置。

UPDATE 还会通过 `t_ctid` 链到新版本。

HOT update 还会设置 HOT 相关标志。

阶段四：删除事务仍在进行。

VACUUM 视角通常是 `HEAPTUPLE_DELETE_IN_PROGRESS`。

不能清理。

阶段五：删除事务提交，但 horizon 未越过。

VACUUM 视角是 `HEAPTUPLE_RECENTLY_DEAD`。

这个状态最容易被误解。

它不是“事务最近才提交”的 wall-clock 概念。

它是：

```text
删除 XID 相对当前 cleanup horizon 还不够老
```

阶段六：horizon 越过删除 XID。

`dead_after < OldestXmin` 成立。

VACUUM 视角升级为 `HEAPTUPLE_DEAD`。

阶段七：页内物理清理。

pruning 或 VACUUM 可以改 line pointer。

有索引引用的 root item 可能先变 `LP_DEAD`。

没有索引风险或 heap-only tuple 可能直接变 `LP_UNUSED`。

索引清理完成后，VACUUM 第二个 heap pass 会把记录的 `LP_DEAD` 变为 `LP_UNUSED`。

ownership 也要分层理解。

tuple 存储属于 heap page。

tuple 可见性由事务系统解释。

cleanup 许可由 horizon 保护。

line pointer 复用由 buffer cleanup lock、WAL 和索引一致性约束。

没有一个单独字段能代表完整生命周期。

## 9. 正确性机制层次

第一层正确性是事务结果。

`TransactionIdDidCommit()`、`TransactionIdIsInProgress()`、hint bit 和 CLOG 共同决定 `xmin` / `xmax` 的结果。

hint bit 只是缓存。

它不能改变事务事实。

第二层正确性是锁语义。

`xmax` 可能只是 row lock。

`HEAP_XMAX_IS_LOCKED_ONLY()` 和 MultiXact 相关检查必须先排除锁。

否则 VACUUM 会把被锁但仍 live 的 tuple 当成 dead。

第三层正确性是 horizon。

`OldestXmin` 必须足够保守。

它要考虑 active snapshot、registered snapshot、prepared transaction、replication slot、standby feedback 和 relation kind。

如果 horizon 过于激进，会删除旧 snapshot 仍需要的版本。

第四层正确性是 HOT / index 边界。

即使 tuple 已经 dead，也不能随意把索引可达的 line pointer 清成 unused。

索引中还可能有 TID 指向 root line pointer。

这就是 `LP_DEAD` 存在的原因。

第五层正确性是 WAL 和 crash recovery。

真正改变 page layout、line pointer 或 VM bit 时，需要 WAL 记录可重放。

hint bit 的持久化规则不同。

第六层正确性是 buffer lock。

`heap_page_prune_and_freeze()` 要在 cleanup lock 或合适的 exclusive lock 下应用 page 修改。

判断可以在 critical section 外做。

修改必须在正确锁和 critical section 中做。

## 10. 错误路径 / 异常路径 / fallback

### 插入事务 abort

插入事务 abort 的 tuple 不需要等待 cleanup horizon。

它从未对其他事务可见。

`HeapTupleSatisfiesVacuumHorizon()` 可以直接返回 `HEAPTUPLE_DEAD`。

### 删除事务还在进行

删除事务进行中时，tuple 不是 recently dead。

它是 delete in progress。

VACUUM 不能把它当成删除已完成。

### `xmax` 只是锁

行锁会写入 `xmax`。

但锁不是删除。

如果 `xmax` 是 locked-only，tuple 在 VACUUM 视角仍 live。

### MultiXact

MultiXact 可能包含多个 locker 和 updater。

对于真正更新者，源码会取 `HeapTupleGetUpdateXid()`。

如果 updater 已提交，`dead_after` 是 updater XID。

如果只是 lockers，tuple 仍 live。

### cleanup lock 拿不到

`lazy_scan_heap()` 可能拿不到 cleanup lock。

这时不能做完整 pruning。

`lazy_scan_noprune()` 仍可在 share lock 下统计和收集已有 `LP_DEAD`。

但如果 aggressive VACUUM 需要 freeze，可能必须等待 cleanup lock。

### horizon 被长期钉住

长事务、prepared transaction、logical replication slot 或 standby feedback 都可能让 `OldestXmin` 长期停在过去。

结果是大量 tuple 保持 recently dead。

VACUUM 可以扫描。

但不能移除这些版本。

这会表现为 bloat 增长。

### failsafe

如果 relfrozenxid 或 relminmxid 太旧，VACUUM 会触发 failsafe。

failsafe 会绕过非必要维护，比如进一步 index vacuuming / heap vacuuming。

这不是“清理更彻底”。

它是为了优先完成防 wraparound 所需的工作。

## 11. 成本、资源与跨模块传播

tuple 状态判断的成本主要来自几个维度。

第一，CLOG / pg_xact 查询。

如果 hint bit 已经设置，判断会更便宜。

如果 hint bit 没有设置，可能需要查询事务状态。

第二，ProcArray / horizon 计算。

VACUUM 不会为每个 tuple 重新全量计算 horizon。

它在 relation 级别获取 cutoffs。

但 GlobalVisState 仍然是系统级近似边界的一部分。

第三，MultiXact。

含有 MultiXact 的 tuple 需要额外解释 locker / updater。

这会比普通单 XID `xmax` 更贵。

第四，cleanup lock contention。

如果页面被 pin 或被其他 backend 使用，VACUUM 可能拿不到 cleanup lock。

这会把一些本可清理的 tuple 推迟到下次。

第五，索引数量。

tuple 成为 dead 不代表立刻释放所有空间。

如果 root TID 仍被索引引用，VACUUM 需要记录 dead item，再跑 index bulk delete。

索引越多，`LP_DEAD -> LP_UNUSED` 的后续成本越高。

第六，统计与观测滞后。

`pg_stat_user_tables.n_dead_tup` 不是逐 tuple 强一致计数。

它适合看趋势，不适合逐条对账。

跨模块传播可以总结为：

```text
heap tuple header
  -> heapam_visibility.c 判定状态
  -> procarray.c 提供保守 horizon
  -> pruneheap.c 修改 page-local structure
  -> vacuumlazy.c 协调 heap/index cleanup
  -> stats/progress/log 输出观测信号
```

这条链路比“VACUUM 删除死行”复杂得多。

## 12. 观测与诊断入口

先看是否有长 snapshot：

```sql
SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

再看表级 dead tuple 压力：

```sql
SELECT relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum,
       vacuum_count, autovacuum_count
FROM pg_stat_user_tables
WHERE relname = 'htsv_demo';
```

再看 VACUUM 进度：

```sql
SELECT pid, phase, heap_blks_total, heap_blks_scanned,
       heap_blks_vacuumed, index_vacuum_count,
       max_dead_tuple_bytes, dead_tuple_bytes,
       num_dead_item_ids
FROM pg_stat_progress_vacuum;
```

再用 verbose 输出确认 removable cutoff：

```sql
VACUUM (VERBOSE) htsv_demo;
```

关注几类信息：

```text
tuples removed
tuples remain
dead but not yet removable
removable cutoff
index scans
pages scanned
```

如果 dead but not yet removable 很多，优先查 horizon。

如果 dead item identifiers 很多，优先查 index cleanup 和 HOT/pruning。

如果 missed dead tuples 出现，优先查 cleanup lock contention。

如果 failsafe 出现，优先查 relfrozenxid、长事务、slot 和 autovacuum 是否长期跟不上。

源码断点建议：

```text
HeapTupleSatisfiesVacuumHorizon
HeapTupleSatisfiesVacuum
heap_prune_satisfies_vacuum
lazy_scan_prune
vacuum_get_cutoffs
GetOldestNonRemovableTransactionId
```

断点里重点看：

```text
tuple xmin
tuple xmax
t_infomask
HTSV_Result
dead_after
OldestXmin
relation kind
```

## 13. 常见误区

误区一：当前查询看不到旧版本，所以 VACUUM 可以删。

正确理解：当前查询不可见只说明本 snapshot 不需要它。

VACUUM 还要保护所有仍可能需要旧版本的观察者。

误区二：`RECENTLY_DEAD` 表示删除事务还没提交。

正确理解：删除事务通常已经提交。

只是 `dead_after` 还没有早于 cleanup horizon。

误区三：`xmax` valid 就是 dead。

正确理解：`xmax` 可能只是锁，也可能是 MultiXact。

必须判断 locked-only、updater、commit / abort / in progress。

误区四：`HEAPTUPLE_DEAD` 等于页面空间已经释放。

正确理解：它只是 tuple-level cleanup 许可。

page-level 还要看 HOT chain、line pointer、index cleanup 和 WAL。

误区五：`LP_DEAD` 已经可以给新 tuple 复用。

正确理解：`LP_DEAD` 仍是 line pointer 状态，不等于 `LP_UNUSED`。

索引中可能还有 TID 指向它。

误区六：`n_dead_tup` 可以精确解释每次 VACUUM 结果。

正确理解：它是统计估算。

要结合 verbose、progress、backend_xmin 和源码判断。

误区七：只要加大 autovacuum 频率就能解决 recently dead。

正确理解：如果 horizon 被钉住，VACUUM 再勤也不能安全删除。

要找钉住 horizon 的对象。

## 14. 课堂实验

### 实验一：长事务制造 recently dead

Session A：

```sql
DROP TABLE IF EXISTS htsv_demo;
CREATE TABLE htsv_demo(id int primary key, payload text);
INSERT INTO htsv_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 50000) AS g;

BEGIN;
SELECT count(*) FROM htsv_demo;
```

Session B：

```sql
DELETE FROM htsv_demo WHERE id <= 30000;
VACUUM (VERBOSE) htsv_demo;
```

观察：

```sql
SELECT pid, backend_xmin, xact_start, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

预期解释：

```text
DELETE 已提交。
新 snapshot 看不到旧版本。
旧 snapshot 钉住 OldestXmin。
VACUUM 只能把一部分 tuple 归为 recently dead。
```

然后提交 Session A：

```sql
COMMIT;
```

Session B：

```sql
VACUUM (VERBOSE) htsv_demo;
```

比较两次 verbose 输出。

重点不是数字完全一致。

重点是 horizon 释放后的清理能力变化。

### 实验二：区分锁和删除

Session A：

```sql
DROP TABLE IF EXISTS lock_not_dead;
CREATE TABLE lock_not_dead(id int primary key, payload text);
INSERT INTO lock_not_dead VALUES (1, 'a');

BEGIN;
SELECT * FROM lock_not_dead WHERE id = 1 FOR UPDATE;
```

Session B：

```sql
VACUUM (VERBOSE) lock_not_dead;
SELECT * FROM lock_not_dead;
```

预期解释：

```text
FOR UPDATE 可能写 xmax 或 MultiXact。
但它不是删除。
HeapTupleSatisfiesVacuum() 不能把 locked-only tuple 判为 dead。
```

### 实验三：观察 index cleanup 后的第二阶段

```sql
DROP TABLE IF EXISTS htsv_index_demo;
CREATE TABLE htsv_index_demo(id int primary key, payload text);
INSERT INTO htsv_index_demo
SELECT g, repeat('z', 100)
FROM generate_series(1, 80000) AS g;

DELETE FROM htsv_index_demo WHERE id % 2 = 0;
VACUUM (VERBOSE) htsv_index_demo;
```

观察 verbose 中：

```text
index scans
dead item identifiers
heap pages vacuumed
```

把它映射回：

```text
HEAPTUPLE_DEAD
  -> LP_DEAD
  -> index bulk delete
  -> LP_UNUSED
```

### 实验四：源码断点

在测试环境启动 backend 后设置断点：

```gdb
break HeapTupleSatisfiesVacuumHorizon
break HeapTupleSatisfiesVacuum
break heap_prune_satisfies_vacuum
```

运行：

```sql
VACUUM htsv_demo;
```

观察变量：

```gdb
print res
print *dead_after
print OldestXmin
```

把断点结果和 SQL 现象对应起来。

## 15. 讨论题

1. 为什么 PostgreSQL 不直接用当前 VACUUM 的 snapshot 判断 tuple 是否可删除？

2. 如果 `RECENTLY_DEAD` tuple 数量长期很高，优先排查哪些对象？

3. `LP_DEAD` 为什么不能直接等价为页面空间可复用？

4. 为什么 locked-only `xmax` 必须在 VACUUM 状态判断中被特别排除？

5. `dead_after < OldestXmin` 这个比较为什么必须使用 XID 顺序而不是 wall-clock 时间？

6. 如果 logical replication slot 长期持有很老的 `catalog_xmin`，普通表和 catalog 表的 cleanup 影响有什么不同？

7. `HeapTupleIsSurelyDead()` 为什么只能在非常保守的前提下返回 true？

## 16. 本节小结

本节建立的是一个边界模型。

```text
当前查询不可见
  !=
全局安全可回收
```

`HeapTupleSatisfiesVacuumHorizon()` 把 tuple header、事务结果、锁语义和 MultiXact 解释成 VACUUM 视角状态。

`HEAPTUPLE_RECENTLY_DEAD` 表示删除已提交但 horizon 还没有证明安全。

`HeapTupleSatisfiesVacuum()` 用 `OldestXmin` 把 recently dead 升级为 dead。

`HEAPTUPLE_DEAD` 只是 tuple-level cleanup 许可。

真正释放页面空间还要经过 page pruning、HOT chain 规则、索引清理、line pointer 转换和 WAL。

本节可以压缩成一句可迁移规律：

```text
物理回收不能依赖局部不可见性；
它必须依赖全局、保守、可解释的安全边界。
```

下一节沿着这个边界继续向页内推进：当 tuple 已经可以清理时，heap page pruning 如何在 HOT chain、line pointer 和 index TID 之间保持一致。
