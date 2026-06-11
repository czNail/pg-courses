# PostgreSQL apply worker 消费 logical replication protocol 并应用事务
## 课程定位
前置知识：
已经理解 logical slot 如何保护 `restart_lsn`、`catalog_xmin` 和 `confirmed_flush`。
已经理解 `pgoutput` 如何把 `ReorderBufferChange` 编码成 logical replication protocol。
已经理解 relation schema、replica identity 和 `RELATION` message 是 subscriber 定位本地行的前提。
本节唯一主问题：
```text
apply worker 如何连接 publisher、消费 logical replication protocol，
并把远端事务按 commit 顺序应用到本地表？
```
核心矛盾：
publisher 发来的是一条网络协议流。
subscriber 要落地的是本地事务、约束、索引、触发器、replication origin 和 WAL flush 进度。
两者不能简单一一对应。
一个远端事务在协议里可能是：
```text
BEGIN
  RELATION / TYPE
  INSERT / UPDATE / DELETE / TRUNCATE
COMMIT
```
也可能因为大事务 streaming 被拆成：
```text
STREAM START
  RELATION / TYPE
  INSERT / UPDATE / DELETE ...
STREAM STOP
...
STREAM COMMIT
```
apply worker 必须做到：
```text
按 publisher 输出顺序消费协议
  -> 按 relation sync 状态决定是否应用
  -> 把 remote tuple 转成本地 executor slot
  -> 用本地事务提交
  -> 只在本地 commit WAL 已 flush 后反馈远端 flush LSN
```
学完后应能判断：
为什么 `received_lsn` 前进不等于 remote slot 已经可以确认。
为什么 `RELATION` 消息只是更新 relation map，不马上验证本地 schema。
为什么 `UPDATE` / `DELETE` 需要本地 replica identity 或可用索引。
为什么 streamed transaction 可以提前收到行变更，但普通 `streaming = on` 直到 `STREAM COMMIT` 才真正应用到表。
为什么 apply worker 出错时必须清空 `replorigin_xact_state`，否则会把未完整应用的远端事务伪装成已应用。
本课基于本地源码：
```text
/home/nail/postgres
master commit 0e1f1ed157e90741e12a3715909e1b2d71ff9344
```
## 1. 本节在总主线中的位置
前几节站在 publisher 和 output plugin 一侧。
它们回答的是：
```text
WAL record 如何变成 logical change？
pgoutput 如何写 BEGIN / RELATION / UPDATE / COMMIT 等协议消息？
relation schema 和 replica identity 如何进入协议？
```
本节换到 subscriber 一侧。
我们只跟一条 apply 主链路：
```text
logical replication launcher
  -> logicalrep_worker_launch()
  -> ApplyWorkerMain()
  -> InitializeLogRepWorker()
  -> run_apply_worker()
  -> walrcv_connect()
  -> walrcv_startstreaming()
  -> LogicalRepApplyLoop()
  -> apply_dispatch()
  -> local executor
  -> CommitTransactionCommand()
  -> send_feedback()
```
这个链路刻意不展开冲突检测的全部细节。
也不展开 sequence sync、two phase apply 和 publication DDL 的完整规则。
它们都会出现，但只作为“远端事务如何安全落地”的边界。
本节的 runtime truth 是：
```text
apply worker 收到远端 WALData 消息
  != 远端事务已经本地提交
  != 本地 commit WAL 已经 flush
  != publisher slot 已确认 flush LSN
```
真正让 publisher 可以推进的是 `send_feedback()` 发回的 flush 位置。
这个 flush 位置来自本地 `XactLastCommitEnd` 与远端 commit end LSN 的映射。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
apply worker 作为 background worker 启动后读取 pg_subscription，
用 libpqwalreceiver 连接 publisher 并发送 START_REPLICATION SLOT ... LOGICAL；
LogicalRepApplyLoop 从 COPY BOTH 流中读取 WALData / Keepalive / status 消息；
apply_dispatch 按 logical protocol message type 分派；
BEGIN 记录远端事务边界，DML 用 relation map 转成本地 slot 并调用 executor，
COMMIT 设置 replication origin 后提交本地事务；
send_feedback 只把本地已 flush 的 commit 对应的远端 LSN 回报给 publisher。
```
tension 在这里：
```text
远端协议流是顺序字节流
  vs
本地 apply 是事务系统、executor、catalog cache、relation sync 和 WAL flush 的组合结果
```
如果 subscriber 只按收到位置回 ACK，会丢事务。
场景很直接：
```text
apply worker 收到 COMMIT message
  -> 已经更新 received_lsn
  -> 本地 INSERT 还没 commit 或 commit WAL 还没 flush
  -> 进程 crash
  -> publisher 以为 subscriber 已 flush
  -> 这个事务不会再发
```
PostgreSQL 避开这个问题的方法是把进度拆成三层：
| 层次 | 状态 | 含义 |
| --- | --- | --- |
| receive | `last_received` / `received_lsn` | apply worker 已从网络读到的位置。 |
| write/apply | `lsn_mapping` 中最后一个 remote end | 本地已执行并提交过哪些远端事务。 |
| flush | `GetFlushRecPtr()` 覆盖的 `XactLastCommitEnd` | 本地 commit WAL 已经落盘到哪个位置。 |
这也是为什么 logical apply 的主问题不是“如何解析协议”。
解析协议只是入口。
真正的内核问题是：
```text
什么时候可以把远端 commit 视为在本地可靠完成？
```
## 3. 核心文件分工与阅读顺序
阅读顺序按状态推进，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/replication/logical/launcher.c` | `logicalrep_worker_launch()` 如何分配 `LogicalRepWorker` slot 并注册 `ApplyWorkerMain`。 |
| 2 | `src/include/replication/logicalworker.h` | apply / tablesync / parallel apply worker 入口声明。 |
| 3 | `src/backend/replication/logical/worker.c` | `ApplyWorkerMain()`、`InitializeLogRepWorker()`、`run_apply_worker()`、`LogicalRepApplyLoop()`、`apply_dispatch()`。 |
| 4 | `src/include/replication/walreceiver.h` | `WalRcvStreamOptions` 和 `walrcv_*` 函数表。 |
| 5 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | `libpqrcv_startstreaming()` 如何拼 `START_REPLICATION SLOT ... LOGICAL`。 |
| 6 | `src/include/replication/logicalproto.h` | `LogicalRepMsgType`、`LogicalRepTupleData`、`LogicalRepRelation`、commit/begin 数据结构。 |
| 7 | `src/backend/replication/logical/proto.c` | `logicalrep_read_*()` 如何解析 protocol payload。 |
| 8 | `src/backend/replication/logical/relation.c` | `logicalrep_relmap_update()`、`logicalrep_rel_open()`、relation sync cache、schema/attr map。 |
| 9 | `src/include/replication/logicalrelation.h` | `LogicalRepRelMapEntry` 字段语义。 |
| 10 | `src/backend/executor/execReplication.c` | `ExecSimpleRelationInsert()` / `Update()` / `Delete()` 的本地 executor 边界。 |
| 11 | `src/backend/replication/logical/tablesync.c` | 初始表同步状态机如何影响 apply 是否跳过 relation。 |
| 12 | `src/backend/access/transam/xact.c` | `StartTransactionCommand()`、`CommitTransactionCommand()`、`replorigin_session_advance()`、`XactLastCommitEnd`。 |
推荐顺序是入口、连接、主循环、DML handler、commit/feedback，始终围绕“远端 commit 何时成为本地 durable 进度”读。
## 4. 关键状态：worker identity、subscription、origin、relation map
### 4.1 `LogicalRepWorker`: shared worker slot
`LogicalRepWorker` 的定义在 `launcher.c` 相关 shared memory 中使用。
当前 worker 的 backend-local 指针是：
```text
MyLogicalRepWorker
```
这个 slot 保存 worker 的共享身份：
| 字段 | 语义 |
| --- | --- |
| `type` | `WORKERTYPE_APPLY`、`TABLESYNC`、`PARALLEL_APPLY` 等。 |
| `in_use` | launcher 是否已分配这个 slot。 |
| `proc` | worker attach 后写入的 `PGPROC`。 |
| `dbid` / `userid` / `subid` | worker 要连接哪个数据库、用哪个用户、服务哪个 subscription。 |
| `relid` | tablesync worker 的目标 relation；leader apply worker 为 `InvalidOid`。 |
| `relstate` / `relstate_lsn` | tablesync 和 apply 协作时的 relation state。 |
| `stream_fileset` | streaming transaction 序列化文件集合。 |
| `leader_pid` | parallel apply worker 指向 leader apply worker。 |
| `last_lsn` / `reply_lsn` / times | `pg_stat_subscription` 的运行时进度来源。 |
`logicalrep_worker_launch()` 在 `LogicalRepWorkerLock` 下找到空 slot。
然后注册 dynamic background worker。
对 leader apply worker，它把 `bgw_function_name` 设置为：
```text
ApplyWorkerMain
```
`ApplyWorkerMain()` 启动后调用 `logicalrep_worker_attach()`。
attach 成功后 `MyLogicalRepWorker->proc = MyProc`。
退出时 `logicalrep_worker_onexit()` 会断开远端连接、清理 worker slot、删除 streaming fileset，并唤醒 launcher。
### 4.2 `MySubscription`: worker 的本地 catalog 镜像
`InitializeLogRepWorker()` 做数据库连接和 subscription 初始化。
关键动作是：
```text
SetConfigOption("session_replication_role", "replica", ...)
BackgroundWorkerInitializeConnectionByOid(dbid, userid, ...)
SetConfigOption("search_path", "", ...)
ApplyContext = AllocSetContextCreate(...)
StartTransactionCommand()
LockSharedObject(SubscriptionRelationId, subid, ...)
MySubscription = GetSubscription(subid, true, true)
CommitTransactionCommand()
```
`session_replication_role = replica` 不是装饰。
它影响触发器和规则执行语义。
`search_path = ""` 也不是装饰。
apply worker 会执行类型输入函数、默认值、索引表达式、触发器等用户代码边界。
空 search path 用来降低被恶意对象重定向的风险。
subscription 不是读一次就永远有效；`InitializeLogRepWorker()` 注册 subscription、foreign server、user mapping、FDW、role 和 subscription-relmap 相关 syscache callback，主循环空闲时会调用 `maybe_reread_subscription()`。
如果 subscription 被禁用、删除、参数变化或 owner 权限变化，worker 可能退出或重启。
### 4.3 replication origin: 远端进度的本地事务状态
`run_apply_worker()` 为 subscription 创建 origin 名：
```text
ReplicationOriginNameForLogicalRep(subid, InvalidOid, originname, ...)
```
leader apply worker 的 origin 形如：
```text
pg_<subid>
```
tablesync worker 的 origin 形如：
```text
pg_<subid>_<relid>
```
然后它执行：
```text
StartTransactionCommand()
originid = replorigin_by_name(originname, true)
if invalid:
  originid = replorigin_create(originname)
replorigin_session_setup(originid, 0)
replorigin_xact_state.origin = originid
origin_startpos = replorigin_session_get_progress(false)
CommitTransactionCommand()
```
这个 `origin_startpos` 就是 `START_REPLICATION` 的起点。
commit 时 `apply_handle_commit_internal()` 会设置：
```text
replorigin_xact_state.origin_lsn = commit_data->end_lsn
replorigin_xact_state.origin_timestamp = commit_data->committime
CommitTransactionCommand()
```
`xact.c` 的 commit path 看到 `replorigin_xact_state.origin` 有效后，会调用：
```text
replorigin_session_advance(origin_lsn, XactLastRecEnd)
```
所以 origin progress 跟本地事务 commit 绑定。
这正是 ERROR path 必须清理 `replorigin_xact_state` 的原因。
如果一个未完整应用的事务把 origin advance 留下来，下次启动就会从错误 LSN 开始。
### 4.4 relation map: remote relation id 到 local relation
protocol 里的 DML 不带 schema/name。
它只带 remote relation id。
subscriber 用 `RELATION` message 建立映射：
```text
apply_handle_relation()
  -> logicalrep_read_rel()
  -> logicalrep_relmap_update()
  -> logicalrep_partmap_reset_relmap()
```
`LogicalRepRelMapEntry` 的关键字段：
| 字段 | 语义 |
| --- | --- |
| `remoterel.remoteid` | protocol 中 DML 引用的 relation id。 |
| `remoterel.nspname` / `relname` | publisher 发来的 relation 名。 |
| `remoterel.attnames` / `atttyps` / `attkeys` | remote column schema 和 replica identity key。 |
| `localrelvalid` | 派生的本地映射是否仍有效。 |
| `localreloid` / `localrel` | subscriber 上的目标 relation。 |
| `attrmap` | local attribute number 到 remote attribute number 的映射。 |
| `updatable` | UPDATE / DELETE 是否有足够定位信息。 |
| `localindexoid` | 定位本地行使用的 index，可能为 `InvalidOid`。 |
| `state` / `statelsn` | `pg_subscription_rel` 中的同步状态。 |
`logicalrep_relmap_update()` 只缓存远端 schema。
它不马上打开本地 relation。
真正验证发生在第一条 change 到来时：
```text
logicalrep_rel_open(remoteid, lockmode)
  -> 若本地 mapping 无效，按 namespace/name 找 local relation
  -> 检查 relkind 是否兼容
  -> 建 attrmap
  -> 检查 missing / generated column
  -> logicalrep_rel_mark_updatable()
  -> FindLogicalRepLocalIndex()
  -> 读取 subscription relation state
```
这种延迟验证减少锁持有和无用工作。
代价是 schema mismatch 通常在真正 apply 某条 change 时才暴露。
## 5. 连接 publisher 与 START_REPLICATION
`run_apply_worker()` 是 leader apply worker 的连接主入口。
它先准备 origin，再连接 publisher：
```text
LogRepWorkerWalRcvConn =
  walrcv_connect(MySubscription->conninfo,
                 true,
                 true,
                 must_use_password,
                 MySubscription->name,
                 &err)
```
`must_use_password` 来自：
```text
MySubscription->passwordrequired && !MySubscription->ownersuperuser
```
连接失败会 ERROR：
```text
apply worker for subscription "... " could not connect to the publisher
```
连接后调用：
```text
walrcv_identify_system()
```
apply worker 不直接使用结果，但这会让 upstream 做必要初始化。
然后 `set_stream_options()` 把 subscription 配置转成 `WalRcvStreamOptions`：
```text
options->logical = true
options->startpoint = origin_startpos
options->slotname = MySubscription->slotname
options->proto.logical.proto_version = ...
options->proto.logical.publication_names = MySubscription->publications
options->proto.logical.binary = MySubscription->binary
options->proto.logical.streaming_str = "parallel" / "on" / NULL
options->proto.logical.twophase = false initially
options->proto.logical.origin = MySubscription->origin
```
protocol version 按 publisher 版本降级：
```text
>= 16: LOGICALREP_PROTO_STREAM_PARALLEL_VERSION_NUM
>= 15: LOGICALREP_PROTO_TWOPHASE_VERSION_NUM
>= 14: LOGICALREP_PROTO_STREAM_VERSION_NUM
else:  LOGICALREP_PROTO_VERSION_NUM
```
`libpqrcv_startstreaming()` 最终拼命令：
```text
START_REPLICATION SLOT "<slot>" LOGICAL <lsn>
  (proto_version '<n>',
   streaming 'parallel' | 'on',
   two_phase 'on',
   origin '<origin>',
   publication_names '<pubs>',
   binary 'true')
```
具体 option 有版本保护：`binary` 只发给 14+，`two_phase` 只发给 15+，`origin` 只发给 16+。
这里的边界是：
```text
apply worker 是 pgoutput 的客户端；
它不是直接读 publisher WAL。
```
publisher 侧仍然是 walsender + output plugin。
subscriber 侧只是用 `walrcv_*` 函数表复用 libpqwalreceiver。
## 6. 主流程源码 walkthrough
### 6.1 launcher 到 worker attach
launcher 主循环在 `ApplyLauncherMain()`。
它扫描 enabled subscriptions：
```text
get_subscription_list()
  -> 对每个 enabled subscription
     -> logicalrep_worker_find(WORKERTYPE_APPLY, subid, InvalidOid, false)
     -> 若没有 running worker
        -> 按 wal_retrieve_retry_interval 限速
        -> logicalrep_worker_launch(WORKERTYPE_APPLY, ...)
```
`logicalrep_worker_launch()` 先占用 shared worker slot。
然后注册 dynamic background worker：
```text
bgw_library_name = "postgres"
bgw_function_name = "ApplyWorkerMain"
bgw_type = "logical replication apply worker"
bgw_main_arg = worker slot number
```
worker 进程进入：
```text
ApplyWorkerMain(main_arg)
  -> SetupApplyOrSyncWorker(worker_slot)
     -> logicalrep_worker_attach(worker_slot)
     -> load_file("libpqwalreceiver", false)
     -> InitializeLogRepWorker()
  -> run_apply_worker()
```
一旦 attach，worker slot 里的 `proc` 指向当前 `MyProc`。
其它进程可以通过 `LogicalRepWorkerLock` 找到它并 `SetLatch()`。
### 6.2 apply loop 接收 COPY BOTH 消息
`LogicalRepApplyLoop(last_received)` 创建两个 memory context：
```text
ApplyMessageContext:
  每个 replication protocol message 后 reset。
LogicalStreamingContext:
  streaming transaction 每个 stream stop 后 reset。
```
然后进入无限循环：
```text
walrcv_receive(conn, &buf, &fd)
  -> 如果 len > 0，处理所有已到达数据
  -> 如果 len == 0，等待 socket/latch/timeout
  -> 如果 len < 0，stream ended
```
每个网络包先读外层 replication protocol byte：
```text
PqReplMsg_WALData
PqReplMsg_Keepalive
PqReplMsg_PrimaryStatusUpdate
```
`WALData` 里再包含 logical replication protocol message。
处理 `WALData`：
```text
start_lsn = pq_getmsgint64()
end_lsn = pq_getmsgint64()
send_time = pq_getmsgint64()
last_received = max(last_received, start_lsn, end_lsn)
UpdateWorkerStats(last_received, send_time, false)
apply_dispatch(&s)
```
处理 `Keepalive`：
```text
end_lsn
timestamp
reply_requested
last_received = max(last_received, end_lsn)
send_feedback(last_received, reply_requested, false)
UpdateWorkerStats(last_received, timestamp, true)
```
处理 `PrimaryStatusUpdate` 主要服务 retain dead tuples / conflict detection。
本节只需要知道它不会直接 apply 行变更。
主循环每次处理完可用消息都会：
```text
send_feedback(last_received, false, false)
MemoryContextReset(ApplyMessageContext)
```
空闲且不在远端事务/streaming transaction 时，还会：
```text
AcceptInvalidationMessages()
maybe_reread_subscription()
ProcessSyncingRelations(last_received)
```
这解释了一个现场现象：
```text
subscription DDL 或 pg_subscription_rel 状态变化，不一定在正在 apply 的事务中立刻生效；
空闲点、commit 点和 invalidation 处理点才是常见收敛位置。
```
### 6.3 logical protocol dispatcher
`apply_dispatch()` 读 logical message type：
```text
action = pq_getmsgbyte(s)
```
当前源码中的主要分支：
| message | handler | 本节语义 |
| --- | --- | --- |
| `LOGICAL_REP_MSG_BEGIN` | `apply_handle_begin()` | 记录远端事务 final LSN，进入 running 状态。 |
| `LOGICAL_REP_MSG_COMMIT` | `apply_handle_commit()` | 校验 commit LSN，提交本地事务，推进 origin 和 flush mapping。 |
| `LOGICAL_REP_MSG_INSERT` | `apply_handle_insert()` | remote tuple -> local slot -> `ExecSimpleRelationInsert()`。 |
| `LOGICAL_REP_MSG_UPDATE` | `apply_handle_update()` | 定位本地旧行 -> 修改 slot -> `ExecSimpleRelationUpdate()`。 |
| `LOGICAL_REP_MSG_DELETE` | `apply_handle_delete()` | 定位本地旧行 -> `ExecSimpleRelationDelete()`。 |
| `LOGICAL_REP_MSG_TRUNCATE` | `apply_handle_truncate()` | 打开所有目标 relation，调用 `ExecuteTruncateGuts()`。 |
| `LOGICAL_REP_MSG_RELATION` | `apply_handle_relation()` | 更新 relation map cache。 |
| `LOGICAL_REP_MSG_TYPE` | `apply_handle_type()` | 读出后丢弃。 |
| `LOGICAL_REP_MSG_ORIGIN` | `apply_handle_origin()` | 只校验出现位置，当前不支持多 origin tracking。 |
| `LOGICAL_REP_MSG_MESSAGE` | 无实际 apply | logical replication 目前不使用 generic logical message。 |
| `STREAM_*` | `apply_handle_stream_*()` | streaming transaction 分块、序列化、parallel apply 和最终 commit。 |
未知 message type 会 ERROR：
```text
invalid logical replication message type
```
`apply_dispatch()` 还维护 `apply_error_callback_arg.command`。
任何后续 ERROR 都能在 errcontext 里带出 message type。
### 6.4 BEGIN 并不立即开启本地事务
`apply_handle_begin()` 做的事很少：
```text
logicalrep_read_begin()
set_apply_error_context_xact(xid, final_lsn)
remote_final_lsn = begin_data.final_lsn
maybe_start_skipping_changes(begin_data.final_lsn)
in_remote_transaction = true
pgstat_report_activity(STATE_RUNNING, NULL)
```
它没有调用 `StartTransactionCommand()`。
本地事务在第一条实际 replication step 到来时才启动：
```text
begin_replication_step()
  -> SetCurrentStatementStartTimestamp()
  -> if !IsTransactionState():
       StartTransactionCommand()
       maybe_reread_subscription()
  -> PushActiveSnapshot(GetTransactionSnapshot())
  -> MemoryContextSwitchTo(ApplyMessageContext)
```
每条 INSERT/UPDATE/DELETE/TRUNCATE 后调用：
```text
end_replication_step()
  -> PopActiveSnapshot()
  -> CommandCounterIncrement()
```
所以远端事务边界和本地 transaction state 不是同一个瞬间开始。
设计原因很实际：
```text
空事务、被 skip 的事务、只有 metadata 的事务，不必提前打开本地事务。
```
但是一旦第一条 DML 开始，本地事务会跨越远端事务内多条 change。
直到 `COMMIT` message 才统一提交。
### 6.5 INSERT 如何落到本地表
`apply_handle_insert()` 的路径：
```text
if is_skipping_changes() or handle_streamed_transaction(...):
  return
begin_replication_step()
relid = logicalrep_read_insert(s, &newtup)
rel = logicalrep_rel_open(relid, RowExclusiveLock)
if !should_apply_changes_for_rel(rel):
  close and return
switch to table owner if needed
edata = create_edata_for_relation(rel)
remoteslot = ExecInitExtraTupleSlot(...)
slot_store_data(remoteslot, rel, &newtup)
slot_fill_defaults(rel, estate, remoteslot)
ExecOpenIndices()
apply_handle_insert_internal()
ExecCloseIndices()
finish_edata()
logicalrep_rel_close(rel, NoLock)
end_replication_step()
```
`slot_store_data()` 是协议 tuple 到本地 slot 的关键边界。
它按 `rel->attrmap` 遍历本地 columns。
如果 remote value 是 text：
```text
getTypeInputInfo()
OidInputFunctionCall()
```
如果 remote value 是 binary：
```text
getTypeBinaryInputInfo()
OidReceiveFunctionCall()
```
如果 binary receive function 没吃完整个 buffer，会 ERROR：
```text
incorrect binary data format in logical replication column
```
`apply_handle_insert_internal()` 最终调用：
```text
TargetPrivilegesCheck(..., ACL_INSERT)
ExecSimpleRelationInsert(relinfo, estate, remoteslot)
```
`ExecSimpleRelationInsert()` 在 `execReplication.c` 中继续做：
```text
CheckCmdReplicaIdentity(rel, CMD_INSERT)
BEFORE ROW INSERT triggers
stored generated columns
constraints
partition check
simple_table_tuple_insert()
ExecInsertIndexTuples()
conflict reporting if needed
AFTER ROW INSERT triggers
```
这说明 logical apply 不是绕过 executor 直接写 heap。
它使用简化 executor path，但仍进入本地约束、索引、触发器和 WAL 机制。
### 6.6 UPDATE / DELETE 如何定位本地行
`apply_handle_update()` 先读：
```text
logicalrep_read_update(s, &has_oldtup, &oldtup, &newtup)
```
protocol 中 old tuple 可能是 `K` old key，也可能是 `O` old full tuple。
如果没有 old tuple，则用 new tuple 里的 replica identity 列作为搜索 tuple。
本地 relation 打开后先检查：
```text
check_relation_updatable(rel)
```
如果 subscriber 没有足够 replica identity，而 publisher 也没有 `REPLICA IDENTITY FULL`，会 ERROR。
真正定位在：
```text
FindReplTupleInLocalRel(edata, localrel, remoterel, localindexoid, remoteslot, &localslot)
```
它优先走 index：
```text
RelationFindReplTupleByIndex(..., LockTupleExclusive, ...)
```
没有可用 index 且 remote 是 `REPLICA IDENTITY FULL` 时才走：
```text
RelationFindReplTupleSeq(..., LockTupleExclusive, ...)
```
UPDATE 找到本地 tuple 后：
```text
slot_modify_data(remoteslot, localslot, relmapentry, &newtup)
ExecSimpleRelationUpdate(relinfo, estate, &epqstate, localslot, remoteslot)
```
`ExecSimpleRelationUpdate()` 继续执行：
```text
CheckCmdReplicaIdentity(rel, CMD_UPDATE)
BEFORE ROW UPDATE triggers
stored generated columns
constraints
partition check
simple_table_tuple_update()
index maintenance
conflict reporting
AFTER ROW UPDATE triggers
```
DELETE 找到本地 tuple 后：
```text
ExecSimpleRelationDelete(relinfo, estate, &epqstate, localslot)
```
`ExecSimpleRelationDelete()` 做：
```text
CheckCmdReplicaIdentity(rel, CMD_DELETE)
BEFORE ROW DELETE triggers
simple_table_tuple_delete()
AFTER ROW DELETE triggers
```
如果 UPDATE / DELETE 找不到本地行，当前源码不是简单 ERROR。
它会报告 apply conflict：
```text
CT_UPDATE_MISSING
CT_UPDATE_DELETED
CT_DELETE_MISSING
```
具体统计进入 `pg_stat_subscription_stats`。
### 6.7 COMMIT 才把远端事务变成本地进度
`apply_handle_commit()` 先解析：
```text
logicalrep_read_commit(s, &commit_data)
```
然后校验：
```text
commit_data.commit_lsn == remote_final_lsn
```
不匹配就是 protocol violation。
真正提交在 `apply_handle_commit_internal()`。
如果本地已经打开事务：
```text
clear_subscription_skip_lsn(commit_data->commit_lsn)
replorigin_xact_state.origin_lsn = commit_data->end_lsn
replorigin_xact_state.origin_timestamp = commit_data->committime
CommitTransactionCommand()
pgstat_report_stat(false)
store_flush_position(commit_data->end_lsn, XactLastCommitEnd)
```
`xact.c` 的 commit path 会写本地 commit WAL。
如果有 replication origin：
```text
replorigin_session_advance(origin_lsn, XactLastRecEnd)
```
commit 完成后：
```text
XactLastCommitEnd = XactLastRecEnd
```
apply worker 把：
```text
remote end LSN -> local commit WAL end LSN
```
放进 `lsn_mapping`。
这一步才让后续 `send_feedback()` 有能力判断“哪个远端事务已经在本地 durable”。
如果远端事务没有打开本地事务，`apply_handle_commit_internal()` 只处理 invalidation 和 subscription reread。
最后：
```text
in_remote_transaction = false
```
## 7. relation sync cache 与 tablesync 边界
apply worker 不是所有收到的 DML 都立刻应用。
它先问：
```text
should_apply_changes_for_rel(rel)
```
对 leader apply worker：
```text
rel->state == SUBREL_STATE_READY
  或
rel->state == SUBREL_STATE_SYNCDONE
  且 rel->statelsn <= remote_final_lsn
```
这解决初始表复制期间的重复 apply 问题。
tablesync 的状态机在 `tablesync.c` 顶部注释中写得很清楚：
```text
INIT
  -> DATASYNC
  -> FINISHEDCOPY
  -> SYNCWAIT
  -> CATCHUP
  -> SYNCDONE
  -> READY
```
几个关键点：
```text
DATASYNC / FINISHEDCOPY / SYNCDONE / READY 持久化在 pg_subscription_rel。
SYNCWAIT / CATCHUP 只在 shared memory。
```
tablesync worker 先 COPY 初始数据。
然后创建自己的 tablesync slot 和 origin。
COPY 完成后，它进入 `SYNCWAIT`。
leader apply worker 在 `ProcessSyncingRelations()` 中看到后，把它切到 `CATCHUP`。
tablesync worker 随后也运行 `LogicalRepApplyLoop()`，只 apply 自己 relation 的 catchup change。
到达指定 LSN 后写 `SYNCDONE`。
leader apply worker 到达同一个位置后把 relation 标为 `READY`。
这解释了两个常见现象：
```text
新加表时 leader apply worker 可能暂时跳过该表 change。
pg_subscription_rel.srsubstate 还没 READY 时，
received_lsn 前进不代表该 relation 已完全由 leader apply。
```
## 8. streaming transaction apply
publisher 可能在事务 commit 前发送大事务分块。
subscriber 收到：
```text
STREAM START
  streamed INSERT / UPDATE / DELETE / RELATION / TYPE ...
STREAM STOP
```
之后某个时刻才收到：
```text
STREAM COMMIT
```
普通 `streaming = on` 的路径是序列化到文件。
`apply_handle_stream_start()` 设置：
```text
in_streamed_transaction = true
stream_xid = logicalrep_read_stream_start(...)
```
然后 `stream_start_internal()`：
```text
begin_replication_step()
if !stream_fileset:
  FileSetInit()
stream_open_file(subid, xid, first_segment)
subxact_info_read() if needed
end_replication_step()
```
后续 DML handler 进入 `handle_streamed_transaction()`。
在 `TRANS_LEADER_SERIALIZE` 下，它不会应用行变更。
它只写：
```text
stream_write_change(action, s)
```
on-disk record 格式是：
```text
length
action byte
message payload after xid
```
`STREAM STOP` 调用：
```text
subxact_info_write()
stream_close_file()
CommitTransactionCommand()
MemoryContextReset(LogicalStreamingContext)
```
注意这个 commit 提交的是“序列化文件操作”的本地事务。
它不是远端业务事务的 apply commit。
真正 apply 在 `STREAM COMMIT`：
```text
apply_handle_stream_commit()
  -> apply_spooled_messages(stream_fileset, xid, commit_lsn)
     -> begin_replication_step()
     -> stream_fd = BufFileOpenFileSet(...)
     -> in_remote_transaction = true
     -> remote_final_lsn = commit_lsn
     -> 循环读 file record
        -> apply_dispatch(&s2)
  -> apply_handle_commit_internal(&commit_data)
  -> stream_cleanup_files(subid, xid)
```
所以普通 streaming 的语义是：
```text
提前接收和落临时文件
  !=
提前把行变更提交到用户表
```
`streaming = parallel` 是另一条 slow path。
`set_stream_options()` 在 publisher 16+ 且 subscription streaming 为 parallel 时设置：
```text
streaming_str = "parallel"
MyLogicalRepWorker->parallel_apply = true
```
leader apply worker 会尝试 `pa_allocate_worker()`。
如果可以，就把 streamed change 发给 parallel apply worker。
如果发送失败或超时，会切到 `TRANS_LEADER_PARTIAL_SERIALIZE`。
课程的主线不用展开 `applyparallelworker.c` 的全部协议。
只要抓住边界：
```text
parallel apply 改变的是 streamed transaction 的执行并行度；
远端 transaction finish 仍由 STREAM COMMIT / STREAM PREPARE 协调；
leader apply worker 仍负责整体接收、worker 管理和反馈进度。
```
## 9. feedback flush LSN：为什么不能直接回报 received_lsn
`send_feedback(recvpos, force, requestReply)` 构造：
```text
PqReplMsg_StandbyStatusUpdate
  write position
  flush position
  apply position
  sendTime
  replyRequested
```
但当前源码里参数顺序容易让人误读。
它发送的是：
```text
pq_sendint64(reply_message, recvpos)   /* write */
pq_sendint64(reply_message, flushpos)  /* flush */
pq_sendint64(reply_message, writepos)  /* apply */
```
核心判断来自：
```text
get_flush_position(&writepos, &flushpos, &have_pending_txes)
```
`get_flush_position()` 遍历 `lsn_mapping`。
每个 `FlushPosition` 记录：
```text
local_end:
  本地 commit WAL end，也就是 XactLastCommitEnd。
remote_end:
  远端事务 end_lsn。
```
它读取本地 flush 位置：
```text
local_flush = GetFlushRecPtr(NULL)
```
然后只把满足：
```text
pos->local_end <= local_flush
```
的 remote end LSN 推进为 `flushpos`。
如果还有 pending transaction 没 flush，就保留它们。
如果没有 pending transactions：
```text
flushpos = writepos = recvpos
```
这对同步复制很重要。
空闲时没有未 flush 本地事务，subscriber 可以回报最新 received position。
但只要有本地 commit WAL 未 flush，它就不会把远端事务伪确认。
这也是诊断 replication lag 时最重要的边界：
```text
pg_stat_subscription.received_lsn:
  网络接收进度。
publisher slot confirmed_flush:
  受 subscriber feedback flush 影响。
两者之间可能卡在本地 WAL flush、synchronous_commit、fsync、apply transaction 或 worker error。
```
## 10. 正确性机制层次
apply worker 的正确性不是一个锁或一个 LSN 保证的。
它是多层机制叠加。
| 层次 | 机制 | 保证 |
| --- | --- | --- |
| remote ordering | publisher logical decoding / pgoutput | 普通事务按 commit 边界输出；streamed transaction 有 finish message。 |
| protocol framing | `proto.c` / `logicalproto.h` | 每条消息有明确 type、relation id、tuple role 和 commit LSN。 |
| worker identity | `LogicalRepWorker` + `PGPROC` | launcher、tablesync、parallel apply 可以定位和唤醒 worker。 |
| relation mapping | `LogicalRepRelMapEntry` | remote relation id 映射到 local relation、attrmap、index 和 sync state。 |
| transaction | `StartTransactionCommand()` / `CommitTransactionCommand()` | 多条 change 在本地事务中原子提交。 |
| origin | `replorigin_xact_state` | 本地 commit 与远端 end LSN 绑定，重启后从正确位置继续。 |
| executor | `ExecSimpleRelation*()` | 本地约束、索引、触发器、partition check、conflict reporting。 |
| durability | `XactLastCommitEnd` + `GetFlushRecPtr()` | 只有本地 commit WAL flush 后才反馈 flush。 |
| cleanup | memory context / resource owner / exit callback | message 内存、stream fileset、remote connection、worker slot 在 ERROR/exit 时收尾。 |
protocol 只表达远端发生了什么；subscriber correctness 还依赖本地 catalog、schema、replica identity、executor、transaction commit、origin 和 feedback。`COMMIT` message 只是远端结束通知，本地 durable 边界在 `CommitTransactionCommand()` 后的 `XactLastCommitEnd`，并且要等 `GetFlushRecPtr()` 覆盖它。
## 11. 错误路径 / 异常路径 / fallback
### 11.1 协议错误
`proto.c` 的 read 函数会检查 tuple role：`logicalrep_read_update()` 只接受 `K` / `O` / `N` 并要求最后有 `N` new tuple，`logicalrep_read_delete()` 只接受 `K` / `O`。`apply_handle_commit()` 还检查：
```text
commit_lsn == remote_final_lsn
```
这些错误通常是 `ERRCODE_PROTOCOL_VIOLATION`。
发生后 worker 退出，launcher 按 `wal_retrieve_retry_interval` 重启。
如果 subscription 设置了 `disable_on_error`，`start_apply()` 的 `PG_CATCH` 会调用：
```text
DisableSubscriptionAndExit()
```
### 11.2 schema mismatch 和类型输入错误
`RELATION` message 到来时不立即检查所有本地 schema。
第一条 DML 到来时 `logicalrep_rel_open()` 可能报：
```text
target relation does not exist
missing replicated column
incompatible generated column
system columns in REPLICA IDENTITY index
```
`slot_store_data()` / `slot_modify_data()` 可能在 type input/receive 阶段报：
```text
incorrect binary data format in logical replication column
```
这些错误的 errcontext 会带：
```text
origin name
message type
remote transaction id
finish LSN
target relation
remote column name
```
上下文来自 `apply_error_callback()`。
诊断时不要只看主错误。
errcontext 往往告诉你是哪条 protocol message、哪个 relation、哪列触发了问题。
### 11.3 UPDATE / DELETE 找不到本地行
不是所有找不到行都会 ERROR。
`apply_handle_update_internal()` / `apply_handle_delete_internal()` 找不到 local tuple 时会报告 conflict。
常见类型：
```text
CT_UPDATE_MISSING
CT_UPDATE_DELETED
CT_DELETE_MISSING
```
统计进入：
```text
pg_stat_subscription_stats
```
如果目标 relation 根本不可更新，则 `check_relation_updatable()` 会 ERROR。
这类 ERROR 通常说明：
```text
publisher 没发足够 old key
  或
subscriber 没有可用 replica identity / primary key / FULL fallback
  或
column list / row filter 与 replica identity 不兼容
```
### 11.4 timeout 和连接中断
`LogicalRepApplyLoop()` 空闲等待时用：
```text
WaitLatchOrSocket(..., WAIT_EVENT_LOGICAL_APPLY_MAIN)
```
如果超过 `wal_receiver_timeout` 没收到 publisher 消息，会 ERROR：
```text
terminating logical replication worker due to timeout
```
在 timeout 一半时，worker 会发带 `requestReply = true` 的 feedback ping。
如果 publisher stream 结束：
```text
data stream from publisher has ended
```
主循环退出后调用：
```text
walrcv_endstreaming()
```
worker 退出后 launcher 后续重启。
### 11.5 ERROR cleanup 和 origin 防丢失
`start_apply()` 包住 `LogicalRepApplyLoop()`：
```text
PG_TRY()
  LogicalRepApplyLoop(origin_startpos)
PG_CATCH()
  replorigin_xact_clear(true)
  if disableonerr:
    DisableSubscriptionAndExit()
  else:
    AbortOutOfAnyTransaction()
    pgstat_report_subscription_error()
    PG_RE_THROW()
```
`InitializeLogRepWorker()` 还注册：
```text
before_shmem_exit(on_exit_clear_xact_state, 0)
```
注释说得很直接：
shutdown 时如果不清理 transaction-level origin state，可能把 incomplete apply operation 的 origin advance 带到 abort 路径，造成事务丢失。
这就是 apply worker ERROR path 的核心不变量：
```text
未完整 commit 的远端事务，绝不能推进 replication origin。
```
## 12. 成本、资源与跨模块传播
apply worker 的成本不是只有网络。
它会沿这些维度扩张。
第一，tuple decode 和 type input 成本随列数、行数和数据类型扩张。
`slot_store_data()` 对每个非 dropped、已映射列调用 text input 或 binary receive。
binary 可以省部分文本解析，但类型 receive 仍是 per-column 调用。
第二，UPDATE / DELETE 定位成本取决于 replica identity。
有 `localindexoid` 时走 index lookup。
`REPLICA IDENTITY FULL` 且没有可用 index 时走 seq scan。
这种成本随表大小和变更行数相乘。
第三，本地 executor 成本来自约束、索引、触发器和 partition routing。
logical apply 不是物理 replay。
它会执行本地约束和触发器边界。
schema 设计不同会直接改变 subscriber apply 成本。
第四，streaming transaction 成本可能转移到临时文件。
`streaming = on` 用 `FileSet` / `BufFile` 序列化。
这降低内存压力，但增加临时 IO，并且最终 `STREAM COMMIT` 仍要重放全部 change。
第五，feedback 可能被本地 WAL flush 阻塞。
`lsn_mapping` 里 pending transaction 多时，`send_feedback()` 不能把 flushpos 提到最新 received LSN。
这会表现为 publisher 侧 slot lag 或同步复制等待。
跨模块传播也很清楚：
```text
publisher walsender / pgoutput
  -> libpqwalreceiver
  -> apply worker transaction / executor
  -> WAL flush
  -> replication origin
  -> feedback
  -> publisher slot confirmed_flush
```
任何一层慢，都会把“逻辑复制延迟”表现成一个整体症状。
## 13. 观测与诊断入口
先看 worker 是否存在：
```sql
SELECT subid, subname, worker_type, pid, leader_pid, relid,
       received_lsn, latest_end_lsn,
       last_msg_send_time, last_msg_receipt_time
FROM pg_stat_subscription;
```
`worker_type` 能区分：
```text
apply
table synchronization
parallel apply
```
`received_lsn` 只说明 subscriber 已接收。
不要把它解释成 durable flush。
再看错误和 conflict 计数：
```sql
SELECT *
FROM pg_stat_subscription_stats
WHERE subname = '...';
```
重点列包括：
```text
apply_error_count
sync_table_error_count
confl_insert_exists
confl_update_missing
confl_update_deleted
confl_delete_missing
```
再看 wait event：
```sql
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type LIKE 'logical replication%';
```
常见 wait event：
```text
LogicalApplyMain:
  apply worker 主循环等待 socket/latch/timeout。
LogicalSyncData:
  tablesync COPY 等 publisher 数据。
LogicalSyncStateChange:
  apply worker 和 tablesync worker 等状态推进。
LogicalApplySendData:
  leader apply worker 等待把数据发给 parallel apply worker。
applytransaction:
  等待远端 transaction apply lock。
```
看初始同步状态：
```sql
SELECT srrelid::regclass, srsubstate, srsublsn
FROM pg_subscription_rel
WHERE srsubid = '...'::regclass::oid;
```
`srsubstate` 的字符来自 `pg_subscription_rel.h`：
```text
i INIT
d DATASYNC
f FINISHEDCOPY
s SYNCDONE
r READY
```
`SYNCWAIT` 和 `CATCHUP` 不在 catalog 中持久化。
它们只在 shared worker state 中出现。
看 publisher 侧：
```sql
SELECT slot_name, confirmed_flush_lsn, restart_lsn, catalog_xmin
FROM pg_replication_slots
WHERE slot_name = '...';
```
如果 subscriber `received_lsn` 前进而 publisher `confirmed_flush_lsn` 不动，优先怀疑：
```text
本地事务还没 commit
本地 commit WAL 还没 flush
apply worker error 后重启
feedback 被 wal_receiver_status_interval 限制
network / keepalive / timeout
```
日志诊断要抓 errcontext。
apply worker 的错误上下文会写出：
```text
processing remote data for replication origin "pg_<subid>"
during message type "UPDATE"
for replication target relation "schema.table"
column "col"
in transaction <xid>, finished at <lsn>
```
这比单独的 SQLSTATE 更接近源码状态。
## 14. 常见误区
误区一：
```text
BEGIN message 会立刻开启本地事务。
```
不准确。
`apply_handle_begin()` 只记录远端事务状态。
本地事务通常由第一条 `begin_replication_step()` 开启。
误区二：
```text
RELATION message 到来就完成了 schema 校验。
```
不准确。
`apply_handle_relation()` 只更新 remote relation map。
本地 relation 打开、attrmap、missing column、updatable 和 local index 都在 `logicalrep_rel_open()` 时重建。
误区三：
```text
received_lsn 等于 publisher slot confirmed_flush_lsn。
```
不准确。
confirmed flush 要等 subscriber `send_feedback()` 回报 flush。
flush 又依赖本地 `XactLastCommitEnd` 是否被 `GetFlushRecPtr()` 覆盖。
误区四：
```text
streaming transaction 收到 chunk 就已经应用到用户表。
```
普通 `streaming = on` 下不是。
chunk 先序列化到 `FileSet` / `BufFile`。
`STREAM COMMIT` 时才重放并提交业务变更。
误区五：
```text
UPDATE / DELETE 找不到行一定是协议错误。
```
不一定。
当前源码会报告 apply conflict，并计入 subscription stats。
真正的 protocol error 是消息格式、LSN 边界或 tuple role 不符合协议。
误区六：
```text
apply worker 绕过本地约束和触发器。
```
不准确。
它走 `ExecSimpleRelation*()`，仍会执行本地约束、索引维护、row triggers、partition check 和权限/RLS 边界。
## 15. 课堂实验
### 实验一：跟一条普通 INSERT 事务
准备 publisher / subscriber 后，在 subscriber 侧给这些函数下断点：
```text
ApplyWorkerMain
run_apply_worker
LogicalRepApplyLoop
apply_dispatch
apply_handle_begin
apply_handle_insert
ExecSimpleRelationInsert
apply_handle_commit_internal
send_feedback
```
publisher 执行 `INSERT INTO t VALUES (1, 'a');`。观察 `BEGIN` 只设置 `remote_final_lsn`，`INSERT` 才进入 `StartTransactionCommand()`，`COMMIT` 设置 `replorigin_xact_state.origin_lsn`，`store_flush_position()` 记录 `remote_end -> XactLastCommitEnd`，后续 `send_feedback()` 才可能推进 flush。
### 实验二：把现象压回诊断链
制造一个 subscriber 缺列的 schema mismatch，观察 apply worker 日志中的 errcontext；再同时查询：
```sql
SELECT subname, worker_type, received_lsn, latest_end_lsn
FROM pg_stat_subscription;
SELECT slot_name, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = '...';
```
把三类状态分开解释：协议接收到哪里、哪条 relation/column 触发 ERROR、publisher slot 是否已经收到 subscriber 的 durable flush feedback。
## 16. 讨论题
1. 为什么 `apply_handle_begin()` 不直接调用 `StartTransactionCommand()`？
2. `RELATION` message 为什么只更新 relation map，而不是立即打开并校验本地表？
3. `received_lsn`、`remote end_lsn`、`XactLastCommitEnd`、publisher slot `confirmed_flush_lsn` 分别代表什么？
4. `UPDATE` 没有 old tuple 时，subscriber 为什么仍可能定位本地行？
5. `streaming = on` 为什么选择 `FileSet` / `BufFile`，而不是把大事务所有 change 留在内存？
6. ERROR path 为什么必须调用 `replorigin_xact_clear(true)`？
7. 哪些问题能从 `pg_stat_subscription` 直接看到，哪些只能从日志 errcontext 或 publisher slot 侧推断？
8. parallel apply 改变了 streamed transaction 的哪个成本维度？它没有改变哪个 commit / feedback 正确性边界？
## 17. 本节小结
本节的核心链路是：
```text
launcher 分配 LogicalRepWorker
  -> ApplyWorkerMain attach worker slot
  -> InitializeLogRepWorker 读取 subscription 并设置 replica session
  -> run_apply_worker 设置 replication origin 和 START_REPLICATION options
  -> LogicalRepApplyLoop 消费 COPY BOTH 消息
  -> apply_dispatch 分派 logical protocol
  -> DML 经 relation map 转成本地 slot
  -> ExecSimpleRelation* 写本地表
  -> COMMIT 设置 origin 并提交本地事务
  -> store_flush_position 建 remote/local LSN 映射
  -> send_feedback 只回报本地已 flush 的远端 commit
```
核心状态不是单个 LSN。
它是：
```text
worker slot
subscription catalog snapshot
replication origin
relation map cache
relation sync state
local transaction state
lsn_mapping
```
ownership 和 cleanup 的关键点：
```text
ApplyContext 持有 worker 长生命周期对象。
ApplyMessageContext 每条协议消息 reset。
LogicalStreamingContext 在 stream stop 后 reset。
FileSet / BufFile 承载 streamed transaction slow path。
before_shmem_exit 清理 transaction-level origin state。
logicalrep_worker_onexit 断开远端连接并清理 worker slot。
```
错误路径的核心不变量：
```text
未完整本地 commit 的远端事务，不能推进 replication origin，也不能反馈为 flush。
```
诊断时要把现象分层：
```text
网络是否收到：pg_stat_subscription.received_lsn / last_msg_receipt_time。
本地是否 apply：日志、conflict stats、wait event。
本地是否 durable：send_feedback 的 flush 由 GetFlushRecPtr 和 lsn_mapping 决定。
publisher 是否确认：pg_replication_slots.confirmed_flush_lsn。
relation 是否可应用：pg_subscription_rel 和 relation map/schema/replica identity。
```
可迁移的系统规律是：
```text
跨节点复制的 ACK 不能跟接收字节绑定；
它必须跟本地 recovery 后仍可证明的 durable state 绑定。
```
在 logical replication apply 中，这个 durable state 不是一个字段。
它是远端 commit end LSN、本地事务 commit WAL、replication origin advance 和 feedback flush LSN 的组合语义。
