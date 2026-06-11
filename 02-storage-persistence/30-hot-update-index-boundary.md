# PostgreSQL HOT update 与 index 维护边界

## 课程定位

上一节已经把 heap page、line pointer、tuple header 和 `t_ctid` 的基本语义拆开。
本节继续沿着 heap access method 往前走。
前置知识：已经理解 MVCC tuple version、heap page line pointer、buffer content lock、WAL-before-data、visibility map 和普通 btree index scan 的基本行为。

本节唯一主问题：
什么时候 `UPDATE` 可以不为新 tuple version 新增普通索引 entry？
本节围绕的核心矛盾：
索引希望一个 key 能稳定指向 heap 中可见的行版本。
MVCC 又要求 `UPDATE` 生成新 tuple version，而不是原地覆盖旧版本。
如果每次 `UPDATE` 都给每个索引新增 entry，索引膨胀、WAL、随机写和 VACUUM 清理成本都会随更新频率放大。
如果随便省略索引 entry，index scan 就可能找不到最新可见版本，或者用错误 key 找到不该返回的行。
HOT 的折中是：
只在新版本仍在同一 heap page，且所有会持有 tuple TID 的索引所引用的列没有实际变化时，才把新版本做成 heap-only tuple。
旧的 index entry 继续指向 HOT chain 的 root line pointer。
index scan 从 root TID 进入 heap page，再沿 `t_ctid` 和 HOT flags 找到当前 snapshot 可见的版本。
读完本节，你应该能独立判断：
- 一个 `UPDATE` 为什么是 HOT-safe，但最后仍然不是 HOT update。
- 哪些列会阻断 HOT，哪些 summarizing index 只要求更新 summary。
- 为什么 same page 是必要条件，而不只是性能优化。
- `HEAP_HOT_UPDATED` 和 `HEAP_ONLY_TUPLE` 分别标在哪个 tuple version 上。
- root line pointer 为什么不能随旧 root tuple 一起消失。
- `LP_REDIRECT`、`LP_DEAD` 和 `LP_UNUSED` 在 HOT pruning 中承担什么边界。
- index scan 拿到旧 root TID 后，如何找到新版本。
- 为什么 heap scan 不需要沿 HOT chain 追链接。
- HOT update 的 WAL 记录、pruning WAL 记录和 page LSN 分别保护什么。
- abort、broken chain、CREATE INDEX、CREATE INDEX CONCURRENTLY 会在哪些边界上限制 HOT。
- 哪些现象能用 SQL 和 `pageinspect` 直接看到，哪些只能用断点或源码推断。

源码基线：本课使用当前实际源码路径 `/home/highgo/postgres`，branch `master`，commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；核心源码分工见第 3 节。

## 1. 本节在总主线中的位置

一个普通 heap `UPDATE` 能不新增普通索引 entry，需要同时满足三个条件。
第一，新 tuple version 必须放在旧 version 所在的同一个 heap page。
第二，所有 HOT-blocking indexes 引用的列，在旧 tuple 和新 tuple 之间没有变化。
第三，这次更新不能把可达性责任交给索引之外的模块无法理解的状态。
在当前源码里，第二条不是看 SQL 的 `SET` 列表。
它看 `HeapDetermineColumnsInfo()` 对 old/new tuple 的二进制值比较。
`heapam.c:3382-3384` 根据 `interesting_attrs` 得到 `modified_attrs`。
`heapam.c:3972-3992` 在 `newbuf == buffer` 时检查 `modified_attrs` 是否和 `hot_attrs` 有交集。
没有交集，`use_hot_update = true`。
有交集，或者新 tuple 放到别的 page，就不是 HOT。
HOT update 实际写 page 时发生三件关键事。
旧 tuple 被标成 `HEAP_HOT_UPDATED`。
新 tuple 被标成 `HEAP_ONLY_TUPLE`。
旧 tuple 的 `t_ctid` 指向新 tuple 的 TID。
这些动作在 `heapam.c:4029-4060`。
如果是 HOT，`heap_update()` 最后不会要求 executor 为所有索引插入新 entry。
`heapam.c:4159-4167` 返回：

```text
TU_None          不需要更新任何索引
TU_Summarizing   只更新 summarizing indexes
TU_All           更新所有索引
```

`TU_None` 才是最狭义的“不新增 index entry”。
`TU_Summarizing` 是当前 master 的重要细化：
HOT chain 仍然成立，普通 TID-bearing index 不新增 entry，但 BRIN 这类 summarizing index 可能还要更新 summary。
所以本节的问题不能写成“indexed column 不变就不写索引”。
更准确的说法是：
会让索引 entry 直接引用 heap tuple TID 的索引，其引用列必须不变。
summarizing index 不直接持有单个 tuple 的 TID，所以不阻断 HOT chain。
但如果被它 summarizing 的列变化，summary 仍必须更新。
`README.HOT:34-46` 正是用这两个 special cases 解释 HOT 的设计目标。
第一个 special case 是 indexed columns 不变的 repeated update。
第二个 special case 是只影响不包含 tuple IDs、按 block 做 summary 的索引。
`README.HOT:149-157` 明确给出三类结果：
非 summarizing indexed columns 变化，不能 HOT，所有索引更新。
没有 indexed columns 变化，可以 HOT，不更新索引。
只变化 summarizing indexed columns，可以 HOT，但更新 summarizing indexes。
这就是本节的核心边界。

## 2. 核心矛盾与一句话运行模型

HOT 不是把索引更新延后到 VACUUM。
HOT 是在可达性模型上改变了 index entry 的含义。
普通 update 中，新 tuple version 有自己的索引 entry。
index entry 的 TID 直接指向这个 version 的 line pointer。
HOT update 中，新 tuple version 没有自己的普通索引 entry。
它只能通过同一页上的祖先 root tuple 间接到达。
`README.HOT:56-64` 给出这个模型：
没有 HOT 时，update chain 中每个 row version 都有自己的 index entries。
有 HOT 时，同一页上、索引列不变的新 version 不获得新 index entries。
旧 version 标 `HEAP_HOT_UPDATED`。
新 version 标 `HEAP_ONLY_TUPLE`。
旧 version 的 `t_ctid` 链接到新 version。
这个模型接受了一个局部代价：
index scan 找到 root TID 后，可能还要在 heap page 内追 HOT chain。
它换来的是一个全局收益：
重复更新非索引列时，普通索引不增长，索引 WAL 和后续 index vacuum 成本也减少。
这里的矛盾不是“写索引慢，所以少写”这么简单。
真正的问题是：
如果省略新索引 entry，系统必须提供另一条严格正确的查找路径。
这条路径不能依赖重新计算 index expression。
不能依赖用户定义函数仍然 immutable。
不能跨 page 让 index scan 多做不受控 heap fetch。
不能让 VACUUM 在清理旧 tuple 后丢失 index entry 指向 live version 的能力。
HOT 的答案是 page-local update chain 加稳定 root line pointer。

## 3. 核心文件分工与阅读顺序

建议先读 `README.HOT`。
它不是用户文档，而是实现不变量说明。
尤其要读这些段落：

```text
README.HOT:15-32    为什么 page-at-a-time vacuum 不能重算索引 key
README.HOT:53-130   单个 index entry 覆盖一个 HOT chain
README.HOT:136-157  HOT applicability 和 summarizing index 例外
README.HOT:159-181  abort case 和 XMIN/XMAX matching
README.HOT:184-197  index scan 和 sequential scan 的差异
README.HOT:199-274  pruning / defragmentation 时机
README.HOT:301-414  CREATE INDEX / CREATE INDEX CONCURRENTLY 边界
README.HOT:448-520  glossary
```

再读 `htup_details.h`。
这里定义 tuple header flags 的稳定语义。
`htup_details.h:291-296` 定义：

```text
HEAP_KEYS_UPDATED
HEAP_HOT_UPDATED
HEAP_ONLY_TUPLE
```

`htup_details.h:519-560` 定义 HOT/heap-only 的 accessor。
这里最容易忽略的是：
`HeapTupleHeaderIsHotUpdated()` 不只是检查 bit。
它还要求 `HEAP_XMAX_INVALID` 没有被设置，并且 tuple xmin 没有 invalid。
也就是说，一个 aborted update 不应该继续被当成有效 HOT link。
第三读 `heapam.c` 的 `heap_update()`。
它是本节的主入口。
核心行段：

```text
heapam.c:3286-3297   取 HOT-blocking、summarized、key、identity attr bitmap
heapam.c:3382-3384   计算 modified_attrs
heapam.c:3972-3992   same-page HOT eligibility
heapam.c:4029-4060   标 HOT flags 并设置 t_ctid
heapam.c:4097-4107   写 heap update WAL 并设置 page LSN
heapam.c:4141        统计 HOT update / new page update
heapam.c:4159-4167   返回 TU_None / TU_Summarizing / TU_All
heapam.c:4360-4438   比较 old/new tuple 的实际列值
heapam.c:8775-8988   log_heap_update()
```

第四读 `relcache.c` 的 `RelationGetIndexAttrBitmap()`。
它回答“indexed columns”到底怎么得来。
`relcache.c:5280-5287` 描述各类 bitmap。
`relcache.c:5363-5370` 说明 HOT-safety 要考虑所有 `RelationGetIndexList()` 返回的索引，即便它们还不是 `indisready` 或 `indisvalid`。
这和 `CREATE INDEX CONCURRENTLY` 的正确性直接相关。
`relcache.c:5427-5435` 把 summarizing index 和 HOT-blocking index 分开。
`relcache.c:5437-5479` 收集 simple index columns、INCLUDE columns、expression columns 和 predicate columns。
第五读 executor 和 index AM 边界。
`nodeModifyTable.c:2608-2617` 根据 `updateIndexes` 决定是否调用 `ExecInsertIndexTuples()`。
`execIndexing.c:291-297` 说明 `EIIT_ONLY_SUMMARIZING` 只更新 `amsummarizing` indexes。
`execIndexing.c:449-457` 最终调用 `index_insert()`。
`indexam.c:214-234` 把插入委派给具体 AM 的 `aminsert`。
第六读 index scan 回 heap。
`indexam.c:598-635` 让 index AM 返回一个 TID。
`indexam.c:657-664` 通过 table AM fetch heap tuple。
`tableam.h:1280-1302` 说明 `table_index_fetch_tuple()` 可以把一个 index entry 映射到多个 row versions。
`heapam_indexscan.c:89-229` 的 `heap_hot_search_buffer()` 是 HOT chain 查找核心。
最后读 `pruneheap.c`。
它回答旧 root tuple 死亡后，为什么 root line pointer 还必须存在。
核心行段：

```text
pruneheap.c:242-360    opportunistic pruning 入口和 cleanup lock
pruneheap.c:542-645    一次性分类 root items 和 heap-only items
pruneheap.c:662-696    aborted heap-only tuple 与错误边界
pruneheap.c:1452-1480  HOT chain pruning 的不变量说明
pruneheap.c:1504-1621  追 chain 并根据 HTSV 决定 pruning
pruneheap.c:2091-2167  LP_REDIRECT / LP_DEAD 的断言边界
pruneheap.c:2228-2268  page_verify_redirects()
pruneheap.c:2273-2391  heap_get_root_tuples()
```

## 4. 关键状态与不变量

HOT 的核心状态不在一个单独结构体里。
它分散在四层：
heap tuple header flags。
`t_ctid`。
line pointer state。
executor 和 relcache 的 index-maintenance decision。
第一层是 tuple header flags。
`HEAP_HOT_UPDATED` 标在旧 tuple version 上。
它的语义是：
这个 tuple 被 HOT-updated，`t_ctid` 指向的后继是 heap-only tuple。
`HEAP_ONLY_TUPLE` 标在新 tuple version 上。
它的语义是：
没有普通 index entry 直接指向这个 tuple。
它必须通过祖先 root line pointer 间接到达。
这两个 flag 不能反着理解。
旧 tuple 不是 heap-only。
新 tuple 不是 HOT-updated，除非它之后又作为中间节点被 HOT-updated。
第二层是 `t_ctid`。
普通 live tuple 的 `t_ctid` 通常指向自己。
update 后，旧 tuple 的 `t_ctid` 指向新 tuple。
HOT update 中，这个 link 必须留在同一 page。
`heapam_indexscan.c:215-221` 在追 HOT chain 时断言 `t_ctid` 的 block number 仍是当前 block。
cold update 可以把 `t_ctid` 指向别的 page。
但 cold update 的旧 tuple 不设置 `HEAP_HOT_UPDATED`。
所以 index scan 不会靠旧索引 entry 去跨页追新 tuple。
新 tuple 自己有新 index entry。
第三层是 line pointer。
root line pointer 是 index entry 指向的 `(block, offset)`。
只要 HOT chain 中还有任何非 dead 成员，root line pointer 就不能被随便复用。
旧 root tuple 的 tuple bytes 可以被 pruning 回收。
但是 root line pointer 可能要变成 `LP_REDIRECT`，继续把 index entry 引到链内的 live heap-only tuple。
`README.HOT:82-111` 用 root tuple 被替换成 redirect 的图解释了这个边界。
当整个 HOT chain 都 dead 后，root line pointer 可以变成 `LP_DEAD`。
它仍不能立即 `LP_UNUSED`，因为索引里可能还有 entry 指向它。
后续 regular VACUUM 清理索引 entry 后，line pointer 才能真正复用。
heap-only tuple 的 line pointer 不应该变成 `LP_DEAD`。
因为没有索引 entry 直接指向 heap-only tuple。
`pruneheap.c:2117-2122` 强调 `LP_REDIRECT` 是为了让 VACUUM 知道该从索引删除哪个 root TID。
同一段还说明 heap-only tuple 不能成为 `LP_DEAD`。
第四层是 `TU_UpdateIndexes`。
这是 heap AM 返回给上层 executor 的维护边界。
`tableam.h:132-141` 定义：

```text
TU_None          no indexed columns updated, TID addressing unchanged
TU_All           non-summarizing indexed column changed, or TID changed
TU_Summarizing   only summarized columns changed, TID unchanged
```

注意这里的 “TID unchanged” 不是说新 tuple 的 physical TID 没变。
UPDATE 总是生成新 tuple version。
它指的是普通索引 entry 用来定位逻辑行的 root TID 不需要改变。
HOT chain 让 index entry 的 TID 继续是 root TID。
新 tuple version 的 `t_self` 当然是新的 offset。
这个差异是理解 HOT 的关键。

## 5. HOT eligibility 的源码判断

`heap_update()` 开始时先准备 attr bitmap。
`heapam.c:3286-3292` 取四组 bitmap：

```text
hot_attrs  = INDEX_ATTR_BITMAP_HOT_BLOCKING
sum_attrs  = INDEX_ATTR_BITMAP_SUMMARIZED
key_attrs  = INDEX_ATTR_BITMAP_KEY
id_attrs   = INDEX_ATTR_BITMAP_IDENTITY_KEY
```

`hot_attrs` 服务 HOT eligibility。
`sum_attrs` 服务 summarizing index 维护。
`key_attrs` 服务 tuple lock mode 和外键并发。
`id_attrs` 服务 replica identity WAL/logical decoding。
它们相关，但不能混为一谈。
一个 INCLUDE column 不是 btree key column。
它通常不影响外键 key lock mode。
但它仍然存储在索引 tuple 里。
所以它变化时会阻断 HOT。
`relcache.c:5442-5450` 专门说明 covering indexes 的 non-key columns 必须加入 HOT-blocking 或 summarized bitmap。
表达式索引更保守。
`relcache.c:5475-5479` 对 index expressions 和 index predicate 调 `pull_varattnos()`。
这意味着：

```sql
CREATE INDEX ON t ((lower(name)));
CREATE INDEX ON t (id) WHERE active;
```

`name` 和 `active` 都可能阻断 HOT。
即使 `active` 不存储在索引 tuple 中，只出现在 partial index predicate 中，也必须视为 indexed column。
原因在 `README.HOT:18-32`。
page-at-a-time vacuum 不应该通过重跑用户函数或 predicate 来找旧索引 entry。
函数可能被错误标记 immutable。
predicate 和表达式也可能依赖复杂语义。
HOT 选择保守地在 update 时阻断风险，而不是在 cleanup 时冒险重算 key。
`RelationGetIndexAttrBitmap()` 还有一个并发边界。
`relcache.c:5363-5370` 说 HOT-safety 要考虑所有 `RelationGetIndexList()` 返回的索引。
即使索引还没有 ready for inserts 或 valid for searches，也要考虑。
这正是 `CREATE INDEX CONCURRENTLY` 所需。
如果一个正在创建的新索引还没有参与 HOT-safety，别的事务可能把新索引列从 `X=1` HOT-update 到 `X=2`。
构建进程已经为 `X=1` 生成了 root TID entry。
之后 live tuple 却变成 `X=2`。
这个错误 entry 没有好办法局部删除。
所以并发建索引必须先让后续事务知道这个索引会阻断 HOT。
`README.HOT:368-386` 用这个例子解释 `CREATE INDEX CONCURRENTLY` 的等待和 catalog state。
拿到 bitmap 后，`heap_update()` 计算实际变化列。
`heapam.c:3382-3384` 调 `HeapDetermineColumnsInfo()`。
这个函数只看 `interesting_attrs`，即 HOT、summary、key、identity 相关列的并集。
它不是把整个 row 全量 deform 后逐列比较。
`heapam.c:4360-4438` 是核心。
它对每个 interesting attr 取 old/new value。
然后用 `heap_attr_equals()` 做比较。
`heapam.c:4325-4333` 解释为什么用二进制比较。
调用 datatype-specific equality operator 有两个问题。
第一，一个 datatype 在不同 opclass 里可能有不同 equality 语义。
第二，不能在持有 exclusive buffer lock 的路径上随便调用用户定义函数。
所以 HOT 用 bitwise equality 做保守判断。
这带来一个诊断结论：
SQL 层认为“逻辑相等”的更新，不一定 HOT-safe。
如果二进制表示不同，`modified_attrs` 会包含该列。
相反，SQL `SET indexed_col = indexed_col` 不一定阻断 HOT。
如果最终 old/new datum 二进制一致，它不会进入 `modified_attrs`。
HOT eligibility 的实际判断在 `heapam.c:3972-3992`。
伪流程是：

```text
if newbuf == buffer:
    if modified_attrs does not overlap hot_attrs:
        use_hot_update = true
        if modified_attrs overlaps sum_attrs:
            summarized_update = true
else:
    PageSetFull(old_page)
```

这段代码同时说明两个边界。
只有 same page 才“可能 HOT”。
没有 HOT-blocking attr 变化才“允许 HOT”。
只影响 summarizing attrs 时，`use_hot_update` 仍然是 true。
但 `summarized_update` 会让上层只更新 summarizing indexes。
如果 `newbuf != buffer`，就算所有 indexed columns 都没变，也不是 HOT。
这时旧 tuple 可能通过普通 update chain 指向新 tuple。
但旧 tuple 不设置 `HEAP_HOT_UPDATED`。
新 tuple 不设置 `HEAP_ONLY_TUPLE`。
上层会得到 `TU_All`，为新 tuple 插入全部 index entries。
这就是 HOT-safe 和 actual HOT update 的区别。
HOT-safe 描述列变化边界。
actual HOT update 还要求物理落点在同一页。

## 6. 主流程源码 walkthrough

从 executor 看，UPDATE 的主线可以压缩成：

```text
ExecUpdateAct()
  -> table_tuple_update()
     -> heapam_tuple_update()
        -> heap_update()
           -> 判断 HOT / 写 heap page / 返回 TU_UpdateIndexes
  -> ExecUpdateEpilogue()
     -> 如果 updateIndexes != TU_None:
          ExecInsertIndexTuples()
```

`heapam_handler.c:223-240` 把 table AM API 接到 `heap_update()`。
`heapam_handler.c:243-262` 对 `heap_update()` 返回的 `update_indexes` 做断言。
如果新 tuple 不是 heap-only，必须是 `TU_All`。
如果新 tuple 是 heap-only，只能是 `TU_None` 或 `TU_Summarizing`。
进入 `heap_update()` 后，先做并发和可见性处理。
`heapam.c:3313-3315` 拿旧 page 的 exclusive buffer lock 并定位 old line pointer。
如果 line pointer 已经不是 `LP_NORMAL`，`heapam.c:3340-3359` 对 syscache 特殊入口返回 `TM_Deleted`。
这个路径不是普通 SQL UPDATE 的常见路径。
普通 SQL UPDATE 持有 snapshot，通常能保证 old TID 仍是 normal tuple。
`heapam.c:3431` 调 `HeapTupleSatisfiesUpdate()` 判断旧 tuple 是否可更新。
如果遇到并发 locker/updater，`heapam.c:3443-3618` 会等待、保留 locker 信息、必要时 `goto l2` 重新检查。
这个阶段还不是 HOT 判断。
它先回答“这条 old tuple version 能不能被我更新”。
之后才回答“新 version 是否能省索引 entry”。
`heapam.c:3382-3384` 在锁住旧 tuple 之后计算 `modified_attrs`。
这个顺序很微妙。
源码注释 `heapam.c:3270-3278` 说 attr bitmap 必须在 buffer lock 前取。
最坏情况下，如果更新的是系统目录，在 buffer lock 下再去取 relcache/index 信息可能死锁。
所以它先从 relcache 拿各类 bitmap。
再在 old tuple 可访问时比较 old/new value。
接下来 `heap_update()` 为新 tuple 选择落点。
这部分代码分散在 `heapam.c:3663-3945`，本节不逐行展开。
对 HOT 来说，只需要抓住最终变量：

```text
buffer  = old tuple page
newbuf  = new tuple page
heaptup = 实际要写入 page 的 tuple
```

`heapam.c:3972` 之后，`newbuf == buffer` 才能进入 HOT 判断。
如果 `newbuf != buffer`，`heapam.c:3995-3998` 对 old page 设置 full hint。
这会让后续访问更积极尝试 pruning/defragmentation。
它不是 correctness 状态，只是空间管理 hint。
真正修改 heap page 之前，`heapam.c:4000-4010` 先提取 replica identity tuple。
注释说明这是为了避免进入 critical section 后再发生内存分配失败。
`heapam.c:4012-4013` 进入 critical section。
从这里到 WAL 记录完成之间不能 `ereport(ERROR)`。
因为 heap page 已经会被修改。
在 critical section 内，`heapam.c:4025-4027` 对 old/new page 设置 prunable hint。
如果事务提交，旧 tuple 迟早会 dead。
如果事务 abort，后续 pruning 会 no-op 或清理 aborted tuple。
`heapam.c:4029-4044` 根据 `use_hot_update` 设置或清理 HOT flags。
HOT 时：

```text
old tuple: HeapTupleSetHotUpdated()
new tuple: HeapTupleSetHeapOnly()
caller copy: HeapTupleSetHeapOnly()
```

非 HOT 时：

```text
old tuple: HeapTupleClearHotUpdated()
new tuple: HeapTupleClearHeapOnly()
caller copy: HeapTupleClearHeapOnly()
```

这里清理 flags 也很重要。
tuple 可能来自上层私有副本或重用路径。
非 HOT 更新必须显式保证新旧 tuple 不带 HOT 语义。
`heapam.c:4046` 调 `RelationPutHeapTuple()` 把新 tuple 放进 `newbuf`。
随后 `heapam.c:4050-4057` 更新 old tuple 的 xmax、infomask 和 cmax。
`heapam.c:4059-4060` 把 old tuple 的 `t_ctid` 设置为 new tuple 的 `t_self`。
这一步对 HOT 和 cold update 都发生。
区别是：
HOT 旧 tuple 还有 `HEAP_HOT_UPDATED`。
cold 旧 tuple 没有。
index scan 只会在看到 HOT-updated tuple 时沿 `t_ctid` 追 heap-only 后继。
`heapam.c:4062-4076` 清理 visibility map 的 all-visible bits。
UPDATE 产生了新版本和旧版本的可见性变化。
即使是 HOT，它也破坏了 page all-visible 的条件。
所以 index-only scan 后续可能要重新访问 heap，直到 VACUUM/pruning 重新设置 VM。
`heapam.c:4078-4080` 标 dirty buffer。
`heapam.c:4082-4107` 写 WAL。
`log_heap_update()` 根据新 tuple 是否 heap-only 选择 WAL info。
`heapam.c:8799-8802`：

```text
HeapTupleIsHeapOnly(newtup) ? XLOG_HEAP_HOT_UPDATE : XLOG_HEAP_UPDATE
```

如果 old/new 在同一 page，并且不需要 logical tuple data，也不需要 full-page image，`heapam.c:8824-8854` 会记录新旧 tuple payload 的 common prefix/suffix 以减少 WAL。
这不是 HOT correctness 的核心。
但它解释了为什么 same-page update 在 WAL 上还有额外优化空间。
`heapam.c:4103-4107` 把 WAL LSN 设置到 new page 和 old page。
同一 page 时只设置一次。
`heapam.c:4110` 结束 critical section。
之后释放 buffer locks、释放 pins、释放 tuple lock。
`heapam.c:4141` 统计 heap update。
这个调用会把 HOT update 和 new page update 计入 pgstat。
最后 `heapam.c:4159-4167` 返回 index maintenance decision。
伪代码是：

```text
if use_hot_update:
    if summarized_update:
        update_indexes = TU_Summarizing
    else:
        update_indexes = TU_None
else:
    update_indexes = TU_All
```

上层 `ExecUpdateEpilogue()` 只看这个结果。
`nodeModifyTable.c:2608-2617` 如果 `TU_None`，不调用 `ExecInsertIndexTuples()`。
如果 `TU_Summarizing`，设置 `EIIT_ONLY_SUMMARIZING`。
`execIndexing.c:373-377` 在 only summarizing 模式下跳过非 summarizing indexes。
`execIndexing.c:449-457` 对剩下的索引调用 `index_insert()`。
`indexam.c:214-234` 再调用具体 index AM 的 `aminsert`。
这就是 heap 和 index 的责任边界。
heap AM 判断“是否需要新增索引 entry”。
executor 负责对需要维护的索引形成 index datum。
index AM 负责具体插入、唯一性检查和 WAL。

## 7. index scan 如何找到新版本

HOT 的正确性最终要在 index scan 中兑现。
普通 index scan 的第一步不懂 HOT。
`indexam.c:598-615` 调具体 index AM 的 `amgettuple()`。
index AM 找到满足 scan key 的 index entry，把 TID 放到 `scan->xs_heaptid`。
这个 TID 是 root line pointer。
它可能指向一个普通 tuple。
也可能指向一个已经被 pruning 成 `LP_REDIRECT` 的 root line pointer。
`indexam.c:657-664` 随后调用：

```text
table_index_fetch_tuple(scan->xs_heapfetch,
                        &scan->xs_heaptid,
                        scan->xs_snapshot,
                        slot,
                        &scan->xs_heap_continue,
                        &all_dead)
```

`tableam.h:1280-1302` 对这个 API 的注释非常关键。
它说 `tid` 在返回 true 时可能被修改。
原因是一个 index entry 可能通过 heap AM 的 HOT 支持到达多个 row versions。
这和 `table_tuple_fetch_row_version()` 不同。
后者只检查给定 TID 上的那个 tuple。
heap AM 的实现是 `heapam_index_fetch_tuple()`。
`heapam_indexscan.c:244-263` 先保证对应 heap block 的 buffer pinned。
第一次 pin 一个 page 时，会调用 `heap_page_prune_opt()` 做 opportunistic pruning。
这不是 index scan correctness 的必需步骤。
它是顺手缩短 HOT chain、回收空间、修复 VM 状态的维护入口。
然后 `heapam_indexscan.c:268-276` 对 heap buffer 加 share lock，并调用 `heap_hot_search_buffer()`。
`heap_hot_search_buffer()` 的入口约定是：
`*tid` 是 simple tuple 或 HOT chain root 的 TID。
buffer 已经 pinned 且至少 share locked。
函数寻找第一个符合 snapshot 的 chain member。
找到后，把 `*tid` 改成实际可见 tuple 的 offset。
第一条边界在 `heapam_indexscan.c:127-139`。
如果 line pointer 不是 normal：
只有 chain start 位置允许看到 `LP_REDIRECT`。
遇到 root redirect，就跳到 redirect target。
如果非 root 位置看到 unused、dead 或 redirected，就认为 chain 结束。
这和 `README.HOT:171-176` 的 abort/broken chain 规则一致。
追 HOT link 时，不能假设 line pointer 还保留原 tuple。
它可能已经被 abort cleanup 或 pruning 改掉。
第二条边界在 `heapam_indexscan.c:153-157`。
chain start 不应该是 heap-only tuple。
如果 index entry 直接指向 heap-only tuple，说明 root TID 不变量被破坏。
函数会停止，而不是把它当正常 chain。
第三条边界在 `heapam_indexscan.c:160-166`。
当前 tuple 的 xmin 必须等于上一跳 tuple 的 update xid。
也就是：

```text
prev_xmax == current_xmin
```

这个检查防止链路被 pruning、abort 或 line pointer reuse 后误连到无关 tuple。
只靠 buffer pin 不够。
`README.HOT:179-181` 特别指出，早期 HOT 代码曾假设 page pin 能防住这类问题。
现在更稳妥的规则是 XMIN/XMAX matching。
第四条边界是 visibility。
`heapam_indexscan.c:177-189` 对当前 tuple 调 `HeapTupleSatisfiesVisibility()`。
如果当前 snapshot 可见，就更新 `tid` 的 offset，并返回 true。
对 MVCC-like snapshot，HOT chain 中最多一个 tuple 可见。
`heapam_indexscan.c:280-287` 因此在找到一个 tuple 后把 `heap_continue` 设成 false。
对非 MVCC snapshot，可能需要继续调用来拿 chain 中其他可见版本。
第五条边界是继续追链。
`heapam_indexscan.c:211-224` 只有 `HeapTupleIsHotUpdated(heapTuple)` 为 true 才继续。
它断言后继仍在同一个 block。
然后取 `t_ctid` 的 offset，记录 `prev_xmax`，循环。
如果不是 HOT-updated，就到 chain 末尾。
这个流程说明：
index entry 没有直接指向新 version。
但 index scan 仍能找到新 version。
可达性由 root TID、HOT flags、`t_ctid`、same-page 和 XMIN/XMAX matching 共同保证。
sequential scan 为什么不需要追 HOT chain？
`README.HOT:194-196` 给出答案。
seq scan 扫 page 上每个 line pointer。
heap-only tuple 本身也在 page 上。
它会被当作普通 tuple version 做 visibility check。
如果它对 snapshot 可见，seq scan 能直接看到。
因此 HOT 的额外追链成本主要落在 index scan。

## 8. root line pointer 与 pruning 边界

HOT 能省 index entry，代价是 root line pointer 必须长期承担 index 可达性。
当 root tuple bytes 已经对所有事务 dead 时，不能直接把 root line pointer 变成 `LP_UNUSED`。
因为索引 entry 还指向这个 offset number。
如果复用这个 offset 给无关 tuple，旧索引 entry 就可能指向错误行。
HOT pruning 的核心任务是缩短 chain，同时保持 root TID 语义。
`pruneheap.c:242-360` 的 `heap_page_prune_opt()` 是常见入口。
它是 opportunistic。
调用者必须已经 pin buffer，但不能持有 buffer lock。
函数先看 `pd_prune_xid`。
如果没有可能可剪枝的 XID，快速返回。
如果 XID horizon 显示可能可剪枝，再看 page 是否 full 或 free space 低于 fillfactor/10% 阈值。
然后尝试 `ConditionalLockBufferForCleanup()`。
拿不到 cleanup lock 就放弃。
这解释了一个运行时现象：
HOT chain 可能变长。
不是每次 index scan 或 update 都一定 pruning。
cleanup lock 拿不到，或者启发式不满足，系统宁可推迟维护。
`pruneheap.c:542-645` 先扫描 page，把 items 分成 root items 和 heap-only items。
非 heap-only normal tuple 和 `LP_REDIRECT` 都进入 root items。
heap-only tuple 进入 heaponly items。
然后对每个 root 调 `heap_prune_chain()`。
`pruneheap.c:1452-1480` 给出 HOT pruning 的不变量说明。
如果 item 是 index-referenced tuple，也就是非 heap-only tuple，就从 chain 开头移除 dead tuples。
root line pointer 会 redirect 到最后一个 dead tuple 之后的第一个 live successor。
如果整个 chain 都 dead，root line pointer 变成 `LP_DEAD`。
pruning 不应该留下仍有 tuple storage 的 DEAD tuple。
VACUUM 不准备处理这种状态。
`heap_prune_chain()` 追 chain 的逻辑和 index scan 类似。
`pruneheap.c:1553-1558` 检查 `priorXmax` 和当前 tuple xmin。
`pruneheap.c:1607-1621` 只有 HOT-updated tuple 才继续沿 `t_ctid` 追后继。
这说明 pruning 和 index scan 共享同一类 chain validity 规则。
`pruneheap.c:2091-2134` 在 assertion build 中检查 redirect 的形状。
被设置为 `LP_REDIRECT` 的 from item 必须是 HOT chain 第一个 item。
如果 from item 仍有 tuple storage，它不能是 heap-only。
redirect target 必须是 normal item，并且 target tuple 必须是 heap-only。
每条 HOT chain 最多一个 redirect item。
`pruneheap.c:2146-2167` 检查 `LP_DEAD` 的形状。
如果一个要变成 `LP_DEAD` 的 item 仍有 tuple storage，它不能是 heap-only。
如果没有 tuple storage，则说明整个 HOT chain dead，原来应该是 redirect root。
`pruneheap.c:2228-2268` 的 `page_verify_redirects()` 在 pruning 后整页检查所有 redirects。
每个 redirect target 必须 used、normal、有 storage，并且是 heap-only tuple。
这类断言不是普通防御式编程。
它对应一个真实错误边界：
如果 redirect 指向被移除或复用的 tuple，index entry 会经 root TID 找到无关内容。
这就是 HOT 最危险的 corruption 形态之一。
`pruneheap.c:662-696` 还处理 aborted heap-only tuple。
如果 heap-only tuple 是 DEAD 且没有继续 chain，可以直接 unused。
这主要处理 aborted HOT updates。
但如果一个 DEAD heap-only tuple 仍标 HOT-updated，却没有从任何 chain 链到，源码选择 `elog(ERROR)`。
注释说这是为了保留证据，也避免留下 VACUUM 无法处理的 DEAD tuple。
这个路径说明 HOT 的错误边界很硬：
heap-only tuple 必须属于某个 chain，或者作为 aborted 残留被安全移除。
不能把它伪装成普通 index-referenced tuple。
pruning 的实际 page 变化在 `heap_page_prune_execute()`。
`pruneheap.c:2054-2063` 说明如果不是只截断 `LP_DEAD`，调用者必须持有 cleanup lock。
`pruneheap.c:1294-1310` 在 page 修改后写 `XLOG_HEAP2_PRUNE*` WAL 记录。
所以 HOT update 和 HOT pruning 是两个不同生命周期阶段。
前者创建 chain。
后者缩短 chain、redirect root、释放 heap-only line pointers。
两者都修改 heap page，都需要 WAL 保证 crash recovery。

## 9. CREATE INDEX 为什么是 HOT 边界

HOT 的一个复杂边界来自新增索引。
已有 HOT chain 对旧索引是安全的。
因为旧索引引用的列在 chain 内没有变化。
但新建索引可能引用一个在已有 HOT chain 内发生过变化的列。
这样 chain 对新索引就是 broken HOT chain。
`README.HOT:304-329` 解释 regular CREATE INDEX 的方案。
新索引 entry 指向 HOT chain 的 root TID。
但用于生成 index key 的值来自 chain 末端的最新 live tuple。
这对未来能使用该索引的事务是安全的。
因为这些事务不能看到较老、不匹配 key 的 tuple versions。
问题是旧 snapshot。
旧 snapshot 可能还能看到 chain 中更早的 tuple。
如果它使用新索引，就可能用新 key 找到一个自己可见但 key 不匹配的旧 tuple。
PostgreSQL 用 `pg_index.indcheckxmin` 处理这个边界。
`README.HOT:341-353` 说明，当 `indcheckxmin` 为 true 时，只有当 `pg_index.xmin` 低于事务的 `TransactionXmin` horizon，事务才允许使用该索引。
`index.c:3118-3167` 在 non-concurrent CREATE INDEX 遇到 potential broken HOT chains 时设置 `indcheckxmin`。
这不是 HOT update 主流程的一部分。
但它说明 HOT 的可达性设计泄漏到了 catalog index validity 规则。
`CREATE INDEX CONCURRENTLY` 的路径更严格。
`README.HOT:368-386` 说，它先创建 `pg_index` entry，但标记为 not ready for inserts。
然后提交并等待已经打开表的事务结束。
这样后续事务在 HOT-safety 判断时已经能看到新索引。
它们会把新索引列视为 HOT-blocking。
即使这个索引还不能用于查询或插入，也必须参与 HOT-safety。
这正是 `relcache.c:5363-5370` 的原因：
HOT-safety 要考虑 not indisready/not indisvalid 的索引。
`DROP INDEX CONCURRENTLY` 则相反。
当索引进入不应再被任何事务触碰的阶段，`indislive` 会被清掉。
不 live 的索引不再由 `RelationGetIndexList()` 返回。
于是也不再参与 HOT-safety。
这里的可迁移规律是：
索引是否可用于 read、是否 ready for inserts、是否 live、是否参与 HOT-safety，是四个不同问题。
不要把 `indisvalid` 当成“这个索引对所有维护路径都不存在”。

## 10. 生命周期 / ownership / cleanup

HOT chain 的创建者是执行 UPDATE 的 backend。
它在 `heap_update()` 中持有 old page 和 new page 的 buffer pin。
修改 page 时持有 exclusive buffer content lock。
需要等待 tuple locker/updater 时，会释放 buffer lock，持有或获取 tuple lock，再回来重新检查。
page bytes 的 owner 是 buffer manager 和 storage manager。
`heap_update()` 不拥有一个长期 heap object。
它只在 critical section 中修改 page image，并通过 WAL 让修改可恢复。
索引 entry 的 owner 是各 index AM。
HOT update 选择不创建普通新 index entry。
这不是给 index AM 留一个 pending task。
`TU_None` 的含义是这次 UPDATE 后普通索引不需要维护。
可达性已经由 heap page 内状态承担。
root line pointer 的生命周期跨越多个 tuple version。
它最初是普通 `LP_NORMAL`，指向 root tuple bytes。
当 root tuple dead 但 chain 还有 live 后继时，pruning 可把它变成 `LP_REDIRECT`。
当整条 chain dead 时，可变成 `LP_DEAD`。
只有 index vacuum 清掉对应 index entries 后，才能最终 `LP_UNUSED`。
heap-only tuple 的生命周期更短。
它没有直接 index entry。
一旦对所有事务 dead，且 pruning 能安全处理 chain，就可以把它的 line pointer 变成 `LP_UNUSED`。
它不需要经历 `LP_DEAD`。
ERROR/abort 的 cleanup 分两层。
事务 abort 会让插入的新 heap-only tuple 的 xmin invalid。
`HeapTupleHeaderIsHotUpdated()` 对 old tuple 的 hot link 也会因为 update xmax invalid 而不再认为有效。
后续 pruning 可以移除 aborted heap-only tuple。
如果 crash 发生在 heap page 修改期间，critical section 和 WAL 规则保证不会留下只改 page 不写 WAL 的状态。
`heapam.c:4012-4110` 把 page 修改、WAL register/insert、page LSN 放在同一 critical section。
`CacheInvalidateHeapTuple()` 在 `heapam.c:4116-4124` 处理系统缓存 invalidation。
这和 HOT correctness 不是同一层。
它保证 catalog tuple update 后 relcache/syscache 语义过期能传播。
不要把 cache invalidation 当成索引可达性机制。
Resource cleanup 是常规 buffer/lock cleanup。
`heapam.c:4112-4139` 解锁并释放 buffers、visibility map buffers 和 tuple lock。
`heapam.c:4169-4177` 释放 old replica identity tuple 和 bitmapsets。
如果中途返回非 `TM_Ok`，`heapam.c:3639-3660` 也会释放 buffer、tuple lock、bitmapsets，并设置 `update_indexes = TU_None`。
失败 update 不应该让 executor 插入 index entries。

## 11. 正确性机制层次

HOT 正确性不是一个 flag 保证的。
它至少依赖八层机制。
第一层是 MVCC visibility。
`HeapTupleSatisfiesUpdate()` 决定旧 tuple 能否被当前事务更新。
`HeapTupleSatisfiesVisibility()` 决定 index scan 或 heap scan 看到哪个 version。
visibility 不提供物理互斥。
第二层是 tuple lock 和 transaction wait。
`heap_update()` 在遇到 concurrent locker/updater 时等待。
如果等待期间 tuple xmax 改了，源码回到 `l2` 重新检查。
这保证 update 冲突不会在旧状态上继续执行。
tuple lock 不保证 HOT eligibility。
它只保证更新并发顺序。
第三层是 buffer pin。
pin 保证 page buffer 不被 evict。
它不保证 line pointer 不被 pruning 成 redirect/dead/unused。
HOT chain 追踪必须用 content lock 和 XMIN/XMAX matching。
第四层是 buffer content lock。
UPDATE 修改 page 时持 exclusive lock。
index fetch 检查 HOT chain 时持 share lock。
pruning 需要 cleanup lock 或特定场景下 exclusive lock。
content lock 保护 page memory 的并发访问。
它不表达事务可见性。
第五层是 root TID invariant。
普通索引 entry 指向 root line pointer。
root line pointer 在 chain live 时不得复用给无关 tuple。
这个不变量由 HOT flags、line pointer states 和 pruning 规则共同维护。
第六层是 XMIN/XMAX matching。
追 chain 时必须检查前一跳 update xid 等于当前 tuple xmin。
这抵抗 abort、pruning 和 line pointer reuse 造成的错连。
第七层是 WAL。
HOT update 用 `XLOG_HEAP_HOT_UPDATE` 或 `XLOG_HEAP_UPDATE` 重放 heap page 修改。
pruning 用 `XLOG_HEAP2_PRUNE*` 重放 redirect/dead/unused 和 freeze 相关动作。
WAL 保证 crash recovery 后 page 状态和 LSN 顺序一致。
它不保证查询一定低延迟。
第八层是 relcache/catalog index state。
`RelationGetIndexAttrBitmap()` 的结果决定 HOT-blocking attrs。
`indisready`、`indisvalid`、`indislive`、`indcheckxmin` 决定新索引在维护和查询上的不同边界。
这是 HOT 和 catalog/index build 的跨模块连接。

## 12. 错误路径 / fallback

第一类 fallback：same page 空间不足。
即使列变化 HOT-safe，只要新 tuple 不能放在 old page，`newbuf != buffer`。
`heapam.c:3972-3998` 不会设置 `use_hot_update`。
`heapam.c:4166-4167` 返回 `TU_All`。
executor 为新 tuple 插入所有索引 entry。
运行时能看到 `n_tup_newpage_upd` 增加。
第二类 fallback：HOT-blocking attr 变化。
只要 `modified_attrs` 和 `hot_attrs` 有交集，就不是 HOT。
这包括 btree key columns、INCLUDE columns、expression referenced columns、partial predicate columns，以及任何非 summarizing index AM 的相关列。
这时即使新 tuple 在同一 page，也返回 `TU_All`。
第三类特殊路径：只影响 summarizing attrs。
`use_hot_update = true`，但 `summarized_update = true`。
返回 `TU_Summarizing`。
executor 只维护 `ii_Summarizing` 的 indexes。
普通 TID-bearing indexes 不新增 entry。
第四类错误边界：chain link 不可信。
index scan 或 pruning 追 HOT chain 时，遇到非 normal line pointer、xmin/xmax 不匹配、heap-only 出现在 root、redirect 出现在非 root，都不能继续当正常链处理。
index scan 通常停止当前 chain。
pruning 在某些不变量被破坏时会 `elog(ERROR)`。
第五类 abort 边界。
heap-only tuple 的 xmin aborted 时，它从未对其他事务可见，可以被移除。
HOT-updated tuple 的 update xmax aborted 时，不需要沿 `t_ctid` 追后继。
`HeapTupleHeaderIsHotUpdated()` 把这些事务状态纳入判断。
所以看 `HEAP_HOT_UPDATED` raw bit 不能直接推断当前仍有有效 HOT successor。
第六类 relcache retry。
`RelationGetIndexAttrBitmap()` 在遍历 index list 时可能收到 relcache flush。
`relcache.c:5484-5510` 重新取 index list，并比较 index list、primary key index、replica identity index 是否变化。
变化就释放临时 bitmaps 并 `goto restart`。
HOT-safety 不能建立在过期的 index set 上。
第七类 recovery 边界。
`heap_page_prune_opt()` 在 recovery 中直接返回。
standby 不写 pruning WAL。
主库之后可能产生清理 WAL。
这意味着 HOT chain 的物理缩短不是每个 reader 都能主动推进。

## 13. 成本、资源与跨模块传播

HOT 减少的成本主要在索引侧。
它减少普通索引 entry 数量。
减少 index page split 和 index WAL。
减少后续 VACUUM 需要从索引删除的 dead TIDs。
减少重复 key update 对 cache locality 的破坏。
但 HOT 不是免费。
第一，index scan 的 heap fetch 可能要追 HOT chain。
链越长，page 内 visibility check 越多。
`README.HOT:249-251` 提到 pruning 缩短 chain 可以降低后续 index searches 成本。
第二，HOT 依赖 old page 有可用空间。
fillfactor 太高、row 变宽、page 上 dead tuple 不能及时 prune，都会让 HOT-safe update 退化成 cold update。
这会把成本从 heap page local chain 转回全部索引维护。
第三，HOT eligibility 本身要比较 interesting attrs。
索引列、表达式、predicate 越多，`HeapDetermineColumnsInfo()` 要取值比较的列越多。
`heapam.c:4405-4409` 也承认多 indexed columns 时逐列 `heap_getattr()` 可能低效。
第四，pruning 需要 cleanup lock。
高并发读写同一 page 时，`ConditionalLockBufferForCleanup()` 可能失败。
失败不会破坏 correctness。
但会推迟空间回收，增加后续 cold update 概率。
第五，summarizing index 把一部分成本带回来。
只更新 BRIN 相关列时仍可 HOT。
但 summary 必须更新，否则 block min/max 等 summary 可能漏掉新值。
因此 `TU_Summarizing` 是 correctness cost，不是优化可选项。
第六，logical decoding 和 replica identity 可能增加 WAL 数据。
`heapam.c:4000-4010` 提前提取 old key tuple。
`heapam.c:8866-8875` 在需要 logical tuple data 时记录新 tuple 和 old key/old tuple。
这和 HOT 是否省普通 index entry 是相邻但独立的成本层。
跨模块传播可以按路径记：

```text
schema/index definition
  -> relcache attr bitmap
  -> heap_update HOT decision
  -> executor index maintenance decision
  -> index AM insertion or skip
  -> heap index fetch chain following
  -> pruning/VACUUM cleanup
```

如果诊断 HOT 比例下降，不要只看 heap。
schema 上新增 expression index、partial index predicate、INCLUDE column、BRIN、fillfactor、long transaction、autovacuum 延迟、checkpoint/full page writes 都可能改变现象。

## 14. 观测与诊断入口

本节锚定一个 runtime truth：
更新非 HOT-blocking 列且新 tuple 留在同页时，`UPDATE` 可以增加 heap tuple version，但不为普通索引新增 entry。
这个 truth 可以从三类现象验证。
第一类是统计视图。
`pg_stat_user_tables` 有：

```text
n_tup_upd
n_tup_hot_upd
n_tup_newpage_upd
```

`n_tup_hot_upd` 是 HOT update 计数。
`n_tup_newpage_upd` 是 update 新版本落到新 page 的计数。
这些是 relation-level 累计统计。
它们不能告诉你哪一行 HOT，也不能告诉你为什么某次 update 失败。
第二类是 `EXPLAIN (ANALYZE, BUFFERS, WAL)`。
它能比较 HOT-friendly update 和 cold update 的 WAL records/bytes、buffer dirtied。
但它不直接标注“这条 update 是 HOT”。
WAL 数字还受 full-page writes、checkpoint 距离、tuple width、logical WAL 需求影响。
第三类是 `pageinspect`。
`heap_page_items(get_raw_page(...))` 能看到：

```text
lp
lp_flags
lp_off
lp_len
t_xmin
t_xmax
t_ctid
t_infomask
t_infomask2
```

`heap_tuple_infomask_flags()` 能把 `HEAP_HOT_UPDATED` 和 `HEAP_ONLY_TUPLE` 解码成文本 flags。
它能直接看到 HOT chain 的 page-level 痕迹。
但它是 raw page 观察。
不要把 raw flag 当作完整 visibility 语义。
第四类是断点。
适合设断点的位置：

```text
heap_update
HeapDetermineColumnsInfo
log_heap_update
heap_hot_search_buffer
heap_prune_chain
ExecInsertIndexTuples
index_insert
```

断点能看到 `use_hot_update`、`summarized_update`、`modified_attrs`、`hot_attrs`、`sum_attrs`。
这些状态 SQL 层很难直接看到。
第五类是索引尺寸和 bloat。
重复 HOT update 不应该让普通 btree index 像 cold update 那样增长。
但 index size 是滞后指标。
它受 page split、dedup、vacuum、fillfactor 和 workload 分布影响。
不要用一次 `pg_relation_size(index)` 直接归因。
第六类是 wait event 和锁。
HOT 本身没有专门 wait event。
如果看到 buffer content lock 或 tuple lock 等待，需要回到 update 并发路径分析。
不能把等待直接解释为 HOT 问题。

## 15. 课堂实验一：HOT 与 cold 的最小对照

目标：
观察同样是 UPDATE，非索引列 same-page 更新会产生 HOT，索引列更新会退化为 cold。
准备：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS hot_demo;
CREATE TABLE hot_demo (
    id int PRIMARY KEY,
    v int,
    payload text
) WITH (fillfactor = 60);

INSERT INTO hot_demo
SELECT g, 0, repeat('x', 80)
FROM generate_series(1, 2000) AS g;

VACUUM hot_demo;
SELECT pg_stat_reset_single_table_counters('hot_demo'::regclass);
```

先更新非索引列：

```sql
UPDATE hot_demo
SET v = v + 1
WHERE id BETWEEN 1 AND 500;

SELECT pg_stat_force_next_flush();

SELECT n_tup_upd, n_tup_hot_upd, n_tup_newpage_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_demo';
```

预期：
`n_tup_upd` 增加。
`n_tup_hot_upd` 应该大量增加。
如果环境、page 空间和 tuple width 不同，可能不是 500 条全 HOT。
这正是 same-page 是 actual HOT 条件的体现。
再创建一个会阻断 HOT 的索引：

```sql
CREATE INDEX hot_demo_v_idx ON hot_demo(v);
SELECT pg_stat_reset_single_table_counters('hot_demo'::regclass);

UPDATE hot_demo
SET v = v + 1
WHERE id BETWEEN 1 AND 500;

SELECT pg_stat_force_next_flush();

SELECT n_tup_upd, n_tup_hot_upd, n_tup_newpage_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_demo';
```

预期：
这次 `v` 是 btree index column。
`modified_attrs` 与 `hot_attrs` 有交集。
`heap_update()` 返回 `TU_All`。
`n_tup_hot_upd` 应显著下降或为 0。
对照源码：

```text
relcache.c:5437-5479   v 被收进 hotblockingattrs
heapam.c:3382-3384    modified_attrs 包含 v
heapam.c:3979         bms_overlap(modified_attrs, hot_attrs) 为 true
heapam.c:4167         update_indexes = TU_All
```

## 16. 课堂实验二：用 pageinspect 看 HOT flags

目标：
看到 `HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE` 和 `t_ctid` 的链路。
重新准备一个小表：

```sql
DROP TABLE IF EXISTS hot_page_demo;
CREATE TABLE hot_page_demo (
    id int PRIMARY KEY,
    v int,
    payload text
) WITH (fillfactor = 50);

INSERT INTO hot_page_demo
SELECT g, 0, repeat('x', 40)
FROM generate_series(1, 20) AS g;

VACUUM hot_page_demo;

UPDATE hot_page_demo SET v = v + 1 WHERE id = 1;
UPDATE hot_page_demo SET v = v + 1 WHERE id = 1;
```

找目标行所在 block：

```sql
SELECT ctid, * FROM hot_page_demo WHERE id = 1;
```

如果 `ctid` 是 `(0,5)`，观察 block 0：

```sql
SELECT h.lp,
       h.lp_flags,
       h.t_xmin,
       h.t_xmax,
       h.t_ctid,
       f.raw_flags,
       f.combined_flags
FROM heap_page_items(get_raw_page('hot_page_demo', 0)) AS h
LEFT JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) AS f
ON h.t_infomask IS NOT NULL
ORDER BY h.lp;
```

你应该寻找这些模式：
旧 tuple 的 `raw_flags` 中有 `HEAP_HOT_UPDATED`。
新 tuple 或中间 tuple 的 `raw_flags` 中有 `HEAP_ONLY_TUPLE`。
旧 tuple 的 `t_ctid` 指向下一跳。
最新可见 version 的 `t_ctid` 通常指向自己。
然后执行：

```sql
VACUUM hot_page_demo;
```

再次观察 page。
可能看到 root line pointer 从 `LP_NORMAL` 变成 redirect。
也可能因为 snapshot horizon、page 状态、版本和 autovacuum 时机不同，没有立刻看到 redirect。
这个实验的重点不是强行制造某个固定 `lp_flags` 数字。
重点是把可见 flags 回到源码：

```text
heapam.c:4029-4036          设置 HOT flags
heapam_indexscan.c:127-139  index scan 能跟随 root redirect
pruneheap.c:1452-1468       root redirect / LP_DEAD 的 pruning 规则
```

## 17. 课堂实验三：same page 边界

目标：
证明“indexed columns 不变”还不够。
如果新 tuple 不能放在 same page，仍然要 cold update。
准备一个更容易撑满 page 的表：

```sql
DROP TABLE IF EXISTS hot_space_demo;
CREATE TABLE hot_space_demo (
    id int PRIMARY KEY,
    v int,
    payload text
) WITH (fillfactor = 100);

INSERT INTO hot_space_demo
SELECT g, 0, repeat('x', 1500)
FROM generate_series(1, 200) AS g;

VACUUM hot_space_demo;
SELECT pg_stat_reset_single_table_counters('hot_space_demo'::regclass);
```

更新非索引列，但让 tuple 变宽：

```sql
UPDATE hot_space_demo
SET payload = payload || repeat('y', 200)
WHERE id BETWEEN 1 AND 50;

SELECT pg_stat_force_next_flush();

SELECT n_tup_upd, n_tup_hot_upd, n_tup_newpage_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_space_demo';
```

预期：
`payload` 没有普通索引。
列变化本身 HOT-safe。
但 page 没空间时，新 version 要落到别的 page。
`n_tup_newpage_upd` 会增加。
`n_tup_hot_upd` 不会等于全部 update 数。
对照源码：

```text
heapam.c:3972  只有 newbuf == buffer 才检查 HOT
heapam.c:3995  newbuf != buffer 时 PageSetFull(page)
heapam.c:4167  非 HOT 返回 TU_All
```

如果你的实验仍然出现很多 HOT，说明 page 空间仍足够，或者 TOAST 行为改变了 tuple size。
把 payload 调大或降低行数重新试。
这是 workload-dependent 现象。

## 18. 课堂实验四：summarizing index 例外

目标：
观察 BRIN 这类 summarizing index 不阻断 HOT chain，但会触发 summarizing index 维护。
准备：

```sql
DROP TABLE IF EXISTS hot_brin_demo;
CREATE TABLE hot_brin_demo (
    id int PRIMARY KEY,
    v int,
    payload text
) WITH (fillfactor = 60);

INSERT INTO hot_brin_demo
SELECT g, 0, repeat('x', 80)
FROM generate_series(1, 2000) AS g;

CREATE INDEX hot_brin_demo_v_brin ON hot_brin_demo USING brin(v);
VACUUM hot_brin_demo;
SELECT pg_stat_reset_single_table_counters('hot_brin_demo'::regclass);
```

更新 BRIN summarizing column：

```sql
UPDATE hot_brin_demo
SET v = v + 1
WHERE id BETWEEN 1 AND 500;

SELECT pg_stat_force_next_flush();

SELECT n_tup_upd, n_tup_hot_upd, n_tup_newpage_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_brin_demo';
```

预期：
在 page 空间足够时，`n_tup_hot_upd` 仍会增加。
这说明 BRIN 的 `v` 没有进入 HOT-blocking attrs。
但源码会返回 `TU_Summarizing`，executor 只更新 summarizing indexes。
这个状态 SQL 统计不能直接显示。
要确认它，适合在 `heap_update()` 返回前或 `ExecInsertIndexTuples()` 设置 flags 处打断点。
对照源码：

```text
relcache.c:5427-5435  amsummarizing -> summarizedattrs
heapam.c:3990-3991   modified_attrs overlaps sum_attrs
heapam.c:4161-4164   update_indexes = TU_Summarizing
execIndexing.c:373-377 跳过非 summarizing indexes
```

然后把 BRIN 改成 btree：

```sql
DROP INDEX hot_brin_demo_v_brin;
CREATE INDEX hot_brin_demo_v_btree ON hot_brin_demo(v);
SELECT pg_stat_reset_single_table_counters('hot_brin_demo'::regclass);

UPDATE hot_brin_demo
SET v = v + 1
WHERE id BETWEEN 1 AND 500;

SELECT pg_stat_force_next_flush();

SELECT n_tup_upd, n_tup_hot_upd, n_tup_newpage_upd
FROM pg_stat_user_tables
WHERE relname = 'hot_brin_demo';
```

这次 `v` 进入 HOT-blocking attrs。
HOT 比例应显著下降。

## 19. gdb / 断点练习

如果你在本地 debug build 上跑实验，可以设置这些断点：

```gdb
break heap_update
break HeapDetermineColumnsInfo
break log_heap_update
break heap_hot_search_buffer
break heap_prune_chain
break ExecInsertIndexTuples
break index_insert
```

重点观察四组状态：

```text
heap_update: use_hot_update, summarized_update, *update_indexes, newbuf == buffer
HeapDetermineColumnsInfo: modified_attrs, hot_attrs, sum_attrs
log_heap_update: HeapTupleIsHeapOnly(newtup), info
heap_hot_search_buffer: offnum, at_chain_start, prev_xmax, current xmin, HOT flag
```

练习目标不是背变量。
而是画出 `index root TID -> root/redirect -> heap-only successor -> visible tuple` 这条链。

## 20. 常见误区

误区一：
“只要 UPDATE 不改索引 key，就是 HOT。”
不够。
必须同 page。
还要看 expression index、partial predicate、INCLUDE columns 和 summarizing index 分类。
误区二：
“HOT update 完全不更新索引。”
不准确。
`TU_None` 不更新索引。
`TU_Summarizing` 仍会更新 summarizing indexes。
普通 TID-bearing indexes 不新增 entry。
误区三：
“`HEAP_HOT_UPDATED` bit 有值就一定要追 `t_ctid`。”
不够。
`HeapTupleHeaderIsHotUpdated()` 还检查 update 是否 aborted、tuple xmin 是否 invalid。
追链还要做 XMIN/XMAX matching。
误区四：
“root tuple dead 以后 root line pointer 可以复用。”
错误。
索引 entry 仍指向 root TID。
root line pointer 要么 redirect 到 live heap-only successor，要么在整链 dead 后变成 `LP_DEAD`，等待 index cleanup。
误区五：
“page pin 可以防止 HOT chain 被破坏。”
错误。
pin 只保护 buffer 不被 evict。
line pointer 状态仍可能因 pruning 改变。
chain validity 要靠 content lock、line pointer state 和 XMIN/XMAX matching。
误区六：
“`indisvalid = false` 的索引不影响 HOT。”
错误。
`CREATE INDEX CONCURRENTLY` 期间，尚未 valid 或 ready 的索引仍可能必须参与 HOT-safety。
`RelationGetIndexAttrBitmap()` 明确考虑这些索引。
误区七：
“`n_tup_hot_upd / n_tup_upd` 就是 schema 是否合理的完整答案。”
不够。
这个比例受 fillfactor、tuple width、page free space、long transaction、pruning 成功率、autovacuum、checkpoint 和 workload 分布影响。

## 21. 讨论题

1. 为什么 HOT 要求 same page？
如果允许跨 page HOT chain，index scan、pruning 和 VACUUM 的哪些边界会改变？
2. 为什么 expression index 和 partial index predicate 中引用的列也要阻断 HOT？
这和不重跑用户定义函数有什么关系？
3. `HEAP_HOT_UPDATED` 和 `HEAP_ONLY_TUPLE` 分别标在哪个 tuple version 上？
为什么不能只用一个 flag？
4. 为什么 heap-only tuple 不能变成 `LP_DEAD`？
如果这么做，VACUUM 从索引删除 TID 时会遇到什么问题？
5. `TU_Summarizing` 为什么不是 `TU_None`？
BRIN minmax summary 漏更新会造成哪类错误？
6. regular CREATE INDEX 为什么需要 `indcheckxmin`？
旧 snapshot 使用新索引会如何看到 broken HOT chain？
7. HOT ratio 下降时，你会按什么顺序排查 schema、page space、long transaction 和 pruning？
8. `HeapDetermineColumnsInfo()` 为什么用二进制 equality，而不是 opclass equality？

## 22. 本节小结

本节只回答一个问题：
什么时候 UPDATE 可以不为新 tuple version 新增普通索引 entry。
答案是：
新 tuple 必须留在 old tuple 的同一 heap page。
所有 HOT-blocking indexes 引用的列必须没有二进制变化。
如果只影响 summarizing indexes，可以保持 HOT chain，但仍要更新 summaries。
HOT 的核心链路是：

```text
heap_update()
  -> RelationGetIndexAttrBitmap()
  -> HeapDetermineColumnsInfo()
  -> newbuf == buffer && !overlap(modified_attrs, hot_attrs)
  -> old HEAP_HOT_UPDATED
  -> new HEAP_ONLY_TUPLE
  -> old.t_ctid = new.t_self
  -> return TU_None / TU_Summarizing / TU_All
```

index scan 的补偿链路是：

```text
index_getnext_tid()
  -> root TID
  -> table_index_fetch_tuple()
  -> heap_hot_search_buffer()
  -> root redirect / HOT flags / t_ctid / XMIN-XMAX matching
  -> visible tuple for snapshot
```

pruning 的补偿链路是：

```text
root tuple dead
  -> root line pointer cannot be reused
  -> LP_REDIRECT to heap-only successor
  -> intermediate heap-only line pointers can become LP_UNUSED
  -> whole chain dead makes root LP_DEAD
  -> regular VACUUM later removes index entries
```

核心状态不是某个字段。
它是：

```text
root line pointer
+ HEAP_HOT_UPDATED / HEAP_ONLY_TUPLE
+ t_ctid
+ same-page invariant
+ XMIN/XMAX matching
+ snapshot visibility
+ WAL / pruning rules
```

ownership 上，heap AM 创建 chain 并返回 index maintenance decision。
executor 根据 `TU_UpdateIndexes` 决定是否调用索引插入。
index AM 不需要知道 heap 为什么省略普通 entry。
但 index scan 必须通过 table AM 回 heap，才能兑现 HOT 可达性。
错误路径上，空间不足退回 cold update。
HOT-blocking attr 变化退回 cold update。
aborted HOT update 通过 transaction status 和 pruning cleanup 收尾。
broken chain 通过停止追链、断言或 ERROR 暴露。
CREATE INDEX 通过 `indcheckxmin` 或 concurrent build 等待，把新索引纳入 HOT-safety。
观测上，`n_tup_hot_upd`、`n_tup_newpage_upd`、`EXPLAIN WAL` 和 `pageinspect` 都有用。
但它们分别是累计统计、单 query 成本、raw page 状态。
没有一个指标能单独解释完整因果。
从本节带走的可迁移规律是：
当系统为了减少二级结构维护而省略一条物理 entry 时，必须在主存储结构里留下另一条可验证、可恢复、可清理的可达性路径。
HOT 的路径是 page-local root TID 加 heap-only chain。
它的所有限制，都是为了让这条替代路径在并发、abort、crash recovery、VACUUM 和 CREATE INDEX 下仍然成立。
