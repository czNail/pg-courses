# PostgreSQL output plugin callback 生命周期
## 课程定位
前置知识：已经理解 logical slot 会保护 `restart_lsn`、`catalog_xmin` 和 `confirmed_flush`，也已经理解 `decode.c` 把 WAL record 转成 `ReorderBufferChange`，`reorderbuffer.c` 再按事务提交边界输出。
本节唯一主问题：
```text
output plugin 如何通过 startup、begin、change、commit、message、truncate 和 shutdown callback
接管 logical change 输出，哪些 callback 只定义格式而不改变解码正确性？
```
核心矛盾：logical decoding 的正确性由 WAL 顺序读取、historic snapshot、subtransaction 重组、abort 过滤、commit 边界和 slot 确认共同保证；但外部系统真正看到的字节格式、relation 过滤、协议版本、事务空输出、schema 消息和 message 是否交付，都交给 output plugin。
PostgreSQL 的选择是把 correctness path 和 output path 分成两层。 第一层在 core 中完成：
```text
LogicalDecodingProcessRecord()
  -> DecodeInsert() / DecodeUpdate() / DecodeDelete() / DecodeTruncate()
  -> ReorderBufferQueueChange()
  -> DecodeCommit()
  -> ReorderBufferCommit()
  -> ReorderBufferProcessTXN()
```
第二层在 plugin callback 中完成：
```text
begin callback
  -> change / truncate / message callbacks
  -> commit callback
  -> OutputPluginPrepareWrite()
  -> OutputPluginWrite()
```
学完后应能判断：为什么 output plugin 可以把同一事务写成 `test_decoding` 的文本行，也可以写成 `pgoutput` 的 binary protocol；为什么插件可以跳过某个 publication 外的 relation，却不能把 abort 事务变成 committed；为什么 `message_cb` 和 `truncate_cb` 缺失时只是输出缺口，不是解码错误；为什么插件 `ERROR` 通常会让 slot 位置停在客户端已确认的位置，而不是停在 decoder 已经读到的 WAL 位置。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
注意命名边界：源码里没有一个“output plugin 主循环”函数。
真正的桥接点分散在 `logical.c` 的 wrapper、`reorderbuffer.c` 的事务 replay、插件自己的 `_PG_output_plugin_init()`、以及消费端的 writer callback 中。
## 1. 本节在总主线中的位置
前一节已经把主线推进到：
```text
WAL record
  -> rmgr decode callback
  -> ReorderBufferChange
  -> commit record
  -> ReorderBufferProcessTXN()
```
到这里，PostgreSQL 已经知道：
```text
这个事务提交了吗？
哪些 subtransaction 存活？
每条 change 属于哪个 relation？
应该用哪个 historic snapshot 读 catalog？
哪些 internal snapshot / command id / invalidation 必须先处理？
```
本节开始回答另一个问题：
```text
这些已经正确排序的 logical change，如何变成客户端看到的输出？
```
这不是简单的打印层。 `pgoutput` 会解析 replication protocol 选项。 `pgoutput` 会维护 publication cache。 `pgoutput` 会决定是否发送 relation schema。
`pgoutput` 会用 row filter 把 UPDATE 转成 INSERT 或 DELETE。 `test_decoding` 则几乎只做文本格式化。 两个插件看到的是同一组 core callback。
它们的行为差异说明一个关键边界：
```text
decode correctness 属于 core；
external representation 属于 plugin。
```
本节不展开 subscriber apply worker。 本节也不展开 publication DDL 的完整规则。 这些内容只作为 callback 做决策时的输入。
主线只跟一个事务从 commit replay 到 plugin write 的生命周期：
```text
CreateDecodingContext()
  -> LoadOutputPlugin()
  -> startup_cb_wrapper()
  -> ReorderBufferProcessTXN()
     -> begin_cb_wrapper()
     -> change_cb_wrapper()
     -> truncate_cb_wrapper()
     -> message_cb_wrapper()
     -> commit_cb_wrapper()
  -> OutputPluginPrepareWrite()
  -> OutputPluginWrite()
  -> consumer confirms LSN
  -> FreeDecodingContext()
  -> shutdown_cb_wrapper()
```
## 2. 核心矛盾与一个容易误判的 runtime 现象
先看一个现象。 用 `test_decoding` 创建 slot：
```sql
SELECT * FROM pg_create_logical_replication_slot('cb_demo', 'test_decoding');
```
执行一个普通事务：
```sql
CREATE TABLE cb_t(id int primary key, note text);
INSERT INTO cb_t VALUES (1, 'one');
UPDATE cb_t SET note = 'uno' WHERE id = 1;
TRUNCATE cb_t;
```
消费：
```sql
SELECT * FROM pg_logical_slot_get_changes('cb_demo', NULL, NULL);
```
你会看到类似：
```text
BEGIN ...
table public.cb_t: INSERT: ...
table public.cb_t: UPDATE: ...
table public.cb_t: TRUNCATE: ...
COMMIT ...
```
如果换成 `pgoutput`，客户端不再看到这些文本。 它看到的是 logical replication protocol 消息。
同一条 `ReorderBufferChange` 可以被写成文本，也可以被写成 binary protocol。 这说明 `change_cb` 并不是“解码出 change”。
它只是“把已经解码好的 change 写出去”。 再看另一个现象。
如果插件在某个 callback 中 `ERROR`，SQL `pg_logical_slot_get_changes()` 不会在函数尾调用 `LogicalConfirmReceivedLocation()`。
下一次消费通常会从旧的 `confirmed_flush` 继续。 walsender 路径更明显。
`XLogSendLogical()` 读 WAL 并调用 `LogicalDecodingProcessRecord()` 后会更新本地 `sentPtr`。
但 logical slot 的 `confirmed_flush` 只有客户端反馈 `flushPtr` 后，`ProcessRepliesIfAny()` 才调用 `LogicalConfirmReceivedLocation(flushPtr)` 推进。 所以：
```text
decoder 读到某个 WAL record
  !=
plugin 写出了某个消息
  !=
slot 已经确认可以从这里以后继续
```
这就是本节的 runtime truth。
## 3. 核心文件分工与阅读顺序
阅读顺序按生命周期，不按文件名。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/output_plugin.h` | `OutputPluginCallbacks`、callback 类型、`OutputPluginOptions`、`OutputPluginPrepareWrite()` / `OutputPluginWrite()`。 |
| 2 | `src/include/replication/logical.h` | `LogicalDecodingContext` 中的 `callbacks`、`options`、`out`、`output_plugin_private`、`accept_writes`、`write_location`。 |
| 3 | `src/backend/replication/logical/logical.c` | `StartupDecodingContext()`、`LoadOutputPlugin()`、各 callback wrapper、error context、slot confirmation helper。 |
| 4 | `src/include/replication/reorderbuffer.h` | `ReorderBuffer` callback slots、`ReorderBufferTXN.output_plugin_private`、`ReorderBufferChange`。 |
| 5 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferProcessTXN()` 如何在 historic snapshot 和内部事务中调用 plugin。 |
| 6 | `src/backend/replication/logical/decode.c` | commit、abort、message 和 origin/prepare filter 进入 reorder buffer 的边界。 |
| 7 | `contrib/test_decoding/test_decoding.c` | 最小文本插件，展示 callback 如何只格式化输出。 |
| 8 | `src/backend/replication/pgoutput/pgoutput.c` | 生产协议插件，展示 startup option、publication filter、schema cache、streaming/2PC callback。 |
| 9 | `src/backend/replication/logical/proto.c` | `pgoutput` 最终调用的 protocol writer。 |
| 10 | `src/backend/replication/walsender.c` | walsender writer callback、client feedback 推进 `confirmed_flush`。 |
| 11 | `src/backend/replication/logical/logicalfuncs.c` | SQL SRF writer callback、`get_changes` 与 `peek_changes` 的确认差异。 |
推荐先读 `output_plugin.h`。 它告诉你插件 API 的 public surface。 再读 `logical.c`。
它告诉你 core 为什么不直接把 `ReorderBuffer` callback 暴露给插件，而是加了一层 wrapper。 然后读 `reorderbuffer.c`。
它告诉你 callback 何时被调用，以及调用前后有哪些 transaction、snapshot、resource owner cleanup。 最后对照 `test_decoding.c` 和 `pgoutput.c`。
这样才能避免把某个插件的选择误认为 logical decoding 的通用语义。
## 4. 关键状态：plugin 只拿到 context，不拥有 decoder
`LogicalDecodingContext` 是 decoding session 的顶层对象。 它在 `StartupDecodingContext()` 中分配在 `"Logical decoding context"` memory context 下。 它保存：
| 字段 | 语义 |
| --- | --- |
| `slot` | 当前 acquired logical slot。 |
| `reader` | `XLogReaderState`，保存当前 WAL 读取位置。 |
| `reorder` | `ReorderBuffer`，按事务聚合 change。 |
| `snapshot_builder` | `SnapBuild`，维护 historic snapshot 和 consistency state。 |
| `callbacks` | 插件注册的 `OutputPluginCallbacks`。 |
| `options` | startup callback 设置的 `OutputPluginOptions`。 |
| `output_plugin_options` | SQL 或 replication protocol 传入的 plugin option list。 |
| `out` | 插件写入的 `StringInfo` 输出缓冲区。 |
| `output_plugin_private` | 插件 session 级私有状态。 |
| `output_writer_private` | 消费端 writer 私有状态。 |
| `streaming` | 插件是否支持并启用 streaming callback。 |
| `twophase` | 插件是否支持并启用 two-phase callback。 |
| `accept_writes` | 当前 callback 是否允许调用 `OutputPluginPrepareWrite()`。 |
| `write_location` | 当前输出消息关联的 LSN。 |
| `write_xid` | 当前输出消息关联的 XID。 |
| `end_xact` | 当前是否在输出事务结束位置。 |
这里有两个 private data 指针。 `ctx->output_plugin_private` 属于插件。 `ctx->output_writer_private` 属于消费端。
`test_decoding` 在 startup 中分配 `TestDecodingData`，挂到 `ctx->output_plugin_private`。
`pgoutput` 在 startup 中分配 `PGOutputData`，也挂到 `ctx->output_plugin_private`。 SQL SRF 的 tuplestore state 则挂在 `ctx->output_writer_private`。
walsender 不需要用同一个结构承载插件状态。 不要把这两个 private data 混在一起。 `ReorderBufferTXN` 还有一个 `output_plugin_private`。
这是 transaction 级插件状态。 `pgoutput_begin_txn()` 会分配 `PGOutputTxnData`，记录 `sent_begin_txn`。
`test_decoding` 会分配 `TestDecodingTxnData`，记录这个事务或 stream 是否已经写出 change。 这些字段由插件维护。
core 只负责在正确时间把同一个 `ReorderBufferTXN *txn` 传回插件。
## 5. `OutputPluginCallbacks` 的真实 contract
`src/include/replication/output_plugin.h` 中的 `OutputPluginCallbacks` 包含几组 callback。 普通事务输出组：
```text
startup_cb
begin_cb
change_cb
truncate_cb
commit_cb
message_cb
filter_by_origin_cb
shutdown_cb
```
two-phase 组：
```text
filter_prepare_cb
begin_prepare_cb
prepare_cb
commit_prepared_cb
rollback_prepared_cb
```
streaming 组：
```text
stream_start_cb
stream_stop_cb
stream_abort_cb
stream_prepare_cb
stream_commit_cb
stream_change_cb
stream_message_cb
stream_truncate_cb
```
`LoadOutputPlugin()` 只强制三项：
```text
begin_cb
change_cb
commit_cb
```
缺少这三项会 `ERROR`。 因为普通事务至少需要开始、change 和结束的输出边界。 `startup_cb` 可选。
缺少它时，插件无法设置 options 或私有状态，但 API 允许。 `shutdown_cb` 可选。 缺少它时，`FreeDecodingContext()` 只释放 core context。
`message_cb` 可选。 缺少它时，`message_cb_wrapper()` 直接 return。 `truncate_cb` 可选。 缺少它时，`truncate_cb_wrapper()` 直接 return。
这个行为说明一个重要事实：
```text
message 和 truncate callback 的缺失不会破坏 reorder buffer 的正确性；
它只表示插件不输出这类事件。
```
streaming callback 的规则更严格。 `StartupDecodingContext()` 只要发现任一 streaming callback 非 NULL，就先把 `ctx->streaming` 置为 true。
但真正调用 wrapper 时，`stream_start_cb`、`stream_stop_cb`、`stream_abort_cb`、`stream_commit_cb` 和 `stream_change_cb` 在 streaming mode 下是必需的。
`stream_message_cb` 和 `stream_truncate_cb` 仍然是可选的。 two-phase callback 也类似。
只要插件注册了 begin/prepare/commit prepared/rollback prepared/filter prepare 中任一项，core 认为它可能支持 two-phase。
真正启用还要看 slot 和启动选项。
启用 two-phase 后，`begin_prepare_cb`、`prepare_cb`、`commit_prepared_cb`、`rollback_prepared_cb` 在对应路径上是必需的。 `filter_prepare_cb` 是可选 filter。
这组规则的意思不是“插件可以随意半支持 streaming”。 它的意思是：
```text
callback table 是 ABI；
具体模式是否启用，要等 startup option、slot 状态和 wrapper 共同判定。
```
## 6. plugin 装载：`_PG_output_plugin_init()` 只填表
output plugin 是一个 shared library。 core 通过 `load_external_function(plugin, "_PG_output_plugin_init", false, NULL)` 找到符号。 真实函数类型是：
```text
LogicalOutputPluginInit
```
插件必须导出：
```text
_PG_output_plugin_init(OutputPluginCallbacks *cb)
```
这个函数只应该填 callback 指针。 `test_decoding` 的版本很直接：
```text
cb->startup_cb = pg_decode_startup
cb->begin_cb = pg_decode_begin_txn
cb->change_cb = pg_decode_change
cb->truncate_cb = pg_decode_truncate
cb->commit_cb = pg_decode_commit_txn
cb->message_cb = pg_decode_message
...
```
`pgoutput` 也类似：
```text
cb->startup_cb = pgoutput_startup
cb->begin_cb = pgoutput_begin_txn
cb->change_cb = pgoutput_change
cb->truncate_cb = pgoutput_truncate
cb->message_cb = pgoutput_message
cb->commit_cb = pgoutput_commit_txn
...
```
这里不应该开始读 WAL。 这里也不应该访问某个事务的 relation change。 原因很简单。 此时 `LogicalDecodingContext` 还没有开始 replay 某个事务。
`LoadOutputPlugin()` 的职责只是检查 callback 表是否基本完整。 真正的 plugin session 初始化在 `startup_cb_wrapper()` 调用 `startup_cb` 时发生。
## 7. startup callback：设置输出面，而不是证明一致性
`CreateInitDecodingContext()` 和 `CreateDecodingContext()` 都会调用 `StartupDecodingContext()`。 这个函数会：
```text
创建 LogicalDecodingContext
装载 output plugin
创建 XLogReader
创建 ReorderBuffer
创建 SnapBuild
把 ReorderBuffer callback 指向 logical.c wrapper
创建 ctx->out
保存 writer callbacks
```
随后外层才调用 startup wrapper。 slot 创建路径传入：
```text
is_init = true
```
已有 slot 开始消费路径传入：
```text
is_init = false
```
`startup_cb_wrapper()` 做三件事。 第一，它把 error context 标记为 `startup`。 第二，它设置：
```text
ctx->accept_writes = false
ctx->end_xact = false
```
第三，它调用插件的 `startup_cb(ctx, opt, is_init)`。 startup callback 不能调用 `OutputPluginPrepareWrite()`。
如果调用，会因为 `accept_writes` 为 false 而 `ERROR`。 `test_decoding` 在 startup 中做的是：
```text
分配 TestDecodingData
创建 "text conversion context"
设置 include_xids / include_timestamp / skip_empty_xacts / only_local
设置 opt->output_type = OUTPUT_PLUGIN_TEXTUAL_OUTPUT
设置 opt->receive_rewrites = false
解析 options
用 stream-changes 选项收窄 ctx->streaming
```
`pgoutput` 在 startup 中做的是：
```text
分配 PGOutputData
创建 logical replication output context
创建 cache context
创建 publication list context
设置 ctx->output_plugin_private
设置 opt->output_type = OUTPUT_PLUGIN_BINARY_OUTPUT
解析 proto_version / publication_names / binary / messages / streaming / two_phase / origin
注册 publication 和 relation cache invalidation callback
初始化 RelationSyncCache
```
如果 `is_init = true`，`pgoutput` 会禁用 streaming 和 two-phase。 因为 slot 初始化阶段只是在找一致性点，不应该按协议 stream 用户事务。
startup 可以改变输出 surface。 比如：
```text
opt->output_type
opt->receive_rewrites
ctx->streaming
ctx->twophase_opt_given
```
但 startup 不负责证明 `SNAPBUILD_CONSISTENT`。 startup 也不负责判断某个 WAL record 是否属于 committed transaction。
这些仍然由 `SnapBuild`、`decode.c` 和 `reorderbuffer.c` 完成。
## 8. `logical.c` wrapper 为什么存在
`ReorderBuffer` 不直接调用插件函数。 `StartupDecodingContext()` 把 callback 指向 wrapper：
```text
ctx->reorder->begin = begin_cb_wrapper
ctx->reorder->apply_change = change_cb_wrapper
ctx->reorder->apply_truncate = truncate_cb_wrapper
ctx->reorder->commit = commit_cb_wrapper
ctx->reorder->message = message_cb_wrapper
```
streaming 和 two-phase 也接到 wrapper。 这一层 wrapper 的职责不是改变 change。 它负责给插件建立受控的输出环境。
每个 wrapper 都会设置 error context。 错误信息会带上：
```text
slot name
output plugin name
callback name
associated LSN
```
这就是你在日志里看到类似：
```text
in the change callback, associated LSN ...
```
的来源。 wrapper 还会设置：
```text
ctx->accept_writes
ctx->write_xid
ctx->write_location
ctx->end_xact
```
`begin_cb_wrapper()` 使用 `txn->first_lsn` 作为 report location。 `change_cb_wrapper()` 使用 `change->lsn`。 `truncate_cb_wrapper()` 使用 `change->lsn`。
`message_cb_wrapper()` 使用 `message_lsn`。 `commit_cb_wrapper()` 的 error report 使用 `txn->final_lsn`。 但它把 `ctx->write_location` 设置成 `txn->end_lsn`。
这个差异很重要。 错误定位通常指向 commit record 开始。 输出确认位置通常要指向 commit record 结束。
这就是为什么 commit callback 中写出的最后消息关联的是事务结束后的 LSN。
## 9. `OutputPluginPrepareWrite()` / `OutputPluginWrite()` 是唯一写出口
插件不会直接把数据写到 socket。 插件也不会直接往 SQL tuplestore 塞 row。 它把字节追加到 `ctx->out`。 然后调用：
```text
OutputPluginPrepareWrite(ctx, last_write)
OutputPluginWrite(ctx, last_write)
```
`OutputPluginPrepareWrite()` 先检查：
```text
ctx->accept_writes
```
只有 begin、change、truncate、message、commit、prepare、stream 等允许输出的 callback 才能调用。
filter callback、startup callback、shutdown callback、progress callback 都不能写普通输出。 然后它调用 context 中的 writer prepare callback：
```text
ctx->prepare_write(ctx, ctx->write_location, ctx->write_xid, last_write)
```
`OutputPluginWrite()` 检查：
```text
ctx->prepared_write
```
如果插件没有先调用 prepare，就会 `ERROR`：
```text
OutputPluginPrepareWrite needs to be called before OutputPluginWrite
```
然后它调用：
```text
ctx->write(ctx, ctx->write_location, ctx->write_xid, last_write)
```
SQL SRF 和 walsender 的 writer 不同。 SQL SRF 在 `logicalfuncs.c` 中把 `ctx->out` 转成 tuplestore row。
walsender 在 `walsender.c` 中把 `ctx->out` 包装成 `PqReplMsg_WALData`。 `WalSndPrepareWrite()` 会设置 `dataStart` 和 `walEnd`。
如果 `last_write = false`，它把 LSN 置为 `InvalidXLogRecPtr`，避免 sync rep 被同一个 LSN 多次混淆。
`WalSndWriteData()` 才真正把 CopyData 放入 libpq 输出缓冲。 所以 plugin callback 的写语义是：
```text
构造一条 output message
  -> 交给 consumer-specific writer
  -> writer 决定写到 SQL result 还是 replication socket
```
这也解释了为什么同一个 output plugin API 能服务 SQL 函数、walsender 和其他内嵌消费者。
## 10. 主流程 walkthrough：committed transaction 如何到达 plugin
主流程从 commit record 开始。 `decode.c` 的 `DecodeCommit()` 先调用：
```text
SnapBuildCommitTxn()
```
然后判断该事务是否需要跳过。 跳过原因包括 database 不匹配、起点前事务、origin filter 等。
如果不跳过，它先把 surviving subtransaction 告诉 reorder buffer：
```text
ReorderBufferCommitChild()
```
然后普通事务走：
```text
ReorderBufferCommit()
  -> ReorderBufferReplay()
  -> ReorderBufferProcessTXN()
```
`ReorderBufferReplay()` 会把事务元信息放入 `txn`：
```text
txn->final_lsn = commit_lsn
txn->end_lsn = end_lsn
txn->commit_time = commit_time
txn->origin_id = origin_id
txn->origin_lsn = origin_lsn
```
如果 `txn->base_snapshot == NULL`，说明没有需要解码的数据库 change。 这个事务可能只对 logical decoding 没有输出价值。
`ReorderBufferReplay()` 会清理或返回。 如果有 base snapshot，就进入 `ReorderBufferProcessTXN()`。 这里才是真正的 callback 调用链。
它先构造 tuple cid hash：
```text
ReorderBufferBuildTupleCidHash()
```
再安装 historic snapshot：
```text
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
然后启动一个内部事务。 如果当前已经在 SQL transaction 中，它用 internal subtransaction。 否则它用 `StartTransactionCommand()`。
这样做是因为 output plugin 可能访问 syscache、打开 relation、执行 row filter expression、读 catalog。 这些都需要 transaction state 和 ResourceOwner。
接着非 streaming 普通事务先调用：
```text
rb->begin(rb, txn)
```
也就是 `begin_cb_wrapper()`。 然后 `ReorderBufferIterTXNNext()` 按 LSN merge top transaction 和 subtransaction 的 change。 对每个 change，它根据 action 分派。
INSERT / UPDATE / DELETE 路径会：
```text
RelidByRelfilenumber()
RelationIdGetRelation()
RelationIsLogicallyLogged()
ReorderBufferToastReplace()
ReorderBufferApplyChange()
RelationClose()
```
`ReorderBufferApplyChange()` 根据是否 streaming 选择：
```text
rb->apply_change()
rb->stream_change()
```
普通路径最终到 `change_cb_wrapper()`。 TRUNCATE 路径会打开 relation 数组。 过滤掉非 logically logged relation。 然后调用：
```text
ReorderBufferApplyTruncate()
```
普通路径最终到 `truncate_cb_wrapper()`。 MESSAGE 路径调用：
```text
ReorderBufferApplyMessage()
```
普通路径最终到 `message_cb_wrapper()`。 所有 change 处理完成后，非 streaming 普通事务调用：
```text
rb->commit(rb, txn, commit_lsn)
```
也就是 `commit_cb_wrapper()`。 最后 `ReorderBufferProcessTXN()` 做清理。 它检查 output plugin 没有分配 XID：
```text
GetCurrentTransactionIdIfAny() == InvalidTransactionId
```
然后 teardown historic snapshot。 然后 `AbortCurrentTransaction()`。 注意这里是 abort 内部事务。 这不是 abort 被解码的用户事务。
它只是为了释放 callback 期间获取的锁、资源和 catalog access 状态，并保证插件不会产生持久数据库写入。 之后执行 invalidation。
最后清理 reorder buffer transaction。 这条主线说明：
```text
plugin callback 运行在一个 core 创建的受控 internal transaction 中；
callback 输出的是 committed transaction 的表示；
callback 自己不是提交判定者。
```
## 11. begin callback：事务输出边界，不是事务开始判定
`begin_cb` 的签名是：
```text
LogicalDecodeBeginCB(ctx, txn)
```
它在 `ReorderBufferProcessTXN()` 中被调用。 这个调用发生在 commit record 已经到达之后。 所以 BEGIN callback 不是“数据库事务刚开始”的通知。
它是“core 准备把一个已经通过 commit 边界的事务输出给插件”的通知。 `test_decoding` 默认在 begin callback 立即写：
```text
BEGIN xid
```
如果启用 `skip-empty-xacts`，它不会立即写 BEGIN。 它等第一条 change 或 message 到来时再写。 `pgoutput` 更进一步。
`pgoutput_begin_txn()` 只分配 `PGOutputTxnData`。 它不发送 protocol BEGIN。 真正发送在 `pgoutput_send_begin()`。
而 `pgoutput_send_begin()` 只会在确认有要发布的 change 后调用。 这样 publication 外的 empty transaction 就不会产生 BEGIN/COMMIT 对。
这说明 begin callback 可以改变外部消息形态。 它不能改变以下事实：
```text
事务已经通过 DecodeCommit()
subtransaction 已经 merge
abort 事务不会进入普通 begin/change/commit 链路
historic snapshot 已经由 core 安装
```
所以 begin callback 是格式和协议边界。 它不是 correctness 边界。
## 12. change callback：格式、过滤和投影，不是 WAL 解释
`change_cb` 的签名是：
```text
LogicalDecodeChangeCB(ctx, txn, relation, change)
```
它收到的是已经被 core 打开的 `Relation`。
它收到的是已经经过 toast 重组、relation lookup、historic snapshot 和 logical logging 检查的 `ReorderBufferChange`。
`change->action` 只可能是 INSERT、UPDATE 或 DELETE 这类用户可见 DML。 内部 snapshot、command id、tuplecid、speculative insertion 等不会直接暴露给插件。
`test_decoding` 的 `pg_decode_change()` 很适合看最小模型。 它做的事情是：
```text
必要时输出 BEGIN
读取 relation name 和 tuple descriptor
切到插件私有 memory context
OutputPluginPrepareWrite()
根据 change->action 拼文本
MemoryContextReset(data->context)
OutputPluginWrite()
```
它几乎不改变语义。 它只是把 tuple image 转成文本。 `pgoutput_change()` 则复杂得多。 它先判断：
```text
is_publishable_relation(relation)
```
然后读取 `RelationSyncEntry`。 这个 entry 包含：
```text
pubactions
row filter ExprState
publish_as_relid
AttrMap
column list
schema_sent
streamed_txns
```
`pgoutput` 会按 publication action 过滤 INSERT / UPDATE / DELETE。 DELETE 如果没有 old tuple，它可能直接跳过。
如果 publication 使用 partition root，它会把 partition tuple 转成 ancestor schema。 它会执行 row filter。
UPDATE 可能因为 old/new 是否满足 filter 而转成 INSERT 或 DELETE。 它会在第一次相关 change 前发送 relation schema。
最后调用 `logicalrep_write_insert()`、`logicalrep_write_update()` 或 `logicalrep_write_delete()`。 这些函数在 `proto.c` 中写 protocol bytes。
从 core 角度看，`pgoutput_change()` 的过滤是 destination-level filtering。 它不是把 WAL 解释成 change 的地方。 它也不是保证 commit ordering 的地方。
如果它跳过一条 change，外部 consumer 看不到这条 change。 但 core 仍然会推进这个事务的 replay 流程。
## 13. truncate callback：可选事件输出
TRUNCATE 在 reorder buffer 中是 `REORDER_BUFFER_CHANGE_TRUNCATE`。 `ReorderBufferProcessTXN()` 会把 relid 数组打开成 `Relation` 数组。 然后调用：
```text
apply_truncate
```
普通路径最终进入 `truncate_cb_wrapper()`。 wrapper 先检查：
```text
if (!ctx->callbacks.truncate_cb)
    return;
```
所以缺少 truncate callback 时，TRUNCATE 事件被插件层忽略。 这不代表 decoder 没有正确理解事务。 它只代表该插件不输出 TRUNCATE。
`test_decoding` 会输出：
```text
table ...: TRUNCATE: restart_seqs cascade
```
`pgoutput` 会逐个 relation 检查：
```text
is_publishable_relation()
relentry->pubactions.pubtruncate
publish_via_partition_root
```
只有至少一个 relation 要发布时，它才写 protocol truncate message。 如果 transaction 中只有不发布的 TRUNCATE，`pgoutput` 可能连 BEGIN 都不发送。
这再次说明：
```text
callback 可以选择 external visibility；
core 已经保证 transaction visibility。
```
## 14. message callback：transactional 与 non-transactional 的边界
logical message 不是普通 heap change。 `decode.c` 的 `logicalmsg_decode()` 会解析 message record。 如果 message 是 transactional，它被排入 reorder buffer。
它会在事务 commit 后通过 `message_cb_wrapper()` 输出。 如果 message 是 non-transactional，它要求有可用 snapshot。
`ReorderBufferQueueMessage()` 会立即设置 historic snapshot 并调用：
```text
rb->message(rb, txn, lsn, false, prefix, size, message)
```
这意味着 non-transactional message 不等某个事务 commit。 但它仍然通过 same plugin callback 输出。
`message_cb_wrapper()` 如果发现插件没有 `message_cb`，直接 return。 `pgoutput_message()` 还会检查 plugin option：
```text
data->messages
```
默认不发送 message。 `test_decoding` 会把 prefix、size 和 content 写成文本。 streaming 下，`test_decoding` 对 transactional message 不直接显示内容。
它只说明有 streaming message。 原因是 in-progress transaction 未来可能 abort。 这不是 core correctness 的要求。
这是该测试插件避免误导用户的展示选择。
## 15. commit callback：输出确认边界，但不是客户端确认
`commit_cb` 的签名是：
```text
LogicalDecodeCommitCB(ctx, txn, commit_lsn)
```
wrapper 设置：
```text
ctx->accept_writes = true
ctx->write_xid = txn->xid
ctx->write_location = txn->end_lsn
ctx->end_xact = true
```
`txn->final_lsn` 是 commit record 开始。 `txn->end_lsn` 是 commit record 结束后位置。 `commit_lsn` 参数来自 replay path，通常是 commit record 开始位置。
输出消息的确认位置要用结束位置。 `test_decoding` 在 commit callback 中写：
```text
COMMIT xid
```
可选附带 timestamp。 如果 `skip-empty-xacts` 且没有写过 change，它直接 return。 `pgoutput_commit_txn()` 会先检查 `sent_begin_txn`。
如果没有发送过 BEGIN，说明这个事务对该 publication 是 empty transaction。 它调用：
```text
OutputPluginUpdateProgress(ctx, !sent_begin_txn)
```
然后跳过 COMMIT message。 如果已经发送 BEGIN，它才写：
```text
logicalrep_write_commit()
```
commit callback 结束不等于 slot confirmed。 在 SQL `pg_logical_slot_get_changes()` 中，函数尾部如果 `confirm = true`，才调用：
```text
LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
```
在 walsender 中，`WalSndWriteData()` 只是发出 CopyData。 slot 的 `confirmed_flush` 等客户端反馈 flush LSN。 这个边界对诊断非常重要。
插件成功写出 COMMIT 后，如果客户端没有确认，重连可能重复看到同一个事务。 这属于 logical replication 的 at-least-once 交付边界。
## 16. shutdown callback：释放插件状态，不写 change
`FreeDecodingContext()` 会调用：
```text
shutdown_cb_wrapper(ctx)
```
前提是插件注册了 `shutdown_cb`。 shutdown wrapper 设置：
```text
ctx->accept_writes = false
ctx->end_xact = false
```
所以 shutdown callback 不应输出 logical change。 `test_decoding` 在 shutdown 中删除自己的 `"text conversion context"`。
`pgoutput` 在 shutdown 中释放 publication list 和 cache 相关状态。 core 随后会释放：
```text
ReorderBuffer
SnapBuild
XLogReader
Logical decoding context
```
如果插件把私有内存挂在 `ctx->context` 下，即使忘记在 shutdown 中释放，也会随 `MemoryContextDelete(ctx->context)` 回收。
但如果插件注册了额外 callback 或持有非 memory context 资源，shutdown callback 才是清理边界。 shutdown 不改变 slot 位置。
shutdown 也不决定事务是否已经交付。 它只是 plugin session lifecycle 的尾部。
## 17. 哪些 callback 只定义格式，哪些会改变暴露边界
普通非 streaming、非 two-phase 链路中，正确性已经在 callback 前建立。 这一组 callback 主要定义输出格式和外部过滤：
```text
begin_cb
change_cb
truncate_cb
message_cb
commit_cb
shutdown_cb
```
其中 `shutdown_cb` 只做 cleanup。 `message_cb` 和 `truncate_cb` 甚至是可选的。 `change_cb` 可以做 publication、row filter、column list、schema mapping。
这些会改变下游看到什么。 但它不会改变：
```text
WAL record 是否可解码
事务是否 committed
abort subtransaction 是否已剔除
historic snapshot 是否正确
slot restart_lsn 如何推进
```
`begin_cb` 和 `commit_cb` 也一样。 它们可以延迟 BEGIN、跳过 empty transaction、写不同协议。 它们不能把未提交事务输出成 committed transaction。
startup 比较特殊。 它可以设置：
```text
output_type
receive_rewrites
streaming
twophase_opt_given
```
这会改变 callback surface。 比如 `receive_rewrites` 会让 reorder buffer 是否输出 rewrite 期间的临时 heap change 发生变化。
但 startup 仍不负责证明数据正确。 `filter_by_origin_cb` 和 `filter_prepare_cb` 是更接近语义 gate 的 callback。
`filter_by_origin_cb` 会让 `decode.c` 跳过某些 origin 的事务或 change。
`filter_prepare_cb` 会决定 prepare record 是否走 prepare-time decoding，还是等 COMMIT PREPARED。 它们改变的是“这条流对这个 consumer 暴露什么”。
它们仍不改变 core 对 WAL、snapshot 和 commit ordering 的正确性规则。 可以把边界压缩成一句话：
```text
plugin callback 可以选择格式、过滤和协议时机；
core 负责证明 change 是否存在、是否完整、是否 committed、按什么顺序交付。
```
## 18. MemoryContext：session 私有状态与 per-change 临时状态
output plugin 最常见的 bug 是内存生命周期混乱。 `LogicalDecodingContext` 自己有 `ctx->context`。 这个 context 生命周期覆盖整个 decoding session。
插件 session 级对象适合挂在这里。 `test_decoding` 的 `TestDecodingData` 就是 session 级。 它还创建一个 `"text conversion context"`。
每次 change 输出后 reset。 这样 tuple to string 时产生的临时内存不会跨 change 累积。 `pgoutput` 创建多个 context。
`data->context` 用于输出过程中的短期分配。 `data->cachectx` 用于 relation sync cache。 `data->pubctx` 用于 publication list。
`pgoutput_memory_context_reset()` 注册在 `ctx->context` reset callback 上。 目的是 SQL interface ERROR 时也能清理全局 `RelationSyncCache`。
`ReorderBuffer` 也有自己的 context。 `ReorderBufferAllocate()` 创建：
```text
"ReorderBuffer"
  -> "Change" SlabContext
  -> "TXN" SlabContext
  -> "Tuples" GenerationContext
```
这些内存属于 core。 插件不应该保存 `ReorderBufferChange *` 或 tuple 指针到 callback 返回后继续使用。
因为 change 对象和 tuple image 会随 reorder buffer cleanup、truncate 或 spill restore 被释放或复用。
如果插件需要跨 callback 保存信息，要复制到自己的 context。 `ReorderBufferTXN.output_plugin_private` 是 transaction 级挂钩。
它可以在 begin callback 分配，在 commit callback 释放。 `pgoutput_commit_txn()` 就会 `pfree(txndata)` 并把指针清空。 streaming 和 abort 路径更复杂。
如果插件支持 streaming，就必须考虑同一个 transaction 可能分块输出。 transaction 私有状态不能假设 begin/change/commit 只调用一次连续序列。
## 19. transaction boundary：插件在内部事务中运行
`ReorderBufferProcessTXN()` 的内部事务设计很容易被忽略。 logical decoding 需要读 historic catalog。 `pgoutput` 还可能构造 executor state 执行 row filter。
这些操作需要：
```text
CurrentResourceOwner
syscache
relation open/close
locks
memory context
snapshot stack
```
所以 core 在调用 callback 前启动内部 transaction 或 subtransaction。 调用结束后，它会：
```text
TeardownHistoricSnapshot(false)
AbortCurrentTransaction()
执行 invalidations
RollbackAndReleaseCurrentSubTransaction() 或恢复 CurrentResourceOwner
```
这里的 abort 是 cleanup 手段。 它不代表用户事务 abort。 相反，用户事务已经是 committed transaction。 内部事务 abort 的目的包括：
```text
释放 callback 期间打开 relation 获得的资源
释放锁和 pins
防止 output plugin 对数据库做持久写入
清理 syscache 状态
```
源码还会检查插件是否分配了 XID。 如果 callback 中有操作导致 `GetCurrentTransactionIdIfAny()` 返回有效 XID，core 会报：
```text
output plugin used XID ...
```
这个检查体现了插件边界：
```text
output plugin 可以读 catalog；
output plugin 不应该写数据库状态。
```
## 20. ERROR 路径：callback 失败如何收尾
每个 wrapper 都把 `output_plugin_error_callback` 挂到 `error_context_stack`。 所以插件内部 `ereport(ERROR)` 时，日志能定位 callback 名和相关 LSN。
`ReorderBufferProcessTXN()` 也有 `PG_TRY()` / `PG_CATCH()`。 普通 ERROR 进入 catch 后会：
```text
结束 iterator
TeardownHistoricSnapshot(true)
AbortCurrentTransaction()
执行 invalidations 或 InvalidateSystemCaches()
恢复 subtransaction / ResourceOwner
ReorderBufferCleanupTXN()
PG_RE_THROW()
```
这保证插件 ERROR 不会让 historic snapshot 和内部 transaction 泄漏。 但它不会假装输出成功。
SQL interface 中，`pg_logical_slot_get_changes_guts()` 只有正常完成并且 `confirm = true` 时才调用：
```text
LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
```
如果 callback ERROR，中途跳出，就不会执行这个确认。 peek 函数本来就传 `confirm = false`。
walsender 中，`XLogSendLogical()` 在成功处理 record 后更新 `sentPtr`。 但是 `confirmed_flush` 的推进在另一条路径：
```text
客户端 status update
  -> ProcessRepliesIfAny()
  -> LogicalConfirmReceivedLocation(flushPtr)
```
所以 plugin ERROR 对 slot 位置的影响要分开看。 当前 backend 可能已经读过 WAL。 当前 socket 可能已经写出部分消息。
但 slot 持久确认位置通常不会因为“读过”或“尝试写过”自动前进。
如果客户端没有确认到某个 commit LSN，重连后可能再次收到同一事务。
这就是为什么 logical consumer 必须具备幂等或基于 LSN/transaction 的去重能力。
## 21. missing callback 与错误 fallback
缺少 mandatory callback 是启动时错误。 `LoadOutputPlugin()` 会检查：
```text
begin_cb
change_cb
commit_cb
```
缺一项就 `ERROR`。 缺少 optional callback 不是错误。 普通 `truncate_cb` 缺失时，TRUNCATE 不输出。 普通 `message_cb` 缺失时，message 不输出。
streaming 下 `stream_message_cb` 和 `stream_truncate_cb` 也可选。
但 streaming 模式启用后，如果缺少 `stream_start_cb`、`stream_stop_cb`、`stream_abort_cb`、`stream_commit_cb` 或 `stream_change_cb`，wrapper 会在实际调用时 `ERROR`。
two-phase 也一样。
启用 prepare-time decoding 后，如果缺少 `begin_prepare_cb`、`prepare_cb`、`commit_prepared_cb` 或 `rollback_prepared_cb`，对应 wrapper 会 `ERROR`。
这不是风格问题。 streaming 和 two-phase 会让 downstream 看到 in-progress 或 prepared transaction 的中间状态。
一旦插件声明要进入这种模式，core 必须能发送补偿消息：
```text
stream_abort
stream_commit
stream_prepare
rollback_prepared
```
否则下游无法知道已经收到的 partial changes 应该 apply 还是 discard。
## 22. streaming callbacks：提前输出不等于提前提交
大事务可能超过 `logical_decoding_work_mem`。 reorder buffer 可以 spill 到磁盘。
如果插件支持 streaming，还可以把 in-progress transaction 的 change 分块发给下游。 streaming 主线是：
```text
stream_start
  -> stream_change / stream_message / stream_truncate
  -> stream_stop
  -> later stream_commit or stream_abort
```
`ReorderBufferStreamTXN()` 和 `ReorderBufferProcessTXN(..., streaming=true)` 负责分块 replay。
streaming chunk 中，`stream_start_cb_wrapper()` 设置 `write_location` 为 first change LSN。 `stream_change_cb_wrapper()` 用每条 change 的 LSN。
`stream_stop_cb_wrapper()` 用 last LSN。 但这些位置仍不足以确认整个事务。 注释里也明确说：
```text
This won't ever be enough to confirm receipt of this transaction.
```
原因是 transaction 未来可能 abort。 因此下游必须把 stream changes 暂存起来。 直到收到：
```text
stream_commit
```
才能 apply。 如果收到：
```text
stream_abort
```
必须丢弃已经暂存的 change。 `test_decoding` 在 streaming change 中不输出 tuple 细节。 它只写“streaming change for TXN ...”。
这是测试插件的展示策略。 `pgoutput` 则会写 logical replication stream protocol。 它用 `data->in_streaming` 控制 change message 中是否带 XID。
streaming callback 改变 delivery timing。 它没有改变 commit correctness。 commit correctness 仍然由 commit/abort record 和 reorder buffer 决定。
## 23. two-phase callbacks：prepare-time decoding 的额外边界
two-phase logical decoding 让插件可以在 `PREPARE TRANSACTION` 时输出事务内容。 相关 callback 包括：
```text
begin_prepare_cb
prepare_cb
commit_prepared_cb
rollback_prepared_cb
filter_prepare_cb
stream_prepare_cb
```
`decode.c` 在看到 prepare record 时会先保存 prepare 信息：
```text
ReorderBufferRememberPrepareInfo()
```
然后根据 snapshot state、database/origin/filter 判断是否需要 prepare-time decoding。 `filter_prepare_cb` 可以按 GID 决定是否过滤。
`test_decoding` 示例里，如果 GID 包含 `_nodecode`，就过滤 prepare-time decoding。
如果走 prepare-time decoding，`ReorderBufferPrepare()` 会 replay 事务 change。 此时 callback 链是：
```text
begin_prepare
change / truncate / message
prepare
```
真正 `COMMIT PREPARED` 到达时，再调用：
```text
commit_prepared
```
`ROLLBACK PREPARED` 到达时调用：
```text
rollback_prepared
```
这里的正确性风险比普通事务更高。 prepared transaction 已经在上游处于 prepared 状态。 下游必须区分：
```text
prepared but not committed
committed prepared
rolled back prepared
```
所以 two-phase callback 不是普通 begin/commit 的简单别名。 它改变了外部协议状态机。 但 WAL 解释、snapshot、一致性和 subxact merge 仍在 core。
## 24. `test_decoding` 与 `pgoutput` 的对照
`test_decoding` 是学习 callback 生命周期的最小插件。 它的输出类型是：
```text
OUTPUT_PLUGIN_TEXTUAL_OUTPUT
```
它关注：
```text
include-xids
include-timestamp
skip-empty-xacts
only-local
include-rewrites
stream-changes
```
它基本不维护 relation publication cache。 它每条 change 都把 relation name 和 tuple 转成文本。 它的 memory model 简单：
```text
TestDecodingData
  -> text conversion context
TestDecodingTxnData
  -> xact_wrote_changes
  -> stream_wrote_changes
```
`pgoutput` 是生产逻辑复制协议插件。 它的输出类型是：
```text
OUTPUT_PLUGIN_BINARY_OUTPUT
```
它必须解析：
```text
proto_version
publication_names
binary
messages
streaming
two_phase
origin
```
它维护 `RelationSyncEntry`。 它会处理：
```text
publication actions
row filters
column lists
generated columns
partition root publishing
schema cache invalidation
relation/type protocol messages
```
`test_decoding` 展示的是 callback API 最小语义。 `pgoutput` 展示的是同一 API 承载真实复制协议时的复杂性。 两者共同说明：
```text
插件可以非常薄，也可以很厚；
厚的是输出协议和过滤逻辑，不是 WAL decoding correctness。
```
## 25. `pgoutput` 的 relation cache 为什么属于插件层
`pgoutput` 的 `RelationSyncEntry` 保存 relation 是否发布、发布哪些动作、使用哪个 schema、哪些列、哪些 row filter。 这些信息来自 catalog。
它们会被 cache invalidation 影响。 `pgoutput_startup()` 注册：
```text
CacheRegisterSyscacheCallback(PUBLICATIONOID, ...)
CacheRegisterRelSyncCallback(...)
```
`get_rel_sync_entry()` 在需要时构造或刷新 entry。 `maybe_send_schema()` 决定 relation schema 是否已经发送。
`send_relation_and_attrs()` 写 type 和 relation protocol message。 这些行为不是 `decode.c` 能做的。
`decode.c` 不知道某个 downstream 订阅了哪些 publication。 `decode.c` 也不应该理解 logical replication protocol version。
因此 relation cache 放在 plugin 层。 但 relation 是否能被 historic snapshot 正确打开，仍由 core 提供。
`ReorderBufferProcessTXN()` 负责安装 historic snapshot。 `pgoutput` 在这个 snapshot 语境里读 catalog。
如果插件离开 callback 后继续用旧 relation 指针，就越过了生命周期边界。
## 26. `receive_rewrites`：startup option 的一个真实影响
`OutputPluginOptions` 有两个字段：
```text
output_type
receive_rewrites
```
`output_type` 决定 SQL 函数能否用 text 或 binary 接收。 `receive_rewrites` 更接近 decoding exposure。
`ReorderBufferProcessTXN()` 中遇到 relation rewrite 时，会检查：
```text
relation->rd_rel->relrewrite
rb->output_rewrites
```
`ctx->reorder->output_rewrites` 来自：
```text
ctx->options.receive_rewrites
```
`test_decoding` 默认 `false`，但支持 `include-rewrites` 选项。 这说明 startup callback 不是纯显示设置。 它能让某些特殊 change 暴露给 plugin。
但即便如此，rewrite change 的 WAL 解释、snapshot 和事务边界仍由 core 负责。 插件只是选择是否接收这类已解码 change。
## 27. origin 与 prepare filter：真正的插件 gate
`filter_by_origin_cb` 和 `filter_prepare_cb` 是 callback 表中最容易被误解的两项。 它们不是普通 output formatter。
`decode.c` 的 `FilterByOrigin()` 会在 heap、truncate、message、commit 等路径上检查 origin。
如果插件返回需要过滤，相关 change 或 transaction 可能不会进入后续输出。
`pgoutput_origin_filter()` 根据 `origin` option 判断是否只发布 no-origin changes。 `test_decoding` 的 `pg_decode_filter()` 根据 `only-local` 过滤 origin。
`filter_prepare_cb` 则影响 prepare-time decoding。 它不会改变 prepared transaction 在 WAL 中的事实。
它只决定这个 output plugin 是否在 prepare 阶段输出内容。 这些 filter 是 plugin 参与语义裁剪的地方。
但它们仍不是 transaction correctness 的源头。 core 仍然负责在跳过事务时处理必要 invalidation，避免 catalog cache 污染。
## 28. 成本模型：callback 厚度会迁移瓶颈
output plugin 在 logical decoding hot path 上。 每个 committed transaction 都要经过 callback。 成本来源至少有四类。 第一是 per-change CPU。
`test_decoding` 的 tuple-to-string 会 deform tuple、查 type output、拼接文本。 这对诊断友好，但不是生产高效协议。
`pgoutput` 的 binary option 可以减少一部分文本转换成本。 第二是 catalog 和 cache。 `pgoutput` 每个 relation 首次出现时可能构造 `RelationSyncEntry`。
它可能发送 relation schema 和 type message。 publication invalidation 后还要重新计算。 第三是 row filter 和 column list。
row filter 会构造 executor state，执行 expression。 UPDATE 还可能评估 old/new 两侧，甚至改变输出 action。 第四是 large transaction。
如果没有 streaming，大事务可能 spill 到磁盘，commit 时再 replay。 如果有 streaming，callback 会更早、更频繁地被调用。
下游需要暂存 partial changes。 成本随这些变量扩张：
```text
事务内 change 数
涉及 relation 数
publication 数
row filter 复杂度
tuple 宽度和 toasted datum 数
schema cache invalidation 频率
streaming chunk 数
client ack 延迟
```
不要把 logical decoding 性能问题只归因于 WAL 读取。
很多系统中，瓶颈会迁移到 plugin formatting、publication filtering、网络 backpressure 或 downstream ack。
## 29. 观测入口：能看到什么，看不到什么
能直接看见的状态包括：
```sql
SELECT slot_name, plugin, restart_lsn, confirmed_flush_lsn, catalog_xmin
FROM pg_replication_slots
WHERE slot_name = '...';
```
这里能看 slot 确认位置和保留边界。 它不能告诉你某个 callback 是否已经执行。 walsender 路径可以看：
```sql
SELECT pid, state, sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;
```
`sent_lsn` 更接近 walsender 已发送位置。 `flush_lsn` 来自客户端反馈。 logical slot 的 `confirmed_flush_lsn` 会随客户端 flush feedback 推进。
日志能看到 plugin ERROR 的 callback context。 如果 wrapper 设置了 associated LSN，你能定位大概哪条 change 或 commit 出错。
`pg_stat_replication_slots` 能看到 spill/stream 相关累计统计。 但它不告诉你每条 message 为什么被 publication filter 掉。 这种问题通常要结合：
```text
publication 定义
relation replica identity
row filter
column list
plugin options
断点或临时日志
```
源码断点推荐：
```text
LoadOutputPlugin
startup_cb_wrapper
begin_cb_wrapper
change_cb_wrapper
truncate_cb_wrapper
message_cb_wrapper
commit_cb_wrapper
OutputPluginPrepareWrite
OutputPluginWrite
ReorderBufferProcessTXN
pgoutput_change
pg_decode_change
LogicalConfirmReceivedLocation
```
如果你看到 slot 不前进，不要先怀疑 `change_cb`。 先区分：
```text
插件是否输出了消息？
consumer writer 是否成功？
walsender 是否收到 flush feedback？
SQL get_changes 是否 confirm=true？
callback 是否 ERROR？
```
## 30. 常见误区
误区一：把 `change_cb` 理解成 WAL decode callback。 真实 WAL decode callback 在 rmgr 层，比如 `heap_decode()`、`xact_decode()`、`logicalmsg_decode()`。
`change_cb` 收到的是 reorder buffer 已经准备好的 relation-level change。 误区二：以为 `begin_cb` 在事务开始时调用。
普通非 streaming 路径中，begin callback 在 commit record 之后、事务 replay 输出前调用。 误区三：以为 `OutputPluginWrite()` 推进 slot。
它只调用 consumer writer。 slot `confirmed_flush` 需要 SQL confirm 或 replication feedback。 误区四：以为可选 callback 缺失是 decoding failure。
`message_cb` 和 `truncate_cb` 缺失只表示插件不输出这类事件。 误区五：把 plugin private data 当成跨 transaction 永久安全对象。
session 级状态可以放 `ctx->output_plugin_private`。 transaction 级状态应放 `txn->output_plugin_private` 并在 commit/abort/stream 结束时清理。
change 指针不能跨 callback 长期保存。 误区六：以为 `pgoutput` 的 publication filter 属于 `decode.c`。
`decode.c` 做 database/origin/snapshot/transaction 边界。 publication、row filter、column list 和 schema protocol 属于 `pgoutput`。
## 31. 课堂实验一：跟一次 `test_decoding` callback
准备：
```sql
SELECT * FROM pg_create_logical_replication_slot('cb1', 'test_decoding');
CREATE TABLE cb1_t(id int primary key, note text);
INSERT INTO cb1_t VALUES (1, 'a');
SELECT * FROM pg_logical_slot_get_changes('cb1', NULL, NULL);
```
断点：
```text
startup_cb_wrapper
pg_decode_startup
begin_cb_wrapper
pg_decode_begin_txn
change_cb_wrapper
pg_decode_change
commit_cb_wrapper
pg_decode_commit_txn
OutputPluginPrepareWrite
OutputPluginWrite
```
观察：
```text
startup 的 accept_writes 为 false
begin/change/commit 的 accept_writes 为 true
change 的 write_location 是 change->lsn
commit 的 write_location 是 txn->end_lsn
txn->output_plugin_private 在 begin 分配，在 commit 释放
```
把观察结果画成：
```text
core replay state
  -> wrapper state
  -> plugin state
  -> writer state
```
不要只记录函数名。 重点记录每一步改变了哪个状态。
## 32. 课堂实验二：比较 `test_decoding` 和 `pgoutput`
先用 `test_decoding` 观察文本。 再创建 publication：
```sql
CREATE PUBLICATION p_cb FOR TABLE cb1_t;
```
用 logical replication protocol 或测试客户端启动 `pgoutput`，传入：
```text
proto_version
publication_names
```
源码断点：
```text
pgoutput_startup
parse_output_parameters
pgoutput_begin_txn
pgoutput_change
maybe_send_schema
logicalrep_write_insert
pgoutput_commit_txn
```
对比两个插件：
```text
startup 解析哪些 option
begin 是否立即输出
change 是否检查 publication
是否发送 schema
commit 是否跳过 empty transaction
输出类型是 text 还是 binary
```
实验目标不是学 protocol 字节。 目标是确认：
```text
同一个 core callback 生命周期，可以承载完全不同的 external representation。
```
## 33. 课堂实验三：验证 ERROR 与 slot 位置
写一个临时测试插件，或在 `test_decoding` 的 `pg_decode_change()` 中人为对某个 relation `ereport(ERROR)`。 执行：
```sql
SELECT confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'cb1';
```
然后触发：
```sql
SELECT * FROM pg_logical_slot_get_changes('cb1', NULL, NULL);
```
观察 ERROR 后再次查询 `confirmed_flush_lsn`。 再去掉 ERROR，重新消费。 你应看到同一段未确认事务仍可能再次输出。 用源码解释：
```text
wrapper 提供 error context
ReorderBufferProcessTXN() 清理 snapshot 和 internal transaction
SQL 函数没有走到 LogicalConfirmReceivedLocation()
slot confirmed_flush_lsn 不会因已经读 WAL 自动前进
```
walsender 版本的实验可以用一个会断开连接的客户端。 重点看 `pg_replication_slots.confirmed_flush_lsn` 与客户端反馈的关系。
不要把 `sent_lsn` 当成 `confirmed_flush_lsn`。
## 34. 讨论题
1. 为什么 `LoadOutputPlugin()` 只强制 begin/change/commit，而不强制 startup/message/truncate/shutdown？
2. `pgoutput_begin_txn()` 为什么不立即发送 BEGIN？
3. `commit_cb_wrapper()` 为什么 error report 用 `txn->final_lsn`，但 `write_location` 用 `txn->end_lsn`？
4. output plugin 在 callback 里为什么可以读 catalog，却不应该产生 XID？
5. 如果 `message_cb` 缺失，transactional logical message 会发生什么？这和 abort 事务被丢弃有什么本质区别？
6. streaming change 已经发给下游后，上游事务 abort，为什么必须有 `stream_abort_cb`？
7. `filter_by_origin_cb` 与 publication filter 的边界在哪里？
8. plugin ERROR 后，为什么不能用“decoder 已经读到的 LSN”判断 slot 安全推进？
## 35. 本节小结
本节的核心链路是：
```text
output plugin 注册 callback table
  -> startup 设置 output options 和 plugin private data
  -> reorder buffer 在 committed transaction replay 中调用 wrapper
  -> wrapper 设置 write state 和 error context
  -> plugin 格式化或过滤输出
  -> OutputPluginPrepareWrite/Write 交给 SQL SRF 或 walsender
  -> consumer 确认后 slot confirmed_flush 才推进
  -> shutdown 清理 plugin session 状态
```
核心状态是 `LogicalDecodingContext`、`OutputPluginCallbacks`、`ReorderBufferTXN.output_plugin_private` 和 `ctx->out`。
`ctx->output_plugin_private` 是 session 级插件状态。 `txn->output_plugin_private` 是事务级插件状态。
`ctx->accept_writes`、`write_location`、`write_xid` 和 `end_xact` 由 wrapper 控制。
普通 begin/change/truncate/message/commit callback 主要定义输出格式、过滤和协议时机。
它们不负责 WAL record 解释、snapshot consistency、subxact merge、abort filtering 或 commit ordering。
startup 可以选择 output type、rewrite exposure、streaming 和 two-phase option。 filter callback 可以裁剪 origin 或 prepare-time decoding。
但 core correctness 仍在 `decode.c`、`snapbuild.c` 和 `reorderbuffer.c`。
ERROR 路径中，wrapper 提供可诊断上下文，`ReorderBufferProcessTXN()` 清理 historic snapshot、internal transaction 和 resource owner。
slot 位置不因插件读到 WAL 自动推进。 SQL `get_changes` 要正常结束并 confirm。 walsender 要等客户端 flush feedback。 可迁移的系统规律是：
```text
一个可扩展 callback API 必须把“谁证明状态正确”和“谁决定外部表示”分开；
否则插件生态会把 protocol 灵活性、filter 需求和内核 correctness 绑死在一起。
```
判断 logical decoding 问题时，先问：
```text
问题发生在 WAL -> reorder buffer 的 correctness 层，
还是发生在 plugin output/filter/protocol 层，
还是发生在 consumer confirmation 层？
```
这三个层次对应的源码、指标和修复方向完全不同。
