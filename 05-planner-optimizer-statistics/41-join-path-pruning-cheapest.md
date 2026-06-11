# PostgreSQL Join Path 剪枝、pathkeys 与 cheapest path

## 课程定位

前置知识：理解 joinrel、三类 join path、参数化路径、pathkeys 和 cost model 的基本字段。

本节唯一主问题：

```text
add_path() 为什么不能只保留 total cost 最低的 join path，而要同时考虑 pathkeys、startup cost、parameterization、parallel safety 和 rows？
```

核心矛盾：join 搜索必须不断剪枝，否则候选 path 数量会快速膨胀；但如果剪得太狠，后续 ORDER BY、LIMIT、nested loop 参数化、并行 gather 或 upper planning 可能失去真正有价值的输入顺序和启动成本。

学完后应能判断一个没有进入最终 EXPLAIN 的 join path 是从未生成、生成后被 add_path() 丢弃，还是保留下来但在 set_cheapest() 或上层阶段输掉。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讨论 GEQO 如何在大连接查询中用采样式搜索替代完整动态规划。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 upper path 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

```text
Query  -> PlannerInfo
base RelOptInfo  -> base Path  -> join RelOptInfo
join / upper Path  -> Plan  -> PlannedStmt
```

本节仍属于 join search 末端，不讨论新的 join order 枚举，也不重新讲 Nested Loop、Merge Join、Hash Join 的成本公式。

它关心的是：当 join path 已经被生成出来，planner 如何决定“这个候选还值得放在 pathlist 里”。

这个判断直接影响后续 upper planning，因为 ORDER BY、LIMIT、DISTINCT、Gather Merge 和 create_plan 只能消费已经幸存的 path。

下一节转入 upper planning，解释 scan/join 搜索完成后为什么还要继续构造一串 upper relation。

阅读这一组课程时，始终把函数名放回时间轴中：哪个状态先存在，哪个函数消费它，哪个阶段之后这个状态就不再是合法输入。

这个位置决定了本节不是函数索引。函数名只用来定位源码边界，真正要抓住的是 planner 如何把一个运行期现象压缩成可比较的候选状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`make_join_rel()` 和 `add_paths_to_joinrel()` 生成候选 join path，`add_path()` 在同一 `RelOptInfo` 的 pathlist 内做 dominance 比较，只有在成本、顺序、外部参数、行数和并行安全性都不再提供额外价值时才丢弃 path；所有候选生成完后，`set_cheapest()` 抽出 cheapest_total、cheapest_startup 和 cheapest_parameterized_paths。
```

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、RTE、RelOptInfo、统计信息、约束、GUC、hook 或已生成的下层 Path。 |
| 局部状态 | 只在一次 planner 调用中存在，通常挂在 `PlannerInfo`、`RelOptInfo` 或 Path 子类上。 |
| 正确性边界 | 不能破坏 SQL 语义、join 顺序约束、targetlist 语义、权限和 executor contract。 |
| 性能收益 | 减少不必要搜索，同时保留对后续阶段仍有价值的候选。 |
| 可观测结果 | 最终多半只能在 `EXPLAIN`、GUC 对照、断点和源码路径中间接还原。 |

```text
搜索空间爆炸
  vs
保留对后续阶段仍可能有用的非最低成本候选
```

这里的关键不是“哪个函数更重要”，而是一个候选什么时候仍有未来价值。

planner 的很多决策不是为了立刻生成 executor 节点，而是为了让后续阶段能在足够小的候选集合上继续比较。

因此，看到最终计划缺少某个节点，不能马上推断该节点从未被考虑；要先区分候选未生成、生成后被剪枝、保留后输给其它候选三种情况。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/optimizer/util/pathnode.c | `add_path()`、`add_partial_path()`、`set_cheapest()`、`compare_path_costs_fuzzily()`，本节主入口。 |
| 2 | src/include/optimizer/pathnode.h | `add_path()`、`set_cheapest()` 和 Path 构造器声明，便于先建立调用边界。 |
| 3 | src/backend/optimizer/path/joinrels.c | `join_search_one_level()`、`make_join_rel()` 何时触发 join path 生成。 |
| 4 | src/backend/optimizer/path/joinpath.c | `add_paths_to_joinrel()` 如何生成 nested loop、merge、hash join 候选。 |
| 5 | src/backend/optimizer/path/allpaths.c | `make_rel_from_joinlist()`、`generate_gather_paths()` 与 join 搜索收尾。 |
| 6 | src/backend/optimizer/path/pathkeys.c | `compare_pathkeys()` 与 pathkeys canonical comparison。 |
| 7 | src/backend/optimizer/path/costsize.c | 各类 path 的 rows、startup cost、total cost 来源。 |
| 8 | src/include/nodes/pathnodes.h | `RelOptInfo`、`Path`、`ParamPathInfo`、`JoinPath` 字段语义。 |
| 9 | src/backend/optimizer/README | parameterized path 与 path pruning 的设计背景。 |
| 10 | src/backend/optimizer/plan/planner.c | 后续 upper planning 如何依赖幸存 path。 |

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
| `RelOptInfo.pathlist` | 同一 relation 的普通候选 path 容器，`add_path()` 直接修改它。 |
| `RelOptInfo.partial_pathlist` | 并行 partial path 的候选容器，比较规则相似但使用 `add_partial_path()`。 |
| `RelOptInfo.cheapest_total_path` | 所有候选生成结束后由 `set_cheapest()` 选出的总成本代表。 |
| `RelOptInfo.cheapest_startup_path` | 只在 startup cost 对上层有意义时保留，常被 LIMIT 类需求放大。 |
| `RelOptInfo.cheapest_parameterized_paths` | 参数化路径集合，支持 lateral、相关子查询和 nested loop 内侧访问。 |
| `Path.pathkeys` | 输出顺序，不是 executor 节点类型；同样成本下可能决定一个 path 能否避免上层 sort。 |
| `Path.param_info` | 需要外层 relation 提供参数的语义边界，不能和普通 path 随意互相替代。 |
| `Path.parallel_safe` | 表示是否允许放入并行计划的安全性，不等于成本更低。 |
| `Path.rows` | 同一成本和顺序下用于裁剪的补充维度，尤其影响参数化 path。 |

raw field 不是语义。planner 中一个字段只有和所在阶段、候选集合、正确性边界一起看，才形成可用于诊断的语义。

同一个字段在不同阶段也可能只是中间判断。例如 pathkeys 在 Path 阶段是候选比较维度，进入 Plan 后会被转成 Sort key、Merge key 或直接消失。

### 状态组合如何服务主问题

**`RelOptInfo.pathlist`**：同一 relation 的普通候选 path 容器，`add_path()` 直接修改它。

在 join path 剪枝中，这个字段决定 pathlist、pathkeys、required_outer、cost 和 parallel_safe 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 add_path()/set_cheapest() 处观察它的值。

**`RelOptInfo.partial_pathlist`**：并行 partial path 的候选容器，比较规则相似但使用 `add_partial_path()`。

这个字段要和 joinrel 等价、参数化边界和输出顺序一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 add_paths_to_joinrel、add_path 和 create_ordered_paths 中已经发生过的候选保留或淘汰。

**`RelOptInfo.cheapest_total_path`**：所有候选生成结束后由 `set_cheapest()` 选出的总成本代表。

它的诊断价值在于定位 pathlist、pathkeys、required_outer、cost 和 parallel_safe 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

**`RelOptInfo.cheapest_startup_path`**：只在 startup cost 对上层有意义时保留，常被 LIMIT 类需求放大。

在 join path 剪枝中，这个字段决定 pathlist、pathkeys、required_outer、cost 和 parallel_safe 能否继续被后续阶段消费。

最终 EXPLAIN 只暴露胜出的 Plan；要解释原因，需要在 add_path()/set_cheapest() 处观察它的值。

**`RelOptInfo.cheapest_parameterized_paths`**：参数化路径集合，支持 lateral、相关子查询和 nested loop 内侧访问。

这个字段要和 joinrel 等价、参数化边界和输出顺序一起读；离开阶段边界就只剩一个 raw value。

如果只看最终节点，会漏掉 add_paths_to_joinrel、add_path 和 create_ordered_paths 中已经发生过的候选保留或淘汰。

**`Path.pathkeys`**：输出顺序，不是 executor 节点类型；同样成本下可能决定一个 path 能否避免上层 sort。

它的诊断价值在于定位 pathlist、pathkeys、required_outer、cost 和 parallel_safe 是在生成、剪枝、包装还是转换时发生变化。

它通常不会完整出现在 EXPLAIN 里，需要用断点或最小对照查询验证。

### 本节要避免的字段误读

- 把 cheapest_total_path 当作唯一重要候选，忽略 cheapest_startup_path。

- 以为 pathkeys 是 plan 节点属性，而不是 planner 对输出顺序的抽象。

- 以为禁用某类 join 只是把成本调高，忽略 `disabled_nodes` 的优先级。

- 看到最终没有 Sort 就断言 planner 没考虑排序成本。

这些误读的共同点，是把最终 Plan 中能看到的结果当成 planner 曾经拥有的全部信息。

## 5. 主流程源码 walkthrough

```text
join 搜索已经决定要把两个 RelOptInfo 合成一个 joinrel。
`make_join_rel()` 先检查合法性，再取得或创建目标 joinrel。
`add_paths_to_joinrel()` 按 join 类型、join qual、pathkeys、parameterization 生成候选。
每个候选进入 `add_path()` 前已经有 rows、startup cost、total cost、pathkeys 和 required_outer。
`add_path()` 先把 parameterized path 的 pathkeys 当作 NIL，避免组合数被排序维度继续放大。
`compare_path_costs_fuzzily()` 用 fuzzy 比较避免浮点细节制造平台相关计划差异。
若 startup 和 total cost 一个好一个差，两个 path 都可能保留。
若成本没有分出胜负，`compare_pathkeys()` 判断一个输出顺序是否支配另一个。
然后比较 required_outer、rows 和 parallel_safe，决定新 path、旧 path 或两者都留。
被新 path 支配的旧 path 会从 pathlist 移除；被旧 path 支配的新 path 不会进入 pathlist。
候选全部生成后，`set_cheapest()` 扫描幸存 path，写入 cheapest_* 字段。
后续 upper planning 只在这些幸存 path 上继续堆叠 sort、aggregate、window、limit 或 gather。
```

步骤 1：join 搜索已经决定要把两个 RelOptInfo 合成一个 joinrel。

这里要记录的是 pathlist、pathkeys、required_outer、cost 和 parallel_safe 如何改变，而不是只看函数是否返回。

断点优先打印 pathlist、pathkeys、required_outer、cost 和 parallel_safe，它们能区分候选、剪枝和转换。

步骤 2：`make_join_rel()` 先检查合法性，再取得或创建目标 joinrel。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 add_path()/set_cheapest() 之前未生成，还是之后被剪枝。

步骤 3：`add_paths_to_joinrel()` 按 join 类型、join qual、pathkeys、parameterization 生成候选。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

步骤 4：每个候选进入 `add_path()` 前已经有 rows、startup cost、total cost、pathkeys 和 required_outer。

这里要记录的是 pathlist、pathkeys、required_outer、cost 和 parallel_safe 如何改变，而不是只看函数是否返回。

建议同时记录输入 path 数量、输出 path 类型和与 join path 剪枝有关的关键字段。

步骤 5：`add_path()` 先把 parameterized path 的 pathkeys 当作 NIL，避免组合数被排序维度继续放大。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

不要把最终缺失直接写成未考虑；先确认候选是否进入过本阶段 pathlist。

步骤 6：`compare_path_costs_fuzzily()` 用 fuzzy 比较避免浮点细节制造平台相关计划差异。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

下一阶段还会继续包装或比较它，所以不要把当前 path 直接等同最终 Plan。

步骤 7：若 startup 和 total cost 一个好一个差，两个 path 都可能保留。

这里要记录的是 pathlist、pathkeys、required_outer、cost 和 parallel_safe 如何改变，而不是只看函数是否返回。

如果字段很多，先只打印能解释 joinrel 等价、参数化边界和输出顺序的那几项。

步骤 8：若成本没有分出胜负，`compare_pathkeys()` 判断一个输出顺序是否支配另一个。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

这里的分叉决定后续该查能力检查、成本比较，还是 createplan 转换。

步骤 9：然后比较 required_outer、rows 和 parallel_safe，决定新 path、旧 path 或两者都留。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

它的意义要等到后续阶段消费 pathlist、pathkeys、required_outer、cost 和 parallel_safe时才能完整解释。

步骤 10：被新 path 支配的旧 path 会从 pathlist 移除；被旧 path 支配的新 path 不会进入 pathlist。

这里要记录的是 pathlist、pathkeys、required_outer、cost 和 parallel_safe 如何改变，而不是只看函数是否返回。

断点优先打印 pathlist、pathkeys、required_outer、cost 和 parallel_safe，它们能区分候选、剪枝和转换。

步骤 11：候选全部生成后，`set_cheapest()` 扫描幸存 path，写入 cheapest_* 字段。

这个入口的关键输出是 planner-local 状态变化，要把写入字段记下来。

若最终计划缺少对应节点，先判断它在 add_path()/set_cheapest() 之前未生成，还是之后被剪枝。

步骤 12：后续 upper planning 只在这些幸存 path 上继续堆叠 sort、aggregate、window、limit 或 gather。

读到这一步时，优先确认它消费了哪个输入状态，又把结果挂到哪里。

此时输出仍是 planner-local 状态，只有进入 createplan 后才变成 executor contract。

### walkthrough 的读法

先顺着时间轴读一次，不追所有辅助函数。第二次只追改变状态集合的调用，例如添加 path、替换 target、创建 upperrel 或构造 Plan 节点。

第三次再进入成本函数和表达式函数。这样可以避免把课程读成 API 清单，也能保留源码里真实存在的历史耦合和边界条件。

## 6. 生命周期 / ownership / cleanup

| 阶段 | 状态推进 |
| --- | --- |
| 创建 | join path 由 `try_nestloop_path()`、`try_mergejoin_path()`、`try_hashjoin_path()` 等路径构造并交给 `add_path()`。 |
| 持有 | 幸存 path 由 planner memory context 持有，挂在同一个 `RelOptInfo` 的 pathlist 上。 |
| 删除 | `add_path()` 可立即 `pfree()` 被拒绝的 Path 节点，但不会深度释放共享子结构。 |
| 引用边界 | 更高层 joinrel 尚未生成时删除旧 path 是安全的；某些 IndexPath 被 BitmapHeapPath 引用时不能随意释放。 |
| 结束 | 整个 planner 结束后，planner context 批量释放这些 Path 和 RelOptInfo。 |

创建阶段的重点是：join path 由 `try_nestloop_path()`、`try_mergejoin_path()`、`try_hashjoin_path()` 等路径构造并交给 `add_path()`。

这份状态由 planner context 持有；只有 被 create_plan 选中的 best path 链 会进入后续 executor contract。

持有阶段的重点是：幸存 path 由 planner memory context 持有，挂在同一个 `RelOptInfo` 的 pathlist 上。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

删除阶段的重点是：`add_path()` 可立即 `pfree()` 被拒绝的 Path 节点，但不会深度释放共享子结构。

executor 不持有这个 Path 状态，只消费已经转换后的 Plan 字段。

引用边界阶段的重点是：更高层 joinrel 尚未生成时删除旧 path 是安全的；某些 IndexPath 被 BitmapHeapPath 引用时不能随意释放。

这份状态由 planner context 持有；只有 被 create_plan 选中的 best path 链 会进入后续 executor contract。

结束阶段的重点是：整个 planner 结束后，planner context 批量释放这些 Path 和 RelOptInfo。

正常返回或 ERROR 都依赖 planner context 清理这些搜索期对象。

planner 的内存释放主要依赖 MemoryContext，而不是逐个 `pfree()` 所有中间对象。只有少数热点剪枝路径会主动释放被淘汰的 Path 节点。

ERROR 发生时，当前 planner 调用栈会退出，planner context 跟随上层错误恢复边界释放；已经交给 executor 的 PlannedStmt 则有自己的生命周期。

这也是为什么课程要区分 Path、Plan 和 PlanState：它们分别属于搜索期、执行契约期和运行期。

## 7. 正确性机制层次

| 机制 | 本节作用 |
| --- | --- |
| 等价语义 | 同一 `RelOptInfo` 内的 path 必须输出同一 relation-like 结果，才能比较成本和顺序。 |
| 参数化边界 | `required_outer` 不同意味着调用者必须提供不同外层值，不能只按 total cost 比较。 |
| 顺序边界 | `pathkeys` 表示上层可复用的 ordering；丢掉它可能强迫后续额外 sort。 |
| 并行边界 | `parallel_safe` 不是优化偏好，而是可放入并行计划的安全条件。 |
| 浮点稳定性 | fuzzy comparison 接受微小差异，减少不同平台上的计划抖动。 |
| 中断响应 | `CHECK_FOR_INTERRUPTS()` 放在 `add_path()`，因为复杂 planner 会频繁经过这里。 |

`等价语义` 这一层保证的是：同一 `RelOptInfo` 内的 path 必须输出同一 relation-like 结果，才能比较成本和顺序。

它只能约束 joinrel 等价、参数化边界和输出顺序，不能代替成本比较、能力检查或 Plan 字段契约。

`参数化边界` 这一层保证的是：`required_outer` 不同意味着调用者必须提供不同外层值，不能只按 total cost 比较。

如果这个层面通过，仍要继续检查 try_*_path、add_path 和 disabled_nodes，以及后续 createplan contract。

`顺序边界` 这一层保证的是：`pathkeys` 表示上层可复用的 ordering；丢掉它可能强迫后续额外 sort。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

`并行边界` 这一层保证的是：`parallel_safe` 不是优化偏好，而是可放入并行计划的安全条件。

它只能约束 joinrel 等价、参数化边界和输出顺序，不能代替成本比较、能力检查或 Plan 字段契约。

`浮点稳定性` 这一层保证的是：fuzzy comparison 接受微小差异，减少不同平台上的计划抖动。

如果这个层面通过，仍要继续检查 try_*_path、add_path 和 disabled_nodes，以及后续 createplan contract。

`中断响应` 这一层保证的是：`CHECK_FOR_INTERRUPTS()` 放在 `add_path()`，因为复杂 planner 会频繁经过这里。

把这一层当成全部正确性，会漏掉 SQL 语义和 executor 消费边界。

优化器正确性通常不是一个 if 条件保证的，而是由语义阶段、状态字段、能力检查、fallback 和最终 executor contract 叠加出来。

阅读源码时，看到一个候选被跳过，先判断这是语义不合法、能力不支持、成本不值得，还是只是为了控制搜索空间。

## 8. 错误路径 / 异常路径 / fallback

| 路径 | 表现 |
| --- | --- |
| 没有 path | `set_cheapest()` 发现 pathlist 为空会报错，说明 planner 无法为该 relation 构造可执行方案。 |
| 过度参数化 | parameterized path 若没有外层 rel 提供参数，不能作为普通最终 path 使用。 |
| 排序假象 | parameterized path 的 pathkeys 在 `add_path()` 比较中被压成 NIL，这是有意剪枝策略。 |
| disabled path | `disabled_nodes` 在成本比较中高于 startup/total cost，是 GUC 影响搜索空间的入口。 |
| 内存压力 | 复杂 join search 中被淘汰 path 及时释放，避免 planner context 随候选爆炸。 |

没有 path：`set_cheapest()` 发现 pathlist 为空会报错，说明 planner 无法为该 relation 构造可执行方案。

诊断这类路径时，优先用 try_*_path、add_path 和 disabled_nodes确认它在生成前还是剪枝后消失。

过度参数化：parameterized path 若没有外层 rel 提供参数，不能作为普通最终 path 使用。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

排序假象：parameterized path 的 pathkeys 在 `add_path()` 比较中被压成 NIL，这是有意剪枝策略。

最小 SQL 加单变量对照，比继续堆 EXPLAIN 选项更容易定位边界。

disabled path：`disabled_nodes` 在成本比较中高于 startup/total cost，是 GUC 影响搜索空间的入口。

诊断这类路径时，优先用 try_*_path、add_path 和 disabled_nodes确认它在生成前还是剪枝后消失。

内存压力：复杂 join search 中被淘汰 path 及时释放，避免 planner context 随候选爆炸。

最终 EXPLAIN 多半只显示 fallback 后的胜出路径，不能证明原候选从未出现。

fallback 不等于“随便选一个还能跑的方案”。在 planner 中，fallback 往往意味着放弃某种优化机会，同时保留语义正确性。

如果 fallback 发生在候选生成前，最终计划看起来就像从未考虑过另一种策略；如果发生在剪枝后，则需要检查 pathlist 中是否曾经有对应候选。

## 9. 成本、资源与跨模块传播

| 成本来源 | 传播方式 |
| --- | --- |
| 候选数量 | join relation 数、join qual 数、pathkeys 组合和参数化组合都会增加 `add_path()` 调用次数。 |
| 比较成本 | 每个新 path 要与当前 pathlist 比较，pathlist 越长，planner CPU 越高。 |
| 排序复用 | 保留有用 pathkeys 可能减少上层 sort，牺牲一点本层 total cost 是合理的。 |
| LIMIT 效应 | startup cost 在 LIMIT 或游标场景中可能比 total cost 更关键。 |
| 并行传播 | parallel_safe 幸存 path 才能被后续 Gather 或 Gather Merge 利用。 |
| 估算误差 | rows 错误会改变 dominance 判断，让真正有价值的 path 被过早淘汰或保留过多。 |

候选数量：join relation 数、join qual 数、pathkeys 组合和参数化组合都会增加 `add_path()` 调用次数。

这个成本会先影响 compare_path_costs_fuzzily 与 pathkeys dominance，再通过幸存 Path 传到下一阶段或 Plan。

比较成本：每个新 path 要与当前 pathlist 比较，pathlist 越长，planner CPU 越高。

它不是执行毫秒数，而是候选之间排序的相对信号。

排序复用：保留有用 pathkeys 可能减少上层 sort，牺牲一点本层 total cost 是合理的。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

LIMIT 效应：startup cost 在 LIMIT 或游标场景中可能比 total cost 更关键。

这个成本会先影响 compare_path_costs_fuzzily 与 pathkeys dominance，再通过幸存 Path 传到下一阶段或 Plan。

并行传播：parallel_safe 幸存 path 才能被后续 Gather 或 Gather Merge 利用。

它不是执行毫秒数，而是候选之间排序的相对信号。

估算误差：rows 错误会改变 dominance 判断，让真正有价值的 path 被过早淘汰或保留过多。

一旦对应 Path 被剪枝，这个成本也不会再影响后续阶段。

本节相关成本至少跨越 optimizer、executor 和统计信息三个边界。统计信息影响 rows，rows 影响 path cost，path cost 影响 Plan，Plan 再影响执行期资源使用。

不要把成本误读成真实耗时。PostgreSQL planner cost 是相对比较单位，目的是在候选之间排序，而不是预测毫秒数。

资源压力会迁移。一个阶段节省排序，可能把压力转移到 hash table；一个阶段保留并行，可能把压力转移到 Gather 通信或 finalization。

## 10. 观测与诊断入口

观测入口 1：用 `EXPLAIN (ANALYZE, BUFFERS)` 只能看到最终 Plan，看不到被 `add_path()` 丢弃的 path。

看到这个现象后，先回到add_paths_to_joinrel、add_path 和 create_ordered_paths，不要只根据节点名归因。

观测入口 2：改变 `enable_hashjoin`、`enable_mergejoin`、`enable_nestloop` 可以观察 disabled path 对搜索的影响。

先确认这是输入事实、候选生成、剪枝结果，还是 Path 到 Plan 的转换结果。

观测入口 3：给查询增加 `ORDER BY` 或 `LIMIT`，比较计划是否转向保留排序或低 startup 的 path。

节点名只是最终形态，真正原因通常在pathlist、pathkeys、required_outer、cost 和 parallel_safe的变化中。

观测入口 4：LATERAL 或相关子查询能制造 parameterized path，适合断点观察 `param_info`。

看到这个现象后，先回到add_paths_to_joinrel、add_path 和 create_ordered_paths，不要只根据节点名归因。

观测入口 5：在 `add_path()` 设置断点，记录 `pathkeys`、`startup_cost`、`total_cost`、`required_outer` 和 `parallel_safe`，比只看 EXPLAIN 更接近真实原因。

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

### join path 剪枝的源码细读节点

下面这组检查点用于把最终计划现象追回到源码中的状态变化。它不是函数索引，而是按调试时最常用的状态边界组织。

检查点 1：`compare_path_costs_fuzzily()`

先比较 total cost，再判断 startup cost 是否足以让两个候选都保留；这解释了 LIMIT 场景中低 startup path 的生存空间。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 pathlist、pathkeys、required_outer、cost 和 parallel_safe，再对照最终 EXPLAIN。

检查点 2：`disabled_nodes`

禁用路径类型的计数优先于普通 cost；这让 `enable_*` GUC 成为搜索空间控制，而不是简单加一点惩罚。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 3：`PATH_REQ_OUTER()`

比较 required outer rels 时要看集合包含关系，参数化更少的 path 才可能支配参数化更多的 path。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

检查点 4：`compare_pathkeys()`

ordering 不是附加说明，而是 dominance 维度；更好顺序可能保留一个 total cost 稍高的 path。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

如果信息过多，先打印能解释 join path 剪枝的字段，再补 rows 和 cost。

检查点 5：`Path.rows`

成本和顺序接近时，rows 成为裁剪维度；但 rows 本身来自估算，所以错误会传播到 path pruning。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

若它从未生成，排查try_*_path、add_path 和 disabled_nodes；若生成后消失，再看剪枝规则。

检查点 6：`parallel_safe`

并行安全参与 dominance，是为了给后续 Gather 或 upper parallel path 留机会。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

Path 层只能说明候选语义，executor contract 要看具体 Plan 字段。

检查点 7：`param_info ? NIL : pathkeys`

parameterized path 的 pathkeys 在比较中被故意忽略，用来控制参数化排序组合爆炸。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

记录字段时要标明它属于输入 path、输出 path，还是后续 Plan。

检查点 8：`set_cheapest()`

它不生成 path，只从幸存 pathlist 中抽取 cheapest slots；不要把它当成搜索算法。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

这个判断能避免把能力限制误写成成本输掉。

检查点 9：`cheapest_parameterized_paths`

这里保留的是可供 nested loop 内侧或 lateral 依赖消费的路径集合。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

尤其是 targetlist、Param、sort key 和 group key，必须追到转换函数。

检查点 10：`add_partial_path()`

partial path 使用相似剪枝思想，但目标是 worker 局部结果，不能直接等同普通 path。

读这个检查点时，先确认它消费的输入来自哪一阶段，再看它写回哪个字段。

断点实验建议打印 pathlist、pathkeys、required_outer、cost 和 parallel_safe，再对照最终 EXPLAIN。

检查点 11：`CHECK_FOR_INTERRUPTS()`

复杂 planner 可能长时间循环，`add_path()` 的高频调用点承担了响应取消的职责。

把它放回时间轴，可以区分候选生成、候选裁剪和 Plan 转换。

如果 EXPLAIN 看不到它，就用 pathlist/cheapest 字段判断它是否曾经存在。

检查点 12：IndexPath 释放特例

被 BitmapHeapPath 引用的 IndexPath 不能随便 pfree，说明 path 节点之间也可能有共享子结构。

不要从函数顶部平铺读；先找这个检查点改变了哪份状态。

如果这个状态 会进入 Plan，继续追 createplan.c 或 setrefs.c 的字段转换。

### 边界案例：从现象回到状态

案例 1：ORDER BY 导致非最低 total path 留下

一个 Merge Join path total cost 略高，但 pathkeys 正好满足上层 ORDER BY。若只看 join 层 total cost，会误以为应当被丢弃；真正判断要看 `compare_pathkeys()` 和后续 `create_ordered_paths()` 是否能省掉 Sort。

分析这个案例时，先固定最小 SQL，再改变一个输入因素，并在 add_path()/set_cheapest() 验证。

节点名只是终点；本课要把原因写回pathlist、pathkeys、required_outer、cost 和 parallel_safe和源码状态。

案例 2：LATERAL 让参数化 path 存活

内侧 index path 依赖外层 relation，不能拿来和普通 unparameterized path 直接互换。调试时打印 `PATH_REQ_OUTER()`，才能解释它为什么既不支配普通 path，也不被普通 path 简单取代。

案例复盘要写清楚候选是否生成、在哪个字段变化、最终如何进入 Plan。

真正的因果通常发生在候选生成、能力检查、剪枝或 Path 到 Plan 转换之间。

案例 3：GUC 禁用 join 类型后的计划变化

设置 `enable_hashjoin=off` 不是把 Hash Join 成本调到稍高，而是在 `disabled_nodes` 维度改变比较优先级。最终计划变化需要回到 `compare_path_costs_fuzzily()` 的第一段判断。

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

信号 1：final plan 避免了 Sort

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到pathlist、pathkeys、required_outer、cost 和 parallel_safe字段，而不是停在 EXPLAIN 节点名。

信号 2：LIMIT 后 join 类型改变

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

信号 3：LATERAL 内侧出现 Index Scan

如果复现不稳定，先固定数据量、ANALYZE 状态和相关成本参数。

把这个分界写进记录，后续分析才不会混淆。

如果无法指出字段，就说明还需要回到源码断点。

信号 4：禁用 hashjoin 后仍没有 merge join

先确认这个现象可重复，再只改变一个因素，避免多个 planner 机制同时变化。

若变化发生在生成前，断点不会命中目标候选；若发生在剪枝后，pathlist 或 cheapest 字段会变化。

最终解释要落到pathlist、pathkeys、required_outer、cost 和 parallel_safe字段，而不是停在 EXPLAIN 节点名。

信号 5：并行 path 没有进入 final plan

对照实验只动一个输入：索引、GUC、统计信息、work_mem 或 LIMIT。

这个分界决定下一步查能力检查还是查 add_path/set_cheapest。

节点名负责描述结果；字段变化负责解释原因。

### 最小化 SQL 练习形态

建议用这些形态做最小复现，然后再逐步加回真实业务 SQL 的复杂部分。

形态 1：两个表等值连接并带 ORDER BY join key

最小复现只保留能触发 join path 剪枝的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较pathlist、pathkeys、required_outer、cost 和 parallel_safe。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

形态 2：同一连接查询分别加和不加 LIMIT 1

先把查询压缩到只出现本课关心的节点，再逐步加回业务复杂度。

每次实验都保存 SQL、EXPLAIN、ANALYZE 状态和改动项。

有些机制只在行数、选择率或排序键规模达到阈值后才可见。

形态 3：LATERAL 子查询内侧按外层 key 查索引

无关 join、函数和投影会干扰add_path()/set_cheapest()的判断。

对照目标是让一个字段变化清楚地解释最终计划变化。

如果候选没有生成，继续放大数据规模或改变单一 GUC 做验证。

形态 4：切换 enable_hashjoin / enable_mergejoin 做对照

最小复现只保留能触发 join path 剪枝的表、索引、表达式或 GUC。

先记录默认计划，再只改变一个条件，比较pathlist、pathkeys、required_outer、cost 和 parallel_safe。

计划不变时，先检查数据规模、统计信息和成本参数是否足以触发替代候选。

### 推荐断点顺序

断点顺序应当沿状态推进排列。这样能区分“没有生成候选”和“候选生成后输掉”。

1. `make_join_rel`

在这个位置打印 pathlist、pathkeys、required_outer、cost 和 parallel_safe，并标明它属于输入 path、输出 path 还是 Plan 字段。

2. `add_paths_to_joinrel`

不要无差别打印全部结构；只打印能解释 join path 剪枝的字段。

3. `add_path`

记录前后差异，比记录单次静态值更有诊断价值。

4. `compare_path_costs_fuzzily`

在这个位置打印 pathlist、pathkeys、required_outer、cost 和 parallel_safe，并标明它属于输入 path、输出 path 还是 Plan 字段。

5. `set_cheapest`

不要无差别打印全部结构；只打印能解释 join path 剪枝的字段。

6. `create_ordered_paths`

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

- 把 cheapest_total_path 当作唯一重要候选，忽略 cheapest_startup_path。

纠正方法是把最终节点放回add_path()/set_cheapest()的时间轴，确认对应状态何时改变。

- 以为 pathkeys 是 plan 节点属性，而不是 planner 对输出顺序的抽象。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 以为禁用某类 join 只是把成本调高，忽略 `disabled_nodes` 的优先级。

如果无法定位阶段，就不能直接写成确定结论。

- 看到最终没有 Sort 就断言 planner 没考虑排序成本。

纠正方法是把最终节点放回add_path()/set_cheapest()的时间轴，确认对应状态何时改变。

- 把 parameterized path 的低成本误认为可作为任意位置的全局最优。

先问这个节点来自候选生成、剪枝、包装还是 Plan 转换。

- 认为被丢弃的 path 一定语义错误；多数时候只是被支配。

如果无法定位阶段，就不能直接写成确定结论。

这些误区在慢 SQL 诊断中很常见，因为最终计划隐藏了大量候选生成和淘汰历史。

## 12. 课堂实验

### 实验 1：排序复用实验

创建两表 join，并在 join key 上建立能提供顺序的索引，对比有无 `ORDER BY join_key` 时 Merge Join 或 Index path 的选择。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 2：startup cost 实验

在相同查询上加 `LIMIT 1`，观察 planner 是否偏向低 startup path，再回到 `compare_path_costs_fuzzily()` 看为什么两类 cost 不能压成一个数字。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

### 实验 3：参数化路径实验

写一个 LATERAL 子查询，断点停在 `add_path()`，记录 `PATH_REQ_OUTER()` 与普通 path 的比较结果。

实验步骤：

1. 先写一个最小 SQL，只保留触发本节机制所需的表、索引和表达式。
2. 用 `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)` 记录最终计划和估算误差。
3. 改动一个 GUC、索引、统计信息或 SQL 形态，保持其它条件不变。
4. 在本节列出的入口函数设置断点，记录关键字段是否变化。
5. 把观察结果写回主流程，说明变化发生在候选生成、剪枝还是 Plan 转换阶段。

实验判断标准不是“计划变了没有”，而是能否解释变化发生在哪个状态边界。

## 13. 讨论题

1. 为什么 parameterized path 在 `add_path()` 中不靠 pathkeys 赢得排序比较？

回答时要指向一个具体状态或源码入口，而不是只给结论。

2. 如果只保留 total cost 最低的 path，ORDER BY 或 LIMIT 可能丢失什么机会？

回答时要指向一个具体状态或源码入口，而不是只给结论。

3. `parallel_safe` 为什么参与 dominance，而不是留到 executor 再判断？

回答时要指向一个具体状态或源码入口，而不是只给结论。

4. fuzzy cost comparison 减少了什么类型的计划不稳定？

回答时要指向一个具体状态或源码入口，而不是只给结论。

5. 被 `add_path()` 淘汰的 path 能从 EXPLAIN 直接看到吗？如果不能，应如何验证？

回答时要指向一个具体状态或源码入口，而不是只给结论。

6. 哪些 rows 估算错误会改变 path pruning 的方向？

回答时要指向一个具体状态或源码入口，而不是只给结论。

如果一个回答同时引入两个同等重要的主问题，应当拆成两节课，而不是在本节继续展开。

## 14. 本节小结

- join path pruning 的核心不是“找最低成本”，而是保留不被支配的少量实现方式。

- `startup_cost`、`total_cost`、`pathkeys`、`required_outer`、`rows`、`parallel_safe` 共同决定一个 path 是否还有后续价值。

- `set_cheapest()` 是候选生成后的抽取动作，不是替代 `add_path()` 的搜索策略。

- 最终 EXPLAIN 只能展示胜出 Plan，不能展示 pathlist 里的淘汰历史。

- 可迁移规律是：优化器中的剪枝必须剪掉冗余，不得剪掉未来阶段仍能消费的信息。

本节的可迁移 mental model 是：先区分语义边界，再看候选如何被生成、保留、剪枝或契约化。

这种读法可以迁移到 planner 之外的 PostgreSQL 内核模块：不要把最终可见状态误认为完整运行过程，要追踪状态在生命周期中的推进和收束。
