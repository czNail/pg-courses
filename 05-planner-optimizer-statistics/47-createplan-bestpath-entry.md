# PostgreSQL Best Path 选择与 create_plan 入口

## 课程定位

前置知识：理解 final_rel、cheapest path、Path/Plan 区别、targetlist、NestLoopParam 和 initPlan。

本节唯一主问题：

```text
当每个 relation 已经有 cheapest path 后，create_plan() 如何从 final_rel 的 best path 递归生成可执行 Plan tree，哪些 Path 信息会被保留，哪些只是 planner 阶段的临时决策？
```

核心矛盾：Path 保存的是搜索过程中的候选和比较信息；Plan 是 executor contract。转换过早会丢失搜索空间，转换过晚又必须把 targetlist、qual、param、initPlan 和 cost 等字段整理成执行器能直接消费的结构。

学完后应能从 `standard_planner()` 的 final_rel best path 追到 `create_plan()`，判断一个 Path 字段是否 会进入 Plan，还是只服务 planner 比较。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节完成 upper planning 中 partial path 到完整 path 的收束。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节是 Path 世界到 Plan 世界的入口课。

它不展开每种节点构造细节，而是回答 create_plan 为什么要作为单一递归入口存在。

理解入口后，下一节再读 `create_scan_plan()`、`create_join_plan()`、`create_agg_plan()` 才不会迷路。

下一节 会进入 createplan 内部，逐类讲 Scan、Join、Agg 等 Plan 节点如何从 Path 字段构造。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`standard_planner()` 在 `UPPERREL_FINAL` 上运行 `set_cheapest()`，取出 best path 交给 `create_plan()`；`create_plan()` 初始化 createplan workspace，递归调用 `create_plan_recurse()`，生成 Plan tree 后贴上 top targetlist label 和 initPlans，并校验 NestLoopParam 全部被分配。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
planner 搜索状态
  vs
executor 需要稳定、紧凑、可执行的 Plan contract
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/plan/planner.c | `standard_planner()` 调用 `grouping_planner()`、`set_cheapest()` 和 `create_plan()`。 |
| 2 | src/backend/optimizer/plan/createplan.c | `create_plan()`、`create_plan_recurse()`、`copy_generic_path_info()` 主入口。 |
| 3 | src/include/optimizer/planmain.h | `create_plan()` 对外声明和部分 Plan 构造器声明。 |
| 4 | src/include/nodes/pathnodes.h | `Path`、`PlannerInfo`、`RelOptInfo` 字段来源。 |
| 5 | src/include/nodes/plannodes.h | `Plan`、`PlannedStmt` 和各类 Plan 节点字段。 |
| 6 | src/backend/optimizer/plan/setrefs.c | `set_plan_references()` 后续完成 varno、param、rangetable 引用修正。 |
| 7 | src/backend/optimizer/plan/subselect.c | SubPlan / InitPlan 生成和 `plan_params` 交互。 |
| 8 | src/backend/optimizer/util/pathnode.c | `set_cheapest()` 与 best path 来源。 |
| 9 | src/backend/executor/execMain.c | ExecutorStart 后如何消费 Plan tree。 |
| 10 | src/include/executor/execdesc.h | 执行器 QueryDesc 持有 PlannedStmt 的边界。 |

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
| `final_rel->cheapest_total_path` | 通常作为顶层 best path 输入 `create_plan()`。 |
| `PlannerInfo.plan_params` | 当前 query level 待分配的 outer params，create_plan 入口要求为空。 |
| `PlannerInfo.curOuterRels` | 递归生成 nested loop 内侧时记录当前外层 relids。 |
| `PlannerInfo.curOuterParams` | 递归过程中收集还需分配给 NestLoop 的参数。 |
| `root->processed_tlist` | 顶层输出列名和 decoration 的来源。 |
| `Plan.targetlist` | executor 实际看到的输出和中间列。 |
| `Plan.startup_cost/total_cost/plan_rows` | 从 Path 复制给 EXPLAIN 和执行器参考的估算信息。 |
| `initPlan` | 当前 query level 生成并挂到顶层 Plan 的初始化子计划。 |
| `NestLoopParam` | 参数化 path 转成 executor nested loop param 的契约。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`final_rel->cheapest_total_path`**：通常作为顶层 best path 输入 `create_plan()`。

在 create_plan 入口中，这个字段决定 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 standard_planner()/create_plan()/create_plan_recurse() 处观察它的值。

**`PlannerInfo.plan_params`**：当前 query level 待分配的 outer params，create_plan 入口要求为空。

这个字段要和 Path 到 Plan 的单向转换和参数闭合一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 create_plan 入口、SS_attach_initplans 和 set_plan_references 中已经发生过的候选保留或淘汰。

**`PlannerInfo.curOuterRels`**：递归生成 nested loop 内侧时记录当前外层 relids。

它的诊断价值在于定位 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`PlannerInfo.curOuterParams`**：递归过程中收集还需分配给 NestLoop 的参数。

在 create_plan 入口中，这个字段决定 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 standard_planner()/create_plan()/create_plan_recurse() 处观察它的值。

**`root->processed_tlist`**：顶层输出列名和 decoration 的来源。

这个字段要和 Path 到 Plan 的单向转换和参数闭合一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 create_plan 入口、SS_attach_initplans 和 set_plan_references 中已经发生过的候选保留或淘汰。

**`Plan.targetlist`**：executor 实际看到的输出和中间列。

它的诊断价值在于定位 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 以为 create_plan 还会重新搜索 plan。

- 把 Path 的所有字段都理解成会出现在 Plan 中。

- 忽略 `set_plan_references()` 对最终 executor contract 的作用。

- 认为 initPlan 可以随便挂在引用点附近。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
`standard_planner()` 完成 `grouping_planner()` 后得到 `UPPERREL_FINAL`。
它对 final rel 调用 `set_cheapest()`，确定最终 best path。
在调用 `create_plan()` 前，planner 仍然处于 Path 世界；pathlist 中可能还有其它候选。
`create_plan()` 断言当前 query level 的 `plan_params` 未被占用。
它初始化 `root->curOuterRels` 和 `root->curOuterParams`，准备处理参数化路径。
`create_plan_recurse()` 根据 `best_path->pathtype` 分发到 scan、join、agg、sort、limit、gather 等构造函数。
每个构造函数先递归生成 child plan，再把当前 Path 的语义字段转成 Plan 字段。
通用成本和 rows 通过 `copy_generic_path_info()` 写入 Plan。
顶层非 ModifyTable plan 会通过 `apply_tlist_labeling()` 恢复输出列 decoration。
`SS_attach_initplans()` 把当前 query level 的 initPlans 挂到 top plan。
若 `curOuterParams` 仍有未分配项，说明参数化路径没有正确转成 NestLoopParam，create_plan 报错。
最后 Plan tree 还会经过 `set_plan_references()`，再进入 PlannedStmt 和 executor。
```

步骤 1：`standard_planner()` 完成 `grouping_planner()` 后得到 `UPPERREL_FINAL`。

这里要记录的是 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 如何改变，而不是只看函数是否返回。

断点优先打印 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan，它们能区分候选、剪枝和转换。

步骤 2：它对 final rel 调用 `set_cheapest()`，确定最终 best path。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 standard_planner()/create_plan()/create_plan_recurse() 之前未生成，还是之后被剪枝。

步骤 3：在调用 `create_plan()` 前，planner 仍然处于 Path 世界；pathlist 中可能还有其它候选。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：`create_plan()` 断言当前 query level 的 `plan_params` 未被占用。

这里要记录的是 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 create_plan 入口有关的关键字段。

步骤 5：它初始化 `root->curOuterRels` 和 `root->curOuterParams`，准备处理参数化路径。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：`create_plan_recurse()` 根据 `best_path->pathtype` 分发到 scan、join、agg、sort、limit、gather 等构造函数。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：每个构造函数先递归生成 child plan，再把当前 Path 的语义字段转成 Plan 字段。

这里要记录的是 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 Path 到 Plan 的单向转换和参数闭合的那几项。

步骤 8：通用成本和 rows 通过 `copy_generic_path_info()` 写入 Plan。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：顶层非 ModifyTable plan 会通过 `apply_tlist_labeling()` 恢复输出列 decoration。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan时才能完整解释。

步骤 10：`SS_attach_initplans()` 把当前 query level 的 initPlans 挂到 top plan。

这里要记录的是 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan 如何改变，而不是只看函数是否返回。

断点优先打印 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan，它们能区分候选、剪枝和转换。

步骤 11：若 `curOuterParams` 仍有未分配项，说明参数化路径没有正确转成 NestLoopParam，create_plan 报错。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 standard_planner()/create_plan()/create_plan_recurse() 之前未生成，还是之后被剪枝。

步骤 12：最后 Plan tree 还会经过 `set_plan_references()`，再进入 PlannedStmt 和 executor。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| best path 选择 | final_rel 的 pathlist 完成后由 `set_cheapest()` 抽取，best path 仍属于 planner context。 |
| Plan 创建 | Plan 节点用 `makeNode()` 在 planner context 中创建，结构比 Path 更接近 executor contract。 |
| 字段复制 | 部分 Path 字段复制到 Plan，许多比较用字段如 pathkeys、param_info 只在转换中消费。 |
| 后处理 | initPlan attach、targetlist labeling、setrefs 和 SS_finalize_plan 在 create_plan 之后继续整理。 |
| 执行期 | executor 不再访问 `RelOptInfo` pathlist，它消费 PlannedStmt 中的 Plan tree。 |

best path 选择阶段的重点是：final_rel 的 pathlist 完成后由 `set_cheapest()` 抽取，best path 仍属于 planner context。

这份状态由 planner context 持有；只有 top Plan、initPlans 和 PlannedStmt 会进入后续 executor contract。

Plan 创建阶段的重点是：Plan 节点用 `makeNode()` 在 planner context 中创建，结构比 Path 更接近 executor contract。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

字段复制阶段的重点是：部分 Path 字段复制到 Plan，许多比较用字段如 pathkeys、param_info 只在转换中消费。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

后处理阶段的重点是：initPlan attach、targetlist labeling、setrefs 和 SS_finalize_plan 在 create_plan 之后继续整理。

这份状态由 planner context 持有；只有 top Plan、initPlans 和 PlannedStmt 会进入后续 executor contract。

执行期阶段的重点是：executor 不再访问 `RelOptInfo` pathlist，它消费 PlannedStmt 中的 Plan tree。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| 唯一入口 | create_plan 保证从 best path 开始递归，不会混入未胜出的 sibling path。 |
| targetlist | 顶层输出 decoration 必须来自 `processed_tlist`，中间 tlist 则服务 executor 输入。 |
| param 分配 | parameterized path 必须转成 NestLoopParam，否则 executor 不知道从哪里取外层值。 |
| initPlan | 当前 query level 的 initPlans 挂到 top node，保证引用位置可见。 |
| stack depth | `create_plan_recurse()` 检查栈深，复杂查询不能无限递归。 |
| Plan 引用 | Vars 仍是 planner 编号，后续 `setrefs.c` 才完成最终引用修正。 |

`唯一入口` 这一层保证的是：create_plan 保证从 best path 开始递归，不会混入未胜出的 sibling path。

它只能约束 Path 到 Plan 的单向转换和参数闭合，不能代替成本比较、能力检查或 Plan 字段契约。

`targetlist` 这一层保证的是：顶层输出 decoration 必须来自 `processed_tlist`，中间 tlist 则服务 executor 输入。

如果这个层面通过，仍要继续检查 plan_params 断言、NestLoopParam 检查和 initPlan 挂载，以及后续 createplan contract。

`param 分配` 这一层保证的是：parameterized path 必须转成 NestLoopParam，否则 executor 不知道从哪里取外层值。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`initPlan` 这一层保证的是：当前 query level 的 initPlans 挂到 top node，保证引用位置可见。

它只能约束 Path 到 Plan 的单向转换和参数闭合，不能代替成本比较、能力检查或 Plan 字段契约。

`stack depth` 这一层保证的是：`create_plan_recurse()` 检查栈深，复杂查询不能无限递归。

如果这个层面通过，仍要继续检查 plan_params 断言、NestLoopParam 检查和 initPlan 挂载，以及后续 createplan contract。

`Plan 引用` 这一层保证的是：Vars 仍是 planner 编号，后续 `setrefs.c` 才完成最终引用修正。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 未知 pathtype | `create_plan_recurse()` 遇到未识别 node type 会 ERROR。 |
| NestLoopParam 未分配 | `curOuterParams` 非空会报错，说明参数化转换失败。 |
| stack overflow | 过深 plan tree 通过 `check_stack_depth()` 防止 backend 崩溃。 |
| ModifyTable 特例 | 顶层 ModifyTable targetlist 不按普通 SELECT output labeling 处理。 |
| subquery best path | SubPlan 内部也会调用 `create_plan()`，但使用子 query level 的 PlannerInfo。 |

未知 pathtype：`create_plan_recurse()` 遇到未识别 node type 会 ERROR。

诊断这类路径时，优先用 plan_params 断言、NestLoopParam 检查和 initPlan 挂载确认它在生成前还是剪枝后消失。

NestLoopParam 未分配：`curOuterParams` 非空会报错，说明参数化转换失败。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

stack overflow：过深 plan tree 通过 `check_stack_depth()` 防止 backend 崩溃。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

ModifyTable 特例：顶层 ModifyTable targetlist 不按普通 SELECT output labeling 处理。

诊断这类路径时，优先用 plan_params 断言、NestLoopParam 检查和 initPlan 挂载确认它在生成前还是剪枝后消失。

subquery best path：SubPlan 内部也会调用 `create_plan()`，但使用子 query level 的 PlannerInfo。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| 递归成本 | create_plan 本身通常不是主要瓶颈，但极复杂 plan tree 会增加递归和 list 处理成本。 |
| tlist 大小 | build_path_tlist 和 projection 决策会影响 Plan 宽度与 executor 投影成本。 |
| param 替换 | parameterized plan 需要 replace_nestloop_params，复杂表达式会增加 planner CPU。 |
| initPlan | initPlan 的 cost 已在选择 best path 前计入，create_plan 负责挂接而非重新比较。 |
| EXPLAIN 信息 | Plan 成本来自 Path 复制，后续执行不会重新计算 planner cost。 |
| 内存 | Plan tree 与剩余 Path 都在 planner context 中，最终 PlannedStmt 保留必要 Plan 结构。 |

递归成本：create_plan 本身通常不是主要瓶颈，但极复杂 plan tree 会增加递归和 list 处理成本。

这个成本会先影响 copy_generic_path_info、targetlist 宽度和递归节点数量，再通过幸存 Path 传到下一阶段或 Plan。

tlist 大小：build_path_tlist 和 projection 决策会影响 Plan 宽度与 executor 投影成本。

它不是执行毫秒数，而是候选之间排序的相对信号。

param 替换：parameterized plan 需要 replace_nestloop_params，复杂表达式会增加 planner CPU。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

initPlan：initPlan 的 cost 已在选择 best path 前计入，create_plan 负责挂接而非重新比较。

这个成本会先影响 copy_generic_path_info、targetlist 宽度和递归节点数量，再通过幸存 Path 传到下一阶段或 Plan。

EXPLAIN 信息：Plan 成本来自 Path 复制，后续执行不会重新计算 planner cost。

它不是执行毫秒数，而是候选之间排序的相对信号。

内存：Plan tree 与剩余 Path 都在 planner context 中，最终 PlannedStmt 保留必要 Plan 结构。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：最终 EXPLAIN 展示的是 Plan tree，不是 pathlist。

看到这个现象后，先回到create_plan 入口、SS_attach_initplans 和 set_plan_references，不要只根据节点名归因。

观测入口 2：Plan 节点上的 cost/rows/width 多数来自 `copy_generic_path_info()`。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：参数化 nested loop 可通过查看 `NestLoopParam` 和 inner plan Param 引用确认。

节点名只是最终形态，真正原因通常在final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan的变化中。

观测入口 4：设置断点在 `create_plan()` 可以确认 final best path 的 pathtype。

看到这个现象后，先回到create_plan 入口、SS_attach_initplans 和 set_plan_references，不要只根据节点名归因。

观测入口 5：设置断点在 `set_plan_references()` 可以看到 create_plan 后 Vars 仍需修正。

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

### create_plan 入口的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`standard_planner()`

在 grouping_planner 之后对 final_rel 运行 `set_cheapest()`，准备 best path。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan，再对照最终 EXPLAIN。

检查点 2：`create_plan()`

Path 到 Plan 的单一入口，不重新搜索候选。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`Assert(root->plan_params == NIL)`

确保当前 query level 没有未处理参数状态进入入口。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`root->curOuterRels`

递归生成 nested loop 内侧时记录可见外层 relids。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 create_plan 入口的字段，再补 rows 和 cost。

检查点 5：`root->curOuterParams`

记录还没有绑定到 NestLoopParam 的外层参数。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查plan_params 断言、NestLoopParam 检查和 initPlan 挂载；若生成后消失，再看剪枝规则。

检查点 6：`create_plan_recurse()`

根据 pathtype 分发到具体 Plan 构造函数。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`CP_EXACT_TLIST`

顶层要求精确 targetlist，避免输出契约与 SQL 结果不一致。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`apply_tlist_labeling()`

把顶层输出列名和 decoration 从 processed_tlist 贴回 Plan。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`SS_attach_initplans()`

把当前 query level 生成的 initPlans 挂到 top Plan。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`curOuterParams != NIL` 检查

参数化路径没有被正确转成 NestLoopParam 时立即报错。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan，再对照最终 EXPLAIN。

检查点 11：`copy_generic_path_info()`

把 cost、rows、width 和 parallel flags 从 Path 复制到 Plan。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：`set_plan_references()`

create_plan 后继续修正 Var、Param、RTE 引用，是下一层 contract 收尾。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：最终 best path 不等于唯一幸存 path

final_rel 可能有多个 path；create_plan 只消费 `set_cheapest()` 选出的 best path。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 standard_planner()/create_plan()/create_plan_recurse() 验证。

节点名只是终点；本课要把原因写回final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan和源码状态。

案例 2：initPlan 挂到顶层

不相关子查询可能在规划中生成 initPlan，create_plan 末尾统一 attach，避免分散在任意引用点。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：NestedLoopParam 漏分配

如果参数化依赖没有在 nested loop 构造中绑定，入口收尾检查会报错。

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

信号 1：最终 EXPLAIN 只显示一个 best path

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan字段，而不是停在 EXPLAIN 节点名。

信号 2：initPlan 挂在 top plan

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：NestedLoopParam 出现在 inner plan

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：create_plan 后还要 setrefs

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan字段，而不是停在 EXPLAIN 节点名。

信号 5：ModifyTable targetlist 是特例

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：不相关子查询生成 initPlan

最小复现只保留能触发 create_plan 入口的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：LATERAL 触发 parameterized nested loop

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：SELECT 与 UPDATE RETURNING 对照

无关 join、函数和投影会干扰standard_planner()/create_plan()/create_plan_recurse()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：复杂 UNION 外层 LIMIT

最小复现只保留能触发 create_plan 入口的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `standard_planner`

在这个位置打印 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `set_cheapest`

不要无差别打印全部结构；只打印能解释 create_plan 入口的字段。

3. `create_plan`

记录前后差异，比记录单次静态值更有诊断价值。

4. `create_plan_recurse`

在这个位置打印 final_rel best path、plan_params、curOuterRels、curOuterParams 和 initPlan，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `SS_attach_initplans`

不要无差别打印全部结构；只打印能解释 create_plan 入口的字段。

6. `set_plan_references`

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

- 以为 create_plan 还会重新搜索 plan。

纠正方法是把最终节点放回standard_planner()/create_plan()/create_plan_recurse()的时间轴，确认对应状态何时改变。

- 把 Path 的所有字段都理解成会出现在 Plan 中。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 忽略 `set_plan_references()` 对最终 executor contract 的作用。

如果无法定位阶段，就不能直接写成确定结论。

- 认为 initPlan 可以随便挂在引用点附近。

纠正方法是把最终节点放回standard_planner()/create_plan()/create_plan_recurse()的时间轴，确认对应状态何时改变。

- 把 parameterized path 的外层依赖当成普通 qual。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 以为 EXPLAIN 展示的是 optimizer 全部候选。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：入口断点实验

在 `standard_planner()` 调用 `create_plan()` 前后打印 final_rel cheapest path 的 pathtype 和 rows。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：参数化实验

构造 LATERAL nested loop，断点观察 `curOuterRels`、`curOuterParams` 和 `NestLoopParam`。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：initPlan 实验

使用不相关子查询，观察 `SS_attach_initplans()` 后 top Plan 的 initPlan 列表。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. create_plan 为什么从 final best path 开始，而不是从 Query tree 重新生成？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. 哪些 Path 字段会复制到 Plan，哪些只在转换时消费？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. parameterized path 如何变成 executor 可用的 NestLoopParam？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. 为什么顶层 targetlist labeling 在 create_plan 末尾处理？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. `set_plan_references()` 解决了 create_plan 之后的哪些 contract 问题？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 如何从 EXPLAIN 反推 create_plan 入口选择的 best path？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- create_plan 是 Path 世界到 Plan 世界的单一递归入口。

- 它不重新搜索，只消费 final_rel 的 best path。

- 关键状态包括 `curOuterRels`、`curOuterParams`、`processed_tlist`、initPlans 和 NestLoopParam。

- 错误路径集中在未知 pathtype、栈深、参数未分配和 targetlist 特例。

- 可迁移规律是：候选搜索结构和执行契约结构应分离，转换点必须显式处理丢弃、复制和重编码。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
