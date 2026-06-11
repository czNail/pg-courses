# PostgreSQL index path 的 clause、ORDER BY 与参数化匹配

## 课程定位

前置知识：已经理解 base pathlist 的候选竞争，也知道 `RestrictInfo`、EquivalenceClass 和 PathKey 会把 SQL 条件转成 planner 状态。

本节唯一主问题：

planner 如何判断一个 predicate 或 ORDER BY 需求能被某个索引使用，为什么同一个索引既可能用于过滤，也可能用于输出顺序、parameterized nested loop 和 index-only scan？

核心矛盾：索引是访问方法暴露的能力集合，不是列名相同就可用；planner 必须在 operator family、collation、表达式、partial predicate、排序语义和外部参数之间保守匹配。

学完后应能判断：面对有索引却没用的计划，能区分 predicate 不匹配、partial predicate 不成立、ORDER BY 不能映射、index-only 不满足、bitmap 被选择，还是 parameterized path 只能在特定 join order 中使用。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节解释普通 base rel 如何把 index path 和其它 scan path 放进同一个 pathlist。

05 目录的主线是：SQL 语义树先被规范化为 planner-local 搜索状态，再经过 rows、cost、path 和 join order 的连续压缩，最后转成 executor 能执行的 `Plan` tree。

本节只处理一个主问题；相邻机制只在解释这条状态链时出现。

下一节会讨论 partition/append、FDW、tablesample、subquery 和 custom scan 如何复用同一个框架。

阅读时从 `IndexOptInfo` 追到 `IndexClauseSet`、`IndexClause`、pathkeys 和 parameterized path；索引是否可用不是一个布尔判断，而是一组能力匹配。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：`set_plain_rel_pathlist()` 调用 `create_index_paths()`；`create_index_paths()` 遍历 `rel->indexlist`；partial index 若 `indpred != NIL && !predOK` 被跳过；`match_restriction_clauses_to_index()` 匹配 base restriction；`get_index_paths()` 生成 plain index path 和 bitmap path；`match_join_clauses_to_index()` 收集参数化 join clause；`match_eclass_clauses_to_index()` 从 EC 派生条件；`consider_index_join_clauses()` 生成 parameterized index path；`generate_bitmap_or_paths()` 处理 OR；`match_pathkeys_to_index()` 处理排序需求。

1. `set_plain_rel_pathlist()` 调用 `create_index_paths()`
   进入这里时 seqscan 已经作为基础候选加入，索引路径是在同一 `rel->pathlist` 里补充更具体的访问能力。

2. `create_index_paths()` 遍历 `rel->indexlist`
   每个 `IndexOptInfo` 都带着 access method 能力、opfamily、collation、predicate 和 INCLUDE/覆盖信息，不能只按列名判断可用性。

3. partial index 若 `indpred != NIL && !predOK` 被跳过
   这是正确性过滤：查询条件不能推出 predicate 时，使用 partial index 可能漏行，所以连成本比较都不会发生。

4. `match_restriction_clauses_to_index()` 匹配 base restriction
   匹配结果按 index column 放进 `IndexClauseSet`，每个 `IndexClause` 记录原始 `RestrictInfo`、派生 indexqual 和对应列号。

5. `get_index_paths()` 生成 plain index path 和 bitmap path
   `build_index_paths()` 会按 index key 顺序组装 clauses，同时计算 pathkeys、index-only 可能性和 bitmap/native SAOP 等访问形态。

6. `match_join_clauses_to_index()` 收集参数化 join clause
   这些 clause 不能直接用于独立扫描，但可以让该索引成为 nested loop inner side 的 parameterized path。

7. `match_eclass_clauses_to_index()` 从 EC 派生条件
   下游不会重新解析 SQL 原文，只会消费这里留下的状态。

8. `consider_index_join_clauses()` 生成 parameterized index path
   它把 restriction、join clause 和 EC 派生 clause 组合成不同 `required_outer` 的索引候选，供 join order 搜索选择。

9. `generate_bitmap_or_paths()` 处理 OR
   OR 不是简单拆成多个普通 IndexPath，而是先构造 BitmapOr/BitmapAnd 候选，再由 bitmap heap path 回表。

10. `match_pathkeys_to_index()` 处理排序需求
   ORDER BY 需要 PathKey、opfamily、排序方向、NULLS 规则和 AM ordering 能力同时匹配；前缀匹配还能给 incremental sort 留机会。

这条链路的重点是：索引路径既可能提供过滤，也可能提供顺序、bitmap 组合、index-only 覆盖或外部参数化访问。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/path/indxpath.c` | `create_index_paths()`、`match_clause_to_indexcol()`、`build_index_paths()`。 |
| 2 | `src/include/nodes/pathnodes.h` | `IndexOptInfo`、`IndexClause`、`PathKey`、`ParamPathInfo`。 |
| 3 | `src/backend/optimizer/util/plancat.c` | 从 catalog 构造 `IndexOptInfo`。 |
| 4 | `src/backend/optimizer/util/pathnode.c` | `create_index_path()`、bitmap path 构造。 |
| 5 | `src/backend/optimizer/path/equivclass.c` | EquivalenceClass 派生 index join clause。 |
| 6 | `src/backend/optimizer/path/costsize.c` | `cost_index()` 和 bitmap cost。 |
| 7 | `src/include/access/amapi.h` | 访问方法能力边界。 |

推荐阅读顺序是入口、状态结构、判断条件、fallback、观测入口；不要按文件名排序，也不要把表中职责翻译成 API 清单。

## 4. 从可观测现象进入源码

### 现象：表达式写法变化导致索引失效

源码解释：`match_clause_to_indexcol()` 检查形态、opfamily、collation

验证办法：比较 `lower(name)` 索引。

表达式索引诊断要比较 parse tree 形态和 collation；看起来等价的 SQL 字符串不一定能生成同一个 `IndexClause`。

### 现象：partial index 没被考虑

源码解释：`predOK` false 时跳过

验证办法：改写 WHERE 让 predicate 可推出。

partial index 首先看谓词证明是否成立；只有 `predOK` 为 true，后续 opfamily、选择率和成本才有讨论意义。

### 现象：ORDER BY 没走索引

源码解释：pathkeys 必须可映射到索引顺序

验证办法：改变 collation 或方向。

ORDER BY 诊断要看 `query_pathkeys` 与 `build_index_pathkeys()` 或 `match_pathkeys_to_index()` 的匹配长度，不只看索引列顺序。

### 现象：bitmap 赢过 plain index

源码解释：多个低选择率条件组合时 bitmap 可能更便宜

验证办法：两列单独索引加 AND。

Bitmap 胜出通常说明多个索引条件的组合价值高于单个有序 IndexPath；它牺牲输出顺序，换取更低回表成本。

### 现象：index-only scan 没出现

源码解释：`check_index_only()` 要求所需列被覆盖

验证办法：添加 INCLUDE 列。

index-only scan 还受目标列、recheck qual、visibility map 成本影响；列被索引覆盖只是进入候选的必要条件。

### 现象：join 中才出现 index scan

源码解释：parameterized index path 需要外部 rel 值

验证办法：构造小表驱动大表。

这类 path 只有在对应外部 rel 已经位于 outer side 时可用，所以单表 EXPLAIN 或错误 join order 下看不到它。

## 5. 关键数据结构与状态边界

| 状态 | 语义 | 写入或持有者 | 主要消费者 |
| --- | --- | --- | --- |
| `rel->indexlist` | 可考虑索引列表 | plancat | `create_index_paths()` |
| `IndexOptInfo.indpred` | partial index predicate | catalog | predicate 证明 |
| `IndexOptInfo.predOK` | predicate 是否被查询蕴含 | `check_index_predicates()` | partial index 过滤 |
| `IndexClauseSet` | 按 index column 分组的匹配 clause | match_* 函数 | `build_index_paths()` |
| `IndexClause` | clause 到 index column 的匹配结果 | `match_clause_to_indexcol()` | cost 和 executor qual |
| `amcanorderbyop` | AM 是否支持 ordering operator | plancat | ORDER BY path |
| `amoptionalkey` | 是否允许无前导 key 扫描 | plancat | path 构造 |
| `ParamPathInfo` | parameterized path 的外部依赖 | join clause index match | nestloop inner |

### `rel->indexlist`

语义：可考虑索引列表

写入或持有者：plancat

主要消费者：`create_index_paths()`

诊断提示：为空则无 index path

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `IndexOptInfo.indpred`

语义：partial index predicate

写入或持有者：catalog

主要消费者：predicate 证明

诊断提示：partial index 首先是正确性问题

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `IndexOptInfo.predOK`

语义：predicate 是否被查询蕴含

写入或持有者：`check_index_predicates()`

主要消费者：partial index 过滤

诊断提示：不能靠 GUC 修正

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `IndexClauseSet`

语义：按 index column 分组的匹配 clause

写入或持有者：match_* 函数

主要消费者：`build_index_paths()`

诊断提示：多列索引依赖列序

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `IndexClause`

语义：clause 到 index column 的匹配结果

写入或持有者：`match_clause_to_indexcol()`

主要消费者：cost 和 executor qual

诊断提示：不是字符串匹配

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `amcanorderbyop`

语义：AM 是否支持 ordering operator

写入或持有者：plancat

主要消费者：ORDER BY path

诊断提示：不同 AM 语义不同

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `amoptionalkey`

语义：是否允许无前导 key 扫描

写入或持有者：plancat

主要消费者：path 构造

诊断提示：不能套用 btree 经验

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

### `ParamPathInfo`

语义：parameterized path 的外部依赖

写入或持有者：join clause index match

主要消费者：nestloop inner

诊断提示：受 join order 限制

单独打印这个字段通常不够，需要同时记录 relids、阶段、path 类型和调用栈。

## 6. 主流程源码 walkthrough

### 1. `set_plain_rel_pathlist()` 调用 `create_index_paths()`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 2. `create_index_paths()` 遍历 `rel->indexlist`

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 3. partial index 若 `indpred != NIL && !predOK` 被跳过

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 4. `match_restriction_clauses_to_index()` 匹配 base restriction

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 5. `get_index_paths()` 生成 plain index path 和 bitmap path

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 6. `match_join_clauses_to_index()` 收集参数化 join clause

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 7. `match_eclass_clauses_to_index()` 从 EC 派生条件

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 8. `consider_index_join_clauses()` 生成 parameterized index path

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 9. `generate_bitmap_or_paths()` 处理 OR

输入状态：来自上一阶段已经写好的 planner-local 字段、列表或 GUC。

源码动作：当前函数读取这些状态，并决定继续、剪枝、fallback 或写入新的候选。

输出状态：下一个阶段不会重新理解 SQL 原文，而是读取这里留下的字段、pathlist 或 joinrel。

断点价值：在这一点比较进入前和返回后的字段，能把最终计划现象还原成源码状态变化。

### 10. `match_pathkeys_to_index()` 处理排序需求

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

- operator family 必须支持该比较语义。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- collation 必须匹配，尤其是文本排序。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- partial predicate 必须被当前查询保证。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- lossy index qual 需要 executor recheck。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parameterized path 的外部变量必须已可用。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- index-only 的覆盖判断不等于运行时一定不访问 heap。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 9. 错误路径 / 异常路径 / fallback

- clause 不匹配时保留 seqscan 和其它 path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- partial predicate 不成立时跳过该 index。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- ORDER BY 不匹配时上层仍可 Sort。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- index-only 不满足时退回普通 index scan。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- OR 不能拆成 bitmap 时作为普通 qual 保留。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- parameterization 不可用时 join search 不会使用该 path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

## 10. 成本、资源与跨模块传播

- 选择率影响 index rows 和 heap fetch。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 相关性影响随机 I/O 成本。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- loop count 会放大 parameterized path 成本。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- bitmap build 降低 heap 访问但增加 bitmap 成本。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- 排序收益可能保留局部更贵的 index path。

  这一点要回扣本节主流程：它要么保护 SQL 语义，要么控制搜索空间，要么把成本信息传播给下一阶段。

- index-only 收益依赖 visibility map 和覆盖列。

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

1. planner 如何判断一个 predicate 或 ORDER BY 需求能被某个索引使用，为什么同一个索引既可能用于过滤，也可能用于输出顺序、parameterized nested loop 和 index-only scan？

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

- 本节主问题：planner 如何判断一个 predicate 或 ORDER BY 需求能被某个索引使用，为什么同一个索引既可能用于过滤，也可能用于输出顺序、parameterized nested loop 和 index-only scan？

## 17. 诊断案例切片

### 案例 1: 有索引但 predicate 不匹配

现象：列上有索引，WHERE 看起来也用了该列，但没有 Index Scan。

源码入口：入口是 `create_index_paths()`，真正的判断在 `match_clause_to_indexcol()`。

状态变化：它检查 expression shape、operator family、collation 和 index column。

fallback 或边界：函数包裹、类型转换或 collation 差异都可能让匹配失败。

诊断要点：断点应记录 clause node type 和 indexcol。

最小实验：可用实验是比较 `name = x` 与 `lower(name) = x`。


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

可迁移结论：这个案例说明 index match 不是文本层面的列名匹配。

### 案例 2: partial index 的 predOK 门槛

现象：partial index 存在，但整个 index 被跳过。

源码入口：`create_index_paths()` 对 `indpred != NIL && !predOK` 直接 continue。

状态变化：`predOK` 来自更早的 `check_index_predicates()`。

fallback 或边界：如果 WHERE 不能证明 predicate 成立，使用该 index 会破坏正确性。

诊断要点：诊断时先打印 predicate implication 结果，再看成本。

最小实验：可用实验是把隐含条件写成更直接的等价形式。


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

可迁移结论：这个案例说明 partial index 的第一问题是语义安全。

### 案例 3: ORDER BY 与 index ordering

现象：过滤走了索引，但 ORDER BY 仍然需要 Sort。

源码入口：`match_pathkeys_to_index()` 尝试把 pathkeys 映射到 index ordering。

状态变化：排序方向、NULLS 顺序、collation 和 AM 能力都会影响结果。

fallback 或边界：`amcanorderbyop` 为真的 AM 还会走 ordering operator 逻辑。

诊断要点：断点时打印 pathkey 的 EC 和 index column。

最小实验：可用实验是改变索引 collation 或 ORDER BY 方向。


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

可迁移结论：这个案例说明过滤可用索引不等于顺序也可用。

### 案例 4: bitmap 不是没有使用索引

现象：计划显示 Bitmap Heap Scan，没有普通 Index Scan。

源码入口：多个 index bitmap path 先被收集，再由 `choose_bitmap_and()` 选择组合。

状态变化：最终 heap 访问与 index bitmap 构造是两个层次。

fallback 或边界：多个条件组合时 bitmap 可能比单个 index scan 更便宜。

诊断要点：诊断时查 BitmapAnd/BitmapOr 的子路径。

最小实验：可用实验是两个单列索引加 AND/OR 条件。


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

可迁移结论：这个案例说明 index 的使用形态不只有 Index Scan。

### 案例 5: parameterized index path

现象：join 中 inner 表使用外部表的值做索引条件。

源码入口：join clause 匹配 index 时会生成 parameterized path。

状态变化：`ParamPathInfo` 限定 required_outer，因此只能在特定 join order 中执行。

fallback 或边界：这类 path 往往驱动 nested loop，而不是 hash join。

诊断要点：断点时打印 join clause 和 required_outer。

最小实验：可用实验是小表连接大表，改变小表选择率。


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

可迁移结论：这个案例说明 index path 也能表达 join order 依赖。

### 案例 6: index-only 的 planner 与 executor 边界

现象：索引覆盖了 SELECT 列，但实际仍有 heap fetch。

源码入口：planner 的 `check_index_only()` 只判断所需属性是否被 index 覆盖。

状态变化：运行时是否访问 heap 还受 visibility map 影响。

fallback 或边界：不要把 Index Only Scan 的 heap fetch 当成 planner match 失败。

诊断要点：诊断时同时看 INCLUDE 列、目标列和 visibility map 状态。

最小实验：可用实验是 VACUUM 前后比较 heap fetch。


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

可迁移结论：这个案例说明 planner 判断可行性，executor 处理可见性。
