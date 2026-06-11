# PostgreSQL B-tree search、scankey 与并发页面移动

## 课程定位

前置知识：
- 已理解 buffer pin 和 buffer content lock 的区别。
- 已理解 relation fork、page special area、WAL-before-data。
- 已理解 index scan 返回 TID 后还要回 heap 做 MVCC 可见性判断。
- 已理解 B-tree 的基本排序性质、leaf/internal page、root 和 downlink。
- 已理解 PostgreSQL 中 `ScanKey` 是执行器和访问方法之间传递谓词的载体。
本节唯一主问题：
`nbtree` 为什么能在只短暂锁住单个页面的情况下，仍然在并发 split 发生后找到正确的 key range？
本节围绕的核心矛盾：
搜索路径希望极短。
它不能从 root 到 leaf 长时间持有整条路径的锁。
否则普通查询会和插入、split、VACUUM 形成严重 contention。
但 B-tree 页面 split 会改变父子 downlink 和 leaf keyspace 的边界。
搜索如果只相信刚才读到的 parent downlink，就可能落到已经被 split 后的旧页面。
PostgreSQL 的选择是：
每个非右端页面保存 high key。
每个页面保存 rightlink。
搜索下降时只把 parent downlink 当成一个候选入口。
真正的定位在 child 页面上用 high key 重新验证。
如果搜索 key 已经越过当前页面的 high key，就沿 rightlink 追赶到新的 sibling。
这把“并发 split 让 parent 指针暂时过期”的问题，转化成“在同一层向右追赶到正确页面”的问题。
学完本节，你应该能独立判断：
- `_bt_search()` 返回的是第一个可能包含目标 key 的 leaf 页面，不是全扫描结果。
- `_bt_moveright()` 为什么要在每一层下降后执行，而不是只在 leaf 执行。
- `_bt_moveright()` 为什么比较 page high key，而不是回头重读 parent。
- `search-type scankey` 和 `insertion-type scankey` 为什么必须区分。
- `_bt_preprocess_keys()` 的核心产物不是“更少的 qual”，而是 `SK_BT_REQFWD/SK_BT_REQBKWD` 边界语义。
- `_bt_first()` 如何从预处理后的 search keys 制造临时 insertion scankey。
- `nextkey`、`backward`、`scantid` 如何影响初始定位。
- split 为什么先让 reader 可通过 rightlink 找到右半页，再补 parent downlink。
- `BTP_INCOMPLETE_SPLIT` 标志为什么在左页，而缺失 downlink 的是右页。
- read path 为什么通常不修复 incomplete split，write path 为什么必须修复。
- PostgreSQL 这里说的 latch coupling 更准确地落在 buffer pin/content lock coupling 上。
- search 下降路径为什么通常不做 parent-child lock coupling。
- split 写路径在哪些地方必须短暂持有相邻 page 或 parent-child 锁。
- 哪些现象能用 SQL 和 `pageinspect` 观察，哪些只能用 gdb、日志或源码注入点推断。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `nl -ba <source-file>`；重点源码和对照入口统一放在第 3 节的阅读顺序里。

这不是把课程扩展成插入课或 pageinspect 课。
`nbtinsert.c` 只用于解释 split 如何给 `_bt_moveright()` 留下可恢复状态。
`nbtreadpage.c` 只用于解释 `_bt_preprocess_keys()` 的 required flags 如何在 page 内扫描时收束 `continuescan`。

## 1. 本节在总主线中的位置

前面几节从 heap page、FSM/VM、WAL 和 buffer 边界讲到了“一个 page 内部状态怎么安全变化”。
本节向 index AM 的查找路径移动。
heap 的问题通常是：
给定一个 `(block, offset)`，怎样解释 tuple version。
B-tree search 的问题是：
还没拿到 heap TID 前，怎样在一个会被并发 split 改写边界的索引结构里找到起点。
这不是普通数据结构课里的 B-tree 查找。
内存里的单线程 B-tree 可以从 root 一路跟 child pointer 到 leaf。
PostgreSQL 的 nbtree 不能这样想。
它的页面在 shared buffers 中被多个 backend 同时读写。
它的 split 需要 WAL。
它的 leaf scan 要把 TID 返回给执行器。
它的 page deletion 和 VACUUM 还会让物理 sibling 链出现暂时需要绕行的状态。
本节只抓住一个主流程链路：

```text
executor search qual
  -> search-type ScanKey
  -> _bt_preprocess_keys()
  -> _bt_first() 选择起始边界
  -> 临时 BTScanInsertData
  -> _bt_search()
  -> 每层 _bt_moveright()
  -> leaf 上 _bt_binsrch()
  -> _bt_readfirstpage()/后续 leaf scan
```

这条链路对应真实 runtime 现象：
并发插入触发 leaf split 时，读者不需要重启整棵树搜索。
它可能落到旧左页。
它随后用 high key 发现自己已经越界。
它沿 `btpo_next` 追赶到右页。
如果 split 尚未补 parent downlink，读者仍然能找到数据。
如果写者要在一个 incomplete split 页面上继续插入，它会先把 split 补完。
本节不讲完整 unique check。
本节不讲 bottom-up deletion 的全部策略。
本节不讲 dedup posting list 的空间优化细节。
这些都会出现，但只服务一个问题：
搜索如何在页面移动边界时继续找对位置。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
PostgreSQL nbtree 搜索只把 parent downlink 当作候选入口；
child page 的 high key/rightlink 才是并发 split 后修正方向的运行时事实。
```

普通 B-tree 查找依赖一个隐含条件：
父节点指向的子节点 key range 在查找期间不变。
PostgreSQL 不能要求这个条件。
如果读者持有 parent read lock，再持有 child read lock，再释放 parent，这就是典型 lock coupling。
如果从 root 到 leaf 都做这种 coupling，读路径会变重。
如果写者 split 时还要等待大量读者的跨层锁链，插入延迟会扩散。
Lehman-Yao B-link tree 的关键思想是：
给每个页面加 rightlink。
给页面加 high key。
这样 parent downlink 可以暂时过期。
只要页面自身告诉你“我的右边界在哪里”，搜索就能在同一层向右修正。
PostgreSQL 的 README 在开头说明了这个模型。
`src/backend/access/nbtree/README:17-29` 说，right-link 和 high key 让搜索能检测 concurrent page split。
如果 search key 大于 high key，就跟 rightlink 找包含目标 key range 的新页面。
这个动作可能重复，因为页面可能 split 不止一次。
本节要强调两个边界。
第一个边界：
read path 需要 page-level read lock。
Lehman-Yao 原论文假设内存 page copy 不共享。
PostgreSQL 的 shared buffers 是共享内存。
所以 README `63-68` 明确说 PostgreSQL 对 btree page 做 page-level read locking。
这个锁不是事务语义锁。
它只是保护读页面内容时没有 writer 改写该 page。
第二个边界：
read path 不长期持有 parent lock。
`_bt_search()` 注释 `87-94` 明确说，返回时 leaf buffer locked and pinned，但 parent pages 不持锁。
搜索每一层只持有当前 page 的 pin 和 lock。
选中 downlink 后，通过 `_bt_relandgetbuf()` 释放当前页面并获取 child。
这就制造了 race：
刚才 parent 上看到的 downlink 可能已经被并发 split 弄旧。
`_bt_search()` 在下一轮开始调用 `_bt_moveright()` 来修正。
所以本节的核心矛盾 不是“B-tree 如何二分查找”。
真正的矛盾是：
搜索必须快到几乎只锁单页。
但 split 会让页面 keyspace 向右移动。
系统必须让短锁搜索仍然有可恢复路径。
这也是为什么 high key/rightlink 不是 pageinspect 里好看的元数据。
它们是运行时正确性边界。

## 3. 核心文件分工与阅读顺序

推荐按运行链路读，不按文件名读。

| 顺序 | 文件 | 读什么 | 为什么 |
| --- | --- | --- | --- |
| 1 | `src/backend/access/nbtree/README` | Lehman-Yao rightlink/high key、锁规则、split 两阶段 | 先建立 mental model |
| 2 | `src/include/access/nbtree.h` | `BTPageOpaqueData`、`BTScanInsertData`、`BTScanOpaqueData`、flag 宏 | 先知道状态在哪里 |
| 3 | `src/backend/access/nbtree/nbtpreprocesskeys.c` | `_bt_preprocess_keys()`、required flags、array/skip array | search qual 如何变成可定位边界 |
| 4 | `src/backend/access/nbtree/nbtsearch.c` | `_bt_first()`、`_bt_search()`、`_bt_moveright()`、`_bt_binsrch()` | 主查找链路 |
| 5 | `src/backend/access/nbtree/nbtpage.c` | `_bt_getroot()`、`_bt_getbuf()`、`_bt_relandgetbuf()`、lock wrapper | pin/lock 生命周期 |
| 6 | `src/backend/access/nbtree/nbtutils.c` | `_bt_mkscankey()`、`_bt_truncate()` | insertion scankey 和 split high key |
| 7 | `src/backend/access/nbtree/nbtinsert.c` | `_bt_split()`、`_bt_insert_parent()`、`_bt_finish_split()`、`_bt_getstackbuf()` | split 给搜索留下什么并发状态 |
| 8 | `src/backend/access/nbtree/nbtreadpage.c` | `_bt_readpage()`、`_bt_checkkeys()` | 预处理 flags 如何结束 leaf scan |
| 9 | `contrib/pageinspect` | `bt_metap()`、`bt_page_stats()`、`bt_page_items()` | 观察 high key、rightlink 和 page flags |

不要先从 `btgettuple()` 顺读。
`btgettuple()` 是 AM 对外入口。
本节的关键是 search key 如何定位 leaf。
也不要从 `_bt_split()` 的空间计算开始。
split point 的选择会很有趣，但不是本节主问题。
先抓住这些入口：

```text
_bt_preprocess_keys()
_bt_first()
_bt_search()
_bt_moveright()
_bt_split()
_bt_insert_parent()
_bt_finish_split()
```

读源码时优先找状态变化：
`ScanKey` 变成 `so->keyData[]`。
`so->keyData[]` 变成 `BTScanInsertData inskey`。
parent downlink 变成 child buffer。
child high key 判断变成 rightlink 追赶。
split 把 old page 变成 left page。
split 创建 right page。
parent downlink 延迟补入。
`BTP_INCOMPLETE_SPLIT` 标志清除。

## 4. 关键状态与边界

先看 page-level 状态。
`BTPageOpaqueData` 在 `src/include/access/nbtree.h:63-70`。
它位于 page special area，不是普通 tuple。
`btpo_next` 是 rightlink，`btpo_prev` 是 leftlink。
`btpo_level` 区分 leaf 和 internal 层级。
`btpo_flags` 保存 leaf/root/deleted/half-dead/incomplete split 等状态。
`btpo_cycleid` 主要服务 VACUUM 处理 concurrent split。
本节最重要的 flags 是 `BTP_LEAF`、`BTP_ROOT`、`BTP_DELETED`、`BTP_HALF_DEAD`、`BTP_SPLIT_END` 和 `BTP_INCOMPLETE_SPLIT`。
对应宏在 `src/include/access/nbtree.h:219-228`。
`P_RIGHTMOST(opaque)` 由 `btpo_next == P_NONE` 判断。
这意味着右端页面没有 high key。
`P_IGNORE(opaque)` 同时覆盖 deleted 和 half-dead。
搜索遇到 `P_IGNORE` 通常要沿 sibling 链绕过去。
`P_INCOMPLETE_SPLIT(opaque)` 表示左页已经 split，但右页的 downlink 还没有插入 parent。
注意语义方向：
标志在左页。
缺失 parent downlink 的是右页。
因为从左页沿 rightlink 到右页时，系统马上知道“这个右页还需要补 downlink”。
再看 high key。
`src/include/access/nbtree.h:350-365` 解释 high key。
Lehman-Yao 要求每个非右端 page 有 high key。
它是该页面 key range 的上界。
非右端页面的 high key 位于 item 1。
`P_HIKEY` 是 offset 1。
`P_FIRSTDATAKEY(opaque)` 根据是否 rightmost 选择第一个 data item。
右端页没有 high key，所以 data item 从 item 1 开始。
非右端页 data item 从 item 2 开始。
这解释了一个常见调试误区：
`bt_page_items()` 看到 itemoffset 1，不一定是用户 tuple。
在非右端 leaf page 上，它可能是 high key。
再看 scan 状态。
`BTScanOpaqueData` 在 `src/include/access/nbtree.h:1053-1095`。
它是一个 index scan 的 backend-local 状态。
这里最关键的是 `keyData`。
它不是执行器原始传入的 `scan->keyData[]`。
它是 `_bt_preprocess_keys()` 之后的私有副本。
它被加上了 nbtree 私有 flags。
这些 flags 在 `src/include/access/nbtree.h:1100-1117`。
本节最重要的是 `SK_BT_REQFWD`、`SK_BT_REQBKWD`、`SK_BT_SKIP`、`SK_BT_MINVAL/MAXVAL`、`SK_BT_NEXT/PRIOR`、`SK_BT_DESC` 和 `SK_BT_NULLS_FIRST`。
`SK_BT_REQFWD` 的意思不是“这个 qual 必须为真”。
所有 scan qual 本来都必须为真才能返回 tuple。
它的意思是：
向前扫描时，如果这个 key 不满足，就可以停止当前 primitive scan。
`SK_BT_REQBKWD` 同理用于向后扫描。
这就是 scankey preprocessing 的核心。
它给 `_bt_first()` 和 `_bt_checkkeys()` 一个一致的边界模型。
再看 insertion scankey。
`BTScanInsertData` 在 `src/include/access/nbtree.h:795-805`。
它叫 insertion scankey，但不只用于插入。
`_bt_first()` 也会临时构造它，用它调用 `_bt_search()` 找扫描起点。
`nextkey=false` 表示找第一个 `>= key` 的 item。
`nextkey=true` 表示找第一个 `> key` 的 item。
`backward=true` 表示为 backward scan 找最后一个候选位置。
`scantid` 是 heap TID tie-breaker。
在 heapkeyspace index 中，它被当作排序 key 的最后一列。
这让 logical duplicate 在物理排序上仍然有全序。
本节要记住：
raw field 不是语义。
`btpo_next + high key + buffer lock` 才能解释 search 是否要右移。
`sk_strategy + SK_BT_REQFWD/SK_BT_REQBKWD + scan direction` 才能解释一个 key 是否能结束 scan。
`scantid + heapkeyspace + nextkey` 才能解释 `_bt_compare()` 的定位结果。

## 5. scankey preprocessing：从 qual 到可停止边界

执行器传给 AM 的 `ScanKey` 是 search-type scankey。
它表达谓词，例如：

```sql
WHERE x = 1 AND y < 4 AND z < 5
```

但 B-tree scan 不只是逐项检查谓词。
它还要决定：
从哪里开始。
什么时候可以停止。
哪些 key 可以用于二分定位。
哪些 key 只能用于过滤。
这就是 `_bt_preprocess_keys()` 的工作。
`src/backend/access/nbtree/nbtpreprocesskeys.c:84-201` 是最重要的注释。
它说明输入来自 `scan->keyData[]`。
输出复制到 `so->keyData[]`。
输出 keys 会被加上额外 flags。
这些 flags 包括 index column 的 DESC/NULLS FIRST 信息。
DESC 列会翻转 strategy。
更重要的是，它会标记 required keys。
例如 `x = 1 AND y < 4 AND z < 5`。
当向前扫描时：
`x = 1` 必须满足，否则后续 keyspace 已经离开 `x=1`。
`y < 4` 必须满足，否则后续 `y` 只会更大。
`z < 5` 不能单独决定停止。
因为还可能存在 `(1,3,z)`。
所以 README 式的直觉不够。
真正规则在 `_bt_preprocess_keys()` 注释 `113-128`。
leading equality keys 通常同时标记 `SK_BT_REQFWD` 和 `SK_BT_REQBKWD`。
第一个没有 equality 的属性：
`<` 和 `<=` 在 forward scan 中可结束 scan。
`>` 和 `>=` 在 backward scan 中可结束 scan。
后续属性通常不能单独结束 scan。
这个逻辑服务两个调用者。
第一个调用者是 `_bt_first()`。
它要选择 start boundary。
第二个调用者是 `_bt_checkkeys()`。
它要在 leaf page 内判断 tuple 是否匹配，以及是否还能继续。
这两个调用者必须对称。
`nbtpreprocesskeys.c:160-168` 明确说，`_bt_first` 和 `_bt_checkkeys` 必须可靠地同意哪些 key 用来开始或结束 scan。
否则 array scan advancing 会错误预测下一次 primitive scan 是否能前进。
预处理还做 redundancy elimination。
同一列上多个 `>` 或 `>=`，尽量只保留最紧的下界。
同一列上多个 `<` 或 `<=`，尽量只保留最紧的上界。
如果有 equality key，通常其它 inequality 都是冗余的。
`nbtpreprocesskeys.c:139-145` 总结了这点。
如果发现矛盾，例如 `x = 1 AND x > 2`，就设置 `so->qual_ok = false`。
`nbtpreprocesskeys.c:170-175` 明确这个路径。
`_bt_first()` 随后在 `nbtsearch.c:904-913` 直接返回 false。
这个 fallback 很重要。
它不是 planner 优化。
它是 AM 内部在 runtime 对 scan keys 的再验证。
预处理不能假设 opfamily 一定完整。
跨类型 operator 可能缺少某些比较。
`nbtpreprocesskeys.c:98-102` 说，如果 opfamily 缺少完整 cross-type operators，可能无法检测所有冗余或矛盾。
这不会破坏 correctness。
代价是扫描更宽。
源码选择的是：
能证明的就删。
不能证明的就保留。
保留时还要让 `_bt_first()` 和 `_bt_checkkeys()` 对 required keys 达成一致。
`_bt_unmark_keys()` 的注释 `1525-1536` 专门处理这种情况。
当无法消除某些冗余 key 时，它会确保同一属性没有多个 required `>` 或 `>=`，也没有多个 required `<` 或 `<=`。
这样 `_bt_first()` 不会用一组 key 定位，而 `_bt_checkkeys()` 用另一组 key 停止。
这就是“保守但一致”的设计。
array keys 是本版本必须提到的复杂性。
`_bt_preprocess_array_keys()` 从 `nbtpreprocesskeys.c:1822-1843` 开始说明。
它处理 `SK_SEARCHARRAY`，也可能生成 skip array keys。
skip scan 的目标是：
当前导列没有 equality 条件，但后续列有可用条件时，给前导列制造一个可枚举的 equality-like 边界。
skip array 可能使用 `MINVAL/MAXVAL/NEXT/PRIOR` sentinel。
`_bt_first()` 需要在初始定位时把这些 sentinel 解释成合适的 lower/upper bound。
如果 opclass 缺少 skip support，它可能退化。
这再次说明：
scankey preprocessing 不是简单“把 WHERE 条件排个序”。
它是在构造一个可被 B-tree order 安全消费的状态机。
这个状态机有三个长期不变量。
第一，输出 key 存在于 scan 的私有状态中，不能原地改执行器传入的 `scan->keyData[]`。
第二，required flags 必须和 scan direction 绑定。
一个 key 对 forward 是 required，不代表对 backward 也是 required。
第三，当无法证明冗余或矛盾时，不能为了性能假装知道。
只能保留更多 key，让 `_bt_checkkeys()` 做更多 tuple-level 判断。
本节后面的 `_bt_first()` 正是消费这些预处理结果。

## 6. `_bt_first()`：把 search-type keys 变成定位用的 insertion scankey

`_bt_first()` 位于 `src/backend/access/nbtree/nbtsearch.c:883`。
它的注释 `863-881` 说：
它寻找 scan 的第一个 item。
成功后把当前 page 的匹配 tuple 装入 `so->currPos`。
返回给调用者前会释放 page lock。
通常保留 pin。
如果是 `so->dropPin` scan，则连 pin 也释放。
最先发生的是 preprocessing。
`nbtsearch.c:898-902` 调用 `_bt_preprocess_keys(scan)`。
如果 `so->qual_ok` 为 false，`nbtsearch.c:904-913` 直接结束。
接着，如果是 parallel index scan，`_bt_first()` 可能先 seize parallel scan state。
这不是本节核心。
parallel scan 不能破坏单个 primitive scan 的定位规则。
`_bt_first()` 的关键在 `nbtsearch.c:956-1030` 那段长注释。
它从 `so->keyData[]` 中选择最多每列一个 boundary key。
它要求这些 key 是 preprocessing 标记出来的 required key。
向前扫描通常使用 `=`、`>`、`>=` 作为 start boundary。
向后扫描通常使用 `=`、`<`、`<=` 作为 start boundary。
它只能沿着 index attribute 的前缀使用这些 key。
前一列如果不是 equality-like 边界，后一列就不能用于初始定位。
否则会跳过可能匹配的 keyspace。
这个规则和普通复合索引使用规则类似，但这里是源码级状态。
不是 planner 层的启发式。
`_bt_first()` 还会处理 NOT NULL 推导。
如果某个 key 不能作为普通 boundary，但能证明目标区间不含 NULL，它可以构造 `SK_SEARCHNOTNULL` boundary。
相关逻辑在 `nbtsearch.c:1004-1011` 和 `1110-1131`。
这是为了避免无谓扫描大量 NULL，不改变语义。
当没有任何可用 boundary key 时，`_bt_first()` 调用 `_bt_endpoint()`。
`nbtsearch.c:1237-1245` 对应这个路径。
向前扫描从最左 leaf 开始。
向后扫描从最右 leaf 开始。
`_bt_endpoint()` 是 `_bt_search()` 的简化版本。
它用 `_bt_get_endpoint()` 沿边缘下降。
如果找到了 boundary keys，`_bt_first()` 会制造 `BTScanInsertData inskey`。
这一步非常重要。
`scan->keyData[]` 和 `so->keyData[]` 是 search-type scankey。
`_bt_search()` 需要的是 insertion-type scankey。
因为 tree descent 使用三路比较函数。
`nbtsearch.c:1382-1432` 把普通 comparison key 转换为 insertion scankey。
如果不是 cross-type comparison，就使用 opclass 的 `BTORDER_PROC`。
如果是 cross-type comparison，就从 opfamily 查找对应三路比较支持函数。
缺失 support function 时会 ERROR。
这不是 optional 优化。
没有它就无法在 index tuple 和 scankey argument 之间做正确排序比较。
`_bt_first()` 还会设置 `nextkey` 和 `backward`。
`nbtsearch.c:1435-1506` 是策略映射。
`<`：
向后找最后一个 `< key` 的 item。
`<=`：
向后找最后一个 `<= key` 的 item。
`=`：
forward scan 等价于 `>=`。
backward scan 等价于 `<=`。
`>=`：
向前找第一个 `>= key` 的 item。
`>`：
向前找第一个 `> key` 的 item。
这正是 `nextkey` 的含义。
`nextkey=false` 找 `>=`。
`nextkey=true` 找 `>`。
`backward` 决定 `_bt_binsrch()` 在 leaf 上返回第一个候选还是最后一个候选。
`_bt_first()` 在 `nbtsearch.c:1512-1513` 调用：

```text
_bt_search(rel, NULL, &inskey, &so->currPos.buf, BT_READ, false)
```

注意第二个参数 `heaprel` 是 NULL。
读路径不会创建 root，也不会 finish split。
它只需要找到 leaf。
如果返回 InvalidBuffer，说明 index 为空。
Serializable 隔离级别下有一个特殊边界。
`nbtsearch.c:1519-1532` 会先对 relation 加 predicate lock，然后重试 `_bt_search()`。
这是为了关闭 `_bt_search()` 和 `PredicateLockRelation()` 之间的窄窗口。
这个锁不是 buffer lock。
它服务 SSI predicate locking。
它说明 B-tree search 的 correctness 不只来自 page latch。
MVCC、predicate lock、buffer lock、rightlink/high key 分别解决不同层的问题。
找到 leaf 后，`_bt_first()` 用 `_bt_binsrch()` 定位 page 内 offset。
`nbtsearch.c:1541-1549` 解释：
当 `nextkey=false/backward=false` 时，offset 是第一个 `>= inskey` 的 non-pivot tuple。
其它组合由 `_bt_binsrch()` 的 `nextkey/backward` 规则处理。
然后 `_bt_readfirstpage()` 和 `_bt_readpage()` 会把匹配 item 复制到 `so->currPos.items[]`。
从这个点开始，课程可以转入 leaf page scan。
但本节的主问题已经完成了一半：
预处理和 `_bt_first()` 为 `_bt_search()` 提供了一个能被 high key/rightlink 正确消费的定位 key。

## 7. `_bt_search()`：候选 downlink 与逐层修正

`_bt_search()` 在 `src/backend/access/nbtree/nbtsearch.c:99-208`。
它的函数注释 `76-98` 非常直接。
它搜索某个 scankey。
更准确地说，是找第一个可能包含该 key 的 leaf page。
传入的 scankey 是 insertion-type scankey。
它可以省略右侧一些 index columns。
返回时：
`*bufP` 是 leaf page buffer。
它 locked and pinned。
parent page locks 不保留。
如果 `returnstack=true`，它返回一条 `BTStack`。
这个 stack 记录下降时每层选择的 parent page block 和 offset。
插入时如果 leaf split，需要回 parent 插入新 downlink。
普通读 scan 不需要这个 stack。
所以 `_bt_first()` 传 `returnstack=false`。
写路径 `_bt_search_insert()` 传 `returnstack=true`。
`_bt_search()` 第一步是 `_bt_getroot()`。
`nbtsearch.c:110-115` 调用 `_bt_getroot(rel, heaprel, access)`。
`_bt_getroot()` 在 `nbtpage.c:320-345` 的注释说明：
root 位置必须从 metapage 读。
返回的可能不是 true root，而是 fast root。
如果 root split 发生在“飞行途中”，可能返回 old root。
这没问题。
因为 old root 会变成某一层的 leftmost page。
搜索可以继续沿 rightlink 修正。
`_bt_getroot()` 通常还会使用 `rel->rd_amcache` 缓存 metapage 数据。
`nbtpage.c:360-404` 是 fast path。
缓存可能 stale。
它会检查 root page 是否 live、level 是否匹配、是否 alone on level。
不满足就丢弃缓存。
这里的语义是：
metapage cache 是性能优化。
high key/rightlink 才是并发修正机制。
进入循环后，`_bt_search()` 每层先调用 `_bt_moveright()`。
对应 `nbtsearch.c:128-141`。
注释说：
刚抓到的 page 可能在我们读 parent downlink 之后 split。
如果 split 了，就可能需要向右移动到新的 sibling。
写模式下，还允许 `_bt_moveright()` finish incomplete splits。
这就是 `_bt_moveright()` 放在循环开头的原因。
它不是 leaf-only 修正。
internal page 也可能 split。
如果当前 page 是 leaf，`nbtsearch.c:143-147` 直接结束。
否则，用 `_bt_binsrch()` 在 internal page 上找到 pivot tuple。
`nbtsearch.c:149-157`：
`_bt_binsrch()` 返回一个 offset。
该 pivot tuple 的 downlink 指向下一层 child。
`BTreeTupleGetDownLink(itup)` 取出 child block。
如果需要 stack，`nbtsearch.c:159-172` 记录当前 parent block 和 offset。
这个 stack 是 private memory。
它不是共享状态。
它可能过期。
`src/include/access/nbtree.h:734-741` 明确说，private stack 可能因为 concurrent page splits 和 page deletions 变旧，但不应变成不可挽救的坏图。
随后是锁边界。
`nbtsearch.c:182-183` 调用 `_bt_relandgetbuf(rel, *bufP, child, page_access)`。
这会释放当前 page lock 和 pin，再读取并锁住 child。
`_bt_relandgetbuf()` 在 `nbtpage.c:995-1035`。
如果目标 block 与当前 buffer 相同，它只是换锁模式。
否则释放旧 buffer，再 `ReadBuffer` 新 block，并 `_bt_lockbuf()`。
这不是 parent-child lock coupling。
这里故意不同时持有 parent 和 child。
所以需要下一轮 `_bt_moveright()` 重新验证 child。
如果 `access == BT_WRITE`，`_bt_search()` 会在距离 leaf 一层时把下一页锁模式改成 write。
`nbtsearch.c:174-180`：
如果当前 internal page level 是 1，而调用者要求写 leaf，那么 child 必然是 leaf。
于是 `page_access = BT_WRITE`。
这个边界避免先读锁 leaf 再升级。
但根页同时是 leaf 的小索引例外。
`nbtsearch.c:188-205` 处理这个例外。
如果 root leaf 先以 read lock 拿到，但调用者需要 write，它会释放 read lock，再获取 write lock。
释放和重拿之间 root leaf 可能 split。
所以它再次调用 `_bt_moveright()`。
这里是本节的一个重要模式：
只要中间释放过 page lock，就必须准备重新验证 high key。
`_bt_search()` 本身不保证 parent chain 永远新鲜。
它只保证最终返回的 leaf 已经按传入 insertion scankey 右追赶过。

## 8. `_bt_moveright()`：high key 判断与 rightlink 追赶

`_bt_moveright()` 在 `nbtsearch.c:241-322`。
注释 `211-239` 是本节核心。
当搜索跟随某个 pointer 到达 page 后，该 page 可能已经改变。
如果发生 split，原 page 上的数据要么仍在原 page，要么严格在它右边。
所以搜索只需要判断是否该右移。
判断依据是当前 page 的 high key。
如果 high key 严格小于 scankey，说明目标 key 已经不属于当前 page。
如果 `key.nextkey=true`，则 high key 等于 scankey 也要右移。
因为 `nextkey=true` 要找的是第一个 `> key`。
`nbtsearch.c:257-268` 解释了这个比较。
代码把比较阈值写成：

```text
cmpval = key->nextkey ? 0 : 1
```

随后在循环里执行：

```text
P_IGNORE(opaque) || _bt_compare(rel, key, page, P_HIKEY) >= cmpval
```

如果为真，就通过 `opaque->btpo_next` 右移。
这段在 `nbtsearch.c:307-311`。
解释要非常小心。
`_bt_compare()` 返回的是 `scankey` 相对 tuple 的比较。
`>= 1` 表示 scankey 大于 high key。
`>= 0` 表示 scankey 大于等于 high key。
所以 `nextkey=false` 时只在 `scankey > highkey` 时右移。
`nextkey=true` 时在 `scankey >= highkey` 时右移。
如果 page 是 rightmost，没有 high key。
`nbtsearch.c:280-281` 直接停止。
如果 page 是 ignored page，例如 deleted 或 half-dead，就右移。
如果一直走到末端仍是 ignored page，`nbtsearch.c:317-319` 报错。
正常结构中不应从 live keyspace 掉出 index。
右移不是一次性动作。
`nbtsearch.c:267-268` 明确说 page 可能 split 不止一次。
所以 `_bt_moveright()` 是循环。
这正是 rightlink 追赶 split 的 runtime 现象。
想象一个页面 A。
搜索从 parent 读到 downlink A。
搜索释放 parent 后，另一个 backend 把 A split 成 A 和 B。
它又因为插入压力把 A 或 B 再 split。
当搜索拿到 A 时，A 的 high key 可能已经小于 search key。
搜索走到 A 的 `btpo_next`。
如果还不够，再继续走。
它不需要重读 root。
它不需要找到最新 parent downlink。
它只相信同一层的 high key/rightlink 链。
`_bt_moveright()` 在写模式下还有额外职责。
`nbtsearch.c:232-235` 说，如果 `forupdate=true`，会尝试 finish incomplete splits。
插入不允许在 incomplete split 的 page 上继续插入。
`nbtsearch.c:283-304` 实现这个规则。
如果当前 page 不是 rightmost，且 `P_INCOMPLETE_SPLIT(opaque)`，并且 forupdate：
先根据需要把 read lock 升级到 write lock。
再次检查 flag。
如果仍然 incomplete，就调用 `_bt_finish_split(rel, heaprel, buf, stack)`。
然后重新获取原 block 的锁并重检。
为什么 read path 不做这件事？
README `652-655` 说，缺失 downlink 也可以让 search 通过 rightlink 找到页面。
读路径修复 split 会把原本 read-only 操作变成写操作。
在 hot standby 上也不可能这么做。
所以修复放在 insertion path。
这就是 read/write path 的 correctness boundary：
读者只需要 tree 对搜索一致。
写者需要 parent downlink 一致到足以安全继续 split 或插入。
`_bt_moveright()` 的参数 `stack` 也只在 forupdate 时有意义。
因为 `_bt_finish_split()` 需要找到 parent 插入 missing downlink。
普通 `_bt_first()` 调用 `_bt_search(... BT_READ, false)`。
它没有 stack。
也不会修复 incomplete split。
这里可以形成一个小不变量：

```text
缺失 parent downlink 不影响 reader 找到右页；
缺失 parent downlink 会影响 writer 后续 split；
所以 reader 追 rightlink，writer 先补 downlink。
```

## 9. `_bt_binsrch()` 与 `_bt_compare()`：页内定位不是并发修正

`_bt_moveright()` 只决定当前 page 是否属于正确 key range。
它不负责页内 offset。
页内 offset 由 `_bt_binsrch()` 处理。
`_bt_binsrch()` 在 `nbtsearch.c:343-450`。
internal page 上，它返回最后一个 `< scankey` 的 pivot tuple。
如果 `nextkey=true`，返回最后一个 `<= scankey` 的 pivot tuple。
这个 pivot tuple 的 downlink 就是下一层 child。
leaf page 上，它返回初始定位结果。
forward scan 通常返回第一个 `>= key` 或 `> key` 的 non-pivot tuple。
backward scan 则返回最后一个 `< key` 或 `<= key` 的 tuple。
`_bt_binsrch()` 注释 `339-341` 特别强调：
它不负责 walking right。
它没有 buffer lock/refcount side effects。
这很重要。
如果把 `_bt_binsrch()` 当成并发修正点，就会误读源码。
并发 split 修正边界在 `_bt_moveright()`。
页内 binary search 只在已经 locked/pinned 的 page 上计算 offset。
`_bt_compare()` 在 `nbtsearch.c:688-860`。
它把 insertion-type scankey 和 page 上某个 tuple 比较。
它处理 NULL 排序。
它处理 truncated pivot tuple。
它处理 heap TID tie-breaker。
`nbtsearch.c:679-685` 有一个关键注释：
在 non-leaf page 上，第一个 data key 被当作 minus infinity。
这实现了 Lehman-Yao “第一个 downlink 在第一个 separator 之前”的约定。
version 3+ index 会显式把它截断为 0 attributes。
但 `_bt_compare()` 不依赖物理值。
它直接在 non-leaf first data item 上返回 `scankey > tuple`。
这解释了为什么 internal page 的第一个 item 看起来怪。
它不是普通 separator。
它是 negative infinity downlink。
`_bt_compare()` 还会处理 heapkeyspace。
`BTScanInsertData.scantid` 在 heapkeyspace index 中是最后 tie-breaker。
`src/include/access/nbtree.h:780-786` 明确说，search code 把 `scantid` 当作另一个 insertion scankey attribute。
这对 duplicate key 很关键。
没有 heap TID tie-breaker，logical duplicates 无法给 Lehman-Yao 的严格 key range 提供稳定边界。
README `42-49` 也说明：
所有 B-tree keys 要唯一，PostgreSQL 通过把 heap TID 当作 tie-breaker 满足这个要求。
logical duplicates 按 heap TID 排序。
这里的 takeaway 是：
`_bt_search()` 不是用 SQL operator 直接比较 tuple。
它使用 insertion scankey 的三路比较语义。
preprocessing 决定可定位边界。
`_bt_first()` 转换为 insertion scankey。
`_bt_compare()` 在 high key、pivot tuple、leaf tuple 上使用同一套排序语义。

## 10. split 写路径：为什么 rightlink 追赶一定可用

只读 `_bt_moveright()` 会知道“怎么追”。
但要相信它，还要看 split 如何制造可追的状态。
split 主体在 `src/backend/access/nbtree/nbtinsert.c`。
本节只对照关键点。
`_bt_insertonpg()` 注释 `1089-1115` 列出它的工作：
必要时 split target page。
插入 new tuple。
如果 page split，就弹出 parent stack，找 parent 上插入新 child pointer 的位置。
然后递归向上插入 downlink。
`_bt_split()` 注释 `1462-1487` 说明：
进入时原 page pinned and write-locked。
返回新右 sibling，且左页和右页都保持 pinned and write-locked。
`_bt_split()` 先选择 split point。
`nbtinsert.c:1543-1568` 说明 split point 可以理解为在两个 item 之间。
然后构造临时 leftpage 和 rightpage。
`nbtinsert.c:1575-1583` 很关键。
leftpage 继承原 page flags，但清掉 root、split_end、garbage。
然后设置 `BTP_INCOMPLETE_SPLIT`。
这意味着 split 第一阶段完成后，左页会带着 incomplete 标志。
此时右页即将存在，但 parent 还没有它的 downlink。
`_bt_split()` 随后为左页生成新的 high key。
leaf split 会调用 `_bt_truncate()`。
`nbtinsert.c:1623-1683` 解释：
leaf 左页 high key 是 `firstright` 的可能截断副本。
它必须大于等于 left side 的最后 item。
它必须严格小于 right side 的第一个 item。
这给 `_bt_moveright()` 提供了可判断的边界。
如果 search key 已经越过左页 high key，它就必须右移。
internal split 不做 leaf 那种 suffix truncation。
`nbtinsert.c:1685-1713` 解释：
internal page split 必须保留 separator key 的一致链条。
这里不要展开“为什么分隔边界重要”。
只要记住：
internal high key 也要能让搜索在上层和下层一致地判断边界。
获取新右页后，`nbtinsert.c:1754-1774` 填充 sibling link。
左页 `btpo_next = rightpagenumber`。
右页 `btpo_prev = origpagenumber`。
右页 `btpo_next = old right sibling`。
二者 `btpo_cycleid` 相同。
如果原页不是 rightmost，还会把旧右 sibling 的 leftlink 改成新右页。
`nbtinsert.c:1907-1915` 先锁住旧右 sibling。
`nbtinsert.c:1977-1980` 在 critical section 中更新旧右 sibling 的 `btpo_prev`。
这里体现了写路径的 lock coupling。
split 持有 left page write lock。
它获得 new right page write lock。
如果有 old right sibling，它按 left-to-right 顺序锁旧右 sibling。
README `76-80` 也说明了这个 extra step：
为了支持 backward scan，split 时要锁 former right sibling 并更新 left-link。
锁顺序 left-to-right 避免 deadlock。
最关键的 atomic boundary 在 `nbtinsert.c:1944-2095`。
进入 critical section 后，原 page 被覆盖成 leftpage。
right buffer 被填充成 rightpage。
必要时更新 old right sibling leftlink。
写 WAL split record。
设置 LSN。
此时即使 parent downlink 还没有插入，读者也能通过 left page rightlink 到达 right page。
`_bt_insertonpg()` 在 split 后调用 `_bt_insert_parent()`。
`nbtinsert.c:1238-1263` 的注释列出 split 后状态：
target page 已经 split。
原 tuple 已经插入。
old left 和 new right 都持有 write locks。
parent insertion 接下来发生。
`_bt_insert_parent()` 注释 `2114-2128` 是 split 两阶段的核心。
进入时 left/right split pages 仍然 write locked。
它会在找到 parent 并加 write lock 后释放 right page lock。
left page lock 会和 parent page lock 一起释放。
因为 left page 的 `INCOMPLETE_SPLIT` 必须和 parent 中插入 downlink 的动作在同一个 atomic operation 中完成。
`_bt_insert_parent()` 取 left page 的 high key，复制成指向 right page 的 downlink。
`nbtinsert.c:2208-2215` 对应这一步。
然后 `_bt_getstackbuf()` 重新找到 parent 上指向 left child 的 pivot tuple。
`nbtinsert.c:2216-2225` 说，downlink 位置可能已经因并发 split 改变。
`_bt_getstackbuf()` 会检测并恢复，更新 stack。
之后 `_bt_insertonpg()` 递归插入 parent。
当向 internal page 插入 downlink 完成 split 时，会清 child 的 `BTP_INCOMPLETE_SPLIT`。
`nbtinsert.c:1318-1329` 是非 split parent insertion 清 flag 的路径。
如果 parent 也需要 split，则 `_bt_split()` 在 `nbtinsert.c:1983-1993` 清 child flag。
这说明：
child incomplete flag 的清除和 parent downlink 插入或 parent split WAL 在同一个 critical section。
如果 crash 或 ERROR 发生在 split 第一阶段之后、parent insertion 之前，left page 可能保留 `BTP_INCOMPLETE_SPLIT`。
`_bt_finish_split()` 就是恢复路径。
`nbtinsert.c:2259-2317` 说明：
insertion routines 不允许插入到 incompletely split page。
进入 `_bt_finish_split()` 时左页必须 write locked。
它锁右 sibling。
必要时读 metapage 判断是否 root split。
然后调用 `_bt_insert_parent()`。
这就是 `_bt_moveright(forupdate=true)` 遇到 incomplete split 时调用 `_bt_finish_split()` 的原因。

## 11. latch/lock coupling 边界

PostgreSQL 代码里主要说 buffer lock，而不是直接说 latch。
很多论文或系统实现会把 page-level mutual exclusion 叫 latch。
在本节，最好把它具体化为：
buffer pin 保护 buffer frame 不被驱逐或回收。
buffer content lock 保护 page 内容读写的一致性。
heavyweight lock 或 predicate lock 解决逻辑冲突。
这些不是同一个东西。
`nbtpage.c:832-848` 对 `_bt_getbuf()` 的注释给出 nbtree 的基本规则：
访问 nbtree page 时必须同时持有 buffer pin 和 buffer lock。
返回时 buffer locked and pinned。
raw `LockBuffer()` 调用在 nbtree 中不鼓励，应该走 `_bt_lockbuf()` 等 wrapper。
`_bt_lockbuf()` 在 `nbtpage.c:1057-1092`。
它要求 caller 已经持有 pin。
`_bt_unlockbuf()` 在 `nbtpage.c:1094-1111`。
它释放 lock，但保留 pin。
`_bt_relbuf()` 在 `nbtpage.c:1037-1055`。
它同时释放 lock 和 pin。
`_bt_relandgetbuf()` 在 `nbtpage.c:995-1035`。
它是下降和右移常用边界。
如果切换到不同 block，它先释放旧 buffer，再读新 buffer 并加锁。
这就是 read search 的普通边界：
不持有 parent-child lock chain。
读取 parent。
选 downlink。
释放 parent。
锁 child。
在 child 上用 high key 验证。
如果 child 被 split 影响，就右追。
这个模式减少锁持有时间。
它把 correctness 责任交给 high key/rightlink。
什么时候需要 lock coupling？
第一，split 更新 sibling 链时。
`nbtinsert.c:1907-1915` 在非 rightmost split 时锁 old right sibling。
因为要更新它的 leftlink。
这是同一层 left-to-right coupling。
README `106-110` 说，大多数情况下移动到下一页前先释放当前页。
少数地方必须先锁下一页再释放当前页。
向右或向上是安全的。
向左或向下可能制造 deadlock。
第二，parent insertion 完成 split 时。
`_bt_insert_parent()` 必须让 left child 的 `BTP_INCOMPLETE_SPLIT` 清除和 parent downlink 插入成为同一 atomic action。
所以它持有 left child 和 parent。
right child 在 parent lock 获得后释放。
这是 parent-child coupling，但只在写路径中短暂发生。
第三，backward scan 处理 leftlink 时。
向后扫描不能只信 leftlink。
README `81-86` 说，跟随 leftlink 后必须考虑 left sibling 在读取前 split。
所以要向右移动，直到找到 rightlink 指回原 page 的页面。
`_bt_lock_and_validate_left()` 在 `nbtsearch.c:1955-2080` 实现这个逻辑。
它最多沿右走若干步验证。
如果找不到，回到原 page 看它是否被删除或 leftlink 是否改变。
这不是本节主链路，但它说明 leftlink 比 rightlink 难用。
rightlink 是 Lehman-Yao 搜索恢复主轴。
leftlink 是 PostgreSQL 为 backward scan 增加的复杂性。
第四，root/metapage 边界。
`_bt_getroot()` 可以从 cached metapage 拿 fast root。
它不会在拿 root 时一直持有 metapage lock。
`_bt_gettrueroot()` 注释 `576-582` 明确说，持有 metapage lock 再移动到 root 可能和 concurrent root split deadlock。
所以这里也避免跨层锁链。
总的锁边界可以压缩成：

```text
read descent: usually one page locked at a time
move right: release current, lock right sibling
split sibling update: lock left then right
finish split parent insert: hold child/parent only across atomic update
move left: validate by walking right, because leftlink may be stale
```

如果用 latch coupling 这个词，必须说清楚是哪一类。
本节里的“latch”不是 PostgreSQL `Latch` 事件等待对象。
它是 page-level buffer content lock 的并发控制语义。

## 12. 生命周期、ownership 与 cleanup

search scan 的长期状态在 backend-local memory。
`BTScanOpaqueData` 由 btree scan 初始化并挂在 `IndexScanDesc->opaque`。
`_bt_preprocess_keys()` 把输出 key 放到 `so->keyData[]`。
array 相关数据放到 `so->arrayContext`。
`nbtpreprocesskeys.c:1880-1889` 说明：
如果还没有 array context，就创建 `"BTree array context"`。
如果已经有，就 reset。
这使 array preprocessing 的中间数据跟 scan/rescan 生命周期绑定。
`_bt_preprocess_keys()` 多次调用是 no-op。
`nbtpreprocesskeys.c:219-226` 说，如果 `so->numberOfKeys > 0`，直接返回。
所以 rescan 必须在更上层重置对应状态。
否则不会重复 preprocessing。
`BTStack` 是 `_bt_search()` 下降时按需 palloc 的 private stack。
`nbtsearch.c:165-172` 创建 stack entry。
`_bt_doinsert()` 在插入结束后 `_bt_freestack(stack)`。
`nbtinsert.c:273-276` 对应这个 cleanup。
这个 stack 不受 shared memory 管理。
它只是一个可能 stale 的导航线索。
如果 ERROR 发生，普通 palloc memory 会由 MemoryContext 清理。
buffer pin 和 lock 则由 ResourceOwner/abort cleanup 兜底释放。
但 shared page 修改不是 MemoryContext 能回滚的东西。
一旦 split critical section 写了 WAL 并修改 page，后续 ERROR 不能把 page 还原。
所以 split 设计必须允许“已经完成第一阶段但未完成 parent insertion”的状态存在。
这就是 `BTP_INCOMPLETE_SPLIT` 的生命周期。
创建：
`_bt_split()` 在 leftpage 上设置 `BTP_INCOMPLETE_SPLIT`。
持有：
split 第一阶段完成后，left page 可能带着这个 flag。
清除：
`_bt_insert_parent()` 通过 `_bt_insertonpg()` 在 parent 插入 downlink 时清除。
恢复：
后续 writer 在 `_bt_moveright(forupdate=true)` 或 `_bt_getstackbuf()` 看到 flag，会调用 `_bt_finish_split()`。
ERROR/abort 兜底：
锁和 pin 会释放。
已经 WAL 记录的 page 状态保留。
后续 insertion 修复 incomplete split。
读者如何处理：
读者不关心 incomplete flag。
它只用 high key/rightlink 找数据。
README `705-724` 讨论 recovery 时也强调，reader 不需要关心 incomplete split flag。
这个 flag 主要是防止多个 writer 同时尝试补同一个 downlink。
如果允许多个 writer 观察同一个 incomplete split 并各自补 parent，会破坏 parent。
所以 primary 上 writer 通过持锁和 finish split 规则串行化。
这里也能看到 crash safety 边界。
split first phase 是一个 WAL record。
parent insertion 是第二个 WAL record。
系统可以 crash 在两者之间。
恢复后 tree 对 reader 仍一致。
但 writer 后续会补 parent downlink。
这是一种典型的“局部结构先可搜索，再延迟补全上层索引”的设计。

## 13. 正确性机制层次

第一层是 B-link tree 不变量。
非右端 page 有 high key。
rightlink 指向右 sibling。
split 后 old page 成为 left page。
right page 持有被移动的 key range。
旧 parent downlink 即使暂时只指向 left page，search 也能通过 left high key/rightlink 找到 right page。
第二层是 buffer pin/content lock。
读 page 内容前必须 locked and pinned。
`_bt_getbuf()` 保证这一点。
`_bt_search()` 不持有整条路径，但每个页面被检查时内容不会被 writer 同时改写。
第三层是 insertion scankey 的全序比较。
`_bt_compare()` 必须对 leaf tuple、pivot tuple、high key 使用一致排序。
heap TID tie-breaker 让 duplicates 也有稳定位置。
truncated pivot tuple 的 minus infinity 语义让 internal page downlink 规则成立。
第四层是 preprocessing 和 leaf scan 停止规则。
`SK_BT_REQFWD/SK_BT_REQBKWD` 把“此 key 不满足是否可以停止”编码到 `so->keyData[]`。
`_bt_first()` 和 `_bt_checkkeys()` 必须对这些规则对称。
否则会出现开始位置和结束条件不一致。
第五层是 WAL atomic action。
split first phase 让 rightlink/high key 可用。
parent insertion 清 incomplete flag。
crash 可以落在两个阶段之间。
恢复后 reader 仍能搜索。
writer 后续 finish split。
第六层是 predicate locking。
Serializable 空 index 情况下，`_bt_first()` 需要 relation-level predicate lock 并重试。
这是为了防止 `_bt_search()` 看到空和 predicate lock 建立之间插入者漏看冲突。
它不是 B-tree page latch 能解决的问题。
第七层是 page deletion/VACUUM 的旁路规则。
`P_IGNORE` 页面要跳过。
backward scan 要验证 left sibling。
VACUUM linear scan 要用 cycleid 处理 concurrent split。
这些不是 `_bt_search()` 主路径，但它们解释了为什么 page opaque 中有 deleted、half-dead、cycleid 等状态。
这些机制不能互相替代。
MVCC snapshot 决定 heap tuple 对 SQL 是否可见。
它不决定 index page 是否能并发读写。
buffer lock 决定 page 内容是否能安全读取。
它不决定一个 TID 对 snapshot 是否可见。
rightlink/high key 决定 search 如何修正 stale downlink。
它不保证 parent downlink 已经补全。
WAL 保证 crash 后结构能恢复到某个一致状态。
它不减少前台 search 的比较次数。
predicate lock 保证 SSI 逻辑冲突。
它不保护 page 内存安全。

## 14. 错误路径、异常路径与 fallback

第一类是 scan qual 矛盾。
`_bt_preprocess_keys()` 发现 `x = 1 AND x > 2` 这类条件时设置 `so->qual_ok=false`。
`_bt_first()` 直接返回 false，边界在 `nbtpreprocesskeys.c:170-175` 和 `nbtsearch.c:904-913`。
第二类是跨类型 opfamily 信息不足。
预处理可能无法证明冗余或矛盾，于是保留更多 key。
`_bt_unmark_keys()` 确保 required key 不会变成多重、不一致的起止边界。
性能可能下降，正确性不受影响。
第三类是 index 为空。
`_bt_getroot()` 在 read mode 不创建 root，`_bt_search()` 返回 InvalidBuffer。
Serializable 下 `_bt_first()` 先拿 relation predicate lock 再重试。
第四类是 metapage cache stale。
`_bt_getroot()` 使用 `rel->rd_amcache` 时会验证 fast root page。
如果 page 被删除、level 不符、不是 alone on level，就丢弃 cache 并重读 metapage。
缓存只服务性能，不能当正确性来源。
第五类是 downlink stale。
`_bt_search()` 跟随 parent downlink 后，child 可能已经 split。
`_bt_moveright()` 用 high key 检测越界，并沿 `btpo_next` 追赶。
这不是 ERROR，而是正常 slow path。
第六类是页面被 deletion 标记为 ignored。
`_bt_moveright()` 看到 `P_IGNORE` 会右移。
如果最终落在末尾仍然 ignored，就 ERROR，通常表示结构损坏。
第七类是 incomplete split。
reader 不修复。
writer 在 `_bt_moveright(forupdate=true)` 或 `_bt_getstackbuf()` 中修复。
`_bt_finish_split()` 锁右 sibling，判断是否 root split，然后调用 `_bt_insert_parent()`。
第八类是 parent stack stale。
`BTStack` 可能因并发 split 或 deletion 变旧。
`_bt_getstackbuf()` 从 stack 的 page/offset 附近找 child downlink，失败时沿 parent rightlink 继续。
关键源码在 `nbtinsert.c:2320-2348` 和 `2395-2454`。
第九类是 root split 或 concurrent root split。
true root split 走 `_bt_newlevel()`。
如果 stack 为空但 page 已经不是 root，说明发生 concurrent root split，需要从更高一层重新定位。
root 也不能绕开 rightlink/high key reasoning。
第十类是 backward scan 的 leftlink stale。
`_bt_lock_and_validate_left()` 不直接相信 leftlink。
它检查候选 left sibling 的 `btpo_next` 是否指回原 page，不符合时向右修正。
本节主线是 rightlink 追赶，但调试 backward scan 时必须记住这个边界。

## 15. 成本、资源与跨模块传播

成本一是 tree height。
普通 `_bt_search()` 每层一次 page 访问和一次 binary search。
`_bt_getroot()` cache miss 时还要访问 metapage。
成本二是 rightlink chase。
并发 split 或 page deletion 会让 search 多走 sibling。
这个成本不直接出现在 `EXPLAIN` 的某个字段里，只能从 buffer hits、gdb、perf 或自加计数器推断。
成本三是 scankey preprocessing。
复杂 array keys、skip arrays、cross-type operators 会带来排序、去重、opfamily lookup 和 arrayContext 分配。
成本四是 page-level contention。
读者持 share buffer lock，插入和 split 持 exclusive buffer lock。
rightmost leaf page 的持续插入会形成热点。
`_bt_search_insert()` 的 rightmost leaf cache fastpath 使用 conditional lock，拿不到锁就放弃优化。
成本五是 WAL 和 critical section。
split first phase 与 parent insertion 是不同 WAL action，parent insertion 还可能递归 split。
成本六是 pageinspect 和诊断开销。
`bt_page_stats()` 和 `bt_page_items()` 适合实验和离线诊断，不适合生产高频循环。
跨模块传播：
buffer manager 提供 pin、content lock、read/extend buffer。
relation cache 提供 metapage cache `rd_amcache`。
WAL 提供 split、insert、newroot、reuse page 等 crash recovery。
SSI predicate locking 处理 serializable phantom 边界。
heap/table AM 处理最终 TID 可见性和 unique check 中的 heap 访问。
VACUUM/page deletion 维护 half-dead/deleted page，并依赖 rightlink 可恢复性。
pageinspect 把 page opaque 和 high key 暴露给 SQL 观察。

## 16. 观测与诊断入口

能直接观察的状态包括 `bt_metap()` 的 root/level/fastroot，`bt_page_stats()` 的 sibling link、level、flags，以及 `bt_page_items()` 的 page item。
非右端 page 的 itemoffset 1 通常是 high key。
`EXPLAIN (ANALYZE, BUFFERS)`、`pg_stat_all_indexes.idx_scan` 和 `pg_stat_io` 可以辅助观察访问成本。
只能间接推断的状态包括 `_bt_moveright()` 追了几个 rightlink、`_bt_first()` 使用了哪些 startKeys、`_bt_preprocess_keys()` 是否生成 skip array、`_bt_getroot()` 是否命中 `rd_amcache`。
这些通常需要 gdb、perf、tracepoint、临时日志或源码计数器。
短暂 buffer lock coupling、critical section 内的 split 两阶段、某个 writer 是否刚调用 `_bt_finish_split()`，几乎不能从 SQL 直接看到。
实验时可以使用这些断点：

```text
break _bt_preprocess_keys
break _bt_first
break _bt_search
break _bt_moveright
break _bt_split
break _bt_insert_parent
break _bt_finish_split
break _bt_getstackbuf
```

如果 PostgreSQL 构建启用了 injection points，可以关注：

```text
nbtree-leave-leaf-split-incomplete
nbtree-leave-internal-split-incomplete
nbtree-finish-incomplete-split
```

这些注入点正好位于 split 第一阶段和 finish split 路径。
没有 injection point 时，gdb breakpoint 也足够完成课堂实验。
诊断时不要把所有 index scan 变慢都归因于 `_bt_moveright()`。
更常见的原因包括 selectivity 错估、heap fetch 太多、visibility map 不足、cache miss、rightmost page insert contention、unique check 等待 heap 事务，以及 VACUUM/page deletion 干扰。
如果你怀疑 concurrent split 影响 search，先用 pageinspect 看页面 sibling/high key 结构，再用 gdb/perf 观察 `_bt_moveright()` 是否频繁命中右移路径。

## 17. 课堂实验一：用 pageinspect 观察 high key 和 rightlink

目标：
看到 B-tree leaf page 的 high key、rightlink 和 level。
准备：
需要安装 `pageinspect` extension。
SQL：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS btree_move_demo;
CREATE TABLE btree_move_demo(id bigint, payload text);
CREATE INDEX btree_move_demo_id_idx ON btree_move_demo(id);
INSERT INTO btree_move_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 20000) AS g;
ANALYZE btree_move_demo;

SELECT * FROM bt_metap('btree_move_demo_id_idx');

SELECT blkno, type, live_items, free_size, btpo_prev, btpo_next, btpo_level, btpo_flags
FROM bt_page_stats('btree_move_demo_id_idx', 1);

SELECT itemoffset, ctid, itemlen, data, dead, htid
FROM bt_page_items('btree_move_demo_id_idx', 1)
ORDER BY itemoffset
LIMIT 5;
```

观察点：
如果 block 1 是非右端 leaf，它的 `btpo_next` 不是 0。
`btpo_level=0` 表示 leaf。
`bt_page_items()` 的 `itemoffset=1` 在非右端 leaf 上是 high key。
它不是普通 heap TID 返回项。
换几个 block 调用 `bt_page_stats()`。
沿 `btpo_next` 可以看到 leaf sibling 链。
解释回源码：
`BTPageOpaqueData.btpo_next` 对应 `src/include/access/nbtree.h:65-66`。
`P_HIKEY` 和 `P_FIRSTDATAKEY()` 对应 `src/include/access/nbtree.h:368-370`。
`_bt_moveright()` 正是用 item 1 的 high key 判断是否沿 `btpo_next` 继续。
注意：
`pageinspect` 看到的是某一刻 page 内容。
它不能告诉你某个查询是否刚刚调用了 `_bt_moveright()`。
这个实验只是把源码字段和物理页面对应起来。

## 18. 课堂实验二：观察 `_bt_first()` 选择起点与 scankey preprocessing

目标：
理解 search-type scankey 如何变成定位用 insertion scankey。
准备：
用 debug build 或可附加 gdb 的本地 PostgreSQL。
建立复合索引：

```sql
DROP TABLE IF EXISTS btree_scankey_demo;
CREATE TABLE btree_scankey_demo(a int, b int, c int, payload text);
INSERT INTO btree_scankey_demo
SELECT g % 100, g % 50, g % 20, repeat('p', 20)
FROM generate_series(1, 200000) AS g;
CREATE INDEX btree_scankey_demo_abc_idx ON btree_scankey_demo(a, b, c);
ANALYZE btree_scankey_demo;
```

查询一：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM btree_scankey_demo
WHERE a = 10 AND b < 4 AND c < 5;
```

查询二：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM btree_scankey_demo
WHERE b = 4;
```

gdb 断点：

```text
break _bt_preprocess_keys
break _bt_first
break _bt_search
```

观察点：
在 `_bt_preprocess_keys()` 后查看 `((BTScanOpaque) scan->opaque)->keyData`。
关注 `sk_attno`、`sk_strategy`、`sk_flags`。
`a = 10` 应成为 leading equality boundary。
`b < 4` 在 forward scan 中可以成为停止边界。
`c < 5` 通常不能单独结束 scan，除非 skip array 改写让前缀连续。
在 `_bt_first()` 中查看 `keysz`、`strat_total`、`inskey.nextkey`、`inskey.backward`。
解释回源码：
required flags 由 `nbtpreprocesskeys.c:113-128` 的规则决定。
`_bt_first()` 的 start key 选择规则在 `nbtsearch.c:956-1030`。
strategy 到 `nextkey/backward` 的映射在 `nbtsearch.c:1435-1506`。
讨论：
比较查询一和查询二。
查询二如果启用 skip scan，预处理可能生成 skip array。
如果没有 skip support 或成本不合适，表现可能接近宽扫描。
这个实验的重点不是性能数字。
重点是看 `so->keyData[]` 如何从 SQL predicate 变成 scan 状态机。

## 19. 课堂实验三：让 split 和 `_bt_moveright()` 出现在调试现场

目标：
观察并发插入触发 split 时，搜索侧如何进入 `_bt_moveright()`。
准备：
使用 debug build。
最好关闭过多优化。
让两个 psql session 并发操作同一索引。
建表：

```sql
DROP TABLE IF EXISTS btree_split_demo;
CREATE TABLE btree_split_demo(id bigint, payload text);
CREATE INDEX btree_split_demo_id_idx ON btree_split_demo(id) WITH (fillfactor = 50);
INSERT INTO btree_split_demo
SELECT g, repeat('x', 400)
FROM generate_series(1, 50000) AS g;
ANALYZE btree_split_demo;
```

Session A 持续插入：

```sql
INSERT INTO btree_split_demo
SELECT g, repeat('y', 400)
FROM generate_series(50001, 200000) AS g;
```

Session B 持续查询：

```sql
SELECT count(*)
FROM btree_split_demo
WHERE id BETWEEN 75000 AND 76000;
```

gdb 断点：

```text
break _bt_split
break _bt_moveright
commands
  bt 5
  continue
end
```

如果输出太多，可以在 `_bt_moveright()` 内部 `P_IGNORE` 或 high key compare 分支附近加条件断点。
源码级观察点：
在 `_bt_split()` 看 `origpagenumber`、`rightpagenumber`。
在 `_bt_moveright()` 看 `BufferGetBlockNumber(buf)` 和 `opaque->btpo_next`。
在 `_bt_compare(rel, key, page, P_HIKEY)` 返回值大于等于阈值时，搜索会沿 rightlink。
如果启用了 injection points，可以在 `nbtree-leave-leaf-split-incomplete` 停住 writer。
然后让另一个 writer 触发 `_bt_finish_split()`。
观察 `nbtree-finish-incomplete-split`。
解释回源码：
split 在 `_bt_split()` 中设置 left page high key 和 `btpo_next`。
parent insertion 可能还没完成。
reader 不需要 parent downlink。
writer 遇到 incomplete split 会补 parent downlink。
注意：
这个实验有 timing 依赖。
如果没有命中 `_bt_moveright()` 右移分支，不说明机制不存在。
它只说明当次查询没有踩到 stale downlink 窗口。
可以增大并发、降低 fillfactor、使用大 payload、或用 injection point 扩大窗口。

## 20. 常见误区

误区一：
把 parent downlink 当作永远准确的 child key range。
在 nbtree 中，downlink 是候选入口。
child high key/rightlink 才负责并发 split 后的 runtime 修正。
误区二：
把 high key 当作普通索引 tuple。
high key 是 pivot tuple。
它不指向 heap tuple。
非右端 leaf 的 item 1 不是扫描返回项。
误区三：
认为 `_bt_moveright()` 只服务 leaf scan。
它在 `_bt_search()` 每层下降后执行。
internal page split 同样可能让刚跟随的 downlink 变旧。
误区四：
认为 read path 应该顺手修复 incomplete split。
读者不需要 missing downlink。
在 hot standby 上也不能执行写修复。
写者才需要 `_bt_finish_split()`。
误区五：
把 `SK_BT_REQFWD` 理解成“必须满足此 qual 才能返回 tuple”。
所有 qual 都要满足才能返回。
`SK_BT_REQFWD` 表示 forward scan 中不满足它时可以停止当前 primitive scan。
误区六：
把 buffer pin 当成 buffer lock。
pin 防止 buffer frame 被回收。
content lock 防止 page 内容并发改写。
index scan 返回 TID 给上层时通常释放 lock，某些情况下保留 pin。
误区七：
看到 `btpo_next` 就以为 forward scan 总读当前 rightlink。
README `88-101` 说 scan 会记住扫描页面当时的 rightlink。
如果之后当前页 split，直接读最新 rightlink 可能重复扫描被移动的 item。
误区八：
把 `EXPLAIN BUFFERS` 中的 buffer hit 数直接解释成 `_bt_moveright()` 次数。
rightlink chase 可能贡献 buffer hit。
但 buffer hit 还包括 root/internal/leaf 正常访问、heap fetch、visibility map 等。
需要源码断点或计数器才能确认。

## 21. 讨论题

1. 为什么 `_bt_search()` 不能只在 root 读到 downlink 后一路相信 parent 指针？
2. 如果 split 后先插 parent downlink，再更新 left page high key/rightlink，会破坏哪些读者假设？
3. `BTP_INCOMPLETE_SPLIT` 为什么标在左页，而不是标在缺 downlink 的右页？
4. `SK_BT_REQFWD` 和普通 SQL qual 的“必须满足”有什么区别？
5. `_bt_first()` 为什么要把 search-type scankey 转成 insertion-type scankey？
6. 为什么 read-only index scan 不修复 incomplete split？
7. 为什么 backward scan 不能像 forward scan 一样简单相信 sibling link？
8. `pageinspect` 能帮助你看到什么？它看不到 `_bt_moveright()` 的哪些 runtime 事实？

## 22. 本节小结

本节唯一主问题是：
nbtree 如何在短 page lock 搜索下，仍然对 concurrent split 保持正确定位。
核心答案是：
parent downlink 只是候选入口。
child page 的 high key/rightlink 是并发 split 后的修正事实。
`_bt_preprocess_keys()` 把 search qual 变成带 required flags 的 scan 状态机。
`_bt_first()` 从这个状态机选择起始边界，并制造临时 `BTScanInsertData`。
`_bt_search()` 从 fast root 或 root 下降。
它每层只短暂持有一个 page 的 pin/lock。
下降到 child 后用 `_bt_moveright()` 验证 high key。
如果 search key 已越过 high key，就沿 `btpo_next` 追赶。
`_bt_binsrch()` 只负责页内 offset。
它不负责并发修正。
split 写路径先让 left high key 和 rightlink 可用。
再通过 parent insertion 补 right page downlink。
两阶段之间可能存在 `BTP_INCOMPLETE_SPLIT`。
reader 仍能通过 rightlink 找到页面。
writer 必须先 `_bt_finish_split()`，避免在 parent 结构未补全时继续插入。
锁边界可以记成：
read descent 通常单页锁。
move right 通常释放旧页再锁右页。
split 更新 sibling 链需要 left-to-right 短 coupling。
parent insertion 需要 child/parent atomic coupling。
backward scan 需要额外验证 leftlink。
可观测入口包括 `pageinspect`、`EXPLAIN BUFFERS`、`pg_stat_all_indexes`、gdb、perf 和 injection points。
SQL 能看到 page opaque 和 high key。
SQL 很难直接看到一次 `_bt_moveright()` 右追赶。
从本节抽象出的可迁移规律是：
高并发树结构不要把上层导航指针当作唯一事实。
当结构修改会移动 keyspace 边界时，要在被导航到的对象本身保存可验证边界，并提供单调恢复方向。
PostgreSQL nbtree 用 high key/rightlink 把 stale parent pointer 变成局部右追赶。
这个设计把 read path 的锁持有时间压低，同时把少量复杂性转移给 split、WAL 和 writer-side finish split。
