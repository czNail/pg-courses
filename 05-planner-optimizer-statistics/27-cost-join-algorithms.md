# PostgreSQL Nested Loop、Merge Join 与 Hash Join 成本

## 课程定位

前置知识：已经理解 scan、sort、hash 与 materialize 的局部成本模型。

本节唯一主问题：

```text
planner 为什么会在 Nested Loop、Merge Join 与 Hash Join 之间切换；rows 估错时，为什么 join 类型错误常比单个 scan 错误更快放大成慢 SQL？
```

核心矛盾：join algorithm cost 同时依赖外层 rows、内层 rescan、排序前提、hash build 内存、join qual CPU 和参数化边界；这些输入本身又来自下层 Path 的估算和剪枝。

学完后应能从 EXPLAIN 的 loops、Sort、Hash Batches、Join Filter 和 Index Cond 反推 join cost 的主要驱动变量。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节把 Sort / Hash / Material 的局部成本讲清楚，本节看这些局部成本如何进入 join 算法选择。

本节不讲 join order 动态规划；这里假设 joinrel 已经被构造，只看三类 join path 的 cost 与可行性。

下一组课程会回到 RelOptInfo 和 Path，解释这些 join 候选如何存储、剪枝和转成 Plan。

本组课程的推进顺序是：

```text
selectivity / rows
  -> cost model
  -> scan / memory / join / parallel path
  -> RelOptInfo / Path / PathTarget
  -> parameterized path
  -> base path、join search、upper planning 和 createplan
```

这一节阅读时只跟一条状态链：

```text
输入事实
  -> RelOptInfo / Path / PathTarget 中的 planner-local 状态
  -> cost 或 legality 判断
  -> pathlist 中的保留或淘汰
  -> EXPLAIN 中能看到的最终影子
```

如果某个函数没有改变这条链上的状态，可以先作为旁路阅读，不要把课程读成函数清单。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
`joinpath.c` 枚举可行 join path；`try_nestloop_path()`、`try_mergejoin_path()`、`try_hashjoin_path()` 先做参数化与 legality 检查，再调用 `initial_cost_*` 得到可剪枝下界；通过 precheck 后，`pathnode.c` 创建具体 JoinPath，并由 `final_cost_*` 写入 rows、startup_cost、total_cost 和 node-specific 状态。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `JoinCostWorkspace` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `outer_path->rows` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `inner_path->rows` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `restrictlist` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `hashclauses` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/joinpath.c` | `add_paths_to_joinrel()`、`try_nestloop_path()`、`try_mergejoin_path()`、`try_hashjoin_path()` 枚举 join algorithm。 |
| 2 | `src/backend/optimizer/path/costsize.c` | `initial_cost_nestloop()`、`final_cost_nestloop()`、`initial_cost_mergejoin()`、`final_cost_mergejoin()`、`initial_cost_hashjoin()`、`final_cost_hashjoin()`。 |
| 3 | `src/backend/optimizer/util/pathnode.c` | `create_nestloop_path()`、`create_mergejoin_path()`、`create_hashjoin_path()` 写入 JoinPath。 |
| 4 | `src/backend/optimizer/path/joinrels.c` | `make_join_rel()` 与 `join_is_legal()` 保证 joinrel 合法。 |
| 5 | `src/backend/optimizer/util/relnode.c` | `build_join_rel()` 保存 joinrel rows、reltarget、joininfo 和 lateral 边界。 |
| 6 | `src/include/nodes/pathnodes.h` | `JoinPath`、`NestPath`、`MergePath`、`HashPath`、`JoinCostWorkspace`、`SpecialJoinInfo`。 |
| 7 | `src/include/optimizer/paths.h` | join path 生成入口和 hook 声明。 |

推荐阅读路径：

```text
先读状态结构
  -> 找入口函数
  -> 找写入 rows/cost/required_outer/target 的语句
  -> 找 add_path / set_cheapest / create_plan 消费点
  -> 回到 EXPLAIN 或断点观察公开影子
```

注意保留源码里的真实形状：一些判断分散在 `allpaths.c`、`pathnode.c`、`costsize.c` 和 `createplan.c`，这不是文档组织问题，而是 optimizer 在搜索、计价和执行契约之间切换的结果。

## 4. 可复现运行现象

本节从能观察到的计划变化进入源码，而不是先背所有函数名。

### 4.1. 小外层 + 参数化索引内层

Nested Loop 在外层 rows 小且内层 index qual 完整时很强；外层 rows 估低会把 loops 放大成灾难。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. 有序输入省 Sort

Merge Join 可复用 pathkeys；如果两侧都要排序，成本会先叠加 Sort。


### 4.3. Hash Join 受内层 build 约束

inner path 宽、rows 大或 work_mem 小会让 numbatches 上升。


### 4.4. SEMI / ANTI 提前停止

`final_cost_nestloop()` 和 `final_cost_hashjoin()` 对首个匹配后的停止有专门分支。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `JoinCostWorkspace` | initial 与 final cost 的交接区 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `outer_path->rows` | 外层循环规模 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `inner_path->rows` | 内层扫描或 build 规模 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `restrictlist` | join 节点需要检查的 clause | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `hashclauses` | Hash Join 可用等值条件 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `mergeclauses` | Merge Join 可用排序等值条件 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `semifactors` | SEMI/ANTI 匹配比例 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `inner_unique` | 内层是否唯一 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `materialize_inner` | Merge Join 内层物化标志 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `skip_mark_restore` | Merge Join 是否省 mark/restore | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `num_batches` | Hash Join 批次数 | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |
| `required_outer` | join path 仍依赖的外层 relids | `src/backend/optimizer/path/joinpath.c` 等主线文件消费或写入。 |

### 5.1. `JoinCostWorkspace`

语义：initial 与 final cost 的交接区。

来源：initial 阶段保存 startup、total 下界和私有中间量。

消费：final 阶段复用它补 CPU、选择率、batch、materialize 等成本。

偏差后果：它解释了 join cost 为何分两段计算。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `outer_path->rows`

语义：外层循环规模。

来源：来自被选 outer Path 的 rows。

消费：Nested Loop 直接用它放大 inner rescan；Hash/Merge 也用于 CPU 和输出估算。

偏差后果：低估外层 rows 是 loops 爆炸的常见入口。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `inner_path->rows`

语义：内层扫描或 build 规模。

来源：来自 inner Path。

消费：Nested Loop 用于每次 rescan，Hash Join 用于 hash table，Merge Join 用于排序和比较。

偏差后果：宽而大的 inner 对 Hash Join 特别敏感。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `restrictlist`

语义：join 节点需要检查的 clause。

来源：由 joinrel 构造与 parameterization 下推共同决定。

消费：final cost 对 joinrestrictinfo 计算 CPU。

偏差后果：被下推到 index qual 的 clause 不应在同一层重复解释。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `hashclauses`

语义：Hash Join 可用等值条件。

来源：来自 hashable join 条件筛选。

消费：没有它就不能生成 Hash Join path。

偏差后果：Join Filter 不是 Hash Cond，不能支撑 hash table lookup。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `mergeclauses`

语义：Merge Join 可用排序等值条件。

来源：来自 mergejoinable 条件与 pathkeys。

消费：决定是否需要 sort，以及 mark/restore 语义。

偏差后果：排序方向、nulls ordering 和 opfamily 都会参与。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `semifactors`

语义：SEMI/ANTI 匹配比例。

来源：`compute_semi_anti_join_factors()` 提供。

消费：final cost 估外层匹配行数和平均匹配次数。

偏差后果：EXISTS 子查询慢时，这个估算常是关键。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `inner_unique`

语义：内层是否唯一。

来源：由唯一性推导或 unique-ified path 传入。

消费：允许 Nested Loop / Hash Join 在首个匹配后停止。

偏差后果：唯一性证明失败会让成本保守上升。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `materialize_inner`

语义：Merge Join 内层物化标志。

来源：`final_cost_mergejoin()` 设置。

消费：影响 mark/restore、sort spill 和 rescan 成本。

偏差后果：它不是普通 Path 字段，而是 MergePath 的执行契约。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `skip_mark_restore`

语义：Merge Join 是否省 mark/restore。

来源：特定 join 类型与唯一性允许跳过。

消费：影响 executor 是否需要在内层回退。

偏差后果：只从 EXPLAIN 不一定能直接看到，需要源码或调试器。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `num_batches`

语义：Hash Join 批次数。

来源：`final_cost_hashjoin()` 写入 HashPath。

消费：createplan 后进入 Hash 节点预期。

偏差后果：实际 Batches 偏离常回到 rows、width、skew。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.12. `required_outer`

语义：join path 仍依赖的外层 relids。

来源：`calc_nestloop_required_outer()` 或 `calc_non_nestloop_required_outer()` 计算。

消费：决定 path 能否继续参与上层 join。

偏差后果：参数化错误会让候选被拒绝，而不是成本变高。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
add_paths_to_joinrel()
  -> match_unsorted_outer()
  -> try_nestloop_path()
  -> initial_cost_nestloop()
  -> final_cost_nestloop()
  -> try_mergejoin_path()
  -> initial_cost_mergejoin()
  -> final_cost_mergejoin()
  -> try_hashjoin_path()
  -> initial_cost_hashjoin()
  -> final_cost_hashjoin()
  -> add_path()
```

### 6.1. `add_paths_to_joinrel()`

拿到一对 outerrel/innerrel 和 restrictlist 后，为同一个 joinrel 尝试多种 algorithm。

观察锚点：`JoinCostWorkspace`。

### 6.2. `match_unsorted_outer()`

以未排序 outer 为中心枚举 Nested Loop 和 Hash Join 的常见组合。

观察锚点：`outer_path->rows`。

### 6.3. `try_nestloop_path()`

先计算 required_outer，拒绝不合理参数化，再调用 initial_cost_nestloop()。

观察锚点：`inner_path->rows`。

### 6.4. `initial_cost_nestloop()`

只估下层 path startup/run 和 inner rescan 下界，暂不展开昂贵 join qual。

观察锚点：`restrictlist`。

### 6.5. `final_cost_nestloop()`

补 SEMI/ANTI early stop、indexed join qual、restrict qual CPU 和 tlist cost。

观察锚点：`hashclauses`。

### 6.6. `try_mergejoin_path()`

检查 mergeclauses、排序需求和 required_outer，再用 precheck 控制候选数量。

观察锚点：`mergeclauses`。

### 6.7. `initial_cost_mergejoin()`

把必要 sort / incremental sort 成本先算进去，并估两侧扫描比例。

观察锚点：`semifactors`。

### 6.8. `final_cost_mergejoin()`

设置 skip_mark_restore / materialize_inner，再补 merge qual、qp qual 和输出 target 成本。

观察锚点：`inner_unique`。

### 6.9. `try_hashjoin_path()`

只在有 hashclauses 时生成 HashPath，并检查非 NestLoop 参数化是否合法。

观察锚点：`materialize_inner`。

### 6.10. `initial_cost_hashjoin()`

把 inner build、hash function CPU、batches I/O 下界写入 workspace。

观察锚点：`skip_mark_restore`。

### 6.11. `final_cost_hashjoin()`

估 bucket size、MCV、hash qual CPU、输出 rows，并写入 `num_batches`。

观察锚点：`num_batches`。

### 6.12. `add_path()`

同一 joinrel 上多种 algorithm 最终按成本、pathkeys、参数化和 parallel 安全性共存或淘汰。

观察锚点：`required_outer`。

## 7. 生命周期 / ownership / cleanup

这些对象都属于一次 planner invocation。

`PlannerInfo` 持有规划上下文，Path、RelOptInfo、ParamPathInfo、PathTarget、cost workspace 和 List 节点大多在这个上下文中分配。

正常路径中，候选对象在 planner 阶段不断创建、比较、剪枝和被 cheapest 指针引用。

`add_path()` 可以释放被拒绝的 Path 节点，但不会盲目释放共享子结构；IndexPath 还有被 bitmap path 引用的特殊边界。

ERROR 路径不依赖逐个 pfree，而是依赖 PostgreSQL 的 MemoryContext cleanup。

createplan 之后，executor 拿到的是 Plan tree，而不是整个 planner 搜索图。

因此调试本节主题时，最好的现场在 planner 阶段；等 executor 启动后，大多数候选已经不可见。

如果扩展 hook 在 planner 中插入自定义 Path，也应遵守同样的上下文生命周期和字段契约。

## 8. 正确性机制层次

| 层次 | 作用 | 本节关注点 |
| --- | --- | --- |
| SQL 语义 | 保证结果集合、NULL 语义、排序/分组/参数依赖不被改变 | legality 先于 cost。 |
| planner 状态 | 把语义树映射成可搜索、可剪枝的候选状态 | RelOptInfo / Path / PPI / target 必须一致。 |
| 成本模型 | 在合法候选之间选择相对便宜者 | startup、total、rows、width、I/O、CPU、memory、parallel 都是近似。 |
| 执行契约 | 最终 Plan 必须携带 executor 所需信息 | 未选中的 planner 状态不会补救执行期缺失。 |

这四层不能混成一句“优化器觉得更便宜”。

一个候选能否生成，首先看语义与执行契约。

一个候选能否留下，再看 cost、pathkeys、parameterization、parallel safety 和剪枝策略。

一个计划运行是否快，还要看执行期数据、缓存、worker、临时文件和统计偏差。

## 9. 错误路径 / 异常路径 / fallback

fallback 的危险在于它经常返回一个看似正常的数字或一个合法但退化的候选。

### 9.1. 无 hashable clause

Hash Join 候选不会生成；这不是 cost 太高，而是语义输入缺失。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. 无 mergejoinable clause

Merge Join 候选不会生成，或者需要额外排序但仍缺少可比较 pathkeys。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. Nested Loop 全内层重扫

`has_indexed_join_quals()` 不成立时，未匹配 outer row 也可能付出昂贵 inner scan。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. Hash Join 分批

`numbatches > 1` 时 planner 计入临时批次 I/O；执行期可能因 skew 继续增批。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. Merge Join 物化

内层不支持 mark/restore 或排序可能 spill 时，Materialize 可能出现。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. enable GUC 影响算法

关闭 enable_hashjoin 等不会删除所有可能性，而是通过 disabled_nodes 参与比较。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

join 算法的关键是先看算法前提，再看 rows 放大、rescan、sort/hash build 和 join qual CPU。

状态传播可以按这一条链追：

```text
catalog / statistics / GUC / SQL shape
  -> planner-local state
  -> legality 或 cost 判断
  -> pathlist / partial_pathlist / cheapest 指针
  -> createplan.c 执行契约
  -> EXPLAIN 与 executor instrumentation
```

| 切入点 | 源码锚点 | 下游影响 |
| --- | --- | --- |
| Hash Cond / Merge Cond 是否真实存在 | `add_paths_to_joinrel()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| outer rows 是否放大 inner rescan | `match_unsorted_outer()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| inner path 是否 indexed join quals | `try_nestloop_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Sort 成本是否被并入 Merge Join | `initial_cost_nestloop()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Hash Join numbatches 是否过大 | `final_cost_nestloop()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| SEMI/ANTI 是否 early stop | `try_mergejoin_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| required_outer 是否限制算法 | `initial_cost_mergejoin()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| disabled_nodes 是否来自 enable GUC | `final_cost_mergejoin()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

这张表不替代源码阅读；它只是把排查顺序固定下来，避免直接从最终 Plan 反推所有原因。

## 11. 观测与诊断入口

公开观测从 EXPLAIN 开始，但不能停在 EXPLAIN。

| 入口 | 看什么 | 回到源码哪里 |
| --- | --- | --- |
| `EXPLAIN` | node type、rows、width、startup/total、workers、sort/hash 附属信息 | Path 成本写入点。 |
| `EXPLAIN (ANALYZE, BUFFERS)` | actual rows、loops、Buffers、Temp、Batches、Workers Launched | rows/width/memory/parallel 偏差。 |
| `pg_stats` / `pg_class` | ndistinct、MCV、correlation、relpages、reltuples | selectivity 与 relation size 来源。 |
| GDB 断点 | RelOptInfo、Path、PPI、target 字段 | 本节第 3 节列出的入口函数。 |

推荐断点组合：

- 在候选生成入口断一次，确认候选是否存在。
- 在 cost 函数断一次，记录输入和输出字段。
- 在 `add_path()` 或 `add_partial_path()` 断一次，确认候选是留下还是被淘汰。
- 在 `set_cheapest()` 断一次，确认后续阶段实际拿哪个 Path。
- 在 `create_plan()` 或 `create_plan_recurse()` 断一次，确认 executor contract 是否保留了需要的信息。

- 现场记录 `Hash Cond / Merge Cond 是否真实存在` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `outer rows 是否放大 inner rescan` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `inner path 是否 indexed join quals` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Sort 成本是否被并入 Merge Join` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Hash Join numbatches 是否过大` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `SEMI/ANTI 是否 early stop` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `required_outer 是否限制算法` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `disabled_nodes 是否来自 enable GUC` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 把 join 类型当固定优劣排序；每类算法都有输入前提。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 只调 enable_nestloop 证明“更快”；这只能说明另一个候选存在，不说明原估算错在哪里。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 忽略 inner rescan；Nested Loop 的核心成本常在 inner_path 的第二次以后。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 把 Join Filter 当 Hash Cond；只有 hashclauses 才能构建 hash table。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 只看 join 输出 rows，不看处理 tuple 数；join qual CPU 常按被检查的组合数计费。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. Nested Loop loops 放大

SQL：

```sql
CREATE TABLE j_outer AS SELECT g, g AS k FROM generate_series(1,1000) g;
CREATE TABLE j_inner AS SELECT g, g AS k, md5(g::text) AS v FROM generate_series(1,1000000) g;
CREATE INDEX ON j_inner(k);
ANALYZE j_outer; ANALYZE j_inner;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM j_outer o JOIN j_inner i ON i.k = o.k WHERE o.k < 10;
```

预期观察：看 outer rows 与 inner loops 是否匹配估算。

源码回看：跟 `final_cost_nestloop()` 的 `outer_path_rows` 和 `has_indexed_join_quals()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. Hash Join batches

SQL：

```sql
SET work_mem = 2MB;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM j_outer o JOIN j_inner i ON i.k = o.k;
```

预期观察：观察 Hash 节点 Batches 和 Memory Usage。

源码回看：跟 `initial_cost_hashjoin()` 的 `numbatches`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. Merge Join sort 前提

SQL：

```sql
SET enable_hashjoin = off;
SET enable_nestloop = off;
EXPLAIN SELECT * FROM j_outer o JOIN j_inner i ON i.k = o.k ORDER BY o.k;
```

预期观察：看两侧是否需要 Sort，或索引 pathkeys 是否被利用。

源码回看：跟 `initial_cost_mergejoin()` 中 `outersortkeys` / `innersortkeys`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. SEMI join early stop

SQL：

```sql
EXPLAIN SELECT * FROM j_outer o WHERE EXISTS (SELECT 1 FROM j_inner i WHERE i.k = o.k);
```

预期观察：观察 Semi Join 计划和 rows。

源码回看：跟 `semifactors` 和 final cost 分支。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `add_paths_to_joinrel()`

先用注释和调用者确认它的职责：拿到一对 outerrel/innerrel 和 restrictlist 后，为同一个 joinrel 尝试多种 algorithm。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `match_unsorted_outer()`

先用注释和调用者确认它的职责：以未排序 outer 为中心枚举 Nested Loop 和 Hash Join 的常见组合。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `try_nestloop_path()`

先用注释和调用者确认它的职责：先计算 required_outer，拒绝不合理参数化，再调用 initial_cost_nestloop()。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `initial_cost_nestloop()`

先用注释和调用者确认它的职责：只估下层 path startup/run 和 inner rescan 下界，暂不展开昂贵 join qual。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `final_cost_nestloop()`

先用注释和调用者确认它的职责：补 SEMI/ANTI early stop、indexed join qual、restrict qual CPU 和 tlist cost。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `try_mergejoin_path()`

先用注释和调用者确认它的职责：检查 mergeclauses、排序需求和 required_outer，再用 precheck 控制候选数量。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `initial_cost_mergejoin()`

先用注释和调用者确认它的职责：把必要 sort / incremental sort 成本先算进去，并估两侧扫描比例。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `final_cost_mergejoin()`

先用注释和调用者确认它的职责：设置 skip_mark_restore / materialize_inner，再补 merge qual、qp qual 和输出 target 成本。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

join 算法的关键是先看算法前提，再看 rows 放大、rescan、sort/hash build 和 join qual CPU。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `outer_path->rows` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `inner_path->rows` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `hashclauses` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `required_outer` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | Hash Cond / Merge Cond 是否真实存在 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `add_paths_to_joinrel()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | outer rows 是否放大 inner rescan | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `match_unsorted_outer()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | inner path 是否 indexed join quals | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `try_nestloop_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | Sort 成本是否被并入 Merge Join | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `initial_cost_nestloop()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | Hash Join numbatches 是否过大 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `final_cost_nestloop()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | SEMI/ANTI 是否 early stop | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `try_mergejoin_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | required_outer 是否限制算法 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `initial_cost_mergejoin()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | disabled_nodes 是否来自 enable GUC | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `final_cost_mergejoin()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `Hash Cond / Merge Cond 是否真实存在` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `outer rows 是否放大 inner rescan` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `inner path 是否 indexed join quals` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `Sort 成本是否被并入 Merge Join` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `Hash Join numbatches 是否过大` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | planner 为什么会在 Nested Loop、Merge Join 与 Hash Join 之间切换；rows 估错时，为什么 join 类型错误常比单个 scan 错误更快放大成慢 SQL？ |
| 运行模型 | `joinpath.c` 枚举可行 join path；`try_nestloop_path()`、`try_mergejoin_path()`、`try_hashjoin_path()` 先做参数化与 legality 检查，再调用 `initial_cost_*` 得到可剪枝下界；通过 precheck 后，`pathnode.c` 创建具体 JoinPath，并由 `final_cost_*` 写入 rows、startup_cost、total_cost 和 node-specific 状态。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
