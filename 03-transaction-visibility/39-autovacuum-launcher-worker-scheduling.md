# PostgreSQL autovacuum launcher / worker 调度

## 课程定位

前置知识：已经理解 VACUUM 的 heap / index 分阶段清理、freeze 与 anti-wraparound 语义、visibility map、后台进程、latch、LWLock、shared memory 与 pgstat 的基本作用。

本节唯一主问题：

```text
autovacuum launcher 如何扫描 database、调度 worker，
并在全局 worker 限制下选择下一个维护目标？
```

核心矛盾：

```text
系统必须持续推进 vacuum / analyze / freeze，
否则 dead tuple、统计信息和 XID/MultiXact age 会变成全局风险；
但 launcher 又不能频繁扫描所有 database 的所有 relation，
不能无限启动 worker，
也不能让多个 worker 重复维护同一个目标。
```

本节只讨论调度：launcher 何时醒来、如何选择 database、如何向 postmaster 请求 worker、worker 如何在 database 内选择 relation、全局 worker slot 如何限制并发、table claim 和 cost balance 如何避免重复与资源放大。

本节不重复第 28 节的 lazy VACUUM heap / index cleanup，也不展开第 40 节的 threshold / scale factor 和第 41 节的 anti-wraparound 细节。这里关心的是“谁被选中去执行维护”。

学完后应能判断：

```text
为什么 launcher 保存 database schedule，而不是全局 table queue；
为什么 database 选择使用 pg_database + pgstat，relation 选择必须留给 worker；
为什么 autovacuum_worker_slots 和 autovacuum_max_workers 是两个不同限制；
为什么 av_startingWorker 一次只允许一个 worker 处于启动交接；
为什么同一张表需要 AutovacuumScheduleLock 做 claim；
为什么 worker 还要 table_recheck_autovac()；
为什么 cost limit 要在运行 worker 之间动态平衡；
哪些调度状态能从 SQL / log / wait event 看到，哪些只能推断或断点观察。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。

## 1. 本节在总主线中的位置

前面的 VACUUM 课程回答的是 tuple、page、relation 内部的回收边界。本节把主语换成整个实例：实例里有多个 database，每个 database 又有多个 relation；维护动作必须自动推进，但不能把后台维护变成比前台 workload 更大的干扰源。

第 28 节的核心对象是一次 `vacuum()` 内的 `LVRelState` 和 `dead_items`。本节的核心对象是 `DatabaseList`、`WorkerInfoData`、`AutoVacuumShmemStruct`、`TableToProcess` 和 `AutoVacuumScores`。前者解释“VACUUM 如何清理一张表”，后者解释“autovacuum 如何决定让谁去清理哪张表”。

autovacuum 调度跨越五个模块：

```text
postmaster:
  监督 launcher / worker 进程，真正 fork worker。

launcher:
  常驻进程，维护 database-level schedule，决定何时请求 worker。

worker:
  连接一个 database，扫描 pg_class，选择 relation 并调用 vacuum()/analyze。

pgstat:
  提供 database last_autovac_time 和 relation dead/mod/insert counters。

VACUUM/ANALYZE:
  执行真实维护并回写 relation stats、progress 和 catalog frozen xid。
```

本节的 runtime 现象锚点：

```text
把 autovacuum_naptime 调低，把 autovacuum_max_workers 调小；
制造多个 database / relation 的 dead tuples；
观察 pg_stat_activity 中 launcher 的 AutovacuumMain wait event、
worker 并发数上限、pg_stat_progress_vacuum、pg_stat_all_tables 的 last_autovacuum；
再回到 AutoVacLauncherMain()、do_start_worker()、do_autovacuum()。
```

这个现象会证明：autovacuum 不是“每张表一个定时器”，也不是“全局 top-N 脏表队列”。它是两级近似调度：

```text
launcher 粗粒度选择 database；
worker 在该 database 内精细选择 relation；
shared memory 只保存必要 ownership 和 worker whereabouts。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
AutoVacLauncherMain() 用 DatabaseList 维护 database 的下一次可启动时间；
到期且 av_worker_available() 为真时，do_start_worker() 扫 pg_database 和 database pgstat，
把一个 WorkerInfo 从 av_freeWorkers 移到 av_startingWorker，
再用 PMSIGNAL_START_AUTOVAC_WORKER 让 postmaster fork worker；
AutoVacWorkerMain() 接管 WorkerInfo、连接 database、运行 do_autovacuum()；
do_autovacuum() 扫 pg_class 和 relation pgstat，按 AutoVacuumScores 排序候选表，
通过 AutovacuumScheduleLock claim table，recheck 后调用 vacuum()/analyze()；
退出时 FreeWorkerInfo() 归还 slot，并触发 cost rebalance。
```

这条模型故意分成 database 选择和 relation 选择。launcher 不连接每个 database 去扫 `pg_class`，所以它只能使用 `pg_database` 的 `datfrozenxid` / `datminmxid` 和 database-level pgstat。worker 已经连接一个 database，可以读 reloptions、pg_class、relation-level pgstat，所以它才能判断具体表。

如果 launcher 维护全局 relation queue，它必须跨 database 收集 relation stats、reloptions、catalog snapshot 和权限上下文，还要处理 database drop、cache invalidation、shared catalog 和 stale stats。这会把一个低侵入后台调度器变成高 contention 全局元数据系统。

如果 worker 随机选择 database，热 database 可能重复被选中，冷 database 或 wraparound 风险 database 可能被饿死。如果 worker 数无限增长，VACUUM I/O、WAL、buffer churn、relation locks 和 CPU 会放大前台延迟。

PostgreSQL 的折中是：

```text
launcher:
  轻量 database schedule，近似公平，wraparound 优先。

worker:
  database-local relation scoring，执行前 recheck。

shared state:
  只记录 worker lifecycle、当前 table claim、work item 和 cost balance 计数。

执行层:
  仍由 vacuum.c / analyze.c 保证 visibility、lock、WAL 和 cleanup correctness。
```

这个实现不是理想化 architecture。`DatabaseList` 是 launcher-local，worker slot 在 shared memory，postmaster 通过 PMSIGNAL 间接启动 worker，relation 候选先收集再 recheck，database schedule 与 table score 使用不同信息源。这些历史痕迹共同服务一个目标：在可恢复失败、低共享状态和低全局扫描成本下，持续推进维护。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/autovacuum.c` | 主文件：`AutoVacLauncherMain()`、`launcher_determine_sleep()`、`rebuild_database_list()`、`do_start_worker()`、`launch_worker()`、`AutoVacWorkerMain()`、`do_autovacuum()`、`relation_needs_vacanalyze()`。 |
| 2 | `src/include/postmaster/autovacuum.h` | autovacuum GUC extern、launcher / worker 入口、`AutoVacuumRequestWork()`。 |
| 3 | `src/include/storage/lwlocklist.h` | 固定 LWLock：`AutovacuumLock`、`AutovacuumScheduleLock`。 |
| 4 | `src/backend/storage/lmgr/lwlock.c` | LWLock acquire/release/wait event 基础，本节只关心短临界区和等待观测。 |
| 5 | `src/backend/postmaster/postmaster.c` | `StartAutovacuumWorker()`、fork failure、worker exit 后唤醒 launcher。 |
| 6 | `src/backend/postmaster/pmchild.c` | `B_AUTOVAC_WORKER` child pool 大小来自 `autovacuum_worker_slots`。 |
| 7 | `src/include/postmaster/proctypelist.h` | `B_AUTOVAC_LAUNCHER`、`B_AUTOVAC_WORKER` 与 main function 绑定。 |
| 8 | `src/backend/utils/activity/pgstat_database.c` | `pgstat_report_autovac()`、`pgstat_fetch_stat_dbentry()`。 |
| 9 | `src/backend/utils/activity/pgstat_relation.c` | `pgstat_report_vacuum()`、`pgstat_report_analyze()` 更新 relation autovacuum / autoanalyze stats。 |
| 10 | `src/include/pgstat.h` | `PgStat_StatDBEntry` 与 relation stats 字段。 |
| 11 | `src/backend/utils/activity/wait_event_names.txt` | `AUTOVACUUM_MAIN`、`Autovacuum`、`AutovacuumSchedule`。 |
| 12 | `src/include/commands/progress.h`、`src/backend/access/heap/vacuumlazy.c` | `PROGRESS_VACUUM_STARTED_BY_AUTOVACUUM` 等 progress 入口。 |

推荐阅读顺序：

```text
state structs
  -> launcher main loop
  -> database list rebuild
  -> do_start_worker database choice
  -> worker takeover
  -> do_autovacuum relation choice
  -> table claim / recheck
  -> cleanup and failure paths
```

当前本地源码树没有 `src/backend/catalog/pg_database.c`。本课仍然讨论 `pg_database` 扫描，但真实实现路径在 `autovacuum.c` 的 `get_database_list()`：`table_open(DatabaseRelationId)`、`table_beginscan_catalog()`、`heap_getnext()`。源码课程要以当前本地树为准。

## 4. 关键数据结构与状态

### 4.1 `avl_dbase`: launcher-local schedule entry

`avl_dbase` 是 launcher 的 `DatabaseList` entry，不在 shared memory 中。它位于 `DatabaseListCxt`，ERROR recovery 或 rebuild 会丢弃并重建。

| 字段 | 语义 |
| --- | --- |
| `adl_datid` | database OID，也是 hash key。 |
| `adl_next_worker` | 这个 database 下一次允许启动 worker 的时间。 |
| `adl_score` | rebuild 时保存相对顺序的临时分数。 |
| `adl_node` | `DatabaseList` 的 dlist 节点。 |

核心不变量：

```text
DatabaseList 按 adl_next_worker 从远到近组织；
head 是更晚处理的 database；
tail 是最早到期的 database；
某 database 被 launch 后，adl_next_worker = now + autovacuum_naptime，并移到 head。
```

因此 `DatabaseList` 是 schedule cache，不是 durable queue，也不是其它进程可见的 shared truth。

### 4.2 `avw_dbase`: database candidate

`avw_dbase` 是 `get_database_list()` 返回的候选，分配在调用者 context。

| 字段 | 语义 |
| --- | --- |
| `adw_datid` | database OID。 |
| `adw_name` | database name，主要用于日志或调试。 |
| `adw_frozenxid` | `pg_database.datfrozenxid`，XID wraparound 输入。 |
| `adw_minmulti` | `pg_database.datminmxid`，MultiXact wraparound 输入。 |
| `adw_entry` | `PgStat_StatDBEntry`，`do_start_worker()` 后续填充。 |

launcher 没有连接具体 database，但可以在特殊初始化状态下 scan `pg_database`。这让 database 选择保持轻量，也意味着 launcher 不能直接知道每个 database 内哪些 table 最急。

### 4.3 `AutoVacuumShmemStruct`: 全局调度共享状态

`AutoVacuumShmemRequest()` 按 `autovacuum_worker_slots` 申请 fixed struct + `WorkerInfoData` 数组；`AutoVacuumShmemInit()` 初始化 free list、running list、starting pointer、work items 和 cost balance counter。

| 字段 | 语义 |
| --- | --- |
| `av_signal[]` | 其它进程设置给 launcher 的 shared flags，如 fork failed、rebalance。 |
| `av_launcherpid` | 当前 launcher pid。 |
| `av_freeWorkers` | 空闲 `WorkerInfoData` 链表。 |
| `av_runningWorkers` | 已接管并运行中的 worker 链表。 |
| `av_startingWorker` | launcher 已分配、postmaster 正在启动、worker 尚未接管的单个 slot。 |
| `av_workItems` | 其它模块请求 autovacuum 处理的附加任务，如 BRIN summarize。 |
| `av_nworkersForBalance` | 参与 cost limit 平衡的 worker 数。 |

大部分字段由 `AutovacuumLock` 保护。`wi_tableoid` / `wi_sharedrel` 的 claim 语义由 `AutovacuumScheduleLock` 保护。不要把两个 LWLock 混成一个“autovacuum 大锁”：前者管 worker lifecycle，后者管 table claim 原子窗口。

### 4.4 `WorkerInfoData`: worker whereabouts

`WorkerInfoData` 是 shared memory 中每个 autovacuum worker slot 的状态。

| 字段 | 语义 |
| --- | --- |
| `wi_dboid` | worker 目标 database。 |
| `wi_tableoid` | 当前 claim 的 relation；空闲时为 `InvalidOid`。 |
| `wi_sharedrel` | 当前 relation 是否 shared catalog。 |
| `wi_proc` | worker 接管后写入自己的 `PGPROC *`。 |
| `wi_launchtime` | launcher 放入 `av_startingWorker` 的时间。 |
| `wi_dobalance` | atomic flag，用于 cost balance 分支判断。 |
| `wi_links` | free list 或 running list 节点。 |

生命周期：

```text
free:
  在 av_freeWorkers 中，未绑定 database。

starting:
  launcher 弹出 free slot，写 wi_dboid / wi_launchtime，挂到 av_startingWorker。

running:
  worker 写 wi_proc，移入 av_runningWorkers，注册 FreeWorkerInfo。

claiming table:
  worker 写 wi_tableoid / wi_sharedrel，防止其它 worker 重复维护。

free again:
  FreeWorkerInfo 清字段并放回 av_freeWorkers。
```

`wi_proc == NULL` 不等于 slot 空闲。slot 在 `av_startingWorker` 中时，`wi_proc` 正常就是 `NULL`，表示 postmaster 或 worker 还没完成接管。

### 4.5 `AutoVacuumScores` 与 `TableToProcess`

`AutoVacuumScores` 是 relation-level 调度分数，`TableToProcess` 只保存 `oid` 和 `score`。

| 分数 | 来源 |
| --- | --- |
| `xid` | `relfrozenxid` age / freeze max age，带 weight。 |
| `mxid` | `relminmxid` age / multixact freeze max age，带 weight。 |
| `vac` | `dead_tuples` / vacuum threshold，带 weight。 |
| `vac_ins` | `ins_since_vacuum` / insert vacuum threshold，带 weight。 |
| `anl` | `mod_since_analyze` / analyze threshold，带 weight。 |
| `max` | 上述组件最大值，用于排序。 |

worker 第一遍 scan 只把候选 OID 和 score 放进 `tables_to_process`。真正执行前还要重新读取 catalog / reloptions / pgstat。候选列表不是强一致任务队列，它只是当前 transaction 中的一次估算。

### 4.6 `autovac_table`: recheck 后的执行包

`table_recheck_autovac()` 返回 `autovac_table`。它保存 relation OID、`VacuumParams`、per-table cost 参数、是否参与 cost balance，以及用于 error context / activity string 的 relation、schema、database name。

如果返回 `NULL`，worker 必须释放 table claim 并继续。返回 `NULL` 的常见原因是 relation 被 drop、stats 已被别人更新、手动 VACUUM 抢先处理、reloptions 改变或 threshold 不再满足。

### 4.7 `AutoVacuumWorkItem`

`AutoVacuumWorkItem` 是附加任务槽，当前公开类型包括 `AVW_BRINSummarizeRange`。它用 `avw_used`、`avw_active`、`avw_database`、`avw_relation` 和 `avw_blockNumber` 描述任务。worker 完成普通 table list 后，会处理属于当前 `MyDatabaseId` 的 work item。

它不是本节主线，但能说明 autovacuum worker 不只执行 threshold-driven VACUUM / ANALYZE，也能承接模块提交的 database-local maintenance work。

## 5. 主流程源码 walkthrough

### 5.1 shared memory 和 process type 初始化

autovacuum 不是 extension background worker。`proctypelist.h` 把 `B_AUTOVAC_LAUNCHER` 绑定到 `AutoVacLauncherMain()`，把 `B_AUTOVAC_WORKER` 绑定到 `AutoVacWorkerMain()`。

启动期顺序：

```text
postmaster startup
  -> AutoVacuumShmemRequest()
  -> AutoVacuumShmemInit()
  -> autovac_init() 检查配置
  -> later StartChildProcess(B_AUTOVAC_LAUNCHER)
```

`autovacuum_worker_slots` 在 shared memory 和 `pmchild.c` 的 child pool 中确定容量，postmaster 启动后不能靠 SIGHUP 扩大。`autovacuum_max_workers` 是 launcher 实际可用的并发上限。若 `worker_slots < max_workers`，源码只 warning，实际上限仍是 slots。

### 5.2 launcher 初始化

`AutoVacLauncherMain()` 先释放 `PostmasterContext`，安装 signal handlers，调用 `InitProcess()`、`BaseInit()`、`InitPostgres(NULL, InvalidOid, ...)`，进入 `NormalProcessing`，创建 `AutovacMemCxt`。

`InitProcess()` 让 launcher 拥有 `PGPROC`，从而能使用 shared memory、LWLock 和 latch。`InitPostgres(NULL, InvalidOid, ...)` 不表示连接普通 database；它给 launcher 足够的 catalog / transaction 基础设施，用于 scan `pg_database`。

launcher 强制设置：

```text
search_path = ''
zero_damaged_pages = false
statement_timeout = 0
transaction_timeout = 0
lock_timeout = 0
idle_in_transaction_session_timeout = 0
default_transaction_isolation = read committed
stats_fetch_consistency = none
```

这些是后台维护边界：autovacuum 不应该被用户级 timeout 阻止，不应该承受 serializable 额外成本，也需要每次取最新 pgstat 视图。

### 5.3 emergency mode

`AutoVacuumingActive()` 同时要求 `autovacuum_start_daemon` 和 `pgstat_track_counts`。正常 autovacuum inactive 时，launcher 有特殊路径：

```text
if (!AutoVacuumingActive())
{
    if (!ShutdownRequestPending)
        do_start_worker();
    proc_exit(0);
}
```

这服务 emergency / anti-wraparound 场景。`autovacuum=off` 不等于永远没有 autovacuum worker；防止 XID/MultiXact wraparound 是 correctness requirement。

### 5.4 `rebuild_database_list()` 建立 schedule

正常模式下，launcher 写 `AutoVacuumShmem->av_launcherpid = MyProcPid`，然后 `rebuild_database_list(InvalidOid)`。

`rebuild_database_list()` 的目标是建立 database schedule，不是判断哪个表最脏。它会：

```text
创建新的 DatabaseListCxt 和 tmp context；
用 hash 保存 database OID -> avl_dbase；
先插入 newdb，再保留旧 DatabaseList 中仍有 pgstat entry 的 database；
调用 get_database_list() scan pg_database；
只加入有 pgstat entry 的普通候选；
按 adl_score 排序；
把候选均匀分布到 autovacuum_naptime 区间；
用 adl_next_worker 构建新的 DatabaseList；
删除旧 DatabaseListCxt。
```

`millis_increment = 1000 * autovacuum_naptime / nelems`，但不会小于 `MIN_AUTOVAC_SLEEPTIME` 附近。database 很多时，`autovacuum_naptime` 是调度目标，不是每个 database 的精确定时器。

### 5.5 launcher wait loop

launcher 主循环先计算 sleep：

```text
launcher_determine_sleep(av_worker_available(), false, &nap)
```

如果当前不能启动 worker，就 sleep `autovacuum_naptime`。如果 `DatabaseList` 非空，就看 tail entry 的 `adl_next_worker`。如果 list 为空，也 sleep `autovacuum_naptime`。

当 sleep 结果为 0，源码会 `rebuild_database_list()` 并只递归重算一次。这个 fallback 防止 worker 长时间忙碌后多个 database 同时过期，launcher 进入紧密循环。

等待点：

```text
WaitLatch(MyLatch,
          WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
          timeout,
          WAIT_EVENT_AUTOVACUUM_MAIN)
```

`pg_stat_activity` 中 `wait_event = AutovacuumMain` 只能说明 launcher 在主循环等待。它不显示下一个 database，也不区分 timeout、SIGHUP、SIGUSR2 或 postmaster death。

### 5.6 SIGUSR2 和 launcher flags

`avl_sigusr2_handler()` 只做：

```text
got_SIGUSR2 = true;
SetLatch(MyLatch);
```

醒来后主循环处理 `got_SIGUSR2`，再看 shared flags：

```text
AutoVacRebalance:
  持 AutovacuumLock，重算 av_nworkersForBalance。

AutoVacForkFailed:
  清 flag，sleep 1s，重新发送 PMSIGNAL_START_AUTOVAC_WORKER。
```

SIGUSR2 可能表示 worker 接管完成、worker 退出或 postmaster fork failure。它只是 wakeup，真实语义在 `AutoVacuumShmem` 中。

### 5.7 worker slot 可用性

`av_worker_available()` 是全局 worker 限制的核心：

```text
free_slots = count(av_freeWorkers)
reserved_slots = max(autovacuum_worker_slots - autovacuum_max_workers, 0)
return free_slots > reserved_slots
```

这解释了两个 GUC 的关系。`autovacuum_worker_slots` 是容量，`autovacuum_max_workers` 是可用配额；当 max 大于 slots，实际上限仍是 slots。

launcher 还检查 `av_startingWorker`。只要有 worker 处于 starting 状态，就不再启动下一个 worker。如果 starting 超过 `min(autovacuum_naptime, 60s)`，launcher 会把 slot 清回 freelist 并 warning：`autovacuum worker took too long to start; canceled`。

### 5.8 `do_start_worker()` 选择 database

`do_start_worker()` 先在 `AutovacuumLock` 下快速确认有 worker slot，然后进入临时 context，调用 `get_database_list()`。

`get_database_list()` 是 launcher 唯一使用 transaction 的函数。它 `table_open(DatabaseRelationId, AccessShareLock)`，对 `pg_database` 做 catalog scan，跳过 `database_is_invalid_form()`，记录 database OID、name、`datfrozenxid`、`datminmxid`。

database 选择优先级：

```text
1. 若有 datfrozenxid 早于 recentXid - autovacuum_freeze_max_age，
   选 datfrozenxid 最老的 database。

2. 否则若有 datminmxid 早于 recentMulti - MultiXactMemberFreezeThreshold()，
   选 datminmxid 最老的 database。

3. 否则只考虑有 pgstat database entry 的 database；
   跳过 DatabaseList 显示刚被安排过的 database；
   选择 last_autovac_time 最老的 database。
```

普通 database 没有 pgstat entry 会被跳过，因为没有活动统计就不值得定期处理。wraparound 风险是例外：它不能因为 stats 缺失而被忽略。

源码注释还指出一个限制：launcher 没有 database-level “表脏度” metric，因此普通选择使用 `last_autovac_time` 近似公平，而不是跨 database 统计所有 relation。

### 5.9 发布 starting worker

选中 database 后：

```text
LWLockAcquire(AutovacuumLock, LW_EXCLUSIVE)
pop av_freeWorkers
worker->wi_dboid = selected_db
worker->wi_proc = NULL
worker->wi_launchtime = now
AutoVacuumShmem->av_startingWorker = worker
LWLockRelease(AutovacuumLock)
SendPostmasterSignal(PMSIGNAL_START_AUTOVAC_WORKER)
```

launcher 不直接 fork worker。postmaster 是进程监督者，负责 child slot、fork/exec、crash 和 exit accounting。launcher 只发布 intent 并发 PMSIGNAL。

`launch_worker(now)` 包装 `do_start_worker()`。如果返回有效 dbid，就在 `DatabaseList` 中把该 database 的 `adl_next_worker` 推进到 `now + autovacuum_naptime`，并移到 head；如果 list 中找不到，就 `rebuild_database_list(dbid)`。

### 5.10 postmaster fork 与 failure

`postmaster.c` 看到 `PMSIGNAL_START_AUTOVAC_WORKER` 后调用 `StartAutovacuumWorker()`。如果 fork 或 slot 获取失败，postmaster 调 `AutoVacWorkerFailed()`，设置 `av_signal[AutoVacForkFailed]`，再 signal launcher。

launcher 收到后不会重选 database，因为 `av_startingWorker` 仍在 shared memory 中；它 sleep 1s 后重发 PMSIGNAL。源码注释承认这里没有 retry 上限。

### 5.11 worker 接管 slot

`AutoVacWorkerMain()` 初始化 signal、`InitProcess()`、`BaseInit()` 和安全 GUC 后，持 `AutovacuumLock` exclusive 接管 starting slot：

```text
MyWorkerInfo = av_startingWorker
dbid = MyWorkerInfo->wi_dboid
MyWorkerInfo->wi_proc = MyProc
move WorkerInfo to av_runningWorkers
av_startingWorker = NULL
on_shmem_exit(FreeWorkerInfo, 0)
kill(launcherpid, SIGUSR2)
```

这一步完成 ownership 转移。之前 slot 属于 launcher / starting state，之后属于 worker。若 worker 发现 `av_startingWorker == NULL`，说明 launcher 可能已因 timeout 回收，它会 warning 并退出。

### 5.12 worker 连接 database

worker 在连接前先：

```text
pgstat_report_autovac(dbid)
```

它更新 database stats 的 `last_autovac_time`。故意放在 `InitPostgres(dbid)` 之前，是为了避免刚被删除或无法打开的 database 被 launcher 反复立即选中。

随后：

```text
InitPostgres(NULL, dbid, NULL, InvalidOid, INIT_PG_OVERRIDE_ALLOW_CONNS, dbname)
SetProcessingMode(NormalProcessing)
set_ps_display(dbname)
recentXid = ReadNextTransactionId()
recentMulti = ReadNextMultiXactId()
do_autovacuum()
```

`INIT_PG_OVERRIDE_ALLOW_CONNS` 允许 autovacuum 维护 `datallowconn=false` 的 database。`do_autovacuum()` 内会根据 `datistemplate` 或 `!datallowconn` 把默认 freeze ages 设为 0。

### 5.13 `do_autovacuum()` 扫描 relation

`do_autovacuum()` 创建 worker 级 `AutovacMemCxt`，开启事务，读取当前 `pg_database` tuple，打开 `pg_class`，复制 tuple descriptor，并建立 TOAST 映射 hash。

`pg_class` 扫描分两遍：

```text
第一遍:
  处理 RELKIND_RELATION 和 RELKIND_MATVIEW；
  跳过其它 backend 的 temp table；
  记录 orphan temp table；
  读取 reloptions；
  调 relation_needs_vacanalyze()；
  把需要 vacuum/analyze 的 OID + score 放进 tables_to_process；
  记录 main relation 到 toast relation 的 reloptions fallback。

第二遍:
  处理 RELKIND_TOASTVALUE；
  使用 toast 自身 reloptions，或 main table reloptions fallback；
  调 relation_needs_vacanalyze()；
  只保留 dovacuum，忽略 analyze。
```

TOAST 单独评估是必要的：短宽表可能在 TOAST table 上产生比 main table 更高比例的维护需求，不能简单把 toast 当作 main table vacuum 的附带动作。

### 5.14 `relation_needs_vacanalyze()`

这个函数同时返回是否需要 vacuum/analyze 和排序 score。输入来自三类状态：

```text
pg_class:
  reltuples、relpages、relallfrozen、relfrozenxid、relminmxid、relkind。

reloptions / GUC:
  thresholds、scale factors、freeze ages、cost 参数、autovacuum_enabled。

pgstat relation entry:
  dead_tuples、ins_since_vacuum、mod_since_analyze。
```

它先计算 wraparound force：`relfrozenxid` 或 `relminmxid` 是否超过对应 force limit。force vacuum 会设置 `wraparound` 和 `dovacuum`，即使普通 autovacuum 被关闭也不能跳过。

普通阈值公式在第 40 节展开，本节只记住三类比例：

```text
vac score:
  dead_tuples / vacuum threshold

vac_ins score:
  ins_since_vacuum / insert vacuum threshold

anl score:
  mod_since_analyze / analyze threshold
```

另外还有：

```text
xid score:
  xid age / freeze_max_age

mxid score:
  multixact age / multixact_freeze_max_age
```

每个 score 会乘以对应 weight，`scores.max` 是最大组件。当前分支还提供 `pg_stat_get_autovacuum_scores()`，用于查看当前 database 中相关表的 autovacuum score。

没有 relation pgstat entry 时，普通 threshold 逻辑不能继续；但 wraparound force 在此之前已经判断。这是 stats 近似与 correctness override 的边界。

### 5.15 relation 排序与逃生阀

候选收集完后：

```text
if any autovacuum_*_score_weight != 0
    list_sort(tables_to_process, TableToProcessComparator)
```

`TableToProcessComparator` 按 `score` 降序。若所有 score weight 都设为 0，源码跳过排序，注释称这是恢复旧行为的 escape hatch。

所以“下一个维护目标”不是一个全局答案，而是两段选择：

```text
launcher:
  wraparound database first；否则 oldest database last_autovac_time；
  受 DatabaseList naptime schedule 抑制。

worker:
  selected database 内按 scores.max 处理 relation；
  执行前仍需 claim 和 recheck。
```

### 5.16 table claim 防重复

worker 处理每个 `TableToProcess` 前先 claim：

```text
SearchSysCache1(RELOID) 判断 relisshared
LWLockAcquire(AutovacuumScheduleLock, LW_EXCLUSIVE)
LWLockAcquire(AutovacuumLock, LW_SHARED)
scan av_runningWorkers
  skip myself
  skip workers in other database unless target is shared
  if worker->wi_tableoid == relid: skip
LWLockRelease(AutovacuumLock)
if not skip:
  MyWorkerInfo->wi_tableoid = relid
  MyWorkerInfo->wi_sharedrel = isshared
LWLockRelease(AutovacuumScheduleLock)
```

`AutovacuumLock` 稳定 running worker list，`AutovacuumScheduleLock` 串行化“检查别人 claim + 写入自己 claim”的窗口。没有后者，两个 worker 可能同时扫描不到对方，再同时写同一个 `relid`。

shared relation 要跨 database 检查。非 shared relation 只需要避免同一 database 内重复；shared catalog 可能被不同 database 的 worker 同时看到。

### 5.17 claim 后 recheck

claim 后调用 `table_recheck_autovac()`。它重新读取 `pg_class` tuple、reloptions、TOAST fallback reloptions，并再次调用 `relation_needs_vacanalyze()`。

如果返回 `NULL`：

```text
clear wi_tableoid / wi_sharedrel under AutovacuumScheduleLock
continue
```

这一步承认候选列表只是估算。第一遍 scan 后，另一个 worker、手动 VACUUM、DDL、stats flush 或 reloptions 变更都可能让该表不再需要维护。

### 5.18 cost balance 状态

`table_recheck_autovac()` 设置 `at_storage_param_vac_cost_delay`、`at_storage_param_vac_cost_limit` 和 `at_dobalance`。没有 per-table cost override 的表会参与全局 cost limit 平衡。

当前源码里 `wi_dobalance` 是 `pg_atomic_flag`。要按 atomic flag 语义读：

```text
pg_atomic_test_set_flag(&wi_dobalance):
  flag set；pg_atomic_unlocked_test_flag() 返回 false；参与 balance。

pg_atomic_clear_flag(&wi_dobalance):
  flag clear；pg_atomic_unlocked_test_flag() 返回 true；不参与 balance。
```

`autovac_recalculate_workers_for_balance()` 扫 `av_runningWorkers`，只统计已接管且 flag 处于 set 状态的 worker。`AutoVacuumUpdateCostLimit()` 在参与 balance 时把 `vacuum_cost_limit` 除以 `av_nworkersForBalance`，至少为 1。

这也是一个典型例子：字段名不是语义，必须结合 atomic API、分支和生命周期读。

### 5.19 执行 vacuum/analyze

worker 每张表前 reset `PortalContext`，保存 relation/schema/database name，用 per-table `PG_TRY()` 包围：

```text
autovacuum_do_vac_analyze(tab, bstrategy)
```

`autovacuum_do_vac_analyze()` 更新 activity string，创建 `Vacuum` memory context，构造 `VacuumRelation`，再调用：

```text
vacuum(rel_list, &tab->at_params, bstrategy, vac_context, true)
```

进到这里后，真正的 lock、visibility、heap scan、index cleanup、WAL 和 progress 更新都属于 VACUUM / ANALYZE 执行层。

### 5.20 per-table ERROR 与继续

如果某表失败，`PG_CATCH()` 会：

```text
记录 automatic vacuum/analyze errcontext；
EmitErrorReport()；
AbortOutOfAnyTransaction()；
FlushErrorState()；
MemoryContextReset(PortalContext)；
StartTransactionCommand()；
继续下一个候选。
```

单表损坏、锁冲突、表达式错误或取消，不应该阻止同 database 其它表维护。顶层 worker ERROR 则不同：worker 直接退出，由 launcher 后续再启动新 worker。

### 5.21 worker 收尾

每张表结束后清理 claim：

```text
LWLockAcquire(AutovacuumScheduleLock, LW_EXCLUSIVE)
wi_tableoid = InvalidOid
wi_sharedrel = false
LWLockRelease(AutovacuumScheduleLock)
```

所有候选处理完后，worker 处理 `av_workItems` 中属于当前 database 的任务。最后：

```text
if (did_vacuum || !found_concurrent_worker)
    vac_update_datfrozenxid();
CommitTransactionCommand();
proc_exit(0);
```

`vac_update_datfrozenxid()` 即使没有 vacuum 某表也可能重要，因为 relation drop 或其它变化可能允许 database frozen xid 和 `xidVacLimit` 前进。但如果本 worker 没做工作且看到过其它 worker claim，源码避免盲目更新，防止特殊配置下的重启循环。

### 5.22 `FreeWorkerInfo()`

worker 通过 `on_shmem_exit(FreeWorkerInfo, 0)` 注册 cleanup。退出时：

```text
LWLockAcquire(AutovacuumLock, LW_EXCLUSIVE)
remove from av_runningWorkers
clear wi_dboid / wi_tableoid / wi_sharedrel / wi_proc / wi_launchtime
pg_atomic_clear_flag(&wi_dobalance)
push to av_freeWorkers
MyWorkerInfo = NULL
av_signal[AutoVacRebalance] = true
LWLockRelease(AutovacuumLock)
```

postmaster 在 autovacuum worker exit 后也会 signal launcher。下一轮 launcher 看到 SIGUSR2，可能 rebalance cost，并发现 worker slot 可用。

完整状态流：

```text
av_freeWorkers
  -> av_startingWorker
  -> av_runningWorkers
  -> wi_tableoid claim / clear repeated
  -> work items
  -> vac_update_datfrozenxid
  -> FreeWorkerInfo back to av_freeWorkers
```

## 6. 生命周期 / ownership / cleanup

`DatabaseList` 由 launcher 创建和持有，只存在于 launcher-local memory。`rebuild_database_list()` 删除旧 `DatabaseListCxt`，ERROR recovery 会 `MemoryContextReset(AutovacMemCxt)`、置空 `DatabaseListCxt` 并重新初始化 `DatabaseList`。所以它不能被当作 shared state。

`WorkerInfoData` 由 `AutoVacuumShmemInit()` 创建，数量等于 `autovacuum_worker_slots`。launcher 从 free list 取出后拥有 starting ownership；worker 接管后拥有 running ownership；退出时 `FreeWorkerInfo()` 归还。`on_shmem_exit` 是 slot 不泄漏的关键。

table claim 由 worker 创建：持 `AutovacuumScheduleLock` 写 `wi_tableoid` / `wi_sharedrel`。其它 worker 在 claim 前读取它。正常每张表结束后清除；per-table ERROR 后走后续 cleanup；worker 顶层 ERROR 或退出由 `FreeWorkerInfo()` 清空。

launcher 的 transaction 生命周期很短：`get_database_list()` 内 `StartTransactionCommand()`，scan `pg_database`，`CommitTransactionCommand()`。结果分配在 caller context，避免 transaction end 后丢失。

worker 的候选列表必须跨 transaction 存活，所以 `AutovacMemCxt` 保存 `tables_to_process`、TOAST map 等长期数据；每张表使用 `PortalContext` 和 `Vacuum` context 清理 per-table allocations。per-table ERROR 后 `AbortOutOfAnyTransaction()` 并重新 `StartTransactionCommand()`。

pgstat ownership 分两层：`pgstat_report_autovac(dbid)` 更新 database-level `last_autovac_time`；`pgstat_report_vacuum()` / `pgstat_report_analyze()` 更新 relation-level `last_autovacuum`、`autovacuum_count`、`last_autoanalyze`、`autoanalyze_count`、live/dead tuple 和相关 counters。调度读取 pgstat，但不把 pgstat 当锁。

postmaster 持有 process ownership。launcher 只写 shared memory intent 并发 PMSIGNAL；postmaster 负责 worker fork、failure notification 和 exit notification。这保持了进程监督边界。

## 7. 正确性机制层次

autovacuum 调度正确性不是单靠 MVCC visibility。它由多层机制组合：

| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `AutovacuumLock` | worker slot、running list、starting pointer、work item 的 shared state 一致性。 | 某张表一定需要 vacuum。 |
| `AutovacuumScheduleLock` | table claim 的检查和写入原子性。 | relation-level heavyweight lock。 |
| latch / SIGUSR2 | launcher 能被 signal、worker exit、fork failure 等事件及时唤醒。 | 消息队列或事件计数。 |
| postmaster PMSIGNAL | worker 启动由监督者完成。 | worker 一定 fork 成功。 |
| catalog scan / syscache | 在当前 transaction 中解释 `pg_database` / `pg_class`。 | 候选在执行时仍存在。 |
| pgstat | 提供调度输入和完成记录。 | 强一致任务队列。 |
| VACUUM visibility | 真正判断 tuple 是否可回收。 | 调度公平性。 |

`AutovacuumLock` 的临界区很短：pop/push worker、读写 whereabouts、设置 flags、claim work item、重算 balance。launcher 不持锁 scan `pg_database`，worker 不持锁跑 VACUUM。

`AutovacuumScheduleLock` 的核心不变量是：

```text
scan other workers' wi_tableoid
  and
write my wi_tableoid
must be atomic with respect to other claimers.
```

stats stale 是设计接受的事实。系统用 claim + recheck + VACUUM execution locks 来收敛，而不是让 pgstat 变成同步队列。

anti-wraparound 是 correctness override。普通 autovacuum 可以受 `autovacuum=off`、reloption、stats 缺失、worker slot、cost delay 影响；XID/MultiXact wraparound 不能简单跳过。

## 8. 错误路径 / 异常路径 / fallback

launcher 顶层 ERROR recovery 会报告错误、abort transaction、释放 LWLock、清等待状态、reset `AutovacMemCxt`、清 `DatabaseList`，sleep 1s 后继续。它的目标是长期进程不因一次 catalog 或 shared state ERROR 死掉。

worker 顶层 ERROR recovery 不尝试继续。源码注释明确说 autovacuum worker ERROR 后直接 `proc_exit(0)`，由 launcher 之后再启动新 worker。`InitProcess()` 和 `on_shmem_exit(FreeWorkerInfo)` 负责清理 PGPROC 和 worker slot。

per-table ERROR 是局部捕获。`do_autovacuum()` 在每张表的 `autovacuum_do_vac_analyze()` 外包 `PG_TRY()`，失败后 abort 当前 transaction、reset per-table context、重启 transaction 并继续下一张表。

starting worker 超时：launcher 最多等 `min(autovacuum_naptime, 60s)`。如果 `av_startingWorker` 仍未被接管，它清字段、放回 `av_freeWorkers`、清 `av_startingWorker` 并 warning。这只覆盖 worker 接管前的异常。

fork failure retry：postmaster 调 `AutoVacWorkerFailed()` 设置 `AutoVacForkFailed`，launcher 收到 SIGUSR2 后 sleep 1s 并重发 `PMSIGNAL_START_AUTOVAC_WORKER`。源码没有 retry 上限；持续 fork failure 是操作层面问题。

database stale：被选中的 database 可能刚被删除。worker 在连接前先 `pgstat_report_autovac(dbid)`，然后 `InitPostgres()` 失败退出。这样可以避免 launcher 立即反复选中同一个坏 database。

relation stale：worker 执行前会 `SearchSysCache1(RELOID)`、claim、`table_recheck_autovac()`、name lookup。任何一步发现 relation 不存在或不再需要维护，就跳过并释放 claim。

orphan temp table fallback：worker 先在 scan 中记录候选，再用 conditional relation lock、namespace lock 和 syscache recheck 确认，最后 `performDeletion()`。这是“先发现候选，destructive 操作前重新加锁 recheck”的同一模式。

config reload：launcher SIGHUP 后可 shutdown、warning、rebuild database list；worker 每张表前处理 SIGHUP，但不能因为 `autovacuum` 被关就退出，因为当前 worker 可能是 wraparound emergency worker。

## 9. 成本、资源与跨模块传播

launcher database scan 成本随 database 数增长，`rebuild_database_list()` 还要 hash、copy array、qsort，近似 `O(N log N)`。这仍远小于跨 database scan 所有 relation。

worker relation scan 成本随 `pg_class` tuple、relation 数、TOAST 表、partition/matview 数、reloptions 和 pgstat lookup 增长。autovacuum 不是只检查最近修改过的表；它通过 pgstat counters 快速判断，但仍要扫描 catalog 候选空间。

worker slot 是全局资源。`autovacuum_worker_slots` 决定 shared memory array 和 postmaster child pool 容量；`autovacuum_max_workers` 决定 launcher 使用多少。max 太小会推迟维护，max 太大会放大 I/O、WAL、buffer churn、lock conflict 和 CPU。

`av_startingWorker` 串行化 worker 启动阶段。正常 fork 很快，这不是瓶颈；在系统 fork 慢、slot 紧张或 worker early init 卡住时，它会让 launcher 即使还有 free slots 也暂缓新启动。

table claim contention 会出现在多个 worker 同时进入候选表处理时。可观测 wait event 是 `LWLock: Autovacuum` 和 `LWLock: AutovacuumSchedule`。这些只说明 shared state 上等待，不直接说明具体表。

pgstat stale 会造成无效工作：候选被选中后 recheck 发现不需要、database last_autovac_time 与真实完成时间不一致、dead tuple 估算与真实清理量不一致。PostgreSQL 接受这些近似，因为严格准确会引入更高 contention。

cost limit 平衡把全局维护吞吐传播到每个 worker。`autovac_recalculate_workers_for_balance()` 统计参与 balance 的 workers，`AutoVacuumUpdateCostLimit()` 把 `vacuum_cost_limit` 分摊。表级 cost reloptions 会让某个 worker 走不同路径，因此诊断 I/O 不能只看实例 GUC。

VACUUM 执行层会继续传播到 shared buffers、WAL、checkpoint、lock manager、ProcArray xmin horizon、replication slot 等模块。调度层不决定 tuple 是否可回收，但决定这些成本何时并发发生。

shared catalog 是跨 database 的特殊目标。`wi_sharedrel` 让不同 database 的 worker 也能互相看见 shared relation claim，避免重复维护同一 shared catalog。

parallel vacuum 不是 autovacuum worker。`autovacuum_parallel_workers` reloption 影响一次 `vacuum()` 内部 parallel vacuum worker；`autovacuum_max_workers` 限制的是 autovacuum worker process。两个 worker 概念不要混淆。

## 10. 观测与诊断入口

看 launcher 和 worker：

```sql
SELECT pid, backend_type, datname, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type IN ('autovacuum launcher', 'autovacuum worker')
ORDER BY backend_type, pid;
```

常见 launcher wait：

```text
wait_event_type = Activity
wait_event = AutovacuumMain
```

看 VACUUM progress：

```sql
SELECT pid, datname, relid::regclass, phase,
       heap_blks_total, heap_blks_scanned, heap_blks_vacuumed,
       index_vacuum_count, num_dead_item_ids, started_by
FROM pg_stat_progress_vacuum;
```

看 relation 结果：

```sql
SELECT relid::regclass, n_dead_tup, n_mod_since_analyze,
       last_autovacuum, last_autoanalyze,
       autovacuum_count, autoanalyze_count
FROM pg_stat_all_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

看当前分支提供的 score：

```sql
SELECT *
FROM pg_stat_autovacuum_scores
ORDER BY score DESC
LIMIT 20;
```

这个 view 底层调用 `pg_stat_get_autovacuum_scores()`。如果旧版本没有该 view，用 `\df+ pg_stat_get_autovacuum_scores` 或系统 catalog 查看函数定义。

打开日志：

```sql
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
SELECT pg_reload_conf();
```

日志能看到已经执行的 autovacuum 动作、耗时、扫描页、dead tuple、index cleanup 等。日志看不到 `DatabaseList`、`av_startingWorker`、free worker list、`tables_to_process` 或 recheck skip 原因。

wait event 粒度要分清：`Activity / AutovacuumMain` 是 launcher 主循环等待；`LWLock / Autovacuum` 是等 autovacuum worker shared state；`LWLock / AutovacuumSchedule` 是等 table claim 调度锁；其它 IO、Lock、BufferPin、WAL wait 多数已经进入 VACUUM 执行层。

只能推断或断点看的状态包括：`DatabaseList` 每个 `adl_next_worker`、`av_freeWorkers` 精确链表、`av_startingWorker` 指向哪个 database、worker 内存中的 `tables_to_process`、`table_recheck_autovac()` skip 原因、`av_nworkersForBalance` 当前值。

适合 gdb / DEBUG 日志的断点：`AutoVacLauncherMain()`、`launcher_determine_sleep()`、`rebuild_database_list()`、`do_start_worker()`、`AutoVacWorkerMain()`、`do_autovacuum()`、`relation_needs_vacanalyze()`、`table_recheck_autovac()`、`autovac_recalculate_workers_for_balance()`、`FreeWorkerInfo()`。

诊断时要区分三类状态：能直接观测的是 `backend_type`、wait event、progress、relation stats 和 logs；能近似推断的是 database schedule、candidate ordering、stale stats 和 cost balance；几乎不可见的是 launcher-local `DatabaseList` 的瞬时内容和 per-worker candidate list。

## 11. 常见误区

误区一：把 `autovacuum_naptime` 理解成每张表定时器。实际它主要参与 database schedule；worker 被派到 database 后才扫描 relation。

误区二：把 `last_autovacuum` 当成 database 调度时间。relation-level `last_autovacuum` 表示某表完成 autovacuum；database-level `last_autovac_time` 在 worker 尝试处理 database 时更新，甚至早于 `InitPostgres()`。

误区三：把 pgstat 当任务队列。pgstat 是近似统计输入，不提供“一定执行”“立即执行”“全局最优排序”的语义。

误区四：只调 `autovacuum_max_workers`，忽略 `autovacuum_worker_slots`。当前源码中 slots 影响 shared memory 和 postmaster child pool，实际并发不可能超过 slots。

误区五：把 `wi_tableoid` 当 relation lock。它只是 autovacuum worker 间的调度 claim；真正 relation lock 在 VACUUM 执行层。

误区六：忽略 shared relation 跨 database claim。shared catalog 不是 database-local，`wi_sharedrel` 会让其它 database 的 worker 也参与冲突判断。

误区七：看到 `AutovacuumMain` 就判断 autovacuum 卡住。它通常只是 launcher sleep；要结合 worker 数、logs、progress、pgstat 和 wait event。

误区八：认为 `autovacuum=off` 完全禁止 worker。anti-wraparound worker 仍可能启动，这是 correctness override。

## 12. 课堂实验

### 实验 1：worker 上限与 launcher wait

目的：把 `autovacuum_max_workers` / `autovacuum_worker_slots` 与 `av_worker_available()` 对上。

测试库配置：

```sql
ALTER SYSTEM SET autovacuum = on;
ALTER SYSTEM SET autovacuum_naptime = '5s';
ALTER SYSTEM SET autovacuum_max_workers = 1;
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
SELECT pg_reload_conf();
```

造两张表：

```sql
CREATE TABLE av_sched_a(id int, payload text);
CREATE TABLE av_sched_b(id int, payload text);
INSERT INTO av_sched_a SELECT g, repeat('a', 200) FROM generate_series(1, 200000) g;
INSERT INTO av_sched_b SELECT g, repeat('b', 200) FROM generate_series(1, 200000) g;
DELETE FROM av_sched_a WHERE id % 2 = 0;
DELETE FROM av_sched_b WHERE id % 2 = 0;
```

观察：

```sql
SELECT pid, backend_type, datname, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type IN ('autovacuum launcher', 'autovacuum worker')
ORDER BY backend_type, pid;
```

预期：launcher 多数时间在 `AutovacuumMain`，worker 同时最多 1 个，server log 记录 autovacuum action。源码回扣：`AutoVacLauncherMain()` -> `av_worker_available()` -> `launch_worker()`。

### 实验 2：relation score 与 recheck

目的：观察 worker 在 database 内如何按 score 处理 relation。

降低 threshold：

```sql
ALTER SYSTEM SET autovacuum_vacuum_threshold = 10;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.01;
ALTER SYSTEM SET autovacuum_analyze_threshold = 10;
ALTER SYSTEM SET autovacuum_analyze_scale_factor = 0.01;
SELECT pg_reload_conf();
```

造三张表：

```sql
CREATE TABLE av_score_hot(id int, payload text);
CREATE TABLE av_score_warm(id int, payload text);
CREATE TABLE av_score_anl(id int, payload text);
INSERT INTO av_score_hot SELECT g, repeat('h', 100) FROM generate_series(1, 200000) g;
INSERT INTO av_score_warm SELECT g, repeat('w', 100) FROM generate_series(1, 50000) g;
INSERT INTO av_score_anl SELECT g, repeat('x', 100) FROM generate_series(1, 50000) g;
DELETE FROM av_score_hot WHERE id <= 100000;
DELETE FROM av_score_warm WHERE id <= 5000;
UPDATE av_score_anl SET payload = payload || 'y' WHERE id <= 20000;
```

查看：

```sql
SELECT * FROM pg_stat_autovacuum_scores
ORDER BY score DESC
LIMIT 10;
```

如果手动 `VACUUM av_score_hot;` 抢先处理，autovacuum worker 之后可能在 `table_recheck_autovac()` 跳过它。源码回扣：`relation_needs_vacanalyze()` -> `TableToProcessComparator` -> `table_recheck_autovac()`。

### 实验 3：table claim

目的：理解 `AutovacuumScheduleLock` 和 `wi_tableoid` 防重复。

配置：

```sql
ALTER SYSTEM SET autovacuum_max_workers = 3;
ALTER SYSTEM SET autovacuum_naptime = '3s';
SELECT pg_reload_conf();
```

制造多张大表的 dead tuples 后观察：

```sql
SELECT pid, datname, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type = 'autovacuum worker'
ORDER BY pid;
```

源码练习：在 `do_autovacuum()` claim table 前后临时加 DEBUG 日志，打印 `MyDatabaseId`、`relid`、`MyWorkerInfo->wi_tableoid`、`skipit`。预期同一个 `relid` 不会被两个 worker 同时 claim。不要把该日志 patch 带入产品分支。

## 13. 讨论题

1. 为什么 launcher 不维护全局 relation priority queue？
2. `DatabaseList` 为什么是 launcher-local，而不是 shared memory？
3. `pgstat_report_autovac(dbid)` 为什么在 `InitPostgres(dbid)` 之前执行？
4. `autovacuum_worker_slots` 与 `autovacuum_max_workers` 分别限制什么？
5. `AutovacuumLock` 和 `AutovacuumScheduleLock` 的分工是什么？
6. 如果删除 `table_recheck_autovac()`，哪些 stale stats 或 DDL race 会暴露？
7. `scores.max` 使用最大组件而不是求和，会如何影响 vacuum/analyze/freeze 的优先级？
8. `pg_stat_activity` 能看到哪些 autovacuum 状态，哪些不能？
9. 为什么 per-table ERROR 后 worker 可以继续，而 top-level worker ERROR 直接退出？
10. per-table vacuum cost reloptions 如何改变全局 cost balance？

## 14. 本节小结

本节主链路：

```text
postmaster starts launcher
  -> launcher builds DatabaseList
  -> WaitLatch AutovacuumMain
  -> av_worker_available()
  -> do_start_worker() scans pg_database and database pgstat
  -> av_startingWorker + PMSIGNAL_START_AUTOVAC_WORKER
  -> worker takeover
  -> do_autovacuum() scans pg_class and relation pgstat
  -> relation_needs_vacanalyze() builds scores
  -> sort tables_to_process
  -> AutovacuumScheduleLock claim
  -> table_recheck_autovac()
  -> vacuum()/analyze()
  -> clear claim
  -> vac_update_datfrozenxid()
  -> FreeWorkerInfo()
```

核心状态：

```text
DatabaseList:
  launcher-local schedule cache。

AutoVacuumShmem:
  worker lifecycle and shared signals。

WorkerInfoData:
  database assignment, running state, current table claim。

pgstat:
  approximate scheduling input and completed-work reporting。

VacuumParams:
  execution-layer maintenance contract。
```

异常收尾：

```text
launcher ERROR:
  reset memory and DatabaseList，继续。

worker top-level ERROR:
  proc_exit，FreeWorkerInfo 归还 slot。

per-table ERROR:
  abort transaction，reset PortalContext，继续下一张表。

fork failure:
  shared flag + SIGUSR2 + retry PMSIGNAL。

starting timeout:
  launcher 回收 starting slot。
```

观测边界：SQL 能看到 launcher/worker、wait event、progress、relation stats、logs 和当前 score；看不到完整 `DatabaseList`、free worker list、候选表列表、recheck skip 原因和瞬时 cost balance，只能通过推断、日志或断点补齐。

可迁移规律：

```text
后台维护调度通常不会建立强一致全局最优队列；
它会把选择拆成轻量 coarse scheduling 和局部精细 recheck，
用 shared state 只保存必要 ownership，
用短临界区 claim 防重复，
把 stale input 留给 recheck 和执行层处理。
```

对 autovacuum 来说，就是：

```text
database schedule is approximate；
relation score is local；
worker slots are global；
table claim is shared；
VACUUM correctness remains in VACUUM。
```

下一节可继续追第 40 节：insert/update/delete 计数、threshold、scale factor 和 reloptions 如何决定一张表是否触发 vacuum/analyze。
