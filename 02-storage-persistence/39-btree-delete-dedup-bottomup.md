# PostgreSQL B-tree deletion、deduplication 与 bottom-up cleanup
## 课程定位
本节主题：B-tree deletion、deduplication 与 bottom-up cleanup。
上一组课程已经讲过 heap tuple version、HOT、heap delete 和 page pruning。
现在把视角移动到 nbtree leaf page。
前置知识：
- 已理解 MVCC tuple version 不是原地覆盖。
- 已理解 HOT update 为什么可以避免部分 index entry。
- 已理解 heap page pruning 何时把 tuple 变成 vacuumable。
- 已理解 buffer pin、buffer content lock、cleanup lock 和 WAL-before-data。
- 已知道 btree index scan 最终返回 heap TID，而不是直接返回 heap tuple。
本节唯一主问题：
当 B-tree leaf page 因 MVCC version churn 即将分裂时，PostgreSQL 如何在不破坏可见性、TID recycling 和 crash safety 的前提下，先删除 dead index tuple，再用 posting list dedup 兜底，尽量避免 index bloat？
本节围绕的核心矛盾：
MVCC update/delete 会留下旧 heap tuple version。
索引 entry 可能还要在一段时间内指向旧 heap TID。
如果每次 leaf page 放不下新 entry 就直接 split，短暂存在的旧版本会永久改变 B-tree 形状。
这会把 version churn 转化为长期 index bloat、更多 WAL、更多缓存失效和更深的扫描成本。
如果前台插入路径过度积极地检查 heap 可见性和重写 index page，又会把每次 insert/update 变成昂贵的 heap/index 协同清理。
如果 index AM 自己判断 heap tuple 是否可删，就会越过 table AM 和 MVCC horizon 的边界。
nbtree 的折中是：
先把“已知 dead”的 index tuple 用 `LP_DEAD` 做轻量 hint。
当目标 leaf page 即将 split 时，优先做 simple deletion。
如果 simple deletion 仍不足，并且插入看起来来自 version churn，再尝试 bottom-up deletion。
如果 deletion 仍不能释放足够空间，且索引允许 dedup，再把相邻 duplicate key 合并成 posting list。
如果三者都失败，才走普通 page split。
读完本节，你应该能独立判断：
- 为什么 `LP_DEAD` 是 dead index tuple cleanup 的起点，而不是最终回收。
- 为什么 VACUUM 删除 index tuple 要 full cleanup lock，前台 single-page cleanup 通常不需要。
- 为什么 posting list 是物理压缩，不改变 logical index contents。
- 为什么 posting list tuple 只能在所有 TID 都 dead 时整体标 `LP_DEAD`。
- 为什么 deleting a subset of posting list TIDs 要“更新” posting tuple，而不是只删 line pointer。
- 为什么 bottom-up deletion 是 insert path 上的 backstop，而不是 VACUUM 的替代品。
- 为什么 `indexUnchanged` hint 对 version churn 很关键，但不能作为正确性条件。
- 为什么 unique index 也会有物理 duplicate entry。
- 为什么 old snapshot 会让 bottom-up deletion 失败，然后 dedup 只是买时间。
- 为什么 WAL 里区分 `XLOG_BTREE_DELETE`、`XLOG_BTREE_VACUUM` 和 `XLOG_BTREE_DEDUP`。
- 为什么 `BTP_HAS_GARBAGE` 和 `LP_DEAD` 都不能被当成持久、完整的因果证据。
- 哪些现象能用 `pageinspect`、`VACUUM VERBOSE`、`pg_stat_*` 和 WAL 观察，哪些只能推断。
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
行号来自：
```text
nl -ba <source-file>
```
本节重点阅读：
```text
src/backend/access/nbtree/README
src/backend/access/nbtree/nbtinsert.c
src/backend/access/nbtree/nbtdedup.c
src/backend/access/nbtree/nbtpage.c
src/backend/access/nbtree/nbtree.c
src/backend/access/nbtree/nbtxlog.c
```
辅助阅读：
```text
src/include/access/nbtree.h
src/include/access/tableam.h
src/backend/access/heap/heapam.c
src/backend/access/nbtree/nbtutils.c
src/backend/access/nbtree/nbtsearch.c
```
辅助文件不是本节主角。
它们只用于解释 `LP_DEAD` 的产生、table AM 的删除裁决，以及 heap visibility horizon 的边界。
## 1. 先给结论
B-tree leaf page 即将放不下新 index tuple 时，当前源码不是马上 split。
`nbtinsert.c:913-920` 在 heapkeyspace index 中检测页面空间不足。
如果不足，调用 `_bt_delete_or_dedup_one_page()`。
这就是本节主流程的入口。
`_bt_delete_or_dedup_one_page()` 的注释在 `nbtinsert.c:2702-2710`。
它明确列出三个候选动作：
simple index deletion。
bottom-up index deletion。
deduplication。
三者都没释放足够空间，调用者才继续 split。
顺序很重要。
第一步总是 simple deletion。
`nbtinsert.c:2749-2766` 扫 leaf page 的 line pointer，收集 `ItemIdIsDead()` 的 offset。
这些是 `LP_DEAD` index tuple。
如果发现至少一个，`nbtinsert.c:2768-2776` 调用 `_bt_simpledel_pass()`。
如果页面已经能容纳新 tuple，直接返回。
第二步才是 bottom-up deletion。
`nbtinsert.c:2804-2823` 在两个条件下尝试：
executor 给了 `indexUnchanged` hint。
或者 unique check 过程中确认页面上存在 duplicate。
这两个条件都在表达同一个怀疑：
页面快满不是因为真正的 key-space 增长，而是因为同一 logical row 或同一 logical key 的旧版本堆在一起。
第三步是 deduplication。
`nbtinsert.c:2825-2828` 要求 reloption `deduplicate_items` 打开，且 scan key 上 `allequalimage` 为真。
dedup 不问 heap visibility。
它只改变 leaf page 上相同 key 的物理表示。
多个非 pivot tuple 可以合并成一个 posting list tuple。
posting list 是 heap TID array。
它不让 dead tuple 变 live。
它也不让 live tuple 变 dead。
它只是把同一 key 的多个 TID 放进一个 physical index tuple。
本节一句话运行模型：
```text
insert path 发现 leaf page 将满 -> simple deletion 先删 LP_DEAD -> bottom-up deletion 猜测 version churn 并让 table AM 裁决 -> dedup 把 duplicate TID 压成 posting list -> 都失败才 page split。
```
这条链路解释了为什么 nbtree 能在许多 version churn workload 下保持 index size 稳定。
它也解释了为什么这个稳定性不是保证。
old snapshot、replication slot、long transaction、heap page 访问成本、dedup eligibility、operator class equalimage 语义和 page layout 都会影响最终结果。
## 2. 这节课不讲什么
本节不完整展开 Lehman-Yao B-tree 查找和 split 算法。
只在 page split 作为 fallback 时提到它。
本节不完整展开 page deletion 的 half-dead subtree 算法。
只讲 leaf tuple cleanup 与 empty page deletion 的接口。
本节不完整展开 heap pruning 和 freeze。
只讲 index AM 通过 table AM 问“这个 heap TID 是否 vacuumable”的边界。
本节不完整展开 unique check 的所有 wait/retry 逻辑。
只讲它怎样产生 `LP_DEAD`，以及怎样触发 unique index 的 bottom-up/dedup。
本节的主角是 leaf page 上的 index tuple。
对象生命周期是：
一个 index tuple 被插入。
之后它可能被 index scan 标成 `LP_DEAD`。
在页面空间紧张或 VACUUM 时被物理删除。
如果它属于 posting list，删除可能表现为重写 posting list。
如果 deletion 暂时做不到，dedup 可能把多个 duplicate 合并起来。
如果这些机制都不足，leaf page split 让临时 version churn 变成长期结构成本。
## 3. 核心文件分工与阅读顺序
建议先读 `nbtree/README`。
它解释了为什么 index tuple deletion 不能只看 B-tree search correctness。
`README:166-193` 说明 VACUUM 删除 leaf item 前要 full cleanup lock。
原因不是 B-tree scan 会丢位置。
真正原因是避免 index scan 还拿着 TID 去 heap 时，VACUUM 已经让 heap TID recyclable。
`README:505-553` 解释 simple deletion。
`README:559-613` 解释 bottom-up deletion。
`README:903-987` 解释 deduplication 和 unique index dedup。
第二读 `nbtinsert.c`。
这是前台插入路径。
关键入口：
```text
_bt_doinsert()                 nbtinsert.c:104
_bt_findinsertloc()            nbtinsert.c:828
_bt_delete_or_dedup_one_page() nbtinsert.c:2730
_bt_simpledel_pass()           nbtinsert.c:2859
_bt_deadblocks()               nbtinsert.c:2984
```
第三读 `nbtdedup.c`。
它同时实现 dedup pass 和 bottom-up deletion candidate collection。
关键入口：
```text
_bt_dedup_pass()               nbtdedup.c:58
_bt_bottomupdel_pass()         nbtdedup.c:308
_bt_bottomupdel_finish_pending() nbtdedup.c:647
_bt_dedup_start_pending()      nbtdedup.c:434
_bt_dedup_save_htid()          nbtdedup.c:485
_bt_dedup_finish_pending()     nbtdedup.c:556
_bt_form_posting()             nbtdedup.c:863
_bt_update_posting()           nbtdedup.c:923
```
第四读 `nbtpage.c`。
它完成真正的 page mutation 和 WAL。
关键入口：
```text
_bt_upgradelockbufcleanup()    nbtpage.c:1136
_bt_delitems_vacuum()          nbtpage.c:1181
_bt_delitems_delete()          nbtpage.c:1312
_bt_delitems_update()          nbtpage.c:1434
_bt_delitems_delete_check()    nbtpage.c:1542
```
第五读 `nbtree.c`。
它连接 access method callback、index scan kill hint 和 VACUUM。
关键入口：
```text
bthandler()                    nbtree.c:117
btgettuple()                   nbtree.c:229
btrescan()                     nbtree.c:387
btendscan()                    nbtree.c:455
btbulkdelete()                 nbtree.c:1121
btvacuumcleanup()              nbtree.c:1151
btvacuumscan()                 nbtree.c:1239
btvacuumpage()                 nbtree.c:1414
btreevacuumposting()           nbtree.c:1752
```
第六读 `nbtxlog.c`。
它说明 crash recovery 看到的不是你的 C 调用栈。
它只看到 WAL record。
关键入口：
```text
btree_xlog_vacuum()            nbtxlog.c:586
btree_xlog_delete()            nbtxlog.c:640
btree_xlog_dedup()             nbtxlog.c:480 附近
btree_mask()                   nbtxlog.c:1080
```
最后读 `tableam.h` 和 `heapam.c` 的接口。
`tableam.h:177-277` 定义 `TM_IndexDeleteOp`。
`tableam.h:1397-1415` 定义 `table_index_delete_tuples()` 的语义。
`heapam.c:8098-8401` 是 heap AM 的实现。
这段代码回答：
nbtree 为什么只提供候选 TID 和 hint，而不直接决定删除。
## 4. 状态和不变量
第一个状态是 leaf page line pointer 的 `LP_DEAD`。
这是 `ItemId` 的 flag。
它表示 index AM 已经知道对应 index tuple 指向的 heap TID 对所有相关观察者 dead。
但 `LP_DEAD` 不是物理删除。
line pointer 仍存在。
index tuple bytes 仍在 page 上。
如果后续没有删除 pass，它仍占空间。
第二个状态是 `BTP_HAS_GARBAGE`。
它在 `nbtree.h:83` 定义。
注释已经说它是 deprecated。
`nbtinsert.c:2716-2727` 说明现代 heapkeyspace index 不再把它当 deletion 的 gate。
因为一旦页面即将 split，扫描 line pointer array 的成本很低。
靠 hint flag 反而可能漏掉已经可删的 tuple。
但是该 flag 仍被维护。
pg_upgrade 得来的旧格式索引和某些路径仍可能从它获益。
它也是一个典型例子：
flag 仍存在，不等于它仍是核心判断条件。
第三个状态是 posting list tuple。
非 pivot index tuple 可以有一个 posting list。
posting list 保存多个 heap TID。
`nbtdedup.c:845-861` 说明 `_bt_form_posting()` 的约定：
当 `nhtids == 1` 时形成普通 non-pivot tuple。
当 `nhtids > 1` 时形成 posting list tuple。
posting list 内的 TID 必须 unique 且升序。
posting list 不是新的逻辑索引项。
它是同一 key 的多个 heap TID 的物理表示。
第四个状态是 `BTDedupState`。
它是 backend-local、一次 pass 内的临时状态。
`nbtdedup.c:86-101` 初始化 dedup pass 的 state。
`base` 是当前 pending interval 的基准 tuple。
`htids` 是要合并进 posting list 的 heap TID array。
`nitems` 是 pending interval 包含的 physical items 数。
`phystupsize` 用于计算 dedup 能节省多少 page space。
`intervals` 用于 WAL replay 重新构造同样的 posting list 合并。
第五个状态是 `TM_IndexDeleteOp`。
这是 index AM 和 table AM 之间的共享协议对象。
它不是 shared memory。
它由调用者在当前 backend 中分配并传给 table AM。
`tableam.h:266-277` 包括：
`irel` 和 `iblknum` 表示目标 index relation 和 index block。
`bottomup` 表示 simple deletion 还是 bottom-up deletion。
`bottomupfreespace` 是 bottom-up 希望释放的空间目标。
`deltids` 保存候选 heap TID。
`status` 保存每个候选的 index page offset、是否已知可删、是否 promising、删除后释放多少空间。
第六个状态是 `snapshotConflictHorizon`。
前台 single-page deletion 生成 `XLOG_BTREE_DELETE` 时需要它。
`tableam.h:1406-1409` 说明它可能在 standby REDO 时触发 snapshot conflict。
VACUUM 的 `XLOG_BTREE_VACUUM` 没有相同字段。
`nbtpage.c:1174-1179` 解释原因：
VACUUM 依赖初始 heap table scan 侧 WAL 记录间接处理 conflict horizon。
前台 delete 必须自己从 table AM 拿 horizon。
第七个状态是 vacuum cycle ID。
它不是 bottom-up deletion 的状态。
它属于 VACUUM 线性扫描和 concurrent split 的互锁。
`README:215-230` 说明它用来识别当前 VACUUM cycle 开始后发生过 split 的 page。
`btvacuumpage()` 会在必要时 backtrack。
前台 `_bt_delitems_delete()` 不能清 `btpo_cycleid`。
`nbtpage.c:1355-1358` 明确说只有 VACUUM command 控制 vacuum cycle ID。
## 5. 主流程一：index tuple 从 live 到 LP_DEAD
dead index tuple cleanup 的第一阶段通常不是物理删除。
它是普通 index scan 或 unique check 设置 `LP_DEAD`。
`btgettuple()` 在 `nbtree.c:229-285`。
当调用者发现上一个返回的 heap tuple dead，可以设置 `scan->kill_prior_tuple`。
下一次 `btgettuple()` 进入时，`nbtree.c:253-270` 把当前 itemIndex 记入 `so->killedItems`。
它没有立刻改 page。
它等到 scan 离开当前 page。
`btrescan()` 在 `nbtree.c:393-400` 离开当前 page 前调用 `_bt_killitems()`。
`btendscan()` 在 `nbtree.c:459-465` 结束 scan 前也调用 `_bt_killitems()`。
这样可以把同一页上的多个 dead hint 批量处理。
`_bt_killitems()` 定义在 `nbtutils.c`。
本节不展开全部代码。
需要记住三点。
第一，它只是设置 line pointer dead flag。
第二，它会设置 `BTP_HAS_GARBAGE` hint。
第三，它必须检查 page LSN，避免 scan 离开 pin 后页面被 VACUUM 删除并重用，导致错误标记新的 unrelated tuple。
`README:474-489` 描述这个 race。
如果 leaf page LSN 已变，affected index scan 保守地不设置任何 `LP_DEAD`。
unique check 也能产生 `LP_DEAD`。
`nbtinsert.c:679-709` 在确认 conflicting tuple 或 posting list 所有 HOT chain 都 dead 后，尝试把当前 index item 标 dead。
它使用 hint-bit infrastructure。
这条路径仍然不是 correctness requirement。
代码注释说：
能标就标，不能标也不影响正确性。
这就是 `LP_DEAD` 的定位：
它是未来删除的 cheap signal。
它不是决定 heap tuple 是否 dead 的唯一信息源。
它不是 WAL 持久语义。
它也不是同步屏障。
## 6. 主流程二：插入即将 split 时先 simple deletion
`_bt_doinsert()` 在 `nbtinsert.c:104-279`。
它先构造 scan key。
如果需要 unique check，就先 `_bt_check_unique()`。
如果要等待其他事务，`nbtinsert.c:216-235` 释放 buffer，等待，再从 `search` 标签重新开始。
这解释了一个重要错误路径：
unique check 的 wait 会让之前的 page state 全部过期。
不能等待后继续使用旧 buffer 和旧 bounds。
真正插入前，`nbtinsert.c:261-265` 调用 `_bt_findinsertloc()`，再 `_bt_insertonpg()`。
`_bt_findinsertloc()` 是本节最关键的前台入口。
它的 `indexUnchanged` 注释在 `nbtinsert.c:808-817`。
意思是：
这是一个 UPDATE 产生的新 index tuple。
indexed value 逻辑上没变。
但因为新 heap tuple version 需要自己的 TID，所以仍要插入新 entry。
这正是 version churn 的形状。
当目标 leaf page 空间不足时，`nbtinsert.c:913-920` 调用 `_bt_delete_or_dedup_one_page()`。
进入 `_bt_delete_or_dedup_one_page()` 后，第一件事是扫描所有 data items。
`nbtinsert.c:2756-2766` 从 `P_FIRSTDATAKEY()` 到 `PageGetMaxOffsetNumber()`。
遇到 `ItemIdIsDead(itemId)` 就把 offset 放进 `deletable`。
如果 `ndeletable > 0`，调用 `_bt_simpledel_pass()`。
simple deletion 不只删这些 `LP_DEAD` tuple。
`nbtinsert.c:2838-2854` 说明它还会尝试删除 extra index tuples。
但 extra tuple 必须足够便宜。
它们通常指向同一批 heap block。
`_bt_deadblocks()` 在 `nbtinsert.c:2966-3051` 先收集所有 `LP_DEAD` tuple 指向的 heap block。
它还总是加入 incoming newitem 的 heap block。
`nbtinsert.c:2973-2980` 解释原因：
incoming tuple 所在 heap block 几乎肯定已经在 cache 中。
这个 block 上可能还有最近变 dead 的旧版本。
这个 locality 假设把“多检查一点”变成低成本。
`_bt_simpledel_pass()` 初始化 `TM_IndexDeleteOp`。
`nbtinsert.c:2873-2880` 设置：
`bottomup = false`。
`bottomupfreespace = 0`。
`deltids` 和 `status` 是 per-call palloc 数组。
对普通 tuple，`nbtinsert.c:2893-2916` 如果 TID block 属于 deadblocks，就加入候选。
`knowndeletable` 等于 `ItemIdIsDead(itemid)`。
对 posting list，`nbtinsert.c:2918-2950` 会把 posting list 中属于 deadblocks 的每个 TID 都加入候选。
最后 `nbtinsert.c:2958-2959` 调用 `_bt_delitems_delete_check()`。
这里的名字很关键。
它不是直接删。
它先让 table AM 裁决。
## 7. table AM 裁决：index AM 不直接判断 visibility
`_bt_delitems_delete_check()` 在 `nbtpage.c:1542-1709`。
第一步就是 `table_index_delete_tuples()`。
`nbtpage.c:1555-1557` 调用 table AM，拿到 `snapshotConflictHorizon` 和 catalog relation 信息。
这一步之前，nbtree 只有候选 TID。
这一步之后，候选数组可能被排序、缩短、标记为可删。
`tableam.h:177-231` 是接口契约。
simple deletion caller 通常包括 known-dead entries。
也可以包括一些 speculative extra entries。
bottom-up deletion caller 全部是 speculative。
`tableam.h:192-199` 明确说 bottom-up caller 不允许把任何 entry 预先标成 `knowndeletable = true`。
原因很直接：
bottom-up 是猜测 version churn。
猜测只能指导成本。
不能替代 visibility 判断。
heap AM 的实现是 `heap_index_delete_tuples()`。
`heapam.c:8093-8096` 提醒这可能产生不少 I/O。
因此它会 prefetch，并合并对同一 heap block 的重复访问。
`heapam.c:8123-8127` 初始化 `SnapshotNonVacuumable` 并按 TID 排序。
这个 snapshot 的意义是：
如果 HOT chain 中还有任何 non-vacuumable tuple，这个 index entry 不能删。
`heapam.c:8294-8300` 调用 `heap_hot_search_buffer()`。
如果找到 non-vacuumable tuple，continue。
否则把 `istatus->knowndeletable = true`。
这就是安全删除的核心边界：
index entry 可以被删，不是因为 index tuple 自己 dead。
而是因为它指向的 heap TID 所在 HOT chain 对相关 horizon 已经 vacuumable。
同一函数还维护 `snapshotConflictHorizon`。
`heapam.c:8312-8384` 沿 HOT chain 检查 tuple header，调用 `HeapTupleHeaderAdvanceConflictHorizon()`。
如果遇到 `LP_DEAD` line pointer，它依赖更早的 heap prune WAL 记录已经处理过 conflict horizon。
`heapam.c:8342-8355` 解释了这一点。
这也是为什么前台 index deletion 需要 table AM。
只有 heap AM 能把 HOT chain、line pointer、tuple header 和 conflict horizon 串起来。
## 8. 主流程三：真正改 leaf page
table AM 返回后，`_bt_delitems_delete_check()` 要把结果重新映射回 leaf page。
`nbtpage.c:1563-1579` 先按 `id` 恢复 leaf-page-wise order。
如果 bottom-up caller 最后没有任何可删 entry，`ndeltids == 0`，直接返回。
这是一条正常 fallback。
它不是 error。
它表示这次猜测没有找到可删 tuple。
对普通 non-posting tuple，`nbtpage.c:1609-1615` 只有 `knowndeletable` 为真才把 offset 放进 `deletable`。
对 posting list tuple，`nbtpage.c:1618-1678` 会逐个 TID 比对。
如果所有 TID 都可删，`nbtpage.c:1686-1691` 把整个 posting tuple offset 放进 `deletable`。
如果只有部分 TID 可删，`nbtpage.c:1693-1699` 把它放进 `updatable`。
最后 `nbtpage.c:1702-1704` 调用 `_bt_delitems_delete()`。
这就是 single-page cleanup 的 page mutation 入口。
它的 VACUUM 版本是 `_bt_delitems_vacuum()`。
两者对 page 的修改非常接近：
先用 `_bt_delitems_update()` 为 posting list 生成删除部分 TID 后的新 tuple。
再覆盖 posting tuple。
再 `PageIndexMultiDelete()` 删除整个 tuple。
再清 `BTP_HAS_GARBAGE`。
再写 WAL 或 fake LSN。
不同点在于 WAL 和 vacuum cycle ID。
`_bt_delitems_vacuum()` 在 `nbtpage.c:1235-1240` 清 `btpo_cycleid`。
`_bt_delitems_delete()` 在 `nbtpage.c:1355-1358` 明确不能清。
VACUUM 的 WAL 是 `XLOG_BTREE_VACUUM`。
`nbtpage.c:1254-1278` 记录 `ndeleted`、`nupdated`、deleted offsets、updated offsets 和 update payload。
前台 single-page deletion 的 WAL 是 `XLOG_BTREE_DELETE`。
`nbtpage.c:1372-1398` 还记录 `snapshotConflictHorizon` 和 `isCatalogRel`。
两者都在 critical section 内先改 page，再插 WAL，再设置 page LSN。
`nbtpage.c:1203-1204` 和 `nbtpage.c:1335-1336` 都写着：
在 changes logged 前不能 `ereport(ERROR)`。
如果 `PageIndexTupleOverwrite()` 失败，代码 `elog(PANIC)`。
这是因为进入 critical section 后页面已经在被原子修改。
不能用普通 ERROR 回滚内存栈来修复 page。
## 9. VACUUM 线：为什么它不是简单的后台版 simple deletion
`bthandler()` 在 `nbtree.c:117-160` 注册 access method callbacks。
`ambulkdelete = btbulkdelete`。
`amvacuumcleanup = btvacuumcleanup`。
这两条是 VACUUM 进入 nbtree 的入口。
`btbulkdelete()` 在 `nbtree.c:1121-1143`。
它用 `PG_ENSURE_ERROR_CLEANUP()` 包住 `_bt_start_vacuum()` 和 `btvacuumscan()`。
如果 VACUUM 中途 ERROR，`_bt_end_vacuum_callback` 仍会清共享状态。
这是本节的 ERROR cleanup 例子。
它不是“事务结束自然释放”这么简单。
VACUUM 的 btree scan 是物理块顺序。
`btvacuumscan()` 在 `nbtree.c:1291-1308` 说明必须反复检查 relation length。
原因是 concurrent split 可能新增 page。
VACUUM 不能漏掉新 page 上的 deletable tuple。
`btvacuumpage()` 是单页处理入口。
它先 read lock，然后对 leaf page 升级到 cleanup lock。
`nbtree.c:1533-1539` 注释说：
VACUUM scan 过程中每个 leaf page 都要拿 full cleanup lock。
即使页面没有 deletable tuple，也要拿。
这对应 `README:195-202`。
因为 page split 可能把 index scan 看过的 tuple 移到右边。
只有拿到每个 leaf page 的 cleanup lock，才能和仍持 page pin 的 scan 建立 TID recycling 互锁。
VACUUM 删除 leaf tuple 时，`btvacuumpage()` 遍历所有 tuple。
普通 tuple 通过 callback 判断是否删除。
posting list tuple 通过 `btreevacuumposting()` 判断要删除哪些 TID。
`nbtree.c:1592-1629` 分三类：
posting list 没有变化。
posting list 部分 TID 删除，加入 `updatable`。
posting list 所有 TID 删除，加入 `deletable`。
最后 `nbtree.c:1634-1642` 每页只调用一次 `_bt_delitems_vacuum()`。
注释说这是为了减少 WAL traffic。
如果 leaf page 变空，`nbtree.c:1673-1695` 尝试 page deletion。
这一步超出本节主线。
但要记住：
tuple deletion 回收 leaf page 内空间。
page deletion 把空 leaf page 从 B-tree 结构中删除。
page recycling 又要等到安全放入 FSM。
`README:383-435` 说明 deleted page 必须先作为 tombstone 留一段时间。
不能马上复用为 unrelated page。
## 10. Bottom-up deletion 的触发条件
bottom-up deletion 不是周期任务。
它发生在 insert path。
它被 page space pressure 触发。
没有即将 split 的压力，就不会主动做 bottom-up pass。
触发入口是 `_bt_delete_or_dedup_one_page()`。
simple deletion 后，如果仍不够空间，代码来到 `nbtinsert.c:2804-2828`。
`indexUnchanged` 是第一类触发条件。
它来自 executor 对 UPDATE 的 hint。
含义不是“这次不需要 index entry”。
恰恰相反：
这次由于非 HOT 更新需要插入 index entry。
但对当前这个 index 来说，indexed value 没变。
所以新 index entry 与旧 entry 是 logical duplicate，只是 heap TID 不同。
unique duplicate 是第二类触发条件。
unique index 在 MVCC 下可以有多个 physical entries 拥有相同 key。
只要不会让两个 live tuple 同时对新 snapshot 可见，就不违反 uniqueness。
因此 unique index 也会受 version churn 影响。
`nbtinsert.c:2805-2810` 说这两类触发有大量 overlap。
保留两类条件，是为了覆盖连续 INSERT/DELETE 相关 churn。
bottom-up deletion 的目标不是“尽量删多”。
`nbtdedup.c:282-291` 说它的目标是 entirely prevent unnecessary page splits caused by MVCC version churn。
这是 qualitative，不是 quantitative。
它不关心一次性删掉很多 tuple。
它关心让同一类 leaf page 长期不因短命旧版本 split。
因此 bottom-up deletion 的成功定义也不是“这次删了多少行”。
`nbtdedup.c:297-303` 说返回 true 表示 caller 可以认为 page split 会被合理地避免一段时间。
有时即使没有释放足够空间，也会返回 true，让 caller 跳过 dedup 直接 split。
这体现了成本控制：
如果当前 page 根本没有 duplicate interval，dedup 也没意义。
## 11. Bottom-up deletion 如何选择候选 TID
`_bt_bottomupdel_pass()` 在 `nbtdedup.c:308-423`。
它借用了 dedup 的 equality grouping 逻辑。
但它不真的形成 posting list。
`nbtdedup.c:325-337` 初始化 `BTDedupState`。
这里 `maxpostingsize = BLCKSZ`。
注释说：
我们不是真的 deduplicating。
只是复用“找 duplicate interval 并收集 TID”的机制。
`nbtdedup.c:339-354` 初始化 `TM_IndexDeleteOp`。
重要字段：
`bottomup = true`。
`bottomupfreespace = Max(BLCKSZ / 16, newitemsz)`。
这个目标告诉 table AM：
如果释放到这个空间，就差不多可以停止。
`nbtdedup.c:363-394` 遍历 leaf page。
相邻 tuple 如果 `_bt_keep_natts_fast()` 判断 key 相等，且 `_bt_dedup_save_htid()` 能保存 TID，就属于同一个 interval。
遇到不相等 tuple，调用 `_bt_bottomupdel_finish_pending()`。
`_bt_bottomupdel_finish_pending()` 在 `nbtdedup.c:647-753`。
它决定哪些 entry 是 promising。
普通 non-posting tuple 的规则很简单。
`nbtdedup.c:665-674` 把每个 tuple 的 TID 放进 `deltids`。
`promising = dupinterval`。
如果 interval 有多个 physical items，就认为它们像 version churn。
`freespace` 是该 line pointer 和 item size。
posting list tuple 的规则更保守。
`nbtdedup.c:680-689` 说明：
最多只把 posting list 中一个 presumed logical row 标成 promising。
因为 posting list 本来就是 dedup 或 index build 形成的。
它不太可能代表最近版本 churn。
当 posting list 也处在 duplicate interval 内时，才给一个弱 hint。
`nbtdedup.c:696-734` 选择 first 或 last TID 做 promising。
选择依据是 TID 所在 heap block 的分布。
这不是 correctness 规则。
它只是让 heap AM 更容易选到值得访问的 heap block。
这一段最能体现 bottom-up deletion 的本质：
index AM 只识别“形状像 version churn 的 index page 区域”。
table AM 再决定哪些 heap TID 真正 vacuumable。
## 12. Heap AM 如何给 bottom-up 控成本
bottom-up deletion 如果检查 leaf page 上所有 TID，可能会很贵。
一页 index 可以对应很多 heap block。
盲目访问会把 insert path 变成随机 heap I/O 放大器。
因此 heap AM 对 bottom-up caller 有特殊成本控制。
`heap_index_delete_tuples()` 在 `heapam.c:8128-8137` 检测 `delstate->bottomup`。
如果为真，调用 `bottomup_sort_and_shrink()`。
`bottomup_sort_and_shrink()` 在 `heapam.c:8637-8768`。
它按 heap block group 汇总 TID。
优先保留 promising TID 多的 heap block。
最终只保留 `BOTTOMUP_MAX_NBLOCKS` 个最 promising 的 heap block。
`heapam.c:8646-8650` 注释说：
这经常把候选数组缩小到原来的一小部分。
随后 heap AM 访问 heap block。
`heapam.c:8188-8221` 有两个 early stop 规则。
如果已经进入 final block，break。
如果访问一个新 block 后没有增加 actual free space，break。
这是控制 bottom-up 成本的主规则。
它保证失败时不至于扫描太多 heap blocks。
`heapam.c:8223-8251` 还会衰减 `curtargetfreespace`。
如果连续 heap block 有 favorable locality，就更有耐心。
如果不是 favorable block，就把目标减半。
这说明 bottom-up deletion 不是只看“能不能删”。
它同时看“删这些 tuple 是否值得现在付出 heap 访问成本”。
当某个候选 TID 通过 `heap_hot_search_buffer()` 发现 whole HOT chain vacuumable，`heapam.c:8294-8300` 把 `knowndeletable` 设为 true。
如果 bottom-up，`heapam.c:8302-8308` 累加 `actualfreespace`。
达到目标后设置 `bottomup_final_block`。
最后 `heapam.c:8392-8401` 会把 `ndeltids` 缩小到最终需要 index AM 处理的范围。
bottom-up caller 可以收到 `ndeltids == 0`。
这就是“没找到可删 tuple”的正常结果。
## 13. Deduplication 的运行模型
deduplication 的目标不是 visibility cleanup。
它不问 heap tuple 是否 dead。
它只把 equal key 的 non-pivot tuples 合并成 posting list。
`README:903-921` 说得很清楚：
deduplication alters physical representation without changing logical contents。
它 lazy 地发生在 page split 前。
且应该在 `LP_DEAD` items 被删除之后才发生。
`_bt_dedup_pass()` 在 `nbtdedup.c:58-280`。
它先把 incoming item size 加上 line pointer overhead。
`nbtdedup.c:74-75` 说明 caller 传来的 `newitemsz` 不包括 line pointer。
然后初始化 `BTDedupState`。
`nbtdedup.c:80-89` 把 posting list size 限制在 `BTMaxItemSize / 2` 和 `INDEX_SIZE_MASK` 之间。
注释解释：
理论上可以用 page 三分之一，但实际限制到六分之一更利于后续 split point。
`nbtdedup.c:120-121` 用 `PageGetTempPageCopySpecial()` 拷贝当前 page。
新 page 会复制原 page LSN。
注释说 XLogInsert 可能检查 LSN 并决定是否写 full page image。
然后它遍历 page 上所有 data items。
相同 key 的 tuple 被收进 pending posting list。
`nbtdedup.c:151-153` 用 `_bt_keep_natts_fast()` 判断是否等于 base tuple。
用 `_bt_dedup_save_htid()` 保存 heap TID。
遇到不同 key 或保存失败，就 `_bt_dedup_finish_pending()`。
`_bt_dedup_finish_pending()` 在 `nbtdedup.c:556-609`。
如果 pending interval 只有一个 item，就把原 tuple 原样加到 newpage。
如果有多个 item，就调用 `_bt_form_posting()` 形成 posting list tuple。
空间节省来自：
多个 physical tuples 的 tuple header、line pointer 和重复 key payload 被合并。
posting list 只保存多个 heap TID。
如果整页没有任何 interval 能 dedup，`nbtdedup.c:208-225` 直接释放临时内存并返回。
不会写 page。
这也是重要 fallback。
dedup pass 本身不保证避免 split。
`nbtdedup.c:274-275` 的 assert 只是确认如果 saving 足够，则 page free space 能满足目标。
如果 saving 不够，caller 仍会 split。
## 14. Posting list 的删除和更新
posting list 让 cleanup 变复杂。
一个 line pointer 可能代表多个 heap TID。
如果所有 TID 都 dead，可以删除整个 posting tuple。
如果只有部分 TID dead，就要保留同一 key 的其他 live TID。
这时不能简单 `PageIndexMultiDelete()`。
要重写 posting tuple。
VACUUM 路径由 `btreevacuumposting()` 准备 partial delete。
`nbtree.c:1752-1794` 遍历 posting list 中的每个 TID。
如果 callback 判断 dead，就把 TID ordinal 放进 `BTVacuumPostingData.deletetids`。
返回的 `nremaining` 告诉调用者：
没有变化。
部分删除。
全部删除。
前台 deletion 路径由 `_bt_delitems_delete_check()` 准备 partial delete。
`nbtpage.c:1618-1699` 将 table AM 返回的 deletable TID 映射回 posting list ordinal。
如果只删部分 TID，放进 `updatable`。
真正生成新 tuple 的函数是 `_bt_update_posting()`。
它在 `nbtdedup.c:923` 开始。
`nbtdedup.c:913-921` 注释说：
它被 VACUUM 和 index deletion 共同使用。
调用者传入旧 posting list 和要删除的 TID ordinal。
函数返回 palloc 出来的 replacement tuple。
`_bt_delitems_update()` 在 `nbtpage.c:1434-1487` 是 WAL 准备层。
它调用 `_bt_update_posting()`。
同时生成 `xl_btree_update` payload。
redo 时 `nbtxlog.c:545-583` 的 `btree_xlog_updates()` 会根据 WAL payload 重新调用 `_bt_update_posting()`。
这是一条重要不变量：
WAL 不需要存整条 replacement posting tuple。
它存删除哪些 posting list ordinal。
redo 根据旧 page image 上的 original posting tuple 重新构造。
因此 posting list 的 TID 顺序、ordinal 和 tuple layout 必须稳定。
## 15. 为什么 dedup 需要 allequalimage
dedup 会把多个 index tuples 合并。
它要求 operator class 的 equality 语义足够安全。
`_bt_delete_or_dedup_one_page()` 在 `nbtinsert.c:2825-2828` 检查：
`BTGetDeduplicateItems(rel)`。
`itup_key->allequalimage`。
`BTGetDeduplicateItems()` 在 `nbtree.h:1135-1139`。
默认 true。
但用户可通过 reloption 关闭。
`allequalimage` 来自 metapage 和 operator class support。
本节不展开 equalimage support function。
要记住边界：
bottom-up deletion 使用 equality grouping 来找 duplicate shape。
但它不 merge tuple。
因此 `nbtinsert.c:2816-2819` 说 bottom-up deletion deliberately omit index-is-allequalimage test。
dedup 会改变物理表示。
它必须确认“这些 equal key 的 image 合并不会破坏后续比较和重构”。
所以 dedup 需要 `allequalimage`。
这解释了一个常见现象：
某个 index 明明有很多 duplicate key，仍可能不 dedup。
原因可能不是没有 version churn。
而是 dedup reloption 关闭，或者 operator class 没有给出安全 equalimage 语义。
## 16. Version churn 下如何避免 bloat
考虑一个有两个索引的表。
`id` 是 primary key。
`v` 上还有普通索引。
如果不断执行：
```sql
UPDATE t SET v = v + 1 WHERE id = 42;
```
对 primary key index 来说，`id` 没变。
但这次 UPDATE 不是 HOT，因为 `v` 的索引必须维护。
executor 仍要给 primary key index 插入新 entry，指向新 heap TID。
旧 primary key index entry 暂时不能删。
它可能仍被旧 snapshot 需要。
这就形成 unique index 内部的 physical duplicates。
如果没有 bottom-up deletion，这些短命 duplicate 可能把 primary key leaf page 推到 split。
split 一旦发生，B-tree 形状永久改变。
旧 version 后来被 VACUUM 清掉，也不能自动把两个半空 page merge 回一个 page。
nbtree 的策略是把 split 作为最后手段。
页面满时先 simple deletion。
如果 scan 或 unique check 已经标了 `LP_DEAD`，先删。
如果没有 `LP_DEAD`，但 incoming tuple 被标记 `indexUnchanged`，尝试 bottom-up。
bottom-up 会优先检查 duplicate interval 指向的 heap blocks。
如果旧 versions 已经过了 vacuum horizon，就删掉旧 index entries。
如果 old snapshot 仍持有它们，table AM 不会允许删除。
这时 dedup 可能仍能把 duplicates 压成 posting list。
dedup 不解决 dead tuple。
它只是把同一 key 的多个 heap TID 用更紧凑的形式存起来。
它为未来 cleanup 争取时间。
等 old snapshot 释放后，下一次 page pressure、simple deletion、bottom-up pass 或 VACUUM 可能真正删除这些 TID。
这就是 `README:979-987` 说的关系：
bottom-up deletion 是 preferred way。
dedup 可以 augment bottom-up deletion。
当 deletion 因 old snapshot 失败时，dedup 提供额外容量。
## 17. WAL 和 redo 视角
本节涉及三类主要 WAL record。
第一类是 `XLOG_BTREE_DELETE`。
它来自前台 simple deletion 或 bottom-up deletion 最终的 `_bt_delitems_delete()`。
`nbtpage.c:1375-1398` 记录：
`snapshotConflictHorizon`。
`ndeleted`。
`nupdated`。
`isCatalogRel`。
deleted offsets。
updated posting offsets。
posting update payload。
redo 函数是 `btree_xlog_delete()`。
`nbtxlog.c:648-661` 在 Hot Standby 下先处理 recovery conflict。
它调用 `ResolveRecoveryConflictWithSnapshot()`。
这必须发生在修改 index page 之前。
`nbtxlog.c:663-699` 然后更新 posting list、删除 offsets、清 `BTP_HAS_GARBAGE`、设置 LSN。
它不清 `btpo_cycleid`。
第二类是 `XLOG_BTREE_VACUUM`。
它来自 VACUUM 的 `_bt_delitems_vacuum()`。
`nbtpage.c:1257-1278` 不带 `snapshotConflictHorizon`。
redo 是 `btree_xlog_vacuum()`。
`nbtxlog.c:594-599` 说明 recovery 期间这里需要 cleanup lock。
但不需要像 VACUUM 正常执行那样锁每个 leaf page。
有 items to kill 的 page 足够。
`nbtxlog.c:621-630` redo 删除 tuple 后清 `btpo_cycleid` 和 `BTP_HAS_GARBAGE`。
第三类是 `XLOG_BTREE_DEDUP`。
它来自 `_bt_dedup_pass()`。
`nbtdedup.c:247-265` 记录 `nintervals` 和 intervals array。
redo 逻辑在 `nbtxlog.c:480-539` 附近。
它根据 intervals 重建 newpage。
`nbtxlog.c:508-524` 复用 `_bt_dedup_start_pending()`、`_bt_dedup_save_htid()` 和 `_bt_dedup_finish_pending()`。
这说明 dedup WAL 是逻辑描述：
不是直接存完整页面的新 bytes。
而是存哪些 interval 要合并。
另外还要记住两个“没有 WAL”的状态。
`LP_DEAD` 可以不经 WAL 改变。
`nbtxlog.c:1093-1100` 的 consistency mask 会 mask leaf page 的 line pointer flags。
`BTP_HAS_GARBAGE` 也是 unlogged hint。
`nbtxlog.c:1103-1108` 会 mask 掉它。
所以观察到 standby 或 crash 后状态不同，不一定是 bug。
只要后续 physical deletion WAL 和 heap WAL 保证正确性即可。
## 18. 错误路径、异常路径和 fallback
第一类 fallback：unique check wait 后重试。
`nbtinsert.c:201-235` 如果要等其他事务，释放锁，等待，然后 `goto search`。
等待期间 leaf page 可以 split、delete、dedup。
旧 buffer、旧 stack、旧 bounds 都不能继续用。
第二类 fallback：simple deletion 没有足够空间。
`_bt_delete_or_dedup_one_page()` 在 simple deletion 后重新检查 page free space。
`nbtinsert.c:2774-2776` 如果够了就返回。
不够就继续。
这不是 failure。
这是正常进入 bottom-up 或 dedup。
第三类 fallback：bottom-up 找不到可删 tuple。
`_bt_delitems_delete_check()` 在 `nbtpage.c:1576-1579` 允许 `ndeltids == 0`。
它只在 bottom-up caller 中合法。
这表示 table AM 认为没有候选现在可删。
第四类 fallback：bottom-up 即使没释放足够空间也可能阻止 dedup。
`nbtdedup.c:417-422` 如果 `neverdedup` 为真，返回 true。
如果已经释放到一定空间，也返回 true。
caller 看到 true 就不会 dedup。
这反映了启发式：
如果没有 duplicate interval，dedup 也不该浪费时间。
第五类 fallback：dedup 没有 interval。
`nbtdedup.c:208-225` 释放临时 page 和 state 后直接返回。
caller 会继续 split。
第六类 fallback：dedup 不被允许。
`BTGetDeduplicateItems(rel)` 可能 false。
`allequalimage` 可能 false。
这时 `_bt_dedup_pass()` 根本不会被调用。
第七类 fallback：三种机制都不能避免 split。
这时 `_bt_insertonpg()` 会继续普通 B-tree insert。
如果 page 仍放不下，它会走 `_bt_split()`。
这保证正确性优先。
避免 bloat 是 best effort，不是 correctness precondition。
第八类 ERROR/PANIC 边界：critical section。
`_bt_delitems_vacuum()` 和 `_bt_delitems_delete()` 都在 critical section 里改 page。
进入后不能普通 ERROR。
`PageIndexTupleOverwrite()` 失败时 `elog(PANIC)`。
这不是夸张。
它说明此时 page mutation 和 WAL 原子性已经进入不能局部回滚的区域。
第九类 cleanup 边界：VACUUM shared state。
`btbulkdelete()` 用 `PG_ENSURE_ERROR_CLEANUP()` 保证 `_bt_end_vacuum()` 相关清理发生。
如果这里漏清，后续 VACUUM cycle ID 和 shared vacuum state 可能污染下一次操作。
## 19. 正确性机制层次
MVCC visibility 决定 heap tuple 是否 vacuumable。
它不由 nbtree 判断。
nbtree 只把候选 TID 交给 table AM。
heap AM 用 `SnapshotNonVacuumable`、HOT chain 和 tuple header 裁决。
buffer exclusive lock 保护 leaf page 的物理修改。
前台 `_bt_delitems_delete()` 要求 caller 持有 pinned and write locked buffer。
VACUUM `_bt_delitems_vacuum()` 要求 full cleanup lock。
cleanup lock 额外保证没有其他 backend pin 住该 buffer。
这对 TID recycling 互锁必要。
`README:169-181` 解释了这个要求。
`README:189-193` 同时说明前台 opportunistic deletion 不需要 cleanup lock。
因为它不会让 heap TID recyclable。
只有 VACUUM 后续才能把 heap line pointer 变成可复用状态。
WAL 保护 crash safety。
`XLOG_BTREE_DELETE` 和 `XLOG_BTREE_VACUUM` 保护 index page 删除/更新。
`XLOG_BTREE_DEDUP` 保护 posting list 合并。
heap prune/VACUUM WAL 保护 heap 侧 conflict horizon。
standby recovery conflict 由 `snapshotConflictHorizon` 驱动。
pin/refcount 保护 buffer 生命周期。
它不保证 tuple visibility。
它也不保证页面语义不变。
所以 scan 需要 LSN 检查来避免 kill_prior_tuple race。
hint bits 和 unlogged flags 只是优化。
`LP_DEAD` 和 `BTP_HAS_GARBAGE` 都可能丢失、滞后或不完整。
系统不能依赖它们保证正确性。
page split fallback 保证 forward progress。
如果 deletion 和 dedup 都做不了，insert 仍然必须成功或按普通错误路径失败。
不能为了避免 bloat 阻塞正确插入。
## 20. 成本模型
simple deletion 的成本主要是 leaf page line pointer scan。
在 page 即将 split 时，这个成本很低。
因为 split 本来就会读写该 page，并且比线性扫描 page 更贵。
额外成本来自 table AM 检查 heap blocks。
`_bt_deadblocks()` 把范围限制到 `LP_DEAD` tuple 和 incoming tuple 所在 blocks。
这利用 heap locality 控制成本。
bottom-up deletion 的成本主要是 heap block 访问。
它可能检查很多候选 TID。
heap AM 用 promising hint、block grouping、`BOTTOMUP_MAX_NBLOCKS`、early stop 和 target decay 控制。
失败时应该便宜。
成功时应该释放足够空间，避免 split。
dedup 的成本主要是重写一个 temp page、比较相邻 tuples、形成 posting list、写 WAL。
它不访问 heap。
所以当 old snapshot 阻止 deletion 时，dedup 仍可能有用。
但 dedup 会增加后续删除复杂度。
因为一个 posting tuple 可能需要 partial TID deletion。
VACUUM 的成本是全 index scan 和 cleanup lock。
它不是每次前台 insert 都能承受的成本。
但它能处理全局垃圾、空页删除、deleted page FSM 放置等前台路径不做的工作。
page split 的成本是长期成本。
它会增加 page 数、改变树结构、写更多 WAL、让未来 scan 访问更多 leaf page。
在 version churn workload 下，split 的问题不是当前一次写放大。
问题是短命旧版本造成了长期结构膨胀。
## 21. 跨模块传播
executor 向 index AM 传 `indexUnchanged` hint。
这个 hint 是 bottom-up deletion 的关键触发信号。
它不保证可删。
它只说明当前 index 的 logical key 没变。
heap AM 负责 visibility 和 HOT chain。
nbtree 不能越过 heap AM 判断 tuple 是否 vacuumable。
table AM 接口让 B-tree deletion 支持其他 table AM。
WAL/recovery 负责 crash 后重放。
redo 不关心 C 函数调用栈。
所以 page mutation 必须被编码成可重放的 record。
standby snapshot conflict 连接 primary cleanup 和 replica query。
`XLOG_BTREE_DELETE` 里的 horizon 可以取消 standby 上仍需要旧 tuple 的查询。
autovacuum/VACUUM 继续承担全局 cleanup。
bottom-up deletion 只处理局部 page pressure。
它不会扫描全 index。
它不会删除 empty subtree。
它不会把 deleted page 放入 FSM。
free space map 只在 page deletion/recycling 之后参与。
leaf tuple deletion 增加 page 内 free space。
page deletion 才产生可回收 index page。
deleted page 放入 FSM 又有额外安全 horizon。
这些是三个不同层次。
## 22. 观测入口
第一类可直接观察：index page 上的 dead flag、posting list 和 TID。
`pageinspect` 的 `bt_page_items()` 可以看到 leaf page item。
现代版本通常能看到 `dead`、`htid` 和 `tids` 列。
`dead` 对应 line pointer 是否 `LP_DEAD`。
`tids` 非空通常说明这是 posting list tuple。
第二类可观察：VACUUM 删除统计。
`VACUUM (VERBOSE)` 会报告 index scan、tuples removed、pages deleted 等信息。
但它不能告诉你某次前台 bottom-up deletion 删除了多少 tuple。
第三类可观察：relation size 和 page count。
`pg_relation_size(index)`、`pg_class.relpages`、`pgstattuple` 或 `pgstatindex()` 能观察 index size 变化。
它们是结果指标，不是因果指标。
第四类可观察：WAL 量。
`EXPLAIN (ANALYZE, BUFFERS, WAL)` 能看到语句级 WAL records/bytes。
`pg_stat_wal` 能看实例累计。
`pg_waldump` 能看到 btree WAL record 类型，但需要对 LSN 范围有控制。
第五类只能推断：bottom-up deletion 是否触发。
没有内置 pg_stat counter 直接显示 `_bt_bottomupdel_pass()` 次数。
你可以从 page split 减少、index size 稳定、WAL 形态、断点或临时日志推断。
第六类几乎不可见：table AM 在 bottom-up pass 中放弃的具体原因。
例如 old snapshot、某个 heap block 没有可删 tuple、target decay 提前停止。
这些需要源码断点或临时 instrumentation。
## 23. 实验一：观察 version churn 与 primary key index 稳定性
准备环境：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE EXTENSION IF NOT EXISTS pgstattuple;

DROP TABLE IF EXISTS btree_churn_lab;
CREATE TABLE btree_churn_lab (
    id integer PRIMARY KEY,
    churn integer NOT NULL,
    payload text NOT NULL
) WITH (autovacuum_enabled = off);

CREATE INDEX btree_churn_lab_churn_idx
ON btree_churn_lab (churn);

INSERT INTO btree_churn_lab
SELECT g, 0, repeat('x', 100)
FROM generate_series(1, 20000) AS g;

VACUUM (ANALYZE) btree_churn_lab;
```
制造非 HOT update：
```sql
UPDATE btree_churn_lab
SET churn = churn + 1
WHERE id BETWEEN 1 AND 5000;

UPDATE btree_churn_lab
SET churn = churn + 1
WHERE id BETWEEN 1 AND 5000;

UPDATE btree_churn_lab
SET churn = churn + 1
WHERE id BETWEEN 1 AND 5000;
```
这里 `churn` 索引列变化，HOT 不成立。
primary key index 的 `id` 没变。
但 primary key index 仍要为新 heap TID 插入 entry。
这会给 primary key leaf pages 制造 physical duplicates。
观察 index 大小：
```sql
SELECT
    pg_size_pretty(pg_relation_size('btree_churn_lab_pkey')) AS pkey_size,
    pg_size_pretty(pg_relation_size('btree_churn_lab_churn_idx')) AS churn_idx_size;

SELECT * FROM pgstatindex('btree_churn_lab_pkey');
```
再执行：
```sql
VACUUM (VERBOSE, ANALYZE) btree_churn_lab;
```
对照 VACUUM 前后 index size 和 `pgstatindex()`。
如果 workload、page layout 和 horizon 允许，primary key index 不会随 update 次数线性膨胀。
这不证明每次都触发 bottom-up deletion。
它只说明 cleanup/dedup/VACUUM 组合阻止了最坏情况。
要进一步确认，需要断点 `_bt_delete_or_dedup_one_page()` 和 `_bt_bottomupdel_pass()`。
## 24. 实验二：用 old snapshot 阻止 deletion，再观察 dedup 的作用
会话 A：
```sql
BEGIN;
SELECT count(*) FROM btree_churn_lab;
```
保持事务不提交。
会话 B：
```sql
UPDATE btree_churn_lab
SET churn = churn + 1
WHERE id BETWEEN 1 AND 5000;

UPDATE btree_churn_lab
SET churn = churn + 1
WHERE id BETWEEN 1 AND 5000;
```
会话 A 的 snapshot 可能让旧 versions 不能 vacuumable。
这时 bottom-up deletion 即使触发，也可能被 heap AM 拒绝。
如果 dedup eligibility 满足，dedup 可能仍把 duplicate entries 合并成 posting list。
观察某些 leaf page：
```sql
SELECT blkno, type, live_items, dead_items, free_size
FROM bt_page_stats('btree_churn_lab_pkey', 1);
```
实际 leaf block 要先通过 `bt_metap()`、`bt_page_stats()` 或扫描 `generate_series()` 找到。
示例：
```sql
SELECT g.blkno, s.live_items, s.dead_items, s.free_size
FROM generate_series(1, 50) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('btree_churn_lab_pkey', g.blkno) AS s
WHERE type = 'l'
ORDER BY free_size
LIMIT 10;
```
查看某个 leaf block 的 posting list：
```sql
SELECT itemoffset, ctid, dead, htid, tids
FROM bt_page_items('btree_churn_lab_pkey', 10)
LIMIT 20;
```
如果 `tids` 非空，说明该 item 是 posting list tuple。
如果 old snapshot 释放后再 `VACUUM`，posting list 中的一部分 TID 可能被删除。
这对应 `_bt_update_posting()` 路径。
实验结论要谨慎：
是否出现 posting list 取决于数据分布、页空间、dedup eligibility 和版本。
没有看到 posting list 不等于 dedup 代码没工作。
可能只是 page pressure 不在你观察的 block 上。
## 25. 实验三：源码断点看主链路
用 debug build 启动 PostgreSQL。
在一个 backend 上设置断点：
```text
break _bt_delete_or_dedup_one_page
break _bt_simpledel_pass
break _bt_bottomupdel_pass
break _bt_delitems_delete_check
break heap_index_delete_tuples
break _bt_dedup_pass
break _bt_delitems_delete
```
执行制造非 HOT version churn 的 UPDATE。
每次命中 `_bt_delete_or_dedup_one_page()` 时记录：
`simpleonly`。
`checkingunique`。
`uniquedup`。
`indexUnchanged`。
`PageGetFreeSpace(page)`。
`insertstate->itemsz`。
进入 `_bt_simpledel_pass()` 时记录：
`ndeletable`。
`deadblocks` 中的 heap block 数。
进入 `_bt_bottomupdel_pass()` 时记录：
`delstate.bottomupfreespace`。
`delstate.ndeltids`。
`promising` entry 数。
进入 `heap_index_delete_tuples()` 时记录：
`delstate->bottomup`。
`nblocksfavorable`。
`nblocksaccessed`。
`actualfreespace`。
返回 `_bt_delitems_delete_check()` 后记录：
最终 `delstate->ndeltids`。
`ndeletable`。
`nupdatable`。
命中 `_bt_dedup_pass()` 时记录：
`state->nintervals`。
是否走 `singlevalstrat`。
这个实验的目标不是背参数。
目标是把一条 UPDATE 造成的 leaf page pressure 还原成：
candidate selection。
heap visibility decision。
page mutation。
WAL record。
split fallback。
## 26. 常见误区
误区一：`LP_DEAD` 等于 index tuple 已删除。
不对。
`LP_DEAD` 只是 line pointer flag。
空间仍在 page 上。
真正删除发生在 `_bt_delitems_delete()`、`_bt_delitems_vacuum()` 或 page split rewrite 等路径。
误区二：`BTP_HAS_GARBAGE` 控制现代 simple deletion。
不对。
当前 heapkeyspace index 的 page-split-avoidance 路径会直接扫描 line pointer array。
该 flag 仍维护，但不是主要 gate。
误区三：bottom-up deletion 是 VACUUM 的轻量版。
不对。
VACUUM 做全 index scan、cleanup lock、empty page deletion 和 deleted page recycling。
bottom-up deletion 是前台 insert path 的局部 backstop。
误区四：dedup 会删除 dead TID。
不对。
dedup 不做 visibility 判断。
它只合并 logical equal 的 index tuples。
删除 posting list 中 dead TID 是后续 deletion/VACUUM 的事情。
误区五：unique index 不会有 duplicate。
逻辑唯一不等于物理没有 duplicate。
MVCC 允许同一 key 的多个 heap TID 暂时共存。
只要没有两个对新 snapshot 可见的 conflicting rows，就不违反唯一性。
误区六：看到 index bloat 就说明 bottom-up deletion 失效。
不一定。
old snapshots、long transactions、replication slots、low locality、dedup disabled、operator class equalimage、fillfactor、update pattern 都会影响。
bottom-up 是 heuristic。
不是强制 compaction。
误区七：前台 deletion 不需要 WAL，因为只是删 dead tuple。
不对。
physical index page mutation 必须 crash safe。
前台 deletion 还要带 snapshot conflict horizon。
误区八：posting list 越大越好。
不对。
nbtree 限制 posting list size。
过大的 posting list 会破坏 split point 选择、page accounting 和 partial deletion 成本。
## 27. 诊断策略
第一步，确认 workload 是否是 version churn。
看 UPDATE 是否频繁。
看是否非 HOT。
看 indexed key 对某些索引是否逻辑不变。
如果更新列被其他索引引用，HOT 可能被阻断。
第二步，确认 long snapshot 是否阻止 cleanup。
查 `pg_stat_activity` 的 long transaction。
查 replication slot 的 xmin。
查 standby feedback。
这些都会让 heap tuple 不能 vacuumable。
第三步，观察 index page 形态。
用 `pageinspect` 看 leaf page 上是否有 `dead` items、posting list、free space。
不要只看 `pg_relation_size()`。
size 是结果。
page item 形态更接近机制。
第四步，对照 VACUUM。
`VACUUM VERBOSE` 能告诉你 index cleanup 是否发生。
但前台 bottom-up deletion 不会以单独统计暴露。
第五步，必要时加源码 instrumentation。
在 `_bt_delete_or_dedup_one_page()` 打计数。
在 `_bt_bottomupdel_pass()` 打 promising 数。
在 `heap_index_delete_tuples()` 打访问 heap block 数和最终 deletable 数。
这比从 SQL 指标硬猜可靠。
第六步，区分 bloat 来源。
append-only key growth 的 split 是正常增长。
random insert 的 split 是 key distribution。
version churn-driven split 才是本节机制主要针对的对象。
## 28. 讨论题
1. 为什么 `LP_DEAD` 可以用较轻的方式设置，但物理删除 index tuple 需要更强的锁和 WAL？
2. 为什么 VACUUM 删除 leaf tuple 前要 full cleanup lock，而前台 bottom-up deletion 通常只需要 exclusive buffer lock？
3. 如果 bottom-up deletion 已经收集了 leaf page 上所有 TID，为什么 table AM 仍然可以只检查其中一部分？
4. 为什么 `indexUnchanged` 只能作为 hint，而不能作为“这些旧 index tuple 一定可删”的证据？
5. posting list 中只有一个 TID dead 时，为什么不能直接把整个 posting tuple 标 `LP_DEAD`？
6. 为什么 dedup 在 old snapshot 阻止 deletion 时仍可能有用？
7. 为什么 `XLOG_BTREE_DELETE` 需要 `snapshotConflictHorizon`，而 `XLOG_BTREE_VACUUM` 不需要同样的字段？
8. 如果你看到 index size 没有下降，如何区分是 page 内空间可复用、empty page 不能回收，还是根本没有发生 deletion？
## 29. 本节小结
本节的核心链路是 insert-time split avoidance。
leaf page 放不下新 tuple 时，nbtree 先删除 `LP_DEAD` tuple。
如果仍不够，并且页面形态像 version churn，就尝试 bottom-up deletion。
bottom-up deletion 用 duplicate interval 和 promising hint 选择候选。
table AM 用 heap visibility、HOT chain 和 horizon 决定真正可删 TID。
如果 deletion 失败，dedup 可能把 duplicate key 的多个 heap TID 压进 posting list。
如果 simple deletion、bottom-up deletion 和 dedup 都失败，系统回到普通 page split。
核心状态不是单个 flag。
`LP_DEAD`、`BTP_HAS_GARBAGE`、posting list、`TM_IndexDeleteOp`、`snapshotConflictHorizon`、vacuum cycle ID 和 page LSN 必须放在生命周期里解释。
ownership 也分层。
`BTDedupState` 和 `TM_IndexDeleteOp` 是 backend-local 临时状态。
leaf page mutation 由 buffer lock 和 critical section 保护。
VACUUM 的 cleanup lock 额外保护 TID recycling。
heap tuple 是否 vacuumable 属于 table AM。
crash safety 属于 WAL/redo。
错误路径的共同模式是 fallback。
不能删就不删。
不能 dedup 就 split。
等待事务就释放锁后重搜。
bottom-up 找不到 deletable TID 就返回。
critical section 内不能普通 ERROR。
观测上，`pageinspect` 能看到 page item 形态。
`VACUUM VERBOSE` 能看到 VACUUM 侧 cleanup。
`pg_relation_size()` 和 `pgstatindex()` 能看结果。
前台 bottom-up deletion 没有完整 SQL 级计数器。
很多结论要靠断点、WAL、pageinspect 和 workload 事实共同推断。
可迁移的系统规律是：
不要把临时 version churn 立即变成长期结构变化。
先用 cheap hint 捕获已知垃圾。
再在局部压力点上用 table-owned visibility 规则做受控 cleanup。
如果 correctness horizon 暂时不允许删除，就用物理压缩买时间。
最终仍保留简单、可靠的结构性 fallback。
