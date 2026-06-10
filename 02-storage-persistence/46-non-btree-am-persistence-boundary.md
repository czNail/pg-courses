# PostgreSQL 非 B-tree 索引 AM 的持久化边界
## 课程定位
本节主题：非 B-tree 索引 AM 的持久化边界。
前面几节已经讲过 TableAM/IndexAM 契约、build/insert 路径、scan/recheck、index vacuum callback。
也已经用 B-tree 解释过 page split、high key、rightlink、dedup、bottom-up deletion 和 WAL redo。
现在把视角从 B-tree 移到 GiST、GIN、BRIN、hash、SP-GiST。
本节不是五种 AM 的百科全书。
本节只问一个主问题。
本节唯一主问题：
为什么 PostgreSQL 能让不同索引 AM 共享 `IndexAmRoutine` 契约，但把各自的页面结构、WAL redo、pending/summarization/bucket/split 等持久化细节留在 AM 内部？
本节围绕的核心矛盾：
executor、planner、VACUUM、DDL、WAL 框架需要一个稳定的通用入口。
但是每个索引 AM 持久化的状态机完全不同。
GiST 有 parent/child downlink、`F_FOLLOW_RIGHT`、NSN 和 incomplete split repair。
GIN 有 entry tree、posting list/posting tree、fastupdate pending list 和 list page cleanup。
BRIN 没有 per-tuple TID entry，只有 page range summary、revmap 和 unsummarized range。
hash 有 metapage、bucket、overflow chain、split flags、moved-by-split tuple 和 split cleanup。
SP-GiST 有 inner tuple、leaf tuple chain、redirect/dead/placeholder tuple 和 last-used-page cache。
如果 core 把这些状态抽象成统一页面格式，接口会变成最低公分母，性能和正确性都被拖垮。
如果 core 完全不了解索引，只让每个 AM 自己接 executor/VACUUM/WAL，系统边界会失控。
PostgreSQL 的折中是：
`IndexAmRoutine` 统一的是能力、调用时机、返回语义和资源收口。
AM 私有代码统一不了的是页内状态机、结构变化、redo record 语义和 fallback。
读完本节，你应该能独立判断：
- 哪些语义属于 IndexAM 公共契约。
- 哪些语义只能留在具体 AM 内部。
- 为什么 GIN/BRIN 可以没有 `amgettuple`，但仍然是合法 IndexAM。
- 为什么 WAL resource manager 要给 GiST/GIN/hash/SP-GiST/BRIN 各自的 redo。
- 为什么 pending list cleanup、BRIN summarization、hash split cleanup、SP-GiST redirect cleanup 不是 generic VACUUM 能统一完成的工作。
- 为什么观察一个非 B-tree 索引膨胀或查询变慢时，必须先判断慢在公共契约层还是 AM 私有持久化层。
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
本节行号来自：
```text
nl -ba <source-file>
```
本节重点文件：
```text
src/include/access/amapi.h
src/include/access/relscan.h
src/include/access/genam.h
src/backend/access/index/indexam.c
src/backend/access/index/genam.c
src/include/access/rmgrlist.h
src/backend/access/gist/*
src/backend/access/gin/*
src/backend/access/brin/*
src/backend/access/hash/*
src/backend/access/spgist/*
src/include/access/gist*.h
src/include/access/gin*.h
src/include/access/brin*.h
src/include/access/hash*.h
src/include/access/spg*.h
```
辅助定位：
```text
src/backend/access/rmgrdesc/gistdesc.c
src/backend/access/rmgrdesc/gindesc.c
src/backend/access/rmgrdesc/brindesc.c
src/backend/access/rmgrdesc/hashdesc.c
src/backend/access/rmgrdesc/spgdesc.c
src/backend/commands/vacuum.c
src/backend/executor/nodeIndexscan.c
src/backend/executor/nodeBitmapIndexscan.c
src/backend/executor/nodeBitmapHeapscan.c
```
辅助文件只用于确认调用边界。
它们不是本节展开对象。
## 1. 本节在总主线中的位置
第 42 节说明了 TableAM 与 IndexAM 之间只共享定位候选，而不共享 tuple visibility。
第 43 节说明了 build path 和 insert path 通过同一个 `IndexAmRoutine` 接进 core。
第 44 节说明了 index scan 和 bitmap recheck 如何把 AM 的 lossy/approximate 判断交给 executor。
第 45 节说明了 heap VACUUM 如何用 dead TID callback 驱动不同 index AM 删除 entry。
这些课程已经形成一个稳定结论：
IndexAM 公共层不拥有 tuple 可见性，也不拥有每个索引的页结构。
本节把这个结论推进一步。
问题不再是 heap 与 index 的边界。
问题是 generic IndexAM 与每个非 B-tree AM 内部持久化状态机的边界。
B-tree 容易让人误以为所有 index AM 都是“树页 + leaf tuple + split redo”的变体。
这个误解在 PostgreSQL 源码里很危险。
GIN 的一个 key 可能对应一段 compressed posting list，也可能对应 posting tree。
GIN 的 pending list entry 又临时使用普通 `IndexTuple.t_tid` 存单个 heap TID。
BRIN 的 index tuple 代表一段 heap pages，不代表一个 heap tuple。
hash 的 scan 只需要一个 bucket，但 bucket split 可能让读者同时看 old/new bucket。
SP-GiST 的 search path 可能遇到 redirect tuple，然后跳到另一个 page/offset。
GiST 的 search path 需要用 parent LSN、child NSN 和 `F_FOLLOW_RIGHT` 判断是否跟 rightlink。
这些状态都是真实持久化状态。
它们会写入 index relation 主 fork。
它们会写 WAL。
它们会参与 crash recovery。
它们也会影响普通查询延迟、VACUUM 成本和 standby conflict。
但是 generic `indexam.c` 基本不理解这些状态。
这不是缺陷。
这是边界。
本节的阅读目标是把这条边界画清楚：
core 统一“什么时候调用 AM”。
AM 决定“这个调用如何改变自己的持久化结构”。
WAL 框架统一“redo 入口如何分派”。
AM redo 决定“如何重放自己的页面状态机”。
VACUUM 统一“dead TID callback 和 cleanup 阶段”。
AM 决定“哪些 page、tuple、summary、redirect、bucket flag、pending list 需要清理”。
## 2. 核心矛盾与一句话运行模型
核心矛盾可以压缩成一句话：
PostgreSQL 需要一个稳定的 IndexAM 公共契约来连接 planner/executor/VACUUM/WAL，但 crash-safe 的索引状态变化只能由拥有页格式的 AM 自己描述和重放。
这不是“抽象层次越高越好”的问题。
IndexAM 公共契约必须很窄。
窄到足以承接 executor 的 scan、DML 的 insert、DDL 的 build、VACUUM 的 deletion 和 planner 的 cost。
也必须窄到不需要知道 `GinMetaPageData.head`、`HashMetaPageData.hashm_maxbucket`、`SpGistDeadTupleData.xid` 或 `BrinMetaPageData.lastRevmapPage`。
但这个契约不能窄到失去语义。
它仍然要表达：
是否支持 `amgettuple`。
是否支持 `amgetbitmap`。
是否可以返回 index-only scan data。
是否需要 core 做 predicate lock。
是否是 summarizing AM。
是否支持 parallel build 或 parallel vacuum。
是否有 `aminsertcleanup`。
是否使用 maintenance memory。
是否支持 unique。
这些是 core 需要知道的能力。
页面如何组织不是 core 需要知道的能力。
本节一句话运行模型：
```text
core 用 IndexAmRoutine 调用 ambuild/aminsert/amgettuple/amgetbitmap/ambulkdelete/amvacuumcleanup -> AM 在自己的页格式和锁/WAL规则下推进状态 -> WAL rmgr 根据 AM 私有 record redo 页面 -> generic 层只接收候选 TID、bitmap、stats 或 cleanup 结果。
```
这条模型有两个方向。
前台方向是 core 调 AM。
executor 或 DML 给 AM 一个通用请求。
AM 修改或读取自己的持久化结构。
返回的是 TID、bitmap、bool、stats 或 nothing。
恢复方向是 WAL 调 AM redo。
startup process 不调用 `index_insert()`。
startup process 根据 `RM_GIST_ID`、`RM_GIN_ID`、`RM_HASH_ID`、`RM_SPGIST_ID`、`RM_BRIN_ID` 调对应 redo。
redo 读的是 AM 私有 WAL record。
它不经过 `IndexAmRoutine`。
这正是持久化边界的关键证据。
共享契约服务运行时入口。
私有 redo 服务 crash recovery。
两者不重合。
## 3. 核心文件分工与阅读顺序
阅读顺序不要按目录名排序。
先读公共合同，再读 dispatch，再读 AM handler，再读页状态，最后读 redo。
第一步读 `src/include/access/amapi.h`。
`amapi.h:112-142` 定义 build、insert、insert cleanup、bulk delete、vacuum cleanup 回调类型。
`amapi.h:186-207` 定义 scan 回调。
`amapi.h:230-326` 定义 `IndexAmRoutine`。
注意注释说明 AM 通常静态分配这个结构，core 不复制也不释放。
这说明它是能力表，不是 scan 生命周期对象。
第二步读 `src/include/access/relscan.h`。
`relscan.h:146-205` 定义 `IndexScanDescData`。
它的 `heapRelation`、`indexRelation`、`xs_snapshot`、`keyData` 是公共 scan 输入。
它的 `kill_prior_tuple`、`ignore_killed_tuples`、`xactStartedInRecovery` 是 generic 层和 AM 之间的 hint/安全边界。
它的 `opaque` 是 AM 私有 scan state。
它的 `xs_heaptid`、`xs_heap_continue`、`xs_recheck` 是 AM 向 generic/executor 返回候选的通道。
第三步读 `src/backend/access/index/genam.c`。
`genam.c:79-130` 的 `RelationGetIndexScan()` 初始化公共 scan descriptor。
它把 `ignore_killed_tuples` 设成不在 recovery 中才 true。
注释解释 primary 的 killed hint 不能直接用于 standby MVCC。
这个细节提醒你：
公共 scan state 也有 correctness 语义。
但它仍然不是 AM 页面语义。
第四步读 `src/backend/access/index/indexam.c`。
`index_insert()` 在 `indexam.c:213-234` 检查 `aminsert` 并转发。
`index_beginscan()` 在 `indexam.c:257-291` 建 scan，并准备 table fetch。
`index_beginscan_bitmap()` 在 `indexam.c:300-319` 建 bitmap scan，但不准备 table fetch。
`index_getnext_tid()` 在 `indexam.c:599-635` 调 `amgettuple`，要求 AM 把 TID 放进 `scan->xs_heaptid`。
`index_fetch_heap()` 在 `indexam.c:657-679` 回到 TableAM fetch visible tuple，并只把 `all_dead` 作为 kill hint 回传。
`index_getbitmap()` 在 `indexam.c:743-760` 调 `amgetbitmap` 并统计 TID 数。
`index_bulk_delete()` 和 `index_vacuum_cleanup()` 在 `indexam.c:772-803` 只是调用 AM 的 VACUUM 回调。
这一层没有 `GinPageOpaqueData`、`HashPageOpaqueData`、`BrinTuple`、`SpGistLeafTuple` 或 `GistPageOpaque`。
第五步读各 AM handler。
GiST handler 在 `src/backend/access/gist/gist.c:58-118`。
GIN handler 在 `src/backend/access/gin/ginutil.c:38-96`。
BRIN handler 在 `src/backend/access/brin/brin.c:253-313`。
hash handler 在 `src/backend/access/hash/hash.c:69-129`。
SP-GiST handler 在 `src/backend/access/spgist/spgutils.c:43-103`。
handler 是本节最重要的对照表。
它说明这些 AM 都能被 core 当作 IndexAM。
也说明它们暴露的能力并不相同。
GIN 和 BRIN 的 `.amgettuple = NULL`。
GIN 和 BRIN 只通过 bitmap 进入 executor。
BRIN 的 `.amsummarizing = true`。
hash 的 `.amcanhash = true`、`.amcanmulticol = false`、`.amkeytype = INT4OID`。
GiST 和 SP-GiST 的 `.amcanorderbyop = true`，但它们不是 B-tree ordering。
这些能力位是公共合同。
它们不是页面格式。
第六步读每个 AM 的私有头文件。
GiST 读 `src/include/access/gist.h` 和 `gist_private.h`。
GIN 读 `ginblock.h`、`gin_private.h`、`ginxlog.h`。
BRIN 读 `brin.h`、`brin_page.h`、`brin_revmap.h`、`brin_tuple.h`、`brin_internal.h`。
hash 读 `hash.h`、`hash_xlog.h`。
SP-GiST 读 `spgist_private.h`、`spgxlog.h`。
这里要找的不是所有字段。
只找三个问题：
这个 AM 的持久化单位是什么？
哪些字段能在 crash 后继续解释状态？
哪些字段必须由 redo 恢复？
第七步读 redo 注册。
`src/include/access/rmgrlist.h:40-45` 把 hash、GIN、GiST、SP-GiST、BRIN 注册到不同 WAL resource manager。
这说明 WAL 层也没有统一的 “index redo”。
它统一的是 rmgr dispatch。
具体 redo 仍由 AM 自己实现。
第八步读 redo 文件。
GiST 读 `src/backend/access/gist/gistxlog.c`。
GIN 读 `src/backend/access/gin/ginxlog.c`。
BRIN 读 `src/backend/access/brin/brin_xlog.c`。
hash 读 `src/backend/access/hash/hash_xlog.c`。
SP-GiST 读 `src/backend/access/spgist/spgxlog.c`。
读 redo 文件不要从头背到尾。
直接看 `*_redo()` switch。
再回到 record header 看每个 case 的 payload。
最后找 `XLogReadBufferForRedo()`、`XLogInitBufferForRedo()`、`ResolveRecoveryConflictWithSnapshot()`、`MarkBufferDirty()` 和 `PageSetLSN()`。
第九步读每个 AM 的一个前台状态推进点。
GiST 读 `gistplacetopage()`、`gistfixsplit()`、`gistfinishsplit()`。
GIN 读 `gininsert()`、`ginHeapTupleFastInsert()`、`ginInsertCleanup()`、`gingetbitmap()`。
BRIN 读 `brininsert()`、`brininsertcleanup()`、`brinvacuumcleanup()`、`brinsummarize()`。
hash 读 `_hash_doinsert()`、`_hash_expandtable()`、`hashbulkdelete()`、`hashbucketcleanup()`。
SP-GiST 读 `spginsert()`、`spgdoinsert()`、`spgPageIndexMultiDelete()`、`vacuumRedirectAndPlaceholder()`、`spgWalk()`。
这一步只为回答本节主问题。
不要把它扩展成每个 AM 的完整算法课。
## 4. 关键数据结构与状态
### 4.1 `IndexAmRoutine` 是公共能力表，不是页格式
`IndexAmRoutine` 的上半部分是能力位。
这些字段回答 core 的问题：
这个 AM 能不能排序？
能不能唯一？
能不能多列？
能不能 bitmap scan？
能不能 index-only scan？
能不能 parallel build？
是不是 summarizing AM？
这些问题属于 planner、executor、DDL、VACUUM 的公共决策。
它们不回答“页面上有什么”。
例如 BRIN 的 `amsummarizing = true` 告诉 core 和周边工具这是 summarizing AM。
但它不告诉 core revmap 怎么定位 summary tuple。
例如 GIN 的 `amusemaintenanceworkmem = true` 告诉 maintenance path 这个 AM 可能使用维护内存。
但它不告诉 core pending list 怎么 flush。
例如 hash 的 `amcanhash = true` 告诉 planner/hash semantics 相关能力。
但它不告诉 core bucket split 怎么完成。
`IndexAmRoutine` 的下半部分是 callback。
callback 回答“什么时候进入 AM”。
`ambuild` 是 create/reindex build 入口。
`aminsert` 是每个 heap TID 产生 index entry 的入口。
`aminsertcleanup` 是一条 command 中所有 insert 后的收口入口。
`ambeginscan`/`amrescan`/`amgettuple`/`amgetbitmap`/`amendscan` 是 scan 入口。
`ambulkdelete`/`amvacuumcleanup` 是 VACUUM 入口。
这些入口让 core 能调 AM。
它们不让 core 理解 AM。
### 4.2 `IndexScanDescData` 是 scan 运行时边界
`IndexScanDescData` 由 generic 层和 AM 共同使用。
其中公共字段保存 relation、snapshot、scan key、orderby key、instrumentation。
`opaque` 指向 AM 私有 scan state。
GiST 的 `opaque` 是 `GISTScanOpaqueData`。
GIN 的 `opaque` 是 `GinScanOpaqueData`。
hash 的 `opaque` 是 `HashScanOpaqueData`。
SP-GiST 的 `opaque` 是 `SpGistScanOpaqueData`。
BRIN 的 `opaque` 是 BRIN scan opaque。
generic 层不解引用这些私有结构。
AM 自己在 `ambeginscan` 中分配。
AM 自己在 `amendscan` 中释放。
`xs_heaptid` 是 amgettuple path 的候选 TID 输出。
GIN/BRIN 没有 `amgettuple`，所以不使用这条路径。
`xs_recheck` 是 executor recheck 边界。
GiST/SP-GiST/GIN/BRIN 都可能因为 lossy 或 opclass 判断不完整而要求 recheck。
hash 通常是 equality hash lookup，但仍然通过同一 scan descriptor 返回 heap TID。
`kill_prior_tuple` 是 generic 层给 AM 的 hint。
它来自 TableAM fetch 后的 `all_dead`。
这不是“AM 自己判断 heap tuple dead”。
它是 TableAM 判断后的回传。
`xactStartedInRecovery` 阻止 standby 上使用 primary killed hint。
这说明公共 scan state 只承担跨 AM 共同安全约束。
它不会变成某个 AM 的页状态机。
### 4.3 `IndexBulkDeleteResult` 是 VACUUM 统计边界
`genam.h:52-62` 的 `IndexVacuumInfo` 是 VACUUM 给 AM 的公共输入。
它包含 index relation、heap relation、analyze-only 标记、progress 标记、heap tuple 估计和 buffer strategy。
`genam.h:83-88` 的 `IndexBulkDeleteResult` 是公共统计头。
注释说明 AM 可以返回一个更大的结构，把公共 struct 放在开头。
这正是边界设计：
VACUUM core 需要公共统计。
AM 可能还要在 `ambulkdelete` 和 `amvacuumcleanup` 之间传私有状态。
公共 struct 不能预定义所有 AM 的 vacuum 内部状态。
BRIN 的 `brinbulkdelete()` 在 `brin.c:1307-1316` 基本只分配/返回 stats。
原因是 BRIN 没有 per-heap-tuple index entry。
hash 的 `hashbulkdelete()` 需要扫描 bucket，处理 dead TID callback，还要处理 split cleanup。
GIN 的 `ginbulkdelete()` 首次调用时会先 cleanup pending inserts。
SP-GiST 的 cleanup 还可能处理 redirect/placeholder。
同一个 `ambulkdelete` 合同承接了完全不同的私有工作。
这就是本节主问题的核心样本。
### 4.4 GiST 持久化状态：downlink、rightlink、NSN、follow-right
GiST 的页面结构不是 B-tree 页面。
它的内部 tuple 通常是 child page 的 bounding predicate。
`gist_private.h:264-284` 说明早期 GiST 曾用 invalid tuple 修复 incomplete split。
当前实现用 left half 的 `F_FOLLOW_RIGHT` 标记 incomplete split。
`gist/README:275-284` 解释 scan 如何用 `F_FOLLOW_RIGHT`、child NSN 和 parent LSN 判断是否跟 rightlink。
这套状态不是 IndexAM 公共合同。
它是 GiST 自己的结构变化协议。
`gist.c:705-720` 中 insert descent 看到 `GistFollowRight()` 会调用 `gistfixsplit()`。
`gist.c:1200-1247` 的 `gistfixsplit()` 沿 rightlink 读 split chain，形成 downlink，再由 `gistfinishsplit()` 插入父页。
这个 fallback 是 crash-safety 与并发 scan 共同塑造出来的。
generic `index_insert()` 只调用 `gistinsert()`。
它不知道 incomplete split。
GiST redo 也只在 `gist_redo()` 内处理 GiST record。
`gistxlog.h:20-29` 定义 `XLOG_GIST_PAGE_UPDATE`、`XLOG_GIST_DELETE`、`XLOG_GIST_PAGE_REUSE`、`XLOG_GIST_PAGE_SPLIT`、`XLOG_GIST_PAGE_DELETE`。
`gistxlog.c:394-430` 的 `gist_redo()` 按这些 record 分派。
这说明 GiST 的持久化语言属于 GiST。
### 4.5 GIN 持久化状态：entry tree、posting、pending list
GIN 是 inverted index。
`gin/README:95-105` 说明 GIN 包含 metapage、key entry tree、posting tree，以及可选 fastupdate pending list。
一个 key entry 可能有内联 posting list。
太大时会指向 posting tree。
fastupdate 打开时，插入先进入 pending list。
`ginblock.h:55-101` 的 `GinMetaPageData` 保存 pending list head/tail、tail free size、pending page/heap tuple 计数，以及 planner stats。
这些字段是持久化状态。
`ginblock.h:30-49` 的 `GinPageOpaqueData` 通过 flags 区分 data page、leaf page、deleted page、meta page、list page、incomplete split、compressed page。
`gin/README:201-216` 特别说明 pending list entry 使用 `IndexTuple.t_tid` 存一个 heap item pointer，并要求同一个 heap tuple 的 pending entries 连续。
这与普通 GIN entry tuple 的 posting list/posting tree 语义不同。
generic IndexAM 不可能把这些都抽象成统一 tuple。
GIN handler 的 `.amgettuple = NULL`。
`gingetbitmap()` 在 `ginget.c:1930-1960` 先扫 pending list，再扫主 index。
注释说明如果 pending item 被并发 post 到主 index，bitmap 会重复设同一个 bit，这是可接受的。
同一段注释也说明这正是 GIN 不能支持 `amgettuple` 的重要原因。
`ginfast.c:766-782` 的 `ginInsertCleanup()` 把 pending pages 移进 regular GIN structure。
注释说明如果 crash 发生在 posting entries 已进入主 index、但 pending list 尚未删除之后，redo 后重复 posting 也不会坏。
这是一种 AM 私有 idempotence/fallback 设计。
core 只知道有 `aminsert` 和 `amvacuumcleanup`。
它不知道“先 pending、后 merge、允许 bitmap 去重”。
GIN redo record 也覆盖这些私有状态。
`ginxlog.h:19-210` 定义 create posting tree、insert、split、vacuum page、delete page、update meta page、insert list page、delete list page。
`ginxlog.c:723-770` 的 `gin_redo()` 按这些 record 分派。
这不是 generic index redo。
这是 GIN 自己的持久化语言。
### 4.6 BRIN 持久化状态：range summary、revmap、unsummarized range
BRIN 是 summarizing AM。
`brin/README:6-13` 说明 BRIN 记录连续 heap page group 的 summary values。
`brin/README:19-23` 明确说 BRIN 不在 index 中存 item pointers，所以不能支持 `amgettuple`，只提供 lossy `amgetbitmap`。
`brin_page.h:64-70` 的 `BrinMetaPageData` 保存 `pagesPerRange` 和 `lastRevmapPage`。
`brin_page.h:77-95` 说明 revmap page 存 fixed-size `ItemPointerData` 数组。
这个 TID 不是 heap TID。
它是 summary tuple 在 BRIN regular page 中的位置。
`brin/README:76-90` 说明 revmap 每个 page range 存一个 TID，指向该 range 的 summary tuple。
如果 new heap tuple 超出 summary，BRIN 更新 summary tuple。
如果新 tuple 让 summary 变大且当前页放不下，BRIN 可以插入新的 summary tuple，并更新 revmap 指向它。
`brin/README:92-101` 说明 revmap entry 为 invalid TID 时，range 被视为 unsummarized。
scan 时 unsummarized range 也要进入 TIDBitmap。
这是 correctness fallback：
没有 summary 不等于可以跳过。
没有 summary 等于必须把这个 heap range 交给 bitmap heap scan recheck。
BRIN insert path 在 `brin.c:349-511`。
它根据 heap TID 算 page range。
如果 range 未 summarised，直接 nothing to do。
如果 summary 存在，调用 opclass `addValue` 更新 in-memory tuple。
需要变化时调用 `brin_doupdate()`。
失败时回到 loop 顶部重读 revmap，因为并发更新可能让 revmap 指向了别的 tuple。
`brininsertcleanup()` 在 `brin.c:517-534` 释放 command-lifetime `BrinInsertState` 和 revmap access。
BRIN vacuum 在 `brin.c:1318-1347` 不是删除 per-tuple index entry。
它调用 `brin_vacuum_scan()` 和 `brinsummarize()`，把 unsummarized ranges 尝试补 summary。
`brin_summarize_new_values()`、`brin_summarize_range()`、`brin_desummarize_range()` 是 SQL callable AM 控制入口。
它们需要检查 recovery、lock table before index、检查 AM 类型和权限。
这些入口不是 `IndexAmRoutine` 公共接口。
但它们确实改变 BRIN 持久化状态。
这说明 AM 可以有公共 IndexAM 之外的 AM-specific maintenance surface。
BRIN redo record 也围绕这些状态。
`brin_xlog.h:31-43` 定义 create index、insert、update、samepage update、revmap extend、desummarize。
`brin_xlog.c:308-336` 的 `brin_redo()` 按 `XLOG_BRIN_OPMASK` 分派。
redo 会更新 regular page 和 revmap page。
它不会调用 generic `index_insert()`。
### 4.7 hash 持久化状态：metapage、bucket、overflow、split cleanup
hash AM 的核心持久化对象是 bucket。
`hash.h:244-265` 的 `HashMetaPageData` 保存 bucket 数、mask、splitpoint、overflow bitmap 信息和 hash function OID。
`hash.h:77-84` 的 `HashPageOpaqueData` 保存 prev/next block、bucket number、page flags 和 page id。
`hash.h:53-61` 定义 page flags。
其中 `LH_BUCKET_BEING_POPULATED`、`LH_BUCKET_BEING_SPLIT`、`LH_BUCKET_NEEDS_SPLIT_CLEANUP`、`LH_PAGE_HAS_DEAD_TUPLES` 都是持久化状态机的一部分。
`hash/README:216-234` 解释 split flags。
old bucket 有 being-split。
new bucket 有 being-populated。
split-cleanup 表示 old bucket 仍有复制到 new bucket 的 tuple。
tuple 上的 moved-by-split 标记会永久保留。
这是 hash 的并发 scan 协议。
generic IndexAM 不知道 bucket，也不知道 moved-by-split。
hash scan 在 `hashsearch.c:305-314` 明确不支持 whole-index scan。
原因是 hash scan 要读一个 bucket，而全索引扫描无法实用地锁住所有 bucket 以抵抗 split/compaction。
`hashsearch.c:359-407` 说明 scan 如果从 being-populated bucket 开始，需要 pin old bucket，避免 vacuum split-cleanup 删除扫描还需要的 tuple。
`hashsearch.c:626-633` 和 `hashsearch.c:671-679` 会跳过 moved-by-split tuple 和 killed tuple。
hash insert 在 `_hash_doinsert()`。
`hashinsert.c:103-126` 中，如果目标 bucket 正在 split 且能拿 cleanup 条件，就先 `_hash_finish_split()`，再 retry insert。
`hashinsert.c:128-191` 在 bucket chain 中寻找空间，必要时清 dead tuple 或加 overflow page。
`hashinsert.c:193-254` 插入 tuple、更新 metapage tuple count，并在需要时 `_hash_expandtable()`。
`hash/README:376-383` 说明 split 中断不会破坏索引，后续 insert 会重试完成 split。
`hash/README:596-613` 说明 crash 后 old/new bucket flags 表示 split in progress，reader algorithm 仍然正确；完成 split 放在后续 insert/split，而不是 search 或 VACUUM。
这是典型 AM 私有 fallback。
hash redo record 也围绕 bucket 状态。
`hash_xlog.h:27-46` 定义 init meta、init bitmap、insert、add overflow、split allocate、split page、split complete、move contents、squeeze、delete、split cleanup、update meta、vacuum one page。
`hash_xlog.c:1062-1111` 的 `hash_redo()` 按这些 record 分派。
hash split 是多 record 状态机。
generic WAL 不可能用一个“index split record”覆盖它。
### 4.8 SP-GiST 持久化状态：inner/leaf/dead tuple 与 redirect
SP-GiST 的页结构与 B-tree/GiST/GIN/hash 都不同。
`spgist_private.h:46-50` 定义 metapage、normal root、null root 的固定 block。
`spgist_private.h:60-67` 的 page opaque 只保存 flags、redirection count、placeholder count 和 page id。
`spgist_private.h:271-275` 定义 tuple state：
`SPGIST_LIVE`、`SPGIST_REDIRECT`、`SPGIST_DEAD`、`SPGIST_PLACEHOLDER`。
`spgist_private.h:385-394` 的 leaf tuple 保存 `heapPtr`。
`spgist_private.h:429-436` 的 dead tuple 用同一位置保存 redirect pointer 和 xid。
`spgist/README:258-266` 说明当 insertion 改变 downlink 指向的位置时，会在旧位置放 redirect tuple。
已经从父页出发、正在路上的 scan 可以沿 redirect 找到新位置。
等所有可能看到 redirect 的事务结束后，VACUUM 可以把 redirect 变成 placeholder。
`spgist/README:271-306` 解释 live/redirect/dead/placeholder 的含义。
这套状态是 SP-GiST 的并发和持久化核心。
generic IndexAM 只看到 `spggettuple` 或 `spggetbitmap` 返回 TID/bitmap。
`spginsert.c:183-213` 建临时 context，初始化 `SpGistState`，循环调用 `spgdoinsert()`，冲突时 reset context 并重试。
`spgdoinsert.c:120-177` 的 `spgPageIndexMultiDelete()` 会把删除的 tuple 替换为 redirect/dead/placeholder。
注释特别说明它也用于 WAL replay，不能变得太 smart。
`spgdoinsert.c:499-510` 在移动 leaf tuple 时，旧 chain head 留 redirect，parent downlink 指向新位置。
`spgscan.c:768-801` 的 leaf scan 遇到 redirect 会把 search item 的 pointer 改到 redirect 目标。
`spgscan.c:895-904` 的 inner scan 遇到 redirect 也会跳转。
`spgvacuum.c:485-615` 清理 redirect/placeholder。
它用 `GlobalVisTestIsRemovableXid()` 判断 redirect 是否足够旧。
然后把 redirect 变 placeholder，或者删除末尾 placeholder，写 `XLOG_SPGIST_VACUUM_REDIRECT`。
SP-GiST redo record 覆盖 add leaf、move leafs、add node、split tuple、picksplit、vacuum leaf/root/redirect。
`spgxlog.c:28-42` 的 `fillFakeState()` 说明 redo 只需要足够形成 dead tuple 的 fake state。
`spgxlog.c:926-965` 的 `spg_redo()` 按 SP-GiST record 分派。
这再次说明 redo 只依赖 AM 私有 record。
它不依赖 generic `IndexAmRoutine`。
## 5. 主流程源码 walkthrough
本节用一条主流程串起来：
一次 DML 插入产生 index maintenance，随后查询和 VACUUM/redo 如何在公共契约与 AM 私有状态之间来回。
这条主流程不是某一个 AM 的完整算法。
它展示边界如何被反复使用。
### 5.1 入口：executor/DML 只知道 `index_insert()`
DML path 对每个 ready index 形成 values/isnull 和 heap TID。
然后调用 generic `index_insert()`。
`indexam.c:213-234` 做三件事。
第一，检查 relation 是 index 且不是正在 reindex 的目标。
第二，检查 `aminsert` 是否存在。
第三，如果 AM 不自己处理 predicate locks，则 core 做 serializable conflict check。
最后转发给 `indexRelation->rd_indam->aminsert()`。
这里没有 switch on AM。
GiST 进入 `gistinsert()`。
GIN 进入 `gininsert()`。
BRIN 进入 `brininsert()`。
hash 进入 `hashinsert()`。
SP-GiST 进入 `spginsert()`。
公共合同在这里结束。
### 5.2 GiST insert：generic 调用后，incomplete split 由 GiST 自己修复
GiST `gistinsert()` 在 `gist.c:166-198` 初始化 `GISTSTATE`，形成 index tuple，设置 heap TID，然后调用 `gistdoinsert()`。
插入下降时，`gist.c:705-720` 看到 page 上 `F_FOLLOW_RIGHT` 会拿 exclusive lock 并调用 `gistfixsplit()`。
这个 page 可能来自之前 crash 或 ERROR 中断的 split。
`gistfixsplit()` 在 `gist.c:1200-1247` 沿 rightlink 构造 split page 的 downlink，再调用 `gistfinishsplit()`。
`gistfinishsplit()` 在 `gist.c:1354-1425` 从右往左把 downlink 插回父页。
这个 repair 与普通 insert 共享 GiST 内部函数。
generic `index_insert()` 不知道 repair 发生了。
它也不需要知道。
GiST `gistplacetopage()` 的 split 分支在 `gist.c:455-472` 对非最右 split page 设置 `F_FOLLOW_RIGHT` 并复制 old NSN。
`gist.c:483-628` 在 critical section 中写 WAL、设置 LSN、并在插入父 downlink 后清 left child 的 `F_FOLLOW_RIGHT`。
这正是 GiST 的持久化边界：
公共 `aminsert` 只说“插入这个 index tuple”。
GiST 内部决定是否 split、是否暂时 follow-right、何时修复 downlink、如何 redo。
### 5.3 GIN insert：同一个 `aminsert` 可能先写 pending list
GIN `gininsert()` 在 `gininsert.c:865-913` 初始化 `GinState` 和 insert temporary context。
如果 `GinGetUseFastUpdate(index)` 为 true，它先用 `ginHeapTupleFastCollect()` 把一个 heap tuple 拆成多个 pending entries。
然后调用 `ginHeapTupleFastInsert()`。
如果 fastupdate 关闭，则逐 key 调 `ginHeapTupleInsert()` 进入 regular structure。
在 fast path 中，`ginfast.c:219-472` 修改 metapage 和 pending list pages。
它需要维护 `head`、`tail`、`tailFreeSize`、`nPendingPages`、`nPendingHeapTuples`。
它在 critical section 中写 `XLOG_GIN_UPDATE_META_PAGE`。
pending list 太大时，它在退出 critical section 后调用 `ginInsertCleanup()`。
`ginfast.c:448-471` 特别强调 cleanup 可能很慢，不应在 critical section 内调用。
这是一条重要 ownership 边界：
写 pending list 的 WAL 原子性属于当前 insert。
把 pending list merge 到 regular structure 是后续 cleanup 工作。
同一个 `aminsert` 回调内部可以选择立即做 retail insert，也可以选择先写 AM 私有缓冲结构。
core 只关心 `aminsert` 返回。
### 5.4 BRIN insert：同一个 `aminsert` 可能什么也不写
BRIN `brininsert()` 在 `brin.c:349-511` 根据 heap TID 算出 page range start block。
如果 autosummarize 开启且刚进入新 range，它可能向 autovacuum work queue 请求 summarization。
然后它用 revmap 查当前 range 的 summary tuple。
如果 range 未 summarized，`brtup == NULL`，它直接 break。
也就是说 heap tuple 已插入，BRIN `aminsert` 可以不产生任何 index tuple 写入。
这不是漏索引。
这是 BRIN 的语义：
unsummarized range 扫描时必须返回为 lossy bitmap。
如果 summary 存在，BRIN 把新 values 合并进 summary。
如果新 summary 需要写，调用 `brin_doupdate()`。
如果并发导致更新失败，reset 临时 context 并重读 revmap。
generic `index_insert()` 无法也不应该理解这种 retry。
BRIN 的 `aminsertcleanup` 在 `brin.c:517-534` 释放 `BrinInsertState` 和 revmap access。
这说明 BRIN 是本组 AM 里少数实现 `.aminsertcleanup` 的例子。
公共 API 提供 cleanup hook。
是否需要它由 AM 自己决定。
### 5.5 hash insert：bucket split 是 AM 内部慢路径
hash `hashinsert()` 形成 hash tuple 后进入 `_hash_doinsert()`。
`hashinsert.c:65-91` 先读 metapage并锁目标 bucket。
`hashinsert.c:103-126` 如果 bucket 正在 split，且当前 backend 能完成 split，就先 `_hash_finish_split()`，释放 buffer 后跳回 `restart_insert`。
这不是 generic retry。
这是 hash bucket 状态机的 retry。
`hashinsert.c:128-191` 在 bucket chain 中找空间。
如果 page 有 dead tuple 且能拿 cleanup 条件，可以调用 `_hash_vacuum_one_page()` 先清理。
如果没有空间，沿 overflow chain 走，或加 overflow page。
`hashinsert.c:193-254` 在 critical section 里添加 tuple、更新 metapage tuple count、写 `XLOG_HASH_INSERT`，最后可能 `_hash_expandtable()`。
如果 expand 失败或 split 留半完成，hash README 说明后续 insert 会重试。
generic `index_insert()` 仍然只看到 `aminsert` 返回 false。
### 5.6 SP-GiST insert：冲突时 AM 自己循环重试
SP-GiST `spginsert()` 在 `spginsert.c:183-213` 创建 insert temporary context。
它初始化 `SpGistState`。
然后在 while 循环中调用 `spgdoinsert()`。
如果 `spgdoinsert()` 返回 false，说明并发插入或结构变化要求重试。
它 reset context，重新初始化 state，再试。
这条 retry 不暴露给 generic IndexAM。
`spgdoinsert.c:499-510` 移动 leaf tuple 时，旧位置用 redirect 或 placeholder 替换。
构建期间没有并发 scan，可以用 placeholder。
普通插入期间可能有 scan in flight，必须用 redirect。
`spgdoinsert.c:1163-1178` 在 picksplit 等结构变化中同样会先放一个指向“不可能值”的 redirect，再在知道新位置后更新。
`spgdoinsert.c:1278-1283` 和 `1315-1320` 再修正 redirect link。
这是 SP-GiST 的并发可达性协议。
generic `aminsert` 不知道 redirect。
### 5.7 Scan：generic 层统一候选，AM 内部决定枚举方式
普通 index scan path：
```text
executor
  -> index_beginscan()
  -> AM ambeginscan()
  -> index_rescan()
  -> AM amrescan()
  -> index_getnext_slot()
     -> index_getnext_tid()
        -> AM amgettuple()
     -> index_fetch_heap()
        -> table_index_fetch_tuple()
```
bitmap path：
```text
executor
  -> index_beginscan_bitmap()
  -> AM ambeginscan()
  -> index_getbitmap()
     -> AM amgetbitmap()
  -> bitmap heap/table scan
     -> recheck when needed
```
GiST 支持 `amgettuple` 和 `amgetbitmap`。
GiST scan page 时，`gistget.c:360-383` 根据 `F_FOLLOW_RIGHT` 或 parent LSN 与 child NSN 关系，把 right sibling 加入 search queue。
GIN 只支持 `amgetbitmap`。
`ginget.c:1951-1960` 必须先扫 pending list，再扫主 index，避免并发 cleanup 造成 miss。
BRIN 只支持 `amgetbitmap`。
它返回 page ranges 的 lossy bitmap。
hash 支持 `amgettuple` 和 `amgetbitmap`，但 `hashsearch.c:305-314` 禁止 whole-index scan。
SP-GiST 支持两种 scan。
`spgscan.c:768-801` 和 `895-904` 遇到 redirect 会跳转。
generic `index_getnext_tid()` 只要求 `amgettuple` 把 TID 放进 `xs_heaptid`。
generic `index_getbitmap()` 只要求 `amgetbitmap` 把候选放进 `TIDBitmap`。
至于候选如何枚举，属于 AM 私有持久化状态。
### 5.8 VACUUM：公共 callback，私有 cleanup
VACUUM 对所有 index AM 有公共入口：
`index_bulk_delete()` 调 `ambulkdelete`。
`index_vacuum_cleanup()` 调 `amvacuumcleanup`。
但每个 AM 的工作不同。
GiST `gistvacuum.c:373-390` 用 callback 决定 leaf tuple 是否删除。
`gistvacuum.c:393-418` 一页一条 WAL record 批量删除 offsets。
GiST page deletion 还要把 leaf page 标 deleted，并删 parent downlink。
GIN `ginvacuum.c:621-631` 首次 bulkdelete 会 cleanup pending inserts。
随后 vacuum entry page 和 posting tree。
BRIN `brinbulkdelete()` 不需要用 dead TID callback 删除 per-tuple entry。
`brinvacuumcleanup()` 反而 summarize unsummarized ranges。
hash `hashbucketcleanup()` 既可以用 dead TID callback 删除 tuple，也可以删除 split moved tuple。
SP-GiST bulkdelete/vacuum cleanup 既处理 heap TID death，也处理 redirect/placeholder cleanup。
这说明 `ambulkdelete` 的公共语义是“给 AM 一个清理机会和 dead TID 判断入口”。
它不是“所有 AM 都按 heap TID 删除 index tuple”的同构流程。
### 5.9 WAL/redo：恢复路径完全绕开 `IndexAmRoutine`
WAL 插入前台通常在 AM 内部 critical section 中发生。
例如 GiST split 写 `XLOG_GIST_PAGE_SPLIT`。
GIN pending list 写 `XLOG_GIN_UPDATE_META_PAGE` 和 list page records。
BRIN update 写 `XLOG_BRIN_UPDATE` 或 `XLOG_BRIN_SAMEPAGE_UPDATE`。
hash insert/split 写一串 hash-specific records。
SP-GiST tuple move/picksplit/redirect vacuum 写 SP-GiST-specific records。
恢复时，`rmgrlist.h` 把 WAL resource manager 映射到对应 redo 函数。
startup process 读 WAL record 的 rmgr id。
然后调用 `gist_redo()`、`gin_redo()`、`brin_redo()`、`hash_redo()` 或 `spg_redo()`。
这些 redo 函数的 switch 都在各自 AM 文件中。
它们读 AM 私有 payload。
它们操作 AM 私有 page flags、tuple states、metapage fields、revmap entries。
这条路径不调用 `ambuild`、`aminsert`、`amgettuple` 或 `ambulkdelete`。
这就是本节主问题的最终答案：
运行时共享的是 IndexAM callback。
恢复共享的是 WAL rmgr 框架。
持久化细节从来没有被抽象成一个 generic index page protocol。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建公共对象
`IndexAmRoutine` 由 AM handler 返回。
它通常是 static const。
core 不复制也不释放。
`Relation->rd_indam` 持有这个能力表指针。
scan descriptor 由 AM 的 `ambeginscan` 创建。
多数 AM 内部调用 `RelationGetIndexScan()` 初始化公共字段。
`IndexScanDescData.opaque` 由 AM 填入私有状态。
例如 GiST 分配 queue、memory context、GISTSTATE。
GIN 分配 key arrays、entry arrays 和 temp context。
hash 分配 `HashScanOpaqueData`。
SP-GiST 分配 scan queue 和 traversal/temp contexts。
BRIN 初始化 revmap access。
### 6.2 谁持有 buffer pin 和 lock
generic `indexam.c` 只管理 index relation refcount 和 table fetch state。
`index_beginscan_internal()` 在 `indexam.c:339-347` 增加 index relcache refcount，然后调 `ambeginscan`。
`index_endscan()` 在 `indexam.c:399-416` 先结束 table fetch，再调 AM `amendscan`，最后减少 relation refcount。
AM 自己管理 index buffer pin/lock。
GiST scan page 时读 buffer、share lock、检查 page、再 unlock release。
GIN pending scan 要 pin pending head，避免 vacuum 删除。
hash scan 会保留 bucket primary page pin，以及 split old/new bucket pin。
SP-GiST scan 在 walk 中根据 item pointer 切换 buffer。
BRIN scan 持有 revmap/regular page buffer。
这些 pin/lock 的安全性依赖 AM 私有并发协议。
generic 层不能替代。
### 6.3 谁持有 command-lifetime AM cache
一些 AM 使用 `IndexInfo->ii_AmCache`。
GiST `gistinsert()` 首次调用时把 `GISTSTATE` 放在 `indexInfo->ii_Context`。
GIN `gininsert()` 首次调用时把 `GinState` 放在 `ii_AmCache`。
BRIN `brininsert()` 首次调用时创建 `BrinInsertState` 并保存到 `ii_AmCache`。
BRIN 还提供 `brininsertcleanup()`。
`index_insert_cleanup()` 在 `indexam.c:241-249` 会在 AM 提供 hook 时调用它。
BRIN cleanup 会先把 `ii_AmCache` 设为 NULL，再释放 revmap access 和 state。
这个顺序是防 dangling pointer。
GIN 和 GiST 没有 `aminsertcleanup`，它们依赖 `IndexInfo` context 生命周期收口。
SP-GiST 每次 insert 使用 temporary context，不通过 `ii_AmCache` 长期持有。
所以公共 API 提供 cleanup 钩子。
AM 自己选择是否需要 command-level cleanup。
### 6.4 谁释放 AM 私有 scan state
`index_endscan()` 只调用 AM 的 `amendscan`。
AM 私有 `opaque` 由 AM 自己释放。
这很重要。
generic 层知道 `void *opaque`，但不知道里面是否有 pairing heap、TIDBitmap iterator、buffer pin、revmap access、memory context 或 reconstructed tuples。
SP-GiST `spgWalk()` 可能在 scan queue 中保存 reconstructed values 和 traversal state。
GiST ordered scan 用 pairing heap。
GIN scan key 可能有 match bitmap 和 iterators。
BRIN scan opaque 持有 revmap access。
释放顺序必须留在 AM 内部。
### 6.5 ERROR/abort 时谁兜底
buffer pins、locks、relation refs、MemoryContext 和 ResourceOwner 是基础兜底。
但 AM 仍然必须让 on-disk state 在 ERROR/crash 后可继续解释。
GiST split 如果在父 downlink 插入前中断，left page 上的 `F_FOLLOW_RIGHT` 让 scan 能跟 rightlink，后续 insert 能 `gistfixsplit()`。
GIN pending cleanup 如果在 posting 进入 main index 后、删除 pending pages 前 crash，redo/后续 cleanup 可以重复处理而不破坏 correctness。
BRIN concurrent update 失败时回到 loop 顶部重读 revmap。
hash split 中断后留下 being-split/being-populated flags，reader 扫 old/new bucket，后续 insert 完成 split。
SP-GiST 结构移动时用 redirect 保护已经在路上的 scan，VACUUM 后续转 placeholder。
这些不是 ResourceOwner 能解决的资源泄漏问题。
这些是 AM 持久化状态机的 crash/ERROR fallback。
## 7. 正确性机制层次
### 7.1 公共层保证什么
公共 IndexAM 层保证 callback 调用时机。
它保证 executor、DML、VACUUM 和 AM 之间的最小参数合同。
它管理 relation refcount。
它对缺失 callback 报错。
它在必要时做 predicate lock relation/page 级入口。
它把 table fetch 和 snapshot visibility 留给 TableAM。
它维护基本统计，例如 index tuples count 和 heap fetch count。
它不保证某个 AM page split 的结构正确。
它不保证 GIN pending list 不过大。
它不保证 BRIN summary tight。
它不保证 hash bucket split 马上完成。
它不保证 SP-GiST redirect 立即消失。
### 7.2 AM 私有层保证什么
AM 私有层保证自己的 page invariant。
GiST 保证 scan 不漏 split 后右侧页面。
GIN 保证 pending list 与 main index 合起来不漏候选 TID。
BRIN 保证 summary 或 unsummarized fallback 不漏 heap page。
hash 保证 bucket split 期间 scan 能覆盖 old/new bucket 的正确集合。
SP-GiST 保证 downlink 变更期间 scan 能通过 redirect 找到新位置。
AM 私有层还保证自己的 WAL record 足以 redo 页面。
这包括 page flag、metapage counter、revmap entry、tuple offset、rightlink、delete xid、snapshot conflict horizon。
这些字段只有在 AM 的 lifecycle 中才有语义。
单独看 raw field 不够。
例如 hash `LH_BUCKET_NEEDS_SPLIT_CLEANUP` 必须结合 old bucket、new bucket、moved-by-split tuple 和 cleanup lock 解释。
例如 SP-GiST `SPGIST_REDIRECT` 必须结合 redirect xid 和 GlobalVis horizon 解释。
例如 BRIN invalid revmap TID 必须解释为 unsummarized range，而不是 corruption。
### 7.3 WAL-before-data 与 redo idempotence
所有 AM 修改持久化 page 都遵守 WAL-before-data。
前台进入 critical section 后 mark dirty、注册 buffer/data、插入 WAL、设置 LSN。
但每个 AM 的 redo idempotence 不同。
GiST split redo 会重新初始化 split pages，设置 rightlink、NSN、follow-right。
GIN metapage redo 有些地方像 full-page restore，避免 torn page hazard。
BRIN insert/update redo 同时更新 regular page 和 revmap page。
hash redo 需要在 split、overflow、squeeze 中重新建立 bucket chain 和 flags。
SP-GiST redo 用 fake state 构造 dead tuple，再重放 redirect/placeholder 变换。
generic WAL 框架只保证 record 分发、page LSN 检查、FPI、redo 顺序。
具体 record 如何做到 idempotent 是 AM 内部责任。
### 7.4 MVCC 与 recovery conflict
索引 AM 通常不判断 heap tuple visibility。
但索引页面删除或 redirect cleanup 可能影响 standby 上正在运行的查询。
因此某些 WAL record 携带 `snapshotConflictHorizon` 或 delete xid。
GiST `gistxlogDelete` 和 `gistxlogPageReuse` 带 conflict horizon。
hash `xl_hash_vacuum_one_page` 带 `snapshotConflictHorizon`。
SP-GiST `spgxlogVacuumRedirect` 带 `snapshotConflictHorizon`。
GIN/BRIN 的相应路径有自己的冲突或无冲突处理。
这些不是 executor recheck。
这是 recovery correctness。
公共 IndexAM 层不会统一这个字段。
每个 AM 必须知道什么 page state 的删除可能与 standby snapshot 冲突。
### 7.5 Predicate lock 边界
`IndexAmRoutine.ampredlocks` 告诉 core AM 是否自己处理 predicate locks。
GiST、GIN、hash 设置为 true。
BRIN 和 SP-GiST 在当前 handler 中设置为 false。
`index_insert()` 对不自己处理 predicate locks 的 AM 调 `CheckForSerializableConflictIn()`。
`index_beginscan_internal()` 对不自己处理的 AM 调 `PredicateLockRelation()`。
但是 GIN pending list 有更细的 metapage predicate locking。
`ginfast.c:245-250` 说明 pending insert 逻辑上可能属于树中任何位置，因此需要与所有 serializable scans 冲突。
`ginget.c:1851-1855` 扫 pending list 时对 metapage拿 predicate lock。
这说明公共能力位只能决定谁承担 predicate locking。
具体锁粒度仍可能是 AM 私有协议。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 callback 不存在
`indexam.c` 用 `CHECK_REL_PROCEDURE` 和 `CHECK_SCAN_PROCEDURE` 检查 callback。
如果调用了不存在的 callback，就报错。
这不是 runtime fallback。
这是契约保护。
例如 GIN/BRIN 的 `.amgettuple = NULL`。
它们不能用于普通 tuple-at-a-time index scan path。
planner/executor 应该选择 bitmap path。
如果错误地调用 `index_getnext_tid()`，会在 generic 层失败。
### 8.2 GIN pending cleanup 与重复候选
GIN `gingetbitmap()` 先扫 pending list。
如果并发 cleanup 把 pending item post 到 main index，主 scan 可能再次看到它。
bitmap 重复设 bit 不影响 correctness。
因此 GIN 可以允许这种重复。
但这也解释了为什么它不能支持 `amgettuple`。
tuple-at-a-time API 很难表达“可能重复但 bitmap 去重安全”的语义。
这个 fallback 是 GIN 选择 `amgetbitmap` 的核心原因。
### 8.3 BRIN unsummarized range
BRIN revmap invalid TID 不是错误。
它表示 range 未 summarized。
scan 必须把这个 heap range 加入 lossy bitmap。
这会增加 recheck 成本。
但不会漏结果。
BRIN insert 对 unsummarized range 什么也不写。
BRIN vacuum 或 SQL control function 后续可 summarize。
这是 latency vs maintenance 的折中。
### 8.4 hash split 中断
hash split 是多 record、多 page 的状态机。
中断后 old/new bucket flags 保留 split in progress。
scan 根据 flags 扫 old/new bucket。
insert 在进入 old bucket 时尝试完成 split。
VACUUM 清理 split moved tuple 时要拿 cleanup lock，等旧 scan 结束。
如果 split cleanup flag 未清，bucket 不能再次 split。
这用持久化 flags 把 crash/ERROR 后的“半完成结构变化”变成可重试状态。
### 8.5 SP-GiST redirect 老化
SP-GiST redirect 不能立即删除。
因为某个 scan 可能已经从 parent 出发，正在去旧 page/offset 的路上。
redirect 保存新位置。
VACUUM 用 redirect xid 和 global visibility 判断是否可以转 placeholder。
placeholder 在 page 末尾时才能删掉。
如果中间 placeholder 被删，会改变后续 tuple offset，破坏 downlink。
这说明 cleanup 不是“看到 dead 就删”。
它是 offset stability、visibility horizon 和 WAL redo 共同约束的流程。
### 8.6 GiST incomplete split repair
GiST split 期间父 downlink 可能还没插入。
left page 的 `F_FOLLOW_RIGHT` 让 scan 跟 rightlink，不漏结果。
后续 insert 看到 follow-right 后会 `gistfixsplit()`。
这把 crash 后的结构缺口留给 AM 内部修复。
generic `index_insert()` 不需要知道页处于 incomplete split。
## 9. 成本、资源与跨模块传播
### 9.1 CPU 和内存成本
公共 IndexAM 层的 CPU 成本很小。
一次 callback 调用、若干断言/检查、统计计数和 table fetch。
真正成本在 AM 内部。
GiST ordered scan 用 pairing heap 管 search queue。
GIN query 需要 extract query entries、merge posting streams、构建 bitmap。
BRIN scan 主要按 revmap/page range 做 summary consistent 判断。
hash scan 通常只扫一个 bucket，但 overflow chain 和 split 状态会增加成本。
SP-GiST scan 需要 traversal state、redirect jump 和 opclass callbacks。
内存也类似。
`amusemaintenanceworkmem` 只告诉 maintenance path 可能用维护内存。
GIN pending cleanup 实际会用 `work_mem`、`maintenance_work_mem` 或 `autovacuum_work_mem`，取决于调用来源。
GiST/GIN/SP-GiST 都大量使用短生命周期 MemoryContext 避免 opclass callback 泄漏。
BRIN insert 为一个 tuple update 建 temporary context，失败重试时 reset。
### 9.2 WAL 和 IO 放大
非 B-tree AM 的 WAL 放大模式不同。
GiST split 可以一条 record 覆盖多个 split pages 和 child follow-right 清理。
GIN pending insert 可能先写 list pages，后续 cleanup 再写 main structure 和 delete list page records。
BRIN 一个 summary update 可能更新 regular page 和 revmap page。
hash 一个插入可能写 insert record、overflow allocation record、split allocation/move/complete records。
SP-GiST picksplit/move leafs 可能同时改 source、dest、inner、parent pages。
所以分析 WAL 增长不能只看“每条 heap tuple 几条 index entry”。
必须看 AM 私有慢路径是否被触发。
GIN pending list 太大时，前台或 VACUUM 触发 cleanup 会造成一段集中 WAL/IO。
BRIN summarization 会扫描 heap ranges 并写 summary/revmap。
hash overflow 和 split 会放大随机 IO。
SP-GiST redirect cleanup 会在 VACUUM 中产生额外 page update。
### 9.3 锁与 contention
generic `indexam.c` 不持有 AM page locks。
contention 多发生在 AM 内部。
GIN pending insert 需要 metapage lock 来更新 head/tail/counters。
GIN pending cleanup 用 page lock 防止并发 cleanup，但允许并发 pending insertion。
BRIN revmap update 会竞争 revmap page 和 regular page。
hash bucket scan/insert/vacuum 都围绕 bucket primary page cleanup lock 协调。
GiST split repair 和 parent downlink update 会在树路径上锁 page。
SP-GiST redirect/picksplit 可能同时修改 old/new/current/parent pages。
这些 lock 的 wait event 可能表面都像 buffer content lock。
但根因要回到 AM 状态。
### 9.4 跨模块传播
第一个传播到 executor。
AM 返回 `xs_recheck` 或 lossy bitmap，会增加 heap/table recheck 成本。
BRIN 的 unsummarized range 直接扩大 bitmap heap scan 访问范围。
GIN/bitmap 去重可以隐藏重复候选，但 heap recheck 仍然可能很重。
第二个传播到 VACUUM/autovacuum。
GIN pending cleanup 可能在 autovacuum analyze 或 vacuum cleanup 中发生。
BRIN autosummarize 通过 `AutoVacuumRequestWork()` 请求后台工作。
hash split cleanup 需要 VACUUM bucket cleanup。
SP-GiST redirect/placeholder cleanup 也依赖 VACUUM。
第三个传播到 WAL/checkpoint/recovery。
AM 私有 record 增加 WAL bandwidth。
standby replay 必须执行 AM redo。
某些 cleanup record 可能触发 recovery conflict。
第四个传播到 planner。
AM handler 的 capability 和 cost estimate 决定 plan shape。
BRIN 的 lossy nature、GIN 的 pending list、GiST/SP-GiST 的 recheck 都会通过 selectivity/cost 间接影响 plan。
但很多 AM 私有状态不是 planner 可见的即时状态。
### 9.5 成本随规模扩张的变量
GiST 成本随树高、split 频率、opclass penalty/picksplit 质量、ordered scan queue size 扩张。
GIN 成本随 distinct key 数、posting list/tree 大小、pending list pages、query entry 数、bitmap 大小扩张。
BRIN 成本随 heap blocks、pages_per_range、unsummarized ranges、summary selectivity、recheck rows 扩张。
hash 成本随 bucket count、overflow chain length、split frequency、dead tuple density 扩张。
SP-GiST 成本随 tree shape、leaf chain 长度、redirect/placeholder 残留、opclass picksplit 质量扩张。
这些变量大多不在 `IndexAmRoutine` 中。
它们是 AM 私有持久化状态与 workload 的组合。
## 10. 观测与诊断入口
### 10.1 先判断慢在公共层还是 AM 私有层
公共层可见现象：
`EXPLAIN (ANALYZE, BUFFERS, WAL)` 能看到 index scan、bitmap index scan、bitmap heap scan、recheck rows、buffer hit/read/dirtied、WAL records/bytes。
`pg_stat_all_indexes` 能看到 index scan 次数、tuples read/fetched。
`pg_stat_wal` 能看到实例级 WAL 增长。
`pg_stat_io` 能看到 IO 类别和对象粒度的累计现象。
wait event 可以看到 buffer content/IO/lock 等等待。
但这些通常不能直接告诉你“GIN pending list 太大”或“hash split cleanup 未完成”。
AM 私有状态需要 extension、pageinspect、amcheck、SQL control function、日志或 gdb。
### 10.2 能直接观测的状态
BRIN 有 SQL control functions。
`brin_summarize_new_values()` 可以触发 summarize。
`brin_summarize_range()` 可以触发某个 range summarize。
`brin_desummarize_range()` 可以标记 range unsummarized。
它们在 recovery 中会报错。
GIN 有 `gin_clean_pending_list()`。
它不能在 recovery 中运行。
pageinspect 可以观察不少 AM page。
例如 `gin_metapage_info()`、`gin_page_opaque_info()`、`brin_page_type()`、`hash_page_stats()`、`gist_page_opaque_info()`、`spgist_page_opaque_info()` 等函数在安装 pageinspect 后可用，具体函数随版本变化要以本地 extension 为准。
`pg_relation_size()` 可以看 index relation 体积。
`VACUUM VERBOSE` 可以看到某些 index vacuum 统计。
`EXPLAIN (ANALYZE, BUFFERS)` 可以看 BRIN/GIN 是否导致大量 lossy recheck。
### 10.3 只能推断的状态
GIN pending list 对查询的影响可以从 bitmap index scan 时间、heap recheck、pending cleanup 后的变化推断。
hash overflow chain 和 split cleanup 可以从 index size、bucket stats、VACUUM 行为、buffer reads 和 pageinspect 抽样推断。
GiST incomplete split 通常很短暂，除非 crash 后残留；可以从日志 `fixing incomplete split` 或 gdb 断点推断。
SP-GiST redirect/placeholder 残留可以从 pageinspect、VACUUM 后体积/性能变化和源码断点推断。
BRIN unsummarized ranges 可以通过 summarize function 返回值、pageinspect revmap、bitmap heap lossy 访问量推断。
### 10.4 几乎不可见的状态
每次 AM 内部 retry 的次数通常不可见。
GiST opclass penalty/picksplit 的局部决策不可直接从 pg_stat 看出。
GIN posting list recompression 的 action 细节通常只在 WAL/debug/gdb 层可见。
hash split 的中间 flag 很难在普通 SQL 视图中稳定捕捉。
SP-GiST redirect 被 scan 跟随的次数没有内置统计。
这些需要 perf、DTrace/eBPF、gdb、临时日志或源码插桩。
### 10.5 推荐诊断顺序
第一步看 plan shape。
确认是 index scan、bitmap index scan、bitmap heap scan 还是 index-only scan。
第二步看 `BUFFERS` 和 `WAL`。
判断慢在读、写、recheck 还是 WAL。
第三步看 AM 类型和 reloptions。
GIN 的 fastupdate/pending list、BRIN 的 pages_per_range/autosummarize、hash fillfactor、SP-GiST/GiST opclass 都很关键。
第四步用 AM-specific 工具抽样页面。
不要只看 index size。
第五步回到源码判断状态是否有 fallback。
例如 BRIN unsummarized range 是正确 fallback，不是 corruption。
hash moved-by-split 残留是设计选择，不一定要清。
SP-GiST placeholder 不在 page 末尾时不能删。
## 11. 常见误区
误区一：
`IndexAmRoutine` 是一个统一索引实现框架。
更准确地说，它是统一调用合同。
它不是统一 page layout，也不是统一 redo protocol。
误区二：
所有 index AM 都能返回 tuple-at-a-time TID。
GIN 和 BRIN 在当前 handler 中没有 `amgettuple`。
它们只能走 bitmap。
这是语义和并发设计结果，不是少实现一个函数。
误区三：
BRIN 没有 per-tuple TID，所以不用维护。
BRIN 仍然要维护 summary、revmap、unsummarized ranges。
VACUUM cleanup 还会 summarize ranges。
它只是不按 dead heap TID 删除 index tuple。
误区四：
GIN pending list 是纯内存缓冲。
pending list 是 index relation 中的持久化 list pages。
它有 metapage head/tail 和 WAL redo。
误区五：
hash split 完成后应该清掉所有 moved-by-split tuple flag。
源码 README 明确说 moved-by-split flag 会永久保留也无害，因为清它会产生额外 IO。
误区六：
SP-GiST redirect 是损坏。
redirect 是并发 scan 的正确性机制。
只有足够旧后才能转 placeholder。
误区七：
AM redo 可以用 generic index redo 合并。
各 AM 的 WAL record 语义完全不同。
合并会把私有 page state 泄漏到一个庞大且脆弱的 generic redo 层。
误区八：
`xs_recheck` 是 AM 不精确的缺陷。
对 GiST、GIN、BRIN、SP-GiST 来说，recheck 经常是设计的一部分。
它把 AM 的候选生成与 executor 的完整 qual 判断分开。
## 12. 课堂实验
### 实验 1：从 handler 验证公共契约的差异
目标：
确认五个非 B-tree AM 都共享 `IndexAmRoutine`，但能力位不同。
步骤：
在源码仓库执行：
```bash
rg -n "Datum\\n(gist|gin|brin|hash|spg).*handler|\\.amgettuple|\\.amgetbitmap|\\.amsummarizing|\\.aminsertcleanup" src/backend/access/{gist,gin,brin,hash,spgist}
```
观察点：
GIN 和 BRIN 的 `.amgettuple = NULL`。
BRIN 的 `.amsummarizing = true`。
BRIN 的 `.aminsertcleanup = brininsertcleanup`。
hash 的 `.amcanmulticol = false`。
GiST/SP-GiST 的 `amcanreturn` 与 `amcanorderbyop` 配置。
回到本节问题：
这些差异都被 `IndexAmRoutine` 表达为能力或入口。
但没有任何字段描述 pending list、revmap、bucket split 或 redirect tuple。
### 实验 2：用 SQL 观察 GIN pending list 与 bitmap path
目标：
观察 GIN fastupdate 把前台 insert 成本和后续 cleanup/recheck 成本分开。
准备：
安装 `pg_trgm` 和 `pageinspect`。
示例 SQL：
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE TABLE t_gin AS
SELECT g, md5(g::text) || ' postgres storage persistence' AS txt
FROM generate_series(1, 200000) AS g;
CREATE INDEX t_gin_txt_idx ON t_gin USING gin (txt gin_trgm_ops) WITH (fastupdate = on);
INSERT INTO t_gin
SELECT g, md5(g::text) || ' postgres storage persistence'
FROM generate_series(200001, 260000) AS g;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM t_gin WHERE txt LIKE '%storage%';
SELECT gin_clean_pending_list('t_gin_txt_idx');
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM t_gin WHERE txt LIKE '%storage%';
```
观察点：
对比 cleanup 前后的 bitmap index scan 时间、buffer 访问和 heap recheck。
如果 pageinspect 函数在本地版本可用，再查看 GIN metapage pending 信息。
回到源码：
把现象对应到 `gininsert.c:892-905`、`ginfast.c:219-472`、`ginget.c:1951-1960`。
### 实验 3：用 BRIN unsummarized range 观察 lossy fallback
目标：
验证 BRIN 未 summarized range 不会漏结果，而是扩大 bitmap heap scan 的 recheck 范围。
示例 SQL：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE TABLE t_brin AS
SELECT g, g AS k, repeat('x', 100) AS pad
FROM generate_series(1, 500000) AS g;
CREATE INDEX t_brin_k_idx ON t_brin USING brin (k) WITH (pages_per_range = 32, autosummarize = off);
INSERT INTO t_brin
SELECT g, g AS k, repeat('x', 100)
FROM generate_series(500001, 560000) AS g;
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM t_brin WHERE k BETWEEN 540000 AND 540100;
SELECT brin_summarize_new_values('t_brin_k_idx');
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM t_brin WHERE k BETWEEN 540000 AND 540100;
```
观察点：
summarize 前后 heap blocks、lossy bitmap 和 recheck 工作量可能变化。
不要期待一定变快。
效果依赖数据分布、pages_per_range、cache 和版本。
回到源码：
把现象对应到 `brin/README:92-101`、`brin.c:430-432`、`brin.c:1318-1347`、`brin.c:1878-1976`。
## 13. 讨论题
1. 为什么 `IndexAmRoutine` 要允许 `amgettuple` 为 NULL，而不是强制所有 AM 都实现 tuple-at-a-time scan？
2. GIN pending list cleanup 为什么可以容忍“pending 和 main index 中暂时重复”，而普通 `amgettuple` API 很难容忍？
3. BRIN 的 invalid revmap entry 为什么表示 unsummarized fallback，而不是 index corruption？
4. hash bucket split 为什么把完成 split 的责任放到后续 insert/split，而不是放到 search 或 VACUUM？
5. SP-GiST redirect tuple 为什么需要 XID/horizon，而 placeholder 为什么不能随便删除？
6. GiST 为什么用 `F_FOLLOW_RIGHT` 和 NSN，而不是让 generic scan 层统一处理 page split？
7. 如果你要给一个新的 index AM 设计 WAL record，哪些状态必须写在 AM 私有 record 中，哪些状态可以交给 generic WAL 框架？
8. 诊断非 B-tree 索引性能问题时，哪些信息来自公共视图，哪些必须通过 AM-specific 工具或源码断点补齐？
## 14. 本节小结
本节唯一主问题是：
PostgreSQL 为什么能共享 IndexAM 契约，却把非 B-tree AM 的持久化细节留在 AM 内部。
答案是：
共享的是调用边界，不是状态机。
`IndexAmRoutine` 统一 build、insert、scan、bitmap、vacuum、cost、capability。
`indexam.c` 统一 relation refcount、callback 检查、table fetch、snapshot 边界、统计和 cleanup 调用时机。
但页面结构、并发协议、WAL record、redo、fallback、private cleanup 必须由 AM 自己拥有。
GiST 的边界样本是 `F_FOLLOW_RIGHT`、NSN、rightlink 和 incomplete split repair。
GIN 的边界样本是 pending list、posting list/tree、bitmap 去重和 pending cleanup。
BRIN 的边界样本是 page range summary、revmap、unsummarized range 和 summarization。
hash 的边界样本是 bucket/overflow、split flags、moved-by-split 和 split cleanup。
SP-GiST 的边界样本是 inner/leaf tuple、redirect/dead/placeholder 和 redirect aging。
正确性不是一个机制单独保证的。
公共 callback 保证入口。
AM page locks 保证局部并发。
TableAM 保证 heap visibility。
WAL/redo 保证 crash recovery。
GlobalVis/snapshot conflict horizon 保证 standby 和 cleanup 安全。
MemoryContext/ResourceOwner/buffer manager 保证资源收口。
诊断时先分层：
如果问题是 plan path、callback 能力、bitmap recheck、heap fetch，先看公共 IndexAM/TableAM/executor 层。
如果问题是 pending list、unsummarized ranges、bucket split、redirect cleanup、GiST split repair，必须回到 AM 私有状态和 redo。
可迁移的系统规律是：
一个稳定内核 abstraction 不应该试图统一所有物理状态。
它应该统一调用时机、资源边界和可组合语义。
真正影响 crash safety 和并发正确性的局部状态机，必须留给拥有数据布局的模块。
这也是 PostgreSQL 能同时支持多种索引 AM 和扩展生态的原因。
