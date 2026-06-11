# PostgreSQL Correlation 与物理顺序成本影响

## 课程定位

前置知识：熟悉 index scan、heap fetch、random_page_cost、seq_page_cost，以及 `pg_stats.correlation` 的用户可见含义。

本节唯一主问题：

```text
为什么同样的选择率，在列值与 heap 物理顺序相关或无关时，index scan 成本会完全不同？
```

核心矛盾：planner 需要把逻辑过滤行数转换成物理 I/O 预期；但 catalog 里只有一个近似 correlation 系数，无法精确描述缓存、聚簇退化和多列索引访问模式。

学完后应能判断 correlation 何时会影响 index scan 成本，为什么 CLUSTER 后计划可能变化，以及为什么 rows 正确但计划仍可能慢。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释了列统计如何估算 rows；本节说明 rows 相同并不意味着访问成本相同。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节进入扩展统计对象，解决单列统计无法表达多列关系和表达式分布的问题。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`ANALYZE` 在标量值排序时计算列值顺序和样本 tuple 物理顺序的相关系数；btree cost estimate 把它转成 index order 与 heap order 的相关性，再由 `cost_index()` 在随机 I/O 与顺序 I/O 之间插值。
```

这句话要同时包含状态来源、判断时机和后续消费者。缺少其中任何一环，课程就会退化成函数名索引。

| 侧面 | 本节关注点 |
| --- | --- |
| 输入事实 | SQL 形态、catalog 统计、relids、操作符语义或样本分布。 |
| planner-local 状态 | 只在一次规划过程中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或局部统计结构上。 |
| 正确性边界 | 不能为了更低成本破坏 SQL 语义、权限边界、outer join null-extension 或统计含义。 |
| 性能收益 | rows 更接近现实、path 搜索更少走弯路、cost 比较更稳定。 |
| 诊断入口 | `EXPLAIN`、`pg_stats`、catalog 查询、gdb 断点和源码断点共同还原判断过程。 |

后文只沿这条链路展开：先看状态如何形成，再看它如何被消费、失效和观测。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/commands/analyze.c` | `compute_scalar_stats()` 生成 `STATISTIC_KIND_CORRELATION`。 |
| 2 | `src/include/catalog/pg_statistic.h` | `STATISTIC_KIND_CORRELATION` 定义 correlation slot。 |
| 3 | `src/backend/utils/adt/selfuncs.c` | `btcostestimate()`、`btcost_correlation()`、`genericcostestimate()` 读取 index correlation。 |
| 4 | `src/backend/optimizer/path/costsize.c` | `cost_index()`、`index_pages_fetched()` 用 correlation 影响 heap page cost。 |
| 5 | `src/backend/access/nbtree/nbtree.c` | btree AM 的 `amcostestimate` 指向 `btcostestimate()`。 |
| 6 | `src/include/nodes/pathnodes.h` | `IndexPath`、`IndexOptInfo` 保存索引路径和统计上下文。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：为什么同样的选择率，在列值与 heap 物理顺序相关或无关时，index scan 成本会完全不同？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

correlation 是普通列统计中最容易被误解的字段。

它不是 SQL 层的列相关性。

它描述的是某一列的排序顺序与 heap 物理 tuple 顺序之间的相关程度。

| 状态 | 语义边界 |
| --- | --- |
| `STATISTIC_KIND_CORRELATION` | `pg_statistic` 中的 slot kind，表示物理顺序与逻辑排序顺序的相关系数。 |
| `stanumbers[0]` | correlation 值，范围大致在 -1 到 1。 |
| `staop` | 计算 correlation 所用的 ordering operator。 |
| `indexCorrelation` | index AM cost estimate 返回给 `cost_index()` 的相关性输入。 |
| `csquared` | `cost_index()` 中对 correlation 平方后用于 I/O cost 插值。 |
| `tuples_fetched` | index selectivity 估算出的 heap tuple 数。 |
| `pages_fetched` | 基于 Mackert-Lohman 公式和 correlation 插值得到的 heap page 数。 |
| `random_page_cost` / `seq_page_cost` | tablespace 级或全局 page cost 参数。 |

写入端在 `compute_scalar_stats()`。

源码排序样本值时保留每个值的原始样本序号。

排序位置和原始序号越一致，correlation 越接近正相关。

如果 index 反向扫描，btree cost 读取端会把符号反过来。

如果多列 btree index 只从第一列拿 correlation，源码会对多列 index 做折扣。

读取端通常在 `src/backend/utils/adt/selfuncs.c` 的 btree cost 估算。

`btcost_correlation()` 调用 `get_attstatsslot()` 读取 `STATISTIC_KIND_CORRELATION`。

然后 index AM 的 `amcostestimate` 把 `indexCorrelation` 传给 `cost_index()`。

`cost_index()` 不把 correlation 当绝对耗时。

它用 `csquared = indexCorrelation * indexCorrelation` 在随机 I/O 成本和顺序 I/O 成本之间插值。

所以 -0.9 和 +0.9 在 heap page 局部性上接近，符号主要通过扫描方向在 index AM 层处理。

## 5. 从 SQL 现象进入源码

correlation 的外部现象通常是 index scan cost 的变化。

同样的 selectivity 下，CLUSTER 过的表或按 index key 物理有序的表，index scan 的 heap page I/O 估算更低。

随机插入或频繁更新后的表，correlation 下降，index scan 可能不再便宜。

一个实验路径是：

```sql
CREATE TABLE t AS
SELECT g AS id, g AS k, repeat('x', 50) AS pad
FROM generate_series(1, 200000) g;

CREATE INDEX ON t(k);
ANALYZE t;
SELECT correlation FROM pg_stats WHERE tablename = 't' AND attname = 'k';
EXPLAIN SELECT * FROM t WHERE k BETWEEN 1000 AND 20000;

CREATE TABLE t2 AS SELECT * FROM t ORDER BY random();
CREATE INDEX ON t2(k);
ANALYZE t2;
SELECT correlation FROM pg_stats WHERE tablename = 't2' AND attname = 'k';
EXPLAIN SELECT * FROM t2 WHERE k BETWEEN 1000 AND 20000;
```

两个表的逻辑分布相同。

差异主要来自 heap 物理顺序。

如果 selectivity、pages 和 cost 参数相同，correlation 会改变 index scan 与 seq scan 的相对 cost。

这类问题不能只看 `n_distinct` 或 MCV。

也不能把 `EXPLAIN (ANALYZE)` 的真实耗时直接倒推成 correlation。

缓存、visibility map、并发 I/O、预读和硬件都会影响真实耗时。

源码中 correlation 只是 cost model 的一个输入。

## 6. 主流程源码 walkthrough

写入链路仍然从 `compute_scalar_stats()` 开始。

第一步，样本值按列 ordering operator 排序。

每个 `ScalarItem` 保存 `tupno`，表示该值来自样本中的第几个 tuple。

第二步，源码在扫描排序结果时累加 `corr_xysum`。

这里的 x 是排序后位置，y 是原始样本位置。

如果样本物理顺序与值排序顺序接近，乘积和会表现出强相关。

第三步，在 values 数量大于 1 时生成 correlation slot。

源码用等差序列的和、平方和公式计算相关系数。

结果写入 `STATISTIC_KIND_CORRELATION`，`staop` 写排序 operator，`stanumbers[0]` 写系数。

第四步，planner 估算 btree index path。

`btcostestimate()` 会调用 `btcost_correlation()`。

`btcost_correlation()` 从变量统计中读取 correlation slot。

如果 index 是 reverse sort，会反转符号。

如果 index 有多个 key column，会把第一列 correlation 乘以 0.75，表达多列场景下物理局部性的保守折扣。

第五步，index AM cost 函数把 `indexSelectivity`、`indexCorrelation` 和 index pages 交给 `cost_index()`。

`cost_index()` 先用 selectivity 计算 `tuples_fetched`。

再用 `index_pages_fetched()` 估算随机访问情况下的 heap pages。

对于完全相关情况，它用 `ceil(indexSelectivity * baserel->pages)` 估算更接近顺序访问的 page 数。

第六步，源码计算两个 I/O 成本端点。

`max_IO_cost` 代表完全不相关时按随机访问收费。

`min_IO_cost` 代表完全相关时首 page 随机、后续 page 顺序收费。

第七步，用 correlation 平方插值：

```text
run_cost += max_IO_cost + csquared * (min_IO_cost - max_IO_cost)
```

这条公式解释了为什么 correlation 接近 0 时 index scan I/O cost 高，接近 ±1 时 I/O cost 低。

## 7. 生命周期 / ownership / cleanup

correlation 写入 catalog 的生命周期与普通列统计相同。

`ANALYZE` 期间，它先是 `VacAttrStats` 中的一个 `stanumbers` 指针。

`compute_scalar_stats()` 必须把这个 float4 分配在 `stats->anl_context`。

`update_attstats()` 后，它成为 `pg_statistic` 中某个 slot 的 float4 array。

planner 读取时通过 `get_attstatsslot()` 得到 `AttStatsSlot.numbers`。

`numbers` 指向 `numbers_arr` 的内部数据，释放时只 free `numbers_arr`。

`btcost_correlation()` 用完后立即调用 `free_attstatsslot()`。

`indexCorrelation` 只是 path costing 中的局部 double。

它不会进入 executor。

执行器不会根据 correlation 改变 index scan 行为。

最终计划只是选择了某条 path。

如果表在 analyze 后被大量重写、随机插入或 CLUSTER，catalog 中的 correlation 可能陈旧。

新的 `ANALYZE` 才能刷新这个统计事实。

## 8. 正确性机制层次

correlation 影响成本，不影响 SQL 正确性。

即使它完全错误，planner 也只是可能选择较慢 path，不会改变查询结果。

它的统计语义依赖排序 operator。

如果谓词或 index 使用的 ordering 与 slot 的 `staop` 不匹配，读取端不应直接使用。

它也依赖采样代表性。

小样本可能误判物理局部性。

但是 PostgreSQL 接受这种近似，因为全表精确计算 correlation 的成本太高。

它不是多列相关性。

`a` 和 `b` 在业务上相关，不会体现在单列 `a` 的 physical-order correlation 中。

多列 filter 相关性应看 extended statistics dependencies 或 multivariate MCV。

它也不是 cache hit ratio。

`cost_index()` 中的 page cost 仍由 `random_page_cost`、`seq_page_cost`、`effective_cache_size` 相关公式和 tablespace 参数共同决定。

correlation 只负责 heap fetch 局部性这一层。

## 9. 错误路径 / 异常路径 / fallback

没有 correlation slot 时，`btcost_correlation()` 返回 0。

这会让 `cost_index()` 更接近完全随机访问假设。

类型没有排序 operator时，不会生成普通 scalar correlation。

样本中可排序非 NULL 值不足两个时，也不会生成有效 correlation。

index AM 可以返回自己的 correlation 估算。

btree 是最常见阅读对象，但不要把 btree 细节推广到所有 AM。

index-only scan 会在 `cost_index()` 中用 all-visible fraction 减少 heap fetch 估算。

这意味着同样的 correlation，对 index scan 和 index-only scan 的成本影响不完全相同。

并行 index path 如果无法分配 worker，`cost_index()` 会提前返回，不继续计算完整成本。

真实执行中如果数据已经缓存，`random_page_cost` 与 `seq_page_cost` 的默认比例可能高估随机 I/O 代价。

这不是 correlation 字段错误，而是 cost 参数与硬件/workload 不匹配。

## 10. 成本、资源与跨模块传播

correlation 最直接传播到 `cost_index()`。

它不改变 index selectivity。

它改变的是访问命中 tuple 后，需要触碰多少 heap page，以及这些 page 更像随机读还是顺序读。

成本传播链路是：

```text
ANALYZE 写 correlation
  -> btree cost 读取 indexCorrelation
  -> cost_index 计算 heap I/O cost
  -> IndexPath total_cost 改变
  -> path 比较可能在 seq scan / bitmap scan / index scan 间切换
```

如果 rows 估算已经错，correlation 只能在错误的 `tuples_fetched` 基础上工作。

所以诊断 index plan 时，先看 selectivity，再看 correlation。

如果 selectivity 正确但 index scan 仍被估得太贵，再检查：

```text
pg_stats.correlation；
random_page_cost / seq_page_cost；
effective_cache_size；
visibility map 与 index-only scan；
表是否在 analyze 后物理顺序发生变化。
```

本节的可迁移规律是：

```text
成本模型里的统计字段通常只解释某一层资源假设；
不要把它扩展成业务相关性或真实性能的完整解释。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_stats.correlation` 能直接看到列值与物理顺序的近似关系。

`CLUSTER` 后执行 analyze，常能看到 correlation 接近正负一。

`EXPLAIN (ANALYZE, BUFFERS)` 中 heap block 命中和读取数量能检验成本假设。

rows 估算准确但 buffer 访问远超预期时，应检查 correlation、缓存和成本参数。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

创建按 id 顺序插入的表，analyze 后比较 id 条件的 index scan 成本。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

打乱同一张表的物理顺序后重新 analyze，比较 `pg_stats.correlation` 和计划变化。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

执行 `CLUSTER` 后再 analyze，观察普通 index scan 是否更容易被选择。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

改变 `random_page_cost`，验证 correlation 与成本参数共同影响路径选择。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

如果实验没有出现预期差异，先检查数据规模是否太小、统计是否刷新、GUC 是否强行禁用了候选路径。

## 14. 源码阅读练习

源码练习以断点和变量观察为主，不要求修改代码。

- 在 `src/backend/commands/analyze.c` 的入口函数附近设置断点，确认本节主路径是否进入。
- 在构造 planner-local 状态的位置打印关键字段，确认它们来自 SQL、catalog 还是样本。
- 在 fallback 分支设置断点，验证输入条件不满足时系统如何继续生成计划。
- 在选择率或 cost 消费点打印 rows/cost，观察误差从哪里开始放大。
- 在 `EXPLAIN` 结果中只解释能回到源码状态的差异，不把缓存波动当成 planner 结论。

先确认 SQL 现象，再回到 planner 中保存状态的位置。

先看状态字段由谁写入，再看后续阶段如何消费。

不要把单个布尔字段当成完整语义，必须连同 relids、生命周期和调用场景一起解释。

如果执行计划看起来反直觉，优先检查 rows、width、cost 和统计信息是否一致。

如果源码路径里出现 fallback，先问这个 fallback 是保护 correctness，还是保护 planner 可以继续给出计划。

如果一个判断依赖 catalog tuple，要区分 catalog 中保存的事实和 planner 从事实推导出的估算。

阅读源码时保留真实实现的历史痕迹，不要把多个分支整理成想象中的干净架构。

## 15. 常见误区

把 planner 的估算当成真实执行承诺。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

只看最终 plan，不找第一个 rows 或 cost 失真点。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

把 catalog 统计字段孤立解释，不看操作符、collation、relids 或生命周期。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

看到 fallback 就认为是 bug，而不是先确认前置条件是否满足。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

把提高 statistics target 当成所有慢 SQL 的通用修复。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

在没有复现实验的情况下，只凭函数名推断优化器行为。

修正方式是把判断链写完整：输入是什么，状态在哪里，源码如何分支，计划如何体现。

误区本身不是问题，问题是停在误区上继续调参。优化器诊断需要把每个猜测压回可验证状态。

## 16. 讨论题

为什么 correlation 影响成本而不是选择率？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

负相关为什么也可能对 index scan 有利？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

如果生产库缓存命中率很高，catalog correlation 和 cost 参数哪个更需要校准？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

- 哪些结论是当前源码版本的实现细节？
- 哪些结论是 SQL 语义或 catalog 语义带来的长期边界？
- 哪些判断会受数据分布、硬件、缓存或 GUC 影响？
- 如果要给生产库建议，需要先收集哪些证据？

## 17. 源码断点矩阵

这一节课到这里已经有了主流程，但真正做内核诊断时，还需要知道断点应该落在哪里。断点矩阵不是为了覆盖所有函数，而是为了让 SQL 现象能回到一个有限的状态集合。

| 断点层次 | 目的 | 退出条件 |
| --- | --- | --- |
| 入口断点 | 确认本次 SQL 是否进入本节主路径。 | 能看到目标 relation、clause、统计对象或 cost path。 |
| 状态写入断点 | 确认关键字段第一次被赋值的位置。 | 能说明字段来自 SQL、catalog、样本还是默认值。 |
| 判断分支断点 | 确认为什么选择精确路径或 fallback。 | 能列出至少一个导致分支的布尔条件。 |
| 消费断点 | 确认 rows、cost、path 或 plan 节点如何使用前面状态。 | 能把状态变化映射到 `EXPLAIN` 中的一个可见差异。 |

断点入口 1：`src/backend/commands/analyze.c`。`compute_scalar_stats()` 生成 `STATISTIC_KIND_CORRELATION`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/include/catalog/pg_statistic.h`。`STATISTIC_KIND_CORRELATION` 定义 correlation slot。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/backend/utils/adt/selfuncs.c`。`btcostestimate()`、`btcost_correlation()`、`genericcostestimate()` 读取 index correlation。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/backend/optimizer/path/costsize.c`。`cost_index()`、`index_pages_fetched()` 用 correlation 影响 heap page cost。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/access/nbtree/nbtree.c`。btree AM 的 `amcostestimate` 指向 `btcostestimate()`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/include/nodes/pathnodes.h`。`IndexPath`、`IndexOptInfo` 保存索引路径和统计上下文。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

## 18. 状态到计划的反向诊断

正向读源码是从入口走向 plan；排查慢 SQL 时经常要反过来，从 `EXPLAIN` 的异常节点倒推哪个状态不可信。

| 反向线索 | 优先怀疑 |
| --- | --- |
| 叶子 scan rows 已经偏差很大 | 单列统计、表达式统计、predicate 分类或统计陈旧。 |
| join rows 在第一个 join 后突然放大 | join selectivity、列相关性、join order 约束或外连接语义。 |
| rows 近似正确但 scan 成本离谱 | correlation、成本参数、缓存状态或 page 估算模型。 |
| 计划缺少预期 index path | qual 形态、操作符族、collation、下推安全性或 parameterized path。 |
| 扩展统计没有生效 | 统计对象未 analyze、clause 不兼容、表达式树不匹配或 inherit 标记不一致。 |

反向检查 `STATISTIC_KIND_CORRELATION`：列值排序顺序与 tuple 物理顺序的相关系数。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `indexCorrelation`：index AM cost estimate 交给 `cost_index()` 的相关性。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `tuples_fetched`：index 条件预计返回的 heap tuple 数。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pages_fetched`：根据缓存模型估算的 heap page 访问量。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `max_IO_cost`：完全随机访问时的 heap I/O 成本上界。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `min_IO_cost`：完全相关访问时的 heap I/O 成本下界。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `csquared`：`cost_index()` 使用 correlation 平方做插值权重。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `random_page_cost` / `seq_page_cost`：把 page 访问模式转换成相对成本单位。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`compute_scalar_stats()` 把样本值排序，同时保留样本 tuple 的原始序号。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：源码累积排序位置和原始位置的相关性，生成单个 float4 correlation。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：`update_attstats()` 把 correlation 写入 `pg_statistic` 的 correlation slot。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：规划 index path 时，btree AM 调用 `btcostestimate()`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`btcost_correlation()` 读取 index leading column 对应列统计。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：多列索引场景下，后续列相关性会被折减，不会被当作完整 heap order 描述。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：`genericcostestimate()` 估算 index 自身访问成本和选择率。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：`cost_index()` 用 `index_pages_fetched()` 估算随机访问页数，再按 correlation 插值 heap I/O。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 9：最终 path 的 startup/total cost 影响 seq scan、bitmap scan 和 index scan 的竞争。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

## 20. 复盘案例：从一个估算偏差回到源码

课堂复盘不要求固定数据集，而是要求路径完整。下面四类案例可以套用到本节不同主题。

案例：估算明显偏小。常见原因是多个条件被当成独立、热点值没有进入 MCV、predicate 没有在预期层级执行，或者扩展统计没有命中。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

案例：估算明显偏大。常见原因是 histogram 太粗、ndistinct 过低、NULL 比例变化、过滤条件被延迟到更高层级，或者 fallback 选择率过于保守。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

案例：rows 接近但 plan 仍慢。常见原因是物理访问模型、correlation、缓存、并行成本、work_mem 或上层节点资源消耗没有被真实反映。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

案例：创建对象后计划不变。常见原因是没有重新 analyze、表达式树不匹配、统计对象 kind 不适用于当前 clause，或者 planner 已经由更强信息完成估算。

- 先保存 `EXPLAIN (ANALYZE, BUFFERS)`，标记第一个估算失真节点。
- 查询 `pg_stats`、`pg_statistic` 或扩展统计视图，确认统计是否存在且新鲜。
- 在本节源码入口和消费点各设一个断点，确认状态是否被写入和读取。
- 只做一个最小变更，再观察 plan 是否朝预期方向移动。

复盘结论应该写成因果链，而不是写成单句调参建议。

## 21. 版本边界与源码阅读陷阱

本课基于指定的本地源码提交。优化器代码会持续演进，因此课程强调的是当前源码可验证的不变量，而不是把某个局部分支当作永久接口。

注释里提到的历史限制可能已经被附近代码部分修正，读注释时要结合当前调用者。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

同名字段在不同上下文中的解释可能不同，尤其是 relids、inherit、kind、op 和 collation。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

planner 里的默认估算不是错误处理，而是缺少精确信息时继续规划的机制。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

EXPLAIN 不显示大多数中间候选 path，不能只凭最终 plan 判断搜索空间中发生了什么。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

测试数据太小会让 seq scan、index scan、join path 的成本差异被启动成本淹没。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

缓存状态和硬件延迟会影响实际耗时，但不会反向改变 planner 已经做出的估算。

把这类问题写入诊断报告时，要说明它影响的是源码判断、统计输入、成本模型还是运行时环境。

## 22. 课堂练习检查清单

完成本节练习时，可以用下面的清单检查材料是否闭环。

- 是否明确本节唯一主问题，没有把相邻主题混成一个大问题。
- 是否给出至少一个 SQL 或 catalog 现象，并能回到源码解释。
- 是否指出状态由谁创建、谁读取、何时失效。
- 是否区分 correctness fallback 和 cost fallback。
- 是否说明哪些结论依赖数据分布、统计新鲜度、GUC 或硬件。
- 是否能在计划中指出一个可观察结果，而不是只描述内部函数。
- 是否避免把默认估算、保守路径或未命中扩展统计直接称为 bug。
- 是否能给出下一步验证实验，而不是直接给生产修复建议。

如果清单中有任一项无法回答，说明还需要回到源码入口或实验设计补证据。

## 23. 实验记录模板

这一节的实验记录建议固定成同一种格式，便于后续课程继续复用。

记录对象：PostgreSQL Correlation 与物理顺序成本影响。

记录第一项：SQL 形态。

写清楚谓词、统计对象、索引、数据分布和相关 GUC，不要只贴最终计划。

记录第二项：统计状态。

至少保存 `pg_stats` 或相关 catalog 查询结果，并注明 analyze 发生在实验前还是实验后。

记录第三项：估算位置。

在计划树中标出第一个 estimated rows 与 actual rows 明显分离的节点。

记录第四项：源码入口。

把本节第 3 节中的入口函数映射到这次实验，不要临时跳到无关模块。

记录第五项：状态字段。

只记录能解释本节主问题的字段，避免把 gdb 输出变成无边界日志。

记录第六项：变更动作。

一次只改变一个因素：统计目标、数据分布、SQL 写法、统计对象或成本参数。

记录第七项：计划变化。

说明变化体现在 rows、cost、Filter 位置、path 类型还是最终节点选择。

记录第八项：结论边界。

把结论限定在当前数据分布和源码版本内；不能从一个小表实验直接推出生产库规则。

这份记录模板的目的，是把 runtime 现象压回源码可验证的状态链。

## 24. 课后源码索引

课后复习 PostgreSQL Correlation 与物理顺序成本影响 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/commands/analyze.c`：`compute_scalar_stats()` 生成 `STATISTIC_KIND_CORRELATION`。

- `src/include/catalog/pg_statistic.h`：`STATISTIC_KIND_CORRELATION` 定义 correlation slot。

- `src/backend/utils/adt/selfuncs.c`：`btcostestimate()`、`btcost_correlation()`、`genericcostestimate()` 读取 index correlation。

- `src/backend/optimizer/path/costsize.c`：`cost_index()`、`index_pages_fetched()` 用 correlation 影响 heap page cost。

- `src/backend/access/nbtree/nbtree.c`：btree AM 的 `amcostestimate` 指向 `btcostestimate()`。

- `src/include/nodes/pathnodes.h`：`IndexPath`、`IndexOptInfo` 保存索引路径和统计上下文。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

rows 是逻辑数量，correlation 把逻辑数量连接到 heap 物理访问模式。

`cost_index()` 使用相关性在随机 I/O 与顺序 I/O 之间插值。

调试 index scan 选择时，要同时看选择率、correlation、成本参数和实际 buffer 访问。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
