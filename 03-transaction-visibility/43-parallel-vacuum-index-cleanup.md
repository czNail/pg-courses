# PostgreSQL parallel vacuum / index cleanup
## 课程定位
前置知识：已经读过第 28 节 lazy VACUUM 的 heap scan、index vacuum、heap cleanup 分阶段协议。
也应该理解 `dead_items` 为什么是 heap 与 index 之间的跨阶段任务集合。
本节唯一主问题：
```text
parallel vacuum 如何拆分 index cleanup，哪些状态仍必须由 leader 汇总？
```
核心矛盾：
```text
多个 index 的 bulk delete / cleanup 可以并行执行，
但 heap visibility、dead TID 集合、LP_DEAD 回收顺序、统计持久化和 ERROR 收尾
仍然必须由同一个 leader-owned VACUUM 生命周期统一收束。
```
学完后应能判断：
```text
parallel vacuum 并行的是 index phase，不是 heap scan。
一个 index 在一个 phase 中最多由一个进程处理。
worker 可以执行 ambulkdelete / amvacuumcleanup。
worker 不决定 cleanup horizon，不执行 heap second pass，不最终更新 pg_class。
DSM / DSA 只承载最小共享任务状态和统计 handoff。
```
本课基于本地 `/home/nail/postgres` 源码。
源码分支为 `master`，短提交为 `0e1f1ed157e`。
本地源码没有 `src/include/access/vacuum.h`。
parallel vacuum 的声明在 `src/include/commands/vacuum.h`。
## 1. 本节在总主线中的位置
第 26 节回答 dead tuple 什么时候越过 cleanup horizon。
第 27 节回答 heap page pruning 如何保持 HOT chain 与 line pointer 不变量。
第 28 节回答 VACUUM 为什么必须先清索引，再把 heap `LP_DEAD` 变成 `LP_UNUSED`。
第 39 节回答 autovacuum launcher / worker 如何选库选表。
本节只替换第 28 节主链路中的两段执行方式。
原来的串行模型是：
```text
heap scan
  -> collect dead_items
  -> loop all indexes: ambulkdelete
  -> heap second pass
  -> loop all indexes: amvacuumcleanup
  -> relstats / pgstat / truncation
```
parallel vacuum 后变成：
```text
heap scan
  -> collect dead_items in leader
  -> leader + workers process index bulk-delete tasks
  -> leader heap second pass
  -> leader + workers process index cleanup tasks
  -> leader copies stats out of DSM
  -> relstats / pgstat / truncation
```
没有改变的部分更重要。
heap scan 仍由 leader 执行。
`dead_items` 仍由 leader 追加。
heap second pass 仍由 leader 执行。
index relation stats 最终仍由 leader 更新。
parallel worker 只存在于 index phase 的局部窗口。
这就是本节主线：
```text
parallel vacuum 是 index-level task parallelism，
不是把 lazy VACUUM 状态机整体并行化。
```
## 2. 一句话运行模型
一句话运行模型：
```text
leader 在 heap scan 中把 LP_DEAD TID 写入共享 TidStore；
进入 index bulk-delete 或 index cleanup 时，
leader 在 DSM 中给每个 index 写任务状态，
启动 parallel workers；
leader 和 workers 用原子 idx 抢 index slot；
每个 slot 调用对应 index AM 的 ambulkdelete 或 amvacuumcleanup；
每个 index 的 IndexBulkDeleteResult 写回 DSM；
phase barrier 后 leader 汇总 worker usage、WAL usage、buffer usage、cost balance 和 index stats。
```
本节有三组 tension。
第一组是吞吐与启动成本。
多个 index scan 可以并行。
但 worker 启动、DSM 初始化、DSA attach 和 relation reopen 都有固定成本。
所以源码用 index 数、index 大小、AM capability 和 GUC 控制是否并行。
第二组是 AM 自治与 VACUUM 全局一致性。
每个 index AM 自己实现 vacuum callback。
但 dead TID 集合、heap line pointer 回收顺序、progress 和 relation stats 由 VACUUM core 统一维护。
所以 worker 只拿到自己的 `Relation`、`IndexVacuumInfo`、shared `TidStore` 和 DSM stats slot。
第三组是并行执行与 leader 汇总。
某个 index 可以由任意 worker 执行。
但 worker 私有内存不能成为后续 phase 的 owner。
所以 `IndexBulkDeleteResult` 必须被拷入 DSM，再由 leader 拷回本地。
本节最重要的不变量：
```text
并行化不能产生第二套 visibility 决策。
```
具体说：
```text
OldestXmin / GlobalVisState:
  leader 建立。
dead_items:
  leader 写入，worker 只读。
index bulk delete:
  worker 只消费 dead_items。
heap cleanup:
  所有 index 对同一批 dead_items 完成 bulk delete 后，leader 才能做。
final cleanup:
  worker 可以执行 AM callback，但最终 stats 仍由 leader 使用。
```
## 3. 核心文件与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/vacuumlazy.c` | `LVRelState`、`dead_items_alloc()`、`lazy_vacuum_all_indexes()`、`lazy_cleanup_all_indexes()`、`dead_items_cleanup()`。 |
| 2 | `src/backend/commands/vacuumparallel.c` | DSM / DSA 初始化、worker 数计算、index task 分发、worker 入口、结果汇总。 |
| 3 | `src/include/commands/vacuum.h` | AM parallel flags、`VacDeadItemsInfo`、`PVWorkerUsage`、parallel vacuum API。 |
| 4 | `src/backend/commands/vacuum.c` | `PARALLEL` option、cost delay、`vac_bulkdel_one_index()`、`vac_cleanup_one_index()`。 |
| 5 | `src/include/access/genam.h` | `IndexVacuumInfo`、`IndexBulkDeleteResult`、`IndexBulkDeleteCallback`。 |
| 6 | `src/include/access/amapi.h` | `IndexAmRoutine.amparallelvacuumoptions`。 |
| 7 | `src/backend/access/nbtree/nbtree.c` | B-tree parallel bulkdelete 和 conditional cleanup。 |
| 8 | `src/backend/access/gin/ginutil.c` | GIN parallel bulkdelete 和 cleanup。 |
| 9 | `src/backend/access/gist/gist.c` | GiST conditional cleanup。 |
| 10 | `src/backend/access/hash/hash.c` | hash parallel bulkdelete。 |
| 11 | `src/backend/access/spgist/spgutils.c` | SP-GiST conditional cleanup。 |
| 12 | `src/backend/access/brin/brin.c` | BRIN parallel cleanup。 |
| 13 | `src/include/commands/progress.h` | progress phase 与 index counters。 |
| 14 | `src/backend/catalog/system_views.sql` | `pg_stat_progress_vacuum` 列映射。 |
推荐阅读顺序：
```text
LVRelState
  -> dead_items_alloc
  -> parallel_vacuum_init
  -> parallel_vacuum_process_all_indexes
  -> parallel_vacuum_main
  -> parallel_vacuum_end
```
不要先从 SQL option 或文档读起。
本节要追的是 state ownership。
## 4. runtime 现象锚点
准备一张多索引表。
索引要超过 `min_parallel_index_scan_size`。
课堂环境可以临时把阈值设为 0。
```sql
DROP TABLE IF EXISTS pv_index_cleanup_demo;
CREATE TABLE pv_index_cleanup_demo(
    id bigint,
    k1 int,
    k2 int,
    k3 int,
    payload text
) WITH (autovacuum_enabled = off);
INSERT INTO pv_index_cleanup_demo
SELECT g, g % 1000, g % 100, g % 17, repeat('x', 120)
FROM generate_series(1, 800000) AS g;
CREATE INDEX pv_idx_id ON pv_index_cleanup_demo(id);
CREATE INDEX pv_idx_k1 ON pv_index_cleanup_demo(k1);
CREATE INDEX pv_idx_k2 ON pv_index_cleanup_demo(k2);
CREATE INDEX pv_idx_k3 ON pv_index_cleanup_demo(k3);
DELETE FROM pv_index_cleanup_demo WHERE id % 3 = 0;
SET max_parallel_maintenance_workers = 4;
SET min_parallel_index_scan_size = 0;
VACUUM (VERBOSE, PARALLEL 4, INDEX_CLEANUP ON) pv_index_cleanup_demo;
```
另一个会话观察：
```sql
SELECT pid, relid::regclass, phase,
       index_vacuum_count,
       indexes_total,
       indexes_processed,
       num_dead_item_ids,
       dead_tuple_bytes,
       delay_time
FROM pg_stat_progress_vacuum
WHERE relid = 'pv_index_cleanup_demo'::regclass;
```
典型 phase：
```text
scanning heap
vacuuming indexes
vacuuming heap
cleaning up indexes
performing final cleanup
```
`VACUUM VERBOSE` 中还可能看到：
```text
parallel workers: index vacuum: <planned> planned, <launched> launched in total
parallel workers: index cleanup: <planned> planned, <launched> launched
```
这说明三个事实。
第一，parallel worker 出现在 index phase。
第二，planned 不等于 launched。
第三，progress view 是 leader 聚合视图，不显示每个 worker 正在处理哪个 index。
## 5. 关键结构与状态边界
### 5.1 `LVRelState`: leader-owned 主状态
`LVRelState` 定义在 `vacuumlazy.c`。
它是一张 heap relation 的一次 lazy VACUUM 运行态。
本节只抓四组字段。
```text
relation/index:
  rel, indrels, nindexes, bstrategy, pvs
visibility and cleanup:
  cutoffs, vistest, do_index_vacuuming, do_index_cleanup
dead item task set:
  dead_items, dead_items_info, num_index_scans
result and diagnostics:
  indstats, worker_usage, phase, indname, blkno
```
`LVRelState` 不放入 DSM，worker 也没有 leader 的 `LVRelState *`。
parallel vacuum 只把 worker 必须看到的状态抽到 `PVShared`、`PVIndStats[]`、shared `TidStore`、usage arrays 和 query text。
### 5.2 `ParallelVacuumState`: 进程本地句柄
`ParallelVacuumState` 是 opaque type。
声明在 `src/include/commands/vacuum.h`。
实际结构在 `vacuumparallel.c`。
leader 端持有 `pcxt`、heap/index relations、DSM pointers、shared `TidStore`、worker usage arrays、`will_parallel_vacuum[]` 和各类 eligible index 计数。
worker 端也构造这个结构，但它是 worker 栈上的本地对象。
两端通过 DSM key 和 DSA handle 找到同一批共享状态。
```text
共享的是 DSM/DSA object，
不是 leader 的虚拟地址指针。
```
### 5.3 `PVShared`: DSM 中的 phase 输入
`PVShared` 是 DSM chunk。
它保存 worker 执行 index task 所需的共享参数。
它包括 relation identity、phase tuple estimate、worker memory/ring 设置、shared cost balance、原子 `idx`、shared `TidStore` handles、`dead_items_info` 和 autovacuum cost params。
它不保存 heap scan 进度、freeze cutoffs 或 relation stats 的最终更新权。
这些仍在 leader。
### 5.4 `PVIndStats`: 每个 index 的任务与结果槽
`PVIndStats` 是 DSM 数组。
数组长度等于 `nindexes`。
每个 index 一个 slot。
它包含 `status`、`parallel_workers_can_process`、`istat_updated` 和内嵌 `IndexBulkDeleteResult istat`。
它解决的是跨进程 stats ownership。
AM 第一次 `ambulkdelete()` 可能在 worker 私有内存里 `palloc` stats。
后续 phase 可能由另一个进程处理同一 index。
所以源码把第一次返回的 stats 拷入 DSM，后续调用传入 `&PVIndStats.istat`。
`parallel_vacuum_end()` 再把 DSM stats 拷回 leader 本地 `vacrel->indstats[i]`。
### 5.5 shared `TidStore`
`VacDeadItemsInfo` 在 `vacuum.h` 中只包含：
```text
max_bytes
num_items
```
serial VACUUM 使用 `TidStoreCreateLocal()`。
parallel VACUUM 使用 `TidStoreCreateShared()`。
leader 在 heap scan 中调用：
```text
TidStoreSetBlockOffsets()
dead_items_info->num_items += num_offsets
```
worker 在 index AM callback 中只读：
```text
vac_tid_reaped()
  -> TidStoreIsMember(dead_items, itemptr)
```
这说明 shared `TidStore` 的语义很窄：
```text
leader 产生 dead TID 集合；
worker 把它当作 membership set；
worker 不追加 dead TID。
```
### 5.6 AM parallel vacuum flags
`amparallelvacuumoptions` 挂在 `IndexAmRoutine` 上。
核心 flag：
| flag | 语义 |
| --- | --- |
| `VACUUM_OPTION_NO_PARALLEL` | 不参与 parallel vacuum。 |
| `VACUUM_OPTION_PARALLEL_BULKDEL` | `ambulkdelete` 可并行。 |
| `VACUUM_OPTION_PARALLEL_COND_CLEANUP` | 未做过 bulkdelete 时 cleanup 可并行。 |
| `VACUUM_OPTION_PARALLEL_CLEANUP` | cleanup 总是可并行。 |
本地源码中的例子：
```text
B-tree:
  PARALLEL_BULKDEL | PARALLEL_COND_CLEANUP
GiST:
  PARALLEL_BULKDEL | PARALLEL_COND_CLEANUP
SP-GiST:
  PARALLEL_BULKDEL | PARALLEL_COND_CLEANUP
GIN:
  PARALLEL_BULKDEL | PARALLEL_CLEANUP
hash:
  PARALLEL_BULKDEL
BRIN:
  PARALLEL_CLEANUP
```
这些 flag 是 AM 与 VACUUM core 的契约。
不是纯性能 hint。
AM 如果不能安全延续 stats，不能声明 parallel vacuum。
## 6. 主流程 walkthrough
### 6.1 SQL option 到 `nworkers`
`ExecVacuum()` 中默认：
```text
params.nworkers = 0
```
含义是启用 parallel vacuum，由系统自动选择 worker 数。
用户写：
```sql
VACUUM (PARALLEL 0) t;
```
源码设置：
```text
params.nworkers = -1
```
含义是禁用 parallel vacuum。
用户写：
```sql
VACUUM (PARALLEL 4) t;
```
`4` 是上限。
还会被 index 数、AM flag、index size 和 GUC 裁剪。
`VACUUM FULL` 不能和 `PARALLEL` 同时使用。
因为 `VACUUM FULL` 是 rewrite 路径，不走 lazy VACUUM 的 index task model。
autovacuum path 中，`autovacuum_parallel_workers` reloption 会写入 `at_params.nworkers`。
`autovacuum_max_parallel_workers` 再作为单个 autovacuum leader 可用的 parallel vacuum 上限。
### 6.2 leader 建立 `LVRelState`
`heap_vacuum_rel()` 已经持有 relation lock。
主链路：
```text
vac_open_indexes()
vacuum_get_cutoffs()
GlobalVisTestFor()
lazy_check_wraparound_failsafe()
dead_items_alloc()
lazy_scan_heap()
dead_items_cleanup()
update_relstats_all_indexes()
vac_close_indexes()
lazy_truncate_heap()
vac_update_relstats()
pgstat_report_vacuum()
```
parallel vacuum 只从 `dead_items_alloc()` 进入。
它不改变 cutoffs。
它不改变 `GlobalVisTestFor()`。
它不改变 heap scan 的 owner。
### 6.3 `dead_items_alloc()` 决定是否并行
并行初始化需要：
```text
nworkers >= 0
nindexes > 1
do_index_vacuuming
not temporary table using local buffers
```
`nindexes > 1` 的原因在源码注释里很直接：
```text
As of now, only one worker can be used for an index.
```
一张只有一个 index 的表没有 index-level task parallelism。
临时表使用 local buffers。
parallel worker 不能访问 leader 的 local buffer state。
所以临时表禁用 parallel vacuum。
满足条件时调用：
```text
parallel_vacuum_init(rel, indrels, nindexes, nworkers, vac_work_mem, elevel, bstrategy)
```
如果返回非 NULL：
```text
vacrel->pvs = pvs
vacrel->dead_items = parallel_vacuum_get_dead_items(...)
```
否则 fallback 到 local `TidStore`。
### 6.4 计算 worker 数
`parallel_vacuum_compute_workers()` 先选择 GUC 上限。
```text
manual VACUUM:
  max_parallel_maintenance_workers
autovacuum:
  autovacuum_max_parallel_workers
```
然后逐个 index 过滤。
条件是：
```text
amparallelvacuumoptions != VACUUM_OPTION_NO_PARALLEL
RelationGetNumberOfBlocks(indrel) >= min_parallel_index_scan_size
```
源码统计两个维度：
```text
nindexes_parallel_bulkdel
nindexes_parallel_cleanup
```
并行 index 数取两者最大值。
然后减一。
减一是因为 leader 也处理一个 index task。
再套用户请求和 GUC 上限。
所以：
```text
PARALLEL 8
```
不表示一定启动 8 个 worker。
如果只有 3 个 eligible indexes，最多只需要 2 个 workers。
### 6.5 初始化 DSM / DSA
`parallel_vacuum_init()` 主链路：
```text
EnterParallelMode()
CreateParallelContext("postgres", "parallel_vacuum_main", parallel_workers)
shm_toc_estimate_chunk(PVIndStats[])
shm_toc_estimate_chunk(PVShared)
shm_toc_estimate_chunk(BufferUsage[])
shm_toc_estimate_chunk(WalUsage[])
shm_toc_estimate_chunk(query text)
InitializeParallelDSM()
shm_toc_allocate(PVIndStats[])
shm_toc_allocate(PVShared)
TidStoreCreateShared()
shm_toc_allocate(BufferUsage[] / WalUsage[])
shm_toc_insert(...)
```
DSM keys：
```text
PARALLEL_VACUUM_KEY_SHARED
PARALLEL_VACUUM_KEY_QUERY_TEXT
PARALLEL_VACUUM_KEY_BUFFER_USAGE
PARALLEL_VACUUM_KEY_WAL_USAGE
PARALLEL_VACUUM_KEY_INDEX_STATS
```
shared `TidStore` 在 DSA area 中。
`PVShared` 只保存：
```text
dead_items_dsa_handle
dead_items_handle
```
worker 用这两个 handle attach。
如果 leader 是 autovacuum worker，`parallel_vacuum_init()` 还初始化 shared cost params 和 generation counter。
### 6.6 leader heap scan 写 dead TID
`lazy_scan_heap()` 扫 heap。
页面处理可能调用：
```text
lazy_scan_prune()
lazy_scan_noprune()
dead_items_add()
```
`dead_items_add()`：
```text
TidStoreSetBlockOffsets(dead_items, blkno, offsets, num_offsets)
dead_items_info->num_items += num_offsets
```
同时更新 progress：
```text
PROGRESS_VACUUM_NUM_DEAD_ITEM_IDS
PROGRESS_VACUUM_DEAD_TUPLE_BYTES
```
parallel 情况下，写入的 `TidStore` 是 shared。
但写入者仍只有 leader。
当 memory usage 超过 `max_bytes` 时，leader 暂停 heap scan。
它释放 visibility map buffer pin。
然后调用：
```text
lazy_vacuum(vacrel)
```
heap scan 结束后，如果还有 dead items，也调用同一函数。
### 6.7 `lazy_vacuum()` 保持 heap/index 不变量
`lazy_vacuum()` 先考虑 bypass optimization。
如果本轮决定不做 index vacuuming，它不会做 heap second pass。
如果执行 index vacuuming：
```text
lazy_vacuum_all_indexes()
lazy_vacuum_heap_rel()
dead_items_reset()
```
`lazy_vacuum_heap_rel()` 把同一批 `dead_items` 对应的 heap line pointer 从 `LP_DEAD` 标成 `LP_UNUSED`。
这一步必须晚于所有 index 完成 bulk delete。
parallel worker 不执行这一步。
### 6.8 parallel index bulk delete
serial path：
```text
for each index:
  lazy_vacuum_one_index()
    -> vac_bulkdel_one_index()
       -> index_bulk_delete()
```
parallel path：
```text
parallel_vacuum_bulkdel_all_indexes(
    pvs,
    old_live_tuples,
    num_index_scans,
    &worker_usage.vacuum)
```
bulkdelete phase 中：
```text
PVShared.reltuples = old_live_tuples
PVShared.estimated_count = true
```
`vac_bulkdel_one_index()` 的 callback 是：
```text
vac_tid_reaped()
  -> TidStoreIsMember(dead_items, itemptr)
```
worker 删除的是 shared `dead_items` 中的 TID。
worker 不重新判断 tuple visibility。
### 6.9 parallel index cleanup
final cleanup 入口在 `lazy_cleanup_all_indexes()`。
serial path：
```text
for each index:
  lazy_cleanup_one_index()
    -> vac_cleanup_one_index()
       -> index_vacuum_cleanup()
```
parallel path：
```text
parallel_vacuum_cleanup_all_indexes(
    pvs,
    reltuples,
    num_index_scans,
    estimated_count,
    &worker_usage.cleanup)
```
cleanup 的拆分依据是：
```text
AM flag
index size
num_index_scans
```
`VACUUM_OPTION_PARALLEL_COND_CLEANUP` 的关键判断：
```text
if num_index_scans > 0:
  conditional cleanup 不再 worker-safe
```
因为这类 AM 在 bulkdelete 已经扫描过 index 后，cleanup 通常不需要再做完整扫描。
如果没有做过 bulkdelete，cleanup-only scan 可能仍然昂贵，可以并行。
这就是 index cleanup 的实际拆分：
```text
leader 根据 AM flags 和本次 VACUUM 历史给每个 index 标记 cleanup task；
safe index 可由 worker 或 leader 抢；
unsafe index 由 leader 处理；
每个 index cleanup callback 只执行一次；
结果写回该 index 的 PVIndStats。
```
### 6.10 worker 启动、leader 参与与 task 分发
`parallel_vacuum_process_all_indexes()` 为每个 index 写入 `PVIndStats.status` 和 `parallel_workers_can_process`。
bulkdelete phase 使用 `NEED_BULKDELETE`。
cleanup phase 使用 `NEED_CLEANUP`。
后续 phase 或后续 batch 会复用 DSM，并通过 `ReinitializeParallelWorkers()` 与 `LaunchParallelWorkers()` 启动 worker。
leader 不是纯 coordinator。
它先用 `parallel_vacuum_process_unsafe_indexes()` 处理小 index 或 AM 不允许 worker 处理的 index。
然后 leader 自己也加入 `parallel_vacuum_process_safe_indexes()`。
safe task 通过原子 `idx` 分发：
```text
idx = pg_atomic_fetch_add_u32(&shared->idx, 1)
if idx >= nindexes:
  break
if !indstats[idx].parallel_workers_can_process:
  continue
parallel_vacuum_process_one_index(...)
```
每个 index slot 只有一个 `idx` 值。
所以同一 phase 中，一个 index 只会被一个进程处理。
`PVIndStats.status` 更新不需要 per-index lock。
writer 唯一。
如果没有 worker 成功启动，leader 仍会处理所有任务。
这就是 worker shortage 的正常 fallback。
### 6.11 单个 index task 与 stats handoff
`parallel_vacuum_process_one_index()` 构造：
```text
IndexVacuumInfo ivinfo
```
核心字段：
```text
index = indrel
heaprel = pvs->heaprel
analyze_only = false
report_progress = false
estimated_count = shared->estimated_count
num_heap_tuples = shared->reltuples
strategy = pvs->bstrategy
```
bulkdelete：
```text
vac_bulkdel_one_index(&ivinfo, istat, pvs->dead_items, &shared->dead_items_info)
```
cleanup：
```text
vac_cleanup_one_index(&ivinfo, istat)
```
完成后：
```text
PVIndStats.status = COMPLETED
pgstat_progress_parallel_incr_param(PROGRESS_VACUUM_INDEXES_PROCESSED, 1)
```
如果 DSM 中已有 stats：
```text
istat = &PVIndStats.istat
```
如果本次 AM 返回了第一份 stats：
```text
memcpy(&PVIndStats.istat, istat_res, sizeof(IndexBulkDeleteResult))
PVIndStats.istat_updated = true
pfree(istat_res)
```
这一步的意义：
```text
AM 的跨调用统计不能留在某个 worker 的私有内存里。
```
### 6.12 phase barrier 与结束 parallel mode
leader 等待 worker：
```text
WaitForParallelWorkersToFinish()
```
然后汇总：
```text
InstrAccumParallelQuery(buffer_usage, wal_usage)
```
最后检查所有 index：
```text
status must be COMPLETED
status = INITIAL
```
如果某个 index 没完成，直接 ERROR。
这阻止未完成的 index phase 继续进入 heap cleanup。
`dead_items_cleanup()` 调用：
```text
parallel_vacuum_end(pvs, vacrel->indstats)
```
`parallel_vacuum_end()` 做：
```text
copy PVIndStats.istat into leader-local indstats
TidStoreDestroy(shared dead_items)
DestroyParallelContext()
ExitParallelMode()
```
之后 leader 才能继续：
```text
update_relstats_all_indexes()
vac_update_relstats()
pgstat_report_vacuum()
```
这解释了哪些状态必须由 leader 汇总。
## 7. 生命周期 / ownership / cleanup
谁创建：
```text
leader 创建 LVRelState。
leader 创建 ParallelContext。
leader 初始化 DSM toc。
leader 创建 PVShared 和 PVIndStats[]。
leader 创建 DSA-backed shared TidStore。
```
谁持有：
```text
ParallelContext:
  leader。
DSM segment:
  ParallelContext 管理。
shared TidStore:
  leader 创建，worker attach。
PVIndStats:
  DSM 中间态，leader 最终复制。
```
worker 做什么：
```text
lookup PVShared
reopen heap relation
vac_open_indexes()
attach shared TidStore
construct worker-local ParallelVacuumState
process safe index tasks
write BufferUsage / WalUsage
detach and close relations
```
worker 不做什么：
```text
不持有 LVRelState。
不写 cutoffs。
不追加 dead_items。
不执行 lazy_vacuum_heap_rel。
不更新 pg_class relation stats。
```
`dead_items` batch 生命周期：
```text
leader finds LP_DEAD
  -> dead_items_add()
  -> index bulkdelete consumes it
  -> leader marks LP_DEAD as LP_UNUSED
  -> dead_items_reset()
```
parallel reset 会：
```text
TidStoreDestroy(old shared TidStore)
TidStoreCreateShared(max_bytes)
update PVShared handles
dead_items_info.num_items = 0
```
index stats 生命周期：
```text
first callback returns local stats
  -> copy to DSM PVIndStats.istat
  -> later callbacks use DSM stats
  -> parallel_vacuum_end copies stats to leader local indstats
```
autovacuum cost params 生命周期：
```text
leader initializes PVSharedCostParams
leader increments generation when params change
worker notices generation mismatch in vacuum_delay_point()
worker refreshes local cost params
DSM detach callback clears leader static pointer
```
## 8. 正确性机制层次
visibility 层：
```text
leader 计算 cutoffs。
leader 扫 heap。
leader 产生 dead_items。
worker 不重新判断 MVCC visibility。
```
heap/index 顺序层：
```text
all indexes finished bulkdelete
  -> leader may run heap second pass
```
这个顺序保护：
```text
no live index entry points to LP_UNUSED heap line pointer
```
AM contract 层：
```text
amparallelvacuumoptions declares worker-safe phases。
core 不猜 AM 内部是否安全。
```
DSM/DSA 层：
```text
Relation pointers 不跨进程共享。
worker 用 relid 重新 open。
TidStore 用 DSA handle attach。
IndexBulkDeleteResult 内容放入 DSM。
```
cost delay 层：
```text
VacuumSharedCostBalance 聚合多个参与者的 I/O cost。
VacuumActiveNWorkers 用于按活跃 worker 数计算 sleep 阈值。
```
progress 层：
```text
worker 递增 parallel progress counter。
leader 展示聚合 phase。
progress 不是 per-index trace。
```
## 9. 异常路径 / fallback
并行度为 0。
常见原因：
```text
standalone backend
GUC 上限为 0
PARALLEL 0
只有一个 index
index 太小
AM 不支持 parallel vacuum
index_cleanup off
```
结果：
```text
parallel_vacuum_init returns NULL
dead_items uses local TidStore
serial VACUUM continues
```
worker 启动少于 planned。
结果：
```text
leader still processes remaining tasks
worker_usage.nlaunched records actual count
```
小 index 或 unsafe index。
结果：
```text
parallel_workers_can_process = false
leader handles it in parallel_vacuum_process_unsafe_indexes()
```
temporary table。
结果：
```text
parallel vacuum disabled
explicit PARALLEL request gets warning
```
failsafe 触发。
parallel path 在 bulkdelete 前 precheck，在 bulkdelete 后 postcheck。
如果触发：
```text
lazy_vacuum_all_indexes returns false
lazy_vacuum_heap_rel is skipped
future index/heap vacuuming disabled
wraparound safety takes priority over reclaim
```
worker ERROR。
worker 安装：
```text
parallel_vacuum_error_callback()
```
context 会指出：
```text
while vacuuming index ...
while cleaning up index ...
```
DSM detach cleanup。
autovacuum leader 的 shared cost pointer 指向 DSM。
ERROR 路径下 DSM detach callback 会清空它。
这避免后续误用已经 detach 的 shared memory。
index task 未完成。
leader 在 phase barrier 检查所有 `PVIndStats.status`。
如果不是 `COMPLETED`，ERROR。
这是进入 heap cleanup 前的安全门。
## 10. 成本、资源与跨模块传播
并行度随 index task 数扩张。
不随 heap block 数扩张。
当前实现：
```text
one index in one phase -> one process
```
单个巨大 index 不能被多个 parallel vacuum workers 分片。
`min_parallel_index_scan_size` 是启动成本过滤器。
小 index 由 leader 串行处理。
这避免 worker 启动成本超过 index scan 收益。
shared `TidStore` 不因为 parallel 而获得更多容量。
`max_bytes` 仍来自 `maintenance_work_mem` 或 `autovacuum_work_mem`。
低内存下仍可能多轮：
```text
heap scan segment
  -> scan all indexes
  -> heap second pass
  -> reset dead_items
```
parallel 只能缩短每轮 index phase wall-clock time。
它不能消除多轮 index scan amplification。
worker memory 需要限制。
若 AM 使用 maintenance work mem，源码设置：
```text
maintenance_work_mem_worker =
  vac_work_mem / Min(parallel_workers, nindexes_mwm)
```
避免总内存简单乘以 worker 数。
worker lifecycle 也有成本。
worker 在每个 phase 前启动，在 phase 结束后退出。
后续 phase 复用 DSM，但 worker 仍要 launch / attach / close。
跨模块边界：
```text
parallel infrastructure:
  ParallelContext, DSM, shm_toc, worker launch。
DSA / TidStore:
  shared dead TID membership set。
index AM:
  ambulkdelete / amvacuumcleanup callback。
pgstat / progress:
  phase 和 counter 聚合。
autovacuum:
  leader 可能是 autovacuum worker，并传播 cost params。
```
## 11. 观测与诊断
`pg_stat_progress_vacuum` 能看：
```sql
SELECT pid, phase,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       indexes_total,
       indexes_processed,
       max_dead_tuple_bytes,
       dead_tuple_bytes,
       num_dead_item_ids,
       delay_time,
       mode,
       started_by
FROM pg_stat_progress_vacuum;
```
它能回答：
```text
当前是不是 index phase。
当前 phase 有多少 index。
已经处理多少 index。
dead_items 有多少 TID。
是否进入 failsafe。
是否由 autovacuum 启动。
```
它不能回答：
```text
哪个 worker 处理哪个 index。
哪个 index 被判定 unsafe。
当前 shared->idx 到多少。
PVIndStats.status 是什么。
conditional cleanup 是否被跳过。
```
`VACUUM VERBOSE` 能看 planned / launched。
重点对照：
```text
planned > launched:
  worker pool 或 runtime launch 不足。
planned = 0:
  大概率没有足够 eligible index。
```
`pg_stat_activity` 能看 leader / worker 关系：
```sql
SELECT pid, leader_pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE leader_pid IS NOT NULL
   OR query LIKE 'VACUUM%';
```
但它不显示 index task。
断点建议：
```text
parallel_vacuum_compute_workers
parallel_vacuum_init
parallel_vacuum_process_all_indexes
parallel_vacuum_process_one_index
parallel_vacuum_main
parallel_vacuum_end
```
重点观察：
```text
pvs->nindexes_parallel_bulkdel
pvs->nindexes_parallel_cleanup
pvs->nindexes_parallel_condcleanup
pvs->shared->idx
pvs->indstats[i].status
pvs->indstats[i].parallel_workers_can_process
pvs->shared->dead_items_info.num_items
```
CPU 诊断用 `perf`。
B-tree 常见热点在 `btvacuumscan()`、page walk 和 callback membership check 附近。
不同 AM、dead TID 密度、cache 状态和 storage latency 会改变热点。
不要把 progress counter 当完整因果链。
## 12. 常见误区
误区一：
```text
parallel vacuum 会并行 heap scan。
```
不对。
当前源码并行的是 index bulkdelete 和 index cleanup。
误区二：
```text
PARALLEL n 一定启动 n 个 worker。
```
不对。
`n` 是上限。
实际数受 eligible index、GUC、AM flag、index size 和 worker pool 影响。
误区三：
```text
一个 index 会被多个 worker 分片处理。
```
不对。
当前实现中，一个 index 在一个 phase 中只由一个进程处理。
误区四：
```text
worker 可以最终更新 index relstats。
```
不对。
worker 只写 DSM stats。
leader 退出 parallel mode 后更新 relstats。
误区五：
```text
index_cleanup off 只跳过 final cleanup。
```
不对。
`heap_vacuum_rel()` 中它会关闭 index vacuuming 和 cleanup。
也会阻止 parallel vacuum 初始化。
误区六：
```text
pg_stat_progress_vacuum 可以看完整调度。
```
不对。
它是 phase-level 聚合。
DSM task 状态需要源码断点或日志推断。
## 13. 课堂实验
### 实验 1：并行度来自 index 数
```sql
SET max_parallel_maintenance_workers = 8;
SET min_parallel_index_scan_size = 0;
DROP TABLE IF EXISTS pv_degree_demo;
CREATE TABLE pv_degree_demo(id int, a int, b int, c int, d int)
WITH (autovacuum_enabled = off);
INSERT INTO pv_degree_demo
SELECT g, g % 10, g % 100, g % 1000, g % 7
FROM generate_series(1, 500000) AS g;
CREATE INDEX pv_degree_demo_a ON pv_degree_demo(a);
CREATE INDEX pv_degree_demo_b ON pv_degree_demo(b);
CREATE INDEX pv_degree_demo_c ON pv_degree_demo(c);
CREATE INDEX pv_degree_demo_d ON pv_degree_demo(d);
UPDATE pv_degree_demo SET a = a WHERE id % 2 = 0;
VACUUM (VERBOSE, PARALLEL 8) pv_degree_demo;
```
验证：
```text
planned workers 不超过 eligible indexes - 1。
leader 也会处理一个 index task。
```
源码入口：
```text
parallel_vacuum_compute_workers()
parallel_vacuum_process_all_indexes()
```
### 实验 2：小 index 由 leader 处理
```sql
SET max_parallel_maintenance_workers = 4;
SET min_parallel_index_scan_size = '128kB';
DROP TABLE IF EXISTS pv_small_index_demo;
CREATE TABLE pv_small_index_demo(a int, b int)
WITH (autovacuum_enabled = off);
INSERT INTO pv_small_index_demo
SELECT g, g % 10
FROM generate_series(1, 10000) AS g;
CREATE INDEX pv_big_1 ON pv_small_index_demo(a);
CREATE INDEX pv_big_2 ON pv_small_index_demo(a + 1);
CREATE INDEX pv_small_const ON pv_small_index_demo((1));
DELETE FROM pv_small_index_demo WHERE a % 2 = 0;
VACUUM (VERBOSE, PARALLEL 4, INDEX_CLEANUP ON) pv_small_index_demo;
```
验证：
```text
parallel vacuum 可以启动。
小 index 不会被 worker 抢。
leader 仍处理它。
```
源码入口：
```text
will_parallel_vacuum[i]
parallel_workers_can_process
parallel_vacuum_process_unsafe_indexes()
```
### 实验 3：cleanup-only 路径
在没有大量 dead TID 的情况下执行：
```sql
SET max_parallel_maintenance_workers = 4;
SET min_parallel_index_scan_size = 0;
VACUUM (VERBOSE, PARALLEL 4, INDEX_CLEANUP ON) pv_index_cleanup_demo;
```
观察是否进入 parallel index cleanup。
回到源码：
```text
parallel_vacuum_cleanup_all_indexes()
parallel_vacuum_index_is_parallel_safe(vacuum=false)
VACUUM_OPTION_PARALLEL_COND_CLEANUP
```
讨论：
```text
为什么 num_index_scans > 0 会让 conditional cleanup 退出 worker-safe 集合？
```
## 14. 讨论题
1. 为什么 parallel vacuum 选择 index-level task parallelism，而不是把 heap block range 分给多个 worker？
2. 如果某个 index AM 的 stats 包含私有指针，它还能声明 parallel vacuum 吗？
3. 为什么 leader 要先处理 unsafe indexes，再加入 safe index task pool？
4. `VACUUM_OPTION_PARALLEL_COND_CLEANUP` 为什么依赖 `num_index_scans`？
5. 如果一个 worker ERROR，leader 能否继续把 heap `LP_DEAD` 改成 `LP_UNUSED`？
6. 为什么 `pg_stat_progress_vacuum` 不能当作完整 parallel task trace？
## 15. 本节小结
本节主问题是：
```text
parallel vacuum 如何拆分 index cleanup，哪些状态仍必须由 leader 汇总？
```
答案：
```text
PostgreSQL 把 index bulkdelete 和 index cleanup 拆成每个 index 一个 task；
leader 与 parallel workers 通过 DSM 中的 PVIndStats.status 和原子 idx 抢任务；
但 heap visibility、dead TID 生成、heap line pointer 回收、phase barrier、
IndexBulkDeleteResult 最终拷贝、relstats/pgstat 更新和资源 cleanup 都由 leader 汇总。
```
核心状态：
```text
LVRelState:
  leader-local，一次 relation VACUUM 的完整生命周期。
PVShared:
  DSM phase 参数、shared TidStore handle、cost balance。
PVIndStats:
  DSM 中每个 index 的 task status 和 stats handoff。
TidStore:
  DSA-backed shared membership set，worker 只读。
```
正确性边界：
```text
worker 不决定 visibility。
worker 不追加 dead_items。
worker 不执行 heap second pass。
worker 不最终更新 pg_class。
所有 index 对同一批 dead_items 完成 bulkdelete 后，leader 才能回收 heap line pointer。
```
异常收尾：
```text
worker 不可用时 leader 串行处理。
小 index 或 unsafe index 由 leader 处理。
failsafe 触发时优先 wraparound 安全，跳过后续 index/heap cleanup。
ERROR context 指向具体 index phase。
DSM detach callback 清理 autovacuum cost pointer。
```
可观测状态：
```text
pg_stat_progress_vacuum:
  phase、indexes_total、indexes_processed、dead tuple bytes、delay time。
VACUUM VERBOSE:
  planned / launched workers。
pg_stat_activity:
  leader / parallel worker 关系。
```
不可直接观测状态：
```text
PVIndStats.status。
parallel_workers_can_process。
shared->idx 分配过程。
每个 worker 处理的具体 index。
```
可迁移规律：
```text
当一个阶段可以并行，但 correctness horizon 和生命周期不能并行拥有时，
PostgreSQL 倾向于只共享最小任务状态，
让 worker 执行可分割 callback，
再由 leader 通过 barrier 和 stats handoff 把结果收回主状态机。
```
