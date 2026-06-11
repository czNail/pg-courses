# PostgreSQL reorder buffer spill 与 logical decoding 内存压力

## 课程定位

前置知识：已经理解 logical replication slot 会保留 WAL 和 xmin 边界，也知道 logical decoding 不是把 WAL record 原样发给下游，而是要按事务、snapshot、subtransaction 和 output plugin 协议重新组织变更。

本节唯一主问题：

```text
大事务或长事务会如何撑大 reorder buffer，
什么时候需要把 change spill 到磁盘，
logical_decoding_work_mem 如何在吞吐、内存和磁盘 I/O 之间折中？
```

核心矛盾：

```text
logical decoding 必须按事务边界、提交顺序和历史 snapshot 输出完整 change；
但 WAL 是按物理发生顺序持续到达的，
一个未提交的大事务可能产生远超 decoding backend 可承受内存的 decoded change。
```

PostgreSQL 的解法不是让 logical slot 无限占用内存，也不是一看到大事务就拒绝解码。它在 reorder buffer 内维护 transaction / subtransaction 级内存账本；当账本达到 `logical_decoding_work_mem` 后，优先尝试 streaming in-progress top transaction，不能 streaming 时把最大的 transaction 或 subtransaction 的 changes 序列化到 slot 目录下的 spill 文件。

学完后应能判断：`logical_decoding_work_mem` 限制的是什么；为什么 `ReorderBufferCheckMemoryLimit()` 选择 largest transaction/subtransaction；为什么 top transaction 和 subtransaction 要分开统计；什么时候看 `spill_count` / `spill_bytes`，什么时候看 `stream_count` / `stream_bytes`；spill file 如何命名、恢复、删除；ERROR、abort 和 server restart 后哪些路径兜底。

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

当前 commit 下，`logical_decoding_work_mem` 的主 GUC 定义在 `src/backend/utils/misc/guc_parameters.dat`；`src/backend/utils/misc/guc_tables.c` 仍包含 `debug_logical_replication_streaming_options` 这类枚举表。旧资料如果说所有 GUC 都在 `guc_tables.c`，要按当前源码修正阅读入口。

## 1. 本节在总主线中的位置

前面 slot 课程回答的是 WAL 和可见性历史能不能保住：

```text
logical replication slot
  -> restart_lsn 保留 WAL
  -> catalog_xmin / xmin 保护解码还需要的版本
  -> pg_replication_slots 暴露 WAL retention 风险
```

本节进入当前 logical decoding backend 内部：

```text
WAL record 被读到
  -> decode.c 解析成 ReorderBufferChange
  -> reorderbuffer.c 按 xid 放进 ReorderBufferTXN
  -> 未提交事务持续积累 decoded changes
  -> 超过 logical_decoding_work_mem
  -> streaming 或 spill
  -> commit / abort / prepare 边界输出或清理
```

这里的压力不是 shared memory 中的 slot state，而是 decoding session 的 backend-local memory 和 `$PGDATA/pg_replslot/<slot>` 下的本地文件 I/O。

但它会向外传播：

```text
reorder buffer 处理慢
  -> logical walsender 发送慢
  -> confirmed_flush_lsn 推进慢
  -> restart_lsn 保守
  -> slot 保留更多 WAL
  -> pg_wal 空间风险和 downstream apply lag 增加
```

因此 `logical_decoding_work_mem` 不是单纯的“内存越大越好”。它在三件事之间折中：更高值减少 spill/restore，但提高单个 decoding backend 的峰值内存；更低值更早约束内存，但更容易产生 I/O、callback 和延迟抖动；开启 streaming 可以提前下推大事务，但要求 plugin/protocol 支持，并且仍要处理 partial change、abort、snapshot 和 cleanup。

## 2. 核心矛盾与一句话运行模型

本节运行模型：

```text
ReorderBufferQueueChange() 每加入一个 decoded change 都更新 rb->size 和 txn->size；
当 rb->size 达到 logical_decoding_work_mem * 1024，
ReorderBufferCheckMemoryLimit() 选择最大的可驱逐对象；
若可 streaming，就调用 ReorderBufferStreamTXN() 提前输出；
否则调用 ReorderBufferSerializeTXN() 写入 xid-*-lsn-*.spill 文件；
commit 时再通过 ReorderBufferRestoreChanges() 分批读回，并按 LSN merge 输出。
```

这不是 executor sort/hash 的 temp file spill。reorder buffer spill 文件不是匿名临时文件，而是 logical decoding 对某个 slot、某个 xid、某个 WAL segment 范围的命名文件。它也不是 WAL 本身；它保存的是 `ReorderBufferDiskChange` 加 tuple、message、snapshot、invalidation 等 decoded payload，用来减少 decoded changes 常驻内存。

核心 tension 是：

```text
事务语义要求：
  未提交事务不能作为普通 commit transaction 输出。
  subtransaction 必须并入 top transaction。
  catalog changes 需要 historic snapshot、tuplecid 和 invalidation 边界。

资源语义要求：
  decoding backend 不能让大事务无限增长内存。
  spilling 后仍要在 commit 时按 LSN 顺序恢复。
  output plugin ERROR、transaction abort、server restart 都不能留下错误状态。
```

PostgreSQL 把问题拆成四层状态：`ReorderBuffer` 保存 session 全局账本和回调；`ReorderBufferTXN` 保存每个 xid/subxid 的 change list、snapshot、subtxn、内存大小和 spill 状态；spill file 保存已从内存驱逐的 decoded changes；`pg_stat_replication_slots` 暴露 slot 粒度的累计 spill/stream/memory exceeded 统计。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/reorderbuffer.h` | `ReorderBuffer`、`ReorderBufferTXN`、`ReorderBufferChange` 字段语义。 |
| 2 | `src/backend/replication/logical/reorderbuffer.c` | queue、memory accounting、spill、restore、streaming、cleanup 主实现。 |
| 3 | `src/backend/replication/logical/decode.c` | WAL record 如何变成 `ReorderBufferChange`，commit/abort 如何进入 reorder buffer。 |
| 4 | `src/backend/replication/logical/logical.c` | `StartupDecodingContext()` 创建 reorder buffer，`UpdateDecodingStats()` 写 slot 统计。 |
| 5 | `src/backend/utils/misc/guc_parameters.dat` | 当前 commit 下 `logical_decoding_work_mem` 的 GUC 定义。 |
| 6 | `src/backend/utils/misc/guc_tables.c` | `debug_logical_replication_streaming` 的枚举值。 |
| 7 | `src/backend/storage/file/fd.c` / `src/include/storage/fd.h` | `OpenTransientFile()`、`PathNameOpenFile()`、`FileRead()`、`FileClose()` 的文件边界。 |
| 8 | `src/backend/catalog/system_views.sql` / `src/backend/utils/adt/pgstatfuncs.c` | `pg_stat_replication_slots` 视图列和函数输出。 |
| 9 | `src/backend/utils/activity/pgstat_replslot.c` / `src/include/pgstat.h` | replication slot stats 的累计更新路径。 |
| 10 | `src/backend/access/transam/xlogutils.c` | 作为边界文件读：本节不是 redo page fetch，不要把 logical decoding spill 和 physical redo buffer 混淆。 |

推荐源码阅读顺序：

```text
ReorderBufferAllocate()
  -> ReorderBufferQueueChange()
  -> ReorderBufferChangeMemoryUpdate()
  -> ReorderBufferCheckMemoryLimit()
  -> ReorderBufferSerializeTXN()
  -> ReorderBufferRestoreChanges()
  -> ReorderBufferProcessTXN()
  -> ReorderBufferCleanupTXN()
```

然后回到 `decode.c` 看调用者：

```text
LogicalDecodingProcessRecord()
  -> rmgr.rm_decode()
  -> DecodeInsert() / DecodeUpdate() / DecodeDelete() / DecodeTruncate()
     -> ReorderBufferQueueChange()

DecodeCommit()
  -> ReorderBufferCommitChild()
  -> ReorderBufferCommit()
  -> UpdateDecodingStats()

DecodeAbort()
  -> ReorderBufferAbort()
  -> UpdateDecodingStats()
```

## 4. `ReorderBuffer`：backend-local 全局账本

`StartupDecodingContext()` 在 `logical.c` 中创建 `LogicalDecodingContext`，随后：

```text
ctx->reorder = ReorderBufferAllocate()
ctx->snapshot_builder = AllocateSnapshotBuilder(...)
ctx->reorder->private_data = ctx
```

`ReorderBufferAllocate()` 创建独立 memory context，并在其下创建三类子 context：

```text
ReorderBuffer
  -> change_context: SlabContext, 固定大小 ReorderBufferChange
  -> txn_context:    SlabContext, 固定大小 ReorderBufferTXN
  -> tup_context:    GenerationContext, 变长 tuple payload
```

这不是随意选择 allocator。`ReorderBufferChange` 和 `ReorderBufferTXN` 是固定大小对象，适合 Slab；tuple payload 大小差异明显，并且大事务内一批 tuple 生命周期相近，适合 Generation。

和本节最相关的 `ReorderBuffer` 字段：

| 字段 | 语义 |
| --- | --- |
| `by_txn` | xid 到 `ReorderBufferTXN` 的 lookup table。 |
| `toplevel_by_lsn` | 可能是 top transaction 的事务，按 first LSN 排序。 |
| `txns_by_base_snapshot_lsn` | 有 base snapshot 的事务，streaming candidate 会扫这里。 |
| `txn_heap` | 按 `txn->size` 排序的 max-heap，用于快速找 largest transaction/subtransaction spill。 |
| `size` | reorder buffer 当前 decoded changes 的内存账本。 |
| `spillTxns` / `spillCount` / `spillBytes` | pending spill 统计，最终进入 `pg_stat_replication_slots`。 |
| `streamTxns` / `streamCount` / `streamBytes` | pending streaming 统计。 |
| `memExceededCount` | 达到 `logical_decoding_work_mem` 阈值的次数。 |
| `totalTxns` / `totalBytes` | 输出给 plugin 的事务和 decoded bytes 统计。 |

这些都是当前 decoding backend 的本地状态。`pg_stat_replication_slots` 看到的是 `UpdateDecodingStats()` 把 pending counters 累加到 stats subsystem 后的 slot 级累计值。

## 5. `ReorderBufferTXN`：top 与 subtransaction 的双层统计

`ReorderBufferTXN` 同时表示 top transaction 和 subtransaction。判断方式在 `reorderbuffer.h`：

```text
txn->toptxn == NULL:
  top transaction

txn->toptxn != NULL:
  subtransaction

rbtxn_get_toptxn(txn):
  subtxn 返回 top；top 返回自己
```

本节最重要的字段是：

| 字段 | 语义 |
| --- | --- |
| `nentries` | 该事务总 change 数，包括已经 spill 到磁盘的。 |
| `nentries_mem` | 该事务当前仍在内存中的 change 数。 |
| `changes` | 当前内存中的 `ReorderBufferChange` 链。 |
| `subtxns` / `nsubtxns` | top transaction 持有的 non-aborted subtransactions。 |
| `size` | 该 top/sub transaction 当前在内存中的 decoded change 字节数。 |
| `total_size` | top transaction 汇总自身和 subtransactions 的 change 大小。 |
| `txn_flags` | `RBTXN_IS_SERIALIZED`、`RBTXN_IS_STREAMED`、`RBTXN_HAS_PARTIAL_CHANGE` 等生命周期标记。 |

一个大事务可能是：

```text
top xid 100:
  txn->size       = 2MB
  txn->total_size = 80MB
  subxid 101 size = 50MB
  subxid 102 size = 28MB
```

如果没有 streaming，largest spill 可以直接选 subxid 101，因为 `txn_heap` 按每个 top/sub transaction 的 `size` 排序。这样能更快释放最大内存来源。

如果可以 streaming，PostgreSQL 不能把某个 subtransaction 单独作为 downstream 可见的 in-progress transaction stream 出去。它必须选择 top transaction，所以 `ReorderBufferLargestStreamableTopTXN()` 看的是 top transaction 的 `total_size` 和 streamable 状态。

因此同一套账本必须同时有 per-subtransaction `size` 和 top-level `total_size`。

## 6. `ReorderBufferChange` 与 `logical_decoding_work_mem`

`decode.c` 把 WAL payload 转成 `ReorderBufferChange`。以 `DecodeInsert()` 为例：

```text
DecodeInsert()
  -> ReorderBufferAllocChange()
  -> action = REORDER_BUFFER_CHANGE_INSERT
  -> ReorderBufferAllocTupleBuf()
  -> DecodeXLogTuple()
  -> ReorderBufferQueueChange()
```

一个 change 可能是普通 INSERT/UPDATE/DELETE tuple，也可能是 TRUNCATE relation OID list、logical message、historic snapshot、command id change、tuplecid mapping、cache invalidation、speculative insert internal record。

`ReorderBufferChangeSize()` 计算 decoded change 在内存中的大小。`ReorderBufferChangeMemoryUpdate()` 特意忽略 `REORDER_BUFFER_CHANGE_INTERNAL_TUPLECID`，因为 tuplecid 存在独立列表里，达到内存阈值时并不会被 evict。这里的结论很重要：

```text
logical_decoding_work_mem 不是 reorder buffer 全部内存的硬上限；
它限制的是被纳入 decoded change accounting、并可 spill/stream 的那部分。
```

事务记录、hash 表、snapshot builder、output plugin 私有状态、syscache、WAL reader buffer、C runtime 和 fd table 都不属于这个数字的直接上限。

当前 GUC 定义在 `guc_parameters.dat`：

```text
name: logical_decoding_work_mem
context: PGC_USERSET
group: RESOURCES_MEM
unit: GUC_UNIT_KB
default: 65536
min: 64
description: each internal reorder buffer before spilling to disk
```

默认值是 64MB。`ReorderBufferCheckMemoryLimit()` 使用：

```text
rb->size >= logical_decoding_work_mem * (Size) 1024
```

这是 per decoding context 的阈值。多个 logical slots 或多个 decoding clients 并行时，内存压力可以乘以 session 数。它也是触发 eviction 的阈值，不是精确 RSS ceiling；最近一个 change 加入之后才检查，单个大 tuple/message 可以让账本先越过阈值再处理。

## 7. WAL change 入队主流程

每条 WAL record 进入 `LogicalDecodingProcessRecord()`：

```text
LogicalDecodingProcessRecord(ctx, record)
  -> txid = XLogRecGetTopXid(record)
  -> 如果 top xid 有效:
       ReorderBufferAssignChild(ctx->reorder, topxid, record xid, lsn)
  -> rmgr.rm_decode(ctx, &buf)
```

heap rmgr 的 decode 路径进入 `DecodeInsert()`、`DecodeUpdate()`、`DecodeDelete()`、`DecodeTruncate()` 等函数。这些函数把 WAL payload 转成 change，再调用 `ReorderBufferQueueChange()`。

主路径：

```text
ReorderBufferQueueChange(rb, xid, lsn, change, toast_insert)
  -> ReorderBufferTXNByXid(rb, xid, create=true, ...)
  -> 如果 txn 已知 aborted:
       ReorderBufferFreeChange()
       return
  -> 标记 top transaction 有 streamable change
  -> change->lsn = lsn
  -> change->txn = txn
  -> dlist_push_tail(&txn->changes, &change->node)
  -> txn->nentries++
  -> txn->nentries_mem++
  -> ReorderBufferChangeMemoryUpdate(..., addition=true, size)
  -> ReorderBufferProcessPartialChange()
  -> ReorderBufferCheckMemoryLimit()
```

内存检查发生在每次 queue decoded change 之后，不是后台周期任务。

`ReorderBufferChangeMemoryUpdate()` 同时更新三层：

```text
txn->size:
  当前事务或子事务在内存中的 change 大小。

rb->size:
  当前 reorder buffer 所有在内存 decoded changes 的总大小。

rbtxn_get_toptxn(txn)->total_size:
  top transaction 汇总自身和 subtransactions 的大小。
```

addition 路径维护 `txn_heap`：如果 old size 非零，先从 pairing heap 移除，再按新 size 加回。subtraction 路径同样维护 heap，size 归零时不再加入。这样每次 queue/free 多付出少量账本成本，换来超过阈值时可以快速找到 largest transaction/subtransaction。

## 8. `ReorderBufferCheckMemoryLimit()`：阈值与驱逐选择

阈值检查第一步：

```text
if rb->size >= logical_decoding_work_mem * 1024:
  rb->memExceededCount += 1
else if debug_logical_replication_streaming == buffered:
  return
```

`debug_logical_replication_streaming = immediate` 是开发测试用 GUC。它可以在未超过阈值时也持续触发 streaming 或 serialization，用来覆盖测试路径；不要把它当生产调参手段。

真正驱逐是 while loop：

```text
while rb->size >= limit
   or debug immediate and rb->size > 0:

  if ReorderBufferCanStartStreaming(rb)
     and largest streamable top txn exists:
       ReorderBufferStreamTXN(rb, txn)
  else:
       txn = ReorderBufferLargestTXN(rb)
       ReorderBufferSerializeTXN(rb, txn)

  Assert(txn->size == 0)
  Assert(txn->nentries_mem == 0)
```

超过阈值后可能驱逐多个事务。源码注释说明了一个实际原因：用户可以运行中把 `logical_decoding_work_mem` 调得更低，驱逐一个 largest transaction 不一定能让 `rb->size` 回到阈值以下。

streaming 优先于 spill，但 `ReorderBufferCanStartStreaming()` 要求：

```text
ctx->streaming 为真
SnapBuildCurrentState(builder) >= SNAPBUILD_CONSISTENT
当前 reader position 不处于 SnapBuildXactNeedsSkip() 要求跳过的状态
```

`ctx->streaming` 来自 output plugin callbacks。`StartupDecodingContext()` 只要看到 streaming callback 之一存在，就认为插件支持 streaming；wrapper 后续会负责缺失 callback 的细节检查或 no-op。

streaming 选择只看 top transaction。`ReorderBufferLargestStreamableTopTXN()` 扫 `txns_by_base_snapshot_lsn`，跳过有 partial change、没有 streamable change、已知 aborted、`total_size == 0` 的事务。

如果不能 streaming，则 `ReorderBufferLargestTXN()` 从 `txn_heap` 取最大 `txn->size`，对象可以是 top transaction，也可以是 subtransaction。

## 9. partial change 为什么阻止 streaming

`ReorderBufferProcessPartialChange()` 处理两类 incomplete change：

```text
toast chunk:
  toast table insert 到主表 insert/update 完成前，不能单独 stream。

speculative insert:
  INTERNAL_SPEC_INSERT 到 confirm/abort 记录到达前，不能当普通 INSERT stream。
```

如果在这些边界中间 streaming，downstream 可能看到无法重建完整 tuple 或语义尚未确认的 change。因此源码用 `RBTXN_HAS_PARTIAL_CHANGE` 标记 top transaction。主表 INSERT/UPDATE 清理 toast，或 speculative confirm/abort 到达后，partial 标记才会被清除。

这里体现了一个重要取舍：

```text
如果允许 partial change 中途 streaming，
实现必须记录每个 subtxn、spill file、toast/spec state 的精确恢复 offset。
当前实现选择更保守：
  有 partial change 时先不 stream；
  如果已经 serialized，等 complete change 到达后再尽快 stream。
```

所以启用 streaming 不代表完全没有 spill。某些事务在一段时间内不可 stream，仍可能先被 spill 到磁盘。

## 10. `ReorderBufferSerializeTXN()`：spill 到磁盘

当不能 streaming 或没有 streamable top transaction 时，`ReorderBufferCheckMemoryLimit()` 调用 `ReorderBufferSerializeTXN(rb, txn)`。

它做三件事：递归序列化 child subtransactions；把 `txn->changes` 中每个 change 写入 spill file；从内存链表删除 change，释放对象并更新内存账本。

伪调用链：

```text
ReorderBufferSerializeTXN()
  -> foreach subtxn:
       ReorderBufferSerializeTXN(rb, subtxn)
  -> foreach change in txn->changes:
       if fd == -1 or change->lsn 不在当前 WAL segment:
          CloseTransientFile(old fd)
          ReorderBufferSerializedPath(...)
          OpenTransientFile(path, O_CREAT | O_WRONLY | O_APPEND | PG_BINARY)
       ReorderBufferSerializeChange(rb, txn, fd, change)
       dlist_delete(&change->node)
       ReorderBufferFreeChange(rb, change, false)
       spilled++
  -> ReorderBufferChangeMemoryUpdate(rb, NULL, txn, false, size)
  -> 更新 spillCount / spillBytes / spillTxns
  -> txn->nentries_mem = 0
  -> txn->txn_flags |= RBTXN_IS_SERIALIZED
```

这里 `ReorderBufferFreeChange(..., false)` 不更新账本，因为函数最后一次性用 `size = txn->size` 做 subtraction。这样避免对每个 change 都维护 `txn_heap`。

spill 文件路径由 `ReorderBufferSerializedPath()` 生成：

```text
pg_replslot/<slotname>/xid-<xid>-lsn-<hi>-<lo>.spill
```

`<hi>-<lo>` 来自该 WAL segment 起点的 LSN：

```text
XLogSegNoOffsetToRecPtr(segno, 0, wal_segment_size, recptr)
LSN_FORMAT_ARGS(recptr)
```

例如：

```text
pg_replslot/my_slot/xid-742-lsn-0-16000000.spill
```

这里不关心 timeline。源码注释说明这些文件只在一次 decoding run 内使用，同一 run 中 LSN 可以定位特定 WAL record。

`ReorderBufferSerializeChange()` 写的 on-disk record 形态：

```text
ReorderBufferDiskChange:
  Size size
  ReorderBufferChange change
  variable data follows
```

variable data 按 action 写出：INSERT/UPDATE/DELETE 写 `HeapTupleData` 和 tuple data；MESSAGE 写 prefix 和 message；INVALIDATION 写 `SharedInvalidationMessage` 数组；INTERNAL_SNAPSHOT 写 `SnapshotData`、xip、subxip；TRUNCATE 写 relation OID array。

写入时使用普通 `write()`，并设置 `WAIT_EVENT_REORDER_BUFFER_WRITE`。如果写入字节数不等于 `ondisk->size`，代码关闭 fd；如果 errno 为空则按 `ENOSPC` 处理；然后 `ereport(ERROR, "could not write to data file for XID ...")`。磁盘满或文件系统错误不会退回到“继续占内存”，而是终止当前 decoding 操作，让上层 ERROR cleanup 收尾。

## 11. spill 统计：`spill_txns`、`spill_count`、`spill_bytes`

`ReorderBufferSerializeTXN()` 只有实际 spilled change 时才更新：

```text
rb->spillCount += 1
rb->spillBytes += size
rb->spillTxns += first time this txn serialized ? 1 : 0
UpdateDecodingStats(ctx)
```

三个字段含义不同：

```text
spill_txns:
  有多少 transaction/subtransaction 曾经被 spill。
  同一个事务重复 spill 不重复计数。

spill_count:
  spill-to-disk invocation 次数。
  同一个事务可能多次达到阈值，所以它可能大于 spill_txns。

spill_bytes:
  被 spill 到磁盘的 decoded data 字节数累计。
```

`UpdateDecodingStats()` 立即把 pending counters 累加到 slot stats，然后把 `rb->spill*` 清零。源码注释说明统计不只在 commit/abort 时更新，spill 或 stream 时也更新，避免 decoding session 中断导致已经发生的 I/O 压力完全不可见。

## 12. restore TXN 路径：当前源码的真实函数名

有些资料会把“恢复一个 spilled transaction”口头称为 restore TXN。当前 commit 下没有名为 `ReorderBufferRestoreTXN()` 的函数。真实路径是：

```text
ReorderBufferIterTXNInit()
  -> 如果 txn 或 subtxn rbtxn_is_serialized():
       ReorderBufferSerializeTXN(rb, txn)     // 把剩余内存 change 也写盘
       ReorderBufferRestoreChanges(rb, txn, ...)

ReorderBufferIterTXNNext()
  -> 当前内存批次用完且 nentries != nentries_mem:
       ReorderBufferRestoreChanges(rb, txn, ...)

ReorderBufferRestoreChange()
  -> 把单个 on-disk change 转回 in-memory ReorderBufferChange
```

本节所说的“restore TXN 路径”指这组函数组合，文件和函数名按当前源码为准。

commit 时不能简单把 top transaction 和 subtransactions 的链表拼起来。WAL 中每个 transaction/subtransaction 的 change stream 内部按 LSN 有序，commit 输出要用 binary heap 做 k-way merge：

```text
ReorderBufferIterTXNInit()
  -> 计算 top + subtxns 中有 change 的 stream 数
  -> 每个 stream 放入第一条 change 的 LSN
  -> binaryheap_build()

ReorderBufferIterTXNNext()
  -> 取最小 LSN change
  -> 推进该 stream 到下一条 change
  -> 如果当前 batch 用完，ReorderBufferRestoreChanges() 读下一批
```

如果某个 txn 被 spill，`ReorderBufferIterTXNInit()` 会先 `ReorderBufferSerializeTXN()` 把剩余内存部分也写出，再 restore 第一批。原因是同一 transaction 的 change 要在同一个恢复模型里有序读取，不能让“部分内存链表 + 部分 spill file”破坏 iterator 顺序。

## 13. `ReorderBufferRestoreChanges()` 与 `ReorderBufferRestoreChange()`

恢复函数每次最多读 `max_changes_in_memory = 4096` 个 change。这个常量当前只用于 restore。源码顶部注释说明，restore 时不能直接用 `logical_decoding_work_mem` 精确切分，因为 subtransactions 独立恢复，简单按 N 等分容易产生 thrashing。

主流程：

```text
ReorderBufferRestoreChanges(rb, txn, file, segno)
  -> 先释放 txn->changes 中已有内存 batch
  -> txn->nentries_mem = 0
  -> 从 txn->first_lsn 所在 segment 开始
  -> PathNameOpenFile(spill path, O_RDONLY | PG_BINARY)
  -> 如果 ENOENT，segno++ 继续找下一个可能文件
  -> FileRead(header) with WAIT_EVENT_REORDER_BUFFER_READ
  -> FileRead(variable data) with WAIT_EVENT_REORDER_BUFFER_READ
  -> ReorderBufferRestoreChange()
  -> restored++
  -> restored == 4096 或超过 final_lsn segment 时返回
```

如果读 header 或 payload 短读，代码直接 ERROR，例如 “could not read from reorderbuffer spill file: read X instead of Y bytes”。这通常指向 spill 文件损坏、底层文件系统问题、手工干预或非预期版本/二进制不一致。

restore 时不能复用磁盘里的指针。序列化前的指针只在旧 backend-local memory context 中有意义，写盘后更不可能跨时间使用。`ReorderBufferRestoreChange()` 会重新分配 change、tuple、message、snapshot、invalidation 和 relids，并重建 tuple `t_data`、snapshot `xip/subxip` 等内部指针。

核心不变量：

```text
spill file 保存的是可重建状态，不保存可复用指针。
```

`ReorderBufferIterTXNNext()` 当前 batch 用完时，会把当前返回给调用者的 change 放进 `state->old_change` 延迟释放，再调用 `ReorderBufferRestoreChanges()` 读新 batch。`ReorderBufferIterTXNFinish()` 最后关闭还打开的 file VFD，释放 `old_change`、binary heap 和 iterator state。

## 14. streaming 与 spill 的关系

如果 output plugin 支持 streaming，且 snapshot builder 已经 consistent，超过阈值后 `ReorderBufferCheckMemoryLimit()` 优先：

```text
txn = ReorderBufferLargestStreamableTopTXN(rb)
ReorderBufferStreamTXN(rb, txn)
```

`ReorderBufferStreamTXN()` 准备或复用 historic snapshot，然后调用：

```text
ReorderBufferProcessTXN(rb, txn, InvalidXLogRecPtr,
                        snapshot_now, command_id, true)
```

streaming 路径会走 `stream_start`、`stream_change` / `stream_message` / `stream_truncate`、`stream_stop`，随后 `ReorderBufferTruncateTXN()` 丢掉已经输出的 changes，但保留 invalidations、snapshot、tuplecids 等剩余事务状态。事务最终 commit 时，如果 `rbtxn_is_streamed(txn)`，`ReorderBufferStreamCommit()` 会先 stream 剩余部分，再调用 `stream_commit`，最后 `ReorderBufferCleanupTXN()`。

如果事务最终 abort：

```text
ReorderBufferAbort()
  -> 如果 rbtxn_is_streamed(txn):
       rb->stream_abort(rb, txn, lsn)
  -> ReorderBufferCleanupTXN()
```

所以 streaming 是减少内存和提前下推大事务的能力，不是把事务语义改成“每个 change 独立提交”。

`stream_txns`、`stream_count`、`stream_bytes` 的含义：

```text
stream_txns:
  有多少 top transaction 曾经以 streaming 方式输出。

stream_count:
  streaming invocation 次数。
  同一个大事务超过阈值多次，可以被 stream 多次。

stream_bytes:
  streaming 路径解码输出的 transaction data 字节数。
```

如果 `mem_exceeded_count` 增长、`stream_count` 增长、`spill_count` 不增长，通常表示 streaming 正在承担大事务内存压力。如果 `mem_exceeded_count` 和 `spill_count` 增长而 `stream_count` 为 0，常见原因是插件不支持 streaming、订阅未启用 streaming、snapshot builder 尚未 consistent、事务暂时有 partial change，或测试环境强制 buffered 行为。如果 `stream_count` 和 `spill_count` 都增长，不一定矛盾；不同事务可能走不同路径，同一事务也可能先因 partial change spill，后续 complete 后再 stream。

## 15. `ReorderBufferProcessTXN()` 的 ERROR 与 concurrent abort 路径

`ReorderBufferProcessTXN()` 用 `PG_TRY()` 包住 output plugin 调用和 historic snapshot 处理。正常路径结束后：

```text
TeardownHistoricSnapshot(false)
AbortCurrentTransaction()
执行 invalidations
if streaming or prepared:
  ReorderBufferTruncateTXN()
else:
  ReorderBufferCleanupTXN()
```

`PG_CATCH()` 中会：

```text
ReorderBufferIterTXNFinish()
TeardownHistoricSnapshot(true)
AbortCurrentTransaction()
执行 invalidations 或 InvalidateSystemCaches()
if errcode == ERRCODE_TRANSACTION_ROLLBACK and streaming/prepared:
  标记 curtxn aborted
  ReorderBufferResetTXN()
else:
  ReorderBufferCleanupTXN()
  PG_RE_THROW()
```

这个特殊的 `ERRCODE_TRANSACTION_ROLLBACK` 路径服务 streaming/prepared decoding：解码可能发生在最终 commit record 到达前，被解码事务可能并发 abort。`SetupCheckXidLive()` 检查到后，用这个错误码表示当前 chunk 不能继续按正常事务输出。系统清理当前状态、标记 aborted，然后等待后续 abort record 走 `ReorderBufferAbort()`。

普通 output plugin ERROR 则会清理 reorder state 后重抛，调用者会看到 decoding 失败。

## 16. cleanup：内存、spill file 与 stale 文件

普通 commit：

```text
DecodeCommit()
  -> ReorderBufferCommit()
     -> ReorderBufferReplay()
        -> ReorderBufferProcessTXN()
           -> ReorderBufferCleanupTXN()
```

abort：

```text
DecodeAbort()
  -> ReorderBufferAbort()
     -> stream_abort if needed
     -> ReorderBufferCleanupTXN()
```

skip committed transaction：

```text
DecodeCommit()
  -> DecodeTXNNeedSkip()
  -> ReorderBufferForget()
     -> 必要时执行 catalog invalidations
     -> ReorderBufferCleanupTXN()
```

`ReorderBufferForget()` 不能简单等同于 abort。被 skip 的事务已经 commit，可能修改 shared catalogs；即使当前 decoder 不输出它，也可能需要执行 invalidations，防止后续 decoding 使用旧 syscache。

`ReorderBufferCleanupTXN()` 做完整收尾：递归 cleanup subtransactions；释放 `txn->changes` 并更新 memory accounting；释放 tuplecid changes；释放 base snapshot refcount；释放 last streamed snapshot；从链表和 `by_txn` hash 移除；如果 `rbtxn_is_serialized(txn)`，调用 `ReorderBufferRestoreCleanup()` 删除 spill 文件；最后 `ReorderBufferFreeTXN()`。

`ReorderBufferRestoreCleanup()` 根据 `txn->first_lsn` 和 `txn->final_lsn` 算出 first/last WAL segment，枚举可能文件名并 unlink：

```text
for cur = first..last:
  ReorderBufferSerializedPath(path, slot, txn->xid, cur)
  unlink(path)
```

如果 unlink 失败且不是 `ENOENT`，直接 ERROR。

session/free/startup 的 stale cleanup：

```text
ReorderBufferAllocate()
  -> ReorderBufferCleanupSerializedTXNs(slotname)

ReorderBufferFree()
  -> ReorderBufferCleanupSerializedTXNs(slotname)

StartupReorderBuffer()
  -> 扫 pg_replslot 下合法 slot directory
  -> ReorderBufferCleanupSerializedTXNs(slotname)
```

`ReorderBufferCleanupSerializedTXNs()` 删除 slot 目录下名字以 `xid` 开头的文件。这说明 spill file 不是 crash-safe durable protocol。server restart 后会从 WAL 和 slot restart 边界重建需要的 reorder state，旧 spill 文件必须清掉，避免重复事务或错误输入。

## 17. 正确性机制层次

事务 ordering：WAL 到达顺序不是 output 顺序。`ReorderBufferCommitChild()` 把 commit record 中的 subxacts 合并到 top transaction；`ReorderBufferIterTXN*` 再按 LSN 做 k-way merge。

snapshot correctness：输出 change 时使用 historic snapshot，而不是普通 MVCC snapshot。`ReorderBufferProcessTXN()` 调用 `SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)` 和 `TeardownHistoricSnapshot()`；catalog changes、command id、tuplecid 和 invalidation 都会影响 logical decoding 看到的 schema 和 tuple identity。

memory correctness：`logical_decoding_work_mem` 不是靠 MemoryContext reset 实现的。MemoryContext 负责释放对象；何时可以丢弃 change 必须受事务语义控制。真正的边界是 change list ownership、memory accounting、spill/restore 可重建性和 commit/abort/forget/free/startup cleanup。

I/O correctness：spill 写失败、restore 短读、cleanup unlink 异常都会 ERROR。spill file 是当前 decoding run 的必要中间状态；一旦不可读写，不能假装继续 decode。但 server restart 后会删除 stale spill files，因为 durable source of truth 是 WAL + slot state，不是 spill file。

## 18. 成本、资源与跨模块传播

内存增长变量包括 change 数量、tuple payload 大小、UPDATE 是否携带 old key/new tuple、logical message 大小、TRUNCATE relid 数量、snapshot xip/subxip 数量、subtransaction 数量、toast reconstruction 状态和 output plugin 私有状态。`logical_decoding_work_mem` 只管被 `ReorderBufferChangeSize()` 纳入账本的部分，所以 backend RSS 高于该 GUC 不一定是 bug。

正常 queue path 的 CPU 成本包括 `ReorderBufferChangeSize()`、`ReorderBufferChangeMemoryUpdate()`、pairingheap remove/add、partial change 标记和阈值检查。超过阈值后的成本包括 largest transaction selection、action-specific serialization、commit 时 restore 反序列化、k-way merge、historic snapshot setup/teardown 和 output plugin callbacks。

spill I/O 是写放大和读放大组合：

```text
queue change:
  decoded tuple 已经存在内存

spill:
  写出 decoded representation
  释放内存

commit:
  再读回 decoded representation
  重建内存对象
  输出给 plugin
```

如果 `spill_count` 高而 `spill_bytes` 不大，说明系统频繁小批量 spill，可能有明显 syscall/file open/restore 开销。如果 `spill_bytes` 很大，瓶颈可能转移到存储吞吐、I/O queue 和下游 apply 速度。

spill 本身不直接保留更多 WAL；保留 WAL 的是 slot `restart_lsn`。但 spill/restore 让 decoding 变慢后，output plugin 写出慢，client acknowledge 慢，`confirmed_flush_lsn` 推进慢，`restart_lsn` 可能长期落后，最终导致 pg_wal 被 slot 保留更多。

因此诊断时要把两类视图合起来看：

```sql
SELECT slot_name, restart_lsn, confirmed_flush_lsn, wal_status, safe_wal_size
FROM pg_replication_slots
WHERE slot_type = 'logical';

SELECT slot_name, spill_count, spill_bytes,
       stream_count, stream_bytes,
       mem_exceeded_count, total_bytes
FROM pg_stat_replication_slots;
```

第一条看 WAL retention 风险，第二条看 decoding 内部是否在内存阈值附近退化。

## 19. 观测入口：`pg_stat_replication_slots`

当前视图列包括：

```text
slot_name
spill_txns
spill_count
spill_bytes
stream_txns
stream_count
stream_bytes
mem_exceeded_count
total_txns
total_bytes
slotsync_skip_count
slotsync_last_skip
stats_reset
```

常用查询：

```sql
SELECT slot_name,
       spill_txns,
       spill_count,
       pg_size_pretty(spill_bytes) AS spill,
       stream_txns,
       stream_count,
       pg_size_pretty(stream_bytes) AS stream,
       mem_exceeded_count,
       pg_size_pretty(total_bytes) AS decoded
FROM pg_stat_replication_slots
ORDER BY spill_bytes + stream_bytes DESC;
```

解释时看比例：

```text
spill_bytes / total_bytes 高:
  大量 decoded data 经过磁盘。

stream_bytes / total_bytes 高:
  大事务主要通过 streaming 提前输出。

mem_exceeded_count 高但 spill/stream bytes 低:
  阈值频繁被触碰，可能每次驱逐很少，需要结合日志和等待事件判断。

spill_count 远大于 spill_txns:
  同一批大事务反复超过阈值。
```

统计是累计值。实验前要记录 baseline 或重置对应 stats；否则不要把历史压力当成当前事务的因果。

## 20. 观测入口：wait event、日志和文件

spill write wait event：

```text
WAIT_EVENT_REORDER_BUFFER_WRITE
```

restore read wait event：

```text
WAIT_EVENT_REORDER_BUFFER_READ
```

可以短时观察：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE backend_type IN ('walsender', 'client backend')
  AND wait_event LIKE 'ReorderBuffer%';
```

wait event 是瞬时状态。没看到不代表没有 spill；只说明采样时 backend 没有正处在该等待点。

DEBUG2 日志中可见：

```text
spill %u changes in XID %u to disk
restored %u/%u changes from disk
UpdateDecodingStats: updating stats ...
```

实验环境可以观察文件：

```bash
find "$PGDATA/pg_replslot/<slotname>" -name 'xid-*.spill' -ls
```

但不要让监控依赖这些文件长期存在。正常运行中 spill 文件可能很快被 restore cleanup 删除；手工删除运行中的 `xid-*.spill` 会让后续 restore 报错。

gdb 断点：

```gdb
break ReorderBufferQueueChange
break ReorderBufferChangeMemoryUpdate
break ReorderBufferCheckMemoryLimit
break ReorderBufferSerializeTXN
break ReorderBufferRestoreChanges
break ReorderBufferStreamTXN
break ReorderBufferCleanupTXN
break UpdateDecodingStats
```

观察字段：

```gdb
p rb->size
p logical_decoding_work_mem
p txn->xid
p txn->size
p txn->total_size
p txn->nentries
p txn->nentries_mem
p/x txn->txn_flags
p rb->spillCount
p rb->spillBytes
p rb->streamCount
p rb->streamBytes
```

注意 `UpdateDecodingStats()` 调用后，`rb->spill*` / `rb->stream*` pending counters 会清零，真实累计值已经进入 pgstat。

## 21. 常见异常路径与 fallback

不支持 streaming：`ReorderBufferCanStream()` 为 false，超过阈值后直接 `ReorderBufferLargestTXN()` + `ReorderBufferSerializeTXN()`。诊断上常见 `mem_exceeded_count` 和 `spill_count` 增长，`stream_count` 不增长。

支持 streaming 但暂时不能 stream：常见原因是 SnapBuild 尚未 consistent、当前事务需要 skip、事务没有 base snapshot、事务没有 streamable change、事务有 toast/speculative partial change、事务已知 aborted。fallback 仍是 spill。

磁盘写失败：`ReorderBufferSerializeChange()` 关闭 fd，设置 errno，报 “could not write to data file for XID ...”。此时查 slot 所在文件系统空间和 I/O 错误，不要只查 `work_mem`。

spill 文件读失败：`ReorderBufferRestoreChanges()` 读 header/payload 失败或短读会 ERROR。优先检查是否有人手工删除/修改 `pg_replslot/<slot>/xid-*.spill`，底层文件系统是否异常，或进程异常退出后 stale 文件清理是否没有执行。

output plugin ERROR：普通 commit decoding 中会 finish iterator、teardown historic snapshot、abort internal transaction、执行 invalidations、cleanup txn 后重抛。streaming/prepared 中的 concurrent abort 则走 `ERRCODE_TRANSACTION_ROLLBACK` 特殊路径，标记 aborted 并 reset txn，等待 abort record。

## 22. 课堂实验

### 实验 1：低阈值触发 spill

启用 logical replication，并把阈值调低：

```sql
ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM SET max_replication_slots = 4;
ALTER SYSTEM SET max_wal_senders = 4;
ALTER SYSTEM SET logical_decoding_work_mem = '64kB';
SELECT pg_reload_conf();
```

如果 `wal_level` 需要重启，重启后继续。

创建 slot：

```sql
SELECT * FROM pg_create_logical_replication_slot('rb_spill_slot', 'test_decoding');
```

制造一个大事务：

```sql
CREATE TABLE rb_spill_test(id int primary key, payload text);

BEGIN;
INSERT INTO rb_spill_test
SELECT g, repeat(md5(g::text), 20)
FROM generate_series(1, 20000) AS g;
COMMIT;
```

消费变化：

```sql
SELECT count(*)
FROM pg_logical_slot_get_changes('rb_spill_slot', NULL, NULL);
```

查看统计：

```sql
SELECT slot_name, spill_txns, spill_count, spill_bytes,
       stream_txns, stream_count, stream_bytes, mem_exceeded_count
FROM pg_stat_replication_slots
WHERE slot_name = 'rb_spill_slot';
```

回到源码解释：

```text
DecodeInsert()
  -> ReorderBufferQueueChange()
  -> ReorderBufferCheckMemoryLimit()
  -> ReorderBufferSerializeTXN()
  -> UpdateDecodingStats()
```

### 实验 2：观察 streaming 与 spill 的差异

参考 `src/test/subscription/t/015_stream.pl` 的思路，在 publisher 上设置 `logical_decoding_work_mem = 64kB`，创建 publication；subscriber 用：

```sql
CREATE SUBSCRIPTION tap_sub
CONNECTION '...'
PUBLICATION tap_pub
WITH (streaming = on);
```

制造大事务后在 publisher 查看：

```sql
SELECT slot_name, spill_count, spill_bytes,
       stream_count, stream_bytes,
       mem_exceeded_count
FROM pg_stat_replication_slots;
```

预期：支持 streaming 且事务可 stream 时，`stream_count` / `stream_bytes` 增长；如果事务中存在暂时 incomplete toast/speculative change，仍可能看到 spill。

源码对应：

```text
ReorderBufferCanStartStreaming()
ReorderBufferLargestStreamableTopTXN()
ReorderBufferStreamTXN()
ReorderBufferProcessTXN(..., streaming=true)
```

### 实验 3：断点看 top/subtransaction accounting

SQL：

```sql
BEGIN;
SAVEPOINT s1;
INSERT INTO rb_spill_test
SELECT 100000 + g, repeat(md5(g::text), 20)
FROM generate_series(1, 5000) AS g;
RELEASE SAVEPOINT s1;

SAVEPOINT s2;
INSERT INTO rb_spill_test
SELECT 200000 + g, repeat(md5(g::text), 20)
FROM generate_series(1, 5000) AS g;
RELEASE SAVEPOINT s2;
COMMIT;
```

gdb：

```gdb
break ReorderBufferChangeMemoryUpdate
commands
  silent
  printf "xid=%u size=%zu total=%zu rb=%zu nmem=%lu\n", txn->xid, txn->size, rbtxn_get_toptxn(txn)->total_size, rb->size, txn->nentries_mem
  continue
end
```

观察：subtransaction 自己的 `size` 增长；top transaction 的 `total_size` 汇总增长；`rb->size` 是所有 decoded changes 的全局账本。

## 23. 常见误区

误区一：`logical_decoding_work_mem` 是 walsender 总内存上限。更准确地说，它是每个 internal reorder buffer 在 spill 前可使用的 decoded change 账本阈值。

误区二：spill 表示事务太大，必须失败。spill 是正常 fallback，目标是让 decoding 继续前进，只是用更多 I/O 和 restore 成本换取内存边界。

误区三：开启 streaming 后不会再有 spill。streaming 依赖 plugin callbacks、consistent snapshot 和完整 change。toast/speculative partial change、不可 stream 的阶段和不支持 streaming 的插件仍会 spill。

误区四：`spill_count` 等于大事务个数。`spill_txns` 更接近“多少事务/子事务曾经 spill”；`spill_count` 是 invocation 次数，同一个事务可多次 spill。

误区五：可以手工删除 `pg_replslot/<slot>/xid-*.spill` 释放空间。运行中的 decoder 可能正要 restore 这些文件；手工删除会导致 restore ERROR。

误区六：看到 `mem_exceeded_count` 就应该盲目调大参数。如果 streaming 正常承担压力且延迟可接受，触碰阈值未必是问题；如果 spill I/O 压垮存储或导致 slot lag，则需要结合 workload 和内存预算调参。

## 24. 讨论题

1. 为什么 `logical_decoding_work_mem` 不能简单改成“超过就丢弃事务并报错”？

2. `rb->size`、`txn->size`、`toptxn->total_size` 分别回答什么问题？

3. 为什么 `ReorderBufferSerializeTXN()` 可以选择 subtransaction spill，但 `ReorderBufferStreamTXN()` 只能处理 top transaction？

4. 为什么有 toast chunk 或 speculative insert partial change 时，streaming 要暂时跳过该 transaction？

5. `spill_count` 很高但 `spill_txns` 很低，通常说明什么 workload 形态？

6. 为什么 `ReorderBufferRestoreChange()` 必须重建 tuple/snapshot/message 指针，而不是直接使用磁盘里的 `ReorderBufferChange` 内容？

7. 一个 committed 事务因为 database/origin 被 skip，为什么要走 `ReorderBufferForget()` 而不是 `ReorderBufferAbort()`？

8. 如果手工删除运行中 slot 目录下的 `xid-*.spill` 文件，后续最可能在哪条源码路径报错？

## 25. 版本与边界说明

本课基于 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。当前源码有几个需要特别标注的点：

```text
GUC 数据:
  当前主定义在 guc_parameters.dat，
  guc_tables.c 保留枚举 option 表。

restore 函数名:
  当前源码没有 ReorderBufferRestoreTXN()；
  restore TXN 路径由 ReorderBufferRestoreChanges()
  和 ReorderBufferRestoreChange() 组成。

stats 列:
  当前 pg_stat_replication_slots 包含
  mem_exceeded_count、total_txns、total_bytes、slotsync_*。

streaming 行为:
  是否能 streaming 取决于 output plugin callbacks、
  protocol/subscription options 和 snapshot builder 状态。
```

稳定语义是：reorder buffer 必须按 transaction 边界重组 WAL changes；大事务不能无限占内存，必须有 spill 或 streaming slow path；spill file 是 decoded change 的临时中间状态，不是 durable truth；slot stats 是诊断入口，但不是完整因果证明。

## 26. 本节小结

本节主链路：

```text
WAL record
  -> decode.c 生成 ReorderBufferChange
  -> ReorderBufferQueueChange() 挂到 xid/subxid
  -> ReorderBufferChangeMemoryUpdate() 更新 rb/txn/top counters
  -> ReorderBufferCheckMemoryLimit()
     -> streaming 可用则 ReorderBufferStreamTXN()
     -> 否则 ReorderBufferSerializeTXN()
  -> commit/prepare 时 ReorderBufferRestoreChanges() 分批读回
  -> ReorderBufferProcessTXN() 按 LSN 和 snapshot 输出
  -> ReorderBufferCleanupTXN() 清理内存和 spill files
```

核心状态：

```text
rb->size:
  当前 reorder buffer decoded change 内存账本。

txn->size:
  单个 top/sub transaction 当前内存中的 change 大小。

txn->total_size:
  top transaction 汇总大小，用于 streaming 和统计。

txn->nentries / txn->nentries_mem:
  区分总 change 数和当前内存 batch。

RBTXN_IS_SERIALIZED / RBTXN_IS_STREAMED:
  标记事务是否进入过 spill 或 streaming 生命周期。
```

错误路径靠多层收尾：write/read/unlink 失败直接 ERROR；output plugin ERROR 会做 iterator、historic snapshot 和 internal transaction cleanup 后重抛；concurrent abort 在 streaming/prepared 路径中标记 aborted 并等待 abort record；session free 和 startup 会删除 stale `xid-*.spill` 文件。

观测入口是 `pg_stat_replication_slots` 的 `spill_count`、`spill_bytes`、`stream_count`、`stream_bytes`、`mem_exceeded_count`，`pg_stat_activity` 中瞬时 `ReorderBufferRead` / `ReorderBufferWrite` wait event，以及 DEBUG2 log 和 gdb 断点。

可迁移规律：

```text
当系统必须保留完整语义顺序，却面对可能远超内存的大对象时，
不要把“内存上限”做成简单拒绝；
要把状态拆成可记账、可驱逐、可恢复、可观测的生命周期层。
```

对 logical decoding 来说，`logical_decoding_work_mem` 的调优不是追求永不触发阈值，而是在可接受内存峰值、可接受 spill I/O、可接受 streaming/restore 延迟之间找 workload-specific 的平衡点。
