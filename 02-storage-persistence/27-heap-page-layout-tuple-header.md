# PostgreSQL Heap page layout、line pointer 与 tuple header
## 课程定位
本节主题：heap page layout、line pointer 与 tuple header。
上一组课程把 buffer、WAL、relation fork、checkpoint 和 crash restart 的持久化边界讲到 page 级别。
从这一节开始，视角进入 heap access method。
前置知识：已理解 buffer pin/content lock、WAL-before-data、page LSN、relation fork block addressing 和 MVCC snapshot 的基本含义。
本节唯一主问题：
heap page 如何同时支持定位、MVCC 可见性、空间复用和后续 pruning？
本节围绕的核心矛盾：
索引、executor 和外部观察者希望用一个稳定的 `(block, offset)` 找到 tuple。
MVCC 又要求同一逻辑行可以留下多个版本。
UPDATE/DELETE 不能立即搬走或覆盖旧版本，因为旧 snapshot 可能还要读它。
但 page 只有固定大小，tuple bytes 必须能被压缩、复用、剪枝。
PostgreSQL 用 slotted page、line pointer、tuple header、infomask 和 `t_ctid` 把这些要求拆到不同层。
读完本节，你应该能判断：
- `PageHeaderData` 哪些字段是 page 空间管理状态，哪些只是 hint。
- `ItemIdData` 为什么是稳定 TID 和可移动 tuple bytes 之间的间接层。
- `LP_NORMAL`、`LP_REDIRECT`、`LP_DEAD`、`LP_UNUSED` 分别允许什么，不允许什么。
- `HeapTupleData` 和 `HeapTupleHeaderData` 为什么不能混为一谈。
- `xmin`、`xmax`、`infomask`、`infomask2` 如何组合成可见性语义。
- 为什么不能把 `xmax` 有值直接理解成“tuple 已删除”。
- `t_ctid` 什么时候指向自己，什么时候指向新版本，什么时候可能是特殊 token。
- `t_hoff` 的边界为什么决定 user data 从哪里开始，而不是决定 tuple 是否可见。
- heap scan、index fetch、insert、update、delete、pruning 分别读写哪一层状态。
- 哪些 page 状态可以通过 `pageinspect` 看见，哪些只能通过源码断点推断。
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
src/include/storage/bufpage.h
src/backend/storage/page/bufpage.c
src/include/access/htup_details.h
src/include/access/htup.h
src/backend/access/heap/heapam.c
src/backend/access/heap/heapam_visibility.c
src/backend/access/heap/pruneheap.c
```
为讲清 line pointer 和插入落点，本节辅助核对：
```text
src/include/storage/itemid.h
src/backend/access/heap/hio.c
contrib/pageinspect/pageinspect--1.5.sql
contrib/pageinspect/heapfuncs.c
```
行号来自：
```text
nl -ba <source-file>
```
## 1. 先给结论
heap page 是一个 slotted page。
`bufpage.h:24-78` 的注释给出基本布局：
```text
PageHeaderData
line pointer array
free space
tuple bytes growing backward
special space
```
heap page 通常没有 access-method special space。
但 page 通用格式仍然保留 `pd_special`。
这让 heap、btree、hash、GIN 等 AM 共享同一套 page header 和 line pointer 机制。
`PageHeaderData` 记录 page 级物理状态。
最关键的是：
```text
pd_lower   line pointer array 的末端
pd_upper   tuple bytes 区域的起点
pd_special special space 的起点
pd_prune_xid 可能值得 pruning 的最老 XID hint
pd_flags   free line、page full、all visible 等 page hint/flag
pd_lsn     最后修改该 page 的 WAL 位置
```
`pd_lower` 向高地址增长。
`pd_upper` 向低地址增长。
两者之间才是可连续分配的 free space。
page 是否还能插入 tuple，不是看 page 里有没有洞。
而是看 `pd_lower` 到 `pd_upper` 之间还能不能容纳一个 line pointer 和 tuple bytes。
`ItemIdData` 是 line pointer。
它不等同于 tuple。
它是 `(offset number -> tuple bytes)` 的映射。
索引 TID 和 `HeapTupleData.t_self` 使用的是 block number 加 offset number。
它们指向 line pointer 编号，而不是 tuple 在 page 里的字节偏移。
这就是 page 内可以 compact tuple bytes 而不改 index TID 的原因。
`ItemIdData` 有三个物理字段：
```text
lp_off    15 bits
lp_flags  2 bits
lp_len    15 bits
```
`lp_flags` 的四个状态定义在 `itemid.h:35-41`：
```text
LP_UNUSED
LP_NORMAL
LP_REDIRECT
LP_DEAD
```
这四个状态是本节最重要的不变量之一。
`LP_NORMAL` 有 tuple storage。
`LP_REDIRECT` 没有 tuple storage，但 `lp_off` 存的是另一个 offset number。
`LP_DEAD` 表示 index 可能还指向这个 root TID，所以不能随便复用。
`LP_UNUSED` 才可以被新 tuple 立即复用。
`HeapTupleHeaderData` 是 tuple bytes 的开头。
它记录 MVCC 和 tuple data 边界。
最关键的是：
```text
t_choice.t_heap.t_xmin
t_choice.t_heap.t_xmax
t_choice.t_heap.t_field3
t_ctid
t_infomask2
t_infomask
t_hoff
t_bits[]
```
`xmin` 是插入事务。
`xmax` 是删除、更新或锁定相关状态。
但 `xmax` 的语义必须和 `t_infomask`、`t_infomask2`、MultiXact、lock-only bit、snapshot 一起解释。
单看 `xmax` 不构成语义。
`t_ctid` 是版本链线索。
新插入 tuple 的 `t_ctid` 指向自己。
UPDATE 后，旧版本的 `t_ctid` 指向替代版本。
最新版本通常仍指向自己。
分区移动和 speculative insertion 还会使用特殊表示。
所以 `t_ctid` 不是“这个 tuple 自己在哪里”的唯一表达。
`HeapTupleData.t_self` 才是当前内存对象对应的物理 TID。
`t_hoff` 是 user data 起点。
它包含固定 header、null bitmap、padding 和历史 OID 字段占用后的 offset。
它必须满足 alignment。
tuple deforming 从 `t_data + t_hoff` 开始读用户列。
`t_hoff` 不参与事务可见性判断。
但它决定 pageinspect 和 executor 能否正确解释后续 bytes。
完整运行模型可以压缩成一句话：
```text
TID 定位到 line pointer，line pointer 定位 tuple bytes，tuple header 决定版本语义，pruning 改 line pointer 状态和压缩 bytes 但尽量保留仍可能被 index 引用的 TID 入口。
```
## 2. 本节主问题的四个子约束
为了回答唯一主问题，要把 heap page 同时承担的四件事分开看。
第一件事是定位。
executor、index AM、system column `ctid` 和 HOT chain 都需要稳定 TID。
TID 的 offset number 是 line pointer 编号。
只要 line pointer 还存在，tuple bytes 就能在 page 内移动。
第二件事是 MVCC 可见性。
一个 tuple version 是否对某个 snapshot 可见，由 tuple header、事务状态、snapshot 和 hint bits 一起决定。
这不是 page header 能独立回答的问题。
第三件事是空间复用。
DELETE 和 UPDATE 通常只改旧版本 header。
物理空间要等旧版本对所有相关 snapshot 都不可见后才能回收。
回收又分成 tuple bytes 回收、line pointer 复用、line pointer array 尾部收缩。
第四件事是后续 pruning。
HOT update 要让 index entry 仍然指向旧 root TID，同时 heap 内部能沿 `t_ctid` 找到新版本。
pruning 要在不破坏 index TID 的情况下剪掉不可见旧版本。
这正是 `LP_REDIRECT` 和 `LP_DEAD` 存在的原因。
这四件事不能靠一个字段同时完成。
PostgreSQL 的设计是拆层：
```text
PageHeaderData     page 级空间边界和 pruning hint
ItemIdData         stable offset number 到 tuple storage 的间接层
HeapTupleHeader    tuple version 的事务语义和 data 起点
Snapshot/CLOG/MXID tuple header 之外的可见性事实
PruneState/WAL     延迟执行空间回收并能恢复
```
读源码时要不断问：
当前代码是在处理定位、可见性、空间、还是 pruning？
如果把这几层混起来，最容易得出错误结论。
## 3. 源码阅读顺序
不要从 `heapam.c` 顶部顺序读。
本节推荐按状态边界读。
| 顺序 | 文件 | 读什么 | 为什么先读 |
| --- | --- | --- | --- |
| 1 | `src/include/storage/bufpage.h` | page layout、`PageHeaderData`、`PageGetItem()`、`PageSetPrunable()` | 先建立 page 级物理边界 |
| 2 | `src/include/storage/itemid.h` | `ItemIdData` 和 `LP_*` 状态 | line pointer 是定位和 pruning 的交点 |
| 3 | `src/backend/storage/page/bufpage.c` | `PageInit()`、`PageAddItemExtended()`、`PageRepairFragmentation()` | 看 page 状态如何真实变化 |
| 4 | `src/include/access/htup.h` | `HeapTupleData` | 区分内存指针壳和磁盘 tuple header |
| 5 | `src/include/access/htup_details.h` | `HeapTupleHeaderData`、infomask、`t_ctid`、`t_hoff` | 建立 tuple version 状态模型 |
| 6 | `src/backend/access/heap/hio.c` | `RelationPutHeapTuple()` | 看 tuple 如何落到 line pointer |
| 7 | `src/backend/access/heap/heapam.c` | scan/fetch/insert/update/delete | 看主流程如何读写这些状态 |
| 8 | `src/backend/access/heap/heapam_visibility.c` | MVCC 和 vacuum visibility | 看 header 如何被解释成语义 |
| 9 | `src/backend/access/heap/pruneheap.c` | prune planning 和 execute | 看空间如何安全回收 |
本节刻意不展开：
- FSM 如何选择目标 page。
- VM all-visible/all-frozen 的完整协议。
- WAL redo 如何恢复 heap 操作。
- HOT update 的全部索引维护边界。
- VACUUM lazy scan 的完整两阶段行为。
这些内容分别属于后续 `28`、`30`、`32`、`33` 节。
本节只讲它们和 page layout 的接口边界。
## 4. `PageHeaderData` 的边界
`PageHeaderData` 定义在 `bufpage.h:184-197`。
它是所有 page-organized block 的通用头。
heap page、index page 和其它 AM page 都从这里开始。
本节只关心这些字段：
```text
pd_lsn
pd_flags
pd_lower
pd_upper
pd_special
pd_prune_xid
pd_linp[]
```
`pd_lsn` 是 WAL-before-data 的边界。
heap insert、update、delete 和 prune 在写 WAL 后调用 `PageSetLSN()`。
buffer manager 写脏页前必须确保 WAL 至少 flush 到 page LSN。
本节不展开 WAL 细节，但要记住：page layout 修改不是单纯内存动作。
`pd_lower` 是 line pointer array 的末尾。
`PageGetMaxOffsetNumber()` 用它计算当前最大 offset number。
源码在 `bufpage.h:396-405`：
```text
maxoff = (pd_lower - SizeOfPageHeaderData) / sizeof(ItemIdData)
```
这说明 offset number 的数量来自 line pointer array 长度。
不是来自 live tuple 数量。
`LP_DEAD`、`LP_REDIRECT`、`LP_UNUSED` 都可能仍占据 offset number。
`pd_upper` 是 tuple storage 区域的低地址边界。
`PageAddItemExtended()` 插入 tuple 时会把 `pd_upper` 向低地址移动。
tuple bytes 在 page 中是从尾部向前堆放的。
`pd_special` 是 special space 起点。
heap page 通常没有 AM opaque trailer。
但通用 page code 仍然用 `pd_special` 判断 tuple bytes 不能越界。
`pd_prune_xid` 是 pruning hint。
`bufpage.h:167-168` 明确说它帮助判断 pruning 是否有用。
它不是可见性事实。
它可能为零。
它可能滞后。
它的存在是为了避免每次访问 page 都计算全页 tuple visibility。
`pd_flags` 中本节相关的是：
- `PD_HAS_FREE_LINES`
- `PD_PAGE_FULL`
- `PD_ALL_VISIBLE`
`PD_HAS_FREE_LINES` 是 line pointer free hint。
`bufpage.h:205-207` 说明它应该被当作 hint，因为变更不 WAL logged。
如果 hint 说有 free line，`PageAddItemExtended()` 会扫描确认。
如果扫描发现没有，就清掉 hint。
`PD_PAGE_FULL` 是 update 插入新版本失败时留下的 hint。
它提示后续访问可以尝试 pruning。
这不是严格“page 满”的事实。
因为 page 可能随后被其它动作改变。
`PD_ALL_VISIBLE` 是 heap page 和 visibility map 的交点。
本节只要记住：DML 修改 page 时如果 page all-visible，必须清掉 page flag 和 VM bit。
insert 在 `heapam.c:2062-2068` 做这个动作。
update 在 `heapam.c:4062-4076` 做这个动作。
delete 在 `heapam.c:2997-3002` 做这个动作。
`PageInit()` 在 `bufpage.c:42-60` 建立初始 page：
```text
pd_lower = SizeOfPageHeaderData
pd_upper = pageSize - specialSize
pd_special = pageSize - specialSize
pd_prune_xid = 0
```
这时 line pointer array 为空。
tuple storage 为空。
free space 是 `pd_lower` 到 `pd_upper` 之间的整段连续空间。
## 5. `ItemIdData` 和 line pointer 状态
`ItemIdData` 定义在 `itemid.h:25-30`。
它只有 32 bits。
其中 `lp_off` 和 `lp_len` 都是 15 bits。
这也是 page size 上限和 tuple item length 表达能力的一个历史边界。
`bufpage.h:180-181` 提到 page 最大只能支持到 32KB，原因就是 `lp_off/lp_len` 只有 15 bits。
line pointer 的四个状态是：
| 状态 | 是否有 storage | 是否可直接作为 tuple | 是否可立即复用 | 主要用途 |
| --- | --- | --- | --- | --- |
| `LP_UNUSED` | no | no | yes | 空 slot |
| `LP_NORMAL` | yes | yes | no | 普通 heap tuple |
| `LP_REDIRECT` | no | no | no | HOT root redirect |
| `LP_DEAD` | usually no | no | no | index 可能仍引用的 dead root |
`ItemIdHasStorage()` 只看 `lp_len != 0`。
所以“是否 in use”和“是否有 storage”不是同一个问题。
`LP_REDIRECT` 是 in use，但没有 storage。
`LP_DEAD` 是 in use，但通常没有 storage。
`LP_UNUSED` 既不是 in use，也没有 storage。
`LP_REDIRECT` 的特殊点在 `itemid.h:75-79`：
```text
lp_off stores redirect target offset number
```
也就是说，`LP_REDIRECT.lp_off` 不再是字节偏移。
它是另一个 line pointer 编号。
读 line pointer 时必须先看 `lp_flags`。
不能把 `lp_off` 无条件解释成 page byte offset。
`PageGetItem()` 在 `bufpage.h:378-385` 只接受有 storage 的 item。
它断言 `ItemIdHasStorage(itemId)`。
这就是为什么普通 heap scan 在 `heapam.c:1011-1014` 先判断 `ItemIdIsNormal(lpp)`，再调用 `PageGetItem()`。
`PageAddItemExtended()` 是 line pointer 分配的核心。
它在 `bufpage.c:202-364` 完成四件事：
- 校验 `pd_lower/pd_upper/pd_special` 是否 sane。
- 选择 offset number。
- 检查新 line pointer 加 tuple bytes 是否放得下。
- 写 line pointer、复制 tuple bytes、推进 `pd_lower/pd_upper`。
如果调用者不指定 offset number，它会优先复用 free line pointer。
但只有当 `PD_HAS_FREE_LINES` hint 置位时才扫描。
见 `bufpage.c:260-290`。
这说明 PostgreSQL 不想在每次插入时无条件扫描整个 line pointer array。
如果没有可复用 line pointer，就使用 `limit = maxoff + 1`。
这会让 `pd_lower` 增加一个 `sizeof(ItemIdData)`。
如果复用已有 `LP_UNUSED`，`pd_lower` 不变。
heap page 还要遵守 `MaxHeapTuplesPerPage`。
`PageAddItemExtended()` 在 `bufpage.c:306-310` 拒绝超过上限。
`PageGetHeapFreeSpace()` 在 `bufpage.c:986-1000` 也把这个上限纳入 free space 计算。
原因是即使 tuple bytes 还有空间，过多 redirect/dead line pointer 也不能让 heap page 的 offset number 无限增长。
`PageRepairFragmentation()` 是 page 内 tuple bytes 压缩。
它在 `bufpage.c:707-822`：
- 扫描 line pointer array。
- 收集所有仍有 storage 的 item。
- 把 tuple bytes compact 到 page 尾部。
- 调整各 line pointer 的 `lp_off`。
- 截掉尾部连续 `LP_UNUSED` line pointer。
- 更新 `PD_HAS_FREE_LINES` hint。
这一步不会改变还存在的 offset number 语义。
如果 offset 5 仍然是 `LP_NORMAL`，外部 TID `(block, 5)` 仍能通过 line pointer 找到它。
tuple bytes 的物理地址可以变。
TID 不能变。
## 6. `HeapTupleData` 不是磁盘 header
`HeapTupleData` 定义在 `htup.h:62-69`。
它是内存中的指针壳：
```text
t_len
t_self
t_tableOid
t_data
```
`htup.h:31-60` 的注释列出几种使用方式。
最重要的是：当它指向 buffer 中的 tuple 时，`t_data` 直接指向 shared buffer 的 page bytes。
代码必须持有相应 buffer pin。
但 pin 不记录在 `HeapTupleData` 自身。
这会带来一个常见误区：
看见一个 `HeapTupleData *`，不能推断它拥有 tuple bytes。
它可能只是指向 page 的临时视图。
释放 buffer 后继续读 `t_data` 就可能变成 use-after-release。
heap scan 在 `heapam.c:1014-1017` 填充这个壳：
```text
t_data = PageGetItem(page, lpp)
t_len = ItemIdGetLength(lpp)
t_self = (block, lineoff)
```
`heap_fetch()` 在 `heapam.c:1740-1747` 也做类似事情。
成功返回时，调用者必须释放 `userbuf`。
`heap_fetch()` 的注释在 `heapam.c:1657-1659` 明确说明这个 ownership。
所以本节有两个不同的 TID 概念：
- `HeapTupleData.t_self`：这个内存 tuple view 当前来自哪个 `(block, offset)`。
- `HeapTupleHeaderData.t_ctid`：存储在 tuple header 里的版本链或特殊 token。
新手经常把这两个字段混起来。
它们都叫 TID 相关字段，但语义不同。
## 7. `HeapTupleHeaderData` 的物理布局
`HeapTupleHeaderData` 定义在 `htup_details.h:153-181`。
它是 tuple bytes 的固定开头。
`htup_details.h:65-71` 给出 tuple 总体结构：
```text
fixed fields
nulls bitmap
alignment padding
old OID field if any
user data fields
```
固定 header 中本节最重要的字段是：
```text
t_choice.t_heap.t_xmin
t_choice.t_heap.t_xmax
t_choice.t_heap.t_field3
t_ctid
t_infomask2
t_infomask
t_hoff
t_bits[]
```
`t_choice` 是 union。
同一段物理空间在 heap tuple 和 composite datum 场景下有不同解释。
插入 heap relation 前，`heap_prepare_insert()` 会写入事务字段。
见 `heapam.c:2216-2225`。
`t_field3` 复用 `cmin`、`cmax` 和历史 `xvac`。
`htup_details.h:73-84` 解释了这个复用。
`cmin/cmax` 只对 originating transaction 的 command visibility 有意义。
同一事务 insert 又 delete 时需要 combo CID。
所以不要把 `t_field3` 当成全局稳定语义。
`t_infomask` 存储事务状态、lock mode、tuple physical 属性等 flag。
`htup_details.h:190-219` 定义它。
本节要记住这些：
```text
HEAP_HASNULL
HEAP_HASVARWIDTH
HEAP_HASEXTERNAL
HEAP_XMAX_KEYSHR_LOCK
HEAP_COMBOCID
HEAP_XMAX_EXCL_LOCK
HEAP_XMAX_LOCK_ONLY
HEAP_XMIN_COMMITTED
HEAP_XMIN_INVALID
HEAP_XMAX_COMMITTED
HEAP_XMAX_INVALID
HEAP_XMAX_IS_MULTI
HEAP_UPDATED
HEAP_MOVED_OFF
HEAP_MOVED_IN
```
`t_infomask2` 存储列数和 HOT 相关 flag。
`htup_details.h:291-296` 定义：
```text
HEAP_NATTS_MASK
HEAP_KEYS_UPDATED
HEAP_HOT_UPDATED
HEAP_ONLY_TUPLE
```
`HEAP_HOT_UPDATED` 在旧版本上。
它表示这条 tuple 被 HOT 更新过，`t_ctid` 可以指向同 page 的 heap-only successor。
`HEAP_ONLY_TUPLE` 在新版本上。
它表示这条 tuple 没有自己的 index entry，只能通过 heap 内链条到达。
`HeapTupleHeaderIsHotUpdated()` 在 `htup_details.h:524-531` 还要求：
- `HEAP_HOT_UPDATED` 置位。
- `HEAP_XMAX_INVALID` 不置位。
- `xmin` 不 invalid。
这说明 HOT 状态不是只看一个 bit。
如果 updater aborted，hint bit 可能让这条链不再被当作有效 HOT chain。
`HeapTupleHeaderIsHeapOnly()` 在 `htup_details.h:545-549` 只看 `HEAP_ONLY_TUPLE`。
但它的安全性依赖 update 和 pruning 的不变量。
heap-only tuple 不能被 index 直接引用。
`t_hoff` 在 `htup_details.h:117-119` 被要求是 `MAXALIGN` 的倍数。
它是 fixed header、null bitmap、padding 后 user data 的起点。
`HeapTupleHeaderGetDatumLength()` 和 tuple deforming 最终都会依赖这个边界。
`t_bits[]` 只有当 `HEAP_HASNULL` 置位时才有意义。
没有 null 时，不应想当然读取 null bitmap。
pageinspect 中 `t_bits` 可能为空。
## 8. `xmin`、`xmax` 和 infomask 的组合语义
`xmin` 是插入事务 XID。
但 `HeapTupleHeaderGetXmin()` 在 `htup_details.h:328-333` 会把 frozen tuple 返回成 `FrozenTransactionId`。
`HeapTupleHeaderGetRawXmin()` 才是原始 header 值。
所以诊断时要区分 raw field 和 accessor 语义。
`HeapTupleHeaderSetXminFrozen()` 在 `htup_details.h:360-365` 通过 infomask 表示 frozen。
现代 PostgreSQL 不一定把原始 xmin 改写成 `FrozenTransactionId`。
这正是“raw field 不是语义”的例子。
`xmax` 更容易误读。
它可能表示：
- 删除事务。
- 更新事务。
- tuple locker。
- MultiXactId。
- abort 后留下的无效值。
- lock-only 状态。
`HEAP_XMAX_LOCK_ONLY` 说明 `xmax` 只是 locker，不是 deleter/updater。
`HEAP_XMAX_IS_MULTI` 说明 raw xmax 是 MultiXactId，不是普通 XID。
`HeapTupleHeaderGetUpdateXid()` 在 `htup_details.h:381-396` 会在必要时解析 MultiXact。
源码注释提醒这可能涉及 multixact I/O，所以不能随便在 hot path 调用。
判断 tuple 对 MVCC snapshot 是否可见，主入口是 `HeapTupleSatisfiesMVCC()`。
它在 `heapam_visibility.c:939-1095`。
这个函数按时间语义推进：
- 先判断 `xmin` 是否 committed、invalid、current transaction、in snapshot。
- 再判断 `xmax` 是否 invalid、lock-only、MultiXact、current transaction、in snapshot、committed。
- 在合适时机设置 hint bits。
- 最后返回当前 snapshot 是否可见。
注意 `heapam_visibility.c:922-936` 的注释。
如果插入或删除事务在当前 snapshot 中仍运行，函数故意不查询真实事务状态来急着设置 hint bit。
因为那会访问高竞争共享结构，且不会改变当前 snapshot 的判断结果。
hint bit 可以留给更晚的访问者设置。
`SetHintBitsExt()` 在 `heapam_visibility.c:102-203` 还有 WAL 安全边界。
设置 committed hint bit 前，必须确认 commit record 已经 flush，或者 page LSN 已经提供 interlock。
否则可能把“看似 committed”的 page 先写盘，crash 后找不到对应 commit WAL。
所以可见性不是这个公式：
```text
xmin committed && xmax invalid
```
真实判断更接近：
```text
tuple header bits
+ raw xmin/xmax
+ possible MultiXact
+ current transaction command id
+ MVCC snapshot
+ transaction status / CLOG
+ hint bit safety
= visibility result
```
`HeapTupleSatisfiesVacuum()` 是另一个语义。
它在 `heapam_visibility.c:1113-1132`。
它要回答的不是“当前 snapshot 能不能看见”。
它要回答“是否可能被任何运行事务看见，能不能移除”。
`HeapTupleSatisfiesVacuumHorizon()` 在 `heapam_visibility.c:1147-1328` 返回：
```text
HEAPTUPLE_DEAD
HEAPTUPLE_LIVE
HEAPTUPLE_RECENTLY_DEAD
HEAPTUPLE_INSERT_IN_PROGRESS
HEAPTUPLE_DELETE_IN_PROGRESS
```
`HEAPTUPLE_RECENTLY_DEAD` 是 pruning/vacuum 的关键状态。
它说明删除或更新已经提交，但可能仍被旧 snapshot 看见。
是否能变成 `HEAPTUPLE_DEAD` 还要和 horizon 比较。
这就是本节主问题里的 MVCC 和空间复用冲突：
executor 已经看不见的 tuple，不一定能被 pruning 立刻移除。
必须确认全局可见性边界允许。
## 9. `t_ctid` 的边界
`htup_details.h:86-112` 是读 `t_ctid` 必看的注释。
它给出几个不变量。
新 tuple 存到磁盘时，`t_ctid` 初始化为自己的 TID。
`RelationPutHeapTuple()` 在 `hio.c:65-79` 完成这件事。
它先调用 `PageAddItem()` 得到 `offnum`。
再设置 `tuple->t_self = (block, offnum)`。
最后把 page 中实际 tuple header 的 `t_ctid` 也设成这个 `t_self`。
如果 tuple 被 UPDATE，旧 tuple 的 `t_ctid` 会改指新版本。
`heap_update()` 在 `heapam.c:4059-4060` 做这件事：
```text
old.t_ctid = new.t_self
```
如果 tuple 是最新版本，通常 `t_ctid` 指向自己。
如果 `xmax` valid 但 `t_ctid` 仍指向自己，这个 tuple 可能被锁定或被删除。
不能只凭 `t_ctid == self` 判断可见。
沿 `t_ctid` 追版本链时必须验证目标 tuple。
`htup_details.h:96-103` 说明 VACUUM 可能先清理被指向的新版本。
目标 slot 可能为空，也可能被复用给无关 tuple。
因此要检查目标 tuple 的 `xmin` 是否等于引用 tuple 的 `xmax`。
`heap_get_latest_tid()` 在 `heapam.c:1782-1895` 是这个规则的运行例子。
它循环追 `t_ctid`。
每次跟到下一条后，用 `priorXmax` 和目标 tuple 的 `xmin` 做匹配。
如果 offset 无效、line pointer 不 normal、`xmin` 不匹配，就停止。
这说明 `t_ctid` 是候选链路，不是不可怀疑的指针。
它必须和 line pointer 状态、`xmin/xmax`、snapshot 一起验证。
## 10. 主流程一：insert 如何建立定位和可见性状态
主入口是 `heap_insert()`，见 `heapam.c:2004-2193`。
第一步是生成事务状态。
`heap_insert()` 取 `GetCurrentTransactionId()`。
然后调用 `heap_prepare_insert()`。
`heap_prepare_insert()` 在 `heapam.c:2201-2241`：
- 清掉旧事务相关 bits。
- 设置 `HEAP_XMAX_INVALID`。
- 设置 `xmin = current xid`。
- 必要时设置 frozen。
- 设置 `cmin`。
- 设置 `xmax = 0`。
- 设置 `t_tableOid`。
- 必要时 TOAST。
这一步还没有把 tuple 放进 page。
它只是准备要写入 page 的 tuple bytes。
第二步是选 page。
`heap_insert()` 调用 `RelationGetBufferForTuple()`。
这个函数会和 FSM、relation extension、VM pin 等模块交互。
本节只关心结果：返回一个已经 pin 且适合插入的 buffer。
第三步是进入 critical section。
`heapam.c:2056-2057` 明确写着：
```text
NO EREPORT(ERROR) from here till changes are logged
START_CRIT_SECTION()
```
原因是之后会修改 shared buffer page。
如果修改 page 后、写 WAL 前抛 ERROR，内存页和持久化恢复协议会失配。
第四步是 `RelationPutHeapTuple()`。
它在 `hio.c:27-80`：
- 要求调用者持有 buffer exclusive lock。
- 调 `PageAddItem()`。
- 失败则 `PANIC`，不能普通 ERROR。
- 设置 `tuple->t_self`。
- 非 speculative insert 时，把 tuple header 里的 `t_ctid` 设置为 `t_self`。
第五步是更新 page hint。
如果 page 原来 all-visible，要清 `PD_ALL_VISIBLE` 和 visibility map。
然后设置 `pd_prune_xid`。
`heapam.c:2071-2085` 的注释很重要：
即使只是 insert，也设置 `pd_prune_xid`。
原因是插入事务可能 abort，tuple 会变成可剪枝的 dead tuple。
同时后续也可能借 prune 机会把 page 设回 all-visible。
第六步是 mark dirty 和写 WAL。
`heap_insert()` 在 `heapam.c:2087-2165`：
- `MarkBufferDirty(buffer)`。
- `XLogBeginInsert()`。
- 注册 heap insert record。
- 注册 buffer 和 tuple header/data。
- `XLogInsert(RM_HEAP_ID, info)`。
- `PageSetLSN(page, recptr)`。
第七步是退出 critical section，释放 buffer。
插入完成后，外部看到的定位关系是：
```text
index TID or heap scan TID -> (block, offnum)
(block, offnum) -> ItemIdData
ItemIdData -> tuple bytes
tuple header xmin=current xid, xmax invalid, t_ctid=self
```
但 MVCC 可见性仍取决于事务是否提交和读取者 snapshot。
插入落到 page 不等于对所有人可见。
## 11. 主流程二：scan/fetch 如何从 TID 到可见 tuple
顺序 heap scan 的核心在 `heapgettup()`。
关键片段是 `heapam.c:963-1030`。
扫描一个 page 时，它做的是：
- 取得 page。
- 遍历 offset number。
- `PageGetItemId(page, lineoff)`。
- 如果 `!ItemIdIsNormal(lpp)`，跳过。
- `PageGetItem()` 得到 tuple header pointer。
- 填充 `HeapTupleData.t_len` 和 `t_self`。
- 调 `HeapTupleSatisfiesVisibility()`。
这里有两个边界。
第一，普通 scan 不把 `LP_REDIRECT` 当 tuple。
`LP_REDIRECT` 是 HOT pruning 后保留给 index TID 的入口。
顺序 scan 会扫描实际 `LP_NORMAL` tuple。
第二，定位成功不代表可见。
`PageGetItem()` 只是拿到了 tuple bytes。
`HeapTupleSatisfiesVisibility()` 才把 tuple header 和 snapshot 解释成结果。
`heap_fetch()` 是按 TID 取 tuple。
它在 `heapam.c:1651-1779`。
调用者传入 `tuple->t_self`。
函数读对应 block，检查 offset 是否在 `PageGetMaxOffsetNumber()` 范围内。
然后检查 line pointer 是否 `LP_NORMAL`。
如果不是，就返回 false。
`heap_fetch()` 注释在 `heapam.c:1671-1672` 明说：
它不跟 HOT chain。
它只 fetch 请求的精确 TID。
需要跟 HOT chain 的路径由 index scan 的 heap HOT search 处理。
index heap fetch 在 `heapam_indexscan.c:260-274` 会先尝试 `heap_page_prune_opt()`，再 `heap_hot_search_buffer()`。
这就是 index TID、HOT chain 和 pruning 的交界。
本节不展开 `heap_hot_search_buffer()` 的完整实现，只保留边界：
index entry 指向 root line pointer。
heap 内部负责从 root 找到对 snapshot 可见的版本。
## 12. 主流程三：delete 为什么只改 header
`heap_delete()` 在 `heapam.c:2717-3142`。
它不是把 tuple bytes 从 page 上删掉。
它先定位 tuple：
- `ReadBuffer()`。
- exclusive lock buffer。
- `PageGetItemId()`。
- `PageGetItem()`。
- 填充 `HeapTupleData`。
然后调用 `HeapTupleSatisfiesUpdate()`。
这是 DML 冲突语义，不是普通 MVCC scan 语义。
如果 tuple 正被其它事务修改，可能要等待 `xmax` 指向的事务或 MultiXact。
等待前必须复制 `xwait` 和 `infomask`。
等待后重新加锁，并检查 `xmax` 是否变化。
见 `heapam.c:2801-2899`。
真正 delete 的 critical section 在 `heapam.c:2986-3098`。
它做的核心动作是：
- `PageSetPrunable(page, xid)`。
- 清 all-visible/VM。
- 清旧 `xmax` bits。
- 写新的 `xmax` 和 infomask。
- `HeapTupleHeaderClearHotUpdated()`。
- 设置 `cmax`。
- 把 `t_ctid` 设回 `t_self`。
- mark dirty。
- 写 `XLOG_HEAP_DELETE`。
- `PageSetLSN()`。
delete 后，line pointer 仍然是 `LP_NORMAL`。
tuple bytes 仍在 page 上。
只是 header 表达“这个版本被 xid 删除”。
为什么不立即 `LP_UNUSED`？
因为旧 snapshot 可能仍能看见这个版本。
index entry 也可能仍指向这个 TID。
所以 delete 只推进版本语义。
空间回收留给 pruning/vacuum。
## 13. 主流程四：update 如何形成版本链
`heap_update()` 在 `heapam.c:3201-4150`。
它比 delete 更能体现本节主问题。
update 先定位旧 tuple 并做 DML visibility/conflict 检查。
如果遇到 concurrent locker/updater，会释放 buffer lock、等待事务或 MultiXact、重新加锁、重新检查。
这说明 tuple header 是并发协议的一部分。
然后准备新 tuple 的 header。
`heapam.c:3732-3742`：
- 清新 tuple 的事务 bits。
- 设置新 tuple `xmin = current xid`。
- 设置新 tuple `cmin`。
- 设置 `HEAP_UPDATED`。
- 设置新 tuple 的 `xmax` 和 infomask。
接着判断新版本是否能放在同一 page。
如果需要 TOAST 或当前 page free space 不够，可能释放旧 page content lock。
释放前必须临时把旧 tuple 标成 locked。
`heapam.c:3785-3794` 解释了原因：
要防止其它会话同时更新它。
而且这个临时 header 修改也要 WAL log。
如果新版本能放同一 page，并且 HOT blocking index 列没有变化，就可以 HOT。
`heapam.c:3972-3992` 决定 `use_hot_update`。
实际 update 的 critical section 在 `heapam.c:4012-4110`。
核心动作是：
- 对 old page 设置 `pd_prune_xid`。
- 如果新版本在不同 page，也对 new page 设置 `pd_prune_xid`。
- HOT 时旧 tuple 设置 `HEAP_HOT_UPDATED`。
- HOT 时新 tuple 设置 `HEAP_ONLY_TUPLE`。
- 调 `RelationPutHeapTuple()` 插入新 tuple。
- 旧 tuple 写入新的 `xmax`、infomask、`cmax`。
- 旧 tuple `t_ctid = new tuple t_self`。
- 清 all-visible/VM。
- mark dirty。
- 写 update WAL。
- 给 old/new page 设置 LSN。
update 后，同一逻辑行至少有两个 tuple version。
旧版本仍占一个 line pointer 和 tuple bytes。
新版本也占一个 line pointer 和 tuple bytes。
旧版本 header 的 `xmax` 表示更新事务。
旧版本 header 的 `t_ctid` 指向新版本。
如果是 HOT update：
index entry 仍指向旧 root TID。
新版本没有自己的 index entry。
heap page 内部用 `t_ctid` 和 `HEAP_HOT_UPDATED/HEAP_ONLY_TUPLE` 找到新版本。
后续 pruning 可以把 root tuple storage 移除，但要留下 `LP_REDIRECT`。
如果不是 HOT update：
索引会为新版本生成新的 index entry。
旧版本最终可以变成 `LP_DEAD` 或 `LP_UNUSED`，但要遵守 index cleanup 和 visibility horizon。
## 14. 主流程五：pruning 如何回收空间但保留必要入口
on-access pruning 的入口是 `heap_page_prune_opt()`。
它在 `pruneheap.c:271-360`。
调用者必须持有 buffer pin，但不能已经持有 buffer lock。
函数先做便宜判断：
- recovery 中直接返回，因为不能写 WAL。
- 如果 `pd_prune_xid` invalid，直接返回。
- 用 `GlobalVisTestIsRemovableXid()` 判断这个 hint 是否已经到可移除 horizon。
- 检查 `PD_PAGE_FULL` 或 free space 低于 fillfactor 目标。
只有这些条件成立，才尝试 `ConditionalLockBufferForCleanup()`。
拿不到 cleanup lock 就返回。
这说明 pruning 是 opportunistic。
它不能为了每次 scan 都阻塞等待。
`heap_prepare_pagescan()` 在 `heapam.c:611-646` 展示普通 pagemode scan 的入口。
它先 prune，再用 shared buffer lock 检查 tuple visibility。
bitmap heap scan 和 index scan 也有类似入口。
真正计划 pruning 的核心是 `prune_freeze_plan()`。
它在 `pruneheap.c:531-705`。
这个函数有一个关键不变量：
先计算每个 tuple 的 HTSV 结果，存入 `PruneState.htsv`，再处理 HOT chain。
`pruneheap.c:545-550` 说明为什么不能对同一个 tuple 反复跑 HTSV。
另一个 tuple 的检查可能推进 global visibility test。
一个 `RECENTLY_DEAD` 结果可能变成 `DEAD`。
insert in progress 也可能在第二次检查时已经 abort。
如果边看边改，pruning 决策会不稳定。
所以 pruning 分两阶段：
```text
plan: 读取 page，计算 htsv，收集 redirect/dead/unused/freeze 计划
execute: 进入 critical section，批量应用 line pointer 和 tuple 修改
```
`heap_prune_chain()` 在 `pruneheap.c:1451-1682` 处理 HOT chain。
它从 root offset 开始。
如果 root 是 `LP_REDIRECT`，先跳到 redirect target。
如果 line pointer 是 `LP_NORMAL`，读 tuple header。
如果有 `priorXmax`，要求当前 tuple 的 `xmin` 等于前一个版本的 update XID。
这就是防止跟到被 VACUUM 复用的无关 slot。
chain 的处理结果有三类：
- 没有 dead tuple，保持不变。
- 整条 chain dead，root 变 `LP_DEAD` 或 `LP_UNUSED`，其它成员变 `LP_UNUSED`。
- chain 前缀 dead，root 变 `LP_REDIRECT` 指向第一个 non-dead successor，前缀中间成员变 `LP_UNUSED`。
`heap_page_prune_execute()` 在 `pruneheap.c:2055-2223` 应用这些变更。
它按三组数组执行：
- `redirected[]` 调 `ItemIdSetRedirect()`。
- `nowdead[]` 调 `ItemIdSetDead()`。
- `nowunused[]` 调 `ItemIdSetUnused()`。
然后调用 `PageRepairFragmentation()`。
这一步把被移除 tuple bytes 产生的洞压缩掉。
但仍保留不能丢的 line pointer 入口。
`LP_REDIRECT` 的注释在 `pruneheap.c:2111-2122` 说明了设计目的。
原始 root tuple 被剪掉后，仍需要保留一个 redirect item。
这样 VACUUM 之后仍能知道应该从 index 删除哪个 TID。
而 heap-only tuple 永远不能变成 `LP_DEAD`。
`LP_DEAD` 的注释在 `pruneheap.c:2146-2158` 说明另一条边界。
当原始 item 可能仍被 index TID 引用时，必须留下 `LP_DEAD`。
这给后续 index vacuum 一个清理目标。
执行阶段在 `pruneheap.c:1230-1314` 的 critical section 中。
如果只是更新 `pd_prune_xid` 或清 `PD_PAGE_FULL`，可能只是 dirty hint。
如果真的 prune、freeze 或设置 VM，会 mark dirty 并写 `XLOG_HEAP2_PRUNE*`。
这就是本节主问题的闭环：
line pointer 让 TID 稳定。
tuple header 让旧版本延迟可见。
pruning 用 line pointer 状态表达“入口还在、storage 可移除、index 后续再清”的中间态。
## 15. 生命周期、ownership 与 cleanup
heap page 的持久状态在 relation main fork 的 block 中。
它通过 buffer manager 映射到 shared buffer。
backend 访问 page 时必须持有 buffer pin。
读 tuple header 需要相应 buffer content lock 或调用路径保证。
修改 page 需要 exclusive content lock。
pruning 通常需要 cleanup lock。
`HeapTupleData` 的 ownership 分两种。
如果它指向 buffer page，调用者不拥有 tuple bytes。
buffer pin 生命周期决定 pointer 是否安全。
如果它来自 palloc copy，内存由当前 MemoryContext 管理。
`heap_insert()` 和 `heap_update()` 中 TOAST 可能返回 private tuple，完成后用 `heap_freetuple()` 释放。
事务 abort 不会回滚 page bytes 到插入前。
已经插入的 tuple 仍在 page 上。
它的 `xmin` 最终会被判断为 aborted。
visibility 函数可以设置 `HEAP_XMIN_INVALID` hint。
pruning 之后才能把它的 line pointer 变成 `LP_UNUSED` 或移除 storage。
DELETE/UPDATE abort 也类似。
旧 tuple header 上可能已经写了 `xmax`。
如果事务 abort，后续可见性判断会设置 `HEAP_XMAX_INVALID` hint。
tuple 仍被认为 live。
这就是为什么 delete/update 的物理动作必须和事务状态解释分离。
ERROR cleanup 的边界有两个。
普通等待、TOAST、内存分配等可能 ERROR 的工作要尽量放在 critical section 之前。
进入 critical section 后，代码要求“不抛普通 ERROR，直到 WAL 记录完成”。
`RelationPutHeapTuple()` 如果插入失败直接 `PANIC`。
这是 crash safety 的选择，不是用户级错误处理。
line pointer 的 cleanup 是分阶段的。
`LP_NORMAL` 可以变 `LP_REDIRECT`、`LP_DEAD` 或 `LP_UNUSED`。
`LP_REDIRECT` 通常保留 HOT root 入口。
`LP_DEAD` 等待 index vacuum 语义。
`LP_UNUSED` 才能被 `PageAddItemExtended()` 复用。
尾部连续 `LP_UNUSED` 可以让 `PageRepairFragmentation()` 或 `PageTruncateLinePointerArray()` 缩短 line pointer array。
## 16. 异常路径和 fallback
第一类异常是 page header 损坏。
`PageAddItemExtended()` 在 `bufpage.c:220-227` 检查 `pd_lower/pd_upper/pd_special`。
如果 page pointer 不 sane，报 `PANIC`。
`PageRepairFragmentation()` 在 `bufpage.c:732-740` 做类似检查，但报 `ERROR`。
区别来自调用上下文和已经修改共享页的风险。
第二类异常是 line pointer 不是预期状态。
普通 scan 遇到非 `LP_NORMAL` 会跳过。
`heap_fetch()` 遇到 offset 超界或非 `LP_NORMAL` 返回 false。
`heap_update()` 对 syscache 来源的 pruned TID 有特殊处理。
`heapam.c:3317-3359` 说明，如果没有 pin/snapshot，可能看到 `LP_UNUSED` 或 `LP_DEAD`。
它选择返回 `TM_Deleted`，并在日志/错误语义上保守处理。
第三类异常是 concurrent updater/locker。
`heap_delete()` 和 `heap_update()` 都可能看到 `TM_BeingModified`。
它们会释放 buffer lock，获取 tuple lock，等待事务或 MultiXact。
等待后必须重新锁 buffer 并检查 `xmax/infomask` 是否变化。
如果变化，跳回重新判断。
这就是 tuple header 并发协议的 retry loop。
第四类异常是 VM pin 竞态。
DML 修改 all-visible page 时需要清 VM bit。
如果锁 page 前没 pin VM page，但锁住后发现 page all-visible，就必须释放 heap page lock，pin VM，再重试。
`heap_delete()` 的 `l1` loop 和 `heap_update()` 的 `l2` loop 都体现这个边界。
第五类异常是 on-access pruning 拿不到 cleanup lock。
`heap_page_prune_opt()` 使用 `ConditionalLockBufferForCleanup()`。
拿不到就返回。
这不是错误。
系统宁可暂时不回收空间，也不让普通读路径为了 pruning 阻塞。
第六类异常是 recovery。
`heap_page_prune_opt()` 在 recovery 中直接返回。
recovery 不能产生新的 pruning WAL。
主库产生的 prune WAL 会在 replay 中恢复 page 变更。
第七类异常是 hint bit 不能安全设置。
`SetHintBitsExt()` 发现 commit LSN 尚未 flush 且 page LSN 不能提供 interlock，就暂时不设置 committed hint。
tuple 语义不受影响。
代价是下次可见性判断还要再查事务状态。
## 17. 成本、资源与跨模块传播
heap page layout 的 hot path 成本主要有四类。
第一是 line pointer 扫描成本。
顺序 scan 遍历 offset number，而不是 live tuple 数。
如果 page 上有很多 `LP_DEAD`、`LP_REDIRECT`、`LP_UNUSED`，scan 仍要检查 line pointer。
pruning 能降低后续 tuple visibility 成本，但不能免费。
第二是 visibility check 成本。
没有 hint bit 时，`HeapTupleSatisfiesMVCC()` 可能查询事务状态。
如果涉及 MultiXact，还可能进入更重的路径。
hint bit 能摊销成本，但设置 hint bit 又受 WAL flush 和 buffer I/O 状态限制。
第三是 page fragmentation 成本。
UPDATE/DELETE 留下死版本。
HOT chain 和 dead line pointer 增加 page 内部碎片。
`PageRepairFragmentation()` 需要移动 tuple bytes 并更新 line pointer offset。
这通常需要 cleanup lock。
第四是空间压力传播。
如果 page free space 不够，UPDATE 可能放到新 page。
这会增加 index 维护，破坏 HOT 机会，扩大 table bloat。
旧 page 设 `PD_PAGE_FULL`，后续访问尝试 pruning。
如果长事务拖住 horizon，pruning 不能移除 recently dead tuple，空间压力继续传播到 FSM、autovacuum 和更多 page extension。
跨模块边界至少有这些：
- Buffer manager 提供 pin、content lock、cleanup lock 和 dirty/LSN 协议。
- WAL 记录 heap insert/update/delete/prune 的 page 修改。
- Transaction/CLOG/MultiXact 提供 `xmin/xmax` 状态解释。
- Snapshot/ProcArray/GlobalVisState 决定当前可见性和可回收 horizon。
- FSM 只保存 free space 近似值，不保存 tuple 真相。
- VM 与 `PD_ALL_VISIBLE` 协作优化可见性判断，但不是 tuple header 的替代。
- Index AM 持有 TID，依赖 heap 保持必要 line pointer 入口。
- autovacuum/VACUUM 推进 pruning、index cleanup 和 freeze。
如果出现 heap bloat，不能只盯着 `n_dead_tup`。
要同时问：
- 是否有长事务或 replication slot 拖住 horizon。
- UPDATE 是否破坏 HOT 条件。
- page 是否有大量 redirect/dead line pointer。
- FSM 是否还认为 page 有可用空间。
- autovacuum 是否及时执行第二阶段 index cleanup。
- workload 是否频繁更新 indexed columns。
## 18. 观测与诊断入口
最直接的观测工具是 `pageinspect`。
`pageinspect--1.5.sql:22-32` 定义 `page_header()`。
它能看到：
```text
lsn
checksum
flags
lower
upper
special
pagesize
version
prune_xid
```
`pageinspect--1.5.sql:38-54` 定义 `heap_page_items()`。
它能看到：
```text
lp
lp_off
lp_flags
lp_len
t_xmin
t_xmax
t_field3
t_ctid
t_infomask2
t_infomask
t_hoff
t_bits
t_data
```
这些字段是 raw observation。
不要把它们直接当语义。
例如：
- `t_xmax` 非零不等于 deleted。
- `lp_flags = 2` 表示 redirect，不是 tuple bytes offset。
- `prune_xid` 非零不等于一定能 prune。
- `lower/upper` 只能说明连续 free space，不说明所有 dead tuple 是否可回收。
`heap_tuple_infomask_flags()` 可以把 raw infomask 转成 flag 名称。
它有助于课堂实验。
但 flag 名称仍不是完整可见性结果。
完整结果还依赖 snapshot 和 transaction status。
SQL 层可以看见：
- system column `ctid`。
- `xmin` 和 `xmax` system columns。
- `VACUUM VERBOSE` 中 dead/removable 相关信息。
- `pg_stat_all_tables.n_dead_tup` 的估计。
- `pg_stat_user_tables.n_tup_hot_upd` 的 HOT update 计数。
SQL 层看不见或只能间接推断：
- 某个 tuple 对某个 snapshot 的完整 `HeapTupleSatisfiesMVCC()` 分支。
- `GlobalVisTestIsRemovableXid()` 当前内部判断。
- page pruning 是否因为 cleanup lock 失败而跳过。
- hint bit 未设置是因为事务仍在 snapshot 中，还是 commit LSN safety 不满足。
源码断点推荐：
```gdb
break RelationPutHeapTuple
break HeapTupleSatisfiesMVCC
break HeapTupleSatisfiesVacuumHorizon
break heap_page_prune_opt
break heap_prune_chain
break heap_page_prune_execute
break PageRepairFragmentation
```
观察变量：
```gdb
p ((PageHeader) page)->pd_lower
p ((PageHeader) page)->pd_upper
p ((PageHeader) page)->pd_prune_xid
p *itemId
p *tuple
p tuple->t_infomask
p tuple->t_infomask2
p tuple->t_ctid
```
如果只做 SQL 诊断，建议形成这个闭环：
```text
pageinspect 看到 lp/t_xmin/t_xmax/t_ctid/infomask
-> 结合当前事务和 snapshot 解释可见性
-> 对照 heapam_visibility.c 分支
-> VACUUM 或访问触发 pruning
-> 再看 lp_flags/lower/upper/prune_xid 是否变化
```
## 19. 课堂实验一：看 insert 后的 page/header/line pointer
目标：确认 insert 建立的是 line pointer 和 tuple header，而不是“直接把行放进一个数组”。
准备：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS heap_layout_demo;
CREATE TABLE heap_layout_demo (
    id int PRIMARY KEY,
    v int,
    pad text
) WITH (fillfactor = 100, autovacuum_enabled = false);
INSERT INTO heap_layout_demo VALUES (1, 10, repeat('a', 80));
CHECKPOINT;
```
观察 page header：
```sql
SELECT *
FROM page_header(get_raw_page('heap_layout_demo', 0));
```
观察 line pointer 和 tuple header：
```sql
SELECT h.lp,
       h.lp_off,
       h.lp_flags,
       h.lp_len,
       h.t_xmin,
       h.t_xmax,
       h.t_ctid,
       h.t_infomask,
       h.t_infomask2,
       h.t_hoff,
       f.raw_flags,
       f.combined_flags
FROM heap_page_items(get_raw_page('heap_layout_demo', 0)) AS h
LEFT JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) AS f
ON h.lp_flags = 1
ORDER BY h.lp;
```
回到源码解释：
- `lp = 1` 对应 `PageAddItemExtended()` 返回的 offset number。
- `lp_flags = 1` 对应 `LP_NORMAL`。
- `t_ctid = (0,1)` 来自 `RelationPutHeapTuple()`。
- `t_xmin` 是插入事务。
- `t_xmax` 通常为 0，且 `HEAP_XMAX_INVALID` 才是语义。
- `t_hoff` 是 user data 起点。
继续执行：
```sql
SELECT ctid, xmin, xmax, *
FROM heap_layout_demo;
```
比较 SQL system columns 和 pageinspect 的 header 字段。
注意 SQL 的 `ctid` 来自 tuple self TID。
pageinspect 的 `t_ctid` 是 header 内字段。
新插入最新版本时二者相同。
UPDATE 后它们可能不同。
## 20. 课堂实验二：看 HOT update 和 `t_ctid`
目标：确认 UPDATE 生成新 tuple version，旧版本用 `t_ctid` 指向新版本。
执行：
```sql
UPDATE heap_layout_demo SET v = 11 WHERE id = 1;
UPDATE heap_layout_demo SET v = 12 WHERE id = 1;
UPDATE heap_layout_demo SET v = 13 WHERE id = 1;
```
观察：
```sql
SELECT h.lp,
       h.lp_flags,
       h.lp_off,
       h.lp_len,
       h.t_xmin,
       h.t_xmax,
       h.t_ctid,
       h.t_infomask,
       h.t_infomask2,
       f.raw_flags,
       f.combined_flags
FROM heap_page_items(get_raw_page('heap_layout_demo', 0)) AS h
LEFT JOIN LATERAL heap_tuple_infomask_flags(h.t_infomask, h.t_infomask2) AS f
ON h.lp_flags = 1
ORDER BY h.lp;
```
预期现象：
- 会出现多个 `LP_NORMAL`。
- 较老版本的 `t_ctid` 指向后续 offset。
- 某些旧版本可能带 `HEAP_HOT_UPDATED`。
- 后续版本可能带 `HEAP_ONLY_TUPLE`。
- SQL 查询只返回当前 snapshot 可见的一个版本。
回到源码：
- 新 tuple 由 `RelationPutHeapTuple()` 插入。
- 旧 tuple 的 `t_ctid` 在 `heapam.c:4059-4060` 改写。
- HOT bits 在 `heapam.c:4029-4037` 设置。
- MVCC 可见性由 `HeapTupleSatisfiesMVCC()` 决定。
如果没有观察到 HOT：
- 检查 UPDATE 是否修改了 indexed column。
- 检查新版本是否仍能放在同一 page。
- 检查 table fillfactor 和 tuple size。
- 检查是否有其它 index 阻止 HOT。
## 21. 课堂实验三：看 pruning 改 line pointer
目标：确认空间回收先改变 line pointer 状态，再通过 page repair 压缩 tuple bytes。
先观察更新后的 header：
```sql
SELECT lower, upper, special, flags, prune_xid
FROM page_header(get_raw_page('heap_layout_demo', 0));
```
执行 VACUUM：
```sql
VACUUM heap_layout_demo;
```
再次观察：
```sql
SELECT lower, upper, special, flags, prune_xid
FROM page_header(get_raw_page('heap_layout_demo', 0));
SELECT lp,
       lp_flags,
       lp_off,
       lp_len,
       t_xmin,
       t_xmax,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('heap_layout_demo', 0))
ORDER BY lp;
```
你可能看到：
- 部分旧版本变成 `LP_UNUSED`。
- HOT root 可能变成 `LP_REDIRECT`。
- 整条 dead chain 的 root 可能变成 `LP_DEAD`。
- `upper` 变大，说明 tuple bytes 被 compact 后连续 free space 增加。
- `lower` 可能因尾部连续 `LP_UNUSED` 被截断而变小。
- `prune_xid` 可能被清零或更新为新的最低 soon-prunable XID。
回到源码：
- `heap_page_prune_opt()` 决定是否尝试。
- `prune_freeze_plan()` 计算每个 tuple 的 HTSV。
- `heap_prune_chain()` 计算 redirect/dead/unused。
- `heap_page_prune_execute()` 应用 line pointer 修改。
- `PageRepairFragmentation()` 压缩 storage。
长事务边界实验：
```sql
-- session A
BEGIN;
SELECT * FROM heap_layout_demo;
-- session B
UPDATE heap_layout_demo SET v = 14 WHERE id = 1;
VACUUM heap_layout_demo;
-- session A
COMMIT;
-- session B
VACUUM heap_layout_demo;
```
比较 session A commit 前后的 pageinspect 输出。
如果旧 snapshot 拖住 horizon，某些 recently dead tuple 不能变成 removable。
这能把 MVCC 可见性和空间复用的冲突直接观察出来。
## 22. 常见误区
误区一：把 `xmax` 有值理解成 tuple 已删除。
正确说法：`xmax` 可能是 locker、updater、deleter、MultiXact 或 aborted transaction。
必须结合 infomask、snapshot 和事务状态。
误区二：把 `ctid` 和 `t_ctid` 混为一谈。
正确说法：`HeapTupleData.t_self` 表示当前物理位置。
`HeapTupleHeaderData.t_ctid` 是版本链字段或特殊 token。
误区三：认为 pruning 会随 DELETE 立即发生。
正确说法：DELETE 只改 header。
pruning 依赖 horizon、cleanup lock、page fullness/free space heuristic 和 WAL 条件。
误区四：认为 `LP_DEAD` 已经可以被普通 insert 复用。
正确说法：普通可立即复用的是 `LP_UNUSED`。
`LP_DEAD` 可能还要等待 index cleanup 语义。
误区五：把 `pd_prune_xid` 当成准确事实。
正确说法：它是 hint。
它用于避免无意义 pruning，但不替代 tuple visibility 判断。
误区六：看到 `lower/upper` 间空间小就断言 page 无法回收。
正确说法：`lower/upper` 表示连续 free space。
page 内可能有 dead tuple storage，需要 pruning/repair 后才变成连续空间。
误区七：认为 hint bit 只是性能优化，不涉及正确性边界。
正确说法：hint bit 本身优化可见性判断，但 committed hint bit 的设置要遵守 WAL flush/page LSN interlock。
## 23. 讨论题
1. 为什么 index TID 指向 line pointer 编号，而不是 tuple bytes 的 byte offset？
2. 为什么 `LP_REDIRECT` 要把 `lp_off` 解释成 offset number，而不是保留旧 tuple 的 storage？
3. 为什么 `LP_DEAD` 不能总是直接变成 `LP_UNUSED`？
4. 如果 `xmax` 是 MultiXact，为什么 `HeapTupleHeaderGetUpdateXid()` 可能变贵？
5. 为什么 `HeapTupleSatisfiesMVCC()` 在某些情况下故意不立即设置 hint bit？
6. 为什么 `heap_page_prune_opt()` 拿不到 cleanup lock 时选择返回，而不是等待？
7. HOT update 为什么要求新版本在同一 page，并且不能修改 HOT blocking index 列？
8. `t_hoff` 损坏会影响哪些路径？它会不会直接改变 MVCC 可见性？
9. 长事务如何把 DELETE 的可见性结果和空间回收结果分开？
10. pageinspect 能看到哪些 raw field？哪些结论必须回到源码或 snapshot 才能判断？
## 24. 本节小结
heap page 的核心不是“page 里放了一堆 tuple”。
它是一个分层协议。
`PageHeaderData` 管 page 级空间边界、LSN、all-visible 和 pruning hint。
`ItemIdData` 管稳定 offset number 到 tuple storage 的间接映射。
`HeapTupleHeaderData` 管单个 tuple version 的事务语义、版本链和 user data 起点。
`Snapshot`、transaction status、MultiXact 和 hint bits 把 raw header 转成可见性结果。
`pruneheap.c` 再把“所有相关 snapshot 都不需要的版本”转成 line pointer 状态变化和 page 内空间压缩。
本节主链路是：
```text
insert 建立 LP_NORMAL 和 xmin/t_ctid=self
scan 通过 line pointer 找 tuple，再用 snapshot 解释 header
update 插入新 version，并让旧 version 的 xmax/t_ctid 指向新语义
delete 只写 xmax，不立即移除 storage
pruning 在 horizon 允许后把 HOT chain 前缀变成 redirect/dead/unused 并 repair fragmentation
```
本节最重要的不变量：
- TID 稳定性来自 line pointer，不来自 tuple bytes 固定位置。
- MVCC 可见性来自 header bits 加 snapshot，不来自单个字段。
- 空间复用必须晚于可见性安全边界。
- pruning 可以移动 tuple bytes，但不能破坏仍可能被 index 引用的 TID 入口。
- hint 字段可以加速判断，但不能替代 correctness check。
可迁移的系统规律：
当一个系统必须同时支持稳定外部引用、版本化可见性和延迟回收时，通常需要一个间接层。
PostgreSQL heap 的 line pointer 就是这个间接层。
它把“外部定位稳定”和“内部物理布局可变”拆开。
tuple header 再把“版本语义”和“物理存在”拆开。
pruning 则把“逻辑不可见”和“物理可复用”之间的延迟显式化。
诊断时不要问“这行为什么还在 page 上”。
要按层问：
- line pointer 还在什么状态？
- tuple header 表示哪个版本语义？
- 当前 snapshot 和 global horizon 是否允许移除？
- index 是否仍可能引用这个 offset？
- page 是否已经有机会拿到 cleanup lock 并写出 prune WAL？
只有把这五个问题连起来，才能解释 heap page 如何同时支持定位、MVCC 可见性、空间复用和后续 pruning。
