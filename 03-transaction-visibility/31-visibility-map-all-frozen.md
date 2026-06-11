# PostgreSQL visibility map 的 all-frozen 位

## 课程定位

前置知识：已经理解 all-visible bit 是 page-level 可见性缓存，也已经理解 freeze 的目标是移除旧 XID / MultiXact 依赖，推进 `relfrozenxid` / `relminmxid`。

本节唯一主问题：

```text
既然 all-visible 已经说明整页对所有事务可见，PostgreSQL 为什么还需要单独的 all-frozen 位？
```

核心矛盾：VACUUM 既想跳过已经全可见的页面来减少 I/O，又必须保证 anti-wraparound 不会漏掉仍含旧 XID 或 MultiXact 的 tuple；all-visible 只能证明可见性，不能证明这个页面未来不再需要 freeze。

学完后应能判断：`VISIBILITYMAP_ALL_FROZEN` 为什么必须依赖 all-visible，什么时候设置，什么时候只清 all-frozen 而保留 all-visible，以及 aggressive VACUUM 为什么可以信任 all-frozen 页面。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 30 节讲 all-visible。

它回答：

```text
这个 heap page 上所有 tuple 是否对所有事务可见？
```

本节讲 all-frozen。

它回答另一个问题：

```text
这个 heap page 上是否不再含有需要未来 freeze 的事务身份？
```

两个问题相邻，但不相同。

一个页面可以 all-visible。

它上面的 tuple 都对所有事务可见。

但 tuple header 里仍可能有普通 XID 或 MultiXact。

这些事务身份现在已经能解释。

但随着 XID / MultiXact 空间推进，未来仍可能需要 freeze。

所以 all-visible 不能让 anti-wraparound VACUUM 永远跳过这个页面。

all-frozen 是更强承诺。

```text
all-frozen
  -> all-visible
  -> index-only scan 可跳过 heap visibility check

all-visible
  -> 不一定 all-frozen
  -> 未来 VACUUM 可能还要回来 freeze
```

本节主线是：

```text
VACUUM / pruning 证明页面全可见且所有 tuple 已冻结
  -> 同时设置 all-visible 和 all-frozen
  -> 普通 VACUUM 与 aggressive VACUUM 可利用 all-frozen 跳过页面
  -> 后续写入或锁语义变化清除对应 bit
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
heap_page_prune_and_freeze() 在 page-level pruning 和 tuple-level freeze 后，如果能证明页面既 all-visible 又没有任何需要未来 freeze 的 XID/MXID，就用 visibilitymap_set() 同时设置 VISIBILITYMAP_ALL_VISIBLE 和 VISIBILITYMAP_ALL_FROZEN；
后续 VACUUM 可以安全跳过 all-frozen 页面，因为它们不会阻碍 relfrozenxid / relminmxid 推进。
```

本节核心矛盾是：

```text
VACUUM 需要尽量少扫大表
  vs
anti-wraparound 要求不能漏掉任何老 XID / MultiXact
```

all-frozen 的价值就在这里。

它把“这个 page 已经不需要 freeze”缓存下来。

这比 all-visible 强。

也比 `relfrozenxid` 更局部。

relation-level `relfrozenxid` 是整张表的下界。

page-level all-frozen 是单个 block 的跳过许可。

两者配合，VACUUM 才能在大表上既安全又可控。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/access/visibilitymapdefs.h` | 定义 `VISIBILITYMAP_ALL_FROZEN`，并说明 VM 每个 heap block 两个 bit。 |
| 2 | `src/include/access/visibilitymap.h` | `VM_ALL_FROZEN()`、`visibilitymap_get_status()`、`visibilitymap_set()`、`visibilitymap_clear()`。 |
| 3 | `src/backend/access/heap/visibilitymap.c` | `visibilitymap_set()` 断言不能只 set all-frozen；`visibilitymap_clear()` 断言不能只清 all-visible 留 all-frozen。 |
| 4 | `src/backend/access/heap/pruneheap.c` | `heap_page_prune_and_freeze()` 如何设置 `set_all_frozen`、`new_vmbits` 和 freeze plan。 |
| 5 | `src/backend/access/heap/vacuumlazy.c` | VACUUM 如何跳过 all-frozen page、eager freeze all-visible but not all-frozen page、统计 `new_all_frozen_pages`。 |
| 6 | `src/backend/access/heap/heapam.c` | 写入和 tuple lock 路径如何清 VM bit，某些路径只清 all-frozen。 |
| 7 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()` 产生 freeze cutoffs，决定 aggressive VACUUM。 |
| 8 | `src/include/commands/vacuum.h` | freeze 参数、cutoffs、progress 模式。 |

本节不要把 all-frozen 看成 all-visible 的别名。

读源码时要盯住：

```text
set_all_visible
set_all_frozen
new_vmbits
FreezeLimit
MultiXactCutoff
skippedallvis
aggressive
```

## 4. 一个 runtime 现象先定锚

准备表并执行 freeze。

```sql
DROP TABLE IF EXISTS vm_frozen_demo;
CREATE TABLE vm_frozen_demo(id int primary key, payload text);

INSERT INTO vm_frozen_demo
SELECT g, repeat('x', 80)
FROM generate_series(1, 50000) AS g;

VACUUM (FREEZE, VERBOSE) vm_frozen_demo;
```

如果安装 `pg_visibility`：

```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT count(*) FILTER (WHERE all_visible) AS all_visible_pages,
       count(*) FILTER (WHERE all_frozen) AS all_frozen_pages,
       count(*) AS total_pages
FROM pg_visibility_map('vm_frozen_demo'::regclass);
```

常见现象是 all-visible pages 和 all-frozen pages 都增加。

现在做一个普通 UPDATE。

```sql
UPDATE vm_frozen_demo
SET payload = payload || 'y'
WHERE id = 1;
```

再看：

```sql
SELECT blkno, all_visible, all_frozen
FROM pg_visibility_map('vm_frozen_demo'::regclass)
ORDER BY blkno
LIMIT 20;
```

被修改的 page 对应 bit 会失效。

再次 VACUUM 后，bit 可能重新设置。

这个实验说明：

```text
all-frozen 是页面级 freeze 证明。
写入会破坏这个证明。
VACUUM 才能重新建立它。
```

如果只看 SELECT 行数，你看不到 all-frozen 的意义。

要看 VM bit、VACUUM verbose 和 relfrozenxid。

## 5. all-frozen 是更强承诺

`visibilitymapdefs.h` 里两个 bit 是：

```text
VISIBILITYMAP_ALL_VISIBLE = 0x01
VISIBILITYMAP_ALL_FROZEN  = 0x02
```

源码强制：

```text
不能只设置 all-frozen
```

`visibilitymap_set()` 里有断言：

```text
flags != VISIBILITYMAP_ALL_FROZEN
```

这说明 set all-frozen 时必须同时 set all-visible。

原因很直接。

如果页面上所有 tuple 都冻结了，那么它们必然已经可被所有正常事务解释为稳定可见或无须进一步可见性判断。

所以 all-frozen implies all-visible。

但反过来不成立。

一个页面上所有 tuple 可能都已经 committed 并对所有事务可见。

但 tuple header 里仍有正常 XID。

这些 XID 还没有到必须 freeze 的程度。

这个 page 可以 all-visible。

但不能 all-frozen。

因此两个 bit 分开。

执行器只需要 all-visible。

anti-wraparound VACUUM 需要 all-frozen。

## 6. 设置 all-frozen 的条件

设置 all-frozen 需要先满足 all-visible。

然后还要满足：

```text
页面上没有需要未来 freeze 的 XID；
页面上没有需要未来处理的 MultiXact；
当前 pruning/freezing 已应用所有必要 freeze plan；
没有 LP_DEAD item 阻止 all-visible；
页面上的 newest live xid 不再被任何 snapshot 认为 running。
```

`pruneheap.c` 里初始：

```text
set_all_frozen = attempt_freeze && attempt_set_vm
```

这只是候选。

扫描页面过程中，只要遇到不能证明冻结完成的 tuple，就会清掉这个候选。

如果 `heap_prepare_freeze_tuple()` 发现需要 freeze，会产生 freeze plan。

如果 `heap_page_will_freeze()` 决定执行，freeze plan 应用后页面可能恢复 all-frozen 候选。

如果决定不 freeze，页面不能 all-frozen。

所以 all-frozen 是页面扫描、tuple freeze plan 和 VM 设置三者合成的结果。

它不是单个字段判断。

## 7. 主流程源码 walkthrough

主流程从 VACUUM 扫描页面开始。

```text
lazy_scan_heap()
  -> lazy_scan_prune()
     -> heap_page_prune_and_freeze()
        -> prune_freeze_setup()
        -> old_vmbits = visibilitymap_get_status()
        -> fast path if already all-frozen
        -> prune_freeze_plan()
           -> heap_prepare_freeze_tuple()
        -> heap_page_will_freeze()
        -> adjust set_all_visible / set_all_frozen
        -> heap_page_will_set_vm()
        -> heap_freeze_prepared_tuples()
        -> visibilitymap_set(... ALL_VISIBLE | ALL_FROZEN ...)
        -> log_heap_prune_and_freeze()
```

第一步，`prune_freeze_setup()` 读取旧 VM bits。

如果旧 bits 已经 all-frozen，并且允许 fast path，很多页面检查可以跳过。

这是 all-frozen 的直接价值。

第二步，如果不能 fast path，`prune_freeze_plan()` 扫描 tuple。

它会计算：

```text
live_tuples
recently_dead_tuples
lpdead_items
newest_live_xid
freeze plan
```

第三步，`heap_prepare_freeze_tuple()` 判断 tuple 中哪些 XID / MXID 需要 freeze。

如果需要，会把动作写入 `HeapTupleFreeze`。

第四步，`heap_page_will_freeze()` 决定是否执行。

如果页面上存在早于 `FreezeLimit` 或 `MultiXactCutoff` 的身份，并且必须推进 relation horizon，就不能跳过。

如果不是必须，但当前已经要因为 pruning 写 WAL，也可能 opportunistic freeze。

第五步，执行修改。

如果 `do_freeze` 为 true：

```text
heap_freeze_prepared_tuples()
```

应用 tuple header 修改。

第六步，如果页面同时满足 all-visible 和 all-frozen：

```text
new_vmbits = VISIBILITYMAP_ALL_VISIBLE | VISIBILITYMAP_ALL_FROZEN
visibilitymap_set()
```

第七步，`PruneFreezeResult` 返回：

```text
newly_all_visible
newly_all_frozen
newly_all_visible_frozen
nfrozen
```

`vacuumlazy.c` 用这些字段更新 verbose 和 stats。

## 8. VACUUM 如何使用 all-frozen

VACUUM 使用 VM bit 做 page skipping。

all-visible page 可以跳过某些工作。

但如果它不是 all-frozen，普通 VACUUM 可能会 eager scan 一部分页面来提前 freeze。

aggressive VACUUM 更关心 freeze 进展。

它不能跳过可能含有旧 XID 的 all-visible but not all-frozen pages。

all-frozen page 则不同。

它已经证明页面没有需要 freeze 的 XID/MXID。

因此即使 anti-wraparound 目标很强，也可以跳过。

这就是 all-frozen 的核心收益：

```text
让 aggressive VACUUM 不必反复读取已完成 freeze 的冷页面。
```

`vacuumlazy.c` 还维护 eager freeze 相关状态。

普通 VACUUM 会有选择地扫描 all-visible but not all-frozen pages。

成功把页面设为 all-frozen 后，计入 eager freeze success。

如果在某个区域失败太多，会暂停 eager scanning。

这说明 all-frozen 不是只服务极端 wraparound。

它也服务长期降低未来 VACUUM 成本。

## 9. 清除 all-frozen 的路径

all-frozen 失效不总是和 all-visible 一起失效。

普通 INSERT / UPDATE / DELETE 修改页面内容，通常清：

```text
VISIBILITYMAP_VALID_BITS
```

也就是 all-visible 和 all-frozen 都清。

因为页面可见性承诺本身被破坏。

但某些路径可能只清 all-frozen。

例如对 tuple 加锁或 MultiXact 相关变化可能让页面仍然对所有事务可见，但引入了新的事务身份或 MultiXact 状态。

这时 all-visible 仍可能成立。

但 all-frozen 不再能保证。

源码中可以看到 `visibilitymap_clear(... VISIBILITYMAP_ALL_FROZEN)` 的调用。

这表达了一个重要层次：

```text
可见性承诺可以保留；
无需 future freeze 的承诺必须撤销。
```

因此 all-frozen 不是 all-visible 的计数增强。

它是另一类长期维护承诺。

## 10. 生命周期 / ownership / cleanup

一个页面的 all-frozen 生命周期如下。

阶段一：普通页面。

VM bits 可能 clear。

页面上有正常 XID / MXID。

阶段二：页面 all-visible。

VACUUM 证明所有 tuple 对所有事务可见。

设置 all-visible。

但 tuple 仍可能含有普通 XID。

阶段三：freeze 发生。

VACUUM 生成并应用 freeze plan。

旧 XID / MXID 依赖被消除。

阶段四：设置 all-frozen。

VM fork 中同时设置 all-visible 和 all-frozen。

阶段五：后续 VACUUM 跳过。

普通 VACUUM 和 aggressive VACUUM 都能利用它减少扫描。

阶段六：写入或锁语义变化。

如果页面内容可见性被破坏，清所有 valid bits。

如果只是引入 future-freeze 相关身份，可能只清 all-frozen。

阶段七：下一次 VACUUM 重新证明。

页面再次满足条件后，all-frozen 可以重新设置。

ownership 分层：

```text
tuple freeze:
  heap page 内容修改

all-frozen bit:
  VM fork 中的 page-level proof cache

relfrozenxid:
  pg_class 中 relation-level proof

pg_xact / multixact cleanup:
  依赖许多 relation-level proof 推进
```

## 11. 正确性机制层次

第一层是 implication。

```text
all-frozen -> all-visible
```

源码用断言保证不能只 set all-frozen。

第二层是安全跳过。

VACUUM 只能在 all-frozen 证明足够强时跳过 anti-wraparound 必需扫描。

第三层是 relation horizon 推进。

如果 VACUUM 跳过了 all-visible but not all-frozen 页面，就不能无条件推进 `relfrozenxid`。

第四层是 partial invalidation。

某些操作可以只清 all-frozen。

这保留 index-only scan 收益，同时撤销 freeze 证明。

第五层是 WAL。

设置 all-frozen 发生在持锁和 critical section 中，并随 heap page 修改记录 WAL。

第六层是 VM / heap consistency。

VM all-frozen set 时，VM all-visible 也必须 set。

如果 page-level hint 和 VM 不一致，pruning 会修复。

第七层是 conservative fallback。

清掉 all-frozen 只是损失优化。

错误保留 all-frozen 可能导致 VACUUM 漏扫页面，最终影响 wraparound 安全。

因此失效必须保守。

## 12. 错误路径 / 异常路径 / fallback

### all-visible but not all-frozen backlog

大表可能有大量 all-visible but not all-frozen pages。

普通查询能受益于 index-only scan。

但未来 anti-wraparound VACUUM 仍要处理这些页面。

VACUUM 的 eager freeze 机制就是为了逐步降低这个 backlog。

### DISABLE_PAGE_SKIPPING

如果指定 `VACUUM (DISABLE_PAGE_SKIPPING)`，VACUUM 不能信任 VM skip。

它会更全面扫描页面。

这通常用于怀疑 VM 损坏或需要强制检查。

### aggressive VACUUM

aggressive VACUUM 必须推进 freeze horizon。

它不会像普通 VACUUM 那样轻易跳过 all-visible but not all-frozen pages。

但 all-frozen pages 仍可安全跳过。

### tuple lock 清 all-frozen

某些 row locking 路径可能只清 all-frozen。

这解释了为什么一个 page 仍能 all-visible，却不再 all-frozen。

### VM corruption 修复

如果 VM bits 与 heap page hint 不一致，`pruneheap.c` 中的修复逻辑会清 VM bit 或修 page hint。

all-frozen 错误比 all-visible 更隐蔽。

因为它主要影响未来 VACUUM，而不是当前 SELECT 结果。

### failsafe

failsafe 下 VACUUM 更关注 freeze 必要工作。

如果 all-frozen 证明缺失，系统可能不得不扫描更多页面。

如果 all-frozen 证明可靠，可以减少 emergency 模式下的负担。

## 13. 成本、资源与跨模块传播

all-frozen 的主要收益是未来 VACUUM 成本下降。

它不直接让普通 SELECT 更快。

普通 SELECT 主要消费 all-visible。

all-frozen 主要让 VACUUM 可以证明：

```text
这个页面不会阻碍 relfrozenxid / relminmxid 推进。
```

成本模型：

| 维度 | 说明 |
| --- | --- |
| 冷数据表 | all-frozen 比例高，anti-wraparound VACUUM 成本低。 |
| 高频更新表 | all-frozen bit 频繁失效，收益低。 |
| row lock / MultiXact | 可能只破坏 all-frozen，留下 all-visible。 |
| eager freeze | 普通 VACUUM 额外扫描部分页面，换取未来跳过。 |
| WAL / FPI | freeze 和 VM set 可能产生写放大。 |
| VM 可靠性 | false positive 会威胁 freeze 安全，必须避免。 |

跨模块传播：

```text
vacuum_get_cutoffs()
  -> 决定哪些 XID / MXID 需要 freeze

heap_page_prune_and_freeze()
  -> 应用 tuple freeze，并决定 all-frozen

visibilitymap_set()
  -> 记录 page-level freeze proof

heap write / lock path
  -> 清 all-frozen 或清全部 VM bit

vacuumlazy.c
  -> 使用 all-frozen skip 和 eager freeze 策略

pg_class
  -> 维护 relallfrozen 与 relfrozenxid 相关统计
```

## 14. 观测与诊断入口

如果有 `pg_visibility`：

```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT count(*) FILTER (WHERE all_visible) AS all_visible_pages,
       count(*) FILTER (WHERE all_frozen) AS all_frozen_pages,
       count(*) AS total_pages
FROM pg_visibility_map('vm_frozen_demo'::regclass);
```

看 `pg_class`：

```sql
SELECT relname, relpages, relallvisible, relallfrozen,
       age(relfrozenxid) AS xid_age,
       mxid_age(relminmxid) AS mxid_age
FROM pg_class
WHERE relname = 'vm_frozen_demo';
```

看 VACUUM verbose：

```sql
VACUUM (VERBOSE) vm_frozen_demo;
```

关注：

```text
visibility map: pages set all-visible
visibility map: pages set all-frozen
frozen pages
tuples frozen
new relfrozenxid
new relminmxid
```

看最老关系：

```sql
SELECT oid::regclass AS rel,
       age(relfrozenxid) AS xid_age,
       mxid_age(relminmxid) AS mxid_age,
       relallvisible,
       relallfrozen
FROM pg_class
WHERE relkind IN ('r', 'm', 't')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

源码断点建议：

```text
heap_page_prune_and_freeze
heap_page_will_freeze
heap_freeze_prepared_tuples
visibilitymap_set
visibilitymap_clear
heap_vac_scan_next_block
```

断点里关注：

```text
old_vmbits
new_vmbits
set_all_visible
set_all_frozen
nfrozen
NewRelfrozenXid
skippedallvis
aggressive
```

## 15. 常见误区

误区一：all-frozen 只是 all-visible 的另一种叫法。

正确理解：all-frozen 是更强的 future-freeze 证明。

误区二：all-frozen 主要服务 index-only scan。

正确理解：index-only scan 主要需要 all-visible。

all-frozen 主要服务 VACUUM / anti-wraparound。

误区三：all-visible page 永远不需要 VACUUM 再看。

正确理解：如果不是 all-frozen，未来仍可能为了 freeze 被扫描。

误区四：更新一行只影响一个 tuple 的 freeze 状态。

正确理解：VM bit 是 page-level。

一个 tuple 的变化会清 page-level proof。

误区五：all-frozen clear 表示页面不可见。

正确理解：all-frozen clear 只说明不能证明无需 future freeze。

页面仍可能 all-visible。

误区六：relfrozenxid 推进只看全表最老 XID，不需要 VM。

正确理解：VM all-frozen 让 VACUUM 能安全跳过已经证明过的页面。

否则大表 anti-wraparound 成本会更高。

误区七：`VACUUM FREEZE` 后所有页面都一定 all-frozen。

正确理解：并发、锁、页面状态、跳过策略和特殊 tuple 都可能影响结果。

## 16. 课堂实验

### 实验一：观察 all-frozen

```sql
DROP TABLE IF EXISTS vm_frozen_demo;
CREATE TABLE vm_frozen_demo(id int primary key, payload text);

INSERT INTO vm_frozen_demo
SELECT g, repeat('x', 80)
FROM generate_series(1, 50000) AS g;

VACUUM (FREEZE, VERBOSE) vm_frozen_demo;

CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT count(*) FILTER (WHERE all_visible) AS all_visible_pages,
       count(*) FILTER (WHERE all_frozen) AS all_frozen_pages,
       count(*) AS total_pages
FROM pg_visibility_map('vm_frozen_demo'::regclass);
```

解释 all-visible 和 all-frozen 的差异。

### 实验二：写入清 bit

```sql
UPDATE vm_frozen_demo
SET payload = payload || 'y'
WHERE id = 1;

SELECT blkno, all_visible, all_frozen
FROM pg_visibility_map('vm_frozen_demo'::regclass)
ORDER BY blkno
LIMIT 20;
```

再执行：

```sql
VACUUM (FREEZE, VERBOSE) vm_frozen_demo;
```

比较前后 VM 状态。

### 实验三：all-visible but not all-frozen

构造一张表，执行普通 VACUUM。

```sql
DROP TABLE IF EXISTS vm_visible_not_frozen_demo;
CREATE TABLE vm_visible_not_frozen_demo(id int primary key, payload text);

INSERT INTO vm_visible_not_frozen_demo
SELECT g, repeat('z', 80)
FROM generate_series(1, 50000) AS g;

VACUUM (VERBOSE) vm_visible_not_frozen_demo;

SELECT count(*) FILTER (WHERE all_visible) AS all_visible_pages,
       count(*) FILTER (WHERE all_frozen) AS all_frozen_pages
FROM pg_visibility_map('vm_visible_not_frozen_demo'::regclass);
```

讨论为什么 all-visible 可能多于 all-frozen。

### 实验四：源码断点

```gdb
break heap_page_will_freeze
break heap_freeze_prepared_tuples
break visibilitymap_set
break visibilitymap_clear
```

执行：

```sql
VACUUM (FREEZE) vm_frozen_demo;
```

观察：

```gdb
print prstate->set_all_visible
print prstate->set_all_frozen
print prstate->new_vmbits
print presult->nfrozen
```

把 all-frozen 设置与 freeze plan 对应。

## 17. 讨论题

1. 为什么 all-frozen 必须 imply all-visible？

2. 为什么 all-visible 不能替代 all-frozen 服务 anti-wraparound？

3. 什么情况下只清 all-frozen 而不清 all-visible 是合理的？

4. eager freeze 为什么要限制成功率或失败率？

5. 如果一张冷数据大表 all-visible 很高但 all-frozen 很低，会对未来 VACUUM 造成什么影响？

6. 为什么 VM all-frozen false positive 比 false negative 更危险？

7. `relfrozenxid`、all-frozen VM bit、tuple freeze plan 三者分别处在哪个粒度？

## 18. 本节小结

本节把 VM 的第二个 bit 拆清楚了。

all-visible 是可见性承诺。

all-frozen 是 future-freeze 承诺。

all-frozen 必须建立在 all-visible 之上。

它让 VACUUM 尤其是 aggressive VACUUM 可以安全跳过页面。

它也让大表的 anti-wraparound 成本不会无限重复。

本节可迁移规律是：

```text
同一个缓存结构里可以保存不同强度的证明；
弱证明服务热路径读取，强证明服务长期维护边界。
```

下一节把 cleanup delay 的外部原因串起来：长事务、prepared transaction、replication slot 和 standby feedback 如何让 dead tuple、VACUUM、freeze 与 VM 都停在保守边界上。
