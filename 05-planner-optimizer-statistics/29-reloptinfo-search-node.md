# PostgreSQL RelOptInfo 作为搜索空间节点

## 课程定位

前置知识：已经理解 cost model 如何写入 Path，现在进入承载 Path 的搜索空间对象。

本节唯一主问题：

```text
为什么 optimizer 要用 RelOptInfo 表示 base rel、joinrel、upper rel 和其它 relation-like 对象，而不是直接在 Query 的 RangeTblEntry 或 JoinExpr 上挂候选计划？
```

核心矛盾：Query 语义树需要保持稳定，planner 搜索却要为 relids 集合、upper stage、inheritance child、FDW/custom 扩展和参数化边界生成大量临时状态；把二者混在一起会破坏语义、剪枝和最终 Plan 契约。

学完后应能看到 `root->simple_rel_array`、`join_rel_list` 或 `upper_rels` 时，判断当前 optimizer 正在搜索哪个 relation-like 对象以及候选 Path 挂在哪里。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

25-28 讲 Path 成本如何形成，本节回答这些 Path 属于哪个搜索节点。

本节不展开每类 Path 的内部字段；下一节会单独解释 Path 候选如何比较。

RelOptInfo 是 Query 语义树与最终 Plan 之间的 planner-local 工作对象。

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
`query_planner()` 建立 simple_rel_array；`build_simple_rel()` 为 base RTE 创建 RelOptInfo；`make_one_rel()` 逐层构造 joinrel；`fetch_upper_rel()` 为 grouping、window、distinct、ordered、final 等 upper stage 提供容器；每个 RelOptInfo 用 `relids` / kind 标识搜索节点，用 `pathlist`、`partial_pathlist`、`ppilist` 和 cheapest 指针保存候选与剪枝结果。
```

这句话里有三个边界。

第一，planner 的状态是 backend-local 的临时搜索状态，生命周期属于一次规划。

第二，成本数字不是执行时间预言，而是同一搜索空间内候选之间的比较货币。

第三，最终 Plan 只带 executor 需要的契约；未选中的 Path、PPI、workspace 和中间 target 都会留在 planner 阶段。

本节要训练的判断不是背公式，而是识别一次计划选择中哪个输入先被压缩，哪个状态继续传播，哪个 fallback 让结果看起来合理但来源已经不精确。

| 判断点 | 本节读法 |
| --- | --- |
| `reloptkind` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `relids` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `rows` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `reltarget` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |
| `pathlist` | 先确认它由谁写入，再确认谁消费；单独打印字段值通常不构成语义。 |

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/plan/planmain.c` | `query_planner()` 调 `setup_simple_rel_arrays()`、`deconstruct_jointree()`、`make_one_rel()`。 |
| 2 | `src/backend/optimizer/util/relnode.c` | `setup_simple_rel_arrays()`、`build_simple_rel()`、`find_base_rel()`、`build_join_rel()`、`find_join_rel()`、`fetch_upper_rel()`。 |
| 3 | `src/backend/optimizer/path/allpaths.c` | `make_one_rel()`、`set_base_rel_sizes()`、`set_base_rel_pathlists()` 生成 base / join path。 |
| 4 | `src/backend/optimizer/plan/initsplan.c` | `deconstruct_jointree()`、`distribute_restrictinfo_to_rels()` 把 qual 归属到 rel。 |
| 5 | `src/backend/optimizer/path/joinrels.c` | `standard_join_search()`、`make_join_rel()` 按 relids 集合构造 joinrel。 |
| 6 | `src/backend/optimizer/plan/planner.c` | `grouping_planner()` 和 `fetch_upper_rel()` 处理 upper relation 阶段。 |
| 7 | `src/include/nodes/pathnodes.h` | `RelOptInfo`、`RelOptKind`、`UpperRelationKind` 和 pathlist 字段。 |
| 8 | `src/include/optimizer/pathnode.h` | RelOptInfo 构造、查找、ParamPathInfo 接口声明。 |

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

### 4.1. EXPLAIN 只看最终 Plan

最终计划不会展示所有 RelOptInfo；断在 `add_path()` 才能看到同一 relids 上候选如何出现和消失。

观察分层：

| 层次 | 公开影子 | 源码回看 |
| --- | --- | --- |
| 计划形态 | node type、estimated rows、startup/total cost | 候选生成函数是否创建了对应 Path。 |
| 估算输入 | `pg_stats`、`pg_class`、GUC、relation size | rows、width、页数、内存或 required_outer 的写入点。 |
| 执行偏差 | actual rows、loops、Temp、Batches、Workers、Sort Method | 判断偏差来自统计、资源、并行调度还是 executor fallback。 |

每个现象都只选一个主变量做对照；一次实验同时改统计、GUC 和 SQL 形态，会让源码解释失去方向。

### 4.2. joinrel 以 relids 为身份

`a join b` 和 `b join a` 语义上可能形成同一个 relids 集合，但路径方向仍可不同。


### 4.3. upper rel 不是 base table

GROUP BY、WINDOW、DISTINCT、ORDERED、FINAL 都可能有自己的 RelOptInfo。


### 4.4. FDW/custom 扩展边界

扩展通常接触 RelOptInfo，而不是直接改 Query tree。


## 5. 关键数据结构与状态边界

本节只讲会影响主问题的字段组合，不复制结构体源码。

raw field 不是语义；field 加创建时机、访问者、合法性边界和下游消费位置，才构成可以诊断的状态。

| 状态 | 语义 | 主要源码位置 |
| --- | --- | --- |
| `reloptkind` | 搜索节点类别 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `relids` | 搜索节点身份 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `rows` | relation-like 输出行数 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `reltarget` | 默认输出 PathTarget | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `pathlist` | 完整 Path 候选集合 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `partial_pathlist` | partial Path 候选集合 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `ppilist` | ParamPathInfo 缓存 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `cheapest_total_path` | 全量最便宜 Path | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `cheapest_startup_path` | 启动最便宜 Path | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `cheapest_parameterized_paths` | 保留下来的参数化候选 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `baserestrictinfo` | base rel restriction | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `joininfo` | 与其它 rel 相关的 join clause | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |
| `lateral_relids` | 最小 lateral 参数需求 | `src/backend/optimizer/plan/planmain.c` 等主线文件消费或写入。 |

### 5.1. `reloptkind`

语义：搜索节点类别。

来源：base、join、upper、other member 等不同关系对象共用 RelOptInfo。

消费：许多字段只在特定 kind 下有效。

偏差后果：先看 kind 再读字段，能避免把 base-only 字段套到 upper rel。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.2. `relids`

语义：搜索节点身份。

来源：用 rangetable indexes 的 Bitmapset 表示包含哪些 base rel。

消费：join search、clause 归属和 joinrel hash 都依赖它。

偏差后果：它是 planner identity，不等于 SQL alias 字符串。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.3. `rows`

语义：relation-like 输出行数。

来源：base size、join size 或 upper stage row estimate 写入。

消费：Path 默认继承它，parameterized path 可覆盖。

偏差后果：一旦 rows 写错，下游 cost 仍会显得自洽。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.4. `reltarget`

语义：默认输出 PathTarget。

来源：描述该 RelOptInfo 的默认输出表达式、成本和 width。

消费：scan/join/upper path 通常指向它，projection path 可换 target。

偏差后果：宽 target 会把 sort/hash/parallel 成本向上游传播。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.5. `pathlist`

语义：完整 Path 候选集合。

来源：`add_path()` 维护，保存可直接生成完整结果的路径。

消费：`set_cheapest()` 从这里挑 cheapest 指针。

偏差后果：它是 planner-local，未选中 Path 不进入 executor。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.6. `partial_pathlist`

语义：partial Path 候选集合。

来源：只保存 worker 局部结果，需要 Gather 才能变完整。

消费：parallel planning 和 upper gather paths 消费它。

偏差后果：不能把 partial path 当最终 Plan 节点。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.7. `ppilist`

语义：ParamPathInfo 缓存。

来源：同一 required_outer 的行数和 clause 信息缓存到这里。

消费：parameterized path 复用它保证 row estimate 一致。

偏差后果：它只在 planner memory 中存在。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.8. `cheapest_total_path`

语义：全量最便宜 Path。

来源：`set_cheapest()` 写入。

消费：多数上层阶段默认从这里取输入。

偏差后果：它不是唯一重要指针；LIMIT 可能看 startup。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.9. `cheapest_startup_path`

语义：启动最便宜 Path。

来源：只有 consider_startup 使其有意义。

消费：LIMIT / EXISTS 等场景关注它。

偏差后果：如果 parent_rel 不关心 startup，相关候选会更早被剪。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.10. `cheapest_parameterized_paths`

语义：保留下来的参数化候选。

来源：用于 join search 继续组合。

消费：不同 required_outer 通常不能互相支配。

偏差后果：数量过多会放大搜索空间。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.11. `baserestrictinfo`

语义：base rel restriction。

来源：只属于 base relation。

消费：size estimate、scan qual cost 和 index clause 匹配都会读取。

偏差后果：joinrel 不应直接拿它当 join qual。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.12. `joininfo`

语义：与其它 rel 相关的 join clause。

来源：记录当前 rel 和外部 rel 的连接机会。

消费：joinrel 构造、parameterized path 和 join path 枚举都会用。

偏差后果：clause 迁移后还要看 RestrictInfo 的 relids。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

### 5.13. `lateral_relids`

语义：最小 lateral 参数需求。

来源：由 lateral 引用传播得到。

消费：限制 join order 和 base path required_outer。

偏差后果：它解释了某些看似可交换的 join 为什么不能交换。

阅读这个字段时，先问它是在候选生成前已经存在，还是在 cost 函数中写回，或是在 pathlist 剪枝后才被 cheapest 指针固化。

如果它参与 legality 判断，缺失候选通常不会留下“成本太高”的痕迹；如果它只参与 cost，EXPLAIN 中通常还能看到一个连续数字。

## 6. 主流程源码 walkthrough

下面按时间线走一条能在源码里跟到的主流程。

```text
setup_simple_rel_arrays()
  -> build_simple_rel()
  -> deconstruct_jointree()
  -> set_base_rel_sizes()
  -> set_base_rel_pathlists()
  -> make_one_rel()
  -> build_join_rel()
  -> find_join_rel()
  -> fetch_upper_rel()
  -> set_cheapest()
```

### 6.1. `setup_simple_rel_arrays()`

按 rangetable 大小创建 simple_rel_array / simple_rte_array，为 base RelOptInfo 留槽。

观察锚点：`reloptkind`。

### 6.2. `build_simple_rel()`

根据 RTE 创建 base RelOptInfo，初始化 relids、kind、target、pathlist、lateral 和 FDW/custom 扩展空间。

观察锚点：`relids`。

### 6.3. `deconstruct_jointree()`

遍历 jointree，把 join order 约束、outer join 信息和 qual 分类准备好。

观察锚点：`rows`。

### 6.4. `set_base_rel_sizes()`

为每个 base RelOptInfo 写 rows、width、baserestrictcost、consider_parallel。

观察锚点：`reltarget`。

### 6.5. `set_base_rel_pathlists()`

在同一个 base RelOptInfo 上添加 seqscan、index、bitmap、foreign、custom 等 Path。

观察锚点：`pathlist`。

### 6.6. `make_one_rel()`

从 base rels 出发，通过 join search 构造包含全部 relids 的 final joinrel。

观察锚点：`partial_pathlist`。

### 6.7. `build_join_rel()`

用左右输入 relids 的 union 创建 join RelOptInfo，并收集目标、rows、joininfo、lateral 信息。

观察锚点：`ppilist`。

### 6.8. `find_join_rel()`

按 relids 查找已有 joinrel，避免同一集合重复构造多个容器。

观察锚点：`cheapest_total_path`。

### 6.9. `fetch_upper_rel()`

为 upper planning 阶段按 UpperRelationKind 和 relids 创建或查找 RelOptInfo。

观察锚点：`cheapest_startup_path`。

### 6.10. `set_cheapest()`

在每个 RelOptInfo 的 pathlist 中固化 cheapest startup、total 和 parameterized paths。

观察锚点：`cheapest_parameterized_paths`。

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

### 9.1. RTE 不参与当前查询

simple_rel_array 可能有空槽；不能用数组下标存在判断 relation 一定活跃。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.2. dummy relation

约束证明为空时，RelOptInfo 可被标记为 dummy，下游只生成空结果路径。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.3. inheritance child

child RelOptInfo 可能是 OTHER_MEMBER_REL，并与 top parent 保持映射关系。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.4. upper rel relids 为空

FINAL 或某些 upper stage 不一定绑定具体 base relids。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.5. FDW/custom 插入路径

扩展可能在 hook 中改 pathlist 或 private 状态，诊断时要确认是否有非 core path。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

### 9.6. lateral 约束传播

join order 被 lateral_relids 限制时，候选不是被 cost 淘汰，而是搜索空间就不合法。

诊断时先区分这是候选没生成、候选生成后被剪枝，还是候选留下但成本输入偏离。

如果是候选没生成，优先读 legality 和字段依赖。

如果是成本输入偏离，优先回到 rows、width、页数、内存、worker 或 required_outer 的来源。

## 10. 成本、资源与跨模块传播

RelOptInfo 的关键是识别当前 relation-like 搜索节点和它持有的候选集合。

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
| 当前 reloptkind 是什么 | `setup_simple_rel_arrays()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| relids 是否匹配预期关系集合 | `build_simple_rel()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| rows 在哪个阶段写入 | `deconstruct_jointree()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| pathlist 和 partial_pathlist 是否都有内容 | `set_base_rel_sizes()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| ppilist 是否已有 required_outer | `set_base_rel_pathlists()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| cheapest 指针是否已 set | `make_one_rel()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| lateral_relids 是否限制 join order | `build_join_rel()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |
| upper rel kind 是否解释当前阶段 | `find_join_rel()` | 判断候选是缺失、被剪枝、成本输入偏差，还是执行期资源偏差。 |

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

- 现场记录 `当前 reloptkind 是什么` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `relids 是否匹配预期关系集合` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `rows 在哪个阶段写入` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `pathlist 和 partial_pathlist 是否都有内容` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `ppilist 是否已有 required_outer` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `cheapest 指针是否已 set` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `lateral_relids 是否限制 join order` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。
- 现场记录 `upper rel kind 是否解释当前阶段` 时，同时写下一个源码函数、一个字段值和一个 EXPLAIN 影子。

## 12. 常见误区

1. 把 RelOptInfo 当 RangeTblEntry 的简单包装；它是搜索状态容器。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

2. 直接读 base-only 字段解释 joinrel 或 upper rel。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

3. 认为 EXPLAIN 没显示的 path 没生成过；可能在 pathlist 中被剪掉。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

4. 忽略 relids 身份；同名 alias 和 relids 不是一回事。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

5. 把 cheapest_total_path 当唯一选择；startup、parameterized 和 pathkeys 都会让其它 Path 存活。

   更稳的读法是先回到候选生成和状态写入点，再解释最终 Plan。

## 13. 课堂实验

实验只改变一个主变量，其他变量保持稳定。

### 13.1. 看 base RelOptInfo

SQL：

```sql
在 `build_simple_rel()` 断点，执行两表查询。
打印 `relid`、`relids`、`rtekind`、`reltarget->width`。
```

预期观察：确认 Query RTE 如何变成 planner-local relation。

源码回看：源码入口在 `src/backend/optimizer/util/relnode.c`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.2. 看 joinrel 身份

SQL：

```sql
在 `build_join_rel()` 断点，执行三表 join。
打印左右输入 `relids` 和新 joinrel `relids`。
```

预期观察：观察 relids union 如何成为 join search key。

源码回看：源码入口在 `src/backend/optimizer/util/relnode.c` 与 `joinrels.c`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.3. 看 pathlist 增长

SQL：

```sql
在 `add_path()` 条件断点过滤某个 parent_rel->relids。
每次打印 path node type、rows、startup_cost、total_cost。
```

预期观察：看到同一 RelOptInfo 上不同候选如何共存。

源码回看：源码入口在 `src/backend/optimizer/util/pathnode.c`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

### 13.4. 看 upper rel

SQL：

```sql
对 GROUP BY / ORDER BY 查询，在 `fetch_upper_rel()` 断点打印 kind。
```

预期观察：确认 upper planning 也使用 RelOptInfo。

源码回看：源码入口在 `src/backend/optimizer/plan/planner.c`。

实验记录里保留 estimated rows、actual rows、startup cost、total cost 和本节主字段。

## 14. 源码练习

练习目标不是读完所有 helper，而是沿本节主状态走完整生命周期。

### 14.1. 跟 `setup_simple_rel_arrays()`

先用注释和调用者确认它的职责：按 rangetable 大小创建 simple_rel_array / simple_rte_array，为 base RelOptInfo 留槽。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.2. 跟 `build_simple_rel()`

先用注释和调用者确认它的职责：根据 RTE 创建 base RelOptInfo，初始化 relids、kind、target、pathlist、lateral 和 FDW/custom 扩展空间。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.3. 跟 `deconstruct_jointree()`

先用注释和调用者确认它的职责：遍历 jointree，把 join order 约束、outer join 信息和 qual 分类准备好。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.4. 跟 `set_base_rel_sizes()`

先用注释和调用者确认它的职责：为每个 base RelOptInfo 写 rows、width、baserestrictcost、consider_parallel。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.5. 跟 `set_base_rel_pathlists()`

先用注释和调用者确认它的职责：在同一个 base RelOptInfo 上添加 seqscan、index、bitmap、foreign、custom 等 Path。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.6. 跟 `make_one_rel()`

先用注释和调用者确认它的职责：从 base rels 出发，通过 join search 构造包含全部 relids 的 final joinrel。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.7. 跟 `build_join_rel()`

先用注释和调用者确认它的职责：用左右输入 relids 的 union 创建 join RelOptInfo，并收集目标、rows、joininfo、lateral 信息。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

### 14.8. 跟 `find_join_rel()`

先用注释和调用者确认它的职责：按 relids 查找已有 joinrel，避免同一集合重复构造多个容器。

再找它读了哪些输入状态。

继续找它写了哪些输出状态。

最后找下游哪个函数消费这些输出。

如果这一站没有改变主状态，把它标为旁路，避免把阅读时间耗在无关 helper。

## 15. 排查剧本

RelOptInfo 的关键是识别当前 relation-like 搜索节点和它持有的候选集合。

排查时按“外部现象 -> planner 字段 -> 源码入口 -> 结论类型”记录。

### 15.0. 现场最小记录

这四个字段足够建立第一轮判断：

| 字段 | 记录方式 |
| --- | --- |
| `reloptkind` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `relids` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `pathlist` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |
| `cheapest_total_path` | 写下断点位置、字段值、对应 EXPLAIN 影子，以及它改变的是候选生成、成本还是执行契约。 |

如果这四项都解释不了现象，再扩展到相邻模块；不要先把整个 planner 调用栈摊开。

| 序号 | 切入点 | 外部现象 | 源码入口 | 结论类型 |
| --- | --- | --- | --- | --- |
| 1 | 当前 reloptkind 是什么 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `setup_simple_rel_arrays()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 2 | relids 是否匹配预期关系集合 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `build_simple_rel()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 3 | rows 在哪个阶段写入 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `deconstruct_jointree()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 4 | pathlist 和 partial_pathlist 是否都有内容 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `set_base_rel_sizes()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 5 | ppilist 是否已有 required_outer | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `set_base_rel_pathlists()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 6 | cheapest 指针是否已 set | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `make_one_rel()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 7 | lateral_relids 是否限制 join order | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `build_join_rel()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |
| 8 | upper rel kind 是否解释当前阶段 | 计划节点、估算/实际 rows、loops 或节点附属指标围绕这个切入点变化。 | `find_join_rel()` | 候选缺失、剪枝、成本输入偏差或执行期资源偏差。 |

记录完成后，把结论压成一句可以复现的话。

例：在当前统计和 GUC 下，某个字段让某类 Path 没有生成，或生成后被 `add_path()` 丢弃，或保留但在执行期被资源边界放大。

## 16. 讨论题

1. 当 `当前 reloptkind 是什么` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

2. 当 `relids 是否匹配预期关系集合` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

3. 当 `rows 在哪个阶段写入` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

4. 当 `pathlist 和 partial_pathlist 是否都有内容` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

5. 当 `ppilist 是否已有 required_outer` 解释不了最终计划时，你会沿哪两个源码方向继续查？

   可以从候选生成、cost 输入、pathlist 剪枝、createplan 契约中任选两条路径展开。

## 17. 本节小结

| 项目 | 结论 |
| --- | --- |
| 唯一主问题 | 为什么 optimizer 要用 RelOptInfo 表示 base rel、joinrel、upper rel 和其它 relation-like 对象，而不是直接在 Query 的 RangeTblEntry 或 JoinExpr 上挂候选计划？ |
| 运行模型 | `query_planner()` 建立 simple_rel_array；`build_simple_rel()` 为 base RTE 创建 RelOptInfo；`make_one_rel()` 逐层构造 joinrel；`fetch_upper_rel()` 为 grouping、window、distinct、ordered、final 等 upper stage 提供容器；每个 RelOptInfo 用 `relids` / kind 标识搜索节点，用 `pathlist`、`partial_pathlist`、`ppilist` 和 cheapest 指针保存候选与剪枝结果。 |
| 生命周期 | planner-local 状态在一次规划中创建、剪枝、转交给 createplan，未选中候选不进入 executor。 |
| 诊断闭环 | 从 EXPLAIN 现象回到具体字段，再回到写入函数和剪枝函数。 |
| 可迁移规律 | 优化器不是选择“看起来高级”的节点，而是在当前语义、统计、资源和搜索边界下保留可比较候选。 |

读下一节时继续保持同一个动作：先找唯一主问题，再找状态生命周期，最后把 runtime 现象压回可迁移的 planner 判断。
