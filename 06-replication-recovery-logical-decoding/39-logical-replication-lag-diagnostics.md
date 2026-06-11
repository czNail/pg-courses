# PostgreSQL logical replication lag diagnostics
## 课程定位
前置知识：
已经理解 logical decoding、replication slot、walsender、subscription apply worker 和 replication origin 的基本职责。
本节唯一主问题：
```text
如何用 confirmed_flush_lsn、restart_lsn、subscription stats、apply worker wait event
和目标端事务压力判断 logical replication lag 来自解码、传输、apply 还是冲突重试？
```
核心矛盾：
```text
逻辑复制需要把 publisher 的 WAL 流变成 subscriber 上的事务重放；
但“进度”分散在 publisher slot、walsender feedback、subscriber worker、
replication origin、pgstat counter 和目标端 executor 等多个边界。
```
学完后应能判断：
```text
confirmed_flush_lsn 不动，是否一定是 publisher 解码慢？
restart_lsn 不动，是否一定是 subscriber 没应用？
received_lsn 前进，是否代表目标表已经追上？
latest_end_lsn 是 apply lsn 还是 keepalive end lsn？
apply_error_count 和 conflict counters 分别说明什么？
apply worker 的 wait_event 如何区分空闲等网络、目标端锁等待和并行 apply 等待？
schema mismatch、unique conflict、missing row、concurrent retry 分别如何表现？
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面几节已经把逻辑复制拆成三段：
```text
publisher:
  WAL record -> logical decoding -> reorder buffer -> output plugin -> walsender
transport:
  CopyData / keepalive / status feedback
subscriber:
  apply worker -> target executor -> local commit -> replication origin
```
本节不重新讲协议格式。
本节只讲诊断：
```text
同一个 lag，应该归因到哪一层？
```
诊断的难点是字段名字容易产生错觉。
例如：
```text
confirmed_flush_lsn:
  在 publisher slot 上。
  但它由 subscriber feedback 推进。
restart_lsn:
  在 publisher slot 上。
  但它表达重新解码和 WAL retention 起点。
received_lsn:
  在 subscriber stats 上。
  但它只说明 apply worker 收到了消息。
latest_end_lsn:
  在 subscriber stats 上。
  但当前源码中它来自 keepalive/status 的 end LSN，不是 apply 完成 LSN。
apply_error_count:
  在 subscriber subscription stats 上。
  但 conflict counters 并不全都会导致 ERROR。
```
因此本节先建立一个统一状态图：
```text
publisher current WAL
  -> logical decoding/reorder
  -> walsender sent_lsn
  -> subscriber received_lsn
  -> target executor
  -> subscriber local commit WAL
  -> local WAL flush
  -> status feedback flushPtr
  -> publisher confirmed_flush_lsn
  -> candidate restart condition
  -> publisher restart_lsn
```
这条链路里任一边界停住，表面上都可能叫 replication lag。
但修复方向完全不同。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
walsender 从 logical slot 的 restart_lsn 开始读 WAL，
以 confirmed_flush_lsn 作为已经确认消费的位置；
apply worker 收到 WALData 后先更新 received_lsn，
只有远端事务在 subscriber 本地 commit 且本地 commit WAL flush 后，
send_feedback() 才把对应远端 LSN 作为 flush 反馈给 publisher；
walsender 收到 flush feedback 后调用 LogicalConfirmReceivedLocation() 推进 confirmed_flush_lsn，
并在 candidate restart 条件满足时推进 restart_lsn。
```
这句话包含两个核心区别。
第一：
```text
confirmed_flush_lsn 是消费确认。
```
它说明：
```text
publisher 收到 client 确认，认为该 logical slot 已经 flush 到某个 LSN。
```
对 subscription apply worker 来说，这个确认不是“刚收到消息”。
它来自：
```text
目标端事务已经本地提交；
本地 commit WAL 已经 flush；
feedback 把远端事务 end_lsn 作为 flushPtr 发回 publisher。
```
第二：
```text
restart_lsn 是重新解码和 WAL retention 起点。
```
它说明：
```text
如果现在要重新读取 WAL 来恢复 decoding state，最早可能还要从哪里开始。
```
所以：
```text
confirmed_flush_lsn:
  消费确认边界。
restart_lsn:
  WAL 保留和重新解码边界。
```
两者不能互换。
一个常见状态是：
```text
confirmed_flush_lsn 持续前进；
restart_lsn 长期不前进。
```
这不必然表示 subscriber apply 慢。
它可能表示：
```text
decoding restart horizon 被长事务、prepared transaction、SnapBuild 或 ReorderBuffer 状态钉住。
```
反过来：
```text
received_lsn 前进；
confirmed_flush_lsn 不前进。
```
这才更像：
```text
subscriber 已经收到，但 apply / local commit / local WAL flush / feedback 还没追上。
```
本课把 lag 拆成四类：
```text
decoding backlog:
  publisher WAL 已有，但 logical decoding/reorder/output 产出慢。
transport backlog:
  publisher 已发送或可发送，但 subscriber receive 或 feedback 慢。
subscriber apply backlog:
  subscriber 已收到，但目标端执行、本地提交或 WAL flush 慢。
conflict / retry stall:
  schema、权限、唯一冲突、missing row、目标端并发事务造成 ERROR、LOG conflict 或 retry。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/catalog/system_views.sql` | 当前视图字段，避免编造列。 |
| 2 | `src/include/replication/slot.h` | `restart_lsn`、`confirmed_flush`、candidate restart/xmin、`last_saved_restart_lsn`。 |
| 3 | `src/backend/replication/slot.c` | slot acquire/release、required LSN、invalidation、checkpoint save。 |
| 4 | `src/backend/replication/logical/logical.c` | `CreateDecodingContext()`、`LogicalConfirmReceivedLocation()`、`UpdateDecodingStats()`。 |
| 5 | `src/backend/replication/walsender.c` | `StartLogicalReplication()`、`XLogSendLogical()`、`ProcessStandbyReplyMessage()`。 |
| 6 | `src/backend/replication/logical/reorderbuffer.c` | spill/stream/memory exceeded 统计来源。 |
| 7 | `src/backend/replication/logical/worker.c` | receive loop、`send_feedback()`、`store_flush_position()`、ERROR cleanup。 |
| 8 | `src/backend/replication/logical/launcher.c` | worker shared slot、`pg_stat_get_subscription()`、launcher retry。 |
| 9 | `src/backend/utils/activity/pgstat_subscription.c` | subscription error/conflict counter。 |
| 10 | `src/backend/utils/activity/pgstat_replslot.c` | replication slot decoding stats。 |
| 11 | `src/backend/executor/execReplication.c` | target executor、tuple lock retry、unique conflict。 |
| 12 | `src/backend/replication/logical/origin.c` | origin `remote_lsn/local_lsn` 与 commit progress。 |
阅读顺序不要按文件名。
按诊断链读：
```text
视图字段
  -> slot 状态
  -> walsender feedback
  -> apply worker receive/apply/feedback
  -> pgstat counters
  -> target executor retry/conflict
```
## 4. 当前源码中的真实视图字段
### 4.1 `pg_replication_slots`
当前 `system_views.sql` 定义的字段是：
```text
slot_name
plugin
slot_type
datoid
database
temporary
active
active_pid
xmin
catalog_xmin
restart_lsn
confirmed_flush_lsn
wal_status
safe_wal_size
two_phase
two_phase_at
inactive_since
conflicting
invalidation_reason
failover
synced
slotsync_skip_reason
```
本节重点：
| 字段 | 含义 |
| --- | --- |
| `active` / `active_pid` | publisher 上是否有进程持有 slot，通常可关联 walsender。 |
| `restart_lsn` | slot 仍可能需要读取的最老 WAL，WAL retention 以它为核心。 |
| `confirmed_flush_lsn` | client 已确认 flush 的消费边界。 |
| `wal_status` | 从 `restart_lsn` 判断 WAL 是否 reserved / extended / unreserved / lost。 |
| `safe_wal_size` | 在 WAL 保留限制下，以 `restart_lsn` 推导还剩多少空间。 |
| `inactive_since` | slot 变 inactive 的时间。 |
| `conflicting` | logical slot 与 recovery conflict 相关，不是普通 apply conflict。 |
| `invalidation_reason` | slot invalidation 原因，如 WAL removed、horizon、wal_level、idle timeout。 |
源码边界：
```text
slotfuncs.c: pg_get_replication_slots()
  -> 输出 restart_lsn / confirmed_flush_lsn
  -> wal_status 用 GetWALAvailability(restart_lsn)
  -> safe_wal_size 也以 restart_lsn 所在 segment 计算
```
不要把 `safe_wal_size` 当 subscriber 还能落后多少。
它是 publisher slot WAL retention 风险指标。
### 4.2 `pg_stat_replication_slots`
当前字段是：
```text
slot_name
spill_txns
spill_count
spill_bytes
stream_txns
stream_count
stream_bytes
mem_exceeded_count
total_txns
total_bytes
slotsync_skip_count
slotsync_last_skip
stats_reset
```
视图定义会排除 physical slot：
```text
WHERE r.datoid IS NOT NULL
```
这些是 publisher logical decoding / reorder buffer / slot sync 统计。
它们不是 subscriber apply 统计。
源码路径：
```text
reorderbuffer.c: ReorderBufferCheckMemoryLimit()
  -> memExceededCount++
  -> ReorderBufferStreamTXN() 或 ReorderBufferSerializeTXN()
reorderbuffer.c: ReorderBufferSerializeTXN()
  -> spillCount / spillBytes / spillTxns
reorderbuffer.c: ReorderBufferStreamTXN()
  -> streamCount / streamBytes / streamTxns
logical.c: UpdateDecodingStats()
  -> pgstat_report_replslot()
```
诊断含义：
```text
spill_bytes 上升:
  publisher reorder buffer 进入磁盘 spill slow path。
stream_bytes 上升:
  大事务正在 streaming 输出。
mem_exceeded_count 上升:
  logical_decoding_work_mem 被触发。
total_bytes 上升:
  decoding 确实在产出数据。
```
### 4.3 `pg_stat_subscription`
当前字段是：
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
这些字段来自 `launcher.c: pg_stat_get_subscription()`。
它读取 `LogicalRepWorker` 共享槽：
| 视图字段 | 共享槽字段 | 注意点 |
| --- | --- | --- |
| `worker_type` | `type` | apply、parallel apply、table sync、sequence sync。 |
| `pid` | `proc->pid` | 当前 worker PID。 |
| `leader_pid` | `leader_pid` | parallel apply worker 才有。 |
| `relid` | `relid` | table sync worker 才有。 |
| `received_lsn` | `last_lsn` | WALData/keepalive 推进 `last_received` 后更新；PrimaryStatusUpdate 不把 remote_lsn 当作 received_lsn。 |
| `last_msg_send_time` | `last_send_time` | publisher 消息时间。 |
| `last_msg_receipt_time` | `last_recv_time` | subscriber 收到消息时间。 |
| `latest_end_lsn` | `reply_lsn` | keepalive 相关 end LSN。 |
| `latest_end_time` | `reply_time` | keepalive timestamp。 |
当前源码里 `pg_stat_subscription` 没有：
```text
apply_error_count
sync_table_error_count
sync_seq_error_count
conflict counters
apply_lsn
flush_lsn
last_error_lsn
```
关键误区：
```text
received_lsn 不是 apply 完成 LSN。
latest_end_lsn 不是 apply 完成 LSN。
```
在 `worker.c: LogicalRepApplyLoop()` 中，收到 `PqReplMsg_WALData` 后：
```text
UpdateWorkerStats(last_received, send_time, false)
apply_dispatch(&s)
```
统计更新发生在 apply 之前。
### 4.4 `pg_stat_subscription_stats`
当前字段是：
```text
subid
subname
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
源码路径：
```text
pgstat_subscription.c:
  pgstat_report_subscription_error()
    -> 根据 worker type 增加 apply/sync error counter
  pgstat_report_subscription_conflict()
    -> conflict_count[type]++
```
它回答：
```text
subscriber apply/sync worker 是否发生过 ERROR。
当前源码识别的 logical apply conflict 是否增长。
```
它不回答：
```text
当前 worker 卡在哪个锁。
失败事务的 finish_lsn。
publisher decoding 是否在 spill。
网络是否慢。
```
失败事务 LSN 通常要看 server log 中 `apply_error_callback()` 输出的 context。
## 5. 关键数据结构与状态
### 5.1 `ReplicationSlotPersistentData`
`slot.h` 中和本课相关的字段：
```text
restart_lsn:
  oldest LSN that might be required by this replication slot.
confirmed_flush:
  oldest LSN that the client has acked receipt for.
xmin / catalog_xmin:
  slot 保护的数据和 catalog xmin horizon。
invalidated:
  slot 是否失效以及失效原因。
```
slot 结构体中还有 logical slot 的候选状态：
```text
candidate_catalog_xmin
candidate_xmin_lsn
candidate_restart_valid
candidate_restart_lsn
last_saved_confirmed_flush
inactive_since
last_saved_restart_lsn
```
诊断含义：
```text
candidate_restart_lsn:
  decoding 层已经知道 restart_lsn 可以前移到哪里。
candidate_restart_valid:
  只有 confirmed flush 到这个位置后，candidate_restart_lsn 才能生效。
last_saved_restart_lsn:
  persistent slot 计算 WAL removal 下界时可能更保守。
```
### 5.2 `LogicalRepWorker`
`worker_internal.h` 中 `LogicalRepWorker` 保存 worker 共享状态。
本节重点字段：
```text
type
proc
subid
relid
leader_pid
parallel_apply
last_lsn
last_send_time
last_recv_time
reply_lsn
reply_time
oldest_nonremovable_xid
```
`pg_stat_subscription` 只是这些字段的快照。
worker 重启会改变 `pid` 和共享槽状态。
统计 counter 则在 pgstat 中另行保存。
### 5.3 `lsn_mapping`
subscriber 上 apply worker 用 `lsn_mapping` 连接两个世界：
```text
remote_end:
  publisher 事务 end_lsn。
local_end:
  subscriber 本地 commit record LSN。
```
主路径：
```text
apply_handle_commit_internal()
  -> replorigin_xact_state.origin_lsn = commit_data->end_lsn
  -> CommitTransactionCommand()
  -> store_flush_position(commit_data->end_lsn, XactLastCommitEnd)
```
feedback 路径：
```text
get_flush_position()
  -> local_flush = GetFlushRecPtr(NULL)
  -> 只有 local_end <= local_flush
     才把 remote_end 报成 flushpos
```
这说明：
```text
subscriber 本地 WAL flush 慢，也会让 publisher confirmed_flush_lsn 慢。
```
### 5.4 replication origin
`origin.c` 中 `ReplicationState` 保存：
```text
remote_lsn:
  latest commit from the remote side.
local_lsn:
  local commit record LSN.
```
视图 `pg_replication_origin_status` 当前字段是：
```text
local_id
external_id
remote_lsn
local_lsn
```
apply worker 启动时：
```text
replorigin_session_setup(originid, 0)
origin_startpos = replorigin_session_get_progress(false)
```
commit 时：
```text
CommitTransaction()
  -> replorigin_session_advance(replorigin_xact_state.origin_lsn, XactLastRecEnd)
```
ERROR 时：
```text
start_apply() PG_CATCH
  -> replorigin_xact_clear(true)
  -> AbortOutOfAnyTransaction()
  -> pgstat_report_subscription_error()
```
这保证失败事务不会误推进 origin。
## 6. 主流程源码 walkthrough
### 6.1 publisher logical stream start
主流程：
```text
walsender.c: StartLogicalReplication()
  -> ReplicationSlotAcquire()
  -> CreateDecodingContext()
  -> XLogBeginRead(reader, MyReplicationSlot->data.restart_lsn)
  -> sentPtr = MyReplicationSlot->data.confirmed_flush
  -> MyWalSnd->sentPtr = MyReplicationSlot->data.restart_lsn
  -> WalSndLoop(XLogSendLogical)
```
这段解释了一个核心事实：
```text
读 WAL 从 restart_lsn 开始。
发送/确认基线从 confirmed_flush_lsn 开始。
pg_stat_replication 初始 sent_lsn 可能先暴露 restart_lsn，随后由 XLogSendLogical() 推进。
```
如果 slot `active=false`：
```text
先查 worker 是否存在、连接是否失败、subscription 是否 disabled、slot 是否 invalidated。
```
不要先分析 reorder buffer。
### 6.2 publisher decoding
主流程：
```text
walsender.c: XLogSendLogical()
  -> XLogReadRecord()
  -> LogicalDecodingProcessRecord()
  -> sentPtr = reader->EndRecPtr
  -> MyWalSnd->sentPtr = sentPtr
```
decoding context：
```text
logical.c: StartupDecodingContext()
  -> XLogReaderAllocate()
  -> ReorderBufferAllocate()
  -> AllocateSnapshotBuilder()
  -> output plugin callback wrappers
```
reorder buffer pressure：
```text
reorderbuffer.c: ReorderBufferCheckMemoryLimit()
  -> memExceededCount++
  -> stream or spill
```
观测：
```text
pg_stat_replication_slots.spill_bytes
pg_stat_replication_slots.stream_bytes
pg_stat_replication_slots.mem_exceeded_count
pg_stat_replication_slots.total_bytes
```
### 6.3 subscriber receive
主流程：
```text
worker.c: LogicalRepApplyLoop()
  -> walrcv_receive()
  -> if PqReplMsg_WALData:
       read start_lsn/end_lsn/send_time
       last_received = max(last_received, end_lsn)
       UpdateWorkerStats(last_received, send_time, false)
       apply_dispatch(&s)
```
因此：
```text
received_lsn 前进
  只说明消息已经进入 apply worker。
```
它不说明：
```text
target DML 已完成。
本地事务已 commit。
本地 commit WAL 已 flush。
publisher 已收到 feedback。
```
### 6.4 subscriber apply and commit
普通事务：
```text
apply_handle_begin()
  -> set_apply_error_context_xact()
  -> remote_final_lsn = begin_data.final_lsn
  -> in_remote_transaction = true
apply_handle_insert/update/delete()
  -> begin_replication_step()
  -> logicalrep_read_*()
  -> logicalrep_rel_open()
  -> slot_store_data()
  -> ExecSimpleRelationInsert/Update/Delete()
  -> end_replication_step()
apply_handle_commit()
  -> apply_handle_commit_internal()
```
commit 内部：
```text
apply_handle_commit_internal()
  -> replorigin_xact_state.origin_lsn = commit_data->end_lsn
  -> CommitTransactionCommand()
  -> pgstat_report_stat(false)
  -> store_flush_position(commit_data->end_lsn, XactLastCommitEnd)
```
目标端慢可能发生在：
```text
relation lock
tuple lookup
tuple lock
constraint check
unique index maintenance
trigger execution
partition routing
local WAL insert/flush
```
### 6.5 subscriber feedback
主流程：
```text
worker.c: send_feedback(recvpos, force, requestReply)
  -> get_flush_position(&writepos, &flushpos, &have_pending_txes)
  -> if no pending txes:
       flushpos = writepos = recvpos
  -> send StandbyStatusUpdate:
       write = recvpos
       flush = flushpos
       apply = writepos
```
`get_flush_position()` 的核心：
```text
local_flush = GetFlushRecPtr(NULL)
if mapping.local_end <= local_flush:
  flush = mapping.remote_end
else:
  have_pending_txes = true
```
所以：
```text
subscriber 已收到不够。
subscriber 已执行也不够。
必须本地 commit WAL flush 后，publisher confirmed_flush_lsn 才能前进。
```
### 6.6 publisher handles feedback
主流程：
```text
walsender.c: ProcessStandbyReplyMessage()
  -> read writePtr / flushPtr / applyPtr
  -> update MyWalSnd write/flush/apply
  -> if logical slot:
       LogicalConfirmReceivedLocation(flushPtr)
```
`logical.c: LogicalConfirmReceivedLocation()`：
```text
if lsn > confirmed_flush:
  confirmed_flush = lsn
if candidate_xmin_lsn <= lsn:
  update catalog_xmin
if candidate_restart_valid <= lsn:
  restart_lsn = candidate_restart_lsn
  ReplicationSlotSave()
  ReplicationSlotsComputeRequiredLSN()
```
这给出核心判别：
```text
confirmed_flush_lsn 停:
  查 subscriber receive/apply/flush/feedback。
restart_lsn 停但 confirmed_flush_lsn 进:
  查 decoding restart horizon、长事务、prepared、SnapBuild/ReorderBuffer。
```
## 7. 分层诊断方法
### 7.1 基础 SQL
publisher slot：
```sql
SELECT slot_name, active, active_pid,
       restart_lsn, confirmed_flush_lsn,
       wal_status, safe_wal_size,
       inactive_since, invalidation_reason
FROM pg_replication_slots
WHERE slot_type = 'logical';
```
publisher walsender：
```sql
SELECT pid, application_name, state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag, reply_time
FROM pg_stat_replication;
```
logical subscription 中的 `replay_lsn` 是 subscriber feedback 的 `applyPtr`，
不是 physical standby 的 REDO replay 位置。
publisher decoding stats：
```sql
SELECT slot_name,
       spill_txns, spill_count, spill_bytes,
       stream_txns, stream_count, stream_bytes,
       mem_exceeded_count, total_txns, total_bytes,
       stats_reset
FROM pg_stat_replication_slots;
```
subscriber worker：
```sql
SELECT subname, worker_type, pid, leader_pid, relid,
       received_lsn, last_msg_send_time, last_msg_receipt_time,
       latest_end_lsn, latest_end_time
FROM pg_stat_subscription;
```
subscriber stats：
```sql
SELECT subname,
       apply_error_count,
       sync_seq_error_count,
       sync_table_error_count,
       confl_insert_exists,
       confl_update_origin_differs,
       confl_update_exists,
       confl_update_deleted,
       confl_update_missing,
       confl_delete_origin_differs,
       confl_delete_missing,
       confl_multiple_unique_conflicts,
       stats_reset
FROM pg_stat_subscription_stats;
```
subscriber origin：
```sql
SELECT local_id, external_id, remote_lsn, local_lsn
FROM pg_replication_origin_status;
```
worker wait：
```sql
SELECT pid, backend_type, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE backend_type LIKE 'logical replication%';
```
### 7.2 decoding backlog
现象：
```text
publisher WAL 持续产生。
slot active。
sent_lsn 推进慢。
received_lsn 推进慢。
pg_stat_replication_slots spill/stream/mem_exceeded 增长。
```
源码解释：
```text
XLogSendLogical()
  -> LogicalDecodingProcessRecord()
  -> ReorderBuffer accumulates changes
  -> spill or stream when logical_decoding_work_mem is reached
```
优先排查：
```text
大事务。
大量 subtransaction。
TOAST 或大行。
catalog changes。
logical_decoding_work_mem 太小。
publisher IO/CPU。
output plugin 开销。
```
不要误判：
```text
restart_lsn 不动本身不等于 decoding backlog。
```
它可能只是 restart horizon 保守。
### 7.3 transport backlog
现象：
```text
publisher sent_lsn 前进。
subscriber received_lsn 不前进或落后。
last_msg_receipt_time 变旧。
publisher reply_time 变旧。
confirmed_flush_lsn 不前进。
```
源码解释：
```text
walsender 已推进 sentPtr。
subscriber LogicalRepApplyLoop() 没有及时从 walrcv_receive() 读到数据。
```
优先排查：
```text
网络。
subscriber worker 是否退出。
连接认证或 timeout。
publisher walsender socket send。
wal_sender_timeout / wal_receiver_timeout 日志。
```
注意：
```text
latest_end_lsn 来自 keepalive/status。
它不是 apply 完成位置。
```
### 7.4 subscriber apply backlog
现象：
```text
received_lsn 前进。
origin remote_lsn 不前进，或明显落后。
confirmed_flush_lsn 不前进。
apply worker 存在且 wait_event 指向锁、IO、WAL 或 CPU busy。
```
源码解释：
```text
worker 已收到消息。
但还没完成 target executor、本地 commit、本地 WAL flush 和 feedback。
```
拆分：
```text
target executor 慢:
  relation lock、tuple lock、索引、约束、触发器、分区。
local commit/flush 慢:
  CommitTransactionCommand() 之后，local_end 尚未 <= GetFlushRecPtr(NULL)。
```
判断：
```text
received_lsn 前进，origin remote_lsn 不进:
  apply / target executor / local transaction。
origin remote_lsn 前进，confirmed_flush_lsn 不进:
  local WAL flush、feedback 周期、网络回包。
confirmed_flush_lsn 进，restart_lsn 不进:
  不是 subscriber apply backlog 的直接证据。
```
### 7.5 conflict / retry stall
ERROR 类：
```text
CT_INSERT_EXISTS
CT_UPDATE_EXISTS
CT_MULTIPLE_UNIQUE_CONFLICTS
schema mismatch
permission / RLS
replica identity 不足
constraint / trigger ERROR
```
源码路径：
```text
execReplication.c: CheckAndReportConflict(... ERROR ...)
worker.c: start_apply() PG_CATCH
  -> replorigin_xact_clear(true)
  -> AbortOutOfAnyTransaction()
  -> pgstat_report_subscription_error()
```
表现：
```text
apply_error_count 增长。
unique conflict counter 增长。
worker pid 可能反复变化。
confirmed_flush_lsn 停在失败事务前。
日志有 remote xid / finish_lsn / relation context。
```
LOG 后继续类：
```text
CT_UPDATE_ORIGIN_DIFFERS
CT_UPDATE_DELETED
CT_UPDATE_MISSING
CT_DELETE_ORIGIN_DIFFERS
CT_DELETE_MISSING
```
表现：
```text
对应 conflict counter 增长。
apply_error_count 不一定增长。
worker 可继续推进。
业务仍需判断数据偏差。
```
concurrent retry 类：
```text
FindReplTupleInLocalRel()
  -> InitDirtySnapshot()
  -> xwait valid:
       XactLockTableWait()
       retry
  -> table_tuple_lock(... LockWaitBlock ...)
  -> TM_Updated / TM_Deleted:
       LOG retry
       retry
```
表现：
```text
worker 等 target-side transaction lock。
apply_error_count 不增长。
conflict counter 不一定增长。
confirmed_flush_lsn 不进。
```
## 8. 生命周期 / ownership / cleanup
publisher slot：
```text
ReplicationSlotCreate()
ReplicationSlotReserveWal()
ReplicationSlotAcquire()
ReplicationSlotRelease()
ReplicationSlotDropAcquired()
ReplicationSlotSave()
CheckPointReplicationSlots()
```
WAL retention：
```text
ReplicationSlotsComputeRequiredLSN()
  -> 遍历所有有效 slot
  -> 取最小 restart_lsn
  -> persistent slot 可能用 last_saved_restart_lsn 更保守
  -> XLogSetReplicationSlotMinimumLSN()
```
decoding context：
```text
CreateDecodingContext()
  -> StartupDecodingContext()
  -> ReorderBufferAllocate()
  -> AllocateSnapshotBuilder()
FreeDecodingContext()
  -> shutdown callback
  -> ReorderBufferFree()
```
subscriber worker：
```text
launcher.c: ApplyLauncherMain()
  -> logicalrep_worker_launch()
  -> 按 wal_retrieve_retry_interval 节流重启
worker.c: ApplyWorkerMain()
  -> run_apply_worker()
  -> replorigin_session_setup()
  -> walrcv_connect()
  -> walrcv_startstreaming()
  -> start_apply()
```
ERROR cleanup：
```text
start_apply() PG_CATCH
  -> replorigin_xact_clear(true)
  -> AbortOutOfAnyTransaction()
  -> pgstat_report_subscription_error()
```
不变量：
```text
未成功 commit 的远端事务不能推进 replication origin。
```
如果破坏这个不变量：
```text
失败事务可能被永久跳过。
```
## 9. 正确性机制层次
LSN ordering：
```text
remote end_lsn
  -> subscriber target transaction
  -> local commit WAL
  -> local WAL flush
  -> feedback flushPtr
  -> confirmed_flush_lsn
```
slot retention：
```text
restart_lsn 保护重新解码需要的 WAL。
confirmed_flush_lsn 不能单独决定 WAL 是否可删。
```
target executor：
```text
TargetPrivilegesCheck()
ExecConstraints()
ExecInsertIndexTuples()
ExecBR* / ExecAR* triggers
table_tuple_lock()
CommitTransactionCommand()
```
pgstat 粒度：
```text
pg_stat_subscription:
  当前 worker 快照。
pg_stat_subscription_stats:
  subscriber per-subscription counters。
pg_stat_replication_slots:
  publisher per-slot decoding counters。
```
这些层次不能互相替代。
## 10. 错误与边界
### 10.1 timeout
`LogicalRepApplyLoop()` 使用：
```text
WaitLatchOrSocket(..., WAIT_EVENT_LOGICAL_APPLY_MAIN)
```
timeout 后：
```text
超过 wal_receiver_timeout:
  ERROR terminating logical replication worker due to timeout.
超过 wal_receiver_timeout / 2:
  requestReply = true，发送反馈请求。
```
### 10.2 slot invalidation
当前 invalidation cause 包括：
```text
RS_INVAL_WAL_REMOVED
RS_INVAL_HORIZON
RS_INVAL_WAL_LEVEL
RS_INVAL_IDLE_TIMEOUT
```
对应视图字段：
```text
pg_replication_slots.invalidation_reason
```
如果 `wal_status='lost'`：
```text
这是 WAL retention 失败。
不是普通 lag。
```
### 10.3 schema / permission / RLS
`worker.c: TargetPrivilegesCheck()`：
```text
pg_class_aclcheck()
check_enable_rls()
```
RLS enabled 时当前源码 ERROR。
schema mismatch 还可能在：
```text
slot_store_data()
slot_fill_defaults()
logicalrep_rel_open()
check_relation_updatable()
CheckCmdReplicaIdentity()
```
### 10.4 conflict
`conflict.c` 当前 conflict names：
```text
insert_exists
update_origin_differs
update_exists
update_missing
update_deleted
delete_origin_differs
delete_missing
multiple_unique_conflicts
```
unique 类通常 ERROR。
missing/origin/deleted 类在当前 apply path 中多为 LOG 后继续。
诊断时不要把所有 conflict 都当 worker stop。
## 11. 观测与诊断矩阵
| 现象 | 更可能层次 | 下一步 |
| --- | --- | --- |
| `active=false` 且 subscriber 无 worker | worker 未运行 | 查 subscription enabled、日志、launcher retry、apply_error_count。 |
| `spill_bytes` / `mem_exceeded_count` 上升 | decoding backlog | 查大事务、`logical_decoding_work_mem`、publisher IO。 |
| `sent_lsn` 前进，`received_lsn` 不前进 | transport backlog | 查网络、worker 存活、receipt time、timeout。 |
| `received_lsn` 前进，origin `remote_lsn` 不进 | apply backlog | 查 worker wait event、locks、target executor。 |
| origin `remote_lsn` 前进，`confirmed_flush_lsn` 不进 | flush/feedback backlog | 查 subscriber WAL flush、feedback interval、网络回包。 |
| `confirmed_flush_lsn` 前进，`restart_lsn` 不进 | restart horizon | 查长事务、prepared、SnapBuild/ReorderBuffer。 |
| `apply_error_count` 增长 | deterministic apply ERROR | 查日志 error context、schema、权限、unique、RI。 |
| conflict counter 增长但 worker 继续 | LOG conflict | 查 missing/origin/deleted 类业务偏差。 |
| wait_event_type = `Lock` | target-side contention | 查 blocker 和 subscriber 本地写入。 |
| wait_event = `LogicalApplyMain` | 主循环等待 | 结合 received_lsn 判断是空闲还是断流。 |
逻辑复制相关 wait events：
```text
LogicalApplyMain:
  apply worker 主循环等待 socket/latch/timeout。
LogicalLauncherMain:
  launcher 主循环等待。
LogicalParallelApplyMain:
  parallel apply worker 主循环等待。
LogicalApplySendData:
  leader apply worker 等 shm_mq 发送给 parallel apply worker。
LogicalParallelApplyStateChange:
  leader 等 parallel apply worker transaction state。
LogicalRepWorker:
  LWLock，读写 logical replication worker shared state。
```
目标端锁等待通常显示为普通 lock wait。
不要期望所有 apply stall 都显示 logical apply wait event。
## 12. 常见误区
误区 1：
```text
restart_lsn 落后 = subscriber 没应用。
```
正确：
```text
restart_lsn 是重新解码和 WAL retention 起点。
```
误区 2：
```text
confirmed_flush_lsn 不动 = subscriber 没收到。
```
正确：
```text
subscriber 可能已收到，但还没 apply、commit、flush 或 feedback。
```
误区 3：
```text
received_lsn 前进 = target table 已追上。
```
正确：
```text
received_lsn 在 apply_dispatch() 前更新。
```
误区 4：
```text
latest_end_lsn = apply LSN。
```
正确：
```text
当前源码中 latest_end_lsn 来自 worker.reply_lsn，主要和 keepalive/status end LSN 相关。
```
误区 5：
```text
LogicalApplyMain = 正在 apply 很慢。
```
正确：
```text
它是主循环等待点；target-side 慢通常表现为 Lock、IO、WAL、LWLock 等。
```
误区 6：
```text
conflict counter 增长就一定 worker 停。
```
正确：
```text
unique conflict 类 ERROR；missing/origin/deleted 类当前多为 LOG 后继续。
```
## 13. 课堂实验
### 实验 1：target-side lock wait
subscriber：
```sql
BEGIN;
UPDATE replicated_table SET v = v + 1 WHERE id = 1;
-- 不提交
```
publisher：
```sql
UPDATE replicated_table SET v = v + 10 WHERE id = 1;
```
观察：
```sql
SELECT pid, wait_event_type, wait_event, state
FROM pg_stat_activity
WHERE backend_type LIKE 'logical replication%';
```
预期：
```text
apply worker 等目标端事务。
received_lsn 可能前进。
confirmed_flush_lsn 不前进。
```
源码回扣：
```text
FindReplTupleInLocalRel()
  -> XactLockTableWait()
  -> retry
```
### 实验 2：decoding spill
publisher：
```sql
ALTER SYSTEM SET logical_decoding_work_mem = '64kB';
SELECT pg_reload_conf();
BEGIN;
INSERT INTO replicated_table SELECT g, repeat('x', 1000)
FROM generate_series(1, 50000) AS g;
COMMIT;
```
观察：
```sql
SELECT slot_name, spill_count, spill_bytes,
       mem_exceeded_count, total_txns, total_bytes
FROM pg_stat_replication_slots;
```
预期：
```text
spill/mem exceeded 增长说明 publisher decoding/reorder 层进入 slow path。
仍需另看 received_lsn、origin remote_lsn 和 confirmed_flush_lsn 判断 apply 是否慢。
```
### 实验 3：unique conflict ERROR
subscriber：
```sql
INSERT INTO replicated_table(id, v) VALUES (100, 1);
```
publisher：
```sql
INSERT INTO replicated_table(id, v) VALUES (100, 2);
```
观察：
```sql
SELECT subname, apply_error_count,
       confl_insert_exists, confl_multiple_unique_conflicts
FROM pg_stat_subscription_stats;
```
预期：
```text
apply_error_count 增长。
unique conflict counter 增长。
worker 退出或重启。
confirmed_flush_lsn 停在失败事务前。
```
## 14. 讨论题
1. 为什么 logical walsender 从 `restart_lsn` 读 WAL，却用 `confirmed_flush_lsn` 作为发送确认基线？
2. `confirmed_flush_lsn` 前进但 `restart_lsn` 不动时，为什么不能直接归因 subscriber apply 慢？
3. `received_lsn` 在 `apply_dispatch()` 前更新，对诊断 apply backlog 有什么影响？
4. 为什么 `send_feedback()` 要等本地 commit WAL flush 后才推进 flushpos？
5. `confl_update_missing` 增长但 `apply_error_count` 不增长，说明当前源码采用了什么行为？
6. apply worker 等 `LogicalApplyMain` 与等 `transactionid` lock，分别停在哪个边界？
7. subscriber 本地直接写复制表为什么会造成 replication lag 或 conflict？
8. slot `wal_status='lost'` 和普通 apply lag 的根本区别是什么？
## 15. 本节小结
本节主链路：
```text
publisher restart_lsn
  -> logical decoding / reorder buffer
  -> walsender sent_lsn
  -> subscriber received_lsn
  -> target executor / local commit
  -> local WAL flush
  -> feedback flushPtr
  -> publisher confirmed_flush_lsn
  -> candidate restart satisfied
  -> restart_lsn advances
```
核心判断：
```text
pg_replication_slots:
  publisher slot 和 WAL retention。
pg_stat_replication_slots:
  publisher decoding/reorder stats。
pg_stat_subscription:
  subscriber worker 当前 receive/keepalive 状态。
pg_stat_subscription_stats:
  subscriber apply/sync ERROR 与 conflict counters。
pg_stat_activity wait_event:
  当前 worker 等待点。
pg_replication_origin_status:
  subscriber 已提交的远端事务进度。
```
可迁移规律：
```text
raw LSN 不是语义。
LSN + owner + lifecycle + durability boundary + feedback direction 才是语义。
```
最终诊断顺序：
```text
先定位边界：
  解码、传输、apply、flush feedback、restart horizon。
再找原因：
  reorder spill、网络、target executor、local WAL flush、schema/conflict/retry。
最后看风险：
  restart_lsn 与 wal_status/safe_wal_size 决定 WAL retention 风险。
```
版本边界：
本课字段和函数名按当前本地 `/home/nail/postgres` master 源码核对。
旧版本资料中的字段名、conflict 行为和 wait event 名称不应直接套用。
