# PostgreSQL WAL Sender 主循环与发送位置推进
## 课程定位
前置知识：已经理解 WAL record 先进入 WAL buffer、再写出和 fsync 到 `pg_wal`，
也知道 physical streaming replication 使用 replication connection 和 CopyBoth
协议把 primary 上的 WAL 流传给 standby。
本节唯一主问题：
```text
walsender 如何在读取 WAL、等待新 WAL、发送 CopyData 和处理反馈之间循环，
为什么 sent、write、flush、apply 位置必须分开记录？
```
核心矛盾：primary 希望尽快把已经安全的 WAL 推给 standby，减少复制延迟；
但它不能发送 primary 自己崩溃后可能消失的 WAL，也不能把“已经发到 socket”、
“standby 写到文件”、“standby fsync”和“startup process 已回放”混成一个位置。
学完后应能判断：
```text
pg_stat_replication.sent_lsn 落后时，问题更可能在主库 WAL flush / walsender / socket；
write_lsn 落后时，问题可能在网络或 walreceiver 写入；
flush_lsn 落后时，问题可能在 standby WAL fsync；
replay_lsn 落后时，问题可能在 startup redo、冲突、recovery pause 或 apply 侧压力。
```
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
上一节讲 physical replication connection 如何进入复制协议：
```text
连接以 replication=true 进入 walsender
  -> 执行 IDENTIFY_SYSTEM / START_REPLICATION
  -> primary 校验 system identifier、timeline 和起始 LSN
  -> 进入 CopyBoth 流模式
```
本节从 `START_REPLICATION` 之后继续。
连接已经变成一个长期 WAL 流。
此时系统不再是普通 SQL request / response。
它变成一个持续循环：
```text
从本地 WAL 可发送边界取一段字节
  -> 封装为 CopyData
  -> 尝试非阻塞写给 standby
  -> 同时读取 standby 的状态反馈
  -> 如果没有新 WAL 或 socket 不可写，就等待 latch、socket 或 timeout
```
这个循环看起来像简单的 producer / consumer。
但它必须同时维护四种不同的 LSN 语义。
`sentPtr` 表示 primary 的 walsender 已经把 WAL 消息排入发送缓冲到哪里。
`write` 表示 standby walreceiver 已经把 WAL 写入本地 WAL segment 到哪里。
`flush` 表示 standby 已经把这些 WAL fsync 到可靠存储到哪里。
`apply` 表示 standby startup process 已经回放到哪里。
`pg_stat_replication` 里显示的 `replay_lsn` 对应源码里的 `WalSnd.apply`。
名字不同，是因为视图面向用户使用 replay 这个词。
源码内部沿同步复制 wait mode 使用 apply 这个词。
本节不展开 replication slot 的 WAL 保留边界。
也不展开 synchronous replication 的完整 commit wait 策略。
但会解释 walsender 为什么必须把反馈位置放进 shared memory，
使同步复制和监控视图能共享同一组进度事实。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
WalSndLoop() 每轮先处理 standby 反馈，再在没有待发送数据时调用 XLogSendPhysical()；
XLogSendPhysical() 只读取 primary 已 flush 的 WAL，构造 WALData CopyData 并推进 sentPtr；
standby 通过 StandbyStatusUpdate 回报 write/flush/apply；
walsender 把这些反馈写入 WalSnd shared state，供 pg_stat_replication 和 SyncRep 使用。
```
这里有两个不能偷懒合并的边界。
第一，primary 的发送边界不是当前插入 LSN。
physical walsender 在 primary 上调用 `GetFlushRecPtr(NULL)`。
它只发送已经写出并 fsync 的 WAL。
源码注释说得很直接：
```text
standbys must not have applied any WAL that got lost on the primary
```
如果 primary 把尚未持久化的 WAL 发送给 standby，
primary crash 后这些 WAL 可能从 primary 历史中消失。
standby 却可能已经回放它们。
这会制造不可接受的历史分叉。
第二，standby 的接收边界不是单个位置。
从网络收到一段 WAL，只说明字节进入 walreceiver。
写入 `pg_wal` 表示它已经进入本地文件。
fsync 表示 standby crash 后还能保留。
startup process 回放表示只读查询、promotion 语义和远端可见性又前进一步。
这三者经常相等。
但系统必须为它们不相等的时刻建模。
同步复制正是在这些差异上定义等待语义：
```text
remote_write:
  等 write_lsn
remote_flush / on:
  等 flush_lsn
remote_apply:
  等 replay_lsn / apply
```
如果只保存一个 standby 位置，
就无法同时支持低延迟写确认、崩溃安全确认和回放可见确认。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/replication/walsender.c` | `StartReplication()`、`WalSndLoop()`、`XLogSendPhysical()`、`ProcessRepliesIfAny()` 和 `ProcessStandbyReplyMessage()` 的主线。 |
| 2 | `src/include/replication/walsender_private.h` | `WalSnd`、`WalSndCtlData`、`WALSNDSTATE_*` 和同步复制队列状态。 |
| 3 | `src/include/replication/walsender.h` | walsender 对外入口、GUC、`WalSndWakeupRequest()` 和 `WalSndWakeupProcessRequests()`。 |
| 4 | `src/backend/access/transam/xlog.c` | WAL flush 后如何请求并执行 walsender wakeup，为什么唤醒要离开 WAL 写锁之后做。 |
| 5 | `src/backend/access/transam/xlogreader.c` | `WALRead()` 如何按字节从 WAL segment 读取，供 physical sender 填充输出消息。 |
| 6 | `src/backend/replication/walreceiver.c` | standby 如何处理 `WALData`、写入 WAL、flush，并构造 `StandbyStatusUpdate`。 |
| 7 | `src/backend/access/transam/xlogrecovery.c` | startup process 如何推进 replay LSN，walreceiver 如何通过 `GetXLogReplayRecPtr()` 取得 apply 位置。 |
| 8 | `src/backend/replication/syncrep.c` | `SyncRepReleaseWaiters()` 如何使用 write/flush/apply 释放等待提交的 backend。 |
| 9 | `src/backend/catalog/system_views.sql` | `pg_stat_replication` 如何映射 `pg_stat_get_wal_senders()` 的结果。 |
阅读时不要先从复制协议消息格式背起。
更好的顺序是沿着一个位置推进：
```text
START_REPLICATION 起始 LSN
  -> sentPtr 初始化
  -> XLogSendPhysical() 选择可发送边界
  -> CopyData 进入 libpq output buffer
  -> standby 写入 / flush / replay
  -> ProcessStandbyReplyMessage() 更新 WalSnd
  -> pg_stat_replication 和 SyncRep 读取这些字段
```
这条线把“发送”和“确认”分开。
walsender 不是一个只写 socket 的进程。
它是 primary 上复制进度的 owner。
## 4. 关键数据结构与状态
### 4.1 backend-local 的 `sentPtr`
`walsender.c` 中有一个 static 变量：
```text
static XLogRecPtr sentPtr = InvalidXLogRecPtr;
```
源码注释强调它的真实语义：
```text
How far have we sent WAL already?
Actually, this is the next WAL location to send.
```
所以 `sentPtr` 不是“最后一个已经发送的字节地址”。
它是下一次应该从哪里开始发送。
如果 `sentPtr = 0/5000000`，
表示 `< 0/5000000` 的范围已经被 walsender 放进发送路径。
`StartReplication()` 在进入 CopyBoth 模式后把它设置为客户端请求的 startpoint。
随后每次 `XLogSendPhysical()` 成功构造一条 `WALData` 消息，
就把它推进到本次消息的 `endptr`。
这个变量是当前 walsender backend-local 状态。
其它 backend 不能直接读它。
因此每次推进后还要复制到 shared state。
### 4.2 shared memory 中的 `WalSnd`
`src/include/replication/walsender_private.h` 定义了每个 walsender 一个 `WalSnd`。
它放在 `WalSndCtlData.walsnds[]` 中。
关键字段是：
```text
pid
state
sentPtr
needreload
write
flush
apply
writeLag / flushLag / applyLag
sync_standby_priority
mutex
replyTime
kind
```
这个结构的注释很重要。
大多数字段由 `mutex` spinlock 保护。
但部分字段只由 walsender 本进程写，
本进程可以不持锁读取自己负责的值。
其它读取者，比如 `pg_stat_get_wal_senders()` 和 `syncrep.c`，
必须持锁取快照。
`sentPtr` 和 `write/flush/apply` 的来源不同。
`sentPtr` 由 primary 本地 `XLogSendPhysical()` 推进。
`write/flush/apply` 由 standby 的反馈消息推进。
这就是为什么它们必须同时放在 `WalSnd` 里。
一个表示 primary 已经把数据交给发送路径。
三个表示 standby 已经把同一条流推进到不同本地阶段。
### 4.3 `WalSndCtlData`
`WalSndCtlData` 是整个实例的 walsender shared control block。
它包含：
```text
SyncRepQueue[NUM_SYNC_REP_WAIT_MODE]
lsn[NUM_SYNC_REP_WAIT_MODE]
sync_standbys_status
wal_flush_cv
wal_replay_cv
wal_confirm_rcv_cv
walsnds[]
```
前三项属于同步复制等待队列。
`wal_flush_cv` 和 `wal_replay_cv` 是唤醒 walsender 的 condition variable。
physical walsender 等待 WAL 被 flush。
logical walsender 在 standby 上还可能等待 WAL 被 replay。
`wal_confirm_rcv_cv` 用于 failover slot 相关的 standby confirmation。
本节关注 physical streaming，
但要记住 `WalSndLoop()` 是共享框架。
同一个循环也用于 logical sender，
只是 `send_data` callback 不同。
### 4.4 `output_message` 与 `reply_message`
walsender 在 Copy 模式中同时处理两个方向的协议消息。
发送方向使用 `output_message`。
接收方向使用 `reply_message`。
物理 WALData 消息被编码成：
```text
CopyData
  -> PqReplMsg_WALData ('w')
  -> dataStart
  -> walEnd
  -> sendtime
  -> WAL payload bytes
```
standby 状态反馈被编码成：
```text
CopyData
  -> PqReplMsg_StandbyStatusUpdate ('r')
  -> writePtr
  -> flushPtr
  -> applyPtr
  -> replyTime
  -> replyRequested
```
这两个消息方向在同一个循环里交替推进。
因此 walsender 主循环不能只盯着“有没有 WAL 可发”。
它还必须在每轮开始处理来自 standby 的输入。
否则同步复制 waiters、slot restart_lsn、timeout 判断和统计视图都会滞后。
### 4.5 `WalSndCaughtUp`
`WalSndCaughtUp` 是一个 backend-local bool。
它表示当前循环认为没有更多 WAL 可以发送。
它不是持久事实。
只要 primary 又 flush 了新 WAL，
walsender 会被唤醒并重新计算可发送边界。
`XLogSendPhysical()` 在这些情况下设置它：
```text
没有可发送 WAL:
  true
本次发送到当前 flush 边界:
  true
本次只发送 MAX_SEND_SIZE 的一部分:
  false
libpq output buffer 还有未 flush 数据:
  WalSndLoop() 设为 false
```
这个状态的用途是决定 loop 是否需要等待。
它不是用户可见的复制进度。
用户看到的是 `WalSnd.sentPtr` 和 standby 反馈位置。
### 4.6 `last_reply_timestamp` 与 `waiting_for_ping_response`
`last_reply_timestamp` 记录最近一次 `ProcessRepliesIfAny()` 看到 standby 消息的时间。
`waiting_for_ping_response` 表示 walsender 已经发出需要对端响应的 keepalive。
它们共同驱动 `wal_sender_timeout`。
timeout 不是按“没有 WAL 可发”判断。
一个完全空闲的复制连接也必须定期互相确认活性。
因此主循环在没有 WAL 时仍会醒来发送 keepalive。
如果超过 `wal_sender_timeout` 没有任何 standby reply，
walsender 认为复制连接不可用并退出。
### 4.7 lag tracker
`LagTracker` 是 `walsender.c` 中的 backend-local 时间样本环。
`XLogSendPhysical()` 在观察到当前可发送边界 `SendRqstPtr` 时调用：
```text
LagTrackerWrite(SendRqstPtr, GetCurrentTimestamp())
```
随后 `ProcessStandbyReplyMessage()` 用 standby 回报的 write/flush/apply LSN
分别调用：
```text
LagTrackerRead(SYNC_REP_WAIT_WRITE, writePtr, now)
LagTrackerRead(SYNC_REP_WAIT_FLUSH, flushPtr, now)
LagTrackerRead(SYNC_REP_WAIT_APPLY, applyPtr, now)
```
这不是精确的因果计时。
源码注释说它是 primary 本地 WAL flush 时间的近似采样。
但它足够支撑 `pg_stat_replication.write_lag`、`flush_lag`、`replay_lag`。
诊断时要把这些 lag 视为“从 primary 采样时间到 standby 回报该 LSN 的 elapsed time”。
不要把它们解释成纯网络延迟或纯 replay 耗时。
## 5. 主流程源码 walkthrough：从 `START_REPLICATION` 到 CopyBoth
物理复制的入口在 `StartReplication()`。
它先创建一个 `XLogReaderState`：
```text
XLogReaderAllocate(wal_segment_size,
                   NULL,
                   XL_ROUTINE(.segment_open = WalSndSegmentOpen,
                              .segment_close = wal_segment_close),
                   NULL)
```
这里容易误解。
physical streaming 不需要把 WAL record 解码成逻辑事件。
它只是需要一个 WAL reader state 来管理 segment open / close、timeline 和读取上下文。
真正搬运的是 WAL 字节。
`StartReplication()` 随后处理 replication slot、timeline 和起始位置。
如果起始点超过当前 server 的 flush LSN，
它会报错：
```text
requested starting point ... is ahead of the WAL flush position
```
这正是 primary 不能发送未 flush WAL 的边界。
通过检查后，walsender 发送 `CopyBothResponse`。
这表示连接进入双向 COPY 流。
之后 server 可以连续发 `CopyData`。
standby 也可以连续回 `CopyData` 反馈。
进入循环前，`StartReplication()` 做三件关键状态初始化：
```text
WalSndSetState(WALSNDSTATE_CATCHUP)
sentPtr = cmd->startpoint
MyWalSnd->sentPtr = sentPtr
SyncRepInitConfig()
```
`WALSNDSTATE_CATCHUP` 表示 standby 初始还没追上 upstream。
等 `WalSndLoop()` 发现没有更多可发 WAL 且 output buffer 已空，
状态会切到 `WALSNDSTATE_STREAMING`。
这个状态不是装饰。
同步复制会忽略非 streaming / stopping 的 sender。
监控也靠它区分初始追赶和正常流式复制。
## 6. `WalSndLoop()` 的主循环
`WalSndLoop()` 接收一个 callback：
```text
static void WalSndLoop(WalSndSendDataCallback send_data)
```
physical streaming 传入的是 `XLogSendPhysical`。
logical streaming 传入的是 `XLogSendLogical`。
本节主要看 physical，
但主循环的调度纪律是共享的。
循环开始时初始化：
```text
last_reply_timestamp = GetCurrentTimestamp()
waiting_for_ping_response = false
```
这意味着一旦进入 streaming，
replication timeout 就开始生效。
每一轮的时间线是：
```text
ResetLatch(MyLatch)
CHECK_FOR_INTERRUPTS()
WalSndHandleConfigReload()
ProcessRepliesIfAny()
检查 CopyDone 双向结束
如果 output buffer 没有 pending，就调用 send_data()
否则不生成新数据，并认为还没 caught up
pq_flush_if_writable()
如果 caught up 且无 pending，必要时切 streaming 或处理 shutdown
WalSndCheckTimeOut()
WalSndCheckShutdownTimeout()
WalSndKeepaliveIfNecessary()
必要时 WalSndWait()
```
这个顺序有两个重要含义。
第一，反馈优先于发送。
`ProcessRepliesIfAny()` 在 `send_data()` 之前执行。
如果 standby 刚刚回报了更高的 flush/apply，
同步复制 waiters 可以尽快被释放。
slot 的 confirmed location 也能尽快推进。
第二，发送和 socket flush 分离。
`XLogSendPhysical()` 只是把 WALData 放进 libpq output buffer。
`pq_flush_if_writable()` 才尝试把 buffer 写到 socket。
如果 socket 当前不可写，walsender 不会继续无限构造新 CopyData。
`WalSndLoop()` 看到 `pq_is_send_pending()` 为 true，
就跳过下一次 `send_data()` 并等待 socket writable。
这避免单个慢 standby 把 walsender 本地 output buffer 越堆越大。
`WalSndLoop()` 不是 while true busy loop。
它只在两类情况下睡眠：
```text
physical sender 已 caught up 且没有 CopyDone 待结束
或者 libpq output buffer 还有 pending 数据
```
等待事件由 `WalSndWait()` 执行。
它同时等待：
```text
socket readable:
  standby 反馈或 CopyDone
socket writable:
  pending output buffer 可继续写
latch / condition variable:
  新 WAL flush、replay 推进、shutdown、reload 等
timeout:
  keepalive 和 wal_sender_timeout
```
所以 walsender 的主循环是一条 cooperative event loop。
它没有一个独立线程专门读反馈。
也没有一个独立线程专门写 WAL。
同一个循环必须保证输入、输出、等待和 timeout 都不饿死。
## 7. `XLogSendPhysical()` 如何选择可发送边界
`XLogSendPhysical()` 的注释给出函数职责：
```text
Read up to MAX_SEND_SIZE bytes of WAL that's been flushed to disk,
but not yet sent to the client.
```
第一步是确定 `SendRqstPtr`。
这个位置是本轮最多可以发送到哪里。
如果发送的是 historic timeline，
边界是 timeline fork point：
```text
SendRqstPtr = sendTimeLineValidUpto
```
如果当前 server 是 cascading standby，
它调用：
```text
SendRqstPtr = GetStandbyFlushRecPtr(&SendRqstTLI)
```
意思是只能转发本 standby 已经接收并 flush 的 WAL。
它还要处理 promotion 或 timeline switch。
一旦发现当前发送 timeline 变成 historic，
就读 timeline history，
计算 `sendTimeLineValidUpto`。
如果当前 server 是 primary，
它调用：
```text
SendRqstPtr = GetFlushRecPtr(NULL)
```
这里的边界是本节最重要的不变量：
```text
physical walsender 不能发送 primary 自己尚未 flush 的 WAL。
```
然后 `XLogSendPhysical()` 写入 lag tracker。
采样的 LSN 是 `SendRqstPtr`。
这意味着 lag 采样点可能比本条消息实际 payload 的 endptr 更远。
源码接受这个近似。
目标是低成本估算“某个本地 flush 位置被 standby 回报时过去了多久”，
而不是为每条 WALData 保存精确时间戳。
## 8. `XLogSendPhysical()` 如何切片和读取 WAL
如果 `SendRqstPtr <= sentPtr`，
没有工作可做。
函数设置：
```text
WalSndCaughtUp = true
```
然后返回主循环。
否则本轮从 `sentPtr` 开始发送。
`MAX_SEND_SIZE` 定义为：
```text
XLOG_BLCKSZ * 16
```
函数先尝试发送最多这么多字节。
如果超过 `SendRqstPtr`，回退到 `SendRqstPtr`。
如果仍然有更多 WAL 可发，则把 `endptr` 向下对齐到 WAL page boundary。
这个对齐不是纯性能优化。
源码注释说明 walreceiver 依赖 sender 不把一个 WAL record 拆到两个消息里。
长 WAL record 如果跨页，会按 continuation record 的方式在 page boundary 继续。
所以 page boundary 是安全的发送切分点。
消息构造顺序是：
```text
resetStringInfo(&output_message)
pq_sendbyte(PqReplMsg_WALData)
pq_sendint64(dataStart = startptr)
pq_sendint64(walEnd = SendRqstPtr)
pq_sendint64(sendtime = 0)
enlargeStringInfo(nbytes)
读取 WAL payload 到 output_message.data
最后补写 sendtime
pq_putmessage_noblock(PqMsg_CopyData, ...)
```
读 WAL 时先尝试：
```text
WALReadFromBuffers(...)
```
如果 WAL bytes 还在 WAL buffers 中，
可以避免读文件。
剩余部分再调用：
```text
WALRead(xlogreader, ..., WalSndSegmentOpen, ...)
```
`WALRead()` 来自 `xlogreader.c`。
它按 segment 打开文件，
用 `pg_pread()` 读取 requested bytes，
并报告 `WAIT_EVENT_WAL_READ` 和 `pg_stat_io` WAL read 统计。
如果读取失败，
它把错误信息填进 `WALReadError`，
调用者用 `WALReadRaiseError()` 报错。
读取后 `XLogSendPhysical()` 调用：
```text
CheckXLogRemoved(segno, xlogreader->seg.ws_tli)
```
这是防止读到已经被移除或回收的 WAL。
在 cascading standby 中，
当前打开的 WAL segment 还可能被 archive 中同名文件替换。
所以函数检查 `MyWalSnd->needreload`。
如果需要 reload，
关闭当前 segment 并 `goto retry`。
这段逻辑保留了 physical replication 的真实 awkwardness：
```text
walsender 不是只读 primary 本地稳定文件；
在 standby 上做 cascading 时，它面对的是 recovery、archive、streaming 和 timeline 共同推进的文件系统状态。
```
成功构造消息后，函数推进：
```text
sentPtr = endptr
MyWalSnd->sentPtr = sentPtr
```
共享状态更新在 `walsnd->mutex` 下完成。
这就是 `pg_stat_replication.sent_lsn` 的来源。
注意此时并不表示 standby 已收到。
它只表示 walsender 已经把这段 WAL 放进 CopyData 输出路径。
如果 socket buffer 堵住，
`sent_lsn` 可以领先于 standby 的 `write_lsn`。
## 9. 等待新 WAL 如何发生
physical `XLogSendPhysical()` 本身通常不阻塞等待新 WAL。
它读取当前 `GetFlushRecPtr()` 或 cascading flush 边界。
如果没有新 WAL，
它把 `WalSndCaughtUp` 设为 true 并返回。
真正等待发生在 `WalSndLoop()`。
当 caught up 且没有待发送 output，
主循环进入：
```text
WalSndWait(wakeEvents, sleeptime, WAIT_EVENT_WAL_SENDER_MAIN)
```
`WalSndWait()` 不只等 socket。
它先根据当前 walsender 类型准备睡在 condition variable 上：
```text
physical:
  WalSndCtl->wal_flush_cv
logical:
  WalSndCtl->wal_replay_cv
WAIT_FOR_STANDBY_CONFIRMATION:
  WalSndCtl->wal_confirm_rcv_cv
```
然后仍然用 `WaitEventSetWait()` 等 FeBe socket、latch 和 timeout。
这种做法的关键是：
```text
ConditionVariablePrepareToSleep() 只是把进程加入 CV waitlist；
真正等待仍由 WaitEventSetWait() 完成；
ConditionVariableBroadcast() 最终 SetLatch() 唤醒等待者。
```
为什么需要 CV，而不是遍历所有 walsender slot 发 latch？
因为 WAL flush 路径非常热。
`xlog.c` 在写出 / flush WAL 时只调用：
```text
WalSndWakeupRequest()
```
等释放 WAL 写锁、离开 critical path 后再执行：
```text
WalSndWakeupProcessRequests(true, !RecoveryInProgress())
```
`WalSndWakeup()` 对 `wal_flush_cv` 或 `wal_replay_cv` 做 broadcast。
这样 WAL 写路径不需要逐个拿每个 `WalSnd` 的 spinlock。
它只表达“有新进度，等待者自己醒来重新检查 predicate”。
standby 上 cascading 场景也有类似唤醒。
`walreceiver.c` 在 `XLogWalRcvFlush()` 推进 `LogstreamResult.Flush` 后：
```text
WakeupRecovery()
if (AllowCascadeReplication())
    WalSndWakeup(true, false)
```
startup process 在 `xlogrecovery.c` 回放 WAL 后，
也会在允许 cascading 时唤醒 logical 或 physical sender。
这说明等待新 WAL 不是一个单点 sleep。
它是“状态推进者负责唤醒，walsender 醒来重新计算可发送边界”的 predicate wait 模型。
## 10. `ProcessRepliesIfAny()` 如何处理反馈
walsender 每轮先调用 `ProcessRepliesIfAny()`。
这个函数以非阻塞方式读取 standby 发来的消息。
如果当前已经收到 standby 的 `CopyDone`，
后续消息属于下一条命令，
它会停止读取。
否则它循环调用：
```text
pq_startmsgread()
pq_getbyte_if_available(&firstchar)
pq_getmessage(&reply_message, maxmsglen)
```
允许的外层消息包括：
```text
PqMsg_CopyData
PqMsg_CopyDone
PqMsg_Terminate
```
`CopyData` 内部再交给 `ProcessStandbyMessage()`。
它根据第一字节区分：
```text
PqReplMsg_StandbyStatusUpdate
PqReplMsg_HotStandbyFeedback
PqReplMsg_PrimaryStatusRequest
```
本节关注普通 status update。
真正解析发生在 `ProcessStandbyReplyMessage()`：
```text
writePtr = pq_getmsgint64(&reply_message)
flushPtr = pq_getmsgint64(&reply_message)
applyPtr = pq_getmsgint64(&reply_message)
replyTime = pq_getmsgint64(&reply_message)
replyRequested = pq_getmsgbyte(&reply_message)
```
如果 standby 请求 reply，
walsender 立即发送 keepalive。
随后它计算三种 lag。
然后在 `MyWalSnd->mutex` 下更新：
```text
walsnd->write = writePtr
walsnd->flush = flushPtr
walsnd->apply = applyPtr
walsnd->writeLag = writeLag
walsnd->flushLag = flushLag
walsnd->applyLag = applyLag
walsnd->replyTime = replyTime
```
如果当前不是 cascading walsender，
它调用：
```text
SyncRepReleaseWaiters()
```
这一步把 standby 的反馈接到 primary 上等待 commit 的 backend。
最后，如果当前 walsender 持有 replication slot，
并且 `flushPtr` 有效，
它还会推进 slot 的接收确认：
```text
logical slot:
  LogicalConfirmReceivedLocation(flushPtr)
physical slot:
  PhysicalConfirmReceivedLocation(flushPtr)
```
对 physical slot 来说，
`PhysicalConfirmReceivedLocation()` 会把 slot 的 `restart_lsn` 更新到 flushPtr，
然后重新计算 required LSN。
这说明 `flush` 位置不仅是监控指标。
它还影响 primary 对 WAL 文件保留的判断。
standby 只写了但没有 flush 的 WAL，
不能作为 slot 安全释放旧 WAL 的确认。
## 11. standby 端为什么能给出三种位置
位置差异来自 standby 内部的三段生命周期。
walreceiver 收到 `PqReplMsg_WALData` 后，
`XLogWalRcvProcessMsg()` 解析：
```text
dataStart
walEnd
sendTime
payload
```
然后调用：
```text
XLogWalRcvWrite(buf, len, dataStart, tli)
```
`XLogWalRcvWrite()` 把 payload 写入本地 WAL segment。
每次 `pg_pwrite()` 成功后推进：
```text
LogstreamResult.Write = recptr
```
写完后它把 `WalRcv->writtenUpto` 发布到 shared memory。
这就是 standby 端 write 位置的来源。
随后 `XLogWalRcvFlush()` 在需要时执行：
```text
issue_xlog_fsync(recvFile, recvSegNo, tli)
LogstreamResult.Flush = LogstreamResult.Write
walrcv->flushedUpto = LogstreamResult.Flush
WakeupRecovery()
```
这就是 flush 位置的来源。
最后 apply 位置不由 walreceiver 推进。
它来自 startup process 的 recovery 状态。
`xlogrecovery.c` 中 `GetXLogReplayRecPtr()` 在 `XLogRecoveryCtl->info_lck` 下读取：
```text
lastReplayedEndRecPtr
lastReplayedTLI
```
walreceiver 在构造 status update 时调用它：
```text
applyPtr = GetXLogReplayRecPtr(NULL)
```
所以一条 standby status update 同时跨了两个进程：
```text
walreceiver:
  write / flush
startup process:
  apply / replay
```
这也是为什么 `XLogWalRcvSendReply()` 有 `checkApply` 参数。
检查 apply 位置需要拿 recovery shared state 的 spinlock。
如果当前调用只是由写入或 flush 触发，
可以只比较 write/flush 是否推进，
避免每次都读 apply。
当 startup process 明确请求 apply feedback 时，
walreceiver 才用 `checkApply = true` 发送更及时的 replay 位置。
## 12. 为什么 sent/write/flush/apply 不能合并
可以把这四个位置看成一条 pipeline。
```text
primary WAL flush
  -> walsender sent
  -> network
  -> standby walreceiver write
  -> standby fsync flush
  -> startup process apply
```
每个箭头都可能停住。
因此每个 LSN 都回答不同问题。
`sent_lsn` 回答：
```text
primary walsender 已经尝试发送到哪里？
```
它由 primary 本地推进。
它不证明 standby 收到。
`write_lsn` 回答：
```text
standby walreceiver 已经把 WAL 写入本地 WAL 文件到哪里？
```
它可以满足 `remote_write` 风格的低延迟确认。
但 standby crash 后，尚未 fsync 的部分可能丢失。
`flush_lsn` 回答：
```text
standby 已经把 WAL 持久化到哪里？
```
它是 physical slot 安全确认和 `remote_flush` 语义的关键。
它不证明 standby 的查询已经能看到这些变化。
`apply` / `replay_lsn` 回答：
```text
standby startup process 已经回放到哪里？
```
它用于 `remote_apply`，
也用于判断 standby 只读查询和 promotion 后可见历史是否追上。
如果把它们合成一个 `confirmed_lsn`，
系统会立刻失去三种能力。
第一，无法用 `remote_write` 换取更低 commit latency。
第二，无法用 `remote_flush` 表示 standby crash-safe 边界。
第三，无法用 `remote_apply` 等待 standby 回放可见性。
更糟的是，诊断会失去分层。
看到一个位置落后，
你无法判断问题在主库发送、网络、standby 写盘、fsync 还是 replay。
PostgreSQL 选择把这些位置分开，
不是为了显示更多列。
它是在源码状态上承认复制链路是多阶段 pipeline。
## 13. 同步复制如何连接这些位置
`syncrep.c` 中 `SyncRepReleaseWaiters()` 是连接点。
当 `ProcessStandbyReplyMessage()` 更新了 `MyWalSnd->write/flush/apply` 后，
primary 上的 walsender 会尝试释放同步复制等待者。
函数先过滤掉不合格 sender：
```text
sync_standby_priority == 0:
  不是 potential sync standby
state 不是 STREAMING / STOPPING:
  不参与释放
flush 无效:
  没有有效 standby 确认
```
然后在 `SyncRepLock` 下调用：
```text
SyncRepGetSyncRecPtr(&writePtr, &flushPtr, &applyPtr, &am_sync)
```
这一步根据 `synchronous_standby_names` 的 priority 或 quorum 策略，
从候选 standby 中计算当前可确认的 write/flush/apply 位置。
接着它分别推进三条等待队列的 release LSN：
```text
walsndctl->lsn[SYNC_REP_WAIT_WRITE] = writePtr
walsndctl->lsn[SYNC_REP_WAIT_FLUSH] = flushPtr
walsndctl->lsn[SYNC_REP_WAIT_APPLY] = applyPtr
```
并调用 `SyncRepWakeQueue()` 释放对应 wait mode 的 backend。
这解释了为什么 walsender 不能只把反馈保存在 backend-local 变量里。
同步复制等待者不是 standby 连接自己。
它们是 primary 上正在 commit 的普通 backend。
它们需要从 shared `WalSndCtlData` 看到每个同步 standby 的状态。
也解释了为什么 `write/flush/apply` 要在 `WalSnd` 中持锁更新。
它们既是 walsender 的输入消息结果，
也是其它 backend 的同步条件。
## 14. 统计视图如何连接这些位置
用户看到的 `pg_stat_replication` 定义在 `system_views.sql`。
它连接：
```text
pg_stat_get_activity(NULL) AS S
JOIN pg_stat_get_wal_senders() AS W ON S.pid = W.pid
```
`pg_stat_get_wal_senders()` 在 `walsender.c` 中遍历 `WalSndCtl->walsnds[]`。
对每个 active slot，
它在 `walsnd->mutex` 下复制：
```text
pid
sentPtr
state
write
flush
apply
writeLag
flushLag
applyLag
sync_standby_priority
replyTime
```
然后输出成视图列：
```text
sent_lsn   <- sentPtr
write_lsn  <- write
flush_lsn  <- flush
replay_lsn <- apply
```
这里有两个诊断边界。
第一，视图是瞬时快照。
walsender 更新这些字段后，
standby 可能马上又推进了。
视图不会表达字段之间的原子全局因果。
它只是在每个 `WalSnd` spinlock 下取到一组当前值。
第二，权限会隐藏细节。
非 superuser 且没有 `pg_read_all_stats` 权限的用户，
只能看到 pid，
其它列会是 NULL。
所以线上诊断时先确认权限。
不要把 NULL 误解成 standby 没有反馈。
lag 列也要小心。
`write_lag`、`flush_lag`、`replay_lag` 不是“当前 backlog 除以吞吐”的估算。
它们是 lag tracker 根据曾经采样的 primary flush time 和 standby 回报 LSN 算出的 elapsed time。
当 standby 已经追上且位置在连续两次 reply 中不变，
`ProcessStandbyReplyMessage()` 会清空 stale lag。
这就是空闲系统里 lag 可能变 NULL 的原因。
## 15. 生命周期、ownership 与 cleanup
`WalSndCtlData` 是 shared memory 对象。
它在 postmaster 启动期通过 `WalSndShmemRequest()` 请求空间，
通过 `WalSndShmemInit()` 初始化。
大小取决于：
```text
offsetof(WalSndCtlData, walsnds)
  + max_wal_senders * sizeof(WalSnd)
```
单个 walsender 进程启动时调用 `InitWalSenderSlot()`。
它遍历 `WalSndCtl->walsnds[]`，
找一个 `pid == 0` 的 slot，
在 spinlock 下写入：
```text
pid = MyProcPid
state = WALSNDSTATE_STARTUP
sentPtr = InvalidXLogRecPtr
write/flush/apply = InvalidXLogRecPtr
lag = -1
sync_standby_priority = 0
replyTime = 0
kind = physical 或 logical
```
然后把 `MyWalSnd` 指向这个 slot。
进程退出时通过：
```text
on_shmem_exit(WalSndKill, 0)
```
把 `pid` 重新置 0。
这表示 slot 可被后续 walsender 复用。
注意 shared `WalSnd` slot 本身不释放。
释放的是“这个 slot 当前被哪个 pid 占用”的语义。
物理流期间还可能持有 replication slot。
`WalSndErrorCleanup()` 会在 ERROR 路径释放：
```text
LWLockReleaseAll()
ConditionVariableCancelSleep()
pgstat_report_wait_end()
关闭 xlogreader 当前 WAL segment
ReplicationSlotRelease()
ReplicationSlotCleanup(false)
ReleaseAuxProcessResources(false)
WalSndSetState(WALSNDSTATE_STARTUP)
```
walsender 不是普通事务 backend。
因此它需要一个类似事务 abort 的专用 cleanup。
这也是源码课程必须关注错误路径的原因。
只看 happy path 会漏掉 WAL segment fd、replication slot 和 CV sleep 状态的收尾。
## 16. 超时、断线与 Copy 结束边界
`ProcessRepliesIfAny()` 处理三类结束或异常。
如果读取 socket 返回 EOF 或 unexpected error，
它报 `COMMERROR` 并 `proc_exit(0)`。
如果收到非法 standby message type，
它报 `FATAL` 或 `COMMERROR`。
如果收到 `PqMsg_Terminate`，
它直接退出。
如果收到 `PqMsg_CopyDone`，
walsender 在尚未发送 CopyDone 时回一个 CopyDone，
并设置：
```text
streamingDoneReceiving = true
streamingDoneSending = true 或等待本端完成
```
主循环只有在两端 CopyDone 都完成且 output buffer 为空时，
才真正 break。
这避免一端已经请求结束时丢掉本端已排队的结束消息。
timeout 由 `WalSndCheckTimeOut()` 处理。
它用 `last_reply_timestamp + wal_sender_timeout` 计算 deadline。
判断时用的是 `last_processing`。
源码注释解释了原因：
```text
Using last_processing as the reference point avoids counting server-side stalls against the client.
```
也就是说，
walsender 自己因为 server-side stall 很久没有处理循环，
不应该马上把这段时间全算作客户端无响应。
但这个策略也有边界。
如果 server-side stall 太长，
keepalive 可能已经晚于预期很久。
standby 仍然需要很快回复，
否则会被认为 timeout。
shutdown 边界由 `WalSndCheckShutdownTimeout()`、`WalSndDone()` 和
`WalSndDoneImmediate()` 管理。
正常 shutdown 希望发送剩余 WAL，
等待 standby 复制到 `sentPtr`，
再优雅退出。
`WalSndDone()` 判断：
```text
replicatedPtr = flush 有效 ? flush : write
WalSndCaughtUp && sentPtr == replicatedPtr && output buffer empty
```
如果满足，就发送 Copy 完成消息并退出。
如果不满足，
它会发 keepalive 请求对端回复。
如果 `wal_sender_shutdown_timeout` 到期，
`WalSndDoneImmediate()` 会尽量非阻塞 flush 已有输出，
然后警告退出。
这条路径明确承认：
```text
强制 shutdown 可能发生在所有 WAL 都复制到 receiver 之前。
```
## 17. 成本、资源与跨模块传播
walsender hot path 的成本不只来自网络。
第一类成本是 WAL 读取。
如果 WAL 还在 WAL buffers，
`WALReadFromBuffers()` 可以减少文件 I/O。
否则 `WALRead()` 会读 WAL segment，
形成 `WAIT_EVENT_WAL_READ` 和 `pg_stat_io` WAL read。
落后很久的 standby 更容易迫使 walsender 从文件读取旧 WAL。
第二类成本是 socket 背压。
`pq_putmessage_noblock()` 只把消息放进 output buffer。
真正发送依赖 `pq_flush_if_writable()`。
如果 standby 或网络慢，
`pq_is_send_pending()` 会持续为 true。
主循环会停止生成更多 WALData，
并等待 `WL_SOCKET_WRITEABLE`。
第三类成本是 feedback 频率。
standby 的 `wal_receiver_status_interval` 控制周期性反馈。
反馈太慢会让 `pg_stat_replication`、同步复制释放和 slot 确认滞后。
反馈太频繁会增加消息、唤醒和 spinlock 成本。
第四类成本是同步复制的 fan-in。
每次有效反馈都可能触发 `SyncRepReleaseWaiters()`。
它需要拿 `SyncRepLock`，
扫描候选 synchronous standby，
并唤醒等待队列。
在高 commit 并发和多个同步 standby 下，
这个路径会把 standby feedback 的节奏传播到 primary commit latency。
第五类成本是 recovery apply。
`replay_lsn` 落后可能来自 startup process 正在处理大量 WAL、
等待 I/O、被 hot standby conflict 限制，
或者 recovery pause。
这部分成本不在 walsender 内部。
但它通过 apply feedback 反向影响 `remote_apply` 等待。
## 18. 诊断：从四个位置差值反推卡点
诊断 physical replication lag 时，
先把四个位置按 pipeline 排好：
```text
sent_lsn >= write_lsn >= flush_lsn >= replay_lsn
```
实际视图瞬时快照中可能因为时序和权限出现 NULL 或短暂不一致感。
但长期判断按这条 pipeline 做。
### 18.1 `sent_lsn` 落后 primary 当前 flush LSN
如果 primary 当前 `pg_current_wal_flush_lsn()` 明显领先 `sent_lsn`，
问题在 walsender 还没有把可发送 WAL 推出去。
常见原因：
```text
walsender 正在读旧 WAL 文件
socket output buffer pending，无法继续 send_data()
walsender 被调度延迟
timeline / cascading reload / WAL removed 检查导致重试或错误
```
回源码看：
```text
WalSndLoop()
  -> pq_is_send_pending()
  -> XLogSendPhysical()
  -> WALReadFromBuffers() / WALRead()
  -> pq_putmessage_noblock()
```
可以结合 `pg_stat_activity.wait_event`。
如果是 `WalSenderMain`，
可能是在等 WAL、socket 或 timeout。
如果看到 WAL read wait，
说明 sender 正在从 WAL segment 读取。
### 18.2 `sent_lsn - write_lsn` 大
这表示 primary 已经把 WAL 交给发送路径，
但 standby 还没有回报写到本地文件。
常见解释：
```text
网络带宽或延迟
standby walreceiver 没有及时读取 socket
standby 写 WAL 文件慢
standby walreceiver 已写但反馈尚未发送或尚未被 primary 处理
```
回源码看：
```text
primary:
  ProcessRepliesIfAny()
  ProcessStandbyReplyMessage()
standby:
  XLogWalRcvProcessMsg()
  XLogWalRcvWrite()
  XLogWalRcvSendReply()
```
这个差值不能单独证明网络慢。
因为 write_lsn 是通过反馈消息回来的。
反馈周期、walreceiver 写盘和 primary 处理反馈都会影响它。
### 18.3 `write_lsn - flush_lsn` 大
这表示 standby 已写 WAL 文件，
但尚未 fsync 到 flush 位置。
常见解释：
```text
standby WAL fsync 慢
存储队列拥塞
walreceiver 正在批量 flush
standby 处于 I/O 压力
```
回源码看：
```text
XLogWalRcvWrite()
  -> LogstreamResult.Write
XLogWalRcvFlush()
  -> issue_xlog_fsync()
  -> LogstreamResult.Flush
  -> XLogWalRcvSendReply()
```
如果同步复制等待的是 `remote_flush`，
这个差值会直接影响 primary commit。
如果等待的是 `remote_write`，
它可能不影响 commit，
但影响 standby crash-safe 程度和 slot 确认。
### 18.4 `flush_lsn - replay_lsn` 大
这表示 standby 已经安全收到 WAL，
但 startup process 没有回放到同样位置。
常见解释：
```text
redo 本身慢
standby 存储读取数据页慢
hot standby 查询冲突拖慢或阻塞回放
recovery pause
promotion / timeline 切换边界
CPU 或 checkpoint / restartpoint 压力
```
回源码看：
```text
walreceiver:
  applyPtr = GetXLogReplayRecPtr(NULL)
startup:
  XLogRecoveryCtl->lastReplayedEndRecPtr
```
如果业务要求 standby 查询尽快看到 primary commit，
这个差值比 `sent_lsn - flush_lsn` 更重要。
如果业务只要求灾备节点 crash 后能保留 WAL，
`flush_lsn` 已经足够。
这就是为什么诊断必须先问同步语义。
### 18.5 lag 时间与 LSN 差值一起看
LSN 差值表示 backlog 字节。
lag 时间表示某些采样点被 standby 回报时花了多久。
二者不是同一个维度。
可能出现：
```text
LSN 差值很小，但 lag 时间大:
  WAL 生成量低，少量 WAL 长时间未被回报。
LSN 差值很大，但 lag 时间暂时 NULL:
  系统刚启动、采样不足、空闲后 lag 被清空，或权限隐藏。
```
源码里的 `clearLagTimes` 逻辑会在 standby 完全追上且两次 reply 位置不变时清空 lag。
所以空闲系统里 NULL lag 常常是正常现象。
不要把 NULL lag 解读为没有复制延迟。
要结合 LSN 位置判断。
## 19. 常见误区
误区一：`sent_lsn` 表示 standby 已收到。
不是。
它表示 primary walsender 已经发送到哪里。
它不跨网络证明。
误区二：`write_lsn` 和 `flush_lsn` 通常相等，所以可以合并。
不能。
它们在 I/O 压力、batch flush、standby crash-safety 语义和 sync rep wait mode 上不同。
误区三：`replay_lsn` 落后就是网络慢。
不是。
`replay_lsn` 落后说明 startup process apply 落后。
网络慢通常先表现为 `sent_lsn - write_lsn`。
误区四：walsender 一直在阻塞读 WAL。
physical sender 通常先计算当前 flush 边界。
没有新 WAL 时返回主循环等待唤醒。
socket pending 时也会停止构造新 WALData。
误区五：`wal_sender_timeout` 只在有 WAL 发送时生效。
不是。
空闲连接也依赖 keepalive 和 standby status update 维持活性。
误区六：`write_lag`、`flush_lag`、`replay_lag` 是三段耗时分解。
不是严格分解。
它们都是从 primary 本地采样时间到 standby 回报对应位置的 elapsed time。
## 20. 课堂实验
### 实验一：跟读一轮 physical send
在 primary 上对源码下断点：
```text
break StartReplication
break WalSndLoop
break XLogSendPhysical
break ProcessStandbyReplyMessage
```
观察变量：
```text
p sentPtr
p SendRqstPtr
p WalSndCaughtUp
p MyWalSnd->sentPtr
p MyWalSnd->write
p MyWalSnd->flush
p MyWalSnd->apply
```
目标是画出：
```text
cmd->startpoint
  -> sentPtr 初始化
  -> endptr
  -> shared sentPtr
  -> standby reply 更新 write/flush/apply
```
不要把第一次进入 `WalSndLoop()` 时的 `write/flush/apply` 无效值解释成异常。
standby 还没发第一条 status update 时，
这些字段本来就是 invalid。
### 实验二：观察 socket 背压
构造一个低带宽或高延迟 standby，
或者临时限制 standby 端接收速度。
在 primary 观察：
```sql
SELECT pid, state, sent_lsn, write_lsn, flush_lsn, replay_lsn,
       write_lag, flush_lag, replay_lag, sync_state, reply_time
FROM pg_stat_replication;
```
同时观察 walsender wait event。
如果 `sent_lsn` 领先但 `write_lsn` 不动，
回到源码看：
```text
pq_is_send_pending()
pq_flush_if_writable()
ProcessRepliesIfAny()
```
目标是区分“primary 还没发送”和“standby 还没确认写入”。
### 实验三：制造 replay 落后
在 standby 上制造长查询或暂停 recovery。
观察：
```sql
SELECT pg_last_wal_replay_lsn();
```
在 primary 上观察 `pg_stat_replication.replay_lsn`。
预期现象：
```text
write_lsn / flush_lsn 继续推进
replay_lsn 停住或慢很多
```
源码解释：
```text
walreceiver:
  XLogWalRcvFlush() 已经推进 flush
startup:
  GetXLogReplayRecPtr() 没有推进
walsender:
  ProcessStandbyReplyMessage() 只能记录 standby 回报的 applyPtr
```
目标是理解 replay lag 不属于 walsender 发送性能问题。
### 实验四：验证 sync rep wait mode
分别设置不同 `synchronous_commit` 语义，
并使用 synchronous standby。
观察 primary commit 等待与这些位置的关系：
```text
remote_write:
  主要等 write_lsn
remote_flush / on:
  主要等 flush_lsn
remote_apply:
  主要等 replay_lsn
```
回源码验证：
```text
ProcessStandbyReplyMessage()
  -> SyncRepReleaseWaiters()
     -> walsndctl->lsn[SYNC_REP_WAIT_WRITE]
     -> walsndctl->lsn[SYNC_REP_WAIT_FLUSH]
     -> walsndctl->lsn[SYNC_REP_WAIT_APPLY]
```
目标是把 SQL 侧 commit latency 和 walsender shared state 连起来。
## 21. 讨论题
1. 为什么 physical walsender 在 primary 上不能使用 WAL insert end LSN 作为发送边界？
2. 如果 `sent_lsn` 快速推进但 `write_lsn` 不动，你会按什么顺序排查？
3. 为什么 physical slot 用 standby `flushPtr` 确认接收，而不是 `writePtr`？
4. `remote_apply` 为什么不能只等 walreceiver flush？
5. `WalSndLoop()` 为什么每轮先处理 reply，再尝试发送更多 WAL？
6. `pg_stat_replication` 的 lag 列为什么会在空闲追上后变成 NULL？
7. 如果要降低反馈滞后，调小 `wal_receiver_status_interval` 会带来哪些成本？
8. cascading standby 上的 walsender 为什么要特别处理 timeline switch 和 WAL segment reload？
## 22. 本节小结
walsender 的主循环不是“读 WAL，然后 write socket”。
它是一条持续调度链：
```text
处理 standby 输入
  -> 按 primary / cascading / historic timeline 计算可发送边界
  -> 只发送已经安全 flush 的 physical WAL bytes
  -> 非阻塞写 CopyData
  -> 在 socket、latch、condition variable 和 timeout 之间等待
  -> 把 standby write/flush/apply 反馈发布到 shared state
```
核心状态分成两类。
`sentPtr` 是 walsender 本地发送进度，
并同步到 `MyWalSnd->sentPtr` 供观察。
`write/flush/apply` 是 standby 反馈进度，
由 `ProcessStandbyReplyMessage()` 写入 `WalSnd`。
`pg_stat_replication` 读取这些字段。
`SyncRepReleaseWaiters()` 也读取这些字段。
因此这四个位置既是监控指标，
也是 correctness 和 commit wait 的运行时状态。
异常路径同样服务这条主线。
`wal_sender_timeout` 防止无反馈连接无限存在。
CopyDone 双向状态防止流结束时丢消息。
shutdown timeout 承认优雅追完和实例停机之间存在取舍。
`WalSndErrorCleanup()` 负责释放 WAL segment、replication slot、CV sleep 和资源 owner。
可迁移规律：
```text
复制系统里的一个“进度”通常不是单点事实；
只要数据跨越发送、接收、持久化和应用多个阶段，
就应该把每个阶段的 owner、可见性和失败语义分开记录。
```
下一节进入 standby 侧，
看 `walreceiver` 如何把 primary 发来的 WALData 写成本地 segment，
并通过 shared state、latch 和 status update 让 startup process 与 primary 都看到进度。
