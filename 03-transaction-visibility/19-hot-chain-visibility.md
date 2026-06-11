# PostgreSQL HOT chain 可见性

## 课程定位

前置知识：已经理解普通 UPDATE 会生成新 tuple version，并通过旧版本的 `t_ctid` 连接后继；也理解 `HeapTupleSatisfiesMVCC()` 只判断单个 tuple version 对 snapshot 是否可见。

本节唯一主问题：

```text
HOT update 如何让一个索引入口代表同页上的多个版本，index scan 又如何沿 HOT chain 找到对当前 snapshot 可见的版本？
```

本节核心矛盾：

```text
UPDATE 不改变索引列时，系统希望避免为每个新版本插入重复索引项；
但 index scan 仍必须从旧索引入口出发，正确找到同一逻辑行当前对 snapshot 可见的版本，同时允许 page-local pruning 回收旧版本。
```

学完本节后，你应该能独立判断：

- HOT update 的基本条件是什么。
- 为什么 HOT child 被标记为 `HEAP_ONLY_TUPLE`。
- 为什么 root 或前驱版本要标记 `HEAP_HOT_UPDATED`。
- 为什么 HOT chain 必须限制在同一个 heap page。
- 为什么 index scan 进入 heap 后要调用 `heap_hot_search_buffer()`。
- 为什么 HOT chain 跟随仍要验证 `prev_xmax == child.xmin`。
- 为什么 pruning 可以把 root line pointer 改成 redirect。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 VACUUM index cleanup 的完整流程。

这里只讲 HOT chain 如何保持 index reachability 与 MVCC visibility。

## 1. 本节在总主线中的位置

上一节讲普通 UPDATE chain。

普通 update 如果生成新版本，通常需要为新版本建立索引项。

但有一种常见情况：

```text
UPDATE 只修改非索引列。
新版本能放在同一个 heap page。
```

这时为新版本插入一组完全相同的索引项很浪费。

PostgreSQL 的 HOT 机制让索引仍指向 chain root。

新版本作为 heap-only tuple 存在。

index scan 从 root TID 进入 heap page 后，沿同页 HOT chain 找到对当前 snapshot 可见的版本。

这就是 HOT 的主线：

```text
index entry 指向 root line pointer
  -> root tuple 或 redirect line pointer 进入 HOT chain
  -> heap_hot_search_buffer() 在同一 buffer 内遍历 chain
  -> 对每个 chain member 做 visibility 判断
  -> 返回当前 snapshot 可见的那个版本
```

HOT 不是改变 MVCC。

它是改变 index reachability。

单个 tuple version 的可见性仍由 visibility routine 决定。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
当 UPDATE 没有改变普通索引依赖列且新版本能留在同一 heap page 时，heap_update() 可以不给新版本建立普通索引项，而是把前驱标记为 HEAP_HOT_UPDATED、把新版本标记为 HEAP_ONLY_TUPLE；index scan 命中 root 后在同一 buffer 中调用 heap_hot_search_buffer() 沿 t_ctid 链寻找 snapshot 可见版本。
```

这里的 tension 是：

```text
减少重复索引项能降低 update 和 index bloat 成本；
但任何索引入口都不能指向一个已经无法找到当前可见版本的 heap chain。
```

HOT 的正确性依赖三个边界：

```text
链必须在同一 heap page。
不改变需要普通索引项重新定位的列。
每次跟随 chain 都要验证 line pointer 和 xmin/xmax 关系。
```

只要这些边界成立，HOT 就能同时获得：

```text
少写索引。
page-local pruning。
index scan 仍不丢行。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/README.HOT` | 先读 HOT 的设计动机、单索引入口、多版本链、redirect line pointer 和 pruning。 |
| 2 | `src/include/access/htup_details.h` | 对照 `HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE`、`HeapTupleHeaderIsHotUpdated()`、`HeapTupleHeaderIsHeapOnly()`。 |
| 3 | `src/backend/access/heap/heapam.c` | 看 `heap_update()` 如何计算 HOT 条件、设置 flags、决定 `TU_UpdateIndexes`。 |
| 4 | `src/backend/access/heap/heapam_indexscan.c` | 主读 `heapam_index_fetch_tuple()` 和 `heap_hot_search_buffer()`。 |
| 5 | `src/backend/access/heap/pruneheap.c` | 看 HOT pruning 如何处理 redirect、dead、unused line pointer。 |
| 6 | `src/include/access/heapam.h` | 对照 HOT search 的公开声明。 |
| 7 | `src/backend/access/heap/vacuumlazy.c` | 了解 VACUUM 如何最终清理 dead line pointer 和索引项。 |

推荐阅读顺序：

```text
先读 README.HOT 的图
  -> 读 htup_details.h 的两个 HOT flags
  -> 读 heap_update() 何时 use_hot_update
  -> 读 heapam_indexscan.c 如何从索引 TID 沿 chain 找可见 tuple
  -> 最后读 pruneheap.c 的 redirect line pointer
```

## 4. 一个 runtime 现象先定锚

构造一张表。

索引只建在 `id` 上。

反复更新非索引列 `payload`。

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS hot_demo;
CREATE TABLE hot_demo(id int primary key, payload text);

INSERT INTO hot_demo VALUES (1, 'v1');
UPDATE hot_demo SET payload = 'v2' WHERE id = 1;
UPDATE hot_demo SET payload = 'v3' WHERE id = 1;

SELECT xmin, xmax, ctid, * FROM hot_demo WHERE id = 1;

SELECT lp,
       lp_flags,
       lp_off,
       t_xmin,
       t_xmax,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('hot_demo', 0))
ORDER BY lp;
```

你通常会看到多个 tuple version 在同一个 page 上。

某些旧版本带 HOT-updated 标志。

后继版本可能是 heap-only tuple。

索引仍然可以通过 `id = 1` 找到当前版本。

关键现象是：

```text
新版本可能没有自己的普通索引入口；
但 index scan 仍能从 root 进入 heap page 并沿 HOT chain 找到它。
```

这就是 HOT 的 runtime anchor。

## 5. HOT 解决的问题

普通 UPDATE 如果每次都更新索引，会带来成本：

```text
写 heap 新版本。
写每个相关索引的新 entry。
旧索引 entry 以后要 vacuum cleanup。
索引 bloat 增加。
缓存局部性变差。
```

如果 UPDATE 没改变索引相关列，新旧版本的索引 key 相同。

为新版本插入一模一样的索引 entry，很多情况下只是重复。

HOT 让一个索引 entry 覆盖同页链上的多个版本。

旧 root line pointer 仍是索引入口。

新版本不需要普通索引项。

这个优化只在两个核心条件下成立：

```text
更新不改变普通索引需要精确指向 tuple 的列。
新版本放在同一个 heap page。
```

如果不满足，就不能 HOT。

因为索引入口将无法可靠找到正确版本。

## 6. HOT chain 的 header 标志

两个 flag 是 HOT 的核心：

```text
HEAP_HOT_UPDATED
HEAP_ONLY_TUPLE
```

`HEAP_HOT_UPDATED` 标在前驱 tuple 上。

它表示：

```text
这个 tuple 被 HOT update；
它的 t_ctid 指向同页 HOT 后继。
```

`HEAP_ONLY_TUPLE` 标在 child tuple 上。

它表示：

```text
这个 tuple 没有自己的普通索引入口。
它只能通过 HOT chain 从 root 找到。
```

这两个 flag 必须配合。

如果 child 没有索引入口，却不能从 root chain 到达，index scan 会丢行。

如果前驱错误标记 HOT，index scan 可能沿错链。

因此 `HeapTupleHeaderIsHotUpdated()` 不只看 `HEAP_HOT_UPDATED`。

它还检查：

```text
xmax 不是 invalid。
xmin 不是 invalid。
```

这避免 aborted update 造成错误链路。

## 7. HOT 为什么必须同页

README.HOT 强调 HOT chain 限制在同一个 page。

原因是 page-local pruning。

如果 child 在别的 page，index scan 从 root 跳到 child 会多一次 heap page fetch。

更重要的是，page-local cleanup 就无法只靠一个 buffer cleanup lock 完成。

HOT 的价值之一是：

```text
在不扫描索引、不调用用户定义函数、不跨页追踪的情况下，清理 page 内旧版本。
```

同页限制让系统可以：

```text
把 root line pointer 改成 redirect。
移除死的 heap-only child。
压缩 page 空间。
继续保留索引入口可达性。
```

如果跨页，redirect 和 pruning 的边界会复杂很多。

所以 HOT 选择牺牲一部分适用范围，换取强 page-local correctness。

## 8. `heap_update()` 如何决定 HOT

`heap_update()` 会计算修改了哪些列。

其中关键集合来自 relcache：

```text
INDEX_ATTR_BITMAP_HOT_BLOCKING
INDEX_ATTR_BITMAP_SUMMARIZED
INDEX_ATTR_BITMAP_KEY
INDEX_ATTR_BITMAP_IDENTITY_KEY
```

如果修改列不触碰 HOT-blocking 索引列，且新 tuple 能放在同一个 page，就可能使用 HOT。

HOT 成功时：

```text
旧版本:
  设置 HEAP_HOT_UPDATED。
  t_ctid 指向同页新版本。

新版本:
  设置 HEAP_ONLY_TUPLE。
  没有普通索引 entry。
```

返回给上层的索引更新决策也会不同。

如果不是 HOT：

```text
新版本需要普通索引项。
```

如果是 HOT：

```text
普通索引可以不插入重复 entry。
```

注意 summarized index 另有规则。

本节只抓住普通 HOT 主线。

## 9. index scan 如何进入 HOT chain

索引扫描先从索引得到一个 heap TID。

这个 TID 通常指向 HOT chain root。

在 heap AM 中，入口是：

```text
heapam_index_fetch_tuple()
```

它负责：

```text
读取 heap buffer。
必要时对 page 做 opportunistic pruning。
加 share lock。
调用 heap_hot_search_buffer()。
把找到的 tuple 存进 slot。
```

`heap_hot_search_buffer()` 做真正 chain 搜索。

它在同一个 buffer 中循环：

```text
从 root offset 开始。
如果 root line pointer 是 redirect，跟到 redirect target。
读取当前 tuple。
如果 chain start 是 HEAP_ONLY_TUPLE，说明异常，停止。
如果 prev_xmax 有效，要求 prev_xmax == current xmin。
判断当前 tuple 对 snapshot 是否可见。
如果可见，返回。
如果不可见且调用者关心 all_dead，检查是否全局 dead。
如果当前 tuple 是 HOT-updated，沿 t_ctid 到下一个 offset。
否则停止。
```

这个流程说明：

```text
HOT chain 的可见性不是一条新规则。
每个 chain member 仍调用 HeapTupleSatisfiesVisibility()。
```

HOT 只改变从索引入口到 tuple version 的到达方式。

## 10. `heap_hot_search_buffer()` 的校验

HOT chain 搜索中有三个关键防线。

第一，root 不能是 heap-only tuple。

root 是索引入口能到达的对象。

heap-only tuple 没有自己的索引入口。

如果链起点就是 heap-only，说明调用者给的 TID 不是合法 root。

第二，后继必须在同一 block。

源码中对 `HEAP_HOT_UPDATED` 后继有断言：

```text
t_ctid 的 block number 等于当前 buffer block。
```

第三，后继 `xmin` 必须匹配前驱 update xid。

这和上一节普通 chain 一样。

如果不匹配，说明 slot 可能被重用或链已经断裂。

正确行为是停止。

这些校验让 HOT 在允许 page pruning 的同时不误读无关 tuple。

## 11. MVCC snapshot 与 HOT chain

在 MVCC snapshot 下，一个 HOT chain 最多有一个可见成员。

`README.HOT` 也指出，MVCC 下找到可见 tuple 后可以停止。

原因是同一逻辑行的版本在一个 snapshot 中不会同时可见多个。

但非 MVCC snapshot 不一定满足这个优化。

例如 `SnapshotAny` 或某些内部视角可能需要继续。

所以 `heapam_index_fetch_tuple()` 在找到 tuple 后设置：

```text
*heap_continue = !IsMVCCLikeSnapshot(snapshot)
```

这让调用者知道是否还可能继续取同一 HOT chain 的其它成员。

这条边界很重要。

HOT chain 不是只服务普通 MVCC。

但普通 MVCC 是最常见路径。

## 12. pruning 与 redirect line pointer

HOT 的另一个价值是 page-local pruning。

当 root tuple 对所有事务都死了，但索引仍指向 root line pointer，系统不能直接删除 root line pointer。

否则索引 entry 会悬空。

HOT 的做法是：

```text
把 root line pointer 变成 redirect。
redirect 指向链上仍需要的后继。
```

这样索引仍指向 root offset。

heap_hot_search_buffer() 遇到 redirect 时跟到 redirect target。

当中间 heap-only tuple 全局 dead 后，可以把它 prune 掉。

因为没有索引直接指向它。

最终如果整条 chain 都 dead，root line pointer 可以标记 dead。

之后普通 VACUUM 可以清理索引 entry。

这就是 HOT 同页限制和 index reachability 的结合点。

## 13. 主流程源码 walkthrough

### 13.1 UPDATE 选择 HOT

```text
heap_update()
  -> 计算 modified_attrs
  -> 判断是否触碰 HOT-blocking attrs
  -> 判断新 tuple 是否能放同一 page
  -> use_hot_update = true
```

如果成立，进入 HOT 写入路径。

### 13.2 写 HOT flags

```text
旧 tuple:
  t_ctid = new tuple tid
  HEAP_HOT_UPDATED

新 tuple:
  HEAP_ONLY_TUPLE
  t_ctid = self
```

旧 tuple 仍由索引入口可达。

新 tuple 由 HOT chain 可达。

### 13.3 index scan 命中 root

```text
index AM returns TID(root)
  -> heapam_index_fetch_tuple()
  -> ReadBuffer(root block)
  -> heap_page_prune_opt()
  -> LockBuffer(shared)
  -> heap_hot_search_buffer()
```

### 13.4 搜索可见成员

```text
heap_hot_search_buffer()
  -> check root / redirect
  -> build HeapTupleData
  -> validate chain
  -> HeapTupleSatisfiesVisibility()
  -> if visible return tuple
  -> if hot updated follow t_ctid offset
```

### 13.5 返回 slot

如果找到可见 tuple：

```text
ExecStoreBufferHeapTuple()
```

slot 持有 buffer tuple。

如果没有找到：

```text
index entry 对当前 snapshot 没有可见 tuple。
```

调用者继续扫描。

## 14. 生命周期 / ownership / cleanup

HOT chain 的 owner 是 heap page。

它依赖同一 buffer 内的 line pointer 和 tuple header。

索引 entry 仍属于索引。

但它只指向 root line pointer。

HOT child 没有普通索引 entry。

这带来一个不变量：

```text
只要 chain 中还有可能可见的 heap-only child，root line pointer 或 redirect 必须保持从索引可达。
```

cleanup 生命周期：

```text
UPDATE 创建 HOT chain。
普通扫描沿 chain 找可见版本。
pruning 在 buffer cleanup lock 下移除 dead chain member 或设置 redirect。
VACUUM 最终清理 dead root 对应的索引 entry。
```

任何 cleanup 都不能破坏索引到可见版本的路径。

## 15. 正确性机制层次

第一层是 HOT 条件。

不改变普通索引依赖列。

第二层是同页约束。

chain member 都在一个 heap page。

第三层是 header flags。

前驱 `HEAP_HOT_UPDATED`。

child `HEAP_ONLY_TUPLE`。

第四层是 `t_ctid`。

前驱指向后继 offset。

第五层是链路校验。

后继 `xmin` 必须等于前驱 update xid。

第六层是 visibility routine。

每个 member 仍按 snapshot 判断。

第七层是 pruning redirect。

清理旧版本时仍保持 index reachability。

这些层合在一起，才保证：

```text
少建索引项，不等于牺牲可见性 correctness。
```

## 16. 错误路径 / 异常路径 / fallback

### 16.1 更新了索引列

如果 UPDATE 修改 HOT-blocking 索引列，不能 HOT。

fallback 是普通 update：

```text
新版本获得新的索引项。
旧版本按普通 update chain 处理。
```

### 16.2 同页空间不足

如果新 tuple 放不进同一 page，不能 HOT。

fallback 也是普通 update。

跨页 HOT 会破坏 page-local pruning 和 index scan 成本边界。

### 16.3 aborted HOT update

如果 HOT updater 回滚，前驱的 HOT link 不应被当作有效。

`HeapTupleHeaderIsHotUpdated()` 会考虑 `xmax` 和 `xmin` invalid 状态。

搜索时如果发现链断裂，停止。

### 16.4 redirect line pointer

index scan 遇到 root line pointer redirect，要跟到 redirect target。

这不是异常。

这是 HOT pruning 后的正常状态。

### 16.5 child slot 被 prune 或重用

如果 line pointer 不 normal，停止。

如果 child `xmin` 不匹配前驱 update xid，停止。

这避免误读无关 tuple。

## 17. 成本、资源与跨模块传播

HOT 降低的成本：

```text
少插入普通索引项。
减少索引 bloat。
减少后续 index vacuum 压力。
允许 page-local pruning。
提升更新非索引列的吞吐。
```

HOT 增加的成本：

```text
index scan 需要在 heap page 内沿 chain 搜索。
page 上可能有更长 chain。
pruning 逻辑更复杂。
line pointer redirect 增加状态组合。
```

HOT 的折中是：

```text
把一部分 update 成本从索引维护转移到 heap page 内链路搜索和 pruning。
```

跨模块传播：

```text
heapam.c:
  决定 HOT 是否适用并写 flags。

heapam_indexscan.c:
  沿 HOT chain 找可见 tuple。

pruneheap.c:
  维护 redirect 和 dead line pointer。

vacuumlazy.c:
  最终清理索引和 heap dead 状态。

README.HOT:
  解释设计约束和图示。

htup_details.h:
  定义 flags 和 accessor。
```

## 18. 观测与诊断入口

SQL 观察：

```sql
SELECT xmin, xmax, ctid, * FROM hot_demo WHERE id = 1;
```

pageinspect：

```sql
SELECT lp,
       lp_flags,
       t_xmin,
       t_xmax,
       t_ctid,
       t_infomask,
       t_infomask2
FROM heap_page_items(get_raw_page('hot_demo', 0))
ORDER BY lp;
```

源码断点：

```text
break heap_update
break heapam_index_fetch_tuple
break heap_hot_search_buffer
break heap_page_prune_opt
```

关键变量：

```text
use_hot_update
modified_attrs
hot_attrs
heapTuple->t_data->t_infomask2
prev_xmax
offnum
all_dead
*heap_continue
```

诊断 HOT 是否发生：

```text
看 update 是否修改索引列。
看新版本是否同页。
看 t_infomask2 是否有 HOT/heap-only flags。
看 index scan 是否进入 heap_hot_search_buffer()。
```

## 19. 常见误区

误区一：

```text
HOT 改变了 MVCC 可见性规则。
```

不对。

每个 member 仍按 snapshot 判断。

误区二：

```text
HOT child 不在索引里，所以 index scan 找不到。
```

不对。

index scan 从 root 进入 HOT chain。

误区三：

```text
HOT chain 可以跨 page。
```

不对。

同页是 HOT 的核心边界。

误区四：

```text
redirect line pointer 是损坏。
```

不对。

它是 HOT pruning 的正常状态。

误区五：

```text
只要不改索引列就一定 HOT。
```

不对。

新版本还必须能放在同一 page，并且满足其它实现条件。

## 20. 课堂实验

### 实验一：制造 HOT update

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

DROP TABLE IF EXISTS hot_demo;
CREATE TABLE hot_demo(id int primary key, payload text);

INSERT INTO hot_demo VALUES (1, 'v1');
UPDATE hot_demo SET payload = 'v2' WHERE id = 1;
UPDATE hot_demo SET payload = 'v3' WHERE id = 1;

SELECT xmin, xmax, ctid, * FROM hot_demo WHERE id = 1;

SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('hot_demo', 0))
ORDER BY lp;
```

解释目标：

```text
多个版本在同页。
旧版本指向后继。
后继可能是 heap-only tuple。
```

### 实验二：打断 HOT 条件

```sql
UPDATE hot_demo SET id = 2 WHERE id = 1;

SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('hot_demo', 0))
ORDER BY lp;
```

解释目标：

```text
修改 primary key 索引列后，不能继续用同一个索引入口代表新版本。
```

### 实验三：断点跟踪 index scan

断点：

```text
break heapam_index_fetch_tuple
break heap_hot_search_buffer
```

执行：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM hot_demo WHERE id = 2;
```

观察：

```text
index 返回 root TID。
heap_hot_search_buffer() 在同一 buffer 内遍历 chain。
HeapTupleSatisfiesVisibility() 判断每个 member。
```

### 实验四：pruning 后观察 redirect

```sql
VACUUM hot_demo;

SELECT lp, lp_flags, t_xmin, t_xmax, t_ctid, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('hot_demo', 0))
ORDER BY lp;
```

观察目标：

```text
line pointer 状态可能变化。
root 可能变成 redirect 或 dead。
这取决于 cleanup horizon 和页面状态。
```

## 21. 讨论题

1. HOT 为什么不能跨 page？

2. 为什么 heap-only tuple 没有普通索引项仍能被 index scan 找到？

3. 如果 pruning 直接删除 root line pointer，会破坏什么？

4. 为什么 HOT search 仍要调用普通 visibility routine？

5. 修改 indexed column 时为什么必须结束 HOT chain？

6. HOT 把成本从哪里转移到了哪里？

7. 为什么 MVCC snapshot 下 HOT chain 最多只有一个可见 member？

## 22. 本节小结

HOT 的核心不是新的可见性规则。

它是新的索引可达性结构。

当 UPDATE 不改变普通索引依赖列且新版本同页时：

```text
索引仍指向 root。
前驱标记 HEAP_HOT_UPDATED。
child 标记 HEAP_ONLY_TUPLE。
index scan 从 root 沿 t_ctid 找 snapshot 可见 member。
pruning 用 redirect 保持索引可达性。
```

HOT 的正确性靠同页约束、header flags、`t_ctid`、`xmin/xmax` 校验、普通 visibility routine 和 pruning redirect 共同维护。

到这里，03 目录从 snapshot 到 tuple header，再到 UPDATE/HOT 版本链，已经形成一条完整读写可见性主线。

后续课程可以继续进入 heap page pruning、VACUUM heap scan、index cleanup 和 freeze。
