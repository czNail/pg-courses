# PostgreSQL Path 作为候选实现方式

## 课程定位

前置知识：已经理解 RelOptInfo 是搜索空间节点，并知道一个 RelOptInfo 可以挂多个候选。

本节唯一主问题：

```text
同一个 RelOptInfo 为什么需要保留多个 Path；`startup_cost`、`total_cost`、`rows`、`pathkeys`、`param_info`、parallel 字段如何共同描述一种可比较的实现方式？
```

核心矛盾：planner 想尽早剪掉无意义候选控制搜索空间；但如果只保留 total_cost 最低的实现，会丢失快速启动、有序输出、参数化内层、parallel 变体和上层复用机会。

学完后应能读懂一次 `add_path()` 调试现场：为什么较贵 Path 仍保留，为什么较便宜 Path 会被丢弃，以及为什么 parameterized path 的 pathkeys 被有意弱化。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释 RelOptInfo 是候选容器，本节聚焦候选本身。

本节不逐一讲完每个派生 Path 结构，只围绕 Path 基类字段和 add_path 策略展开。

下一节 PathTarget 会解释 `pathtarget` 为什么也是候选实现的一部分。

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
`create_*_path()` 构造某种实现方式并写入 pathtype、parent、pathtarget、param_info、parallel、rows、startup_cost、total_cost、pathkeys；`add_path_precheck()` 用低成本信息提前拒绝明显劣势候选；`add_path()` 用 fuzzy cost、pathkeys、required_outer、rows、parallel_safe 和 disabled_nodes 决定替换、共存或回收；`set_cheapest()` 再把 surviving pathlist 压缩成后续阶段常用指针。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `pathtype` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `parent` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `pathtarget` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `param_info` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `parallel_aware` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/util/pathnode.c` | `add_path()`、`add_path_precheck()`、`add_partial_path()`、`compare_path_costs_fuzzily()`、`set_cheapest()` 和 `create_*_path()` 家族。 |
| 2 | `src/backend/optimizer/path/allpaths.c` | `set_base_rel_pathlists()`、`generate_gather_paths()` 把 base/parallel path 放入 RelOptInfo。 |
| 3 | `src/backend/optimizer/path/joinpath.c` | `try_nestloop_path()`、`try_mergejoin_path()`、`try_hashjoin_path()` 生成 join path。 |
| 4 | `src/backend/optimizer/path/costsize.c` | cost 函数写入 rows、startup_cost、total_cost 和 node-specific 字段。 |
| 5 | `src/backend/optimizer/path/pathkeys.c` | PathKeys 比较和排序有用性判断。 |
| 6 | `src/backend/optimizer/plan/createplan.c` | `create_plan()` 只消费最终选中的 Path。 |
| 7 | `src/include/nodes/pathnodes.h` | Path 基类、IndexPath、JoinPath、ProjectionPath、GatherPath 等派生结构。 |
| 8 | `src/include/optimizer/pathnode.h` | Path 创建、剪枝和 parameterization 接口声明。 |

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

### 4.1. 有序 IndexPath 被保留

一个过滤成本较高的 Index Scan 可能因 pathkeys 避免 Sort 而留在 pathlist。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. LIMIT 关注 startup

`LIMIT 1`、EXISTS 或 cursor 场景让 startup_cost 低的候选有价值。


### 4.3. parameterized path 不能随意换边

内层 index path 依赖 outer rel 时，只能在满足 required_outer 的 join 顺序中使用。


### 4.4. partial path 需要 Gather

parallel partial path 不是完整结果，必须转成 Gather / Gather Merge 后才能与普通 path 比较。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `pathtype` | 未来 Plan 节点类型 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `parent` | 所属 RelOptInfo | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `pathtarget` | 该 Path 输出表达式 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `param_info` | 参数化信息 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `parallel_aware` | 并行协同标志 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `parallel_safe` | 能否放进 parallel plan | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `parallel_workers` | 计划 worker 数 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `rows` | Path 产出行数 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `disabled_nodes` | 被禁用节点数 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `startup_cost` | 第一行前成本 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `total_cost` | 全量输出成本 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |
| `pathkeys` | 输出排序 | `src/backend/optimizer/util/pathnode.c` 等主线文件消费或写入。 |

### 5.1. `pathtype`

语义：未来 Plan 节点类型。

来源：例如 T_SeqScan、T_IndexScan、T_HashJoin。

消费：createplan 根据它选择构造函数。

偏差后果：它不是 Path 对象的 NodeTag，而是执行节点契约提示。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `parent`

语义：所属 RelOptInfo。

来源：说明这个 Path 实现哪个搜索节点。

消费：`add_path()` 把它放入 parent->pathlist。

偏差后果：不能跨 RelOptInfo 复用 Path。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `pathtarget`

语义：该 Path 输出表达式。

来源：可能等于 parent->reltarget，也可能是 ProjectionPath 的新 target。

消费：target cost/width 会进入上层 sort/hash/parallel。

偏差后果：它让“同一个 rel”存在不同输出形态。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `param_info`

语义：参数化信息。

来源：非空表示依赖外层 rels。

消费：`PATH_REQ_OUTER()` 统一读取 required_outer。

偏差后果：它决定 path 不能独立执行。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `parallel_aware`

语义：并行协同标志。

来源：表示节点本身知道 worker 如何分割输入。

消费：parallel scan 类路径使用。

偏差后果：不等同于 parallel_safe。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `parallel_safe`

语义：能否放进 parallel plan。

来源：由表达式、子 path 和 rel 属性推导。

消费：add_path 同成本时偏好 safer path。

偏差后果：unsafe path 可完整执行，但不能进入 worker。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `parallel_workers`

语义：计划 worker 数。

来源：cost 函数可按 divisor 缩 rows/CPU。

消费：createplan 把它写进并行 Plan。

偏差后果：实际 worker 数可能不同。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `rows`

语义：Path 产出行数。

来源：普通 path 多来自 parent->rows，参数化 path 可来自 ppi_rows。

消费：成本、add_path rows 比较和上层 cardinality 都读它。

偏差后果：同一 parent 下不同 path rows 不一定相同。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `disabled_nodes`

语义：被禁用节点数。

来源：enable GUC 通过它比 cost 更高优先级地影响比较。

消费：pathlist 按 disabled_nodes 再 total_cost 排序。

偏差后果：禁用不是删除。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `startup_cost`

语义：第一行前成本。

来源：Sort/Hash build 等会提高它。

消费：LIMIT / EXISTS 会关注它。

偏差后果：低 startup 高 total 的路径可能长期保留。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `total_cost`

语义：全量输出成本。

来源：普通全量消费的主比较维度。

消费：add_path fuzzy comparison 使用它。

偏差后果：它不能单独决定 path 支配关系。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.12. `pathkeys`

语义：输出排序。

来源：有序 index、merge append、sort 等路径携带。

消费：上层 ORDER BY、Merge Join、GroupAggregate 可复用。

偏差后果：parameterized path 在 add_path 比较中被当作没有 pathkeys。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
create_seqscan_path()
  -> create_index_path()
  -> create_nestloop_path()
  -> create_projection_path()
  -> create_gather_path()
  -> add_path_precheck()
  -> add_path()
  -> add_partial_path()
  -> set_cheapest()
  -> create_plan()
```

### 6.1. `create_seqscan_path()`

最简单 Path 也要写 parent、pathtarget、param_info、parallel 字段并调用 `cost_seqscan()`。

观察锚点：`pathtype`。

### 6.2. `create_index_path()`

在 Path 基类外保存 indexclauses、orderbys、scan direction、indexinfo 和 indextotalcost。

观察锚点：`parent`。

### 6.3. `create_nestloop_path()`

把 outer/inner path 连接成 JoinPath，计算 required_outer，并交给 final_cost_nestloop()。

观察锚点：`pathtarget`。

### 6.4. `create_projection_path()`

当 target 改变时包装 subpath，调整成本、并行安全性和输出契约。

观察锚点：`param_info`。

### 6.5. `create_gather_path()`

把 partial path 转成完整 Path，添加 parallel setup/tuple transfer 成本。

观察锚点：`parallel_aware`。

### 6.6. `add_path_precheck()`

用 pathkeys、required_outer、startup/total 下界快速拒绝明显不可能存活的候选。

观察锚点：`parallel_safe`。

### 6.7. `add_path()`

逐个与旧 path 比较，可能插入新 path、删除旧 path，或回收新 path。

观察锚点：`parallel_workers`。

### 6.8. `add_partial_path()`

在 partial_pathlist 内比较，不考虑 parameterized partial path。

观察锚点：`rows`。

### 6.9. `set_cheapest()`

扫描 surviving pathlist，设置 cheapest_startup_path、cheapest_total_path、cheapest_parameterized_paths。

观察锚点：`disabled_nodes`。

### 6.10. `create_plan()`

只消费最终 best_path；Path 的大部分搜索状态不会进入 executor。

观察锚点：`startup_cost`。

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

### 9.1. fuzzy cost 防抖

极小浮点差异不会导致平台间频繁换 plan。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. parameterized pathkeys 弱化

add_path 把 parameterized path 当作 NIL pathkeys，以控制数量。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. IndexPath 不立即 pfree

被 BitmapHeapPath 引用的 IndexPath 即使被 pathlist 拒绝也不能随便释放。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. startup 是否有价值取决于 parent

`consider_startup` / `consider_param_startup` 关闭时，低 startup 候选更容易被丢弃。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. partial path 独立剪枝

partial_pathlist 不能立刻生成 Gather 引用，否则被删除会悬空。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. disabled_nodes 优先于 cost

enable GUC 造成的 disabled count 是更高阶成本。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

Path 候选的关键是成本、排序、参数化、并行和 disabled_nodes 的共同支配关系。

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
| new_path 的 parent 是否正确 | `create_seqscan_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| startup 与 total 是否一方各优 | `create_index_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| pathkeys 是否不同 | `create_nestloop_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| required_outer 是否相同或包含 | `create_projection_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| rows 是否不同 | `create_gather_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| parallel_safe 是否影响支配 | `add_path_precheck()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| disabled_nodes 是否先于 cost | `add_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| set_cheapest 是否已经运行 | `add_partial_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `new_path 的 parent 是否正确` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `startup 与 total 是否一方各优` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `pathkeys 是否不同` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `required_outer 是否相同或包含` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `rows 是否不同` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `parallel_safe 是否影响支配` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `disabled_nodes 是否先于 cost` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `set_cheapest 是否已经运行` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 只看 total_cost 判断 Path 存亡。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 忘记 parameterized path 不能独立执行。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 把 partial path 当完整候选解释。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 认为 enable GUC 会移除节点。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 在 executor 阶段寻找已被淘汰的 Path。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. 观察 add_path 共存

SQL：

```sql
在 `add_path()` 断点打印 new/old 的 startup、total、pathkeys、PATH_REQ_OUTER。
构造 ORDER BY 查询和普通过滤查询对比。
```

预期观察：看到 cost 不是唯一维度。

源码回看：入口 `src/backend/optimizer/util/pathnode.c`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. LIMIT startup

SQL：

```sql
CREATE TABLE t_path AS SELECT g, g AS k, md5(g::text) AS v FROM generate_series(1,500000) g;
CREATE INDEX ON t_path(k);
ANALYZE t_path;
EXPLAIN SELECT * FROM t_path WHERE k > 0 LIMIT 1;
EXPLAIN SELECT * FROM t_path WHERE k > 0;
```

预期观察：比较 startup 低的路径是否更有吸引力。

源码回看：跟 `set_cheapest()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. pathkeys 价值

SQL：

```sql
EXPLAIN SELECT * FROM t_path ORDER BY k LIMIT 100;
EXPLAIN SELECT * FROM t_path ORDER BY v LIMIT 100;
```

预期观察：一个可复用 index pathkeys，一个需要 Sort。

源码回看：跟 `pathkeys.c` 和 `add_path()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. parameterized path

SQL：

```sql
CREATE TABLE t_outer AS SELECT g FROM generate_series(1,100) g;
EXPLAIN SELECT * FROM t_outer o CROSS JOIN LATERAL (SELECT * FROM t_path p WHERE p.k = o.g) s;
```

预期观察：观察 inner path 的 required_outer。

源码回看：跟 `get_baserel_parampathinfo()` 和 `PATH_REQ_OUTER()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `create_seqscan_path()`

先用注释和调用者确认它的职责：最简单 Path 也要写 parent、pathtarget、param_info、parallel 字段并调用 `cost_seqscan()`。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `create_index_path()`

先用注释和调用者确认它的职责：在 Path 基类外保存 indexclauses、orderbys、scan direction、indexinfo 和 indextotalcost。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `create_nestloop_path()`

先用注释和调用者确认它的职责：把 outer/inner path 连接成 JoinPath，计算 required_outer，并交给 final_cost_nestloop()。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `create_projection_path()`

先用注释和调用者确认它的职责：当 target 改变时包装 subpath，调整成本、并行安全性和输出契约。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `create_gather_path()`

先用注释和调用者确认它的职责：把 partial path 转成完整 Path，添加 parallel setup/tuple transfer 成本。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `add_path_precheck()`

先用注释和调用者确认它的职责：用 pathkeys、required_outer、startup/total 下界快速拒绝明显不可能存活的候选。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `add_path()`

先用注释和调用者确认它的职责：逐个与旧 path 比较，可能插入新 path、删除旧 path，或回收新 path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `add_partial_path()`

先用注释和调用者确认它的职责：在 partial_pathlist 内比较，不考虑 parameterized partial path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

Path 候选的关键是成本、排序、参数化、并行和 disabled_nodes 的共同支配关系。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `startup_cost` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `total_cost` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `pathkeys` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `param_info` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | new_path 的 parent 是否正确 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_seqscan_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | startup 与 total 是否一方各优 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_index_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | pathkeys 是否不同 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_nestloop_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | required_outer 是否相同或包含 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_projection_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | rows 是否不同 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_gather_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | parallel_safe 是否影响支配 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `add_path_precheck()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | disabled_nodes 是否先于 cost | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `add_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | set_cheapest 是否已经运行 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `add_partial_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `new_path 的 parent 是否正确` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `startup 与 total 是否一方各优` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `pathkeys 是否不同` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `required_outer 是否相同或包含` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `rows 是否不同` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | 同一个 RelOptInfo 为什么需要保留多个 Path；`startup_cost`、`total_cost`、`rows`、`pathkeys`、`param_info`、parallel 字段如何共同描述一种可比较的实现方式？ |
| 运行模型 | `create_*_path()` 构造某种实现方式并写入 pathtype、parent、pathtarget、param_info、parallel、rows、startup_cost、total_cost、pathkeys；`add_path_precheck()` 用低成本信息提前拒绝明显劣势候选；`add_path()` 用 fuzzy cost、pathkeys、required_outer、rows、parallel_safe 和 disabled_nodes 决定替换、共存或回收；`set_cheapest()` 再把 surviving pathlist 压缩成后续阶段常用指针。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
