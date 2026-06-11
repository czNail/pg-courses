# PostgreSQL join path 生成：Nested Loop、Merge、Hash 与扩展路径竞争

## 课程定位

前置知识：已经理解 joinrel 动态规划会提出合法 join pair，也知道 `make_join_rel()` 在合法性通过后得到或创建目标 joinrel。

本节唯一主问题：

`add_paths_to_joinrel()` 如何在同一个合法 joinrel 中生成 nested loop、merge join、hash join 等候选，并让它们在成本、排序、参数化、唯一性和 join type 约束下竞争？

核心矛盾：一个 joinrel 的 SQL 语义已经固定，但物理实现还没有固定；planner 既要保留不同算法的机会，又要根据 join type、merge/hash 条件、parameterization、inner uniqueness、parallel 和 FDW 能力限制候选数量。

学完后应能判断：看到 Hash Join、Merge Join、Nested Loop、Memoize、Foreign Join 或缺失某类 join path 时，能定位到 `add_paths_to_joinrel()` 内部哪个判断让候选生成、跳过或被剪枝。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释了哪些 join pair 可以合法进入 path generation。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节讲 relation 数量过大时，GEQO 如何替代 exhaustive join order 搜索。

阅读时沿一个合法 joinrel 看 `JoinPathExtraData` 怎样被填充，再看 nested loop、merge、hash、foreign join 和 hook 各自消耗哪些字段。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`make_join_rel()` 合法后调用 `build_join_rel()`；非 dummy joinrel 计算 restrictlist；按 join type 调用 `add_paths_to_joinrel()`；初始化 `JoinPathExtraData`；`join_path_setup_hook` 可调整 `pgs_mask`；判断 inner side 是否 unique；选择 mergejoin clauses；semi/anti 或 inner_unique 时计算 correction factors；计算 parameterized path 来源限制；生成 sort-based merge paths；生成 unsorted outer 的 nestloop/merge paths；生成 hash paths；FDW 添加 foreign join paths；`set_join_pathlist_hook` 最后修改 pathlist。

1. `make_join_rel()` 合法后调用 `build_join_rel()`
   合法性只确定这个 relids 组合可以存在，`build_join_rel()` 才创建或复用承载 rows、restrictlist 和 pathlist 的 joinrel。

2. 非 dummy joinrel 计算 restrictlist
   `restrictlist` 是当前 join 层可以执行的 qual 集合，决定 selectivity、merge/hash key 以及 join filter 的边界。

3. 按 join type 调用 `add_paths_to_joinrel()`
   JOIN_LEFT、JOIN_FULL、JOIN_SEMI、JOIN_ANTI 等类型会改变允许的算法、输入方向和 extra cost 修正。

4. 初始化 `JoinPathExtraData`
   `extra` 把 `restrictlist`、`sjinfo`、`pgs_mask`、merge clauses、semi factors 和 parameterization 限制集中传给算法生成函数。

5. `join_path_setup_hook` 可调整 `pgs_mask`
   hook 可以按特定 joinrel/outer/inner 清除或恢复算法位，因此诊断禁用算法时要读 `extra.pgs_mask` 而不只读 GUC。

6. 判断 inner side 是否 unique
   `inner_unique` 会改变 semi/anti 或普通 join 的 stop-early 估算，也会影响 Memoize 是否值得被考虑。

7. 选择 mergejoin clauses
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. semi/anti 或 inner_unique 时计算 correction factors
   `semifactors` 把匹配比例和平均匹配数交给 costsize，使 nested loop、hash 和 merge 的成本能反映提前停止。

9. 计算 parameterized path 来源限制
   `param_source_rels` 控制 joinrel 结果还能依赖哪些外部 rel，避免高层搜索被无意义参数化 path 撑爆。

10. 生成 sort-based merge paths
   `sort_inner_and_outer()` 从 cheapest-total 输入加显式排序，适合没有天然顺序但有 mergejoinable clause 的场景。

11. 生成 unsorted outer 的 nestloop/merge paths
   `match_unsorted_outer()` 遍历 outer path，既生成 nested loop，也利用已有 outer pathkeys 尝试少排序的 merge join。

12. 生成 hash paths
   `hash_inner_and_outer()` 只接受 hashjoinable 且两侧匹配的 clause；outer join 还要排除 pushed-down qual。

13. FDW 添加 foreign join paths
   foreign join path 代表整个 join 可以被 FDW 接管或下推，core 仍把它作为同一个 joinrel 上的候选参与 `add_path()`。

14. `set_join_pathlist_hook` 最后修改 pathlist
   最后的 hook 看到所有 core join algorithm 候选，扩展可以追加 CustomPath，也可以清理不想暴露给上层的路径。

这条链路的重点是：join path generation 固定 SQL 语义后再比较物理实现，同一个 joinrel 可能同时保存多种算法和多种输入方向。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/joinrels.c` | `make_join_rel()`。 |
| 2 | `src/backend/optimizer/path/joinpath.c` | `add_paths_to_joinrel()`、`sort_inner_and_outer()`、`match_unsorted_outer()`、`hash_inner_and_outer()`。 |
| 3 | `src/backend/optimizer/path/costsize.c` | join cost 和 semi/anti factors。 |
| 4 | `src/backend/optimizer/util/pathnode.c` | NestedLoopPath、MergePath、HashPath、MemoizePath 构造。 |
| 5 | `src/backend/optimizer/plan/analyzejoins.c` | `innerrel_is_unique()`。 |
| 6 | `src/include/nodes/pathnodes.h` | Path、ParamPathInfo、SpecialJoinInfo。 |
| 7 | `src/include/foreign/fdwapi.h` | FDW join pushdown 回调。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：Nested Loop 出现在大表 join

源码解释：inner path 可能 parameterized

验证办法：打印 required_outer。

大表 nested loop 先看 inner path 的 `param_info` 和外层 rows；参数化索引路径可能让每次 probe 很便宜。

### 现象：Merge Join 缺失

源码解释：可能没有 mergejoinable clause

验证办法：看 `select_mergejoin_clauses()`。

Merge Join 缺失要区分没有 mergeable operator、outer join 需要全部 clause 可 merge、以及 `pgs_mask` 禁用这三种情况。

### 现象：Hash Join 缺失

源码解释：可能没有 hashable clause 或 pgs_mask 禁止

验证办法：断点 `hash_inner_and_outer()`。

Hash Join 诊断要检查 `hashjoinoperator`、`clause_sides_match_join()` 和 outer join pushed-down qual 过滤。

### 现象：Memoize 出现

源码解释：参数化 inner path 有 cache key 且重复扫描

验证办法：看 `get_memoize_path()`。

Memoize 是 nested loop 内侧的缓存候选，出现与否取决于参数化 key、预计重复率、inner uniqueness 和内存成本。

### 现象：FDW join pushdown

源码解释：FDW 添加 ForeignPath

验证办法：比较本地 join 与远端 join。

foreign join pushdown 要看两个输入是否属于同一 FDW/server/user mapping，以及 FDW 是否实现 `GetForeignJoinPaths()`。

### 现象：禁用算法仍出现 FULL JOIN 支持路径

源码解释：FULL JOIN 可能必须尝试 merge/hash

验证办法：构造 full outer join。

FULL JOIN 会绕过部分 enable 位继续尝试 merge/hash，因为 nested loop 不能实现该语义时需要保留可执行路径。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `restrictlist` | 当前 join 层可执行 restrictions | `build_join_rel()` | selectivity 和 cost |
| `mergeclause_list` | 可用于 merge join 的 clauses | `select_mergejoin_clauses()` | merge path |
| `inner_unique` | inner side 对 outer 是否唯一 | `innerrel_is_unique()` | semi/anti cost |
| `semifactors` | semi/anti 匹配比例修正 | `compute_semi_anti_join_factors()` | costsize |
| `param_source_rels` | 允许 parameterized join path 的来源 | join_info_list/lateral | 搜索空间控制 |
| `pgs_mask` | path generation strategy mask | GUC 或 hook | 算法尝试 |
| `pathkeys` | join path 输出顺序 | merge/ordered path | 上层 sort/group |
| `fdwroutine` | foreign join 入口 | FDW | GetForeignJoinPaths |

### `restrictlist`

语义：当前 join 层可执行 restrictions

写入或持有者：`build_join_rel()`

主要消费者：selectivity 和 cost

诊断提示：不同于 base restriction

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `mergeclause_list`

语义：可用于 merge join 的 clauses

写入或持有者：`select_mergejoin_clauses()`

主要消费者：merge path

诊断提示：缺失解释 Merge Join 消失

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `inner_unique`

语义：inner side 对 outer 是否唯一

写入或持有者：`innerrel_is_unique()`

主要消费者：semi/anti cost

诊断提示：不是索引名称

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `semifactors`

语义：semi/anti 匹配比例修正

写入或持有者：`compute_semi_anti_join_factors()`

主要消费者：costsize

诊断提示：影响 stop-early 成本

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `param_source_rels`

语义：允许 parameterized join path 的来源

写入或持有者：join_info_list/lateral

主要消费者：搜索空间控制

诊断提示：star schema 有放宽

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `pgs_mask`

语义：path generation strategy mask

写入或持有者：GUC 或 hook

主要消费者：算法尝试

诊断提示：要读 `extra.pgs_mask`

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `pathkeys`

语义：join path 输出顺序

写入或持有者：merge/ordered path

主要消费者：上层 sort/group

诊断提示：成本外的重要属性

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `fdwroutine`

语义：foreign join 入口

写入或持有者：FDW

主要消费者：GetForeignJoinPaths

诊断提示：远端能力边界

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `make_join_rel()` 合法后调用 `build_join_rel()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. 非 dummy joinrel 计算 restrictlist

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. 按 join type 调用 `add_paths_to_joinrel()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. 初始化 `JoinPathExtraData`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. `join_path_setup_hook` 可调整 `pgs_mask`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. 判断 inner side 是否 unique

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. 选择 mergejoin clauses

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. semi/anti 或 inner_unique 时计算 correction factors

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. 计算 parameterized path 来源限制

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. 生成 sort-based merge paths

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 11. 生成 unsorted outer 的 nestloop/merge paths

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 12. 生成 hash paths

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 13. FDW 添加 foreign join paths

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 14. `set_join_pathlist_hook` 最后修改 pathlist

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

- 算法必须支持对应 join type。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- merge join 需要 mergejoinable 条件。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hash join 需要 hashable equality。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parameterized inner 的参数必须由 outer 或更外层提供。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW pushdown 必须保持远端语义和权限。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hook 不能添加 executor 无法执行的 path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 没有 merge clause 时跳过 merge。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 没有 hash clause 时跳过 hash。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- GUC 关闭时 pgs_mask 清除，但 FULL JOIN 可能例外。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- inner 不 unique 时使用普通成本模型。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parameterization 受限时不生成膨胀搜索的 path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW 不支持 join 时回到本地算法。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- nested loop 成本常是 outer rows 乘 inner scan。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- Memoize 用内存换重复参数缓存。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- merge join 用排序成本换流式有序输出。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- hash join build 成本受 inner rows、width、work_mem 影响。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- semi/anti 可提前停止，semifactors 很关键。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FDW join 可减少传输但依赖远端估算。

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

1. `add_paths_to_joinrel()` 如何在同一个合法 joinrel 中生成 nested loop、merge join、hash join 等候选，并让它们在成本、排序、参数化、唯一性和 join type 约束下竞争？

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

- 本节主问题：`add_paths_to_joinrel()` 如何在同一个合法 joinrel 中生成 nested loop、merge join、hash join 等候选，并让它们在成本、排序、参数化、唯一性和 join type 约束下竞争？

## 17. 诊断案例切片

### 案例 1: Nested Loop 为什么赢

现象：大表 join 中出现 Nested Loop 不一定错误。

源码入口：parameterized index path 可能只能作为 inner。

状态变化：`match_unsorted_outer()` 会生成这类路径。

fallback 或边界：hash/merge 无法消费某些外部参数。

诊断要点：打印 inner `param_info` 和 required_outer。

最小实验：小表驱动大表索引。


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

可迁移结论：算法选择受 path 可执行位置影响。

### 案例 2: Merge Join 消失

现象：两个表有等值条件，但没有 Merge Join。

源码入口：`select_mergejoin_clauses()` 只接受 mergejoinable clauses。

状态变化：operator family、collation、join type 和 pgs_mask 都可能限制它。

fallback 或边界：没有 merge clause 时 sort cost 再低也不会生成 merge path。

诊断要点：打印 `mergeclause_list` 和 `mergejoin_allowed`。

最小实验：换成不可 merge 的 operator。


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

可迁移结论：算法候选首先需要语义能力。

### 案例 3: Hash Join 消失

现象：等值 join 仍没有 Hash Join。

源码入口：`hash_inner_and_outer()` 需要 hashable 条件，且 `pgs_mask` 没有禁止。

状态变化：FULL JOIN 有时即使 GUC 关闭也必须尝试可实现路径。

fallback 或边界：GUC 影响 path generation，但不改变 SQL contract。

诊断要点：看 hash operator 和 pgs_mask。

最小实验：关闭 hashjoin 后构造 FULL JOIN。


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

可迁移结论：GUC 不是合法性开关。

### 案例 4: Memoize 的缓存键

现象：Nested Loop 内侧出现 Memoize。

源码入口：`get_memoize_path()` 要求参数化 clauses 或 lateral vars 作为 cache key。

状态变化：outer rows 太少时普通 memoize 可能不值得。

fallback 或边界：binary mode 与 hash equality 还会影响可行性。

诊断要点：打印 param_exprs 和 hash operators。

最小实验：制造重复 outer 参数。


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

可迁移结论：Memoize 是 nested loop 内侧的成本优化。

### 案例 5: semi/anti 的 stop-early 成本

现象：EXISTS 或 NOT EXISTS 的 join cost 与普通 inner join 不同。

源码入口：`compute_semi_anti_join_factors()` 写 semifactors。

状态变化：inner_unique 或 semi/anti 允许估计较早停止 inner scan。

fallback 或边界：误读 semifactors 会误判 nested loop 成本。

诊断要点：记录 `inner_unique` 和 semifactors。

最小实验：给 RHS 加唯一约束。


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

可迁移结论：join type 会改变算法成本公式。

### 案例 6: FDW join pushdown

现象：两个 foreign table join 被推到远端。

源码入口：`add_paths_to_joinrel()` 后段调用 `GetForeignJoinPaths()`。

状态变化：FDW path 与本地 hash/merge/nestloop 一起竞争。

fallback 或边界：远端权限、server/user、表达式可下推性都影响可行性。

诊断要点：检查 joinrel 的 fdwroutine。

最小实验：postgres_fdw 同 server 与跨 server 对照。


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

可迁移结论：扩展路径仍进入 core pathlist 竞争。
