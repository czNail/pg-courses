# PostgreSQL joinrel 动态规划搜索与 relids 子问题复用

## 课程定位

前置知识：已经理解 base `RelOptInfo` 拥有 pathlist，知道 `relids` 表示一组 base relation，也知道 join clause 和 parameterized path 会限制候选组合。

本节唯一主问题：

optimizer 为什么按 relids 集合逐层构造 joinrel，而不是递归枚举所有 join tree 后再统一计算成本？

核心矛盾：join order 搜索空间随 relation 数量爆炸，但许多不同 join tree 会产生相同 relids 集合；动态规划复用相同子问题，同时用 join clause、join-order restriction、path pruning 和 fallback 控制搜索规模。

学完后应能判断：读懂 `root->join_rel_level` 如何从 level 2 增长到 N，并解释 clauseless join、bushy join、GEQO threshold 和 `set_cheapest()` 在搜索中的位置。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前四节已经为每个 base relation 生成了可竞争的扫描路径。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节专门解释 `join_is_legal()` 如何保护 outer join、semi/anti join、LATERAL 和 PlaceHolderVar 等语义边界。

阅读时以 `join_rel_level` 为时间轴：level 1 是 base pathlist，level k 是可复用 relids 子问题，level N 才是 scan/join 搜索的最终输入。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`make_rel_from_joinlist()` 把 joinlist 转成 initial rels；`join_search_hook` 可替换标准搜索；达到 GEQO 条件时调用 `geqo()`；标准路径调用 `standard_join_search()`；`standard_join_search()` 分配 `join_rel_level`；level 1 保存 initial rels；level 2 到 N 调用 `join_search_one_level()`；`make_join_rel()` 创建或复用 joinrel；新 joinrel 立即生成 paths、Gather 和 cheapest；level N 返回覆盖全部 rels 的 RelOptInfo。

1. `make_rel_from_joinlist()` 把 joinlist 转成 initial rels
   `initial_rels` 是 DP 的 level 1 输入，里面可能是 base rel，也可能是子 joinlist 已经规划出的局部 rel。

2. `join_search_hook` 可替换标准搜索
   hook 必须返回覆盖同一 `all_query_rels` 的 final rel，但可以用不同搜索算法和不同中间 joinrel 集合实现。

3. 达到 GEQO 条件时调用 `geqo()`
   当 initial rel 数达到阈值时，planner 不再完整填充 `join_rel_level`，而把 join order 搜索交给 GEQO 近似评估。

4. 标准路径调用 `standard_join_search()`
   标准路径按 level 逐层扩展，每个 level 只讨论固定大小的 relids 子问题，而不是一次性枚举完整 join tree。

5. `standard_join_search()` 分配 `join_rel_level`
   `join_rel_level[k]` 是本节最重要的时间轴：level 1 是输入，level N 是覆盖全部查询 rel 的结果。

6. level 1 保存 initial rels
   每个 base rel 已经带着自己的 pathlist 和 cheapest path；join search 从这些叶子候选开始组合。

7. level 2 到 N 调用 `join_search_one_level()`
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. `make_join_rel()` 创建或复用 joinrel
   `relids` 是复用 key；不同 join pair 可能指向同一个目标 joinrel，并继续向它追加物理 path。

9. 新 joinrel 立即生成 paths、Gather 和 cheapest
   一个 joinrel 只有在 path 生成、partitionwise/gather 补充和 `set_cheapest()` 后，才能作为更高 level 的输入。

10. level N 返回覆盖全部 rels 的 RelOptInfo
   最终 rel 仍然是 planner-local 候选集合，upper planning 会继续在它的 cheapest 与 ordered path 上叠加 aggregate、sort、limit 等阶段。

这条链路的重点是：DP 复用的是相同 `relids` 的子问题，而不是复用 SQL 文本顺序；每个 joinrel 内部仍保留多条物理 path。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/allpaths.c` | `make_rel_from_joinlist()`、`standard_join_search()`。 |
| 2 | `src/backend/optimizer/path/joinrels.c` | `join_search_one_level()`、`make_rels_by_clause_joins()`、`make_join_rel()`。 |
| 3 | `src/backend/optimizer/path/joinpath.c` | `add_paths_to_joinrel()` 为 joinrel 添加算法路径。 |
| 4 | `src/backend/optimizer/util/relnode.c` | `build_join_rel()` 创建或复用 join `RelOptInfo`。 |
| 5 | `src/backend/optimizer/path/costsize.c` | `set_joinrel_size_estimates()` 写 joinrel rows。 |
| 6 | `src/include/nodes/pathnodes.h` | `join_rel_level`、`join_cur_level`、`join_rel_list`。 |
| 7 | `src/include/optimizer/paths.h` | `join_search_hook` 声明。 |
| 8 | `src/backend/optimizer/geqo/geqo_main.c` | GEQO 入口。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：planning time 低于全排列枚举

源码解释：DP 复用 relids 子问题

验证办法：统计每层 joinrel 数量。

对比全排列时要统计的是每层唯一 `relids` 集合数量，而不是 SQL join tree 的文本排列数。

### 现象：缺 predicate 出现 Cartesian product

源码解释：clauseless join 是 fallback

验证办法：删除 join 条件。

Cartesian product 通常说明 `joininfo`、EC 或 join order restriction 没有把两个子问题连接起来，DP 只能保底生成可执行组合。

### 现象：bushy plan 不大量出现

源码解释：只在有相关 clause 或 restriction 时尝试

验证办法：构造两个独立 join 子图。

bushy join 受 `have_relevant_joinclause()` 与 `have_join_order_restriction()` 控制，这是控制规划时间的关键剪枝。

### 现象：同一 relids 有多条 path

源码解释：不同 pair 向同一 joinrel 添加 path

验证办法：打印 build/reuse。

同一 `relids` 下多条 path 是 DP 复用的正常结果，差异来自 outer/inner 方向、算法、pathkeys 和 parameterization。

### 现象：GEQO 后没有完整 level DP

源码解释：GEQO 改用 tour 和 clump

验证办法：调低 threshold。

进入 GEQO 后不要期待 `join_rel_level[2..N]` 完整存在，诊断入口应切到 `geqo_eval()` 和 `gimme_tree()`。

### 现象：某层为空但最终可规划

源码解释：special join 可能让中间层暂时无合法组合

验证办法：构造 semi join 子问题。

中间层为空要继续看 last-ditch cartesian 和 special join legality，不能直接断言 planner 没有搜索对应大小的子问题。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `initial_rels` | join search level 1 输入 | `make_rel_from_joinlist()` | standard DP、GEQO、hook |
| `join_rel_level` | level k 的 joinrel list | `standard_join_search()` | `join_search_one_level()` |
| `join_cur_level` | 当前构造层级 | join search | `build_join_rel()` |
| `RelOptInfo.relids` | joinrel 集合身份 | rel 构造 | reuse 和 path competition |
| `joininfo` | 与外部 rel 相关 join clauses | initsplan/joininfo | clause joins |
| `has_eclass_joins` | EC 可派生 join clause | equivclass | join search |
| `join_rel_list` | 所有 joinrel 总表 | relnode | GEQO restore 和调试 |
| `all_query_rels` | 最终 relids 集合 | planner init | top rel 判断 |

### `initial_rels`

语义：join search level 1 输入

写入或持有者：`make_rel_from_joinlist()`

主要消费者：standard DP、GEQO、hook

诊断提示：jointree flatten 影响它

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `join_rel_level`

语义：level k 的 joinrel list

写入或持有者：`standard_join_search()`

主要消费者：`join_search_one_level()`

诊断提示：最直接时间轴

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `join_cur_level`

语义：当前构造层级

写入或持有者：join search

主要消费者：`build_join_rel()`

诊断提示：新 joinrel 挂到这里

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `RelOptInfo.relids`

语义：joinrel 集合身份

写入或持有者：rel 构造

主要消费者：reuse 和 path competition

诊断提示：DP 复用 key

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `joininfo`

语义：与外部 rel 相关 join clauses

写入或持有者：initsplan/joininfo

主要消费者：clause joins

诊断提示：为空不代表不能 join

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `has_eclass_joins`

语义：EC 可派生 join clause

写入或持有者：equivclass

主要消费者：join search

诊断提示：不要只看显式 qual

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `join_rel_list`

语义：所有 joinrel 总表

写入或持有者：relnode

主要消费者：GEQO restore 和调试

诊断提示：不等于层级时间轴

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `all_query_rels`

语义：最终 relids 集合

写入或持有者：planner init

主要消费者：top rel 判断

诊断提示：区分中间和最终 joinrel

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `make_rel_from_joinlist()` 把 joinlist 转成 initial rels

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. `join_search_hook` 可替换标准搜索

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. 达到 GEQO 条件时调用 `geqo()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. 标准路径调用 `standard_join_search()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. `standard_join_search()` 分配 `join_rel_level`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. level 1 保存 initial rels

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. level 2 到 N 调用 `join_search_one_level()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. `make_join_rel()` 创建或复用 joinrel

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. 新 joinrel 立即生成 paths、Gather 和 cheapest

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. level N 返回覆盖全部 rels 的 RelOptInfo

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

- 输入 relids 不能 overlap。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- join legality 先于 path generation。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- restrictlist 必须在正确 join 层级执行。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hook 必须返回覆盖全部 rels 的合法 RelOptInfo。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- clauseless fallback 不能绕过 special join 约束。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- top rel 与中间 rel 的 Gather 时机不同。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 无 join clause 时生成 Cartesian product。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 某层无合法 join 时 special join/lateral 场景可继续。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 搜索过大时 GEQO 接管。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hook 不存在时使用 standard DP。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 非法 pair 返回 NULL 后尝试其它 pair。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 没有 special join 且无法构造会报 sanity error。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- 相同 relids 只维护一个 joinrel。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 每个 joinrel 的 pathlist 立即剪枝。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- bushy 限制降低 planning time。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- Cartesian fallback 保证可规划但可能很贵。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- GEQO threshold 用近似搜索换上界。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- `set_cheapest()` 降低下一层比较成本。

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

1. optimizer 为什么按 relids 集合逐层构造 joinrel，而不是递归枚举所有 join tree 后再统一计算成本？

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

- 本节主问题：optimizer 为什么按 relids 集合逐层构造 joinrel，而不是递归枚举所有 join tree 后再统一计算成本？

## 17. 诊断案例切片

### 案例 1: level 数组如何增长

现象：三表 join 中 `join_rel_level[2]` 和 `[3]` 逐层出现。

源码入口：`standard_join_search()` 分配数组，`join_search_one_level()` 写当前层。

状态变化：level k 表示包含 k 个 initial rel 的 joinrel，不等于 plan tree 深度。

fallback 或边界：中间层可能被剪枝，但只要有合法 final rel，搜索还能继续。

诊断要点：打印每层 relids 集合，而不是只看最终 join tree。

最小实验：三表链式 join。


复盘路径：

- 先把现象定位到一个可见 Plan 节点、join level、pathlist 变化或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝、合法性拒绝或 fallback。

课堂追问：

- 这个案例里最早被写错、写保守或拒绝的状态是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：DP 的时间轴是 relids 集合大小。

### 案例 2: 同一 relids 的复用

现象：不同 join tree 可能汇入同一个 final relids。

源码入口：`build_join_rel()` 创建或复用 `RelOptInfo`。

状态变化：不同 pair 向同一 joinrel 的 pathlist 添加不同 path。

fallback 或边界：复用失败会带来重复子问题和 planning time 放大。

诊断要点：记录 joinrel 是否已经存在。

最小实验：四表 join 中断 `build_join_rel()`。


复盘路径：

- 先把现象定位到一个可见 Plan 节点、join level、pathlist 变化或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝、合法性拒绝或 fallback。

课堂追问：

- 这个案例里最早被写错、写保守或拒绝的状态是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：joinrel 是子问题，不是一棵固定树。

### 案例 3: clauseless fallback

现象：缺少 join predicate 时仍能生成计划。

源码入口：`make_rels_by_clauseless_joins()` 对不相交 relids 生成 Cartesian join。

状态变化：这个 fallback 保证可规划性，不表示 optimizer 推荐笛卡尔积。

fallback 或边界：仍要经过 `join_is_legal()`，不能绕过 special join。

诊断要点：检查 `joininfo`、`has_eclass_joins` 和 fallback 调用点。

最小实验：删除一个 join 条件。


复盘路径：

- 先把现象定位到一个可见 Plan 节点、join level、pathlist 变化或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝、合法性拒绝或 fallback。

课堂追问：

- 这个案例里最早被写错、写保守或拒绝的状态是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：fallback 保护存在性，不保证最优性。

### 案例 4: bushy join 的限制

现象：planner 没有枚举所有 bushy tree。

源码入口：`join_search_one_level()` 只在相关 clause 或 restriction 下尝试 bushy pair。

状态变化：这控制 planning time，避免组合爆炸。

fallback 或边界：低成本 bushy 形态可能因为缺少相关约束而不被枚举。

诊断要点：看 `have_relevant_joinclause()` 与 `have_join_order_restriction()`。

最小实验：两个独立 join 子图。


复盘路径：

- 先把现象定位到一个可见 Plan 节点、join level、pathlist 变化或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝、合法性拒绝或 fallback。

课堂追问：

- 这个案例里最早被写错、写保守或拒绝的状态是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：搜索空间是工程折中，不是数学全集。

### 案例 5: special join 让中间层为空

现象：某个 level 没有 joinrel，但最终仍可能有合法 plan。

源码入口：源码允许 special join 或 lateral 场景下中间层暂时失败。

状态变化：后续 bushy 组合可能恢复可行计划。

fallback 或边界：没有 special join/lateral 时同样失败会触发 sanity error。

诊断要点：同时看 `join_info_list` 和 `hasLateralRTEs`。

最小实验：多个 semi join 子问题。


复盘路径：

- 先把现象定位到一个可见 Plan 节点、join level、pathlist 变化或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝、合法性拒绝或 fallback。

课堂追问：

- 这个案例里最早被写错、写保守或拒绝的状态是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：中间层为空不一定等于无法规划。

### 案例 6: GEQO 分叉

现象：relation 数量变大后，看不到完整 `join_rel_level`。

源码入口：`make_rel_from_joinlist()` 在 hook 之后按阈值调用 `geqo()`。

状态变化：GEQO 替换 join order 搜索，不替换 join path generation。

fallback 或边界：调低 threshold 会改变搜索入口和可解释性。

诊断要点：记录 `enable_geqo`、`geqo_threshold` 和 initial rel 数。

最小实验：调低 `geqo_threshold`。


复盘路径：

- 先把现象定位到一个可见 Plan 节点、join level、pathlist 变化或 planning time 变化。

- 再回到本案例给出的源码入口，确认当前 SQL 是否真的经过这个分支。

- 接着打印本案例涉及的核心字段，不要只记录函数是否被调用。

- 然后构造一个只改变单个输入因素的对照 SQL。

- 最后把差异解释为状态写入、候选生成、候选剪枝、合法性拒绝或 fallback。

课堂追问：

- 这个案例里最早被写错、写保守或拒绝的状态是哪一个？

- 如果只看最终 `EXPLAIN`，会漏掉哪一个中间状态？

- 哪个 fallback 保护了正确性，哪个 fallback 只是保留可规划性？

- 这个结论在统计信息、GUC 或源码版本变化后还成立吗？

可迁移结论：DP 与 GEQO 的分界发生在 join order 搜索入口。
