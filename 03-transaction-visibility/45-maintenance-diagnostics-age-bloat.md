# PostgreSQL maintenance diagnostics / age / bloat
## 课程定位
前置知识：已经理解 heap tuple visibility、VACUUM heap/index 分阶段清理、freeze、visibility map、autovacuum 调度，以及 pgstat 是累计统计系统而不是事务语义本身。 本节唯一主问题：
```text
如何把 pg_stat_all_tables、age、dead tuple、VM、bloat 和 logs 组合成一条能回到源码入口的维护诊断路径？
```
核心矛盾：
```text
维护诊断必须尽快判断一张表是否正在积累不可接受的空间和 XID 风险， 但可见的指标分别来自 pgstat、catalog、VM fork、VACUUM runtime 和日志； 其中很多字段是估算、滞后或单次维护后的快照，不能单独解释因果。
```
本节不是讲“怎么调 autovacuum 参数”。 本节要训练的是源码级诊断路径：
```text
一个 runtime 现象 -> 找到 SQL 观测字段 -> 判断字段来自哪个源码状态 -> 区分直接事实、近似估算和推断 -> 回到 VACUUM / autovacuum / visibility map 的入口验证
```
学完后应能判断：
```text
n_dead_tup 高，是 autovacuum 没跑、跑了但回收不了、统计滞后，还是只是估算误差？ age(relfrozenxid) 高，是普通 bloat 问题，还是 wraparound 风险？ relallvisible 低，是 dead tuple 多，还是页面刚被写入后 VM 尚未恢复？ autovacuum log 里的 pages scanned / tuples removed / dead but not yet removable 应该如何接到源码字段？ 表文件变大，哪些部分可以从 SQL 直接看到，哪些只能推断为 bloat？
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
第 26 节回答 dead / recently dead 的 tuple-level 边界。 第 28 节回答 lazy VACUUM 为什么拆成 heap scan、index vacuum、heap cleanup。 第 29 节回答 freeze 和 anti-wraparound。 第 30、31 节回答 visibility map 的 all-visible 和 all-frozen。 第 39 节回答 autovacuum launcher / worker 如何调度。 本节把这些局部机制压成一个诊断地图。 主语不是单个 tuple，也不是一次 VACUUM。 主语是一张线上表在一段时间里的维护健康状态。 这张表可能出现这样的现场：
```text
业务说查询变慢； 磁盘使用持续上涨； pg_stat_all_tables.n_dead_tup 很高； last_autovacuum 刚更新过； age(relfrozenxid) 仍然很高； autovacuum log 显示大量 dead but not yet removable； relallvisible / relpages 比例很低； VACUUM 后 pg_relation_size 没明显下降。
```
这不是一个单字段能解释的问题。 它要求把六类状态放到同一条时间线上：
```text
pgstat relation counters: relation 层累计、估算和 last_* 时间。
catalog age: pg_class.relfrozenxid 与 pg_database.datfrozenxid 的推进边界。
heap dead state: VACUUM runtime 对 live / recently dead / removable 的判断。
visibility map: page-level all-visible / all-frozen 缓存。
bloat: 物理文件大小与可复用空间的推断，不是一个内核单字段。
logs / progress: 一次 VACUUM 执行过程留下的运行证据。
```
本节的核心目标是形成一条稳定路径：
```text
看到表膨胀或维护滞后 -> 先用 SQL 分层定位风险类型 -> 再用日志和 progress 确认 VACUUM 实际做了什么 -> 最后回到源码理解为什么指标会这样
```
## 2. 一个 runtime 现象先定锚
准备一张高 churn 表。 它有更新、删除、索引、长事务和 autovacuum。 现场常见观察是：
```text
pg_stat_all_tables.n_dead_tup 上升； last_autovacuum 周期性变化； autovacuum_count 也在增长； 但是 pg_relation_size 没下降； age(relfrozenxid) 下降得不明显； EXPLAIN 中 index-only scan 的 Heap Fetches 变多； 日志里出现 dead but not yet removable。
```
这组现象容易被误判成一句话：
```text
autovacuum 没有清理 bloat。
```
这句话通常太粗。 源码视角下至少要拆成五个问题：
```text
1. autovacuum 是否被调度并成功完成？ 2. VACUUM 看到的是 removable dead，还是 recently dead？ 3. VACUUM 是否因为 VM page skipping 没扫描所有页面？ 4. relfrozenxid / relminmxid 是否被允许推进？ 5. 文件大小没有下降，是因为空间仍被 tuple 占用，还是已经变成可复用空洞？
```
这五个问题分别落在不同源码状态上。 `pg_stat_all_tables` 只能回答其中一部分。 `age()` 只能回答 XID / MultiXact 风险的一部分。 VM 只能回答 page-level 可见性缓存。 autovacuum log 只能回答一次维护运行结束时的统计。 bloat 则多半是推断，不是 PostgreSQL core 暴露的单一事实。 因此本节的诊断不是表格查询技巧。 它是一个状态合成问题。
## 3. 核心矛盾与一句话运行模型
一句话运行模型：
```text
应用写入不断改变 heap tuple 和 VM bit； autovacuum worker 根据 pgstat counters、catalog age 和 reloptions 选择 relation； lazy VACUUM 用 GlobalVisState / cutoffs 判断 tuple 是否可回收、是否可 freeze； VACUUM 结束时同时更新 pg_class relstats、pgstat relation counters 和 verbose/autovacuum log； 诊断者只能把这些不同粒度、不同新鲜度的状态拼成近似因果链。
```
本节的系统 tension 是：
```text
维护系统需要低成本、异步、近似地追踪大量 relation 的健康状态 vs 诊断者希望获得精确、实时、可归因的 bloat / age / dead tuple 真相
```
PostgreSQL 没有选择为每张表维护一个精确的 bloat truth。 原因很直接：
```text
精确 bloat 需要扫描 heap page、tuple header、line pointer、FSM、VM 和索引。 对所有 relation 高频维护这个 truth，会把诊断本身变成维护负载。
```
所以 core 暴露的是几类低成本证据：
```text
pgstat: 事务和 VACUUM 报告出来的累计计数。
catalog: pg_class / pg_database 中的 relpages、reltuples、relfrozenxid、relallvisible。
VM: 紧凑 page-level bitmap，可被 VACUUM 和 index-only scan 使用。
progress: 当前 VACUUM worker 的阶段性状态。
log: 一次 VACUUM 完成后的详细总结。
```
诊断的关键不是相信某个字段。 诊断的关键是问：
```text
这个字段是由谁、在什么时候、基于什么扫描范围写入的？ 它代表当前 truth，还是上一次维护结束时的估算？ 它能否解释 bloat，还是只能提示下一步该看哪里？
```
## 4. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/catalog/system_views.sql` | `pg_stat_all_tables` 如何把 `pg_stat_get_*` 函数组合成用户看到的视图。 |
| 2 | `src/include/pgstat.h` | `PgStat_StatTabEntry` 中 relation stats 的字段边界。 |
| 3 | `src/backend/utils/activity/pgstat_relation.c` | `pgstat_report_vacuum()` / `pgstat_report_analyze()` 如何写 live/dead、last_* 和 count。 |
| 4 | `src/backend/postmaster/autovacuum.c` | `relation_needs_vacanalyze()` 如何使用 dead tuples、insert counters、age 和 reloptions 决定维护目标。 |
| 5 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 的 runtime counters、VM 统计、log 输出、`pgstat_report_vacuum()` 调用。 |
| 6 | `src/backend/access/heap/visibilitymap.c` | VM bit 的 set / clear / get / count；为什么 VM 是保守 hint。 |
| 7 | `src/include/access/visibilitymap.h` | `VM_ALL_VISIBLE()`、`VM_ALL_FROZEN()` 与 VM API。 |
| 8 | `src/include/catalog/pg_class.h` | `relpages`、`reltuples`、`relallvisible`、`relallfrozen`、`relfrozenxid`、`relminmxid`。 |
| 9 | `src/backend/commands/vacuum.c` | `vac_update_relstats()`、`vac_update_datfrozenxid()`、cutoff / aggressive 判定。 |
| 10 | `src/backend/utils/adt/xid.c` | `xid_age()` / `mxid_age()`，也就是 SQL 中 XID/MXID age 的 C 入口。 |
| 11 | `src/backend/utils/adt/pgstatfuncs.c` | `pg_stat_get_dead_tuples()` 等 SQL 函数如何读取 pgstat entry。 |
推荐阅读顺序：
```text
SQL view: pg_stat_all_tables 暴露了哪些字段。
pgstat update: 字段由 VACUUM / DML / ANALYZE 什么时候写。
autovacuum decision: worker 为什么选择这张表。
VACUUM runtime: 一次执行中 dead / recently dead / VM / freeze 如何变化。
catalog update: relfrozenxid、relallvisible、relpages 如何落到 pg_class。
diagnostic reconstruction: 用 SQL、progress、logs 反推运行路径。
```
不要从所有 GUC 开始。 也不要从 bloat SQL 公式开始。 本节的主线是：
```text
现象 -> 字段来源 -> 维护执行 -> 源码入口 -> 推断边界
```
## 5. 关键状态与字段边界
### 5.1 `pg_stat_all_tables` 是视图，不是存储
`system_views.sql` 中的 `pg_stat_all_tables` 组合了 `pg_class`、`pg_namespace`、索引扫描统计和一组 `pg_stat_get_*` 函数。 最相关字段可以压缩为：
```sql
SELECT
  C.oid AS relid,
  N.nspname AS schemaname,
  C.relname AS relname,
  pg_stat_get_live_tuples(C.oid) AS n_live_tup,
  pg_stat_get_dead_tuples(C.oid) AS n_dead_tup,
  pg_stat_get_mod_since_analyze(C.oid) AS n_mod_since_analyze,
  pg_stat_get_ins_since_vacuum(C.oid) AS n_ins_since_vacuum,
  pg_stat_get_last_vacuum_time(C.oid) AS last_vacuum,
  pg_stat_get_last_autovacuum_time(C.oid) AS last_autovacuum,
  pg_stat_get_vacuum_count(C.oid) AS vacuum_count,
  pg_stat_get_autovacuum_count(C.oid) AS autovacuum_count
FROM pg_class C ...
```
这些字段不是直接从 heap page 实时扫描出来的。 它们来自 cumulative stats 系统。 这意味着：
```text
n_dead_tup: 当前最好理解为 relation-level dead tuple 估算。
last_autovacuum: 最近一次 autovacuum 报告完成的时间，不等于当前正在 vacuum。
autovacuum_count: 计数增加代表有 autovacuum 报告过，不代表每次都清掉了空间。
n_ins_since_vacuum: insert-triggered vacuum 的输入之一，不等于 heap 里当前 insert tuple 数。
```
诊断时必须把它们当作证据。 不能把它们当作完整事实。
### 5.2 `PgStat_StatTabEntry` 是 relation 统计状态
本地源码中的核心字段：
```c
typedef struct PgStat_StatTabEntry
{
    PgStat_Counter live_tuples;
    PgStat_Counter dead_tuples;
    PgStat_Counter mod_since_analyze;
    PgStat_Counter ins_since_vacuum;

    TimestampTz last_vacuum_time;
    PgStat_Counter vacuum_count;
    TimestampTz last_autovacuum_time;
    PgStat_Counter autovacuum_count;

    TimestampTz last_analyze_time;
    PgStat_Counter analyze_count;
    TimestampTz last_autoanalyze_time;
    PgStat_Counter autoanalyze_count;
} PgStat_StatTabEntry;
```
字段边界：
```text
PgStat_StatTabEntry: relation 粒度。
pg_class relstats: catalog 持久元数据，VACUUM / ANALYZE 更新。
VM fork: page 粒度，独立 relation fork。
VACUUM runtime counters: 一次执行内的 backend-local 状态。
log line: 一次执行结束后的文本化摘要。
```
这些状态不是同一个 owner。 也不是同一个 freshness。 把它们拼起来时，要先问 owner：
```text
谁写的？ 什么时候写？ 写入是否跟当前事务提交有关？ 失败时是否会回滚？
```
### 5.3 `pg_class` 中的维护元数据
维护诊断常用的 `pg_class` 字段：
| 字段 | 粒度 | 语义 |
| --- | --- | --- |
| `relpages` | relation | planner 估算用页数，VACUUM / ANALYZE 更新。 |
| `reltuples` | relation | planner 估算用 tuple 数。 |
| `relallvisible` | relation | VM all-visible page 计数的 catalog 近似。 |
| `relallfrozen` | relation | VM all-frozen page 计数的 catalog 近似。 |
| `relfrozenxid` | relation | 这张表仍可能需要保留的最老 XID 边界。 |
| `relminmxid` | relation | 这张表仍可能需要保留的最老 MultiXact 边界。 |
这些字段被更新后也可能很快过时。 但它们足够便宜，能作为诊断起点。 `relallvisible / relpages` 的比例经常被用来粗看 VM 覆盖率。 但要注意：
```text
relallvisible 是上次维护更新到 pg_class 的计数； 真实 VM bit 在 VM fork 中； visibilitymap_count() 可以重新数 VM，但结果本身仍可能马上过时。
```
### 5.4 age 字段来自 XID 差值，不来自 bloat
诊断里常见 SQL：
```sql
SELECT
  c.oid::regclass,
  age(c.relfrozenxid) AS xid_age,
  age(c.relminmxid) AS mxid_age,
  s.n_dead_tup,
  s.last_autovacuum
FROM pg_class c
JOIN pg_stat_all_tables s ON s.relid = c.oid
WHERE c.relkind IN ('r', 't', 'm');
```
这里的 `age(c.relfrozenxid)` 是 wraparound 风险信号。 它不是 bloat 信号。 两者可能相关：
```text
VACUUM 长期不能推进，可能同时留下 dead tuples 和老 XID。
```
但两者不是同一件事：
```text
一张 append-only 大表可能 age 高，但 dead tuple 很低。 一张高 churn 小表可能 bloat 明显，但 age 不高。
```
源码入口是 `src/backend/utils/adt/xid.c` 中的 `xid_age()` 和 `mxid_age()`。 catalog 推进入口在 `vac_update_relstats()` 和 `vac_update_datfrozenxid()`。
### 5.5 VM 是 page-level 证明，不是 dead tuple 计数
visibility map 每个 heap page 两个 bit：
```text
VISIBILITYMAP_ALL_VISIBLE VISIBILITYMAP_ALL_FROZEN
```
源码注释里的核心语义是：
```text
bit set: PostgreSQL 知道这个条件为真。
bit clear: 条件可能为真，也可能为假，只是没有证明。
```
诊断时的边界：
```text
all-visible 低: 说明很多页面不能被 VM 证明全可见。 它可能来自 dead/recently dead，也可能来自近期写入后还没 vacuum。
all-frozen 低: 说明很多页面仍可能含有需要未来 freeze 的 XID/MXID。 它比 all-visible 更接近 anti-wraparound 工作量。
VM 高: 常常意味着 VACUUM 和 index-only scan 成本低。 但不直接证明没有 bloat，因为可复用空洞也可能在 all-visible 页之外。
```
### 5.6 bloat 是推断，不是 core 单字段
PostgreSQL core 没有一个 `pg_stat_all_tables.bloat_bytes`。 原因是 bloat 不是单一内核状态。 常见来源包括：
```text
heap page 内 LP_UNUSED / free space 不能归还给 OS； dead tuple 尚未回收； recently dead 因长事务或 slot 不能回收； 索引页分裂和删除滞后； FSM 可复用空间分布不适合当前插入模式； TOAST table 与主表维护节奏不同。
```
诊断中可以用以下证据推断：
```text
pg_relation_size(): 文件物理大小。
pg_total_relation_size(): heap + indexes + toast。
pgstattuple / pgstattuple_approx: 扩展提供更接近 page-level 的估算。
relpages / reltuples / avg row width: planner 统计的粗略密度估算。
autovacuum log: removed / remain / recently dead / scanned / VM 信息。
```
但任何公式都必须标注边界。 尤其是：
```text
VACUUM 后文件没变小，不等于 VACUUM 没工作。 普通 VACUUM 主要让空间在 relation 内部可复用。 只有尾部可截断页面满足条件时，heap 文件才可能缩小。
```
## 6. `pg_stat_all_tables` 字段来源 walkthrough
这一节从 SQL 字段反向跟源码。 入口：
```text
pg_stat_all_tables.n_dead_tup -> system_views.sql -> pg_stat_get_dead_tuples(C.oid) -> pgstatfuncs.c 读取 relation stats entry -> PgStat_StatTabEntry.dead_tuples -> pgstat_relation.c 的 pgstat_report_vacuum()
```
`pgstat_report_vacuum()` 的核心动作：
```c
tabentry->live_tuples = livetuples;
tabentry->dead_tuples = deadtuples;
tabentry->ins_since_vacuum = 0;

if (AmAutoVacuumWorkerProcess())
{
    tabentry->last_autovacuum_time = ts;
    tabentry->autovacuum_count++;
}
else
{
    tabentry->last_vacuum_time = ts;
    tabentry->vacuum_count++;
}
```
这段代码带来三个诊断结论。 第一，`n_dead_tup` 是 VACUUM 报告后的 relation 估算，而不是当前 heap 实时扫描。 第二，`last_autovacuum` 表示 autovacuum worker 成功走到 report 路径。 第三，`ins_since_vacuum` 在 VACUUM 后被清零，即使 non-aggressive VACUUM 可能跳过了部分页面。 这也是为什么诊断 insert-only 表时，不能只看 `n_dead_tup`。 新插入 tuple 的 freeze 工作会通过 insert counter 触发维护。 `relation_needs_vacanalyze()` 的注释明确把 insert threshold、dead tuple threshold、age threshold、analyze threshold 都纳入 scoring。 因此一张表进入 autovacuum 队列，可能不是因为 `n_dead_tup` 高。 也可能是因为：
```text
insert since vacuum 高； relfrozenxid 接近 freeze_max_age； relminmxid 接近 multixact freeze max age； analyze threshold 超过；
```
## 7. age 诊断路径
age 诊断先分 database 和 relation。 实例风险从 database 看：
```sql
SELECT
  datname,
  age(datfrozenxid) AS dat_xid_age,
  age(datminmxid) AS dat_mxid_age
FROM pg_database
ORDER BY dat_xid_age DESC;
```
表级风险从 relation 看：
```sql
SELECT
  c.oid::regclass AS rel,
  age(c.relfrozenxid) AS rel_xid_age,
  age(c.relminmxid) AS rel_mxid_age,
  c.relpages,
  c.reltuples
FROM pg_class c
WHERE c.relkind IN ('r', 't', 'm')
ORDER BY rel_xid_age DESC
LIMIT 20;
```
诊断顺序：
```text
1. datfrozenxid 高: 找数据库内最老 relfrozenxid。
2. relfrozenxid 高: 判断这张表最近是否被 autovacuum / manual vacuum。
3. vacuum 跑过但 age 不降: 查 log 中是否 aggressive、scanned 百分比、new relfrozenxid 是否更新。
4. non-aggressive vacuum 跳过 all-visible page: 它可能不能推进 relfrozenxid。
5. aggressive / anti-wraparound vacuum: 必须扫描需要扫描的页面并推进 freeze 边界，除非被错误、锁、取消等路径中断。
```
源码上，关系是：
```text
vacuum_set_xid_limits() -> 设置 FreezeLimit / OldestXmin / MultiXactCutoff -> lazy VACUUM 扫描、freeze、记录 NewRelfrozenXid / NewRelminMxid -> vac_update_relstats() -> 可能更新 pg_class.relfrozenxid / relminmxid -> vac_update_datfrozenxid() -> 汇总 pg_class 最小值到 pg_database.datfrozenxid / datminmxid
```
诊断时要小心：
```text
age 高不是 bloat。 age 高是 XID/MXID horizon 风险。 它可能造成更 aggressive 的维护，进而制造 I/O、WAL 和 lock 压力。
```
如果只用 `n_dead_tup` 排序，可能漏掉最危险的 anti-wraparound 表。 如果只用 `age(relfrozenxid)` 排序，可能把没有空间问题的 append-only 大表误判为 bloat 源。
## 8. dead tuple 诊断路径
dead tuple 的第一层问题：
```text
n_dead_tup 是高，还是持续升高？
```
推荐查询：
```sql
SELECT
  now() AS sampled_at,
  s.relid::regclass AS rel,
  s.n_live_tup,
  s.n_dead_tup,
  s.n_dead_tup::numeric / GREATEST(s.n_live_tup, 1) AS dead_live_ratio,
  s.n_mod_since_analyze,
  s.n_ins_since_vacuum,
  s.last_autovacuum,
  s.autovacuum_count
FROM pg_stat_all_tables s
WHERE s.schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY s.n_dead_tup DESC
LIMIT 30;
```
第二层问题：
```text
autovacuum 是否根本没跑？
```
证据：
```text
last_autovacuum 为空或很旧； autovacuum_count 不增长； pg_stat_activity 没有 autovacuum worker； 日志没有对应 relation； 可能被 autovacuum 禁用、阈值未达、worker 被全局限制、database 调度没轮到。
```
第三层问题：
```text
autovacuum 跑了但没清掉？
```
证据：
```text
last_autovacuum 更新； autovacuum_count 增长； log 中 removed 很少； log 中 dead but not yet removable 很高； 或 tuples missed due to cleanup lock contention 高。
```
这里要回到 VACUUM visibility 判定。 在 `vacuumlazy.c` 中，runtime counters 区分：
```text
live_tuples: 扫描后估算仍存活的 tuple。
recently_dead_tuples: 对当前 VACUUM 来说 dead，但仍可能被某些 snapshot 看到。
missed_dead_tuples: 因 cleanup lock contention 等原因错过的 dead tuple。
tuples_deleted: 本次真正删除或回收的 tuple。
```
日志中对应：
```text
tuples: ... removed, ... remain, ... are dead but not yet removable tuples missed: ... dead from ... pages not removed due to cleanup lock contention removable cutoff: ..., which was ... XIDs old when operation ended
```
这说明：
```text
n_dead_tup 高 + autovacuum 已完成
```
并不直接等于 autovacuum bug。 可能只是 cleanup horizon 被长事务、replication slot、prepared transaction 或 standby feedback 压住。
## 9. visibility map 诊断路径
先看 catalog 近似：
```sql
SELECT
  c.oid::regclass AS rel,
  c.relpages,
  c.relallvisible,
  c.relallfrozen,
  round(100.0 * c.relallvisible / NULLIF(c.relpages, 0), 2) AS pct_all_visible,
  round(100.0 * c.relallfrozen / NULLIF(c.relpages, 0), 2) AS pct_all_frozen
FROM pg_class c
WHERE c.relkind IN ('r', 't', 'm')
ORDER BY pct_all_visible NULLS FIRST
LIMIT 30;
```
解释规则：
```text
pct_all_visible 高: index-only scan 和 VACUUM page skipping 可能受益。
pct_all_visible 低: 页面近期被修改、存在 dead/recently dead、或 VACUUM 尚未重新证明页面全可见。
pct_all_frozen 高: aggressive vacuum / anti-wraparound 扫描压力通常更低。
pct_all_frozen 低: 未来 freeze 工作可能更重，但是否紧急仍要结合 age。
```
源码路径：
```text
lazy VACUUM final cleanup -> visibilitymap_count(rel, &new_rel_allvisible, &new_rel_allfrozen) -> vac_update_relstats(..., new_rel_allvisible, new_rel_allfrozen, ...) -> pg_class.relallvisible / relallfrozen
```
`visibilitymap_count()` 自己也承认结果是近似的。 它遍历 VM page，不锁 VM page，因为结果马上就可能过时。 这正是诊断边界：
```text
relallvisible 适合做方向判断。 它不适合证明某个 block 当前一定 all-visible。
```
如果需要验证 runtime 行为，应该用 `EXPLAIN (ANALYZE, BUFFERS)` 看 index-only scan 的 `Heap Fetches`。 示例：
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM demo
WHERE id BETWEEN 100 AND 10000;
```
如果计划是 index-only scan 但 `Heap Fetches` 高，说明执行器不能充分依赖 VM。 这可能来自 VM bit 被清、页面近期写入、VACUUM 未覆盖，或 plan 访问范围集中在非 all-visible 页。
## 10. bloat 推断路径
bloat 诊断必须先分 heap、index、toast。 基础查询：
```sql
SELECT
  c.oid::regclass AS rel,
  pg_size_pretty(pg_relation_size(c.oid)) AS heap_size,
  pg_size_pretty(pg_indexes_size(c.oid)) AS index_size,
  pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size,
  c.relpages,
  c.reltuples,
  s.n_live_tup,
  s.n_dead_tup
FROM pg_class c
JOIN pg_stat_all_tables s ON s.relid = c.oid
WHERE c.relkind IN ('r', 't', 'm')
ORDER BY pg_total_relation_size(c.oid) DESC
LIMIT 30;
```
推断步骤：
```text
1. total_size 大: 先拆 heap / index / toast，不要把 index bloat 误判为 heap dead tuple。
2. heap_size 大 + n_dead_tup 高: 可能有可回收 dead tuple，继续看 autovacuum 是否运行和 log 结果。
3. heap_size 大 + n_dead_tup 低: 可能是可复用空洞、宽行、低 fillfactor、append 历史、统计滞后，或真正 live data。
4. index_size 大: VACUUM index cleanup、页面分裂、dedup、删除滞后、REINDEX 边界需要另查。
5. VACUUM 后文件大小不降: 普通 VACUUM 通常只把空间交给 FSM，除非尾部 page 可 truncation。
```
这里的关键边界是：
```text
dead tuple: 仍在 heap 中、可以或不可以被本次 VACUUM 回收的 tuple 版本。
free space: 已经可复用，但 relation 文件仍保留的空间。
bloat: workload 下不再需要但仍占据文件大小或索引结构成本的空间。
```
三者相关，但不是同义词。 源码中 lazy VACUUM 的职责主要是：
```text
找到 removable dead items； 删除索引 TID； 把 heap line pointer 变成可复用； 更新 FSM / VM / pgstat / pg_class； 必要时尝试 truncation。
```
它不会把所有内部空洞都还给 OS。 这就是为什么诊断 bloat 时要把 `VACUUM`、`VACUUM FULL`、`CLUSTER`、`REINDEX` 的语义分开。 本节不展开重写表类维护，只强调：
```text
普通 VACUUM 的成功标志不是文件一定缩小。
```
## 11. 主流程源码 walkthrough
现在把一条完整 runtime 主线串起来。 现象：
```text
orders 表磁盘持续增长； n_dead_tup 很高； last_autovacuum 每几分钟更新； age(relfrozenxid) 也偏高； 日志显示 dead but not yet removable。
```
### 11.1 SQL 入口定位 relation
第一步：
```sql
SELECT
  s.relid::regclass AS rel,
  s.n_live_tup,
  s.n_dead_tup,
  s.last_autovacuum,
  s.autovacuum_count,
  age(c.relfrozenxid) AS xid_age,
  c.relpages,
  c.relallvisible,
  c.relallfrozen
FROM pg_stat_all_tables s
JOIN pg_class c ON c.oid = s.relid
WHERE s.relid = 'public.orders'::regclass;
```
这一步把观测分成三层：
```text
pgstat: n_live_tup, n_dead_tup, last_autovacuum, autovacuum_count。
catalog: relfrozenxid, relpages, relallvisible, relallfrozen。
size: 需要另用 pg_relation_size / pg_total_relation_size。
```
### 11.2 回到 autovacuum 选择逻辑
如果 `last_autovacuum` 更新，说明 worker 至少完成过一次 VACUUM report。 worker 为什么会处理它，读 `autovacuum.c`：
```text
do_autovacuum() -> 扫 pg_class -> 读取 relation pgstat entry -> relation_needs_vacanalyze() -> dead tuple threshold -> insert threshold -> analyze threshold -> relfrozenxid age -> relminmxid age -> score -> table_recheck_autovac() -> autovacuum_do_vac_analyze()
```
这里的诊断问题：
```text
这张表是因为 dead tuple 被 vacuum， 还是因为 insert/freeze/wraparound 被 vacuum？
```
`pg_stat_all_tables` 本身不能完全回答。 需要看：
```text
n_dead_tup; n_ins_since_vacuum; age(relfrozenxid); age(relminmxid); autovacuum log 是否写 prevent wraparound / aggressive。
```
### 11.3 进入 lazy VACUUM runtime
`autovacuum_do_vac_analyze()` 最终进入 VACUUM 主执行路径。 heap 表的核心在 `vacuumlazy.c`。 简化链路：
```text
vacuum() -> vacuum_rel() -> heap_vacuum_rel() -> lazy_scan_heap() -> heap page pruning / freezing / dead item collection -> lazy_vacuum_all_indexes() -> lazy_vacuum_heap_rel() -> visibilitymap_count() -> vac_update_relstats() -> pgstat_report_vacuum() -> verbose/autovacuum log
```
这条链路解释为什么同一次 VACUUM 会同时影响：
```text
heap page line pointer; index entries; VM bits; pg_class.relpages / reltuples / relallvisible / relfrozenxid; pg_stat_all_tables.n_dead_tup / last_autovacuum; server log。
```
### 11.4 dead but not yet removable 的源码含义
日志里的：
```text
are dead but not yet removable
```
对应 `vacrel->recently_dead_tuples`。 含义不是“VACUUM 忘了删”。 含义是：
```text
tuple 对当前某些可见性规则已是 dead； 但根据 cleanup horizon，仍可能被某些 snapshot 需要； 本次 VACUUM 不能把它当作 removable。
```
回到第 12、32 节的主线：
```text
active snapshot; registered snapshot; prepared transaction; replication slot; hot standby feedback;
```
都可能延后 cleanup horizon。
### 11.5 VM 与 relfrozenxid 推进
VACUUM 结束前会判断是否可以更新 relation freeze 边界。 `vacuumlazy.c` 中有一个关键分支：
```text
如果 non-aggressive VACUUM 因 VM 跳过 all-visible page range， 并且跳过会使 NewRelfrozenXid / NewRelminMxid 不可靠， 则保留原始 relfrozenxid / relminmxid。
```
源码中体现为：
```text
if (vacrel->skippedallvis) NewRelfrozenXid = InvalidTransactionId;
```
诊断结论：
```text
last_autovacuum 更新，不保证 relfrozenxid 推进。
```
如果 age 高，必须看这次 VACUUM 是否 aggressive / anti-wraparound，以及是否扫描了需要扫描的页面。
### 11.6 log 回填诊断结论
`vacuumlazy.c` 的 autovacuum log 会输出：
```text
index scans; pages removed / remain / scanned / eagerly scanned; tuples removed / remain / dead but not yet removable; tuples missed; removable cutoff; new relfrozenxid; new relminmxid; frozen pages / tuples frozen; visibility map pages set all-visible / all-frozen; index scan needed / not needed;
```
这些字段不是装饰。 它们是把一次 VACUUM runtime 状态带回诊断者的桥。 如果 SQL 里看到：
```text
n_dead_tup 高 + autovacuum_count 增长
```
那么 log 可以回答：
```text
本次到底扫描了多少页？ removed 是少还是多？ recently dead 是否多？ cleanup lock 是否错过页面？ freeze 边界是否推进？ VM 是否恢复？
```
这就是从 runtime 回到源码的完整闭环。
## 12. 生命周期 / ownership / cleanup
### 12.1 DML 对 pgstat 的影响
普通 INSERT / UPDATE / DELETE 在 backend-local pgstat 状态中累计计数。 事务结束、backend flush 或 stats flush 时，计数进入 cumulative stats。 这些计数是维护调度输入。 它们不是 tuple 可见性本身。 如果事务 abort，tuple 语义由事务状态和 visibility 处理。 pgstat 也有事务性统计处理，避免明显错误地把回滚操作当成 committed 表变化。 诊断上要知道：
```text
pgstat 是维护信号。 MVCC visibility 才决定 tuple 能否被回收。
```
### 12.2 autovacuum worker 持有一次维护任务
worker 生命周期来自第 39 节。 这里关心 relation 任务：
```text
worker 连接 database； 扫描 pg_class 和 pgstat； 构造 table list； claim relation； recheck； 调用 VACUUM / ANALYZE； 捕获 ERROR 后继续下一张表； 退出时归还 worker slot。
```
ownership 边界：
```text
launcher: 不持有 relation 维护任务的语义 truth。
worker: 持有本 database 内的候选 relation list。
lazy VACUUM: 持有一次 relation VACUUM 的 LVRelState。
pgstat: 持有 relation 统计结果。
catalog: 持有 relfrozenxid、relpages、relallvisible 等持久元数据。
```
### 12.3 `LVRelState` 是一次 VACUUM 的 runtime owner
`LVRelState` 中维护：
```text
cutoffs; dead_items; live_tuples; recently_dead_tuples; missed_dead_tuples; new_all_visible_pages; new_all_frozen_pages; NewRelfrozenXid; NewRelminMxid; progress / log counters。
```
它不是 shared memory。 它属于当前 worker/backend 的一次 VACUUM。 成功完成后，一部分状态被写入：
```text
pg_class: relpages, reltuples, relallvisible, relallfrozen, relfrozenxid, relminmxid。
pgstat: live_tuples, dead_tuples, last_vacuum/autovacuum, count, duration。
log: verbose/autovacuum textual summary。
```
失败时，这些结果可能不会完整出现。 所以诊断要看：
```text
是否有 progress 但没有 last_autovacuum 更新？ 是否有 ERROR context？ 是否 autovacuum_count 未增加？
```
### 12.4 VM bit 的 owner
VM bit 是 relation fork 中的物理状态。 设置者通常是 VACUUM / pruning 相关路径。 清除者是 heap 修改路径。 正确性边界：
```text
set: 必须证明 page all-visible/all-frozen； 持 heap page lock； pin 正确 VM page； 与 heap page/WAL 关系保持 crash safety。
clear: 必须在任何可能破坏 all-visible 的 heap 修改中保守执行。
```
VM 不属于 pgstat。 pg_class 的 `relallvisible` 只是 VM count 的 catalog 摘要。
## 13. 正确性机制层次
维护诊断容易把性能指标误读成 correctness。 本节把正确性分层。
| 层次 | 机制 | 保证 | 不能保证 |
| --- | --- | --- | --- |
| MVCC visibility | `HeapTupleSatisfiesVacuum*`、GlobalVisState、cutoffs | tuple 是否可被本次维护回收或 freeze | autovacuum 何时会跑 |
| snapshot horizon | ProcArray、replication slot、prepared xact 等 | 仍可能被读到的旧版本不能提前移除 | 空间一定很快回收 |
| index/heap 协议 | dead TID collection、index bulk delete、heap cleanup | 索引不留下危险 TID | 文件大小立即下降 |
| VM correctness | `visibilitymap_set/clear`、`PD_ALL_VISIBLE`、WAL ordering | VM set 时必须为真，错误 set 会导致错结果 | clear bit 表示页面一定不可全可见 |
| pgstat | cumulative stats | 低成本调度和诊断信号 | 精确实时 page truth |
| catalog relstats | `vac_update_relstats()` | planner 和维护边界的持久近似 | 不过时 |
| logs | VACUUM end summary | 一次运行的强证据 | 当前状态仍相同 |
关键不变量：
```text
incorrect VM set 是 correctness bug。 stale n_dead_tup 是诊断误差。 relfrozenxid 不能随意倒退。 dead tuple 不能在 cleanup horizon 之前回收。 index TID 必须先清理，再把 heap LP_DEAD 变成 LP_UNUSED。
```
这些不变量解释了为什么 VACUUM 有时看起来“保守”。 保守不是无效。 它是避免错误结果、wraparound 和索引危险指针的代价。
## 14. 错误路径 / 异常路径 / fallback
### 14.1 autovacuum worker ERROR 后继续
`autovacuum.c` 中每张表维护外层有 `PG_TRY()` / `PG_CATCH()`。 如果某张表 VACUUM 或 ANALYZE 报错：
```text
EmitErrorReport(); AbortOutOfAnyTransaction(); FlushErrorState(); MemoryContextReset(PortalContext); StartTransactionCommand(); 继续下一张表。
```
诊断含义：
```text
一个 worker 进程还在，不代表每张表都成功维护。 必须看 relation 对应 last_autovacuum 是否更新，和日志中是否有 ERROR context。
```
### 14.2 cleanup lock contention
VACUUM 有时拿不到页面 cleanup lock。 日志可能出现：
```text
tuples missed: ... dead from ... pages not removed due to cleanup lock contention
```
这类 dead tuple 不是 visibility horizon 问题。 它是并发访问导致本次跳过清理。 后续 VACUUM 可能再处理。 如果长期存在，需要看访问模式、长 scan、buffer pin、index scan 和页面热点。
### 14.3 non-aggressive page skipping
普通 VACUUM 会利用 VM 跳过安全页面。 这降低 I/O。 但它也意味着某些 freeze 边界不能被完整证明。 源码中通过 `skippedallvis` 保护 `relfrozenxid` 推进。 诊断含义：
```text
看到 VACUUM 扫描百分比很低，不一定是问题。 但如果 age 风险高，就要确认是否需要 aggressive vacuum。
```
### 14.4 stats stale 或 reset
`pg_stat_all_tables` 可能因 stats reset、relation recreate、刚启动、刚 vacuum 后尚未 flush 而不符合直觉。 诊断要看：
```sql
SELECT stats_reset
FROM pg_stat_database
WHERE datname = current_database();
```
也要看 relation 是否被重建：
```sql
SELECT oid, relfilenode, relpages, reltuples
FROM pg_class
WHERE oid = 'public.orders'::regclass;
```
### 14.5 table drop / relcache stale
autovacuum 选择 relation 和真正执行之间，表可能被 drop、重命名或 reloptions 改变。 worker 会在执行前 recheck。 诊断含义：
```text
候选出现过，不等于最后一定执行。 源码里 table_recheck_autovac() 是防 stale decision 的边界。
```
## 15. 成本、资源与跨模块传播
维护诊断必须估计成本如何扩散。
### 15.1 CPU 与 heap page 扫描
VACUUM 扫描 heap page 时要读取 page、检查 line pointer、判断 tuple visibility、可能查事务状态。 成本随：
```text
heap page 数； tuple 数； dead/recently dead 比例； VM coverage； 冻结检查数量；
```
扩张。 如果 VM coverage 高，普通 VACUUM 可跳过更多 page。 如果 VM coverage 低，VACUUM 更像全表维护。
### 15.2 index cleanup 放大
dead TID 需要从每个索引删除。 成本随：
```text
索引数量； dead item 数； index AM 实现； parallel vacuum 可用性； maintenance_work_mem / dead_items 容量；
```
扩张。 这解释了为什么同样 dead tuple 数，多索引表的 VACUUM 成本更高。
### 15.3 WAL 与 checkpoint 压力
VACUUM 可能产生 WAL：
```text
heap cleanup； VM set； freeze； index cleanup； truncation；
```
这些 WAL 会传播到：
```text
replication lag； checkpoint pressure； archive bandwidth； IO queue；
```
诊断时不要只看 relation 级指标。 实例级还应看：
```sql
SELECT *
FROM pg_stat_wal;
```
以及 checkpoint、IO、replication 相关视图。
### 15.4 autovacuum worker 并发与前台干扰
autovacuum 通过 worker slot、cost delay、cost balance 限制后台并发。 但如果 age 接近 wraparound，系统会更倾向保证安全。 传播路径：
```text
age 高 -> aggressive / anti-wraparound vacuum -> 更多页面不能跳过 -> I/O 与 WAL 增加 -> 前台查询延迟增加 -> 如果仍被长事务压住，recently dead 继续积累
```
这就是 maintenance lag 的典型正反馈。
### 15.5 跨模块边界
本节至少连接这些模块：
```text
pgstat: 提供 relation counters 和 last_* 时间。
autovacuum: 使用 counters、age、reloptions 调度 worker。
heap visibility: 决定 tuple 是否 live / dead / recently dead。
visibility map: 决定 page skipping、index-only scan 和 all-frozen 证据。
catalog: 保存 relfrozenxid、relpages、relallvisible。
WAL / storage: 保证 VM / heap / index cleanup 的 crash safety。
```
诊断错误往往来自跨模块边界混淆。 例如：
```text
把 pgstat dead tuple 当作 heap page 实时 truth； 把 relallvisible 当作 VM 实时 truth； 把 age 当作 bloat； 把 VACUUM log 的 removed 当作文件缩小量；
```
## 16. 观测与诊断入口
### 16.1 第一层：relation 排名
```sql
SELECT
  s.relid::regclass AS rel,
  pg_size_pretty(pg_total_relation_size(s.relid)) AS total_size,
  s.n_live_tup,
  s.n_dead_tup,
  s.n_dead_tup::numeric / GREATEST(s.n_live_tup, 1) AS dead_live_ratio,
  s.last_autovacuum,
  s.autovacuum_count,
  age(c.relfrozenxid) AS xid_age,
  round(100.0 * c.relallvisible / NULLIF(c.relpages, 0), 2) AS pct_all_visible
FROM pg_stat_all_tables s
JOIN pg_class c ON c.oid = s.relid
WHERE s.schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(s.relid) DESC
LIMIT 50;
```
这个查询只做 triage。 它不能直接宣布 bloat。
### 16.2 第二层：当前 VACUUM progress
```sql
SELECT
  pid,
  relid::regclass AS rel,
  phase,
  heap_blks_total,
  heap_blks_scanned,
  heap_blks_vacuumed,
  index_vacuum_count,
  num_dead_item_ids,
  dead_tuple_bytes
FROM pg_stat_progress_vacuum;
```
粒度：
```text
当前正在运行的 VACUUM。
```
它回答“现在在做什么”。 不回答“上一次为什么这样结束”。
### 16.3 第三层：autovacuum logs
开启：
```conf
log_autovacuum_min_duration = 0
```
或在生产中设置为一个合理阈值。 重点读：
```text
automatic vacuum of table ... automatic aggressive vacuum ... automatic vacuum to prevent wraparound ... pages: removed, remain, scanned tuples: removed, remain, dead but not yet removable removable cutoff new relfrozenxid new relminmxid visibility map index scan needed / not needed
```
粒度：
```text
一次 VACUUM 执行完成后的 summary。
```
它通常比 `pg_stat_all_tables` 更接近“为什么这次没有清掉”。
### 16.4 第四层：index-only scan 与 VM
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT indexed_col
FROM public.orders
WHERE indexed_col BETWEEN 1000 AND 2000;
```
看：
```text
Index Only Scan; Heap Fetches; shared hit/read;
```
`Heap Fetches` 高时，回到 VM。 但注意：
```text
Heap Fetches 高不一定是 VACUUM 失败。 访问范围可能正好落在近期修改页。
```
### 16.5 第五层：只能推断的状态
这些状态不能由一个 core 视图直接证明：
```text
heap bloat 精确字节； index bloat 精确字节； 每个 page 的 free space 分布； 某个 n_dead_tup 值的误差； 某次 autovacuum 未触发的全部原因； 长事务对某张表每个 tuple 的具体阻塞贡献；
```
需要结合：
```text
pgstattuple 扩展； pageinspect； server log； pg_stat_activity； pg_replication_slots； pg_prepared_xacts； perf / gdb / tracepoint； 源码断点。
```
## 17. 课堂实验
### 实验 1：dead tuple 与 long transaction
目标：观察 `n_dead_tup`、autovacuum log、recently dead 的关系。 会话 A：
```sql
BEGIN;
SELECT count(*) FROM demo_churn;
```
保持事务不提交。 会话 B：
```sql
UPDATE demo_churn
SET payload = payload || 'x'
WHERE id % 2 = 0;

VACUUM (VERBOSE) demo_churn;
```
观察：
```sql
SELECT n_live_tup, n_dead_tup, last_vacuum
FROM pg_stat_all_tables
WHERE relid = 'demo_churn'::regclass;
```
预期：
```text
VACUUM 可能报告 dead but not yet removable。 会话 A 释放后再次 VACUUM，removable 数会变化。
```
源码回看：
```text
vacuumlazy.c: recently_dead_tuples removable cutoff
visibility / ProcArray: cleanup horizon 为什么被长事务压住。
```
### 实验 2：VM coverage 与 Heap Fetches
目标：观察 all-visible bit 如何影响 index-only scan。
```sql
CREATE TABLE vm_diag(id int primary key, payload text);
INSERT INTO vm_diag
SELECT g, repeat('x', 50)
FROM generate_series(1, 100000) AS g;

VACUUM (ANALYZE, VERBOSE) vm_diag;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM vm_diag WHERE id BETWEEN 1000 AND 50000;
```
然后：
```sql
UPDATE vm_diag
SET payload = payload || 'y'
WHERE id BETWEEN 2000 AND 3000;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM vm_diag WHERE id BETWEEN 1000 AND 50000;
```
预期：
```text
更新后相关页面 VM bit 被清。 index-only scan 的 Heap Fetches 可能上升。 再次 VACUUM 后下降。
```
源码回看：
```text
visibilitymap_clear() visibilitymap_set() visibilitymap_count() heap_page_would_be_all_visible()
```
### 实验 3：age 与 freeze 推进
目标：区分 dead tuple 清理和 relfrozenxid 推进。 查询：
```sql
SELECT
  c.oid::regclass,
  age(c.relfrozenxid),
  c.relallfrozen,
  c.relpages,
  s.last_autovacuum
FROM pg_class c
JOIN pg_stat_all_tables s ON s.relid = c.oid
WHERE c.oid = 'demo_churn'::regclass;
```
运行：
```sql
VACUUM (VERBOSE, FREEZE) demo_churn;
```
观察 verbose output 中：
```text
new relfrozenxid; frozen pages; visibility map all-frozen;
```
源码回看：
```text
vacuum_set_xid_limits() vac_update_relstats() vac_update_datfrozenxid()
```
### 实验 4：源码断点练习
在本地 debug build 上设置断点：
```text
pgstat_report_vacuum relation_needs_vacanalyze heap_page_would_be_all_visible visibilitymap_set vac_update_relstats
```
观察：
```text
pg_stat_all_tables.n_dead_tup 更新发生在 VACUUM 结束附近； relallvisible 来自 visibilitymap_count； relfrozenxid 是否更新取决于 NewRelfrozenXid 是否有效； autovacuum log 与 LVRelState counters 对应。
```
这个实验的目的不是记函数参数。 目的是把 SQL 字段和 runtime owner 对上。
## 18. 常见误区
### 18.1 把 `n_dead_tup` 当成精确实时值
`n_dead_tup` 是 relation 统计。 它会滞后、估算、被 reset，也会被 VACUUM report 覆盖。 它适合排序和趋势。 不适合证明某个 page 现在有多少 dead tuple。
### 18.2 把 `age(relfrozenxid)` 当成 bloat
age 是 wraparound / freeze 维度。 bloat 是空间维度。 两者可能一起出现，但不能互相替代。
### 18.3 以为 `last_autovacuum` 更新就代表维护成功
它只说明 autovacuum report 发生。 它不说明：
```text
removed 很多； age 推进； 文件缩小； index bloat 消失； recently dead 被清掉；
```
要看 log、progress、age、VM 和 size。
### 18.4 以为 VM bit clear 表示页面一定不可全可见
VM 是保守 hint。 set 必须为真。 clear 只是没有证明。
### 18.5 以为 VACUUM 后 relation size 必须下降
普通 VACUUM 主要释放 relation 内部可复用空间。 只有尾部连续可截断页满足条件时，文件大小才可能下降。 否则空间留给后续 INSERT / UPDATE 复用。
### 18.6 把 autovacuum log 当成当前状态
log 是一次结束时的 summary。 高 churn 表在 log 写出后马上又可能产生新的 dead tuple。 诊断要结合采样时间。
## 19. 讨论题
1. 为什么 PostgreSQL 不在 core 中维护一个精确的 `bloat_bytes` 字段？
2. `n_dead_tup` 很高但 `last_autovacuum` 刚更新，至少有哪些源码层面的解释？
3. `age(relfrozenxid)` 很高但 `n_dead_tup` 很低，这张表是否一定需要 bloat 处理？
4. 为什么 VM bit clear 不能直接解释为“页面上有 dead tuple”？
5. autovacuum log 中 `dead but not yet removable` 应该回到哪些 visibility / cleanup horizon 状态解释？
6. 为什么 non-aggressive VACUUM 跳过 all-visible page 后，可能不能推进 `relfrozenxid`？
7. 一张表有 8 个索引时，dead tuple 对 VACUUM 成本的放大路径是什么？
8. 如果 `pg_stat_progress_vacuum` 显示 worker 正在 `vacuuming indexes`，而业务查询延迟升高，你会继续看哪些实例级指标？
## 20. 本节小结
本节只回答一个问题：
```text
如何把 pg_stat_all_tables、age、dead tuple、VM、bloat 和 logs 组合成维护诊断路径。
```
核心链路是：
```text
runtime 现象 -> pg_stat_all_tables 定位 relation 和 dead/last/autovacuum 证据 -> pg_class / age 判断 freeze 和 wraparound 风险 -> relallvisible / relallfrozen 判断 VM 覆盖方向 -> pg_stat_progress_vacuum 和 logs 解释一次 VACUUM 实际做了什么 -> relation size / extension / page 工具推断 bloat -> 回到 pgstat_relation.c、autovacuum.c、vacuumlazy.c、visibilitymap.c、vacuum.c 验证字段来源
```
核心状态和边界：
```text
pgstat: 低成本累计统计，不是 heap 实时 truth。
catalog: relation 持久元数据，服务 planner、freeze 和诊断。
VM: page-level 保守证明，set 必须为真，clear 只是未知。
LVRelState: 一次 VACUUM 的 runtime owner，最终把部分结果写入 pgstat、pg_class 和 log。
bloat: 多状态推断，不是单一内核字段。
```
ownership / cleanup：
```text
DML 产生 pgstat 维护信号； autovacuum worker 持有一次 relation 任务； lazy VACUUM 持有 runtime counters； VACUUM 成功结束后更新 pg_class、pgstat 和 logs； ERROR 路径会 abort 当前事务、清理 context，并在 autovacuum worker 中继续下一张表。
```
正确性依赖：
```text
cleanup horizon 防止过早回收； heap/index 分阶段协议防止危险 TID； VM set/clear 与 WAL ordering 防止错误结果； relfrozenxid 推进必须基于足够扫描和 freeze 证据； pgstat 滞后只影响诊断和调度近似，不改变 MVCC 语义。
```
可迁移规律：
```text
维护诊断不是寻找一个万能指标。 它是把不同 owner、不同粒度、不同新鲜度的状态放到同一条时间线上， 再判断哪些是直接事实，哪些是近似估算，哪些只是 bloat 推断。
```
仍然依赖 workload、硬件、版本和配置的判断：
```text
dead/live ratio 多高算异常； VM coverage 多低会伤害 index-only scan； autovacuum cost delay 是否太保守； VACUUM 后空间能否复用； 是否需要 VACUUM FULL / CLUSTER / REINDEX；
```
这些不能只靠源码给出固定答案。 但源码能告诉你：
```text
每个观测字段来自哪里， 它何时被写入， 它不能证明什么， 以及下一步应该验证哪个状态。
```
