# PostgreSQL 复制、恢复与解码的故障边界图

## 课程定位

前置知识：
已经理解 physical streaming、replication slot、startup recovery、
timeline、logical decoding、output plugin 和 subscription apply 的基本路径。

本节唯一主问题：

```text
面对延迟、WAL 缺失、slot 膨胀、apply 停滞或 timeline 错误时，
如何先判断问题属于 physical streaming、recovery、timeline、
logical decoding、output plugin，还是 subscription apply？
```

核心矛盾：

```text
现场排障希望一个 lag 数字直接给出原因
  vs
PostgreSQL 把同一段 WAL 的生命周期拆给多个 owner 推进。
```

这些 owner 包括：

```text
primary walsender
standby walreceiver
startup recovery process
timeline history reader
replication slot
logical decoder
output plugin
subscription apply worker
replication origin
```

学完后应能判断：

```text
一个症状首先属于哪个 owner；
哪个视图读到了这个 owner 的状态；
哪个 LSN 是发送、接收、flush、replay、保留、确认或 apply 边界；
什么时候该看 logs、pg_waldump、pg_controldata；
哪些现象只是相邻模块传播过来的压力，不是本层根因。
```

本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

本目录前面按机制拆开：

```text
01-05:
  physical replication connection、walsender、walreceiver、sync rep、timeout。

06-10:
  slot 类型、restart_lsn、xmin、持久化、监控。

11-19:
  startup process、REDO、Hot Standby、recovery target、timeline。

20-28:
  reorder buffer、logical decoding、snapshot、confirmed_flush。

29-37:
  output plugin、pgoutput、subscription apply、origin、conflict。

38-41:
  physical lag、logical lag、WAL retention、standby feedback bloat。
```

本节不是复述这些课。
本节只回答现场第一步：

```text
当前卡点在哪一层？
```

如果第一步分层错误，后续证据会互相污染：

```text
replay_lsn 不动，不等于 primary 网络慢；
restart_lsn 旧，不等于 subscriber 已经没消费；
WAL 文件存在，不等于 timeline history 接受它；
subscriber 没某表变化，不一定是 apply 停滞，也可能是 pgoutput 没发布。
```

## 2. 核心矛盾与一句话运行模型

physical 路径：

```text
primary WAL flushed
  -> walsender sent_lsn
  -> walreceiver written_lsn
  -> walreceiver flushed_lsn
  -> startup process replay_lsn
```

logical 路径：

```text
publisher WAL
  -> slot restart_lsn 保护可重读下界
  -> logical decoder 从 WAL 重建事务级 change
  -> output plugin 编码和过滤
  -> subscriber apply worker 接收消息
  -> 本地事务 commit 后推进 replication origin
  -> feedback 回到 publisher
  -> confirmed_flush_lsn 前进
```

timeline 横切 recovery：

```text
WAL bytes 存在
  -> 还必须属于 recovery target timeline 的 history
  -> startup process 才能 replay
```

slot 横切 physical / logical：

```text
slot 负责保留 WAL / xmin
  -> 它保护未来还能追赶或重建
  -> 它不证明消费者已经处理成功
```

核心不变量：

```text
raw LSN 不是语义；
LSN + owner + lifecycle + feedback source 才是语义。
```

## 3. 分层 decision tree

### 3.1 第一问：链路形态

```text
physical standby 延迟？
  -> 先看 pg_stat_replication、pg_stat_wal_receiver、pg_stat_recovery。

logical subscription 延迟？
  -> 先看 pg_replication_slots、pg_stat_subscription、origin、logs。

WAL 缺失或 timeline 错误？
  -> 先看 startup logs、pg_controldata、.history、pg_waldump -t。

pg_wal 膨胀？
  -> 先看 pg_replication_slots.restart_lsn、wal_status、safe_wal_size。

apply 停滞？
  -> 先看 subscriber worker、origin、subscription stats、apply logs。
```

不要先解释 `lag`。
先找 owner。

### 3.2 physical branch

```text
sent_lsn 不前进
  -> upstream walsender 或 upstream WAL flush。

sent_lsn 前进，write_lsn 不前进
  -> standby walreceiver、network、feedback。

write_lsn 前进，flush_lsn 不前进
  -> standby WAL fsync / storage。

flush_lsn 前进，replay_lsn 不前进
  -> startup recovery、REDO、conflict、pause、apply delay。

replay_lsn 前进，但 standby 查询仍旧
  -> snapshot、query routing、application read point，不是 streaming 本身。
```

### 3.3 recovery / timeline branch

```text
wait event = RecoveryRetrieveRetryInterval
  -> pg_wal、archive、streaming 都暂时没有可用 WAL。

wait event = RecoveryWalStream
  -> startup process 正在等 streaming WAL。

wait event = RecoveryPause
  -> recovery 被人为 pause 或停在 target。

wait event = RecoveryApplyDelay
  -> recovery_min_apply_delay 之类的延迟。

logs 出现 requested timeline / unexpected timeline ID
  -> 先修 timeline history，再谈 lag。

WAL 文件存在但 pg_waldump -t 指定 TLI 不匹配
  -> 文件存在，不代表属于当前 expected history。
```

### 3.4 logical branch

```text
confirmed_flush_lsn 不前进
  -> consumer 没确认。
  -> 继续查 subscriber 是否收到、apply 是否 commit、feedback 是否返回。

confirmed_flush_lsn 前进，restart_lsn 不前进
  -> decoder 仍需要旧 WAL。
  -> 查 long transaction、snapbuild、reorder buffer、prepared / streaming txn。

restart_lsn 很旧，spill_bytes 增长
  -> logical decoder / reorder buffer 压力。

subscriber received_lsn 前进，origin 不前进
  -> apply worker 本地事务边界。

subscriber received_lsn 不前进
  -> 回到 publisher logical walsender、decoder、network。
```

### 3.5 plugin / apply branch

```text
publisher 没发某张表
  -> publication、row filter、column list、partition root、origin filter。

publisher logs 报 output plugin ERROR
  -> output plugin callback 边界。

subscriber schema mismatch
  -> pgoutput 已发送 relation metadata，但 subscriber 本地 relation 不接受。

pg_stat_subscription.pid 为空
  -> launcher 没启动 worker，或 worker 反复退出。

apply_error_count 增长
  -> apply worker ERROR。

同一 finish LSN 反复失败
  -> origin 未推进，重启后重复同一远端事务。
```

## 4. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/catalog/system_views.sql` | `pg_stat_replication`、`pg_stat_wal_receiver`、`pg_replication_slots`、`pg_stat_replication_slots`、`pg_stat_subscription`、`pg_stat_subscription_stats`。 |
| 2 | `src/backend/utils/activity/wait_event_names.txt` | replication、recovery、slot、snapbuild、reorder、apply wait event 名称。 |
| 3 | `src/backend/replication/walsender.c` | `XLogSendPhysical()`、`XLogSendLogical()`、`ProcessStandbyReplyMessage()`。 |
| 4 | `src/include/replication/walsender_private.h` | `WalSnd.sentPtr`、`write`、`flush`、`apply`、lag。 |
| 5 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()`、`XLogWalRcvWrite()`、`XLogWalRcvFlush()`、reply。 |
| 6 | `src/backend/replication/walreceiverfuncs.c` | `RequestXLogStreaming()`、`GetWalRcvFlushRecPtr()`、`GetWalRcvWriteRecPtr()`。 |
| 7 | `src/include/replication/walreceiver.h` | `WalRcvData.writtenUpto`、`flushedUpto`、`latestWalEnd`。 |
| 8 | `src/backend/access/transam/xlogrecovery.c` | `PerformWalRecovery()`、`ReadRecord()`、`WaitForWALToBecomeAvailable()`、timeline checks。 |
| 9 | `src/backend/access/transam/xlog.c` | end-of-recovery、new timeline、control file、WAL cleanup。 |
| 10 | `src/backend/access/transam/timeline.c` | `.history` 文件读写。 |
| 11 | `src/include/replication/slot.h` | `restart_lsn`、`confirmed_flush`、`xmin`、`catalog_xmin`、invalidation。 |
| 12 | `src/backend/replication/slot.c` | slot 保留、required LSN / xmin、checkpoint save、invalidation。 |
| 13 | `src/backend/replication/slotfuncs.c` | `pg_get_replication_slots()`、`wal_status`、`safe_wal_size`。 |
| 14 | `src/backend/replication/logical/logical.c` | decoding context、`LogicalConfirmReceivedLocation()`、candidate restart / xmin。 |
| 15 | `src/backend/replication/logical/decode.c` | `LogicalDecodingProcessRecord()`、commit / abort 分发。 |
| 16 | `src/backend/replication/logical/snapbuild.c` | running xacts、consistent snapshot、restart candidate。 |
| 17 | `src/backend/replication/logical/reorderbuffer.c` | transaction reassembly、spill、stream stats。 |
| 18 | `src/backend/replication/pgoutput/pgoutput.c` | callback、publication、schema、row filter、origin filter。 |
| 19 | `src/backend/replication/logical/worker.c` | apply loop、local xact、feedback、origin、ERROR cleanup。 |
| 20 | `src/backend/replication/logical/launcher.c` | launcher shared state、worker launch、`pg_stat_get_subscription()`。 |
| 21 | `src/backend/replication/logical/origin.c` | replication origin progress。 |
| 22 | `src/backend/utils/activity/pgstat_replslot.c` | logical slot spill / stream stats 累计。 |
| 23 | `src/backend/utils/activity/pgstat_subscription.c` | subscription error / conflict stats 累计。 |

阅读顺序：

```text
先读视图和 wait event，确认能看到什么；
再按 symptom 选择 owner 源码；
最后读这个 owner 的 cleanup / retry / ERROR 路径。
```

## 5. symptom -> state -> owner -> source file

### 5.1 physical streaming

| symptom | state | owner | source |
| --- | --- | --- | --- |
| primary 上没有 standby 行 | 无 active walsender | replication connection / walsender | `walsender.c` |
| `sent_lsn` 不前进 | `WalSnd.sentPtr` 停 | upstream walsender | `XLogSendPhysical()` |
| `sent_lsn` 前进，`write_lsn` 不动 | standby 没回 write | walreceiver / network | `XLogWalRcvWrite()`、`XLogWalRcvSendReply()` |
| `write_lsn` 前进，`flush_lsn` 不动 | fsync 边界停 | walreceiver / storage | `XLogWalRcvFlush()` |
| `flush_lsn` 前进，`replay_lsn` 不动 | redo 没完成 | startup process | `ApplyWalRecord()` |
| sync commit 等 standby | sync feedback 不满足 | walsender / syncrep | `ProcessStandbyReplyMessage()`、`syncrep.c` |

physical streaming 负责传 WAL bytes 和反馈位置。
它不负责 output plugin，也不负责 subscriber 本地约束。

### 5.2 recovery / timeline

| symptom | state | owner | source |
| --- | --- | --- | --- |
| standby 等 WAL | `RecoveryRetrieveRetryInterval` | startup process | `WaitForWALToBecomeAvailable()` |
| standby 等 stream | `RecoveryWalStream` | startup / walreceiver | `WaitForWALToBecomeAvailable()` |
| recovery pause | `pause_state` | startup process | `recoveryPausesHere()` |
| apply delay | `RecoveryApplyDelay` | startup process | `recoveryApplyDelay()` |
| snapshot conflict | conflict wait / cancellation | startup + standby queries | `standby.c` |
| requested timeline error | target history 不匹配 | startup recovery | `xlogrecovery.c` |
| unexpected timeline ID | WAL record 不在 expected history | startup recovery | `ReadRecord()`、`tliOfPointInHistory()` |

recovery 负责 replay 和 timeline acceptance。
WAL 文件存在但 history 不接受时，不能继续 redo。

### 5.3 slot / retention

| symptom | state | owner | source |
| --- | --- | --- | --- |
| `pg_wal` 增长 | min `restart_lsn` 很旧 | slot + checkpoint | `ReplicationSlotsComputeRequiredLSN()` |
| `wal_status='extended'` | 超过普通保留但仍保留 | slot / WAL recycling | `pg_get_replication_slots()` |
| `wal_status='lost'` | 所需 WAL removed | slot invalidation | `InvalidateObsoleteReplicationSlots()` |
| `invalidation_reason='wal_removed'` | WAL 保留失败 | slot | `RS_INVAL_WAL_REMOVED` |
| `invalidation_reason='rows_removed'` | 所需行版本消失 | slot / vacuum horizon | `RS_INVAL_HORIZON` |
| `catalog_xmin` 不动 | historical catalog 被保护 | logical slot / snapbuild | `LogicalIncreaseXminForSlot()` 相关路径 |
| `inactive_since` 很久 | consumer 长期未持有 slot | slot lifecycle | `ReplicationSlotRelease()` |

slot 是保留边界，不是消费成功证明。

### 5.4 logical decoding / output plugin

| symptom | state | owner | source |
| --- | --- | --- | --- |
| `confirmed_flush_lsn` 不动 | consumer 未确认 | logical walsender / consumer | `LogicalConfirmReceivedLocation()` |
| `confirmed_flush_lsn` 前进，`restart_lsn` 不动 | decoder 仍需旧 WAL | snapbuild / reorder | `LogicalIncreaseRestartDecodingForSlot()` |
| `spill_bytes` 增长 | reorder buffer spill | logical decoder | `reorderbuffer.c` |
| `stream_bytes` 增长 | 大事务 streaming | reorder buffer / output plugin | `ReorderBufferReplay()` |
| publisher 不发某表 | publication / filter | pgoutput | `get_rel_sync_entry()`、`pgoutput_change()` |
| plugin 报 ERROR | callback ERROR | output plugin | `OutputPluginCallbacks`、`pgoutput.c` |

logical decoding 负责 WAL 到事务级 change。
output plugin 负责输出什么和怎么输出。
二者都不负责 subscriber 本地 DML 成功。

### 5.5 subscription apply

| symptom | state | owner | source |
| --- | --- | --- | --- |
| `pg_stat_subscription.pid` 为 NULL | worker 未运行 | launcher / worker | `ApplyLauncherMain()` |
| `received_lsn` 不前进 | 没收到 publisher 消息 | apply worker / upstream | `LogicalRepApplyLoop()` |
| `received_lsn` 前进，origin 不前进 | 本地 apply 未 commit | apply worker | `apply_handle_commit_internal()` |
| `apply_error_count` 增长 | apply ERROR | apply worker | `start_apply()` |
| conflict counters 增长 | conflict diagnostics | apply worker / conflict.c | `ReportApplyConflict()` |
| 重启反复同一 LSN | origin 未推进 | apply worker / origin | `replorigin_session_get_progress()` |

apply worker 只在本地事务成功 commit 后推进 origin。
ERROR 后清 origin xact state，重启后通常会重放同一远端事务。

## 6. 关键状态与视图边界

### 6.1 `pg_stat_replication`

`system_views.sql` 中该视图来自：

```text
pg_stat_get_activity(NULL)
  JOIN pg_stat_get_wal_senders()
```

关键列：

| column | owner | meaning |
| --- | --- | --- |
| `state` | walsender | startup / catchup / streaming / stopping 等。 |
| `sent_lsn` | walsender | `WalSnd.sentPtr`，已发送边界。 |
| `write_lsn` | standby feedback | standby 回报写入位置。 |
| `flush_lsn` | standby feedback | standby 回报 fsync 位置。 |
| `replay_lsn` | standby feedback | standby 回报 replay 位置。 |
| `write_lag` | lag tracker | primary WAL flush 样本到 write ack。 |
| `flush_lag` | lag tracker | primary WAL flush 样本到 flush ack。 |
| `replay_lag` | lag tracker | primary WAL flush 样本到 apply ack。 |
| `backend_xmin` | walsender backend | standby feedback 对 vacuum horizon 的线索。 |
| `sync_state` | sync rep | async / potential / sync / quorum。 |

边界：

```text
这是 upstream 对 downstream 的视角。
它看不到 standby redo 内部，也看不到 logical subscriber 本地冲突。
```

### 6.2 `pg_stat_wal_receiver`

该视图来自 `pg_stat_get_wal_receiver()`。

| column | owner | meaning |
| --- | --- | --- |
| `status` | walreceiver | connecting / streaming / waiting 等。 |
| `receive_start_lsn` | startup request | 本轮 streaming 起点。 |
| `receive_start_tli` | startup request | 本轮 streaming timeline。 |
| `written_lsn` | walreceiver | `XLogWalRcvWrite()` 后推进。 |
| `flushed_lsn` | walreceiver | `XLogWalRcvFlush()` 后推进。 |
| `received_tli` | walreceiver | 当前收到 WAL 的 TLI。 |
| `latest_end_lsn` | upstream message | 上游声明的 WAL end。 |
| `last_msg_send_time` | upstream clock | 最近消息发送时间。 |
| `last_msg_receipt_time` | standby clock | standby 收到时间。 |
| `slot_name` | walreceiver | upstream physical slot 名。 |

`written_lsn` 不是 durable。
`flushed_lsn` 才是 walreceiver 已 fsync 的边界。
startup replay 仍是下一层。

### 6.3 `pg_replication_slots`

当前源码中 `pg_get_replication_slots()` 暴露这些诊断字段：

```text
slot_name / plugin / slot_type
active / active_pid
xmin / catalog_xmin
restart_lsn / confirmed_flush_lsn
wal_status / safe_wal_size
inactive_since / conflicting / invalidation_reason
failover / synced / slotsync_skip_reason
```

字段语义：

```text
restart_lsn:
  slot 可能还需要的最旧 WAL。

confirmed_flush_lsn:
  logical client acked receipt 的边界。

wal_status:
  reserved / extended / unreserved / lost 等 WAL 可用性状态。

safe_wal_size:
  slot 未 lost 且配置 max_slot_wal_keep_size 时才有意义。

invalidation_reason:
  wal_removed、rows_removed、wal_level_insufficient、idle_timeout 等。
```

### 6.4 `pg_stat_replication_slots`

该视图只排 logical slots：

```text
FROM pg_replication_slots r,
     LATERAL pg_stat_get_replication_slot(slot_name) s
WHERE r.datoid IS NOT NULL
```

核心列：

```text
spill_txns / spill_count / spill_bytes
stream_txns / stream_count / stream_bytes
mem_exceeded_count
total_txns / total_bytes
slotsync_skip_count / slotsync_last_skip
```

这些是 decoder、reorder buffer、slot sync 压力线索，
不能单独证明 subscriber apply 慢。

### 6.5 `pg_stat_subscription` 与 origin

`pg_stat_subscription` 来自：

```text
pg_subscription
  LEFT JOIN pg_stat_get_subscription(NULL)
```

关键列：

```text
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

`received_lsn` 表示 worker 收到消息。
已成功 apply 的 durable 进度要看 replication origin，
也就是 `pg_replication_origin_status`。

### 6.6 `pg_stat_subscription_stats`

核心列：

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
```

这些是累计统计。
当前卡住的 relation、remote xid、finish LSN 通常要看日志。

### 6.7 logs、`pg_waldump`、`pg_controldata`

logs 负责给语义错误：

```text
requested WAL segment ...
requested timeline ...
unexpected timeline ID ...
replication slot ... invalidated
logical replication worker ... exited
conflict detected on relation ...
```

`pg_waldump` 负责看 WAL record：

```text
pg_waldump -t TLI -s LSN -e LSN
pg_waldump --stats
pg_waldump -r RMGR
```

`pg_controldata` 负责看 control file：

```text
Latest checkpoint location
Latest checkpoint's TimeLineID
Latest checkpoint's REDO location
Minimum recovery ending location
Min recovery ending loc's timeline
Database cluster state
```

`pg_waldump` 证明 WAL 内容。
`pg_controldata` 证明本 data directory 的控制状态。
`.history` 证明 timeline parent 和 fork LSN。

### 6.8 wait events

本节重点 wait events：

```text
physical:
  WalSenderMain
  WalSenderWaitForWAL
  WalSenderWriteData
  WalReceiverMain
  WalReceiverWaitStart
  LibpqWalReceiverConnect
  LibpqWalReceiverReceive

recovery:
  RecoveryWalStream
  RecoveryRetrieveRetryInterval
  RecoveryPause
  RecoveryApplyDelay
  RecoveryConflictSnapshot
  RecoveryConflictTablespace
  RestoreCommand
  RecoveryEndCommand

slot / decoding:
  ReplicationSlotControl
  ReplicationSlotAllocation
  ReplicationSlotIO
  ReplicationSlotRead / Write / Sync
  SnapbuildRead / Write / Sync
  ReorderBufferRead / Write

logical apply:
  LogicalLauncherMain
  LogicalApplyMain
  LogicalSyncData
  LogicalApplySendData
  applytransaction
```

wait event 是当前等待点，不是完整根因。

## 7. 主流程源码 walkthrough

### 7.1 physical WAL

主库发送：

```text
XLogSendPhysical()
  -> WalSndWaitForWal()
  -> read WAL bytes
  -> pq_putmessage_noblock()
  -> sentPtr = endptr
  -> MyWalSnd->sentPtr = sentPtr
```

备库接收：

```text
WalReceiverMain()
  -> XLogWalRcvProcessMsg()
  -> XLogWalRcvWrite()
  -> XLogWalRcvFlush()
  -> XLogWalRcvSendReply()
```

主库处理反馈：

```text
ProcessStandbyReplyMessage()
  -> update write / flush / apply
  -> update lag tracker
```

边界：

```text
sent_lsn 不保证 standby 收到；
written_lsn 不保证 fsync；
flushed_lsn 不保证 replay；
replay_lsn 不代表 logical apply。
```

### 7.2 recovery 和 timeline

startup 主循环：

```text
PerformWalRecovery()
  -> ReadRecord()
  -> WaitForWALToBecomeAvailable()
  -> recovery target / pause / delay checks
  -> ApplyWalRecord()
```

WAL 来源顺序可以在 `WaitForWALToBecomeAvailable()` 中看到：

```text
本地 pg_wal
  -> archive restore
  -> walreceiver streaming
  -> retry / wait
```

timeline acceptance：

```text
expectedTLEs 描述目标 timeline 的 history；
ReadRecord() 检查 page TLI；
ApplyWalRecord() 处理 shutdown checkpoint / end-of-recovery timeline switch。
```

### 7.3 slot 和 logical decoding

slot 保留：

```text
ReplicationSlotsComputeRequiredLSN()
  -> min restart_lsn
  -> XLogSetReplicationSlotMinimumLSN()

ReplicationSlotsComputeRequiredXmin()
  -> xmin / catalog_xmin
  -> ProcArraySetReplicationSlotXmin()
```

logical decoder：

```text
StartLogicalReplication()
  -> CreateDecodingContext()
  -> XLogBeginRead(ctx->reader, slot->data.restart_lsn)
  -> WalSndLoop(XLogSendLogical)

XLogSendLogical()
  -> XLogReadRecord()
  -> LogicalDecodingProcessRecord()
  -> ReorderBufferQueueChange()
  -> ReorderBufferCommit()
  -> output plugin callbacks
```

确认：

```text
ProcessStandbyReplyMessage()
  -> LogicalConfirmReceivedLocation(flushPtr)
  -> confirmed_flush 只向前
  -> 覆盖 candidate_xmin_lsn 后推进 catalog_xmin
  -> 覆盖 candidate_restart_valid 后推进 restart_lsn
```

### 7.4 output plugin 和 apply

`pgoutput`：

```text
_PG_output_plugin_init()
  -> 注册 begin / change / commit / stream / origin filter callbacks

pgoutput_change()
  -> get_rel_sync_entry()
  -> publication / partition root / column list / row filter
  -> maybe_send_schema()
  -> logicalrep_write_insert/update/delete()
```

apply worker：

```text
run_apply_worker()
  -> ReplicationOriginNameForLogicalRep()
  -> replorigin_session_setup()
  -> origin_startpos = replorigin_session_get_progress(false)
  -> LogicalRepApplyLoop(origin_startpos)
```

commit 与 ERROR：

```text
apply_handle_commit_internal()
  -> CommitTransactionCommand()
  -> store_flush_position(commit_data->end_lsn, XactLastCommitEnd)

start_apply() PG_CATCH
  -> replorigin_xact_clear(true)
  -> pgstat_report_subscription_error()
  -> worker exit or DisableSubscriptionAndExit()
```

不变量：

```text
没有成功 commit 的远端事务不能推进 origin。
```

## 8. 生命周期 / ownership / cleanup

| object | owner | cleanup / error |
| --- | --- | --- |
| `WalSnd` | walsender process | `WalSndKill()` 清 shared slot。 |
| `WalRcvData` | walreceiver + startup | walreceiver exit 后 shared state 回到等待/停止。 |
| recovery state | startup process | end-of-recovery 更新 control file 和 timeline。 |
| slot shared state | slot holder via `active_proc` | `ReplicationSlotRelease()` 清 active；persistent data 保留。 |
| slot control file | slot subsystem | `ReplicationSlotSave()` / checkpoint；ERROR 不随 MemoryContext 丢。 |
| decoding context | logical walsender / SQL decoding backend | session-local cleanup；未确认位置不推进。 |
| reorder buffer | decoder backend | memory reset + spill cleanup；stats 汇总到 replslot stats。 |
| pgoutput private state | output plugin lifecycle | `pgoutput_shutdown()` / memory context reset。 |
| apply worker state | logical worker process | worker exit detach；launcher 后续重启。 |
| origin xact state | apply worker | ERROR 时 `replorigin_xact_clear(true)`。 |

核心区分：

```text
MemoryContext 管 session-local 内存；
slot 持久化管跨重启进度；
origin 管 subscriber 已应用远端 LSN；
wait event 管当前等待点；
invalidation 管 slot 是否仍可用。
```

## 9. 错误路径与边界案例

### 9.1 WAL 缺失

physical standby 缺 WAL：

```text
startup process 需要 RecPtr；
pg_wal 没有；
restore_command 没取到；
streaming 也不能提供；
WaitForWALToBecomeAvailable() retry。
```

logical slot lost：

```text
slot restart_lsn 早于可用 WAL；
pg_replication_slots.wal_status='lost'；
invalidation_reason 可能是 wal_removed。
```

timeline 不接受：

```text
WAL segment 可能存在；
但不在 expectedTLEs；
recovery 拒绝 replay。
```

分诊顺序：

```text
1. logs 看缺 segment 还是 timeline mismatch。
2. pg_replication_slots 看 slot 是否 lost。
3. pg_controldata 看 checkpoint / min recovery point TLI。
4. pg_waldump -t 验证具体 WAL record。
```

### 9.2 slot 膨胀

传播：

```text
consumer 停滞或 decoder protection 不前进
  -> restart_lsn 旧
  -> required LSN 降低
  -> pg_wal 增长
  -> safe_wal_size 变小
  -> wal_status 可能走向 lost
```

分层：

| 组合 | 更可能 owner |
| --- | --- |
| `confirmed_flush_lsn` 不动，subscriber `received_lsn` 不动 | publisher decoder / network / worker 未收。 |
| `confirmed_flush_lsn` 不动，subscriber `received_lsn` 前进 | subscriber apply / feedback。 |
| `confirmed_flush_lsn` 前进，`restart_lsn` 不动 | snapbuild / long transaction / reorder buffer。 |
| `catalog_xmin` 不动 | historical catalog snapshot 保护。 |
| physical slot `restart_lsn` 旧 | standby flush feedback 或 walreceiver startpoint。 |

### 9.3 restart vs confirmed

```text
restart_lsn:
  下次为了正确重建状态，最早可能要读哪段 WAL。

confirmed_flush_lsn:
  logical consumer 已确认安全接收到哪里。
```

`confirmed_flush_lsn` 前进不保证 `restart_lsn` 立即前进。
`restart_lsn` 旧也不一定说明 subscriber 没收到。

### 9.4 apply conflict

当前源码中 conflict 不全等价：

```text
ReportApplyConflict() 会累加 subscription conflict stats；
调用者决定 elevel；
部分 conflict LOG 后继续；
部分 ERROR 会 abort 当前远端事务。
```

常见分类：

| case | owner | effect |
| --- | --- | --- |
| duplicate key / unique conflict | local executor / apply | 通常 ERROR。 |
| UPDATE / DELETE missing row | apply conflict handling | 当前源码可 LOG 并继续。 |
| origin differs | apply / origin | 当前源码可 LOG 并继续。 |
| schema / type mismatch | relation map / executor | ERROR。 |
| permission / RLS / trigger / constraint | local executor | ERROR。 |

ERROR 后：

```text
本地事务 abort；
origin 不推进；
apply_error_count 增长；
worker 退出或 disable subscription；
重启后通常重放同一远端事务。
```

### 9.5 standby feedback bloat

physical feedback：

```text
standby 为减少 Hot Standby conflict，
把 xmin 反馈给 primary；
primary 可从 pg_stat_replication.backend_xmin 找线索。
```

logical slot：

```text
logical decoding 需要历史 catalog tuple；
slot.catalog_xmin 保护这些版本。
```

区分：

```text
backend_xmin:
  physical standby feedback。

catalog_xmin:
  logical decoding historical catalog。
```

## 10. 成本、资源与跨模块传播

WAL bandwidth：

```text
foreground WAL generation
  -> WAL flush
  -> walsender read / send
  -> network
  -> walreceiver write / sync
  -> startup redo
```

WAL retention：

```text
restart_lsn 旧
  -> WAL recycling 下界降低
  -> pg_wal 增长
  -> checkpoint / archive 压力
  -> max_slot_wal_keep_size 后可能 lost
```

decoder pressure：

```text
大事务
  -> ReorderBufferTXN change 增长
  -> logical_decoding_work_mem 超过
  -> spill
  -> ReorderBufferWrite wait
  -> pg_stat_replication_slots.spill_bytes 增长
```

apply-side pressure：

```text
subscriber lock / index / trigger / FK / RLS / bloat
  -> local DML 慢或 ERROR
  -> origin 不推进
  -> feedback 不推进
  -> publisher confirmed_flush_lsn 不推进
  -> slot retention 增长
```

这个传播会让 publisher 看起来是 slot 问题，
但根因可能在 subscriber executor。

## 11. 观测入口

SQL 入口按 owner 读：

| owner | 入口 | 核心字段 |
| --- | --- | --- |
| upstream walsender | `pg_stat_replication` | `sent_lsn`、`write_lsn`、`flush_lsn`、`replay_lsn`、`backend_xmin`、`sync_state`。 |
| standby walreceiver | `pg_stat_wal_receiver` | `status`、`written_lsn`、`flushed_lsn`、`received_tli`、`latest_end_lsn`、message times。 |
| startup recovery | `pg_stat_recovery`、`pg_last_wal_receive_lsn()`、`pg_last_wal_replay_lsn()` | `pause_state`、replay LSN / TLI、replay timestamp。 |
| slot | `pg_replication_slots` | `restart_lsn`、`confirmed_flush_lsn`、`wal_status`、`safe_wal_size`、`catalog_xmin`、`invalidation_reason`。 |
| decoder stats | `pg_stat_replication_slots` | `spill_*`、`stream_*`、`mem_exceeded_count`、`total_*`。 |
| subscriber worker | `pg_stat_subscription` | `worker_type`、`pid`、`received_lsn`、`latest_end_lsn`、message times。 |
| apply errors | `pg_stat_subscription_stats` | `apply_error_count`、sync error counts、conflict counters。 |
| subscriber apply progress | `pg_replication_origin_status` | origin remote/local progress。 |
| current waits | `pg_stat_activity` | `backend_type`、`wait_event_type`、`wait_event`。 |

WAL / timeline 工具：

```text
pg_controldata "$PGDATA":
  control file、checkpoint TLI、min recovery point。

pg_waldump -t <TLI> -s <LSN> -e <LSN>:
  指定 timeline 验证 WAL record。

pg_waldump --stats:
  粗看 rmgr 分布，辅助判断 WAL 压力来源。
```

可见性边界：

```text
直接可见:
  LSN、slot 状态、worker pid、wait event、stats counters、logs。

只能推断:
  network buffer、具体 long transaction 贡献、业务冲突是否可跳过。

SQL 基本看不全:
  expectedTLEs 内存列表、plugin private state、reorder buffer 详细 change。
```

## 12. 常见误区

1. 把 `replay_lag` 当网络延迟。
   它可能包含 write、flush、redo、conflict、pause、feedback 周期。

2. 把 `pg_last_wal_receive_lsn()` 当 written LSN。
   当前源码通过 `GetWalRcvFlushRecPtr()` 读 flushed 边界。

3. 把 `restart_lsn` 当 consumer 确认进度。
   logical consumer 确认边界是 `confirmed_flush_lsn`。

4. 把 slot 膨胀都归因到 apply 慢。
   long transaction、snapbuild、reorder spill、plugin ERROR 都可能参与。

5. 把 WAL 文件存在当作 timeline 正确。
   recovery 必须确认 WAL 属于 expected history。

6. 把 output plugin 当 apply worker。
   plugin 在 publisher 编码和过滤，apply 在 subscriber 执行本地 DML。

7. 把 wait event 当完整根因。
   wait event 只是当前等待点。

## 13. 课堂实验

### 实验 1：physical 卡点图

```text
目标:
  用 LSN 差值和 wait event 判断卡在 send、write、flush 还是 replay。
步骤:
  搭建 primary + physical standby，持续写入；
  在 standby 制造长查询或 I/O 压力；
  采集 pg_stat_replication、pg_stat_wal_receiver、pg_stat_activity；
  画 current WAL -> sent -> written -> flushed -> replayed。
断点:
  XLogSendPhysical()
  ProcessStandbyReplyMessage()
  XLogWalRcvWrite()
  XLogWalRcvFlush()
  ApplyWalRecord()
```

### 实验 2：restart 与 confirmed

```text
目标:
  观察 logical slot 中消费确认和 WAL 保留下界不是同一状态。
步骤:
  创建 logical slot 或 subscription；
  开一个大事务并持续写入，不立刻 commit；
  观察 pg_replication_slots 和 pg_stat_replication_slots；
  commit 后让 consumer 消费并确认；
  比较 restart_lsn 和 confirmed_flush_lsn 推进顺序。
断点:
  LogicalConfirmReceivedLocation()
  LogicalIncreaseRestartDecodingForSlot()
  SnapBuildProcessRunningXacts()
  ReorderBufferCommit()
```

### 实验 3：apply ERROR 与 origin

```text
目标:
  验证 subscriber 本地冲突会让 worker 停在远端事务边界。
步骤:
  建立 publication / subscription；
  在 subscriber 造出 duplicate key；
  publisher 插入同 key 行；
  观察 pg_stat_subscription、pg_stat_subscription_stats、origin status；
  查日志中的 remote xid、finish LSN、relation。
断点:
  apply_handle_commit_internal()
  store_flush_position()
  start_apply()
  ReportApplyConflict()
  replorigin_session_get_progress()
  replorigin_xact_clear()
```

### 实验 4：timeline 不匹配

```text
目标:
  理解 WAL segment 存在不等于 recovery 可以接受。
步骤:
  promotion 产生新 timeline；
  保留旧 timeline WAL segment；
  配错 standby recovery target timeline；
  对比 logs、.history、pg_controldata；
  用 pg_waldump -t 验证不同 TLI。
断点:
  readTimeLineHistory()
  tliOfPointInHistory()
  ReadRecord()
  rescanLatestTimeLine()
```

## 14. 讨论题

1. `sent_lsn` 追上 primary current WAL 时，
   为什么 standby 查询仍可能看不到这些变更？

2. `flushed_lsn` 前进但 `pg_last_wal_replay_lsn()` 不动时，
   为什么下一步看 startup process？

3. `confirmed_flush_lsn` 前进但 `restart_lsn` 不前进时，
   更像 subscriber 慢还是 decoder protection？

4. 为什么 timeline 错误必须先于 lag 诊断？

5. pgoutput row filter 过滤掉变化时，
   subscriber 侧应该表现为 lag 还是没有消息？

6. apply duplicate key ERROR 后，
   为什么不能直接推进 origin 跳过失败行？

7. standby feedback bloat 和 logical `catalog_xmin` bloat
   如何从 owner 上区分？

8. 为什么 `pg_stat_subscription_stats` 不能替代日志中的 finish LSN？

## 15. 本节小结

本节边界图：

```text
physical:
  walsender sent
    -> walreceiver write / flush
    -> startup replay
    -> timeline history acceptance

logical:
  slot restart_lsn
    -> decoder / snapbuild / reorder buffer
    -> output plugin
    -> apply worker
    -> origin
    -> feedback confirmed_flush_lsn
```

ownership：

```text
walsender owns upstream send / feedback；
walreceiver owns standby receive / write / flush；
startup owns replay and timeline acceptance；
slot owns retention and xmin protection；
decoder owns WAL-to-change reconstruction；
plugin owns encoding and filtering；
apply worker owns subscriber local transaction and origin progress。
```

cleanup：

```text
decoder context 是 session-local；
persistent slot 是跨重启状态；
apply ERROR 清 origin xact state；
timeline / recovery 错误不会猜测跳过 WAL。
```

观测：

```text
SQL 视图给出分层 LSN、slot 状态、worker 状态和 counters；
wait event 给当前等待点；
logs 给 ERROR 和 timeline / conflict 细节；
pg_waldump 与 pg_controldata 给 WAL / timeline 文件事实。
```

可迁移规律：

```text
不要先解释 lag。
先定位 owner。
再判断这个 owner 的状态是否前进。
最后才把资源、错误和配置传播到相邻模块。
```
