# PostgreSQL Reorder Buffer 事务重组与提交顺序输出
## 课程定位
前置知识：已经理解 WAL 按 LSN 追加、logical slot 保护 `restart_lsn` 和 `catalog_xmin`，也知道 logical decoding 需要从历史 WAL 和历史 catalog 版本中还原事务语义。

本节唯一主问题：

```text
logical decoding 按 WAL 顺序读取变更，
为什么还需要按事务提交顺序输出，
ReorderBufferTXN 如何把跨 record、跨 subxact 的变更重新组织起来？
```

核心矛盾：WAL record 的顺序是物理产生顺序；logical consumer 需要的是事务可见顺序。多个事务的 heap record 可以交错写入 WAL，subtransaction 和 top-level transaction 的关系可能到 assignment 或 commit record 才完整，abort 事务也可能已经写过很多 heap WAL。

PostgreSQL 不在 `DecodeInsert()`、`DecodeUpdate()` 里直接调用 output plugin。它先把 WAL record 解成 `ReorderBufferChange`，按 XID 挂到 `ReorderBufferTXN`，等 commit record 到达时再用 `ReorderBufferCommit()` 输出整个顶层事务。

学完后应能判断：为什么 `by_txn` 和 LSN 视角必须同时存在；为什么 subxact 的 change 不能简单拼到父事务尾部；为什么没有 `base_snapshot` 的事务即使提交也可能没有可输出内容；为什么 abort cleanup 和 catalog invalidation 是 logical decoding correctness 的一部分。

本课基于本地源码 `/home/nail/postgres`，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置
前面 replication slot 课程解决的是：

```text
历史 WAL 和历史 catalog 版本还在不在？
```

本节解决的是：

```text
这些历史 record 如何被重新组织成事务级 logical stream？
```

外部消费者看到的 logical stream 类似：

```text
BEGIN xid
INSERT / UPDATE / DELETE / MESSAGE / TRUNCATE
COMMIT xid
```

但 WAL 里不是这样排布。

一个真实时间线可以是：

```text
LSN 10: xid A heap insert
LSN 20: xid B heap insert
LSN 30: xid B commit
LSN 40: xid A heap update
LSN 50: xid A commit
```

如果按 record 顺序输出，消费者会先看到 A 的 insert。

这泄露了一个尚未提交的事务。

正确输出应先交付 B，再交付 A：

```text
BEGIN B
B insert
COMMIT B

BEGIN A
A insert
A update
COMMIT A
```

所以本节主线是：

```text
WAL 顺序读取
  -> 按 XID 聚合 change
  -> 建立 top/subxact 关系
  -> commit record 到达
  -> 用 historic snapshot 解码
  -> 在顶层事务内部按 change LSN merge 输出
  -> cleanup
```

本节只把 output plugin 当作最终 callback 边界，不展开格式协议。

大事务 spill 只作为资源退化路径出现；下一节再专门讨论 `logical_decoding_work_mem`。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：

```text
decode.c 按 WAL record 顺序把 heap/xact/logical-message record 转成 ReorderBufferChange；
reorderbuffer.c 先按 XID 找到 ReorderBufferTXN 并把 change 追加到该事务自己的 LSN 有序链；
等 xact commit record 到达后，再把顶层事务和各 subtransaction 的 change 做 k-way merge，
在 historic snapshot 下调用 output plugin 的 begin/change/commit callbacks。
```

这里有两个顺序。

第一个是 WAL record 顺序：

```text
所有 record 按 LSN 单调读取。
```

第二个是 logical transaction 输出顺序：

```text
只输出 committed transaction；
事务之间按 commit record 到达顺序输出；
事务内部按各 change 的 LSN 顺序输出。
```

WAL 顺序适合 recovery，因为 recovery 要按物理因果恢复 page、CLOG、subtrans、relmap 和 invalidation。

logical decoding 面向外部语义，必须遵守：

```text
未提交事务不可见；
abort 事务不可见；
subxact abort 的变更不可见；
catalog lookup 必须使用当时可见的 schema；
输出插件看到的是事务边界内的稳定 change stream。
```

最小反例是两个并发事务。

A 很早写第一条 row。

B 晚一点写 row 但先提交。

按 WAL change 输出会让下游先应用 A。

按 commit 输出会让下游先应用 B。

数据库 MVCC 的语义是后者。

ReorderBuffer 的职责是把“按 record 到来的碎片”改造成“按提交边界交付的事务”。

## 3. 核心文件分工与阅读顺序
不要从 `reorderbuffer.c` 第一行读到最后。

先读入口、状态、commit 输出边界。

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/replication/logical/logical.c` | `StartupDecodingContext()` 创建 `LogicalDecodingContext`、`ReorderBuffer`、`SnapBuild`，并把 reorder buffer callback 接到 output plugin wrapper。 |
| 2 | `src/backend/replication/logical/decode.c` | `LogicalDecodingProcessRecord()` 按 WAL record 分发；heap record 进入 `DecodeInsert()` / `DecodeUpdate()` / `DecodeDelete()`；xact record 进入 `DecodeCommit()` / `DecodeAbort()`。 |
| 3 | `src/include/replication/reorderbuffer.h` | `ReorderBuffer`、`ReorderBufferTXN`、`ReorderBufferChange` 的字段语义和 callback 类型。 |
| 4 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferQueueChange()`、`ReorderBufferAssignChild()`、`ReorderBufferCommitChild()`、`ReorderBufferCommit()`、`ReorderBufferProcessTXN()`。 |
| 5 | `src/backend/replication/logical/snapbuild.c` | `SnapBuildProcessChange()`、`SnapBuildCommitTxn()`、`SnapBuildProcessRunningXacts()` 决定能否解码和使用哪个 base snapshot。 |
| 6 | `src/backend/access/rmgrdesc/xactdesc.c` | `ParseCommitRecord()`、`ParseAbortRecord()`、`ParsePrepareRecord()` 从 xact WAL record 解析 subxact 数组。 |
| 7 | `src/backend/access/transam/xact.c` | `AssignTransactionId()`、`XactLogCommitRecord()`、`XactLogAbortRecord()` 产生 assignment 和 commit/abort record。 |

阅读时一直追问：

```text
当前 record 是 data change，还是 transaction boundary？
当前 XID 是 top xid，还是 subxid？
这条 change 已经可输出了吗？
如果当前事务 abort，哪些状态必须消失？
如果当前事务 commit，输出时需要哪个 snapshot？
```

## 4. 核心状态：ReorderBuffer / TXN / Change
`ReorderBuffer` 是 logical decoding backend 内的私有对象。

它不是 shared memory，普通指针不能跨进程使用。

`StartupDecodingContext()` 中创建它：

```text
ctx->reorder = ReorderBufferAllocate();
ctx->snapshot_builder = AllocateSnapshotBuilder(...);
ctx->reorder->private_data = ctx;
```

`ReorderBufferAllocate()` 建立自己的 memory context：

```text
"ReorderBuffer"
  -> "Change"  SlabContext
  -> "TXN"     SlabContext
  -> "Tuples"  GenerationContext
```

`Change` 和 `TXN` 是定长对象，适合 slab。

tuple image 是变长数据，放入 generation context 降低长事务碎片。

`ReorderBuffer` 最关键的不是一个全局 change 队列，而是两个视角。

第一个视角是 `by_txn`：

```text
HTAB *by_txn;
TransactionId by_txn_last_xid;
ReorderBufferTXN *by_txn_last_txn;
```

它回答：

```text
我现在读到 xid X 的 record，应该追加到哪个事务对象？
```

第二个视角是 LSN。

源码里没有一个字段直接叫 `by_lsn`。

本节说的 by_lsn 视角主要落在：

```text
ReorderBuffer.toplevel_by_lsn
每个 ReorderBufferTXN.changes 链表中的 change->lsn 顺序
```

`toplevel_by_lsn` 保存“可能是顶层事务”的 `ReorderBufferTXN`，按第一次相关 record 的 LSN 排序。

每个 `txn->changes` 保存该事务自己的 change，按 WAL 到达顺序追加。

`ReorderBufferTXN` 可以表示 top-level transaction，也可以表示 subtransaction。

关键字段组合是：

```text
xid
toplevel_xid
toptxn
first_lsn
final_lsn
end_lsn
base_snapshot
changes
subtxns
tuplecids
invalidations
txn_flags
```

`xid` 是对象自己的 XID。

`toplevel_xid` 是已知顶层 XID。

`toptxn` 是指向顶层事务对象的 backend-local 指针。

`toptxn == NULL` 时，宏 `rbtxn_is_toptxn(txn)` 为真。

`toptxn != NULL` 时，它是 subtransaction。

`first_lsn` 不是 XID 第一次出现在任何 WAL record 的 LSN，而是第一次 logical-decoding-relevant data carrying record 的 LSN。

`final_lsn` 是导致 commit、prepare 或 abort 的 record LSN；在 spill 过程中也可能临时表示已写到磁盘的最后 change LSN。

`end_lsn` 是 commit record 的结束位置。

`base_snapshot` 是解码该事务变更时使用的 historic snapshot 起点。

没有 `base_snapshot`，说明这个事务没有可解码的数据变更。

这类事务即使 commit，`ReorderBufferReplay()` 也会 cleanup 而不输出 row change。

`changes` 只保存当前 XID 自己的 change。

subtransaction 的 change 留在 subxact 自己的 `changes` 中。

顶层事务的 `subtxns` 是 non-hierarchical 列表。

嵌套 subxact 树在 reorder buffer 中被压平成 top-level 下的一层列表。

这是可以的，因为 logical decoding 只需要知道哪些 subxact surviving，并且 aborted subxact 会被 abort record 单独清理。

`ReorderBufferChange` 是 WAL record 的 logical 片段。

它包含：

```text
lsn
action
txn
origin_id
union data
node
```

用户可见 action 包括：

```text
REORDER_BUFFER_CHANGE_INSERT
REORDER_BUFFER_CHANGE_UPDATE
REORDER_BUFFER_CHANGE_DELETE
REORDER_BUFFER_CHANGE_MESSAGE
REORDER_BUFFER_CHANGE_TRUNCATE
```

内部控制 action 包括 snapshot、command id、tuple cid、invalidation、speculative insertion confirm/abort。

因此 `txn->changes` 不是纯 row change 链表。

它还夹着输出 row data 所需的 snapshot、CID 和 invalidation 控制事件。

## 5. 主流程 walkthrough：从 WAL record 到 commit 输出
逻辑解码读 WAL 的入口是 `LogicalDecodingProcessRecord()`。

它构造 `XLogRecordBuffer`：

```text
buf.origptr = ctx->reader->ReadRecPtr
buf.endptr = ctx->reader->EndRecPtr
buf.record = record
```

然后读取 top-level xid：

```text
txid = XLogRecGetTopXid(record)
```

如果 top xid 有效，先建立 top/subxact 关系：

```text
ReorderBufferAssignChild(ctx->reorder,
                         txid,
                         XLogRecGetXid(record),
                         buf.origptr)
```

这一步对所有带 top xid 的 WAL record 都会发生，不只处理 `XLOG_XACT_ASSIGNMENT` record。

随后才按 rmgr 分发：

```text
rmgr = GetRmgr(XLogRecGetRmid(record))

if rmgr.rm_decode != NULL:
  rmgr.rm_decode(ctx, &buf)
else:
  ReorderBufferProcessXid(...)
```

对 heap record，`heap_decode()` / `heap2_decode()` 处理行变更。

对 xact record，`xact_decode()` 处理 commit、abort、prepare、assignment 和 invalidation。

`XLOG_XACT_ASSIGNMENT` 在 `xact_decode()` 里基本不做事，因为 assignment 主路径已经前移到 `LogicalDecodingProcessRecord()`。

以 INSERT 为例。

`heap_decode()` 看到 `XLOG_HEAP_INSERT` 后先问 snapshot builder：

```text
SnapBuildProcessChange(builder, xid, buf->origptr)
```

如果还没有足够 snapshot，返回 false。

这条 change 不进入 reorder buffer。

如果可以解码，且不是 fast-forward，进入 `DecodeInsert()`。

`DecodeInsert()` 做几件事：

```text
检查 WAL record 是否含 new tuple
检查 block tag 的 database 是否等于 slot database
检查 origin 过滤
分配 ReorderBufferChange
复制 tuple image 到 reorder buffer 的 tuple buffer
设置 clear_toast_afterwards
调用 ReorderBufferQueueChange()
```

`ReorderBufferQueueChange()` 先按 XID 找事务：

```text
txn = ReorderBufferTXNByXid(rb, xid, true, NULL, lsn, true)
```

`create=true` 表示不存在就创建。

`create_as_top=true` 表示暂时把它当作 top-level transaction 放入 `toplevel_by_lsn`。

这不是说它一定是 top-level。

如果后来 `ReorderBufferAssignChild()` 发现它是 subxact，会把它从 `toplevel_by_lsn` 删除，再挂到父事务的 `subtxns`。

入队时写入：

```text
change->lsn = lsn
change->txn = txn
dlist_push_tail(&txn->changes, &change->node)
txn->nentries++
txn->nentries_mem++
```

然后更新内存计数并检查限制：

```text
ReorderBufferChangeMemoryUpdate(...)
ReorderBufferCheckMemoryLimit(rb)
```

到这里没有 output plugin callback。

data record 的职责只是：

```text
找到事务对象
复制需要的 logical data
按 LSN 追加到该事务自己的 change 链
维护内存和 spill 前置状态
```

subxact 关系的纠正路径是 `ReorderBufferAssignChild()`：

```text
txn = ReorderBufferTXNByXid(rb, xid, true, &new_top, lsn, true)
subtxn = ReorderBufferTXNByXid(rb, subxid, true, &new_sub, lsn, false)
```

如果 subtxn 之前被当作 top-level 放入 `toplevel_by_lsn`，现在会：

```text
dlist_delete(&subtxn->node)
```

然后设置：

```text
subtxn->txn_flags |= RBTXN_IS_SUBXACT
subtxn->toplevel_xid = xid
subtxn->toptxn = txn
dlist_push_tail(&txn->subtxns, &subtxn->node)
txn->nsubtxns++
```

subtransaction 的 changes 不会搬到父事务的 `changes` 尾部。

假设：

```text
LSN 10: top xid 10 change A
LSN 20: subxid 11 change B
LSN 30: top xid 10 change C
```

如果 commit 时把 B append 到父链尾部，输出会是：

```text
A, C, B
```

正确顺序应该是：

```text
A, B, C
```

因此 PostgreSQL 保留每个 XID 的独立 change stream，提交时做 merge。

commit record 的入口是 `xact_decode()`。

它用 `ParseCommitRecord()` 解析：

```text
dbId
xact_time
nsubxacts
subxacts
origin
twophase_xid / gid
```

然后进入 `DecodeCommit()`。

`DecodeCommit()` 先通知 snapshot builder：

```text
SnapBuildCommitTxn(ctx->snapshot_builder,
                   buf->origptr,
                   xid,
                   parsed->nsubxacts,
                   parsed->subxacts,
                   parsed->xinfo)
```

随后用 `DecodeTXNNeedSkip()` 判断是否跳过。

跳过原因包括：

```text
还没到 start_decoding_at
事务属于其他 database
origin 被插件过滤
fast_forward 模式
```

如果跳过，走 `ReorderBufferForget()`，不是 `ReorderBufferAbort()`。

原因是事务已经提交，可能仍需要执行 catalog invalidation。

如果不跳过，先处理 surviving subtransactions：

```text
for each parsed->subxacts:
  ReorderBufferCommitChild(ctx->reorder, xid, subxid,
                           buf->origptr, buf->endptr)
```

`ReorderBufferCommitChild()` 会设置 subtxn 的 `final_lsn` 和 `end_lsn`，再调用 `ReorderBufferAssignChild()`。

最后才是：

```text
ReorderBufferCommit(ctx->reorder,
                    xid,
                    buf->origptr,
                    buf->endptr,
                    commit_time,
                    origin_id,
                    origin_lsn)
```

`ReorderBufferCommit()` 查 `by_txn`。

未知事务直接返回。

这可能是空事务、没有 logical-relevant change 的事务，或已经被 skip 的事务。

已知事务进入 `ReorderBufferReplay()`。

`ReorderBufferReplay()` 先填 commit 信息：

```text
txn->final_lsn = commit_lsn
txn->end_lsn = end_lsn
txn->commit_time = commit_time
txn->origin_id = origin_id
txn->origin_lsn = origin_lsn
```

如果 `txn->base_snapshot == NULL`，说明没有可解码数据变更。

它 cleanup 后返回。

否则进入：

```text
ReorderBufferProcessTXN(rb, txn, commit_lsn,
                        txn->base_snapshot,
                        FirstCommandId,
                        false)
```

这才是普通 committed transaction 的输出路径。

## 6. 事务内部 merge 与 output callback 边界
`ReorderBufferProcessTXN()` 不直接遍历 `txn->changes`。

它先构建 tuple CID hash：

```text
ReorderBufferBuildTupleCidHash(rb, txn)
```

然后设置 historic snapshot：

```text
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```

再启动 decoder 自己的内部事务环境：

```text
StartTransactionCommand()
```

如果调用时已经在事务块里，则使用 internal subtransaction。

随后 ordinary commit replay 的 callback 顺序是：

```text
rb->begin(rb, txn)
for each merged change:
  rb->apply_change / rb->apply_truncate / rb->message
rb->commit(rb, txn, commit_lsn)
```

这些 callback 在 `StartupDecodingContext()` 中被接到 `logical.c` wrapper：

```text
ctx->reorder->begin = begin_cb_wrapper
ctx->reorder->apply_change = change_cb_wrapper
ctx->reorder->apply_truncate = truncate_cb_wrapper
ctx->reorder->commit = commit_cb_wrapper
ctx->reorder->message = message_cb_wrapper
```

wrapper 再调用真正 output plugin callback。

本节只关心前置条件：

```text
事务已经提交；
change 已按事务内部 LSN 顺序 merge；
historic snapshot 已 setup；
decoder 内部事务环境已经存在；
plugin 不能分配 XID 写数据库。
```

事务内部 merge 由：

```text
ReorderBufferIterTXNInit()
ReorderBufferIterTXNNext()
```

完成。

iterator 收集 top-level txn 和所有 non-empty subtxn 的 change 链。

每条链内部已经按 LSN 有序。

多条链之间用 `binaryheap` 维护当前 head 的最小 LSN。

伪流程：

```text
for each stream in top + subtxns:
  if stream has changes:
    heap.add(stream.head.lsn)

while heap not empty:
  stream = heap.min_lsn_stream
  change = stream.current
  advance stream
  if stream has next:
    heap.replace(next.lsn)
  else:
    heap.remove
  return change
```

这就是 reorder buffer 的核心重组动作。

它不是重新排序整个 WAL。

它是在一个已提交顶层事务范围内，把 top/subxact 多个局部有序流合并成一个全局 LSN 有序流。

MULTI_INSERT 可能让多个 change 有相同 LSN，所以 `ReorderBufferProcessTXN()` 只断言：

```text
prev_lsn <= change->lsn
```

不是严格小于。

行变更输出时还要打开 relation：

```text
reloid = RelidByRelfilenumber(...)
relation = RelationIdGetRelation(reloid)
RelationIsLogicallyLogged(relation)
```

这些 catalog lookup 都依赖 historic snapshot。

内部 change 也会影响输出环境：

```text
REORDER_BUFFER_CHANGE_INTERNAL_SNAPSHOT
  -> 切换 historic snapshot

REORDER_BUFFER_CHANGE_INTERNAL_COMMAND_ID
  -> 推进 snapshot curcid

REORDER_BUFFER_CHANGE_INVALIDATION
  -> 执行本地 cache invalidation
```

所以 output plugin 看到的是稳定 row change。

但 reorder buffer 内部链表中穿插着 snapshot、CID 和 invalidation 控制事件。

输出完成后，`ReorderBufferProcessTXN()` 会：

```text
ReorderBufferIterTXNFinish()
rb->commit(...)
TeardownHistoricSnapshot(false)
AbortCurrentTransaction()
执行 invalidations
ReorderBufferCleanupTXN()
```

这里的 `AbortCurrentTransaction()` 不是 abort 被解码的用户事务。

它只是清理 decoder 为 catalog access 开启的内部事务环境，释放锁和 ResourceOwner 资源，不让 callback 中的访问留下数据库写入效果。

## 7. snapshot / catalog visibility 边界
`snapbuild.c` 不负责排序。

它负责回答：

```text
当前是否已经有足够 historic snapshot？
这条 change 是否可以被收集？
某个 commit 是否会推进 catalog snapshot 状态？
slot 的 xmin / restart decoding LSN 能否前进？
```

`heap_decode()` 在 data change 前检查：

```text
if SnapBuildCurrentState(builder) < SNAPBUILD_FULL_SNAPSHOT:
  return
```

`SnapBuildProcessChange()` 还会拒绝那些开始太早、没有足够信息解码的事务。

如果事务首次需要 base snapshot，它会：

```text
builder->snapshot = SnapBuildBuildSnapshot(builder)
SnapBuildSnapIncRefcount(builder->snapshot)
ReorderBufferSetBaseSnapshot(builder->reorder, xid, lsn, builder->snapshot)
```

这说明 reorder buffer 不一定收集读到的所有 WAL change。

它只收集 snapshot builder 认为可解释的 logical-relevant change。

`SnapBuildProcessRunningXacts()` 处理 running-xacts record 时会：

```text
尝试构造或序列化 snapshot
更新 builder->xmin
清理不再需要的 catalog xact 状态
调用 ReorderBufferGetOldestXmin()
调用 ReorderBufferGetOldestTXN()
推进 slot 的 xmin 和 restart decoding LSN
```

这解释了 `toplevel_by_lsn` 的另一个用途。

它不只服务输出，还帮助判断：

```text
当前最老的 in-progress logical transaction 从哪里开始？
如果 decoder 崩溃重启，slot 最早要从哪个 snapshot serialization point 重新读？
```

因此 `by_txn` 和 LSN 视角共同维护：

```text
当前 record 归属
事务输出顺序
restart_lsn 推进边界
snapshot xmin 保留边界
```

## 8. XID assignment 与 xact record 来源
`xact.c` 的 `AssignTransactionId()` 解释了为什么 logical decoding 需要 assignment 信息。

当 effective WAL level 支持 logical 信息时，subtransaction 的 XID 不能裸露在 WAL 中而 top xid 完全未知。

必要时会写：

```text
XLOG_XACT_ASSIGNMENT
```

record 内容是：

```text
xl_xact_assignment
  xtop
  nsubxacts
  xsub[]
```

commit record 本身也可能包含 subxacts 数组。

`ParseCommitRecord()` 会解析 `xl_xact_subxacts`。

因此 ReorderBuffer 有两类机会知道关系：

```text
record header / assignment record 让关系提前可见
commit / abort record 让 surviving 或 aborted subxacts 最终可见
```

提前知道关系可以减少 `toplevel_by_lsn` 中的临时误判。

最终 commit/abort 仍然是输出和 cleanup 边界。

## 9. 生命周期 / ownership / cleanup
谁创建？

`StartupDecodingContext()` 创建 `ReorderBuffer`。

`ReorderBufferAllocate()` 创建内部 memory contexts、`by_txn` hash、`toplevel_by_lsn`、`txns_by_base_snapshot_lsn`、`catchange_txns` 和 `txn_heap`。

谁持有？

`LogicalDecodingContext` 持有 `ctx->reorder`。

每个 decoding backend 私有持有自己的 reorder buffer。

output plugin 只能通过 callback 参数短暂看到 `ReorderBufferTXN` 和 `ReorderBufferChange`。

它不能把这些指针当长期 public contract。

谁释放？

普通 committed transaction 输出后：

```text
ReorderBufferProcessTXN()
  -> ReorderBufferCleanupTXN()
```

abort record 到达时：

```text
DecodeAbort()
  -> ReorderBufferAbort(subxacts...)
  -> ReorderBufferAbort(top)
  -> ReorderBufferCleanupTXN()
```

decoding context 释放时：

```text
FreeDecodingContext()
  -> ReorderBufferFree()
```

`ReorderBufferCleanupTXN()` 做完整事务级 cleanup：

```text
递归清理 subtransactions
释放 txn->changes 中的 ReorderBufferChange
释放 tuple buffers、message buffers、truncate relids、snapshot
清理 tuplecids、tuplecid_hash、toast hash
对 base_snapshot 做 SnapBuildSnapDecRefcount()
从父列表或 toplevel_by_lsn 删除 node
从 catchange_txns 删除 catalog-changing txn
从 by_txn hash 删除 xid
删除 serialized spill 文件
释放 ReorderBufferTXN
```

abort cleanup 不是性能优化。

如果 abort 事务的 change 留在 reorder buffer 中，后续可能：

```text
占用 logical_decoding_work_mem
污染 toast reconstruction
保留过老 base snapshot
影响 restart_lsn 推进
污染 catalog invalidation 推理
```

`ReorderBufferAbortOld()` 是另一条 cleanup 路径。

它由 `standby_decode()` 处理 `XLOG_RUNNING_XACTS` 时调用，清理比 `oldestRunningXid` 更老且不可能还在运行的事务。

这对应 crash/immediate restart 后没有显式 abort record 的旧状态。

`ReorderBufferForget()` 用于已提交但当前 decoder 不输出内容的事务。

它不能当 abort 处理。

因为 committed transaction 可能包含 catalog invalidations，decoder 本地 cache 仍必须推进。

## 10. 正确性机制层次
ReorderBuffer 的 correctness 不是一个链表保证的。

它由多层机制叠加。

| 层次 | 机制 | 保证 | 不能保证 |
| --- | --- | --- | --- |
| WAL 顺序 | `XLogReadRecord()` / LSN | record 按物理顺序读入 | 事务已经可见 |
| XID 聚合 | `by_txn` / `ReorderBufferTXNByXid()` | 同一 XID 的 change 聚到同一对象 | top/sub 关系一定已知 |
| top/sub 关系 | `ReorderBufferAssignChild()` / `ReorderBufferCommitChild()` | subxact 归入顶层事务 | subxact change 已经在父链排好 |
| 事务边界 | `ReorderBufferCommit()` / `ReorderBufferAbort()` | commit 输出、abort 清理 | catalog snapshot 自动正确 |
| 可见性 | `SnapBuildProcessChange()` / `SetupHistoricSnapshot()` | 用历史 catalog 解释 change | output plugin 可以写数据库 |
| 内部事务 | `StartTransactionCommand()` / `AbortCurrentTransaction()` | decoder 能访问 syscache 并释放锁 | 被解码事务真的 abort |
| invalidation | `ReorderBufferExecuteInvalidations()` | 清理 decoder 本地 cache 语义 | 阻塞并发 catalog change |
| 资源控制 | memory accounting / spill | 限制 decoded change 内存 | 限制所有事务元数据内存 |

一个 raw field 不能单独解释语义。

`final_lsn` 只有结合事务状态才表示 commit/abort/prepare 边界。

`txn->changes` 只有结合 `txn->subtxns` 和 iterator 才表示输出顺序。

`base_snapshot` 只有结合 snapshot builder refcount 和 command id 才能用于 catalog lookup。

## 11. 异常路径 / fallback
第一类异常是 abort record。

`DecodeAbort()` 解析 abort record 后，对普通事务执行：

```text
for each parsed->subxacts:
  ReorderBufferAbort(ctx->reorder, subxid, EndRecPtr, abort_time)

ReorderBufferAbort(ctx->reorder, xid, EndRecPtr, abort_time)
```

未知事务直接返回。

已知事务清理所有 change、snapshot、toast、tuplecid、spill 文件和 `by_txn` entry。

第二类异常是 skip committed transaction。

`DecodeTXNNeedSkip()` 可能因为 database、origin、start LSN 或 fast-forward 决定跳过事务。

这时 `DecodeCommit()` 调用 `ReorderBufferForget()`。

语义是：

```text
不输出 row changes；
但如果 committed transaction 有 catalog invalidations，decoder 本地 cache 仍要处理。
```

第三类异常是 output callback 中途 ERROR。

`ReorderBufferProcessTXN()` 用 `PG_TRY()` 包住输出。

`PG_CATCH()` 中会：

```text
ReorderBufferIterTXNFinish()
TeardownHistoricSnapshot(true)
AbortCurrentTransaction()
执行 invalidations 或 InvalidateSystemCaches()
释放 internal subtransaction
ReorderBufferCleanupTXN()
PG_RE_THROW()
```

特殊的 streaming/prepared 并发 abort 会用 `ERRCODE_TRANSACTION_ROLLBACK` 走 reset 路径。

本节主线不展开 streaming。

普通 committed transaction 的 ERROR 会清理当前 reorder buffer 状态并重抛。

slot 的确认位置不会因为失败而前进到消费者没有安全接收的位置。

下次继续解码时，依靠 slot 的 restart/confirmed flush 边界重新读取需要的 WAL。

第四类 fallback 是内存压力。

`ReorderBufferChangeMemoryUpdate()` 更新：

```text
rb->size
txn->size
toptxn->total_size
txn_heap
```

超过 `logical_decoding_work_mem` 后，`ReorderBufferCheckMemoryLimit()` 会选择大事务 spill。

这里只记住：

```text
spill 驱逐 decoded changes；
ReorderBufferTXN 元数据仍在内存；
commit 时 iterator 可能边输出边 restore change。
```

## 12. 成本、资源与跨模块传播
ReorderBuffer 的成本随几个变量扩张。

第一，活跃事务数。

每个 logical-relevant XID 都可能有 `ReorderBufferTXN`。

活跃事务越多，`by_txn` 越大，`toplevel_by_lsn` 越长，slot restart_lsn 越容易被老事务拖住。

第二，单事务 change 数。

每条 row change 至少有 `ReorderBufferChange`。

INSERT/UPDATE/DELETE 还会有 tuple buffer。

`txn->nentries`、`txn->size`、`rb->size` 随行数增长。

第三，subxact 数。

提交时 iterator 要为 top + non-empty subxacts 建 heap。

成本近似是：

```text
changes * log(number_of_non_empty_streams)
```

大量 savepoint 或 PL/pgSQL exception block 会放大这部分成本。

第四，catalog change。

catalog change 带来 snapshot、tuple CID、invalidations 和 syscache churn。

后续 change 可能需要切换 snapshot 或推进 command id。

第五，内存限制和 spill。

spill 会把内存压力传播成磁盘 I/O。

长事务还会拖住 `restart_lsn` 和 `catalog_xmin`，进一步影响 WAL 保留和 VACUUM 清理。

相邻模块边界是：

```text
decode.c:
  把 WAL record 变成 logical change 或 transaction boundary。

snapbuild.c:
  决定 change 是否可解码，以及使用哪个 historic snapshot。

logical.c:
  建立 context、output callback wrapper、统计更新和消费位置。

replication slot:
  持久化 restart_lsn / confirmed_flush_lsn / catalog_xmin。

xact.c / xactdesc.c:
  产生和解析 commit、abort、assignment、subxacts。
```

## 13. 观测与诊断入口
第一个 runtime truth 是提交顺序输出。用 `test_decoding` 创建 slot 后，让会话 A 先 `BEGIN; INSERT id=1;` 但不提交；会话 B 后写 `id=2` 并 `COMMIT`。读取 `pg_logical_slot_get_changes()` 时应先看到 B，再在 A 提交后看到 A。这个现象对应源码边界：`DecodeInsert()` 只 `ReorderBufferQueueChange()`，`DecodeCommit()` 才 `ReorderBufferCommit()`。

第二个 runtime truth 是 subxact 重组。用 savepoint 写出：

```text
top change
subxact change
top change
```

output plugin 看到一个顶层事务中的多条 change；源码里这些 change 可能属于不同 `ReorderBufferTXN`，提交时由 `ReorderBufferIterTXNNext()` 按 LSN merge。

第三个 runtime truth 是资源压力。当前源码的 `pg_stat_replication_slots` 暴露：

```text
spill_txns / spill_count / spill_bytes
stream_txns / stream_count / stream_bytes
mem_exceeded_count / total_txns / total_bytes
```

这些统计由 `logical.c` 的 `UpdateDecodingStats()` 从 `ReorderBuffer` 字段上报。`mem_exceeded_count`、`spill_count`、`spill_bytes` 增长，说明进入 reorder buffer 内存压力路径；但它不能告诉你具体哪个 `ReorderBufferTXN` 或 subxact 正在占内存。

`pg_waldump --rmgr=XACT` 能对照 commit、abort、assignment record。`xactdesc.c` 中的 `xact_desc_subxacts()` 和 `xact_desc_assignment()` 是相关文本来源。

建议断点：

```text
LogicalDecodingProcessRecord
ReorderBufferAssignChild
ReorderBufferQueueChange
DecodeCommit
ReorderBufferCommitChild
ReorderBufferCommit
ReorderBufferProcessTXN
ReorderBufferIterTXNNext
ReorderBufferAbort
ReorderBufferCleanupTXN
```

观察字段：

```text
XLogRecGetXid(record)
XLogRecGetTopXid(record)
buf.origptr
parsed->nsubxacts
txn->xid / first_lsn / base_snapshot
change->lsn / action
```

诊断时要区分三类状态：slot 输出顺序、`pg_stat_replication_slots` 和 `pg_waldump` 是直接可见；subxact 是否已被纠正为 child、restart_lsn 是否被最老 in-progress txn 拖住只能推断；iterator heap、完整 `txn->changes` 链和 internal snapshot 切换点基本只能靠断点看。

## 14. 常见误区
误区一：WAL 顺序就是 logical 输出顺序。正确模型是 WAL 顺序负责输入，commit record 顺序决定事务输出，事务内部再按 change LSN merge。

误区二：`ReorderBufferTXN` 只表示顶层事务。它也表示 subtransaction，是否 top-level 要看 `toptxn`、`toplevel_xid` 和 `RBTXN_IS_SUBXACT` 的生命周期状态。

误区三：subxact commit 时把 change 搬到父事务尾部就行。这样会破坏 LSN 顺序；PostgreSQL 保留多个局部有序链，提交时用 heap merge。

误区四：没有输出的 commit 可以当 abort 清理。跳过 committed transaction 仍可能需要 catalog invalidation，所以有 `ReorderBufferForget()`。

误区五：output plugin callback 可以随便访问数据库。callback 在 decoding 内部事务和 historic snapshot 下运行，不能分配 XID，也不能把 decoder 的内部事务当成业务写事务。

误区六：`pg_stat_replication_slots` 能解释所有 reorder buffer 状态。它只能给累计统计，subxact 关系、change 链和 snapshot 选择通常只能从源码断点或 WAL record 推断。

## 15. 课堂实验
实验一：创建 `test_decoding` slot，两个会话制造“A 先写但 B 先提交”。读取 slot 应先看到 B，提交 A 后才看到 A。回到源码验证 `DecodeInsert()` 只入队，`DecodeCommit()` 才输出。

实验二：在一个事务中使用 savepoint 写三条 row：top、subxact、top。断 `ReorderBufferIterTXNNext()`，记录每次 `change->lsn` 和 `change->txn->xid`，确认 change 可能来自不同 XID，但输出 LSN 非递减。

实验三：批量 insert 后 rollback。断 `DecodeAbort()`、`ReorderBufferAbort()`、`ReorderBufferCleanupTXN()`、`ReorderBufferFreeChange()`，观察 `txn->changes` 释放、`by_txn` entry 删除、`base_snapshot` refcount 下降和 spill 文件清理。

实验四：降低 `logical_decoding_work_mem`，制造大事务并消费 slot，查询：

```sql
SELECT slot_name, spill_count, spill_bytes, mem_exceeded_count, total_txns, total_bytes
FROM pg_stat_replication_slots
WHERE slot_name = 'rb_demo_slot';
```

`spill` / `mem_exceeded` 增长说明进入内存压力路径，但不说明具体哪个 change 或 subxact 在内存里。

## 16. 讨论题
1. 为什么 logical decoding 不能在 `DecodeInsert()` 中直接调用 output plugin 的 change callback？
2. `ReorderBufferTXNByXid()` 为什么允许先把一个事务创建为 top-level，再由 `ReorderBufferAssignChild()` 改成 subtransaction？
3. 如果把 subtransaction 的 changes 直接 append 到父事务 `changes` 尾部，会在哪种 WAL 顺序下输出错误？
4. `ReorderBufferCommit()` 中 `txn == NULL` 为什么不是错误？
5. `base_snapshot == NULL` 的 committed transaction 为什么可以 cleanup 后不输出？
6. `ReorderBufferForget()` 与 `ReorderBufferAbort()` 的核心语义差异是什么？
7. 为什么 output plugin callback 结束后要 `AbortCurrentTransaction()`？
8. `pg_stat_replication_slots.spill_bytes` 增长能说明什么，不能说明什么？

## 17. 本节小结
本节核心链路是：

```text
LogicalDecodingProcessRecord()
  -> rmgr decode
  -> ReorderBufferQueueChange()
  -> ReorderBufferAssignChild()
  -> DecodeCommit()
  -> ReorderBufferCommitChild()
  -> ReorderBufferCommit()
  -> ReorderBufferProcessTXN()
  -> output plugin callbacks
  -> ReorderBufferCleanupTXN()
```

核心状态是 backend-local 的 `ReorderBuffer`。`by_txn` 用于按 XID 聚合，`toplevel_by_lsn` 和各事务 `changes` 链提供 LSN 视角；`ReorderBufferTXN` 保存事务、subxact、snapshot、invalidation、tuplecid 和内存统计；`ReorderBufferChange` 保存单条 logical change 或内部控制事件。

关键不变量是：WAL record 顺序是输入顺序；commit record 顺序是事务输出顺序；事务内部要把 top/subxact 多条 change stream 按 LSN merge；abort 和 skip commit 必须分别 cleanup 和 invalidation；historic snapshot 决定 catalog 解释边界。

异常路径中，abort record 会清理事务状态，`ReorderBufferAbortOld()` 会处理 crash/restart 后不再运行的旧事务，`PG_CATCH()` 会清理 iterator、historic snapshot、内部事务和 invalidation，再按场景 cleanup 或重抛。

观测上，可以用 `test_decoding` 看提交顺序输出，用 `pg_waldump --rmgr=XACT` 对照 assignment/commit/abort record，用 `pg_stat_replication_slots` 观察 spill 和 total 统计；但 change 链、subxact 迁移和 iterator heap 通常不能直接从 SQL 视图看到。

可迁移的系统规律是：

```text
当输入日志的顺序不是外部语义的交付顺序时，
系统需要一个中间状态同时保存 identity 视角和 order 视角；
identity 视角负责归属，
order 视角负责边界到达后的稳定输出。
```

在 ReorderBuffer 中，这两个视角就是 `XID -> ReorderBufferTXN` 和 `commit LSN / change LSN -> output ordering`。下一节会把焦点从“如何重组”推进到“重组过程中内存不够怎么办”。
