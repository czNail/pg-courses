# PostgreSQL logical apply 冲突与 ERROR 边界
## 课程定位
前置知识：已经理解逻辑解码输出的 `BEGIN`、`RELATION`、`INSERT`、`UPDATE`、`DELETE`、`COMMIT` 消息，也知道 subscription apply worker 使用 replication origin 记录已经应用到哪个远端 LSN。
本节唯一主问题：
```text
目标端唯一约束、缺失行、权限、schema 不匹配或 replica identity 不足时，
apply worker 为什么必须把可回滚的修改收在远端事务边界内，
哪些冲突会变成 ERROR 并停止 worker，
哪些只记录 conflict diagnostics，
以及哪些问题必须由运维或应用语义解决？
```
核心矛盾：
```text
subscriber 必须按远端事务顺序应用变化
  vs
subscriber 本地 schema、约束、权限、触发器和已有数据可能已经偏离 publisher
```
PostgreSQL 的选择不是在 apply worker 内做业务合并。
它只做三件事：
1. 在一个本地事务中重放一个远端事务的数据修改。
2. 遇到无法保证语义等价的 ERROR 时 abort 本地事务并停止当前 worker。
3. 尽量把冲突类型、origin、远端 XID、finish LSN、relation 和列名放进日志与统计。
学完后应能判断：
```text
哪些 apply 问题是确定性错误，重启后还会在同一事务重复失败；
哪些 current master 中只 LOG conflict 并继续应用；
为什么 duplicate key 不能自动转成 UPSERT；
为什么 missing row 不能凭内核猜测是否该跳过；
为什么 replica identity 和 schema mismatch 必须先修配置或 DDL；
为什么 disable_on_error 只是停机保护，不是冲突解决器；
如何用日志、pg_stat_subscription、pg_stat_subscription_stats 和 origin LSN 定位停点。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前几节已经讲过 logical decoding 如何把 WAL 重组为事务、如何通过协议发送 relation metadata 和 tuple data。
本节站在 subscriber apply 侧看同一条链路的失败边界。
publisher 发来的不是一条条可独立解释的 SQL。
它发来的是远端事务中的行级变化。
subscriber 只知道：
```text
远端事务 XID
远端 finish LSN
message type
remote relation id
old tuple / new tuple
relation metadata
subscription 配置
当前 subscriber 本地状态
```
它不知道：
```text
应用业务想要 last-write-wins 还是 first-write-wins
唯一键冲突是否代表同一业务对象
本地删掉的行是故意补偿还是误操作
目标表新增的 NOT NULL 列应该填什么业务默认值
触发器副作用是否应该为复制流执行
RLS policy 对复制流应该如何解释
```
因此本节不把冲突讲成一个统一的“自动修复”问题。
我们只沿一条线：
```text
远端事务开始
  -> 本地事务按 message 应用
  -> tuple 定位和写入触发本地 executor 边界
  -> 有些冲突 LOG 后继续
  -> 有些错误 ERROR 后 abort
  -> origin 不前进
  -> worker 退出或禁用 subscription
  -> launcher 后续决定是否重启
  -> DBA 或应用根据语义修复
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
apply worker 把一个远端事务映射为一个本地事务；
每个 INSERT / UPDATE / DELETE 在 executor 中像普通 DML 一样检查本地权限、RLS、约束、触发器、索引和 replica identity；
如果抛出 ERROR，外层 start_apply() 清除本事务 origin 状态、abort 本地事务、记录 subscription error，然后让 worker 退出或禁用 subscription。
```
关键不变量：
```text
没有成功 commit 的远端事务，不能推进 subscriber 的 replication origin。
```
如果这条不变量被破坏，就会出现数据丢失：
```text
远端事务只应用了一半
  -> origin 被推进到事务 end_lsn
  -> publisher 认为 subscriber 已经消费
  -> worker 重启后从新的 origin 继续
  -> 失败事务的剩余变化永远不再发送
```
所以 apply ERROR 的第一层边界不是“赶紧继续复制”。
第一层边界是：
```text
把本地事务和 origin xact state 恢复到未应用该远端事务的状态。
```
这也是为什么 worker 看起来“停在事务边界”。
它不是停在某一行的持久半成品上。
它是在远端事务尚未本地 commit 前失败，随后整体 abort。
重启后会重新从 origin progress 之后读取，通常再次遇到同一个远端事务。
如果错误是 schema、权限、唯一约束或 replica identity 这种确定性条件，重启只会重复失败。
## 3. 当前源码中的真实分工
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/replication/logical/worker.c` | `ApplyWorkerMain()`、`LogicalRepApplyLoop()`、`apply_dispatch()`、`apply_handle_*()`、`start_apply()`、`DisableSubscriptionAndExit()`。 |
| 2 | `src/backend/executor/execReplication.c` | `ExecSimpleRelationInsert()`、`ExecSimpleRelationUpdate()`、`ExecSimpleRelationDelete()`、`CheckCmdReplicaIdentity()`、tuple 查找与唯一冲突报告。 |
| 3 | `src/backend/replication/logical/relation.c` | `logicalrep_relmap_update()`、`logicalrep_rel_open()`、`logicalrep_rel_mark_updatable()`、本地 relation 和远端 relation 的映射。 |
| 4 | `src/backend/replication/logical/proto.c` | `logicalrep_read_rel()`、`logicalrep_read_tuple()`、协议里有哪些信息，哪些没有。 |
| 5 | `src/backend/replication/logical/conflict.c` | `ReportApplyConflict()`、冲突类型、ERROR / LOG 级别、`pg_stat_subscription_stats` 计数。 |
| 6 | `src/backend/access/transam/xact.c` | `CommitTransactionCommand()`、`AbortOutOfAnyTransaction()`、ERROR 后本地事务如何收尾。 |
| 7 | `src/backend/commands/subscriptioncmds.c` | `disable_on_error`、`run_as_owner`、`ALTER SUBSCRIPTION ... SKIP` 的 catalog 配置。 |
| 8 | `src/backend/utils/activity/pgstat_subscription.c` | apply error 与 conflict 统计如何累加。 |
本节要保留一个 awkward truth：
```text
logical apply 的“冲突”不是都等价。
```
当前 master 中：
```text
duplicate key:
  ReportApplyConflict(..., ERROR, CT_INSERT_EXISTS / CT_UPDATE_EXISTS)
  -> 停止当前 worker
missing row for UPDATE / DELETE:
  ReportApplyConflict(..., LOG, CT_UPDATE_MISSING / CT_DELETE_MISSING)
  -> 记录 conflict
  -> 当前操作不修改目标行
  -> worker 继续
origin differs:
  ReportApplyConflict(..., LOG, CT_UPDATE_ORIGIN_DIFFERS / CT_DELETE_ORIGIN_DIFFERS)
  -> 记录 conflict
  -> UPDATE / DELETE 仍继续执行
schema / permission / RLS / replica identity / type input / constraint / trigger ERROR:
  ereport(ERROR)
  -> abort 当前远端事务
  -> worker 退出或禁用 subscription
```
不要把旧资料中“logical replication conflict 都会停住”的说法直接搬到当前源码。
也不要把当前 conflict diagnostics 误解成自动 conflict resolution。
LOG 级 conflict 只是“内核能描述这个偏差，并按当前实现继续”。
ERROR 级失败才是“当前事务无法继续保持可接受语义”。
## 4. 状态：apply worker 手里有什么
apply worker 的关键运行状态分散在多个层次。
### 4.1 远端事务状态
`worker.c` 中有几个本节必须记住的状态：
| 状态 | 语义 |
| --- | --- |
| `in_remote_transaction` | 当前是否正在处理远端非 streaming 事务。 |
| `remote_final_lsn` | `BEGIN` 消息携带的远端事务最终 LSN，用于和 `COMMIT` 校验。 |
| `apply_error_callback_arg.remote_xid` | 日志 error context 中的远端事务 XID。 |
| `apply_error_callback_arg.finish_lsn` | 日志 error context 中的远端事务 finish LSN。 |
| `apply_error_callback_arg.command` | 当前正在处理的 logical replication message type。 |
| `apply_error_callback_arg.rel` | 当前目标 relation，用于错误上下文。 |
| `apply_error_callback_arg.remote_attnum` | 当前列解析失败时的远端列号。 |
| `skip_xact_finish_lsn` | `ALTER SUBSCRIPTION ... SKIP` 命中的事务 finish LSN。 |
这些状态不是独立语义。
组合起来才回答：
```text
当前 ERROR 属于哪个远端事务？
属于哪个 message type？
属于哪个 target relation？
如果是列值解析，属于哪一列？
失败事务 finish LSN 是多少？
```
### 4.2 replication origin 状态
`run_apply_worker()` 为 subscription 建立 origin name：
```text
ReplicationOriginNameForLogicalRep(suboid, InvalidOid, ...)
replorigin_session_setup(originid, 0)
origin_startpos = replorigin_session_get_progress(false)
```
`apply_handle_commit_internal()` 只在本地事务准备提交时设置：
```text
replorigin_xact_state.origin_lsn = commit_data->end_lsn
replorigin_xact_state.origin_timestamp = commit_data->committime
CommitTransactionCommand()
store_flush_position(commit_data->end_lsn, XactLastCommitEnd)
```
这意味着 origin 进度是 commit 边界的状态。
行级 apply 过程中不应提前推进它。
`start_apply()` 的 `PG_CATCH()` 第一件关键事就是：
```text
replorigin_xact_clear(true)
```
它防止一个失败事务在 abort 或退出过程中误推进 origin。
### 4.3 relation map 状态
`LogicalRepRelMapEntry` 是 subscriber 侧把 remote relation id 映射到本地 relation 的缓存。
本节关注字段：
| 字段 | 语义 |
| --- | --- |
| `remoterel` | publisher 发来的 relation metadata，包括列名、远端列数、replica identity 标记。 |
| `localrelvalid` | 本地 relation 映射是否还有效，relcache invalidation 会让它重建。 |
| `localreloid` / `localrel` | subscriber 本地 relation OID 和打开的 relcache entry。 |
| `attrmap` | 本地列号到远端列号的映射，不是类型转换规则。 |
| `updatable` | 本地 identity 是否足以定位 UPDATE / DELETE 目标行。 |
| `localindexoid` | 用于 tuple 查找的本地 PK、RI 或 RI FULL 可用索引。 |
`logicalrep_read_attrs()` 会接收远端 `atttyps`。
但 relation map 阶段主要按列名建立 `attrmap`。
当前 apply 建 slot 时使用的是本地列类型的 input / receive 函数。
所以类型不匹配常常不是在 `logicalrep_rel_open()` 直接比较 OID 后报错。
它通常在 `slot_store_data()`、`slot_modify_data()`、约束检查或二进制解析阶段报错。
### 4.4 subscription 配置状态
`pg_subscription` 中和本节直接相关的字段：
| catalog 字段 | `Subscription` 字段 | 影响 |
| --- | --- | --- |
| `subdisableonerr` | `disableonerr` | worker ERROR 后是否自动 disable subscription。 |
| `subrunasowner` | `runasowner` | apply 动作以 subscription owner 还是 relation owner 执行。 |
| `subskiplsn` | `skiplsn` | 跳过一个 finish LSN 精确匹配的远端事务。 |
| `subretaindeadtuples` | `retaindeadtuples` | 是否保留 dead tuple 信息用于更细 conflict diagnostics。 |
| `submaxretention` | `maxretention` | dead tuple conflict 信息最多保留多久。 |
| `subretentionactive` | `retentionactive` | 当前是否仍在主动保留 conflict detection 信息。 |
`disable_on_error` 在当前源码真实存在。
默认值在 `parse_subscription_options()` 中是 `false`。
`CREATE SUBSCRIPTION ... WITH (disable_on_error = true)` 和 `ALTER SUBSCRIPTION ... SET (disable_on_error = true)` 都能设置它。
它不改变冲突语义。
它只改变 ERROR 后的 operational action：
```text
false:
  abort 当前事务
  记录 apply_error_count
  worker 抛出错误退出
  launcher 之后按 wal_retrieve_retry_interval 重启
true:
  输出原错误
  abort 当前事务
  记录 apply_error_count
  在新事务中 DisableSubscription()
  worker clean exit
  subscription 不再自动继续
```
## 5. 主流程源码 walkthrough：从 BEGIN 到 COMMIT
入口是 `ApplyWorkerMain()`：
```text
ApplyWorkerMain()
  -> SetupApplyOrSyncWorker()
     -> InitializeLogRepWorker()
  -> run_apply_worker()
     -> replication origin setup
     -> walrcv_startstreaming()
     -> start_apply(origin_startpos)
        -> LogicalRepApplyLoop(origin_startpos)
```
`InitializeLogRepWorker()` 有两个本节相关的配置动作。
第一，它把 session replication role 设为 replica：
```text
SetConfigOption("session_replication_role", "replica", ...)
```
这会影响触发器 firing。
普通 origin trigger 不会在 replica role 下触发。
`ENABLE REPLICA` 和 `ENABLE ALWAYS` 触发器仍可能触发。
第二，它根据 subscription owner 建立数据库连接，并读取 `MySubscription`。
之后每个 relation 操作还可能根据 `run_as_owner` 切换用户上下文。
### 5.1 接收消息
`LogicalRepApplyLoop()` 从 walreceiver connection 读消息。
读到 `PqReplMsg_WALData` 后：
```text
UpdateWorkerStats(last_received, send_time, false)
apply_dispatch(&s)
maybe_advance_nonremovable_xid(&rdt_data, false)
```
这个循环每处理完一个 replication protocol message，就 reset `ApplyMessageContext`。
但远端事务不因此提交。
事务提交只发生在 `COMMIT`、`PREPARE` 或 streaming finish 相关路径。
### 5.2 BEGIN
`apply_dispatch()` 遇到 `LOGICAL_REP_MSG_BEGIN`：
```text
apply_handle_begin()
  -> logicalrep_read_begin()
  -> set_apply_error_context_xact(xid, final_lsn)
  -> remote_final_lsn = begin_data.final_lsn
  -> maybe_start_skipping_changes(final_lsn)
  -> in_remote_transaction = true
```
这里还没有本地事务。
本地事务通常延迟到第一条需要执行的 replication step 才启动。
这是 `begin_replication_step()` 的工作。
### 5.3 一个 replication step
`begin_replication_step()` 的模型：
```text
SetCurrentStatementStartTimestamp()
if !IsTransactionState():
  StartTransactionCommand()
  maybe_reread_subscription()
PushActiveSnapshot(GetTransactionSnapshot())
MemoryContextSwitchTo(ApplyMessageContext)
```
`end_replication_step()` 的模型：
```text
PopActiveSnapshot()
CommandCounterIncrement()
```
它不会提交。
它只让同一个本地事务内前一步变化对后续步骤可见。
所以一个远端事务里多条 INSERT / UPDATE / DELETE，会共享同一个本地事务边界。
如果第 N 条变化 ERROR，前 N-1 条变化也会随 abort 回滚。
### 5.4 COMMIT
`apply_handle_commit()` 先检查协议一致性：
```text
commit_data.commit_lsn == remote_final_lsn
```
然后进入 `apply_handle_commit_internal()`。
如果当前存在本地事务：
```text
clear_subscription_skip_lsn(commit_data->commit_lsn)
replorigin_xact_state.origin_lsn = commit_data->end_lsn
replorigin_xact_state.origin_timestamp = commit_data->committime
CommitTransactionCommand()
pgstat_report_stat(false)
store_flush_position(commit_data->end_lsn, XactLastCommitEnd)
```
`store_flush_position()` 把远端 end LSN 和本地 WAL end 关联起来。
后续 feedback 只能报告已经本地 flush 的远端位置。
这里是 apply 正常路径的事务边界：
```text
本地数据修改 commit
  + origin progress commit
  + flush position tracking
```
任何 ERROR 发生在这个边界之前，都必须让这三者一起不生效。
## 6. INSERT：duplicate key 为什么是 ERROR
`apply_handle_insert()` 的主链路：
```text
apply_handle_insert()
  -> begin_replication_step()
  -> logicalrep_read_insert()
  -> logicalrep_rel_open(relid, RowExclusiveLock)
  -> SwitchToUntrustedUser(owner) if !run_as_owner
  -> create_edata_for_relation()
  -> slot_store_data()
  -> slot_fill_defaults()
  -> ExecOpenIndices()
  -> apply_handle_insert_internal()
     -> InitConflictIndexes()
     -> TargetPrivilegesCheck(..., ACL_INSERT)
     -> ExecSimpleRelationInsert()
  -> finish_edata()
  -> RestoreUserContext()
  -> logicalrep_rel_close()
  -> end_replication_step()
```
关键边界在 `ExecSimpleRelationInsert()`。
它不是绕过 executor 直接写 heap。
它会执行：
```text
CheckCmdReplicaIdentity(rel, CMD_INSERT)
BEFORE ROW INSERT triggers
generated stored columns
ExecConstraints()
ExecPartitionCheck()
simple_table_tuple_insert()
ExecInsertIndexTuples()
AFTER ROW INSERT triggers
```
`InitConflictIndexes()` 只收集当前实现支持诊断的 unique indexes：
```text
ii_Unique == true
indimmediate == true
```
也就是非 deferrable 的唯一约束相关索引。
当 executor 插入索引发现唯一冲突时，logical apply 希望给出更好的 replication conflict 诊断。
因此 `ExecSimpleRelationInsert()` 对这些 indexes 使用 `EIIT_NO_DUPE_ERROR`，先不让普通唯一错误直接抛出。
随后：
```text
if conflict:
  CheckAndReportConflict(..., CT_INSERT_EXISTS, ...)
    -> FindConflictTuple()
    -> GetTupleTransactionInfo()
    -> ReportApplyConflict(..., ERROR, CT_INSERT_EXISTS, ...)
```
`ReportApplyConflict()` 做两件事：
1. `pgstat_report_subscription_conflict(MySubscription->oid, type)`。
2. `ereport(ERROR, ...)` 或 `ereport(LOG, ...)`，由 caller 传入 elevel 决定。
对 duplicate key，caller 传的是 `ERROR`。
所以它会进入 `start_apply()` 的 `PG_CATCH()`。
为什么不能自动 `ON CONFLICT DO NOTHING`？
因为 apply worker 不知道业务语义。
同一个唯一键冲突可能表示：
```text
本地误插入，需要删除本地行后重放远端事务；
两端生成了同一个业务 id，但内容不同，需要人工合并；
远端应该赢，需要把本地行替换；
本地应该赢，需要跳过整个远端事务；
schema 或 sequence 配置错了，继续跳过会掩盖更大问题。
```
内核若自动选择 `DO NOTHING`，会把远端事务的某一行静默丢掉。
内核若自动选择 `DO UPDATE`，会在没有 SQL 语义的情况下构造一条业务 merge。
这都会把 logical replication 从“重放远端提交结果”变成“内核自作主张改写业务历史”。
因此 duplicate key 是典型必须由运维或应用语义解决的 ERROR。
可选动作通常是：
```text
修正 subscriber 本地数据后让 worker 重放；
修正 publisher / subscriber sequence 或 key 生成策略；
确认该远端事务整体可丢弃后，用 ALTER SUBSCRIPTION ... SKIP 指定 finish LSN；
临时 disable subscription，做人工 reconcile，再 enable。
```
## 7. UPDATE / DELETE：tuple 定位、missing row 和当前真实边界
UPDATE 和 DELETE 比 INSERT 多一个步骤：
```text
先用 replica identity 定位 subscriber 本地行
再执行本地 update/delete
```
`apply_handle_update_internal()`：
```text
found = FindReplTupleInLocalRel(..., remoteslot, &localslot)
if found:
  maybe ReportApplyConflict(LOG, CT_UPDATE_ORIGIN_DIFFERS)
  slot_modify_data()
  TargetPrivilegesCheck(..., ACL_UPDATE)
  ExecSimpleRelationUpdate()
else:
  type = CT_UPDATE_DELETED or CT_UPDATE_MISSING
  ReportApplyConflict(LOG, type)
```
`apply_handle_delete_internal()`：
```text
found = FindReplTupleInLocalRel(..., remoteslot, &localslot)
if found:
  maybe ReportApplyConflict(LOG, CT_DELETE_ORIGIN_DIFFERS)
  TargetPrivilegesCheck(..., ACL_DELETE)
  ExecSimpleRelationDelete()
else:
  ReportApplyConflict(LOG, CT_DELETE_MISSING)
```
当前 master 的真实边界是：
```text
missing row for update/delete:
  LOG conflict
  increment conflict counter
  do not throw ERROR
  worker continues
```
这点必须和 duplicate key 区分。
missing row 仍然是业务冲突。
但当前实现选择把它诊断出来并继续，而不是停止整个 apply。
它不会自动补行。
也不会自动把 UPDATE 转 INSERT。
也不会把 DELETE 变成成功删除。
原因同样是语义不足：
```text
行缺失可能表示 subscriber 已经被本地应用删除；
可能表示历史复制中曾经跳过一个 INSERT；
可能表示 replica identity 配置导致找错目标；
可能表示分区路由或 schema 演化改变了目标位置；
可能表示这个 DELETE 本来就是幂等业务语义中可接受的重复删除。
```
内核无法判断哪一个成立。
当前实现只在可见层面留下证据。
如果你看到 `confl_update_missing` 或 `confl_delete_missing` 增长，不应马上假设 worker 停了。
要先看 `apply_error_count`、worker pid、日志 ERROR 和 origin progress。
### 7.1 tuple 查找如何工作
`FindReplTupleInLocalRel()`：
```text
TargetPrivilegesCheck(localrel, ACL_SELECT)
if localidxoid valid:
  RelationFindReplTupleByIndex(..., LockTupleExclusive, ...)
else:
  RelationFindReplTupleSeq(..., LockTupleExclusive, ...)
```
UPDATE / DELETE 需要 `SELECT` 权限是因为 apply worker 必须读取 identity columns 来定位行。
这不是普通用户 SQL 层面“UPDATE 可以不显式 SELECT”的抽象。
在 replication apply 的内部实现中，定位目标行本身就是读操作。
`RelationFindReplTupleByIndex()` 使用 dirty snapshot 查找候选 tuple。
遇到正在被其他事务锁住或修改的 tuple，会等待并 retry。
找到后再用 `table_tuple_lock(..., LockTupleExclusive, ...)` 锁定目标。
这保证 apply 不会在并发本地修改中盲写。
但它不提供跨节点业务冲突解决。
### 7.2 update_deleted 与 retain_dead_tuples
当前 master 有更细的 conflict diagnostics。
`FindDeletedTupleInLocalRel()` 可能把 update missing 细分为 update deleted。
前提包括：
```text
MySubscription->retaindeadtuples
track_commit_timestamp
leader apply worker still retains oldest_nonremovable_xid
dead tuple information not expired by max_retention
```
如果这些条件不满足，旧删除历史可能只能表现为 `update_missing`。
这说明 diagnostics 是有边界的。
统计值不是完整事实。
它只是当前配置和保留窗口下能识别出的事实。
## 8. replica identity 不足：为什么不能猜目标行
UPDATE / DELETE 的本质问题是：
```text
远端发来的旧行信息能否唯一定位 subscriber 本地行？
```
`relation.c` 中 `logicalrep_rel_mark_updatable()` 只做标记。
它不会立刻 ERROR。
它检查的是：
```text
subscriber 本地是否有 replica identity index 或 primary key；
如果没有，本地能否依赖 publisher 的 REPLICA IDENTITY FULL；
本地 identity 需要的列是否都出现在 publisher 发来的 identity columns 中。
```
如果不足：
```text
entry->updatable = false
```
真正抛错发生在 UPDATE / DELETE 路径的 `check_relation_updatable()`。
错误有两类典型文本：
```text
publisher did not send replica identity column expected by the logical replication target relation
logical replication target relation ... has neither REPLICA IDENTITY index nor PRIMARY KEY
and published relation does not have REPLICA IDENTITY FULL
```
`execReplication.c` 还有一层 `CheckCmdReplicaIdentity()`。
它检查 publication row filter、column list 和 generated columns 是否覆盖 replica identity。
典型错误包括：
```text
Column used in the publication WHERE expression is not part of the replica identity.
Column list used by the publication does not cover the replica identity.
Replica identity must not contain unpublished generated columns.
cannot update/delete table ... because it does not have a replica identity and publishes updates/deletes
```
为什么不能 fallback 到“随便扫全表找相等行”？
有些情况可以扫。
例如远端是 `REPLICA IDENTITY FULL` 时，当前代码允许没有 index 时走 sequential scan。
但这依赖 publisher 确实发送了足够的旧行列值。
如果 publisher 没发 subscriber identity 需要的列，subscriber 没有信息构造搜索条件。
这不是性能问题。
这是信息缺失。
内核不能凭当前 new tuple 或部分列猜测目标行。
猜错会比停住更坏：
```text
可能更新另一行业务对象；
可能删除本不该删除的行；
可能让唯一约束和外键错误延迟到更难诊断的位置；
可能推进 origin 后永久丢失正确变化。
```
所以 replica identity mismatch 是必须修 DDL / publication / subscription 配置的问题。
修复后通常让同一远端事务重放。
## 9. schema、列、类型和生成列边界
`RELATION` 消息只更新 relation map。
`apply_handle_relation()` 的注释强调：
```text
不在收到 RELATION 时立即验证本地 schema；
验证推迟到第一次真正对该 relation 应用变化。
```
原因是减少不必要锁和验证。
某个 relation metadata 可能被发送，但当前 worker 并不需要实际应用它。
真正打开本地 relation 的地方是 `logicalrep_rel_open()`。
它会：
```text
按 remote namespace/name 找本地 relation
检查 relkind 是否支持并兼容
按列名建立 attrmap
检查 publisher 发送的 replicated columns 在 subscriber 是否存在
检查 subscriber 同名 generated columns 是否兼容
标记 updatable
选择本地查找 index
```
### 9.1 缺失 relation 或缺失列
如果本地 relation 不存在：
```text
logical replication target relation "schema.table" does not exist
```
如果本地缺少 publisher 复制的列：
```text
logical replication target relation "schema.table" is missing replicated column: "col"
```
这会 ERROR。
因为继续应用意味着丢弃远端列值。
内核不能替用户决定某列可以丢。
### 9.2 本地额外列
本地额外列不一定错误。
`slot_store_data()` 对远端没有的本地列先置 NULL。
之后 `slot_fill_defaults()` 会填本地默认值。
如果额外列没有默认值又是 NOT NULL，后续 `ExecConstraints()` 会 ERROR。
这个 ERROR 属于 subscriber schema 与复制流不兼容。
解决方式是 DDL 或默认值语义，不是 apply 自动生成业务值。
### 9.3 类型不匹配
`logicalrep_read_attrs()` 会读远端 type oid。
但当前 apply 不是简单比较远端 type oid 和本地 type oid。
行值进入 slot 时：
```text
TEXT mode:
  getTypeInputInfo(local_atttypid)
  OidInputFunctionCall(local input function, remote text)
BINARY mode:
  getTypeBinaryInputInfo(local_atttypid)
  OidReceiveFunctionCall(local receive function, remote bytes)
  检查 colvalue cursor 是否消费完整
```
所以类型不匹配的表现取决于传输格式和本地类型输入函数。
可能是：
```text
invalid input syntax
incorrect binary data format in logical replication column N
domain / check constraint failure
typmod failure
```
二进制复制尤其要求两端类型二进制表示兼容。
内核不会把任意远端类型自动 cast 成本地业务类型。
这同样是 schema migration 和发布列设计问题。
### 9.4 relkind mismatch
`CheckSubscriptionRelkind()` 允许普通表和分区表在 table 语义上互通。
但 sequence 必须两端一致。
如果 source 是 sequence、target 是 table，或相反，会 ERROR：
```text
relation "schema.name" type mismatch: source "...", target "..."
```
这不是行级冲突。
这是复制对象类型配置错误。
## 10. 权限、run_as_owner、RLS 和触发器
logical apply 不是 superuser 魔法写入。
它在明确的用户上下文中运行。
### 10.1 run_as_owner
当前源码支持 `run_as_owner`。
默认是 `false`。
`CREATE SUBSCRIPTION` 文档和 `subscriptioncmds.c` 中都能看到该选项。
在 `apply_handle_insert()` / `apply_handle_update()` / `apply_handle_delete()` 中：
```text
run_as_owner = MySubscription->runasowner
if !run_as_owner:
  SwitchToUntrustedUser(relowner, &ucxt)
...
if !run_as_owner:
  RestoreUserContext(&ucxt)
```
因此：
```text
run_as_owner = true:
  复制动作作为 subscription owner 执行
run_as_owner = false:
  每张表的复制动作作为 relation owner 执行
```
`TargetPrivilegesCheck()` 使用当前 `GetUserId()` 做 ACL 检查。
INSERT 检查 `ACL_INSERT`。
UPDATE 检查 `ACL_SELECT` 定位行，再检查 `ACL_UPDATE`。
DELETE 检查 `ACL_SELECT` 定位行，再检查 `ACL_DELETE`。
TRUNCATE 检查 `ACL_TRUNCATE`。
权限不足会 ERROR。
这必须由授权、owner、subscription 配置或安全模型修正。
apply worker 不能自动提升权限。
### 10.2 RLS
`TargetPrivilegesCheck()` 还有一段非常硬的边界：
```text
if check_enable_rls(...) == RLS_ENABLED:
  ERROR "cannot replicate into relation with row-level security enabled"
```
源码注释说明：当前缺少 honor RLS policies 的基础设施。
即使 TRUNCATE 通常不适用 RLS，logical apply 也统一拒绝。
这是安全语义问题。
内核不能在不知道 policy 应该如何应用于复制流的情况下绕过或模拟 RLS。
所以 RLS enabled target relation 是需要 DBA 处理的配置边界。
### 10.3 触发器
`InitializeLogRepWorker()` 设置：
```text
session_replication_role = replica
```
`trigger.c` 的 `TriggerEnabled()` 在 replica role 下：
```text
TRIGGER_FIRES_ON_ORIGIN:
  不触发
TRIGGER_DISABLED:
  不触发
TRIGGER_FIRES_ON_REPLICA:
  触发
TRIGGER_FIRES_ALWAYS:
  触发
```
`ExecSimpleRelationInsert()`、`ExecSimpleRelationUpdate()`、`ExecSimpleRelationDelete()` 会调用 row-level BEFORE / AFTER triggers。
BEFORE trigger 可以返回 skip tuple。
触发器也可以抛 ERROR。
如果 replica / always trigger 抛 ERROR，这和约束错误一样进入 apply ERROR 边界。
注意当前 `ExecSimpleRelationInsert()` 注释还提到：after statement triggers 在 replication 路径中并没有像普通 executor 一样完整触发。
不要把普通 SQL DML 的所有触发器行为直接等同于 logical apply。
## 11. ERROR 后：abort、worker 退出和重启
所有能抛 ERROR 的路径最终都会被 `start_apply()` 包住：
```text
void
start_apply(XLogRecPtr origin_startpos)
{
  PG_TRY();
  {
    LogicalRepApplyLoop(origin_startpos);
  }
  PG_CATCH();
  {
    replorigin_xact_clear(true);
    if (MySubscription->disableonerr)
      DisableSubscriptionAndExit();
    else
    {
      AbortOutOfAnyTransaction();
      pgstat_report_subscription_error(MySubscription->oid);
      PG_RE_THROW();
    }
  }
  PG_END_TRY();
}
```
这段路径回答了四个诊断问题。
第一，为什么本地事务会 abort？
因为 `AbortOutOfAnyTransaction()` 会离开任何 active transaction 或 subtransaction。
`xact.c` 中它会对活跃事务执行：
```text
AbortTransaction()
CleanupTransaction()
blockState = TBLOCK_DEFAULT
MemoryContextSwitchTo(TopMemoryContext)
```
这会释放快照、locks、buffers、relcache refs、after trigger state、stats pending transaction state 等事务资源。
第二，为什么 origin 不前进？
因为 catch 中先执行 `replorigin_xact_clear(true)`。
失败事务没有经过 `apply_handle_commit_internal()` 的 `CommitTransactionCommand()`。
第三，为什么 worker 会退出？
`disable_on_error = false` 时，catch 在 abort 和 stats 后 `PG_RE_THROW()`。
apply background worker 没有在同一个进程里继续吞掉这个错误并跳到下一事务。
错误会结束当前 worker。
logical replication launcher 之后看到 enabled subscription 没有 apply worker，会按 `wal_retrieve_retry_interval` 控制重启节奏。
这避免确定性错误导致 tight restart loop。
第四，`disable_on_error = true` 改了什么？
`DisableSubscriptionAndExit()` 会：
```text
EmitErrorReport()
AbortOutOfAnyTransaction()
FlushErrorState()
pgstat_report_subscription_error()
StartTransactionCommand()
PushActiveSnapshot(GetTransactionSnapshot())
DisableSubscription(MySubscription->oid)
CommitTransactionCommand()
ereport(LOG, "subscription ... has been disabled because of an error")
proc_exit(0)
```
它先把失败事务 abort 掉，再用新事务修改 `pg_subscription.subenabled`。
这很重要。
禁用 subscription 不能和失败的 apply 事务混在一起。
如果混在一起，apply abort 会把 disable catalog update 一起回滚。
所以 disable-on-error 是一个 operational breaker。
它不是在失败事务内部做修复。
## 12. 为什么停在事务边界，而不是停在行边界
远端事务是 logical replication 的最小一致性单元。
行级 message 只是事务内部变化。
本地执行时：
```text
BEGIN message:
  记录远端事务状态
第一个 DML:
  StartTransactionCommand()
每个 DML:
  PushActiveSnapshot()
  执行本地 executor
  PopActiveSnapshot()
  CommandCounterIncrement()
COMMIT message:
  设置 origin_lsn / origin_timestamp
  CommitTransactionCommand()
```
如果第 100 行失败，前 99 行也不能保留。
因为 publisher 的事务语义是 all-or-nothing。
如果 subscriber 保留前 99 行，等于把远端一个事务拆成了两个结果：
```text
subscriber 已经持久化部分变化
publisher 认为事务是完整提交
origin 又不能推进
重启后同一事务还会重放
```
这会制造 duplicate、missing、trigger side effects 和约束错误的二次问题。
所以 PostgreSQL 选择：
```text
ERROR:
  abort 整个本地 apply 事务
  不推进 origin
  让同一远端事务之后重新出现
```
如果 DBA 确认整个远端事务不应该应用，可以用：
```text
ALTER SUBSCRIPTION subname SKIP (lsn = 'finish_lsn')
```
这也是事务级的 skip。
`maybe_start_skipping_changes()` 只在 `BEGIN` 或 streaming transaction start 知道 finish LSN 后启用 skip。
`apply_handle_insert()` / `update()` / `delete()` 看到 `is_skipping_changes()` 会 quick return。
`clear_subscription_skip_lsn()` 在事务完成或跳过后清掉 `subskiplsn`。
它不是行级跳过。
这符合本节核心边界：
```text
内核可以按 finish LSN 跳过整个远端事务；
不能在没有业务语义的情况下跳过某一行并提交剩余行。
```
## 13. conflict diagnostics：能看到什么
当前 master 对 logical replication conflict 的观测比旧版本丰富。
### 13.1 日志 error context
`LogicalRepApplyLoop()` 注册 `apply_error_callback()`。
它会根据当前状态输出类似信息：
```text
processing remote data for replication origin "pg_..." during message type "INSERT"
processing remote data for replication origin "pg_..." during message type "UPDATE" in transaction ...
processing remote data for replication origin "pg_..." during message type "UPDATE" for replication target relation "s.t" column "c" in transaction ..., finished at ...
```
这些 context 来自：
```text
apply_error_callback_arg.origin_name
apply_error_callback_arg.command
apply_error_callback_arg.remote_xid
apply_error_callback_arg.finish_lsn
apply_error_callback_arg.rel
apply_error_callback_arg.remote_attnum
```
诊断时不要只看最上面的错误文本。
要看 CONTEXT 行。
finish LSN 是后续 `ALTER SUBSCRIPTION ... SKIP` 和定位远端事务的关键值。
### 13.2 ReportApplyConflict 日志
`ReportApplyConflict()` 会输出：
```text
conflict detected on relation "schema.table": conflict=...
```
conflict name 来自 `conflict.c`：
```text
insert_exists
update_origin_differs
update_exists
update_deleted
update_missing
delete_origin_differs
delete_missing
multiple_unique_conflicts
```
对于唯一冲突，它会尽量报告 conflicting key、本地 tuple、远端 tuple、local xmin、origin 和 commit timestamp。
是否能显示 origin 和 timestamp 依赖 `track_commit_timestamp` 以及 origin 是否仍存在。
对于 update_deleted，它还依赖 dead tuple retention。
### 13.3 pg_stat_subscription
`pg_stat_subscription` 是 worker 当前状态视图。
核心字段：
| 字段 | 粒度 |
| --- | --- |
| `worker_type` | apply、parallel apply、table synchronization、sequence synchronization。 |
| `pid` | 当前 worker 进程。 |
| `leader_pid` | parallel apply worker 对应 leader。 |
| `relid` | sync worker 正在同步的 relation。 |
| `received_lsn` | worker 最近收到的位置。 |
| `latest_end_lsn` | 最近上报的结束位置。 |
它回答：
```text
worker 是否还在？
是 leader apply 还是 sync worker？
当前大致收到哪里？
```
它不回答：
```text
哪个远端事务语义正确？
冲突应该怎么解决？
失败事务内部哪一行业务应该保留？
```
### 13.4 pg_stat_subscription_stats
`pg_stat_subscription_stats` 是错误和冲突累计统计。
当前 view 包括：
```text
apply_error_count
sync_seq_error_count
sync_table_error_count
confl_insert_exists
confl_update_origin_differs
confl_update_exists
confl_update_deleted
confl_update_missing
confl_delete_origin_differs
confl_delete_missing
confl_multiple_unique_conflicts
stats_reset
```
`pgstat_subscription.c` 中：
```text
pgstat_report_subscription_error()
  -> apply_error_count++ for WORKERTYPE_APPLY
pgstat_report_subscription_conflict()
  -> conflict_count[type]++
```
注意：
```text
会导致 apply ERROR 的 conflict，会同时增加 apply_error_count 和对应 confl_*。
LOG 级 missing row / origin differs，会增加 confl_*，但不一定增加 apply_error_count。
```
所以诊断时先区分：
```text
confl_* 增长:
  表示检测到某类 conflict
apply_error_count 增长:
  表示 worker 应用变化时发生 ERROR
pid 消失且 subscription enabled:
  可能正在等待 launcher 重启
subenabled = false:
  可能 disable_on_error 生效或人工 disable
```
## 14. 诊断流程：看到停住后怎么回到源码
第一步，看 subscription 是否 enabled 和 worker 是否存在：
```sql
SELECT subname, subenabled, subdisableonerr, subskiplsn
FROM pg_subscription
WHERE subname = 'mysub';
SELECT *
FROM pg_stat_subscription
WHERE subname = 'mysub';
```
如果 `subenabled = false` 且日志有：
```text
subscription "..." has been disabled because of an error
```
先按 `DisableSubscriptionAndExit()` 理解。
它说明 `disable_on_error` 生效，worker 已经 abort 失败事务并禁用 subscription。
第二步，看错误和 conflict 计数：
```sql
SELECT *
FROM pg_stat_subscription_stats
WHERE subname = 'mysub';
```
如果 `confl_insert_exists` 或 `confl_update_exists` 增长并且 `apply_error_count` 增长：
回到 `ExecSimpleRelationInsert()` / `ExecSimpleRelationUpdate()` 的唯一冲突 ERROR 路径。
如果 `confl_update_missing` 或 `confl_delete_missing` 增长但 worker 仍在：
回到 `apply_handle_update_internal()` / `apply_handle_delete_internal()` 的 LOG 路径。
不要把它误诊为当前事务停住。
第三步，从日志 CONTEXT 找 finish LSN：
```text
finished at X/Y
```
这个 LSN 对应远端事务边界。
如果最终业务判断是“这个远端事务整体不应应用”，才考虑：
```sql
ALTER SUBSCRIPTION mysub SKIP (lsn = 'X/Y');
ALTER SUBSCRIPTION mysub ENABLE;
```
不要在没有业务确认时跳过。
跳过是事务级丢弃。
第四步，按错误类型回到边界：
| 现象 | 优先读源码 | 判断 |
| --- | --- | --- |
| duplicate key | `ExecSimpleRelationInsert()`、`CheckAndReportConflict()`、`ReportApplyConflict()` | 本地已有行和远端新行冲突，需合并或修数据。 |
| cannot update/delete no replica identity | `logicalrep_rel_mark_updatable()`、`check_relation_updatable()`、`CheckCmdReplicaIdentity()` | DDL / publication columns / row filter 配置不足。 |
| missing replicated column | `logicalrep_rel_open()`、`logicalrep_report_missing_or_gen_attrs()` | subscriber schema 缺列或 generated column 不兼容。 |
| invalid input syntax / binary format | `slot_store_data()`、`slot_modify_data()`、`logicalrep_read_tuple()` | 类型输入或二进制格式不兼容。 |
| permission denied | `TargetPrivilegesCheck()`、`run_as_owner` 相关切换 | owner、ACL 或 subscription 安全配置不符合。 |
| RLS enabled | `TargetPrivilegesCheck()` | 当前 apply 不支持 honor RLS policy。 |
| trigger error | `ExecSimpleRelation*()`、`TriggerEnabled()` | replica / always trigger 的业务代码失败。 |
第五步，看 origin 是否前进。
如果事务 ERROR，正常情况下 origin 不应推进到失败事务 end LSN。
这来自 `replorigin_xact_clear(true)` 和未执行 `CommitTransactionCommand()`。
如果你怀疑 origin 已推进，需要核对：
```text
日志中失败事务 finish LSN
pg_replication_origin_status 中 subscription origin remote_lsn
publisher slot confirmed_flush_lsn
subscriber pg_stat_subscription latest_end_lsn
```
这些指标粒度不同。
不要把某个 LSN 单独解释为“事务已经成功应用”。
## 15. 为什么不自动解决业务冲突
logical apply 的输入信息比业务 SQL 少。
它没有原始 SQL predicate。
它没有应用层 merge rule。
它没有用户意图。
它只有远端提交后的 tuple images 和 relation metadata。
因此自动解决会破坏至少一个边界。
### 15.1 duplicate key
可选业务语义包括：
```text
远端 wins
本地 wins
字段级 merge
丢弃远端事务
删除本地冲突行后重放
修 sequence 后重放
```
内核没有理由选择其中任何一个。
### 15.2 missing row
可选业务语义包括：
```text
UPDATE missing 视为 no-op
UPDATE missing 视为补 INSERT
DELETE missing 视为幂等成功
missing 表示必须报警并人工修复
```
当前实现对 missing row 选择 LOG conflict 并继续。
但这仍不是“解决”。
它只是当前内核选定的 apply 行为。
业务仍应解释为什么 subscriber 缺行。
### 15.3 schema mismatch
缺列、类型不兼容、generated column、不满足 NOT NULL 或 CHECK，都属于 schema contract 破裂。
自动填 NULL、自动 cast、自动丢列都会制造不可追踪的数据语义变化。
### 15.4 permission / RLS
权限失败和 RLS 失败是安全边界。
自动绕过安全边界会比复制延迟更严重。
正确动作是修 owner、ACL、RLS 配置或复制拓扑。
### 15.5 replica identity mismatch
identity 不足是信息不足。
自动扫描不一定能构造正确搜索条件。
如果远端没发送 subscriber 需要的 identity columns，内核没有可验证的目标行。
正确动作是修改 `REPLICA IDENTITY`、publication column list / row filter，或让 publisher 使用 `REPLICA IDENTITY FULL` 并接受成本。
## 16. 成本、资源与跨模块传播
apply conflict 不只是 correctness。
它也会影响资源。
### 16.1 重启成本
`disable_on_error = false` 时，确定性 ERROR 会形成：
```text
worker start
  -> connect publisher
  -> start streaming from origin
  -> receive same transaction
  -> fail at same point
  -> abort
  -> exit
  -> launcher waits wal_retrieve_retry_interval
  -> restart
```
这会消耗 connection、WAL sender、launcher 和日志资源。
`wal_retrieve_retry_interval` 是防 tight loop 的 operational throttle。
### 16.2 大事务 abort 成本
远端事务越大，失败越晚，本地 abort 丢弃的工作越多。
成本包括：
```text
heap/index insert/update/delete 的本地 WAL
锁和 buffer 活动
触发器执行成本
constraint check 成本
内存上下文与 executor state cleanup
再次重放同一事务的 CPU / IO
```
事务没有 commit 不代表没有本地工作量。
它只是 crash/recovery 和可见性语义上不会留下提交结果。
### 16.3 replica identity FULL 成本
如果 publisher 使用 `REPLICA IDENTITY FULL`，subscriber 可能可以定位更多 UPDATE / DELETE。
但成本会放大：
```text
publisher 发送更多旧行列值
subscriber 可能需要按 tuple equality 比较
没有可用 index 时 sequential scan
大量 UPDATE / DELETE 时 CPU 和 IO 随表大小放大
```
`FindUsableIndexForReplicaIdentityFull()` 尝试找可用 index。
但它有严格条件：
```text
非 partial index
每个 key column 有 equality strategy
类型有 equality operator
leftmost index field 是远端列对应的本地列
index AM 支持 amgettuple
```
找不到时就退到 seq scan。
### 16.4 conflict diagnostics 成本
为了报告 unique conflict，当前路径选择：
```text
先尝试插入 heap / index
发现 conflict 后再找 conflicting tuple
```
源码注释说明这是为了避免每次 INSERT 都先额外 index scan。
冲突应是少数。
如果冲突很多，这个选择会带来额外 cleanup 和重复工作。
这也是为什么 conflict 不应作为常态业务 merge 机制使用。
### 16.5 retain_dead_tuples 成本
`retain_dead_tuples` 为 update_deleted 等诊断保留 dead tuple 信息。
launcher 维护名为 `pg_conflict_detection` 的内部 slot。
这会影响 dead tuple 可移除边界。
`max_retention_duration` 用来限制保留时间。
启用它是 diagnostics 和 vacuum / bloat 风险之间的取舍。
不要为所有系统无脑启用。
## 17. 常见误区
误区一：看到 `conflict` 就等于 worker 停了。当前 master 中 missing row 和 origin differs 是 LOG 级 conflict，要结合 `apply_error_count`、worker pid 和 ERROR 日志判断。
误区二：`disable_on_error` 会解决冲突。它只是 ERROR 后自动 disable subscription，避免持续重启，数据和 schema 仍需人工或应用处理。
误区三：`ALTER SUBSCRIPTION ... SKIP` 可以跳过某一行。它按远端事务 finish LSN 跳过整个事务的数据修改。
误区四：`REPLICA IDENTITY FULL` 总能安全而便宜地解决 UPDATE / DELETE 定位。它能提供更多旧行信息，但可能导致大 tuple 传输、seq scan 和 equality 比较成本。
误区五：subscriber 多出的列会被 publisher 自动填好。本地默认值可以填，否则约束失败就是 schema contract 问题。
误区六：replica role 下触发器都不会触发。`ENABLE REPLICA` 和 `ENABLE ALWAYS` 触发器仍可能触发并抛 ERROR。
误区七：RLS policy 会像普通 SQL 一样被 apply 自动执行。当前 apply 缺少 honor RLS policies 的基础设施，目标 relation 启用 RLS 会 ERROR。
## 18. 课堂实验
### 实验一：duplicate key 停在事务边界
目标：看到唯一冲突如何变成 ERROR，并确认失败事务不会推进 origin。
步骤：
```text
1. publisher 建表 t(id primary key, v text)，加入 publication。
2. subscriber 建同名表和 subscription。
3. 等初始同步完成。
4. 在 subscriber 本地插入一行会和 publisher 下一次 INSERT 冲突的 id。
5. 在 publisher 执行一个事务，插入该 id，并在同一事务内再插入另一行。
6. 观察 subscriber 日志和 pg_stat_subscription_stats。
7. 删除或修正 subscriber 冲突行，重新 enable / 等 worker 重启。
8. 确认远端事务两行一起应用，而不是只应用后半段。
```
源码断点：`ExecSimpleRelationInsert`、`CheckAndReportConflict`、`ReportApplyConflict`、`start_apply` 的 `PG_CATCH`、`AbortOutOfAnyTransaction`。
要画出的状态：
```text
remote_xid
finish_lsn
IsTransactionState()
replorigin_xact_state.origin_lsn
pg_stat_subscription_stats.apply_error_count
pg_stat_subscription_stats.confl_insert_exists
```
### 实验二：missing row 是 LOG 级 conflict
目标：区分 conflict 计数和 apply ERROR。
步骤：
```text
1. 两端建相同表 t(id primary key, v text)。
2. publisher 插入 id=1，同步到 subscriber。
3. 在 subscriber 本地删除 id=1。
4. publisher 更新 id=1。
5. 观察 subscriber 日志出现 update_missing 或 update_deleted。
6. 查看 pg_stat_subscription_stats 的 confl_update_missing / confl_update_deleted。
7. 查看 apply_error_count 是否因为这一条单独增加。
8. 查看 worker 是否继续处理后续事务。
```
源码断点：`apply_handle_update_internal`、`FindReplTupleInLocalRel`、`FindDeletedTupleInLocalRel`、`ReportApplyConflict`。
关键判断：`ReportApplyConflict` 的 `elevel` 是 `LOG` 还是 `ERROR`。
### 实验三：replica identity mismatch
目标：看到信息不足如何在 UPDATE / DELETE 前变成 ERROR。
步骤：
```text
1. publisher 表没有足够 replica identity，或 publication column list 不包含 identity 所需列。
2. subscriber 表使用需要该列的 primary key / replica identity。
3. publisher 执行 UPDATE 或 DELETE。
4. 观察 subscriber ERROR 和 CONTEXT。
5. 修正 REPLICA IDENTITY 或 publication column list 后重放。
```
源码断点：`logicalrep_rel_open`、`logicalrep_rel_mark_updatable`、`check_relation_updatable`、`CheckCmdReplicaIdentity`。
### 实验四：RLS 与触发器边界
目标：看到安全和用户代码如何参与 apply ERROR。
步骤：
```text
1. subscriber 目标表启用 RLS，触发 publisher 写入。
2. 观察 TargetPrivilegesCheck 抛出 RLS ERROR。
3. 关闭 RLS，创建 ENABLE REPLICA trigger，让 trigger 抛 ERROR。
4. 再次触发 publisher 写入。
5. 观察 trigger ERROR 进入同一个 start_apply abort 边界。
```
源码断点：`TargetPrivilegesCheck`、`ExecSimpleRelationInsert`、`TriggerEnabled`、`start_apply`、`DisableSubscriptionAndExit`。
## 19. 讨论题
1. 为什么 duplicate key 当前是 ERROR，而 missing row 当前是 LOG？
2. 如果把 duplicate key 自动改成 `DO NOTHING`，哪条 replication origin 不变量会被破坏？
3. `apply_handle_update_internal()` 找不到目标行时，为什么不能默认把 UPDATE 转 INSERT？
4. 为什么 `logicalrep_rel_open()` 推迟 schema validation，而不是收到 `RELATION` 消息立即验证？
5. `attrmap` 能解决列名映射，为什么不能解决任意类型转换？
6. `run_as_owner = false` 为什么通常更安全，它对权限错误诊断有什么影响？
7. `retain_dead_tuples` 能改善哪类诊断，又会把成本传播到哪些模块？
8. `ALTER SUBSCRIPTION ... SKIP` 为什么按 finish LSN 跳过整个事务，而不是跳过当前失败行？
## 20. 本节小结
logical apply 的错误边界不是“某行失败就随便跳过”。
它的核心链路是：
```text
BEGIN 记录远端事务 finish LSN
  -> DML 在本地事务中通过 executor 应用
  -> COMMIT 才推进 replication origin
  -> ERROR 则清除 origin xact state 并 abort 本地事务
  -> worker 退出或 disable subscription
```
核心状态包括：
```text
remote_xid / finish_lsn / message type / relation / column context
LogicalRepRelMapEntry 的 attrmap / updatable / localindexoid
Subscription 的 disableonerr / runasowner / skiplsn / retaindeadtuples
replication origin xact state
pg_stat_subscription 和 pg_stat_subscription_stats
```
当前源码的真实边界要分清：
```text
duplicate key:
  conflict diagnostics + ERROR
missing row for update/delete:
  conflict diagnostics + LOG
origin differs:
  conflict diagnostics + LOG
schema / permission / RLS / replica identity / type input / constraints / trigger:
  ERROR
```
ERROR 后的 cleanup 由 `start_apply()` 和 `AbortOutOfAnyTransaction()` 收尾。
`disable_on_error` 只决定是否把 subscription 自动 disable。
它不解决冲突。
诊断上：
```text
日志 CONTEXT 给远端 origin、message type、relation、column、XID、finish LSN；
pg_stat_subscription 看 worker 当前状态和 LSN；
pg_stat_subscription_stats 看 apply_error_count 与 confl_* 累计；
origin LSN 用来判断失败事务是否已经成功越过 commit 边界。
```
从本节可迁移出的系统规律是：
```text
复制系统可以自动保证传输顺序、事务原子性和失败回滚；
但一旦冲突需要解释业务对象等价性、安全策略或 schema migration，
内核只能停止、记录或有限诊断，不能替应用选择语义。
```
哪些判断仍然有边界：
```text
missing row 是否可接受依赖业务；
update_deleted 能否识别依赖 retain_dead_tuples、track_commit_timestamp 和保留窗口；
replica identity FULL 的成本依赖表大小、索引和 UPDATE/DELETE 频率；
trigger ERROR 是否应修触发器还是跳过事务依赖应用设计；
skip LSN 是否安全只能由理解远端事务语义的人决定。
```
