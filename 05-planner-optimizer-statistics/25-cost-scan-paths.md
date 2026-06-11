# PostgreSQL SeqScan、IndexScan 与 BitmapScan 成本

## 课程定位

前置知识：已经理解成本单位、rows 估算和 base RelOptInfo 的 pages / tuples 来源。

本节唯一主问题：

```text
同一张表和同一个谓词，planner 为什么会在 Seq Scan、Index Scan、Index Only Scan 与 Bitmap Heap Scan 之间给出不同成本，并且这种判断为什么会被 all-visible 页比例、索引相关性和缓存假设改写？
```

核心矛盾：扫描成本要在执行前把 heap page、index page、tuple CPU、qual CPU 和随机 I/O 压成两个数字；真实缓存命中、visibility map 命中、bitmap lossy 程度和数据聚簇程度却都在执行期才显现。

学完后应能把一个 scan 选择拆成统计输入、Path 生成、cost 函数、add_path 剪枝和 EXPLAIN 影子，而不是只说“优化器选错扫描方式”。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前一节建立了成本参数的相对单位，本节把这些单位落到最常见的 base relation scan。

本节不讲索引匹配算法本身；`indxpath.c` 里 clause 如何变成 IndexClause 会在后续 base path 课程继续展开。

本节关注一个更窄的问题：scan 候选已经能生成时，成本函数如何让它们可比较。

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
`plancat.c` 把 relation / index 元数据写入 `RelOptInfo` 和 `IndexOptInfo`；`allpaths.c` 与 `indxpath.c` 生成 scan 候选；`pathnode.c` 创建具体 Path；`costsize.c` 用页数、选择率、相关性、allvisfrac、qual cost 和 parallel divisor 写入 `startup_cost` / `total_cost`；`add_path()` 决定候选是否留下。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `RelOptInfo.pages` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `RelOptInfo.tuples` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `RelOptInfo.allvisfrac` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `IndexOptInfo.pages` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `IndexPath.indexselectivity` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/costsize.c` | `cost_seqscan()`、`cost_index()`、`index_pages_fetched()`、`cost_bitmap_heap_scan()`、`compute_bitmap_pages()` 是成本主线。 |
| 2 | `src/backend/optimizer/path/indxpath.c` | `create_index_paths()`、`get_index_paths()`、`build_index_paths()`、`choose_bitmap_and()` 和 `check_index_only()` 生成索引相关候选。 |
| 3 | `src/backend/optimizer/path/allpaths.c` | `set_plain_rel_pathlist()` 和 `create_plain_partial_paths()` 把普通 seqscan 与 parallel seqscan 放入 base rel。 |
| 4 | `src/backend/optimizer/util/pathnode.c` | `create_seqscan_path()`、`create_index_path()`、`create_bitmap_heap_path()` 初始化 Path 并调用 cost 函数。 |
| 5 | `src/backend/utils/adt/selfuncs.c` | `btcostestimate()`、`genericcostestimate()` 给 index cost 提供选择率、相关性和索引页估算。 |
| 6 | `src/backend/optimizer/util/plancat.c` | `get_relation_info()` 读取 relation size、indexlist、allvisfrac 等输入状态。 |
| 7 | `src/include/nodes/pathnodes.h` | `RelOptInfo`、`IndexOptInfo`、`IndexPath`、`BitmapHeapPath` 字段定义。 |
| 8 | `src/include/optimizer/cost.h` | scan cost 函数原型和成本参数声明。 |

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

### 4.1. random_page_cost 改变扫描边界

同一高选择性谓词，在大表上把 `random_page_cost` 从 4 调到 1.1，Index Scan 的 `total_cost` 会更容易压过 Seq Scan。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. VACUUM 改写 index-only 成本

覆盖索引满足输出列时，`VACUUM` 后 all-visible 页比例提高，`cost_index()` 对 heap fetch 的折扣会变大。


### 4.3. BitmapAnd 的两段成本

两个单列索引组合时，Bitmap Index Scan 先付索引和 bitmap 构建成本，Bitmap Heap Scan 再按 heap 页和 recheck CPU 计费。


### 4.4. 相关性影响随机 I/O

CLUSTER 过的索引列会让 `indexCorrelation` 接近有序访问，`cost_index()` 在随机页和顺序页之间插值。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `RelOptInfo.pages` | heap page 规模 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `RelOptInfo.tuples` | 表级 tuple 规模 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `RelOptInfo.allvisfrac` | all-visible 页比例 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `IndexOptInfo.pages` | 索引页规模 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `IndexPath.indexselectivity` | 索引谓词选择率 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `IndexPath.indextotalcost` | 索引访问内部成本 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `indexCorrelation` | 索引顺序与 heap 顺序相关性 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `loop_count` | 重复扫描次数估计 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `BitmapHeapPath.bitmapqual` | bitmap tree 根节点 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `path.disabled_nodes` | 被 enable GUC 惩罚的节点数 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |
| `path.pathkeys` | 扫描天然输出排序 | `src/backend/optimizer/path/costsize.c` 等主线文件消费或写入。 |

### 5.1. `RelOptInfo.pages`

语义：heap page 规模。

来源：由 `get_relation_info()` 根据 pg_class / storage 信息写入，是 seqscan I/O 与 parallel worker 粗估的基础。

消费：被 `cost_seqscan()`、`cost_index()`、`cost_bitmap_heap_scan()` 和 `compute_parallel_worker()` 消费。

偏差后果：偏小会让全表扫描和 bitmap heap 访问显得过分便宜。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `RelOptInfo.tuples`

语义：表级 tuple 规模。

来源：与 selectivity 相乘形成 rows，也进入 CPU tuple cost。

消费：scan cost 里 tuple CPU 和 index pages fetched 都依赖它。

偏差后果：ANALYZE 过旧时，rows 与实际 loops 的偏差会从这里开始。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `RelOptInfo.allvisfrac`

语义：all-visible 页比例。

来源：`plancat.c` 根据 visibility map 统计写入，是 Index Only Scan 的 heap fetch 折扣。

消费：`cost_index()` 用 `1.0 - allvisfrac` 缩小估计 heap pages。

偏差后果：它是全表比例，不是谓词命中的精确页比例。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `IndexOptInfo.pages`

语义：索引页规模。

来源：来自索引 relation 元数据。

消费：`cost_index()` 估 index I/O，parallel index scan 也会读取它。

偏差后果：重建索引或膨胀后，这个值会改变 scan 边界。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `IndexPath.indexselectivity`

语义：索引谓词选择率。

来源：AM cost estimate 写回 `IndexPath`。

消费：普通 Index Scan、Bitmap Heap Scan 和 BitmapAnd/Or 都复用这个值。

偏差后果：它不是最终 rows；bitmap 组合和 qpqual 仍会继续改变输出。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `IndexPath.indextotalcost`

语义：索引访问内部成本。

来源：`cost_index()` 保存，不包括全部 heap tuple 检查。

消费：`cost_bitmap_tree_node()` 读取它计算 bitmap tree。

偏差后果：如果只看 Bitmap Heap 节点，很容易漏掉前置 indexTotalCost。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `indexCorrelation`

语义：索引顺序与 heap 顺序相关性。

来源：由 access method cost 估算返回。

消费：`cost_index()` 用平方相关性在随机 I/O 与顺序 I/O 成本之间插值。

偏差后果：CLUSTER、乱序插入和表膨胀都会改变它的解释。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `loop_count`

语义：重复扫描次数估计。

来源：参数化 nested loop 内层 index scan 会传入大于 1 的循环次数。

消费：`index_pages_fetched()` 用它折算缓存摊销。

偏差后果：这个值让“单次便宜”的 index scan 在外层 loops 放大时重新计价。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `BitmapHeapPath.bitmapqual`

语义：bitmap tree 根节点。

来源：可能是单个 IndexPath，也可能是 BitmapAndPath / BitmapOrPath。

消费：`compute_bitmap_pages()` 遍历它拿成本、选择率和 heap 页估算。

偏差后果：它不是 executor 的 TIDBitmap，只是 planner 的候选结构。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `path.disabled_nodes`

语义：被 enable GUC 惩罚的节点数。

来源：cost 函数根据 `pgs_mask` 和 scan 类型写入。

消费：`add_path()` 先比较 disabled_nodes，再比较 startup / total。

偏差后果：关闭 enable_seqscan 并不会删除 Seq Scan，只是把它放到更差的比较层。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `path.pathkeys`

语义：扫描天然输出排序。

来源：只有有序索引路径可能携带。

消费：上层 ORDER BY 或 Merge Join 会用它避免 Sort。

偏差后果：一个 scan 不是仅靠 total_cost 留下，排序价值也能保留候选。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
get_relation_info()
  -> set_plain_rel_pathlist()
  -> create_plain_partial_paths()
  -> create_index_paths()
  -> build_index_paths()
  -> cost_index()
  -> index_pages_fetched()
  -> choose_bitmap_and()
  -> create_bitmap_heap_path()
  -> add_path()
```

### 6.1. `get_relation_info()`

读取 heap pages、tuples、allvisfrac 和 indexlist，把执行期不可直接知道的物理事实压到 base RelOptInfo。

观察锚点：`RelOptInfo.pages`。

### 6.2. `set_plain_rel_pathlist()`

先把无参数或带 lateral 参数的 Seq Scan 放进 pathlist，这是扫描搜索空间的保底候选。

观察锚点：`RelOptInfo.tuples`。

### 6.3. `create_plain_partial_paths()`

当 relation 允许 parallel 且页数达到阈值时，生成 partial seqscan，后面还要经 Gather 回到完整 Path。

观察锚点：`RelOptInfo.allvisfrac`。

### 6.4. `create_index_paths()`

遍历 indexlist，把可用 index clause、ORDER BY 支持、predicate、index-only 条件拆成普通 index path 与 bitmap path 两类。

观察锚点：`IndexOptInfo.pages`。

### 6.5. `build_index_paths()`

为一个索引构造 IndexPath，决定 pathtype 是 IndexScan 还是 IndexOnlyScan，并传入 required_outer。

观察锚点：`IndexPath.indexselectivity`。

### 6.6. `cost_index()`

调用 AM cost estimate，保存 `indextotalcost` / `indexselectivity`，再估 heap fetch、相关性插值、qpqual CPU 和 target cost。

观察锚点：`IndexPath.indextotalcost`。

### 6.7. `index_pages_fetched()`

用 Mackert-Lohman 近似和 `effective_cache_size` 折算缓存效果，不假装知道真实 buffer cache。

观察锚点：`indexCorrelation`。

### 6.8. `choose_bitmap_and()`

在多个 bitmap 输入中寻找值得组合的集合，避免把每个索引都机械 AND 进去。

观察锚点：`loop_count`。

### 6.9. `create_bitmap_heap_path()`

把 bitmapqual 包成 BitmapHeapPath，再由 `cost_bitmap_heap_scan()` 计算 heap 访问与 recheck CPU。

观察锚点：`BitmapHeapPath.bitmapqual`。

### 6.10. `add_path()`

把 scan Path 与已有候选比较，成本、pathkeys、required_outer 和 parallel_safe 共同决定是否保留。

观察锚点：`path.disabled_nodes`。

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

### 9.1. 没有可匹配 index clause

`create_index_paths()` 可能只留下 Seq Scan；诊断时应先确认候选有没有生成，而不是先调 cost 参数。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. Index Only Scan 被拒绝

`check_index_only()` 会检查输出和谓词列是否由索引覆盖；不覆盖时 allvisfrac 再高也不能生成 index-only path。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. bitmap recheck 保守计费

`cost_bitmap_heap_scan()` 对 scan clauses 采取偏保守的 recheck CPU 估算，执行期 lossy 与 non-lossy 不会在 planner 中完全精确区分。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. allvisfrac 粒度过粗

全表 all-visible 比例无法表达“谓词命中的那批页是否 all-visible”，这是 index-only 成本的稳定近似边界。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. 缓存近似不可验证

`index_pages_fetched()` 使用 `effective_cache_size` 模型，不读取当前 shared_buffers 和 OS cache。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. enable GUC 只是惩罚

`enable_seqscan=false` 等 GUC 会进入 `disabled_nodes`，保底路径仍可能在没有合法替代时被选中。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

scan 计划的关键是区分候选缺失、heap/index 页成本、visibility map 折扣和 bitmap recheck。

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
| 候选是否生成 | `get_relation_info()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| rows 是否来自当前统计 | `set_plain_rel_pathlist()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| heap pages 与 index pages 哪个支配成本 | `create_plain_partial_paths()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| allvisfrac 是否解释 index-only 差异 | `create_index_paths()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| pathkeys 是否让较贵 scan 保留 | `build_index_paths()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| disabled_nodes 是否来自 enable GUC | `cost_index()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| loop_count 是否放大 index scan | `index_pages_fetched()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| bitmapqual 是否隐藏了 indexTotalCost | `choose_bitmap_and()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `候选是否生成` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `rows 是否来自当前统计` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `heap pages 与 index pages 哪个支配成本` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `allvisfrac 是否解释 index-only 差异` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `pathkeys 是否让较贵 scan 保留` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `disabled_nodes 是否来自 enable GUC` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `loop_count 是否放大 index scan` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `bitmapqual 是否隐藏了 indexTotalCost` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 把 `cost` 当运行时间；scan cost 是相对比较单位，不包含当前 cache 的真实状态。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 只看最终 Plan；被 `add_path()` 丢掉的候选才解释了为什么另一个扫描方式没有出现。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 把 Index Only Scan 理解成永不访问 heap；visibility map 不满足时仍可能需要 heap fetch。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 认为 bitmap 一定比普通 index scan 适合多条件；bitmap 构建、heap recheck 和 page 数都会重新计价。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 用 enable GUC 作为永久调优答案；它适合做实验隔离，不适合替代统计和 schema 分析。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. 扫描 GUC 对照

SQL：

```sql
CREATE TABLE t_scan AS SELECT g AS id, md5(g::text) AS payload FROM generate_series(1,1000000) g;
CREATE INDEX ON t_scan(id);
ANALYZE t_scan;
SET random_page_cost = 4;
EXPLAIN SELECT * FROM t_scan WHERE id = 42;
SET random_page_cost = 1.1;
EXPLAIN SELECT * FROM t_scan WHERE id = 42;
```

预期观察：观察 Seq Scan / Index Scan 是否切换，同时记录 estimated rows 是否没变。

源码回看：断点放在 `cost_seqscan()` 与 `cost_index()`，比较 I/O 分量而不是 rows。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. Index Only Scan 与 VACUUM

SQL：

```sql
CREATE TABLE t_ios AS SELECT g AS id, g AS v FROM generate_series(1,300000) g;
CREATE INDEX ON t_ios(id) INCLUDE (v);
ANALYZE t_ios;
EXPLAIN SELECT v FROM t_ios WHERE id BETWEEN 10 AND 20;
VACUUM t_ios;
EXPLAIN SELECT v FROM t_ios WHERE id BETWEEN 10 AND 20;
```

预期观察：如果 path 生成条件满足，VACUUM 后 index-only 的 heap 折扣更明显。

源码回看：跟 `RelOptInfo.allvisfrac` 和 `cost_index()` 中 indexonly 分支。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. BitmapAnd 分界

SQL：

```sql
CREATE TABLE t_bitmap AS SELECT g, g % 100 AS a, g % 200 AS b FROM generate_series(1,500000) g;
CREATE INDEX ON t_bitmap(a);
CREATE INDEX ON t_bitmap(b);
ANALYZE t_bitmap;
EXPLAIN SELECT * FROM t_bitmap WHERE a = 7 AND b = 107;
```

预期观察：观察 BitmapAnd、Bitmap Index Scan 与 Bitmap Heap Scan 的两段成本。

源码回看：跟 `choose_bitmap_and()`、`cost_bitmap_tree_node()`、`cost_bitmap_heap_scan()`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. 相关性变化

SQL：

```sql
CREATE TABLE t_corr AS SELECT g AS id, repeat(x,20) AS pad FROM generate_series(1,500000) g;
CREATE INDEX ON t_corr(id);
ANALYZE t_corr;
EXPLAIN SELECT * FROM t_corr WHERE id BETWEEN 1000 AND 2000;
CLUSTER t_corr USING t_corr_id_idx;
ANALYZE t_corr;
EXPLAIN SELECT * FROM t_corr WHERE id BETWEEN 1000 AND 2000;
```

预期观察：比较 index scan 成本中 heap I/O 的变化趋势。

源码回看：跟 selfuncs 返回的 correlation 与 `cost_index()` 插值。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `get_relation_info()`

先用注释和调用者确认它的职责：读取 heap pages、tuples、allvisfrac 和 indexlist，把执行期不可直接知道的物理事实压到 base RelOptInfo。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `set_plain_rel_pathlist()`

先用注释和调用者确认它的职责：先把无参数或带 lateral 参数的 Seq Scan 放进 pathlist，这是扫描搜索空间的保底候选。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `create_plain_partial_paths()`

先用注释和调用者确认它的职责：当 relation 允许 parallel 且页数达到阈值时，生成 partial seqscan，后面还要经 Gather 回到完整 Path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `create_index_paths()`

先用注释和调用者确认它的职责：遍历 indexlist，把可用 index clause、ORDER BY 支持、predicate、index-only 条件拆成普通 index path 与 bitmap path 两类。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `build_index_paths()`

先用注释和调用者确认它的职责：为一个索引构造 IndexPath，决定 pathtype 是 IndexScan 还是 IndexOnlyScan，并传入 required_outer。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `cost_index()`

先用注释和调用者确认它的职责：调用 AM cost estimate，保存 `indextotalcost` / `indexselectivity`，再估 heap fetch、相关性插值、qpqual CPU 和 target cost。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `index_pages_fetched()`

先用注释和调用者确认它的职责：用 Mackert-Lohman 近似和 `effective_cache_size` 折算缓存效果，不假装知道真实 buffer cache。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `choose_bitmap_and()`

先用注释和调用者确认它的职责：在多个 bitmap 输入中寻找值得组合的集合，避免把每个索引都机械 AND 进去。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

scan 计划的关键是区分候选缺失、heap/index 页成本、visibility map 折扣和 bitmap recheck。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `RelOptInfo.pages` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `IndexPath.indexselectivity` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `RelOptInfo.allvisfrac` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `BitmapHeapPath.bitmapqual` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | 候选是否生成 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `get_relation_info()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | rows 是否来自当前统计 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `set_plain_rel_pathlist()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | heap pages 与 index pages 哪个支配成本 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_plain_partial_paths()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | allvisfrac 是否解释 index-only 差异 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_index_paths()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | pathkeys 是否让较贵 scan 保留 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `build_index_paths()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | disabled_nodes 是否来自 enable GUC | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `cost_index()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | loop_count 是否放大 index scan | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `index_pages_fetched()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | bitmapqual 是否隐藏了 indexTotalCost | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `choose_bitmap_and()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `候选是否生成` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `rows 是否来自当前统计` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `heap pages 与 index pages 哪个支配成本` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `allvisfrac 是否解释 index-only 差异` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `pathkeys 是否让较贵 scan 保留` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | 同一张表和同一个谓词，planner 为什么会在 Seq Scan、Index Scan、Index Only Scan 与 Bitmap Heap Scan 之间给出不同成本，并且这种判断为什么会被 all-visible 页比例、索引相关性和缓存假设改写？ |
| 运行模型 | `plancat.c` 把 relation / index 元数据写入 `RelOptInfo` 和 `IndexOptInfo`；`allpaths.c` 与 `indxpath.c` 生成 scan 候选；`pathnode.c` 创建具体 Path；`costsize.c` 用页数、选择率、相关性、allvisfrac、qual cost 和 parallel divisor 写入 `startup_cost` / `total_cost`；`add_path()` 决定候选是否留下。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
