# PostgreSQL Multivariate MCV 与 clause selectivity 修正

## 课程定位

前置知识：已经理解 `ANALYZE` 的采样过程、单列 `pg_statistic` 的 MCV 与 histogram、`RestrictInfo` 如何承载限制条件，以及 `clauselist_selectivity_ext()` 如何把条件列表压缩成一个 `Selectivity`。
本节唯一主问题可以写成这个判断：为什么一个 multivariate MCV list 只能修正它覆盖到的具体高价值组合，而不能替代所有单列 selectivity 和所有谓词组合。
核心矛盾：planner 需要尽早修正列间相关性，否则会把强相关条件当成独立事件相乘；但 `ANALYZE` 只能从有限样本中保存有限个组合，不能把整张表的联合分布塞进系统目录。
学完后应能判断：一个 base relation 的 rows 偏差，是来自没有 MCV 统计、MCV 没有覆盖当前组合、子句不兼容、inherit 数据不匹配，还是来自 MCV 修正后继续被 path cost 放大。
本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面几节已经把扩展统计对象、表达式统计和 functional dependencies 分开讲过。
functional dependencies 回答的是“一个列的取值能在多大程度上决定另一个列”。
multivariate MCV 回答的是“某些具体值组合在真实样本中是不是比独立性假设更常见或更少见”。
这两个机制都服务相关性修正，但它们不是同一个 fallback 层。
本节只追 MCV 这一条链路。
从运行时间看，链路分为两个阶段。
构建阶段发生在 `ANALYZE` 中，`BuildRelationExtStatistics()` 调用 `statext_mcv_build()`，再把序列化结果写入 `pg_statistic_ext_data.stxdmcv`。
消费阶段发生在 planner 中，`clauselist_selectivity_ext()` 调用 `statext_clauselist_selectivity()`，再由 `statext_mcv_clauselist_selectivity()` 尝试用 MCV 修正 clause list。
本节不展开单个 operator 的 selectivity estimator，因为那些入口已经在选择率课程中覆盖。
本节也不把 `pg_statistic_ext` 写成 catalog 清单，因为主问题不是目录字段，而是 MCV 什么时候能够改变 `RelOptInfo.rows`。
下一节会切换到 ndistinct/grouping，那里关注的不是 WHERE 命中多少行，而是 GROUP BY、DISTINCT 和 HashAgg 会输出多少组。

本节只处理下面这些源码边界与诊断边界：

- `CREATE STATISTICS ... (mcv)` 只声明统计对象，真正的数据要等 `ANALYZE` 构建。
- `MCVList` 保存的是有限个组合值及其频率，不是完整联合分布。
- `mcv_combine_selectivities()` 只把 MCV 覆盖部分作为 correction，并保留普通估算处理未覆盖部分。
- `estimatedclauses` 防止同一个 clause 同时被 MCV、dependencies 和普通估算重复消费。
- `RelOptInfo.rows` 改变后会继续影响 scan、join、sort、aggregate 等 path cost。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`ANALYZE` 从样本中构建 `MCVList`，planner 在 clause list 估算时用 `mcv_sel - mcv_basesel` 修正独立性假设覆盖不到的相关性，再把剩余部分交给普通 selectivity 估算。
这里最容易误解的是“有多列 MCV 就直接用它算 selectivity”。
源码实际做法比这个误解更保守。
`simple_sel` 表示不使用扩展统计时的普通估算。
`mcv_sel` 表示当前 clause list 命中的 MCV item 的真实频率和。
`mcv_basesel` 表示这些 MCV item 如果按单列独立性估算时应该占多少。
`mcv_totalsel` 表示整张 MCV list 覆盖了表中多大比例。
`mcv_combine_selectivities()` 先用 `simple_sel - mcv_basesel` 得到未覆盖部分，再把它和 `mcv_sel` 相加。
这个模型避免了诊断中最常见的两个极端。
如果完全相信单列乘法，强相关列会系统性误估。
如果完全相信 MCV，未覆盖长尾会被错误地当成已知组合处理。
PostgreSQL 的折中是让 MCV 做局部校正，而不是让它接管全部概率空间。

| 估算层次 | 源码入口 | 本节解释的语义 |
| --- | --- | --- |
| 统计对象声明 | `CreateStatistics()` | 记录统计对象覆盖哪些列、表达式和 kind。 |
| 扩展样本规模 | `ComputeExtStatisticsRows()` | 为扩展统计提高或决定采样行数。 |
| MCV 构建 | `statext_mcv_build()` | 从样本中找组合、算真实频率和 base frequency。 |
| 目录存储 | `statext_store()` | 把 MCV 序列化到 `pg_statistic_ext_data.stxdmcv`。 |
| 统计装载 | `get_relation_statistics()` | 把已构建 kind 变成 `RelOptInfo.statlist`。 |
| 统计选择 | `choose_best_statistics()` | 在未估算 clause 中选择覆盖最好的统计对象。 |
| MCV 匹配 | `mcv_get_match_bitmap()` | 逐个 MCV item 判断是否满足 clause list。 |
| 修正组合 | `mcv_combine_selectivities()` | 把普通估算和 MCV 覆盖部分合并。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/clausesel.c` | `clauselist_selectivity_ext()` 何时进入扩展统计，何时回到普通 clause 估算。 |
| 2 | `src/backend/statistics/extended_stats.c` | `BuildRelationExtStatistics()` 和 `statext_mcv_clauselist_selectivity()` 分别处理构建与消费。 |
| 3 | `src/backend/statistics/mcv.c` | `statext_mcv_build()`、`mcv_get_match_bitmap()`、`mcv_combine_selectivities()` 是 MCV 主线。 |
| 4 | `src/include/statistics/statistics.h` | `MCVList`、`MCVItem`、`STATS_MCV_MAGIC` 和 `STATS_MCVLIST_MAX_ITEMS` 的状态语义。 |
| 5 | `src/include/statistics/extended_stats_internal.h` | `StatsBuildData`、`SortItem`、`MultiSortSupport` 是构建阶段的临时状态。 |
| 6 | `src/backend/optimizer/util/plancat.c` | `get_relation_statistics_worker()` 根据 `pg_statistic_ext_data` 生成 `StatisticExtInfo`。 |
| 7 | `src/include/nodes/pathnodes.h` | `RelOptInfo.statlist` 和 `StatisticExtInfo` 是 planner 消费统计对象的边界。 |
| 8 | `src/backend/catalog/system_views.sql` | `pg_stats_ext` 与 `pg_mcv_list_items()` 是观察 MCV 摘要的 SQL 入口。 |
| 9 | `src/test/regress/sql/stats_ext.sql` | 回归测试给出了 `check_estimated_rows()` 和 MCV 观测的可复现实验骨架。 |

推荐阅读顺序是先读消费者，再读构建者。
先看 `clausesel.c`，因为它定义了 MCV 是否进入 plan rows 的真实门槛。
再看 `extended_stats.c`，因为它同时解释构建时的 `BuildRelationExtStatistics()` 和估算时的 `statext_clauselist_selectivity()`。
最后再进入 `mcv.c`，因为 `MCVList` 的序列化格式只有在知道它如何被消费后才有诊断意义。
不要从 `pg_mcv_list_items()` 的输出开始倒背字段。
诊断时要从 rows 偏差回到 `RelOptInfo.statlist`，再回到 `mcv_combine_selectivities()` 的四个输入。

## 4. 从一个 rows 偏差现场切入

考虑一张订单表，其中 `country` 与 `city` 强相关，`city = Beijing` 基本只出现在 `country = CN` 的行中。
如果没有扩展统计，`country = CN` 与 `city = Beijing` 会分别走单列统计，然后在 AND 条件下近似相乘。
假设单列选择率分别是 0.2 和 0.01，简单乘法会得到 0.002。
真实选择率可能接近 0.01，因为 `city = Beijing` 已经隐含了大部分 `country = CN` 信息。
MCV 的作用就是在这种具体组合上校正独立性假设。
但它不会校正所有城市，也不会理解任意业务规则。
只有当统计对象覆盖了这些列，`ANALYZE` 构建出了 MCV list，并且 clause 能被 `statext_is_compatible_clause()` 接受，planner 才可能使用它。
这个现场的诊断顺序应该是从 `EXPLAIN` 的 estimated rows 开始，而不是先猜索引问题。
如果 MCV 修正前后 rows 改变明显，后续 path 选择改变只是结果，不是根因。

把这个现场回到源码时，推荐按下面顺序定位：

1. 确认 query 是 base relation restriction，而不是 join selectivity 或 upper relation 的 rows。
2. 确认 `CREATE STATISTICS` 指定了 `mcv`，或者未指定 kind 时默认包含了 `mcv`。
3. 确认 `ANALYZE` 已经执行，并且 `pg_statistic_ext_data.stxdmcv` 不是空值。
4. 确认 `plancat.c` 在当前 planner run 中把 built MCV kind 装进 `RelOptInfo.statlist`。
5. 确认 `statext_mcv_clauselist_selectivity()` 能从 clause 中抽出被统计对象覆盖的 attnums 或表达式。
6. 确认 `choose_best_statistics()` 没有被另一个覆盖更多 clause 的统计对象抢走当前轮机会。
7. 确认命中的 MCV item 的 `frequency` 与 `base_frequency` 差异足够影响 `stat_sel`。
8. 确认未被 MCV 覆盖的 clause 仍然会在普通估算路径中继续处理。

## 5. 关键数据结构与状态边界

`MCVList` 定义在 `src/include/statistics/statistics.h`。
它是一个变长对象，保存 magic、type、item 数、维度数、每个维度的数据类型，以及一个 `MCVItem` 数组。
`MCVItem` 保存一组多维值、每个维度是否为 NULL、该组合的真实频率和独立性假设下的 base frequency。
这几个字段必须放在同一个语义单元里解释。
只看 `values`，只能知道样本中保存了哪个组合。
只看 `frequency`，不知道它相对独立性假设是更常见还是更少见。
只看 `base_frequency`，不知道真实样本是否支持这个组合。
`frequency - base_frequency` 才是 MCV 对普通估算的主要修正信号。
`StatisticExtInfo` 定义在 `src/include/nodes/pathnodes.h`，它是 planner 的轻量索引。
它保存统计对象 OID、kind、inherit 标志、keys 和 expressions，但不直接保存 MCV list 内容。
真正的 MCV list 在需要估算时由 `statext_mcv_load()` 从 `pg_statistic_ext_data` 读取并反序列化。
`RelOptInfo.statlist` 是当前 base relation 可用扩展统计的入口。
如果这里为空，后续 `statext_mcv_clauselist_selectivity()` 会直接返回 AND 语义下的 1.0 或 OR 语义下的 0.0。
`estimatedclauses` 是估算阶段的重要边界。
它不是统计对象状态，而是本次 clause-list 估算的 bookkeeping。
一个 clause 被 MCV 消费后，bit 会被置位，后续 dependencies 和普通估算就不能再把它当成未处理条件。
没有这个边界，同一条件可能被重复缩放，rows 会被压得过小。

| 状态 | 生命周期 | 诊断意义 |
| --- | --- | --- |
| `pg_statistic_ext` tuple | 统计对象定义存在期间 | 说明对象声明了哪些列、表达式和 kind。 |
| `pg_statistic_ext_data.stxdmcv` | 每次 `ANALYZE` 重建并替换 | 说明 MCV 数据是否已经实际构建。 |
| `StatsBuildData` | `BuildRelationExtStatistics()` 的工作 context 内 | 承载样本值、NULL 标记和属性统计入口。 |
| `MCVList` 构建结果 | 构建 context 内生成，随后被序列化 | 只是一份有限组合摘要，不是完整分布表。 |
| `StatisticExtInfo` | planner run 内 backend-local | 让估算器知道哪些 built kind 可用。 |
| `estimatedclauses` | 一次 clause-list selectivity 调用期间 | 防止多种扩展统计重复消费同一 clause。 |
| `RelOptInfo.rows` | 当前 planner run 的 base rel 状态 | 承接 selectivity，并向所有 path 成本传播。 |

## 6. 构建阶段源码 walkthrough

构建阶段从 `ANALYZE` 开始，不从 `CREATE STATISTICS` 开始。
`CREATE STATISTICS` 只在 `src/backend/commands/statscmds.c` 的 `CreateStatistics()` 中写入统计对象定义。
`CreateStatistics()` 会检查只能针对单个 relation，检查 relation kind，检查 ownership，并禁止系统列等不支持目标。
当用户指定 `(mcv)` 时，`CreateStatistics()` 把 `STATS_EXT_MCV` 放入 `stxkind`。
如果用户没有指定 kind 且统计对象覆盖两个或更多列或表达式，源码会默认开启 ndistinct、dependencies 和 mcv。
统计对象定义写入 `pg_statistic_ext` 后，会记录列依赖、表达式依赖、namespace 与 owner 依赖，并通过 `CacheInvalidateRelcache()` 让后续 planner 能看到新对象。
真正的数据构建发生在 `src/backend/commands/analyze.c` 调用扩展统计入口之后。
`ComputeExtStatisticsRows()` 会扫描 relation 的统计对象，决定扩展统计需要的最大 statistics target，并返回 `300 * target` 作为采样规模贡献。
`BuildRelationExtStatistics()` 会打开 `pg_statistic_ext`，取出当前 relation 的统计对象列表。
它创建名为 `BuildRelationExtStatistics` 的 memory context，并把每个统计对象的构建状态放在这个 context 下。
每处理完一个统计对象，源码调用 `MemoryContextReset(cxt)` 清理临时状态。
这个 reset 边界很重要，因为 MCV 构建会分配排序数组、组合值数组、频率数组和序列化输入。
如果某个统计对象需要的列没有被本次 `ANALYZE` 覆盖，`lookup_var_attr_stats()` 会返回空，普通 analyze 不会因此失败。
如果统计对象 target 为 0，`BuildRelationExtStatistics()` 会跳过重建，并保留旧值，行为与普通列统计 target 为 0 的语义一致。
对 MCV kind，`BuildRelationExtStatistics()` 调用 `statext_mcv_build(data, totalrows, stattarget)`。
`statext_mcv_build()` 先用 `build_mss()` 为多维排序准备 `MultiSortSupport`。
随后它调用 `build_sorted_items()` 从样本中抽取目标维度值，并按多维 key 排序。
再由 `build_distinct_groups()` 把连续相同组合压缩成 group，并按频率排序。
`get_mincount_for_mcv_list()` 根据样本行数和总行数计算保留 MCV item 的最低出现次数。
源码注释明确说明，多列 MCV 不能照搬单列 MCV 策略，因为它既要关注异常常见组合，也要关注相对 base frequency 异常的组合。
构建 `MCVItem` 时，`frequency` 来自组合在样本中的计数除以样本行数。
构建 `base_frequency` 时，源码会计算每个单独维度值的频率，然后把各维度频率相乘。
这一点直接解释了 planner 估算阶段为什么需要同时拿 `mcv_sel` 和 `mcv_basesel`。
最后 `statext_store()` 调用 `statext_mcv_serialize()`，把 MCV list 写入 `pg_statistic_ext_data.stxdmcv`。
`statext_store()` 写入前会调用 `RemoveStatisticsDataById(statOid, inh)` 删除旧数据，再插入新 tuple。
这个“删旧再插新”的实现说明扩展统计数据是 analyze 结果，不是随 DML 实时维护的索引。

构建阶段可以按下面的时间顺序阅读：

1. `CreateStatistics()` 记录对象定义、kind、keys、表达式和依赖。
2. `ANALYZE` 准备普通列统计，并把样本行交给扩展统计入口。
3. `ComputeExtStatisticsRows()` 让扩展统计影响采样规模。
4. `BuildRelationExtStatistics()` 为每个统计对象建立临时 memory context。
5. `lookup_var_attr_stats()` 确认本次样本和 `VacAttrStats` 能覆盖对象需要的列和表达式。
6. `statext_compute_stattarget()` 计算对象级、列级和默认 target 的合成结果。
7. `make_build_data()` 把样本值整理成 `StatsBuildData`。
8. `statext_mcv_build()` 对多维值排序、分组、筛选并计算 frequency。
9. `statext_mcv_serialize()` 把 in-memory `MCVList` 变成 varlena 数据。
10. `statext_store()` 替换 `pg_statistic_ext_data` 中对应 inherit 版本的数据。

## 7. 估算阶段源码 walkthrough

估算阶段从 `src/backend/optimizer/path/clausesel.c` 开始。
`clauselist_selectivity_ext()` 处理 AND 语义的 clause list。
当 `use_extended_stats` 为真，并且当前 relation 有 `statlist` 时，它会先调用 `statext_clauselist_selectivity()`。
`statext_clauselist_selectivity()` 首先尝试 MCV，因为 MCV 能表达具体值组合的联合频率。
只有在不是 OR 的场景下，它才继续调用 functional dependencies 修正剩余 clause。
这条顺序说明 MCV 比 dependencies 更具体。
`statext_mcv_clauselist_selectivity()` 先检查 `rel->statlist` 里是否存在 `STATS_EXT_MCV` kind。
如果没有 MCV kind，AND 场景返回 1.0，OR 场景返回 0.0，调用者继续处理普通估算。
接着它遍历 clause list，用 `statext_is_compatible_clause()` 提取每个 clause 涉及的 attnums 或表达式。
已经在 `estimatedclauses` 中置位的 clause 会被跳过。
这一步把 SQL 表达式降成“统计对象是否覆盖”的问题。
之后源码进入一个 greedy loop。
每一轮调用 `choose_best_statistics()`，从剩余可用 clause 中选择覆盖最多属性或表达式的 MCV 统计对象。
选中统计对象后，源码再过滤出完全被该对象覆盖的 `stat_clauses`。
这些 clause 会被加入本轮 MCV 估算，并同步把原 clause 位置加入 `estimatedclauses`。
AND 场景下，源码先调用 `clauselist_selectivity_ext(..., false)` 得到 `simple_sel`。
这里传入 false 是为了避免递归再次使用扩展统计。
然后 `mcv_clauselist_selectivity()` 加载 MCV list，调用 `mcv_get_match_bitmap()` 得到每个 item 是否满足所有 clause。
它遍历所有 MCV item，累加命中项的 `frequency` 得到 `mcv_sel`，累加命中项的 `base_frequency` 得到 `mcv_basesel`，并累加所有 MCV item 的 `frequency` 得到 `mcv_totalsel`。
最后 `mcv_combine_selectivities(simple_sel, mcv_sel, mcv_basesel, mcv_totalsel)` 输出本统计对象的 `stat_sel`。
`statext_mcv_clauselist_selectivity()` 把每轮 `stat_sel` 乘到总 `sel` 上。
OR 场景下，源码不能简单相乘。
它逐个 clause 调用 `mcv_clause_selectivity_or()`，维护一个 `or_matches` 位图，并用 `P(A OR B) = P(A) + P(B) - P(A AND B)` 的形式估算重叠。
这解释了为什么 OR 估算里 MCV 不再继续进入 functional dependencies。
最终，`clauselist_selectivity_ext()` 会对 `estimatedclauses` 没覆盖的 clause 继续调用普通 `clause_selectivity_ext()`。
这样 MCV 只修正它有证据的部分，不会吞掉未知部分。

估算阶段的主链路可以压缩成下面十步：

1. `set_baserel_size_estimates()` 需要一个 relation restriction selectivity。
2. `clauselist_selectivity_ext()` 收到 `baserestrictinfo` 中的 clause list。
3. `statext_clauselist_selectivity()` 先尝试 MCV，后尝试 dependencies。
4. `statext_mcv_clauselist_selectivity()` 为每个 clause 抽取 attnums 与 expressions。
5. `choose_best_statistics()` 在 `RelOptInfo.statlist` 中选择覆盖最好的 MCV 对象。
6. `stat_clauses` 只保留完全被该统计对象覆盖的兼容 clause。
7. `mcv_get_match_bitmap()` 对每个 MCV item 计算是否满足 clause list。
8. `mcv_clauselist_selectivity()` 汇总 `mcv_sel`、`mcv_basesel` 和 `mcv_totalsel`。
9. `mcv_combine_selectivities()` 合并普通估算和 MCV 覆盖部分。
10. `set_baserel_size_estimates()` 把最终 selectivity 乘到 relation tuples 上，形成 `RelOptInfo.rows`。

## 8. 生命周期 / ownership / cleanup

扩展统计对象有三个不同的生命周期。
第一个生命周期是对象定义。
`CreateStatistics()` 写入 `pg_statistic_ext`，这个 tuple 由系统目录持有，直到 `DROP STATISTICS`、列删除或表删除触发依赖清理。
`RemoveStatisticsById()` 删除定义时，会先删除 `pg_statistic_ext_data` 中 inherit true 与 false 两份可能存在的数据。
第二个生命周期是统计数据。
`ANALYZE` 期间 `statext_store()` 替换 `pg_statistic_ext_data`，这份数据反映上一次 analyze 样本，不随普通 DML 实时变化。
第三个生命周期是 planner-local 状态。
`get_relation_statistics()` 与 `get_relation_statistics_worker()` 在当前 planner run 中创建 `StatisticExtInfo`。
这些对象挂在 planner 的内存上下文里，查询规划结束后释放。
`statext_mcv_load()` 反序列化出的 `MCVList` 也是 backend-local 内存，不是 shared state，也不能跨 backend 共享指针。
构建阶段的临时状态更加短命。
`BuildRelationExtStatistics()` 为所有统计对象构建创建一个 context，并在每个对象后 reset。
这个 context 包含 `StatsBuildData`、排序数组、group 数组、临时频率数组和待序列化对象。
如果构建阶段发生 ERROR，PostgreSQL 的 memory context 机制会沿事务或语句错误恢复路径清理 backend-local 临时内存。
目录 tuple 写入属于普通 catalog 修改，会通过事务和 WAL 保证 crash safety。
但 MCV 估算本身不需要 executor 正确性保证，因为它只影响 plan choice。

| 对象 | owner | 释放或失效边界 |
| --- | --- | --- |
| 统计对象定义 | `pg_statistic_ext` catalog | `DROP STATISTICS`、列依赖、表依赖或事务回滚。 |
| 统计对象数据 | `pg_statistic_ext_data` catalog | 下一次 `ANALYZE` 替换，或统计对象删除时清理。 |
| 构建临时状态 | `BuildRelationExtStatistics` memory context | 每个统计对象后 reset，函数结束时 delete。 |
| planner 统计索引 | 当前 planner memory context | planner run 结束后释放。 |
| 反序列化 MCV | 当前 backend 的普通 palloc 内存 | 所在 memory context reset 或 delete 时释放。 |
| rows 估算结果 | `RelOptInfo` 与 `Path` | 当前 planner run 内传播，执行时不会再修正。 |

## 9. 正确性机制与 fallback

MCV selectivity 的“正确性”不是 SQL 语义正确性。
即使 MCV 完全缺失，executor 仍然会返回正确结果。
这里的正确性是 planner 估算边界：不能让统计对象不存在、过期或不兼容时产生错误缩放。
第一个 fallback 是对象缺失。
如果 `RelOptInfo.statlist` 里没有 `STATS_EXT_MCV`，MCV 入口返回中性值，后续普通估算继续运行。
第二个 fallback 是数据未构建。
`get_relation_statistics_worker()` 只有在 `pg_statistic_ext_data` 中对应 kind 已构建时才创建 `StatisticExtInfo`。
因此正常 planner 路径不会对未构建 kind 调用 `statext_mcv_load()`。
第三个 fallback 是 clause 不兼容。
`statext_is_compatible_clause()` 只接受能安全映射到统计对象维度的表达式。
不能抽取 attnums 或表达式的 clause 会留给普通 estimator。
第四个 fallback 是统计对象覆盖不全。
`stat_clauses` 只包含被当前统计对象完全覆盖的 clause。
部分覆盖不代表部分使用，因为那会把一个谓词的语义切成难以解释的概率片段。
第五个 fallback 是 MCV list 不覆盖当前值组合。
这种情况下 `mcv_sel` 可能很小或为零，但 `simple_sel - mcv_basesel` 仍然为未覆盖部分保留普通估算。
第六个 fallback 是 OR 场景。
OR 只使用 MCV 的重叠估算，不继续使用 dependencies，因为 dependency 模型只适合 AND 条件。

诊断 fallback 时要避免下面这些混淆：

- 不要把 `CREATE STATISTICS` 成功当作 MCV 数据已经存在，必须确认 `ANALYZE` 后的数据列。
- 不要把 `pg_stats_ext` 里有对象当作当前 query 一定使用对象，还要看 clause 是否兼容。
- 不要把 MCV 未命中理解成 bug，有限 list 本来就只覆盖一部分概率空间。
- 不要把 MCV 对 AND 的修正公式套到 OR，OR 的 overlap 路径是另一套位图逻辑。
- 不要把估算错误理解成执行错误，统计只影响计划选择，不改变过滤谓词。

## 10. 成本、资源与跨模块传播

MCV 的直接输出是 `Selectivity`，但工程后果是 rows。
`set_baserel_size_estimates()` 把 selectivity 乘到 base relation tuples 上，得到 `RelOptInfo.rows`。
`RelOptInfo.rows` 之后会进入 scan path、joinrel 构造、join selectivity、sort 成本、hash 表规模和并行路径成本。
因此一个 MCV 修正可能让 planner 从 seq scan 变成 index scan，也可能改变 join order。
这种变化不是 MCV 直接控制 path，而是 rows 进入 cost model 后产生的间接结果。
构建成本主要发生在 `ANALYZE`。
多列 MCV 需要对样本做多维排序、分组，并计算每个维度的单值频率。
`ComputeExtStatisticsRows()` 可能提高采样规模，统计对象 target 越高，构建和 catalog 存储成本越大。
planner 消费成本通常小于构建成本，但不是零。
每次估算需要选择统计对象、加载并反序列化 MCV list、遍历 MCV item 并评估 clause。
这也是为什么源码不尝试对所有统计对象和所有 clause 做穷举组合。
`choose_best_statistics()` 的 greedy 策略是一个成本边界。
它牺牲全局最优搜索，换取 planner hot path 的可控开销和确定性。
资源传播还包括 catalog 空间。
`STATS_MCVLIST_MAX_ITEMS` 受 `MAX_STATISTICS_TARGET` 约束，序列化格式还要保存每个维度去重值、NULL 标记、频率和 base frequency。
过高的 statistics target 会增加 analyze 时间和 catalog 体积，但不保证每个 workload 都收益。

| 成本位置 | 扩张因子 | 诊断入口 |
| --- | --- | --- |
| ANALYZE 构建 | 样本行数、维度数、statistics target | `EXPLAIN ANALYZE` 不显示，需要观察 analyze 时间和 progress。 |
| catalog 存储 | MCV item 数、类型宽度、NULL 标记、去重值数量 | `pg_statistic_ext_data.stxdmcv` 和 `pg_mcv_list_items()`。 |
| planner 估算 | clause 数、统计对象数、MCV item 数 | 需要源码断点或 debug 日志间接判断。 |
| base rows 传播 | `RelOptInfo.rows` 的误差倍数 | `EXPLAIN` 中 scan rows 与 join rows。 |
| path 成本传播 | rows、width、cpu_tuple_cost、random_page_cost | `EXPLAIN` 的 cost 与 chosen path。 |

## 11. 观测与诊断入口

MCV 是否存在可以从 SQL 层观察。
`pg_stats_ext` 会通过 `src/backend/catalog/system_views.sql` 暴露扩展统计摘要。
`pg_mcv_list_items()` 可以把 `stxdmcv` 展开成 item、values、null flags、frequency 和 base frequency。
这些观测能说明数据已经构建，却不能单独证明某条 query 使用了 MCV。
要证明使用路径，仍然要回到 clause 是否兼容、统计对象是否被装入 `RelOptInfo.statlist`、以及 rows 是否在 `ANALYZE` 前后发生符合预期的变化。
最实用的外部现象是 `EXPLAIN` 中 base scan 的 estimated rows。
先在没有统计对象时记录 rows。
再创建 `(mcv)` 统计对象并执行 `ANALYZE`。
然后重新 `EXPLAIN` 同一个 WHERE 条件。
如果 rows 从单列独立性乘法结果移动到更接近真实结果，说明 MCV 可能介入。
但还要排除同时构建的 dependencies 影响。
为了隔离 MCV，实验中最好显式写 `CREATE STATISTICS s (mcv) ON a, b FROM t`。
如果未指定 kind，PostgreSQL 会默认构建多种扩展统计，诊断会混杂。
源码级诊断可以在 `statext_mcv_clauselist_selectivity()`、`choose_best_statistics()`、`mcv_clauselist_selectivity()` 和 `mcv_combine_selectivities()` 设断点。
断点观察的关键不是所有字段，而是 `stat_clauses`、`estimatedclauses`、`simple_sel`、`mcv_sel`、`mcv_basesel` 和 `mcv_totalsel`。

建议使用下面这套从 catalog 到 rows 的诊断流程：

1. 先确认 `pg_statistic_ext` 中对象的 `stxkind` 包含 MCV kind。
2. 再确认 `pg_statistic_ext_data` 中对应 inherit 标志下的 `stxdmcv` 不是空值。
3. 用 `pg_mcv_list_items()` 展开 MCV list，观察目标组合是否在 item 中。
4. 用 `EXPLAIN` 比较创建统计对象前后的 base scan estimated rows。
5. 如果 rows 没变，检查 query 的 clause 是否能映射到统计对象的 key 或 expression。
6. 如果 rows 变化异常，检查 MCV item 的 `frequency` 与 `base_frequency` 差异。
7. 如果 path 变化过大，继续跟 `RelOptInfo.rows` 如何影响下游 join order。
8. 最后用实际 `EXPLAIN ANALYZE` rows 评估估算误差，但不要把执行后 rows 反写成 planner 事实。

## 12. 课堂实验：观察 MCV 修正的局部性

实验目标是让学员看到 MCV 只修正具体组合，而不是让所有条件自动变准。
准备一张两列强相关的表，例如让 `a` 与 `b` 在大多数行中相同，同时加入少量噪声行。
第一步只执行普通 `ANALYZE`，不创建扩展统计。
记录 `WHERE a = 42 AND b = 42` 的 estimated rows 和 actual rows。
第二步创建 `CREATE STATISTICS s_ab_mcv (mcv) ON a, b FROM t_mcv`，再执行 `ANALYZE`。
重新记录同一个查询的 estimated rows。
第三步查询一个不在 MCV list 中的低频组合，观察估算是否仍然依赖普通 selectivity。
第四步把 statistics target 调高，再 analyze，观察 MCV item 数和 estimated rows 是否改变。
第五步改写 predicate，例如把列包进不匹配统计表达式的函数，观察 MCV 是否失效。
第六步显式创建 `(dependencies)` 统计对象，比较它和 `(mcv)` 对同一 AND 条件的估算差异。
第七步使用 OR 条件，例如 `a = 42 OR b = 42`，观察 rows 变化并回到 OR overlap 逻辑解释。
第八步删除统计对象，再 analyze 或重新连接，确认估算回到普通路径。

| 实验现象 | 应回到的源码 | 解释重点 |
| --- | --- | --- |
| 强相关 AND 条件 rows 变准 | `mcv_combine_selectivities()` | MCV 覆盖组合修正了独立性假设。 |
| 低频组合 rows 变化不明显 | `mcv_get_match_bitmap()` | 目标组合不在有限 MCV item 中。 |
| 提高 target 后 item 增加 | `statext_mcv_build()` | 更多 group 通过 mincount 与 target 边界。 |
| 表达式改写后 rows 回退 | `statext_is_compatible_clause()` | clause 无法映射到统计对象维度。 |
| OR 条件变化与 AND 不同 | `mcv_clause_selectivity_or()` | OR 使用重叠项而不是乘法。 |

## 13. 源码练习：从 rows 偏差回到四个 selectivity 输入

源码练习要避免泛读，并且要绑定一条具体查询。
这节课只要求围绕一个具体 query 走通状态变化。
练习一：从 WHERE rows 偏差回到 MCV 修正。

1. 在 `clausesel.c` 的 `clauselist_selectivity_ext()` 设断点，确认当前 relation 的 `statlist` 是否为空。
2. 进入 `statext_clauselist_selectivity()`，确认 MCV 在 dependencies 之前被尝试。
3. 进入 `statext_mcv_clauselist_selectivity()`，打印每个 clause 抽取出的 attnums 与 expressions。
4. 观察 `choose_best_statistics()` 返回的 `StatisticExtInfo`，确认 kind、keys、exprs 和 inherit。
5. 观察 `stat_clauses` 与原始 clause list 的差别，确认哪些 clause 被 MCV 覆盖。
6. 进入 `mcv_clauselist_selectivity()`，记录 `mcv->nitems` 和 match bitmap。
7. 打印 `mcv_sel`、`mcv_basesel`、`mcv_totalsel` 和 `simple_sel`。
8. 回到 `set_baserel_size_estimates()`，观察 selectivity 如何变成 `RelOptInfo.rows`。

练习二：从 ANALYZE 构建回到 catalog 数据。

1. 在 `BuildRelationExtStatistics()` 观察 `statslist` 中是否包含目标统计对象。
2. 确认 `lookup_var_attr_stats()` 是否能找到对象需要的列统计入口。
3. 进入 `statext_mcv_build()`，观察样本行数、维度数和 `stattarget`。
4. 观察 `build_distinct_groups()` 之后 group 数和前几个 group 的 count。
5. 观察 `get_mincount_for_mcv_list()` 返回值如何限制保留 item。
6. 观察每个 `MCVItem` 的 `frequency` 和 `base_frequency`。
7. 进入 `statext_store()`，确认写入的是 `stxdmcv` 而不是 ndistinct 或 dependencies。
8. 用 SQL 层 `pg_mcv_list_items()` 验证 catalog 里看到的 item 与断点一致。

## 14. 常见误区：把 MCV 当成完整联合分布

下面这些判断在排查 MCV 时很常见，但都不够严谨：

- 误区一：看到 `CREATE STATISTICS` 成功就认为 planner 一定使用了 MCV。
- 误区二：看到 `pg_stats_ext` 有 most common values 就认为任意组合都能被联合估算。
- 误区三：把 MCV 看成完整二维或多维频率表，忽略了 `statistics target` 的有限边界。
- 误区四：把 `frequency` 当成最终 selectivity，忽略了 `simple_sel` 和未覆盖部分。
- 误区五：把 `base_frequency` 当成单列估算结果，忽略它只针对 MCV item 组合。
- 误区六：把 dependencies 与 MCV 的效果混在一起，导致无法解释 rows 改变来自哪一层。
- 误区七：只看 scan path 改变，不回到 `RelOptInfo.rows` 的来源。
- 误区八：认为估算错误会影响 SQL 结果正确性，实际上它只影响计划选择。
- 误区九：忽略 inherit 标志，导致父表、继承表和 partition 场景下看错统计数据。
- 误区十：忽略表达式匹配，认为 `lower(col)` 和 `col` 的统计对象可以互相替代。

## 15. 讨论题：围绕 correction 与 fallback

讨论题要求回答者引用源码入口和可观测现象，不能只给经验结论：

- 为什么 `mcv_combine_selectivities()` 不直接返回 `mcv_sel`。
- 为什么 `statext_clauselist_selectivity()` 要先尝试 MCV，再尝试 functional dependencies。
- 为什么 OR 条件不能沿用 AND 条件的乘法组合。
- 为什么 `estimatedclauses` 是 selectivity correctness 的一部分。
- 为什么提高 statistics target 可能改善估算，也可能只增加 analyze 和 catalog 成本。
- 为什么 MCV 对具体常量组合更强，但对未覆盖长尾并不神奇。
- 为什么 `pg_mcv_list_items()` 能证明数据存在，却不能单独证明 planner 使用了该数据。
- 为什么扩展统计数据不随普通 DML 实时维护，而是由 `ANALYZE` 重建。

## 16. 本节小结：有限摘要如何变成局部修正

Multivariate MCV 的核心不是“多列统计更准”这一句口号。
它的源码模型是：有限样本构建有限组合摘要，估算阶段只对匹配到的组合做局部校正，未覆盖部分继续依赖普通 selectivity。
`frequency` 与 `base_frequency` 的差值解释了 MCV 为什么能修正相关性。
`mcv_totalsel` 解释了 MCV 为什么不能吞掉未覆盖概率空间。
`estimatedclauses` 解释了 MCV 为什么要和 dependencies、普通估算共享同一个消费边界。
构建侧的主要入口是 `BuildRelationExtStatistics()`、`statext_mcv_build()` 和 `statext_store()`。
消费侧的主要入口是 `clauselist_selectivity_ext()`、`statext_mcv_clauselist_selectivity()` 和 `mcv_combine_selectivities()`。
诊断时要沿 `pg_statistic_ext_data.stxdmcv -> RelOptInfo.statlist -> stat_clauses -> RelOptInfo.rows -> Path cost` 这条线走。
下一节的 ndistinct 会把扩展统计从 WHERE selectivity 推到 GROUP BY 和 DISTINCT 的 group count。

## 17. MCV 源码阅读检查点

- 检查点 01（对象定义）：`CreateStatistics()` 是否把 `STATS_EXT_MCV` 放入 `stxkind`。
- 检查点 02（对象定义）：`CreateStatistics()` 是否因为未指定 kind 而默认构建所有多变量统计。
- 检查点 03（对象定义）：`CreateStatistics()` 是否拒绝系统列和没有默认排序操作符的数据类型。
- 检查点 04（对象定义）：`CacheInvalidateRelcache()` 是否让后续 planner 看到新对象定义。
- 检查点 05（采样规模）：`ComputeExtStatisticsRows()` 是否看到目标统计对象。
- 检查点 06（采样规模）：`statext_compute_stattarget()` 是否受对象 target 和列 target 共同影响。
- 检查点 07（构建入口）：`BuildRelationExtStatistics()` 是否为每个统计对象 reset 临时 context。
- 检查点 08（构建入口）：`lookup_var_attr_stats()` 是否因为 partial analyze 跳过目标对象。
- 检查点 09（构建入口）：`make_build_data()` 是否正确填充 `StatsBuildData.values` 和 `StatsBuildData.nulls`。
- 检查点 10（MCV 构建）：`build_mss()` 是否按每个维度准备默认排序支持。
- 检查点 11（MCV 构建）：`build_sorted_items()` 是否返回可排序的多维样本数组。
- 检查点 12（MCV 构建）：`build_distinct_groups()` 是否把相同组合合并成一个 group。
- 检查点 13（MCV 构建）：`get_mincount_for_mcv_list()` 是否限制低置信度组合。
- 检查点 14（MCV 构建）：`STATS_MCVLIST_MAX_ITEMS` 是否限制最终 item 数上限。
- 检查点 15（MCV 构建）：`MCVItem.frequency` 是否来自 group count 除以样本行数。
- 检查点 16（MCV 构建）：`MCVItem.base_frequency` 是否来自每个维度单值频率相乘。
- 检查点 17（目录写入）：`statext_mcv_serialize()` 是否保存类型、去重值、NULL 标记和频率。
- 检查点 18（目录写入）：`statext_store()` 是否只在 `mcv != NULL` 时写入 `stxdmcv`。
- 检查点 19（目录写入）：`RemoveStatisticsDataById()` 是否先删旧数据再插入新数据。
- 检查点 20（统计装载）：`get_relation_statistics_worker()` 是否只为 built kind 创建 `StatisticExtInfo`。
- 检查点 21（统计装载）：`StatisticExtInfo.kind` 是否等于 `STATS_EXT_MCV`。
- 检查点 22（统计装载）：`StatisticExtInfo.inherit` 是否匹配当前 RTE 的 inherit 语义。
- 检查点 23（估算入口）：`clauselist_selectivity_ext()` 是否传入 `use_extended_stats` 为真。
- 检查点 24（估算入口）：`statext_clauselist_selectivity()` 是否先调用 MCV 入口。
- 检查点 25（兼容检查）：`statext_is_compatible_clause()` 是否能抽取 attnums 或 expression。
- 检查点 26（兼容检查）：`estimatedclauses` 是否跳过已经由其他统计处理的 clause。
- 检查点 27（统计选择）：`choose_best_statistics()` 是否选择覆盖最多剩余 clause 的对象。
- 检查点 28（统计选择）：`stat_covers_expressions()` 是否要求表达式完全覆盖。
- 检查点 29（AND 估算）：`clauselist_selectivity_ext(..., false)` 是否避免递归使用扩展统计。
- 检查点 30（AND 估算）：`mcv_clauselist_selectivity()` 是否加载并遍历 MCV list。
- 检查点 31（AND 估算）：`mcv_get_match_bitmap()` 是否为每个 item 计算匹配结果。
- 检查点 32（AND 估算）：`mcv_basesel` 是否只累加命中 item 的 base frequency。
- 检查点 33（AND 估算）：`mcv_totalsel` 是否累加所有 MCV item 的真实 frequency。
- 检查点 34（AND 估算）：`mcv_combine_selectivities()` 是否把 non-MCV 部分限制在 `1 - mcv_totalsel` 内。
- 检查点 35（OR 估算）：`mcv_clause_selectivity_or()` 是否维护 `or_matches` 位图。
- 检查点 36（OR 估算）：OR 路径是否用 overlap 项修正 `P(A OR B)`。
- 检查点 37（后续传播）：`set_baserel_size_estimates()` 是否把 selectivity 变成 `rel->rows`。
- 检查点 38（后续传播）：下游 path 是否只是消费 rows，而不是重新读取 MCV。
- 检查点 39（观测入口）：`pg_stats_ext` 是否能看到统计对象摘要。
- 检查点 40（观测入口）：`pg_mcv_list_items()` 是否能展开 `stxdmcv` 的 item。

## 附录. MCV 阅读检查清单

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

- 检查点 95：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 95：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 95：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 96：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 96：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 96：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 97：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 97：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 97：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 98：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 98：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 98：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 99：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 99：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 99：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。

- 检查点 100：先说明这个判断来自哪一个源码入口，再说明它会改变哪个 planner 状态。
- 检查点 100：如果这个判断只来自 `EXPLAIN` 的 rows 字段，需要补充一个 catalog 或源码证据。
- 检查点 100：如果统计对象没有被选中，需要区分对象不存在、kind 未构建、子句不兼容和 inherit 标志不匹配。
