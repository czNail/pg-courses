# PostgreSQL subscription metadata 与 logical replication launcher
## 课程定位
前置知识：已经理解 logical decoding、replication slot、replication origin、walsender/walreceiver，以及后台 worker 的基本生命周期。
本节唯一主问题：
```text
pg_subscription、pg_subscription_rel 和 launcher 如何决定启动哪些 apply worker / table sync worker，
为什么 subscription 是 catalog 状态而不是单纯连接配置？
```
核心矛盾：
```text
subscription 看起来像一组连接参数
  vs
订阅端真正需要跨数据库发现、跨进程调度、跨崩溃恢复、DDL invalidation、slot/origin cleanup 和 relation 级同步状态
```
学完后应能判断：一个 enabled subscription 为什么没有 apply worker；一张表为什么还在 table sync；哪些字段变化会让 worker 原地更新、退出重启或停止；`pg_stat_subscription` 能看到什么，必须回到 catalog、日志或源码推断什么。
本课基于本地 `/home/nail/postgres`，分支 `master`，commit `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面课程关注 publisher 侧如何从 WAL 产生 logical change，以及 slot / origin 如何保存进度。
本节切到 subscriber 侧，追一条完整启动链：
```text
CREATE / ALTER SUBSCRIPTION
  -> 写 pg_subscription / pg_subscription_rel
  -> commit 后唤醒 logical replication launcher
  -> launcher 扫描 enabled subscriptions
  -> 启动缺失的 apply worker
  -> apply worker 读取完整 subscription 配置并连接 publisher
  -> apply worker 根据 pg_subscription_rel 启动和协调 table sync worker
  -> table sync 完成后 relation 进入 READY
```
这条链路中只有 apply worker 真正连接 publisher。
launcher 不负责解释 publication，不负责连接 slot，也不直接调度每张表的同步。
launcher 解决的是更前置的问题：
```text
在没有用户 session 的后台进程中，如何发现所有应该运行的 subscription？
```
apply worker 解决的是：
```text
一个 subscription 的完整连接、权限、stream 选项、slot 和 publication 如何变成一条 replication stream？
```
table sync worker 解决的是：
```text
一张还没初始同步完成的表，如何在 COPY snapshot 和后续 logical stream 之间接上？
```
所以本节不是 `CREATE SUBSCRIPTION` 语法课，而是 catalog truth、worker truth 和 relation sync truth 如何互相推进的源码课。
## 2. 核心矛盾与一句话运行模型
一句话模型：
```text
pg_subscription 保存订阅级持久意图；
pg_subscription_rel 保存 relation 级持久同步状态；
launcher 只根据 enabled subscription 维护 apply worker；
apply worker 读取完整 Subscription 并根据 pg_subscription_rel 维护 table sync worker；
worker shared memory 保存当前进程是否真的在跑。
```
三层状态不要混淆：
| 层次 | 位置 | 代表什么 |
| --- | --- | --- |
| catalog truth | `pg_subscription` / `pg_subscription_rel` | 订阅和 relation 同步状态的持久事实。 |
| cache truth | `MySubscription`、`MySubscriptionValid`、`table_states_not_ready` | worker 当前解释出的运行视图。 |
| process truth | `LogicalRepCtx->workers[]` | 当前 worker slot 是否占用、是否 attach、是哪种 worker。 |
一个字段单独不是语义。
例如 `subenabled = true` 只说明“应该维护 apply worker”。
它不保证当前一定有 pid。
还要看 worker 槽位、bgworker 槽位、重启节流、连接是否成功，以及 worker 是否刚启动就退出。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_subscription.h` | `pg_subscription` 字段、shared catalog 设计注释、`Subscription` 内存对象。 |
| 2 | `src/include/catalog/pg_subscription_rel.h` | relation 同步状态、`SubscriptionRelState`、catalog 状态和 IPC 状态。 |
| 3 | `src/backend/catalog/pg_subscription.c` | `GetSubscription()`、`AddSubscriptionRelState()`、`UpdateSubscriptionRelState()`、`GetSubscriptionRelations()`。 |
| 4 | `src/backend/commands/subscriptioncmds.c` | `CREATE/ALTER/DROP SUBSCRIPTION` 如何写 catalog、refresh relation mapping、唤醒 launcher。 |
| 5 | `src/backend/replication/logical/launcher.c` | `ApplyLauncherRegister()`、`ApplyLauncherMain()`、`logicalrep_worker_launch()`、worker 槽位和 `pg_stat_get_subscription()`。 |
| 6 | `src/backend/replication/logical/worker.c` | apply worker 初始化、SubscriptionCache、`maybe_reread_subscription()`、apply loop。 |
| 7 | `src/backend/replication/logical/syncutils.c` | `FetchRelationStates()`、`ProcessSyncingRelations()`、`launch_sync_worker()`。 |
| 8 | `src/backend/replication/logical/tablesync.c` | table sync COPY、catchup、状态机、slot/origin cleanup。 |
| 9 | `src/backend/postmaster/bgworker.c` | internal background worker entrypoint 映射。 |
| 10 | `src/backend/catalog/system_views.sql` | `pg_stat_subscription` 如何把 catalog 和 worker shared memory 左连接。 |
推荐阅读时先从两个 header 的状态定义开始，再读 launcher 主循环，最后沿一张表的 `INIT -> READY` 走完。
如果先读 `CREATE SUBSCRIPTION` 的 option parsing，很容易把本机制误读成参数解析。
## 4. 为什么 `pg_subscription` 是 shared catalog
`pg_subscription.h` 顶部注释说明了核心设计：
```text
subscription 技术上属于某个 database；
但 replication launcher 需要访问所有 subscription 以启动 worker；
因此 pg_subscription 是 shared, nailed catalog。
```
这句话是本节答案的第一半。
如果 subscription 只是某个 database 里的普通配置，launcher 要发现所有订阅就必须逐库连接扫描，或者维护另一套全局索引。
PostgreSQL 直接把订阅根状态放进 shared catalog。
`subdbid` 仍然存在，因为 apply worker 真正运行时要连接目标 database：
```text
launcher:
  BackgroundWorkerInitializeConnection(NULL, NULL, 0)
  只访问 nailed pg_subscription
apply worker:
  BackgroundWorkerInitializeConnectionByOid(subdbid, subowner, 0)
  进入 subscription 所属 database
```
所以 `subdbid` 不是反证。
它恰好说明 discovery 和 execution 被拆成两层。
## 5. `pg_subscription` 字段如何参与启动
`pg_subscription` 的字段可以沿启动链理解，而不是背清单。
launcher 只需要少量字段：
```text
oid
subdbid
subname
subowner
subenabled
subretaindeadtuples
subretentionactive
```
这些字段足以回答：
```text
是否应该有 apply worker？
worker 应该连哪个 database？
worker 应该用哪个 owner 身份？
是否需要维护 conflict detection slot？
```
apply worker 启动后才需要完整字段：
```text
subconninfo / subserver
subslotname
subpublications
subbinary
substream
subtwophasestate
suborigin
subsynccommit
subwalrcvtimeout
subpasswordrequired
subrunasowner
subfailover
subdisableonerr
subskiplsn
```
`subconninfo` 或 `subserver` 决定如何连接 publisher。
`subslotname` 决定主 logical slot。
`subpublications` 进入 `WalRcvStreamOptions.publication_names`。
`substream` 决定 streamed transaction 是关闭、写临时文件，还是 parallel apply。
`subtwophasestate` 是 tri-state，而不是用户看到的简单 bool。
它有：
```text
DISABLED
PENDING
ENABLED
```
这是因为 two_phase 只有在 table sync 完成后才能安全打开。
否则 prepared transaction 可能被 apply worker 跳过，却无法由 table sync worker 正确补齐。
`subdisableonerr` 影响错误路径。
如果为 true，worker ERROR 后会在新事务里把 subscription disable。
这又说明 subscription 是持久运行状态，不只是连接配置。
## 6. `GetSubscription()` 与 SubscriptionCache
`GetSubscription()` 在 `pg_subscription.c` 中把 catalog tuple 转成 backend-local `Subscription`。
它会为对象创建独立 memory context：
```text
cxt = AllocSetContextCreate(CurrentMemoryContext, "subscription", ...)
sub->cxt = cxt
```
apply worker 初始化后把这个 context 挂到 `ApplyContext` 下。
如果 subscription 使用 foreign server，`GetSubscription()` 不直接取 `subconninfo`。
它会：
```text
GetForeignServer(subserver)
object_aclcheck(..., subowner, ACL_USAGE)
ForeignServerConnectionString(subowner, server)
```
所以连接信息可能受 foreign server、user mapping、FDW connection function 和权限影响。
当前 commit 没有一个字面叫 `SubscriptionCache` 的结构体。
本课把下面这一组 worker-local 机制称为 SubscriptionCache 层：
```text
MySubscription
MySubscriptionValid
subscription_change_cb()
maybe_reread_subscription()
CacheRegisterSyscacheCallback(SUBSCRIPTIONOID, ...)
CacheRegisterSyscacheCallback(FOREIGNSERVEROID, ...)
CacheRegisterSyscacheCallback(USERMAPPINGOID, ...)
CacheRegisterSyscacheCallback(FOREIGNDATAWRAPPEROID, ...)
CacheRegisterSyscacheCallback(AUTHOID, ...)
```
它的语义是：
```text
catalog tuple:
  持久事实
Subscription object:
  worker 当前可执行配置
syscache invalidation:
  告诉 worker 这个配置可能过期
maybe_reread_subscription():
  决定原地更新、退出重启或停止
```
这层缓存避免每条 logical message 都重新查 catalog，同时又保留 DDL 变更后的可见性边界。
## 7. `pg_subscription_rel` 保存 relation 级协议
`pg_subscription_rel` 很小：
```text
srsubid
srrelid
srsubstate
srsublsn
```
但它不是展示进度用的附属表。
它直接决定 apply worker 是否可以对某张表应用变更。
catalog 中可存储的状态是：
```text
SUBREL_STATE_INIT
SUBREL_STATE_DATASYNC
SUBREL_STATE_FINISHEDCOPY
SUBREL_STATE_SYNCDONE
SUBREL_STATE_READY
```
源码中还有两个只用于 IPC 的状态：
```text
SUBREL_STATE_SYNCWAIT
SUBREL_STATE_CATCHUP
```
这两个不会写入 catalog。
原因是它们只描述当前 apply worker 与当前 table sync worker 之间的握手。
崩溃后不应该恢复到“正在等待某个已经不存在的 worker”的状态。
`srsublsn` 也不能单独解释。
它只有结合 `srsubstate` 和 apply stream 当前 LSN 才有意义。
例如 `SYNCDONE` 里的 LSN 表示 table sync worker 已经追到的位置。
apply worker 只有自己也追到这个 LSN 后，才能把该 relation 改成 `READY`。
## 8. DDL 如何写入持久状态
`CreateSubscription()` 在 `subscriptioncmds.c` 中写入 `pg_subscription`。
它设置：
```text
subdbid = MyDatabaseId
subenabled = opts.enabled
subowner = owner
subconninfo 或 subserver
subslotname
subpublications
subsynccommit
subwalrcvtimeout
suborigin
subtwophasestate
```
如果需要初始数据同步，refresh 逻辑会写 `pg_subscription_rel`。
新增 relation 的初始状态取决于 `copy_data`：
```text
copy_data = true:
  AddSubscriptionRelState(..., SUBREL_STATE_INIT, InvalidXLogRecPtr)
copy_data = false:
  AddSubscriptionRelState(..., SUBREL_STATE_READY, InvalidXLogRecPtr)
```
这说明 relation membership 不是 apply worker 临时发现的。
它是 DDL 产生的持久状态。
DDL 结束时不直接启动 worker。
它调用：
```text
ApplyLauncherWakeupAtCommit()
```
真正唤醒在事务提交后发生：
```text
AtEOXact_ApplyLauncher(isCommit)
  -> ApplyLauncherWakeup()
```
这样 launcher 只会看到已提交 catalog tuple。
这是 worker 启动链的 visibility 边界。
## 9. `ALTER SUBSCRIPTION` 改变运行协议
`ALTER SUBSCRIPTION ... ENABLE` 更新 `subenabled`，并在 commit 后唤醒 launcher。
`ALTER SUBSCRIPTION ... DISABLE` 更新 catalog 后，已运行 worker 会通过 invalidation 重读并退出。
因此 enabled/disabled 分两层：
```text
launcher:
  enabled 为 true 才启动缺失 apply worker
worker:
  运行中发现 enabled 为 false 后退出
```
`ALTER SUBSCRIPTION ... SET PUBLICATION ... refresh` 会重新连接 publisher，获取 publication relation list，再更新 `pg_subscription_rel`。
新增 relation 会写 `INIT` 或 `READY`。
移除 relation 时要处理：
```text
停止对应 table sync worker
删除 pg_subscription_rel entry
清理 tablesync origin
必要时 drop tablesync slot
```
这也是 catalog 状态必要的原因。
DDL、worker、publisher slot 和 replication origin cleanup 必须围绕同一个持久事实协调。
## 10. `ApplyLauncherRegister()`：launcher 本身如何出现
postmaster 启动路径中调用 `ApplyLauncherRegister()`。
如果：
```text
max_logical_replication_workers == 0
IsBinaryUpgrade
```
则不注册 launcher。
否则它注册一个 background worker：
```text
bgw_library_name = "postgres"
bgw_function_name = "ApplyLauncherMain"
bgw_start_time = BgWorkerStart_RecoveryFinished
bgw_restart_time = 5
```
`bgworker.c` 的 `InternalBGWorkers[]` 包含这些 logical replication entrypoint：
```text
ApplyLauncherMain
ApplyWorkerMain
ParallelApplyWorkerMain
SequenceSyncWorkerMain
TableSyncWorkerMain
```
所以核心 logical replication worker 不通过 extension 动态符号查找。
launcher 是常驻 supervisor。
apply worker、table sync worker、sequence sync worker、parallel apply worker 则由运行时动态注册。
## 11. 主流程源码 walkthrough：`ApplyLauncherMain()` 主循环
launcher 启动后：
```text
BackgroundWorkerInitializeConnection(NULL, NULL, 0)
```
注释说它只访问 nailed `pg_subscription`。
随后主循环每轮创建临时 context：
```text
"Logical Replication Launcher sublist"
```
`get_subscription_list()` 在一个事务里扫描 `pg_subscription`。
它只填启动所需字段：
```text
oid
dbid
owner
enabled
name
retaindeadtuples
retentionactive
```
然后对每个 subscription：
```text
如果 retain_dead_tuples:
  维护 conflict detection slot 相关状态
如果 !enabled:
  不启动 apply worker
如果 enabled:
  在 LogicalRepWorkerLock 下查 WORKERTYPE_APPLY 是否存在
如果不存在:
  检查 wal_retrieve_retry_interval
  调用 logicalrep_worker_launch(WORKERTYPE_APPLY, ...)
```
launcher 不读 `subconninfo`。
不读 `subslotname`。
不读 `subpublications`。
这些字段属于 apply worker 的 stream setup，不属于 launcher 的 discovery loop。
这条分工是本节主问题的关键。
## 12. `LogicalRepWorker`：当前 worker truth
logical replication worker 的共享数组在 `LogicalRepCtx->workers[]`。
大小来自：
```text
max_logical_replication_workers
```
每个槽包含身份和运行状态：
```text
type
launch_time
in_use
generation
proc
dbid
userid
subid
relid
relstate
relstate_lsn
leader_pid
parallel_apply
last_lsn
reply_lsn
last_send_time
last_recv_time
reply_time
```
`in_use` 表示槽被分配。
`proc` 非空表示 worker 已 attach。
`generation` 区分同一个 slot 的不同生命周期。
这能处理启动竞态：
```text
in_use = true, proc = NULL:
  dynamic bgworker 已注册但还没 attach
in_use = true, proc != NULL:
  worker 已运行并 attach slot
in_use = false:
  slot 空闲
```
`pg_subscription` 说“应该运行”。
`LogicalRepWorker` 说“现在是否真的有进程在跑”。
`pg_stat_subscription` 正是把这两层放到一张视图里。
## 13. `logicalrep_worker_launch()` 启动协议
`logicalrep_worker_launch()` 是所有 logical replication worker 的共用启动入口。
它先在 `LogicalRepWorkerLock` 下找空槽，并检查限制：
```text
max_logical_replication_workers
max_sync_workers_per_subscription
max_parallel_apply_workers_per_subscription
```
对于 table sync worker，`relid` 必须有效。
对于 apply worker，`relid` 必须是 `InvalidOid`。
它写入槽位：
```text
worker->type = wtype
worker->launch_time = now
worker->in_use = true
worker->generation++
worker->proc = NULL
worker->dbid = dbid
worker->userid = userid
worker->subid = subid
worker->relid = relid
```
然后注册 dynamic background worker：
```text
WORKERTYPE_APPLY:
  ApplyWorkerMain
WORKERTYPE_TABLESYNC:
  TableSyncWorkerMain
WORKERTYPE_SEQUENCESYNC:
  SequenceSyncWorkerMain
WORKERTYPE_PARALLEL_APPLY:
  ParallelApplyWorkerMain
```
`bgw_restart_time = BGW_NEVER_RESTART`。
也就是说，单个 worker 崩溃后不是 bgworker infrastructure 自动重启。
apply worker 由 launcher 下轮扫描重启。
table sync worker 由 apply worker 下轮处理 relation states 时重启。
如果没有 logical replication worker 槽，会报：
```text
out of logical replication worker slots
hint max_logical_replication_workers
```
如果 background worker 槽不足，会报：
```text
out of background worker slots
hint max_worker_processes
```
这两个资源上限必须同时满足。
## 14. apply worker 启动后必须重新读 catalog
`ApplyWorkerMain()` 入口很短：
```text
logicalrep_worker_attach(worker_slot)
SetupApplyOrSyncWorker(worker_slot)
run_apply_worker()
```
`InitializeLogRepWorker()` 做关键初始化：
```text
session_replication_role = replica
BackgroundWorkerInitializeConnectionByOid(dbid, userid, 0)
search_path = ''
StartTransactionCommand()
LockSharedObject(SubscriptionRelationId, subid, AccessShareLock)
MySubscription = GetSubscription(subid, true, true)
```
launcher 扫描到 subscription 和 worker 真正 attach 之间可能发生 DROP 或 DISABLE。
所以 worker 必须重新验证 catalog。
如果 subscription 被删除：
```text
worker will not start because the subscription was removed during startup
proc_exit(0)
```
如果 subscription 被 disable：
```text
worker will not start because the subscription was disabled during startup
apply_worker_exit()
```
这就是 catalog truth 与 worker truth 之间的 race closure。
worker 不能只相信 launcher 传来的 `subid`、`dbid`、`owner`。
## 15. apply worker 如何使用 conninfo / slot / publications
`run_apply_worker()` 才开始处理真正的 replication stream。
它先设置 subscription 级 replication origin：
```text
origin name = pg_<suboid>
replorigin_by_name()
replorigin_create()
replorigin_session_setup()
origin_startpos = replorigin_session_get_progress(false)
```
然后连接 publisher：
```text
walrcv_connect(MySubscription->conninfo,
               true,
               true,
               must_use_password,
               MySubscription->name,
               &err)
```
接着填 `WalRcvStreamOptions`：
```text
options.logical = true
options.startpoint = origin_startpos
options.slotname = MySubscription->slotname
options.proto.logical.publication_names = MySubscription->publications
options.proto.logical.binary = MySubscription->binary
options.proto.logical.streaming_str = 根据 substream 和 publisher 版本决定
options.proto.logical.origin = MySubscription->origin
```
因此：
```text
slot / conninfo / publications:
  apply worker 建 stream 时使用
enabled / dbid / owner:
  launcher 和 worker bootstrap 使用
pg_subscription_rel:
  apply worker 后续决定 per-relation apply / sync 使用
```
这就是为什么 launcher 不需要读取所有字段。
把所有字段都塞进 launcher 只会扩大职责，而不能解决 relation sync。
## 16. `maybe_reread_subscription()` 的重读规则
worker 注册 syscache callback 后，subscription 相关变化会使：
```text
MySubscriptionValid = false
```
apply loop 在安全边界调用：
```text
AcceptInvalidationMessages()
maybe_reread_subscription()
```
如果 subscription 被删除，worker 退出。
如果 `enabled = false`，worker 退出。
如果这些字段变化，worker 退出，让 launcher 重新拉起：
```text
conninfo
name
slotname
binary
stream
passwordrequired
origin
owner
publications
```
这些变化影响 remote connection 或 stream 协议。
在已有 stream 中热切换不值得。
如果只是：
```text
synchronous_commit
wal_receiver_timeout
```
worker 可以原地执行：
```text
SetConfigOption("synchronous_commit", ...)
set_wal_receiver_timeout()
```
所以 SubscriptionCache 的重读结果有三类：
```text
停止:
  removed / disabled
重启:
  连接、slot、publication、owner、origin、stream 语义变化
原地更新:
  worker-local GUC 类参数
```
这比“cache invalidation 后简单 reload”更精细。
## 17. relation state cache 与 `syncutils.c`
apply worker 不会每条消息都查 `pg_subscription_rel`。
当前 commit 把同步调度共用逻辑放在 `syncutils.c`。
关键状态：
```text
relation_states_validity
table_states_not_ready
```
`FetchRelationStates()` 在缓存失效时：
```text
GetSubscriptionRelations(MySubscription->oid, true, true, true)
把非 READY table 复制到 CacheMemoryContext
记录是否存在非 READY sequence
必要时调用 HasSubscriptionTables()
```
失效 callback：
```text
CacheRegisterSyscacheCallback(SUBSCRIPTIONRELMAP,
                              InvalidateSyncingRelStates,
                              ...)
```
这层缓存的语义：
```text
pg_subscription_rel:
  持久 relation state
table_states_not_ready:
  当前 apply worker 的调度视图
invalidation:
  DDL 或 state update 后让视图重建
```
注意它是每个 worker 的本地视图。
不是新的持久状态。
## 18. table sync worker 由 apply worker 启动
table sync worker 不是 launcher 直接启动的。
主链路是：
```text
ApplyWorkerMain
  -> run_apply_worker()
     -> start_apply()
        -> LogicalRepApplyLoop()
           -> ProcessSyncingRelations(current_lsn)
              -> ProcessSyncingTablesForApply(current_lsn)
                 -> launch_sync_worker(WORKERTYPE_TABLESYNC, ...)
                    -> logicalrep_worker_launch(WORKERTYPE_TABLESYNC, ...)
```
`ProcessSyncingRelations()` 根据当前 worker 类型分派：
```text
WORKERTYPE_APPLY:
  ProcessSyncingTablesForApply()
  ProcessSequencesForSync()
WORKERTYPE_TABLESYNC:
  ProcessSyncingTablesForSync()
WORKERTYPE_PARALLEL_APPLY:
  skip, because parallel apply handles only READY tables
```
所以排查 table sync worker 没起来时，先问：
```text
apply worker 是否存在？
apply worker 是否在处理 invalidation？
pg_subscription_rel 是否有非 READY table？
sync worker limit 是否达到？
wal_retrieve_retry_interval 是否还没过？
```
只看 launcher 不够。
## 19. table sync 状态机
`tablesync.c` 文件头给出状态推进：
```text
INIT
  -> DATASYNC
  -> FINISHEDCOPY
  -> SYNCWAIT
  -> CATCHUP
  -> SYNCDONE
  -> READY
```
写入 catalog 的状态：
```text
INIT
DATASYNC
FINISHEDCOPY
SYNCDONE
READY
```
只在共享内存里的状态：
```text
SYNCWAIT
CATCHUP
```
table sync worker 入口：
```text
TableSyncWorkerMain
  -> SetupApplyOrSyncWorker()
  -> run_tablesync_worker()
     -> start_table_sync()
        -> LogicalRepSyncTableStart()
```
`LogicalRepSyncTableStart()` 先读：
```text
GetSubscriptionRelState(subid, relid, &relstate_lsn)
```
如果状态已经是 `SYNCDONE`、`READY` 或 `UNKNOWN`，它直接 `FinishSyncWorker()`。
如果是 `DATASYNC`，说明上次 COPY 没完成。
它会尝试 drop 可能存在的 tablesync slot，并重新 COPY。
如果是 `FINISHEDCOPY`，说明 COPY 已完成但 catchup 没完成。
它会恢复 origin progress，跳过 COPY，进入 catchup 之后的流程。
这就是 `FINISHEDCOPY` 写 catalog 的价值。
大表 COPY 成功后崩溃，不需要重新 COPY。
## 20. COPY 与 catchup 的衔接
table sync 不是简单把 publisher 表全量拉下来。
它需要保证 COPY 快照和后续 logical stream 接上。
因此它在 publisher 上开启：
```text
BEGIN READ ONLY ISOLATION LEVEL REPEATABLE READ
```
再创建 tablesync logical slot：
```text
walrcv_create_slot(..., CRS_USE_SNAPSHOT, origin_startpos)
```
`origin_startpos` 是 COPY snapshot 和后续 catchup 的连接点。
本地还要维护 tablesync origin：
```text
origin name = pg_<suboid>_<relid>
replorigin_advance(originid, origin_startpos, ...)
replorigin_session_setup(originid, ...)
```
COPY 完成后，worker 写：
```text
UpdateSubscriptionRelState(..., SUBREL_STATE_FINISHEDCOPY, ...)
```
然后在共享内存中设置：
```text
relstate = SUBREL_STATE_SYNCWAIT
relstate_lsn = origin_startpos
```
它等待 apply worker 把它改成 `CATCHUP`。
这一步只属于当前进程握手，所以不进 catalog。
## 21. apply worker 与 table sync worker 的握手
apply worker 在 `ProcessSyncingTablesForApply()` 中遍历非 READY table。
如果还没有 table sync worker，它用 `launch_sync_worker()` 启动。
如果已经有 worker，它读取 worker shared memory：
```text
syncworker->relstate
syncworker->relstate_lsn
```
当看到 `SYNCWAIT`：
```text
apply worker:
  syncworker->relstate = CATCHUP
  syncworker->relstate_lsn = Max(syncworker->relstate_lsn, current_lsn)
  wake up sync worker
  等待 catalog 进入 SYNCDONE
```
table sync worker 看到 `CATCHUP` 后开始 logical apply。
它只处理自己的 `relid`。
当它追到指定 LSN：
```text
ProcessSyncingTablesForSync()
  -> shared state = SYNCDONE
  -> UpdateSubscriptionRelState(..., SYNCDONE, current_lsn)
  -> end streaming
  -> drop tablesync slot
  -> drop tablesync origin
  -> FinishSyncWorker()
```
apply worker 看到 `SYNCDONE` 后，还要等自己的 stream LSN 达到 `srsublsn`。
然后才写：
```text
UpdateSubscriptionRelState(..., READY, current_lsn)
```
`READY` 是 apply worker 正常处理该 relation 的边界。
## 22. apply worker 为什么跳过未 READY relation
`should_apply_changes_for_rel()` 把 relation state 用进每条 change 的执行判断。
table sync worker：
```text
只处理 MyLogicalRepWorker->relid
```
leader apply worker：
```text
READY:
  正常 apply
SYNCDONE 且 statelsn <= remote_final_lsn:
  可以 apply
其它非 READY:
  跳过，等待 table sync 路径负责
```
parallel apply worker 更严格。
如果目标 relation 不是 READY 且不是 UNKNOWN，会 ERROR。
原因是 streamed transaction 在 parallel apply 中可能还不知道 remote final LSN，不能安全判断一个仍在同步的表该不该应用。
这说明 `pg_subscription_rel` 直接影响 correctness。
它不是展示层状态。
## 23. ERROR 与自动 disable
apply worker 的主循环由 `start_apply()` 包裹。
table sync worker 的初始同步由 `start_table_sync()` 包裹。
如果发生 ERROR 且 `subdisableonerr` 为 true：
```text
DisableSubscriptionAndExit()
```
这个函数先从 ERROR 状态恢复：
```text
EmitErrorReport()
AbortOutOfAnyTransaction()
FlushErrorState()
```
再开启新事务：
```text
StartTransactionCommand()
PushActiveSnapshot(GetTransactionSnapshot())
DisableSubscription(MySubscription->oid)
CommitTransactionCommand()
```
这样失败的 apply transaction 不会和 disable catalog update 混在一起。
如果 `disable_on_error` 为 false：
```text
AbortOutOfAnyTransaction()
pgstat_report_subscription_error()
PG_RE_THROW()
```
worker 退出后，apply worker 由 launcher 按重启节流重新启动。
table sync worker 由 apply worker 后续根据 relation state 重启。
## 24. crash / restart 与节流
logical replication dynamic workers 设置：
```text
bgw_restart_time = BGW_NEVER_RESTART
```
所以 postmaster 不自动重启具体 worker。
apply worker 重启靠 launcher：
```text
enabled subscription
  + 没有 WORKERTYPE_APPLY worker
  + wal_retrieve_retry_interval 已过
  -> logicalrep_worker_launch(WORKERTYPE_APPLY)
```
launcher 用 DSA + dshash 保存每个 subscription 的 last start time：
```text
ApplyLauncherSetWorkerStartTime()
ApplyLauncherGetWorkerStartTime()
ApplyLauncherForgetWorkerStartTime()
```
参数变更导致的预期重启会 forget start time，从而不等待 retry interval。
table sync worker 也有 per-relation 节流。
`ProcessSyncingTablesForApply()` 维护 `last_start_times` hash。
`launch_sync_worker()` 只有在：
```text
nsyncworkers < max_sync_workers_per_subscription
last_start_time 已过 wal_retrieve_retry_interval
```
才会启动新的 sync worker。
这防止一张坏表或坏连接不断 fork worker。
## 25. table sync 崩溃后的恢复
table sync 的 catalog 状态就是恢复协议。
如果崩溃在 `DATASYNC`：
```text
下一次启动:
  relstate = DATASYNC
  drop tablesync slot if exists, missing_ok = true
  重新 COPY
```
因为 COPY 是否完整不可确定。
如果崩溃在 `FINISHEDCOPY`：
```text
下一次启动:
  relstate = FINISHEDCOPY
  恢复 tablesync origin progress
  跳过 COPY
  进入 catchup
```
因为 COPY 已经由 catalog 状态证明完成。
如果 table sync 已写 `SYNCDONE`，但 apply worker 还没写 `READY`：
```text
apply worker 继续等待 current_lsn >= srsublsn
然后写 READY
```
所以 `INIT/DATASYNC/FINISHEDCOPY/SYNCDONE/READY` 不是 UI 进度条。
它们是崩溃后继续推进所需的最小持久状态。
## 26. ownership 与 cleanup
catalog tuple 的 owner 是 SQL DDL。
运行时 `Subscription` 对象由 worker 的 `ApplyContext` 管。
worker slot 由 `LogicalRepWorkerLock` 保护。
生命周期大致是：
```text
logicalrep_worker_launch()
  -> 标记 worker slot in_use
  -> RegisterDynamicBackgroundWorker()
  -> WaitForReplicationWorkerAttach()
worker entrypoint
  -> logicalrep_worker_attach()
  -> worker->proc = MyProc
  -> before_shmem_exit(logicalrep_worker_onexit)
worker exit
  -> logicalrep_worker_detach()
  -> logicalrep_worker_cleanup()
  -> ApplyLauncherWakeup()
```
table sync 的外部资源包括 publisher tablesync slot 和 subscriber replication origin。
清理位置不止一个：
```text
sync worker 正常 SYNCDONE 后清理
apply worker READY 推进时补清理 origin
ALTER/DROP SUBSCRIPTION 路径按状态补清理 slot/origin
```
这些路径经常使用 `missing_ok = true`。
原因不是资源无关紧要，而是 DDL、apply worker 和 table sync worker 可能并发清理同一对象。
cleanup 必须尽量幂等。
## 27. 正确性机制层次
这条链路不是一个锁或一个 flag 保证的。
catalog transaction 保证：
```text
未提交 subscription 不会启动 worker；
relation state update 有事务可见性边界。
```
shared nailed catalog 保证：
```text
launcher 不进入具体 database 也能发现所有 subscription。
```
`LockSharedObject(SubscriptionRelationId, subid, ...)` 保证：
```text
worker 初始化时 subscription 不会被并发 drop 到不可解释状态。
```
syscache invalidation 保证：
```text
worker 的 SubscriptionCache 和 relation state cache 会过期。
```
`LogicalRepWorkerLock` 保证：
```text
worker slot 查找、分配、cleanup 不看到撕裂状态。
```
`relmutex` 保证：
```text
table sync worker 的 relstate / relstate_lsn 短临界区更新。
```
latch 保证：
```text
launcher、apply worker、table sync worker 能在等待中被唤醒。
```
slot 和 origin 保证：
```text
publisher 侧不会丢掉 COPY snapshot 之后需要 catchup 的 WAL；
subscriber 侧能保存 apply progress。
```
invalidation 不是 lock。
worker 何时处理 invalidation，取决于 apply loop 安全点。
## 28. 成本、资源与扩张变量
subscription 数量增加时，launcher 每轮扫描 `pg_subscription` 成本增加。
但这不是每条 change 的 hot path。
relation 数量增加时，成本主要在：
```text
ALTER SUBSCRIPTION refresh relation list 匹配
GetSubscriptionRelations() 扫非 READY relation
ProcessSyncingTablesForApply() 遍历 table_states_not_ready
```
table sync 并行度受：
```text
max_sync_workers_per_subscription
max_logical_replication_workers
max_worker_processes
```
三者任何一个不足都会限制实际 worker 数。
`max_logical_replication_workers` 是 logical replication 自己的 worker slot 数。
这个池同时服务：
```text
apply worker
table sync worker
sequence sync worker
parallel apply worker
```
`max_worker_processes` 是更外层的 postmaster background worker 槽。
publisher 侧也有成本。
每个 table sync worker 可能创建自己的 tablesync slot，持有 snapshot 相关资源，复制大量数据，并在 subscriber 端产生写入和 WAL。
提高 sync worker 数量可能缩短初始同步，也可能把瓶颈迁移到网络、I/O、publisher logical decoding 或 subscriber WAL flush。
## 29. `pg_stat_subscription` 诊断入口
`system_views.sql` 定义：
```text
pg_subscription su
LEFT JOIN pg_stat_get_subscription(NULL) st
  ON st.subid = su.oid
```
这意味着视图同时展示：
```text
catalog 中存在的 subscription
当前 shared memory 中 attach 的 worker
```
主要列：
```text
subid
subname
worker_type
pid
leader_pid
relid
received_lsn
last_msg_send_time
last_msg_receipt_time
latest_end_lsn
latest_end_time
```
`worker_type` 可能是：
```text
apply
parallel apply
sequence synchronization
table synchronization
```
如果 subscription 行存在但 worker 列为 NULL，说明当前没有 attach 的 worker。
它不直接说明原因。
原因可能是：
```text
subenabled = false
launcher 还没醒
wal_retrieve_retry_interval 还没过
max_logical_replication_workers 用尽
max_worker_processes 用尽
worker 启动后立即退出
publisher 连接失败
slot / 权限 / publication 配置错误
```
错误计数在：
```sql
SELECT *
FROM pg_stat_subscription_stats;
```
它能区分 apply error、sequence sync error、table sync error 以及冲突计数。
## 30. catalog 诊断组合
判断 apply worker 是否应该存在：
```sql
SELECT oid, subname, subenabled, subslotname, subpublications
FROM pg_subscription;
```
判断当前 worker：
```sql
SELECT subname, worker_type, pid, leader_pid, relid,
       received_lsn, latest_end_lsn
FROM pg_stat_subscription
ORDER BY subname, worker_type, relid;
```
判断 table sync：
```sql
SELECT srsubid, srrelid, srsubstate, srsublsn
FROM pg_subscription_rel
ORDER BY srsubid, srrelid;
```
典型解释：
```text
subenabled = true, pg_stat_subscription 没有 apply pid:
  查 launcher、worker limit、bgworker limit、重启节流、日志。
apply worker 存在，pg_subscription_rel 有 INIT/DATASYNC/FINISHEDCOPY:
  table sync 仍在推进或尚未启动。
pg_subscription_rel = SYNCDONE:
  table sync 已追到边界，apply worker 还要追到 srsublsn 才能写 READY。
pg_subscription_rel = READY:
  apply worker 应按普通 relation apply 路径处理该表。
```
日志关键词：
```text
logical replication launcher started
logical replication apply worker ... has started
logical replication table synchronization worker ... has started
out of logical replication worker slots
out of background worker slots
subscription ... has been disabled because of an error
worker ... will restart because of a parameter change
```
`pg_stat_subscription` 是入口。
`pg_subscription_rel` 和日志给状态解释。
源码里的重启节流和 worker limit 给因果边界。
## 31. wait event 与断点
常用 wait event：
```text
WAIT_EVENT_LOGICAL_LAUNCHER_MAIN
WAIT_EVENT_BGWORKER_STARTUP
WAIT_EVENT_BGWORKER_SHUTDOWN
WAIT_EVENT_LOGICAL_SYNC_DATA
WAIT_EVENT_LOGICAL_SYNC_STATE_CHANGE
```
源码断点：
```gdb
break ApplyLauncherMain
break get_subscription_list
break logicalrep_worker_launch
break ApplyWorkerMain
break InitializeLogRepWorker
break maybe_reread_subscription
break ProcessSyncingRelations
break launch_sync_worker
break LogicalRepSyncTableStart
break ProcessSyncingTablesForApply
break ProcessSyncingTablesForSync
```
在 `logicalrep_worker_launch()` 看：
```gdb
p wtype
p subid
p relid
p nsyncworkers
p nparallelapplyworkers
p max_logical_replication_workers
```
在 table sync worker 看：
```gdb
p MyLogicalRepWorker->relstate
p MyLogicalRepWorker->relstate_lsn
p *MySubscription
```
这些断点能把 SQL 看到的状态直接连回源码状态转移。
## 32. 常见误区
误区一：把 `pg_subscription` 当作连接字符串表。
它还保存 owner、enabled、slot、publication、streaming、two_phase、origin、错误策略和冲突检测状态。
误区二：以为 launcher 直接启动 table sync worker。
launcher 启动 apply worker；table sync worker 由 apply worker 根据 `pg_subscription_rel` 调度。
误区三：以为 `subenabled = true` 就一定有 worker。
它只是持久意图，不保证资源和连接都成功。
误区四：以为 `SYNCWAIT` 和 `CATCHUP` 应该能在 `pg_subscription_rel` 看到。
它们只在 shared memory 中表示当前 worker 握手。
误区五：以为 invalidation 会立即中断当前 apply。
invalidation 只标记缓存过期，worker 在安全点重读并决定是否退出。
误区六：以为 `READY` 之前 apply worker 完全不管该表。
apply worker 会用 relation state 判断跳过、等待或最终写 READY。
## 33. 课堂实验
实验一：观察 apply worker 启动。
```sql
SELECT oid, subname, subenabled, subslotname, subpublications
FROM pg_subscription;
SELECT subname, worker_type, pid, relid
FROM pg_stat_subscription
ORDER BY subname, worker_type, relid;
```
配合断点：
```gdb
break ApplyLauncherMain
break get_subscription_list
break logicalrep_worker_launch
break ApplyWorkerMain
break InitializeLogRepWorker
```
画出：
```text
CREATE SUBSCRIPTION commit
  -> ApplyLauncherWakeupAtCommit
  -> launcher 扫 pg_subscription
  -> logicalrep_worker_launch(WORKERTYPE_APPLY)
  -> ApplyWorkerMain
  -> GetSubscription
  -> walrcv_startstreaming
```
实验二：观察 table sync 状态。
```sql
SELECT srrelid::regclass, srsubstate, srsublsn
FROM pg_subscription_rel
ORDER BY srrelid;
SELECT worker_type, pid, relid::regclass, received_lsn, latest_end_lsn
FROM pg_stat_subscription
WHERE subname = '<sub>'
ORDER BY worker_type, relid;
```
配合断点：
```gdb
break ProcessSyncingTablesForApply
break launch_sync_worker
break LogicalRepSyncTableStart
break ProcessSyncingTablesForSync
```
目标是观察：
```text
INIT -> DATASYNC -> FINISHEDCOPY -> SYNCDONE -> READY
```
并记住 `SYNCWAIT` / `CATCHUP` 只能从 shared memory 或断点看到。
实验三：观察参数变更。
修改 `synchronous_commit`，观察 worker 原地更新。
修改 publication / slot / origin / stream 语义，观察 worker 退出并由 launcher 重启。
关键断点：
```gdb
break subscription_change_cb
break maybe_reread_subscription
break ApplyLauncherForgetWorkerStartTime
```
## 34. 讨论题
1. 为什么 `pg_subscription` 有 `subdbid`，却仍然必须是 shared catalog？
2. launcher 为什么只读少数字段，不读 `subconninfo` 和 `subpublications`？
3. `subenabled = true` 但 `pg_stat_subscription.pid IS NULL`，可能有哪些不同原因？
4. `SYNCWAIT` 和 `CATCHUP` 为什么不写入 `pg_subscription_rel`？
5. `DATASYNC` 崩溃和 `FINISHEDCOPY` 崩溃后的恢复策略为什么不同？
6. 为什么修改 `publications` 通常要重启 worker，而修改 `synchronous_commit` 可以原地更新？
7. `pg_stat_subscription` 左连接 `pg_subscription` 的设计有什么诊断好处和盲区？
8. 如果把 subscription 做成配置文件而不是 catalog，会在哪些 DDL、权限、invalidation、crash recovery 场景中变复杂？
## 35. 本节小结
本节主链路：
```text
pg_subscription 保存订阅级持久意图
  -> launcher 扫描 enabled subscription
  -> logicalrep_worker_launch 分配 shared worker slot 并注册 dynamic bgworker
  -> apply worker 重新读取完整 Subscription
  -> apply worker 根据 pg_subscription_rel 的非 READY 状态启动 table sync worker
  -> table sync worker 用 slot/origin/COPY/catchup 推进 relation 到 READY
```
核心状态边界：
```text
pg_subscription:
  subscription identity、enabled、owner、connection、slot、publication、stream、错误策略
pg_subscription_rel:
  relation membership 与 INIT/DATASYNC/FINISHEDCOPY/SYNCDONE/READY
LogicalRepWorker shared memory:
  当前 worker slot、worker type、pid、relstate IPC、统计 LSN
SubscriptionCache:
  worker-local MySubscription + invalidation + maybe_reread_subscription
```
错误路径的规律：
```text
worker 失败可以重启；
catalog 状态必须足够说明重启后从哪里继续；
slot/origin cleanup 要能在并发和崩溃后重复执行；
重启节流和 worker limit 防止错误被无限放大。
```
可迁移规律：
```text
只要后台任务需要跨数据库发现、跨进程调度、跨崩溃恢复和 DDL 可见性，
它就不能只是 backend-local 配置；
它需要持久 catalog truth、当前 worker truth，以及二者之间的 invalidation / restart 协议。
```
