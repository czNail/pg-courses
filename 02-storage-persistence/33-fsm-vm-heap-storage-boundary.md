# PostgreSQL FSM、VM 与 heap storage 协作
## 课程定位
本节主题：FSM、VM 与 heap storage 协作。
上一节已经把 heap page 的物理布局、line pointer、tuple header 和 pruning 入口拆开。
这一节继续沿着 heap page 的状态往外看。
前置知识：
- 已理解 relation fork、buffer pin/content lock、page LSN、WAL-before-data。
- 已理解 heap tuple header、`LP_*` line pointer、`PD_ALL_VISIBLE`、`pd_prune_xid`。
- 已理解 MVCC snapshot、VACUUM、index-only scan 的基本执行现象。
本节唯一主问题：
heap page 的真实状态很贵，为什么 PostgreSQL 还要维护两个可重建的辅助 fork 来近似回答“哪里能写”和“哪里可不读”？
本节围绕的核心矛盾：
插入路径希望快速找到有空间的 page，不能每次全表扫描。
VACUUM 和 index-only scan 希望快速跳过不需要访问的 heap page，不能每次逐 tuple 做可见性判断。
但 heap page 的真实状态会被并发事务、pruning、HOT、freeze、crash recovery 不断改变。
如果辅助状态要求绝对实时准确，维护成本会压过收益。
如果辅助状态可以随便不准，又会返回错误结果或造成严重 bloat。
PostgreSQL 的选择是：
`FSM` 对空间采用近似、滞后、可修正的 hint。
`VM` 对可见性采用保守、单向可信、由 WAL 协同维护的辅助事实。
本节主流程：heap insert 从 FSM 找候选页并在失败后修正空间 hint；heap update/delete 清 VM bit 让 index-only scan 回退 heap；VACUUM 根据 visibility horizon 设置 VM all-visible/all-frozen 并更新 FSM；redo 用 WAL 恢复 VM 并维护可重建的辅助状态。
读完本节，你应该能判断：
- `FSM_FORKNUM` 里记录的是空间类别，不是 heap page 的真实 free bytes。
- `GetPageWithFreeSpace()` 返回的是候选页，不是插入承诺。
- `RecordAndGetPageWithFreeSpace()` 为什么同时修正旧页并继续搜索。
- `FreeSpaceMapVacuumRange()` 为什么让底层空间变化传播到上层 FSM page。
- `VM` 的 `all-visible` 和 `all-frozen` 分别允许跳过什么。
- 为什么 `all-frozen` 必须是 `all-visible` 的子集。
- 为什么 `VM` bit clear 是安全保守状态，bit set 是 correctness-sensitive 状态。
- heap insert/update/delete 为什么先准备 VM page pin，再在 critical section 里清 VM。
- VACUUM 为什么既消费 VM 来跳页，又生产 VM 来服务后续查询。
- index-only scan 为什么 VM bit clear 时退回 heap fetch。
- 为什么 FSM 和 VM 都是可重建辅助状态，但正确性风险完全不同。
- WAL redo 如何修复 VM，并如何顺手更新 FSM。
- 哪些现象可以用 SQL 和扩展看到，哪些只能用源码断点或日志推断。
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
src/backend/storage/freespace/freespace.c
src/backend/storage/freespace/fsmpage.c
src/backend/storage/freespace/indexfsm.c
src/backend/access/heap/visibilitymap.c
src/backend/access/heap/heapam.c
src/backend/access/heap/vacuumlazy.c
src/include/access/visibilitymap.h
src/include/storage/freespace.h
```
为把主链路补完整，本节还辅助核对：
```text
src/backend/access/heap/hio.c
src/backend/access/heap/pruneheap.c
src/backend/access/heap/heapam_xlog.c
src/backend/executor/nodeIndexonlyscan.c
src/include/access/visibilitymapdefs.h
src/include/storage/fsm_internals.h
contrib/pg_freespacemap
contrib/pg_visibility
```
行号来自：
```text
nl -ba <source-file>
```
## 1. 本节在总主线中的位置
第 27 节讲的是一个 heap page 内部如何保存 tuple。
那一层回答的是：
给定一个 `(block, offset)`，page 内如何定位 tuple bytes，并判断 tuple version 的语义。
本节的问题往外扩一层：
如果还不知道 block number，插入路径怎么找到候选 page？
如果已经有 index TID，执行器怎么知道能不能跳过 heap page？
如果 VACUUM 扫到 page，怎样把“这页有空间”或“这页全局可见”传给未来的 insert/scan？
这三个问题都不能只靠 heap page 本身解决。
因为 heap page 本身在 main fork。
逐页读 main fork 太贵。
PostgreSQL 因此在 relation 旁边维护两个辅助 fork：
```text
main fork: heap tuple 和 page header 的真实状态
fsm fork: 每个 heap block 的近似可用空间
vm fork: 每个 heap block 的 all-visible/all-frozen bits
```
本节不是分别介绍 FSM 和 VM。
本节只围绕一个主链路：
```text
heap page 改变
  -> 辅助 fork 被更新、滞后或清除
  -> insert/vacuum/index-only scan 消费这些辅助状态
  -> 命中失败或状态过期时回到 heap page 真相
```
这个链路是后续理解 table bloat、VACUUM skip pages、index-only scan Heap Fetches、standby promotion 后插入停顿的基础。
## 2. 核心矛盾与一句话运行模型
先给一句话模型：
```text
FSM 用“可错的空间近似”减少找页成本；VM 用“只允许保守缺失的可见性事实”减少读页成本。
heap page 仍然是唯一真相，所有消费者都必须在边界处准备 retry、heap fetch、redo 或重新扫描。
```
`FSM` 的设计偏向 performance hint。
一个 FSM entry 说某页有空间，最后可能没有。
这不会返回错误结果。
插入路径锁住 heap page 后发现空间不足，会记录真实空间并重试。
一个 FSM entry 说某页没空间，最后可能有。
这也不破坏 correctness。
代价是空间复用延后，甚至提前扩表。
`VM` 的设计偏向 correctness-sensitive cache。
一个 VM all-visible bit 没有设置，不代表 page 不是 all-visible。
这只是让 VACUUM 或 index-only scan 多做工作。
一个 VM all-visible bit 错误设置，就可能让 index-only scan 不访问 heap，从而返回对当前 snapshot 不可见的 tuple。
所以 VM 是辅助状态，但不是“随便错都可以”的 hint。
它只能在一个方向上保守：
```text
bit clear: 不知道，必须回到 heap 或扫描
bit set: 调用者可以相信，需要 WAL 和锁顺序保证
```
`all-frozen` 也是这样。
它告诉 VACUUM：
这个 page 上所有 tuple 都已经 frozen，反 wraparound VACUUM 也可以跳过。
如果错设 all-frozen，`relfrozenxid` 或 `relminmxid` 推进就可能越过仍需保护的 tuple。
因此 `visibilitymap.c:25-35` 明确说 VM 是 conservative。
## 3. 核心文件分工与阅读顺序
推荐按消费关系读，不按文件名读。
| 顺序 | 文件 | 读什么 | 为什么 |
| --- | --- | --- | --- |
| 1 | `src/include/storage/freespace.h` | `GetPageWithFreeSpace()`、`RecordPageWithFreeSpace()`、`FreeSpaceMapVacuumRange()` | 先看 FSM 对外承诺 |
| 2 | `src/backend/storage/freespace/freespace.c` | category、tree search、retry、truncate、redo helper | 建立“近似空间索引”模型 |
| 3 | `src/backend/storage/freespace/fsmpage.c` | page 内 binary tree、`fp_next_slot`、rebuild | 看一个 FSM page 内如何搜索和自修复 |
| 4 | `src/include/access/visibilitymap.h` | `VM_ALL_VISIBLE()`、`visibilitymap_set/clear/get_status()` | 先看 VM API 边界 |
| 5 | `src/include/access/visibilitymapdefs.h` | `VISIBILITYMAP_ALL_VISIBLE`、`VISIBILITYMAP_ALL_FROZEN` | 明确两个 bit 的物理定义 |
| 6 | `src/backend/access/heap/visibilitymap.c` | conservative 语义、pin/set/clear、WAL notes | 看 VM 为什么比 FSM 更严格 |
| 7 | `src/backend/access/heap/hio.c` | `RelationGetBufferForTuple()` | heap insert 真实找 page 的地方 |
| 8 | `src/backend/access/heap/heapam.c` | heap insert/update/delete、heap scan fast path | 看 heap 修改如何清 VM，scan 如何用 `PD_ALL_VISIBLE` |
| 9 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM skip、prune、FSM update、VM counters | 看 VACUUM 同时消费和生产 FSM/VM |
| 10 | `src/backend/access/heap/pruneheap.c` | `HEAP_PAGE_PRUNE_SET_VM` 和 `visibilitymap_set()` | VM set 的执行层在这里 |
| 11 | `src/backend/executor/nodeIndexonlyscan.c` | `VM_ALL_VISIBLE()` 和 heap fetch fallback | 看 VM 如何影响用户查询 |
| 12 | `src/backend/storage/freespace/indexfsm.c` | index page free/used FSM | 看相同 FSM 结构如何服务 index AM |
不要从 `heapam.c` 顶部顺序读。
这节课要抓住四个入口：
```text
RelationGetBufferForTuple()
visibilitymap_set()/visibilitymap_clear()
lazy_scan_heap()/lazy_scan_prune()
IndexOnlyNext()
```
`freespace.c` 和 `visibilitymap.c` 是 fork-level API。
`hio.c`、`heapam.c`、`vacuumlazy.c`、`nodeIndexonlyscan.c` 是调用者。
真正的边界不在文件名，而在调用者拿到候选状态后怎么验证。
## 4. 关键状态与边界
先区分三种状态。
第一种是 heap main fork 的真实状态。
它包括：
```text
PageHeaderData.pd_lower/pd_upper/pd_flags/pd_prune_xid
line pointer array
tuple header xmin/xmax/infomask/t_ctid
```
插入是否能落页，最终看 `PageGetHeapFreeSpace(page)`。
page 是否全局可见，最终要看每个 tuple 的 MVCC 事实和 horizon。
第二种是 FSM fork。
它是每个 relation 一个独立 fork。
`freespace.c:16-20` 说明它用于快速找到 relation 中有足够空间的 page。
FSM 不保存 exact bytes。
`freespace.c:36-67` 把可用空间压缩成 256 个 category。
默认 8KB page 下，一个 category 大约表示 32 bytes 的粒度。
`fsm_space_avail_to_cat()` 向下取整。
`fsm_space_needed_to_cat()` 向上取整。
这说明：
记录空间和请求空间不是同一个映射方向。
记录要保守压缩。
请求要避免把不够的 category 当够用。
第三种是 VM fork。
它也是每个 heap relation 一个独立 fork。
`visibilitymapdefs.h:16-23` 定义每个 heap block 两个 bit：
```text
VISIBILITYMAP_ALL_VISIBLE = 0x01
VISIBILITYMAP_ALL_FROZEN  = 0x02
VISIBILITYMAP_VALID_BITS  = 0x03
```
`visibilitymap.c:25-31` 给出语义：
`all-visible` 表示这个 page 上所有 tuple 对所有事务可见。
`all-frozen` 表示这个 page 上所有 tuple 都已 frozen。
`all-frozen` 必须只在 page 已经 all-visible 时设置。
所以 VM 的合法状态不是四个任意组合。
实际语义是：
```text
00: 不知道，或不满足
01: all-visible，但不一定 all-frozen
11: all-visible 且 all-frozen
10: 非法语义，源码用 assert 避免
```
`PD_ALL_VISIBLE` 不是 VM bit。
它在 heap page header 里。
VM bit 在 visibility map fork 里。
这两者必须协同维护。
`visibilitymap.c:50-63` 强调：
清 VM 时，redo 必须能清掉 VM。
设置 VM 时，redo 必须能设置 heap page 上的 `PD_ALL_VISIBLE`。
否则后续 heap 修改可能不知道需要清 VM。
这就是 VM 和普通 hint bit 的区别。
## 5. FSM 的空间近似模型
FSM 的核心不是“记录 free space”。
核心是：
```text
用很小的、可滞后的树状索引，把找 page 的成本从扫描 heap 降为搜索 FSM fork。
```
`src/backend/storage/freespace/README:13-19` 说得很直接：
FSM 为了保持小，不记录 exact free space。
每个 heap page 用一个 byte。
`freespace.c:64-67` 定义：
```text
FSM_CATEGORIES = 256
FSM_CAT_STEP = BLCKSZ / 256
MaxFSMRequestSize = MaxHeapTupleSize
```
最高 category 255 代表至少 `MaxFSMRequestSize`。
普通 category 代表一个区间的下界。
`fsm_space_avail_to_cat()` 在 `freespace.c:401-420` 向下取整。
`fsm_space_cat_to_avail()` 在 `freespace.c:423-435` 返回该 category 的下界。
`fsm_space_needed_to_cat()` 在 `freespace.c:437-459` 向上取整。
这组函数决定了 FSM 的第一个边界：
`FSM` 不承诺 exact bytes。
它只承诺 category-level 近似。
第二个边界是树状传播。
`fsmpage.c:57-113` 的 `fsm_set_avail()` 先更新 leaf node，再把父节点更新成两个 child 的 max。
`fsmpage.c:145-315` 的 `fsm_search_avail()` 先看 page 内 root node。
root 不够，整个 FSM page 都不够。
root 够，再从 `fp_next_slot` 附近找满足条件的 leaf。
`fsm_internals.h:24-43` 定义 `FSMPageData`：
```text
fp_next_slot
fp_nodes[]
```
`fp_next_slot` 是 hint。
`fsm_internals.h:27-33` 说明它用于让多个 backend 不总是拿到同一个 page。
`fsmpage.c:295-312` 明确接受在 shared lock 下更新 `fp_next_slot`。
这个字段错了也只是影响分布，不影响 correctness。
第三个边界是跨 page tree。
`freespace.c:68-78` 定义固定深度 FSM tree。
默认 8KB 下通常三层足够覆盖 2^32-1 个 heap block。
`freespace.c:80-98` 用 `FSMAddress` 表示 logical level 和 page number。
`fsm_logical_to_physical()` 在 `freespace.c:461-495` 把 logical address 映射到 FSM fork block。
读这一段时不要把 FSM 想成一个普通数组。
它是：
```text
heap block -> bottom-level FSM slot
bottom-level page root -> upper-level FSM slot
...
root-level page root -> whole relation 最大可用空间
```
第四个边界是更新滞后。
`RecordPageWithFreeSpace()` 在 `freespace.c:187-204` 更新某个 heap block 对应的 bottom-level slot。
注释 `freespace.c:189-191` 说明：
如果新记录的空间比旧值更高，这个空间可能要等下一次 `FreeSpaceMapVacuum()` 才能被上层 page 搜到。
所以更新 bottom-level 并不意味着全树立即完全准确。
这就是 VACUUM 后还要调用 `FreeSpaceMapVacuumRange()` 的原因。
## 6. FSM 搜索与插入页选择
heap insert 不直接遍历 FSM。
它通过 `RelationGetBufferForTuple()`。
入口在 `hio.c:435-499`。
这个函数返回：
```text
一个 pinned 且 exclusive-locked 的 heap buffer
这个 page 当前确实有足够空间容纳 tuple
```
注意这里的承诺来自 heap page recheck，不来自 FSM。
FSM 只提供候选 block。
`hio.c:558-570` 给出主循环的核心注释：
先尝试 relcache 或 bulk insert state 里的 target page。
失败后问 FSM。
FSM 信息可能过期，所以要准备多次 retry。
如果 FSM 没有记录，就尝试最后一页。
再不行就扩展 relation。
伪调用链是：
```text
heap_insert()
  -> heap_prepare_insert()
  -> RelationGetBufferForTuple()
     -> RelationGetTargetBlock()
     -> GetPageWithFreeSpace()
     -> ReadBufferBI()/LockBuffer(EXCLUSIVE)
     -> PageGetHeapFreeSpace()
     -> RecordAndGetPageWithFreeSpace() on miss
     -> RelationAddBlocks() on no candidate
  -> RelationPutHeapTuple()
```
`GetPageWithFreeSpace()` 在 `freespace.c:121-142` 只返回一个 block number。
注释 `freespace.c:126-131` 已经提醒调用者：
返回的 page 可能到锁住时已经没有空间。
调用者必须报告实际空间，再 retry。
这个 retry 就是 `RecordAndGetPageWithFreeSpace()`。
`RecordAndGetPageWithFreeSpace()` 在 `freespace.c:145-184` 先更新 old page 的实际 category。
然后优先在同一个 bottom-level FSM page 上继续找接近的候选。
找不到再从 root 搜。
`hio.c:700-706` 是边界点：
只有在 `PageGetHeapFreeSpace(page)` 满足 `targetFreeSpace` 时，才把这个 page 作为未来 target block 返回给上层。
`hio.c:708-762` 是 stale FSM 的正常路径：
page 空间不足。
释放 heap lock/pin。
用 `RecordPageWithFreeSpace()` 或 `RecordAndGetPageWithFreeSpace()` 修正 FSM。
换下一个候选。
这说明 `FSM` 的 false positive 不会污染 heap。
它只增加 retry 成本。
如果没有候选，`hio.c:765-883` 扩展 relation。
扩展多个 page 时，`hio.c:308-321` 会决定哪些新 page 进入 FSM。
`hio.c:391-405` 对未立即使用的扩展页调用 `RecordPageWithFreeSpace()` 和 `FreeSpaceMapVacuumRange()`。
被返回给当前 backend 马上使用的 page 通常不立即进入 FSM。
`hio.c:872-880` 的注释说：
当前 backend 很可能继续插入，先把 page 留给本 backend 的 target block 更划算。
这也是一个 workload-dependent 的取舍。
## 7. VM 的可见性事实模型
VM 的核心不是空间。
它回答两个问题：
```text
这个 heap page 是否所有 tuple 对所有事务可见？
这个 heap page 是否所有 tuple 都已经 frozen？
```
`visibilitymap.c:33-35` 给出 conservative 原则：
bit set 时，条件必须为真。
bit clear 时，条件可能真，也可能假。
这决定了 VM 消费者的 fallback。
index-only scan 看到 bit clear，要访问 heap。
VACUUM 看到 all-visible clear，要扫描 page。
VACUUM 看到 all-frozen clear，aggressive vacuum 可能仍要扫描冻结。
但 bit set 不能错。
`visibilitymap.c:43-48` 明确说：
如果 bit 错误设置，可能造成 data corruption 和 wrong results。
`visibilitymap_clear()` 在 `visibilitymap.c:144-185` 清指定 bit。
它要求传入正确 VM page buffer。
`visibilitymap.c:160-162` 有两个 assert：
必须清合法 bit。
不能只清 `all-visible` 而留下 `all-frozen`。
所以通常 heap 修改传 `VISIBILITYMAP_VALID_BITS`，一次清掉两个 bit。
`visibilitymap_set()` 在 `visibilitymap.c:235-297` 设置 bit。
它要求：
VM buffer 已 pin 且 exclusive-locked。
调用在 recovery 中，或在 critical section 中。
不能只设置 `all-frozen`。
这对应 `visibilitymap.c:273-280` 的 assert。
`visibilitymap_get_status()` 在 `visibilitymap.c:300-357` 不锁 VM page。
注释 `visibilitymap.c:311-316` 说：
调用者自己处理并发问题。
这点对 index-only scan 非常重要。
VM page 大小由 `visibilitymap.c:115-125` 定义。
每个 heap block 两个 bit。
一个 VM page 可以覆盖很多 heap block。
这就是为什么 index-only scan 只需少量 VM page IO，就能跳过大量 heap page。
但同样因为一个 VM page 覆盖大量 heap block，VM page 锁竞争和 cache locality 也会影响高并发写入路径。
## 8. heap insert 如何同时使用 FSM 和 VM
`heap_insert()` 在 `heapam.c:1985-2003` 说明会设置 tuple header，并把 `t_self` 更新成实际 TID。
真正找 page 的调用在 `heapam.c:2028-2035`：
```text
RelationGetBufferForTuple(..., &vmbuffer, ...)
```
这行同时连接了 FSM 和 VM。
FSM 用来找候选 page。
如果候选 page 当前 `PD_ALL_VISIBLE`，插入必须清 VM。
但是读 VM page 可能产生 IO。
PostgreSQL 不希望持 heap buffer content lock 做 VM IO。
所以 `hio.c:606-612` 的流程是：
锁 heap page 前先看 `PageIsAllVisible(page)`。
如果看起来需要清 VM，就先 `visibilitymap_pin()`。
然后再锁 heap page。
但这个预检查可能过期。
`hio.c:657-680` 调用 `GetVisibilityMapPins()` 重新检查。
如果锁住 page 后发现刚刚被 VACUUM 设置成 all-visible，而我们没有 pin 正确 VM page，就必须释放 heap lock，pin VM page，再重新锁。
`hio.c:128-209` 的 `GetVisibilityMapPins()` 就是在做这个 retry。
这段代码体现一个重要边界：
```text
VM page pin 可以在 heap page lock 前做。
真正清 VM 必须在持 heap page lock 的 critical section 中做。
```
`heap_insert()` 真正修改 page 的 critical section 在 `heapam.c:2056-2167`。
步骤是：
```text
START_CRIT_SECTION()
  RelationPutHeapTuple()
  if PageIsAllVisible(page):
     PageClearAllVisible(page)
     visibilitymap_clear(..., VISIBILITYMAP_VALID_BITS)
  PageSetPrunable(page, xid)
  MarkBufferDirty(buffer)
  XLogInsert()
  PageSetLSN(page, recptr)
END_CRIT_SECTION()
```
这里清的不只是 `all-visible`。
`visibilitymap_clear()` 传 `VISIBILITYMAP_VALID_BITS`。
原因是：
插入一个普通未 frozen tuple 后，page 既不是 all-visible，也不是 all-frozen。
`heapam.c:2117-2120` 把 `XLH_INSERT_ALL_VISIBLE_CLEARED` 写进 WAL record。
redo 时 `heapam_xlog.c:393-405` 即使 heap page 已经因为 LSN interlock 跳过 redo，也仍会清 VM。
这是 VM correctness 的关键。
`heap_multi_insert()` 还有一个特殊分支。
如果 `COPY FREEZE` 或类似路径向空 page 插入 frozen tuple，`heapam.c:2431-2456` 可以设置 `PD_ALL_VISIBLE`，并把 VM 设置成 all-visible/all-frozen。
对应 WAL flag 是 `XLH_INSERT_ALL_FROZEN_SET`，见 `heapam.c:2502-2515`。
redo 在 `heapam_xlog.c:657-674` 重新设置 VM。
这不是普通 insert 路径。
普通 insert 会清 VM，并设置 `pd_prune_xid`，期待后续 pruning/VACUUM 再把 page 标成 all-visible。
## 9. heap update/delete 与 VM 清除
DELETE 和 UPDATE 不一定立刻释放空间。
它们主要改变旧 tuple header。
但它们一定会让原来 all-visible 的 page 不再 all-visible。
因为旧 tuple 可能被当前事务删除或更新，对某些 snapshot 的可见性需要重新判断。
`heap_delete()` 在 `heapam.c:2758-2765` 锁 heap page 前预先 pin VM page。
如果没 pin 到而锁住后发现 page 变成 all-visible，`heapam.c:2779-2790` 会 unlock、pin、relock。
真正清 VM 在 `heapam.c:2986-3003`：
```text
PageSetPrunable(page, xid)
if PageIsAllVisible(page):
    PageClearAllVisible(page)
    visibilitymap_clear(..., VISIBILITYMAP_VALID_BITS)
```
随后 WAL record 记录 `XLH_DELETE_ALL_VISIBLE_CLEARED`。
redo 在 `heapam_xlog.c:306-319` 先修 VM。
如果 heap page 需要 redo，`heapam_xlog.c:347-355` 也清 `PD_ALL_VISIBLE` 并更新 tuple header。
UPDATE 的边界更复杂。
它可能在同 page 插入新 tuple，也可能通过 `RelationGetBufferForTuple()` 找新 page。
`heapam.c:3880-3911` 解释为什么更新跨 page 时要让 `RelationGetBufferForTuple()` 同时锁 old/new buffer。
否则两个 backend 可能以相反顺序锁两个 page，造成 deadlock。
这也说明 FSM 和 VM 不只是空间模块。
它们参与 buffer lock ordering。
`heapam.c:4012-4108` 是 UPDATE critical section。
它先 `RelationPutHeapTuple()` 插入新版本。
再把旧版本 `xmax` 和 `t_ctid` 指向新 tuple。
如果 old page 或 new page 是 all-visible，就清对应 `PD_ALL_VISIBLE` 和 VM bit。
`log_heap_update()` 会记录 old/new page 是否清过 VM。
redo 在 `heapam_xlog.c:742-839` 分别清 old/new VM。
同 page HOT update 也要清 VM。
HOT 只减少 index 维护，不改变 MVCC 可见性事实需要重新判断。
## 10. VACUUM 如何消费 VM
VACUUM 既消费 VM，也生产 VM。
消费 VM 的入口在 `vacuumlazy.c:1734-1840`。
`find_next_unskippable_block()` 调用 `visibilitymap_get_status()`。
它不锁 heap page。
它的目标只是决定下一段扫描能不能跳过。
基本规则是：
```text
VM all-visible clear: page 不可跳过
最后一页: 不跳过，用于判断 relation tail 是否可截断
DISABLE_PAGE_SKIPPING: 不跳过
all-frozen set: 可以跳过
aggressive VACUUM: all-visible 但未 all-frozen 的 page 不可跳过
normal VACUUM: all-visible page 通常可跳过，但可能 eager scan 以尝试冻结
```
`vacuumlazy.c:1739-1745` 承认 VM 判断会马上变 stale。
但只要所有可能包含旧 XID/MXID 的 page 被扫描，就安全。
`vacuumlazy.c:1804-1816` 是 all-frozen 与 aggressive vacuum 的关键。
all-frozen page 不含需要冻结的旧 XID。
aggressive vacuum 不能跳过 all-visible 但未 all-frozen 的 page。
这就是两个 bit 分开的原因。
如果只有 all-visible bit，反 wraparound vacuum 仍然不知道 page 是否还需要 freeze。
`vacuumlazy.c:936-945` 还有一个容易忽略的边界。
normal VACUUM 如果跳过了 all-visible range，就不能用这次扫描推进 `relfrozenxid`/`relminmxid`。
因为它没有看见那些 page 上的 unfrozen XID。
所以 `skippedallvis` 会让新 frozen horizon 失效。
VM 可以让 VACUUM 少读 heap。
但它不能让 VACUUM 假装已经检查过未扫描的 tuple。
## 11. VACUUM 如何生产 VM 和 FSM
VACUUM 扫到 page 时会尝试 prune、freeze，并决定是否能设置 VM。
`vacuumlazy.c:1408-1413` 在处理 heap page 前 pin VM page。
`vacuumlazy.c:1415-1424` 先尝试 cleanup lock。
拿不到 cleanup lock 时，可能走 `lazy_scan_noprune()` 的降级路径。
如果 aggressive vacuum 发现必须处理 freeze，则会等待 cleanup lock。
`lazy_scan_prune()` 在 `vacuumlazy.c:2003-2135` 调用：
```text
heap_page_prune_and_freeze()
```
传入 options：
```text
HEAP_PAGE_PRUNE_FREEZE
HEAP_PAGE_PRUNE_SET_VM
```
如果 relation 没有 index，还加 `HEAP_PAGE_PRUNE_MARK_UNUSED_NOW`。
真正设置 VM 的代码在 `pruneheap.c:1260-1305`。
当 `do_set_vm` 为真：
```text
PageSetAllVisible(page)
PageClearPrunable(page)
visibilitymap_set(block, vmbuffer, new_vmbits, locator)
MarkBufferDirty(buffer)
log_heap_prune_and_freeze(...)
```
`pruneheap.c:480-490` 说明剪枝冻结过程中会追踪 page 是否最终 all-visible。
`pruneheap.c:493-518` 说明 all-frozen 只有在 caller 同时允许 freeze 和 set VM 时才可能设置。
`heap_page_would_be_all_visible()` 在 `vacuumlazy.c:3571-3752` 是另一个重要辅助。
它用来判断移除已知 dead line pointer 后 page 是否将变成 all-visible。
它不会在 critical section 中运行。
因为可见性检查可能做 IO 或分配内存。
这符合课程标准里的重要边界：
```text
复杂判断在 critical section 外完成。
真正 page mutation、VM set、WAL record 在 critical section 内完成。
```
VACUUM 对 FSM 的更新更宽松。
`vacuumlazy.c:1527-1554` 说明：
目标是在最后一次触碰 page 时更新 FSM。
如果还有第二遍 heap pass 要把 `LP_DEAD` 变 `LP_UNUSED`，先不更新。
如果没有 index，或者不需要第二遍，就现在记录 `PageGetHeapFreeSpace(page)`。
注释 `vacuumlazy.c:1538-1544` 还说：
某些 corner case 可能完全错过 FSM 更新。
这是可以接受的。
因为 FSM 是 desirable but not absolutely required。
第二遍 heap pass 在 `vacuumlazy.c:2660-2723`。
它对已经从 index 删除的 `LP_DEAD` 调用 `lazy_vacuum_heap_page()`。
之后计算 `PageGetHeapFreeSpace(page)`，再 `RecordPageWithFreeSpace()`。
这时释放出来的空间才真正进入 FSM。
`vacuumlazy.c:1374-1380` 和 `1555-1566` 周期性调用 `FreeSpaceMapVacuumRange()`。
作用是把底层 page 的新 free space 传播到上层 FSM tree。
所以 VACUUM 的输出不是一个动作。
它同时推进：
```text
heap page pruning/freezing
VM all-visible/all-frozen bits
FSM recorded free space
pg_class relallvisible/relallfrozen 统计
```
这些推进之间允许短暂不一致。
但每个消费者都有对应的 fallback。
## 12. heap scan 与 index-only scan 如何使用 VM
heap scan 不直接读 VM bit。
`heap_prepare_pagescan()` 在 `heapam.c:611-701` 使用 page-level `PD_ALL_VISIBLE`。
如果 page all-visible 且 snapshot 不是 recovery snapshot，`page_collect_tuples()` 可以跳过逐 tuple MVCC 检查。
`heapam.c:651-671` 的注释解释了 hot standby 差异。
index-only scan 完全依赖 VM。
入口在 `nodeIndexonlyscan.c:61-255`。
主循环拿到 index TID 后执行：
```text
if !VM_ALL_VISIBLE(heapRelation, block, &ioss_VMBuffer):
    index_fetch_heap()
else:
    use index tuple as data source
```
这对应 `nodeIndexonlyscan.c:130-173`。
如果 VM all-visible bit clear，执行器必须访问 heap 来做 visibility check。
`EXPLAIN (ANALYZE)` 里的 `Heap Fetches` 来自 `nodeIndexonlyscan.c:171` 计数。
输出位置在 `explain.c:1995-1997`。
`nodeIndexonlyscan.c:135-162` 有一段非常重要的 memory ordering 注释。
VM read 不锁 VM buffer。
为什么不会错？
插入必须先清 VM，再更新 index page。
index-only scan 如果看到了新 index TID，就通过 index page lock/unlock 的 barrier 看到对应 VM clear。
删除不更新 index page。
但删除后 tuple 对当前 statement/snapshot 的不可见性变化有事务提交和 ProcArrayLock 的顺序约束。
这段逻辑说明：
VM 的 correctness 不是 VM lock 单独保证的。
它依赖 heap 修改顺序、index page lock、snapshot 获取顺序和 WAL/redo 共同保证。
## 13. index FSM 的边界
`indexfsm.c` 很短，但能帮助理解 FSM 是通用空间索引。
`indexfsm.c:16-20` 说明：
index FSM 不记录 page 剩余 bytes。
它只记录 page 是完全空闲还是已使用。
用同一套 heap FSM 实现：
```text
used page: 0
unused page: BLCKSZ - 1
```
`GetFreeIndexPage()` 在 `indexfsm.c:32-45` 调用：
```text
GetPageWithFreeSpace(rel, BLCKSZ / 2)
RecordUsedIndexPage(rel, blkno)
```
它拿到 free page 后立即在 FSM 里标 used。
`RecordFreeIndexPage()` 在 `indexfsm.c:49-55` 把 page 标成 `BLCKSZ - 1`。
`RecordUsedIndexPage()` 在 `indexfsm.c:58-65` 把 page 标成 0。
这说明 index FSM 和 heap FSM 共享的是“可搜索的 page 状态树”。
但上层语义不同。
heap 用它近似 free bytes。
index AM 可以用它管理整页回收。
不要把 `GetPageWithFreeSpace()` 名字理解成只服务 heap。
## 14. 生命周期、ownership 与 cleanup
FSM 和 VM 都是 relation fork。
它们不是 backend-local cache。
也不是 static shared memory。
它们的生命周期跟 relation storage 绑定。
创建：
FSM fork 在需要记录或搜索时通过 `fsm_readbuf(..., extend=true)` 扩展。
`freespace.c:563-607` 读不存在的 FSM page 时，如果 `extend` 为真就 `fsm_extend()`。
VM fork 在需要设置 bit 时通过 `visibilitymap_pin()` 触发 `vm_readbuf(..., extend=true)`。
`visibilitymap.c:188-217` 说明 pin 时如果 VM page 不存在会扩展。
持有：
前台 backend 通过 buffer pin 持有 FSM/VM page。
修改时通过 buffer content lock 保护 page contents。
VM set 还要求 heap page pin 和 exclusive lock。
FSM search 多数使用 shared buffer lock。
FSM update 使用 exclusive lock。
释放：
正常路径用 `ReleaseBuffer()` 或 `UnlockReleaseBuffer()`。
例如 `heap_insert()` 在 `heapam.c:2169-2171` 释放 heap buffer 和 vmbuffer。
UPDATE 在 `heapam.c:4126-4133` 释放 old/new heap buffer 和 old/new vmbuffer。
ERROR/abort：
buffer pin 是 ResourceOwner 管的外部资源。
如果 ERROR 发生在普通路径，资源清理会释放 pin 和 lock。
但进入 critical section 后，代码必须避免 `ereport(ERROR)`。
因此 `heapam.c:2056` 和 `vacuumlazy.c:2809` 这种 critical section 前要完成可能分配内存或 IO 的判断。
如果 critical section 内出现不可恢复错误，PostgreSQL 会走更重的 PANIC 级别恢复，而不是普通 ERROR 返回。
失效与截断：
relation 截断时，FSM 通过 `FreeSpaceMapPrepareTruncateRel()` 清尾部 slot 并返回新 FSM 长度。
VM 通过 `visibilitymap_prepare_truncate()` 清尾部 bit 并返回新 VM 长度。
这些操作对应 `freespace.c:273-359` 和 `visibilitymap.c:410-511`。
如果主 fork 将来再次扩展，旧的 VM/FSM 尾部状态不能泄漏到新 heap block。
这就是清 tail bit/slot 的原因。
## 15. 正确性机制层次
本节最重要的判断是：
FSM 和 VM 的正确性层次不同。
FSM 的正确性边界：
```text
真实空间判断在 heap page exclusive lock 下完成。
FSM 命中失败只导致 retry 或扩表。
FSM false positive 不会写坏 page。
FSM false negative 不会返回错误 tuple。
```
VM 的正确性边界：
```text
VM set 必须只在 page 真正 all-visible/all-frozen 后发生。
VM clear 必须和 heap 修改在 WAL/redo 中绑定。
index-only scan 可以基于 VM bit 跳过 heap。
VACUUM 可以基于 VM bit 跳过 page 或推进 freeze 相关判断。
```
涉及的机制分层如下。
| 机制 | 在本节保证什么 | 不能保证什么 |
| --- | --- | --- |
| heap buffer lock | page bytes 修改互斥 | tuple 对 snapshot 可见 |
| buffer pin | page 不被 eviction/reuse | FSM/VM 语义仍准确 |
| FSM category | 快速找到候选 page | page 当前一定有空间 |
| VM all-visible | 可跳过 heap visibility check | page 未来不会被修改 |
| VM all-frozen | 可跳过 freeze 需要 | relation 永远不需要 VACUUM |
| WAL redo | crash 后恢复 VM/heap 顺序 | 前台不会遇到 stale FSM |
| ResourceOwner | ERROR 时释放 pin/lock | critical section 内可随意 ERROR |
| ProcArray/snapshot | 判断 tuple 是否对 snapshot 可见 | 互斥 heap page 修改 |
因此不要说“VM 保证可见性”。
更准确的说法是：
VM 保存一个由 heap/VACUUM/WAL 共同维护的 page-level 可见性事实，消费者在看到 bit set 时可以省掉更贵的 heap 检查。
## 16. WAL、crash 与 redo 协作
FSM 不是普通 WAL-logged 数据结构。
`src/backend/storage/freespace/README:168-199` 说明：
FSM 不显式 WAL-log。
它依赖 self-correcting 措施。
`freespace.c:587-592` 读 FSM page 用 `RBM_ZERO_ON_ERROR`。
因为 FSM 信息本来就是近似的，遇到 checksum mismatch 或 torn page，清掉比报错更合适。
`fsmpage.c:263-288` 在 search 时如果 parent 说 child 有空间，但 child 都不够，会 rebuild page。
`fsmpage.c:346-380` 的 `fsm_rebuild_page()` 从 leaf 重新计算 upper nodes。
`freespace.c:740-755` 如果 FSM 指向 relation 尾部之外的 heap block，会把 slot 置 0 并从 root 重搜。
`freespace.c:935-955` 的 `fsm_does_block_exist()` 解释了这种情况：
WAL replay 后，FSM 可能已经落盘，但新扩展的 main fork page 没有。
这时不能返回不存在的 heap block。
`freespace.c:791-799` 还有 emergency valve。
如果 upper pages 太旧导致搜索反复重启超过阈值，返回 `InvalidBlockNumber`，上层扩展 relation。
这是 correctness-first 的 fallback。
VM 的 WAL 语义更严格。
`visibilitymap.c:37-41` 说明 VM bit 改变不单独 WAL-log。
调用者必须让 heap 操作的 WAL redo 同步 set/clear VM。
清 VM 的例子：
普通 insert 清 all-visible page 时，`heapam.c:2117-2120` 记录 `XLH_INSERT_ALL_VISIBLE_CLEARED`。
redo 在 `heapam_xlog.c:393-405` 先 pin 并清 VM。
DELETE redo 在 `heapam_xlog.c:306-319` 清 VM。
UPDATE redo 在 `heapam_xlog.c:742-839` 分别清 old/new page 的 VM。
设置 VM 的例子：
VACUUM/prune 在 `pruneheap.c:1273-1305` 设置 `PD_ALL_VISIBLE` 和 VM，并通过 `log_heap_prune_and_freeze()` 记录。
redo 在 `heapam_xlog.c:228-249` 即使 heap page 已经 up-to-date，也要 redo VM。
`heapam_xlog.c:635-655` 对 multi insert all-frozen 解释了为什么：
VM page 覆盖多个 heap page。
后续 WAL 可能依赖这个 VM page 的 FPI 来防 torn page。
heap page 的 LSN interlock 不能代替 VM page redo。
VM set 时还要设置 VM page LSN。
`visibilitymap.c:90-94` 说明：
set bit 时更新 VM page LSN，避免 VM page 在使其成立的 WAL flush 前落盘。
clear bit 从 correctness 上总是安全的，所以不需要同样的 LSN 约束。
但清 bit 必须在 redo 中发生。
否则 crash 后 VM 可能错误保留 all-visible。
## 17. 错误路径、异常路径与 fallback
异常路径一：FSM 候选页实际空间不足。
路径是：
```text
GetPageWithFreeSpace()
  -> 返回 candidate block
RelationGetBufferForTuple()
  -> 锁 heap page
  -> PageGetHeapFreeSpace() 不足
  -> RecordAndGetPageWithFreeSpace()
  -> retry
```
调用者看到的只是插入慢一点。
不会写坏 page。
异常路径二：FSM 过于悲观。
如果 FSM 没记录某些 page 的真实空间，`GetPageWithFreeSpace()` 返回 `InvalidBlockNumber`。
`hio.c:586-596` 会先尝试 relation 最后一页。
还不行就扩展 relation。
这可能造成 bloat，但仍然正确。
异常路径三：FSM page corrupt 或 torn。
`freespace.c:587-592` 选择 `RBM_ZERO_ON_ERROR`。
`fsmpage.c:263-288` 搜索时发现树内部不一致，会 `fsm_rebuild_page()`。
`FreeSpaceMapVacuumRange()` 后续也会把 bottom-level 信息重新传播。
异常路径四：VM page 不存在或 corrupt。
`visibilitymap_get_status()` 在 `visibilitymap.c:341-345` 读不到 VM page 就返回 0。
这意味着“不要相信 VM，回到 heap”。
`vm_readbuf()` 在 `visibilitymap.c:560-563` 也用 zero-on-error。
清 VM 是安全保守 fallback。
异常路径五：锁住 heap page 后发现 VM pin 不够。
`GetVisibilityMapPins()` 会释放 heap locks，做 `visibilitymap_pin()`，再按顺序重新锁。
DELETE/UPDATE 也有同样 unlock/relock 分支。
这是为了避免持 heap content lock 做 VM fork IO。
异常路径六：VACUUM 拿不到 cleanup lock。
`vacuumlazy.c:1415-1453` 可以先走 share lock 和 `lazy_scan_noprune()`。
如果 aggressive vacuum 必须推进 freeze，就等待 cleanup lock 再 prune。
这说明 VM/FSM 不能绕开 heap page cleanup lock 的 correctness 边界。
异常路径七：index-only scan 遇到 VM bit clear。
`nodeIndexonlyscan.c:164-173` 调用 `index_fetch_heap()`。
这就是用户能看到的 `Heap Fetches`。
它不是 index-only scan “失败”。
它是 VM 保守状态下的正确 fallback。
## 18. 成本、资源与跨模块传播
FSM 的成本主要随 relation page 数扩张。
搜索成本是 FSM tree 高度和每个 FSM page 内 tree search。
默认 BLCKSZ 下高度很小。
真正的成本常常来自 stale candidate 导致多次 heap page read/lock/retry。
高并发 insert 下，如果很多 backend 被同一个 stale FSM range 吸引，会在 heap page 和 FSM page 上形成额外竞争。
`fp_next_slot` 的目的就是分散这种热点。
VM 的成本主要随 heap page 修改和 VACUUM/index-only scan 访问模式扩张。
写路径如果修改 all-visible page，要额外 pin VM page。
如果锁后发现 pin 不够，还要 unlock/relock。
这是一条 rare but painful path。
读路径如果 VM bit set，可以少读 heap page。
如果 bit clear，index-only scan 要 heap fetch。
所以 VM 的收益高度依赖 workload：
只读或 append 后 VACUUM 充分的表，index-only scan 受益大。
频繁 update/delete 的表，VM bits 经常被清，Heap Fetches 会高。
VACUUM 的成本传播更复杂：
VM all-visible page 可跳过，减少 heap IO。
all-visible 但未 all-frozen 的 page 在 aggressive VACUUM 中仍要扫描。
FSM 更新本身便宜，但 `FreeSpaceMapVacuumRange()` 让 freed space 在 upper FSM pages 可搜索。
WAL 成本也被 VM 影响。
设置 VM 常与 prune/freeze WAL record 绑定。
清 VM 通过 heap insert/update/delete WAL flag 绑定。
FSM 通常不显式 WAL-log，但 hint FPI 和 redo helper 会影响恢复后空间提示质量。
planner 也消费 VM 的统计投影。
`pg_class.relallvisible` 由 VACUUM/ANALYZE 维护。
`plancat.c` 里 `allvisfrac = relallvisible / curpages`。
`costsize.c` 用 `1.0 - allvisfrac` 折减 index-only scan 预计 heap page fetch。
所以 VM 影响三层：
```text
executor runtime 是否 heap fetch
VACUUM 是否扫描 page
planner 是否偏向 index-only scan
```
## 19. 观测与诊断入口
能直接看见的状态：
```text
pg_freespacemap.pg_freespace()
pg_visibility.pg_visibility_map()
pg_visibility.pg_visibility()
pg_visibility.pg_visibility_map_summary()
pg_class.relallvisible
pg_class.relallfrozen
EXPLAIN (ANALYZE, BUFFERS) 的 Heap Fetches
VACUUM (VERBOSE) 的 visibility map 统计
pg_stat_progress_vacuum.heap_blks_scanned / heap_blks_vacuumed
```
只能间接推断的状态：
```text
FSM upper-level page 是否滞后
RelationGetBufferForTuple() retry 次数
GetVisibilityMapPins() 是否发生 unlock/relock
index-only scan 读到的 VM byte 是否 stale
```
基本不可见或需要断点的状态：
```text
fsm_search() restarts
fp_next_slot 的并发扰动
visibilitymap_get_status() 的无锁读竞争
heap_page_would_be_all_visible() 中 newest_live_xid
pruneheap.c 中 do_set_vm / new_vmbits
```
推荐断点：
```text
break RelationGetBufferForTuple
break GetPageWithFreeSpace
break RecordAndGetPageWithFreeSpace
break visibilitymap_clear
break visibilitymap_set
break lazy_scan_prune
break heap_page_would_be_all_visible
break IndexOnlyNext
```
推荐 SQL 观测入口：
```sql
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;
CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT * FROM pg_visibility_map_summary('your_table'::regclass);
SELECT * FROM pg_visibility('your_table'::regclass) LIMIT 20;
SELECT * FROM pg_freespace('your_table'::regclass) ORDER BY blkno LIMIT 20;
SELECT relpages, relallvisible, relallfrozen
FROM pg_class
WHERE oid = 'your_table'::regclass;
```
诊断时不要把 `pg_freespace()` 当真实 heap free space。
它展示的是 FSM recorded space。
也不要把 `relallvisible` 当实时 VM count。
它是 lazily maintained catalog statistic。
想看 VM 当前 bit，用 `pg_visibility_map_summary()`。
想看 VM 和 heap page flag 是否一致，用 `pg_visibility()` 的 `pd_all_visible`。
## 20. 常见误区
误区一：
`GetPageWithFreeSpace()` 找到 page，就一定能插入。
实际：
它只给候选 block。
真正判断在 heap page exclusive lock 下完成。
误区二：
FSM 不准就是数据损坏。
实际：
FSM 不准通常只造成 retry、提前扩表、空间复用变慢。
误区三：
VM 和 FSM 一样只是随便 hint。
实际：
VM bit clear 是 hint-like fallback。
VM bit set 是 index-only scan 和 VACUUM 会信任的 correctness-sensitive 辅助事实。
误区四：
`all-visible` 等于 `all-frozen`。
实际：
all-visible 只说明所有 tuple 对所有事务可见。
all-frozen 还说明 tuple 不再需要 XID/MXID freeze。
误区五：
index-only scan 没有 heap read。
实际：
VM bit clear 时必须 heap fetch。
`Heap Fetches` 就是这个 fallback 的观测点。
误区六：
VACUUM 跳过 all-visible page 后仍可安全推进 relfrozenxid。
实际：
normal VACUUM 如果跳过 all-visible 但未 all-frozen page range，不能用这次扫描结果推进 frozen horizon。
误区七：
`PD_ALL_VISIBLE` 和 VM all-visible 是同一个 bit。
实际：
前者在 heap page header，后者在 VM fork。
它们必须协同，但物理位置和消费者不同。
误区八：
清 VM 很贵，所以可以延迟。
实际：
heap 修改 all-visible page 时必须在同一 WAL/critical section 边界清 VM。
延迟清 VM 可能让 index-only scan 返回错结果。
## 21. 课堂实验一：FSM 是近似空间记录
目标：
观察 FSM recorded space 和 heap 真实空间回收之间的滞后。
准备：
```sql
CREATE EXTENSION IF NOT EXISTS pg_freespacemap;
DROP TABLE IF EXISTS fsm_vm_lab;
CREATE TABLE fsm_vm_lab (
    id int PRIMARY KEY,
    payload text
) WITH (fillfactor = 80, autovacuum_enabled = off);

INSERT INTO fsm_vm_lab
SELECT i, repeat('x', 200)
FROM generate_series(1, 20000) AS g(i);

VACUUM (ANALYZE, VERBOSE) fsm_vm_lab;
```
看 FSM：
```sql
SELECT blkno, avail
FROM pg_freespace('fsm_vm_lab'::regclass)
ORDER BY avail DESC, blkno
LIMIT 10;
```
制造 dead tuple：
```sql
UPDATE fsm_vm_lab
SET payload = repeat('y', 200)
WHERE id BETWEEN 1 AND 5000;
```
再次看 FSM：
```sql
SELECT blkno, avail
FROM pg_freespace('fsm_vm_lab'::regclass)
ORDER BY avail DESC, blkno
LIMIT 10;
```
你可能不会立刻看到大量可复用空间。
原因是 UPDATE 先制造旧 tuple version。
空间要等 pruning/VACUUM 之后才真正进入 page free space。
执行：
```sql
VACUUM (VERBOSE, ANALYZE) fsm_vm_lab;

SELECT blkno, avail
FROM pg_freespace('fsm_vm_lab'::regclass)
ORDER BY avail DESC, blkno
LIMIT 10;
```
回到源码解释：
`heap_update()` 修改 tuple header 并设置 `pd_prune_xid`。
`lazy_scan_prune()`/第二遍 heap pass 把 dead line pointer 处理掉。
`RecordPageWithFreeSpace()` 把 `PageGetHeapFreeSpace()` 写进 FSM。
`FreeSpaceMapVacuumRange()` 把 bottom-level free space 推到 upper FSM pages。
实验边界：
`pg_freespace()` 看到的是 FSM recorded space。
它不是在读 heap page 真实 free bytes。
## 22. 课堂实验二：VM 决定 index-only scan 的 Heap Fetches
目标：
观察 VM all-visible bit 对 index-only scan 的影响。
准备：
```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;
DROP TABLE IF EXISTS ios_vm_lab;
CREATE TABLE ios_vm_lab (
    id int PRIMARY KEY,
    payload text
) WITH (autovacuum_enabled = off);

INSERT INTO ios_vm_lab
SELECT i, repeat('a', 20)
FROM generate_series(1, 50000) AS g(i);

VACUUM (ANALYZE, VERBOSE) ios_vm_lab;
```
确认 VM：
```sql
SELECT * FROM pg_visibility_map_summary('ios_vm_lab'::regclass);
SELECT relpages, relallvisible, relallfrozen
FROM pg_class
WHERE oid = 'ios_vm_lab'::regclass;
```
强制观察 index-only scan：
```sql
SET enable_seqscan = off;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM ios_vm_lab
WHERE id BETWEEN 1000 AND 20000;
```
如果 VM all-visible 覆盖充分，`Heap Fetches` 应该很低，常见情况下为 0。
清掉部分 VM：
```sql
UPDATE ios_vm_lab
SET payload = repeat('b', 20)
WHERE id BETWEEN 1000 AND 1100;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM ios_vm_lab
WHERE id BETWEEN 1000 AND 20000;
```
现在 `Heap Fetches` 通常会上升。
原因是 UPDATE 清了受影响 heap page 的 VM bits。
index-only scan 看到 `VM_ALL_VISIBLE()` false 后，调用 `index_fetch_heap()`。
再次 VACUUM：
```sql
VACUUM (ANALYZE, VERBOSE) ios_vm_lab;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM ios_vm_lab
WHERE id BETWEEN 1000 AND 20000;
```
如果没有长事务阻止 all-visible，`Heap Fetches` 会下降。
回到源码解释：
`nodeIndexonlyscan.c:164-173` 是 fallback。
`heapam.c:4062-4075` 是 UPDATE 清 VM。
`pruneheap.c:1273-1291` 和 `vacuumlazy.c:2003-2135` 是 VACUUM 重新设置 VM。
## 23. 课堂实验三：长事务阻止 page 变 all-visible
目标：
观察 all-visible 依赖 global visibility horizon。
先准备表：
```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;
DROP TABLE IF EXISTS vm_horizon_lab;
CREATE TABLE vm_horizon_lab (id int PRIMARY KEY, payload text)
WITH (autovacuum_enabled = off);

INSERT INTO vm_horizon_lab VALUES (0, 'seed');
VACUUM (VERBOSE, ANALYZE) vm_horizon_lab;
```
Session A：
```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM vm_horizon_lab;
SELECT txid_current_snapshot();
```
保持事务打开。
Session B：
```sql
INSERT INTO vm_horizon_lab
SELECT i, repeat('z', 50)
FROM generate_series(1, 10000) AS g(i);

VACUUM (VERBOSE, ANALYZE) vm_horizon_lab;
SELECT * FROM pg_visibility_map_summary('vm_horizon_lab'::regclass);
```
如果 Session A 的 snapshot 早于 Session B 的 insert，这些 tuple 对 Session A 不可见。
VACUUM 不能把相关 page 标成 all-visible。
Session A：
```sql
COMMIT;
```
Session B：
```sql
VACUUM (VERBOSE, ANALYZE) vm_horizon_lab;
SELECT * FROM pg_visibility_map_summary('vm_horizon_lab'::regclass);
```
回到源码解释：
`heap_page_would_be_all_visible()` 在 `vacuumlazy.c:3735-3747` 检查 newest live xid 是否仍可能被某个 snapshot 认为 running。
如果是，page 不能 all-visible。
`pruneheap.c:1892-1905` 也追踪 page 上 newest live xmin。
这个实验说明：
VM 不是由“tuple 已提交”单独决定。
它还依赖 snapshot horizon。
## 24. 源码练习
练习一：画出 `RelationGetBufferForTuple()` 的 retry 状态机。
要求标出：
```text
targetBlock from relcache/bistate
GetPageWithFreeSpace()
last page fallback
heap page lock
PageGetHeapFreeSpace()
RecordAndGetPageWithFreeSpace()
RelationAddBlocks()
```
关键断点：
```text
RelationGetBufferForTuple
GetPageWithFreeSpace
RecordAndGetPageWithFreeSpace
RecordPageWithFreeSpace
```
练习二：画出一次 UPDATE 清 VM 的 WAL 边界。
要求标出：
```text
visibilitymap_pin before heap lock
PageClearAllVisible
visibilitymap_clear
log_heap_update
heap_xlog_update clears VM during redo
```
关键断点：
```text
heap_update
visibilitymap_clear
log_heap_update
heap_xlog_update
```
练习三：在 index-only scan 中验证 fallback。
要求标出：
```text
index_getnext_tid()
VM_ALL_VISIBLE()
index_fetch_heap()
InstrCountTuples2()
```
关键断点：
```text
IndexOnlyNext
visibilitymap_get_status
index_fetch_heap
```
练习四：观察 VACUUM 生产 VM/FSM。
要求标出：
```text
find_next_unskippable_block consumes VM
lazy_scan_prune produces VM
lazy_vacuum_heap_page records FSM after LP_DEAD cleanup
FreeSpaceMapVacuumRange propagates FSM
```
关键断点：
```text
find_next_unskippable_block
lazy_scan_prune
heap_page_prune_and_freeze
lazy_vacuum_heap_page
FreeSpaceMapVacuumRange
```
## 25. 讨论题
1. 为什么 FSM 可以接受 false positive，而 VM all-visible 不能接受 false positive？
2. `RecordPageWithFreeSpace()` 已经更新 bottom-level slot，为什么还需要 `FreeSpaceMapVacuumRange()`？
3. 如果 index-only scan 读 VM 不加锁，为什么看到新插入的 index TID 时仍能正确 fallback 到 heap？
4. 为什么 heap 修改 all-visible page 时要先 pin VM page，再锁 heap page，而不是锁住 heap page 后再读 VM fork？
5. normal VACUUM 跳过 all-visible page 后，为什么不能总是推进 `relfrozenxid`？
6. `all-frozen` 为什么不能独立于 `all-visible` 设置？
7. standby promotion 后，为什么 stale FSM 可能导致插入长时间命中 unusable page，但不造成错误结果？
8. `pg_freespace()`、`pg_visibility_map_summary()`、`relallvisible` 三者各自能看到什么，不能看到什么？
9. 如果 VM fork 被截断或清零，系统会变慢还是变错？为什么？
10. 如果 FSM fork 被截断或清零，系统会变慢、变胖还是变错？为什么？
## 26. 本节小结
本节主链路是：
```text
heap page 状态变化
  -> FSM/VM 辅助 fork 被记录、清除、设置或延后传播
  -> insert/vacuum/index-only scan 消费这些状态
  -> 失配时回到 heap page 真相或走 redo/retry/fallback
```
FSM 的核心是不精确空间索引。
它用 1 byte category 和树状 max 信息减少找页成本。
它允许 stale、torn、false positive、false negative。
因为最终插入前必须锁 heap page 并重新检查真实 free space。
VM 的核心是保守可见性事实。
它用每 heap block 两个 bit 表达 all-visible/all-frozen。
bit clear 只是“不知道”，会导致更多 heap scan 或 heap fetch。
bit set 会被 VACUUM 和 index-only scan 信任，所以必须由 heap lock、critical section、WAL redo 和 snapshot ordering 共同维护。
heap insert/update/delete 在修改 all-visible page 时清 `PD_ALL_VISIBLE` 和 VM bits。
VACUUM/prune 在确认 page 可见性和冻结状态后设置 VM，并把真实 free space 记录到 FSM。
index-only scan 用 VM 决定是否跳过 heap。
VM 不足时退回 `index_fetch_heap()`，用户可通过 `Heap Fetches` 观察。
FSM 和 VM 都是可重建辅助状态。
但它们的风险不同：
FSM 错了通常只是慢或空间复用差。
VM 错设会错结果。
可迁移规律是：
高性能内核常把昂贵真相拆成便宜辅助状态。
关键不是辅助状态永远准确。
关键是明确哪些错误方向可接受，哪些错误方向必须由锁、WAL、horizon 和 fallback 禁止。
