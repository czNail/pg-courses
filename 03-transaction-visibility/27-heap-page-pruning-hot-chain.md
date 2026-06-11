# PostgreSQL heap page pruning 与 HOT chain 修剪

## 课程定位

前置知识：已经理解 `HEAPTUPLE_DEAD` 与 `HEAPTUPLE_RECENTLY_DEAD` 的区别，也已经知道 heap tuple 的 `t_ctid` 可以把旧版本链接到新版本。

本节唯一主问题：

```text
当 heap page 上出现可回收的旧版本时，PostgreSQL 为什么不能直接删除 tuple storage，而要通过 HOT chain、LP_REDIRECT、LP_DEAD 和 LP_UNUSED 分阶段修剪？
```

核心矛盾：页内空间希望尽快复用，索引却可能仍通过 root TID 找到这条逻辑行；如果 pruning 破坏 root line pointer 或 HOT chain 可达性，索引扫描会找不到仍然可见的新版本。

学完后应能判断：HOT update 什么时候减少索引写入，pruning 什么时候能缩短链，为什么 heap-only tuple 不能被索引直接指向，以及为什么 `LP_DEAD` 不是最终释放状态。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲的是 tuple-level vacuum visibility。

它回答：

```text
这个 tuple version 是否已经安全地不再需要？
```

本节进入 heap page 内部。

即使某个 tuple version 已经 `HEAPTUPLE_DEAD`，页内也不能随意删除。

原因是 heap page 同时服务三类访问。

第一类是 sequential scan。

它顺序读取 line pointer 和 tuple storage。

第二类是 index scan。

它从索引得到 heap TID，再回表检查 heap tuple。

第三类是 HOT chain traversal。

它从索引指向的 root line pointer 出发，沿着 `t_ctid` 找到同一页上的新版本。

本节只讲这三类访问之间的页内契约。

后续第 28 节再讲 VACUUM 怎样跨 heap 和 index 做批量清理。

第 30、31 节再讲 pruning / VACUUM 后为什么能设置 visibility map bit。

所以本节的主线是：

```text
UPDATE 形成 HOT chain
  -> 页上出现 dead chain prefix
  -> on-access pruning 或 VACUUM 拿 cleanup lock
  -> heap_page_prune_and_freeze() 计划 line pointer 改动
  -> root redirect / LP_DEAD / LP_UNUSED 保持索引可达性
  -> 页面空间和扫描成本下降
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
HOT update 让索引只指向 root line pointer；
新版本如果仍在同一 heap page 且没有破坏 HOT-safe 条件，就作为 heap-only tuple 挂在 root 后面；
pruning 在安全 horizon 之后删除 chain 前缀，把 root 改成 LP_REDIRECT、LP_DEAD 或 LP_UNUSED，同时保证任何索引 TID 仍能找到正确的可见版本或安全失败。
```

本节核心矛盾是：

```text
页内回收希望把 dead tuple storage 立刻变成可用空间
  vs
索引和 HOT chain 要求 root TID 的语义稳定
```

如果直接把旧 root 的 line pointer 改成 unused，旧索引 entry 仍可能指向这个 TID。

索引扫描会回表找不到 chain。

如果直接删除中间 dead tuple，但没有重连 chain，后续新版本可能不可达。

如果把 heap-only tuple 暴露给索引，HOT 的基本假设就失效。

因此 PostgreSQL 不做“删除 tuple”这么简单的动作。

它做的是页内状态转换：

```text
LP_NORMAL root
  -> LP_REDIRECT to first live heap-only tuple
  -> LP_DEAD when whole indexed chain dead
  -> LP_UNUSED only when不再需要索引回表保护
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/README.HOT` | 先读 HOT 的设计约束：root、heap-only tuple、HOT-safe、broken chain、redirect。 |
| 2 | `src/include/storage/itemid.h` | 对照 line pointer 状态：`LP_UNUSED`、`LP_NORMAL`、`LP_REDIRECT`、`LP_DEAD`。 |
| 3 | `src/include/access/htup_details.h` | 对照 `t_ctid`、`HEAP_HOT_UPDATED`、`HEAP_ONLY_TUPLE`。 |
| 4 | `src/backend/access/heap/heapam.c` | 阅读 `heap_update()` 如何决定 HOT update，并设置 `t_ctid`、HOT bit 和 `PageSetPrunable()`。 |
| 5 | `src/backend/access/heap/pruneheap.c` | 本节主文件：`heap_page_prune_opt()`、`heap_page_prune_and_freeze()`、`heap_prune_chain()`、`heap_page_prune_execute()`。 |
| 6 | `src/backend/access/heap/heapam_visibility.c` | 对照 pruning 使用的 vacuum visibility 分类。 |
| 7 | `src/backend/access/heap/vacuumlazy.c` | 看 VACUUM 何时调用 `lazy_scan_prune()`，以及何时把 `LP_DEAD` 记录到 `dead_items`。 |
| 8 | `src/include/access/heapam_xlog.h` | 对照 pruning / freeze WAL record 里如何描述 redirect、dead、unused。 |

本节阅读源码时不要把 HOT 当成“少建索引”的单点优化。

更准确地说：

```text
HOT 是一个索引 TID 稳定性协议。
```

少建索引是结果。

root 可达性和页内 chain 正确性才是约束。

## 4. 一个 runtime 现象先定锚

先做一张适合 HOT update 的表。

```sql
DROP TABLE IF EXISTS hot_prune_demo;
CREATE TABLE hot_prune_demo(
    id int primary key,
    payload text,
    note text
) WITH (fillfactor = 70);

INSERT INTO hot_prune_demo
SELECT g, repeat('x', 100), 'v0'
FROM generate_series(1, 2000) AS g;
```

只更新非索引列。

```sql
UPDATE hot_prune_demo SET note = 'v1' WHERE id <= 1000;
UPDATE hot_prune_demo SET note = 'v2' WHERE id <= 1000;
UPDATE hot_prune_demo SET note = 'v3' WHERE id <= 1000;
```

观察统计趋势：

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'hot_prune_demo';
```

如果页面有足够空间，`n_tup_hot_upd` 会增长。

这说明很多更新没有为新版本创建新的 index entry。

现在执行：

```sql
VACUUM (VERBOSE) hot_prune_demo;
```

现象通常是：

```text
heap 上 dead version 被 pruning / vacuum 清理
索引清理量可能低于非 HOT 更新场景
后续 scan 回表仍能通过 root TID 找到 live version
```

如果安装了 `pageinspect`，还可以更直观看到 line pointer。

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, lp_flags, lp_off, lp_len, t_ctid, t_infomask2
FROM heap_page_items(get_raw_page('hot_prune_demo', 0))
ORDER BY lp;
```

这里的重点不是 pageinspect 输出中每个数字固定。

重点是你会看到页内 item slot 的状态变化。

某些 line pointer 会从 normal tuple storage，变成 redirect、dead 或 unused。

这就是 pruning 的可观察入口。

## 5. HOT chain 的页内契约

普通 update 会产生新 heap tuple version。

如果更新列影响某个索引，必须为新版本插入索引 entry。

索引 entry 指向新版本的 heap TID。

HOT update 的特殊性是：

```text
新版本仍在同一 heap page；
被更新的列不破坏 HOT-safe 条件；
索引仍然只指向 chain root；
新版本标记为 HEAP_ONLY_TUPLE；
旧版本标记为 HEAP_HOT_UPDATED；
旧版本的 t_ctid 指向新版本。
```

这形成一个页内链：

```text
index entry
  -> root line pointer
     -> tuple v1
        t_ctid -> tuple v2
           t_ctid -> tuple v3
```

对于 index scan，root TID 不能消失。

对于 heap-only tuple，索引不能直接指向它。

这两个约束共同决定 pruning 的形状。

如果 root dead 了，但后面还有 live heap-only version，不能把 root 变成 unused。

必须把 root 变成 redirect。

如果整条 chain 都 dead，root 可以变成 `LP_DEAD`。

但它仍未必能直接 unused。

因为索引中可能还有 entry 指向 root。

只有索引清理完成，或没有索引引用风险，才能把它变成 `LP_UNUSED`。

## 6. line pointer 状态不是 tuple 状态

`itemid.h` 里有四种 line pointer 状态。

| 状态 | 语义 |
| --- | --- |
| `LP_UNUSED` | slot 空闲，可复用，通常没有 tuple storage。 |
| `LP_NORMAL` | slot 指向普通 tuple storage。 |
| `LP_REDIRECT` | HOT root redirect，slot 不存 tuple，指向同页另一个 offset。 |
| `LP_DEAD` | 语义上 dead，可能仍被索引 TID 指向，不能当成完全空闲。 |

这些状态回答的是页内 slot 的问题。

上一节的 `HEAPTUPLE_DEAD` 回答的是 tuple version 是否安全可清的问题。

两者组合才是 pruning 语义。

例如：

```text
HEAPTUPLE_DEAD + heap-only tuple
  -> 可以直接 LP_UNUSED

HEAPTUPLE_DEAD + indexed root but chain has live successor
  -> root LP_REDIRECT

HEAPTUPLE_DEAD + indexed root and whole chain dead
  -> root LP_DEAD

LP_DEAD + index cleanup done
  -> LP_UNUSED
```

如果把这几层混在一起，就会误读 VACUUM。

`VACUUM` 可能报告 dead item identifiers removed。

这不等于所有 dead tuple 都是在第一遍 heap scan 时释放。

它可能是：

```text
第一遍 heap scan:
  tuple-level dead
  -> LP_DEAD
  -> 记录 TID 到 dead_items

index vacuum:
  删除索引中的 TID

第二遍 heap pass:
  LP_DEAD
  -> LP_UNUSED
```

## 7. 主流程源码 walkthrough

本节主流程选 on-access pruning。

这是普通查询或修改路径中最容易触发的页内清理。

```text
heap scan / heap access
  -> heap_page_prune_opt()
     -> PageGetPruneXid()
     -> GlobalVisTestFor()
     -> GlobalVisTestIsRemovableXid()
     -> ConditionalLockBufferForCleanup()
     -> visibilitymap_pin()
     -> heap_page_prune_and_freeze()
        -> prune_freeze_setup()
        -> prune_freeze_plan()
        -> heap_prune_chain()
        -> heap_page_prune_execute()
        -> log_heap_prune_and_freeze()
```

第一步，`heap_page_prune_opt()` 是 opportunistic。

它不会每次访问页面都做重清理。

它先看 `PageGetPruneXid(page)`。

如果没有 prune xid，说明页上没有明显值得 pruning 的候选。

函数快速返回。

第二步，它用 `GlobalVisTestFor(relation)` 获取 relation-aware visibility test。

再用 `GlobalVisTestIsRemovableXid()` 判断 `pd_prune_xid` 是否可能已经可移除。

如果 horizon 还没越过，没必要拿锁。

第三步，它看页面 free space。

只有 page full 或 free space 低于 fillfactor 目标时，才尝试 pruning。

这说明 pruning 是受成本控制的 housekeeping。

它不是每次访问都扫全页。

第四步，它用 `ConditionalLockBufferForCleanup(buffer)`。

拿不到 cleanup lock 就返回。

on-access pruning 不愿意阻塞前台路径太久。

这和 VACUUM 的行为不同。

第五步，拿到 cleanup lock 后，它 pin visibility map page。

这是因为 pruning 后页面可能变成 all-visible。

设置 VM bit 需要提前 pin 对应 VM page，避免在持 heap lock 时做 I/O。

第六步，进入 `heap_page_prune_and_freeze()`。

这个函数先 `prune_freeze_plan()`。

计划阶段会扫描 line pointer，计算 tuple visibility，找 HOT chain root 和 heap-only tuple。

然后才在 critical section 中执行页面修改。

第七步，`heap_prune_chain()` 处理单条 HOT chain。

它从 root 出发，沿着 `t_ctid` 走同页 offset。

它用已经计算好的 `HTSV_Result` 判断每个 chain item。

如果 chain 前缀都是 dead，就可以剪掉前缀。

如果剪掉后还有 live successor，root 要 redirect 到第一个非 dead successor。

如果整条 chain dead，root 变为 dead 或 unused。

第八步，`heap_page_prune_execute()` 应用计划。

它处理 redirect、nowdead、nowunused 三类变化。

这一步才真正改变 line pointer。

如果需要 WAL，就通过 `log_heap_prune_and_freeze()` 记录。

核心不变量是：

```text
计划阶段可以做可见性查询；
执行阶段只做已经计划好的页内原子修改。
```

## 8. 生命周期 / ownership / cleanup

HOT chain 的生命周期从 update 开始。

阶段一：root tuple 插入。

```text
index entry -> root TID
root line pointer = LP_NORMAL
root tuple t_ctid points to itself
```

阶段二：HOT update 成功。

`heap_update()` 检查 HOT-safe 条件。

如果新版本能放在同页，且索引列约束允许，旧 tuple 设置 HOT updated。

新 tuple 设置 heap-only。

旧 tuple 的 `t_ctid` 指向新 tuple。

索引不为新 tuple 新增 entry。

阶段三：多次 HOT update。

chain 变长。

前面版本逐渐对新 snapshot 不可见。

旧版本可能先是 recently dead。

再随着 horizon 前移变成 dead。

阶段四：页面被访问或 VACUUM 扫描。

`pd_prune_xid` 提示页面值得 pruning。

on-access pruning 尝试 cleanup lock。

VACUUM 在 heap scan 中更系统地调用 pruning。

阶段五：剪掉 dead prefix。

如果 root dead 但后面有 live tuple：

```text
root LP_NORMAL
  -> LP_REDIRECT to live successor
dead heap-only tuple
  -> LP_UNUSED
```

索引仍指向 root offset。

root redirect 让索引 scan 能继续到 live successor。

阶段六：整条 chain dead。

root 可能变为 `LP_DEAD`。

如果可以安全立刻 unused，才变 `LP_UNUSED`。

阶段七：VACUUM index cleanup 后回收 `LP_DEAD`。

当索引中的 TID 被删除，第二遍 heap pass 可以把 `LP_DEAD` 改成 `LP_UNUSED`。

这样 slot 才真正可复用。

这个生命周期说明：

```text
pruning 的 owner 是 heap page；
索引 entry 是外部可达性约束；
horizon 是清理许可；
buffer cleanup lock 是修改保护；
WAL 是 crash recovery 保护。
```

## 9. 正确性机制层次

第一层是 HOT-safe 判断。

只有不破坏索引语义的 update 才能 HOT。

如果被更新列影响索引表达式或谓词，不能只依赖旧 index entry。

第二层是同页约束。

HOT chain 必须在同一 heap page 内。

跨页 HOT 会破坏索引 scan 的局部 traversal 假设，也会让 pruning 成本和锁范围复杂化。

第三层是 root 稳定性。

索引 entry 指向 root TID。

root line pointer 不能随意 unused。

`LP_REDIRECT` 正是为了保留 root offset。

第四层是 visibility horizon。

只有 dead prefix 才能剪。

recently dead 不能因为“看起来旧”就剪掉。

第五层是 cleanup lock。

pruning 修改 page layout。

必须拿 buffer cleanup lock 或 VACUUM 对应锁。

普通 share lock 不够。

第六层是 WAL。

redirect、dead、unused 和 freeze 的组合要进入 WAL。

redo 必须能重放同样的 page state transition。

第七层是 VM consistency。

pruning 后页面可能 all-visible。

如果设置 `PD_ALL_VISIBLE` 和 VM bit，必须保证 page hint 和 VM 一致性。

这部分第 30 节会展开。

## 10. 错误路径 / 异常路径 / fallback

### recovery 中不做 on-access pruning

`heap_page_prune_opt()` 在 recovery 中直接返回。

因为恢复过程中不能像 primary 那样写新的 pruning WAL。

primary 之后会产生相关清理记录。

### 没有 prune xid

`PageGetPruneXid(page)` invalid 时，函数快速返回。

这不是证明页面没有任何可优化空间。

它只说明没有足够强的页内 hint 触发 pruning。

### horizon 还不够老

即使页面有 `pd_prune_xid`，如果 `GlobalVisTestIsRemovableXid()` 认为不可移除，pruning 返回。

这避免把 recently dead 当成 dead。

### cleanup lock 拿不到

on-access pruning 使用 conditional lock。

拿不到就放弃。

这避免前台查询为了清理页面长时间等待。

VACUUM 后续还会再尝试。

### heap-only tuple 不在任何 chain 中

`pruneheap.c` 对某些不一致状态会报错。

例如 dead heap-only tuple 没有从任何 HOT chain 链接到。

这是 corruption 级别的问题。

因为 heap-only tuple 按定义不能被索引直接找到。

### VM 与页面 hint 不一致

`heap_page_prune_and_freeze()` 开始时会检查 VM bit 与 `PD_ALL_VISIBLE`。

如果 VM set 但 page hint clear，会修复。

如果 page hint 错误，也会按路径清理。

这说明 pruning 不只是删除 tuple。

它还修复部分页级可见性 hint。

### mark unused 的限制

没有索引的表，VACUUM 可以更直接地把 dead item 变 unused。

有索引时，很多 root item 必须先变 `LP_DEAD`。

否则索引中的 TID 可能失去安全回表目标。

## 11. 成本、资源与跨模块传播

HOT 的收益是减少索引写入。

但 HOT chain 也带来成本。

链越长，index scan 回表后可能要跟更多 `t_ctid`。

页面上的 dead versions 越多，free space 越碎。

pruning 的作用是把这两个成本拉回来。

成本可以分成几类。

| 成本来源 | 表现 |
| --- | --- |
| HOT chain traversal | index scan 回表后需要跟链，直到找到可见版本或链尾。 |
| page fragmentation | dead tuple storage 占用页内空间，影响同页 HOT update 成功率。 |
| cleanup lock | pruning 需要强页锁，可能被 pin 或并发访问阻挡。 |
| WAL | 实际修改 redirect、dead、unused、VM bit 时需要 WAL。 |
| index cleanup | 有索引时，`LP_DEAD` 到 `LP_UNUSED` 需要先清索引 TID。 |
| stats | pruning 需要更新 heap dead tuple 统计，避免 VACUUM 触发判断偏离太大。 |

跨模块传播如下：

```text
heap_update()
  -> 决定 HOT / non-HOT
  -> 设置 t_ctid 和 HOT bit
  -> PageSetPrunable()

heap_page_prune_opt()
  -> 前台访问时机会性清理

lazy_scan_prune()
  -> VACUUM 系统性清理

heap_page_prune_and_freeze()
  -> 页内计划和执行

vacuumlazy.c
  -> 收集 LP_DEAD，协调 index cleanup
```

这说明 HOT 不是局部优化。

它把 executor 访问、heap update、index AM、VACUUM、WAL 和 VM 全部连接起来。

## 12. 观测与诊断入口

先看表级 HOT 更新比例：

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 2) AS hot_pct,
       n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'hot_prune_demo';
```

再看 VACUUM verbose：

```sql
VACUUM (VERBOSE) hot_prune_demo;
```

关注：

```text
dead item identifiers
index scans
pages scanned
pages vacuumed
tuples removed
```

再看 progress：

```sql
SELECT pid, phase, heap_blks_scanned, heap_blks_vacuumed,
       index_vacuum_count, num_dead_item_ids
FROM pg_stat_progress_vacuum;
```

如果有 `pageinspect`：

```sql
SELECT lp, lp_flags, lp_off, lp_len, t_ctid, t_infomask2
FROM heap_page_items(get_raw_page('hot_prune_demo', 0))
ORDER BY lp;
```

`lp_flags` 的数值要结合 `itemid.h` 解释。

不要只看 tuple header。

要同时看 line pointer。

源码断点建议：

```text
heap_page_prune_opt
heap_page_prune_and_freeze
prune_freeze_plan
heap_prune_chain
heap_page_prune_execute
lazy_scan_prune
```

断点里关注：

```text
rootoffnum
chainitems
ndeadchain
nredirected
ndead
nunused
latest_xid_removed
new_prune_xid
```

## 13. 常见误区

误区一：HOT 就是“不更新索引”。

正确理解：HOT 是保持 root TID 稳定并把新版本限制在同页的协议。

不更新索引只是协议允许后的结果。

误区二：dead tuple 可以直接从页面删除。

正确理解：root line pointer 可能仍被索引指向。

必须通过 redirect、dead、unused 分阶段处理。

误区三：`LP_REDIRECT` 是坏状态。

正确理解：它是 HOT pruning 后保护索引可达性的正常状态。

误区四：heap-only tuple 可以被索引直接找到。

正确理解：heap-only tuple 没有自己的 index entry。

它只能通过 HOT chain 从 root 找到。

误区五：on-access pruning 一定会执行。

正确理解：它是 opportunistic。

没有 prune hint、horizon 不够、free space 不紧张、cleanup lock 拿不到都会返回。

误区六：VACUUM 第一遍 heap scan 就释放所有页面空间。

正确理解：有索引时，第一遍可能只把 item 标记为 `LP_DEAD` 并记录 TID。

第二遍 heap pass 才能把它变 `LP_UNUSED`。

误区七：HOT chain 越长越好。

正确理解：HOT 减少索引写入，但 chain traversal 和页内碎片会增加。

pruning 是 HOT 成本模型的一部分。

## 14. 课堂实验

### 实验一：HOT update 与 pruning

```sql
DROP TABLE IF EXISTS hot_prune_demo;
CREATE TABLE hot_prune_demo(
    id int primary key,
    payload text,
    note text
) WITH (fillfactor = 70);

INSERT INTO hot_prune_demo
SELECT g, repeat('x', 100), 'v0'
FROM generate_series(1, 2000) AS g;

UPDATE hot_prune_demo SET note = 'v1' WHERE id <= 1000;
UPDATE hot_prune_demo SET note = 'v2' WHERE id <= 1000;
UPDATE hot_prune_demo SET note = 'v3' WHERE id <= 1000;

SELECT relname, n_tup_upd, n_tup_hot_upd, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'hot_prune_demo';
```

执行：

```sql
VACUUM (VERBOSE) hot_prune_demo;
```

解释：

```text
非索引列更新更容易 HOT。
HOT chain 形成后，pruning 可以剪掉 dead prefix。
索引 entry 仍指向 root。
```

### 实验二：破坏 HOT-safe 条件

```sql
DROP TABLE IF EXISTS hot_break_demo;
CREATE TABLE hot_break_demo(
    id int primary key,
    k int,
    payload text
) WITH (fillfactor = 70);

CREATE INDEX hot_break_demo_k_idx ON hot_break_demo(k);

INSERT INTO hot_break_demo
SELECT g, g, repeat('x', 100)
FROM generate_series(1, 2000) AS g;

UPDATE hot_break_demo SET k = k + 1 WHERE id <= 1000;

SELECT relname, n_tup_upd, n_tup_hot_upd, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'hot_break_demo';
```

解释：

```text
更新被索引列时，旧 index entry 不能代表新版本。
新版本需要新的 index entry。
HOT 比例下降。
```

### 实验三：pageinspect 看 line pointer

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, lp_flags, lp_off, lp_len, t_ctid, t_infomask2
FROM heap_page_items(get_raw_page('hot_prune_demo', 0))
ORDER BY lp;
```

在多次 update、VACUUM 前后比较。

把 `lp_flags` 映射到：

```text
0 = LP_UNUSED
1 = LP_NORMAL
2 = LP_REDIRECT
3 = LP_DEAD
```

注意：具体页和 offset 会随填充、版本、并发和 vacuum 时机变化。

### 实验四：源码断点

```gdb
break heap_page_prune_opt
break heap_page_prune_and_freeze
break heap_prune_chain
break heap_page_prune_execute
```

执行触发清理的 SQL：

```sql
VACUUM hot_prune_demo;
```

观察：

```gdb
print rootoffnum
print ndeadchain
print prstate->nredirected
print prstate->ndead
print prstate->nunused
```

把断点观察和 pageinspect 输出对应起来。

## 15. 讨论题

1. 为什么 HOT chain 必须限制在同一个 heap page 内？

2. 为什么 root line pointer 不能因为 root tuple dead 就直接 `LP_UNUSED`？

3. `LP_REDIRECT` 和 `t_ctid` 分别解决什么问题？

4. 为什么 on-access pruning 使用 conditional cleanup lock？

5. 如果某张表 HOT update 比例高但 bloat 仍增长，可能有哪些原因？

6. 为什么没有索引的 heap page 可以更激进地把 dead item 变 unused？

7. 如果 pageinspect 看到 `LP_DEAD`，为什么还不能断言页面空间已经完全可复用？

## 16. 本节小结

本节把上一节的 tuple-level dead 推进到 page-level cleanup。

HOT 的关键不是简单减少索引更新。

它是一个 root TID 稳定协议。

索引指向 root。

heap-only tuple 只能通过同页 HOT chain 到达。

pruning 的任务是在不破坏这个协议的前提下剪掉 dead prefix。

因此 PostgreSQL 使用：

```text
LP_REDIRECT
LP_DEAD
LP_UNUSED
```

来表达不同清理阶段。

`HEAPTUPLE_DEAD` 说明 tuple 可以清。

`LP_DEAD` 说明 line pointer 语义上死了但可能仍要保护索引回表。

`LP_UNUSED` 才说明 slot 可复用。

本节可迁移规律是：

```text
局部空间回收必须服从外部可达性协议；
数据结构里看似多余的中间状态，通常是在保护一个跨模块不变量。
```

下一节继续把范围扩大到 relation 级 VACUUM：heap scan、index cleanup 和第二次 heap cleanup 为什么必须被拆成阶段。
