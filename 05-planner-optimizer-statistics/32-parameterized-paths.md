# PostgreSQL Parameterized Path 与 LATERAL / Nested Loop 依赖

## 课程定位

前置知识：已经理解 RelOptInfo、Path 和 PathTarget 的搜索空间边界。

本节唯一主问题：

```text
一个 Path 为什么可能依赖外层 relation 提供参数；parameterized path 如何支持 LATERAL、相关子查询和索引内层 Nested Loop，同时又限制 join order 的可交换性？
```

核心矛盾：参数化路径能把外层值下推到内层索引或子查询中，显著降低内层 rows；但它不能独立执行，必须由包含 required_outer 的 join 顺序驱动，过度参数化还会扩大搜索状态和 rescan 风险。

学完后应能判断一个 index path 为什么只能作为 nested loop inner，也能解释某些 join order 为什么被 LATERAL 或 required_outer 约束排除。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节建立了 Path 的共同字段，本节聚焦其中最容易误读的 `param_info`。

本节只讲 planner 侧 parameterized path 语义，不展开 executor Param 赋值和 SubPlan 全部细节。

后续 base path 与 join search 课程会继续使用 required_outer 判断候选是否合法。

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
`initsplan.c` 先提取 lateral 引用并写入 `direct_lateral_relids` / `lateral_relids`；Path 创建函数根据 required_outer 调 `get_baserel_parampathinfo()` 或 `get_joinrel_parampathinfo()`；`ParamPathInfo` 缓存 `ppi_req_outer`、`ppi_rows`、可下推 clauses 和 serials；`joinpath.c` 只允许 required_outer 合法的路径进入 NestLoop 或非 NestLoop join；`createplan.c` 用 `replace_nestloop_params()` 把 planner 里的外层 Var 落成 executor Param。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `Path.param_info` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `ParamPathInfo.ppi_req_outer` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `ParamPathInfo.ppi_rows` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `ParamPathInfo.ppi_clauses` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `ParamPathInfo.ppi_serials` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/nodes/pathnodes.h` | `ParamPathInfo`、`Path.param_info`、`PATH_REQ_OUTER()`、`RelOptInfo.lateral_relids`。 |
| 2 | `src/backend/optimizer/plan/initsplan.c` | `extract_lateral_references()`、`create_lateral_join_info()` 填充 lateral 边界。 |
| 3 | `src/backend/optimizer/util/relnode.c` | `get_baserel_parampathinfo()`、`get_joinrel_parampathinfo()`、`get_appendrel_parampathinfo()`、`find_param_path_info()`。 |
| 4 | `src/backend/optimizer/util/pathnode.c` | `calc_nestloop_required_outer()`、`calc_non_nestloop_required_outer()`、`reparameterize_path()` 和各 create path 函数。 |
| 5 | `src/backend/optimizer/path/allpaths.c` | base rel path 生成、append child path 重参数化、cheapest parameterized child path。 |
| 6 | `src/backend/optimizer/path/joinpath.c` | `try_nestloop_path()` 检查 required_outer，决定参数化 inner 是否可用。 |
| 7 | `src/backend/optimizer/path/joinrels.c` | `join_is_legal()` 与 lateral 约束共同限制 joinrel 构造。 |
| 8 | `src/backend/optimizer/plan/createplan.c` | `replace_nestloop_params()`、`process_subquery_nestloop_params()` 把参数化落实到 Plan。 |

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

### 4.1. LATERAL 索引内层

外层每行提供参数，内层 index scan rows 大幅下降，但该 path 只能放在 Nested Loop inner。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. 相关 EXISTS

子查询被拉成 semi join 后，内层路径可能仍带 required_outer。


### 4.3. 非 NestLoop 参数化受限

Hash/Merge Join 不能像 Nested Loop 那样逐外层行提供参数。


### 4.4. join order 不再完全可交换

LATERAL 和 required_outer 会让某些 rel 必须先于另一些 rel 出现。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `Path.param_info` | Path 的参数化身份 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `ParamPathInfo.ppi_req_outer` | 提供参数的外层 relids | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `ParamPathInfo.ppi_rows` | 参数可用时的 rows | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `ParamPathInfo.ppi_clauses` | base rel 参数化下推 clauses | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `ParamPathInfo.ppi_serials` | 已执行 RestrictInfo serial 集合 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `RelOptInfo.direct_lateral_relids` | 直接 lateral 引用 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `RelOptInfo.lateral_relids` | 最小参数化需求 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `lateral_referencers` | 谁引用当前 rel | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `calc_nestloop_required_outer()` | Nested Loop 结果参数化计算 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `calc_non_nestloop_required_outer()` | 非 NestLoop 参数化计算 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `reparameterize_path()` | child path 重参数化 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |
| `replace_nestloop_params()` | Plan 阶段参数替换 | `src/include/nodes/pathnodes.h` 等主线文件消费或写入。 |

### 5.1. `Path.param_info`

语义：Path 的参数化身份。

来源：非空指向 ParamPathInfo。

消费：`PATH_REQ_OUTER()` 通过它读取 required_outer。

偏差后果：这是执行依赖，不是普通注释字段。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `ParamPathInfo.ppi_req_outer`

语义：提供参数的外层 relids。

来源：由 required_outer 决定。

消费：add_path、joinpath、createplan 都会读取。

偏差后果：同一 rel + 同一 req_outer 共享一个 PPI。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `ParamPathInfo.ppi_rows`

语义：参数可用时的 rows。

来源：`get_parameterized_baserel_size()` 或 joinrel size 估算。

消费：Path rows 直接取它。

偏差后果：它可能小于 parent->rows，是参数化价值来源。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `ParamPathInfo.ppi_clauses`

语义：base rel 参数化下推 clauses。

来源：movable join clauses 和 EC 派生 equality。

消费：createplan 需要知道哪些 qual 已由 path 执行。

偏差后果：joinrel PPI 通常不在这里保存同样语义。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `ParamPathInfo.ppi_serials`

语义：已执行 RestrictInfo serial 集合。

来源：base rel PPI 维护。

消费：避免 createplan 或上层重复处理相同 qual。

偏差后果：serial 比文本比较更稳定。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `RelOptInfo.direct_lateral_relids`

语义：直接 lateral 引用。

来源：来自当前 rel 表达式直接提到的外层 rel。

消费：create_lateral_join_info 用它传播。

偏差后果：直接依赖和间接依赖要区分。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `RelOptInfo.lateral_relids`

语义：最小参数化需求。

来源：包含直接和间接 lateral 依赖。

消费：base path required_outer 至少覆盖它。

偏差后果：不满足时 path 不是成本高，而是不合法。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `lateral_referencers`

语义：谁引用当前 rel。

来源：帮助 join order 判断。

消费：外层 rel 被哪些 lateral 项使用可在这里看到。

偏差后果：诊断 LATERAL 约束时常和 lateral_relids 成对看。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `calc_nestloop_required_outer()`

语义：Nested Loop 结果参数化计算。

来源：允许 inner path 引用 outer rel，并计算剩余外部需求。

消费：try_nestloop_path 用它判断候选能否生成。

偏差后果：这是 Nested Loop 能支持参数化 inner 的核心边界。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `calc_non_nestloop_required_outer()`

语义：非 NestLoop 参数化计算。

来源：Hash/Merge Join 不逐行传参，只能保留更严格的外部需求。

消费：try_hashjoin_path / try_mergejoin_path 使用。

偏差后果：解释参数化路径为何常推向 Nested Loop。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `reparameterize_path()`

语义：child path 重参数化。

来源：append/inheritance 场景把 parent relids 翻译到 child。

消费：allpaths 与 createplan 周边使用。

偏差后果：失败时候选会被放弃。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.12. `replace_nestloop_params()`

语义：Plan 阶段参数替换。

来源：把外层 Var / PlaceHolderVar 替换成 NestLoopParam。

消费：createplan 中 scan quals、join quals、index quals 都可能调用。

偏差后果：planner param_info 最终要落到 executor Param 才能执行。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
extract_lateral_references()
  -> create_lateral_join_info()
  -> create_index_path()
  -> get_baserel_parampathinfo()
  -> add_path()
  -> try_nestloop_path()
  -> get_joinrel_parampathinfo()
  -> try_hashjoin_path() / try_mergejoin_path()
  -> reparameterize_path()
  -> create_nestloop_plan()
  -> replace_nestloop_params()
```

### 6.1. `extract_lateral_references()`

扫描 base rel 的 lateral Vars / PlaceHolderVars，记录直接外层依赖。

观察锚点：`Path.param_info`。

### 6.2. `create_lateral_join_info()`

传播 lateral 依赖，形成每个 rel 的 lateral_relids 和 referencers。

观察锚点：`ParamPathInfo.ppi_req_outer`。

### 6.3. `create_index_path()`

带 outer rel 的 index clause 会传入 required_outer，并调用 get_baserel_parampathinfo()。

观察锚点：`ParamPathInfo.ppi_rows`。

### 6.4. `get_baserel_parampathinfo()`

查找或构造 PPI，收集 movable join clauses、EC equality、serials 并估 ppi_rows。

观察锚点：`ParamPathInfo.ppi_clauses`。

### 6.5. `add_path()`

不同 required_outer 的 Path 通常都保留，因为参数越多 rows 越低但可用 join order 越窄。

观察锚点：`ParamPathInfo.ppi_serials`。

### 6.6. `try_nestloop_path()`

计算 join path 的 required_outer，允许 inner 参数来自 outer。

观察锚点：`RelOptInfo.direct_lateral_relids`。

### 6.7. `get_joinrel_parampathinfo()`

为参数化 joinrel 估 rows，并把必须在该 join 层执行的 clauses 加入 restrict_clauses。

观察锚点：`RelOptInfo.lateral_relids`。

### 6.8. `try_hashjoin_path() / try_mergejoin_path()`

用非 NestLoop required_outer 规则拒绝无法满足的参数化组合。

观察锚点：`lateral_referencers`。

### 6.9. `reparameterize_path()`

append child 或 partition 场景把路径参数改写到 child relids。

观察锚点：`calc_nestloop_required_outer()`。

### 6.10. `create_nestloop_plan()`

生成 Nested Loop Plan 时收集 NestLoopParam。

观察锚点：`calc_non_nestloop_required_outer()`。

### 6.11. `replace_nestloop_params()`

把 scan/join/index qual 中的外层引用替换成 executor Param。

观察锚点：`reparameterize_path()`。

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

### 9.1. required_outer 为空

函数直接返回 NULL ParamPathInfo，普通 path 不带参数化成本。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. PPI 已存在

`find_param_path_info()` 命中时复用 rows，避免同一参数化不同 Path 估出不同 rows。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. movable clause 不满足

join clause 不能下推时，参数化路径失去选择率收益。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. 非 NestLoop 不接受参数传递

Hash/Merge Join 候选可能在 required_outer 检查处直接返回。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. child path 无法重参数化

partition / inheritance 场景下 translate 失败会放弃候选。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. createplan 替换失败风险

如果 planner 侧 param_info 与实际 qual 不一致，NestLoopParam 交接会出错；源码用 serials 和替换流程维持一致性。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

参数化路径的关键是 required_outer 是否满足，以及 planner 参数如何落到 NestLoopParam。

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
| Path.param_info 是否非空 | `extract_lateral_references()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| ppi_req_outer 包含哪些 relids | `create_lateral_join_info()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| ppi_rows 是否解释 inner rows 下降 | `create_index_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| ppi_clauses 是否来自 movable join clauses | `get_baserel_parampathinfo()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| lateral_relids 是否限制 base path | `add_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| NestLoop 是否唯一可行落点 | `try_nestloop_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| Hash/Merge 是否在 required_outer 处被拒绝 | `get_joinrel_parampathinfo()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| replace_nestloop_params 是否覆盖 qual/indexqual | `try_hashjoin_path() / try_mergejoin_path()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `Path.param_info 是否非空` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `ppi_req_outer 包含哪些 relids` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `ppi_rows 是否解释 inner rows 下降` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `ppi_clauses 是否来自 movable join clauses` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `lateral_relids 是否限制 base path` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `NestLoop 是否唯一可行落点` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `Hash/Merge 是否在 required_outer 处被拒绝` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `replace_nestloop_params 是否覆盖 qual/indexqual` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 把 parameterized path 当可独立执行的普通 path。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 只看成本，不看 required_outer 是否满足。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 认为 LATERAL 只是语法糖；它会进入 join order 约束。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 把 ppi_rows 与 parent->rows 混用。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 忽略 createplan 阶段还要把外层 Var 替换成 NestLoopParam。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. LATERAL index inner

SQL：

```sql
CREATE TABLE p_outer AS SELECT g FROM generate_series(1,1000) g;
CREATE TABLE p_inner AS SELECT g, g AS k, md5(g::text) AS v FROM generate_series(1,1000000) g;
CREATE INDEX ON p_inner(k);
ANALYZE p_outer; ANALYZE p_inner;
EXPLAIN SELECT * FROM p_outer o CROSS JOIN LATERAL (SELECT * FROM p_inner i WHERE i.k = o.g) s;
```

预期观察：观察 Nested Loop inner 的 Index Scan。

源码回看：断点看 `get_baserel_parampathinfo()` 的 `ppi_req_outer`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. 禁用 Nested Loop

SQL：

```sql
SET enable_nestloop = off;
EXPLAIN SELECT * FROM p_outer o CROSS JOIN LATERAL (SELECT * FROM p_inner i WHERE i.k = o.g) s;
RESET enable_nestloop;
```

预期观察：看参数化路径失去主要落点后计划如何变化。

源码回看：跟 `try_nestloop_path()` 与 disabled_nodes。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. 相关 EXISTS

SQL：

```sql
EXPLAIN SELECT * FROM p_outer o WHERE EXISTS (SELECT 1 FROM p_inner i WHERE i.k = o.g);
```

预期观察：观察 semi join 与参数化内层是否出现。

源码回看：跟 subquery pull-up 后的 joinpath。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. PPI 复用

SQL：

```sql
在 `find_param_path_info()` 和 `get_baserel_parampathinfo()` 断点，执行有多个可用索引的 LATERAL 查询。
```

预期观察：同一 required_outer 的 rows 应复用。

源码回看：源码入口 `src/backend/optimizer/util/relnode.c`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `extract_lateral_references()`

先用注释和调用者确认它的职责：扫描 base rel 的 lateral Vars / PlaceHolderVars，记录直接外层依赖。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `create_lateral_join_info()`

先用注释和调用者确认它的职责：传播 lateral 依赖，形成每个 rel 的 lateral_relids 和 referencers。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `create_index_path()`

先用注释和调用者确认它的职责：带 outer rel 的 index clause 会传入 required_outer，并调用 get_baserel_parampathinfo()。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `get_baserel_parampathinfo()`

先用注释和调用者确认它的职责：查找或构造 PPI，收集 movable join clauses、EC equality、serials 并估 ppi_rows。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `add_path()`

先用注释和调用者确认它的职责：不同 required_outer 的 Path 通常都保留，因为参数越多 rows 越低但可用 join order 越窄。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `try_nestloop_path()`

先用注释和调用者确认它的职责：计算 join path 的 required_outer，允许 inner 参数来自 outer。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `get_joinrel_parampathinfo()`

先用注释和调用者确认它的职责：为参数化 joinrel 估 rows，并把必须在该 join 层执行的 clauses 加入 restrict_clauses。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `try_hashjoin_path() / try_mergejoin_path()`

先用注释和调用者确认它的职责：用非 NestLoop required_outer 规则拒绝无法满足的参数化组合。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

参数化路径的关键是 required_outer 是否满足，以及 planner 参数如何落到 NestLoopParam。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `ppi_req_outer` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `ppi_rows` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `lateral_relids` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `replace_nestloop_params` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | Path.param_info 是否非空 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `extract_lateral_references()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | ppi_req_outer 包含哪些 relids | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_lateral_join_info()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | ppi_rows 是否解释 inner rows 下降 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `create_index_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | ppi_clauses 是否来自 movable join clauses | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `get_baserel_parampathinfo()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | lateral_relids 是否限制 base path | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `add_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | NestLoop 是否唯一可行落点 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `try_nestloop_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | Hash/Merge 是否在 required_outer 处被拒绝 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `get_joinrel_parampathinfo()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | replace_nestloop_params 是否覆盖 qual/indexqual | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `try_hashjoin_path() / try_mergejoin_path()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `Path.param_info 是否非空` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `ppi_req_outer 包含哪些 relids` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `ppi_rows 是否解释 inner rows 下降` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `ppi_clauses 是否来自 movable join clauses` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `lateral_relids 是否限制 base path` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | 一个 Path 为什么可能依赖外层 relation 提供参数；parameterized path 如何支持 LATERAL、相关子查询和索引内层 Nested Loop，同时又限制 join order 的可交换性？ |
| 运行模型 | `initsplan.c` 先提取 lateral 引用并写入 `direct_lateral_relids` / `lateral_relids`；Path 创建函数根据 required_outer 调 `get_baserel_parampathinfo()` 或 `get_joinrel_parampathinfo()`；`ParamPathInfo` 缓存 `ppi_req_outer`、`ppi_rows`、可下推 clauses 和 serials；`joinpath.c` 只允许 required_outer 合法的路径进入 NestLoop 或非 NestLoop join；`createplan.c` 用 `replace_nestloop_params()` 把 planner 里的外层 Var 落成 executor Param。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
