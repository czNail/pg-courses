# PostgreSQL Parallel Upper Planning 与 Gather

## 课程定位

前置知识：理解 partial path、parallel safety、Gather/Gather Merge、partial/final aggregate 和 upperrel pipeline。

本节唯一主问题：

```text
partial path 如何经过 partial aggregate、parallel append、gather、gather merge 和 finalize aggregate 变成完整结果，哪些 upper 操作会阻断并行计划？
```

核心矛盾：并行 partial path 可以降低单个 worker 的扫描和聚合压力，但 SQL 顶层结果通常需要全局顺序、全局去重、final aggregate 或单一输出流。planner 必须在并行收益和汇总成本之间切换。

学完后应能判断一个查询为什么只在 scan/join 部分并行、为什么 upper 阶段停止并行，或者为什么出现 Gather Merge 而不是 Gather。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释 ORDER BY、incremental sort 和 LIMIT 如何影响 upper path。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节连接 04 目录的执行器并行课程和 05 目录的 planner path 课程。

这里不讲 worker 启动或 DSM 状态，只讲 optimizer 如何把 partial path 纳入 upper planning。

并行计划不是一个开关，而是一组 path 在每个语义阶段能否继续传播的问题。

下一节进入 createplan，观察 final_rel 的 best path 如何变成 Plan tree。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
lower relation 的 `partial_pathlist` 只有经过 Gather/Gather Merge 或 partial/final upper path 才能成为完整 path；`grouping_planner()` 在 aggregate、distinct、ordered 和 final 阶段持续检查 `consider_parallel`、target parallel safety、partial pathlist 和 ordering 需求。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
worker 局部结果
  vs
SQL 顶层需要全局语义和单一输出契约
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/path/allpaths.c | `generate_gather_paths()`、`generate_useful_gather_paths()`、partial path 转完整 path。 |
| 2 | src/backend/optimizer/plan/planner.c | partial grouping、partial distinct、ordered gather merge 和 final upperrel。 |
| 3 | src/backend/optimizer/util/pathnode.c | `create_gather_path()`、`create_gather_merge_path()`、`create_agg_path()`。 |
| 4 | src/backend/optimizer/path/costsize.c | parallel setup、tuple communication、Gather/Gather Merge 成本。 |
| 5 | src/include/nodes/pathnodes.h | `GatherPath`、`GatherMergePath`、partial pathlist、parallel flags。 |
| 6 | src/backend/optimizer/plan/createplan.c | `create_gather_plan()`、`create_gather_merge_plan()`、partial aggregate Plan 转换。 |
| 7 | src/backend/executor/execParallel.c | 执行期并行查询入口，对照 planner 写入的计划。 |
| 8 | src/backend/executor/nodeGather.c | Gather 执行期消费 worker tuple。 |
| 9 | src/backend/executor/nodeGatherMerge.c | Gather Merge 执行期保序合并。 |
| 10 | src/include/optimizer/paths.h | Gather path 生成入口声明。 |

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
| `RelOptInfo.partial_pathlist` | worker 局部候选 path，只表示 partial result。 |
| `RelOptInfo.consider_parallel` | 当前 relation 是否仍允许考虑并行 path。 |
| `Path.parallel_safe` | 表达式和节点是否能放进 worker。 |
| `Path.parallel_aware` | 节点是否知道自己在并行环境下分工执行。 |
| `GatherPath.subpath` | 被汇总的 partial path。 |
| `GatherMergePath.pathkeys` | worker 局部有序输出被全局保序合并后的 pathkeys。 |
| `partially_grouped_rel` | partial aggregate 之后、finalize 之前的中间 relation。 |
| `AggPath.aggsplit` | 区分 partial、finalize 和 simple aggregate。 |
| `root->glob->parallelModeNeeded` | 最终 PlannedStmt 是否需要并行模式的全局信号。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`RelOptInfo.partial_pathlist`**：worker 局部候选 path，只表示 partial result。

在 parallel upper planning中，这个字段决定 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 generate_gather_paths()/generate_useful_gather_paths() 处观察它的值。

**`RelOptInfo.consider_parallel`**：当前 relation 是否仍允许考虑并行 path。

这个字段要和 partial result 到全局结果的收束边界一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 partial_pathlist、partially_grouped_rel 和 create_gather_plan 中已经发生过的候选保留或淘汰。

**`Path.parallel_safe`**：表达式和节点是否能放进 worker。

它的诊断价值在于定位 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`Path.parallel_aware`**：节点是否知道自己在并行环境下分工执行。

在 parallel upper planning中，这个字段决定 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 generate_gather_paths()/generate_useful_gather_paths() 处观察它的值。

**`GatherPath.subpath`**：被汇总的 partial path。

这个字段要和 partial result 到全局结果的收束边界一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 partial_pathlist、partially_grouped_rel 和 create_gather_plan 中已经发生过的候选保留或淘汰。

**`GatherMergePath.pathkeys`**：worker 局部有序输出被全局保序合并后的 pathkeys。

它的诊断价值在于定位 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 以为打开并行 GUC 后所有 upper 节点都会并行。

- 把 partial path 当成完整结果 path。

- 认为 Gather Merge 只是 Gather 的更快版本。

- 忽略 final aggregate/distinct 的全局语义。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
base relation 或 join relation 先生成普通 pathlist 和 partial_pathlist。
`generate_gather_paths()` 通常把 cheapest partial path 包成 GatherPath，加入普通 pathlist。
`generate_useful_gather_paths()` 还会考虑有用 pathkeys，必要时生成 Gather Merge。
进入 grouping 阶段后，`create_partial_grouping_paths()` 尝试在 worker 内先做 partial aggregate。
`gather_grouping_paths()` 把 partial grouped result 收集回来。
随后 final aggregate path 在 leader 层完成全局聚合语义。
distinct 阶段可先做 partial distinct，再 Gather，再 final distinct。
ORDER BY 阶段若 partial path 能排序，可通过 Gather Merge 保持全局顺序。
如果 target、HAVING、LIMIT 表达式或 rowMarks 不 parallel safe，parallel 传播停止。
final_rel 若输入 still consider_parallel 且 LIMIT 表达式 safe，可继续标记 consider_parallel。
`create_plan()` 看到 Gather/GatherMerge Path 后生成对应 Plan，并标记并行相关字段。
executor 根据最终 Plan 才启动 worker；planner 阶段不启动任何 worker。
```

步骤 1：base relation 或 join relation 先生成普通 pathlist 和 partial_pathlist。

这里要记录的是 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 如何改变，而不是只看函数是否返回。

断点优先打印 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath，它们能区分候选、剪枝和转换。

步骤 2：`generate_gather_paths()` 通常把 cheapest partial path 包成 GatherPath，加入普通 pathlist。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 generate_gather_paths()/generate_useful_gather_paths() 之前未生成，还是之后被剪枝。

步骤 3：`generate_useful_gather_paths()` 还会考虑有用 pathkeys，必要时生成 Gather Merge。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：进入 grouping 阶段后，`create_partial_grouping_paths()` 尝试在 worker 内先做 partial aggregate。

这里要记录的是 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 parallel upper planning有关的关键字段。

步骤 5：`gather_grouping_paths()` 把 partial grouped result 收集回来。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：随后 final aggregate path 在 leader 层完成全局聚合语义。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：distinct 阶段可先做 partial distinct，再 Gather，再 final distinct。

这里要记录的是 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 partial result 到全局结果的收束边界的那几项。

步骤 8：ORDER BY 阶段若 partial path 能排序，可通过 Gather Merge 保持全局顺序。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：如果 target、HAVING、LIMIT 表达式或 rowMarks 不 parallel safe，parallel 传播停止。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath时才能完整解释。

步骤 10：final_rel 若输入 still consider_parallel 且 LIMIT 表达式 safe，可继续标记 consider_parallel。

这里要记录的是 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath 如何改变，而不是只看函数是否返回。

断点优先打印 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath，它们能区分候选、剪枝和转换。

步骤 11：`create_plan()` 看到 Gather/GatherMerge Path 后生成对应 Plan，并标记并行相关字段。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 generate_gather_paths()/generate_useful_gather_paths() 之前未生成，还是之后被剪枝。

步骤 12：executor 根据最终 Plan 才启动 worker；planner 阶段不启动任何 worker。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| partial path 创建 | scan/join 阶段在 planner context 中创建 partial path，等待 gather 或 upper partial 消费。 |
| gather path 创建 | `create_gather_path()` 和 `create_gather_merge_path()` 把 partial path 转成完整输出 path。 |
| upper continuation | partial aggregate/distinct 会先生成中间 relation，再生成 final relation path。 |
| Plan 转换 | `create_gather_plan()`、`create_gather_merge_plan()` 保留 worker 数、single_copy、rescan_param 等字段。 |
| 执行期 | worker DSM、tuple queue、Instrumentation 属于 executor，不属于 planner path 生命周期。 |

partial path 创建阶段的重点是：scan/join 阶段在 planner context 中创建 partial path，等待 gather 或 upper partial 消费。

这份状态由 planner context 持有；只有 Gather/GatherMerge Plan 与 parallelModeNeeded 会进入后续 executor contract。

gather path 创建阶段的重点是：`create_gather_path()` 和 `create_gather_merge_path()` 把 partial path 转成完整输出 path。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

upper continuation阶段的重点是：partial aggregate/distinct 会先生成中间 relation，再生成 final relation path。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

Plan 转换阶段的重点是：`create_gather_plan()`、`create_gather_merge_plan()` 保留 worker 数、single_copy、rescan_param 等字段。

这份状态由 planner context 持有；只有 Gather/GatherMerge Plan 与 parallelModeNeeded 会进入后续 executor contract。

执行期阶段的重点是：worker DSM、tuple queue、Instrumentation 属于 executor，不属于 planner path 生命周期。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| partial 不完整 | partial path 不能直接作为最终输出，必须有 gather 或 finalization。 |
| 聚合语义 | partial aggregate 只能用于支持 combine/finalize 的 aggregate。 |
| 去重语义 | worker 内去重不能保证全局去重，必须 final distinct。 |
| 排序语义 | Gather 不保序，ORDER BY 需要 Gather Merge 或 Gather 后再 Sort。 |
| parallel safety | unsafe function、volatile side effect、row lock 或 DML 语义会阻断并行。 |
| 参数化边界 | 某些 parameterized path 不能作为普通 parallel partial path 自由移动。 |

`partial 不完整` 这一层保证的是：partial path 不能直接作为最终输出，必须有 gather 或 finalization。

它只能约束 partial result 到全局结果的收束边界，不能代替成本比较、能力检查或 Plan 字段契约。

`聚合语义` 这一层保证的是：partial aggregate 只能用于支持 combine/finalize 的 aggregate。

如果这个层面通过，仍要继续检查 parallel safety、Gather 成本、Gather Merge pathkeys 和 final aggregate/distinct，以及后续 createplan contract。

`去重语义` 这一层保证的是：worker 内去重不能保证全局去重，必须 final distinct。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`排序语义` 这一层保证的是：Gather 不保序，ORDER BY 需要 Gather Merge 或 Gather 后再 Sort。

它只能约束 partial result 到全局结果的收束边界，不能代替成本比较、能力检查或 Plan 字段契约。

`parallel safety` 这一层保证的是：unsafe function、volatile side effect、row lock 或 DML 语义会阻断并行。

如果这个层面通过，仍要继续检查 parallel safety、Gather 成本、Gather Merge pathkeys 和 final aggregate/distinct，以及后续 createplan contract。

`参数化边界` 这一层保证的是：某些 parameterized path 不能作为普通 parallel partial path 自由移动。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 无 partial path | 下层没有 partial_pathlist 时，upper 阶段无法凭空制造并行。 |
| unsafe target | target 或 qual 不 parallel safe 时，upperrel 不再 consider_parallel。 |
| Gather Merge 不可用 | 没有 worker 局部 pathkeys 或排序成本过高时，planner 可能选择 Gather 后 Sort。 |
| 并行收益不足 | parallel setup 和 tuple communication 成本可能让普通 path 胜出。 |
| finalization 缺失 | 不支持 partial 的 aggregate 不会生成 partial/final pipeline。 |

无 partial path：下层没有 partial_pathlist 时，upper 阶段无法凭空制造并行。

诊断这类路径时，优先用 parallel safety、Gather 成本、Gather Merge pathkeys 和 final aggregate/distinct确认它在生成前还是剪枝后消失。

unsafe target：target 或 qual 不 parallel safe 时，upperrel 不再 consider_parallel。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

Gather Merge 不可用：没有 worker 局部 pathkeys 或排序成本过高时，planner 可能选择 Gather 后 Sort。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

并行收益不足：parallel setup 和 tuple communication 成本可能让普通 path 胜出。

诊断这类路径时，优先用 parallel safety、Gather 成本、Gather Merge pathkeys 和 final aggregate/distinct确认它在生成前还是剪枝后消失。

finalization 缺失：不支持 partial 的 aggregate 不会生成 partial/final pipeline。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| setup cost | 启动 worker 和并行基础设施有固定成本，小查询可能不划算。 |
| tuple communication | worker 输出 tuple 越多，Gather 通信成本越高。 |
| partial reduction | partial aggregate 若大幅减少 rows，可抵消 gather 成本。 |
| merge cost | Gather Merge 需要保持每路有序并做 leader merge。 |
| leader participation | leader 是否参与影响可用 CPU 和输出汇总延迟。 |
| upper blocking | Sort、final distinct、final aggregate 等 blocking 阶段会改变并行收益的位置。 |

setup cost：启动 worker 和并行基础设施有固定成本，小查询可能不划算。

这个成本会先影响 parallel setup、tuple communication、worker 数和 finalization，再通过幸存 Path 传到下一阶段或 Plan。

tuple communication：worker 输出 tuple 越多，Gather 通信成本越高。

它不是执行毫秒数，而是候选之间排序的相对信号。

partial reduction：partial aggregate 若大幅减少 rows，可抵消 gather 成本。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

merge cost：Gather Merge 需要保持每路有序并做 leader merge。

这个成本会先影响 parallel setup、tuple communication、worker 数和 finalization，再通过幸存 Path 传到下一阶段或 Plan。

leader participation：leader 是否参与影响可用 CPU 和输出汇总延迟。

它不是执行毫秒数，而是候选之间排序的相对信号。

upper blocking：Sort、final distinct、final aggregate 等 blocking 阶段会改变并行收益的位置。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：EXPLAIN 中 `Gather` 下方节点通常表示 worker 执行的 partial plan。

看到这个现象后，先回到partial_pathlist、partially_grouped_rel 和 create_gather_plan，不要只根据节点名归因。

观测入口 2：`Gather Merge` 出现说明 planner 需要保留全局顺序。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：`Partial Aggregate` / `Finalize Aggregate` 是 upper parallel planning 的典型证据。

节点名只是最终形态，真正原因通常在partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath的变化中。

观测入口 4：设置 `max_parallel_workers_per_gather`、`parallel_setup_cost`、`parallel_tuple_cost` 可以观察边界。

看到这个现象后，先回到partial_pathlist、partially_grouped_rel 和 create_gather_plan，不要只根据节点名归因。

观测入口 5：断点检查 `input_rel->partial_pathlist`，能区分“没有并行候选”和“候选输给普通 path”。

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

### parallel upper planning 的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`partial_pathlist`

下层 relation 是否有 worker 局部候选，是 upper parallel 的起点。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath，再对照最终 EXPLAIN。

检查点 2：`generate_gather_paths()`

把 cheapest partial path 包装成普通完整 path。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`generate_useful_gather_paths()`

额外考虑有用 ordering，可能生成 Gather Merge。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`create_partial_grouping_paths()`

worker 内先聚合，减少 Gather 输出 rows。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 parallel upper planning的字段，再补 rows 和 cost。

检查点 5：`gather_grouping_paths()`

把 partial grouped paths 收束为可 finalize 的路径。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查parallel safety、Gather 成本、Gather Merge pathkeys 和 final aggregate/distinct；若生成后消失，再看剪枝规则。

检查点 6：`create_partial_distinct_paths()`

局部 distinct 后仍要 final distinct，不能把局部结果当全局结果。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`consider_parallel`

输入 relation 和 upper target 都安全时才继续传播并行候选。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`parallel_safe`

表达式级安全条件，和节点是否 parallel-aware 不是一回事。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`create_gather_path()`

不保证输出顺序，只保证 worker tuple 被汇总到单一输出流。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`create_gather_merge_path()`

要求子路径有序，并在 leader 层维护全局 pathkeys。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath，再对照最终 EXPLAIN。

检查点 11：`compute_gather_rows()`

估算 Gather/Gather Merge 汇总后的 rows。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：`glob->parallelModeNeeded`

最终 PlannedStmt 是否要进入 parallel mode 的全局信号。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：scan 并行但 upper 不并行

下层出现 Parallel Seq Scan，不代表 GROUP BY 或 DISTINCT 也能并行；upper target 或 aggregate 能力可能阻断传播。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 generate_gather_paths()/generate_useful_gather_paths() 验证。

节点名只是终点；本课要把原因写回partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath和源码状态。

案例 2：Partial Aggregate 反而更慢

worker 局部聚合若不能显著减少 rows，Gather 和 finalize 成本可能让普通 path 胜出。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：Gather Merge 的必要性

ORDER BY 查询中，只有 Gather Merge 能在汇总时保序；普通 Gather 不能携带 pathkeys。

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

信号 1：Parallel Seq Scan 下方有 Partial Aggregate

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath字段，而不是停在 EXPLAIN 节点名。

信号 2：Gather Merge 保留排序

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：普通 Gather 丢失 pathkeys

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：parallel unsafe 函数阻断并行

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath字段，而不是停在 EXPLAIN 节点名。

信号 5：partial distinct 后仍有 final distinct

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：大表 GROUP BY 开启并行

最小复现只保留能触发 parallel upper planning的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：ORDER BY 并行查询

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：SELECT 中加入 parallel unsafe 函数

无关 join、函数和投影会干扰generate_gather_paths()/generate_useful_gather_paths()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：DISTINCT 大表并行去重

最小复现只保留能触发 parallel upper planning的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `generate_gather_paths`

在这个位置打印 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `generate_useful_gather_paths`

不要无差别打印全部结构；只打印能解释 parallel upper planning的字段。

3. `create_partial_grouping_paths`

记录前后差异，比记录单次静态值更有诊断价值。

4. `gather_grouping_paths`

在这个位置打印 partial_pathlist、consider_parallel、parallel_safe、GatherPath 和 GatherMergePath，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `create_partial_distinct_paths`

不要无差别打印全部结构；只打印能解释 parallel upper planning的字段。

6. `create_gather_plan`

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

- 以为打开并行 GUC 后所有 upper 节点都会并行。

纠正方法是把最终节点放回generate_gather_paths()/generate_useful_gather_paths()的时间轴，确认对应状态何时改变。

- 把 partial path 当成完整结果 path。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为 Gather Merge 只是 Gather 的更快版本。

如果无法定位阶段，就不能直接写成确定结论。

- 忽略 final aggregate/distinct 的全局语义。

纠正方法是把最终节点放回generate_gather_paths()/generate_useful_gather_paths()的时间轴，确认对应状态何时改变。

- 把 parallel_safe 和 parallel_aware 混为一谈。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 看到没有并行计划就只怀疑 worker 参数，而不看 target 和 qual safety。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：partial aggregate 实验

对大表做 GROUP BY，调高并行参数，观察 Partial/Finalize Aggregate 与 Gather。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：保序并行实验

在并行查询上加入 ORDER BY，对比 Gather 后 Sort 和 Gather Merge。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：阻断实验

在 SELECT list 中加入 parallel-unsafe 函数或 FOR UPDATE，观察并行 path 如何消失。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 partial path 必须经过 Gather 或 finalization 才能成为完整结果？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. Gather 和 Gather Merge 在 ordering 语义上有什么根本区别？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. partial aggregate 的收益来自哪里，成本又在哪里增加？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. 哪些 upper 操作会阻断并行传播？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. `parallel_safe` 与 `parallel_aware` 分别回答什么问题？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 如何区分没有生成并行候选和并行候选最终输掉？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- parallel upper planning 的核心是把 worker 局部结果转成全局 SQL 结果。

- `partial_pathlist`、`consider_parallel`、`GatherPath`、`GatherMergePath` 和 `aggsplit` 是关键状态。

- 聚合、去重和排序都可能先局部处理，再通过 final 阶段恢复全局语义。

- 并行收益受 setup、通信、merge、finalization 和 blocking 节点影响。

- 可迁移规律是：并行不是把任意节点复制给 worker，而是把可分解语义拆成局部阶段和全局收束阶段。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
