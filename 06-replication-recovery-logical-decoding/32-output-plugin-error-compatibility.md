# PostgreSQL output plugin ERROR 与消费者兼容性边界
## 课程定位
前置知识：
已经理解 logical slot 的 `restart_lsn`、`confirmed_flush_lsn` 和 `catalog_xmin`。
已经理解 reorder buffer 如何把 WAL record 重组成事务级 logical stream。
已经理解上一节 2PC 和 streaming 把普通事务边界拆成 prepare、stream chunk、stream commit / abort。
本节唯一主问题：
```text
plugin 在输出过程中 ERROR 会如何影响 slot 消费位置，
格式升级、schema 变更和消费者兼容性应该在哪些边界上显式处理？
```
核心矛盾：
output plugin 一边把内部 WAL / catalog / tuple 状态翻译成外部协议，
一边又运行在 PostgreSQL backend 的 ERROR 语义和 replication slot 确认语义之内。
网络已经写出的字节不能被事务 abort 收回。
slot 位置又不能因为 plugin 试图输出就自动前进。
消费者格式升级、schema 变化和 apply 语义也不能靠“下次重连再猜”来修复。
所以本节的 tension 是：
```text
输出过程可以失败、可以重试、可以重复发送
  vs
slot 只能在消费者明确确认持久接收之后推进，
格式和 schema 兼容性必须在协议边界上显式握手。
```
学完后应能判断：
一个 output plugin callback 抛出 ERROR 时，哪些本地 decoding 状态会被清理。
`ctx->write_location` 只是当前输出消息关联的 LSN，不是 slot 消费位置。
`confirmed_flush_lsn` 只由 SQL get 路径或 walsender feedback 路径推进。
消费者未确认的输出会在重连或重试后再次出现。
protocol version、output option、relation schema 和消费者 apply 进度应分别在哪一层处理。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面第 27 节回答了：
```text
读取位置、输出位置和 confirmed_flush_lsn 为什么不能混为一谈？
```
前面第 28 节回答了：
```text
prepared transaction 和 streaming transaction 为什么需要更细的协议状态？
```
本节继续沿同一条链往外走一步：
```text
WAL record
  -> reorder buffer
  -> logical.c callback wrapper
  -> output plugin
  -> writer callback
  -> SQL tuplestore 或 walsender socket
  -> consumer durable state
  -> feedback / confirm
  -> slot confirmed_flush_lsn
```
本节只关心这条链上的一个问题：
```text
输出阶段失败或格式不兼容时，系统在哪里停止，哪里重试，哪里显式拒绝？
```
不要把它写成“如何写 output plugin API”的清单。
API 本身很小。
真正困难的是这些 API 处在三个状态边界之间。
第一个边界是 PostgreSQL backend 内部 ERROR 边界。
`ReorderBufferProcessTXN()` 会启动内部事务或子事务，设置 historic snapshot，调用 plugin。
ERROR 时它必须撤销这些内部执行状态。
第二个边界是 replication slot 的确认边界。
`OutputPluginWrite()` 可能已经把消息交给 writer。
但 `slot->data.confirmed_flush` 不会因为这件事自动移动。
第三个边界是外部消费者兼容性边界。
consumer 是否懂这个协议版本、tuple 编码、relation metadata、schema change 和幂等策略，不是 backend 能凭空推断的。
所以本节的线性主线是：
```text
先看 output plugin 写出时有哪些状态。
再看 callback wrapper 和 PG_TRY/PG_CATCH 如何收拾 ERROR。
再看 walsender / SQL 如何推进 confirmed_flush_lsn。
最后回到兼容性：版本、schema 和消费者 ACK 应该在哪里显式处理。
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
logical.c 在 begin/change/commit/stream 等 wrapper 中设置 ctx->write_location，
output plugin 调 OutputPluginPrepareWrite()/OutputPluginWrite() 构造并写出消息，
ReorderBufferProcessTXN() 用 PG_TRY/PG_CATCH 清理输出阶段的内部事务和 historic snapshot，
但 slot->data.confirmed_flush 只在 SQL get 成功确认或 walsender 收到 flush feedback 时推进。
```
这个模型里最容易错的是“写出”和“消费”。
`OutputPluginWrite()` 的名字容易让人以为 slot 已经消费。
实际上它只调用 `ctx->write()`。
在 streaming walsender 中，这个 writer 是 `WalSndWriteData()`。
它把当前 `ctx->out` 包进 `CopyData`，尝试写 socket，并在需要时处理 reply。
在 SQL 函数中，这个 writer 是 `LogicalOutputWrite()`。
它把当前输出放进 tuplestore。
两者都不是 slot 的跨会话确认状态。
slot 的跨会话确认状态在：
```text
ReplicationSlotPersistentData.confirmed_flush
```
它只通过这些路径前进：
```text
SQL get 路径:
  pg_logical_slot_get_changes_guts(confirm = true)
    -> LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)

streaming walsender 路径:
  ProcessStandbyReplyMessage()
    -> LogicalConfirmReceivedLocation(flushPtr)

slot advance 路径:
  pg_replication_slot_advance()
    -> LogicalConfirmReceivedLocation(moveto)
```
因此 output plugin ERROR 的直接含义是：
```text
当前 decoding 调用失败；
backend 内部 replay 状态要清理；
尚未被确认的 logical 输出仍可能再次发送；
已经被消费者确认的 LSN 不会因为 ERROR 自动后退。
```
这不是 exactly-once。
PostgreSQL logical decoding 提供的是：
```text
以 LSN 和事务边界为基础的可重放输出流；
consumer 必须把 apply state 和 ACK state 做成可恢复协议。
```
## 3. 核心文件分工与阅读顺序
阅读顺序按状态推进，不按文件名排序。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/logical.h` | `LogicalDecodingContext` 的 writer callbacks、`out`、`write_location`、`write_xid`、`end_xact`。 |
| 2 | `src/include/replication/output_plugin.h` | output plugin callback ABI：startup、begin、change、commit、message、stream、2PC。 |
| 3 | `src/backend/replication/logical/logical.c` | callback wrapper、error context、`OutputPluginPrepareWrite()`、`OutputPluginWrite()`、`LogicalConfirmReceivedLocation()`。 |
| 4 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferProcessTXN()` 的 `PG_TRY/PG_CATCH`、historic snapshot、internal transaction cleanup。 |
| 5 | `src/backend/replication/walsender.c` | `StartLogicalReplication()`、`WalSndPrepareWrite()`、`WalSndWriteData()`、`ProcessStandbyReplyMessage()`。 |
| 6 | `src/backend/replication/logical/logicalfuncs.c` | SQL `get` / `peek` 的确认差异和文本 / 二进制输出检查。 |
| 7 | `src/backend/replication/pgoutput/pgoutput.c` | option negotiation、protocol capability、relation schema cache、invalidation。 |
| 8 | `src/backend/replication/logical/proto.c` | relation、type、tuple、streaming、2PC wire message 编码。 |
| 9 | `contrib/test_decoding/test_decoding.c` | 文本调试 plugin 的输出顺序、选项解析、与 `pgoutput` 的对照。 |
推荐跟读路线：
```text
StartLogicalReplication()
  -> CreateDecodingContext()
     -> StartupDecodingContext()
        -> LoadOutputPlugin()
        -> setup reorder buffer callbacks
        -> startup_cb_wrapper()
  -> XLogBeginRead(ctx->reader, slot->data.restart_lsn)
  -> WalSndLoop(XLogSendLogical)
     -> XLogReadRecord()
     -> LogicalDecodingProcessRecord()
     -> ReorderBufferCommit()
     -> ReorderBufferProcessTXN()
     -> begin/change/commit wrapper
     -> output plugin callback
     -> OutputPluginPrepareWrite()/OutputPluginWrite()
     -> WalSndPrepareWrite()/WalSndWriteData()
  -> downstream reply
     -> ProcessStandbyReplyMessage()
     -> LogicalConfirmReceivedLocation(flushPtr)
```
SQL 路径用来验证重复输出：
```text
pg_logical_slot_get_changes_guts(confirm = true)
  -> 成功结束后确认 ctx->reader->EndRecPtr

pg_logical_slot_peek_changes()
  -> confirm = false
  -> 不推进 confirmed_flush_lsn
```
## 4. 核心状态一：LogicalDecodingContext 的输出字段
`LogicalDecodingContext` 是 backend-local 对象。
它定义在 `src/include/replication/logical.h`。
本节只关心输出相关字段。
| 字段 | 语义 |
| --- | --- |
| `prepare_write` | 用例相关的 prepare writer；SQL 和 walsender 不同。 |
| `write` | 用例相关的真正 writer；负责 tuplestore 或 socket。 |
| `update_progress` | 可选进度回调；walsender 用它处理 keepalive / lag。 |
| `out` | output plugin 填写的临时 `StringInfo`。 |
| `accept_writes` | 当前 callback 是否允许调用 `OutputPluginPrepareWrite()`。 |
| `prepared_write` | 是否已经 prepare 但还没 write。 |
| `write_location` | 当前输出消息关联的 LSN。 |
| `write_xid` | 当前输出消息关联的 XID。 |
| `end_xact` | 当前 callback 是否处在事务结束边界。 |
这些字段都是当前 decoding session 的状态。
它们不是 shared memory。
ERROR 或断线后，下一次会重新创建 context。
`ctx->write_location` 的含义尤其要窄化。
它不是“消费者已经消费到哪里”。
它也不是“decoder 当前读 WAL 到哪里”。
它只是 writer 本次消息应携带的 LSN。
`begin_cb_wrapper()` 把它设成：
```text
txn->first_lsn
```
`change_cb_wrapper()` 把它设成：
```text
change->lsn
```
`commit_cb_wrapper()` 把它设成：
```text
txn->end_lsn
```
streaming change 也用 change LSN。
stream commit / stream prepare 用 transaction end LSN。
这个设计让 walsender 能在 WALData 头部报告一个和 logical message 相关的 LSN。
但客户端是否可以 ACK 这个 LSN，取决于它是否已经持久保存对应输出语义。
PostgreSQL 不会替 consumer 判断“我已经安全 apply 了”。
## 5. 核心状态二：slot confirmed_flush_lsn
slot 的跨会话状态在 replication slot 里。
`confirmed_flush_lsn` 对应源码字段：
```text
slot->data.confirmed_flush
```
它和 `ctx->write_location` 的关系是：
```text
ctx->write_location:
  当前输出消息携带的 LSN。

slot->data.confirmed_flush:
  consumer 已经确认收到并能从这里继续的持久边界。
```
两者之间没有自动赋值。
`OutputPluginWrite()` 不调用 `LogicalConfirmReceivedLocation()`。
`WalSndWriteData()` 也不直接调用它。
真正推进发生在 feedback 或 SQL 成功路径。
`LogicalConfirmReceivedLocation()` 的不变量很简单：
```text
if lsn > confirmed_flush:
  confirmed_flush = lsn
```
它不会让 confirmed_flush 后退。
这样可以防止客户端重启后带着较旧的 ACK 把 slot 消费位置倒退。
但这个函数本身不验证客户端是否真的解析、落盘或 apply。
这是一条协议信任边界。
如果消费者过早 ACK，一个实现良好的 server 也会认为它确认了。
如果消费者没有 ACK，server 即使已经写出 socket，也会在重连后再次输出。
这就是本节最重要的运行模型：
```text
plugin write is not commit of consumption;
consumer flush feedback is commit of consumption.
```
中文说就是：
```text
输出不是消费确认；
ACK 才是 slot 消费确认。
```
## 6. OutputPluginPrepareWrite()/OutputPluginWrite() 的真实边界
`OutputPluginPrepareWrite()` 在 `logical.c` 中非常薄。
它检查当前 callback 是否允许写，然后调用：
```text
ctx->prepare_write(ctx, ctx->write_location, ctx->write_xid, last_write)
```
随后把 `ctx->prepared_write` 置为 true。
`OutputPluginWrite()` 也很薄。
它检查是否已经 prepare，然后调用：
```text
ctx->write(ctx, ctx->write_location, ctx->write_xid, last_write)
```
随后把 `ctx->prepared_write` 置回 false。
这里有两个关键边界。
第一，plugin 必须按顺序调用 prepare 再 write。
否则 `OutputPluginWrite()` 会 ERROR。
第二，`last_write` 影响 writer 如何暴露 LSN。
在 walsender 路径里，`WalSndPrepareWrite()` 有一条重要逻辑：
```text
if (!last_write)
  lsn = InvalidXLogRecPtr
```
原因是避免同步复制或下游 feedback 被同一个 LSN 的多个中间消息混淆。
`pgoutput` 会把一个 logical change 拆成多条 wire message。
例如第一次遇到 relation 时，可能先发送 type message，再发送 relation message，再发送 row change。
前面的 schema / type message 通常不是 `last_write`。
只有最后那条代表当前 logical event 的消息携带有效 LSN。
所以 `last_write` 不是“事务结束”。
它只是“当前 logical event 的最后一次 writer 调用”。
事务结束由 `ctx->end_xact` 表示。
`WalSndUpdateProgress()` 只在 `end_xact` 为真时才写 lag tracker。
这说明核心边界是两层：
```text
message framing:
  last_write

transaction progress:
  ctx->end_xact
```
如果 plugin 在一次 event 的中间写了 schema，然后 ERROR。
那些已经进入 socket 的 schema bytes 无法撤回。
但如果最终 LSN 没有被 consumer flush feedback 确认，slot 仍会重放这段输出。
## 7. callback wrapper：设置输出状态和错误上下文
`StartupDecodingContext()` 会把 reorder buffer 的 callbacks 指到 `logical.c` 的 wrapper。
例如：
```text
ctx->reorder->begin = begin_cb_wrapper
ctx->reorder->apply_change = change_cb_wrapper
ctx->reorder->commit = commit_cb_wrapper
ctx->reorder->message = message_cb_wrapper
ctx->reorder->stream_change = stream_change_cb_wrapper
ctx->reorder->stream_commit = stream_commit_cb_wrapper
```
这些 wrapper 做三件事。
第一，压入 `ErrorContextCallback`。
ERROR 日志里会出现类似信息：
```text
slot "...", output plugin "...", in the change callback, associated LSN ...
```
第二，设置 output state。
以 change 为例：
```text
ctx->accept_writes = true
ctx->write_xid = txn->xid
ctx->write_location = change->lsn
ctx->end_xact = false
```
以 commit 为例：
```text
ctx->accept_writes = true
ctx->write_xid = txn->xid
ctx->write_location = txn->end_lsn
ctx->end_xact = true
```
第三，调用真正的 plugin callback。
例如：
```text
ctx->callbacks.change_cb(ctx, txn, relation, change)
```
注意这里的 wrapper 本身不是 PG_TRY wrapper。
它主要负责补充状态和 error context。
真正处理 replay cleanup 的 `PG_TRY/PG_CATCH` 在 `reorderbuffer.c` 的 `ReorderBufferProcessTXN()`。
这点很重要。
如果把每个 callback wrapper 想象成一个局部 try/catch，就会误判 cleanup 责任。
callback wrapper 负责告诉你“哪个 callback、哪个 LSN 出错”。
reorder buffer 的 try/catch 负责把内部事务、snapshot、iterator、cache invalidation 收回来。
## 8. ReorderBufferProcessTXN() 的 PG_TRY/PG_CATCH
`ReorderBufferProcessTXN()` 是输出阶段的关键 runtime 边界。
它的主流程是：
```text
SetupHistoricSnapshot()
if SQL SRF already has a transaction:
  BeginInternalSubTransaction()
else:
  StartTransactionCommand()

rb->begin()
iterate changes in LSN order
  open relation
  apply toast reconstruction
  rb->apply_change()
  execute invalidations / command id changes
rb->commit()
AbortCurrentTransaction()
execute invalidations outside valid transaction
cleanup reorder buffer txn
```
它为什么要启动内部事务？
因为 logical decoding callback 期间会访问 syscache、relcache、type output function、row filter expression 等。
这些路径需要 ResourceOwner、snapshot、locks 和 transaction state。
但 output plugin 不应在这里产生持久数据库写入。
所以函数末尾还有检查：
```text
if (GetCurrentTransactionIdIfAny() != InvalidTransactionId)
  elog(ERROR, "output plugin used XID ...")
```
`PG_CATCH` 路径做几件事。
它结束 iterator。
它 teardown historic snapshot。
它 `AbortCurrentTransaction()`。
它执行或刷新 invalidation，避免 cache pollution。
如果用了内部子事务，它 `RollbackAndReleaseCurrentSubTransaction()` 并恢复 `CurrentResourceOwner`。
普通 output plugin ERROR 会走到：
```text
ReorderBufferCleanupTXN(rb, txn)
PG_RE_THROW()
```
也就是清理当前 reorder buffer transaction，然后把 ERROR 继续抛给上层。
这不会调用 `LogicalConfirmReceivedLocation()`。
这里有一个例外分支。
如果 SQLSTATE 是 `ERRCODE_TRANSACTION_ROLLBACK`，并且当前在 streaming 或 prepared 输出边界内，代码会把它解释为 concurrent abort 检测。
它会清理临时错误状态，标记 transaction aborted，并允许后续通过 stream abort 等路径收尾。
这个分支是为 in-progress streaming / 2PC 并发 abort 设计的。
不要把它理解为“普通 plugin ERROR 可以被吞掉继续输出”。
普通 ERROR 仍然 rethrow。
## 9. walsender 输出：socket 写出不等于确认
logical replication walsender 的 writer 在 `walsender.c`。
`WalSndPrepareWrite()` 会重置 `ctx->out`，写入 WALData header：
```text
PqReplMsg_WALData
dataStart = lsn
walEnd = lsn
sendtime = 0 placeholder
```
如果不是当前 event 的最后一次 write，它把 LSN 改成 invalid。
`WalSndWriteData()` 再填 send timestamp，调用：
```text
pq_putmessage_noblock(PqMsg_CopyData, ctx->out->data, ctx->out->len)
pq_flush_if_writable()
ProcessPendingWrites()
```
`ProcessPendingWrites()` 会在 socket 有 pending write 或接近 timeout 时处理 reply。
这意味着一个 output plugin callback 内部可能发生这些事：
```text
plugin calls OutputPluginWrite()
  -> WalSndWriteData() writes bytes
  -> while flushing, walsender processes downstream replies
  -> a previous or current flushPtr may advance confirmed_flush
plugin continues
plugin may still ERROR
```
所以 ERROR 和输出之间不是事务原子关系。
如果 ERROR 发生在 bytes 进入 socket 之前，consumer 看不到这条消息。
如果 ERROR 发生在 bytes 已进入 socket 之后但 consumer 没 ACK，重连后会重复输出。
如果 ERROR 发生在 bytes 已进入 socket 之后，并且 server 已经处理到 consumer 的 flush feedback，`confirmed_flush` 可能已经前进。
此时 ERROR 不会自动把 slot 位置倒回。
这不是 bug。
这是 logical replication 明确采用的外部协议边界。
只要 consumer 遵守：
```text
先持久保存 / apply 到可恢复状态
再反馈 flush LSN
```
重复输出就可以通过幂等 apply 或 durable progress 处理。
如果 consumer 先 ACK 后落盘，PostgreSQL 无法替它补救。
## 10. SQL 输出：get 和 peek 的确认差异
SQL 路径在 `logicalfuncs.c`。
`LogicalOutputPrepareWrite()` 只是 reset `ctx->out`。
`LogicalOutputWrite()` 把 `ctx->out` 放进 tuplestore，并递增 `returned_rows`。
`pg_logical_slot_get_changes_guts()` 的参数 `confirm` 决定是否消费。
`pg_logical_slot_get_changes()` 传：
```text
confirm = true
```
`pg_logical_slot_peek_changes()` 传：
```text
confirm = false
```
成功路径中，`get` 在循环结束后调用：
```text
LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
ReplicationSlotMarkDirty()
```
`peek` 不调用。
因此 `peek` 重复看到同一批输出是设计行为。
如果 plugin 在 SQL 输出过程中 ERROR，会跳到 `PG_CATCH`。
该 catch 清理 timetravel syscache：
```text
InvalidateSystemCaches()
PG_RE_THROW()
```
由于成功路径没有走完，`LogicalConfirmReceivedLocation()` 不会被调用。
SQL statement 返回 ERROR，tuplestore 结果也不会作为成功结果交给调用者。
所以 SQL 路径很适合验证本节模型：
```text
ERROR before confirm:
  confirmed_flush_lsn 不动
  下一次成功 get 会再次看到输出

peek:
  没有 ERROR 也不 confirm
  下一次 peek 或 get 仍从旧 confirmed_flush_lsn 输出
```
## 11. output plugin ERROR 的时间线
把输出过程拆成时间线更容易判断重复和丢失。
普通事务的理想路径是：
```text
reader reads commit record
  -> ReorderBufferCommit()
  -> begin callback
  -> change callback(s)
  -> commit callback
  -> messages reach consumer
  -> consumer persists/apply
  -> consumer sends flush LSN >= commit end LSN
  -> ProcessStandbyReplyMessage()
  -> LogicalConfirmReceivedLocation(flushPtr)
```
如果 ERROR 发生在 startup callback：
```text
CreateDecodingContext()
  -> startup_cb_wrapper()
  -> pgoutput_startup() or plugin startup
  -> ERROR
```
此时还没进入 CopyBoth streaming。
slot 已 acquire，但没有 logical data 输出。
确认位置不应前进。
`pgoutput` 的 protocol version 和 option 错误就应该在这里抛出。
如果 ERROR 发生在 begin callback：
```text
begin_cb_wrapper sets write_location = txn->first_lsn
plugin may or may not write BEGIN
ERROR
```
如果没有 ACK 到事务结束 LSN，重试会重新输出该事务。
如果 BEGIN bytes 已经到达 consumer，consumer 必须把它当作未完成事务的临时状态。
如果 ERROR 发生在 change callback：
```text
change_cb_wrapper sets write_location = change->lsn
plugin writes schema/type/relation/row messages
ERROR
```
schema 或 row bytes 可能已经到达 consumer。
但没有 commit end LSN 的确认，普通事务不会被视为完整消费。
下次会从 slot 的 confirmed boundary 之后重新输出。
如果 ERROR 发生在 commit callback：
```text
commit_cb_wrapper sets write_location = txn->end_lsn
plugin writes COMMIT message
ERROR before or after writer flush
```
这里最细。
如果 consumer 没有 ACK `txn->end_lsn`，下次会重复该事务。
如果 consumer 已经 ACK 到 `txn->end_lsn`，slot 可能已经前进。
ERROR 不会让 confirmed_flush_lsn 回退。
所以 plugin 不应在写出 commit 消息后再做容易失败的外部检查。
格式检查、schema 检查、option 检查要尽量前置。
## 12. confirmed_flush 不确认时为什么会重复输出
重复输出不是异常恢复的副作用。
它是 logical decoding 的基础契约。
`CreateDecodingContext()` 在没有显式 start LSN 时，从：
```text
slot->data.confirmed_flush
```
建立输出基准。
但 `XLogBeginRead()` 从：
```text
slot->data.restart_lsn
```
开始读 WAL。
这使 decoder 可以重建长事务、snapshot 和 catalog 状态。
同时只把 `confirmed_flush` 之后的事务作为可输出内容。
如果 consumer 没有确认某个事务的 commit end LSN，重连后它还会再出现。
常见重复场景包括：
```text
consumer 收到消息后崩溃，没来得及反馈 flush LSN。
walsender 写 socket 后 ERROR，反馈没被 server 处理。
网络断开，consumer 其实收到 bytes，但 server 没收到 ACK。
consumer 使用 peek 接口。
SQL get 过程中 plugin ERROR，成功 confirm 没执行。
```
这要求 consumer 做两件事。
第一，输出消息必须可幂等。
至少要能用 transaction LSN、XID、origin、message key 或业务主键识别重复。
第二，ACK 必须晚于 durable apply progress。
比如 subscriber apply worker 应该在本地事务提交或状态持久化之后，再报告 flush/apply 位置。
一个自研 consumer 如果把 ACK 当作“已经从 socket read 到内存”，它会制造丢数风险。
PostgreSQL slot 无法区分“读到内存”和“落盘可恢复”。
## 13. protocol version 与 option negotiation 边界
兼容性第一层应该在 startup callback 中处理。
`pgoutput_startup()` 是标准示例。
它在非 slot initialization 模式下调用：
```text
parse_output_parameters(ctx->output_plugin_options, data)
```
当前源码要求两个必填 option：
```text
proto_version
publication_names
```
它还解析：
```text
binary
messages
streaming
two_phase
origin
```
未知 option 会：
```text
elog(ERROR, "unrecognized pgoutput option: ...")
```
重复 option 会报 syntax error。
`proto_version` 的能力定义在 `src/include/replication/logicalproto.h`。
当前 commit 下：
```text
LOGICALREP_PROTO_MIN_VERSION_NUM = 1
LOGICALREP_PROTO_VERSION_NUM = 1
LOGICALREP_PROTO_STREAM_VERSION_NUM = 2
LOGICALREP_PROTO_TWOPHASE_VERSION_NUM = 3
LOGICALREP_PROTO_STREAM_PARALLEL_VERSION_NUM = 4
LOGICALREP_PROTO_MAX_VERSION_NUM = 4
```
这说明 native protocol version 仍是 1。
更高版本号代表额外 capability。
`pgoutput_startup()` 会显式拒绝：
```text
client requested proto_version > max
client requested proto_version < min
streaming requested with proto_version < 2
parallel streaming requested with proto_version < 4
two_phase requested with proto_version < 3
feature requested but plugin callbacks do not support it
```
这是正确的兼容性边界。
不要等到已经输出一半事务后才发现 consumer 不懂 stream message。
也不要在 row message 里临时塞一个字段，希望旧 consumer “忽略掉”。
logical replication protocol 不是 JSON 文档协议。
wire message 的一个字节类型和字段顺序就是契约。
新增 message type、tuple encoding、streaming 状态，都需要 negotiation。
## 14. output_type：文本 / 二进制消费者边界
`OutputPluginOptions` 里有：
```text
output_type
receive_rewrites
```
`pgoutput` 设置：
```text
OUTPUT_PLUGIN_BINARY_OUTPUT
```
`test_decoding` 默认设置：
```text
OUTPUT_PLUGIN_TEXTUAL_OUTPUT
```
但 `test_decoding` 的 `force-binary` option 可以切到 binary。
SQL 函数会检查 consumer 类型。
`pg_logical_slot_get_changes()` 期望 textual output。
如果 plugin 声明 binary output，它会 ERROR：
```text
logical decoding output plugin "..." produces binary output,
but function "..." expects textual data
```
`pg_logical_slot_get_binary_changes()` 则允许二进制。
这个检查发生在 decoding 正式输出前。
它是一个很小但很重要的边界。
消费者类型不匹配应早失败。
不要让文本 consumer 去猜 bytea 是不是 UTF-8。
也不要让 binary consumer 依赖 test_decoding 的调试文本格式。
output type 只能区分“文本”或“二进制”。
它不等于完整 protocol version。
自定义 plugin 如果要支持长期兼容，仍然应在自己的 option 中显式声明：
```text
format_version
feature flags
required schema metadata behavior
compression or encoding
```
然后在 startup callback 一次性校验。
## 15. schema change 的边界：pgoutput 的 relation cache
schema 兼容性第二层在 relation metadata。
`pgoutput` 不把每条 row 都写成自描述完整记录。
它维护 relation schema cache，并在需要时发送 relation message。
核心结构是 `RelationSyncEntry`。
本节关注这些字段：
| 字段 | 语义 |
| --- | --- |
| `relid` | publisher 端 relation OID，也是 protocol relation id。 |
| `replicate_valid` | 当前 entry 是否仍能用于过滤和编码。 |
| `schema_sent` | 当前 relation schema 是否已经发送给 downstream。 |
| `pubactions` | insert/update/delete/truncate 是否发布。 |
| `publish_as_relid` | partition 通过 root 发布时使用哪个 schema。 |
| `attrmap` | partition tuple 到 ancestor tuple descriptor 的映射。 |
| `columns` | publication column list。 |
| `exprstate[]` | row filter expression state。 |
| `streamed_txns` | streaming 事务里已经发送过 schema 的 top xid 列表。 |
`maybe_send_schema()` 是运行时 schema 边界。
每次准备发送 row change 前，`pgoutput_change()` 会先取得 `RelationSyncEntry`。
如果需要发送这条 change，它会调用：
```text
maybe_send_schema(ctx, change, relation, relentry)
```
如果 `schema_sent` 为 false，先发送 type / relation metadata。
然后再发送 insert/update/delete。
这保证 consumer 在看到 row message 之前已经有 relation metadata。
但这不是永久保证。
DDL、publication 变更、schema rename、column list、row filter 都可能使缓存失效。
所以 `pgoutput` 注册 invalidation callback。
## 16. relcache invalidation 为什么只标记 invalid
`pgoutput` 的 relation sync cache 初始化在：
```text
init_rel_sync_cache()
```
它注册：
```text
CacheRegisterRelcacheCallback(rel_sync_cache_relation_cb, ...)
CacheRegisterSyscacheCallback(NAMESPACEOID, rel_sync_cache_publication_cb, ...)
```
startup 还会为 publication 相关状态注册：
```text
CacheRegisterSyscacheCallback(PUBLICATIONOID, publication_invalidation_cb, ...)
CacheRegisterRelSyncCallback(rel_sync_cache_relation_cb, ...)
```
`rel_sync_cache_relation_cb()` 的设计很克制。
如果拿到具体 relid，它只找 cache entry，然后：
```text
entry->replicate_valid = false
```
如果是 whole-cache invalidation，它遍历所有 entry 标 false。
它不会在 callback 中释放 entry 的子结构。
源码注释解释了原因：
invalidation 可能发生在 logical decoding callback 中。
此时可能还有当前 callback 正在使用 entry 的指针或子结构。
如果 invalidation callback 直接破坏 entry，当前输出路径可能 use-after-free。
所以它只标记语义过期。
下一次 `get_rel_sync_entry()` 看到 `replicate_valid` false，再重建 publication、row filter、column list、tuple slots 和 attrmap。
重建时会：
```text
entry->schema_sent = false
entry->pubactions = empty
free streamed_txns / columns
rebuild publication membership
rebuild row filter and column list
```
这就是 schema 变更的正确边界。
invalidation 不阻塞正在输出的 callback。
它让下一次使用前重新解释 relation schema。
## 17. streaming schema_sent 为什么按事务跟踪
非 streaming 模式中，relation schema message 随事务输出。
consumer 在 commit 后更新自己的 relation cache。
如果事务 abort，普通事务根本不会输出这些 change。
streaming 模式不同。
publisher 可以在事务提交前发送 schema 和 row chunks。
subscriber 可能暂存这些 chunks。
如果事务最终 abort，subscriber 会丢弃这个 xid 的 stream 内容。
因此 `pgoutput` 不能简单把 `schema_sent = true` 当作全局事实。
它用 `streamed_txns` 记录：
```text
哪个 top-level xid 在 streaming 中已经发送过这个 relation schema
```
`maybe_send_schema()` 在 streaming 时调用：
```text
get_schema_sent_in_streamed_txn(relentry, topxid)
set_schema_sent_in_streamed_txn(relentry, topxid)
```
事务结束后，`cleanup_rel_sync_cache(xid, is_commit)` 处理列表。
如果 stream transaction commit，说明 downstream 会接受对应 schema。
这时可以把 `schema_sent` 置 true。
如果 stream transaction abort，downstream 会丢弃 schema records。
这时不能把 `schema_sent` 置 true。
这个细节直接服务本节主问题。
schema compatibility 不是“发送过一次 relation message”这么简单。
它要和事务最终命运绑定。
尤其在 streaming 和 2PC 中，schema metadata 本身也处在事务边界里。
## 18. pgoutput 与 test_decoding 的对照
`pgoutput` 是面向 logical replication protocol 的生产 plugin。
它的特点是：
```text
binary output
explicit proto_version negotiation
publication_names required
relation/type/tuple protocol messages
relation sync cache
schema invalidation and resend
streaming / two_phase capability checks
origin filtering
row filters and column lists
```
`test_decoding` 是面向测试和调试的 contrib plugin。
它的特点是：
```text
textual output by default
human-readable BEGIN / table / COMMIT lines
simple option parsing
include-xids / include-timestamp / skip-empty-xacts
force-binary only for output type boundary testing
stream-changes option gates streaming callbacks
```
两者都遵守同一套 output plugin callback API。
两者都调用 `OutputPluginPrepareWrite()` 和 `OutputPluginWrite()`。
但兼容性目标完全不同。
`test_decoding` 的文本格式适合课堂观察：
```text
BEGIN
table public.t: INSERT: id[integer]:1
COMMIT
```
它不适合作为长期外部协议。
一旦 consumer 依赖它的字符串拼接格式，就把调试输出当成了 ABI。
`pgoutput` 则把兼容性放在 protocol version、message type、relation metadata 和 subscriber apply 代码上。
对自定义 plugin 的启发是：
调试输出可以宽松。
生产输出必须显式版本化。
消费者不能依赖“当前字符串刚好长这样”。
## 19. 消费者兼容性策略
从本节源码可以压缩出一套 consumer 兼容性策略。
第一，连接时显式声明能力。
例如：
```text
format_version = 3
supports_streaming = true
supports_two_phase = false
supports_binary_tuple = true
```
plugin startup callback 必须拒绝不支持组合。
第二，消息中保留可恢复位置。
每个事务或 stream chunk 要能关联到 LSN、XID、origin 或自定义 sequence。
consumer 的 durable progress 要记录到足以判断重复。
第三，schema metadata 要有边界。
如果协议不是像 `pgoutput` 那样发送 relation message，就要在自己的消息里携带 schema version、column mapping 或 migration marker。
不要让 consumer 靠列顺序猜。
第四，ACK 要晚于 durable apply。
对事务 apply consumer，ACK 到 commit end LSN 应该发生在本地事务提交或幂等状态持久化后。
对 streaming consumer，收到 stream chunk 只能持久化暂存状态。
最终 ACK 策略要能处理 stream abort。
第五，升级要双写或 negotiate。
如果要改变 tuple encoding，应先让新 consumer 用新 version 连接。
旧 version 连接继续输出旧格式，或者 startup 直接拒绝。
不要在同一个 slot 上悄悄切换格式。
第六，错误要 fail closed。
如果 plugin 无法确定 consumer 是否支持某个 schema 或 option，应该在 startup 或 metadata boundary ERROR。
不要输出半个事务后才让 downstream parser 崩。
## 20. ERROR 后 slot 位置的判定表
诊断时可以按下面的状态问。
| 现象 | confirmed_flush_lsn | 下一次输出 |
| --- | --- | --- |
| startup option ERROR | 不前进 | 从旧 confirmed boundary 开始。 |
| SQL `peek` 成功 | 不前进 | 重复看到同一批输出。 |
| SQL `get` 中 plugin ERROR | 不前进 | 成功重试后再次输出。 |
| SQL `get` 成功 | 前进到 `ctx->reader->EndRecPtr` | 已确认部分通常跳过。 |
| walsender 写出后断线，未收到 flush feedback | 不前进 | 重连后可能重复。 |
| walsender 写出后收到 flush feedback | 前进到 feedback flushPtr | ERROR 不会回退。 |
| consumer 过早 ACK 后崩溃 | 已前进 | server 可能跳过，风险在 consumer。 |
| plugin commit callback 写出后 ERROR，未 ACK commit end | 未确认该事务结束 | 该事务会重复。 |
这张表的重点不是背答案。
重点是先问：
```text
有没有调用 LogicalConfirmReceivedLocation()？
传入的 LSN 是否大于旧 confirmed_flush？
这个 LSN 是否覆盖事务结束边界？
consumer 是否在 ACK 前持久化了自己的 apply state？
```
只要这四个问题回答清楚，现象通常能回到源码解释。
## 21. 异常路径一：插件调用顺序错误
plugin 如果在 callback 中直接调用 `OutputPluginWrite()`，但没有先 `OutputPluginPrepareWrite()`，会 ERROR。
错误来自：
```text
OutputPluginPrepareWrite needs to be called before OutputPluginWrite
```
这类错误不会产生合法输出。
`ReorderBufferProcessTXN()` 会清理内部事务并 rethrow。
slot 是否前进仍取决于是否已有 feedback。
如果错误发生在一次事务输出的最早阶段，一般没有新的 ACK。
重试会再次输出。
这个错误说明 output plugin API 把 framing 责任交给 plugin。
PostgreSQL 不会替 plugin 补一个 prepare。
## 22. 异常路径二：callback 中访问 catalog 失败
`pgoutput` 在 tuple 编码中会访问 type cache。
`proto.c` 的 `logicalrep_write_tuple()` 会查 `TYPEOID`，并调用 text output 或 binary send 函数。
如果历史 catalog snapshot 或 type lookup 失败，会 ERROR。
`ReorderBufferProcessTXN()` 的 catch 会清理 snapshot 和内部事务。
这种错误通常不是 consumer 兼容性错误。
它更可能是 decoding 所需 WAL / catalog horizon、插件 bug、历史 snapshot 边界或源码改动问题。
诊断要看 ERROR context 中的 callback 和 associated LSN。
然后回到对应 relation / type / schema 变更时间线。
不要直接把它解释成网络问题。
## 23. 异常路径三：consumer 不懂新协议
如果 consumer 请求：
```text
proto_version = 1
streaming = on
```
`pgoutput_startup()` 会 ERROR：
```text
requested proto_version=1 does not support streaming, need 2 or higher
```
这是好的失败。
它发生在 startup。
没有事务输出。
slot 消费位置不应因为这次失败前进。
如果一个自定义 plugin 没有 startup negotiation，而是在遇到大事务时突然输出新 message type。
旧 consumer 可能已经 ACK 到某个较早 LSN，又在新 message 处崩溃。
这会把错误恢复变复杂。
所以格式升级应在 startup option 或明确的 stream metadata 边界上处理。
不要在 row message 中静默切换。
## 24. 异常路径四：schema 变更与旧 relation metadata
schema 变更的典型风险是：
```text
publisher ALTER TABLE
plugin 仍按旧 TupleDesc 输出
consumer 按旧列映射 apply
```
`pgoutput` 的处理不是让 consumer 猜。
它在 publisher 端通过 relcache invalidation 标记 `RelationSyncEntry` 过期。
下一次 change 需要输出时，`get_rel_sync_entry()` 重建 entry。
`schema_sent` 重置为 false。
`maybe_send_schema()` 重新发送 relation metadata。
consumer 在 row message 之前拿到新的 relation definition。
如果 consumer 端无法兼容某种 schema change，例如删除列、类型改变、replica identity 改变，它应该在 apply 端报错并停止 ACK。
这样 slot 不会确认超过未成功 apply 的边界。
如果 consumer 报错但仍 ACK，就可能丢失需要重放的 change。
## 25. 成本与资源传播
本节虽然聚焦 ERROR 和兼容性，也有成本模型。
第一，重复输出会放大 decoding 成本。
consumer 不 ACK 时，publisher 可能反复从 `restart_lsn` 重读 WAL，重建 reorder buffer 和 historic snapshot。
长事务会让这个成本更明显。
第二，schema invalidation 会放大 metadata 输出。
频繁 DDL、分区层级变化、publication column list 或 row filter 变化，会让 `RelationSyncEntry` 重建并重复发送 relation/type messages。
第三，错误循环会放大 WAL retention。
如果 confirmed_flush 卡住，candidate restart 和 catalog xmin 也可能不能安全应用。
`restart_lsn` 停住会保留 WAL。
`catalog_xmin` 停住会影响 VACUUM 清理 catalog tuple。
第四，consumer 兼容性错误会传播到 apply lag。
publisher 看到的是 flush LSN 停住。
downstream 看到的是 parser / apply ERROR。
业务看到的是 replication lag 和 slot WAL 增长。
第五，streaming 降低单个大事务的峰值等待，但会引入暂存和 abort 处理成本。
consumer 必须能保存 stream chunk，并在 stream abort 时丢弃。
这些成本不是单点函数能解释的。
要沿着这条链看：
```text
format/schema error
  -> consumer stops ACK
  -> confirmed_flush_lsn stalls
  -> candidate restart/xmin cannot apply
  -> WAL and catalog retention grow
  -> reconnect repeats output
```
## 26. 观测入口：能直接看到什么
最直接的 slot 状态在：
```sql
SELECT slot_name,
       plugin,
       restart_lsn,
       confirmed_flush_lsn,
       catalog_xmin,
       wal_status,
       safe_wal_size
FROM pg_replication_slots;
```
`confirmed_flush_lsn` 不动，说明 consumer 确认边界停住。
`restart_lsn` 不动，说明重启解码所需 WAL 下界停住。
两者都不动时，要区分是 consumer 没 ACK，还是 long transaction / snapshot / candidate restart 约束。
walsender 状态在：
```sql
SELECT pid,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       write_lag,
       flush_lag,
       replay_lag
FROM pg_stat_replication;
```
这里的 `flush_lsn` 是 downstream feedback。
logical slot 的 `confirmed_flush_lsn` 会由该 flush feedback 推进。
但 `sent_lsn` 新不代表 consumer 已确认。
server log 的 ERROR context 很关键。
`logical.c` 的 `output_plugin_error_callback()` 会补充：
```text
slot name
output plugin name
callback name
associated LSN
```
如果看到：
```text
in the change callback, associated LSN X/Y
```
优先去看 relation、schema、type output、row filter 和 plugin change callback。
如果看到：
```text
in the startup callback
```
优先看 option negotiation、output_type 和 plugin 初始化。
## 27. 观测入口：只能推断什么
有些状态不能直接从一个 `pg_stat_*` 指标看出来。
例如：
```text
consumer 是否已经把消息持久化后才 ACK
plugin ERROR 前有哪些 bytes 已到达 consumer
consumer 是否把重复事务做了幂等处理
schema metadata 是否已经被 consumer 的本地 cache 接受
```
这些要从 consumer 日志、apply 状态表、replication origin progress 或抓包 / debug log 推断。
如果是 PostgreSQL 内置 subscriber，可以看 subscription worker 日志和 apply 端错误。
如果是自研 consumer，必须自己记录：
```text
last durable commit LSN
last received message LSN
last ACKed flush LSN
protocol version
schema version / relation id cache
in-flight streaming xids
```
不要把 publisher 端 `confirmed_flush_lsn` 当作完整因果。
它只证明 publisher 收到过某个 flush feedback。
它不证明 consumer 的业务语义一定正确。
## 28. 常见误区
误区一：`OutputPluginWrite()` 成功返回，所以 slot 已经消费。
错；它只说明 writer callback 本次没报错，slot 位置由 `LogicalConfirmReceivedLocation()` 推进。
误区二：`ctx->write_location` 等于 `confirmed_flush_lsn`。
错；前者是本次输出消息关联的 LSN，后者是 consumer 确认边界。
误区三：plugin ERROR 会把已经发给 consumer 的消息回滚。
错；网络输出不是事务资源，ERROR 只能清理 backend 内部状态。
误区四：重复输出说明 PostgreSQL 发送了脏数据。
错；未确认输出重复是 at-least-once 恢复策略。
误区五：schema change 可以靠 consumer 自动发现。
错；`pgoutput` 显式发送 relation metadata，并用 invalidation 决定何时重发。
误区六：`proto_version` 只是文档字段。
错；它决定 streaming、two_phase、parallel streaming 等 message type 是否允许出现。
## 29. 课堂实验一：SQL get / peek 与 ERROR 不确认
目标：
验证 SQL 路径中只有成功 `get` 会推进 `confirmed_flush_lsn`。
准备：
```sql
CREATE TABLE ld_err_demo(id int primary key, v text);
SELECT pg_create_logical_replication_slot('ld_err_slot', 'test_decoding');
INSERT INTO ld_err_demo VALUES (1, 'a');
SELECT slot_name, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'ld_err_slot';
```
先执行一个会在 startup option parse 中失败的调用：
```sql
SELECT *
FROM pg_logical_slot_get_changes(
  'ld_err_slot',
  NULL,
  NULL,
  'nonexistent-option',
  '1'
);
```
预期是 `test_decoding` 报 unknown option。
再查：
```sql
SELECT slot_name, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'ld_err_slot';
```
它不应因为这次 ERROR 前进。
然后执行：
```sql
SELECT *
FROM pg_logical_slot_peek_changes(
  'ld_err_slot',
  NULL,
  NULL,
  'include-xids',
  '0',
  'skip-empty-xacts',
  '1'
);
```
能看到变化，但 confirmed 仍不前进。
最后执行：
```sql
SELECT *
FROM pg_logical_slot_get_changes(
  'ld_err_slot',
  NULL,
  NULL,
  'include-xids',
  '0',
  'skip-empty-xacts',
  '1'
);
```
成功后再查 slot。
这次 `confirmed_flush_lsn` 应前进。
源码回扣：
```text
logicalfuncs.c
  pg_logical_slot_get_changes_guts(confirm)
    -> confirm true 时才 LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
```
## 30. 课堂实验二：在 plugin callback 后半段注入 ERROR
目标：
观察“已经 OutputPluginWrite，但未确认”时的重复输出。
这是源码实验，不要求把 patch 留在产品代码。
在实验分支中临时修改 `contrib/test_decoding/test_decoding.c`。
可以在 `pg_decode_change()` 中某次 `OutputPluginWrite(ctx, true)` 之后加：
```c
ereport(ERROR,
        (errmsg("injected error after test_decoding write")));
```
重新编译安装 `test_decoding`。
运行：
```sql
CREATE TABLE ld_inject_demo(id int primary key);
SELECT pg_create_logical_replication_slot('ld_inject_slot', 'test_decoding');
INSERT INTO ld_inject_demo VALUES (1);
SELECT *
FROM pg_logical_slot_get_changes(
  'ld_inject_slot',
  NULL,
  NULL,
  'include-xids',
  '0',
  'skip-empty-xacts',
  '1'
);
```
预期 SQL statement ERROR。
再查 `pg_replication_slots.confirmed_flush_lsn`。
它不应因失败的 SQL get 成功确认。
去掉注入 ERROR 或改成只在第一次触发。
再次执行 `get_changes`。
应能看到同一事务输出。
源码回扣：
```text
ReorderBufferProcessTXN()
  -> PG_CATCH cleanup
  -> PG_RE_THROW

pg_logical_slot_get_changes_guts()
  -> 成功路径之后才 confirm
```
如果用 walsender 和自研 consumer 做同类实验，结论要多一个条件。
server 是否已经处理 consumer feedback，会影响 confirmed_flush 是否前进。
## 31. 课堂实验三：pgoutput protocol negotiation
目标：
验证格式升级必须在 startup 边界拒绝。
创建 publication：
```sql
CREATE TABLE ld_pub_demo(id int primary key, v text);
CREATE PUBLICATION ld_pub FOR TABLE ld_pub_demo;
SELECT pg_create_logical_replication_slot('ld_pgoutput_slot', 'pgoutput');
```
用 logical replication 客户端请求不兼容组合。
例如请求 `proto_version = 1` 且 `streaming = on`。
预期 `pgoutput_startup()` ERROR，提示需要更高 protocol version。
然后请求：
```text
proto_version = 4
publication_names = ld_pub
streaming = parallel
```
如果服务端和 plugin 支持，startup 可通过。
源码回扣：
```text
pgoutput.c
  parse_output_parameters()
  pgoutput_startup()

logicalproto.h
  LOGICALREP_PROTO_STREAM_VERSION_NUM
  LOGICALREP_PROTO_TWOPHASE_VERSION_NUM
  LOGICALREP_PROTO_STREAM_PARALLEL_VERSION_NUM
```
这个实验的重点不是客户端命令本身。
重点是错误发生在输出事务之前。
slot 位置不会因为不兼容 startup 前进。
## 32. 诊断流程：从现象回到源码
遇到“logical replication 卡住或重复输出”时，按这个顺序查。
第一步，看 publisher slot。
```sql
SELECT slot_name,
       plugin,
       restart_lsn,
       confirmed_flush_lsn,
       catalog_xmin,
       wal_status,
       safe_wal_size
FROM pg_replication_slots;
```
如果 `confirmed_flush_lsn` 不动，说明 ACK 边界没有推进。
第二步，看 walsender feedback。
```sql
SELECT pid,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn
FROM pg_stat_replication;
```
如果 sent 新、flush 旧，优先查 consumer。
第三步，看 server log 的 output plugin error context，记录 callback 名和 associated LSN。
第四步，看 startup options：`proto_version`、`publication_names`、`streaming`、`two_phase` 或自定义 `format_version`。
第五步，看 schema invalidation：DDL 后 relation metadata 是否重发，consumer 是否更新 relation cache。
第六步，看 consumer ACK 策略：是否在 durable apply 后才反馈 flush LSN。
如果 consumer 日志只有 received LSN，没有 durable LSN，就无法可靠诊断重复和丢失。
## 33. 讨论题
1. 为什么 `OutputPluginWrite()` 不能直接推进 `confirmed_flush_lsn`？
2. plugin 在 commit callback 写出 COMMIT 后又 ERROR，哪些情况下重连会重复该事务？
3. 为什么 `pgoutput` 要在 startup 阶段拒绝不支持的 `proto_version` 和 streaming 组合？
4. relcache invalidation callback 为什么只标记 `replicate_valid = false`，而不是立即释放 cache entry？
5. streaming transaction abort 后，为什么 `schema_sent` 不能简单按“发送过”处理？
6. 自研 consumer 应该把 last received LSN、durable apply LSN、ACKed flush LSN 分别记录在哪里？
## 34. 本节小结
本节把 output plugin ERROR 压缩成一条状态链。
`logical.c` 的 callback wrapper 设置 `ctx->write_location`、`write_xid` 和 `end_xact`，并提供错误上下文。
`OutputPluginPrepareWrite()` / `OutputPluginWrite()` 只调用 writer。
它们不推进 slot。
`ReorderBufferProcessTXN()` 的 `PG_TRY/PG_CATCH` 清理 historic snapshot、内部事务、ResourceOwner 和 reorder buffer 状态。
普通 plugin ERROR 会 rethrow。
slot 的 `confirmed_flush_lsn` 只通过 `LogicalConfirmReceivedLocation()` 前进。
SQL `get` 成功会确认。
SQL `peek` 不确认。
walsender 只有收到 downstream flush feedback 才确认。
因此未确认输出重复是正常恢复语义。
已确认位置不会因为后续 ERROR 自动回退。
格式升级应该在 startup option / protocol version negotiation 边界处理。
`pgoutput` 用 `proto_version` 明确 gating streaming、two_phase 和 parallel streaming。
schema 变更应该在 relation metadata 和 invalidation 边界处理。
`pgoutput` 用 `RelationSyncEntry`、`schema_sent`、`replicate_valid` 和 relcache/syscache callback 决定何时重发 relation schema。
消费者兼容性最终落在 durable apply 与 ACK 顺序上。
PostgreSQL 可以提供可重放输出流和 monotonic slot confirmation。
它不能替 consumer 证明消息已经被业务语义安全接受。
可迁移规律是：
```text
任何把内部事件翻译成外部协议的系统，
都必须把“已生成”、“已发送”、“已持久接收”、“已应用”和“已确认”分成不同状态；
ERROR cleanup 只能处理本地状态，
跨进程兼容性必须靠显式 protocol / schema / ACK 边界维护。
```
