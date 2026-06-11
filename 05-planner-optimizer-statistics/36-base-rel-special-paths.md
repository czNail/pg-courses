# PostgreSQL append、FDW、tablesample 与特殊 base path 边界

## 课程定位

前置知识：已经理解普通 base table 的 size pass 和 pathlist pass，也知道 `RelOptInfo` 用同一种抽象承载不同扫描对象。

本节唯一主问题：

分区表、继承表、外表、TABLESAMPLE、subquery、function、CTE 和 custom scan 为什么能进入同一个 base rel pathlist 框架，而不需要为每类对象单独设计一套 optimizer？

核心矛盾：planner 需要把来源不同、执行约束不同的关系统一放进 scan/join 搜索；但每类对象又有自己的 size 来源、路径回调、并行安全、重复扫描和 executor contract。

学完后应能判断：看到 Append、MergeAppend、ForeignScan、SampleScan、SubqueryScan、FunctionScan 或 CustomScan 时，能判断它们在哪个阶段进入 pathlist，以及哪些边界由 core、FDW、采样方法或扩展负责。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前两节已经讲完普通 base rel 的 rows 写入和 path 候选生成。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节进入 join search，解释 base pathlist 如何被动态规划组合成 joinrel。

阅读时把每类 RTE 的 size 来源、path 回调和 executor contract 对齐；同一个 `RelOptInfo` 框架下面承载的是不同扫描对象。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`set_rel_size()` 按 RTE 和 relkind 分流；inheritance 或 partition parent 进入 `set_append_rel_size()`；FDW 进入 `set_foreign_size()` 并调用 `GetForeignRelSize()`；TABLESAMPLE 调用采样方法估算 pages/tuples；subquery、CTE、tuplestore、result 有些在 size 阶段建 path；`set_rel_pathlist()` 按同样分类进入 pathlist 阶段；append parent 通过 child path 组合路径；FDW 通过 `GetForeignPaths()` 添加 ForeignPath；TABLESAMPLE 生成 SampleScan，必要时 Materialize；hook 可加入 CustomPath。

1. `set_rel_size()` 按 RTE 和 relkind 分流
   分流点决定 rows/pages/width 从 core 统计、child 汇总、FDW 回调、采样方法还是子查询规划结果获得。

2. inheritance 或 partition parent 进入 `set_append_rel_size()`
   parent 自己通常不是实际扫描对象；这里要建立 child rel、变量映射、live child 集合和 parent rows 汇总边界。

3. FDW 进入 `set_foreign_size()` 并调用 `GetForeignRelSize()`
   core 会先给一个保守 size 估算，FDW 可以根据远端统计或 remote EXPLAIN 覆写 rows、width、cost 输入和 `fdw_private`。

4. TABLESAMPLE 调用采样方法估算 pages/tuples
   `SampleScanGetSampleSize()` 返回的是采样后会读多少页、吐多少 tuple，后续只为这个采样语义生成 SampleScan path。

5. subquery、CTE、tuplestore、result 有些在 size 阶段建 path
   这些 RTE 的 rows 往往来自子计划、函数返回集或 tuplestore 协议，不再按本地 heap 表的 pages 与 indexlist 推导。

6. `set_rel_pathlist()` 按同样分类进入 pathlist 阶段
   size 阶段选择的数据来源会在 pathlist 阶段继续约束可生成节点，例如 ForeignPath、AppendPath、SampleScanPath 或 FunctionScan。

7. append parent 通过 child path 组合路径
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. FDW 通过 `GetForeignPaths()` 添加 ForeignPath
   FDW path 的远端 SQL、startup/total cost、pathkeys 和 pushdown 能力由 wrapper 负责解释，core 只负责让它参与统一竞争。

9. TABLESAMPLE 生成 SampleScan，必要时 Materialize
   如果采样方法不能保证 repeatable scan，planner 可能加 Materialize，避免上层重复读取时得到不一致样本。

10. hook 可加入 CustomPath
   CustomPath 复用同一 `RelOptInfo` 与 `Path` contract，因此扩展路径必须声明自己的成本、并行性、参数化和 createplan 私有状态。

这条链路的重点是：特殊 base path 共享 pathlist 竞争模型，但 rows 来源、重复扫描语义、并行边界和 createplan 责任各不相同。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/allpaths.c` | `set_append_rel_size()`、`set_append_rel_pathlist()`、`set_foreign_size()`、`set_tablesample_rel_pathlist()`。 |
| 2 | `src/backend/optimizer/path/allpaths.c` | `add_paths_to_append_rel()` 组合 child path。 |
| 3 | `src/backend/optimizer/util/pathnode.c` | AppendPath、ForeignPath、SampleScanPath、MaterialPath、CustomPath 构造。 |
| 4 | `src/include/foreign/fdwapi.h` | FDW size/path 回调契约。 |
| 5 | `src/include/access/tsmapi.h` | TABLESAMPLE 估算与 repeatability 能力。 |
| 6 | `src/include/nodes/pathnodes.h` | `reloptkind`、`fdwroutine`、`top_parent_relids`。 |
| 7 | `src/backend/optimizer/prep/prepunion.c` | appendrel 与 child 映射准备。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：分区表显示 Append

源码解释：parent path 来自 live child

验证办法：查询单个分区范围。

Append 的诊断重点是 `live_childrels` 和每个 child 的 cheapest path；parent 只是组合者，不代表还有一张实体表被扫描。

### 现象：某些 child 被排除

源码解释：child 被 constraint exclusion 标成 dummy

验证办法：断点 `IS_DUMMY_REL()`。

child 消失要区分 partition pruning、constraint exclusion 和权限/继承展开边界；它们发生的阶段不同，残留状态也不同。

### 现象：ForeignScan rows 不像本地表

源码解释：FDW 覆盖 size 估算

验证办法：比较 use_remote_estimate。

ForeignScan rows 需要读 FDW 回调写入的 `fdw_private` 或远端估算策略；不能按本地 `pg_statistic` 规则硬套。

### 现象：SampleScan 被 Materialize

源码解释：采样方法不保证 repeatable across scans

验证办法：把 sample 放入 join。

Materialize 是执行契约补丁：上层要求重复扫描同一输入时，不能让非 repeatable sample 每次重新抽样。

### 现象：CTE 不参与 worker scan

源码解释：CTE tuplestore 不共享给 worker

验证办法：与 inline subquery 对照。

并行限制来自 RTE 的执行载体，而不是 SQL 文本复杂度；CTE tuplestore、临时状态和 worker 可见性要分开判断。

### 现象：CustomPath 出现

源码解释：通常来自 hook 或扩展

验证办法：打印 hook 前后 pathlist。

CustomPath 出现后要检查扩展填入的 `custom_private`、path target、parallel flags 和 createplan 回调，而不是只看 core allpaths 分支。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `reloptkind` | 区分 baserel、other member、joinrel、upper rel | rel 构造阶段 | path 和 join search |
| `append_rel_list` | parent/child RTE 映射 | prepunion | append size/path |
| `live_childrels` | 仍需扫描的 child rel | `set_append_rel_pathlist()` | `add_paths_to_append_rel()` |
| `consider_partitionwise_join` | 是否允许 partitionwise join | `set_append_rel_size()` | join search |
| `fdwroutine` | FDW 回调表 | plancat/FDW | size/path/createplan |
| `fdw_private` | FDW planner-private 状态 | FDW 回调 | create ForeignScan |
| `TableSampleClause` | 采样方法与参数 | parser/rewrite | TSM size/path |
| `top_parent_relids` | child 指向顶层 parent 的 relids | appendrel 构造 | parameterization 和 join restriction |

### `reloptkind`

语义：区分 baserel、other member、joinrel、upper rel

写入或持有者：rel 构造阶段

主要消费者：path 和 join search

诊断提示：分区 child 不能当普通 top-level rel

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `append_rel_list`

语义：parent/child RTE 映射

写入或持有者：prepunion

主要消费者：append size/path

诊断提示：变量替换依赖它

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `live_childrels`

语义：仍需扫描的 child rel

写入或持有者：`set_append_rel_pathlist()`

主要消费者：`add_paths_to_append_rel()`

诊断提示：缺分区先看它

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `consider_partitionwise_join`

语义：是否允许 partitionwise join

写入或持有者：`set_append_rel_size()`

主要消费者：join search

诊断提示：不是立即生成 join

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `fdwroutine`

语义：FDW 回调表

写入或持有者：plancat/FDW

主要消费者：size/path/createplan

诊断提示：远端能力集中在这里

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `fdw_private`

语义：FDW planner-private 状态

写入或持有者：FDW 回调

主要消费者：create ForeignScan

诊断提示：core 不解释其内部

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `TableSampleClause`

语义：采样方法与参数

写入或持有者：parser/rewrite

主要消费者：TSM size/path

诊断提示：repeatability 影响 Materialize

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `top_parent_relids`

语义：child 指向顶层 parent 的 relids

写入或持有者：appendrel 构造

主要消费者：parameterization 和 join restriction

诊断提示：避免混淆 child/parent 身份

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `set_rel_size()` 按 RTE 和 relkind 分流

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. inheritance 或 partition parent 进入 `set_append_rel_size()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. FDW 进入 `set_foreign_size()` 并调用 `GetForeignRelSize()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. TABLESAMPLE 调用采样方法估算 pages/tuples

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. subquery、CTE、tuplestore、result 有些在 size 阶段建 path

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. `set_rel_pathlist()` 按同样分类进入 pathlist 阶段

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. append parent 通过 child path 组合路径

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. FDW 通过 `GetForeignPaths()` 添加 ForeignPath

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. TABLESAMPLE 生成 SampleScan，必要时 Materialize

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. hook 可加入 CustomPath

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

## 7. 生命周期 / ownership / cleanup

- 对象由 planner memory context 持有，生命周期短于 executor。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 正常路径中状态逐层写入并被下游读取。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- `ERROR` 跳出时依赖 MemoryContext 释放，不逐个析构 Path 或 RelOptInfo。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 扩展或 FDW 可以保存 private 状态，但不能把 planner 指针泄漏到长生命周期对象。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 最终 Plan 只携带 executor 需要的稳定信息。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 诊断时不要把一次 planning 的指针当成跨语句状态。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 8. 正确性机制层次

- parent Vars 必须通过 `adjust_appendrel_attrs()` 转成 child Vars。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- constraint exclusion 只能排除可证明为空的 child。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW path 必须保持权限、collation 和远端语义。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 不可重复采样不能在多次扫描场景无保护执行。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parallel safety 对 FDW、sample、CTE、函数都有效。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- CustomPath 不能绕过 relids、targetlist、qual 和 parameterization 约束。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 所有 child dummy 时 parent dummy。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 某个 child parallel unsafe 时 whole appendrel 可能变 unsafe。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW 不提供 path 会导致没有可执行候选。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- TSM 不 repeatable 时加 Materialize。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- ONLY partitioned table 无 leaf 时 dummy。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hook 不存在时 core 仍处理普通特殊路径。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- append rows 是 live child rows 之和。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- append width 按 child rows 加权。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parallel append 依赖所有相关 child 安全。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW cost 可能来自本地估算或远端估算。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- SampleScan pages/tuples 由采样方法决定。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- Materialize 用资源换重复扫描稳定性。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 11. 观测与诊断入口

- 先确认当前 SQL 是否真的进入本节入口函数。

  诊断结论需要同时给出一个源码入口、一个状态字段和一个可复现实验。

- 打印 relids 或 path type，避免只看最终 Plan 节点。

  诊断结论需要同时给出一个源码入口、一个状态字段和一个可复现实验。

- 记录 GUC、统计状态、schema 和 SQL 版本。

  诊断结论需要同时给出一个源码入口、一个状态字段和一个可复现实验。

- 对同一 SQL 只改变一个输入因素做对照。

  诊断结论需要同时给出一个源码入口、一个状态字段和一个可复现实验。

- 把可见现象映射回一个字段写入点。

  诊断结论需要同时给出一个源码入口、一个状态字段和一个可复现实验。

- 把断点放在状态写入前后，而不是函数最外层。

  诊断结论需要同时给出一个源码入口、一个状态字段和一个可复现实验。

## 12. 课堂实验

### 实验 1

操作：构造最小表和索引，先跑 `EXPLAIN`，再加 `ANALYZE` 或 GUC 对照。

观察：记录 `EXPLAIN (ANALYZE, BUFFERS)`、planning time、关键 rows/cost 和断点输出。

结论：把观察结果映射回本节某一个字段写入点或 fallback 分支。

### 实验 2

操作：在主入口和 fallback 分支打断点，记录输入输出字段。

观察：记录 `EXPLAIN (ANALYZE, BUFFERS)`、planning time、关键 rows/cost 和断点输出。

结论：把观察结果映射回本节某一个字段写入点或 fallback 分支。

### 实验 3

操作：用一个能触发正路径的 SQL 和一个触发 fallback 的 SQL 对照。

观察：记录 `EXPLAIN (ANALYZE, BUFFERS)`、planning time、关键 rows/cost 和断点输出。

结论：把观察结果映射回本节某一个字段写入点或 fallback 分支。

### 实验 4

操作：改变统计信息或 predicate 写法，观察计划差异是否符合源码分支。

观察：记录 `EXPLAIN (ANALYZE, BUFFERS)`、planning time、关键 rows/cost 和断点输出。

结论：把观察结果映射回本节某一个字段写入点或 fallback 分支。

### 实验 5

操作：把最终 Plan 与中间 pathlist 或 joinrel 数量对照。

观察：记录 `EXPLAIN (ANALYZE, BUFFERS)`、planning time、关键 rows/cost 和断点输出。

结论：把观察结果映射回本节某一个字段写入点或 fallback 分支。

### 实验 6

操作：对照普通场景和边界场景，确认结论不是 workload 偶然。

观察：记录 `EXPLAIN (ANALYZE, BUFFERS)`、planning time、关键 rows/cost 和断点输出。

结论：把观察结果映射回本节某一个字段写入点或 fallback 分支。

## 13. 源码练习

1. 在主入口打印当前 relids、rows、pathlist 长度或 join level。

   练习目标是确认状态边界，不是背调用栈。

2. 在第一个 fallback 分支打印触发条件。

   练习目标是确认状态边界，不是背调用栈。

3. 在成本写入后记录 startup_cost、total_cost、rows 和 pathkeys。

   练习目标是确认状态边界，不是背调用栈。

4. 在 `set_cheapest()` 前后比较保留的 path。

   练习目标是确认状态边界，不是背调用栈。

5. 把同一 SQL 改写一次，解释源码分支为何变化。

   练习目标是确认状态边界，不是背调用栈。

## 14. 常见误区

- 把最终 `EXPLAIN` 节点当成完整搜索空间。

  修正方式：回到本节主链路，确认该结论对应哪个源码入口和哪个状态字段。

- 只看成本数值，不看 pathkeys、required_outer、relids 和 join type。

  修正方式：回到本节主链路，确认该结论对应哪个源码入口和哪个状态字段。

- 把 GUC 强制出的计划当成统计或语义问题已经修复。

  修正方式：回到本节主链路，确认该结论对应哪个源码入口和哪个状态字段。

- 忽略 fallback 的正确性目的，只用性能好坏评价它。

  修正方式：回到本节主链路，确认该结论对应哪个源码入口和哪个状态字段。

- 把当前版本的实现顺序误读成所有版本不变的系统本质。

  修正方式：回到本节主链路，确认该结论对应哪个源码入口和哪个状态字段。

- 只打印 raw field，不记录字段所属阶段和调用栈。

  修正方式：回到本节主链路，确认该结论对应哪个源码入口和哪个状态字段。

## 15. 讨论题

1. 分区表、继承表、外表、TABLESAMPLE、subquery、function、CTE 和 custom scan 为什么能进入同一个 base rel pathlist 框架，而不需要为每类对象单独设计一套 optimizer？

   回答时要落到源码入口、状态字段和可观察现象。

2. 如果这个机制的判断发生偏差，最先污染哪个后续阶段？

   回答时要落到源码入口、状态字段和可观察现象。

3. 哪些信息必须在 planner 阶段决定，哪些可以安全推迟到 executor？

   回答时要落到源码入口、状态字段和可观察现象。

4. 面对生产慢 SQL，如何用三个最小实验证明问题来自本节机制？

   回答时要落到源码入口、状态字段和可观察现象。

5. 哪些结论依赖当前源码版本，哪些是长期稳定的抽象边界？

   回答时要落到源码入口、状态字段和可观察现象。

## 16. 本节小结

- 本节围绕一个唯一主问题展开，而不是枚举函数。

- 关键状态必须结合生命周期、relids、path 属性和下游消费者理解。

- fallback 通常保证可规划性或正确性，不保证最优性。

- 最终 Plan 只展示胜者，不能自动展示完整搜索空间。

- 可迁移规律是：optimizer 用局部保守判断和全局候选竞争共同换取正确性与性能。

- 本节主问题：分区表、继承表、外表、TABLESAMPLE、subquery、function、CTE 和 custom scan 为什么能进入同一个 base rel pathlist 框架，而不需要为每类对象单独设计一套 optimizer？

## 17. 诊断案例切片

### 案例 1: append parent 不直接扫描数据

现象：分区表计划显示 Append，而不是扫描 parent heap。

源码入口：`set_append_rel_size()` 遍历 child，并把 live child rows 汇总到 parent。

状态变化：parent pages 有意保持 0，避免 `total_table_pages` 重复计数。

fallback 或边界：`set_append_rel_pathlist()` 再组合 child path。

诊断要点：诊断时同时看 parent rel 和 child rel。

最小实验：可用实验是查询单个分区范围。


复盘路径：

- 先把现象定位到一个可见 Plan 节点或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝或 fallback。

课堂追问：

- 这个案例里最早被写错或写保守的字段是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：这个案例说明 append parent 是组合节点，不是普通 heap scan。

### 案例 2: child 约束排除

现象：某些分区或继承 child 没有进入 Append。

源码入口：child 在 `set_append_rel_size()` 内可以被 `relation_excluded_by_constraints()` 标成 dummy。

状态变化：dummy child 不进入 live_childrels。

fallback 或边界：这一步发生在 path 组合之前。

诊断要点：诊断时打印每个 child 的 `IS_DUMMY_REL()`。

最小实验：可用实验是给分区约束写互斥 WHERE。


复盘路径：

- 先把现象定位到一个可见 Plan 节点或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝或 fallback。

课堂追问：

- 这个案例里最早被写错或写保守的字段是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：这个案例说明分区裁剪和约束证明会收缩 child 集合。

### 案例 3: FDW size 与 path 分离

现象：外表 estimated rows 和 path 成本来自 FDW 逻辑。

源码入口：`set_foreign_size()` 调用 `GetForeignRelSize()`。

状态变化：`set_foreign_pathlist()` 调用 `GetForeignPaths()`。

fallback 或边界：size 回调写估算，path 回调添加候选，两者职责不同。

诊断要点：诊断时不要只看本地 pg_statistic。

最小实验：可用实验是切换远端估算选项。


复盘路径：

- 先把现象定位到一个可见 Plan 节点或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝或 fallback。

课堂追问：

- 这个案例里最早被写错或写保守的字段是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：这个案例说明 FDW 通过 callback 接入 core path 框架。

### 案例 4: TABLESAMPLE 的 repeatability

现象：SampleScan 在 join 中被 Materialize 包裹。

源码入口：TSM 提供 sample size 估算和 repeatable across scans 能力。

状态变化：如果采样方法不保证重复扫描稳定，planner 可能用 Materialize 降低风险。

fallback 或边界：这不是普通成本优化，而是执行语义保护。

诊断要点：诊断时查 `repeatable_across_scans`。

最小实验：可用实验是把 sample 放到 nestloop inner 位置。


复盘路径：

- 先把现象定位到一个可见 Plan 节点或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝或 fallback。

课堂追问：

- 这个案例里最早被写错或写保守的字段是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：这个案例说明特殊扫描方法可以改变 path 形态。

### 案例 5: CTE 与 parallel safety

现象：CTE scan 没有进入 worker。

源码入口：`set_rel_consider_parallel()` 对 CTE tuplestore 保守返回。

状态变化：原因是 tuplestore 不在 worker 间共享，且 CTE 生产计划必须执行一次。

fallback 或边界：即使数据量很大，也不能只凭成本下放 worker。

诊断要点：诊断时区分 inline subquery 和 materialized CTE。

最小实验：可用实验是改写同一查询为 subquery。


复盘路径：

- 先把现象定位到一个可见 Plan 节点或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝或 fallback。

课堂追问：

- 这个案例里最早被写错或写保守的字段是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：这个案例说明资源共享边界会限制 parallel path。

### 案例 6: CustomPath 的接入位置

现象：扩展希望为特殊存储添加扫描路径。

源码入口：常见入口是 `set_rel_pathlist_hook`，通过 `add_path()` 或 `add_partial_path()` 接入。

状态变化：CustomPath 仍要带正确 relids、target、cost 和 parameterization。

fallback 或边界：createplan 阶段还需要对应 CustomScanMethods。

诊断要点：诊断时检查 hook 前后 pathlist。

最小实验：可用实验是最小扩展添加一个始终高成本的 CustomPath。


复盘路径：

- 先把现象定位到一个可见 Plan 节点或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝或 fallback。

课堂追问：

- 这个案例里最早被写错或写保守的字段是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：这个案例说明自定义扫描也必须服从 core optimizer contract。
