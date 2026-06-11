# PostgreSQL 物理复制延迟诊断
## 课程定位
前置知识：已经理解 physical replication handshake、walsender 主循环、
walreceiver 接收写入、startup process recovery state machine、
Hot Standby conflict、recovery pause 和 synchronous replication 的基本语义。
本节唯一主问题：
```text
如何用 sent_lsn / write_lsn / flush_lsn / replay_lsn、
replay timestamp 和 wait events，
判断物理复制延迟卡在 primary send、network、standby write、standby flush、
REDO replay、Hot Standby 查询冲突，还是人为 pause / recovery target？
```
核心矛盾：复制延迟是一个跨进程、跨机器、跨存储层的现象。
诊断者希望看到一个“lag”数字后直接定位原因。
但 PostgreSQL 的实现故意不提供一个万能 lag。
它把同一段 WAL 的生命周期拆成多个状态：
```text
primary WAL flush
  -> walsender sent
  -> walreceiver received message
  -> walreceiver write
  -> walreceiver flush
  -> startup process replay
  -> standby query visibility / sync replication release
```
每个状态的 owner、锁、时间来源和观测入口都不同。
把它们合并会让诊断看起来简单，却会把 crash safety、
网络传输、standby I/O、REDO CPU、查询冲突和人为延迟混成一个现象。
学完后应能独立判断：
```text
sent_lsn 落后时，应该先看 primary 上 walsender 是否真的有可发送 WAL；
sent_lsn 已前进但 write_lsn 不前进时，如何区分 socket / network / receiver；
write_lsn 前进但 flush_lsn 不前进时，为什么问题在 standby WAL durable 边界；
flush_lsn 前进但 replay_lsn 不前进时，如何继续拆 REDO 慢、冲突、pause、apply delay；
write_lag / flush_lag / replay_lag 为什么不是“当前已经卡住多久”；
pg_last_wal_receive_lsn() 为什么对应 flush 而不是 write；
pg_last_xact_replay_timestamp() 为什么只能辅助估计 replay 新鲜度；
pg_stat_database_conflicts 为什么是累计结果，不是当前 blocker 列表；
cascading standby 上为什么必须分开看上游接收面和下游发送面。
```
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面的 replication 课程已经分别讲过：
```text
physical replication handshake:
  replication connection 如何进入 START_REPLICATION。
walsender main loop:
  primary 如何读取 WAL、封装 CopyData、推进 sentPtr、处理 standby reply。
walreceiver write status:
  standby 如何接收 WALData、写 pg_wal、fsync、上报 write / flush / apply。
startup process recovery state:
  standby 如何从 archive、pg_wal、stream 三类来源读取 WAL 并回放。
Hot Standby conflict:
  replay 为什么可能等待或取消 standby 查询。
```
本节不是再讲一遍这些机制。
本节把它们压缩成一个现场诊断问题：
```text
复制延迟变大时，当前卡点在哪里？
```
这个问题不能只在 primary 上回答。
`pg_stat_replication` 在 upstream walsender 进程里读 shared state。
它能告诉你 upstream 对某个 downstream 的已发送、已写、已 flush、已 replay 位置。
但它看不到 downstream 当前是否正在 `WALWrite`、`WALSync`、`RecoveryPause`，
也看不到 standby 查询为什么持有 snapshot 或 buffer pin。
这个问题也不能只在 standby 上回答。
`pg_stat_wal_receiver` 能告诉你 standby 从上游接收到哪里、写到哪里、flush 到哪里、
最近消息发送和接收时间是什么。
但它不知道 upstream 当前是否没有 WAL、socket send buffer 是否堆积、
同步复制是否正在等待这个 standby 的 apply。
所以本节的主线是：
```text
先把 LSN 分层建模；
再把每一层绑定到 owner 进程和源码字段；
然后用 wait event 与 timestamp 修正 LSN 推断；
最后用 conflict stats / recovery pause / cascading 拆掉常见假象。
```
一个好的诊断结论应该长这样：
```text
primary 的 sent_lsn 已经追到 pg_current_wal_flush_lsn()；
standby 的 latest_end_lsn 也在前进；
但 written_lsn 追不上 latest_end_lsn，walreceiver 当前 wait_event 是 WALWrite；
因此卡点在 standby WAL write I/O，而不是 primary send 或 REDO。
```
而不是这样：
```text
replay_lag 很大，所以网络慢。
```
`replay_lag` 只说明 standby apply 位置晚于某个 primary WAL flush 样本。
它本身不告诉你中间哪一段慢。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
primary walsender 把已 flush 的 WAL 放入 CopyData 并推进 sentPtr；
standby walreceiver 接收消息后先写本地 WAL segment，再 fsync 并发布 flushedUpto；
startup process 从已 flush 的 WAL 读取 record，redo 成功后推进 lastReplayedEndRecPtr；
standby 定期或在进度变化时把 write / flush / apply 回报给 primary；
primary 上的 pg_stat_replication 显示的是 walsender shared state 中这组回报。
```
这套模型有四个诊断不变量。
第一，LSN 是顺序边界，不是原因。
```text
sent_lsn >= write_lsn >= flush_lsn >= replay_lsn
```
在正常单 upstream / single standby 连接上，这个顺序通常成立。
但每个差值只告诉你“上游阶段已经超过下游阶段”。
它不能自动解释为什么。
原因要靠 owner 进程的 wait event、日志、I/O、CPU 和 conflict state 补齐。
第二，`sent_lsn` 的语义比名字弱。
`walsender.c` 中 `sentPtr` 的注释说它实际是“next WAL location to send”。
`XLogSendPhysical()` 在 `pq_putmessage_noblock()` 后推进 `sentPtr`，
然后复制到 `MyWalSnd->sentPtr`。
这表示 WAL 已经进入 walsender 的输出路径。
它不保证 standby 已经收到，也不保证内核 socket buffer 已经全部排空。
第三，standby 的 receive 也分 write 和 flush。
`walreceiver.c` 中 `XLogWalRcvWrite()` 成功 `pg_pwrite()` 后推进
`LogstreamResult.Write` 和 shared atomic `WalRcv->writtenUpto`。
`XLogWalRcvFlush()` 调用 `issue_xlog_fsync()` 后才推进
`LogstreamResult.Flush` 和 `WalRcv->flushedUpto`。
`pg_last_wal_receive_lsn()` 调的正是 `GetWalRcvFlushRecPtr()`。
因此它表示“已接收并同步到磁盘”的边界，而不是“刚写过”的边界。
第四，replay 是另一个进程的完成状态。
`GetXLogReplayRecPtr()` 从 `XLogRecoveryCtl->lastReplayedEndRecPtr` 读值。
`ApplyWalRecord()` 先把 `replayEndRecPtr` 设成正在 apply 的 record end，
真正 redo 成功后才推进 `lastReplayedEndRecPtr`。
所以 `pg_last_wal_replay_lsn()` 和 primary 上的 `replay_lsn`
代表已完成回放，不代表正在回放的 record 已经完成。
本节 tension 可以压缩为：
```text
诊断希望用一个 lag 数字给出原因
  vs
正确性要求把“发送、写入、持久化、回放、可见、反馈”拆成多个独立状态
```
PostgreSQL 选择了后者。
源码和视图都围绕这个拆分组织。
## 3. 核心文件分工与阅读顺序
本节实际核对的核心文件如下。
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/catalog/system_views.sql` | `pg_stat_replication`、`pg_stat_wal_receiver`、`pg_stat_recovery`、`pg_stat_database_conflicts` 的 SQL 映射。 |
| 2 | `src/include/replication/walsender_private.h` | `WalSnd` shared state: `sentPtr`、`write`、`flush`、`apply`、lag 和 `replyTime`。 |
| 3 | `src/backend/replication/walsender.c` | `XLogSendPhysical()` 推进 `sentPtr`，`ProcessStandbyReplyMessage()` 写入 write/flush/apply 和 lag，`pg_stat_get_wal_senders()` 暴露视图。 |
| 4 | `src/include/replication/walreceiver.h` | `WalRcvData` shared state: `writtenUpto`、`flushedUpto`、message timestamps、latest WAL end。 |
| 5 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()`、`XLogWalRcvProcessMsg()`、`XLogWalRcvWrite()`、`XLogWalRcvFlush()`、`XLogWalRcvSendReply()`、`pg_stat_get_wal_receiver()`。 |
| 6 | `src/backend/replication/walreceiverfuncs.c` | startup process 如何 `RequestXLogStreaming()`，如何读 `GetWalRcvFlushRecPtr()` / `GetWalRcvWriteRecPtr()`。 |
| 7 | `src/backend/access/transam/xlogrecovery.c` | `WaitForWALToBecomeAvailable()`、`PerformWalRecovery()`、`ApplyWalRecord()`、pause、apply delay、replay LSN 和 replay timestamp。 |
| 8 | `src/backend/access/transam/xlogfuncs.c` | `pg_last_wal_receive_lsn()`、`pg_last_wal_replay_lsn()`、pause 函数、`pg_last_xact_replay_timestamp()`、`pg_stat_get_recovery()`。 |
| 9 | `src/backend/replication/syncrep.c` | `SyncRepReleaseWaiters()` 如何用 write/flush/apply 释放 commit 等待者。 |
| 10 | `src/backend/utils/activity/wait_event_names.txt` | wait event 名称和用户可见描述。 |
| 11 | `src/backend/storage/ipc/standby.c` | Hot Standby conflict 的等待、取消和日志。 |
| 12 | `src/backend/utils/activity/pgstat_database.c` | recovery conflict counters 如何累计到 database 统计。 |
推荐阅读顺序不是从视图开始横向背列名。
更好的顺序是沿一段 WAL 走：
```text
primary already flushed WAL
  -> XLogSendPhysical()
  -> WalSnd.sentPtr
  -> walreceiver receives WALData
  -> XLogWalRcvWrite()
  -> WalRcv.writtenUpto
  -> XLogWalRcvFlush()
  -> WalRcv.flushedUpto
  -> XLogWalRcvSendReply()
  -> ProcessStandbyReplyMessage()
  -> WalSnd.write / flush / apply
  -> pg_stat_replication
  -> startup ApplyWalRecord()
  -> GetXLogReplayRecPtr()
```
然后再读异常支路：
```text
no WAL:
  WaitForWALToBecomeAvailable() / RecoveryWalStream / RecoveryRetrieveRetryInterval
socket / receiver idle:
  WalSndLoop() / WalReceiverMain() / keepalive / timeout
standby write or sync:
  WALWrite / WALSync wait events
replay blocked:
  recoveryPausesHere()
  recoveryApplyDelay()
  ResolveRecoveryConflictWithVirtualXIDs()
  ResolveRecoveryConflictWithBufferPin()
```
## 4. 关键状态与语义边界
### 4.1 `pg_stat_replication`: upstream 对 downstream 的视角
`system_views.sql` 中 `pg_stat_replication` 不是直接读表。
它把 `pg_stat_get_activity(NULL)` 与 `pg_stat_get_wal_senders()` 按 pid join。
核心列来自 `pg_stat_get_wal_senders()`：
| 列 | 源码字段 | 更新者 | 诊断语义 |
| --- | --- | --- | --- |
| `sent_lsn` | `WalSnd.sentPtr` | upstream walsender | walsender 已把 WAL 推进到的发送边界。 |
| `write_lsn` | `WalSnd.write` | upstream walsender 处理 standby reply | standby 回报已写到本地 WAL 文件的位置。 |
| `flush_lsn` | `WalSnd.flush` | upstream walsender 处理 standby reply | standby 回报已 fsync 的位置。 |
| `replay_lsn` | `WalSnd.apply` | upstream walsender 处理 standby reply | standby 回报 startup process 已 replay 的位置。 |
| `write_lag` | `WalSnd.writeLag` | upstream walsender lag tracker | WAL flush 样本到 standby write 回报的 elapsed time。 |
| `flush_lag` | `WalSnd.flushLag` | upstream walsender lag tracker | WAL flush 样本到 standby flush 回报的 elapsed time。 |
| `replay_lag` | `WalSnd.applyLag` | upstream walsender lag tracker | WAL flush 样本到 standby apply 回报的 elapsed time。 |
| `reply_time` | `WalSnd.replyTime` | upstream walsender 处理 standby reply | standby 在 status update 中带回的发送时间。 |
这个视图是 upstream-local shared memory 视角。
在 primary 上，它描述 primary 到 standby。
在 cascading standby 上，它描述这个中间节点到下游 standby。
`pg_stat_get_wal_senders()` 在 `WalSnd->mutex` 下复制字段。
如果调用者没有 `pg_read_all_stats` 权限，当前源码只保留 pid，
其它细节列置 NULL。
所以诊断脚本要先确认权限。
`sync_state` 也是信息列。
当前源码里：
```text
priority == 0:
  async
priority standby and considered sync:
  priority mode -> sync
  quorum mode -> quorum
priority standby but not currently selected:
  potential
```
不要把 `sync_state='sync'` 直接解释成“没有延迟”。
它只描述同步复制候选和选择状态。
延迟阶段仍然要看 write/flush/replay LSN。
### 4.2 `pg_stat_wal_receiver`: standby 对 upstream 的视角
`system_views.sql` 中 `pg_stat_wal_receiver` 来自
`pg_stat_get_wal_receiver()`，并过滤 `pid IS NOT NULL`。
核心列来自 `WalRcvData`：
| 列 | 源码字段 | 更新者 | 诊断语义 |
| --- | --- | --- | --- |
| `status` | `walRcvState` | startup process / walreceiver | connecting、streaming、waiting、restarting 等状态。 |
| `receive_start_lsn` | `receiveStart` | startup process | 本轮 streaming 请求从哪里开始。 |
| `written_lsn` | `writtenUpto` | walreceiver | `pg_pwrite()` 成功后推进，atomic 读。 |
| `flushed_lsn` | `flushedUpto` | walreceiver | `issue_xlog_fsync()` 成功后推进，spinlock 保护。 |
| `last_msg_send_time` | `lastMsgSendTime` | walreceiver | 最近收到的上游消息携带的发送时间。 |
| `last_msg_receipt_time` | `lastMsgReceiptTime` | walreceiver | standby 本地收到消息的时间。 |
| `latest_end_lsn` | `latestWalEnd` | walreceiver | 上游消息报告的 end-of-WAL。 |
| `latest_end_time` | `latestWalEndTime` | walreceiver | `latest_end_lsn` 对应的上游发送时间。 |
当前源码特意提醒：
```text
writtenUpto 可无 spinlock 读取，
但可能和其它 spinlock-protected 字段不一致，
不应用于 data integrity checks。
```
因此 `written_lsn` 是诊断线索。
`flushed_lsn` 才是 startup process 可稳定读取 streaming WAL 的边界。
`status='waiting'` 也不等于异常。
`WalRcvWaitForStartPosition()` 会把 walreceiver 置为 waiting，
等待 startup process 给新的 timeline / startpoint。
timeline 切换、旧 timeline 结束、archive 与 stream 重新选择时都可能出现。
### 4.3 standby SQL 函数
`xlogfuncs.c` 中三个函数常被混用。
| 函数 | 源码调用 | 语义 |
| --- | --- | --- |
| `pg_last_wal_receive_lsn()` | `GetWalRcvFlushRecPtr(NULL, NULL)` | walreceiver 已接收并 sync 到磁盘的 LSN。 |
| `pg_last_wal_replay_lsn()` | `GetXLogReplayRecPtr(NULL)` | startup process 已完成 replay 的 LSN。 |
| `pg_last_xact_replay_timestamp()` | `GetLatestXTime()` | 最近处理的 commit / abort WAL record 的事务时间。 |
`pg_last_wal_receive_lsn()` 名字里有 receive，
但源码注释说它用于判断 WAL “guaranteed to be received and synced to disk”。
所以它和 `pg_stat_wal_receiver.flushed_lsn` 对齐，
不是和 `written_lsn` 对齐。
`pg_last_wal_replay_lsn()` 用来判断只读连接在 recovery 中能看到多少 WAL 的效果。
它不包含当前 redo 函数正在处理、但尚未成功完成的 record。
`pg_last_xact_replay_timestamp()` 不是当前时间。
它是 WAL 里事务 commit / abort record 的时间。
如果 standby 没有 replay 新事务，它会保持旧值。
如果 workload 主要是非事务完成类 WAL，LSN 可以前进而这个 timestamp 不更新。
如果 primary / standby 时钟不同，用 `now() - pg_last_xact_replay_timestamp()`
估算 replay delay 会带入时钟误差。
当前源码还提供 `pg_stat_recovery`。
它暴露：
```text
last_replayed_read_lsn
last_replayed_end_lsn
replay_end_lsn
recovery_last_xact_time
current_chunk_start_time
pause_state
```
这对区分“已完成 replay”与“正在 replay 某条 record”很有用。
`replay_end_lsn` 来自 `XLogRecoveryCtl->replayEndRecPtr`。
`last_replayed_end_lsn` 来自 `lastReplayedEndRecPtr`。
### 4.4 write / flush / replay lag 的真实含义
`walsender.c` 中有 `LagTracker`。
它维护一个 8192 entries 的环形样本：
```text
lsn:
  primary 上某个 WAL 位置。
time:
  这个 WAL 位置在 upstream 本地 flush 的时间。
```
`LagTrackerWrite()` 只在 LSN 前进时记录新样本。
`ProcessStandbyReplyMessage()` 收到 standby status update 后，
用 standby 回报的 write / flush / apply LSN 去 `LagTrackerRead()` 中匹配样本，
计算当前时间与样本时间的差。
所以：
```text
write_lag:
  upstream WAL flush sample -> standby 回报 write_lsn 穿过该 sample 的时间。
flush_lag:
  upstream WAL flush sample -> standby 回报 flush_lsn 穿过该 sample 的时间。
replay_lag:
  upstream WAL flush sample -> standby 回报 apply_lsn 穿过该 sample 的时间。
```
限制也来自源码。
第一，没有新样本时可能返回 -1，
视图显示 NULL。
第二，standby 完全追上且连续两次 reply 位置不变时，
`ProcessStandbyReplyMessage()` 会清空 lag，
避免在空闲系统里展示陈旧 lag。
通常会在 `wal_receiver_status_interval` 后被清掉。
第三，环形缓冲会因为慢 reader 溢出而使用 overflow sample。
这仍是近似。
第四，如果本地时钟倒退，`LagTrackerRead()` 把结果当作无效。
第五，`replay_lag` 可以在 apply 完全卡住时通过插值显示增长。
这有助于看到 stuck apply，
但仍然不是 causality。
因此课程里的诊断规则是：
```text
LSN delta 用于定位阶段；
lag interval 用于估计阶段耗时；
wait event / 日志 / counters 用于确认原因。
```
## 5. 主流程源码 walkthrough
### 5.1 primary 发送阶段
物理复制开始后，`StartReplication()` 设置：
```text
sentPtr = cmd->startpoint
MyWalSnd->sentPtr = sentPtr
WalSndSetState(WALSNDSTATE_CATCHUP)
WalSndLoop(XLogSendPhysical)
```
`WalSndLoop()` 每轮先做这些事：
```text
ResetLatch(MyLatch)
CHECK_FOR_INTERRUPTS()
WalSndHandleConfigReload()
ProcessRepliesIfAny()
如果没有 pending output:
  XLogSendPhysical()
否则:
  等 socket writable，先把已有输出 flush 出去
WalSndCheckTimeOut()
WalSndKeepaliveIfNecessary()
WalSndWait(... WAIT_EVENT_WAL_SENDER_MAIN)
```
`XLogSendPhysical()` 的关键不是“读多少 WAL”。
关键是发送前先确认这个 upstream 已经有安全 WAL 可读：
```text
WalSndWaitForWal(targetPagePtr + reqLen)
  -> primary 上用 GetFlushRecPtr(NULL)
  -> recovery 中可能用 replay/standby 相关边界
  -> 不够则 WAIT_EVENT_WAL_SENDER_WAIT_FOR_WAL
```
然后它填充 WALData 消息：
```text
dataStart
walEnd
sendTime
WAL payload
```
最后：
```text
pq_putmessage_noblock(PqMsg_CopyData, ...)
sentPtr = endptr
MyWalSnd->sentPtr = sentPtr
```
诊断含义：
```text
sent_lsn 落后于 upstream 当前 flush LSN:
  walsender 没把可发送 WAL 排出去。
sent_lsn 已经追上 upstream 当前 flush LSN:
  upstream 没有更多可安全发送的 WAL，
  此时不应把 downstream replay_lag 归因于 primary send。
```
但还要注意：
```text
sent_lsn 前进:
  只说明 walsender 输出路径已推进。
standby 收到:
  要看 standby pg_stat_wal_receiver.last_msg_receipt_time / latest_end_lsn，
  或 primary 后续收到的 write_lsn。
```
### 5.2 standby 接收、写入和 flush 阶段
`WalReceiverMain()` 建立连接后进入 streaming loop。
每轮核心步骤是：
```text
walrcv_receive(wrconn, &buf, &wait_fd)
  -> 有消息:
       XLogWalRcvProcessMsg()
       XLogWalRcvSendReply(false, false, false)
       XLogWalRcvFlush(false, startpointTLI)
  -> 没消息:
       WaitLatchOrSocket(... WAIT_EVENT_WAL_RECEIVER_MAIN)
       timeout 时发送 reply / hot standby feedback / ping
```
`XLogWalRcvProcessMsg()` 处理两类复制消息。
`PqReplMsg_WALData`：
```text
读取 dataStart / walEnd / sendTime
ProcessWalSndrMessage(walEnd, sendTime)
XLogWalRcvWrite(payload, dataStart, tli)
```
`PqReplMsg_Keepalive`：
```text
读取 walEnd / sendTime / replyRequested
ProcessWalSndrMessage(walEnd, sendTime)
需要时 XLogWalRcvSendReply(true, false, false)
```
`ProcessWalSndrMessage()` 更新：
```text
latestWalEnd
latestWalEndTime
lastMsgSendTime
lastMsgReceiptTime
```
因此 `latest_end_lsn` 前进但 `written_lsn` 不前进，
说明消息层已经到达，
问题更靠近 standby 写 WAL 或进程调度。
`XLogWalRcvWrite()` 的关键路径：
```text
需要时 XLogFileInit()
pgstat_report_wait_start(WAIT_EVENT_WAL_WRITE)
pg_pwrite(recvFile, buf, segbytes, startoff)
pgstat_report_wait_end()
LogstreamResult.Write = recptr
pg_atomic_write_membarrier_u64(&WalRcv->writtenUpto, LogstreamResult.Write)
WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_WRITE, LogstreamResult.Write)
```
`XLogWalRcvFlush()` 的关键路径：
```text
if LogstreamResult.Flush < LogstreamResult.Write:
  issue_xlog_fsync(recvFile, recvSegNo, tli)
  LogstreamResult.Flush = LogstreamResult.Write
  WalRcv->flushedUpto = LogstreamResult.Flush
  WalRcv->receivedTLI = tli
  WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_FLUSH, LogstreamResult.Flush)
  WakeupRecovery()
  如果允许 cascading:
    WalSndWakeup(true, false)
  XLogWalRcvSendReply(false, false, false)
```
诊断含义：
```text
latest_end_lsn - written_lsn 大:
  standby 收到或至少看到上游 end-of-WAL 信息，
  但 write 没追上。
written_lsn - flushed_lsn 大:
  standby write 成功多于 fsync 成功，
  卡点在 durable WAL 边界。
pg_last_wal_receive_lsn():
  应与 flushed_lsn 对齐，而不是 written_lsn。
```
### 5.3 standby status update 反馈阶段
`XLogWalRcvSendReply()` 构造 `StandbyStatusUpdate`：
```text
writePtr = LogstreamResult.Write
flushPtr = LogstreamResult.Flush
applyPtr = GetXLogReplayRecPtr(NULL)
reply timestamp = GetCurrentTimestamp()
requestReply flag
```
它的发送时机包括：
```text
force=true:
  初始 reply、primary request reply 等。
位置前进:
  write / flush 前进时。
checkApply=true:
  startup process 要求 apply feedback 时。
status interval 到期:
  wal_receiver_status_interval。
```
startup process 在 `WaitForWALToBecomeAvailable()` 中准备等待新 WAL 前，
如果已经 replay 了当前已收到的所有 WAL，
会调用 `WalRcvRequestApplyReply()`。
源码注释说明这是为了避免 `pg_stat_replication` 展示陈旧 replay 信息。
primary 上的 `ProcessStandbyReplyMessage()` 读取：
```text
writePtr
flushPtr
applyPtr
replyTime
replyRequested
```
然后：
```text
writeLag = LagTrackerRead(SYNC_REP_WAIT_WRITE, writePtr, now)
flushLag = LagTrackerRead(SYNC_REP_WAIT_FLUSH, flushPtr, now)
applyLag = LagTrackerRead(SYNC_REP_WAIT_APPLY, applyPtr, now)
MyWalSnd->write = writePtr
MyWalSnd->flush = flushPtr
MyWalSnd->apply = applyPtr
MyWalSnd->writeLag / flushLag / applyLag = ...
MyWalSnd->replyTime = replyTime
```
如果不是 cascading walsender，还会：
```text
SyncRepReleaseWaiters()
```
如果 walsender 持有 replication slot，
还会用 flushPtr 推进 slot 确认位置。
诊断含义：
```text
primary 上 write_lsn / flush_lsn / replay_lsn 的新鲜度
依赖 standby 发送 reply，
也依赖 primary walsender 处理 reply。
standby 已 replay 但 primary replay_lsn 还旧:
  可能只是 apply feedback 尚未发出或尚未被处理。
```
### 5.4 startup process replay 阶段
`PerformWalRecovery()` 初始化 replay shared state：
```text
lastReplayedReadRecPtr
lastReplayedEndRecPtr
lastReplayedTLI
replayEndRecPtr
recoveryLastXTime
currentChunkStartTime
recoveryPauseState
```
主 redo loop 中每条 record：
```text
ProcessStartupProcInterrupts()
如果 pause requested:
  recoveryPausesHere(false)
如果达到 recovery target:
  break 或 pause/promote/shutdown
如果 recovery_min_apply_delay 命中:
  recoveryApplyDelay()
ApplyWalRecord()
WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_REPLAY, lastReplayedEndRecPtr)
ReadRecord()
```
`ApplyWalRecord()` 内部：
```text
设置 replayEndRecPtr = xlogreader->EndRecPtr
记录 KnownAssigned XID
调用 rmgr redo
redo 成功后:
  lastReplayedReadRecPtr = ReadRecPtr
  lastReplayedEndRecPtr = EndRecPtr
  lastReplayedTLI = replayTLI
必要时唤醒 cascading walsender / walreceiver apply reply
```
诊断含义：
```text
replay_end_lsn > last_replayed_end_lsn:
  当前可能正在 apply 某条 record。
flush_lsn 大幅领先 last_replayed_end_lsn:
  WAL 已经在本地 durable，
  卡点不在网络或 walreceiver flush，
  应继续看 replay CPU/I/O、conflict、pause、apply delay。
```
## 6. 生命周期 / ownership / cleanup
### 6.1 walsender shared slot
`WalSndCtlData` 是 shared memory 中的全局结构。
每个 walsender 在 `InitWalSenderSlot()` 中找一个空闲 `WalSnd`：
```text
pid = MyProcPid
state = WALSNDSTATE_STARTUP
sentPtr = InvalidXLogRecPtr
write / flush / apply = InvalidXLogRecPtr
writeLag / flushLag / applyLag = -1
replyTime = 0
kind = physical 或 logical
```
退出时 `WalSndKill()` 把 `pid` 置 0。
视图扫描 `max_wal_senders` 个 slot，
只返回 `pid != 0` 的条目。
ERROR cleanup 不是事务 abort。
`WalSndErrorCleanup()` 做 walsender 专属收尾：
```text
LWLockReleaseAll()
ConditionVariableCancelSleep()
pgstat_report_wait_end()
关闭 xlogreader segment
释放 replication slot
ReplicationSlotCleanup(false)
replication_active = false
必要时 ReleaseAuxProcessResources(false)
WalSndSetState(WALSNDSTATE_STARTUP)
```
所以诊断时看到 walsender pid 消失，
不代表 shared memory 丢了。
它可能是连接结束、错误、timeout、shutdown 或 standby 断开。
### 6.2 walreceiver shared state
`WalRcvData` 由 `WalRcvShmemInit()` 初始化：
```text
walRcvState = WALRCV_STOPPED
ConditionVariableInit(walRcvStoppedCV)
SpinLockInit(mutex)
writtenUpto = 0
procno = INVALID_PROC_NUMBER
```
startup process 通过 `RequestXLogStreaming()`：
```text
必要时把 recptr 调整到 segment 起点
设置 slotname / temp slot
STOPPED -> STARTING 或 WAITING -> RESTARTING
初始化 flushedUpto / receivedTLI / latestChunkStart
写 writtenUpto
设置 receiveStart / receiveStartTLI
启动 walreceiver 或 SetLatch 复用已有进程
```
walreceiver 启动后在 `WalReceiverMain()`：
```text
pid = MyProcPid
walRcvState = CONNECTING
procno = MyProcNumber
ready_to_display = true
连接成功后填 sender_host / sender_port / conninfo
START_REPLICATION 成功后 CONNECTING -> STREAMING
```
退出时 `WalRcvDie()`：
```text
XLogWalRcvFlush(true, startpointTLI)
walRcvState = STOPPED
pid = 0
procno = INVALID_PROC_NUMBER
ready_to_display = false
ConditionVariableBroadcast(walRcvStoppedCV)
断开连接
WakeupRecovery()
```
这里的 cleanup 解释了一个诊断现象：
```text
walreceiver 退出前会尽量 flush 已收到 WAL；
因此连接断开不等于已收到 WAL 丢失；
但后续 replay 能不能继续，取决于 startup process 能否从 pg_wal / archive / stream 找到目标 WAL。
```
### 6.3 startup process recovery state
`XLogRecoveryCtlData` 也是 shared memory。
它持有：
```text
recoveryWakeupLatch
lastReplayedReadRecPtr
lastReplayedEndRecPtr
lastReplayedTLI
replayEndRecPtr
recoveryLastXTime
currentChunkStartTime
recoveryPauseState
recoveryNotPausedCV
info_lck
```
`recoveryWakeupLatch` 专门给 WAL arrival / promotion 这类 recovery 推进使用。
源码注释强调 startup process 还有 `procLatch` 用于 recovery conflict。
两个 latch 分开，是为了避免 walreceiver wakeup 和 conflict wakeup 互相污染语义。
这对诊断很关键：
```text
RecoveryWalStream:
  startup process 等 WAL 到达。
RecoveryPause:
  startup process 等用户恢复 replay。
BufferCleanup / lock wait / RecoveryConflictSnapshot:
  startup process 等冲突消失或准备取消查询。
```
这些 wait event 都可能表现为 `replay_lsn` 不动，
但原因完全不同。
## 7. 正确性机制层次
本节依赖四层正确性边界。
WAL safety：primary walsender 不发送本 server crash 后可能消失的 WAL；standby 只有在 `XLogWalRcvFlush()` fsync 后才发布 `flushedUpto` 并唤醒 startup process。
Shared state：`WalSnd`、`WalRcvData`、`XLogRecoveryCtlData` 分别用 spinlock 或 atomic 发布状态；`writtenUpto` 是诊断进度，不是 data integrity 边界。
Wakeup：walreceiver flush 后 `WakeupRecovery()`，cascading 场景还会 `WalSndWakeup(true, false)`；wait event 只说明当前睡眠点，不说明完整因果。
Sync replication：`syncrep.h` 的 wait mode 是 write / flush / apply，`SyncRepReleaseWaiters()` 用 walsender reply 状态释放 commit waiters；`sync_state` 是同步选择状态，不是 lag root cause。
诊断时要记住：
```text
raw field + owner process + lock/atomic boundary + wait point
才构成语义。
```
## 8. 观测入口与诊断分层
### 8.1 最小观测集合
primary 或 upstream 上先看：
```sql
SELECT application_name, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag, sync_state, reply_time
FROM pg_stat_replication;
```
standby 上看 receiver 和 replay：
```sql
SELECT status, written_lsn, flushed_lsn, last_msg_send_time,
       last_msg_receipt_time, latest_end_lsn, latest_end_time
FROM pg_stat_wal_receiver;
SELECT pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp(), pg_get_wal_replay_pause_state();
```
当前源码还有 `pg_stat_recovery`，可用来区分 `last_replayed_end_lsn` 与 `replay_end_lsn`，并直接看 `pause_state`。
当前等待点和查询侧线索来自：
```sql
SELECT pid, backend_type, state, wait_event_type, wait_event,
       backend_xmin, xact_start, query_start, query
FROM pg_stat_activity
WHERE backend_type IN ('walsender', 'walreceiver', 'startup')
   OR wait_event IS NOT NULL;
```
Hot Standby conflict 的累计结果来自 `pg_stat_database_conflicts`：
`confl_tablespace`、`confl_lock`、`confl_snapshot`、`confl_bufferpin`、
`confl_deadlock`、`confl_active_logicalslot`。
### 8.2 LSN delta 的基本读法
先把 upstream 和 standby 的 LSN 差值转成 bytes。
```sql
SELECT
  application_name,
  pg_wal_lsn_diff(sent_lsn, write_lsn) AS send_to_write_bytes,
  pg_wal_lsn_diff(write_lsn, flush_lsn) AS write_to_flush_bytes,
  pg_wal_lsn_diff(flush_lsn, replay_lsn) AS flush_to_replay_bytes
FROM pg_stat_replication;
```
解释：
```text
send_to_write_bytes 大:
  sent 已经领先 standby write。
  可能是 socket/network，也可能是 walreceiver 收到但写慢。
write_to_flush_bytes 大:
  standby 已写但未 fsync。
  首先看 standby WALSync / storage。
flush_to_replay_bytes 大:
  standby 已 durable 接收但未 replay。
  继续看 startup process。
```
如果有权限和连接条件，
把 standby 侧 `pg_stat_wal_receiver` 加进来。
```text
latest_end_lsn - written_lsn 大:
  receiver 已看到上游 end-of-WAL 信息但 write 没追上。
written_lsn - flushed_lsn 大:
  write 已完成多于 flush。
flushed_lsn - pg_last_wal_replay_lsn() 大:
  WAL 已 durable，replay 追不上。
```
注意：
`latest_end_lsn` 来自上游消息报告的 WAL end。
它不是一个“已接收 payload 到哪里”的精确字段。
但它和 `last_msg_receipt_time` 结合，
足以判断 standby 是否仍在收到上游消息。
### 8.3 wait event 对照表
当前源码中的相关 wait events：
| wait event | 典型 owner | 诊断含义 |
| --- | --- | --- |
| `WalSenderMain` | walsender | walsender 主循环等待 socket / latch / timeout；需要结合 LSN 判断是否空闲或 socket blocked。 |
| `WalSenderWaitForWal` | walsender | 等 upstream 有足够已 flush WAL 可发送；如果 sent 已追上 upstream flush，通常不是复制瓶颈。 |
| `WalSenderWriteData` | walsender | 处理 pending output 时等 socket 活动，常见于 output buffer 还没排空。 |
| `WalReceiverMain` | walreceiver | walreceiver 主循环等上游 socket / latch / timeout。 |
| `LibpqWalReceiverReceive` | walreceiver | libpq walreceiver 等远端数据。 |
| `WalWrite` | walreceiver 或 WAL writer | WAL 文件写 I/O。 |
| `WalSync` | walreceiver 或 WAL writer | WAL fsync / durable storage。 |
| `RecoveryWalStream` | startup | 等 walreceiver 带来更多 streaming WAL。 |
| `RecoveryRetrieveRetryInterval` | startup | archive / pg_wal / stream 都没有目标 WAL，按 retry interval 睡眠。 |
| `RecoveryApplyDelay` | startup | `recovery_min_apply_delay` 正在人为延迟 apply。 |
| `RecoveryPause` | startup | replay pause 或 target action pause。 |
| `RecoveryConflictSnapshot` | startup | 等 snapshot conflict 解决，通常由 vacuum cleanup 类 WAL 触发。 |
| `RecoveryConflictTablespace` | startup | 等 tablespace drop conflict。 |
| `BufferCleanup` | startup | 等 exclusive buffer pin，可能是 buffer pin conflict。 |
| lock wait event | startup | replay 需要 AccessExclusiveLock，被 standby query lock 阻塞。 |
| `SyncRep` | user backend on primary | commit 等 synchronous replication 确认。 |
| `WaitForWalWrite` | SQL/backend | 等 standby write LSN 达到目标。 |
| `WaitForWalFlush` | SQL/backend | 等 WAL flush LSN 达到目标。 |
| `WaitForWalReplay` | SQL/backend | 等 standby replay LSN 达到目标。 |
wait event 的边界：
```text
有 wait event:
  说明进程正在一个可标注等待点。
没有 wait event:
  可能正在 CPU redo、执行 rmgr redo、构造消息、处理 reply、持锁短临界区。
wait event 名称:
  不能告诉你谁造成等待，也不一定告诉你等待多久。
```
## 9. 按卡点诊断
### 9.1 primary 没有把 WAL 送出去
先在 upstream 看：
```text
pg_current_wal_flush_lsn() - sent_lsn 是否持续增长？
walsender wait_event 是什么？
state 是 catchup 还是 streaming？
```
如果 `sent_lsn` 落后 upstream 当前 flush LSN，
说明 walsender 没有把已有安全 WAL 推进到发送边界。
常见线索：
```text
walsender wait_event = WalSenderWaitForWal:
  如果 upstream 当前 flush LSN 也没有超过 sent_lsn，
  这通常只是没有更多安全 WAL。
walsender wait_event = WalSenderMain 且 sent_lsn 不动:
  可能在等 socket / reply / timeout，
  需要看是否有 pending output 的间接现象：
  sent 已领先 write、standby last_msg_receipt_time 不更新。
primary WAL flush 本身落后 insert:
  这是 upstream WAL flush / storage / commit path 问题，
  不应归因于 replication transport。
```
源码回扣：
```text
WalSndLoop()
  -> 没有 pending output 才调用 XLogSendPhysical()
  -> 有 pending output 时先 pq_flush_if_writable()
  -> 必要时 WalSndWait(... WalSenderMain)
```
诊断结论应写成：
```text
upstream 可发送 WAL 是否存在？
walsender 是否把它排入 output？
output 是否排不出 socket？
```
不要只说 “sender 慢”。
### 9.2 network / socket / receiver 没收到
典型现象：
```text
primary sent_lsn 已前进；
primary write_lsn 长时间不前进；
standby pg_stat_wal_receiver.last_msg_receipt_time 不更新；
standby latest_end_lsn 不更新；
walsender 可能在 WalSenderMain 或 WalSenderWriteData。
```
这更像：
```text
primary 输出还没到 standby；
网络断续；
receiver 连接卡住；
上游和下游之间的 socket backpressure。
```
`write_lag` 大不能单独证明网络慢。
如果 standby 已经收到但写盘慢，primary 也会看到 write_lsn 落后。
因此要加 standby 侧判断：
```text
last_msg_receipt_time 最近有更新:
  网络仍在送消息。
latest_end_lsn 前进:
  上游消息到达。
written_lsn 不前进:
  更像 standby write。
```
`last_msg_send_time` 与 `last_msg_receipt_time` 的差也不能当纯网络延迟。
`GetReplicationTransferLatency()` 的源码注释明确说：
它包含两台服务器时钟设置差异以及 timezone。
这个差值适合作为“最近消息是否在流动”的线索，
不适合作为精确 RTT。
### 9.3 standby write 卡住
典型现象：
```text
standby last_msg_receipt_time 继续更新；
standby latest_end_lsn 前进；
standby written_lsn 不追 latest_end_lsn；
walreceiver wait_event 可能是 WalWrite；
primary 上 sent_lsn - write_lsn 变大。
```
源码回扣：
```text
XLogWalRcvProcessMsg()
  -> ProcessWalSndrMessage() 先更新 latestWalEnd / msg time
  -> XLogWalRcvWrite()
     -> pg_pwrite()
     -> LogstreamResult.Write
     -> WalRcv->writtenUpto
```
如果 `WALWrite` 等待明显，
优先看 standby WAL 目录所在存储、I/O 饱和、文件系统、WAL segment 初始化。
如果 walreceiver 没有 wait event，
但 CPU 很高或频繁上下文切换，
也可能是进程调度或消息处理 overhead。
这时 `pg_stat_*` 不能证明原因，
需要操作系统 I/O、perf 或日志。
### 9.4 standby flush 卡住
典型现象：
```text
standby written_lsn 明显领先 flushed_lsn；
pg_last_wal_receive_lsn() 接近 flushed_lsn；
walreceiver wait_event 可能是 WalSync；
primary write_lsn 前进但 flush_lsn 不前进。
```
源码回扣：
```text
XLogWalRcvFlush()
  -> issue_xlog_fsync()
  -> LogstreamResult.Flush = LogstreamResult.Write
  -> WalRcv->flushedUpto = LogstreamResult.Flush
  -> WakeupRecovery()
  -> XLogWalRcvSendReply()
```
这里的关键是：
```text
startup process 等的是 flushedUpto；
primary remote_flush 等的是 standby reply 的 flushPtr；
pg_last_wal_receive_lsn() 也是 flush 边界。
```
所以 `written_lsn` 追上不代表 standby durable 接收完成。
如果同步复制配置要求 remote_flush 或 `synchronous_commit=on`，
primary commit 也会受这个阶段影响。
### 9.5 REDO replay 慢
典型现象：
```text
standby flushed_lsn 或 pg_last_wal_receive_lsn() 前进；
pg_last_wal_replay_lsn() 不前进或显著落后；
primary flush_lsn 前进但 replay_lsn 落后；
没有 RecoveryPause / RecoveryApplyDelay / conflict wait；
startup process 可能无 wait event、CPU 高，或出现 data file I/O wait。
```
源码回扣：
```text
ApplyWalRecord()
  -> 设置 replayEndRecPtr
  -> rmgr redo
  -> 成功后设置 lastReplayedEndRecPtr
```
如果当前源码的 `pg_stat_recovery` 可用，
看：
```text
replay_end_lsn
last_replayed_end_lsn
```
`replay_end_lsn` 领先 `last_replayed_end_lsn`
说明 startup process 可能正在处理一条 record。
如果这个差长时间不动，
再结合 wait event / perf / I/O 判断 redo 函数是否卡在 CPU、buffer I/O 或锁。
REDO 慢常见来源不是复制协议本身：
```text
大量 full-page image
index cleanup / btree delete / page split redo
relation extension / smgr I/O
checkpoint 后数据页读取
standby 缓存冷
IOPS 饱和
```
这类问题不能靠 `write_lag` 解释。
要回到 rmgr redo、buffer manager 和 storage。
### 9.6 Hot Standby 查询冲突或 query blocking
典型现象：
```text
standby receive/flush 已追上；
replay_lsn 不前进；
startup process 出现 RecoveryConflictSnapshot、RecoveryConflictTablespace、
BufferCleanup 或 lock wait；
日志可能出现 recovery still waiting / finished waiting；
pg_stat_database_conflicts 计数增加。
```
snapshot conflict 的源码链：
```text
rmgr redo 遇到 cleanup conflict horizon
  -> ResolveRecoveryConflictWithSnapshot()
  -> GetConflictingVirtualXIDs()
  -> ResolveRecoveryConflictWithVirtualXIDs(... RecoveryConflictSnapshot ...)
  -> 等到 max_standby_streaming_delay / max_standby_archive_delay
  -> SignalRecoveryConflictWithVirtualXID()
```
被取消 backend 的链：
```text
PROCSIG_RECOVERY_CONFLICT
  -> HandleRecoveryConflictInterrupt()
  -> ProcessRecoveryConflictInterrupts()
  -> report_recovery_conflict()
  -> pgstat_report_recovery_conflict()
  -> ERROR 或 FATAL
```
`pg_stat_database_conflicts` 是累计计数。
当前源码字段包括：
```text
confl_tablespace
confl_lock
confl_snapshot
confl_bufferpin
confl_deadlock
confl_active_logicalslot
```
它不能告诉你当前 blocker 是哪个 pid。
当前 blocker 要看：
```text
startup process wait_event
server log
standby 上长事务 / idle in transaction
backend_xmin
open cursor
lock wait
buffer pin 相关 wait
```
特别注意 buffer pin。
`ResolveRecoveryConflictWithBufferPin()` 等的是 `WAIT_EVENT_BUFFER_CLEANUP`。
wait event 名字不是 `RecoveryConflictBufferPin`。
`confl_bufferpin` 增加只能说明曾经取消过相关 backend。
### 9.7 recovery pause / target recovery / apply delay
典型现象：
```text
receive/flush 前进；
replay_lsn 停住；
startup process wait_event = RecoveryPause 或 RecoveryApplyDelay；
pg_get_wal_replay_pause_state() 不是 'not paused'；
pg_stat_recovery.pause_state 显示 pause requested / paused。
```
`pg_wal_replay_pause()` 源码链：
```text
pg_wal_replay_pause()
  -> SetRecoveryPause(true)
  -> WakeupRecovery()
startup redo loop:
  -> 检查 recoveryPauseState
  -> recoveryPausesHere(false)
  -> ConditionVariableTimedSleep(... WAIT_EVENT_RECOVERY_PAUSE)
```
`recovery_target_action=pause` 的链：
```text
recovery target reached
  -> SetRecoveryPause(true)
  -> recoveryPausesHere(true)
```
`recovery_min_apply_delay` 的链：
```text
recoveryApplyDelay()
  -> 只对达到一致性后的 commit / commit prepared 记录考虑
  -> 用 WAL record 的事务时间 + delay 计算 delayUntil
  -> WaitLatch(... WAIT_EVENT_RECOVERY_APPLY_DELAY)
```
源码注释还说明：
这个 delay 使用 WAL record log time 与 standby 当前时间比较，
会受时钟同步影响。
因此：
```text
RecoveryApplyDelay:
  是配置要求的延迟，不是 REDO 慢。
RecoveryPause:
  是人为 pause 或 target action，不是 network 慢。
pause requested:
  表示请求已设置，但 startup process 可能还没有到 pause point。
```
### 9.8 WAL 不可用或 stream source 切换
如果 startup process 等不到目标 WAL，
`WaitForWALToBecomeAvailable()` 在这些 source 之间循环：
```text
archive
pg_wal
stream
rescan timelines
wal_retrieve_retry_interval sleep
```
典型 wait event：
```text
RecoveryWalStream:
  正在等 walreceiver 带来更多 streaming WAL。
RecoveryRetrieveRetryInterval:
  archive、pg_wal、stream 都没拿到目标 WAL，等待重试。
WalReceiverWaitStart:
  walreceiver 等 startup process 给新的 startpoint。
```
这类问题会表现为 replay_lsn 不前进，
但不一定是 replay 慢。
它可能是：
```text
上游没有 WAL；
primary_conninfo 不可用；
slot / timeline / archive 问题；
walreceiver 断开；
目标 timeline 不存在或尚未发现 history file。
```
诊断时要先问：
```text
standby 已经 durable 收到目标 LSN 了吗？
```
如果没有，
就不要进入 REDO 性能分析。
## 10. Cascading Standby 诊断边界
cascading standby 同时是上游的 downstream、下游的 upstream。
因此它有两张状态面。
在中间节点 B 上，`pg_stat_wal_receiver` 描述 A -> B 的接收、write、flush、message time；`pg_stat_replication` 描述 B -> C 的 sent/write/flush/replay。
源码线索是 `walsender.c` 的 `am_cascading_walsender = RecoveryInProgress()`、`walreceiver.c` 中 flush 后的 `WalSndWakeup(true, false)`、以及 `xlogrecovery.c` replay 后按 timeline / logical 需要唤醒 walsender。
分段诊断：
```text
A -> B:
  A.pg_stat_replication(B) + B.pg_stat_wal_receiver
B replay:
  B.pg_last_wal_receive_lsn() / pg_last_wal_replay_lsn() + startup wait event
B -> C:
  B.pg_stat_replication(C) + C.pg_stat_wal_receiver
```
`B.pg_stat_wal_receiver.flushed_lsn` 追上 A 不代表 C 追上；B 可能没发出，C 也可能 write/flush/replay 慢。
A 上看到 B 的 `replay_lsn` 落后，也不能直接说明 B 不能给 C 发送；那是 A 对 B 的 apply 反馈，不是 B -> C 的发送阶段。
## 11. 常见误区
误区一：把 `sent_lsn` 当成 standby 已收到。
`sent_lsn` 来自 upstream `WalSnd.sentPtr`。
它在 `pq_putmessage_noblock()` 后推进。
standby 是否收到，要看 standby message time / latest_end_lsn / write_lsn。
误区二：把 `pg_last_wal_receive_lsn()` 当成 `written_lsn`。
源码调用的是 `GetWalRcvFlushRecPtr()`。
它对应 durable receive / flushed boundary。
误区三：把 `write_lag`、`flush_lag`、`replay_lag` 当作当前停顿时长。
它们来自 `LagTracker` 样本。
空闲后会清空。
样本不足时为 NULL。
它们用于估算已回报位置跨过某些 WAL flush sample 的 elapsed time。
误区四：只看 byte lag 不看 wait event。
`flush_lsn - replay_lsn` 大只能说明 WAL durable 接收领先 replay。
原因可能是 REDO CPU、data I/O、RecoveryPause、RecoveryApplyDelay、
snapshot conflict、lock conflict 或 buffer pin。
误区五：只看 wait event 不看 LSN。
`WalSenderMain` 可能是正常空闲，也可能是 socket backpressure。
`RecoveryWalStream` 可能说明等新 WAL，也可能说明 walreceiver 还没带来目标 WAL。
没有 LSN delta，wait event 很容易被过度解释。
误区六：看到 conflict counter 增加就认为当前仍被 blocking。
`pg_stat_database_conflicts` 是累计计数。
当前是否在等待，要看 startup process wait event、日志和当前查询。
误区七：用 `now() - pg_last_xact_replay_timestamp()` 作为绝对 replay lag。
这个 timestamp 来自 WAL 中最近 commit / abort 的事务时间。
非事务完成 WAL 不更新它。
跨机器时钟差会污染结果。
误区八：在 cascading standby 上混用上游和下游视图。
中间节点的 `pg_stat_wal_receiver` 和 `pg_stat_replication`
描述的是两个方向。
必须分段诊断。
## 12. 课堂实验
### 实验一：pause 下 receive/flush 与 replay 分离
步骤：搭建 primary + standby；standby 执行 `pg_wal_replay_pause()`；primary 产生 WAL；分别查询 `pg_stat_replication`、`pg_stat_wal_receiver`、`pg_last_wal_receive_lsn()`、`pg_last_wal_replay_lsn()`、`pg_get_wal_replay_pause_state()`；最后 `pg_wal_replay_resume()`。
预期：receive/flush 继续前进，replay 停住；startup wait event 可能是 `RecoveryPause`。
源码回到 `SetRecoveryPause()`、`WakeupRecovery()`、`recoveryPausesHere()`。
### 实验二：观察 standby write / flush 边界
步骤：对 standby WAL 目录施加 I/O 压力；primary 持续产生 WAL；观察 `pg_stat_wal_receiver.written_lsn/flushed_lsn`、`pg_last_wal_receive_lsn()`、walreceiver wait event。
预期：`written_lsn` 可领先 `flushed_lsn`，而 `pg_last_wal_receive_lsn()` 对齐 flush。
断点放在 `XLogWalRcvWrite()`、`XLogWalRcvFlush()`、`pg_stat_get_wal_receiver()`。
### 实验三：Hot Standby conflict 造成 replay lag
步骤：standby 开长事务或 cursor；primary 做 delete + vacuum 等会产生 cleanup conflict 的操作；观察 startup wait event、`pg_stat_database_conflicts` 和 `log_recovery_conflict_waits` 日志。
预期：flush 已追上但 replay 不动，原因是 conflict，而不是 network。
源码链是 `ResolveRecoveryConflictWithSnapshot()`、`ResolveRecoveryConflictWithVirtualXIDs()`、`SignalRecoveryConflictWithVirtualXID()`、`ProcessRecoveryConflictInterrupts()`。
## 13. 讨论题与诊断练习题
1. `sent_lsn` 等于 `pg_current_wal_flush_lsn()`，`write_lsn` 落后 5GB，standby `last_msg_receipt_time` 更新且 `latest_end_lsn` 前进但 `written_lsn` 不动；卡点在哪里？
2. `written_lsn` 比 `flushed_lsn` 领先 2GB，walreceiver wait event 是 `WalSync`；为什么 `pg_last_wal_receive_lsn()` 不追 `written_lsn`？
3. `flush_lsn` 追上 `sent_lsn`，`replay_lsn` 不动，startup wait event 是 `RecoveryPause`；这和 REDO 慢有什么区别？
4. `replay_lag` 为 NULL，但 `pg_wal_lsn_diff(flush_lsn, replay_lsn)` 很大；列出两个源码层面的原因。
5. `confl_snapshot` 增加，但当前没有 `RecoveryConflictSnapshot` wait event；为什么不矛盾？
6. A -> B -> C 中 B 的 `pg_stat_wal_receiver.flushed_lsn` 追上 A，C 仍落后；应在 B 看哪张视图？
7. 为什么 `last_msg_send_time` 与 `last_msg_receipt_time` 的差不能当网络 SLA？
8. quorum sync 下 `sync_state='quorum'` 是否代表这个 standby 最慢？回到 `syncrep.c` 哪层状态解释？
## 14. 本节小结
物理复制延迟诊断的稳定 mental model 是：
```text
一个 WAL 位置经过 primary flush、walsender sent、standby write、
standby flush、startup replay 和 standby feedback 多个边界；
每个边界有不同 owner、锁、cleanup 和观测入口。
```
`pg_stat_replication` 是 upstream walsender 视角。
`pg_stat_wal_receiver` 是 standby receiver 视角。
`pg_last_wal_receive_lsn()` 是 standby durable receive 边界。
`pg_last_wal_replay_lsn()` 是 startup process 已完成 replay 边界。
`pg_last_xact_replay_timestamp()` 是最近事务完成 WAL 的时间，
不是通用 lag clock。
诊断顺序应固定：
```text
先用 LSN delta 定位阶段；
再用 wait event 判断当前等待点；
再用 replay pause、apply delay、conflict counters 和日志排除伪因；
最后在 cascading topology 中按每条连接分段分析。
```
本节可迁移的系统规律是：
```text
跨进程诊断不能把 raw field 当语义。
field + owner process + lifecycle + wait point + feedback timing
才构成可解释状态。
```
仍然需要保留边界感：
```text
write_lag / flush_lag / replay_lag 是样本估计；
wait event 是当前等待点，不是完整因果；
conflict stats 是累计结果，不是当前 blocker；
timestamp 受 workload 和时钟影响；
跨机器、跨 cascading 节点的视图没有原子一致性。
```
