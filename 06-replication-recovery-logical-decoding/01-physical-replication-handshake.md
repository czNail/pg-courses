# PostgreSQL 物理复制连接握手与复制协议入口
## 课程定位
前置知识：已经理解 WAL 是 PostgreSQL crash recovery 和复制的共同日志，也知道 standby 的 startup process 会按 LSN 顺序重放 WAL。
本节唯一主问题：
```text
standby 如何通过 replication connection、IDENTIFY_SYSTEM、START_REPLICATION 和 system identifier 校验，确认自己正在追随同一个集群历史？
```
核心矛盾：
```text
standby 必须尽快从上游拉取 WAL，进入连续流式复制；
但它绝不能把另一个集群、另一个不兼容 timeline 或一个未来 LSN 的 WAL 当成本集群历史继续重放。
```
这不是一个普通连接建立问题。
普通 SQL 连接只需要认证用户、选择数据库、进入查询循环。
物理复制连接还要回答三个更窄的问题：
```text
我连到的是 replication-mode 后端吗？
它暴露的 system identifier 是否等于本地 pg_control 中的 system identifier？
它是否能从我请求的 timeline 和 LSN 开始提供 WAL？
```
学完后应能判断：
```text
为什么 replication=true 的连接会变成 WAL sender；
为什么 IDENTIFY_SYSTEM 的 systemid 校验发生在 standby 侧；
为什么 START_REPLICATION 需要同时携带 LSN 和 TIMELINE；
为什么 timeline 不是 system identifier 的替代品；
为什么握手成功只说明“可沿这条历史继续读 WAL”，不说明延迟、同步提交或 apply 进度健康。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
用户给出的建议阅读文件中，当前 commit 下不存在 `src/include/replication/libpqwalreceiver.h`。
本课按真实源码处理：libpq walreceiver 的 hook ABI 在 `src/include/replication/walreceiver.h`，libpq 实现在 `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c`。
## 1. 本节在总主线中的位置
本目录第一组课讨论 physical streaming replication。
后续课程会继续讲 walsender 主循环、walreceiver 写盘、同步复制反馈和 keepalive。
本节只处理进入 streaming 前的握手边界。
也就是：
```text
startup process 决定需要从上游拉 WAL
  -> walreceiver 连接上游
  -> 上游把该连接识别为 replication connection
  -> standby 发送 IDENTIFY_SYSTEM
  -> standby 校验 system identifier 和 primary timeline
  -> standby 发送 START_REPLICATION start_lsn TIMELINE start_tli
  -> walsender 校验 timeline / LSN / slot
  -> 两端进入 COPY BOTH
```
进入 COPY BOTH 之后，walsender 如何读 WAL、等待 WAL、发送 CopyData，属于下一节。
walreceiver 如何把 CopyData 写成 segment 并推进 `flushedUpto`，属于第三节。
本节的终点是：
```text
standby 已经证明自己连到的上游可以在同一个 system identifier 下，
沿着请求 timeline 的历史，从请求 LSN 开始供应 WAL。
```
这个证明有清楚的边界。
它不是强一致性选主协议。
它不是节点身份认证的完整模型。
它也不是“这台上游一定是最新 primary”的证明。
它只是 PostgreSQL physical replication 在连接层建立的一组最低正确性检查。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
walreceiver 用 libpq 建立 replication=true 连接；
walsender 在 startup packet 阶段被标记为 WAL sender；
walreceiver 先执行 IDENTIFY_SYSTEM 并用本地 pg_control 的 system_identifier 比对远端 systemid；
随后用 START_REPLICATION 携带本地需要的 startpoint 和 startpointTLI；
walsender 根据自己的当前 flush position、timeline history 和 replication slot 状态决定是否切换到 COPY BOTH 发送 WAL。
```
这条链路把“同一个集群历史”拆成三层判断。
第一层是连接类型。
客户端必须在 startup packet 中声明 `replication=true` 或 `replication=database`。
否则服务端就是普通 backend，复制协议命令不会进入 walsender 的命令解析路径。
第二层是集群身份。
`IDENTIFY_SYSTEM` 返回上游 `ControlFile->system_identifier`。
standby 把它和本地 `GetSystemIdentifier()` 的结果比较。
不一致时，walreceiver 直接 ERROR。
第三层是历史位置。
`START_REPLICATION` 携带请求的 LSN 和 timeline。
walsender 检查请求 timeline 是否在本机历史中、请求起点是否没有落在错误分叉之后、请求起点是否不超过本机已 flush 的 WAL 位置。
这三个判断不能互相替代。
`replication=true` 只说明连接协议正确。
`system_identifier` 相同只说明 WAL 属于同一个初始化出来的 cluster lineage。
timeline 和 LSN 只说明当前请求点在某条可供应的 WAL 历史上。
把它们叠起来，standby 才能把网络上的字节流当作本地恢复历史的延续。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xlogrecovery.c` | startup process 在 `WaitForWALToBecomeAvailable()` 中决定启动或唤醒 walreceiver，并传入 `PrimaryConnInfo`、起始 LSN 和起始 TLI。 |
| 2 | `src/backend/replication/walreceiverfuncs.c` | `RequestXLogStreaming()` 如何把 `receiveStart`、`receiveStartTLI`、slot 和连接信息写入 `WalRcvData`。 |
| 3 | `src/backend/replication/walreceiver.c` | `WalReceiverMain()` 如何连接上游、执行 `IDENTIFY_SYSTEM`、校验 sysid / timeline、执行 `START_REPLICATION`。 |
| 4 | `src/backend/replication/libpqwalreceiver/libpqwalreceiver.c` | libpq 侧如何把 `replication=true` 放进连接参数，如何发送 `IDENTIFY_SYSTEM` 和 `START_REPLICATION ... TIMELINE ...`。 |
| 5 | `src/backend/tcop/backend_startup.c` | `ProcessStartupPacket()` 如何识别 startup packet 中的 `replication` 参数并设置 `am_walsender` / `am_db_walsender`。 |
| 6 | `src/backend/utils/init/postinit.c` | `InitPostgres()` 如何检查 REPLICATION 权限，物理 walsender 为什么不连接具体数据库。 |
| 7 | `src/backend/tcop/postgres.c` | `PostgresMain()` 如何在 walsender 连接上调用 `InitWalSender()` 和 `exec_replication_command()`。 |
| 8 | `src/backend/replication/repl_scanner.l` | replication command scanner 如何只看第一 token 判断是否走复制协议解析器。 |
| 9 | `src/backend/replication/repl_gram.y` | `IDENTIFY_SYSTEM` 和 `START_REPLICATION [SLOT] [PHYSICAL] RECPTR [TIMELINE]` 的语法节点。 |
| 10 | `src/backend/replication/walsender.c` | `IdentifySystem()` 与 `StartReplication()` 的服务端执行路径，以及 `WalSnd` shared state。 |
| 11 | `src/include/replication/walreceiver.h` | `WalRcvData` 和 `WalRcvStreamOptions` 的状态边界。 |
| 12 | `src/include/replication/walsender.h`、`src/include/replication/walsender_private.h` | `am_walsender`、`WalSndState`、`WalSnd`、`WalSndCtlData`。 |
| 13 | `src/include/nodes/replnodes.h` | replication grammar 产生的 `IdentifySystemCmd`、`StartReplicationCmd`。 |
| 14 | `src/include/catalog/pg_control.h`、`src/backend/access/transam/xlog.c` | `ControlFileData.system_identifier` 和 `GetSystemIdentifier()`。 |
推荐阅读顺序不是按文件名排序。
先从 standby 为什么要启动 walreceiver 开始。
再读 walreceiver 如何发起 replication connection。
然后切到服务端 startup packet，确认这条连接如何成为 walsender。
最后回到两条复制协议命令：`IDENTIFY_SYSTEM` 和 `START_REPLICATION`。
不要从 `WalSndLoop()` 开始。
`WalSndLoop()` 是流式复制进入 COPY BOTH 后的主循环。
本节关注的是进入这个循环前，哪些状态已经被确认。
## 4. 关键状态与边界
### `system_identifier`: 集群身份，不是节点名
`ControlFileData` 定义在 `src/include/catalog/pg_control.h`。
其中 `system_identifier` 的注释说明它用于确保 WAL 文件和生成它的安装实例匹配。
`GetSystemIdentifier()` 在 `src/backend/access/transam/xlog.c` 中只是一个很薄的 wrapper：
```text
GetSystemIdentifier()
  -> return ControlFile->system_identifier
```
所以本节的身份来源不是 GUC。
不是 `primary_conninfo`。
不是 replication slot。
不是 `application_name`。
而是本地 `pg_control` 里的 cluster-level identifier。
standby 是由 base backup 或其它物理复制方式从某个 cluster 派生出来的。
它的 `pg_control` 应该继承相同的 `system_identifier`。
如果你把一个独立 `initdb` 出来的目录错误配置成另一个 primary 的 standby，它们即使 schema 一样、用户名一样、LSN 格式一样，`system_identifier` 也会不同。
walreceiver 会在 `IDENTIFY_SYSTEM` 之后阻止继续 streaming。
这就是第一条硬边界：
```text
system_identifier 相同是 physical WAL 可被认为属于同一 cluster lineage 的必要条件。
```
它不是充分条件。
同一个 system identifier 下仍然可能有多个 timeline。
promotion、PITR、级联复制和历史分叉都可能让“接下来应该读哪条历史”变成 timeline 问题。
### `WalRcvData`: standby 侧的共享握手状态
`WalRcvData` 在 `src/include/replication/walreceiver.h`。
它是 shared memory 中的 walreceiver 管理状态。
startup process、walreceiver 和部分观测函数都会读写它。
本节关注这些字段组合：
| 字段 | 语义 |
| --- | --- |
| `walRcvState` | walreceiver 当前状态：`STOPPED`、`STARTING`、`CONNECTING`、`STREAMING`、`WAITING`、`RESTARTING`、`STOPPING`。 |
| `pid`、`procno` | 当前 walreceiver 进程身份；startup process 可以用 `procno` 设置 latch。 |
| `receiveStart`、`receiveStartTLI` | startup process 要求 walreceiver 从哪个 LSN 和 timeline 开始接收。 |
| `flushedUpto`、`receivedTLI` | walreceiver 已经 flush 到本地磁盘的 WAL 位置和对应 timeline。 |
| `writtenUpto` | 已写但不作为完整持久性判断的原子位置。 |
| `latestWalEnd`、`latestWalEndTime` | 上游最近报告的 WAL 末端位置和时间。 |
| `conninfo`、`sender_host`、`sender_port` | 对用户可见的连接信息，连接成功后会隐藏敏感字段。 |
| `slotname`、`is_temp_slot` | walreceiver 要使用或创建的 replication slot。 |
| `ready_to_display` | 统计视图是否可以显示连接信息。 |
这些字段不是 walreceiver 私有内存。
`RequestXLogStreaming()` 在 startup process 路径中设置 `receiveStart` 和 `receiveStartTLI`。
`WalReceiverMain()` 读取这些字段后，拿它们构造 `START_REPLICATION`。
`pg_stat_get_wal_receiver()` 也会在持有 `WalRcv->mutex` 时读取一组一致的状态。
所以 `receiveStart + receiveStartTLI` 是本节的 standby-side 请求语义。
单独看 `receiveStart` 不足以判断历史。
同一个 LSN 在不同 timeline 上可能代表不同历史分支。
### `WalRcvStreamOptions`: 发给 hook 的跨模块边界
walreceiver 不直接硬编码 libpq 的所有细节。
它通过 `WalReceiverFunctionsType` 调用 walreceiver module。
当前物理 standby 载入的是 `libpqwalreceiver`。
`WalRcvStreamOptions` 把上层意图传给实现模块：
| 字段 | 物理复制含义 |
| --- | --- |
| `logical` | 本节为 `false`。 |
| `slotname` | 如果配置了 slot，放入 `START_REPLICATION SLOT ...`。 |
| `startpoint` | 要求开始 streaming 的 LSN。 |
| `proto.physical.startpointTLI` | 要求开始 streaming 的 timeline。 |
`libpqrcv_startstreaming()` 会把这些字段拼成复制协议命令：
```text
START_REPLICATION [SLOT "slotname"] start_lsn TIMELINE start_tli
```
物理复制不使用 logical replication 的 publication、proto version、binary 等选项。
这个边界很重要。
`walreceiver.c` 决定“我要从哪里开始”。
`libpqwalreceiver.c` 决定“如何用 libpq 协议把这个意图发出去”。
### walsender flags: replication connection 的服务端身份
服务端在 `src/include/replication/walsender.h` 暴露了几个全局状态：
| 状态 | 语义 |
| --- | --- |
| `am_walsender` | 当前 backend 是否是 WAL sender。 |
| `am_db_walsender` | 是否是连接到具体数据库的 walsender，主要服务 logical replication 或需要数据库上下文的场景。 |
| `am_cascading_walsender` | 当前 walsender 是否运行在 recovery 中，也就是级联 standby 上的 walsender。 |
| `MyWalSnd` | 当前 walsender 在 `WalSndCtl->walsnds[]` 中占用的 shared state。 |
`replication=true` 的物理连接会设置 `am_walsender = true`，但不会连接具体数据库。
`replication=database` 会同时设置 `am_db_walsender = true`。
本节主线是物理 streaming，所以真实 walreceiver 侧调用 `walrcv_connect(conninfo, true, false, ...)`。
在 `libpqrcv_connect()` 中，这会把 libpq 参数设置为：
```text
replication = true
dbname = replication
fallback_application_name = cluster_name 或 walreceiver
```
这里的 `dbname=replication` 主要用于 `.pgpass` 查找。
服务端在物理 replication mode 下会忽略数据库名，并把 `port->database_name[0]` 置空。
### `WalSnd`: primary 侧可观测复制状态
`WalSnd` 定义在 `src/include/replication/walsender_private.h`。
它是 shared memory 中每个 walsender 的状态槽。
本节关注：
| 字段 | 语义 |
| --- | --- |
| `pid` | 当前 walsender 是否占用该 slot。 |
| `state` | `startup`、`catchup`、`streaming`、`stopping` 等可观测状态。 |
| `sentPtr` | 已发送到哪个 WAL 位置，实际是下一次发送位置。 |
| `write`、`flush`、`apply` | standby 反馈回来的写入、flush、apply 位置。 |
| `replyTime` | 最近收到 standby reply 的时间。 |
| `kind` | 物理或逻辑 walsender。 |
`InitWalSenderSlot()` 会在 `InitWalSender()` 中给当前进程占用一个 `WalSnd`。
`StartReplication()` 成功进入流式复制前，会把状态设为 `WALSNDSTATE_CATCHUP`，并初始化 `sentPtr` 和 `MyWalSnd->sentPtr`。
后续是否进入 `streaming`，以及 write/flush/apply 如何推进，是后续课程内容。
本节只需要知道：握手成功之后，`pg_stat_replication` 看到的是 `WalSnd` 里的共享状态，而不是 `IDENTIFY_SYSTEM` 的返回值。
### replication command parse nodes
`src/include/nodes/replnodes.h` 中的 `StartReplicationCmd` 是服务端执行 `START_REPLICATION` 的语义载体：
```text
StartReplicationCmd:
  kind
  slotname
  timeline
  startpoint
  options
```
物理复制中：
```text
kind = REPLICATION_KIND_PHYSICAL
slotname = optional
timeline = START_REPLICATION 命令中的 TIMELINE，缺省为 0
startpoint = RECPTR
options = NULL
```
`timeline = 0` 不是 timeline 0。
它表示客户端没有显式指定 timeline，服务端会选择当前 flush/replay 位置对应的 timeline。
walreceiver 生成的物理复制命令通常会显式带上 `TIMELINE startpointTLI`。
这样服务端能检查 standby 想追随的是哪条历史。
## 5. 主流程源码 walkthrough
### 5.1 startup process 决定需要 streaming
standby 上真正重放 WAL 的是 startup process。
它在 recovery 过程中调用 `WaitForWALToBecomeAvailable()`。
当本地 `pg_wal` 和 archive 都不能满足下一个需要读取的位置，并且 `PrimaryConnInfo` 非空时，它会进入 streaming 分支。
关键路径在 `src/backend/access/transam/xlogrecovery.c`：
```text
WaitForWALToBecomeAvailable(RecPtr, ...)
  -> 计算应该从哪个 ptr 和 tli 开始 stream
  -> SetInstallXLogFileSegmentActive()
  -> RequestXLogStreaming(tli, ptr, PrimaryConnInfo,
                          PrimarySlotName,
                          wal_receiver_create_temp_slot)
  -> 等待 WalRcvStreaming()
```
这里的 `ptr` 不一定就是调用时的 `RecPtr`。
如果正在 fetch checkpoint，代码可能用 `RedoStartLSN` 和 `RedoStartTLI`。
普通路径中，它会用当前请求位置和 `tliOfPointInHistory()` 从 expected timeline history 中算出 `tli`。
这一步已经体现了本节主问题：
```text
startup process 不是只说“给我某个 LSN”；
它必须同时说“这个 LSN 属于哪条 timeline”。
```
随后 `RequestXLogStreaming()` 在 `src/backend/replication/walreceiverfuncs.c` 里写 `WalRcvData`。
它会把非 segment 起点的 `recptr` 向下调整到 segment 开头。
这样可以避免流式复制创建一个前半段没有 record 的坏 segment。
然后它在 `WalRcv->mutex` 下设置：
```text
slotname / is_temp_slot
walRcvState = STARTING 或 RESTARTING
receiveStart = recptr
receiveStartTLI = tli
flushedUpto / receivedTLI / latestChunkStart 初始位置
writtenUpto 原子位置
```
如果 walreceiver 还没启动，它发送 `PMSIGNAL_START_WALRECEIVER`。
如果 walreceiver 正在 `WAITING`，它用 `procno` 找到 walreceiver 的 `PGPROC` 并 set latch。
所以 `RequestXLogStreaming()` 是 startup process 到 walreceiver 的指令边界。
它没有连接网络。
它只把“从哪个历史位置开始拉 WAL”发布到 shared memory。
### 5.2 walreceiver 读取请求并建立 replication connection
walreceiver 进程入口是 `WalReceiverMain()`。
它先走 `AuxiliaryProcessMainCommon()`，获得 auxiliary process 的 shared memory 身份。
随后它把 `WalRcv` 状态从 `STARTING` 切到 `CONNECTING`。
同时复制出连接需要的本地快照：
```text
conninfo = walrcv->conninfo
slotname = walrcv->slotname
is_temp_slot = walrcv->is_temp_slot
startpoint = walrcv->receiveStart
startpointTLI = walrcv->receiveStartTLI
```
这一步很重要。
后续网络连接和复制协议命令不直接在持锁状态下读取 `WalRcv`。
`WalRcv->mutex` 保护 shared state 的一致性。
真正握手使用的是 walreceiver 本地变量。
然后 walreceiver 注册退出回调：
```text
on_shmem_exit(WalRcvDie, PointerGetDatum(&startpointTLI))
```
如果握手过程中 ERROR 或进程退出，`WalRcvDie()` 会 flush 已收到 WAL、把 `walRcvState` 设成 `STOPPED`、清掉 `pid` / `procno` / `ready_to_display`，断开连接，并唤醒 startup process。
接下来它载入 libpq 实现：
```text
load_file("libpqwalreceiver", false)
WalReceiverFunctions != NULL
```
然后建立连接：
```text
wrconn = walrcv_connect(conninfo, true, false, false, appname, &err)
```
四个布尔参数中，本节最关键的是：
```text
replication = true
logical = false
```
这会让 `libpqrcv_connect()` 在 libpq 参数里放入 `replication=true`。
也就是说，physical replication connection 的入口不是 SQL 命令 `START_REPLICATION` 本身。
入口从 startup packet 就开始了。
如果连接失败，walreceiver 抛出：
```text
streaming replication receiver "..." could not connect to the primary server: ...
```
此时还没有 `IDENTIFY_SYSTEM`。
问题属于网络、认证、连接参数或上游可达性，不属于 system identifier mismatch。
### 5.3 服务端 startup packet 把连接切成 walsender
服务端普通连接从 `BackendStartup()` 到 `BackendInitialize()`。
`ProcessStartupPacket()` 读取 frontend startup packet 中的 name/value 参数。
当它遇到参数名 `replication` 时，有两种合法 replication mode：
```text
replication = database:
  am_walsender = true
  am_db_walsender = true
replication = true / 1 / on:
  am_walsender = true
  am_db_walsender = false
```
如果值不是 bool，也不是 `database`，服务端 FATAL：
```text
invalid value for parameter "replication"
```
只要 `am_walsender` 为真，`ProcessStartupPacket()` 就会把 `MyBackendType` 设为 `B_WAL_SENDER`。
对物理 walsender，还会清空数据库名：
```text
if (am_walsender && !am_db_walsender)
    port->database_name[0] = '\0';
```
这就是物理复制连接和普通 SQL 连接的第一条服务端分叉。
服务端稍后进入 `PostgresMain()`。
`InitPostgres()` 会检查：
```text
if (am_walsender && !has_rolreplication(GetUserId()))
    FATAL "permission denied to start WAL sender"
```
物理 walsender 不连接具体数据库。
`InitPostgres()` 在处理 startup options、client encoding、stats backend finalization 和 connection warnings 后直接返回。
它不会继续查找 `pg_database`、设置 `MyDatabaseId` 或加载普通 session library。
这解释了一个常见边界：
```text
物理复制连接可以执行 replication protocol commands；
但它不是某个数据库里的普通 SQL session。
```
`PostgresMain()` 完成通用初始化后，如果 `am_walsender` 为真，会调用 `InitWalSender()`。
`InitWalSender()` 里：
```text
am_cascading_walsender = RecoveryInProgress()
InitWalSenderSlot()
```
`InitWalSenderSlot()` 在 `WalSndCtl->walsnds[]` 中找空槽，设置 `pid`、`state = WALSNDSTATE_STARTUP`、`sentPtr = InvalidXLogRecPtr`、反馈 LSN 为 invalid，并根据 `MyDatabaseId` 决定 `kind`。
物理 walsender 的 `MyDatabaseId` 是 `InvalidOid`，因此 `kind = REPLICATION_KIND_PHYSICAL`。
最后注册：
```text
on_shmem_exit(WalSndKill, 0)
```
这条服务端路径完成后，连接已经是 walsender，但还没有开始 stream。
它还在等客户端发 replication protocol command。
### 5.4 replication command parser 接管 Query message
PostgreSQL 复制协议命令不是 extended query protocol。
在 `PostgresMain()` 主循环中，收到 `PqMsg_Query` 后：
```text
if (am_walsender)
{
    if (!exec_replication_command(query_string))
        exec_simple_query(query_string);
}
else
    exec_simple_query(query_string);
```
也就是说 walsender 连接先尝试用 replication command parser 解析 query string。
其它消息类型会被 `forbidden_in_wal_sender()` 拦截。
例如 extended query protocol 在 replication connection 中不支持。
`exec_replication_command()` 做了几件本节相关的事。
第一，它检查 walsender 是否处于 `WALSNDSTATE_STOPPING`。
如果 primary 正在进入 shutdown 相关状态，新的复制命令会被拒绝。
第二，它为 replication command 准备一个长期复用的 `cmd_context`。
这个 context 挂在 `TopMemoryContext` 下，每次命令前 reset。
原因是 replication command 执行过程中可能打开或结束事务，不能用随事务退出就失效的 context 管命令解析状态。
第三，它调用：
```text
replication_scanner_init(cmd_string, &scanner)
replication_scanner_is_replication_command(scanner)
```
`repl_scanner.l` 的 first-token 判断只识别 replication command 的第一个关键字。
它不会把普通 SQL 完整 lex 一遍。
如果第一 token 不是 replication command，物理 walsender 又没有数据库上下文，则报错：
```text
cannot execute SQL commands in WAL sender for physical replication
```
第四，真正解析由 `replication_yyparse()` 完成。
`repl_gram.y` 中本节核心语法是：
```text
IDENTIFY_SYSTEM
START_REPLICATION [SLOT slot] [PHYSICAL] RECPTR [TIMELINE uint]
```
解析结果进入 switch：
```text
T_IdentifySystemCmd
  -> IdentifySystem()
T_StartReplicationCmd
  -> StartReplication(cmd)   // physical
```
这就是服务端 replication protocol command 的入口。
### 5.5 `IDENTIFY_SYSTEM`: 上游暴露身份和当前位置
walreceiver 建立连接后，第一条核心复制协议命令是 `IDENTIFY_SYSTEM`。
在 standby 侧，`libpqrcv_identify_system()` 执行：
```text
libpqsrv_exec(conn->streamConn,
              "IDENTIFY_SYSTEM",
              WAIT_EVENT_LIBPQWALRECEIVER_RECEIVE)
```
它要求返回 `PGRES_TUPLES_OK`。
返回列少于 3 或行数不是 1 时，walreceiver 报 protocol violation。
当前版本服务端返回 4 列。
历史上 PostgreSQL 9.3 及更早版本返回 3 列，所以客户端接受“至少 3 列”。
服务端执行函数在 `walsender.c`：
```text
IdentifySystem()
  -> sysid = GetSystemIdentifier()
  -> if RecoveryInProgress()
         logptr = GetStandbyFlushRecPtr(&currTLI)
     else
         logptr = GetFlushRecPtr(&currTLI)
  -> 输出 systemid, timeline, xlogpos, dbname
```
这四列的语义分别是：
| 列 | 含义 |
| --- | --- |
| `systemid` | 上游 `pg_control` 中的 `system_identifier`，以字符串返回。 |
| `timeline` | 上游当前可报告的 timeline。primary 上来自 flush position；级联 standby 上来自 standby flush position。 |
| `xlogpos` | 上游当前 flush 到的位置，用文本 LSN 返回。 |
| `dbname` | 如果 walsender 连接了数据库则返回数据库名；物理 `replication=true` 连接通常为 NULL。 |
`IDENTIFY_SYSTEM` 自己不接收 standby 的 system identifier。
也就是说，primary 并不知道这个 standby 的本地 sysid 是多少。
校验发生在 standby 侧。
这是一个设计边界：
```text
walsender 负责如实暴露自己的 cluster identity；
walreceiver 负责判断这个 identity 是否能被本地 recovery 接受。
```
这个边界也适用于级联复制。
一个 standby 上的 walsender 如果处于 recovery 中，会用 `GetStandbyFlushRecPtr()` 报告自己已经 flush 的 WAL 位置和 timeline。
它不需要知道下游 standby 的本地 control file。
### 5.6 standby 校验 system identifier
`WalReceiverMain()` 在每轮连接或重启 streaming 前执行：
```text
primary_sysid = walrcv_identify_system(wrconn, &primaryTLI)
standby_sysid = GetSystemIdentifier()
if (strcmp(primary_sysid, standby_sysid) != 0)
    ERROR "database system identifier differs between the primary and standby"
```
错误 detail 会同时打印 primary 和 standby 的 identifier。
这是误连不同 cluster 时最关键的可观测信号。
它通常说明：
```text
standby 不是从这个 primary 的 base backup 派生；
或者 primary_conninfo 指向了错误实例；
或者测试环境中复用了错误的数据目录；
或者运维切换时把 standby 指到了另一个独立 initdb 出来的集群。
```
它不说明：
```text
用户名错；
密码错；
网络不通；
slot 不存在；
timeline history 缺失。
```
这些会在更早或更晚的路径上报错。
sysid mismatch 是连接已经成功、上游已经能执行 replication command、但本地拒绝继续把远端 WAL 当作同一历史的错误。
注意比较使用的是字符串。
服务端 `IdentifySystem()` 把 uint64 sysid 格式化成文本。
客户端 `libpqrcv_identify_system()` 也把第一列作为字符串返回。
walreceiver 不需要把它转换回 uint64 才比较。
这不是核心设计，只是协议兼容和实现细节。
语义上比较的是同一个 `ControlFile->system_identifier`。
### 5.7 standby 校验 primary timeline 不落后
sysid 通过后，walreceiver 继续检查：
```text
if (primaryTLI < startpointTLI)
    ERROR "highest timeline %u of the primary is behind recovery timeline %u"
```
这个检查解决的是另一个问题。
即使 system identifier 相同，上游也可能不是 standby 当前要追的 timeline 的后代或同代。
例如 standby 已经知道自己要在 timeline 5 上继续恢复，却连到了一个只知道 timeline 3 的上游。
这时继续发 `START_REPLICATION ... TIMELINE 5` 没有意义。
上游最高 timeline 已经落后于 standby 的恢复目标。
这里的检查只是“上游最高 timeline 不小于请求 timeline”。
它不是完整 history membership 校验。
完整的请求点是否在上游历史中，要由后续 `StartReplication()` 中的 timeline history 检查完成。
然后 walreceiver 调用：
```text
WalRcvFetchTimeLineHistoryFiles(startpointTLI, primaryTLI)
```
它会尽量从上游获取缺失的 timeline history 文件。
源码注释说明，即使当前不打算追某条 timeline，也会获取。
这样如果将来本 standby 被 promotion，不容易选择一个上游已经使用过的 timeline ID。
这不是 bullet-proof 的全局 timeline 分配协议。
但它减少了明显的 timeline id collision。
### 5.8 `START_REPLICATION`: standby 把本地恢复位置变成协议请求
校验通过后，walreceiver 构造 `WalRcvStreamOptions`：
```text
options.logical = false
options.startpoint = startpoint
options.slotname = slotname 或 NULL
options.proto.physical.startpointTLI = startpointTLI
```
然后调用：
```text
walrcv_startstreaming(wrconn, &options)
```
libpq 实现拼出的命令大致是：
```text
START_REPLICATION SLOT "slot" 0/5000000 TIMELINE 3
```
如果没有 slot，就没有 `SLOT` 子句。
如果是物理复制，就不会带 `LOGICAL` 和 options。
`libpqrcv_startstreaming()` 期望两类正常结果。
如果返回 `PGRES_COPY_BOTH`，说明服务端已经切换到 bidirectional COPY，streaming 开始。
如果返回 `PGRES_COMMAND_OK`，说明命令成功执行但没有进入 COPY 模式。
这个 false 返回通常表示请求的 timeline / 起点上已经没有 WAL 可流，服务端在历史 timeline 尽头结束 streaming。
其它状态会报：
```text
could not start WAL streaming: ...
```
这也是一个边界：
```text
START_REPLICATION 成功进入 COPY BOTH，才意味着 walsender 接受了请求位置并开始流；
仅连接成功或 IDENTIFY_SYSTEM 成功，不等于已经开始 streaming。
```
### 5.9 服务端解析 `START_REPLICATION`
服务端 `repl_gram.y` 把物理命令解析成 `StartReplicationCmd`。
语法里 `PHYSICAL` 是可选关键字。
所以这两个命令在物理路径上等价：
```text
START_REPLICATION 0/5000000 TIMELINE 3
START_REPLICATION PHYSICAL 0/5000000 TIMELINE 3
```
如果带 slot：
```text
START_REPLICATION SLOT standby_slot 0/5000000 TIMELINE 3
```
`opt_timeline` 中 timeline 必须大于 0。
不带 timeline 时，`cmd->timeline = 0`。
walreceiver 的物理命令会显式带 timeline，因为它从 startup process 收到了 `receiveStartTLI`。
`exec_replication_command()` 看到 `cmd->kind == REPLICATION_KIND_PHYSICAL` 后调用：
```text
StartReplication(cmd)
```
这时还没有开始发送 WAL。
服务端要先完成 slot、timeline 和 LSN 检查。
### 5.10 walsender 校验 slot 类型
`StartReplication()` 的第一步之一是创建 physical WAL reader：
```text
XLogReaderAllocate(...)
```
如果指定了 slot：
```text
ReplicationSlotAcquire(cmd->slotname, true, true)
if (SlotIsLogical(MyReplicationSlot))
    ERROR "cannot use a logical replication slot for physical replication"
```
这里没有校验 slot 的 `restart_lsn` 是否覆盖请求起点。
源码注释说，walsender 依赖调用者请求 starting point。
如果对应 WAL segment 不存在，后面读取 WAL 时会失败。
因此 slot 在本节握手中有两个作用边界：
```text
slot 类型必须与 physical replication 匹配；
slot 的 WAL 保留能力不是 START_REPLICATION 入口处立即完整证明的。
```
slot 如何保留 WAL、`restart_lsn` 如何推进，是后续 replication slot 课程。
本节只把它作为 START_REPLICATION 的可选约束。
### 5.11 walsender 选择并校验 timeline
`StartReplication()` 接着确定上游当前可发送到哪里：
```text
if RecoveryInProgress()
    FlushPtr = GetStandbyFlushRecPtr(&FlushTLI)
else
    FlushPtr = GetFlushRecPtr(&FlushTLI)
```
如果客户端显式指定了 timeline：
```text
sendTimeLine = cmd->timeline
```
如果 `sendTimeLine == FlushTLI`，它是当前 timeline。
如果不同，walsender 把它视为 historic timeline。
然后：
```text
timeLineHistory = readTimeLineHistory(FlushTLI)
switchpoint = tliSwitchPoint(cmd->timeline, timeLineHistory,
                             &sendTimeLineNextTLI)
```
这里的关键语义是：
```text
请求的 timeline 必须存在于本服务端当前历史中。
```
如果请求 timeline 不在当前历史中，`tliSwitchPoint()` 或相关 timeline 读取路径会报错。
如果找到了 timeline，walsender 继续检查请求起点是否在它的历史上。
源码里的条件是：
```text
if (switchpoint is valid && switchpoint < cmd->startpoint)
    ERROR "requested starting point ... on timeline ... is not in this server's history"
```
这句话容易读反。
当服务端历史从请求 timeline fork 到下一条 timeline 的 switchpoint 早于客户端请求的 startpoint 时，说明客户端想在旧 timeline 的 fork 之后继续读旧 timeline。
这已经不属于服务端当前历史。
所以 walsender 拒绝。
源码注释也说明这个检查有意比较宽松。
客户端可以请求从包含 switchpoint 的 WAL segment 开头开始读新 timeline，以避免产生 partial segment。
如果请求位置太老，真正读取 segment 时再失败。
因此 timeline 检查不是“每个字节都精确证明”。
它是为了防止明显错误的历史分叉请求。
### 5.12 walsender 校验 startpoint 不在未来
timeline 检查之后，如果确实有内容可 stream，walsender 先把状态设成 catchup，并发送 `CopyBothResponse`。
随后检查：
```text
if (FlushPtr < cmd->startpoint)
    ERROR "requested starting point ... is ahead of the WAL flush position of this server ..."
```
这条检查防止 standby 请求一个上游还没有 flush 到本地磁盘的位置。
physical replication 的发送边界是已 flush 的 WAL，而不是内存中可能存在的未持久 WAL。
这和 crash safety 相关。
如果 primary crash 后未 flush 的 WAL 丢失，而 standby 却已经接收并重放这些 WAL，就会制造不可恢复的历史分歧。
所以 walsender 不能把未来 flush position 之后的 WAL 当作可复制历史。
校验通过后：
```text
sentPtr = cmd->startpoint
MyWalSnd->sentPtr = sentPtr
SyncRepInitConfig()
replication_active = true
WalSndLoop(XLogSendPhysical)
```
到这里，本节的握手主流程完成。
两端已经进入 COPY BOTH。
之后 walsender 会在 `XLogSendPhysical()` 中读取 WAL，walreceiver 会用 `XLogWalRcvProcessMsg()` 和 `XLogWalRcvWrite()` 写入本地 segment。
那些是下一节和第三节的主问题。
## 6. walsender 与 walreceiver 的边界
这条握手链路最容易误解的地方，是把 walsender 和 walreceiver 看成对称的两个半边。
它们不是对称的。
standby 侧持有“我本地恢复需要什么”的状态。
primary 侧持有“我能供应什么”的状态。
双方通过两个复制命令交换最小信息。
`IDENTIFY_SYSTEM` 的信息方向是：
```text
walsender -> walreceiver:
  systemid
  current timeline
  current xlog flush position
  optional dbname
```
校验方向是：
```text
walreceiver:
  compare remote systemid with local GetSystemIdentifier()
  compare primaryTLI with local startpointTLI
  fetch timeline history files
```
`START_REPLICATION` 的信息方向是：
```text
walreceiver -> walsender:
  optional slot
  start LSN
  start timeline
```
校验方向是：
```text
walsender:
  slot type
  requested timeline in server history
  requested startpoint not after fork point on requested timeline
  requested startpoint not beyond local flush position
```
这个分工让 PostgreSQL 可以支持级联复制。
级联 standby 作为下游的 primary 时，它的 walsender 只需要报告自己的已 flush 位置和 timeline。
它不需要知道下游是怎样从 base backup 派生出来的。
下游 walreceiver 自己拿本地 `pg_control` 做 sysid 校验。
这个分工也解释了为什么没有一个单独的“握手结构体”跨网络传输所有状态。
PostgreSQL 复用了 libpq startup packet、simple query 形式的 replication commands、COPY BOTH 和 shared memory stats。
它是历史兼容、运维可观测性和已有 FE/BE 协议共同形成的边界。
## 7. 正确性机制层次
### 第一层：认证和角色权限
连接到上游之前，libpq 连接字符串、`.pgpass`、host-based authentication 和 SSL/GSS 等机制先决定“能否建立连接”。
服务端 `InitPostgres()` 再检查角色是否有 `REPLICATION` 权限。
这层保证：
```text
不是任意用户都能启动 WAL sender。
```
它不保证：
```text
这个用户连到的是正确 cluster。
```
正确 cluster 的判断在 `IDENTIFY_SYSTEM`。
### 第二层：replication connection mode
startup packet 中的 `replication=true` 把服务端切到 walsender 路径。
这层保证：
```text
后续 Query message 会优先进入 replication command parser。
```
它不保证：
```text
该连接一定会开始 streaming；
也不保证请求的 LSN / timeline 正确。
```
### 第三层：system identifier
`system_identifier` 来自 `pg_control`。
这层保证：
```text
远端 WAL 声称属于同一个 cluster lineage。
```
它不保证：
```text
远端处于当前最新 timeline；
远端能供应我需要的 LSN；
远端没有配置成错误的级联链路。
```
所以 sysid 通过后还必须继续检查 timeline 和 LSN。
### 第四层：timeline history
timeline 检查保证：
```text
请求的 startpointTLI 在上游当前历史中；
请求点没有位于上游已经从该 timeline 分叉之后的位置。
```
它不保证：
```text
本地已经拥有所有需要的 WAL segment；
slot 一定保留住所有历史；
archive 一定完整。
```
这些会在读取、fetch history file 或 recovery 主循环中继续暴露。
### 第五层：flush position
`FlushPtr < cmd->startpoint` 检查保证：
```text
walsender 不从尚未 flush 到本机磁盘的位置开始发送。
```
这层和 crash safety 相关。
它把 physical replication 的发送边界固定在 durable WAL 上。
它不保证 standby 已经写入、flush 或 replay。
standby 进度要靠后续 reply message 和 `pg_stat_replication` 中的 write/flush/replay 位置观测。
### 第六层：COPY BOTH 之后的持续协议
握手完成后，walsender 和 walreceiver 进入持续流式协议。
keepalive、reply_requested、timeout、write/flush/apply feedback 都在这层。
本节不展开这层。
但要记住：
```text
握手正确只保证起点正确；
持续复制健康还要看后续位置推进和超时机制。
```
## 8. 生命周期、ownership 与 cleanup
### walreceiver 的生命周期
`RequestXLogStreaming()` 创建的是启动请求，不是进程本身。
真正进程由 postmaster 根据 `PMSIGNAL_START_WALRECEIVER` 启动。
walreceiver 在 `WalReceiverMain()` 中把自己登记到 `WalRcvData`：
```text
pid = MyProcPid
procno = MyProcNumber
walRcvState = WALRCV_CONNECTING
ready_to_display = false -> true
```
连接成功后，它把用户可见的 conninfo、sender host、sender port 写回 `WalRcvData`。
握手成功进入 streaming 后，它把状态从 `CONNECTING` 切到 `STREAMING`。
如果 streaming 到达 timeline 尽头，它可能进入 `WAITING`，等待 startup process 下达新的 startpoint。
如果被要求停止，它进入 `STOPPING`，最终 `WalRcvDie()` 切到 `STOPPED`。
`WalRcvDie()` 是 walreceiver 的 cleanup 兜底。
它做四件事：
```text
flush 已收到的 WAL
清理 WalRcvData 中的 pid / procno / ready_to_display
断开 libpq connection
唤醒 startup process
```
所以即使握手在 `IDENTIFY_SYSTEM` 或 `START_REPLICATION` 处失败，startup process 也能观察到 walreceiver 停止，而不是永远等待。
### walsender 的生命周期
walsender 是普通 backend 分叉后的特殊模式。
它先通过 startup packet 变成 `am_walsender`。
然后在 `PostgresMain()` 中调用 `InitWalSender()` 占用一个 `WalSnd` slot。
`WalSndKill()` 是 shared-memory slot cleanup。
它会把 `walsnd->pid = 0`。
如果 `exec_replication_command()` 或后续命令 ERROR，`PostgresMain()` 的顶层错误恢复会调用：
```text
WalSndErrorCleanup()
ReplicationSlotRelease()
ReplicationSlotCleanup(false)
```
这解释了为什么 `StartReplication()` 里 acquire 的 slot 不应该依赖普通事务 abort 自动清理。
复制命令可能跨普通事务边界管理资源。
顶层 walsender error cleanup 才是这个连接级资源的兜底。
### 命令解析 context 的生命周期
`exec_replication_command()` 复用静态 `cmd_context`。
每条复制命令前 reset。
它不为每条命令新建 context。
源码注释解释了原因：复制命令内部可能开始或结束事务，而事务退出会切换回事务开始时的 CurrentMemoryContext。
如果每条命令都创建并销毁 context，ERROR 路径上很容易让事务 cleanup 回到已经释放的 context。
所以这里的 ownership 不是“最短生命周期最好”。
而是：
```text
命令工作内存必须跨复制命令中的事务边界保持有效；
但每条顶层命令开始前必须 reset，避免 ERROR 后泄漏。
```
这个细节和握手正确性间接相关。
`IDENTIFY_SYSTEM` 可能短暂打开事务读取数据库名。
`START_REPLICATION` 可能 acquire slot。
复制命令解析和执行不能简单套用普通 SQL command 的内存生命周期。
## 9. 错误路径与异常边界
### 连接和权限失败
如果 libpq 连接失败，walreceiver 报 connection failure。
这时没有 remote sysid。
日志和 `pg_stat_wal_receiver` 可能只看到 `connecting` 或短暂进程退出。
如果服务端用户没有 `REPLICATION` 权限，`InitPostgres()` FATAL：
```text
permission denied to start WAL sender
Only roles with the REPLICATION attribute may start a WAL sender process.
```
如果 `max_wal_senders` 或连接 slot 不足，walsender backend 建立会在更早的 process 初始化或 walsender slot 初始化处失败。
这类问题不是历史不匹配。
它们阻止 replication connection 成立。
### replication startup parameter 错误
startup packet 中 `replication` 参数必须是 bool 或 `database`。
非法值在 `ProcessStartupPacket()` 中 FATAL。
这类错误发生在 walsender 还没进入复制命令循环前。
如果客户端没有设置 replication 参数，却直接发 `IDENTIFY_SYSTEM`，普通 SQL parser 不认识这条命令。
如果物理 walsender 连接上发普通 SQL，`exec_replication_command()` 会返回 false，随后因为没有数据库上下文而报：
```text
cannot execute SQL commands in WAL sender for physical replication
```
### `IDENTIFY_SYSTEM` 返回异常
`libpqrcv_identify_system()` 要求：
```text
result status = PGRES_TUPLES_OK
rows = 1
fields >= 3
```
否则报 protocol violation。
这通常说明上游不是兼容的 PostgreSQL replication endpoint，或者连接被中间层错误代理。
它和 sysid mismatch 不一样。
sysid mismatch 是远端正常返回身份，但本地拒绝。
protocol violation 是远端响应本身不符合复制协议预期。
### system identifier mismatch
这是本节最典型的失败路径。
`WalReceiverMain()` 中的错误是：
```text
database system identifier differs between the primary and standby
```
detail 会打印两个 identifier。
处理这个错误时，不要从 slot、timeline 或 LSN 先查。
先确认：
```text
standby 数据目录是否来自这个 primary 的 base backup；
primary_conninfo 是否指向了正确实例；
DNS、VIP、端口转发或容器服务名是否把 standby 带到了另一个 cluster；
测试环境是否克隆了配置文件但重新 initdb 了数据目录。
```
只要 sysid 不同，physical replication 就不能继续。
手工改配置、改 slot、改 timeline 都不能让另一个 cluster 的 WAL 变成本 cluster 的 WAL。
### primary timeline 落后
错误：
```text
highest timeline X of the primary is behind recovery timeline Y
```
这说明 standby 需要的 `startpointTLI` 比上游报告的 highest timeline 更高。
常见原因：
```text
standby 已经跟随过新的 primary，但又被指回旧 primary；
故障切换后 primary_conninfo 没有更新；
级联上游没有拿到新的 timeline history；
recovery target timeline 配置和实际上游历史不匹配。
```
这个错误发生在 standby 侧，`START_REPLICATION` 之前。
它是 `IDENTIFY_SYSTEM` 之后的 timeline sanity check。
### 请求 timeline 不在上游历史中
如果 standby 发送 `START_REPLICATION ... TIMELINE tli`，walsender 会用 `readTimeLineHistory()` 和 `tliSwitchPoint()` 检查。
错误可能表现为：
```text
requested timeline ... is not in this server's history
requested starting point ... on timeline ... is not in this server's history
```
这类错误说明 sysid 可能相同，但历史分支不匹配。
system identifier 只能证明 lineage。
timeline history 才能证明当前请求点是否位于上游可供应的历史上。
### 请求 LSN 超过上游 flush position
错误：
```text
requested starting point ... is ahead of the WAL flush position of this server ...
```
这说明 standby 请求了一个上游还没有 durable 的位置。
这可能来自错误的 startpoint、错误的 timeline、错误上游，或某些测试中手工构造了未来 LSN。
它不是网络慢。
网络慢会导致连接、receive 或 timeout 问题。
这里是服务端在开始复制前拒绝了一个不可能安全供应的起点。
### physical slot 类型错误
如果 `START_REPLICATION SLOT slotname ...` 指向 logical slot，walsender 报：
```text
cannot use a logical replication slot for physical replication
```
这说明 replication connection 和 sysid 都可能正确，但 slot 类型不匹配。
slot 是保留资源的命名边界。
它不能把 logical decoding slot 当作 physical WAL streaming slot 使用。
## 10. 成本、资源与跨模块传播
握手本身不是 CPU hot path。
它只发生在连接、重连、timeline 切换或 walreceiver restart 时。
但它的失败会把资源压力传播到其它模块。
### connection churn
如果 `primary_conninfo` 错误、权限错误或 sysid mismatch，walreceiver 会反复启动、连接、失败、退出。
startup process 仍然需要 WAL。
如果 archive 也不能提供缺失 WAL，recovery 不能推进。
这类压力表现为：
```text
日志重复出现连接或握手错误；
pg_stat_wal_receiver 短暂出现或为空；
startup process 等待 WAL；
standby replay_lsn 不前进。
```
### timeline history 文件
timeline 检查依赖 `.history` 文件。
walreceiver 会主动 fetch 缺失 history files。
如果上游、archive 或本地 `pg_wal` 中缺失关键 history file，standby 可能无法判断请求点所属历史。
这会从复制连接问题传播到 recovery target timeline 问题。
### slot 和 WAL 保留
握手阶段可以 acquire physical slot。
slot 类型错误会立即失败。
但 slot 是否足以保留 standby 需要的旧 WAL，不在 START_REPLICATION 入口一次性证明。
如果 standby 请求太旧的位置，而上游已经回收对应 segment，后续读 WAL 会失败。
这会表现为 WAL segment missing，而不是 sysid mismatch。
### max_wal_senders
`max_wal_senders` 同时影响：
```text
walsender process slot
WalSnd shared-memory array
pg_stat_replication 可见连接数
primary 允许多少 standby / backup / receivewal 同时连接
```
如果这个资源耗尽，standby 连不上 replication endpoint。
不要把它误判成 timeline 或 sysid 问题。
### flush position 边界
walsender 只从 flush 位置以内开始发送。
这把复制握手和 WAL persistence 连接起来。
如果 workload 产生 WAL 很快，但 WAL flush 被 I/O 卡住，新的 standby 请求未来位置会被拒绝。
更常见的影响在进入 streaming 后：walsender 可发送位置受 flush 推进限制。
这属于后续 walsender 主循环课程。
## 11. 观测与诊断入口
### 直接看 cluster identity
最直接的离线工具是：
```bash
pg_controldata $PGDATA | rg 'Database system identifier'
```
primary 和 standby 的值必须一致。
如果不一致，physical replication 不应该继续排查 slot 或 timeline。
在服务端运行 `IDENTIFY_SYSTEM` 可以看到上游对外暴露的 systemid。
实际 walreceiver 使用的是 `replication=true` 物理连接。
人工排查时，可以用支持 replication connection 的客户端或工具。
如果用 `psql` 做实验，常见做法是连接 `replication=database`，这样会有数据库上下文，也能执行 replication commands。
但要记住：
```text
replication=database 的 dbname 列可能非 NULL；
真实物理 walreceiver 使用 replication=true，通常不连接具体数据库。
```
不要把实验连接的 `dbname` 列差异误解为物理 standby 行为差异。
### 观察 standby 侧
`pg_stat_wal_receiver` 来自 `pg_stat_get_wal_receiver()`。
它能看到：
```text
status
receive_start_lsn
receive_start_tli
written_lsn
flushed_lsn
received_tli
latest_end_lsn
slot_name
sender_host
sender_port
conninfo
```
这些字段对应 `WalRcvData`。
它不能直接看到：
```text
上一次 IDENTIFY_SYSTEM 返回的 remote systemid；
START_REPLICATION 命令文本；
walsender 在 timeline history 检查中读到的 switchpoint。
```
所以 sysid mismatch 主要看 server log。
`pg_stat_wal_receiver` 更适合确认 walreceiver 是否已经进入 streaming，以及它接收、写入、flush 到哪里。
### 观察 primary 侧
`pg_stat_replication` 的底层来自 `WalSnd`。
握手刚开始时，你可能看到 state 为 `startup`。
`StartReplication()` 接受请求并进入流式复制后，通常先进入 `catchup`。
后续追上后才可能进入 `streaming`。
这里能看到：
```text
pid
state
sent_lsn
write_lsn
flush_lsn
replay_lsn
sync_state
reply_time
```
它不能看到：
```text
standby 本地 system_identifier；
standby 发送 START_REPLICATION 前的 IDENTIFY_SYSTEM 校验是否刚刚通过；
timeline history 文件 fetch 是否完整。
```
如果连接失败在 `IDENTIFY_SYSTEM` 前或 sysid mismatch 后，`pg_stat_replication` 可能只短暂出现，或者根本来不及被人工查询到。
### 日志入口
几个日志点非常有用：
```text
replication connection authorized: user=...
received replication command: IDENTIFY_SYSTEM
received replication command: START_REPLICATION ...
started streaming WAL from primary at ... on timeline ...
database system identifier differs between the primary and standby
highest timeline ... of the primary is behind recovery timeline ...
requested starting point ... is ahead of the WAL flush position ...
```
`received replication command` 受 `log_replication_commands` 控制。
关闭时，源码仍可能以 DEBUG1 级别记录，用于兼容。
生产诊断时，如果怀疑握手阶段失败，可以短时间打开：
```text
log_replication_commands = on
```
然后观察上游是否收到 `IDENTIFY_SYSTEM` 和 `START_REPLICATION`。
如果只看到连接授权，没有看到 `IDENTIFY_SYSTEM`，问题可能卡在 walreceiver 连接、权限、协议或进程启动阶段。
如果看到 `IDENTIFY_SYSTEM` 后 standby 报 sysid mismatch，问题已经进入本节核心校验。
如果看到 `START_REPLICATION` 后失败，优先看 timeline、slot 和 LSN。
### wait event
walreceiver libpq 连接和接收路径使用这些 wait event 常量：
```text
WAIT_EVENT_LIBPQWALRECEIVER_CONNECT
WAIT_EVENT_LIBPQWALRECEIVER_RECEIVE
WAIT_EVENT_WAL_RECEIVER_MAIN
```
它们能说明进程当前等在哪里。
它们不能说明 systemid 是否匹配。
wait event 是瞬时等待位置，不是完整因果。
如果 `pg_stat_activity` 或相关视图看到 walreceiver 等在 receive，不代表握手已经完全健康，也不代表没有 timeline 问题。
必须结合日志、`pg_stat_wal_receiver` 和 LSN 推进判断。
### 三类状态可见性
能直接观测：
```text
pg_control 中的 system identifier
pg_stat_wal_receiver 的 receive_start_tli / received_tli / lsn
pg_stat_replication 的 state / sent_lsn / write_lsn / flush_lsn / replay_lsn
server log 中的复制命令和错误文本
```
只能推断：
```text
walreceiver 是否刚刚完成 IDENTIFY_SYSTEM 校验
START_REPLICATION 的历史检查走到哪一步
上游是否因为请求点太旧而将在后续 WAL read 中失败
```
基本不可见或不应依赖：
```text
cmd_context 内部生命周期
replication scanner 的 first-token pushback
StartReplication() 中的局部变量 switchpoint / FlushPtr
```
这些需要源码阅读、DEBUG 日志或断点实验。
## 12. 常见误区
### 误区一：把 system identifier 当作 primary 节点 ID
`system_identifier` 是 cluster lineage ID。
同一个物理复制拓扑中的 primary、standby、级联 standby 通常都相同。
promotion 不会生成新的 system identifier。
promotion 会生成新的 timeline。
所以它不能用来区分当前哪个节点是 primary。
### 误区二：认为 timeline 相同就一定是同一集群
timeline ID 是小整数。
不同 `initdb` 出来的集群都可能有 timeline 1。
timeline 必须和 system identifier 一起解释。
没有 sysid 校验，timeline 没有跨 cluster 的全局意义。
### 误区三：认为 IDENTIFY_SYSTEM 成功就已经开始复制
`IDENTIFY_SYSTEM` 只返回身份、timeline 和位置。
真正进入 streaming 要等 `START_REPLICATION` 返回 `PGRES_COPY_BOTH`。
在这两者之间，standby 还可能因为 sysid mismatch、timeline 落后、history file 问题或 slot 创建失败而停下。
### 误区四：认为 START_REPLICATION 只需要 LSN
物理恢复需要 `LSN + timeline`。
同一个 LSN 在不同 timeline 上可能代表不同历史。
walreceiver 的 `receiveStart` 和 `receiveStartTLI` 必须一起看。
### 误区五：把 pg_stat_replication 当作握手审计日志
`pg_stat_replication` 是当前 walsender shared state。
它不是每次复制命令的历史日志。
短暂失败的握手可能不会长期留在统计视图里。
诊断握手失败要看日志和 standby 侧错误。
## 13. 课堂实验
### 实验一：观察一次正常握手
准备一个 primary 和由它 base backup 出来的 standby。
在 primary 上临时打开：
```text
log_replication_commands = on
```
重启或 reload 后，让 standby 重新连接。
观察 primary 日志中是否出现：
```text
replication connection authorized: user=...
received replication command: IDENTIFY_SYSTEM
received replication command: START_REPLICATION ...
```
在 standby 上观察：
```sql
SELECT status,
       receive_start_lsn,
       receive_start_tli,
       written_lsn,
       flushed_lsn,
       received_tli,
       latest_end_lsn,
       slot_name,
       sender_host,
       sender_port
FROM pg_stat_wal_receiver;
```
把 `receive_start_tli` 和 primary 日志中的 `START_REPLICATION ... TIMELINE ...` 对上。
然后回到源码：
```text
WalReceiverMain()
  -> walrcv_identify_system()
  -> GetSystemIdentifier() 比较
  -> walrcv_startstreaming()
exec_replication_command()
  -> IdentifySystem()
  -> StartReplication()
```
实验目标不是看延迟。
实验目标是把日志中的两条 replication command 对回源码路径。
### 实验二：制造 system identifier mismatch
在隔离测试环境中，准备两个独立 `initdb` 出来的 primary A 和 primary B。
不要在生产环境做这个实验。
把从 A base backup 出来的 standby 的 `primary_conninfo` 指向 B。
启动 standby。
预期 walreceiver 报：
```text
database system identifier differs between the primary and standby
```
记录 detail 中两个 identifier。
再分别在 A、B、standby 上用 `pg_controldata` 查看 `Database system identifier`。
回到源码确认错误来自：
```text
WalReceiverMain()
  -> primary_sysid = walrcv_identify_system(...)
  -> standby_sysid = GetSystemIdentifier()
  -> strcmp(primary_sysid, standby_sysid) != 0
```
实验目标是建立一个诊断规则：
```text
只要 sysid mismatch，优先修正数据目录来源或连接目标；
不要先调 slot、wal_keep_size、timeline 或 timeout。
```
## 14. 讨论题
1. 为什么 `IDENTIFY_SYSTEM` 不让 standby 把自己的 system identifier 发给 primary，由 primary 来判断是否接受？
2. 为什么 `replication=true` 的物理 walsender 要清空数据库名，而 `replication=database` 又必须连接具体数据库？
3. 如果两个节点 `system_identifier` 相同，但 `START_REPLICATION ... TIMELINE ...` 报 history mismatch，应该优先怀疑什么？
4. 为什么 `START_REPLICATION` 的请求必须包含 LSN 和 timeline，而不能只用 primary 当前最新 LSN？
5. `pg_stat_replication` 中看不到某次 `IDENTIFY_SYSTEM` 的结果，这是不是统计视图缺陷？为什么？
6. `FlushPtr < cmd->startpoint` 的拒绝和 crash safety 有什么关系？
## 15. 本节小结
本节只围绕一个问题：
```text
standby 如何确认自己正在追随同一个集群历史？
```
答案不是一个单点校验。
PostgreSQL 把它拆在连接、命令和恢复状态之间。
`replication=true` 在 startup packet 阶段把服务端 backend 切成 walsender。
`IDENTIFY_SYSTEM` 让 walsender 暴露 `system_identifier`、当前 timeline 和 flush 位置。
walreceiver 用本地 `GetSystemIdentifier()` 比对远端 systemid，并确认远端 highest timeline 不落后于本地请求 timeline。
`START_REPLICATION` 再把本地 `receiveStart + receiveStartTLI` 发给 walsender。
walsender 负责检查 slot 类型、timeline history、请求点是否在历史上，以及请求 LSN 是否不超过本机 flush position。
核心状态边界是：
```text
ControlFile->system_identifier:
  cluster lineage identity。
WalRcvData.receiveStart / receiveStartTLI:
  standby startup process 要求 walreceiver 拉取的历史位置。
StartReplicationCmd.startpoint / timeline:
  网络协议中表达的请求位置。
WalSnd.sendTimeLine / sentPtr / state:
  primary 侧接受请求后用于发送和观测的状态。
```
ownership 和 cleanup 也有明确分工。
walreceiver 失败由 `WalRcvDie()` 清理 shared state、断开连接并唤醒 startup process。
walsender 失败由顶层错误恢复、`WalSndErrorCleanup()`、slot release 和 `WalSndKill()` 兜底。
观测上，`pg_controldata` 和日志最适合确认 sysid 问题。
`pg_stat_wal_receiver` 和 `pg_stat_replication` 适合看握手后是否进入 streaming，以及 LSN 是否推进。
它们不能替代握手错误日志。
可迁移的系统规律是：
```text
跨进程、跨机器传输持久化历史时，不要把“连接成功”当作“历史连续”。
必须把身份、分支、位置和持久化边界拆开校验，并明确每一端负责检查哪一部分。
```
本节建立的是 physical streaming 的入口正确性。
下一节沿着已经进入 COPY BOTH 的连接，继续看 walsender 如何读取 WAL、等待新 WAL、发送 CopyData，并推进 sent position。
