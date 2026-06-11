# PostgreSQL base relation size 估算与叶子 rows 传播

## 课程定位

前置知识：已经理解 `RelOptInfo`、`RestrictInfo`、`pg_class.reltuples`、`pg_statistic`、选择率函数和基本成本单位。

本节唯一主问题：

为什么 PostgreSQL 必须先为每个 base relation 写入 rows、width、restriction cost 和并行安全边界，再开始生成具体 scan path？

核心矛盾：base rows 是后续 scan、join、sort、hash、parallel worker 选择的第一层事实；但这个事实只能由统计信息、约束推理、AM 估算和保守 fallback 共同拼出来，既不能等 executor 才知道，也不能假装它是精确真相。

学完后应能判断：从 `EXPLAIN` 叶子节点 rows 反推问题更可能来自统计信息、predicate 归属、partial index predicate、constraint exclusion，还是 size pass 的保守 fallback。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面几节已经讲过 rows、selectivity、cost、`RelOptInfo` 和 `Path` 的抽象边界。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节会在 rows、pages、width 和 parallel 标志已经就绪的前提下，解释 base relation 为什么要生成多个 scan path。

阅读时把 `rows`、`pages`、`width`、`predOK` 和 `consider_parallel` 当成同一批叶子状态看；任何一个字段偏差都会在后续 path 和 join 成本里被放大。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`make_one_rel()` 先调用 `set_base_rel_consider_startup()`；`set_base_rel_sizes()` 扫描 `root->simple_rel_array`；`set_rel_consider_parallel()` 在 size 之前判断 worker 安全性；`set_rel_size()` 按 RTE 类型分派；普通表进入 `set_plain_rel_size()`；`check_index_predicates()` 写 partial index 的 `predOK`；`set_baserel_size_estimates()` 写 rows、width 和 restriction cost；`make_one_rel()` 汇总 `root->total_table_pages`；pathlist 和 join search 读取这些字段。

1. `make_one_rel()` 先调用 `set_base_rel_consider_startup()`
   这里只标记少数 semi/anti join RHS base rel 的 `consider_param_startup`，提示后续参数化 nested loop 可以重视 fast-start path。

2. `set_base_rel_sizes()` 扫描 `root->simple_rel_array`
   这个 pass 跳过空槽和非 top-level baserel，只为真正参与 scan/join 搜索的叶子关系写 size 状态。

3. `set_rel_consider_parallel()` 在 size 之前判断 worker 安全性
   `consider_parallel` 必须先定下来，因为 append parent、sample、subquery 等 size 逻辑可能立即消费或收缩这个并行边界。

4. `set_rel_size()` 按 RTE 类型分派
   普通表、append parent、foreign table、TABLESAMPLE、subquery 和 function RTE 在这里分流，各自决定 rows/pages 的来源。

5. 普通表进入 `set_plain_rel_size()`
   普通 heap 表先处理 partial index predicate，再把 catalog 统计、base restriction 和列宽估算压进 `RelOptInfo`。

6. `check_index_predicates()` 写 partial index 的 `predOK`
   `predOK` 是后续 index path 是否可用的正确性门槛，不是成本偏好；false 的 partial index 不能靠调低成本强行使用。

7. `set_baserel_size_estimates()` 写 rows、width 和 restriction cost
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. `make_one_rel()` 汇总 `root->total_table_pages`
   这里累加非 dummy simple rel 的 `pages`，给 parallel worker 数、I/O 成本和全局 scan 规模提供粗粒度输入。

9. pathlist 和 join search 读取这些字段
   后续 `create_seqscan_path()`、`create_index_paths()` 和 join rows 估算不再回头查 SQL 文本，而是把这些字段当成叶子事实。

这条链路的重点是：base size pass 不选择 scan 算法，它先把每个叶子 relation 变成后续搜索可以消费的数字边界和正确性标志。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/allpaths.c` | `make_one_rel()`、`set_base_rel_sizes()`、`set_rel_size()`、`set_plain_rel_size()` 是主链路。 |
| 2 | `src/backend/optimizer/path/costsize.c` | `set_baserel_size_estimates()`、`set_rel_width()` 写 rows、width 和 qual cost。 |
| 3 | `src/backend/optimizer/path/clausesel.c` | `clauselist_selectivity()` 聚合 base restriction 选择率。 |
| 4 | `src/backend/optimizer/path/indxpath.c` | `check_index_predicates()` 判断 partial index predicate。 |
| 5 | `src/backend/optimizer/util/plancat.c` | `relation_excluded_by_constraints()` 使用约束排除空关系。 |
| 6 | `src/include/nodes/pathnodes.h` | `RelOptInfo`、`IndexOptInfo`、`PathTarget` 字段语义。 |
| 7 | `src/include/optimizer/cost.h` | size 与 cost 函数声明。 |
| 8 | `src/backend/optimizer/plan/planmain.c` | `query_planner()` 提供 scan/join 搜索上层入口。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：叶子 `Seq Scan` estimated rows 与 actual rows 差异很大

源码解释：看 `baserestrictinfo` 和 `clauselist_selectivity()` 是否拿到有效统计

验证办法：ANALYZE 前后比较同一 SQL。

如果 `rel->tuples` 接近真实表规模而 `rel->rows` 偏离明显，优先沿 `baserestrictinfo`、单列统计和扩展统计回查。

### 现象：partial index 没有出现

源码解释：`predOK` 为 false 时 `create_index_paths()` 会跳过该 index

验证办法：断点 `check_index_predicates()`。

这里要同时打印 partial index 的 `indpred` 和证明用 clause list；索引缺失首先是 predicate 蕴含失败，不是 scan cost 输了。

### 现象：CHECK 约束让 rel 变空

源码解释：`relation_excluded_by_constraints()` 成功后设置 dummy path

验证办法：构造互斥 CHECK 与 WHERE。

dummy rel 会让后续 pathlist 与 join search 看到一个已经被裁掉的叶子，诊断时应从 constraint exclusion 而不是 join 枚举开始。

### 现象：临时表没有 parallel path

源码解释：`set_rel_consider_parallel()` 对临时表保持 false

验证办法：普通表与临时表对照。

并行缺失要先确认 `root->glob->parallelModeOK` 和 `rel->consider_parallel`，否则 pathlist 阶段根本不会生成 partial scan。

### 现象：列宽估计影响 hash/sort

源码解释：`set_rel_width()` 写 `reltarget->width`

验证办法：改变投影列集合。

宽度错误会继续进入 hash、sort、materialize 和 FDW cost，所以要把 rows 偏差与 width 偏差分开判断。

### 现象：FDW rows 不像本地表

源码解释：`GetForeignRelSize()` 可覆盖 core 默认估算

验证办法：比较 postgres_fdw 远端估算选项。

外表 rows 要看 FDW 是否覆写 `rows`、`tuples`、`fdw_private`，core 只保证回调前后能挂接到统一 `RelOptInfo`。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `RelOptInfo.rows` | base restriction 后的输出行数估计 | `set_baserel_size_estimates()` 或具体 RTE size 函数 | scan cost、join size、parallel rows、upper planning |
| `RelOptInfo.tuples` | 关系级 tuple 总量 | relation metadata、统计或 FDW 回调 | rows、distinct/group 估算 |
| `RelOptInfo.pages` | 页数估计 | relation size 估算或 child 汇总逻辑 | seqscan cost、bitmap heap cost、parallel worker |
| `reltarget->width` | 当前输出目标平均宽度 | `set_rel_width()` | hash、sort、materialize、FDW cost |
| `baserestrictinfo` | 已归属到 base rel 的过滤条件 | jointree 分解和 qual 分发 | selectivity、index match、tidscan |
| `IndexOptInfo.predOK` | partial index predicate 是否被蕴含 | `check_index_predicates()` | index path 生成 |
| `consider_parallel` | base rel 是否允许 worker 扫描 | `set_rel_consider_parallel()` | partial path 生成 |
| dummy rel pathlist | 已证明为空的 rel 标记 | `set_dummy_rel_pathlist()` | pathlist 阶段和 join search |

### `RelOptInfo.rows`

语义：base restriction 后的输出行数估计

写入或持有者：`set_baserel_size_estimates()` 或具体 RTE size 函数

主要消费者：scan cost、join size、parallel rows、upper planning

诊断提示：叶子 rows 先从这里解释

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `RelOptInfo.tuples`

语义：关系级 tuple 总量

写入或持有者：relation metadata、统计或 FDW 回调

主要消费者：rows、distinct/group 估算

诊断提示：它不是 restriction 后 rows

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `RelOptInfo.pages`

语义：页数估计

写入或持有者：relation size 估算或 child 汇总逻辑

主要消费者：seqscan cost、bitmap heap cost、parallel worker

诊断提示：append parent pages 有意为 0

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `reltarget->width`

语义：当前输出目标平均宽度

写入或持有者：`set_rel_width()`

主要消费者：hash、sort、materialize、FDW cost

诊断提示：宽投影会抬高后续成本

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `baserestrictinfo`

语义：已归属到 base rel 的过滤条件

写入或持有者：jointree 分解和 qual 分发

主要消费者：selectivity、index match、tidscan

诊断提示：join qual 不应提前算入 base rows

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `IndexOptInfo.predOK`

语义：partial index predicate 是否被蕴含

写入或持有者：`check_index_predicates()`

主要消费者：index path 生成

诊断提示：这是正确性门槛

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `consider_parallel`

语义：base rel 是否允许 worker 扫描

写入或持有者：`set_rel_consider_parallel()`

主要消费者：partial path 生成

诊断提示：没有 Gather 时先查它

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### dummy rel pathlist

语义：已证明为空的 rel 标记

写入或持有者：`set_dummy_rel_pathlist()`

主要消费者：pathlist 阶段和 join search

诊断提示：dummy 不是普通低成本 path

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `make_one_rel()` 先调用 `set_base_rel_consider_startup()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. `set_base_rel_sizes()` 扫描 `root->simple_rel_array`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. `set_rel_consider_parallel()` 在 size 之前判断 worker 安全性

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. `set_rel_size()` 按 RTE 类型分派

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. 普通表进入 `set_plain_rel_size()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. `check_index_predicates()` 写 partial index 的 `predOK`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. `set_baserel_size_estimates()` 写 rows、width 和 restriction cost

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. `make_one_rel()` 汇总 `root->total_table_pages`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. pathlist 和 join search 读取这些字段

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

- 约束排除只能删除可证明为空的 rel。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- partial index predicate 不能靠成本猜测，必须由查询条件蕴含。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parallel-unsafe qual、targetlist、临时表和 CTE 不能下放 worker。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 非 dummy rel 必须有正 rows，避免后续 cost 和除法边界失真。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- append child 的变量替换和约束判断必须先于 parent 汇总。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- planner 的 rows 是估算，最终 tuple visibility 仍由 executor 和 MVCC 判断。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 无统计时使用默认选择率和类型平均宽度。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 约束无法证明矛盾时保留 rel。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW 不修正 rows 时 core 仍会 clamp rows。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parallel 不安全时保留 serial path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- partial index predicate 不成立时仍保留其它 scan path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 所有 child dummy 时 parent appendrel 也变成 dummy。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- rows 误差会放大 nested loop 次数和 hash/sort 输入量。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- pages 误差影响 seqscan、bitmap heap 和 parallel worker。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- width 误差影响 hash footprint、sort run 和 materialize。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- restriction cost 会进入 scan path 成本。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- `total_table_pages` 给并行判断一个全局背景。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 单独的 size pass 用少量 planning time 换取统一输入。

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

1. 为什么 PostgreSQL 必须先为每个 base relation 写入 rows、width、restriction cost 和并行安全边界，再开始生成具体 scan path？

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

- 本节主问题：为什么 PostgreSQL 必须先为每个 base relation 写入 rows、width、restriction cost 和并行安全边界，再开始生成具体 scan path？

## 17. 诊断案例切片

### 案例 1: 叶子 rows 从哪里开始偏离

现象：`EXPLAIN` 中 base scan estimated rows 明显小于 actual rows。

源码入口：从 `set_base_rel_sizes()` 进入 `set_plain_rel_size()`，再到 `set_baserel_size_estimates()`。

状态变化：重点打印 `rel->tuples`、`rel->rows`、`baserestrictinfo` 长度和 `reltarget->width`。

fallback 或边界：如果 `tuples` 已经离谱，先查 `pg_class.reltuples` 和 ANALYZE 时机。

诊断要点：如果 `tuples` 正常但 `rows` 偏离，问题多半在 restriction selectivity。

最小实验：可用实验是删除一个 predicate 或刷新统计，观察 rows 是否按比例改变。


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

可迁移结论：这个案例说明 base size 不是最终成本比较，而是后面所有 path 的输入事实。

### 案例 2: partial index 为什么影响 size pass

现象：查询明明有 partial unique index，但后续路径没有使用它。

源码入口：先不要跳到 `create_index_paths()`；普通表 size 阶段已经调用 `check_index_predicates()`。

状态变化：`IndexOptInfo.predOK` 为 false 时，后续 index path 根本不会考虑该 partial index。

fallback 或边界：断点应放在 `set_plain_rel_size()` 内部的 predicate 检查之后。

诊断要点：如果 predicate 写法不能被蕴含证明接受，成本 GUC 不会修复它。

最小实验：可用实验是把 WHERE 改写成直接蕴含 partial predicate 的形式。


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

可迁移结论：这个案例说明可用性门槛要早于候选路径生成。

### 案例 3: constraint exclusion 与 dummy rel

现象：CHECK 约束证明某个 rel 不可能产生行，计划里看不到普通扫描。

源码入口：`relation_excluded_by_constraints()` 成功后直接调用 `set_dummy_rel_pathlist()`。

状态变化：dummy rel 不是 cost 很低的 path，而是一个语义标记。

fallback 或边界：后续 `set_rel_pathlist()` 看到 `IS_DUMMY_REL()` 就不会再生成 seqscan 或 indexscan。

诊断要点：诊断时打印 rel 是否已经 dummy，而不是只看 pathlist 为空。

最小实验：可用实验是对同一表切换互斥和非互斥 WHERE。


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

可迁移结论：这个案例说明 correctness 证明可以先于成本模型收缩搜索空间。

### 案例 4: parallel 标志为什么在 size 前写入

现象：一个大表没有 `Gather`，但 rows 和 cost 看起来支持并行。

源码入口：`set_rel_consider_parallel()` 在 `set_rel_size()` 之前执行。

状态变化：临时表、CTE、parallel-unsafe qual、parallel-unsafe targetlist 都会让 `consider_parallel` 保持 false。

fallback 或边界：appendrel 和部分 RTE 会在 size/path 阶段立即消费这个标志。

诊断要点：断点要放在被拒绝的 return 前，记录 RTE 类型和 unsafe 表达式。

最小实验：可用实验是在 WHERE 中加入 parallel unsafe 函数再移除。


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

可迁移结论：这个案例说明并行首先是安全边界，其次才是成本问题。

### 案例 5: width 误差如何传到上层

现象：叶子 rows 还可以，但 Hash 或 Sort 成本异常偏高。

源码入口：从 `set_rel_width()` 看 `reltarget->width` 和 `attr_widths`。

状态变化：宽度估算会影响 hash table footprint、sort run 数和 materialize 成本。

fallback 或边界：投影列和表达式类型会改变 width，即使过滤条件完全不变。

诊断要点：诊断时把 SELECT list 当作成本输入，而不是只看 WHERE。

最小实验：可用实验是把宽文本列从投影中移除。


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

可迁移结论：这个案例说明 base size pass 同时写 rows 和 width，两者都会向上传播。

### 案例 6: FDW rows 的 ownership 边界

现象：外表的 estimated rows 与本地统计经验不一致。

源码入口：`set_foreign_size()` 先做 core 默认估算，再调用 FDW 的 `GetForeignRelSize()`。

状态变化：FDW 可以覆盖 rows、width 和 private 状态，但 core 仍会 clamp rows。

fallback 或边界：诊断时要区分本地 catalog 统计、远端估算选项和 FDW 自己的成本模型。

诊断要点：可用实验是切换 postgres_fdw 的远端估算设置。

最小实验：如果 FDW 不修正 rows，`rel->tuples` 还会被 rows 下界保护。


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

可迁移结论：这个案例说明特殊 AM/FDW 仍必须把结果落回 `RelOptInfo`。
