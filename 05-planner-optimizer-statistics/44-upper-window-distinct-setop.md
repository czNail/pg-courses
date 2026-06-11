# PostgreSQL WindowAgg、Distinct 与 SetOp Path

## 课程定位

前置知识：理解 pathkeys、PathTarget、WindowClause、DISTINCT 语义和 set operation 的基本 SQL 行为。

本节唯一主问题：

```text
window function、DISTINCT、UNION / INTERSECT / EXCEPT 需要哪些排序、哈希或分组语义，planner 如何在复用已有 pathkeys 和新增排序之间取舍？
```

核心矛盾：这些语义都依赖“相等”“顺序”或“分区”边界，但它们不是同一种节点。planner 要尽量复用已有 ordering，同时不能让 DISTINCT ON、window frame、set operation column identity 或 volatile targetlist 改变语义。

学完后应能从 WindowAgg、Unique、HashAggregate、SetOp、Append 等计划节点回到对应 upper planning 或 setop planning 入口。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲 aggregate upperrel 的策略选择。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节把几个看似相邻但语义不同的 upper/setop 机制放在一起，是为了比较它们如何消费 ordering。

它不展开 executor 中 WindowAgg 的 frame 执行，也不讲 SetOp 节点的全部运行期算法。

核心问题是 planner 在 Path 阶段如何决定是否排序、是否哈希、是否复用现有 pathkeys。

下一节聚焦 ORDER BY、Incremental Sort 和 LIMIT 对 startup cost 的影响。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`create_window_paths()` 在每个有用输入 path 上堆叠 WindowAgg 和必要 sort；`create_distinct_paths()` 在排序 unique 与 hash aggregate 间选择；set operation 由 `prepunion.c` 先构造 append/setop relation，再回到 `grouping_planner()` 做顶层 ORDER/LIMIT。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
复用已有顺序和哈希能力
  vs
保护 window、distinct 和 set operation 各自不同的语义边界
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/plan/planner.c | `create_window_paths()`、`create_one_window_path()`、`create_distinct_paths()`。 |
| 2 | src/backend/optimizer/prep/prepunion.c | `plan_set_operations()`、`generate_setop_tlist()`、`generate_append_tlist()`。 |
| 3 | src/backend/optimizer/util/pathnode.c | `create_windowagg_path()`、`create_unique_path()`、`create_setop_path()`、`create_agg_path()`。 |
| 4 | src/backend/optimizer/path/pathkeys.c | window、distinct、sort 所需 pathkeys 的构造与比较。 |
| 5 | src/backend/optimizer/path/costsize.c | Unique、Agg、SetOp、Sort 相关成本估算。 |
| 6 | src/include/nodes/pathnodes.h | `WindowAggPath`、`UniquePath`、`SetOpPath`、`RecursiveUnionPath`。 |
| 7 | src/include/nodes/plannodes.h | `WindowAgg`、`Unique`、`SetOp`、`RecursiveUnion` Plan 节点。 |
| 8 | src/backend/optimizer/plan/createplan.c | `create_windowagg_plan()`、`create_unique_plan()`、`create_setop_plan()`。 |
| 9 | src/include/optimizer/prep.h | `plan_set_operations()` 对外声明。 |
| 10 | src/backend/optimizer/plan/setrefs.c | setop/output targetlist 后续修正边界。 |

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
| `activeWindows` | 经过优化和排序后的 WindowClause 列表，决定 WindowAgg 堆叠顺序。 |
| `root->window_pathkeys` | 当前 window 阶段最希望复用的输入顺序。 |
| `WindowAggPath.runcondition` | 可由 window function 推导出的运行条件。 |
| `root->distinct_pathkeys` | DISTINCT 或 DISTINCT ON 所需的去重顺序。 |
| `root->processed_distinctClause` | 去重表达式的规范化结果。 |
| `SetOpPath.strategy` | set operation 使用 sorted 还是 hashed 实现。 |
| `AppendPath` | UNION ALL 或 setop 子输入的合并路径。 |
| `generate_setop_tlist()` 输出 | set operation 为了列类型和 collation 统一而构造的 targetlist。 |
| `create_upper_paths_hook` | window/distinct 阶段允许扩展补充 path 的边界。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`activeWindows`**：经过优化和排序后的 WindowClause 列表，决定 WindowAgg 堆叠顺序。

在 window/distinct/setop ordering中，这个字段决定 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_window_paths()/create_distinct_paths()/plan_set_operations() 处观察它的值。

**`root->window_pathkeys`**：当前 window 阶段最希望复用的输入顺序。

这个字段要和 window order、DISTINCT ON 前缀和 setop 列身份一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 create_one_window_path、create_final_distinct_paths 和 prepunion.c 中已经发生过的候选保留或淘汰。

**`WindowAggPath.runcondition`**：可由 window function 推导出的运行条件。

它的诊断价值在于定位 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`root->distinct_pathkeys`**：DISTINCT 或 DISTINCT ON 所需的去重顺序。

在 window/distinct/setop ordering中，这个字段决定 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_window_paths()/create_distinct_paths()/plan_set_operations() 处观察它的值。

**`root->processed_distinctClause`**：去重表达式的规范化结果。

这个字段要和 window order、DISTINCT ON 前缀和 setop 列身份一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 create_one_window_path、create_final_distinct_paths 和 prepunion.c 中已经发生过的候选保留或淘汰。

**`SetOpPath.strategy`**：set operation 使用 sorted 还是 hashed 实现。

它的诊断价值在于定位 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 把所有去重都理解成同一个 Unique 节点。

- 以为 WindowAgg 自己会负责排序。

- 认为 DISTINCT ON 可以用 hash 实现。

- 把 UNION ALL 和 UNION 的 planner 路径混为一谈。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
普通 SELECT 进入 `grouping_planner()` 后，先通过 `find_window_functions()` 找 targetlist 中的 window function。
`select_active_windows()` 选择实际需要执行的 window clauses，并决定堆叠顺序。
若存在 window，`create_window_paths()` 创建 `UPPERREL_WINDOW`。
它会从 cheapest_total_path 和满足或部分满足 window pathkeys 的 path 出发。
`create_one_window_path()` 对每个 WindowClause 判断是否已有所需顺序。
不满足顺序时，planner 在 full sort 和 incremental sort 之间选择。
每层 WindowAggPath 会扩展或替换 PathTarget，保证后续窗口函数能看到前层输出。
DISTINCT 阶段创建 `UPPERREL_DISTINCT`，先尝试排序 Unique，再考虑 hash aggregate。
DISTINCT ON 必须遵守 ORDER BY 前缀语义，因此不能随意改成 hash。
parallel DISTINCT 需要先在 worker 内局部 distinct，再 Gather 后做最终 distinct。
set operation 不走普通 distinct upperrel 起点，而是在 `prepunion.c` 里构造 setop result relation。
UNION ALL 常以 Append/MergeAppend 组合子路径；INTERSECT/EXCEPT 可能生成 SetOpPath。
set operation result 回到 `grouping_planner()` 后，还可以继续处理顶层 ORDER BY 和 LIMIT。
```

步骤 1：普通 SELECT 进入 `grouping_planner()` 后，先通过 `find_window_functions()` 找 targetlist 中的 window function。

这里要记录的是 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 如何改变，而不是只看函数是否返回。

断点优先打印 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist，它们能区分候选、剪枝和转换。

步骤 2：`select_active_windows()` 选择实际需要执行的 window clauses，并决定堆叠顺序。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_window_paths()/create_distinct_paths()/plan_set_operations() 之前未生成，还是之后被剪枝。

步骤 3：若存在 window，`create_window_paths()` 创建 `UPPERREL_WINDOW`。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：它会从 cheapest_total_path 和满足或部分满足 window pathkeys 的 path 出发。

这里要记录的是 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 window/distinct/setop ordering有关的关键字段。

步骤 5：`create_one_window_path()` 对每个 WindowClause 判断是否已有所需顺序。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：不满足顺序时，planner 在 full sort 和 incremental sort 之间选择。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：每层 WindowAggPath 会扩展或替换 PathTarget，保证后续窗口函数能看到前层输出。

这里要记录的是 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 window order、DISTINCT ON 前缀和 setop 列身份的那几项。

步骤 8：DISTINCT 阶段创建 `UPPERREL_DISTINCT`，先尝试排序 Unique，再考虑 hash aggregate。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：DISTINCT ON 必须遵守 ORDER BY 前缀语义，因此不能随意改成 hash。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist时才能完整解释。

步骤 10：parallel DISTINCT 需要先在 worker 内局部 distinct，再 Gather 后做最终 distinct。

这里要记录的是 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 如何改变，而不是只看函数是否返回。

断点优先打印 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist，它们能区分候选、剪枝和转换。

步骤 11：set operation 不走普通 distinct upperrel 起点，而是在 `prepunion.c` 里构造 setop result relation。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_window_paths()/create_distinct_paths()/plan_set_operations() 之前未生成，还是之后被剪枝。

步骤 12：UNION ALL 常以 Append/MergeAppend 组合子路径；INTERSECT/EXCEPT 可能生成 SetOpPath。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 13：set operation result 回到 `grouping_planner()` 后，还可以继续处理顶层 ORDER BY 和 LIMIT。

这里要记录的是 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 window/distinct/setop ordering有关的关键字段。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| window path | 每个 WindowAggPath 持有 subpath、windowFuncs、runcondition 和 topwindow 标记。 |
| distinct path | UniquePath 或 AggPath 挂在 distinct upperrel 上，直到 createplan 生成 Unique 或 Agg。 |
| setop path | prepunion 生成的 relation 作为当前输入返回，之后仍由通用 final 阶段收尾。 |
| targetlist | setop targetlist 在 planner 中统一类型和 collation，后续 setrefs 再修正 varno。 |
| 释放 | 所有 Path 与 upperrel 都属于 planner context，executor 只接收最终 Plan 字段。 |

window path阶段的重点是：每个 WindowAggPath 持有 subpath、windowFuncs、runcondition 和 topwindow 标记。

这份状态由 planner context 持有；只有 WindowAgg、Unique/Agg 或 SetOp Plan 字段 会进入后续 executor contract。

distinct path阶段的重点是：UniquePath 或 AggPath 挂在 distinct upperrel 上，直到 createplan 生成 Unique 或 Agg。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

setop path阶段的重点是：prepunion 生成的 relation 作为当前输入返回，之后仍由通用 final 阶段收尾。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

targetlist阶段的重点是：setop targetlist 在 planner 中统一类型和 collation，后续 setrefs 再修正 varno。

这份状态由 planner context 持有；只有 WindowAgg、Unique/Agg 或 SetOp Plan 字段 会进入后续 executor contract。

释放阶段的重点是：所有 Path 与 upperrel 都属于 planner context，executor 只接收最终 Plan 字段。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| window 顺序 | WindowAgg 的 partition/order 语义必须由 pathkeys 或显式 sort 保证。 |
| window target | 输入 target 必须包含窗口计算所需 Vars、Aggs、partition key 和 order key。 |
| DISTINCT ON | 必须保留 DISTINCT 与 ORDER BY 的前缀关系，hash 不能表达“每组第一行”。 |
| setop 列身份 | set operation 子查询输出要统一列类型、collation 和 targetlist 位置。 |
| volatile 表达式 | 排序和投影阶段不能错误重复计算 volatile 表达式。 |
| parallel distinct | worker 内局部去重不能替代全局去重，Gather 后仍需要 final distinct。 |

`window 顺序` 这一层保证的是：WindowAgg 的 partition/order 语义必须由 pathkeys 或显式 sort 保证。

它只能约束 window order、DISTINCT ON 前缀和 setop 列身份，不能代替成本比较、能力检查或 Plan 字段契约。

`window target` 这一层保证的是：输入 target 必须包含窗口计算所需 Vars、Aggs、partition key 和 order key。

如果这个层面通过，仍要继续检查 DISTINCT ON 限制、window 多顺序和 setop 能力检查，以及后续 createplan contract。

`DISTINCT ON` 这一层保证的是：必须保留 DISTINCT 与 ORDER BY 的前缀关系，hash 不能表达“每组第一行”。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`setop 列身份` 这一层保证的是：set operation 子查询输出要统一列类型、collation 和 targetlist 位置。

它只能约束 window order、DISTINCT ON 前缀和 setop 列身份，不能代替成本比较、能力检查或 Plan 字段契约。

`volatile 表达式` 这一层保证的是：排序和投影阶段不能错误重复计算 volatile 表达式。

如果这个层面通过，仍要继续检查 DISTINCT ON 限制、window 多顺序和 setop 能力检查，以及后续 createplan contract。

`parallel distinct` 这一层保证的是：worker 内局部去重不能替代全局去重，Gather 后仍需要 final distinct。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 不可排序不可哈希 | DISTINCT 或 SetOp 所需类型若无法提供必要 equality/order 能力，会在 planner 阶段失败。 |
| DISTINCT ON 限制 | 即使 hash aggregate 可用，也不能用于 DISTINCT ON 的语义。 |
| window 多顺序 | 多个窗口需要不同 ordering 时，planner 只能按选择的 active window 顺序堆叠 sort/window。 |
| setop row locking | UNION/INTERSECT/EXCEPT 不允许顶层 FOR UPDATE/SHARE，planner 会拒绝。 |
| 估算退化 | distinct rows 估算来自 `estimate_num_groups()`，统计不足会影响 hash/sort 选择。 |

不可排序不可哈希：DISTINCT 或 SetOp 所需类型若无法提供必要 equality/order 能力，会在 planner 阶段失败。

诊断这类路径时，优先用 DISTINCT ON 限制、window 多顺序和 setop 能力检查确认它在生成前还是剪枝后消失。

DISTINCT ON 限制：即使 hash aggregate 可用，也不能用于 DISTINCT ON 的语义。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

window 多顺序：多个窗口需要不同 ordering 时，planner 只能按选择的 active window 顺序堆叠 sort/window。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

setop row locking：UNION/INTERSECT/EXCEPT 不允许顶层 FOR UPDATE/SHARE，planner 会拒绝。

诊断这类路径时，优先用 DISTINCT ON 限制、window 多顺序和 setop 能力检查确认它在生成前还是剪枝后消失。

估算退化：distinct rows 估算来自 `estimate_num_groups()`，统计不足会影响 hash/sort 选择。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| 已有顺序 | 窗口和 distinct 都会优先复用已有 pathkeys，减少 full sort。 |
| 增量排序 | 部分有序输入可用 incremental sort，成本取决于 presorted key 数和组分布。 |
| hash distinct | 不需要排序，但受内存、distinct rows 和 hashability 影响。 |
| setop append | UNION ALL 成本多由子路径累加，INTERSECT/EXCEPT 还要去重或计数。 |
| target width | window 堆叠会增加中间 tlist 宽度，影响后续 sort/materialize 成本。 |
| parallel 合并 | partial distinct 降低每 worker 数据量，但必须支付 Gather 和 final distinct。 |

已有顺序：窗口和 distinct 都会优先复用已有 pathkeys，减少 full sort。

这个成本会先影响 pathkeys 复用、incremental sort、hash distinct 和 setop append，再通过幸存 Path 传到下一阶段或 Plan。

增量排序：部分有序输入可用 incremental sort，成本取决于 presorted key 数和组分布。

它不是执行毫秒数，而是候选之间排序的相对信号。

hash distinct：不需要排序，但受内存、distinct rows 和 hashability 影响。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

setop append：UNION ALL 成本多由子路径累加，INTERSECT/EXCEPT 还要去重或计数。

这个成本会先影响 pathkeys 复用、incremental sort、hash distinct 和 setop append，再通过幸存 Path 传到下一阶段或 Plan。

target width：window 堆叠会增加中间 tlist 宽度，影响后续 sort/materialize 成本。

它不是执行毫秒数，而是候选之间排序的相对信号。

parallel 合并：partial distinct 降低每 worker 数据量，但必须支付 Gather 和 final distinct。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：EXPLAIN 中连续 Sort + WindowAgg 通常对应多个 window ordering 需求。

看到这个现象后，先回到create_one_window_path、create_final_distinct_paths 和 prepunion.c，不要只根据节点名归因。

观测入口 2：DISTINCT ON 计划常表现为 Sort + Unique，而不是 HashAggregate。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：UNION ALL 更可能看到 Append，INTERSECT/EXCEPT 更可能看到 SetOp 或 HashSetOp 类结果。

节点名只是最终形态，真正原因通常在activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist的变化中。

观测入口 4：关闭 `enable_hashagg` 可迫使 DISTINCT 更偏向排序实现，便于观察路径差异。

看到这个现象后，先回到create_one_window_path、create_final_distinct_paths 和 prepunion.c，不要只根据节点名归因。

观测入口 5：断点停在 `create_one_window_path()`，记录 `presorted_keys`，可以解释 incremental sort 是否出现。

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

### window、distinct 与 setop 的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`find_window_functions()`

只从 processed targetlist 找窗口函数，ORDER BY 表达式已在 targetlist 中维护。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist，再对照最终 EXPLAIN。

检查点 2：`select_active_windows()`

选择实际执行窗口并排序，影响 WindowAgg 堆叠顺序。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`make_pathkeys_for_window()`

为每个 WindowClause 生成 partition/order 所需 pathkeys。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`create_one_window_path()`

在一个输入 path 上逐层添加 Sort/IncrementalSort/WindowAgg。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 window/distinct/setop ordering的字段，再补 rows 和 cost。

检查点 5：`WindowFunc.runCondition`

部分 window function 可提供运行条件，planner 把它转成 WindowAggPath 字段。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查DISTINCT ON 限制、window 多顺序和 setop 能力检查；若生成后消失，再看剪枝规则。

检查点 6：`create_final_distinct_paths()`

普通 DISTINCT 在排序 Unique 和 hash aggregate 之间比较。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`get_useful_pathkeys_for_distinct()`

尝试重排 DISTINCT pathkeys 以复用输入顺序。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`parse->hasDistinctOn`

DISTINCT ON 阻止 hash 实现，因为必须保留每组首行语义。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`create_partial_distinct_paths()`

worker 局部去重后仍要 Gather 并做 final distinct。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`plan_set_operations()`

set operation 的主入口，不从普通 DISTINCT upperrel 起步。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist，再对照最终 EXPLAIN。

检查点 11：`generate_setop_tlist()`

统一 setop 输出列的类型、collation 和 varno 形态。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：`create_setop_path()`

INTERSECT/EXCEPT 等语义转成 SetOpPath 的入口。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：多个窗口排序

两个 window clause 使用不同 ordering 时，WindowAgg 之间可能夹着 Sort 或 Incremental Sort。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 create_window_paths()/create_distinct_paths()/plan_set_operations() 验证。

节点名只是终点；本课要把原因写回activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist和源码状态。

案例 2：DISTINCT ON 与 HashAggregate

普通 DISTINCT 可能用 HashAggregate，DISTINCT ON 通常需要 Sort + Unique。原因不是成本，而是语义。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：UNION ALL 与 UNION

UNION ALL 可以更像 Append，UNION 需要全局去重；两者在 `prepunion.c` 的路径分叉不同。

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

信号 1：多层 WindowAgg 中间夹 Sort

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist字段，而不是停在 EXPLAIN 节点名。

信号 2：DISTINCT ON 使用 Unique

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：普通 DISTINCT 使用 HashAggregate

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：UNION ALL 使用 Append

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist字段，而不是停在 EXPLAIN 节点名。

信号 5：INTERSECT / EXCEPT 使用 SetOp

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：两个不同 ORDER BY 的 window function

最小复现只保留能触发 window/distinct/setop ordering的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：DISTINCT 与 DISTINCT ON 对照

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：UNION ALL / UNION 对照

无关 join、函数和投影会干扰create_window_paths()/create_distinct_paths()/plan_set_operations()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：INTERSECT 与 EXCEPT 对照

最小复现只保留能触发 window/distinct/setop ordering的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `create_window_paths`

在这个位置打印 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `create_one_window_path`

不要无差别打印全部结构；只打印能解释 window/distinct/setop ordering的字段。

3. `create_distinct_paths`

记录前后差异，比记录单次静态值更有诊断价值。

4. `create_final_distinct_paths`

在这个位置打印 activeWindows、window_pathkeys、distinct_pathkeys、SetOpPath.strategy 和 setop tlist，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `plan_set_operations`

不要无差别打印全部结构；只打印能解释 window/distinct/setop ordering的字段。

6. `create_setop_path`

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

- 把所有去重都理解成同一个 Unique 节点。

纠正方法是把最终节点放回create_window_paths()/create_distinct_paths()/plan_set_operations()的时间轴，确认对应状态何时改变。

- 以为 WindowAgg 自己会负责排序。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 DISTINCT ON 可以用 hash 实现。

如果无法定位阶段，就不能直接写成确定结论。

- 把 UNION ALL 和 UNION 的 planner 路径混为一谈。

纠正方法是把最终节点放回create_window_paths()/create_distinct_paths()/plan_set_operations()的时间轴，确认对应状态何时改变。

- 忽略 set operation targetlist 的类型和 collation 对齐。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 看到 Sort 就断言下层没有 pathkeys；有时只是窗口或 DISTINCT 需要更严格顺序。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：窗口顺序实验

写两个 window function，使用不同 PARTITION/ORDER，观察是否出现多层 Sort/WindowAgg。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：DISTINCT ON 实验

比较 `DISTINCT` 与 `DISTINCT ON`，观察 hash aggregate 是否只在普通 DISTINCT 中出现。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：SetOp 实验

分别执行 UNION ALL、UNION、INTERSECT、EXCEPT，查看 Append、Unique、SetOp 组合，再回到 `prepunion.c` 对照。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 WindowAggPath 可能在一个输入 path 上堆叠多层？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. DISTINCT ON 为什么不能简单改成 HashAggregate？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. set operation 为什么要在 `prepunion.c` 中先统一 targetlist？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. partial distinct 为什么需要 final distinct？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. incremental sort 在 window 和 distinct 中各自解决什么问题？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 哪些信息最终能从 EXPLAIN 看到，哪些只能从 pathkeys 断点看到？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- Window、Distinct 和 SetOp 都使用 equality/order 信息，但语义边界不同。

- planner 的任务是在不破坏语义的前提下复用已有 pathkeys 或 hash 能力。

- `create_window_paths()`、`create_distinct_paths()` 和 `plan_set_operations()` 是三条不同入口。

- 错误路径集中在类型能力、DISTINCT ON 顺序语义、setop targetlist 和 parallel finalization。

- 可迁移规律是：相同物理工具可能服务不同语义，优化器必须先区分语义边界再谈成本复用。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
