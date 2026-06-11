# PostgreSQL Reorder Buffer cleanup 与错误恢复

## 课程定位

前置知识：
已经理解 replication slot 的 `restart_lsn`、`confirmed_flush`、`catalog_xmin`。
已经理解 logical decoding 会从 WAL 中重建事务，再交给 output plugin 输出。
已经理解上一组 reorder buffer 课程中的事务重组、spill、subxact 和 TOAST 组装。

本节唯一主问题：

```text
消费者断开、slot 释放、事务 abort 或 decoding ERROR 时，
reorder buffer 中的内存、spill 文件和事务状态如何清理，
哪些状态必须保留给下次继续解码？
```

核心矛盾：
logical decoding 的运行态必须非常容易丢弃。
连接断了，backend ERROR 了，output plugin 报错了，本次解码会话中的 `ReorderBufferTXN`、tuple image、iterator、snapshot 和 open fd 都不能泄漏。
但 logical slot 的进度不能随会话一起丢。
`confirmed_flush`、`restart_lsn`、`catalog_xmin` 和候选推进状态决定下次从哪里读 WAL、哪些 WAL 不能回收、哪些 catalog tuple 不能被 VACUUM 清走。
PostgreSQL 的选择是把本次会话的 reorder buffer 做成 backend-local、可重建、可批量清理的状态，把跨会话正确性放在 replication slot 中。

学完后应能判断：
一个大事务 spill 到磁盘后，正常 commit、abort、consumer disconnect、walsender ERROR 分别由哪条路径删除文件。
`ReplicationSlotRelease()` 为什么不能清掉 `restart_lsn` 或 `confirmed_flush`。
output plugin 在 change/commit callback 中 ERROR 后，为什么下次仍必须从 slot 的 `restart_lsn` 读 WAL。
普通 aborted transaction 为什么不会输出为 commit 事件。
为什么有些 cleanup 是立即删除，有些 cleanup 是下一次使用同一 slot 时补偿完成。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

第 20 节回答的是 reorder buffer 为什么存在：

```text
WAL 按 record 顺序到来；
logical output 必须按事务语义输出；
reorder buffer 把 record 重组成 ReorderBufferTXN。
```

第 21 节回答的是内存压力：

```text
大事务不能无限占用 backend-local memory；
logical_decoding_work_mem 触发 streaming 或 spill；
spill 文件把 change 临时放到 pg_replslot/<slot>/。
```

第 22 节回答的是语义组装：

```text
subxact、TOAST、tuplecid、snapshot 和 invalidation
共同保证 output plugin 看到的是完整事务语义。
```

这一节只看收尾。
收尾不是附属逻辑。
logical decoding 的 correctness 很大一部分来自“哪些东西能丢，哪些东西不能丢”。

如果把所有状态都留到 slot 中，slot 会变成巨大的持久事务缓存。
这会让 crash recovery、slot 保存、WAL 回收和 catalog horizon 都变得不可控。
如果把所有状态都当成本地内存，消费者断开后下次无法知道哪里继续。
PostgreSQL 在这两者之间画了一条硬边界：

```text
ReorderBuffer / LogicalDecodingContext:
  本次 backend 解码会话的可重建运行态。

ReplicationSlot:
  跨会话、跨 ERROR、跨 reconnect 必须保留的解码边界。
```

这也是本节的主线：

```text
问题
  -> 哪些状态存在
  -> 正常输出后如何清理
  -> abort 如何清理
  -> disconnect / ERROR 如何兜底
  -> 下次为什么从 restart_lsn 继续
  -> 如何诊断遗留内存、spill 文件和 slot 进度
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
walsender 或 SQL logical decoding 先 acquire logical slot，
CreateDecodingContext() 在本地创建 LogicalDecodingContext、SnapBuild 和 ReorderBuffer，
decoder 从 slot->data.restart_lsn 读 WAL，
只把消费者确认过的输出推进到 slot->data.confirmed_flush，
只在确认覆盖 candidate_restart_valid 后推进 slot->data.restart_lsn，
事务 commit/abort 或 ERROR 时清掉本地 ReorderBufferTXN、change 和 spill 文件，
slot release 只解除 active ownership，不清掉下次继续解码所需的 LSN 边界。
```

这里有两种完全不同的“清理”。

第一种是对象生命周期清理。
它释放内存、关闭 fd、删除 `xid-*.spill` 文件、从 hash/list 中移除事务。
这些对象可以从 WAL 再构造出来。

第二种是语义边界清理。
它不能随便发生。
`confirmed_flush` 只有在消费者确认 flush 后才能前进。
`restart_lsn` 只有在 decoder 知道更早 WAL 不再需要，并且消费者确认覆盖对应位置后才能前进。
`catalog_xmin` 只有在新的 horizon 已经安全写入 slot 后才能放宽。

本节的 tension 可以压缩成：

```text
本地运行态要积极清理，避免 backend 泄漏；
slot 进度要保守保留，避免下次跳过未确认或仍需重建的 WAL。
```

这个区别解释了很多现象。
消费者断开后，`pg_replication_slots.active` 会变成 false。
但 `restart_lsn` 仍然可能很旧。
大事务 abort 后，output plugin 看不到 commit。
但如果事务曾经被 streaming 输出过，reorder buffer 可能还要发 stream abort。
decoding ERROR 后，本地 memory context 会被清掉。
但 slot 仍保留旧的 `confirmed_flush`，下次可能重复输出上次未确认的变更。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/reorderbuffer.h` | `ReorderBuffer`、`ReorderBufferTXN`、txn flags、memory/spill 统计字段。 |
| 2 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferFree()`、`ReorderBufferCleanupTXN()`、`ReorderBufferAbort()`、spill serialize/restore cleanup。 |
| 3 | `src/backend/replication/logical/logical.c` | `CreateDecodingContext()`、`FreeDecodingContext()`、`LogicalConfirmReceivedLocation()`、candidate LSN 推进。 |
| 4 | `src/backend/replication/logical/decode.c` | commit/abort WAL record 如何调用 reorder buffer，哪些事务被 skip/forget/abort。 |
| 5 | `src/backend/replication/slot.c` | `ReplicationSlotRelease()`、temporary slot cleanup、checkpoint 保存 `confirmed_flush` / `restart_lsn`。 |
| 6 | `src/backend/replication/walsender.c` | logical streaming 主循环、consumer disconnect、`WalSndErrorCleanup()`。 |
| 7 | `src/backend/storage/file/fd.c` | spill I/O 使用的 transient fd 在 ERROR/subxact abort/backend exit 时如何关闭。 |

推荐阅读顺序不是从 `decode.c` 顶部开始。
先读状态结构。
然后读清理函数。
再读谁调用清理函数。
最后回到 slot 进度。

可以按下面顺序跟源码：

```text
ReorderBufferAllocate()
  -> ReorderBufferCleanupSerializedTXNs()
  -> CreateDecodingContext()
  -> XLogBeginRead(... restart_lsn ...)
  -> LogicalDecodingProcessRecord()
  -> DecodeCommit() / DecodeAbort()
  -> ReorderBufferCommit() / ReorderBufferAbort()
  -> ReorderBufferCleanupTXN()
  -> LogicalConfirmReceivedLocation()
  -> ReplicationSlotRelease()
```

如果从 `pg_replication_slots` 开始读，很容易把 `active = false` 误解成 slot 已经释放资源。
正确入口是 ownership：

```text
谁持有本次解码对象？
谁持有跨会话进度？
ERROR 后哪个 cleanup 链能找到它？
下次重建时从哪里开始读 WAL？
```

## 4. 状态边界：什么能丢，什么必须保留

### 4.1 `LogicalDecodingContext` 是本次解码会话

`logical.c` 的 `StartupDecodingContext()` 创建 `"Logical decoding context"` memory context。
在这个 context 中创建 `LogicalDecodingContext`。
它持有：

```text
ctx->slot
ctx->reader
ctx->reorder
ctx->snapshot_builder
ctx->callbacks
ctx->out
ctx->prepare_write / write / update_progress
ctx->fast_forward / streaming / twophase flags
```

这些字段服务当前 backend。
它们不是 slot 的持久状态。
连接断开或 ERROR 后，新的 backend 会重新创建一套。

`FreeDecodingContext()` 的正常释放顺序是：

```text
shutdown_cb_wrapper(ctx)
ReorderBufferFree(ctx->reorder)
FreeSnapshotBuilder(ctx->snapshot_builder)
XLogReaderFree(ctx->reader)
MemoryContextDelete(ctx->context)
```

这条顺序说明 output plugin 有一次 shutdown 机会。
然后 reorder buffer 被整体释放。
snapshot builder 和 WAL reader 也随之释放。
最后删除整个 decoding context。

但这不是唯一 cleanup 路径。
如果 ERROR 从外层直接跳走，未必能走到 `FreeDecodingContext()`。
因此本课后面会看到补偿清理：
fd.c 兜底关 fd。
slot exit callback 兜底 release slot。
`ReorderBufferAllocate()` 和 `StartupReorderBuffer()` 兜底删旧 spill 文件。

### 4.2 `ReorderBuffer` 是事务重组的本地 owner

`ReorderBuffer` 定义在 `reorderbuffer.h`。
本节只关心三组状态。
第一组是 lookup 和排序结构：
`by_txn`、`toplevel_by_lsn`、`txns_by_base_snapshot_lsn`、`catchange_txns`。
它们让 decoder 能按 XID 找事务、按 LSN 找最早事务、按 snapshot LSN 推进 restart 边界。
第二组是 owner：
`change_context`、`txn_context`、`tup_context`。
它们让 change、transaction shell 和 tuple image 有清楚的本地生命周期。
第三组是 pressure 和 callback：
`size`、`txn_heap`、spill/stream 统计和 `private_data`。
这些字段决定何时 spill/stream，并把 output plugin 回调接回 `LogicalDecodingContext`。

`ReorderBufferAllocate()` 会创建独立 memory context：

```text
"ReorderBuffer"
  -> "Change"
  -> "TXN"
  -> "Tuples"
```

这不是为了语义隔离。
这是为了让内存生命周期和统计更清楚。
普通 change、txn 对象和 tuple image 的分配模式不同。
错误路径也可以把整个 context 删除。

### 4.3 `ReorderBufferTXN` 是一个可丢弃的事务外壳

`ReorderBufferTXN` 的字段也不能单独解释。
`txn_flags` 表示是否 serialized、streamed、prepared、committed 或 aborted。
`xid`、`toplevel_xid`、`toptxn` 描述 subtransaction 关系。
`first_lsn`、`final_lsn`、`end_lsn` 和 `restart_decoding_lsn` 把事务 shell 绑定到 WAL 时间线。
`base_snapshot`、`snapshot_now`、`tuplecids`、`toast_hash` 和 invalidation 数组则服务 historic MVCC、TOAST 重组和 syscache 正确性。
一个 `ReorderBufferTXN` 可能还在进行中，可能已经 spill，可能已经 streamed 一部分，可能已经知道 aborted 但还没读到最终 ABORT record，也可能是 prepared transaction。
所以 cleanup 不能只有一个函数：
有的路径彻底删除事务，有的路径只删 changes 并保留事务外壳，有的路径还要先发 stream abort。

### 4.4 Slot 状态是跨会话保留边界

`ReplicationSlot` 在 shared memory 中。
它的 `data` 部分会写入 `pg_replslot/<slot>/state`。
本节最关心 `data.restart_lsn`、`data.confirmed_flush`、`data.catalog_xmin`、`effective_catalog_xmin`、candidate LSN/xmin、`active_proc`、`persistency`、`last_saved_restart_lsn` 和 `last_saved_confirmed_flush`。

`active_proc` 表示当前哪个 backend 持有 slot。
它不是“slot 是否仍然保护 WAL”的开关。
consumer disconnect 后，persistent slot 的 `active_proc` 会清掉。
但 `restart_lsn` 仍参与 WAL 保留。
`catalog_xmin` 仍可能保护 catalog tuple。
`confirmed_flush` 仍决定下次 consumer 的逻辑进度。

这条边界非常重要：

```text
释放当前持有者
  != 删除 slot
  != 确认消费者已经接收
  != 放宽 WAL 保留
  != 放宽 catalog xmin
```

## 5. 主流程一：正常 logical streaming 会话如何创建和释放

先看 walsender 走 logical replication protocol 的主线。

`StartLogicalReplication()` 在 `walsender.c` 中：

```text
CheckLogicalDecodingRequirements(false)
ReplicationSlotAcquire(cmd->slotname, true, true)
CreateDecodingContext(cmd->startpoint, ...)
XLogBeginRead(ctx->reader, MyReplicationSlot->data.restart_lsn)
sentPtr = MyReplicationSlot->data.confirmed_flush
WalSndLoop(XLogSendLogical)
FreeDecodingContext(logical_decoding_ctx)
ReplicationSlotRelease()
```

这里有两个 LSN 同时出现。
读 WAL 从 `restart_lsn` 开始。
对外认为已经发送过的逻辑边界从 `confirmed_flush` 开始。

这不是重复。
logical decoding 需要从较早 WAL 重建事务和 snapshot。
但 output plugin 不应该把 `confirmed_flush` 之前已经确认的事务再次作为新事务交给消费者。
`CreateDecodingContext()` 的注释也明确区分：

```text
start_lsn:
  decoding context 的基准消费位置，默认使用 confirmed_flush。

XLogBeginRead(... restart_lsn ...):
  WAL reader 的真实读取起点。
```

`XLogSendLogical()` 每次读一条 WAL record。
如果 `XLogReadRecord()` 返回记录，就调用：

```text
LogicalDecodingProcessRecord(logical_decoding_ctx, reader)
sentPtr = reader->EndRecPtr
```

真正写网络的是 output plugin wrapper 调用的 writer。
对 walsender，writer 是：

```text
WalSndPrepareWrite()
WalSndWriteData()
WalSndUpdateProgress()
```

`WalSndWriteData()` 只把数据放进 CopyData 并尽力 flush。
它不代表消费者已经持久化。
消费者确认来自 standby status update 里的 `flushPtr`。
`ProcessStandbyReplyMessage()` 收到 `flushPtr` 后：

```text
if logical slot:
  LogicalConfirmReceivedLocation(flushPtr)
```

所以正常会话的进度推进是异步的：

```text
decoder 读到某个 WAL record
  -> output plugin 生成消息
  -> walsender 写到 socket
  -> consumer 处理并回报 flushPtr
  -> LogicalConfirmReceivedLocation(flushPtr)
  -> confirmed_flush 可能前进
  -> candidate_restart_lsn 可能转正为 restart_lsn
```

正常退出时，`WalSndLoop()` 返回。
`StartLogicalReplication()` 走到：

```text
FreeDecodingContext(logical_decoding_ctx)
ReplicationSlotRelease()
```

这会释放本地解码状态。
但 slot 中的确认位置保留。
这就是正常 disconnect / stop 之后仍可继续消费的原因。

## 6. 主流程二：commit 后 `ReorderBufferCleanupTXN()` 如何彻底移除事务

普通 commit 主线在 `decode.c`：

```text
DecodeCommit()
  -> SnapBuildCommitTxn()
  -> DecodeTXNNeedSkip()
  -> ReorderBufferCommitChild() for surviving subxacts
  -> ReorderBufferCommit()
     -> ReorderBufferReplay()
        -> ReorderBufferProcessTXN()
           -> output plugin begin/change/commit
           -> ReorderBufferCleanupTXN()
```

`ReorderBufferCleanupTXN()` 是彻底删除一个事务的核心。
它通常在事务已经 commit 输出完成、abort 完成、forget 完成或 ERROR 需要丢弃时调用。

它先清 subtransactions。
reorder buffer 把 subtransactions 挂在 top transaction 下。
cleanup 会遍历 `txn->subtxns`。
对每个 subtxn 递归调用 `ReorderBufferCleanupTXN()`。
源码注释强调 subtransactions 总是直接关联到 top-level TXN。
因此这里不会无限递归多层。

然后清 `txn->changes`。
每个 `ReorderBufferChange` 可能带 tuple、message、snapshot、invalidation 或 truncate relids。
`ReorderBufferFreeChange()` 会按 action 释放内部对象。
例如：

```text
INSERT / UPDATE / DELETE:
  free oldtuple / newtuple

MESSAGE:
  free prefix / message

INTERNAL_SNAPSHOT:
  ReorderBufferFreeSnap()

TRUNCATE:
  ReorderBufferFreeRelids()
```

这里有一个性能细节。
`ReorderBufferCleanupTXN()` 不对每个 change 都更新 memory accounting。
它先累计 `mem_freed`。
最后调用一次 `ReorderBufferChangeMemoryUpdate()`。
这是为了减少维护 max-heap 的成本。

接着清 tuplecid。
tuplecid 用来支持 catalog-changing transaction 的 historic MVCC。
它们总是放在 top-level transaction。
事务结束后，tuplecid change 也要释放。

然后清 snapshot。
如果 `base_snapshot` 存在：

```text
SnapBuildSnapDecRefcount(txn->base_snapshot)
dlist_delete(&txn->base_snapshot_node)
```

如果 `snapshot_now` 存在，说明事务曾经被 streaming。
cleanup 用 `ReorderBufferFreeSnap()` 删除它。

然后从所有索引结构移除事务：

```text
dlist_delete(&txn->node)
if catalog changes:
  dclist_delete_from(&rb->catchange_txns, &txn->catchange_node)
hash_search(rb->by_txn, &txn->xid, HASH_REMOVE, &found)
```

最后处理 spill 文件。
如果 `rbtxn_is_serialized(txn)`：

```text
ReorderBufferRestoreCleanup(rb, txn)
```

`ReorderBufferRestoreCleanup()` 根据 `txn->first_lsn` 到 `txn->final_lsn` 覆盖的 WAL segment 生成 spill 文件名并逐个 `unlink()`。
如果文件不存在，`ENOENT` 被允许。
如果其他错误，抛 ERROR。

最后 `ReorderBufferFreeTXN()` 释放事务外壳。
它会清掉 lookup cache、`gid`、`tuplecid_hash`、invalidations、distributed invalidations、TOAST hash，然后 `pfree(txn)`。

彻底 cleanup 后，reorder buffer 中不再能通过 XID 找到这个事务。
内存 accounting 回落。
spill 文件删除。
slot 进度是否推进则另走 `LogicalConfirmReceivedLocation()`。
这两个动作不要混在一起。

## 7. 主流程三：abort 为什么不输出普通事务

abort record 主线在 `decode.c`：

```text
DecodeAbort()
  -> DecodeTXNNeedSkip()
  -> for each subxact:
       ReorderBufferAbort()
  -> ReorderBufferAbort(top xid)
  -> UpdateDecodingStats()
```

普通事务 abort 不会走 `ReorderBufferCommit()`。
也不会调用 output plugin 的 commit callback。
它只需要丢弃已缓存的 changes。

`ReorderBufferAbort()` 先查 XID：

```text
txn = ReorderBufferTXNByXid(..., create=false)
if txn == NULL:
  return
```

如果 reorder buffer 里从未为这个 XID 积累过可解码内容，就没有东西要清。
这是常见路径，不是异常。

如果事务曾经被 streaming 输出过，情况更复杂。
下游可能已经看到该事务的一部分 stream。
此时 `ReorderBufferAbort()` 会调用：

```text
rb->stream_abort(rb, txn, lsn)
```

然后如果该事务有 invalidations，还会执行 immediate invalidation。
原因是事务内的 DDL/catalog 访问可能已经污染了本地 syscache。
后续事务不能用这些错误 cache entries。

最后它设置 `txn->final_lsn = lsn`。
然后调用 `ReorderBufferCleanupTXN()`。
因此 abort 的最终清理和 commit 后清理共享同一个彻底释放函数。
差异在进入 cleanup 前是否输出过、是否需要 stream abort、是否要 immediate invalidation。

这解释了本节第一个运行时不变量：

```text
普通 aborted transaction 不会输出 commit/change 语义；
它的 reorder buffer 状态被丢弃；
如果它已经通过 streaming API 暴露给下游，必须输出 stream abort 来让下游回滚已见片段。
```

## 8. Truncate 与 Reset：为什么有时不能立刻删掉 TXN

当前源码没有公开的 `ReorderBufferReset()`。
本节讨论的 reset 边界主要是 `ReorderBufferTruncateTXN()`、`ReorderBufferResetTXN()` 和 `ResetLogicalStreamingState()`。
`ReorderBufferTruncateTXN()` 和 `ReorderBufferCleanupTXN()` 的差异是本节重点：
cleanup 表示事务对象结束，truncate 表示 changes 可以丢，但事务外壳还要保留。
源码注释明确说它会 discard changes，同时保留 transaction、tuplecids、invalidations 和 snapshots。
它服务大事务已 streaming 一轮、prepared transaction 已在 PREPARE 解码、memory pressure 前发现事务已 abort、以及 streaming/prepared decoding 中检测到 concurrent abort 这些场景。

truncate 会递归处理 subtxns，删除 `txn->changes`，批量更新 memory accounting。
如果 `txn_prepared = true`，还会清 tuplecids。
如果事务已经 serialized 到磁盘，它会 `ReorderBufferRestoreCleanup()` 删除 spill 文件，清掉 `RBTXN_IS_SERIALIZED`，设置 `RBTXN_IS_SERIALIZED_CLEAR`，并把 `nentries` / `nentries_mem` 归零。
但它不从 `by_txn` hash 删除事务，也不释放 `ReorderBufferTXN`。
这就是 reset 的核心语义：
清掉本轮已输出或已无用的 change，保留后续 commit/abort/prepared 边界还要用的身份和元数据。

`ReorderBufferResetTXN()` 更窄。
它用于 `ReorderBufferProcessTXN()` 捕获 concurrent abort。
当 callback 或 catalog scan 抛出 `ERRCODE_TRANSACTION_ROLLBACK`，catch 块会标记事务 aborted，然后调用 `ReorderBufferResetTXN(rb, txn, snapshot_now, command_id, prev_lsn, specinsert)`。
这个函数会 truncate txn、reset TOAST、释放 speculative insert change；如果事务已经 streamed，还会 `stream_stop()` 并保存当前 snapshot / command id。
它不是结束事务，而是停止处理这轮不再安全的 changes，等待真正的 ABORT 或 ROLLBACK PREPARED record 完成语义收尾。

`ResetLogicalStreamingState()` 只清 `CheckXidAlive` 和 `bsysscan`。
它不释放 reorder buffer 内存，也不推进 slot 进度。
所以不要把 reset 理解成统一 API。
在这个模块里，reset 分三层：

```text
change 层:
  TruncateTXN 丢掉 changes。

transaction 层:
  ResetTXN 保留事务外壳以继续处理 abort/prepared 边界。

logical streaming 检测层:
  ResetLogicalStreamingState 清全局检测变量。

context 层:
  FreeDecodingContext / MemoryContextDelete 丢掉整次解码会话。
```

## 9. Spill 文件的生命周期

reorder buffer 的 spill 文件不是 PostgreSQL 通用 temp file。
它们是 slot 目录下有名字的文件。
路径由 `ReorderBufferSerializedPath()` 生成：

```text
pg_replslot/<slot>/xid-<xid>-lsn-<hi>-<lo>.spill
```

文件名中没有 timeline。
源码注释说明它们只在单次运行中使用。
同一 LSN 在该运行中只对应一个 WAL record。

写入主线在 `ReorderBufferSerializeTXN()`。
当 `logical_decoding_work_mem` 被超过，`ReorderBufferCheckMemoryLimit()` 选择 largest transaction。
如果不能 streaming，就 serialize。

serialize 过程是：

```text
遍历 subtxns 并递归 serialize
遍历 txn->changes
  根据 change->lsn 所在 WAL segment 选择 spill file
  OpenTransientFile(path, O_CREAT | O_WRONLY | O_APPEND | PG_BINARY)
  ReorderBufferSerializeChange()
  dlist_delete(change)
  ReorderBufferFreeChange(change, false)
更新 memory accounting
txn->nentries_mem = 0
txn->txn_flags |= RBTXN_IS_SERIALIZED
CloseTransientFile(fd)
```

这里的 fd 是 `OpenTransientFile()` 返回的 raw fd。
`fd.c` 会把它放进 `allocatedDescs`。
正常路径由 `CloseTransientFile()` 关闭。
如果 ERROR 跳出，`AtEOSubXact_Files()`、`AtEOXact_Files()` 或 process exit cleanup 会关闭遗留 descriptor。

但关闭 fd 不等于删除 spill 文件。
这些文件是有名字的 slot 文件。
删除由 reorder buffer 自己负责。

事务正常结束后：

```text
ReorderBufferCleanupTXN()
  -> if RBTXN_IS_SERIALIZED:
       ReorderBufferRestoreCleanup()
```

`ReorderBufferRestoreCleanup()` 会从 `first_lsn` 到 `final_lsn` 的 segment 范围生成可能文件名并 `unlink()`。
这覆盖一个事务在多个 WAL segment 中 spill 的情况。

整次 decoding context 正常释放时：

```text
ReorderBufferFree()
  -> MemoryContextDelete(rb->context)
  -> ReorderBufferCleanupSerializedTXNs(slotname)
```

注意顺序。
它先删除 memory context。
再清 slot 目录中的 `xid*` spill 文件。
这说明 spill 文件不是 memory context 的孩子。
它们需要文件系统级清理。

还有两个补偿清理点。

第一个在 `ReorderBufferAllocate()` 开头。
创建新的 reorder buffer 后，会调用：

```text
ReorderBufferCleanupSerializedTXNs(NameStr(MyReplicationSlot->data.name))
```

注释说明原因：
如果 prior exit 避免调用 `ReorderBufferFree()`，旧 spill 文件可能留在 slot 目录。
下一次使用同一 slot 时必须清掉。
否则可能产生 duplicated txns。

第二个在 `StartupReorderBuffer()`。
服务器启动时扫描 `pg_replslot`。
对每个看起来像 slot 的目录，删除 `xid-*` spill 文件。
这是 crash / immediate restart 后的兜底。

因此 spill 文件的正确 mental model 是：

```text
spill 文件是本次解码会话的临时事务缓存；
正常事务结束时按事务删除；
正常 context free 时按 slot 删除；
异常退出后下一次使用 slot 或服务器启动时补偿删除；
它们不是 slot 跨会话进度的一部分。
```

## 10. `ReplicationSlotRelease()` 清什么，不清什么

`ReplicationSlotRelease()` 的注释很关键：

```text
This or another backend can re-acquire the slot later.
Resources this slot requires will be preserved.
```

这句话就是本节边界。
release 表示当前 backend 不再持有 slot。
它不表示消费者已经确认所有输出。
也不表示 WAL 可以回收。
也不表示 slot 可以删除。

对 persistent slot，release 的核心动作是：

```text
清 temporary effective_xmin if needed
slot->active_proc = INVALID_PROC_NUMBER
ReplicationSlotSetInactiveSince(...)
ConditionVariableBroadcast(&slot->active_cv)
MyReplicationSlot = NULL
清 MyProc 的 PROC_IN_LOGICAL_DECODING
```

`active_proc` 清掉后，别的 backend 可以 acquire 这个 slot。
`inactive_since` 让诊断和 idle timeout 有时间依据。
`PROC_IN_LOGICAL_DECODING` 清掉后，ProcArray 不再把当前进程当作 logical decoding backend。

但这些字段不会被清：

```text
slot->data.restart_lsn
slot->data.confirmed_flush
slot->data.catalog_xmin
slot->last_saved_restart_lsn
slot->last_saved_confirmed_flush
slot candidate fields unless already applied elsewhere
```

如果 slot 是 `RS_EPHEMERAL`，release 会 drop。
这是创建 persistent logical slot 时的过渡态。
失败时不能留下半初始化 persistent slot。

如果 slot 是 temporary，session cleanup 会 drop。
`ReplicationSlotCleanup(false)` 清理当前 session 的 temporary slots。

但常见 logical replication slot 是 persistent。
消费者断开后，它的保护边界必须保留。
否则 reconnect 会不知道从哪里继续。

## 11. `confirmed_flush` 与 `restart_lsn` 的保留边界

`confirmed_flush` 和 `restart_lsn` 经常被混淆。
本节只抓 cleanup 相关语义。

`confirmed_flush` 表示：

```text
consumer 已确认安全收到的 logical output 位置。
```

它由 `LogicalConfirmReceivedLocation(lsn)` 推进。
这个函数会阻止倒退：

```text
if lsn > data.confirmed_flush:
  data.confirmed_flush = lsn
```

如果消费者断开前没有发出更高的 `flushPtr`，server 不能猜它已经持久化。
因此 `confirmed_flush` 不前进。
下次 reconnect 可能重发未确认部分。
这是 logical replication 的 at-least-once 边界。

`restart_lsn` 表示：

```text
decoder 为了重建所有尚未安全跳过的事务，最早还需要读取的 WAL 位置。
```

它通常比 `confirmed_flush` 更老。
原因是 decoder 可能需要较早 WAL 中的 running-xacts、snapshot、subxact、catalog change 或未提交大事务信息。

`LogicalIncreaseRestartDecodingForSlot(current_lsn, restart_lsn)` 不会立刻把新值写进 `data.restart_lsn`。
它先设置候选：

```text
candidate_restart_valid = current_lsn
candidate_restart_lsn = restart_lsn
```

只有当消费者确认位置覆盖 `candidate_restart_valid`：

```text
LogicalConfirmReceivedLocation(lsn)
  if candidate_restart_valid <= lsn:
     data.restart_lsn = candidate_restart_lsn
     candidate_restart_lsn = Invalid
     candidate_restart_valid = Invalid
     ReplicationSlotMarkDirty()
     ReplicationSlotSave()
     ReplicationSlotsComputeRequiredLSN()
```

这条顺序保护的是 crash-safety 和 reconnect correctness。
如果先放宽 `restart_lsn`，但消费者还没有确认覆盖对应 output，crash 后可能删掉重放未确认事务所需的 WAL。

`catalog_xmin` 也类似。
候选 catalog xmin 只有在确认覆盖对应 LSN 后才能转成 `data.catalog_xmin`。
并且先写 slot 到磁盘，再更新 `effective_catalog_xmin`。
否则 crash 后可能不知道某些 catalog tuple 已经允许被清。

所以 cleanup 的第二个不变量是：

```text
本地 reorder buffer 可以被释放；
slot 的 confirmed_flush / restart_lsn 只能按确认和持久化边界前进；
release、disconnect、ERROR 都不能代替消费者确认。
```

## 12. Consumer disconnect：进程退出与下次继续

consumer 断开有几类表现。
客户端发 `CopyDone`，walsender 可以有序结束 copy mode。
socket 写失败或读取到 EOF，会走 `WalSndShutdown()`。
`WalSndShutdown()` 做的事很少：

```text
if whereToSendOutput == DestRemote:
  whereToSendOutput = DestNone
proc_exit(0)
```

这不是一个可恢复的普通 SQL ERROR。
它直接退出 walsender 进程。

如果 `WalSndLoop()` 有序返回，`StartLogicalReplication()` 会执行：

```text
FreeDecodingContext(logical_decoding_ctx)
ReplicationSlotRelease()
```

如果走 `proc_exit()`，进程退出回调兜底。
`ReplicationSlotInitialize()` 注册：

```text
before_shmem_exit(ReplicationSlotShmemExit, 0)
```

`ReplicationSlotShmemExit()` 做：

```text
if MyReplicationSlot != NULL:
  ReplicationSlotRelease()
ReplicationSlotCleanup(false)
```

因此 persistent slot 不会因为连接断开而保持 active。
temporary slot 会在 session cleanup 中删除。

memory context 随进程退出消失。
raw fd 也会被进程退出关闭。
但命名 spill 文件如果没有正常 `ReorderBufferFree()`，可能留在 `pg_replslot/<slot>/`。
这就是为什么 `ReorderBufferAllocate()` 和 `StartupReorderBuffer()` 都会清理 `xid-*` 文件。

下次消费者重连时：

```text
ReplicationSlotAcquire(slot)
CreateDecodingContext(startpoint or confirmed_flush)
XLogBeginRead(reader, slot->data.restart_lsn)
```

也就是说：

```text
逻辑消费位置从 confirmed_flush 判断；
WAL 读取起点从 restart_lsn 判断；
中间缺失的本地 reorder buffer 状态从 WAL 重建。
```

如果断开前 server 已经写出消息，但 consumer 没有确认 flush，`confirmed_flush` 不前进。
下次可能重新输出。
这不是 cleanup bug。
这是确认协议选择的正确结果。

## 13. Decoding ERROR：内存、资源与语义进度如何收尾

ERROR 路径要分两层看。

第一层在 `ReorderBufferProcessTXN()` 内部。
它处理 output plugin callback、catalog access、historic snapshot 和 relation open 等可能 ERROR 的代码。

函数进入时保存：

```text
MemoryContext ccxt = CurrentMemoryContext
ResourceOwner cowner = CurrentResourceOwner
```

然后建立 historic snapshot。
必要时启动 internal subtransaction。
随后迭代 transaction changes。

正常结束时，它会：

```text
TeardownHistoricSnapshot(false)
执行 invalidations
RollbackAndReleaseCurrentSubTransaction() if using_subtxn
ReorderBufferTruncateTXN() or ReorderBufferCleanupTXN()
```

catch 块更重要。
如果 callback 或 catalog access ERROR：

```text
MemoryContextSwitchTo(ccxt)
CopyErrorData()
ReorderBufferIterTXNFinish() if iterstate
TeardownHistoricSnapshot(true)
AbortCurrentTransaction()
执行 invalidations 或 InvalidateSystemCaches()
RollbackAndReleaseCurrentSubTransaction() if using_subtxn
```

然后分两种情况。

如果错误码是 `ERRCODE_TRANSACTION_ROLLBACK`，并且这是 streaming 未结束或 prepared decoding 场景：

```text
FlushErrorState()
txn->txn_flags |= RBTXN_IS_ABORTED
ReorderBufferResetTXN(...)
return gracefully
```

这是并发 abort 的特殊恢复。
decoder 不把它当成普通 ERROR 传播。
它清掉当前 changes，等待后续 abort/prepared 边界完成语义收尾。

其他 ERROR：

```text
ReorderBufferCleanupTXN(rb, txn)
MemoryContextSwitchTo(ecxt)
PG_RE_THROW()
```

这会把当前事务从 reorder buffer 中彻底清掉。
外层看到 ERROR。
slot 进度不会因为这个 ERROR 被当作确认。

第二层在 backend / walsender 顶层。
`postgres.c` 的 outer error recovery 会：

```text
error_context_stack = NULL
AbortCurrentTransaction()
if am_walsender:
  WalSndErrorCleanup()
PortalErrorCleanup()
if MyReplicationSlot != NULL:
  ReplicationSlotRelease()
ReplicationSlotCleanup(false)
MemoryContextSwitchTo(MessageContext)
FlushErrorState()
```

`WalSndErrorCleanup()` 针对 walsender 补充：

```text
LWLockReleaseAll()
ConditionVariableCancelSleep()
pgstat_report_wait_end()
pgaio_error_cleanup()
close xlogreader segment if open
ReplicationSlotRelease()
ReplicationSlotCleanup(false)
ReleaseAuxProcessResources(false) if no transaction
WalSndSetState(WALSNDSTATE_STARTUP)
```

这里要注意：
top-level ERROR cleanup 不是正常 `FreeDecodingContext()`。
它主要保证锁、wait、slot ownership、aux resources 和 transaction state 不泄漏。
decoding context 的 memory 会随 message/top-level memory cleanup 或进程退出被回收。
如果 `ReorderBufferFree()` 没有执行，spill 文件靠后续补偿清理。

fd.c 负责另一个边界。
`OpenTransientFile()` 打开的 raw fd 进入 `allocatedDescs`。
subtransaction abort 会通过 `AtEOSubXact_Files()` 关闭该 subtransaction 打开的 descriptors。
end-of-transaction 或 process exit 会通过 fd.c cleanup 关闭 remaining allocated descriptors。
所以 ERROR 后通常不会泄漏打开的 fd。

但 fd.c 不知道哪个 spill file 属于哪个 decoded transaction。
删除 `xid-*.spill` 仍然是 reorder buffer 的职责。

## 14. 观测与诊断入口

诊断时先分三类状态。

第一类能直接观测。
`pg_replication_slots` 可以看到：

```text
slot_name
active
active_pid
restart_lsn
confirmed_flush_lsn
catalog_xmin
xmin
wal_status
safe_wal_size
inactive_since
invalidation_reason
```

这些字段回答的是 slot 边界。
它们不显示 reorder buffer 当前有多少 `ReorderBufferTXN`。
也不显示某个事务有哪些 changes。

`pg_stat_replication_slots` 可以看到 logical slot 统计：

```text
spill_txns
spill_count
spill_bytes
stream_txns
stream_count
stream_bytes
mem_exceeded_count
total_txns
total_bytes
```

这些统计由 `UpdateDecodingStats()` 上报。
spill 或 stream 时会尽量更新，commit/prepare/abort 时也会更新。
但它们是累计统计。
不能直接证明当前还有哪个 spill 文件。

第二类只能间接观测。
`pg_replslot/<slot>/xid-*.spill` 文件能在文件系统看到。
但它们是内部临时实现细节。
看到文件存在，说明当前或上次解码会话曾经 spill。
看不到文件，不代表没有内存中的大事务。
ERROR 后短暂残留也可能在下次 allocate 时被清。

第三类基本不可见。
`ReorderBufferTXN` 的 `txn_flags`、`base_snapshot`、`tuplecid_hash`、`toast_hash`、`candidate_restart_valid` 的组合不通过 SQL 暴露。
需要 gdb、断点、DEBUG 日志或源码插桩。

一条实用诊断链是：

```text
消费者断开或 apply 停住
  -> pg_replication_slots.active / active_pid
  -> confirmed_flush_lsn 是否停住
  -> restart_lsn 与 confirmed_flush_lsn 差距
  -> pg_stat_replication_slots.spill_bytes / mem_exceeded_count
  -> pg_wal 增长和 wal_status
  -> server log 是否有 output plugin ERROR 或 walsender disconnect
  -> 必要时检查 pg_replslot/<slot>/xid-*.spill 是否反复出现
```

如果 `confirmed_flush_lsn` 不动，但 walsender 还 active，优先看 consumer 是否回报 flush。
如果 `confirmed_flush_lsn` 前进，但 `restart_lsn` 长期不动，优先怀疑仍有长事务、prepared / streaming 边界、catalog horizon 或 candidate 未被确认覆盖。
如果 slot inactive 且 `restart_lsn` 很旧，问题不是 reorder buffer 泄漏，而是消费者停了。
如果反复 output plugin ERROR，slot 进度通常不会前进，下次会从旧边界重建并可能重复输出。

日志入口：

```text
log_replication_commands:
  acquired/released logical replication slot

DEBUG2:
  spill ... changes in XID ... to disk
  UpdateDecodingStats ...

ERROR context:
  slot "<name>", output plugin "<plugin>", in the <callback> callback
```

最后这个 error context 来自 `logical.c` 的 wrapper。
它会把 callback 名称和相关 LSN 放进错误上下文。
定位 output plugin ERROR 时非常有用。

## 15. 常见误区

误区一：
把 `ReplicationSlotRelease()` 理解成 slot 清理。
它只是释放当前 backend 对 slot 的 active ownership。
persistent slot 的 `restart_lsn`、`confirmed_flush` 和保留边界继续存在。

误区二：
看到 `active = false` 就认为 WAL 可以回收。
只要 persistent slot 的 `restart_lsn` 还旧，WAL 保留压力仍然存在。

误区三：
把 `confirmed_flush_lsn` 当成 WAL reader 下次起点。
logical decoder 下次真实读取从 `restart_lsn` 开始。
`confirmed_flush` 是输出确认边界，不是重建所需 WAL 的最早位置。

误区四：
认为 abort 只是“不输出”。
普通 abort 不输出 commit。
但 streamed transaction 已经暴露过片段时，需要 stream abort。
committed but skipped transaction 还可能需要 invalidation。

误区五：
把 spill 文件当成持久队列。
`xid-*.spill` 是本次 decoding session 的临时缓存。
它能在 crash/ERROR 后残留，但会被 startup 或下次 allocate 清掉。
跨会话恢复靠 WAL 和 slot LSN，不靠 spill 文件。

误区六：
认为 output plugin ERROR 后 server 已经知道消费者没收到哪些消息。
server 只相信 `flushPtr` 确认。
ERROR 或 socket 断开不会自动推进 `confirmed_flush`。

## 16. 课堂实验

实验一跟读正常 commit。
在 `DecodeCommit()`、`ReorderBufferProcessTXN()`、`ReorderBufferCleanupTXN()`、`LogicalConfirmReceivedLocation()`、`ReplicationSlotRelease()` 设置断点。
用 `test_decoding` 创建 slot，执行一个 `BEGIN; INSERT; COMMIT;`。
要验证的顺序是：
commit record 先触发 output plugin begin/change/commit，随后 `ReorderBufferCleanupTXN()` 删除事务状态；只有 consumer reply 带来 flush LSN 后，`LogicalConfirmReceivedLocation()` 才推进 `confirmed_flush`，会话结束时 `ReplicationSlotRelease()` 只清 active ownership。

实验二观察大事务 spill 后 abort。
把 `logical_decoding_work_mem` 调小，制造足够大的事务后 `ROLLBACK`。
同时看：

```sql
SELECT * FROM pg_stat_replication_slots WHERE slot_name = '<slot>';
SELECT slot_name, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = '<slot>';
```

再观察 `pg_replslot/<slot>/xid-*.spill`。
预期是 spill 统计可能增加，abort record 到来后 `ReorderBufferAbort()` 调 `ReorderBufferCleanupTXN()` 清理事务，普通 abort 不输出 commit；短暂出现的 spill 文件最终由 transaction cleanup、context free 或补偿 cleanup 删除。

实验三模拟 output plugin ERROR。
让测试 plugin 在 change 或 commit callback 抛 ERROR，或临时在 wrapper 附近插入受控 `elog(ERROR, ...)`。
观察错误上下文是否包含 slot、plugin、callback 和 associated LSN。
再确认 slot 被 release，`confirmed_flush_lsn` 没有越过未确认位置，下次重新消费仍从 `restart_lsn` 读 WAL 并可能重发未确认事务。
这个实验验证的是：ERROR 清理本地资源，但不伪造消费者确认。

## 17. 讨论题

1. 为什么 `ReorderBufferFree()` 可以直接 `MemoryContextDelete()`，但仍必须额外清 spill 文件？
2. 为什么 `ReplicationSlotRelease()` 不应该清掉 persistent slot 的 `restart_lsn`？
3. 一个事务已经被 output plugin 写到 socket，但 consumer 没有确认 flush，ERROR 后下次是否应该重发？为什么？
4. `ReorderBufferAbort()` 和 `ReorderBufferForget()` 都会 cleanup，为什么不能合并？
5. 为什么 prepared transaction 在 prepare skip 时不能立刻删除 `ReorderBufferTXN`？
6. 如果看到 `restart_lsn` 长期落后，哪些证据能区分 consumer 不确认、大事务未结束、candidate 未应用和 slot 已 inactive？
7. `OpenTransientFile()` 的 fd cleanup 能解决什么问题，不能解决什么问题？
8. 为什么 spill 文件不适合作为 crash 后继续解码的持久队列？

## 18. 本节小结

本节的核心链路是：

```text
slot acquire
  -> CreateDecodingContext
  -> ReorderBuffer 从 restart_lsn 重建事务
  -> commit/abort/prepared/skip 决定 cleanup 方式
  -> consumer flush reply 推进 confirmed_flush
  -> candidate 覆盖后推进 restart_lsn
  -> context free 或 ERROR cleanup 释放本地状态
  -> slot release 保留跨会话边界
```

最重要的状态边界是：
`LogicalDecodingContext` 和 `ReorderBuffer` 是本次 backend 的可丢弃运行态。
`ReplicationSlot` 是下次继续解码必须保留的持久边界。

ownership 上：
内存由 memory context 批量释放。
fd 由 fd.c 和 resource cleanup 兜底关闭。
spill 文件由 reorder buffer 的 transaction cleanup、context cleanup、startup cleanup 和下一次 allocate cleanup 删除。
slot active ownership 由 `ReplicationSlotRelease()` 和 exit callback 释放。

错误路径上：
普通 aborted transaction 不输出 commit。
streamed aborted transaction 需要 stream abort。
output plugin 或 decoding ERROR 会清理本地事务和资源，但不会自动推进消费者确认。
consumer disconnect 会释放 active slot ownership，但 persistent slot 的 `restart_lsn` 和 `confirmed_flush` 仍保留。

诊断上：
`pg_replication_slots` 看 slot 边界。
`pg_stat_replication_slots` 看 spill/stream 累计压力。
`pg_replslot/<slot>/xid-*.spill` 只能作为临时实现细节辅助判断。
真正的因果仍要回到 WAL 读取起点、消费者确认位置和 reorder buffer cleanup 路径。

可迁移规律：

```text
能从持久日志重建的运行态，应该 aggressive cleanup；
会改变恢复起点或清理 horizon 的进度状态，必须 conservative advance；
ERROR cleanup 负责不泄漏资源，不负责伪造成功确认。
```
