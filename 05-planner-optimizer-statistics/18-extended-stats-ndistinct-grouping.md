# PostgreSQL Multivariate ndistinct 与 grouping rows 估算

## 课程定位

前置知识：已经理解单列 `n_distinct`、`ANALYZE` 采样、扩展统计对象定义、`RelOptInfo.rows` 如何传播到 path cost，以及上节 MCV 如何修正 restriction selectivity。
本节唯一主问题可以写成这个判断：GROUP BY 或 DISTINCT 同时使用多个相关表达式时，planner 为什么不能简单相乘单列 ndistinct，又如何用 multivariate ndistinct 给出可用的 group count。
核心矛盾：上层 path 需要一个具体组数来估计 HashAgg 表大小、Sort 成本、Unique 输出行数和 Hash Join bucket size；但真实组合数依赖列间相关性，有限样本只能保存少数已声明组合的估计。
学完后应能判断：一个 HashAggregate、GroupAggregate、Unique 或 DISTINCT 的 rows 偏差，是来自没有 ndistinct 扩展统计、统计对象未覆盖当前 grouping expression、过滤缩放误差，还是 group count 被后续成本模型放大。
本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节的 multivariate MCV 服务 WHERE clause selectivity。
本节的 multivariate ndistinct 服务 group count。
两者都属于 extended statistics，但消费位置完全不同。
MCV 进入 `clauselist_selectivity_ext()`，影响 base relation 的 restriction rows。
ndistinct 进入 `estimate_num_groups()`，影响 upper planning 中 grouping、distinct、unique 和部分 hash bucket 估算。
因此本节不再围绕某个常量组合是否命中。
本节围绕一组表达式会产生多少不同组合展开。
比如 `GROUP BY country, city` 中，单列 distinct 的乘积可能远大于真实城市组合数。
如果 planner 高估组数，HashAgg 可能被估成更大内存需求，Sort 或 GroupAggregate 可能被错误偏好。
如果 planner 低估组数，HashAgg spill、batch 数和 CPU 成本都可能被低估。
本节只追“组数估算如何进入路径成本”这一条主线。
它不展开 aggregate executor 的 hash table 细节，也不讲 grouping sets 的全部规划算法。

本节和上一节的边界可以这样区分：

- MCV 处理的是布尔条件命中多少行，ndistinct 处理的是输入行会被压成多少组。
- MCV 保存具体值组合和频率，ndistinct 保存属性组合的 distinct 数估计。
- MCV 主要影响 `RelOptInfo.rows`，ndistinct 主要影响 upper path 的 `numGroups` 和 output rows。
- MCV 的诊断重点是 `frequency` 与 `base_frequency`，ndistinct 的诊断重点是组合 item 是否匹配当前 grouping vars。
- MCV 未覆盖时回到普通 clause selectivity，ndistinct 未覆盖时回到单列 distinct 乘法加 clamp 的启发式。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`ANALYZE` 为统计对象中的每个二列及以上组合构建 `MVNDistinctItem`，planner 在 `estimate_num_groups()` 中优先匹配同一 relation 上覆盖最多 grouping vars 的 ndistinct 统计，匹配失败时回到单列 distinct 乘法和行数 clamp。
这个模型解决的不是执行期分组。
executor 仍然会真实构建 HashAgg 或 Sort/Group 的结果。
planner 需要的是执行前的预估组数。
这个组数会决定 `AggPath.path.rows`、`AggPath.numGroups`、hash aggregate 预估表大小、sort 输入输出成本和 unique path 的 rows。
源码里最关键的函数是 `estimate_num_groups()`。
它先把 group expressions 转成 `GroupVarInfo`，再按 base relation 分组。
对同一 relation 内的多个变量，它会循环调用 `estimate_multivariate_ndistinct()`。
如果找到匹配的 `MVNDistinctItem`，就把这个组合的 `ndistinct` 乘进 relation 级 group count，并从待估算变量列表中移除已匹配变量。
如果找不到，剩余变量的单列 distinct 会被相乘。
随后源码用 relation tuples、relation rows 和输入行数做一系列 clamp。
这说明 ndistinct 不是简单替代整个估算公式，而是替代同一 relation 上一组变量的组合 distinct 输入。

| 估算层次 | 源码入口 | 本节解释的语义 |
| --- | --- | --- |
| 统计对象定义 | `CreateStatistics()` | 声明对象覆盖哪些列、表达式和 ndistinct kind。 |
| 样本构建 | `statext_ndistinct_build()` | 为所有二列及以上组合构建 `MVNDistinctItem`。 |
| 组合估算 | `ndistinct_for_combination()` | 对样本中的组合 distinct 数外推到全表。 |
| 目录存储 | `statext_store()` | 把 `MVNDistinct` 序列化到 `stxdndistinct`。 |
| planner 消费 | `estimate_num_groups()` | 将 grouping expressions 压缩成可用于 path 的组数。 |
| 统计匹配 | `estimate_multivariate_ndistinct()` | 选择匹配最多 vars 或 expressions 的 ndistinct 对象。 |
| 路径构造 | `create_agg_path()` | 把 `numGroups` 放入 `AggPath` 并调用 `cost_agg()`。 |
| 成本传播 | `cost_agg()` | 用 `numGroups` 估算输出行、CPU 成本和 HashAgg spill 风险。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/adt/selfuncs.c` | `estimate_num_groups()` 与 `estimate_multivariate_ndistinct()` 是 planner 消费主线。 |
| 2 | `src/backend/statistics/mvdistinct.c` | `statext_ndistinct_build()`、`ndistinct_for_combination()`、`estimate_ndistinct()` 是构建主线。 |
| 3 | `src/include/statistics/statistics.h` | `MVNDistinct` 和 `MVNDistinctItem` 描述组合属性与 distinct 估算。 |
| 4 | `src/backend/statistics/extended_stats.c` | `BuildRelationExtStatistics()` 调度 ndistinct 构建并通过 `statext_store()` 写目录。 |
| 5 | `src/backend/commands/statscmds.c` | `CreateStatistics()` 处理 ndistinct kind、列限制、表达式限制和依赖。 |
| 6 | `src/backend/optimizer/util/plancat.c` | `get_relation_statistics_worker()` 把 built ndistinct 变成 `StatisticExtInfo`。 |
| 7 | `src/backend/optimizer/plan/planner.c` | grouping、distinct、unique 等 upper planning 多处调用 `estimate_num_groups()`。 |
| 8 | `src/backend/optimizer/util/pathnode.c` | `create_agg_path()`、`create_group_path()`、`create_upper_unique_path()` 消费组数。 |
| 9 | `src/backend/optimizer/path/costsize.c` | `cost_agg()`、`cost_group()`、`estimate_hashagg_tablesize()` 把组数转成成本。 |
| 10 | `src/test/regress/sql/stats_ext.sql` | 回归测试里的 grouping rows 案例提供可复现实验骨架。 |

阅读顺序要从 `estimate_num_groups()` 开始。
因为构建出来的 `MVNDistinct` 只有在这里被转化成 planner 能消费的 group count。
如果先读 `mvdistinct.c`，容易把 Duj1 estimator 当成主问题。
本节真正的问题是：有限统计如何被嵌入一个包含表达式、等价类、restriction selectivity、clamp 和 SRF multiplier 的估算流程。

## 4. 从 HashAggregate rows 偏差切入

考虑一张地理维表，`country` 与 `city` 强相关。
`country` 有 200 个 distinct，`city` 有 10000 个 distinct。
如果简单相乘，`GROUP BY country, city` 会得到 2000000 个组合。
真实组合可能只有 10000 左右，因为一个城市通常只属于一个国家。
这个误差会直接进入 `HashAggregate` 的 estimated rows。
如果组数高估，planner 可能认为 hash table 很大，并提高 spill 或 sort 路径的相对吸引力。
如果组数低估，planner 可能选择 HashAgg，但执行时发现 batch 和磁盘 I/O 远高于预期。
multivariate ndistinct 的目标就是在已声明统计对象覆盖的组合上避免这种单列乘法误差。
它不会告诉 planner 每个组的频率。
它也不会告诉 planner 某个具体城市出现多少行。
它只告诉 planner：这一组 grouping dimensions 预计会产生多少不同组合。

把这个现场回到源码时，推荐按下面顺序定位：

1. 确认 `EXPLAIN` 中偏差发生在 Aggregate、Group、Unique 或 DISTINCT 上层节点。
2. 确认 grouping expressions 是否来自同一个 base relation，因为多变量 ndistinct 按 relation 匹配。
3. 确认 `CREATE STATISTICS` 指定了 `ndistinct`，或者默认 kind 包含 ndistinct。
4. 确认 `ANALYZE` 后 `pg_statistic_ext_data.stxdndistinct` 已经存在。
5. 确认 `RelOptInfo.statlist` 中存在 `STATS_EXT_NDISTINCT` kind 且 inherit 标志匹配。
6. 确认 `estimate_multivariate_ndistinct()` 能在 `MVNDistinct.items` 中找到当前组合。
7. 确认找到的 `ndistinct` 如何进入 relation-level `reldistinct`。
8. 确认 `estimate_num_groups()` 后续 clamp 是否又改变了这个估算。

## 5. 关键数据结构与状态边界

`MVNDistinct` 定义在 `src/include/statistics/statistics.h`。
它包含 magic、type、item 数以及一个 `MVNDistinctItem` 数组。
每个 `MVNDistinctItem` 保存 `ndistinct`、`nattributes` 和 `attributes`。
它不保存具体值，也不保存频率。
这一点和 MCV 完全不同。
MCV 的 item 能回答“这个具体组合有多常见”。
ndistinct 的 item 只能回答“这组维度有多少不同组合”。
当统计对象覆盖三列时，`statext_ndistinct_build()` 会为二列组合和三列组合都生成 item。
例如 `(a,b,c)` 会产生 `(a,b)`、`(a,c)`、`(b,c)` 和 `(a,b,c)`。
这解释了为什么消费阶段必须在 `stats->items` 中寻找精确匹配的组合。
`GroupVarInfo` 是 `estimate_num_groups()` 内部使用的 planner-local 结构。
它把一个 grouping expression、所属 relation、单列 distinct 估计和 default 标记放在一起。
`estimate_multivariate_ndistinct()` 接收 `List **varinfos`，匹配成功后会构造一个移除已匹配变量的新 list。
这个 ownership 很关键。
匹配成功的变量不应该再进入单列乘法，否则会重复计数。
`StatisticExtInfo` 仍然只是 planner 对统计对象的索引。
它告诉估算器哪一个对象可能包含 ndistinct kind，真正的 `MVNDistinct` 仍然通过 `statext_ndistinct_load()` 从 catalog 读取。

| 状态 | 生命周期 | 诊断意义 |
| --- | --- | --- |
| `pg_statistic_ext.stxkind` | 统计对象定义存在期间 | 说明对象是否声明或默认包含 ndistinct。 |
| `pg_statistic_ext_data.stxdndistinct` | 每次 `ANALYZE` 重建并替换 | 说明组合 distinct 数据是否已构建。 |
| `MVNDistinctItem` | 构建后序列化，消费时反序列化 | 保存某个属性或表达式组合的 distinct 估算。 |
| `GroupVarInfo` | 一次 `estimate_num_groups()` 调用期间 | 承载 grouping expression 与单列 distinct 估算。 |
| `RelOptInfo.statlist` | 当前 planner run 内 | 让 `estimate_multivariate_ndistinct()` 找到可用统计对象。 |
| `numGroups` | upper path 构造期间 | 进入 Agg、Group、Unique、SetOp 和 HashAgg 成本。 |

## 6. 构建阶段源码 walkthrough

ndistinct 的构建与 MCV 共享 `BuildRelationExtStatistics()` 入口。
`CreateStatistics()` 首先决定是否构建 ndistinct kind。
用户显式写 `(ndistinct)` 时，`build_ndistinct` 置为 true。
用户没有指定 kind 且对象覆盖两个或更多列或表达式时，源码默认同时构建 ndistinct、dependencies 和 MCV。
对象定义写入 `pg_statistic_ext`，真实数据等到 `ANALYZE` 时构建。
`BuildRelationExtStatistics()` 获取对象列表后，为每个统计对象准备 `StatsBuildData`。
如果当前 analyze 没有覆盖对象需要的列，构建会跳过。
如果 target 为 0，构建会跳过并保留旧统计数据。
对 ndistinct kind，`BuildRelationExtStatistics()` 调用 `statext_ndistinct_build(totalrows, data)`。
`statext_ndistinct_build()` 计算统计对象维度数，然后调用 `num_combinations(numattrs)` 预留所有组合 item 的空间。
它从 k 等于 2 开始，一直生成到全部维度数。
每个 k 通过 `generator_init()` 和 `generator_next()` 枚举组合。
对每个组合，源码把组合下标翻译成真实 attnum，写入 `MVNDistinctItem.attributes`。
真正的 distinct 外推发生在 `ndistinct_for_combination()`。
它为当前组合创建一个 k 维 `MultiSortSupport`。
它把样本中的这些维度值复制进临时 `SortItem` 数组。
它用 `qsort_interruptible()` 对样本组合排序。
排序后，源码扫描相邻组合，统计样本中 distinct 组合数 `d` 和只出现一次的组合数 `f1`。
最后 `estimate_ndistinct(totalrows, numrows, d, f1)` 使用与 `analyze.c` 单列统计相同的 Duj1 estimator 外推全表 distinct 数。
估计值会被 clamp 到至少 `d`，至多 `totalrows`，并四舍五入到整数。
构建出的 `MVNDistinct` 由 `statext_ndistinct_serialize()` 写成 varlena。
`statext_store()` 把它放入 `pg_statistic_ext_data.stxdndistinct`。

构建阶段可以按下面的时间顺序阅读：

1. `CreateStatistics()` 确定 ndistinct kind 是否写入 `stxkind`。
2. `ANALYZE` 准备样本和普通列统计入口。
3. `BuildRelationExtStatistics()` 选择当前 relation 可构建的统计对象。
4. `make_build_data()` 整理样本值、NULL 标记和属性编号。
5. `statext_ndistinct_build()` 预留所有二列及以上组合的 item。
6. `generator_next()` 枚举当前 k 值对应的属性组合。
7. `ndistinct_for_combination()` 对样本组合排序并统计 `d` 与 `f1`。
8. `estimate_ndistinct()` 用 Duj1 estimator 外推到全表。
9. `statext_ndistinct_serialize()` 保存 magic、type、item 数、ndistinct 和属性编号。
10. `statext_store()` 替换 `pg_statistic_ext_data.stxdndistinct`。

## 7. `estimate_num_groups()` 消费阶段 walkthrough

消费阶段从 `src/backend/utils/adt/selfuncs.c` 的 `estimate_num_groups()` 开始。
这个函数不仅服务 GROUP BY，也服务 DISTINCT 和一些 unique 过滤场景。
它的输入是 grouping expressions、输入行数、可选 grouping set 过滤器和 estimation info。
函数首先把 `input_rows` 通过 `clamp_row_est()` 保护到合理范围。
如果没有 grouping columns，它直接返回 1.0。
随后它遍历每个 group expression。
布尔表达式被当成最多两个组，并从后续变量分析中排除。
如果表达式能被 `examine_variable()` 识别为一个有统计信息的变量或表达式，就通过 `add_unique_group_var()` 加入 `varinfos`。
如果不能识别，就用 `pull_var_clause()` 抽取表达式内的 Vars。
如果表达式没有变量但含 volatile function，函数直接假设每个输入行都可能产生不同组，返回 `input_rows`。
收集完 `varinfos` 后，函数按 relation 分组处理。
每次取一个 relation 的所有 `GroupVarInfo`，形成 `relvarinfos`。
在同一个 relation 内，源码优先调用 `estimate_multivariate_ndistinct()`。
如果匹配成功，返回的 `mvndistinct` 乘入 `reldistinct`，并从 `relvarinfos` 中移除已匹配变量。
如果匹配失败，剩余变量按单列 distinct 相乘。
当同一 relation 上有多个独立组合时，源码会循环尝试 multivariate 匹配，并把多个结果相乘。
这说明 ndistinct 统计对象不是全局唯一答案，而是 relation 内若干变量组的估算片段。
之后源码对 `reldistinct` 做 clamp。
如果同一 relation 上有多个变量，clamp 初值是 `rel->tuples * 0.1`，但不会低于最大的单个 distinct 估计。
然后源码根据 restriction selectivity 调整 distinct 数，使用 Yao 公式近似过滤后仍能看到多少 distinct 值。
最后所有 relation 的 group count 相乘，再乘上 SRF multiplier，向上取整，并限制在 `[1, input_rows]`。

`estimate_num_groups()` 的主链路可以压缩成下面十二步：

1. 保护 `input_rows`，避免返回零组导致后续除零或成本异常。
2. 处理没有 grouping columns 的特殊情况，直接返回一个组。
3. 把 boolean grouping expression 计为两个可能组。
4. 用 `examine_variable()` 尝试把表达式识别成有统计信息的变量或表达式。
5. 无法识别时用 `pull_var_clause()` 抽出表达式内部 Vars。
6. 遇到 variable-free volatile expression 时保守返回 `input_rows`。
7. 用 `add_unique_group_var()` 去重，并处理等价类中已知相等的变量。
8. 按 relation 把 `GroupVarInfo` 分组处理。
9. 对每个 relation 循环调用 `estimate_multivariate_ndistinct()`。
10. 匹配失败的剩余变量退回单列 distinct 乘法。
11. 用 relation tuples、rows 和 restriction selectivity 调整 relation 级 distinct。
12. 把所有 relation 结果相乘，应用 SRF multiplier，并 clamp 到输入行数范围内。

## 8. `estimate_multivariate_ndistinct()` 匹配细节

`estimate_multivariate_ndistinct()` 是本节最关键的匹配函数。
它首先检查 `rel->statlist` 是否为空。
如果 relation 没有扩展统计，函数立即返回 false。
它遍历 `rel->statlist`，只考虑 kind 等于 `STATS_EXT_NDISTINCT` 的 `StatisticExtInfo`。
它还要求统计对象的 inherit 标志匹配当前 RTE。
对每个候选对象，它统计当前 `varinfos` 中有多少简单 Var 命中 `info->keys`，有多少表达式命中 `info->exprs`。
匹配总数少于二的对象会被跳过。
当多个对象都可用时，源码选择表达式匹配更多的对象；表达式匹配数相等时，选择变量匹配更多的对象。
`rel->statlist` 按统计对象 OID 排序，因此同等匹配下选择是确定的。
找到候选对象后，函数调用 `statext_ndistinct_load(statOid, rte->inh)` 读取并反序列化 `MVNDistinct`。
随后它构造 matched bitmap。
普通 Var 用 attnum 匹配 `StatisticExtInfo.keys`。
表达式通过 `equal()` 与 `StatisticExtInfo.exprs` 中的表达式树匹配。
如果统计对象包含表达式，源码会对 attnum 做 offset，避免表达式编号与普通列编号冲突。
最后函数扫描 `stats->items`，寻找 attributes 集合与 matched bitmap 对应的 item。
找到后，它把 item 的 `ndistinct` 写入输出参数，并构造一个移除已匹配 variables 的新 `varinfos` list。
这个“移除已匹配变量”的动作是防止重复估算的核心。
如果找不到精确 item，函数返回 false，调用者会使用单列 distinct fallback。

| 匹配条件 | 源码表现 | 失败后的行为 |
| --- | --- | --- |
| 没有 statlist | `rel->statlist` 为空 | 直接返回 false。 |
| kind 不匹配 | `info->kind != STATS_EXT_NDISTINCT` | 跳过该统计对象。 |
| inherit 不匹配 | `info->inherit != rte->inh` | 跳过该统计对象。 |
| 匹配维度不足 | 变量和表达式命中总数小于二 | 跳过该统计对象。 |
| 表达式不等价 | `equal(varinfo->var, expr)` 不成立 | 该表达式不算命中。 |
| 没有 item | `MVNDistinct.items` 中无精确组合 | 返回 false 并退回单列估算。 |

## 9. 生命周期 / ownership / cleanup

ndistinct 的生命周期也分为对象定义、统计数据、构建临时状态和 planner-local 消费状态。
对象定义由 `pg_statistic_ext` 持有。
`CreateStatistics()` 记录列依赖和表达式依赖，因此列或表删除时统计对象会被自动清理。
统计数据由 `pg_statistic_ext_data.stxdndistinct` 持有。
它不是实时维护的物化视图，而是上一次 `ANALYZE` 的采样外推结果。
构建临时状态属于 `BuildRelationExtStatistics` memory context。
`statext_ndistinct_build()` 为所有组合分配 `MVNDistinctItem` 和 attributes 数组。
`ndistinct_for_combination()` 为每个组合分配排序所需的 `SortItem`、values 和 nulls 数组。
这些内存在对象构建结束后通过 context reset 回收。
反序列化出的 `MVNDistinct` 由 `statext_ndistinct_load()` 分配在当前 backend memory context 中。
`statext_ndistinct_free()` 可以释放其中每个 item 的 attributes 数组和整体对象。
planner 估算状态如 `GroupVarInfo`、`varinfos` 和 `newvarinfos` 属于当前 planner run。
`numGroups` 一旦写入 path，就成为该 path 成本的一部分。
executor 不会重新读取 `pg_statistic_ext_data` 来修正这个估算。

| 对象 | owner | 释放或失效边界 |
| --- | --- | --- |
| 统计对象定义 | `pg_statistic_ext` | `DROP STATISTICS`、依赖对象删除或事务回滚。 |
| ndistinct 数据 | `pg_statistic_ext_data` | 下一次 `ANALYZE` 替换或统计对象删除。 |
| 构建临时数组 | `BuildRelationExtStatistics` context | 每个统计对象后 reset。 |
| 反序列化对象 | 当前 backend memory context | 估算结束或上层 context reset。 |
| `GroupVarInfo` 列表 | `estimate_num_groups()` 调用路径 | 当前 planner run 结束后释放。 |
| `numGroups` path 字段 | 具体 `Path` 节点 | path 被丢弃或 planner run 结束。 |

## 10. 正确性机制与 fallback

ndistinct 估算错误不会改变 SQL 结果，只会改变计划选择和成本预测。
因此这里的正确性边界是“不能让缺失统计造成重复计数、无界估算或 planner 崩溃”。
第一个 fallback 是没有扩展统计。
`estimate_multivariate_ndistinct()` 返回 false，`estimate_num_groups()` 使用单列 distinct 估算。
第二个 fallback 是统计对象未构建。
`get_relation_statistics_worker()` 只有在 `pg_statistic_ext_data` 中对应 kind 存在时才创建 `StatisticExtInfo`。
第三个 fallback 是匹配不足两个维度。
多变量 ndistinct 不服务单个普通列，因为单列统计已经在 `pg_statistic` 中。
第四个 fallback 是 volatile expression。
如果 group expression 不含变量但含 volatile function，`estimate_num_groups()` 保守返回 `input_rows`。
第五个 fallback 是多个 relation。
`estimate_num_groups()` 对每个 relation 分别估算，再把 relation 级结果相乘；join clauses 默认不减少 group count。
第六个 fallback 是 filtering 缩放。
即使有 multivariate ndistinct，`estimate_num_groups()` 仍要用 relation rows 与 tuples 估计过滤后还能看到多少 distinct。
第七个 fallback 是 clamp。
最终结果不会小于 1，也不会大于 `input_rows`。

这些 fallback 是诊断时必须分开的层次：

- 对象缺失说明 schema 没有提供多变量信息。
- 数据缺失说明对象存在但 `ANALYZE` 没有产生可用数据。
- 匹配失败说明 grouping expression 与统计对象定义不一致。
- 单列 fallback 说明 planner 仍能继续规划，但相关性误差会回到乘法假设。
- clamp 改写说明即使匹配成功，最终 rows 也可能被 input rows 或 relation rows 边界限制。

## 11. 成本、资源与跨模块传播

ndistinct 的直接输出是 group count。
这个数字在源码里以不同字段名传播。
`estimate_num_groups()` 返回 double。
`planner.c` 在 grouping、distinct 和 unique planning 中多次调用它。
`pathnode.c` 的 `create_agg_path()` 把 `numGroups` 写入 `AggPath.numGroups`，并把 output rows 设置为组数。
`costsize.c` 的 `cost_agg()` 用 `numGroups` 估算 final function 成本、输出 tuple 成本和 HashAgg 的 spill 风险。
`estimate_hashagg_tablesize()` 用 group count 乘以 hash entry size，得到 hash aggregate 表大小估计。
`cost_group()` 和 sort 相关路径也会受到 group count 影响。
`estimate_multivariate_bucketsize()` 还把 multivariate ndistinct 用于 hash join inner side 多列 hash key 的 bucket size 估算。
这条相邻路径说明 ndistinct 不只服务 SQL 层的 GROUP BY。
它本质上服务“多列 key 会产生多少不同值”的成本问题。
构建成本方面，`statext_ndistinct_build()` 需要为统计对象中所有二列及以上组合做样本排序和 distinct 计数。
维度越多，需要构建和匹配的组合数就越多。
`STATS_MAX_DIMENSIONS` 限制扩展统计最多八个维度，这既是语义边界，也是成本边界。
过多组合会增加 analyze CPU、内存和 catalog 存储成本。

| 传播位置 | 字段或函数 | 错误后果 |
| --- | --- | --- |
| grouping rows | `estimate_num_groups()` | Aggregate 或 DISTINCT 节点 rows 估错。 |
| AggPath 构造 | `create_agg_path()` | `path.rows` 和 `numGroups` 被写入 path。 |
| HashAgg 成本 | `cost_agg()` | 内存、batch、spill 和 CPU 成本估错。 |
| hash 表大小 | `estimate_hashagg_tablesize()` | hash aggregate 空间估计偏大或偏小。 |
| Group path 成本 | `cost_group()` | GroupAggregate 路径比较受影响。 |
| Hash Join bucket | `estimate_multivariate_bucketsize()` | 多列 hash key 的 bucket size 估计受影响。 |

## 12. 观测与诊断入口

SQL 层可以通过 `pg_stats_ext` 观察 ndistinct 摘要。
也可以直接查看 `pg_statistic_ext_data.stxdndistinct` 的文本输出，因为 `pg_ndistinct` 有输出函数。
这些输出可以证明统计数据存在，并显示组合属性与估算值。
但它们不能直接证明某条 query 使用了对应 item。
要证明使用路径，需要结合 `EXPLAIN` 中 Aggregate、Unique 或 DISTINCT 节点的 estimated rows。
推荐比较三组计划来隔离 ndistinct 的影响。
第一组没有扩展统计，只执行普通 analyze。
第二组创建 `(ndistinct)` 统计并 analyze。
第三组改变 grouping expression，让它不再匹配统计对象。
如果第二组 rows 改善而第三组回退，说明 multivariate ndistinct 很可能介入。
源码级诊断可以在 `estimate_num_groups()` 和 `estimate_multivariate_ndistinct()` 设断点。
最有价值的观察点是 `varinfos`、`relvarinfos`、候选 `StatisticExtInfo`、matched bitmap、返回的 `mvndistinct` 和最终 clamp 后的 `numdistinct`。
还要观察 `rel->rows` 与 `rel->tuples`。
因为即使 `MVNDistinctItem.ndistinct` 准确，过滤后缩放仍可能带来误差。

建议使用下面这套从 catalog 到 numGroups 的诊断流程：

1. 先确认 `pg_statistic_ext.stxkind` 中包含 ndistinct kind。
2. 再确认 `pg_statistic_ext_data.stxdndistinct` 在目标 inherit 标志下不是空值。
3. 观察 `stxdndistinct` 输出中是否存在目标组合。
4. 用 `EXPLAIN` 比较创建统计对象前后的 Aggregate 或 Unique estimated rows。
5. 在 `estimate_num_groups()` 中确认 grouping expression 被加入 `GroupVarInfo`。
6. 在 `estimate_multivariate_ndistinct()` 中确认统计对象被选中。
7. 确认匹配成功后已匹配变量从 `relvarinfos` 中移除。
8. 确认最终 group count 是否被 `input_rows` 或 relation-level clamp 改写。

## 13. 课堂实验：观察 GROUP BY 组数修正

实验目标是让学员看到 ndistinct 如何修正 GROUP BY 组数，而不是 WHERE rows。
准备一张表，让 `a` 与 `b` 强相关，例如 `b = a` 或 `b = a / 10`。
第一步只执行普通 `ANALYZE`，记录 `GROUP BY a, b` 的 estimated rows 和 actual rows。
第二步执行 `CREATE STATISTICS s_ab_nd (ndistinct) ON a, b FROM t_nd`，再执行 `ANALYZE`。
第三步重新解释同一个 GROUP BY 查询，观察 Aggregate 节点 rows 是否接近真实组数。
第四步执行 `GROUP BY a`，确认单列 group count 本来不需要多变量 ndistinct。
第五步执行 `GROUP BY a, b, c`，观察两列统计对象是否不能覆盖三列组合。
第六步创建三列 ndistinct 统计，再观察 `(a,b,c)` 组合是否改善。
第七步加入 WHERE 过滤，观察 `estimate_num_groups()` 后续过滤缩放带来的变化。
第八步把 grouping expression 改成未声明表达式，例如 `a + 1`，观察匹配是否失败。
第九步显式查询 `pg_statistic_ext_data.stxdndistinct`，把组合 item 与 `EXPLAIN` rows 对照。

| 实验现象 | 应回到的源码 | 解释重点 |
| --- | --- | --- |
| GROUP BY rows 明显改善 | `estimate_multivariate_ndistinct()` | 组合 item 替代了单列乘法。 |
| GROUP BY 单列无变化 | `estimate_num_groups()` | 单列统计已经覆盖普通列。 |
| 三列 grouping 无改善 | `MVNDistinct.items` 匹配 | 二列统计对象没有三列 item。 |
| WHERE 后 rows 仍偏 | `estimate_num_groups()` 过滤缩放 | 组合 distinct 之后仍有 restriction selectivity 近似。 |
| 表达式改写后失效 | `equal(varinfo->var, expr)` | grouping expression 没有匹配统计对象表达式。 |

## 14. 源码练习：跟踪 group count 如何形成

源码练习围绕一个 `GROUP BY a, b` 查询展开。
练习一：跟踪 group count 如何形成。

1. 在 `planner.c` 中找到当前 query 调用 `estimate_num_groups()` 的位置。
2. 进入 `estimate_num_groups()`，记录 `groupExprs` 与 `input_rows`。
3. 观察 boolean、volatile 和 expression 处理是否触发特殊路径。
4. 观察 `add_unique_group_var()` 后 `varinfos` 中每个 `GroupVarInfo` 的 relation。
5. 进入 relation 分组循环，确认 `relvarinfos` 是否包含两个或更多变量。
6. 进入 `estimate_multivariate_ndistinct()`，观察候选 `StatisticExtInfo`。
7. 确认 `statext_ndistinct_load()` 读取的是正确 inherit 标志的数据。
8. 确认 `MVNDistinctItem.ndistinct` 返回后如何乘入 `reldistinct`。
9. 观察剩余变量是否继续走单列 distinct fallback。
10. 观察 clamp 和 filtering 缩放后最终返回的 `numdistinct`。

练习二：跟踪 ndistinct 如何构建。

1. 在 `BuildRelationExtStatistics()` 观察目标统计对象的 kind 列表。
2. 进入 `statext_ndistinct_build()`，确认 `numattrs` 与 `num_combinations()`。
3. 观察 k 从 2 到 `numattrs` 的组合枚举。
4. 进入 `ndistinct_for_combination()`，观察当前组合的 attnums。
5. 观察样本值如何复制到 `SortItem` 数组。
6. 观察排序后如何统计 distinct 组合数 `d` 和 singleton 数 `f1`。
7. 进入 `estimate_ndistinct()`，确认输出被 clamp 到 `[d, totalrows]`。
8. 进入 `statext_store()`，确认写入的是 `stxdndistinct`。

## 15. 常见误区：把 ndistinct 当成 WHERE 选择率

下面这些判断在排查 ndistinct 时很常见，但都需要修正：

- 误区一：认为 ndistinct 会改善 WHERE 条件选择率，实际上 WHERE 主要看 MCV 和 dependencies。
- 误区二：认为两列统计对象可以自动服务三列 GROUP BY，实际上必须有匹配的组合 item。
- 误区三：认为 `MVNDistinctItem.ndistinct` 是过滤后的组数，实际上它先表示 relation 级组合 distinct。
- 误区四：忽略 `rel->rows < rel->tuples` 时的过滤缩放，导致把误差全部归因于统计对象。
- 误区五：把 `input_rows` clamp 后的结果误认为 catalog 中保存的 ndistinct。
- 误区六：把 expression stats 和普通列 stats 混用，忽略表达式树必须匹配。
- 误区七：看到 HashAgg spill 就直接调 `work_mem`，没有先确认 group count 是否被低估。
- 误区八：把 actual groups 当成 planner 事前可知事实，忽略统计只能从样本外推。
- 误区九：忽略 inherit 标志，导致 partition 或 inheritance 查询使用了另一份统计数据。
- 误区十：忽略 `estimate_multivariate_bucketsize()`，以为 ndistinct 只影响 Aggregate。

## 16. 讨论题：围绕组合匹配、clamp 与成本传播

讨论题要求回答者引用源码入口和可观测现象，不能只给经验结论：

- 为什么 `estimate_num_groups()` 不能直接把所有单列 distinct 相乘。
- 为什么 multivariate ndistinct 统计对象需要为所有二列及以上组合生成 item。
- 为什么匹配成功后必须从 `varinfos` 中移除已匹配变量。
- 为什么 `estimate_num_groups()` 仍然要在 ndistinct 后做 relation-level clamp。
- 为什么 WHERE 过滤后 group count 的缩放仍然可能产生误差。
- 为什么 `GROUP BY a + 1, b` 不能自动使用 `ON a, b` 的统计对象。
- 为什么 ndistinct 能影响 HashAgg 成本，但不能保证 HashAgg 一定不会 spill。
- 为什么 `pg_statistic_ext_data.stxdndistinct` 能证明数据存在，却不能证明当前 query 使用了它。

## 17. 本节小结：组合 distinct 如何进入 upper path

Multivariate ndistinct 的核心不是让 planner 知道每个组的内容，而是让 planner 对多维 key 的组合数量少犯系统性乘法错误。
构建侧的主线是 `BuildRelationExtStatistics()` 调用 `statext_ndistinct_build()`，再由 `ndistinct_for_combination()` 和 `estimate_ndistinct()` 从样本外推组合 distinct。
消费侧的主线是 `estimate_num_groups()` 把 grouping expressions 转成 `GroupVarInfo`，并用 `estimate_multivariate_ndistinct()` 替换同一 relation 上匹配组合的单列乘法。
匹配失败时，源码不会中断 planning，而是退回单列 distinct 乘法、relation clamp、过滤缩放和 input rows clamp。
这个 group count 会继续传播到 `create_agg_path()`、`cost_agg()`、`estimate_hashagg_tablesize()`、`cost_group()` 和相关 Unique 路径。
诊断时要沿 `stxdndistinct -> RelOptInfo.statlist -> GroupVarInfo -> estimate_multivariate_ndistinct -> numGroups -> path cost` 这条线走。
下一组课程会继续讨论选择率 API 和默认值，解释当统计缺失时 planner 如何用更粗的估算继续完成规划。

## 18. ndistinct 源码阅读检查点

- 检查点 01（对象定义）：`CreateStatistics()` 是否把 `STATS_EXT_NDISTINCT` 写入 `stxkind`。
- 检查点 02（对象定义）：未指定 kind 时是否因为多列对象默认开启 ndistinct。
- 检查点 03（对象定义）：单个普通列是否被拒绝创建扩展统计。
- 检查点 04（对象定义）：表达式统计是否通过 `stxexprs` 保存表达式树。
- 检查点 05（采样规模）：`ComputeExtStatisticsRows()` 是否把对象 target 转成样本行数贡献。
- 检查点 06（构建入口）：`BuildRelationExtStatistics()` 是否成功拿到目标统计对象。
- 检查点 07（构建入口）：`lookup_var_attr_stats()` 是否因为 partial analyze 返回空。
- 检查点 08（构建入口）：`statext_compute_stattarget()` 是否得到非零 target。
- 检查点 09（构建数据）：`StatsBuildData.nattnums` 是否等于统计对象维度数。
- 检查点 10（构建数据）：`StatsBuildData.values` 是否包含每个维度的样本值。
- 检查点 11（组合枚举）：`num_combinations()` 是否为所有二列及以上组合计数。
- 检查点 12（组合枚举）：`generator_next()` 是否按 k 值枚举组合。
- 检查点 13（组合估算）：`ndistinct_for_combination()` 是否为当前组合建立 `MultiSortSupport`。
- 检查点 14（组合估算）：`qsort_interruptible()` 是否对样本组合排序。
- 检查点 15（组合估算）：扫描排序结果时是否正确统计 `d` 和 `f1`。
- 检查点 16（组合估算）：`estimate_ndistinct()` 是否把估算 clamp 到 `[d, totalrows]`。
- 检查点 17（目录写入）：`statext_ndistinct_serialize()` 是否保存 item 的属性编号。
- 检查点 18（目录写入）：`statext_store()` 是否写入 `stxdndistinct`。
- 检查点 19（统计装载）：`get_relation_statistics_worker()` 是否只为 built ndistinct kind 创建 info。
- 检查点 20（统计装载）：`StatisticExtInfo.inherit` 是否匹配当前 RTE。
- 检查点 21（消费入口）：`planner.c` 当前路径是否调用了 `estimate_num_groups()`。
- 检查点 22（消费入口）：`estimate_num_groups()` 是否把 `input_rows` 先 clamp。
- 检查点 23（表达式处理）：boolean grouping expression 是否被当成两个组。
- 检查点 24（表达式处理）：volatile variable-free expression 是否让函数返回 `input_rows`。
- 检查点 25（变量整理）：`add_unique_group_var()` 是否去掉重复变量。
- 检查点 26（变量整理）：同一 equivalence class 中已知相等的跨 relation 变量是否被减少。
- 检查点 27（relation 分组）：`relvarinfos` 是否只包含当前 relation 的变量。
- 检查点 28（多变量匹配）：`estimate_multivariate_ndistinct()` 是否看到 `rel->statlist`。
- 检查点 29（多变量匹配）：候选统计对象是否要求 kind 等于 `STATS_EXT_NDISTINCT`。
- 检查点 30（多变量匹配）：候选对象是否要求匹配至少两个变量或表达式。
- 检查点 31（多变量匹配）：多个对象竞争时是否优先表达式匹配数。
- 检查点 32（多变量匹配）：`statext_ndistinct_load()` 是否成功反序列化 catalog 数据。
- 检查点 33（多变量匹配）：表达式匹配是否依赖 `equal()` 对表达式树比较。
- 检查点 34（多变量匹配）：普通列匹配是否依赖 attnum 和 `info->keys`。
- 检查点 35（多变量匹配）：找到 item 后是否输出 `MVNDistinctItem.ndistinct`。
- 检查点 36（多变量匹配）：匹配变量是否从 `varinfos` 中移除。
- 检查点 37（fallback）：剩余变量是否回到单列 distinct 乘法。
- 检查点 38（fallback）：单列 default distinct 是否设置 `SELFLAG_USED_DEFAULT`。
- 检查点 39（clamp）：同一 relation 多变量时是否应用 `rel->tuples * 0.1` clamp。
- 检查点 40（clamp）：过滤后缩放是否使用 `rel->rows < rel->tuples` 条件。
- 检查点 41（输出）：最终结果是否向上取整并限制在 `[1, input_rows]`。
- 检查点 42（路径传播）：`create_agg_path()` 是否把 `numGroups` 写入 path rows。
- 检查点 43（路径传播）：`cost_agg()` 是否用 `numGroups` 估算输出 tuple 和 HashAgg 成本。
- 检查点 44（相邻用途）：`estimate_multivariate_bucketsize()` 是否用 ndistinct 修正多列 hash key bucket。

## 附录. ndistinct 阅读检查清单

- 检查点 01：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 01：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 01：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 02：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 02：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 02：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 03：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 03：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 03：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 04：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 04：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 04：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 05：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 05：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 05：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 06：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 06：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 06：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 07：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 07：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 07：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 08：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 08：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 08：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 09：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 09：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 09：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 10：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 10：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 10：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 11：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 11：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 11：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 12：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 12：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 12：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 13：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 13：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 13：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 14：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 14：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 14：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 15：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 15：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 15：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 16：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 16：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 16：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 17：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 17：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 17：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 18：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 18：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 18：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 19：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 19：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 19：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 20：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 20：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 20：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 21：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 21：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 21：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 22：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 22：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 22：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 23：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 23：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 23：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 24：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 24：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 24：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 25：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 25：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 25：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 26：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 26：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 26：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 27：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 27：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 27：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 28：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 28：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 28：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 29：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 29：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 29：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 30：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 30：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 30：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 31：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 31：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 31：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 32：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 32：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 32：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 33：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 33：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 33：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 34：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 34：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 34：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 35：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 35：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 35：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 36：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 36：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 36：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 37：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 37：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 37：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 38：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 38：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 38：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 39：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 39：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 39：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 40：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 40：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 40：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 41：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 41：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 41：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 42：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 42：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 42：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 43：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 43：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 43：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 44：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 44：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 44：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 45：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 45：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 45：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 46：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 46：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 46：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 47：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 47：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 47：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 48：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 48：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 48：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 49：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 49：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 49：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 50：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 50：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 50：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 51：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 51：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 51：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 52：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 52：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 52：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 53：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 53：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 53：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 54：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 54：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 54：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 55：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 55：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 55：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 56：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 56：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 56：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 57：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 57：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 57：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 58：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 58：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 58：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 59：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 59：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 59：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 60：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 60：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 60：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 61：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 61：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 61：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 62：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 62：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 62：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 63：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 63：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 63：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 64：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 64：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 64：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 65：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 65：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 65：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 66：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 66：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 66：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 67：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 67：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 67：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 68：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 68：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 68：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 69：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 69：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 69：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 70：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 70：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 70：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 71：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 71：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 71：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 72：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 72：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 72：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 73：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 73：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 73：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 74：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 74：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 74：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 75：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 75：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 75：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 76：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 76：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 76：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 77：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 77：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 77：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 78：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 78：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 78：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 79：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 79：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 79：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 80：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 80：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 80：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 81：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 81：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 81：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 82：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 82：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 82：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 83：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 83：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 83：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 84：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 84：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 84：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 85：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 85：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 85：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 86：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 86：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 86：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 87：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 87：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 87：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 88：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 88：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 88：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 89：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 89：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 89：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 90：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 90：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 90：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 91：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 91：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 91：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 92：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 92：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 92：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 93：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 93：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 93：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 94：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 94：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 94：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。
