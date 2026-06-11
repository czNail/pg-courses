# PostgreSQL join order 合法性与 SpecialJoinInfo 语义边界

## 课程定位

前置知识：已经理解 standard join search 如何按 relids 构造 joinrel，也知道不同 join pair 会在 `make_join_rel()` 中进入合法性检查。

本节唯一主问题：

join search 为什么不能只按成本自由交换 join 顺序，`join_is_legal()` 究竟在保护哪些 SQL 语义边界？

核心矛盾：inner join 在很多场景下可以交换和结合，但 outer join、semi/anti join、LATERAL、PlaceHolderVar、nullingrels 和 join identity 让便宜的顺序不一定合法。

学完后应能判断：看到某个 join order 没被考虑时，能判断它是成本失败、启发式未枚举，还是被 `SpecialJoinInfo`、LATERAL 或 RHS violation 正确拒绝。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲了 joinrel 动态规划如何提出候选 join pair。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节在 join pair 已经合法的前提下，讲 `add_paths_to_joinrel()` 如何生成 join algorithm path。

阅读时先把成本放到一边，沿 `SpecialJoinInfo`、LATERAL 和 PlaceHolderVar 看哪些 relids 组合根本不能进入算法竞争。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`reduce_outer_joins()` 先简化可降级 outer join；`deconstruct_jointree()` 拆 joinlist 并分发 quals；`make_outerjoininfo()` 创建 `SpecialJoinInfo`；join search 提出 rel pair；`make_join_rel()` 构造 union relids；`join_is_legal()` 扫描 `join_info_list`；匹配 special join 时返回 sjinfo 和 reversed；LATERAL 检查方向与 direct reference；非法 pair 返回 NULL；合法 pair 进入 path generation。

1. `reduce_outer_joins()` 先简化可降级 outer join
   能被严格 qual 证明退化的 outer join 会提前变成更自由的 join type，减少后续 `SpecialJoinInfo` 约束。

2. `deconstruct_jointree()` 拆 joinlist 并分发 quals
   joinlist 决定 join search 初始子问题，qual 分发决定哪些 clause 进入 base restriction、joininfo 或 special join 边界。

3. `make_outerjoininfo()` 创建 `SpecialJoinInfo`
   `min_lefthand`、`min_righthand`、`syn_*`、`lhs_strict` 和 `jointype` 共同记录 outer/semi/anti join 的最小语义边界。

4. join search 提出 rel pair
   DP 只负责提出可能组合的 `rel1` 和 `rel2`，合法性检查才决定这个组合是否能保持 SQL 的 null-extension 与半连接语义。

5. `make_join_rel()` 构造 union relids
   传给 `join_is_legal()` 的 union relids 还不含 outer join RT index；只有通过检查后才会被规范化为 canonical joinrelids。

6. `join_is_legal()` 扫描 `join_info_list`
   它先排除与当前 union 无关的 special join，再处理 RHS overlap、FULL JOIN、SEMI unique-ify 和多重 special join 冲突。

7. 匹配 special join 时返回 sjinfo 和 reversed
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. LATERAL 检查方向与 direct reference
   LATERAL 与 PlaceHolderVar 会通过 relids 约束 join direction；有 join clause 不代表方向可以自由交换。

9. 非法 pair 返回 NULL
   `make_join_rel()` 对非法 pair 直接丢弃，不创建 joinrel，也不会进入 `add_paths_to_joinrel()` 的成本竞争。

10. 合法 pair 进入 path generation
   通过合法性检查之后，成本模型才能讨论 nested loop、merge 或 hash；合法性失败与算法成本无关。

这条链路的重点是：join legality 是成本模型之前的语义门槛；不合法的 join pair 不会生成 path，也不会出现在后续剪枝记录里。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/prep/prepjointree.c` | `reduce_outer_joins()`。 |
| 2 | `src/backend/optimizer/plan/initsplan.c` | `deconstruct_jointree()`、`make_outerjoininfo()`、`find_lateral_references()`。 |
| 3 | `src/backend/optimizer/path/joinrels.c` | `join_is_legal()`、`make_join_rel()`、`have_join_order_restriction()`。 |
| 4 | `src/backend/optimizer/plan/analyzejoins.c` | semi join unique-ify 和 join removal 相邻逻辑。 |
| 5 | `src/backend/optimizer/util/relnode.c` | `build_join_rel()`。 |
| 6 | `src/include/nodes/pathnodes.h` | `SpecialJoinInfo`、`PlaceHolderInfo`、lateral 字段。 |
| 7 | `src/backend/optimizer/util/joininfo.c` | join clause 与 order restriction 辅助判断。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：LEFT JOIN 没被交换

源码解释：null-extension 语义限制交换

验证办法：A LEFT JOIN B 再 join C。

LEFT JOIN 的 RHS 不能在保持 null-extension 语义前被随意合并；要看 `min_righthand` 是否与候选 union 发生冲突。

### 现象：SEMI RHS 先 unique-ify

源码解释：semi join 有特殊放宽

验证办法：用 IN 子查询。

SEMI RHS unique-ify 会把部分顺序转成普通 inner join 讨论，但前提是 `create_unique_paths()` 能产生可用唯一化路径。

### 现象：LATERAL 强制方向

源码解释：referencer 必须在内侧

验证办法：构造 LATERAL subquery。

LATERAL 的方向来自 `direct_lateral_relids` 和 `lateral_relids`，错误方向即使成本很低也不能生成 path。

### 现象：FULL JOIN 限制更强

源码解释：两侧都可能 null-extension

验证办法：加入 FULL JOIN。

FULL JOIN 的限制更强，很多重排无法用普通 inner join identity 证明安全，所以 legality 往往早于成本剪掉候选。

### 现象：某层没有合法 join

源码解释：special join 可让中间层缺失

验证办法：构造多个 semi join 子问题。

某层没有合法 joinrel 时，要检查 special join 的最小左右手集合，而不是只看 `join_search_one_level()` 是否遍历到了该层。

### 现象：有 join clause 仍不能 join

源码解释：相关 clause 不等于合法顺序

验证办法：同时断 `have_relevant_joinclause()` 和 `join_is_legal()`。

`have_relevant_joinclause()` 只说明有连接条件，`join_is_legal()` 才回答这两个 relids 现在能不能按这个方向合并。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `min_lefthand` | special join 最小左侧 relids | `make_outerjoininfo()` | `join_is_legal()` |
| `min_righthand` | special join 最小右侧 relids | `make_outerjoininfo()` | RHS overlap 检查 |
| `syn_lefthand/syn_righthand` | 语法层左右输入 | jointree 分解 | semi unique 逻辑 |
| `jointype` | JOIN_LEFT/FULL/SEMI/ANTI 等 | SpecialJoinInfo | legality 和 path generation |
| `lhs_strict` | outer join identity 需要的严格性 | make_outerjoininfo | RHS association |
| `lateral_relids` | 仍依赖的外部 rels | lateral 分析 | legality 和 parameterization |
| `direct_lateral_relids` | 直接 lateral 来源 | lateral 分析 | 方向检查 |
| `join_info_list` | special join 集合 | initsplan | `join_is_legal()` |

### `min_lefthand`

语义：special join 最小左侧 relids

写入或持有者：`make_outerjoininfo()`

主要消费者：`join_is_legal()`

诊断提示：保护最小语义边界

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `min_righthand`

语义：special join 最小右侧 relids

写入或持有者：`make_outerjoininfo()`

主要消费者：RHS overlap 检查

诊断提示：过早合入 RHS 会改变语义

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `syn_lefthand/syn_righthand`

语义：语法层左右输入

写入或持有者：jointree 分解

主要消费者：semi unique 逻辑

诊断提示：不要与 min hand 混用

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `jointype`

语义：JOIN_LEFT/FULL/SEMI/ANTI 等

写入或持有者：SpecialJoinInfo

主要消费者：legality 和 path generation

诊断提示：决定可交换空间

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `lhs_strict`

语义：outer join identity 需要的严格性

写入或持有者：make_outerjoininfo

主要消费者：RHS association

诊断提示：三值逻辑进入 planner

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `lateral_relids`

语义：仍依赖的外部 rels

写入或持有者：lateral 分析

主要消费者：legality 和 parameterization

诊断提示：不能只看 join clause

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `direct_lateral_relids`

语义：直接 lateral 来源

写入或持有者：lateral 分析

主要消费者：方向检查

诊断提示：间接依赖不能跳级

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `join_info_list`

语义：special join 集合

写入或持有者：initsplan

主要消费者：`join_is_legal()`

诊断提示：为空不代表没有普通 join clause

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `reduce_outer_joins()` 先简化可降级 outer join

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. `deconstruct_jointree()` 拆 joinlist 并分发 quals

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. `make_outerjoininfo()` 创建 `SpecialJoinInfo`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. join search 提出 rel pair

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. `make_join_rel()` 构造 union relids

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. `join_is_legal()` 扫描 `join_info_list`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. 匹配 special join 时返回 sjinfo 和 reversed

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. LATERAL 检查方向与 direct reference

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. 非法 pair 返回 NULL

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. 合法 pair 进入 path generation

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

- outer join nulling 不能被错误重排。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- semi/anti 是存在性语义，不等同 inner join。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- FULL JOIN 两边保留未匹配行，限制最大。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- LATERAL 外部参数必须先可用。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- PlaceHolderVar 的 evaluation level 不能错位。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- outer join identity 的 strictness 条件必须满足。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 非法 pair 返回 NULL，搜索其它 pair。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- semi RHS 可 unique 时扩大合法顺序。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- RHS 内部必要 join 可被允许继续。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 某层无 joinrel 时 special join/lateral 可等待后续层。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 没有 special join 时失败触发 sanity error。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- LATERAL 无法满足时拒绝该 order。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- 合法性先于成本，非法 order 不参与比较。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- unique-ify 用去重成本换更大搜索空间。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- outer join 限制可能阻止低成本物理顺序。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- LATERAL 常迫使 nested loop 和 parameterized path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- strictness 检查用搜索空间换正确性。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 多试合法 pair 的 planning 成本低于接受错误 order 的风险。

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

1. join search 为什么不能只按成本自由交换 join 顺序，`join_is_legal()` 究竟在保护哪些 SQL 语义边界？

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

- 本节主问题：join search 为什么不能只按成本自由交换 join 顺序，`join_is_legal()` 究竟在保护哪些 SQL 语义边界？

## 17. 诊断案例切片

### 案例 1: LEFT JOIN 不能随意交换

现象：成本更低的顺序可能改变 null-extension 语义。

源码入口：`SpecialJoinInfo.min_lefthand` 和 `min_righthand` 描述最小合法边界。

状态变化：`join_is_legal()` 发现 RHS violation 时拒绝 proposed join。

fallback 或边界：这一步先于任何算法成本比较。

诊断要点：打印 proposed joinrelids 和 SJ hands。

最小实验：A LEFT JOIN B 再 join C。


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

可迁移结论：合法性是 correctness firewall。

### 案例 2: semi join 的 unique-ify 放宽

现象：IN/EXISTS 场景有时可以先 unique RHS。

源码入口：`join_is_legal()` 对 JOIN_SEMI 有 `create_unique_paths()` 分支。

状态变化：RHS 可 unique 时，某些 inner join 顺序变得合法。

fallback 或边界：这是等价转换，不是语义放松。

诊断要点：记录 `syn_righthand` 和 unique path。

最小实验：小 RHS 的 IN 子查询。


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

可迁移结论：legality 也会为性能保留空间。

### 案例 3: FULL JOIN 的强限制

现象：FULL JOIN 两侧都保留未匹配行。

源码入口：很多 outer join identity 对 FULL JOIN 不成立。

状态变化：`join_is_legal()` 对 FULL JOIN 和 lateral 组合更保守。

fallback 或边界：禁用算法也不能让非法顺序变合法。

诊断要点：先查 jointype，再谈 cost。

最小实验：把 LEFT JOIN 改成 FULL JOIN。


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

可迁移结论：join type 决定搜索自由度。

### 案例 4: LATERAL 的方向要求

现象：一个 relation 引用另一个 relation 的列。

源码入口：`lateral_fwd` 和 `lateral_rev` 判断依赖方向。

状态变化：双向 lateral 直接失败；单向 lateral 常要求 nested loop 方向。

fallback 或边界：间接引用也可能被拒绝。

诊断要点：打印 `lateral_relids` 和 `direct_lateral_relids`。

最小实验：LATERAL subquery 引用外表。


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

可迁移结论：join order 受表达式求值依赖控制。

### 案例 5: PlaceHolderVar 与 eval level

现象：outer join 上方表达式不能随便提前求值。

源码入口：PlaceHolderInfo 记录表达式应在哪个 relids 层级可用。

状态变化：错误层级会改变 NULL 语义。

fallback 或边界：这类问题不一定表现为显式 join clause。

诊断要点：查 placeholder_list 和 ph_eval_at。

最小实验：outer join 上方引用表达式。


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

可迁移结论：合法性不只来自 join quals。

### 案例 6: 普通 inner join 的 dummy sjinfo

现象：inner join 没有 `SpecialJoinInfo`，但估算函数仍需要 join sides。

源码入口：`make_join_rel()` 会临时初始化 dummy sjinfo。

状态变化：它只服务 selectivity/cost 估算，不代表 query 有 special join。

fallback 或边界：不要把 dummy sjinfo 与 `join_info_list` 成员混淆。

诊断要点：看 sjinfo 是否来自 root 列表。

最小实验：纯 inner join 对照 outer join。


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

可迁移结论：实现 helper 不等于语义对象。
