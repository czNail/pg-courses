# PostgreSQL B-tree WAL 与结构变化 redo
## 课程定位
本节主题：B-tree WAL 与结构变化 redo。
上一节已经讲过 B-tree leaf deletion、deduplication 与 bottom-up cleanup。
再上一节讲过 page split、parent insertion 与 incomplete split。
现在把视角切到 crash recovery：
前台结构变化已经发生，WAL 已经写入。
系统崩溃后，startup process 只看到 WAL record 和磁盘页。
它不能重新执行一次 `_bt_doinsert()` 或 `_bt_pagedel()`。
它只能根据每条 record 的 redo contract 把页恢复到一个可搜索、可继续修复的状态。

前置知识：
- 已理解 buffer dirty、page LSN、WAL-before-data。
- 已理解 WAL resource manager redo 的基本契约。
- 已理解 B-tree page 的 high key、rightlink、leftlink。
- 已理解 split 不是单个跨层原子动作。
- 已理解 `BTP_INCOMPLETE_SPLIT` 表示右 sibling 的 parent downlink 还没补。
- 已理解 leaf tuple deletion、deduplication、posting list 的基本形状。
- 已理解 VACUUM page deletion 会经历 half-dead、deleted、FSM reuse。

本节唯一主问题：
B-tree 的 page split、new root、page deletion、dedup、posting list split 和 page reuse 都会改变多个页或改变页内物理布局；PostgreSQL 如何把这些结构变化拆成有限大小的 WAL 原子动作，并让 redo 后的每一步都既保持搜索正确，又能在后续前台路径或 VACUUM 中继续收尾？

本节围绕的核心矛盾：
B-tree 结构变化天然跨页、跨层。
一个 leaf split 至少涉及左页、新右页、旧右 sibling、父页，甚至 metapage 和更高层。
一次 page deletion 至少涉及 parent downlink、half-dead leaf、左右 sibling、deleted tombstone、metapage fast root、FSM reuse horizon。
如果把整个结构变化写成一个巨大的 WAL record，锁范围、WAL 体积、FPI 概率、错误恢复和级联 split 都会失控。
如果把每个物理页修改都写成互不相关的小 record，crash recovery 中间状态又可能让搜索找不到 key range，或让后续写路径无法判断结构债务。

PostgreSQL 的选择是：
```text
把结构变化拆成一组可 redo 的 WAL 原子动作。
每个原子动作完成后，B-tree 对读者保持可搜索。
跨层未完成的结构债务用 durable page state 表达。
后续写路径或 VACUUM 按状态继续修复，而不是依赖 recovery 记账。
```

学完后你应该能独立判断：
- `btree_redo()` 为什么按 record type dispatch，而不是调用前台算法。
- 为什么 split redo 重建右页、重建左页，却不插入 parent downlink。
- 为什么 `XLOG_BTREE_INSERT_UPPER` 和 `XLOG_BTREE_NEWROOT` 要清 child 的 `BTP_INCOMPLETE_SPLIT`。
- 为什么 redo 中可以放松跨层 lock coupling，但不能随意放松同层 sibling link 更新。
- 为什么 posting list split redo 需要 `_bt_swap_posting()` 重构最终 tuple。
- 为什么 dedup redo 记录的是 interval，而不是整页逻辑描述。
- 为什么 page deletion redo 分成 mark half-dead 和 unlink page 两类 record。
- 为什么 deleted page 复用还要有 `XLOG_BTREE_REUSE_PAGE` 这种不改页的 WAL record。
- 为什么 `btree_mask()` 要屏蔽部分 hint 或 recovery 不重放的差异。
- 哪些状态可用 `pageinspect`、`pg_waldump`、日志、断点看到，哪些只能从现象推断。

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
src/backend/access/nbtree/nbtxlog.c
src/backend/access/nbtree/nbtinsert.c
src/backend/access/nbtree/nbtpage.c
src/backend/access/nbtree/nbtdedup.c
src/include/access/nbtxlog.h
src/include/access/nbtree.h
src/backend/access/nbtree/README
```

行号来自：
```text
nl -ba <source-file>
```

本节不把 `nbtxlog.c` 写成 rmgr API 清单。
我们只关心一个问题：
WAL record 如何表达 B-tree 结构变化的最小 durable 状态，redo 如何把这个状态恢复成读者可接受、写者可继续推进的树。

## 1. 本节在总主线中的位置
前面几节已经建立了三条线。
第一条是 buffer/WAL 线：
数据页先在 shared buffer 中被修改。
修改前后必须有 WAL-before-data。
crash 后 redo 按 page LSN 判断是否需要重放。

第二条是 B-tree 并发结构线：
search 不需要依赖一个永远完美的 parent downlink 集合。
high key 和 rightlink 让 search 能从旧路径移动到正确页面。
这允许 split 先完成本层物理修改，再补 parent downlink。

第三条是 cleanup 线：
index tuple deletion、dedup、page deletion 和 FSM reuse 都不是事务 abort 时简单撤销。
它们维护的是物理结构和未来可达性。
它们必须 crash-safe，也必须和 standby snapshot 冲突处理对齐。

本节把这三条线合在一起：
```text
前台结构变化
  -> 写出特定 B-tree WAL record
  -> crash 后 startup process 调用 btree_redo()
  -> redo 按 record 恢复页状态
  -> 每个恢复后的状态都允许 search、insert、VACUUM 继续
```

重点不是“有哪些 op code”。
重点是每类 op code 承担哪一段结构原子性。
`XLOG_BTREE_SPLIT_L/R` 承担本层 split。
`XLOG_BTREE_INSERT_UPPER` 承担 parent downlink insert。
`XLOG_BTREE_NEWROOT` 承担新 root 和 metapage 切换。
`XLOG_BTREE_MARK_PAGE_HALFDEAD` 承担从 parent 移除 downlink，并把 leaf 标成 half-dead。
`XLOG_BTREE_UNLINK_PAGE` 承担同层 sibling 链修改和 tombstone 建立。
`XLOG_BTREE_REUSE_PAGE` 不改页，却承担 Hot Standby conflict point。

这一节的运行模型可以压缩成一句话：
```text
B-tree redo 重放的是“结构状态转移”，不是“前台控制流”；它依靠 high key/rightlink、durable flags、LSN gating 和后续 lazy repair，把跨页结构变化拆成多个可恢复的局部原子动作。
```

## 2. 核心矛盾与一句话运行模型
先看一个 leaf split。
前台 `_bt_split()` 会把原页变成左页。
它会分配并初始化右页。
它会改旧右 sibling 的 leftlink。
它会把左页标成 `BTP_INCOMPLETE_SPLIT`。
然后 `_bt_insert_parent()` 再把右页 downlink 插入 parent。
如果 parent 也满，还会递归 split parent。
如果原页是 root，还会走 `_bt_newlevel()` 创建新 root。

这里无法把所有事情塞进一个不可失败的大动作。
原因很实际：
parent insertion 可能继续 split。
split parent 又可能继续 split grandparent。
需要分配新页。
需要写多个 WAL record。
需要遵守 lock order。
还可能在补 parent downlink 前遇到 ERROR、OOM、out-of-disk 或 crash。

所以 split 的 durable 表达分成两类：
```text
本层 split record：
  左页和右页已经形成 sibling pair。
  search 可通过 rightlink 找到新右页。
  左页带 BTP_INCOMPLETE_SPLIT。

后续 parent insertion/newroot record：
  parent 拥有新右页 downlink。
  同一 WAL 原子动作清掉左 child 的 BTP_INCOMPLETE_SPLIT。
```

page deletion 也类似。
不能直接把空叶子页从所有路径里消失。
先从 parent 移除 downlink，把 leaf 标成 half-dead。
half-dead leaf 仍在 sibling chain 中。
后续再 unlink target page，建立 deleted tombstone。
deleted page 还不能马上复用。
只有当 `safexid` 可被全局可见性判断认为足够旧，VACUUM 才能把它放进 FSM。
在 standby 上，真正复用前还要用 `XLOG_BTREE_REUSE_PAGE` 和 snapshot conflict 对齐。

本节的核心矛盾是：
```text
结构变化需要跨页完整性。
WAL redo 只能可靠重放有限大小的原子动作。
读路径必须持续可用。
写路径必须能识别并修复未完成结构债务。
```

PostgreSQL 的工程答案不是追求“每时每刻 parent tree 都完美”。
它追求的是：
```text
每个 WAL record 结束时，读者能找到正确 key range；
写者看到结构债务时有明确修复入口；
redo 不需要保存额外全局待修列表；
page LSN 和 record block refs 足以让重放幂等。
```

这个模型比“B-tree 修改要么全部成功要么全部失败”更接近真实源码。
B-tree 结构变化可以局部完成。
局部完成状态本身就是合法状态。
合法不等于最终状态。
`BTP_INCOMPLETE_SPLIT`、`BTP_HALF_DEAD`、`BTP_DELETED` 和 `safexid` 都是在表达这种中间但可恢复的合法状态。

## 3. 核心文件分工与源码阅读顺序
推荐按 redo contract 读，而不是按文件名顺序读。

| 顺序 | 文件 | 读什么 | 目的 |
| --- | --- | --- | --- |
| 1 | `src/backend/access/nbtree/README` | WAL Considerations、Scans during Recovery、page deletion、posting list splits | 先建立 record 粒度为什么这样拆 |
| 2 | `src/include/access/nbtree.h` | `BTPageOpaqueData`、`BTMetaPageData`、`BTDeletedPageData`、`BTDedupStateData` | 明确状态放在 page、metapage、backend-local workspace 还是 WAL payload |
| 3 | `src/include/access/nbtxlog.h` | `XLOG_BTREE_*` op code 和 `xl_btree_*` record | 明确 WAL 能表达什么、不能表达什么 |
| 4 | `src/backend/access/nbtree/nbtinsert.c` | `_bt_insertonpg()`、`_bt_split()`、`_bt_insert_parent()`、`_bt_newlevel()`、`_bt_finish_split()` | 找前台 split 和 newroot 的 WAL 生成点 |
| 5 | `src/backend/access/nbtree/nbtpage.c` | `_bt_mark_page_halfdead()`、`_bt_unlink_halfdead_page()`、`_bt_allocbuf()`、`_bt_pendingfsm_finalize()` | 找 page deletion、tombstone、reuse record 的生成点 |
| 6 | `src/backend/access/nbtree/nbtdedup.c` | `_bt_dedup_pass()`、`_bt_update_posting()`、`_bt_swap_posting()` | 找 dedup/posting list split 的物理重写规则 |
| 7 | `src/backend/access/nbtree/nbtxlog.c` | `btree_redo()` 和各 `btree_xlog_*()` | 最后读 redo 如何只用 WAL 和旧页状态恢复 |

`README` 中最关键的段落是 `WAL Considerations`。
它明确说单个 WAL entry 是一个 atomic action。
split 是本层 split record 后跟 parent insertion record。
root split 的 follow-on record 是 `new root`。
如果 crash 发生在两者之间，missing downlink 是允许存在的。
后续 insert 会 on-the-fly 建立 missing downlink。

`Scans during Recovery` 是第二个关键段落。
redo 不需要重建前台的所有锁耦合。
原因是 standby/recovery 里没有并发 writer。
读者不关心 incomplete split flag。
但 redo 仍然保守地维护同层锁顺序，避免读者观察到 sibling chain 的内部不一致。

最短源码主线：
```text
nbtinsert.c
  _bt_insertonpg()
    -> _bt_split()
       -> XLOG_BTREE_SPLIT_L/R
    -> _bt_insert_parent()
       -> _bt_insertonpg(parent)
          -> XLOG_BTREE_INSERT_UPPER / INSERT_META
       -> _bt_newlevel()
          -> XLOG_BTREE_NEWROOT

nbtxlog.c
  btree_redo()
    -> btree_xlog_split()
    -> btree_xlog_insert()
    -> btree_xlog_newroot()
```

page deletion 主线：
```text
nbtpage.c
  _bt_pagedel()
    -> _bt_mark_page_halfdead()
       -> XLOG_BTREE_MARK_PAGE_HALFDEAD
    -> _bt_unlink_halfdead_page()
       -> XLOG_BTREE_UNLINK_PAGE / UNLINK_PAGE_META
    -> _bt_pendingfsm_add()
  _bt_pendingfsm_finalize()
    -> RecordFreeIndexPage()
  _bt_allocbuf()
    -> XLOG_BTREE_REUSE_PAGE when reusing deleted page for Hot Standby

nbtxlog.c
  btree_xlog_mark_page_halfdead()
  btree_xlog_unlink_page()
  btree_xlog_reuse_page()
```

dedup/posting 主线：
```text
nbtdedup.c
  _bt_dedup_pass()
    -> XLOG_BTREE_DEDUP
  _bt_update_posting()
  _bt_swap_posting()

nbtinsert.c
  _bt_insertonpg()
    -> XLOG_BTREE_INSERT_POST
  _bt_split()
    -> XLOG_BTREE_SPLIT_L/R with postingoff when needed

nbtxlog.c
  btree_xlog_dedup()
  btree_xlog_insert(posting=true)
  btree_xlog_split()
```

读源码时不要从 `btree_redo()` 的 switch 直接背 op code。
先问每个 record 的结构原子性边界：
它修改哪些 block refs？
哪些页可能被 FPI 覆盖？
哪些 payload 必须在没有 FPI 时用于重建？
redo 为什么能幂等？
redo 后哪个 durable state 表示“后续还要收尾”？

## 4. 关键数据结构与状态
第一组状态在 page special area。
`nbtree.h:63` 定义 `BTPageOpaqueData`。
本节需要的字段：
```text
btpo_prev
btpo_next
btpo_level
btpo_flags
btpo_cycleid
```

`btpo_prev` 和 `btpo_next` 是同层 sibling chain。
它们让 search、scan、page deletion、redo 都能在 parent downlink 不完整时移动。
`btpo_level` 区分 leaf 和 internal page。
`btpo_flags` 才是结构状态的核心。
`btpo_cycleid` 主要服务 VACUUM 和 split group，本节只需要知道 redo 会在某些场景把它清零或 mask。

关键 flags 在 `nbtree.h:77-84`：
```text
BTP_LEAF
BTP_ROOT
BTP_DELETED
BTP_META
BTP_HALF_DEAD
BTP_SPLIT_END
BTP_HAS_GARBAGE
BTP_INCOMPLETE_SPLIT
BTP_HAS_FULLXID
```

不要把 flag 单独解释成完整语义。
`BTP_INCOMPLETE_SPLIT` 只有和 `btpo_next`、left page high key、parent downlink 缺口一起解释才有意义。
`BTP_HALF_DEAD` 只有和 leaf high key 里保存的 top parent link 一起解释才有意义。
`BTP_DELETED` 只有和 page contents 里的 `BTDeletedPageData.safexid` 一起解释才有意义。
`BTP_HAS_GARBAGE` 是未 WAL 记录的 hint，redo consistency check 会 mask。

第二组状态在 metapage。
`nbtree.h:104` 定义 `BTMetaPageData`。
核心字段：
```text
btm_root
btm_level
btm_fastroot
btm_fastlevel
btm_last_cleanup_num_delpages
btm_allequalimage
```

`btm_root` 是 true root。
`btm_fastroot` 是 ordinary search 可直接开始的 effective root。
page deletion 不降低树高，但可以让 lower level 成为 fast root。
root split 或 fast root 调整必须 WAL 记录。
redo 通过 `_bt_restore_meta()` 重建 metapage。
它不是逐字段增量 patch。
它用 `xl_btree_metadata` 重新初始化 metapage image。

第三组状态在 WAL record。
`nbtxlog.h:27-43` 定义 op code：
```text
XLOG_BTREE_INSERT_LEAF
XLOG_BTREE_INSERT_UPPER
XLOG_BTREE_INSERT_META
XLOG_BTREE_SPLIT_L
XLOG_BTREE_SPLIT_R
XLOG_BTREE_INSERT_POST
XLOG_BTREE_DEDUP
XLOG_BTREE_DELETE
XLOG_BTREE_UNLINK_PAGE
XLOG_BTREE_UNLINK_PAGE_META
XLOG_BTREE_NEWROOT
XLOG_BTREE_MARK_PAGE_HALFDEAD
XLOG_BTREE_VACUUM
XLOG_BTREE_REUSE_PAGE
XLOG_BTREE_META_CLEANUP
```

这些 op code 不是按函数一一对应。
它们按“可 redo 的结构动作”划分。
一个 `_bt_insertonpg()` 可能写 leaf insert、posting insert、upper insert、insert meta。
一个 `_bt_split()` 写 split record。
一个 `_bt_newlevel()` 写 newroot record。
一个 `_bt_unlink_halfdead_page()` 可能写 unlink 或 unlink meta。

第四组状态在 backend-local workspace。
`BTStackData` 在 `nbtree.h:743`。
它是前台 descent stack，用来回找 parent downlink。
redo 不依赖它。
crash 后 stack 消失。
所以 WAL record 必须携带 redo 所需的 page image/payload，而不能期待重跑 descent。

`BTInsertStateData` 在 `nbtree.h:820`。
其中 `postingoff` 表示 incoming tuple 落在 existing posting list 内部。
前台用它触发 `_bt_swap_posting()`。
redo 的 `xl_btree_insert` 或 `xl_btree_split` 也需要记录 `postingoff`，否则无法重建 posting list split 的最终物理布局。

`BTDedupStateData` 在 `nbtree.h:876`。
它是 dedup pass 的工作区。
`BTDedupInterval` 记录从哪个 offset 开始、合并多少 tuple。
redo dedup 不是执行完整策略选择。
它使用 WAL 中的 interval 重放已经决定好的合并结果。

第五组状态是 deleted page tombstone。
`BTDeletedPageData` 在 `nbtree.h:236`。
`BTPageSetDeleted()` 在 `nbtree.h:240` 把 page 标成 deleted，并把 `safexid` 写到 page contents。
`BTPageIsRecyclable()` 在 `nbtree.h:292` 用 `GlobalVisCheckRemovableFullXid()` 判断 deleted page 是否可复用。
这解释了为什么 page deletion redo 不能把 page 直接清成 free page。
deleted page 需要保留一段时间，让可能持有旧链接的 scan/search 能看到 tombstone 并继续移动。

## 5. WAL record 粒度：哪些动作是原子的
`nbtxlog.h` 是理解 redo 的入口。
先按粒度分类。

单页 leaf insert：
```text
XLOG_BTREE_INSERT_LEAF
```
它只给 leaf page 加一个 index tuple。
redo 用 `PageAddItem()` 把 payload 加回目标 offset。

单页 leaf insert 但伴随 posting list split：
```text
XLOG_BTREE_INSERT_POST
```
它不是简单插入一个 tuple。
它还要在同一页原地替换 old posting tuple。
redo 需要先读取现有 posting tuple，再用 WAL 中的原始 newitem 和 `postingoff` 调 `_bt_swap_posting()`，得到 replacement posting tuple 和最终 newitem。

internal page insert：
```text
XLOG_BTREE_INSERT_UPPER
```
它不仅在 parent/internal page 插入 downlink。
它还清 child left page 的 `BTP_INCOMPLETE_SPLIT`。
这是 parent downlink insert 的结构原子性。

internal page insert 同时更新 metapage fast root：
```text
XLOG_BTREE_INSERT_META
```
它覆盖 `INSERT_UPPER` 的语义，并且用 `xl_btree_metadata` 重建 metapage。
这个场景来自非 root、但该 level 原来只有一个 page 的 split。

本层 page split：
```text
XLOG_BTREE_SPLIT_L
XLOG_BTREE_SPLIT_R
```
两者共享 `xl_btree_split`。
差异是 incoming newitem 最终在左页还是右页。
split record 的原子性是：
左页变窄并带 new high key。
新右页完整建立。
旧右 sibling 的 leftlink 更新。
如果 split 的是 internal page，还顺便清下一层 child incomplete split。
但它不插入当前右页的 parent downlink。

新 root：
```text
XLOG_BTREE_NEWROOT
```
它建立 root page，清 left child incomplete split，并重建 metapage。
它是 root split 的第二阶段。

dedup：
```text
XLOG_BTREE_DEDUP
```
它把 leaf page 上连续 duplicate tuple 合并成 posting list。
它记录 interval 数组，而不是记录整页所有 tuple。
redo 重新走 `_bt_dedup_start_pending()`、`_bt_dedup_save_htid()`、`_bt_dedup_finish_pending()` 的构造逻辑。

leaf tuple deletion：
```text
XLOG_BTREE_VACUUM
XLOG_BTREE_DELETE
```
两者都表达删除 leaf index tuple 或更新 posting tuple。
`DELETE` 带 standby snapshot conflict horizon。
`VACUUM` 不需要单独带这个字段，因为 heap pruning/VACUUM 相关冲突已经在别处处理。

page deletion 第一阶段：
```text
XLOG_BTREE_MARK_PAGE_HALFDEAD
```
它改 parent page 的 downlink/key arrangement，并把 empty leaf 重写成 half-dead leaf。

page deletion 第二阶段：
```text
XLOG_BTREE_UNLINK_PAGE
XLOG_BTREE_UNLINK_PAGE_META
```
它改左右 sibling links，把 target 重写成 deleted tombstone。
如果 fast root 改变，还包含 metapage 重建。

page reuse conflict point：
```text
XLOG_BTREE_REUSE_PAGE
```
这条 record 不注册 buffer。
它只在 Hot Standby 中调用 `ResolveRecoveryConflictWithSnapshotFullXid()`。
它的存在说明 WAL 不只是“把页改回来”。
WAL 还负责把 primary 上的可见性边界传播到 standby。

metapage cleanup：
```text
XLOG_BTREE_META_CLEANUP
```
它更新 cleanup 相关 metapage 字段。
redo 仍然用 `_bt_restore_meta()` 重建 metapage image。

## 6. 主流程 walkthrough：一次 split 的 redo
先从前台写 WAL 的位置看。
`_bt_insertonpg()` 在 `nbtinsert.c:1119`。
它先判断目标页是否需要 split。
如果 `PageGetFreeSpace(page) < itemsz`，它调用 `_bt_split()`。
`_bt_split()` 在 `nbtinsert.c:1489`。
它选择 split point，准备 left temp page 和 right temp page，分配新右页，更新 sibling links，并进入 critical section。

前台 `_bt_split()` 在 `nbtinsert.c:1582` 给 left page 设置 `BTP_INCOMPLETE_SPLIT`。
在 `nbtinsert.c:2011-2083` 组装 `xl_btree_split` 和 block data。
核心 payload 是：
```text
xlrec.level
xlrec.firstrightoff
xlrec.newitemoff
xlrec.postingoff
left page optional newitem/orignewitem
left page new high key
right page tuple bytes
```

redo 入口是 `btree_redo()`。
`nbtxlog.c:1004` 按 `XLogRecGetInfo(record)` dispatch。
`XLOG_BTREE_SPLIT_L` 调 `btree_xlog_split(true, record)`。
`XLOG_BTREE_SPLIT_R` 调 `btree_xlog_split(false, record)`。

`btree_xlog_split()` 在 `nbtxlog.c:247`。
它先取 block 0 的 original/left block number，block 1 的 right block number，block 2 的 old right sibling。
如果 split 的页是 internal page，还会先调用 `_bt_clear_incomplete_split(record, 3)`。
这是因为 internal split 可能同时完成下一层 child 的 incomplete split。
这不是当前 split 的 flag。
当前 split 的 flag 仍然保留在 left page，等待 parent insertion 或 newroot record 清除。

然后 redo 重建新右页。
`XLogInitBufferForRedo(record, 1)` 初始化 block 1。
redo 设置 right page opaque：
```text
btpo_prev = origpagenumber
btpo_next = spagenumber
btpo_level = xlrec->level
btpo_flags = BTP_LEAF 或 0
btpo_cycleid = 0
```
然后 `_bt_restore_page()` 把 block data 中的 tuple bytes 按 item-number order 还原。
`_bt_restore_page()` 在 `nbtxlog.c:36`。
它先扫描 tuple size，再反向 `PageAddItem()`。
原因是 WAL payload 保存的是 page upper area 的 tuple bytes。
line pointer 不直接保存，redo 必须重建 line pointer order。

接着 redo 重建左页。
如果 `XLogReadBufferForRedo(record, 0, &buf)` 返回 `BLK_NEEDS_REDO`，才需要修改 block 0。
这就是 redo 幂等边界：
磁盘页 LSN 已经足够新，就跳过这个 block。
需要重做时，redo 从旧 left/orig page 读取未移走的旧 item。
再结合 WAL payload 中的 left high key 和 optional newitem，构造 temp left page。

左页重建有一个细节：
如果 split 同时涉及 posting list split，redo 需要用 `_bt_swap_posting()` 重构 replacement posting tuple。
`xlrec->postingoff` 非零时，WAL 中的 newitem 是 orignewitem。
redo 从旧页取 old posting tuple。
调用 `_bt_swap_posting(newitem, oposting, postingoff)`。
得到 `nposting` 和被修改成最终 TID 的 `newitem`。
这样 redo 生成的左页物理布局与前台一致。

最后 redo 修 left opaque：
```text
btpo_flags = BTP_INCOMPLETE_SPLIT
if leaf: add BTP_LEAF
btpo_next = rightpagenumber
btpo_cycleid = 0
```
注意 redo 不设置 `BTP_SPLIT_END`。
`btree_mask()` 后面也会 mask 掉 `BTP_SPLIT_END` 和 `btpo_cycleid`，因为 recovery 里不完全重现这些 VACUUM optimization 细节。

如果原 left page 之前有 right sibling，redo 再更新那个 sibling 的 `btpo_prev`。
这一步是同层 sibling chain 的原子性。
`btree_xlog_split()` 最后才释放 right page、left page、old right sibling。
README 说 redo 对同层锁仍然偏保守，避免 recovery 中读者看到同层链接不一致。

split redo 结束后，树可能处于这个状态：
```text
parent 没有指向 rightpagenumber 的 downlink
left page 的 rightlink 指向 rightpagenumber
left page 带 BTP_INCOMPLETE_SPLIT
right page 已经包含正确 key range
search 可通过 left high key 发现应向右移动
```
这不是损坏。
这是合法的持久中间态。

## 7. 主流程 walkthrough：parent insertion 和 newroot redo
split 的第二阶段来自 `_bt_insert_parent()`。
`_bt_insert_parent()` 在 `nbtinsert.c:2130`。
它拿 left page 的 high key，复制成 parent 中的新 pivot tuple，并把 downlink 改成 right page block number。
如果 split 的是非 root page，就调用 `_bt_insertonpg()` 在 parent page 插入这个 downlink。
如果 split 的是 true root，就调用 `_bt_newlevel()`。

普通 parent insert 的 WAL 在 `_bt_insertonpg()` 的非 split 分支。
`nbtinsert.c:1328` 清 child page 的 `BTP_INCOMPLETE_SPLIT`。
`nbtinsert.c:1362` 选择 `XLOG_BTREE_INSERT_UPPER`。
如果还更新 metapage fast root，`nbtinsert.c:1368` 改成 `XLOG_BTREE_INSERT_META`。
`nbtinsert.c:1409` 调 `XLogInsert()`。

redo 路径是 `btree_xlog_insert()`。
`nbtxlog.c:158` 定义它。
当 `isleaf == false` 时，`nbtxlog.c:176` 先调用 `_bt_clear_incomplete_split(record, 1)`。
block 1 是 child left sibling。
`_bt_clear_incomplete_split()` 在 `nbtxlog.c:137`。
它读 child block。
如果需要 redo，就 assert page 有 incomplete flag，并清掉该 bit。
然后设置 child page LSN、MarkBufferDirty。

之后 `btree_xlog_insert()` 才 redo parent page 的 `PageAddItem()`。
前台路径中清 child flag 和插入 parent downlink 是同一个 critical section、同一个 WAL record。
redo 里虽然不保持跨层 lock coupling，但 record 的原子性仍然表达这两个状态一起完成。

`XLOG_BTREE_INSERT_META` 还会调用 `_bt_restore_meta(record, 2)`。
`_bt_restore_meta()` 在 `nbtxlog.c:80`。
它用 `XLogInitBufferForRedo()` 初始化 metapage，调用 `_bt_pageinit()`，填 `BTMetaPageData`，设置 `BTP_META`，并修 `pd_lower`。
这和前台 `_bt_initmetapage()` 对 `pd_lower` 的要求一致。
否则 xlog page compression 可能丢掉 metadata bytes。

root split 的第二阶段是 `_bt_newlevel()`。
`_bt_newlevel()` 在 `nbtinsert.c:2492`。
它分配新 root page，拿 metapage write lock，创建两个 pivot tuple：
左 child 用 minus infinity item 指向 old root。
右 child 用 left page high key 指向 new right page。
然后更新 metapage root/fastroot。
它在 `nbtinsert.c:2597` 清 old root left child 的 `BTP_INCOMPLETE_SPLIT`。
在 `nbtinsert.c:2639` 写 `XLOG_BTREE_NEWROOT`。

redo 路径是 `btree_xlog_newroot()`。
`nbtxlog.c:927` 定义它。
redo 先初始化 new root page。
如果 `xlrec->level > 0`，用 `_bt_restore_page()` 还原两个 root tuple。
然后 `_bt_clear_incomplete_split(record, 1)` 清 left child flag。
最后 `_bt_restore_meta(record, 2)` 重建 metapage。

这里有一个重要边界：
new root redo 不需要重新判断“这个 split 是否仍是 root split”。
WAL record 已经是前台判断后的 durable fact。
redo 只恢复这个 fact。
这就是为什么 redo 不调用 `_bt_insert_parent()`。
`_bt_insert_parent()` 需要 stack、查 parent、处理并发 root split。
这些是前台控制流。
redo 只需要 record 中的 block refs 和 payload。

## 8. 主流程 walkthrough：dedup 和 posting list redo
dedup 是页内物理表示变化。
它不改变 logical index contents。
多个相邻 duplicate tuple 合并成一个 posting list tuple。
但 redo 仍然要精确恢复物理布局，因为后续 binary search、posting list search、page split 和 WAL consistency check 都依赖一致的 page image。

前台 `_bt_dedup_pass()` 在 `nbtdedup.c:59`。
它扫描 leaf page，构造 `BTDedupStateData`。
每个 candidate group 用 `BTDedupInterval` 表达：
```text
baseoff
nitems
```
如果没有可 dedup 的 interval，函数直接返回，不写 WAL。
如果有，它在 critical section 里 `PageRestoreTempPage(newpage, page)`，然后写 `XLOG_BTREE_DEDUP`。
`nbtdedup.c:265` 是 `XLogInsert(RM_BTREE_ID, XLOG_BTREE_DEDUP)`。

redo 路径是 `btree_xlog_dedup()`。
`nbtxlog.c:454` 定义它。
redo 读 WAL payload 中的 intervals。
它初始化一个新的 `BTDedupStateData`，但这不是重新做策略选择。
redo 使用已记录的 interval 边界：
当当前 `offnum` 落在当前 interval 内，就调用 `_bt_dedup_save_htid()`。
否则结束 pending group。
最后 assert 生成的 intervals 和 WAL 中的 intervals 完全一致。

这个 assert 很重要。
它说明 redo 不是信任“相同 key 比较逻辑再跑一次肯定得到同样 group”。
它用 WAL 中记录的 interval 约束重构过程。
如果现有 page 和 WAL payload 不匹配，说明页状态或 WAL 解析出了问题。

posting list split 是另一个物理重写点。
当 incoming tuple 的 TID 落在已有 posting list 中间时，不能简单插在 page 上。
必须把 incoming 原始 TID 放入 posting list 内部，并把 posting list 的最大 TID 放到 newitem 上。
这样最终 newitem 会出现在 posting list 右侧，整体顺序仍正确。

核心函数是 `_bt_swap_posting()`。
它在 `nbtdedup.c:1022`。
输入：
```text
newitem    caller 的 mutable copy
oposting   page 上原 posting list tuple
postingoff posting list 内部 split offset
```
输出：
```text
nposting   replacement posting tuple
newitem    被原地修改为最终要插入的 item
```

前台简单 insert 的 posting split 在 `_bt_insertonpg()`。
`nbtinsert.c:1357` 选择 `XLOG_BTREE_INSERT_POST`。
`nbtinsert.c:1404-1405` 记录 `postingoff` 和 `origitup`。
注意它记录的是 `origitup`，不是已被 `_bt_swap_posting()` 修改后的 final newitem。
redo 必须重做 `_bt_swap_posting()`，才能得到同样的 replacement posting tuple 和 final newitem。

redo 在 `btree_xlog_insert(posting=true)` 中处理。
它从 WAL payload 读 `postingoff`。
从 page 上读取 `OffsetNumberPrev(xlrec->offnum)` 的 old posting tuple。
复制 WAL 中的 orignewitem。
调用 `_bt_swap_posting()`。
然后用 `memcpy()` 原地替换 old posting tuple，再 `PageAddItem()` 插入 final newitem。

split + posting split 更复杂。
`_bt_split()` 可能在 split record 中设置 `xlrec.postingoff`。
只有当 replacement posting tuple 或 final newitem 影响左页重建时，redo 才需要额外处理。
如果二者都在右页，右页 payload 已经完整包含最终 tuple bytes，redo 不需要知道 posting split。
这解释了 `nbtxlog.h` 对 `xl_btree_split.postingoff` 的长注释：
非零不只是“前台发生过 posting split”，而是“redo 需要为左页重构 replacement posting tuple”。

这里的可迁移规律是：
WAL record 不一定记录“发生了什么算法事件”。
它记录“redo 还需要哪些最小事实才能重建目标状态”。
这两个集合经常不同。

## 9. 主流程 walkthrough：page deletion redo
page deletion 的 redo 分两阶段。
这和 split 分两阶段是同一类工程选择：
不能把删除 parent downlink、更新 sibling chain、删除可能的 internal chain、更新 fast root、FSM reuse 都做成一个巨大原子动作。

第一阶段前台入口是 `_bt_mark_page_halfdead()`。
它在 `nbtpage.c:2122`。
调用前，leaf page 必须为空、不是 root、不是 rightmost、不是 incomplete split。
函数要找到 subtree parent。
然后在 parent page 上把目标 downlink 改成 right sibling downlink，并删除后一个 pivot tuple。
同时把 leaf page 标成 `BTP_HALF_DEAD`。
leaf high key 被改成 dummy tuple，用 `BTreeTupleSetTopParent()` 保存 top parent block。
如果 leaf 自己就是 top parent，则保存 `InvalidBlockNumber`。

它在 `nbtpage.c:2307` 写 `XLOG_BTREE_MARK_PAGE_HALFDEAD`。
WAL record 的 block 0 是 leaf。
block 1 是 subtree parent。
payload 包含：
```text
poffset
leafblk
leftblk
rightblk
topparent
```

redo 路径是 `btree_xlog_mark_page_halfdead()`。
`nbtxlog.c:705` 定义它。
redo 先修改 parent page。
它找到 `poffset` 和 `poffset+1`。
把后一项的 downlink 拷到前一项。
删除后一项。
这等价于前台让 key space 向右 sibling 转移。

然后 redo 用 `XLogInitBufferForRedo(record, 0)` 重写 leaf page。
它 `_bt_pageinit()` 后设置：
```text
btpo_prev = xlrec->leftblk
btpo_next = xlrec->rightblk
btpo_level = 0
btpo_flags = BTP_HALF_DEAD | BTP_LEAF
btpo_cycleid = 0
```
再构造 dummy high key，把 top parent 写进去。
这一阶段完成后，leaf 没有 parent downlink，但仍在 sibling chain 中。
search 到达它时会 ignore/move right。
VACUUM 下次可以继续第二阶段。

第二阶段前台入口是 `_bt_unlink_halfdead_page()`。
它在 `nbtpage.c:2349`。
它可能 unlink leaf 本身，也可能先 unlink half-dead leaf 指向的 internal top parent。
它按 left sibling、target、right sibling 的顺序加锁。
然后修改 sibling links。
接着把 target page 重写成 deleted tombstone。
`nbtpage.c:2684` 读取 `ReadNextFullTransactionId()` 作为 `safexid`。
`nbtpage.c:2685` 调 `BTPageSetDeleted(page, safexid)`。
如果删除导致 fast root 变化，还更新 metapage。
最后在 `nbtpage.c:2755` 写 `XLOG_BTREE_UNLINK_PAGE` 或 `XLOG_BTREE_UNLINK_PAGE_META`。

redo 路径是 `btree_xlog_unlink_page()`。
`nbtxlog.c:789` 定义它。
redo 的动作是：
如果有 left sibling，修它的 `btpo_next`。
用 `XLogInitBufferForRedo(record, 0)` 把 target 重写成 deleted page。
设置 target 的 `btpo_prev`、`btpo_next`、`btpo_level`，再 `BTPageSetDeleted(page, safexid)`。
修 right sibling 的 `btpo_prev`。
如果 target 是 internal page，redo 还会重建 half-dead leaf 的 dummy high key，让它指向 chain 中下一层。
如果 record 是 `_META` 变体，redo 调 `_bt_restore_meta(record, 4)`。

这一阶段完成后，target page 是 tombstone。
它没有从物理文件消失。
它仍保留 sibling links 和 `safexid`。
这是为了允许持有旧链接的 scan/search 至少能识别 deleted page 并继续移动。
README 把这叫 drain technique。

`XLOG_BTREE_REUSE_PAGE` 是 page deletion 的延迟后果。
当 `_bt_allocbuf()` 从 FSM 拿到 deleted page，并且 `BTPageIsRecyclable()` 返回 true，它可以复用该 page。
如果需要 Hot Standby conflict 信息，`nbtpage.c:958` 写 `XLOG_BTREE_REUSE_PAGE`。
这条 record 不改 buffer。
redo 在 `btree_xlog_reuse_page()` 中只处理 conflict。
`nbtxlog.c:993` 定义它。
如果 standby 上仍有 snapshot 可能看到旧结构，`ResolveRecoveryConflictWithSnapshotFullXid()` 会让冲突查询退出或等待配置策略生效。

## 10. `btree_redo()` 的执行模型
`btree_redo()` 在 `nbtxlog.c:1004`。
它做三件事。

第一，取 record info。
```text
info = XLogRecGetInfo(record) & ~XLR_INFO_MASK
```
然后 switch 到对应 `btree_xlog_*()`。
未知 op code 直接 `PANIC`。
redo 不能“尽量跳过未知 record”。
WAL stream 是 crash recovery 的事实来源。
无法理解 record 就不能安全启动。

第二，切到 `opCtx`。
`btree_xlog_startup()` 在 `nbtxlog.c:1063` 创建 `Btree recovery temporary context`。
`btree_redo()` 每条 record 后 `MemoryContextReset(opCtx)`。
这给 posting split、dedup reconstruction、vacuum posting update 这些临时 palloc 提供了明确 cleanup 边界。
redo 不是事务上下文。
它不能依赖前台 executor/query 的 MemoryContext 生命周期。

第三，各 redo routine 自己负责 buffer 获取、page LSN、dirty 标记、unlock/release。
典型 pattern：
```text
if (XLogReadBufferForRedo(record, block_id, &buf) == BLK_NEEDS_REDO)
{
    page = BufferGetPage(buf);
    mutate page;
    PageSetLSN(page, record->EndRecPtr);
    MarkBufferDirty(buf);
}
if (BufferIsValid(buf))
    UnlockReleaseBuffer(buf);
```

`XLogReadBufferForRedo()` 的返回值是幂等性的核心。
如果 page LSN 已经不小于 record LSN，redo 不重复物理修改。
但仍可能需要释放 buffer。
`XLogInitBufferForRedo()` 用于 record 语义上要重建整页的场景。
例如新右页、new root、half-dead leaf、deleted target page、metapage。

`btree_redo()` 不知道事务提交或 abort。
B-tree index tuple 的物理插入不会因为事务 abort 在 redo 中“撤销”。
可见性由 heap/MVCC 在查询时判断。
B-tree redo 的工作是恢复 index access method 的物理导航结构。
这也是为什么 leaf insert、dedup、page deletion redo 只关心 page bytes 和结构 flags。

## 11. 生命周期 / ownership / cleanup
前台 split 的 ownership：
`_bt_insertonpg()` 进入时持有目标 page 的 pin 和 write lock。
如果需要 split，`_bt_split()` 分配 `rbuf`。
split 完成后，left buffer 和 right buffer 都仍然 write-locked。
`_bt_insert_parent()` 在 parent 定位后释放 right child。
left child 作为 `cbuf` 传入 parent insertion，直到 parent downlink insert 和 flag clear 同一个 critical section 后释放。

前台 WAL ownership：
在 critical section 中先修改 page，再注册 WAL，再 `XLogInsert()`，再设置 page LSN。
如果 `RelationNeedsWAL(rel)` 为 false，用 fake LSN。
这通常对应 unlogged/temp/local buffer 等不需要常规 WAL 的路径。
课程里不要把 “no WAL” 误解为 “无需 page LSN”。
page LSN 仍然是 buffer/page 管理和一致性路径需要的状态。

redo buffer ownership：
每个 `btree_xlog_*()` 负责释放自己获取的 buffer。
同层多页修改通常延迟释放，以降低读者观察到中间 sibling state 的机会。
跨层 lock coupling 在 redo 中不完整重现，因为 recovery 中没有 concurrent writer。
但 block-level redo 仍按需要持有 buffer lock，防止 standby reader 看到正在修改的 page。

redo memory ownership：
`opCtx` 是每条 B-tree redo record 的临时内存上下文。
`btree_redo()` 切入它。
record 完成后 reset。
`btree_xlog_cleanup()` 在 recovery 结束时删除它。
这覆盖了 `CopyIndexTuple()`、`palloc()` 的 `BTDedupStateData`、`BTVacuumPostingData` 等临时对象。

VACUUM page deletion ownership：
`_bt_pagedel()` 注释明确说会 leak memory。
它依赖 VACUUM 的 temp context 周期性 reset。
page deletion 的持久结果不是由内存对象持有。
持久结果在 page flags、sibling links、dummy high key、deleted page `safexid` 和 WAL record 中。

pending FSM ownership：
`_bt_pendingfsm_init()` 在 `nbtpage.c:2991` 初始化 `BTVacState.pendingpages`。
它受 `work_mem` 上限约束。
`_bt_pendingfsm_add()` 只保存当前 VACUUM 新删 page 的 block 和 `safexid`。
`_bt_pendingfsm_finalize()` 用这些 local state 判断是否能 `RecordFreeIndexPage()`。
如果数组满了，只丢弃优化信息，不影响 correctness。
后续 VACUUM 仍可再次发现 old deleted page。

ERROR/abort 兜底：
前台 critical section 内不能 `ereport(ERROR)`。
源码中多处注释写着 “No ereport(ERROR) until changes are logged”。
一旦进 critical section，失败通常升级为 PANIC，依赖 crash recovery 重放 WAL。
如果在进入 critical section 前失败，页面还没有被原地改成持久不一致状态。
如果 split record 已写、parent insertion 未写，`BTP_INCOMPLETE_SPLIT` 是 durable recovery state。
后续 `_bt_finish_split()` 会修。

## 12. 正确性机制层次
第一层是 WAL-before-data。
前台修改持久页前写 WAL。
redo 用 page LSN 判断是否需要重放。
这个层次保证 crash 后页修改不会丢失或重复应用。

第二层是 record 原子性。
单条 B-tree WAL record 表示一个可恢复的结构动作。
split record 不承诺 parent downlink 存在。
insert upper/newroot record 才承诺 downlink 存在并清 incomplete flag。
mark halfdead record 不承诺 target 已 unlink。
unlink record 才承诺 target tombstone 建立。

第三层是 Lehman-Yao 搜索不变量。
high key 和 rightlink 让读者在 parent downlink stale 或 missing 时仍能向右恢复。
这使得 split 的第一阶段可以单独 crash-safe。
page deletion 也保留 tombstone 和 sibling links，让读者能恢复。

第四层是 page lock 和 buffer pin。
前台 writer 用 buffer write lock 保护 page bytes。
某些路径需要跨页 lock order 防止 deadlock。
redo 中没有 concurrent writer，但 standby reader 可能存在。
所以 redo 仍要通过 buffer locks 保护 page mutation，并保守处理同层 sibling chain。

第五层是 durable flags。
`BTP_INCOMPLETE_SPLIT` 让后续写路径知道必须先完成 parent insertion。
`BTP_HALF_DEAD` 让 search 忽略该 leaf，并让 VACUUM 知道 deletion 第一阶段已完成。
`BTP_DELETED` 和 `safexid` 让 reuse 受 visibility horizon 约束。

第六层是 visibility conflict。
B-tree page reuse 不是纯物理问题。
primary 上 deleted page 可复用时，standby 上可能还有旧 snapshot。
`XLOG_BTREE_REUSE_PAGE` 把 conflict horizon 传给 standby。
这保证 standby query 不会用旧快照观察到被复用后的 page，误以为旧结构仍然有效。

第七层是 consistency check mask。
`btree_mask()` 在 `nbtxlog.c:1081` 屏蔽 LSN、checksum、hint bits、leaf line pointer flags、`BTP_HAS_GARBAGE`、`BTP_SPLIT_END` 和 `btpo_cycleid`。
这些状态可能不被 WAL 完整记录，或者 recovery 有意不完全重现。
WAL consistency checking 要比较的是 contract 内的持久语义，不是所有 bit。

## 13. 错误路径 / 异常路径 / fallback
异常路径一：split 写完，parent insertion 失败。
这可能来自 crash、OOM、磁盘空间不足或 recoverable ERROR。
恢复后 left page 带 `BTP_INCOMPLETE_SPLIT`。
search 仍可靠 rightlink 找到 right page。
insert 路径遇到该 flag 时调用 `_bt_finish_split()`。
`_bt_finish_split()` 在 `nbtinsert.c:2272`。
它锁 right sibling，判断是否 root split，再调用 `_bt_insert_parent()`。
VACUUM 在某些内部页删除路径也会完成 interrupted split，因为它需要可靠 re-find parent item。

异常路径二：redo 时 block 不需要重放。
`XLogReadBufferForRedo()` 可能返回已经应用。
redo routine 必须不重复插入 tuple 或重复删除 item。
所以 B-tree redo 不先“看逻辑状态是否像目标”，而依赖 page LSN。
这也是为什么每个被修改 page 都必须 `PageSetLSN(page, record->EndRecPtr)`。

异常路径三：right sibling link 不匹配。
前台 page deletion 里有多处 sibling link 验证。
如果找不到 valid left sibling，`_bt_unlink_halfdead_page()` 会 LOG index corruption，并放弃本次 deletion，而不是继续把结构改坏。
有些场景用 ERROR，因为状态不应变化。
这些差异说明 page deletion fallback 的首要目标是保持现有结构可搜索，而不是强行回收空间。

异常路径四：posting list split offset 异常。
`_bt_swap_posting()` 检查 `postingoff > 0 && postingoff < nhtids`。
如果索引损坏导致新 tuple 与 plain tuple TID 完全重叠，或 posting list 结构不成立，会 ERROR。
前台 `_bt_insertonpg()` 也检查 old item 必须是 posting tuple，且不能是 LP_DEAD。
redo 遇到不一致通常会 PANIC 或 ERROR，表示 WAL 和 page state 无法组成合法恢复。

异常路径五：dedup 没有 interval。
前台 `_bt_dedup_pass()` 如果没有可合并 group，直接释放内存返回，不写 WAL。
redo 没有“空 dedup record”要处理。
这避免了把纯策略尝试写进 WAL。
WAL 只记录实际 page mutation。

异常路径六：pending FSM 数组达到 `work_mem` 上限。
`_bt_pendingfsm_add()` 到上限就 return。
这只是丢掉“当前 VACUUM 结束时顺手放 FSM”的优化机会。
deleted page 仍带 tombstone。
后续 VACUUM 还能发现并回收。
correctness 不依赖这块 local memory。

异常路径七：reusing page 时 FSM 不可信。
`_bt_allocbuf()` 从 FSM 取 page 后，不直接使用。
它必须 conditional lock page。
如果 page 是 new，初始化使用。
如果是 recyclable deleted page，才复用。
如果不可 lock 或不可 recycle，就丢掉这次 FSM 候选。
最坏只是空间复用推迟到下次 VACUUM。

异常路径八：Hot Standby conflict。
`btree_xlog_delete()` 在 `InHotStandby` 下调用 `ResolveRecoveryConflictWithSnapshot()`。
`btree_xlog_reuse_page()` 调 `ResolveRecoveryConflictWithSnapshotFullXid()`。
这些不是 B-tree 结构算法本身。
它们是 WAL replay 把 primary cleanup/reuse 边界传播给 standby 查询的异常处理。

## 14. 成本、资源与跨模块传播
WAL 体积成本：
split record 有意完整记录右页 tuple bytes。
这样通常比让 XLogInsert 把新右页当成整页 FPI 更省。
左页则走标准增量/FPI 机制。
如果 checkpoint 后第一次改页，FPI 仍可能出现。
`full_page_writes`、checkpoint 频率、page split 频率都会影响 WAL 放大。

CPU 成本：
redo split 要 `_bt_restore_page()` 扫 tuple bytes，重建 line pointer order。
dedup redo 要重新构造 posting list。
posting split redo 要 `_bt_swap_posting()` 拷贝 tuple 并移动 TID array。
这些成本随页上 item 数、posting list TID 数、dedup interval 数增长。
它们不是按表行数全局增长，而是按单页 payload 和 WAL record 数量扩张。

contention 成本：
前台 split 需要持有目标页、新右页、旧右 sibling，随后还要 parent page。
parent 也 split 时递归放大。
redo 中没有 concurrent writer，但 standby reader 仍可能和 redo 争 buffer content lock。
高 split WAL replay 会表现为 standby apply latency、read query conflict 或 buffer IO 等待。

IO 成本：
redo 需要读取 block refs。
如果数据页已经在磁盘上且 LSN 足够新，可以跳过物理修改。
如果需要重做但 page 不在 buffer，会产生 read IO。
`XLogInitBufferForRedo()` 对新页/重写页可避免读取旧内容。
split 右页、newroot、halfdead leaf、deleted target 常用这种模式。

metapage 传播：
root split、fast root update、cleanup info update 都会触碰 metapage。
metapage 是 B-tree 的导航入口。
但 ordinary search 有 relcache `rd_amcache`。
stale fastroot 通常只导致多走几层或依赖 rightlink/high key 修正，不等于错误。
redo 重建 metapage 后，正常运行阶段会通过 relcache invalidation/stat update 等机制逐步清掉旧缓存。

VACUUM 传播：
page deletion 留下 tombstone。
`safexid` 推迟 FSM reuse。
old snapshot、long transaction、standby feedback、replication slot 都可能延迟全局 horizon。
这会让 deleted page 继续占磁盘。
`btm_last_cleanup_num_delpages` 又会影响之后 `btvacuumcleanup()` 是否需要 cleanup scan。

standby 传播：
`XLOG_BTREE_DELETE` 和 `XLOG_BTREE_REUSE_PAGE` 可能导致 recovery conflict。
这不是“索引坏了”。
这是 primary 上物理删除/复用与 standby 上旧 snapshot 的冲突。
在只看 primary 性能时容易忽略。
在 streaming replication 中，它会表现为 standby query 被取消、replay lag、或 recovery conflict 统计增加。

dedup 和 split 的资源边界：
dedup 减少 leaf tuple 数和 page split 频率。
但它让 posting list update、posting split redo 和 deletion subset update 变复杂。
posting list 大小受 `maxpostingsize` 控制。
过度压缩可能让 subset delete 成本变高，也让 split point 选择更难。
当前实现选择 lazy dedup 和受限 posting size，是为了控制 hot path 和 redo 复杂度。

## 15. 观测与诊断入口
能直接观测的状态：
`pageinspect` 的 `bt_page_stats()` 可以看 page type、live/dead items、free size、btpo flags 的部分表达。
`bt_page_items()` 可以看 leaf/internal item。
`bt_metap()` 可以看 root、level、fastroot、fastlevel、allequalimage 等 metapage 状态。
这些是单页/单索引粒度。

能直接观测的 WAL：
`pg_waldump --rmgr=Btree` 可以看到 B-tree record 类型。
结合 `--start`、`--end` 可以定位某段 workload 的 split、insert、dedup、delete、newroot、unlink record。
这能证明“发生了某类结构动作”。
它不能单独证明某个 SQL 是唯一原因。

能从统计看趋势的入口：
`pg_stat_wal` 看 WAL bytes、records、FPI 趋势。
`pg_stat_io` 看 relation read/write 形态。
`pg_stat_user_indexes` 看 idx_scan、idx_tup_read、idx_tup_fetch。
`pg_stat_all_tables` 和 autovacuum 日志看 VACUUM 是否及时推进。
这些是 database/instance 累计统计。
它们不能直接告诉你哪一个 page 发生 split。

能从日志看 cleanup 的入口：
`VACUUM VERBOSE` 会报告 index pages deleted、reusable 等信息。
autovacuum verbose log 可以看到 cleanup 频率和 dead tuple 处理。
但它不会告诉你每条 `XLOG_BTREE_UNLINK_PAGE` 的 block id。
需要配合 `pg_waldump` 或断点。

能通过断点看的状态：
在开发环境中可以对这些函数打断点：
```text
btree_redo
btree_xlog_split
btree_xlog_insert
_bt_clear_incomplete_split
btree_xlog_mark_page_halfdead
btree_xlog_unlink_page
btree_xlog_reuse_page
_bt_swap_posting
```
在前台路径可以断：
```text
_bt_split
_bt_insert_parent
_bt_newlevel
_bt_mark_page_halfdead
_bt_unlink_halfdead_page
_bt_allocbuf
_bt_dedup_pass
```

只能推断的状态：
某个 search 是否曾经通过 rightlink 绕过 missing downlink，通常只能从断点、trace、injection point 或构造实验推断。
`BTP_INCOMPLETE_SPLIT` 通常非常短命。
生产系统里用 SQL 恰好看到它的概率很低。
如果使用 injection point 留住 incomplete split，可以观察得更稳定。

几乎不可见的状态：
redo 中 `opCtx` 的 per-record reset、XLogReadBufferForRedo 的 skip 决策、FPI 是否让 block data payload 被省略，通常需要 gdb、trace log 或读取 WAL 解析细节。
普通 `pg_stat_*` 不会暴露这些细节。

诊断时的因果边界：
看到 B-tree WAL bytes 增加，不等于一定是 split。
dedup、delete、vacuum、newroot、metapage cleanup 都会产生 B-tree WAL。
看到 standby conflict，不等于索引损坏。
可能是 delete/reuse record 与 standby snapshot 冲突。
看到 index bloat，不等于 redo 问题。
更常见原因是 workload version churn、old snapshot、VACUUM 延迟、dedup 不适用或 fillfactor/index key shape。

## 16. 常见误区
误区一：
把 redo 理解成重新执行 `_bt_doinsert()`。
redo 没有 executor state、BTStack、snapshot、uniqueness wait，也不重新搜索 parent。
它只按 WAL record 修改 block refs。

误区二：
认为 split record 后 parent 一定有 downlink。
`XLOG_BTREE_SPLIT_L/R` 只完成本层 split。
parent downlink 来自后续 `XLOG_BTREE_INSERT_UPPER`、`INSERT_META` 或 `NEWROOT`。
两者之间 crash 后，missing downlink 是合法状态。

误区三：
看到 `BTP_INCOMPLETE_SPLIT` 就认为索引损坏。
它是 durable repair marker。
如果长期存在，说明后续写路径还没经过该区域，或测试使用 injection point 留住它。
只有 parent repair 失败、结构无法 re-find、或 sibling link 不一致时，才进入损坏诊断。

误区四：
认为 redo 里的 lock 可以完全忽略。
recovery 没有 concurrent writer，但 Hot Standby 有 reader。
redo 不需要重现所有跨层 lock coupling。
但同层 sibling chain 修改仍要保守，避免读者看到局部不一致。

误区五：
把 `XLOG_BTREE_REUSE_PAGE` 当成 page rewrite record。
它不注册 buffer，也不修改 page。
它是 Hot Standby snapshot conflict point。
真正 page reuse 的物理初始化发生在后续 split/alloc 使用该 buffer 时。

误区六：
认为 `BTP_HAS_GARBAGE`、LP_DEAD、`BTP_SPLIT_END` 都必须通过 WAL 精确恢复。
其中一些是 hint 或 optimization。
`btree_mask()` 会屏蔽不属于 WAL consistency contract 的 bit。
不要用这些 bit 的恢复差异判断 corruption。

误区七：
认为 dedup redo 可以重新比较 key 自己决定 intervals。
WAL 已经记录 interval。
redo 重构必须受 interval 约束，并校验生成结果和 WAL 一致。
策略选择属于前台，不属于 redo。

误区八：
认为 deleted page 放进 FSM 就等于 page deletion record 的一部分。
page deletion 建立 tombstone。
FSM reuse 是延迟优化。
是否能放 FSM 取决于 `safexid` 和 global visibility horizon。
这可以跨 VACUUM 周期发生。

## 17. 课堂实验一：观察 split、parent insert 与 newroot WAL
目标：
用小表制造 B-tree split，观察 `pg_waldump` 里的 Btree record 顺序。

步骤：
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
CREATE TABLE bt_wal_split_demo(id int primary key, pad text);
INSERT INTO bt_wal_split_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 20000) AS g;
CHECKPOINT;
SELECT pg_current_wal_lsn() AS lsn_before;
INSERT INTO bt_wal_split_demo
SELECT g, repeat('y', 200)
FROM generate_series(20001, 26000) AS g;
SELECT pg_current_wal_lsn() AS lsn_after;
```

然后用 `pg_waldump`：
```bash
pg_waldump --rmgr=Btree --start=<lsn_before> --end=<lsn_after> $PGDATA/pg_wal
```

观察点：
- 是否出现 `SPLIT_L` 或 `SPLIT_R`。
- 是否紧跟 `INSERT_UPPER` 或 `NEWROOT`。
- 是否看到 `INSERT_META` 或 `NEWROOT` 这种 metapage 相关 record。
- WAL 顺序是否符合“本层 split 先，parent/newroot 后”。

回到源码解释：
`_bt_split()` 写 `XLOG_BTREE_SPLIT_L/R`。
`_bt_insertonpg(parent)` 写 `XLOG_BTREE_INSERT_UPPER` 或 `INSERT_META`。
`_bt_newlevel()` 写 `XLOG_BTREE_NEWROOT`。
redo 中分别进入 `btree_xlog_split()`、`btree_xlog_insert()`、`btree_xlog_newroot()`。

## 18. 课堂实验二：源码断点跟踪 redo contract
目标：
在开发环境里验证 redo 不走前台算法。

准备：
启动一个 debug build 的 PostgreSQL。
制造包含 Btree WAL 的 workload。
做一次 immediate shutdown 或在测试环境中构造 crash。
重启时 attach startup process。

建议断点：
```text
btree_redo
btree_xlog_split
btree_xlog_insert
btree_xlog_newroot
btree_xlog_mark_page_halfdead
btree_xlog_unlink_page
_bt_clear_incomplete_split
_bt_restore_meta
_bt_restore_page
_bt_swap_posting
```

观察点：
- `btree_redo()` switch 的 `info` 值对应哪个 `XLOG_BTREE_*`。
- `XLogReadBufferForRedo()` 返回 `BLK_NEEDS_REDO` 还是 skip。
- split redo 是否调用 `_bt_insert_parent()`。
- insert upper/newroot redo 是否只清 child incomplete flag，而不是重做 parent search。
- dedup redo 是否使用 WAL 中的 interval。
- reuse page redo 是否只处理 standby conflict。

回到源码解释：
redo 的输入是 WAL record 和 block refs。
前台的 `BTStack`、`BTInsertState`、VACUUM local state 都不存在。
这就是为什么 `xl_btree_*` record 必须携带足够重建目标 page state 的信息。

## 19. 讨论题
1. 为什么 `XLOG_BTREE_SPLIT_L/R` 不直接插入 parent downlink？
2. 为什么 `BTP_INCOMPLETE_SPLIT` 标在左页，而不是缺 downlink 的右页？
3. redo 中不重现前台跨层 lock coupling，为什么仍然正确？它依赖哪些前提？
4. `XLOG_BTREE_REUSE_PAGE` 不修改任何 page，为什么仍然必须 WAL 记录？
5. dedup redo 为什么记录 interval，而不是重新跑一遍“哪些 tuple key 相等”的策略判断？
6. page deletion 为什么要先 half-dead，再 unlink，再等待 safexid 才能 FSM reuse？
7. `btree_mask()` 屏蔽 `BTP_HAS_GARBAGE` 和 `BTP_SPLIT_END` 说明了 WAL consistency check 的什么边界？
8. 如果 `pg_waldump` 看到大量 Btree split record，你还需要哪些信息才能判断根因是 workload、VACUUM 延迟、fillfactor、checkpoint/FPI，还是实现问题？

## 20. 本节小结
本节唯一主问题是：
B-tree 跨页、跨层结构变化如何拆成可 redo 的 WAL 原子动作，同时保持搜索正确和后续可修复。

核心链路是：
```text
前台结构变化生成 XLOG_BTREE_* record
  -> crash 后 btree_redo() dispatch
  -> redo 按 block refs 和 payload 恢复页状态
  -> high key/rightlink/flags/tombstone 保证中间状态合法
  -> 后续 insert 或 VACUUM 继续修复未完成结构债务
```

核心状态是：
`BTPageOpaqueData` 里的 sibling links、level 和 flags。
`BTMetaPageData` 里的 root/fastroot。
`xl_btree_*` record 里的最小 redo facts。
`BTDeletedPageData.safexid` 里的 recycle horizon。
backend-local 的 `BTStack`、`BTDedupStateData`、`BTVacState.pendingpages` 只服务前台或 VACUUM 当前过程，不能作为 crash recovery 事实来源。

ownership 和 cleanup 的关键是：
前台 critical section 内不允许普通 ERROR 破坏 WAL-before-data。
redo 每条 record 使用 `opCtx` 临时上下文，结束后 reset。
buffer 由各 redo routine 获取、修改、设置 LSN、dirty、释放。
VACUUM pending FSM state 只是优化，丢失不影响 correctness。

正确性不是单个机制保证的。
WAL 负责 crash recovery。
page LSN 负责幂等。
buffer lock 负责 page bytes 并发安全。
high key/rightlink 负责 missing downlink 时的可达性。
durable flags 负责表达未完成结构债务。
visibility horizon 和 recovery conflict 负责 deleted page reuse 的跨主备边界。

错误路径的核心规律是：
PostgreSQL 不强迫 recovery 结束时把所有 B-tree 结构债务立即修完。
incomplete split 可以留给后续 insert。
half-dead page 可以留给后续 VACUUM。
deleted page 可以留给后续 visibility horizon 和 FSM reuse。
只要每一步的持久状态对读者可搜索、对写者可识别、对 redo 可幂等，它就是合法状态。

观测上：
`pg_waldump --rmgr=Btree` 能看到 record 类型和顺序。
`pageinspect` 能看 metapage 和单页状态。
`pg_stat_wal`、`pg_stat_io`、VACUUM 日志能看趋势。
但 missing downlink 修复、redo skip、posting split 重构、Hot Standby conflict 的细节通常需要断点或定向实验。

本节可迁移的系统规律：
复杂存储结构的 crash safety 往往不是“把高级操作做成一个大原子事务”。
更常见的工程答案是：
把操作拆成多个小的持久状态转移。
每个状态转移都有明确 redo contract。
每个中间状态都满足读路径安全。
未完成工作用持久状态显式标记。
后续写路径或后台维护路径按标记继续推进。
这个模型会在 GiST、GIN、FSM/VM、heap pruning、logical decoding 的许多边界里反复出现。
