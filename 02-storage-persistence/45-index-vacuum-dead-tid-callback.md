# PostgreSQL Index vacuum 与 dead TID callback

## 课程定位

上一组课程已经讲过 heap pruning、`LP_DEAD`、FSM/VM、B-Tree page split、dedup 和 bottom-up deletion。
现在把视角放到普通 VACUUM 的 index vacuum 阶段。
前置知识：
- 已理解 MVCC tuple version、HOT chain 和 heap line pointer。
- 已理解 `LP_DEAD` 与 `LP_UNUSED` 的差别。
- 已理解 B-Tree leaf page、posting list、cleanup lock 和 page deletion。
- 已理解 visibility horizon、`OldestXmin`、`GlobalVisState` 和 VM all-visible。
- 已理解 access method callback 是 PostgreSQL 内核跨模块边界的一种形式。
本节唯一主问题：
heap VACUUM 已经知道哪些 heap TID 的旧版本可以被移除，PostgreSQL 如何把这组 dead TID 交给不同 index AM 删除对应 index entry，同时保证 heap line pointer 不会在所有 index entry 删除前被复用？
本节围绕的核心矛盾：
heap 是 MVCC visibility 的裁判。
index AM 是 index page 物理结构的 owner。
heap 不能懂每一种 index AM 的页面格式。
index AM 也不应该重新推导 heap tuple 是否 globally dead。
如果让 heap 逐条调用 index 删除接口，成本会随 dead tuple 数和 index 数爆炸。
如果让 index AM 自己扫描 heap 判断可见性，就会破坏 table AM/MVCC 边界。
如果在删除所有 index entry 前就把 heap `LP_DEAD` 变成 `LP_UNUSED`，旧 index TID 可能指向被复用的 line pointer。
如果为了避免这个风险而永远保留 `LP_DEAD`，heap page 空间又无法彻底回收。
PostgreSQL 的折中是：
heap VACUUM 在第一阶段收集 dead TID 到一个有内存上限的 `TidStore`。
index vacuum 阶段把 `TidStore` 作为 callback state 交给每个 index AM。
index AM 全索引扫描自己的物理结构。
它每看到一个 heap TID，就调用 callback 判断这个 TID 是否在 dead set 中。
所有 index 都处理完后，heap VACUUM 才第二次回到 heap，把同一批 `LP_DEAD` 设成 `LP_UNUSED`。
读完本节，你应该能独立判断：
- 为什么 dead TID callback 是 heap 和 index AM 的边界，而不是普通函数参数。
- 为什么 callback 返回 true 不等于 index AM 做了 MVCC 判断。
- 为什么 `TidStore` 的 memory limit 会转化为多轮全索引扫描。
- 为什么 `IndexBulkDeleteResult` 要跨多次 `ambulkdelete()` 传递。
- 为什么 B-Tree VACUUM 要拿 leaf cleanup lock，并且要处理并发 split。
- 为什么 `INDEX_CLEANUP AUTO/OFF/ON`、bypass optimization 和 failsafe 会改变可见现象。
- 为什么 final heap cleanup 必须在 index vacuum 之后。
本节一句话运行模型：

```text
heap scan/prune 产生 LP_DEAD TID -> TidStore 保存这批 TID -> 每个 index AM 全索引扫描并用 callback 查询 membership -> 所有 index entry 删除完成 -> heap 第二阶段把同一批 LP_DEAD 释放为 LP_UNUSED -> cleanup 更新 index/table stats。
```

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `nl -ba <source-file>`；源码文件分工统一放在第 3 节的阅读顺序里。

课程只依赖当前基线的真实源码。
函数名和状态边界是长期有用的知识。
行号只是定位辅助。

## 1. 本节在总主线中的位置

第 31 节和第 32 节解释了 heap delete 与 pruning。
那里已经看到，旧 tuple version 被剪掉后，line pointer 可能保留为 `LP_DEAD`。
`LP_DEAD` 的核心意义不是“这里还有 tuple bytes”。
它的核心意义是“这个 offset number 还不能被新 tuple 复用”。
原因很直接。
索引 entry 保存的是 heap TID。
heap TID 是 `(block, offset)`。
如果 index entry 还存在，heap offset 就不能被复用成另一个 unrelated tuple。
第 39 节解释了 B-Tree insert path 上的 simple deletion、bottom-up deletion 和 dedup。
那些是前台 insert path 的局部自救。
它们能减轻 index bloat，但不能替代普通 VACUUM。
本节进入普通 VACUUM 的主链路。
它要处理整个 heap relation、所有 index、VM/FSM、统计和 failsafe。
本节只问一个问题：
一批 heap `LP_DEAD` 如何安全穿过 index AM 抽象边界，并最终变成 heap `LP_UNUSED`？
这个问题把三个模块绑在一起：
heap VACUUM 负责发现 dead TID。
`indexam.c` 和 `genam.h` 负责统一 callback API。
具体 index AM 负责删除自己的物理 index entry。
B-Tree 是本节主要 index AM 样例，但 dead TID callback 本身不是 B-Tree 私有机制。
本节不展开所有 AM，只用 B-Tree 展示一个真实复杂实现。

## 2. 核心矛盾与一句话运行模型

核心矛盾可以压缩为一行：

```text
heap 才知道 TID 是否 dead，index AM 才知道 index tuple 在哪里，但 heap TID 复用必须等所有 index AM 都完成删除。
```

这不是简单的分层问题。
它同时是 correctness、cost 和 ownership 问题。
correctness 上，index AM 不能拿一个 heap TID 自己随意判断可删。
它最多能问 heap 传来的 callback：
这个 TID 是否属于本轮 VACUUM 已经确认可删除的集合？
cost 上，heap 不能对每个 dead TID、每个 index 做 point delete。
B-Tree、GIN、GiST 等物理结构都更适合由 AM 自己批量扫描。
ownership 上，heap page 的 `LP_DEAD` 不能由 index AM 直接释放。
index AM 删除的是 index page 上的 tuple。
heap VACUUM 最后释放 heap line pointer。
所以本节的运行模型是：

```text
phase I: heap VACUUM 扫 heap，prune/freeze，收集 LP_DEAD TID 到 TidStore。
phase II: heap VACUUM 对每个 index 调 ambulkdelete，callback 查询 TidStore membership。
phase III: heap VACUUM 只访问 TidStore 中的 heap pages，把对应 LP_DEAD 改成 LP_UNUSED。
final cleanup: 调每个 index 的 amvacuumcleanup，更新统计，可能做 index AM 自己的后处理。
```

这个模型的关键不是“三阶段”这个名字。
关键是 ownership 传递：
heap 发现 dead。
index 删除引用。
heap 释放 line pointer。
任何顺序反过来都可能让旧 index TID 指到新 tuple。

## 3. 核心文件分工与阅读顺序

建议阅读顺序不要按文件名排序。
先读状态，再读入口，再读 AM 实现。
第一组读 heap VACUUM 状态机：
`src/backend/access/heap/vacuumlazy.c`。
重点函数是 `heap_vacuum_rel()`、`lazy_scan_heap()`、`lazy_scan_prune()`、`lazy_scan_noprune()`、`lazy_vacuum()`、`lazy_vacuum_all_indexes()`、`lazy_vacuum_one_index()`、`lazy_vacuum_heap_rel()`、`lazy_vacuum_heap_page()`、`lazy_cleanup_all_indexes()` 和 `dead_items_*()`。
这组代码回答 dead TID 从哪里来、何时触发 index vacuum、何时回到 heap 释放 `LP_DEAD`。
第二组读 index AM 通用 API：
`src/include/access/genam.h`、`src/include/access/amapi.h`、`src/backend/access/index/indexam.c`、`src/backend/commands/vacuum.c`、`src/include/commands/vacuum.h`。
重点对象是 `IndexVacuumInfo`、`IndexBulkDeleteResult`、`IndexBulkDeleteCallback`、`VacDeadItemsInfo`，以及 `index_bulk_delete()`、`index_vacuum_cleanup()`、`vac_bulkdel_one_index()`、`vac_cleanup_one_index()`、`vac_tid_reaped()`。
这组代码回答 callback 类型、callback state、stats 跨轮传递，以及 VACUUM 命令层与 AM 层的边界。
第三组读 B-Tree AM 的 vacuum 实现：
`src/backend/access/nbtree/nbtree.c`、`src/backend/access/nbtree/nbtpage.c`、`src/include/access/nbtree.h`、`src/backend/access/nbtree/README`。
重点函数是 `bthandler()`、`btbulkdelete()`、`btvacuumcleanup()`、`btvacuumscan()`、`btvacuumpage()`、`btreevacuumposting()`、`_bt_delitems_vacuum()`、`_bt_pagedel()`、`_bt_pendingfsm_finalize()` 和 `_bt_vacuum_needs_cleanup()`。
这组代码回答 B-Tree 如何应用 callback、处理 posting list、处理并发 split、删除 empty leaf page，以及延迟复用 deleted index page。
第四组读并行 VACUUM：
`src/backend/commands/vacuumparallel.c` 和 `src/include/commands/vacuum.h`。
重点对象是 `ParallelVacuumState`、`PVIndStats`、`parallel_vacuum_*()`。
它回答 shared `TidStore`、DSM stats、worker 处理 index bulkdelete/cleanup 的边界。
第五组读观测和配置：
`src/include/commands/progress.h`、`src/backend/catalog/system_views.sql`、`doc/src/sgml/ref/vacuum.sgml`、`doc/src/sgml/config.sgml`。
重点字段是 `phase`、`heap_blks_*`、`index_vacuum_count`、`dead_tuple_bytes`、`num_dead_item_ids`、`indexes_total`、`indexes_processed` 和 `mode`。

## 4. 关键数据结构与状态

### 4.1 `LVRelState`

`vacuumlazy.c:252-413` 定义 `LVRelState`。
它是单个 heap relation 的 VACUUM 本地状态。
它不是 shared memory 结构。
普通串行 VACUUM 中，它只属于当前 backend。
parallel vacuum 时，`LVRelState` 仍在 leader backend，本节相关 shared state 放在 `ParallelVacuumState` 和 shared `TidStore` 中。
本节最重要的字段组合：

```text
rel / indrels / nindexes
do_index_vacuuming / do_index_cleanup / do_rel_truncate
dead_items / dead_items_info
lpdead_item_pages / lpdead_items
indstats
num_index_scans
bstrategy / pvs
vistest / cutoffs
phase / blkno / offnum / indname
```

`rel` 是目标 heap relation。
`indrels` 是已打开的 index relation 数组。
`nindexes` 决定是否需要两阶段策略。
当 `nindexes == 0` 时，heap pruning 可以用 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW` 直接把 dead item 设成 `LP_UNUSED`。
当 `nindexes > 0` 时，VACUUM 必须先删除 index entry，再释放 heap line pointer。
`do_index_vacuuming` 表示是否执行 `ambulkdelete`。
`do_index_cleanup` 表示是否执行 `amvacuumcleanup` 和后续 index stats 更新。
它们通常一起为 true。
但 bypass optimization 会把 `do_index_vacuuming` 关掉，同时保留 cleanup。
failsafe 会把二者都关掉。
`dead_items` 是 dead TID set。
它的语义不是“所有 dead tuple”。
它只保存本轮还需要 index AM 删除的 heap TID。
这些 TID 指向 heap page 上的 `LP_DEAD` line pointer。
`dead_items_info` 保存两项补充信息：

```text
max_bytes
num_items
```

`max_bytes` 来自 `maintenance_work_mem` 或 `autovacuum_work_mem`。
但 `TidStoreCreateLocal(max_bytes, ...)` 不强制硬限制。
调用者要用 `TidStoreMemoryUsage()` 观察是否超过，然后触发一轮处理。
`lpdead_item_pages` 是至少有一个 `LP_DEAD` item 的 heap page 数。
`lpdead_items` 是本次 VACUUM 累计的 dead item identifier 数。
`dead_items_info->num_items` 是当前 `TidStore` 中还没 reset 的 item 数。
不要把这三个数混为一谈。
当 VACUUM 多轮 index scan 时，`lpdead_items` 是全局累计。
`dead_items_info->num_items` 是当前轮次的 set 大小。
`indstats` 是每个 index 的 `IndexBulkDeleteResult *`。
它必须跨多个 `ambulkdelete` 调用保留。
`num_index_scans` 是本次 VACUUM 已经启动过多少轮 index vacuum。
它不是 index 个数。
一次 round 会处理所有 index。
如果 dead TID storage 不够，`num_index_scans` 可能大于 1。
`phase / blkno / offnum / indname` 是 error context 状态。
它们服务 `vacuum_error_callback()`。
这说明课程里的“状态”不只服务正常路径。
错误路径也需要明确的生命周期。

### 4.2 `TidStore` 与 `VacDeadItemsInfo`

`tidstore.c:6-12` 说明 `TidStore` 是内存里的 TID 存储结构。
内部用 radix tree。
key 是 `BlockNumber`。
value 是该 block 的 offset bitmap 或小 offset array。
`tidstore.h` 只暴露 opaque API。
本节主要 API：

```text
TidStoreCreateLocal()
TidStoreCreateShared()
TidStoreAttach()
TidStoreDestroy()
TidStoreSetBlockOffsets()
TidStoreIsMember()
TidStoreBeginIterate()
TidStoreIterateNext()
TidStoreGetBlockOffsets()
TidStoreEndIterate()
TidStoreMemoryUsage()
```

`TidStoreSetBlockOffsets()` 要求 offsets 递增。
`lazy_scan_prune()` 因此会先对 `presult.deadoffsets` 排序。
`lazy_scan_noprune()` 从 line pointer 顺序扫描，所以天然递增。
`TidStoreIsMember()` 是 callback 的真正工作。
它先按 block 查 radix tree。
再检查 offset 是否存在。
这使 index AM 不必知道 dead TID set 的物理表示。
`TidStoreBeginIterate()` 服务 heap phase III。
它按 block 迭代，让 heap VACUUM 只回访存在 `LP_DEAD` 的 heap page。
这不是全表第二次扫描。
它是 dead TID set 驱动的 heap page 回访。
parallel vacuum 中，shared `TidStore` 位于 DSA。
leader 和 worker 都持有 backend-local `TidStore *` wrapper。
真实数据在 shared radix tree 中。
`TidStoreLock*()` 只有 shared case 才实际加锁。
本节主路径中，leader 在 heap scan 阶段填充 set。
index vacuum 阶段 worker 读取 set。

### 4.3 `IndexVacuumInfo`

`genam.h:52-62` 定义 `IndexVacuumInfo`。
它是传给 `ambulkdelete` 和 `amvacuumcleanup` 的输入参数。
核心字段：

```text
index
heaprel
analyze_only
report_progress
estimated_count
message_level
num_heap_tuples
strategy
```

`index` 是当前 index relation。
`heaprel` 是它所属的 heap relation。
`num_heap_tuples` 在 `ambulkdelete` 中总是估算值。
`estimated_count` 告诉 AM 统计值是否可信。
`strategy` 传递 buffer access strategy。
这让 index AM 的 IO 行为纳入 VACUUM 的资源控制。
`IndexVacuumInfo` 不包含 dead TID set。
dead TID set 通过 callback state 传入。
这是一个很重要的边界。
AM API 不规定 callback state 的具体类型。
当前 heap VACUUM 传的是 `TidStore *`。

### 4.4 `IndexBulkDeleteResult`

`genam.h:83-92` 定义 `IndexBulkDeleteResult`。
它是 AM 返回给 VACUUM 的 stats。
核心字段：

```text
num_pages
estimated_count
num_index_tuples
tuples_removed
pages_newly_deleted
pages_deleted
pages_free
```

这个结构通常由第一次 `ambulkdelete()` 分配。
后续同一个 VACUUM 中同一个 index 的 `ambulkdelete()` 会复用它。
`amvacuumcleanup()` 也会收到它。
如果没有发生 bulkdelete，`amvacuumcleanup()` 必须准备自己分配。
`IndexBulkDeleteResult` 允许 AM 返回更大的私有结构。
公开结构必须是第一个字段。
这给 AM 在 bulkdelete 与 cleanup 之间传递私有状态留下扩展空间。
本节不要把它当只用于 `VACUUM VERBOSE` 的输出对象。
它也是 AM 跨阶段状态传递的一部分。

### 4.5 `IndexBulkDeleteCallback`

`genam.h:95` 定义：

```text
typedef bool (*IndexBulkDeleteCallback) (ItemPointer itemptr, void *state);
```

callback 输入是 heap TID。
输出是 bool。
true 表示这个 index entry 引用的 heap TID 属于本轮要删除的 dead TID set。
false 表示 AM 不能因本轮 VACUUM 删除它。
这个 bool 不是 visibility verdict。
visibility verdict 已经在 heap phase I 中形成。
callback 是 membership test。
这条边界可以防止 index AM 重新发明 MVCC。

### 4.6 B-Tree `BTVacState`

`nbtree.h:331-347` 定义 `BTVacState`。
它是一次 B-Tree vacuum scan 的本地状态。
核心字段：

```text
info
stats
callback
callback_state
cycleid
pagedelcontext
pendingpages / npendingpages / maxbufsize
```

`callback` 和 `callback_state` 直接来自 `btbulkdelete()`。
cleanup-only scan 中 callback 为 NULL。
`cycleid` 用于处理并发 page split。
`pagedelcontext` 用于运行 `_bt_pagedel()`。
`pendingpages` 记录本次 index vacuum 新删除的 B-Tree pages。
这些 pages 不能一定立刻进入 FSM。
它们要等 `safexid` 对所有可能观察者都不再需要。
`BTVacState` 是 B-Tree 私有状态。
heap VACUUM 不应该依赖它的字段。

## 5. 主流程源码 walkthrough

### 5.1 入口：`heap_vacuum_rel()`

`heap_vacuum_rel()` 是 heap AM 的 VACUUM 主入口。
调用者已经建立事务并打开、锁住 relation。
本节关注它对 index vacuum 的准备。
`vacuumlazy.c:686-696` 分配 `LVRelState` 并挂 error context callback。
`vacuumlazy.c:699-701` 调 `vac_open_indexes()`。
index 用 `RowExclusiveLock` 打开。
这个锁不是 page-level cleanup lock。
它是 relation-level 维护操作所需的锁。
`vacuumlazy.c:724-748` 初始化 index cleanup 相关开关。
默认：

```text
do_index_vacuuming = true
do_index_cleanup = true
consider_bypass_optimization = true
```

`INDEX_CLEANUP OFF` 会同时关闭 index vacuuming 和 cleanup。
`INDEX_CLEANUP ON` 会禁止 bypass optimization。
但 failsafe 仍然可以覆盖它。
`vacuumlazy.c:802-804` 获取 pruning/freeze cutoff 和 `GlobalVisState`。
这一步发生在 heap scan 前。
后续 `lazy_scan_prune()` 用它决定 tuple 是 `DEAD` 还是 `RECENTLY_DEAD`。
`vacuumlazy.c:863-865` 先做 failsafe precheck，再分配 dead item storage。
`dead_items_alloc()` 根据是否能 parallel vacuum 决定 local `TidStore` 或 shared `TidStore`。
`vacuumlazy.c:881` 调 `lazy_scan_heap()`。
从这里开始进入三阶段状态机。

### 5.2 phase I：heap scan 收集 dead TID

`lazy_scan_heap()` 的注释在 `vacuumlazy.c:1243-1277`。
这段注释给出本节最重要的不变量：
只要 relation 有 index，heap VACUUM 不能提前把 `LP_DEAD` 改成 `LP_UNUSED`。
因为所有 index AM 都依赖这个不变量：
没有 extant index tuple 能指向 heap 上已经 `LP_UNUSED` 的 line pointer。
`lazy_scan_heap()` 先设置 progress：

```text
phase = scanning heap
heap_blks_total = rel_pages
max_dead_tuple_bytes = dead_items_info->max_bytes
```

然后按 heap block 读取。
每个 page 尝试拿 cleanup lock。
如果拿到，走 `lazy_scan_prune()`。
如果拿不到，先走 `lazy_scan_noprune()`。
非 aggressive VACUUM 可以接受少做 pruning。
aggressive VACUUM 如果发现必须 freeze，可能必须等待 cleanup lock。
`lazy_scan_prune()` 构造 `PruneFreezeParams`。
有 index 时，它不会设置 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
无 index 时，它设置这个 option，允许 pruning 直接释放 dead item。
这就是有无 index 的关键分叉。
`heap_page_prune_and_freeze()` 返回 `PruneFreezeResult`。
本节关心：

```text
lpdead_items
deadoffsets[]
ndeleted
live_tuples
recently_dead_tuples
newly_all_visible
newly_all_frozen
```

`presult.lpdead_items` 是 pruning 后页面上仍为 `LP_DEAD` 的 item 数。
它包含本次 newly dead 和之前已经存在的 `LP_DEAD`。
如果它大于 0，`lazy_scan_prune()` 会：
先增加 `lpdead_item_pages`。
再对 `deadoffsets` 排序。
再调用 `dead_items_add(vacrel, blkno, deadoffsets, lpdead_items)`。
最后累计 `lpdead_items`。
`dead_items_add()` 调 `TidStoreSetBlockOffsets()`。
它也更新 progress：

```text
num_dead_item_ids = dead_items_info->num_items
dead_tuple_bytes = TidStoreMemoryUsage(dead_items)
```

注意名字里的 `dead_tuple_bytes` 是系统视图列名。
当前源码语义更准确地说是 dead item/TID storage bytes。
如果拿不到 cleanup lock，`lazy_scan_noprune()` 不做 pruning/freeze。
它仍然可以扫描 line pointer。
已经存在的 `LP_DEAD` 会被加入 `TidStore`。
`HEAPTUPLE_DEAD` 但没法 prune 的 tuple 会计入 `missed_dead_tuples`。
这解释了 VACUUM 日志里可能出现 missed dead tuples。
它不是 index callback 漏删。
它是 heap page 没拿到 cleanup lock，无法把某些 heap tuple 推进到 `LP_DEAD`。

### 5.3 内存边界：何时中途做一轮 index vacuum

`lazy_scan_heap()` 在读取下一页前检查 `TidStoreMemoryUsage()`。
如果 `dead_items_info->num_items > 0` 且 memory usage 超过 `max_bytes`，就暂停 heap scan。
它先释放 VM buffer pin。
然后把 `consider_bypass_optimization` 设为 false。
再调用 `lazy_vacuum(vacrel)`。
这表示：
一旦因为内存不够中途进入 index vacuum，就不能再使用“dead TID 很少所以 bypass”的优化。
因为这时 VACUUM 很可能不止一轮 index scan。
中途 `lazy_vacuum()` 完成后，VACUUM 会 vacuum FSM range，然后回到 heap scan。
所以一个 relation 的 VACUUM 可能是：

```text
heap scan prefix
  -> all indexes bulkdelete
  -> heap cleanup for prefix dead_items
  -> TidStore reset
heap scan next prefix
  -> all indexes bulkdelete
  -> heap cleanup
...
heap scan suffix
  -> all indexes bulkdelete
  -> heap cleanup
final index cleanup
```

成本上这很重要。
每次 `TidStore` 满，都要全索引扫描一遍。
dead TID storage 过小会把 index vacuum 成本放大成多轮。

### 5.4 phase II：`lazy_vacuum()` 调所有 index AM

`lazy_vacuum()` 是 phase II 和 phase III 的协调函数。
入口断言：

```text
nindexes > 0
lpdead_item_pages > 0
```

如果 `do_index_vacuuming` 已经是 false，它只 reset `dead_items` 后返回。
这对应 `INDEX_CLEANUP OFF` 或 failsafe/bypass 后的路径。
正常路径先考虑 bypass optimization。
只有第一次、单轮候选情况下才可能 bypass。
条件大致是：
`lpdead_item_pages` 小于 relation page 数的 2%。
且 `TidStoreMemoryUsage()` 小于 32MB。
如果 bypass 成立，VACUUM 表现得像没有 dead TID。
它跳过 index vacuum 和 heap vacuum。
但保留 index cleanup。
这会让少量 `LP_DEAD` 留在 heap 中，等未来 VACUUM 再处理。
如果不 bypass，`lazy_vacuum()` 调 `lazy_vacuum_all_indexes()`。
`lazy_vacuum_all_indexes()` 先做 failsafe precheck。
然后设置 progress：

```text
phase = vacuuming indexes
indexes_total = nindexes
indexes_processed = 0..nindexes
```

串行路径逐个 index 调 `lazy_vacuum_one_index()`。
parallel 路径调用 `parallel_vacuum_bulkdel_all_indexes()`。
每处理一个 index 后，会再次检查 failsafe。
如果 failsafe 触发，当前 round 被标为未完整完成。
函数仍会增加 `num_index_scans`。
这使日志能显示 failsafe 是第几轮之后触发的。

### 5.5 `lazy_vacuum_one_index()` 到 `vac_tid_reaped()`

`lazy_vacuum_one_index()` 构造 `IndexVacuumInfo`。
关键设置：

```text
ivinfo.index = indrel
ivinfo.heaprel = vacrel->rel
ivinfo.analyze_only = false
ivinfo.estimated_count = true
ivinfo.num_heap_tuples = old_live_tuples
ivinfo.strategy = vacrel->bstrategy
```

然后设置 error context：

```text
phase = VACUUM_ERRCB_PHASE_VACUUM_INDEX
indname = current index name
```

接着调用：

```text
vac_bulkdel_one_index(&ivinfo, istat, vacrel->dead_items, vacrel->dead_items_info)
```

`vac_bulkdel_one_index()` 是命令层包装。
它调用：

```text
index_bulk_delete(ivinfo, istat, vac_tid_reaped, dead_items)
```

`index_bulk_delete()` 在 `indexam.c` 里只做通用检查，然后 dispatch 到：

```text
indexRelation->rd_indam->ambulkdelete(...)
```

真正的 callback 是 `vac_tid_reaped()`。
它只有一行语义：

```text
return TidStoreIsMember(dead_items, itemptr);
```

这就是 dead TID callback 的最小核心。
AM 传入任意 index entry 的 heap TID。
callback 回答它是不是本轮 heap VACUUM 已收集的 dead TID。
callback 不知道 index AM。
index AM 不知道 `TidStore` 内部结构。
双方只共享 `ItemPointer` 和 bool。

### 5.6 B-Tree：`btbulkdelete()` 建立 vacuum cycle

B-Tree handler 在 `nbtree.c:120-153` 设置：

```text
ambulkdelete = btbulkdelete
amvacuumcleanup = btvacuumcleanup
amparallelvacuumoptions =
  VACUUM_OPTION_PARALLEL_BULKDEL |
  VACUUM_OPTION_PARALLEL_COND_CLEANUP
```

这说明 B-Tree 支持 parallel bulkdelete。
cleanup 只有在没有 bulkdelete 时才适合并行条件 cleanup。
`btbulkdelete()` 如果 stats 为 NULL，就分配 `IndexBulkDeleteResult`。
然后用 `PG_ENSURE_ERROR_CLEANUP()` 包住 vacuum cycle。
它调用 `_bt_start_vacuum()` 获得 `cycleid`。
再调用：

```text
btvacuumscan(info, stats, callback, callback_state, cycleid)
```

最后 `_bt_end_vacuum()`。
`PG_ENSURE_ERROR_CLEANUP()` 的意义是：
如果中间 ERROR，仍要清理 B-Tree vacuum cycle shared/local 状态。
这不是 memory context 能自动替代的。

### 5.7 B-Tree：`btvacuumscan()` 全索引扫描

`btvacuumscan()` 做三件事。
第一，重置当前 index-wide stats。
它不重置 `tuples_removed` 和 `pages_newly_deleted`。
这两个字段要跨同一 VACUUM 的多轮扫描累计。
第二，初始化 `BTVacState`。
这里把 callback 和 callback_state 存入 `vstate`。
也创建 `_bt_pagedel` 临时 memory context。
第三，按物理 block order 扫 index pages。
它反复读取 relation size。
对非 local relation，它在读 relation length 时拿 relation extension lock。
这是为了避免看到还没初始化完的新 index page。
`btvacuumscan()` 使用 read stream。
它从 `BTREE_METAPAGE + 1` 开始。
每个 buffer 交给 `btvacuumpage()`。
扫描结束后，设置 `stats->num_pages`。
删除临时 context。
再调用 `_bt_pendingfsm_finalize()`。
如果有 reusable pages，调用 `IndexFreeSpaceMapVacuum()`。

### 5.8 B-Tree：`btvacuumpage()` 应用 callback

`btvacuumpage()` 是 B-Tree index vacuum 的核心页级函数。
它先拿 read lock，检查 page 类型。
如果 page 可回收，记录 FSM。
如果 page 已 deleted 但还不能回收，只统计 `pages_deleted`。
如果 page 是 half-dead，尝试继续 page deletion。
如果 page 是 leaf，进入 dead TID callback 主路径。
对 leaf page，B-Tree 会把 read lock 升级为 full cleanup lock。
注释明确说：
即使 page 上没有 deletable tuple，也要在整个 vacuum scan 中拿 cleanup lock。
这是为了和 index scan 的 buffer pin 形成 interlock。
`nbtree/README:166-181` 解释得很清楚：
VACUUM 删除 index tuple 后，随后会让 heap TID 变得可复用。
如果某个 index scan 已从 leaf page 读到 TID，正准备访问 heap，VACUUM 不能在它完成前释放对应 heap slot。
cleanup lock 会等待其他 backend 的 pin 释放。
这比让所有 index scan 都重新校验 heap key 便宜。
接下来，B-Tree 检查 `cycleid`。
如果并发 page split 把尚未处理页面上的 tuple 移到了已经扫过的低 block page，VACUUM 可能漏掉它们。
`btpo_cycleid` 和 backtracking 机制用来发现并补扫这种情况。
对每个 leaf item：
如果不是 posting tuple，直接调用：

```text
callback(&itup->t_tid, callback_state)
```

返回 true，加入 `deletable[]`。
返回 false，计入 live TID。
如果是 posting list tuple，调用 `btreevacuumposting()`。
它遍历 posting list 中的每个 TID。
每个 TID 都调用 callback。
如果没有 TID 要删，返回 NULL。
如果有一部分 TID 要删，返回 `BTVacuumPosting`。
后续 `_bt_delitems_vacuum()` 会重写这个 posting tuple。
如果所有 TID 都要删，整个 posting tuple 加入 `deletable[]`。
这解释了一个常见误区：
posting list 不是一个 TID。
callback 的粒度仍然是 heap TID。
物理删除可能表现为删除整个 index tuple，也可能表现为更新 posting tuple。
一个 leaf page 的所有删除和 posting update 会合并成一次 `_bt_delitems_vacuum()`。
这是为了减少 WAL traffic。
删除后增加 `stats->tuples_removed`。
如果 page 变空，尝试 `_bt_pagedel()`。
否则计数剩余 index TID。
cleanup-only scan 中 callback 为 NULL。
这时 B-Tree 不逐个展开 posting list 计数。
它直接按 index tuple 数估算。
因此 stats 会被标记为 estimated。

### 5.9 B-Tree：`_bt_delitems_vacuum()` 的页修改

`_bt_delitems_vacuum()` 位于 `nbtpage.c:1163-1292`。
调用者必须已经持有 full cleanup lock。
`deletable` 和 `updatable` offset 数组必须递增。
函数先为 posting list update 生成新 tuple 数据。
然后进入 critical section。
顺序是：
先处理 posting tuple update。
再做 simple deletes。
再清 page 的 `btpo_cycleid`。
再清 `BTP_HAS_GARBAGE`。
再 mark buffer dirty。
再写 `XLOG_BTREE_VACUUM`。
最后设置 page LSN 并退出 critical section。
这里有两个边界要记住。
第一，B-Tree VACUUM WAL 和普通 B-Tree delete WAL 不同。
注释说明，普通 delete 要直接生成 snapshot conflict horizon。
VACUUM 依赖初始 heap table scan 的 WAL 间接处理这个问题。
第二，`btpo_cycleid` 由 VACUUM 控制。
普通 single-page cleanup 不能清它。
这保证并发 split/backtracking 的语义只归 VACUUM 管。

### 5.10 B-Tree empty page deletion 与 FSM

如果 leaf page 删除 index tuples 后变空，B-Tree 可能调用 `_bt_pagedel()`。
B-Tree page deletion 是两阶段。
先把 leaf 标 half-dead 并从 parent downlink 移除。
再 unlink sibling links 并标 deleted。
如果 VACUUM 中断或系统崩溃，下一次 VACUUM 会继续处理 half-dead page。
这不是 corruption。
这是 page deletion 协议的一部分。
deleted page 不能马上复用。
`_bt_unlink_halfdead_page()` 会用 `ReadNextFullTransactionId()` 形成 `safexid`。
这个 `safexid` 代表：
在它之前可能已经看到 stale link 的扫描还需要这个 deleted page 作为 tombstone。
`_bt_pendingfsm_add()` 把 newly deleted page 和 safexid 放入 `pendingpages`。
`_bt_pendingfsm_finalize()` 在本次 index scan 结束时重新计算 horizon。
如果 `GlobalVisCheckRemovableFullXid(heaprel, safexid)` 为 true，才把该 page 放进 index FSM。
如果不安全，留给未来 VACUUM。
这说明 index vacuum 删除 index tuple 与 index page recycle 是两个层次。
前者由 dead TID callback 驱动。
后者由 B-Tree physical structure 和 snapshot horizon 驱动。

### 5.11 phase III：回到 heap 释放 `LP_DEAD`

当 `lazy_vacuum_all_indexes()` 成功处理所有 indexes 后，`lazy_vacuum()` 调 `lazy_vacuum_heap_rel()`。
这是 heap 的第二阶段。
它不是全 heap scan。
它调用 `TidStoreBeginIterate(vacrel->dead_items)`。
然后 read stream 只读取 TidStore 里出现的 heap blocks。
对每个 block：
取出 offsets。
pin 对应 VM page。
拿 heap buffer exclusive lock。
调用 `lazy_vacuum_heap_page()`。
`lazy_vacuum_heap_page()` 先检查：
这些 `LP_DEAD` 变成 `LP_UNUSED` 后，page 是否会成为 all-visible/all-frozen。
这个检查可能做 IO 和分配内存，所以必须在 critical section 外。
如果需要设置 VM，它先锁 VM buffer。
然后进入 critical section。
对每个 dead offset：
断言 `ItemIdIsDead(itemid) && !ItemIdHasStorage(itemid)`。
调用 `ItemIdSetUnused(itemid)`。
再尝试 `PageTruncateLinePointerArray(page)`。
如果 VM flags 有变化，设置 `PD_ALL_VISIBLE` 并调用 `visibilitymap_set()`。
mark buffer dirty。
需要 WAL 时调用 `log_heap_prune_and_freeze()`。
reason 是 `PRUNE_VACUUM_CLEANUP`。
最后退出 critical section。
这一步才真正让 heap offset number 可复用。
它能发生的前提是：
本轮所有 indexes 的 bulkdelete 成功完成。
如果 failsafe 中途触发，phase III 不会发生。

### 5.12 final cleanup：`amvacuumcleanup()`

heap scan 结束后，如果 `dead_items_info->num_items > 0`，`lazy_scan_heap()` 会再调用一次 `lazy_vacuum()`。
之后 vacuum FSM。
再把 `heap_blks_vacuumed` progress 更新到 rel_pages。
最后，如果有 indexes 且 `do_index_cleanup` 为 true，调用 `lazy_cleanup_all_indexes()`。
`lazy_cleanup_all_indexes()` 设置 progress：

```text
phase = cleaning up indexes
indexes_total = nindexes
indexes_processed = 0..nindexes
```

串行路径逐个调用 `lazy_cleanup_one_index()`。
它构造 `IndexVacuumInfo`。
这次 `estimated_count` 取决于 heap 是否完整扫描。
然后调用：

```text
vac_cleanup_one_index()
  -> index_vacuum_cleanup()
     -> indexRelation->rd_indam->amvacuumcleanup()
```

B-Tree 的 `btvacuumcleanup()` 有两个分支。
如果 `info->analyze_only`，直接 no-op。
如果 stats 为 NULL，说明没有发生 `btbulkdelete()`。
它会调用 `_bt_vacuum_needs_cleanup()` 判断是否需要 cleanup-only physical scan。
如果不需要，返回 NULL。
如果需要，分配 stats，调用 `btvacuumscan(info, stats, NULL, NULL, 0)`。
然后把 `stats->estimated_count = true`。
如果 stats 不为 NULL，说明之前发生过 bulkdelete。
B-Tree 通常不再扫描 index。
它维护 metapage cleanup info。
然后在可能准确的情况下，把 `num_index_tuples` capped 到 heap tuple count。
这解释了为什么 `amvacuumcleanup()` 不是简单的“最后打印 stats”。
它也是 AM 自己决定是否需要额外 cleanup scan 的入口。

## 6. 生命周期 / ownership / cleanup

### 6.1 谁创建

`LVRelState` 由 `heap_vacuum_rel()` 在当前 memory context 中 `palloc0`。
index relation 数组由 `vac_open_indexes()` 打开并返回。
`indstats` 数组由 `heap_vacuum_rel()` 分配。
`dead_items` 由 `dead_items_alloc()` 创建。
串行 VACUUM 创建 local `TidStore`。
parallel vacuum 创建 shared `TidStore` 并放进 `ParallelVacuumState`。
`IndexVacuumInfo` 是每次调用 index AM 前栈上构造的临时对象。
`IndexBulkDeleteResult` 通常由 index AM 第一次 `ambulkdelete()` 分配。
B-Tree 在 `btbulkdelete()` 中分配。
`BTVacState` 是 `btvacuumscan()` 的栈上对象。
`pagedelcontext` 是 B-Tree 每次 vacuum scan 的临时 MemoryContext。

### 6.2 谁持有

heap VACUUM leader 持有 `LVRelState`。
`dead_items` 的逻辑 owner 是 heap VACUUM。
index AM 只在 `ambulkdelete()` 调用期间通过 callback state 借用它。
AM 不应该保存 callback state 到调用结束后。
`IndexBulkDeleteResult` 的 owner 跨层。
AM 分配和更新它。
heap VACUUM 把指针保存在 `vacrel->indstats[idx]`。
final cleanup 后，heap VACUUM 用它更新 pg_class index stats。
最后 `heap_vacuum_rel()` 释放这些 stats。
parallel vacuum 中，每个 index 的 stats 先在 DSM 的 `PVIndStats` 中保存。
`parallel_vacuum_end()` 把 DSM stats 拷回 leader 本地 `indstats`。

### 6.3 谁释放

串行 local `TidStore` 没有在 `dead_items_cleanup()` 显式 destroy。
源码注释写的是“不 bother with pfree here”。
它依赖 VACUUM 所在 memory context 的生命周期。
但 `dead_items_reset()` 会在每轮处理后 destroy 并重建 local `TidStore`。
parallel case 必须显式清理。
`parallel_vacuum_end()` 会 `TidStoreDestroy(pvs->dead_items)`、销毁 ParallelContext，并 `ExitParallelMode()`。
B-Tree 的 `pagedelcontext` 在 `btvacuumscan()` 末尾 `MemoryContextDelete()`。
posting list update 临时内存在 `_bt_delitems_vacuum()` 和 `btvacuumpage()` 中释放。

### 6.4 ERROR / abort 时谁兜底

普通 palloc 内存由 MemoryContext reset 兜底。
buffer pin、buffer lock、relation lock 等外部资源由 ResourceOwner 和错误展开路径释放。
B-Tree vacuum cycle 不能只靠 MemoryContext。
`btbulkdelete()` 用 `PG_ENSURE_ERROR_CLEANUP(_bt_end_vacuum_callback, rel)`。
如果 `btvacuumscan()` 中 ERROR，callback 会结束 B-Tree vacuum cycle。
heap VACUUM 的 error context callback 不释放资源。
它只负责报告当前位置：
扫描 heap block。
vacuum heap block。
vacuum index name。
index cleanup name。
truncate target。
这对定位很有用。
资源清理仍由 MemoryContext、ResourceOwner、AM 自己的 cleanup callback 和 parallel context 负责。

### 6.5 长期状态如何失效

`dead_items` 只在一轮 index vacuum 和 heap cleanup 之间有效。
`dead_items_reset()` 后，旧 set 语义消失。
`IndexBulkDeleteResult` 只在一个 VACUUM 命令内有效。
它不是 catalog truth。
`pg_class.relpages/reltuples` 也只有在 stats 准确时才更新。
B-Tree deleted page 的 `safexid` 是持久 page 状态的一部分。
它要跨 VACUUM 存活，直到未来 horizon 允许 page 进入 FSM。
`btm_last_cleanup_num_delpages` 位于 B-Tree metapage。
它影响下次 `btvacuumcleanup()` 是否需要 cleanup-only scan。

## 7. 正确性机制层次

### 7.1 MVCC visibility 层

heap VACUUM 在 phase I 使用 `VacuumCutoffs` 和 `GlobalVisState`。
`lazy_scan_prune()` 通过 `heap_page_prune_and_freeze()` 判断 tuple 是否可以变成 `LP_DEAD`。
`lazy_scan_noprune()` 用 `HeapTupleSatisfiesVacuum()` 统计 live、recently dead、missed dead。
`GlobalVisState` 来自 `GlobalVisTestFor(rel)`。
它根据 relation 类型选择 shared/catalog/data/temp horizon。
这层回答：
哪些 heap tuple version 可以被认为对所有相关 snapshot 不再需要。
index AM 不参与这个判断。

### 7.2 dead TID membership 层

callback 层只回答 membership：
这个 heap TID 是否在本轮 VACUUM 的 `TidStore` 中？
这层不重新看 clog。
不访问 heap tuple header。
不判断 commit/abort。
它的正确性依赖 phase I 收集 set 时已经做完 visibility 裁决。
这让 index AM 可以保持物理结构 owner 的边界。

### 7.3 heap TID recycling 层

heap `LP_DEAD -> LP_UNUSED` 只能在所有 index AM 对本轮 dead set 完成删除后发生。
`lazy_vacuum_all_indexes()` 成功后才调用 `lazy_vacuum_heap_rel()`。
源码断言也表达了这个顺序。
如果 failsafe 使所有 index 没有完整处理，heap cleanup 不会发生。
这层保证旧 index entry 不会指向可复用 line pointer。

### 7.4 lock / pin 层

heap phase I pruning 需要 buffer cleanup lock。
拿不到时，非 aggressive VACUUM 可以降级走 `lazy_scan_noprune()`。
heap phase III 只需要 exclusive lock 来把已知 `LP_DEAD` 设为 `LP_UNUSED`。
B-Tree bulkdelete 对每个 leaf page 升级为 cleanup lock。
这让它等待仍持有 page pin 的 index scans。
这个 interlock 不是为了 B-Tree search 不迷路。
而是为了避免 index scan 从 leaf page 读到 TID 后，heap slot 被 VACUUM 复用。

### 7.5 WAL / critical section 层

heap pruning/freeze 和 heap vacuum cleanup 都通过 `log_heap_prune_and_freeze()` 记录 WAL。
VM set 要和 heap page 修改在同一 critical section 中协调。
B-Tree `_bt_delitems_vacuum()` 在 critical section 中修改 page、写 `XLOG_BTREE_VACUUM`、设置 page LSN。
B-Tree page deletion 写 half-dead/unlink WAL。
每一步都遵守 WAL-before-data。
错误不能在 critical section 中发生。
所以可分配内存、可做 IO 的检查要在进入 critical section 前完成。

### 7.6 并发 split / page deletion 层

B-Tree VACUUM 按物理 block order 扫描。
并发 split 可能把尚未扫过 page 的 tuple 移到已经扫过的低 block page。
vacuum cycle id 和 backtracking 处理这个问题。
empty page deletion 又有 half-dead、deleted、safexid、pending FSM 这些状态。
这些是 B-Tree AM 内部正确性。
heap VACUUM 不需要知道细节。
但它要允许 AM 在 `ambulkdelete()` 内部做这些工作。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 `INDEX_CLEANUP OFF`

用户或 reloption 可以关闭 index cleanup。
`heap_vacuum_rel()` 会设置：

```text
do_index_vacuuming = false
do_index_cleanup = false
```

之后如果 phase I 收集到 dead items，`lazy_vacuum()` 会 reset `dead_items` 并返回。
不会调用 index AM。
也不会 phase III 回 heap 释放 `LP_DEAD`。
结果是：
heap 和 index 中的 dead state 会保留到未来 VACUUM。
这可以让紧急 VACUUM 更快完成。
代价是 index bloat 和 heap dead line pointer 累积。
文档也提醒，如果不定期做 index cleanup，性能会下降。

### 8.2 bypass optimization

默认 `INDEX_CLEANUP AUTO` 允许 bypass。
当 dead TID 很少，并且预计本次 VACUUM 只有一轮 index scan 时，`lazy_vacuum()` 可以跳过 index vacuum 和 heap vacuum。
条件包括：
有 `LP_DEAD` 的 heap page 数小于 relation page 数约 2%。
`TidStore` 使用内存小于 32MB。
这是为了避免 tiny amount of dead index entries 带来整索引扫描成本。
它尤其服务几乎都是 HOT update 的 workload。
代价是少量 `LP_DEAD` 留在 heap 和 index 中。
这不是 correctness 问题。
它是延后清理。

### 8.3 wraparound failsafe

failsafe 是最后防线。
`vacuum_xid_failsafe_check()` 根据 `relfrozenxid/relminmxid` 与 failsafe age 判断。
触发后，`lazy_check_wraparound_failsafe()` 会：
设置 `VacuumFailsafeActive`。
丢弃 buffer access strategy，使 VACUUM 可使用全部 shared buffers。
关闭 index vacuuming。
关闭 index cleanup。
关闭 heap truncation。
停止 cost-based delay。
更新 progress mode 为 `failsafe`。
发 WARNING。
这说明 PostgreSQL 在 wraparound 风险下愿意牺牲非必要维护。
index vacuum 是非必要维护。
推进 freeze horizon 才是必要维护。
如果 failsafe 在 index round 中途触发，`lazy_vacuum_all_indexes()` 返回 false。
`lazy_vacuum()` 不会调用 `lazy_vacuum_heap_rel()`。
这避免释放 heap `LP_DEAD`，因为不是所有 index 都完成了删除。

### 8.4 cleanup lock contention

heap phase I 拿不到 cleanup lock 时，普通 VACUUM 可以走 `lazy_scan_noprune()`。
它仍能收集已有 `LP_DEAD`。
但不能产生新的 `LP_DEAD`。
如果页面上有 `HEAPTUPLE_DEAD`，会计入 missed dead。
aggressive VACUUM 如果需要 freeze，则不能简单跳过。
它可能等待 cleanup lock。
B-Tree bulkdelete 必须拿 leaf cleanup lock。
如果有 index scan 持有 leaf page pin，VACUUM 会等。
这可能在 `pg_stat_activity.wait_event` 中表现为 buffer pin 等待。

### 8.5 `amvacuumcleanup()` cleanup-only skip

如果没有 dead TID，`ambulkdelete()` 不会被调用。
但 final cleanup 仍会调用 `amvacuumcleanup()`。
B-Tree 这时先调用 `_bt_vacuum_needs_cleanup()`。
如果 metapage 显示没有必要 cleanup，直接返回 NULL。
如果需要，它会做 cleanup-only `btvacuumscan()`。
这时 callback 为 NULL。
它不删除 leaf TID，只统计和回收 old deleted pages。

### 8.6 pending FSM optimization 失败

B-Tree newly deleted page 可能在当前 vacuum scan 末尾就能进入 FSM。
`_bt_pendingfsm_finalize()` 尝试根据 `safexid` 和 global visibility 判断。
如果 horizon 还不允许，停止。
如果 pendingpages 数组达到 `work_mem` 上限，后续 newly deleted page 不记录进数组。
这不破坏 correctness。
它只放弃“本轮提前放入 FSM”的优化。
未来 VACUUM 扫到 deleted page 时仍可回收。

### 8.7 中断和 crash

B-Tree page deletion 可能留下 half-dead page。
下一次 VACUUM 会继续删除。
WAL redo 能重放每一步。
heap phase III 如果 crash，在 redo 后保持一致。
如果 index 已删除但 heap `LP_DEAD` 还没变 `LP_UNUSED`，只是空间回收延后。
反过来会让 index entry 指向可复用 heap slot。

## 9. 成本、资源与跨模块传播

### 9.1 成本主模型

index vacuum 成本近似为：

```text
heap phase I scanned pages
+ num_index_scans * sum(index pages scanned by AM)
+ phase III touched heap pages from TidStore
+ final cleanup AM cost
```

`num_index_scans` 是放大器。
它由 dead TID storage 是否足够决定。
`maintenance_work_mem` 或 `autovacuum_work_mem` 太小，会让同一个 heap relation 做多轮全索引扫描。
多一个 index，就多一次每轮 AM scan。
多一个 round，就把所有 index 的 bulkdelete 成本再乘一遍。

### 9.2 callback CPU 成本

B-Tree 对每个 leaf TID 调 callback。
普通 index tuple 是一次 callback。
posting list tuple 是 posting list 中每个 TID 一次 callback。
callback 本身是 `TidStoreIsMember()`。
它通常是 radix tree lookup 加 offset bitmap check。
这比访问 heap tuple header 便宜得多。
但对很大的 index，次数仍然等于 index AM 扫到的 heap TID 数。

### 9.3 IO 与 WAL 成本

heap phase I 读 heap pages，可能写 heap page 和 VM。
index phase 读 index pages，删除 leaf items 会写 index page WAL。
B-Tree posting list update 也写 WAL。
B-Tree empty page deletion 可能写多个 pages 的 WAL。
heap phase III 写 heap page，把 `LP_DEAD` 改 `LP_UNUSED`，可能同时设置 VM。
`VACUUM VERBOSE` 的 WAL usage 可以看到总量。
但它不能按 phase 拆分每条 WAL 来源。
需要 `pg_waldump`、断点或 perf 才能更细。

### 9.4 buffer pin / cleanup lock contention

heap cleanup lock contention 会减少 pruning 产出，增加 missed dead。
B-Tree cleanup lock 等待会拖慢 index vacuum。
长时间 index-only scan 或 cursor 可能持有 leaf page pin。
B-Tree README 说明 index-only scan 不能提前丢 pin，因为它不能可靠容忍 referenced TID 被复用。
这会把查询模式传播到 VACUUM latency。

### 9.5 parallel vacuum 的资源传播

parallel vacuum 并行的是 index bulkdelete/cleanup。
heap scan 和 dead TID 收集仍在 leader。
shared `TidStore` 位于 DSM/DSA。
每个 worker 处理一个或多个 index。
worker 把 `IndexBulkDeleteResult` 写回 DSM 的 `PVIndStats`。
leader 在 `parallel_vacuum_end()` 拷回本地。
并行只在有多个 index 时有意义。
`dead_items_alloc()` 注释也说明当前只有至少两个 indexes 才考虑 parallel vacuum。
temporary table 不能 parallel vacuum。
parallel worker 对 VACUUM delay 的处理通过 shared cost balance 协调。

### 9.6 后台进程和相邻模块

autovacuum worker 可以触发同一条链路。
它使用 `autovacuum_work_mem`，如果该值为 -1 则回退 `maintenance_work_mem`。
checkpointer 和 walwriter 不推进 dead TID 状态，但它们承接 VACUUM 产生的 dirty buffers 和 WAL flush 压力。
startup process 在 recovery 中重放 heap/B-Tree WAL。
standby query 通过 horizon 和 conflict 影响哪些 XID 仍被认为需要保留。
logical replication slot、long transaction 和 old snapshot 会影响 global visibility horizon。
这些因素不是 index callback 的字段。
但它们会改变 phase I 收集多少 dead TID，以及 B-Tree deleted pages 何时可进入 FSM。

## 10. 观测与诊断入口

### 10.1 `pg_stat_progress_vacuum`

当前基线的 progress view 在 `system_views.sql:1320-1340`。
关键查询：

```sql
SELECT pid,
       phase,
       heap_blks_total,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       max_dead_tuple_bytes,
       dead_tuple_bytes,
       num_dead_item_ids,
       indexes_total,
       indexes_processed,
       mode
FROM pg_stat_progress_vacuum;
```

`phase = scanning heap` 时，`num_dead_item_ids` 会随着 `dead_items_add()` 增长。
如果 `dead_tuple_bytes` 接近或超过 `max_dead_tuple_bytes`，源码会触发一轮 `lazy_vacuum()`。
`phase = vacuuming indexes` 时，`indexes_processed` 推进。
`index_vacuum_count` 是已启动的 index vacuum round 数。
`phase = vacuuming heap` 时，heap 正在把 `LP_DEAD` 释放为 `LP_UNUSED`。
`mode = failsafe` 表示已经绕过非必要维护。
注意列名 `max_dead_tuple_bytes/dead_tuple_bytes`。
在当前源码里，它们对应 dead item/TID storage。
不要按“heap tuple bytes”理解。

### 10.2 `VACUUM VERBOSE` 和 autovacuum log

`heap_vacuum_rel()` 的 verbose/log 输出会包括：
index scans 数。
pages scanned。
tuples removed/remain/recently dead。
tuples missed。
index scan needed / not needed / bypassed / bypassed by failsafe。
dead item identifiers 数。
parallel workers。
每个 index 的 pages、newly deleted、currently deleted、reusable。
dead item storage memory usage 和 reset 次数。
buffer usage。
WAL usage。
system usage。
诊断时先看三类字段：
第一，`index scans` 是否大于 1。
这通常指向 dead TID storage 不够，或者 churn 太大。
第二，`index scan bypassed` 还是 `index scan needed`。
这区分 bypass optimization 和真实 bulkdelete。
第三，`tuples missed` 是否明显。
这指向 heap cleanup lock contention。

### 10.3 `pg_stat_all_tables` 与 `pg_stat_all_indexes`

`pg_stat_all_tables.n_dead_tup` 是统计估算。
它不是 `TidStore` 当前大小。
它也不是 heap page 上 `LP_DEAD` 的精确数量。
`pg_stat_all_indexes` 不能告诉你某个 index 里还有多少 dead index entry。
index bloat 需要结合 size、workload、VACUUM 日志、pageinspect、amcheck 或离线工具判断。
不要把 stats view 当成 callback 内部状态。

### 10.4 pageinspect / amcheck / pg_visibility

`pageinspect` 可以看 B-Tree page items、posting list 和 page metadata。
它适合实验中观察 duplicate key、posting list 和 page 状态。
heap `LP_DEAD/LP_UNUSED` 也可以通过 pageinspect 函数观察。
`pg_visibility` 可以看 VM all-visible/all-frozen。
它不能告诉你 callback 判定过程。
`amcheck` 可以验证 B-Tree 结构一致性。
它不是 dead TID 计数器。

### 10.5 wait event 和 profiling

VACUUM cost delay 可见为 `VacuumDelay`。
cleanup lock 等待可能表现为 buffer pin 相关等待。
IO 压力可通过 `pg_stat_io`、`pg_stat_wal`、系统 IO 工具观察。
callback CPU 成本通常需要 perf、flamegraph 或 gdb。
可打断点：

```text
vac_tid_reaped
TidStoreIsMember
lazy_vacuum_one_index
btvacuumpage
btreevacuumposting
_bt_delitems_vacuum
lazy_vacuum_heap_page
```

断点观察要注意：
`vac_tid_reaped()` 调用次数可能非常大。
对大 index 直接断每次调用会严重改变时序。
更好的方式是加计数器或条件断点。

## 11. 常见误区

误区一：
`LP_DEAD` 就等于 heap space 已经回收。
不对。
有 index 的表中，`LP_DEAD` 仍然保留 offset number。
只有 phase III 的 `ItemIdSetUnused()` 后，slot 才能复用。
误区二：
callback 在判断 tuple visibility。
不对。
callback 只是 `TidStoreIsMember()`。
visibility 判断发生在 heap phase I。
误区三：
index AM 可以只扫描 dead TID 对应的 index entry。
通常不行。
AM 不知道某个 heap TID 在自己物理结构里的位置。
B-Tree VACUUM 通过全索引扫描和 callback 找到匹配 entry。
误区四：
`maintenance_work_mem` 是 `TidStore` 的硬上限。
不准确。
`TidStoreCreateLocal()` 把 max_bytes 用作 memory context block size hint。
VACUUM 用 `TidStoreMemoryUsage()` 超限检查触发一轮处理。
因此可能略超后才进入 index vacuum。
误区五：
`amvacuumcleanup()` 只有在 `ambulkdelete()` 后才有意义。
不对。
AM 必须处理 stats 为 NULL 的情况。
B-Tree 可能做 cleanup-only scan，也可能快速返回 NULL。
误区六：
bypass optimization 表示没有 dead tuple。
不对。
它表示 dead TID 很少，整索引扫描成本可能不划算。
少量 `LP_DEAD` 可能被延后处理。
误区七：
parallel vacuum 会并行 heap scan。
当前主路径不是这样。
heap scan 和 `TidStore` 收集在 leader。
parallel worker 主要处理 index vacuum/cleanup。
误区八：
B-Tree page 进入 FSM 是 index tuple 删除的直接结果。
不对。
empty leaf page 删除后还要等 `safexid` 对所有相关 snapshot 安全。
pending FSM 只是优化。

## 12. 课堂实验

### 实验 1：观察 dead TID storage 触发多轮 index vacuum

目标：
把 runtime 现象和 `TidStoreMemoryUsage() > max_bytes -> lazy_vacuum()` 对上。
步骤：

```sql
CREATE TABLE vac_cb_t (id bigserial PRIMARY KEY, k int, v text);
CREATE INDEX vac_cb_t_k_idx ON vac_cb_t(k);
CREATE INDEX vac_cb_t_v_idx ON vac_cb_t(v);

INSERT INTO vac_cb_t(k, v)
SELECT g % 1000, repeat('x', 100)
FROM generate_series(1, 800000) AS g;

UPDATE vac_cb_t
SET v = repeat('y', 100)
WHERE id % 2 = 0;

SET maintenance_work_mem = '1MB';
VACUUM (VERBOSE, INDEX_CLEANUP ON) vac_cb_t;
```

另一个 session 观察：

```sql
SELECT phase,
       heap_blks_scanned,
       heap_blks_vacuumed,
       index_vacuum_count,
       dead_tuple_bytes,
       max_dead_tuple_bytes,
       num_dead_item_ids,
       indexes_total,
       indexes_processed
FROM pg_stat_progress_vacuum
WHERE relid = 'vac_cb_t'::regclass;
```

预期观察：
`index_vacuum_count` 可能大于 1。
`VACUUM VERBOSE` 的 `index scans` 可能大于 1。
dead item storage memory usage 会显示 reset 次数。
源码回扣：
`lazy_scan_heap()` 超过 `max_bytes` 后调用 `lazy_vacuum()`。
每轮 `lazy_vacuum_all_indexes()` 都处理所有 indexes。

### 实验 2：比较 `INDEX_CLEANUP AUTO` 与 `ON`

目标：
观察 bypass optimization。
步骤：

```sql
CREATE TABLE vac_bypass_t (id bigserial PRIMARY KEY, k int, pad text);
CREATE INDEX vac_bypass_t_k_idx ON vac_bypass_t(k);

INSERT INTO vac_bypass_t(k, pad)
SELECT g, repeat('a', 100)
FROM generate_series(1, 200000) AS g;

DELETE FROM vac_bypass_t WHERE id BETWEEN 1 AND 10;

VACUUM (VERBOSE, INDEX_CLEANUP AUTO) vac_bypass_t;

DELETE FROM vac_bypass_t WHERE id BETWEEN 11 AND 20;

VACUUM (VERBOSE, INDEX_CLEANUP ON) vac_bypass_t;
```

预期观察：
AUTO 路径可能显示 index scan bypassed。
ON 路径会强制 index vacuum，只要有 dead TID 且 failsafe 未触发。
这个实验受 page 分布和版本行为影响。
如果没有触发 bypass，可加大表、减少删除行数，或重复观察 `lpdead_item_pages / rel_pages` 的比例。
源码回扣：
`lazy_vacuum()` 用 `BYPASS_THRESHOLD_PAGES` 和 32MB dead item storage 条件判断。
`INDEX_CLEANUP ON` 会让 `consider_bypass_optimization = false`。

### 实验 3：gdb 观察 callback ownership

目标：
证明 index AM 调的是 membership callback，而不是 heap visibility 判断。
步骤：
在测试环境启动 PostgreSQL。
对执行 `VACUUM` 的 backend 附加 gdb。
设置断点：

```text
break vac_tid_reaped
break TidStoreIsMember
break btvacuumpage
break _bt_delitems_vacuum
break lazy_vacuum_heap_page
```

执行能产生 dead index entry 的 UPDATE/DELETE 后：

```sql
VACUUM (VERBOSE, INDEX_CLEANUP ON) target_table;
```

观察顺序：
先进入 `btvacuumpage()`。
再多次进入 `vac_tid_reaped()` 和 `TidStoreIsMember()`。
如果 page 有删除，进入 `_bt_delitems_vacuum()`。
所有 index round 成功后，才进入 `lazy_vacuum_heap_page()`。
源码回扣：
这条断点顺序就是本节核心 ownership：
index 删除引用在前。
heap 释放 line pointer 在后。

## 13. 讨论题

1. 为什么 callback 的输入是 heap TID，而不是 index tuple pointer？
2. 如果 `TidStore` 内存不足导致三轮 index vacuum，`IndexBulkDeleteResult` 的哪些字段应该跨轮累计，哪些字段应该每轮重算？
3. 为什么 B-Tree VACUUM 要在没有 deletable tuple 的 leaf page 上也拿 cleanup lock？
4. `INDEX_CLEANUP OFF` 为什么不会破坏 correctness，但会让后续性能变差？
5. `amvacuumcleanup()` 收到 NULL stats 时，为什么不能简单断言“没有工作可做”？
6. `pg_stat_progress_vacuum.dead_tuple_bytes` 能说明什么，不能说明什么？
7. 如果 failsafe 在处理第三个 index 前触发，为什么 heap phase III 不能释放本轮 `LP_DEAD`？
8. B-Tree newly deleted page 为什么还要等 `GlobalVisCheckRemovableFullXid()` 才能进入 FSM？

## 14. 本节小结

本节唯一主问题是：
heap 已经知道 dead TID，如何让各 index AM 删除对应 index entry，并在所有 index 删除完成前阻止 heap TID 复用。
核心链路是：

```text
lazy_scan_heap()
  -> lazy_scan_prune()/lazy_scan_noprune()
  -> dead_items_add()
  -> lazy_vacuum()
  -> lazy_vacuum_all_indexes()
  -> lazy_vacuum_one_index()
  -> vac_bulkdel_one_index()
  -> index_bulk_delete()
  -> AM ambulkdelete(callback)
  -> lazy_vacuum_heap_rel()
  -> lazy_vacuum_heap_page()
  -> lazy_cleanup_all_indexes()
```

核心状态是：
`LVRelState.dead_items` 保存本轮 dead TID set。
`VacDeadItemsInfo` 保存 set 的内存上限和当前 item 数。
`IndexVacuumInfo` 把 relation、估算 tuple 数和 access strategy 传给 AM。
`IndexBulkDeleteResult` 在同一 VACUUM 中跨多次 `ambulkdelete()` 和 `amvacuumcleanup()` 传递 stats。
`IndexBulkDeleteCallback` 是 heap dead set 与 index AM 物理扫描之间的边界。
ownership 顺序是：
heap 发现 `LP_DEAD`。
index AM 删除 index entry。
heap 释放 `LP_DEAD` 为 `LP_UNUSED`。
cleanup 更新 AM 统计和可回收 index page 状态。
正确性不是一个机制保证的。
MVCC horizon 决定 heap TID 是否进入 dead set。
callback 保证 AM 只删除本轮 set 中的 TID。
cleanup lock/pin interlock 防止 heap TID recycling 与 index scan 交错。
WAL 和 critical section 保证 crash recovery 重放同样的 page mutation。
B-Tree cycle id/backtracking 保证线性 index scan 不漏掉并发 split 后的 tuple。
错误和 fallback 路径同样重要。
`INDEX_CLEANUP OFF` 和 bypass 会延后清理。
failsafe 会牺牲 index vacuum/cleanup/truncation 来优先避免 wraparound。
cleanup lock contention 会让 heap pruning 降级。
B-Tree pending FSM 失败只会延后 page reuse。
观测上，`pg_stat_progress_vacuum` 能看到 phase、dead item storage、index round 和 failsafe mode。
`VACUUM VERBOSE` 能看到 index scans、bypass、memory reset、per-index pages 和 WAL usage。
它们都看不到每一次 callback 判断。
callback 粒度要靠 gdb、perf、条件断点或临时 instrumentation。
可迁移的系统规律是：
当一个模块拥有语义判断，另一个模块拥有物理结构时，不要把语义复制到物理 owner 里。
用一个窄 callback 传递已经裁决过的集合。
再用清晰的 lifecycle 保证物理引用删除完成后，原对象身份才被释放和复用。
本节结论仍有边界。
index vacuum 是否成为瓶颈，取决于 index 数、index size、dead TID 数、`maintenance_work_mem`、posting list 密度、cleanup lock 等待、IO 带宽、WAL 带宽、long transaction、replication slot 和版本实现。
这些因素不能只靠一个 pg_stat 指标完全归因。
