# PostgreSQL Heap insert、FSM 与 page selection

## 课程定位

上一节已经解释了 heap page layout、line pointer 和 tuple header。
这一节把视角推进到“一个 tuple 如何真正落到某个 heap page 上”。
前置知识：
- 已理解 buffer pin 和 buffer content lock 的区别。
- 已理解 heap page 的 `pd_lower`、`pd_upper`、line pointer 和 `t_ctid`。
- 已理解 WAL-before-data、page LSN、full-page write 的基本边界。
- 已理解 visibility map 的 all-visible/all-frozen 是跨 heap page 的辅助状态。
本节唯一主问题：
`heap_insert` 如何在 FSM 可能过时、并发 backend 可能抢走空间、WAL/VM 又要求严格顺序的情况下，低成本选择一个 page 并安全插入 tuple？
本节围绕的核心矛盾：
插入路径希望避免从 relation 头扫到尾，也希望避免每次都追加新 page。
所以它需要一个低成本的 free-space 索引。
但 heap page 的真实可用空间只能在持有该 page 的 exclusive buffer content lock 后才能确认。
FSM 只是近似、可滞后、可丢失的提示状态。
它可以帮忙找候选页，却不能作为插入成功的正确性依据。
PostgreSQL 因此把问题拆成两层：
候选 page selection 追求低成本和局部性；
最终插入决策由 locked heap page 的真实 free space 决定。
读完本节，你应该能独立判断：
- `heap_insert` 为什么不是简单 append。
- `RelationGetBufferForTuple` 为什么必须在 critical section 前完成。
- `smgr_targblock`、`BulkInsertState` 和 FSM 分别缓存什么。
- FSM 返回的 block 为什么必须重新读 page 并验 free space。
- stale FSM 如何被写回真实 free space 并触发 retry。
- 为什么扩 relation 后也可能要重新检查空间。
- tuple bytes、line pointer、VM bit、WAL record 和 page LSN 的更新顺序。
- `MarkBufferDirty` 和 `MarkBufferDirtyHint` 在本节语义上有什么差别。
- abort 后插入 tuple 为什么不是立即物理删除。
- 哪些现象能用 SQL、pageinspect、pg_freespacemap、pg_waldump、gdb 观察。

源码基线：本课使用当前实际源码路径 `/home/highgo/postgres`，branch `master`，commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；核心源码分工见第 3 节。

## 1. 本节在总主线中的位置

前面几节已经建立了持久化主线：
buffer 负责把 page 放进 shared buffers。
WAL 负责 crash 后重放 page change。
checkpoint、bgwriter 和 checkpointer 负责把 dirty page 推向磁盘。
heap page layout 负责在一个 page 内放 line pointer 和 tuple bytes。
这一节连接这些局部模型：
executor 产生一个待插入 tuple；
heap AM 选择一个目标 page；
page 内 `PageAddItem` 分配 line pointer 和 tuple bytes；
WAL 描述这个 page change；
page LSN 把 heap page 和 WAL 顺序绑定；
FSM 记录 page 剩余空间，影响后续插入。
这一节不讨论 index insert。
真实 SQL INSERT 通常还要插 index tuple。
但 page selection 的核心矛盾发生在 heap AM：
先找到 heap page，才能得到 `t_self`/`ctid`，后续 index entry 才有物理 TID。
这一节也不展开 MVCC visibility 判断。
新 tuple 的 `xmin`、`cmin` 和 `xmax` 会被设置。
但本节关心的是：
这些 tuple header 字段如何被放进 page，并如何在 WAL/abort/prune 边界中保持可恢复。

## 2. 核心矛盾与一句话运行模型

唯一主问题可以压缩为一句话：
如何把“低成本找到可能有空间的 page”和“在并发与 crash 下确认插入成功”拆开？
一句话运行模型：

```text
heap_insert 先准备 tuple header/toast，再让 RelationGetBufferForTuple 用 targetBlock、BulkInsertState、FSM、尾页和 relation extension 找候选 page；候选 page 必须被 pin 并持有 exclusive content lock 后重新计算真实 heap free space；确认可插入后进入 critical section，PageAddItem 写 tuple，清 VM all-visible，MarkBufferDirty，写 WAL，设置 page LSN，最后释放 buffer、VM pin、记录 cache invalidation 和 pgstat。
```

关键不是 FSM 找到了哪个 page。
关键是 PostgreSQL 从不把 FSM 当作事实。
FSM 的返回值只意味着“值得试一下”。
真正的事实是 locked heap page 上的 `PageGetHeapFreeSpace(page)`。
如果这个事实否定了 FSM，前台 backend 会把真实 free space 写回 FSM，再继续找。
这形成了本节的核心状态故事：

```text
local tuple
  -> candidate block number
  -> pinned and x-locked heap buffer
  -> real free space check
  -> page tuple insertion
  -> WAL record and page LSN
  -> FSM/VM/cache/stat side effects
```

## 3. 核心文件分工与阅读顺序

建议按下面顺序读，而不是按文件名读。
`src/include/access/tableam.h`
定义 AM 层插入入口。
`table_tuple_insert()` 调用 relation 当前 table AM 的 `tuple_insert`。
它说明 `BulkInsertState` 会被传到 AM，返回时 slot 的 `tts_tid` 会被更新。
`src/backend/access/heap/heapam_handler.c`
把 table AM 入口转到 heap AM。
`heapam_tuple_insert()` 从 slot 取 `HeapTuple`，设置 table oid，调用 `heap_insert()`，再把 `tuple->t_self` 复制回 `slot->tts_tid`。
`src/backend/access/heap/heapam.c`
`heap_insert()` 是本节主入口。
它准备 tuple header/toast，调用 `RelationGetBufferForTuple()` 选页，进入 critical section 插入 tuple，处理 VM、WAL、LSN、buffer release、cache invalidation 和 pgstat。
同一文件还定义 `GetBulkInsertState()`、`FreeBulkInsertState()` 和 `ReleaseBulkInsertStatePin()`。
这些函数解释 bulk insert 的 pin ownership。
`src/backend/access/heap/hio.c`
`RelationGetBufferForTuple()` 是 page selection 的核心。
它处理 target block、FSM search、stale FSM retry、tail-page fallback、relation extension、visibility map pin、lock ordering 和 bulk extension。
`RelationPutHeapTuple()` 是真正把 tuple 放进 page 的 helper。
它要求 caller 已经持有 exclusive buffer lock。
`src/backend/storage/freespace/freespace.c`
这是 relation-level FSM 入口。
`GetPageWithFreeSpace()` 把字节需求转成 category，再搜索 FSM tree。
`RecordAndGetPageWithFreeSpace()` 把一个失败候选页的真实空间写回 FSM，并尽量在同一 FSM page 找下一个候选。
`RecordPageWithFreeSpace()` 只记录空间。
`XLogRecordPageWithFreeSpace()` 是 redo 侧更新 FSM 的入口。
`src/backend/storage/freespace/fsmpage.c`
这是单个 FSM page 内部小树。
每个 leaf slot 是一个 heap block 的 free-space category。
非叶节点保存子树最大值。
`fp_next_slot` 用来分散多个 backend 的搜索起点。
`src/include/storage/fsm_internals.h`
定义 `FSMPageData`、`fp_next_slot`、`fp_nodes` 和每页 slot 数。
这个 header 能帮助你理解 FSM 是“page 内小树 + 多层 FSM page tree”。
`src/backend/storage/buffer/bufmgr.c`
`MarkBufferDirty()` 要求 buffer pinned 且 exclusive-locked。
`MarkBufferDirtyHint()` 用于非关键变化。
FSM 常规更新常用 dirty hint。
heap tuple insert 本身不能用 hint dirty。
`src/include/access/heapam_xlog.h`
定义 `XLOG_HEAP_INSERT`、`XLOG_HEAP_INIT_PAGE`、`xl_heap_insert`、`xl_heap_header`、`XLH_INSERT_*`。
这些字段决定 heap insert WAL record 能重放什么。
`src/backend/access/heap/heapam_xlog.c`
redo 侧的 `heap_xlog_insert()` 和 `heap_xlog_multi_insert()` 重放 tuple insert。
它们在 page 低空间时调用 `XLogRecordPageWithFreeSpace()` 更新 FSM。
这说明 FSM 可以从 heap page redo 结果再推导，并不是每次前台 insert 都必须 WAL-log FSM。

## 4. 关键数据结构与状态

第一类状态是待插入 tuple。
`heapam_tuple_insert()` 从 `TupleTableSlot` 取 `HeapTuple`。
`heap_insert()` 之后，caller 会看到 `t_self` 被设置成真实 TID。
但如果 tuple 被 toast，`heaptup` 可能不是原始 `tup`。
`heap_insert()` 注释明确说：
返回时 `tup->t_self` 会匹配实际存储位置；
但 toasted 字段不会反写到 caller 原 tuple。
第二类状态是 heap page 真实 free space。
真实 free space 不在 FSM 中。
它在 heap page header 的 `pd_lower` 和 `pd_upper` 之间。
`PageGetHeapFreeSpace()` 在 `bufpage.c` 中会基于 `PageGetFreeSpace()` 再扣除 line pointer 约束。
它还检查是否已经达到 `MaxHeapTuplesPerPage`。
所以 heap page 是否能插入，至少包括两件事：
tuple bytes 是否放得下；
是否还能创建或复用 line pointer。
第三类状态是 relation 的 target block。
`RelationGetTargetBlock(relation)` 读取 `relation->rd_smgr->smgr_targblock`。
`RelationSetTargetBlock(relation, block)` 写回这个缓存。
它是 backend-local relcache/smgr 状态。
它不是 shared truth。
它的作用是让同一个 backend 后续 insert 优先尝试刚刚成功的 page。
如果 smgr invalidation 发生，这个 target block 可以被丢弃。
第四类状态是 `BulkInsertStateData`。
`src/include/access/hio.h` 中它包含：

```text
strategy
current_buf
next_free
last_free
already_extended_by
```

`current_buf` 表示 bulk insert 持有的额外 pin。
`next_free..last_free` 记录一次 bulk extension 后尚未用完的新 page 范围。
这些 page 到真正使用时可能已经被别人用掉。
所以使用前仍然必须重新检查。
`already_extended_by` 影响下一次扩 relation 的批量大小。
第五类状态是 FSM fork。
FSM 不是 heap page 内字段。
它是 relation 的 `FSM_FORKNUM`。
每个 heap block 对应 FSM 中一个 slot。
slot 只保存 1 byte category。
在默认 8k BLCKSZ 下，category 大致以 32 bytes 为步长。
`freespace.c` 注释说明最高 category 255 表示至少 `MaxFSMRequestSize`。
这意味着 FSM 记录的是量化后的近似空间。
第六类状态是单个 FSM page 内部小树。
`FSMPageData` 有两个关键字段：

```text
fp_next_slot
fp_nodes[]
```

`fp_nodes` 的 leaf 保存 heap block 的 free-space category。
非叶节点保存子节点最大 category。
所以搜索一个 FSM page 时，可以先看 root 是否满足需求。
如果 root 不够大，整个 FSM page 都不需要继续扫。
`fp_next_slot` 是 round-robin hint。
它可以在 shared lock 下被更新。
源码明确接受它偶尔 garbled。
这是性能优先的 hint，不是正确性状态。
第七类状态是 visibility map pin。
如果目标 heap page 是 all-visible，插入新 tuple 前必须清 heap page 的 `PD_ALL_VISIBLE` 和 VM bit。
清 VM 可能需要读 VM page。
`RelationGetBufferForTuple()` 不能在持有 heap buffer content lock 时做这种 I/O。
所以它在 lock 前尝试 pin VM page，lock 后再复查。
第八类状态是 WAL record 和 page LSN。
heap page 被实际修改后，如果 relation 需要 WAL，`heap_insert()` 注册 heap buffer 和 tuple data，调用 `XLogInsert(RM_HEAP_ID, info)`，然后 `PageSetLSN(page, recptr)`。
这个 LSN 是 buffer flush 判断 WAL-before-data 的关键边界。
第九类状态是 cache invalidation 和 pgstat。
`heap_insert()` 在释放 heap buffer 后调用 `CacheInvalidateHeapTuple()`。
它还调用 `pgstat_count_heap_insert()`。
这些不决定 tuple 是否已经在 page 中。
它们是上层语义和诊断侧效应。

## 5. 主流程源码 walkthrough

先从 AM 入口看。
`table_tuple_insert()` 在 `tableam.h:1457-1463` 调用当前 table AM 的 `tuple_insert`。
heap 的实现是 `heapam_tuple_insert()`。
`heapam_handler.c:149-166` 的流程很短：

```text
ExecFetchSlotHeapTuple()
  -> 设置 table oid
  -> heap_insert()
  -> 把 tuple->t_self 复制到 slot->tts_tid
```

这里的关键状态是 `t_self`。
heap page selection 没完成前，slot 没有最终 TID。
index insert、RETURNING `ctid`、后续 executor 状态都依赖这个结果。
进入 `heap_insert()`。
`heapam.c:2004-2012` 初始化当前事务 XID、`heaptup`、heap buffer、page、`vmbuffer` 和 `all_visible_cleared`。
这时还没有触碰 shared heap page。
`heapam.c:2018` 调用 `AssertHasSnapshotForToast(relation)`。
本节不展开 toast。
但要记住：
大 tuple 可能先被 toast。
`heapam.c:2026` 调用 `heap_prepare_insert()`。
这个函数设置 tuple header：
`xmin` 是当前事务；
`cmin` 是 command id；
`xmax` 清成 0；
`HEAP_XMAX_INVALID` 被设置；
`HEAP_INSERT_FROZEN` 会把 `xmin` 标成 frozen。
如果 tuple 太大或有外部 toast datum，它会调用 toaster。
从这一点开始，`heaptup` 才是要写入 heap page 的 tuple。
`heapam.c:2032-2035` 调用 `RelationGetBufferForTuple()`。
参数里 `otherBuffer` 是 `InvalidBuffer`。
这说明单 tuple insert 不需要同时锁 old page 和 new page。
`options` 会影响是否使用 FSM、是否 frozen、是否 speculative。
`bistate` 可能为空，也可能来自 COPY 或 bulk operation。
`RelationGetBufferForTuple()` 返回的 buffer 已经：
pinned；
exclusive content locked；
真实 free space 已经满足本次需求；
如果需要清 all-visible，对应 VM page 已经被 pin。
这个返回契约非常重要。
`heap_insert()` 后面进入 critical section。
所以所有会 `ereport(ERROR)` 的普通失败都必须在这里之前完成。
`heapam.c:2054` 先做 `CheckForSerializableConflictIn()`。
heap insert 只需要检查 table-level SSI locks。
它不会检查 page gap。
这是 heap 与 index 的差异：
heap page 没有 index gap lock 那类语义。
`heapam.c:2057` 进入 `START_CRIT_SECTION()`。
这行之后到 `END_CRIT_SECTION()` 之间，普通 ERROR 不能出现。
如果出现不可恢复错误，系统会升级处理。
这就是为什么 page selection、oversize tuple、VM pin I/O 都要在 critical section 前完成。
`heapam.c:2059-2060` 调用 `RelationPutHeapTuple()`。
`hio.c:27-33` 写明这个 helper 禁止 `ereport(ERROR)`，失败只能 PANIC。
`hio.c:61` 调用 `PageAddItem()`。
`PageAddItem()` 返回 `offnum`。
`hio.c:65-66` 把 `(block, offnum)` 写到 `tuple->t_self`。
如果不是 speculative insertion，`hio.c:73-78` 把 page 内 tuple header 的 `t_ctid` 设置为自己的 TID。
所以新 tuple 的 TID 在这里诞生。
此时 page 已经被修改。
但 WAL 和 LSN 还没完成。
`heapam.c:2062-2069` 处理 all-visible。
如果 page 原来是 all-visible，插入新 tuple 后它不再能保持这个语义。
代码设置 `all_visible_cleared = true`。
然后清 heap page 的 all-visible flag，并调用 `visibilitymap_clear()` 清 VM bits。
注意：
VM pin 是 `RelationGetBufferForTuple()` 提前准备好的。
这里不能突然去读 VM page。
`heapam.c:2084-2085` 设置 `pd_prune_xid`。
这看起来像 update/delete 相关字段，但 insert 也会设置。
原因有两个：
如果插入事务最后 abort，新 tuple 会变成可 prune 的 DEAD tuple；
如果 page 后续满了，on-access prune/freezing 可能帮助它重新达到 all-visible。
bootstrap 或 frozen insert 不需要这个 hint。
`heapam.c:2087` 调用 `MarkBufferDirty(buffer)`。
`bufmgr.c:3147-3155` 注释要求 buffer pinned 且 exclusive-locked。
heap insert 满足这个条件。
这个 dirty 标记表示 shared buffer 内容已经改变，未来需要写出。
它不是 WAL record。
如果 relation 需要 WAL，`heapam.c:2090` 进入 WAL 分支。
`heapam.c:2092-2096` 准备 `xl_heap_insert`、`xl_heap_header`、`recptr`、`info` 和 buffer flags。
`heapam.c:2102-2103` 对逻辑解码可访问 relation 记录 combo CID。
catalog/逻辑解码需要知道 command id 组合。
这不是 page selection 的核心，但它说明 heap insert WAL 不只服务 physical redo。
`heapam.c:2110-2115` 检查是否是 page 的第一条也是唯一一条 tuple。
如果是，设置 `XLOG_HEAP_INIT_PAGE` 和 `REGBUF_WILL_INIT`。
redo 时可以 reinit 整个 page，而不必依赖旧 page image。
`heapam.c:2117-2123` 填 `xlrec.offnum` 和 flags。
`XLH_INSERT_ALL_VISIBLE_CLEARED` 记录 VM/heap all-visible 被清。
`XLH_INSERT_IS_SPECULATIVE` 记录 speculative insert。
`heapam.c:2130-2137` 对逻辑记录保留 tuple data。
如果 relation 需要 logical logging 且没有 `HEAP_INSERT_NO_LOGICAL`，即使有 full-page image，也要保留 tuple data。
这避免逻辑解码只拿到物理 page image 却拿不到 row。
`heapam.c:2140-2157` 注册 WAL data。
`XLogRegisterData(&xlrec, SizeOfHeapInsert)` 注册 heap insert header。
`XLogRegisterBuffer(0, buffer, REGBUF_STANDARD | bufflags)` 注册被修改的 heap buffer。
`XLogRegisterBufData()` 注册必要 tuple header 字段和 tuple data。
`xl_heap_header` 只保存 `t_infomask2`、`t_infomask`、`t_hoff`。
`xmin` 和 `cmin` 可从 WAL record XID 和 redo 规则恢复。
`heapam.c:2162` 调用 `XLogInsert(RM_HEAP_ID, info)`。
`heapam.c:2164` 用返回的 `recptr` 设置 heap page LSN。
这一步把 page change 和 WAL record 顺序绑定起来。
`heapam.c:2167` 离开 critical section。
从这里开始可以正常 release 和做非关键侧效应。
`heapam.c:2169-2171` 释放 heap buffer 和 VM buffer。
heap buffer release 同时释放 content lock 和 pin。
VM buffer 只是 pin，按需要释放。
`heapam.c:2179` 调用 `CacheInvalidateHeapTuple()`。
注释强调这可以在释放 buffer 后做。
因为 `heaptup` 在 local memory 中，不依赖 shared buffer 内容。
如果事务 abort，catalog cache invalidation 仍需要知道这个 tuple 相关信息。
`heapam.c:2182` 统计 heap insert。
speculative insertion 即使后面 abort，也会被计入。
这个统计口径和最终可见行数不同。
`heapam.c:2188` 以后，如果 `heaptup != tup`，把 `t_self` 复制回原 tuple 并释放 toasted/private copy。
这完成了单 tuple insert 的生命周期。

## 6. Page selection 的真实流程

`RelationGetBufferForTuple()` 是本节最重要的函数。
它的入口注释在 `hio.c:434-498`。
第一句契约是：
返回一个有足够 free space 的 pinned and exclusive-locked buffer。
注意这里说的是“返回时”。
候选来自哪里不重要。
返回前必须拿锁并确认。
`hio.c:518` 先 `MAXALIGN(len)`。
插入判断按对齐后的长度保守计算。
`hio.c:530-534` 如果超过 `MaxHeapTupleSize`，立即 `ereport(ERROR)`。
这在任何 buffer 修改前发生。
这就是错误路径的正确位置。
`hio.c:536-551` 计算 `targetFreeSpace`。
它不是简单的 `len`。
默认路径会考虑 fillfactor：

```text
saveFreeSpace = RelationGetTargetPageFreeSpace(...)
targetFreeSpace = len + saveFreeSpace
```

但大 tuple 和 nearly-empty page 有特殊处理。
如果 `len + saveFreeSpace` 超过 nearly-empty 阈值，目标变成 `Max(len, nearlyEmptyFreeSpace)`。
这避免低 fillfactor 表插入大 tuple 时过度扩 relation。
这一点经常被误读。
fillfactor 不是“insert 永远只填到某个百分比”。
`hio.c:491-494` 注释说明：
这个函数不会把已有 page 填过 fillfactor，除非是 nearly-empty page 上的大 tuple。
same-page update 的 fillfactor 预留空间不靠这个路径维护。
`hio.c:571-574` 首先找 cached target block。
如果有 `bistate->current_buf`，用它的 block。
否则用 `RelationGetTargetBlock(relation)`。
这是一层局部性优化。
它能让同一个 backend 连续插入时重复尝试刚成功的 page。
如果没有 target block 且允许 FSM，`hio.c:576-582` 调 `GetPageWithFreeSpace(relation, targetFreeSpace)`。
这里得到的是候选 block number。
它尚未被锁。
它可能不存在、空间不足、已被其他 backend 用掉，或者 FSM 上层信息过时。
如果 target block 仍是 invalid，`hio.c:590-596` 尝试 relation 最后一页。
注释说这是为了避免 bootstrapping 或刚启动系统里 FSM 为空时出现 one-tuple-per-page syndrome。
这个 fallback 很重要：
FSM 为空不等于 relation 没有可用空间。
进入 `loop` 后，只要有 `targetBlock`，就读 buffer 并加锁。
单 tuple insert 的 easy case 在 `hio.c:614-628`：
`ReadBufferBI()` 读目标 block；
如果 page 看起来 all-visible，先 pin VM；
如果 frozen insert 且 page empty，也先 pin VM；
再 `LockBuffer(buffer, BUFFER_LOCK_EXCLUSIVE)`。
VM pin 的判断在 lock 前读 page flag。
这个判断可能过时。
所以 `hio.c:657-680` 会调用 `GetVisibilityMapPins()` 复查。
如果发现 pin 不够，它会释放 heap lock，做 VM I/O，再重新 lock。
这是一个典型的 PostgreSQL 边界：
为了避免持有 buffer content lock 做 I/O，宁愿承担一次少见的 unlock/relock/recheck。
`hio.c:694-698` 如果 page 是 new page，初始化并 mark dirty。
这个路径一般发生在读到新扩展但未初始化的 page。
`hio.c:700` 计算真实 `pageFreeSpace = PageGetHeapFreeSpace(page)`。
这是 page selection 的事实边界。
`hio.c:701-705` 如果 `targetFreeSpace <= pageFreeSpace`，设置 relation target block 并返回 locked buffer。
此时 caller 可以安全进入 critical section。
如果空间不够，`hio.c:714-722` 释放当前 buffer lock/pin。
然后分三种情况。
第一种：
`hio.c:724-746` 发现 `bistate->next_free` 有尚未使用的 bulk-extended page。
它会先用 `RecordPageWithFreeSpace()` 把失败页真实 free space 记录进 FSM。
然后直接试 bulk extension 范围里的下一个 page。
它不先问 FSM。
原因是这些 page 是本 backend 上次批量扩出来的，局部上更可能适合继续插入。
第二种：
`hio.c:747-750` 如果 `use_fsm` 为 false，跳出循环，准备扩 relation。
这就是 `HEAP_INSERT_SKIP_FSM` 的效果：
当前 target page 不行，就 append/extend。
第三种：
`hio.c:752-761` 正常使用 FSM。
它调用 `RecordAndGetPageWithFreeSpace(relation, targetBlock, pageFreeSpace, targetFreeSpace)`。
这个函数同时做两件事：
把刚失败 page 的真实 free space 写回 FSM；
尽量返回另一个有足够空间的候选页。
这一步保证不会在同一个 stale FSM entry 上无限循环。
如果循环最终没有目标页，`hio.c:765-767` 调 `RelationAddBlocks()` 扩 relation。
扩展返回的 page 通常是新 page，并且希望已经 locked。
但事情仍然没有结束。
`RelationAddBlocks()` 可能为了把额外 page 录入 FSM 而释放 first page 的 lock。
`hio.c:858-870` 因此必须再次检查 free space。
如果 page 在 unlock 窗口被其他 backend 用掉，并且确实不够放当前 tuple，就释放并 `goto loop`。
如果没有 unlock 却不够，那违反前面 oversize 检查和新 page 假设，只能 PANIC。
`hio.c:872-881` 对新 page 也设置 relation target block。
注释里有一个非常实际的取舍：
新 page 不会马上录入 FSM。
它先留给当前 backend 作为短期插入目标。
这降低其他 backend 立刻抢同一页造成的 contention。

## 7. FSM 搜索、更新和近似性

`GetPageWithFreeSpace()` 在 `freespace.c:136-142` 很简单。
它把需要的 bytes 转成 category，然后调用 `fsm_search()`。
转换在 `freespace.c:441-459`。
`fsm_space_needed_to_cat()` 对需求向上取整。
记录实际空间时，`fsm_space_avail_to_cat()` 在 `freespace.c:401-420` 向下取整。
这组方向很重要：
需求向上，记录向下。
这样 FSM 不会因为量化而乐观承诺一个实际放不下的 category。
但并发和滞后仍然会让它过时。
`RecordAndGetPageWithFreeSpace()` 在 `freespace.c:153-184`。
它先定位 old heap block 对应的 FSM address 和 slot。
然后调用 `fsm_set_and_search()`。
如果同一个 FSM page 内能找到合适 slot，就返回那个 heap block。
否则再从 root 调 `fsm_search()`。
这是一种局部性优化：
失败页附近可能还有合适页，先少走一点树。
`RecordPageWithFreeSpace()` 在 `freespace.c:193-204`。
注释提醒：
如果新 free space 比旧值更高，空间可能要等下一次 `FreeSpaceMapVacuum` 更新上层 page 后才对搜索者可见。
这再次说明 FSM 不是强一致索引。
`fsm_set_and_search()` 在 `freespace.c:655-681`。
它读 FSM page，exclusive lock，调用 `fsm_set_avail()` 更新 slot。
如果 page 改了，用 `MarkBufferDirtyHint(buf, false)`。
然后可选地在同一个 page 内搜索。
`MarkBufferDirtyHint()` 是关键：
FSM 常规变化是 hint-like。
丢了不破坏 heap correctness。
最坏后果是后续插入多走 retry 或多扩 page。
`fsm_search()` 在 `freespace.c:687-805`。
它从 `FSM_ROOT_ADDRESS` 开始。
每层读一个 FSM page，shared lock，调用 `fsm_search_avail()`。
如果找到 slot：
不是 bottom level 就 descend；
是 bottom level 就转回 heap block number，并调用 `fsm_does_block_exist()` 验证是否仍在 relation 内。
如果 heap block 已经超过 relation 末尾，说明 FSM stale。
代码把该 slot 置 0，dirty hint，然后从 root 重启。
如果某个下层 page 找不到满足需求的 slot，但上层曾承诺有空间，说明上层过时。
代码把该 page 的 `max_avail` 写回 parent，然后从 root 重启。
源码给了 emergency valve：
restart 超过 10000 次返回 `InvalidBlockNumber`。
这不是正常性能路径。
它是“即使 FSM 坏得离谱，也不要无限循环”的 fallback。
`fsm_search_avail()` 在 `fsmpage.c:157-315`。
单个 FSM page 内部不是线性扫描。
它先看 root `fp_nodes[0]`。
如果 root 小于 `minvalue`，立即返回 -1。
否则从 `fp_next_slot` 开始，向右扩展搜索三角形，再下降到 leaf。
这样在一个 page 内接近 `log2(N)`。
搜索成功后，它更新 `fp_next_slot`。
如果只有 shared lock，也可能通过 hint bit 机制更新。
源码明确说：
为了避免 exclusive lock 的并发损失，偶尔 garbled next pointer 是可以接受的。
如果 page 内小树看起来 corrupt，例如父节点承诺有空间但两个子节点都没有，`fsmpage.c:263-288` 会 rebuild page。
日志级别是 `DEBUG1`。
这类修复服务的是 FSM 自身一致性。
即使 FSM 不准，heap page insert 的正确性仍靠 locked page recheck。

## 8. Relation extension 与 bulk insert 状态

当 cached target、FSM、尾页都不能提供候选，`RelationGetBufferForTuple()` 调 `RelationAddBlocks()`。
`RelationAddBlocks()` 在 `hio.c:235-430`。
它不是单纯扩 1 页。
如果 caller 知道需要多页，`num_pages` 可以大于 1。
`heap_multi_insert()` 会利用这一点。
`hio.c:251-305` 决定扩多少页。
如果没有 `bistate` 且不使用 FSM，只能扩 1 页。
否则可以至少扩 `num_pages`。
还会根据 relation extension lock waiter 数增加扩展量。
这是一种 contention 缓解：
如果很多 backend 都在等扩 relation，一次多扩一些，减少大家反复抢 extension lock。
扩展量还受 `MAX_BUFFERS_TO_EXTEND_BY` 限制。
当前上限是 64。
因为这些新 page 的 buffer 会被临时 pin。
扩太多会把局部优化变成 buffer 资源压力。
`hio.c:308-321` 决定哪些新 page 进入 FSM。
如果有 `bistate`，当前 backend 自己马上要用的 page 不急着放进 FSM。
否则其他 backend 会立刻过来抢。
如果没有 `bistate`，就只能通过 FSM 让额外 page 被未来 insert 找到。
`hio.c:339-344` 调 `ExtendBufferedRelBy()`。
它要求第一个返回 page locked。
`hio.c:355-362` 检查 page 必须是 new，然后 `PageInit()` 并 `MarkBufferDirty()`。
如果新 page 不是 empty，这是 ERROR。
这发生在 actual tuple insert 前。
所以仍可普通报错。
如果决定把额外 page 放进 FSM，`hio.c:369-375` 会释放 first page 的 lock。
原因仍然是：
不要持有 buffer content lock 做 FSM I/O。
释放后 caller 必须重新检查。
`hio.c:382-397` 释放除 first page 外的 buffer pin。
如果 page 应该进 FSM，就用 `RecordPageWithFreeSpace()` 记录它们大约有 `BLCKSZ - SizeOfPageHeaderData` 空间。
`hio.c:400-405` 对范围调用 `FreeSpaceMapVacuumRange()`。
这会推进 FSM 上层 page 的 max 信息。
否则 leaf 更新可能暂时不会被 root 搜索看到。
如果有 `bistate`，`hio.c:407-427` 记录 `next_free`、`last_free`、`current_buf` 和 `already_extended_by`。
`current_buf` 会额外 pin first page。
`FreeBulkInsertState()` 在 `heapam.c:1953-1959` 释放这个 pin 和 bulk write strategy。
`ReleaseBulkInsertStatePin()` 在 `heapam.c:1965-1982` 还会重置 `next_free` 和 `last_free`。
注释给出真实原因：
如果同一个 bulk state 被不同 partition 复用，旧 partition 的 `next_free` 会让新 partition 查错 page。
这正是 ownership 和 relation identity 不能混淆的例子。

## 9. 生命周期 / ownership / cleanup

待插入 tuple 的 owner 是当前 backend。
`heap_prepare_insert()` 可能返回原 tuple，也可能返回 toaster 产生的 private copy。
`heap_insert()` 在末尾判断 `heaptup != tup`。
如果是 private copy，就把 `t_self` 复制回原 tuple，然后 `heap_freetuple(heaptup)`。
shared heap page 的 owner 不是当前 backend。
backend 只持有 buffer pin 和 content lock。
`RelationGetBufferForTuple()` 返回时 pin 和 exclusive lock 都归 caller。
`heap_insert()` 在 critical section 后用 `UnlockReleaseBuffer(buffer)` 释放。
如果中途在 critical section 前 ERROR，ResourceOwner 会释放已登记的 buffer pin/lock。
如果已经进入 critical section，普通 ERROR 不被允许。
这就是 `RelationPutHeapTuple()` 失败要 PANIC 的原因。
VM buffer 的 ownership 更窄。
它只是 pin，用来保证 `visibilitymap_clear()` 不在持 heap lock 时做 I/O。
`heap_insert()` 在 heap buffer release 后释放 `vmbuffer`。
如果目标 page 并非 all-visible，`vmbuffer` 可能仍是 `InvalidBuffer`。
FSM buffer 的 ownership 在 freespace functions 内部闭合。
`GetPageWithFreeSpace()`、`RecordAndGetPageWithFreeSpace()`、`RecordPageWithFreeSpace()` 自己读、锁、更新、释放 FSM buffer。
heap insert caller 不持有 FSM buffer 跨函数返回。
BulkInsertState 的 owner 是上层 bulk operation。
COPY、CTAS、materialized view refresh 等会创建它。
谁创建谁释放。
`heap_insert()` 只使用它，不释放它。
如果 `bistate->current_buf` 有额外 pin，上层必须最终 `FreeBulkInsertState()`。
事务 abort 时，已经插入的 heap tuple 通常仍留在 page 上。
它的 inserting XID abort 后，对正常 snapshot 不可见。
物理空间由 pruning 或 vacuum 后续回收。
`pd_prune_xid` 的设置就是为了让后续 page access 有机会发现这类 abort tuple。
cache invalidation 的生命周期不是 buffer 生命周期。
`CacheInvalidateHeapTuple()` 可以在释放 buffer 后做。
因为它使用 local `heaptup`。
事务结束时 invalidation 的投递/撤销按 cache invalidation 机制处理。
本节只需要记住：
catalog tuple insert 即使 abort，也要让 cache 侧能收尾。

## 10. 正确性机制层次

第一层是 relation-level 语义锁。
SQL INSERT 的调用者通常已经持有 relation 的合适 lock，例如 RowExclusiveLock。
这保证 DDL/DML 语义边界。
它不保护某个 heap page 的内存并发修改。
第二层是 buffer pin。
pin 保证 buffer 不会在使用期间被驱逐或重用。
pin 不保证 page 内容稳定。
读写 page 内容还需要 content lock。
第三层是 buffer content lock。
`RelationGetBufferForTuple()` 返回 exclusive-locked buffer。
`RelationPutHeapTuple()` 要求 caller 持有 exclusive lock。
`MarkBufferDirty()` 也断言 pinned 且 exclusive-locked。
这层保证 page header、line pointer 和 tuple bytes 的内存修改互斥。
第四层是真实 free-space recheck。
FSM、targetBlock、`bistate->next_free` 都只是候选来源。
真正保证 `PageAddItem()` 不失败的是：
在 exclusive lock 下调用 `PageGetHeapFreeSpace()` 并确认空间足够。
第五层是 VM invariant。
如果 heap page 被标 all-visible，插入新 tuple 会破坏这个语义。
所以 critical section 中必须清 heap page flag 和 VM bit。
VM pin 必须提前准备，避免持 heap page lock 做 I/O。
第六层是 WAL/LSN。
`heap_insert()` 在修改 page 后先 `MarkBufferDirty()`。
然后注册 WAL data、调用 `XLogInsert()`、设置 page LSN。
实际 WAL-before-data 的强制发生在 dirty buffer 被写出前：
buffer manager/checkpointer 会确保 page LSN 对应的 WAL 已经 flush。
heap insert 自己负责把正确 LSN 放到 page 上。
第七层是 redo contract。
`xl_heap_insert` 记录 offnum、flags 和必要 tuple header/data。
如果是 init page，redo 可以重新初始化整个 page。
如果清了 all-visible，redo 也必须修 VM。
redo 侧还会在低 free space 时更新 FSM。
第八层是 Serializable conflict。
`CheckForSerializableConflictIn()` 在 actual insert 前做。
heap insert 只检查 table-level predicate lock。
它不使用 heap page gap lock。
这层保证 SSI 语义，不保证 page memory safety。
第九层是 FSM 非正确性边界。
FSM 不参与 tuple visibility。
FSM 不保证 page 一定有空间。
FSM 不需要和 heap insert WAL record 一一对应。
FSM 错了最多导致 retry、额外 search、额外 extension 或低效空间利用。

## 11. 错误路径 / 异常路径 / fallback

第一个错误路径是 tuple 过大。
`RelationGetBufferForTuple()` 在任何 buffer 修改前检查 `len > MaxHeapTupleSize`。
这会普通 `ereport(ERROR)`。
调用者看到 SQL 错误。
shared buffer 中不会留下半个 tuple。
第二个 fallback 是没有 cached target。
先问 FSM。
FSM 没有答案时试 relation 最后一页。
最后一页也不行才扩 relation。
这避免 FSM 为空时每条 tuple 都扩新 page。
第三个 fallback 是 FSM stale。
FSM 返回的 page 真实空间不足。
前台 backend 释放 heap buffer，把真实 `pageFreeSpace` 写回 FSM，然后继续搜索。
这是本节最重要的 retry loop。
第四个 fallback 是 FSM 指向 relation 末尾之后。
`fsm_search()` 会检查 block 是否存在。
如果不存在，把 slot 置 0 并从 root 重启。
这可能发生在 truncation、crash/recovery 或过时 FSM 信息之后。
第五个 fallback 是 FSM 上下层不一致。
下层找不到上层承诺的空间时，`fsm_search()` 把下层 `max_avail` 写回 parent，再从 root 重启。
超过 emergency restart 次数就返回 `InvalidBlockNumber`。
调用者随后扩 relation。
第六个 fallback 是 FSM page 内小树 corrupt。
`fsm_search_avail()` 发现父子节点不一致时，会 `fsm_rebuild_page()` 并 DEBUG1 记录。
之后 restart。
这处理的是 FSM 自身 torn/corrupt 情况。
第七个 fallback 是 VM pin 过时。
`RelationGetBufferForTuple()` lock 前看 all-visible，lock 后复查。
如果发现需要 VM pin 但没有，它会释放 heap locks，pin VM page，再重新 lock。
重新 lock 后还要重查 free space。
第八个 fallback 是 relation extension 后 lock 被释放。
为了把额外 page 写进 FSM，`RelationAddBlocks()` 可能释放 first page lock。
重新获得 lock 后，如果空间已被别人用掉，就 `goto loop`。
这说明“新扩 page”也只是候选，不能免除 recheck。
第九个异常路径是 `PageAddItem()` 失败。
这发生在 critical section 内。
`RelationPutHeapTuple()` 只能 PANIC。
因为 caller 已经确认空间并禁止普通 ERROR。
如果这里失败，说明 page selection 或 page state 出现了不应发生的不一致。
第十个特殊路径是 `HEAP_INSERT_SKIP_FSM`。
它跳过 FSM。
当前 target page 不行就扩 relation。
rewrite/unlogged additions 等场景会用这个选项。
正确性依赖 caller 持有足够 relation lock，并且处理 `smgr_targblock`，避免把未 WAL 保护的新数据和旧提交数据混在一起。
第十一个特殊路径是 relation 不需要 WAL。
`RelationNeedsWAL(relation)` 为 false 时，heap page 仍会被修改、mark dirty、设置 tuple header。
但不会写 heap WAL record，也不会设置来自 insert record 的 page LSN。
这适用于 unlogged/temp 或特定 rewrite 场景。
crash 后语义由 relation 类型决定，不是由 FSM 保证。

## 12. 成本、资源与跨模块传播

CPU 成本的第一部分是 page selection。
最佳情况只尝试 cached target block。
成本接近一次 buffer lookup、一次 content lock、一次 `PageGetHeapFreeSpace()`。
如果 cached target 失效，就走 FSM。
FSM 搜索成本与 FSM tree depth 和单 page 内小树搜索有关。
它不是随 heap relation blocks 线性增长。
但 stale FSM 会导致多次 restart。
CPU 成本的第二部分是 tuple preparation。
`heap_prepare_insert()` 可能触发 toast。
toast 会产生额外 heap insert 到 toast table。
本节主链路仍成立，只是 relation 变成 toast relation，WAL 和 FSM 也作用于 toast heap。
contention 成本的第一部分是 buffer content lock。
大量 backend 同时插同一 relation，cached target、FSM round-robin、bulk extension 都在减少大家撞同一 page 的概率。
但最终每个 page 修改仍需要 exclusive content lock。
contention 成本的第二部分是 relation extension。
当没有 page 可用时，backend 需要扩 relation。
`RelationAddBlocks()` 会根据 extension lock waiter 数增加扩展页数。
这把很多小扩展合并成较少的大扩展。
但它会临时 pin 多个 buffers，所以有上限。
WAL 成本随 tuple bytes、full-page writes、logical logging 需求扩张。
如果 page 第一次修改发生在 checkpoint 后，full-page write 可能放大 WAL。
如果 relation logically logged，heap insert 要保留 tuple data，即使存在 FPW。
这会把前台 latency、WAL bandwidth、replication lag 和 checkpoint pressure 连起来。
FSM 成本是空间利用和插入延迟之间的折中。
记录每次准确空间、强制同步上层树，会提高 insert hot path 成本。
允许 FSM 近似和滞后，会带来 retry 和偶尔额外 extension。
PostgreSQL 选择让 correctness 落在 heap page recheck 上，让 FSM 保持低成本。
VM 成本体现在 all-visible page 被插入时。
插入会清 VM bit。
这会影响 index-only scan 机会。
也会让 autovacuum/pruning/freezing 之后重新设置 VM bit。
所以一个 insert 不只改变 heap page，还改变未来查询路径的可用优化。
后台进程参与状态推进：
checkpointer 负责把 dirty heap/FSM/VM page 刷到磁盘并保证 WAL 顺序。
bgwriter 可以提前写 dirty buffers，缓解前台 eviction 写压力。
walwriter 推进 WAL flush，影响 commit 和 page flush 等待。
autovacuum/vacuum 会 prune/free space，并更新 FSM 与 VM。
startup process 在 crash recovery 中重放 heap insert WAL，并可能更新 FSM。
资源传播路径可以这样记：

```text
tuple size -> page free space -> FSM search/retry -> relation extension
tuple bytes -> WAL bytes -> walwriter/replication/checkpoint pressure
all-visible page insert -> VM clear -> index-only scan opportunity decrease
many backends -> page/contention + extension lock wait -> bulk extension heuristic
abort insert -> dead tuple -> pruning/vacuum -> FSM update
```

## 13. 观测与诊断入口

本节锚定的 runtime truth 是：
FSM 记录的 free space 可以和 heap page 的真实可用空间不一致；
insert 路径必须通过 locked page recheck 把候选变成事实。
能直接观察的状态：
- 新插入行的 `ctid`，可以看到 block 分布。
- heap page 上 line pointer 和 tuple header，可用 `pageinspect` 看。
- FSM 记录的 per-block free space，可用 `pg_freespacemap` 看。
- WAL record 类型和 flags，可用 `pg_waldump` 看。
- query 级 buffer/WAL 用量，可用 `EXPLAIN (ANALYZE, BUFFERS, WAL)` 看。
- instance 级 WAL 量，可用 `pg_stat_wal` 看。
只能推断的状态：
- `RelationGetTargetBlock()` 当前 cached target。
- `BulkInsertState.current_buf`、`next_free`、`last_free`。
- `RelationGetBufferForTuple()` 在一个 SQL 内 retry 了几次。
- FSM search restart 次数。
- VM pin 是否发生了 unlock/relock。
这些通常需要 gdb、tracepoint、临时日志或源码计数器。
几乎不可见或不宜直接依赖的状态：
- `fp_next_slot` 的瞬时值。
- 某个 backend 的 local relcache/smgr target block。
- 一个 FSM hint update 是否已经持久化到磁盘。
这些状态对诊断有帮助，但不能作为 SQL 层稳定语义。
SQL 观察入口示例：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

CREATE TABLE heap_insert_fsm_lab (
  id int,
  payload text
) WITH (fillfactor = 70, autovacuum_enabled = off);

INSERT INTO heap_insert_fsm_lab
SELECT g, repeat('x', 100)
FROM generate_series(1, 1000) AS g;

SELECT ctid, id
FROM heap_insert_fsm_lab
ORDER BY ctid
LIMIT 20;

SELECT *
FROM pg_freespace('heap_insert_fsm_lab')
ORDER BY blkno
LIMIT 20;

SELECT lp, lp_flags, lp_len, t_xmin, t_ctid
FROM heap_page_items(get_raw_page('heap_insert_fsm_lab', 0));
```

用 `ctid` 看 block 分布。
用 `pg_freespace` 看 FSM 记录。
用 `heap_page_items` 看 page 内 line pointer 和 tuple header。
不要期望三者每一刻完全一致。
FSM 是近似和滞后的。
WAL 观察入口：

```bash
pg_waldump --rmgr=Heap --path="$PGDATA/pg_wal" <start-seg>
```

关注 `INSERT`、`MULTI_INSERT`、`INIT_PAGE` 和 flags。
如果使用 logical logging，注意 heap insert record 可能保留 tuple data。
性能观察入口：

```sql
EXPLAIN (ANALYZE, BUFFERS, WAL)
INSERT INTO heap_insert_fsm_lab
SELECT g, repeat('y', 100)
FROM generate_series(1001, 2000) AS g;
```

`BUFFERS` 能看到 shared hit/read/dirtied/written。
`WAL` 能看到 record、FPI 和 bytes。
它看不到 FSM retry 次数。
如果怀疑 retry 或 extension contention，要用源码断点或临时计数器。
建议 gdb 断点：

```text
heap_insert
RelationGetBufferForTuple
RecordAndGetPageWithFreeSpace
RelationAddBlocks
RelationPutHeapTuple
XLogInsert
PageSetLSN
```

诊断时区分三类原因：
workload 让 tuple 太大或 fillfactor 太低；
concurrency 让很多 backend 抢同一 relation/page；
kernel path 因 FSM stale、VM pin、extension lock 或 WAL flush 增加 latency。
不要把所有 INSERT 慢都解释成 FSM 问题。

## 14. 常见误区

误区一：
FSM 决定 tuple 放在哪个 page。
更准确地说：
FSM 只给候选。
locked heap page 的真实 free space 才决定能不能放。
误区二：
FSM 不准会造成 correctness bug。
通常不会。
FSM 不准导致 retry、空间利用变差或多扩 page。
tuple visibility、page memory safety 和 crash safety 不靠 FSM。
误区三：
fillfactor 是硬限制。
不是。
`RelationGetBufferForTuple()` 会尽量保留 `saveFreeSpace`。
但 nearly-empty page 上的大 tuple 可以越过简单的 fillfactor 直觉。
same-page update 的预留空间也不是通过 insert path 强制维护。
误区四：
new page 一定不会被别人抢。
不一定。
如果扩 relation 后为了 FSM 或 VM pin 释放过 lock，其他 backend 可能先用掉空间。
所以代码必须 recheck。
误区五：
`MarkBufferDirtyHint()` 和 `MarkBufferDirty()` 只是名字不同。
不是。
heap tuple insert 是关键 page change，必须 `MarkBufferDirty()` 并配合 WAL/LSN。
FSM 常规更新可用 hint dirty，因为丢失不破坏 heap correctness。
误区六：
插入事务 abort 会把 tuple 立刻从 page 删除。
不是。
abort 后 tuple 变成不可见/可清理状态。
物理空间由 pruning 或 vacuum 回收。
这也是 insert 设置 `pd_prune_xid` 的原因。
误区七：
`pg_stat_*` 能直接告诉你 page selection 选了哪个 page。
不能。
`pg_stat_wal`、`pg_stat_io`、`EXPLAIN BUFFERS/WAL` 给的是聚合或 query 级现象。
page selection 细节通常要用 `ctid`、pageinspect、pg_freespacemap、gdb 或源码计数器拼出来。

## 15. 课堂实验

实验一：观察 `ctid`、heap page 和 FSM 记录的差异。
目标：
看到 insert 后行落在哪些 block，FSM 记录是什么，page 内 line pointer 是什么。
步骤：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;

DROP TABLE IF EXISTS heap_insert_fsm_lab;
CREATE TABLE heap_insert_fsm_lab (
  id int,
  payload text
) WITH (fillfactor = 70, autovacuum_enabled = off);

INSERT INTO heap_insert_fsm_lab
SELECT g, repeat('x', 100)
FROM generate_series(1, 1000) AS g;
```

观察 block 分布、FSM 与 page 内 line pointer：

```sql
SELECT split_part(trim(both '()' from ctid::text), ',', 1)::int AS blk,
       count(*) AS tuples
FROM heap_insert_fsm_lab
GROUP BY 1
ORDER BY 1;

SELECT *
FROM pg_freespace('heap_insert_fsm_lab')
ORDER BY blkno
LIMIT 20;

SELECT lp, lp_flags, lp_len, t_xmin, t_ctid
FROM heap_page_items(get_raw_page('heap_insert_fsm_lab', 0));
```

解释要求：
把 `ctid` block 分布和 `pg_freespace` 记录对上。
再用 pageinspect 看 block 0 的 line pointer。
说明为什么 `pg_freespace` 的值不是 page truth 的唯一来源。
回到源码：
`RelationGetBufferForTuple()` 用 `GetPageWithFreeSpace()` 找候选。
但在 `hio.c:700-705` 重新检查 page。
实验二：制造 dead tuple，然后观察 FSM 何时变得更有用。
目标：
看到 abort/delete 后的空间不一定立刻变成 FSM 可搜索空间，VACUUM/prune 会推进记录。
步骤：

```sql
DROP TABLE IF EXISTS heap_insert_fsm_reuse;
CREATE TABLE heap_insert_fsm_reuse (
  id int,
  payload text
) WITH (fillfactor = 100, autovacuum_enabled = off);

INSERT INTO heap_insert_fsm_reuse
SELECT g, repeat('x', 200)
FROM generate_series(1, 500) AS g;

DELETE FROM heap_insert_fsm_reuse
WHERE id <= 250;

SELECT *
FROM pg_freespace('heap_insert_fsm_reuse')
ORDER BY blkno
LIMIT 20;

VACUUM heap_insert_fsm_reuse;

SELECT *
FROM pg_freespace('heap_insert_fsm_reuse')
ORDER BY blkno
LIMIT 20;
```

解释要求：
DELETE 只改变 tuple visibility/header。
空间能否进入 FSM，取决于 pruning/vacuum 对 page 的实际清理和记录。
回到源码：
本节 insert path 只在发现候选页不适合或 bulk extension 时记录 FSM。
更系统的 free-space 发现由 vacuum/prune 相关路径推进。
实验三：用断点看 page selection 和 WAL/LSN 边界。
目标：
亲手确认候选页、locked page recheck、`PageAddItem`、`XLogInsert`、`PageSetLSN` 的时间顺序。
建议断点：

```text
b heap_insert
b RelationGetBufferForTuple
b RecordAndGetPageWithFreeSpace
b RelationPutHeapTuple
b XLogInsert
b PageSetLSN
```

观察变量：

```text
heaptup->t_len
targetBlock
targetFreeSpace
pageFreeSpace
BufferGetBlockNumber(buffer)
heaptup->t_self
all_visible_cleared
recptr
```

解释要求：
画出一次 insert 的状态线：
candidate block 何时出现；
真实 free space 何时确认；
`t_self` 何时写入；
WAL record 何时产生；
page LSN 何时设置。
如果命中 `RecordAndGetPageWithFreeSpace()`，说明 FSM 候选页被真实 page recheck 否定。
这正是本节核心现象。

## 16. 讨论题

1. 为什么 PostgreSQL 不直接在每次 insert 时从 relation 尾页开始线性向前找 free space？
2. 为什么 FSM 返回的 block number 不能直接作为插入成功的依据？
3. `RelationGetBufferForTuple()` 为什么必须在 `START_CRIT_SECTION()` 之前完成所有可能 `ereport(ERROR)` 的工作？
4. 如果 `RecordPageWithFreeSpace()` 把一个 page 的空间记大了，后续 insert 如何保持 correctness？
5. 为什么扩 relation 后还要重新检查 free space？
6. 为什么 heap insert 清 VM bit 要提前 pin VM page，而不是持 heap buffer lock 时再读 VM page？
7. `MarkBufferDirtyHint()` 可以用于 FSM，却不能替代 heap page insert 的 `MarkBufferDirty()`，边界在哪里？
8. `pg_freespace`、`ctid`、`heap_page_items` 分别能看到什么，又分别看不到什么？

## 17. 本节小结

本节唯一主问题是：
`heap_insert` 如何在 FSM 可能过时、并发可能抢空间、WAL/VM 又要求严格顺序的条件下选 page 并插入 tuple。
核心链路是：
`table_tuple_insert()` 进入 heap AM；
`heap_insert()` 准备 tuple；
`RelationGetBufferForTuple()` 用 target block、bulk state、FSM、尾页和 extension 找候选；
locked page 上用 `PageGetHeapFreeSpace()` 确认真相；
`RelationPutHeapTuple()` 用 `PageAddItem()` 写 line pointer 和 tuple；
VM bit 被按需清除；
heap page 被 mark dirty；
WAL record 被插入；
page LSN 被设置；
buffer 和 VM pin 被释放；
cache invalidation 和 pgstat 完成侧效应。
核心状态边界是：
FSM 是近似候选索引；
heap page 是事实来源；
buffer pin 保证对象存在；
content lock 保证 page 修改互斥；
WAL/LSN 保证 crash recovery 顺序；
VM 保证 visibility map 与 heap page flag 的一致性。
ownership 和 cleanup 的重点是：
tuple copy 属于当前 backend；
heap buffer lock/pin 属于 `heap_insert()` 的 critical path；
FSM buffer 在 freespace 函数内部闭合；
BulkInsertState 由上层 bulk caller 创建和释放；
abort 后 tuple 物理空间由后续 pruning/vacuum 回收。
错误和 fallback 的重点是：
oversize tuple 在修改前 ERROR；
FSM stale 时写回真实 free space 并 retry；
FSM 不一致时 repair/restart；
没有候选时扩 relation；
扩展后如果 lock 曾释放仍要 recheck；
critical section 内 `PageAddItem` 失败只能 PANIC。
观测上：
`ctid`、pageinspect、pg_freespacemap、EXPLAIN BUFFERS/WAL、pg_waldump 能拼出 runtime 现象。
但 targetBlock、bulk state、FSM restart、VM unlock/relock 通常只能通过断点、trace 或临时计数器看到。
可迁移的系统规律是：
高频写入路径常把“低成本近似索引”和“持锁事实验证”拆开。
近似状态负责减少搜索成本；
真实对象上的 lock、WAL、LSN 和 cleanup 负责 correctness。
判断 INSERT 性能时仍要标注边界：
tuple width、fillfactor、backend 数、WAL bandwidth、checkpoint、autovacuum、relation size、版本实现都会改变瓶颈位置。
