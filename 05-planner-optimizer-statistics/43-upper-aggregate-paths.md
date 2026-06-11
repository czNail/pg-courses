# PostgreSQL GroupAggregate、HashAggregate 与 Mixed Aggregate Path

## 课程定位

前置知识：理解 PathTarget、GROUP BY、aggregate、pathkeys、work_mem 与 partial path 的基本含义。

本节唯一主问题：

```text
GROUP BY 和 aggregate 为什么可能选择排序聚合、哈希聚合或混合策略，分组基数、pathkeys、work_mem、partial aggregation 和 rollup 如何影响路径选择？
```

核心矛盾：聚合语义要求同组行被合并成稳定结果；实现上既可以依赖输入顺序，也可以构建 hash table，还可以在 grouping sets 中混合两者。不同策略在内存、排序、并行和溢出风险上的代价完全不同。

学完后应能从 EXPLAIN 中的 HashAggregate、GroupAggregate、Mixed Aggregate、Partial/Finalize Aggregate 回到 `create_grouping_paths()` 的选择条件。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节建立了 upperrel pipeline。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节只处理 aggregate upperrel，不展开 executor `nodeAgg.c` 的转移状态细节。

我们关注的是 planner 如何决定把哪种 `AggPath` 交给 createplan。

聚合是 upper planning 中最容易把 rows 估算、work_mem、pathkeys 和并行策略连在一起的节点。

下一节会讲 WindowAgg、DISTINCT 和 SetOp 如何在 ordering 与 hashing 之间取舍。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`create_grouping_paths()` 创建 `UPPERREL_GROUP_AGG`，根据 grouping 是否可排序、可哈希、是否支持 partial aggregate、是否有 grouping sets 和 partitionwise aggregate，向 grouped_rel 添加 sorted、hashed、mixed、partial/final aggregate path，再用 `set_cheapest()` 选出阶段候选代表。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
单一聚合语义
  vs
多种实现策略在内存、排序、并行和分组基数之间竞争
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/plan/planner.c | `create_grouping_paths()`、`create_ordinary_grouping_paths()`、`add_paths_to_grouping_rel()` 主流程。 |
| 2 | src/backend/optimizer/util/pathnode.c | `create_agg_path()`、`create_group_result_path()`、`create_projection_path()`。 |
| 3 | src/backend/optimizer/path/costsize.c | `cost_agg()`、hash table size 和排序成本来源。 |
| 4 | src/include/nodes/pathnodes.h | `AggPath`、`GroupingSetsPath`、`RollupData`、`GroupPathExtraData`。 |
| 5 | src/include/optimizer/pathnode.h | 聚合相关 Path 构造器声明。 |
| 6 | src/backend/optimizer/path/allpaths.c | partitionwise aggregate 和 partial path 相关辅助入口。 |
| 7 | src/backend/optimizer/util/tlist.c | targetlist 与 grouping expression 的表达式工具。 |
| 8 | src/backend/optimizer/plan/createplan.c | `create_agg_plan()`、`create_groupingsets_plan()` 转成 Plan。 |
| 9 | src/backend/executor/nodeAgg.c | 对照 executor 如何消费 planner 写入的 aggstrategy/aggsplit。 |
| 10 | src/include/nodes/plannodes.h | `Agg` Plan 节点字段。 |

推荐阅读顺序不是按文件名排序，而是先找入口，再找状态结构，再找 fallback 和 cleanup。

```text
入口函数
  -> 状态结构
  -> 候选生成或转换
  -> 剪枝 / fallback
  -> Plan 或 EXPLAIN 中的可见结果
```

本节涉及的文件都在当前本地源码中核对过。若未来 PostgreSQL 版本移动函数，优先保留这里的系统边界，再更新具体路径。

不要从函数顶部逐行读到尾。先问当前函数正在消费哪个 planner-local 状态，再决定是否需要进入它调用的辅助函数。

## 4. 关键数据结构与状态

| 状态 | 语义 |
| --- | --- |
| `AggClauseCosts` | 统计 aggregate transition、combine、serialize、deserialise、final function 的成本。 |
| `GroupPathExtraData.flags` | 记录可排序、可哈希、可 partial aggregate 等能力。 |
| `grouping_sets_data` | 保存 grouping sets、rollups、hashable/sortable 组合和估算组数。 |
| `AggPath.aggstrategy` | 区分 `AGG_SORTED`、`AGG_HASHED`、`AGG_MIXED` 等执行策略。 |
| `AggPath.aggsplit` | 区分 simple、partial、finalize 等聚合拆分模式。 |
| `AggPath.numGroups` | 分组基数估算，影响 hash table size 与输出 rows。 |
| `AggPath.transitionSpace` | transition state 空间估计，影响 hash 聚合内存压力。 |
| `partially_grouped_rel` | partial aggregate 的中间 upperrel，之后还需要 Gather 和 Finalize。 |
| `root->group_pathkeys` | 排序聚合可复用的输入顺序。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`AggClauseCosts`**：统计 aggregate transition、combine、serialize、deserialise、final function 的成本。

在 aggregate path 选择中，这个字段决定 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_grouping_paths()/add_paths_to_grouping_rel() 处观察它的值。

**`GroupPathExtraData.flags`**：记录可排序、可哈希、可 partial aggregate 等能力。

这个字段要和 grouping equality、HAVING 位置和 partial aggregate 能力一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 GROUPING_CAN_USE_* flags、partially_grouped_rel 和 create_agg_plan 中已经发生过的候选保留或淘汰。

**`grouping_sets_data`**：保存 grouping sets、rollups、hashable/sortable 组合和估算组数。

它的诊断价值在于定位 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`AggPath.aggstrategy`**：区分 `AGG_SORTED`、`AGG_HASHED`、`AGG_MIXED` 等执行策略。

在 aggregate path 选择中，这个字段决定 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_grouping_paths()/add_paths_to_grouping_rel() 处观察它的值。

**`AggPath.aggsplit`**：区分 simple、partial、finalize 等聚合拆分模式。

这个字段要和 grouping equality、HAVING 位置和 partial aggregate 能力一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 GROUPING_CAN_USE_* flags、partially_grouped_rel 和 create_agg_plan 中已经发生过的候选保留或淘汰。

**`AggPath.numGroups`**：分组基数估算，影响 hash table size 与输出 rows。

它的诊断价值在于定位 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 以为 GROUP BY 一定先排序。

- 以为 HashAggregate 只由 `enable_hashagg` 决定。

- 把 partial aggregate 当成 executor 自动优化，而不是 planner 显式生成的 Path。

- 忽略 HAVING 的阶段位置。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
`grouping_planner()` 判断存在 grouping、aggregate 或 HAVING，调用 `create_grouping_paths()`。
`get_agg_clause_costs()` 先收集 aggregate expression 的 CPU 与 transition 代价。
`make_grouping_rel()` 创建 `UPPERREL_GROUP_AGG`，设置 target、parallel 和 FDW 属性。
若是 degenerate grouping，planner 可以生成 `GroupResultPath`，不必消费原始输入行。
普通聚合先判断 grouping 是否 sortable，写入 `GROUPING_CAN_USE_SORT`。
再判断是否允许 hashed aggregation，ordered aggregate 和 ordered-set aggregate 会限制 hash。
`can_partial_agg()` 判断是否可以生成 partial aggregate。
`create_partial_grouping_paths()` 先尝试 partial path，为并行聚合准备中间 relation。
partitionwise aggregate 根据分区键和 GROUP BY 关系决定 full 或 partial 策略。
`gather_grouping_paths()` 把 partial grouped path 汇总成可 finalize 的完整路径。
`add_paths_to_grouping_rel()` 对输入 path 逐个尝试 sorted、hashed、mixed 方案。
grouping sets 中 `consider_groupingsets_paths()` 会在 sorted 和 mixed hash/sort 之间组合 rollup。
若没有任何实现，planner 报无法实现 GROUP BY，而不是交给 executor 失败。
FDW 与 upper hook 可追加聚合 path，最后 `set_cheapest(grouped_rel)` 完成阶段收束。
```

步骤 1：`grouping_planner()` 判断存在 grouping、aggregate 或 HAVING，调用 `create_grouping_paths()`。

这里要记录的是 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 如何改变，而不是只看函数是否返回。

断点优先打印 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags，它们能区分候选、剪枝和转换。

步骤 2：`get_agg_clause_costs()` 先收集 aggregate expression 的 CPU 与 transition 代价。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_grouping_paths()/add_paths_to_grouping_rel() 之前未生成，还是之后被剪枝。

步骤 3：`make_grouping_rel()` 创建 `UPPERREL_GROUP_AGG`，设置 target、parallel 和 FDW 属性。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：若是 degenerate grouping，planner 可以生成 `GroupResultPath`，不必消费原始输入行。

这里要记录的是 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 aggregate path 选择有关的关键字段。

步骤 5：普通聚合先判断 grouping 是否 sortable，写入 `GROUPING_CAN_USE_SORT`。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：再判断是否允许 hashed aggregation，ordered aggregate 和 ordered-set aggregate 会限制 hash。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：`can_partial_agg()` 判断是否可以生成 partial aggregate。

这里要记录的是 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 grouping equality、HAVING 位置和 partial aggregate 能力的那几项。

步骤 8：`create_partial_grouping_paths()` 先尝试 partial path，为并行聚合准备中间 relation。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：partitionwise aggregate 根据分区键和 GROUP BY 关系决定 full 或 partial 策略。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags时才能完整解释。

步骤 10：`gather_grouping_paths()` 把 partial grouped path 汇总成可 finalize 的完整路径。

这里要记录的是 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 如何改变，而不是只看函数是否返回。

断点优先打印 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags，它们能区分候选、剪枝和转换。

步骤 11：`add_paths_to_grouping_rel()` 对输入 path 逐个尝试 sorted、hashed、mixed 方案。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_grouping_paths()/add_paths_to_grouping_rel() 之前未生成，还是之后被剪枝。

步骤 12：grouping sets 中 `consider_groupingsets_paths()` 会在 sorted 和 mixed hash/sort 之间组合 rollup。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 13：若没有任何实现，planner 报无法实现 GROUP BY，而不是交给 executor 失败。

这里要记录的是 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 aggregate path 选择有关的关键字段。

步骤 14：FDW 与 upper hook 可追加聚合 path，最后 `set_cheapest(grouped_rel)` 完成阶段收束。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| 创建 | `AggPath` 在 planner context 中创建，subpath 指向输入 relation 的某个幸存 path。 |
| 持有 | `grouped_rel->pathlist` 持有完整聚合候选，`partially_grouped_rel` 持有 partial 候选。 |
| 转换 | `create_agg_plan()` 把 `groupClause`、`aggstrategy`、`aggsplit` 和 `numGroups` 写入 `Agg` Plan。 |
| 执行 | executor 在 `nodeAgg.c` 中根据 Plan 字段创建 transition state，planner 不持有运行期 state。 |
| 释放 | planner path 随 planner context 释放，executor transition state 属于执行期 memory context。 |

创建阶段的重点是：`AggPath` 在 planner context 中创建，subpath 指向输入 relation 的某个幸存 path。

这份状态由 planner context 持有；只有 Agg Plan 中的 aggstrategy、aggsplit 和 group key 会进入后续 executor contract。

持有阶段的重点是：`grouped_rel->pathlist` 持有完整聚合候选，`partially_grouped_rel` 持有 partial 候选。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

转换阶段的重点是：`create_agg_plan()` 把 `groupClause`、`aggstrategy`、`aggsplit` 和 `numGroups` 写入 `Agg` Plan。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

执行阶段的重点是：executor 在 `nodeAgg.c` 中根据 Plan 字段创建 transition state，planner 不持有运行期 state。

这份状态由 planner context 持有；只有 Agg Plan 中的 aggstrategy、aggsplit 和 group key 会进入后续 executor contract。

释放阶段的重点是：planner path 随 planner context 释放，executor transition state 属于执行期 memory context。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| 分组等价 | 排序聚合和哈希聚合必须使用同一组 grouping expressions 和 equality semantics。 |
| ordered aggregate | 带 ORDER BY 或 DISTINCT 的 aggregate 会限制 hash 聚合，因为 transition 输入顺序有语义。 |
| HAVING | HAVING qual 属于 grouped result 之后的过滤，不能当作 base restriction。 |
| partial aggregate | 只有 aggregate 支持 combine 等操作时，才能拆成 partial/final。 |
| grouping sets | 不同 grouping set 可能有不同 sortable/hashable 能力，mixed strategy 是语义保持下的实现折中。 |
| parallel safety | 输入、target、aggregate function 和 finalize 阶段都必须满足并行安全。 |

`分组等价` 这一层保证的是：排序聚合和哈希聚合必须使用同一组 grouping expressions 和 equality semantics。

它只能约束 grouping equality、HAVING 位置和 partial aggregate 能力，不能代替成本比较、能力检查或 Plan 字段契约。

`ordered aggregate` 这一层保证的是：带 ORDER BY 或 DISTINCT 的 aggregate 会限制 hash 聚合，因为 transition 输入顺序有语义。

如果这个层面通过，仍要继续检查 hashability、sortable、can_partial_agg 和 hashagg 内存估算，以及后续 createplan contract。

`HAVING` 这一层保证的是：HAVING qual 属于 grouped result 之后的过滤，不能当作 base restriction。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`partial aggregate` 这一层保证的是：只有 aggregate 支持 combine 等操作时，才能拆成 partial/final。

它只能约束 grouping equality、HAVING 位置和 partial aggregate 能力，不能代替成本比较、能力检查或 Plan 字段契约。

`grouping sets` 这一层保证的是：不同 grouping set 可能有不同 sortable/hashable 能力，mixed strategy 是语义保持下的实现折中。

如果这个层面通过，仍要继续检查 hashability、sortable、can_partial_agg 和 hashagg 内存估算，以及后续 createplan contract。

`parallel safety` 这一层保证的是：输入、target、aggregate function 和 finalize 阶段都必须满足并行安全。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 无可用实现 | 某些分组列类型只支持 hashing 或 sorting 的组合互相冲突时，会抛出无法实现 GROUP BY。 |
| hash 内存边界 | hash aggregate path 可以被内存估算限制，尤其 grouping sets 中可能回退到 sorted/mixed。 |
| partial 不可用 | aggregate 缺少 combine/serialize 支持时不会生成 partial aggregate。 |
| 分区聚合降级 | GROUP BY 不覆盖分区键时，full partitionwise aggregate 可能降为 partial 或完全不可用。 |
| 估算偏差 | numGroups 估错会让 HashAggregate 和 Sort+GroupAggregate 的成本比较失真。 |

无可用实现：某些分组列类型只支持 hashing 或 sorting 的组合互相冲突时，会抛出无法实现 GROUP BY。

诊断这类路径时，优先用 hashability、sortable、can_partial_agg 和 hashagg 内存估算确认它在生成前还是剪枝后消失。

hash 内存边界：hash aggregate path 可以被内存估算限制，尤其 grouping sets 中可能回退到 sorted/mixed。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

partial 不可用：aggregate 缺少 combine/serialize 支持时不会生成 partial aggregate。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

分区聚合降级：GROUP BY 不覆盖分区键时，full partitionwise aggregate 可能降为 partial 或完全不可用。

诊断这类路径时，优先用 hashability、sortable、can_partial_agg 和 hashagg 内存估算确认它在生成前还是剪枝后消失。

估算偏差：numGroups 估错会让 HashAggregate 和 Sort+GroupAggregate 的成本比较失真。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| 分组基数 | `numGroups` 越大，hash table 和输出 rows 成本越高。 |
| 输入顺序 | 已有 `group_pathkeys` 可直接支持 sorted aggregate，避免显式 Sort。 |
| work_mem | hash 聚合受 hash memory limit 影响，sort 聚合受 tuplesort 内存和 spill 影响。 |
| transition state | 宽 transition state 会放大 hash aggregate 内存和 CPU cache 压力。 |
| partial/final | partial aggregate 降低 worker 输出行数，但增加 combine/finalize 和 Gather 成本。 |
| grouping sets | 多组 grouping set 可能产生多层 sort/hash 组合，成本不再是单一 GROUP BY 的线性扩展。 |

分组基数：`numGroups` 越大，hash table 和输出 rows 成本越高。

这个成本会先影响 numGroups、work_mem、transitionSpace 与 partial/final 成本，再通过幸存 Path 传到下一阶段或 Plan。

输入顺序：已有 `group_pathkeys` 可直接支持 sorted aggregate，避免显式 Sort。

它不是执行毫秒数，而是候选之间排序的相对信号。

work_mem：hash 聚合受 hash memory limit 影响，sort 聚合受 tuplesort 内存和 spill 影响。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

transition state：宽 transition state 会放大 hash aggregate 内存和 CPU cache 压力。

这个成本会先影响 numGroups、work_mem、transitionSpace 与 partial/final 成本，再通过幸存 Path 传到下一阶段或 Plan。

partial/final：partial aggregate 降低 worker 输出行数，但增加 combine/finalize 和 Gather 成本。

它不是执行毫秒数，而是候选之间排序的相对信号。

grouping sets：多组 grouping set 可能产生多层 sort/hash 组合，成本不再是单一 GROUP BY 的线性扩展。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：EXPLAIN 中 `HashAggregate`、`GroupAggregate`、`MixedAggregate` 对应不同 `aggstrategy`。

看到这个现象后，先回到GROUPING_CAN_USE_* flags、partially_grouped_rel 和 create_agg_plan，不要只根据节点名归因。

观测入口 2：`Partial HashAggregate` 和 `Finalize Aggregate` 表示 planner 生成了 partial/final aggregate pipeline。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：修改 `work_mem`、`enable_hashagg`、`enable_sort` 能观察 path 选择边界。

节点名只是最终形态，真正原因通常在AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags的变化中。

观测入口 4：统计信息过旧时，`numGroups` 偏差会直接反映在 rows 估算和 hash aggregate 选择上。

看到这个现象后，先回到GROUPING_CAN_USE_* flags、partially_grouped_rel 和 create_agg_plan，不要只根据节点名归因。

观测入口 5：断点观察 `create_grouping_paths()` 的 flags，比只看最终计划更容易判断为什么某类 path 没生成。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

### 建议的诊断顺序

```text
EXPLAIN 观察最终 Plan
  -> 调整一个 GUC 或 schema 因素做对照
  -> 在入口函数断点确认候选是否生成
  -> 在剪枝或转换函数断点确认字段变化
  -> 回到 SQL 现象解释为什么计划改变或没有改变
```

`EXPLAIN (ANALYZE, BUFFERS)` 能把估算和实际行数放在一起，但它仍然看不到被丢弃的 Path。

如果问题是 planner 搜索过程，gdb、临时日志或最小化 SQL 往往比继续增加 EXPLAIN 选项更有效。

### aggregate path 选择的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`get_agg_clause_costs()`

先估算 aggregate transition、combine、final 等函数成本，为 sorted/hash/partial 比较提供输入。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags，再对照最终 EXPLAIN。

检查点 2：`make_grouping_rel()`

创建 `UPPERREL_GROUP_AGG`，并把 parallel 和 FDW 属性从输入 relation 传播上来。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`is_degenerate_grouping()`

HAVING 或空 grouping sets 的特殊路径可直接生成 Result 类 path。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`GROUPING_CAN_USE_SORT`

只有 grouping expression 可排序，sorted aggregate 才是候选。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 aggregate path 选择的字段，再补 rows 和 cost。

检查点 5：`GROUPING_CAN_USE_HASH`

hash aggregate 受 group hashability、ordered aggregate 和 ordered-set aggregate 限制。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查hashability、sortable、can_partial_agg 和 hashagg 内存估算；若生成后消失，再看剪枝规则。

检查点 6：`GROUPING_CAN_PARTIAL_AGG`

partial aggregate 必须得到 aggregate 函数能力支持。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`create_partial_grouping_paths()`

先生成 worker 局部聚合候选，为 Gather + Finalize 做准备。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`gather_grouping_paths()`

把 partial grouped partial paths 转成完整路径，之后才能 finalize。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`estimate_hashagg_tablesize()`

hash 聚合是否受内存约束，依赖 rows、width、numGroups 和 transition space。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`consider_groupingsets_paths()`

grouping sets 中 sorted、hashed、mixed 组合在这里展开。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags，再对照最终 EXPLAIN。

检查点 11：`add_paths_to_grouping_rel()`

对输入 path 逐个尝试聚合实现，最后进入 `add_path()` 竞争。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：`create_agg_plan()`

把 AggPath 的策略和 groupClause 转成 executor Agg 节点字段。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：HashAggregate 消失

即使 `enable_hashagg=on`，带 ORDER BY aggregate 或不可 hash 的 group expression 也会阻止 hash path 生成。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 create_grouping_paths()/add_paths_to_grouping_rel() 验证。

节点名只是终点；本课要把原因写回AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags和源码状态。

案例 2：Partial Aggregate 不出现

并行扫描存在并不保证 partial aggregate，aggregate 函数必须支持 combine/finalize。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：GroupAggregate 无显式 Sort

下层 pathkeys 若已经满足 group_pathkeys，sorted aggregate 可以复用顺序，EXPLAIN 中不会出现独立 Sort。

如果没有源码检查点，只能说是经验猜测，不能作为内核诊断结论。

最终报告要写字段变化，而不是只写“因为出现了某节点”。

### 最小诊断记录

建议在实验笔记中固定记录这些字段：

| 字段 | 记录方式 |
| --- | --- |
| SQL 形态 | 只保留触发本节机制的最小查询，避免无关 join、函数或索引干扰。 |
| 统计状态 | 记录是否刚执行 ANALYZE，以及关键列的 ndistinct、MCV 或 correlation 是否明显影响估算。 |
| GUC 差异 | 一次只改变一个开关，例如 enable_hashagg、enable_incremental_sort、parallel cost 或 work_mem。 |
| 候选入口 | 记录断点是否命中候选生成函数，避免把未生成误判为生成后输掉。 |
| 剪枝入口 | 记录 `add_path()` 或阶段性 `set_cheapest()` 后 pathlist 数量变化。 |
| 转换入口 | 记录 `create_plan_recurse()` 选择的 pathtype 和生成的 Plan 节点类型。 |
| 最终 EXPLAIN | 对比 estimated rows、actual rows、startup/total cost 和节点顺序。 |
| 推断边界 | 明确哪些结论来自源码断点，哪些只是从计划形态反推。 |

这份记录能把 SQL 现象、planner 状态和源码边界连接起来。没有这层记录，很容易把成本误差、搜索空间限制和 executor 运行期开销混为一谈。

### 推断边界

- 最终 Plan 只能证明胜出的路径是什么，不能证明其它候选从未出现。

- estimated rows 与 actual rows 的差异可以提示估算问题，但不能直接定位是哪一个 selectivity 函数造成。

- GUC 对照能缩小搜索空间，却不能替代源码断点对候选生成过程的确认。

- work_mem、parallel cost 和硬件缓存状态会改变相对收益，课程中的判断需要回到具体 workload 验证。

- 扩展 hook 或 FDW 可以添加候选 path；调试生产系统时要确认是否加载了改变 planner 行为的扩展。

- 不同 PostgreSQL 版本可能移动函数或调整 cost 细节，但 Path 到 upperrel 到 Plan 的边界仍是阅读主轴。

### 源码到 EXPLAIN 的闭环验证

下面这些信号适合把一个最终计划现象拆回 planner 状态。每个信号都要配合一个对照实验，否则只能算猜测。

信号 1：HashAggregate 与 GroupAggregate 切换

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags字段，而不是停在 EXPLAIN 节点名。

信号 2：Partial Aggregate 消失

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：Finalize Aggregate 出现在 Gather 上方

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：Grouping Sets 出现 Mixed 策略

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags字段，而不是停在 EXPLAIN 节点名。

信号 5：HAVING 过滤在聚合之后

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：普通 GROUP BY 单列聚合

最小复现只保留能触发 aggregate path 选择的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：GROUP BY 加 ordered aggregate

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：ROLLUP / GROUPING SETS

无关 join、函数和投影会干扰create_grouping_paths()/add_paths_to_grouping_rel()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：大表并行聚合并改变 work_mem

最小复现只保留能触发 aggregate path 选择的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `create_grouping_paths`

在这个位置打印 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `create_ordinary_grouping_paths`

不要无差别打印全部结构；只打印能解释 aggregate path 选择的字段。

3. `create_partial_grouping_paths`

记录前后差异，比记录单次静态值更有诊断价值。

4. `consider_groupingsets_paths`

在这个位置打印 AggPath.strategy、aggsplit、numGroups、transitionSpace 和 grouping flags，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `add_paths_to_grouping_rel`

不要无差别打印全部结构；只打印能解释 aggregate path 选择的字段。

6. `create_agg_plan`

记录前后差异，比记录单次静态值更有诊断价值。

调试结束后，把每个断点看到的字段变化按时间顺序写下来。只要时间轴清楚，最终计划的很多“反直觉”选择都会变成可以解释的状态推进。

### 读完源码后的自查

自查 1：能否不用函数清单，按时间顺序说出本节状态从哪里来、写到哪里去。

如果只能背函数名，说明还没有抓住本节主线。

自查 2：能否指出最终 EXPLAIN 中哪些节点只是结果，哪些字段才是 planner 决策的来源。

如果把节点名称当成全部因果，诊断会停在表面。

自查 3：能否说明一个 fallback 发生时，系统牺牲了哪类优化机会，又保住了哪条语义边界。

fallback 的价值在于保守推进，而不是生成看起来更简单的计划。

自查 4：能否用一个最小 SQL 复现本节现象，并通过一个 GUC、索引或统计信息改动让计划发生可解释变化。

不能复现的结论，不适合写进内核问题诊断报告。

自查 5：能否说清楚哪些判断依赖 workload、硬件或统计信息，哪些判断来自源码中的硬边界。

这个区分能避免把估算误差误写成 PostgreSQL 行为保证。

自查 6：能否指出本节和前后课程的连接点，并说明哪些内容应该留到下一节，不在本节展开。

课程边界清楚，源码阅读才不会变成横向铺开的资料清单。

## 11. 常见误区

- 以为 GROUP BY 一定先排序。

纠正方法是把最终节点放回create_grouping_paths()/add_paths_to_grouping_rel()的时间轴，确认对应状态何时改变。

- 以为 HashAggregate 只由 `enable_hashagg` 决定。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 把 partial aggregate 当成 executor 自动优化，而不是 planner 显式生成的 Path。

如果无法定位阶段，就不能直接写成确定结论。

- 忽略 HAVING 的阶段位置。

纠正方法是把最终节点放回create_grouping_paths()/add_paths_to_grouping_rel()的时间轴，确认对应状态何时改变。

- 把 grouping sets 当成多个普通 GROUP BY 的简单拼接。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 work_mem 只影响执行期，不影响 planner path cost。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：Hash 与 Sort 对照

对同一张表执行 GROUP BY，分别关闭 `enable_hashagg` 和降低 `work_mem`，观察 HashAggregate 与 GroupAggregate 切换。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：partial aggregate 实验

提高并行参数并使用大表聚合，观察 Partial/Finalize Aggregate 是否出现，再加入 parallel-unsafe 函数复测。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：grouping sets 实验

构造 ROLLUP 或 GROUPING SETS，断点停在 `consider_groupingsets_paths()`，记录哪些 rollup 被排序、哪些被 hash。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 ordered aggregate 会限制 hash aggregate？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. `AggClauseCosts` 对 planner 有什么作用，哪些成本仍然只是估算？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. partial aggregate 为什么需要 executor 函数级别的 combine 能力？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. GroupAggregate 什么时候可以避免显式 Sort？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. grouping sets 为什么会出现 mixed strategy？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. EXPLAIN 中看不到哪些 planner 阶段判断？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- aggregate upperrel 的核心是把同一聚合语义映射到多种实现 path。

- 排序、哈希、混合和 partial/final 不是 executor 随机选择，而是 planner 在 path 阶段比较后的结果。

- 关键状态包括 `AggClauseCosts`、`GroupPathExtraData`、`AggPath`、`numGroups` 和 `partially_grouped_rel`。

- 错误路径来自无法排序/哈希的类型组合、partial aggregate 能力不足和内存边界。

- 可迁移规律是：一个语义节点越重，planner 越需要保留多种物理实现直到足够晚的阶段再剪枝。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
