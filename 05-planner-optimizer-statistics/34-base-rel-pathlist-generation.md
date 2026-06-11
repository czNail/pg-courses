# PostgreSQL base relation pathlist 生成与候选扫描竞争

## 课程定位

前置知识：已经理解 base size pass 写入 rows、pages、width、`consider_parallel`，也知道 `Path` 是 planner 选择执行形态之前的候选表示。

本节唯一主问题：

一个 base relation 为什么要同时生成 seqscan、tidscan、indexscan、bitmap、parallel scan 等多个 Path，再交给 `add_path()` 竞争，而不是在这里直接选一个看起来最便宜的扫描方式？

核心矛盾：扫描方式的优劣不只取决于当前 rel 的局部成本，还取决于排序、参数化、并行、启动成本和后续 join 需求；过早只留一个 winner 会让后续阶段失去必要选择。

学完后应能判断：从叶子节点计划判断候选是没有生成、生成后被 `add_path()` 支配剪掉，还是保留到最后但输给其它 path。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节已经把 base relation 的 rows、pages、width 和 parallel safety 写入 `RelOptInfo`。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节会把普通 pathlist 中最复杂的 index path 单独拿出来讲。

阅读时盯住 `rel->pathlist`、`rel->partial_pathlist` 和 `set_cheapest()` 前后的变化；最终 EXPLAIN 只显示胜者，不显示被生成又被剪掉的候选。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`make_one_rel()` 在 size pass 后调用 `set_base_rel_pathlists()`；`set_base_rel_pathlists()` 只处理 `RELOPT_BASEREL`；`set_rel_pathlist()` 跳过 dummy rel 并按 RTE 类型分派；普通表进入 `set_plain_rel_pathlist()`；`create_tidscan_paths()` 可因 `CurrentOfExpr` 强制返回；`create_seqscan_path()` 添加基本 fallback；`create_plain_partial_paths()` 添加 partial seqscan；`create_index_paths()` 添加 index、bitmap、parameterized index path；`set_rel_pathlist_hook` 可修改 pathlist；`generate_useful_gather_paths()` 包装 partial path；`set_cheapest()` 记录 cheapest 指针。

1. `make_one_rel()` 在 size pass 后调用 `set_base_rel_pathlists()`
   进入这里时每个 base rel 已经有 rows、pages、width 和 parallel 边界，path 生成只是在这些状态上展开物理访问候选。

2. `set_base_rel_pathlists()` 只处理 `RELOPT_BASEREL`
   parent appendrel、other member rel 和 dummy rel 不按同一入口重复处理，避免同一个扫描对象被多次生成 path。

3. `set_rel_pathlist()` 跳过 dummy rel 并按 RTE 类型分派
   dummy rel 直接保留空结果语义；普通表、append、foreign、sample 等对象分别进入自己的 pathlist 分支。

4. 普通表进入 `set_plain_rel_pathlist()`
   这里先把 `lateral_relids` 转成 `required_outer`，再按 TID、seqscan、partial seqscan、index path 的顺序添加候选。

5. `create_tidscan_paths()` 可因 `CurrentOfExpr` 强制返回
   `WHERE CURRENT OF` 是执行语义约束，命中后普通 seqscan/indexscan 不再竞争，因为 executor 需要定位游标当前 tuple。

6. `create_seqscan_path()` 添加基本 fallback
   seqscan 是普通表的基础完整 path，带上 `required_outer`、rows 和 cost，保证没有可用索引时仍有可执行方案。

7. `create_plain_partial_paths()` 添加 partial seqscan
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. `create_index_paths()` 添加 index、bitmap、parameterized index path
   index path 不只竞争过滤成本，还可能保留 pathkeys、bitmap 合并能力和外部参数依赖，给 join 阶段继续选择。

9. `set_rel_pathlist_hook` 可修改 pathlist
   hook 看到的是 core 已经生成的候选集合，扩展可以追加 CustomPath，也可以改变一个 relation 暴露给 join search 的物理能力。

10. `generate_useful_gather_paths()` 包装 partial path
   partial path 只是 worker 内部扫描形态，Gather/Gather Merge 才把它转换成上层可消费的完整 path。

11. `set_cheapest()` 记录 cheapest 指针
   `cheapest_startup_path`、`cheapest_total_path` 和 parameterized cheapest 列表共同决定后续 join search 的输入，而不是最终计划的唯一答案。

这条链路的重点是：base pathlist 阶段不是替用户选一个扫描节点，而是保留能影响 join、排序、LIMIT 和并行的几类访问形态。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/allpaths.c` | `set_base_rel_pathlists()`、`set_rel_pathlist()`、`set_plain_rel_pathlist()`。 |
| 2 | `src/backend/optimizer/path/tidpath.c` | `create_tidscan_paths()` 处理 TID 与 `CurrentOfExpr`。 |
| 3 | `src/backend/optimizer/path/indxpath.c` | `create_index_paths()` 添加 index 和 bitmap 候选。 |
| 4 | `src/backend/optimizer/util/pathnode.c` | Path 构造器、`add_path()`、`set_cheapest()`。 |
| 5 | `src/include/nodes/pathnodes.h` | `Path`、`ParamPathInfo`、`pathlist`、`partial_pathlist`。 |
| 6 | `src/include/optimizer/pathnode.h` | Path 构造和 pathlist 操作声明。 |
| 7 | `src/include/optimizer/paths.h` | `set_rel_pathlist_hook` 和路径搜索入口。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：最终是 `Seq Scan`

源码解释：只说明 seqscan 胜出，不说明 index path 没生成

验证办法：在 `add_path()` 打断点。

如果 `add_path()` 曾接收 index path，最终 `Seq Scan` 更可能是成本、pathkeys 或 parameterization 竞争结果，而不是索引匹配失败。

### 现象：有索引但没有 `Index Scan`

源码解释：先看 `create_index_paths()` 是否被调用

验证办法：打印 `rel->indexlist`。

`rel->indexlist` 为空说明 catalog/plancat 层没有暴露索引；非空还要继续看 `predOK`、clause match 和 `add_path()` 剪枝。

### 现象：出现 `Bitmap Heap Scan`

源码解释：说明 index bitmap 参与了访问

验证办法：构造多列 AND 条件。

Bitmap path 的出现说明 planner 选择先组合索引命中再回表，它不等价于普通 Index Scan 失败，而是另一类候选胜出。

### 现象：`WHERE CURRENT OF` 只用 TID scan

源码解释：`create_tidscan_paths()` 可以 early return

验证办法：用游标场景验证。

这个 early return 会阻断其它普通 scan path，诊断时不应把它解释为 `enable_seqscan` 或索引成本异常。

### 现象：parallel seqscan 没出现

源码解释：可能是 `consider_parallel` false 或 worker 数为 0

验证办法：调大表和并行 GUC。

如果 `partial_pathlist` 为空，先看 `consider_parallel`、`required_outer` 和 `compute_parallel_worker()`，再看 Gather 是否被生成。

### 现象：候选生成后不可见

源码解释：可能被 `add_path()` 支配剪枝

验证办法：打印被删除 path 的 cost、pathkeys、required_outer。

被剪掉的 path 仍是一次真实候选尝试；要比较 `startup_cost`、`total_cost`、`pathkeys`、`required_outer` 才能解释为什么不可见。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `rel->pathlist` | 普通完整路径候选集合 | `add_path()` | join search |
| `rel->partial_pathlist` | worker 内可执行候选 | `add_partial_path()` | Gather/parallel append |
| `Path.startup_cost` | 取第一批 tuple 前成本 | 各 Path 构造器 | LIMIT、EXISTS、semi join |
| `Path.total_cost` | 完整输出成本 | 各 Path 构造器 | 普通 plan 竞争 |
| `Path.pathkeys` | 输出顺序 | index/merge/sort path | ORDER BY、merge join、grouping |
| `Path.param_info` | 需要外部 rel 参数 | lateral 或 parameterized index path | nestloop inner |
| `set_rel_pathlist_hook` | 扩展修改 base pathlist 的入口 | core path 后调用 | CustomPath、partial path |
| `cheapest_total_path` | 完整输出最便宜 path | `set_cheapest()` | join search |

### `rel->pathlist`

语义：普通完整路径候选集合

写入或持有者：`add_path()`

主要消费者：join search

诊断提示：最终 plan 只显示胜者

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `rel->partial_pathlist`

语义：worker 内可执行候选

写入或持有者：`add_partial_path()`

主要消费者：Gather/parallel append

诊断提示：partial 不是最终节点

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Path.startup_cost`

语义：取第一批 tuple 前成本

写入或持有者：各 Path 构造器

主要消费者：LIMIT、EXISTS、semi join

诊断提示：不能只看 total cost

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Path.total_cost`

语义：完整输出成本

写入或持有者：各 Path 构造器

主要消费者：普通 plan 竞争

诊断提示：单位不是毫秒

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Path.pathkeys`

语义：输出顺序

写入或持有者：index/merge/sort path

主要消费者：ORDER BY、merge join、grouping

诊断提示：局部贵的有序 path 可能保留

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Path.param_info`

语义：需要外部 rel 参数

写入或持有者：lateral 或 parameterized index path

主要消费者：nestloop inner

诊断提示：不能随意放在 outer side

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `set_rel_pathlist_hook`

语义：扩展修改 base pathlist 的入口

写入或持有者：core path 后调用

主要消费者：CustomPath、partial path

诊断提示：hook 顺序影响 Gather

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `cheapest_total_path`

语义：完整输出最便宜 path

写入或持有者：`set_cheapest()`

主要消费者：join search

诊断提示：不是 pathlist 全部

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `make_one_rel()` 在 size pass 后调用 `set_base_rel_pathlists()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. `set_base_rel_pathlists()` 只处理 `RELOPT_BASEREL`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. `set_rel_pathlist()` 跳过 dummy rel 并按 RTE 类型分派

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. 普通表进入 `set_plain_rel_pathlist()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. `create_tidscan_paths()` 可因 `CurrentOfExpr` 强制返回

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. `create_seqscan_path()` 添加基本 fallback

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. `create_plain_partial_paths()` 添加 partial seqscan

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. `create_index_paths()` 添加 index、bitmap、parameterized index path

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. `set_rel_pathlist_hook` 可修改 pathlist

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. `generate_useful_gather_paths()` 包装 partial path

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 11. `set_cheapest()` 记录 cheapest 指针

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

- 普通表至少有 seqscan fallback，除非 dummy 或强制 TID scan。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- `CurrentOfExpr` 场景不能替换成任意扫描。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parameterized path 只能在参数可用的位置执行。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- partial path 必须能在 worker 中安全执行。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- pathkeys 是上层可消费的物理属性。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 扩展添加 path 仍必须满足 executor contract。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 无索引时 seqscan 兜底。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 索引不匹配时保留其它 path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- worker 数为 0 时不生成 partial path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- `CurrentOfExpr` 强制 TID scan。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hook 不存在时 core pathlist 独立完整。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- sample 不可重复时可能包 Materialize。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- seqscan 受 pages、rows、qual cost 影响。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- indexscan 受选择率、相关性、随机 I/O 和排序收益影响。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- bitmap 多用于组合 predicate。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- partial path 降低扫描成本但增加 Gather 成本。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- startup cost 影响 LIMIT 类查询。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- pathkeys 可抵消上层 Sort 成本。

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

1. 一个 base relation 为什么要同时生成 seqscan、tidscan、indexscan、bitmap、parallel scan 等多个 Path，再交给 `add_path()` 竞争，而不是在这里直接选一个看起来最便宜的扫描方式？

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

- 本节主问题：一个 base relation 为什么要同时生成 seqscan、tidscan、indexscan、bitmap、parallel scan 等多个 Path，再交给 `add_path()` 竞争，而不是在这里直接选一个看起来最便宜的扫描方式？

## 17. 诊断案例切片

### 案例 1: Seq Scan 胜出不等于 index path 缺失

现象：最终计划是 `Seq Scan`，但表上存在可用索引。

源码入口：先在 `set_plain_rel_pathlist()` 入口确认普通表路径分支被执行。

状态变化：再在 `create_index_paths()` 和 `add_path()` 处分别观察候选生成和剪枝。

fallback 或边界：如果 index path 生成后被支配，最终 EXPLAIN 不会显示它。

诊断要点：GUC 对照只能帮助暴露候选，不等于修复 rows 或成本。

最小实验：可用实验是临时关闭 seqscan，然后回到 `add_path()` 比较原始成本。


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

可迁移结论：这个案例说明最终节点不是完整搜索空间。

### 案例 2: TID scan 的 early return

现象：游标相关 SQL 只生成 TID scan，没有普通 seqscan 或 indexscan。

源码入口：`create_tidscan_paths()` 对 `CurrentOfExpr` 命中时会返回 true。

状态变化：`set_plain_rel_pathlist()` 看到 true 后直接返回。

fallback 或边界：这不是 cost 选择，而是 executor contract 要求。

诊断要点：断点应放在 early return 前，确认 `baserestrictinfo` 中的表达式。

最小实验：可用实验是用 `WHERE CURRENT OF` 与普通 TID 条件对照。


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

可迁移结论：这个案例说明某些 path 是语义强制，不参与普通竞争。

### 案例 3: partial path 到 Gather 的两阶段

现象：`partial_pathlist` 中有候选，但最终计划里不一定看到它。

源码入口：partial path 先由 `add_partial_path()` 保存。

状态变化：`generate_useful_gather_paths()` 后才变成普通 pathlist 中可比较的 Gather path。

fallback 或边界：如果 rel 是 inheritance child，Gather 可能推迟到 parent appendrel。

诊断要点：诊断时同时打印 `pathlist` 与 `partial_pathlist`。

最小实验：可用实验是让大表满足 parallel safety，再调 worker GUC。


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

可迁移结论：这个案例说明 worker 内路径和最终可执行路径之间还有包装层。

### 案例 4: pathkeys 让局部较贵路径保留

现象：一个 index path 成本略高，却没有在 `add_path()` 中被删除。

源码入口：检查它是否携带有用 `pathkeys`。

状态变化：上层 ORDER BY、Merge Join 或 GroupAggregate 可能利用这些 pathkeys。

fallback 或边界：`add_path()` 的支配关系不是只比较 total_cost。

诊断要点：断点时打印 pathkeys、startup_cost、total_cost 和 required_outer。

最小实验：可用实验是加 ORDER BY 再移除 ORDER BY。


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

可迁移结论：这个案例说明 pathlist 保留的是多维候选。

### 案例 5: parameterized path 的位置限制

现象：某个 path 只能在 nested loop inner 出现。

源码入口：`Path.param_info` 记录 required_outer。

状态变化：当前 join order 没有提供这些外部 rel 时，该 path 不能执行。

fallback 或边界：不要把 parameterized path 与普通 path 简单按 total_cost 横比。

诊断要点：诊断时打印 `PATH_REQ_OUTER(path)`。

最小实验：可用实验是改变 join 顺序或外表大小，观察 inner index path 是否出现。


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

可迁移结论：这个案例说明 path 可用性依赖上层 join search。

### 案例 6: hook 与 Gather 的顺序

现象：扩展添加 partial path 后，希望它能被包装成 Gather。

源码入口：`set_rel_pathlist_hook` 在 core path 之后、Gather 生成之前调用。

状态变化：扩展如果想增加 parallel-aware path，应使用 `add_partial_path()`。

fallback 或边界：如果 hook 太晚删除 path，已被 `add_path()` 丢弃的候选很难恢复。

诊断要点：诊断时记录 hook 前后 pathlist 和 partial_pathlist。

最小实验：可用实验是用最小扩展添加 CustomPath。


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

可迁移结论：这个案例说明扩展入口也必须服从 pathlist 生命周期。
