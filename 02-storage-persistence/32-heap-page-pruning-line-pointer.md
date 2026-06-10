# PostgreSQL Heap page pruning、redirect/dead line pointer 与空间回收
## 课程定位
本节主题：Heap page pruning、redirect/dead line pointer 与空间回收。
上一节已经建立 heap page layout、line pointer、tuple header 和 `t_ctid` 的基本模型。
本节只讨论一个主问题：
已经被 UPDATE/DELETE 留在 heap page 上的旧 tuple version，什么时候能被安全剪掉，并且为什么剪掉时不能破坏 index TID？
前置知识：
你应该已经理解 buffer pin/content lock、WAL-before-data、page LSN、MVCC snapshot、heap line pointer 和 HOT update 的基本含义。
本节围绕的核心矛盾：
heap page 需要尽早回收旧版本占用的空间。
index entry 又可能长期指向旧 root TID。
旧 snapshot、standby query、logical decoding 和 index vacuum 仍然要求 tuple removal 有明确的 visibility horizon。
同时 pruning 是一个页内维护动作，不能因为每个 UPDATE 都扫描所有 index 而变成昂贵的全局操作。
PostgreSQL 的解决方案不是“直接删除旧 tuple”。
它把事情拆成三层：
line pointer 状态保留 TID 入口。
tuple bytes 可被剪掉并通过 page compaction 回收。
index entry 的最终删除交给 VACUUM/index AM。
本节读完后，你应该能独立判断：
- `LP_REDIRECT`、`LP_DEAD`、`LP_UNUSED` 的安全边界。
- HOT chain 被 pruning 压缩时，哪些 offset number 必须保留。
- 为什么 root line pointer 不能随便复用。
- 为什么 heap-only tuple 的 line pointer 可以比 root 更早变 `LP_UNUSED`。
- `pd_prune_xid` 为什么只是 hint，不是正确性事实。
- `RECENTLY_DEAD` 和 `DEAD` 的差别为什么取决于 visibility horizon。
- on-access pruning 为什么只能 opportunistic 地拿 cleanup lock。
- VACUUM 为什么要把 `LP_DEAD` 和 index deletion 分成两个阶段。
- WAL record 为什么需要记录 redirect/dead/unused offsets 和 snapshot conflict horizon。
- pageinspect、pg_visibility、pg_stat 和 WAL dump 分别能看到什么。
本节最重要的一句话运行模型：
```text
pruning 在 cleanup lock 下，用 visibility horizon 判断哪些 tuple version 可移除；它把 HOT chain 的稳定 index 入口保留为 root line pointer，把旧 tuple bytes 剪掉并 compact page，同时用 WAL 记录足够的信息让 redo 和 standby 冲突处理重放同一个边界。
```
## 源码基线
源码仓库：
```text
/home/nail/postgres-lab
```
源码基线：
```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```
本节重点源码：
```text
src/backend/access/heap/pruneheap.c
src/backend/access/heap/heapam.c
src/backend/access/heap/heapam_indexscan.c
src/backend/access/heap/heapam_visibility.c
src/backend/access/heap/heapam_xlog.c
src/backend/access/heap/vacuumlazy.c
src/backend/access/heap/README.HOT
src/include/storage/itemid.h
src/include/storage/bufpage.h
src/include/access/htup_details.h
src/include/access/heapam.h
src/include/access/heapam_xlog.h
```
阅读顺序不要按文件名排序。
先读 `itemid.h` 和 `README.HOT`，建立 root TID 不能丢的约束。
再读 `heap_update()`，看 HOT chain 怎么形成。
然后读 `heap_page_prune_opt()` 和 `heap_page_prune_and_freeze()`，看 pruning 怎么被触发。
接着读 `heap_prune_chain()` 和 `heap_page_prune_execute()`，看 line pointer 状态如何真实变化。
最后读 `heapam_visibility.c`、`vacuumlazy.c` 和 `heapam_xlog.c`，看 horizon、VACUUM 两阶段和 WAL redo。
行号来自当前基线源码的 `rg -n` / `nl -ba` 结果。
源码会继续演化，所以本课把函数名和状态边界当稳定知识，把具体行号当定位辅助。
## 1. 本节在总主线中的位置
前面几节讲的是 page 能不能安全读写和落盘。
这一节进入 page 内部的生命周期。
一个 heap page 不是只存当前行。
它还存旧版本、HOT chain、中间 line pointer 和可复用空间。
UPDATE 在 MVCC 下通常不是原地覆盖。
它写入新 tuple version。
旧 tuple header 记录删除/更新事务。
旧 tuple 的 `t_ctid` 指向新版本。
如果更新满足 HOT 条件，新版本不需要新的普通 index entry。
这会产生一个问题。
index entry 仍然指向旧 root TID。
旧 root tuple 本身可能已经对所有事务不可见。
但 root offset number 仍然是 index 进入 heap page 的入口。
如果 pruning 把 root line pointer 直接设成 `LP_UNUSED`，以后同一个 offset number 可能被新行复用。
旧 index entry 就会指向无关 tuple。
这就是 index corruption。
所以本节不是泛泛介绍 VACUUM。
本节主线是：
```text
HOT update 形成 chain
  -> 旧版本变得 globally removable
  -> pruning 压缩 chain
  -> line pointer 保留或释放
  -> tuple bytes compact
  -> WAL/redo/standby 保持同样边界
  -> VACUUM 最终删除 index entry 并释放 LP_DEAD
```
这条链路里每个状态都有明确边界。
`LP_REDIRECT` 是保留 index 入口并跳到 heap-only successor。
`LP_DEAD` 是保留 index TID 的墓碑。
`LP_UNUSED` 是可以被新 tuple 立即复用的空 slot。
三者不能互换。
`LP_REDIRECT` 和 `LP_DEAD` 都不代表 tuple bytes 还在。
`LP_UNUSED` 才代表 offset number 可复用。
本节的系统 tension 可以压缩为：
```text
page-local reclaim wants to erase bytes now; index stability and MVCC horizon require a durable, replayable, delayed ownership transfer for TID slots.
```
## 2. 核心文件分工与阅读顺序
`src/include/storage/itemid.h` 定义 `ItemIdData`。
这里能看到 `lp_off`、`lp_flags`、`lp_len` 和四个 `LP_*` 状态。
`LP_UNUSED` 表示 immediate re-use。
`LP_NORMAL` 表示有 tuple storage。
`LP_REDIRECT` 表示 HOT redirect，`lp_off` 不再是 byte offset，而是目标 offset number。
`LP_DEAD` 表示 dead stub，可能没有 storage，但仍不能随便复用。
`src/include/storage/bufpage.h` 定义 `PageHeaderData`。
这里能看到 `pd_lower`、`pd_upper`、`pd_special`、`pd_prune_xid` 和 `pd_flags`。
`pd_prune_xid` 是 oldest potentially prunable XID hint。
`PD_HAS_FREE_LINES` 和 `PD_PAGE_FULL` 也是 hint。
hint 可以帮助触发 pruning，但不能作为 correctness 事实。
`src/backend/storage/page/bufpage.c` 实现 page 空间整理。
`PageRepairFragmentation()` 是 pruning 后把 tuple bytes compact 到页尾的关键函数。
`PageTruncateLinePointerArray()` 是 VACUUM 第二阶段清理尾部 `LP_UNUSED` line pointer 的轻量路径。
`PageGetHeapFreeSpace()` 会考虑 `MaxHeapTuplesPerPage` 和 free line pointer hint。
`src/include/access/htup_details.h` 定义 tuple header 和 HOT flags。
`HEAP_HOT_UPDATED` 表示旧 tuple 的 successor 是同页 heap-only tuple。
`HEAP_ONLY_TUPLE` 表示这个 tuple 没有自己的普通 index entry。
`t_ctid` 是 update chain 线索，但追链时必须校验 successor 的 `xmin` 等于 predecessor 的 update xid。
`src/backend/access/heap/README.HOT` 是理解本节设计约束的最好入口。
它解释了为什么 page-at-a-time vacuum 不能重新计算任意 index key。
它也解释了 redirecting line pointer 和 dead line pointer 为什么存在。
`src/backend/access/heap/heapam.c` 里看 UPDATE/DELETE 如何制造可 prune 状态。
`heap_update()` 负责判断 HOT-safe update。
同页且 HOT-blocking index 列没变时，旧 tuple 设 `HEAP_HOT_UPDATED`，新 tuple 设 `HEAP_ONLY_TUPLE`。
旧 tuple 的 `t_ctid` 指向新 tuple 的 `t_self`。
`heap_delete()` 把 tuple 标成删除，并清掉 HOT forward link。
两者都会用 `PageSetPrunable(page, xid)` 留下后续 pruning hint。
`src/backend/access/heap/heapam_indexscan.c` 里看 index scan 如何消费 HOT chain。
`heapam_index_fetch_tuple()` 在 pin heap page 后会先尝试 `heap_page_prune_opt()`。
然后拿 share lock 调 `heap_hot_search_buffer()`。
`heap_hot_search_buffer()` 只允许在 chain start 遇到 `LP_REDIRECT`。
它会跟随 redirect 或 `t_ctid`，并检查 `xmin == prior xmax`。
`src/backend/access/heap/pruneheap.c` 是本节主文件。
`heap_page_prune_opt()` 是普通访问路径上的 opportunistic pruning。
`heap_page_prune_and_freeze()` 是共享主流程。
`prune_freeze_plan()` 先扫描并规划。
`heap_prune_chain()` 决定 HOT chain 每个成员命运。
`heap_page_prune_execute()` 在 critical section 内改 line pointer 并 compact page。
`log_heap_prune_and_freeze()` 写 `XLOG_HEAP2_PRUNE*` WAL。
`src/backend/access/heap/heapam_visibility.c` 解释 horizon。
`HeapTupleSatisfiesVacuumHorizon()` 返回 `LIVE`、`RECENTLY_DEAD`、`DEAD` 等结果。
`heap_prune_satisfies_vacuum()` 再把 `RECENTLY_DEAD` 和 `GlobalVisState` / VACUUM cutoff 比较。
`src/backend/access/heap/vacuumlazy.c` 解释 VACUUM 的参与方式。
`lazy_scan_prune()` 调 `heap_page_prune_and_freeze()`。
有 index 时，它收集 `LP_DEAD` TID 给 index vacuum。
无 index 时，它可以传 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`，直接把 would-be dead item 设成 `LP_UNUSED`。
`lazy_vacuum_heap_page()` 在 index tuple 删除后，把已记录的 `LP_DEAD` 设成 `LP_UNUSED`。
`src/backend/access/heap/heapam_xlog.c` 解释 redo。
`heap_xlog_prune_freeze()` 会按 WAL 里的 flags 决定是否拿 cleanup lock。
它重放 line pointer state change、freeze 和 VM change。
hot standby 下还会按 snapshot conflict horizon 处理恢复冲突。
## 3. 核心状态：line pointer 不是 tuple
`ItemIdData` 是 heap page 内最关键的间接层。
索引里的 heap TID 是 `(block, offset)`。
这个 offset 是 line pointer number。
它不是 tuple bytes 的页内偏移。
tuple bytes 可以在 `PageRepairFragmentation()` 中移动。
只要 line pointer number 仍然指向正确含义，index TID 就没变。
这就是 slotted page 支持 compaction 的根本原因。
`itemid.h` 把 `ItemIdData` 压到 32 bits：
```text
lp_off    15 bits
lp_flags   2 bits
lp_len    15 bits
```
这个布局也解释了为什么 PostgreSQL page size 上限和 line pointer 有关系。
`lp_off` 在 `LP_NORMAL` 下是 tuple bytes 起点。
`lp_len` 在 `LP_NORMAL` 下是 tuple bytes 长度。
`lp_flags` 决定如何解释其它字段。
raw field 不是语义。
`lp_off + lp_flags + lp_len + HOT context` 才是语义。
四个状态的边界如下：
| 状态 | 是否有 tuple storage | 是否可被新 tuple 复用 | index 还能否安全指向它 | pruning 语义 |
| --- | --- | --- | --- | --- |
| `LP_UNUSED` | 否 | 是 | 否 | 空 slot，可复用 |
| `LP_NORMAL` | 是 | 否 | 可能 | 正常 tuple 或 HOT chain member |
| `LP_REDIRECT` | 否 | 否 | 是 | HOT root 被剪掉后，保留 root TID 并跳到 successor |
| `LP_DEAD` | 否或曾有 | 否 | 是，但表示 dead | index entry 可能还没删，VACUUM 后才能释放 |
`LP_UNUSED` 是唯一能立即复用的状态。
这句话非常重要。
`LP_REDIRECT` 没有 tuple bytes，但不能复用。
因为 index entry 可能仍然指向这个 offset。
`LP_DEAD` 没有 tuple bytes，但不能复用。
因为 index entry 可能仍然指向这个 offset，VACUUM 还需要用它找到要删的 index tuple。
`LP_NORMAL` 通常有 tuple storage。
但 tuple 是否 live 要看 tuple header 和 visibility。
`LP_NORMAL` 不等于“对当前 snapshot 可见”。
`LP_NORMAL` 只说明 line pointer 指向一段 tuple bytes。
`LP_REDIRECT` 的特殊性在 `ItemIdGetRedirect()`。
它把 `lp_off` 当 offset number 用。
同一个字段在不同 `lp_flags` 下含义不同。
因此不能写调试脚本时把 `lp_off` 一律解释成 byte offset。
`LP_DEAD` 的特殊性在于它是 index cleanup 的协议状态。
它告诉 VACUUM：
这个 offset 对应的 heap tuple 已经不需要被任何事务看到。
但 index 里可能还有指向它的 TID。
所以 index vacuum 要删除这些 index TID。
只有删除完 index entry，heap 才能把 `LP_DEAD` 改成 `LP_UNUSED`。
`LP_UNUSED` 是最终空间复用状态。
新插入 tuple 可以重用这个 offset number。
如果有旧 index entry 还指向它，就会造成旧 index entry 命中新行。
这就是为什么 `LP_UNUSED` 需要最严格的前置条件。
## 4. HOT chain 如何形成
HOT 的核心目标是减少 index entry。
如果 UPDATE 不改变任何 HOT-blocking index 相关列，并且新 tuple 能放在同一个 heap page，就可以 HOT update。
普通 index 继续指向 root TID。
新版本作为 heap-only tuple 插在同页。
旧版本的 `t_ctid` 指向新版本。
`README.HOT` 给的基本图是：
```text
index -> lp 1
lp[1] normal tuple, HEAP_HOT_UPDATED, t_ctid -> lp[2]
lp[2] normal tuple, HEAP_ONLY_TUPLE
```
`heap_update()` 里对应的关键判断在 `heapam.c:3976` 附近。
如果 `newbuf == buffer`，说明新 tuple 放在旧页。
如果 `modified_attrs` 不和 `hot_attrs` 重叠，就设置 `use_hot_update = true`。
`hot_attrs` 来自 `RelationGetIndexAttrBitmap(... INDEX_ATTR_BITMAP_HOT_BLOCKING)`。
这不是“所有 index 列”的简单集合。
它表达的是会阻止 HOT 的 index 依赖列。
BRIN 这类 summarizing index 走另一条边界。
当 `use_hot_update` 为真时，`heap_update()` 做三件事：
旧 tuple 设 `HEAP_HOT_UPDATED`。
新 tuple 设 `HEAP_ONLY_TUPLE`。
旧 tuple 的 `t_ctid` 指向新 tuple 的 `t_self`。
这三件事必须一起理解。
旧 tuple 的 `HEAP_HOT_UPDATED` 允许 index scan 继续沿 `t_ctid` 找 successor。
新 tuple 的 `HEAP_ONLY_TUPLE` 表示它没有自己的普通 index entry。
旧 tuple 的 `t_ctid` 给出链路。
如果更新不是 HOT，新 tuple 会插入所有需要的 index。
旧 tuple 的 `HEAP_HOT_UPDATED` 会被清掉。
此时旧 tuple 的 `t_ctid` 仍可能指向新版本，但 index scan 不会把它当同一个 HOT chain 继续追。
HOT chain 限制在单页。
跨页 HOT 会让 page-local pruning 失去意义。
如果 successor 在别的 page，回收旧页空间还要考虑另一页状态。
这会把一个轻量页内维护动作变成跨页和跨 index 协议。
PostgreSQL 没有这么做。
HOT chain 的 root 是 index entry 指向的 offset。
root 可以是 `LP_NORMAL`。
root tuple bytes 被剪掉后，root 可以变成 `LP_REDIRECT`。
root 也可以在整条 chain 都 dead 后变成 `LP_DEAD`。
root 不能在 index entry 删除前变成 `LP_UNUSED`。
heap-only tuple 的 offset 没有普通 index entry 直接指向。
因此一旦它在 chain 中被证明可移除，pruning 可以把它设成 `LP_UNUSED`。
这就是 root 和 heap-only successor 在空间回收上的不对称。
## 5. index scan 如何读取被压缩的 HOT chain
理解 pruning 前，先看消费者。
如果 index scan 不能正确读取 redirect/dead 状态，pruning 就没有意义。
入口在 `heapam_indexscan.c`。
`heapam_index_fetch_tuple()` 先确认当前 heap block 是否已经 pin。
如果第一次进入该 block，它会 `ReadBuffer()` 并调用 `heap_page_prune_opt()`。
然后持 buffer share lock 调 `heap_hot_search_buffer()`。
`heap_hot_search_buffer()` 的输入 TID 来自 index AM。
它假设这个 TID 是简单 tuple 或 HOT chain root。
它会在同一个 buffer 内搜索第一个满足 snapshot 的 chain member。
如果遇到 `LP_REDIRECT`，只能在 chain start 接受。
也就是说，redirect 是 root entry 的状态。
中间出现 redirect 会被当作 chain 结束或异常边界处理。
如果遇到 `LP_UNUSED` 或 `LP_DEAD`，index scan 不会把它当 tuple。
这通常意味着 chain 已结束，或者 index entry 指向了已经 vacuumable 的 dead stub。
如果遇到 `LP_NORMAL`，它从 line pointer 拿 tuple bytes。
然后检查 chain start 不应该是 heap-only tuple。
如果 chain start 直接是 `HEAP_ONLY_TUPLE`，说明 index TID 指向了不该直接被 index 指向的 tuple。
这是潜在 corruption 边界。
追 `t_ctid` 时，它不仅看 offset。
它还检查 successor 的 `xmin` 是否等于 predecessor 的 update xid。
这个校验来自 `htup_details.h` 对 `t_ctid` 的说明。
原因是被 `t_ctid` 指向的 slot 可能已经被 VACUUM 回收并复用。
只看 offset 会把无关 tuple 当成后继版本。
这也是 HOT chain 读路径和 pruning 互相配合的关键。
pruning 可以释放 heap-only tuple 的 line pointer。
读路径必须准备好遇到空 slot、dead slot 或 unrelated tuple。
系统安全来自双方共同遵守协议。
## 6. on-access pruning 的触发条件
普通 SELECT/UPDATE/DELETE 访问 heap page 时可能触发 on-access pruning。
入口是 `heap_page_prune_opt()`。
它是 opportunistic 函数。
调用者必须持有 buffer pin，但不能已经持有 buffer lock。
它首先排除 recovery。
recovery 不能随意写新的 WAL。
primary 很可能会发送对应的 cleaning WAL record。
所以 standby 上直接跳过 on-access pruning。
然后它读 `PageGetPruneXid(page)`。
如果 `pd_prune_xid` 无效，直接返回。
这里避免昂贵 horizon 检查。
`pd_prune_xid` 是 hint。
没有 hint 不代表绝对没有 dead tuple。
有 hint 也不代表一定能 prune。
它只表示“值得再看一眼”。
接着它构造 `GlobalVisState`。
如果 `GlobalVisTestIsRemovableXid(vistest, prune_xid, true)` 认为这个 XID 还不能移除，直接返回。
这一步把 page hint 和全局 visibility horizon 连接起来。
随后它判断页面是否值得 prune。
当前启发式是：
page 被标记 `PD_PAGE_FULL`。
或者 heap free space 小于 fillfactor target 和 `BLCKSZ / 10` 的较大值。
注意这里最初不持锁。
`PageGetHeapFreeSpace()` 可能读到近似值。
源码注释承认这只是 heuristic。
避免无谓锁竞争比得到绝对准确 free space 更重要。
如果看起来值得 prune，它尝试 `ConditionalLockBufferForCleanup(buffer)`。
失败就返回。
on-access pruning 不阻塞等 cleanup lock。
因为这是前台访问路径，pruning 是机会性维护。
拿到 cleanup lock 后，它会重新检查 page free space。
因为之前没锁时的判断可能过期。
如果仍然值得 prune，才进入 `heap_page_prune_and_freeze()`。
on-access pruning 不传 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
原因是它不能安全判断 relation 是否没有 index。
因此它不会把可能被 index 指向的 root 直接设成 `LP_UNUSED`。
它可以做 redirect、dead、heap-only unused 和 defrag。
它通常也不会更新 FSM。
`heap_page_prune_opt()` 注释说明：
它避免把 unrelated UPDATE/INSERT 产生的 free space 立刻报告给 FSM。
这样新释放的空间更倾向被同一 page 的 UPDATE 复用。
这也是 HOT locality 的一部分。
## 7. 主流程：`heap_page_prune_and_freeze()`
`heap_page_prune_and_freeze()` 是 pruning、freezing、VM update 的共享实现。
本节只关注 pruning 相关部分。
调用者必须持有 heap buffer pin 和 cleanup lock。
参数用 `PruneFreezeParams` 传入：
relation。
buffer。
pinned VM buffer。
reason。
options。
`GlobalVisState`。
VACUUM cutoffs。
`reason` 有三个值：
`PRUNE_ON_ACCESS`。
`PRUNE_VACUUM_SCAN`。
`PRUNE_VACUUM_CLEANUP`。
这三个 reason 在 redo 语义上大多相同。
但 WAL 描述和调试分析需要区分来源。
`options` 的关键 flag 是：
`HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
`HEAP_PAGE_PRUNE_FREEZE`。
`HEAP_PAGE_PRUNE_ALLOW_FAST_PATH`。
`HEAP_PAGE_PRUNE_SET_VM`。
本节最重要的是 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
它只在 caller 确定 dead item 不需要保留 index TID 时使用。
VACUUM 在 relation 没有 index 时可以用它。
普通 on-access pruning 不用它。
主流程先调用 `prune_freeze_setup()` 初始化 `PruneState`。
`PruneState` 是一次 page pruning 的 backend-local 工作状态。
它记录待执行的 redirection、dead、unused offsets。
它也记录 visibility 结果、HOT root 列表、heap-only 列表和统计结果。
它还记录 `latest_xid_removed`。
这个字段最后参与 snapshot conflict horizon。
`prune_freeze_setup()` 的重要设计是：
先规划，再执行。
扫描和判断时只填数组。
真正改 page 的动作放到 critical section 内。
这样 critical section 尽量短。
也让 WAL replay 能用同一批数组执行同样 page change。
初始化时 `new_prune_xid` 设为 invalid。
如果后面发现 `RECENTLY_DEAD`、`DELETE_IN_PROGRESS` 或 insert-in-progress 等未来可能变 prunable 的 tuple，就更新这个 hint。
初始化还读取 visibility map 状态。
如果 VM bit 和 page hint 不一致，会先修复。
这不是本节重点，但它说明 pruning 不是孤立 page 动作。
page header、VM 和 WAL 必须保持一致边界。
随后进入 `prune_freeze_plan()`。
这个函数扫描 line pointer。
它从高 offset number 向低 offset number 扫描。
源码注释说明这对 CPU prefetch 更友好，因为 tuple bytes 通常按相反方向排列。
这不是正确性要求。
这是 hot path 成本优化。
规划阶段对每个 offset 做分类。
`LP_UNUSED` 记录 unchanged unused。
`LP_DEAD` 根据 `mark_unused_now` 决定保持 dead 或转 unused。
`LP_REDIRECT` 加到 root items。
`LP_NORMAL` 则构造 `HeapTupleData` 并调用 `heap_prune_satisfies_vacuum()`。
visibility 结果只计算一次。
源码特别强调这是 correctness 要求。
同一个 tuple 如果调用 visibility 两次，结果可能因 horizon 更新或事务结束而变化。
例如 `RECENTLY_DEAD` 可能变成 `DEAD`。
例如 `INSERT_IN_PROGRESS` 可能因插入事务 abort 变成 `DEAD`。
pruning 规划必须基于一次一致的 per-tuple classification。
随后它把非 heap-only normal tuple 放入 root items。
把 heap-only tuple 放入 heaponly items。
先处理 root items，才能沿 HOT chain 做整体判断。
未被任何 root chain 处理到的 heap-only tuple 会被单独处理。
这主要处理 aborted HOT update 产生的孤儿 heap-only tuple。
如果 dead heap-only tuple 仍然显示自己 hot-updated，却没有被任何 chain 链到，源码选择报错。
因为继续删除可能隐藏 page 结构损坏。
## 8. HOT chain pruning 如何压缩 chain
核心决策在 `heap_prune_chain()`。
它从 root offset 开始。
root 可能是 `LP_NORMAL`。
root 也可能已经是 `LP_REDIRECT`。
如果 root 是 redirect，它跳到 redirect target。
然后沿 HOT chain 走。
每走一个 `LP_NORMAL` tuple，它检查两件事。
第一，当前 offset 必须未处理。
第二，如果有 `priorXmax`，当前 tuple 的 `xmin` 必须等于 `priorXmax`。
第二个检查很关键。
它防止追到被复用的无关 tuple。
如果 offset 已经越界、unused、dead、redirect 出现在非链头，或者 `xmin` 不匹配，chain traversal 停止。
这不是“尽量继续”的逻辑。
这是防止 corruption 扩大的保守边界。
对每个 chain member，函数读取预先计算的 `htsv[offnum]`。
如果是 `HEAPTUPLE_DEAD`，记录目前 chain 中最新的 dead 位置。
同时用 `HeapTupleHeaderAdvanceConflictHorizon()` 推进 `latest_xid_removed`。
如果是 `HEAPTUPLE_RECENTLY_DEAD`，它会继续往后看。
源码注释说明：
如果 `RECENTLY_DEAD` 前面或后面出现真正 `DEAD`，某些 `RECENTLY_DEAD` 可以随 dead prefix 一起被剪掉。
这是因为链中后继 tuple 的插入事务更新，能提供更保守的 conflict horizon。
如果遇到 `LIVE`、`INSERT_IN_PROGRESS` 或 `DELETE_IN_PROGRESS`，进入处理阶段。
这表示 chain 后半段不能被移除。
然后根据 `ndeadchain` 和 `nchain` 得出三种结果。
第一种结果：
没有发现 dead tuple。
chain 保持不变。
root 如果是 redirect，就保持 redirect。
所有 normal tuple 记录 unchanged。
第二种结果：
整条 chain 都 dead。
root 变 `LP_DEAD` 或在允许时直接变 `LP_UNUSED`。
其它 heap-only 成员变 `LP_UNUSED`。
这是整条 HOT chain 不再有任何 non-dead member 的情况。
root 不能普通释放，因为 index 可能还指向 root TID。
第三种结果：
chain 前缀 dead，后面还有 non-dead member。
root 变 `LP_REDIRECT`，指向第一个 non-dead member。
dead prefix 中的 heap-only tuple 变 `LP_UNUSED`。
后面的 normal tuple 保持不变。
这就是 HOT chain 压缩。
压缩前：
```text
index -> lp[1]
lp[1] normal old root, HOT-updated, dead
lp[2] normal heap-only, dead
lp[3] normal heap-only, live or recently dead
```
压缩后：
```text
index -> lp[1]
lp[1] redirect -> lp[3]
lp[2] unused
lp[3] normal heap-only, live or recently dead
```
注意 `lp[3]` 仍然是 heap-only tuple。
它没有自己的普通 index entry。
但 index scan 先从 `lp[1]` 进入，再通过 redirect 到 `lp[3]`。
这就是为什么 redirect target 必须是 heap-only tuple。
`heap_page_prune_execute()` 的 assertion 也检查了这一点。
如果整条 chain 都 dead：
```text
index -> lp[1]
lp[1] dead
lp[2] unused
lp[3] unused
```
此时 index entry 仍可能指向 `lp[1]`。
所以 `lp[1]` 是 `LP_DEAD`，不是 `LP_UNUSED`。
VACUUM index pass 删除 index entry 后，heap pass 才能把 `lp[1]` 改成 `LP_UNUSED`。
如果 relation 没有 index，VACUUM 可以传 `mark_unused_now`。
这时 would-be `LP_DEAD` 可以直接 `LP_UNUSED`。
因为没有 index entry 需要保留 root TID。
这是重要的边界例外。
## 9. `LP_REDIRECT`、`LP_DEAD`、`LP_UNUSED` 的精确边界
`LP_REDIRECT` 的含义是：
这个 offset number 仍然是外部 TID 入口。
它已经没有 tuple storage。
它的 `lp_off` 是另一个 offset number。
它只能用于 HOT chain root。
它保留 index entry 可达性。
它不表示 tuple dead。
它也不表示 redirect target 对所有 snapshot 可见。
target 仍然要按 snapshot 检查。
`LP_REDIRECT` 存在时，pageinspect 会看到 `lp_flags = 2`。
`lp_off` 数值看起来可能很小。
此时不要把它当 byte offset。
它是 redirect target offset。
`LP_DEAD` 的含义是：
这个 offset number 对应的 tuple 已经可被所有相关事务视为 removable。
但 index entry 可能还没有删。
因此 offset 不能复用。
`LP_DEAD` 是 heap 和 index vacuum 的协议桥。
它通常没有 tuple storage。
pageinspect 会看到 `lp_flags = 3`，`lp_len = 0`。
`LP_DEAD` 不是“普通可见性 dead tuple”的同义词。
一个 `LP_NORMAL` tuple 也可能对 vacuum 判断为 `HEAPTUPLE_DEAD`。
pruning 执行后才可能把它变成 `LP_DEAD` 或 `LP_UNUSED`。
`LP_UNUSED` 的含义是：
这个 offset number 空闲。
新 tuple 可以立即占用它。
它不能再被旧 index entry 引用。
如果旧 index entry 还存在，`LP_UNUSED` 会让 index TID 失去可验证目标。
这会在 index deletion 检查中被视为 corruption。
`LP_UNUSED` 也不等于 page free space 已经连续可用。
它只释放 line pointer。
tuple bytes 的洞要靠 `PageRepairFragmentation()` compact 后才变成 `pd_lower..pd_upper` 之间的连续空间。
`LP_DEAD` 和 `LP_REDIRECT` 都可能没有 storage。
所以“有没有 tuple bytes”不是判断能否复用的标准。
能否复用要看是否还有外部引用和协议阶段。
边界总结：
```text
LP_REDIRECT: root TID still needed; chain still has reachable successor.
LP_DEAD: root/simple TID may still exist in indexes; tuple logically removable.
LP_UNUSED: no remaining external TID obligation; slot can be reused.
```
## 10. 为什么不能破坏 index TID
普通 btree index tuple 里存 heap TID。
这个 heap TID 是 `(heap block, line pointer offset)`。
index AM 并不知道 heap tuple bytes 被移动到哪里。
它只知道 offset number。
如果 heap page 内 compaction 移动 tuple bytes，只要 line pointer 更新 `lp_off`，index 不受影响。
如果 line pointer offset 被释放并复用，旧 index entry 就可能指向新 tuple。
这就是最危险的错误。
HOT 进一步强化了这个约束。
HOT update 不给 heap-only successor 建普通 index entry。
所以所有 index entry 都指向 root TID。
root tuple bytes 可以消失。
root line pointer 不能在 index cleanup 前消失。
`LP_REDIRECT` 解决的是“root tuple bytes 消失，但 root TID 仍能进入 chain”。
`LP_DEAD` 解决的是“整条 chain 消失，但 index cleanup 还需要 root TID 作为待删对象”。
`LP_UNUSED` 只能在没有 index obligation 后出现。
`README.HOT` 解释了为什么 PostgreSQL 不选择 page-local pruning 时重新计算 index key 删除 index entry。
函数索引、partial index、用户定义函数和 allegedly immutable 函数都让重新计算 index entry 有 correctness 风险。
即使能计算，也会把一个 page-local 维护动作变成任意 index/search/code execution。
这不是 HOT 设计接受的复杂度。
因此 HOT 的工程取舍是：
更新时严格限制 HOT-safe。
读取时通过 root TID 追同页 chain。
剪枝时保留 root line pointer 语义。
VACUUM 时批量清理 index。
这个设计让单页 pruning 不需要访问 index。
代价是 heap page 上会留下 `LP_REDIRECT` 和 `LP_DEAD` stub。
这正是 line pointer bloat 的来源之一。
PostgreSQL 用 `MaxHeapTuplesPerPage` 限制最坏情况。
`PageGetHeapFreeSpace()` 在 line pointer 数达到上限且没有 free line 时返回 0。
## 11. visibility horizon：什么时候算可移除
pruning 不是按当前查询 snapshot 判断。
它要判断 tuple 是否还可能被任何相关事务看到。
这就是 vacuum-style visibility。
`HeapTupleSatisfiesVacuumHorizon()` 是核心函数。
它先判断 `xmin`。
插入事务 abort，则 tuple 从未对任何事务可见，可以 `HEAPTUPLE_DEAD`。
插入事务 still running，则 `HEAPTUPLE_INSERT_IN_PROGRESS`。
插入事务 committed，继续判断 `xmax`。
如果 `xmax` invalid 或只是 lock-only，tuple 仍 live。
如果 deleting/updating transaction still running，则 `HEAPTUPLE_DELETE_IN_PROGRESS`。
如果 deleting/updating transaction committed，函数通常返回 `HEAPTUPLE_RECENTLY_DEAD` 并输出 `dead_after`。
`RECENTLY_DEAD` 表示：
删除/更新已提交。
但是否能物理移除，还要和 horizon 比较。
`HeapTupleSatisfiesVacuum()` 用 `OldestXmin` 比较。
`heap_prune_satisfies_vacuum()` 在 `pruneheap.c` 中先调用 `HeapTupleSatisfiesVacuumHorizon()`。
如果结果不是 `RECENTLY_DEAD`，直接返回。
如果有 VACUUM cutoffs，且 `dead_after` 早于 `OldestXmin`，返回 `HEAPTUPLE_DEAD`。
否则再问 `GlobalVisTestIsRemovableXid()`。
如果全局 visibility state 认为可移除，也返回 `HEAPTUPLE_DEAD`。
否则保留 `RECENTLY_DEAD`。
这里有两个重要细节。
第一，on-access pruning 没有 VACUUM cutoffs。
它主要依赖 `GlobalVisState`。
第二，VACUUM 的 `OldestXmin` 是 relation vacuum 开始时计算的 cutoff。
但 `GlobalVisState` 可能在 vacuum 期间变得更先进。
所以同一 tuple 在不同时间点可能从 `RECENTLY_DEAD` 变 `DEAD`。
这就是 pruning 规划阶段只计算一次 visibility 的原因。
重复计算可能让同一个 chain 内的判断不一致。
long transaction 会拖住 horizon。
只要某个旧 snapshot 仍可能看到旧版本，pruning 不能把它变 `DEAD`。
因此你可能看到大量 HOT old versions 留在 page 上。
这不是 cleanup lock 或 VACUUM “没工作”。
可能是 visibility horizon 不允许。
standby query 也要考虑。
WAL replay 物理移除 tuple 时，如果 standby 上有 query 还可能看到这些 tuple，recovery 需要冲突处理。
这就是 WAL record 中 snapshot conflict horizon 的来源。
## 12. cleanup lock：为什么普通 exclusive lock 不够
pruning 会移动 tuple bytes。
它也会把 line pointer 改成 redirect/dead/unused。
其它 backend 可能持有 buffer pin，并且有指向 page 内 tuple bytes 的 `HeapTuple` 指针。
如果 pruning 在它们持 pin 时移动 bytes，指针就会悬空或指向错误内容。
因此需要 buffer cleanup lock。
cleanup lock 的语义比普通 content exclusive lock 更强。
它要求没有其它 backend 持有会阻塞 cleanup 的 pin。
`README.HOT` 直接说明：
不能拿到 cleanup lock 时，不能 prune/defrag。
on-access pruning 用 `ConditionalLockBufferForCleanup()`。
拿不到就放弃。
这是前台路径的 fallback。
不会为了 reclaim page space 阻塞用户查询。
VACUUM 也先尝试 conditional cleanup lock。
如果拿不到，非 aggressive vacuum 可以走 `lazy_scan_noprune()`。
它持 share lock 扫描 page。
它不 prune，不 freeze。
但它可以收集已有 `LP_DEAD` 给 index cleanup，并统计 live/recently-dead。
如果 aggressive VACUUM 发现需要 freeze 才能推进 relfrozenxid，它会等待 cleanup lock。
这是 wraparound safety 边界。
空间回收可以延后。
防止 XID/MXID wraparound 不能无限延后。
所以 cleanup lock 失败不是单一结果。
普通访问：直接跳过 pruning。
普通 VACUUM：可能降级到 noprune。
aggressive VACUUM：必要时等待。
redo：如果 WAL record 要移动 tuple 或处理 redirect/dead，按 record flag 获取 cleanup lock。
这些差异来自同一个不变量：
移动 tuple bytes 或改变外部 TID 语义时，必须排除并发 pin 引用。
## 13. 执行阶段：从 plan arrays 到 page change
`heap_page_prune_and_freeze()` 规划完成后，会决定三个布尔值。
`do_prune` 表示有 redirect/dead/unused 变化。
`do_hint_prune` 表示只需要更新 `pd_prune_xid` 或清 `PD_PAGE_FULL`。
`do_freeze` 表示还要执行 freeze。
本节关注 `do_prune`。
真正改 page 前，它计算 `conflict_xid`。
如果设置 VM，用 newest live xid。
如果 freeze，用 freeze conflict xid。
如果 prune，用 `latest_xid_removed`。
最终取最保守的新值。
进入 critical section 后，如果需要 prune，就调用 `heap_page_prune_execute()`。
这个函数接受三类数组：
`redirected`。
`nowdead`。
`nowunused`。
`redirected` 每个 redirection 占两个 offset：
from offset。
to offset。
执行 redirection 时，它验证 from 是 HOT chain root。
如果 from 有 storage，必须是非 heap-only normal tuple。
to 必须是 normal tuple，且 tuple header 是 heap-only。
然后 `ItemIdSetRedirect(fromlp, tooff)`。
执行 nowdead 时，它验证需要留下 `LP_DEAD` 的 line pointer 不是 heap-only tuple。
如果有 storage，必须是普通非 heap-only tuple。
如果没有 storage，通常是 whole HOT chain dead 时的 redirect root。
然后 `ItemIdSetDead(lp)`。
执行 nowunused 时，普通 pruning 下如果同时有 dead items，unused 项必须是 heap-only tuple。
因为 root/simple tuple 如果可能被 index 指向，不能跳过 `LP_DEAD` 阶段。
`lp_truncate_only` 用于 VACUUM 第二阶段。
这时只能把已 `LP_DEAD` 且无 storage 的 line pointer 改成 `LP_UNUSED`。
它不需要完整 cleanup lock。
最后，如果不是 `lp_truncate_only`，调用 `PageRepairFragmentation(page)`。
`PageRepairFragmentation()` 会扫描仍有 storage 的 line pointer。
它把 surviving tuple bytes compact 到页尾。
它更新每个 surviving line pointer 的 `lp_off`。
它删除 line pointer array 尾部的 `LP_UNUSED`。
它设置或清除 `PD_HAS_FREE_LINES` hint。
这一步才让旧 tuple bytes 形成连续 free space。
如果只是 `ItemIdSetUnused()` 而不 compact，页面中间只是洞。
新 tuple 通常不能直接利用那些碎片。
因此 pruning 和 defragmentation 通常一起出现。
## 14. WAL：pruning 不是无日志内存整理
pruning 改的是 heap page 物理内容。
它可能改变 line pointer 状态。
它可能移动 tuple bytes。
它可能清 `pd_prune_xid`。
它可能设置 `PD_ALL_VISIBLE` 和 VM bits。
这些变化必须能 crash safe。
`log_heap_prune_and_freeze()` 写 `XLOG_HEAP2_PRUNE*` record。
当前基线有三种 opcode：
`XLOG_HEAP2_PRUNE_ON_ACCESS`。
`XLOG_HEAP2_PRUNE_VACUUM_SCAN`。
`XLOG_HEAP2_PRUNE_VACUUM_CLEANUP`。
`heapam_xlog.h` 说明这三者 redo 操作没有本质区别。
区分主要用于调试和分析来源。
WAL record 的主体是 `xl_heap_prune`。
它用 flags 表示包含哪些 sub-record。
可能包含：
freeze plans。
redirected offsets。
dead offsets。
unused offsets。
snapshot conflict horizon。
VM all-visible/all-frozen flags。
cleanup-lock required flag。
`redirected` sub-record 存 `2 * nredirected` 个 offset number。
`dead` sub-record 存 `ndead` 个 offset number。
`unused` sub-record 存 `nunused` 个 offset number。
这说明 redo 不重新做 visibility 判断。
redo 只是执行 primary 已经做出的 page change plan。
这是 crash recovery 的关键边界。
visibility 判断依赖当时的 ProcArray、CLOG、MultiXact 和 horizon。
恢复时不能重新推导同样答案。
所以 WAL 必须记录结果，而不是记录“请再 prune 一次”。
`XLHP_CLEANUP_LOCK` 表示 redo 需要 cleanup lock。
如果 WAL record 包含 redirection 或 dead item，通常需要 cleanup lock。
VACUUM 第二阶段只把已 `LP_DEAD` 改 `LP_UNUSED`，不移动 tuple bytes，可以用普通 exclusive lock。
`heap_xlog_prune_freeze()` 在 redo 时读取该 flag。
它调用 `XLogReadBufferForRedoExtended()` 时传入是否获取 cleanup lock。
如果 record 带 `XLHP_HAS_CONFLICT_HORIZON`，hot standby 会调用 `ResolveRecoveryConflictWithSnapshot()`。
这表示：
如果 standby 上有 query 的 snapshot 还可能需要这些被移除的 tuple，恢复必须等待或取消冲突 query。
否则 standby 会在物理状态上提前丢掉 query 还需要的版本。
`latest_xid_removed` 和 `HeapTupleHeaderAdvanceConflictHorizon()` 就是为这个服务。
WAL 还处理 full-page image。
如果 heap page LSN invalid，`log_heap_prune_and_freeze()` 会强制 FPI。
原因是 heap extension 本身不 WAL-log page initialization。
恢复时如果只看到 prune record 而 page 还像新页，会 PANIC。
这类细节说明 pruning 不是“可有可无的 hint bit”。
只要真的改 page physical layout，就必须遵守 WAL-before-data。
如果只有 `pd_prune_xid` 或 `PD_PAGE_FULL` hint 更新，且没有 prune/freeze/VM change，可以走 `MarkBufferDirtyHint()`。
这类 hint 不构成本节的空间回收主体。
## 15. VACUUM 的两阶段边界
VACUUM 不只是调用 pruning。
它还负责最终清理 index entry。
`lazy_scan_prune()` 在第一轮 heap pass 中调用 `heap_page_prune_and_freeze()`。
有 index 时，它不会直接把 root/simple dead item 变 `LP_UNUSED`。
它会收集 `presult.deadoffsets`。
这些 offset 对应 page 上的 `LP_DEAD`。
然后 `dead_items_add()` 把 `(block, offset)` 放进 `TidStore`。
后续 `vac_bulkdel_one_index()` 用这些 TID 删除 index entry。
所有 index 都处理完后，`lazy_vacuum_heap_page()` 再访问 heap page。
它把记录的 `LP_DEAD` 改成 `LP_UNUSED`。
然后调用 `PageTruncateLinePointerArray()` 尝试收缩尾部 line pointer array。
这个第二阶段用 `PRUNE_VACUUM_CLEANUP` WAL record。
它传 `cleanup_lock = false`。
因为它只处理已 `LP_DEAD` 的 line pointer，不移动 normal tuple bytes。
无 index relation 是例外。
`lazy_scan_prune()` 在 `vacrel->nindexes == 0` 时传 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
这允许 pruning 阶段直接把 would-be `LP_DEAD` 设成 `LP_UNUSED`。
因为没有 index entry 需要先删除。
这就是 `LP_DEAD -> index vacuum -> LP_UNUSED` 协议的核心边界。
如果你看到有 index 表上出现 `LP_DEAD`，这不是泄漏。
它可能是正常的两阶段中间状态。
如果 VACUUM 被取消、failsafe 跳过 index cleanup、或者 cleanup lock 拿不到，`LP_DEAD` 可能延后存在。
下次 VACUUM 再收尾。
## 16. 错误路径、异常路径与 fallback
第一类 fallback 是 on-access pruning 拿不到 cleanup lock。
`heap_page_prune_opt()` 直接返回。
前台查询继续走正常 visibility。
空间暂时不回收。
最坏结果通常是后续 UPDATE 找不到同页空间，变成 cold update 或放到新页。
这影响 bloat 和 HOT ratio，不破坏 correctness。
第二类 fallback 是 `pd_prune_xid` 误导。
hint 可能过期。
如果 hint 无效，可能错过一次 pruning 机会。
如果 hint 有效但 horizon 不允许，函数返回。
如果 hint 指示可 prune 但重拿锁后页面 free space 已变，重新检查会跳过。
hint 错误最多导致多做或少做维护，不决定可见性。
第三类 fallback 是 VACUUM 没拿到 cleanup lock。
非 aggressive VACUUM 可以走 `lazy_scan_noprune()`。
它仍然能收集已有 `LP_DEAD`。
它仍然能统计 live/recently-dead。
它不能移动 tuple、不能 prune chain、不能 freeze 需要修改的 tuple。
如果 aggressive VACUUM 必须 freeze，它会等待 cleanup lock。
第四类 fallback 是 relation 有 index。
即使 tuple 已 globally dead，也不能直接 `LP_UNUSED`。
必须留下 `LP_DEAD` 给 index vacuum。
这不是保守过度。
这是避免 index TID 指向复用 slot 的必要协议。
第五类异常是 HOT chain 断裂或 page corruption。
`heap_prune_chain()` 遇到不匹配的 `xmin/xmax` 会停止追链。
`heap_hot_search_buffer()` 也会把不匹配当作链结束。
这是因为 offset 可能已复用。
如果发现 dead heap-only tuple 没有被任何 HOT chain 链到且仍然 hot-updated，`prune_freeze_plan()` 会报错。
源码选择保留证据并抛 ERROR，而不是冒险删除。
第六类异常是 VM/page hint 不一致。
`heap_page_prune_and_freeze()` 会检测 VM bit set 但 page `PD_ALL_VISIBLE` clear 的情况。
也会处理 page all-visible 但存在 `LP_DEAD` 或非全可见 tuple 的情况。
这类修复说明 pruning 过程也会维护 VM correctness。
第七类异常是 WAL/redo 冲突。
standby replay 需要移除 tuple 时，如果 standby query 仍可能看到它，`ResolveRecoveryConflictWithSnapshot()` 可能等待或取消 query。
这不是 pruning 自己“慢”。
这是 physical recovery 必须保持 MVCC 语义。
第八类异常是 failsafe。
VACUUM 为避免 wraparound failure，可能绕过非必要 index vacuuming、index cleanup 和 heap truncation。
这会让某些 `LP_DEAD` 或 dead items 延后清理。
但它优先保证 anti-wraparound 安全。
## 17. 成本、资源与跨模块传播
pruning 的 CPU 成本随 page 上 line pointer 数增长。
`prune_freeze_plan()` 要扫描每个 line pointer。
对 `LP_NORMAL` tuple 要做 visibility classification。
HOT chain 处理还要追 `t_ctid`。
单页上 line pointer 数被 `MaxHeapTuplesPerPage` 限制。
这个上限也是避免 HOT pruning 产生无限 line pointer bloat 的工程限制。
visibility 成本可能触发 CLOG 或 MultiXact 查询。
hint bits 已经设置时成本低。
没有 hint bits 时，需要查事务状态。
MultiXact 还可能访问 pg_multixact。
因此同样的 page layout，在不同 hint-bit 温度下成本不同。
cleanup lock 是 contention 成本。
前台路径不等待。
VACUUM 可能等待。
如果有长时间 pin 的 backend，pruning 和 defrag 会延后。
这会传播成 HOT ratio 降低、page free space 不可用、更多 cold update、更多 index churn。
WAL 成本来自 `XLOG_HEAP2_PRUNE*`。
如果 pruning 只是 hint update，可能不写完整 WAL。
如果改变 line pointer 或移动 tuple bytes，就要写 WAL。
如果需要 FPI，WAL 量会放大。
checkpoint 和 full_page_writes 状态会影响一次 pruning 的 WAL 体积。
index vacuum 成本来自 `LP_DEAD`。
有 index 的 relation 不能直接释放 root TID。
VACUUM 需要把 dead TID 存进 `TidStore`。
然后对每个 index 批量删除。
`maintenance_work_mem` 或 `autovacuum_work_mem` 会影响 dead_items 容量。
容量不足会增加 index vacuum cycle。
visibility horizon 是跨模块传播点。
长事务、replication slot、logical decoding、hot standby query 都可能推迟 dead tuple 的 physical removal。
最终表现为 heap bloat、HOT chain 变长、VACUUM 不能清理。
这不一定是 heapam 本身性能问题。
它可能是 workload 和事务生命周期问题。
## 18. 可观测入口：能看到什么，不能看到什么
`pageinspect` 能看到 page 当前物理状态。
`heap_page_items(get_raw_page(...))` 能看到 `lp`、`lp_off`、`lp_flags`、`lp_len`、`t_xmin`、`t_xmax`、`t_ctid`、`t_infomask`、`t_infomask2`。
`heap_tuple_infomask_flags()` 能把 `HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE` 等 flag 解码。
`page_header(get_raw_page(...))` 能看到 `lower`、`upper`、`special`、`prune_xid` 和 page flags。
这可以观察 compact 前后 free space 边界变化。
`pg_visibility` 能看到 VM 和 `PD_ALL_VISIBLE` 的关系。
它不能告诉你某条 HOT chain 为什么没被 prune。
`pg_stat_all_tables` 能看到 `n_tup_hot_upd`、`n_dead_tup`、`vacuum_count`、`autovacuum_count`。
这是 relation-level 累计统计。
它不能告诉你某个 line pointer 的状态。
`pg_stat_activity` 能看到长事务的 `backend_xmin`、`xact_start`、wait event。
它不能直接告诉你哪个 page 被哪个 horizon 卡住。
`pg_waldump` 能看到 `PRUNE_ON_ACCESS`、`PRUNE_VACUUM_SCAN`、`PRUNE_VACUUM_CLEANUP` 记录。
`heapdesc.c` 会打印 snapshot conflict horizon、VM flags 和 prune sub-record 信息。
它能证明 pruning 写了 WAL。
它不能直接还原 SQL 级原因。
`VACUUM VERBOSE` 能看到 missed dead tuples、dead item counts、index vacuum cycles 等信息。
cleanup lock contention 可能表现为 dead tuples missed。
但日志仍然是聚合信息，不是 page-level trace。
`gdb` 或临时日志最适合观察主流程。
推荐断点：
`heap_page_prune_opt()`。
`heap_page_prune_and_freeze()`。
`heap_prune_chain()`。
`heap_page_prune_execute()`。
`PageRepairFragmentation()`。
`log_heap_prune_and_freeze()`。
观察时重点看：
`prstate->nredirected`。
`prstate->ndead`。
`prstate->nunused`。
`prstate->latest_xid_removed`。
`prstate->new_prune_xid`。
`redirected[]`。
`nowdead[]`。
`nowunused[]`。
## 19. 课堂实验一：观察 HOT chain 变成 `LP_REDIRECT`
这个实验目标是看到：
HOT update 后，index entry 仍指向 root TID。
on-access pruning 后，root line pointer 变 `LP_REDIRECT`。
中间 heap-only tuple 变 `LP_UNUSED`。
最新版本仍然是 `LP_NORMAL` + `HEAP_ONLY_TUPLE`。
准备：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS hp_hot;
CREATE TABLE hp_hot (
    id int PRIMARY KEY,
    v int,
    pad text
) WITH (fillfactor = 70, autovacuum_enabled = off);
INSERT INTO hp_hot VALUES (1, 0, repeat('x', 1000));
```
执行多次 HOT-safe update。
`id` 是索引列，不改。
`v` 和 `pad` 没有普通 index。
```sql
DO $$
BEGIN
  FOR i IN 1..20 LOOP
    UPDATE hp_hot SET v = v + 1 WHERE id = 1;
  END LOOP;
END $$;
```
先看 page 原始状态：
```sql
SELECT lp, lp_flags, lp_off, lp_len, t_ctid, t_xmin, t_xmax,
       raw_flags, combined_flags
FROM heap_page_items(get_raw_page('hp_hot', 0)) h
LEFT JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) f
       ON h.lp_flags = 1
ORDER BY lp;
```
你通常会看到多个 `LP_NORMAL`。
旧版本带 `HEAP_HOT_UPDATED`。
新版本带 `HEAP_ONLY_TUPLE`。
某些环境下，读取 page 前可能已经触发过 pruning。
这是正常的 timing-dependent 现象。
强制走 index fetch 并触发 on-access pruning：
```sql
SET enable_seqscan = off;
SELECT * FROM hp_hot WHERE id = 1;
RESET enable_seqscan;
```
再次查看：
```sql
SELECT lp, lp_flags, lp_off, lp_len, t_ctid,
       raw_flags, combined_flags
FROM heap_page_items(get_raw_page('hp_hot', 0)) h
LEFT JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) f
       ON h.lp_flags = 1
ORDER BY lp;
```
期望观察：
`lp_flags = 2` 的行表示 `LP_REDIRECT`。
这一行的 `lp_off` 是 redirect target offset，不是 tuple byte offset。
某些中间 offset 可能变 `lp_flags = 0`。
这是 `LP_UNUSED`。
最新 tuple 仍是 `lp_flags = 1`。
它的 flags 通常包含 `HEAP_ONLY_TUPLE`。
如果没有看到 `LP_REDIRECT`，可能原因有三类。
第一，页面还没有达到 on-access pruning heuristic。
第二，有长事务拖住 horizon。
第三，更新没有形成同页 HOT chain。
可以增加 update 次数，或检查 `n_tup_hot_upd`：
```sql
SELECT n_tup_upd, n_tup_hot_upd, n_dead_tup
FROM pg_stat_all_tables
WHERE relname = 'hp_hot';
```
把现象回到源码：
`heap_update()` 设置 `HEAP_HOT_UPDATED` / `HEAP_ONLY_TUPLE`。
`heapam_index_fetch_tuple()` 调 `heap_page_prune_opt()`。
`heap_prune_chain()` 把 dead prefix 压缩成 redirect。
`heap_page_prune_execute()` 调 `ItemIdSetRedirect()` 和 `ItemIdSetUnused()`。
`PageRepairFragmentation()` 合并 tuple bytes 空洞。
## 20. 课堂实验二：长事务如何阻止 pruning
这个实验目标是看到 visibility horizon 不是当前 session 的可见性。
Session A：
```sql
BEGIN;
SELECT * FROM hp_hot WHERE id = 1;
```
保持事务不提交。
Session B：
```sql
DO $$
BEGIN
  FOR i IN 1..20 LOOP
    UPDATE hp_hot SET v = v + 1 WHERE id = 1;
  END LOOP;
END $$;
SET enable_seqscan = off;
SELECT * FROM hp_hot WHERE id = 1;
RESET enable_seqscan;
```
Session B 查看 page：
```sql
SELECT lp, lp_flags, lp_off, lp_len, t_ctid
FROM heap_page_items(get_raw_page('hp_hot', 0))
ORDER BY lp;
```
你可能看到旧 tuple 仍然保留为 `LP_NORMAL`。
因为 Session A 的 snapshot 可能还需要旧版本。
Session B 当前查询看不到这些旧版本，不代表系统能物理移除它们。
查看 horizon 线索：
```sql
SELECT pid, state, xact_start, backend_xmin, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY xact_start;
```
提交 Session A：
```sql
COMMIT;
```
Session B 再触发一次访问或 VACUUM：
```sql
SET enable_seqscan = off;
SELECT * FROM hp_hot WHERE id = 1;
RESET enable_seqscan;
```
再查看 page。
旧版本更可能被 redirect/unused 压缩。
把现象回到源码：
`HeapTupleSatisfiesVacuumHorizon()` 先给出 `RECENTLY_DEAD`。
`heap_prune_satisfies_vacuum()` 用 `GlobalVisTestIsRemovableXid()` 判断是否能升级为 `DEAD`。
长事务让 `GlobalVisState` 不能越过旧 XID。
## 21. 课堂实验三：观察 `LP_DEAD` 与 VACUUM 收尾
这个实验目标是理解 `LP_DEAD` 是 index cleanup 前的协议状态。
准备一张有 index 的表：
```sql
DROP TABLE IF EXISTS hp_dead;
CREATE TABLE hp_dead (
    id int PRIMARY KEY,
    pad text
) WITH (fillfactor = 80, autovacuum_enabled = off);
INSERT INTO hp_dead
SELECT g, repeat('x', 500)
FROM generate_series(1, 20) g;
DELETE FROM hp_dead WHERE id = 10;
```
让 page 有 prune 机会：
```sql
SET enable_seqscan = off;
SELECT * FROM hp_dead WHERE id = 1;
RESET enable_seqscan;
```
观察 page：
```sql
SELECT lp, lp_flags, lp_off, lp_len, t_ctid
FROM heap_page_items(get_raw_page('hp_dead', 0))
ORDER BY lp;
```
如果看到 `lp_flags = 3`，就是 `LP_DEAD`。
这表示 heap tuple bytes 可以没有了。
但 index entry 可能还没删，所以 slot 不能复用。
执行 VACUUM：
```sql
VACUUM hp_dead;
```
再观察：
```sql
SELECT lp, lp_flags, lp_off, lp_len
FROM heap_page_items(get_raw_page('hp_dead', 0))
ORDER BY lp;
```
VACUUM 可能把 `LP_DEAD` 变成 `LP_UNUSED`。
如果这个 offset 在 line pointer array 尾部，还可能被 `PageTruncateLinePointerArray()` 收缩掉。
这个实验具有 timing 依赖。
如果 on-access pruning 没先留下 `LP_DEAD`，VACUUM 可能在一个完整流程里很快完成 index cleanup 和 heap cleanup。
这时你看到的是最终 `LP_UNUSED` 或更短的 line pointer array。
这不违背模型。
它只是没抓到中间态。
想稳定观察中间态，可以在源码里给 `lazy_scan_prune()`、`lazy_vacuum_heap_page()` 加临时日志或断点。
不要把调试日志提交到课程仓库。
## 22. 源码跟读实验
用 debug build 启动 PostgreSQL。
设置断点：
```text
break heap_update
break heap_page_prune_opt
break heap_page_prune_and_freeze
break heap_prune_chain
break heap_page_prune_execute
break PageRepairFragmentation
break log_heap_prune_and_freeze
```
在 `heap_update()` 里观察：
`use_hot_update`。
`oldtup.t_data->t_infomask2`。
`heaptup->t_data->t_infomask2`。
`oldtup.t_data->t_ctid`。
在 `heap_page_prune_opt()` 里观察：
`prune_xid`。
`PageIsFull(page)`。
`PageGetHeapFreeSpace(page)`。
`ConditionalLockBufferForCleanup()` 是否成功。
在 `heap_prune_chain()` 里观察：
`rootoffnum`。
`chainitems[]`。
`ndeadchain`。
`nchain`。
`priorXmax`。
在 `heap_page_prune_execute()` 里观察：
`nredirected`。
`ndead`。
`nunused`。
`redirected[]` 的 from/to offset。
`nowdead[]`。
`nowunused[]`。
在 `PageRepairFragmentation()` 前后观察：
`pd_lower`。
`pd_upper`。
surviving line pointer 的 `lp_off`。
在 `log_heap_prune_and_freeze()` 里观察：
`reason`。
`cleanup_lock`。
`conflict_xid`。
是否写 FPI。
这条实验能把 SQL 现象、line pointer 状态和 WAL record 连接起来。
## 23. 常见误区
误区一：
把 `LP_DEAD` 当成可复用 slot。
正确理解是：
`LP_DEAD` 是 dead TID stub。
它不能复用，除非 index obligation 已经解除。
误区二：
把 `LP_REDIRECT` 当成 tuple。
正确理解是：
`LP_REDIRECT` 没有 tuple storage。
它只把 root offset 连接到 successor offset。
误区三：
把 `lp_off` 总是解释成 byte offset。
正确理解是：
`LP_REDIRECT` 下 `lp_off` 是 offset number。
误区四：
把 `xmax` committed 当成立刻可 prune。
正确理解是：
必须看 `OldestXmin` / `GlobalVisState` / standby conflict horizon。
`RECENTLY_DEAD` 还不能物理移除。
误区五：
认为 SELECT 不会改变 page。
正确理解是：
heap/index scan 访问 page 时可能触发 on-access pruning。
它会写 WAL 并修改 page。
误区六：
认为 VACUUM 一定能 prune。
正确理解是：
cleanup lock、horizon、failsafe、index cleanup 和 aggressive freeze 都会改变路径。
误区七：
认为 pageinspect 看到 `LP_UNUSED` 就能证明没有历史 index entry。
正确理解是：
在正确系统里它应该表示没有旧 index obligation。
但如果 index corruption 已经发生，pageinspect 只能看到 heap page 当前状态，不能证明 index 完整性。
误区八：
把 `pd_prune_xid` 当作准确 dead tuple 计数。
正确理解是：
它只是 oldest potentially prunable XID hint。
它不计数，也不保证一定能 prune。
## 24. 讨论题
为什么 HOT update 必须限制在同一个 heap page？
如果 `LP_REDIRECT` 可以指向跨页 tuple，会破坏哪些边界？
为什么 root tuple bytes 可以被删除，但 root line pointer 不能立即变 `LP_UNUSED`？
为什么 heap-only tuple 的 line pointer 通常可以更早变 `LP_UNUSED`？
`HeapTupleSatisfiesVacuumHorizon()` 为什么返回 `RECENTLY_DEAD` 而不是直接判断 `DEAD`？
on-access pruning 为什么使用 conditional cleanup lock，而不是阻塞等待？
VACUUM 在 relation 没有 index 时为什么可以传 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`？
redo 为什么不能重新运行 visibility 判断来决定 pruning？
如果 standby query 和 pruning WAL record 冲突，应该牺牲谁，为什么？
`pageinspect` 看到 `lp_flags = 2` 时，`lp_off` 为什么不能按 byte offset 解释？
## 25. 本节小结
本节唯一主问题是：
旧 tuple version 什么时候能被安全剪掉，并且为什么不能破坏 index TID。
核心链路是：
HOT update 让 index entry 继续指向 root TID。
旧版本在 horizon 允许后变成 removable。
pruning 在 cleanup lock 下规划并执行 line pointer 状态变化。
root dead prefix 被压缩成 `LP_REDIRECT`。
整条 chain dead 时 root 变 `LP_DEAD`。
heap-only 或已无 index obligation 的 slot 变 `LP_UNUSED`。
`PageRepairFragmentation()` 再把 tuple bytes 合并成连续 free space。
核心状态是：
`ItemIdData.lp_flags`。
`HeapTupleHeader.t_ctid`。
`HEAP_HOT_UPDATED`。
`HEAP_ONLY_TUPLE`。
`pd_prune_xid`。
`GlobalVisState`。
`PruneState` 中的 redirected/dead/unused arrays。
核心边界是：
`LP_UNUSED` 才能复用。
`LP_REDIRECT` 保留 root TID 并跳转。
`LP_DEAD` 保留待 index cleanup 的 dead TID。
`RECENTLY_DEAD` 不能物理移除。
cleanup lock 保护 tuple bytes 移动和 line pointer 语义变化。
WAL 记录 pruning 的结果而不是重新计算条件。
错误和 fallback 的共同规律是：
空间回收可以延后。
TID 稳定性和 visibility correctness 不能降级。
拿不到 cleanup lock 就跳过或降级。
horizon 不允许就保留。
有 index obligation 就留下 `LP_DEAD`。
redo 需要冲突处理就按 WAL horizon 等待或取消 standby query。
可观测性上：
pageinspect 能看 line pointer 当前状态。
pg_visibility 能看 VM/page all-visible 边界。
pg_stat 能看 relation-level HOT/dead 统计。
pg_waldump 能看 pruning WAL。
但没有单个 SQL 视图能完整解释某个 tuple 为什么没被 prune。
最终可迁移规律是：
当一个系统要同时支持稳定外部引用和内部空间回收时，不能把“对象 bytes 的生命周期”和“外部 handle 的生命周期”混成一个状态。
PostgreSQL 用 line pointer state 把这两个生命周期拆开。
`LP_REDIRECT` 和 `LP_DEAD` 是延迟解除外部引用的协议状态。
`LP_UNUSED` 才是资源真正回到 allocator 的状态。
这个模型不仅适用于 heap pruning。
它也适用于理解 buffer pin、WAL redo、index deletion、cache invalidation 和任何需要延迟回收的内核结构。
