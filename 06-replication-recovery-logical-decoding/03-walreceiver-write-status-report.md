# PostgreSQL WAL Receiver 接收、写入与状态发布
## 课程定位
前置知识：已经理解 WAL segment、timeline、standby recovery、physical replication protocol、latch、spinlock 和基本 shared memory 访问边界。
本节唯一主问题：
```text
walreceiver 如何把主库发来的 WAL 流写成本地 segment，
并通过 shared state、latch 和状态上报让 startup process 与主库看到接收进度？
```
核心矛盾：standby 想要尽快接收主库 WAL，让 recovery 能继续 replay，也让主库同步复制和复制监控看到进度。
但接收、写入、fsync、replay、反馈是不同时间点。
如果把它们合成一个“standby 已收到”位置，会同时破坏 crash safety、同步复制语义和诊断。
PostgreSQL 的选择是拆成几层状态：
```text
network message:
  walreceiver 刚从 libpq CopyData 读到的字节。
writtenUpto:
  walreceiver 已经 pwrite 到本地 WAL segment 的位置。
flushedUpto:
  walreceiver 已经 fsync 并可供 startup process 当作安全 WAL 来源的位置。
replay position:
  startup process 已经 redo 到的位置。
status reply:
  standby 回给主库的 write / flush / apply 三个位置。
```
学完后应能判断：
```text
为什么 walreceiver 写 WAL 后还要单独 flush；
为什么 startup process 主要看 flushedUpto 而不是 writtenUpto；
为什么 written_lsn 和 flushed_lsn 在 pg_stat_wal_receiver 中可能不一致；
为什么 SetLatch 只是唤醒 startup process 重新检查状态；
为什么 last_msg_send_time / last_msg_receipt_time 只能近似解释传输延迟；
为什么 primary 断流后 walreceiver 进入 waiting 或退出都不是数据库 crash；
为什么可观测 replication lag 必须按 sent/write/flush/replay 分层。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前两节围绕 WAL sender / receiver 的连接和发送侧循环。
这一节站到 standby 侧。
我们只追一条链：
```text
startup process 需要某个 LSN
  -> 发现 archive / pg_wal 里还没有
  -> 请求 walreceiver 从主库拉 WAL
  -> walreceiver 收 CopyData
  -> 写入本地 pg_wal segment
  -> fsync 后发布 flushedUpto
  -> SetLatch 唤醒 startup process
  -> 发送 standby status update 给主库
```
本节不展开 walsender 如何读主库 WAL，也不展开 REDO record 如何修改数据页。
这些模块只作为边界出现。
本节关心的对象是接收进度。
这个进度同时被三类观察者使用：
```text
startup process:
  它要知道哪些 WAL 已经能读来 redo。
primary walsender:
  它要知道 standby 已经 write / flush / apply 到哪里。
运维和诊断入口:
  pg_stat_wal_receiver、pg_stat_replication、wait event、日志和 gdb。
```
一个常见误判是看到 standby 上 `pg_stat_wal_receiver.written_lsn` 追上了主库，就认为 recovery 也追上了。
这不成立。
`written_lsn` 只表示 walreceiver 进程把字节写进了本地 segment。
startup process 真正能稳定消费的边界是 flush 后发布的 `flushed_lsn`，而用户可见只读查询追上的边界又是 replay position。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
walreceiver 是 standby 上的 auxiliary process；
它从 libpqwalreceiver 读取主库 CopyData 消息，
把 WALData payload 按 LSN 写入 pg_wal segment，
fsync 后在 WalRcv shared state 中推进 flushedUpto，
用 WakeupRecovery() 唤醒 startup process，
再用 StandbyStatusUpdate 把 write / flush / apply LSN 回报给 primary。
```
这里的 tension 是：
```text
尽快暴露接收进度
  vs
不能把未 fsync、未 replay、或者只是网络层已到达的字节伪装成更强语义
```
如果只追求低延迟，可以在收到 CopyData 后立刻告诉 startup process “WAL 到了”。
问题是这时 WAL 可能还没有写入 segment。
如果 pwrite 成功后就告诉 recovery “可以用了”，也仍有问题。
startup process 可能随后读取该 segment，而进程或系统崩溃后这些字节未必持久。
因此 PostgreSQL 把进度拆成多个位置：
```text
LogstreamResult.Write:
  walreceiver backend-local，表示本进程已经写到哪里。
WalRcv->writtenUpto:
  shared atomic，给诊断和 WaitLSN 的 standby_write 等待使用。
LogstreamResult.Flush:
  walreceiver backend-local，表示本进程已经 fsync 到哪里。
WalRcv->flushedUpto:
  shared spinlock-protected，startup process 用它判断能否从 stream 来源读 WAL。
GetXLogReplayRecPtr():
  startup process redo 进度，status reply 中的 apply 位置来自这里。
```
`SetLatch()` 只负责打断睡眠。
真正的消息在 shared state 和 WAL segment 文件中。
startup process 被唤醒后必须重新读 `WalRcv->flushedUpto`，不能把 latch 本身解释成“某个 LSN 已经达到”。
主库看到的也是同样的拆分。
standby status reply 发给 walsender 的不是一个 LSN，而是：
```text
write LSN
flush LSN
apply LSN
reply timestamp
replyRequested flag
```
主库上的同步复制、slot restart_lsn 推进和 lag 统计都依赖这组分层位置。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/walreceiver.h` | `WalRcvData` shared state、`WalRcvState`、walreceiver 回调接口和查询入口。 |
| 2 | `src/backend/replication/walreceiverfuncs.c` | startup process 如何请求 streaming、停止 walreceiver、读取 write / flush 位置。 |
| 3 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()` 主循环、消息解析、写 segment、flush、reply、退出 cleanup。 |
| 4 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | libpq 实现如何把 replication COPY 流抽象成 `walrcv_receive()`。 |
| 5 | `src/backend/access/transam/xlogrecovery.c` | startup process 的 `WaitForWALToBecomeAvailable()` 如何启动 walreceiver 并等待 `flushedUpto`。 |
| 6 | `src/backend/access/transam/xlog.c` | `XLogFileInit()`、`issue_xlog_fsync()`、`XLogShutdownWalRcv()` 等 WAL 文件和 shutdown 边界。 |
| 7 | `src/backend/access/transam/xlogwait.c` | standby write / flush LSN 等待如何读 walreceiver 进度并通过 latch 唤醒。 |
| 8 | `src/backend/replication/walsender.c` | 主库如何处理 standby status update，更新 write / flush / apply 和 lag。 |
| 9 | `src/backend/catalog/system_views.sql` | `pg_stat_wal_receiver` 如何把 shared state 暴露成 SQL 视图。 |
推荐阅读顺序：
```text
先读 walreceiver.h 的 WalRcvData；
再读 walreceiverfuncs.c 的 RequestXLogStreaming()；
进入 walreceiver.c 的 WalReceiverMain()；
沿 XLogWalRcvProcessMsg() -> XLogWalRcvWrite() -> XLogWalRcvFlush() 跟一条 WALData；
回到 xlogrecovery.c 看 startup process 如何等待 flushedUpto；
最后读 XLogWalRcvSendReply() 和 walsender.c 的 ProcessStandbyReplyMessage()。
```
不要从 libpq 连接参数开始读。
连接只是入口。
本课的核心是状态推进：
```text
收到消息
  -> 写本地 segment
  -> flush
  -> 发布 shared state
  -> 唤醒 recovery
  -> 反馈 primary
```
也不要把 `pg_stat_wal_receiver` 当成源码入口。
这个视图只是把 `WalRcv` 中一部分字段和 `writtenUpto` 读出来。
它不能告诉你 walreceiver 当前卡在网络、pwrite、fsync、startup replay，还是主库没有更多 WAL。
## 4. 关键数据结构与状态
### 4.1 `WalRcvData`: startup process 与 walreceiver 的 shared contract
`WalRcvData` 位于 main shared memory。
初始化在 `WalRcvShmemInit()`：
```text
walRcvState = WALRCV_STOPPED
ConditionVariableInit(walRcvStoppedCV)
SpinLockInit(mutex)
writtenUpto = 0
procno = INVALID_PROC_NUMBER
```
这个结构不是“walreceiver 的所有状态”。
它只保存其它进程需要观察或修改的状态。
核心字段可以分成几组：
| 字段 | 更新者 | 语义 |
| --- | --- | --- |
| `procno` / `pid` | walreceiver | startup process 用来发 SIGTERM 或 SetLatch。 |
| `walRcvState` | startup process 与 walreceiver | STOPPED、STARTING、CONNECTING、STREAMING、WAITING、RESTARTING、STOPPING。 |
| `receiveStart` / `receiveStartTLI` | startup process | 请求 walreceiver 从哪个 LSN 和 timeline 开始。 |
| `flushedUpto` / `receivedTLI` | walreceiver | 已经接收并 fsync 的 WAL 边界。 |
| `latestChunkStart` | walreceiver | 最近一次 flush 前的旧 `flushedUpto`。 |
| `writtenUpto` | walreceiver | 已 pwrite 但不保证 fsync 的位置。 |
| `lastMsgSendTime` / `lastMsgReceiptTime` | walreceiver | 最近收到的主库消息发送时间和本地接收时间。 |
| `latestWalEnd` / `latestWalEndTime` | walreceiver | 主库在消息里报告的 end-of-WAL。 |
| `conninfo` / `sender_host` / `sender_port` / `slotname` | startup process 与 walreceiver | 连接和可观测信息。 |
| `apply_reply_requested` | startup process | 请求 walreceiver 尽快发送 apply 反馈。 |
除 `writtenUpto` 和 `apply_reply_requested` 外，大部分字段在 `WalRcv->mutex` spinlock 下访问。
`writtenUpto` 用 `pg_atomic_uint64`。
原因是它会在 hot path 的每次写 WAL 后推进，读者又主要用于诊断和 standby write 等待。
源码注释明确说：
```text
writtenUpto 类似 flushedUpto，
但它在 write 后、flush 前推进；
其它进程可以读到这个位置之前的数据，
但不应用它做 data integrity 判断。
```
所以字段本身不是语义。
`writtenUpto + 未 fsync` 的组合语义是“写入 OS 文件接口成功”。
`flushedUpto + receivedTLI + spinlock` 的组合语义才是 startup process 可用的 streaming WAL 边界。
### 4.2 `WalRcvState`: walreceiver 不是只有 running / stopped
`WalRcvState` 的状态集合：
```text
WALRCV_STOPPED
WALRCV_STARTING
WALRCV_CONNECTING
WALRCV_STREAMING
WALRCV_WAITING
WALRCV_RESTARTING
WALRCV_STOPPING
```
这些状态服务两个问题：
```text
startup process 是否需要请求 postmaster 启动新进程？
已有 walreceiver 是否只是等待新 startpoint，可以复用同一连接？
```
典型流转：
```text
STOPPED
  -> RequestXLogStreaming() 设置 STARTING
  -> postmaster 启动 WalReceiverMain()
  -> WalReceiverMain() 设置 CONNECTING
  -> START_REPLICATION 成功后设置 STREAMING
  -> primary 结束当前 timeline 的 COPY 后设置 WAITING
  -> startup process 给新起点，设置 RESTARTING
  -> walreceiver 醒来，回到 CONNECTING / STREAMING
  -> shutdown 时进入 STOPPING
  -> WalRcvDie() 设置 STOPPED
```
`WAITING` 很容易被误解成异常。
它只是 primary 对当前请求的 timeline/起点没有更多 WAL，或者 timeline 需要重新扫描时的等待模式。
startup process 会把这类情况当作断流一样处理：
```text
重新扫描 archive / pg_wal；
必要时请求新的 streaming 起点；
如果 walreceiver 还在 WAITING，就 SetLatch 让它重启；
如果进程已停，则发 PMSIGNAL_START_WALRECEIVER。
```
### 4.3 walreceiver backend-local 写入状态
`walreceiver.c` 还有一组 static 状态，不在 shared memory：
| 字段 | 语义 |
| --- | --- |
| `recvFile` | 当前打开的本地 WAL segment fd，`-1` 表示未打开。 |
| `recvFileTLI` | 当前 fd 对应的 timeline。 |
| `recvSegNo` | 当前 fd 对应的 segment number。 |
| `LogstreamResult.Write` | 本进程已经写到的 last byte + 1。 |
| `LogstreamResult.Flush` | 本进程已经 fsync 到的 last byte + 1。 |
| `reply_message` | 复用的输出缓冲，构造 status reply 和 hot standby feedback。 |
| `wakeup[]` | timeout、ping、status reply、HS feedback 的下一次唤醒时间。 |
这些状态只有 walreceiver 自己能直接访问。
它们与 `WalRcvData` 的关系是：
```text
backend-local state:
  描述当前进程实际写哪个 fd、写到哪里、flush 到哪里。
shared state:
  只发布其它进程需要看到的边界。
```
这也是 `LogstreamResult.Write` 和 `WalRcv->writtenUpto` 同时存在的原因。
前者是本地事实。
后者是共享发布。
### 4.4 libpqwalreceiver 的连接状态
`WalReceiverConn` 是 libpqwalreceiver 模块内部的 opaque 连接对象。
server-facing 代码只通过 `WalReceiverFunctionsType` 间接调用：
```text
walrcv_connect()
walrcv_identify_system()
walrcv_startstreaming()
walrcv_receive()
walrcv_send()
walrcv_endstreaming()
walrcv_disconnect()
```
这层 indirection 有一个现实原因：
```text
walreceiver.c 属于 backend；
libpq-specific 逻辑放在 libpqwalreceiver 模块中动态加载，
避免 server 主体直接链接 libpq。
```
对本节来说，最关键的回调是 `walrcv_receive()`。
它的语义很窄：
```text
返回 > 0:
  已收到一个 CopyData message，buffer 指向消息内容。
返回 0:
  当前没有完整消息，wait_fd 是可等待 socket。
返回 -1:
  对端结束 COPY streaming。
ERROR:
  网络、协议或 libpq 状态异常。
```
它不解析 WALData header。
它不写本地 WAL。
它只是把 replication COPY 流交给 walreceiver 主循环。
## 5. 主流程源码 walkthrough
### 5.1 startup process 请求开始 streaming
入口在 `xlogrecovery.c` 的 `WaitForWALToBecomeAvailable()`。
startup process 正在按 LSN 顺序读取 WAL。
当 archive 和 `pg_wal` 都不能满足请求，并且处于 standby mode，它会切到 `XLOG_FROM_STREAM`。
启动接收前会设置一个边界：
```text
SetInstallXLogFileSegmentActive()
RequestXLogStreaming(tli, ptr, PrimaryConnInfo, PrimarySlotName, wal_receiver_create_temp_slot)
```
`RequestXLogStreaming()` 做几件重要事情。
第一，它把请求位置向下对齐到 segment 起点：
```text
if recptr 不在 segment 起点:
  recptr -= XLogSegmentOffset(recptr)
```
原因是避免 streaming 写出一个前半段没有 record 的破损 segment。
第二，它在 `WalRcv->mutex` 下设置共享请求：
```text
slotname / is_temp_slot
walRcvState = STARTING 或 RESTARTING
startTime = now
flushedUpto / receivedTLI / latestChunkStart
writtenUpto
receiveStart / receiveStartTLI
```
第三，它决定如何唤醒执行者：
```text
如果 walreceiver 原来 STOPPED:
  SendPostmasterSignal(PMSIGNAL_START_WALRECEIVER)
如果 walreceiver 原来 WAITING:
  设置 WALRCV_RESTARTING
  SetLatch(walreceiver procLatch)
```
这一步说明 startup process 并不直接创建 walreceiver。
它修改 shared state，然后通知 postmaster 或已有 walreceiver。
### 5.2 WalReceiverMain 早期发布自身状态
`WalReceiverMain()` 启动后，第一件关键事不是连接主库。
它先把自己登记到 `WalRcv`：
```text
检查 walRcvState 必须是 STARTING；
pid = MyProcPid；
walRcvState = CONNECTING；
复制 conninfo / slotname / startpoint / startpointTLI；
初始化 lastMsgSendTime / lastMsgReceiptTime / latestWalEndTime；
procno = MyProcNumber；
```
这段代码在 `WalRcv->mutex` 下完成。
为什么要这么早？
如果后续连接失败或动态加载失败，退出回调 `WalRcvDie()` 仍能把状态设成 `STOPPED`。
否则 startup process 可能一直以为 walreceiver 还在 STARTING，直到启动超时。
接着它注册：
```text
on_shmem_exit(WalRcvDie, &startpointTLI)
```
这条 cleanup 路径非常重要。
正常断线、ERROR、SIGTERM 触发 FATAL，最终都会在 exit 时：
```text
XLogWalRcvFlush(true, tli)
walRcvState = STOPPED
pid = 0
procno = INVALID_PROC_NUMBER
ready_to_display = false
ConditionVariableBroadcast(walRcvStoppedCV)
walrcv_disconnect()
WakeupRecovery()
```
`dying = true` 让 flush 尽量完成接收 WAL 的 fsync 和 shared state 发布，但避免再发送 reply。
退出路径不是附录。
它决定 startup process 在断线后能及时重试，而不是睡在旧状态上。
### 5.3 连接 primary 并进入 START_REPLICATION
`WalReceiverMain()` 动态加载 `libpqwalreceiver` 后，调用：
```text
walrcv_connect(conninfo, true, false, false, appname, &err)
```
成功后，它用可展示版本的 conninfo 覆盖 shared state 中的连接信息，并填入 sender host / port。
然后主循环会先运行：
```text
walrcv_identify_system()
```
它校验主库 system identifier 与 standby 一致，并取得 primary 当前 timeline。
如果 primary timeline 落后于请求 timeline，直接 ERROR。
如果有缺失 timeline history file，会调用 `WalRcvFetchTimeLineHistoryFiles()` 从 primary 拉取并写入 `pg_wal`。
随后构造 physical stream options：
```text
options.logical = false
options.startpoint = startpoint
options.slotname = slotname 或 NULL
options.proto.physical.startpointTLI = startpointTLI
```
libpq 层的 `libpqrcv_startstreaming()` 会发送类似：
```text
START_REPLICATION [SLOT "..."] <LSN> TIMELINE <tli>
```
如果返回 COPY BOTH，walreceiver 切换成 STREAMING。
如果 primary 正常执行命令但没有进入 COPY 模式，表示该请求 timeline / 起点上没有更多 WAL，后面会进入 waiting。
### 5.4 STREAMING 状态初始化
进入 streaming 后：
```text
walRcvState 从 CONNECTING 切到 STREAMING；
LogstreamResult.Write = GetXLogReplayRecPtr(NULL)；
LogstreamResult.Flush = GetXLogReplayRecPtr(NULL)；
初始化 wakeup[]；
发送 initial reply；
发送 initial hot standby feedback；
```
这里的 `LogstreamResult` 从 replay position 初始化，而不是从 receiveStart 原样复制。
它表达的是 walreceiver 进程当前本地写/flush 进度的起点。
真正的请求起点已经在 shared state 中由 startup process 设置。
然后进入内部收流循环。
循环每次做几类事情：
```text
处理信号和 SIGHUP；
调用 walrcv_receive() 尝试立即读取消息；
连续处理所有不阻塞的消息；
写完后发送一次 status reply；
如果写入了 WAL，则 flush 并唤醒 recovery；
没有消息时计算最近的 timeout / reply / feedback 唤醒时间；
WaitLatchOrSocket() 等 socket、latch 或 timeout。
```
这说明 walreceiver 是事件循环，不是单纯阻塞在网络读。
它要同时处理：
```text
socket readable；
startup process 请求 apply reply；
wal_receiver_timeout；
wal_receiver_status_interval；
hot_standby_feedback；
postmaster death；
SIGTERM / SIGHUP。
```
### 5.5 libpqwalreceiver receive: 网络层只交付 CopyData
`libpqrcv_receive()` 的核心路径：
```text
PQgetCopyData(..., async = 1)
  -> rawlen > 0:
       返回一条 CopyData message
  -> rawlen == 0:
       PQconsumeInput()
       再试 PQgetCopyData()
       如果仍没有，返回 0 并给出 PQsocket()
  -> rawlen == -1:
       读取后续 PGresult
       PGRES_COMMAND_OK 或 PGRES_COPY_IN 表示 COPY 结束，返回 -1
  -> 其它错误:
       ereport(ERROR)
```
它每次会释放旧 `recvBuf`。
所以 `*buffer` 只在下一次 libpq receiver 调用前有效。
walreceiver 主循环收到 `len > 0` 后，调用：
```text
XLogWalRcvProcessMsg(buf[0], &buf[1], len - 1, startpointTLI)
```
第一个字节是 replication message type。
物理流里本节关心两类消息：
```text
PqReplMsg_WALData ('w')
PqReplMsg_Keepalive ('k')
```
`walrcv_receive()` 返回 0 时，walreceiver 不忙等。
它把 socket fd 交给：
```text
WaitLatchOrSocket(MyLatch,
                  WL_EXIT_ON_PM_DEATH | WL_SOCKET_READABLE |
                  WL_TIMEOUT | WL_LATCH_SET,
                  wait_fd,
                  nap,
                  WAIT_EVENT_WAL_RECEIVER_MAIN)
```
所以网络空闲、timeout、startup process 唤醒和 postmaster death 在同一个 wait point 上统一处理。
### 5.6 解析 WALData 和 Keepalive
`XLogWalRcvProcessMsg()` 对 WALData 的 header 期望是三个 int64：
```text
dataStart
walEnd
sendTime
```
解析后先调用：
```text
ProcessWalSndrMessage(walEnd, sendTime)
```
它在 `WalRcv->mutex` 下更新：
```text
if latestWalEnd < walEnd:
  latestWalEndTime = sendTime
latestWalEnd = walEnd
lastMsgSendTime = sendTime
lastMsgReceiptTime = GetCurrentTimestamp()
```
注意 `lastMsgSendTime` 和 `lastMsgReceiptTime` 每条消息都会更新。
它们表示“最近一条主库消息”。
`latestWalEndTime` 只有当主库报告的 WAL end 前进时才更新。
这两个时间组合可用于估计 transfer latency，但有边界：
```text
sendTime 来自主库时钟；
receiptTime 来自 standby 时钟；
两端时钟偏差会直接污染毫秒差值。
```
WALData header 后的 payload 才是真正要写入本地 segment 的 WAL 字节。
函数继续调用：
```text
XLogWalRcvWrite(payload, len, dataStart, tli)
```
Keepalive 消息也会调用 `ProcessWalSndrMessage()`，但不会写 WAL。
如果 keepalive 的 `replyRequested` 为 true，walreceiver 立即：
```text
XLogWalRcvSendReply(true, false, false)
```
这个 reply 是心跳响应，不代表 WAL payload 一定前进。
### 5.7 XLogWalRcvWrite: 按 LSN 写入本地 segment
`XLogWalRcvWrite()` 是本节最核心的写入函数。
它的循环以 `recptr` 和 `nbytes` 为主轴：
```text
while nbytes > 0:
  如果当前 recvFile 已经不包含 recptr:
    XLogWalRcvClose(recptr, tli)
  如果没有打开文件:
    XLByteToSeg(recptr, recvSegNo)
    recvFile = XLogFileInit(recvSegNo, tli)
    recvFileTLI = tli
  startoff = XLogSegmentOffset(recptr)
  segbytes = min(nbytes, wal_segment_size - startoff)
  pg_pwrite(recvFile, buf, segbytes, startoff)
  recptr += byteswritten
  LogstreamResult.Write = recptr
```
这段代码有几个边界。
第一，写入按 `dataStart` 指定的 LSN 定位。
walreceiver 不是把网络字节简单 append 到某个文件尾。
它用 LSN 算出 segment number 和 offset。
第二，一条 WALData message 可以跨 segment。
循环每次只写当前 segment 的剩余空间。
写到 segment 边界时，会关闭当前文件，再打开下一个 segment。
第三，文件由 `XLogFileInit()` 创建或打开。
`XLogFileInit()` 走 `xlog.c` 的 WAL segment 初始化逻辑：
```text
目标 segment 已存在:
  直接 open。
目标 segment 不存在:
  创建临时文件；
  zero-fill 或写末尾字节；
  fsync 初始化结果；
  install 到 pg_wal 的正式 segment 名。
```
这让 walreceiver 复用 PostgreSQL WAL segment 的标准创建路径，而不是自己随便创建普通文件。
第四，写失败是 PANIC。
源码在 `pg_pwrite()` 返回 `<= 0` 时，如果 errno 没设就当作 ENOSPC，然后：
```text
PANIC: could not write to WAL segment ...
```
对 standby 来说，不能继续在一个可能缺 WAL 字节的本地 pg_wal 上恢复。
### 5.8 writtenUpto: 写入进度的 shared 发布
`XLogWalRcvWrite()` 完成本次 buffer 写入后发布：
```text
pg_atomic_write_membarrier_u64(&WalRcv->writtenUpto, LogstreamResult.Write)
WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_WRITE, LogstreamResult.Write)
```
这里有两层语义。
`writtenUpto` 让其它 backend 可以看到 standby write LSN。
`WaitLSNWakeup()` 唤醒正在等待 standby write LSN 达到某个目标的进程。
它不是 startup process redo 的主要路径。
startup process 在 `WaitForWALToBecomeAvailable()` 中看的是 `GetWalRcvFlushRecPtr()`。
`writtenUpto` 适合这类场景：
```text
SQL 或内部等待只要求 standby 已经写到某个 LSN；
诊断想看网络接收和本地 pwrite 是否已追上；
pg_stat_wal_receiver.written_lsn 展示接收写入进度。
```
但它不能替代 `flushedUpto`。
因为 `pg_pwrite()` 成功不等于 `fsync()` 完成。
### 5.9 segment 完成后的 close 和归档通知
写完后如果当前 segment 已经完整：
```text
if recvFile >= 0 && !XLByteInSeg(recptr, recvSegNo, wal_segment_size):
  XLogWalRcvClose(recptr, tli)
```
`XLogWalRcvClose()` 先调用：
```text
XLogWalRcvFlush(false, tli)
```
然后关闭 fd。
关闭后根据归档模式创建 archive status：
```text
if XLogArchiveMode != ARCHIVE_MODE_ALWAYS:
  XLogArchiveForceDone(xlogfname)
else:
  XLogArchiveNotify(xlogfname)
```
这段逻辑的含义是：
```text
普通 standby 不应把从主库流来的 segment 再次归档；
所以多数情况下直接标记 .done。
archive_mode = always 时，standby 也承担归档职责；
因此创建 .ready 通知 archiver。
```
这个 close 动作不是持久化边界的唯一来源。
`XLogWalRcvFlush()` 也会在每批收到 WAL 后调用。
close 只是 segment 完成时尽快收尾，避免归档状态拖到下一个 segment 才处理。
### 5.10 XLogWalRcvFlush: 从写入事实到 recovery 可用事实
每次处理完一批可立即读取的消息后，主循环调用：
```text
XLogWalRcvFlush(false, startpointTLI)
```
它只在：
```text
LogstreamResult.Flush < LogstreamResult.Write
```
时真正工作。
主路径：
```text
issue_xlog_fsync(recvFile, recvSegNo, tli)
LogstreamResult.Flush = LogstreamResult.Write
SpinLockAcquire(&WalRcv->mutex)
if WalRcv->flushedUpto < LogstreamResult.Flush:
  latestChunkStart = old flushedUpto
  flushedUpto = LogstreamResult.Flush
  receivedTLI = tli
SpinLockRelease(&WalRcv->mutex)
WaitLSNWakeup(WAIT_LSN_TYPE_STANDBY_FLUSH, LogstreamResult.Flush)
WakeupRecovery()
if AllowCascadeReplication():
  WalSndWakeup(true, false)
update process title
XLogWalRcvSendReply(false, false, false)
XLogWalRcvSendHSFeedback(false)
```
`issue_xlog_fsync()` 会根据 `wal_sync_method` 选择 `fsync`、`fdatasync`、`fsync_writethrough`，或在 open sync 模式下快速返回。
失败时是 PANIC。
`flushedUpto` 更新在 spinlock 下完成。
这不是为了让 fsync 本身被锁保护。
锁保护的是 shared state 的一致快照：
```text
flushedUpto
receivedTLI
latestChunkStart
```
startup process 读取这些字段时也在同一把 spinlock 下。
### 5.11 WakeupRecovery: 唤醒 startup process 的真实含义
`WakeupRecovery()` 在 `xlogrecovery.c` 中非常短：
```text
SetLatch(&XLogRecoveryCtl->recoveryWakeupLatch)
```
它不传 LSN。
它不传 timeline。
它只让 startup process 从：
```text
WAIT_EVENT_RECOVERY_WAL_STREAM
```
醒来，重新检查：
```text
flushedUpto = GetWalRcvFlushRecPtr(&latestChunkStart, &receiveTLI)
```
`WaitForWALToBecomeAvailable()` 中的判断是：
```text
如果 RecPtr < flushedUpto 且 receiveTLI == curFileTLI:
  havedata = true
  必要时更新 XLogReceiptTime 和 current chunk start time
  XLogFileRead(readSegNo, receiveTLI, XLOG_FROM_STREAM, false)
```
否则继续等待。
这就是 latch 的正确 mental model：
```text
shared state 是事实；
latch 是重新检查事实的通知。
```
如果多个 flush 在 startup process 睡眠期间发生，latch 只会保持 set。
startup process 醒来后读取最新 `flushedUpto`，不需要知道中间发生了几次。
### 5.12 startup process 等不到 WAL 时请求 apply reply
`WaitForWALToBecomeAvailable()` 在准备睡等更多 streaming WAL 前，会做一件容易忽略的事：
```text
if !streaming_reply_sent:
  WalRcvRequestApplyReply()
  streaming_reply_sent = true
```
注释给出的原因是：当 startup process 已经 replay 完当前已收到的 WAL，并准备等待更多 WAL 时，应该尽快把 replay 位置告诉上游，避免主库 `pg_stat_replication` 看到 stale apply 信息。
`WalRcvRequestApplyReply()`：
```text
WalRcv->apply_reply_requested = true
读取 WalRcv->procno
SetLatch(walreceiver procLatch)
```
walreceiver 主循环在 `WL_LATCH_SET` 分支处理：
```text
ResetLatch(MyLatch)
if apply_reply_requested:
  apply_reply_requested = false
  pg_memory_barrier()
  XLogWalRcvSendReply(false, false, true)
```
`checkApply = true` 表示这次发送 reply 时要把 apply position 纳入“是否有进展”的判断。
因为 apply position 来自 startup process 的 replay 状态，读取它需要额外锁路径，普通 write/flush 触发的 reply 不总是检查它。
这条链路很重要：
```text
startup process replay 进度推进
  -> 请求 walreceiver 发 apply reply
  -> walreceiver 被 latch 唤醒
  -> standby status update 带新的 apply LSN
  -> primary walsender 更新 apply / replay_lsn
```
### 5.13 receiver status reply: 回给主库的三段进度
`XLogWalRcvSendReply(force, requestReply, checkApply)` 构造 `PqReplMsg_StandbyStatusUpdate`。
消息内容：
```text
message type = 'r'
writePtr = LogstreamResult.Write
flushPtr = LogstreamResult.Flush
applyPtr = GetXLogReplayRecPtr(NULL)
reply timestamp = GetCurrentTimestamp()
requestReply byte
```
发送条件不是“每次循环都发”。
如果 `force = false` 且 `wal_receiver_status_interval <= 0`，直接返回。
否则在这些情况下发送：
```text
force = true；
达到下一次 WALRCV_WAKEUP_REPLY；
write / flush 位置相对上次 reply 前进；
checkApply = true 且 apply 位置前进。
```
`requestReply = true` 用在心跳。
当 walreceiver 长时间没收到 primary 消息，达到 `wal_receiver_timeout / 2`，它会发一个要求对方立即回复的 status update。
如果达到 `wal_receiver_timeout` 仍没有任何 primary 消息，则 ERROR：
```text
terminating walreceiver due to timeout
```
主库 walsender 在 `ProcessStandbyReplyMessage()` 中解析这条消息。
它会更新 `MyWalSnd` shared state：
```text
write = writePtr
flush = flushPtr
apply = applyPtr
writeLag / flushLag / applyLag
replyTime
```
如果有 physical replication slot，还会用 `flushPtr` 推进 slot 的 received location。
如果不是 cascading walsender，还会调用 `SyncRepReleaseWaiters()` 释放满足条件的同步提交等待者。
因此 standby reply 不是单纯监控数据。
它参与主库上的同步复制和 WAL 保留边界。
### 5.14 timeout、ping 和 periodic wakeup
walreceiver 用 `wakeup[]` 统一周期任务：
```text
WALRCV_WAKEUP_TERMINATE:
  wal_receiver_timeout 到期。
WALRCV_WAKEUP_PING:
  wal_receiver_timeout / 2，到期后请求 primary 立即回复。
WALRCV_WAKEUP_REPLY:
  wal_receiver_status_interval，到期后发送 status update。
WALRCV_WAKEUP_HSFEEDBACK:
  hot_standby_feedback 周期。
```
收到任何 primary 消息后，主循环重新计算 terminate 和 ping 时间。
这说明 timeout 判断基于“最近收到主库消息”，不是基于“最近收到 WALData payload”。
Keepalive 也能刷新 `lastMsgReceiptTime` 和 timeout。
这能区分两类空闲：
```text
主库没有新 WAL，但连接还活着:
  primary keepalive 或 reply 让 walreceiver 不超时。
网络或上游进程失联:
  walreceiver 收不到任何消息，最终 timeout。
```
## 6. 生命周期 / ownership / cleanup
### 谁创建
startup process 不直接 fork walreceiver。
它通过 `RequestXLogStreaming()` 设置 `WalRcv` shared state，然后：
```text
STOPPED:
  SendPostmasterSignal(PMSIGNAL_START_WALRECEIVER)
WAITING:
  SetLatch(existing walreceiver procLatch)
```
postmaster 看到信号后启动 walreceiver auxiliary process。
### 谁持有
walreceiver 持有：
```text
primary replication connection；
当前打开的 recvFile fd；
LogstreamResult.Write / Flush；
reply_message buffer；
当前 wakeup schedule；
WalRcv shared state 中的 pid / procno / state。
```
startup process 持有：
```text
recovery state machine；
当前 WAL source；
recoveryWakeupLatch；
readFile / readSegNo；
replay position。
```
两者通过文件系统和 `WalRcv` shared state 连接。
startup process 不拿 walreceiver 的 fd。
walreceiver 不直接 redo WAL record。
### 谁释放
正常 stream 结束时：
```text
walrcv_endstreaming()
XLogWalRcvFlush(false)
close(recvFile)
处理 archive status
WalRcvWaitForStartPosition()
```
如果 primary 只是结束当前 timeline 的 streaming，walreceiver 不一定退出。
它进入 `WALRCV_WAITING`，等待 startup process 给新起点。
显式停止时，startup process 调用 `ShutdownWalRcv()`：
```text
walRcvState = STOPPING
kill(pid, SIGTERM)
ConditionVariableSleep(walRcvStoppedCV)
```
walreceiver 收到 SIGTERM 后通过 FATAL/exit 路径触发 `WalRcvDie()`。
### ERROR / exit 时谁兜底
兜底是 `on_shmem_exit(WalRcvDie)`。
它先尝试：
```text
XLogWalRcvFlush(true, tli)
```
然后把 shared state 设为 STOPPED 并广播 condition variable。
最后断开 replication connection 并 `WakeupRecovery()`。
这个顺序让 startup process 能够：
```text
先消费已经 flush 的 WAL；
再发现 walreceiver 已经停止；
随后回到 archive / pg_wal / streaming retry 状态机。
```
注意 `WalRcvDie()` 中的 flush 仍可能遇到底层 WAL fsync 问题。
如果本地 WAL 不能可靠持久化，PANIC 比继续恢复更合理。
## 7. 正确性机制层次
| 层次 | 机制 | 保证 | 不保证 |
| --- | --- | --- | --- |
| WAL 文件写入 | `XLogFileInit()` + `pg_pwrite()` | 字节按 LSN 落到目标 segment offset。 | 数据已持久。 |
| WAL 持久化 | `issue_xlog_fsync()` | 本地接收 WAL 到达 flush 边界。 | startup 已经 replay。 |
| shared state 一致性 | `WalRcv->mutex` | `flushedUpto`、`receivedTLI`、`latestChunkStart` 一起发布。 | `writtenUpto` 与这些字段同一快照一致。 |
| write 进度发布 | `pg_atomic_write_membarrier_u64(writtenUpto)` | 低成本发布 pwrite 进度。 | 可作为 recovery data integrity 边界。 |
| recovery 唤醒 | `WakeupRecovery()` / latch | startup process 会重新检查 `flushedUpto`。 | latch 携带 LSN 或消息数量。 |
| apply reply 请求 | `apply_reply_requested` + memory barrier + latch | startup process 可要求 walreceiver 及时发送 replay 进度。 | 主库一定立刻收到。 |
| primary 反馈 | `StandbyStatusUpdate` | 主库看到 write / flush / apply 分层进度。 | 两端时钟完全同步或 lag 完整归因。 |
| WaitLSN | `WaitLSNWakeup()` + proc latch | 等待 standby write / flush LSN 的进程能被唤醒。 | 直接说明网络或磁盘谁慢。 |
这里没有单一“复制进度”。
正确性来自分层：
```text
网络接收不等于写入；
写入不等于 fsync；
fsync 不等于 replay；
replay 不等于主库已经收到反馈；
主库收到反馈不等于所有 lag 指标都有因果解释。
```
这也是为什么 PostgreSQL 保留多个 LSN 字段。
多字段看起来复杂，但它避免把不同耐久性和可见性边界混在一起。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 primary 正常结束 COPY
`libpqrcv_receive()` 返回 `-1` 表示 server ended COPY。
walreceiver 记录日志：
```text
replication terminated by primary server
End of WAL reached on timeline ...
```
然后跳出 streaming loop，调用：
```text
walrcv_endstreaming(wrconn, &primaryTLI)
WalRcvFetchTimeLineHistoryFiles(startpointTLI, primaryTLI)
XLogWalRcvFlush(false)
close(recvFile)
WalRcvWaitForStartPosition()
```
这条路径通常不是错误。
它可能表示当前 timeline 到头了，startup process 需要重新扫描 timeline history。
walreceiver 可以进入 WAITING，等待新指令。
### 8.2 网络错误或协议错误
`libpqrcv_receive()` 在这些场景会 `ereport(ERROR)`：
```text
PQconsumeInput() 失败；
收到 unexpected result；
COPY 状态不符合协议；
rawlen < -1；
walrcv_send() 中 PQputCopyData / PQflush 失败。
```
walreceiver ERROR 后通过 exit callback 设置 STOPPED。
startup process 的状态机随后看到 `WalRcvStreaming()` 不成立，把当前 source 视为失败。
它会：
```text
回到 archive / pg_wal 尝试；
必要时 sleep wal_retrieve_retry_interval；
再请求 streaming。
```
所以复制连接断开不等于 standby crash。
它是 recovery 获取 WAL 的一个来源失败。
### 8.3 wal_receiver_timeout
如果长时间没有收到任何 primary 消息：
```text
达到 timeout / 2:
  发送 requestReply=true 的 status update，要求 primary 回复。
达到 timeout:
  ereport(ERROR, "terminating walreceiver due to timeout")
```
timeout 依赖 `ProcessWalSndrMessage()` 推进的接收时间。
只要 primary keepalive 正常，哪怕没有新 WAL payload，也不会触发 timeout。
诊断时要把“无新 WAL”和“连接无响应”分开。
### 8.4 本地写盘或 fsync 失败
`XLogWalRcvWrite()` 写失败会 PANIC。
`issue_xlog_fsync()` 失败也会 PANIC。
原因是 standby recovery 的输入 WAL 已经可能局部写坏或无法保证耐久。
继续运行会让 startup process 在不可靠的 WAL 文件上做判断。
这类故障通常表现为 standby 重启，而不是简单重连 primary。
### 8.5 config change 和 restart 边界
`xlogrecovery.c` 中有 `pendingWalRcvRestart`。
当 primary_conninfo、slot 或相关配置变化时，startup process 可能需要重启 walreceiver。
在 `XLOG_FROM_STREAM` 状态里：
```text
if pendingWalRcvRestart && !startWalReceiver:
  XLogShutdownWalRcv()
  可能 rescan latest timeline
  startWalReceiver = true
```
这说明有些变化不能靠正在运行的 walreceiver 热更新。
因为连接参数和 slot 是 `RequestXLogStreaming()` 传给 walreceiver 的启动指令。
### 8.6 streaming source 失败后的 archive fallback
`WaitForWALToBecomeAvailable()` 把 WAL 来源建成状态机：
```text
archive / pg_wal
  -> stream
  -> rescan timeline
  -> sleep wal_retrieve_retry_interval
  -> 回到 archive / pg_wal
```
如果 streaming 失败，它会先确保 walreceiver 不再活跃。
原因是：
```text
当 startup process 准备从 archive 恢复某个 segment 时，
不能让 walreceiver 同时覆盖或安装同一段 WAL。
```
源码中用 `XLogShutdownWalRcv()` 和 `ResetInstallXLogFileSegmentActive()` 维护这个边界。
## 9. 成本、资源与跨模块传播
walreceiver 的成本主要来自五处：
| 成本 | 触发点 | 影响 |
| --- | --- | --- |
| 网络 CopyData 接收 | `walrcv_receive()` | 受主库发送、网络和 libpq 缓冲影响。 |
| 本地 WAL pwrite | `XLogWalRcvWrite()` | WAL 量越大，写 I/O 越多。 |
| segment 初始化 | `XLogFileInit()` | 新 segment 可能 zero-fill 和 fsync，受 `wal_init_zero` 影响。 |
| WAL fsync | `XLogWalRcvFlush()` | `wal_sync_method`、存储延迟和 I/O 队列决定 flush lag。 |
| status reply / feedback | `XLogWalRcvSendReply()` | 受 `wal_receiver_status_interval` 和同步复制需求影响。 |
资源压力的传播路径：
```text
主库 WAL 生成快
  -> walsender 发送压力上升
  -> standby walreceiver pwrite / fsync 压力上升
  -> flushedUpto 推进变慢
  -> startup process 等待 WAIT_EVENT_RECOVERY_WAL_STREAM
  -> primary 上 flush_lag / replay_lag 可能扩大
```
如果瓶颈在 standby fsync，常见形态是：
```text
written_lsn 接近 latest_end_lsn；
flushed_lsn 明显落后 written_lsn；
pg_stat_activity 中 walreceiver 可能在 WAL_SYNC / WAL_WRITE 附近等待；
startup process 等待 streaming WAL 或 replay 进度变慢。
```
如果瓶颈在 redo apply，形态不同：
```text
flushed_lsn 已经接近 latest_end_lsn；
replay_lsn 或 apply LSN 落后；
startup process 忙于 redo 或被 Hot Standby conflict / I/O 拖慢；
主库 pg_stat_replication 的 replay_lag 扩大。
```
如果瓶颈在网络或 primary sender：
```text
latest_end_lsn 与主库当前 WAL write LSN 差距扩大；
last_msg_receipt_time 长时间不更新；
walreceiver 主要睡在 WAIT_EVENT_WAL_RECEIVER_MAIN；
timeout / ping 行为可能出现。
```
跨模块连接有四条：
```text
xlogrecovery.c:
  startup process 用 flushedUpto 选择 XLOG_FROM_STREAM 是否可读。
xlog.c:
  walreceiver 复用 segment init / fsync / shutdown 边界。
xlogwait.c:
  standby write / flush LSN 等待依赖 writtenUpto / flushedUpto 和 WaitLSNWakeup。
walsender.c:
  primary 用 standby status update 更新 pg_stat_replication、同步复制和 slot flush 确认。
```
## 10. 观测与诊断入口
### 10.1 standby 上的 `pg_stat_wal_receiver`
视图来自 `pg_stat_get_wal_receiver()`。
核心字段：
| 字段 | 来源 | 解释 |
| --- | --- | --- |
| `status` | `WalRcv->walRcvState` | connecting、streaming、waiting 等状态。 |
| `receive_start_lsn` | `WalRcv->receiveStart` | 本轮请求 streaming 的起点。 |
| `written_lsn` | `WalRcv->writtenUpto` | walreceiver 已 pwrite 的位置。 |
| `flushed_lsn` | `WalRcv->flushedUpto` | walreceiver 已 fsync 并发布给 recovery 的位置。 |
| `received_tli` | `WalRcv->receivedTLI` | flush 进度对应 timeline。 |
| `last_msg_send_time` | `WalRcv->lastMsgSendTime` | 主库消息中的 sendTime。 |
| `last_msg_receipt_time` | `WalRcv->lastMsgReceiptTime` | standby 接收该消息的本地时间。 |
| `latest_end_lsn` | `WalRcv->latestWalEnd` | 主库在最近消息里报告的 WAL end。 |
| `latest_end_time` | `WalRcv->latestWalEndTime` | latest_end_lsn 前进时对应的主库时间。 |
诊断时先分层：
```sql
SELECT status,
       written_lsn,
       flushed_lsn,
       latest_end_lsn,
       last_msg_send_time,
       last_msg_receipt_time
FROM pg_stat_wal_receiver;
```
解释边界：
```text
latest_end_lsn - written_lsn:
  standby 还没写到主库报告的 end-of-WAL。
written_lsn - flushed_lsn:
  已写但未 flush，可能是 fsync 或 flush 调度边界。
flushed_lsn - replay_lsn:
  WAL 已到 standby 本地并持久，但 startup process 还没 replay。
```
最后一项不在 `pg_stat_wal_receiver` 里。
需要结合 standby 的 recovery 视图或函数，例如 `pg_last_wal_replay_lsn()`。
### 10.2 primary 上的 `pg_stat_replication`
primary walsender 从 standby status update 获得：
```text
write_lsn
flush_lsn
replay_lsn
write_lag
flush_lag
replay_lag
```
这些字段滞后于 standby 实际状态。
原因是它们依赖 walreceiver 发送 reply，且 reply 受：
```text
wal_receiver_status_interval；
WAL write / flush 进展；
startup process 是否请求 apply reply；
keepalive requestReply；
网络可用性。
```
所以 primary 上看到 lag 不动，不一定表示 standby 没动。
可能只是 status reply 还没发或还没到。
### 10.3 wait event
本节常见 wait event：
| wait event | 典型含义 |
| --- | --- |
| `WAIT_EVENT_WAL_RECEIVER_MAIN` | walreceiver 等 socket、latch 或 timeout。 |
| `WAIT_EVENT_WAL_RECEIVER_WAIT_START` | walreceiver 已停止 streaming，等待 startup process 给新起点。 |
| `WAIT_EVENT_RECOVERY_WAL_STREAM` | startup process 等 walreceiver flush 更多 WAL。 |
| `WAIT_EVENT_RECOVERY_RETRIEVE_RETRY_INTERVAL` | 所有来源暂时失败，按 retry interval 睡眠。 |
| `WAIT_EVENT_WAL_WRITE` | walreceiver 正在写 WAL segment。 |
| `WAIT_EVENT_WAL_SYNC` | walreceiver 正在 fsync WAL segment。 |
| `WAIT_EVENT_WAIT_FOR_WAL_WRITE` | backend 等 standby write LSN 达到目标。 |
| `WAIT_EVENT_WAIT_FOR_WAL_FLUSH` | backend 等 standby flush LSN 达到目标。 |
wait event 是当前位置，不是完整因果。
例如 startup process 等 `RECOVERY_WAL_STREAM`，可能是网络慢、主库没 WAL、walreceiver fsync 慢、或者 walreceiver 已退出等待 retry。
需要回到 LSN 分层判断。
## 11. 常见误区
误区一：`written_lsn` 表示 standby 已经安全接收。
不准确。
`written_lsn` 是 pwrite 进度。
startup process 读取 streaming WAL 的安全边界是 `flushed_lsn`。
误区二：`WakeupRecovery()` 传递了新 WAL 的 LSN。
不是。
它只是 `SetLatch(&recoveryWakeupLatch)`。
startup process 醒来后必须重新读 shared state。
误区三：`last_msg_receipt_time` 不更新就一定是 standby apply 慢。
不是。
它只说明 walreceiver 最近没有处理主库消息。
可能是主库无新消息、网络断开、walsender 卡住或 walreceiver 进程不在 streaming。
误区四：primary 上 `replay_lag` 大就一定是 walreceiver 慢。
不一定。
如果 standby `flushed_lsn` 已经接近 `latest_end_lsn`，瓶颈更可能在 startup replay 或 hot standby conflict。
误区五：primary 结束 streaming 是异常。
不总是。
timeline 到头或切换时，walreceiver 可能进入 WAITING，等待 startup process 重新给起点。
误区六：status reply 只用于监控。
不是。
它还影响同步复制等待释放、主库 walsender shared state 和 replication slot 的 received location。
## 12. 课堂实验
### 实验一：跟一条 WALData 到 flush 发布
在 standby 上对 walreceiver 下断点：
```text
break XLogWalRcvProcessMsg
break XLogWalRcvWrite
break XLogWalRcvFlush
```
在 primary 上执行产生 WAL 的事务。
记录：
```text
WALData header:
  dataStart
  walEnd
  sendTime
写入后:
  LogstreamResult.Write
  WalRcv->writtenUpto
flush 后:
  LogstreamResult.Flush
  WalRcv->flushedUpto
  WalRcv->receivedTLI
```
目标：画出 network receive、write、flush、shared publish 的顺序。
如果 `writtenUpto` 已经前进但 `flushedUpto` 还没前进，解释为什么 startup process 不应把这段 WAL 当作 streaming source 成功。
### 实验二：观察 startup process 如何等待 WAL
在 startup process 上跟：
```text
break WaitForWALToBecomeAvailable
break GetWalRcvFlushRecPtr
```
同时观察：
```sql
SELECT pid, wait_event_type, wait_event
FROM pg_stat_activity
WHERE backend_type IN ('startup', 'walreceiver');
```
关注：
```text
startup process 进入 WAIT_EVENT_RECOVERY_WAL_STREAM；
walreceiver flush 后调用 WakeupRecovery；
startup process 醒来重新读取 flushedUpto；
如果 RecPtr < flushedUpto 且 timeline 匹配，打开 XLOG_FROM_STREAM 文件。
```
目标：理解 latch 唤醒与 LSN 条件检查是两步，不是一个原子消息。
### 实验三：比较 standby 与 primary 的可观测 lag
在 standby 查询：
```sql
SELECT status,
       written_lsn,
       flushed_lsn,
       latest_end_lsn,
       last_msg_send_time,
       last_msg_receipt_time
FROM pg_stat_wal_receiver;
```
在 primary 查询：
```sql
SELECT application_name,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```
对照回答：
```text
standby written_lsn 与 primary write_lsn 是否一致？
standby flushed_lsn 与 primary flush_lsn 是否一致？
primary replay_lsn 何时更新？
wal_receiver_status_interval 会让 primary 视图滞后多久？
```
目标：把“lag”拆成传输、写、flush、apply、反馈五个阶段。
## 13. 讨论题
1. 为什么 startup process 等 streaming WAL 时使用 `flushedUpto`，而不是 `writtenUpto`？
2. `latestChunkStart` 为什么要保存“上一次 flush 前的 flushedUpto”，它和 Hot Standby conflict grace time 有什么关系？
3. 如果 walreceiver 每条 WALData 后都立刻发送 status reply，会改善哪些延迟，又会增加哪些成本？
4. `WalRcvRequestApplyReply()` 为什么只设置一个 flag 并 SetLatch，而不是直接由 startup process 给主库发 reply？
5. `lastMsgSendTime` 和 `lastMsgReceiptTime` 能诊断网络延迟吗？哪些情况下这个推断会失真？
6. primary 结束 COPY 后 walreceiver 为什么可以进入 WAITING，而不是必须退出？
7. 如果 `written_lsn` 追上 `latest_end_lsn` 但 `flush_lsn` 不动，优先看哪些 wait event 和源码路径？
8. 如果 primary 上 `replay_lag` 大，但 standby 上 `flushed_lsn` 已追上，问题更可能在哪个模块？
## 14. 本节小结
walreceiver 的核心链路不是“收 WAL 然后写文件”这么简单。
它是一条分层发布协议：
```text
libpq CopyData
  -> XLogWalRcvProcessMsg()
  -> XLogWalRcvWrite()
  -> writtenUpto
  -> XLogWalRcvFlush()
  -> flushedUpto / receivedTLI / latestChunkStart
  -> WakeupRecovery()
  -> startup process 读取 XLOG_FROM_STREAM
  -> XLogWalRcvSendReply()
  -> primary walsender 更新 write / flush / apply
```
核心状态分成 backend-local 和 shared 两层。
`LogstreamResult`、`recvFile`、`recvSegNo` 描述 walreceiver 当前真实写入动作。
`WalRcvData` 只发布其它进程需要观察或修改的进度和控制状态。
正确性边界来自多机制组合：
```text
pwrite 发布 writtenUpto；
fsync 后发布 flushedUpto；
spinlock 保护 flush 相关字段一致性；
latch 唤醒 startup process 重新检查；
status reply 把 write / flush / apply 三个 LSN 回传主库；
exit callback 确保断线或 ERROR 后 shared state 收尾。
```
错误路径中，primary 结束 streaming、网络 ERROR、timeout、config restart 和 timeline 切换分别走不同边界。
不能把它们都解释成“复制坏了”。
可观测入口也必须分层：
```text
pg_stat_wal_receiver:
  standby 看到的接收、写、flush 和上游消息时间。
pg_stat_replication:
  primary 看到的 standby feedback，可能受 status interval 滞后。
wait event:
  当前等待点，不是完整因果。
gdb / 日志:
  验证 Write、Flush、WakeupRecovery 和 SendReply 的真实顺序。
```
可迁移规律：
```text
跨进程数据流不要把“我看见了字节”包装成一个全能进度。
先把网络、写入、持久化、消费和反馈拆成独立状态，
再用 shared state 发布事实，用 latch 触发重新检查，
用状态上报把本地事实投影给远端。
```
哪些判断仍然依赖环境？
```text
write_lag / flush_lag / replay_lag 的主因依赖网络、存储、WAL 生成速率、redo 成本和 status interval；
last_msg_send_time 与 last_msg_receipt_time 依赖两端时钟；
walreceiver timeout 与 keepalive 行为依赖配置；
timeline 和 archive fallback 依赖恢复目标与归档完整性。
```
下一节可以沿主库方向继续看：当 primary commit 需要等待 standby write、flush 或 apply 时，这些 receiver status update 如何进入同步复制等待和释放边界。
