# PostgreSQL autovacuum freeze / wraparound priority
## 课程定位
前置知识：
已经理解 XID / MultiXact wraparound、heap tuple freeze、visibility map all-frozen、
以及 autovacuum launcher / worker 的基本调度。
本节唯一主问题：
```text
freeze age、multixact age 和 wraparound danger
如何让 anti-wraparound vacuum 压过普通成本策略？
```
核心矛盾：
普通 autovacuum 应该服从 threshold、score、worker balance 和 cost delay，
避免后台维护抢占前台 workload。
但 XID 和 MultiXact age 是 correctness horizon。
一旦 relation 或 database 的 frozen horizon 太旧，
系统必须把 vacuum 从“收益型维护”提升为“防止 wraparound 的安全任务”。
学完后应能判断：
为什么 `autovacuum_enabled=false` 不能阻止 anti-wraparound vacuum；
为什么 `relfrozenxid` 和 `relminmxid` 都会强制 vacuum；
为什么 `is_wraparound`、`aggressive` 和 `failsafe` 不是同一个状态；
为什么 wraparound vacuum 一开始仍可使用 cost delay；
为什么 failsafe 触发后会停止 cost delay、放弃 buffer access strategy、
并跳过 index vacuuming、index cleanup 和 heap truncation；
为什么诊断必须同时看 `pg_database`、`pg_class`、progress view、日志和 horizon holder。
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
第 29 节已经讲过 freeze：
旧 tuple 的 `xmin`、`xmax` 或 MultiXact 引用，
什么时候能被 VACUUM 改写或标记为 frozen。
第 39 节已经讲过 autovacuum 调度：
launcher 选 database，worker 连接 database 后扫 `pg_class` 选 relation。
本节把两者接起来：
当 `relfrozenxid` 或 `relminmxid` 太旧时，
autovacuum 如何把一张表从普通维护队列提升为 anti-wraparound 任务。
本节不重复 lazy VACUUM 的全部 heap / index cleanup。
本节也不展开普通 threshold / scale factor。
这些只在解释优先级和成本边界时出现。
本节 runtime 现象：
一张表可以没有很多 dead tuples，
但因为 `age(relfrozenxid)` 或 `mxid_age(relminmxid)` 太大，
仍被 autovacuum 处理。
`pg_stat_activity` 的 activity string 可能带有
`to prevent wraparound`。
`pg_stat_progress_vacuum.started_by` 可能是
`autovacuum_wraparound`。
日志可能出现 `automatic aggressive vacuum to prevent wraparound`
或 failsafe bypass 提示。
核心判断路径：
```text
pg_database age
  -> launcher 选 database
  -> worker 扫 pg_class
  -> relation_needs_vacanalyze() 判断 force_vacuum / wraparound / score
  -> table_recheck_autovac() 生成 VacuumParams
  -> vacuum_get_cutoffs() 决定 aggressive
  -> lazy_check_wraparound_failsafe() 决定是否停止成本节流
```
这个顺序很重要。
anti-wraparound 不是一个独立 vacuum 程序。
它沿用普通 autovacuum worker 和普通 VACUUM 执行层，
但在若干边界上改变选择、锁退避、扫描义务和成本策略。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
relation_needs_vacanalyze() 用 recentXid/recentMulti 与
pg_class.relfrozenxid/relminmxid 计算 age；
超过 freeze max age 或 effective multixact freeze max age 时，
设置 wraparound=true、dovacuum=true，并把 xid/mxid score 纳入排序；
table_recheck_autovac() 把 wraparound 写入 VacuumParams.is_wraparound；
vacuum_get_cutoffs() 根据 freeze table age 决定 aggressive；
lazy_check_wraparound_failsafe() 在执行中发现 age 已过 failsafe 边界时，
关闭 VacuumCostActive、清空 VacuumCostBalance、放弃 buffer access strategy，
并跳过非必要 index/heap 维护。
```
这条链路里有四层阈值。
第一层是普通 threshold。
它来自 dead tuple、insert count、mod count、`reltuples`、threshold 和 scale factor。
它回答：
从维护收益看，现在是否值得 vacuum 或 analyze。
第二层是 force wraparound threshold。
它来自：
```text
recentXid - freeze_max_age
recentMulti - multixact_freeze_max_age
```
它回答：
这张表是否已经不能等待普通 threshold。
第三层是 aggressive threshold。
它由 `vacuum_get_cutoffs()` 使用
`vacuum_freeze_table_age` 和 `vacuum_multixact_freeze_table_age` 计算。
它回答：
本次 VACUUM 是否必须扫描足够页面，
至少能推进 `relfrozenxid` 到 `FreezeLimit`，
推进 `relminmxid` 到 `MultiXactCutoff`。
第四层是 failsafe threshold。
它来自：
```text
Max(vacuum_failsafe_age, autovacuum_freeze_max_age * 1.05)
Max(vacuum_multixact_failsafe_age,
    autovacuum_multixact_freeze_max_age * 1.05)
```
它回答：
本次 VACUUM 是否已经危险到必须跳过非必要维护并停止 cost delay。
三个状态不要混用：
`is_wraparound` 是选择层标记。
`aggressive` 是扫描和 horizon 推进义务。
`VacuumFailsafeActive` 是执行层 emergency mode。
这三个状态通常相关。
但它们回答的问题不同。
```text
is_wraparound:
  为什么这次 autovacuum 被强制做。
aggressive:
  这次 heap vacuum 是否必须推进 relation frozen horizon。
failsafe:
  是否必须停止普通成本节流并裁掉可延后维护。
```
本节的可迁移规律：
后台任务可以用成本模型做平滑。
correctness horizon 不能被成本模型无限拖延。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/autovacuum.c` | `do_autovacuum()`、`relation_needs_vacanalyze()`、`table_recheck_autovac()`、`VacuumUpdateCosts()`。 |
| 2 | `src/include/commands/vacuum.h` | `VacuumParams`、`VacuumCutoffs`、`VACOPT_SKIP_LOCKED`、freeze/failsafe GUC。 |
| 3 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()`、`vacuum_xid_failsafe_check()`、database frozen horizon 更新。 |
| 4 | `src/backend/access/heap/vacuumlazy.c` | `heap_vacuum_rel()`、progress mode、aggressive、VM skipping、`lazy_check_wraparound_failsafe()`。 |
| 5 | `src/backend/access/heap/heapam.c` | tuple freeze 准备和应用：`heap_prepare_freeze_tuple()`、`heap_freeze_prepared_tuples()`。 |
| 6 | `src/backend/access/heap/pruneheap.c` | `heap_page_prune_and_freeze()` 把 pruning 与 freeze 放在页级操作中。 |
| 7 | `src/backend/access/transam/varsup.c` | `SetTransactionIdLimit()` 设置 `xidVacLimit`、`xidWarnLimit`、`xidStopLimit`、`xidWrapLimit`。 |
| 8 | `src/backend/access/transam/multixact.c` | `MultiXactMemberFreezeThreshold()` 根据 members 压力收缩有效 MXID freeze max age。 |
| 9 | `src/include/access/transam.h` | `VariableCacheData` 中 XID limit 状态。 |
| 10 | `src/backend/catalog/system_views.sql` | 当前本地 `pg_stat_progress_vacuum` 的 `mode` 和 `started_by` 映射。 |
推荐阅读顺序：
```text
selection:
  do_autovacuum()
  -> relation_needs_vacanalyze()
  -> TableToProcess score
  -> table_recheck_autovac()
execution:
  autovacuum_do_vac_analyze()
  -> vacuum()
  -> vacuum_rel()
  -> heap_vacuum_rel()
  -> vacuum_get_cutoffs()
  -> lazy_check_wraparound_failsafe()
global limit:
  vac_update_datfrozenxid()
  -> SetTransactionIdLimit()
  -> MultiXactMemberFreezeThreshold()
```
源码有几个容易误读的地方。
`relation_needs_vacanalyze()` 同时处理普通 threshold 和 wraparound score。
这是为了 worker 扫 `pg_class` 时一次拿到 reloptions、catalog age 和 pgstat。
`is_wraparound` 从 autovacuum 选择层传入 `VacuumParams`。
但 `heap_vacuum_rel()` 仍要再次调用 `vacuum_get_cutoffs()` 决定 aggressive。
failsafe 不在选择层决定。
它在执行过程中反复检查。
因为 VACUUM 运行期间 `nextXID` 和 `nextMXID` 可能继续前进。
## 4. 关键数据结构与状态
### 4.1 `pg_class.relfrozenxid` / `pg_class.relminmxid`
`relfrozenxid` 是 relation-level 承诺：
```text
这张表中没有比 relfrozenxid 更老的未冻结普通 XID 需要保留。
```
`relminmxid` 是 MultiXact 对应承诺：
```text
这张表中没有比 relminmxid 更老的 MultiXactId 引用需要保留。
```
它们不是“最近一次 vacuum 的时间”。
也不是页面中最老 tuple ID 的精确记录。
它们是 VACUUM 在扫描、freeze、VM skipping 和 catalog update 后能公开的下界。
普通 VACUUM 如果跳过 all-visible 但非 all-frozen 页面，
可能无法证明页面里没有更老未冻结 ID。
因此不能随便推进 `relfrozenxid` 或 `relminmxid`。
aggressive VACUUM 的语义更强：
它必须能把 relation horizon 推进到本次 cutoff 要求。
诊断入口：
```sql
SELECT n.nspname,
       c.relname,
       c.relkind,
       age(c.relfrozenxid) AS xid_age,
       mxid_age(c.relminmxid) AS mxid_age,
       c.relfrozenxid,
       c.relminmxid
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY GREATEST(age(c.relfrozenxid), mxid_age(c.relminmxid)) DESC
LIMIT 30;
```
这能看到 relation catalog horizon。
它看不到具体哪些 heap tuple 还需要 freeze。
### 4.2 `pg_database.datfrozenxid` / `pg_database.datminmxid`
database-level horizon 是该 database 中 relation horizon 的聚合。
launcher 没有连接所有 database 去扫每张表。
它先用 `pg_database` 的 `datfrozenxid` 和 `datminmxid` 选 database。
这解释了调度边界：
```text
launcher:
  只能看到 database-level danger。
worker:
  连接 database 后才能看到 relation-level danger。
```
诊断入口：
```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       mxid_age(datminmxid) AS mxid_age,
       datfrozenxid,
       datminmxid
FROM pg_database
ORDER BY GREATEST(age(datfrozenxid), mxid_age(datminmxid)) DESC;
```
如果 database age 很高，
下一步必须下钻到 `pg_class`。
常见拖后腿对象包括 TOAST 表、很少访问的大表、materialized view 和 shared catalog。
### 4.3 `AutoVacuumScores`
`AutoVacuumScores` 在 `autovacuum.c` 中返回 table score。
字段包括：
```text
max
xid
mxid
vac
vac_ins
anl
```
普通组件来自：
```text
dead_tuples / vacuum_threshold
insert_since_vacuum / insert_threshold
mod_since_analyze / analyze_threshold
```
wraparound 组件来自：
```text
xid_age / freeze_max_age
mxid_age / multixact_freeze_max_age
```
`scores.max` 用于排序。
因此 autovacuum 没有单独的全局 wraparound table queue。
它把 ordinary maintenance 和 wraparound pressure 放进同一候选列表。
但 wraparound 有额外提升：
超过 force threshold 时直接 `dovacuum=true`。
超过 effective failsafe age 时，
XID/MXID score 会被 aggressive scaling。
本地源码还有 score weight：
`autovacuum_freeze_score_weight`、
`autovacuum_multixact_freeze_score_weight`、
`autovacuum_vacuum_score_weight`、
`autovacuum_vacuum_insert_score_weight`、
`autovacuum_analyze_score_weight`。
这些 weight 影响排序。
它们不取消 wraparound correctness 底线。
### 4.4 `AutoVacOpts`
`AutoVacOpts` 来自 reloptions。
它可以覆盖普通 threshold、freeze age 和 cost 参数。
对本节最重要的是：
```text
freeze_max_age =
  reloption exists
    ? Min(reloption, autovacuum_freeze_max_age)
    : autovacuum_freeze_max_age
multixact_freeze_max_age =
  reloption exists
    ? Min(reloption, effective_multixact_freeze_max_age)
    : effective_multixact_freeze_max_age
```
per-table 设置可以让某表更早触发。
不能让它比全局安全边界更晚触发。
`autovacuum_enabled=false` 只关闭普通 autovacuum。
如果 `force_vacuum=true`，
`relation_needs_vacanalyze()` 仍会设置 `dovacuum=true`。
这是本节的关键 operational 结论：
per-table disable 不能作为 wraparound protection 的开关。
### 4.5 `VacuumParams`
`VacuumParams` 是 autovacuum 选择层传给 VACUUM 执行层的合同。
本节关注字段：
```c
typedef struct VacuumParams
{
    uint32 options;
    int freeze_min_age;
    int freeze_table_age;
    int multixact_freeze_min_age;
    int multixact_freeze_table_age;
    bool is_wraparound;
    int log_vacuum_min_duration;
    VacOptValue index_cleanup;
    VacOptValue truncate;
    double max_eager_freeze_failure_rate;
    int nworkers;
} VacuumParams;
```
`is_wraparound` 影响 progress 和日志。
它也影响 autovacuum 选项：
`table_recheck_autovac()` 在非 wraparound 时加入 `VACOPT_SKIP_LOCKED`，
wraparound 时不加入。
这意味着 anti-wraparound vacuum 不使用普通 skip-locked 退避。
它仍会受 relation locks、buffer locks、page pins 和事务语义约束。
它不是无锁 vacuum。
### 4.6 `VacuumCutoffs`
`VacuumCutoffs` 是 `vacuum_get_cutoffs()` 的输出。
核心字段：
```text
relfrozenxid
relminmxid
OldestXmin
OldestMxact
FreezeLimit
MultiXactCutoff
```
`OldestXmin` 来自 `GetOldestNonRemovableTransactionId(rel)`。
它保护仍可能看见旧版本的事务和其它 cleanup horizon holder。
`FreezeLimit` 来自 `nextXID - freeze_min_age`。
但源码强制：
```text
FreezeLimit <= OldestXmin
```
`MultiXactCutoff` 同理不能超过 `OldestMxact`。
这说明 anti-wraparound priority 不会破坏 MVCC。
长事务或 stale slot 可以让 VACUUM 被迫保守。
### 4.7 `LVRelState`
`LVRelState` 是 lazy VACUUM 的单 relation 执行状态。
本节关注：
```text
aggressive
cutoffs
NewRelfrozenXid
NewRelminMxid
skipwithvm
skippedallvis
do_index_vacuuming
do_index_cleanup
do_rel_truncate
bstrategy
num_index_scans
```
`NewRelfrozenXid` 和 `NewRelminMxid` 是准备写回 `pg_class` 的新 horizon。
如果普通 VACUUM 因 VM 跳过页面而无法证明所有旧 ID 已处理，
`skippedallvis` 会阻止推进 horizon。
failsafe 触发后，
`do_index_vacuuming`、`do_index_cleanup`、`do_rel_truncate` 被关闭，
`bstrategy` 被设为 NULL，
`VacuumCostActive` 被关掉。
这表示系统放弃可延后维护，
把剩余吞吐集中到必须完成的 freeze / horizon 推进。
## 5. 主流程源码 walkthrough
### 5.1 launcher 先选 database
launcher 不能全局扫描所有 relation。
它先读 `pg_database`。
`do_start_worker()` 使用 database-level frozen horizon 判断风险。
XID 风险优先于 MultiXact 风险。
如果有 database 已接近 XID wraparound，
launcher 优先给它启动 worker。
这一步只决定 database。
具体 relation 仍由 worker 在连接后选择。
### 5.2 worker 扫 `pg_class`
`do_autovacuum()` 创建 `AutovacMemCxt`，
启动事务，
打开 `pg_class`，
第一遍收集普通表和 materialized view，
第二遍处理 TOAST 表。
每个 candidate 调用：
```text
relation_needs_vacanalyze(relid, relopts, classForm,
                          effective_multixact_freeze_max_age,
                          DEBUG3,
                          &dovacuum, &doanalyze, &wraparound,
                          &scores)
```
TOAST 表不能 analyze。
但 TOAST 表可以因为 XID/MultiXact age 被 vacuum。
诊断 database age 时不能忽略 `relkind='t'`。
### 5.3 先合并 freeze max age
`relation_needs_vacanalyze()` 先合并 reloptions 和 GUC。
普通 XID 使用 `autovacuum_freeze_max_age`。
MultiXact 使用 `effective_multixact_freeze_max_age`。
后者来自：
```text
MultiXactMemberFreezeThreshold()
```
当 `pg_multixact/members` 使用压力超过低水位时，
这个函数会把有效 MXID freeze max age 调小。
达到高水位附近时，
它甚至可以把结果压到 0。
因此 MultiXact danger 不只是 ID age。
members storage pressure 也会让系统更早 freeze MultiXact。
### 5.4 force vacuum 判定发生在普通 stats 前
函数读取：
```text
relfrozenxid = classForm->relfrozenxid
relminmxid = classForm->relminmxid
```
然后计算：
```text
xidForceLimit = recentXid - freeze_max_age
force_vacuum =
  TransactionIdIsNormal(relfrozenxid) &&
  TransactionIdPrecedes(relfrozenxid, xidForceLimit)
```
如果 XID 没触发，
再看 MultiXact：
```text
multiForceLimit = recentMulti - multixact_freeze_max_age
force_vacuum =
  MultiXactIdIsValid(relminmxid) &&
  MultiXactIdPrecedes(relminmxid, multiForceLimit)
```
一旦触发：
```text
*wraparound = true
*dovacuum = true
```
这个判断在 `pgstat_fetch_stat_tabentry_ext()` 前面。
所以 stats 缺失或 dead tuple 很少，
都不能阻止 forced vacuum。
这是 anti-wraparound priority 的第一道边界：
catalog age 压过 ordinary pgstat threshold。
### 5.5 score 决定候选表顺序
force vacuum 只表示必须做。
如果 database 中有多张表都需要处理，
worker 还要排序。
XID/MXID score 是：
```text
xid_age / freeze_max_age
mxid_age / multixact_freeze_max_age
```
普通 score 是：
```text
dead_tuples / vacuum_threshold
insert_since_vacuum / insert_threshold
mod_since_analyze / analyze_threshold
```
`scores.max` 取最大 component。
`do_autovacuum()` 将 `TableToProcess` 列表排序。
如果 age 超过 effective failsafe age，
XID/MXID score 会被放大，
使危险表更可能排到前面。
score 是调度近似。
源码注释也说明它不可能对所有 workload 完美。
所以执行前还需要 claim 和 recheck。
### 5.6 claim 防止重复 worker
worker 处理候选表时，
先拿 `AutovacuumScheduleLock` 和 `AutovacuumLock`，
查看其它 running worker 的 `wi_tableoid`。
如果同一 relation 已被别的 worker 处理，
当前 worker 跳过。
否则写入：
```text
MyWorkerInfo->wi_tableoid = relid
MyWorkerInfo->wi_sharedrel = isshared
```
这表示当前 worker claim 了 relation。
claim 只防重复。
它不证明表仍需要 vacuum。
因此下一步要 recheck。
### 5.7 recheck 生成 `VacuumParams`
`table_recheck_autovac()` 再读 syscache、reloptions 和 stats。
它再次调用 `relation_needs_vacanalyze()`。
如果已经不需要维护，
返回 NULL，
worker 释放 claim。
如果仍需要维护，
构造 `autovac_table`，
并设置：
```text
tab->at_params.freeze_min_age
tab->at_params.freeze_table_age
tab->at_params.multixact_freeze_min_age
tab->at_params.multixact_freeze_table_age
tab->at_params.is_wraparound = wraparound
```
选项里还有关键差异：
```text
(!wraparound ? VACOPT_SKIP_LOCKED : 0)
```
普通 autovacuum 可以 skip locked。
wraparound vacuum 不使用这个普通退避。
这是 anti-wraparound priority 的第二道边界：
锁冲突不能轻易让 correctness 任务一直跳过。
### 5.8 wraparound worker 初始仍使用 cost delay
执行某张表前，
worker 保存 per-table cost reloptions：
```text
av_storage_param_cost_delay
av_storage_param_cost_limit
```
有 per-table cost 参数时，
该 worker 不参与普通 worker 间 cost balance。
没有 per-table cost 参数时，
它使用 autovacuum 或全局 cost 参数，
并按当前参与 balance 的 worker 数分摊 cost limit。
`VacuumUpdateCosts()` 选择：
```text
per-table storage parameter
  -> autovacuum_vac_cost_delay / autovacuum_vac_cost_limit
  -> VacuumCostDelay / VacuumCostLimit
```
如果 `vacuum_cost_delay > 0` 且 failsafe 未触发，
`VacuumCostActive=true`。
所以不要误解：
`to prevent wraparound` 不等于马上无视成本节流。
真正压过 cost delay 的边界是 failsafe。
### 5.9 进入 VACUUM 执行层
`autovacuum_do_vac_analyze()` 设置 activity string。
wraparound 时会带 `to prevent wraparound`。
然后进入普通 `vacuum()`。
手工 `VACUUM (FREEZE)` 也会进入这套执行层。
但手工 VACUUM 的：
```text
params.is_wraparound = false
```
因此手工 `VACUUM (FREEZE)` 可以 aggressive，
但不是 autovacuum wraparound worker。
### 5.10 `vacuum_get_cutoffs()` 计算 cutoff
`heap_vacuum_rel()` 调用：
```text
vacrel->aggressive = vacuum_get_cutoffs(rel, params, &vacrel->cutoffs)
```
`vacuum_get_cutoffs()` 读取：
```text
OldestXmin = GetOldestNonRemovableTransactionId(rel)
OldestMxact = GetOldestMultiXactId()
nextXID = ReadNextTransactionId()
nextMXID = ReadNextMultiXactId()
```
然后计算：
```text
FreezeLimit = nextXID - freeze_min_age
MultiXactCutoff = nextMXID - multixact_freeze_min_age
```
并限制：
```text
FreezeLimit <= OldestXmin
MultiXactCutoff <= OldestMxact
```
如果 `OldestXmin` 或 `OldestMxact` 已经被拖得太旧，
源码会 WARNING，
提示关闭旧事务、prepared transaction 或 stale replication slots。
### 5.11 aggressive 决定扫描义务
`vacuum_get_cutoffs()` 用 table freeze age 判断 aggressive。
普通 XID：
```text
freeze_table_age =
  Min(vacuum_freeze_table_age, autovacuum_freeze_max_age * 0.95)
aggressiveXIDCutoff = nextXID - freeze_table_age
relfrozenxid <= aggressiveXIDCutoff => aggressive
```
MultiXact 类似：
```text
multixact_freeze_table_age =
  Min(vacuum_multixact_freeze_table_age,
      effective_multixact_freeze_max_age * 0.95)
relminmxid <= aggressiveMXIDCutoff => aggressive
```
`0.95` 的作用是给计划性 VACUUM 留余地：
在真正 anti-wraparound force vacuum 之前，
普通或手工 VACUUM 也有机会推进 horizon。
aggressive 的义务：
```text
Aggressive VACUUM must advance relfrozenxid to FreezeLimit
and relminmxid to MultiXactCutoff, at minimum.
```
因此 aggressive 不是“日志里看起来更严重”。
它是 relation horizon 更新的执行合同。
### 5.12 progress 标记
`heap_vacuum_rel()` 在 progress 中记录 started_by。
当前本地 `system_views.sql` 映射为：
```text
started_by:
  manual
  autovacuum
  autovacuum_wraparound
mode:
  normal
  aggressive
  failsafe
```
这也是诊断时区分状态的入口。
`started_by='autovacuum_wraparound'` 表示选择层来自 wraparound。
`mode='aggressive'` 表示执行层扫描义务。
`mode='failsafe'` 表示已经进入 emergency mode。
### 5.13 failsafe 反复检查
`heap_vacuum_rel()` 在分配 dead_items 前先检查 failsafe。
扫描过程中也会定期检查。
index/heap vacuum 阶段前后也会检查。
底层函数是：
```text
vacuum_xid_failsafe_check(&vacrel->cutoffs)
```
普通 XID 条件：
```text
skip_index_vacuum =
  Max(vacuum_failsafe_age, autovacuum_freeze_max_age * 1.05)
xid_skip_limit = ReadNextTransactionId() - skip_index_vacuum
relfrozenxid < xid_skip_limit => failsafe
```
MultiXact 条件同理。
failsafe 使用本次 VACUUM 开始时的
`cutoffs.relfrozenxid` 和 `cutoffs.relminmxid`。
它判断这张表进入本次 VACUUM 时的 catalog horizon 是否已经危险。
### 5.14 failsafe 压过普通成本策略
`lazy_check_wraparound_failsafe()` 触发后：
```text
VacuumFailsafeActive = true
vacrel->bstrategy = NULL
vacrel->do_index_vacuuming = false
vacrel->do_index_cleanup = false
vacrel->do_rel_truncate = false
VacuumCostActive = false
VacuumCostBalance = 0
```
这就是本节主问题的执行层答案。
ordinary cost strategy 被压过的边界是：
wraparound danger 达到 failsafe threshold。
系统此时放弃：
cost-based sleep、
VACUUM buffer access strategy 的 ring 限制、
后续 index vacuuming、
index cleanup、
heap truncation。
系统不放弃：
heap scan 中必须完成的 prune/freeze 语义、
WAL/crash safety、
MVCC cutoff 约束、
relation/page lock 约束。
### 5.15 回写 relation 和 database horizon
扫描完成后，
`heap_vacuum_rel()` 根据扫描结果和 VM 统计调用：
```text
vac_update_relstats(...,
                    NewRelfrozenXid,
                    NewRelminMxid,
                    ...)
```
如果 horizon 成功推进，
`pg_class.relfrozenxid` 和 `relminmxid` 前移。
随后 database-level 聚合通过 `vac_update_datfrozenxid()` 推进。
`SetTransactionIdLimit()` 再根据最老 `datfrozenxid`
设置 `xidVacLimit`、`xidWarnLimit`、`xidStopLimit`、`xidWrapLimit`。
这条链路把一页一页的 freeze，
最终连接回整个实例的 XID 分配安全边界。
## 6. 生命周期 / ownership / cleanup
`relfrozenxid` 和 `relminmxid` 是 catalog 持久状态。
它们由 relation 创建、rewrite 和 VACUUM 等路径设置或推进。
本节只关注 VACUUM 推进。
`datfrozenxid` 和 `datminmxid` 是 database-level 聚合状态。
launcher 用它们选 database。
worker 处理完 relation 后，VACUUM 路径再更新 database horizon。
`TableToProcess` 和 `autovac_table` 属于一个 autovacuum worker。
它们分配在 `AutovacMemCxt`。
其它 worker 不能直接访问。
跨 worker 的当前 relation ownership 在 shared memory 中：
```text
WorkerInfoData.wi_dboid
WorkerInfoData.wi_tableoid
WorkerInfoData.wi_sharedrel
```
`wi_tableoid` 和 `wi_sharedrel` 由 `AutovacuumScheduleLock` 保护。
这只保护 claim。
不代表 relation 一定仍需要 vacuum。
`LVRelState` 属于单次 `heap_vacuum_rel()`。
它在一张表的 VACUUM 生命周期内记录 cutoff、aggressive、progress、VM skipping、
dead_items、index cleanup 和 failsafe 状态。
ERROR 时，
当前 table 的事务 abort。
locks、pins、内存和资源由事务/ResourceOwner/MemoryContext 机制清理。
autovacuum worker 的 `PG_TRY()` 路径还要释放 table claim，
避免其它 worker 永久认为该表正在处理。
每张表处理结束后，
worker reset `PortalContext`，
清空 `wi_tableoid`，
再处理下一张候选表。
worker 退出时，
`WorkerInfo` 回到 free list，
并触发 cost balance 重新计算。
## 7. 正确性机制层次
第一层是 XID/MultiXact 环形空间。
TransactionId 是有限空间。
PostgreSQL 依赖 `TransactionIdPrecedes()` 这类比较在半环内成立。
因此最老未冻结 XID 不能无限停留。
`SetTransactionIdLimit()` 用 oldest `datfrozenxid` 计算：
```text
xidVacLimit
xidWarnLimit
xidStopLimit
xidWrapLimit
```
超过 `xidVacLimit` 后会更积极请求 autovacuum。
接近 `xidStopLimit` 后会阻止继续分配普通 XID。
第二层是 MVCC cleanup horizon。
freeze 仍受 `OldestXmin` 和 `OldestMxact` 限制。
VACUUM 不能因为 wraparound 风险就把仍可能被旧 snapshot 解释的事务身份提前抹掉。
第三层是 relation/page 并发。
anti-wraparound vacuum 不绕过 relation lock。
也不绕过 buffer pin、page lock、WAL 和 critical section。
它只是减少普通退避，并在 failsafe 后放弃可延后维护。
第四层是 WAL/crash safety。
tuple freeze 和 visibility map 修改仍需要正确 WAL 语义。
failsafe 跳过 index cleanup 和 truncation，
不是跳过必须持久化的 heap freeze。
第五层是 stats 与 catalog 的边界。
pgstat 是 ordinary scheduling input。
catalog age 是 wraparound risk input。
真正的 correctness proof 来自 heap scan/freeze 后更新 `pg_class`。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 per-table disable 不能阻断 forced vacuum
路径：
```text
autovacuum_enabled=false
  -> ordinary threshold 不设置 dovacuum
  -> force_vacuum=true
  -> wraparound=true
  -> dovacuum=true
```
这是有意设计。
否则一个表级配置错误就能把整个 database 推向 wraparound。
### 8.2 stats missing 不阻断 forced vacuum
`relation_needs_vacanalyze()` 在没有 pgstat entry 时会 return。
但 force vacuum 判定已经发生。
所以：
```text
not forced + no stats:
  skipped
forced by age + no stats:
  already dovacuum
```
这体现了 pgstat 与 catalog age 的优先级差异。
### 8.3 config reload 不应取消 emergency worker
worker 每张表前处理 `ConfigReloadPending`。
源码注释明确提醒：
看到 autovacuum 被关闭也不要直接退出。
当前 worker 可能是 for-wraparound emergency worker。
这说明 `autovacuum=off` 不是安全地关闭 wraparound protection。
### 8.4 candidate stale 由 claim + recheck 兜底
候选列表来自一次 `pg_class` scan。
在执行前，
其它 worker 可能已经处理同一表。
如果 claim 时发现别的 worker 正在处理，
当前 worker 跳过。
如果 claim 成功但 recheck 发现不再需要，
`table_recheck_autovac()` 返回 NULL，
当前 worker 释放 claim。
这避免重复 vacuum。
### 8.5 lock conflict 不再普通 skip locked
wraparound vacuum 不设置 `VACOPT_SKIP_LOCKED`。
因此它可能等待阻塞锁。
这不是 bug。
如果 anti-wraparound vacuum 总是因为锁冲突跳过，
age 会继续增长。
诊断应查 `pg_stat_activity` 和 `pg_locks`。
不要只把等待解释成 autovacuum 自身慢。
### 8.6 failsafe 留下 bloat
failsafe 后跳过 index vacuuming、index cleanup 和 heap truncation。
因此 dead index entries 或 heap tail 空间可能留到后续普通 VACUUM。
这不是 VACUUM 失败。
这是 emergency trade-off：
先推进 freeze horizon，
再处理空间回收质量。
### 8.7 long horizon 让 VACUUM 推不动
长期事务、prepared transaction、replication slot 或 hot standby feedback
可能让 `OldestXmin` / `OldestMxact` 留在过去。
此时 forced vacuum 可以启动，
但 cutoff 被 clamp，
`relfrozenxid` 或 `relminmxid` 可能推进有限。
增加 worker 数不能解决这种问题。
必须找 horizon holder。
### 8.8 MultiXact member pressure 提前触发
`MultiXactMemberFreezeThreshold()` 会在 members 空间压力过大时
收缩 effective multixact freeze max age。
高并发行锁、外键检查、热点行上的 key-share/share locks，
都可能增加 MultiXact members 压力。
只看 XID age 会漏掉这类风险。
## 9. 成本、资源与跨模块传播
普通 cost delay 控制：
```text
VacuumCostActive
VacuumCostBalance
vacuum_cost_delay
vacuum_cost_limit
```
autovacuum worker 还会按参与 balance 的 worker 数分摊 cost limit。
这保护前台 workload，
降低连续 I/O、buffer churn 和后台吞吐尖峰。
wraparound vacuum 初始仍可使用这些机制。
这让系统在风险可控时仍尽量平滑。
failsafe 后成本传播改变：
```text
VacuumCostActive=false
bstrategy=NULL
skip further index vacuuming
skip index cleanup
skip heap truncation
```
代价包括：
更集中 I/O、
shared buffer cache 被冲刷、
WAL 和 checkpoint 压力上升、
index bloat 留存、
heap tail 空间不回收。
这些代价被接受，
因为 wraparound failure 是 correctness 风险。
主要规模变量：
```text
number of relations:
  worker 扫 pg_class 和计算 score 的成本。
heap pages:
  aggressive/freeze 扫描成本。
unfrozen tuples:
  tuple header 修改和 WAL 成本。
indexes:
  普通 vacuum 的 index cleanup 成本；
  failsafe 后可能跳过。
active backends / slots / prepared xacts:
  OldestXmin / OldestMxact 推进成本。
MultiXact members:
  effective multixact freeze max age 收缩和 SLRU storage pressure。
```
跨模块传播：
```text
autovacuum.c:
  把 catalog age 和 pgstat threshold 转成 selection、score、is_wraparound。
vacuum.c:
  把 VacuumParams 转成 VacuumCutoffs、aggressive 和 failsafe check。
vacuumlazy.c:
  把 aggressive/failsafe 转成 VM skipping、cost delay、index cleanup 和 truncation 策略。
heapam/pruneheap:
  实际修改 tuple header 和 page/VM 状态。
varsup.c:
  把 database frozen horizon 转成 XID 分配 warning/stop/wrap limit。
multixact.c:
  把 members storage pressure 转成更早的 MXID freeze pressure。
pgstat/progress/log:
  暴露当前执行与历史维护结果。
```
## 10. 观测与诊断入口
先看 database-level risk：
```sql
SELECT datname,
       age(datfrozenxid) AS xid_age,
       mxid_age(datminmxid) AS mxid_age,
       datfrozenxid,
       datminmxid
FROM pg_database
ORDER BY GREATEST(age(datfrozenxid), mxid_age(datminmxid)) DESC;
```
再下钻 relation-level risk：
```sql
SELECT n.nspname,
       c.relname,
       c.relkind,
       age(c.relfrozenxid) AS xid_age,
       mxid_age(c.relminmxid) AS mxid_age,
       c.relpages,
       c.relallfrozen
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY GREATEST(age(c.relfrozenxid), mxid_age(c.relminmxid)) DESC
LIMIT 50;
```
看当前 worker：
```sql
SELECT pid,
       backend_type,
       state,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE backend_type LIKE 'autovacuum%';
```
看 progress：
```sql
SELECT pid,
       relid::regclass,
       phase,
       heap_blks_total,
       heap_blks_scanned,
       index_vacuum_count,
       mode,
       started_by
FROM pg_stat_progress_vacuum;
```
当前本地源码的 view 会把 mode 映射成：
`normal`、`aggressive`、`failsafe`。
started_by 映射成：
`manual`、`autovacuum`、`autovacuum_wraparound`。
看日志：
```text
log_autovacuum_min_duration = 0
```
重点匹配：
```text
automatic aggressive vacuum to prevent wraparound
automatic vacuum to prevent wraparound
bypassing nonessential maintenance ... as a failsafe
index scan bypassed by failsafe
cutoff for removing and freezing tuples is far in the past
cutoff for freezing multixacts is far in the past
database ... must be vacuumed within ... transactions
```
看 horizon holder：
```sql
SELECT pid,
       backend_xmin,
       age(backend_xmin) AS xmin_age,
       xact_start,
       state,
       query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```
```sql
SELECT gid, database, owner, prepared
FROM pg_prepared_xacts
ORDER BY prepared;
```
```sql
SELECT slot_name,
       slot_type,
       active,
       xmin,
       catalog_xmin,
       restart_lsn
FROM pg_replication_slots;
```
这些入口能说明：
为什么 VACUUM 启动了但 horizon 仍推不动。
它们不能说明：
具体哪个 page 上还有未冻结 tuple。
那需要 VACUUM 日志、page inspection、断点或源码插桩。
建议源码断点：
`relation_needs_vacanalyze`、`table_recheck_autovac`、`VacuumUpdateCosts`、
`vacuum_get_cutoffs`、`vacuum_xid_failsafe_check`、
`lazy_check_wraparound_failsafe`、`heap_vacuum_rel`。
观察变量：
`force_vacuum`、`wraparound`、`scores`、`params->is_wraparound`、
`vacrel->aggressive`、`VacuumFailsafeActive`、`VacuumCostActive`、
`vacrel->do_index_vacuuming`、`vacrel->do_index_cleanup`、`vacrel->bstrategy`。
## 11. 常见误区
误区一：
`autovacuum_enabled=false` 可以完全阻止 autovacuum。
不对。
它不能阻止 anti-wraparound vacuum。
误区二：
dead tuple 少就不需要 vacuum。
不对。
freeze 关注 live tuple 中的旧 XID/MXID 依赖。
误区三：
`to prevent wraparound` 一定表示 cost delay 已关闭。
不对。
只有 failsafe 触发后才关闭 `VacuumCostActive`。
误区四：
aggressive 等于 failsafe。
不对。
aggressive 是扫描义务。
failsafe 是 emergency execution mode。
误区五：
只看 `age(datfrozenxid)` 就够。
不对。
必须同时看 `mxid_age(datminmxid)`，
以及 relation-level `relfrozenxid` / `relminmxid`。
误区六：
failsafe 跳过 index cleanup 表示 VACUUM 失败。
不对。
它表示系统优先推进 freeze horizon，
空间回收质量留给后续维护。
误区七：
增加 autovacuum worker 数总能解决 wraparound。
不对。
如果瓶颈是 long transaction、slot、prepared xact 或锁，
更多 worker 可能只会增加竞争。
## 12. 课堂实验
### 实验一：源码跟读 force vacuum
目标：
确认 forced vacuum 发生在 ordinary pgstat threshold 之前。
断点：
```text
relation_needs_vacanalyze
```
观察：
```text
classForm->relfrozenxid
classForm->relminmxid
recentXid
recentMulti
freeze_max_age
multixact_freeze_max_age
force_vacuum
*wraparound
*dovacuum
```
问题：
为什么 stats missing 不能阻止 forced table？
### 实验二：区分 wraparound、aggressive、failsafe
目标：
建立三个状态的映射。
断点：
```text
table_recheck_autovac
heap_vacuum_rel
lazy_check_wraparound_failsafe
```
观察：
```text
tab->at_params.is_wraparound
vacrel->aggressive
VacuumFailsafeActive
```
同时查询：
```sql
SELECT pid, relid::regclass, mode, started_by
FROM pg_stat_progress_vacuum;
```
问题：
为什么 `started_by='autovacuum_wraparound'`
不必然等于 `mode='failsafe'`？
### 实验三：long transaction 拖住 cutoff
会话 A：
```sql
BEGIN;
SELECT txid_current();
SELECT * FROM some_large_table LIMIT 1;
```
会话 B：
```sql
VACUUM (VERBOSE, FREEZE) some_large_table;
```
会话 C：
```sql
SELECT pid, backend_xmin, age(backend_xmin), xact_start, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```
断点：
```text
vacuum_get_cutoffs
```
观察：
```text
OldestXmin
FreezeLimit
safeOldestXmin
```
问题：
为什么 anti-wraparound pressure 不能越过 `OldestXmin`？
## 13. 讨论题
1. 为什么 per-table `autovacuum_enabled=false` 不能阻止 anti-wraparound vacuum？
2. 为什么 force vacuum 判定要早于 ordinary pgstat threshold？
3. `is_wraparound`、`aggressive`、`failsafe` 分别回答什么问题？
4. 为什么 failsafe 可以跳过 index cleanup，却不能跳过 heap freeze correctness？
5. 为什么 `FreezeLimit` 必须不超过 `OldestXmin`？
6. 为什么 per-table freeze max age 只能让表更早，而不能比全局边界更晚？
7. 当日志出现 `cutoff for freezing multixacts is far in the past`，你会先查哪些视图？
8. 为什么增加 worker 数可能无法解决 wraparound danger？
## 14. 本节小结
本节主链路：
```text
pg_database frozen horizon
  -> launcher 选 database
  -> worker 扫 pg_class
  -> relation_needs_vacanalyze() 判定 force_vacuum / wraparound / score
  -> table_recheck_autovac() 生成 VacuumParams
  -> vacuum_get_cutoffs() 计算 cutoffs 和 aggressive
  -> heap_vacuum_rel() 执行 freeze / horizon 推进
  -> lazy_check_wraparound_failsafe() 在危险过深时停止成本节流
```
核心状态：
`relfrozenxid` 和 `relminmxid` 是 relation-level frozen horizon。
`datfrozenxid` 和 `datminmxid` 是 database-level 聚合 horizon。
`AutoVacuumScores` 把普通维护压力和 wraparound pressure 放进同一排序模型。
`VacuumParams.is_wraparound` 标记 autovacuum selection layer 的强制原因。
`VacuumCutoffs` 把 MVCC horizon 和 freeze cutoff 绑定。
`LVRelState` 记录执行期 aggressive、VM skipping、new horizon 和 failsafe。
正确性边界：
anti-wraparound priority 不绕过 MVCC、lock、pin 或 WAL。
它压过的是普通维护的退避和成本节流。
在 failsafe 前，wraparound vacuum 仍可使用 cost delay。
在 failsafe 后，系统停止 cost delay，放弃非必要维护，
优先推进 freeze horizon。
诊断边界：
`pg_database` 看到 database risk。
`pg_class` 看到 relation risk。
`pg_stat_activity` 看到当前 worker。
`pg_stat_progress_vacuum` 看到 `started_by` 和 `mode`。
日志看到 anti-wraparound、aggressive、failsafe 和扫描结果。
`backend_xmin`、prepared xacts、replication slots 解释 horizon 为什么推不动。
可迁移规律：
当一个后台维护系统同时承担性能维护和 correctness 推进时，
普通成本模型必须是可退让层；
correctness horizon 必须有强制路径和 emergency fallback。
