# PostgreSQL TableAM 与 IndexAM 的持久化契约

## 课程定位

上一组课程已经讲过 heap tuple version、HOT、B-tree split、dedup 和 index cleanup。
现在把视角从某一个 AM 的页格式，移动到 TableAM 和 IndexAM 之间的接口边界。
前置知识：
- 已理解 heap tuple 的物理地址是 `ItemPointer`。
- 已理解 index tuple 通常保存 heap TID，而不是保存完整 row。
- 已理解 MVCC visibility 由 tuple header、snapshot 和事务状态共同决定。
- 已理解 HOT chain 允许一个 index entry 间接到达多个 row version。
- 已理解 bitmap heap scan 先收集 TID，再按 heap block 访问表页。
- 已理解 index cleanup 不能只看 index page 自己的状态。
本节唯一主问题：
IndexAM 为什么只能持久化和返回“定位候选”，而不能独立决定一个 tuple 对当前查询可见、对所有事务可删，或在 HOT/bitmap/recovery 场景下可以被安全跳过？
本节围绕的核心矛盾：
索引必须足够通用。
B-tree、hash、GiST、GIN、BRIN、SP-GiST 等 AM 都要通过同一套 executor 和 planner 入口工作。
索引还必须足够快。
IndexAM 的 hot path 希望只比较 key、移动 scan position、返回 TID、维护少量私有状态。
但是持久化真相不在 index tuple 里。
一个 TID 指向的 heap slot 可能是旧版本、HOT chain root、redirect line pointer、recently dead tuple，或者对当前 snapshot 不可见。
一个 index AM 在没有 TableAM 语义的情况下，既不知道如何沿 HOT chain 找到可见版本，也不知道什么时候所有 backend 都不需要某个 TID。
如果让 IndexAM 直接解释 tuple header，会破坏 TableAM 抽象。
如果让 TableAM 完全吞掉索引扫描，又会让每种索引丢掉自己的 key ordering、lossy 匹配、posting list 和空间管理能力。
PostgreSQL 的折中是：
IndexAM 持久化的是 access path 状态。
TableAM 持久化的是 row version 状态。
executor 和 `indexam.c` 把两者串成一个运行时契约。
IndexAM 负责找到候选 TID 或候选页。
TableAM 负责把候选解释成 visible tuple、HOT 后续版本、all-dead hint 和 vacuumable 判定。
二者通过 `ItemPointer`、`Snapshot`、`ScanKey`、`IndexScanDesc`、`TableScanDesc`、`IndexFetchTableData` 和 `TM_IndexDeleteOp` 传递语义。
读完本节，你应该能独立判断：
- 为什么 `amgettuple` 返回 TID，不等于返回可见 tuple。
- 为什么 `index_getnext_slot()` 仍然必须调用 `table_index_fetch_tuple()`。
- 为什么 `xs_recheck` 是 executor 的责任边界，不是 TableAM 自己能完成的全部过滤。
- 为什么 bitmap index scan 必须限制在 MVCC-like snapshot 的语义下。
- 为什么 HOT 让 `table_index_fetch_tuple()` 和 `table_tuple_fetch_row_version()` 不是同一个接口。
- 为什么 `kill_prior_tuple` 只是 index cleanup hint，不是可见性真相。
- 为什么恢复中事务不能相信 primary 产生的 killed tuple hint。
- 为什么 `table_index_delete_tuples()` 要返回 `snapshotConflictHorizon`。
- 为什么 index vacuum callback 和 bottom-up deletion 都必须回到 TableAM 判断 heap TID。
- 为什么 `TableScanDesc`、`IndexScanDesc` 和 `IndexFetchTableData` 的 owner、pin 和 cleanup 顺序不能混用。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `nl -ba <source-file>`；重点源码和辅助阅读统一放在第 3 节的阅读顺序里。

辅助文件不是为了扩展本节主题。
它们只用于确认 callback 类型、scan descriptor 初始化、heap HOT fetch 和 index tuple deletion 的真实实现位置。

## 1. 本节在总主线中的位置

前面课程已经说明一个 heap tuple version 如何被插入、更新、删除、prune 和 vacuum。
也已经说明 B-tree 如何把 key 和 heap TID 组织成可搜索的页结构。
这些课程容易让人形成一个错误直觉：
index tuple 指向 heap tuple，所以 index scan 找到的就是查询要返回的 row。
这个直觉在源码层面不成立。
IndexAM 的持久化对象通常是 index relation 的页、line pointer、index tuple、posting list、pending list 或 summary entry。
TableAM 的持久化对象是 table relation 的 tuple storage、visibility metadata、page-level 辅助状态和版本链。
二者共享的最小物理交汇点通常是 TID。
TID 不是行语义。
TID 只是一个可被 TableAM 解释的定位 token。
因此本节不讲某一个索引的内部算法。
本节讲 executor 如何让两个 AM 在不互相泄漏内部格式的前提下，共同完成一次持久化对象到可见 row 的转换。
本节也不完整展开 `ambuild`、`aminsert`、`ambulkdelete`、`amvacuumcleanup`。
它们将在后续课程展开。
本节只抓住扫描和删除判断两个最能暴露契约边界的路径。

## 2. 核心矛盾与一句话运行模型

核心矛盾可以压缩成一句话：
IndexAM 需要高效持久化 access path，TableAM 需要独占解释 row version 语义，而 executor 需要把二者组合成一个看起来像“扫描表”的算子。
如果没有这个边界，最简单的实现是：
索引直接保存 row 的可见性信息。
索引扫描时，IndexAM 自己判断 `xmin`、`xmax`、HOT chain 和 all-dead。
这看起来少了一次 TableAM callback。
但代价很快爆炸。
第一，索引要理解 heap tuple header。
这会把 heap AM 的实现细节固化进所有 index AM。
第二，未来 TableAM 不能改变 row storage。
列存、压缩表、外部 AM 或不同版本链模型都难以接入。
第三，同一个 index AM 要在普通 scan、bitmap scan、index-only scan、VACUUM、recovery 和 serializable predicate lock 中重复实现 visibility 边界。
第四，standby recovery 中 primary 侧的 killed hint 不一定适用于 standby 的 snapshot horizon。
所以当前源码使用更窄的持久化契约：
IndexAM 可以说“我在索引中找到了一个候选 TID”。
TableAM 再说“这个 TID 对这个 snapshot 是否能产生一个 tuple，以及这个候选是否已经全局 dead”。
IndexAM 可以说“我怀疑这些 index tuple 对应的 table TID 可删”。
TableAM 再说“这些 TID 指向的 table tuple 是否 vacuumable，以及对应删除 WAL 需要的 conflict horizon 是什么”。
本节一句话运行模型：

```text
executor 给 index AM scan key 和 snapshot -> index AM 返回候选 TID 或 bitmap -> table AM 用 snapshot、page pin、buffer lock 和版本链解释 TID -> executor 做 recheck 和 qual -> cleanup 路径再把 all-dead/vacuumable 结果回传给 index AM。
```

注意这个模型中有两个方向的数据流。
读路径是 index 到 table。
IndexAM 产生候选，TableAM 解释候选。
清理路径是 table 到 index。
TableAM 判断候选是否可删除，IndexAM 修改自己的 page。
这两个方向共享 TID，但语义完全不同。

## 3. 核心文件分工与阅读顺序

建议先读 `src/include/access/relscan.h`。
原因是 scan descriptor 是状态边界，而不是实现细节。
`relscan.h:33-74` 定义 `TableScanDescData`。
它包含 `rs_rd`、`rs_snapshot`、`rs_key`、`rs_flags`、parallel scan 指针和 instrumentation 指针。
这是 TableAM scan 的 backend-local 基类。
`relscan.h:128-137` 定义 `IndexFetchTableData`。
它只保存 table relation 和 scan flags。
具体 AM 可以把它嵌入更大的结构。
heap 的实现就是 `IndexFetchHeapData`。
`relscan.h:146-205` 定义 `IndexScanDescData`。
这是普通 index scan 和 bitmap index scan 共用的 descriptor。
关键字段包括 `heapRelation`、`indexRelation`、`xs_snapshot`、`keyData`、`orderByData`、`kill_prior_tuple`、`ignore_killed_tuples`、`xactStartedInRecovery`、`opaque`、`xs_heaptid`、`xs_heap_continue`、`xs_heapfetch`、`xs_recheck` 和 `parallel_scan`。
第二读 `src/include/access/tableam.h`。
`tableam.h:321-537` 定义 `TableAmRoutine`。
这一段把 table AM 的职责分成 slot、table scan、parallel scan、index fetch、tuple operation、index deletion、relation operation、planner 和 executor callback。
本节最重要的是 `index_fetch_begin`、`index_fetch_reset`、`index_fetch_end`、`index_fetch_tuple`、`tuple_fetch_row_version`、`tuple_satisfies_snapshot`、`index_delete_tuples` 和 `scan_bitmap_next_tuple`。
`tableam.h:1245-1314` 是 index fetch wrapper。
它说明 `table_index_fetch_tuple()` 在 index scan 中按 TID fetch tuple，并且可能修改 `tid`。
这正是 HOT 的接口痕迹。
`tableam.h:1334-1358` 定义 `table_tuple_fetch_row_version()`。
它只检查给定 TID 的那一个 row version。
它不承担“一个 index entry 到当前可见版本”的语义。
`tableam.h:1397-1415` 定义 `table_index_delete_tuples()`。
它是 index AM 删除 index tuple 前回到 table AM 的裁决入口。
第三读 `src/include/access/amapi.h`。
`amapi.h:233-326` 定义 `IndexAmRoutine`。
对本节来说，`ambeginscan`、`amrescan`、`amgettuple`、`amgetbitmap`、`amendscan`、`aminsert`、`ambulkdelete` 和 `amvacuumcleanup` 是核心。
注意 `amgettuple` 和 `amgetbitmap` 可以有一个为 NULL。
这意味着 generic executor 不能假设所有 index AM 都支持同一种扫描形态。
第四读 `src/backend/access/index/indexam.c`。
这是 generic index 层。
它不是某个索引的实现。
它把 executor、relcache、predicate lock、TableAM fetch、parallel scan 和 index AM callback 串起来。
关键入口：

```text
index_insert()                 indexam.c:213
index_beginscan()              indexam.c:257
index_beginscan_bitmap()       indexam.c:301
index_beginscan_internal()     indexam.c:325
index_rescan()                 indexam.c:368
index_endscan()                indexam.c:394
index_getnext_tid()            indexam.c:599
index_fetch_heap()             indexam.c:657
index_getnext_slot()           indexam.c:698
index_getbitmap()              indexam.c:743
index_bulk_delete()            indexam.c:772
index_vacuum_cleanup()         indexam.c:793
```

第五读 `src/backend/access/heap/heapam_handler.c`。
这个文件定义 heap table AM 的 callback 表。
`heapam_handler.c:2665-2721` 把 heap 的函数挂到 `TableAmRoutine` 上。
它告诉你 heap AM 对外暴露哪些能力。
但要注意，普通 index fetch 的实际实现不在 handler 文件里。
`heapam_handler.c:2682-2685` 指向 `heapam_index_fetch_begin()`、`heapam_index_fetch_reset()`、`heapam_index_fetch_end()` 和 `heapam_index_fetch_tuple()`。
这些函数实际在 `src/backend/access/heap/heapam_indexscan.c`。
`heapam_handler.c:2094-2142` 实现 bitmap heap scan 每次返回一个 visible tuple 的 callback。
`heapam_handler.c:2508-2658` 是 bitmap heap scan 换页、prune、visibility check 和 lossy/exact page 统计的核心。
第六读 `src/backend/executor/nodeIndexscan.c`。
这个文件说明 executor 如何消费 generic index 层。
`nodeIndexscan.c:81-155` 的 `IndexNext()` 是普通 index scan 主入口。
`nodeIndexscan.c:525-543` 的 `ExecIndexScan()` 把它接到 `ExecScan()`。
`nodeIndexscan.c:791-830` 的 `ExecEndIndexScan()` 负责关闭 scan 和 index relation。
`nodeIndexscan.c:557-596` 的 `ExecReScanIndexScan()` 说明 runtime key、reorder queue 和 index rescan 的 cleanup 顺序。
第七读 `src/backend/executor/nodeBitmapHeapscan.c`。
`nodeBitmapHeapscan.c:101-165` 的 `BitmapTableScanSetup()` 先执行外层 bitmap-producing subplan，再用 `table_beginscan_bm()` 建立 table scan。
`nodeBitmapHeapscan.c:174-220` 的 `BitmapHeapNext()` 逐个调用 `table_scan_bitmap_next_tuple()`。
`nodeBitmapHeapscan.c:274-307` 的 `ExecReScanBitmapHeapScan()` 释放 iterator、rescan table、free bitmap。
`nodeBitmapHeapscan.c:315-380` 的 `ExecEndBitmapHeapScan()` 结束 iterator、关闭 table scan、释放 bitmap。
最后读三个辅助文件。
`src/include/access/genam.h` 给出 generic index API 的声明和 `IndexUniqueCheck`、`IndexVacuumInfo`、`IndexBulkDeleteResult`。
`src/backend/access/index/genam.c` 的 `RelationGetIndexScan()` 初始化 `IndexScanDescData`。
`src/backend/access/heap/heapam_indexscan.c` 实现 heap index fetch 和 HOT chain walk。
`src/backend/access/heap/heapam.c` 的 `heap_index_delete_tuples()` 实现 `table_index_delete_tuples()` 的 heap AM 侧裁决。

## 4. 关键数据结构与状态

第一组状态是 `IndexAmRoutine`。
它是 server-lifetime 的静态 callback 表。
`amapi.h:230-232` 明确 core code 不复制也不释放它。
这意味着它不是每次 scan 的运行时状态。
它是 relation access method 的能力表。
`amcanorder`、`amcanorderbyop`、`amcanunique`、`amcanparallel`、`amsummarizing` 等字段是 planner 和 executor 的能力输入。
`ambeginscan`、`amrescan`、`amgettuple`、`amgetbitmap`、`amendscan` 是运行时入口。
这些 callback 的语义是“索引 AM 如何遍历自己的持久化结构”。
它们不拥有 table tuple visibility。
第二组状态是 `TableAmRoutine`。
它同样是静态 callback 表。
`tableam.h:311-320` 要求通常以 server-lifetime 方式分配，并通过 table AM handler 返回。
本节关注的 callback 分三类。
第一类是 scan callback。
`scan_begin`、`scan_end`、`scan_rescan`、`scan_getnextslot` 管 table scan。
第二类是 index fetch callback。
`index_fetch_begin`、`index_fetch_reset`、`index_fetch_end`、`index_fetch_tuple` 管“index entry 到 table tuple”的短路径。
第三类是 index cleanup callback。
`index_delete_tuples` 管“index AM 候选删除项能否根据 table 状态删除”。
第三组状态是 `IndexScanDescData`。
它是 backend-local scan descriptor。
它由 index AM 的 `ambeginscan` 返回。
但通用字段由 `RelationGetIndexScan()` 初始化。
`genam.c:80-129` 初始化 `heapRelation = NULL`、`xs_heapfetch = NULL`、`indexRelation`、`xs_snapshot = InvalidSnapshot`、key 数组、orderby 数组、`kill_prior_tuple`、`xactStartedInRecovery`、`ignore_killed_tuples`、`opaque` 和 index-only scan 相关字段。
`opaque` 属于具体 IndexAM。
core code 不能解释它。
`xs_heaptid` 是 index AM 返回给 generic layer 的候选 TID。
`xs_heapfetch` 是 generic index layer 为 table AM fetch 准备的状态。
`xs_heap_continue` 表示同一个 index entry 可能还要继续产生 table tuple。
heap 中这主要来自 HOT chain 和非 MVCC snapshot。
`xs_recheck` 表示 index AM 返回的是 lossy 或需要重新检查的匹配。
executor 必须用原始 qual 再判断。
第四组状态是 `TableScanDescData`。
它也是 backend-local。
`relscan.h:33-74` 显示它保存 `rs_rd`、`rs_snapshot`、`rs_nkeys`、`rs_key`、`rs_flags`、parallel scan 指针和 instrumentation。
bitmap heap scan 复用这个结构。
`TableScanDescData.st.rs_tbmiterator` 是 bitmap scan 使用的 iterator。
这解释了为什么 `table_beginscan_bm()` 仍返回 `TableScanDesc`。
它不是普通 seqscan，但有足够共同性使用同一个基类。
第五组状态是 `IndexFetchTableData`。
`relscan.h:128-137` 中只有 `rel` 和 `flags`。
heap AM 在 `heapam.h:120-133` 中把它嵌入 `IndexFetchHeapData`。
heap 的扩展字段包括 `xs_cbuf`、`xs_blk` 和 `xs_vmbuffer`。
注释说得很关键：
如果 `xs_blk` 不是 `InvalidBlockNumber`，则 `xs_cbuf` 中持有 pin。
这说明 index fetch 的资源生命周期不是由 `TupleTableSlot` 单独承担。
scan descriptor 也可能跨多次 `index_getnext_slot()` 保留 heap page pin。
第六组状态是 `TM_IndexDeleteOp`。
`tableam.h:178-277` 说明它由 index AM 在调用 `table_index_delete_tuples()` 时准备。
它包含 `irel`、`iblknum`、`bottomup`、`bottomupfreespace`、`ndeltids`、`deltids` 和 `status`。
`TM_IndexDelete` 保存 table TID 和 status array id。
`TM_IndexStatus` 保存 index page offset、`knowndeletable`、`promising` 和 `freespace`。
这个结构不是持久化状态。
它是 index AM 和 table AM 之间的一次性协商状态。
但它影响持久化修改，因为它决定 index AM 后续能删哪些 index tuple，并且返回的 `snapshotConflictHorizon` 要进 index deletion WAL record。
第七组状态是 `kill_prior_tuple` 和 `ignore_killed_tuples`。
它们在 `relscan.h:159-163` 中定义。
这是 IndexAM 与 TableAM 之间最容易被误读的 hint 状态。
`index_fetch_heap()` 在发现整个 HOT chain 对所有事务 dead 时，把 `kill_prior_tuple` 设成 true。
下一次 `amgettuple` 可以把刚才那个 index entry 标成 killed。
但这只是加速未来扫描的 hint。
它不是 visibility 判定。
也不是 crash recovery 必须依赖的持久化真相。
`genam.c:107-119` 说明恢复中事务必须忽略 killed tuple hints。
原因是 primary 的 xmin horizon 可能晚于 standby。
同一个 killed hint 在 standby 上可能会破坏 MVCC。
第八组状态是 `recheck`。
普通 index scan 中是 `IndexScanDesc.xs_recheck`。
bitmap heap scan 中是 `BitmapHeapScanState.recheck`，由 `table_scan_bitmap_next_tuple()` 输出。
它表达的是“这个候选需要 executor 用原始 qual 再验证”。
它不是可见性。
TableAM 已经在返回 tuple 前做了 snapshot visibility。
recheck 是 index predicate/key semantics 的边界。

## 5. 主流程源码 walkthrough

本节主流程选普通 `IndexScan`。
原因是它完整经过 IndexAM、generic index layer、TableAM、executor recheck 和 cleanup hint。

### 5.1 executor 初始化 scan

入口是 `ExecIndexScan()`。
`nodeIndexscan.c:525-543` 判断是否有 runtime key。
如果 runtime key 还没准备好，先 `ExecReScan()`。
然后按是否存在 `ORDER BY` operator 选择 `IndexNextWithReorder` 或 `IndexNext`。
普通路径使用 `IndexNext()`。
`IndexNext()` 在 `nodeIndexscan.c:81-155`。
它先从 `IndexScanState` 中取出 `EState`、scan direction、`IndexScanDesc`、`ExprContext` 和 result slot。
如果 `node->iss_ScanDesc == NULL`，说明这是第一次执行。
它调用：

```text
index_beginscan(heap rel, index rel, estate snapshot, instrumentation, scan key count, orderby count, flags)
```

源码位置是 `nodeIndexscan.c:111-118`。
这里传入的是 executor 的 snapshot。
这很重要。
IndexAM 能看到 `scan->xs_snapshot`，但最终 table tuple visibility 仍由 TableAM 用这个 snapshot 判断。
`index_beginscan()` 在 `indexam.c:257-292`。
它先拒绝对非 catalog table 使用 historic snapshot 的非法 logical decoding 场景。
然后调用 `index_beginscan_internal()`。
`index_beginscan_internal()` 在 `indexam.c:325-353`。
它检查 relation，检查 `ambeginscan` 是否存在。
如果 IndexAM 不自己处理 predicate lock，则调用 `PredicateLockRelation()`。
之后对 index relation 增加 relcache reference count。
接着调用具体 IndexAM 的 `ambeginscan`。
`ambeginscan` 返回 `IndexScanDesc`。
`index_beginscan_internal()` 设置 `parallel_scan` 和 `xs_temp_snap`。
回到 `index_beginscan()` 后，generic layer 填入 `heapRelation`、`xs_snapshot`、instrumentation。
然后调用 `table_index_fetch_begin(heapRelation, flags)`。
`indexam.c:288-289` 是这一行的关键。
从这一刻开始，一个 index scan descriptor 中同时挂着 IndexAM scan state 和 TableAM fetch state。
它们生命周期不同，但由同一个 `index_endscan()` 收尾。

### 5.2 把 scan key 交给 IndexAM

第一次建立 scan 后，`IndexNext()` 会在 `nodeIndexscan.c:126-129` 调用 `index_rescan()`。
`index_rescan()` 在 `indexam.c:368-387`。
它检查 `amrescan`，确认 key 数量没有变化。
如果已有 `xs_heapfetch`，先调用 `table_index_fetch_reset()`。
然后把 `kill_prior_tuple` 置 false，把 `xs_heap_continue` 置 false。
最后调用具体 IndexAM 的 `amrescan`。
这一步只重置 scan position 和 AM 私有状态。
它不会释放整个 scan descriptor。
heap AM 的 `heapam_index_fetch_reset()` 在 `heapam_indexscan.c:41-51`。
它故意是 no-op。
注释说，避免丢掉 `xs_cbuf` 和 `xs_vmbuffer` 中的 pin，可以在某些 tight nested loop join 中减少重复 pin/unpin。
这说明 reset 不等于 release。
cleanup 语义要看具体 callback。

### 5.3 IndexAM 返回候选 TID

`IndexNext()` 的核心循环在 `nodeIndexscan.c:135-155`。
它调用 `index_getnext_slot(scandesc, direction, slot)`。
`index_getnext_slot()` 在 `indexam.c:698-727`。
如果 `xs_heap_continue` 为 false，它先调用 `index_getnext_tid()`。
`index_getnext_tid()` 在 `indexam.c:599-636`。
它检查 `amgettuple` 必须存在。
然后调用：

```text
scan->indexRelation->rd_indam->amgettuple(scan, direction)
```

具体 IndexAM 在这个 callback 中搜索自己的索引结构。
它把候选 TID 放入 `scan->xs_heaptid`。
它还可以设置 `xs_recheck`，也可以填充 index-only scan 需要的 `xs_itup` 或 `xs_hitup`。
generic layer 在这里不 fetch heap。
它只在 `found` 为 true 时统计 index tuple，并返回 `&scan->xs_heaptid`。
这一步的契约很窄：
IndexAM 说“这个 index entry 满足索引层条件，并指向这个 TID”。
IndexAM 没有说“这个 tuple 对当前 snapshot 可见”。
IndexAM 也没有说“这个 TID 没有 HOT 后续版本”。

### 5.4 TableAM 解释 TID

回到 `index_getnext_slot()`。
拿到 TID 后，它调用 `index_fetch_heap(scan, slot)`。
`index_fetch_heap()` 在 `indexam.c:657-680`。
核心调用是：

```text
table_index_fetch_tuple(scan->xs_heapfetch,
                        &scan->xs_heaptid,
                        scan->xs_snapshot,
                        slot,
                        &scan->xs_heap_continue,
                        &all_dead)
```

`table_index_fetch_tuple()` 是 `tableam.h:1304-1314` 的 inline wrapper。
真正工作交给 table relation 的 `rd_tableam->index_fetch_tuple`。
heap AM 的实现是 `heapam_index_fetch_tuple()`。
它位于 `heapam_indexscan.c:232-298`。
heap AM 首先检查目标 TID 是否落在当前已 pin 的 block 上。
如果不是同一 block，它释放旧 `xs_cbuf`，读取新 heap block，并调用 `heap_page_prune_opt()`。
这一步说明 TableAM 可以在 index scan 的表页访问中做 opportunistic prune。
IndexAM 不参与这件事。
然后 heap AM 对 buffer 加 share content lock。
它调用 `heap_hot_search_buffer()`。
`heap_hot_search_buffer()` 在 `heapam_indexscan.c:89-229`。
它从 index entry 指向的 TID 开始，沿 HOT chain 查找对 snapshot 可见的版本。
如果找到了可见版本，它可能把传入的 `tid` 改成 HOT chain 中实际可见 tuple 的 offset。
这解释了 `tableam.h:1283-1285` 为什么说 `*tid` 可能被修改。
如果没有找到可见 tuple，它返回 false。
如果 `all_dead` 参数非空，它还会判断整个 HOT chain 是否已经对所有事务 dead。
这一步的契约是：
TableAM 负责把 index entry 的 TID 解释成当前 snapshot 下的 tuple。
对 heap 来说，这包括 page pin、buffer lock、visibility check、HOT redirect、HOT chain traversal、predicate lock 和 all-dead 判断。
IndexAM 不知道这些细节。

### 5.5 all-dead hint 回流给 IndexAM

`index_fetch_heap()` 调用 TableAM 后，如果 `found` 为 true，就统计 heap fetch。
然后它处理 `all_dead`。
`indexam.c:669-677` 说明：
如果扫描完整个 HOT chain 后只发现 dead tuple，并且当前事务不是在 recovery 中启动，就把 `scan->kill_prior_tuple` 设为 `all_dead`。
这个 flag 会在下一次 `amgettuple` 中被具体 IndexAM 看到。
例如 B-tree 可以用它把上一个返回的 index tuple 标成 killed。
但 `index_getnext_tid()` 在每次 `amgettuple` 返回后马上把 `kill_prior_tuple` reset 为 false。
这是为了避免 hint 泄漏到错误的 index tuple。
hint 的生命周期只有“上一个返回的 index entry 到下一次 amgettuple 调用”这段很短的窗口。
recovery 中不能使用这个 hint。
`RelationGetIndexScan()` 在 `genam.c:107-119` 明确设置 `ignore_killed_tuples = !xactStartedInRecovery`。
standby 上的 snapshot horizon 可能早于 primary。
primary 认为没人需要看的 tuple，standby 上的查询可能还需要看。
所以 killed hint 不是持久化正确性机制。

### 5.6 executor recheck

`index_getnext_slot()` 返回 true 后，控制回到 `IndexNext()`。
`nodeIndexscan.c:139-151` 检查 `scandesc->xs_recheck`。
如果为 true，executor 用 `indexqualorig` 对已经 fetch 出来的 tuple 再跑一次 `ExecQualAndReset()`。
失败就统计 filtered，然后继续循环。
这一步说明 recheck 不属于 TableAM。
TableAM 已经做了 visibility。
recheck 处理的是 index AM 匹配语义可能 lossy、operator recheck 或 order-by recheck。
在 GiST、GIN、BRIN 等 AM 中，这个边界尤其重要。
一个 index entry 命中可能只表示“可能匹配”。
必须把 tuple 取出后用 executor 表达式再判定。

### 5.7 返回 tuple

如果不需要 recheck，或者 recheck 通过，`IndexNext()` 返回 slot。
上层 `ExecScan()` 再处理 scan qual、projection、instrumentation 等通用逻辑。
注意 slot 中的 tuple 生命周期和底层 buffer pin 有关。
heap index fetch 使用 `ExecStoreBufferHeapTuple()`。
后续 `index_getnext_tid()`、`index_fetch_heap()` 或 `index_endscan()` 会释放旧 pin。
调用者不能把 slot 背后的 buffer pin 语义误解成长期 ownership。

## 6. Bitmap Heap Scan 的旁路 walkthrough

bitmap 路径是同一个契约的另一种组合方式。
它不是一边从 index 取一个 TID 一边 fetch heap。
它先把候选 TID 聚合成 `TIDBitmap`，再按 heap block 访问表页。
入口是 `ExecBitmapHeapScan()`。
`nodeBitmapHeapscan.c:259-267` 把 `BitmapHeapNext()` 接到 `ExecScan()`。
`BitmapHeapNext()` 在 `nodeBitmapHeapscan.c:174-220`。
如果还没有初始化，它先调用 `BitmapTableScanSetup()`。
`BitmapTableScanSetup()` 在 `nodeBitmapHeapscan.c:101-165`。
非 parallel 情况下，它调用 `MultiExecProcNode(outerPlanState(node))`。
外层通常是 `BitmapIndexScan` 或 `BitmapAnd/BitmapOr`。
外层 index 子计划通过 `index_getbitmap()` 把 TID 加入 `TIDBitmap`。
`index_getbitmap()` 在 `indexam.c:743-760`。
它检查 `amgetbitmap` callback，调用具体 index AM 的 `amgetbitmap`，再统计返回的 TID 数。
这里没有 `table_index_fetch_tuple()`。
index AM 只负责生产 bitmap。
可见性稍后才处理。
`indexam.c:730-736` 的注释很重要。
它说 index scan 和后续 heap access 之间没有 interlock。
到 heap access 时，heap slot 可能已经被新 tuple 替换。
所以这个接口只对 MVCC-based snapshot 安全。
这不是实现细节。
这是 bitmap 路径的持久化契约。
`BitmapTableScanSetup()` 之后，如果还没有 table scan descriptor，就调用 `table_beginscan_bm()`。
源码在 `nodeBitmapHeapscan.c:155-160`。
`table_beginscan_bm()` 在 `tableam.h:991-999`。
它设置 `SO_TYPE_BITMAPSCAN | SO_ALLOW_PAGEMODE`。
然后调用 `table_beginscan_common()`。
此后 `BitmapHeapNext()` 循环调用 `table_scan_bitmap_next_tuple()`。
`table_scan_bitmap_next_tuple()` 在 `tableam.h:2034-2046`。
它把控制交给 table AM 的 `scan_bitmap_next_tuple`。
heap AM 的 callback 是 `heapam_scan_bitmap_next_tuple()`。
它在 `heapam_handler.c:2094-2142`。
如果当前 page 的 visible tuple offset 已耗尽，就调用 `BitmapHeapScanNextBlock()`。
`BitmapHeapScanNextBlock()` 在 `heapam_handler.c:2508-2658`。
它从 read stream 取下一个 buffer。
对 exact page，它提取 bitmap 中的 tuple offset。
对 lossy page，它必须扫描整个 page 的 line pointer。
然后它调用 `heap_page_prune_opt()`，加 share buffer lock，做 visibility check。
exact page 会对每个 bitmap offset 调 `heap_hot_search_buffer()`。
lossy page 会逐个 normal line pointer 调 `HeapTupleSatisfiesVisibility()`。
最后把可见 offset 存入 `rs_vistuples`，更新 exact/lossy page 计数，释放 content lock，但保留 buffer pin。
之后 `heapam_scan_bitmap_next_tuple()` 从 `rs_vistuples` 中逐个构造 `HeapTuple`，调用 `ExecStoreBufferHeapTuple()` 放入 slot。
如果 `node->recheck` 为 true，`BitmapHeapNext()` 在 `nodeBitmapHeapscan.c:197-208` 用 `bitmapqualorig` 做 executor recheck。
这个路径体现了同一条原则：
IndexAM 生产候选集合。
TableAM 在访问表页时解释候选。
executor 对 lossy index semantics 做 recheck。
bitmap 只是改变访问顺序和资源模型，不改变语义 ownership。

## 7. 生命周期 / ownership / cleanup

先看普通 index scan。
`ExecInitIndexScan()` 创建 `IndexScanState`。
它打开 scan relation 和 index relation，准备 scan key、orderby key、runtime key context、slot 和 instrumentation。
真正的 `IndexScanDesc` 可能延迟到第一次 `IndexNext()` 才创建。
`index_beginscan()` 创建或接收 `IndexScanDesc`。
具体 IndexAM 的 `ambeginscan` 通常调用 `RelationGetIndexScan()`。
`RelationGetIndexScan()` 在 `genam.c:80-129` 分配 descriptor 和 key array。
generic index layer 对 index relation 增加 refcount。
随后 `table_index_fetch_begin()` 创建 TableAM index fetch 状态。
heap AM 的 `heapam_index_fetch_begin()` 在 `heapam_indexscan.c:27-39`。
它 `palloc0` 一个 `IndexFetchHeapData`。
它初始化 relation、flags、`xs_cbuf`、`xs_blk` 和 `xs_vmbuffer`。
普通执行中，`heapam_index_fetch_tuple()` 可能持有 heap buffer pin 和 visibility map buffer pin。
这些 pin 存在于 `IndexFetchHeapData` 中。
它们不是 ResourceOwner 外的裸资源。
PostgreSQL 的 buffer pin 会登记在当前 backend 的资源管理机制中。
但是正常路径仍应由 AM callback 按顺序释放，避免把释放推给 ERROR cleanup。
`index_rescan()` 的 cleanup 不是完整 cleanup。
它调用 `table_index_fetch_reset()`。
heap AM reset 故意不释放 pin。
这是性能选择。
真正释放发生在 `index_endscan()`。
`index_endscan()` 在 `indexam.c:394-417`。
顺序是：
先 `table_index_fetch_end()`。
再调用 IndexAM 的 `amendscan()`。
再释放 index relcache refcount。
如果是 temp snapshot，注销 snapshot。
最后 `IndexScanEnd()` 释放 descriptor 的 generic storage。
这个顺序不是随意的。
table fetch 可能还持有 heap page pin。
AM endscan 可能释放 index page pin、opaque state 或 mark position。
refcount 要在 AM state 不再访问 relation 后释放。
executor 层的 `ExecEndIndexScan()` 在 `nodeIndexscan.c:791-830`。
它先把 worker instrumentation 合并回 shared memory。
然后 `index_endscan()`。
最后 `index_close(indexRelationDesc, NoLock)`。
如果 scan 从未启动，`iss_ScanDesc` 可以是 NULL。
这就是 lazy begin 的边界。
bitmap heap scan 的 lifecycle 不同。
`BitmapTableScanSetup()` 先执行 bitmap 子计划。
它创建或接收 `TIDBitmap`。
然后通过 `tbm_begin_iterate()` 准备 iterator。
第一次 setup 时调用 `table_beginscan_bm()` 创建 table scan。
`TableScanDesc.st.rs_tbmiterator` 保存 iterator 状态。
`ExecReScanBitmapHeapScan()` 在 `nodeBitmapHeapscan.c:274-307`。
它先结束未耗尽的 bitmap iterator。
然后调用 `table_rescan()` 释放 table scan 可能持有的 page pin。
再 `tbm_free()` 释放 bitmap。
最后重置 `initialized` 和 `recheck`，并 rescan 外层计划。
`ExecEndBitmapHeapScan()` 在 `nodeBitmapHeapscan.c:315-380`。
它先合并 worker stats。
然后结束外层子计划。
如果 table scan 存在，先结束未耗尽 iterator，再 `table_endscan()`。
最后释放 `TIDBitmap`。
parallel bitmap 还有 shared state。
`ParallelBitmapHeapState` 在 `nodeBitmapHeapscan.c:87-93`。
它包含 shared iterator pointer、spinlock、状态和 condition variable。
`BM_INITIAL`、`BM_INPROGRESS`、`BM_FINISHED` 在 `nodeBitmapHeapscan.c:72-77`。
这说明 bitmap 创建可以由一个 worker 负责，其他 worker 等待。
但 table tuple visibility 仍在各 worker 的 table scan callback 中完成。
ERROR/abort 时谁兜底？
短生命周期内存由 executor memory context 和 per-query context 回收。
buffer pin、relcache refcount、snapshot registration 等外部资源应由正常 end/rescan 路径释放。
如果 ERROR 跳过正常路径，ResourceOwner 和 transaction cleanup 会兜底释放 pin、lock、snapshot 等资源。
但课程里的工程要求是：
不要把 ResourceOwner cleanup 当成常规控制流。
正常路径必须清楚地调用 `index_endscan()`、`table_endscan()`、`table_index_fetch_end()`、`tbm_end_iterate()` 和 `tbm_free()`。
长期对象如何失效？
`IndexAmRoutine` 和 `TableAmRoutine` 是 server-lifetime static routine，不按 query 失效。
relation descriptor、opclass support function、relcache 中的 `rd_indam` 和 `rd_tableam` 会受到 relcache invalidation 影响。
scan 开始后通常持有 relation refcount 和锁，防止 scan 期间对象被并发 drop 或重写成不可访问状态。
relation 文件的物理替换、truncate、VACUUM cleanup 和 page deletion 则靠更低层的 lock、buffer pin、WAL 和 smgr 规则保证。

## 8. 正确性机制层次

第一层是 snapshot visibility。
TableAM 用 snapshot 解释 table tuple。
heap AM 中，普通 index fetch 最终调用 `HeapTupleSatisfiesVisibility()`。
bitmap heap scan 也在 heap page 上做 visibility check。
IndexAM 返回 TID 前不能证明该 tuple 对当前 snapshot 可见。
第二层是 HOT/version-chain 语义。
`table_index_fetch_tuple()` 与 `table_tuple_fetch_row_version()` 的区别在 `tableam.h:1297-1302`。
前者用于 index entry 到 table tuple lookup。
它必须支持一个 index entry 到多个 row version 的关系。
heap 的 HOT chain 就是这个关系。
后者只检查给定 TID 那一个 version。
把二者混用会导致旧版本、HOT-only tuple 或 redirect line pointer 处理错误。
第三层是 buffer pin 与 buffer content lock。
heap index fetch 在访问 heap page 时持有 buffer pin。
检查 visibility 时加 share content lock。
返回 slot 后通常释放 content lock，但 pin 可继续存在。
这保证 slot 背后的 tuple bytes 不会因为 buffer eviction 消失。
但 pin 不保证 tuple 对未来 snapshot 仍可见。
pin 只是内存和页生命周期保护。
第四层是 predicate locking 和 SSI。
`index_beginscan_internal()` 在 IndexAM 不处理 predicate locks 时调用 `PredicateLockRelation()`。
heap HOT search 中也会调用 `PredicateLockTID()` 和 `HeapCheckForSerializableConflictOut()`。
这说明 serializable correctness 不是 IndexAM 或 TableAM 单独完成的。
它跨 generic index layer、heap visibility check 和 executor snapshot。
第五层是 recheck。
`xs_recheck` 和 bitmap `recheck` 是 index match semantics 的正确性边界。
它处理 lossy index representation、operator class 近似匹配、bitmap lossy page 和 order-by distance lower bound。
recheck 不是 MVCC visibility。
也不是 heap tuple existence。
第六层是 killed tuple hint。
`kill_prior_tuple` 依赖 TableAM 返回 `all_dead`。
IndexAM 可以用它避免未来返回同一个 dead TID。
但恢复中禁用。
它不能写成“如果 killed 就一定不可见”。
第七层是 vacuumable 和 conflict horizon。
`table_index_delete_tuples()` 由 TableAM 判定哪些 TID 对应 vacuumable table tuple。
返回的 `snapshotConflictHorizon` 会进入 index deletion WAL record。
Hot Standby redo 时可能需要据此产生 recovery conflict。
这说明 index page 删除不是单纯本地页优化。
它会影响 standby snapshot correctness。
第八层是 relation lock 和 relcache refcount。
`index_beginscan()` 注释要求 caller 持有 heap 和 index 的合适 lock。
generic layer 对 index relcache entry 增加 reference count。
这保护的是对象身份和 metadata 生命周期。
它不替代 buffer lock，也不替代 snapshot visibility。

## 9. 错误路径 / 异常路径 / fallback

第一个异常路径是 IndexAM 不支持某个 callback。
`indexam.c` 中的 `CHECK_REL_PROCEDURE` 和 `CHECK_SCAN_PROCEDURE` 会在 callback 为 NULL 时 `elog(ERROR)`。
普通 index scan 要求 `amgettuple`。
bitmap index scan 要求 `amgetbitmap`。
`amapi.h:312-313` 明确这两个 callback 可以为 NULL。
所以 planner 和 executor 不能把所有 index AM 当成同质能力。
第二个异常路径是 reindex 中访问 index。
`indexam.c:76-85` 的 `RELATION_CHECKS` 检查 `ReindexIsProcessingIndex()`。
如果当前 index 正在 rebuild 或 pending rebuild，普通扫描或 retail insert 会 ERROR。
这防止系统 catalog reindex 和用户表达式访问自身 index 时出现不可维护的状态。
第三个异常路径是 logical decoding historic snapshot。
`index_beginscan()` 在 `indexam.c:268-276` 拒绝对非 catalog table 使用 historic snapshot。
`table_index_fetch_begin()` 和 `table_tuple_fetch_row_version()` 也检查 `CheckXidAlive`。
源码位置分别是 `tableam.h:1251-1256` 和 `tableam.h:1349-1355`。
这说明 snapshot 类型不是任意可传。
不同 runtime context 对 TableAM/IndexAM 契约有额外约束。
第四个异常路径是 bitmap scan 的 MVCC 限制。
`index_getbitmap()` 注释说明 index scan 和后续 heap access 没有 interlock。
所以 heap slot 可能在访问时已经被替换。
只有 MVCC snapshot 能让这种先收集 TID、后访问 heap 的模式安全。
`ExecInitBitmapHeapScan()` 在 `nodeBitmapHeapscan.c:397-401` 用 assert 要求 unsafe snapshot 不出现。
第五个异常路径是 recovery 中 killed tuple hint。
`RelationGetIndexScan()` 在 recovery-started transaction 中设置 `ignore_killed_tuples = false`。
`index_fetch_heap()` 也不设置 `kill_prior_tuple`。
这是一个典型 fallback：
为了 standby MVCC correctness，放弃 primary 上可能有效的 index cleanup hint。
结果是 standby 查询可能扫描更多 killed entries。
但不会因为 hint 错误跳过可见 tuple。
第六个异常路径是 `table_index_delete_tuples()` 检测 index corruption。
heap AM 的 `heap_index_delete_tuples()` 调用 `index_delete_check_htid()`。
`heapam.c:8028-8085` 检查 index tuple 的 heap TID 是否越过 heap page line pointer array、指向 unused item，或直接指向 heap-only tuple。
这些都是 index corruption。
它会 `ereport(ERROR, ERRCODE_INDEX_CORRUPTED)`。
这说明 TableAM 在 index deletion 裁决中不仅判断可删，还能发现 index 指针与 table storage 不变量不一致。
第七个异常路径是 bottom-up deletion 给不出结果。
`heap_index_delete_tuples()` 中 bottom-up caller 可能最终 `ndeltids = 0`。
`tableam.h:227-229` 也允许 bottom-up caller 在返回后看到 `ndeltids` 为 0。
IndexAM 必须处理这个 fallback。
它不能假设 speculative 候选一定可删。
第八个异常路径是 bitmap lossy page。
当 `TIDBitmap` 因 `work_mem` 压力变 lossy 时，bitmap heap scan 不再知道精确 offsets。
heap AM 必须扫描整页 line pointer，找出对 snapshot 可见的 tuple。
executor 还必须做 qual recheck。
这是资源压力下的 correctness fallback。
它把 memory pressure 转换成更多 CPU 和 heap page filtering。

## 10. 成本、资源与跨模块传播

第一类成本是 callback indirection。
一次普通 index tuple 命中至少经过 executor、generic index layer、IndexAM、generic index layer、TableAM、heap visibility、executor recheck。
这比单个硬编码 heap-btree path 多了函数指针和状态跳转。
PostgreSQL 接受这部分成本，是为了 TableAM 和 IndexAM 的独立演化。
真正的 hot path 优化在于让每个 callback 内部保留紧凑状态。
例如 heap index fetch 保留当前 heap buffer pin，避免相邻 TID 重复读同一页。
第二类成本是 heap fetch amplification。
IndexAM 返回的候选越多，TableAM 要做的 heap page 访问、visibility check 和 HOT traversal 越多。
一个 low-selectivity index scan 可能在 index 层很快，但在 heap fetch 层变慢。
`pgstat_count_index_tuples()` 和 `pgstat_count_heap_fetch()` 分别统计 index tuples 和 heap fetch。
如果 index tuples 很多但返回 rows 很少，常见原因是 MVCC dead version、lossy recheck、visibility filtering 或 poor selectivity。
第三类成本是 random I/O 与 buffer locality。
普通 index scan 按 index order 返回 TID。
heap block 访问可能随机。
heap index fetch 只能缓存当前 block 的 pin。
如果相邻 index entries 指向不同 heap block，缓存帮助有限。
bitmap heap scan 通过 `TIDBitmap` 按 block 重排访问，降低随机 I/O。
代价是先构建 bitmap，需要内存，并且 lossy fallback 可能增加 recheck。
第四类成本是 HOT chain traversal。
`heap_hot_search_buffer()` 可能沿多个 tuple version 走。
长 HOT chain 会增加 CPU、branch、visibility check 和 predicate lock 成本。
但它避免了非 HOT update 产生的新 index entries。
这就是 heap storage 与 index maintenance 之间的成本传播。
第五类成本是 bitmap memory pressure。
`TIDBitmap` 受 `work_mem` 影响。
exact page 可以只检查 bitmap 中列出的 offsets。
lossy page 必须扫描整个 heap page，并对每个 visible tuple 做 qual recheck。
因此 bitmap scan 的成本不是“index 返回多少 TID”一个变量。
还要看 heap page 分布、work_mem、tuple 密度、lossy page 比例和 recheck qual 成本。
第六类成本是 cleanup hint 的不确定性。
`kill_prior_tuple` 能减少未来 index scan 返回 dead entry。
但它依赖查询实际访问到 dead entry。
如果 workload 不扫描这些 key，hint 不会产生。
如果 standby recovery 禁用 hint，收益也消失。
如果 IndexAM 没有使用 killed hint，generic layer 也不能强制它使用。
第七类成本是 index deletion 裁决。
`heap_index_delete_tuples()` 可能为一个 index page 上的候选 TID 访问多个 heap blocks。
`heapam.c:8093-8096` 注释说这可能产生相当多 I/O，因此使用 prefetch，并合并同一 heap block 的重复访问。
bottom-up deletion 还会限制访问最 promising 的 heap block。
它用成本控制换取前台 cleanup 的可接受延迟。
第八类成本是 WAL 和 recovery conflict。
IndexAM 删除 index tuple 会修改 index page 并写 WAL。
但是否需要 snapshot conflict horizon，要 TableAM 根据 table tuple header 给出。
这把 table visibility horizon 传播到 index WAL。
standby 上 redo index deletion 时，可能因此和查询 snapshot 冲突。
第九类成本是 parallel state。
parallel index scan 需要 `ParallelIndexScanDescData`。
`index_parallelscan_estimate()` 会估算 snapshot serialization 和 AM-specific state。
bitmap heap scan 的 parallel shared state 还包含 condition variable。
这把一个本来 backend-local 的 scan 扩展成 DSM 状态同步问题。
第十类成本是 instrumentation 粒度。
Index scan instrumentation 能统计 nsearches。
Bitmap heap scan instrumentation 能统计 exact/lossy pages 和 I/O。
这些不是免费的。
只有 executor 开启对应 instrumentation 时，相关 flags 才会传播到 table scan。
例如 `BitmapTableScanSetup()` 在 `nodeBitmapHeapscan.c:152-153` 根据 `INSTRUMENT_IO` 设置 `SO_SCAN_INSTRUMENT`。

## 11. 观测与诊断入口

第一类现象：index scan returned rows 少，但 heap fetch 很多。
观察入口：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM t WHERE indexed_col = 42;
```

能看到的：
- plan node 的实际 rows。
- shared/local/temp buffer hit/read/dirtied。
- Index Scan 或 Bitmap Heap Scan 的节点耗时。
- Bitmap Heap Scan 的 `Heap Blocks: exact=... lossy=...`。
看不到的：
- 每个 index entry 是否因为 snapshot 不可见被丢弃。
- `heap_hot_search_buffer()` 走了几步 HOT chain。
- `kill_prior_tuple` 设了多少次。
- TableAM fetch 当前持有哪些 pin。
解释方式：
如果 index scan 很快但整体慢，重点看 heap fetch 和 buffer read。
如果 rows 很少但 buffer 访问多，候选 TID 被 visibility、recheck 或 qual 过滤的概率高。
第二类现象：Bitmap Heap Scan 出现 lossy pages。
观察入口：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM t WHERE a BETWEEN 100 AND 200 OR b BETWEEN 100 AND 200;
```

能看到 `Heap Blocks: lossy=...`。
lossy pages 表示 `TIDBitmap` 退化到 page 粒度。
源码解释是：
`BitmapHeapScanNextBlock()` 在 lossy page 上扫描整页 line pointer。
`BitmapHeapNext()` 对 `node->recheck` 为 true 的 tuple 执行 `bitmapqualorig`。
诊断边界：
lossy 不是 index corruption。
它通常是 bitmap 内存压力、候选分布或 bitmap 操作复杂度导致的 fallback。
第三类现象：standby 上 index scan 比 primary 多做 heap check。
观察入口：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

同时看 standby 是否在 recovery。
源码解释是：
`RelationGetIndexScan()` 在 recovery-started transaction 中不忽略 killed tuples。
`index_fetch_heap()` 也不会设置 `kill_prior_tuple`。
因此 standby 上不能把 primary 侧的 killed hint 当成可见性真相。
第四类现象：VACUUM 或前台 bottom-up deletion 访问大量 heap blocks。
观察入口：

```sql
VACUUM (VERBOSE) t;
EXPLAIN (ANALYZE, BUFFERS) UPDATE ...;
```

如果能用 `perf` 或断点，重点看：

```text
heap_index_delete_tuples
heap_hot_search_buffer
index_delete_sort
bottomup_sort_and_shrink
```

源码解释是：
IndexAM 删除候选 index tuple 前，需要 TableAM 根据 heap TID 判断 vacuumable。
候选分布越离散，heap block 访问越多。
bottom-up deletion 会 shrink 和排序候选，试图限制成本。
第五类现象：`Index Cond` 命中但 `Rows Removed by Index Recheck` 增加。
观察入口：

```sql
EXPLAIN (ANALYZE)
SELECT ...;
```

源码解释是：
IndexAM 设置 `xs_recheck` 或 bitmap scan 设置 `recheck`。
executor 用 `indexqualorig` 或 `bitmapqualorig` 重新验证。
这通常来自 lossy AM、lossy bitmap page 或需要重新验证的 operator semantics。
不要把它解释成 MVCC visibility。
第六类现象：index-only scan 仍然访问 heap。
本节不展开 index-only scan。
但边界相同：
IndexAM 可以返回 index tuple data。
是否可以跳过 heap，要看 visibility map 和 snapshot。
这仍是 TableAM/visibility 边界，不是 IndexAM 单独能证明的事情。
第七类现象：gdb 中观察主链路。
可以设置断点：

```text
break IndexNext
break index_getnext_tid
break index_fetch_heap
break heapam_index_fetch_tuple
break heap_hot_search_buffer
```

观察对象：

```text
scan->xs_heaptid
scan->xs_heap_continue
scan->xs_recheck
scan->kill_prior_tuple
((IndexFetchHeapData *) scan->xs_heapfetch)->xs_blk
((IndexFetchHeapData *) scan->xs_heapfetch)->xs_cbuf
```

注意断点会显著改变 timing。
不要用断点实验判断微观性能。
它适合验证状态迁移。

## 12. 常见误区

误区一：把 index TID 当成可见 row。
TID 只是定位候选。
普通 index scan 必须经 `table_index_fetch_tuple()`。
bitmap heap scan 必须经 `table_scan_bitmap_next_tuple()`。
误区二：把 `table_tuple_fetch_row_version()` 当成 index scan fetch。
它只检查给定 TID 的那个版本。
Index scan 需要处理 HOT chain。
因此应该使用 `table_index_fetch_tuple()`。
误区三：把 `kill_prior_tuple` 当成持久化删除。
它只是 hint。
它可能让 IndexAM 标记 killed entry。
它不等于 index tuple 已经物理删除。
它也不能在 recovery 中作为跳过依据。
误区四：把 `xs_recheck` 解释成 visibility 失败。
`xs_recheck` 是 index match semantics 的 recheck。
visibility 由 TableAM 在 heap fetch 中处理。
误区五：认为 `index_rescan()` 会释放所有 table fetch 资源。
generic layer 会调用 `table_index_fetch_reset()`。
heap AM 的 reset 故意不释放 pin。
完整释放在 `table_index_fetch_end()`。
误区六：认为 bitmap scan 精确保存所有 TID。
`TIDBitmap` 可以退化成 lossy page。
lossy page 上必须扫描整页并 recheck。
误区七：认为 IndexAM deletion 可以只看 index page。
index page 上的 TID 是否可删，需要 TableAM 判断 table tuple 是否 vacuumable。
还可能需要把 `snapshotConflictHorizon` 写入 WAL。
误区八：把 callback 表当成 scan 状态。
`IndexAmRoutine` 和 `TableAmRoutine` 是静态能力表。
`IndexScanDesc`、`TableScanDesc` 和 `IndexFetchTableData` 才是每次 scan 的 runtime state。

## 13. 课堂实验

### 实验一：跟普通 index scan 的 TID 到 tuple 转换

目标：
观察 `xs_heaptid` 如何从 IndexAM 返回，又如何被 heap AM 用 snapshot 和 HOT chain 解释。
准备：
创建一个有索引的表。
插入若干行。
对非索引列做多次 UPDATE，制造 HOT chain。
建议 SQL：

```sql
CREATE TABLE t_contract(id int primary key, payload text) WITH (fillfactor = 70);
INSERT INTO t_contract
SELECT g, repeat('a', 50)
FROM generate_series(1, 10000) g;
UPDATE t_contract SET payload = payload || 'b' WHERE id BETWEEN 1 AND 1000;
UPDATE t_contract SET payload = payload || 'c' WHERE id BETWEEN 1 AND 1000;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM t_contract WHERE id = 42;
```

源码断点：

```text
break index_getnext_tid
break index_fetch_heap
break heapam_index_fetch_tuple
break heap_hot_search_buffer
```

观察：
- `scan->xs_heaptid` 在 `amgettuple` 后是什么。
- `heap_hot_search_buffer()` 是否修改 TID offset。
- `scan->xs_heap_continue` 在 MVCC snapshot 下通常为什么为 false。
- `all_dead` 什么时候可能为 true。
回到源码解释：
IndexAM 返回的是候选 root TID。
heap AM 才能沿 HOT chain 找到 snapshot 可见版本。

### 实验二：制造 bitmap lossy recheck

目标：
观察 bitmap scan 中候选集合如何退化成 lossy page，以及 recheck 如何回到 executor。
准备：
降低 `work_mem`。
构造低选择性 bitmap 条件。
建议 SQL：

```sql
SET work_mem = '64kB';
CREATE TABLE t_bitmap(a int, b int, payload text);
INSERT INTO t_bitmap
SELECT g % 100, g % 200, repeat('x', 100)
FROM generate_series(1, 500000) g;
CREATE INDEX ON t_bitmap(a);
CREATE INDEX ON t_bitmap(b);
ANALYZE t_bitmap;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM t_bitmap WHERE a BETWEEN 1 AND 80 OR b BETWEEN 1 AND 160;
```

源码断点：

```text
break index_getbitmap
break BitmapTableScanSetup
break BitmapHeapScanNextBlock
break heapam_scan_bitmap_next_tuple
break BitmapHeapNext
```

观察：
- `node->tbm` 什么时候建立。
- `tbmres->lossy` 是否出现。
- `*recheck` 如何传给 `BitmapHeapNext()`。
- `lossy_pages` 和 `exact_pages` 如何增长。
回到源码解释：
IndexAM 只生产 bitmap。
heap AM 负责在 page 上做 visibility。
executor 负责 bitmap qual recheck。

### 实验三：观察 index deletion 回到 TableAM

目标：
理解 index AM 删除候选 index tuple 前，为什么还要让 TableAM 裁决。
准备：
使用频繁 UPDATE/DELETE 后执行 VACUUM，或在 B-tree insert path 中制造 bottom-up deletion。
不要求修改产品代码。
如果可以使用断点，设置：

```text
break table_index_delete_tuples
break heap_index_delete_tuples
break index_delete_check_htid
break heap_hot_search_buffer
```

观察：
- `delstate->bottomup` 是 true 还是 false。
- `delstate->ndeltids` 调用前后是否变化。
- `status[i].knowndeletable` 如何变化。
- 返回的 `snapshotConflictHorizon` 是否为 valid XID。
回到源码解释：
IndexAM 提供候选 TID 和 index page offset。
TableAM 判断 table tuple 是否 vacuumable。
IndexAM 根据 TableAM 改写后的 `delstate` 修改自己的 index page。

## 14. 讨论题

1. 为什么 `amgettuple` 只返回 `xs_heaptid`，而不是直接返回 `TupleTableSlot`？
2. 如果一个新的 TableAM 不支持 HOT，它仍然必须如何实现 `table_index_fetch_tuple()`？
3. `table_index_fetch_tuple()` 为什么允许修改传入的 `tid`？
4. 为什么 bitmap scan 的“先收集 TID、后访问 heap”只适合 MVCC-based snapshot？
5. `kill_prior_tuple` 为什么不能在 standby recovery 中使用？
6. `xs_recheck`、snapshot visibility 和 scan qual 三者分别由谁负责？
7. `table_index_delete_tuples()` 返回 `snapshotConflictHorizon`，这说明 index deletion 和 standby 查询之间有什么关系？
8. heap AM 的 `table_index_fetch_reset()` 为什么可以选择不释放 buffer pin？这带来什么性能收益和 cleanup 风险？

## 15. 本节小结

本节唯一主问题是：
IndexAM 为什么只能持久化和返回定位候选，而不能独立决定 tuple visibility、all-dead 和 deletion safety。
核心链路是：
executor 建立 `IndexScanDesc`。
IndexAM 的 `amgettuple` 返回候选 `xs_heaptid`。
generic index layer 调用 `table_index_fetch_tuple()`。
TableAM 用 snapshot、buffer pin、content lock 和版本链解释 TID。
executor 根据 `xs_recheck` 做原始 qual recheck。
cleanup hint 和 deletion 裁决再把 table 侧判断回传给 IndexAM。
核心状态有三类。
`IndexAmRoutine` 和 `TableAmRoutine` 是静态能力表。
`IndexScanDesc`、`TableScanDesc` 和 `IndexFetchTableData` 是 backend-local runtime state。
`TM_IndexDeleteOp` 是 index deletion 时 index AM 与 table AM 的一次性协商状态。
ownership 边界是：
executor 拥有 plan state 和 slot。
generic index layer 拥有 scan wrapper 生命周期和 index relcache refcount。
IndexAM 拥有 `opaque` 和 index page traversal state。
TableAM 拥有 table fetch state、heap buffer pin 和 visibility interpretation。
正常 cleanup 走 `index_endscan()`、`table_index_fetch_end()`、`amendscan()`、`table_endscan()`、`tbm_end_iterate()` 和 `tbm_free()`。
ERROR/abort 时 ResourceOwner 和 memory context 兜底，但不能把兜底当常规释放路径。
正确性不是一个机制完成的。
snapshot 决定可见性。
HOT traversal 决定一个 index entry 能否到达当前 row version。
buffer pin/content lock 保护 page bytes。
predicate lock 和 SSI 保护 serializable 语义。
recheck 保护 lossy index semantics。
WAL conflict horizon 保护 standby recovery correctness。
异常路径的共同模式是：
当信息不完整或 hint 不安全时，系统退回到更保守的解释。
standby 忽略 killed hint。
bitmap lossy page 扫描整页并 recheck。
bottom-up deletion 没有足够证据时返回零个可删项。
callback 缺失或 snapshot 类型不合法时直接 ERROR。
能观测的主要是 plan、buffers、exact/lossy pages、rows removed by recheck、pg_stat 统计和断点中的局部状态。
不能直接观测的是每个 TID 的 HOT traversal 步数、`all_dead` hint 的产生次数、TableAM 内部 pin 的精确持有时间，以及每个 index deletion 候选为何失败。
本节可迁移的系统规律是：
持久化 access path 和持久化 data truth 应该通过最小 token 连接。
这个 token 在 PostgreSQL 中通常是 TID。
但是 token 本身不是语义。
token 必须在拥有语义上下文的模块中解释。
这就是 TableAM 与 IndexAM 的持久化契约。
下一节会沿着这个契约继续看索引创建与维护路径：
`ambuild`、`aminsert` 和普通 DML 维护索引时，候选 TID、WAL、排序、唯一性检查和批量构建如何传播成本与正确性边界。
