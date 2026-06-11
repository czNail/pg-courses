# PostgreSQL MultiXact wraparound 与冻结边界

## 课程定位

前置知识：已经理解 `xmax` 可以引用 MultiXactId，知道 MultiXact 成员保存 row lock / updater 状态，也知道 VACUUM cleanup horizon 不能只看当前查询可见性。

本节唯一主问题：

```text
MultiXactId 也会回卷时，VACUUM 如何判断 tuple 上的旧 MultiXact 引用能否删除、替换或保留？
```

本节围绕的核心矛盾：

```text
pg_multixact SLRU 必须截断旧段以避免 wraparound 和磁盘膨胀；
但只要某个 heap tuple 的 xmax 仍引用旧 MultiXact，就必须保留足够信息来解释 locker 和 updater 成员。
```

学完后应能独立判断：

- 为什么 MultiXact cleanup 不等于成员事务结束。
- 为什么 VACUUM 同时计算 `OldestMxact` 和 `MultiXactCutoff`。
- 为什么 `relminmxid` 是每张表的 MultiXact 保留边界。
- 为什么 `datminmxid` 是数据库级截断边界。
- 为什么 `FreezeMultiXactId()` 有 invalidate、return XID、return Multi、noop 四类结果。
- 为什么 pure locker 可以清理，而 committed updater 可能必须保留。
- 为什么 VACUUM 有时会创建新的 MultiXact。
- 为什么 MultiXact member 空间膨胀会让 autovacuum 更激进。
- 为什么 wraparound 保护会在极端情况下拒绝分配新的 MultiXactId。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 22 节已经说明：

```text
MultiXact 是 tuple xmax 的间接引用层；
它把多个 row lock / updater member 保存到 pg_multixact SLRU。
```

第 24 节说明：

```text
外键 key-share 等场景会制造大量 MultiXact。
```

现在要回答生命周期的最后一段：

```text
这些 MultiXact 什么时候可以被清理？
```

这和普通 XID freeze 类似，但更复杂。

普通 `xmin` freeze 主要关心：

```text
这个 XID 是否足够老，并且已知 committed？
```

MultiXact freeze 还要关心：

- MultiXactId 本身是否太老。
- 成员 XID 是否太老。
- 成员是 locker 还是 updater。
- updater 是否 committed。
- 是否仍有 running locker。
- 是否可以把 MultiXact 替换成单个 updater XID。
- 是否必须创建一个只含 surviving members 的新 MultiXact。
- 表的 `relminmxid` 能否推进。

本节主线是：

```text
VACUUM 计算 cutoffs
  -> heap scan 遇到 xmax is MultiXact
  -> FreezeMultiXactId() 展开成员
  -> 生成 freeze plan
  -> heap_execute_freeze_tuple() 改写 tuple header
  -> lazy VACUUM 推进 relminmxid
  -> database datminmxid 推进
  -> checkpoint / TruncateMultiXact() 截断 SLRU
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
VACUUM 用 vacuum_get_cutoffs() 计算 OldestMxact 和 MultiXactCutoff；
heap_prepare_freeze_tuple() 遇到 xmax MultiXact 时调用 FreezeMultiXactId()；
FreezeMultiXactId() 根据 MultiXact 年龄、成员事务状态和 cutoff 决定清掉 xmax、替换为 updater XID、替换为新 MultiXact 或暂时不动；
只有所有表的 relminmxid 和数据库 datminmxid 推进后，pg_multixact SLRU 才能截断。
```

核心矛盾可以拆成两句：

```text
太早截断 pg_multixact，会让仍在 heap tuple header 中的 xmax 无法解释。
太晚清理旧 MultiXact，会造成 wraparound 风险、SLRU 膨胀和 autovacuum 压力。
```

所以 PostgreSQL 不是简单地：

```text
成员事务都结束 -> 删除 MultiXact。
```

它必须先消除 tuple header 引用。

消除引用又必须保持 tuple visibility 和 row lock 语义。

本节的系统规律是：

```text
持久化间接 ID 的回收分两阶段：
先改写所有引用者，使旧 ID 不再被解释；
再推进全局最小保留边界，安全截断 ID 所在的持久化存储。
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/heap/README.tuplock` | MultiXact 与 `relminmxid` / `datminmxid` 的关系。 |
| 2 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()` 计算 `OldestMxact`、`MultiXactCutoff`、aggressive vacuum。 |
| 3 | `src/include/commands/vacuum.h` | `VacuumCutoffs` 字段语义。 |
| 4 | `src/backend/access/heap/heapam.c` | `FreezeMultiXactId()`、`heap_prepare_freeze_tuple()`、`heap_freeze_tuple()`。 |
| 5 | `src/include/access/heapam.h` | heap freeze plan 结构与 table AM 边界。 |
| 6 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 如何执行 freeze plan、推进 `relminmxid`。 |
| 7 | `src/backend/access/transam/multixact.c` | `GetOldestMultiXactId()`、`SetMultiXactIdLimit()`、`TruncateMultiXact()`。 |
| 8 | `src/backend/postmaster/autovacuum.c` | `relminmxid` 年龄如何触发 anti-wraparound autovacuum。 |
| 9 | `src/backend/utils/adt/multixactfuncs.c` | `pg_get_multixact_stats()` 观测当前 MultiXact 使用量。 |
| 10 | `src/include/catalog/pg_class.h` / `src/include/catalog/pg_database.h` | `relminmxid`、`datminmxid` 元数据。 |

推荐阅读顺序：

```text
先读 vacuum_get_cutoffs()
  -> 读 FreezeMultiXactId()
  -> 读 heap_prepare_freeze_tuple()
  -> 读 vacuumlazy.c 推进 relminmxid
  -> 读 multixact.c 的 GetOldestMultiXactId() 和 TruncateMultiXact()
  -> 最后读 autovacuum.c 的 wraparound trigger
```

不要从 `TruncateMultiXact()` 开始。

截断是最后一步。

前面的关键是：

```text
heap tuple header 中是否还可能引用旧 MultiXact？
```

## 4. 一个 runtime 现象先定锚

MultiXact wraparound 不适合在普通开发库里强行复现。

但可以复现它的轻量前置现象：

```text
多个事务同时 key-share / share 锁同一行
  -> tuple xmax 变成 MultiXactId
  -> VACUUM 需要处理 relminmxid
  -> pg_get_multixact_stats() 显示 MultiXact 使用量变化
```

Session 0：

```sql
DROP TABLE IF EXISTS mx_freeze_demo;
CREATE TABLE mx_freeze_demo(id int primary key, payload text);
INSERT INTO mx_freeze_demo VALUES (1, 'a');
CREATE EXTENSION IF NOT EXISTS pageinspect;
```

多个 session：

```sql
BEGIN;
SELECT * FROM mx_freeze_demo WHERE id = 1 FOR SHARE;
```

观察：

```sql
SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('mx_freeze_demo', 0));

SELECT *
FROM pg_get_multixact_stats();
```

结束所有事务后：

```sql
VACUUM (VERBOSE, FREEZE) mx_freeze_demo;
```

观察 verbose 输出中 `relminmxid` 相关信息。

这个实验不要求制造真正 wraparound。

它锚定一个事实：

```text
MultiXact 是写进 tuple header 的持久引用；
VACUUM 必须处理这个引用后，系统才能推进 MultiXact 保留边界。
```

## 5. `vacuum_get_cutoffs()` 计算了什么

`vacuum_get_cutoffs()` 是本节入口。

它为 VACUUM 计算一组 cutoffs。

和 MultiXact 相关的字段包括：

```text
relminmxid
OldestMxact
MultiXactCutoff
```

`relminmxid` 来自 relation 元数据：

```text
rel->rd_rel->relminmxid
```

它表示：

```text
这张表中仍可能存在的最老 MultiXact 引用边界。
```

`OldestMxact` 来自：

```text
GetOldestMultiXactId()
```

它表示：

```text
当前仍可能被 running transaction 认为 live 的最老 MultiXactId。
```

`MultiXactCutoff` 根据 next MXID 和 `vacuum_multixact_freeze_min_age` 计算。

但它必须满足：

```text
MultiXactCutoff <= OldestMxact
```

如果计算出的 cutoff 太新，必须回退到 `OldestMxact`。

原因是：

```text
不能 freeze 掉仍可能被活动事务解释的 MultiXact。
```

当表的 `relminmxid` 太老时，VACUUM 进入 aggressive 模式。

aggressive vacuum 必须努力推进：

```text
relminmxid >= MultiXactCutoff
```

## 6. `OldestMxact` 与 `MultiXactCutoff` 的区别

这两个名字容易混。

`OldestMxact` 是安全边界：

```text
比它更老的 MultiXact 不应有 running member。
```

`MultiXactCutoff` 是 freeze 目标：

```text
VACUUM 希望消除或改写小于它的 MultiXact 引用。
```

通常：

```text
MultiXactCutoff <= OldestMxact
```

如果一个 MultiXact 小于 `MultiXactCutoff`，VACUUM 必须处理。

如果一个 MultiXact 大于等于 cutoff，VACUUM 可能暂时保留。

但还要看成员 XID 是否小于 `FreezeLimit`。

因为 MultiXact 本身不老，不代表成员 XID 都不老。

这就是 MultiXact 比普通 XID freeze 复杂的地方：

```text
要同时考虑 ID 年龄和成员 XID 年龄。
```

## 7. `FreezeMultiXactId()` 的四类结果

`FreezeMultiXactId()` 是本节核心函数。

它输入：

```text
multi
t_infomask
VacuumCutoffs
HeapPageFreeze
```

输出：

- 新 `xmax`。
- flags。
- page freeze 状态。

可以把 flags 结果压缩成四类。

### 7.1 invalidate xmax

对应：

```text
FRM_INVALIDATE_XMAX
```

含义：

```text
这个 MultiXact 不再有需要保留的语义；
tuple xmax 可以标成 invalid。
```

常见场景：

- MultiXact 无效。
- pg_upgrade 旧 lock-only 形态。
- old lock-only MultiXact 已无 running member。
- updater abort，且没有 surviving locker。

### 7.2 return single updater XID

对应：

```text
FRM_RETURN_IS_XID
```

含义：

```text
MultiXact 中只有一个需要保留的 updater；
可以把 xmax 从 MultiXactId 改回普通 XID。
```

如果 updater 已提交，还可能设置：

```text
FRM_MARK_COMMITTED
```

这样后续 visibility 能用普通 XID + hint bit 解释。

### 7.3 return new MultiXact

对应：

```text
FRM_RETURN_IS_MULTI
```

含义：

```text
旧 MultiXact 太老或成员太老；
但仍有多个 surviving members 必须保留；
VACUUM 创建一个新的 MultiXact 来替换旧引用。
```

这看起来反直觉：

```text
VACUUM 为什么会创建新的 MultiXact？
```

原因是它要消除旧 ID。

如果仍有多个成员不能丢，只能创建一个新的、足够新的 MultiXact。

### 7.4 noop

对应：

```text
FRM_NOOP
```

含义：

```text
暂时不改这个 xmax；
但 VACUUM 可能需要回退 page-level relfrozenxid / relminmxid tracker。
```

noop 不是没看见。

它表示当前不需要或不能处理，但必须确保 relation metadata 不会推进到越过仍保留的引用。

## 8. locker 和 updater 的不同命运

MultiXact 成员分两类。

pure locker：

```text
ForKeyShare
ForShare
ForNoKeyUpdate
ForUpdate
```

updater：

```text
NoKeyUpdate
Update
```

成员事务结束后，两类命运不同。

pure locker：

```text
如果不再 running，就没有长期语义。
可以丢掉。
```

updater：

```text
如果 abort，可以丢掉。
如果 commit，必须保留更新语义。
```

所以 `FreezeMultiXactId()` 的第二 pass 会遍历成员：

```text
locker:
  只保留 still running locker

updater:
  如果 in progress 或 committed，可能保留
  如果 aborted，忽略
```

它还检查不能出现两个 updater。

如果发现两个 updater，是数据损坏级错误。

这里的正确性边界是：

```text
VACUUM 可以删除锁的历史；
但不能删除已经提交的版本链事实。
```

## 9. 为什么 old Multi 可能变成普通 XID

假设一个 MultiXact 包含：

```text
locker A
updater B
locker C
```

VACUUM 时：

- A 已结束。
- C 已结束。
- B 已提交。

此时 MultiXact 作为集合已经没必要保留。

但 B 的更新语义必须保留。

所以最好结果是：

```text
xmax = B
HEAP_XMAX_COMMITTED
不再 HEAP_XMAX_IS_MULTI
```

这样后续 visibility 不必再展开 MultiXact。

也减少 SLRU 依赖。

这就是 `FRM_RETURN_IS_XID`。

它体现了一个优化方向：

```text
能从复杂间接引用退回简单 XID 时，就退回简单 XID。
```

## 10. 为什么 VACUUM 有时创建新 MultiXact

再看另一个场景。

旧 MultiXact 太老。

但成员里仍有：

- 一个 running locker。
- 一个 committed updater。

旧 MultiXactId 小于 `MultiXactCutoff`。

VACUUM 不能让旧 ID 留在 tuple header。

但也不能丢掉 surviving members。

因此它创建新 MultiXact：

```text
newmembers = surviving members
newxmax = MultiXactIdCreateFromMembers(newmembers)
```

然后 tuple header 改写成新 MultiXactId。

这样：

```text
旧 pg_multixact 段可以在全局边界推进后截断；
tuple 仍能表达必要的 row lock / updater 语义。
```

源码注释也强调：

```text
VACUUM 分配新 MultiXact 应尽量避免；
但有时为了满足 cutoff 后置条件只能这么做。
```

这是一种典型系统取舍：

```text
短期多写一点元数据；
换取长期 ID 空间和 SLRU 截断安全。
```

## 11. `heap_prepare_freeze_tuple()` 的 freeze plan

`heap_prepare_freeze_tuple()` 不直接随手改 tuple。

它先生成 freeze plan。

输入是：

- tuple header。
- `VacuumCutoffs`。
- page-level freeze state。

输出是：

- `HeapTupleFreeze` 计划。
- `totally_frozen`。
- page freeze requirement。

处理 `xmax` 时：

```text
如果 xmax 是 MultiXact
  -> FreezeMultiXactId()
  -> 根据 flags 修改 freeze plan

如果 xmax 是普通 XID
  -> 按 XID freeze 规则处理
```

为什么要先计划？

因为 VACUUM 以 page 为单位处理。

它需要：

- 判断 page 是否必须 freeze。
- 维护 page-level relfrozenxid / relminmxid tracker。
- 决定是否执行 freeze plan。
- 让 WAL 和 buffer 修改集中发生。

这也解释了 `FRM_NOOP` 的意义。

noop 仍可能影响 page-level tracker。

因为留下一个旧 MultiXact，就不能把 page 或 relation 的 `relminmxid` 推得太新。

## 12. `relminmxid` 如何推进

每张 heap relation 有：

```text
pg_class.relminmxid
```

它表示这张表中仍可能出现的最老 MultiXactId。

VACUUM 要推进它，必须证明：

```text
表里没有小于新 relminmxid 的 MultiXact 引用。
```

证明来源是 heap scan。

VACUUM 扫描 page，处理 tuple 上的 MultiXact。

如果某些 page 被跳过，或者某些 old MultiXact 被保留，`relminmxid` 不能越过它们。

lazy VACUUM 会跟踪：

- `NewRelminMxid`。
- page-level `FreezePageRelminMxid`。
- aggressive vacuum 是否必须推进。

当 VACUUM 完成后，它更新 `pg_class.relminmxid`。

verbose 输出中可能看到：

```text
new relminmxid: ...
```

这不是统计信息。

它是后续 `pg_multixact` 截断的前提。

## 13. `datminmxid` 与 SLRU 截断

单表 `relminmxid` 推进还不够。

一个数据库里可能有很多表。

数据库级边界是：

```text
pg_database.datminmxid
```

它是该数据库所有 relation `relminmxid` 的最小值。

整个集群截断 `pg_multixact` 时，还要考虑所有 database 的 `datminmxid`。

`multixact.c` 中 `TruncateMultiXact()` 的注释说明：

```text
只有在没有任何表可能引用更老 MultiXact 时，才能截断 SLRU。
```

截断涉及两个 SLRU：

- offsets。
- members。

members 截断要找到对应 offset。

所以 `SetOldestOffset()` 会计算最老 member offset。

这就是为什么 MultiXact cleanup 比普通 tuple hint bit 清理慢得多。

它是 relation -> database -> cluster -> checkpoint 的链路。

## 14. wraparound 保护

MultiXactId 是有限空间。

源码中 `GetNewMultiXactId()` 检查：

- `multiVacLimit`。
- `multiWarnLimit`。
- `multiStopLimit`。
- `multiWrapLimit`。

越过 vacuum limit 时，系统会启动 autovacuum launcher。

越过 warn limit，会发 warning。

越过 stop limit，会拒绝分配新的 MultiXactId。

错误信息大意是：

```text
database is not accepting commands that assign new MultiXactIds to avoid wraparound data loss
```

这是防止 ID 回卷导致错误解释旧 tuple header 的 correctness 保护。

如果 MultiXactId 回卷，而旧 tuple header 仍引用同一个数值，系统就无法知道它指向旧成员还是新成员。

所以必须在回卷前通过 VACUUM 消除旧引用。

## 15. member 空间膨胀的特殊压力

MultiXactId 数量不是唯一风险。

members 空间也可能膨胀。

例如很多 MultiXact 每个都有大量成员。

这时：

```text
MultiXactId 数量增长不算特别快；
但 member offset 增长很快。
```

`MultiXactMemberFreezeThreshold()` 会读取：

- 当前 multixacts 数量。
- next offset。
- oldest offset。
- members 使用量。

如果 member space 超过阈值，它会降低 effective multixact freeze max age。

极端情况下，它可以返回 0。

这会让 VACUUM 对 MultiXact freeze 更激进。

这个机制说明：

```text
MultiXact 压力有两个维度：
ID 年龄；
成员存储量。
```

诊断时不能只看 age。

也要看成员空间。

## 16. autovacuum 如何参与

`autovacuum.c` 中 `relation_needs_vacanalyze()` 会检查：

```text
relfrozenxid 是否太老
relminmxid 是否太老
dead tuple 是否超过阈值
insert/update/delete 是否超过 analyze 阈值
```

对 MultiXact 来说，关键是：

```text
relminmxid < recentMulti - multixact_freeze_max_age
```

如果成立，强制 vacuum。

这就是 anti-wraparound autovacuum 的 MultiXact 版本。

autovacuum 还会计算 score。

当前源码对 approaching wraparound 的表会提高优先级。

这意味着：

```text
高 MultiXact age 的表，即使 dead tuple 不多，也会被 vacuum。
```

不要把这种 autovacuum 误判成普通膨胀清理。

它的主要目标是推进冻结边界。

## 17. 异常路径 / fallback

### 17.1 MultiXact 早于 `relminmxid`

如果 `FreezeMultiXactId()` 发现 tuple 上的 MultiXact 早于 relation 的 `relminmxid`，会报数据损坏级错误。

因为 relation 元数据已经承诺：

```text
表中不应再有这么老的 MultiXact 引用。
```

这说明 `relminmxid` 是 correctness contract。

### 17.2 old Multi 仍 running

如果 MultiXact 小于 `OldestMxact`，理论上不应有 running member。

源码仍会验证。

如果发现仍 running，会报 corruption。

这是保护不变量：

```text
cutoff 推进必须保守。
```

### 17.3 committed updater 太老

如果 MultiXact 中 committed updater 早于 `OldestXmin`，按正常逻辑旧 tuple 应该已经可移除。

如果它还在被 freeze 处理，源码会进行一致性检查。

异常表示 heap pruning / visibility 边界有问题。

### 17.4 无法截断 members

`SetOldestOffset()` 如果找不到 oldest MultiXact 的 offset，会记录日志并禁用 member truncation。

这是保守 fallback。

系统宁可不截断，也不能误删仍可能需要的 member 数据。

### 17.5 failsafe

当 `relfrozenxid` 或 `relminmxid` 太危险时，VACUUM 会进入 failsafe。

它会跳过某些非关键工作，优先推进 freeze 边界。

这是 correctness 优先于清理质量的典型路径。

## 18. 成本、资源与跨模块传播

### 18.1 heap scan 成本

处理 MultiXact freeze 需要展开成员。

这可能访问 `pg_multixact` SLRU。

如果成员多或 cache miss 多，VACUUM CPU 和 I/O 都会上升。

### 18.2 WAL 成本

freeze 改写 tuple header 需要 WAL。

如果 VACUUM 创建新 MultiXact，也要写 MultiXact WAL。

所以 aggressive MultiXact cleanup 可能产生明显 WAL。

### 18.3 autovacuum 传播

MultiXact age 可以独立触发 autovacuum。

即使表 dead tuple 不多，也可能被 vacuum。

这会影响 I/O、buffer cache、WAL 和锁等待。

### 18.4 SLRU 磁盘空间

如果 `relminmxid` / `datminmxid` 被老表钉住，`pg_multixact` 不能截断。

磁盘空间会增长。

热点外键或共享锁 workload 会放大。

### 18.5 停写风险

接近 stop limit 时，分配新 MultiXactId 的命令会失败。

受影响的不只是显式 `SELECT FOR SHARE`。

外键检查、UPDATE、DELETE 都可能需要 MultiXact。

所以这是实例级风险。

## 19. 观测与诊断入口

### 19.1 relation 边界

```sql
SELECT relname, relminmxid, mxid_age(relminmxid)
FROM pg_class
WHERE relkind IN ('r', 't', 'm')
ORDER BY mxid_age(relminmxid) DESC
LIMIT 20;
```

关注最老的表。

这些表可能钉住 database `datminmxid`。

### 19.2 database 边界

```sql
SELECT datname, datminmxid, mxid_age(datminmxid)
FROM pg_database
ORDER BY mxid_age(datminmxid) DESC;
```

最老数据库决定集群截断压力。

### 19.3 MultiXact stats

```sql
SELECT *
FROM pg_get_multixact_stats();
```

看 next、oldest、offset、members 使用量等版本相关字段。

以本地函数输出为准。

### 19.4 VACUUM verbose

```sql
VACUUM (VERBOSE, FREEZE) mx_freeze_demo;
```

关注：

- old / new `relminmxid`。
- skipped pages。
- aggressive vacuum。
- frozen tuple。

### 19.5 日志告警

搜索：

```text
cutoff for freezing multixacts is far in the past
database must be vacuumed before ... MultiXactIds are used
not accepting commands that assign new MultiXactIds
```

这些告警优先级高。

不要按普通 bloat 问题处理。

### 19.6 源码断点

建议断点：

```text
vacuum_get_cutoffs
heap_prepare_freeze_tuple
FreezeMultiXactId
MultiXactMemberFreezeThreshold
GetOldestMultiXactId
TruncateMultiXact
relation_needs_vacanalyze
```

观察变量：

```text
OldestMxact
MultiXactCutoff
relminmxid
multi
nmembers
members[i].status
FRM_* flags
FreezePageRelminMxid
```

## 20. 常见误区

误区一：

```text
成员事务结束后 MultiXact 就可以从磁盘删掉。
```

错误。

必须先消除所有 tuple header 引用。

误区二：

```text
relminmxid 是统计值。
```

错误。

它是 relation 级 correctness 边界。

误区三：

```text
MultiXact freeze 只会清理，不会创建新 MultiXact。
```

错误。

必要时 VACUUM 会创建新 MultiXact 来替换旧引用。

误区四：

```text
只要没有 dead tuple，就不需要 vacuum。
```

错误。

MultiXact age 太老也会强制 vacuum。

误区五：

```text
pg_multixact 只和 SELECT FOR SHARE 有关。
```

错误。

外键、key-share、no-key update、update conflict 都可能参与。

## 21. 课堂实验

### 实验一：制造 MultiXact 引用并观察 relminmxid

步骤：

```sql
DROP TABLE IF EXISTS mx_freeze_demo;
CREATE TABLE mx_freeze_demo(id int primary key, payload text);
INSERT INTO mx_freeze_demo VALUES (1, 'a');
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT relminmxid, mxid_age(relminmxid)
FROM pg_class
WHERE oid = 'mx_freeze_demo'::regclass;
```

多个 session：

```sql
BEGIN;
SELECT * FROM mx_freeze_demo WHERE id = 1 FOR SHARE;
```

再观察 tuple header。

### 实验二：展开成员

```sql
SELECT lp, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('mx_freeze_demo', 0));

SELECT *
FROM pg_get_multixact_members('<mxid>'::xid);
```

确认成员模式。

### 实验三：VACUUM FREEZE

结束所有事务后：

```sql
VACUUM (VERBOSE, FREEZE) mx_freeze_demo;

SELECT relminmxid, mxid_age(relminmxid)
FROM pg_class
WHERE oid = 'mx_freeze_demo'::regclass;
```

### 实验四：数据库级边界

```sql
SELECT datname, datminmxid, mxid_age(datminmxid)
FROM pg_database
ORDER BY mxid_age(datminmxid) DESC;
```

比较单表 `relminmxid` 和数据库 `datminmxid`。

### 实验五：源码跟读 freeze 决策

断点：

```text
FreezeMultiXactId
heap_prepare_freeze_tuple
```

观察一个 tuple 上的 MultiXact 被判为：

```text
FRM_INVALIDATE_XMAX
FRM_RETURN_IS_XID
FRM_RETURN_IS_MULTI
FRM_NOOP
```

画出决策树。

## 22. 讨论题

1. 为什么 `OldestMxact` 和 `MultiXactCutoff` 不是同一个值？

2. 为什么 `relminmxid` 不能越过表中仍保留的 MultiXact？

3. committed updater 和 ended locker 在 freeze 中有什么不同？

4. VACUUM 为什么有时要创建新的 MultiXact？

5. member 空间膨胀为什么会改变 effective multixact freeze age？

6. 为什么 `datminmxid` 比单表 `relminmxid` 更接近 SLRU 截断边界？

7. 如何区分普通 autovacuum 和 MultiXact anti-wraparound autovacuum？

8. 为什么 wraparound stop limit 要拒绝新的 MultiXactId 分配？

## 23. 本节小结

本节回答了 MultiXact 生命周期的最后一个问题：

```text
旧 MultiXactId 如何在不破坏 tuple 语义的前提下被 freeze、替换和最终截断？
```

核心链路是：

```text
vacuum_get_cutoffs()
  -> OldestMxact / MultiXactCutoff
  -> heap_prepare_freeze_tuple()
  -> FreezeMultiXactId()
  -> invalidate xmax / return XID / return Multi / noop
  -> 执行 freeze plan
  -> 推进 relminmxid
  -> 推进 datminmxid
  -> TruncateMultiXact()
```

核心状态是：

- `OldestMxact`。
- `MultiXactCutoff`。
- `relminmxid`。
- `datminmxid`。
- `OldestMemberMXactId`。
- `OldestVisibleMXactId`。
- MultiXact offsets SLRU。
- MultiXact members SLRU。
- `FRM_*` freeze flags。

ownership 和 cleanup 的边界是：

```text
tuple header 持有 MultiXact 引用；
VACUUM 负责消除或改写引用；
pg_class.relminmxid 记录表级最老引用；
pg_database.datminmxid 聚合数据库级边界；
checkpoint 后才能截断不再需要的 SLRU 段。
```

错误路径的核心规律是：

```text
任何 cutoff 推进都必须保守；
如果 tuple 中出现早于 relminmxid 的 MultiXact，或者 old Multi 仍 running，说明边界被破坏，必须报错而不是继续清理。
```

可观测入口包括：

- `pg_class.relminmxid`。
- `pg_database.datminmxid`。
- `mxid_age()`。
- `pg_get_multixact_stats()`。
- `VACUUM VERBOSE`。
- wraparound warning 日志。
- gdb 断点。

本节的可迁移模型是：

```text
任何可回卷 ID 一旦被持久化引用，就必须有两阶段回收协议：
先修改或删除所有引用者，
再推进全局最小保留边界并截断 ID 存储；
MultiXact freeze 正是这个协议在 row lock 集合上的实现。
```
