# PostgreSQL PathTarget 与投影成本

## 课程定位

前置知识：已经理解 Path 描述候选实现，并知道 Path 可以携带不同输出 target。

本节唯一主问题：

```text
为什么 optimizer 要把 targetlist 转成 PathTarget；表达式求值成本、平均宽度、sortgroupref 和 projection placement 如何影响 scan、join、aggregate、sort 与最终 Plan？
```

核心矛盾：投影越早可能减少后续 tuple width 和传输成本；投影越晚可能避免重复计算昂贵表达式，或保护 SRF、排序、分组语义。planner 必须在搜索阶段表达这个取舍，而 executor 只接受最终 Plan targetlist。

学完后应能解释为什么同一查询的 scan path、grouping input path、sort path 和 final path 可能携带不同 PathTarget。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节说明 Path 的 `pathtarget` 是候选实现的一部分，本节解释 target 自身为什么不是简单输出列列表。

本节不讲 createplan.c 里所有 targetlist 修正，只聚焦 planner 阶段投影位置和成本。

下一节 parameterized path 会继续讨论 Path 上另一个容易误读的字段：param_info。

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
`tlist.c` 把 TargetEntry 列表压缩成 PathTarget；`set_pathtarget_cost_width()` 估表达式 CPU 与平均 width；`planner.c` 在 scan/join、grouping、sort、final 和 SRF 阶段构造多个 target；`pathnode.c` 用 `create_projection_path()` 或 `apply_projection_to_path()` 把 target 变化接入 Path 成本；`createplan.c` 最终把选中 Path 的目标转成 Plan targetlist。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `PathTarget.exprs` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `sortgrouprefs` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `cost.startup` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `cost.per_tuple` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `width` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/nodes/pathnodes.h` | `PathTarget` 的 exprs、sortgrouprefs、cost、width、has_volatile_expr。 |
| 2 | `src/backend/optimizer/util/tlist.c` | `make_pathtarget_from_tlist()`、`make_tlist_from_pathtarget()`、`split_pathtarget_at_srfs()`。 |
| 3 | `src/include/optimizer/tlist.h` | `create_pathtarget` 宏连接 tlist 转换与 cost/width 计算。 |
| 4 | `src/backend/optimizer/path/costsize.c` | `set_pathtarget_cost_width()`、`cost_qual_eval_node()`、`get_expr_width()`。 |
| 5 | `src/backend/optimizer/util/pathnode.c` | `create_projection_path()`、`apply_projection_to_path()`、`create_set_projection_path()`。 |
| 6 | `src/backend/optimizer/plan/planner.c` | `make_group_input_target()`、`make_partial_grouping_target()`、`apply_scanjoin_target_to_paths()`、`adjust_paths_for_srfs()`。 |
| 7 | `src/backend/optimizer/plan/createplan.c` | `create_plan_recurse()` 与 tlist 精确性标志。 |
| 8 | `src/include/optimizer/cost.h` | `set_pathtarget_cost_width()` 原型。 |

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

### 4.1. 宽列提前携带

SELECT 大 text 列会抬高 Sort、Hash 和 Gather 成本，即使过滤条件不变。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. 昂贵表达式位置

昂贵函数若在 inner loop 下层重复计算，CPU 成本会被 loops 放大。


### 4.3. GROUP BY 输入 target

分组前需要 grouping key 和 aggregate 参数，不一定等于最终 SELECT list。


### 4.4. SRF 分层

targetlist 中非顶层 SRF 会被拆成多个 ProjectSet / projection 层。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `PathTarget.exprs` | 要向上输出的表达式列表 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `sortgrouprefs` | 排序/分组引用编号 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `cost.startup` | 表达式一次性成本 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `cost.per_tuple` | 每行表达式成本 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `width` | 输出平均宽度 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `has_volatile_expr` | volatile 状态缓存 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `Path.pathtarget` | 候选实际输出 target | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `scanjoin_target` | scan/join 层输出需求 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `grouping_target` | 分组阶段输出需求 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `sort_input_target` | 排序前需要的 target | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `final_target` | 用户可见输出 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `ProjectSetPath` | SRF 投影层 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |

### 5.1. `PathTarget.exprs`

语义：要向上输出的表达式列表。

来源：由 tlist 去掉 TargetEntry 包装得到。

消费：Path、RelOptInfo 和 projection path 都引用它。

偏差后果：表达式 identity 影响 SRF 拆分和重复列处理。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `sortgrouprefs`

语义：排序/分组引用编号。

来源：从 TargetEntry.ressortgroupref 转入数组。

消费：GROUP BY、ORDER BY、DISTINCT 阶段靠它保持语义。

偏差后果：不是每个 expr 都有非零 ref。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `cost.startup`

语义：表达式一次性成本。

来源：`set_pathtarget_cost_width()` 计算。

消费：projection path 和 cost 函数加到 startup。

偏差后果：函数初始化或子表达式 startup 会体现这里。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `cost.per_tuple`

语义：每行表达式成本。

来源：来自 `cost_qual_eval_node()`。

消费：乘以 Path rows 后进入 total。

偏差后果：loops 放大时它比一次性 startup 更危险。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `width`

语义：输出平均宽度。

来源：由 `get_expr_width()` 和类型宽度估算。

消费：sort/hash/materialize/Gather 都会消费。

偏差后果：宽度是 planner 平均值，不是每行真实长度。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `has_volatile_expr`

语义：volatile 状态缓存。

来源：初始 unknown，第一次检查后缓存。

消费：限制投影移动，避免改变语义。

偏差后果：不能为了省成本随意提前/延后 volatile 表达式。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `Path.pathtarget`

语义：候选实际输出 target。

来源：可能不同于 parent->reltarget。

消费：createplan 根据它生成 Plan tlist。

偏差后果：同一 parent 可有多个 target 变体。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `scanjoin_target`

语义：scan/join 层输出需求。

来源：`grouping_planner()` 根据上层需求构造。

消费：下推到 base/join paths，减少不必要表达式。

偏差后果：它常比 final_target 更窄。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `grouping_target`

语义：分组阶段输出需求。

来源：包括 group key 和 aggregate 所需输入。

消费：GroupAggregate / HashAggregate 路径使用。

偏差后果：不是最终显示列。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `sort_input_target`

语义：排序前需要的 target。

来源：ORDER BY 表达式即使不输出，也可能必须保留。

消费：Sort 或 Incremental Sort 读取它。

偏差后果：丢掉会破坏排序语义。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `final_target`

语义：用户可见输出。

来源：Query targetlist 的最终形态。

消费：FINAL upper rel 和 createplan 消费。

偏差后果：resjunk 列不等于最终可见列。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.12. `ProjectSetPath`

语义：SRF 投影层。

来源：`create_set_projection_path()` 生成。

消费：保证 SRF 只在 executor 支持的位置求值。

偏差后果：这是语义约束，不是纯成本优化。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
make_pathtarget_from_tlist()
  -> set_pathtarget_cost_width()
  -> make_group_input_target()
  -> make_partial_grouping_target()
  -> apply_scanjoin_target_to_paths()
  -> create_projection_path()
  -> apply_projection_to_path()
  -> split_pathtarget_at_srfs()
  -> adjust_paths_for_srfs()
  -> create_plan_recurse()
```

### 6.1. `make_pathtarget_from_tlist()`

把 TargetEntry 列表转成 exprs 和 sortgrouprefs，先不计算 cost/width。

观察锚点：`PathTarget.exprs`。

### 6.2. `set_pathtarget_cost_width()`

遍历 exprs 估 CPU 与 width，写入 PathTarget。

观察锚点：`sortgrouprefs`。

### 6.3. `make_group_input_target()`

为 GROUP BY 输入保留 grouping key、aggregate 参数、HAVING 和 resjunk 需要的 Var。

观察锚点：`cost.startup`。

### 6.4. `make_partial_grouping_target()`

为 partial aggregate 输出 partial Aggref 和必要 Vars。

观察锚点：`cost.per_tuple`。

### 6.5. `apply_scanjoin_target_to_paths()`

把 scan/join relation 的 pathlist 调整到新的 scanjoin target。

观察锚点：`width`。

### 6.6. `create_projection_path()`

无法原地修改或需要明确投影层时，包一层 ProjectionPath。

观察锚点：`has_volatile_expr`。

### 6.7. `apply_projection_to_path()`

在安全时原地修改 path target；不安全或多引用时退回 projection path。

观察锚点：`Path.pathtarget`。

### 6.8. `split_pathtarget_at_srfs()`

把含 SRF 的 target 拆成 executor 能接受的层次。

观察锚点：`scanjoin_target`。

### 6.9. `adjust_paths_for_srfs()`

在 upper rel 的每条 path 上叠加 ProjectSet 或 projection。

观察锚点：`grouping_target`。

### 6.10. `create_plan_recurse()`

最终按选中 Path 和 flags 构造精确或可调整的 Plan tlist。

观察锚点：`sort_input_target`。

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

### 9.1. width 缺少精确统计

表达式宽度常靠类型默认值或启发式估算。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. volatile 限制移动

含 volatile function 的 target 不能随意重排求值位置。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. SRF 必须拆层

executor 只支持顶层 SRF 的 ProjectSet 形态。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. projection 原地修改受限

当 path 可能被多处引用时，`apply_projection_to_path()` 会创建新 ProjectionPath。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. createplan 重新校正 tlist

某些节点最终 targetlist 可能比 planner projection cost 略有差异。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. resjunk 影响中间 target

ORDER BY / GROUP BY 需要的列可能不出现在最终输出。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

PathTarget 的关键是投影位置同时影响语义、CPU 和 tuple width。

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
| exprs 是否包含 resjunk 语义需要 | `make_pathtarget_from_tlist()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| sortgrouprefs 是否和 group/order 对齐 | `set_pathtarget_cost_width()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| cost.per_tuple 是否被 rows 放大 | `make_group_input_target()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| width 是否支配内存节点 | `make_partial_grouping_target()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| has_volatile_expr 是否限制移动 | `apply_scanjoin_target_to_paths()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| scanjoin_target 是否比 final_target 窄 | `create_projection_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| ProjectionPath 是否新增 | `apply_projection_to_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| ProjectSet 是否因 SRF 出现 | `split_pathtarget_at_srfs()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `exprs 是否包含 resjunk 语义需要` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `sortgrouprefs 是否和 group/order 对齐` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `cost.per_tuple 是否被 rows 放大` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `width 是否支配内存节点` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `has_volatile_expr 是否限制移动` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `scanjoin_target 是否比 final_target 窄` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `ProjectionPath 是否新增` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `ProjectSet 是否因 SRF 出现` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 把 PathTarget 当最终 SELECT list。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 忽略 width 对 Sort/Hash/Gather 的连锁影响。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 只看表达式是否输出，不看它是否为 ORDER BY/GROUP BY 所需。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 把 volatile 表达式当普通 CPU 成本移动。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 忘记 SRF 是语义约束，不是简单函数调用。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. 宽 target 成本

SQL：

```sql
CREATE TABLE t_target AS SELECT g AS id, md5(g::text) AS v, repeat(md5(g::text), 20) AS payload FROM generate_series(1,400000) g;
ANALYZE t_target;
EXPLAIN SELECT id FROM t_target ORDER BY v;
EXPLAIN SELECT id, payload FROM t_target ORDER BY v;
```

预期观察：比较 Sort cost 和 width。

源码回看：跟 `set_pathtarget_cost_width()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. ORDER BY resjunk

SQL：

```sql
EXPLAIN SELECT id FROM t_target ORDER BY lower(v);
```

预期观察：排序表达式不输出但必须进入 sort input target。

源码回看：跟 `sort_input_target`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. GROUP BY 输入 target

SQL：

```sql
EXPLAIN SELECT id % 10, count(length(payload)) FROM t_target GROUP BY id % 10;
```

预期观察：group input target 需要 aggregate 参数，不等于 final target。

源码回看：跟 `make_group_input_target()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. SRF 拆层

SQL：

```sql
EXPLAIN SELECT generate_series(1,2), id FROM t_target LIMIT 5;
```

预期观察：观察 ProjectSet 或投影层。

源码回看：跟 `split_pathtarget_at_srfs()` 和 `adjust_paths_for_srfs()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `make_pathtarget_from_tlist()`

先用注释和调用者确认它的职责：把 TargetEntry 列表转成 exprs 和 sortgrouprefs，先不计算 cost/width。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `set_pathtarget_cost_width()`

先用注释和调用者确认它的职责：遍历 exprs 估 CPU 与 width，写入 PathTarget。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `make_group_input_target()`

先用注释和调用者确认它的职责：为 GROUP BY 输入保留 grouping key、aggregate 参数、HAVING 和 resjunk 需要的 Var。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `make_partial_grouping_target()`

先用注释和调用者确认它的职责：为 partial aggregate 输出 partial Aggref 和必要 Vars。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `apply_scanjoin_target_to_paths()`

先用注释和调用者确认它的职责：把 scan/join relation 的 pathlist 调整到新的 scanjoin target。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `create_projection_path()`

先用注释和调用者确认它的职责：无法原地修改或需要明确投影层时，包一层 ProjectionPath。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `apply_projection_to_path()`

先用注释和调用者确认它的职责：在安全时原地修改 path target；不安全或多引用时退回 projection path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `split_pathtarget_at_srfs()`

先用注释和调用者确认它的职责：把含 SRF 的 target 拆成 executor 能接受的层次。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

PathTarget 的关键是投影位置同时影响语义、CPU 和 tuple width。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `PathTarget.width` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `cost.per_tuple` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `sortgrouprefs` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `has_volatile_expr` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | exprs 是否包含 resjunk 语义需要 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `make_pathtarget_from_tlist()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | sortgrouprefs 是否和 group/order 对齐 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `set_pathtarget_cost_width()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | cost.per_tuple 是否被 rows 放大 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `make_group_input_target()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | width 是否支配内存节点 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `make_partial_grouping_target()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | has_volatile_expr 是否限制移动 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `apply_scanjoin_target_to_paths()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | scanjoin_target 是否比 final_target 窄 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_projection_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | ProjectionPath 是否新增 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `apply_projection_to_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | ProjectSet 是否因 SRF 出现 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `split_pathtarget_at_srfs()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `exprs 是否包含 resjunk 语义需要` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `sortgrouprefs 是否和 group/order 对齐` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `cost.per_tuple 是否被 rows 放大` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `width 是否支配内存节点` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `has_volatile_expr 是否限制移动` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | 为什么 optimizer 要把 targetlist 转成 PathTarget；表达式求值成本、平均宽度、sortgroupref 和 projection placement 如何影响 scan、join、aggregate、sort 与最终 Plan？ |
| 运行模型 | `tlist.c` 把 TargetEntry 列表压缩成 PathTarget；`set_pathtarget_cost_width()` 估表达式 CPU 与平均 width；`planner.c` 在 scan/join、grouping、sort、final 和 SRF 阶段构造多个 target；`pathnode.c` 用 `create_projection_path()` 或 `apply_projection_to_path()` 把 target 变化接入 Path 成本；`createplan.c` 最终把选中 Path 的目标转成 Plan targetlist。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
