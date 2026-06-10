# PostgreSQL B-tree page、high key 与 sibling link
## 课程定位
本节主题：B-tree page、high key 与 sibling link。
上一组课程已经从 heap page 走到 FSM、VM 与 heap storage 的辅助状态。
从这一节开始，主线进入 B-tree。
前置知识：
- 已理解 buffer pin 与 content lock 的职责边界。
- 已理解 page header、line pointer、special space、page LSN。
- 已理解 WAL-before-data 和 redo 的基本契约。
- 已理解 heap TID 是 index tuple 指向 heap tuple 的物理地址。
- 已理解普通 index scan 要先在 index 中定位，再回 heap 做可见性判断。
本节唯一主问题：
父页 downlink 可能因为并发 split 变得过期时，PostgreSQL B-tree 搜索为什么仍能找到正确的叶子页？
本节围绕的核心矛盾：
读路径希望只拿短暂的单页读锁，不能沿 root 到 leaf 长时间锁住整条路径。
写路径又必须允许 leaf 和 internal page 在并发插入中被拆分。
split 会让父页里的 downlink 在短时间内不能完整描述子层最新结构。
如果搜索完全相信父页 downlink，就可能落到已经被拆分后只覆盖较小 key range 的旧页。
如果搜索为了避免这个问题而锁住整棵树或整条路径，并发插入和 range scan 的吞吐会崩掉。
PostgreSQL 的答案是：
每个非 rightmost page 用 high key 描述本页 key range 的上界。
每个 page opaque 里保存 right sibling link。
搜索先按父页 downlink 下降。
下降到页面后再用本页 high key 自检。
如果 search key 已经超过本页 high key，就沿 rightlink 追到右边兄弟页。
这个过程可以重复，因为同一页可能被拆分多次。
学完本节，你应该能独立判断：
- `BTPageOpaqueData` 为什么在 page special space，而不是普通 index tuple。
- `btpo_next` 为什么是并发 split 之后搜索恢复正确性的核心状态。
- `btpo_prev` 为什么主要服务 backward scan 和 sibling 校验，而不是本节主问题。
- `P_HIKEY` 为什么是 item 1，而不是页面末尾的特殊字段。
- rightmost page 为什么没有物理 high key，而是把 high key 视为正无穷。
- internal page 的第一个 data item 为什么要被比较逻辑当作负无穷。
- 为什么父页 downlink 可以短暂落后，但搜索不能因此丢 key。
- `_bt_search()` 为什么每下降一层都调用 `_bt_moveright()`。
- `_bt_moveright()` 为什么比较 `P_HIKEY` 后可能沿 `btpo_next` 右移。
- 普通定位和 `nextkey` 定位为什么分别使用 `>` 和 `>=` high key 的边界。
- page split 为什么先把左右兄弟链和 high key 做成可搜索状态，再插入父页 downlink。
- `BTP_INCOMPLETE_SPLIT` 为什么标在左页，而不是标在缺少 downlink 的右页。
- 为什么 reader 通常不关心 incomplete split，而 writer 必须帮忙 finish。
- root split 之后，即使 backend 读到了旧 metapage，为什么仍能通过 rightlink 找到移动到右边的数据。
- WAL redo 为什么也把 split 拆成读者可接受的原子状态。
- 哪些现象能用 `pageinspect` 看到，哪些只能通过断点、DEBUG 日志或 injection point 观察。
## 源码基线
源码仓库：
```text
/home/nail/postgres-lab
```
基线：
```text
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
commit date: 2026-06-09
```
本节重点阅读：
```text
src/backend/access/nbtree/README
src/backend/access/nbtree/nbtpage.c
src/backend/access/nbtree/nbtsearch.c
src/backend/access/nbtree/nbtinsert.c
src/backend/access/nbtree/nbtxlog.c
src/include/access/nbtree.h
```
辅助观察：
```text
contrib/pageinspect/btreefuncs.c
doc/src/sgml/pageinspect.sgml
```
行号来自：
```text
git -C /home/nail/postgres-lab show bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8:<file> | nl -ba
```
本节不会展开完整 unique check、dedup、page deletion 和 index vacuum。
这些内容会在后续第 37、39、41 节继续讲。
本节只把它们与 high key、rightlink、page opaque 相交的边界讲清楚。
## 1. 先给结论
PostgreSQL 的 nbtree 使用 Lehman-Yao 风格的高并发 B-tree。
`README:6-12` 明确说这个目录实现的是 Lehman and Yao high-concurrency B-tree algorithm，并借用了 Lanin and Shasha 的删除逻辑。
这个算法对普通 B-tree 加了两个关键状态。
第一个是每页的 right sibling link。
第二个是每个非 rightmost page 的 high key。
`README:17-22` 给出核心理由：
rightlink 和 high key 让搜索可以检测并发 page split。
检测到自己落在 split 后的左页时，搜索沿 rightlink 右移。
这样就不需要为了读路径而长期持有整条 search path 上的锁。
PostgreSQL 还保留 page-level read lock。
`README:63-68` 解释原因：
Lehman-Yao 假设内存页副本不共享，而 PostgreSQL 的 shared buffer 在多个 backend 之间共享。
所以读者仍要拿 page read lock，保证读页面时页面内容不会被并发修改。
这不是为了防止父指针过期。
这是为了防止同一 buffer page 被读写并发撕裂。
本节的一句话运行模型是：
```text
父页 downlink 只负责把搜索带到某个不在目标 key 右侧的候选页；
候选页的 high key 负责判断这个页是否已经太靠左；
rightlink 负责把搜索修正到当前覆盖目标 key 的右侧兄弟页。
```
这句话里有三个边界。
第一，父页 downlink 不是全局实时 truth。
它可能落后于子层 split。
第二，high key 是 page-local truth。
读者拿着该页锁时，可以把它作为本页 key range 的上界。
第三，rightlink 是 monotonic recovery path。
split 只把一部分 key 搬到新右页。
因此一个过旧 downlink 最多让搜索落在目标 key 的左边，不会落在目标 key 的右边。
只要持续向右检查 high key，就能追到正确页。
这个机制解释了两个常见现象。
第一个现象：
range scan 过程中可能看见页分裂，但不会漏掉应该返回的 index tuple。
第二个现象：
插入压力很大时，B-tree 可能出现短暂 incomplete split。
reader 通常仍能搜索。
后续 writer 会在插入前完成缺失的父页 downlink。
## 2. 本节在总主线中的位置
第 27 到 33 节讲的是 heap page 与 heap 辅助 fork。
那些课程回答的是：
tuple version 如何存在 heap page 上。
heap page 的空间和可见性如何被近似记录。
现在进入 index page。
index page 的第一个问题不是“索引如何比较 key”。
更基础的问题是：
一个索引页在并发结构修改中，如何让读者判断自己还在正确的 key range 里。
如果没有这个判断，后面所有 B-tree search、insert、unique check、dedup、vacuum 都没有可靠基础。
经典 B-tree 可以假设一次 search 按父页 separator key 下降。
父页告诉你该去哪个 child。
child 的内容在搜索期间不会发生会破坏路径的变化。
PostgreSQL 不能这么假设。
一个 backend 从父页读到 downlink 后会释放父页锁，再去锁 child。
在这段时间里，child 可能已经 split。
父页可能还没有插入新右页 downlink。
甚至 root split 后，backend 可能从旧 metapage cache 开始下降。
所以本节不把 B-tree page 当静态布局讲。
本节从一个 race 展开：
```text
T1 读父页 downlink，准备去 child C。
T2 把 C split 成 C 和 R。
T2 先让 C.highkey 和 C.rightlink 指向新状态。
T1 仍然沿旧 downlink 到 C。
T1 发现 search key 超过 C.highkey。
T1 沿 C.rightlink 到 R。
T1 在 R 重新检查 high key，直到停在正确页。
```
这个故事是本节后面所有结构字段、函数调用和实验的主轴。
## 3. 源码阅读顺序
推荐先读 `README`，再读头文件，最后读搜索和 split 代码。
不要从 `nbtpage.c` 顶部线性读。
本节阅读顺序如下。
| 顺序 | 文件 | 读什么 | 目标 |
| --- | --- | --- | --- |
| 1 | `src/backend/access/nbtree/README` | Lehman-Yao、high key、rightlink、incomplete split、recovery scan | 先建立并发结构修改模型 |
| 2 | `src/include/access/nbtree.h` | `BTPageOpaqueData`、`P_HIKEY`、`P_FIRSTDATAKEY`、page flags | 明确状态放在哪里 |
| 3 | `src/backend/access/nbtree/nbtpage.c` | `_bt_getroot()`、`_bt_getbuf()`、`_bt_pageinit()`、`_bt_allocbuf()` | 看 root、buffer pin/lock 和 page special space |
| 4 | `src/backend/access/nbtree/nbtsearch.c` | `_bt_search()`、`_bt_moveright()`、`_bt_binsrch()`、`_bt_compare()` | 跟搜索如何从过期 downlink 恢复 |
| 5 | `src/backend/access/nbtree/nbtinsert.c` | `_bt_insertonpg()`、`_bt_split()`、`_bt_insert_parent()`、`_bt_finish_split()` | 看 writer 如何制造 high key/rightlink 状态 |
| 6 | `src/backend/access/nbtree/nbtxlog.c` | `btree_xlog_split()`、`btree_xlog_insert()`、`_bt_clear_incomplete_split()` | 看 crash recovery 如何保持同一不变量 |
如果只读一条主链路，按这个顺序：
```text
_bt_search()
  -> _bt_moveright()
  -> _bt_binsrch()
  -> _bt_compare()
  -> _bt_insertonpg()
  -> _bt_split()
  -> _bt_insert_parent()
  -> _bt_finish_split()
  -> btree_xlog_split()
```
其中 `_bt_insert_parent()` 很长，本节只抓两件事。
它把右页 downlink 插到父页。
它清掉左页上的 `BTP_INCOMPLETE_SPLIT`。
## 4. B-tree page 的三个状态层
一个 nbtree page 至少有三层状态。
第一层是通用 page header。
这层来自 buffer/page 子系统。
它保存 `pd_lower`、`pd_upper`、`pd_special`、page LSN 等。
本节只用到两点。
items 存在 page 的 item area。
opaque 存在 special space。
第二层是 index tuple array。
leaf page 的普通 data item 保存 index key 加 heap TID。
internal page 的 data item 是 pivot tuple。
pivot tuple 不指向 heap tuple，而是用于 tree navigation。
`README:31-40` 说明 PostgreSQL 用 pivot tuple 来表达 separator key、downlink 或两者的组合。
第三层是 nbtree page opaque。
`nbtree.h:63-70` 定义 `BTPageOpaqueData`。
它包括：
```text
btpo_prev    left sibling, or P_NONE
btpo_next    right sibling, or P_NONE
btpo_level   leaf level is 0
btpo_flags   page status flags
btpo_cycleid vacuum cycle id of latest split
```
这三个层次不能混成一个语义。
`btpo_next` 不在普通 index tuple 里。
它不参与 key ordering。
它是结构恢复路径。
high key 是普通 page item 的一个特殊位置。
它参与 key comparison。
它是 key range 上界。
`btpo_flags` 判断 page 是否 leaf、root、deleted、half-dead 或 incomplete split。
这些 flag 不直接参与 key comparison，但会影响搜索是否应该忽略当前页或 writer 是否必须 finish split。
所以本节要记住：
```text
page item 决定 key range；
page opaque 决定 sibling topology 与 page lifecycle；
buffer lock/pin 决定读写这两个状态时的内存安全。
```
## 5. `BTPageOpaqueData` 的语义边界
`nbtree.h:33-37` 的注释直接说明：
page opaque 保存左右 sibling 指针。
next-page link 对 recovery 很关键。
当搜索因为并发 split 或 deletion 导航到错误页面时，它可以用 next link 恢复。
这里的 recovery 不是 crash recovery 专用词。
它也指正常运行时从 stale path 恢复。
`btpo_prev` 是左 sibling。
它服务 backward scan 和 split 时维护双向链。
它也用于 sibling link 校验。
但本节主问题靠的是 `btpo_next`。
因为 split 是 split right。
旧 page 仍在左边。
新 page 出现在右边。
过期 downlink 导致的错误方向也是“太靠左”。
`btpo_next` 是右 sibling。
如果它是 `P_NONE`，宏 `P_RIGHTMOST(opaque)` 返回 true。
rightmost page 没有物理 high key。
它的 high key 是正无穷。
因此 `_bt_moveright()` 在 `P_RIGHTMOST` 时停止。
`btpo_level` 从 leaf 的 0 开始向上数。
这让 root split 不需要给已有页面重新编号。
`README:1046-1050` 明确说 level 从 leaf 0 数到 root depth minus 1。
`btpo_flags` 包含多类状态。
本节最重要的是：
```text
BTP_LEAF
BTP_ROOT
BTP_DELETED
BTP_HALF_DEAD
BTP_INCOMPLETE_SPLIT
```
`nbtree.h:219-228` 提供宏：
`P_ISLEAF()` 判断 leaf。
`P_RIGHTMOST()` 判断 rightmost。
`P_IGNORE()` 判断 deleted 或 half-dead。
`P_INCOMPLETE_SPLIT()` 判断 split 是否缺少父页 downlink。
`P_IGNORE()` 出现在 `_bt_moveright()` 的右移条件里。
这说明搜索不仅要从 split 中恢复，也要绕过删除中的页面。
但 page deletion 不是本节主线。
我们只需要知道：
half-dead/deleted 页不能承载正常搜索结果。
遇到这类页时，搜索也沿 rightlink 继续。
## 6. high key 的物理位置
`nbtree.h:360-370` 是理解 page layout 的最短入口。
非 rightmost page 的 high key 存在 item 1。
宏定义为：
```text
P_HIKEY = 1
P_FIRSTKEY = 2
P_FIRSTDATAKEY(opaque) = P_RIGHTMOST(opaque) ? P_HIKEY : P_FIRSTKEY
```
这意味着：
非 rightmost page 的第一个真实 data item 从 offset 2 开始。
rightmost page 没有 high key，所以第一个真实 data item 从 offset 1 开始。
`README:1061-1067` 也解释了这个布局。
把 high key 放在左侧看起来有点反直觉。
但这样插入普通 data item 时，不需要一直移动页面右端的 high key。
这也是为什么用 `pageinspect.bt_page_items()` 看非 rightmost leaf page 时，offset 1 往往不是一个普通 heap TID item。
它是 pivot-style high key。
high key 的 link/downlink 部分不用于 leaf high key。
它的 key 部分用于和 search key 比较。
rightmost page 没有 high key item。
如果你在 rightmost page 上把 offset 1 当 high key，就会误读页面。
正确判断必须先看 `bt_page_stats(...).btpo_next` 或 page opaque。
如果 `btpo_next = 0`，它是 rightmost。
这时 offset 1 是普通 data item。
如果 `btpo_next <> 0`，offset 1 才是 high key。
## 7. high key 的语义不是最大值缓存
high key 不是“本页当前最大 key 的缓存”。
它是本页 key range 的上界。
大多数时候它和左页最后一个 data item 有紧密关系。
但在 PostgreSQL 当前实现里，suffix truncation 会让 pivot tuple 只保存足够区分页范围的前缀。
`README:51-55` 说明 suffix truncation 必须仍然满足 Lehman-Yao 不变量。
缺失或截断的属性在 pivot tuple 中被视为负无穷。
`nbtsearch.c:783-792` 也在比较时把被截断的属性按负无穷处理。
因此 high key 的 raw bytes 不能简单解释成用户 key 的完整最大值。
正确语义是：
```text
high key + heapkeyspace 比较规则 + suffix truncation 规则 + page 是否 rightmost = 本页上界
```
`README:42-49` 还补了一条重要约束。
PostgreSQL B-tree 把 heap TID 当作 tie-breaker attribute。
逻辑重复 key 会按 heap TID 排序。
这让同一层上的 key 可以有可靠的全序。
没有这个全序，判断 `search key > high key` 的边界会在重复 key 上变得含糊。
所以 high key 的正确性不是一个字段单独保证的。
它依赖 operator class 比较函数、NULL 排序、DESC 标志、heap TID tie-breaker 和 pivot truncation 一起成立。
## 8. internal page 的第一个 data item
internal page 有一个容易误读的点。
每个 internal data item 是一个 downlink。
downlink 左侧有一个 lower bound。
但第一个 downlink 没有真实 lower bound。
`README:1072-1081` 说，non-leaf page 的第一个 data item 没有 lower bound，或者说 lower bound 是负无穷。
`nbtsearch.c:679-685` 在 `_bt_compare()` 的注释里把这点写得很明确。
在 internal page 上，第一个 data key 会被当作负无穷。
`nbtsearch.c:707-712` 直接实现了这个规则：
如果不是 leaf，并且比较目标是 `P_FIRSTDATAKEY(opaque)`，比较函数强制认为 scankey 大于这个 item。
这样 `_bt_binsrch()` 在 internal page 上总能找到某个 child downlink。
这也解释了为什么 high key 和 data item 的角色不能混淆。
high key 是本页最后一个 child range 的上界。
第一个 data item 是第一个 child 的 downlink，比较时相当于负无穷 lower bound。
它们都可能是 pivot tuple。
但它们在导航中的方向相反。
## 9. 搜索主流程概览
`_bt_search()` 是本节读路径主入口。
`nbtsearch.c:76-95` 的注释说明：
它搜索某个 scan key 可以所在的第一个 leaf page。
返回时 leaf page buffer 被 pin 并按请求模式加锁。
父页锁不会保留。
这正是本节 race 的来源。
搜索路径不能依赖父页锁一直保护 child。
主流程是：
```text
_bt_search()
  -> _bt_getroot()
  -> loop per level
     -> _bt_moveright()
     -> if leaf stop
     -> _bt_binsrch()
     -> BTreeTupleGetDownLink()
     -> _bt_relandgetbuf(child)
```
`nbtsearch.c:110-115` 先通过 `_bt_getroot()` 找 root。
如果 read 模式下空索引没有 root，就返回 `InvalidBuffer`。
`nbtsearch.c:117-186` 是逐层下降循环。
每一层刚拿到页面后，第一件事不是二分查找。
而是调用 `_bt_moveright()`。
`nbtsearch.c:128-140` 注释解释了 race：
刚拿到的页面可能在父页 downlink 被读取之后 split。
如果 split 发生，就可能需要向右移动到新 sibling。
`_bt_moveright()` 修正完页面后，`_bt_search()` 才在该页上判断是否 leaf。
如果不是 leaf，`_bt_binsrch()` 在当前 internal page 找合适 pivot tuple。
再从 pivot tuple 中取 downlink 下降。
下降时 `_bt_relandgetbuf()` 释放当前页锁和 pin，然后读并锁 child。
这意味着同一时刻通常只持有一个页面锁。
读路径没有沿整条路径做 lock coupling。
正确性就转移给 high key 和 rightlink。
## 10. `_bt_moveright()` 的停止条件
`_bt_moveright()` 的注释在 `nbtsearch.c:211-239`。
它说明：
当我们沿某个指针到达一个 page 时，这个 page 可能已经变化。
如果变化发生，保证是 split right。
原来在这个 page 上的数据要么还在这个 page，要么在它右边。
这句话是整个机制的关键。
它让搜索只需要向一个方向修正。
核心判断在 `nbtsearch.c:275-315`。
循环里先取 page opaque。
如果 `P_RIGHTMOST(opaque)`，停止。
因为 rightmost 的 high key 是正无穷。
如果 writer 模式并且页面有 `P_INCOMPLETE_SPLIT`，先 finish split。
这个路径后面单独讲。
如果 `P_IGNORE(opaque)`，向右移动。
如果 `_bt_compare(rel, key, page, P_HIKEY) >= cmpval`，向右移动。
否则停止。
`cmpval` 由 `key->nextkey` 决定。
`nbtsearch.c:256-265` 解释了两个边界：
普通场景 `nextkey = false`，搜索想找第一个 `>= key` 的 item。
只有当 `search key > high key` 时才右移。
`nextkey = true` 时，搜索想找第一个 `> key` 的 item。
当 `search key >= high key` 时就右移。
写成表是：
| 场景 | 目标 | high key 边界 | 右移条件 |
| --- | --- | --- | --- |
| `nextkey=false` | first item `>= key` | high key 仍可能属于左页 | `key > high key` |
| `nextkey=true` | first item `> key` | 等于 high key 的 item 不再满足 | `key >= high key` |
这不是微优化。
这是 range scan 初始定位的正确性边界。
`_bt_first()` 会根据 `<`、`<=`、`=`、`>=`、`>` 构造 insertion-style scan key。
`nbtsearch.c:1435-1500` 把不同策略映射到 `nextkey` 和 `backward`。
所以 high key 比较同时服务点查、range scan 和 backward scan 的起点定位。
## 11. `_bt_binsrch()` 只负责页内定位
`_bt_binsrch()` 不负责右移。
`nbtsearch.c:339-341` 明确说它只检查给定页面，没有 lock 或 refcount 副作用。
这点很重要。
如果你在调试时看到 `_bt_binsrch()` 返回了 page 内某个 offset，不要以为它已经确认 page 是全局正确 leaf。
全局正确性已经由前面的 `_bt_moveright()` 负责。
`_bt_binsrch()` 在 internal page 上返回的是 child downlink 的 offset。
`nbtsearch.c:327-331` 说，它返回最后一个 `< scankey` 的 key，或者 `nextkey` 场景返回最后一个 `<= scankey` 的 key。
由于第一个 data item 被视为负无穷，所以一定能找到一个 downlink。
在 leaf page 上，`_bt_binsrch()` 返回初始 scan positioning 的结果。
`nbtsearch.c:409-427` 说明 forward scan 返回第一个满足边界的 non-pivot tuple。
backward scan 则退一步，定位到最后一个满足边界的 non-pivot tuple。
所以搜索整体分两段：
```text
跨页正确性：_bt_moveright()
页内定位：_bt_binsrch()
```
把这两段分开理解，才能解释很多断点现象。
你可能在一个 leaf page 上看到 `_bt_binsrch()` 返回 `maxoff + 1`。
这不一定是错误。
`nbtsearch.c:1560-1563` 说明，如果 scan key 小于 high key 但大于所有 non-pivot tuples，offset 可能越过最后一个 item。
后续 scan 读取逻辑会处理跨页移动。
## 12. split 为什么只向右
Lehman-Yao 机制成立的前提是 split right。
原 page 作为 left page 保留原 block number。
新 page 分配为 right sibling。
父页旧 downlink 仍指向原 block。
这让旧 downlink 至少能把搜索带到目标范围左边或目标范围本身。
如果 split 可能把原页变成右页，旧 downlink 就可能把搜索带到目标范围右边。
那时 rightlink 无法修正。
`_bt_split()` 维护这个前提。
`nbtinsert.c:1462-1487` 的注释说明：
入口 buffer 是要 split 的页。
返回的是新的 right sibling。
原 buffer 的 pin 和 lock 保持。
`nbtinsert.c:1523-1534` 解释工作空间：
`leftpage` 是临时 buffer，成功后会复制回原 page。
`rightpage` 是新页的临时内容，最终复制到新分配的 right buffer。
`nbtinsert.c:1955-1961` 在 critical section 中把 `leftpage` 复制回原 page。
这一步守住了“左页不移动”的不变量。
原 block number 仍然是父页旧 downlink 指向的 block。
新 right page 通过 `lopaque->btpo_next` 接到右边。
这就是 stale downlink 可恢复的物理基础。
## 13. split 如何生成 high key
split 先选 split point。
`nbtinsert.c:1543-1568` 把 split point 解释成两个 item 之间的位置。
在概念上，必须把即将插入的新 item 也想象成已经在原页上。
然后选出 left side 的最后 item 和 right side 的 first item。
`firstright` 是右页第一个 item。
left page 的新 high key 来自这个边界。
leaf page 上会尝试 suffix truncation。
`nbtinsert.c:1623-1641` 解释：
leaf split 的新 left high key 是 `firstright` 的可能截断副本。
它不必精确复制 lastleft。
只要保证 left high key 大于等于 lastleft，并且严格小于 firstright，就能维持搜索边界。
当不能截断时，会接近 Lehman-Yao 的精确做法。
`nbtinsert.c:1659-1683` 调用 `_bt_truncate()` 生成 leaf high key。
internal page split 则不同。
`nbtinsert.c:1687-1714` 说明 internal split 不能对 `firstright` 做同样的 suffix truncation 来生成 left high key。
因为上层 separator key 与下层 high key 之间必须保持一致的导航边界。
这里不要把 leaf high key 和 internal separator 的历史来源混在一起。
本节只需要记住：
leaf high key 可以为了节省空间做 suffix truncation。
internal split 的 left high key 要保留从下一层传上来的 separator 语义。
`nbtinsert.c:1717-1729` 把新 high key 加到 left page 的 `P_HIKEY`。
这个动作发生在分配 right buffer 之前。
源码注释强调：
这是 split 仍可能失败且尚无持久后果的最后阶段之一。
## 14. split 如何维护 sibling link
新 right page 分配后，`_bt_split()` 填 opaque。
`nbtinsert.c:1762-1764` 设置：
```text
left.btpo_next = right block
left.btpo_cycleid = current vacuum cycle id
```
`nbtinsert.c:1769-1774` 设置 right page：
```text
right flags = old flags minus root/split_end/garbage
right.btpo_prev = old block
right.btpo_next = old.btpo_next
right.btpo_level = old.btpo_level
right.btpo_cycleid = left.btpo_cycleid
```
如果原 page 不是 rightmost，right page 还要继承原 page 的 high key。
`nbtinsert.c:1777-1801` 把原 `P_HIKEY` 加到 right page 的 `P_HIKEY`。
如果原 page 是 rightmost，新 right page 仍然是 rightmost。
它没有 high key。
left page 总是得到新的 high key。
right page 是否有 high key 取决于它是不是 rightmost。
split 还必须更新原右兄弟的 left link。
`nbtinsert.c:1907-1914` 在原 page 非 rightmost 时锁住旧右兄弟。
`nbtinsert.c:1917-1924` 校验旧右兄弟的 `btpo_prev` 是否指向原 page。
如果不匹配，报 index corruption。
`nbtinsert.c:1977-1980` 在 critical section 中把旧右兄弟的 `btpo_prev` 改成新 right page。
这服务 backward scan 和 sibling consistency。
但是 forward search 的 split recovery 主要依赖 left page 的 `btpo_next`。
## 15. split 的 critical section 边界
`_bt_split()` 在真正改原 page 之前可以报错。
例如 high key 加入临时 page 失败，或者 sibling link 校验失败。
一旦进入 critical section，就不能普通 ERROR 退出。
`nbtinsert.c:1944-1952` 明确说：
right sibling 已锁，新 siblings 已准备好，原 page 尚未更新。
从这里开始进入 critical section。
critical section 内的关键动作是：
```text
copy leftpage back to original page
copy rightpage temp content to new right buffer
mark left and right dirty
if old right sibling exists, update its btpo_prev
if split is internal-child completion, clear child incomplete flag
write WAL
set page LSNs
```
`nbtinsert.c:1961-1980` 完成前三类页面内容变化。
`nbtinsert.c:1983-1994` 在 internal insert 场景清掉下一层 child 的 incomplete flag。
`nbtinsert.c:1996-2075` 开始组织 WAL。
其中 `nbtinsert.c:2058-2066` 特别 WAL 记录 left page 的新 high key。
这告诉 redo：
重建 split 时，left page 的 high key 是结构修改的一等状态。
不是可以从其他 item 随便推导的 hint。
split WAL record 必须足够重建 right page 和 left page 的边界。
否则 crash 后的读者无法用 high key/rightlink 恢复搜索。
## 16. parent insertion 为什么是第二个动作
split 本身只让子层变成两个 sibling page。
它还没有把新 right page 的 downlink 插入 parent。
`nbtinsert.c:1238-1263` 在 `_bt_insertonpg()` 里描述 split 后状态：
target page 已被 split。
原 tuple 已插入。
左右 child buffer 都有 write lock。
要插入 parent 的 key 已知。
这个 key 就是 left child page 上的 high key。
然后调用 `_bt_insert_parent()`。
为什么不把 parent 插入和 child split 做成一个巨大原子操作？
因为 parent 可能也满。
插入 parent downlink 可能导致 parent split。
parent split 又可能递归向上。
如果把整条链都做成一个不可中断动作，锁持有和错误恢复会变得不可接受。
PostgreSQL 的设计是：
child split 先形成一个对 reader 正确的状态。
parent downlink 插入作为第二个结构动作完成。
两个动作之间用 `BTP_INCOMPLETE_SPLIT` 约束 writer。
reader 靠 high key/rightlink 仍能找到 right page。
writer 不能继续在 incomplete split 的左页上插入，因为 parent 还不知道右页。
这就是 read correctness 和 write structural completion 的分工。
## 17. `BTP_INCOMPLETE_SPLIT` 为什么标在左页
`README:666-681` 解释得非常直接。
page split 后，左页被标记为 `INCOMPLETE_SPLIT`。
当新 right page 的 downlink 插入 parent 后，这个 flag 和 parent 插入一起原子清除。
flag 标在左页，虽然真正缺少 downlink 的是右页。
原因是：
沿 left page 的 rightlink 到 right page 时，系统已经知道 right page 还需要 parent downlink。
这对后续 writer 很方便。
`nbtree.h:84` 定义：
```text
BTP_INCOMPLETE_SPLIT = right sibling's downlink is missing
```
注意注释里的主语。
它不是说“当前页缺少 downlink”。
它说当前页的右 sibling 缺少 downlink。
`nbtinsert.c:1579-1582` 在准备 left page 时设置这个 flag。
`nbtinsert.c:1318-1329` 在 parent insert 到 internal page 时清 child flag。
`nbtinsert.c:1983-1994` 在 page split 插入内部 downlink 时也处理 cascading split 的 child flag。
所以 incomplete split 的生命周期是：
```text
child split 创建右页
  -> left page 标 BTP_INCOMPLETE_SPLIT
  -> parent 插入 right page downlink
  -> 同一 critical section 清 left page flag
```
## 18. reader 为什么通常不关心 incomplete split
reader 的目标是找到 key。
只要 sibling chain 和 high key 已经正确，reader 不需要 parent 已经有 right page downlink。
如果 reader 通过旧 parent downlink 到 left page，它会用 high key 判断并沿 rightlink 到 right page。
如果 reader 从更新后的 parent downlink 直接到 right page，也没问题。
这两个路径都能到达正确 key range。
`README:705-710` 说 Hot Standby 中的 reader 也遵循类似规则。
每个 WAL atomic action 都让树处在 reader 一致的状态。
reader 可能要像 primary 上一样右移，来从“并发” split 或 deletion 中恢复。
`README:715-724` 进一步说明：
redo 不重建 primary 上 parent-child lock coupling。
这是安全的，因为 reader 不关心 incomplete split flag。
primary 上额外持锁主要是为了防止第二个 writer 观察到同一个 incomplete split，并重复尝试完成它。
所以 incomplete split 不是“读者看到就不安全”的状态。
它是“writer 继续修改前必须补齐父链接”的状态。
## 19. writer 如何 finish incomplete split
`_bt_moveright()` 有 writer-only 分支。
`nbtsearch.c:232-235` 说：
如果 `forupdate` 为 true，遇到 incomplete split 时会尝试 finish。
插入目标页前必须这样做，因为不允许在 split 未完成的页上插入。
实际代码在 `nbtsearch.c:283-304`。
如果当前页 `P_INCOMPLETE_SPLIT`：
必要时把 read lock 升级到 write lock。
再次检查 flag。
如果 flag 仍在，调用 `_bt_finish_split()`。
之后重新获取原 block 的指定 lock mode，再重新检查。
`_bt_finish_split()` 在 `nbtinsert.c:2259-2317`。
入口要求 left buffer 已经 write locked。
它锁住 left page 的 right sibling，也就是缺少 downlink 的右页。
如果没有 stack，它还会检查这是否可能是 root split。
然后调用 `_bt_insert_parent()`。
`nbtinsert.c:2312-2314` 有 DEBUG1 日志：
```text
finishing incomplete split of left/right block
```
这条日志是观察 incomplete split fallback 的少数直接入口。
源码里还提供 injection point：
`nbtinsert.c:1258` 是 `nbtree-leave-leaf-split-incomplete`。
`nbtinsert.c:1260` 是 `nbtree-leave-internal-split-incomplete`。
`nbtinsert.c:2312` 附近是 `nbtree-finish-incomplete-split`。
如果本地编译启用了 injection points，可以用它们人为扩大窗口。
## 20. root split 为什么也安全
root split 是经典 B-tree 最容易特殊化的地方。
PostgreSQL 的 README 明确说 Lehman-Yao 没有讨论 root 满了怎么办。
`README:112-123` 给出 PostgreSQL 的做法：
root 像普通页一样 split。
然后构造新 root，指向 split 后的两个 page。
再通过 metapage 安装新 root。
关键句是：
root 不以其他方式被特殊对待。
如果旧 root 的 link 被设置，search 也会沿它的 link 向右移动。
因此，即使 backend 在 metapage 更新前读到旧 root，也能通过 rightlink 找到移动到右兄弟的数据。
`_bt_getroot()` 还有 fast root 缓存。
`nbtpage.c:360-380` 先尝试使用 relcache 中的 metapage cache。
`nbtpage.c:384-403` 会校验 cached fast root 是否仍然 live、level 是否匹配、是否单独位于其 level。
如果缓存不合适，就丢掉缓存重新读 metapage。
这里允许 cache slightly stale。
正确性仍靠下降后的 page-level 校验和 rightlink。
root/fast root 的原则是：
入口可以是一个保守可用的起点。
它不必是每个瞬间的完美最新 root。
只要 search descent 和 move-right 能修正，就不会漏 key。
## 21. WAL redo 如何保持同一不变量
split 的 WAL redo 在 `nbtxlog.c:247-451`。
它重建新 right sibling page。
`nbtxlog.c:283-297` 初始化 right page opaque：
```text
right.btpo_prev = original page
right.btpo_next = old right sibling
right.btpo_level = WAL level
right.btpo_flags = leaf ? BTP_LEAF : 0
```
然后用 `_bt_restore_page()` 恢复 right page items。
接着重建 original page，也就是 split 后的 left page。
`nbtxlog.c:363-369` 把 WAL record 里的 left high key 加到 `P_HIKEY`。
`nbtxlog.c:412-419` 修正 left page opaque：
```text
left.btpo_flags = BTP_INCOMPLETE_SPLIT plus BTP_LEAF if leaf
left.btpo_next = right block
left.btpo_cycleid = 0
```
如果存在旧右兄弟，`nbtxlog.c:425-441` 更新它的 `btpo_prev`。
最后 `nbtxlog.c:444-450` 一起释放相关 buffers。
源码注释说这些 buffer 必须一起释放，避免 reader 观察到 same-level inconsistency。
parent downlink 插入的 redo 则通过 insert/newroot 类 WAL record 清 incomplete flag。
`nbtxlog.c:130-155` 的 `_bt_clear_incomplete_split()` 清 `BTP_INCOMPLETE_SPLIT`。
`nbtxlog.c:167-176` 说明 internal page insertion 会完成下一层 incomplete split。
redo 不需要重建 primary 上全部锁耦合。
原因仍然是：
reader 能靠 high key/rightlink 搜索。
writer 在 recovery 中不存在。
## 22. page allocation 与 split 的失败边界
split 需要一个新 right page。
`_bt_allocbuf()` 在 `nbtpage.c:864-993`。
它先尝试从 index FSM 找可回收页。
`nbtpage.c:883-904` 说明 FSM 结果不能完全信任。
page 可能已被复用。
甚至不能假设锁这个 page 一定安全，因为可能和当前调用者已持有的锁形成 deadlock。
所以 `_bt_allocbuf()` 对 FSM 返回页使用 conditional lock。
如果锁不到，认为暂时不可用，释放 pin。
如果 page 是全零页或可回收页，初始化后返回。
否则继续找。
找不到可用 FSM page 时，扩展 relation。
`nbtpage.c:976-990` 用 `ExtendBufferedRel()` 扩展并初始化新页。
这说明 split 的 page allocation 也有 fallback。
它不是 blind trust FSM。
它宁可扩展文件，也不会为了复用一个可疑页破坏锁顺序或 standby conflict 语义。
本节主线里这点很重要：
rightlink 和 high key 解决搜索正确性。
但 writer 要制造新 right sibling，还必须处理可回收页、standby conflict、条件锁和 relation extension。
这些是结构修改的资源边界。
## 23. scan 为什么记住扫描时的 rightlink
普通 `_bt_search()` 是定位起点。
index scan 之后会在 leaf level 顺序走页。
`README:88-104` 讲了一个容易忽略的规则。
scan 在一个 leaf page 上会一次性找出匹配 item，并把 heap TID copy 到 backend-local storage。
随后访问 heap 时不再持有 index page lock。
有时仍保留 leaf page pin，以配合 VACUUM 和 TID recycling。
如果 scan 读完一个 page 后释放锁，页面可能被 split。
这时 scan 不能简单读取该 page 当前的 `btpo_next`。
因为当前 `btpo_next` 可能指向 split 新产生的 right page。
而新 right page 中的 item 在 split 前已经被 scan 从旧 page 读过。
如果 scan 跟当前 rightlink 走，就可能重复扫描这些 item。
所以 README 要求：
scan 必须记住 page 被扫描时的 rightlink。
之后 forward scan 移动到这个记住的 rightlink。
这和 `_bt_moveright()` 的起点定位不同。
起点定位用当前 high key/rightlink 修正 stale descent。
已开始的 scan 用扫描时保存的 sibling link 保持“页间位置”的一致性。
两者都依赖 sibling link。
但一个防漏，一个防重。
## 24. backward scan 与 leftlink
Lehman-Yao 原始算法只需要 rightlink 来支持并发搜索。
PostgreSQL 支持 backward scan，所以还保存 `btpo_prev`。
`README:70-86` 说明：
forward scan 可以直接使用 right sibling pointers。
backward scan 需要 left sibling link。
但 backward scan 还有额外复杂性。
当 scan 沿 leftlink 到左页时，左页可能在它被读取前发生 split。
于是 scan 需要向右移动，直到找到 rightlink 匹配自己来处的 page。
所以 leftlink 不是 high key/rightlink 机制的镜像替代。
它是 backward scan 的起点。
但为了验证它仍然是正确的左邻居，仍要结合 rightlink。
这也是为什么 split 时必须更新旧右兄弟的 `btpo_prev`。
如果这个 link 错了，backward scan 和 deletion 校验都会出问题。
`nbtinsert.c:1917-1924` 对旧右兄弟 leftlink 不匹配直接报 index corruption。
这类错误通常意味着索引物理结构已经破坏。
## 25. 正确性机制层次
本节的正确性不是 high key 一个字段单独保证的。
它至少有六层。
第一层是 buffer pin。
pin 保证页面所在 buffer 不会被替换。
`nbtpage.c:833-847` 说 nbtree 访问页面必须同时持有 pin 和 buffer lock。
第二层是 buffer content lock。
读者拿 read lock 时，writer 不能改页面内容。
writer 拿 write lock 时，可以安全重排 item、修改 opaque 和设置 LSN。
第三层是 page-local key range。
high key 表示本页上界。
rightmost page 的上界是正无穷。
第四层是 sibling topology。
`btpo_next` 让搜索从过旧 downlink 到达右侧新页。
`btpo_prev` 让 backward scan 和 sibling 校验成立。
第五层是 parent downlink。
parent downlink 让未来搜索更直接。
但短时间内它允许滞后。
滞后由 high key/rightlink 补偿。
第六层是 WAL atomic action。
split WAL、insert parent WAL、newroot WAL 和 redo 顺序必须让 crash 后每个中间状态都能被 reader 搜索。
如果缺一层，就会出不同类型的问题。
没有 buffer lock，会读到撕裂中的 page。
没有 high key，读者无法判断自己是否落在 split 后的旧左页。
没有 rightlink，读者即使知道自己太靠左，也不知道去哪。
没有 incomplete split，多个 writer 可能重复修复同一个缺失 parent downlink。
没有 WAL 对 high key/rightlink 的记录，crash recovery 后结构可能不可搜索。
## 26. 错误路径与 fallback
第一类 fallback 是 stale root 或 stale fast root。
`_bt_getroot()` 可以使用 relcache 中的 metapage cache。
如果 cached page 不再 live、level 不匹配或不是单页 fast root，就丢弃 cache 并重新读 metapage。
这不是错误。
这是允许局部缓存滞后后的正常修正。
第二类 fallback 是 stale parent downlink。
`_bt_search()` 每到一层都调用 `_bt_moveright()`。
如果 search key 超过 high key，就右移。
这也是正常路径的一部分。
第三类 fallback 是 incomplete split。
reader 忽略它。
writer 遇到它时调用 `_bt_finish_split()`。
如果 `_bt_finish_split()` 需要向上递归，可能进一步 split parent 或新建 root。
第四类 fallback 是 FSM 返回不可用 index page。
`_bt_allocbuf()` 如果 conditional lock 失败，或者 page 不可回收，就丢弃该候选页。
最坏情况扩展 relation。
第五类 fallback 是 page deletion 相关的 ignored page。
`_bt_moveright()` 遇到 `P_IGNORE()` 继续右移。
如果最后在 rightmost 仍是 ignored，报 `"fell off the end of index"`。
这通常是 corruption 级别，不是普通并发现象。
第六类是 sibling inconsistency。
split 时如果旧右兄弟的 leftlink 不指向原 page，`_bt_split()` 报 index corrupted。
这不是 retry。
因为同层拓扑已经不满足 B-tree 结构不变量。
第七类是 redo 中的 incomplete split。
redo split 后可能留下 left page 的 `BTP_INCOMPLETE_SPLIT`。
后续 insert/newroot WAL record 会清掉它。
如果 crash 发生在两者之间，恢复后的 tree 对 reader 仍可搜索。
之后 primary 上的 writer 会 lazily finish。
## 27. 成本与资源传播
high key/rightlink 的收益是避免读路径锁住整条 search path。
成本是每次下降到页面都要做一次 high key 检查。
在没有并发 split 时，这通常只是一次 page-local compare。
在有 split race 时，可能沿 rightlink 多读几个 page。
`nbtsearch.c:267-268` 注释明确说：
page 可能 split 多次，所以要按需一直扫描到右边。
这个额外成本与并发插入热点有关。
递增 key 的 workload 容易让 rightmost leaf 成为热点。
随机 key 则可能在多个 leaf page 上分散 split。
重复 key 多的 workload 还会受 heap TID tie-breaker、posting list 和 dedup 影响。
这些会改变 page split 频率和 high key 形态。
split 的写成本更高。
一次 leaf split 至少涉及：
原 page。
新 right page。
如果原 page 不是 rightmost，还涉及旧 right sibling。
之后还要插入 parent。
parent 可能继续 split。
如果 root split，还要更新 metapage。
WAL 也会随之放大。
`nbtinsert.c:1996-2075` 组织 split WAL。
right page 通常通过 WAL record 重建。
left high key 也被记录。
如果需要 full page image，WAL 体积进一步增加。
不过这换来的是读者可以在每个结构修改中间状态继续搜索。
换句话说：
PostgreSQL 在写路径接受复杂结构修改和 WAL 成本，换取读路径短锁和高并发搜索。
## 28. 观测入口：pageinspect 看 page opaque
最直接的 SQL 观测入口是 `pageinspect`。
`pageinspect` 提供：
```sql
CREATE EXTENSION pageinspect;
SELECT * FROM bt_metap('index_name');
SELECT * FROM bt_page_stats('index_name', blkno);
SELECT * FROM bt_page_items('index_name', blkno);
```
`pageinspect--1.8--1.9.sql` 中 `bt_page_stats()` 的输出包括：
```text
blkno
type
live_items
dead_items
avg_item_size
page_size
free_size
btpo_prev
btpo_next
btpo_level
btpo_flags
```
这正好对应本节 page opaque 的核心字段。
`bt_metap()` 输出包括：
```text
root
level
fastroot
fastlevel
allequalimage
```
这可以观察 root 和 fast root。
`bt_page_items()` 可以看 item offset。
在非 rightmost leaf page 上，offset 1 是 high key。
在 rightmost leaf page 上，offset 1 是普通 data item。
所以观察 high key 前必须先用 `bt_page_stats()` 判断 `btpo_next`。
## 29. 课堂实验 1：观察 high key 与 rightlink
准备数据：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS bt_hikey_demo;
CREATE TABLE bt_hikey_demo(id bigint PRIMARY KEY, payload text);
INSERT INTO bt_hikey_demo
SELECT g, repeat('x', 80)
FROM generate_series(1, 20000) AS g;
ANALYZE bt_hikey_demo;
```
查看 metapage：
```sql
SELECT * FROM bt_metap('bt_hikey_demo_pkey');
```
找到 leaf pages：
```sql
SELECT blkno, type, live_items, btpo_prev, btpo_next, btpo_level, btpo_flags
FROM generate_series(1, pg_relation_size('bt_hikey_demo_pkey') / 8192 - 1) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('bt_hikey_demo_pkey', g.blkno) AS s
WHERE type = 'l'
ORDER BY blkno
LIMIT 20;
```
挑一个 `btpo_next <> 0` 的 leaf page。
它不是 rightmost。
查看 offset 1：
```sql
SELECT *
FROM bt_page_items('bt_hikey_demo_pkey', <leaf_blkno>)
ORDER BY itemoffset
LIMIT 5;
```
预期观察：
offset 1 是该页 high key。
后面的 offset 才是普通 leaf data items。
再查看它的右兄弟：
```sql
SELECT *
FROM bt_page_stats('bt_hikey_demo_pkey', <btpo_next>);
SELECT *
FROM bt_page_items('bt_hikey_demo_pkey', <btpo_next>)
ORDER BY itemoffset
LIMIT 5;
```
如果右兄弟仍不是 rightmost，它也有自己的 high key。
你会看到 leaf pages 通过 `btpo_next` 串起来。
这个实验能看到 page opaque 的 `btpo_next` 和 high key 物理位置。
它不能直接证明并发 split 中 `_bt_moveright()` 被调用。
那个需要断点或人为扩大 race 窗口。
## 30. 课堂实验 2：rightmost page 没有 high key
找到 rightmost leaf：
```sql
SELECT blkno, type, live_items, btpo_prev, btpo_next, btpo_level, btpo_flags
FROM generate_series(1, pg_relation_size('bt_hikey_demo_pkey') / 8192 - 1) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('bt_hikey_demo_pkey', g.blkno) AS s
WHERE type = 'l' AND btpo_next = 0;
```
查看 rightmost leaf 的前几个 item：
```sql
SELECT *
FROM bt_page_items('bt_hikey_demo_pkey', <rightmost_leaf_blkno>)
ORDER BY itemoffset
LIMIT 5;
```
对比实验 1 的非 rightmost leaf。
预期观察：
rightmost leaf 的 itemoffset 1 是普通 data item。
它没有 high key。
这对应 `P_FIRSTDATAKEY(opaque)` 的宏。
非 rightmost page 的 first data key 是 2。
rightmost page 的 first data key 是 1。
诊断意义：
当你用 pageinspect 人工解析 B-tree page 时，不能写死 offset 1 是 high key。
必须先看 `btpo_next`。
## 31. 课堂实验 3：观察 split 后 sibling chain 变化
用较低 fillfactor 让 split 更容易发生：
```sql
DROP TABLE IF EXISTS bt_split_demo;
CREATE TABLE bt_split_demo(id bigint, payload text);
CREATE INDEX bt_split_demo_id_idx ON bt_split_demo USING btree (id) WITH (fillfactor = 50);
INSERT INTO bt_split_demo
SELECT g, repeat('x', 100)
FROM generate_series(1, 50000) AS g;
```
记录当前 leaf chain：
```sql
CREATE TEMP TABLE before_pages AS
SELECT blkno, btpo_prev, btpo_next, btpo_level, live_items
FROM generate_series(1, pg_relation_size('bt_split_demo_id_idx') / 8192 - 1) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('bt_split_demo_id_idx', g.blkno) AS s
WHERE type = 'l';
```
继续插入递增 key：
```sql
INSERT INTO bt_split_demo
SELECT g, repeat('y', 100)
FROM generate_series(50001, 90000) AS g;
```
再次观察：
```sql
SELECT blkno, btpo_prev, btpo_next, btpo_level, live_items
FROM generate_series(1, pg_relation_size('bt_split_demo_id_idx') / 8192 - 1) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('bt_split_demo_id_idx', g.blkno) AS s
WHERE type = 'l'
ORDER BY blkno;
```
预期观察：
leaf page 数增加。
rightmost 或接近 rightmost 的 sibling chain 改变。
某些旧 page 的 `btpo_next` 指向新 page。
这不能保证你捕捉到正在进行中的 incomplete split。
但它能稳定展示 split 后的持久拓扑。
如果你在插入前后对比同一个旧 leaf page 的 `btpo_next`，常能看到它从旧右兄弟变成新分裂页。
## 32. 课堂实验 4：用 gdb 看 `_bt_moveright()`
这个实验适合本地开发环境。
启动一个 PostgreSQL backend 后连接 gdb。
设置断点：
```gdb
break _bt_moveright
commands
  silent
  printf "_bt_moveright rel=%s access=%d forupdate=%d\n", RelationGetRelationName(rel), access, forupdate
  continue
end
```
为了避免断点过多，可以加条件。
例如只观察某个 index relation name。
在另一个会话中执行：
```sql
SELECT count(*) FROM bt_split_demo WHERE id BETWEEN 10000 AND 80000;
INSERT INTO bt_split_demo SELECT g, repeat('z', 100) FROM generate_series(90001, 120000) AS g;
```
更细的断点可以打在 `nbtsearch.c:307` 附近。
那里是 `_bt_compare(..., P_HIKEY) >= cmpval` 的右移判断。
观察变量：
```gdb
p cmpval
p opaque->btpo_next
p opaque->btpo_flags
p opaque->btpo_level
```
如果刚好命中 split race，能看到 `_bt_moveright()` 沿 rightlink 继续。
如果没有命中，也能看到每层 search 都会进入 `_bt_moveright()`。
这说明 high key 检查是常规路径，不是异常修复代码。
## 33. 课堂实验 5：人为扩大 incomplete split 窗口
源码中有 injection points。
相关位置：
```text
nbtinsert.c:1258  nbtree-leave-leaf-split-incomplete
nbtinsert.c:1260  nbtree-leave-internal-split-incomplete
nbtinsert.c:2312  nbtree-finish-incomplete-split
```
如果你的构建启用了 injection point 支持，可以在 leaf split 完成 child half 之后、parent insertion 之前暂停。
目标是制造这个状态：
```text
left page has BTP_INCOMPLETE_SPLIT
left.btpo_next points to right page
right page exists and is searchable via rightlink
parent downlink for right page is not yet inserted
```
然后在另一个会话执行读查询。
读查询应该仍能通过 high key/rightlink 找到数据。
再执行写入同一 index key range 的语句。
writer 会在插入前尝试 finish incomplete split。
如果设置 `client_min_messages = debug1`，有机会看到：
```text
finishing incomplete split of left/right
```
如果本地没有 injection point 支持，可以用 gdb 在 `nbtinsert.c:1258` 或 `_bt_insert_parent()` 前手动暂停 backend。
这个实验的重点不是让 SQL 永远稳定复现日志。
重点是观察两个事实：
reader 不需要 parent downlink 已经补齐。
writer 不能忽略 incomplete split。
## 34. 课堂实验 6：用 pageinspect 验证 key range
选择一个非 rightmost leaf page：
```sql
WITH leaf AS (
  SELECT blkno, btpo_next
  FROM generate_series(1, pg_relation_size('bt_hikey_demo_pkey') / 8192 - 1) AS g(blkno)
  CROSS JOIN LATERAL bt_page_stats('bt_hikey_demo_pkey', g.blkno) AS s
  WHERE type = 'l' AND btpo_next <> 0
  ORDER BY blkno
  LIMIT 1
)
SELECT leaf.blkno, leaf.btpo_next, i.*
FROM leaf
CROSS JOIN LATERAL bt_page_items('bt_hikey_demo_pkey', leaf.blkno) AS i
ORDER BY i.itemoffset
LIMIT 10;
```
观察 itemoffset 1 的 `data`。
再看右兄弟的前几个 item：
```sql
WITH leaf AS (
  SELECT blkno, btpo_next
  FROM generate_series(1, pg_relation_size('bt_hikey_demo_pkey') / 8192 - 1) AS g(blkno)
  CROSS JOIN LATERAL bt_page_stats('bt_hikey_demo_pkey', g.blkno) AS s
  WHERE type = 'l' AND btpo_next <> 0
  ORDER BY blkno
  LIMIT 1
)
SELECT i.*
FROM leaf
CROSS JOIN LATERAL bt_page_items('bt_hikey_demo_pkey', leaf.btpo_next) AS i
ORDER BY i.itemoffset
LIMIT 10;
```
你会看到 high key 和右页第一个普通 item 的关系。
在简单 bigint index 上，这比较直观。
在多列、DESC、NULL、重复 key 或 suffix truncation 场景下，不要用肉眼按文本字段做完整比较。
源码里的 `_bt_compare()` 才是语义。
这个实验的诊断价值是：
确认 offset、rightlink、rightmost 这些结构事实。
不要把 pageinspect 的 `data text` 当作完整 comparator。
## 35. 常见误区
误区一：
high key 是页面最大 key。
更准确地说，high key 是页面上界 pivot。
它可能被 suffix truncation。
rightmost page 没有物理 high key。
误区二：
父页 downlink 必须实时覆盖所有 child page。
实际上 parent downlink 可以短暂缺少新 right page。
reader 可以通过 left page high key 和 rightlink 到达 right page。
writer 用 `BTP_INCOMPLETE_SPLIT` 保证最终补齐。
误区三：
incomplete split 是读错误。
不是。
它是 writer 结构修复未完成。
reader 通常不关心这个 flag。
误区四：
rightlink 只用于 range scan。
rightlink 同时用于 forward scan 和 search recovery。
本节的主问题是后者。
误区五：
leftlink 可以像 rightlink 一样修复 stale parent descent。
不能。
stale descent 由于 split right 只会太靠左。
修复方向是向右。
leftlink 主要服务 backward scan 和双向 sibling consistency。
误区六：
`btpo_next = 0` 是 block 0。
在 nbtree 里 `P_NONE` 是 0。
block 0 是 metapage。
普通 sibling link 中 `btpo_next = P_NONE` 表示 rightmost。
不要把它当作指向 metapage 的普通 rightlink。
误区七：
pageinspect offset 1 永远是 high key。
只有非 rightmost page 才是。
rightmost page 的 offset 1 是 data item。
误区八：
`_bt_binsrch()` 能判断页面是否全局正确。
不能。
它只在当前 page 内二分。
跨页修正由 `_bt_moveright()` 完成。
误区九：
WAL redo 必须完整重建 primary 的锁耦合。
不需要。
redo 要保证每个原子状态对 reader 可搜索。
primary 的额外锁耦合主要防止并发 writer 同时修复同一个 incomplete split。
## 36. 源码练习 1：画出一次 split 的状态表
读 `nbtinsert.c:1462-2075`。
把一次 leaf split 画成四个阶段。
阶段一：
原 page 尚未修改。
临时 leftpage 已有新 high key。
right buffer 可能还没分配。
这个阶段普通 ERROR 没有持久后果。
阶段二：
right buffer 已分配。
left/right 临时页都填好了。
旧右兄弟如果存在也被锁住。
尚未进入 critical section。
阶段三：
critical section 内。
原 page 被复制成 left。
right buffer 被复制成 right。
旧右兄弟的 `btpo_prev` 被更新。
WAL record 记录 split。
left page 带 `BTP_INCOMPLETE_SPLIT`。
阶段四：
`_bt_insert_parent()` 插入 right page downlink。
同一动作清 left page incomplete flag。
如果 parent 满，递归向上 split。
画图时一定标出：
```text
old parent downlink -> left/original block
left.btpo_next -> right/new block
right.btpo_prev -> left/original block
right.btpo_next -> old right sibling or P_NONE
```
如果能从图里解释 stale downlink 如何被修正，这个练习才算完成。
## 37. 源码练习 2：跟一次 search 的时间线
读 `nbtsearch.c:100-208`。
用一个三层 B-tree 举例。
给定 key `K`。
时间线写成：
```text
读 fast root
  -> _bt_moveright(root candidate)
  -> _bt_binsrch(root)
  -> 取 child block
  -> 释放 root，锁 child
  -> _bt_moveright(child candidate)
  -> _bt_binsrch(child)
  -> 取 leaf block
  -> 释放 child，锁 leaf
  -> _bt_moveright(leaf candidate)
  -> leaf 内定位
```
然后在每个箭头之间插入一个可能发生的 concurrent split。
回答：
如果 split 发生在 root 下面，哪个 `_bt_moveright()` 会修正？
如果 split 发生在 leaf，哪个 `_bt_moveright()` 会修正？
如果 root split 发生在 `_bt_getroot()` 之后，旧 root 为什么仍然可作为起点？
这个练习的重点是：
不要把 search path 当作一次性可信路径。
它每一层都要重新验证当前 page 的 high key。
## 38. 源码练习 3：比较 read 和 write search
读 `_bt_search()` 的 `access` 参数。
`nbtsearch.c:90-95` 说：
`BT_WRITE` 模式会 finish 遇到的 incomplete split。
`BT_READ` 模式不会。
再读 `nbtsearch.c:193-204`。
当 root page 本身是 leaf，且一开始只拿到 read lock，而 caller 需要 write lock 时，会先解锁再加 write lock。
这中间 leaf 可能 split。
所以加写锁后还要再 `_bt_moveright()`。
练习问题：
为什么不能只把 read lock 原地升级成 write lock 并相信页面没变？
为什么 relock 后必须重新检查 high key？
为什么 writer 模式下 `_bt_moveright()` 要处理 incomplete split？
答案应该回到本节主问题：
只要你释放过页面锁，页面 key range 就可能变化。
重新持锁后必须通过 page-local 状态重新确认。
## 39. 讨论题
讨论题一：
如果没有 high key，只保留 rightlink，搜索能否从 stale downlink 恢复？
提示：
你知道可以向右走，但不知道当前页是否已经太靠左。
没有 page-local 上界，就不能判断是否需要走。
讨论题二：
如果没有 rightlink，只保留 high key，搜索能否恢复？
提示：
你能发现自己太靠左，但不知道下一个候选页在哪里。
只能回到父页或重启搜索。
这会把读路径复杂度和锁策略推回更重的方案。
讨论题三：
为什么 parent downlink 滞后对 reader 可接受，对 writer 不可无限期接受？
提示：
reader 可以沿 sibling chain 到达正确页。
writer 需要 parent 结构最终完整，否则后续 split、deletion 和 root 维护会失去唯一结构边界。
讨论题四：
为什么 `BTP_INCOMPLETE_SPLIT` 标在 left page 更方便？
提示：
沿 rightlink 从 left 到 right 时，正好发现 right 的 downlink 缺失。
讨论题五：
为什么 high key 放 item 1，而不是 page opaque？
提示：
它需要使用 index tuple comparator、suffix truncation、operator class 和 WAL item restore 逻辑。
把它做成普通 pivot tuple 更贴近 B-tree key representation。
## 40. 本节小结
本节唯一主问题是：
父页 downlink 可能因为并发 split 过期时，搜索为什么仍正确。
答案不是“B-tree search 很快”。
答案是一个结构不变量：
```text
split 只向右产生新页；
旧 downlink 仍指向左页；
左页 high key 能判断目标 key 是否已经属于右侧；
左页 rightlink 能把搜索带到右侧；
这个判断可以重复直到 page 覆盖目标 key。
```
`BTPageOpaqueData` 保存 sibling topology、level 和 flags。
`P_HIKEY` 保存 page-local upper bound。
`_bt_search()` 每层都先 `_bt_moveright()`，再在页内 `_bt_binsrch()`。
`_bt_split()` 先让 left/right sibling 和 high key 对 reader 成为正确状态。
`_bt_insert_parent()` 后续补齐 parent downlink，并清 `BTP_INCOMPLETE_SPLIT`。
reader 通常不关心 incomplete split。
writer 插入前必须 finish incomplete split。
WAL redo 保持同样的原子状态边界。
可迁移的系统规律是：
```text
当全局导航结构允许短暂滞后时，局部对象必须携带足够的边界信息和单向修复路径。
局部边界负责发现 stale path，单向 link 负责修复 stale path，后台或后续 writer 再补齐全局结构。
```
这条规律不只适用于 B-tree。
它也解释了很多高并发存储结构为什么会把“可读正确性”和“结构完整性收敛”拆成两个阶段。
下一节可以在这个基础上继续读 `_bt_search()` 的 scankey、`nextkey`、backward scan 和并发页面移动细节。
