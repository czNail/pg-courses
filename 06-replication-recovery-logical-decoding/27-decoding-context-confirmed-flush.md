# PostgreSQL Decoding Context confirmed_flush_lsn 与消费者确认边界
## 课程定位
前置知识：
已经理解 replication slot 的类型、生命周期、`restart_lsn`、`catalog_xmin`。
已经理解 reorder buffer 如何把 WAL record 重组成事务级 logical stream。
已经理解 logical decoding 会从历史 WAL 和历史 catalog snapshot 中还原事务语义。
本节唯一主问题：
```text
logical decoding 如何用 confirmed_flush_lsn 表示消费者已经安全接收的位置，
为什么读取位置、输出位置和确认位置不能混为一谈？
```
核心矛盾：
logical decoder 必须从足够早的 WAL 位置重新读取，才能重建未完成事务、snapshot 和 catalog 语义。
output plugin 又只能把 `confirmed_flush_lsn` 之后的事务作为新内容交给消费者。
slot 只有在消费者确认自己已经安全接收之后，才能把跨会话进度向前推进。
如果把这三个位置混成一个 LSN，断线重连、长事务、skip transaction、streaming feedback 和 slot WAL 保留都会变得不可靠。
学完后应能判断：
`ctx->reader->ReadRecPtr` / `ctx->reader->EndRecPtr` 表示 decoder 正在读到哪里。
`ctx->write_location` 和 `ctx->end_xact` 表示 output plugin 正在写哪类 logical 输出。
`slot->data.confirmed_flush` 表示消费者确认过的 logical 输出边界。
`slot->data.restart_lsn` 和 `candidate_restart_lsn` 表示下次重建解码状态所需的 WAL 起点。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面 slot 课程回答过：
```text
restart_lsn 为什么保留 WAL？
catalog_xmin 为什么阻止 VACUUM 清走 catalog tuple？
slot release 为什么不能清掉跨会话进度？
```
前面 reorder buffer 课程回答过：
```text
WAL record 如何按 XID 重组成 ReorderBufferTXN？
大事务如何 spill 或 streaming？
abort / ERROR 时哪些本地状态可以丢？
```
本节接在这两组状态之后。
它只追踪一条边界：
```text
消费者已经确认安全接收到哪里？
```
这个边界在 SQL 视图里叫 `confirmed_flush_lsn`。
在源码结构里叫：
```text
ReplicationSlotPersistentData.confirmed_flush
```
这个字段的名字容易诱导两个误解。
第一，它不是 decoder 当前读 WAL 的位置。
decoder 读 WAL 常常从更早的 `restart_lsn` 开始。
第二，它不是 sender 已经写入 socket 的位置。
streaming walsender 只有收到下游反馈的 flush LSN，才调用 `LogicalConfirmReceivedLocation()`。
因此本节主线是：
```text
CreateDecodingContext()
  -> 以 confirmed_flush 作为输出基准
  -> 以 restart_lsn 作为 WAL 读取起点
  -> ctx->reader 推进读取位置
  -> reorder buffer 在 commit 边界输出
  -> output writer 暴露 write_location
  -> SQL get 或 walsender feedback 调用 LogicalConfirmReceivedLocation()
  -> confirmed_flush 前进
  -> 候选 restart_lsn / catalog_xmin 可能被应用并保存 slot
```
只要记住这条链，很多现象就不再矛盾。
消费者断开后重新连接，可能从旧 WAL 重读。
但它不会重新输出已经确认的事务。
`restart_lsn` 很旧，可能只是 decoder 仍需要重建某个未完成事务。
`confirmed_flush_lsn` 很旧，才说明消费者确认边界停住了。
`sent_lsn` 很新，也不代表 slot 可以推进确认边界。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
logical decoding session 创建 LogicalDecodingContext，
用 slot->data.restart_lsn 初始化 XLogReader 的读取起点，
用 slot->data.confirmed_flush 初始化输出起点，
ctx->reader->ReadRecPtr / EndRecPtr 随 WAL record 前进，
output plugin 在 change / commit callback 中使用 ctx->write_location 写 logical data，
消费者通过 SQL get 或 streaming feedback 确认 flush LSN，
LogicalConfirmReceivedLocation() 只向前推进 slot->data.confirmed_flush，
并且只有确认覆盖 candidate_restart_valid 时才应用 candidate_restart_lsn。
```
这里至少有四个 LSN 视角。
```text
读取位置:
  ctx->reader->ReadRecPtr
  ctx->reader->EndRecPtr
输出位置:
  ctx->write_location
  ReorderBufferTXN.first_lsn / final_lsn / end_lsn
确认位置:
  slot->data.confirmed_flush
  pg_replication_slots.confirmed_flush_lsn
重启读取下界:
  slot->data.restart_lsn
  slot->candidate_restart_lsn
  slot->candidate_restart_valid
```
这些位置之间不是简单大小关系。
正常 streaming 时，`ctx->reader->EndRecPtr` 可以远超 `confirmed_flush_lsn`。
长事务时，decoder 已经读到很多 change，但还没有 commit callback，消费者还不能确认事务最终输出。
大事务 streaming 时，消费者可能收到 in-progress change，但事务最终 abort 后还需要 stream abort。
slot 刚重连时，decoder 从 `restart_lsn` 重读旧 WAL，但 `CreateDecodingContext()` 会把输出起点约束在 `confirmed_flush` 之后。
本节要建立的不变量是：
```text
读得远，不等于已经输出。
输出了，不等于消费者安全接收。
消费者确认了，也不等于 decoder 下次不需要更早 WAL。
```
## 3. 核心文件分工与阅读顺序
阅读顺序不要从 `logical.c` 第一行开始。
先看状态，再看谁推进状态。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/logical.h` | `LogicalDecodingContext` 字段：`reader`、`write_location`、`write_xid`、`end_xact`、writer callbacks。 |
| 2 | `src/include/replication/slot.h` | `ReplicationSlotPersistentData.confirmed_flush`、`restart_lsn`、candidate restart / xmin 字段、`last_saved_confirmed_flush`。 |
| 3 | `src/backend/replication/logical/logical.c` | `StartupDecodingContext()`、`CreateDecodingContext()`、`DecodingContextFindStartpoint()`、`LogicalConfirmReceivedLocation()`。 |
| 4 | `src/backend/replication/logical/decode.c` | `LogicalDecodingProcessRecord()` 如何用 `ReadRecPtr` / `EndRecPtr` 填 record buffer，commit / abort 如何进入 reorder buffer。 |
| 5 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferCommit()`、`ReorderBufferReplay()`、`update_progress_txn` 如何把事务进度交给 logical layer。 |
| 6 | `src/backend/replication/walsender.c` | `StartLogicalReplication()`、`XLogSendLogical()`、`WalSndPrepareWrite()`、`ProcessStandbyReplyMessage()`。 |
| 7 | `src/backend/replication/logical/logicalfuncs.c` | SQL `pg_logical_slot_get_changes()` 与 `pg_logical_slot_peek_changes()` 的确认差异。 |
| 8 | `src/backend/replication/logical/worker.c` | subscriber apply worker 如何计算 write / flush feedback。 |
| 9 | `src/backend/replication/slot.c` | `ReplicationSlotSave()`、`CheckPointReplicationSlots()`、`last_saved_restart_lsn` 与 slot 持久化。 |
推荐跟读路线：
```text
StartLogicalReplication()
  -> CreateDecodingContext()
     -> StartupDecodingContext()
  -> XLogBeginRead(ctx->reader, slot->data.restart_lsn)
  -> WalSndLoop(XLogSendLogical)
     -> XLogReadRecord(ctx->reader)
     -> LogicalDecodingProcessRecord(ctx, ctx->reader)
     -> output plugin callbacks
     -> WalSndPrepareWrite() / WalSndWriteData()
  -> ProcessStandbyReplyMessage()
     -> LogicalConfirmReceivedLocation(flushPtr)
```
SQL 路径用另一条入口验证同一个模型：
```text
pg_logical_slot_get_changes_guts(confirm = true)
  -> CreateDecodingContext(InvalidXLogRecPtr, ...)
  -> XLogBeginRead(ctx->reader, MyReplicationSlot->data.restart_lsn)
  -> LogicalDecodingProcessRecord()
  -> tuplestore_putvalues()
  -> LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
```
`peek` 路径同样读 WAL、同样执行 decoding，但不确认。
这正是它能重复看见同一批输出的原因。
## 4. 核心状态一：LogicalDecodingContext
`LogicalDecodingContext` 是本次 decoding session 的 backend-local 对象。
它定义在 `src/include/replication/logical.h`。
它不在 shared memory。
断线、ERROR 或 session 结束后，下一次会重新创建。
本节最重要的字段不是很多：
| 字段 | 本节语义 |
| --- | --- |
| `slot` | 当前 acquire 的 replication slot。跨会话进度在 slot 中。 |
| `reader` | `XLogReaderState`，保存当前读 WAL 的 record 指针。 |
| `reorder` | 本次会话的 reorder buffer，重组事务。 |
| `snapshot_builder` | 决定从哪里开始有 consistent snapshot，何时推进 restart candidate。 |
| `prepare_write` / `write` / `update_progress` | consumer-specific writer 回调。SQL 和 walsender 的实现不同。 |
| `out` | output plugin 写入的临时 `StringInfo`。 |
| `write_location` | 当前 output callback 关联的 LSN。 |
| `write_xid` | 当前 output callback 关联的事务 id。 |
| `end_xact` | 当前 callback 是否处在事务结束输出边界。 |
注意这里没有 `ctx->startptr`。
源码中有局部变量 `startptr`，例如 walsender 发送物理 WAL 或计算消息范围时使用。
但 logical decoding 的 context 本身不保存一个叫 `startptr` 的字段。
本节如果说“读取起点”，对应的是：
```text
XLogBeginRead(ctx->reader, slot->data.restart_lsn)
```
如果说“当前读到的 record”，对应的是：
```text
ctx->reader->ReadRecPtr
ctx->reader->EndRecPtr
```
如果说“当前 output plugin 正在写的 logical LSN”，对应的是：
```text
ctx->write_location
ctx->end_xact
```
这三个名字的区分非常重要。
`ReadRecPtr` 是当前 WAL record 的起始位置。
`EndRecPtr` 是当前 WAL record 的结束位置。
`write_location` 是 wrapper 为 output plugin 设置的语义位置。
它可能来自 change LSN，也可能来自 transaction end LSN。
`end_xact` 则告诉 walsender 的 progress callback：
```text
这次 progress 是否代表事务结束位置。
```
`WalSndUpdateProgress()` 明确只把 end-of-transaction LSN 写入 lag tracker。
原因也在注释里：
downstream 不会给任意中间 LSN 提供完整 ack 语义。
logical lag 跟踪只能可靠地围绕事务结束位置。
## 5. 核心状态二：ReplicationSlot 中的确认边界
`confirmed_flush_lsn` 在 SQL 视图里出现。
源码字段是：
```text
ReplicationSlotPersistentData.confirmed_flush
```
结构体注释给出的语义是：
```text
Oldest LSN that the client has acked receipt for.
```
它有两个直接用途。
第一，如果 client 没有指定 start LSN，`CreateDecodingContext()` 从它继续：
```text
start_lsn = slot->data.confirmed_flush
```
第二，如果 client 指定的 start LSN 比它还旧，源码会把 start LSN 向前修正：
```text
if (start_lsn < slot->data.confirmed_flush)
  start_lsn = slot->data.confirmed_flush
```
这里没有报错。
源码注释解释了原因：
客户端可能确认了某个 LSN，但这段 WAL 没有产生需要持久记录的 logical change。
重连时客户端只记得更旧的业务位置。
为了支持这种情况，server 允许向前修正。
但这也暴露了客户端责任：
如果客户端需要严格断点续传语义，它应检查 `confirmed_flush_lsn` 是否符合自己的预期。
slot 里还有一组 candidate 字段：
```text
candidate_catalog_xmin
candidate_xmin_lsn
candidate_restart_valid
candidate_restart_lsn
```
它们不是消费者进度。
它们是 decoder 在处理 WAL 时发现的“将来可以放宽的保护边界”。
例如 `SnapBuildProcessRunningXacts()` 看到 running-xacts record 后，会判断：
```text
如果还有 in-progress transaction:
  restart candidate = 该事务开始前最后可重建 snapshot 的位置
如果没有 in-progress transaction:
  restart candidate = last_serialized_snapshot
```
然后调用：
```text
LogicalIncreaseRestartDecodingForSlot(current_lsn, restart_lsn)
```
`current_lsn` 表示这个候选值从哪个 WAL 位置之后才安全。
消费者只有确认 flush 覆盖了 `candidate_restart_valid`，slot 才能真正采用 `candidate_restart_lsn`。
因此：
```text
candidate_restart_lsn 是 decoder 的建议。
confirmed_flush_lsn 是消费者的确认。
restart_lsn 是二者交汇后才能发布的读取下界。
```
## 6. CreateDecodingContext：输出基准与读取起点分离
`CreateDecodingContext()` 的参数名是 `start_lsn`。
这个名字也容易误导。
它不是 `XLogReader` 马上开始读 WAL 的位置。
它是 decoding context 的输出基准。
源码路径是：
```text
CreateDecodingContext(start_lsn, ...)
  -> 如果 start_lsn 无效，使用 slot->data.confirmed_flush
  -> 如果 start_lsn 早于 confirmed_flush，向前修正
  -> StartupDecodingContext(... start_lsn ...)
     -> AllocateSnapshotBuilder(... start_lsn ...)
```
真正开始读 WAL 的地方在调用者。
walsender 路径：
```text
StartLogicalReplication()
  -> CreateDecodingContext(cmd->startpoint, ...)
  -> XLogBeginRead(logical_decoding_ctx->reader,
                   MyReplicationSlot->data.restart_lsn)
```
SQL 路径：
```text
pg_logical_slot_get_changes_guts()
  -> CreateDecodingContext(InvalidXLogRecPtr, ...)
  -> XLogBeginRead(ctx->reader,
                   MyReplicationSlot->data.restart_lsn)
```
这解释了本节最关键的分离：
```text
confirmed_flush:
  我已经交付到哪里，不要再把更早提交事务当作新输出。
restart_lsn:
  我为了正确重建后续事务，必须从哪里开始重读 WAL。
```
如果某个长事务在 `confirmed_flush` 之前就开始，但在 `confirmed_flush` 之后才提交，decoder 下次仍必须重读它早期的 WAL record。
否则 commit 到来时，reorder buffer 缺少前半段 change。
这就是为什么 SQL 函数注释写得很直白：
```text
Decoding of WAL must start at restart_lsn so that the entirety of
xacts that committed after the slot's confirmed_flush can be accumulated.
```
这句话是本节的核心源码证据。
它把读取位置和输出位置永久分开了。
## 7. StartupDecodingContext：本地运行态的 owner
`StartupDecodingContext()` 创建 `"Logical decoding context"` memory context。
然后创建：
```text
ctx = palloc0_object(LogicalDecodingContext)
ctx->reader = XLogReaderAllocate(...)
ctx->reorder = ReorderBufferAllocate()
ctx->snapshot_builder = AllocateSnapshotBuilder(...)
ctx->reorder->private_data = ctx
```
这些状态服务当前 backend。
`FreeDecodingContext()` 的释放顺序是：
```text
shutdown_cb_wrapper(ctx)
ReorderBufferFree(ctx->reorder)
FreeSnapshotBuilder(ctx->snapshot_builder)
XLogReaderFree(ctx->reader)
MemoryContextDelete(ctx->context)
```
这说明 `confirmed_flush` 不在 context 里。
context 可以被释放。
slot 的确认边界继续存在。
断线重连后重新创建 context。
新 context 会再次读取：
```text
slot->data.restart_lsn
slot->data.confirmed_flush
```
前者决定从哪里读。
后者决定哪些提交事务已经不应作为新输出。
这也解释了 ERROR 后的恢复模型：
output plugin callback ERROR 会丢弃当前 backend 的 context、reader、snapshot builder、reorder buffer 和 spill 文件状态。
但只要 `LogicalConfirmReceivedLocation()` 没有成功推进并保存相关 slot 状态，跨会话边界不会被错误放宽。
PostgreSQL 宁愿重复读取和重组 WAL，也不会把未确认输出当成已确认。
## 8. WAL 读取位置：ReadRecPtr 与 EndRecPtr
`LogicalDecodingProcessRecord()` 是 WAL record 进入 logical decode 的统一入口。
它一开始就把 reader 的位置复制到 record buffer：
```text
buf.origptr = ctx->reader->ReadRecPtr
buf.endptr = ctx->reader->EndRecPtr
buf.record = record
```
这两个位置服务 decode 和 reorder。
heap insert/update/delete 的 change 会用 `buf->origptr` 或 `buf->endptr` 作为 change LSN。
xact commit/abort 会把 commit record 的 start/end LSN 传给 reorder buffer。
`DecodeCommit()` 中的关键调用是：
```text
SnapBuildCommitTxn(... buf->origptr ...)
ReorderBufferCommit(... buf->origptr, buf->endptr, ...)
```
因此 `ctx->reader->EndRecPtr` 表示：
```text
decoder 已经读过这个 WAL record。
```
它不表示：
```text
消费者已经收到该 record 对应的 logical output。
```
也不表示：
```text
slot 下次不需要更早 WAL。
```
SQL `get` 路径在确认时使用：
```text
LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
```
这成立的前提是 SQL 函数已经把结果放入 tuplestore，并且 `confirm = true`。
walsender 路径不会因为 `ctx->reader->EndRecPtr` 前进就确认。
它只会更新本地 `sentPtr`。
确认必须等下游 feedback。
这就是 SQL 函数和 streaming walsender 的重要分歧。
SQL `get` 是一次函数调用内的 pull API。
walsender 是持续 streaming API，网络、下游 apply、下游 flush 和 feedback 都在函数返回之外发生。
## 9. 输出位置：write_location 与 end_xact
logical decoding 不把 WAL record 原样发送给 consumer。
它先经过 reorder buffer。
真正调用 output plugin 时，`logical.c` 的 wrapper 会设置 `ctx->write_location`。
普通 begin callback：
```text
ctx->write_xid = txn->xid
ctx->write_location = txn->first_lsn
ctx->end_xact = false
```
change callback：
```text
ctx->write_xid = txn->xid
ctx->write_location = change->lsn
ctx->end_xact = false
```
commit callback：
```text
ctx->write_xid = txn->xid
ctx->write_location = txn->end_lsn
ctx->end_xact = true
```
`txn->end_lsn` 是 commit record 结束位置。
`txn->final_lsn` 通常是导致事务 commit / abort / prepare 的 record 起始位置。
`txn->first_lsn` 是事务第一条 relevant WAL record 的位置。
这三个字段都在 `ReorderBufferTXN`。
它们分别回答不同问题：
```text
first_lsn:
  这个事务的逻辑相关内容最早从哪里开始？
final_lsn:
  哪条 record 让这个事务进入最终状态？
end_lsn:
  commit / prepare record 的结束位置，常用作事务完成边界。
```
`ctx->end_xact` 不是多余字段。
`WalSndUpdateProgress()` 用它决定是否把这个 LSN 写入 lag tracker。
源码注释说明：
```text
We don't have a mechanism to get the ack for any LSN other than end xact LSN.
```
因此 change callback 的 `write_location` 可以帮助 output plugin 生成 per-change LSN。
但它不能被解释为消费者已经可以确认整个事务。
尤其是 streaming in-progress transaction 时，中间 change 被发送，不等于事务最终 committed。
## 10. 主流程源码 walkthrough：walsender 发送位置不是确认位置
logical walsender 的启动路径在 `StartLogicalReplication()`。
它做三件关键事：
```text
ReplicationSlotAcquire(cmd->slotname, ...)
CreateDecodingContext(cmd->startpoint, ...)
XLogBeginRead(ctx->reader, MyReplicationSlot->data.restart_lsn)
```
然后设置：
```text
sentPtr = MyReplicationSlot->data.confirmed_flush
MyWalSnd->sentPtr = MyReplicationSlot->data.restart_lsn
```
这里已经出现两个 sender 视角。
`sentPtr` 用于逻辑发送进度。
`MyWalSnd->sentPtr` 是共享内存中给监控和 walsender 状态看的位置。
它们也不等于 `confirmed_flush_lsn` 的最终推进。
主循环在 `XLogSendLogical()`：
```text
record = XLogReadRecord(logical_decoding_ctx->reader, &errm)
LogicalDecodingProcessRecord(logical_decoding_ctx, logical_decoding_ctx->reader)
sentPtr = logical_decoding_ctx->reader->EndRecPtr
```
这只是 sender 读过并尝试发送 logical data。
真正的确认入口在 `ProcessStandbyReplyMessage()`。
它从 standby status update 里解析：
```text
writePtr
flushPtr
applyPtr
replyTime
replyRequested
```
然后：
```text
if (MyReplicationSlot && XLogRecPtrIsValid(flushPtr))
{
  if (SlotIsLogical(MyReplicationSlot))
    LogicalConfirmReceivedLocation(flushPtr);
  else
    PhysicalConfirmReceivedLocation(flushPtr);
}
```
对 logical slot 来说，推进 `confirmed_flush` 的是 feedback 里的 `flushPtr`。
不是 `sentPtr`。
不是 socket write 成功。
不是 output plugin 已经调用 `OutputPluginWrite()`。
server 信任 downstream 发送的 flush LSN。
如果 downstream 是 PostgreSQL logical apply worker，flush LSN 的计算有自己的本地 WAL durable 逻辑。
如果 downstream 是第三方客户端，客户端必须保证自己只在安全持久化之后 ack。
## 11. subscriber apply worker：flush feedback 从哪里来
`src/backend/replication/logical/worker.c` 展示了 PostgreSQL subscriber 如何构造 feedback。
apply worker 不能简单把“最后收到的 remote LSN”报成 flush。
原因很直接：
```text
remote transaction 已经收到
  -> 本地 apply 可能已经执行
  -> 本地事务 WAL 可能还没有 flush
  -> subscriber crash 后可能丢掉这次 apply
```
所以 worker 维护 remote/local LSN 映射。
提交时会调用类似：
```text
store_flush_position(remote_commit_end_lsn, local_commit_end_lsn)
```
`get_flush_position()` 读取本地：
```text
local_flush = GetFlushRecPtr(NULL)
```
然后遍历映射表：
```text
如果 pos->local_end <= local_flush:
  flush = pos->remote_end
  从 mapping 中删除
否则:
  还有 pending transaction
  不能把更晚 remote LSN 报成 flush
```
`send_feedback()` 最后发送：
```text
write = recvpos 或最新 remote_end
flush = 已经本地 flush 的 remote_end
apply = writepos
```
当前源码里 logical worker 的 status update 字段写入顺序是：
```text
pq_sendint64(reply_message, recvpos)    /* write */
pq_sendint64(reply_message, flushpos)   /* flush */
pq_sendint64(reply_message, writepos)   /* apply */
```
这和 physical standby 的命名习惯不完全相同。
但 walsender 端只关心第二个字段 `flushPtr` 来调用 `LogicalConfirmReceivedLocation()`。
因此 `confirmed_flush_lsn` 的含义应该读成：
```text
publisher 收到了 consumer 声称已安全 flush 的 logical output LSN。
```
它不是 publisher 自己验证出来的。
它也不是 subscriber 已经对外可见到业务层的完整证明。
PostgreSQL 内置 subscriber 会尽量把这个确认绑定到本地 WAL flush。
第三方 consumer 的安全性取决于它自己的 ack discipline。
## 12. SQL get 与 peek：同一条 decoder，不同确认语义
SQL logical decoding 入口在 `logicalfuncs.c`。
四个函数共用：
```text
pg_logical_slot_get_changes_guts(fcinfo, confirm, binary)
```
`get` 系列传：
```text
confirm = true
```
`peek` 系列传：
```text
confirm = false
```
两者前半段基本一样：
```text
ReplicationSlotAcquire(...)
CreateDecodingContext(InvalidXLogRecPtr, ...)
XLogBeginRead(ctx->reader, MyReplicationSlot->data.restart_lsn)
while (ctx->reader->EndRecPtr < end_of_wal)
  XLogReadRecord()
  LogicalDecodingProcessRecord()
  output rows into tuplestore
```
差异只在最后：
```text
if (XLogRecPtrIsValid(ctx->reader->EndRecPtr) && confirm)
{
  LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr);
  ReplicationSlotMarkDirty();
}
```
所以 `peek` 不是“不推进 reader”。
它会读 WAL。
它会构造 reorder buffer。
它会调用 output plugin。
它只是不调用 `LogicalConfirmReceivedLocation()`。
因此下一次 peek 仍以旧的 `confirmed_flush` 为输出基准。
这解释了一个常见现象：
```text
连续执行 pg_logical_slot_peek_changes()
  -> 看到相同 change
执行 pg_logical_slot_get_changes()
  -> 返回 change 并推进 confirmed_flush
再次 peek 或 get
  -> 不再返回已经确认的 change
```
SQL `get` 还会 `ReplicationSlotMarkDirty()`。
源码注释说得很谨慎：
如果只有 `confirmed_flush_lsn` 改了，`LogicalConfirmReceivedLocation()` 不一定把 slot 标 dirty。
SQL 用户没有 streaming client 那样的自有进度管理。
所以 SQL 路径更努力让 clean shutdown 后保留进度。
但这仍不等于每次 SQL get 后都立即 crash-safe。
注释明确说：
```text
We'll still lose its position on crash, as documented.
```
持久化是 slot save / checkpoint 维度的问题。
它不是 `confirmed_flush` 内存字段本身能保证的。
## 13. LogicalConfirmReceivedLocation：只向前推进
`LogicalConfirmReceivedLocation(XLogRecPtr lsn)` 是确认边界的核心函数。
它的第一条规则：
```text
if (lsn > MyReplicationSlot->data.confirmed_flush)
  MyReplicationSlot->data.confirmed_flush = lsn
```
如果收到旧 ack，不会把 `confirmed_flush` 往回退。
源码注释解释了这样做的原因：
客户端可能 ack 了一个不需要持久记录的 LSN。
重启后客户端又发送自己持久保存的更旧 LSN。
server 如果接受回退，就会把已经复制过的 change 再当成未确认。
所以确认边界是单向的。
这条单向性不是优化。
它是避免重复输出的正确性边界。
第二条规则：
```text
如果 candidate_xmin_lsn <= lsn:
  应用 candidate_catalog_xmin
如果 candidate_restart_valid <= lsn:
  data.restart_lsn = candidate_restart_lsn
```
也就是说，`confirmed_flush` 前进时可能顺带推进两个保护边界：
```text
catalog_xmin:
  VACUUM / catalog tuple 保留边界可以放宽。
restart_lsn:
  WAL 保留边界可以放宽。
```
但这两个推进都要求 candidate 的 valid LSN 已经被消费者确认覆盖。
否则 crash 后可能出现：
```text
server 删除了更早 WAL 或 catalog tuple
consumer 实际没有安全接收对应 logical output
重连时 decoder 又需要这些历史
```
第三条规则：
如果应用了 `catalog_xmin` 或 `restart_lsn`，函数会：
```text
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```
然后：
```text
updated_xmin    -> ReplicationSlotsComputeRequiredXmin(false)
updated_restart -> ReplicationSlotsComputeRequiredLSN()
```
顺序也重要。
新 xmin 要先写入 slot state，再更新 effective catalog xmin。
新 restart LSN 要先安全写入磁盘，再重新计算全局 WAL 保留需求。
这保证 crash 后不会以为更早资源已经安全可删。
## 14. restart_lsn candidate：为什么确认了还不能立刻丢 WAL
`confirmed_flush_lsn` 前进不等于 `restart_lsn` 立刻追上。
`restart_lsn` 是重新解码所需的最老 WAL。
它由 snapshot builder 和 reorder buffer 共同决定。
`SnapBuildProcessRunningXacts()` 的关键逻辑是：
```text
xmin = ReorderBufferGetOldestXmin(builder->reorder)
如果没有 reorder buffer xmin:
  xmin = running->oldestRunningXid
LogicalIncreaseXminForSlot(lsn, xmin)
```
然后它考虑 restart candidate：
```text
txn = ReorderBufferGetOldestTXN(builder->reorder)
if txn != NULL and txn->restart_decoding_lsn valid:
  LogicalIncreaseRestartDecodingForSlot(lsn, txn->restart_decoding_lsn)
else if txn == NULL and current_restart_decoding_lsn valid
        and last_serialized_snapshot valid:
  LogicalIncreaseRestartDecodingForSlot(lsn, last_serialized_snapshot)
```
这个 candidate 不是每个 commit 都更新。
注释说明原因：
改变 `restart_lsn` 意味着 slot state fsync。
所以它通常借助 running-xacts record 这种 snapshot 维护点更新。
这带来一个诊断结论：
`confirmed_flush_lsn` 已经前进，但 `restart_lsn` 暂时不动，不一定是 bug。
可能只是还没有新的 safe restart candidate。
也可能有 in-progress transaction 的 `restart_decoding_lsn` 很旧。
也可能 candidate 已经产生，但 `confirmed_flush_lsn` 尚未覆盖 `candidate_restart_valid`。
因此诊断时不要问：
```text
为什么 confirmed_flush_lsn != restart_lsn？
```
应该问：
```text
当前是否有未提交或正在 streaming 的大事务？
是否有 running-xacts record 产生新的 restart candidate？
消费者 ack 是否覆盖 candidate_restart_valid？
slot state 是否已经保存并触发 required LSN recompute？
```
## 15. 三个位置不能混用的最小反例
考虑一个长事务 A 和短事务 B。
WAL 时间线：
```text
LSN 10: A insert row 1
LSN 20: B insert row 1
LSN 30: B commit
LSN 40: A insert row 2
LSN 50: A commit
```
假设消费者已经确认到 LSN 30。
这表示：
```text
B 的 logical output 已经安全接收。
```
它不表示：
```text
decoder 下次可以从 LSN 30 开始读。
```
如果从 LSN 30 开始读，A 的 LSN 10 change 丢了。
当 LSN 50 commit 到来时，reorder buffer 不能完整输出 A。
所以 `restart_lsn` 可能仍然在 LSN 10 之前。
另一个反例来自输出位置。
在 A 的 LSN 40 处，decoder 已经读到第二条 change。
`ctx->reader->EndRecPtr` 已经越过 40。
但 A 还没 commit。
普通 non-streaming 输出插件不能把 A 当成已提交事务输出。
消费者更不可能确认 A 的 commit。
再看 streaming 大事务。
decoder 可能在 A commit 前把部分 change 以 stream change 形式发给 downstream。
但如果 A 后来 abort，downstream 需要 stream abort 清理。
因此中间 change 的 LSN 不是 `confirmed_flush_lsn` 的完整事务语义。
这三个反例共同说明：
```text
读取推进是 parser progress。
输出推进是 logical protocol progress。
确认推进是 consumer durability contract。
restart_lsn 是 crash/reconnect reconstruction boundary。
```
## 16. 断线重连：为什么可能重读但不应乱跳
logical walsender 断开后会释放本次 context。
典型收尾是：
```text
FreeDecodingContext(logical_decoding_ctx)
ReplicationSlotRelease()
```
slot 变成 inactive。
但 `slot->data.confirmed_flush` 和 `slot->data.restart_lsn` 留在 shared memory / state file 语义中。
重连时：
```text
CreateDecodingContext(cmd->startpoint, ...)
```
如果 client 没指定 startpoint，就从 `confirmed_flush` 继续。
如果 client 指定了更旧 startpoint，server 会向前修正到 `confirmed_flush`。
然后仍然：
```text
XLogBeginRead(ctx->reader, slot->data.restart_lsn)
```
所以重连通常包含两个阶段：
```text
重读旧 WAL:
  为了重建 reorder buffer 和 snapshot builder。
跳过已确认输出:
  为了不重复交付 confirmed_flush 之前的事务。
```
如果断线发生在 output 已经发送、但 consumer 没有 feedback flush 之前，`confirmed_flush` 不会前进。
重连后该事务可能再次输出。
这是 logical decoding 的至少一次交付语义。
精确一次不是 server 单独能保证的。
consumer 需要用幂等 apply、origin、事务 id、业务键或自己的 durable offset 来处理重复。
如果断线发生在 consumer 已经 flush 并发送反馈之后，但 feedback 还没被 server 处理，那么 server 也可能重复输出。
这取决于消息在连接断开时是否已经到达并执行 `ProcessStandbyReplyMessage()`。
因此从外部只能把 `confirmed_flush_lsn` 看成 publisher 端已经接受的确认边界。
不能把它看成网络上某个 ack 曾经发出过的证明。
## 17. slot save：confirmed_flush 什么时候落盘
`confirmed_flush` 是 `ReplicationSlotPersistentData` 的一部分。
persistent slot 会把它写入：
```text
pg_replslot/<slot>/state
```
但 PostgreSQL 不会每次 confirmed_flush 改变都立即 fsync slot state。
`slot.c` 的 `CheckPointReplicationSlots()` 注释说明：
slot data 不在每次 `confirmed_flush` 更新时 flush。
否则会造成频繁写。
shutdown checkpoint 时，如果 logical slot 的 `confirmed_flush` 比 `last_saved_confirmed_flush` 新，会强制 dirty。
普通 checkpoint 则保存 dirty slot。
而 `LogicalConfirmReceivedLocation()` 只有在应用了 candidate xmin 或 restart LSN 时，才会：
```text
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```
SQL `get` 路径在 confirm 后额外调用 `ReplicationSlotMarkDirty()`。
但源码也说 crash 仍可能丢掉这次位置推进。
这给出两个层次：
```text
内存 confirmed_flush:
  当前运行中 publisher 接受的确认边界。
磁盘 confirmed_flush:
  server crash / restart 后能恢复的确认边界。
```
运行时诊断通常看内存视图：
```text
pg_replication_slots.confirmed_flush_lsn
```
crash-safety 诊断要问：
```text
slot 是否 persistent？
最近是否有 checkpoint 或 shutdown checkpoint？
此次推进是否伴随 restart_lsn / catalog_xmin save？
client 自己是否有 durable offset？
```
这也是为什么 reliable logical consumer 不能只依赖 server slot state。
它还要把自己的 apply/flush 位置持久化。
PostgreSQL 内置 subscriber 通过本地 WAL flush 和 feedback discipline 来实现这一层。
第三方 consumer 必须自己实现。
## 18. 正确性机制层次
本节涉及的正确性不是一个锁解决的。
它是多层机制叠起来的。
| 层次 | 机制 | 保证 | 不能保证 |
| --- | --- | --- | --- |
| WAL 读取 | `XLogReaderState`、`restart_lsn` | decoder 能从足够早位置重建事务。 | 不代表 output 已送达。 |
| 事务语义 | reorder buffer | 只在 commit / prepare / stream 边界输出正确事务语义。 | 不代表 consumer 已持久化。 |
| 输出语义 | `ctx->write_location`、`ctx->end_xact` | output plugin 知道当前 logical record 对应哪个 LSN。 | 不代表可推进 slot。 |
| 消费确认 | `LogicalConfirmReceivedLocation()` | 只向前接受 consumer flush ack。 | 无法验证第三方 consumer 是否真的 durable。 |
| WAL 保留 | `candidate_restart_lsn`、`ReplicationSlotsComputeRequiredLSN()` | 只有确认覆盖 safe point 后才放宽 restart。 | 不保证 confirmed_flush 与 restart_lsn 接近。 |
| crash safety | `ReplicationSlotSave()`、checkpoint | 持久 slot state 能在 crash 后恢复。 | 不保证每次 confirmed_flush 更新都立即落盘。 |
锁边界也要分清。
slot 字段更新时使用 `slot->mutex`。
`ReplicationSlotSave()` 使用 slot I/O lock 和 state file fsync。
`ReplicationSlotsComputeRequiredLSN()` 会扫描所有 slot，把每个有效 slot 的 restart boundary 聚合给 WAL 删除逻辑。
这些锁保护内存一致性和持久化顺序。
它们不定义 logical 消费语义。
logical 消费语义来自：
```text
consumer 什么时候发送 flush feedback
server 是否接受该 feedback
confirmed_flush 是否单调前进
restart candidate 是否已经被确认覆盖
```
## 19. 异常路径一：旧 ack 与重复输出
`LogicalConfirmReceivedLocation()` 不允许 confirmed_flush 回退。
收到旧 ack 时只忽略。
这解决的是重复输出风险。
场景：
```text
consumer 收到事务 T
consumer 发现 T 对自己无业务影响
consumer ack 到 T 的 commit LSN
consumer 没把这个 ack 位置写到自己的 durable metadata
consumer 重启后只记得更旧位置
consumer 发送旧 flush ack
```
如果 server 接受回退，slot 会认为 T 未确认。
后续重连可能重复输出 T。
所以 server 选择：
```text
confirmed_flush 只能前进。
```
这不是说重复输出永远不会发生。
如果之前的新 ack 从未到达 server，server 当然不知道。
本节只说明：
一旦 publisher 端已经接受了更高确认，旧确认不能撤销它。
## 20. 异常路径二：ack 太慢与 candidate 被卡住
`LogicalIncreaseRestartDecodingForSlot()` 有一个保守策略。
如果新 `restart_lsn` 早于等于当前 `data.restart_lsn`，忽略。
如果 `current_lsn <= confirmed_flush`，candidate 可以立刻应用。
如果还没有 pending candidate，记录 candidate。
如果已经有 pending candidate，而 consumer ack 太慢，新的 candidate 可能只打印 debug 信息，不覆盖旧 candidate。
源码注释说：
```text
A missed value here will just cause some extra effort after reconnecting.
```
也就是说，最坏结果通常是：
```text
restart_lsn 比理论最优值更旧
重连后多读一些 WAL
pg_wal 多保留一些 segment
```
它不应该导致跳过必要 WAL。
这是典型的 PostgreSQL 保守边界：
可以接受额外读取和额外保留。
不能接受确认不足时提前释放历史。
诊断时如果看到 `restart_lsn` 停住，要先确认消费者反馈是否持续。
再看是否有长事务或 snapshot builder 还没有产生新的 safe restart point。
不要先把它归因于 walsender 没有读 WAL。
## 21. 异常路径三：peek、ERROR 与本地状态丢弃
`peek` 的异常路径很适合作边界练习。
它会建立 context。
它会读取 WAL。
它会调用 output plugin。
它会把行放进 tuplestore。
但它不会确认。
因此任何 ERROR 后，context 被丢弃，slot 进度不变。
下次仍可重新读取。
output plugin ERROR 也是类似。
`logical.c` 的 callback wrapper 会把 error context 补充成：
```text
slot name
output plugin name
callback name
associated LSN
```
ERROR 跳出后，本次 reorder buffer、snapshot builder、reader 都是 backend-local 状态。
它们可以通过 memory context 和 cleanup 路径释放。
跨会话确认边界仍在 slot。
如果 ERROR 发生在 consumer 已经收到部分 output 之后、确认之前，后续可能重复输出。
这正是 logical consumer 要按 commit 边界和 durable ack 设计的原因。
不要在 change callback 中把“已经写出一行”理解成 server 已确认消费。
server 只在 `LogicalConfirmReceivedLocation()` 后才更新 slot 确认边界。
## 22. 异常路径四：WAL removed 与 slot invalidation
`confirmed_flush_lsn` 停住会间接制造 WAL retention 压力。
但真正保护 WAL 的是 `restart_lsn`。
如果 `restart_lsn` 长期很旧，checkpoint 删除旧 WAL 时会受到 slot 约束。
如果 `max_slot_wal_keep_size` 有上限，超过后 slot 可能被标记为 lost。
`pg_replication_slots.wal_status` 会从 `reserved`、`extended`、`unreserved` 走向 `lost`。
此时问题链是：
```text
consumer 不确认
  -> confirmed_flush_lsn 不动
  -> candidate_restart_lsn 不能应用
  -> restart_lsn 不动
  -> required WAL 变旧
  -> pg_wal 增长或 slot lost
```
也可能是另一条链：
```text
consumer 确认正常
  -> confirmed_flush_lsn 前进
  -> 但存在长事务 / 无 safe snapshot candidate
  -> restart_lsn 仍旧
  -> WAL 仍被保留
```
这两条链的处理完全不同。
前者查 consumer feedback。
后者查事务、snapshot builder、reorder buffer 和 running-xacts。
因此诊断必须同时看：
```text
confirmed_flush_lsn
restart_lsn
active
wal_status
safe_wal_size
pg_stat_replication.flush_lsn
pg_stat_subscription.received_lsn / latest_end_lsn
```
单看一个字段很容易误判。
## 23. 成本、资源与跨模块传播
`confirmed_flush_lsn` 本身只是 8 字节 LSN。
它的成本来自状态不推进造成的传播。
第一类成本是重读 WAL。
如果 `restart_lsn` 很旧，重连或 slot advance fast-forward 需要从旧位置重新 digest WAL。
这会消耗 CPU、WAL I/O、catalog lookup 和 reorder buffer 内存。
第二类成本是 WAL 保留。
`restart_lsn` 通过：
```text
ReplicationSlotsComputeRequiredLSN()
  -> XLogSetReplicationSlotMinimumLSN()
  -> checkpoint / KeepLogSeg()
```
传播到 `pg_wal` 删除边界。
第三类成本是 slot state fsync。
每次应用 `candidate_restart_lsn` 或 `candidate_catalog_xmin` 都要保存 slot state。
所以源码不会在每个 commit 后推进 restart。
它把推进机会放到 running-xacts 和 snapshot serialization 相关边界。
第四类成本是反馈频率。
logical worker 的 `wal_receiver_status_interval` 控制周期性反馈。
反馈太慢会让 publisher 端 `confirmed_flush_lsn`、sync replication release、slot candidate 应用都变慢。
反馈太频繁又会增加网络和 wakeup 成本。
第五类成本是 consumer 侧 durable apply。
PostgreSQL subscriber 不会把未 flush 的本地事务报成 flush。
如果本地 WAL flush 慢，publisher 端的 confirmed boundary 也会慢。
所以一个 publisher 上的 `confirmed_flush_lsn` lag，可能根因在 subscriber 本地 I/O。
## 24. 观测入口：先看三个 LSN
publisher 上最直接的入口：
```sql
SELECT slot_name,
       active,
       restart_lsn,
       confirmed_flush_lsn,
       wal_status,
       safe_wal_size
FROM pg_replication_slots
WHERE slot_type = 'logical';
```
判断顺序：
```text
confirmed_flush_lsn 是否前进？
restart_lsn 是否前进？
二者之间差距是否扩大？
wal_status 是否接近 lost？
safe_wal_size 是否下降？
slot 是否 active？
```
如果 slot active，再看 walsender 反馈：
```sql
SELECT pid,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       reply_time
FROM pg_stat_replication;
```
对 logical slot，`flush_lsn` 是 walsender 收到的 downstream flush feedback。
它是推进 `confirmed_flush_lsn` 的直接来源。
如果 `sent_lsn` 前进而 `flush_lsn` 不前进，问题不在 decoder 读 WAL。
问题在下游接收、apply、flush 或 feedback。
subscriber 上可看：
```sql
SELECT subname,
       worker_type,
       received_lsn,
       latest_end_lsn,
       last_msg_send_time,
       last_msg_receipt_time
FROM pg_stat_subscription;
```
`received_lsn` 接近 publisher `sent_lsn`，但 publisher `flush_lsn` 不前进时，要怀疑 subscriber apply 或本地 flush。
如果 subscriber 没有 pending transaction，`send_feedback()` 会把 `recvpos` 同时报成 write/flush。
如果有 pending local flush mapping，flush 只能推进到已经本地 flush 的 remote LSN。
这能解释：
```text
网络正常收到数据
但 publisher confirmed_flush_lsn 仍滞后
```
## 25. 诊断入口：日志与断点
源码跟读实验可以打这些断点：
```text
CreateDecodingContext
StartupDecodingContext
XLogBeginRead
LogicalDecodingProcessRecord
commit_cb_wrapper
WalSndPrepareWrite
ProcessStandbyReplyMessage
LogicalConfirmReceivedLocation
LogicalIncreaseRestartDecodingForSlot
SnapBuildProcessRunningXacts
```
关键观察变量：
```text
MyReplicationSlot->data.confirmed_flush
MyReplicationSlot->data.restart_lsn
MyReplicationSlot->candidate_restart_valid
MyReplicationSlot->candidate_restart_lsn
ctx->reader->ReadRecPtr
ctx->reader->EndRecPtr
ctx->write_location
ctx->end_xact
flushPtr in ProcessStandbyReplyMessage()
```
在 `LogicalConfirmReceivedLocation()` 前后观察：
```text
传入 lsn 是否大于 confirmed_flush？
是否存在 candidate_restart_valid？
candidate_restart_valid 是否 <= lsn？
是否调用 ReplicationSlotSave()？
是否调用 ReplicationSlotsComputeRequiredLSN()？
```
在 `CreateDecodingContext()` 观察：
```text
传入 start_lsn 是 InvalidXLogRecPtr 还是客户端指定值？
slot->data.confirmed_flush 是多少？
是否发生 start_lsn 向前修正？
```
在 `XLogBeginRead()` 观察：
```text
读取起点是否仍是 slot->data.restart_lsn？
```
这组断点能直接验证本节模型：
```text
输出基准来自 confirmed_flush。
读取起点来自 restart_lsn。
确认推进来自 downstream flush feedback 或 SQL get。
restart_lsn 推进还要等 candidate 被确认覆盖。
```
## 26. 常见误区
误区一：
```text
confirmed_flush_lsn 就是 walsender 已经发送到的位置。
```
不是。
walsender 发送位置看 `sent_lsn`。
`confirmed_flush_lsn` 来自 consumer flush feedback 或 SQL `get` 的确认。
误区二：
```text
confirmed_flush_lsn 前进后 restart_lsn 应该马上追上。
```
不是。
restart 需要 safe restart candidate。
长事务、snapshot serialization、candidate valid LSN 和 slot save 都会让它滞后。
误区三：
```text
ctx->reader->EndRecPtr 可以当成消费者确认位置。
```
只能在 SQL `get` 这种 confirm 路径里，由函数完成输出后显式调用 `LogicalConfirmReceivedLocation()`。
walsender 不能这样做。
误区四：
```text
peek 不读 WAL。
```
peek 读 WAL，也做 decoding。
它只是不确认。
误区五：
```text
output plugin 写出 change 后，该 change 就不可重复。
```
不是。
重复边界由 consumer ack 和 confirmed_flush 决定。
确认前断线或 ERROR，都可能重复输出。
误区六：
```text
confirmed_flush_lsn 是完全 crash-safe 的最新位置。
```
不总是。
内存字段前进和 slot state 落盘是两个层次。
SQL 路径会 mark dirty，shutdown checkpoint 会更积极保存，但 crash 仍可能丢最近位置。
## 27. 课堂实验一：peek 与 get 的确认差异
准备一个 test decoding slot。
```sql
SELECT * FROM pg_create_logical_replication_slot('s_confirm', 'test_decoding');
CREATE TABLE t_confirm(id int primary key, v text);
INSERT INTO t_confirm VALUES (1, 'a');
```
先看 slot：
```sql
SELECT restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 's_confirm';
```
连续执行两次：
```sql
SELECT * FROM pg_logical_slot_peek_changes('s_confirm', NULL, NULL);
```
预期：
```text
两次都能看到同一批输出。
confirmed_flush_lsn 不应因为 peek 前进。
```
然后执行：
```sql
SELECT * FROM pg_logical_slot_get_changes('s_confirm', NULL, NULL);
```
再看：
```sql
SELECT restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 's_confirm';
```
预期：
```text
confirmed_flush_lsn 前进。
restart_lsn 可能前进，也可能暂时不动。
```
源码回扣：
```text
peek:
  confirm = false
  不调用 LogicalConfirmReceivedLocation()
get:
  confirm = true
  LogicalConfirmReceivedLocation(ctx->reader->EndRecPtr)
```
## 28. 课堂实验二：观察 walsender feedback
在 publisher 建 logical publication 和 subscriber。
让 subscriber 正常运行后，在 publisher 查：
```sql
SELECT slot_name, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_type = 'logical';
```
再查：
```sql
SELECT pid, state, sent_lsn, write_lsn, flush_lsn, replay_lsn, reply_time
FROM pg_stat_replication;
```
然后在 subscriber 制造本地 flush 压力。
例如让磁盘繁忙，或在测试环境里用断点停在 `store_flush_position()` 之后、`send_feedback()` 之前。
观察：
```text
publisher sent_lsn 可以继续变化。
publisher flush_lsn / confirmed_flush_lsn 可能停住。
restart_lsn 也可能停住。
```
源码回扣：
```text
XLogSendLogical():
  sentPtr = ctx->reader->EndRecPtr
ProcessStandbyReplyMessage():
  LogicalConfirmReceivedLocation(flushPtr)
logical worker:
  get_flush_position() 只把本地已 flush 的 remote_end 报成 flushpos
```
## 29. 讨论题
1. 为什么 `CreateDecodingContext()` 要把过旧 `start_lsn` 向前修正到 `confirmed_flush`，而不是直接报错？
2. 为什么 SQL `get` 可以用 `ctx->reader->EndRecPtr` 调用 `LogicalConfirmReceivedLocation()`，而 walsender 不能在读完 record 后直接确认？
3. 如果 `confirmed_flush_lsn` 前进但 `restart_lsn` 不动，至少列出三种可能原因。
4. `ctx->write_location` 在 change callback 和 commit callback 中分别表示什么？为什么 change LSN 不能直接代表事务确认？
5. 为什么 `LogicalConfirmReceivedLocation()` 只允许 confirmed_flush 前进，不允许回退？
6. 如果 consumer 在收到 output 后、持久化 offset 前崩溃，publisher 端的 `confirmed_flush_lsn` 可能出现哪些情况？
7. 为什么应用 `candidate_restart_lsn` 之前必须等 `candidate_restart_valid <= confirmed_flush_lsn`？
## 30. 本节小结
本节的核心链路是：
```text
slot->data.restart_lsn
  -> XLogBeginRead(ctx->reader, restart_lsn)
  -> ctx->reader->ReadRecPtr / EndRecPtr
  -> LogicalDecodingProcessRecord()
  -> reorder buffer commit / stream callbacks
  -> ctx->write_location / ctx->end_xact
  -> SQL get 或 walsender feedback
  -> LogicalConfirmReceivedLocation()
  -> slot->data.confirmed_flush
  -> candidate restart/xmin 可能被应用
  -> ReplicationSlotSave() / RequiredLSN recompute
```
核心状态边界是：
```text
LogicalDecodingContext:
  backend-local，本次会话的 reader、reorder、snapshot builder 和 output state。
ReplicationSlot:
  shared / persistent，跨会话保存 confirmed_flush、restart_lsn、catalog_xmin。
```
读取位置、输出位置和确认位置必须分开。
`ctx->reader->EndRecPtr` 只说明 decoder 读过 WAL record。
`ctx->write_location` 只说明当前 output callback 关联哪个 logical LSN。
`slot->data.confirmed_flush` 才说明 publisher 接受了 consumer 对该 LSN 的 flush 确认。
即使确认前进，`restart_lsn` 也必须等 safe restart candidate 被确认覆盖后才能推进。
这保证断线重连和 crash 后仍能从 WAL 重建未完成事务、snapshot 和 catalog 状态。
本节可迁移的系统规律是：
```text
一个 durable stream 协议里，parser progress、producer output progress、
consumer durable ack 和 replay reconstruction boundary 通常是四个状态。
把它们合并成一个 offset，会把重复、丢失、资源回收和恢复语义混在一起。
```
PostgreSQL 在 logical decoding 中选择保守推进：
可以重复读取 WAL。
可以让 `restart_lsn` 暂时滞后。
可以让消费者在未确认时看到重复输出。
但不能在消费者确认不足时释放 decoder 仍可能需要的 WAL 或 catalog 历史。
