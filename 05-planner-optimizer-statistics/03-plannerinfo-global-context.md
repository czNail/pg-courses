# PostgreSQL PlannerInfo 与全局优化上下文

## 课程定位

前置知识：已经理解 `Query` 到 `PlannedStmt` 的转换链路，并知道 `PlannerInfo` 是 subquery_planner 创建的 per-query 工作根。

本节唯一主问题：

```text
`PlannerInfo` 为什么集中保存 rangetable、equivalence class、append relation、placeholder、join info、upper rel 等 planner 状态，它如何避免优化过程散落到各个语法节点上？
```

核心矛盾：优化器需要在多处函数中共享同一组语义推导和搜索状态，但这些状态既不能写回所有 Query 节点，也不能成为 executor 契约；集中 root 能统一 ownership、查找、失效和阶段推进。

学完后应能判断某个 planner 状态为什么应放在 `PlannerInfo`、`PlannerGlobal`、`RelOptInfo`、`RestrictInfo` 或局部变量里。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节把上一节的结构转换推进到 root 内部。后续课程会围绕 root 中最早被大量消费的 preprocessing 产物展开：表达式 canonicalize、subquery pullup、outer join reduction 和 RestrictInfo 包装。

```text
SQL text
  -> parse / analyze / rewrite
  -> Query
  -> planner preprocessing
  -> RelOptInfo / Path search
  -> Plan
  -> PlannedStmt
  -> executor
```

05 目录关注 planner 如何把语义树转成可比较的候选实现，并最终压缩成执行器契约。本节只围绕自己的唯一主问题展开，相邻主题只在解释当前状态推进时出现。

阅读时要避免从文件顶部线性背函数。更好的顺序是先确认对象生命周期，再看哪些字段被创建、填充、冻结、转移或丢弃。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
每个 Query level 有一个 `PlannerInfo root`；root 链接输入 Query、跨层 PlannerGlobal、base/join/upper rel、等价类、placeholder、join 约束和参数信息；函数之间传 root，不传一长串上下文参数。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/include/nodes/pathnodes.h`：`PlannerInfo`、`PlannerGlobal`、`RelOptInfo`、`JoinDomain`、`SpecialJoinInfo`、`PlaceHolderInfo`。 |
| 2 | `src/backend/optimizer/plan/planner.c`：`subquery_planner()` 初始化 root 字段和 upper rel 流程。 |
| 3 | `src/backend/optimizer/plan/planmain.c`：`query_planner()` 初始化 join/path 相关 root 字段。 |
| 4 | `src/backend/optimizer/plan/initsplan.c`：`build_base_rel_tlists()`、`deconstruct_jointree()`、`distribute_qual_to_rels()` 填充 root 状态。 |
| 5 | `src/backend/optimizer/util/relnode.c`：`setup_simple_rel_arrays()`、`fetch_upper_rel()` 和 RelOptInfo 创建/查找。 |
| 6 | `src/backend/optimizer/path/equivclass.c`：EquivalenceClass 如何挂在 root 并反向生成约束。 |
| 7 | `src/backend/optimizer/util/placeholder.c`：`find_placeholders_in_jointree()` 和 PlaceHolderInfo 生命周期。 |
| 8 | `src/backend/optimizer/util/appendinfo.c`：append relation 和 child rel 替换如何使用 root。 |

源码核对入口：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望分支是 `master`，提交是本课课程定位中写明的完整提交号。

读这些文件时优先看状态边界，不按文件名排序。先看入口，再看状态结构，随后看 ownership 和 cleanup，最后看观测入口。

## 4. 关键数据结构与状态边界

本节只讲帮助解释主问题的状态，不把结构体字段当百科。

| 项 | 说明 |
| --- | --- |
| root->parse | 当前 Query level 的输入树，preprocessing 会逐步改写。 |
| root->glob | 跨 Query level 输出和共享状态，尤其是 subplans、finalrtable、Param 类型和依赖。 |
| root->planner_cxt | planner-local memory context，root 下的大量 List 和 Path 都在这里自然释放。 |
| root->simple_rel_array | 按 rangetable index 快速定位 base RelOptInfo。 |
| root->join_rel_list / join_rel_hash | joinrel 查找和动态规划搜索的共享索引。 |
| root->eq_classes | 等价类推导结果，连接 join clause、pathkeys、index path 和 implied equality。 |
| root->join_info_list | outer/semi/anti join 的顺序约束和语义保护。 |
| root->upper_rels | grouping、window、distinct、ordered、final 等 upper relation 的阶段输出。 |
| root->processed_tlist | 经过 preprocessing 的目标列，后续 PathTarget 和 Plan targetlist 会引用它的语义。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：root->parse

当前 Query level 的输入树，preprocessing 会逐步改写。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：root->glob

跨 Query level 输出和共享状态，尤其是 subplans、finalrtable、Param 类型和依赖。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：root->planner_cxt

planner-local memory context，root 下的大量 List 和 Path 都在这里自然释放。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：root->simple_rel_array

按 rangetable index 快速定位 base RelOptInfo。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：root->join_rel_list / join_rel_hash

joinrel 查找和动态规划搜索的共享索引。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：root->eq_classes

等价类推导结果，连接 join clause、pathkeys、index path 和 implied equality。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：root->join_info_list

outer/semi/anti join 的顺序约束和语义保护。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 8：root->upper_rels

grouping、window、distinct、ordered、final 等 upper relation 的阶段输出。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 9：root->processed_tlist

经过 preprocessing 的目标列，后续 PathTarget 和 Plan targetlist 会引用它的语义。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| `subquery_planner()` 分配 root | 初始化 parse、glob、query_level、parent_root、planner_cxt、join_domains、upper_rels 等基础字段。 |
| CTE 与 RTE 预处理 | `SS_process_ctes()`、`preprocess_relation_rtes()` 等把 Query 中的 relation 信息转成 root 可用的 catalog / append 状态。 |
| SubLink 和 subquery pullup | 相关转换会改变 parse tree，并在 root 上记录后续需要的 Param、append relation 或 placeholder。 |
| 表达式 preprocessing | targetlist、qual、having、limit 等被规范化，结果写回 Query 或 root 的 processed 字段。 |
| `query_planner()` 重置搜索字段 | join_rel、pathkeys、join_info、placeholder、fkey、initial_rels 等字段进入 scan/join 搜索阶段。 |
| `setup_simple_rel_arrays()` | 把 rangetable index 映射到 RelOptInfo / RangeTblEntry，后续函数不再反复扫描 rtable。 |
| `deconstruct_jointree()` | 填充 all_baserels、outer_join_rels、join_info_list，并分发 RestrictInfo。 |
| `fetch_upper_rel()` | upper planning 阶段按 UPPERREL_* 在 root->upper_rels 中取或建 RelOptInfo。 |
| `set_plan_references()` | 最终从 root->glob 收集 flat rtable、subplans 和依赖，把 root 的临时状态折叠出去。 |

### 5.1. `subquery_planner()` 分配 root

初始化 parse、glob、query_level、parent_root、planner_cxt、join_domains、upper_rels 等基础字段。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. CTE 与 RTE 预处理

`SS_process_ctes()`、`preprocess_relation_rtes()` 等把 Query 中的 relation 信息转成 root 可用的 catalog / append 状态。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. SubLink 和 subquery pullup

相关转换会改变 parse tree，并在 root 上记录后续需要的 Param、append relation 或 placeholder。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. 表达式 preprocessing

targetlist、qual、having、limit 等被规范化，结果写回 Query 或 root 的 processed 字段。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. `query_planner()` 重置搜索字段

join_rel、pathkeys、join_info、placeholder、fkey、initial_rels 等字段进入 scan/join 搜索阶段。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. `setup_simple_rel_arrays()`

把 rangetable index 映射到 RelOptInfo / RangeTblEntry，后续函数不再反复扫描 rtable。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. `deconstruct_jointree()`

填充 all_baserels、outer_join_rels、join_info_list，并分发 RestrictInfo。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `fetch_upper_rel()`

upper planning 阶段按 UPPERREL_* 在 root->upper_rels 中取或建 RelOptInfo。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.9. `set_plan_references()`

最终从 root->glob 收集 flat rtable、subplans 和依赖，把 root 的临时状态折叠出去。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

## 6. 生命周期 / ownership / cleanup

planner 的 ownership 边界比 executor 更短，但更容易被误用。

| 项 | 说明 |
| --- | --- |
| 创建者 | 当前 Query level 的主要工作状态通常由 `subquery_planner()` 或它调用的初始化函数创建。 |
| 持有者 | `PlannerInfo root` 持有本层 planner 状态，`PlannerGlobal glob` 持有跨层结果。 |
| 释放边界 | 多数对象位于 planner 所在 memory context，规划完成后整体释放，不需要 executor 逐个清理。 |
| 转移边界 | 只有进入 `Plan`、`PlannedStmt`、flat rtable、subplans、rowmarks、invalItems 的状态会越过规划阶段。 |
| 异常路径 | planner 中途 ERROR 时依赖 backend 的 memory context 和 ResourceOwner 清理，不依赖手工释放每个 List 或 Node。 |

这也是为什么扩展 hook 不能把 root 内部指针保存到长期结构里。规划结束后，指针地址可能还像是有效 C 指针，但语义和内存所有权已经结束。

## 7. 正确性机制层次

优化器正确性不是一个函数单独保证的，而是多层边界共同保证。

| 项 | 说明 |
| --- | --- |
| 1 | `PlannerInfo` 是 backend-local 临时状态，不能跨 session、不能进入 PlannedStmt、不能被 executor 保存。 |
| 2 | `PlannerGlobal` 与 `PlannerInfo` 的边界是跨 Query level 和单 Query level 的边界，不是全局变量和局部变量的简单区别。 |
| 3 | `simple_rel_array` 以 rangetable index 为索引，因此 Query 改写期间新增 RTE 后必须重新保持数组一致。 |
| 4 | 等价类、placeholder、SpecialJoinInfo 等状态必须集中在 root，否则 join search 无法在不同子问题之间共享约束。 |
| 5 | root 的字段多数只在特定阶段有效，不能把任意字段当作全程不变量。 |

可以把本节正确性压缩成下面的判断链：

```text
语义是否等价？
状态是否在正确阶段产生？
候选是否只影响搜索空间？
最终 Plan 是否脱离 planner-local 指针？
PlannedStmt 是否包含 executor 和 plan cache 需要的依赖？
```

## 8. 错误路径 / 异常路径 / fallback

planner 的 fallback 往往不是抛错，而是保守地缩小改写、缩小搜索空间，或保留较笨但语义可靠的计划形态。

| 项 | 说明 |
| --- | --- |
| 1 | 如果 pullup 新增 RTE 或 appendrel 后没有更新 root 相关数组，后续 RelOptInfo 查找会错位。 |
| 2 | 如果在 placeholdersFrozen 之后再制造 PlaceHolderInfo，outer join 约束可能漏掉所需变量。  |
| 3 | 如果扩展在 hook 中缓存 root 指针到规划结束后使用，内存和语义都会失效。 |
| 4 | 如果把 `PlannerGlobal` 状态误写到当前 root，子查询和 initplan 的输出契约会丢失。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果 pullup 新增 RTE 或 appendrel 后没有更新 root 相关数组，后续 RelOptInfo 查找会错位。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果在 placeholdersFrozen 之后再制造 PlaceHolderInfo，outer join 约束可能漏掉所需变量。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果扩展在 hook 中缓存 root 指针到规划结束后使用，内存和语义都会失效。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果把 `PlannerGlobal` 状态误写到当前 root，子查询和 initplan 的输出契约会丢失。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | 集中 root 减少函数参数数量，也让频繁访问 rangetable、RelOptInfo、EquivalenceClass 的路径更短。 |
| 2 | `join_rel_hash` 只在 joinrel 较多时体现价值，小查询通常直接扫 list 即可。 |
| 3 | root 持有大量候选 Path 和 List，规划期间内存可能较高，但规划结束可以整体释放。 |
| 4 | 全局上下文越集中，越容易做 hook 和诊断，但也要求读源码时理解字段有效期。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | gdb 中打印 `root->query_level` 可以区分顶层 Query 和子查询规划。 |
| 2 | 打印 `root->simple_rel_array_size`、`root->all_baserels`、`root->join_info_list` 能确认 jointree 分解状态。 |
| 3 | 打印 `root->eq_classes` 可以解释为什么额外 join clause 或 pathkeys 被推导出来。 |
| 4 | `EXPLAIN` 看不到 root，但 final Plan 的 join order、pathkeys 和 row estimate 是 root 状态折叠后的结果。 |

推荐的最小诊断路径：

```text
EXPLAIN (ANALYZE, VERBOSE, BUFFERS)
  -> 找到第一个估算或节点选择异常的位置
  -> 回到对应 planner 阶段
  -> 在源码入口断点观察 root / rel / path / rinfo
  -> 用 GUC 或 SQL 最小化验证假设
```

## 11. 常见误区

- 误区 1：把最终 Plan 当成 planner 搜索过程的完整记录。
- 误区 2：把 enable_* GUC 当成绝对语义开关。
- 误区 3：把 cost 数字当成毫秒。
- 误区 4：把 Query、Path、Plan、PlannedStmt 混成同一个生命周期。
- 误区 5：看到某个函数名就认为它承担全部阶段职责。
- 误区 6：忽略 outer join、security barrier、volatile、lateral 这类语义边界。

这些误区的共同原因是只看静态结构，不看状态随阶段推进的时间线。

## 12. 课堂实验

### 实验 1：root 初始化

断在 `subquery_planner()` 创建 root 后，观察 `planner_cxt`、`query_level`、`parent_root`。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：RelOptInfo 数组

断在 `setup_simple_rel_arrays()` 和 `add_base_rels_to_query()`，查看 simple_rel_array 的填充时机。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：等价类

执行 `a.x=b.x AND b.x=c.x`，在 `generate_base_implied_equalities()` 后观察 `root->eq_classes`。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：upper rel

执行 GROUP BY + ORDER BY + LIMIT，观察 `root->upper_rels` 中哪些阶段被创建。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 如果不用 root，而把状态挂在 Query 节点上，子查询 pullup 和 join search 会遇到什么问题？

2. 哪些状态应该进 PlannerGlobal，而不是 PlannerInfo？

3. root 的集中设计给扩展 hook 带来哪些便利和风险？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/include/nodes/pathnodes.h`

`PlannerInfo`、`PlannerGlobal`、`RelOptInfo`、`JoinDomain`、`SpecialJoinInfo`、`PlaceHolderInfo`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/backend/optimizer/plan/planner.c`

`subquery_planner()` 初始化 root 字段和 upper rel 流程。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/backend/optimizer/plan/planmain.c`

`query_planner()` 初始化 join/path 相关 root 字段。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/initsplan.c`

`build_base_rel_tlists()`、`deconstruct_jointree()`、`distribute_qual_to_rels()` 填充 root 状态。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/util/relnode.c`

`setup_simple_rel_arrays()`、`fetch_upper_rel()` 和 RelOptInfo 创建/查找。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/backend/optimizer/path/equivclass.c`

EquivalenceClass 如何挂在 root 并反向生成约束。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/backend/optimizer/util/placeholder.c`

`find_placeholders_in_jointree()` 和 PlaceHolderInfo 生命周期。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/backend/optimizer/util/appendinfo.c`

append relation 和 child rel 替换如何使用 root。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. `subquery_planner()` 分配 root

初始化 parse、glob、query_level、parent_root、planner_cxt、join_domains、upper_rels 等基础字段。

进入前先确认 `root->parse` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->glob` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. CTE 与 RTE 预处理

`SS_process_ctes()`、`preprocess_relation_rtes()` 等把 Query 中的 relation 信息转成 root 可用的 catalog / append 状态。

进入前先确认 `root->glob` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->planner_cxt` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. SubLink 和 subquery pullup

相关转换会改变 parse tree，并在 root 上记录后续需要的 Param、append relation 或 placeholder。

进入前先确认 `root->planner_cxt` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->simple_rel_array` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. 表达式 preprocessing

targetlist、qual、having、limit 等被规范化，结果写回 Query 或 root 的 processed 字段。

进入前先确认 `root->simple_rel_array` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->join_rel_list / join_rel_hash` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. `query_planner()` 重置搜索字段

join_rel、pathkeys、join_info、placeholder、fkey、initial_rels 等字段进入 scan/join 搜索阶段。

进入前先确认 `root->join_rel_list / join_rel_hash` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->eq_classes` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. `setup_simple_rel_arrays()`

把 rangetable index 映射到 RelOptInfo / RangeTblEntry，后续函数不再反复扫描 rtable。

进入前先确认 `root->eq_classes` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->join_info_list` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. `deconstruct_jointree()`

填充 all_baserels、outer_join_rels、join_info_list，并分发 RestrictInfo。

进入前先确认 `root->join_info_list` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->upper_rels` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `fetch_upper_rel()`

upper planning 阶段按 UPPERREL_* 在 root->upper_rels 中取或建 RelOptInfo。

进入前先确认 `root->upper_rels` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->processed_tlist` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.9. `set_plan_references()`

最终从 root->glob 收集 flat rtable、subplans 和依赖，把 root 的临时状态折叠出去。

进入前先确认 `root->processed_tlist` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `root->parse` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

## 16. 现象到源码的回溯路径

面对慢 SQL 或计划异常时，可以从可观察现象倒推源码阶段。

| 项 | 说明 |
| --- | --- |
| 最终节点类型变化 | 先比较 enable_* 和 cost GUC，再回到 Path 生成与 add_path 剪枝。 |
| 估算行数偏差 | 先定位第一个 rows 失真节点，再回到 RestrictInfo、统计信息和 selectivity。 |
| Subquery Scan 出现 | 先检查 pullup 阻止条件，再看 security barrier、limit、聚合或 lateral。 |
| Join Filter 没有下推 | 先检查 outer join 边界、required_relids、security_level 和 is_pushed_down。 |
| 计划无法并行 | 先看 parallel hazard、Path 的 parallel_safe 和最终 Gather 边界。 |
| 索引没有使用 | 先确认 qual 是否进入 baserestrictinfo，再看 operator family、collation 和 index path 匹配。 |
| 计划缓存失效频繁 | 先看 PlannedStmt 中 relationOids、invalItems、dependsOnRole。 |
| 扩展 hook 改变计划 | 先确认 hook 插入的是 Path 候选、join search，还是替换了 whole planner。 |

### 16.1. 最终节点类型变化

先比较 enable_* 和 cost GUC，再回到 Path 生成与 add_path 剪枝。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.2. 估算行数偏差

先定位第一个 rows 失真节点，再回到 RestrictInfo、统计信息和 selectivity。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.3. Subquery Scan 出现

先检查 pullup 阻止条件，再看 security barrier、limit、聚合或 lateral。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.4. Join Filter 没有下推

先检查 outer join 边界、required_relids、security_level 和 is_pushed_down。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.5. 计划无法并行

先看 parallel hazard、Path 的 parallel_safe 和最终 Gather 边界。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.6. 索引没有使用

先确认 qual 是否进入 baserestrictinfo，再看 operator family、collation 和 index path 匹配。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.7. 计划缓存失效频繁

先看 PlannedStmt 中 relationOids、invalItems、dependsOnRole。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

### 16.8. 扩展 hook 改变计划

先确认 hook 插入的是 Path 候选、join search，还是替换了 whole planner。

回溯时保留一个最小 SQL，不要同时修改统计信息、GUC、索引和 SQL 结构，否则很难判断触发条件。

确认源码阶段以后，再决定是修 SQL、修统计信息、调成本参数、加索引，还是接受该语义边界。

## 17. 源码阅读检查清单

- 检查点 1：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/backend/optimizer/plan/planmain.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/initsplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/util/relnode.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/backend/optimizer/path/equivclass.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/backend/optimizer/util/placeholder.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/backend/optimizer/util/appendinfo.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/backend/optimizer/plan/planmain.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/initsplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/util/relnode.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/backend/optimizer/path/equivclass.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/backend/optimizer/util/placeholder.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/backend/optimizer/util/appendinfo.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/backend/optimizer/plan/planmain.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/initsplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/util/relnode.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/backend/optimizer/path/equivclass.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/backend/optimizer/util/placeholder.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/backend/optimizer/util/appendinfo.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/include/nodes/pathnodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/backend/optimizer/plan/planmain.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/initsplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/util/relnode.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/backend/optimizer/path/equivclass.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/backend/optimizer/util/placeholder.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/backend/optimizer/util/appendinfo.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/include/nodes/pathnodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/backend/optimizer/plan/planmain.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/initsplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/util/relnode.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/backend/optimizer/path/equivclass.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/backend/optimizer/util/placeholder.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/backend/optimizer/util/appendinfo.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/include/nodes/pathnodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/backend/optimizer/plan/planmain.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/initsplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/util/relnode.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/backend/optimizer/path/equivclass.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/backend/optimizer/util/placeholder.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/backend/optimizer/util/appendinfo.c` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

`PlannerInfo` 是 backend-local 临时状态，不能跨 session、不能进入 PlannedStmt、不能被 executor 保存。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.2. 正确性边界

`PlannerGlobal` 与 `PlannerInfo` 的边界是跨 Query level 和单 Query level 的边界，不是全局变量和局部变量的简单区别。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.3. 正确性边界

`simple_rel_array` 以 rangetable index 为索引，因此 Query 改写期间新增 RTE 后必须重新保持数组一致。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planmain.c`。

### 18.4. 正确性边界

等价类、placeholder、SpecialJoinInfo 等状态必须集中在 root，否则 join search 无法在不同子问题之间共享约束。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/initsplan.c`。

### 18.5. 正确性边界

root 的字段多数只在特定阶段有效，不能把任意字段当作全程不变量。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/util/relnode.c`。

### 18.6. 异常或 fallback

如果 pullup 新增 RTE 或 appendrel 后没有更新 root 相关数组，后续 RelOptInfo 查找会错位。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/path/equivclass.c`。

### 18.7. 异常或 fallback

如果在 placeholdersFrozen 之后再制造 PlaceHolderInfo，outer join 约束可能漏掉所需变量。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/util/placeholder.c`。

### 18.8. 异常或 fallback

如果扩展在 hook 中缓存 root 指针到规划结束后使用，内存和语义都会失效。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/util/appendinfo.c`。

### 18.9. 异常或 fallback

如果把 `PlannerGlobal` 状态误写到当前 root，子查询和 initplan 的输出契约会丢失。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.10. 成本传播

集中 root 减少函数参数数量，也让频繁访问 rangetable、RelOptInfo、EquivalenceClass 的路径更短。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.11. 成本传播

`join_rel_hash` 只在 joinrel 较多时体现价值，小查询通常直接扫 list 即可。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/planmain.c`。

### 18.12. 成本传播

root 持有大量候选 Path 和 List，规划期间内存可能较高，但规划结束可以整体释放。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/initsplan.c`。

### 18.13. 成本传播

全局上下文越集中，越容易做 hook 和诊断，但也要求读源码时理解字段有效期。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/util/relnode.c`。

### 18.14. 观测入口

gdb 中打印 `root->query_level` 可以区分顶层 Query 和子查询规划。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/path/equivclass.c`。

### 18.15. 观测入口

打印 `root->simple_rel_array_size`、`root->all_baserels`、`root->join_info_list` 能确认 jointree 分解状态。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/util/placeholder.c`。

### 18.16. 观测入口

打印 `root->eq_classes` 可以解释为什么额外 join clause 或 pathkeys 被推导出来。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/util/appendinfo.c`。

### 18.17. 观测入口

`EXPLAIN` 看不到 root，但 final Plan 的 join order、pathkeys 和 row estimate 是 root 状态折叠后的结果。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/nodes/pathnodes.h`。

## 19. 本节小结

- `PlannerInfo` 是优化器的 per-query 工作根，不是最终计划的一部分。
- 它的价值在于集中保存跨函数共享、但只在规划期间有效的状态。
- 读 planner 源码时，先确认当前函数读写 root 的哪一组字段，再判断它属于 preprocessing、join search 还是 upper planning。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
