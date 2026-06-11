# PostgreSQL B-tree page split、parent insertion 与 incomplete split

## 课程定位

上一组课程已经讲到 heap page、FSM、VM 和 WAL-before-data。
这一节进入 nbtree index page 的结构维护。
前置知识：
- 已理解 buffer pin 与 content lock 的边界。
- 已理解 page LSN、WAL record、redo 的基本关系。
- 已理解 B-tree 搜索依赖 internal page downlink，leaf page 保存 index tuple。
- 已理解 PostgreSQL index tuple 指向 heap TID，index insert 不随事务 abort 物理回滚。
本节唯一主问题：
一个 leaf page 已满时，PostgreSQL 如何在不阻塞并发搜索、不要求跨层大事务原子更新的前提下，完成 page split、parent downlink insertion，并在中断后用 incomplete split 让后来的 backend 帮忙修复结构？
本节围绕的核心矛盾：
搜索路径希望随时能从 root 走到正确 leaf。
插入路径在 leaf full 时又必须把一个物理 page 拆成左右两个 page。
如果要求 split leaf、更新 sibling links、插入 parent downlink、必要时继续分裂 parent、更新 root 全部成为一个巨大原子动作，锁范围、WAL record 和故障恢复都会膨胀。
如果只完成 leaf split 而 parent 没有 downlink，新右页短时间内只能通过左页 rightlink 找到。
这对搜索仍然正确，但对后续结构更新有风险。
PostgreSQL 的选择是：

```text
把 split 拆成多个 WAL 原子动作。
每个动作都让搜索正确。
用 BTP_INCOMPLETE_SPLIT 标记“右 sibling 的 parent downlink 尚未补齐”。
后续写路径遇到该标志时，先帮助完成 parent insertion，再继续自己的工作。
```

学完后你应该能独立判断：
- 为什么 leaf full 不是立刻 split。
- 为什么 `_bt_findsplitloc()` 必须把 incoming item 计入 split point 选择。
- `_bt_split()` 的 WAL 只完成本层 split，不完成 parent insertion。
- 为什么 `BTP_INCOMPLETE_SPLIT` 标在左页，而缺 downlink 的是右页。
- 为什么 search 能容忍 missing downlink，而 insertion 不能继续忽略它。
- parent insertion 为什么必须和清 child incomplete flag 是同一个 WAL 原子动作。
- 为什么 `_bt_moveright()`、`_bt_stepright()`、`_bt_getstackbuf()` 都可能帮助完成 split。
- crash、ERROR、OOM、out-of-disk 后结构如何自愈。
- `pageinspect`、`pg_waldump`、`pg_stat_wal` 各能看到什么，哪些只能推断。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `nl -ba <source-file>`；重点源码和辅助核对入口统一放在第 3 节的阅读顺序里。

## 1. 本节在总主线中的位置

B-tree index 是 relation 的另一种 main fork 内容。
它不保存 heap tuple version 的可见性事实。
它保存的是 key 到 heap TID 的导航结构。
一个 heap insert 或 update 需要为相关 index 插入 index tuple。
执行器最终进入 `btinsert()`，再进入 `nbtinsert.c` 的 `_bt_doinsert()`。
普通情况只是把一个 index tuple 加到 leaf page。
本节关心的是 slow path：

```text
leaf page 找到了
  -> free space 不足
  -> 尝试删除或 dedup 仍不足
  -> split leaf page
  -> 写本层 split WAL
  -> 给新右页插入 parent downlink
  -> 如果 parent 也满，递归分裂 parent
  -> 如果 root 分裂，创建新 root 并更新 metapage
```

这个 slow path 是理解 nbtree 并发搜索、WAL record 粒度、redo、VACUUM page deletion、index bloat 的共同入口。
它不是教材里“一次插入一个 key，然后父节点同步更新”的单线程 B-tree。
PostgreSQL 的 nbtree 是 Lehman and Yao high-concurrency B-tree 的工程化实现。
每个 page 有 rightlink 和 high key。
搜索发现自己落在过旧页面时，可以沿 rightlink 向右追。
这使得“parent downlink 暂时缺失”不等于搜索错误。
但它仍然是结构债务。
`BTP_INCOMPLETE_SPLIT` 就是这笔债务的 durable 标记。
本节不会展开全部 nbtree 特性。
不会详细讲唯一性检查、posting list、dedup 的完整算法。
这些只在解释 split 是否发生、WAL 如何记录、异常路径如何 fallback 时出现。

## 2. 核心矛盾与一句话运行模型

先给一句话模型：

```text
nbtree split 先用 rightlink/high key 保证搜索可达，再用 parent insertion 补齐导航结构；BTP_INCOMPLETE_SPLIT 把“搜索正确但结构未修完”的状态持久化，让后续写路径必须先修复再继续。
```

把它拆成四个不变量：
第一，page split 后左页的 key range 变窄。
左页得到新的 high key。
右页保存被移走的一部分 item。
左页 rightlink 指向右页。
如果原来有右 sibling，那个 sibling 的 leftlink 也要改到新右页。
第二，split WAL 是本层原子动作。
它可以让 redo 重建左页、右页和同层 sibling link。
它不保证 parent 已经有指向新右页的 downlink。
第三，parent insertion 是后续原子动作。
它把左页 high key 复制成 parent pivot tuple，并把 downlink 设置为新右页 block。
同一个动作清除左页的 `BTP_INCOMPLETE_SPLIT`。
第四，如果第二步成功而第三步失败，tree 对 read search 仍然可用。
后续 insert 或部分 VACUUM 路径遇到 incomplete flag，会先调用 `_bt_finish_split()` 补 parent downlink。
这里的矛盾是：

```text
局部 WAL 原子性
  vs
跨层结构完整性
  vs
并发搜索不停顿
```

PostgreSQL 没有让一个 WAL record 覆盖任意层数的 cascade split。
也没有让读者负责修结构。
它把修结构责任放在写路径。

## 3. 核心文件分工与阅读顺序

推荐按状态推进读，不按文件名读。

| 顺序 | 文件 | 读什么 | 为什么 |
| --- | --- | --- | --- |
| 1 | `src/backend/access/nbtree/README` | Lehman-Yao、WAL considerations、incomplete split | 先建立为什么 split 能拆成多步 |
| 2 | `src/include/access/nbtree.h` | `BTPageOpaqueData`、`BTP_INCOMPLETE_SPLIT`、`P_*` 宏 | 明确状态放在哪里 |
| 3 | `src/backend/access/nbtree/nbtinsert.c` | `_bt_doinsert()`、`_bt_findinsertloc()`、`_bt_insertonpg()`、`_bt_split()`、`_bt_insert_parent()`、`_bt_finish_split()` | 主流程 |
| 4 | `src/backend/access/nbtree/nbtsplitloc.c` | `_bt_findsplitloc()` | split point 如何选择 |
| 5 | `src/include/access/nbtxlog.h` | `XLOG_BTREE_SPLIT_L/R`、`XLOG_BTREE_INSERT_UPPER`、`XLOG_BTREE_NEWROOT`、`xl_btree_split` | WAL record 粒度 |
| 6 | `src/backend/access/nbtree/nbtxlog.c` | `btree_xlog_split()`、`btree_xlog_insert()`、`btree_xlog_newroot()` | redo 如何重建和清 flag |
| 7 | `src/backend/access/nbtree/nbtsearch.c` | `_bt_moveright()` | search/write descent 如何遇到并修 incomplete split |
| 8 | `src/backend/access/nbtree/nbtpage.c` | page deletion 对 incomplete split 的检查 | VACUUM 与结构债务的边界 |
| 9 | `src/backend/access/rmgrdesc/nbtdesc.c` | WAL record 文本描述 | `pg_waldump` 实验入口 |
| 10 | `contrib/pageinspect/btreefuncs.c` | `bt_metap()`、`bt_page_stats()` | SQL 观察 page opaque 状态 |

最短跟读链路：

```text
_bt_doinsert()
  -> _bt_search_insert()
  -> _bt_findinsertloc()
  -> _bt_insertonpg()
  -> _bt_split()
  -> _bt_insert_parent()
     -> _bt_getstackbuf()
     -> _bt_insertonpg(parent)
        -> clear BTP_INCOMPLETE_SPLIT
```

中断后的修复链路：

```text
_bt_moveright()
  -> P_INCOMPLETE_SPLIT
  -> _bt_finish_split()
  -> _bt_insert_parent()
_bt_stepright()
  -> P_INCOMPLETE_SPLIT
  -> _bt_finish_split()
_bt_getstackbuf()
  -> P_INCOMPLETE_SPLIT on parent level page
  -> _bt_finish_split()
```

redo 链路：

```text
XLOG_BTREE_SPLIT_L/R
  -> btree_xlog_split()
  -> left page has BTP_INCOMPLETE_SPLIT
XLOG_BTREE_INSERT_UPPER / INSERT_META
  -> btree_xlog_insert()
  -> _bt_clear_incomplete_split(child)
XLOG_BTREE_NEWROOT
  -> btree_xlog_newroot()
  -> _bt_clear_incomplete_split(left child)
```

## 4. 关键状态与边界

先看 page special area。
`nbtree.h:63-70` 定义 `BTPageOpaqueData`。
本节需要的字段是：

```text
btpo_prev    左 sibling block，leftmost 时为 P_NONE
btpo_next    右 sibling block，rightmost 时为 P_NONE
btpo_level   tree level，leaf 为 0
btpo_flags   page 类型和状态位
btpo_cycleid VACUUM cycle ID，用于 VACUUM 识别 split group
```

不要把单个字段当成语义。
`btpo_next` 只是右链接。
`P_HIKEY` 位置的 high key 决定本页 key range 上界。
parent page 里的 pivot tuple/downlink 决定从上层如何导航到某个 child。
`BTP_INCOMPLETE_SPLIT` 决定 split 后 parent downlink 是否已经补齐。
这些组合起来才是“结构状态”。
`nbtree.h:76-85` 定义关键 flags：

```text
BTP_LEAF
BTP_ROOT
BTP_DELETED
BTP_HALF_DEAD
BTP_SPLIT_END
BTP_HAS_GARBAGE
BTP_INCOMPLETE_SPLIT
```

本节最重要的是：

```text
BTP_INCOMPLETE_SPLIT = 1 << 7
含义：right sibling's downlink is missing
```

注意这句话的方向。
flag 在左页。
缺的是右页在 parent 中的 downlink。
为什么标左页？
因为 search 或 insert 从左页沿 rightlink 走向右页时，恰好能在左页知道：

```text
我即将走到的右 sibling 还没有 parent downlink。
```

如果标在右页，帮助者需要先到达右页才知道要修。
但修 parent downlink 时需要左页 high key 构造 pivot tuple。
标左页更直接。
README `666-681` 明确说明了这一点。
`P_INCOMPLETE_SPLIT(opaque)` 是宏封装。
它只看 `btpo_flags`。
它不检查 parent。
所以它是一个 durable state marker，而不是现场验证工具。
正常完成 parent insertion 后，flag 会被清掉。
如果 flag 还在，系统按“parent downlink 缺失”处理。

## 5. 从 leaf insert 进入 split

入口在 `_bt_doinsert()`。
`nbtinsert.c:104-170` 做三件事：

```text
构造 insertion scan key
初始化 BTInsertState
调用 _bt_search_insert() 找到 leaf page，并以 BT_WRITE 锁住
```

`_bt_search_insert()` 还有 rightmost leaf fastpath。
`nbtinsert.c:308-312` 说明它不会在预计要 split 时使用 fastpath。
原因很实际：
split 后要插 parent downlink，需要 descent stack。
fastpath 只有 leaf buffer，没有完整 parent stack。
所以当 rightmost page 空间不足时，代码会回退到从 root descent。
唯一性检查会让路径更复杂。
`_bt_doinsert()` 在 `nbtinsert.c:208-236` 中可能发现需要等待另一个事务。
它会释放 leaf buffer，等待后 `goto search` 重新下降。
这不是本节主线，但要记住：

```text
写路径不能把一次 descent 当成永远有效。
等待、并发 split、page deletion 都可能要求重新定位。
```

真正决定落点的是 `_bt_findinsertloc()`。
它要求 leaf page 不能处于 incomplete split。
`nbtinsert.c:848` 有断言：

```text
P_ISLEAF(opaque) && !P_INCOMPLETE_SPLIT(opaque)
```

这说明 insertion 对 incomplete split 的态度很严格。
读 search 可以靠 rightlink 忍受 parent downlink 缺失。
写 insert 不能在结构债务未清时继续制造新的结构变化。
当目标 leaf 空间不足时，并不是马上 split。
`nbtinsert.c:913-920` 对 heapkeyspace index 会先调用 `_bt_delete_or_dedup_one_page()`。
它可能做三类工作：

```text
simple deletion: 删除 LP_DEAD index tuples
bottom-up deletion: 对版本 churn 造成的重复 index tuples 做底层清理
deduplication: 把相等 key 合并为 posting list
```

`nbtinsert.c:2716-2727` 的注释很关键。
现代 heapkeyspace index 不再只依赖 `BTP_HAS_GARBAGE` 才扫描。
因为如果不扫描，就可能错过可删除 tuple，导致本可避免的 split。
所以 leaf full 的真实路径是：

```text
PageGetFreeSpace(page) < itemsz
  -> 尝试删除或 dedup
  -> 若腾出空间，仍是普通 insert
  -> 若仍不足，进入 _bt_insertonpg() 的 split 分支
```

这个 fallback 是性能和空间放大的边界。
很多“index split 太多”的诊断，不能只看 key 分布。
还要看 UPDATE churn、LP_DEAD 产生速度、dedup 是否可用、autovacuum 是否及时。

## 6. `_bt_insertonpg()` 的分叉点

`_bt_insertonpg()` 是 retail insert 和 split 的共同入口。
`nbtinsert.c:1089-1105` 的函数注释列出递归任务：

```text
必要时拆 posting list
必要时 split target page
插入新 tuple
如果 split，找到 parent 并插入新 child pointer
必要时递归到 parent
必要时更新 metapage
```

函数入口要求：

```text
目标 buffer 已 pin
目标 buffer 已 write-locked
返回时释放这个 buffer 的 pin 和 lock
```

当 `cbuf` 有效时，说明现在是在 parent page 插入下层 split 产生的 downlink。
`nbtinsert.c:1113-1115` 说明 `_bt_insertonpg()` 会清 `cbuf` 的 incomplete flag，并释放它。
这就是 parent insertion 与 child flag 清理绑定的入口。
普通 insert 分支在 `nbtinsert.c:1296-1421`。
它在 critical section 中：

```text
PageAddItem()
MarkBufferDirty()
必要时更新 metapage fastroot
如果是 internal insert，清 child 的 BTP_INCOMPLETE_SPLIT
写 XLOG_BTREE_INSERT_LEAF / INSERT_UPPER / INSERT_META / INSERT_POST
设置相关 page LSN
```

关键点：

```text
清 child incomplete flag 和插入 parent downlink 是同一个 critical section。
同一个 WAL record redo 时也会同时表达这件事。
```

如果普通 leaf insert 空间足够，它只是 `XLOG_BTREE_INSERT_LEAF`。
如果 parent insert 空间足够，它是 `XLOG_BTREE_INSERT_UPPER`。
如果还更新 fastroot metapage，它是 `XLOG_BTREE_INSERT_META`。
split 分支在 `nbtinsert.c:1218-1264`。
当 `PageGetFreeSpace(page) < itemsz`：

```text
rbuf = _bt_split(...)
PredicateLockPageSplit(...)
_bt_insert_parent(...)
```

`_bt_split()` 返回时，左右 child 都还被 write-locked。
`_bt_insert_parent()` 负责在合适时释放它们。
这不是偶然。
左 child 的 incomplete flag 必须等 parent downlink 插入完成时才清。

## 7. split point 不是简单对半

`_bt_findsplitloc()` 位于 `nbtsplitloc.c`。
`nbtsplitloc.c:88-128` 给出核心语义：

```text
目标是在计入 incoming tuple 后，让 split 后两页 free space 合理均衡。
如果是 rightmost page，倾向按 fillfactor 留空间。
候选 split point 还会考虑 suffix truncation 的效果。
返回 firstrightoff 和 newitemonleft。
```

这里有三个容易误解的点。
第一，incoming item 必须参与计算。
如果只按原 page 上已有 item 分半，最后可能发现新 item 应该落到的那半页仍然放不下。
`nbtsplitloc.c:90-93` 专门强调这一点。
第二，rightmost leaf 的 split 不是 50:50。
对于递增 key，比如 sequence 或 timestamp，rightmost leaf 如果总是对半切，会留下大量右侧空间但很快继续打满右页。
`nbtsplitloc.c:95-100` 说明 rightmost page 使用 leaf fillfactor，使连续递增插入形成接近 build index 时的填充状态。
第三，leaf split point 还要考虑 high key 的 suffix truncation。
候选 split point 不只比较空间 delta。
`nbtsplitloc.c:109-113` 说明 suffix truncation 是次要目标，但在大量 duplicates 时会更重要。
`nbtsplitloc.c:1132-1154` 的 `_bt_split_penalty()` 对 leaf 使用 `_bt_keep_natts_fast()` 估算需要保留多少 key attributes。
所以 split point 的 mental model 是：

```text
先保证两页都能容纳实际 item。
再尽量符合 rightmost fillfactor 或普通均衡。
再在可接受范围内选更好的 separator/high key。
```

这会影响实验现象。
你看到 split 后 left/right page 的 item 数不完全相等，不一定是异常。
宽 tuple、duplicates、rightmost pattern、suffix truncation 都会改变分布。

## 8. `_bt_split()` 的本层原子动作

`_bt_split()` 在 `nbtinsert.c:1462-2111`。
它的输入是已 write-locked 的原 page。
它的输出是新右页 `rbuf`，同样 pinned 并 write-locked。
原 page 的 pin 和 write lock 保持不变。
函数前半段尽量在临时内存页中准备。
`nbtinsert.c:1523-1535` 说明：

```text
leftpage 是临时 buffer，最后复制回 origpage。
rightpage 先在临时位置构造。
在拿到新右页 buffer 前，尽量避免对原页留下持久影响。
```

这符合一个常见内核写法：

```text
能失败的准备工作放在持久修改之前。
真正改 buffer 后进入 critical section。
critical section 内不允许普通 ERROR 打断。
```

`_bt_split()` 先调用 `_bt_findsplitloc()`。
然后初始化新的 leftpage opaque。
`nbtinsert.c:1579-1582` 是本节核心状态变化：

```text
lopaque->btpo_flags = oopaque->btpo_flags;
lopaque->btpo_flags &= ~(BTP_ROOT | BTP_SPLIT_END | BTP_HAS_GARBAGE);
lopaque->btpo_flags |= BTP_INCOMPLETE_SPLIT;
```

这说明 split 一开始就把左页设计成：

```text
不再是 root
不继承 split_end / garbage hint
明确标记 incomplete split
```

随后 `_bt_split()` 处理 high key。
leaf page split 会尝试 `_bt_truncate()` 生成左页新 high key。
internal page split 不做 suffix truncation，而直接使用 `firstright` 作为 left high key。
`nbtinsert.c:1687-1713` 解释了 internal page 为什么不能随便截断 separator key。
拿到新右页 buffer 后，设置 sibling links：

```text
left.btpo_next = right block
right.btpo_prev = original block
right.btpo_next = old right sibling
```

如果原 page 不是 rightmost，还要 lock 原右 sibling，并把它的 `btpo_prev` 改到新右页。
`nbtinsert.c:1907-1911` 说明同层锁按从左到右顺序耦合，避免死锁。
真正的持久修改从 `START_CRIT_SECTION()` 开始。
`nbtinsert.c:1952-2095` 里完成：

```text
把临时 leftpage 复制回原 page
把临时 rightpage 复制到新右页 buffer
MarkBufferDirty(left)
MarkBufferDirty(right)
必要时更新旧右 sibling 的 prev link
如果本次 split 是 parent insert 的一部分，清下层 child incomplete flag
写 XLOG_BTREE_SPLIT_L 或 XLOG_BTREE_SPLIT_R
设置 left/right/sibling/child LSN
```

`XLOG_BTREE_SPLIT_L` 和 `XLOG_BTREE_SPLIT_R` 的区别只是 incoming item 在左还是右。
它们都是本层 split record。
它们不插 parent downlink。
`nbtinsert.c:2082-2083` 是 WAL 插入：

```text
xlinfo = newitemonleft ? XLOG_BTREE_SPLIT_L : XLOG_BTREE_SPLIT_R;
recptr = XLogInsert(RM_BTREE_ID, xlinfo);
```

到这里，数据结构处于一个可搜索但未完成的状态：

```text
左页有新的 high key
左页 rightlink 指向新右页
新右页保存右半数据
旧右 sibling 的 leftlink 已更新
左页 BTP_INCOMPLETE_SPLIT 已设置
parent 里还没有新右页 downlink
```

这正是本节的中间态。

## 9. split WAL 记录表达什么

`nbtxlog.h:89-159` 定义 split WAL record。
注释说明：

```text
右页的 item 会按 _bt_restore_page() 格式完整记录。
左页按普通增量方式处理。
左页 high key 总是记录。
record 包含 level、firstrightoff、newitemoff、postingoff。
```

为什么右页要完整记录？
因为新右页通常是 newly initialized page。
如果用普通增量记录，full page image 可能更大或更不可控。
把右页 tuple data 作为 WAL data 记录，redo 可以直接重建右页。
`nbtxlog.c:262-300` 的 `btree_xlog_split()` 先重建右页：

```text
XLogInitBufferForRedo(record, 1)
_bt_pageinit()
设置 btpo_prev/btpo_next/btpo_level/btpo_flags
_bt_restore_page()
```

随后重建左页。
`nbtxlog.c:414-419` 在 redo 中设置：

```text
oopaque->btpo_flags = BTP_INCOMPLETE_SPLIT;
if (isleaf)
    oopaque->btpo_flags |= BTP_LEAF;
oopaque->btpo_next = rightpagenumber;
```

注意 redo 中不试图恢复所有 hint。
`nbtxlog.c:1111-1116` 的 consistency mask 会忽略 `BTP_SPLIT_END` 和 `btpo_cycleid` 差异。
这说明它们不是 crash safety 的核心事实。
真正 crash-safe 的核心是：

```text
right page 能被重建
left page key range 和 rightlink 能被恢复
left page incomplete flag 能被恢复
后续 parent insertion 或 helper 能识别缺失 downlink
```

`nbtxlog.c:425-450` 还会修旧右 sibling 的 leftlink，并在最后一起释放同层相关 buffer。
注释说这是为了避免 reader 观察到同层不一致。
同层 sibling link 对顺序 scan 和 move-right 都重要。

## 10. parent insertion 如何补齐 downlink

`_bt_split()` 返回后，`_bt_insertonpg()` 调用 `_bt_insert_parent()`。
`nbtinsert.c:1238-1254` 的注释说此时：

```text
目标页已经 split
原 tuple 已插入某一半
左右 child 都 write-locked
parent 要插入的 key 是左 child 的 high key
```

`_bt_insert_parent()` 在 `nbtinsert.c:2114-2256`。
它处理两种情况：

```text
split 的是真 root
split 的不是 true root
```

如果 split true root，调用 `_bt_newlevel()`。
`nbtinsert.c:2473-2657` 创建新 root，并更新 metapage。
新 root 有两个 downlink：

```text
minus infinity pivot -> old left root
left high key pivot  -> new right page
```

`nbtinsert.c:2595-2598` 在同一个 critical section 中清左 child 的 incomplete flag。
随后写 `XLOG_BTREE_NEWROOT`。
redo 侧 `nbtxlog.c:950-957` 在恢复新 root 时也会清 left child flag。
如果 split 的不是 true root，`_bt_insert_parent()` 会：

```text
从左 child 的 high key 复制 pivot tuple
把 downlink 设置为新右页 block
用 _bt_getstackbuf() 重新找到 parent page 中指向左 child 的 pivot
释放右 child lock
调用 _bt_insertonpg(parent, cbuf = left child, new_item, offset = old offset + 1)
```

`nbtinsert.c:2208-2215` 是构造 parent downlink 的核心：

```text
ritem = left page P_HIKEY
new_item = CopyIndexTuple(ritem)
BTreeTupleSetDownLink(new_item, right block)
```

这说明 parent 里新 pivot 的 key 来自左页 high key。
它代表：

```text
大于左子树 key range 上界的 key，应该走到新右页子树。
```

为什么需要 `_bt_getstackbuf()` 重新找 parent？
初始 descent stack 只是“当时我从哪个 parent offset 下来”的记录。
从下降到 split 之间，parent 可能也被别人 split 或删除调整。
`nbtinsert.c:2216-2224` 明确说 child 的 downlink 位置甚至 parent 本身都可能改变。
`_bt_getstackbuf()` 通过 child block number 在线性搜索 parent pivot，必要时向右走。
找到 parent 后，调用 `_bt_insertonpg()` 插入 parent downlink。
这个 parent page 也可能满。
于是同一套逻辑递归：

```text
parent 有空间
  -> INSERT_UPPER
  -> 清 child incomplete flag
parent 无空间
  -> split parent
  -> 本层 parent split WAL
  -> parent 的 parent insertion
```

这就是 cascade split。
它不是一个 WAL record。
它是一串本层 split record 加上每一层的 parent insertion record。

## 11. incomplete split 的生命周期

现在把 `BTP_INCOMPLETE_SPLIT` 当成一个对象生命周期看。
创建者：
`_bt_split()` 在准备左页时设置。
真正持久化是在 split critical section 中把 leftpage 复制回 origpage，并写 split WAL。
持有者：
状态存在 index page special area。
它不是 backend-local。
任何 backend 读到这个 page 都能看到 flag。
但不同调用者的处理不同。
正常释放者：
`_bt_insertonpg()` 在 internal page insert 分支中清 child `cbuf` 的 flag。
这和 parent downlink insert 是同一个 critical section、同一个 WAL action。
root split 释放者：
`_bt_newlevel()` 创建新 root 时清 left child flag。
这和新 root/metapage 更新一起写 `XLOG_BTREE_NEWROOT`。
异常后释放者：
后续写路径遇到 flag 后调用 `_bt_finish_split()`。
它 lock 左页和右页，判断是否 root split，再调用 `_bt_insert_parent()`。
redo 释放者：
`btree_xlog_insert()` 对 `INSERT_UPPER`/`INSERT_META` 调用 `_bt_clear_incomplete_split(record, 1)`。
`btree_xlog_newroot()` 对 `NEWROOT` 调用 `_bt_clear_incomplete_split(record, 1)`。
读者：
普通 index scan 不负责清 flag。
README `652-655` 说理论上 search 也能修，但 PostgreSQL 不愿把更新放进 read-only 操作。
Hot Standby 更不能做这种更新。
所以生命周期是：

```text
_bt_split()
  -> left page flag set
  -> split WAL durable
  -> _bt_insert_parent()
     -> parent downlink durable
     -> left page flag clear
如果中断：
  -> flag 留在 page
  -> later write path
     -> _bt_finish_split()
     -> same parent insertion protocol
```

这就是 durable repair marker。

## 12. 谁会帮忙完成 incomplete split

第一类 helper 是 write descent。
`nbtsearch.c:232-235` 说明 `_bt_moveright()` 在 `forupdate` 为 true 时，会尝试完成遇到的 incomplete split。
`nbtsearch.c:286-304` 的逻辑是：

```text
如果 forupdate 且当前 page 有 P_INCOMPLETE_SPLIT
  -> 必要时把 read lock 升级为 write lock
  -> 再检查 flag
  -> 调用 _bt_finish_split()
  -> 重新获取原 page lock 并重查
```

这发生在 insert 从 root 向下走、或因为 high key 需要 move right 时。
它保证写路径不会越过结构债务继续制造新的写。
第二类 helper 是 `_bt_stepright()`。
`nbtinsert.c:1041-1086` 用在 insertion 定位过程中向右走。
`nbtinsert.c:1061-1071` 遇到右侧 page incomplete，会调用 `_bt_finish_split()`。
注释承认这会在持有左 sibling lock 时做较长操作，但应该很少发生。
第三类 helper 是 `_bt_getstackbuf()`。
`nbtinsert.c:2350-2455` 负责在 ascent 时重新找到 parent 中指向 child 的 pivot tuple。
如果它碰到 parent level page 自己 incomplete，`nbtinsert.c:2369-2374` 会先 `_bt_finish_split()`。
这对 page split 和 page deletion 都重要。
第四类有限 helper 是 VACUUM page deletion。
`nbtpage.c:2898-2910` 说明 `_bt_getstackbuf()` 在删除子树时会完成返回 parent buffer 上需要完成的 internal page split。
但 VACUUM 通常不主动完成 leaf incomplete split。
README `656-663` 给出原因：
VACUUM 的目的可能正是回收空间。
如果为了修 downlink 需要 split parent 并分配新 page，可能因为 out-of-disk 让 VACUUM 更糟。
所以 helper 责任不是“任何人看到都修”。
它是：

```text
写路径必须修。
读路径不修。
VACUUM 只在删除路径需要且收益足够大时有限修。
```

## 13. 为什么 search 仍然正确

Lehman-Yao 的核心是 high key 和 rightlink。
README 开头说明：
当 search 从 parent downlink 到 child 后，会用 page high key 检查 search key 是否已经超过本页范围。
如果超过，就沿 rightlink 向右追。
并发 split 可能让 parent downlink 指向旧左页。
search 仍能从旧左页追到新右页。
incomplete split 也是同一种可达性。
parent 暂时没有新右页 downlink。
但 parent 里仍有旧左页 downlink。
search 落到左页后，看到 key 大于左页 high key，就沿 rightlink 到右页。
因此 missing downlink 的直接影响不是错误结果，而是额外 move-right。
README `642-650` 说得很清楚：
crash 可能发生在 split 和 parent insertion 之间。
恢复后新页 downlink 缺失。
search algorithm works correctly。
但如果许多 downlink 缺失，性能会受影响。
更严重的是，如果缺 downlink 的 page 再被 split，插入算法无法在 parent level 找到正确插入位置。
这解释了为什么读者能容忍，写者不能容忍。
读者只需要找到 key。
写者可能要继续 split，并需要精确的 parent structural anchor。

## 14. WAL 原子性层次

本节涉及三类 WAL record。
第一类，普通 insert：

```text
XLOG_BTREE_INSERT_LEAF
XLOG_BTREE_INSERT_POST
```

它们只修改 leaf page。
第二类，本层 split：

```text
XLOG_BTREE_SPLIT_L
XLOG_BTREE_SPLIT_R
```

它们修改：

```text
原 page / 新左页
新右页
旧右 sibling，若存在
下层 child，若这是 internal page split 同时完成了更低层 split
```

但它们不插当前 split 产生的新右页 downlink 到 parent。
第三类，parent completion：

```text
XLOG_BTREE_INSERT_UPPER
XLOG_BTREE_INSERT_META
XLOG_BTREE_NEWROOT
```

它们完成：

```text
在 parent 或 new root 中插入 downlink
清 child left page 的 BTP_INCOMPLETE_SPLIT
必要时更新 metapage
```

`nbtxlog.c:131-155` 的 `_bt_clear_incomplete_split()` 是 redo 侧公共函数。
`nbtxlog.c:166-176` 说明 internal page insertion 完成 child level incomplete split。
正常运行时 child 和 parent 同时加锁，让“插 downlink + 清 flag”对其他 backend 原子可见。
redo 时没有并发 writer，不需要重建跨层 lock coupling。
这个边界很重要：

```text
WAL 原子性不是“整棵树一次一致”。
WAL 原子性是“每条 record redo 后，tree 对读者仍一致；未完成的跨层结构债务有 durable marker”。
```

## 15. ERROR、abort 与 fallback

内核课程不能只讲 happy path。
下面列本节最关键的异常路径。
第一，index tuple 太大。
`_bt_findinsertloc()` 在 `nbtinsert.c:843-846` 检查 `BTMaxItemSize`。
超过限制会进入 `_bt_check_third_page()` 并报错。
这发生在 split 前。
不会留下 incomplete split。
第二，唯一性检查等待。
`_bt_doinsert()` 在 `nbtinsert.c:216-235` 发现需要等其他事务时，释放 buffer，等待后重新 search。
这是 concurrency retry。
它不是 incomplete split，但体现了同一个原则：

```text
等待后不要相信旧 leaf 定位。
```

第三，leaf full 但可清理。
`_bt_delete_or_dedup_one_page()` 会先删 `LP_DEAD`、做 bottom-up deletion 或 dedup。
如果腾出空间，避免 split。
如果没腾出空间，回到 split。
这是一种 performance fallback。
第四，split 准备阶段失败。
`_bt_split()` 在真正改原 page 前尽量只使用临时页。
例如 left high key 添加失败，代码还没进入 persistent critical section。
这种 ERROR 不应留下半写 page。
第五，critical section 内失败。
一旦进入 `START_CRIT_SECTION()`，普通 `ereport(ERROR)` 就不能安全退出。
这里很多失败使用 `elog(PANIC)`。
含义是进程不能带着半写 shared buffer 继续运行。
crash 后依赖 WAL redo 恢复到 record 边界。
第六，split record 已写，但 parent insertion 失败。
这正是 incomplete split 的设计目标。
可能原因包括 OOM、out-of-disk、recoverable ERROR。
README `694-700` 明确说 incomplete split 不只来自 crash recovery，也可能来自可恢复错误。
left page flag 留下。
后续 insert helper 会完成。
第七，parent 无法重新定位。
`_bt_insert_parent()` 调用 `_bt_getstackbuf()`。
如果返回 `InvalidBuffer`，`nbtinsert.c:2242-2246` 报 index corrupted。
正常并发 split 应由 `_bt_getstackbuf()` 向右搜索恢复。
找不到通常意味着结构损坏或不符合 nbtree invariant。
第八，page deletion 遇到 incomplete split。
`nbtpage.c:1711-1749` 的 `_bt_leftsib_splitflag()` 检查目标页左 sibling 是否有 incomplete flag。
如果目标是 incomplete split 的右半页，它没有 parent downlink，page deletion 不准备处理。
所以 deletion 不能继续。
这些 fallback 的共同点是：

```text
能在持久修改前失败，就不留下结构变化。
已经留下结构变化，就必须留下 WAL 或 durable repair marker。
不能确定 parent anchor，就拒绝继续结构删除或结构插入。
```

## 16. ownership 与 cleanup

buffer ownership：
`_bt_search_insert()` 返回时，`insertstate.buf` 是 leaf page buffer，已 pin 且 write-locked。
`_bt_insertonpg()` 承诺返回前释放目标 buffer。
如果 split，`_bt_split()` 返回新右页 `rbuf`，左右页都仍 write-locked。
`_bt_insert_parent()` 接管这些 lock 的释放。
stack ownership：
descent stack 是 backend-local memory。
`_bt_doinsert()` 结束时用 `_bt_freestack()` 释放。
如果唯一性等待后重试，也释放旧 stack。
stack 不是 shared state。
它只是重新定位 parent 的 hint。
`_bt_getstackbuf()` 可以修正 stack 中的 block/offset。
page state ownership：
page opaque flags 属于 shared buffer 和磁盘 page。
修改必须持有 buffer content lock。
持久修改要配合 WAL 和 page LSN。
WAL ownership：
`XLogInsert()` 返回 LSN。
代码在 critical section 内把相关 page 的 LSN 设置成该 recptr。
这让 buffer manager 和 checkpoint 能遵守 WAL-before-data。
ERROR cleanup：
普通 ERROR 会释放 buffer locks 和 pins，靠 ResourceOwner 清理外部资源。
但如果 ERROR 发生在 split WAL 已经写完、parent insertion 前，ResourceOwner 只能释放锁和 pin。
它不会撤销 page split。
结构修复靠 `BTP_INCOMPLETE_SPLIT`。
这个边界要分清：

```text
ResourceOwner 负责“我还持有哪些资源”。
WAL 负责“crash 后能恢复到哪个物理状态”。
BTP_INCOMPLETE_SPLIT 负责“这个物理状态还欠哪一步结构修复”。
```

不要说“事务 abort 会回滚 index split”。
PostgreSQL 不会物理回滚 nbtree page split。
abort 后 heap tuple 不可见，对应 index tuple 以后可被清理。
但 split 造成的 page 和 sibling structure 会保留。

## 17. 正确性机制层次

本节正确性不是一个机制保证的。
第一层，Lehman-Yao high key/rightlink。
它保证 parent 过旧时 search 能向右恢复。
这层让 missing downlink 不破坏读正确性。
第二层，buffer content lock。
page 内 tuple array、opaque links、flags 的修改要在 write lock 下完成。
读者在 read lock 下看一个 page 的自洽快照。
第三层，同层 lock ordering。
split 更新旧右 sibling leftlink 时按 left-to-right lock。
这避免同层 deadlock。
第四层，跨层 lock coupling。
primary 上 parent insertion 会同时持有 child left page 和 parent page lock。
目的是让“插 parent downlink + 清 child incomplete flag”对其他 writer 原子可见。
README `712-724` 解释 replay 不需要重建这种 coupling，因为 recovery 没有并发 writer。
第五层，critical section。
一旦开始改 shared buffer，就不能普通 ERROR 中断。
失败会升级为 PANIC，交给 crash recovery。
第六层，WAL-before-data。
split、insert upper、new root 都写 WAL 并设置 page LSN。
数据页落盘前，对应 WAL 必须先持久化。
第七层，predicate lock。
`_bt_insertonpg()` split 后调用 `PredicateLockPageSplit()`。
这是 SSI 相关边界。
它不维护物理结构，但 split 改变了 predicate lock 需要覆盖的 page。
第八层，VACUUM 与 page deletion 检查。
page deletion 不会删除 rightmost/root/non-empty/incomplete split page。
这避免删除路径在 parent downlink 缺失时做错误重定位。

## 18. 成本、资源与跨模块传播

普通 leaf insert 的成本大致是：

```text
一次 root-to-leaf search
一个 leaf page write lock
一条 INSERT_LEAF WAL
一个 page dirty
```

leaf split 的成本变成：

```text
选择 split point
重写左页和新右页
可能锁旧右 sibling
一条 SPLIT_L/R WAL
parent 重新定位
一条 INSERT_UPPER 或 parent split WAL
可能递归 cascade
```

成本随这些变量扩张：
- key 插入模式：rightmost 递增、随机插入、局部递增 composite key 会影响 split point 和页利用率。
- tuple 宽度：index tuple 越宽，每页 item 越少，split 更频繁，WAL 更大。
- duplicates：可能触发 dedup，也可能让 split point 选择更复杂。
- UPDATE churn：产生更多旧 index tuples，bottom-up deletion 是否能避免 split 很关键。
- tree height：height 越高，parent insertion 和 cascade split 的上界越高。
- WAL bandwidth：split record 和 full page image 会推高 WAL bytes。
- contention：rightmost insert 在高并发下会争抢 leaf write lock，fastpath 会在 lock 不可得时放弃。
资源传播到相邻模块：
- buffer manager：更多 dirty index pages，更多 buffer lock、pin 和 writeback 压力。
- WAL subsystem：更多 `RM_BTREE_ID` record、可能更多 FPI、更多 WAL flush/replication lag。
- checkpointer/bgwriter：dirty index page 最终要写回。
- walwriter/archiver/replication：WAL 量变大后复制和归档链路承压。
- VACUUM：page split 增加 leaf page 数，后续 index cleanup 扫描成本上升；但 bottom-up deletion/dedup 能减少部分 split。
- pageinspect/diagnostics：只能看到 page 当前物理状态，不提供完整因果。
一个实践判断：

```text
看到 btree index 变大，不要直接归因于 split 算法差。
先区分 key pattern、fillfactor、UPDATE churn、dedup 可用性、autovacuum 延迟、WAL/checkpoint 压力。
```

## 19. 观测入口

能直接观测的状态：

```text
bt_metap()    root、level、fastroot、fastlevel
bt_page_stats()  page type、live/dead items、free_size、btpo_prev、btpo_next、btpo_level、btpo_flags
bt_page_items()  high key 和 item offset
pg_stat_wal      cluster 级 WAL records、FPI、wal_bytes
EXPLAIN (ANALYZE, WAL) 单 query WAL 量
pg_waldump --rmgr=Btree  Btree WAL record 类型和字段
```

`contrib/pageinspect/btreefuncs.c:300-311` 显示 `bt_page_stats()` 会输出 `btpo_flags`。
`BTP_INCOMPLETE_SPLIT` 是 bit 7，即 decimal 128。
实验中可用：

```sql
(btpo_flags & 128) <> 0 AS incomplete_split
```

只能推断的状态：

```text
某次 SQL insert 是否触发了某个具体 page split。
parent downlink 是否曾短暂缺失。
helper 是否正好由哪个 backend 完成。
split point 为什么选这个 offset 的全部成本权衡。
```

需要源码断点或 injection point 的状态：

```text
_bt_split() 返回后、_bt_insert_parent() 前的短暂 incomplete split。
_bt_finish_split() 被哪个路径触发。
_bt_getstackbuf() 是否因为 parent page split 向右移动。
```

`pg_waldump` 能看到 Btree record 名称。
`nbtdesc.c:138-190` 定义了文本名称：

```text
INSERT_LEAF
INSERT_UPPER
INSERT_META
SPLIT_L
SPLIT_R
INSERT_POST
NEWROOT
```

`nbtdesc.c:41-49` 对 split record 打印：

```text
level
firstrightoff
newitemoff
postingoff
```

观测边界：
`pg_stat_wal` 是实例累计。
`EXPLAIN WAL` 是单 query 级别。
`pg_waldump` 是 WAL segment 级别。
`pageinspect` 是当前 page 物理状态。
它们都不能单独证明完整因果。
需要把时间窗口、LSN 范围、page state 和源码路径合在一起解释。

## 20. 课堂实验一：观察普通 split 与 WAL record

目标：触发 leaf split，用 `pageinspect` 看 page opaque，用 `pg_waldump` 看 Btree WAL。

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS btsplit_demo;
CREATE TABLE btsplit_demo(id bigint PRIMARY KEY, pad text);
SELECT pg_current_wal_lsn() AS start_lsn \gset
INSERT INTO btsplit_demo
SELECT g, repeat('x', 20)
FROM generate_series(1, 200000) AS g;
SELECT pg_current_wal_lsn() AS end_lsn \gset
```

观察结构：

```sql
SELECT * FROM bt_metap('btsplit_demo_pkey');
SELECT blkno, type, live_items, free_size,
       btpo_prev, btpo_next, btpo_level, btpo_flags,
       (btpo_flags & 128) <> 0 AS incomplete_split
FROM generate_series(1, 20) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('btsplit_demo_pkey', g.blkno)
ORDER BY blkno;
```

预期：leaf page 的 `btpo_level = 0`，leaf 之间有 sibling links，正常情况下 `incomplete_split = false`。
rightmost leaf 的 free space 可能不像 50:50 split，因为 `_bt_findsplitloc()` 会考虑 fillfactor。
观察 WAL：

```bash
pg_waldump --rmgr=Btree --start="$start_lsn" --end="$end_lsn" "$PGDATA/pg_wal"
```

重点找：

```text
SPLIT_L / SPLIT_R: 本层 split
INSERT_UPPER / INSERT_META: parent downlink insertion
NEWROOT: root split 后创建新 root
```

如果看到 `SPLIT_R level: 0` 后出现 `INSERT_UPPER`，就是 leaf split 后 parent insertion。
如果 parent 也 split，不要期待每个 split 后都紧跟一个简单 `INSERT_UPPER`。
对应源码：`nbtinsert.c:2082`、`nbtinsert.c:1362`、`nbtinsert.c:2639`、`nbtdesc.c:41-49`。

## 21. 课堂实验二：用 injection point 留下 incomplete split

目标：让 `_bt_split()` 完成后、`_bt_insert_parent()` 前报错，再观察后续 insert 帮忙完成 split。
要求：PostgreSQL 需要以 injection points 支持构建。
相关点位：

```text
nbtree-leave-leaf-split-incomplete
nbtree-leave-internal-split-incomplete
nbtree-finish-incomplete-split
```

触发故障：

```sql
CREATE EXTENSION IF NOT EXISTS injection_points;
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS btsplit_incomplete;
CREATE TABLE btsplit_incomplete(id bigint PRIMARY KEY, pad text);
INSERT INTO btsplit_incomplete
SELECT g, repeat('x', 20)
FROM generate_series(1, 50000) AS g;
SELECT injection_points_set_local();
SELECT injection_points_attach('nbtree-leave-leaf-split-incomplete', 'error');
INSERT INTO btsplit_incomplete
SELECT g, repeat('x', 20)
FROM generate_series(50001, 100000) AS g;
```

预期错误：

```text
ERROR:  error triggered for injection point nbtree-leave-leaf-split-incomplete
```

这时 split WAL 可能已经写入，parent insertion 被打断。
清掉错误注入，打开 finish notice，再插入：

```sql
ROLLBACK;
SELECT injection_points_detach('nbtree-leave-leaf-split-incomplete');
SELECT injection_points_attach('nbtree-finish-incomplete-split', 'notice');
SET client_min_messages = notice;
INSERT INTO btsplit_incomplete
SELECT g, repeat('x', 20)
FROM generate_series(100001, 120000) AS g;
```

如果 helper 触发，会看到 `nbtree-finish-incomplete-split` notice。
再扫 repair marker：

```sql
SELECT blkno, btpo_prev, btpo_next, btpo_level, btpo_flags,
       (btpo_flags & 128) <> 0 AS incomplete_split
FROM generate_series(1, pg_relation_size('btsplit_incomplete_pkey') / 8192 - 1) AS g(blkno)
CROSS JOIN LATERAL bt_page_stats('btsplit_incomplete_pkey', g.blkno)
WHERE (btpo_flags & 128) <> 0
ORDER BY blkno;
```

正常完成后应为空。
回到源码：错误点在 `nbtinsert.c:1256-1261`，修复点在 `nbtinsert.c:2312`。

## 22. 课堂实验三：用 gdb 跟踪一次 split

目标：不用改产品代码，通过断点观察状态推进。

```gdb
break _bt_doinsert
break _bt_findinsertloc
break _bt_insertonpg
break _bt_split
break _bt_insert_parent
break _bt_finish_split
break btree_xlog_split
break btree_xlog_insert
```

在 `_bt_split()` 中观察 `origpagenumber`、`rightpagenumber`、`firstrightoff`、`newitemonleft` 和 left page `btpo_flags`。
在 `_bt_insert_parent()` 中观察 `bknum`、`rbknum`、`stack->bts_blkno`、`stack->bts_offset`。
在 parent `_bt_insertonpg()` critical section 后观察 `cbuf` 的 `btpo_flags`。
验证路径：

```text
_bt_split(): left page 获得 BTP_INCOMPLETE_SPLIT
_bt_insert_parent(): right block downlink 被构造
_bt_insertonpg(parent): parent insert 与 child flag clear 绑定
_bt_finish_split(): 故障注入或遗留 incomplete split 后才出现
```

`_bt_split()` 中看到 incomplete flag 不是 bug。
它是设计中的中间状态。

## 23. 常见误区

误区一：
“leaf full 就一定 split。”
实际路径先尝试 simple deletion、bottom-up deletion 和 dedup。
split 是这些办法不能腾出空间后的选择。
误区二：
“split 是一个跨层 WAL 原子动作。”
实际是本层 split WAL，加后续 parent insertion WAL。
cascade split 是多条 WAL record。
误区三：
“新右页没有 parent downlink 会导致查询错。”
不会。
rightlink/high key 能让 search 从左页追到右页。
问题主要是性能和后续结构更新。
误区四：
“incomplete split 应该标在右页。”
PostgreSQL 标在左页。
因为从左页沿 rightlink 到右页时，正好能知道右页 downlink 缺失。
误区五：
“读者看到 incomplete split 会修。”
普通读 search 不修。
修复责任在写路径。
Hot Standby 更不能做更新。
误区六：
“事务 abort 会回滚 index split。”
不会。
index tuple 和 page split 是物理结构修改。
abort 影响 heap tuple 可见性，后续清理会回收无用 index tuple，但 split page 结构保留。
误区七：
“`btpo_flags` 一个数字能解释所有状态。”
不能。
需要结合 `btpo_prev`、`btpo_next`、`btpo_level`、high key、parent downlink 和锁上下文。
误区八：
“`pg_stat_wal` 增长说明一定是 btree split。”
不一定。
它是实例累计。
要用 LSN 窗口和 `pg_waldump --rmgr=Btree` 缩小范围。

## 24. 讨论题

1. 为什么 PostgreSQL 不把 leaf split、parent insertion、root split 放进一条巨大 WAL record？
2. 为什么 parent downlink 缺失时 search 仍然正确，但后续 insertion 不能继续忽略？
3. `BTP_INCOMPLETE_SPLIT` 为什么标在左页，而不是缺 downlink 的右页？
4. `_bt_getstackbuf()` 为什么按 child block number 重新找 parent pivot，而不是相信 descent stack offset？
5. 如果 `_bt_split()` 成功后发生 OOM，哪些状态会留在磁盘上？谁负责清理？
6. 为什么 VACUUM 通常不主动完成 leaf incomplete split？
7. 如何用 `pg_waldump` 区分 leaf split、parent insertion 和 root split？
8. `pageinspect` 看到 `(btpo_flags & 128) <> 0` 时，还需要哪些信息才能判断影响范围？

## 25. 本节小结

本节主链路是：

```text
_bt_doinsert()
  -> 找 leaf
  -> leaf full
  -> delete/dedup fallback
  -> _bt_split()
  -> XLOG_BTREE_SPLIT_L/R
  -> 左页 BTP_INCOMPLETE_SPLIT
  -> _bt_insert_parent()
  -> INSERT_UPPER / INSERT_META / NEWROOT
  -> 清 incomplete flag
```

核心状态是 `BTPageOpaqueData`。
`btpo_prev`、`btpo_next`、`btpo_level`、`btpo_flags` 和 page high key 一起定义 nbtree page 的结构语义。
`BTP_INCOMPLETE_SPLIT` 不是 corruption 标记。
它表示 split 后 parent downlink 尚未补齐。
正确性来自多层机制组合：

```text
high key/rightlink 保证 search 可达
buffer lock 保证 page 内一致
critical section 防止半写退出
WAL 保证 crash recovery
parent insert + clear flag 保证结构修复原子可见
helper 路径保证异常后最终修复
```

ERROR 和 crash 的核心处理方式是：

```text
持久修改前失败，不留下结构变化。
本层 split 已持久化后失败，留下 incomplete split。
后续写路径通过 _bt_finish_split() 完成 parent insertion。
```

可观测性要分层：
`pageinspect` 看当前 page opaque 和 item。
`pg_waldump` 看 Btree WAL record。
`pg_stat_wal` 和 `EXPLAIN WAL` 看 WAL 量。
短暂 incomplete split 通常只能靠 injection point、gdb 或故障窗口捕捉。
可迁移的系统规律：

```text
当一个结构更新天然跨多个局部原子边界时，可靠系统常把每一步设计成读者可接受的中间态，并持久化一个可重试的 repair marker，而不是试图扩大单次原子动作覆盖所有层级。
```

这个规律在 nbtree page split 中表现为 incomplete split。
在其它内核模块里，你也会看到类似模式：
先保证最低限度可达或可恢复，再把昂贵的全局一致性修复交给后续可重试路径。
