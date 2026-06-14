# PostgreSQL autovacuum ANALYZE 与 planner 统计信息刷新
## 课程定位
前置知识：已经理解 autovacuum launcher / worker 调度、VACUUM heap / index cleanup、pgstat 累计统计、系统 catalog、relcache / syscache，以及 planner 基本行数估算。
本节唯一主问题：
```text
autovacuum analyze 如何刷新 planner 统计信息，
为什么统计 stale 会跨到 optimizer 目录的问题域？
```
核心矛盾：
```text
后台统计刷新必须便宜、近似、可跳过、可失败后重试；
optimizer 又必须把这些近似统计当作下一次 plan search 的输入。
```
学完后应能判断：
```text
autoanalyze 由哪个 counter 触发；
ANALYZE 写哪些 catalog 和 pgstat 状态；
sample、pg_statistic、pg_statistic_ext_data、pg_class.reltuples 如何连接；
optimizer 在 plancat.c / selfuncs.c 中如何消费这些状态；
为什么 stale stats 会表现为 row estimate、join order、access path 和 cost model 问题；
哪些状态能从 SQL 看到，哪些只能从源码和断点推断。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
第 39 节回答 autovacuum launcher / worker 如何选择 database 和 relation。
本节从 worker 已经选中 relation 开始，追问它执行 autoanalyze 后到底刷新了什么。
第 28 节讲 VACUUM 的 heap scan、index cleanup 和 relation-level cleanup。
本节不重复 tuple dead 判定、index TID 删除或 visibility map。
这里的主链路是：
```text
DML 改变 table
  -> pgstat.mod_since_analyze 增长
  -> autovacuum worker 判断 analyze threshold
  -> analyze.c 抽样并计算统计
  -> pg_statistic / pg_statistic_ext_data / pg_class 被更新
  -> 后续 optimizer 读取这些统计并决定 plan
```
这条链路跨四个问题域。
`postmaster/autovacuum.c` 负责决定是否跑。
`commands/analyze.c` 负责采样和写统计。
`statistics/extended_stats.c` 负责多列、表达式和 MCV extended stats。
`optimizer/util/plancat.c` 与 `utils/adt/selfuncs.c` 负责读取和解释统计。
本节的 runtime 锚点：
```text
先让一张表的数据分布突变；
在 ANALYZE 前后分别 EXPLAIN；
观察 estimated rows、join order、index/seq scan 选择变化；
再回到 relation_needs_vacanalyze()、do_analyze_rel()、update_attstats()、
BuildRelationExtStatistics()、estimate_rel_size()、examine_variable()。
```
本节要建立的判断框架：
```text
统计刷新是 maintenance 生命周期；
统计消费是 optimizer 生命周期；
stale stats 是两者之间的边界延迟。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
autovacuum worker 用 pgstat.mod_since_analyze 与 analyze threshold 判断 doanalyze；
需要时通过 vacuum(... VACOPT_ANALYZE ...) 进入 analyze_rel()；
ANALYZE 随机抽样 heap rows，计算 per-column stats、index expression stats 和 extended stats；
随后更新 pg_statistic、pg_statistic_ext_data、pg_class relation stats 和 pgstat analyze counters；
后续 planner 在下一次规划时读取这些 catalog 状态。
```
PostgreSQL 没有在每次 INSERT / UPDATE / DELETE 上同步维护 MCV、histogram 和 dependencies。
写路径只累加轻量 pgstat counters。
维护路径周期性采样并刷新 catalog。
planner 路径读取最近一次可见 catalog 统计。
统计缺失时，optimizer 使用默认选择率、唯一索引推断、`relpages/reltuples` 密度或其它 fallback。
这个选择带来四类 stale：
```text
trigger stale:
  mod_since_analyze 尚未超过阈值，autoanalyze 不启动。
sample stale:
  ANALYZE 刚完成，但样本没有捕捉偏斜。
catalog stale:
  pg_statistic / pg_class 仍保存旧分布或旧 row count。
plan stale:
  已生成计划不会因为后台 analyze 立刻变成另一个 plan。
```
optimizer 目录无法直接知道 heap 真实分布已经偏离 catalog 多远。
它只能解释当前 planning snapshot 中能看到的统计。
所以 stale stats 不是单纯的 autovacuum 问题。
它会进入 optimizer 的 search space、selectivity、cost 和 path ranking。
本节的核心 tension 可以压缩为：
```text
低成本异步近似统计
  vs
前台规划对及时、精细、稳定数据分布的需求
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/autovacuum.c` | `relation_needs_vacanalyze()`、`table_recheck_autovac()`、`autovacuum_do_vac_analyze()`。 |
| 2 | `src/include/commands/vacuum.h` | `VacuumParams`、`VACOPT_ANALYZE`、`VacAttrStats` 和 ANALYZE callbacks。 |
| 3 | `src/backend/commands/analyze.c` | `analyze_rel()`、`do_analyze_rel()`、`acquire_sample_rows()`、`update_attstats()`。 |
| 4 | `src/backend/statistics/extended_stats.c` | `BuildRelationExtStatistics()`、`statext_store()`。 |
| 5 | `src/include/catalog/pg_statistic.h` | `pg_statistic` tuple、五个 stats slot、MCV / histogram / correlation kind。 |
| 6 | `src/include/catalog/pg_statistic_ext.h` | extended statistics definition。 |
| 7 | `src/include/catalog/pg_statistic_ext_data.h` | extended statistics computed data。 |
| 8 | `src/include/statistics/statistics.h` | `MVNDistinct`、`MVDependencies`、`MCVList` 和 planner API。 |
| 9 | `src/backend/optimizer/util/plancat.c` | `get_relation_info()`、`estimate_rel_size()`、`get_relation_statistics()`。 |
| 10 | `src/backend/utils/adt/selfuncs.c` | `examine_variable()`、`scalarineqsel()`、`get_variable_numdistinct()`。 |
| 11 | `src/backend/utils/activity/pgstat_relation.c` | `pgstat_report_analyze()`。 |
| 12 | `src/backend/commands/vacuum.c` | `vacuum()` 与 `vac_update_relstats()`。 |
推荐阅读顺序：
```text
autovacuum threshold
  -> VacuumParams
  -> analyze_rel lock / relkind checks
  -> sample acquisition
  -> per-column stats
  -> pg_statistic update
  -> extended stats update
  -> pg_class relstats update
  -> pgstat report
  -> optimizer read path
```
不要从 `selfuncs.c` 全部 selectivity functions 开始。
先抓住状态流，再看某个谓词怎样读取这些状态。
也不要把 `pg_statistic_ext` 和 `pg_statistic_ext_data` 合并理解。
前者是定义。
后者是 `ANALYZE` 生成的数据。
## 4. 关键数据结构与状态
### 4.1 pgstat relation entry
autoanalyze 的触发输入是 pgstat relation entry，不是 `pg_statistic`。
关键字段：
| 字段 | 语义 |
| --- | --- |
| `mod_since_analyze` | 自上次完整 analyze 后的 insert/update/delete 近似计数。 |
| `live_tuples` | 最近维护报告的 live tuple 估算。 |
| `dead_tuples` | 最近维护报告的 dead tuple 估算。 |
| `last_autoanalyze_time` | autovacuum worker 完成 autoanalyze 的时间。 |
| `autoanalyze_count` | autoanalyze 次数。 |
| `total_autoanalyze_time` | autoanalyze 累计耗时。 |
`mod_since_analyze` 不描述哪个值变多了。
它只回答：
```text
这张 relation 自上次完整 analyze 后是否已经发生足够多变更？
```
因此它适合触发后台维护，不适合直接给 optimizer 做选择率估算。
### 4.2 `AutoVacuumScores`
`relation_needs_vacanalyze()` 会计算 analyze score：
```text
anlthresh = autovacuum_analyze_threshold
          + autovacuum_analyze_scale_factor * reltuples
scores.anl = mod_since_analyze / Max(anlthresh, 1)
scores.anl *= autovacuum_analyze_score_weight
```
`scores.max` 用于候选表排序。
`doanalyze` 才表示是否执行。
两者不要混淆。
当前源码还排除：
```text
pg_statistic 自身不 analyze；
TOAST table 不 analyze；
autovacuum inactive 时普通 threshold 不触发 analyze。
```
### 4.3 `VacuumParams`
autovacuum worker 不直接调用 `analyze_rel()`。
它构造 `autovac_table`，其中 `at_params.options` 可能包含：
| flag | 语义 |
| --- | --- |
| `VACOPT_ANALYZE` | 执行 ANALYZE。 |
| `VACOPT_VACUUM` | 执行 VACUUM。 |
所以 activity string 可能是：
```text
autovacuum: ANALYZE schema.table
autovacuum: VACUUM ANALYZE schema.table
```
autoanalyze 是 autovacuum worker 的执行模式，不是单独的后台进程类型。
### 4.4 `VacAttrStats`
`VacAttrStats` 是 ANALYZE 为每个 attribute 构造的运行态。
它不是 catalog tuple。
关键字段组：
| 字段组 | 语义 |
| --- | --- |
| `attstattarget` / `minrows` | 决定采样目标和统计粒度。 |
| `compute_stats` | 类型相关统计计算函数。 |
| `rows` / `tupDesc` | sample rows 和 tuple descriptor。 |
| `stanullfrac` / `stawidth` / `stadistinct` | 基础统计。 |
| `stakind[]` / `staop[]` / `stacoll[]` | slot 的 kind、operator、collation。 |
| `stanumbers[]` / `stavalues[]` | slot 的频率和值。 |
| `stats_valid` | 是否生成可写统计。 |
单个字段没有完整语义。
`stavalues1` 只有结合 `stakind1` 才能知道是 MCV、histogram 还是其它 kind。
### 4.5 `pg_statistic`
`pg_statistic` 的唯一键是：
```text
(starelid, staattnum, stainherit)
```
这说明同一 attribute 可以有普通统计和 inherited 统计两份。
核心字段：
| 字段 | 语义 |
| --- | --- |
| `starelid` | relation 或 index OID。 |
| `staattnum` | column 或 index expression slot。 |
| `stainherit` | 是否包含继承 children。 |
| `stanullfrac` | NULL 占比。 |
| `stawidth` | 非 NULL 平均宽度。 |
| `stadistinct` | distinct 数；负数表示相对 row count 的比例。 |
| `stakind1..5` | slot kind。 |
| `stanumbers1..5` | slot numbers。 |
| `stavalues1..5` | slot values。 |
`pg_statistic.h` 明确要求读取者不要假设某种 kind 固定在某个 slot。
正确读取方式是搜索 `stakind`，常见 helper 是 `get_attstatsslot()`。
常见 kind：
| kind | 用途 |
| --- | --- |
| `STATISTIC_KIND_MCV` | equality、IN、join skew。 |
| `STATISTIC_KIND_HISTOGRAM` | range / inequality selectivity。 |
| `STATISTIC_KIND_CORRELATION` | physical order correlation，影响 index scan cost。 |
| `STATISTIC_KIND_MCELEM` | array / tsvector element frequency。 |
| `STATISTIC_KIND_BOUNDS_HISTOGRAM` | range bounds selectivity。 |
MCV 与 histogram 不是独立相加的两个事实。
histogram 通常描述去掉 MCV 后的剩余非 NULL population。
所以 MCV stale 会同时改变 histogram 的解释边界。
### 4.6 extended statistics catalogs
extended stats 分两层：
```text
pg_statistic_ext:
  statistics object definition。
pg_statistic_ext_data:
  ANALYZE 计算出的 ndistinct、dependencies、MCV 和 expression stats。
```
`CREATE STATISTICS` 只创建 definition。
后续 `ANALYZE` 才会填充 data。
planner 只使用实际 built 的 data。
### 4.7 `pg_class` relation stats
`pg_class` 保存 relation-level planner 输入：
| 字段 | 语义 |
| --- | --- |
| `relpages` | page 数估算。 |
| `reltuples` | live tuple 数估算。 |
| `relallvisible` | all-visible page 数。 |
| `relallfrozen` | all-frozen page 数。 |
| `relhasindex` | 是否有 index。 |
`vac_update_relstats()` 负责更新这些字段。
源码注释明确说：它用 in-place update 覆写 `pg_class` tuple，违反普通事务语义。
原因是这些 relation stats 本身是非事务性的近似统计；如果用普通 update，VACUUM `pg_class` 本身会被大量 dead catalog tuples 干扰。
这一点必须和 `update_attstats()` 对 `pg_statistic` 的普通 catalog insert/update 区分开。
### 4.8 optimizer 内存状态
planner 会把 catalog stats 转成 planning-local 状态：
| 结构 | 来源 | 作用 |
| --- | --- | --- |
| `RelOptInfo.pages` | `estimate_rel_size()` | base relation page count。 |
| `RelOptInfo.tuples` | `estimate_rel_size()` | base relation row count。 |
| `RelOptInfo.allvisfrac` | `estimate_rel_size()` | index-only scan 成本。 |
| `RelOptInfo.statlist` | `get_relation_statistics()` | extended stats 描述。 |
| `VariableStatData.statsTuple` | `examine_variable()` | per-column 或 expression stats tuple。 |
| `StatisticExtInfo` | `plancat.c` | extended stats 在 planner 内的对象。 |
这些状态只属于一次 planning。
后台 autoanalyze 完成后，不会修改已经构造出的 `RelOptInfo` 或已经开始执行的 plan。
## 5. runtime 现象：统计 stale 如何变成 plan 问题
准备一张数据分布会突变的表：
```sql
DROP TABLE IF EXISTS analyze_stale_demo;
CREATE TABLE analyze_stale_demo(
    id bigint generated always as identity,
    tenant_id int,
    status int,
    payload text
);
CREATE INDEX analyze_stale_demo_tenant_status_idx
    ON analyze_stale_demo(tenant_id, status);
INSERT INTO analyze_stale_demo(tenant_id, status, payload)
SELECT g % 1000, 0, repeat('x', 40)
FROM generate_series(1, 500000) AS g;
ANALYZE analyze_stale_demo;
```
先看估算：
```sql
EXPLAIN
SELECT * FROM analyze_stale_demo
WHERE tenant_id = 42 AND status = 1;
```
制造分布突变，但先不要 ANALYZE：
```sql
UPDATE analyze_stale_demo
SET status = 1
WHERE tenant_id = 42;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM analyze_stale_demo
WHERE tenant_id = 42 AND status = 1;
```
可能看到 estimated rows 远小于 actual rows。
再执行：
```sql
ANALYZE analyze_stale_demo;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM analyze_stale_demo
WHERE tenant_id = 42 AND status = 1;
```
观察 `pg_stat_all_tables`：
```sql
SELECT n_mod_since_analyze, last_autoanalyze, autoanalyze_count
FROM pg_stat_all_tables
WHERE relname = 'analyze_stale_demo';
```
观察 per-column stats：
```sql
SELECT attname, null_frac, n_distinct,
       most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats
WHERE tablename = 'analyze_stale_demo'
ORDER BY attname;
```
如果组合列相关性强，再创建 extended stats：
```sql
CREATE STATISTICS analyze_stale_demo_dep (dependencies, mcv)
ON tenant_id, status
FROM analyze_stale_demo;
ANALYZE analyze_stale_demo;
```
这个实验要建立三段映射：
```text
pg_stat_all_tables:
  autoanalyze 是否发生。
pg_stats / pg_stats_ext:
  catalog 中的分布模型是什么。
EXPLAIN:
  optimizer 如何把分布模型转成 estimated rows 和 path。
```
`EXPLAIN ANALYZE` 的 actual rows 是执行后观测，不是 planner 输入。
本节解释的是 planner 在执行前为什么只能相信 catalog 里的近似统计。
## 6. 主流程源码 walkthrough
### 6.1 worker 判断是否需要 analyze
`do_autovacuum()` 扫描 `pg_class`。
当前源码第一遍只处理：
```text
RELKIND_RELATION
RELKIND_MATVIEW
```
它跳过其它 backend 的 temp table。
然后读取 reloptions 并调用：
```text
relation_needs_vacanalyze(...)
```
analyze 分支的核心输入：
```text
classForm->reltuples
tabentry->mod_since_analyze
reloptions analyze_threshold / analyze_scale_factor
GUC autovacuum_anl_thresh / autovacuum_anl_scale
```
核心判断：
```text
anltuples = tabentry->mod_since_analyze;
anlthresh = anl_base_thresh + anl_scale_factor * reltuples;
if (av_enabled && anltuples > anlthresh)
    doanalyze = true;
```
这一步只产生候选。
relation 还没有被锁住。
统计还没有刷新。
候选还会在执行前 recheck。
### 6.2 recheck 与 `autovac_table`
worker claim table 后调用 `table_recheck_autovac()`。
它重新读取 `pg_class`、reloptions 和 pgstat。
如果仍然需要 analyze，返回 `autovac_table`。
`autovac_table.at_params.options` 会带上 `VACOPT_ANALYZE`。
如果同表也需要 vacuum，还会带上 `VACOPT_VACUUM`。
recheck 返回 NULL 不是错误。
它说明候选列表已经过期：
```text
别人刚 analyze 过；
relation 被 drop；
reloptions 改了；
pgstat 被 reset；
table claim 发现并发 worker。
```
### 6.3 进入 `vacuum()`
`autovacuum_do_vac_analyze()` 做的事很少：
```text
autovac_report_activity(tab);
创建 Vacuum memory context;
构造 OID-based VacuumRelation;
调用 vacuum(rel_list, &tab->at_params, bstrategy, vac_context, true);
```
从这里开始，manual `ANALYZE` 和 autoanalyze 共享 `analyze.c` 的主要逻辑。
`true` 标识 autovacuum 调用路径。
### 6.4 `analyze_rel()` 打开目标 relation
`analyze_rel()` 用：
```text
vacuum_open_relation(..., ShareUpdateExclusiveLock)
```
打开 relation。
这个锁主要保护：
```text
同一 relation 不被两个 ANALYZE 同时更新 pg_statistic；
ANALYZE 与相关 VACUUM / DDL 边界互斥；
relation 在 stats commit 前不被删除。
```
它不阻塞普通 SELECT。
`analyze_rel()` 会跳过：
```text
权限不足；
其它 backend 的 temp table；
pg_statistic 自身；
不可 analyze 的 relkind。
```
regular table 和 materialized view 使用 `acquire_sample_rows()`。
foreign table 可能走 FDW statistics import 或 FDW analyze callback。
手动 `ANALYZE` 可以处理 partitioned table 的 recursive stats。
当前 autovacuum worker 的 `pg_class` scan 没有把 partitioned table 放入普通候选。
### 6.5 progress start
`analyze_rel()` 调：
```text
pgstat_progress_start_command(PROGRESS_COMMAND_ANALYZE, relid)
```
并设置：
```text
PROGRESS_ANALYZE_STARTED_BY_AUTOVACUUM
PROGRESS_ANALYZE_STARTED_BY_MANUAL
```
这就是 `pg_stat_progress_analyze.started_by` 的来源。
progress 是当前运行态。
ANALYZE 结束后它消失。
### 6.6 `do_analyze_rel()` 建立执行环境
`do_analyze_rel()` 创建 `Analyze` memory context。
它切换到 table owner，并设置 security restricted operation：
```text
SetUserIdAndSecContext(onerel->rd_rel->relowner,
                       SECURITY_RESTRICTED_OPERATION);
RestrictSearchPath();
```
原因是 ANALYZE 可能执行类型 analyze hook、index expression 或 statistics expression。
后台 worker 不能用不受限 search path 执行用户表达式。
### 6.7 选择 columns 与 index expressions
如果用户指定 column list，只处理这些列。
autovacuum 通常没有 column list。
它遍历 attributes，调用：
```text
examine_attribute(onerel, attnum, NULL)
```
返回非 NULL 的 attribute 才会有 `VacAttrStats`。
未处理的 columns 不会被本轮 `update_attstats()` 替换。
如果有 expression index，且本次不是显式 column-list analyze，`do_analyze_rel()` 会为 index expressions 构造 stats。
后续写入目标是 index relation OID：
```text
update_attstats(RelationGetRelid(Irel[ind]), false, ...)
```
所以统计不总是挂在 heap table column 上。
expression index stats 挂在 index relation 上。
### 6.8 决定 sample rows
`targrows` 来自所有 analyzable columns 的 `minrows` 最大值。
extended statistics 也可能提高 sample rows：
```text
minrows = ComputeExtStatisticsRows(onerel, attr_cnt, vacattrstats);
```
最终 `targrows` 至少为 100。
提高 statistics target 会增加：
```text
sample rows；
MCV / histogram entries；
extended stats 计算量；
pg_statistic / pg_statistic_ext_data 大小；
planning 时读取和解释统计的成本。
```
target 应该对计划敏感的列、偏斜列和相关列提高，而不是全局盲目提高。
### 6.9 `acquire_sample_rows()`
普通 heap table 的采样函数是 `acquire_sample_rows()`。
源码注释描述两阶段方法：
```text
stage 1:
  BlockSampler 随机选择 blocks。
stage 2:
  扫描这些 blocks，用 Vitter reservoir sampling 选择 rows。
```
当前主流程：
```text
RelationGetNumberOfBlocks(onerel)
BlockSampler_Init(&bs, totalblocks, targrows, randseed)
table_beginscan_analyze(onerel)
read_stream_begin_relation(READ_STREAM_MAINTENANCE | READ_STREAM_USE_BATCHING)
while table_scan_analyze_next_block(...)
  vacuum_delay_point(true)
  while table_scan_analyze_next_tuple(...)
    更新 liverows / deadrows
    reservoir sample
qsort sample rows by TID
估算 totalrows / totaldeadrows
```
sample rows 按物理位置排序，是为了后续 correlation 估算。
大表不会全表扫描。
freshly analyzed 不等于 perfectly known。
### 6.10 live/dead rows 估算
`acquire_sample_rows()` 返回：
```text
numrows
totalrows
totaldeadrows
```
`totalrows` 和 `totaldeadrows` 通过 sampled blocks 的平均密度外推到全表。
如果 tuple density 在 relation 内部分布极不均匀，估算会偏。
这个偏差会进入：
```text
pg_class.reltuples
pgstat.live_tuples / dead_tuples
后续 autoanalyze threshold
optimizer base row count
```
ANALYZE 的采样误差会反过来影响下一轮维护调度和 planner 估算。
### 6.11 per-column stats 计算
如果 `numrows > 0`，进入 compute stats。
每列调用：
```text
stats->compute_stats(stats, std_fetch_func, numrows, totalrows)
```
常见路径由 `std_typanalyze()` 设置。
结果写回 `VacAttrStats`：
```text
stanullfrac
stawidth
stadistinct
stakind[]
staop[]
stacoll[]
stanumbers[]
stavalues[]
stats_valid
```
如果 column 有 `n_distinct` storage option，源码会覆盖 `stadistinct`。
所以 catalog stats 不只是 sample 的机械输出，也可能受 storage option 控制。
### 6.12 `update_attstats()` 写 `pg_statistic`
`update_attstats()` 打开：
```text
pg_statistic with RowExclusiveLock
```
对每个 `stats_valid` 的 attribute：
```text
构造 pg_statistic tuple；
SearchSysCache3(STATRELATTINH, relid, attnum, inh)；
存在则 CatalogTupleUpdateWithInfo；
不存在则 CatalogTupleInsertWithInfo。
```
边界：
```text
只写本轮处理的 attributes；
只写 stats_valid 的 attributes；
未处理 columns 的旧 rows 保留；
pg_statistic 自身不进入 analyze。
```
这条路径是普通 catalog insert/update，会产生 catalog tuple version 和 index 维护。
### 6.13 extended stats build
`BuildRelationExtStatistics()`：
```text
打开 pg_statistic_ext；
fetch_statentries_for_relation() 找本 relation 的 stats objects；
检查本轮 columns / expressions 是否足够；
计算 effective statistics target；
构造 ndistinct / dependencies / MCV / expression stats；
调用 statext_store()。
```
`statext_store()`：
```text
打开 pg_statistic_ext_data；
RemoveStatisticsDataById(statOid, inh) 删除旧 data；
CatalogTupleInsert() 插入新 data tuple。
```
如果 statistics target 为 0，源码跳过 rebuild，旧值保留。
如果依赖列没有被本轮 analyze 覆盖，manual analyze 会 WARNING。
autovacuum 路径不会为这个情况刷 WARNING。
### 6.14 `vac_update_relstats()` 写 `pg_class`
`do_analyze_rel()` finalize 阶段会：
```text
visibilitymap_count(onerel, &relallvisible, &relallfrozen)
CommandCounterIncrement()
vac_update_relstats(onerel, relpages, totalrows, relallvisible, relallfrozen, ...)
```
index relation 也会被更新：
```text
relpages = RelationGetNumberOfBlocks(index)
reltuples = ceil(index tuple fraction * table totalrows)
```
`vac_update_relstats()` 使用 `systable_inplace_update_*` 覆写 `pg_class` 的固定宽度统计字段。
源码说明这是非事务性统计更新。
如果 transaction 后续 abort，`pg_class.reltuples/relpages` 这类统计不按普通 MVCC 回滚理解。
这也是它能被 VACUUM 和 ANALYZE 共同使用的原因。
### 6.15 `pgstat_report_analyze()`
最后报告 pgstat：
```text
pgstat_report_analyze(onerel, totalrows, totaldeadrows,
                      resetcounter, starttime)
```
它更新：
```text
live_tuples
dead_tuples
mod_since_analyze
last_autoanalyze_time / last_analyze_time
autoanalyze_count / analyze_count
total_autoanalyze_time / total_analyze_time
```
`resetcounter` 的关键条件：
```text
只有分析了所有 columns，才 reset mod_since_analyze。
```
column-list analyze 不清零。
源码还处理一个边界：
```text
ANALYZE 可能运行在已经插入/删除 tuple 的 transaction 内；
报告 live/dead 时要扣除本事务稍后还会提交给 pgstat 的变更，避免 double count。
```
`mod_since_analyze` reset 可能忘掉 analyze 运行期间并发提交的一些 changes。
源码注释说没有好办法估算这些变化。
这是 pgstat 近似性的另一个边界。
### 6.16 optimizer 读取 relation size
后续 query planning 进入 `plancat.c`。
`get_relation_info()` 会调用：
```text
estimate_rel_size(relation, ...)
```
它把 `pg_class` 与当前 physical blocks 结合成：
```text
RelOptInfo.pages
RelOptInfo.tuples
RelOptInfo.allvisfrac
```
对 index relation，`estimate_rel_size()` 读取 `rd_rel->relpages`、`rd_rel->reltuples`、`rd_rel->relallvisible`，并用旧 tuple density 缩放当前 blocks。
所以 stale `reltuples` 会影响 base relation rows、index rows 和后续所有 join cardinality。
### 6.17 optimizer 读取 `pg_statistic`
`selfuncs.c` 中典型入口是：
```text
examine_variable(root, node, varRelid, &vardata)
```
简单 Var 会查 per-column stats。
expression 可能匹配 expression index stats。
extended expression stats 可以从 `pg_statistic_ext_data.stxdexpr` 中抽取一个 `pg_statistic` 形状的 tuple。
选择率函数再读取：
```text
vardata->statsTuple
Form_pg_statistic.stanullfrac
Form_pg_statistic.stadistinct
get_attstatsslot() 返回的 MCV / histogram / correlation
```
`scalarineqsel()` 的分支很典型：
```text
没有 statsTuple:
  使用 DEFAULT_INEQ_SEL 或 CTID 特例。
有 statsTuple:
  计算 MCV 中满足条件的频率；
  用 histogram 估算剩余 population；
  合并 nullfrac、MCV 和 histogram。
```
`get_variable_numdistinct()`：
```text
优先使用 pg_statistic.stadistinct；
若 unique index / DISTINCT / GROUP BY 证明唯一，则覆盖统计；
缺失统计时使用 bool、system column、VALUES RTE 或 DEFAULT_NUM_DISTINCT fallback。
```
这就是 stale stats 进入 optimizer 的具体位置。
旧 MCV、旧 histogram、旧 `stadistinct` 和旧 relation size 都会变成新的 `RelOptInfo` 和 selectivity。
### 6.18 extended stats 进入 optimizer
`plancat.c` 的 `get_relation_statistics()` 读取 relation 的 extended statistics list。
它先读 `pg_statistic_ext` metadata。
再通过 worker 尝试加载 `pg_statistic_ext_data`。
planner-side extended stats 代码会在多列 clauses 上使用：
```text
dependencies
ndistinct
MCV
expression stats
```
关键边界：
```text
有 pg_statistic_ext row:
  只说明 definition 存在。
有 pg_statistic_ext_data:
  才说明 ANALYZE 已经构建过数据。
```
## 7. 生命周期 / ownership / cleanup
pgstat counters 由普通 backend 和维护进程累加。
autovacuum worker 读取它们来做调度。
`pg_statistic` rows 由 `update_attstats()` 创建或替换。
`pg_statistic_ext_data` rows 由 `statext_store()` 删除旧数据后插入新数据。
`pg_class` relation stats 由 `vac_update_relstats()` in-place 覆写。
`VacAttrStats`、sample rows、extended stats build data 都是 backend-local runtime state。
ANALYZE memory context 分层：
```text
autovacuum_do_vac_analyze:
  Vacuum context，供 vacuum() 使用。
do_analyze_rel:
  Analyze context，保存 sample rows、VacAttrStats 和最终统计。
per-column compute:
  Analyze Column context，循环 reset。
extended stats:
  BuildRelationExtStatistics context，每个 object 后 reset。
```
relation ownership：
```text
analyze_rel() 拿 ShareUpdateExclusiveLock；
relation_close(onerel, NoLock) 关闭 relcache reference；
lock 保留到 transaction end。
```
保留 lock 的原因：
```text
避免 pg_statistic rows 刚写完，目标 relation 在 commit 前被删除；
避免并发 ANALYZE 更新同一 stats row；
避免释放锁后暴露 concurrent-update failure。
```
autovacuum per-table ERROR 路径：
```text
EmitErrorReport()
AbortOutOfAnyTransaction()
FlushErrorState()
MemoryContextReset(PortalContext)
StartTransactionCommand()
继续下一张候选表
```
如果 ERROR 发生在 stats 写入提交前，普通 catalog updates 会随 transaction abort 回滚。
`pg_class` relstats 和 pgstat 是近似维护状态，不能简单按用户表 MVCC 语义理解。
table claim 会在 per-table cleanup 或 `FreeWorkerInfo()` 中清掉。
syscache / relcache invalidation 的语义：
```text
catalog 变化提交后，后续 cache lookup 不应继续相信旧 entry；
invalidation 不是 lock；
它不会阻塞并发 query；
它不会修改已经开始执行的 plan。
```
## 8. 正确性机制层次
| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `ShareUpdateExclusiveLock` | 同一 relation 的 ANALYZE 不并发写同一 stats rows。 | 统计精确。 |
| catalog insert/update | `pg_statistic` 和 `pg_statistic_ext_data` 按 catalog 规则可见。 | 已有 plan 立刻改变。 |
| in-place relstats | `pg_class` 近似统计低 churn 更新。 | 普通事务回滚语义。 |
| CommandCounterIncrement | 同一 command 后续 catalog 操作看到前序变更。 | 跨 backend 同步计划。 |
| syscache / relcache invalidation | 后续 lookup 丢弃过期 cache。 | 阻塞并发修改。 |
| pgstat counters | 低成本记录变更与完成时间。 | 精确任务队列。 |
| random sampling | 用受控成本近似真实分布。 | 样本覆盖全部偏斜。 |
| optimizer fallback | 统计缺失时仍生成语义正确 plan。 | plan 性能稳定。 |
PostgreSQL 保证查询结果语义，不保证最优计划。
stale stats 通常造成性能错误，不造成可见性错误。
但性能错误可能很大：
```text
restriction rows 估错；
join rows 逐层放大；
nested loop / hash join 选择错误；
hash table spill；
index-only scan 成本错误；
parallelism 和 join order 错误。
```
这就是统计 stale 跨到 optimizer 问题域的原因。
## 9. 异常路径 / fallback
没有 pgstat entry：
```text
relation_needs_vacanalyze() 无法进行普通 threshold 判断；
普通 autoanalyze 不触发。
```
autovacuum inactive：
```text
普通 analyze threshold 不触发；
anti-wraparound vacuum 仍可能启动 worker；
看到 autovacuum worker 不等于 autoanalyze 正常运行。
```
执行前 recheck 失败：
```text
候选表被别人处理；
relation 被 drop；
reloptions 或 pgstat 变化；
返回 NULL，清 claim，继续下一张表。
```
relation 打不开或权限不允许：
```text
analyze_rel() 返回；
统计继续 stale，等待下一次机会或手动 ANALYZE。
```
空表或样本为 0：
```text
numrows == 0 时不写 per-column stats 和 extended stats；
仍可能更新 pg_class reltuples 和 pgstat；
旧 pg_statistic rows 未必被替换。
```
statistics target 为 0：
```text
对应 column 或 stats object 不重建；
extended stats 源码明确保留旧值。
```
column-list analyze：
```text
只刷新指定 columns；
pgstat_report_analyze(resetcounter=false)；
mod_since_analyze 不清零。
```
FDW：
```text
可能 import remote stats；
可能使用 FDW analyze callback；
不支持则 warning / skip；
freshness 和准确性取决于 FDW。
```
optimizer fallback：
```text
没有 statsTuple 不报错；
selfuncs.c 使用 DEFAULT_INEQ_SEL、DEFAULT_NUM_DISTINCT、
unique 推断、boolean 特例、system column 特例等。
```
fallback 的目标是生成可执行计划，不是生成好计划。
## 10. 成本、资源与跨模块传播
触发成本：
```text
worker 扫 pg_class；
读取 reloptions 和 relation pgstat；
不读取每张表的 pg_statistic 来判断是否 stale。
```
这个设计让调度便宜。
代价是它不知道具体分布变化。
少量 DML 改变高选择性值的频率，也可能没超过 threshold。
sampling IO 成本随这些变量扩张：
主要变量是 statistics target、extended statistics target、sampled blocks、table size、cache hit ratio、tuple width 和 storage locality。
CPU 成本来自：
主要来源是 tuple deforming、类型相关 compare / hash、MCV grouping、histogram boundary、expression index evaluation，以及 extended stats 的 dependencies / ndistinct / MCV build。
catalog churn：
```text
update_attstats() 更新 pg_statistic；
statext_store() 删除并插入 pg_statistic_ext_data；
vac_update_relstats() in-place 更新 pg_class；
频繁 ANALYZE 大量表会增加 system catalog 压力和 invalidation fan-out。
```
invalidation 成本随这些变量扩张：
主要变量是 backend 数、active planning 频率、relation / stats object 数、plan cache dependency 数和 catalog churn 频率。
optimizer 传播路径：
```text
pg_class.reltuples
  -> base relation rows
  -> restriction selectivity
  -> join relation rows
  -> join order search
  -> path cost ranking
  -> executor memory / IO behavior
```
VACUUM 与 ANALYZE 的边界：
```text
VACUUM:
  dead tuple cleanup、freeze、VM、index cleanup、部分 relstats。
ANALYZE:
  data distribution sample、pg_statistic、extended stats、relstats estimate、pgstat analyze counters。
```
`VACUUM ANALYZE` 不是“精确统计”。
ANALYZE 仍然是采样。
## 11. 观测与诊断入口
看是否发生 autoanalyze：
```sql
SELECT schemaname, relname,
       n_live_tup, n_dead_tup, n_mod_since_analyze,
       last_analyze, last_autoanalyze,
       analyze_count, autoanalyze_count,
       total_autoanalyze_time
FROM pg_stat_all_tables
WHERE relname = 'analyze_stale_demo';
```
能看到 relation-level counters 和完成时间。
看不到 sample 质量和 optimizer 是否重规划。
看正在运行的 ANALYZE：
```sql
SELECT pid, relid::regclass, phase,
       sample_blks_total, sample_blks_scanned,
       ext_stats_total, ext_stats_computed,
       started_by
FROM pg_stat_progress_analyze;
```
这是当前 running command 粒度。
结束后 entry 消失。
`pg_stat_activity` 中的 `autovacuum: ANALYZE ...` 或 `autovacuum: VACUUM ANALYZE ...` 可定位当前 worker activity。
看 per-column stats：
```sql
SELECT attname, null_frac, avg_width, n_distinct,
       most_common_vals, most_common_freqs,
       histogram_bounds, correlation
FROM pg_stats
WHERE schemaname = 'public'
  AND tablename = 'analyze_stale_demo'
ORDER BY attname;
```
通常优先看 `pg_stats`，不要让普通诊断直接依赖 superuser-only `pg_statistic`。
看 extended stats：
```sql
SELECT statistics_name, attnames, exprs, kinds,
       n_distinct, dependencies,
       most_common_vals, most_common_freqs
FROM pg_stats_ext
WHERE schemaname = 'public'
  AND tablename = 'analyze_stale_demo';
```
区分 definition 和 data：
```sql
SELECT e.oid, e.stxname, e.stxkind, d.stxdinherit,
       d.stxdndistinct IS NOT NULL AS has_ndistinct,
       d.stxddependencies IS NOT NULL AS has_dependencies,
       d.stxdmcv IS NOT NULL AS has_mcv,
       d.stxdexpr IS NOT NULL AS has_expr
FROM pg_statistic_ext e
LEFT JOIN pg_statistic_ext_data d ON d.stxoid = e.oid
WHERE e.stxrelid = 'analyze_stale_demo'::regclass;
```
看 relation-level stats：
查 `pg_class.reltuples`、`relpages`、`relallvisible`、`relallfrozen`，但记住 `reltuples` 是估算，不是 `COUNT(*)`。
用 EXPLAIN 连接 optimizer：
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM analyze_stale_demo
WHERE tenant_id = 42 AND status = 1;
```
诊断链路：
```text
estimated rows vs actual rows
  -> pg_stats 的 MCV / histogram / stadistinct
  -> pg_class 的 reltuples / relpages
  -> pg_stat_all_tables 的 n_mod_since_analyze / last_autoanalyze
  -> 源码 selfuncs.c / analyze.c
```
日志入口：
```text
log_autoanalyze_min_duration
```
当前源码会为超过阈值的 autoanalyze 输出耗时、buffer、WAL 等信息。
日志适合定位 autoanalyze 慢。
它不能证明统计准确。
断点可放在 `relation_needs_vacanalyze()`、`do_analyze_rel()`、`update_attstats()`、`BuildRelationExtStatistics()`、`pgstat_report_analyze()`、`estimate_rel_size()` 和 `examine_variable()`。
重点变量是 `mod_since_analyze`、`anlthresh`、`doanalyze`、`targrows`、`numrows`、`totalrows`、`stadistinct`、`stakind[]` 和 `statsTuple`。
## 12. 常见误区
误区一：autoanalyze 实时保持 optimizer 统计准确。
实际是 DML 累加 counters，超过阈值后才可能后台采样。
误区二：`last_autoanalyze` 新，计划就一定好。
样本偏差、target 太低、extended stats 缺失、表达式无法匹配、计划缓存都可能继续导致估错。
误区三：`n_mod_since_analyze` 高就一定解释 plan 差。
它只是变更计数，不告诉你变更是否影响查询谓词。
误区四：`CREATE STATISTICS` 立即生效。
必须后续 `ANALYZE` 构建 `pg_statistic_ext_data`。
误区五：`pg_statistic` 某个 slot 固定保存 histogram。
读取时必须看 `stakindN`。
误区六：invalidation 会让正在执行的 query 换计划。
invalidation 影响后续 cache lookup 和后续 planning，不修改已经执行中的 plan。
误区七：`VACUUM` 完成就代表 column distribution 新鲜。
column MCV / histogram 需要 ANALYZE。
VACUUM 和 ANALYZE 相邻，但问题域不同。
## 13. 课堂实验
### 实验 1：从 threshold 到 autoanalyze
```sql
DROP TABLE IF EXISTS autoanalyze_threshold_demo;
CREATE TABLE autoanalyze_threshold_demo(
    id int,
    k int,
    payload text
) WITH (
    autovacuum_analyze_threshold = 50,
    autovacuum_analyze_scale_factor = 0.0
);
INSERT INTO autoanalyze_threshold_demo
SELECT g, g % 10, repeat('x', 20)
FROM generate_series(1, 1000) AS g;
ANALYZE autoanalyze_threshold_demo;
UPDATE autoanalyze_threshold_demo
SET k = 999
WHERE id <= 80;
```
观察：
```sql
SELECT n_mod_since_analyze, last_autoanalyze, autoanalyze_count
FROM pg_stat_all_tables
WHERE relname = 'autoanalyze_threshold_demo';
SELECT most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'autoanalyze_threshold_demo'
  AND attname = 'k';
```
源码对应：
```text
relation_needs_vacanalyze():
  anltuples = mod_since_analyze
  anlthresh = threshold + scale_factor * reltuples
```
讨论：为什么阈值太低会造成 catalog churn，而不是免费提升计划质量？
### 实验 2：stale stats 进入 optimizer
```sql
DROP TABLE IF EXISTS planner_stale_demo;
CREATE TABLE planner_stale_demo(id int, k int, filler text);
CREATE INDEX planner_stale_demo_k_idx ON planner_stale_demo(k);
INSERT INTO planner_stale_demo
SELECT g, g % 10000, repeat('x', 30)
FROM generate_series(1, 300000) AS g;
ANALYZE planner_stale_demo;
EXPLAIN SELECT * FROM planner_stale_demo WHERE k = 7;
UPDATE planner_stale_demo SET k = 7 WHERE id <= 60000;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM planner_stale_demo WHERE k = 7;
ANALYZE planner_stale_demo;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM planner_stale_demo WHERE k = 7;
```
观察：
```text
estimated rows 与 actual rows；
MCV 是否包含 7；
most_common_freqs 是否接近真实比例；
plan 是否在 ANALYZE 后改变。
```
源码对应：
```text
examine_variable()
  -> statsTuple
eqsel() / scalarineqsel()
  -> MCV / histogram / default estimate
```
### 实验 3：extended stats definition 与 data
```sql
DROP TABLE IF EXISTS ext_stats_refresh_demo;
CREATE TABLE ext_stats_refresh_demo(a int, b int, payload text);
INSERT INTO ext_stats_refresh_demo
SELECT g % 1000, g % 1000, repeat('x', 20)
FROM generate_series(1, 200000) AS g;
CREATE STATISTICS ext_stats_refresh_demo_ab (dependencies, mcv)
ON a, b
FROM ext_stats_refresh_demo;
SELECT e.stxname, d.stxoid IS NOT NULL AS has_data
FROM pg_statistic_ext e
LEFT JOIN pg_statistic_ext_data d ON d.stxoid = e.oid
WHERE e.stxrelid = 'ext_stats_refresh_demo'::regclass;
ANALYZE ext_stats_refresh_demo;
SELECT e.stxname,
       d.stxddependencies IS NOT NULL AS has_dep,
       d.stxdmcv IS NOT NULL AS has_mcv
FROM pg_statistic_ext e
LEFT JOIN pg_statistic_ext_data d ON d.stxoid = e.oid
WHERE e.stxrelid = 'ext_stats_refresh_demo'::regclass;
```
源码对应：
```text
BuildRelationExtStatistics()
  -> fetch_statentries_for_relation()
  -> statext_store()
plancat.c:
  -> get_relation_statistics()
```
## 14. 讨论题
1. 为什么 PostgreSQL 不在每次 DML 时同步更新 MCV 和 histogram？
2. `mod_since_analyze` 高但查询计划没有变差，可能有哪些原因？
3. `last_autoanalyze` 很新但 row estimate 仍错，应该先检查哪些状态？
4. 为什么 `pg_statistic_ext` 和 `pg_statistic_ext_data` 要拆开？
5. `ShareUpdateExclusiveLock` 在 ANALYZE 中保护什么，为什么不阻塞普通 SELECT？
6. `stadistinct < 0` 为什么比固定 distinct 数更适合 unique-like column？
7. `ANALYZE t(a)` 后 `n_mod_since_analyze` 没清零，是 bug 还是设计？
8. autoanalyze commit 后，为什么已经开始执行的 query 不会中途换计划？
## 15. 本节小结
本节主链路：
```text
DML 改变 relation
  -> pgstat.mod_since_analyze 增长
  -> relation_needs_vacanalyze() 判断 threshold
  -> autovacuum worker recheck 并调用 vacuum(... VACOPT_ANALYZE ...)
  -> analyze_rel() 拿 ShareUpdateExclusiveLock
  -> do_analyze_rel() 采样、计算 per-column 和 extended stats
  -> update_attstats() 写 pg_statistic
  -> statext_store() 写 pg_statistic_ext_data
  -> vac_update_relstats() in-place 写 pg_class relstats
  -> pgstat_report_analyze() 更新 last_autoanalyze 和 counters
  -> 后续 optimizer 读取这些统计
```
核心状态边界：
```text
pgstat:
  调度输入和完成观测。
pg_statistic:
  per-column / expression stats。
pg_statistic_ext:
  extended stats definition。
pg_statistic_ext_data:
  extended stats computed data。
pg_class:
  relation-level size estimate，使用 in-place relstats update。
RelOptInfo / VariableStatData:
  单次 planning 的内存解释结果。
```
正确性边界：
```text
ANALYZE 保证统计按各自 catalog / relstats 协议刷新；
invalidation 让后续 cache lookup 不继续相信旧 catalog entry；
optimizer 在统计缺失时仍生成语义正确 plan；
系统不保证统计实时、精确，也不保证已有 plan 自动变好。
```
观测边界：
```text
能看 pg_stat_all_tables、pg_stat_progress_analyze、pg_stats、pg_stats_ext、pg_class 和 EXPLAIN；
不能直接看本轮样本质量、所有 backend 的已有 plan freshness、
也不能只凭 last_autoanalyze 证明统计适合当前 query。
```
可迁移规律：
```text
当系统把高成本全量事实压缩成周期性近似统计时，
维护模块负责刷新近似状态的生命周期和成本边界；
优化模块负责在统计可用、陈旧或缺失时做稳健估算和 fallback。
一旦优化决策依赖这些近似统计，
统计 stale 就会从后台维护问题跨入 optimizer 的计划搜索问题。
```
诊断时不要只问“autovacuum 是否跑了”。
更准确的问题是：它是否分析了这张 relation，是否覆盖相关 column / expression / extended statistics object，sample 和 target 是否足以描述当前偏斜，后续 query 是否真的重新规划并读取了新统计。
