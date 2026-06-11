# PostgreSQL Scan、Join、Agg Plan 节点生成

## 课程定位

前置知识：理解 create_plan 入口、PathTarget、RestrictInfo、join qual、sort/group keys 和 executor Plan 节点基本结构。

本节唯一主问题：

```text
create_scan_plan()、create_join_plan()、create_agg_plan() 等函数如何把 Path 字段转成 executor 需要的 targetlist、qual、joinqual、hash clauses、sort keys 和子计划？
```

核心矛盾：Path 可以用抽象字段描述一种实现方式；executor 需要的是节点类型明确、输入输出 tlist 完整、qual 已分类、key 数组可直接执行的 Plan。转换必须保存语义，同时去掉 planner-only 状态。

学完后应能对照 createplan.c 中不同节点构造函数，判断某个 EXPLAIN 节点的 targetlist、qual、hash key、merge key 或 group key 是从哪个 Path 字段转换来的。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲 create_plan 入口如何从 best path 开始递归。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节继续 createplan，但焦点从入口转为节点构造。

它不讲 setrefs 的最终 varno 修正，也不展开 executor 运行期算法。

读这节时要始终问：这个 Plan 字段来自 Path、RelOptInfo、RestrictInfo，还是临时构造的 executor 辅助数组。

后续课程会继续 projection、resjunk、setrefs、SubPlan 和 executor contract。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`create_plan_recurse()` 先按 pathtype 分发；scan path 生成扫描 qual 和 tlist，join path 递归生成左右子计划并拆分 joinqual/otherqual/hash/merge clauses，agg/sort/window/limit path 把 strategy、keys、target 和 subplan 写入对应 Plan 节点，最后通用成本 rows 由 `copy_generic_path_info()` 复制。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
抽象候选字段
  vs
executor 直接执行所需的具体节点契约
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/plan/createplan.c | `create_scan_plan()`、`create_join_plan()`、`create_agg_plan()`、`create_sort_plan()`、`create_limit_plan()`。 |
| 2 | src/include/nodes/pathnodes.h | 各类 Path 字段来源，如 `IndexPath`、`JoinPath`、`AggPath`、`SortPath`。 |
| 3 | src/include/nodes/plannodes.h | executor Plan 节点字段，如 `Scan`、`Join`、`Agg`、`Sort`、`Limit`。 |
| 4 | src/backend/optimizer/util/clauses.c | 表达式处理与常量/qual 相关辅助。 |
| 5 | src/backend/optimizer/util/tlist.c | targetlist、sortgroupref 和 grouping cols 辅助。 |
| 6 | src/backend/optimizer/plan/setrefs.c | createplan 之后继续修正 Plan 引用。 |
| 7 | src/backend/optimizer/path/pathkeys.c | Sort/MergeJoin 从 pathkeys 生成 executor key 的基础。 |
| 8 | src/backend/optimizer/path/costsize.c | Path 成本已在构造 Plan 前估好。 |
| 9 | src/backend/executor/execProcnode.c | executor 按 Plan 节点类型初始化 PlanState。 |
| 10 | src/include/executor/executor.h | 执行器入口与 PlanState 初始化边界。 |

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
| `Path.pathtype` | 决定 create_plan_recurse 分发到哪类构造函数。 |
| `Path.pathtarget` | 通过 `build_path_tlist()` 转成 Plan targetlist。 |
| `RelOptInfo.baserestrictinfo` | base scan qual 的主要来源。 |
| `ParamPathInfo.ppi_clauses` | parameterized scan 需要额外执行的 join clauses。 |
| `JoinPath.joinrestrictinfo` | join qual 分类的输入。 |
| `HashPath.path_hashclauses` | 生成 HashJoin hashclauses、operators、collations 和 hash keys。 |
| `MergePath.path_mergeclauses` | 生成 MergeJoin mergeclauses 和 sort key 数组。 |
| `AggPath.groupClause` | 生成 Agg group column、operator、collation 数组。 |
| `SortPath.path.pathkeys` | 生成 Sort node 的 sort key 数组。 |
| `Plan.lefttree/righttree` | 递归生成的子计划连接点。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`Path.pathtype`**：决定 create_plan_recurse 分发到哪类构造函数。

在 Plan 节点构造中，这个字段决定 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan() 处观察它的值。

**`Path.pathtarget`**：通过 `build_path_tlist()` 转成 Plan targetlist。

这个字段要和 qual 分类、targetlist 完整性和 executor key 数组一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 build_path_tlist、make_sort_from_pathkeys 和 copy_generic_path_info 中已经发生过的候选保留或淘汰。

**`RelOptInfo.baserestrictinfo`**：base scan qual 的主要来源。

它的诊断价值在于定位 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`ParamPathInfo.ppi_clauses`**：parameterized scan 需要额外执行的 join clauses。

在 Plan 节点构造中，这个字段决定 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan() 处观察它的值。

**`JoinPath.joinrestrictinfo`**：join qual 分类的输入。

这个字段要和 qual 分类、targetlist 完整性和 executor key 数组一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 build_path_tlist、make_sort_from_pathkeys 和 copy_generic_path_info 中已经发生过的候选保留或淘汰。

**`HashPath.path_hashclauses`**：生成 HashJoin hashclauses、operators、collations 和 hash keys。

它的诊断价值在于定位 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 以为 Plan 节点只是 Path 节点改名。

- 把 Join Filter、Hash Cond、Merge Cond 的来源混为一谈。

- 认为 Sort 节点可以直接使用 pathkeys 而不用 key 数组。

- 忽略 parameterized path 到 Param 的重编码。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
`create_plan_recurse()` 读取 `best_path->pathtype`，分发到 scan、join、append、sort、agg、window、setop、limit 等构造函数。
`create_scan_plan()` 从 `best_path->parent` 取得 relation 状态。
普通 scan 使用 `baserestrictinfo`，index scan 可使用 `indrestrictinfo` 排除索引谓词已蕴含的 qual。
parameterized scan 会把 `ppi_clauses` 拼入 scan_clauses。
`get_gating_quals()` 找 pseudoconstant qual，必要时在 scan 上方插入 Result。
`build_path_tlist()` 把 PathTarget 表达式转成 TargetEntry，并在参数化路径中替换 nestloop params。
`create_join_plan()` 根据 Hash/Merge/NestLoop 分发。
Nested Loop 递归 outer 后设置 `curOuterRels`，再生成 inner plan，最后识别当前 NestLoopParam。
Merge Join 需要确认或构造左右排序，并把 mergeclauses 转成 executor 可用的 opfamily/collation/reversal 数组。
Hash Join 把 hashclauses 拆成 outer hash keys、inner hash keys、hash operators 和 collations。
`create_agg_plan()` 要求 child 提供带 label 的 tlist，然后从 groupClause 提取 group cols/operators/collations。
`create_sort_plan()` 和 `make_sort_from_pathkeys()` 根据 pathkeys 找到目标 tlist 中的排序表达式。
`create_limit_plan()` 保存 limitOffset、limitCount、limitOption，并递归生成 subplan。
每个节点构造完成后调用 `copy_generic_path_info()` 或相关 helper 复制 cost、rows、width 和 parallel flags。
```

步骤 1：`create_plan_recurse()` 读取 `best_path->pathtype`，分发到 scan、join、append、sort、agg、window、setop、limit 等构造函数。

这里要记录的是 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 如何改变，而不是只看函数是否返回。

断点优先打印 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys，它们能区分候选、剪枝和转换。

步骤 2：`create_scan_plan()` 从 `best_path->parent` 取得 relation 状态。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan() 之前未生成，还是之后被剪枝。

步骤 3：普通 scan 使用 `baserestrictinfo`，index scan 可使用 `indrestrictinfo` 排除索引谓词已蕴含的 qual。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：parameterized scan 会把 `ppi_clauses` 拼入 scan_clauses。

这里要记录的是 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 Plan 节点构造有关的关键字段。

步骤 5：`get_gating_quals()` 找 pseudoconstant qual，必要时在 scan 上方插入 Result。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：`build_path_tlist()` 把 PathTarget 表达式转成 TargetEntry，并在参数化路径中替换 nestloop params。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：`create_join_plan()` 根据 Hash/Merge/NestLoop 分发。

这里要记录的是 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 qual 分类、targetlist 完整性和 executor key 数组的那几项。

步骤 8：Nested Loop 递归 outer 后设置 `curOuterRels`，再生成 inner plan，最后识别当前 NestLoopParam。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：Merge Join 需要确认或构造左右排序，并把 mergeclauses 转成 executor 可用的 opfamily/collation/reversal 数组。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys时才能完整解释。

步骤 10：Hash Join 把 hashclauses 拆成 outer hash keys、inner hash keys、hash operators 和 collations。

这里要记录的是 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 如何改变，而不是只看函数是否返回。

断点优先打印 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys，它们能区分候选、剪枝和转换。

步骤 11：`create_agg_plan()` 要求 child 提供带 label 的 tlist，然后从 groupClause 提取 group cols/operators/collations。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan() 之前未生成，还是之后被剪枝。

步骤 12：`create_sort_plan()` 和 `make_sort_from_pathkeys()` 根据 pathkeys 找到目标 tlist 中的排序表达式。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 13：`create_limit_plan()` 保存 limitOffset、limitCount、limitOption，并递归生成 subplan。

这里要记录的是 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 Plan 节点构造有关的关键字段。

步骤 14：每个节点构造完成后调用 `copy_generic_path_info()` 或相关 helper 复制 cost、rows、width 和 parallel flags。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| 扫描节点 | Plan 节点创建后持有 tlist 和 qual，执行期初始化为对应 ScanState。 |
| 连接节点 | Join Plan 持有左右子计划、join qual、other qual 和算法专属 key。 |
| 聚合节点 | Agg Plan 持有 group key、strategy、split 和 transition space 估算，运行期再创建 AggState。 |
| 排序节点 | Sort Plan 持有 key 数组，执行期在 tuplesort 中分配资源。 |
| 统一释放 | Plan tree 属于 PlannedStmt 生命周期，执行期 PlanState 和工作内存由 executor context 清理。 |

扫描节点阶段的重点是：Plan 节点创建后持有 tlist 和 qual，执行期初始化为对应 ScanState。

这份状态由 planner context 持有；只有 Scan/Join/Agg/Sort/Limit Plan 字段 会进入后续 executor contract。

连接节点阶段的重点是：Join Plan 持有左右子计划、join qual、other qual 和算法专属 key。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

聚合节点阶段的重点是：Agg Plan 持有 group key、strategy、split 和 transition space 估算，运行期再创建 AggState。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

排序节点阶段的重点是：Sort Plan 持有 key 数组，执行期在 tuplesort 中分配资源。

这份状态由 planner context 持有；只有 Scan/Join/Agg/Sort/Limit Plan 字段 会进入后续 executor contract。

统一释放阶段的重点是：Plan tree 属于 PlannedStmt 生命周期，执行期 PlanState 和工作内存由 executor context 清理。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| qual 分类 | joinqual、otherqual、hashclauses、mergeclauses 不能混淆，否则 outer join 或 join algorithm 语义会错。 |
| 参数替换 | outer Vars 必须替换为 Params，inner plan 才能在 nested loop 中按外层行执行。 |
| 排序键 | Sort/MergeJoin key 数组必须匹配 targetlist 中的表达式和 collation。 |
| group key | Agg 的 groupColIdx、groupOperators、groupCollations 必须从 groupClause 和 child tlist 一致提取。 |
| pseudoconstant | 一次性 qual 通过 gating Result 执行，不能当普通 scan qual 重复求值。 |
| parallel flags | Plan 节点的 parallel_aware/parallel_safe 必须从 Path 正确传递。 |

`qual 分类` 这一层保证的是：joinqual、otherqual、hashclauses、mergeclauses 不能混淆，否则 outer join 或 join algorithm 语义会错。

它只能约束 qual 分类、targetlist 完整性和 executor key 数组，不能代替成本比较、能力检查或 Plan 字段契约。

`参数替换` 这一层保证的是：outer Vars 必须替换为 Params，inner plan 才能在 nested loop 中按外层行执行。

如果这个层面通过，仍要继续检查 gating Result、qpqual/indexqual 拆分和 merge input sort，以及后续 createplan contract。

`排序键` 这一层保证的是：Sort/MergeJoin key 数组必须匹配 targetlist 中的表达式和 collation。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`group key` 这一层保证的是：Agg 的 groupColIdx、groupOperators、groupCollations 必须从 groupClause 和 child tlist 一致提取。

它只能约束 qual 分类、targetlist 完整性和 executor key 数组，不能代替成本比较、能力检查或 Plan 字段契约。

`pseudoconstant` 这一层保证的是：一次性 qual 通过 gating Result 执行，不能当普通 scan qual 重复求值。

如果这个层面通过，仍要继续检查 gating Result、qpqual/indexqual 拆分和 merge input sort，以及后续 createplan contract。

`parallel flags` 这一层保证的是：Plan 节点的 parallel_aware/parallel_safe 必须从 Path 正确传递。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 未知 scan type | `create_scan_plan()` 遇到未识别 pathtype 会 ERROR。 |
| pathkeys 不匹配 | MergeJoin 或 Sort key 无法匹配 targetlist / equivalence class 时会报内部错误。 |
| 参数未替换 | NestLoopParam 漏分配会在 create_plan 入口收尾处被发现。 |
| 物理 tlist 失败 | 含 dropped column、system column 或 placeholder 时回退到普通 path tlist。 |
| mark/restore 限制 | MergeJoin inner sort 不用 incremental sort，因为 inner 侧需要 mark/restore。 |

未知 scan type：`create_scan_plan()` 遇到未识别 pathtype 会 ERROR。

诊断这类路径时，优先用 gating Result、qpqual/indexqual 拆分和 merge input sort确认它在生成前还是剪枝后消失。

pathkeys 不匹配：MergeJoin 或 Sort key 无法匹配 targetlist / equivalence class 时会报内部错误。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

参数未替换：NestLoopParam 漏分配会在 create_plan 入口收尾处被发现。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

物理 tlist 失败：含 dropped column、system column 或 placeholder 时回退到普通 path tlist。

诊断这类路径时，优先用 gating Result、qpqual/indexqual 拆分和 merge input sort确认它在生成前还是剪枝后消失。

mark/restore 限制：MergeJoin inner sort 不用 incremental sort，因为 inner 侧需要 mark/restore。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| tlist 宽度 | small tlist 可降低 sort/hash materialization 成本。 |
| hash join batching | 预计 batching 时 outer child 也倾向 small tlist，减少 batch file 体积。 |
| merge join sort | 显式 sort 或 materialize 的成本已计入 Path，createplan 只要生成节点并标注。 |
| gating Result | gating qual cost 已包含在子 plan cost 中，createplan 不重新估算。 |
| projection | 是否插入 Result/projection 影响 executor CPU，但 planner 已在 Path 阶段计入大部分成本。 |
| parallel | parallel flags 复制给 Plan 后影响 executor 初始化 parallel state。 |

tlist 宽度：small tlist 可降低 sort/hash materialization 成本。

这个成本会先影响 tlist width、projection 节点、key 数组和 generic path info，再通过幸存 Path 传到下一阶段或 Plan。

hash join batching：预计 batching 时 outer child 也倾向 small tlist，减少 batch file 体积。

它不是执行毫秒数，而是候选之间排序的相对信号。

merge join sort：显式 sort 或 materialize 的成本已计入 Path，createplan 只要生成节点并标注。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

gating Result：gating qual cost 已包含在子 plan cost 中，createplan 不重新估算。

这个成本会先影响 tlist width、projection 节点、key 数组和 generic path info，再通过幸存 Path 传到下一阶段或 Plan。

projection：是否插入 Result/projection 影响 executor CPU，但 planner 已在 Path 阶段计入大部分成本。

它不是执行毫秒数，而是候选之间排序的相对信号。

parallel：parallel flags 复制给 Plan 后影响 executor 初始化 parallel state。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：EXPLAIN 中 Hash Cond、Merge Cond、Join Filter 对应 createplan 中不同 qual 分类。

看到这个现象后，先回到build_path_tlist、make_sort_from_pathkeys 和 copy_generic_path_info，不要只根据节点名归因。

观测入口 2：Sort Key 和 Group Key 来自 createplan 对 pathkeys/groupClause 的数组化结果。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：Index Cond、Recheck Cond、Filter 等 scan 字段来自更细的 scan plan 构造函数。

节点名只是最终形态，真正原因通常在pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys的变化中。

观测入口 4：Nested Loop 的 inner Param 可以通过 verbose EXPLAIN 或 gdb 观察。

看到这个现象后，先回到build_path_tlist、make_sort_from_pathkeys 和 copy_generic_path_info，不要只根据节点名归因。

观测入口 5：断点停在 `copy_generic_path_info()` 能看到 Path cost 写入 Plan 的统一位置。

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

### Plan 节点构造的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`create_scan_plan()`

根据 pathtype 选择 seq/index/bitmap/tid/function/foreign/custom 等扫描节点构造。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys，再对照最终 EXPLAIN。

检查点 2：`baserestrictinfo`

普通 scan qual 来源；IndexPath 可能改用 `indrestrictinfo`。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`ppi_clauses`

parameterized scan 需要额外执行的外层 join clauses。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`get_gating_quals()`

pseudoconstant qual 被提升为 gating Result，而不是普通逐行 qual。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 Plan 节点构造的字段，再补 rows 和 cost。

检查点 5：`build_path_tlist()`

把 PathTarget exprs 转成 TargetEntry，并替换 nestloop params。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查gating Result、qpqual/indexqual 拆分和 merge input sort；若生成后消失，再看剪枝规则。

检查点 6：`create_join_plan()`

根据 Hash/Merge/NestLoop 分发，并在外层包装 gating Result。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`create_nestloop_plan()`

识别当前 NestLoopParam，并必要时把 PHV 加到 outer tlist。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`create_mergejoin_plan()`

把 mergeclauses 和 pathkeys 转成 opfamily、collation、reversal、nullsfirst 数组。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`create_hashjoin_plan()`

拆出 outer/inner hash keys、hash operators 和 collations。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`create_agg_plan()`

从 groupClause 和 child tlist 提取 group columns/operators/collations。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys，再对照最终 EXPLAIN。

检查点 11：`create_sort_plan()`

从 pathkeys 构造 executor Sort key，必要时要求 child small tlist。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：`copy_generic_path_info()`

节点专属字段设置完后，统一复制通用 cost 和 parallel 信息。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：Hash Cond 与 Join Filter 分离

Hash join 的 hashclauses 进入 Hash Cond，其它 join quals 可能成为 Join Filter；这来自 createplan 的分类，不是 executor 临时决定。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan() 验证。

节点名只是终点；本课要把原因写回pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys和源码状态。

案例 2：MergeJoin inner 不用 incremental sort

源码中明确 inner side 因 mark/restore 限制使用普通 Sort，不能只按成本想象 incremental sort。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：物理 tlist 回退

scan 可以尝试物理 tlist 优化，但遇到 dropped column、system column、placeholder 或 label 限制会回退到 build_path_tlist。

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

信号 1：Hash Cond 与 Join Filter 分离

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys字段，而不是停在 EXPLAIN 节点名。

信号 2：Merge Cond 与 Sort Key 对齐

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：Group Key 来自 child tlist

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：gating Result 包装 pseudoconstant qual

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys字段，而不是停在 EXPLAIN 节点名。

信号 5：物理 tlist 优化回退

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：outer join 同时有 join qual 和 filter qual

最小复现只保留能触发 Plan 节点构造的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：merge join 需要排序的两侧输入

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：GROUP BY 表达式聚合

无关 join、函数和投影会干扰create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：WHERE 中包含 pseudoconstant 条件

最小复现只保留能触发 Plan 节点构造的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `create_scan_plan`

在这个位置打印 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `build_path_tlist`

不要无差别打印全部结构；只打印能解释 Plan 节点构造的字段。

3. `create_join_plan`

记录前后差异，比记录单次静态值更有诊断价值。

4. `create_hashjoin_plan`

在这个位置打印 pathtype、PathTarget、scan clauses、join clauses、hash/merge/group keys，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `create_mergejoin_plan`

不要无差别打印全部结构；只打印能解释 Plan 节点构造的字段。

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

- 以为 Plan 节点只是 Path 节点改名。

纠正方法是把最终节点放回create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan()的时间轴，确认对应状态何时改变。

- 把 Join Filter、Hash Cond、Merge Cond 的来源混为一谈。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 Sort 节点可以直接使用 pathkeys 而不用 key 数组。

如果无法定位阶段，就不能直接写成确定结论。

- 忽略 parameterized path 到 Param 的重编码。

纠正方法是把最终节点放回create_plan_recurse()/create_scan_plan()/create_join_plan()/create_agg_plan()的时间轴，确认对应状态何时改变。

- 把物理 tlist 优化理解成语义必须。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 createplan 会重新估算节点成本。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：qual 分类实验

写 outer join，比较 EXPLAIN 中 Hash Cond、Join Filter、Filter，再断点观察 `extract_actual_join_clauses()`。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：small tlist 实验

构造 hash join 或 merge join 加宽 SELECT list，观察 inner child tlist 和成本变化。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：group key 实验

对 GROUP BY 查询断点 `create_agg_plan()`，检查 `extract_grouping_cols()` 如何从 child tlist 找列。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 Scan、Join、Agg 不能共用一个通用 Path 到 Plan 转换函数？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. Hash Cond、Merge Cond、Join Filter 分别从哪些 Path/RestrictInfo 字段来？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. 为什么 parameterized path 必须转成 executor Param？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. Sort key 和 Group key 为什么要从 targetlist 中抽取索引？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. 物理 tlist 什么时候可用，什么时候必须回退？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 哪些 cost 在 createplan 中只是复制，哪些节点会重新标注 display cost？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- createplan 节点构造的核心是把抽象 Path 字段重编码成 executor Plan contract。

- scan 处理 tlist 和 restriction，join 拆分算法专属 qual，agg/sort/limit 写入 key、strategy 和表达式。

- 正确性依赖 qual 分类、参数替换、targetlist/key 一致性和 parallel flags 传递。

- 异常路径来自未知 pathtype、key 匹配失败、物理 tlist 不可用和参数未分配。

- 可迁移规律是：优化器候选越抽象，落到执行器前越需要一次显式、可审计的契约化转换。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
