# PostgreSQL Parallel Path 成本与 worker 数选择

## 课程定位

前置知识：已经理解单进程 scan、join、sort 与 hash 成本如何写入 Path。

本节唯一主问题：

```text
planner 为什么不会简单地把并行 worker 数当成线性加速；parallel path 如何在 worker 选择、partial path、Gather/Gather Merge 成本和 parallel safety 之间折中？
```

核心矛盾：并行计划可以分摊 CPU 和部分扫描 I/O，但必须支付 worker 启动、tuple 传输、leader 汇总、parallel safety 检查和执行器协同成本；某些参数化和 volatile 场景根本不能并行。

学完后应能解释一个计划为什么没有并行、为什么只有 2 个 workers、为什么 Gather 成本压过收益，以及为什么 Gather Merge 比 Gather 更贵但能保序。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

25-27 已经讲单个 scan、memory 节点和 join algorithm 的成本，本节把 parallel 作为横切属性叠到 Path 上。

本节不讲 executor DSM、parallel query worker 生命周期；这里只追 planner 如何生成、估价和剪枝 parallel Path。

后续 RelOptInfo / Path 课程会继续解释 partial_pathlist 与普通 pathlist 的保存边界。

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
`allpaths.c` 先用 `set_rel_consider_parallel()` 和 `is_parallel_safe()` 判断 relation 是否能并行；base rel 可通过 `create_plain_partial_paths()` 或 parallel index / bitmap 分支生成 partial path；`compute_parallel_worker()` 根据 heap/index pages 和 GUC 选择 worker 数；`costsize.c` 用 `get_parallel_divisor()` 分摊 CPU/rows，再由 `generate_gather_paths()` 创建 Gather 或 Gather Merge，并计入 `parallel_setup_cost` 与 `parallel_tuple_cost`。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `RelOptInfo.consider_parallel` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `Path.parallel_safe` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `Path.parallel_aware` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `Path.parallel_workers` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `partial_pathlist` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/allpaths.c` | `set_rel_consider_parallel()`、`create_plain_partial_paths()`、`compute_parallel_worker()`、`generate_gather_paths()`、`generate_useful_gather_paths()`。 |
| 2 | `src/backend/optimizer/path/costsize.c` | `get_parallel_divisor()`、`cost_gather()`、`cost_gather_merge()` 以及各 cost 函数里的 parallel divisor 分支。 |
| 3 | `src/backend/optimizer/util/pathnode.c` | `create_gather_path()`、`create_gather_merge_path()`、`create_seqscan_path()` 写入 parallel 字段。 |
| 4 | `src/backend/optimizer/util/clauses.c` | `is_parallel_safe()` 检查表达式是否能在 worker 执行。 |
| 5 | `src/backend/optimizer/path/indxpath.c` | parallel index path 和 parallel bitmap heap path 生成位置。 |
| 6 | `src/include/nodes/pathnodes.h` | `Path.parallel_aware`、`parallel_safe`、`parallel_workers`、`partial_pathlist`、`GatherPath`、`GatherMergePath`。 |
| 7 | `src/backend/utils/misc/guc_tables.c` | `max_parallel_workers_per_gather`、`parallel_setup_cost`、`parallel_tuple_cost`、parallel scan size GUC。 |

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

### 4.1. 大表才触发 worker

小表即使 enable parallel，也可能因 `min_parallel_table_scan_size` 返回 0 workers。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. Gather 吃掉收益

输出 rows 很多时，`parallel_tuple_cost * rows` 会让并行计划输给单进程计划。


### 4.3. Gather Merge 保序

有序 partial path 可生成 Gather Merge，额外 heap merge CPU 与 5% IPC bump 换来 pathkeys。


### 4.4. parallel unsafe 表达式

函数、target 或 qual 不安全时，relation 不考虑 parallel path。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `RelOptInfo.consider_parallel` | relation 是否考虑并行 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `Path.parallel_safe` | Path 能否放进 parallel plan | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `Path.parallel_aware` | 节点是否协同分片输入 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `Path.parallel_workers` | 期望 worker 数 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `partial_pathlist` | partial Path 候选集合 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `min_parallel_table_scan_size` | heap 并行阈值 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `min_parallel_index_scan_size` | index 并行阈值 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `max_parallel_workers_per_gather` | 单个 Gather worker 上限 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `parallel_setup_cost` | 启动 worker 成本 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `parallel_tuple_cost` | worker 到 leader 传输成本 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `get_parallel_divisor()` | 并行分摊因子 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |
| `GatherMergePath` | 保序汇总路径 | `src/backend/optimizer/path/allpaths.c` 等主线文件消费或写入。 |

### 5.1. `RelOptInfo.consider_parallel`

语义：relation 是否考虑并行。

来源：`set_rel_consider_parallel()` 结合 RTE、restriction、target 和函数安全性写入。

消费：没有它，partial path 不会生成。

偏差后果：这是候选生成开关，不是成本高低。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `Path.parallel_safe`

语义：Path 能否放进 parallel plan。

来源：Path 创建时从 parent rel、target、qual 和子 path 推导。

消费：`add_path()` 比较时 parallel-safe 也影响支配关系。

偏差后果：parallel safe 不代表 parallel aware。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `Path.parallel_aware`

语义：节点是否协同分片输入。

来源：parallel seqscan / bitmap heap 等节点会设置。

消费：executor 需要它避免多个 worker 重复处理同一输入。

偏差后果：Gather 本身汇总 partial path，不等于下层 aware。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `Path.parallel_workers`

语义：期望 worker 数。

来源：`compute_parallel_worker()` 或 reloption 写入。

消费：cost 函数用它计算 divisor，createplan 用它生成 Plan。

偏差后果：实际 worker 数可能少于计划值。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `partial_pathlist`

语义：partial Path 候选集合。

来源：`add_partial_path()` 维护，不能直接作为最终完整结果。

消费：`generate_gather_paths()` 消费它创建完整 Path。

偏差后果：没有 Gather，partial path 不会出现在最终 Plan 根上。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `min_parallel_table_scan_size`

语义：heap 并行阈值。

来源：GUC 转成页数后参与 `compute_parallel_worker()`。

消费：表太小时直接返回 0。

偏差后果：继承子表有特殊放宽逻辑，因为整体 append 可能值得并行。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `min_parallel_index_scan_size`

语义：index 并行阈值。

来源：与 index_pages 一起决定 parallel index worker 数。

消费：parallel index only scan 可能基于 index pages 而不是 heap pages。

偏差后果：覆盖查询不应因 heap fetch 少就完全丢掉 parallel index 机会。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `max_parallel_workers_per_gather`

语义：单个 Gather worker 上限。

来源：限制 `compute_parallel_worker()` 返回值。

消费：也是实验中最直观的 worker cap。

偏差后果：全局 worker 资源不足还可能让执行期实际 workers 更少。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `parallel_setup_cost`

语义：启动 worker 成本。

来源：`cost_gather()` / `cost_gather_merge()` 加到 startup。

消费：小查询常被它压住。

偏差后果：它是模型参数，不是 fork/exec 的精确时间。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `parallel_tuple_cost`

语义：worker 到 leader 传输成本。

来源：按 Gather 输出 rows 加到 run cost。

消费：宽结果或大量 rows 会放大它。

偏差后果：它解释了并行 scan 后还可能输给单进程。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `get_parallel_divisor()`

语义：并行分摊因子。

来源：考虑 worker 数和 leader 贡献。

消费：各 cost 函数用它缩小 per-worker rows / CPU。

偏差后果：它通常小于简单的 workers + 1 线性加速。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.12. `GatherMergePath`

语义：保序汇总路径。

来源：要求 subpath 有 pathkeys。

消费：`cost_gather_merge()` 额外计 heap merge 比较和略高 IPC。

偏差后果：它让 parallel 与 ORDER BY / Merge Join 有机会共存。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
set_rel_consider_parallel()
  -> set_plain_rel_pathlist()
  -> create_plain_partial_paths()
  -> compute_parallel_worker()
  -> create_seqscan_path()
  -> add_partial_path()
  -> cost_seqscan() / cost_index()
  -> generate_gather_paths()
  -> cost_gather()
  -> cost_gather_merge()
  -> add_path()
```

### 6.1. `set_rel_consider_parallel()`

先排除不能并行的 RTE、restriction、target 和 lateral/unsafe 表达式。

观察锚点：`RelOptInfo.consider_parallel`。

### 6.2. `set_plain_rel_pathlist()`

普通 seqscan 先进入 pathlist；如果 consider_parallel 为真，再尝试 partial path。

观察锚点：`Path.parallel_safe`。

### 6.3. `create_plain_partial_paths()`

调用 `compute_parallel_worker(rel, rel->pages, -1, max_parallel_workers_per_gather)`。

观察锚点：`Path.parallel_aware`。

### 6.4. `compute_parallel_worker()`

优先使用 reloption；否则按 heap/index pages 的 3 倍阈值增长选择 worker 数。

观察锚点：`Path.parallel_workers`。

### 6.5. `create_seqscan_path()`

当 workers > 0 时把 Path 标成 parallel-aware，并把 rows 交给 cost 函数缩放。

观察锚点：`partial_pathlist`。

### 6.6. `add_partial_path()`

在 partial_pathlist 内独立剪枝；它不处理 parameterized partial path。

观察锚点：`min_parallel_table_scan_size`。

### 6.7. `cost_seqscan() / cost_index()`

CPU 成本和 rows 根据 parallel divisor 缩放，I/O 分摊更保守。

观察锚点：`min_parallel_index_scan_size`。

### 6.8. `generate_gather_paths()`

从 cheapest partial path 创建普通 Gather，并为有 pathkeys 的 partial path 创建 Gather Merge。

观察锚点：`max_parallel_workers_per_gather`。

### 6.9. `cost_gather()`

加 parallel setup 和 tuple transfer，把 partial path 变成完整 Path 成本。

观察锚点：`parallel_setup_cost`。

### 6.10. `cost_gather_merge()`

在 Gather 基础上加每 tuple heap merge 比较和略高通信成本。

观察锚点：`parallel_tuple_cost`。

### 6.11. `add_path()`

Gather / Gather Merge 与非并行完整 Path 在同一 pathlist 中比较。

观察锚点：`get_parallel_divisor()`。

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

### 9.1. 表达式 parallel unsafe

候选不生成；调成本参数不会让 unsafe function 进入 worker。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. 页数未过阈值

`compute_parallel_worker()` 返回 0，partial path 直接缺席。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. parameterized partial path 缺席

`add_partial_path()` 注释说明当前不支持参数化 partial path，避免 worker 参数不同步。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. 实际 worker 不足

EXPLAIN ANALYZE 可能显示 Workers Planned 大于 Workers Launched；planner cost 不重新计算。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. Gather 传输过贵

大量 rows 或宽 target 会让 `parallel_tuple_cost` 抵消 CPU 分摊。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. Gather Merge 只在保序有价值时出现

没有 subpath pathkeys 时不会生成，排序可能留在 Gather 上方。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

parallel 计划的关键是先确认 safety 和 partial path，再计算 worker、Gather 和传输成本。

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
| consider_parallel 是否为真 | `set_rel_consider_parallel()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| parallel_safe 是否被 qual/target 破坏 | `set_plain_rel_pathlist()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| compute_parallel_worker 返回几个 worker | `create_plain_partial_paths()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| partial_pathlist 是否有候选 | `compute_parallel_worker()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| CPU 成本是否按 divisor 缩放 | `create_seqscan_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Gather 传输成本是否支配 | `add_partial_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Gather Merge 是否保留 pathkeys | `cost_seqscan() / cost_index()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Workers Planned 与 Launched 是否偏离 | `generate_gather_paths()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `consider_parallel 是否为真` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `parallel_safe 是否被 qual/target 破坏` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `compute_parallel_worker 返回几个 worker` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `partial_pathlist 是否有候选` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `CPU 成本是否按 divisor 缩放` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Gather 传输成本是否支配` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Gather Merge 是否保留 pathkeys` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Workers Planned 与 Launched 是否偏离` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 认为 workers 翻倍就 cost 除以二；leader、通信和启动成本都在模型里。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 只看 Gather 节点，不看 partial path 的 rows 已经按 divisor 缩小。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 把 parallel_safe 和 parallel_aware 混用；一个是安全属性，一个是协同扫描语义。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 忽略实际 worker 短缺；planner 的 planned workers 不是执行期保证。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 试图让参数化 inner path 并行；当前 planner 明确避免 parameterized partial paths。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. parallel seqscan 阈值

SQL：

```sql
CREATE TABLE p_big AS SELECT g, md5(g::text) AS v FROM generate_series(1,2000000) g;
ANALYZE p_big;
SET max_parallel_workers_per_gather = 4;
SET min_parallel_table_scan_size = 8MB;
EXPLAIN SELECT count(*) FROM p_big;
SET min_parallel_table_scan_size = 1GB;
EXPLAIN SELECT count(*) FROM p_big;
```

预期观察：观察 partial path 是否消失。

源码回看：跟 `compute_parallel_worker()` 返回值。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. parallel_tuple_cost

SQL：

```sql
SET parallel_tuple_cost = 0.1;
EXPLAIN SELECT * FROM p_big;
SET parallel_tuple_cost = 1.0;
EXPLAIN SELECT * FROM p_big;
```

预期观察：大量输出下 Gather 成本明显变化。

源码回看：跟 `cost_gather()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. Gather Merge

SQL：

```sql
CREATE INDEX ON p_big(v);
ANALYZE p_big;
SET max_parallel_workers_per_gather = 4;
EXPLAIN SELECT * FROM p_big ORDER BY v LIMIT 10000;
```

预期观察：观察是否出现 Gather Merge 或上层 Sort。

源码回看：跟 `generate_gather_paths()` 和 `cost_gather_merge()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. parallel unsafe 函数

SQL：

```sql
CREATE OR REPLACE FUNCTION f_unsafe(text) RETURNS text LANGUAGE plpgsql PARALLEL UNSAFE AS $$ BEGIN RETURN $1; END $$;
EXPLAIN SELECT f_unsafe(v) FROM p_big WHERE g > 0;
```

预期观察：target 中 unsafe function 会限制并行。

源码回看：跟 `is_parallel_safe()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `set_rel_consider_parallel()`

先用注释和调用者确认它的职责：先排除不能并行的 RTE、restriction、target 和 lateral/unsafe 表达式。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `set_plain_rel_pathlist()`

先用注释和调用者确认它的职责：普通 seqscan 先进入 pathlist；如果 consider_parallel 为真，再尝试 partial path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `create_plain_partial_paths()`

先用注释和调用者确认它的职责：调用 `compute_parallel_worker(rel, rel->pages, -1, max_parallel_workers_per_gather)`。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `compute_parallel_worker()`

先用注释和调用者确认它的职责：优先使用 reloption；否则按 heap/index pages 的 3 倍阈值增长选择 worker 数。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `create_seqscan_path()`

先用注释和调用者确认它的职责：当 workers > 0 时把 Path 标成 parallel-aware，并把 rows 交给 cost 函数缩放。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `add_partial_path()`

先用注释和调用者确认它的职责：在 partial_pathlist 内独立剪枝；它不处理 parameterized partial path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `cost_seqscan() / cost_index()`

先用注释和调用者确认它的职责：CPU 成本和 rows 根据 parallel divisor 缩放，I/O 分摊更保守。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `generate_gather_paths()`

先用注释和调用者确认它的职责：从 cheapest partial path 创建普通 Gather，并为有 pathkeys 的 partial path 创建 Gather Merge。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

parallel 计划的关键是先确认 safety 和 partial path，再计算 worker、Gather 和传输成本。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `consider_parallel` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `parallel_workers` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `partial_pathlist` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `GatherPath` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | consider_parallel 是否为真 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `set_rel_consider_parallel()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | parallel_safe 是否被 qual/target 破坏 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `set_plain_rel_pathlist()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | compute_parallel_worker 返回几个 worker | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_plain_partial_paths()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | partial_pathlist 是否有候选 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `compute_parallel_worker()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | CPU 成本是否按 divisor 缩放 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_seqscan_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | Gather 传输成本是否支配 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `add_partial_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | Gather Merge 是否保留 pathkeys | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_seqscan() / cost_index()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | Workers Planned 与 Launched 是否偏离 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `generate_gather_paths()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `consider_parallel 是否为真` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `parallel_safe 是否被 qual/target 破坏` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `compute_parallel_worker 返回几个 worker` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `partial_pathlist 是否有候选` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `CPU 成本是否按 divisor 缩放` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | planner 为什么不会简单地把并行 worker 数当成线性加速；parallel path 如何在 worker 选择、partial path、Gather/Gather Merge 成本和 parallel safety 之间折中？ |
| 运行模型 | `allpaths.c` 先用 `set_rel_consider_parallel()` 和 `is_parallel_safe()` 判断 relation 是否能并行；base rel 可通过 `create_plain_partial_paths()` 或 parallel index / bitmap 分支生成 partial path；`compute_parallel_worker()` 根据 heap/index pages 和 GUC 选择 worker 数；`costsize.c` 用 `get_parallel_divisor()` 分摊 CPU/rows，再由 `generate_gather_paths()` 创建 Gather 或 Gather Merge，并计入 `parallel_setup_cost` 与 `parallel_tuple_cost`。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
