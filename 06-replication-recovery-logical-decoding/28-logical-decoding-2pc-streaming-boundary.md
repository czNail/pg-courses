# PostgreSQL logical decoding 2PC 与 streaming 边界
## 课程定位
前置知识：已经理解 logical slot 通过 `restart_lsn` 和 `catalog_xmin` 保留历史，也已经理解 `decode.c` 把 WAL record 转成 `ReorderBufferChange`，`reorderbuffer.c` 默认等 commit record 到达后再按事务输出。
本节唯一主问题：
```text
prepared transaction 和 in-progress 大事务为什么需要特殊解码协议，
streaming mode 如何降低 apply 端等待整事务提交的压力？
```
核心矛盾：
```text
logical decoding 必须保持事务原子性、提交顺序和历史 catalog 解释正确；
但 PREPARE TRANSACTION 会把事务停在“已持久、未最终提交”的状态，
而大事务在最终 COMMIT 前可能已经产生大量可解析 change。
```
如果仍坚持“只在最终 commit 时输出完整事务”，prepared transaction 会把下游挡在 `COMMIT PREPARED` 之后，无法表达 prepare/rollback prepared 边界；大事务会让 reorder buffer、walsender 和 subscriber apply 都等待同一个巨大事务完成。
PostgreSQL 的解法是把“事务最终可见”拆成更细的协议状态：
```text
普通事务:
  BEGIN -> change -> COMMIT

prepared transaction:
  BEGIN PREPARE -> change -> PREPARE
  后续再 COMMIT PREPARED 或 ROLLBACK PREPARED

streaming large in-progress transaction:
  STREAM START -> stream_change -> STREAM STOP
  可以重复多个 chunk
  最终 STREAM COMMIT / STREAM ABORT
  如果同时是 2PC，则最终可以 STREAM PREPARE
```
学完后应能判断：`two_phase` slot option 解决的是哪个边界；为什么 `XLOG_XACT_PREPARE` 不能被当作普通 commit；为什么 `stream_start/stream_change/stream_commit/stream_abort` 不是普通 row message 的别名；为什么 streaming 可以降低 apply 端峰值等待和峰值内存，但不能绕过事务原子性；为什么 `confirmed_flush` 与 `restart_lsn` 推进仍受客户端确认和未完成事务约束。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
当前 commit 下，logical replication protocol 的能力版本定义在 `src/include/replication/logicalproto.h`：
```text
LOGICALREP_PROTO_STREAM_VERSION_NUM = 2
LOGICALREP_PROTO_TWOPHASE_VERSION_NUM = 3
LOGICALREP_PROTO_STREAM_PARALLEL_VERSION_NUM = 4
```
旧资料如果只说 `proto_version = 1`，要按当前源码修正。
## 1. 本节在总主线中的位置
前面几节的主线是：
```text
WAL record
  -> LogicalDecodingProcessRecord()
  -> ReorderBufferQueueChange()
  -> ReorderBufferCommit()
  -> ReorderBufferProcessTXN()
  -> output plugin
```
这个模型默认有一个简单边界：
```text
commit record 到达之前只收集；
commit record 到达之后才输出。
```
这对普通事务成立。
但两个场景会让这个边界不够用。
第一个场景是 two-phase commit。
SQL 侧时间线是：
```text
BEGIN
INSERT / UPDATE / DELETE
PREPARE TRANSACTION 'gid'
-- 事务已经 prepared，锁和状态还在
COMMIT PREPARED 'gid'
或
ROLLBACK PREPARED 'gid'
```
`PREPARE TRANSACTION` 不是普通 abort，也不是普通 commit。
它把事务状态写入 2PC WAL 和两阶段状态，事务仍以 prepared xact 的身份存在。
下游如果要参与分布式 2PC，必须看到 prepare 边界。
只在 `COMMIT PREPARED` 时输出普通事务，会丢掉“远端也应先 prepare”的语义。
第二个场景是 large in-progress transaction。
SQL 侧时间线可能是：
```text
BEGIN
写入 100GB changes
等待业务逻辑或外部锁
COMMIT
```
默认 commit-time 输出会让 publisher 的 decoding backend 持有或 spill 大量 decoded change。
subscriber apply 也只能在最后收到完整事务后开始处理。
如果启用 streaming，publisher 可以在事务尚未提交时按 chunk 发送已经解码的 change。
subscriber 可以提前接收、暂存甚至并行处理这些 chunk。
最终 `STREAM COMMIT` 才让下游把这些 change 作为事务提交。
所以本节不是重新讲 reorder buffer。
本节只问一个问题：
```text
当“事务边界”不再只有 COMMIT/ABORT 两种终点时，
logical decoding 怎样把内部状态和外部协议都拆细，
同时不破坏原子性、LSN 确认和错误恢复？
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
decode.c 识别 XLOG_XACT_PREPARE / COMMIT_PREPARED / ABORT_PREPARED，
reorderbuffer.c 用 RBTXN_IS_PREPARED / RBTXN_IS_STREAMED 等 flags 区分事务生命周期，
logical.c 把 reorder buffer 事件包装成 output plugin callbacks，
pgoutput.c 和 proto.c 再把它们编码成 2PC 或 streaming protocol messages。
```
普通事务的状态机很短：
```text
collect changes
  -> commit record
  -> begin/change/commit callbacks
  -> cleanup
```
prepared transaction 的状态机多了一段“已经解码，但不能 cleanup”的区间：
```text
collect changes
  -> prepare record
  -> begin_prepare/change/prepare callbacks
  -> truncate decoded changes, 保留 txn skeleton / invalidation / prepare info
  -> commit prepared record
  -> commit_prepared callback
  -> cleanup
```
如果最终 rollback prepared：
```text
prepare 已经可能发给下游
  -> rollback prepared record
  -> rollback_prepared callback
  -> cleanup
```
streaming 大事务的状态机则把“change 输出”提前到 commit 之前：
```text
collect changes
  -> memory threshold 或 debug immediate
  -> stream_start/stream_change/stream_stop
  -> truncate already streamed changes, 保留 snapshot / command_id / invalidation
  -> 后续继续 collect
  -> commit record
  -> stream remaining changes
  -> stream_commit
  -> cleanup
```
如果该 streaming 事务 abort：
```text
已经 stream 给下游的 change
  -> stream_abort
  -> 下游丢弃该 xid 的暂存内容
```
这个模型里的 tension 是：
```text
提前发送可以降低等待和内存压力
  vs
提前发送不能让下游提前提交，也不能让 slot 过早忘记重启解码所需 WAL。
```
所以 streaming 是“提前传输和预处理”，不是“提前提交”。
2PC 是“把 prepare 变成协议级事务状态”，不是“把 prepared xact 当成 committed xact”。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/reorderbuffer.h` | `RBTXN_IS_PREPARED`、`RBTXN_IS_STREAMED`、`RBTXN_SENT_PREPARE`、callback 类型和 `ReorderBufferTXN` 状态。 |
| 2 | `src/backend/replication/logical/decode.c` | `xact_decode()` 如何处理 `XLOG_XACT_PREPARE`、`XLOG_XACT_COMMIT_PREPARED`、`XLOG_XACT_ABORT_PREPARED`。 |
| 3 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferPrepare()`、`ReorderBufferFinishPrepared()`、`ReorderBufferStreamTXN()`、`ReorderBufferStreamCommit()`、`ReorderBufferProcessTXN()`。 |
| 4 | `src/backend/replication/logical/logical.c` | `StartupDecodingContext()` 如何接 callback；`begin_prepare_cb_wrapper()`、`stream_*_cb_wrapper()` 如何设置 `write_location` 和错误上下文。 |
| 5 | `src/backend/replication/pgoutput/pgoutput.c` | `pgoutput_startup()` 如何校验 protocol capability；`pgoutput_*prepared*` 和 `pgoutput_stream_*` 如何编码消息。 |
| 6 | `src/backend/replication/logical/proto.c` | `logicalrep_write_begin_prepare()`、`logicalrep_write_prepare()`、`logicalrep_write_stream_start()`、`logicalrep_write_stream_commit()` 等 wire message。 |
| 7 | `src/backend/replication/walsender.c` | `CREATE_REPLICATION_SLOT ... TWO_PHASE`、`ALTER_REPLICATION_SLOT`、`StartLogicalReplication()`、flush feedback 如何推进 slot。 |
| 8 | `src/backend/replication/slot.c` / `src/include/replication/slot.h` | `two_phase`、`two_phase_at`、`confirmed_flush`、`restart_lsn` 的持久状态。 |
| 9 | `src/backend/access/transam/twophase.c` | `PREPARE TRANSACTION` 与 `COMMIT/ROLLBACK PREPARED` 如何写 WAL 和保持 prepared xact identity。 |
| 10 | `src/backend/access/transam/xact.c` | `XactLogCommitRecord()` / `XactLogAbortRecord()` 产生普通和 prepared commit/abort record。 |
推荐阅读顺序：
```text
先读 logicalproto.h 的 capability
  -> 看 pgoutput_startup 如何拒绝不兼容选项
  -> 看 decode.c 识别 prepare/commit prepared/rollback prepared
  -> 看 reorderbuffer.c 如何改变 TXN flags 和 callbacks
  -> 看 logical.c wrappers 如何设置 write_lsn/write_xid
  -> 最后回到 walsender/slot.c 看 confirmed_flush/restart_lsn
```
不要从 `twophase.c` 开始读。
`twophase.c` 解释数据库本地 2PC 状态。
本节关心的是 logical decoding 如何把这些状态投影到 replication protocol。
## 4. 为什么普通 commit-time 输出不够
默认 logical decoding 保守得有原因。
如果一个事务还没 commit，外部 consumer 不能把它当成可见数据。
`ReorderBufferQueueChange()` 只把 decoded change 挂到 `ReorderBufferTXN`。
`DecodeCommit()` 到达后才调用 `ReorderBufferCommit()`。
这保证：
```text
abort 事务不会输出普通 change；
subtransaction abort 的 change 不会混入 parent；
catalog lookup 在 historic snapshot 下执行；
output plugin 看到完整事务边界。
```
但 prepared transaction 的问题不是“事务太大”。
它的问题是多了一个稳定中间态。
`PREPARE TRANSACTION` 之后，事务状态已经持久化。
崩溃恢复后它仍要存在。
它持有的锁、XID 语义和 2PC state 不能按普通 backend transaction 清理。
如果 logical replication 只在 `COMMIT PREPARED` 输出普通 `BEGIN/COMMIT`，下游无法知道上游曾经进入 prepared state。
分布式系统会失去一个重要不变量：
```text
参与者必须先都能 prepare，
协调者才能决定最终 commit prepared。
```
large in-progress transaction 的问题也不是“事务语义改变”。
它的问题是资源和等待被压到最终 commit 点。
默认路径要求：
```text
publisher 解码端在 commit 前保存 decoded changes；
walsender 在 commit 后一次性输出；
subscriber apply 在收到完整事务后才开始真正应用或提交；
slot 的 confirmed_flush 只能跟随客户端 flush feedback 推进。
```
当事务很大时，这会把内存、磁盘 spill、网络发送、apply CPU 和临时文件压力集中到最后。
streaming mode 把传输和 apply 准备前移。
但它必须保留最终原子性：
```text
streamed chunks 只是该 xid 的 provisional data；
STREAM COMMIT 才能让它们成为已提交事务；
STREAM ABORT 必须能丢弃已经收到的 chunks。
```
所以两种特殊协议分别解决两个不同边界：
```text
2PC:
  表达 prepared state。

streaming:
  拆分大事务传输和解码压力。
```
它们可以叠加。
一个大事务可以先 stream 多个 chunk，后来进入 `PREPARE TRANSACTION`，这时协议需要 `STREAM PREPARE`。
## 5. 关键状态：slot、context、TXN flags
### `ReplicationSlotPersistentData`
slot 的持久状态在 `src/include/replication/slot.h`。
本节关注四个字段：
| 字段 | 语义 |
| --- | --- |
| `confirmed_flush` | 客户端确认已经 flush 到的位置；逻辑解码重启时默认从这里作为输出基线。 |
| `restart_lsn` | 为了重新解码尚未安全丢弃的事务，slot 必须保留 WAL 的最早位置。 |
| `two_phase` | 是否允许这个 logical slot 解码 prepared transaction。 |
| `two_phase_at` | 从哪个 LSN 开始 two-phase decoding 被认为安全启用。 |
`two_phase_at` 不是装饰字段。
它解决的是启用 two-phase 中途重启的边界：
```text
如果某个 prepared transaction 的 prepare 发生在 two_phase 启用之前，
decoder 不能假装已经把 PREPARE 发给了下游。
```
`ReorderBufferFinishPrepared()` 会比较 `txn->final_lsn` 和 `two_phase_at`。
必要时在 `COMMIT PREPARED` 上补发 prepare 阶段，避免下游只收到 commit prepared 而没有对应 prepared state。
### `LogicalDecodingContext`
`src/include/replication/logical.h` 里有两个开关：
```text
ctx->streaming
ctx->twophase
```
这两个值都不是单靠用户选项决定。
它们是三层条件的交集：
```text
slot 是否允许；
output plugin 是否提供 callbacks；
START_REPLICATION / pgoutput options 是否请求；
protocol version 是否支持。
```
`StartupDecodingContext()` 先根据 plugin callbacks 推断支持能力。
如果 plugin 至少提供一个 streaming callback，`ctx->streaming` 会先被认为可能支持。
如果 plugin 至少提供一个 two-phase callback，`ctx->twophase` 会先被认为可能支持。
随后 `pgoutput_startup()` 会按客户端 option 和 protocol version 再收窄。
最后 `CreateDecodingContext()` 会执行：
```text
ctx->twophase &= (slot->data.two_phase || ctx->twophase_opt_given)
```
如果 start streaming 时请求 two_phase，而 slot 还没标记，则会写：
```text
slot->data.two_phase = true
slot->data.two_phase_at = start_lsn
SnapBuildSetTwoPhaseAt(ctx->snapshot_builder, start_lsn)
```
这说明 two-phase decoding 不是一个无状态输出选项。
它会改变 slot 持久边界。
### `ReorderBufferTXN`
`ReorderBufferTXN` 仍然是 backend-local 对象。
它不跨进程共享。
本节最重要的字段组合：
| 字段 | 语义 |
| --- | --- |
| `xid`、`toplevel_xid`、`toptxn` | 当前事务和 subtransaction 关系。 |
| `gid` | prepared transaction 的 global transaction id。 |
| `first_lsn` | 该事务第一次相关 logical record 的位置。 |
| `final_lsn` | prepare / commit / abort record 的起点，也会被 spill 更新。 |
| `end_lsn` | prepare / commit / abort record 的结束位置。 |
| `prepare_time` / `commit_time` / `abort_time` | 不同终点的时间字段共用 union。 |
| `snapshot_now`、`command_id` | streaming 多次运行之间保存 historic snapshot 进度。 |
| `ninvalidations`、`invalidations` | commit、prepare、rollback 后必须执行的 cache invalidation。 |
| `txn_flags` | prepared、streamed、sent prepare、aborted、committed 等生命周期状态。 |
关键 flags 在 `reorderbuffer.h`：
```text
RBTXN_IS_PREPARED
RBTXN_SKIPPED_PREPARE
RBTXN_SENT_PREPARE
RBTXN_IS_STREAMED
RBTXN_HAS_STREAMABLE_CHANGE
RBTXN_IS_ABORTED
RBTXN_IS_COMMITTED
```
这些 flags 不能单独解释。
例如：
```text
RBTXN_IS_PREPARED:
  说明 prepare info 已经读到，后续应走 prepare/commit_prepared 语义。

RBTXN_SENT_PREPARE:
  说明 prepare 或 stream_prepare 已经发给下游。

RBTXN_SKIPPED_PREPARE:
  说明当前 decoder 记住了 prepare info，但当时没有输出 prepare。
```
三者组合才说明下游是否已经知道该 xid prepared。
同样，`RBTXN_IS_STREAMED` 不是“事务已提交”。
它只说明至少有一部分 change 已通过 streaming callback 发给下游。
最终还要看 `stream_commit` 或 `stream_abort`。
## 6. slot option 与协议能力先决定能不能走特殊路径
`CREATE_REPLICATION_SLOT` 的 replication command option 在 `walsender.c` 中解析。
`parseCreateReplSlotOptions()` 接受 logical slot 的：
```text
two_phase
failover
snapshot
```
当 `CREATE_REPLICATION_SLOT ... LOGICAL ... (TWO_PHASE true)` 进入 `CreateReplicationSlot()` 时，最终调用：
```text
ReplicationSlotCreate(..., two_phase, ...)
```
`slot.c` 初始化：
```text
slot->data.two_phase = two_phase
slot->data.two_phase_at = InvalidXLogRecPtr
```
如果是创建新 logical slot，`DecodingContextFindStartpoint()` 找到 consistent point 后会写：
```text
slot->data.confirmed_flush = ctx->reader->EndRecPtr
if (slot->data.two_phase)
    slot->data.two_phase_at = ctx->reader->EndRecPtr
```
这意味着：
```text
创建时启用 two_phase:
  从初始 consistent point 之后，prepare record 可以被认为安全解码。
```
`ALTER_REPLICATION_SLOT` 也能改变 `two_phase`。
`slot.c` 的注释很谨慎：
```text
随意启用 two_phase 可能让启用前已 prepared 的事务缺失 prepare 记录；
随意禁用 two_phase 时如果仍有 pending prepared transaction，
可能导致已 prepared 的 change 在 commit 时重复复制。
```
所以 subscription 侧不会在任意时刻打开。
`worker.c` 的注释说明，subscription 的 two_phase 状态会等初始 table sync 完成后才启用。
streaming 不是 slot 持久属性。
它是 output plugin session option。
`pgoutput_startup()` 解析：
```text
proto_version
publication_names
streaming
two_phase
```
然后按 protocol version 校验：
```text
streaming = on:
  proto_version >= LOGICALREP_PROTO_STREAM_VERSION_NUM

streaming = parallel:
  proto_version >= LOGICALREP_PROTO_STREAM_PARALLEL_VERSION_NUM

two_phase = true:
  proto_version >= LOGICALREP_PROTO_TWOPHASE_VERSION_NUM
```
如果客户端请求 streaming，但 plugin 不支持 streaming callbacks，会报错。
如果客户端请求 two_phase，但 plugin 不支持 two-phase callbacks，也会报错。
`pgoutput` 自己在 `_PG_output_plugin_init()` 中注册了完整 callback：
```text
begin_prepare_cb
prepare_cb
commit_prepared_cb
rollback_prepared_cb
stream_start_cb
stream_stop_cb
stream_abort_cb
stream_commit_cb
stream_change_cb
stream_message_cb
stream_truncate_cb
stream_prepare_cb
```
因此内置 logical replication 能同时支持 2PC 和 streaming。
第三方 output plugin 不能只设置一个 callback 就假装完整支持。
`logical.c` 的 wrapper 会在实际调用时检查 mandatory callback，缺失就报：
```text
logical streaming requires a stream_start_cb callback
logical replication at prepare time requires a prepare_cb callback
```
这是故意延迟检查。
`StartupDecodingContext()` 先用“是否出现过相关 callback”推断能力，再在 wrapper 中给出更具体错误。
## 7. PREPARE 写入端：prepared xact 是真实持久状态
`twophase.c` 的 `PREPARE TRANSACTION` 路径不是 output plugin 的发明。
本地数据库首先要把 2PC state 持久化。
关键路径在 `src/backend/access/transam/twophase.c`：
```text
EndPrepare()
  -> assemble 2PC state records
  -> START_CRIT_SECTION()
  -> MyProc->delayChkptFlags |= DELAY_CHKPT_START
  -> XLogInsert(RM_XACT_ID, XLOG_XACT_PREPARE)
  -> XLogFlush(gxact->prepare_end_lsn)
  -> gxact->prepare_start_lsn = ProcLastRecPtr
  -> MarkAsPrepared(gxact, false)
  -> END_CRIT_SECTION()
  -> SyncRepWaitForLSN(gxact->prepare_end_lsn, false)
```
这里有一个关键注释：
```text
MarkAsPrepared() 会让 prepared XID 有 dummy ProcArray entry；
它必须发生在 MyProc / ProcGlobal->xids[] 清掉该 XID 之前。
```
否则会出现窗口：
```text
TransactionIdIsInProgress(xid) 看不到该 XID
其它 backend 可能把它当成 crashed/finished
```
PostgreSQL 宁愿短暂让同一个 XID 在 ProcArray 中出现两次，也不能让它短暂消失。
这解释了为什么 prepared transaction 不是普通 backend transaction 的尾声。
它转移到了 `TwoPhaseState` 和 prepared dummy PGPROC 代表的状态。
`COMMIT PREPARED` / `ROLLBACK PREPARED` 进入：
```text
FinishPreparedTransaction(gid, isCommit)
  -> LockGXact()
  -> 读 2PC state
  -> RecordTransactionCommitPrepared()
     或 RecordTransactionAbortPrepared()
  -> pg_xact 标记 commit/abort
  -> 从 ProcArray 移除 prepared xact
  -> post-commit/post-abort callbacks 释放锁等资源
```
`RecordTransactionCommitPrepared()` 总是写 commit WAL。
注释说明 prepared xact 已经写过 PREPARE，因此不可能优化掉 commit record。
logical decoding 依赖这些 WAL record：
```text
XLOG_XACT_PREPARE
XLOG_XACT_COMMIT_PREPARED
XLOG_XACT_ABORT_PREPARED
```
这不是协议层模拟。
它直接对应本地 transaction manager 的持久状态转移。
## 8. `xact_decode()` 如何分辨 prepare / commit prepared / rollback prepared
入口仍然是 `LogicalDecodingProcessRecord()`。
当 rmgr 是 `RM_XACT_ID`，进入 `xact_decode()`。
它先要求：
```text
SnapBuildCurrentState(builder) >= SNAPBUILD_FULL_SNAPSHOT
```
没有 full snapshot 时，不解码 xact record。
commit 分支包括：
```text
XLOG_XACT_COMMIT
XLOG_XACT_COMMIT_PREPARED
```
它调用：
```text
ParseCommitRecord(...)
```
然后决定 xid：
```text
parsed.twophase_xid valid ? parsed.twophase_xid : XLogRecGetXid(record)
```
这点很重要。
prepared commit record 的 record XID 语义和普通 commit 不完全一样。
解码要使用 parsed record 中的 `twophase_xid`。
如果 record 是 `XLOG_XACT_COMMIT_PREPARED`，`xact_decode()` 会调用：
```text
FilterPrepare(ctx, xid, parsed.twophase_gid)
```
如果没有被过滤，`two_phase = true`。
随后进入：
```text
DecodeCommit(ctx, buf, &parsed, xid, two_phase)
```
abort 分支类似：
```text
XLOG_XACT_ABORT
XLOG_XACT_ABORT_PREPARED
ParseAbortRecord(...)
FilterPrepare(...)
DecodeAbort(..., two_phase)
```
prepare 分支是：
```text
XLOG_XACT_PREPARE
ParsePrepareRecord(...)
FilterPrepare(...)
DecodePrepare(...)
```
如果 `FilterPrepare()` 返回 true，说明插件决定不按 prepare-time 方式处理该事务。
此时只执行：
```text
ReorderBufferProcessXid(reorder, parsed.twophase_xid, buf->origptr)
```
并跳过 `DecodePrepare()`。
这就是 `filter_prepare_cb` 的语义：
```text
不是过滤这个事务的全部 change；
而是决定是否把它拆成 prepare + commit_prepared / rollback_prepared 协议。
```
如果跳过 prepare-time decoding，后续 commit prepared 可以退化成普通 commit-style 输出。
但内置 logical replication 的 two_phase 模式通常需要保留 prepare 边界。
## 9. `DecodePrepare()`：prepare record 到 prepared protocol
`DecodePrepare()` 的职责和 `DecodeCommit()` 相似，但它不能 cleanup 整个事务。
主链路：
```text
DecodePrepare()
  -> ReorderBufferRememberPrepareInfo()
  -> 检查 SnapBuildCurrentState >= SNAPBUILD_CONSISTENT
  -> DecodeTXNNeedSkip()
  -> ReorderBufferCommitChild() for surviving subxacts
  -> ReorderBufferPrepare()
  -> UpdateDecodingStats()
```
第一步是：
```text
ReorderBufferRememberPrepareInfo(ctx->reorder,
                                 xid,
                                 prepare_lsn,
                                 end_lsn,
                                 prepare_time,
                                 origin_id,
                                 origin_lsn)
```
它找到 `ReorderBufferTXN`，写入：
```text
txn->final_lsn = prepare_lsn
txn->end_lsn = end_lsn
txn->prepare_time = prepare_time
txn->origin_id = origin_id
txn->origin_lsn = origin_lsn
txn->txn_flags |= RBTXN_IS_PREPARED
```
如果找不到 txn，说明没有相关 decoded change，直接返回 false。
prepare 记录还会检查：
```text
SnapBuildCurrentState(builder) < SNAPBUILD_CONSISTENT
```
如果还没 consistent，调用：
```text
ReorderBufferSkipPrepare()
```
它只设置 `RBTXN_SKIPPED_PREPARE`。
不能简单 `ReorderBufferForget()`。
源码注释说明：
```text
prepared xact 尚未最终 commit；
过早移除可能让 restart_lsn 计算错误。
```
如果 `DecodeTXNNeedSkip()` 判断事务不需要处理，也不能像 ordinary commit skip 一样直接 forget。
它会：
```text
ReorderBufferSkipPrepare()
ReorderBufferInvalidate()
```
原因仍然是 prepared 事务可能还在运行集合中影响 restart boundary。
同时 cache invalidation 仍要处理，避免 historic catalog 被污染。
通过这些门槛后，prepare 像 commit 一样先处理 surviving subtransactions：
```text
ReorderBufferCommitChild(ctx->reorder, xid, subxid, prepare_lsn, end_lsn)
```
然后调用：
```text
ReorderBufferPrepare(ctx->reorder, xid, parsed->twophase_gid)
```
## 10. `ReorderBufferPrepare()`：输出 prepare，但保留事务骨架
`ReorderBufferPrepare()` 要求：
```text
RBTXN_IS_PREPARED 已设置
RBTXN_SKIPPED_PREPARE 未设置
RBTXN_SENT_PREPARE 未设置
txn->final_lsn 有效
```
然后保存：
```text
txn->gid = pstrdup(gid)
```
接着调用：
```text
ReorderBufferReplay(txn,
                    rb,
                    xid,
                    txn->final_lsn,
                    txn->end_lsn,
                    txn->prepare_time,
                    txn->origin_id,
                    txn->origin_lsn)
```
这里看起来像 commit replay，但 `txn` 已经带 `RBTXN_IS_PREPARED`。
`ReorderBufferProcessTXN()` 因此会走 prepared 分支。
非 streaming prepared 事务会调用：
```text
rb->begin_prepare(rb, txn)
rb->apply_change(...)
rb->prepare(rb, txn, prepare_lsn)
```
`logical.c` wrapper 再映射为 output plugin callbacks：
```text
begin_prepare_cb_wrapper()
change_cb_wrapper()
prepare_cb_wrapper()
```
`pgoutput` 最终发送：
```text
BEGIN PREPARE
INSERT/UPDATE/DELETE/TRUNCATE/MESSAGE
PREPARE
```
这些消息的 wire 编码在 `proto.c`：
```text
logicalrep_write_begin_prepare()
logicalrep_write_prepare()
```
prepare 输出后，不能像 ordinary commit 一样 `ReorderBufferCleanupTXN()`。
`ReorderBufferProcessTXN()` 在结尾判断：
```text
if (streaming || rbtxn_is_prepared(txn))
    ReorderBufferTruncateTXN(rb, txn, rbtxn_is_prepared(txn))
else
    ReorderBufferCleanupTXN(rb, txn)
```
prepared 情况下会截断已经输出的 changes 和 tuplecids。
但保留 `ReorderBufferTXN` skeleton、prepare info、invalidation 等后续 `COMMIT PREPARED` / `ROLLBACK PREPARED` 需要的状态。
这就是 prepared transaction 与普通 commit 的生命周期差异：
```text
普通 commit:
  输出后完整 cleanup。

prepare:
  输出后释放大部分 decoded payload，
  但事务对象不能消失。
```
## 11. `COMMIT PREPARED` 与 `ROLLBACK PREPARED`
`DecodeCommit()` 在 `two_phase = true` 时不调用 `ReorderBufferCommit()`。
它调用：
```text
ReorderBufferFinishPrepared(ctx->reorder,
                            xid,
                            commit_lsn,
                            end_lsn,
                            SnapBuildGetTwoPhaseAt(ctx->snapshot_builder),
                            commit_time,
                            origin_id,
                            origin_lsn,
                            gid,
                            true)
```
`DecodeAbort()` 在 prepared abort 且不 skip 时也调用同一个函数：
```text
ReorderBufferFinishPrepared(..., is_commit = false)
```
`ReorderBufferFinishPrepared()` 先找到事务对象。
找不到就返回。
然后保存 prepare record 的边界：
```text
prepare_end_lsn = txn->end_lsn
prepare_time = txn->prepare_time
```
再保存 GID：
```text
txn->gid = pstrdup(gid)
```
关键边界是：
```text
if ((txn->final_lsn < two_phase_at) && is_commit)
    ReorderBufferReplay(... prepare info ...)
```
含义是：
```text
如果这个事务的 prepare 发生在 two_phase_at 之前，
而现在看到了 COMMIT PREPARED，
为了保证下游能看到对应 prepared state，
需要先按 prepare 信息 replay 一次。
```
注意它只对 commit prepared 做这个补偿。
注释说明，如果 abort 且 prepare 未做过，不需要解码整个 xact。
之后 `ReorderBufferFinishPrepared()` 把最终 commit/abort 信息写回 txn：
```text
txn->final_lsn = commit_lsn
txn->end_lsn = end_lsn
txn->commit_time = commit_time
txn->origin_id = origin_id
txn->origin_lsn = origin_lsn
```
然后分支：
```text
is_commit:
  rb->commit_prepared(rb, txn, commit_lsn)

!is_commit:
  rb->rollback_prepared(rb, txn, prepare_end_lsn, prepare_time)
```
`logical.c` wrapper 映射为：
```text
commit_prepared_cb_wrapper()
rollback_prepared_cb_wrapper()
```
`pgoutput` 发送：
```text
logicalrep_write_commit_prepared()
logicalrep_write_rollback_prepared()
```
最后必须执行：
```text
ReorderBufferExecuteInvalidations(txn->ninvalidations, txn->invalidations)
ReorderBufferCleanupTXN(rb, txn)
```
这一步让 decoder 本地 catalog cache 退出该 prepared transaction 的 historic view。
不要把 `COMMIT PREPARED` 看成“普通 commit 的另一个名字”。
在 logical decoding 里，它是已准备事务的最终 resolution。
普通 change 可能已经在 prepare 阶段发过。
最终消息只决定“提交 prepared state”或“撤销 prepared state”。
## 12. Streaming 的触发点：内存压力下提前输出
streaming 不是每个事务都会走。
主要触发点在 `ReorderBufferCheckMemoryLimit()`。
每次 `ReorderBufferQueueChange()` 加入 change 后，会更新内存账本并检查 `logical_decoding_work_mem`。
当：
```text
rb->size >= logical_decoding_work_mem * 1024
```
就进入 eviction loop。
如果允许 streaming，优先找 largest streamable top transaction：
```text
ReorderBufferCanStartStreaming(rb)
ReorderBufferLargestStreamableTopTXN(rb)
```
可 streaming 的必要条件包括：
```text
ctx->streaming 为 true；
SnapBuildCurrentState(builder) >= SNAPBUILD_CONSISTENT；
SnapBuildXactNeedsSkip(builder, ctx->reader->ReadRecPtr) 为 false；
事务有 base_snapshot；
事务有 streamable changes；
事务不是 aborted；
没有 partial change 阻塞 streaming。
```
如果找到，就调用：
```text
ReorderBufferStreamTXN(rb, txn)
```
否则回退：
```text
ReorderBufferSerializeTXN(rb, txn)
```
也就是 spill 到 slot 目录下的 spill file。
这说明 streaming 和 spill 是同一个资源压力决策点的两种后果：
```text
plugin/protocol/snapshot 条件满足:
  提前通过 stream callbacks 发给下游。

条件不满足:
  序列化到磁盘，commit 时再 restore。
```
streaming 可以降低 publisher 端内存和磁盘 spill 压力。
更重要的是，它也把数据提前送给 apply 端。
apply 端不用等最后一个 commit record 才第一次看到这 100GB changes。
它可以边收边写临时文件、边调度 parallel apply worker、边做 row decoding 和 relation cache 准备。
但最终仍然等 `STREAM COMMIT` 才提交。
## 13. `ReorderBufferStreamTXN()`：把未提交事务按 chunk 发出去
`ReorderBufferStreamTXN()` 只能处理 top transaction：
```text
Assert(rbtxn_is_toptxn(txn))
```
它首先处理 snapshot。
第一次 streaming 时，普通 commit 路径还没有调用 `ReorderBufferCommitChild()`。
因此它不能假设 top transaction 已经继承了 subtransaction 的 base snapshot。
源码会遍历 `txn->subtxns`：
```text
ReorderBufferTransferSnapToParent(txn, subtxn)
```
如果最终没有 `base_snapshot`，说明到目前为止没有可解码数据库变化，直接返回。
第一次 streaming 建立：
```text
command_id = FirstCommandId
snapshot_now = ReorderBufferCopySnap(rb, txn->base_snapshot, txn, command_id)
```
后续 streaming chunk 不能从头重建状态。
它会复用上一次保存的：
```text
txn->snapshot_now
txn->command_id
```
并用 `ReorderBufferCopySnap()` 加上新 subtransaction 信息。
随后进入统一处理函数：
```text
ReorderBufferProcessTXN(rb,
                        txn,
                        InvalidXLogRecPtr,
                        snapshot_now,
                        command_id,
                        true)
```
最后更新统计：
```text
rb->streamCount += 1
rb->streamBytes += stream_bytes
rb->streamTxns += txn_is_streamed ? 0 : 1
UpdateDecodingStats(...)
```
并断言当前内存中的 change 已清空：
```text
txn->nentries == 0
txn->nentries_mem == 0
```
这里的关键是：
```text
streaming 不是拷贝一份 change 给下游后继续留在内存里；
它把已经 stream 的 change 从当前 reorder buffer 内存账本中截断。
```
否则 streaming 无法降低 publisher 端内存压力。
## 14. 主流程源码 walkthrough：`ReorderBufferProcessTXN()` 中 streaming 与普通回放的差异
`ReorderBufferProcessTXN()` 同时服务四种场景：
```text
普通 commit replay
prepare-time replay
in-progress streaming
streamed transaction final commit/prepare replay
```
参数 `streaming` 决定 callback 族。
开始时它总会：
```text
ReorderBufferBuildTupleCidHash()
SetupHistoricSnapshot()
StartTransactionCommand() 或 BeginInternalSubTransaction()
```
普通 replay 会先发：
```text
rb->begin()
或
rb->begin_prepare()
```
streaming 不发普通 begin。
它等第一条 change 到来时发：
```text
rb->stream_start(rb, txn, change->lsn)
```
然后每个用户 change 进入：
```text
ReorderBufferApplyChange(rb, txn, relation, change, streaming)
```
这个 helper 内部按 `streaming` 选择：
```text
streaming:
  rb->stream_change(...)

not streaming:
  rb->apply_change(...)
```
truncate 和 message 也类似：
```text
stream_truncate / apply_truncate
stream_message / message
```
当这个 chunk 处理完：
```text
streaming:
  rb->stream_stop(rb, txn, prev_lsn)

not streaming:
  rb->prepare() 或 rb->commit()
```
然后 streaming 保存当前 snapshot 和 command id：
```text
ReorderBufferSaveTXNSnapshot(rb, txn, snapshot_now, command_id)
```
再执行：
```text
ReorderBufferMaybeMarkTXNStreamed(rb, txn)
ReorderBufferTruncateTXN(rb, txn, false)
CheckXidAlive = InvalidTransactionId
```
所以一个大事务可能有多个 streaming chunk：
```text
STREAM START xid=100 first=true
  INSERT ...
STREAM STOP

STREAM START xid=100 first=false
  UPDATE ...
STREAM STOP

STREAM COMMIT xid=100
```
`pgoutput_stream_start()` 里会用 `rbtxn_is_streamed(txn)` 决定 `first_segment`。
第一次发送 replication origin，后续 chunk 不重复发送 origin。
## 15. streaming finalization：commit、abort、prepare
当一个已经 streamed 的事务最终提交，`ReorderBufferReplay()` 会先判断：
```text
if (rbtxn_is_streamed(txn))
{
    ReorderBufferStreamCommit(rb, txn);
    return;
}
```
`ReorderBufferStreamCommit()` 做两件事。
第一，先把剩余未 stream 的 change 发出去：
```text
ReorderBufferStreamTXN(rb, txn)
```
第二，根据是否 prepared 分支：
```text
rbtxn_is_prepared(txn):
  rb->stream_prepare(rb, txn, txn->final_lsn)
  txn->txn_flags |= RBTXN_SENT_PREPARE
  ReorderBufferTruncateTXN(rb, txn, true)

普通 streamed commit:
  rb->stream_commit(rb, txn, txn->final_lsn)
  ReorderBufferCleanupTXN(rb, txn)
```
这解释了两个协议消息：
```text
STREAM COMMIT:
  streamed ordinary transaction 的最终提交。

STREAM PREPARE:
  streamed transaction 在 2PC 中进入 prepared state。
```
如果事务 abort，`ReorderBufferAbort()` 会检查：
```text
if (rbtxn_is_streamed(txn))
    rb->stream_abort(rb, txn, lsn)
```
`pgoutput_stream_abort()` 发送：
```text
logicalrep_write_stream_abort(...)
```
下游据此丢弃该 top xid 或 subxid 的已接收 streaming data。
这就是 streaming 不破坏原子性的核心：
```text
提前发送的 change 都挂在 xid 上；
最终 COMMIT 才 apply；
最终 ABORT 必须能丢弃。
```
对 subscriber 来说，streaming 降低的是“第一次看到数据”和“开始预处理”的等待。
它不允许 subscriber 在 publisher commit 前提前提交。
## 16. output plugin wrapper：写出位置不是随便选
`logical.c` 的 callback wrappers 会设置 `ctx->accept_writes`、`ctx->write_xid`、`ctx->write_location` 和 `ctx->end_xact`，再交给 `OutputPluginPrepareWrite()` / `OutputPluginWrite()`。普通 change 和 streaming change 使用 `change->lsn`，`stream_start` 使用 `first_lsn`，普通 commit、prepare、stream commit 使用 `txn->end_lsn` 并把 `end_xact` 设为 true。
这个位置选择不是随便的。源码注释强调，`change->lsn` 的反馈不能单独确认当前事务已经安全消费，但可能帮助确认另一个事务的 commit。streaming 中客户端可能已经 flush 某个 chunk，但该 xid 尚未 commit；slot 不能因此丢掉重建该事务所需 WAL。`confirmed_flush` 是客户端确认边界，`restart_lsn` 是 server 重启解码和 WAL retention 边界，两者可以靠近，但不能混为一个状态。
## 17. `confirmed_flush` 与 `restart_lsn` 边界
`StartLogicalReplication()` 获得 slot 后创建 decoding context，从 `MyReplicationSlot->data.restart_lsn` 开始读 WAL，同时把 `sentPtr` 设为 `confirmed_flush`。前者回答“从哪里读才能恢复未完成 decoding state”，后者回答“从哪里开始对客户端输出新的 committed/protocol messages”。
如果客户端给的 start_lsn 早于 `confirmed_flush`，`CreateDecodingContext()` 会 forward 到 `confirmed_flush`，避免重复发送已经确认的 changes。`restart_lsn` 推进则更保守：`LogicalIncreaseRestartDecodingForSlot(current_lsn, restart_lsn)` 只写 `candidate_restart_valid` 和 `candidate_restart_lsn`；只有 `LogicalConfirmReceivedLocation(flushPtr)` 看到 `candidate_restart_valid <= flushPtr`，才真正把 `slot->data.restart_lsn` 改成 candidate。
因此 streaming 下可以看到 `confirmed_flush_lsn` 随 chunks 推进，而 `restart_lsn` 仍因未完成事务、snapshot 或 candidate 尚未确认而滞后。把 `confirmed_flush_lsn` 当作 WAL retention 的唯一依据，会误判 slot 风险。
## 18. Protocol messages：为什么需要新消息类型
`logicalproto.h` 的 message enum 明确区分普通事务、2PC 和 streaming：
```text
BEGIN                 'B'
COMMIT                'C'
BEGIN PREPARE         'b'
PREPARE               'P'
COMMIT PREPARED       'K'
ROLLBACK PREPARED     'r'
STREAM START          'S'
STREAM STOP           'E'
STREAM COMMIT         'c'
STREAM ABORT          'A'
STREAM PREPARE        'p'
```
普通 DML 仍是 `INSERT`、`UPDATE`、`DELETE`、`TRUNCATE`、`MESSAGE`，但 streaming 模式下 `proto.c` 会在 DML 后附带 xid，例如 `logicalrep_write_insert()` 在 `TransactionIdIsValid(xid)` 时写入 xid。这样 subscriber 能把 interleaved chunks 归属到对应事务。
`logicalrep_write_stream_start()` 发送 xid 和 `first_segment`；`logicalrep_write_stream_commit()` 发送 xid、flags、commit_lsn、end_lsn、commit_time；`logicalrep_write_stream_abort()` 发送 xid、subxid，parallel streaming 还会带 abort_lsn 和 abort_time。2PC 消息也不能用普通 `BEGIN/COMMIT` 代替：`BEGIN PREPARE` 带 GID 和 prepare 相关上下文，`COMMIT PREPARED` / `ROLLBACK PREPARED` 是对已 prepared state 的 resolution。
## 19. apply 端等待压力如何被降低
默认非 streaming 的大事务有一个集中点：
```text
publisher:
  decode changes -> reorder buffer memory/spill
  commit 到达后统一输出

network:
  commit 后才开始传输大量 DML

subscriber:
  收到后才开始 apply/spool
  最后 commit
```
这会造成三类等待。
第一，publisher 等待 commit 才能释放 reorder buffer。
即使 spill 保护了内存，也会在 commit 时 restore、merge、发送，形成 I/O 和 CPU 峰值。
第二，network 等待 commit 才开始传输最大 payload。
commit 后的复制延迟突然变大。
第三，subscriber apply 等待完整事务到达才开始处理。
大事务的 apply 延迟被集中到事务尾部。
streaming mode 改成：
```text
publisher:
  内存达到阈值后 stream chunk，随后截断已 stream changes。

network:
  在事务运行期间持续传输 chunks。

subscriber:
  按 xid 接收、暂存、预处理或并行 apply chunks。
  最终等待 STREAM COMMIT 决定提交。
```
这样降低的是：
```text
tail latency concentration
publisher reorder buffer peak
commit-time network burst
subscriber first-byte wait
apply 端一次性处理峰值
```
但它不降低：
```text
事务最终原子性要求；
publisher 必须能在 abort 时通知下游丢弃；
slot 必须保留足够 WAL；
下游必须有存放 streamed state 的资源。
```
换句话说，streaming 把大事务从“最后一次性传输和处理”改成“运行期间分批传输和准备”。
它没有把一个大事务变成多个已提交小事务。
这是诊断时最重要的边界。
## 20. 错误路径一：插件或协议不兼容
不兼容最早可能在 `pgoutput_startup()` 抛错。
缺 `proto_version` 或 `publication_names` 会直接错误。
请求 streaming 但协议太低：
```text
requested proto_version=%d does not support streaming, need %d or higher
```
请求 parallel streaming 但协议太低：
```text
requested proto_version=%d does not support parallel streaming, need %d or higher
```
请求 two_phase 但协议太低：
```text
requested proto_version=%d does not support two-phase commit, need %d or higher
```
请求能力但 output plugin 不支持，也会报：
```text
streaming requested, but not supported by output plugin
two-phase commit requested, but not supported by output plugin
```
即使启动时通过，如果 plugin callback 集合不完整，`logical.c` wrapper 会在实际调用时错误：
```text
logical streaming requires a stream_commit_cb callback
logical replication at prepare time requires a rollback_prepared_cb callback
```
错误上下文会包含：
```text
slot name
output plugin name
callback name
associated LSN
```
这是诊断 output plugin 问题的第一入口。
不要只看 subscriber 端报错。
publisher 日志的 output plugin error context 通常更接近根因。
## 21. 错误路径二：prepare-time decoding 与 concurrent abort
prepare-time decoding 和 streaming 都有一个共同风险：
```text
decoder 在最终 commit 之前打开 historic catalog，
而对应 xid 可能并发 abort。
```
`reorderbuffer.c` 为此引入 `CheckXidAlive`。
在 `ReorderBufferProcessTXN()` 中，如果是 streaming 或 prepared change：
```text
SetupCheckXidLive(curtxn->xid)
```
注释里的例子是 catalog tuple 被事务 501 更新，501 后来 abort，另一个事务 502 又更新同一 catalog tuple。
如果 decoder 仍按 501 的视角解释 catalog，可能读到错误版本甚至 crash。
当 catalog scan 发现 `CheckXidAlive` 已 abort，会抛出特定错误码：
```text
ERRCODE_TRANSACTION_ROLLBACK
```
`ReorderBufferProcessTXN()` 的 `PG_CATCH` 会区分：
```text
如果是 transaction rollback 且正在 streaming 或 prepared:
  标记 curtxn RBTXN_IS_ABORTED
  必要时 stream_stop
  ReorderBufferResetTXN()
  返回上层，等待后续 abort / rollback prepared 边界

其它错误:
  ReorderBufferCleanupTXN()
  PG_RE_THROW()
```
这不是普通容错。
它是为了避免在中间态解码时使用错误 catalog。
对于 streaming，已经发出的 change 会在后续 `stream_abort` 中让下游丢弃。
对于 prepared transaction，`DecodePrepare()` 的注释明确说：
```text
即使检测到 concurrent abort，也可能已经发送过一些 changes；
因此仍然发送 prepare，然后等 rollback prepared 时撤销。
```
这是一个工程取舍。
可以设计一个更复杂的 abort API 来少发消息，但会显著增加状态组合。
当前实现选择保持协议状态和 publisher 行为一致。
## 22. 错误路径三：skip、fast-forward 与 restart_lsn
`DecodeTXNNeedSkip()` 可能因为这些原因跳过事务：
```text
snapshot builder 判断不需要；
事务 database 与 slot database 不匹配；
origin 被过滤；
ctx->fast_forward；
```
普通 committed transaction skip 可以：
```text
ReorderBufferForget(subxid)
ReorderBufferForget(xid)
```
因为 commit 边界已经到达。
prepared transaction 的 prepare 阶段不能这么做。
`DecodePrepare()` 中如果 skip：
```text
ReorderBufferSkipPrepare()
ReorderBufferInvalidate()
```
原因是 prepared xact 尚未最终结束。
过早移除会影响 `restart_lsn` 计算。
这也是本节最容易误解的边界之一：
```text
skip output
  !=
forget transaction state immediately
```
在 ordinary commit 上，它们可能重合。
在 prepared transaction 上，它们不能重合。
fast-forward 模式也类似。
它可以绕过输出，但仍要消化 WAL、snapshot 和 slot 边界。
否则 slot 可能认为自己已经安全推进，实际重启后却无法重建 prepared 或 in-progress transaction。
## 23. 成本、资源与跨模块传播
2PC decoding 的主要成本是“状态活得更久”：prepare 阶段输出后，`ReorderBufferTXN` skeleton 仍要保留；本地 prepared xact 仍持有 locks 和 dummy PGPROC；slot 的 `two_phase_at` 还要参与重启边界判断。如果 prepared transaction 长时间不 resolution，影响会传播到 `pg_prepared_xacts`、lock manager、ProcArray/xmin horizon、logical slot `restart_lsn` 和 subscriber prepared state。
streaming 的成本模型不同。它减少 publisher reorder buffer 峰值、commit-time network burst 和 subscriber first-byte wait，但会增加 `stream_start/stream_stop` 消息数、output plugin callback 次数、subscriber 按 xid 暂存状态，以及 `STREAM ABORT` / parallel streaming 的恢复复杂度。它不是无条件降低所有资源，而是把压力从“最终 commit 一次性爆发”迁移到“事务运行期间分批传输和管理”。
## 24. 观测与诊断入口
slot 状态先看：
```sql
select slot_name, plugin, active, confirmed_flush_lsn, restart_lsn,
       two_phase, two_phase_at
from pg_replication_slots;
```
它能说明 slot 是否启用 2PC、`confirmed_flush_lsn` 是否推进、`restart_lsn` 是否滞后；它看不到单个 `ReorderBufferTXN` 的 `RBTXN_IS_STREAMED`、subscriber chunk 暂存状态或 `snapshot_now`。
slot 统计再看：
```sql
select slot_name, spill_count, spill_bytes, stream_count, stream_bytes,
       total_txns, total_bytes
from pg_stat_replication_slots;
```
`stream_count` 增长说明走过 streaming callback；`spill_count` 增长而 `stream_count` 不增长，优先检查客户端 `streaming` option、`proto_version`、plugin callbacks、snapshot consistent、partial change 和事务是否 streamable。
walsender 进度看：
```sql
select pid, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
from pg_stat_replication;
```
`flush_lsn` 会触发 `LogicalConfirmReceivedLocation()`，但不能单独解释 `restart_lsn`。prepared xact 看：
```sql
select gid, prepared, owner, database from pg_prepared_xacts;
```
如果 publisher 或 subscriber 有长期 prepared xact，logical replication 延迟可能不是网络问题，而是 2PC resolution 没发生。日志里如果出现 `requested proto_version=1 does not support streaming`、`requested proto_version=2 does not support two-phase commit`、`logical streaming requires a stream_change_cb callback` 或 `logical replication at prepare time requires a prepare_cb callback`，优先定位 publisher 端 output plugin callback context。
源码断点建议放在 `DecodePrepare()`、`DecodeCommit()`、`DecodeAbort()`、`ReorderBufferPrepare()`、`ReorderBufferFinishPrepared()`、`ReorderBufferCheckMemoryLimit()`、`ReorderBufferStreamTXN()`、`ReorderBufferStreamCommit()`、`stream_start_cb_wrapper()`、`stream_commit_cb_wrapper()`、`commit_prepared_cb_wrapper()`。观察 `txn->txn_flags`、`txn->gid`、`txn->final_lsn`、`txn->end_lsn`、`txn->total_size`、`ctx->streaming`、`ctx->twophase`、`slot->data.confirmed_flush`、`slot->data.restart_lsn`、`slot->data.two_phase_at`。
## 25. 课堂实验
实验一：观察 2PC 边界。创建表和 publication，创建 `TWO_PHASE true` 的 logical slot，启动 `pgoutput` 时传 `proto_version '3'`、`publication_names`、`two_phase 'true'`。执行 `BEGIN; INSERT ...; PREPARE TRANSACTION 'gid_2pc_1';` 后，在 `pg_prepared_xacts` 中确认 prepared state，再执行 `COMMIT PREPARED` 或 `ROLLBACK PREPARED`。源码跟读 `xact_decode()` 的 `XLOG_XACT_PREPARE` 分支，画出 `ReorderBufferRememberPrepareInfo()`、`ReorderBufferPrepare()`、`ReorderBufferFinishPrepared()` 的字段变化。
实验二：触发 streaming。降低 `logical_decoding_work_mem`，用 `proto_version >= 2` 且 `streaming 'on'` 的客户端启动 pgoutput，执行一个包含大量 INSERT 的长事务并延迟 commit。观察 `pg_stat_replication_slots.stream_count` / `stream_bytes`。如果只看到 `spill_count`，按 `pgoutput_startup()` 的 capability 校验和 `ReorderBufferCanStartStreaming()` 的条件回查。
实验三：区分 `confirmed_flush` 和 `restart_lsn`。在 streaming 大事务未最终 commit 时观察 `pg_replication_slots`：`confirmed_flush_lsn` 可以因 flush feedback 前进，但 `restart_lsn` 仍可能滞后，因为未完成事务还需要从更早 WAL 重建。回到源码核对 `LogicalIncreaseRestartDecodingForSlot()` 只设置 candidate，`LogicalConfirmReceivedLocation()` 在 `candidate_restart_valid <= flushPtr` 后才真正更新 `data.restart_lsn`。
## 26. 常见误区
误区一：`PREPARE TRANSACTION` 等于 commit。实际它是持久中间态，logical decoding 需要 `PREPARE`，后续再 `COMMIT PREPARED` 或 `ROLLBACK PREPARED`。
误区二：streaming 会让 subscriber 提前提交大事务的一部分。实际 streaming 只提前发送和预处理，事务可见性仍由 `STREAM COMMIT` 决定。
误区三：`confirmed_flush_lsn` 推进就说明 WAL 可以释放。实际 WAL retention 主要受 `restart_lsn` 约束，未完成或 prepared 事务可能让它保守。
误区四：只要 `pgoutput` 支持 streaming，所有大事务都会 streaming。还要看 protocol version、client option、snapshot consistent、streamable change、partial change 和内存阈值。
误区五：`two_phase` 是纯临时 option。启用它可能写入 slot 的 `two_phase` 和 `two_phase_at`，影响重启后的 prepare 边界判断。
误区六：`stream_abort` 是普通 abort 的重复消息。它专门让下游丢弃已经 streamed 的 provisional changes。
## 27. 讨论题
1. 为什么 `DecodePrepare()` 不能像 skipped commit 一样直接 `ReorderBufferForget()`？
2. 如果 `COMMIT PREPARED` 到达时发现 prepare 发生在 `two_phase_at` 之前，为什么只对 commit prepared 补发 prepare，而不是对 rollback prepared 也解码整个事务？
3. 为什么 `stream_change` 消息必须携带 xid，而普通非 streaming change 不需要？
4. `stream_start` 的 `first_segment` 对 subscriber 有什么意义？
5. 为什么 `change->lsn` 可以更新发送位置，却不能单独证明当前事务已经安全确认？
6. 如果 `stream_count` 为 0 但 `spill_count` 很高，你会按什么顺序排查？
7. prepared transaction 长时间不 resolution，会同时影响哪些模块？
8. 为什么 prepare-time decoding 和 streaming 都需要 `CheckXidAlive` 这种 concurrent abort 检查？
## 28. 本节小结
本节核心链路是：`XLOG_XACT_PREPARE -> DecodePrepare() -> ReorderBufferPrepare() -> begin_prepare/change/prepare -> 保留 prepared TXN skeleton -> COMMIT/ROLLBACK PREPARED -> ReorderBufferFinishPrepared() -> commit_prepared/rollback_prepared -> cleanup`。
streaming 链路是：`ReorderBufferQueueChange() -> ReorderBufferCheckMemoryLimit() -> ReorderBufferStreamTXN() -> stream_start/stream_change/stream_stop -> truncate already streamed changes -> final commit/abort/prepare -> stream_commit/stream_abort/stream_prepare`。
核心状态是 `slot->data.two_phase/two_phase_at`、`slot->data.confirmed_flush/restart_lsn`、`ctx->streaming/twophase`、`txn->txn_flags`、`txn->final_lsn/end_lsn`、`txn->snapshot_now/command_id`。`ReorderBufferTXN` 和 changes 是 decoding backend-local；slot state 是 shared memory 加 on-disk persistent state；prepared xact 本地状态属于 two-phase transaction manager；wire protocol state 属于 output plugin 和 subscriber。
错误路径的收口是：不兼容协议或 callbacks 在 startup/wrapper 报错；concurrent abort 标记 aborted，后续用 `stream_abort` 或 `rollback_prepared` 收口；skip prepare 保留 prepared skeleton，避免 `restart_lsn` 错误；output plugin ERROR 由 `logical.c` error context 指向 callback 和 LSN。
可观测状态分三类：`pg_replication_slots`、`pg_stat_replication_slots`、`pg_stat_replication`、`pg_prepared_xacts` 能直接看；是否因 snapshot、partial change 或 protocol 未 streaming 只能推断；单个 `ReorderBufferTXN` flags、`snapshot_now`、`command_id` 和 chunk 内部状态基本不可见。
可迁移规律是：当外部协议需要表达比 commit/abort 更细的事务中间态时，内核不能只加一个消息类型；它必须同时拆分 runtime 状态机、cleanup 时机、restart 边界、错误恢复和 capability negotiation。2PC 解决 prepared state 的协议表达，streaming 解决大 in-progress transaction 的传输和 apply 等待集中；两者都不能绕过最终事务原子性和 slot `restart_lsn` 边界。
