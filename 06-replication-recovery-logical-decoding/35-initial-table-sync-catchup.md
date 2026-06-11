# PostgreSQL table sync worker 初始拷贝与 catchup 交接

## 课程定位

前置知识：已经理解 logical decoding 会把 WAL 转成 logical change，
知道 subscription apply worker 从 publisher slot 拉取 `pgoutput` 流，
也知道 subscriber 端使用 replication origin 记录已经应用到的远端 LSN。

本节唯一主问题：

```text
新订阅或新表加入时，
table sync worker 如何拷贝初始数据、追赶增量变更，
并把 pg_subscription_rel 状态推进到可由主 apply worker 接管？
```

核心矛盾：

```text
初始 COPY 必须基于 publisher 的一致 snapshot 读取已有行；
但 snapshot 之后的 WAL 不能丢，
主 apply worker 又不能在 COPY 未完成时提前应用同一张表的增量。
```

PostgreSQL 的解法是为每张需要初始化的表启动 tablesync worker。
它使用独立的同步 slot 和 per-table replication origin，
先做 snapshot COPY，
再从同一边界开始 replay 增量，
最后通过 `pg_subscription_rel` 和 shared worker slot 把表交回主 apply worker。

学完后应能判断：

```text
哪些状态写入 pg_subscription_rel；
哪些状态只在 LogicalRepWorker shared memory 中协调；
为什么 FINISHEDCOPY 可恢复而 SYNCWAIT/CATCHUP 不落 catalog；
为什么当前 master 使用 permanent tablesync slot，而不是 temporary slot；
为什么 initial COPY 不会自动 TRUNCATE subscriber 表；
为什么主 apply worker 要等 SYNCDONE LSN 后才能写 READY。
```

本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

logical replication 的常规 apply 链路是：

```text
publisher WAL
  -> logical decoding
  -> pgoutput
  -> walsender
  -> subscriber apply worker
  -> local table insert/update/delete/truncate
```

如果 subscriber 上的表已经有正确基线，
主 apply worker 只要按 WAL 顺序应用 change。

但新订阅或 `ALTER SUBSCRIPTION ... REFRESH PUBLICATION` 加入新表时，
系统还要补一份 publisher 上已经存在的数据。

最简单的实现是：

```text
停止 apply
  -> 对所有表 COPY
  -> 从 COPY 后继续 apply
```

PostgreSQL 没有这样做。
原因是 COPY 可能持续很久，
而且一张大表不应阻塞其它已经同步好的表。

本节的主线是 per-table 初始化：

```text
每张表独立记录同步状态；
主 apply worker 继续推进订阅流；
未 READY 的表由 tablesync worker 负责补齐；
到达交接 LSN 后再由主 apply worker 接管。
```

本节不展开 sequence sync。
不展开 parallel apply 的完整文件队列。
不讲 logical decoding snapshot build 的内部算法。
这些模块只作为 table sync 状态推进的边界出现。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
pg_subscription_rel 保存每张表的持久同步状态；
主 apply worker 发现非 READY 表后启动 tablesync worker；
tablesync worker 在 publisher 上用 REPEATABLE READ + SNAPSHOT 'use' 创建同步 slot，
基于同一 snapshot COPY 初始数据；
COPY 完成后进入 SYNCWAIT；
主 apply worker 给它 CATCHUP target LSN；
tablesync worker replay 到 target 后写 SYNCDONE；
主 apply worker 自己也到达 SYNCDONE LSN 后写 READY。
```

这条链解决的是一个 streaming 系统的典型问题：

```text
bulk snapshot load
  + continuous delta stream
  + exactly-once handoff per relation
```

一个布尔值无法表达这个过程。
系统必须区分：

```text
尚未开始 COPY；
COPY 正在进行，失败后可以重做；
COPY 已完成，失败后不能重做；
sync worker 等待主 apply 给 catchup 目标；
sync worker 正在追赶；
sync worker 追赶完成；
主 apply 到达同一 LSN 并接管。
```

因此当前源码的状态推进是：

```text
INIT
  -> DATASYNC
  -> FINISHEDCOPY
  -> SYNCWAIT
  -> CATCHUP
  -> SYNCDONE
  -> READY
```

其中 `SYNCWAIT` 和 `CATCHUP` 不写 catalog。
它们只用于当前 worker 之间的短期协调。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_subscription_rel.h` | `srsubstate`、`srsublsn`、`SUBREL_STATE_*`。 |
| 2 | `src/backend/catalog/pg_subscription.c` | 增删改查 subscription relation state。 |
| 3 | `src/backend/commands/subscriptioncmds.c` | `CREATE/REFRESH` 如何写 `INIT` 或 `READY`。 |
| 4 | `src/backend/replication/logical/syncutils.c` | 状态缓存、worker 拉起、同步关系分派、sync worker 退出。 |
| 5 | `src/backend/replication/logical/launcher.c` | `LogicalRepWorker` slot 分配、attach、cleanup。 |
| 6 | `src/backend/replication/logical/tablesync.c` | 初始 COPY、sync slot、origin、catchup 交接。 |
| 7 | `src/backend/replication/logical/worker.c` | apply loop、relation 过滤、commit 后同步推进。 |
| 8 | `src/include/replication/worker_internal.h` | `LogicalRepWorker.relstate`、`relstate_lsn`、`relmutex`。 |
| 9 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | `walrcv_create_slot()` 组装 replication command。 |
| 10 | `src/backend/replication/walsender.c` | `CRS_USE_SNAPSHOT` 的 publisher 侧约束。 |
| 11 | `src/backend/commands/copyfrom.c` | `BeginCopyFrom()`、`CopyFrom()`。 |
| 12 | `src/backend/access/table/tableam.c` / `src/include/access/tableam.h` | COPY 插入最终进入 table AM 的边界。 |

推荐阅读顺序是先读状态，再读状态推进。

```text
pg_subscription_rel.h
  -> pg_subscription.c
  -> syncutils.c
  -> tablesync.c
  -> worker.c
  -> launcher.c
```

如果从 `tablesync.c` 顶部一路顺序读，
很容易把 COPY、slot、origin、wait、cleanup 混成一个长函数。

本节要抓住的不是函数数量，
而是一个 relation 从 `INIT` 到 `READY` 的时间线。

## 4. `pg_subscription_rel` 状态边界

`pg_subscription_rel` 是 subscriber 本地 catalog。

关键字段是：

| 字段 | 语义 |
| --- | --- |
| `srsubid` | subscription OID。 |
| `srrelid` | 本地 table 或 sequence OID。 |
| `srsubstate` | relation 在该 subscription 下的状态。 |
| `srsublsn` | 用于协调的远端 LSN；部分状态为 NULL。 |

`pg_subscription_rel.h` 里表同步状态包括：

| 状态 | 常量 | 位置 | `srsublsn` | 语义 |
| --- | --- | --- | --- | --- |
| `i` | `SUBREL_STATE_INIT` | catalog | NULL | 需要初始化，尚未完成 COPY。 |
| `d` | `SUBREL_STATE_DATASYNC` | catalog | NULL | tablesync worker 正在同步初始数据。 |
| `f` | `SUBREL_STATE_FINISHEDCOPY` | catalog | NULL | COPY 已完成，失败重启后不应重做 COPY。 |
| `s` | `SUBREL_STATE_SYNCDONE` | catalog | set | sync worker 已追到交接 LSN。 |
| `r` | `SUBREL_STATE_READY` | catalog | set | 主 apply worker 可以正常接管。 |
| `w` | `SUBREL_STATE_SYNCWAIT` | shared memory | worker field | sync worker 等主 apply 给 target。 |
| `c` | `SUBREL_STATE_CATCHUP` | shared memory | worker field | sync worker 正在追 target。 |
| `\0` | `SUBREL_STATE_UNKNOWN` | 返回值/worker 初始值 | invalid | 无映射或未知。 |

`srsublsn` 只有在 `SYNCDONE` 和 `READY` 这类交接状态里有协调意义。
在 `INIT/DATASYNC/FINISHEDCOPY` 中通常为 NULL。

不要把 raw char 当完整语义。

```text
srsubstate
  + srsublsn
  + 当前 worker 类型
  + shared memory relstate
  + 当前 remote_final_lsn
  = relation 是否能由当前 worker 应用
```

`FINISHEDCOPY` 是最容易被忽略的状态。
它不是为了展示进度。
它是 crash recovery 边界。

COPY 已经成功提交后，
如果 worker 崩溃，
重启不能重复 COPY。
否则 subscriber 可能出现重复行、唯一约束错误或触发器副作用。

`SYNCWAIT` 和 `CATCHUP` 则相反。
它们只描述当前 worker 之间的协作。
worker 崩溃后没有恢复价值。
从 `FINISHEDCOPY` 再次进入等待即可。

## 5. DDL 如何进入状态机

入口在 `subscriptioncmds.c`。

`CREATE SUBSCRIPTION` 或 `ALTER SUBSCRIPTION ... REFRESH PUBLICATION`
会连接 publisher，
获取 publication relation 列表，
在 subscriber 上找到同名本地 relation，
再调用：

```text
AddSubscriptionRelState()
```

核心分叉是：

```text
copy_data = true:
  SUBREL_STATE_INIT

copy_data = false:
  SUBREL_STATE_READY
```

`copy_data=false` 不是延期同步。
它表示系统不做 initial table copy，
该表从一开始就视为可由主 apply worker 正常处理。

因此数据基线正确性的责任转给用户。

`copy_data=true` 才进入 tablesync 状态机。

这里还有一个重要限制。
当 subscription 的 two-phase 已经 enabled 时，
`ALTER SUBSCRIPTION ... REFRESH PUBLICATION` 通常不允许 `copy_data=true`。

原因是 prepared transaction 与未 READY 表的跳过规则很难安全组合。
这不是 SQL 层随意限制。
它来自 `worker.c` 中 relation-level apply 判断。

另一个边界是 TRUNCATE。

initial sync 不会自动 TRUNCATE subscriber 本地表。
`tablesync.c` 走的是 `COPY FROM` 插入路径。
如果本地已有数据，
冲突按普通 COPY/INSERT 语义暴露。

后续 publisher 的 TRUNCATE 是 logical replication message，
在 `worker.c` 的 `apply_handle_truncate()` 中处理，
并同样经过 `should_apply_changes_for_rel()` 判断。

## 6. 主流程源码 walkthrough：主 apply worker 如何发现需要同步的表

主 apply worker 维护一个 backend-local 列表：

```text
table_states_not_ready
```

这个列表定义在 `tablesync.c`，
由 `syncutils.c` 的 `FetchRelationStates()` 重建。

`SetupApplyOrSyncWorker()` 注册 `SUBSCRIPTIONRELMAP` syscache callback：

```text
CacheRegisterSyscacheCallback(SUBSCRIPTIONRELMAP,
                              InvalidateSyncingRelStates,
                              (Datum) 0)
```

`InvalidateSyncingRelStates()` 只做一件事：

```text
relation_states_validity = SYNC_RELATIONS_STATE_NEEDS_REBUILD
```

下一次 `FetchRelationStates()` 会：

```text
GetSubscriptionRelations(MySubscription->oid, true, true, true)
清理旧 table_states_not_ready
把非 READY table 状态复制到 CacheMemoryContext
记录是否还有非 READY sequence
```

所以 DDL 提交后，
apply worker 不一定在任意指令点马上看见新表。
它在下一次处理同步关系时重建缓存。

主 apply 的调度入口是：

```text
ProcessSyncingRelations(current_lsn)
```

`worker.c` 的 `apply_handle_commit()` 在事务提交后调用它：

```text
apply_handle_commit()
  -> apply_handle_commit_internal()
  -> ProcessSyncingRelations(commit_data.end_lsn)
```

对于 leader apply worker，
`ProcessSyncingRelations()` 调用：

```text
ProcessSyncingTablesForApply(current_lsn)
ProcessSequencesForSync()
```

因此 table sync 的很多状态推进发生在 commit 边界。
这和 logical replication 按事务应用的模型一致。

## 7. tablesync worker launch

当 `ProcessSyncingTablesForApply()` 发现某张非 READY 表没有 tablesync worker，
它会调用：

```text
launch_sync_worker(WORKERTYPE_TABLESYNC,
                   nsyncworkers,
                   relid,
                   &last_start_time)
```

`launch_sync_worker()` 做两个限制。

第一，
每个 subscription 的 sync worker 数不能超过：

```text
max_sync_workers_per_subscription
```

第二，
同一张表失败后不能立刻 tight-loop 重启。
它使用 `wal_retrieve_retry_interval` 控制重试间隔。

真正分配 worker slot 的函数在 `launcher.c`：

```text
logicalrep_worker_launch()
```

它在 `LogicalRepCtx->workers[]` 里寻找空槽，
写入：

```text
type = WORKERTYPE_TABLESYNC
subid
relid
dbid
userid
relstate = SUBREL_STATE_UNKNOWN
relstate_lsn = InvalidXLogRecPtr
```

tablesync worker 的 background worker 函数名是：

```text
TableSyncWorkerMain
```

`LogicalRepWorker` 中和本节最相关的字段是：

| 字段 | 语义 |
| --- | --- |
| `in_use` | shared worker slot 是否被占用。 |
| `generation` | slot 重用代数，避免错认旧 worker。 |
| `proc` | worker attach 后的 `PGPROC *`。 |
| `subid` | subscription OID。 |
| `relid` | tablesync worker 负责的表。 |
| `relstate` | worker 间协调状态。 |
| `relstate_lsn` | worker 间协调 LSN。 |
| `relmutex` | 保护 `relstate` 和 `relstate_lsn` 的 spinlock。 |

`logicalrep_worker_launch()` 注册动态后台进程后，
会等待 worker attach。

资源不足时常见报错：

```text
out of logical replication worker slots
out of background worker slots
```

前者通常看 `max_logical_replication_workers`。
后者通常看 `max_worker_processes`。

## 8. tablesync worker 启动与恢复分支

入口调用链是：

```text
TableSyncWorkerMain()
  -> SetupApplyOrSyncWorker(worker_slot)
  -> run_tablesync_worker()
  -> FinishSyncWorker()
```

`SetupApplyOrSyncWorker()` 会 attach shared worker slot，
加载 `libpqwalreceiver`，
初始化本地数据库连接，
读取 `MySubscription`，
并注册 subscription relation state 的 syscache invalidation callback。

`run_tablesync_worker()` 的主线是：

```text
start_table_sync(&origin_startpos, &slotname)
set_stream_options(&options, slotname, &origin_startpos)
walrcv_startstreaming(LogRepWorkerWalRcvConn, &options)
start_apply(origin_startpos)
```

`start_table_sync()` 用 `PG_TRY()` 包住 `LogicalRepSyncTableStart()`。

如果失败且 `disable_on_error` 打开，
会调用：

```text
DisableSubscriptionAndExit()
```

否则 abort 当前事务，
上报 subscription error，
再重新抛出错误。

`LogicalRepSyncTableStart()` 首先读取当前 catalog 状态：

```text
relstate = GetSubscriptionRelState(subid, relid, &relstate_lsn)
```

然后把它写入 shared worker slot：

```text
MyLogicalRepWorker->relstate = relstate
MyLogicalRepWorker->relstate_lsn = relstate_lsn
```

如果状态已经是：

```text
SYNCDONE
READY
UNKNOWN
```

worker 直接 `FinishSyncWorker()`。

如果状态是 `DATASYNC`，
说明上次可能在 COPY 未完成时失败。
源码选择尝试删除旧同步 slot，
然后重新 COPY：

```text
ReplicationSlotDropAtPubNode(..., slotname, true)
```

这里 `missing_ok=true`。
旧 slot 可能没有创建成功，
也可能已经被其它清理路径删除。

如果状态是 `FINISHEDCOPY`，
说明 COPY 已经完成。
worker 不能再 COPY。

它找到 per-table origin，
读取 progress：

```text
replorigin_by_name(originname, false)
replorigin_session_setup(originid, 0)
origin_startpos = replorigin_session_get_progress(false)
```

然后跳到 COPY 完成后的等待逻辑。

这就是 `FINISHEDCOPY` 的恢复语义：

```text
COPY 完成事实持久化；
catchup 起点从 per-table origin 恢复；
不重复导入初始数据。
```

## 9. 远端 snapshot、同步 slot 与 origin

正常路径下，
worker 先把状态改成 `DATASYNC` 并写入 catalog：

```text
UpdateSubscriptionRelState(subid,
                           relid,
                           SUBREL_STATE_DATASYNC,
                           InvalidXLogRecPtr,
                           false)
```

然后创建或获取 per-table replication origin。

origin 名由 `worker.c` 的公共函数生成：

```text
ReplicationOriginNameForLogicalRep(subid, relid, originname, sizeof(originname))
```

tablesync worker 的 origin 名形如：

```text
pg_<subid>_<relid>
```

主 apply worker 的 origin 名不带 relid：

```text
pg_<subid>
```

接着 worker 打开本地目标表：

```text
table_open(relid, RowExclusiveLock)
```

源码注释说明这里不用更强锁，
因为不想阻塞主 apply worker 打开 relation 做 relation mapping。

在 publisher 上，
worker 启动远端事务：

```text
BEGIN READ ONLY ISOLATION LEVEL REPEATABLE READ
```

然后创建同步 slot：

```text
walrcv_create_slot(LogRepWorkerWalRcvConn,
                   slotname,
                   false /* permanent */,
                   false /* two_phase */,
                   MySubscription->failover,
                   CRS_USE_SNAPSHOT,
                   origin_startpos)
```

这里 `temporary` 明确是 `false`。
当前 master 使用的是 permanent tablesync slot。
同步完成后再显式删除。

`libpqwalreceiver.c` 会把 `CRS_USE_SNAPSHOT` 拼成：

```text
CREATE_REPLICATION_SLOT "<slot>" LOGICAL pgoutput (SNAPSHOT 'use')
```

或旧语法：

```text
CREATE_REPLICATION_SLOT "<slot>" LOGICAL pgoutput USE_SNAPSHOT
```

publisher 侧 `walsender.c` 要求 `SNAPSHOT 'use'`：

```text
必须在事务块中；
必须是 REPEATABLE READ；
必须是 READ ONLY；
必须尚未设置 first snapshot；
不能在 subtransaction 中。
```

这就是 tablesync 先发 read-only repeatable read 事务的原因。

slot 返回的 LSN 写入 `origin_startpos`。
tablesync worker 随后推进 per-table origin：

```text
replorigin_advance(originid,
                   *origin_startpos,
                   InvalidXLogRecPtr,
                   true /* go backward */,
                   true /* WAL log */)
```

并设置当前 session origin：

```text
replorigin_session_setup(originid, 0)
replorigin_xact_state.origin = originid
```

这样 COPY 和后续 catchup 共享同一条可恢复进度边界。

## 10. 初始 COPY 的数据路径

COPY 前，
worker 决定用哪个角色写本地表。

如果 subscription 没有 `run_as_owner`，
它切到目标表 owner：

```text
SwitchToUntrustedUser(rel->rd_rel->relowner, &ucxt)
```

然后检查本地 INSERT 权限。

它还会检查 RLS。
`COPY FROM` 不执行普通 RLS policy，
所以对 RLS enabled relation，
没有 BYPASSRLS 的角色不能作为 logical replication 目标写入。

真正 COPY 调用：

```text
PushActiveSnapshot(GetTransactionSnapshot())
copy_table(rel)
PopActiveSnapshot()
```

`copy_table()` 先在 publisher 上取 relation 信息：

```text
fetch_remote_table_info()
```

它会读取远端 relation OID、replica identity、relkind、列列表、row filter、
以及 generated column 是否被发布。

然后更新本地 logical relation map：

```text
logicalrep_relmap_update(&lrel)
logicalrep_rel_open(lrel.remoteid, NoLock)
```

普通表、无 row filter、无 published generated column 时，
远端命令是：

```text
COPY schema.table (col1, col2, ...) TO STDOUT
```

否则使用 SELECT 形式：

```text
COPY (
  SELECT col1, col2, ...
  FROM ONLY schema.table
  WHERE filter1 OR filter2
) TO STDOUT
```

如果 publisher 版本 >= 16 且 subscription 开启 binary，
命令追加：

```text
WITH (FORMAT binary)
```

本地导入复用普通 COPY FROM：

```text
BeginCopyFrom(..., copy_read_data, attnamelist, options)
CopyFrom(cstate)
EndCopyFrom(cstate)
```

`copy_read_data()` 从 walreceiver connection 读 COPY OUT 数据：

```text
walrcv_receive(LogRepWorkerWalRcvConn, &buf, &fd)
```

没有数据时等待：

```text
WAIT_EVENT_LOGICAL_SYNC_DATA
```

所以 COPY 阶段在 `pg_stat_activity` 里常见 wait event 是：

```text
LogicalSyncData
```

`CopyFrom()` 在 `copyfrom.c` 里处理约束、触发器、分区路由和插入。
普通表最终通过 table AM 边界：

```text
table_tuple_insert(...)
  -> rel->rd_tableam->tuple_insert(...)
```

这说明 tablesync 不直接操作 heap page。
它使用 PostgreSQL 普通 COPY/INSERT 语义。

COPY 完成后，
worker 在 publisher 上提交远端事务，
恢复用户上下文，
关闭本地 relation，
执行 `CommandCounterIncrement()`，
然后写：

```text
SUBREL_STATE_FINISHEDCOPY
```

## 11. `FINISHEDCOPY -> SYNCWAIT -> CATCHUP`

写完 `FINISHEDCOPY` 后，
tablesync worker 进入 shared memory 状态：

```text
MyLogicalRepWorker->relstate = SUBREL_STATE_SYNCWAIT
MyLogicalRepWorker->relstate_lsn = *origin_startpos
```

然后等待主 apply worker 改成 `CATCHUP`：

```text
wait_for_worker_state_change(SUBREL_STATE_CATCHUP)
```

这个等待函数会：

```text
检查自己的 relstate；
查找 WORKERTYPE_APPLY worker；
唤醒 apply worker；
用 latch 等待 LogicalSyncStateChange；
timeout 后重试；
如果 apply worker 消失则返回 false。
```

主 apply worker 在 `ProcessSyncingTablesForApply()` 中发现 sync worker 的 `SYNCWAIT` 后，
会改 shared state：

```text
syncworker->relstate = SUBREL_STATE_CATCHUP
syncworker->relstate_lsn =
  Max(syncworker->relstate_lsn, current_lsn)
```

这里的 `current_lsn` 是主 apply 当前处理到的 commit end LSN。

`Max()` 的语义是：

```text
sync worker 至少要追到主 apply 已经走过的位置；
否则主 apply 曾跳过的该表 change 可能丢失。
```

改完后主 apply 唤醒 sync worker。

然后主 apply 等待 catalog 状态变成 `SYNCDONE`：

```text
wait_for_table_state_change(relid, SUBREL_STATE_SYNCDONE)
```

进入等待前，
源码会提交当前事务并释放已持有锁。
注释说明这是为了避免 latch wait 期间形成 deadlock detector 看不见的等待环。

## 12. 主 apply worker 为什么必须过滤未 READY 表

过滤逻辑在 `worker.c`：

```text
should_apply_changes_for_rel()
```

tablesync worker 的规则很简单：

```text
只应用自己的 relid
```

leader apply worker 的规则是：

```text
rel->state == SUBREL_STATE_READY
  ||
(rel->state == SUBREL_STATE_SYNCDONE
 && rel->statelsn <= remote_final_lsn)
```

在 `INIT/DATASYNC/FINISHEDCOPY/SYNCWAIT/CATCHUP` 阶段，
主 apply 不应用该表的 DML。
这些 change 会由 tablesync worker 从同步 slot catch up。

在 `SYNCDONE` 阶段，
主 apply 只有当当前远端事务的 `remote_final_lsn` 已经到达 `statelsn`，
才允许应用该表。

源码注释说明 `<=` 是必要的。
`SYNCDONE` 可能保存的是 initial slot consistent point WAL record 的结束位置加一，
下一条 record 可能就是当前事务 COMMIT。

parallel apply worker 更保守。
如果目标表不是 `READY` 或 `UNKNOWN`，
它会报错退出，
因为 streaming transaction 被拆分后未必能安全使用完整 `remote_final_lsn` 做判断。

TRUNCATE 也经过同一判断。

`apply_handle_truncate()` 打开 relation 后，
如果 `should_apply_changes_for_rel()` 返回 false，
就关闭 relation 并跳过。

所以未 READY 的表不会被主 apply 提前 TRUNCATE。

## 13. tablesync worker catchup 与 `SYNCDONE`

sync worker 被改成 `CATCHUP` 后，
`run_tablesync_worker()` 启动 logical stream：

```text
set_stream_options(&options, slotname, &origin_startpos)
walrcv_startstreaming(LogRepWorkerWalRcvConn, &options)
start_apply(origin_startpos)
```

它进入普通 `LogicalRepApplyLoop()`。

每个远端事务提交后，
`apply_handle_commit()` 调用：

```text
ProcessSyncingRelations(commit_data.end_lsn)
```

由于当前 worker 类型是 `WORKERTYPE_TABLESYNC`，
分派到：

```text
ProcessSyncingTablesForSync(current_lsn)
```

核心判断是：

```text
if relstate == CATCHUP
   and current_lsn >= relstate_lsn:
  relstate = SYNCDONE
  relstate_lsn = current_lsn
```

随后 worker 写 catalog：

```text
UpdateSubscriptionRelState(subid,
                           relid,
                           SUBREL_STATE_SYNCDONE,
                           current_lsn,
                           false)
```

写 `SYNCDONE` 后，
worker 结束 streaming，
删除 publisher 上的 sync slot：

```text
walrcv_endstreaming(...)
ReplicationSlotDropAtPubNode(..., syncslotname, false)
```

这里 `missing_ok=false`。
删除失败必须报错。
因为这是 permanent sync slot，
如果 silent failure，
publisher WAL 可能一直被保留。

然后 worker 开新事务清理 per-table origin：

```text
replorigin_session_reset()
replorigin_xact_clear(true)
replorigin_drop_by_name(originname, true, false)
```

清理 origin 放在 `SYNCDONE` 之后。
如果先清 progress 再写状态，
错误重启时可能无法知道已 replay 到哪里。

最后调用：

```text
FinishSyncWorker()
```

`FinishSyncWorker()` 会提交未完成事务，
执行 `XLogFlush(GetXLogWriteRecPtr())`，
记录完成日志，
并唤醒 leader apply worker。

## 14. 主 apply worker 写 `READY`

`SYNCDONE` 还不是接管完成。

`SYNCDONE` 的语义是：

```text
sync worker 已经把该表 replay 到某个 LSN。
```

`READY` 的语义是：

```text
主 apply worker 自己已经到达该 LSN，
之后可以正常应用该表。
```

主 apply 在 `ProcessSyncingTablesForApply(current_lsn)` 中看到 `SYNCDONE` 后，
只有满足：

```text
current_lsn >= rstate->lsn
```

才写：

```text
SUBREL_STATE_READY
```

写 READY 前，
主 apply 还会尝试删除 per-table origin。
源码注释说明不能完全依赖 tablesync worker 清理。
如果 tablesync worker 清理 origin 时报错，
系统不会为了只清 origin 再重启它。

所以 READY 路径也会兜底：

```text
replorigin_drop_by_name(originname, true, false)
UpdateSubscriptionRelState(... READY ...)
```

写 READY 后，
下一次 `FetchRelationStates()` 重建缓存时，
该表从 `table_states_not_ready` 消失。

从此 `should_apply_changes_for_rel()` 对该表按普通 READY 表处理。

## 15. 生命周期、正确性与 cleanup

本节有四类长期对象：`pg_subscription_rel` 行、shared worker slot、publisher 上的 tablesync slot、
以及 subscriber 上的 per-table replication origin。

`pg_subscription_rel` 由 `AddSubscriptionRelState()` 创建。
tablesync worker 写 `DATASYNC`、`FINISHEDCOPY`、`SYNCDONE`，
主 apply worker 写 `READY`。
DDL refresh、drop subscription 和 `RemoveSubscriptionRel()` 负责删除。

shared worker slot 由 `logicalrep_worker_launch()` 分配，
由 `logicalrep_worker_attach()` 绑定到 `MyLogicalRepWorker`，
并在 `logicalrep_worker_onexit()` 中通过 `logicalrep_worker_cleanup()` 释放。
`relstate` 与 `relstate_lsn` 是 shared memory 字段，
读写用 `relmutex` 的短临界区保护。

tablesync slot 由 `walrcv_create_slot(... temporary=false, CRS_USE_SNAPSHOT, ...)` 创建。
它保留 COPY snapshot 之后的 logical changes。
正常路径在 `ProcessSyncingTablesForSync()` 中用 `ReplicationSlotDropAtPubNode(..., missing_ok=false)` 删除。
`DATASYNC` 重启和 DDL remove 表映射时会用 `missing_ok=true` 兜底删除可能存在的旧 slot。

per-table origin 由 `replorigin_create("pg_<subid>_<relid>")` 创建，
由 `replorigin_advance()` 推进到 `origin_startpos`，
catchup apply loop 使用它记录进度。
tablesync worker 在 `SYNCDONE` 后清理，
主 apply worker 在写 `READY` 前再兜底清理，
DDL/drop subscription 也会用 `missing_ok=true` 清理。

正确性不是单个机制保证的。
`REPEATABLE READ` 加 `SNAPSHOT 'use'` 给 COPY 和 stream 同一边界；
permanent sync slot 保留后续 WAL；
per-table origin 让 catchup progress 可恢复；
`pg_subscription_rel` 让 crash/restart 看到阶段边界；
`LogicalRepWorker.relstate` 让当前 sync/apply worker 协调；
`should_apply_changes_for_rel()` 阻止主 apply 提前处理未 READY 表；
`CATCHUP target -> SYNCDONE -> READY` 防止交接点重叠或漏 apply。

ERROR 路径也围绕 origin 收口。
`start_apply()` 的 `PG_CATCH` 会调用 `replorigin_xact_clear(true)`，
再 `AbortOutOfAnyTransaction()` 并上报 subscription error。
`InitializeLogRepWorker()` 还注册 `before_shmem_exit(on_exit_clear_xact_state, ...)`，
避免 worker 退出时错误确认未完成事务的 origin progress。

## 16. 错误路径、资源成本与传播

worker 启动失败可能来自两个资源池：
`out of logical replication worker slots` 通常指向 `max_logical_replication_workers`，
`out of background worker slots` 通常指向 `max_worker_processes`。
如果 worker slot 长时间 `in_use` 但没有 `proc`，
launcher 会在超过 `wal_receiver_timeout` 后清理旧 slot。
同一张表失败后，`launch_sync_worker()` 用 `wal_retrieve_retry_interval` 控制重启频率。

COPY 中失败时，catalog 多半停在 `DATASYNC`。
下一次启动会尝试 drop 旧 sync slot，然后重做 COPY。
COPY 成功后失败时，状态已经是 `FINISHEDCOPY`，
下一次从 per-table origin 恢复 `origin_startpos`，不再 COPY。
这正是 `DATASYNC` 与 `FINISHEDCOPY` 的恢复差异。

publisher 侧可能报远端表不存在、publication column list 不一致、
row filter 查询失败或 COPY OUT 启动失败。
subscriber 侧可能报 INSERT 权限不足、RLS enabled relation 不允许复制写入、
唯一约束冲突、触发器失败或本地 COPY 插入失败。
如果设置 `disable_on_error`，`DisableSubscriptionAndExit()` 会在新事务中禁用 subscription。

等待对端时，`wait_for_worker_state_change()` 和 `wait_for_table_state_change()` 都会检查对端 worker 是否仍存在，
并使用 latch timeout 避免永久睡眠。
sync slot 删除失败会报错，因为 silent failure 会让 permanent slot 继续保留 publisher WAL。
DDL remove 表映射时拿 `pg_subscription_rel` 的 `AccessExclusiveLock`，
是为了避免 apply worker 同时重新拉起同一张表的 sync worker。

成本随表数、行数和 WAL 量扩张。
每张正在同步的表可能占用一个 tablesync worker、一个 publisher sync slot、
一条 walreceiver/libpq connection、一个 per-table origin、一段 COPY FROM 插入工作和一段 catchup apply 工作。
并发受 `max_sync_workers_per_subscription`、`max_logical_replication_workers` 和 `max_worker_processes` 共同限制。

COPY 不是物理文件复制。
它走 `CopyFrom()`、约束、触发器、分区路由、index maintenance 和 table AM insert。
row filter 或 published generated column 会让远端命令变成 `COPY (SELECT ...) TO STDOUT`，
把更多 executor 成本放在 publisher。
状态虽然是 per-table，
但主 apply 在 `SYNCWAIT -> CATCHUP -> SYNCDONE` 交接时可能等待，
因此一张表的 catchup 卡住也会传播成 subscription-level apply lag。

## 17. 观测与诊断入口

subscriber 上先查 `pg_subscription_rel`：

```sql
SELECT s.subname, r.srrelid::regclass AS rel, r.srsubstate, r.srsublsn
FROM pg_subscription_rel r
JOIN pg_subscription s ON s.oid = r.srsubid
ORDER BY s.subname, rel;
```

这里能看到 `i/d/f/s/r`，
看不到 `SYNCWAIT` 和 `CATCHUP`，
因为后两者只在 shared worker slot 中。

再查 worker：

```sql
SELECT subname, worker_type, pid, relid::regclass AS rel,
       received_lsn, latest_end_lsn, last_msg_receipt_time
FROM pg_stat_subscription
ORDER BY subname, worker_type, rel;
```

`worker_type = 'table synchronization'` 且 `relid` 非 NULL，
表示 tablesync worker 正在运行。
`pg_stat_subscription` 能给 LSN 和 worker 类型，
但不暴露 `LogicalRepWorker.relstate`。

等待点看 `pg_stat_activity`：

```sql
SELECT pid, backend_type, application_name, wait_event_type, wait_event, state
FROM pg_stat_activity
WHERE backend_type LIKE 'logical replication%';
```

`LogicalSyncData` 对应 `copy_read_data()` 等 publisher 发送 COPY 数据。
`LogicalSyncStateChange` 对应 sync/apply worker 等对方推进状态。
`LogicalApplyMain` 表示 apply loop 正在等 replication message。

publisher 上看 sync slot：

```sql
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name LIKE 'pg\\_%\\_sync\\_%' ESCAPE '\\'
ORDER BY slot_name;
```

诊断时按状态分支：
`i` 通常看 worker 是否启动和 slot 资源；
`d` 看 COPY、网络、publisher executor 和 subscriber insert；
`f` 看 tablesync worker 是否在等 `CATCHUP` 或已进入 catchup；
`s` 比较 `srsublsn` 与 apply worker 进度，判断主 apply 是否到达交接点；
`r` 表示 table sync 完成，后续按普通 apply lag 或 relation apply error 分析。
注意 `srsublsn` 是远端 LSN，不能直接和 subscriber 本地 WAL LSN 比。

日志中常见起止点是：
`logical replication table synchronization worker ... has started`，
以及 `logical replication table synchronization worker ... has finished`。
如果 `SYNCDONE` 后 publisher 还遗留 `pg_<subid>_sync_<relid>_<sysid>` slot，
应重点检查 slot drop 报错、worker 退出日志和 DDL cleanup 是否发生。

## 18. 常见误区、课堂实验与讨论题

常见误区有六个。
第一，状态机不是只有 `INIT/DATASYNC/SYNCDONE/READY`，当前 master 还有 `FINISHEDCOPY`，
而 `SYNCWAIT/CATCHUP` 是不落 catalog 的 IPC 状态。
第二，当前 tablesync slot 不是 temporary slot，`walrcv_create_slot()` 的 `temporary` 参数是 `false`。
第三，`FINISHEDCOPY` 不表示表已接管，只表示 initial COPY 已完成。
第四，`SYNCDONE` 不等于 `READY`，前者是 sync worker 到达交接点，后者是主 apply 到达同一 LSN 并接管。
第五，`copy_data=true` 不会自动 TRUNCATE subscriber 表。
第六，`pg_stat_subscription.latest_end_lsn` 前进不代表所有表都 READY。

实验一：创建 `copy_data=true` 的 subscription，
轮询 `pg_subscription_rel` 和 `pg_stat_subscription`，
观察 `i -> d -> f -> s -> r`，
并解释为什么看不到 `w -> c`。
对应源码是 `pg_subscription_rel.h` 的状态定义和 `tablesync.c` 顶部状态机注释。

实验二：在测试源码中让 worker 写入 `SUBREL_STATE_FINISHEDCOPY` 后退出。
重启后应看到它不再 COPY，
而是走 `LogicalRepSyncTableStart()` 的 `FINISHEDCOPY` 分支，
从 per-table origin 读取 progress 后进入 `SYNCWAIT/CATCHUP`。

实验三：在大表或限速网络场景下联查 `pg_stat_subscription` 与 `pg_stat_activity`。
如果 wait event 是 `LogicalSyncData`，
回到 `copy_read_data()` 和 `walrcv_receive()`。
如果是 `LogicalSyncStateChange`，
回到 `wait_for_worker_state_change()` 或 `wait_for_table_state_change()`。

讨论题：
为什么 `FINISHEDCOPY` 必须持久化而 `SYNCWAIT` 不需要？
为什么 `SYNCDONE` 后还要由主 apply 写 `READY`？
为什么 `should_apply_changes_for_rel()` 要比较 `statelsn <= remote_final_lsn`？
为什么 `SNAPSHOT 'use'` 要求远端 read-only repeatable read？
`copy_data=false` 把哪些一致性责任交给了用户？
parallel apply worker 为什么不能直接复用 leader apply 的非 READY 判断？

## 19. 本节小结

tablesync 的核心链路是：

```text
DDL 写 INIT
  -> apply worker 拉起 tablesync worker
  -> tablesync 写 DATASYNC
  -> publisher 上用 SNAPSHOT 'use' 创建 permanent sync slot
  -> COPY 初始数据
  -> 写 FINISHEDCOPY
  -> shared memory 进入 SYNCWAIT
  -> apply worker 写 CATCHUP target LSN
  -> tablesync replay 到 target
  -> 写 SYNCDONE 并 drop sync slot
  -> apply worker 自己到达 SYNCDONE LSN
  -> drop per-table origin
  -> 写 READY
```

核心状态边界是 catalog 中的 `INIT/DATASYNC/FINISHEDCOPY/SYNCDONE/READY`
和 shared worker slot 中的 `SYNCWAIT/CATCHUP/UNKNOWN`。
核心 ownership 是 `pg_subscription_rel` 由 DDL 创建、tablesync/apply 更新、DDL/drop 清理；
tablesync slot 由 sync worker 创建并在 `SYNCDONE` 后显式删除；
per-table origin 由 sync worker 创建和推进，并由 sync worker、apply worker 或 DDL 兜底清理。

可观测入口是 `pg_subscription_rel`、`pg_stat_subscription`、`pg_stat_activity` wait event、
publisher `pg_replication_slots` 和 server log。
不可直接观测的是 `SYNCWAIT/CATCHUP` 的实时 shared memory 值和每个 change 的 `should_apply_changes_for_rel()` 决策。

可迁移的系统规律是：
当系统要合并 snapshot bulk load 和 streaming delta 时，
不要用一个 finished 布尔值表达完成度；
要拆出 bulk 是否可重做、delta 起点、catchup 责任方、handoff LSN、接管确认方和 cleanup 顺序。
