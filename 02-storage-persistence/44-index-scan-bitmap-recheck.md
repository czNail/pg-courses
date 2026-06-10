# PostgreSQL Index scan、bitmap scan 与 recheck
## 课程定位
本节主题：Index scan、Index-only scan、Bitmap index/heap scan 与 recheck。
前置知识：
- 已理解 heap tuple version、HOT chain 和 MVCC snapshot。
- 已理解 buffer pin 与 buffer content lock 的区别。
- 已理解 visibility map 只是一种可滞后的 page-level hint。
- 已理解 B-tree scan 返回 heap TID，而不是直接完成 SQL 语义。
- 已理解 executor node 通过 `ExecScan()` 把 access method 返回的 tuple 再交给 qual/projection。
本节唯一主问题：
当索引访问路径只提供“候选 TID 或候选 page”而不能独立证明 SQL qual 与 MVCC visibility 时，PostgreSQL 如何在普通 index scan、index-only scan 和 bitmap heap scan 中传播、复验并观测这种不确定性？
本节围绕的核心矛盾：
索引访问必须快。
它希望只按 index key 定位 TID、尽量少碰 heap、尽量少加锁、尽量少保存中间状态。
但 PostgreSQL 的正确性最终取决于 heap tuple visibility、HOT chain、page-level VM bit、AM-specific lossy semantics 和 executor 原始 qual。
如果每个索引命中都强制做完整 heap 语义检查，许多 index-only 和 bitmap 场景会失去价值。
如果完全相信索引返回结果，又会在 lossy opclass、lossy bitmap、stale VM bit、HOT update 或 MVCC 变化下返回错误 tuple。
PostgreSQL 的折中是：
索引 AM 返回候选位置。
table AM 负责把候选位置解释成当前 snapshot 下的 tuple。
executor 持有原始 qual，并在 `xs_recheck` 或 bitmap `recheck` 为真时复验。
visibility map 只允许 index-only scan 跳过 heap visibility 检查，不允许跳过必要的 qual recheck。
bitmap scan 可以把大量 TID 压缩成 page-level lossy 表示，但必须把 `lossy` 和 `recheck` 传到 heap scan。
学完本节，你应该能独立判断：
- 为什么 `Index Cond` 不等于最终返回条件。
- 为什么 `Rows Removed by Index Recheck` 不是普通 `Filter`。
- 为什么 bitmap heap scan 的 `Recheck Cond` 总是显示，但不代表每个 tuple 都真的重算。
- 为什么 index-only scan 仍可能出现 `Heap Fetches`。
- 为什么 visibility map bit 允许无锁读取，却仍能保持 index-only scan 正确。
- 为什么 bitmap lossy page 必须扫描整页可见 tuple。
- 为什么 exact bitmap page 也可能要求 recheck。
- 为什么 bitmap index scan 不走普通 `ExecProcNode` tuple-at-a-time convention。
- 为什么 `work_mem` 变化会改变 bitmap exact/lossy 形态，而不是改变逻辑查询结果。
- 为什么 recheck 是 correctness boundary，不是优化器的可选装饰。
## 源码基线
源码仓库：
```text
/home/nail/postgres-lab
```
基线：
```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```
本节重点阅读：
```text
src/backend/executor/nodeIndexscan.c
src/backend/executor/nodeIndexonlyscan.c
src/backend/executor/nodeBitmapIndexscan.c
src/backend/executor/nodeBitmapHeapscan.c
src/backend/access/index/indexam.c
src/backend/access/heap/heapam_handler.c
src/backend/access/heap/visibilitymap.c
```
辅助阅读：
```text
src/include/nodes/execnodes.h
src/include/access/relscan.h
src/include/access/tableam.h
src/include/access/heapam.h
src/include/nodes/tidbitmap.h
src/backend/nodes/tidbitmap.c
src/backend/access/heap/heapam_indexscan.c
src/backend/commands/explain.c
src/include/executor/instrument_node.h
```
辅助文件不是本节主角。
它们只用于解释 executor 状态、table AM 边界、TIDBitmap exact/lossy 表示和 EXPLAIN 观测入口。
## 1. 本节在总主线中的位置
前面几节已经从 heap tuple version、HOT、B-tree search 和 index tuple cleanup 讲到存储层如何产生 TID。
现在进入 executor 读路径。
读路径面对的问题不是“索引能不能找到一个 key”。
真正问题是：
索引找到的 TID 只是候选事实。
候选事实必须被 heap snapshot、HOT chain 和 executor qual 重新解释。
普通 index scan 的对象是：
```text
index entry -> heap TID -> heap visible tuple -> optional index qual recheck -> scan qual/filter
```
Index-only scan 的对象是：
```text
index entry -> heap TID -> VM all-visible? -> optional heap visibility fetch -> index tuple data -> optional recheck
```
Bitmap scan 的对象是：
```text
index entries -> TIDBitmap -> heap page iterator -> visible tuple offsets -> optional bitmap recheck -> scan qual/filter
```
三条路径共享同一个系统规律：
PostgreSQL 不把“位置查找”误当成“SQL 语义已经成立”。
位置查找只减少搜索空间。
真正返回 tuple 前，必须把不确定性显式保存在状态里。
本节只讲这个 recheck boundary。
不完整讲 planner 如何选择 index scan 或 bitmap scan。
不完整讲每个 index AM 的内部搜索算法。
不完整讲 all-visible bit 的 VACUUM 设置细节。
不完整讲 predicate locking 的全部 SSI 语义。
这些内容会出现，但只服务一个问题：
候选结果如何被复验、跳过、降级和观测。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
index AM 产出候选 TID 或 TIDBitmap；table AM 证明 snapshot visibility；executor 在 recheck flag 为真时用原始 qual 复验候选；EXPLAIN 把被复验淘汰的 tuple 和 bitmap exact/lossy 形态暴露出来。
```
这个模型有四个边界。
第一个边界是 index AM。
它知道 index key、operator class、search strategy 和 AM-specific lossy semantics。
它可以说：
这个 TID 是候选。
这个候选需要重算 index qual。
这个 order by distance 可能只是 lower-bound。
它不能通用地判断 heap MVCC visibility。
它也不能通用地执行 executor 原始表达式。
第二个边界是 table AM。
`indexam.c` 调用 `table_index_fetch_begin()`、`table_index_fetch_tuple()` 和 `table_index_fetch_end()`。
对于 heap，这会落到 `heapam_indexscan.c` 的 HOT-aware fetch。
table AM 负责解释一个 index TID 在当前 snapshot 下是否对应可见 tuple。
对于 bitmap heap scan，heap table AM 负责按 page 读取、可见性判断、HOT chain 处理和 tuple offset 收集。
第三个边界是 visibility map。
VM all-visible bit 是 page-level shortcut。
它允许 index-only scan 在部分情况下跳过 heap fetch。
但它不是 tuple-level proof。
它也不证明 index qual 精确。
VM bit 只回答：
这个 heap page 上所有 tuple 是否对所有事务可见。
第四个边界是 executor。
executor 保留 `indexqualorig`、`recheckqual` 或 `bitmapqualorig`。
当 `xs_recheck` 或 bitmap result 的 `recheck` 为真时，executor 用当前 slot 重新执行这些表达式。
所以 recheck 是跨模块合同。
它不是单个函数内部的 if 分支。
它连接 index AM 的“不确定候选”、table AM 的“可见 tuple”、executor 的“原始语义表达式”。
## 3. 核心文件分工与源码阅读顺序
推荐按运行时状态推进阅读。
不要按文件名排序。
第一读 `src/include/nodes/execnodes.h`。
先看四个 executor state：
`IndexScanState`。
`IndexOnlyScanState`。
`BitmapIndexScanState`。
`BitmapHeapScanState`。
这里能看到 `iss_ScanDesc`、`ioss_VMBuffer`、`biss_result`、`tbm`、`recheck` 和 instrumentation 状态。
第二读 `src/include/access/relscan.h`。
重点是 `IndexScanDescData`。
这个结构是普通 index scan、index-only scan 和 bitmap index scan 的共同描述符。
关键字段是：
`xs_heaptid`。
`xs_heapfetch`。
`xs_heap_continue`。
`xs_recheck`。
`xs_itup` / `xs_hitup`。
`xs_orderbyvals`。
`xs_recheckorderby`。
第三读 `src/backend/access/index/indexam.c`。
这里是通用 index AM wrapper。
先读：
`index_beginscan()`。
`index_beginscan_bitmap()`。
`index_rescan()`。
`index_endscan()`。
`index_getnext_tid()`。
`index_fetch_heap()`。
`index_getnext_slot()`。
`index_getbitmap()`。
这一步回答：
谁创建 `IndexScanDesc`。
谁调用 AM callback。
谁调用 table AM 做 heap fetch。
谁要求调用者检查 `xs_recheck`。
第四读 `src/backend/executor/nodeIndexscan.c`。
主入口是：
`ExecInitIndexScan()`。
`ExecIndexScan()`。
`IndexNext()`。
`IndexNextWithReorder()`。
`ExecReScanIndexScan()`。
`ExecEndIndexScan()`。
这一步回答：
普通 index scan 如何把 `xs_recheck` 变成 `ExecQualAndReset(indexqualorig)`。
第五读 `src/backend/executor/nodeIndexonlyscan.c`。
主入口是：
`ExecInitIndexOnlyScan()`。
`ExecIndexOnlyScan()`。
`IndexOnlyNext()`。
`ExecReScanIndexOnlyScan()`。
`ExecEndIndexOnlyScan()`。
这一步回答：
index-only scan 如何通过 `VM_ALL_VISIBLE()` 决定是否访问 heap。
也回答 index tuple data 如何填入 virtual slot。
第六读 `src/backend/access/heap/visibilitymap.c`。
重点是：
`visibilitymap_get_status()`。
`visibilitymap_pin()`。
`visibilitymap_set()`。
`visibilitymap_clear()`。
这一步回答：
为什么 VM 读取可以不锁 VM buffer。
为什么设置 VM bit 需要 heap page lock、VM buffer lock 和 critical section。
第七读 `src/backend/executor/nodeBitmapIndexscan.c`。
主入口是：
`ExecInitBitmapIndexScan()`。
`MultiExecBitmapIndexScan()`。
`ExecReScanBitmapIndexScan()`。
`ExecEndBitmapIndexScan()`。
这一步回答：
bitmap index scan 为什么不返回 tuple slot。
它只构造或填充 `TIDBitmap`。
第八读 `src/include/nodes/tidbitmap.h` 和 `src/backend/nodes/tidbitmap.c`。
重点是：
`TIDBitmap`。
`TBMIterateResult`。
`tbm_add_tuples()`。
`tbm_add_page()`。
`tbm_lossify()`。
`tbm_private_iterate()`。
这一步回答：
exact page、lossy page 和 exact-but-recheck page 的区别。
第九读 `src/backend/executor/nodeBitmapHeapscan.c`。
主入口是：
`ExecInitBitmapHeapScan()`。
`BitmapTableScanSetup()`。
`BitmapHeapNext()`。
`ExecReScanBitmapHeapScan()`。
`ExecEndBitmapHeapScan()`。
并行路径读：
`BitmapShouldInitializeSharedState()`。
`BitmapDoneInitializingSharedState()`。
第十读 `src/backend/access/heap/heapam_handler.c`。
重点是：
`heapam_scan_bitmap_next_tuple()`。
`BitmapHeapScanNextBlock()`。
`heapam_methods.scan_bitmap_next_tuple`。
这一步回答：
bitmap heap scan 如何从 page-level bitmap 变成 visible tuple slot。
第十一读 `src/backend/commands/explain.c`。
重点是：
`Rows Removed by Index Recheck`。
`Heap Fetches`。
`Heap Blocks: exact/lossy`。
这一步回答：
运行时哪些 recheck 成本能观测，哪些只能推断。
## 4. 关键数据结构与状态
第一组状态是 executor node state。
`IndexScanState` 是普通 index scan 的 backend-local 状态。
它持有：
`indexqualorig`。
`iss_ScanKeys`。
`iss_OrderByKeys`。
`iss_RuntimeKeys`。
`iss_RelationDesc`。
`iss_ScanDesc`。
`iss_ReorderQueue`。
`iss_Instrument`。
`IndexScanState` 不持有 heap scan descriptor。
普通 index scan 的 heap fetch 状态藏在 `IndexScanDescData.xs_heapfetch`。
`IndexOnlyScanState` 也是 backend-local 状态。
它多了：
`recheckqual`。
`ioss_TableSlot`。
`ioss_VMBuffer`。
`ioss_NameCStringAttNums`。
`ioss_TableSlot` 用于必须访问 heap 时检查 visibility。
最终返回给上层的 scan slot 通常来自 index tuple data。
`ioss_VMBuffer` 是当前 visibility map page 的 buffer pin。
它可能跨多次 VM lookup 被复用。
`BitmapIndexScanState` 不产出 tuple slot。
它持有：
`biss_result`。
`biss_ScanKeys`。
`biss_RuntimeKeys`。
`biss_ArrayKeys`。
`biss_ScanDesc`。
`biss_result` 可以由父节点预先提供。
这样多个 bitmap index scan 可以把结果 OR 到同一个 bitmap 里，避免额外 UNION 节点。
`BitmapHeapScanState` 是真正返回 tuple 的 bitmap 节点状态。
它持有：
`bitmapqualorig`。
`tbm`。
`stats.exact_pages`。
`stats.lossy_pages`。
`initialized`。
`pstate`。
`recheck`。
`recheck` 是当前 page/tuple 是否需要用 `bitmapqualorig` 复验的 executor-local 状态。
第二组状态是 `IndexScanDescData`。
这是 index AM scan descriptor。
它不是 plan node。
它是 AM wrapper 与具体 index AM 之间的运行时合同。
关键字段组合才有语义。
`heapRelation + indexRelation + xs_snapshot` 定义一次 index scan 的对象和 visibility context。
`keyData + orderByData + numberOfKeys + numberOfOrderBys` 定义传给 AM 的 scankey。
`xs_heaptid` 是 AM 最近返回的候选 heap TID。
`xs_heapfetch` 是 table AM 的 index fetch state。
`xs_heap_continue` 表示同一个 index TID 后面是否还有可能通过 table AM 返回另一个可见 tuple。
对于 heap 的 HOT chain，非 MVCC-like snapshot 可能需要继续走。
普通 MVCC snapshot 下通常最多一个 visible version。
`xs_recheck` 是 AM 设置的 correctness signal。
它表示：
当前候选 tuple 必须用原始 index qual 复验。
它不表示 heap visibility 失败。
也不表示普通 filter 失败。
`xs_itup` / `xs_hitup` 只服务 index-only scan。
当 `xs_want_itup` 为真时，AM 要提供可返回的数据。
有些 AM 提供 `IndexTuple`。
有些 AM 可能提供 heap tuple 格式的数据。
`xs_recheckorderby` 和 `xs_orderbyvals` 服务 ordered index scan。
如果 index order-by distance 是近似值，executor 必须重算并可能在 `iss_ReorderQueue` 中重排。
第三组状态是 heap table AM scan/fetch state。
`IndexFetchHeapData` 在 heap index scan 路径中持有：
`xs_cbuf`。
`xs_blk`。
`xs_vmbuffer`。
如果同一 index scan 连续访问同一个 heap page，它可以复用 buffer pin。
`heapam_index_fetch_reset()` 故意不释放这些 pin。
这样紧密 nested loop rescan 可以少做重复 pin/unpin。
真正释放发生在 `heapam_index_fetch_end()`。
`HeapScanDescData` 在 bitmap heap scan 路径中持有：
`rs_cbuf`。
`rs_cblock`。
`rs_cindex`。
`rs_ntuples`。
`rs_vistuples[]`。
`rs_read_stream`。
`rs_vmbuffer`。
`rs_vistuples[]` 是当前 heap page 上已经通过 visibility 检查、准备逐个返回的 offset 列表。
这解释了 bitmap heap scan 的两阶段结构：
先按 page 生成 visible offset 列表。
再逐 tuple 填 slot 返回给 executor。
第四组状态是 `TIDBitmap`。
`TIDBitmap` 是 backend-local 或 DSA-backed per-query 状态。
它可以是：
空。
一个 exact page。
hash table。
其中 hash entry 可能表示 exact page，也可能表示 lossy chunk。
exact page 保存 tuple offset bit。
lossy chunk 只保存 page bit。
`TBMIterateResult` 对外暴露：
`blockno`。
`lossy`。
`recheck`。
`internal_page`。
`lossy=true` 意味着 heap scan 必须扫描该 page 上所有可见 tuple。
`recheck=true` 意味着最终 tuple 还必须重新执行 bitmap qual。
注意：
lossy page 总是 recheck。
exact page 也可能 recheck。
原因可能是 AM 插入 TID 时设置了 recheck。
也可能是 bitmap AND/OR 过程中信息不完整导致。
第五组状态是 visibility map buffer。
`visibilitymap_get_status()` 接受 `Buffer *vmbuf`。
调用者可以复用已经 pin 住的 VM page。
如果需要换 map page，函数会释放旧 pin、读入新 page。
读取 status 时不加 VM buffer lock。
这不是因为 VM bit 永远准确。
而是因为 index-only scan 的调用者用更高层的 ordering 和 visibility 规则吸收 stale read 风险。
第六组状态是 instrumentation。
`IndexScanInstrumentation.nsearches` 统计 index AM search 次数。
`BitmapHeapScanInstrumentation.exact_pages/lossy_pages` 统计 bitmap heap page 形态。
Plan instrumentation 的 `ntuples2` 在 index-only scan 中显示为 `Heap Fetches`。
Plan instrumentation 的 `nfiltered2` 显示为 `Rows Removed by Index Recheck`。
这些计数是观测入口。
它们不是完整因果图。
## 5. 主流程 walkthrough：普通 Index Scan
普通 index scan 从 `ExecInitIndexScan()` 创建 executor 状态。
它先 `makeNode(IndexScanState)`。
再 `ExecAssignExprContext()`。
再通过 `ExecOpenScanRelation()` 打开 heap relation。
它初始化 scan tuple slot，slot 类型来自 heap relation 的 table AM。
它初始化 executor qual：
`scan.plan.qual` 进入普通 `Filter`。
`indexqualorig` 保存原始 index qual，用于 recheck。
`indexorderbyorig` 保存原始 order-by expression，用于 order-by recheck。
如果只是 `EXPLAIN` 不执行，`EXEC_FLAG_EXPLAIN_ONLY` 会让初始化提前返回。
这允许某些 advisor 插件解释包含不存在 index 的 plan。
真正执行时，节点会打开 index relation。
`index_open(node->indexid, lockmode)` 返回 relation descriptor。
`ExecIndexBuildScanKeys()` 把 planner 的 index qual 转成 `ScanKey`。
常量值直接写入 scan key。
运行时表达式会生成 `IndexRuntimeKeyInfo`。
`ScalarArrayOpExpr` 如果 AM 不支持 `amsearcharray`，会生成 `IndexArrayKeyInfo`。
普通 index scan 不支持 array key fallback。
bitmap index scan 支持。
第一次调用 `ExecIndexScan()` 时，如果 runtime key 还没准备好，会先 `ExecReScan()`。
`ExecReScanIndexScan()` 重置 runtime expr context。
再执行 `ExecIndexEvalRuntimeKeys()`。
这一步把外层 tuple 相关的值写回 scan key。
然后 `ExecIndexScan()` 选择访问函数。
没有 order-by key 时使用 `IndexNext()`。
有 order-by key 时使用 `IndexNextWithReorder()`。
二者都通过 `ExecScan()` 接入通用 scan 框架。
`IndexNext()` 第一次运行时发现 `iss_ScanDesc == NULL`。
它调用：
`index_beginscan(heapRelation, indexRelation, snapshot, instrument, nkeys, norderbys, flags)`。
`index_beginscan()` 在 `indexam.c` 中创建 `IndexScanDesc`。
它会检查 historic snapshot 不能用于非 catalog table 的 logical decoding 场景。
它调用 `index_beginscan_internal()`。
通用内部函数增加 index relation relcache refcount。
再调用具体 AM 的 `ambeginscan`。
然后 `index_beginscan()` 把 heap relation、snapshot、instrument 写入 scan descriptor。
最后调用 `table_index_fetch_begin(heapRelation, flags)`。
对于 heap，`heapam_index_fetch_begin()` 分配 `IndexFetchHeapData`。
这一步建立了 index AM 与 table AM 的边界。
index AM scan descriptor 里有一个 table AM fetch descriptor。
但 index AM 本身不解释 heap visibility。
`IndexNext()` 随后调用 `index_rescan()` 把 scan keys 传给 index AM。
`index_rescan()` 会先重置 table AM fetch state。
对于 heap，这个 reset 是 no-op。
然后清掉 `kill_prior_tuple` 和 `xs_heap_continue`。
最后调用具体 AM 的 `amrescan`。
真正取 tuple 时，`IndexNext()` 进入：
`while (index_getnext_slot(scandesc, direction, slot))`。
`index_getnext_slot()` 是通用 wrapper。
如果 `xs_heap_continue` 为 false，它先调用 `index_getnext_tid()`。
`index_getnext_tid()` 调用具体 AM 的 `amgettuple`。
AM 把候选 TID 放到 `scan->xs_heaptid`。
AM 也可以设置 `scan->xs_recheck`。
如果没有候选，`index_getnext_tid()` 会 reset table AM fetch state 并返回 NULL。
如果有候选，`index_getnext_slot()` 调用 `index_fetch_heap()`。
`index_fetch_heap()` 调用 `table_index_fetch_tuple()`。
对于 heap，这进入 `heapam_index_fetch_tuple()`。
如果 TID 在新 heap block 上，它释放旧 `xs_cbuf` pin，读入新 block。
它会尝试 `heap_page_prune_opt()`。
然后获取 heap buffer share lock。
调用 `heap_hot_search_buffer()`。
这一步沿 HOT chain 找当前 snapshot 下可见的 tuple version。
找到后，把 tuple 存入 slot。
释放 buffer content lock，但保留 buffer pin。
这解释了一个重要边界：
tuple visibility 检查需要 buffer content lock。
返回给 executor 后，slot 与 scan descriptor 仍通过 buffer pin 保证 tuple bytes 不被回收或移动。
如果 heap HOT chain 全 dead，`table_index_fetch_tuple()` 会通过 `all_dead` 告诉上层。
`index_fetch_heap()` 在非 recovery 场景把 `kill_prior_tuple = all_dead`。
这个 hint 会在下一次 AM `amgettuple` 时被 index AM 用来标记 dead index tuple。
这是 index scan 与 index cleanup 的联系。
但它不影响本次 tuple correctness。
`index_getnext_slot()` 成功返回 slot 后，控制回到 `IndexNext()`。
这时 heap visibility 已经通过。
但 index qual 仍可能需要 recheck。
如果 `scandesc->xs_recheck` 为真：
executor 把 `slot` 放进 expr context。
调用 `ExecQualAndReset(node->indexqualorig, econtext)`。
失败则 `InstrCountFiltered2(node, 1)`。
然后继续取下一个 index candidate。
成功才返回 slot 给 `ExecScan()`。
然后 `ExecScan()` 还会执行普通 scan qual。
普通 scan qual 失败会计入 `Rows Removed by Filter`。
所以有两层过滤：
`Rows Removed by Index Recheck` 来自 index qual recheck。
`Rows Removed by Filter` 来自 plan filter。
它们不是同一件事。
### ORDER BY recheck path
`IndexNextWithReorder()` 是普通 index scan 的复杂旁路。
它服务支持 ordering operator 的 AM。
有些 AM 返回的 order-by distance 可能需要 recheck。
`xs_recheckorderby` 为真时，executor 用 `indexorderbyorig` 重算 distance。
如果重算值小于 index 返回值，说明 AM 返回顺序违反合同，报错：
`index returned tuples in wrong order`。
如果重算值大于 index 返回值，当前 tuple 可能要排到后面。
于是 executor 把 tuple copy 进 `iss_ReorderQueue`。
这个 queue 是 pairing heap。
它保存 heap tuple copy 和重算后的 order-by values。
只有当队首 tuple 不大于最近 index 返回的 lower-bound，或者 index 已到末尾，才能安全返回。
这条路径说明：
recheck 不只作用于 where qual。
它也可以作用于 order semantics。
不过 index-only scan 当前不支持 lossy distance recheck。
如果遇到 `xs_recheckorderby`，会报 feature not supported。
## 6. 主流程 walkthrough：Index-only Scan
Index-only scan 的目标不是“不访问 heap”。
准确说，它的目标是：
能从 index tuple 提供 targetlist 数据，并且在 VM 证明 heap page all-visible 时跳过 heap visibility fetch。
`ExecInitIndexOnlyScan()` 创建 `IndexOnlyScanState`。
它打开 heap relation。
但 scan tuple slot 不是 heap relation 的 physical tuple descriptor。
它使用 planner 生成的 `indextlist` 构造 tuple descriptor。
因为 index-only scan 返回的是 INDEX_VAR 语义的 tuple。
它还分配 `ioss_TableSlot`。
这个 slot 使用 table AM callback。
它只在必须访问 heap 验证 visibility 时使用。
执行入口 `ExecIndexOnlyScan()` 也走 `ExecScan()`。
访问函数是 `IndexOnlyNext()`。
第一次进入 `IndexOnlyNext()` 时创建 `IndexScanDesc`。
它调用 `index_beginscan()`，和普通 index scan 一样。
但随后设置：
`scandesc->xs_want_itup = true`。
这告诉 AM：
后续 `amgettuple` 不只要 TID，还要把 index tuple data 填到 `xs_itup` 或 `xs_hitup`。
`IndexOnlyNext()` 调用 `index_getnext_tid()`。
它不直接调用 `index_getnext_slot()`。
原因是 index-only scan 不想默认走 heap fetch。
每拿到一个 TID 后，它先检查 visibility map：
`VM_ALL_VISIBLE(heapRelation, blockno, &node->ioss_VMBuffer)`。
如果 VM bit 说明 page all-visible，就跳过 heap fetch。
如果 VM bit 不满足，就调用 `index_fetch_heap()`。
这一步只为 visibility。
成功后立即 `ExecClearTuple(node->ioss_TableSlot)`。
返回给上层的数据仍来自 index tuple，而不是 heap tuple。
如果 `index_fetch_heap()` 没找到 visible tuple，说明这个 index entry 对当前 snapshot 没有可见 heap tuple。
继续读下一个 index TID。
如果 `xs_heap_continue` 为真，index-only scan 报错。
源码明确说当前只支持 MVCC snapshot。
因为如果同一个 index entry 可返回多个 visible HOT chain member，index-only scan 需要额外状态避免下一轮误读下一个 TID。
这不是当前实现支持的语义。
填充返回 slot 时有两条路径。
如果 AM 提供 `xs_hitup`，用 `ExecForceStoreHeapTuple()`。
否则如果提供 `xs_itup`，调用 `StoreIndexTuple()`。
`StoreIndexTuple()` 用 AM 提供的 tuple descriptor 做 `index_deform_tuple()`。
然后把值放入 virtual slot。
对于 `name` 类型使用 `cstring` 存储的 opclass，还要复制成 `NAMEDATALEN` 长度的 `Name` datum。
这说明 index-only scan 的 tuple layout 也是 AM contract。
它不是简单把 index page 上 bytes 当 heap tuple 返回。
填完 slot 后，仍然检查 `scandesc->xs_recheck`。
如果为真，执行 `recheckqual`。
失败计入 `Rows Removed by Index Recheck`。
注意：
即使跳过 heap fetch，也不能跳过必要的 qual recheck。
VM 只证明 visibility。
不证明 lossy AM qual。
如果未访问 heap，`IndexOnlyNext()` 还会执行 `PredicateLockPage()`。
这是 SSI 边界。
普通 heap fetch 路径会对 tuple 做 predicate lock。
跳过 heap fetch 后，需要显式 page-level predicate lock 来保持 serializable conflict 检测语义。
### VM stale read 为什么可接受
`visibilitymap_get_status()` 通常不锁 VM buffer。
它读取一个 byte。
源码注释把 memory ordering 责任交给调用者。
`IndexOnlyNext()` 里的长注释解释了 index-only scan 为什么可以这样做。
对 insert：
插入新 tuple 会先清 VM bit，再把 TID 插入 index。
index-only scan 如果能在 index page 上看到新 TID，说明它与插入 index tuple 的 index page lock/unlock 已经发生序列化。
锁释放/获取形成足够的 memory ordering。
所以它应能看到 VM bit 被清掉。
于是不会错误地把新插入、尚不可见 tuple 当成 all-visible。
对 delete：
删除会清 VM bit，但不会同步更新 index page。
index-only scan 可能读到旧 VM bit。
这看起来危险。
但删除后 tuple 对某些 snapshot 仍可见，直到删除事务提交或当前语句边界过去。
如果当前 scan 的 snapshot 已经能看到删除结果，说明它获取 snapshot 的 ProcArray synchronization 发生在 VM bit 清除之后。
因此会看到清除后的状态。
这就是 VM 无锁读能正确的关键：
不是 VM bit 永远最新。
而是对错误方向的 stale read，有其他同步关系排除掉会破坏可见性的情况。
这也是为什么这段逻辑不能脱离 `IndexOnlyNext()` 的注释单独复用。
## 7. 主流程 walkthrough：Bitmap Index Scan
Bitmap index scan 的 executor convention 与普通 scan 不同。
`ExecBitmapIndexScan()` 是 stub。
如果有人按 `ExecProcNode` tuple-at-a-time 调它，直接报错：
`BitmapIndexScan node does not support ExecProcNode call convention`。
真正入口是 `MultiExecBitmapIndexScan()`。
它一次性产出一个 `TIDBitmap`。
初始化阶段 `ExecInitBitmapIndexScan()` 不打开 heap relation。
源码注释明确说：
ancestor `BitmapHeapScan` 节点负责持有 heap relation lock。
Bitmap index scan 只打开 index relation。
它构造 scan keys。
它支持 runtime keys。
也支持 array keys。
如果 AM 不支持 `amsearcharray`，`ScalarArrayOpExpr` 会由 executor 拆成多次 index scan。
`ExecInitBitmapIndexScan()` 创建 `IndexScanDesc` 的方式是：
`index_beginscan_bitmap(indexRelation, snapshot, instrument, nkeys)`。
注意这个 descriptor 没有 `heapRelation`。
因此也没有 `xs_heapfetch`。
它不会通过 table AM 做 heap visibility fetch。
这就是 bitmap index scan 与普通 index scan 的根本差异。
`MultiExecBitmapIndexScan()` 先确保 runtime keys ready。
如果 array key 为空，`ExecReScanBitmapIndexScan()` 会让 `biss_RuntimeKeysReady` 保持 false。
于是 `doscan=false`。
此时仍会返回一个空 bitmap，而不是访问 index。
然后它准备 result bitmap。
如果父节点已经把 `biss_result` 填好，就使用这个 bitmap，并把 `biss_result` 置 NULL。
否则创建新 `TIDBitmap`：
`tbm_create(work_mem * 1024, query_dsa_if_shared)`。
这一步把 `work_mem` 引入 runtime 形态。
work_mem 不是决定查询结果。
它决定 bitmap 是否能保持 exact。
扫描循环里，`index_getbitmap(scandesc, tbm)` 调具体 AM 的 `amgetbitmap`。
AM 把候选 TID 加入 `TIDBitmap`。
返回值是 matching tuple 数。
注释说这个值可能只是 approximate，只能用于统计。
如果有 array key，`ExecIndexAdvanceArrayKeys()` 推进到下一组元素。
然后 `index_rescan()` 用新 scan key 重启 index scan。
这意味着一个 bitmap index scan 节点可能对应多次 index AM search。
`Index Searches` 可以大于 1。
`MultiExecBitmapIndexScan()` 完成后返回 `Node *` 指向 `TIDBitmap`。
这个 bitmap 还没有做 heap visibility。
它只是 heap TID 候选集合。
## 8. 主流程 walkthrough：TIDBitmap exact、lossy 与 recheck
`TIDBitmap` 设计目标是：
能保存很大的 TID 集合，同时把内存限制在近似 `work_mem` 内。
它的基本单位是 heap block。
exact page entry 保存该 page 上哪些 tuple offset 命中。
lossy chunk entry 保存一组 page 中哪些 page 需要访问。
lossy entry 不保存 tuple offset。
`tbm_add_tuples(tbm, tids, ntids, recheck)` 添加 tuple-level TID。
如果目标 page 仍 exact，就设置对应 offset bit。
同时把 page entry 的 `recheck` 与传入 recheck 做 OR。
如果 bitmap 已经把该 page 标成 lossy，就不再保存 offset。
如果 hash entry 数超过 `maxentries`，调用 `tbm_lossify()`。
`tbm_lossify()` 会把部分 exact page 转成 lossy page。
当前实现基本按 hashtable iteration 顺序选择，注释也承认这是比较粗糙的策略。
它试图把 entry 数降到 `maxentries / 2`。
如果无法降到限制内，会提高 `maxentries`，承认无法严格 fit within work_mem。
这解释两个现象。
第一，lossy page 出现位置不是稳定的语义属性。
它可能受 work_mem、数据分布、hash iteration、bitmap combination 和版本实现影响。
第二，bitmap scan 的 correctness 不依赖 exact/lossy 选择。
lossy 只影响后续 heap page 上要多检查多少 tuple。
`tbm_add_page(tbm, pageno)` 直接把整个 page 标成 lossy。
它用于 AM 只能给 page-level candidate 的场景。
扫描 bitmap 时，`tbm_private_iterate()` 或 shared iterator 返回 `TBMIterateResult`。
如果返回 lossy page：
`lossy=true`。
`recheck=true`。
`internal_page=NULL`。
如果返回 exact page：
`lossy=false`。
`recheck=page->recheck`。
`internal_page` 指向 exact page entry。
`tbm_extract_page_tuple()` 只能用于 exact page。
它把 offset bit 解出到 offset array。
所以 exact 与 lossy 的区别不是“是否要访问 heap page”。
两者都要访问 heap page。
区别是：
exact page 只检查 bitmap 指定 offset 和必要 HOT chain。
lossy page 必须检查 page 上所有 normal line pointer 的可见 tuple。
recheck 的区别是：
`recheck=false` 时，heap visibility 通过后可以认为 index condition 已成立。
`recheck=true` 时，heap visibility 通过后还要执行 `bitmapqualorig`。
## 9. 主流程 walkthrough：Bitmap Heap Scan
`ExecInitBitmapHeapScan()` 创建 `BitmapHeapScanState`。
它断言不支持 backward scan 和 mark/restore。
它还断言 snapshot 是 MVCC snapshot。
文件开头解释了原因：
bitmap index scan 和 heap access 解耦。
当 heap scan 访问某个 TID 时，原来的 index tuple 可能已经不存在。
更糟的是 line pointer 可能被复用给新 tuple。
MVCC snapshot 能保证新 tuple 不会错误通过 time qual。
非 MVCC snapshot 无法保证这一点。
所以 bitmap heap scan 只适合 MVCC-compliant snapshot。
初始化时：
`tbm=NULL`。
`initialized=false`。
`recheck=true`。
`stats` 清零。
它打开 heap relation。
初始化 outer plan，也就是 bitmap index scan 或 bitmap AND/OR 子树。
它初始化普通 scan qual 和 `bitmapqualorig`。
执行入口 `ExecBitmapHeapScan()` 通过 `ExecScan()` 调 `BitmapHeapNext()`。
第一次进入 `BitmapHeapNext()` 时，`initialized=false`。
于是调用 `BitmapTableScanSetup()`。
非并行路径中，`BitmapTableScanSetup()` 调 `MultiExecProcNode(outerPlanState(node))`。
这会执行下面的 bitmap index scan 子树，返回 `TIDBitmap`。
如果返回对象不是 `TIDBitmap`，报错。
然后 `tbm_begin_iterate()` 创建 iterator。
如果还没有 table scan descriptor，调用：
`table_beginscan_bm(heapRelation, snapshot, 0, NULL, flags)`。
这创建一个 bitmap heap scan 专用的 `TableScanDesc`。
它带 `SO_TYPE_BITMAPSCAN` 和 `SO_ALLOW_PAGEMODE`。
随后把 `rs_tbmiterator` 存进 scan descriptor。
`initialized=true`。
`BitmapHeapNext()` 进入循环：
`table_scan_bitmap_next_tuple(scanDesc, slot, &node->recheck, &lossy_pages, &exact_pages)`。
这通过 table AM callback 进入 heap 的 `heapam_scan_bitmap_next_tuple()`。
如果当前 page 的 visible tuple offset 用完，它调用 `BitmapHeapScanNextBlock()`。
`BitmapHeapScanNextBlock()` 先释放前一个 `rs_cbuf` pin。
再通过 read stream 取下一个 bitmap 指定的 buffer。
它拿到 `TBMIterateResult`。
如果 exact page，先调用 `tbm_extract_page_tuple()` 解出 offset。
然后把 `*recheck = tbmres->recheck`。
接着执行 `heap_page_prune_opt()`。
然后获取 heap buffer share lock。
exact page 路径：
遍历 bitmap 指定的 offsets。
对每个 offset 调 `heap_hot_search_buffer()`。
如果当前 snapshot 下有 visible tuple，就把可见 offset 写入 `rs_vistuples[]`。
lossy page 路径：
遍历 page 上所有 line pointer。
跳过非 normal item。
对每个 tuple 调 `HeapTupleSatisfiesVisibility()`。
visible 就把 offset 写入 `rs_vistuples[]`，并做 predicate lock。
两条路径都调用 `HeapCheckForSerializableConflictOut()`。
然后释放 heap buffer content lock。
把 `rs_ntuples` 设成找到的 visible tuple 数。
根据 `tbmres->lossy` 增加 `lossy_pages` 或 `exact_pages`。
返回 true。
`heapam_scan_bitmap_next_tuple()` 再从 `rs_vistuples[rs_cindex]` 取一个 offset。
它从 heap page 上取 item。
填充 `rs_ctup`。
调用 `ExecStoreBufferHeapTuple()` 把 tuple 放进 slot。
slot 获取 buffer pin。
`rs_cindex++`。
返回 true 给 `BitmapHeapNext()`。
这时 heap visibility 已经通过。
如果 `node->recheck` 为真，`BitmapHeapNext()` 执行 `bitmapqualorig`。
失败则计入 `Rows Removed by Index Recheck`。
然后 `ExecClearTuple(slot)`，继续循环。
成功才返回 slot。
再由 `ExecScan()` 执行普通 `Filter`。
所以 bitmap heap scan 有三层判断：
heap visibility。
bitmap recheck condition。
ordinary filter。
`EXPLAIN` 中的 `Recheck Cond` 表示用于 recheck 的表达式。
它不表示所有 tuple 都实际被 recheck。
是否实际 recheck 取决于 `TBMIterateResult.recheck`。
## 10. 并行 Bitmap Heap Scan 状态推进
Bitmap heap scan 有并行路径。
状态结构是 `ParallelBitmapHeapState`。
它在 DSM 中，包含：
`tbmiterator`。
`mutex`。
`state`。
`cv`。
`state` 有三种：
`BM_INITIAL`。
`BM_INPROGRESS`。
`BM_FINISHED`。
第一个看到 `BM_INITIAL` 的 worker 会把 state 改成 `BM_INPROGRESS`。
它负责执行 outer bitmap index scan，创建 `TIDBitmap`。
然后调用 `tbm_prepare_shared_iterate()` 创建 shared iterator。
最后把 state 改成 `BM_FINISHED` 并 `ConditionVariableBroadcast()`。
其他 worker 如果看到 `BM_INPROGRESS`，就在 `WAIT_EVENT_PARALLEL_BITMAP_SCAN` 上等。
等 leader 完成后，它们共同 attach 到 shared iterator。
这个设计有一个很直接的成本边界：
parallel bitmap heap scan 并不是多个 worker 同时构造同一个 bitmap。
bitmap 构造由一个 leader-like worker 完成。
并行收益主要来自后续共同访问 heap pages。
并行 instrumentation 会把每个 worker 的 `exact_pages/lossy_pages` 汇总到 shared instrumentation。
worker 结束时不能简单 memcpy。
源码用累加方式处理。
原因是 Gather/GatherMerge rescan 可能关闭旧 worker、启动新 worker。
## 11. 生命周期 / ownership / cleanup
普通 index scan 的生命周期：
`ExecInitIndexScan()` 创建 `IndexScanState`。
`index_open()` 打开 index relation。
第一次 `IndexNext()` 或 DSM 初始化时创建 `IndexScanDesc`。
`index_beginscan_internal()` 增加 index relation refcount。
`index_beginscan()` 创建 table AM index fetch state。
执行期间 `IndexScanDesc` 持有具体 AM scan state 和 table fetch state。
`ExecReScanIndexScan()` 重算 runtime keys，清空 reorder queue，调用 `index_rescan()`。
`ExecEndIndexScan()` 调 `index_endscan()`。
`index_endscan()` 先 `table_index_fetch_end()` 释放 table fetch 资源。
然后调用 AM `amendscan`。
再减少 index relation refcount。
如果是 temp snapshot，注销 snapshot。
最后 `IndexScanEnd()` 释放 scan descriptor。
`ExecEndIndexScan()` 再 `index_close(indexRelationDesc, NoLock)`。
heap relation 的关闭由 executor scan relation lifecycle 负责，不在本函数里直接关闭。
Index-only scan 的生命周期类似。
差异是：
它持有 `ioss_VMBuffer`。
`ExecEndIndexOnlyScan()` 首先释放这个 VM buffer pin。
然后才结束 index scan。
如果忘记释放 VM buffer pin，后续 VM page truncation 或 buffer eviction 会被 pin 干扰。
Bitmap index scan 的生命周期：
`ExecInitBitmapIndexScan()` 创建状态并打开 index relation。
它立即创建 `biss_ScanDesc`。
因为 bitmap index scan 的 `MultiExec` 需要直接 `index_getbitmap()`。
`ExecReScanBitmapIndexScan()` 重算 runtime/array keys。
如果 array key 为空，不启动 index scan。
`ExecEndBitmapIndexScan()` 调 `index_endscan()` 和 `index_close()`。
Bitmap heap scan 的生命周期：
`ExecInitBitmapHeapScan()` 创建状态、打开 heap relation、初始化 outer bitmap plan。
第一次 `BitmapHeapNext()` 创建 `TIDBitmap` 和 table scan descriptor。
`ExecReScanBitmapHeapScan()` 如果已有 scan descriptor：
先结束未耗尽的 bitmap iterator。
再 `table_rescan()` 释放任何 page pin。
如果有 `tbm`，调用 `tbm_free()`。
然后 `initialized=false`，`recheck=true`。
最后如果 outer plan 没有 `chgParam`，显式 `ExecReScan(outerPlan)`。
`ExecEndBitmapHeapScan()` 先把 parallel worker stats 累加到 shared instrumentation。
再 `ExecEndNode(outerPlanState(node))`。
如果 table scan descriptor 存在：
结束未耗尽 iterator。
`table_endscan(scanDesc)`。
如果 `tbm` 存在：
`tbm_free(node->tbm)`。
错误路径上的内存释放主要依靠 executor memory context 和 resource owner。
但 buffer pin、relation refcount、AM scan state 仍必须尽量走对应 end/reset 路径。
PostgreSQL 的 ERROR unwinding 会释放 ResourceOwner 管理的 pin/lock。
MemoryContext reset 会释放 backend-local palloc 内存。
但这不是让代码可以随意跳过 `index_endscan()` 的理由。
正常结束路径仍要把 AM 私有状态、instrumentation 聚合、iterator shared area 和 relation refcount 按合同清掉。
## 12. 正确性机制层次
本节正确性不是一个机制保证的。
它由多层机制叠加。
第一层是 MVCC snapshot。
普通 index scan 和 bitmap heap scan 都必须通过 heap visibility。
普通 index scan 通过 `table_index_fetch_tuple()`。
bitmap heap scan 通过 `HeapTupleSatisfiesVisibility()` 或 `heap_hot_search_buffer()`。
MVCC 只能回答 tuple version 是否对 snapshot 可见。
它不回答 index qual 是否精确。
第二层是 HOT chain 解释。
普通 index scan 的 `heapam_index_fetch_tuple()` 对一个 index TID 调 `heap_hot_search_buffer()`。
exact bitmap page 路径也对指定 offset 调 `heap_hot_search_buffer()`。
这允许 index entry 指向 HOT chain root，最终返回当前 snapshot 可见成员。
lossy bitmap page 路径扫描整页所有 line pointer。
它不需要从每个 root 追 HOT chain，因为它会逐个 tuple 检查。
第三层是 buffer content lock 与 pin。
heap visibility 检查时需要 buffer share lock。
返回 tuple 后释放 content lock，但保留 pin。
pin 保证 page 不会被回收或从 buffer 中消失。
pin 不保证 tuple 对 SQL 语义仍然可见。
语义已经由 snapshot 检查确定。
第四层是 index AM recheck flag。
`xs_recheck` 和 bitmap `recheck` 把 AM 的近似或不完整信息传给 executor。
executor 用原始 qual 复验。
这层保证 lossy index AM 或 lossy bitmap 不会产生 false positive。
注意它不能修复 false negative。
如果 AM 漏掉应该返回的 TID，executor 无法补回来。
所以 AM contract 仍必须保证候选集合是 superset。
第五层是 visibility map ordering。
VM bit 可以让 index-only scan 跳过 heap fetch。
但 VM bit 的 correctness 依赖：
修改 heap page 时清 bit。
设置 bit 时持有 heap page lock 和 VM buffer lock。
设置 all-visible 相关变更在 critical section 中与 WAL/heap page 修改保持一致。
index-only scan 侧依赖 index page lock ordering 和 snapshot acquisition ordering。
第六层是 predicate locking。
heap fetch 路径会对 visible tuple 做 predicate lock。
index-only scan 如果跳过 heap fetch，要显式 page-level predicate lock。
bitmap heap scan 在 heap page visibility 检查中也调用 predicate lock/conflict 检查。
这层服务 serializable isolation。
它不是 MVCC visibility。
第七层是 relation lock 和 relcache refcount。
executor 打开 scan relation 持有 relation lock。
`index_beginscan_internal()` 对 index relation relcache entry 增加 refcount。
这保证 scan 期间 relation descriptor 不会在本 backend 中被释放。
它不阻止其他事务修改 heap tuple。
MVCC 才负责解释并发修改。
第八层是 parallel synchronization。
parallel bitmap heap scan 用 spinlock 和 condition variable 协调 bitmap 初始化。
这只保证 workers 不会在 bitmap 未完成时开始 shared iteration。
它不改变 heap tuple visibility 语义。
## 13. 错误路径 / 异常路径 / fallback
第一个异常路径是 bitmap heap scan 拒绝非 MVCC snapshot。
源码在文件头注释说明原因。
index scan 与 heap access 被解耦后，TID 对应的 line pointer 可能被复用。
只有 MVCC snapshot 能确保复用后的新 tuple 不会错误通过。
所以 `ExecInitBitmapHeapScan()` 断言 `IsMVCCSnapshot(estate->es_snapshot)`。
这是 correctness guard，不是性能选择。
第二个异常路径是 bitmap index scan 被错误按 tuple-at-a-time 调用。
`ExecBitmapIndexScan()` 直接 `elog(ERROR)`。
因为 bitmap index scan 的结果不是 slot。
它只在 `MultiExec` convention 下返回 `TIDBitmap`。
第三个异常路径是 runtime array key 为空。
`ExecIndexEvalArrayKeys()` 遇到 NULL 或 empty array 返回 false。
bitmap index scan 会把 `biss_RuntimeKeysReady` 设为 false。
`MultiExecBitmapIndexScan()` 仍创建或使用 bitmap，但不执行 index scan。
结果为空。
这是一个正常 fallback：
空 array 不需要报错，也不需要访问 index。
第四个异常路径是 index-only scan 遇到需要继续 HOT chain 的非 MVCC snapshot 行为。
`IndexOnlyNext()` 在 `xs_heap_continue` 为真时报错：
non-MVCC snapshots are not supported in index-only scans。
原因是当前实现没有状态在下一轮继续同一 TID 的 HOT chain。
第五个异常路径是 index-only scan 遇到 lossy order-by distance。
普通 index scan 可以用 reorder queue 处理 `xs_recheckorderby`。
index-only scan 当前不支持。
如果 `numberOfOrderBys > 0 && xs_recheckorderby`，报 feature not supported。
第六个 fallback 是 VM bit 不满足。
index-only scan 不会猜。
它访问 heap，调用 `index_fetch_heap()` 验证 visibility。
这就是 `Heap Fetches`。
因此 index-only plan 不等于零 heap IO。
第七个 fallback 是 TIDBitmap 超过内存目标。
`tbm_lossify()` 把 exact page 转为 lossy page。
这不会改变结果。
它把内存压力转移成 heap page 扫描和 qual recheck 成本。
第八个 fallback 是 bitmap heap scan rescan。
如果 scan descriptor 中 iterator 尚未耗尽，必须 `tbm_end_iterate()`。
然后 `table_rescan()` 释放 page pin。
再 `tbm_free()`。
否则旧 iterator、旧 pin 或旧 bitmap 会污染下一轮 scan。
第九个异常路径是 parallel bitmap scan 中 worker 等待 bitmap 初始化。
非 leader worker 在 condition variable 上等待。
wait event 是 `WAIT_EVENT_PARALLEL_BITMAP_SCAN`。
如果 leader ERROR，executor/parallel query cleanup 会让 workers 退出，而不是让它们继续读半初始化 bitmap。
第十个错误边界是 `visibilitymap_clear()` 和 `visibilitymap_set()` 的 buffer 参数。
它们要求传入覆盖目标 heap block 的 VM buffer。
如果 buffer 不匹配，直接 `elog(ERROR)`。
这是为了防止 caller 把 VM bit 写到错误 map page。
## 14. 成本、资源与跨模块传播
普通 index scan 的主要成本随 returned candidates 增长。
每个 candidate 可能触发：
index AM search/advance。
heap buffer read/pin。
heap buffer share lock。
HOT chain traversal。
visibility check。
optional index qual recheck。
ordinary filter。
如果 index selectivity 很差，普通 index scan 会产生大量随机 heap access。
这时 bitmap heap scan 可能更合适，因为它把 TID 按 block 排序并批量访问 heap page。
但 bitmap heap scan 先构造整个 bitmap。
它有 startup cost。
对于非常小的 result set，普通 index scan 的 tuple-at-a-time latency 可能更低。
Index-only scan 的成本取决于 all-visible fraction。
VM 命中时跳过 heap fetch。
VM miss 时仍要读 heap page 验证 visibility。
频繁 update/delete 的表会持续清 VM bit。
autovacuum 未及时推进 all-visible，也会增加 heap fetch。
所以 index-only scan 的性能不是“有 covering index 就稳定快”。
它依赖 workload、VACUUM、xid horizon 和 page churn。
Bitmap scan 的成本由四个变量控制。
第一是候选 TID 数。
候选越多，bitmap 构造越重。
第二是 distinct heap blocks 数。
heap block 越多，heap read 越多。
第三是 bitmap exact/lossy 形态。
lossy page 越多，heap page 内需要检查的 tuple 越多。
第四是 recheck selectivity。
recheck false positive 越多，CPU 花在 executor qual 上越多。
`work_mem` 是重要传播点。
较小 `work_mem` 让 `TIDBitmap.maxentries` 变小。
bitmap 更早 lossify。
lossy pages 增加。
heap scan 从检查指定 offsets 退化到扫描整页 visible tuples。
这会增加 CPU、buffer touch 和 `Rows Removed by Index Recheck`。
但如果表很大、page 上 tuple 很密，lossy 退化可能非常明显。
并行 bitmap scan 的资源传播也有边界。
bitmap 构造阶段可能成为串行瓶颈。
shared iterator 和 DSA 引入 per-query shared memory 管理成本。
heap page 访问可以并行化。
但如果 IO 队列饱和或 lossy recheck CPU 成为瓶颈，更多 worker 未必线性提升。
index-only scan 的 VM buffer pin 是资源。
`ioss_VMBuffer` 复用能减少 pin/unpin。
但它也必须在 end 时释放。
heap index fetch 的 `xs_cbuf` 和 `xs_vmbuffer` 也会跨 fetch 复用。
这种局部性优化减少 CPU 和 buffer manager 压力。
但它要求 cleanup 路径可靠。
跨模块连接至少有这些：
planner 决定 scan type 和 index qual/recheck qual 形态。
executor 持有 plan state、expr context、slot 和 instrumentation。
index AM 提供 TID、index tuple、AM-specific recheck flag。
table AM 解释 TID 到 visible tuple。
heap AM 处理 HOT chain、visibility 和 page pruning。
visibility map 提供 index-only shortcut。
TIDBitmap 模块处理 exact/lossy/recheck 状态。
EXPLAIN 把部分 runtime 状态暴露给用户。
没有后台进程直接推进本节 scan state。
但 autovacuum/VACUUM 会推进 visibility map all-visible 状态，间接改变 index-only scan 的 heap fetch 数。
checkpoint/bgwriter 只影响 IO 背景压力，不改变 recheck 语义。
## 15. 观测与诊断入口
最直接入口是：
`EXPLAIN (ANALYZE, BUFFERS)`。
普通 index scan 关注：
`Index Cond`。
`Rows Removed by Index Recheck`。
`Rows Removed by Filter`。
`Buffers`。
`Index Searches`。
Index-only scan 额外关注：
`Heap Fetches`。
Bitmap heap scan 关注：
`Recheck Cond`。
`Rows Removed by Index Recheck`。
`Heap Blocks: exact=... lossy=...`。
`Buffers`。
Bitmap index scan 关注：
子节点的 `Index Cond` 和 `Index Searches`。
这些指标粒度不同。
`Rows Removed by Index Recheck` 是单 plan node 的执行期计数。
`Heap Fetches` 是单 IndexOnlyScan node 的执行期计数。
`Heap Blocks exact/lossy` 是 BitmapHeapScan node 的执行期 page 计数。
`Buffers` 是 buffer hit/read/dirtied/written 的执行期累计。
`pg_stat_*` 是 relation 或 database 级累计，不适合解释单次 recheck 因果。
第二入口是 `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)`。
当怀疑 `work_mem` 改变 bitmap lossy 形态时，记录 setting 很重要。
同一 SQL 在不同 `work_mem` 下可能从 exact pages 变为 lossy pages。
结果不变。
CPU 和 heap page recheck 成本变。
第三入口是 `pg_visibility` 扩展。
它可以观察 relation page 的 all-visible/all-frozen 状态。
它不能告诉你某次 index-only scan 为什么刚好 heap fetch。
但它能解释为什么 covering index 仍大量访问 heap。
如果大量页面不是 all-visible，index-only scan 就会退回 heap visibility fetch。
第四入口是 `pg_stat_user_tables` 和 VACUUM 相关信息。
`n_tup_upd`、`n_tup_del`、`n_dead_tup`、`last_vacuum`、`last_autovacuum` 可以作为 index-only heap fetch 增多的背景证据。
它们不是单次 query 的证明。
第五入口是 `pg_stat_io` 和 buffer/io wait。
Bitmap heap scan lossy 退化可能增加 heap page 访问。
但如果页面都在 shared buffers，主要表现为 CPU/recheck。
如果页面不在内存，表现为 read IO。
这需要结合 `BUFFERS` 和系统 profiling。
第六入口是 `perf` 或 flamegraph。
如果 `Rows Removed by Index Recheck` 很高，但 buffers 不高，瓶颈可能在 expression evaluation、tuple deforming、visibility checks 或 HOT chain traversal。
这时 `pg_stat_*` 不足以归因。
第七入口是 gdb 或临时日志。
可以在这些位置断点：
`IndexNext()` 中 `scandesc->xs_recheck` 分支。
`IndexOnlyNext()` 中 `VM_ALL_VISIBLE()` 分支。
`MultiExecBitmapIndexScan()` 中 `tbm_create()`。
`BitmapHeapScanNextBlock()` 中 `tbmres->lossy` 和 `tbmres->recheck`。
`heapam_scan_bitmap_next_tuple()` 中 `rs_vistuples` 消费。
这些断点能直接看到状态从 AM 传到 executor。
第八入口是 wait event。
parallel bitmap heap scan 中，如果 worker 等在 bitmap 初始化上，可看到 `Parallel Bitmap Scan` 对应 wait event。
这说明并行 worker 在等 shared bitmap 构造完成。
它不说明 heap page 访问慢。
诊断时要分三类状态。
能直接观测：
`Heap Fetches`。
`Rows Removed by Index Recheck`。
`Heap Blocks exact/lossy`。
`Buffers`。
只能推断：
某个 VM bit 的瞬时 stale read。
某次 `xs_recheck` 是哪个 index AM 内部原因导致。
某个 bitmap page 为什么被 lossify。
几乎不可见：
具体 `TIDBitmap` hash entry 选择哪个 page lossify。
普通 SQL 层无法直接看到 `xs_heap_continue`。
executor 内部 slot 与 buffer pin 的精确生命周期。
## 16. 常见误区
误区一：
看到 `Index Cond` 就以为返回 tuple 一定满足它。
正确理解：
`Index Cond` 是传给 index AM 的条件。
如果 AM 或 bitmap 表示是 lossy，executor 还要用原始 qual recheck。
误区二：
看到 Bitmap Heap Scan 的 `Recheck Cond` 就以为每行都 recheck。
正确理解：
`Recheck Cond` 是可能用于 recheck 的表达式。
是否每个 tuple 都执行，取决于 bitmap page/tuple 的 `recheck` 状态。
exact 且 `recheck=false` 的 tuple 不需要执行。
误区三：
以为 lossy bitmap 表示查询结果近似。
正确理解：
lossy bitmap 只表示中间候选集合近似。
最终结果仍必须通过 heap visibility 和 qual recheck。
lossy 影响成本，不改变 SQL 语义。
误区四：
以为 index-only scan 永远不访问 heap。
正确理解：
index-only scan 只有在 VM all-visible 证明 heap page 对所有事务可见时才能跳过 heap visibility fetch。
VM miss、recent update/delete、VACUUM 滞后都会带来 heap fetch。
误区五：
以为 VM bit 是 tuple-level visibility。
正确理解：
VM bit 是 page-level。
all-visible 表示该 page 上所有 tuple 对所有事务可见。
它不能说明某个 index qual 是否精确。
误区六：
以为 `Rows Removed by Index Recheck` 和 `Rows Removed by Filter` 可以合并看。
正确理解：
前者是 index candidate false positive 被原始 index/bitmap qual 淘汰。
后者是已通过 access method 条件后的普通 plan qual 淘汰。
它们对应不同优化方向。
误区七：
以为提高 `work_mem` 一定让 bitmap scan 更快。
正确理解：
提高 `work_mem` 可能减少 lossy pages。
但如果 bottleneck 是 index AM 构造、heap IO、普通 filter 或 CPU cache miss，收益可能有限。
误区八：
以为 `xs_recheck` 是 executor 可以忽略的 hint。
正确理解：
它是 correctness signal。
忽略会返回 false positive。
误区九：
以为 table AM reset 一定释放所有 buffer pin。
正确理解：
heap 的 `heapam_index_fetch_reset()` 故意不释放 `xs_cbuf` 和 `xs_vmbuffer`。
释放发生在 `heapam_index_fetch_end()`。
误区十：
以为 bitmap index scan 可以单独执行并返回 rows。
正确理解：
它只返回 `TIDBitmap`，必须由 BitmapHeapScan 访问 heap 并返回 tuple。
## 17. 课堂实验
### 实验一：观察 bitmap exact/lossy 与 recheck
目标：
用 `work_mem` 改变 bitmap 形态，观察 `Heap Blocks exact/lossy` 和 `Rows Removed by Index Recheck`。
准备表：
```sql
CREATE TABLE t_bm (id int, grp int, pad text);
INSERT INTO t_bm
SELECT g, g % 100, repeat('x', 100)
FROM generate_series(1, 1000000) AS g;
CREATE INDEX t_bm_grp_idx ON t_bm (grp);
ANALYZE t_bm;
```
执行：
```sql
SET enable_seqscan = off;
SET work_mem = '64kB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM t_bm WHERE grp BETWEEN 1 AND 80 AND pad LIKE 'x%';
```
再执行：
```sql
SET work_mem = '64MB';
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM t_bm WHERE grp BETWEEN 1 AND 80 AND pad LIKE 'x%';
```
观察点：
低 `work_mem` 下更容易出现 `Heap Blocks: lossy=...`。
高 `work_mem` 下 exact pages 通常增加。
结果行数应该不因 exact/lossy 变化而改变。
回到源码：
`MultiExecBitmapIndexScan()` 用 `work_mem` 创建 `TIDBitmap`。
`tbm_lossify()` 在 entry 数超过限制时把 exact page 转 lossy。
`BitmapHeapScanNextBlock()` 对 lossy page 扫整页并设置 recheck。
### 实验二：观察 index-only scan 的 heap fetch
目标：
证明 index-only scan 是否访问 heap 取决于 VM all-visible 状态。
准备表：
```sql
CREATE TABLE t_ios (id int PRIMARY KEY, v int);
INSERT INTO t_ios
SELECT g, g
FROM generate_series(1, 200000) AS g;
VACUUM (ANALYZE) t_ios;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM t_ios WHERE id BETWEEN 1000 AND 2000;
```
修改一部分页面：
```sql
UPDATE t_ios SET v = v + 1 WHERE id BETWEEN 1000 AND 2000;
ANALYZE t_ios;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM t_ios WHERE id BETWEEN 1000 AND 2000;
```
再 vacuum：
```sql
VACUUM (ANALYZE) t_ios;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM t_ios WHERE id BETWEEN 1000 AND 2000;
```
观察点：
第一次 vacuum 后，index-only scan 的 `Heap Fetches` 通常很低。
UPDATE 后相关 heap pages 的 VM all-visible bit 被清掉，`Heap Fetches` 可能增加。
再次 VACUUM 后，all-visible 可能恢复。
回到源码：
`IndexOnlyNext()` 先 `VM_ALL_VISIBLE()`。
VM miss 时调用 `index_fetch_heap()`。
`visibilitymap_clear()` 和 `visibilitymap_set()` 维护 VM bit。
### 实验三：源码断点跟踪 recheck 传播
目标：
直接看到 `xs_recheck` 和 bitmap `recheck` 如何进入 executor。
断点建议：
```text
src/backend/executor/nodeIndexscan.c:IndexNext
src/backend/executor/nodeIndexonlyscan.c:IndexOnlyNext
src/backend/executor/nodeBitmapHeapscan.c:BitmapHeapNext
src/backend/access/heap/heapam_handler.c:BitmapHeapScanNextBlock
src/backend/access/index/indexam.c:index_getnext_tid
src/backend/access/index/indexam.c:index_getbitmap
```
调试步骤：
先跑一个普通 B-tree equality index scan，观察 `xs_recheck` 通常为 false。
再跑一个 GiST、GIN 或需要 lossy recheck 的索引场景，观察 `xs_recheck` 或 bitmap `recheck`。
如果不方便构造 AM-specific recheck，可以在 `BitmapHeapScanNextBlock()` 观察 low `work_mem` 下 `tbmres->lossy` 变 true。
需要画出的状态变化：
```text
AM candidate
  -> IndexScanDesc.xs_heaptid / TIDBitmap
  -> table AM visible tuple
  -> xs_recheck 或 TBMIterateResult.recheck
  -> ExecQualAndReset(original qual)
  -> instrumentation nfiltered2
```
## 18. 讨论题
1. 为什么 `index_getnext_slot()` 不自己执行 `indexqualorig` recheck，而是要求 executor 调用者检查 `xs_recheck`？
2. Index-only scan 为什么需要 `ioss_TableSlot`，既然最终返回的数据来自 index tuple？
3. 为什么 VM bit 可以无锁读取，但 `visibilitymap_set()` 需要 VM buffer exclusive lock、heap page lock 和 critical section？
4. Bitmap heap scan 为什么只允许 MVCC-compliant snapshot？如果用 `SnapshotAny`，line pointer reuse 会带来什么问题？
5. exact bitmap page 在什么情况下仍然需要 recheck？这和 lossy page 的 recheck 有什么区别？
6. 如果 `Heap Blocks: lossy` 很高，但 `Rows Removed by Index Recheck` 很低，可能说明什么？如果二者都高，又说明什么？
7. parallel bitmap heap scan 为什么让一个 worker 先构造 shared bitmap，而不是每个 worker 各自扫描 index？
8. `Rows Removed by Filter` 很高和 `Rows Removed by Index Recheck` 很高分别应该优先回到哪些源码路径和 schema/workload 假设检查？
## 19. 本节小结
本节唯一主问题是：
索引访问路径如何在候选位置、heap visibility、近似索引和 lossy bitmap 之间保持正确性。
核心链路是：
普通 index scan 通过 `index_getnext_slot()` 拿 visible heap tuple，再按 `xs_recheck` 执行 `indexqualorig`。
Index-only scan 通过 `index_getnext_tid()` 拿 TID，再用 VM all-visible 决定是否访问 heap，最后仍按 `xs_recheck` 执行 `recheckqual`。
Bitmap index scan 通过 `MultiExecBitmapIndexScan()` 构造 `TIDBitmap`。
Bitmap heap scan 再按 page 访问 heap，把 exact/lossy/recheck 转换成 visible tuple 和 `bitmapqualorig` recheck。
核心状态是：
`IndexScanDescData` 保存 AM 与 executor/table AM 的候选和 recheck 合同。
`IndexFetchHeapData` 和 `HeapScanDescData` 保存 heap buffer pin、HOT/visibility 扫描状态。
`TIDBitmap` 保存 exact page、lossy chunk 和 recheck bit。
`VisibilityMap` 保存 page-level all-visible/all-frozen hint。
ownership 边界是：
executor node state 由 plan lifecycle 管。
`IndexScanDesc` 由 `index_beginscan`/`index_endscan` 管。
heap table fetch state 由 table AM `index_fetch_begin/end` 管。
bitmap iterator 和 `TIDBitmap` 由 bitmap heap scan setup/rescan/end 管。
VM buffer pin 由 index-only scan state 显式释放。
正确性不是“相信索引”。
正确性来自：
index AM 只产出候选 superset。
table AM 证明 snapshot visibility。
executor 按 recheck flag 复验原始 qual。
visibility map 只优化 heap visibility fetch。
bitmap lossy 只改变候选粒度。
错误和 fallback 也服务同一条链路：
VM miss 退回 heap fetch。
bitmap 超内存退回 lossy page。
array key 为空退回空 bitmap。
非 MVCC bitmap heap scan 被拒绝。
index-only 不支持非 MVCC HOT continuation 和 lossy order-by distance。
观测上：
`Heap Fetches` 说明 index-only scan 退回 heap visibility 的次数。
`Heap Blocks exact/lossy` 说明 bitmap 中间状态的 page 粒度。
`Rows Removed by Index Recheck` 说明候选 false positive 被原始 index/bitmap qual 淘汰。
这些指标能定位方向。
它们不能完整证明 AM 内部原因、VM stale read 时序或 lossify 选择。
可迁移的系统规律是：
高性能存储系统经常把“昂贵的精确证明”拆成两步。
第一步快速生成候选集合。
第二步在边界清晰的位置做必要复验。
只要候选集合是 superset，recheck 能恢复 correctness。
一旦候选生成漏掉 true positive，后面的 recheck 无法补救。
所以读源码时要一直区分：
candidate-producing state。
visibility-proving state。
semantic-rechecking state。
resource-cleanup state。
这套区分会在后续理解 index AM、heap pruning、VACUUM 和 planner cost model 时反复出现。
