# PostgreSQL visibility map 的 all-visible 位

## 课程定位

前置知识：已经理解 heap tuple visibility、VACUUM pruning、freeze，以及 heap page 上可能同时存在 live、recently dead、dead tuple。

本节唯一主问题：

```text
PostgreSQL 为什么需要在 heap 之外维护 visibility map 的 all-visible 位，而不是每次 index-only scan 或 VACUUM 都重新检查整个 heap page？
```

核心矛盾：index-only scan 和 VACUUM page skipping 都希望低成本知道一个 heap page 是否对所有事务可见；但这个判断本质上依赖页面上所有 tuple 的事务状态，如果每次都重新逐 tuple 检查，成本会吞掉优化收益。

学完后应能判断：`VISIBILITYMAP_ALL_VISIBLE` 与 heap page 的 `PD_ALL_VISIBLE` 如何配合，什么时候可以设置，什么时候必须清除，为什么 all-visible 不等于 all-frozen。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 26 到 29 节已经把 VACUUM 的三类工作拆开。

```text
dead tuple:
  判断能不能回收

HOT pruning:
  页内缩短 chain，维护索引可达性

index cleanup:
  删除索引 TID，释放 LP_DEAD

freeze:
  移除旧 XID 依赖，推进 relfrozenxid
```

本节讲另一个横跨执行器和 VACUUM 的状态。

```text
visibility map all-visible bit
```

它回答的问题是：

```text
这个 heap page 上的所有 tuple，对所有正常事务是否都可见？
```

如果答案是 yes，两个模块能受益。

第一，index-only scan 可以少回 heap 做 visibility check。

第二，VACUUM 可以跳过许多已经 all-visible 的页面，减少 I/O。

但这个 yes 是一个强承诺。

只要页面上出现新插入、更新、删除、锁导致的可见性变化，承诺就可能失效。

因此 all-visible 的设置和清除都非常谨慎。

第 31 节会继续讲 all-frozen。

本节只讲 all-visible。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
VACUUM 或 pruning 在持 heap page 锁并证明页面上所有 tuple 对所有事务可见后，同时设置 heap page 的 PD_ALL_VISIBLE hint 和 visibility map 中的 VISIBILITYMAP_ALL_VISIBLE；
后续 index-only scan 可以通过 VM bit 跳过 heap visibility check；
任何可能让页面不再 all-visible 的 heap 修改都必须清除 VM bit 和 PD_ALL_VISIBLE。
```

本节核心矛盾是：

```text
全页可见性判断很贵
  vs
执行器和 VACUUM 需要频繁知道这个判断结果
```

PostgreSQL 的折中是缓存 page-level 结论。

但它不把这个结论只放在 heap page header。

它还放在 VM fork 中。

原因是 index-only scan 从 index 走来。

它需要低成本访问“某个 heap block 是否 all-visible”的信息。

VM fork 是按 heap block 编号映射的紧凑 bitmap。

每个 heap page 两个 bit。

本节先看第一个 bit。

```text
VISIBILITYMAP_ALL_VISIBLE = 0x01
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/access/visibilitymapdefs.h` | 定义 `BITS_PER_HEAPBLOCK`、`VISIBILITYMAP_ALL_VISIBLE`、`VISIBILITYMAP_ALL_FROZEN`。 |
| 2 | `src/include/access/visibilitymap.h` | 对外 API：`visibilitymap_get_status()`、`visibilitymap_pin()`、`visibilitymap_set()`、`visibilitymap_clear()`。 |
| 3 | `src/backend/access/heap/visibilitymap.c` | VM fork 的读、pin、set、clear、count 和 truncation 实现。 |
| 4 | `src/backend/access/heap/pruneheap.c` | `heap_page_prune_and_freeze()` 如何在 pruning 后设置 all-visible，并修复 VM / page hint 不一致。 |
| 5 | `src/backend/access/heap/vacuumlazy.c` | VACUUM 如何统计 newly all-visible pages、skip all-visible pages、第二遍 cleanup 后设置 VM。 |
| 6 | `src/backend/access/heap/heapam.c` | INSERT / UPDATE / DELETE / LOCK 路径如何清 VM bit；heap scan 如何利用 `PD_ALL_VISIBLE`。 |
| 7 | `src/backend/access/index/indexam.c` 与各 index AM | index-only scan 使用 VM 判断是否需要 heap fetch 的执行器边界。 |
| 8 | `src/include/storage/bufpage.h` | 对照 heap page header 中 `PD_ALL_VISIBLE`。 |

本节阅读重点：

```text
set 很难；
clear 必须快且保守；
get 可以是无锁近似，但调用者必须处理并发。
```

## 4. 一个 runtime 现象先定锚

准备一张能触发 index-only scan 的表。

```sql
DROP TABLE IF EXISTS vm_visible_demo;
CREATE TABLE vm_visible_demo(id int primary key, payload text);

INSERT INTO vm_visible_demo
SELECT g, repeat('x', 50)
FROM generate_series(1, 100000) AS g;

VACUUM (ANALYZE, VERBOSE) vm_visible_demo;
```

执行 index-only scan：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM vm_visible_demo
WHERE id BETWEEN 100 AND 10000;
```

如果计划选择 index-only scan，你会看到：

```text
Heap Fetches: 0
```

或者很低的 heap fetch 数。

现在做一个写入：

```sql
UPDATE vm_visible_demo
SET payload = payload || 'y'
WHERE id = 500;
```

再执行同样查询。

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM vm_visible_demo
WHERE id BETWEEN 100 AND 10000;
```

你可能看到 heap fetch 增加。

因为被修改的 heap page 不再能被 all-visible bit 证明。

再次执行：

```sql
VACUUM (VERBOSE) vm_visible_demo;
```

再看 index-only scan。

`Heap Fetches` 可能下降。

这个现象说明：

```text
all-visible bit 是执行器和 VACUUM 共享的 page-level 可见性缓存。
```

它不是计划器凭空相信索引。

它来自 VACUUM / pruning 对 heap page 的证明。

## 5. VM fork 与 page-level hint

visibility map 是 relation 的一个 fork。

它不在 main heap fork 里。

每个 heap page 对应 VM 中两个 bit：

```text
bit 0: all-visible
bit 1: all-frozen
```

`visibilitymapdefs.h` 中：

```text
BITS_PER_HEAPBLOCK = 2
VISIBILITYMAP_ALL_VISIBLE = 0x01
VISIBILITYMAP_ALL_FROZEN = 0x02
VISIBILITYMAP_VALID_BITS = 0x03
```

VM fork 的优点是紧凑。

查询某个 heap block 的 bit，不需要读整个 heap page。

但 heap page header 里还有 `PD_ALL_VISIBLE`。

这两个状态不是重复设计。

它们服务不同路径。

| 状态 | 位置 | 主要作用 |
| --- | --- | --- |
| `PD_ALL_VISIBLE` | heap page header | heap page 自身的 hint，扫描 heap page 时可以快速判断。 |
| `VISIBILITYMAP_ALL_VISIBLE` | VM fork | 从 heap 外部按 block 快速判断，服务 index-only scan 和 VACUUM skip。 |

正确性要求：

```text
VM bit set 时，heap page 必须满足 all-visible 语义。
```

`PD_ALL_VISIBLE` set 但 VM bit clear 是可以修复的低效状态。

VM bit set 但 page 不 all-visible 是严重问题。

源码中 pruning 会检查并修复某些不一致。

## 6. 设置 all-visible 为什么很谨慎

设置 all-visible 的前提是页面上没有需要任何当前或未来正常 snapshot 特别判断的 tuple。

更具体地说：

```text
所有 live tuple 对所有事务可见；
没有 LP_DEAD item；
没有 recently dead tuple；
没有 insert/delete in progress；
页面上的 newest live xid 不被 GlobalVisState 认为仍 running；
如果同时要 all-frozen，还要满足更强条件。
```

`heap_page_prune_and_freeze()` 中会维护：

```text
set_all_visible
set_all_frozen
newest_live_xid
lpdead_items
new_vmbits
```

它先扫描 tuple 和 line pointer。

再决定是否能设置。

如果页面上有 `LP_DEAD`，不能设置 all-visible。

因为 index cleanup 还没完成，页面还处于中间状态。

如果 `newest_live_xid` 仍可能被某个 snapshot 认为 running，也不能设置。

即使当前 VACUUM 看起来已经处理过页面，也不能偷懒。

设置 all-visible 是对所有事务的承诺。

这个承诺比“当前 VACUUM 看见 live”强得多。

## 7. 主流程源码 walkthrough

主流程选 VACUUM 设置 all-visible。

```text
lazy_scan_heap()
  -> visibilitymap_pin()
  -> lazy_scan_prune()
     -> heap_page_prune_and_freeze()
        -> visibilitymap_get_status()
        -> prune_freeze_plan()
        -> heap_page_will_set_vm()
        -> LockBuffer(vmbuffer, BUFFER_LOCK_EXCLUSIVE)
        -> START_CRIT_SECTION()
        -> PageSetAllVisible()
        -> PageClearPrunable()
        -> visibilitymap_set(... VISIBILITYMAP_ALL_VISIBLE ...)
        -> log_heap_prune_and_freeze()
        -> END_CRIT_SECTION()
```

第一步，VACUUM 在处理 heap page 前 pin 对应 VM page。

`visibilitymap_pin()` 可能需要 I/O。

源码要求先 pin，再持 heap page 锁做设置。

这是为了避免在持 heap page lock 时读 VM page 导致死锁或长等待。

第二步，`heap_page_prune_and_freeze()` 读旧 VM bit。

```text
old_vmbits = visibilitymap_get_status(...)
```

如果 VM bit 和 page hint 不一致，它可能修复。

第三步，计划阶段扫描页面。

它计算是否还有 dead / recently dead / in-progress tuple。

它也统计是否有 LP_DEAD。

第四步，`heap_page_will_set_vm()` 决定是否设置 VM。

on-access pruning 路径更保守。

VACUUM 路径更系统。

第五步，真正设置时先 lock VM buffer。

然后进入 critical section。

在 critical section 中：

```text
PageSetAllVisible(page)
PageClearPrunable(page)
visibilitymap_set(block, vmbuffer, flags, rlocator)
MarkBufferDirty(heap buffer)
WAL record
```

第六步，`visibilitymap_set()` 只设置 bit，不负责证明语义。

它假设调用者已经持有正确锁并完成证明。

这正是 VM API 的边界：

```text
visibilitymap.c 管位图存取；
heap/vacuum 管语义证明。
```

## 8. 清除 all-visible 的主流程

设置难。

清除必须保守且快速。

典型写入路径在 `heapam.c`。

INSERT、UPDATE、DELETE、LOCK 等路径如果发现页面 `PD_ALL_VISIBLE`，会：

```text
visibilitymap_pin()
PageClearAllVisible(page)
visibilitymap_clear(... VISIBILITYMAP_VALID_BITS)
```

为什么传 `VISIBILITYMAP_VALID_BITS`？

因为 all-frozen 依赖 all-visible。

如果 all-visible 失效，all-frozen 也必须失效。

`visibilitymap_clear()` 中有断言：

```text
不能只清 all-visible 而留下 all-frozen
```

这就是两个 bit 的层次关系。

写路径一般只需要保守清除。

它不需要重新证明页面是否仍 all-visible。

因为重新证明太贵。

以后 VACUUM 或 pruning 会重新设置。

因此 all-visible 是一个可丢失的优化状态。

它可以保守地被清掉。

但不能错误地被保留。

## 9. 生命周期 / ownership / cleanup

一个 heap page 的 all-visible 生命周期如下。

阶段一：普通页面。

VM bit clear。

index-only scan 需要回 heap 检查 visibility。

VACUUM 需要扫描页面。

阶段二：VACUUM 扫描并证明全页可见。

没有 recently dead。

没有 in-progress。

没有 LP_DEAD。

newest live xid 不被认为 running。

阶段三：设置 all-visible。

heap page 设置 `PD_ALL_VISIBLE`。

VM fork 设置 `VISIBILITYMAP_ALL_VISIBLE`。

WAL 记录相关页面修改。

阶段四：执行器消费。

index-only scan 查询 VM bit。

如果 bit set，可以跳过 heap visibility check。

阶段五：VACUUM 消费。

后续 VACUUM 可以根据 VM bit 跳过页面。

如果需要 freeze，是否跳过还要看 all-frozen 和 aggressive 条件。

阶段六：页面被修改。

INSERT、UPDATE、DELETE 或某些锁语义使页面不再保证 all-visible。

写路径清除 page hint 和 VM bit。

阶段七：下一次 VACUUM 重新证明。

如果页面再次满足条件，bit 可以重新设置。

ownership 分层：

```text
VM bit:
  relation fork 中的持久优化状态

PD_ALL_VISIBLE:
  heap page header 中的 page-local hint

证明责任:
  heap pruning / VACUUM

消费责任:
  executor index-only scan / VACUUM page skipping

失效责任:
  heap write path
```

## 10. 正确性机制层次

第一层是保守设置。

只有证明全页对所有事务可见，才能 set。

第二层是保守清除。

任何可能破坏全页可见性的写入都要 clear。

第三层是锁顺序。

设置需要提前 pin VM page。

设置时持 heap page lock 和 VM page lock。

第四层是 WAL 顺序。

设置 VM bit 与让页面 all-visible 的 heap 修改要以可恢复方式记录。

第五层是 page hint / VM consistency。

VM set 但 `PD_ALL_VISIBLE` clear 会被视为不一致并修复。

第六层是消费方兜底。

`visibilitymap_get_status()` 注释明确说调用者要处理并发。

读取 VM bit 不是获得一个锁定到未来的事实。

它是一个在正确协议下可用的优化信号。

第七层是 all-visible / all-frozen 层次。

all-frozen 必须 imply all-visible。

清 all-visible 必须清 all-frozen。

但 all-visible 不 imply all-frozen。

## 11. 错误路径 / 异常路径 / fallback

### VM page 不存在

`visibilitymap_pin()` 如果需要设置 bit，会读取或扩展 VM fork。

如果只是查询状态，`visibilitymap_get_status()` 可能返回 clear。

调用者必须能处理 VM 缺页。

### 传错 VM buffer

`visibilitymap_clear()` 和 `visibilitymap_set()` 都检查 VM buffer 是否覆盖目标 heap block。

传错会 ERROR。

这保护了 block-to-bit 映射。

### on-access pruning 不设置 VM

on-access pruning 只有在 relation 被认为 read-only 等条件满足时才尝试设置 VM。

如果查询马上要修改 relation，设置后又清除没有价值。

### VM 与 page hint 不一致

pruning 会检查：

```text
VM bits set
PD_ALL_VISIBLE clear
```

并修复缺失 page hint 或清 VM bit。

这种修复说明 VM 是持久状态，但仍需要和 heap page 协议保持一致。

### 写入后 heap fetch 增加

更新一个 page 会清除该 page 的 all-visible bit。

后续 index-only scan 可能对该 page 产生 heap fetch。

这不是 planner bug。

这是可见性缓存失效。

### standby / recovery

recovery 中 on-access pruning 不主动写清理 WAL。

VM 的变化主要通过 replay primary 产生的 WAL 达成。

## 12. 成本、资源与跨模块传播

all-visible 的收益体现在两个路径。

第一是 index-only scan。

如果查询只需要索引中的列，并且 heap page all-visible，执行器可以跳过 heap visibility check。

这减少随机 heap 访问。

第二是 VACUUM。

VACUUM 可以跳过 all-visible 页面，尤其在大表上减少 I/O。

但成本也存在。

| 成本 | 说明 |
| --- | --- |
| 设置成本 | 需要扫描页面证明、pin VM page、写 heap page 和 VM bit、可能写 WAL。 |
| 清除成本 | 写路径发现 `PD_ALL_VISIBLE` 时要 pin VM page 并 clear。 |
| 失效频率 | 高频 UPDATE 的 page 很难长期保持 all-visible。 |
| VM I/O | VM fork 很小，但仍是额外 fork 和 buffer。 |
| false negative | 保守清除会让优化暂时失效，直到下次 VACUUM。 |
| correctness risk | false positive 不可接受，会让 index-only scan 看到错误结果。 |

跨模块传播：

```text
VACUUM / pruning:
  证明并设置 all-visible

heap write path:
  清除 all-visible

executor:
  index-only scan 消费 VM bit

planner / stats:
  relallvisible 影响成本估算

pg_class:
  relallvisible 记录 all-visible page 数
```

因此 all-visible 是一个典型的跨层优化状态。

它既不是纯 executor 状态。

也不是纯 storage 状态。

## 13. 观测与诊断入口

看 index-only scan 是否回 heap：

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM vm_visible_demo
WHERE id BETWEEN 100 AND 10000;
```

关注：

```text
Index Only Scan
Heap Fetches
```

看 relation all-visible 比例：

```sql
SELECT relname, relpages, relallvisible,
       round(100.0 * relallvisible / NULLIF(relpages, 0), 2) AS all_visible_pct
FROM pg_class
WHERE relname = 'vm_visible_demo';
```

看 VACUUM verbose：

```sql
VACUUM (VERBOSE) vm_visible_demo;
```

关注：

```text
visibility map: pages set all-visible
pages scanned
pages skipped
```

如果安装 `pg_visibility` extension：

```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT *
FROM pg_visibility_map('vm_visible_demo'::regclass)
LIMIT 20;
```

源码断点建议：

```text
visibilitymap_set
visibilitymap_clear
visibilitymap_get_status
heap_page_prune_and_freeze
lazy_vacuum_heap_page
heap_update
heap_delete
```

断点里关注：

```text
heapBlk
flags
old_vmbits
new_vmbits
PageIsAllVisible(page)
```

## 14. 常见误区

误区一：all-visible 表示页面没有 dead tuple。

正确理解：它表示页面上所有 tuple 对所有事务可见。

如果有 `LP_DEAD`，通常不能 set。

但概念上它不是“无死行”计数。

误区二：all-visible 等于 all-frozen。

正确理解：all-visible 是可见性承诺。

all-frozen 是不再需要 freeze 的更强承诺。

误区三：index-only scan 永远不访问 heap。

正确理解：只有 VM bit 证明对应 heap page all-visible 时，才能跳过 heap visibility check。

否则仍要 heap fetch。

误区四：VM bit 只影响 VACUUM。

正确理解：executor 也消费 VM bit。

index-only scan 是最典型场景。

误区五：写入一个 tuple 只影响这个 tuple。

正确理解：all-visible 是 page-level bit。

写入会清整个 page 的 all-visible 承诺。

误区六：VM bit clear 说明页面一定不可 all-visible。

正确理解：clear 可能只是保守失效。

页面可能事实上全可见，但还没有被 VACUUM 重新证明。

误区七：`relallvisible` 是实时精确值。

正确理解：它由 VACUUM 更新，用于估算。

诊断时要结合 VACUUM 时机。

## 15. 课堂实验

### 实验一：Heap Fetches 与 all-visible

```sql
DROP TABLE IF EXISTS vm_visible_demo;
CREATE TABLE vm_visible_demo(id int primary key, payload text);

INSERT INTO vm_visible_demo
SELECT g, repeat('x', 50)
FROM generate_series(1, 100000) AS g;

VACUUM (ANALYZE, VERBOSE) vm_visible_demo;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM vm_visible_demo
WHERE id BETWEEN 100 AND 10000;
```

记录 `Heap Fetches`。

然后：

```sql
UPDATE vm_visible_demo
SET payload = payload || 'y'
WHERE id = 500;

EXPLAIN (ANALYZE, BUFFERS)
SELECT id
FROM vm_visible_demo
WHERE id BETWEEN 100 AND 10000;
```

解释：

```text
UPDATE 清除了所在 page 的 all-visible bit。
index-only scan 对该 page 可能需要回 heap。
```

再：

```sql
VACUUM (VERBOSE) vm_visible_demo;
```

比较结果。

### 实验二：观察 relallvisible

```sql
SELECT relpages, relallvisible
FROM pg_class
WHERE oid = 'vm_visible_demo'::regclass;
```

做一批 UPDATE。

再执行 VACUUM。

比较 `relallvisible` 的变化。

### 实验三：使用 pg_visibility

```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;

SELECT blkno, all_visible, all_frozen
FROM pg_visibility_map('vm_visible_demo'::regclass)
ORDER BY blkno
LIMIT 50;
```

解释：

```text
all_visible 可以 true。
all_frozen 可以 false。
二者不是同义词。
```

### 实验四：源码断点

```gdb
break visibilitymap_set
break visibilitymap_clear
break heap_page_prune_and_freeze
```

执行：

```sql
VACUUM vm_visible_demo;
UPDATE vm_visible_demo SET payload = payload || 'z' WHERE id = 1000;
```

观察：

```gdb
print heapBlk
print flags
print prstate.new_vmbits
```

把 set / clear 时机和 SQL 对应。

## 16. 讨论题

1. 为什么 all-visible 要放在 VM fork，而不是只放在 heap page header？

2. 为什么写入路径可以保守清除 all-visible，而不能保守设置 all-visible？

3. `PD_ALL_VISIBLE` set 但 VM clear 会导致什么成本问题？

4. VM set 但 heap page 实际不 all-visible 会导致什么 correctness 问题？

5. 为什么 all-visible page 仍可能不是 all-frozen page？

6. 高频 UPDATE 表为什么很难从 index-only scan 中获得稳定收益？

7. `relallvisible` 如何影响 planner 对 index-only scan 的成本估算？

## 17. 本节小结

本节建立 all-visible 的 page-level 缓存模型。

`VISIBILITYMAP_ALL_VISIBLE` 表示：

```text
某个 heap page 上所有 tuple 都对所有事务可见。
```

这个结论由 VACUUM / pruning 证明。

由 executor 和 VACUUM 消费。

由 heap write path 保守清除。

设置 all-visible 需要强证明。

清除 all-visible 只需要怀疑页面可能不再满足条件。

这就是优化状态的基本正确性策略。

本节可迁移规律是：

```text
跨模块缓存可以极大降低热路径成本；
但它必须允许 false negative，绝不能允许 false positive。
```

下一节继续讲第二个 VM bit：all-frozen。它在 all-visible 的基础上进一步承诺页面不再含需要 freeze 的事务身份。
