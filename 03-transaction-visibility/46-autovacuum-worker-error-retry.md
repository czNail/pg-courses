# PostgreSQL autovacuum worker error / retry cleanup
## 课程定位
前置知识：已经理解 heap VACUUM 的 cleanup horizon、freeze / anti-wraparound、visibility map、autovacuum launcher / worker 调度、vacuum cost delay，以及 PostgreSQL 的 MemoryContext、ResourceOwner、heavyweight lock 和 ERROR longjmp。本节唯一主问题：
```text
worker ERROR、锁冲突、表被删除和重试节流分别如何收尾？
```
核心矛盾：
```text
autovacuum 必须长期自动推进维护，不能因为一张表 ERROR、锁冲突或对象消失就停住；
但 VACUUM 会打开 relation、持锁、设置 ProcArray flags、启动事务、更新 stats、
甚至启动 parallel vacuum worker，失败时必须把每一层 ownership 收回到清楚边界。
```
本节只讲收尾和重试边界。不重复第 28 节的 heap / index cleanup 算法。不重复第 39 节的 launcher / worker 调度评分。不重复第 41 节的 wraparound 风险计算。学完后应能判断：
```text
worker 顶层 ERROR 为什么退出进程，而不是继续同一个 database；
每张表 ERROR 为什么可以 catch 后继续下一张候选表；
普通 autovacuum 遇到 relation lock 冲突为什么通常 skip；
anti-wraparound vacuum 为什么不能沿用普通 skip locked 策略；
表被删除为什么在多处 recheck 中变成 NULL / continue；
wi_tableoid、wi_sharedrel 和 WorkerInfo 何时清理；
launcher 如何处理 starting worker 卡住、fork 失败和 database 级重试节流；
statement_timeout / lock_timeout 为什么在 worker 中被强制关闭；
哪些现象能从 SQL / log 直接看到，哪些只能推断或断点观察。
```
本课基于本地 `/home/nail/postgres` 源码。源码分支为 `master`。短提交为 `0e1f1ed157e9`。
## 1. 本节在总主线中的位置
前面的 VACUUM 课程解释的是：
```text
tuple 何时可回收；
page 何时可剪枝；
heap scan、index bulk delete、heap cleanup 如何排阶段；
freeze 如何避免 XID / MultiXact wraparound；
visibility map 如何缓存 page-level 可见性；
autovacuum 如何选择 database 和 relation；
maintenance diagnostics 如何把 age、dead tuple、bloat 和 logs 拼起来。
```
本节接在这些主题之后。它关心的不是一张表“为什么需要 vacuum”。它关心的是：
```text
自动维护已经决定要做了，
但执行过程中遇到 ERROR、锁冲突、DROP TABLE、worker 启动失败或 stale stats 时，
系统如何安全地停下当前层，然后让后续维护仍能继续推进？
```
本节的 runtime 锚点是一组常见现场：
```text
某个 autovacuum worker 日志里出现 automatic vacuum of table ... ERROR；
另一张表仍然在同一轮或下一轮被处理；
热点表长期 n_dead_tup 增长，但抓不到 autovacuum lock wait；
anti-wraparound worker 却真的卡在 Lock wait；
候选表被 DROP 后没有明显 ERROR；
launcher 没有立刻对同一个 database tight loop 重试。
```
这些现象不能用一个“autovacuum failed”解释。它们分别落在不同边界：
```text
worker process lifecycle；
WorkerInfo shared slot；
current table claim；
relation heavyweight lock；
current transaction；
per-table memory context；
ERROR state；
launcher database schedule。
```
本节的可迁移目标是：
```text
学会把长期后台维护进程的失败处理拆成 ownership 层次，
而不是期待一个统一 catch-all 能清理所有状态。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
AutoVacLauncherMain() 选择 database 并把一个 WorkerInfo 放到 av_startingWorker；
AutoVacWorkerMain() 接管 WorkerInfo，连接 database，运行 do_autovacuum()；
do_autovacuum() 扫 pg_class 得到候选 relation list；
每张表先用 AutovacuumScheduleLock 写 wi_tableoid claim；
再用 table_recheck_autovac() 重新读取 pg_class / pgstat / reloptions；
普通 vacuum 通过 VACOPT_SKIP_LOCKED 让 vacuum_open_relation() 非阻塞拿锁；
单表执行包在 PG_TRY/PG_CATCH 中，ERROR 时 abort 当前事务、flush error state、reset PortalContext；
本表结束后清 wi_tableoid；
worker 退出时 FreeWorkerInfo() 归还 slot；
launcher 用 naptime、startingWorker timeout、fork-failed retry 和 database stats 控制下一轮。
```
这个模型里有四个 tension。第一，自动维护要继续前进。但一个 relation 的 ERROR 不能污染下一张 relation。所以单表边界必须足够强：
```text
EmitErrorReport()
AbortOutOfAnyTransaction()
FlushErrorState()
MemoryContextReset(PortalContext)
StartTransactionCommand()
```
第二，普通维护要低干扰。但 wraparound 维护不能永久让位。所以普通 autovacuum 使用 `VACOPT_SKIP_LOCKED`。而 `wraparound == true` 时不设置这个 flag。第三，候选表 list 是过时的。但执行必须基于当前事实。所以源码在 claim 前后、执行前后多次 recheck：
```text
SearchSysCache1(RELOID)
table_recheck_autovac()
get_rel_name() / get_namespace_name()
vacuum_open_relation()
```
第四，失败后要再试。但不能 per-table tight loop。所以 PostgreSQL 没有 durable per-relation retry queue。它把 retry 交给：
```text
launcher database schedule；
autovacuum_naptime；
pgstat freshness；
下一轮 pg_class scan；
执行前 table recheck。
```
本节最重要的判断：
```text
autovacuum retry cleanup 是分层收尾和下一轮重新决策，
不是“失败 job 立即重放”。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/autovacuum.c` | `AutoVacLauncherMain()`、`do_start_worker()`、`launch_worker()`、`AutoVacWorkerMain()`、`do_autovacuum()`、`table_recheck_autovac()`、`FreeWorkerInfo()`、per-table `PG_TRY/PG_CATCH`。 |
| 2 | `src/include/postmaster/autovacuum.h` | autovacuum 对外入口和 worker / launcher 身份。 |
| 3 | `src/backend/commands/vacuum.c` | `vacuum()`、`vacuum_rel()`、`vacuum_open_relation()`，特别是 transaction boundary、`VACOPT_SKIP_LOCKED`、relation lock 和 relation gone。 |
| 4 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM error callback、progress、parallel vacuum cleanup 收束点。 |
| 5 | `src/backend/commands/vacuumparallel.c` | parallel vacuum DSM / worker / leader cleanup，本节只关注 ERROR 时仍由 leader 和事务边界收束。 |
| 6 | `src/backend/storage/lmgr/lmgr.c` | `ConditionalLockRelationOid()`，解释 lock conflict 如何变成 skip。 |
| 7 | `src/backend/access/transam/xact.c` | `AbortOutOfAnyTransaction()`、`CommitTransactionCommand()`、ResourceOwner 和 AtEOXact cleanup 顺序。 |
| 8 | `src/backend/utils/error/elog.c` | `errfinish()`、`EmitErrorReport()`、`FlushErrorState()`，解释 ERROR longjmp 和 ErrorContext。 |
| 9 | `src/backend/utils/activity/pgstat_relation.c` / `pgstat_database.c` | relation / database stats 何时更新，以及失败尝试和成功维护的粒度差异。 |
| 10 | `src/include/storage/lwlocklist.h`、`wait_event_names.txt` | `AutovacuumLock`、`AutovacuumScheduleLock` 和可观测 wait event。 |
推荐阅读顺序：
```text
WorkerInfoData / AutoVacuumShmemStruct
  -> launcher startingWorker 和 naptime
  -> worker 顶层 sigsetjmp
  -> do_autovacuum 候选表扫描
  -> AutovacuumScheduleLock claim
  -> table_recheck_autovac
  -> vacuum_open_relation skip locked
  -> per-table PG_TRY/PG_CATCH
  -> FreeWorkerInfo
```
不要从 `heap_vacuum_rel()` 的 tuple 扫描开始读本节。tuple 清理算法解释不了 `wi_tableoid` 何时清空。也不要从 `pg_stat_all_tables` 单字段倒推。`last_autovacuum` 只能说明 relation 层成功报告过维护。它不能说明上一轮是否尝试、skip、ERROR 或因 relation gone 放弃。
## 4. 关键数据结构与状态
### 4.1 `AutoVacuumShmemStruct`
`AutoVacuumShmemStruct` 是 autovacuum 的 shared memory 根状态。本节关心这些字段：
| 字段 | 语义 |
| --- | --- |
| `av_launcherpid` | 当前 launcher PID，worker 接管后会唤醒 launcher。 |
| `av_freeWorkers` | 可分配 worker slot 链表。 |
| `av_runningWorkers` | 已接管并运行中的 worker 链表。 |
| `av_startingWorker` | launcher 已分配、postmaster 正在启动、worker 尚未接管的单个 slot。 |
| `av_signal[]` | `AutoVacRebalance`、`AutoVacForkFailed` 等 launcher 信号。 |
| `av_nworkersForBalance` | 参与 autovacuum cost balance 的 worker 数。 |
| `av_workItems` | BRIN summarize 等附加 work item。 |
大部分字段由 `AutovacuumLock` 保护。`av_startingWorker` 是启动 handoff 状态。同一时刻只允许一个 worker 处于这个状态。如果 worker 迟迟没有清掉它，launcher 会按 timeout 回收。如果 postmaster fork 失败，`AutoVacWorkerFailed()` 设置 `AutoVacForkFailed`。launcher 收到后等待 1 秒，再重新发送 `PMSIGNAL_START_AUTOVAC_WORKER`。源码注释还留下一个现实限制：
```text
XXX should we put a limit to the number of times we retry?
```
这说明 fork failure retry 是启动层重试，不是 relation 级 retry。
### 4.2 `WorkerInfoData`
`WorkerInfoData` 是每个 autovacuum worker slot 的 shared whereabouts。
| 字段 | 语义 |
| --- | --- |
| `wi_dboid` | worker 目标 database。 |
| `wi_tableoid` | 当前 claim 的 relation；空闲或未处理表时为 `InvalidOid`。 |
| `wi_sharedrel` | 当前 relation 是否 shared catalog。 |
| `wi_proc` | worker 接管后写入自己的 `PGPROC *`。 |
| `wi_launchtime` | launcher 设置，用来判断 starting worker 是否卡住。 |
| `wi_dobalance` | 是否参与 autovacuum cost limit 平衡。 |
不要把 `wi_tableoid` 理解成“VACUUM 正在成功清理这张表”。它只表示：
```text
当前 worker 已经在 AutovacuumScheduleLock 下 claim 了这个 relation，
其它 autovacuum worker 应该避开同一 relation。
```
claim 后仍可能发生：
```text
table_recheck_autovac() 返回 NULL；
relation name 查不到；
vacuum_open_relation() 因 skip locked 返回 NULL；
VACUUM / ANALYZE 中途 ERROR；
worker 收到 SIGTERM；
worker process exit 触发 FreeWorkerInfo。
```
raw field 不是语义。`wi_tableoid + wi_sharedrel + wi_dboid + AutovacuumScheduleLock + worker lifecycle` 才是 table claim 语义。
### 4.3 `AutovacMemCxt` 与 `PortalContext`
`do_autovacuum()` 创建两个关键 memory context。`AutovacMemCxt` 保存 database 本轮需要跨多张表使用的状态：
```text
tables_to_process；
table_toast_map；
pg_class tupledesc copy；
candidate list sorting state。
```
`PortalContext` 是每张表执行使用的短生命周期上下文。每张表开始前和 ERROR 后都会 reset。这只解决内存问题。它不释放 heavyweight lock。它不回滚事务。它不清 buffer pin。它不清 ErrorContext。所以 catch 中还必须调用 `AbortOutOfAnyTransaction()` 和 `FlushErrorState()`。
### 4.4 `autovac_table`
`table_recheck_autovac()` 返回 `autovac_table`。这是本轮本地执行描述，不是 durable job。关键字段组：
```text
at_relid:
  relation OID。

at_params:
  传给 vacuum()/analyze_rel() 的 VacuumParams。

at_storage_param_vac_cost_delay / at_storage_param_vac_cost_limit:
  reloptions 覆盖后的 cost 参数。

at_dobalance:
  是否参与全局 cost balance。

at_relname / at_nspname / at_datname:
  ERROR context 用的名字副本。
```
本节最重要的是 `at_params.options`。普通 autovacuum 会设置：
```text
VACOPT_VACUUM
VACOPT_PROCESS_MAIN
VACOPT_SKIP_DATABASE_STATS
VACOPT_ANALYZE
VACOPT_SKIP_LOCKED   // only when !wraparound
```
`VACOPT_SKIP_LOCKED` 决定 relation lock conflict 是等待还是 skip。
### 4.5 `DatabaseList` 和 `avl_dbase`
`DatabaseList` 是 launcher-local schedule cache。它不在 shared memory。它也不是持久化 job queue。`avl_dbase` 的关键字段：
```text
adl_datid:
  database OID。

adl_next_worker:
  下一次允许为该 database 启动 worker 的时间。

adl_score:
  rebuild list 时的排序辅助值。
```
worker ERROR 后，launcher 通常不知道哪张表失败了。launcher 只知道 worker slot 被释放、database schedule 继续推进。下一轮是否再处理某张表，由 pgstat、pg_class、reloptions、wraparound age 和 recheck 决定。
## 5. 主流程源码 walkthrough
### 5.1 launcher 分配 worker
launcher 主循环判断 worker slot 是否可用。如果可以启动，调用：
```text
launch_worker(now)
  -> do_start_worker()
```
`do_start_worker()` 会选择 database。选中后：
```text
pop av_freeWorkers
worker->wi_dboid = selected database
worker->wi_proc = NULL
worker->wi_launchtime = GetCurrentTimestamp()
AutoVacuumShmem->av_startingWorker = worker
SendPostmasterSignal(PMSIGNAL_START_AUTOVAC_WORKER)
```
这时 slot 已经离开 free list。但 worker 还没有接管。这就是 `av_startingWorker` 的意义。如果 worker 长时间不接管，launcher 检查：
```text
Min(autovacuum_naptime, 60) seconds
```
超过后：
```text
reset wi_dboid / wi_tableoid / wi_sharedrel / wi_proc / wi_launchtime
push slot back to av_freeWorkers
clear av_startingWorker
warning: autovacuum worker took too long to start; canceled
```
如果 postmaster fork 失败：
```text
AutoVacWorkerFailed()
  -> av_signal[AutoVacForkFailed] = true

launcher sees AutoVacForkFailed
  -> clear flag
  -> pg_usleep(1s)
  -> SendPostmasterSignal(PMSIGNAL_START_AUTOVAC_WORKER)
```
这两条都是 worker 启动层 cleanup。都还没有进入某张表的语义。
### 5.2 worker 顶层 ERROR handler
`AutoVacWorkerMain()` 的顶层不是普通 per-table `PG_TRY`。它使用 `sigsetjmp(local_sigjmp_buf, 1)`。正常路径：
```text
InitProcess()
BaseInit()
PG_exception_stack = &local_sigjmp_buf
SetConfigOption(search_path, "")
SetConfigOption(zero_damaged_pages, false)
SetConfigOption(statement_timeout, 0)
SetConfigOption(transaction_timeout, 0)
SetConfigOption(lock_timeout, 0)
SetConfigOption(idle_in_transaction_session_timeout, 0)
SetConfigOption(default_transaction_isolation, read committed)
SetConfigOption(stats_fetch_consistency, none)
take av_startingWorker under AutovacuumLock
move to av_runningWorkers
clear av_startingWorker
on_shmem_exit(FreeWorkerInfo, 0)
InitPostgres(dbid)
do_autovacuum()
proc_exit(0)
```
顶层 ERROR 跳回后：
```text
error_context_stack = NULL
HOLD_INTERRUPTS()
EmitErrorReport()
proc_exit(0)
```
worker 顶层 ERROR 不尝试继续同一个 database。原因是错误可能发生在 backend 初始化、database connection 或事务框架之外。这时最可靠的 cleanup 是退出进程，让 `proc_exit` callback 和 postmaster supervision 收尾。如果 worker 已经注册 `FreeWorkerInfo()`，slot 会归还。如果还没接管 slot，launcher 会按 `av_startingWorker` timeout 回收。
### 5.3 worker 接管 `WorkerInfo`
worker 接管时持有 `AutovacuumLock`。状态变化：
```text
MyWorkerInfo = AutoVacuumShmem->av_startingWorker
dbid = MyWorkerInfo->wi_dboid
MyWorkerInfo->wi_proc = MyProc
dlist_push_head(&av_runningWorkers, &wi_links)
AutoVacuumShmem->av_startingWorker = NULL
on_shmem_exit(FreeWorkerInfo, 0)
```
之后 worker 会唤醒 launcher。launcher 于是可以启动其它 worker。如果 worker 发现 `av_startingWorker == NULL`：
```text
elog(WARNING, "autovacuum worker started without a worker entry")
```
然后退出。这通常意味着 handoff slot 已被 launcher 回收或异常清掉。
### 5.4 database connection failure
worker 在 `InitPostgres()` 前调用：
```text
pgstat_report_autovac(dbid)
```
源码注释说明：
```text
即使连接失败，也更新 last_autovac_time，
避免 autovacuum stuck 在一个打不开的 database 上。
```
这点容易误读。database-level `last_autovac_time` 表示 autovacuum 尝试处理过这个 database。它不等于某张 relation 成功 vacuum。如果 `InitPostgres()` 因 database 被删除或无法打开而 ERROR：
```text
top-level worker handler
  -> EmitErrorReport()
  -> proc_exit(0)
  -> FreeWorkerInfo if registered
```
launcher 后续按 database schedule 再选择目标。
### 5.5 扫 `pg_class` 得到候选表
`do_autovacuum()` 进入 database 后：
```text
AutovacMemCxt = AllocSetContextCreate(...)
StartTransactionCommand()
SearchSysCache1(DATABASEOID, MyDatabaseId)
table_open(RelationRelationId, AccessShareLock)
copy pg_class TupleDesc
build TOAST-to-main relid hash
first pass scan ordinary relations and matviews
second pass scan TOAST tables
list_sort(tables_to_process, score)
```
候选表来自一次 catalog scan 和当时的 pgstat / reloptions。这个 list 不是事实。执行之前可能发生：
```text
另一个 worker 已经 vacuum 完；
手工 VACUUM 完；
用户 DROP TABLE；
reloptions 改变；
pgstat 刷新；
shared catalog 被其它 database 的 worker claim。
```
所以候选表必须 recheck。
### 5.6 table claim
候选表循环开始时，worker 先取 `pg_class` tuple。如果不存在：
```text
continue
```
如果存在，取得 `relisshared`。随后进入 claim 窗口：
```text
LWLockAcquire(AutovacuumScheduleLock, LW_EXCLUSIVE)
LWLockAcquire(AutovacuumLock, LW_SHARED)
scan av_runningWorkers
  if another worker already has same wi_tableoid in same scope:
    skip
LWLockRelease(AutovacuumLock)
if not skip:
  MyWorkerInfo->wi_tableoid = relid
  MyWorkerInfo->wi_sharedrel = isshared
LWLockRelease(AutovacuumScheduleLock)
```
两个锁的职责不同：
```text
AutovacuumLock:
  保护 worker list 和 WorkerInfo lifecycle。

AutovacuumScheduleLock:
  保护“检查其它 worker claim”和“写入自己 claim”之间的原子窗口。
```
claim 只避免 autovacuum worker 之间重复处理。它不阻止手工 VACUUM。它不阻止 DDL。它不阻止 DML。真正对象级互斥仍靠 heavyweight relation lock。
### 5.7 table recheck
claim 后调用：
```text
table_recheck_autovac(relid, table_toast_map, pg_class_desc,
                      effective_multixact_freeze_max_age)
```
它做：
```text
SearchSysCacheCopy1(RELOID, relid)
  -> invalid: return NULL
extract_autovac_opts()
  -> TOAST table 可继承 main table reloptions
relation_needs_vacanalyze()
  -> 重新判断 dovacuum / doanalyze / wraparound / score
if doanalyze || dovacuum:
  build autovac_table
else:
  return NULL
```
如果返回 NULL：
```text
LWLockAcquire(AutovacuumScheduleLock, LW_EXCLUSIVE)
wi_tableoid = InvalidOid
wi_sharedrel = false
LWLockRelease(AutovacuumScheduleLock)
continue
```
这不是 ERROR。它覆盖：
```text
表已经被别人处理；
表消失；
stats 或 reloptions 已改变；
当前已经不需要 vacuum/analyze。
```
### 5.8 relation name 消失
执行前 worker 复制名字用于 ERROR context：
```text
tab->at_relname = get_rel_name(tab->at_relid)
tab->at_nspname = get_namespace_name(get_rel_namespace(tab->at_relid))
tab->at_datname = get_database_name(MyDatabaseId)
if any NULL:
  goto deleted
```
这说明即使 `table_recheck_autovac()` 成功，relation 仍可能在下一次 lookup 前消失。`deleted` label 做本表局部清理：
```text
pfree copied names if non-NULL
pfree(tab)
clear wi_tableoid / wi_sharedrel under AutovacuumScheduleLock
set wi_dobalance
continue
```
这条路径仍不是 ERROR。
### 5.9 per-table `PG_TRY/PG_CATCH`
实际执行包在 per-table try 中：
```c
PG_TRY();
{
    MemoryContextSwitchTo(PortalContext);
    autovacuum_do_vac_analyze(tab, bstrategy);
    QueryCancelPending = false;
}
PG_CATCH();
{
    HOLD_INTERRUPTS();
    errcontext("automatic vacuum/analyze of table ...");
    EmitErrorReport();
    AbortOutOfAnyTransaction();
    FlushErrorState();
    MemoryContextReset(PortalContext);
    StartTransactionCommand();
    RESUME_INTERRUPTS();
}
PG_END_TRY();
```
这段代码回答本节主问题的第一部分。单表 ERROR 的收尾不是退出 worker。而是：
```text
记录带表名 context 的错误；
abort 任意活动事务；
清理 ErrorContext；
清理 per-table memory；
重新启动一个事务，继续外层候选表循环。
```
`AbortOutOfAnyTransaction()` 的源码注释很明确：
```text
aborts any active transaction or transaction block,
leaving the system in a known idle state.
```
这比 `MemoryContextReset()` 强得多。
### 5.10 `autovacuum_do_vac_analyze()`
per-table try 里的主调用是：
```text
autovacuum_do_vac_analyze(tab, bstrategy)
  -> build VacuumRelation from tab->at_relid
  -> vacuum(list_make1(vrel), &tab->at_params, bstrategy, vac_context, true)
```
`vacuum()` 对 autovacuum 的关键点：
```text
VACUUM:
  use_own_xacts = true。

ANALYZE in autovacuum:
  use_own_xacts = true。

vacuum_rel():
  entry and exit are outside a transaction；
  每个 relation 自己 StartTransactionCommand / CommitTransactionCommand。
```
因此 catch 可能接住来自多个内部阶段的 ERROR：
```text
open relation；
heap scan；
index vacuum；
heap cleanup；
ANALYZE sampling；
toast vacuum；
parallel vacuum；
AM callback；
stats / progress update。
```
catch 不需要知道具体阶段。它只要求把当前事务状态带回 idle。
### 5.11 `vacuum_open_relation()` 和 lock conflict
`vacuum_rel()` 选择 lock mode：
```text
VACUUM FULL:
  AccessExclusiveLock

lazy VACUUM:
  ShareUpdateExclusiveLock
```
然后调用：
```text
vacuum_open_relation(relid, relation, params.options,
                     params.log_vacuum_min_duration >= 0, lmode)
```
`vacuum_open_relation()` 的关键分支：
```text
if !(options & VACOPT_SKIP_LOCKED):
  rel = try_relation_open(relid, lmode)
else if ConditionalLockRelationOid(relid, lmode):
  rel = try_relation_open(relid, NoLock)
else:
  rel = NULL
  rel_lock = false
```
普通 autovacuum lock conflict 收尾：
```text
ConditionalLockRelationOid() returns false
  -> vacuum_open_relation() returns NULL
  -> vacuum_rel() PopActiveSnapshot()
  -> CommitTransactionCommand()
  -> return false
  -> do_autovacuum() 清 table claim
  -> 继续下一张表
```
这不是 ERROR。它通常也不是 wait。因此你可能抓不到 autovacuum worker 的 lock wait event。
### 5.12 relation dropped
候选 relation 被删除可能被多层捕获。第一层：
```text
SearchSysCache1(RELOID, relid)
  -> invalid: continue
```
第二层：
```text
table_recheck_autovac()
  -> SearchSysCacheCopy1 invalid: return NULL
```
第三层：
```text
get_rel_name() / get_namespace_name()
  -> NULL: goto deleted
```
第四层：
```text
try_relation_open()
  -> NULL: vacuum_open_relation() returns NULL
```
共同语义：
```text
DROP TABLE racing with autovacuum 是正常并发；
目标是跳过不存在的对象并释放本轮 claim；
不是把并发 DROP 变成维护系统错误。
```
### 5.13 worker 退出
worker 正常结束或顶层 ERROR 后都会走 `proc_exit()`。worker 接管 slot 后注册过：
```text
on_shmem_exit(FreeWorkerInfo, 0)
```
`FreeWorkerInfo()` 做：
```text
AutovacuumLock exclusive
dlist_delete(&wi_links)
wi_dboid = InvalidOid
wi_tableoid = InvalidOid
wi_sharedrel = false
wi_proc = NULL
wi_launchtime = 0
clear wi_dobalance
push slot back to av_freeWorkers
MyWorkerInfo = NULL
av_signal[AutoVacRebalance] = true
```
这保证 worker 进程退出不会永久占用 slot。也保证 cost balance 会在 surviving workers 中重新计算。
### 5.14 launcher database 节流
`launch_worker()` 成功选择 database 后更新：
```text
adl_next_worker = now + autovacuum_naptime
dlist_move_head(DatabaseList, entry)
```
`do_start_worker()` 还会跳过最近处理过的 database。原因是 pgstat 可能还没刷新。如果不跳过，launcher 可能在同一个 database 上 tight loop。这就是 retry cleanup 的 database 层语义：
```text
失败不会永久丢弃 database；
但失败也不会立即无限重试同一个 database。
```
## 6. 生命周期 / ownership / cleanup
本节要把 owner 分清，而不是背单个 cleanup 函数：
| 对象 | 谁创建 / 持有 | 正常释放 | ERROR / 异常兜底 |
| --- | --- | --- | --- |
| `WorkerInfoData` | `AutoVacuumShmemInit()` 创建 slot；`do_start_worker()` 分配；worker 接管后持有 | `FreeWorkerInfo()` reset 字段并放回 `av_freeWorkers` | 接管后由 `on_shmem_exit` 兜底；接管前由 launcher `av_startingWorker` timeout 回收；fork failed 由 `AutoVacForkFailed` 触发 1 秒后重发 start signal |
| table claim | `do_autovacuum()` 在 `AutovacuumScheduleLock` 下写 `wi_tableoid` / `wi_sharedrel` | 单表结束后清空 | `table_recheck_autovac()` NULL、relation name 消失、per-table ERROR、worker exit 都会收束 |
| transaction | worker 外层和 `vacuum()` / `vacuum_rel()` / autovacuum ANALYZE 内部按需启动 | `CommitTransactionCommand()` | `AbortOutOfAnyTransaction()` 让系统回到 idle transaction state |
| relation lock | `vacuum_rel()` 用 `ShareUpdateExclusiveLock`，必要时持 session-level lock 保护 main/TOAST 跨事务访问 | `CommitTransactionCommand()` 和 `UnlockRelationIdForSession()` | skip locked 没拿到锁就无需释放；ERROR 由 transaction abort / ResourceOwner 收尾 |
| per-table memory | `PortalContext` | 每张表前后 reset | catch 中 `MemoryContextReset(PortalContext)` |
| error data | `elog.c` 在 `ErrorContext` 中构造 | 非 ERROR 正常 emit 后 reset | catch 中 `EmitErrorReport()` 后 `FlushErrorState()` |
关键区分是：MemoryContext 只管内存；ResourceOwner / transaction abort 管外部资源；`AutovacuumScheduleLock` 管 autovacuum worker 间 claim；heavyweight lock 管 relation 并发；`FlushErrorState()` 管错误报告生命周期。
## 7. 正确性机制层次
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| worker lifecycle | `av_startingWorker`、`av_runningWorkers`、`FreeWorkerInfo()` | worker slot 不永久泄漏 | relation 立即重试 |
| table duplicate avoidance | `AutovacuumScheduleLock` + `wi_tableoid` | autovacuum worker 间避免重复处理同一 relation | 阻塞手工 VACUUM / DDL / DML |
| relation concurrency | heavyweight relation lock | VACUUM 与 DDL / 维护操作的对象级互斥 | MVCC cleanup horizon |
| ordinary lock conflict | `VACOPT_SKIP_LOCKED` + `ConditionalLockRelationOid()` | 普通 autovacuum 低干扰 skip | wraparound 也 skip |
| stale catalog | syscache recheck + `try_relation_open()` | relation drop race 能变成 skip | 候选 list 永远准确 |
| transaction cleanup | `AbortOutOfAnyTransaction()` | locks、pins、snapshots、ProcArray flags 回到事务边界 | 清理 per-table retry 记录 |
| memory cleanup | `PortalContext` reset | per-table allocation 不泄漏 | external resources cleanup |
| error cleanup | `EmitErrorReport()` + `FlushErrorState()` | 错误可记录，ErrorContext 可复用 | 自动修复 relation |
| retry pacing | `autovacuum_naptime` + `DatabaseList` | database 级避免 tight loop | 精确 per-relation backoff |
| timeout policy | worker 强制 timeout GUC 为 0 | 用户 timeout 不随机截断维护 | worker 永不等待 |
关键区分：
```text
lock conflict skip 是正常 fallback；
ERROR catch 是事务 abort fallback；
relation gone 是 stale catalog fallback；
startingWorker timeout 是 worker bootstrap fallback；
fork failed retry 是 postmaster supervision fallback。
```
这些 fallback 不能合并。因为它们清理的 owner 不同。
## 8. 错误路径 / 异常路径 / fallback
| 场景 | 触发点 | 收尾 | 后续重试 |
| --- | --- | --- | --- |
| worker bootstrap ERROR | `InitProcess`、`BaseInit`、GUC override、slot handoff、`InitPostgres` | 顶层 `sigsetjmp` handler 清 `error_context_stack`、`EmitErrorReport()`、`proc_exit(0)`；已接管 slot 由 `FreeWorkerInfo()` 释放，未接管由 launcher timeout 回收 | launcher 后续按 slot / naptime 再启动 |
| postmaster fork failure | postmaster 无法 fork autovacuum worker | `AutoVacWorkerFailed()` 设置 `AutoVacForkFailed`；launcher 清 flag、sleep 1 秒、重发 `PMSIGNAL_START_AUTOVAC_WORKER` | 启动层重试，不涉及 relation |
| database connect failure | stale stats 选中刚删除或打不开的 database | `pgstat_report_autovac(dbid)` 已记录 attempt；`InitPostgres()` ERROR 后 worker 顶层退出 | database schedule 防止 tight loop |
| table recheck NULL | relation 不存在、stats/reloptions 变化、别人已处理 | 清 `wi_tableoid` / `wi_sharedrel`，继续下一候选表 | 下一轮重新 scan / score / recheck |
| ordinary lock conflict | `VACOPT_SKIP_LOCKED` 下 conditional lock 失败 | `vacuum_open_relation()` 返回 NULL；`vacuum_rel()` commit 小事务并 return false；外层清 claim | 没有立即 retry，下一轮再判断 |
| anti-wraparound lock wait | `wraparound == true` 时不设置 `VACOPT_SKIP_LOCKED` | 允许等待 relation lock；ERROR/cancel 时走单表 catch 或 worker exit | 通过等待换取 age 推进 |
| relation dropped after claim | name lookup、syscache、`try_relation_open()` 发现对象消失 | `goto deleted`、return false 或 per-table catch；清 claim | 下一轮只会看到当前 catalog truth |
| lazy VACUUM internal ERROR | heap scan、index cleanup、AM callback、parallel vacuum 等抛错 | `vacuum.c` `PG_FINALLY` 清 vacuum 全局状态；`do_autovacuum()` catch abort xact / flush error / reset context / 清 claim | 同 worker 继续下一张候选表 |
| work item ERROR | BRIN summarize 等 `av_workItems` 抛错 | 独立 `PG_TRY/PG_CATCH`，同样 abort / flush / reset；源码说明 work item list can be lossy | 调用方接受 lossy 语义 |
worker 初始化还强制关闭 `statement_timeout`、`transaction_timeout`、`lock_timeout`、`idle_in_transaction_session_timeout`。这不是禁止取消，而是避免用户 timeout 随机截断维护，特别是 anti-wraparound vacuum。`SIGINT` 取消当前表，`SIGTERM` clean exit，`SIGQUIT` abandon ship。
## 9. 成本、资源与跨模块传播
### 9.1 为什么没有 durable per-table retry queue
一个显然的替代设计是：
```text
失败表写入 retry queue；
保存 retry_count、last_error、next_retry_time；
launcher 按 queue 精确重试。
```
PostgreSQL 没有这样做。原因包括：
```text
relation OID 可被 DROP / recreate；
reloptions、权限、namespace、TOAST 映射都会变；
pgstat 是异步近似；
wraparound priority 随时间变化；
durable queue 要处理 crash recovery 和 invalidation；
跨 database queue 会制造全局 contention；
很多失败只是 lock conflict，不值得记录成 durable 事件。
```
当前折中是：
```text
下一轮重新 scan catalog；
重新读 stats；
重新计算 threshold / age；
执行前重新 recheck；
用 naptime 控制 database 级节奏。
```
### 9.2 lock conflict 的成本
普通 skip locked 降低前台干扰。代价是：
```text
热点表可能多轮被跳过；
n_dead_tup 和 bloat 继续增长；
age 继续逼近阈值；
诊断上缺少精确 skip counter。
```
wraparound 不 skip 的代价是：
```text
worker 可能进入 Lock wait；
blocking chain 可能暴露业务长事务或 DDL；
维护压力从 bloat / age 变成锁等待。
```
### 9.3 ERROR cleanup 的成本
单表 ERROR 后完整 abort 当前事务。这比局部清理重。但它把复杂度压到事务系统：
```text
ResourceOwnerRelease；
AtEOXact_Buffers；
AtEOXact_RelationCache；
AtEOXact_Inval；
AtEOXact_Snapshot；
AtEOXact_PgStat；
AtEOXact_Files；
AtEOXact_SMgr。
```
autovacuum.c 不需要知道每个子模块的资源细节。
### 9.4 cost balance 传播
worker 退出：
```text
FreeWorkerInfo()
  -> av_signal[AutoVacRebalance] = true
```
表级 reloptions 改变：
```text
table_recheck_autovac()
  -> at_storage_param_vac_cost_*
  -> wi_dobalance
  -> VacuumUpdateCosts()
```
如果 ERROR 后不清 `wi_dobalance` 或 WorkerInfo，surviving workers 的 cost limit 会被错误分摊。所以 cost balance 也是 cleanup 的一部分。
### 9.5 stats 和 progress 边界
`pg_stat_progress_vacuum` 是当前命令进度。ERROR 或命令结束后，progress entry 会消失或清掉。`pg_stat_all_tables.last_autovacuum` 是 relation 级成功维护证据。database-level `last_autovac_time` 可以代表 worker attempt。不要把三者混成同一个事实。
## 10. 观测与诊断入口
### 10.1 当前 worker
```sql
select pid, backend_type, datname, state, wait_event_type, wait_event, query
from pg_stat_activity
where backend_type like 'autovacuum%';
```
能看到：
```text
launcher / worker 是否存在；
worker 当前 database；
是否处于 Lock、Timeout、IO 或其它等待；
activity string 中的当前 relation 线索。
```
不能看到：
```text
wi_tableoid 的精确写入/清空时刻；
table_recheck_autovac() 是否返回 NULL；
普通 skip locked 是否发生过。
```
### 10.2 progress
```sql
select *
from pg_stat_progress_vacuum;
```
能看到：
```text
当前 VACUUM phase；
heap blocks scanned / vacuumed；
index vacuum 进度；
dead tuple bytes；
是否由 autovacuum 启动。
```
看不到：
```text
候选 list 中尚未执行的表；
已经 skip 的表；
ERROR 后继续下一张表的历史。
```
### 10.3 locks
```sql
select a.pid, a.backend_type, l.locktype, l.mode, l.granted,
       l.relation::regclass, a.wait_event_type, a.wait_event
from pg_locks l
join pg_stat_activity a using (pid)
where a.backend_type = 'autovacuum worker';
```
能看到：
```text
worker 已持有的 relation lock；
wraparound 或非 skip path 的 lock wait；
blocking chain。
```
普通 autovacuum skip locked 时，可能没有等待记录。
### 10.4 relation stats
```sql
select relid::regclass, n_dead_tup, last_autovacuum, autovacuum_count,
       last_autoanalyze, autoanalyze_count
from pg_stat_all_tables
order by n_dead_tup desc
limit 20;
```
能看到成功维护后的 relation-level stats。不能看到失败尝试。`last_autovacuum` 没变，不等于 autovacuum 没看过这张表。
### 10.5 日志
建议实验环境：
```conf
log_autovacuum_min_duration = 0
log_min_messages = info
```
成功 VACUUM 日志可能包含：
```text
automatic vacuum of table ...
automatic vacuum to prevent wraparound of table ...
pages removed / remaining；
tuples removed / remain；
dead but not yet removable；
buffer / WAL / system usage。
```
ERROR 日志会带：
```text
automatic vacuum of table "db.schema.rel"
automatic analyze of table "db.schema.rel"
while scanning block ...
while vacuuming index ...
```
但 lock skip 和 relation gone 不一定有日志。
### 10.6 三类状态
能直接观测：
```text
worker backend_type / wait_event；
current progress phase；
pg_locks 中已持有或等待的锁；
成功维护后的 relation stats；
server log 的 summary 和 ERROR context。
```
只能推断：
```text
普通 skip locked；
table_recheck_autovac() 返回 NULL；
候选表是否被 worker 看见；
下一轮是否会再次入选。
```
需要断点或临时日志：
```text
wi_tableoid 写入/清空；
AutovacuumScheduleLock claim 冲突；
av_startingWorker timeout 前状态；
PortalContext reset 的实际内存；
ResourceOwner 在 ERROR 前持有什么。
```
## 11. 常见误区
### 11.1 “worker ERROR 会让 autovacuum 停掉”
单表 ERROR 不会停掉整个 worker。per-table catch 会 abort 当前事务并继续下一张候选表。顶层 worker ERROR 会让当前 worker 退出。launcher 后续仍会启动新 worker。
### 11.2 “锁冲突一定能在 pg_stat_activity 里看到 wait”
普通 autovacuum 多数 relation lock 冲突走 `VACOPT_SKIP_LOCKED`。它可能直接 skip。你抓不到 Lock wait。anti-wraparound vacuum 才更可能等待。
### 11.3 “wi_tableoid 就是正在 VACUUM”
`wi_tableoid` 是 autovacuum worker claim。它不是 progress phase。它不是成功维护证明。它也不阻止 DDL 或手工 VACUUM。
### 11.4 “DROP TABLE racing with autovacuum 应该报错”
并发 DROP 是正常现象。源码多处把 relation gone 转成 NULL / continue。目标是跳过并清理 claim。
### 11.5 “MemoryContext reset 就够了”
MemoryContext reset 只清内存。事务 abort 才释放 locks、pins、snapshots、ProcArray flags 和其它外部资源。ErrorContext 还需要 `FlushErrorState()`。
### 11.6 “last_autovacuum 没变说明没尝试”
`last_autovacuum` 是 relation-level success。skip locked、recheck NULL、relation gone、ERROR 都可能不更新它。诊断必须结合日志、activity、locks、progress 和时间线。
### 11.7 “statement_timeout 可以限制 autovacuum”
worker 会强制把 statement、transaction、lock、idle-in-transaction timeout 设为 0。控制 autovacuum 应使用 autovacuum 自己的配置、reloptions、cost 参数和必要时信号。
## 12. 课堂实验
### 实验 1：普通 autovacuum skip locked
目标：
```text
观察普通 autovacuum 遇到 relation lock conflict 时更可能 skip，而不是等待。
```
准备：
```sql
create table av_retry_demo(id int primary key, v text);
insert into av_retry_demo
select g, repeat('x', 100)
from generate_series(1, 200000) g;
delete from av_retry_demo where id % 2 = 0;
analyze av_retry_demo;
```
会话 A：
```sql
begin;
lock table av_retry_demo in access exclusive mode;
```
观察 autovacuum worker：
```sql
select pid, backend_type, wait_event_type, wait_event, query
from pg_stat_activity
where backend_type = 'autovacuum worker';
```
源码回扣：
```text
table_recheck_autovac()
  -> !wraparound sets VACOPT_SKIP_LOCKED

vacuum_open_relation()
  -> ConditionalLockRelationOid()
  -> false means return NULL
```
注意：
```text
真实 autovacuum 是否选中该表，取决于 threshold、naptime、stats freshness 和 worker slot。
```
### 实验 2：DROP TABLE racing with autovacuum
目标：
```text
验证候选 relation 被删除时 worker 不需要崩溃。
```
建议断点：
```text
do_autovacuum() claim 后；
table_recheck_autovac() 前后；
get_rel_name() 前；
vacuum_open_relation()。
```
步骤：
```text
1. 让 worker 选中一张测试表。
2. 在 claim 后暂停 worker。
3. 另一个会话 DROP TABLE。
4. 放行 worker。
5. 观察它走 NULL / deleted / return false 的哪条路径。
```
观察变量：
```text
relid
MyWorkerInfo->wi_tableoid
tab
tab->at_relname
```
### 实验 3：单表 ERROR 后继续下一张
目标：
```text
验证 per-table PG_CATCH 的隔离边界。
```
建议在本地实验源码中加临时 injection point：
```text
在 autovacuum_do_vac_analyze() 看到指定 relid 时 ereport(ERROR)。
```
准备两张都满足 autovacuum 条件的表。观察：
```text
ERROR 表：
  EmitErrorReport()
  AbortOutOfAnyTransaction()
  FlushErrorState()
  MemoryContextReset(PortalContext)
  clear wi_tableoid

下一张表：
  worker 重新 StartTransactionCommand()
  继续处理。
```
SQL 对照：
```sql
select relid::regclass, last_autovacuum, autovacuum_count
from pg_stat_all_tables
where relid in ('t_error'::regclass, 't_next'::regclass);
```
## 13. 讨论题
1. 为什么 per-table ERROR handler 必须调用 `AbortOutOfAnyTransaction()`，而不是只 reset memory context？
2. `wi_tableoid` 缺少哪些上下文，导致它不能单独解释为“正在 VACUUM”？
3. 普通 autovacuum 使用 skip locked 的收益和代价分别是什么？
4. anti-wraparound vacuum 如果也一直 skip locked，会破坏什么全局边界？
5. 为什么候选表已经来自 `pg_class` scan，执行前仍要 `table_recheck_autovac()`？
6. 如果 ERROR 后忘记清 `wi_tableoid`，其它 worker 会出现什么行为？
7. `last_autovacuum`、database `last_autovac_time`、progress view 和 server log 分别是什么粒度？
8. 如果你要设计 per-table retry queue，需要处理哪些 invalidation、crash recovery 和 OID reuse 问题？
## 14. 本节小结
本节唯一主问题：
```text
worker ERROR、锁冲突、表被删除和重试节流分别如何收尾？
```
核心链路：
```text
launcher 分配 WorkerInfo
  -> worker 接管并注册 FreeWorkerInfo
  -> do_autovacuum 扫 pg_class
  -> table claim 写 wi_tableoid
  -> table_recheck_autovac 重新判断
  -> vacuum_open_relation 处理 lock conflict / relation gone
  -> autovacuum_do_vac_analyze 执行 VACUUM / ANALYZE
  -> per-table PG_CATCH 处理 ERROR
  -> 清 table claim
  -> worker exit 归还 WorkerInfo
  -> launcher 按 naptime / startingWorker / fork failure 控制下一轮
```
四种收尾要分开：
```text
worker ERROR:
  顶层 handler 记录错误并退出进程，由 FreeWorkerInfo 或 launcher timeout 回收 slot。

单表 ERROR:
  per-table PG_CATCH 记录 context、abort transaction、flush error state、reset PortalContext，继续下一张表。

锁冲突:
  普通 autovacuum 通过 VACOPT_SKIP_LOCKED 非阻塞 skip；
  anti-wraparound vacuum 可能等待。

表被删除:
  syscache、name lookup、try_relation_open 多层 recheck，把 stale candidate 变成 skip。
```
正确性边界：
```text
MemoryContext 管内存；
ResourceOwner / transaction abort 管外部资源；
AutovacuumScheduleLock 管 worker 间 table claim；
heavyweight lock 管 relation 并发；
ErrorContext / FlushErrorState 管错误报告生命周期；
launcher schedule 管重试节奏。
```
观测边界：
```text
pg_stat_activity 和 pg_locks 能看到当前 worker 与 lock wait；
pg_stat_progress_vacuum 能看到当前 VACUUM phase；
pg_stat_all_tables 能看到成功维护后的 relation stats；
server log 能看到成功 summary 和 ERROR context；
skip locked、recheck NULL、wi_tableoid 瞬时变化和 per-table retry intent 多半只能推断或断点观察。
```
可迁移规律：
```text
长期后台维护系统不应该把失败处理押在一个全局 catch 上。
它需要把 ownership 拆成 process、shared slot、current object claim、transaction、
external resource、memory context、error state 和 scheduler backoff 多层；
每层用自己的 cleanup 边界收束，下一轮再基于当前事实重新决策。
```
需要保留的推断边界：
```text
某张表是否被 skip locked 通常没有直接计数；
worker 是否看过某张表取决于时序、stats freshness、score sorting 和 reloptions；
wraparound lock wait 的影响取决于 workload 和 blocking chain；
日志粒度受 log_autovacuum_min_duration、log_min_messages 和源码版本影响；
autovacuum scoring、worker slots 和 progress 字段会随 PostgreSQL 版本演化。
```
