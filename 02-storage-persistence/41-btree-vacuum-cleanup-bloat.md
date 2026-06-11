# PostgreSQL B-tree VACUUM、cleanup 与 bloat 控制

## 课程定位

前置知识：

- 你已经知道 PostgreSQL heap tuple 有 MVCC 版本，索引 tuple 保存 heap TID。
- 你已经知道 heap page 上的 line pointer 可以在 `LP_NORMAL`、`LP_DEAD`、`LP_UNUSED` 等状态之间变化。
- 你已经知道 B-tree 使用 Lehman-Yao 风格的 sibling link、high key、page split 来支持并发访问。
- 你不需要先记住所有 nbtree 函数名；本节会按一个 runtime 现象把它们串起来。
本节唯一主问题：
在并发 index scan、page split、heap TID 复用都可能发生时，B-tree VACUUM 怎样安全删除 dead index tuple、回收空页，并尽量控制 index bloat？
本节核心矛盾：

- 系统希望尽快回收 B-tree 空间，否则 dead tuple、空 leaf page、无法复用的 deleted page 会积累成 bloat。
- 系统又不能过早删除或复用结构，因为并发 scan 可能还握着旧 page pin、旧 sibling link、旧 downlink 或旧 TID 语义。
- 因此，B-tree VACUUM 不是单纯“把死东西删掉”，而是在 reclaim 与 concurrent physical navigation 之间维持一个分阶段协议。
学完后你应该能独立判断：

- 一次 `VACUUM` 为什么可能扫描完整个 index，却不让 relation file 变小。
- 为什么 `btbulkdelete()` 必须拿 leaf page cleanup lock，即使该页没有可删 tuple。
- 为什么空 leaf page 被删除后还不能立刻复用。
- 为什么 cleanup-only VACUUM 有时完全跳过 index 物理扫描。
- 为什么 simple deletion、bottom-up deletion、deduplication 可以减缓 bloat，但不能替代 VACUUM 的全局回收链路。
源码基线：

- 本课基于 `/home/highgo/postgres`。
- 本课确认的源码版本为 `master` 上的 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。
- 下面的函数名和字段名都按这个基线书写。
本节只围绕一个 runtime truth：

`VACUUM` 先从 heap 找到可回收 TID，再让每个 index 删除指向这些 TID 的 index tuple，最后 heap 才能把对应 line pointer 变成可复用状态。
这个顺序不能倒过来。
如果 heap 先复用 TID，仍在运行的 index scan 可能把旧 index tuple 指到一个新 heap tuple 上。
这就是本节所有 lock、pin、cycle ID、WAL、FSM 延迟复用逻辑的共同根源。

## 1. 本节在总主线中的位置

本目录讨论 storage 与 persistence。
B-tree VACUUM 处在三个边界之间：

- heap 的 MVCC pruning 与 line pointer 生命周期。
- nbtree 的物理 page graph 与并发导航协议。
- FSM、WAL、visibility horizon、autovacuum 等后台资源推进机制。
从 heap 看，index vacuum 是一个中间阶段。
heap pruning 把可移除版本变成 `LP_DEAD`，并把 TID 收集到 `dead_items`。
index AM 必须先删除所有指向这些 `LP_DEAD` TID 的 index tuple。
随后 heap vacuum 才能把同一批 line pointer 标成 `LP_UNUSED`。
从 B-tree 看，VACUUM 有两类空间回收：

- 删除 leaf page 上的 index tuple 或 posting list 中的部分 TID。
- 删除已经空掉的 B-tree page，并在足够安全时把 deleted page 放进 FSM。
这两类回收的安全边界不同。
tuple 删除关注的是 heap TID 是否可以被 index AM 忘掉。
page 删除关注的是并发 search 或 scan 是否还可能通过旧链接到达这个 page。
所以不要把“删除 dead index tuple”和“回收 B-tree page”混成一个动作。
本节的阅读主线是：

```text
heap lazy VACUUM 收集 dead TID
  -> 对每个 index 调用 ambulkdelete
  -> B-tree 线性扫描 leaf page
  -> 删除普通 tuple 或缩小 posting list
  -> 空 leaf page 进入 two-stage page deletion
  -> deleted page 延迟进入 FSM
  -> amvacuumcleanup 记录 cleanup 状态或触发 cleanup-only scan
```

这个主线解释的是一个“空间何时真正可复用”的问题。
这些相邻模块只在它们改变 reclaim 边界时进入本节。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

`lazy_scan_heap()` 找到 heap 上可移除的 TID，`btbulkdelete()` 在线性 index scan 中删除对应 index tuple，`btvacuumcleanup()` 决定是否还需要 cleanup-only scan，并把未能进入 FSM 的 deleted page 数写回 metapage，留给未来 VACUUM 判断。
这句话包含三个“延迟”：

- heap TID 的复用被延迟到 index tuple 删除之后。
- B-tree page 的复用被延迟到 deleted page 的 `safexid` 对所有相关 scan 安全之后。
- cleanup-only index scan 被延迟到确实可能有足够多未回收 deleted page 时才执行。
这三个延迟对应三类 bloat：

- dead index tuple bloat：旧 heap version 对应的 index entry 还在 leaf page 上。
- page fragmentation bloat：tuple 删除后 leaf page 变稀疏，但 PostgreSQL 不合并部分空 page。
- non-recyclable deleted page bloat：page 已经从 tree unlink，但还没安全进入 FSM。
本节的核心矛盾 不是“VACUUM 快还是慢”。
更准确地说，是：
尽快让空间可复用，与允许并发 scan 在弱 interlock 下继续正确前进，不能同时完全满足。
B-tree 选择的策略是分层推进：

- leaf tuple 删除可以在持 cleanup lock 时完成。
- empty leaf page deletion 分成 half-dead 与 fully deleted 两阶段。
- physical page recycling 再等一个 visibility horizon。
- cleanup-only scan 只在 metapage 记录显示有明显收益时发生。
这个策略的结果是：

- correctness 优先于即时 shrink。
- 空间先变成 index 内部可复用，而不是立刻还给操作系统。
- bloat 控制依赖 workload、TID churn、old snapshot、page split 分布和 VACUUM 节奏。
把这些现象归咎于单个 GUC 通常不够。
你需要先判断 bloat 卡在哪一层：

- heap 还没有把旧版本 pruning 成 `LP_DEAD`。
- index vacuum 被跳过或被 failsafe 中止。
- index tuple 已删但 leaf page 只是变稀疏。
- page 已 deleted 但还没有进入 FSM。
- FSM 有页但后续插入没有复用到对应 key space。

## 3. 核心文件分工与阅读顺序

本课必须先读状态，再读流程。
推荐阅读顺序如下。

1. `src/backend/access/heap/vacuumlazy.c`
   先读 `lazy_scan_heap()`、`lazy_vacuum()`、`lazy_vacuum_all_indexes()`、`lazy_cleanup_all_indexes()`。
   目的不是学习 heap VACUUM 全部逻辑，而是确认 `dead_items` 的生产、消费和 reset 时机。

2. `src/include/access/nbtree.h`
   再读 `BTPageOpaqueData`、`BTMetaPageData`、`BTDeletedPageData`、`BTVacState`、`BTVacuumPostingData`。
   本节最重要的状态边界都在这里：page flags、cycle ID、deleted page 的 `safexid`、pending FSM 数组。

3. `src/backend/access/nbtree/nbtree.c`
   接着读 `btbulkdelete()`、`btvacuumcleanup()`、`btvacuumscan()`、`btvacuumpage()`、`btreevacuumposting()`。
   这是 B-tree VACUUM 的主入口和主循环。

4. `src/backend/access/nbtree/nbtpage.c`
   然后读 `_bt_vacuum_needs_cleanup()`、`_bt_set_cleanup_info()`、`_bt_delitems_vacuum()`、`_bt_pagedel()`、`_bt_pendingfsm_finalize()`。
   这里能看到 tuple 删除、page deletion、FSM 延迟复用、metapage cleanup 信息如何落盘。

5. `src/backend/access/nbtree/nbtinsert.c`
   再读 `_bt_delete_or_dedup_one_page()`、`_bt_simpledel_pass()`，以及 page split 写入 `btpo_cycleid` 的片段。
   这解释了为什么 B-tree bloat 控制不只靠 VACUUM：insert path 会先尝试 simple deletion、bottom-up deletion、deduplication 来避免 split。

6. `src/backend/access/nbtree/nbtdedup.c`
   最后读 `_bt_bottomupdel_pass()` 和 `_bt_update_posting()`。
   前者解释 bottom-up deletion 如何作为 split 前防线；后者解释 VACUUM 如何从 posting list 中删掉部分 TID。

7. `src/backend/access/nbtree/README`
   README 不要当 API 手册读。
   本节重点读这些小节：`Deleting index tuples during VACUUM`、`VACUUM's linear scan, concurrent page splits`、`Deleting entire pages during VACUUM`、`Placing deleted pages in the FSM`、`Simple deletion`、`Bottom-Up deletion`、`Notes about deduplication`。

8. 补充小段：`src/backend/access/nbtree/nbtutils.c`
   只读 `_bt_start_vacuum()`、`_bt_vacuum_cycleid()`、`_bt_end_vacuum()`。
   这解释 cycle ID 的 shared state 与 ERROR cleanup；它服务于主链路，不是本节新主线。
阅读时要抓住四个问题：

- 当前状态是 heap-local、index page-local、backend-local，还是 shared memory。
- 这个状态是在 cleanup lock、write lock、read lock、pin、snapshot horizon 中的哪一个边界下解释。
- 这个状态是 correctness 边界，还是估算、hint、优化。
- 如果当前动作失败，系统是 retry、skip、defer，还是把问题留给下次 VACUUM。

## 4. 关键数据结构与状态

### 4.1 heap 侧的 `dead_items`

`vacuumlazy.c` 中的 `LVRelState.dead_items` 是 heap VACUUM 与 index AM 之间的核心桥。
它保存的是 heap TID，不是 index tuple 地址。
这些 TID 来自 heap page 上仍为 `LP_DEAD` 的 line pointer。

`lazy_scan_prune()` 在拿到 heap page cleanup lock 时 pruning、freezing，并把剩余 `LP_DEAD` offset 加到 `dead_items`。

`lazy_scan_noprune()` 在拿不到 cleanup lock 时也可能收集已有 `LP_DEAD` 项，但不能完成需要 cleanup lock 的 pruning/freezing。

`dead_items_info->num_items` 记录当前批次 TID 数。

`TidStoreMemoryUsage(vacrel->dead_items)` 记录当前批次内存占用。
串行 VACUUM 下，`dead_items` 是 backend-local `TidStore`。
并行 VACUUM 下，`dead_items` 和 `dead_items_info` 分配在 DSM 中。

`dead_items` 的 owner 是当前 VACUUM。
它不是持久状态。
它也不是“所有 dead tuple”的全局真相。
内存满时，`lazy_scan_heap()` 会暂停 heap scan，先跑一轮 `lazy_vacuum()` 消费当前批次。
这就是为什么一次 VACUUM 可能多次扫描同一个 index。

### 4.2 `BTPageOpaqueData`

每个 B-tree page 的 special space 中都有 `BTPageOpaqueData`。
本节关心这些字段：

- `btpo_prev`：左 sibling。
- `btpo_next`：右 sibling。
- `btpo_level`：0 表示 leaf page。
- `btpo_flags`：page 状态位。
- `btpo_cycleid`：最近一次 split 关联的 VACUUM cycle ID。
本节关心这些 flag：

- `BTP_LEAF`：leaf page。
- `BTP_ROOT`：root page。
- `BTP_DELETED`：page 已经从 tree 中删除。
- `BTP_HALF_DEAD`：leaf page 已无 parent downlink，但还没有从 sibling chain unlink 完。
- `BTP_SPLIT_END`：当前 split group 的右边界，帮助 VACUUM backtrack 停止。
- `BTP_HAS_GARBAGE`：历史 hint，现代路径不把它作为主要 gating 条件。
- `BTP_INCOMPLETE_SPLIT`：右 sibling 的 parent downlink 还没补完。
- `BTP_HAS_FULLXID`：deleted page 存储 `BTDeletedPageData`。
这些字段不能单独解释语义。
例如 `BTP_DELETED` 不等于 page 已经可以重用。
只有 `BTP_DELETED` 加上 `BTDeletedPageData.safexid`，再加上 `GlobalVisCheckRemovableFullXid()` 的判断，才表示 page 可以进入 FSM。
又如 `btpo_cycleid` 不表示“这个 page 正在被 VACUUM”。
它表示这个 page 的 split 发生在某个 active VACUUM cycle 可见的时间窗口内。
VACUUM 用它判断物理线性扫描是否可能漏掉被 split 移到低 block number 的 tuple。

### 4.3 `BTMetaPageData`

metapage 的常规职责是记录 root、fast root、level 等 B-tree 全局入口。
本节额外关心两个字段：

- `btm_last_cleanup_num_delpages`
- `btm_last_cleanup_num_heap_tuples`
当前版本中 `btm_last_cleanup_num_heap_tuples` 已废弃，cleanup rewrite 时设为 `-1.0`。

`btm_last_cleanup_num_delpages` 记录上次 cleanup 后仍未进入 FSM 的 deleted page 数。

`_bt_vacuum_needs_cleanup()` 用它决定 cleanup-only VACUUM 是否值得做物理 scan。
判断逻辑很保守：

- 旧 metapage 版本需要动态升级时，返回 true。
- 上次遗留 deleted page 数大于 0 且超过当前 index block 数的 5% 时，返回 true。
- 否则 cleanup-only 可以直接跳过 index 物理扫描。
这是 bloat 控制中的一个成本阀门。
没有 dead TID 时，扫描整个 index 只为找少量可回收 deleted page 通常不划算。

### 4.4 `BTDeletedPageData`

deleted page 的 page contents 中保存 `BTDeletedPageData.safexid`。

`BTPageSetDeleted()` 会：

- 清掉 `BTP_HALF_DEAD`。
- 设置 `BTP_DELETED` 和 `BTP_HAS_FULLXID`。
- 调整 page header 的 `pd_lower`、`pd_upper`。
- 把 `safexid` 写入 page contents。

`safexid` 是 page unlink 时读取的 `ReadNextFullTransactionId()`。
它不是删除者的事务 ID。
VACUUM 运行时通常不需要为了删除 B-tree page 而分配自己的 XID。
这里需要的是一个上界：任何可能看过旧 downlink 或旧 sibling link 的 scan，都必须在 horizon 上被覆盖住。

`BTPageIsRecyclable()` 的关键判断是：

`P_ISDELETED(opaque)` 且 `GlobalVisCheckRemovableFullXid(heaprel, safexid)` 为 true。
这表示 deleted page 不再需要充当 tombstone。

### 4.5 `BTVacState`

`BTVacState` 是一次 `btvacuumscan()` 的 backend-local 工作区。
关键字段：

- `info`：`IndexVacuumInfo`，连接 heap relation、index relation、progress、buffer strategy。
- `stats`：`IndexBulkDeleteResult`，累计 VACUUM 输出与上层统计。
- `callback`：bulkdelete 时判断某个 heap TID 是否应删除。
- `callback_state`：callback 的私有状态，B-tree 不解释。
- `cycleid`：本次 `btbulkdelete()` 的 VACUUM cycle ID；cleanup-only scan 为 0。
- `pagedelcontext`：运行 `_bt_pagedel()` 的临时 MemoryContext。
- `pendingpages`：本轮新删除 page 的 `{target, safexid}` 数组。
- `npendingpages`：pending 数量。
- `maxbufsize`：受 `work_mem` 限制的 pending 数组上限。

`BTVacState` 的 lifetime 只覆盖一次 `btvacuumscan()`。
它不是 shared state。
其中 `pendingpages` 的语义是优化，不是 correctness 必需条件。
如果数组达到上限，`_bt_pendingfsm_add()` 会停止记录更多 newly deleted page。
那些 page 仍然已经安全地从 tree unlink，只是可能要等未来 VACUUM 才能进入 FSM。

### 4.6 `BTVacuumPostingData`

posting list tuple 把多个相同 key 的 heap TID 合并在一个 index tuple 中。
VACUUM 可能只删除其中一部分 TID。
这时不能简单删除整个 index tuple。

`BTVacuumPostingData` 描述“从这个 posting list 中删哪些 TID”：

- `itup`：输入时是原始 posting tuple；输出时会变成新分配的更新后 tuple。
- `updatedoffset`：该 posting tuple 在 page 上的 offset。
- `ndeletedtids`：要删除的 posting list 内部 TID 数。
- `deletetids[]`：posting list 内部下标。

`btreevacuumposting()` 在 `btvacuumpage()` 中构造它。

`_bt_update_posting()` 在 `nbtdedup.c` 中生成不含 dead TID 的新 tuple。

`_bt_delitems_vacuum()` 再用 `PageIndexTupleOverwrite()` 覆盖原 tuple。
这说明 deduplication 与 VACUUM 是连接的。
deduplication 改变物理表示，VACUUM 必须能做粒度更细的删除。

## 5. 主流程源码 walkthrough

### 5.1 heap 先生产 TID 批次

主流程从 `vacuumlazy.c` 开始。

`lazy_scan_heap()` 做第一遍 heap scan。
它负责 pruning、freezing、visibility map、FSM 维护，以及收集 index vacuum 需要的 TID。
简化调用链：

```text
lazy_scan_heap()
  -> lazy_scan_prune() 或 lazy_scan_noprune()
     -> dead_items_add()
  -> lazy_vacuum()
     -> lazy_vacuum_all_indexes()
        -> lazy_vacuum_one_index()
           -> vac_bulkdel_one_index()
              -> index AM ambulkdelete
  -> lazy_vacuum_heap_rel()
     -> 把同一批 heap LP_DEAD 标成 LP_UNUSED
  -> lazy_cleanup_all_indexes()
     -> index AM amvacuumcleanup
```

关键顺序是：
先 index，后 heap `LP_UNUSED`。

`lazy_scan_heap()` 的注释明确说，所有 index AM 都依赖这个基本 invariant：
不存在仍然存活的 index tuple 指向 heap 中已经 `LP_UNUSED` 的 line pointer。
原因是 `LP_UNUSED` 允许 TID 被未来 tuple 复用。
如果 index tuple 还在，TID 语义就可能从“旧死 tuple”变成“新活 tuple”。
这不是简单的可见性问题，而是物理地址复用问题。
当 `dead_items` 内存超过上限时，`lazy_scan_heap()` 会中断 heap scan，跑一轮 `lazy_vacuum()`。
因此一次 VACUUM 可能多次调用 `btbulkdelete()`。
B-tree 的 `btvacuumscan()` 每次都会完整扫描 index。
这就是 `maintenance_work_mem` 或 `autovacuum_work_mem` 太小时，index vacuum 成本可能急剧放大的原因。

### 5.2 `btbulkdelete()` 建立 B-tree VACUUM cycle

B-tree handler 在 `nbtree.c` 中把 `ambulkdelete` 指向 `btbulkdelete`。

`btbulkdelete()` 的主体很短，但语义很重：

```text
btbulkdelete()
  -> stats = palloc0_object(IndexBulkDeleteResult) if needed
  -> PG_ENSURE_ERROR_CLEANUP(_bt_end_vacuum_callback)
     -> cycleid = _bt_start_vacuum(rel)
     -> btvacuumscan(info, stats, callback, callback_state, cycleid)
  -> _bt_end_vacuum(rel)
```

`_bt_start_vacuum()` 在 shared memory 的 `btvacinfo` 中登记一个 active VACUUM。
它分配一个非零 `BTCycleId`。
并发 page split 在 `_bt_vacuum_cycleid(rel)` 中能读到这个 cycle ID。
split 会把它写进 split 后 page 的 `btpo_cycleid`。
如果 `btbulkdelete()` ERROR、FATAL 或中途退出，`PG_ENSURE_ERROR_CLEANUP` 确保 `_bt_end_vacuum_callback()` 删除 shared array entry。
否则会泄漏一个 active VACUUM slot。
这个 cleanup 是 shared state ownership，不是普通内存 cleanup。

### 5.3 `btvacuumscan()` 做物理块顺序扫描

`btvacuumscan()` 从 block 1 开始扫，跳过 metapage。
它按物理 block number 读 index page，不按 key order。
这样可以顺序 I/O，也能简单覆盖整个 relation file。
但物理扫描带来一个问题：
并发 split 可能把尚未扫描 page 的一部分 tuple 移到一个已经扫过的低 block number page。
VACUUM 如果只线性向前走，就会漏删这些 tuple。
所以 `btvacuumscan()` 必须配合 `btpo_cycleid` 与 backtracking。
主循环中有几个关键边界：

- 非 local relation 会拿 relation extension lock 读取当前 block 数。
- 这样避免扫描到刚扩展但还没初始化完成的 all-zero page。
- 每轮外循环都会重新读 relation length。
- 这保证扫描覆盖 VACUUM 开始后新增的 page。
- `read_stream_begin_relation()` 用 maintenance/full/batching 风格读取 page。
- 每个 page 在 `vacuum_delay_point(false)` 之后处理，避免持锁时响应 cost delay。

`btvacuumscan()` 初始化的 `stats` 有两个层次：

- `num_pages`、`num_index_tuples`、`pages_deleted`、`pages_free` 是当前 scan 后的 index 状态。
- `tuples_removed`、`pages_newly_deleted` 是整个 VACUUM command 的累计状态，不在每次 scan 开头重置。
这就是为什么同一次 VACUUM 多轮 index scan 时，统计不能简单理解为某一次函数调用的局部计数。

### 5.4 `btvacuumpage()` 处理单页

`btvacuumpage()` 是本节最重要的单页状态机。
它先用 read lock 读 page。
如果 page 是 new page 或已 deleted 且可回收，则调用 `RecordFreeIndexPage()`，并增加 `pages_deleted` 与 `pages_free`。
如果 page 已 deleted 但还不能回收，则只增加 `pages_deleted`。
如果 page 是 half-dead，则设置 `attempt_pagedel`，尝试完成上次中断的 page deletion。
如果 page 是 live leaf page，则升级到 full cleanup lock。
这个 cleanup lock 是本节正确性核心之一。
B-tree tuple 删除本身不一定需要 cleanup lock 才能维持 B-tree 结构。
但 VACUUM 后续会让 heap TID 可复用。
cleanup lock 确保没有 index scan 仍持有该 leaf page pin 并准备访问 heap。
README 里强调：`btbulkdelete` 必须在整个 VACUUM cycle 中拿到每个 leaf page 的 cleanup lock，即使 page 上没有可删 tuple。
原因是某些 scan 返回的 item 可能已经因 split 移到右侧 sibling。
VACUUM 不能只锁有 dead tuple 的 page。
进入 leaf 处理后：

- 如果本页 split 于当前 cycle，且右链接指向低于当前 scan block 的 page，记录 `backtrack_to`。
- 如果有 `callback`，逐个 index tuple 判断其 heap TID 是否在 `dead_items` 中。
- 普通 tuple 全死时加入 `deletable[]`。
- posting list 部分死时加入 `updatable[]`。
- posting list 全死时把整个 offset 加入 `deletable[]`。
- 每页最多调用一次 `_bt_delitems_vacuum()`，减少 WAL 记录数量。
- 删除后更新 `stats->tuples_removed`。
- 如果没有删除但 page 带当前 cycle ID，则把 `btpo_cycleid` 清零并做 hint dirty。
- 如果 leaf page 变空，且不是 backtracking 过程中访问的页，则尝试 `_bt_pagedel()`。
cleanup-only scan 的 `callback` 为 NULL。
这时 `btvacuumpage()` 不逐个判断 TID，也不会删除 index tuple。
它仍然可以回收 old deleted page、完成 half-dead page、删除空 leaf page，并统计页上物理 tuple 数。
因为 posting list 一个物理 tuple 可能代表多个 heap TID，cleanup-only 的 `num_index_tuples` 只能标成 estimate。

### 5.5 posting list 的局部更新

`btreevacuumposting()` 遍历 posting list 中的每个 heap TID。
如果 callback 判定某个 TID dead，就把其 posting list 内部下标写入 `BTVacuumPostingData.deletetids[]`。
三种结果：

- 没有 dead TID，返回 NULL，原 tuple 不动。
- 一部分 dead，返回 `BTVacuumPostingData`，后续覆盖成更短 posting tuple。
- 全部 dead，调用者释放 `BTVacuumPostingData`，把整个 index tuple 删除。

`_bt_delitems_vacuum()` 对 posting list 的更新发生在 critical section 内。
但真正构造新 tuple 的 `_bt_delitems_update()` 在 critical section 之前完成。
这是典型的 PostgreSQL 页面修改模式：
可失败、可分配内存的准备工作放在 critical section 前。
一旦进入 critical section，就不能再 `ereport(ERROR)`，只能完成 WAL 与 page 修改，或者 PANIC。

### 5.6 `_bt_delitems_vacuum()` 删除 leaf tuple

`_bt_delitems_vacuum()` 假设调用者已持有 full cleanup lock。
它做三件事：

- 先用 `PageIndexTupleOverwrite()` 覆盖需要缩小的 posting list tuple。
- 再用 `PageIndexMultiDelete()` 删除完整 index tuple。
- 最后清掉 `btpo_cycleid` 与 `BTP_HAS_GARBAGE`。
WAL 记录类型是 `XLOG_BTREE_VACUUM`。
它不同于 `_bt_delitems_delete()` 使用的 `XLOG_BTREE_DELETE`。
原因是 VACUUM 的 snapshot conflict horizon 由前面的 heap scan 间接负责。
simple deletion 与 bottom-up deletion 发生在 insert path 中，必须通过 table AM 生成自己的 `snapshotConflictHorizon`。
这就是 `_bt_delitems_vacuum()` 与 `_bt_delitems_delete()` 看起来很像，但不能合并成一个函数的原因。

### 5.7 空 leaf page 进入 two-stage page deletion

当 leaf page 已经没有 data item，`btvacuumpage()` 调用 `_bt_pagedel()`。
page deletion 只从空 leaf page 开始。
B-tree 不合并部分空 page。
这对 bloat 诊断很重要：VACUUM 能删除全空 page，却不会把两个 40% 满的 sibling 合并成一个 80% 满的 page。

`_bt_pagedel()` 有几个硬性限制：

- 不能删除 root page。
- 不能删除某一层的 rightmost page。
- 不能删除非空 page。
- 不能删除 incompletely split page。
- internal page 只能作为删除一条 skinny subtree 的一部分被删除。
第一阶段 `_bt_mark_page_halfdead()`：

- 找到 parent 中指向目标 subtree 的 downlink。
- 检查 right sibling 与 parent key space 是否允许把 key space 移到右侧。
- 在 parent 中把目标 downlink 改成 right sibling downlink，并删除后继 pivot tuple。
- 把 leaf page 标成 `BTP_HALF_DEAD`。
- 在 half-dead leaf high key 位置保存 top parent 信息。
- 写 `XLOG_BTREE_MARK_PAGE_HALFDEAD`。
这时 page 已经没有 parent downlink。
但它仍在 sibling chain 中。
并发 search 到达它时可以继续移动到右侧恢复。
第二阶段 `_bt_unlink_halfdead_page()`：

- 按固定顺序锁 left sibling、target、right sibling，避免 deadlock。
- 更新 siblings 的 side link。
- 用 `ReadNextFullTransactionId()` 取 `safexid`。
- 调用 `BTPageSetDeleted()` 把 target 标成 `BTP_DELETED`。
- 必要时更新 metapage fast root。
- 写 `XLOG_BTREE_UNLINK_PAGE` 或 `XLOG_BTREE_UNLINK_PAGE_META`。
- 更新 `pages_newly_deleted`、`pages_deleted`。
- 调用 `_bt_pendingfsm_add()` 记录 newly deleted page。
如果 VACUUM 在 half-dead 阶段后崩溃或中断，tree 对 search 仍一致。
下一次 VACUUM 遇到 `BTP_HALF_DEAD` 会继续删除。

### 5.8 deleted page 进入 FSM

page fully deleted 后仍不能立刻复用。
deleted page 是 tombstone。
它保留 sibling links，让旧 search 或 scan 能从旧物理链接中恢复。

`BTPageIsRecyclable()` 用 `GlobalVisCheckRemovableFullXid(heaprel, safexid)` 判断是否安全。
old deleted page 的处理发生在 `btvacuumpage()` 开头。
如果 page 已经 deleted 且 safexid 足够旧，VACUUM 调用 `RecordFreeIndexPage()`。
newly deleted page 的优化发生在 `_bt_pendingfsm_finalize()`。
本轮 `_bt_pagedel()` 删除的 page 被记录到 `pendingpages`。
scan 结束时，`_bt_pendingfsm_finalize()` 先调用 `GetOldestNonRemovableTransactionId(heaprel)` 强制刷新本 backend 的 horizon 状态。
然后按 `safexid` 顺序检查 pending page。
遇到第一个仍不可回收的 page 就停止。
原因是 `_bt_pendingfsm_add()` 按 safexid 递增顺序记录；后面的 page 不可能更安全。
通过检查的 page 会进入 FSM，并增加 `stats->pages_free`。
最后如果 `pages_free > 0`，`btvacuumscan()` 调用 `IndexFreeSpaceMapVacuum()` 更新上层 FSM page。
如果 pending 数组内存不够，后续 newly deleted page 不会记录。
这不影响 correctness。
它只会让 page 更晚进入 FSM。

### 5.9 `btvacuumcleanup()` 的条件 cleanup

`amvacuumcleanup` 指向 `btvacuumcleanup()`。
它在 heap VACUUM 的 index cleanup phase 中被调用。
如果本次已经调用过 `btbulkdelete()`，`stats` 非 NULL。
此时 `btvacuumcleanup()` 通常不再物理扫描 index，只维护 cleanup metapage 信息。
如果本次没有 dead TID，`stats` 为 NULL。
这时它调用 `_bt_vacuum_needs_cleanup()`。
如果 metapage 显示没有明显遗留 deleted page，直接返回 NULL。
如果有必要 cleanup，则调用 `btvacuumscan(info, stats, NULL, NULL, 0)`。
这就是 cleanup-only scan。
cleanup-only scan 的目的不是删除 heap dead TID 对应的 index tuple。
它是为了：

- 回收 old deleted page。
- 完成 half-dead page。
- 删除扫描时已经为空的 leaf page。
- 更新 index tuple/page 估算统计。
scan 后 `btvacuumcleanup()` 计算：

```text
num_delpages = stats->pages_deleted - stats->pages_free
```

然后 `_bt_set_cleanup_info()` 把这个值 WAL-log 到 metapage。
这个值越大，未来 cleanup-only scan 越可能被触发。

## 6. 生命周期 / ownership / cleanup

### 6.1 `dead_items`

创建者：

- `dead_items_alloc()`。
持有者：

- 当前 heap VACUUM。
- 并行 VACUUM 下由 DSM 承载，worker 通过 parallel vacuum state 访问。
消费者：

- `vac_bulkdel_one_index()` 及各 index AM 的 `ambulkdelete` callback。
释放或 reset：

- 每轮 `lazy_vacuum()` 结束调用 `dead_items_reset()`。
- 整个 VACUUM 结束调用 `dead_items_cleanup()`。
ERROR 兜底：

- local memory 随 VACUUM command memory context 清理。
- parallel mode 由 `parallel_vacuum_end()` 收尾。
语义边界：

- `dead_items` 消失不等于 dead heap tuple 消失。
- 如果 bypass 或 failsafe 发生，heap `LP_DEAD` 可以留下，等待未来 VACUUM。

### 6.2 B-tree VACUUM cycle ID

创建者：

- `btbulkdelete()` 调用 `_bt_start_vacuum()`。
持有者：

- shared memory 中的 `btvacinfo` slot。
- 本次 `btvacuumscan()` 的 `BTVacState.cycleid`。
- 并发 split 写入 page 的 `btpo_cycleid`。
释放者：

- 正常路径 `_bt_end_vacuum()`。
- ERROR/FATAL 路径 `_bt_end_vacuum_callback()`。
语义边界：

- cycle ID 只用于 bulkdelete 物理扫描与并发 split 的 interlock。
- cleanup-only scan 的 cycle ID 为 0。
- cycle ID 可以 false positive，但不能 false negative。

### 6.3 leaf page lock 与 pin

创建者：

- buffer manager 返回 pinned buffer。
- `btvacuumpage()` 用 `_bt_lockbuf()` 获取 read lock，再升级 cleanup lock。
持有者：

- 当前 VACUUM backend。
释放者：

- 普通路径 `_bt_relbuf()`。
- page deletion 路径 `_bt_pagedel()` 自己释放传入 buffer。
语义边界：

- cleanup lock 等待所有 pin 释放。
- cleanup lock 的核心作用是阻止 TID recycling 与仍在访问 heap 的 index scan 交错。
- 不是每个 B-tree 结构修改都需要 cleanup lock；insert path 的 simple deletion 持 exclusive lock。

### 6.4 `_bt_pagedel()` 临时内存

创建者：

- `btvacuumscan()` 创建 `vstate.pagedelcontext`。
持有者：

- `_bt_pagedel()` 及其递归搜索、copy tuple、stack 相关临时对象。
释放者：

- 每次调用前 `MemoryContextReset(vstate.pagedelcontext)`。
- scan 结束 `MemoryContextDelete(vstate.pagedelcontext)`。
原因：

- `_bt_pagedel()` 注释明确说它会泄漏内存。
- 调用者用临时 context 把复杂 cleanup 简化成生命周期边界。

### 6.5 pending FSM 数组

创建者：

- `_bt_pendingfsm_init()`。
持有者：

- `BTVacState`。
更新者：

- `_bt_unlink_halfdead_page()` 通过 `_bt_pendingfsm_add()` 记录 newly deleted page。
释放者：

- `_bt_pendingfsm_finalize()`。
fallback：

- 超过 `work_mem` 派生的 `maxbufsize` 时，不再记录更多 page。
- 未记录 page 仍然 deleted，只是不会在本轮 scan 末尾尝试进入 FSM。

### 6.6 metapage cleanup 信息

创建者或更新者：

- `_bt_set_cleanup_info()`。
持久位置：

- index metapage 的 `btm_last_cleanup_num_delpages`。
WAL：

- `XLOG_BTREE_META_CLEANUP`。
语义：

- 这是 cleanup-only scan 的未来触发 hint。
- 它不是精确 bloat 指标。
- 它只描述 deleted 但尚未 free 的 page 数。

## 7. 正确性机制层次

### 7.1 MVCC visibility

heap pruning 决定哪些 heap tuple version dead to all。
只有 heap 侧能最终判断 TID 是否可以从所有 index 中删除。
B-tree 通过 callback 问“这个 heap TID 是否在 VACUUM 当前 dead set 中”。
B-tree 不自行解释 heap tuple visibility。
这让 index AM 与 table AM 边界清晰。
但 B-tree page recycling 又需要 visibility horizon。
deleted page 的 `safexid` 用 `GlobalVisCheckRemovableFullXid()` 判断。
这里 visibility 保护的是 physical navigation tombstone，不是 heap tuple 可见性。

### 7.2 buffer pin 与 cleanup lock

pin 表示 buffer 正在被使用。
cleanup lock 表示没有其他 backend 持有 pin。
B-tree index scan 可以选择是否在访问 heap 时继续 pin leaf page。
plain MVCC index scan 常常可以提前 drop pin。
index-only scan 不能提前 drop pin。
非 MVCC-like snapshot 的 plain scan 也不能安全 drop pin。
VACUUM 删除 index tuple 前必须拿 cleanup lock。
这是防止 heap TID 复用与正在访问 heap 的 scan 交错。

### 7.3 page split 与 cycle ID

VACUUM 物理线性扫描不是 key-order scan。
并发 split 能把 tuple 移到已扫描过的低 block。

`_bt_start_vacuum()` 登记 active cycle。
split 时 `_bt_vacuum_cycleid()` 返回该 cycle。
split 后左右 page 写入 `btpo_cycleid`。

`btvacuumpage()` 发现本 cycle split 且右链接指向低 block，就 backtrack。

`BTP_SPLIT_END` 帮助 VACUUM 停止继续向右追。
这个机制允许重复访问，避免漏访问。
重复访问最多带来统计误差。
漏访问会破坏 heap TID recycling 安全性。

### 7.4 WAL-before-data

`_bt_delitems_vacuum()`、`_bt_mark_page_halfdead()`、`_bt_unlink_halfdead_page()`、`_bt_set_cleanup_info()` 都在 critical section 中完成 page 修改与 WAL 记录。
进入 critical section 前要完成可能失败的内存分配和 tuple 构造。
page dirty 必须在 `XLogInsert()` 前设置。
LSN 必须写回被修改 page。
crash 后 redo 能重放：

- leaf tuple vacuum。
- half-dead 标记。
- unlink page。
- metapage cleanup 信息更新。
如果 crash 卡在 half-dead 与 unlink 之间，下一次 VACUUM 继续完成。

### 7.5 B-tree physical ordering

page deletion 不能随意合并 key space。
PostgreSQL 选择把被删 page 的 key space 移到右 sibling。
因此不能删除 parent 的 rightmost child，除非 parent 也要一起删除。
也不能删除某层 rightmost page。
这些限制导致 B-tree 高度不会下降。
fast root 是对 skinny tree 的性能补偿，而不是物理高度 shrink。

### 7.6 FSM 与 deleted page tombstone

从 tree unlink 不等于可复用。
deleted page 必须保留一段时间作为 tombstone。
旧 search 可以看到 deleted page，然后沿 sibling link 恢复。
只有当所有可能看到旧链接的 scan 都过了 horizon，page 才能进入 FSM。
FSM 只决定未来 `_bt_allocbuf()` 是否能复用 page。
FSM 不保证 relation file 缩小。

### 7.7 Predicate lock

`_bt_mark_page_halfdead()` 中的 `PredicateLockPageCombine()` 把被删 leaf page 的 predicate lock 合并到 right sibling。
这是 SERIALIZABLE 语义边界。
page deletion 改变 key space 归属，predicate lock 也必须迁移。
否则 page-level predicate lock 会指向不再承载该 key space 的 page。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 active VACUUM slot 泄漏防护

`btbulkdelete()` 使用 `PG_ENSURE_ERROR_CLEANUP` 包住 `_bt_start_vacuum()` 到 `btvacuumscan()`。
如果中途 ERROR，`_bt_end_vacuum_callback()` 删除 shared slot。

`_bt_start_vacuum()` 检测到同一 index 已有 active VACUUM 时会 ERROR。
它在抛 ERROR 前显式释放 `BtreeVacuumLock`。
这不是普通 coding style，而是因为 transaction abort cleanup 还没机会释放这个 LWLock。

### 8.2 heap VACUUM bypass optimization

当 `dead_items` 接近 0 且只会有一轮 index scan 时，`lazy_vacuum()` 可能 bypass index vacuuming。
它把行为近似成“没有 dead items”。
这会留下少量 `LP_DEAD` 与对应 index tuple。
正确性仍然成立，因为 heap 不会把这些 line pointer 变成 `LP_UNUSED`。
代价是 bloat 回收推迟。
收益是避免很小 dead set 触发完整 index scan 的尖刺延迟。

### 8.3 wraparound failsafe

`lazy_vacuum_all_indexes()` 前后都会检查 wraparound failsafe。
如果 relfrozenxid 或 relminmxid 风险太高，VACUUM 可以停止 index vacuuming。
之后不再做相关 heap vacuuming。
这会留下 index bloat 与 heap `LP_DEAD`。
但系统优先推进 freeze，避免 XID wraparound。
这是 correctness 优先级高于空间回收的典型 fallback。

### 8.4 cleanup-only skip

`btvacuumcleanup()` 在 `stats == NULL` 时不一定 scan index。

`_bt_vacuum_needs_cleanup()` 如果认为 cleanup 收益低，直接返回 NULL。
这不是 VACUUM “忘了清理 index”。
它是有意跳过昂贵的完整物理扫描。
只有 metapage 记录的 non-recyclable deleted page 足够多时，cleanup-only scan 才值得做。

### 8.5 half-dead page 恢复

page deletion 第一阶段已经让 tree 对 search 保持一致。
如果 VACUUM 中断，half-dead leaf page 可以留在 index 中。
下一次 `btvacuumpage()` 遇到 `P_ISHALFDEAD` 会调用 `_bt_pagedel()` 继续。
这就是 two-stage deletion 的 crash/abort recovery 设计。
half-dead internal page 是旧版本或 corruption 痕迹。
当前 `_bt_pagedel()` 会 LOG 并建议 REINDEX，而不是假装能安全修复所有历史状态。

### 8.6 sibling link corruption

`_bt_unlink_halfdead_page()` 验证 left sibling、right sibling 的链接。
如果找不到 valid left sibling，或 right sibling 的 left-link 不匹配，它会 LOG index corruption 并放弃这次 page deletion。
它不会为了回收空间而继续修改可能已损坏的结构。
VACUUM 会继续处理 index 的其它部分。
这体现了 fallback 的边界：
能安全跳过的回收动作就跳过。
不能保证结构正确性的动作不强行执行。

### 8.7 pending FSM overflow

`_bt_pendingfsm_add()` 受 `maxbufsize` 限制。
达到上限后，新 deleted page 不再记录到 pending 数组。
这些 page 不会在本轮 scan 结束时走 newly deleted page 优化。
未来 VACUUM 扫到它们时仍可通过 `BTPageIsRecyclable()` 进入 FSM。
这是性能优化降级，不是数据丢失。

### 8.8 incomplete split

VACUUM 通常避免完成 incomplete split，因为补 downlink 可能需要 split parent，进而消耗磁盘空间。
VACUUM 的目标往往是释放空间。
让 VACUUM 在 out-of-space 场景中要求额外扩展 index 是糟糕的 fallback。
例外是 page deletion 需要 re-find parent 时，`_bt_getstackbuf()` 可能完成 internal page 的 incomplete split。
这个例外是为了避免一个 internal incomplete split 阻止大量 leaf page 删除。

## 9. 成本、资源与跨模块传播

### 9.1 成本模型

B-tree VACUUM 的主成本不是只看 dead tuple 数。
更关键的变量包括：

- heap pages scanned：决定 `dead_items` 的生产速度。
- `dead_items` 内存上限：决定一次 VACUUM 中 index 完整扫描次数。
- index pages：`btvacuumscan()` 是全 index 物理扫描。
- leaf page 数：每个 live leaf page 都要拿 cleanup lock。
- posting list 大小：决定 callback 检查与 posting list update 成本。
- empty page 分布：决定 `_bt_pagedel()` 是否大量触发。
- old snapshot 数量与年龄：决定 deleted page 何时进入 FSM。
- WAL 带宽：tuple vacuum、page unlink、metapage cleanup 都可能写 WAL。
- concurrent split 频率：决定 backtracking 与重复计数概率。
如果 `dead_items` 内存不足，一次 heap VACUUM 可能多次触发：

```text
heap scan 一段
  -> full index scan
  -> heap second pass for that batch
  -> reset dead_items
  -> heap scan 下一段
  -> full index scan again
```

这时成本近似变成：

`index_size * index_scan_rounds`。

### 9.2 lock 与 contention

`btvacuumpage()` 对每个 leaf page 获取 cleanup lock。
index-only scan 不能提前 drop leaf pin。
长时间打开的 cursor 或慢查询可能让 VACUUM 等 cleanup lock。
plain MVCC scan 常能 drop pin，减少阻塞。
这就是 README 中强调“尽量允许 scan opt out of holding pin”的原因。
但不能把所有 VACUUM 等待都解释成 lock contention。
也可能是 I/O、WAL、cost delay、old snapshot、parallel worker 分配、heap pruning 成本。

### 9.3 WAL 与 checkpoint 传播

删除 index tuple 写 `XLOG_BTREE_VACUUM`。
simple deletion/bottom-up deletion 写 `XLOG_BTREE_DELETE`。
page deletion 写 `XLOG_BTREE_MARK_PAGE_HALFDEAD` 与 `XLOG_BTREE_UNLINK_PAGE`。
metapage cleanup 写 `XLOG_BTREE_META_CLEANUP`。
这些 WAL 会传播到：

- foreground VACUUM latency。
- WAL insert/flush 压力。
- replication replay。
- checkpoint 后脏页写回。
- standby recovery conflict 处理。
standby 上 replay VACUUM record 也要维护 B-tree 物理结构。
不要只看 primary 的 SQL latency。

### 9.4 FSM 传播

B-tree tuple 删除释放的是 page 内空间。
这不一定更新 FSM。
只有 page 变成 deleted 且安全可复用，才通过 `RecordFreeIndexPage()` 进入 FSM。

`IndexFreeSpaceMapVacuum()` 把上层 FSM page 更新到 searcher 能找到 free page 的状态。
即使 FSM 有 free page，后续 insert 也可能因为 key order、rightmost growth、fillfactor 等原因先 split 别的 page。
FSM 是物理 page 复用入口，不是 bloat 自动消失器。

### 9.5 simple deletion、bottom-up deletion、deduplication 的位置

`nbtinsert.c` 的 `_bt_delete_or_dedup_one_page()` 是 split 前防线。
顺序是：

- simple deletion：先删已有 `LP_DEAD` index tuple，并可能顺便删 nearby extra tuple。
- bottom-up deletion：针对版本 churn 造成的 duplicate，询问 table AM 哪些 TID 可删。
- deduplication：把重复 key 的多个 TID 合并成 posting list，延迟 split。
这些机制控制的是 leaf page split pressure。
它们不替代 heap VACUUM。
它们也不让 heap `LP_DEAD` 变 `LP_UNUSED`。
它们的目标是减少不必要 page split 与长期 bloat 增长速度。

### 9.6 后台进程与 shared state

主动推进本节状态的主要执行者是：

- 用户 backend 执行手动 `VACUUM`。
- autovacuum worker 执行自动 VACUUM。
- parallel vacuum worker 处理多个 index 的 bulkdelete/cleanup。
间接参与者包括：

- walwriter 与 checkpointer：处理 WAL 与脏页持久化压力。
- startup process：standby replay B-tree WAL。
- 其它 backend：通过 old snapshot、pin、concurrent split、insert path deletion 改变 VACUUM 可推进程度。

`BtreeVacuumLock` 保护 active VACUUM cycle shared state。

`dead_items` 在 parallel vacuum 下进入 DSM。
deleted page 的可回收性由 global visibility horizon 决定。
这些状态跨模块传播，但它们不是同一种 shared state。

## 10. 观测与诊断入口

### 10.1 先锁定 runtime truth

诊断本节问题时先问：
空间卡在哪一层？
建议按这个顺序判断：

- heap 是否还有大量 dead tuple。
- VACUUM 是否真的进入 index vacuum phase。
- index vacuum phase 是否多轮发生。
- index cleanup phase 是否被跳过。
- leaf density 是否下降但 page 没有删除。
- deleted page 是否进入 FSM。
- 后续 insert 是否复用 free index page。
不要一上来就说“索引膨胀是 VACUUM 没跑”。
VACUUM 可能跑了，但只完成了 tuple 删除，没有产生可删除空页。
VACUUM 也可能删除了 page，但 old snapshot 让 page 暂时不能进入 FSM。

### 10.2 SQL 与统计入口

`pg_stat_progress_vacuum` 能看到当前 VACUUM phase。
重点关注：

- `scanning heap`
- `vacuuming indexes`
- `vacuuming heap`
- `cleaning up indexes`
- dead item id 数与 dead tuple bytes。
- indexes total 与 processed。
这些指标是当前 backend 粒度。
它们不能告诉你某个 B-tree page 的 `btpo_flags`。

`VACUUM (VERBOSE)` 能看到每次 VACUUM 的 high-level 报告。
重点关注：

- 是否有 index scans。
- 是否触发 parallel workers。
- 是否报告 dead item storage 多次 reset。
- heap 与 index phase 的耗时和 page 数。

`pg_stat_user_tables` 能看：

- `n_dead_tup`
- `vacuum_count`
- `autovacuum_count`
- 最近 vacuum 时间。
这些是表级累计估算。
它们不能直接解释 B-tree leaf density。

`pg_stat_user_indexes` 能看：

- index scan 使用情况。
- tuple read/fetch 关系。
它不是 bloat 视图。

### 10.3 pageinspect 与 pgstattuple

如果允许安装 contrib extension，可以用 `pageinspect` 看 B-tree page 物理状态。
常用入口：

- `bt_metap()`
- `bt_page_stats()`
- `bt_page_items()`
你可以观察：

- page 是 leaf、root、deleted、half-dead 的线索。
- live item 与 dead item 数。
- sibling link。
- posting list 中的 TID 形态。
如果允许安装 `pgstattuple`，可以用 `pgstatindex()` 粗看：

- leaf pages。
- deleted pages。
- average leaf density。
- leaf fragmentation。
这些指标适合判断 bloat 层次。
但它们不是并发时间线。
它们只能告诉你“此刻 index 物理形态大致如何”。

### 10.4 WAL 与源码断点

如果要验证源码主链路，可以用 `pg_waldump` 查 B-tree WAL record。
你会看到 tuple vacuum、delete、unlink、meta cleanup 的记录类型。
这适合确认 page deletion 是否真的发生。
如果要验证状态变化，最直接的是断点：

- `lazy_vacuum_one_index`
- `btbulkdelete`
- `btvacuumscan`
- `btvacuumpage`
- `_bt_delitems_vacuum`
- `_bt_pagedel`
- `_bt_pendingfsm_finalize`
- `btvacuumcleanup`
建议在 `btvacuumpage()` 看：

- `scanblkno`
- `blkno`
- `backtrack_to`
- `opaque->btpo_flags`
- `opaque->btpo_cycleid`
- `ndeletable`
- `nupdatable`
建议在 `_bt_pendingfsm_finalize()` 看：

- `vstate->npendingpages`
- 每个 pending page 的 `safexid`
- `GlobalVisCheckRemovableFullXid()` 返回值。

### 10.5 可直接观测、只能推断、几乎不可见

可直接观测：

- VACUUM 当前 phase。
- dead item storage 大小。
- index relation size。
- extension 输出的 page 状态。
- WAL record 类型。
- VACUUM verbose 报告。
只能推断：

- cleanup-only scan 被跳过是否因为 `_bt_vacuum_needs_cleanup()` 返回 false。
- old snapshot 是否阻止 newly deleted page 进入 FSM。
- concurrent split 是否触发 backtracking。
- simple deletion 与 bottom-up deletion 对避免 split 的贡献。
几乎不可见：

- 某个 plain index scan 是否在某个时刻提前 drop leaf pin。
- cycle ID false positive 是否导致重复访问。
- pending FSM 数组因 `work_mem` 上限丢弃了多少 newly deleted page 记录。
这就是为什么本节诊断不能只靠 `pg_stat_*`。
需要 SQL 现象、page extension、WAL、源码断点或 perf 组合判断。

## 11. 常见误区

误区一：`VACUUM` 后 index 文件应该马上变小。
纠正：B-tree VACUUM 主要让空间在 index 内部可复用。
relation file shrink 不是常规 B-tree VACUUM 的承诺。
误区二：index tuple 删除和 heap tuple 删除是同一个动作。
纠正：heap 先标 `LP_DEAD` 并收集 TID，index 先删对应 entry，heap 才能把 line pointer 变 `LP_UNUSED`。
误区三：`BTP_DELETED` page 已经可以复用。
纠正：还要等 `safexid` 通过 global visibility horizon。
误区四：cleanup-only scan 是每次 VACUUM 必做。
纠正：`btvacuumcleanup()` 可能完全跳过物理 scan。
误区五：`BTP_HAS_GARBAGE` 是现代 simple deletion 的主要触发条件。
纠正：现代代码不再把它作为主要 gating 条件；它更多是历史兼容与 hint。
误区六：deduplication 会删除 dead tuple。
纠正：deduplication 改变物理表示以延迟 split；删除 dead TID 仍要靠 VACUUM、simple deletion 或 bottom-up deletion。
误区七：old snapshot 只影响 heap bloat。
纠正：old snapshot 也可能阻止 B-tree deleted page 进入 FSM，因为 deleted page tombstone 需要等 scan 安全。
误区八：一次 index scan 次数等于一次 VACUUM。
纠正：`dead_items` 内存不足时，一次 heap VACUUM 可能多次完整扫描每个 index。

## 12. 课堂实验

### 实验一：观察 index tuple 删除不等于文件 shrink

目标：

- 看到 VACUUM 删除 index tuple 后，index relation size 未必下降。
- 结合 `pgstatindex()` 或 pageinspect 判断 leaf density 变化。
步骤：

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
DROP TABLE IF EXISTS bt_vac_lab;
CREATE TABLE bt_vac_lab (
    id bigint PRIMARY KEY,
    k bigint NOT NULL,
    pad text
) WITH (autovacuum_enabled = off);
CREATE INDEX bt_vac_lab_k_idx ON bt_vac_lab(k);
INSERT INTO bt_vac_lab
SELECT g, g, repeat('x', 80)
FROM generate_series(1, 200000) AS g;
SELECT pg_size_pretty(pg_relation_size('bt_vac_lab_k_idx')) AS before_size;
SELECT * FROM pgstatindex('bt_vac_lab_k_idx');
UPDATE bt_vac_lab
SET k = k + 100000000
WHERE id % 2 = 0;
VACUUM (VERBOSE, INDEX_CLEANUP ON) bt_vac_lab;
SELECT pg_size_pretty(pg_relation_size('bt_vac_lab_k_idx')) AS after_size;
SELECT * FROM pgstatindex('bt_vac_lab_k_idx');
```

解释入口：

- UPDATE indexed column 产生新的 index tuple。
- VACUUM 删除旧 TID 对应的 index tuple。
- leaf page 可能只是变稀疏，不一定全空。
- 所以 relation size 不一定下降。
源码回扣：

- `lazy_scan_heap()` 收集 `dead_items`。
- `btbulkdelete()` 调用 `btvacuumscan()`。
- `btvacuumpage()` 找到 `deletable[]`。
- `_bt_delitems_vacuum()` 物理删除 tuple。

### 实验二：观察删除连续 key range 后的 deleted page 与复用

目标：

- 尽量制造全空 leaf page。
- 观察 VACUUM 是否产生 deleted pages。
- 再插入相近 key range，看空间是否复用。
步骤：

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
DROP TABLE IF EXISTS bt_page_del_lab;
CREATE TABLE bt_page_del_lab (
    id bigint PRIMARY KEY,
    pad text
) WITH (autovacuum_enabled = off);
INSERT INTO bt_page_del_lab
SELECT g, repeat('x', 50)
FROM generate_series(1, 500000) AS g;
SELECT pg_size_pretty(pg_relation_size('bt_page_del_lab_pkey')) AS initial_size;
SELECT * FROM pgstatindex('bt_page_del_lab_pkey');
DELETE FROM bt_page_del_lab
WHERE id BETWEEN 100000 AND 300000;
VACUUM (VERBOSE, INDEX_CLEANUP ON) bt_page_del_lab;
SELECT pg_size_pretty(pg_relation_size('bt_page_del_lab_pkey')) AS after_vacuum_size;
SELECT * FROM pgstatindex('bt_page_del_lab_pkey');
INSERT INTO bt_page_del_lab
SELECT g, repeat('y', 50)
FROM generate_series(100000, 300000) AS g;
SELECT pg_size_pretty(pg_relation_size('bt_page_del_lab_pkey')) AS after_reinsert_size;
SELECT * FROM pgstatindex('bt_page_del_lab_pkey');
```

解释入口：

- 连续 key range 更容易让若干 leaf page 完全空。
- VACUUM 可以删除 empty leaf page。
- deleted page 进入 FSM 后，后续 insert 可能复用。
- 如果有 old snapshot，deleted page 可能暂时不能进入 FSM。
源码回扣：

- `btvacuumpage()` 中 `minoff > maxoff` 触发 `attempt_pagedel`。
- `_bt_pagedel()` 执行 half-dead 与 unlink。
- `_bt_pendingfsm_finalize()` 判断 newly deleted page 是否能进 FSM。

### 实验三：断点跟读一次 B-tree VACUUM

目标：

- 把 SQL 现象映射到源码状态。
- 观察 `BTVacState`、page flag、posting list update、pending FSM。
建议断点：

```text
b lazy_vacuum_one_index
b btbulkdelete
b btvacuumscan
b btvacuumpage
b _bt_delitems_vacuum
b _bt_pagedel
b _bt_pendingfsm_finalize
b btvacuumcleanup
```

建议观察：

```text
p *vstate
p scanblkno
p blkno
p backtrack_to
p opaque->btpo_flags
p opaque->btpo_cycleid
p ndeletable
p nupdatable
p stats->tuples_removed
p stats->pages_newly_deleted
p stats->pages_deleted
p stats->pages_free
```

实验变化：

- 一轮只删除少量 tuple，观察是否 bypass index vacuuming。
- 降低 `maintenance_work_mem`，观察一次 VACUUM 是否多轮 index scan。
- 保持另一个事务长时间打开 snapshot，再观察 deleted page 是否进入 FSM。
注意：

- 不要求把实验 patch 合入产品代码。
- 如果要加 `elog(DEBUG1, ...)`，只在本地 lab 做。
- 观察目标是状态推进，不是追求稳定复现每个 timing-sensitive 分支。

## 13. 讨论题

1. 为什么 `btbulkdelete()` 要对每个 leaf page 获取 cleanup lock，而不是只锁有 dead tuple 的 page？

2. 为什么 B-tree page 已经 `BTP_DELETED` 后仍不能立刻进入 FSM？

3. cleanup-only scan 被跳过时，系统牺牲了什么，保留了什么 correctness？

4. posting list 中只删除部分 TID 时，为什么需要 `BTVacuumPostingData`，而不是直接把整个 tuple 删除？

5. 为什么 B-tree 不合并两个部分空 leaf page？这个选择怎样影响 bloat 诊断？

6. simple deletion、bottom-up deletion、deduplication 与 VACUUM 的边界分别是什么？

7. 如果一次 VACUUM 报告多次 index scan，你会先检查哪些状态和配置？

8. 哪些现象能从 `pg_stat_progress_vacuum` 直接看到，哪些必须靠 pageinspect、WAL 或断点推断？

## 14. 本节小结

本节唯一主问题是：
在并发访问和 heap TID 复用存在时，B-tree VACUUM 如何安全回收空间并控制 bloat。
核心链路是：

```text
heap pruning 产生 LP_DEAD TID
  -> dead_items 批量传给 index AM
  -> btbulkdelete 建立 VACUUM cycle
  -> btvacuumscan 物理扫描 index
  -> btvacuumpage 删除 tuple 或缩小 posting list
  -> empty leaf page 进入 two-stage page deletion
  -> deleted page 等 safexid 安全后进入 FSM
  -> btvacuumcleanup 更新 cleanup hint 或触发 cleanup-only scan
```

核心状态和边界：

- `dead_items` 是 heap 与 index AM 的批处理桥。
- `BTPageOpaqueData` 描述 page graph 与 page lifecycle。
- `btpo_cycleid` 处理 VACUUM 线性扫描期间的 concurrent split。
- `BTDeletedPageData.safexid` 决定 deleted page 何时可复用。
- `BTVacState.pendingpages` 是 newly deleted page 快速进入 FSM 的优化。
- `BTMetaPageData.btm_last_cleanup_num_delpages` 是 cleanup-only scan 的触发 hint。
ownership 与 cleanup：

- `dead_items` 由 heap VACUUM 创建、消费、reset。
- active VACUUM cycle 由 `_bt_start_vacuum()` 创建，由 `_bt_end_vacuum()` 或 callback 清理。
- `_bt_pagedel()` 的临时泄漏由 `pagedelcontext` 边界吸收。
- pending FSM 数组在 `_bt_pendingfsm_finalize()` 中释放。
错误路径与 fallback：

- ERROR 时必须释放 active cycle shared slot。
- bypass optimization 可以留下少量 `LP_DEAD`，换取避免完整 index scan。
- failsafe 可以停止 index vacuuming，优先防止 wraparound。
- half-dead page 可以留给下一次 VACUUM 完成。
- pending FSM overflow 只推迟 page 复用，不破坏 correctness。
观测边界：

- `pg_stat_progress_vacuum` 能看 phase 和 dead item storage。
- `VACUUM VERBOSE` 能看 high-level index scan 与 worker 信息。
- `pageinspect`、`pgstattuple` 能看 page/state/density 的近似事实。
- WAL 与断点能验证 `_bt_delitems_vacuum()`、`_bt_pagedel()`、`_bt_pendingfsm_finalize()` 是否发生。
- 具体 old snapshot、cycle ID false positive、pin drop timing 往往只能推断。
可迁移 mental model：
PostgreSQL 的空间回收不是一个 delete 动作，而是一条由 visibility、pin、lock、WAL、FSM、cleanup hint 串起来的延迟 pipeline。
当你诊断 bloat 时，先找卡住的是 pipeline 的哪一段。
只有确认卡点之后，才谈参数、schema、SQL pattern、VACUUM 频率或是否需要 REINDEX。
本节结论依赖 workload、并发 snapshot、硬件 I/O、WAL 带宽和具体 PostgreSQL 版本。
不要把某一次实验中的 size 变化当成 B-tree VACUUM 的普遍承诺。
