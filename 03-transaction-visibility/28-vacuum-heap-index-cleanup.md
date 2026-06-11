# PostgreSQL VACUUM heap scan 与 index cleanup

## 课程定位

前置知识：已经理解 dead / recently dead 的 tuple-level 边界，也已经理解 heap page pruning 为什么会产生 `LP_DEAD`、`LP_UNUSED` 和 HOT chain redirect。

本节唯一主问题：

```text
为什么 lazy VACUUM 不能在发现 dead heap tuple 时立刻完成所有清理，而要拆成 heap scan、index vacuum、heap cleanup、index cleanup 和 final cleanup 多个阶段？
```

核心矛盾：heap 页面上的 dead item 需要尽快释放空间，但索引中可能仍有 TID 指向这些 heap item；如果先释放 heap slot 再清索引，索引会留下危险指针；如果每发现一个 dead item 就清所有索引，成本会失控。

学完后应能判断：`dead_items` 为什么是 VACUUM 的核心跨阶段状态，`LP_DEAD -> index bulk delete -> LP_UNUSED` 为什么是分阶段协议，`index_cleanup=auto/off/on`、bypass optimization 和 failsafe 分别改变哪部分工作。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 26 节回答 tuple 什么时候可以被视为 dead。

第 27 节回答页内 pruning 如何在 HOT chain 和 line pointer 上保持索引可达性。

本节把范围扩大到整张表。

VACUUM 的核心难题不是“看见 dead tuple 就删除”。

真正难题是三方一致性。

```text
heap page:
  dead tuple storage 占空间

index:
  可能仍有 TID 指向 heap root line pointer

VACUUM memory:
  需要记住哪些 TID 待清理，但内存有上限
```

所以 lazy VACUUM 的主线是：

```text
heap scan
  -> pruning / freeze / collect LP_DEAD TIDs
  -> dead_items 达到内存阈值时暂停 heap scan
  -> index bulk delete
  -> second heap pass 把 LP_DEAD 改 LP_UNUSED
  -> 继续 heap scan
  -> final index cleanup / relation stats / truncation
```

本节不展开 freeze 语义。

第 29 节会单独讲 freeze 和 anti-wraparound。

本节也不展开 all-visible / all-frozen bit。

第 30、31 节会拆开讲。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
lazy VACUUM 第一遍扫描 heap，prune 页面并把仍需索引清理的 LP_DEAD TID 放入 dead_items；
当 dead_items 满或 heap scan 完成后，VACUUM 调用各 index AM 的 ambulkdelete 删除这些 TID；
随后第二遍只访问 dead_items 覆盖的 heap pages，把对应 LP_DEAD 改为 LP_UNUSED；
最后执行 index cleanup、统计更新和可选 truncation。
```

本节核心矛盾是：

```text
heap 空间回收需要尽快把 dead item 变 reusable
  vs
索引一致性要求所有 index entry 先删除
  vs
逐 tuple 清索引会把 VACUUM 成本放大到不可接受
```

PostgreSQL 的折中是批处理。

它把 dead heap TID 收集成 `TidStore`。

它让 index AM 批量删除。

它再回到 heap 做二次清理。

这样做牺牲了一点点即时性。

换来索引一致性和批量成本控制。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/vacuumlazy.c` | 本节主文件。阅读 `heap_vacuum_rel()`、`lazy_scan_heap()`、`lazy_vacuum()`、`lazy_vacuum_all_indexes()`、`lazy_vacuum_heap_rel()`。 |
| 2 | `src/backend/access/heap/pruneheap.c` | `heap_page_prune_and_freeze()` 产生 `presult.lpdead_items` 和 `deadoffsets`。 |
| 3 | `src/backend/commands/vacuum.c` | `vacuum_rel()`、`vac_bulkdel_one_index()`、`vac_cleanup_one_index()`、options 和 failsafe cutoff。 |
| 4 | `src/include/commands/vacuum.h` | `VacuumParams`、`VacDeadItemsInfo`、parallel vacuum API、`VacuumCutoffs`。 |
| 5 | `src/include/commands/progress.h` | VACUUM progress phase 与观测字段。 |
| 6 | `src/include/access/tableam.h` | table AM VACUUM 边界，本节关注 heap AM 实现。 |
| 7 | `src/backend/access/nbtree/nbtree.c` | B-tree index AM 如何通过 `ambulkdelete` / `amvacuumcleanup` 参与，作为索引侧对照。 |
| 8 | `src/backend/access/heap/visibilitymap.c` | 第二遍 heap cleanup 后可能设置 VM bit，本节只作为边界提及。 |

阅读顺序要以状态流为轴。

不要从 `vacuum.c` 的 SQL option parsing 开始。

先抓住：

```text
LVRelState
  -> dead_items
  -> lazy_scan_heap
  -> lazy_vacuum
  -> lazy_vacuum_heap_rel
```

## 4. 一个 runtime 现象先定锚

准备一张有多个索引的表。

```sql
DROP TABLE IF EXISTS vacuum_phase_demo;
CREATE TABLE vacuum_phase_demo(
    id bigint primary key,
    k1 int,
    k2 int,
    payload text
);

CREATE INDEX vacuum_phase_demo_k1_idx ON vacuum_phase_demo(k1);
CREATE INDEX vacuum_phase_demo_k2_idx ON vacuum_phase_demo(k2);

INSERT INTO vacuum_phase_demo
SELECT g, g % 1000, g % 100, repeat('x', 120)
FROM generate_series(1, 300000) AS g;

DELETE FROM vacuum_phase_demo WHERE id % 3 = 0;
```

运行：

```sql
VACUUM (VERBOSE) vacuum_phase_demo;
```

同时在另一个会话观察：

```sql
SELECT pid, phase,
       heap_blks_total,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       num_dead_item_ids,
       dead_tuple_bytes
FROM pg_stat_progress_vacuum;
```

你会看到 VACUUM phase 切换。

典型顺序是：

```text
scanning heap
vacuuming indexes
vacuuming heap
index cleanup
final cleanup
```

如果 dead item 数很多，heap scan 过程中还可能多次进入 index vacuum / heap vacuum cycle。

这说明 VACUUM 不是一个单向表扫描。

它是一个受内存上限和索引一致性约束驱动的分阶段状态机。

## 5. `LVRelState` 是 lazy VACUUM 的运行态

`vacuumlazy.c` 中的 `LVRelState` 是本节核心结构。

它不是持久元数据。

它是一次 VACUUM 对某张 relation 的运行态。

可以把字段分成几组。

| 字段组 | 代表字段 | 语义 |
| --- | --- | --- |
| relation 与 indexes | `rel`、`indrels`、`nindexes` | 当前 VACUUM 的 heap relation 和已打开索引。 |
| cutoffs | `cutoffs`、`vistest` | 判断 dead、freeze 和 global visibility 的边界。 |
| 阶段控制 | `do_index_vacuuming`、`do_index_cleanup`、`do_rel_truncate` | 哪些阶段允许执行。 |
| dead item 状态 | `dead_items`、`dead_items_info` | 需要从索引中删除的 heap TID 集合。 |
| 页面计数 | `rel_pages`、`scanned_pages`、`lpdead_item_pages` | 扫描、清理和统计使用。 |
| VM / freeze 计数 | `new_all_visible_pages`、`new_all_frozen_pages`、`tuples_frozen` | 后续可见性图和 freeze 统计。 |
| 错误上下文 | `dbname`、`relnamespace`、`relname`、`indname`、`blkno`、`offnum` | VACUUM 错误报告定位。 |
| progress / instrumentation | `num_index_scans`、`worker_usage`、`total_dead_items_bytes` | progress、verbose 和 autovacuum log 输出。 |

其中 `dead_items` 是本节主角。

它保存的是 heap TID。

这些 TID 指向已经被 pruning 判定为 `LP_DEAD` 的 line pointer。

索引 AM 使用这些 TID 判断哪些 index tuple 应该删除。

然后 heap 第二遍再把相同 TID 对应的 `LP_DEAD` 改为 `LP_UNUSED`。

这就是跨阶段 ownership：

```text
第一遍 heap scan 产生清理任务；
dead_items 持有任务；
index vacuum 消费任务的一半；
second heap pass 完成任务；
dead_items_reset() 清空任务。
```

## 6. 主流程源码 walkthrough

主流程从 `heap_vacuum_rel()` 开始。

```text
vacuum_rel()
  -> table_relation_vacuum()
     -> heap_vacuum_rel()
        -> vac_open_indexes()
        -> vacuum_get_cutoffs()
        -> dead_items_alloc()
        -> lazy_scan_heap()
        -> dead_items_cleanup()
        -> update_relstats_all_indexes()
        -> lazy_truncate_heap()
        -> vac_update_relstats()
        -> pgstat_report_vacuum()
```

第一步，打开 heap relation 和 indexes。

`vac_open_indexes()` 用 `RowExclusiveLock` 打开索引。

VACUUM 后续需要调用索引 AM。

第二步，计算 cutoffs。

`vacuum_get_cutoffs()` 给出：

```text
OldestXmin
OldestMxact
FreezeLimit
MultiXactCutoff
```

本节主要用 `OldestXmin` 判断 dead item。

第三步，分配 `dead_items`。

串行 VACUUM 中，它是 local `TidStore`。

并行 VACUUM 中，相关状态可能在 shared memory 中，供 worker 和 leader 协调。

第四步，进入 `lazy_scan_heap()`。

这是第一遍 heap scan。

它用 read stream 扫 relation block。

每个 block 上尝试拿 cleanup lock。

拿到时调用 `lazy_scan_prune()`。

拿不到时可能调用 `lazy_scan_noprune()`。

第五步，`lazy_scan_prune()` 调用：

```text
heap_page_prune_and_freeze()
```

这个函数返回 `PruneFreezeResult`。

其中：

```text
presult.lpdead_items
presult.deadoffsets
presult.ndeleted
presult.nfrozen
presult.newly_all_visible
presult.newly_all_frozen
```

第六步，如果有 `lpdead_items`，VACUUM 排序 offsets 并调用：

```text
dead_items_add(vacrel, blkno, deadoffsets, lpdead_items)
```

这一步不删除索引。

它只记录“这些 heap TID 之后要从所有索引中清掉”。

第七步，`lazy_scan_heap()` 会监控 `TidStoreMemoryUsage()`。

如果 `dead_items` 超过内存上限，它会暂停 heap scan。

然后调用：

```text
lazy_vacuum(vacrel)
```

第八步，`lazy_vacuum()` 决定是否绕过 index vacuuming。

如果死 TID 极少，且满足 bypass 条件，它可能跳过本轮 index vacuuming。

如果不能 bypass，它调用：

```text
lazy_vacuum_all_indexes()
```

第九步，`lazy_vacuum_all_indexes()` 对每个索引调用 `lazy_vacuum_one_index()`。

`lazy_vacuum_one_index()` 进一步调用：

```text
vac_bulkdel_one_index()
```

索引 AM 的 `ambulkdelete` 使用 `dead_items` 判断 index tuple 是否应该删除。

第十步，索引清理完成后，VACUUM 才调用：

```text
lazy_vacuum_heap_rel()
```

这是第二遍 heap pass。

它只访问 `dead_items` 中出现过的 heap blocks。

每个页上调用：

```text
lazy_vacuum_heap_page()
```

把对应 `LP_DEAD` 改成 `LP_UNUSED`。

第十一步，`dead_items_reset()` 清空本轮 TID。

heap scan 继续。

如果 heap scan 完了，仍有 dead items，也会做最后一轮 `lazy_vacuum()`。

第十二步，final cleanup。

如果允许 index cleanup，调用每个 index AM 的 `amvacuumcleanup`。

然后更新 pg_class 统计、visibility map 计数、relfrozenxid / relminmxid 和 cumulative stats。

## 7. 生命周期 / ownership / cleanup

一次 dead heap item 的生命周期如下。

阶段一：DELETE 或 UPDATE 产生旧版本。

tuple 可能先是 recently dead。

horizon 过去后成为 dead。

阶段二：第一遍 heap scan 发现 dead。

`heap_page_prune_and_freeze()` 判断可以剪掉。

如果该 item 仍可能被索引指向，页面上留下 `LP_DEAD`。

阶段三：`dead_items` 记录 TID。

TID 的 owner 是当前 VACUUM 操作。

它不是表的持久状态。

它也不是索引 AM 私有状态。

它是 VACUUM 在 heap 和 indexes 之间传递的任务集。

阶段四：index bulk delete。

每个索引 AM 遍历自己的 index structure。

当 index tuple 的 heap TID 在 `dead_items` 中，就删除这个 index tuple。

阶段五：第二遍 heap cleanup。

索引侧不再需要这些 TID 后，heap page 可以把 `LP_DEAD` 变 `LP_UNUSED`。

页面空间此时真正更接近可复用。

阶段六：`dead_items_reset()`。

本轮任务完成。

如果 heap scan 继续，新的 dead item 会进入下一轮。

阶段七：final cleanup。

索引 AM 可以更新统计或做后处理。

heap relstats 和 pgstat 报告也在最后完成。

这个生命周期里最重要的不变量是：

```text
不能先复用 heap TID，再删除索引 TID。
```

否则索引可能指向一个已经被新 tuple 复用的 slot。

## 8. 正确性机制层次

第一层是 visibility correctness。

只有 dead tuple 才能进入清理。

recently dead 必须保留。

第二层是 heap / index ordering。

`LP_DEAD` 是 heap 和 index 之间的中间状态。

它表示 heap 已经知道这个 root 不再有 live tuple。

但索引可能还没删除 TID。

第三层是 TID stability。

在 index cleanup 完成前，heap 不能把同一个 line pointer 复用给无关 tuple。

第四层是 memory bound。

`dead_items` 不可能无限增长。

达到上限后，VACUUM 必须暂停 heap scan，先完成一轮 index / heap cleanup。

第五层是 lock boundary。

第一遍 pruning 需要 cleanup lock。

第二遍把 `LP_DEAD` 改 `LP_UNUSED` 只需要对应的 exclusive buffer lock。

索引清理有索引 AM 自己的锁协议。

第六层是 WAL。

page layout 改动、pruning、freeze、VM bit 设置都需要 WAL 或 hint 规则保护。

第七层是 failsafe。

当 wraparound 风险升高时，VACUUM 会绕过非必要工作。

这优先保证 anti-wraparound 进展。

但可能牺牲索引 cleanup 和 heap reclaim 的完整性。

## 9. 错误路径 / 异常路径 / fallback

### `index_cleanup = off`

用户或 reloption 可以关闭 index cleanup。

这会让 VACUUM 不做 index vacuuming 和 cleanup。

heap 第一遍仍可做某些 pruning / freeze。

但有索引引用风险的 `LP_DEAD` 不能完成到 `LP_UNUSED`。

### bypass optimization

默认 `index_cleanup=auto` 时，如果 dead item 极少，VACUUM 可能绕过 index vacuuming。

源码用 `lpdead_item_pages` 占 relation pages 的比例和 `TidStoreMemoryUsage()` 作为判断。

它的目的不是偷懒。

它是为了避免少量 dead TID 导致全索引扫描成本突增。

### failsafe

当 relfrozenxid 或 relminmxid 过老，`lazy_check_wraparound_failsafe()` 会触发。

触发后：

```text
do_index_vacuuming = false
do_index_cleanup = false
do_rel_truncate = false
VacuumCostActive = false
```

VACUUM 优先完成防 wraparound 的必要扫描和 freeze。

这可能留下更多 bloat。

但比 wraparound 风险更可接受。

### cleanup lock 拿不到

第一遍 heap scan 拿不到 cleanup lock 时，可以走 `lazy_scan_noprune()`。

它可以收集已有 `LP_DEAD`。

但不能完整 pruning / freeze。

如果 aggressive VACUUM 需要 freeze 旧 XID，它可能必须等待 cleanup lock。

### dead_items 内存满

`dead_items` 超过 `maintenance_work_mem` 或 `autovacuum_work_mem` 限制时，VACUUM 做一轮清理再继续。

这会增加 index scans 次数。

verbose 中能看到 index scans 多于 1。

### parallel vacuum

parallel vacuum 主要并行索引阶段。

heap scan 仍由 leader 负责。

`dead_items` 在并行场景可能放在 shared memory。

并行不是改变正确性协议。

它只是改变 index vacuum / cleanup 的执行资源。

## 10. 成本、资源与跨模块传播

VACUUM 的成本不是只和 dead tuple 数量成正比。

主要成本维度包括：

| 维度 | 影响 |
| --- | --- |
| heap pages | 第一遍 scan 的 block 数决定 heap I/O 下限。 |
| indexes 数量 | 每轮 index vacuum 都要让每个索引 AM 参与。 |
| dead_items 大小 | 决定内存压力和是否需要多轮 index scan。 |
| dead item 分布 | 少量 dead item 分散在很多页上，比集中在少数页更难受。 |
| HOT 比例 | HOT 高时索引 TID 清理压力通常较低。 |
| cleanup lock contention | 影响 pruning 和 missed dead tuple。 |
| failsafe | 降低 wraparound 风险，但可能跳过非必要清理。 |
| VM bit | all-visible / all-frozen 决定未来能否跳过页面。 |

跨模块路径：

```text
commands/vacuum.c:
  SQL option、relation selection、index option、failsafe cutoff

vacuumlazy.c:
  heap scan、dead_items、index vacuum、heap cleanup、stats

pruneheap.c:
  page-local pruning、freeze、VM 判断

index AM:
  ambulkdelete / amvacuumcleanup

procarray.c:
  OldestXmin / cleanup horizon

pgstat / progress:
  runtime 观测
```

这个传播链解释了为什么 VACUUM 调优不能只调一个参数。

有时瓶颈是 horizon。

有时是 index count。

有时是 work_mem。

有时是 cleanup lock。

有时是 failsafe 被迫绕过。

## 11. 观测与诊断入口

最直接是 progress view。

```sql
SELECT pid,
       phase,
       heap_blks_total,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       max_dead_tuple_bytes,
       dead_tuple_bytes,
       num_dead_item_ids
FROM pg_stat_progress_vacuum;
```

phase 要结合 `progress.h` 理解：

```text
scanning heap
vacuuming indexes
vacuuming heap
index cleanup
truncating heap
performing final cleanup
```

表统计：

```sql
SELECT relname, n_live_tup, n_dead_tup,
       vacuum_count, autovacuum_count,
       last_vacuum, last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'vacuum_phase_demo';
```

索引大小：

```sql
SELECT c.relname,
       pg_size_pretty(pg_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_index i ON i.indexrelid = c.oid
WHERE i.indrelid = 'vacuum_phase_demo'::regclass
ORDER BY pg_relation_size(c.oid) DESC;
```

verbose 输出重点看：

```text
index scans
dead item identifiers
pages scanned
pages removed
tuples removed
tuples are dead but not yet removable
index scan bypassed
index scan bypassed by failsafe
```

源码断点建议：

```text
heap_vacuum_rel
lazy_scan_heap
lazy_scan_prune
dead_items_add
lazy_vacuum
lazy_vacuum_all_indexes
lazy_vacuum_heap_rel
lazy_vacuum_heap_page
lazy_check_wraparound_failsafe
```

断点里重点看：

```text
vacrel->dead_items_info->num_items
TidStoreMemoryUsage(vacrel->dead_items)
vacrel->lpdead_item_pages
vacrel->num_index_scans
vacrel->do_index_vacuuming
vacrel->do_index_cleanup
VacuumFailsafeActive
```

## 12. 常见误区

误区一：VACUUM 扫到 dead tuple 就会立刻释放空间。

正确理解：有索引时，通常要先 `LP_DEAD`，再 index bulk delete，再第二遍 heap cleanup。

误区二：index cleanup 是可有可无的统计维护。

正确理解：index vacuuming 删除 dead TID 对应的 index tuple。

它是 heap TID 复用安全的重要前置。

误区三：`index_cleanup=off` 可以无代价加速 VACUUM。

正确理解：它会减少本次成本，但可能保留索引 bloat 和 `LP_DEAD` 后续成本。

误区四：index scans 越少一定越好。

正确理解：太多 index scans 说明 dead_items 内存不足或 dead item 很多。

但零 index scan 也可能表示 bypass 或 failsafe，未必表示没有 bloat。

误区五：failsafe 会做更彻底的清理。

正确理解：failsafe 是为了避免 wraparound。

它会跳过非必要维护。

误区六：parallel vacuum 会并行 heap scan。

正确理解：本源码基线中并行主要外包索引 vacuum / cleanup。

heap scan 主体仍由 leader 控制。

误区七：`n_dead_tup` 等于 `dead_items` 数量。

正确理解：`n_dead_tup` 是统计估算。

`dead_items` 是一次 VACUUM 运行中收集的具体 heap TID 任务集。

## 13. 课堂实验

### 实验一：观察 VACUUM phase

```sql
DROP TABLE IF EXISTS vacuum_phase_demo;
CREATE TABLE vacuum_phase_demo(
    id bigint primary key,
    k1 int,
    k2 int,
    payload text
);

CREATE INDEX vacuum_phase_demo_k1_idx ON vacuum_phase_demo(k1);
CREATE INDEX vacuum_phase_demo_k2_idx ON vacuum_phase_demo(k2);

INSERT INTO vacuum_phase_demo
SELECT g, g % 1000, g % 100, repeat('x', 120)
FROM generate_series(1, 300000) AS g;

DELETE FROM vacuum_phase_demo WHERE id % 3 = 0;
```

Session A：

```sql
VACUUM (VERBOSE) vacuum_phase_demo;
```

Session B：

```sql
SELECT now(), phase, heap_blks_scanned, heap_blks_vacuumed,
       index_vacuum_count, num_dead_item_ids
FROM pg_stat_progress_vacuum;
```

解释 phase 变化。

把每个 phase 映射回源码函数。

### 实验二：比较 HOT 与 non-HOT 对 index vacuum 的影响

建两张表。

一张只更新非索引列。

一张更新索引列。

比较：

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd, n_dead_tup
FROM pg_stat_user_tables
WHERE relname LIKE 'vacuum_%_demo';
```

再比较 `VACUUM VERBOSE` 中 index scan 和 dead item identifiers。

解释：

```text
HOT 降低新 index entry 数量。
non-HOT 增加索引清理压力。
```

### 实验三：强制关闭 index cleanup

在测试库中执行：

```sql
VACUUM (INDEX_CLEANUP OFF, VERBOSE) vacuum_phase_demo;
```

再执行普通：

```sql
VACUUM (VERBOSE) vacuum_phase_demo;
```

比较 verbose 输出。

关注 index scan 是否被跳过，以及后续是否仍有清理压力。

不要在生产上随意使用这个选项。

### 实验四：源码断点

```gdb
break dead_items_add
break lazy_vacuum
break lazy_vacuum_all_indexes
break lazy_vacuum_heap_rel
break lazy_vacuum_heap_page
```

执行：

```sql
VACUUM vacuum_phase_demo;
```

观察：

```gdb
print vacrel->dead_items_info->num_items
print vacrel->lpdead_item_pages
print vacrel->num_index_scans
print vacrel->do_index_vacuuming
```

把断点顺序和 progress phase 对应。

## 14. 讨论题

1. 为什么不能在第一遍 heap scan 中直接把所有 dead item 变 `LP_UNUSED`？

2. `dead_items` 的内存上限为什么会导致多次 index scans？

3. bypass optimization 为什么关注有 `LP_DEAD` 的 heap page 数，而不只是 dead TID 总数？

4. `index_cleanup=off` 适合什么临时场景，风险是什么？

5. failsafe 为什么宁可跳过索引清理，也要让 VACUUM 更快完成？

6. 如果 VACUUM 长期卡在 `vacuuming indexes`，你会优先检查哪些因素？

7. 为什么 `LP_DEAD` 到 `LP_UNUSED` 的转换必须发生在索引 TID 删除之后？

## 15. 本节小结

本节建立的是 lazy VACUUM 的分阶段模型。

```text
heap scan:
  找 dead item，prune 页面，收集 TID

index vacuum:
  用 dead_items 批量删除索引 TID

heap cleanup:
  把已清索引的 LP_DEAD 变成 LP_UNUSED

index cleanup / final cleanup:
  更新索引统计、relation stats、VM counts 和 pgstat
```

`dead_items` 是跨 heap 和 index 的任务集。

它让 VACUUM 能批量清索引。

也让 heap TID 复用等待索引一致性完成。

本节可迁移规律是：

```text
当一个资源同时被多个访问路径引用时，回收通常不是单点动作；
它需要一个可持有、可批处理、可观测的中间任务集来协调顺序。
```

下一节继续看 VACUUM 的另一个核心目标：不仅清理 dead tuple，还要通过 freeze 把旧 XID 从长期存储语义中移除，避免 transaction ID wraparound。
