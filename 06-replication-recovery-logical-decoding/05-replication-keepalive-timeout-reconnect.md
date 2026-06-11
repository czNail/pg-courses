# PostgreSQL 复制协议 keepalive、超时与断线重连
## 课程定位
前置知识：已经理解 physical streaming replication 的复制连接、`START_REPLICATION`、`walsender` 主循环、`walreceiver` 写入 WAL segment，以及 `sent_lsn`、`write_lsn`、`flush_lsn`、`replay_lsn` 的基本含义。
本节唯一主问题：
```text
primary 和 standby 如何用 keepalive、reply_requested、wal_sender_timeout、
wal_receiver_timeout 和 reconnect 逻辑区分正常空闲、网络中断和 standby 落后？
```
核心矛盾：复制链路大部分时间可能没有新 WAL，但系统仍要判断对端是否活着；同时 standby 可能真的落后很多，却仍在正常接收、写盘或回放。PostgreSQL 不能把“没有新 WAL”“没有收到任何网络消息”“LSN 差距很大”混成同一种状态。
学完后应能独立判断：
```text
正常空闲:
  WAL 没有推进，但双方仍通过状态消息和 keepalive 证明连接活着。
网络中断:
  某一方向长时间收不到任何消息，timeout 把连接转成错误路径。
standby 落后:
  primary 仍收到 standby 回复，只是 write/flush/apply LSN 落后 sent LSN。
重连:
  walreceiver 退出后由 startup process 重新进入 WAL 获取状态机，
  再通过 RequestXLogStreaming() 请求 postmaster 启动或唤醒 walreceiver。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
本节只展开 physical streaming standby 的 keepalive、timeout 和 walreceiver 重连。逻辑复制 apply worker 也有连接、超时和重试，但它的 owner、catalog 状态和 worker 调度边界不同，不放进这一节。
## 1. 本节在总主线中的位置
前几节建立了复制数据面：
```text
standby 发起 replication connection
  -> primary walsender 进入 START_REPLICATION
  -> walsender 读取并发送 WALData
  -> walreceiver 接收、写入、flush
  -> startup process 从本地 pg_wal 回放
  -> standby 把 write/flush/apply 位置回报 primary
```
本节进入控制面：
```text
没有新 WAL 时，双方如何确认对端还活着？
有网络断点时，谁先报错，谁负责重连？
有复制延迟时，为什么不能只看 timeout？
```
这一节的主线不是“复制协议有哪些消息”，而是“时间如何推进状态”：
```text
primary 的时钟:
  last_reply_timestamp
    -> wal_sender_timeout / 2 发送 keepalive(reply_requested = true)
    -> wal_sender_timeout 仍无 standby reply 就关闭 walsender
standby 的时钟:
  WalRcvData.lastMsgReceiptTime / wakeup[WALRCV_WAKEUP_*]
    -> wal_receiver_timeout / 2 发送 StandbyStatusUpdate(reply_requested = true)
    -> wal_receiver_timeout 仍无 primary message 就退出 walreceiver
startup process 的状态机:
  WAL 不可用或 walreceiver 退出
    -> archive / pg_wal / stream 间切换
    -> wal_retrieve_retry_interval 后重试
    -> RequestXLogStreaming() 重新启动或唤醒 walreceiver
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
walsender 用“上次收到 standby 任意回复”的时间检测下游是否还活着；
walreceiver 用“上次收到 primary 任意消息”的时间检测上游是否还活着；
LSN 差距只表示复制进度，不直接表示连接死亡；
walreceiver 死亡后，startup process 通过 recovery WAL source 状态机重启流复制。
```
这里的 tension 是：
```text
空闲时不能误判断线
  vs
断线时不能无限等待
  vs
standby 落后时不能把慢当成死
```
如果只看“有没有新 WAL”，正常空闲会被误判成断线。
如果只看“LSN 是否相等”，磁盘慢、回放冲突、查询延迟都会被误判成网络故障。
如果只依赖 TCP keepalive，PostgreSQL 就无法在复制协议层把断线与延迟、空闲、timeline 切换、状态上报联系起来。
PostgreSQL 的做法是把问题拆成三类状态：
```text
协议活性:
  双方是否还在收发任意复制协议消息。
传输进度:
  primary 已发送到哪里，standby 已写入和 flush 到哪里。
回放进度:
  startup process 已 replay 到哪里，是否被 conflict、delay 或 IO 卡住。
```
`wal_sender_timeout` 和 `wal_receiver_timeout` 只回答第一类问题。
`sent_lsn`、`write_lsn`、`flush_lsn`、`replay_lsn` 回答第二、三类问题。
重连逻辑回答的是：
```text
当协议活性已经失败，standby 如何重新进入 WAL 获取流程？
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/replication/walsender.c` | `WalSndLoop()`、`ProcessRepliesIfAny()`、`ProcessStandbyReplyMessage()`、`WalSndKeepaliveIfNecessary()`、`WalSndCheckTimeOut()`、`XLogSendPhysical()`。 |
| 2 | `src/include/replication/walsender_private.h` | `WalSndState`、`WalSnd.sentPtr`、`write`、`flush`、`apply`、lag 和 `replyTime`。 |
| 3 | `src/include/replication/walsender.h` | `wal_sender_timeout` 对外声明。 |
| 4 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()`、`XLogWalRcvProcessMsg()`、`XLogWalRcvSendReply()`、`ProcessWalSndrMessage()`、`WalRcvComputeNextWakeup()`、`WalRcvDie()`。 |
| 5 | `src/include/replication/walreceiver.h` | `WalRcvData`、`WalRcvState`、walreceiver hooks 和 `wal_receiver_timeout`。 |
| 6 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | `libpqrcv_receive()` / `libpqrcv_send()` 如何把 libpq COPY 状态转成 walreceiver 的收发语义。 |
| 7 | `src/backend/replication/walreceiverfuncs.c` | startup process 与 walreceiver 的共享内存协作：`RequestXLogStreaming()`、`ShutdownWalRcv()`、`WalRcvStreaming()`、`GetWalRcvFlushRecPtr()`。 |
| 8 | `src/backend/access/transam/xlogrecovery.c` | `WaitForWALToBecomeAvailable()` 如何在 archive、pg_wal、stream 和 retry 之间推进，并在断线后触发重连。 |
| 9 | `src/backend/utils/misc/guc_parameters.dat` | 当前 master 中 `wal_sender_timeout`、`wal_receiver_timeout`、`wal_receiver_status_interval` 的 GUC 元数据。 |
注意：用户常会在旧资料里看到 `guc_tables.c`。本课基线的源码中，GUC 条目来自 `guc_parameters.dat`，生成后的表才进入相关 GUC 代码路径。课程以真实本地源码为准。
推荐阅读顺序：
```text
先读协议消息的生成和消费:
  WalSndKeepalive()
  XLogWalRcvSendReply()
  XLogWalRcvProcessMsg()
  ProcessStandbyReplyMessage()
再读两个 timeout 时钟:
  WalSndLoop()
  WalSndComputeSleeptime()
  WalSndKeepaliveIfNecessary()
  WalSndCheckTimeOut()
  WalReceiverMain()
  WalRcvComputeNextWakeup()
最后读断线后的 owner:
  WalRcvDie()
  WaitForWALToBecomeAvailable()
  RequestXLogStreaming()
  ShutdownWalRcv()
```
不要从 `wal_sender_timeout` 的 GUC 定义开始读。GUC 只给出阈值；语义来自循环中什么时候更新 timestamp、什么时候强制 reply、什么时候退出和谁重启。
## 4. 关键状态：raw field 不是语义
### 4.1 walsender 本地时钟
`walsender.c` 中几个 static 状态决定 primary 侧 timeout：
| 字段 | 语义 |
| --- | --- |
| `last_processing` | 最近一次 `ProcessRepliesIfAny()` 开始处理输入的时间。 |
| `last_reply_timestamp` | 最近一次确实收到 standby 消息的时间；为 0 表示当前不参与 `wal_sender_timeout`。 |
| `waiting_for_ping_response` | primary 已经发过 request-reply keepalive，尚未收到任意 standby reply。 |
| `sentPtr` | walsender 已经发送到的下一字节位置，也写入 `MyWalSnd->sentPtr`。 |
| `WalSndCaughtUp` | 当前 walsender 是否已经把可发送 WAL 追上。 |
这些字段组合后才有语义：
```text
last_reply_timestamp 有值
  + wal_sender_timeout > 0
  + waiting_for_ping_response = false
  + last_processing >= last_reply_timestamp + wal_sender_timeout / 2
    -> 发送 keepalive(reply_requested = true)
last_reply_timestamp 有值
  + wal_sender_timeout > 0
  + last_processing >= last_reply_timestamp + wal_sender_timeout
    -> 认为 standby 连接失活，关闭 walsender
```
`last_processing` 而不是 `GetCurrentTimestamp()` 直接参与 timeout 检查，是为了避免把 server-side 长时间停顿简单算到 standby 头上。源码注释明确说明这个选择是近似，不是分布式时钟保证。
### 4.2 `WalSnd` 共享状态
`src/include/replication/walsender_private.h` 的 `WalSnd` 是 `pg_stat_replication` 的主要来源。
本节关注这些字段：
| 字段 | 来源 | 诊断含义 |
| --- | --- | --- |
| `state` | `WalSndSetState()` | `startup`、`catchup`、`streaming`、`stopping` 等 walsender 状态。 |
| `sentPtr` | `XLogSendPhysical()` / logical path | primary 已经放进复制流的 WAL 位置。 |
| `write` | standby status reply | standby 已经写入的 LSN。 |
| `flush` | standby status reply | standby 已经 flush 的 LSN。 |
| `apply` | standby status reply | standby 已经 replay 的 LSN。 |
| `writeLag` / `flushLag` / `applyLag` | `LagTrackerRead()` | 由 primary 采样 WAL send time 和 standby reply 计算的近似延迟。 |
| `replyTime` | standby status / feedback message | standby 生成回复时携带的 timestamp。 |
`WalSnd.state = streaming` 不等于 standby 已经 replay 到最新。
它更接近：
```text
walsender 当前已经把 primary 可发送的 WAL 追上，
从 catchup 进入正常 streaming 循环。
```
standby 是否落后，要看：
```text
sent_lsn - write_lsn
sent_lsn - flush_lsn
sent_lsn - replay_lsn
reply_time 是否仍新鲜
```
### 4.3 walreceiver 共享状态
`src/include/replication/walreceiver.h` 的 `WalRcvData` 是 standby 本地 `pg_stat_wal_receiver` 的主要来源。
本节关注这些字段：
| 字段 | 更新者 | 语义 |
| --- | --- | --- |
| `walRcvState` | startup process / walreceiver | `STOPPED`、`STARTING`、`CONNECTING`、`STREAMING`、`WAITING`、`RESTARTING`、`STOPPING`。 |
| `receiveStart` / `receiveStartTLI` | `RequestXLogStreaming()` | startup process 希望 walreceiver 从哪里开始拉流。 |
| `flushedUpto` / `receivedTLI` | `XLogWalRcvFlush()` | walreceiver 已经 flush 到本地 pg_wal 的位置和 timeline。 |
| `latestChunkStart` | `XLogWalRcvFlush()` | 最近一次 flush 前的起点，startup 用它判断是否追上新 chunk。 |
| `writtenUpto` | `XLogWalRcvWrite()` | 已写入但不一定 fsync 的位置，atomic 读写。 |
| `lastMsgSendTime` | `ProcessWalSndrMessage()` | primary 消息里携带的 send timestamp。 |
| `lastMsgReceiptTime` | `ProcessWalSndrMessage()` | standby 收到 primary 消息的本地时间。 |
| `latestWalEnd` / `latestWalEndTime` | `ProcessWalSndrMessage()` | primary 报告的 WAL end 和对应时间。 |
| `apply_reply_requested` | startup process | replay 有进展时请求 walreceiver 立即回报 apply LSN。 |
`latestWalEnd` 表示 primary 在消息中告诉 standby 的上游 WAL 终点，不等于 standby 已写入或已回放。
`flushedUpto` 表示 walreceiver 写盘边界，不等于 startup process 回放边界。
`lastMsgReceiptTime` 是协议活性的关键观测值。如果它持续不变，但 `wal_receiver_timeout` 已过，standby 会认为上游没有任何消息。
### 4.4 复制协议里的两类 request-reply
primary 到 standby 的 keepalive：
```text
PqReplMsg_Keepalive:
  walEnd
  sendTime
  replyRequested
```
standby 到 primary 的 status update：
```text
PqReplMsg_StandbyStatusUpdate:
  writePtr
  flushPtr
  applyPtr
  replyTime
  replyRequested
```
两个方向都有 `replyRequested`，但含义都很克制：
```text
我希望你立刻给我一个协议层响应，以证明链路还活着。
```
它不表示：
```text
对方已经落后。
对方必须 flush。
对方必须 replay。
这是同步复制 commit 等待的完成条件。
```
primary 发 `replyRequested = true` 后，standby 在 `XLogWalRcvProcessMsg()` 收到 keepalive 时立即调用：
```text
XLogWalRcvSendReply(true, false, false)
```
standby 发 `replyRequested = true` 后，primary 在 `ProcessStandbyReplyMessage()` 中调用：
```text
WalSndKeepalive(false, InvalidXLogRecPtr)
```
注意 primary 的响应 keepalive 是 `requestReply = false`。否则双方在 timeout 边缘可能把 request-reply 变成 ping-pong。
## 5. 主流程源码 walkthrough：正常流复制如何维持活性
### 5.1 standby 请求 streaming
startup process 在 `WaitForWALToBecomeAvailable()` 中找不到目标 WAL 时，会进入 stream source：
```text
archive / pg_wal 读取失败
  -> currentSource = XLOG_FROM_STREAM
  -> startWalReceiver = true
  -> RequestXLogStreaming(tli, ptr, PrimaryConnInfo, PrimarySlotName, ...)
```
`RequestXLogStreaming()` 在 `walreceiverfuncs.c` 中完成 shared state 发布：
```text
如果 WalRcv 是 STOPPED:
  walRcvState = WALRCV_STARTING
  保存 conninfo / slotname
  初始化 receiveStart / receiveStartTLI / flushedUpto / writtenUpto
  SendPostmasterSignal(PMSIGNAL_START_WALRECEIVER)
如果 WalRcv 是 WAITING:
  walRcvState = WALRCV_RESTARTING
  更新 receiveStart / receiveStartTLI
  SetLatch(walreceiver)
```
这里已经能看到 reconnect 的 owner：
```text
walreceiver 不自己无限重连；
startup process 负责决定还缺哪个 WAL，并请求 walreceiver 从哪个 LSN/TLI 拉。
```
### 5.2 walreceiver 启动并进入 STREAMING
`WalReceiverMain()` 一开始把自己发布到 `WalRcvData`：
```text
walRcvState: STARTING -> CONNECTING
pid = MyProcPid
procno = MyProcNumber
lastMsgSendTime / lastMsgReceiptTime / latestWalEndTime = now
```
然后加载 `libpqwalreceiver`，连接 primary，执行：
```text
IDENTIFY_SYSTEM
timeline history checks
START_REPLICATION
```
`walrcv_startstreaming()` 成功后：
```text
walRcvState: CONNECTING -> STREAMING
LogstreamResult.Write = LogstreamResult.Flush = GetXLogReplayRecPtr(NULL)
初始化 WALRCV_WAKEUP_TERMINATE / PING / REPLY / HSFEEDBACK
XLogWalRcvSendReply(true, false, false)
XLogWalRcvSendHSFeedback(true)
```
这个 initial reply 很重要：它让 primary 尽快看到 standby 的 write/flush/apply 位置，而不是等到第一个 status interval。
### 5.3 walsender 进入主循环
primary 端收到 `START_REPLICATION` 后，physical path 设置：
```text
WalSndSetState(WALSNDSTATE_CATCHUP)
sentPtr = requested startpoint
MyWalSnd->sentPtr = sentPtr
WalSndLoop(XLogSendPhysical)
```
`WalSndLoop()` 进入时初始化：
```text
last_reply_timestamp = GetCurrentTimestamp()
waiting_for_ping_response = false
```
随后每轮：
```text
ResetLatch(MyLatch)
CHECK_FOR_INTERRUPTS()
WalSndHandleConfigReload()
ProcessRepliesIfAny()
send_data()
pq_flush_if_writable()
WalSndCheckTimeOut()
WalSndCheckShutdownTimeout()
WalSndKeepaliveIfNecessary()
WalSndWait(...)
```
这说明 timeout 和 keepalive 不是独立后台定时器，而是嵌在 walsender 的 send/receive 循环里。
### 5.4 primary 发送 WALData
`XLogSendPhysical()` 先决定最多可以发送到哪里：
```text
普通 primary:
  SendRqstPtr = GetFlushRecPtr(NULL)
cascading standby:
  SendRqstPtr = GetStandbyFlushRecPtr(&SendRqstTLI)
historic timeline:
  SendRqstPtr = sendTimeLineValidUpto
```
如果没有新 WAL：
```text
if SendRqstPtr <= sentPtr:
  WalSndCaughtUp = true
  return
```
如果有 WAL，则构造：
```text
PqReplMsg_WALData:
  dataStart = startptr
  walEnd = SendRqstPtr
  sendTime = GetCurrentTimestamp()
  payload = WAL bytes
```
然后：
```text
pq_putmessage_noblock(CopyData)
sentPtr = endptr
MyWalSnd->sentPtr = sentPtr
```
这里的 `walEnd` 是 primary 当前知道的 WAL end，不一定等于本消息 payload 的结束位置。它帮助 standby 观测“上游已经到哪里”。
### 5.5 standby 接收 WALData
`WalReceiverMain()` 的内层循环调用：
```text
len = walrcv_receive(wrconn, &buf, &wait_fd)
```
`libpqrcv_receive()` 用 `PQgetCopyData(..., async = 1)`：
```text
rawlen > 0:
  返回一条 CopyData
rawlen == 0:
  PQconsumeInput()
  如果仍无数据，返回 0 并给出 wait_fd
rawlen == -1:
  COPY 结束或错误，返回 -1 或 ERROR
```
收到消息后，`XLogWalRcvProcessMsg()` 根据首字节分发：
```text
PqReplMsg_WALData:
  读取 dataStart / walEnd / sendTime
  ProcessWalSndrMessage(walEnd, sendTime)
  XLogWalRcvWrite(payload, dataStart, tli)
PqReplMsg_Keepalive:
  读取 walEnd / sendTime / replyRequested
  ProcessWalSndrMessage(walEnd, sendTime)
  如果 replyRequested:
    XLogWalRcvSendReply(true, false, false)
```
`ProcessWalSndrMessage()` 只处理控制面状态：
```text
lastMsgReceiptTime = GetCurrentTimestamp()
walrcv->latestWalEnd = walEnd
walrcv->lastMsgSendTime = sendTime
walrcv->lastMsgReceiptTime = lastMsgReceiptTime
如果 walEnd 增大，更新 latestWalEndTime
```
即使 keepalive 没有 WAL payload，它也会刷新 `lastMsgReceiptTime`。这就是正常空闲不会触发 `wal_receiver_timeout` 的关键。
### 5.6 standby 写入和 flush 后回报
`XLogWalRcvWrite()` 把收到的 WAL 写入本地 pg_wal：
```text
写 segment
LogstreamResult.Write = recptr
WalRcv->writtenUpto = recptr
WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_WRITE, recptr)
```
`XLogWalRcvFlush(false, tli)` 负责 fsync 并发布：
```text
LogstreamResult.Flush = LogstreamResult.Write
WalRcv->flushedUpto = LogstreamResult.Flush
WalRcv->receivedTLI = tli
WakeupRecovery()
WalSndWakeup(...) if cascading
XLogWalRcvSendReply(false, false, false)
XLogWalRcvSendHSFeedback(false)
```
这里把三个模块串起来：
```text
walreceiver:
  收到并写入 WAL
startup process:
  被 WakeupRecovery() 唤醒后从 pg_wal 读取并 redo
primary walsender:
  通过 standby status update 看到 write / flush / apply 位置
```
### 5.7 primary 处理 standby status
`ProcessRepliesIfAny()` 在 walsender 循环开头读取 standby 发来的 CopyData。只要收到任意复制消息：
```text
received = true
...
if received:
  last_reply_timestamp = last_processing
  waiting_for_ping_response = false
```
status update 本身由 `ProcessStandbyReplyMessage()` 解析：
```text
writePtr = pq_getmsgint64()
flushPtr = pq_getmsgint64()
applyPtr = pq_getmsgint64()
replyTime = pq_getmsgint64()
replyRequested = pq_getmsgbyte()
```
然后：
```text
如果 replyRequested:
  WalSndKeepalive(false, InvalidXLogRecPtr)
更新 MyWalSnd->write / flush / apply / replyTime
更新 writeLag / flushLag / applyLag
SyncRepReleaseWaiters()
根据 flushPtr 推进 replication slot confirmed location
```
这说明 primary 对“standby 活着”的判断不要求 LSN 前进：
```text
只要收到 status update，
last_reply_timestamp 就会更新。
```
LSN 是否前进，则由 `write`、`flush`、`apply` 字段单独表达。
## 6. 正常空闲：没有 WAL，不等于没有消息
正常空闲的典型状态：
```text
primary 没有新 WAL:
  XLogSendPhysical() 发现 SendRqstPtr <= sentPtr
  WalSndCaughtUp = true
standby 已追上:
  write_lsn = flush_lsn = replay_lsn = sent_lsn
双方仍有控制面消息:
  standby 按 wal_receiver_status_interval 发送 status update
  primary 必要时发送 keepalive
```
在 `WalSndLoop()` 中，如果 walsender 已 catch up 且没有 pending output，会进入 `WalSndWait()`：
```text
wakeEvents = WL_SOCKET_READABLE
sleeptime = WalSndComputeSleeptime(now)
WalSndWait(... WAIT_EVENT_WAL_SENDER_MAIN)
```
`WalSndComputeSleeptime()` 不会随意睡 10 秒到底。若 timeout 启用，它会选择更早的时间点：
```text
如果还没 ping:
  wake at last_reply_timestamp + wal_sender_timeout / 2
如果已经 ping:
  wake at last_reply_timestamp + wal_sender_timeout
```
standby 侧也类似。`WalReceiverMain()` 没读到数据后，会计算多个 wakeup 的最早时间：
```text
WALRCV_WAKEUP_TERMINATE:
  now + wal_receiver_timeout
WALRCV_WAKEUP_PING:
  now + wal_receiver_timeout / 2
WALRCV_WAKEUP_REPLY:
  now + wal_receiver_status_interval
WALRCV_WAKEUP_HSFEEDBACK:
  now + wal_receiver_status_interval if hot_standby_feedback
```
正常空闲下，`wal_receiver_status_interval` 默认 10 秒，`wal_sender_timeout` 默认 60 秒。standby 通常会比 primary 的半超时更早发送 status update，所以 primary 的 `waiting_for_ping_response` 很多时候不会变成 true。
可观察形态：
```text
primary pg_stat_replication:
  state = streaming
  sent_lsn = write_lsn = flush_lsn = replay_lsn
  reply_time 持续刷新，或者至少不超过 timeout 风险窗口
  write_lag / flush_lag / replay_lag 可能为 NULL
standby pg_stat_wal_receiver:
  status = streaming
  last_msg_receipt_time 周期性刷新
  latest_end_lsn 与 written_lsn / flushed_lsn 接近或相等
```
重点是：
```text
没有业务 WAL 不等于复制连接没有协议消息。
```
## 7. primary keepalive 与 `reply_requested`
`WalSndKeepaliveIfNecessary()` 是 primary 侧主动探测 standby 的入口：
```text
如果 wal_sender_timeout <= 0:
  return
如果 last_reply_timestamp <= 0:
  return
如果 waiting_for_ping_response:
  return
ping_time = last_reply_timestamp + wal_sender_timeout / 2
如果 last_processing >= ping_time:
  WalSndKeepalive(true, InvalidXLogRecPtr)
  pq_flush_if_writable()
```
`WalSndKeepalive(true, ...)` 写入：
```text
PqReplMsg_Keepalive
walEnd = sentPtr
sendTime = GetCurrentTimestamp()
replyRequested = 1
```
并设置：
```text
waiting_for_ping_response = true
```
这个 flag 的作用不是 correctness，而是避免在同一个 timeout 窗口内重复发送 request-reply keepalive：
```text
只要没收到 standby 任意回复，
primary 不需要每轮循环都发新的 request-reply ping。
```
standby 收到后走：
```text
XLogWalRcvProcessMsg(PqReplMsg_Keepalive)
  -> ProcessWalSndrMessage(walEnd, sendTime)
  -> if replyRequested:
       XLogWalRcvSendReply(true, false, false)
```
`force = true` 让它无视 `wal_receiver_status_interval`，立即发 status update。
`requestReply = false` 表示这个响应本身不要求 primary 再响应。
所以 primary keepalive 完成的闭环是：
```text
primary:
  半个 wal_sender_timeout 没收到 standby reply
  -> keepalive(reply_requested = true)
standby:
  收到 keepalive
  -> 更新 lastMsgReceiptTime
  -> 立即 StandbyStatusUpdate(reply_requested = false)
primary:
  收到 status update
  -> last_reply_timestamp = last_processing
  -> waiting_for_ping_response = false
```
这条链路只证明协议连接活着，并顺便带回 LSN。
如果 LSN 没变化，primary 仍认为连接活着；延迟诊断需要看 LSN 差距。
## 8. standby ping 与 `wal_receiver_timeout`
standby 的 timeout 方向相反：
```text
primary 没有发来任何 WALData 或 Keepalive
  -> lastMsgReceiptTime 不刷新
  -> wakeup[WALRCV_WAKEUP_PING] 到期
  -> standby 主动发 StandbyStatusUpdate(reply_requested = true)
```
`WalRcvComputeNextWakeup()` 给出两个关键时间：
```text
WALRCV_WAKEUP_PING:
  now + wal_receiver_timeout / 2
WALRCV_WAKEUP_TERMINATE:
  now + wal_receiver_timeout
```
当 `WaitLatchOrSocket()` 因 timeout 返回，walreceiver 做：
```text
now = GetCurrentTimestamp()
if now >= wakeup[WALRCV_WAKEUP_TERMINATE]:
  ERROR "terminating walreceiver due to timeout"
if now >= wakeup[WALRCV_WAKEUP_PING]:
  requestReply = true
  wakeup[WALRCV_WAKEUP_PING] = TIMESTAMP_INFINITY
XLogWalRcvSendReply(requestReply, requestReply, false)
XLogWalRcvSendHSFeedback(false)
```
注意这里两个参数都用 `requestReply`：
```text
force = true:
  即使 wal_receiver_status_interval = 0，也要发送 ping。
requestReply = true:
  要求 primary 立即给一个响应。
```
primary 收到这个 status update 后：
```text
ProcessStandbyReplyMessage()
  if replyRequested:
    WalSndKeepalive(false, InvalidXLogRecPtr)
```
standby 收到 primary 的 keepalive 后，`ProcessWalSndrMessage()` 刷新 `lastMsgReceiptTime`，下一轮 timeout 重新计算。
standby ping 完成的闭环是：
```text
standby:
  半个 wal_receiver_timeout 没收到 primary message
  -> StandbyStatusUpdate(reply_requested = true)
primary:
  收到 status update
  -> 更新 last_reply_timestamp
  -> keepalive(reply_requested = false)
standby:
  收到 keepalive
  -> 更新 lastMsgReceiptTime
```
这条链路解决的是“上游是不是还能给我任何协议响应”，不是“上游是否有新 WAL”。
## 9. 网络中断：两个 timeout 谁先触发都可以
网络中断不是一个单点状态。不同方向、不同 socket 行为、不同 OS 错误时，primary 和 standby 可能看到不同现象。
primary 侧典型路径：
```text
ProcessRepliesIfAny()
  pq_getbyte_if_available() 返回 EOF 或错误
  -> COMMERROR "unexpected EOF on standby connection"
  -> proc_exit(0)
或者:
  last_reply_timestamp + wal_sender_timeout 到期
  -> WalSndCheckTimeOut()
  -> COMMERROR "terminating walsender process due to replication timeout"
  -> WalSndShutdown()
```
standby 侧典型路径：
```text
libpqrcv_receive()
  PQconsumeInput() / PQgetCopyData() 报错
  -> ERROR "could not receive data from WAL stream: ..."
或者:
  wal_receiver_timeout 到期
  -> ERROR "terminating walreceiver due to timeout"
或者:
  primary 正常结束 COPY
  -> LOG "replication terminated by primary server"
  -> endofwal = true
```
walreceiver ERROR 后走 `on_shmem_exit(WalRcvDie, ...)`：
```text
XLogWalRcvFlush(true, tli)
walRcvState = WALRCV_STOPPED
pid = 0
procno = INVALID_PROC_NUMBER
ready_to_display = false
ConditionVariableBroadcast(walRcvStoppedCV)
walrcv_disconnect(wrconn)
WakeupRecovery()
```
这一步很关键：
```text
walreceiver 退出不是静默消失；
它把 shared state 转成 STOPPED，并唤醒 startup process。
```
startup process 被唤醒后，`WaitForWALToBecomeAvailable()` 会在下一次需要 WAL 时看到：
```text
currentSource == XLOG_FROM_STREAM
WalRcvStreaming() == false
  -> lastSourceFailed = true
```
然后流复制 source 失败：
```text
关闭或确认 walreceiver 不再运行
尝试 rescan timeline
如果短时间内刚失败过:
  LOG "waiting for WAL to become available at ..."
  WaitLatch(... WAIT_EVENT_RECOVERY_RETRIEVE_RETRY_INTERVAL)
currentSource = XLOG_FROM_ARCHIVE
```
下一轮从 archive / pg_wal 再试，失败后又回到 stream source：
```text
currentSource = XLOG_FROM_STREAM
startWalReceiver = true
RequestXLogStreaming(...)
```
这就是 standby physical streaming 的 reconnect：
```text
不是 walreceiver 内部 reconnect loop；
而是 startup process 的 WAL source 状态机在缺 WAL 时重新请求 streaming。
```
如果 `primary_conninfo` 配错或网络长期不可达，walreceiver 会反复启动、连接失败、STOPPED，startup process 间隔 `wal_retrieve_retry_interval` 重试，并穿插 archive / pg_wal 搜索。
## 10. standby 落后：LSN 差距大，但协议仍活着
standby 落后的典型状态：
```text
primary pg_stat_replication:
  sent_lsn > write_lsn
  或 sent_lsn > flush_lsn
  或 sent_lsn > replay_lsn
  reply_time 仍然刷新
standby pg_stat_wal_receiver:
  last_msg_receipt_time 仍然刷新
  latest_end_lsn 持续前进
  written_lsn / flushed_lsn 可能落后 latest_end_lsn
  pg_last_wal_replay_lsn() 可能落后 flushed_lsn
```
这与网络中断的本质区别是：
```text
standby 落后:
  控制面消息还在流动。
网络中断:
  至少一方在 timeout 窗口内收不到任何协议消息。
```
滞后可以发生在不同层次：
```text
sent_lsn - write_lsn 大:
  网络传输慢、walreceiver 来不及收、standby 写入慢，或 primary 发送速度远大于接收速度。
write_lsn - flush_lsn 大:
  standby WAL fsync 慢，存储压力大。
flush_lsn - replay_lsn 大:
  startup process REDO 慢、查询冲突、recovery_min_apply_delay、IO 竞争或 CPU 压力。
reply_time 新鲜但 replay_lsn 不动:
  walreceiver 活着，问题更可能在回放层，而不是连接层。
```
`wal_sender_timeout` 不应该因为 standby 落后而触发，只要 standby 仍发送 reply。
`wal_receiver_timeout` 也不应该因为 replay 慢而触发，只要 primary 仍发送 WALData 或 keepalive。
所以判断复制问题时不要先问：
```text
timeout 有没有触发？
```
而要先分层：
```text
协议消息是否还在刷新？
WAL 是否还在接收？
WAL 是否已经 flush？
WAL 是否已经 replay？
```
## 11. 生命周期 / ownership / cleanup：startup process 与 walreceiver 的协作边界
walreceiver 只负责：
```text
连接 primary
接收 WALData / Keepalive
写本地 WAL 文件
flush 后更新 WalRcvData
发送 status / feedback
失败时标记 STOPPED
```
startup process 负责：
```text
决定当前需要哪个 RecPtr / timeline
决定从 archive、pg_wal 还是 stream 获取
在 stream 失败后关停或重启 walreceiver
从本地 WAL 文件读取并 redo
在 replay 进展后请求 walreceiver 回报 apply LSN
```
两者共享边界在 `WalRcvData`：
```text
startup process 写:
  receiveStart
  receiveStartTLI
  conninfo
  slotname
  walRcvState STARTING / RESTARTING / STOPPING
walreceiver 写:
  pid / procno
  walRcvState CONNECTING / STREAMING / WAITING / STOPPED
  flushedUpto
  latestChunkStart
  writtenUpto
  lastMsgSendTime / lastMsgReceiptTime
  latestWalEnd
```
`WalRcvWaitForStartPosition()` 是 end-of-streaming 后的等待路径：
```text
walreceiver:
  walRcvState = WALRCV_WAITING
  receiveStart = InvalidXLogRecPtr
  WakeupRecovery()
  等 startup process 设置 WALRCV_RESTARTING
startup process:
  发现新 timeline 或新的需要位置
  RequestXLogStreaming()
  walRcvState = WALRCV_RESTARTING
  SetLatch(walreceiver)
```
这和 timeout 退出后的 STOPPED 路径不同：
```text
WAITING:
  同一个 walreceiver 连接还可以等待新指令。
STOPPED:
  walreceiver 已退出；需要 postmaster 重新启动。
```
`WalRcvRunning()` 和 `WalRcvStreaming()` 还处理 `WALRCV_STARTING` 超时：
```text
WALRCV_STARTUP_TIMEOUT = 10s
如果 STARTING 过久:
  walRcvState = WALRCV_STOPPED
  ConditionVariableBroadcast(walRcvStoppedCV)
```
这避免 startup process 永远等待一个 postmaster 没能成功启动的 walreceiver。
## 12. 正确性机制层次
| 层次 | 机制 | 解决的问题 |
| --- | --- | --- |
| 协议活性 | keepalive + status update + `replyRequested` | 空闲时确认对端仍能处理复制协议。 |
| primary timeout | `last_reply_timestamp` + `wal_sender_timeout` | primary 不无限等待失联 standby。 |
| standby timeout | `lastMsgReceiptTime` wakeup + `wal_receiver_timeout` | standby 不无限等待失联 primary。 |
| 进度表达 | `sentPtr`、`write`、`flush`、`apply` | 区分发送、写入、flush、回放。 |
| shared state 并发 | `WalSnd.mutex`、`WalRcv.mutex`、`writtenUpto` atomic | 让统计视图和进程间读取看到一致或有边界的状态。 |
| 唤醒 | latch、condition variable | 新 WAL、apply 进展、walreceiver exit 能唤醒等待者。 |
| 重连 owner | `WaitForWALToBecomeAvailable()` + `RequestXLogStreaming()` | 断线后由 startup process 决定从何处继续拉 WAL。 |
| fallback | archive / pg_wal / stream state machine | stream 失败不等于恢复失败，先尝试其它 WAL 来源。 |
`apply_reply_requested` 的边界值得单独看：
```text
startup process:
  replay 特定 WAL 后调用 XLogRequestWalReceiverReply()
  recovery main loop 里调用 WalRcvRequestApplyReply()
WalRcvRequestApplyReply():
  WalRcv->apply_reply_requested = true
  读取 walreceiver procno
  SetLatch(walreceiver)
walreceiver:
  看到 apply_reply_requested
  清 false
  pg_memory_barrier()
  XLogWalRcvSendReply(false, false, true)
```
这服务于 `synchronous_commit = remote_apply` 的及时反馈。它不是 timeout 机制，但它解释了为什么 apply LSN 不是只有 status interval 到期才更新。
## 13. 错误路径与异常路径
### 13.1 primary 认为 standby 超时
路径：
```text
WalSndLoop()
  -> ProcessRepliesIfAny()
  -> 没有收到 standby reply
  -> WalSndCheckTimeOut()
  -> last_processing >= last_reply_timestamp + wal_sender_timeout
  -> COMMERROR "terminating walsender process due to replication timeout"
  -> WalSndShutdown()
```
源码注释说明，典型 replication timeout 被视为通信问题，所以 walsender 不尝试把错误消息发给 standby。
诊断含义：
```text
primary 侧能看到 walsender 退出日志。
standby 侧可能随后看到 WAL stream receive error、primary 终止复制或自己的 wal_receiver_timeout。
```
### 13.2 standby 认为 primary 超时
路径：
```text
WalReceiverMain()
  -> WaitLatchOrSocket(... WAIT_EVENT_WAL_RECEIVER_MAIN)
  -> WL_TIMEOUT
  -> now >= wakeup[WALRCV_WAKEUP_TERMINATE]
  -> ERROR "terminating walreceiver due to timeout"
  -> WalRcvDie()
  -> WakeupRecovery()
```
诊断含义：
```text
standby 侧最先报错。
primary 侧可能还没到 wal_sender_timeout；
它可能在下一次 socket read/write 或 timeout 时才发现 standby 消失。
```
### 13.3 libpq 收发错误
`libpqrcv_receive()` 在 `PQconsumeInput()` 或 `PQgetCopyData()` 报错时：
```text
ERROR "could not receive data from WAL stream: ..."
```
`libpqrcv_send()` 在 `PQputCopyData()` 或 `PQflush()` 失败时：
```text
ERROR "could not send data to WAL stream: ..."
```
这些错误不经过 `wal_receiver_timeout`。例如 TCP 立即返回 reset，walreceiver 会马上 ERROR。
### 13.4 stream 结束但连接未坏
primary 可能因为 timeline end 或请求位置无 WAL 而结束 COPY。walreceiver 看到 `len < 0` 时：
```text
LOG "replication terminated by primary server"
endofwal = true
walrcv_endstreaming()
WalRcvFetchTimeLineHistoryFiles()
WalRcvWaitForStartPosition()
```
这不是网络断线。walreceiver 可能进入 `WALRCV_WAITING`，等待 startup process 根据 timeline history 或 recovery target 重新下发位置。
### 13.5 配置变化触发 restart
`StartupRequestWalReceiverRestart()` 在流复制期间注意到相关配置变化时：
```text
LOG "WAL receiver process shutdown requested"
pendingWalRcvRestart = true
```
`WaitForWALToBecomeAvailable()` 后续会：
```text
if pendingWalRcvRestart:
  XLogShutdownWalRcv()
  startWalReceiver = true
```
这也是 reconnect，但不是网络故障。诊断时要把配置 reload 与真实断线分开。
## 14. 成本、资源与跨模块传播
keepalive 本身很便宜：
```text
一个小 CopyData 消息
一次 timestamp
一次 nonblocking output flush 尝试
```
真正需要关注的成本在边界传播：
| 压力 | 表现 | 传播路径 |
| --- | --- | --- |
| 网络慢 | `sent_lsn - write_lsn` 增大 | primary 发送缓冲、walreceiver receive、status reply 滞后。 |
| standby WAL 写慢 | `written_lsn` 慢、`flush_lsn` 慢 | `XLogWalRcvWrite()` / `XLogWalRcvFlush()`、WAL fsync、`pg_stat_io`。 |
| REDO 慢 | `flush_lsn - replay_lsn` 增大 | startup process、conflict、recovery delay、IO/CPU。 |
| primary WAL 产生快 | `sent_lsn` 快速前进 | replication slot / WAL 保留压力可能增长。 |
| status interval 过大 | `reply_time` 刷新慢 | 正常空闲时可能更接近 timeout 半程才有 ping。 |
| timeout 设置过小 | 偶发抖动被误判 | 网络 jitter、长 GC/IO stall、调度延迟都可能触发断线。 |
`wal_sender_timeout` 和 `wal_receiver_timeout` 不是性能优化旋钮。调小它们能更快发现断线，但会降低对短暂网络抖动、进程调度停顿和 IO stall 的容忍度。
如果复制延迟来自 replay，调 timeout 通常没有帮助。
如果复制延迟来自网络丢包，timeout 只能更快重连，不能提升带宽。
如果 slot 保留导致 primary WAL 目录增长，本节的 keepalive 只能证明 standby 活着；保留风险要看 slot 的 `restart_lsn`、`wal_status`、`safe_wal_size` 和磁盘策略。
## 15. 观测与诊断入口
### 15.1 primary 侧：`pg_stat_replication`
核心查询：
```sql
SELECT pid,
       application_name,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag,
       reply_time,
       now() - reply_time AS reply_age
FROM pg_stat_replication;
```
字段来源：
```text
sent_lsn:
  WalSnd.sentPtr
write_lsn / flush_lsn / replay_lsn:
  ProcessStandbyReplyMessage() 从 standby status update 写入 WalSnd
write_lag / flush_lag / replay_lag:
  LagTrackerRead() 基于发送采样和 standby reply 估算
reply_time:
  standby 在 reply message 里携带的 timestamp
```
判断：
```text
reply_age 持续增大，接近 wal_sender_timeout:
  primary 没收到 standby reply，疑似连接问题。
reply_age 新鲜，但 sent_lsn - replay_lsn 大:
  连接活着，standby 回放落后。
sent_lsn = write_lsn = flush_lsn = replay_lsn，但 reply_time 周期刷新:
  正常空闲。
```
不要把 `write_lag` / `flush_lag` / `replay_lag` 当成绝对真相。它们依赖 primary 采样和 standby reply，空闲时还会被清空以避免展示陈旧 lag。
### 15.2 standby 侧：`pg_stat_wal_receiver`
核心查询：
```sql
SELECT pid,
       status,
       written_lsn,
       flushed_lsn,
       latest_end_lsn,
       last_msg_send_time,
       last_msg_receipt_time,
       now() - last_msg_receipt_time AS receive_age,
       latest_end_time,
       sender_host,
       sender_port
FROM pg_stat_wal_receiver;
```
字段来源：
```text
status:
  WalRcvGetStateString(WalRcv.walRcvState)
written_lsn:
  WalRcv.writtenUpto atomic
flushed_lsn:
  WalRcv.flushedUpto
latest_end_lsn:
  WalRcv.latestWalEnd
last_msg_send_time:
  primary message sendTime
last_msg_receipt_time:
  standby receive timestamp
```
判断：
```text
receive_age 接近 wal_receiver_timeout:
  standby 没收到 primary 任意消息。
latest_end_lsn 前进，written_lsn / flushed_lsn 落后:
  walreceiver 写入或 flush 跟不上。
flushed_lsn 前进，但 pg_last_wal_replay_lsn() 落后:
  startup process replay 层落后。
pg_stat_wal_receiver 无行:
  walreceiver 未运行或尚未 ready_to_display。
```
### 15.3 recovery 与 replay 侧
standby 上补充查询：
```sql
SELECT pg_last_wal_receive_lsn(),
       pg_last_wal_replay_lsn(),
       pg_last_xact_replay_timestamp(),
       now() - pg_last_xact_replay_timestamp() AS replay_time_age;
```
这些函数来自 recovery/replay 侧，不等价于 walreceiver 状态。
常见组合：
```text
pg_stat_wal_receiver.flushed_lsn 增长
  + pg_last_wal_replay_lsn() 不增长
    -> WAL 已到本地，replay 卡住或被 delay。
pg_stat_wal_receiver.latest_end_lsn 不增长
  + last_msg_receipt_time 刷新
    -> primary 没新 WAL，但 keepalive 仍正常。
pg_stat_wal_receiver 无行
  + 日志反复 "waiting for WAL to become available"
    -> startup process 在重试 WAL source。
```
### 15.4 wait event
常见 wait event：
| wait event | 位置 | 含义 |
| --- | --- | --- |
| `WalSenderMain` | `WalSndWait(... WAIT_EVENT_WAL_SENDER_MAIN)` | walsender 等 socket 可读/可写或 timeout。 |
| `WalReceiverMain` | `WaitLatchOrSocket(... WAIT_EVENT_WAL_RECEIVER_MAIN)` | walreceiver 等 primary socket、latch 或 timeout。 |
| `RecoveryWalStream` | `WaitForWALToBecomeAvailable()` | startup process 等 walreceiver 写入更多 WAL。 |
| `RecoveryRetrieveRetryInterval` | stream/archive/pg_wal 都暂时拿不到 WAL | startup process 在 retry interval 中等待。 |
| `WalReceiverExit` | `ShutdownWalRcv()` | startup process 等 walreceiver 退出。 |
| `WalReceiverWaitStart` | `WalRcvWaitForStartPosition()` | walreceiver 等 startup process 下发新 start position。 |
wait event 只说明“正在等什么”，不自动说明根因。
例如 `WalReceiverMain` 可以是正常空闲，也可以是快到 timeout，也可以是在等 socket 可读。
必须结合：
```text
last_msg_receipt_time
reply_time
LSN 差距
server log
GUC timeout 设置
```
### 15.5 日志文本
本节最常见日志锚点：
```text
primary:
  "terminating walsender process due to replication timeout"
  "unexpected EOF on standby connection"
standby:
  "terminating walreceiver due to timeout"
  "could not receive data from WAL stream: ..."
  "could not send data to WAL stream: ..."
  "replication terminated by primary server"
  "started streaming WAL from primary at ..."
  "restarted WAL streaming at ..."
  "walreceiver ended streaming and awaits new instructions"
startup process:
  "waiting for WAL to become available at ..."
  "WAL receiver process shutdown requested"
```
定位时按时间线排序，而不是只找最后一条 ERROR。
网络故障中，真正的第一条日志可能出现在 standby，也可能出现在 primary。
## 16. 常见误区
误区一：`wal_sender_timeout` 表示 standby 没有 replay。
不对。它表示 primary 在超时时间内没有收到 standby 的复制协议回复。reply 里可以带相同的 LSN。
误区二：`wal_receiver_timeout` 表示 primary 没有新 WAL。
不对。它表示 standby 在超时时间内没有收到 primary 的任何复制消息。keepalive 也算消息。
误区三：`reply_requested` 表示对方落后。
不对。它只是请求立即回复，用于 heartbeat。落后由 LSN 差距表达。
误区四：`pg_stat_replication.state = streaming` 表示 standby 完全追上。
不对。它说明 walsender 处于 streaming 状态。standby 可能仍有 write/flush/replay lag。
误区五：walreceiver 会自己重连。
不准确。walreceiver 失败会退出并标记 STOPPED。是否、何时、从哪里重新拉流，由 startup process 的 WAL source 状态机和 `RequestXLogStreaming()` 决定。
误区六：把 `reply_time` 当成本机收到回复的时间。
`reply_time` 是 standby 在 status update 中携带的时间。primary 本地“收到过 reply”的控制时钟是 `last_reply_timestamp`，不直接暴露在 SQL 视图中。
误区七：timeout 设为 0 只是“更宽松”。
设为 0 是禁用对应 timeout。primary 或 standby 将更多依赖 socket 错误、OS keepalive、业务 WAL 或其它协议消息发现断线。
## 17. 课堂实验
### 实验一：观察正常空闲
在没有业务写入时同时观察两侧：
```sql
-- primary
SELECT pid, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       reply_time, now() - reply_time AS reply_age
FROM pg_stat_replication;
-- standby
SELECT status, written_lsn, flushed_lsn, latest_end_lsn,
       last_msg_receipt_time, now() - last_msg_receipt_time AS receive_age
FROM pg_stat_wal_receiver;
```
```text
期望:
  LSN 可以不变。
  reply_time / last_msg_receipt_time 应周期性刷新。
源码:
  XLogWalRcvSendReply()
  ProcessRepliesIfAny()
  ProcessStandbyReplyMessage()
```
### 实验二：跟踪双向 request-reply
用较小 timeout 或断点观察 primary keepalive 与 standby ping：
```gdb
break WalSndKeepaliveIfNecessary
break WalSndKeepalive
break XLogWalRcvProcessMsg
break WalRcvComputeNextWakeup
break XLogWalRcvSendReply
break ProcessWalSndrMessage
break ProcessStandbyReplyMessage
```
```text
primary 半超时:
  WalSndKeepalive(true)
  standby XLogWalRcvSendReply(true, false, false)
standby 半超时:
  XLogWalRcvSendReply(true, true, false)
  primary WalSndKeepalive(false)
```
目标：确认 `reply_requested` 只请求立即响应，不代表 LSN 必须前进。
### 实验三：区分断线、落后和重连
制造 standby replay 慢，或短暂阻断 standby 与 primary 的网络。观察：
```sql
SELECT sent_lsn, write_lsn, flush_lsn, replay_lsn,
       now() - reply_time AS reply_age
FROM pg_stat_replication;
SELECT latest_end_lsn, written_lsn, flushed_lsn,
       now() - last_msg_receipt_time AS receive_age
FROM pg_stat_wal_receiver;
SELECT pg_last_wal_replay_lsn();
```
```text
落后:
  reply_age / receive_age 新鲜，sent_lsn - replay_lsn 增大。
断线:
  reply_age 或 receive_age 停止刷新，日志出现 walsender/walreceiver timeout。
重连:
  WalRcvDie()
  WaitForWALToBecomeAvailable()
  RequestXLogStreaming()
```
目标：确认 reconnect 由 startup process 触发，而不是 walreceiver 在自己的 main loop 中重连。
## 18. 源码检查清单：定位一次复制 timeout
定位时按这个顺序，不要一上来猜网络：
```text
1. primary 是否还看到 pg_stat_replication 行？
2. reply_time 是否新鲜？
3. standby 是否还看到 pg_stat_wal_receiver 行？
4. last_msg_receipt_time 是否新鲜？
5. sent/write/flush/replay 哪个差距最大？
6. standby 日志是否有 walreceiver timeout 或 libpq receive/send error？
7. primary 日志是否有 walsender timeout 或 EOF？
8. startup process 是否在 RecoveryRetrieveRetryInterval 或 RecoveryWalStream？
9. primary_conninfo / slot / timeline / archive 是否导致反复重启？
10. timeout GUC 是否过小或被禁用？
```
如果 `reply_time` 新鲜但 `replay_lsn` 落后，不要改 timeout，先查 recovery。
如果 `last_msg_receipt_time` 不刷新且快到 `wal_receiver_timeout`，先查 primary 到 standby 的消息方向。
如果 primary 报 walsender timeout，但 standby 没明显错误，先查 standby 到 primary 的回复方向。
如果两边都反复重连，还要查：
```text
primary_conninfo
认证和网络路由
replication slot 是否存在
请求 LSN 是否已被回收
timeline history 是否完整
archive / pg_wal 是否能提供缺口
```
## 19. 讨论题
1. 为什么 primary 的 timeout 依据“收到 standby reply”，而不是依据 standby 的 `apply_lsn` 是否前进？
2. 为什么 standby 的 timeout 依据“收到 primary message”，而不是依据是否收到 WALData payload？
3. `reply_requested` 如果双方都原样回 `true`，可能造成什么协议层问题？
4. `wal_receiver_status_interval = 0` 时，哪些 reply 仍然会发送，为什么？
5. 为什么 `pg_stat_replication.state = streaming` 不能证明 standby 没有 replay lag？
6. walreceiver timeout 后为什么不是在 walreceiver 内部 sleep 后重连？
7. 如果 `sent_lsn - write_lsn` 很大但 `reply_time` 新鲜，下一步应该查网络、standby 写盘还是 startup replay？
8. 为什么 `last_reply_timestamp` 不直接暴露给 SQL 视图，而 `reply_time` 也不能完全替代它？
## 20. 本节小结
本节把复制 keepalive 压缩成一个可迁移模型：
```text
协议活性和复制进度必须分开建模。
```
primary 侧：
```text
ProcessRepliesIfAny()
  -> 收到 standby 任意复制消息
  -> 更新 last_reply_timestamp
  -> 清 waiting_for_ping_response
WalSndKeepaliveIfNecessary()
  -> 半个 wal_sender_timeout 无回复
  -> keepalive(reply_requested = true)
WalSndCheckTimeOut()
  -> 整个 wal_sender_timeout 无回复
  -> 关闭 walsender
```
standby 侧：
```text
XLogWalRcvProcessMsg()
  -> 收到 WALData 或 Keepalive
  -> ProcessWalSndrMessage()
  -> 刷新 lastMsgReceiptTime
WalReceiverMain() timeout wakeup
  -> 半个 wal_receiver_timeout 无消息
  -> StandbyStatusUpdate(reply_requested = true)
WalReceiverMain() terminate wakeup
  -> 整个 wal_receiver_timeout 无消息
  -> ERROR 并退出 walreceiver
```
重连侧：
```text
WalRcvDie()
  -> 标记 WalRcv STOPPED
  -> WakeupRecovery()
WaitForWALToBecomeAvailable()
  -> stream source 失败
  -> archive / pg_wal / retry / stream 状态机
  -> RequestXLogStreaming()
```
诊断时的核心判断：
```text
正常空闲:
  LSN 不变，但 reply_time / last_msg_receipt_time 仍刷新。
网络中断:
  协议消息 timestamp 停止刷新，并最终触发 timeout 或 socket error。
standby 落后:
  协议消息仍刷新，但 sent/write/flush/replay 之间出现差距。
```
可迁移规律：
```text
分布式数据流不能只用“数据是否前进”判断健康；
必须把 heartbeat、进度位置、owner 状态机和重试策略拆成独立层次。
heartbeat 证明对端还能响应，
LSN 证明数据推进到了哪个边界，
reconnect 状态机决定失败后从哪里恢复。
```
