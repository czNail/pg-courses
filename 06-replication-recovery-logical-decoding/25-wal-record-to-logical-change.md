# PostgreSQL WAL record 到 logical change 的解释链路
## 课程定位
前置知识：已经理解 WAL 按 LSN 追加，logical slot 保护 `restart_lsn` 和 `catalog_xmin`，也已经理解 `ReorderBuffer` 会把 WAL record 重新组织成事务级输出。
本节唯一主问题：
```text
heap、transaction、relation、database 等 WAL record
如何被 logical decoding 解释成 insert、update、delete、truncate、message 和 commit 事件？
```
核心矛盾：WAL record 首先服务 crash recovery，描述的是物理页面、事务状态、catalog invalidation 和 rmgr 私有语义；logical consumer 需要的却是按事务提交边界交付的关系级事件。
PostgreSQL 的做法不是让每一种 WAL record 都直接输出事件。
它先要求 WAL 写入端携带 logical decoding 所需的额外信息。
再由 `LogicalDecodingProcessRecord()` 按 rmgr 分派。
heap record 只被解释成 `ReorderBufferChange`。
transaction commit record 才触发 `ReorderBufferCommit()`。
最后 output plugin 才把关系级 change 编码成协议消息。
学完后应能判断：为什么不是所有 `RM_HEAP_ID` record 都能解码；为什么 `RM_DBASE_ID`、`RM_RELMAP_ID`、btree/seq WAL 不直接变成 logical DML；为什么 database filter 在 `decode.c` 很早发生，而 relation/publication filter 在 output plugin 边界才发生；为什么 abort 事务的 heap record 可以已经进过 reorder buffer，但最终不会输出。
本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
注意一个命名边界：一些资料会把 heap/xact 的 rmgr decode callback 泛称为 `DecodeHeapOp`、`DecodeXactOp`，但当前源码中的真实函数名是 `heap_decode()`、`heap2_decode()`、`xact_decode()`。
同理，本课为说明方便会说“DecodeMessage 概念入口”，当前源码真实入口是 `logicalmsg_decode()`，并没有名为 `DecodeMessage()` 的函数。
## 1. 本节在总主线中的位置
前两节已经讨论过 reorder buffer：
```text
WAL 顺序读取
  -> 按 XID 聚合 change
  -> commit record 到达
  -> 按事务顺序输出
```
本节把焦点前移一层：
```text
一条原始 WAL record 怎么知道自己是不是 logical decoding 关心的 change？
```
再向后延伸一层：
```text
一个 ReorderBufferChange 怎么变成 pgoutput 的 INSERT / UPDATE / DELETE / TRUNCATE / MESSAGE？
```
本节不展开 slot 生命周期、spill 文件格式、publication DDL 语法或 subscriber apply worker。
这些模块都会出现，但只作为边界。
主线只跟一条 record 的解释过程：
```text
heapam.c / xact.c / message.c 写入 WAL
  -> XLogReader 读出 record
  -> LogicalDecodingProcessRecord() 分派 rmgr
  -> heap_decode() / xact_decode() / logicalmsg_decode()
  -> DecodeInsert() / DecodeUpdate() / DecodeDelete() / DecodeTruncate()
  -> ReorderBufferQueueChange() 或 ReorderBufferQueueMessage()
  -> DecodeCommit() / DecodeAbort()
  -> ReorderBufferCommit() / ReorderBufferAbort()
  -> ReorderBufferProcessTXN()
  -> logical.c callback wrapper
  -> pgoutput.c relation/publication filter
  -> proto.c 编码协议消息
```
这条链路里最容易犯的错误，是把 WAL record 类型和最终 logical 事件类型一一对应。
实际关系更保守。
只有一部分 heap/logicalmsg/xact WAL record 有 logical decode callback。
只有满足 snapshot、database、origin、relation、publication、replica identity 和 commit 边界的 change 才会输出。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
WAL 写入端在 wal_level/effective_wal_level 允许时，把 logical decoding 所需的 tuple image、
old key、dbId、subxact、origin 和 logical message 写入 WAL；
logical decoding 读 WAL 时按 rmgr callback 只把可解释的 record 转成 ReorderBufferChange；
transaction commit record 再把同一 XID 下的 change 作为事务事件交给 output plugin。
```
tension 是：
```text
WAL 必须足够物理，才能高效恢复页面
  vs
logical decoding 必须足够逻辑，才能给外部系统稳定的关系级语义
```
如果 WAL 只保存页面 delta，logical decoding 在没有数据页前态、没有历史 catalog、没有事务提交边界时无法安全还原行级语义。
如果 WAL 直接保存最终 logical stream，crash recovery、page LSN、FPI、rmgr redo 和 access method 的物理恢复成本会变得不可接受。
PostgreSQL 把问题拆开：
```text
heapam.c:
  在必要时把 tuple image 或 replica identity old tuple/key 放进 heap WAL。
xact.c:
  在 commit/abort record 里写 dbId、subxacts、origin、invalidation、2PC 信息。
message.c:
  用 RM_LOGICALMSG_ID 写插件可消费的 arbitrary message。
decode.c:
  只做 WAL record -> reorder buffer change 的局部解释。
reorderbuffer.c:
  等 commit/abort 边界决定输出、丢弃、执行 invalidation 或 cleanup。
pgoutput.c / proto.c:
  把 relation/publication 语义编码给下游。
```
所以本节的核心模型不是“读取 WAL 就输出行变更”。
更准确的模型是：
```text
读取 WAL:
  只收集候选 change。
遇到 commit:
  才证明这些 change 属于一个可见事务。
进入 output plugin:
  才根据 relation/publication/row filter/column list 决定具体发什么。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/access/rmgrlist.h` | 哪些 rmgr 提供 `rm_decode`，哪些只是 physical redo。 |
| 2 | `src/backend/replication/logical/decode.c` | `LogicalDecodingProcessRecord()`、`heap_decode()`、`heap2_decode()`、`xact_decode()`、`logicalmsg_decode()`。 |
| 3 | `src/include/access/heapam_xlog.h` | `xl_heap_insert`、`xl_heap_update`、`xl_heap_delete`、`xl_heap_truncate` 和 flags。 |
| 4 | `src/backend/access/heap/heapam.c` | heap WAL 写入端如何设置 `XLH_*_CONTAINS_*` 和 replica identity old tuple/key。 |
| 5 | `src/backend/access/transam/xact.c` | `XactLogCommitRecord()`、`XactLogAbortRecord()` 如何携带 dbId、subxacts、origin。 |
| 6 | `src/backend/replication/logical/message.c` | `LogLogicalMessage()` 如何写 `RM_LOGICALMSG_ID` record。 |
| 7 | `src/include/replication/reorderbuffer.h` | `ReorderBufferChange`、`ReorderBufferTXN`、callback 类型。 |
| 8 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferQueueChange()`、`ReorderBufferCommit()`、`ReorderBufferAbort()`、`ReorderBufferProcessTXN()`。 |
| 9 | `src/backend/replication/logical/logical.c` | callback wrapper、error context、write LSN、`CheckLogicalDecodingRequirements()`。 |
| 10 | `src/backend/replication/pgoutput/pgoutput.c` | relation/publication/row filter/column list 过滤。 |
| 11 | `src/backend/replication/logical/proto.c` | `logicalrep_write_insert()`、`logicalrep_write_update()`、`logicalrep_write_delete()`、`logicalrep_write_truncate()`、`logicalrep_write_message()`。 |
推荐阅读路径不是按文件大小。
先看 `rmgrlist.h`，因为它直接告诉你哪些 WAL record 有 logical decode callback。
再看 `decode.c`，确认每类 record 的分派和早期过滤。
然后回到 `heapam.c` 和 `xact.c` 看写入端为什么会有这些字段。
最后读 `reorderbuffer.c` 和 `pgoutput.c`，理解“候选 change”如何在 commit 和 publication 边界后才变成输出。
## 4. WAL 写入端先决定信息够不够
logical decoding 不是从任意物理 WAL 中推理出完整 tuple。
它依赖写入端在 WAL record 中保留足够信息。
最底层开关在 `src/include/access/xlog.h`：
```text
XLogLogicalInfoActive()
  -> wal_level >= WAL_LEVEL_LOGICAL || XLogLogicalInfo
```
当前 master 还支持 dynamic logical decoding availability。
`XLogLogicalInfo` 是 process-local cache。
它可以在 `wal_level = replica` 且有 logical slot 时让进程写入 logical decoding 所需信息。
`src/backend/replication/logical/logicalctl.c` 的 `EnsureLogicalDecodingEnabled()` 会在创建或使用 logical slot 时确保系统进入可解码状态。
入口侧还会检查环境。
`src/backend/replication/logical/logical.c` 的 `CheckLogicalDecodingRequirements()` 先调用 slot requirement 检查，再要求当前有 database connection。
它还在 standby 上检查 `IsLogicalDecodingEnabled()`。
这解释了一个诊断现象：
```text
slot 能不能创建、WAL 能不能携带 logical 信息、decoder 能不能启动，
不是同一个状态。
```
heap relation 是否把 tuple image 写进 WAL，取决于 `RelationIsLogicallyLogged(relation)`。
这个宏在 `src/include/utils/rel.h`。
它要求：
```text
XLogLogicalInfoActive()
RelationNeedsWAL(relation)
relation 不是 foreign table
relation 不是普通 system catalog
```
普通系统表内容不会作为用户 DML 输出。
用户定义 catalog table 另当别论。
`heap_insert()` 中，当 relation 需要 logical logging 且不是 `HEAP_INSERT_NO_LOGICAL`，会设置 `XLH_INSERT_CONTAINS_NEW_TUPLE`。
同时会用 `REGBUF_KEEP_DATA` 保证即使有 full-page image，logical decoding 仍能从 record 中取到 tuple data。
`log_heap_update()` 中，`need_tuple_data = walLogical && RelationIsLogicallyLogged(reln)`。
如果需要 logical tuple data，就设置 `XLH_UPDATE_CONTAINS_NEW_TUPLE`。
如果有 `old_key_tuple`，还根据 replica identity 设置 `XLH_UPDATE_CONTAINS_OLD_TUPLE` 或 `XLH_UPDATE_CONTAINS_OLD_KEY`。
`heap_delete()` 在进入 critical section 前先用 `ExtractReplicaIdentity()` 构造 old key tuple。
写 WAL 时，如果 old key 存在，就设置 `XLH_DELETE_CONTAINS_OLD_TUPLE` 或 `XLH_DELETE_CONTAINS_OLD_KEY`。
如果调用者明确不想 logical decode 这次删除，则设置 `XLH_DELETE_NO_LOGICAL`。
因此，解码端看到的不是“堆页面上的所有变化”。
它看到的是写入端已经承诺可供 logical decoding 解释的那部分变化。
## 5. 关键状态：不是 record 单独决定语义
`LogicalDecodingContext` 是当前 decoding backend 的顶层状态。
它持有：
| 状态 | 语义 |
| --- | --- |
| `reader` | 当前 `XLogReaderState`，提供 `ReadRecPtr`、`EndRecPtr`、rmgr、xid、block data、origin。 |
| `slot` | 当前 logical slot，包含 slot database、`confirmed_flush`、`restart_lsn` 等边界。 |
| `snapshot_builder` | `SnapBuild`，决定当前是否已有 full snapshot，某个事务是否应跳过。 |
| `reorder` | `ReorderBuffer`，按 XID 聚合 change 并在 commit 回放。 |
| `callbacks` | output plugin callback。 |
| `fast_forward` | 只推进 slot/snapshot，不输出 change。 |
| `processing_required` | fast-forward 时发现本来需要处理的 record，用于外层控制。 |
`ReorderBufferChange` 是候选 logical change。
核心字段在 `src/include/replication/reorderbuffer.h`：
```text
lsn
action
txn
origin_id
data.tp.rlocator
data.tp.oldtuple
data.tp.newtuple
data.truncate.relids
data.msg.prefix/message
```
`action` 只描述 reorder buffer 内部动作。
它可能是对外 DML：
```text
REORDER_BUFFER_CHANGE_INSERT
REORDER_BUFFER_CHANGE_UPDATE
REORDER_BUFFER_CHANGE_DELETE
REORDER_BUFFER_CHANGE_TRUNCATE
REORDER_BUFFER_CHANGE_MESSAGE
```
也可能是内部动作：
```text
REORDER_BUFFER_CHANGE_INTERNAL_SNAPSHOT
REORDER_BUFFER_CHANGE_INTERNAL_COMMAND_ID
REORDER_BUFFER_CHANGE_INTERNAL_TUPLECID
REORDER_BUFFER_CHANGE_INTERNAL_SPEC_INSERT
REORDER_BUFFER_CHANGE_INTERNAL_SPEC_CONFIRM
REORDER_BUFFER_CHANGE_INTERNAL_SPEC_ABORT
REORDER_BUFFER_CHANGE_INVALIDATION
```
`ReorderBufferTXN` 是按 XID 聚合出来的事务状态。
重要字段组合包括：
| 字段 | 语义 |
| --- | --- |
| `xid`、`toplevel_xid`、`toptxn` | 当前 XID 与顶层事务关系。 |
| `first_lsn`、`final_lsn`、`end_lsn` | 第一次相关 record、commit/abort record 起点、commit record 结束后位置。 |
| `base_snapshot` | 输出 change 时使用的 historic snapshot。 |
| `changes` | 当前事务自己的 change 链表。 |
| `subtxns` | 已知未 abort 的 subtransaction。 |
| `tuplecids` | catalog-changing 事务需要的 cmin/cmax/combocid 映射。 |
| `invalidations` | commit 时要执行的 cache invalidation。 |
| `txn_flags` | prepared、streamed、aborted、catalog change、spill 等状态。 |
raw field 不是语义。
`change->action` 只有结合 transaction commit、snapshot builder 状态、relation lookup 和 output plugin 过滤后，才成为外部可见事件。
## 6. 主流程源码 walkthrough：`LogicalDecodingProcessRecord()` 的固定分派
主入口在 `src/backend/replication/logical/decode.c`。
它把 `XLogReaderState` 包装成 `XLogRecordBuffer`：
```text
buf.origptr = reader->ReadRecPtr
buf.endptr  = reader->EndRecPtr
buf.record  = record
```
然后读取 top-level XID：
```text
txid = XLogRecGetTopXid(record)
```
如果 top-level XID 有效，就先执行：
```text
ReorderBufferAssignChild(ctx->reorder,
                         txid,
                         XLogRecGetXid(record),
                         buf.origptr)
```
这一步解释了为什么 `XLOG_XACT_ASSIGNMENT` 在 `xact_decode()` 里可以是 no-op。
当前 record 如果已经带了 top xid，decode 入口已经把 subxid 关系告诉 reorder buffer。
接着按 rmgr 分派：
```text
rmgr = GetRmgr(XLogRecGetRmid(record))
if rmgr.rm_decode != NULL:
    rmgr.rm_decode(ctx, &buf)
else:
    ReorderBufferProcessXid(ctx->reorder, XLogRecGetXid(record), buf.origptr)
```
`rmgrlist.h` 告诉我们当前有 logical decode callback 的 rmgr：
```text
RM_XLOG_ID        -> xlog_decode
RM_XACT_ID        -> xact_decode
RM_STANDBY_ID     -> standby_decode
RM_HEAP2_ID       -> heap2_decode
RM_HEAP_ID        -> heap_decode
RM_LOGICALMSG_ID  -> logicalmsg_decode
RM_XLOG2_ID       -> xlog2_decode
```
`RM_DBASE_ID`、`RM_RELMAP_ID`、`RM_BTREE_ID`、`RM_SEQ_ID` 等没有 `rm_decode`。
它们不是被 logical decoding “忘了”。
这些 record 的语义通常是物理恢复、catalog/relfilenumber 变化、storage 生命周期或 access method 内部状态。
它们不直接对应用户表上的 INSERT/UPDATE/DELETE。
没有 `rm_decode` 的 record 仍可能用 `ReorderBufferProcessXid()` 更新 reorder buffer 对 XID/LSN 的认识。
但它不会创建对外 change。
## 7. heap rmgr：从物理 heap record 到候选 change
`heap_decode()` 处理 `RM_HEAP_ID`。
它先取：
```text
info = XLogRecGetInfo(record) & XLOG_HEAP_OPMASK
xid  = XLogRecGetXid(record)
```
然后执行：
```text
ReorderBufferProcessXid(ctx->reorder, xid, buf->origptr)
```
这一步保证即使 record 最终不输出，也可能影响事务 LSN 跟踪。
接着是 snapshot 门槛。
如果 `SnapBuildCurrentState(builder) < SNAPBUILD_FULL_SNAPSHOT`，直接返回。
没有 full snapshot 时，decoder 不能安全解释用户数据变化。
之后按 heap op 分派。
核心分支是：
```text
XLOG_HEAP_INSERT:
  SnapBuildProcessChange(...) && !fast_forward && !change_useless_for_repack(...)
  -> DecodeInsert()
XLOG_HEAP_UPDATE / XLOG_HEAP_HOT_UPDATE:
  -> DecodeUpdate()
XLOG_HEAP_DELETE:
  -> DecodeDelete()
XLOG_HEAP_TRUNCATE:
  -> DecodeTruncate()
XLOG_HEAP_CONFIRM:
  -> DecodeSpecConfirm()
XLOG_HEAP_INPLACE:
  ignore
XLOG_HEAP_LOCK:
  ignore
```
`heap2_decode()` 处理 `RM_HEAP2_ID`。
它关心：
```text
XLOG_HEAP2_MULTI_INSERT -> DecodeMultiInsert()
XLOG_HEAP2_NEW_CID      -> SnapBuildProcessNewCid()
XLOG_HEAP2_REWRITE      -> no-op in decode path
```
prune、vacuum cleanup、lock updated 等 record 是 physical heap maintenance。
logical decoding 不把它们输出为 DML。
这就是第一个边界：
```text
heap WAL record 数量
  !=
logical row change 数量
```
多条 physical record 可能服务一个 logical update。
一条 multi-insert record 可能产生多个 `REORDER_BUFFER_CHANGE_INSERT`。
一些 physical record 完全不会输出。
## 8. `DecodeInsert()` 与 `DecodeMultiInsert()`
`DecodeInsert()` 处理普通 `XLOG_HEAP_INSERT`。
它先检查：
```text
xlrec->flags & XLH_INSERT_CONTAINS_NEW_TUPLE
```
如果没有 new tuple，直接返回。
这通常说明写入端认为这条 insert 不应或不能用于 logical decoding。
随后通过 block tag 取 relation locator：
```text
XLogRecGetBlockTag(r, 0, &target_locator, NULL, NULL)
```
早期 database filter 在这里发生：
```text
target_locator.dbOid != ctx->slot->data.database
```
不同 database 的变化不会进入当前 slot 的 reorder buffer。
再执行 origin filter：
```text
FilterByOrigin(ctx, XLogRecGetOrigin(r))
```
通过后才分配 change：
```text
change = ReorderBufferAllocChange(ctx->reorder)
```
如果不是 speculative insert：
```text
change->action = REORDER_BUFFER_CHANGE_INSERT
```
如果是 speculative insert：
```text
change->action = REORDER_BUFFER_CHANGE_INTERNAL_SPEC_INSERT
```
`RelFileLocator` 被复制到：
```text
change->data.tp.rlocator
```
tuple data 来自 block data：
```text
tupledata = XLogRecGetBlockData(r, 0, &datalen)
tuplelen = datalen - SizeOfHeapHeader
```
`DecodeXLogTuple()` 会把 WAL 中的 `xl_heap_header` 和 tuple payload 还原成 `HeapTuple`。
它不会设置真实 `t_self` 和 `t_tableOid`。
源码里明确把它们置成无效，因为这些信息要到 replay/relation lookup 时才能解释。
最后调用：
```text
ReorderBufferQueueChange(ctx->reorder,
                         XLogRecGetXid(r),
                         buf->origptr,
                         change,
                         xlrec->flags & XLH_INSERT_ON_TOAST_RELATION)
```
`DecodeMultiInsert()` 处理 `XLOG_HEAP2_MULTI_INSERT`。
它同样要求 `XLH_INSERT_CONTAINS_NEW_TUPLE`。
它同样先做 database 和 origin filter。
然后对 `xlrec->ntuples` 循环。
每一个 `xl_multi_insert_tuple` 都生成一个 `REORDER_BUFFER_CHANGE_INSERT`。
这解释了一个常见现象：
```text
pg_waldump 看到一条 MULTI_INSERT，
logical output 可能看到多条 INSERT。
```
multi-insert 的 toast cleanup 边界也更细。
只有最后一个 tuple 且带 `XLH_INSERT_LAST_IN_MULTI` 时，`clear_toast_afterwards` 才为 true。
这是 reorder buffer 后续 toast reassembly 的输入，不是 output plugin 的可见字段。
## 9. `DecodeUpdate()`：new tuple 与 old key
`DecodeUpdate()` 处理 `XLOG_HEAP_UPDATE` 和 `XLOG_HEAP_HOT_UPDATE`。
`heap_decode()` 中已经说明：HOT update 对 logical decoding 来说按普通 update 处理。
HOT 是本地 heap 优化，不是下游需要复刻的语义。
`DecodeUpdate()` 的早期过滤和 insert 相同：
```text
block tag -> RelFileLocator
dbOid == slot database
FilterByOrigin() == false
```
通过后分配：
```text
change->action = REORDER_BUFFER_CHANGE_UPDATE
change->origin_id = XLogRecGetOrigin(r)
change->data.tp.rlocator = target_locator
```
新 tuple 是否存在取决于 flag：
```text
XLH_UPDATE_CONTAINS_NEW_TUPLE
```
如果存在，tuple data 来自 block 0 data。
旧 tuple 或旧 key 是否存在取决于：
```text
XLH_UPDATE_CONTAINS_OLD_TUPLE
XLH_UPDATE_CONTAINS_OLD_KEY
```
源码用便利宏 `XLH_UPDATE_CONTAINS_OLD` 判断。
旧 tuple/key 的 data 在 main data 中，位置从 `SizeOfHeapUpdate` 后开始。
这里的 oldtuple 不一定是整行。
它可能只是 replica identity key。
最终 pgoutput 会根据 relation 的 replica identity 把它编码成 `O` 或 `K`：
```text
REPLICA_IDENTITY_FULL  -> old tuple follows, tag 'O'
DEFAULT / INDEX        -> old key follows, tag 'K'
```
所以诊断 UPDATE/DELETE 缺 old key 时，不要只看 decoder。
要回到 heap 写入端和 relation replica identity：
```text
ALTER TABLE ... REPLICA IDENTITY FULL
或有可用 replica identity index
```
如果写入端没有记录 old key，decode 端不能从 heap page “再查一次”补回来。
logical decoding 面对的是历史 WAL stream，不是当前表状态。
## 10. `DecodeDelete()`：删除不等于一定能发布
`DecodeDelete()` 先看 `XLH_DELETE_NO_LOGICAL`。
如果该 flag 被设置，直接返回。
这用于一些不应暴露给 logical decoding 的内部变化，例如 concurrent repack 相关临时表路径。
随后同样做 database filter 和 origin filter。
通过后分配 change。
普通 delete：
```text
change->action = REORDER_BUFFER_CHANGE_DELETE
```
speculative insertion abort：
```text
change->action = REORDER_BUFFER_CHANGE_INTERNAL_SPEC_ABORT
```
旧 tuple/key 是否存在取决于：
```text
XLH_DELETE_CONTAINS_OLD_TUPLE
XLH_DELETE_CONTAINS_OLD_KEY
```
没有 old tuple/key 的 delete 仍可能进入 reorder buffer。
但 `pgoutput_change()` 在 DELETE 分支中会检查：
```text
if (!change->data.tp.oldtuple)
    return
```
它会打 DEBUG1：
```text
didn't send DELETE change because of missing oldtuple
```
这个边界非常重要。
`DecodeDelete()` 能解释 WAL record。
`pgoutput` 仍可能因为缺少可发给 subscriber 的 tuple identity 而不发送 DELETE。
所以 delete 的最终输出依赖三层：
```text
heap 写入端是否记录 old tuple/key
decode.c 是否排队 REORDER_BUFFER_CHANGE_DELETE
pgoutput.c 是否允许该 relation/action 且 oldtuple 可用
```
不要把其中任何一层单独当成完整因果。
## 11. `DecodeTruncate()`：relation OID 列表而不是 tuple
`XLOG_HEAP_TRUNCATE` 的 record 格式在 `heapam_xlog.h`：
```text
xl_heap_truncate:
  dbId
  nrelids
  flags
  relids[]
```
它没有 tuple image。
它直接记录同一 database 内被 truncate 的 relation OID 列表。
`DecodeTruncate()` 先检查：
```text
xlrec->dbId == ctx->slot->data.database
```
再检查 origin filter。
通过后创建：
```text
change->action = REORDER_BUFFER_CHANGE_TRUNCATE
```
然后把 flags 转成 reorder buffer change 字段：
```text
XLH_TRUNCATE_CASCADE       -> change->data.truncate.cascade
XLH_TRUNCATE_RESTART_SEQS  -> change->data.truncate.restart_seqs
```
`relids[]` 被复制到 reorder buffer 自己的内存中。
最终 `ReorderBufferProcessTXN()` 处理 truncate 时，会逐个 `RelationIdGetRelation(relid)`。
它仍会过滤 `RelationIsLogicallyLogged(rel)`。
然后调用 `ReorderBufferApplyTruncate()`。
`pgoutput_truncate()` 还会继续做 publication action filter：
```text
relentry->pubactions.pubtruncate
```
以及 partition root 相关过滤。
最终 `proto.c` 的 `logicalrep_write_truncate()` 编码：
```text
LOGICAL_REP_MSG_TRUNCATE
xid if streaming
nrelids
flags
relids[]
```
这解释了为什么 truncate 的 database filter 比普通 heap DML 更直接。
普通 DML record 通过 block tag 的 `RelFileLocator.dbOid` 判断 database。
truncate record 自己携带 `dbId`。
## 12. Logical message：不是 heap change
logical message 的写入端在 `src/backend/replication/logical/message.c`。
SQL 函数 `pg_logical_emit_message_text()` 和 `pg_logical_emit_message_bytea()` 最终调用：
```text
LogLogicalMessage(prefix, message, size, transactional, flush)
```
如果 message 是 transactional，写入端会强制分配 XID：
```text
GetCurrentTransactionId()
```
record 内容是：
```text
xl_logical_message:
  dbId
  transactional
  prefix_size
  message_size
  message bytes = prefix '\0' + payload
```
redo 端基本是 no-op。
这是只对 logical decoding 有意义的 rmgr。
`logicalmsg_decode()` 先要求 record type 是 `XLOG_LOGICAL_MESSAGE`。
再执行：
```text
ReorderBufferProcessXid(ctx->reorder, xid, buf->origptr)
```
没有 full snapshot 时直接返回。
然后做 database 和 origin filter：
```text
message->dbId == ctx->slot->data.database
FilterByOrigin(ctx, origin_id) == false
```
transactional message 还要通过：
```text
SnapBuildProcessChange(builder, xid, buf->origptr)
```
non-transactional message 则要求：
```text
SnapBuildCurrentState(builder) == SNAPBUILD_CONSISTENT
!SnapBuildXactNeedsSkip(builder, buf->origptr)
```
这是 transactional 和 non-transactional message 的核心差异。
transactional message 进入事务队列。
它会随 commit 输出，随 abort 消失。
non-transactional message 在读到 WAL 时就可以被 `ReorderBufferQueueMessage()` 立即调用 message callback。
`fast_forward` 模式下，non-transactional message 还会设置 `ctx->processing_required = true`。
因为它没有后续 commit/abort record 可用来提醒外层“这里本来有东西要处理”。
## 13. xact rmgr：commit/abort 才是输出边界
`xact_decode()` 处理 `RM_XACT_ID`。
它一开始就检查：
```text
SnapBuildCurrentState(builder) < SNAPBUILD_FULL_SNAPSHOT
```
如果还没有 full snapshot，直接返回。
commit record 分支处理：
```text
XLOG_XACT_COMMIT
XLOG_XACT_COMMIT_PREPARED
```
它解析：
```text
ParseCommitRecord(...)
```
得到 `xl_xact_parsed_commit`。
然后决定 xid：
```text
parsed.twophase_xid valid ? parsed.twophase_xid : XLogRecGetXid(r)
```
普通 commit 进入 `DecodeCommit()`。
abort record 分支类似：
```text
XLOG_XACT_ABORT
XLOG_XACT_ABORT_PREPARED
ParseAbortRecord(...)
DecodeAbort()
```
`XLOG_XACT_ASSIGNMENT` 当前不需要额外做事。
原因前面讲过：`LogicalDecodingProcessRecord()` 已经在每条 record 入口处理 top xid / subxid 关系。
`XLOG_XACT_INVALIDATIONS` 比较特殊。
带 XID 的 invalidation 会被：
```text
ReorderBufferAddInvalidations()
ReorderBufferXidSetCatalogChanges()
```
无 XID 的 invalidation 会立即执行：
```text
ReorderBufferImmediateInvalidation()
```
这说明 transaction WAL record 不只是 commit/abort 边界。
它还携带 logical decoding 读取历史 catalog 时必须尊重的 cache invalidation。
## 14. `DecodeCommit()`：从事务 record 到 `ReorderBufferCommit()`
`DecodeCommit()` 先处理 replication origin。
如果 `parsed->xinfo & XACT_XINFO_HAS_ORIGIN`，commit time 和 origin LSN 使用 origin 信息。
然后通知 snapshot builder：
```text
SnapBuildCommitTxn(ctx->snapshot_builder,
                   buf->origptr,
                   xid,
                   parsed->nsubxacts,
                   parsed->subxacts,
                   parsed->xinfo)
```
这一步让 snapshot builder 知道哪些 XID 在这个 commit 处结束。
接着执行 transaction 级 skip 判断：
```text
DecodeTXNNeedSkip(ctx, buf, parsed->dbId, origin_id)
```
skip 条件包括：
```text
SnapBuildXactNeedsSkip(...)
txn_dbid != InvalidOid && txn_dbid != slot database
FilterByOrigin(...)
ctx->fast_forward
```
如果 skip，源码不是调用 `ReorderBufferAbort()`。
而是对子事务和顶层事务执行：
```text
ReorderBufferForget()
```
注释说明原因：某些 commit 即使不输出，也可能需要执行 invalidation，尤其是跨 database 的 shared catalog invalidation。
不过当前 skip startup 场景下可能还没碰 catalog。
不 skip 时，先把 surviving subtransaction 告诉 reorder buffer：
```text
ReorderBufferCommitChild(ctx->reorder, xid, subxid, buf->origptr, buf->endptr)
```
然后普通 commit 调：
```text
ReorderBufferCommit(ctx->reorder,
                    xid,
                    buf->origptr,
                    buf->endptr,
                    commit_time,
                    origin_id,
                    origin_lsn)
```
two-phase commit prepared 则进入 `ReorderBufferFinishPrepared()`。
本节只抓普通路径。
关键点是：
```text
DecodeInsert/Update/Delete/Truncate 只排队；
DecodeCommit 才触发对外回放。
```
commit record 是 logical visibility 的边界。
不是 heap record 的 LSN。
## 15. `DecodeAbort()`：已经排队的 change 如何消失
`DecodeAbort()` 也先处理 origin 和 abort time。
然后用 `DecodeTXNNeedSkip()` 判断是否需要处理。
普通 abort 路径会先 abort 子事务，再 abort 顶层事务：
```text
for each parsed->subxacts:
    ReorderBufferAbort(ctx->reorder, subxid, EndRecPtr, abort_time)
ReorderBufferAbort(ctx->reorder, xid, EndRecPtr, abort_time)
```
`ReorderBufferAbort()` 的语义是清除该事务在 memory 和 disk 上的内容。
如果事务已经 streaming 过，它还会调用 `stream_abort` callback 通知下游丢弃。
如果这个事务加载过 catalog cache，它还可能执行 invalidation，避免后续 decoding 使用被 abort 事务污染的 cache entry。
普通非 streaming、非 prepared abort 不会输出 row change。
这解释了一个经常让人困惑的事实：
```text
abort 事务的 heap WAL record 可以已经被 DecodeInsert/Update/Delete 处理并排队；
直到 abort record 到达，reorder buffer 才知道这些 change 必须丢弃。
```
logical decoding 不是边读边判定事务最终可见。
它只能按 WAL 时间线收集信息，再在事务边界收口。
## 16. `ReorderBufferQueueChange()`：候选 change 进入事务
所有 heap DML change 最终都会调用 `ReorderBufferQueueChange()`。
它先按 XID 找到或创建事务对象：
```text
txn = ReorderBufferTXNByXid(rb, xid, true, NULL, lsn, true)
```
如果事务已被标记 aborted，当前 change 直接释放。
否则，如果 action 是可 streaming 的对外 change，就在顶层事务 flags 上设置：
```text
RBTXN_HAS_STREAMABLE_CHANGE
```
然后把 change 追加到事务自己的 change 链：
```text
change->lsn = lsn
change->txn = txn
dlist_push_tail(&txn->changes, &change->node)
txn->nentries++
txn->nentries_mem++
```
接着更新 memory accounting。
再处理 partial change，例如 toast。
最后检查 memory limit：
```text
ReorderBufferCheckMemoryLimit(rb)
```
所以 `ReorderBufferQueueChange()` 是三个边界的汇合点：
```text
按 XID 聚合
按 LSN 保持事务内部顺序
按内存压力决定是否 spill/stream
```
它仍然不调用 output plugin。
它只把 record 解释结果放进事务容器。
## 17. `ReorderBufferCommit()` 与真正回放
`ReorderBufferCommit()` 很薄。
它用 XID 查找 `ReorderBufferTXN`。
找不到就返回。
找到就调用：
```text
ReorderBufferReplay(txn, rb, xid, commit_lsn, end_lsn, commit_time, origin_id, origin_lsn)
```
`ReorderBufferReplay()` 设置：
```text
txn->final_lsn
txn->end_lsn
txn->commit_time
txn->origin_id
txn->origin_lsn
```
如果事务已经部分 streamed，走 `ReorderBufferStreamCommit()`。
否则如果 `txn->base_snapshot == NULL`，说明没有可解码的数据库变化。
它 cleanup 后返回，不输出空事务。
有 base snapshot 时进入：
```text
ReorderBufferProcessTXN(rb, txn, commit_lsn, snapshot_now, command_id, false)
```
`ReorderBufferProcessTXN()` 先建立 tuple CID hash。
然后调用：
```text
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
这是 output plugin 能用历史 catalog 解释 relation schema 的关键。
接着开启内部事务或内部 subtransaction。
然后对 transaction/subtransaction 的 change 做 merge 迭代。
每个 change 根据 action 进入不同分支。
INSERT/UPDATE/DELETE 分支会先把 `RelFileLocator` 映射成 relation OID：
```text
RelidByRelfilenumber(spcOid, relNumber)
```
再打开 relation：
```text
RelationIdGetRelation(reloid)
```
然后再次检查：
```text
RelationIsLogicallyLogged(relation)
```
如果是 TOAST relation，会走 toast reassembly。
如果是用户表 change，调用：
```text
ReorderBufferApplyChange(rb, txn, relation, change, streaming)
```
这才进入 output plugin wrapper。
TRUNCATE 分支会逐个打开 relation OID，筛掉不 logically logged 的 relation，再调用 `ReorderBufferApplyTruncate()`。
MESSAGE 分支调用 `ReorderBufferApplyMessage()`。
所以 relation 级解释发生在 commit 回放阶段，而不是 `DecodeInsert()` 读 WAL 的阶段。
## 18. output plugin 边界：relation filter 在这里发生
`src/backend/replication/logical/logical.c` 把 reorder buffer callback 包装成 output plugin callback。
`change_cb_wrapper()` 设置：
```text
ctx->accept_writes = true
ctx->write_xid = txn->xid
ctx->write_location = change->lsn
ctx->end_xact = false
```
然后调用：
```text
ctx->callbacks.change_cb(ctx, txn, relation, change)
```
`commit_cb_wrapper()` 类似，只是：
```text
ctx->write_location = txn->end_lsn
ctx->end_xact = true
```
`message_cb_wrapper()` 会允许 `txn == NULL`，因为 non-transactional message 没有事务对象。
内置 logical replication output plugin 在 `src/backend/replication/pgoutput/pgoutput.c`。
`pgoutput_change()` 的第一层 relation 过滤是：
```text
is_publishable_relation(relation)
get_rel_sync_entry(data, relation)
relentry->pubactions.pubinsert/pubupdate/pubdelete
```
DELETE 还要求 oldtuple 存在。
之后如果 publication 设置 `publish_via_partition_root`，可能把 relation 切到 ancestor。
再把 old/new heap tuple 放入 tuple slot。
再执行 row filter：
```text
pgoutput_row_filter(...)
```
row filter 对 UPDATE 可能把 action 转换成 INSERT 或 DELETE。
例如 old tuple 不匹配、新 tuple 匹配时，输出 INSERT。
old tuple 匹配、新 tuple 不匹配时，输出 DELETE。
这说明最终 logical event 类型不一定等于 reorder buffer 的原始 action。
这是 output plugin 的语义层，不是 WAL decode 层。
通过过滤后，`pgoutput_change()` 才发送 BEGIN。
它刻意避免为最终没有任何可发布 change 的事务发送空 BEGIN/COMMIT。
然后调用 `maybe_send_schema()`，再调用 `proto.c` 编码：
```text
logicalrep_write_insert()
logicalrep_write_update()
logicalrep_write_delete()
```
`pgoutput_truncate()` 对每个 relation 做 publishable 和 `pubtruncate` 检查，再统一调用 `logicalrep_write_truncate()`。
`pgoutput_message()` 只有在 `data->messages` 打开时才发送 message。
## 19. database、relation 和 origin filter 的层次
本节需要把三种 filter 分开。
第一层是 database filter。
它尽量早发生。
普通 heap DML 用 block tag 中的 `RelFileLocator.dbOid` 对比：
```text
ctx->slot->data.database
```
truncate 用 `xl_heap_truncate.dbId` 对比。
logical message 用 `xl_logical_message.dbId` 对比。
transaction commit 用 `parsed->dbId` 在 `DecodeTXNNeedSkip()` 中判断。
第二层是 origin filter。
`decode.c` 用 `FilterByOrigin()` 调 output plugin 的 `filter_by_origin_cb`。
`pgoutput` 中对应 `pgoutput_origin_filter()`。
如果 `publish_no_origin` 打开，非本地 origin 的 change 会被过滤。
第三层是 relation/publication filter。
它通常不能在 `DecodeInsert()` 里完成。
原因是 `DecodeInsert()` 手里只有 `RelFileLocator` 和 tuple image。
publication、row filter、column list、partition root 和 generated column 策略需要打开 relation、访问 catalog，并且要在 historic snapshot 下解释。
这个动作发生在 commit 回放时的 `ReorderBufferProcessTXN()` 和 output plugin callback 中。
因此：
```text
database/origin filter:
  尽量早，减少 reorder buffer 压力。
relation/publication filter:
  较晚，需要 historic snapshot 和 output plugin policy。
```
这也解释了为什么只改 publication 可能影响后续输出，但不能改变已经写入 WAL 的物理记录。
WAL record 是事实。
publication 是输出策略。
## 20. unsupported record 与 no-op 边界
logical decoding 对 unsupported record 的处理有三种。
第一种是没有 `rm_decode`。
例如 `RM_DBASE_ID`、`RM_RELMAP_ID`、`RM_BTREE_ID`、`RM_SEQ_ID` 当前没有 logical decode callback。
`LogicalDecodingProcessRecord()` 只会调用 `ReorderBufferProcessXid()`。
这类 record 不输出 logical event。
第二种是有 `rm_decode`，但 record 类型被明确忽略。
`heap_decode()` 忽略：
```text
XLOG_HEAP_INPLACE
XLOG_HEAP_LOCK
```
`heap2_decode()` 忽略 prune、vacuum cleanup、lock updated 和 rewrite decode path。
这些是物理维护、catalog internal 或 row lock 语义，不是 logical DML。
第三种是有 `rm_decode`，但当前状态不允许解码。
例如：
```text
SnapBuildCurrentState < SNAPBUILD_FULL_SNAPSHOT
ctx->fast_forward
SnapBuildXactNeedsSkip
change_useless_for_repack
dbId mismatch
origin filtered
missing tuple image
```
还有一种是真正的 unexpected。
如果 `heap_decode()` 遇到未知 `RM_HEAP_ID` op，会：
```text
elog(ERROR, "unexpected RM_HEAP_ID record type: %u", info)
```
`heap2_decode()`、`logicalmsg_decode()` 也有类似错误。
这类错误通常表示源码版本、WAL magic、rmgr callback 或 record type 解释边界不一致。
不要把它当成普通 publication filter。
## 21. 生命周期、ownership 与 cleanup
heap decode 分配的 `ReorderBufferChange` 属于 reorder buffer。
tuple image 通过 `ReorderBufferAllocTupleBuf()` 分配。
relation OID 数组通过 `ReorderBufferAllocRelids()` 分配。
message prefix/payload 在 `ReorderBufferQueueMessage()` 中复制到 reorder buffer context。
这些对象不是 output plugin 长期拥有。
它们随 `ReorderBufferTXN` cleanup 释放。
正常 commit 路径：
```text
Decode* queues changes
DecodeCommit()
ReorderBufferCommit()
ReorderBufferProcessTXN()
output callbacks
ReorderBufferCleanupTXN()
```
abort 路径：
```text
Decode* may have queued changes
DecodeAbort()
ReorderBufferAbort()
ReorderBufferCleanupTXN()
```
skip committed transaction 路径：
```text
DecodeTXNNeedSkip()
ReorderBufferForget()
possibly execute invalidations
cleanup
```
ERROR 路径由内部 transaction/subtransaction 和 `PG_TRY/PG_CATCH` 保护。
non-transactional message 在 `ReorderBufferQueueMessage()` 中会：
```text
SetupHistoricSnapshot()
PG_TRY()
  rb->message(...)
  TeardownHistoricSnapshot(false)
PG_CATCH()
  TeardownHistoricSnapshot(true)
  PG_RE_THROW()
```
`ReorderBufferProcessTXN()` 也会在内部事务中设置 historic snapshot，并在异常路径中清理 relation、snapshot、resource owner 和 memory context。
这个 ownership 边界让 output plugin 失败不会把 decoder 留在错误 historic snapshot 中。
## 22. 正确性机制层次
logical decoding 的正确性不是一个机制保证的。
第一层是 WAL 写入端。
`XLogLogicalInfoActive()` 和 `RelationIsLogicallyLogged()` 决定 WAL 里是否有足够 tuple 信息。
没有写进去的信息，decode 端不能补。
第二层是 snapshot builder。
`SnapBuildCurrentState()`、`SnapBuildProcessChange()`、`SnapBuildCommitTxn()` 和 `SnapBuildXactNeedsSkip()` 决定当前 record 是否位于可解码历史中。
第三层是 reorder buffer。
它按 XID 聚合、按 top/subxact 关系组织、按 commit/abort 边界输出或丢弃。
第四层是 historic snapshot。
`SetupHistoricSnapshot()` 让 relation lookup、syscache 和 tuple deforming 使用当时的 catalog 语义。
第五层是 invalidation。
commit/abort record 中的 invalidation 保证 decoder 不继续用过期 catalog cache。
第六层是 output plugin。
它根据 relation/publication/row filter/column list/replica identity 决定协议层到底发什么。
这六层不能互相替代。
例如，relation filter 不能弥补 WAL 中缺 old key。
snapshot builder 不能替代 commit record。
ReorderBufferCommit 不能替代 pgoutput 的 row filter。
## 23. 成本、资源与跨模块传播
第一个成本变量是 WAL 字节量。
`wal_level` 或 effective logical 信息打开后，heap insert/update/delete 可能额外携带 full tuple、old tuple 或 old key。
UPDATE 在 logical 模式下还会避免某些 prefix/suffix 省略，因为 decoder 需要完整 new tuple。
第二个成本变量是 reorder buffer 内存。
它随未提交事务的 change 数、tuple 宽度、toast chunks、subxact 数和 invalidation 数增长。
超过 `logical_decoding_work_mem` 后会 spill 或 stream。
第三个成本变量是 catalog lookup。
commit 回放时，每个 relation change 需要把 `RelFileLocator` 映射到 OID，再打开 relation，在 historic snapshot 下解释 tuple descriptor。
schema churn、partition 数、publication 数、row filter 表达式都会放大这部分成本。
第四个成本变量是 filter 发生时机。
database/origin filter 很早，能减少排队。
relation/publication filter 较晚，可能意味着一个最终不发布的 relation change 已经占用过 reorder buffer 内存。
第五个成本变量是 output protocol。
`logicalrep_write_tuple()` 需要按 column list、generated column 策略、binary/text 选项序列化 tuple。
宽表、toast、更新大字段和 row filter 都可能让 CPU 成为瓶颈。
跨模块传播路径可以压缩成：
```text
wal_level / logical slot
  -> heap WAL 字节量
  -> reorder buffer memory/spill
  -> slot restart_lsn retention
  -> pg_wal 保留
  -> downstream apply lag
  -> vacuum/catalog_xmin pressure
```
因此 logical decoding 的性能问题很少是单点。
它常常在 WAL 写入、slot retention、reorder buffer、catalog lookup 和 output plugin 之间迁移。
## 24. 可观测现象：看到什么，解释什么
最小实验可以用 `test_decoding` 或 `pgoutput`。
先创建 logical slot。
再执行：
```sql
CREATE TABLE t (id int primary key, v text);
INSERT INTO t VALUES (1, 'a');
UPDATE t SET v = 'b' WHERE id = 1;
DELETE FROM t WHERE id = 1;
SELECT pg_logical_emit_message(true, 'demo', 'transactional');
COMMIT;
```
如果用 SQL decoding 函数：
```sql
SELECT * FROM pg_logical_slot_get_changes('slot_name', NULL, NULL);
```
看到的是事务级输出，而不是每条 heap WAL record 的即时输出。
用 `pg_waldump` 对同一段 LSN 看，会看到 heap、xact、logical message 等 rmgr record。
要把两边对应起来，按这个顺序问：
```text
record 的 rmgr 有没有 rm_decode？
heap record flag 是否包含 tuple data？
record database 是否等于 slot database？
origin 是否被过滤？
transaction 是否 commit？
relation 是否 publishable？
publication action 是否允许？
row filter 是否改变 action？
```
`pg_stat_replication_slots` 可以看到 spill、stream、total bytes 等累计信息。
`pg_replication_slots` 可以看 `restart_lsn`、`confirmed_flush_lsn`、`catalog_xmin`。
`pg_stat_wal` 可以看实例级 WAL 量变化。
这些指标都不是完整因果。
它们只能分别观察 slot retention、WAL 字节量和 decoding output 进度。
源码断点更直接。
建议断点：
```text
LogicalDecodingProcessRecord
heap_decode
DecodeInsert
DecodeUpdate
DecodeDelete
DecodeTruncate
logicalmsg_decode
xact_decode
DecodeCommit
DecodeAbort
ReorderBufferQueueChange
ReorderBufferCommit
ReorderBufferAbort
ReorderBufferProcessTXN
pgoutput_change
pgoutput_truncate
pgoutput_message
```
断点观察的核心字段：
```text
reader->ReadRecPtr / EndRecPtr
XLogRecGetRmid()
XLogRecGetInfo()
XLogRecGetXid()
XLogRecGetTopXid()
XLogRecGetOrigin()
change->action
change->data.tp.rlocator
txn->xid
txn->base_snapshot
relentry->pubactions
```
如果最终没有输出，先不要猜 subscriber。
先从这些边界逐层排除。
## 25. 常见误区
误区一：把 heap WAL record 当成 logical event。
heap record 只是候选输入。
commit、snapshot、database、origin、relation 和 publication 都可能让它不输出。
误区二：认为 `DecodeInsert()` 会调用 output plugin。
它只创建 `ReorderBufferChange` 并排队。
output plugin 在 `ReorderBufferProcessTXN()` 回放时才被调用。
误区三：把 database filter 和 relation filter 混在一起。
database filter 在 `decode.c` 尽量早做。
relation/publication filter 在 historic snapshot 下做。
误区四：认为 DELETE 没输出就是 decoder 没读到 WAL。
也可能是 heap 写入端没有 old tuple/key，或 pgoutput 因 replica identity 不足而返回。
误区五：认为 `RM_DBASE_ID` record 会变成 database-level logical event。
当前 rmgrlist 中 `RM_DBASE_ID` 没有 `rm_decode`。
database record 影响 storage/database lifecycle，不直接输出 DML。
误区六：认为 logical message 和 heap DML 生命周期一样。
transactional message 随事务 commit/abort。
non-transactional message 读到 WAL 时即可输出。
## 26. 课堂实验
实验一：从 heap WAL 到 change。
步骤：
```text
1. 建 logical slot，打开 test_decoding。
2. 对普通表执行 INSERT/UPDATE/DELETE/TRUNCATE。
3. 在 DecodeInsert/DecodeUpdate/DecodeDelete/DecodeTruncate 断点。
4. 记录 xl_heap_* flags、RelFileLocator.dbOid、change->action。
5. 在 ReorderBufferCommit 断点确认 commit record 才触发输出。
```
要画出的状态：
```text
WAL record LSN
XID
change action
txn->changes
commit_lsn
最终 output
```
实验二：replica identity 边界。
步骤：
```text
1. 创建没有 primary key 的表。
2. 建 publication 发布 DELETE。
3. DELETE 一行并观察 pgoutput 是否发送。
4. 改成 REPLICA IDENTITY FULL。
5. 再 DELETE 一行。
6. 对比 DecodeDelete 中 oldtuple 是否存在，以及 pgoutput_change 是否返回。
```
要解释的源码：
```text
heap_delete() -> ExtractReplicaIdentity()
DecodeDelete() -> XLH_DELETE_CONTAINS_OLD
pgoutput_change() -> missing oldtuple returns
```
实验三：database filter。
步骤：
```text
1. 在 database A 创建 logical slot。
2. 在 database B 做 INSERT。
3. 用 pg_waldump 找到 heap record。
4. 在 DecodeInsert 观察 target_locator.dbOid。
5. 确认该 record 不进入当前 slot 的 reorder buffer。
```
这个实验能把“WAL 是实例级”与“logical slot 是 database-specific”分开。
## 27. 讨论题
1. 为什么 `DecodeInsert()` 不能直接调用 `logicalrep_write_insert()`？
2. 为什么 `heap_decode()` 忽略 `XLOG_HEAP_LOCK` 是合理的？
3. `RM_DBASE_ID` 没有 `rm_decode` 会不会导致 logical decoding 漏掉用户表 DML？
4. 为什么 commit skip 路径使用 `ReorderBufferForget()`，而不是简单 `ReorderBufferAbort()`？
5. UPDATE 的 row filter 为什么可能把最终输出 action 转换成 INSERT 或 DELETE？
6. 如果 `pg_stat_replication_slots` 显示 spill 增加，应该从哪些源码边界判断是未提交事务大、relation filter 太晚、还是 output plugin 慢？
7. 为什么 non-transactional logical message 在 fast-forward 中也要设置 `processing_required`？
8. 如果 `pg_waldump` 看到 `XLOG_HEAP2_MULTI_INSERT`，为什么下游可能看到多条 INSERT？
## 28. 本节小结
本节的核心链路是：
```text
WAL 写入端先保留 logical 所需信息
  -> LogicalDecodingProcessRecord 按 rmgr 分派
  -> heap/logicalmsg record 变成 ReorderBufferChange 或 message
  -> xact commit/abort record 决定回放、丢弃或 cleanup
  -> ReorderBufferProcessTXN 在 historic snapshot 下打开 relation
  -> output plugin 做 relation/publication/row filter
  -> proto.c 编码最终协议事件
```
核心状态是 `LogicalDecodingContext`、`SnapBuild`、`ReorderBufferTXN` 和 `ReorderBufferChange`。
`ReorderBufferChange` 不是最终事件。
它只是“某个 XID 下可能可见的候选变化”。
ownership 由 reorder buffer 管理。
commit 后输出并 cleanup。
abort 后丢弃并 cleanup。
skip committed transaction 时要注意 invalidation。
database filter 很早。
relation/publication filter 较晚。
unsupported record 不是都报错。
有些 rmgr 没有 decode callback，有些 record 被明确忽略，有些因为 snapshot/fast-forward/origin/database 被跳过。
诊断时不要从最终输出反推单一原因。
要按固定链路逐层定位：
```text
WAL 是否携带 tuple/key
rmgr 是否有 decode callback
snapshot 是否 ready
database/origin 是否匹配
transaction 是否 commit
relation 是否可 logically logged
publication action 和 row filter 是否允许
output plugin 是否发送
```
可迁移的系统规律是：
```text
物理日志到逻辑事件之间通常需要一个延迟解释层。
这个层必须同时持有历史状态、事务边界、对象身份映射和输出策略。
```
在 PostgreSQL 中，这个延迟解释层就是 `decode.c` + `reorderbuffer.c` + output plugin callback 的组合。
任何只看其中一层的诊断，都会把“没有输出”误判成“没有 WAL”或“decoder 丢数据”。
