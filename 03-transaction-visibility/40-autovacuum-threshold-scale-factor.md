# PostgreSQL autovacuum threshold / scale factor
## 课程定位
前置知识：已经理解 MVCC tuple version、dead / recently dead 清理边界、lazy VACUUM 的 heap / index 分阶段清理、visibility map、freeze，以及上一节 autovacuum launcher / worker 调度。 本节唯一主问题：
```text
insert/update/delete 计数、threshold、scale factor 和 reloptions 如何决定一张表是否触发 vacuum/analyze？
```
核心矛盾：
```text
系统必须根据 workload 自动维护每张表，
否则 dead tuple、insert-only unfrozen tuple 和 planner stats 会持续失真；
但触发条件只能依赖近似统计、catalog 估算和可被用户覆盖的 reloptions，
不能在每次 DML 后同步扫描 relation 或构造强一致任务队列。
```
学完后应能独立判断：
```text
n_dead_tup、n_ins_since_vacuum、n_mod_since_analyze 分别驱动什么；
vacuum threshold、insert vacuum threshold、analyze threshold 如何计算；
reltuples、relpages、relallfrozen 如何进入判断；
reloptions 何时覆盖 GUC，何时只是 sentinel 表示继续使用 GUC；
TOAST table、partitioned table、matview 和 temp table 在 autovacuum 判定中有什么边界；
为什么一次触发判断不是强一致承诺，而是可被 recheck、skip locked 和后续统计刷新修正的近似决策。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
第 28 节讲一张表进入 VACUUM 后，heap scan、index cleanup 和 heap cleanup 如何分阶段推进。 第 29 到 31 节讲 freeze、all-visible 和 all-frozen 如何把 MVCC 可见性边界写入持久状态。 第 39 节讲 autovacuum launcher 如何按 database 调度 worker，并让 worker 在一个 database 内选择 relation。 本节接在第 39 节之后，只回答 worker 扫到某张 relation 时的判定问题。 换句话说，第 39 节的问题是：
```text
谁有机会被 worker 检查？
```
本节的问题是：
```text
被检查的这张表，到底是否需要 VACUUM 或 ANALYZE？
```
本节不重复 lazy VACUUM 的页内清理细节。 本节也不展开 anti-wraparound vacuum 的完整 freeze 策略。 但是本节必须把 freeze forced vacuum 放进触发判断里。 原因是 `relation_needs_vacanalyze()` 同时返回三类结果：
```text
dovacuum
doanalyze
wraparound
```
如果只讲 threshold / scale factor，而不讲 forced vacuum 如何绕过 `autovacuum_enabled=false`，就会误解 autovacuum 的 correctness 边界。 本节的 runtime 现象锚点：
```text
设置很低的 per-table autovacuum threshold；
对一张表分别制造 delete、insert-only 和 update；
观察 pg_stat_all_tables 的 n_dead_tup、n_ins_since_vacuum、n_mod_since_analyze；
用 pg_stat_get_autovacuum_scores() 观察 do_vacuum / do_analyze；
再回到 relation_needs_vacanalyze() 验证公式。
```
这条线会证明：
```text
autovacuum 触发不是“表变脏就立即维护”；
它是 pgstat 近似计数 + pg_class 估算大小 + reloptions/GUC 参数
在 worker 扫描时合成的一次阈值判断。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
DML backend 把 relation 级计数累积到 pgstat；
autovacuum worker 扫 pg_class 得到 reltuples、relpages、relallfrozen 和 reloptions；
relation_needs_vacanalyze() 取 pgstat 的 dead/insert/changed counters，
计算 vacuum、insert vacuum 和 analyze 三个 threshold；
只要对应 counter 严格大于 threshold，或者 XID/MultiXact age 触发 forced vacuum，
worker 就把该 relation 加入候选列表、按 score 排序、claim 后 recheck，再调用 vacuum()/analyze()。
```
这条模型里有三组状态源。 第一组是 workload 计数。 它们来自 `PgStat_StatTabEntry`：
```text
dead_tuples
ins_since_vacuum
mod_since_analyze
```
第二组是 relation 大小估算。 它们来自 `pg_class`：
```text
reltuples
relpages
relallfrozen
relfrozenxid
relminmxid
```
第三组是策略参数。 它们来自 GUC 或 `pg_class.reloptions`：
```text
autovacuum_vacuum_threshold
autovacuum_vacuum_scale_factor
autovacuum_vacuum_max_threshold
autovacuum_vacuum_insert_threshold
autovacuum_vacuum_insert_scale_factor
autovacuum_analyze_threshold
autovacuum_analyze_scale_factor
autovacuum_enabled
autovacuum_freeze_max_age
autovacuum_multixact_freeze_max_age
```
核心 tension 在于：
```text
维护触发越敏感，bloat 和 stats staleness 越容易被压住；
但后台 VACUUM / ANALYZE 会消耗 IO、CPU、WAL、buffer cache 和 relation locks。
```
PostgreSQL 没有把这个问题做成一个全局精确优化器。 它选择了一个局部、近似、可覆盖、可重试的模型。 局部，是因为 worker 在当前 database 内扫描 `pg_class`。 近似，是因为 pgstat 和 `pg_class.reltuples` 都可能滞后。 可覆盖，是因为每张表可以通过 reloptions 改写阈值。 可重试，是因为候选列表执行前还会 recheck，并且一次错过会在下一轮 worker 扫描中重新判断。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/autovacuum.c` | `do_autovacuum()` 扫描 `pg_class`，`relation_needs_vacanalyze()` 计算阈值、score 和触发结果，`table_recheck_autovac()` 执行前复查。 |
| 2 | `src/include/postmaster/autovacuum.h` | autovacuum GUC extern：threshold、scale factor、insert threshold、analyze threshold、score weight。 |
| 3 | `src/include/pgstat.h` | `PgStat_StatTabEntry` 中 relation counters 的语义边界。 |
| 4 | `src/backend/utils/activity/pgstat_relation.c` | DML 计数如何 flush，VACUUM / ANALYZE 如何重置 `ins_since_vacuum` 和 `mod_since_analyze`。 |
| 5 | `src/include/utils/rel.h` | `AutoVacOpts` 和 `StdRdOptions` 描述 reloptions 解析后的内存布局。 |
| 6 | `src/backend/access/common/reloptions.c` | autovacuum reloptions 的合法范围、默认 sentinel、TOAST 默认 analyze 禁用。 |
| 7 | `src/backend/utils/misc/guc_parameters.dat` | autovacuum GUC 默认值、范围和 SIGHUP / postmaster 生效边界。 |
| 8 | `src/backend/catalog/system_views.sql` | `pg_stat_all_tables` 暴露 `n_dead_tup`、`n_mod_since_analyze`、`n_ins_since_vacuum`。 |
| 9 | `src/include/catalog/pg_proc.dat` | `pg_stat_get_autovacuum_scores()` 暴露 score 和触发布尔值。 |
| 10 | `src/backend/access/heap/vacuumlazy.c` | VACUUM 完成后如何更新 relation tuple/page stats，作为阈值输入的后续来源。 |
| 11 | `src/backend/commands/analyze.c` | ANALYZE 完成后如何更新 `pg_class.reltuples` 和重置 analyze counter。 |
推荐阅读顺序：
```text
PgStat_StatTabEntry
  -> pgstat_relation_flush_cb_locked()
  -> pgstat_report_vacuum() / pgstat_report_analyze()
  -> AutoVacOpts / reloptions.c
  -> do_autovacuum() first pass and TOAST second pass
  -> relation_needs_vacanalyze()
  -> table_recheck_autovac()
  -> pg_stat_get_autovacuum_scores()
```
不要从 `vacuumlazy.c` 开始。 那会让你先陷入 VACUUM 如何清理页面。 本节的主语是触发判断，不是清理实现。
## 4. 一个 runtime 现象先定锚
准备一张普通 heap 表，把 per-table threshold 设成 `5`、scale factor 设成 `0`，再分别制造 delete、insert-only 和 update。
```sql
ALTER TABLE av_threshold_demo SET (
    autovacuum_vacuum_threshold = 5,
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_vacuum_insert_threshold = 5,
    autovacuum_vacuum_insert_scale_factor = 0,
    autovacuum_analyze_threshold = 5,
    autovacuum_analyze_scale_factor = 0
);
DELETE FROM av_threshold_demo WHERE id <= 10;
INSERT INTO av_threshold_demo(k, payload)
SELECT g, repeat('y', 200) FROM generate_series(1, 10) AS g;
UPDATE av_threshold_demo SET payload = payload || 'z' WHERE id BETWEEN 20 AND 29;
```
然后观察 `pg_stat_all_tables` 的 `n_dead_tup`、`n_ins_since_vacuum`、`n_mod_since_analyze`，以及 `pg_stat_get_autovacuum_scores()` 的 `vacuum_score`、`vacuum_insert_score`、`analyze_score`、`do_vacuum`、`do_analyze`。 你会看到三个不同的 counter 被推进：`DELETE` 和 `UPDATE` 推进 dead tuple 相关估算；`INSERT` 推进 `n_ins_since_vacuum`；三类 DML 都推进 `n_mod_since_analyze`。 如果阈值为 5 且 scale factor 为 0，满足严格大于阈值时才触发：
```text
counter > threshold
```
这意味着等于 5 还不够。 至少 6 才满足普通 threshold 条件。 这不是文档措辞的随意差异。 源码里使用的是 `>`。
## 5. 关键状态与结构
### 5.1 `PgStat_StatTabEntry`: 触发判断的计数来源
`src/include/pgstat.h` 中的 relation stats entry 是本节的第一个核心结构。 本节只关心下面几个字段：
| 字段 | 触发用途 | 观测列 |
| --- | --- | --- |
| `dead_tuples` | 普通 VACUUM 的主输入。 | `pg_stat_all_tables.n_dead_tup` |
| `ins_since_vacuum` | insert vacuum 的主输入。 | `pg_stat_all_tables.n_ins_since_vacuum` |
| `mod_since_analyze` | ANALYZE 的主输入。 | `pg_stat_all_tables.n_mod_since_analyze` |
| `live_tuples` | 运行统计，不直接进入 threshold 公式。 | `pg_stat_all_tables.n_live_tup` |
| `last_autovacuum_time` | 执行结果，不是触发原因。 | `pg_stat_all_tables.last_autovacuum` |
| `last_autoanalyze_time` | 执行结果，不是触发原因。 | `pg_stat_all_tables.last_autoanalyze` |
raw field 不是语义。 `dead_tuples` 是累计统计系统里的估算。 它不是 heap 上实时 `LP_DEAD` 数量。 `ins_since_vacuum` 记录的是自上次 VACUUM 后的 insert 计数。 源码注释明确指出它用 `tuples_inserted` 更新，因此会统计 aborted inserts。 这不是理想语义，但实现上避免了为少见场景增加额外字段。 `mod_since_analyze` 是自上次 ANALYZE 后的 changed tuples。 它服务 planner statistics freshness，而不是空间回收。 因此 `n_mod_since_analyze` 高，不代表一定有 bloat。
### 5.2 `pg_class`: 触发公式的大小和 age 输入
`relation_needs_vacanalyze()` 从 `Form_pg_class` 读取：
| 字段 | 语义 |
| --- | --- |
| `reltuples` | 表中 tuple 数估算，作为 scale factor 的乘数。 |
| `relpages` | relation 页数估算，用于 insert vacuum 的 unfrozen 百分比计算。 |
| `relallfrozen` | all-frozen 页数估算，用于降低 insert vacuum threshold 的 active 范围。 |
| `relfrozenxid` | XID wraparound forced vacuum 输入。 |
| `relminmxid` | MultiXact wraparound forced vacuum 输入。 |
| `relkind` | 判断普通表、matview、TOAST 等边界。 |
| `relpersistence` | temp table 跳过边界。 |
| `reltoastrelid` | 第一遍扫描建立 main table 到 TOAST table 的 reloptions 映射。 |
`reltuples` 不是精确行数。 它通常由 VACUUM 或 ANALYZE 更新。 如果 `reltuples < 0`，源码在阈值计算前把它当作 0。 这是新表或缺少统计时的重要边界。 小表的 scale 部分可能为 0。 此时 base threshold 决定触发点。 大表的 scale 部分会支配触发点。 因此默认配置下，千万行表不会因为 50 个 update 就 autovacuum。
### 5.3 `AutoVacOpts`: reloptions 解析后的策略输入
`src/include/utils/rel.h` 中的 `AutoVacOpts` 是 reloptions 的内存形态。 本节关注这些字段：
| 字段 | 作用 |
| --- | --- |
| `enabled` | 普通 threshold 是否允许触发 autovacuum。 |
| `vacuum_threshold` | 普通 VACUUM base threshold。 |
| `vacuum_max_threshold` | 普通 VACUUM threshold 上限。 |
| `vacuum_ins_threshold` | insert vacuum base threshold，`-1` 可禁用 insert vacuum。 |
| `analyze_threshold` | ANALYZE base threshold。 |
| `vacuum_scale_factor` | 普通 VACUUM scale factor。 |
| `vacuum_ins_scale_factor` | insert vacuum scale factor。 |
| `analyze_scale_factor` | ANALYZE scale factor。 |
| `freeze_max_age` | per-table forced vacuum age，上限受全局 GUC 限制。 |
| `multixact_freeze_max_age` | per-table MultiXact forced vacuum age，上限受有效全局值限制。 |
`AutoVacOpts` 不直接存放 SQL 文本。 SQL reloptions 先保存在 `pg_class.reloptions` 这个 `text[]` 里。 relcache / autovacuum 读取 `pg_class` tuple 时，通过 `extractRelOptions()` 和 `heap_reloptions()` 解析成 `StdRdOptions`。 `extract_autovac_opts()` 再复制其中的 `autovacuum` 部分。 调用链是：
```text
pg_class.reloptions
  -> extractRelOptions()
  -> heap_reloptions()
  -> StdRdOptions.autovacuum
  -> extract_autovac_opts()
  -> AutoVacOpts copy
```
这也解释了为什么 `AutoVacOpts *relopts` 可能为 `NULL`。 `NULL` 不是“禁用 autovacuum”。 它表示这张 relation 没有设置相关 reloptions，应该使用 GUC。
### 5.4 sentinel 值不是普通值
reloptions 用负数作为 sentinel。 这组 sentinel 是本节最容易读错的地方。 普通 threshold 和 scale factor 的规则：
```text
< 0:
  使用对应 GUC
>= 0:
  使用 relation reloption
```
但是 `vacuum_max_threshold` 不同：
```text
-2:
  使用 autovacuum_vacuum_max_threshold GUC
-1:
  禁用 max threshold cap
>= 0:
  使用 relation reloption 作为 cap
```
`vacuum_insert_threshold` 也不同：
```text
-2:
  使用 autovacuum_vacuum_insert_threshold GUC
-1:
  禁用 insert vacuum
>= 0:
  使用 relation reloption
```
这就是为什么源码中有这样的判断：
```text
vac_max_thresh = relopts && relopts->vacuum_max_threshold >= -1
  ? relopts->vacuum_max_threshold
  : autovacuum_vac_max_thresh;

vac_ins_base_thresh = relopts && relopts->vacuum_ins_threshold >= -1
  ? relopts->vacuum_ins_threshold
  : autovacuum_vac_ins_thresh;
```
不要把所有负数都理解成同一件事。 `-1` 在不同字段里可能表示“使用 GUC”，也可能表示“禁用某项”。
### 5.5 `AutoVacuumScores`: 触发之外的排序输入
本地源码里 `relation_needs_vacanalyze()` 还返回 `AutoVacuumScores`。 score 不决定“是否需要维护”。 是否需要维护由 `dovacuum`、`doanalyze`、`wraparound` 决定。 score 决定候选列表排序。 核心思想是：
```text
score = max(counter_or_age / threshold_or_limit)
```
组件包括：
| score | 输入 |
| --- | --- |
| `xid` | `relfrozenxid` age / freeze max age，再乘 freeze score weight。 |
| `mxid` | `relminmxid` age / multixact freeze max age，再乘 multixact score weight。 |
| `vac` | `dead_tuples` / vacuum threshold，再乘 vacuum score weight。 |
| `vac_ins` | `ins_since_vacuum` / insert vacuum threshold，再乘 insert score weight。 |
| `anl` | `mod_since_analyze` / analyze threshold，再乘 analyze score weight。 |
| `max` | 上述最大值。 |
score 是优先级，不是强一致 order。 候选收集和真正执行之间，其他 worker 或手工 VACUUM 可能已经改变状态。 因此执行前还要 `table_recheck_autovac()`。
## 6. 主流程源码 walkthrough
### 6.1 DML backend 产生统计输入
一次 DML 不会同步调用 `relation_needs_vacanalyze()`。 DML backend 先在 relation-local / transaction-local pgstat 状态里记录变化。 最终在 `pgstat_relation_flush_cb_locked()` 把 local counts 合并到 shared stats entry。 本节关注三条累积：
```text
tabentry->dead_tuples += lstats->counts.delta_dead_tuples;
tabentry->mod_since_analyze += lstats->counts.changed_tuples;
tabentry->ins_since_vacuum += lstats->counts.tuples_inserted;
```
`changed_tuples` 对应 insert、update、delete。 `tuples_inserted` 驱动 insert vacuum。 `delta_dead_tuples` 受 update/delete、abort、pruning、vacuum/analyze 估算等路径影响。 注意粒度。 这些是 cumulative stats，不是 heap page 实时状态。 如果刚执行完 DML 就马上查询 `pg_stat_all_tables`，你可能看不到预期值。 原因可能是 pgstat 还未 flush。 这不是 threshold 公式错了。
### 6.2 VACUUM 和 ANALYZE 重置不同 counter
`pgstat_report_vacuum()` 做三件和本节相关的事。 第一，更新 live/dead tuple 估算：
```text
tabentry->live_tuples = livetuples;
tabentry->dead_tuples = deadtuples;
```
第二，重置 insert vacuum counter：
```text
tabentry->ins_since_vacuum = 0;
```
第三，记录 `last_autovacuum_time` 或 `last_vacuum_time`。 它不会重置 `mod_since_analyze`。 原因是 VACUUM 不会生成 planner statistics。 `pgstat_report_analyze()` 也会更新 live/dead tuple 估算。 如果 `resetcounter` 为真，它会重置：
```text
tabentry->mod_since_analyze = 0;
```
它不会重置 `ins_since_vacuum`。 原因是 ANALYZE 不冻结 tuple，也不完成 insert vacuum 的目标。 这形成一个重要不变量：
```text
VACUUM 收尾 insert-vacuum counter；
ANALYZE 收尾 analyze counter；
两者都可能刷新 live/dead tuple 估算；
但它们不会互相替对方的 trigger counter 收尾。
```
### 6.3 worker 扫描 `pg_class`
`do_autovacuum()` 连接一个 database 后，打开 `pg_class`：
```text
classRel = table_open(RelationRelationId, AccessShareLock);
relScan = table_beginscan_catalog(classRel, 0, NULL);
```
第一遍扫描只收集普通 heap relation 和 materialized view。 源码条件是：
```text
relkind == RELKIND_RELATION
  || relkind == RELKIND_MATVIEW
```
临时表被跳过。 其他 backend 的 temp table 不能被 autovacuum 安全处理。 如果看起来是 orphan temp table，worker 会记下 OID，后续用单独路径尝试清理。 这不是 threshold 触发路径。 第一遍扫描还会记录 main table 和 TOAST table 的映射。 这个映射服务第二遍 TOAST 扫描。 原因是 TOAST table 如果没有自己的 reloptions，需要继承 main table 的 autovacuum reloptions。
### 6.4 读取 reloptions
普通表路径：
```text
relopts = extract_autovac_opts(tuple, pg_class_desc);
```
如果 `pg_class.reloptions` 里没有可解析的 autovacuum options，返回 `NULL`。 `relation_needs_vacanalyze()` 会用 GUC。 TOAST 第二遍路径更绕。 先尝试读取 TOAST 自己的 reloptions。 如果没有，再用第一遍保存的 main table reloptions。 调用链是：
```text
TOAST tuple reloptions
  -> if absent, main table reloptions from table_toast_map
  -> relation_needs_vacanalyze()
```
这说明 TOAST table 不是 parent vacuum 的附带清理。 autovacuum worker 把 TOAST table 当独立 relation 进行 threshold 判断。
### 6.5 `relation_needs_vacanalyze()` 合成参数
函数入口：
```text
relation_needs_vacanalyze(relid,
                          relopts,
                          classForm,
                          effective_multixact_freeze_max_age,
                          elevel,
                          &dovacuum,
                          &doanalyze,
                          &wraparound,
                          &scores);
```
第一步是选择策略参数。 普通 vacuum 参数：
```text
vac_base_thresh =
  relopts && relopts->vacuum_threshold >= 0
    ? relopts->vacuum_threshold
    : autovacuum_vac_thresh;

vac_scale_factor =
  relopts && relopts->vacuum_scale_factor >= 0
    ? relopts->vacuum_scale_factor
    : autovacuum_vac_scale;
```
vacuum max threshold：
```text
vac_max_thresh =
  relopts && relopts->vacuum_max_threshold >= -1
    ? relopts->vacuum_max_threshold
    : autovacuum_vac_max_thresh;
```
insert vacuum 参数：
```text
vac_ins_base_thresh =
  relopts && relopts->vacuum_ins_threshold >= -1
    ? relopts->vacuum_ins_threshold
    : autovacuum_vac_ins_thresh;

vac_ins_scale_factor =
  relopts && relopts->vacuum_ins_scale_factor >= 0
    ? relopts->vacuum_ins_scale_factor
    : autovacuum_vac_ins_scale;
```
analyze 参数：
```text
anl_base_thresh =
  relopts && relopts->analyze_threshold >= 0
    ? relopts->analyze_threshold
    : autovacuum_anl_thresh;

anl_scale_factor =
  relopts && relopts->analyze_scale_factor >= 0
    ? relopts->analyze_scale_factor
    : autovacuum_anl_scale;
```
第二步是计算 forced vacuum。 如果 `relfrozenxid` 或 `relminmxid` 太老，`force_vacuum` 为真。 这一步在读取 pgstat entry 之前发生。 因此即使没有普通 stats，anti-wraparound vacuum 仍然可以触发。 第三步是读取 relation stats：
```text
tabentry = pgstat_fetch_stat_tabentry_ext(classForm->relisshared,
                                          relid,
                                          &may_free);
```
如果没有 `tabentry`，普通 threshold 无法判断。 但此前已经设置的 forced vacuum 不会被撤销。 第四步是取三个 counter：
```text
vactuples = tabentry->dead_tuples;
instuples = tabentry->ins_since_vacuum;
anltuples = tabentry->mod_since_analyze;
```
第五步是计算三个 threshold。
### 6.6 普通 VACUUM threshold
普通 VACUUM 公式：
```text
vacthresh = autovacuum_vacuum_threshold
          + autovacuum_vacuum_scale_factor * reltuples
```
如果 reloption 覆盖，就把对应 GUC 替换成 reloption。 如果 max threshold 生效：
```text
if vac_max_thresh >= 0 and vacthresh > vac_max_thresh:
    vacthresh = vac_max_thresh
```
触发条件：
```text
if av_enabled and dead_tuples > vacthresh:
    dovacuum = true
```
默认 GUC 来自 `guc_parameters.dat`：
```text
autovacuum_vacuum_threshold = 50
autovacuum_vacuum_scale_factor = 0.2
autovacuum_vacuum_max_threshold = 100000000
```
因此一张 `reltuples = 1000000` 的表，默认普通 vacuum threshold 约为：
```text
50 + 0.2 * 1000000 = 200050
```
如果 max threshold 默认值还没碰到，就用 200050。 如果表非常大，threshold 可能被 `autovacuum_vacuum_max_threshold` 截断。 这解释了为什么默认 scale factor 对超大表过于保守时，经常需要 per-table 调低。
### 6.7 insert vacuum threshold
insert vacuum 不是为了清理 update/delete 产生的 dead tuples。 它主要服务 insert-heavy 表上的 freezing / visibility map 推进。 公式是：
```text
vacinsthresh = autovacuum_vacuum_insert_threshold
             + autovacuum_vacuum_insert_scale_factor
               * reltuples
               * pcnt_unfrozen
```
默认 GUC：
```text
autovacuum_vacuum_insert_threshold = 1000
autovacuum_vacuum_insert_scale_factor = 0.2
```
`pcnt_unfrozen` 默认是 1。 如果 `relpages > 0` 且 `relallfrozen > 0`，源码计算：
```text
pcnt_unfrozen = 1 - relallfrozen / relpages
```
并且会先把异常的 `relallfrozen > relpages` clamp 到 `relpages`。 这意味着 all-frozen 页越多，insert vacuum 的 scale 部分越小。 它把“表总大小”转换成“未冻结活跃部分”的近似。 触发条件：
```text
if vac_ins_base_thresh >= 0:
    if av_enabled and ins_since_vacuum > vacinsthresh:
        dovacuum = true
```
如果 `vacuum_insert_threshold = -1`，insert vacuum 被禁用。 普通 vacuum 仍然可能因 dead tuples 或 forced vacuum 触发。 不要把 insert vacuum 理解成 “insert 也产生 dead tuples”。 它的目标是让纯插入表也能被周期性 VACUUM，推进 freeze 和 visibility map 状态。
### 6.8 ANALYZE threshold
ANALYZE 公式：
```text
anlthresh = autovacuum_analyze_threshold
          + autovacuum_analyze_scale_factor * reltuples
```
默认 GUC：
```text
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1
```
触发条件：
```text
if relid != pg_statistic
   and relkind != RELKIND_TOASTVALUE
   and av_enabled
   and mod_since_analyze > anlthresh:
       doanalyze = true
```
这里的 counter 是 `mod_since_analyze`。 它包括 insert、update、delete。 因此纯 insert 表可能同时触发 insert vacuum 和 analyze。 update-heavy 表也可能同时触发 vacuum 和 analyze。 TOAST table 不做 analyze。 `pg_statistic` 也被排除。
### 6.9 候选列表、排序与 recheck
`do_autovacuum()` 第一轮扫描中，如果 `dovacuum || doanalyze`，会把 relation OID 和 `scores.max` 放入 `tables_to_process`。 收集完成后，如果 score weights 不全为 0，会排序：
```text
list_sort(tables_to_process, TableToProcessComparator);
```
排序不是执行保证。 真正执行前，worker 会进入 table claim 协议。 它持有 `AutovacuumScheduleLock`，检查其他 running worker 是否已经 claim 同一张表。 如果没有，就写入：
```text
MyWorkerInfo->wi_tableoid = relid;
MyWorkerInfo->wi_sharedrel = isshared;
```
释放 schedule lock 后，再调用：
```text
table_recheck_autovac()
```
recheck 会重新读取 `pg_class` tuple、reloptions 和 pgstat entry，再调用一次 `relation_needs_vacanalyze()`。 如果这时已经不需要维护，就清空 `wi_tableoid` 并跳过。 这条路径是理解 autovacuum 的关键：
```text
first pass:
  近似发现候选

claim:
  避免多个 worker 重复维护同一 relation

recheck:
  避免用过期 stats 执行已经不需要的工作

vacuum/analyze:
  真正进入维护模块
```
## 7. reloptions 与 GUC override 规则
### 7.1 GUC 默认值与生效边界
本地源码的默认值在 `guc_parameters.dat`。
| GUC | 默认值 | context |
| --- | --- | --- |
| `autovacuum_vacuum_threshold` | `50` | `PGC_SIGHUP` |
| `autovacuum_vacuum_scale_factor` | `0.2` | `PGC_SIGHUP` |
| `autovacuum_vacuum_max_threshold` | `100000000` | `PGC_SIGHUP` |
| `autovacuum_vacuum_insert_threshold` | `1000` | `PGC_SIGHUP` |
| `autovacuum_vacuum_insert_scale_factor` | `0.2` | `PGC_SIGHUP` |
| `autovacuum_analyze_threshold` | `50` | `PGC_SIGHUP` |
| `autovacuum_analyze_scale_factor` | `0.1` | `PGC_SIGHUP` |
| `autovacuum_freeze_max_age` | `200000000` | `PGC_POSTMASTER` |
| `autovacuum_multixact_freeze_max_age` | `400000000` | `PGC_POSTMASTER` |
SIGHUP 参数可以 reload。 但 worker 不是每行都重新读配置。 `do_autovacuum()` 在处理每张 collected table 前会检查 `ConfigReloadPending`。 即使 reload 后发现 `autovacuum=off`，worker 也不会简单退出。 源码注释强调不能这样做，因为当前 worker 可能是 wraparound emergency worker。
### 7.2 reloptions 的合法范围
`reloptions.c` 注册了 autovacuum 相关 storage parameters。 几个范围值得记住：
```text
autovacuum_vacuum_threshold:
  default -1 in reloptions parser, min 0

autovacuum_vacuum_max_threshold:
  default -2 in reloptions parser, min -1

autovacuum_vacuum_insert_threshold:
  default -2 in reloptions parser, min -1

autovacuum_analyze_threshold:
  default -1 in reloptions parser, min 0

scale factors:
  default -1, min 0.0, max 100.0
```
这个“parser default”不是用户看到的 GUC default。 它是 reloption 未设置时写入 `StdRdOptions` 的 sentinel。 例如一张表没有设置 `autovacuum_vacuum_scale_factor`，解析后的字段是 `-1`。 `relation_needs_vacanalyze()` 看到 `-1`，才转向 `autovacuum_vac_scale` GUC。
### 7.3 per-table 覆盖示例
大表常见配置：
```sql
ALTER TABLE events SET (
    autovacuum_vacuum_threshold = 10000,
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_analyze_threshold = 10000,
    autovacuum_analyze_scale_factor = 0.02
);
```
这个配置不是“让 autovacuum 更频繁”这么简单。 它改变的是两个公式里的 base 和 scale。 如果 `reltuples = 100000000`，普通 vacuum threshold 是：
```text
10000 + 0.01 * 100000000 = 1010000
```
默认会是：
```text
50 + 0.2 * 100000000 = 20000050
```
这就是为什么大表通常不能只靠默认 scale factor。
### 7.4 禁用与 forced vacuum
per-table 禁用：
```sql
ALTER TABLE t SET (autovacuum_enabled = false);
```
源码语义是：
```text
av_enabled = relopts ? relopts->enabled : true;
av_enabled &= AutoVacuumingActive();
```
普通 threshold 触发依赖 `av_enabled`。 但是 forced vacuum 不依赖普通 threshold。 如果 `relfrozenxid` 或 `relminmxid` 接近危险边界，`force_vacuum` 会设置 `dovacuum = true`。 因此：
```text
autovacuum_enabled=false
  可以阻止普通 dead tuple / insert / analyze threshold 触发；
  不能阻止 anti-wraparound vacuum。
```
这是 correctness 边界。 如果允许用户完全关闭 wraparound vacuum，系统可能失去事务 ID 可用性。
## 8. TOAST、partition、inheritance 与 relkind 边界
### 8.1 TOAST table 是独立触发对象
`do_autovacuum()` 有两遍 `pg_class` 扫描。 第一遍处理 main heap / matview。 第二遍处理 `RELKIND_TOASTVALUE`。 这样做的理由写在源码注释里：
```text
short, wide tables may produce proportionally more activity in the TOAST table
```
也就是说，main table 行数可能不多，但 TOAST table 可能非常活跃。 如果只在 parent 表触发时顺便处理 TOAST，就会错过这种 workload。 autovacuum worker 调用 `vacuum()` 时没有设置 `VACOPT_PROCESS_TOAST`。 它把 main table 和 TOAST table 分开调度。
### 8.2 TOAST reloptions 继承边界
TOAST table 有自己的 `pg_class.reloptions`。 如果它没有设置，autovacuum 会尝试使用 main table 的 autovacuum reloptions。 这不是 SQL inheritance。 这是 `do_autovacuum()` 为 TOAST second pass 构建的 `table_toast_map`。 边界如下：
```text
TOAST has own reloptions:
  use TOAST reloptions

TOAST has no reloptions, parent has autovacuum reloptions:
  use parent AutoVacOpts copy

neither has reloptions:
  use GUC
```
TOAST 默认不做 ANALYZE。 `heap_reloptions()` 对 `RELKIND_TOASTVALUE` 设置：
```text
analyze_threshold = -1
analyze_scale_factor = -1
```
`relation_needs_vacanalyze()` 也显式排除 `RELKIND_TOASTVALUE` 的 analyze。 因此 TOAST 的触发重点是 VACUUM，不是 planner stats。
### 8.3 partitioned table 的边界
当前本地 `autovacuum.c` 的 candidate scan 不包含 `RELKIND_PARTITIONED_TABLE`。 第一遍只处理：
```text
RELKIND_RELATION
RELKIND_MATVIEW
```
第二遍只处理：
```text
RELKIND_TOASTVALUE
```
这意味着本节的 threshold 判断主要作用在有物理存储的 child partitions、普通 heap、matview 和 TOAST table 上。 partitioned parent 的 `reltuples`、继承统计和 recursive analyze 是另一个边界。 `analyze.c` 里有 recursive analyze 的逻辑。 但本节这条 autovacuum candidate scan 不是从 partitioned parent 递归展开。 因此诊断分区表时要问两个问题：
```text
哪个 child partition 的 pg_stat counter 超过 threshold？
partitioned parent 的统计信息是否需要手工或其它路径更新？
```
不要把 parent reloptions 想当然地等同于所有 child 的 autovacuum 触发参数。 child relation 自己的 `pg_class.reloptions` 和 GUC 才是 worker 判定时直接读取的输入。
### 8.4 inheritance 不是 threshold 聚合
传统 inheritance tree 也不要理解成“父表聚合所有子表计数后触发一次 vacuum”。 autovacuum worker 扫描的是 relation 条目。 每个物理 relation 有自己的 pgstat entry 和 `pg_class` 估算。 普通 VACUUM 的对象仍然是具体 relation。 如果 workload 把写入集中到某几个 child，它们应该分别触发。 父表没有物理 heap activity 时，不应期待父表的 `n_dead_tup` 代表整个 inheritance tree。
## 9. 生命周期 / ownership / cleanup
### 9.1 谁创建 counter
relation stats entry 由 cumulative stats system 管理。 DML backend 通过 pgstat relation instrumentation 记录本事务或本 backend 的局部计数。 这些局部计数最后合并到 shared stats entry。 因此 counter 的 owner 不是 autovacuum worker。 autovacuum worker 只是消费者。 这很重要。 如果 pgstat 没有启用或没有 entry，普通 threshold 判断无法成立。 `autovacuum` 还要求 `track_counts`。
### 9.2 谁持有 reloptions
SQL 层的 reloptions 存在 `pg_class.reloptions`。 relcache 可把它解析到 `rd_options`。 autovacuum 扫描 `pg_class` tuple 时没有打开目标 relation。 它用 `extractRelOptions()` 直接解析 tuple 里的 reloptions，并复制 `AutoVacOpts`。 复制出来的 `AutoVacOpts` 是 worker 当前 memory context 中的短生命周期对象。 普通表扫描后会 `pfree(relopts)`。 TOAST second pass 如果用的是 parent map 中的 `AutoVacOpts`，不需要释放。 这就是 `free_relopts` 标志存在的原因。
### 9.3 谁持有候选列表
`tables_to_process` 位于 `AutovacMemCxt`。 worker 要跨事务处理多张表。 因此 `do_autovacuum()` 一开始创建：
```text
AutovacMemCxt = AllocSetContextCreate(TopMemoryContext,
                                      "Autovacuum worker",
                                      ALLOCSET_DEFAULT_SIZES);
```
候选列表不是 shared memory 队列。 它只属于当前 worker。 如果 worker 退出，列表消失。 下一轮 launcher / worker 会重新扫描。
### 9.4 谁释放执行期资源
每张表执行前会创建或 reset `PortalContext`。 `autovacuum_do_vac_analyze()` 内部又创建 `vac_context`，传给 `vacuum()`。 执行完成后删除 `vac_context`。 如果一张表的 VACUUM / ANALYZE 抛错，`PG_CATCH()` 会：
```text
AbortOutOfAnyTransaction();
FlushErrorState();
MemoryContextReset(PortalContext);
StartTransactionCommand();
```
然后继续处理下一张候选表。 这说明一张表失败不必杀死整个 worker。 但是失败表的统计 counter 也不会被成功路径重置。 下一轮可能再次触发。
### 9.5 counter 如何收尾
成功 VACUUM 后：
```text
dead_tuples 被新估算覆盖；
ins_since_vacuum 清零；
last_autovacuum_time 更新。
```
成功 ANALYZE 后：
```text
live_tuples / dead_tuples 被新估算覆盖；
mod_since_analyze 在 resetcounter 为真时清零；
last_autoanalyze_time 更新。
```
如果 VACUUM 和 ANALYZE 同时触发，`vacuum()` 会以 `VACOPT_VACUUM | VACOPT_ANALYZE` 进入维护路径。 但两组 counter 的重置仍由各自 report 函数完成。
## 10. 正确性机制层次
本节的 correctness 不靠一个机制单独完成。 它由几层近似和硬边界组合而成。
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| pgstat | per-relation cumulative counters | 有低成本 workload 信号 | 实时精确 heap 状态 |
| catalog | `pg_class.reltuples` / `relpages` / `relallfrozen` | 有 relation 大小估算 | 当前行数精确值 |
| reloptions | `AutoVacOpts` sentinel 规则 | per-table 策略覆盖 | 用户配置一定合理 |
| worker scan | `do_autovacuum()` 扫 `pg_class` | 定期发现候选 relation | 全局强一致队列 |
| schedule lock | `AutovacuumScheduleLock` claim table | 避免 worker 重复处理同一 relation | 阻止手工 VACUUM 或所有并发 DDL |
| recheck | `table_recheck_autovac()` | 降低 stale candidate 执行概率 | 完全消除 race |
| relation lock | VACUUM / ANALYZE 自己的锁策略 | 执行阶段对象安全 | threshold 判断阶段对象不变化 |
| forced vacuum | XID / MultiXact age | 防止 wraparound correctness risk | 保证低延迟或低 IO |
几个不变量：
```text
threshold 判断只产生候选，不是 durable promise。
执行前必须 recheck。
普通 autovacuum 可以被 reloption 禁用。
anti-wraparound vacuum 不能被普通禁用逻辑阻止。
TOAST table vacuum 独立判断。
ANALYZE 不处理 TOAST table。
```
`autovacuum_enabled=false` 是策略边界，不是 correctness override。 `track_counts=off` 会让 autovacuum 失去普通 threshold 输入。 但是系统仍必须有 anti-wraparound 兜底。
## 11. 错误路径 / 异常路径 / fallback
### 11.1 stats entry 不存在
`pgstat_fetch_stat_tabentry_ext()` 可能返回 `NULL`。 此时普通 threshold 不能判断。 函数会直接返回。 但是 forced vacuum 已经在此前计算。 因此：
```text
normal threshold:
  needs pgstat entry

wraparound threshold:
  can survive missing pgstat entry
```
诊断时，如果 `pg_stat_all_tables` 中某张表统计缺失或刚 reset，不要期待普通 autovacuum 立即基于精确计数触发。
### 11.2 `reltuples < 0`
新 relation 或统计未知时，`pg_class.reltuples` 可能为负。 源码处理：
```text
if (reltuples < 0)
    reltuples = 0;
```
这会让 scale factor 部分归零。 触发主要由 base threshold 决定。 这对小表和新表很重要。 一个刚建的大表在第一次 ANALYZE / VACUUM 前，`reltuples` 估算可能不是你以为的当前行数。
### 11.3 `relallfrozen > relpages`
理论上 `relallfrozen` 不应大于 `relpages`。 但源码考虑了手工更新 stats 或异常统计造成的不一致。 处理方式是 clamp：
```text
relallfrozen = Min(relallfrozen, relpages);
```
这样 `pcnt_unfrozen` 不会变成负数。 这是典型的 defensive stats handling。 统计不可信时，系统宁可退化成保守近似，也不能让阈值公式产生荒谬结果。
### 11.4 insert vacuum 被禁用
当 `vac_ins_base_thresh < 0` 时，insert vacuum 分支不参与触发。 score 日志里会显示 insert disabled。 这不会禁用普通 vacuum。 也不会禁用 forced vacuum。 如果你把 insert vacuum 禁掉，纯 append-only 大表可能更依赖 anti-wraparound vacuum 才推进 freeze。 这可能让未来某次 VACUUM 变得更重。
### 11.5 autovacuum reload 与 emergency worker
worker 处理每张表前会响应配置 reload。 但它不会因为 `autovacuum` 变成 off 就无条件退出。 原因是当前 worker 可能为了 wraparound 而启动。 这体现了两层优先级：
```text
operational preference:
  是否做普通后台维护

correctness requirement:
  必须避免 XID/MultiXact wraparound
```
### 11.6 单表执行失败
如果某张表在 `autovacuum_do_vac_analyze()` 中报错，worker 捕获错误、abort 当前事务、reset per-table context，然后继续下一张候选表。 这条 fallback 对大 database 很重要。 一个坏表、锁冲突或临时错误不应该让整个 autovacuum worker 放弃所有候选。 但失败表自身没有完成 counter reset。 因此它可能下一轮继续出现在候选里。
## 12. 成本、资源与跨模块传播
### 12.1 threshold 太低的成本
降低 threshold 和 scale factor 会更早触发维护。 代价不是常数。 它会沿多个资源传播：
```text
更多 worker 执行机会
  -> 更多 heap scan / index vacuum / analyze sample
  -> 更多 buffer cache churn
  -> 更多 WAL for visibility/freeze/page cleanup
  -> 更多 IO queue 占用
  -> 更多 relation lock 尝试
```
在 update-heavy 表上，太低的 `autovacuum_vacuum_scale_factor` 可能让后台维护频繁打断前台 workload。 在 insert-heavy 表上，太低的 insert threshold 可能让纯插入表过早进入 VACUUM。 在 stats-sensitive OLTP 表上，太低的 analyze threshold 可能让 sampling 成本增加，但 planner stats 更及时。
### 12.2 threshold 太高的成本
threshold 太高也不是“省资源”。 它会把成本延后并放大：
```text
dead tuples accumulate
  -> heap and index bloat
  -> more pages scanned by queries
  -> later VACUUM has more work
  -> freeze horizon pressure increases
  -> emergency vacuum 更难避开高峰
```
ANALYZE threshold 太高会让 planner 使用 stale stats。 这可能导致 join order、index choice、parallelism 和 rowcount estimate 失真。 最终成本可能表现为 query latency，而不是 autovacuum latency。
### 12.3 relation 数量放大
autovacuum worker 在 database 内扫描 `pg_class`。 候选判断成本随 relation 数增加。 分区过多时，即使每个 partition 很小，worker 也要逐 relation 读取 catalog tuple、reloptions 和 pgstat entry。 这类成本不是单张大表的 heap scan 成本。 它是 metadata fan-out 成本。
### 12.4 pgstat 滞后带来的决策漂移
pgstat 是低成本统计系统。 它不是每条 DML 同步写 shared state。 因此 threshold 判断可能看到滞后的 counter。 影响包括：
```text
刚超过 threshold 但尚未 flush:
  worker 可能本轮看不到

刚被手工 VACUUM/ANALYZE 处理:
  first pass 可能已收集候选，但 recheck 会跳过

aborted inserts 被 ins_since_vacuum 计入:
  insert vacuum 可能比理想语义更早触发
```
这类漂移是设计取舍。 如果每次 DML 都同步强一致维护触发状态，前台写路径成本会不可接受。
### 12.5 跨模块连接
本节至少连接六个模块。 `pgstat` 提供 counters。 `pg_class` 提供 relation size 和 age 估算。 `reloptions` 提供 per-table policy。 `autovacuum` 提供 worker scan、candidate sort、claim 和 recheck。 `VACUUM` 执行 heap/index cleanup、freeze、visibility map 推进，并回写 stats。 `ANALYZE` 执行 sampling、统计生成，并回写 `pg_class.reltuples` 和 analyze stats。 这些模块的边界要分清：
```text
pgstat:
  告诉 autovacuum workload 发生了什么，但不执行维护。

autovacuum:
  决定是否启动维护，但不直接清 heap page。

VACUUM:
  清理和 freeze，但不生成 planner column statistics。

ANALYZE:
  生成 planner statistics，但不推进 insert vacuum counter。

reloptions:
  改变策略输入，但不创建独立 scheduler。
```
## 13. 观测与诊断入口
### 13.1 能直接看到的状态
`pg_stat_all_tables`：
```sql
SELECT relid::regclass, n_live_tup, n_dead_tup,
       n_ins_since_vacuum, n_mod_since_analyze,
       last_autovacuum, last_autoanalyze
FROM pg_stat_all_tables
WHERE relid = 'av_threshold_demo'::regclass;
```
`pg_class`：
```sql
SELECT oid::regclass, reltuples, relpages, relallfrozen,
       age(relfrozenxid) AS xid_age,
       mxid_age(relminmxid) AS mxid_age,
       reloptions
FROM pg_class
WHERE oid = 'av_threshold_demo'::regclass;
```
reloptions 展开：
```sql
SELECT * FROM pg_options_to_table(
    (SELECT reloptions FROM pg_class WHERE oid = 'av_threshold_demo'::regclass));
```
本地源码新增或已有的 score 函数：
```sql
SELECT c.oid::regclass, s.score, s.vacuum_score,
       s.vacuum_insert_score, s.analyze_score,
       s.do_vacuum, s.do_analyze, s.for_wraparound
FROM pg_stat_get_autovacuum_scores() AS s
JOIN pg_class AS c ON c.oid = s.oid
WHERE c.oid = 'av_threshold_demo'::regclass;
```
这是本节最贴近源码的观测入口。 它内部也调用 `relation_needs_vacanalyze()`。
### 13.2 只能间接推断的状态
这些状态不能从一个 SQL 列完整看出：
```text
worker first pass 时看到的 pgstat snapshot
tables_to_process 的候选列表
执行前 recheck 是否改变结果
某个 counter 中有多少来自 aborted inserts
dead_tuples 与 heap page 上 LP_DEAD 的实时差距
当前 worker 使用的是 TOAST 自身 reloptions 还是 parent reloptions copy
```
需要用日志、断点或源码插桩观察。
### 13.3 运行中 worker
看当前 autovacuum worker：
```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type LIKE 'autovacuum%';
```
看 VACUUM progress：
```sql
SELECT pid,
       datname,
       relid::regclass,
       phase,
       heap_blks_total,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       num_dead_item_ids
FROM pg_stat_progress_vacuum;
```
progress 只能说明维护正在执行。 它不能说明为什么触发。 触发原因要回到 counters、threshold、reloptions 和 score。
### 13.4 日志入口
设置：
```sql
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
ALTER SYSTEM SET log_autoanalyze_min_duration = 0;
SELECT pg_reload_conf();
```
或者 per-table：
```sql
ALTER TABLE av_threshold_demo SET (
    log_autovacuum_min_duration = 0,
    log_autoanalyze_min_duration = 0
);
```
日志会告诉你实际执行了 autovacuum 或 autoanalyze。 但日志不直接给出当时完整 threshold 公式。 如果需要源码级验证，可以把 server log level 调到能看到 `relation_needs_vacanalyze()` 的 `DEBUG3` 输出。 那条输出包含：
```text
vac tuples, vacuum threshold, vacuum score
insert tuples, insert threshold, insert score
analyze tuples, analyze threshold, analyze score
xid score, mxid score
```
生产环境不要轻易打开大量 DEBUG3。 它可能产生很大日志量。
## 14. 常见误区
### 14.1 把 `n_dead_tup` 当作精确 dead tuple 数
`n_dead_tup` 是 stats estimate。 它不是 heap scan 的实时结果。 VACUUM、ANALYZE、DML flush、abort、pruning 都会让它和真实 page state 产生偏差。 诊断时要把它当触发输入，而不是最终事实。
### 14.2 认为 `scale_factor=0` 等于禁用
`scale_factor=0` 只是去掉与 `reltuples` 成比例的部分。 base threshold 仍然生效。 例如：
```text
threshold = 50 + 0 * reltuples = 50
```
如果想禁用普通 autovacuum，要看 `autovacuum_enabled=false`。 如果想禁用 insert vacuum，要用 `autovacuum_vacuum_insert_threshold=-1`。
### 14.3 忘记触发条件是严格大于
源码使用：
```text
vactuples > vacthresh
instuples > vacinsthresh
anltuples > anlthresh
```
不是 `>=`。 边界实验里，如果 threshold 设置成 5，counter 等于 5 不会触发。
### 14.4 把 `autovacuum_enabled=false` 当作绝对关闭
它不能阻止 anti-wraparound vacuum。 这是系统正确性边界。 如果你看到禁用 autovacuum 的表仍然被 vacuum，先检查日志中是否有 `to prevent wraparound`。
### 14.5 只看 `last_autovacuum`
`last_autovacuum` 是执行结果。 它不是触发条件。 要解释为什么触发，需要同时看：
```text
n_dead_tup
n_ins_since_vacuum
n_mod_since_analyze
reltuples
relpages
relallfrozen
reloptions
GUC
score / do_vacuum / do_analyze
```
### 14.6 认为 ANALYZE 会清掉 insert vacuum counter
ANALYZE 清 `mod_since_analyze`。 VACUUM 清 `ins_since_vacuum`。 两者的收尾函数不同。 纯 insert 表在 autoanalyze 后仍然可能因为 `n_ins_since_vacuum` 触发 insert vacuum。
## 15. 课堂实验
### 实验 1：三个 counter 分别触发
目标：证明普通 vacuum、insert vacuum 和 analyze 由不同 counter 驱动。 步骤：
```sql
DROP TABLE IF EXISTS av_counter_lab;
CREATE TABLE av_counter_lab(id int primary key, payload text);

ALTER TABLE av_counter_lab SET (
    autovacuum_vacuum_threshold = 3,
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_vacuum_insert_threshold = 3,
    autovacuum_vacuum_insert_scale_factor = 0,
    autovacuum_analyze_threshold = 3,
    autovacuum_analyze_scale_factor = 0
);

INSERT INTO av_counter_lab
SELECT g, repeat('x', 100)
FROM generate_series(1, 20) AS g;

VACUUM ANALYZE av_counter_lab;
```
分别执行：
```sql
DELETE FROM av_counter_lab WHERE id BETWEEN 1 AND 4;

INSERT INTO av_counter_lab
SELECT g, repeat('y', 100)
FROM generate_series(100, 103) AS g;

UPDATE av_counter_lab
SET payload = payload || 'z'
WHERE id BETWEEN 10 AND 13;
```
观察：
```sql
SELECT n_dead_tup,
       n_ins_since_vacuum,
       n_mod_since_analyze
FROM pg_stat_all_tables
WHERE relid = 'av_counter_lab'::regclass;

SELECT *
FROM pg_stat_get_autovacuum_scores()
WHERE oid = 'av_counter_lab'::regclass;
```
源码回扣：
```text
pgstat_relation_flush_cb_locked()
  -> dead_tuples / ins_since_vacuum / mod_since_analyze
relation_needs_vacanalyze()
  -> vactuples / instuples / anltuples
```
预期结论：
```text
同一组 DML 可能同时推进多个 counter；
但三个 threshold 的触发语义不同。
```
### 实验 2：严格大于 threshold
目标：验证 `>` 而不是 `>=`。 步骤：
```sql
DROP TABLE IF EXISTS av_gt_lab;
CREATE TABLE av_gt_lab(id int primary key, payload text);

ALTER TABLE av_gt_lab SET (
    autovacuum_vacuum_threshold = 5,
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_analyze_threshold = 5,
    autovacuum_analyze_scale_factor = 0
);

INSERT INTO av_gt_lab
SELECT g, repeat('x', 50)
FROM generate_series(1, 20) AS g;

VACUUM ANALYZE av_gt_lab;

DELETE FROM av_gt_lab WHERE id BETWEEN 1 AND 5;
```
观察 score。 然后再删除第 6 行。
```sql
DELETE FROM av_gt_lab WHERE id = 6;
```
再次观察 score。 源码回扣：
```text
if (av_enabled && vactuples > vacthresh)
    *dovacuum = true;
```
预期结论：
```text
counter 等于 threshold 时不触发；
超过 threshold 才触发。
```
### 实验 3：TOAST 独立触发
目标：观察宽行表的 TOAST relation 有自己的 autovacuum 判定。 步骤：
```sql
DROP TABLE IF EXISTS av_toast_lab;
CREATE TABLE av_toast_lab(id int primary key, payload text);

ALTER TABLE av_toast_lab SET (
    autovacuum_vacuum_threshold = 2,
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_analyze_threshold = 2,
    autovacuum_analyze_scale_factor = 0
);

INSERT INTO av_toast_lab
SELECT g, repeat(md5(g::text), 1000)
FROM generate_series(1, 20) AS g;

VACUUM ANALYZE av_toast_lab;

UPDATE av_toast_lab
SET payload = payload || repeat('x', 1000)
WHERE id BETWEEN 1 AND 5;
```
找到 TOAST relation：
```sql
SELECT c.oid::regclass AS main_rel,
       t.oid::regclass AS toast_rel,
       t.reloptions AS toast_reloptions
FROM pg_class AS c
JOIN pg_class AS t ON t.oid = c.reltoastrelid
WHERE c.oid = 'av_toast_lab'::regclass;
```
观察 score：
```sql
SELECT c.oid::regclass,
       c.relkind,
       s.vacuum_score,
       s.analyze_score,
       s.do_vacuum,
       s.do_analyze
FROM pg_stat_get_autovacuum_scores() AS s
JOIN pg_class AS c ON c.oid = s.oid
WHERE c.oid IN (
    'av_toast_lab'::regclass,
    (SELECT reltoastrelid FROM pg_class WHERE oid = 'av_toast_lab'::regclass)
);
```
预期结论：
```text
TOAST table 可以独立 vacuum；
TOAST table 不 autoanalyze；
TOAST 没有自己的 reloptions 时可能使用 main table 的 AutoVacOpts。
```
## 16. 讨论题
1. 为什么 PostgreSQL 不在每次 DML 后立即判断 threshold 并唤醒 worker？
2. `n_dead_tup` 高但 `do_vacuum=false`，可能有哪些原因？
3. `scale_factor=0` 和 `autovacuum_enabled=false` 的语义差异是什么？
4. 为什么 insert vacuum threshold 要乘 `pcnt_unfrozen`，而普通 vacuum threshold 不这么做？
5. TOAST table 为什么需要单独 threshold 判断，而不是永远随 main table vacuum？
6. 如果一张表的 `reltuples` 严重低估，会如何影响 vacuum/analyze 触发频率？
7. 为什么 `table_recheck_autovac()` 不能完全消除 race，但仍然值得存在？
8. `pg_stat_get_autovacuum_scores()` 能替代日志或 progress view 吗？它能看到什么，看不到什么？
## 17. 本节小结
本节的核心链路是：
```text
DML 产生 relation stats counters
  -> pgstat flush 到 PgStat_StatTabEntry
  -> worker 扫 pg_class 读取 reltuples / relpages / relallfrozen / reloptions
  -> relation_needs_vacanalyze() 计算 threshold 和 score
  -> do_autovacuum() 收集候选并排序
  -> worker claim table
  -> table_recheck_autovac() 复查
  -> vacuum()/analyze() 执行
  -> pgstat_report_vacuum()/pgstat_report_analyze() 重置对应 counter
```
三个 threshold 的核心公式：
```text
ordinary vacuum:
  dead_tuples > vacuum_threshold + vacuum_scale_factor * reltuples
  optionally capped by vacuum_max_threshold

insert vacuum:
  ins_since_vacuum > vacuum_insert_threshold
                    + vacuum_insert_scale_factor * reltuples * pcnt_unfrozen

analyze:
  mod_since_analyze > analyze_threshold + analyze_scale_factor * reltuples
```
核心状态和边界：
```text
PgStat_StatTabEntry:
  workload counters, approximate and flush-based

pg_class:
  relation size and age estimates

AutoVacOpts:
  per-table policy override with sentinel values

tables_to_process:
  worker-local candidate list, not durable queue

WorkerInfoData.wi_tableoid:
  shared claim state, not trigger reason
```
ownership / cleanup：
```text
DML backend owns local stats production；
pgstat owns shared counters；
autovacuum worker owns candidate list；
VACUUM resets ins_since_vacuum；
ANALYZE resets mod_since_analyze；
ERROR during one table aborts that table's transaction and continues with the next candidate。
```
错误路径：
```text
missing pgstat entry skips normal threshold but not forced vacuum；
reltuples < 0 becomes 0；
relallfrozen > relpages is clamped；
insert vacuum can be disabled with threshold -1；
autovacuum_enabled=false does not block anti-wraparound vacuum。
```
观测入口：
```text
pg_stat_all_tables:
  counters and last execution time

pg_class:
  reltuples, relpages, relallfrozen, reloptions, xid/mxid age

pg_stat_get_autovacuum_scores():
  current relation_needs_vacanalyze() result for this database

pg_stat_progress_vacuum / pg_stat_progress_analyze:
  execution progress, not trigger cause

logs:
  actual execution, sometimes debug threshold details
```
可迁移的系统规律：
```text
后台维护触发通常不是强一致实时判定；
它是低成本统计信号、对象大小估算、局部策略覆盖和执行前 recheck 的组合。
```
判断 autovacuum 触发问题时，不要只问“为什么没 vacuum”。 要按顺序问：
```text
counter 有没有 flush？
reltuples 是否合理？
threshold 是否被 reloptions 覆盖？
是否被 strict greater-than 卡在边界？
是否是 TOAST / partition / temp table 边界？
是否被 autovacuum_enabled 禁用但又被 wraparound 强制？
worker 是否已经收集候选但 recheck 后跳过？
```
下一步如果继续沿 autovacuum 主线推进，应该把本节的 threshold 判断和后续 anti-wraparound / freeze emergency 细节接起来，分析普通维护和 correctness vacuum 如何在同一个 worker pipeline 中共存。
