# PostgreSQL B-tree insert、unique check 与 speculative insertion

## 课程定位

上一组课程已经讲到 heap tuple、HOT、FSM/VM 和 heap 写入边界。
本节从 heap tuple 已经拿到 TID 之后继续向索引层推进。
前置知识：已经理解 MVCC tuple version、heap page、buffer content lock、WAL-before-data、B-tree leaf page、HOT chain 和 executor insert/update 的基本流程。

本节唯一主问题：
两个事务同时插入同一个 unique key 时，PostgreSQL 如何保证最终不会出现两个可见的重复 key？
本节围绕的核心矛盾：
unique index 要给用户一个全局唯一性结论。
MVCC 又允许同一个逻辑 key 在 index 中同时存在多个物理 entry 或多个未提交 heap tuple。
B-tree insertion 需要短时间持有 leaf page write lock。
但未提交事务可能持续很久，系统不能在等待事务结束时长期占住 index page lock。
`INSERT ... ON CONFLICT` 还要求冲突时撤销本次插入，而不是 abort 整个事务。
PostgreSQL 的折中是：
B-tree 在 leaf page write lock 保护下先检查同 key 的 index entries。
它不只看 index tuple 是否存在，而是用 `SnapshotDirty` 回 heap 判断对应 tuple 是 committed、aborted、in-progress 还是 speculative。
遇到未提交冲突时，释放 B-tree page lock，等待事务或 speculative token，然后从 root 重新搜索。
普通 unique insert 最终要么插入 index tuple，要么抛 `unique_violation`。
speculative insert 则先把 heap tuple 放入一个可确认、可杀死的中间状态。
unique check 发现冲突时，executor 可以杀死这个 speculative tuple，再执行 `ON CONFLICT` 动作。
读完本节，你应该能独立判断：
- unique index 为什么可以包含相同用户 key 的多个物理 index tuple。
- 为什么 unique check 必须回 heap，而不能只在 B-tree leaf page 上判断。
- 为什么 `_bt_check_unique()` 使用 `SnapshotDirty`，而不是普通 MVCC snapshot。
- 为什么等待未提交 tuple 前必须释放 B-tree buffer lock。
- 为什么等待后必须重新从 root 搜索，而不能沿用旧 leaf page 和 offset。
- `UNIQUE_CHECK_YES`、`UNIQUE_CHECK_PARTIAL` 和 `UNIQUE_CHECK_EXISTING` 分别把责任交给谁。
- speculative insertion token 存在哪里，谁等待它，谁释放它。
- `ON CONFLICT` 为什么需要 pre-check，但 pre-check 为什么不是 correctness 边界。
- heap speculative insert、confirm、abort 各自写什么 WAL。
- B-tree simple insert、page split、incomplete split 和 duplicate error 的错误边界在哪里。
- `pg_stat_activity`、`pg_locks`、isolation test、gdb 分别能看到哪些状态。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `nl -ba <source-file>`；重点源码和辅助引用统一放在第 3 节的阅读顺序里。

## 1. 本节在总主线中的位置

heap insert 的结果是一个已经落到 heap page 上的 tuple。
这个 tuple 有自己的 `ItemPointer`，也就是 index entry 里要保存的 heap TID。
对普通非唯一索引来说，executor 形成 index datum 后调用 index AM 插入即可。
对 unique index 来说，插入动作同时承担约束检查。
这就是本节的位置：

```text
executor
  -> heap tuple 已插入或 speculative 插入
  -> ExecInsertIndexTuples()
  -> index_insert()
  -> btinsert()
  -> _bt_doinsert()
  -> _bt_check_unique()
  -> _bt_insertonpg()
```

这个链路的关键不是“怎么把 tuple 塞进 B-tree page”。
关键是“什么时候可以认定同一个 key 没有另一个会变成可见的版本”。
B-tree 只能保护局部页结构。
事务可见性却来自 heap tuple header、CLOG、ProcArray 和 speculative token。
所以 unique index 的 correctness 是一个跨模块结论。
它不是单个 latch、单个 lock 或单个 WAL record 能独立保证的。
本节只讲 insert path。
index build、`CREATE UNIQUE INDEX CONCURRENTLY` 的完整流程留到后续课程。
但 `_bt_check_unique()` 里为了支持 recheck 和 HOT chain 的边界会被点到为止。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：
B-tree 在“第一个可能出现该 key 的 leaf page”上短暂串行化并发插入，用 dirty heap visibility 把同 key entry 分类为 dead、committed conflict、in-progress conflict 或 speculative conflict；遇到不确定状态就释放页锁等待并重查，`ON CONFLICT` 则把本事务的新 tuple 放入可撤销的 speculative 状态。
这个模型里有三层唯一性。
第一层是物理唯一性。
BTREE version 4 把 heap TID 当作隐含的 trailing key。
`src/include/access/nbtree.h:383-400` 说明非 pivot tuple 的 `t_tid` 指向 heap TID，并且 heap TID 成为 tiebreaker key column。
这让 B-tree 内部可以对多个相同用户 key 的物理 tuple 排序。
第二层是用户可见唯一性。
SQL 约束关心的是某个 MVCC 未来状态下，会不会有两个 live tuple 拥有相同 key。
这个结论不能从 leaf page 上相同 key 的个数直接推出。
第三层是 speculative verdict。
`ON CONFLICT` 的 inserter 在事务还没结束时，先声明“我可能保留这个 tuple，也可能杀掉它”。
其他 backend 不必等整个事务结束。
它们只需要等这个 speculative decision。
本节所有细节都围绕这三层之间的边界展开。
物理 B-tree 可以容纳重复。
unique constraint 不能容纳两个最终可见的重复。
speculative insertion 可以临时容纳一个可撤销的重复。

## 3. 核心文件分工与阅读顺序

建议按下面顺序读源码。
不要从 `nbtinsert.c` 顶部线性背函数。
先抓住状态和等待边界。

```text
src/include/access/genam.h
```

定义 `IndexUniqueCheck`。
`genam.h:103-129` 是 executor 和 index AM 之间的 unique check 合同。

```text
src/backend/executor/execIndexing.c
```

决定每个 index insert 用哪种 unique check 模式。
`execIndexing.c:7-26` 说明 unique index 的原子检查由 index AM 负责。
`execIndexing.c:53-96` 说明 speculative insertion 的设计目标。
`execIndexing.c:410-457` 把 immediate、deferred、speculative 分派成不同 `checkUnique`。

```text
src/backend/executor/nodeModifyTable.c
```

驱动 `INSERT ... ON CONFLICT` 的 speculative lifecycle。
`nodeModifyTable.c:1131-1269` 是本节最重要的 executor 主链路。

```text
src/backend/access/nbtree/nbtree.c
```

B-tree AM 的 public insert 入口。
`nbtree.c:200-223` 形成 index tuple，填入 heap TID，调用 `_bt_doinsert()`。

```text
src/backend/access/nbtree/nbtinsert.c
```

本节主文件。
`_bt_doinsert()` 负责 search、unique check、wait/retry、insert。
`_bt_check_unique()` 负责同 key 扫描和 heap visibility 判断。
`_bt_findinsertloc()` 和 `_bt_insertonpg()` 负责物理插入、split、WAL。

```text
src/backend/access/nbtree/nbtsearch.c
```

解释 `_bt_search()`、`_bt_moveright()`、`_bt_binsrch_insert()`。
这些函数决定“第一个可能包含 key 的 leaf page”和“恢复 heap TID tiebreaker 后的物理位置”。

```text
src/include/access/nbtree.h
```

定义 heapkeyspace、posting list、`BTScanInsertData`、`BTInsertStateData`。
本节只用其中和 unique insert 相关的字段。

```text
src/backend/access/heap/heapam.c
```

普通 heap insert、speculative confirm、speculative abort 的 WAL 和错误边界。

```text
src/backend/access/heap/heapam_visibility.c
```

`HeapTupleSatisfiesDirty()` 如何把 in-progress XID 和 speculative token 返回给 caller。

```text
src/backend/storage/lmgr/lmgr.c
```

speculative insertion lock 的 token 生成、等待和释放。
读完这些文件后，再回到 SQL 现象。
否则很容易把 `ON CONFLICT` 理解成 executor 层的简单 retry。
实际上 retry 的前提是 heap、B-tree、lock manager 已经提供了可等待、可撤销、可重查的状态边界。

## 4. 关键状态与边界

先列本节只需要的状态。
不要把它们孤立背成字段。
每个字段的语义都依赖 lifecycle 和锁上下文。
`IndexUniqueCheck` 是 executor 传给 index AM 的模式。
定义在 `src/include/access/genam.h:123-129`。
`UNIQUE_CHECK_NO` 表示不做唯一性检查。
`UNIQUE_CHECK_YES` 表示 insertion time 立即强制唯一性。
`UNIQUE_CHECK_PARTIAL` 表示只测试，不等待、不报错，返回“确定唯一”或“可能冲突”。
`UNIQUE_CHECK_EXISTING` 表示 deferred constraint 之后回来 recheck，一个已有 index tuple 不再重复插入。
注意 `UNIQUE_CHECK_PARTIAL` 不是 partial index。
它是 partial uniqueness check。
这个名字很容易误导。
`BTScanInsertData` 是 B-tree insertion scankey。
定义在 `src/include/access/nbtree.h:753-805`。
对 heapkeyspace index，`scantid` 是最后的 tiebreaker。
普通物理插入时必须设置。
unique check 期间会临时清空。
这是本节最重要的 B-tree 层状态切换之一。
`BTInsertStateData` 是 `_bt_doinsert()` 的工作区。
定义在 `src/include/access/nbtree.h:810-844`。
它保存待插入 `itup`、tuple size、当前 leaf buffer、binary search bounds 和 posting list offset。
这些状态是 backend-local。
它们不能跨进程共享。
它们只在一次 insert attempt 内有效。
一旦等待、page modification、step right 或 retry，缓存的 bounds 就可能失效。
heap tuple 的 `t_ctid` 在 speculative insertion 中有特殊含义。
`src/include/access/htup_details.h:105-112` 明确说 `t_ctid` 有时存 speculative insertion token，而不是真 TID。
`src/include/storage/itemptr.h:57-63` 说明特殊编码：
`ip_posid` 设为 `SpecTokenOffsetNumber`。
token 存在 `ip_blkid`。
`src/include/access/htup_details.h:451-468` 提供 `HeapTupleHeaderIsSpeculative()`、`HeapTupleHeaderGetSpeculativeToken()` 和 `HeapTupleHeaderSetSpeculativeToken()`。
所以 speculative token 不是 index tuple 字段。
它也不是一个新的 XID。
它是 heap tuple header 中 `t_ctid` 的特殊状态，并配合 lock manager 的 speculative lock tag 使用。
`SnapshotDirty` 是 unique check 使用的 visibility 工具。
它不是普通查询 snapshot。
`src/backend/access/heap/heapam_visibility.c:739-756` 说明它会把影响 tuple visibility 的 in-progress `xmin`、`xmax` 和 `speculativeToken` 填回 snapshot struct。
这让 caller 能判断“这个 tuple 现在算冲突，但结论还没定，需要等谁”。
B-tree leaf page write lock 保护的是局部 key range 的检查与插入窗口。
`nbtinsert.c:190-199` 的注释非常关键：
`_bt_check_unique()` 只能看到已经在 index 里的 key。
为了防止两个 inserter 同时检查都看不到对方，系统让它们都必须锁住“该 value 可能出现的第一页”。
同 key 的 would-be inserter 会竞争同一个 leaf page write lock。
但这个锁不能覆盖整个事务。
遇到未提交 heap tuple 后，`_bt_doinsert()` 会释放 buffer lock 再等待。
等待结束后重新搜索。
这就是 unique insert 的核心并发边界。

## 5. Executor 如何选择 unique check 模式

`ExecInsertIndexTuples()` 是普通 insert/update 后写 index 的主入口。
`execIndexing.c:274-318` 说明它为每个 index 形成 index tuple，并同时处理 unique/exclusion constraint。
它在 `execIndexing.c:410-436` 决定 `checkUnique`：
非 unique index 使用 `UNIQUE_CHECK_NO`。
`EIIT_NO_DUPE_ERROR` 为 true 时使用 `UNIQUE_CHECK_PARTIAL`。
immediate unique index 使用 `UNIQUE_CHECK_YES`。
deferrable unique index 使用 `UNIQUE_CHECK_PARTIAL`。
`EIIT_NO_DUPE_ERROR` 是 speculative insertion path 的关键 flag。
它告诉 index AM：
不要在发现重复时直接抛错。
先告诉 executor 这次 insertion 是否可能冲突。
executor 再决定杀死 speculative tuple 或继续确认。
`execIndexing.c:426-428` 直接把 speculative insertion 和 deferrable unique check 放在同一类：
都先做 `UNIQUE_CHECK_PARTIAL`。
区别在后续处理。
deferrable unique 会把可能冲突的 index OID 交给 later recheck。
speculative insertion 则会在当前 statement 内根据 `specConflict` 重试或执行 `ON CONFLICT` 动作。
`execIndexing.c:502-515` 负责收集这些 possibly conflicting index OID。
如果是 immediate unique index 且 caller 提供了 `specConflict` 指针，发现 possible conflict 会把 `*specConflict = true`。
这一层没有自己证明唯一性。
它只是选择 AM 行为并收集返回结果。
真正的 atomic unique check 在 B-tree AM 里。
这就是 executor 和 index AM 的第一条边界。

## 6. 普通 B-tree insert 入口

`btinsert()` 在 `src/backend/access/nbtree/nbtree.c:200-223`。
它做的事很少。
它调用 `index_form_tuple()` 形成 index tuple。
它把 heap TID 写到 `itup->t_tid`。
然后调用 `_bt_doinsert()`。
这很重要。
unique index 的物理 entry 指向 heap tuple。
即使用户 key 相同，只要 heap TID 不同，B-tree 的物理排序仍然能区分它们。
`src/include/access/nbtree.h:383-400` 解释了这个格式。
用户 key 不是 B-tree 的全部 key。
在 heapkeyspace index 中，heap TID 也是 ordering 的一部分。
但 unique check 不能简单使用完整物理 key。
如果用完整物理 key 搜索，两个不同 heap TID 的相同用户 key 会被当作不同 key。
因此 `_bt_doinsert()` 在 unique check 期间会临时去掉 `scantid`。
`nbtinsert.c:118-124`：
如果需要 unique check 且 key 没有 NULL，就把 `itup_key->scantid = NULL`。
这样 search 语义变成：
找这个用户 key 的第一个可能位置。
而不是找这个用户 key 加 heap TID 的精确物理位置。
如果 key 中有 NULL，默认 unique 语义认为 NULL 与任何值都不相等，包括 NULL。
`nbtinsert.c:127-140` 因此跳过 unique check。
这也是为什么 `UNIQUE (a)` 可以有多行 `a IS NULL`。
如果 index 是 NULLS NOT DISTINCT，则这个优化路径不会按默认 NULL distinct 语义跳过。
本节重点是默认 B-tree unique insert 主链路，不展开 NULLS NOT DISTINCT 的全部实现。

## 7. `_bt_doinsert()` 的时间线

`_bt_doinsert()` 的注释在 `nbtinsert.c:82-104`。
它说明：
`UNIQUE_CHECK_NO` 和 `UNIQUE_CHECK_PARTIAL` 会允许 duplicate index tuple 插入。
`UNIQUE_CHECK_YES` 和 `UNIQUE_CHECK_EXISTING` 会在确定 duplicate 时抛错。
`UNIQUE_CHECK_EXISTING` 只检查，不插入。
主流程从 `nbtinsert.c:163` 的 `search:` label 开始。
第一步是 `_bt_search_insert()`。
`nbtinsert.c:165-170` 注释说它找到并锁住 tuple 应该加入的 leaf page。
返回后 `insertstate.buf` 是 exclusive locked buffer。
第二步是 unique check。
`nbtinsert.c:208-214` 调用 `_bt_check_unique()`。
它返回三类结果。
返回 invalid xid 表示无需等待。
返回 valid xid 表示需要等待另一个事务。
如果发现 definite conflict，函数内部直接 `ereport(ERROR)`，不返回。
第三步是等待。
`nbtinsert.c:216-235`：
如果有 `xwait`，先 `_bt_relbuf(rel, insertstate.buf)` 释放 leaf page。
然后根据 `speculativeToken` 决定等 `SpeculativeInsertionWait()` 还是 `XactLockTableWait()`。
等待完成后释放 descent stack，跳回 `search`。
这一步是本节的核心。
等待发生时不能持有 B-tree page lock。
否则一个长事务可以把同 key range 的所有插入、split、VACUUM 相关动作压住。
更严重的是，等待期间 page 可能 split、delete、rightlink 改变。
所以等待后不能沿用旧 buffer 和 offset。
必须重新从 root 搜索。
第四步是恢复物理 scankey。
`nbtinsert.c:238-240`：
唯一性建立后，如果是 heapkeyspace index，把 `itup_key->scantid = &itup->t_tid`。
这让后续 `_bt_findinsertloc()` 回到完整物理 ordering。
第五步是物理插入。
`nbtinsert.c:243-266`：
如果不是 `UNIQUE_CHECK_EXISTING`，先做 SSI conflict-in 检查，再调用 `_bt_findinsertloc()` 和 `_bt_insertonpg()`。
如果是 `UNIQUE_CHECK_EXISTING`，只释放 buffer，不插入。
这个流程可以压缩为：

```text
_bt_doinsert()
  -> build insertion scankey
  -> unique check 时暂时去掉 heap TID tiebreaker
  -> _bt_search_insert() 找到第一个可能有 duplicate 的 leaf page
  -> _bt_check_unique() 用 SnapshotDirty 回 heap 分类同 key tuple
  -> 需要等待: release buffer, wait, retry from root
  -> 唯一性建立: 恢复 heap TID tiebreaker
  -> _bt_findinsertloc() 找物理 offset
  -> _bt_insertonpg() 修改 page, 写 WAL, release buffer
```

这条时间线必须记住。
本节后面所有错误路径和观测实验都回到它。

## 8. `_bt_search()` 为什么只找到起点

`_bt_search()` 在 `src/backend/access/nbtree/nbtsearch.c:76-99`。
它不是“找到唯一的插入页”。
它更准确地说是找到这个 scankey 第一个可能所在的 leaf page。
在 write mode 下，它会处理 empty root 和 incomplete split。
`nbtsearch.c:87-94` 说明返回的 leaf buffer 被 lock 和 pin。
`nbtsearch.c:96-97` 说明 write mode caller 必须提供 heap relation，因为可能需要分配新 root page。
`_bt_moveright()` 在 `nbtsearch.c:210-321`。
它处理并发 split 后“跟着旧 downlink 到了左边页”的情况。
通过 high key 判断是否需要沿 rightlink 右移。
如果是 insertion write path，`nbtsearch.c:232-235` 说明遇到 incomplete split 会尝试完成它。
这和 unique insert 的 correctness 相关。
如果一个 page split 只完成了 leaf 半边，还没有把 downlink 插入 parent，另一个 inserter 不能假装树已经稳定。
它要么完成 split，要么等后续路径处理。
B-tree 的 Lehman-Yao rightlink 机制允许 search 在并发 split 下继续正确。
但 unique check 对“第一个可能包含 key 的 leaf page”更敏感。
两个同 key inserter 必须在同一个起点 page 上串行化。
这就是 `_bt_doinsert()` 先清空 `scantid` 的原因。
如果一开始就带 heap TID tiebreaker，两个不同 heap TID 可能落到相邻页或不同 offset。
那就不能用同一个 leaf page write lock 串行化“同用户 key”的检查窗口。

## 9. `_bt_check_unique()` 的扫描对象

`_bt_check_unique()` 的注释在 `nbtinsert.c:388-409`。
它返回需要等待的 xid。
如果冲突 tuple 仍处于 speculative insertion，则通过 `*speculativeToken` 返回 token。
如果是 `UNIQUE_CHECK_PARTIAL`，它不等待，设置 `*is_unique = false` 后返回 invalid xid。
它还有一个 side effect：
保存 binary search bounds，供 `_bt_findinsertloc()` 复用。
函数开始时 `InitDirtySnapshot(SnapshotDirty)`。
见 `nbtinsert.c:430-446`。
然后用 `_bt_binsrch_insert()` 找到第一个相等 key 的 offset。
这里的 `itup_key->scantid` 必须是 NULL。
`nbtinsert.c:451-453` 有 assert。
随后循环扫描所有 equal tuples。
注意扫描单位不是 index tuple。
`nbtinsert.c:456-467` 明确说每次迭代处理一个 heap TID。
如果当前 index tuple 是 posting list tuple，会逐个处理 posting list 中的 TID。
这和 dedup 有关。
unique check 的语义不能被 posting list 压缩破坏。
每一个 heap TID 都要拿出来回 heap 判定。
`nbtinsert.c:524-544` 从普通 non-pivot tuple 或 posting list tuple 中取出 `htid`。
然后进入最关键的 heap 检查：

```text
table_index_fetch_tuple_check(heapRel, &htid, &SnapshotDirty, &all_dead)
```

这行在 `nbtinsert.c:563-565`。
它不是普通 heap fetch。
它走 table AM index fetch 路径。
`src/backend/access/table/tableam.c:242-260` 显示它创建 index fetch scan 和 slot，然后调用 `table_index_fetch_tuple()`。
heap AM 的实现会追 HOT chain。
`src/backend/access/heap/heapam_indexscan.c:69-225` 是 `heap_hot_search_buffer()`。
它从 index entry 指向的 TID 出发，必要时沿同页 HOT chain 找第一个满足 snapshot 的 tuple。
所以 unique check 的实际问题是：
这个 index entry 对应的 heap TID 或 HOT chain 上，有没有一个 `SnapshotDirty` 认为仍然有效或未定的 tuple？
不是：
leaf page 上有没有相同 key 的 index tuple？
这两个问题差别很大。
dead index tuple 可以留在 B-tree 中。
HOT chain 可能让一个 index entry 间接代表多个 heap versions。
speculative tuple 可能当前存在但稍后被杀死。

## 10. `SnapshotDirty` 如何表达未定状态

`HeapTupleSatisfiesDirty()` 是 dirty snapshot 的核心。
`heapam_visibility.c:739-756` 的注释是本节必须读的段落。
它说这个 visibility 函数把 open transactions 的 effects 也算进去。
同时用 snapshot struct 作为输出参数。
如果 tuple 的 `xmin` 是另一个 in-progress transaction，填 `snapshot->xmin`。
如果 tuple 的 `xmax` 是另一个 in-progress transaction，填 `snapshot->xmax`。
如果 tuple 是 speculative insertion，填 `snapshot->speculativeToken`。
`heapam_visibility.c:767-768` 每次先清空这三个输出字段。
`heapam_visibility.c:811-829` 处理“另一个事务正在插入”的情况。
如果 tuple header 是 speculative，就取 token。
然后设置 `snapshot->xmin` 为插入者 xid，并返回 true。
这个返回 true 很容易误解。
它不是说该 tuple 已经对普通 MVCC query 可见。
它是说 dirty snapshot 要把这个 in-progress tuple 当作 potential conflict 交给 caller。
caller 再根据 `xmin/xmax/speculativeToken` 决定等待。
`_bt_check_unique()` 在 `nbtinsert.c:585-600` 做这个决定。
它选择 `xwait = SnapshotDirty.xmin` 或 `SnapshotDirty.xmax`。
如果 `xwait` valid，就把 `SnapshotDirty.speculativeToken` 传给 caller。
然后 invalidates bounds 并返回 `xwait`。
普通 in-progress insertion：
`speculativeToken == 0`。
caller 等 transaction lock。
speculative insertion：
`speculativeToken != 0`。
caller 等 speculative token lock。
这是等待对象的根本区别。

## 11. definite conflict 的错误边界

如果 `table_index_fetch_tuple_check()` 返回 true，且没有 in-progress xid，那么 `_bt_check_unique()` 认为这是 definite conflict。
但它在报错前还有一个边界：
`nbtinsert.c:603-633` 会检查自己要插入的 tuple 是否已经 committed dead。
这个路径主要服务 `CREATE INDEX CONCURRENTLY` 等 recheck 场景。
如果自己这个 tuple 已经不活了，就不需要报 unique violation。
正常 insert path 下，自己的 tuple 仍然 live。
然后 `nbtinsert.c:635-642` 做 SSI conflict-in 检查。
接下来是错误边界。
`nbtinsert.c:644-656` 先释放可能持有的 right sibling buffer 和当前 `insertstate->buf`。
注释说 `BuildIndexValueDescription()` 可能访问 catalog。
如果还持有这个 index 的 buffer lock，最坏可能自锁或引入复杂死锁。
只有释放 buffer 后才 `index_deform_tuple()`、`BuildIndexValueDescription()` 并 `ereport(ERROR)`。
这说明 unique violation error 发生时，B-tree page 还没有被本次 insert 修改。
对 `UNIQUE_CHECK_YES` 的普通 duplicate：
冲突检查在物理 insertion 之前。
报错路径不会进入 `_bt_insertonpg()`。
不会写本次 index tuple 的 B-tree insert WAL。
heap tuple 是否已经插入取决于更上层的流程。
普通 insert 中，heap tuple 已经插入，ERROR 会 abort 当前 transaction 或 subtransaction，由事务 cleanup 处理。
speculative insertion 中，index AM 不会直接报错，而是返回 possible conflict 给 executor。
executor 决定杀死 speculative heap tuple。
这两个错误边界不能混在一起。

## 12. `UNIQUE_CHECK_PARTIAL` 的边界

`UNIQUE_CHECK_PARTIAL` 让 AM 做“可能冲突”的检测。
`genam.h:110-116` 要求 AM 不报错、不阻塞、不阻止插入。
如果确定 unique，返回 true。
如果可能 non-unique，返回 false。
`_bt_check_unique()` 在 `nbtinsert.c:570-582` 实现这个规则。
一旦 dirty snapshot 发现 potential conflict，并且 `checkUnique == UNIQUE_CHECK_PARTIAL`，它释放额外 buffer，设置 `*is_unique = false`，返回 invalid xid。
这时 `_bt_doinsert()` 不等待。
它继续执行物理 index insertion。
这也是为什么 deferred unique 和 speculative insert 可以先把重复 key 的物理 index tuple 放进去。
它们把最终 verdict 推迟到更高层或后续 recheck。
对 deferred unique constraint：
executor 记录 recheck index OID。
之后用 `UNIQUE_CHECK_EXISTING` 回来检查已有 tuple。
`nbtinsert.c:243-271` 在 `UNIQUE_CHECK_EXISTING` 下只做 duplicate check，不插入。
对 speculative insertion：
executor 立刻用 `specConflict` 决定 confirm 还是 abort speculative tuple。
这一点很关键。
`UNIQUE_CHECK_PARTIAL` 不是降低 correctness。
它只是改变谁在什么时间承担最终结论。

## 13. `ON CONFLICT` 的 speculative 主链路

`INSERT ... ON CONFLICT` 的主流程在 `nodeModifyTable.c:1131-1269`。
它先进入 `vlock:` label。
第一步是 non-conclusive pre-check。
`nodeModifyTable.c:1143-1154` 说明，这个 pre-check 不持有锁，不能保证后续 insert 不冲突。
它只是避免在明显已经冲突时留下大量 canceled speculative insertions。
`nodeModifyTable.c:1158-1162` 调用 `ExecCheckIndexConstraints()`。
如果找到 committed conflict，就根据 `ON CONFLICT DO UPDATE`、`DO SELECT` 或 `DO NOTHING` 执行对应动作。
这还没进入 speculative insert。
第二步是获取 speculative insertion lock。
`nodeModifyTable.c:1225-1232` 调用：

```text
SpeculativeInsertionLockAcquire(GetCurrentTransactionId())
```

注释说其他 backend 可以用这个 lock 等待本事务决定保留还是放弃插入。
第三步是 heap speculative insert。
`nodeModifyTable.c:1234-1239` 调用 `table_tuple_insert_speculative()`。
heap AM handler 在 `heapam_handler.c:169-185` 里把 token 写入 tuple header：
`HeapTupleHeaderSetSpeculativeToken(tuple->t_data, specToken)`。
同时设置 `HEAP_INSERT_SPECULATIVE`。
第四步是插入 index entries。
`nodeModifyTable.c:1241-1245` 调用 `ExecInsertIndexTuples()`，带 `EIIT_NO_DUPE_ERROR`。
这会让 unique index 使用 `UNIQUE_CHECK_PARTIAL`。
如果 unique check 发现 potential conflict，就设置 `specConflict`。
第五步是完成 speculative tuple。
`nodeModifyTable.c:1247-1249` 调用：

```text
table_tuple_complete_speculative(..., specToken, !specConflict)
```

如果没有冲突，heap tuple 被 confirm。
如果有冲突，heap tuple 被 abort_speculative 杀死。
第六步是释放 speculative insertion lock。
`nodeModifyTable.c:1251-1258` 释放 token lock。
注释说明等待者醒来后会重新检查 tuple。
如果 tuple 已确认，等待者像普通 inserted tuple 一样再等 XID 或报冲突。
如果 tuple 已杀死，等待者会把它当作不存在。
第七步是必要时 retry。
`nodeModifyTable.c:1260-1268`：
如果 `specConflict`，释放 recheckIndexes，跳回 `vlock`。
这不是无限乐观循环。
每次循环都重新做 pre-check。
并且 speculative token 等待避免了两个 backend 在同一个“未定 tuple”上互相死锁。

## 14. speculative token lock 的 ownership

speculative token lock 在 lock manager 中实现。
`src/backend/storage/lmgr/lmgr.c:30-45` 定义每个 backend 一个 `speculativeInsertionToken` 计数器。
它可以 wrap around。
源码注释解释即使极端 wrap，最坏也只是误等后续 unrelated insertion，可能触发 deadlock abort，不会破坏 correctness。
`SpeculativeInsertionLockAcquire()` 在 `lmgr.c:775-803`。
它递增 token。
避开 0，因为 0 表示没有 token。
然后用 `SET_LOCKTAG_SPECULATIVE_INSERTION(xid, token)` 构造 lock tag。
最后获取 `ExclusiveLock`。
`SET_LOCKTAG_SPECULATIVE_INSERTION` 在 `src/include/storage/locktag.h:143-153`。
locktag field1 是 xid。
field2 是 token。
locktag type 是 `LOCKTAG_SPECULATIVE_TOKEN`。
`SpeculativeInsertionWait()` 在 `lmgr.c:827-839`。
等待者构造同一个 locktag。
获取 `ShareLock`。
拿到后立刻释放。
这意味着等待者关心的不是长期持有某个资源。
它只关心 exclusive lock 是否已经释放，也就是 inserter 是否已经做出 confirm/abort decision。
`SpeculativeInsertionLockRelease()` 在 `lmgr.c:805-819`。
它用当前 backend 的 token 释放 `ExclusiveLock`。
注意 release 发生在 `table_tuple_complete_speculative()` 之后。
所以等待者醒来再看 heap tuple 时，tuple header 已经不再处于未定状态。
这就是 token lock 的 happens-before 语义。
它不是 WAL 语义。
它是 backend 间的等待协议。

## 15. speculative token 在 heap tuple 中的生命周期

heap speculative tuple 的创建在 `heapam_handler.c:169-185`。
handler 把 token 写入 `t_ctid`。
然后调用普通 `heap_insert()`，但带 `HEAP_INSERT_SPECULATIVE`。
`heap_insert()` 在 `heapam.c:1986-2190`。
它设置 tuple header、选 heap page、进入 critical section、调用 `RelationPutHeapTuple()`。
`heapam.c:2056-2060` 说明从这里开始到 WAL 完成不能 `ereport(ERROR)`。
`heapam.c:2121-2122` 如果 options 包含 `HEAP_INSERT_SPECULATIVE`，就在 heap insert WAL record 里设置 `XLH_INSERT_IS_SPECULATIVE`。
`heapam.c:2162-2164` 写 WAL 并设置 page LSN。
`heapam.c:2181-2182` 特别说明 speculative insertion 也计入 heap insert stats，即使后续 aborted。
confirm path 在 `heapam.c:6047-6121`。
`heap_finish_speculative()` 拿到 heap page exclusive lock。
进入 critical section。
确认 tuple header 仍是 speculative。
把 `t_ctid` 改为真实 TID，也就是指向自己。
然后写 `XLOG_HEAP_CONFIRM`。
`heapam.c:6054-6060` 解释为什么不能 commit 时隐式当作 confirmed。
显式清 token 让 logical decoding 简单，也避免系统在 committed tuple 上继续处理 speculative token。
abort path 在 `heapam.c:6124-6285`。
`heap_abort_speculative()` 只允许杀死本事务同一 command 插入的 speculative tuple。
`heapam.c:6181-6188` 做 sanity check。
critical section 中把 `xmin` 设为 `InvalidTransactionId`。
`heapam.c:6226-6230` 说明这让 tuple 对所有事务立即不可见。
同时把 `t_ctid` 改回 self。
`heapam.c:6237-6263` 写 WAL。
注释说这里生成的 WAL records match `heap_delete()`，redo 复用 delete recovery。
所以 speculative heap tuple 的状态图是：

```text
not present
  -> heap_insert(... HEAP_INSERT_SPECULATIVE)
       t_ctid = speculative token
       WAL: XLOG_HEAP_INSERT + XLH_INSERT_IS_SPECULATIVE
  -> complete_speculative(succeeded=true)
       t_ctid = self
       WAL: XLOG_HEAP_CONFIRM
  -> commit 后成为普通 tuple
```

或者：

```text
not present
  -> heap_insert(... HEAP_INSERT_SPECULATIVE)
       t_ctid = speculative token
  -> complete_speculative(succeeded=false)
       xmin = InvalidTransactionId
       t_ctid = self
       WAL: XLOG_HEAP_DELETE style record
  -> 对所有 snapshot 都像不存在
```

index tuple 的生命周期不完全同步。
speculative path 中 index tuple 可能已经插入。
如果 heap tuple 被 abort_speculative 杀死，index tuple 变成 dead pointer。
后续 scan、simple deletion、VACUUM 或 bottom-up deletion 会清理。
正确性不要求立刻删除 index tuple。
正确性只要求 unique check 回 heap 后不会把 killed speculative tuple 当作 live conflict。

## 16. B-tree physical insertion 与 WAL 边界

唯一性建立后，或者 `UNIQUE_CHECK_PARTIAL` 选择继续插入后，进入物理 insertion。
`_bt_findinsertloc()` 在 `nbtinsert.c:790-827`。
它的输入 buffer 已经 exclusive locked。
如果前面做过 unique check，它可能保存了 binary search bounds。
但它仍然可能需要向右移动。
`nbtinsert.c:854-920` 说明 heapkeyspace index 下，恢复 `scantid` 后，新 tuple 的物理位置可能不在最初检查 duplicate 的 page。
原因是 unique check 用的是“用户 key without heap TID”。
物理 insertion 用的是“用户 key with heap TID”。
如果 page free space 不够，`_bt_findinsertloc()` 会尝试 simple deletion、bottom-up deletion 或 dedup，避免 split。
`nbtinsert.c:2702-2829` 是这部分。
这不是 unique correctness 的必要条件。
它是空间和写放大的优化。
真正修改页的是 `_bt_insertonpg()`。
注释在 `nbtinsert.c:1089-1116`。
它负责 posting list split、page split、插入 tuple、递归插入 parent downlink、更新 metapage。
simple insert 的 critical section 在 `nbtinsert.c:1296-1421`。
进入 critical section 后：
修改 posting list 或 `PageAddItem()`。
`MarkBufferDirty()`。
必要时修改 metapage 或 child incomplete split flag。
注册 WAL。
根据场景写 `XLOG_BTREE_INSERT_LEAF`、`XLOG_BTREE_INSERT_POST`、`XLOG_BTREE_INSERT_UPPER` 或 `XLOG_BTREE_INSERT_META`。
写入 `XLogInsert()`。
设置 page LSN。
退出 critical section。
`nbtinsert.c:1296` 的注释非常直接：
`No ereport(ERROR) until changes are logged`。
这就是 B-tree insert 的 WAL-before-data 边界。
page split 的 critical section 在 `nbtinsert.c:1952-2095`。
split 前可以在 local temp pages 上准备 left/right page。
但一旦把 new left page copy 回 original buffer，并把 right page 内容写到 destination buffer，就必须进入 critical section。
然后 dirty left/right/right-sibling/child buffers，写 `XLOG_BTREE_SPLIT_L` 或 `XLOG_BTREE_SPLIT_R`，设置 LSN。
如果 parent downlink 还没插入，left page 会带 `BTP_INCOMPLETE_SPLIT`。
`_bt_finish_split()` 在 `nbtinsert.c:2259-2317` 说明 crash 或 failure 可能留下 incomplete split。
后续 insertion search 遇到它会完成 split。
因此 B-tree split 的 crash safety 不要求“一条前台调用链必须一次做完所有层级”。
它要求任何可见中间状态都可被后续操作识别并完成。
这和 unique wait/retry 的思想相似：
不要长时间占住所有资源。
把中间状态做成可识别、可恢复、可重查。

## 17. 为什么等待后必须重查

在 `_bt_doinsert()` 中，等待前释放 `insertstate.buf`。
这一步之后有三件事可能发生。
第一，冲突事务 commit。
那原先的 potential conflict 变成 definite conflict。
第二，冲突事务 abort。
那原先的 potential conflict 消失。
第三，冲突事务是 speculative insertion。
它可能 confirm，也可能 abort_speculative。
如果 confirm，等待者还可能需要等事务结束或报 conflict。
如果 abort，等待者可以继续插入。
同时，B-tree page 结构也可能变化。
等待期间 leaf page 可以 split。
rightlink 可以改变。
LP_DEAD hint 可以被设置。
incomplete split 可以被别的 backend 完成。
posting list 可以被拆分或 dedup。
所以 `_bt_doinsert()` 不允许“等完后继续从旧 offset 扫描”。
它跳回 `search`。
`nbtinsert.c:232-235` 是这个 retry。
这也是观测普通 unique wait 时经常看到的行为：
blocked backend 醒来后，不是直接插入。
它会重新进入 `_bt_search_insert()` 和 `_bt_check_unique()`。
如果 blocker committed，它随后报 duplicate key error。
如果 blocker rolled back，它重新证明唯一性并插入。

## 18. unique index 为什么可以有物理重复

这是本节最容易出错的 mental model。
unique index 并不意味着 leaf page 上每个用户 key 只出现一次。
它意味着任意时刻不能有两个对同一唯一约束冲突的 live heap tuples。
物理重复来自几个来源。
第一，MVCC update。
一个 logical row 的旧版本和新版本可能都有同一个 indexed key。
如果不是 HOT，index 里会有多个 entry。
它们不能同时对同一个 snapshot 都是 live conflict。
第二，未提交 insert。
两个事务可能都在尝试同一个 key。
B-tree 必须把第二个事务挡在 unique check 或等待阶段。
第三，deferrable unique。
`UNIQUE_CHECK_PARTIAL` 可以先允许物理重复，之后 recheck。
第四，speculative insertion。
`ON CONFLICT` 可以先插入 heap 和 index entry，然后在 conflict verdict 后杀死 heap tuple。
第五，dedup posting list。
posting list 压缩多个相同 key 的 heap TID。
`_bt_check_unique()` 因此按 heap TID 迭代，而不是按 index tuple 迭代。
把 unique index 理解成“B-tree 层绝不存重复 key”会导致错误的诊断。
正确说法是：
B-tree 维护物理有序性。
unique check 维护用户可见唯一性。
heap visibility 决定某个 physical duplicate 是否是冲突。

## 19. NULL、partial index 与表达式 index 的边界

默认 unique index 允许多个 NULL。
`_bt_doinsert()` 在 `nbtinsert.c:127-140` 遇到 `anynullkeys` 会绕过 checkingunique。
原因是 core code 认为 NULL 不等于任何值，包括 NULL。
这不是 B-tree 比较函数本身自然推出的唯一语义。
这是 unique constraint 的 SQL 语义在 insert path 上的优化。
partial index 的 predicate 在 executor 层处理。
`execIndexing.c:379-398`：
如果 partial index predicate 不满足，跳过该 index update。
所以 partial unique index 的约束范围先由 predicate 决定。
只有满足 predicate 的 tuple 才进入 B-tree unique check。
表达式 index 的值由 `FormIndexDatum()` 形成。
`execIndexing.c:400-408` 在每个 index 上计算 index datum。
如果表达式不是 immutable，后续 `UNIQUE_CHECK_EXISTING` 可能 re-find 失败。
`nbtinsert.c:768-780` 的 internal error hint 就提到 non-immutable index expression。
这说明 unique check 的正确性依赖 operator class equality、index expression 稳定性和 heap visibility 的组合。
不是只有 B-tree page lock。

## 20. Serializable 与 predicate lock 边界

本节不展开 SSI。
但 unique insert path 里有两个入口要认识。
`index_insert()` 在 `src/backend/access/index/indexam.c:214-235`。
如果 AM 不支持 predicate locks，它会先对 index relation 做 `CheckForSerializableConflictIn()`。
B-tree AM 支持自己的 predicate lock 处理。
`_bt_doinsert()` 在 `nbtinsert.c:247-254` 在即将插入前对目标 page 做 conflict-in 检查。
`_bt_check_unique()` definite conflict 报错前也在 `nbtinsert.c:635-642` 做一次 conflict-in 检查。
注释说虽然它实际上不会写 page，但希望有机会报告会被 unique violation mask 掉的 SSI conflict。
这提醒我们：
unique violation 是一个错误路径。
但错误路径不能随意跳过其他 correctness 机制。
有些机制即使最终抛错，也要在抛错前维护可观察的事务冲突语义。

## 21. lifecycle / ownership / cleanup

executor 持有 `ResultRelInfo` 中打开的 index relations。
`ExecOpenIndices()` 在 `execIndexing.c:162-231` 打开每个 index，并拿 `RowExclusiveLock`。
如果 speculative path 需要 unique index 额外信息，`execIndexing.c:219-223` 调用 `BuildSpeculativeIndexInfo()`。
每个 index tuple 的内存由 caller 局部创建和释放。
`btinsert()` 在 `nbtree.c:215-221` 创建 `itup`，调用 `_bt_doinsert()` 后 `pfree(itup)`。
`_bt_doinsert()` 自己创建 insertion scankey。
结束时在 `nbtinsert.c:273-276` 释放 descent stack 和 scankey。
B-tree buffer lock/pin 的 ownership 很严格。
`_bt_search()` 返回 locked/pinned leaf buffer。
`_bt_check_unique()` 可能额外读右边页。
遇到 wait 时，`_bt_doinsert()` 负责释放当前 insertstate buffer。
遇到 duplicate error 时，`_bt_check_unique()` 在报错前释放所有持有的 buffers。
`_bt_insertonpg()` 成功后释放目标 buffer。
等待对象的 ownership 分两种。
普通 in-progress tuple 等 transaction lock。
这由 `XactLockTableWait()` 完成。
speculative in-progress tuple 等 speculative token lock。
这由 `SpeculativeInsertionWait()` 完成。
token lock 由 speculative inserter 在 heap speculative insert 前 acquire。
在 complete_speculative 后 release。
heap speculative tuple 的 cleanup 分两种。
成功则 `heap_finish_speculative()` 清 token，写 confirm WAL。
失败则 `heap_abort_speculative()` 把 `xmin` 设 invalid，写 delete-style WAL，并计数 heap delete。
事务 abort 的兜底不等同于 speculative abort。
`ON CONFLICT` 正常冲突不需要 abort 整个 transaction。
它要在 statement 内杀死 speculative tuple。
如果整个 transaction ERROR/abort，普通事务 abort 机制会让本事务插入的 heap/index effects 对未来 visibility 失效，后续清理再回收空间。
但源码注释强调不应带着未完成的 speculative insertion commit。
`heapam.c:6054-6060` 说明必须显式 finish 或 abort。

## 22. 错误路径与异常路径

第一类错误路径是 definite duplicate。
`UNIQUE_CHECK_YES` 下 `_bt_check_unique()` 发现 committed live conflict。
它释放 B-tree buffers。
构造 key description。
抛 `ERRCODE_UNIQUE_VIOLATION`。
本次 index tuple 没有插入。
第二类路径是 in-progress conflict。
`_bt_check_unique()` 返回 `xwait`。
`_bt_doinsert()` 释放 page lock。
等待 transaction 或 speculative token。
然后 `goto search`。
这不是错误。
这是 correctness slow path。
第三类路径是 `UNIQUE_CHECK_PARTIAL` potential conflict。
AM 不等待、不报错。
它插入 physical index tuple，并把“可能冲突”返回 executor。
如果这是 speculative insert，executor 会 complete_speculative(false) 并 retry。
如果这是 deferred unique，约束触发器或后续 recheck 会再调用 `UNIQUE_CHECK_EXISTING`。
第四类路径是 B-tree page split 中断。
`_bt_split()` 可以留下 `BTP_INCOMPLETE_SPLIT`。
后续 write search 会调用 `_bt_finish_split()`。
这是 crash/error 恢复边界，不是 unique-specific。
但 unique insert path 必须尊重它。
第五类路径是 index corruption。
`_bt_insertonpg()` 在 posting list split 时会检查 overlapping TID。
如果 plain tuple 或 LP_DEAD 状态不符合预期，`nbtinsert.c:1199-1206` 抛 index corrupted。
`_bt_binsrch_insert()` 也会在重复 table TID 不可定位时抛 index corrupted。
这些错误说明 heap TID tiebreaker 是物理不变量。
用户级 unique conflict 和物理 index corruption 是不同错误。
不要把所有 duplicate 相关错误都解释成 unique violation。

## 23. 成本、资源与跨模块传播

unique insert 的 hot path 成本主要来自四个变量。
第一，相同 key 的物理 duplicates 数量。
`_bt_check_unique()` 需要扫描 equal tuples。
如果 posting list 存在，还要逐个 heap TID 检查。
第二，heap fetch 和 HOT chain 长度。
每个 candidate TID 都要通过 `table_index_fetch_tuple_check()` 回 heap。
heap page miss、HOT chain、visibility check 和 predicate lock 都会增加成本。
第三，等待和 retry 次数。
未提交冲突越多，`_bt_doinsert()` 释放锁、等待、重搜的次数越多。
这会把成本从 CPU/page latch 转移到 lock manager、ProcArray/CLOG、调度延迟。
第四，leaf page 空间压力。
如果目标 page 空间不足，`_bt_findinsertloc()` 会尝试 deletion、bottom-up deletion、dedup。
失败后 split，产生更多 WAL 和 parent insertion。
资源传播路径也要分开。
B-tree page lock contention 影响同 key 或邻近 key 的 inserter。
transactionid lock wait 影响等待未提交普通 tuple 的 backend。
spectoken lock wait 影响等待 speculative verdict 的 backend。
WAL bandwidth 受到 heap insert、heap confirm/delete、B-tree insert/split 的共同影响。
index bloat 受到 aborted speculative tuples、failed duplicates、MVCC churn 和 delayed cleanup 影响。
checkpoint 和 replication lag 只看到 WAL 后果，不知道这是 unique check slow path 还是普通 insert burst。
所以诊断 unique insert 性能时，不能只看一个 wait event。
需要把 `pg_stat_activity` wait、`pg_locks` locktype、`pg_stat_wal`、`EXPLAIN WAL`、heap/index bloat 和 workload SQL pattern 放在一起解释。

## 24. 观测入口：能看见什么

普通 unique conflict 等待最容易看。
session 2 通常会在 `transactionid` lock 上等待 session 1。
这是 `_bt_doinsert()` 走 `XactLockTableWait()` 的现象。
你可以看到：
`pg_stat_activity.wait_event_type = 'Lock'`。
`pg_stat_activity.wait_event = 'transactionid'`。
`pg_locks` 中 waiter 对 transactionid lock 未 granted。
speculative token 等待更难手工捕获。
窗口通常很短。
如果用 gdb、injection point 或 isolation test 暂停 inserter，可以看到：
`pg_locks.locktype = 'spectoken'`。
holder 持有 `ExclusiveLock`。
waiter 请求 `ShareLock`。
这对应 `SpeculativeInsertionLockAcquire()` 和 `SpeculativeInsertionWait()`。
B-tree page lock 本身通常不通过 SQL 直接暴露成“某个 leaf page 被锁住”。
buffer content lock 等 LWLock wait 可以在 `pg_stat_activity` 中以 LWLock wait event 形式出现。
但它不携带 index block number。
如果要看到具体 page，需要 gdb、tracepoint、`pageinspect`、`pg_waldump` 或加临时日志。
WAL 类型可以通过 `pg_waldump` 看。
heap speculative insert 会带 heap insert record 和 speculative flag。
confirm 是 `XLOG_HEAP_CONFIRM`。
abort speculative 走 heap delete-style WAL。
B-tree simple insert、posting insert、upper insert、meta insert、split 都是 nbtree rmgr records。
SQL 层 `EXPLAIN (ANALYZE, WAL)` 只能看到 WAL records/bytes 的累计数量。
它不能直接告诉你某条 WAL record 是 `XLOG_HEAP_CONFIRM` 还是 `XLOG_BTREE_INSERT_LEAF`。
`pg_stat_all_tables` 可能看到 speculative insert/abort 的 insert/delete 计数影响。
但统计是异步累计，且普通 committed conflict pre-check 可能根本不会进入 heap speculative insert。
不要把这个指标当成精确因果。

## 25. 课堂实验一：普通 unique wait

目标：
看到普通 unique insert 如何等待未提交 tuple，并在 blocker commit/rollback 后走不同结果。
准备：

```sql
DROP TABLE IF EXISTS u_demo;
CREATE TABLE u_demo (k int PRIMARY KEY, payload text);
```

session 1：

```sql
BEGIN;
INSERT INTO u_demo VALUES (1, 's1');
```

不要 commit。
session 2：

```sql
INSERT INTO u_demo VALUES (1, 's2');
```

session 2 会阻塞。
session 3 观察：

```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE query LIKE '%u_demo%'
ORDER BY pid;
```

再看 lock：

```sql
SELECT pid, locktype, transactionid, mode, granted
FROM pg_locks
WHERE locktype = 'transactionid'
ORDER BY pid, granted;
```

然后分两次做结论。
第一次让 session 1：

```sql
COMMIT;
```

session 2 应报 duplicate key error。
第二次重新开始，让 session 1：

```sql
ROLLBACK;
```

session 2 应继续成功。
源码解释：
session 2 在 `_bt_check_unique()` 中通过 `SnapshotDirty` 看到 session 1 的 heap tuple。
`SnapshotDirty.xmin` 是 session 1 的 xid。
`speculativeToken == 0`。
`_bt_doinsert()` 释放 leaf page buffer，调用 `XactLockTableWait()`。
等待结束后跳回 `search` 重查。
commit 后变 definite conflict。
rollback 后 tuple 被认为 dead，session 2 可以插入。

## 26. 课堂实验二：观察 speculative token

目标：
稳定看到 `spectoken` lock。
普通 SQL 很难捕获它。
建议用 gdb 或 PostgreSQL 自带 isolation spec。
先准备表：

```sql
DROP TABLE IF EXISTS upsert_demo;
CREATE TABLE upsert_demo (k int PRIMARY KEY, payload text);
```

在 backend 进程上设置断点。
最直接的断点是：

```gdb
break heap_finish_speculative
break heap_abort_speculative
break SpeculativeInsertionWait
continue
```

session 1 执行：

```sql
INSERT INTO upsert_demo VALUES (1, 's1')
ON CONFLICT (k) DO NOTHING;
```

当 session 1 停在 `heap_finish_speculative()` 时，tuple 已经 speculative 插入，token lock 还没释放。
session 2 执行：

```sql
INSERT INTO upsert_demo VALUES (1, 's2')
ON CONFLICT (k) DO NOTHING;
```

session 2 可能停在 `SpeculativeInsertionWait()` 或在 SQL 层等待。
观察：

```sql
SELECT a.pid, a.wait_event_type, a.wait_event,
       l.locktype, l.transactionid, l.objid, l.mode, l.granted
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE l.locktype IN ('spectoken', 'transactionid')
ORDER BY a.pid, l.locktype, l.granted;
```

你应该看到 `spectoken`。
holder 是 speculative inserter 的 `ExclusiveLock`。
waiter 是另一个 backend 的 `ShareLock`。
源码解释：
session 1 在 `nodeModifyTable.c:1232` 获取 token lock。
`heapam_handler.c:180` 把 token 写入 tuple header。
session 2 的 `_bt_check_unique()` 通过 `HeapTupleSatisfiesDirty()` 读到 `SnapshotDirty.speculativeToken`。
`_bt_doinsert()` 调用 `SpeculativeInsertionWait()`。
session 1 complete_speculative 后释放 token lock。
session 2 醒来重查。
如果不想手动 gdb，可以看源码自带 isolation spec：

```text
src/test/isolation/specs/insert-conflict-specconflict.spec
```

这个测试用 advisory lock 控制两个 session 的进度。
其中 `controller_print_speculative_locks` 查询 `pg_locks` 的 `spectoken` 和 `transactionid`。
这比手工 SQL 更稳定。

## 27. 课堂实验三：WAL 与错误边界

目标：
区分 duplicate error 前没有 B-tree insert，和 speculative abort 后留下可清理 index pointer。
普通 duplicate：

```sql
DROP TABLE IF EXISTS wal_u_demo;
CREATE TABLE wal_u_demo (k int PRIMARY KEY, payload text);
INSERT INTO wal_u_demo VALUES (1, 'ok');
EXPLAIN (ANALYZE, WAL)
INSERT INTO wal_u_demo VALUES (1, 'dup');
```

第二条会报 unique violation。
因为报错，`EXPLAIN` 不会像正常完成语句一样给你完整输出。
这本身就是一个边界：
definite conflict 在 `_bt_check_unique()` 报错。
本次 `_bt_insertonpg()` 不执行。
如果要看 WAL record，需要在源码断点或 `pg_waldump` 中对比成功 insert 与报错 insert。
speculative abort：
用实验二的方法制造 concurrent `ON CONFLICT DO NOTHING`。
在 gdb 中观察：

```gdb
break heap_abort_speculative
break heap_finish_speculative
break _bt_check_unique
break _bt_insertonpg
```

重点记录顺序。
可能的顺序是：

```text
heap speculative insert
btree index insert with UNIQUE_CHECK_PARTIAL
conflict reported to executor
heap_abort_speculative
SpeculativeInsertionLockRelease
retry or ON CONFLICT action
```

源码解释：
普通 duplicate 的 error boundary 在 B-tree insertion 前。
speculative conflict 的 cleanup boundary 在 heap complete_speculative(false)。
两者都能产生“用户看到没有插入第二行”的结果。
但 WAL、index garbage 和 stats 行为不同。

## 28. gdb 源码跟读练习

建议断点：

```gdb
break ExecInsertIndexTuples
break _bt_doinsert
break _bt_check_unique
break table_index_fetch_tuple_check
break HeapTupleSatisfiesDirty
break SpeculativeInsertionWait
break heap_finish_speculative
break heap_abort_speculative
```

普通 unique wait 时，画出：

```text
session 1 heap tuple xmin = xid1
session 2 _bt_check_unique sees SnapshotDirty.xmin = xid1
session 2 speculativeToken = 0
session 2 waits transactionid
session 2 retries _bt_search_insert
```

speculative wait 时，画出：

```text
session 1 token lock = (xid1, tokenN)
session 1 heap tuple t_ctid = speculative token
session 2 SnapshotDirty.xmin = xid1
session 2 SnapshotDirty.speculativeToken = tokenN
session 2 waits spectoken
session 1 confirms or aborts tuple
session 2 retries
```

在 `_bt_check_unique()` 里重点看三个变量：

```text
SnapshotDirty.xmin
SnapshotDirty.xmax
SnapshotDirty.speculativeToken
```

在 `_bt_doinsert()` 里重点看：

```text
checkUnique
checkingunique
itup_key->scantid
insertstate.buf
xwait
speculativeToken
```

在 heap confirm/abort 里重点看：

```text
tuple->t_ctid
tuple xmin
WAL record type
page LSN
```

不要只看函数调用栈。
要把每个断点上的状态写成时间线。

## 29. 常见误区

误区一：
unique index leaf page 里不会有重复 key。
实际是：
leaf page 可以有相同用户 key 的多个 physical entries。
用户级唯一性来自 heap visibility check。
误区二：
`UNIQUE_CHECK_PARTIAL` 是 partial index。
实际是：
它是“不等待、不报错，只返回可能冲突”的 unique check 模式。
误区三：
speculative token 存在 index tuple 里。
实际是：
token 存在 heap tuple header 的 `t_ctid` 特殊编码里。
index check 通过 dirty heap visibility 读到 token。
误区四：
`ON CONFLICT` pre-check 已经保证不会冲突。
实际是：
pre-check 不持锁。
它只减少 canceled speculative insertions。
正确性来自 speculative insert 后的 index AM check 和 confirm/abort 协议。
误区五：
发现 conflict 后可以持有 B-tree leaf lock 等事务结束。
实际是：
源码明确释放 buffer 后等待。
等待后重新搜索。
误区六：
WAL 保证 unique constraint。
实际是：
WAL 保证 heap/index page 修改可恢复。
unique constraint 的 runtime correctness 来自 B-tree key-range serialization、heap visibility、transaction wait、speculative token 和 retry。
误区七：
`SnapshotDirty` 返回 true 就表示 tuple 对当前 query 可见。
实际是：
它把 in-progress tuple 也作为 potential conflict 暴露给 unique check。
普通 MVCC visibility 语义不同。
误区八：
duplicate key error 表示本次已经写了 index tuple。
实际是：
`UNIQUE_CHECK_YES` definite duplicate 在 `_bt_insertonpg()` 前报错。
speculative conflict 则可能已经写了 index tuple，但 heap tuple 被 abort_speculative 杀死。

## 30. 讨论题

1. 为什么 unique check 期间要把 `itup_key->scantid` 设为 NULL？
2. 为什么等事务结束前必须释放 B-tree buffer lock？
3. 等待后如果不重新 `_bt_search_insert()`，可能错过哪些并发变化？
4. `SnapshotDirty.xmin`、`SnapshotDirty.xmax` 和 `SnapshotDirty.speculativeToken` 分别表达什么未定状态？
5. 为什么 `ON CONFLICT` 需要 speculative token，而不能只等 transactionid？
6. speculative tuple abort 后，为什么不要求立即删除已经写入的 index tuple？
7. B-tree insert WAL 和 heap confirm WAL 分别保护什么？
8. 如果 `pg_stat_activity` 只看到 `transactionid` wait，你能断言瓶颈一定在 B-tree page lock 吗？

## 31. 本节小结

本节的核心链路是：
heap tuple 先获得 TID。
executor 为每个 index 选择 unique check 模式。
B-tree insert 在 unique check 时临时去掉 heap TID tiebreaker，锁住第一个可能有 duplicate 的 leaf page。
`_bt_check_unique()` 扫描同 key physical entries，并用 `SnapshotDirty` 回 heap 判断 live、dead、in-progress 或 speculative。
遇到未提交冲突就释放 page lock，等待 transactionid 或 spectoken，然后从 root 重搜。
唯一性建立后恢复 heap TID tiebreaker，找到物理位置并在 critical section 内修改 B-tree page、写 WAL、设置 LSN。
speculative insertion 的核心状态在 heap tuple。
token 写在 `t_ctid` 的特殊编码里。
lock manager 用 `(xid, token)` 的 `LOCKTAG_SPECULATIVE_TOKEN` 让其他 backend 等待 verdict。
confirm path 清 token 并写 `XLOG_HEAP_CONFIRM`。
abort path 把 `xmin` 设 invalid，清 token，写 delete-style WAL。
本节最重要的不变量是：
physical duplicate 不是 unique violation。
只有 dirty heap visibility 证明存在另一个会与本 tuple 同时 live 的 row version，才是 conflict。
未定状态必须等待并重查。
等待不能持有 B-tree page lock。
错误边界要分清。
普通 immediate duplicate 在 B-tree physical insert 前抛错。
speculative conflict 可以先写 heap 和 index，再杀死 heap tuple，把失败限制在本次 speculative insertion。
WAL 保护的是 page change 和 crash recovery 顺序，不单独证明唯一性。
可观测入口也要分清。
普通 unique wait 通常看 `transactionid`。
speculative wait 要看 `spectoken`，通常需要 gdb、injection point 或 isolation test 稳定复现。
`EXPLAIN WAL` 只能看聚合 WAL 成本。
`pg_locks` 能看等待对象。
具体 heap tuple token 和 B-tree page 状态往往只能靠断点、pageinspect 或 WAL dump 推断。
可迁移的系统规律是：
当一个局部数据结构要维护全局语义时，不要把全局等待塞进局部 latch。
PostgreSQL 的做法是把状态拆成可识别的中间态，用短锁保护局部结构，用 visibility 和 transaction/token wait 处理未定结论，用 WAL 保护物理修改，再用 retry 把所有并发变化重新纳入判断。
这个规律会在后续 index cleanup、concurrent index build、SSI 和 logical decoding 中反复出现。
