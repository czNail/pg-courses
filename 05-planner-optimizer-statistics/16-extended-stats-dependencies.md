# PostgreSQL Dependencies 与条件独立性修正

## 课程定位

前置知识：熟悉 `clauselist_selectivity()` 默认把多个 filter 选择率相乘，也知道扩展统计对象已经被加载到 `RelOptInfo.statlist`。

本节唯一主问题：

```text
当 `a = ? AND b = ?` 中 b 很大程度由 a 决定时，functional dependencies 如何避免把两个选择率机械相乘造成严重低估？
```

核心矛盾：planner 需要保留独立性相乘这个简单、快速、通用的默认模型；但现实数据中列之间经常相关，完全独立会把 rows 估算压得过低或抬得过高。

学完后应能判断 dependencies 能修正哪些 equality-like clause，不能修正哪些范围或 join 场景，以及为什么它给出的是概率混合修正而不是确定性约束。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节建立扩展统计对象的生命周期；本节只看 dependencies 如何修正多个过滤条件的独立性假设。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

后续课程会继续讲 multivariate MCV 和 ndistinct，分别处理组合热点值与多列 group 基数。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`ANALYZE` 从样本中计算多列 functional dependency degree；planner 在 `statext_clauselist_selectivity()` 中先尝试 MCV，再用 `dependencies_clauselist_selectivity()` 找到匹配 clause 的最强依赖，把独立相乘改成依赖度加权的选择率组合。
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
| 1 | `src/backend/statistics/dependencies.c` | `statext_dependencies_build()`、`dependency_degree()`、`dependencies_clauselist_selectivity()` 是主链路。 |
| 2 | `src/include/statistics/statistics.h` | `MVDependency`、`MVDependencies` 和选择率入口声明。 |
| 3 | `src/backend/statistics/extended_stats.c` | `statext_clauselist_selectivity()` 串联 MCV 与 dependencies。 |
| 4 | `src/backend/optimizer/path/clausesel.c` | `clauselist_selectivity_ext()` 决定何时启用扩展统计。 |
| 5 | `src/backend/optimizer/util/plancat.c` | `get_relation_statistics()` 加载 dependencies 类型的 `StatisticExtInfo`。 |
| 6 | `src/include/catalog/pg_statistic_ext.h` | `STATS_EXT_DEPENDENCIES` 对应 kind `f`。 |
| 7 | `src/include/catalog/pg_statistic_ext_data.h` | `stxddependencies` 保存序列化依赖数据。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：当 `a = ? AND b = ?` 中 b 很大程度由 a 决定时，functional dependencies 如何避免把两个选择率机械相乘造成严重低估？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

functional dependencies 统计描述的是“某些列值在样本中多大程度决定另一个列值”。

它不是数据库约束。

它也不是 join selectivity 模型。

| 状态 | 语义边界 |
| --- | --- |
| `MVDependency.degree` | 依赖成立程度，范围 0 到 1；1 也只是统计事实，不是约束证明。 |
| `MVDependency.attributes` | 前若干属性决定最后一个属性，顺序有语义。 |
| `MVDependencies.ndeps` | 一个统计对象中保存的依赖条目数。 |
| `StatisticExtInfo.keys` | planner 侧对象覆盖的普通列集合。 |
| `StatisticExtInfo.exprs` | planner 侧对象覆盖的表达式树。 |
| `list_attnums` | clauses 列表中每个 clause 对应的属性或表达式编号。 |
| `clauses_attnums` | 本次可用于 dependency 修正的属性集合。 |
| `estimatedclauses` | 已由 MCV 或 dependencies 消费的 clause 位图。 |
| `stxdinherit` | 区分 inherited 统计和普通 relation 统计。 |

构建端结构在 `src/include/statistics/statistics.h`。

`MVDependency` 的 `attributes` 最后一项是被决定的属性。

前面的属性组成 determinant。

例如 `(a,b) => c` 和 `(a,c) => b` 是不同依赖。

读取端在 `dependencies_clauselist_selectivity()` 中先把 clauses 映射到 attnums 或表达式编号。

表达式用负 attnum 临时表示，再通过 offset 放进 bitmapset。

这个技巧避免普通列 attnum 和表达式编号冲突。

`estimatedclauses` 是和 multivariate MCV 协作的边界。

`statext_clauselist_selectivity()` 先尝试 MCV。

dependencies 只处理剩余 clauses。

这保证组合热点值优先使用更具体的统计事实，dependencies 只做整体相关性修正。

## 5. 从 SQL 现象进入源码

dependencies 的典型 SQL 形态是同一张表上的多个 equality-like filter。

例如：

```sql
CREATE STATISTICS s_dep (dependencies) ON a, b FROM t;
ANALYZE t;
EXPLAIN SELECT * FROM t WHERE a = 10 AND b = 10;
```

如果 `b` 很大程度由 `a` 决定，单列选择率相乘会把 rows 压得太低。

dependencies 会把估算从完全独立模型拉回更接近依赖模型。

可观察入口：

```sql
SELECT statistics_name, dependencies
FROM pg_stats_ext
WHERE tablename = 't';
```

计划上，观察 base scan 的 estimated rows。

删除统计对象、重新 `ANALYZE` 或改写谓词形态后，rows 可能回到单列相乘。

dependencies 不适合这些场景：

```text
范围谓词，如 a BETWEEN ... AND ...；
跨表 join 条件；
OR clause 中的相关性；
只有一个匹配属性的 clause；
表达式树匹配不到统计对象中的表达式。
```

如果 multivariate MCV 覆盖了 `(a,b)` 的热点组合，dependencies 的额外效果可能不明显。

这不是 dependencies 没生效，而是更具体的 MCV 已经先消费了相关 clauses。

## 6. 主流程源码 walkthrough

构建链路在 `src/backend/statistics/dependencies.c`。

第一步，`BuildRelationExtStatistics()` 发现对象 kind 包含 `STATS_EXT_DEPENDENCIES`，调用 `statext_dependencies_build()`。

第二步，`statext_dependencies_build()` 从两个属性的依赖开始枚举，一直到覆盖对象中的所有属性。

对于三列 `(a,b,c)`，它会考虑：

```text
a => b
b => a
a => c
c => a
b => c
c => b
(a,b) => c
(a,c) => b
(b,c) => a
```

第三步，每个候选依赖交给 `dependency_degree()`。

该函数按候选属性构建 multi-sort support。

它把样本按所有相关属性排序。

然后按前 `k-1` 个属性分组。

如果组内最后一个属性只有一个值，该组支持该依赖。

如果组内最后一个属性出现多个值，该组违反该依赖。

最终 degree 是支持行数除以样本行数。

第四步，degree 为 0 的依赖不保存。

非零依赖写入 `MVDependency`。

构建过程用单独 memory context 承接每次 degree 计算的临时内存，并在每个候选后 reset。

第五步，dependencies 被序列化成 bytea，存入 `pg_statistic_ext_data.stxddependencies`。

读取链路从 `clauselist_selectivity_ext()` 开始。

第六步，`statext_clauselist_selectivity()` 先调用 multivariate MCV 选择率。

如果是 OR clause，functional dependencies 不继续处理。

第七步，`dependencies_clauselist_selectivity()` 检查 `RelOptInfo.statlist` 中是否存在 dependencies kind。

没有则返回 1.0，让外层选择率不受影响。

第八步，源码遍历 clauses，筛选 compatible equality-like clause 或匹配表达式。

不兼容、已被 MCV 消费或无法映射到属性的 clause 会跳过。

第九步，源码加载可匹配的 dependency 数据。

它要求 inherit 标记匹配，并且统计对象至少覆盖本次 clauses 中两个属性或表达式。

表达式依赖会被重新映射到本次 clauses 的编号空间。

第十步，`find_strongest_dependency()` 按最宽、最强策略挑选可用依赖。

每挑中一个依赖，就从 `clauses_attnums` 中移除被决定属性，避免重复应用。

第十一步，`clauselist_apply_dependencies()` 用 dependency degree 调整选择率。

源码注释给出的基本直觉是：

```text
P(a,b) = f * P(a) + (1-f) * P(a) * P(b)
```

`f` 越接近 1，越接近“b 由 a 决定”。

`f` 越接近 0，越接近普通独立相乘。

## 7. 生命周期 / ownership / cleanup

构建期的依赖计算只活在 `ANALYZE` 命令内。

`dependency_degree()` 中排序数组、group 计数和 multi-sort support 是临时对象。

`statext_dependencies_build()` 使用 `dependency_degree cxt` 反复 reset，避免候选组合越多内存越涨。

构建出来的 `MVDependencies` 在写入前是 C 结构。

`statext_dependencies_serialize()` 把它转成 catalog bytea。

planner 读取时，`statext_dependencies_load()` 从 catalog 反序列化。

`dependencies_clauselist_selectivity()` 用完后会释放反序列化结构。

`list_attnums`、`unique_exprs`、`clauses_attnums` 和中间 dependencies 指针数组都属于当前 planner context。

正常路径显式 pfree 一部分对象，剩余由 planning context 兜底。

`estimatedclauses` 的 owner 是外层 selectivity 计算。

dependencies 修改它，表示某些 clause 的选择率已经由扩展统计处理。

这不是 catalog 状态，只在一次选择率调用中有效。

## 8. 正确性机制层次

dependencies 的第一层边界是 clause 形态。

它主要处理 equality-like filter。

范围谓词、LIKE pattern、复杂表达式或 join 条件不符合依赖公式的前提。

第二层边界是同一 relation。

`RelOptInfo.statlist` 属于一个 base relation。

跨表相关性不是这个统计对象能表达的内容。

第三层边界是 AND 组合。

OR clause 下 dependencies 不应用，因为概率组合方式不同，源码在 MCV 后直接返回。

第四层边界是 degree 的解释。

degree 是样本观察到的依赖强度，不是约束。

即使 degree 接近 1，planner 也不能据此删除检查、改写 SQL 语义或推断唯一性。

第五层边界是与 MCV 协作。

MCV 能表达具体组合值的频率。

dependencies 表达整体函数依赖倾向。

同一 clause 被两者重复处理会把选择率修正过头。

第六层边界是表达式匹配。

表达式 dependencies 只有在查询表达式树与统计对象表达式匹配时才可用。

语义等价但结构不同的 SQL 不保证命中。

## 9. 错误路径 / 异常路径 / fallback

没有 dependencies kind 时，函数返回 1.0。

这让外层选择率保持普通单列组合结果。

匹配到的 clause 少于两个属性或表达式时，也返回 1.0。

没有兼容 equality-like clause 时，不做修正。

inherit 标记不匹配时跳过该统计对象。

统计对象覆盖的表达式与查询表达式不相等时，表达式依赖不参与。

反序列化后如果所有依赖都不能完全覆盖当前 clauses，函数不应用依赖。

已经被 MCV 消费的 clause 不会再被 dependencies 消费。

如果 dependency degree 为 0，构建阶段根本不会保存该依赖。

这些 fallback 都是“保持默认估算”而不是报错。

所以诊断问题时要看 rows 是否回到了独立相乘模型，而不是期待日志里出现明显错误。

## 10. 成本、资源与跨模块传播

构建 dependencies 的成本来自组合枚举和排序。

属性数越多，候选依赖越多。

`STATS_MAX_DIMENSIONS` 限制了最大维度，但高维对象仍然可能让 analyze 变重。

每个候选依赖都要对样本按相关属性排序或比较。

源码用临时 memory context reset 控制峰值，但 CPU 成本仍然存在。

planner 读取 dependencies 的成本来自：

```text
clause 兼容性检查；
表达式匹配；
反序列化 stxddependencies；
挑选最强依赖；
递归应用选择率公式。
```

收益集中在 base relation rows。

更准确的 rows 会影响 index path、join order、join method 和 upper planning。

但 dependencies 不会直接告诉 planner 该用哪个 join。

它只修正输入概率。

诊断顺序应当是：

```text
确认 pg_stats_ext.dependencies 有数据；
确认 clauses 是同一 relation 上的 equality-like AND 条件；
确认没有被 MCV 先消费；
确认 estimated rows 从独立相乘模型被拉回；
再观察 path 选择是否随 rows 改变。
```

本节的可迁移规律是：

```text
当默认模型为了通用性假设独立时，扩展统计用显式对象把“值得相信的相关性”带回 planner，但它仍然只是一种概率修正。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_stats_ext` 能看到 dependencies 摘要。

`EXPLAIN` 中多个等值 filter 的 estimated rows 是最直接的效果观察点。

删除统计对象或关闭扩展统计命中路径后，rows 往往回到单列相乘结果。

如果 multivariate MCV 已经覆盖组合热点，dependencies 的影响可能不明显。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

构造 `zip -> city` 这类强依赖数据，比较创建 dependencies 前后的 `where zip=? and city=?` rows。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

把 city 随机打散一部分，让 dependency degree 降低，观察 rows 修正幅度。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

把等值条件改成范围条件，验证 dependencies 不再命中。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

对表达式创建 dependencies，改写 SQL 表达式形态，观察是否仍能匹配。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

如果实验没有出现预期差异，先检查数据规模是否太小、统计是否刷新、GUC 是否强行禁用了候选路径。

## 14. 源码阅读练习

源码练习以断点和变量观察为主，不要求修改代码。

- 在 `src/backend/statistics/dependencies.c` 的入口函数附近设置断点，确认本节主路径是否进入。
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

为什么 dependencies 不应该被当成唯一约束或外键约束？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

多个扩展统计对象都覆盖同一组 clause 时，怎样选择更合适的对象？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

在维表编码和事实表过滤场景中，dependencies 能解决哪些估算问题，不能解决哪些 join 问题？

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

断点入口 1：`src/backend/statistics/dependencies.c`。`statext_dependencies_build()`、`dependency_degree()`、`dependencies_clauselist_selectivity()` 是主链路。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/include/statistics/statistics.h`。`MVDependency`、`MVDependencies` 和选择率入口声明。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/backend/statistics/extended_stats.c`。`statext_clauselist_selectivity()` 串联 MCV 与 dependencies。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/backend/optimizer/path/clausesel.c`。`clauselist_selectivity_ext()` 决定何时启用扩展统计。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/optimizer/util/plancat.c`。`get_relation_statistics()` 加载 dependencies 类型的 `StatisticExtInfo`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/include/catalog/pg_statistic_ext.h`。`STATS_EXT_DEPENDENCIES` 对应 kind `f`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 7：`src/include/catalog/pg_statistic_ext_data.h`。`stxddependencies` 保存序列化依赖数据。

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

反向检查 `MVDependency.degree`：依赖成立程度，范围 0 到 1。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `MVDependency.attributes`：前若干属性决定最后一个属性的依赖描述。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `MVDependencies.ndeps`：一个统计对象中保存的依赖条目数量。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `StatisticExtInfo.keys`：planner 侧用于匹配 clause attnums 的列集合。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `estimatedclauses`：避免同一个 clause 被 MCV 或 dependencies 重复估算。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `list_attnums`：dependencies 选择率中每个 clause 对应的属性或表达式编号。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `clauses_attnums`：本次可用于依赖修正的属性集合。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stxdinherit`：保证父表统计和普通表统计不混用。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`BuildRelationExtStatistics()` 遇到 kind `STATS_EXT_DEPENDENCIES` 时调用 `statext_dependencies_build()`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：`statext_dependencies_build()` 枚举属性组合，调用 `dependency_degree()` 计算依赖强度。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：结果被 `statext_dependencies_serialize()` 写成 bytea，存入 `pg_statistic_ext_data.stxddependencies`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：planner 通过 `get_relation_statistics()` 发现 dependencies 数据，加入 `RelOptInfo.statlist`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`clauselist_selectivity_ext()` 在 base rel 多个 restriction clause 上调用 `statext_clauselist_selectivity()`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：扩展统计先尝试 multivariate MCV，已被 MCV 估算的 clause 会写入 `estimatedclauses`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：`dependencies_clauselist_selectivity()` 过滤 equality-like clause 和匹配表达式。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：源码加载匹配统计对象，找覆盖当前 clause 集合的最强依赖。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 9：`clauselist_apply_dependencies()` 用 dependency degree 调整选择率，并递归处理更多属性。

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

记录对象：PostgreSQL Dependencies 与条件独立性修正。

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

课后复习 PostgreSQL Dependencies 与条件独立性修正 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/statistics/dependencies.c`：`statext_dependencies_build()`、`dependency_degree()`、`dependencies_clauselist_selectivity()` 是主链路。

- `src/include/statistics/statistics.h`：`MVDependency`、`MVDependencies` 和选择率入口声明。

- `src/backend/statistics/extended_stats.c`：`statext_clauselist_selectivity()` 串联 MCV 与 dependencies。

- `src/backend/optimizer/path/clausesel.c`：`clauselist_selectivity_ext()` 决定何时启用扩展统计。

- `src/backend/optimizer/util/plancat.c`：`get_relation_statistics()` 加载 dependencies 类型的 `StatisticExtInfo`。

- `src/include/catalog/pg_statistic_ext.h`：`STATS_EXT_DEPENDENCIES` 对应 kind `f`。

- `src/include/catalog/pg_statistic_ext_data.h`：`stxddependencies` 保存序列化依赖数据。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

dependencies 是对条件独立性假设的概率修正。

它用 dependency degree 把完全依赖和完全独立之间的选择率组合起来。

判断它是否生效，要看统计对象、clause 形态、inherit 标记、表达式匹配和 MCV 是否已经先处理。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
