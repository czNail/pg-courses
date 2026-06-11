# PostgreSQL UpperRel 阶段划分与 grouping_planner 管线

## 课程定位

前置知识：理解 base/join RelOptInfo、PathTarget、pathkeys、LIMIT tuple_fraction 和 add_path() 剪枝。

本节唯一主问题：

```text
base / join search 结束后，planner 为什么还要按 grouping、window、distinct、order、limit、modify table 等阶段构造 upper relation，而不是直接把节点接到 cheapest path 上？
```

核心矛盾：scan/join search 只解决 FROM/WHERE 的 relation 组合；SQL 顶层语义还包含聚合、窗口、去重、排序、行锁、LIMIT 和 DML 副作用。若过早生成 Plan，后续无法统一比较排序复用、投影位置、并行安全和 FDW upper pushdown。

学完后应能沿 `grouping_planner()` 判断一个计划节点是在 scan/join 阶段产生，还是作为 upperrel path 叠加，并能定位 upper 阶段之间的状态传递。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释了 join path 幸存规则。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节是 05 目录从 join search 进入 upper planning 的过渡课。

它不深入每个 upper 节点的算法，而是先解释为什么这些节点要先作为 Path 竞争，再转成 Plan。

如果把 GroupAggregate、WindowAgg、Unique、Sort 和 Limit 都当成 executor 节点来看，会错过 planner 仍在比较候选的事实。

下一节专门展开 aggregate upperrel 中 sorted、hashed、partial 和 mixed path 的选择。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`grouping_planner()` 先让 `query_planner()` 生成 scan/join 输入 rel，再把不同 SQL 顶层语义转换成一串 upper `RelOptInfo`；每个阶段消费上一个阶段的 pathlist，生成新的 pathlist，最后在 `UPPERREL_FINAL` 中加入 LockRows、Limit 或 ModifyTable。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
过早定型 Plan tree
  vs
保留 upper 语义之间的候选比较空间
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/plan/planner.c | `grouping_planner()` 主流程，upperrel 阶段调度核心。 |
| 2 | src/backend/optimizer/plan/planmain.c | `query_planner()` 返回 scan/join 输入 relation。 |
| 3 | src/backend/optimizer/util/relnode.c | `fetch_upper_rel()` 创建或取得 upper `RelOptInfo`。 |
| 4 | src/backend/optimizer/util/pathnode.c | Upper path 构造器，如 `create_limit_path()`、`create_lockrows_path()`、`create_modifytable_path()`。 |
| 5 | src/include/nodes/pathnodes.h | `UpperRelationKind`、`PlannerInfo.upper_targets` 和 upper path 结构。 |
| 6 | src/include/optimizer/planner.h | `create_upper_paths_hook` 扩展边界。 |
| 7 | src/backend/optimizer/prep/prepunion.c | `plan_set_operations()` 处理 set operation 的旁路入口。 |
| 8 | src/backend/optimizer/path/allpaths.c | scan/join 阶段完成后如何提供输入 path。 |
| 9 | src/backend/optimizer/path/pathkeys.c | sort、distinct、window 所需 pathkeys 的比较基础。 |
| 10 | src/backend/optimizer/plan/createplan.c | 最终把幸存 upper Path 递归转成 Plan。 |

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
| `current_rel` | `grouping_planner()` 中的阶段游标，始终指向当前可供下一阶段消费的 relation。 |
| `final_rel` | `UPPERREL_FINAL`，收束 LockRows、Limit、ModifyTable 和最终输出。 |
| `root->upper_targets[]` | 保存每个 upper 阶段期望输出的 `PathTarget`，也给扩展观察目标表达式。 |
| `limit_tuples` | 已知 LIMIT 对 sort costing 和 top-N 选择的估算输入。 |
| `tuple_fraction` | 调用者期望读取多少输出元组，影响低 startup path 的重要性。 |
| `scanjoin_target` | scan/join 顶部必须产出的表达式集合。 |
| `grouping_target` | 聚合或窗口之前需要保留的表达式集合。 |
| `sort_input_target` | ORDER BY 之前的输入目标，可能因 SRF 延迟投影而变化。 |
| `create_upper_paths_hook` | 扩展能向 upperrel 添加 Path 的边界，不允许破坏 SQL 语义。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`current_rel`**：`grouping_planner()` 中的阶段游标，始终指向当前可供下一阶段消费的 relation。

在 upperrel 阶段管线中，这个字段决定 current_rel、upper_targets、PathTarget 和 limit_tuples 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 grouping_planner()/fetch_upper_rel() 处观察它的值。

**`final_rel`**：`UPPERREL_FINAL`，收束 LockRows、Limit、ModifyTable 和最终输出。

这个字段要和 GROUP/WINDOW/DISTINCT/ORDER/LIMIT 的阶段顺序一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 current_rel 替换、upperrel pathlist 和 final_rel 中已经发生过的候选保留或淘汰。

**`root->upper_targets[]`**：保存每个 upper 阶段期望输出的 `PathTarget`，也给扩展观察目标表达式。

它的诊断价值在于定位 current_rel、upper_targets、PathTarget 和 limit_tuples 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`limit_tuples`**：已知 LIMIT 对 sort costing 和 top-N 选择的估算输入。

在 upperrel 阶段管线中，这个字段决定 current_rel、upper_targets、PathTarget 和 limit_tuples 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 grouping_planner()/fetch_upper_rel() 处观察它的值。

**`tuple_fraction`**：调用者期望读取多少输出元组，影响低 startup path 的重要性。

这个字段要和 GROUP/WINDOW/DISTINCT/ORDER/LIMIT 的阶段顺序一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 current_rel 替换、upperrel pathlist 和 final_rel 中已经发生过的候选保留或淘汰。

**`scanjoin_target`**：scan/join 顶部必须产出的表达式集合。

它的诊断价值在于定位 current_rel、upper_targets、PathTarget 和 limit_tuples 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 以为 scan/join search 结束后 planner 已经选定完整计划。

- 把 upperrel 当成 executor 节点，而不是 planner 阶段 relation-like 状态。

- 认为 LIMIT 总能下推到 scan/join 阶段。

- 忽略 PathTarget 对 volatile 表达式和 SRF 的执行次数约束。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
`subquery_planner()` 进入 `grouping_planner()`，顶层 Query 已完成 rewrite 和 preprocessing。
`preprocess_limit()` 估算 LIMIT/OFFSET，写出 `tuple_fraction` 和 `limit_tuples`。
若存在 set operation，`plan_set_operations()` 先返回 setop result relation，再回到通用 ORDER/LIMIT 收尾。
普通 SELECT 先处理 GROUP BY、aggregate、window、targetlist 和 min/max aggregate 特例。
`query_planner()` 生成 FROM/WHERE 对应的 scan/join relation，作为 upper pipeline 输入。
`create_pathtarget()` 把最终 targetlist 转成 `PathTarget`，并计算 parallel safety。
planner 根据 sort、window、grouping、SRF 关系拆出 scanjoin、grouping、sort input 和 final target。
`apply_scanjoin_target_to_paths()` 调整 scan/join path 输出，使它能喂给上层节点。
如果有 grouping 或 aggregate，调用 `create_grouping_paths()` 生成 `UPPERREL_GROUP_AGG`。
如果有 window function，调用 `create_window_paths()` 生成 `UPPERREL_WINDOW`。
如果有 DISTINCT，调用 `create_distinct_paths()` 生成 `UPPERREL_DISTINCT`。
如果有 ORDER BY，调用 `create_ordered_paths()` 生成 `UPPERREL_ORDERED`。
最后创建 `UPPERREL_FINAL`，在每个幸存 path 上叠加 LockRows、Limit 或 ModifyTable。
调用者再对 final rel 执行 `set_cheapest()`，然后 `create_plan()` 消费最终 best path。
```

步骤 1：`subquery_planner()` 进入 `grouping_planner()`，顶层 Query 已完成 rewrite 和 preprocessing。

这里要记录的是 current_rel、upper_targets、PathTarget 和 limit_tuples 如何改变，而不是只看函数是否返回。

断点优先打印 current_rel、upper_targets、PathTarget 和 limit_tuples，它们能区分候选、剪枝和转换。

步骤 2：`preprocess_limit()` 估算 LIMIT/OFFSET，写出 `tuple_fraction` 和 `limit_tuples`。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 grouping_planner()/fetch_upper_rel() 之前未生成，还是之后被剪枝。

步骤 3：若存在 set operation，`plan_set_operations()` 先返回 setop result relation，再回到通用 ORDER/LIMIT 收尾。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：普通 SELECT 先处理 GROUP BY、aggregate、window、targetlist 和 min/max aggregate 特例。

这里要记录的是 current_rel、upper_targets、PathTarget 和 limit_tuples 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 upperrel 阶段管线有关的关键字段。

步骤 5：`query_planner()` 生成 FROM/WHERE 对应的 scan/join relation，作为 upper pipeline 输入。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：`create_pathtarget()` 把最终 targetlist 转成 `PathTarget`，并计算 parallel safety。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：planner 根据 sort、window、grouping、SRF 关系拆出 scanjoin、grouping、sort input 和 final target。

这里要记录的是 current_rel、upper_targets、PathTarget 和 limit_tuples 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 GROUP/WINDOW/DISTINCT/ORDER/LIMIT 的阶段顺序的那几项。

步骤 8：`apply_scanjoin_target_to_paths()` 调整 scan/join path 输出，使它能喂给上层节点。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：如果有 grouping 或 aggregate，调用 `create_grouping_paths()` 生成 `UPPERREL_GROUP_AGG`。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 current_rel、upper_targets、PathTarget 和 limit_tuples时才能完整解释。

步骤 10：如果有 window function，调用 `create_window_paths()` 生成 `UPPERREL_WINDOW`。

这里要记录的是 current_rel、upper_targets、PathTarget 和 limit_tuples 如何改变，而不是只看函数是否返回。

断点优先打印 current_rel、upper_targets、PathTarget 和 limit_tuples，它们能区分候选、剪枝和转换。

步骤 11：如果有 DISTINCT，调用 `create_distinct_paths()` 生成 `UPPERREL_DISTINCT`。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 grouping_planner()/fetch_upper_rel() 之前未生成，还是之后被剪枝。

步骤 12：如果有 ORDER BY，调用 `create_ordered_paths()` 生成 `UPPERREL_ORDERED`。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 13：最后创建 `UPPERREL_FINAL`，在每个幸存 path 上叠加 LockRows、Limit 或 ModifyTable。

这里要记录的是 current_rel、upper_targets、PathTarget 和 limit_tuples 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 upperrel 阶段管线有关的关键字段。

步骤 14：调用者再对 final rel 执行 `set_cheapest()`，然后 `create_plan()` 消费最终 best path。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| 创建 | 每个 upperrel 由 `fetch_upper_rel()` 在 planner context 中创建，阶段结束后由下个阶段消费。 |
| 持有 | upperrel 保存自己的 pathlist、partial_pathlist、reltarget 和 FDW/parallel flags。 |
| 转换 | 上一个 upperrel 的 Path 不会直接变成 Plan，而是作为下一个 upperrel Path 的 subpath。 |
| 扩展 | FDW 与 `create_upper_paths_hook` 可以添加 upper path，但必须遵守当前 upper kind 的输出语义。 |
| 释放 | 整个 planner context 生命周期结束时统一释放 upperrel 和 Path。 |

创建阶段的重点是：每个 upperrel 由 `fetch_upper_rel()` 在 planner context 中创建，阶段结束后由下个阶段消费。

这份状态由 planner context 持有；只有 final_rel 的 best path 与 top targetlist 会进入后续 executor contract。

持有阶段的重点是：upperrel 保存自己的 pathlist、partial_pathlist、reltarget 和 FDW/parallel flags。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

转换阶段的重点是：上一个 upperrel 的 Path 不会直接变成 Plan，而是作为下一个 upperrel Path 的 subpath。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

扩展阶段的重点是：FDW 与 `create_upper_paths_hook` 可以添加 upper path，但必须遵守当前 upper kind 的输出语义。

这份状态由 planner context 持有；只有 final_rel 的 best path 与 top targetlist 会进入后续 executor contract。

释放阶段的重点是：整个 planner context 生命周期结束时统一释放 upperrel 和 Path。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| 阶段语义 | 每个 upperrel 代表一个已经完成的 SQL 语义层，不能把 DISTINCT 和 ORDER BY 随意交换。 |
| target 边界 | PathTarget 决定某阶段必须计算哪些表达式，避免 volatile 表达式被错误重算。 |
| 并行安全 | upperrel 的 `consider_parallel` 必须同时考虑输入和本阶段表达式。 |
| FDW 边界 | 只有同一 FDW 负责全部输入时，才允许它考虑 upper pushdown。 |
| LIMIT 语义 | LIMIT 可影响 sort cost，但不能在聚合或窗口前错误下推成全局限制。 |
| DML 边界 | ModifyTable 只在 final 阶段接入，因为副作用必须建立在完整结果语义之后。 |

`阶段语义` 这一层保证的是：每个 upperrel 代表一个已经完成的 SQL 语义层，不能把 DISTINCT 和 ORDER BY 随意交换。

它只能约束 GROUP/WINDOW/DISTINCT/ORDER/LIMIT 的阶段顺序，不能代替成本比较、能力检查或 Plan 字段契约。

`target 边界` 这一层保证的是：PathTarget 决定某阶段必须计算哪些表达式，避免 volatile 表达式被错误重算。

如果这个层面通过，仍要继续检查 target 拆分、setop 旁路和 create_upper_paths_hook，以及后续 createplan contract。

`并行安全` 这一层保证的是：upperrel 的 `consider_parallel` 必须同时考虑输入和本阶段表达式。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`FDW 边界` 这一层保证的是：只有同一 FDW 负责全部输入时，才允许它考虑 upper pushdown。

它只能约束 GROUP/WINDOW/DISTINCT/ORDER/LIMIT 的阶段顺序，不能代替成本比较、能力检查或 Plan 字段契约。

`LIMIT 语义` 这一层保证的是：LIMIT 可影响 sort cost，但不能在聚合或窗口前错误下推成全局限制。

如果这个层面通过，仍要继续检查 target 拆分、setop 旁路和 create_upper_paths_hook，以及后续 createplan contract。

`DML 边界` 这一层保证的是：ModifyTable 只在 final 阶段接入，因为副作用必须建立在完整结果语义之后。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 无法实现 | 某些数据类型既不能排序又不能哈希时，grouping 或 distinct 阶段会报无法实现。 |
| SRF 拆分 | targetlist 中 SRF 需要拆分 target，否则投影顺序会改变输出语义。 |
| 并行阻断 | rowMarks、非 SELECT、unsafe target 或 LIMIT 表达式会让 parallel upper path 停止传播。 |
| hook 风险 | 扩展添加 upper path 时若 target 或 pathkeys 不匹配，错误可能推迟到 createplan 或 executor 暴露。 |
| set operation 特例 | UNION/INTERSECT/EXCEPT 通过 `prepunion.c` 生成输入，不能简单套普通 FROM/WHERE pipeline。 |

无法实现：某些数据类型既不能排序又不能哈希时，grouping 或 distinct 阶段会报无法实现。

诊断这类路径时，优先用 target 拆分、setop 旁路和 create_upper_paths_hook确认它在生成前还是剪枝后消失。

SRF 拆分：targetlist 中 SRF 需要拆分 target，否则投影顺序会改变输出语义。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

并行阻断：rowMarks、非 SELECT、unsafe target 或 LIMIT 表达式会让 parallel upper path 停止传播。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

hook 风险：扩展添加 upper path 时若 target 或 pathkeys 不匹配，错误可能推迟到 createplan 或 executor 暴露。

诊断这类路径时，优先用 target 拆分、setop 旁路和 create_upper_paths_hook确认它在生成前还是剪枝后消失。

set operation 特例：UNION/INTERSECT/EXCEPT 通过 `prepunion.c` 生成输入，不能简单套普通 FROM/WHERE pipeline。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| 阶段叠加 | 每个 upper 阶段都会复制或包装 path，候选数由上一阶段 pathlist 决定。 |
| 投影位置 | 过早计算宽表达式会增加 sort、hash、materialize 的 width 成本。 |
| 排序复用 | 已有 pathkeys 能减少上层 sort，upper pipeline 必须保留这些差异。 |
| LIMIT 传播 | `limit_tuples` 影响 sort costing，但只有在不会破坏语义时才传入。 |
| 并行路径 | partial path 必须经过 Gather/Gather Merge 或 final aggregate 才能成为完整结果。 |
| 扩展成本 | FDW upper pushdown 可减少本地执行成本，但远端估算错误会改变整体 path 排序。 |

阶段叠加：每个 upper 阶段都会复制或包装 path，候选数由上一阶段 pathlist 决定。

这个成本会先影响 target 宽度、pathkeys 复用和 upper path 包装，再通过幸存 Path 传到下一阶段或 Plan。

投影位置：过早计算宽表达式会增加 sort、hash、materialize 的 width 成本。

它不是执行毫秒数，而是候选之间排序的相对信号。

排序复用：已有 pathkeys 能减少上层 sort，upper pipeline 必须保留这些差异。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

LIMIT 传播：`limit_tuples` 影响 sort costing，但只有在不会破坏语义时才传入。

这个成本会先影响 target 宽度、pathkeys 复用和 upper path 包装，再通过幸存 Path 传到下一阶段或 Plan。

并行路径：partial path 必须经过 Gather/Gather Merge 或 final aggregate 才能成为完整结果。

它不是执行毫秒数，而是候选之间排序的相对信号。

扩展成本：FDW upper pushdown 可减少本地执行成本，但远端估算错误会改变整体 path 排序。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：EXPLAIN 中 GroupAggregate、WindowAgg、Unique、Sort、Limit 的顺序对应 upper pipeline 的阶段顺序。

看到这个现象后，先回到current_rel 替换、upperrel pathlist 和 final_rel，不要只根据节点名归因。

观测入口 2：改变 `enable_hashagg`、`enable_sort` 或 `enable_incremental_sort` 可以观察不同 upper path 是否幸存。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：加 `ORDER BY ... LIMIT` 后，观察 sort 节点成本中的 startup/total 差异。

节点名只是最终形态，真正原因通常在current_rel、upper_targets、PathTarget 和 limit_tuples的变化中。

观测入口 4：打开断点时，从 `grouping_planner()` 观察 `current_rel` 在每个阶段被替换。

看到这个现象后，先回到current_rel 替换、upperrel pathlist 和 final_rel，不要只根据节点名归因。

观测入口 5：`EXPLAIN` 看不到 `root->upper_targets[]`，需要 gdb 或临时日志才能确认 target 拆分。

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

### upperrel pipeline 的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`preprocess_limit()`

LIMIT/OFFSET 在 upper planning 开头被估算，结果影响 sort costing 和 tuple_fraction。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 current_rel、upper_targets、PathTarget 和 limit_tuples，再对照最终 EXPLAIN。

检查点 2：`query_planner()`

它只返回 scan/join 输入 relation，不负责顶层 GROUP BY、Window、DISTINCT、ORDER 和 LIMIT 的完整语义。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`create_pathtarget()`

把 processed targetlist 转成 PathTarget，后续阶段用它控制投影位置。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`make_sort_input_target()`

决定 ORDER BY 之前是否需要保留额外表达式，避免排序时丢失 key。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 upperrel 阶段管线的字段，再补 rows 和 cost。

检查点 5：`make_window_input_target()`

保证窗口函数所需 partition/order 表达式在 WindowAgg 前已经可用。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查target 拆分、setop 旁路和 create_upper_paths_hook；若生成后消失，再看剪枝规则。

检查点 6：`make_group_input_target()`

聚合输入只保留 group key、aggregate argument 和必要表达式，减少下层输出宽度。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`apply_scanjoin_target_to_paths()`

把 scan/join path 的输出调整为 upper 阶段需要的 target。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`root->upper_targets[]`

保存各 upper 阶段 target，核心代码少用，但对扩展和调试很有价值。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`fetch_upper_rel()`

upperrel 是 relation-like 搜索节点，不是 executor Plan 节点。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`create_upper_paths_hook`

扩展只能在语义已确定的 upper kind 上添加 path，不能重写阶段顺序。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 current_rel、upper_targets、PathTarget 和 limit_tuples，再对照最终 EXPLAIN。

检查点 11：`UPPERREL_FINAL`

最终收束 LockRows、Limit、ModifyTable，而不是所有 upper 操作的同义词。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：setop 旁路

set operation 子树由 `prepunion.c` 先构造结果 relation，再回到通用收尾阶段。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：GROUP BY 之后再 ORDER BY

如果查询同时有 GROUP BY 和 ORDER BY，`current_rel` 会先变成 grouped_rel，再变成 ordered_rel。断点跟踪这个变量，比直接从最终 EXPLAIN 猜阶段顺序更可靠。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 grouping_planner()/fetch_upper_rel() 验证。

节点名只是终点；本课要把原因写回current_rel、upper_targets、PathTarget 和 limit_tuples和源码状态。

案例 2：SRF targetlist 拆分

targetlist 中有 set-returning function 时，planner 需要拆出多个 PathTarget，保证 SRF 在正确层级执行。这个问题不能靠 executor 自己猜。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：DML final 阶段

UPDATE/DELETE/MERGE 的 ModifyTable path 在 final_rel 中加入，因为副作用必须建立在完整输入结果之后。

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

信号 1：GroupAggregate 之后还有 Sort

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到current_rel、upper_targets、PathTarget 和 limit_tuples字段，而不是停在 EXPLAIN 节点名。

信号 2：WindowAgg 位于 Aggregate 之后

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：Distinct 出现在 Limit 之前

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：ModifyTable 位于计划顶层

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到current_rel、upper_targets、PathTarget 和 limit_tuples字段，而不是停在 EXPLAIN 节点名。

信号 5：set operation 子树没有普通 joinrel 形态

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：GROUP BY 后接 ORDER BY 和 LIMIT

最小复现只保留能触发 upperrel 阶段管线的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较current_rel、upper_targets、PathTarget 和 limit_tuples。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：窗口函数叠加 DISTINCT

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：UNION 子查询外层再排序

无关 join、函数和投影会干扰grouping_planner()/fetch_upper_rel()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：UPDATE ... FROM 带 RETURNING

最小复现只保留能触发 upperrel 阶段管线的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较current_rel、upper_targets、PathTarget 和 limit_tuples。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `grouping_planner`

在这个位置打印 current_rel、upper_targets、PathTarget 和 limit_tuples，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `query_planner`

不要无差别打印全部结构；只打印能解释 upperrel 阶段管线的字段。

3. `apply_scanjoin_target_to_paths`

记录前后差异，比记录单次静态值更有诊断价值。

4. `create_grouping_paths`

在这个位置打印 current_rel、upper_targets、PathTarget 和 limit_tuples，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `create_ordered_paths`

不要无差别打印全部结构；只打印能解释 upperrel 阶段管线的字段。

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

- 以为 scan/join search 结束后 planner 已经选定完整计划。

纠正方法是把最终节点放回grouping_planner()/fetch_upper_rel()的时间轴，确认对应状态何时改变。

- 把 upperrel 当成 executor 节点，而不是 planner 阶段 relation-like 状态。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 LIMIT 总能下推到 scan/join 阶段。

如果无法定位阶段，就不能直接写成确定结论。

- 忽略 PathTarget 对 volatile 表达式和 SRF 的执行次数约束。

纠正方法是把最终节点放回grouping_planner()/fetch_upper_rel()的时间轴，确认对应状态何时改变。

- 以为 FDW upper pushdown 是通用重写，而不是受 serverid、userid 和语义阶段约束。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 看到最终 Sort 就断定下层没有提供任何有序 path。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：阶段观察实验

构造包含 GROUP BY、Window、DISTINCT、ORDER BY、LIMIT 的查询，在 `grouping_planner()` 逐步打印 `current_rel->reloptkind` 和 pathlist 数量。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：投影位置实验

在 SELECT list 放入宽表达式并排序，对比是否出现额外 projection path，回到 `apply_projection_to_path()` 判断原因。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：并行传播实验

在并行可用表上执行聚合查询，再加入 parallel-unsafe 函数，观察 upperrel 的并行候选如何消失。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 upper operation 仍然先生成 Path，而不是直接生成 Plan？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. `current_rel` 在 `grouping_planner()` 中表达什么时间轴？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. LIMIT 为什么可以影响 sort costing，却不能任意下推过 aggregate？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. PathTarget 拆分如何保护 SRF 和 volatile 表达式语义？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. FDW upper pushdown 的语义边界是什么？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 哪个状态能从 EXPLAIN 看到，哪个只能在源码断点中确认？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- upperrel pipeline 把 SQL 顶层语义拆成可比较的 planner 阶段。

- `current_rel` 是阶段推进指针，`UPPERREL_FINAL` 是最终收束点。

- 每个阶段都消费上一阶段幸存 path，并保留对下一阶段有用的顺序、并行和投影信息。

- 异常路径主要来自无法实现的语义组合、SRF target 拆分、并行安全和扩展边界。

- 可迁移规律是：不要过早把搜索状态定型成执行状态；先让语义阶段完成可比候选压缩。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
