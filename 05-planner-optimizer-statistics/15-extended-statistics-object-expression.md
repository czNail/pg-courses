# PostgreSQL 扩展统计对象与表达式统计

## 课程定位

前置知识：熟悉单列 `pg_statistic`、`ANALYZE` 主流程、表达式索引和 planner 中 `RelOptInfo.statlist` 的用途。

本节唯一主问题：

```text
为什么 PostgreSQL 要引入 `CREATE STATISTICS` 对象，而不是继续把所有信息塞进单列 `pg_statistic`？
```

核心矛盾：planner 需要理解多列相关性和表达式分布；但统计对象必须是 schema 对象，能被权限、依赖、ANALYZE、relcache 和 planner 生命周期共同管理。

学完后应能判断扩展统计对象在创建、采集、存储、加载和使用阶段分别保存什么状态，以及表达式统计为什么复用 `pg_statistic` 复合结构。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节结束了单列统计的成本影响；本节进入单列统计表达不了的多列和表达式分布。

05 目录的主线是 planner 如何把 SQL 语义、统计事实和成本模型压缩成可执行计划。本节只处理其中一个局部问题，不把相邻主题合并成百科式说明。

```text
SQL / Query 形态
  -> planner-local 状态
  -> 统计或约束参与判断
  -> Path rows / cost 改变
  -> Plan 节点体现选择结果
```

下一节会聚焦 functional dependencies，解释它如何修正多个 filter 选择率机械相乘。

因此，本节的阅读边界是明确的：只解释这一个判断如何成立、如何失败、如何被观测，不追求覆盖优化器的所有入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`CREATE STATISTICS` 在 `pg_statistic_ext` 保存对象定义，`ANALYZE` 读取这些定义并用同一批样本构建 ndistinct、dependencies、MCV 或表达式统计，再把序列化结果写入 `pg_statistic_ext_data`，planner 通过 `plancat.c` 加载成 `StatisticExtInfo`。
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
| 1 | `src/backend/commands/statscmds.c` | `CreateStatistics()`、`AlterStatistics()`、`RemoveStatisticsById()` 管理统计对象。 |
| 2 | `src/include/catalog/pg_statistic_ext.h` | `pg_statistic_ext` 保存对象定义、列集合、kind 和表达式树。 |
| 3 | `src/include/catalog/pg_statistic_ext_data.h` | `pg_statistic_ext_data` 保存 ANALYZE 后的数据。 |
| 4 | `src/backend/statistics/extended_stats.c` | `BuildRelationExtStatistics()`、`ComputeExtStatisticsRows()`、`compute_expr_stats()`、`statext_store()` 是构建链路。 |
| 5 | `src/backend/optimizer/util/plancat.c` | `get_relation_statistics()` 把 catalog 数据加载到 `RelOptInfo.statlist`。 |
| 6 | `src/include/nodes/pathnodes.h` | `StatisticExtInfo` 是 planner 侧的扩展统计描述。 |
| 7 | `src/backend/optimizer/path/clausesel.c` | `clauselist_selectivity_ext()` 调用扩展统计选择率。 |

推荐阅读方式是先从入口函数建立时间轴，再读结构体字段，最后读 fallback 和观测点。不要按文件名排序读，也不要在第一个复杂分支里停太久。

```text
入口函数
  -> 本地状态结构
  -> 判断条件
  -> fallback 或保守路径
  -> rows / cost / catalog / plan 的可见结果
```

- 读本节源码时，始终把问题压回这一句：为什么 PostgreSQL 要引入 `CREATE STATISTICS` 对象，而不是继续把所有信息塞进单列 `pg_statistic`？
- 如果一个分支不能改变这个问题的答案，只做定位提示，不展开成旁支。
- 如果一个字段看起来像结论，要继续追问它由哪个阶段写入、被哪个阶段读取。
- 如果 SQL 现象和源码注释看起来冲突，先确认当前路径是否进入 fallback。
- 如果不同版本实现细节变化，优先保留当前源码中能验证的不变量。

## 4. 关键数据结构与状态边界

扩展统计对象把统计单位从“单列”扩展到“同一 relation 上的一组列或表达式”。

它解决的问题不是取代 `pg_statistic`，而是补上单列摘要无法表达的相关性和表达式分布。

| 状态 | catalog / 内存位置 | 语义边界 |
| --- | --- | --- |
| `pg_statistic_ext.stxrelid` | catalog | 统计对象所属 relation。 |
| `pg_statistic_ext.stxkeys` | catalog | 参与统计的普通列 attnum 集合。 |
| `pg_statistic_ext.stxkind` | catalog | 请求构建的统计种类：ndistinct、dependencies、mcv、expressions。 |
| `pg_statistic_ext.stxexprs` | catalog | 统计对象包含的表达式树。 |
| `pg_statistic_ext_data.stxdinherit` | catalog | 普通统计和 inherited 统计的区分。 |
| `stxdndistinct` / `stxddependencies` / `stxdmcv` | catalog data | 多列 distinct、functional dependencies、multivariate MCV 的序列化数据。 |
| `stxdexpr` | catalog data | 表达式统计对应的单列统计 tuple 序列化结果。 |
| `StatisticExtInfo` | planner memory | `get_relation_statistics()` 加载到 `RelOptInfo.statlist` 的 planner-local 描述。 |

普通列统计在 `pg_statistic` 中按 `(relation, attnum, inherit)` 定位。

扩展统计对象分成两张 catalog：

```text
pg_statistic_ext
  保存对象定义：哪些列/表达式，哪些 kind，target 是多少

pg_statistic_ext_data
  保存 ANALYZE 后生成的数据：ndistinct、dependencies、mcv、expr stats
```

这个拆分让 DDL 定义和统计数据生命周期分离。

`CREATE STATISTICS` 只创建对象定义。

真正的数据要等 `ANALYZE` 构建。

表达式统计有一个特别边界。

它不是把表达式结果伪装成真实列写入 `pg_statistic`。

`BuildRelationExtStatistics()` 会对样本 tuple 计算表达式值，再把表达式统计序列化到 `pg_statistic_ext_data.stxdexpr`。

planner 读取时再通过表达式树匹配找到可用统计。

表达式树匹配是结构匹配，不是任意 SQL 等价证明。

## 5. 从 SQL 现象进入源码

扩展统计对象的外部入口是 `pg_stats_ext` 和 `pg_stats_ext_exprs`。

可以先确认对象是否存在：

```sql
CREATE STATISTICS s_ab (dependencies, mcv, ndistinct) ON a, b FROM t;
CREATE STATISTICS s_expr ON (lower(name)) FROM t;
ANALYZE t;

SELECT statistics_name, kinds, attnames, exprs
FROM pg_stats_ext
WHERE tablename = 't';

SELECT statistics_name, expr, most_common_vals, histogram_bounds
FROM pg_stats_ext_exprs
WHERE tablename = 't';
```

如果只执行 `CREATE STATISTICS` 没有 `ANALYZE`，planner 没有数据可读。

如果对象包含表达式，但查询写法经过不同 cast、collation、函数包装或等价变形，表达式树可能匹配不上。

如果对象只包含 `(a,b)`，它不能解释跨 relation join 的相关性。

扩展统计主要服务同一 base relation 上的 restriction clause。

一个计划现象是：

```text
没有扩展统计时，多个 filter 选择率机械组合；
创建并 ANALYZE 后，base scan rows 更接近真实；
join order 或 index path 随 base rows 改变。
```

但要注意优先级。

`statext_clauselist_selectivity()` 会先尝试 multivariate MCV。

functional dependencies 只修正剩余 AND clauses。

表达式统计则需要 `examine_variable()` 或 extended stats matching 找到同一表达式。

## 6. 主流程源码 walkthrough

构建链路从 `do_analyze_rel()` 调用 `ComputeExtStatisticsRows()` 开始。

第一步，`ComputeExtStatisticsRows()` 打开 `pg_statistic_ext`，取出当前 relation 的统计对象定义。

它只考虑本次 analyze 覆盖的列和表达式。

如果显式 `ANALYZE t(a)` 缺少对象需要的 `b`，这个对象不会抬高样本需求。

第二步，源码用 `statext_compute_stattarget()` 合成对象 target。

输入包括统计对象自己的 target、参与属性的 target 和默认 target。

返回值影响本次样本规模。

第三步，采样完成后，`BuildRelationExtStatistics()` 再次读取统计对象定义。

它创建 `BuildRelationExtStatistics` memory context，逐个对象构建。

第四步，`lookup_var_attr_stats()` 检查对象需要的列/表达式在本次 analyze 中是否可用。

不可用时，手动 analyze 通常会 warning，autovacuum 下避免噪声。

第五步，`make_build_data()` 为该对象准备统一的 `StatsBuildData`。

普通列值来自样本 tuple。

表达式值需要先求值，结果放在构建数据中。

第六步，按 `stxkind` 构建具体统计。

`STATS_EXT_NDISTINCT` 调 `statext_ndistinct_build()`。

`STATS_EXT_DEPENDENCIES` 调 `statext_dependencies_build()`。

`STATS_EXT_MCV` 调 `statext_mcv_build()`。

`STATS_EXT_EXPRESSIONS` 调表达式统计构建路径。

第七步，`statext_store()` 把结果序列化写入 `pg_statistic_ext_data`。

这一步把构建期 C 结构转成 bytea 或表达式 stats datum。

读取链路在 planner 侧。

`get_relation_statistics()` 从 catalog 加载对象定义和可用数据，构造 `StatisticExtInfo` 放入 `RelOptInfo.statlist`。

`clauselist_selectivity_ext()` 处理 base restriction clauses 时调用 `statext_clauselist_selectivity()`。

表达式统计则在 `examine_variable()` 和 extended stats 相关匹配函数中被识别。

## 7. 生命周期 / ownership / cleanup

扩展统计对象定义的 owner 是 catalog。

`CREATE STATISTICS`、`ALTER STATISTICS`、`DROP STATISTICS` 改变定义。

统计数据的 owner 是 `pg_statistic_ext_data`。

`ANALYZE` 按对象定义重新构建或保留旧数据。

构建期的 `StatsBuildData`、表达式求值结果、多列排序数组和 MCV/dependencies 结构都属于 analyze memory context。

写入 catalog 后，这些 C 指针不再有效。

planner 侧的 `StatisticExtInfo` 属于一次规划。

它引用或反序列化 catalog 数据，为 selectivity 函数服务。

规划结束后随 planner context 释放。

序列化数据读取时常有 detoast 和 palloc。

相应 free 逻辑由具体 stats 类型负责，例如 dependencies 和 MCV 的 load/free 路径。

表达式统计还涉及表达式树 ownership。

`stxexprs` 中保存的是解析后的表达式树表示。

匹配时不能用字符串比较 SQL 文本，要用 node tree 语义。

## 8. 正确性机制层次

扩展统计只描述同一 relation 内的统计关系。

它不是跨表约束，也不是 foreign key 或 unique constraint 的替代品。

对象定义必须覆盖当前 clauses。

如果 query 中只出现对象列的一部分，某些 stats kind 不能发挥作用。

表达式统计要求表达式匹配。

`lower(name)` 的统计不自动适用于 `lower(name)::text` 的所有变形，也不自动适用于语义等价但树形不同的表达式。

继承标记必须匹配。

父表 inherited 统计和单表统计不能混用，否则 rows 会把不同总体混在一起。

权限和安全边界仍然存在。

planner 读取统计可以改善估算，但不能越过 security barrier 或 leakproof 规则改变 qual 执行顺序。

多种 stats 同时存在时，要避免重复估算同一 clause。

`estimatedclauses` 位图就是为这个边界服务。

## 9. 错误路径 / 异常路径 / fallback

没有统计对象定义时，planner 使用普通单列统计。

有定义但没有 analyze 数据时，`RelOptInfo.statlist` 没有可用事实或具体 kind load 失败，仍回到普通估算。

对象 target 为 0 时，构建阶段会跳过该对象，保留或不刷新已有数据。

显式 analyze 列列表缺少对象所需列时，扩展统计不能构建。

表达式求值如果在安全受限上下文中不允许，会导致 analyze 报错或对象无法生成。

查询 clause 与表达式树匹配失败时，表达式统计不会命中。

OR clauses 下 functional dependencies 不适用，`statext_clauselist_selectivity()` 对 OR 在 MCV 后直接返回。

如果 multivariate MCV 已经估算了某些 clauses，dependencies 会跳过这些 clauses，避免双重修正。

诊断时要区分：

```text
对象不存在；
对象存在但未 ANALYZE；
对象数据存在但 kind 不适用；
kind 适用但 clause 匹配失败；
匹配成功但被更高优先级 stats 消费。
```

## 10. 成本、资源与跨模块传播

扩展统计的 analyze 成本高于单列统计。

样本需求可能被对象 target 抬高。

构建多列 MCV 需要组合值排序或频率统计。

dependencies 需要枚举属性组合并计算依赖强度。

表达式统计需要对样本 tuple 执行表达式。

catalog 成本也更高。

`pg_statistic_ext_data` 中的 bytea 可能较大。

planner 读取时需要加载、反序列化和匹配 clauses。

收益集中在 base relation rows。

更准确的 base rows 会继续影响：

```text
index path 是否值得生成；
join order 搜索；
join method cost；
aggregation/group rows；
upper planning 的输入规模。
```

扩展统计不是默认对所有列组合启用，是因为组合空间增长很快。

课程里的判断原则是：

```text
只有当单列独立假设造成可观测 rows 偏差，
且这种偏差稳定来自同一 relation 的列/表达式组合时，
扩展统计对象才是合适工具。
```

可迁移规律是：

```text
统计对象把“值得额外花成本描述的相关性”显式交给用户和 ANALYZE，而不是让 planner 猜所有组合。
```

## 11. 观测与诊断入口

planner 内部很多状态没有直接 SQL 视图，但可以通过计划形态和 catalog 摘要推断。

`pg_stats_ext` 能看到对象名、kind、列、表达式和部分统计摘要。

`pg_statistic_ext` 显示定义，`pg_statistic_ext_data` 显示已构建数据。

创建对象后不 analyze，`EXPLAIN` 通常不会变化。

表达式统计是否命中，可以通过改写 SQL 表达式形态和 rows 估算变化间接验证。

| 入口 | 能看到什么 | 看不到什么 |
| --- | --- | --- |
| `EXPLAIN` | 估算 rows、cost、Filter 位置和选中 path。 | 未入选的多数候选 path 和中间判断细节。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | 估算与实际的偏差、buffer 访问形态。 | planner 当时的所有统计 slot 读取过程。 |
| `pg_stats` / catalog | 统计摘要、目标、对象定义和部分扩展统计。 | 选择率函数如何组合这些摘要。 |
| gdb 断点 | 当前 `PlannerInfo`、`RelOptInfo`、slot 或依赖结构。 | 没有复现实验时难以判断业务代表性。 |

日常诊断通常先用 SQL 和 EXPLAIN 缩小范围，再用源码断点验证关键状态。

## 12. 课堂实验一：从计划反推状态

第一个实验只改变查询形态或统计状态，目标是让计划差异能回到本节主问题。

创建两列强相关表，先只 analyze 单列，再创建 `CREATE STATISTICS ... (dependencies, mcv)` 对比 rows。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

创建 `CREATE STATISTICS ... ON lower(email)`，比较 `where lower(email)=...` 的估算。

```text
EXPLAIN (ANALYZE, BUFFERS)
  -> 记录 estimated rows 与 actual rows
  -> 记录 Filter / Index Cond / Join Filter 的位置
  -> 对照 pg_stats 或 catalog 状态
```

实验结果不要求某个固定计划，因为硬件、数据量、GUC 和缓存都会影响 cost；要求是解释计划变化时能回到同一个源码判断。

## 13. 课堂实验二：改变输入验证边界

第二个实验改变输入分布或配置，用来验证 fallback 和成本传播。

只创建统计对象不 analyze，确认计划不变。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

删除统计对象或重置统计目标，观察 planner 回退到单列模型。

- 记录变更前的 catalog 摘要。
- 执行一次明确的 `ANALYZE`，避免把统计陈旧误认为源码行为。
- 只改变一个变量，保留其它 GUC 和 SQL 形态。
- 把计划差异写成“输入变化 -> planner 状态变化 -> path 变化”。

如果实验没有出现预期差异，先检查数据规模是否太小、统计是否刷新、GUC 是否强行禁用了候选路径。

## 14. 源码阅读练习

源码练习以断点和变量观察为主，不要求修改代码。

- 在 `src/backend/commands/statscmds.c` 的入口函数附近设置断点，确认本节主路径是否进入。
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

为什么扩展统计对象必须成为 schema 对象，而不是隐藏在 `pg_statistic` 行里？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

表达式统计和表达式索引分别解决什么问题？

回答时要给出一个可复现实验或一个源码入口，避免只给抽象观点。

在一个宽表上创建很多扩展统计对象，收益和维护成本如何评估？

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

断点入口 1：`src/backend/commands/statscmds.c`。`CreateStatistics()`、`AlterStatistics()`、`RemoveStatisticsById()` 管理统计对象。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 2：`src/include/catalog/pg_statistic_ext.h`。`pg_statistic_ext` 保存对象定义、列集合、kind 和表达式树。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 3：`src/include/catalog/pg_statistic_ext_data.h`。`pg_statistic_ext_data` 保存 ANALYZE 后的数据。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 4：`src/backend/statistics/extended_stats.c`。`BuildRelationExtStatistics()`、`ComputeExtStatisticsRows()`、`compute_expr_stats()`、`statext_store()` 是构建链路。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 5：`src/backend/optimizer/util/plancat.c`。`get_relation_statistics()` 把 catalog 数据加载到 `RelOptInfo.statlist`。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 6：`src/include/nodes/pathnodes.h`。`StatisticExtInfo` 是 planner 侧的扩展统计描述。

在这个位置不要一次性展开整份文件。先确认当前调用栈是否服务本节唯一主问题，再决定是否继续向下单步。

如果这里没有被命中，优先检查 SQL 形态、统计是否存在、GUC 是否关闭了相关路径，以及本节机制是否适用于当前 relation 类型。

如果这里被命中但状态为空，通常说明前置 catalog、样本、relids 或 clause 形态没有满足条件。这个结论比“planner 没优化”更有诊断价值。

断点入口 7：`src/backend/optimizer/path/clausesel.c`。`clauselist_selectivity_ext()` 调用扩展统计选择率。

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

反向检查 `pg_statistic_ext.stxrelid`：统计对象所属 relation。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pg_statistic_ext.stxkeys`：参与统计的普通列集合。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pg_statistic_ext.stxkind`：请求构建的统计类型：ndistinct、dependencies、MCV、expressions。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pg_statistic_ext.stxexprs`：表达式统计的表达式树列表。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `pg_statistic_ext_data.stxdinherit`：区分继承统计和非继承统计。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stxdndistinct` / `stxddependencies` / `stxdmcv`：序列化后的多变量统计数据。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `stxdexpr`：表达式统计数组，元素复用 `pg_statistic` 复合类型。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

反向检查 `StatisticExtInfo`：planner 侧按 kind 拆分后的可用统计对象。

先问这个状态有没有进入当前 planner 生命周期，再问它有没有被后续选择率、路径生成或 cost 计算读取。

如果它只在 catalog 中存在，却没有变成 planner-local 状态，问题多半在加载、匹配或安全边界。

如果它已经进入 planner-local 状态，但计划没有变化，继续找消费点，确认这个状态是否被 fallback、剪枝或更强约束覆盖。

## 19. 主流程中的可验证边界

下面把主流程拆成可验证边界。每个边界都应能用一个 SQL 变体、一个 catalog 查询或一个断点确认。

边界 1：`CreateStatistics()` 校验目标 relation、列、表达式、kind 和命名空间。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 2：对象定义写入 `pg_statistic_ext`，并记录对 relation、列、表达式函数和类型的依赖。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 3：`ANALYZE` 到达 `BuildRelationExtStatistics()` 时扫描目标表上的统计对象定义。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 4：`lookup_var_attr_stats()` 确认本次样本覆盖对象需要的列或表达式。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 5：`statext_compute_stattarget()` 决定对象自己的统计目标。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 6：`make_build_data()` 把样本 tuple 投影成多列或表达式值矩阵。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 7：按 `stxkind` 分别调用 ndistinct、dependencies、MCV 或表达式统计构建函数。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 8：`statext_store()` 把结果写入 `pg_statistic_ext_data`。

验证这个边界时，只改变一个输入因素。比如只改变统计目标、只改变 predicate 写法、只改变数据分布，避免把多个变化混在一起。

如果验证结果和预期不一致，不急着修改源码，先确认是否进入了相邻机制，例如扩展统计、默认选择率、join order 限制或成本参数分支。

这个边界的最终产物应该能在 rows、cost、Filter 位置、catalog 摘要或 debug 变量中看到一种对应变化。

边界 9：planner 打开 relation 时，`get_relation_info()` 通过 `get_relation_statistics()` 加载 `StatisticExtInfo`。

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

记录对象：PostgreSQL 扩展统计对象与表达式统计。

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

课后复习 PostgreSQL 扩展统计对象与表达式统计 时，可以把源码索引压缩成三轮阅读。

第一轮只看入口，确认本节主问题在哪里被提出。

第二轮只看状态，确认字段、catalog tuple 或局部结构在哪里被写入。

第三轮只看消费点，确认 rows、cost、path 或 plan 如何体现这个状态。

- `src/backend/commands/statscmds.c`：`CreateStatistics()`、`AlterStatistics()`、`RemoveStatisticsById()` 管理统计对象。

- `src/include/catalog/pg_statistic_ext.h`：`pg_statistic_ext` 保存对象定义、列集合、kind 和表达式树。

- `src/include/catalog/pg_statistic_ext_data.h`：`pg_statistic_ext_data` 保存 ANALYZE 后的数据。

- `src/backend/statistics/extended_stats.c`：`BuildRelationExtStatistics()`、`ComputeExtStatisticsRows()`、`compute_expr_stats()`、`statext_store()` 是构建链路。

- `src/backend/optimizer/util/plancat.c`：`get_relation_statistics()` 把 catalog 数据加载到 `RelOptInfo.statlist`。

- `src/include/nodes/pathnodes.h`：`StatisticExtInfo` 是 planner 侧的扩展统计描述。

- `src/backend/optimizer/path/clausesel.c`：`clauselist_selectivity_ext()` 调用扩展统计选择率。

复习时不要把这些源码入口平均用力。入口、状态和消费点能串起来，才算读完本节。

如果只能记住一个动作，就是从 `EXPLAIN` 中找第一个异常节点，再回到这里选最小源码入口。

## 25. 本节小结

扩展统计把单列 catalog 摘要提升成 relation 级 schema 对象。

对象定义、构建数据和 planner 加载是三个不同生命周期。

表达式统计让 planner 能估算表达式分布，但不提供访问路径；访问路径仍要靠索引或其它 path。

```text
runtime 现象
  -> planner 状态
  -> 源码判断
  -> rows / cost / plan 表现
  -> 可迁移诊断规律
```

本节结束时，应该能把一个计划现象翻译成源码中的状态判断，而不是只记住某个函数名。

下一节继续沿这个方向推进：用同样方法把相邻主题拆成一个主问题、一条状态链和一个可验证实验。
