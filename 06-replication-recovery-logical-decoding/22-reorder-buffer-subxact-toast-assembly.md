# PostgreSQL ReorderBuffer subxact / TOAST / tuple image 重组
## 课程定位
前置知识：已经理解 logical slot、WAL decoding、SnapBuild 和 output plugin 的基本分工，
也知道 heap tuple 的 UPDATE / DELETE 需要 replica identity 才能在下游定位旧行。
本节唯一主问题：
```text
logical decoding 如何处理 subtransaction commit/abort、TOAST chunk、old key 和 tuple image，
为什么输出插件看到的必须是完整且语义稳定的行级变更？
```
核心矛盾：
```text
WAL 为 crash recovery 服务，天然记录按 LSN 到达的物理片段；
output plugin 为外部系统服务，必须看到按事务语义成立的完整行级 change。
```
这节课只围绕一个对象推进：
```text
一条 heap WAL fragment
  -> 被 decode 成 ReorderBufferChange
  -> 等待 subxact fate、TOAST chunks、tuplecid/snapshot 边界稳定
  -> 在 top transaction commit 或合法 streaming 边界上交给 output plugin
  -> commit/abort/error 后清理所有中间状态
```
学完后应能判断：
```text
为什么不能看到 WAL record 就立刻调用插件；
为什么 ReorderBufferCommitChild() 是 commit 前语义拼接点；
为什么 TOAST chunk 不是插件应该看到的用户行；
为什么 oldtuple 不等于旧整行；
为什么 tuplecid/snapshot 是插件看不到但语义必须依赖的状态；
为什么 abort cleanup 要清理 change、toast、snapshot、tuplecid 和 spill。
```
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面的 replication 课程已经建立了两条边界。
physical replication 只关心 WAL bytes 是否能按 LSN 送到下游。
logical decoding 则要把 WAL bytes 解释成外部可消费的变更流。
这两个目标差异很大。
heap WAL 里的自然单位可能是：
```text
toast relation insert chunk
heap update new tuple
heap update old key
heap delete old key
heap2 new cid
xact assignment
xact commit / abort
```
output plugin 需要的自然单位却是：
```text
BEGIN
  relation R changed one row:
    old key or old tuple if needed
    new tuple if needed
COMMIT
```
于是中间必须有一个 reorder / reconstruction 层。
这个层就是 `src/backend/replication/logical/reorderbuffer.c`。
它不是简单缓存。
它同时维护：
```text
transaction fate
subtransaction membership
LSN ordering
TOAST reconstruction
replica identity tuple images
historic snapshot / command id
tuplecid mapping
memory accounting / spill
abort cleanup
```
一个典型现场是：
```sql
BEGIN;
SAVEPOINT s;
UPDATE t SET big_text = repeat('x', 100000) WHERE id = 1;
ROLLBACK TO s;
UPDATE t SET big_text = repeat('y', 100000) WHERE id = 1;
COMMIT;
```
WAL 可能先出现第一组 TOAST chunks 和 main heap update，
然后才出现 savepoint rollback 对应的 abort 信息，
再出现第二组 TOAST chunks 和最终 commit。
插件不能看到第一组被回滚的行。
插件也不应该看到 toast table 的 chunk rows。
插件更不能被迫自己用 catalog snapshot 和 combo CID 去解释 tuple descriptor。
这节课回答的就是：
```text
PostgreSQL 如何把这些物理碎片压缩成插件能安全消费的逻辑行事件？
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
decode.c 按 WAL record 解析出 ReorderBufferChange；
ReorderBuffer 按 xid 暂存 change，按 top xid 归并 surviving subxacts，
按 toast hash 暂存 chunk，按 tuplecid/snapshot 维护 catalog visibility；
commit record 到来后，以 LSN k-way merge replay top transaction 与 subtransactions，
在插件回调前重组 TOAST 和 old/new tuple image；
abort、skip、concurrent abort 或 ERROR 路径则清理所有未完成语义状态。
```
这里的 tension 可以拆成三条。
第一，WAL 顺序不能破坏。
每个 xid 的 change list 是按 WAL 到达顺序追加的。
commit replay 时 top transaction 和多个 subtransaction 不能简单拼接。
`ReorderBufferProcessTXN()` 要用 iterator 做 k-way merge，
保证插件看到的 change 顺序仍然是全局 LSN 顺序。
第二，事务命运不能提前假设。
一个 `subxid` 可能先作为普通 xid 出现在 heap WAL 中。
后续 record 才告诉 decoder 它属于哪个 top transaction，
而 commit/abort record 才告诉 decoder 它是否 surviving。
第三，row image 不能暴露半成品。
TOAST chunks、old key、new tuple、catalog tuplecid 都在不同 record 中出现。
插件接口不能要求每个插件重新拼这些物理碎片。
所以本节的核心不变量是：
```text
output plugin 看到的是 committed 或合法 streamed 语义下的 row-level change，
不是 WAL parser 当前刚看到的物理 record。
```
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/reorderbuffer.h` | `ReorderBufferChange`、`ReorderBufferTXN`、`oldtuple`、`newtuple`、`tuplecids`、`toast_hash`。 |
| 2 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferQueueChange()`、`ReorderBufferAssignChild()`、`ReorderBufferCommitChild()`、`ReorderBufferAbort()`、`ReorderBufferProcessTXN()`、`ReorderBufferToast*()`。 |
| 3 | `src/backend/replication/logical/decode.c` | `LogicalDecodingProcessRecord()`、`DecodeCommit()`、`DecodeAbort()`、`DecodeInsert()`、`DecodeUpdate()`、`DecodeDelete()`、`DecodeMultiInsert()`。 |
| 4 | `src/backend/replication/logical/snapbuild.c` | `SnapBuildProcessChange()`、`SnapBuildProcessNewCid()`，base snapshot 和 tuplecid mapping 的来源。 |
| 5 | `src/backend/access/heap/heapam.c` | heap insert/update/delete 如何决定 WAL 中包含 new tuple、old key、old tuple、NEW_CID 和 TOAST。 |
| 6 | `src/include/access/heapam_xlog.h` | `XLH_UPDATE_CONTAINS_OLD_KEY`、`XLH_DELETE_CONTAINS_OLD_TUPLE` 等 flags。 |
| 7 | `src/backend/access/common/toast_internals.c` | `toast_save_datum()` 如何把 varlena datum 切成 toast table rows。 |
| 8 | `src/backend/replication/pgoutput/pgoutput.c` | `pgoutput_change()` 如何消费 `oldtuple` / `newtuple`、row filter 和 publication 规则。 |
| 9 | `src/backend/replication/logical/proto.c` | `logicalrep_write_update()` / `logicalrep_write_delete()` 如何把 old key/full old tuple 编成 `K` / `O`。 |
推荐源码路线：
```text
heapam.c 写 WAL flags
  -> decode.c 分配 ReorderBufferChange
  -> reorderbuffer.c 按 xid 入队
  -> xact commit/abort 触发 subxact 归并或清理
  -> ReorderBufferProcessTXN() 按 LSN replay
  -> ReorderBufferToastReplace() 修正 tuple image
  -> pgoutput/proto 消费 old/new tuple
```
不要从单个函数顶部顺读到底。
这节课看的是状态转移：
```text
fragment -> pending semantic state -> complete row change -> callback -> cleanup
```
## 4. 关键数据结构与状态边界
`ReorderBufferChange` 是 row change 的载体，
也承载内部 snapshot、command id、tuplecid 和 invalidation。
本节只展开和行级变更相关的字段。
| 字段 | 语义 |
| --- | --- |
| `lsn` | change 的 WAL 顺序锚点。 |
| `action` | INSERT / UPDATE / DELETE 或 internal snapshot/cid/tuplecid。 |
| `txn` | change 所属 `ReorderBufferTXN`，可能是 subtransaction。 |
| `data.tp.rlocator` | WAL 中的 relation locator，replay 时再映射为 relation OID。 |
| `data.tp.oldtuple` | UPDATE / DELETE 的 old image；可能是 full old tuple，也可能只是 replica identity key。 |
| `data.tp.newtuple` | INSERT / UPDATE 的 new tuple image。 |
| `data.tp.clear_toast_afterwards` | 当前主表 tuple 处理完后，是否可以清掉 pending toast chunks。 |
`oldtuple` 不是“旧整行”的同义词。
`DecodeUpdate()` 和 `DecodeDelete()` 看到任何 old image 都放到这个字段。
它到底是 full old tuple 还是 old key，
要结合 WAL flags 和 relation 的 replica identity 判断。
`ReorderBufferTXN` 维护每个 xid 的解码状态。
对本节最重要的字段是：
| 字段 | 语义 |
| --- | --- |
| `xid` | 当前 transaction 或 subtransaction 的 XID。 |
| `toplevel_xid` / `toptxn` | 如果已知是 subxact，指向 top transaction。 |
| `txn_flags` | `RBTXN_IS_SUBXACT`、`RBTXN_HAS_PARTIAL_CHANGE`、`RBTXN_IS_STREAMED`、`RBTXN_IS_ABORTED` 等状态。 |
| `changes` | 当前 xid 自己的 change list，按 WAL 到达顺序追加。 |
| `subtxns` | top transaction 下 surviving subtransactions 的非层级列表。 |
| `base_snapshot` / `base_snapshot_lsn` | 解码 transaction 时最早需要的 historic snapshot。 |
| `snapshot_now` / `command_id` | streaming run 之间保存的 snapshot/CID 边界。 |
| `tuplecids` / `tuplecid_hash` | catalog tuple 的 `(relfilelocator, ctid) -> (cmin, cmax, combocid)` mapping。 |
| `toast_hash` | 当前 transaction 中等待被主表 tuple 消费的 TOAST chunks。 |
| `nentries` / `nentries_mem` / `size` | memory accounting、streaming 和 spill 的输入。 |
subtransaction 状态有一个重要取舍。
`ReorderBuffer` 不重建完整 SQL savepoint 树。
它把未 abort 的 subtransactions 挂到 top transaction 的 `subtxns` list。
这足以回答 logical decoding 的问题：
```text
这个 change 是否属于最终提交的 top transaction？
如果属于，它在全事务 LSN 顺序中的位置在哪里？
```
插件不需要知道 savepoint 嵌套层级。
TOAST 重组状态是 `ReorderBufferToastEnt`。
它以 `chunk_id` 为 key，
保存：
```text
last_chunk_seq
num_chunks
size
chunks
reconstructed
```
这不是 shared memory。
也不是 plugin API。
它只存在于这个窗口：
```text
toast relation chunk 已入队
  -> 主表 INSERT/UPDATE 尚未 replay
  -> ReorderBufferToastReplace() 找到对应 external datum
  -> 插件回调消费主表 change
  -> ReorderBufferToastReset() 释放 chunks 和 reconstructed datum
```
tuplecid 也是内部状态。
它服务 historic catalog visibility。
`snapbuild.c` 注释说明，
普通 WAL tuple 不携带可直接解释的 cmin/cmax，
combo CID 又只在原 backend 内存中有意义。
所以 heapam 在 logical decoding 需要时写 `XLOG_HEAP2_NEW_CID`，
SnapBuild 再把 mapping 放进 reorder buffer。
结论是：
```text
raw field 不是语义；
field + txn fate + snapshot/CID + relation metadata + cleanup state 才是语义。
```
## 5. heapam.c 先决定 WAL 中有哪些 tuple image
先看生成侧，
因为 `ReorderBuffer` 不能凭空发明 tuple image。
`heap_insert()` 在 `RelationIsLogicallyLogged(relation)` 且没有 `HEAP_INSERT_NO_LOGICAL` 时，
设置：
```text
XLH_INSERT_CONTAINS_NEW_TUPLE
REGBUF_KEEP_DATA
```
这保证 logical decoding 即使遇到 full-page image，
也能从 WAL record 中拿到 tuple bytes。
如果 relation 是 toast relation，
还设置：
```text
XLH_INSERT_ON_TOAST_RELATION
```
这个 flag 让 decode 层知道：
```text
这条 heap insert 是主表 row image 的组成部分，
不是应该直接发布的用户行。
```
`toast_save_datum()` 负责把一个 varlena value 切成多行 toast tuples。
每个 toast tuple 的前三列语义是：
```text
chunk_id    = toast_pointer.va_valueid
chunk_seq   = 0, 1, 2, ...
chunk_data  = 本 chunk 的 bytes
```
它调用 `heap_insert(toastrel, toasttup, mycid, options, NULL)`。
所以 WAL 层看到的仍然是 heap insert。
`heap_update()` 会先确定哪些列被修改，
并识别 replica identity 列是否有 external TOAST。
关键逻辑是：
```text
modified_attrs = HeapDetermineColumnsInfo(..., id_attrs, ..., &id_has_external)
old_key_tuple = ExtractReplicaIdentity(relation, &oldtup,
                                       bms_overlap(modified_attrs, id_attrs) ||
                                       id_has_external,
                                       &old_key_copied)
```
这个 `ExtractReplicaIdentity()` 在 critical section 前执行，
因为它可能分配内存或 flatten TOAST，
critical section 里不能冒着 ERROR 风险做这些事。
UPDATE WAL flags 包括：
```text
XLH_UPDATE_CONTAINS_NEW_TUPLE
XLH_UPDATE_CONTAINS_OLD_TUPLE   -- REPLICA IDENTITY FULL
XLH_UPDATE_CONTAINS_OLD_KEY     -- DEFAULT / INDEX identity key
```
`heap_delete()` 对 old image 的逻辑类似，
但 delete 必须尝试拿旧行定位信息：
```text
ExtractReplicaIdentity(relation, &tp, true, &old_key_copied)
```
如果 relation 是 `REPLICA_IDENTITY_NOTHING`，
或没有可用 replica identity key，
`ExtractReplicaIdentity()` 返回 `NULL`。
这意味着后续 pgoutput 可能无法发布 DELETE。
这不是 `decode.c` 或 `ReorderBuffer` 漏数据。
这是 heap WAL 生成侧没有提供可发布的旧行定位 image。
`ExtractReplicaIdentity()` 的边界很重要：
```text
REPLICA_IDENTITY_NOTHING:
  返回 NULL。
REPLICA_IDENTITY_FULL:
  返回整条旧 tuple；
  如果有 external TOAST，则 toast_flatten_tuple() 内联。
DEFAULT / INDEX:
  只在 key_required 时构造 identity columns tuple；
  非 identity columns 置 NULL；
  如果 key tuple 仍有 external TOAST，也 flatten。
```
所以 old key / old tuple 的语义从 heap WAL 生成时就已经定了。
`ReorderBuffer` 的职责是保留、排序、重组和过滤。
它不负责凭空补出没有写入 WAL 的 old image。
## 6. decode.c：解析 WAL fragment，但不调用插件
`LogicalDecodingProcessRecord()` 是解码入口之一。
它先取：
```text
txid = XLogRecGetTopXid(record)
```
如果 top xid 有效，
就调用：
```text
ReorderBufferAssignChild(ctx->reorder,
                         txid,
                         XLogRecGetXid(record),
                         buf.origptr)
```
这解释了 `XLOG_XACT_ASSIGNMENT` 的边界。
`xact_decode()` 遇到 `XLOG_XACT_ASSIGNMENT` 时基本不做事，
注释说 subxact assignment 已经在处理每条 record 时完成。
也就是说，
只要 WAL record 带 top xid，
decoder 可以在 record 处理时把 subxid 归到 top xid。
heap decode 的入口包括：
```text
DecodeInsert()
DecodeUpdate()
DecodeDelete()
DecodeMultiInsert()
```
它们共同做这些事：
```text
检查 slot database
检查 origin filter
分配 ReorderBufferChange
按 WAL flags 解码 oldtuple / newtuple
记录 rlocator
设置 clear_toast_afterwards
调用 ReorderBufferQueueChange()
```
`DecodeInsert()` 如果没有 `XLH_INSERT_CONTAINS_NEW_TUPLE`，
直接返回。
普通 insert 设置：
```text
action = REORDER_BUFFER_CHANGE_INSERT
newtuple = DecodeXLogTuple(...)
clear_toast_afterwards = true
```
然后把：
```text
xlrec->flags & XLH_INSERT_ON_TOAST_RELATION
```
作为 `toast_insert` 参数传给 `ReorderBufferQueueChange()`。
`DecodeUpdate()` 如果看到 `XLH_UPDATE_CONTAINS_NEW_TUPLE`，
从 block data 解码 `newtuple`。
如果看到 `XLH_UPDATE_CONTAINS_OLD`，
从 record main data 尾部解码 `oldtuple`。
这里 `XLH_UPDATE_CONTAINS_OLD` 是：
```text
XLH_UPDATE_CONTAINS_OLD_TUPLE | XLH_UPDATE_CONTAINS_OLD_KEY
```
所以 decode 层不把 old key 和 old full tuple 放进不同字段。
它们都进入：
```text
change->data.tp.oldtuple
```
`DecodeDelete()` 同样用 `XLH_DELETE_CONTAINS_OLD` 判断是否有 old image。
如果没有，
`change->data.tp.oldtuple` 就是 NULL。
`DecodeMultiInsert()` 还有一个 TOAST 边界。
只有最后一个 multi-insert record 的最后一个 tuple，
在 `XLH_INSERT_LAST_IN_MULTI` 成立时，
才设置：
```text
clear_toast_afterwards = true
```
其它 tuple 先设为 false。
这避免同一次 `heap_multi_insert()` 的 toast reconstruction 被过早清理。
这一步结束后，
change 只是进入 reorder buffer。
插件还没有被调用。
## 7. 入队与 subxact assignment
`ReorderBufferQueueChange()` 的职责是把 change 挂到某个 `ReorderBufferTXN`。
主路径是：
```text
txn = ReorderBufferTXNByXid(rb, xid, create=true, ...)
if rbtxn_is_aborted(txn):
  ReorderBufferFreeChange(...)
  return
change->lsn = lsn
change->txn = txn
dlist_push_tail(&txn->changes, &change->node)
txn->nentries++
txn->nentries_mem++
ReorderBufferChangeMemoryUpdate(...)
ReorderBufferProcessPartialChange(...)
ReorderBufferCheckMemoryLimit(...)
```
如果 transaction 已知 aborted，
后续 WAL fragment 直接丢弃。
这避免 savepoint rollback 或 streaming abort 后继续积累无效 row changes。
`ReorderBufferAssignChild()` 处理的是“晚知道父子关系”的问题。
源码路径是：
```text
txn = ReorderBufferTXNByXid(rb, xid, create=true)
subtxn = ReorderBufferTXNByXid(rb, subxid, create=true)
if subtxn 已经存在但还不是 known subxact:
  dlist_delete(&subtxn->node)   -- 从 top-level txns list 摘掉
subtxn->txn_flags |= RBTXN_IS_SUBXACT
subtxn->toplevel_xid = xid
subtxn->toptxn = txn
dlist_push_tail(&txn->subtxns, &subtxn->node)
txn->nsubtxns++
ReorderBufferTransferSnapToParent(txn, subtxn)
```
`dlist_delete(&subtxn->node)` 是关键语义动作。
一个先被当作 top-level transaction 收集的 xid，
一旦确认是 subxact，
就必须从 top-level transaction list 中移走。
否则后续可能被当作独立事务 replay 或 cleanup。
`ReorderBufferTransferSnapToParent()` 是另一个关键动作。
如果 subtransaction 的第一条 logical change 早于 top transaction，
subtxn 可能已经拿到了更早的 `base_snapshot`。
知道父子关系后，
top transaction 必须继承更早 snapshot。
规则是：
```text
top 没有 base_snapshot:
  转移 subtxn snapshot。
subtxn base_snapshot_lsn 更早:
  释放 top 原 snapshot；
  转移 subtxn snapshot；
  调整 txns_by_base_snapshot_lsn 位置。
top snapshot 已足够早:
  subtxn snapshot dec refcount。
```
所以 subxact assignment 不只是父指针。
它会影响：
```text
historic snapshot
restart_decoding_lsn
txns_by_base_snapshot_lsn 顺序
后续 catalog visibility
```
这也是为什么 raw `ReorderBufferTXN` 不能单独解释语义。
只有：
```text
txn_flags + toplevel_xid + toptxn + list membership + snapshot state
```
组合起来，
才知道它是 top transaction 还是 subtransaction。
## 8. commit / abort record 决定最终命运
`DecodeCommit()` 解析 commit record 后先调用：
```text
SnapBuildCommitTxn(ctx->snapshot_builder, buf->origptr, xid,
                   parsed->nsubxacts, parsed->subxacts,
                   parsed->xinfo)
```
如果 transaction 需要跳过，
它对所有 subxacts 和 top xid 调 `ReorderBufferForget()`。
如果需要解码，
它先对每个 surviving subxact 调：
```text
ReorderBufferCommitChild(ctx->reorder, xid, parsed->subxacts[i],
                         buf->origptr, buf->endptr)
```
然后才调用：
```text
ReorderBufferCommit(ctx->reorder, xid, buf->origptr, buf->endptr,
                    commit_time, origin_id, origin_lsn)
```
`ReorderBufferCommitChild()` 做的事情很少，
但语义很重：
```text
subtxn = ReorderBufferTXNByXid(rb, subxid, create=false)
if subtxn == NULL:
  return
subtxn->final_lsn = commit_lsn
subtxn->end_lsn = end_lsn
ReorderBufferAssignChild(rb, xid, subxid, InvalidXLogRecPtr)
```
如果 subtransaction 没有 logical changes，
没有必要创建状态。
如果它有 changes，
commit child 会确保它挂到 top transaction，
并记录 final/end LSN。
注意 commit record 中的 `parsed->subxacts` 是最终提交树里的 subtransactions。
已经 rollback 的 savepoint 不会成为 surviving subxact stream。
`DecodeAbort()` 的路径相反。
它对每个 subxact 调：
```text
ReorderBufferAbort(ctx->reorder, parsed->subxacts[i],
                   buf->record->EndRecPtr, abort_time)
```
然后对 top xid 调：
```text
ReorderBufferAbort(ctx->reorder, xid,
                   buf->record->EndRecPtr, abort_time)
```
`ReorderBufferAbort()` 的主逻辑是：
```text
txn = ReorderBufferTXNByXid(...)
if txn == NULL:
  return
txn->abort_time = abort_time
if rbtxn_is_streamed(txn):
  rb->stream_abort(rb, txn, lsn)
  if txn->ninvalidations > 0:
    ReorderBufferImmediateInvalidation(...)
txn->final_lsn = lsn
ReorderBufferCleanupTXN(rb, txn)
```
如果 transaction 已经 streaming 到下游，
必须发 stream abort，
让下游丢弃先前收到的 in-progress changes。
如果没有 streaming，
abort 的效果就是本地 cleanup，
插件不会看到这些 row changes。
所以 abort 不是“不输出”这么简单。
它必须清掉所有会污染后续解码的中间状态。
包括：
```text
change list
tuplecid list/hash
base snapshot refcount
snapshot_now
toast_hash
spill files
cache invalidation side effects
by_txn hash entry
```
## 9. commit replay：LSN merge 后才调用插件
`ReorderBufferCommit()` 找到 top transaction 后进入 `ReorderBufferReplay()`。
如果 transaction 没有 `base_snapshot`，
说明没有需要解码的 database changes，
cleanup 后返回。
否则：
```text
snapshot_now = txn->base_snapshot
ReorderBufferProcessTXN(rb, txn, commit_lsn, snapshot_now,
                        FirstCommandId, false)
```
`ReorderBufferProcessTXN()` 是插件回调前的主流程。
它先构造 catalog tuplecid hash：
```text
ReorderBufferBuildTupleCidHash(rb, txn)
```
再建立 historic snapshot：
```text
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
然后开启内部 transaction 或 subtransaction。
原因是 decode 期间会访问 syscache、relcache、lock 和 ResourceOwner。
但这些访问不能留下持久写入。
所以 replay 正常结束后，
源码会：
```text
TeardownHistoricSnapshot(false)
AbortCurrentTransaction()
执行 invalidations
ReorderBufferCleanupTXN(...)
```
真正遍历 change 的部分是：
```text
ReorderBufferIterTXNInit(rb, txn, &iterstate)
while ((change = ReorderBufferIterTXNNext(rb, iterstate)) != NULL)
```
iterator 对 top transaction 和 surviving subtransactions 做 k-way merge。
这保证：
```text
每个 transaction 内部顺序不变；
多个 subtransaction 合并后仍按 LSN 不降序；
插件看到的是一个 top transaction 的全局 logical order。
```
对 INSERT / UPDATE / DELETE，
流程是：
```text
RelidByRelfilenumber(...)
RelationIdGetRelation(...)
RelationIsLogicallyLogged(...)
if !IsToastRelation(relation):
  ReorderBufferToastReplace(rb, txn, relation, change)
  ReorderBufferApplyChange(rb, txn, relation, change, streaming)
  if clear_toast_afterwards:
    ReorderBufferToastReset(rb, txn)
else if action == INSERT:
  dlist_delete(&change->node)
  ReorderBufferToastAppendChunk(rb, txn, relation, change)
```
`ReorderBufferApplyChange()` 才是 plugin 边界：
```text
streaming ? rb->stream_change(...) : rb->apply_change(...)
```
在这之前，
已经完成：
```text
subxact fate 过滤
LSN merge
relation lookup
historic snapshot setup
tuplecid hash setup
TOAST replacement
old/new tuple image 解码
```
所以插件收到的 `Relation` 和 `ReorderBufferChange` 不是裸 WAL parse 结果。
它们是完整语义上下文中的 row-level change。
internal snapshot 和 command id change 也在这个 loop 里推进。
遇到 `REORDER_BUFFER_CHANGE_INTERNAL_SNAPSHOT`：
```text
TeardownHistoricSnapshot(false)
snapshot_now = 新 snapshot 或 copy
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
遇到 `REORDER_BUFFER_CHANGE_INTERNAL_COMMAND_ID`：
```text
command_id = change->data.command_id
必要时 ReorderBufferCopySnap(...)
snapshot_now->curcid = command_id
TeardownHistoricSnapshot(false)
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
如果 internal tuplecid 出现在普通 change queue，
源码直接：
```text
elog(ERROR, "tuplecid value in changequeue")
```
这说明 tuplecid 是 replay 前构造 historic visibility 的输入，
不是插件应该看到的 change。
## 10. TOAST assembly：从 chunk rows 到稳定 tuple
TOAST 是最直观的“物理碎片变逻辑对象”。
`ReorderBufferToastAppendChunk()` 只处理 toast relation INSERT。
它从 toast tuple 中读：
```text
chunk_id  = fastgetattr(newtup, 1, ...)
chunk_seq = fastgetattr(newtup, 2, ...)
chunk     = fastgetattr(newtup, 3, ...)
```
如果 `txn->toast_hash` 还没有初始化，
先调用 `ReorderBufferToastInitHash()`。
hash key 是 `chunk_id`。
新 entry 必须从 `chunk_seq = 0` 开始。
已有 entry 的下一块必须满足：
```text
chunk_seq == ent->last_chunk_seq + 1
```
否则 ERROR。
这不是普通运行时 fallback。
它说明 toast chunk stream 不满足 logical decoding 重组前提。
主表 INSERT/UPDATE 到来时，
`ReorderBufferToastReplace()` 运行。
如果没有 `toast_hash`，
直接返回。
如果有，
它 deform `change->data.tp.newtuple`，
遍历每个 varlena attribute。
对 external datum：
```text
VARATT_EXTERNAL_GET_POINTER(toast_pointer, varlena_pointer)
ent = hash_search(txn->toast_hash, &toast_pointer.va_valueid, HASH_FIND)
```
找不到 entry 时，
说明该 external toasted value 在本次 change 中没改变。
源码注释说这些 unchanged toast tuples 仍然指向 on-disk toast data。
找得到 entry 时，
`ReorderBufferToastReplace()` 分配 `reconstructed`，
按 chunk list memcpy：
```text
memcpy(VARDATA(reconstructed) + data_done,
       VARDATA(chunk),
       VARSIZE(chunk) - VARHDRSZ)
```
然后按 external pointer 的 metadata 设置 varlena header：
```text
SET_VARSIZE_COMPRESSED(...)
或 SET_VARSIZE(...)
```
再构造 `VARTAG_INDIRECT` datum，
把主表 tuple 的对应 attribute 指向内存里的 reconstructed datum。
最后用 `heap_form_tuple()` 重建 tuple，
把结果 copy 回 `change->data.tp.newtuple` 的 tuplebuf。
这一步之后，
插件看到的 main table tuple 已经能解释 changed TOAST value。
处理完主表 change 后，
如果 `clear_toast_afterwards` 为 true，
就调用：
```text
ReorderBufferToastReset(rb, txn)
```
reset 会遍历 hash，
释放 `ent->reconstructed`，
释放每个 chunk change，
destroy hash。
这条生命周期解释了 streaming mode 的 partial-change gate。
`ReorderBufferProcessPartialChange()` 遇到 toast insert 时，
给 top transaction 设置：
```text
RBTXN_HAS_PARTIAL_CHANGE
```
直到主表 INSERT/UPDATE 且 `clear_toast_afterwards` 为 true，
才清掉 partial 标记。
如果允许 pending toast chunks 时 streaming，
下游会收到不可解释的半成品。
所以 streaming 也必须遵守：
```text
只能 stream complete changes。
```
## 11. old key、old tuple、new tuple 与 replica identity
UPDATE / DELETE 的 old image 要分两层理解。
WAL flags 层：
```text
XLH_UPDATE_CONTAINS_OLD_TUPLE
XLH_UPDATE_CONTAINS_OLD_KEY
XLH_DELETE_CONTAINS_OLD_TUPLE
XLH_DELETE_CONTAINS_OLD_KEY
```
decode storage 层：
```text
无论 old key 还是 full old tuple，
decode.c 都放入 change->data.tp.oldtuple。
```
因此不能只看 `oldtuple != NULL` 就说下游有旧整行。
它只能说明：
```text
WAL 中有某种 old image。
```
`pgoutput` 和 logical replication protocol 再把这个区别显式化。
`logicalrep_write_update()` 的规则是：
```text
oldslot != NULL 时:
  REPLICA_IDENTITY_FULL -> 发送 'O'，old tuple follows
  其它 identity 模式   -> 发送 'K'，old key follows
然后发送 'N'，new tuple follows
```
`logicalrep_write_delete()` 也用 `O` / `K` 区分 full old tuple 和 key。
`pgoutput_change()` 在发送前把 tuple image 放进 slots：
```text
ExecStoreHeapTuple(change->data.tp.oldtuple, old_slot, false)
ExecStoreHeapTuple(change->data.tp.newtuple, new_slot, false)
```
然后执行 publication action、row filter、partition root mapping 和 protocol write。
这解释了为什么 row filter 也依赖稳定 old/new tuple。
UPDATE 的 row filter 可能出现四种语义转换：
```text
old 不匹配，new 匹配:
  UPDATE 转 INSERT。
old 匹配，new 不匹配:
  UPDATE 转 DELETE。
old 和 new 都匹配:
  继续 UPDATE。
old 和 new 都不匹配:
  不发送。
```
如果 DELETE 没有 oldtuple，
`pgoutput_change()` 会 DEBUG1：
```text
didn't send DELETE change because of missing oldtuple
```
然后返回。
这通常指向 schema 或 replica identity 边界，
不是 reorder buffer 丢数据。
`logicalrep_write_tuple()` 还有一个 TOAST 相关协议语义。
如果某列仍是 on-disk external toasted datum：
```text
VARATT_IS_EXTERNAL_ONDISK(...)
```
它发送：
```text
LOGICALREP_COLUMN_UNCHANGED
```
这不是不完整。
它表示该 toasted column 在本次 change 中没有变化，
协议不需要重发大值，
subscriber 保留原值即可。
所以“完整且语义稳定”不是说每个 column 都必须内联成完整 bytes。
它的意思是：
```text
需要表达的 changed value 已经完整；
不需要表达的 unchanged value 有明确协议标记；
old key/full old tuple 的区别由 replica identity 和协议显式表达。
```
## 12. tuplecid / snapshot：隐藏状态如何支撑插件语义
logical decoding replay 时要访问 catalog。
它需要把 `rlocator` 映射为 relation OID，
打开 relation，
读取 tuple descriptor，
找类型输出函数，
判断 replica identity、publication 和 row filter。
这些都依赖正确 catalog snapshot。
问题是一个 transaction 可以先改 catalog，
再写用户表。
decode 后面的用户表 change 时，
必须看到 transaction 内部正确 command id 下的 catalog 版本。
普通运行时依赖 cmin/cmax 和 combo CID。
但 `snapbuild.c` 注释说明：
```text
cmin/cmax 不在普通 WAL record 中；
crash recovery 会重置相关 tuple 信息；
combo CID 只在原 backend 内存中有解释表。
```
因此 heapam 在 logical decoding 需要时写：
```text
XLOG_HEAP2_NEW_CID
```
`decode.c` 遇到这个 record 后调用：
```text
SnapBuildProcessNewCid(builder, xid, lsn, xlrec)
```
`SnapBuildProcessNewCid()` 做三件事：
```text
ReorderBufferXidSetCatalogChanges(...)
ReorderBufferAddNewTupleCids(... xlrec->top_xid, target_locator, target_tid, cmin, cmax, combocid)
ReorderBufferAddNewCommandId(... cid + 1)
```
tuplecid mapping 存在 top transaction 上。
这和 subtransaction flatten 一致：
```text
catalog visibility 是 top transaction replay 的整体语义，
不能散落在某个可能已经被归并或 abort 的 subxact 局部状态里。
```
`ReorderBufferBuildTupleCidHash()` 在 replay 前把 list 建成 hash：
```text
key = (relfilelocator, tid)
value = (cmin, cmax, combocid)
```
同一 tuple 多次出现时，
源码要求 cmin 相同，
cmax 可以从 invalid 变成 valid，
且一旦 valid 只能增长。
`ReorderBufferProcessTXN()` 再把 hash 交给：
```text
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
因此插件看似只收到：
```text
Relation relation
ReorderBufferChange *change
```
但 relation 和 tuple 的语义来自：
```text
base snapshot
internal snapshot changes
internal command id changes
tuplecid hash
cache invalidations
subxact fate
```
这也是为什么 tuplecid 不进入插件 API。
它是内部 visibility machinery，
不是外部复制协议的一部分。
## 13. lifecycle / ownership / cleanup
`ReorderBufferAllocate()` 创建 backend-local reorder buffer。
它的 transaction、change、tuplebuf 分别用专门 memory contexts 管理。
`reorderbuffer.c` 顶部注释提到：
```text
SlabContext 用于 fixed-length change / transaction；
GenerationContext 用于 variable-length transaction data。
```
每个 change 的 ownership 随生命周期变化。
普通 row change：
```text
ReorderBufferAllocChange()
  -> ReorderBufferQueueChange()
  -> txn->changes 持有
  -> ReorderBufferProcessTXN() replay
  -> ReorderBufferCleanupTXN() 释放
```
TOAST chunk change：
```text
ReorderBufferQueueChange()
  -> txn->changes 暂存
  -> replay 时发现 IsToastRelation()
  -> dlist_delete(&change->node)
  -> ReorderBufferToastAppendChunk() 挂到 toast_hash
  -> ReorderBufferToastReset() 释放
```
tuplecid change：
```text
ReorderBufferAddNewTupleCids()
  -> txn->tuplecids
  -> ReorderBufferBuildTupleCidHash()
  -> SetupHistoricSnapshot()
  -> cleanup/truncate 时释放
```
snapshot ownership 由 SnapBuild refcount 和 reorder buffer cleanup 共同管理。
`ReorderBufferTransferSnapToParent()` 可能把 subtxn snapshot 转给 top txn，
并调整 `txns_by_base_snapshot_lsn`。
commit 后 `ReorderBufferCleanupTXN()` 会：
```text
清理 subtransactions
释放 changes
更新 memory accounting
释放 tuplecids
SnapBuildSnapDecRefcount(base_snapshot)
释放 snapshot_now
从 list 和 by_txn hash 移除 txn
清理 spill 文件
释放 ReorderBufferTXN
```
streaming 或 prepared transaction 可能先走 `ReorderBufferTruncateTXN()`。
它丢弃已经处理的 changes，
但保留 transaction、tuplecids、invalidations 和 snapshots 等后续 commit/rollback 仍需的状态。
这不是内存优化细节。
它是 streaming correctness 的一部分：
```text
已经发送的 changes 可以本地截断；
但 commit/abort/invalidation/snapshot 边界仍要保留到最终 record。
```
ERROR 路径由 `ReorderBufferProcessTXN()` 的 `PG_TRY/PG_CATCH` 收尾。
catch 中会：
```text
finish iterator
TeardownHistoricSnapshot(true)
AbortCurrentTransaction()
执行 invalidations 或 InvalidateSystemCaches()
恢复外层 MemoryContext / ResourceOwner
对 concurrent abort 特例 graceful reset
否则 ReorderBufferCleanupTXN() 后 rethrow
```
这个路径保证 output plugin 或 catalog access 报错时，
不会留下 historic snapshot、内部 transaction、toast hash 或 relcache 污染。
## 14. 异常路径与边界判断
### TOAST chunk 顺序错误
`ReorderBufferToastAppendChunk()` 要求 chunk sequence 从 0 开始且连续。
如果不满足，
直接 ERROR。
诊断方向应回到：
```text
WAL 是否损坏；
decoder 是否读错范围；
是否有 extension 写入不符合 heap/toast 约定的 WAL；
是否用错源码版本假设。
```
这不是 subscriber apply 慢的问题。
### 找不到 relation 或 toast relation
`ReorderBufferProcessTXN()` 用 `RelidByRelfilenumber()` 映射 relation。
普通 row change 找不到 relation 会 ERROR。
但 mapped catalog tuple without data 是例外：
```text
reloid == InvalidOid
newtuple == NULL
oldtuple == NULL
```
这种 rewrite 相关 record 可以跳过。
`ReorderBufferToastReplace()` 打不开 toast relation 时也会 ERROR。
这通常说明 historic catalog 解释或 relation metadata 边界有问题。
### DELETE 没有 oldtuple
pgoutput 遇到 DELETE 且 `oldtuple == NULL` 会跳过发送。
先检查：
```text
REPLICA IDENTITY NOTHING
是否有 replica identity index
publication 是否发布 DELETE
是否上游操作设置了 no logical old image
```
不要先假设 reorder buffer 漏掉 old tuple。
### partial change 阻止 streaming
大事务超过 `logical_decoding_work_mem` 后不一定立刻 stream。
如果 top transaction 有 pending toast chunks 或 speculative insert，
`RBTXN_HAS_PARTIAL_CHANGE` 会阻止 streaming。
现象可能是：
```text
logical decoding memory 增长；
apply lag 增长；
但没有立即 stream changes。
```
这是完整性门槛，
不是阈值失效。
### concurrent abort during streaming
streaming in-progress transaction 或 decoding prepared transaction 时，
`SetupCheckXidLive()` 会设置 `CheckXidAlive`。
catalog lookup 如果发现 xid 已 abort，
会抛 `ERRCODE_TRANSACTION_ROLLBACK`。
`PG_CATCH` 对这个特例会 graceful reset，
标记 transaction aborted，
等待 abort record 或 rollback prepared 完成最终语义。
### output plugin 使用 XID
`ReorderBufferProcessTXN()` 回调后检查：
```text
GetCurrentTransactionIdIfAny()
```
如果插件在 callback 中分配了 XID，
会 ERROR。
这提醒插件边界：
```text
插件可以读取 catalog 和构造输出；
不应该在 decoding callback 中制造持久数据库写入语义。
```
## 15. 成本、资源与跨模块传播
本节成本主要随四个变量扩张。
第一是 row changes 数。
每个 change 入队都要更新 list、计数、memory accounting，
并可能触发 spill 检查。
第二是 subtransaction 数。
commit replay 要在 top transaction 和 subtransactions 之间做 k-way merge。
subxact 越多，
iterator state、transaction records、snapshot transfer 和 cleanup 成本越明显。
第三是 TOAST chunk 数和 value size。
TOAST reconstruction 包含：
```text
hash lookup
chunk list traversal
palloc reconstructed datum
memcpy chunk bytes
heap_deform_tuple()
heap_form_tuple()
memory accounting update
```
大 varlena UPDATE 的 CPU 和内存峰值可能集中在 commit replay 或 streaming complete-change 边界。
第四是 catalog churn。
catalog-modifying transaction 会产生 `XLOG_HEAP2_NEW_CID`，
增加 tuplecid list/hash，
并让 historic snapshot、syscache 和 invalidation 成本上升。
跨模块传播路径是：
```text
heapam.c / toast_internals.c:
  决定 WAL 中是否有足够 tuple image。
snapbuild.c:
  决定何时有 consistent snapshot、base snapshot、tuplecid 和 new command id。
reorderbuffer.c:
  决定 subxact fate、TOAST assembly、streaming gate、cleanup 和 spill。
pgoutput.c / proto.c:
  决定 old key/full old tuple、row filter、publication 和 wire protocol。
replication slot:
  decode 进度慢会让 restart_lsn / confirmed_flush_lsn 推进慢，
  进一步造成 WAL retention。
```
所以 logical replication lag 不一定是网络慢。
它也可能是：
```text
大事务太大；
subxact 太多；
TOAST reconstruction CPU 高；
spill IO 高；
catalog changes 让 tuplecid/hash/invalidation 成本高；
output plugin row filter 或 type output 函数成本高。
```
## 16. 观测与诊断入口与课堂实验
本节锚定的 runtime truth 是：
```text
插件不会看到 abort subtransaction 的 row change；
插件不会把 toast chunk 当用户表 row change；
UPDATE/DELETE 的 old image 由 replica identity 决定。
```
能直接观测的入口：
```text
pg_logical_slot_get_changes()
pg_stat_replication
pg_stat_subscription
slot restart_lsn / confirmed_flush_lsn
server log
```
只能推断或需要断点的状态：
```text
txn->subtxns
txn->toast_hash
txn->tuplecid_hash
RBTXN_HAS_PARTIAL_CHANGE
ReorderBufferIterTXNState 当前 heap head
```
建议断点：
```text
ReorderBufferAssignChild
ReorderBufferCommitChild
ReorderBufferAbort
ReorderBufferProcessTXN
ReorderBufferToastAppendChunk
ReorderBufferToastReplace
ReorderBufferToastReset
pgoutput_change
logicalrep_write_update
logicalrep_write_delete
```
### 实验一：abort subxact 不输出
```sql
CREATE EXTENSION IF NOT EXISTS test_decoding;
SELECT pg_create_logical_replication_slot('rb_sub_slot', 'test_decoding');
CREATE TABLE rb_sub (id int PRIMARY KEY, v text);
BEGIN;
INSERT INTO rb_sub VALUES (1, 'top-before');
SAVEPOINT s;
INSERT INTO rb_sub VALUES (2, 'sub-abort');
ROLLBACK TO s;
INSERT INTO rb_sub VALUES (3, 'top-after');
COMMIT;
SELECT data
FROM pg_logical_slot_get_changes('rb_sub_slot', NULL, NULL);
```
观察点：
```text
输出不应包含 id = 2；
ReorderBufferAbort() 会清理 rollback subxact；
最终 commit 只 replay surviving changes。
```
### 实验二：TOAST chunks 不越过插件边界
```sql
SELECT pg_create_logical_replication_slot('rb_toast_slot', 'test_decoding');
CREATE TABLE rb_toast (id int PRIMARY KEY, v text);
INSERT INTO rb_toast
VALUES (1, repeat('toast-value-', 20000));
SELECT data
FROM pg_logical_slot_get_changes('rb_toast_slot', NULL, NULL);
```
断点顺序通常是：
```text
toast_save_datum()
DecodeInsert()
ReorderBufferQueueChange(... toast_insert=true)
ReorderBufferToastAppendChunk()
ReorderBufferToastReplace()
output plugin change callback
ReorderBufferToastReset()
```
输出应是主表 row change，
不是 toast relation chunk rows。
### 实验三：old key 与 full old tuple
```sql
SELECT pg_create_logical_replication_slot('rb_old_slot', 'test_decoding');
CREATE TABLE rb_old (
  id int PRIMARY KEY,
  v text
);
INSERT INTO rb_old VALUES (1, 'a');
UPDATE rb_old SET id = 2 WHERE id = 1;
DELETE FROM rb_old WHERE id = 2;
ALTER TABLE rb_old REPLICA IDENTITY FULL;
INSERT INTO rb_old VALUES (3, repeat('z', 10000));
UPDATE rb_old SET v = repeat('q', 10000) WHERE id = 3;
DELETE FROM rb_old WHERE id = 3;
SELECT data
FROM pg_logical_slot_get_changes('rb_old_slot', NULL, NULL);
```
不同 output plugin 的文本格式不同。
要观察的不是固定字符串，
而是：
```text
default/index identity 下 old image 更接近 key；
REPLICA IDENTITY FULL 下 old image 是旧整行；
proto.c 会用 'K' / 'O' 区分。
```
如果 lag 问题主要是 CPU，
`pg_stat_*` 不足以归因。
需要结合：
```text
perf / flamegraph
gdb breakpoint
临时 elog
logical_decoding_work_mem 对照实验
大 TOAST value / 多 subxact workload 对照实验
```
## 17. 常见误区
误区一：WAL 中有 toast relation insert，所以插件应该看到 toast table rows。实际 toast rows 是主表 tuple image 的内部重组输入。
误区二：`oldtuple != NULL` 就是旧整行。实际它可能只是 replica identity key，full old tuple 取决于 `REPLICA IDENTITY FULL`。
误区三：subxact 有 XID，所以可以独立发布。实际插件事务边界是 top transaction，subxact 只提供 surviving change stream。
误区四：commit 时把 subxact list 接到 top list 后面就行。实际必须按 LSN k-way merge，保留 WAL 全局顺序。
误区五：tuplecid 是插件输出内容。实际 tuplecid 是 historic catalog visibility 的内部输入。
误区六：logical replication lag 主要是网络或 subscriber。实际 reorder buffer 层的大事务、TOAST、subxact、spill、catalog churn、plugin CPU 都可能主导延迟。
## 18. 讨论题
1. 为什么 `ReorderBufferCommitChild()` 必须在 `ReorderBufferCommit()` 前调用？
2. 如果 `ReorderBufferToastReset()` 在 toast chunk insert 后立刻执行，会破坏什么？
3. 为什么 UPDATE 的 `oldtuple` 字段不能单独说明“旧整行已存在”？
4. DELETE 没有 oldtuple 时，为什么 pgoutput 只能跳过发送？
## 19. 本节小结
本节核心链路是：
```text
heapam 写出面向 recovery 的 WAL fragment；
decode.c 解析成 ReorderBufferChange；
ReorderBuffer 按 xid 暂存；
subxact assignment 和 commit/abort record 决定 transaction fate；
commit replay 用 historic snapshot 和 tuplecid hash 建立 catalog 语义；
TOAST chunks 在插件前拼回主表 tuple；
old key/full old tuple 由 replica identity 和 protocol 显式区分；
插件只看到完整且语义稳定的 row-level change；
cleanup 清理所有中间状态。
```
核心状态边界是：
```text
ReorderBufferChange:
  oldtuple / newtuple / rlocator / clear_toast_afterwards。
ReorderBufferTXN:
  changes / subtxns / base_snapshot / tuplecids / toast_hash / txn_flags。
ReorderBufferToastEnt:
  chunk_id / last_chunk_seq / chunks / reconstructed。
```
正确性不是单一机制保证的。
它来自：
```text
heap WAL flags 提供必要 tuple bytes；
replica identity 决定 old image；
subxact commit/abort 决定 surviving changes；
LSN k-way merge 保留顺序；
TOAST hash 重组 changed out-of-line datum；
historic snapshot + tuplecid hash 保证 catalog visibility；
cleanup 避免 abort/error 状态污染后续 decoding。
```
能直接看到的是插件输出、slot LSN、replication lag 和部分日志。
看不到的是 `toast_hash`、`tuplecid_hash`、subxact list 的瞬时状态和 iterator heap。
这些需要断点、临时日志或 profiling。
可迁移规律是：
```text
当底层日志为了恢复效率记录物理碎片，
而上层 API 要求稳定逻辑事件时，
系统必须在中间维护 transaction fate、object reconstruction、metadata visibility 和 cleanup 边界；
否则扩展接口会暴露未提交、未重组或依赖瞬时存储状态的对象。
```
哪些判断仍然 workload-dependent：TOAST reconstruction、subxact merge、spill、tuplecid/invalidation、row filter 和 subscriber apply 谁主导成本。
但插件侧不变量不变：
```text
output plugin 应处理完整且语义稳定的 row-level change，
不应承担 subxact fate、TOAST chunk assembly、catalog tuplecid 或 WAL physical layout 的重组责任。
```
