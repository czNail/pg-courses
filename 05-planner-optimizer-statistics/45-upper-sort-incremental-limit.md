# PostgreSQL Sort、Incremental Sort 与 Limit Path

## 课程定位

前置知识：理解 pathkeys、startup/total cost、ORDER BY、LIMIT/OFFSET、incremental sort 和 top-N sort 概念。

本节唯一主问题：

```text
ORDER BY / LIMIT 为什么会改变 startup cost 的重要性，已有 pathkeys、incremental sort、top-N heapsort 和 limit_tuples 如何让 planner 选择看似更贵但更快返回前几行的路径？
```

核心矛盾：ORDER BY 要求全局顺序，LIMIT 又可能只需要很少输出。planner 既不能破坏排序语义，也不能把“总成本最低”误当成“最快返回前几行”。

学完后应能解释 Sort、Incremental Sort、Gather Merge、Limit 计划节点背后的 `create_ordered_paths()` 与 `create_limit_path()` 决策。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节比较了 Window、Distinct 和 SetOp 对 ordering 的不同消费方式。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节属于 upper planning 的 ORDER BY / LIMIT 阶段。

它不展开 executor tuplesort 的内部算法，而是解释 planner 何时选择 full sort、incremental sort 或直接复用已有顺序。

理解这一节后，看到“更高 total cost 但更低 startup cost”的计划就不会立刻判断 planner 选错。

下一节讨论 partial path 如何经过 Gather 或 Gather Merge 进入 upper planning。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`create_ordered_paths()` 逐个检查输入 path 是否满足 `root->sort_pathkeys`，完全满足则直接复用，部分满足且允许 incremental sort 则构造 `IncrementalSortPath`，否则只给 cheapest input 加 full sort；`limit_tuples` 传入 sort cost 后影响 startup/total 比较，final 阶段再用 `create_limit_path()` 包装输出数量语义。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
全局排序语义
  vs
LIMIT 场景下尽快返回少量行的 startup 优先级
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/plan/planner.c | `preprocess_limit()`、`create_ordered_paths()`、final 阶段 `create_limit_path()`。 |
| 2 | src/backend/optimizer/util/pathnode.c | `create_sort_path()`、`create_incremental_sort_path()`、`create_limit_path()`。 |
| 3 | src/backend/optimizer/path/pathkeys.c | `pathkeys_count_contained_in()` 判断已有顺序满足程度。 |
| 4 | src/backend/optimizer/path/costsize.c | `cost_sort()`、`cost_incremental_sort()` 与 bounded sort 成本。 |
| 5 | src/include/nodes/pathnodes.h | `SortPath`、`IncrementalSortPath`、`LimitPath` 字段。 |
| 6 | src/backend/optimizer/plan/createplan.c | `create_sort_plan()`、`create_incrementalsort_plan()`、`create_limit_plan()`。 |
| 7 | src/include/nodes/plannodes.h | `Sort`、`IncrementalSort`、`Limit` Plan 字段。 |
| 8 | src/backend/utils/sort/tuplesort.c | 执行期 full sort / bounded sort 行为对照。 |
| 9 | src/backend/executor/nodeLimit.c | Limit 执行期如何消费 offset/count。 |
| 10 | src/backend/optimizer/README | pathkeys 与 sort ordering 在 optimizer 中的抽象。 |

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
| `root->sort_pathkeys` | ORDER BY 规范化后的目标顺序。 |
| `limit_tuples` | 已知 LIMIT/OFFSET 上界，用于 sort costing，不等同于最终返回行数状态。 |
| `tuple_fraction` | 调用者读取比例，影响低 startup path 的保留价值。 |
| `presorted_keys` | 输入 path 已经满足的前缀 pathkeys 数量。 |
| `SortPath.subpath` | 需要完整排序的输入 path。 |
| `IncrementalSortPath.nPresortedCols` | 可复用的前缀顺序数量。 |
| `LimitPath.limitOffset/limitCount` | final 阶段保存 LIMIT/OFFSET 表达式。 |
| `LimitPath.limitOption` | 区分普通 count、WITH TIES 等语义。 |
| `GatherMergePath.pathkeys` | 并行有序输出合并后的顺序保证。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`root->sort_pathkeys`**：ORDER BY 规范化后的目标顺序。

在 ORDER BY / LIMIT path中，这个字段决定 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_ordered_paths()/create_limit_path() 处观察它的值。

**`limit_tuples`**：已知 LIMIT/OFFSET 上界，用于 sort costing，不等同于最终返回行数状态。

这个字段要和 全局顺序、前缀顺序、WITH TIES 和 Limit 语义一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 pathkeys_count_contained_in、create_sort_path 和 create_incremental_sort_path 中已经发生过的候选保留或淘汰。

**`tuple_fraction`**：调用者读取比例，影响低 startup path 的保留价值。

它的诊断价值在于定位 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`presorted_keys`**：输入 path 已经满足的前缀 pathkeys 数量。

在 ORDER BY / LIMIT path中，这个字段决定 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_ordered_paths()/create_limit_path() 处观察它的值。

**`SortPath.subpath`**：需要完整排序的输入 path。

这个字段要和 全局顺序、前缀顺序、WITH TIES 和 Limit 语义一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 pathkeys_count_contained_in、create_sort_path 和 create_incremental_sort_path 中已经发生过的候选保留或淘汰。

**`IncrementalSortPath.nPresortedCols`**：可复用的前缀顺序数量。

它的诊断价值在于定位 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 把 LIMIT 理解成一定能下推到 scan。

- 以为 Incremental Sort 表示输入已经完全有序。

- 看到 total cost 较高就断定 LIMIT 查询选错计划。

- 把 Gather 和 Gather Merge 的顺序语义混淆。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
`grouping_planner()` 先用 `preprocess_limit()` 估算 offset/count，得到 `limit_tuples`。
若 ORDER BY 存在，`create_ordered_paths()` 创建 `UPPERREL_ORDERED`。
它遍历输入 relation 的 pathlist，读取每个 path 的 pathkeys。
`pathkeys_count_contained_in()` 返回是否完全有序以及已满足的前缀数量。
完全满足 ORDER BY 的 path 可直接进入 ordered_rel。
不完全满足时，非 cheapest path 只有在有 presorted prefix 且 incremental sort 开启时才值得考虑。
没有 presorted prefix 或 incremental sort 关闭时，planner 给 cheapest input 生成 full SortPath。
有 presorted prefix 时，planner 生成 IncrementalSortPath，成本按分组局部排序估算。
若 target expressions 不匹配，`apply_projection_to_path()` 在排序后补投影。
并行场景下，partial path 可先排序或增量排序，再通过 Gather Merge 保持全局顺序。
ordered_rel 返回后不必立即 `set_cheapest()`，因为 final 阶段会继续包装。
final 阶段如果 `limit_needed()`，在每个当前 path 上调用 `create_limit_path()`。
`create_plan()` 最终把 SortPath、IncrementalSortPath 和 LimitPath 转成对应 Plan 节点。
```

步骤 1：`grouping_planner()` 先用 `preprocess_limit()` 估算 offset/count，得到 `limit_tuples`。

这里要记录的是 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 如何改变，而不是只看函数是否返回。

断点优先打印 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段，它们能区分候选、剪枝和转换。

步骤 2：若 ORDER BY 存在，`create_ordered_paths()` 创建 `UPPERREL_ORDERED`。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_ordered_paths()/create_limit_path() 之前未生成，还是之后被剪枝。

步骤 3：它遍历输入 relation 的 pathlist，读取每个 path 的 pathkeys。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：`pathkeys_count_contained_in()` 返回是否完全有序以及已满足的前缀数量。

这里要记录的是 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 ORDER BY / LIMIT path有关的关键字段。

步骤 5：完全满足 ORDER BY 的 path 可直接进入 ordered_rel。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：不完全满足时，非 cheapest path 只有在有 presorted prefix 且 incremental sort 开启时才值得考虑。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：没有 presorted prefix 或 incremental sort 关闭时，planner 给 cheapest input 生成 full SortPath。

这里要记录的是 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 全局顺序、前缀顺序、WITH TIES 和 Limit 语义的那几项。

步骤 8：有 presorted prefix 时，planner 生成 IncrementalSortPath，成本按分组局部排序估算。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：若 target expressions 不匹配，`apply_projection_to_path()` 在排序后补投影。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段时才能完整解释。

步骤 10：并行场景下，partial path 可先排序或增量排序，再通过 Gather Merge 保持全局顺序。

这里要记录的是 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 如何改变，而不是只看函数是否返回。

断点优先打印 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段，它们能区分候选、剪枝和转换。

步骤 11：ordered_rel 返回后不必立即 `set_cheapest()`，因为 final 阶段会继续包装。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_ordered_paths()/create_limit_path() 之前未生成，还是之后被剪枝。

步骤 12：final 阶段如果 `limit_needed()`，在每个当前 path 上调用 `create_limit_path()`。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 13：`create_plan()` 最终把 SortPath、IncrementalSortPath 和 LimitPath 转成对应 Plan 节点。

这里要记录的是 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 ORDER BY / LIMIT path有关的关键字段。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| 排序 path 创建 | `create_sort_path()` 或 `create_incremental_sort_path()` 包装输入 subpath，仍属于 planner context。 |
| LIMIT path 创建 | `create_limit_path()` 在 final_rel 上包装当前 path，保存表达式和估算。 |
| Plan 转换 | `create_sort_plan()` 会要求 child 输出较小 tlist，`create_limit_plan()` 递归生成 subplan。 |
| 执行期资源 | Sort 执行期使用 tuplesort 和 work_mem，Limit 本身主要是状态机和计数。 |
| 清理 | planner path 随 planner context 释放，sort temp file 和 executor state 由执行器生命周期清理。 |

排序 path 创建阶段的重点是：`create_sort_path()` 或 `create_incremental_sort_path()` 包装输入 subpath，仍属于 planner context。

这份状态由 planner context 持有；只有 Sort/IncrementalSort/Limit Plan 字段 会进入后续 executor contract。

LIMIT path 创建阶段的重点是：`create_limit_path()` 在 final_rel 上包装当前 path，保存表达式和估算。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

Plan 转换阶段的重点是：`create_sort_plan()` 会要求 child 输出较小 tlist，`create_limit_plan()` 递归生成 subplan。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

执行期资源阶段的重点是：Sort 执行期使用 tuplesort 和 work_mem，Limit 本身主要是状态机和计数。

这份状态由 planner context 持有；只有 Sort/IncrementalSort/Limit Plan 字段 会进入后续 executor contract。

清理阶段的重点是：planner path 随 planner context 释放，sort temp file 和 executor state 由执行器生命周期清理。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| 全局顺序 | ORDER BY 的 pathkeys 必须完全满足，partial prefix 只能作为 incremental sort 的输入。 |
| WITH TIES | LIMIT WITH TIES 需要排序键支持边界判断，不能只按 count 截断。 |
| 投影顺序 | 排序键表达式必须在排序前可用，最终输出投影不能提前丢掉 resjunk 输入。 |
| 并行顺序 | 普通 Gather 不保证全局顺序，Gather Merge 才能保持 pathkeys。 |
| bounded sort | LIMIT 可影响 sort 成本，但不能改变 ORDER BY 的逻辑结果。 |
| incremental sort | 只复用 prefix，相同 prefix 内仍必须排序剩余 keys。 |

`全局顺序` 这一层保证的是：ORDER BY 的 pathkeys 必须完全满足，partial prefix 只能作为 incremental sort 的输入。

它只能约束 全局顺序、前缀顺序、WITH TIES 和 Limit 语义，不能代替成本比较、能力检查或 Plan 字段契约。

`WITH TIES` 这一层保证的是：LIMIT WITH TIES 需要排序键支持边界判断，不能只按 count 截断。

如果这个层面通过，仍要继续检查 full sort、incremental sort、bounded sort 和 Gather Merge，以及后续 createplan contract。

`投影顺序` 这一层保证的是：排序键表达式必须在排序前可用，最终输出投影不能提前丢掉 resjunk 输入。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`并行顺序` 这一层保证的是：普通 Gather 不保证全局顺序，Gather Merge 才能保持 pathkeys。

它只能约束 全局顺序、前缀顺序、WITH TIES 和 Limit 语义，不能代替成本比较、能力检查或 Plan 字段契约。

`bounded sort` 这一层保证的是：LIMIT 可影响 sort 成本，但不能改变 ORDER BY 的逻辑结果。

如果这个层面通过，仍要继续检查 full sort、incremental sort、bounded sort 和 Gather Merge，以及后续 createplan contract。

`incremental sort` 这一层保证的是：只复用 prefix，相同 prefix 内仍必须排序剩余 keys。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 未知 LIMIT | LIMIT/OFFSET 无法估算时，`limit_tuples` 为 -1，sort costing 回到无界估算。 |
| 无序 partial path | 并行 partial path 若无法通过 Gather Merge 保序，就不能满足顶层 ORDER BY。 |
| incremental sort 关闭 | 即使有 presorted prefix，GUC 关闭也会让 planner 回到 full sort 或跳过非 cheapest path。 |
| 宽 target | 若排序前保留过多列，sort 成本会增大；planner 通过 target placement 尽量避免。 |
| 估算偏差 | prefix group 数估错会让 incremental sort 成本偏离真实执行。 |

未知 LIMIT：LIMIT/OFFSET 无法估算时，`limit_tuples` 为 -1，sort costing 回到无界估算。

诊断这类路径时，优先用 full sort、incremental sort、bounded sort 和 Gather Merge确认它在生成前还是剪枝后消失。

无序 partial path：并行 partial path 若无法通过 Gather Merge 保序，就不能满足顶层 ORDER BY。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

incremental sort 关闭：即使有 presorted prefix，GUC 关闭也会让 planner 回到 full sort 或跳过非 cheapest path。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

宽 target：若排序前保留过多列，sort 成本会增大；planner 通过 target placement 尽量避免。

诊断这类路径时，优先用 full sort、incremental sort、bounded sort 和 Gather Merge确认它在生成前还是剪枝后消失。

估算偏差：prefix group 数估错会让 incremental sort 成本偏离真实执行。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| startup cost | LIMIT 场景中低 startup path 可能胜过低 total path。 |
| top-N | bounded sort 可以减少内存和比较成本，但需要已知 count。 |
| presorted prefix | prefix 越长，incremental sort 越可能比 full sort 便宜。 |
| work_mem | sort 超出内存会产生 temp file，planner 只能基于宽度和行数估算。 |
| target width | 排序前 tuple 越宽，CPU cache 和 I/O 压力越大。 |
| Gather Merge | 保序并行需要 worker 输出有序并支付 merge 成本。 |

startup cost：LIMIT 场景中低 startup path 可能胜过低 total path。

这个成本会先影响 cost_sort、cost_incremental_sort 和 startup/total cost 比较，再通过幸存 Path 传到下一阶段或 Plan。

top-N：bounded sort 可以减少内存和比较成本，但需要已知 count。

它不是执行毫秒数，而是候选之间排序的相对信号。

presorted prefix：prefix 越长，incremental sort 越可能比 full sort 便宜。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

work_mem：sort 超出内存会产生 temp file，planner 只能基于宽度和行数估算。

这个成本会先影响 cost_sort、cost_incremental_sort 和 startup/total cost 比较，再通过幸存 Path 传到下一阶段或 Plan。

target width：排序前 tuple 越宽，CPU cache 和 I/O 压力越大。

它不是执行毫秒数，而是候选之间排序的相对信号。

Gather Merge：保序并行需要 worker 输出有序并支付 merge 成本。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：EXPLAIN 中 `Sort Method: top-N heapsort` 通常说明 LIMIT 影响了执行期排序策略。

看到这个现象后，先回到pathkeys_count_contained_in、create_sort_path 和 create_incremental_sort_path，不要只根据节点名归因。

观测入口 2：`Incremental Sort` 节点会展示 presorted key 信息，能回到 `presorted_keys` 判断。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：普通 Gather 后若仍需 ORDER BY，常会看到 Gather 之上还有 Sort；Gather Merge 则可以保序。

节点名只是最终形态，真正原因通常在sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段的变化中。

观测入口 4：增加或删除索引会改变下层 pathkeys，从而改变上层是否需要 Sort。

看到这个现象后，先回到pathkeys_count_contained_in、create_sort_path 和 create_incremental_sort_path，不要只根据节点名归因。

观测入口 5：断点记录 `limit_tuples` 和 `root->sort_pathkeys`，能区分 LIMIT 只影响 costing 还是实际生成 LimitPath。

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

### sort、incremental sort 与 limit 的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`preprocess_limit()`

估算 count/offset，并将 caller tuple_fraction 调整为更贴近 LIMIT 场景。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段，再对照最终 EXPLAIN。

检查点 2：`root->sort_pathkeys`

ORDER BY 需求的规范化表示，是 create_ordered_paths 的目标。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`pathkeys_count_contained_in()`

判断输入 path 是否完全满足排序，或只满足前缀。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`presorted_keys`

incremental sort 的核心输入，表示已有前缀顺序数量。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 ORDER BY / LIMIT path的字段，再补 rows 和 cost。

检查点 5：`create_sort_path()`

没有可用前缀或 incremental sort 关闭时生成 full sort path。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查full sort、incremental sort、bounded sort 和 Gather Merge；若生成后消失，再看剪枝规则。

检查点 6：`create_incremental_sort_path()`

复用前缀顺序，仅对每个 prefix group 内的剩余 key 排序。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`limit_tuples`

传给 sort costing，用来估算 bounded sort 成本。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`create_gather_merge_path()`

partial path 需要保持全局排序时，通过 Gather Merge 合并 worker 有序输出。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`apply_projection_to_path()`

排序后 target 不匹配时补 projection，避免排序前过早丢表达式。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`create_limit_path()`

final 阶段保存 LIMIT/OFFSET 语义，不是任意下推工具。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段，再对照最终 EXPLAIN。

检查点 11：`create_sort_plan()`

Plan 构造时可能要求 child 输出 small tlist，减少排序数据宽度。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：`make_sort_from_pathkeys()`

把 pathkeys 转成 executor Sort key 数组。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：LIMIT 改变计划

同一 ORDER BY 查询加 LIMIT 后，低 startup path 可能胜出，即使 total cost 更高。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 create_ordered_paths()/create_limit_path() 验证。

节点名只是终点；本课要把原因写回sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段和源码状态。

案例 2：前缀索引触发 Incremental Sort

多列 ORDER BY 只有前缀索引时，planner 可能选择 Incremental Sort，而不是完全放弃索引顺序。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：Gather Merge 与 Sort

并行查询若需要最终顺序，普通 Gather 后可能还要 Sort；Gather Merge 则在汇总时保序。

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

信号 1：Incremental Sort 出现

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段字段，而不是停在 EXPLAIN 节点名。

信号 2：Sort Method 显示 top-N heapsort

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：Gather Merge 替代 Gather + Sort

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：LIMIT 改变 startup cost 比较

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段字段，而不是停在 EXPLAIN 节点名。

信号 5：ORDER BY 复用索引顺序

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：ORDER BY 前缀索引但不完全匹配

最小复现只保留能触发 ORDER BY / LIMIT path的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：ORDER BY LIMIT 与无 LIMIT 对照

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：并行 ORDER BY 查询

无关 join、函数和投影会干扰create_ordered_paths()/create_limit_path()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：宽 targetlist 排序再缩窄输出

最小复现只保留能触发 ORDER BY / LIMIT path的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `preprocess_limit`

在这个位置打印 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `create_ordered_paths`

不要无差别打印全部结构；只打印能解释 ORDER BY / LIMIT path的字段。

3. `pathkeys_count_contained_in`

记录前后差异，比记录单次静态值更有诊断价值。

4. `create_incremental_sort_path`

在这个位置打印 sort_pathkeys、limit_tuples、tuple_fraction、presorted_keys 和 LimitPath 字段，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `create_gather_merge_path`

不要无差别打印全部结构；只打印能解释 ORDER BY / LIMIT path的字段。

6. `create_limit_path`

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

- 把 LIMIT 理解成一定能下推到 scan。

纠正方法是把最终节点放回create_ordered_paths()/create_limit_path()的时间轴，确认对应状态何时改变。

- 以为 Incremental Sort 表示输入已经完全有序。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 看到 total cost 较高就断定 LIMIT 查询选错计划。

如果无法定位阶段，就不能直接写成确定结论。

- 把 Gather 和 Gather Merge 的顺序语义混淆。

纠正方法是把最终节点放回create_ordered_paths()/create_limit_path()的时间轴，确认对应状态何时改变。

- 忽略 target width 对 sort cost 的影响。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 top-N heapsort 是 planner 节点名称，而不是执行期 sort 方法。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：LIMIT startup 实验

对大表 `ORDER BY indexed_col LIMIT 10` 和无索引列排序做对比，观察 startup cost 和 Sort 节点。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：增量排序实验

建立多列索引只覆盖 ORDER BY 前缀，执行完整 ORDER BY，观察 Incremental Sort 是否出现。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：并行保序实验

调整并行参数，对 ORDER BY 查询观察 Gather 与 Gather Merge 的差异。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 ORDER BY / LIMIT 会让 startup cost 变得更重要？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. `limit_tuples` 影响 sort costing 和 `LimitPath` 保存语义有什么区别？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. Incremental Sort 依赖哪个 planner 状态判断 prefix？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. 为什么普通 Gather 不能替代 Gather Merge 的 ordering？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. target projection 放在 sort 前后各有什么成本和正确性影响？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 哪些现象只能从 EXPLAIN runtime sort method 看到，而不是 planner path 字段直接看到？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- ORDER BY / LIMIT 让 planner 同时比较全局顺序语义和前几行返回延迟。

- `create_ordered_paths()` 通过 pathkeys、presorted prefix 和 `limit_tuples` 生成 sort 或 incremental sort path。

- `create_limit_path()` 保存最终输出数量语义，不等同于任意下推。

- 并行保序必须依赖 Gather Merge，普通 Gather 会丢掉 pathkeys。

- 可迁移规律是：优化器中的 cost 不只有总量，startup、ordering 和语义边界会改变“最优”的定义。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
