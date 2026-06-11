# PostgreSQL Sort、Hash、Material 与内存溢出成本

## 课程定位

前置知识：已经理解 scan path 如何把页数、rows 和 tuple CPU 写成 Path 成本。

本节唯一主问题：

```text
为什么同一条 SQL 在 `work_mem` 或 `hash_mem_multiplier` 附近，Sort、HashAggregate、Hash Join 与 Materialize 的成本会出现台阶，甚至切换到另一类执行策略？
```

核心矛盾：planner 要在执行前估算中间状态能否放进内存；真实内存占用却取决于 tuple width、分组数、hash entry 布局、批次数、数据倾斜和 executor 的 spill 决策。

学完后应能把一次 Sort Method 变成 external merge、HashAgg 分批或 Hash Join Batches 增多的现象，回连到 planner 侧 rows、width、work_mem、hash table size 与 rescan 计价。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节看扫描如何产生输入行，本节看 blocking 或半 blocking 节点如何保存、重排或重复读取这些行。

本节只讨论 planner 侧内存和 spill 成本，不展开 tuplesort、tuplestore、hash table executor 的完整实现。

本节结束后再进入 join algorithm cost，因为 join cost 会复用 sort、hash 和 materialize 的判断。

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
`costsize.c` 把 rows 与 width 先换成字节和页；`cost_tuplesort()` 判断内存排序、外部归并和 bounded heap sort；`cost_agg()` 与 `estimate_hashagg_tablesize()` 估 HashAgg 内存；`initial_cost_hashjoin()` 调 executor 同源的 `ExecChooseHashTableSize()` 估 hash join batches；`cost_rescan()` 和 `cost_material()` 描述重复读取时的内存/临时文件边界。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `work_mem` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `hash_mem_multiplier` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `tuples` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `width` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `limit_tuples` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/costsize.c` | `cost_tuplesort()`、`cost_sort()`、`cost_incremental_sort()`、`cost_material()`、`cost_rescan()`、`cost_agg()`、`initial_cost_hashjoin()`。 |
| 2 | `src/backend/optimizer/util/pathnode.c` | `create_sort_path()`、`create_incremental_sort_path()`、`create_material_path()`、`create_agg_path()`、`create_hashjoin_path()` 写入 Path。 |
| 3 | `src/backend/utils/adt/selfuncs.c` | `estimate_hashagg_tablesize()` 估 HashAggregate hash table 尺寸。 |
| 4 | `src/backend/executor/nodeAgg.c` | `hash_agg_set_limits()` 是 HashAgg 执行期 spill 边界的对照。 |
| 5 | `src/backend/executor/nodeHash.c` | `ExecChooseHashTableSize()` 和 `get_hash_memory_limit()` 给 Hash Join planner 成本提供 executor 同源模型。 |
| 6 | `src/backend/utils/misc/guc_tables.c` | `work_mem`、`hash_mem_multiplier` 的 GUC 定义和单位来源。 |
| 7 | `src/include/nodes/pathnodes.h` | `SortPath`、`MaterialPath`、`AggPath`、`HashPath` 共享 Path 成本字段。 |
| 8 | `src/include/executor/nodeAgg.h` | `hash_agg_set_limits()` 声明。 |

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

### 4.1. work_mem 台阶

同一 ORDER BY 在低 work_mem 下估算外部排序 I/O，提高后 cost curve 会回到纯 CPU 比较。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. bounded sort

LIMIT 较小时 `cost_tuplesort()` 可走 heap 方法，startup/total 形状和全量排序不同。


### 4.3. HashAgg 与 GroupAggregate 切换

分组数和 transitionSpace 让 HashAgg 内存超过阈值时，排序分组可能更有吸引力。


### 4.4. Hash Join Batches

inner 侧太宽或太多时，`numbatches > 1` 会把临时文件读写计入 startup 和 run cost。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `work_mem` | 普通排序与 materialize 的内存上限 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `hash_mem_multiplier` | hash 类节点额外内存系数 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `tuples` | 输入行数 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `width` | 平均 tuple 宽度 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `limit_tuples` | 上层 LIMIT 边界 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `numGroups` | 分组数估算 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `transitionSpace` | 聚合状态额外空间 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `hashentrysize` | HashAgg entry 尺寸 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `numbatches` | Hash Join 批次数 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `materialize_inner` | Merge Join 内层 materialize 决策 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `cost_rescan()` | 重复读取成本模型 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |

### 5.1. `work_mem`

语义：普通排序与 materialize 的内存上限。

来源：以 kB GUC 进入 `cost_sort()` / `cost_material()`，在函数里换成字节。

消费：控制 in-memory sort、external sort 和 materialize 溢出估计。

偏差后果：它不是单查询全局内存上限，而是多个节点可能各自使用。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `hash_mem_multiplier`

语义：hash 类节点额外内存系数。

来源：由 `get_hash_memory_limit()` 合并 `work_mem` 得出。

消费：HashAgg、Hash Join、Memoize 等 hash 状态会读取这个 limit。

偏差后果：调高它只影响 hash 类节点，不会让 Sort 直接获得同样内存。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `tuples`

语义：输入行数。

来源：来自下层 Path rows 或 group estimate。

消费：排序比较次数、hash entry 数、materialize 存储量都随它放大。

偏差后果：rows 低估会让 spill 风险在 planner 中消失。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `width`

语义：平均 tuple 宽度。

来源：来自 PathTarget 或 relation 宽度估计。

消费：`relation_byte_size()` 和 `page_size()` 用它判断内存是否越界。

偏差后果：宽 text/json 列被提前投影时，sort/hash 成本会被明显抬高。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `limit_tuples`

语义：上层 LIMIT 边界。

来源：由 sort path 创建者传入。

消费：`cost_tuplesort()` 用它判断 bounded heap sort。

偏差后果：LIMIT 只影响需要多少输出，不等于下层一定少读输入。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `numGroups`

语义：分组数估算。

来源：来自 group cardinality 估计。

消费：`cost_agg()` 与 `estimate_hashagg_tablesize()` 判断 HashAgg 内存。

偏差后果：多列相关性缺失会让它大幅偏离。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `transitionSpace`

语义：聚合状态额外空间。

来源：聚合函数元数据提供。

消费：HashAgg entry size 不只等于输出 tuple width。

偏差后果：自定义 aggregate 状态大时，普通 width 直觉会误导。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `hashentrysize`

语义：HashAgg entry 尺寸。

来源：`cost_agg()` 合并 tuple、transition 和 per-entry overhead。

消费：传给 `hash_agg_set_limits()` 估分区边界。

偏差后果：它体现 executor 结构开销，不是 SQL 输出列宽。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `numbatches`

语义：Hash Join 批次数。

来源：`ExecChooseHashTableSize()` 返回。

消费：`initial_cost_hashjoin()` 用它决定是否计临时批次 I/O。

偏差后果：EXPLAIN ANALYZE 的 Batches 偏离时，通常回看 rows/width/skew。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `materialize_inner`

语义：Merge Join 内层 materialize 决策。

来源：`final_cost_mergejoin()` 根据 mark/restore、sort spill 和 GUC 判断。

消费：决定内层 rescan 是直接重扫还是 Materialize。

偏差后果：它可能是正确性要求，也可能只是性能优化。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `cost_rescan()`

语义：重复读取成本模型。

来源：按 Path 类型估第二次扫描的 startup / total。

消费：Nested Loop 和 Materialize 场景会消费它。

偏差后果：它解释了同一个 inner path 在 join 中为何比单独执行更贵。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
cost_sort()
  -> cost_tuplesort()
  -> cost_incremental_sort()
  -> cost_material()
  -> cost_rescan()
  -> cost_agg()
  -> estimate_hashagg_tablesize()
  -> hash_agg_set_limits()
  -> initial_cost_hashjoin()
  -> final_cost_hashjoin()
```

### 6.1. `cost_sort()`

包装 `cost_tuplesort()`，把输入 startup/total 与排序自身成本合并到 SortPath。

观察锚点：`work_mem`。

### 6.2. `cost_tuplesort()`

根据 `input_bytes`、`sort_mem_bytes` 和 `limit_tuples` 选择 quicksort、external merge 或 bounded heap 估算。

观察锚点：`hash_mem_multiplier`。

### 6.3. `cost_incremental_sort()`

先估 presorted key 形成的 group 数，再按单组 sort 成本累计总成本。

观察锚点：`tuples`。

### 6.4. `cost_material()`

按输入 rows/width 判断 Materialize 是否可能溢出，并把写临时文件风险计入 total。

观察锚点：`width`。

### 6.5. `cost_rescan()`

为 repeated scan 提供第二次读取成本；Materialize、Sort、Memoize 等路径分支不同。

观察锚点：`limit_tuples`。

### 6.6. `cost_agg()`

区分 plain/sorted/hash/mixed aggregate，把 per-tuple trans cost、final cost、group 数和内存风险合并。

观察锚点：`numGroups`。

### 6.7. `estimate_hashagg_tablesize()`

把 group 数、entry width 和 transition 空间换成 HashAgg 预计内存。

观察锚点：`transitionSpace`。

### 6.8. `hash_agg_set_limits()`

executor 侧计算 mem_limit、ngroups_limit 和 partition limit；planner 用它校准 HashAgg spill 边界。

观察锚点：`hashentrysize`。

### 6.9. `initial_cost_hashjoin()`

构造 hash table 前先用 `ExecChooseHashTableSize()` 估 buckets/batches。

观察锚点：`numbatches`。

### 6.10. `final_cost_hashjoin()`

继续计算 bucket size、MCV、hash qual CPU 和输出 tuple CPU。

观察锚点：`materialize_inner`。

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

### 9.1. rows 或 width 估错

planner 仍会产出连续 cost 数字，但 Sort Method 或 Hash Batches 会在执行期暴露偏差。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. 外部排序 I/O 近似

`cost_tuplesort()` 假设磁盘访问有固定随机/顺序比例，不读取当前临时表空间状态。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. bounded sort 不是少扫描

LIMIT 可降低排序结构规模，但下层 path 仍可能要提供全部候选输入。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. HashAgg spill 版本差异

planner 和 executor 共享部分内存模型，但执行期批次还受实际分布和状态大小影响。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. Materialize 可被 GUC 影响

关闭 materialize 会改变部分性能优化路径，但某些正确性需要的 materialize 仍可能存在。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. Hash Join skew 难精确

`ExecChooseHashTableSize()` 考虑 skew 预留，但成本没有完整模拟每个 bucket 的真实分布。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

内存节点的关键是把 rows/width 换成 bytes，再判断是否跨过 work_mem 或 hash memory limit。

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
| input_bytes 是否越过 work_mem | `cost_sort()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| LIMIT 是否触发 bounded sort | `cost_tuplesort()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| incremental sort 是否有 presorted keys | `cost_incremental_sort()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| HashAgg entry size 是否包含 transitionSpace | `cost_material()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| hash_mem_multiplier 是否影响 hash limit | `cost_rescan()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Hash Join numbatches 是否大于 1 | `cost_agg()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Materialize 是正确性还是性能优化 | `estimate_hashagg_tablesize()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| 执行期 Disk/Batches 是否回写到 rows/width 偏差 | `hash_agg_set_limits()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `input_bytes 是否越过 work_mem` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `LIMIT 是否触发 bounded sort` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `incremental sort 是否有 presorted keys` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `HashAgg entry size 是否包含 transitionSpace` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `hash_mem_multiplier 是否影响 hash limit` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Hash Join numbatches 是否大于 1` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Materialize 是正确性还是性能优化` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `执行期 Disk/Batches 是否回写到 rows/width 偏差` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 把 work_mem 当成整个查询只能用一次；多个 executor 节点可以分别消耗。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 只看 Sort 节点，不看输入 Path 的 rows/width 是否已经偏离。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 认为 HashAggregate 一定避免排序所以总是更快；spill 和分组数会改变判断。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 忽略 Materialize 的 rescan 角色；它常出现在上层 join 需要重复读取内层时。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 用 EXPLAIN cost 预测临时文件精确大小；planner 的磁盘 I/O 只是比较模型。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. work_mem 与 Sort Method

SQL：

```sql
CREATE TABLE t_sort AS SELECT g, md5(g::text) AS payload FROM generate_series(1,700000) g;
ANALYZE t_sort;
SET work_mem = 1MB;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_sort ORDER BY payload;
SET work_mem = 128MB;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_sort ORDER BY payload;
```

预期观察：看 Sort Method、Disk、estimated cost 是否同向变化。

源码回看：断点看 `cost_tuplesort()` 的 `input_bytes` 与 `sort_mem_bytes`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. LIMIT bounded sort

SQL：

```sql
SET work_mem = 4MB;
EXPLAIN SELECT * FROM t_sort ORDER BY payload LIMIT 20;
EXPLAIN SELECT * FROM t_sort ORDER BY payload;
```

预期观察：LIMIT 版本可能启动成本和排序方法更低。

源码回看：跟 `limit_tuples` 分支。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. HashAgg 分组数

SQL：

```sql
CREATE TABLE t_agg AS SELECT g, g % 100000 AS k, md5(g::text) AS v FROM generate_series(1,800000) g;
ANALYZE t_agg;
SET work_mem = 4MB;
EXPLAIN SELECT k, count(*) FROM t_agg GROUP BY k;
SET enable_hashagg = off;
EXPLAIN SELECT k, count(*) FROM t_agg GROUP BY k;
```

预期观察：比较 HashAggregate 和 GroupAggregate 的输入前提。

源码回看：跟 `estimate_hashagg_tablesize()` 和 `cost_agg()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. Hash Join 批次

SQL：

```sql
CREATE TABLE hj1 AS SELECT g, g % 200000 AS k FROM generate_series(1,900000) g;
CREATE TABLE hj2 AS SELECT g, g AS k, repeat(x,100) AS pad FROM generate_series(1,400000) g;
ANALYZE hj1; ANALYZE hj2;
SET work_mem = 2MB;
EXPLAIN SELECT * FROM hj1 JOIN hj2 USING (k);
```

预期观察：低 work_mem 下观察 Hash Join 成本和执行期 Batches。

源码回看：跟 `initial_cost_hashjoin()` 与 `ExecChooseHashTableSize()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `cost_sort()`

先用注释和调用者确认它的职责：包装 `cost_tuplesort()`，把输入 startup/total 与排序自身成本合并到 SortPath。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `cost_tuplesort()`

先用注释和调用者确认它的职责：根据 `input_bytes`、`sort_mem_bytes` 和 `limit_tuples` 选择 quicksort、external merge 或 bounded heap 估算。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `cost_incremental_sort()`

先用注释和调用者确认它的职责：先估 presorted key 形成的 group 数，再按单组 sort 成本累计总成本。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `cost_material()`

先用注释和调用者确认它的职责：按输入 rows/width 判断 Materialize 是否可能溢出，并把写临时文件风险计入 total。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `cost_rescan()`

先用注释和调用者确认它的职责：为 repeated scan 提供第二次读取成本；Materialize、Sort、Memoize 等路径分支不同。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `cost_agg()`

先用注释和调用者确认它的职责：区分 plain/sorted/hash/mixed aggregate，把 per-tuple trans cost、final cost、group 数和内存风险合并。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `estimate_hashagg_tablesize()`

先用注释和调用者确认它的职责：把 group 数、entry width 和 transition 空间换成 HashAgg 预计内存。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `hash_agg_set_limits()`

先用注释和调用者确认它的职责：executor 侧计算 mem_limit、ngroups_limit 和 partition limit；planner 用它校准 HashAgg spill 边界。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

内存节点的关键是把 rows/width 换成 bytes，再判断是否跨过 work_mem 或 hash memory limit。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `work_mem` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `width` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `numGroups` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `numbatches` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | input_bytes 是否越过 work_mem | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_sort()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | LIMIT 是否触发 bounded sort | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_tuplesort()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | incremental sort 是否有 presorted keys | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_incremental_sort()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | HashAgg entry size 是否包含 transitionSpace | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_material()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | hash_mem_multiplier 是否影响 hash limit | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_rescan()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | Hash Join numbatches 是否大于 1 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_agg()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | Materialize 是正确性还是性能优化 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `estimate_hashagg_tablesize()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | 执行期 Disk/Batches 是否回写到 rows/width 偏差 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `hash_agg_set_limits()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `input_bytes 是否越过 work_mem` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `LIMIT 是否触发 bounded sort` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `incremental sort 是否有 presorted keys` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `HashAgg entry size 是否包含 transitionSpace` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `hash_mem_multiplier 是否影响 hash limit` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | 为什么同一条 SQL 在 `work_mem` 或 `hash_mem_multiplier` 附近，Sort、HashAggregate、Hash Join 与 Materialize 的成本会出现台阶，甚至切换到另一类执行策略？ |
| 运行模型 | `costsize.c` 把 rows 与 width 先换成字节和页；`cost_tuplesort()` 判断内存排序、外部归并和 bounded heap sort；`cost_agg()` 与 `estimate_hashagg_tablesize()` 估 HashAgg 内存；`initial_cost_hashjoin()` 调 executor 同源的 `ExecChooseHashTableSize()` 估 hash join batches；`cost_rescan()` 和 `cost_material()` 描述重复读取时的内存/临时文件边界。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
