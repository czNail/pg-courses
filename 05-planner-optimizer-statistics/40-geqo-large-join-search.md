# PostgreSQL GEQO 与大连接查询的近似 join order 搜索

## 课程定位

前置知识：已经理解 standard join search 使用 `join_rel_level` 做动态规划，也知道 join path generation 会为合法 joinrel 添加物理算法候选。

本节唯一主问题：

当 join relation 太多导致动态规划不可承受时，GEQO 为什么把 join order 搜索变成遗传算法问题，它牺牲了哪些确定性和可解释性？

核心矛盾：exhaustive DP 能系统比较 relids 子问题，但 relation 数增大时 planning time 和内存会爆炸；GEQO 用随机种群、适应度评估和启发式 clump 合并限制 planning time，却不再保证找到全局最优 join order。

学完后应能判断：判断一个大 join 查询是否进入 GEQO，解释 `geqo_seed`、pool size、generations、invalid tour、`geqo_eval()` memory context 和 `gimme_tree()` clump heuristic 对计划稳定性和诊断的影响。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲了合法 joinrel 如何生成物理 join path。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

后续课程会进入 upper planning，讨论 scan/join rel 之上的 aggregate、sort、window、limit 等阶段。

阅读时把 GEQO 看成 join order 来源的替换件：它改变尝试哪些顺序，不改变每个合法合并时调用 join path generation 的规则。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`make_rel_from_joinlist()` 统计 initial rel 数量；hook 先于 GEQO；`enable_geqo` 且达到 `geqo_threshold` 时调用 `geqo()`；`geqo()` 初始化 private state 和随机种子；计算 pool size 和 generation 数；`random_init_pool()` 生成初始 tours；每个 tour 用 `geqo_eval()` 计算 fitness；`geqo_eval()` 创建临时 MemoryContext；`gimme_tree()` 按 tour 合并 clumps；`merge_clump()` 优先 desirable join；force merge 剩余 clumps；最优 chromosome 的 joinrel 返回。

1. `make_rel_from_joinlist()` 统计 initial rel 数量
   `levels_needed` 来自 joinlist 子节点数量，决定这次 join search 是否还能承受 standard DP。

2. hook 先于 GEQO
   `join_search_hook` 可以完全接管搜索；只有没有 hook 时，core 才在 GEQO 与 standard DP 之间二选一。

3. `enable_geqo` 且达到 `geqo_threshold` 时调用 `geqo()`
   触发 GEQO 后，planner 不再保证完整比较所有 relids 子问题，而是用有限 tour 样本评估 join order。

4. `geqo()` 初始化 private state 和随机种子
   private state 保存 `initial_rels` 的 gene 映射，`geqo_set_seed()` 让随机 tour 在给定 seed 下可复现。

5. 计算 pool size 和 generation 数
   pool 和 generation 决定搜索预算：越大越可能找到好顺序，也越接近把 planning time 花回去。

6. `random_init_pool()` 生成初始 tours
   每条 chromosome 是一个 relation 顺序建议，只有经过 `geqo_eval()` 转成 joinrel 后才有成本意义。

7. 每个 tour 用 `geqo_eval()` 计算 fitness
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. `geqo_eval()` 创建临时 MemoryContext
   评估会临时扩展 `join_rel_list` 和 joinrel hash，退出前截断恢复并删除 context，避免每个 generation 泄漏 planner 状态。

9. `gimme_tree()` 按 tour 合并 clumps
   tour 只是指导顺序；`gimme_tree()` 会把当前可合法合并的 rel 形成 clump，遇到暂时非法或不合适的组合就推迟。

10. `merge_clump()` 优先 desirable join
   desirable join 优先保留有连接条件或顺序约束的合并，减少随机 tour 直接落成糟糕 Cartesian 链的概率。

11. force merge 剩余 clumps
   扫完整个 tour 后仍有多个 clump 时才强制尝试任意合法合并；如果还不能合成一个 rel，fitness 变成 `DBL_MAX`。

12. 最优 chromosome 的 joinrel 返回
   返回的是采样搜索里最便宜 tour 对应的 joinrel，不代表全局最优，也不代表未采样顺序被证明更差。

这条链路的重点是：GEQO 用有限随机样本换取可控 planning time，诊断时要接受“未采样顺序没有成本记录”这个边界。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/allpaths.c` | `make_rel_from_joinlist()` 选择 standard、GEQO 或 hook。 |
| 2 | `src/backend/optimizer/geqo/geqo_main.c` | `geqo()` 主循环。 |
| 3 | `src/backend/optimizer/geqo/geqo_eval.c` | `geqo_eval()`、`gimme_tree()`、`merge_clump()`。 |
| 4 | `src/backend/optimizer/geqo/geqo_pool.c` | `random_init_pool()`、`sort_pool()`、`spread_chromo()`。 |
| 5 | `src/backend/optimizer/geqo/geqo_random.c` | `geqo_set_seed()`。 |
| 6 | `src/include/optimizer/geqo.h` | GEQO 结构和 GUC。 |
| 7 | `src/backend/optimizer/path/joinrels.c` | GEQO 评估仍调用 `make_join_rel()`。 |
| 8 | `src/backend/optimizer/path/joinpath.c` | GEQO 仍复用 join path generation。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：大 join planning time 突然下降

源码解释：可能进入 GEQO

验证办法：调 `geqo_threshold` 对比。

planning time 下降要同时记录 `enable_geqo`、`geqo_threshold` 和 initial rel 数；否则无法证明入口确实切到 GEQO。

### 现象：改 `geqo_seed` 后计划变化

源码解释：随机种子影响 tour

验证办法：固定 seed 复测。

固定 `geqo_seed` 是复现 GEQO 计划的前提；不同 seed 代表不同 tour 样本，不应当按 DP 的确定性来诊断。

### 现象：EXPLAIN 难解释未选顺序

源码解释：未采样 tour 不会完整比较

验证办法：增大 pool/generation 对照。

GEQO 下没有被采样的 join order 没有成本记录；只能通过扩大 pool/generation 或降低 threshold 回到 DP 做对照。

### 现象：随机 tour 无效

源码解释：`geqo_eval()` 返回 `DBL_MAX`

验证办法：加入 outer/lateral 约束。

无效 tour 通常来自 special join 或 LATERAL 限制导致 clump 无法合并，不能把 `DBL_MAX` 理解成真实执行成本。

### 现象：内存不随 generations 线性增长

源码解释：每次 eval 删除临时 context

验证办法：断点 `geqo_eval()`。

这说明 GEQO 评估会频繁构造临时 joinrel，必须同时恢复 `join_rel_list` 和 `join_rel_hash` 才能保持 planner 状态可复用。

### 现象：GEQO 仍生成 hash/merge/nestloop

源码解释：只替换 join order 搜索

验证办法：在 `gimme_tree()` 内看 `make_join_rel()`。

GEQO 只决定尝试哪些 join order；每个合法合并仍调用 `make_join_rel()`、`add_paths_to_joinrel()` 和 `set_cheapest()` 生成物理算法候选。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `enable_geqo` | 是否允许 GEQO | GUC | 入口选择 |
| `geqo_threshold` | 触发阈值 | GUC | 入口选择 |
| `Geqo_seed` | 随机种子 | `geqo_set_seed()` | tour 生成 |
| `Pool` | 种群和 worth | `alloc_pool()` | selection/sort |
| `Chromosome` | join order tour | random/recombination | fitness eval |
| `initial_rels` | gene index 对应 rel | GEQO private state | `gimme_tree()` |
| `Clump` | 已合并的局部 joinrel | `gimme_tree()` | merge_clump |
| `join_rel_list` 保存点 | eval 前保存长度和 hash | `geqo_eval()` | 状态恢复 |

### `enable_geqo`

语义：是否允许 GEQO

写入或持有者：GUC

主要消费者：入口选择

诊断提示：先记录它

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `geqo_threshold`

语义：触发阈值

写入或持有者：GUC

主要消费者：入口选择

诊断提示：看 initial rel 数

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Geqo_seed`

语义：随机种子

写入或持有者：`geqo_set_seed()`

主要消费者：tour 生成

诊断提示：固定 seed 才易复现

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Pool`

语义：种群和 worth

写入或持有者：`alloc_pool()`

主要消费者：selection/sort

诊断提示：太小影响覆盖

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Chromosome`

语义：join order tour

写入或持有者：random/recombination

主要消费者：fitness eval

诊断提示：不是最终 Plan

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `initial_rels`

语义：gene index 对应 rel

写入或持有者：GEQO private state

主要消费者：`gimme_tree()`

诊断提示：错位会破坏评估

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `Clump`

语义：已合并的局部 joinrel

写入或持有者：`gimme_tree()`

主要消费者：merge_clump

诊断提示：多个 clump 代表暂未合法合并

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `join_rel_list` 保存点

语义：eval 前保存长度和 hash

写入或持有者：`geqo_eval()`

主要消费者：状态恢复

诊断提示：避免污染全局 planner

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `make_rel_from_joinlist()` 统计 initial rel 数量

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. hook 先于 GEQO

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. `enable_geqo` 且达到 `geqo_threshold` 时调用 `geqo()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. `geqo()` 初始化 private state 和随机种子

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. 计算 pool size 和 generation 数

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. `random_init_pool()` 生成初始 tours

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. 每个 tour 用 `geqo_eval()` 计算 fitness

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. `geqo_eval()` 创建临时 MemoryContext

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. `gimme_tree()` 按 tour 合并 clumps

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. `merge_clump()` 优先 desirable join

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 11. force merge 剩余 clumps

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 12. 最优 chromosome 的 joinrel 返回

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

- GEQO tour 仍调用 `make_join_rel()` 和 `join_is_legal()`。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 无法形成合法 join tree 的 tour 返回 `DBL_MAX`。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 随机只影响搜索顺序，不允许非法 join。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- clump heuristic 优先 desirable join，force 阶段仍要求合法。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 临时评估不能污染最终 planner 状态。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- join_search_hook 接管时 GEQO 不再替换。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- 未达阈值时 standard DP。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- `enable_geqo` false 时 standard DP。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- invalid tour 丢弃或赋 `DBL_MAX`。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 初始 pool 多次无效会报错避免无限循环。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- desirable merge 失败后 force merge。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 新个体太差时 `spread_chromo()` 丢弃。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- planning time 约随 pool size 和 generations 增长。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 每个 tour 都会构造 joinrel 并估 path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 更大 pool/generation 通常提高搜索质量。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 固定 seed 提高复现但不保证最优。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 临时 context 避免 memory 累积。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- GEQO 降低 exhaustive 可解释性。

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

1. 当 join relation 太多导致动态规划不可承受时，GEQO 为什么把 join order 搜索变成遗传算法问题，它牺牲了哪些确定性和可解释性？

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

- 本节主问题：当 join relation 太多导致动态规划不可承受时，GEQO 为什么把 join order 搜索变成遗传算法问题，它牺牲了哪些确定性和可解释性？

## 17. 诊断案例切片

### 案例 1: 确认是否进入 GEQO

现象：大 join 查询 planning time 与计划形态突然变化。

源码入口：入口在 `make_rel_from_joinlist()`，比较 relation 数、`enable_geqo` 和 `geqo_threshold`。

状态变化：hook 存在时会先于 GEQO 接管。

fallback 或边界：诊断时记录 initial rel 数，而不是简单数 SQL 中表名。

诊断要点：调低 threshold 让小查询进入 GEQO。

最小实验：再调高 threshold 对照 standard DP。


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

可迁移结论：GEQO 是 join order 搜索入口分叉。

### 案例 2: seed 与计划复现

现象：同一 SQL 改 `geqo_seed` 后计划不同。

源码入口：`geqo_set_seed()` 初始化私有随机序列。

状态变化：随机性影响 tour 生成和演化路径，不影响 legality 规则。

fallback 或边界：生产诊断必须记录 seed。

诊断要点：固定 seed 重复执行，再改变 seed。

最小实验：观察 plan cost 和 join order 差异。


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

可迁移结论：GEQO 牺牲部分确定性换取 planning 上界。

### 案例 3: pool 与 generation

现象：调大 GEQO 参数后 planning time 上升。

源码入口：pool size 和 generations 决定 fitness 评估次数。

状态变化：每次评估都会构造 joinrel 并运行普通 path generation。

fallback 或边界：更大搜索通常提高找到低成本 tour 的机会，但不保证最优。

诊断要点：记录 pool size、generations 和 best worth。

最小实验：逐步提高 `geqo_effort`。


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

可迁移结论：GEQO 的成本主要花在重复评估 tour。

### 案例 4: invalid tour

现象：某些随机 tour 无法形成合法 join tree。

源码入口：`geqo_eval()` 对失败返回 `DBL_MAX`。

状态变化：`random_init_pool()` 会丢弃初始 invalid 个体。

fallback 或边界：outer join、semi join 和 LATERAL 会增加无效顺序概率。

诊断要点：统计 bad tour 数量。

最小实验：给大 join 加 LATERAL 依赖。


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

可迁移结论：随机搜索仍必须穿过 legality firewall。

### 案例 5: 评估内存隔离

现象：GEQO generations 很多，但 planner 内存没有线性堆积。

源码入口：`geqo_eval()` 创建临时 MemoryContext，并保存 `join_rel_list` 长度与 hash。

状态变化：评估结束后截断 list、恢复 hash、删除 context。

fallback 或边界：这避免每个 tour 的临时 joinrel 污染最终规划状态。

诊断要点：入口/出口打印 list length。

最小实验：构造 12 表 join 并观察 context。


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

可迁移结论：近似搜索需要明确的状态隔离。

### 案例 6: clump heuristic

现象：GEQO tour 不是机械按排列直接左深连接。

源码入口：`gimme_tree()` 把 relation 合并成 clumps，优先 desirable join。

状态变化：`desirable_join()` 看 relevant join clause 或 join order restriction。

fallback 或边界：最后 force merge 剩余 clumps，但仍要求合法。

诊断要点：观察 clump 数量如何减少。

最小实验：同一个 tour 中加入无 join predicate 的表。


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

可迁移结论：GEQO 仍利用 join graph 信息，而不是纯随机拼接。
