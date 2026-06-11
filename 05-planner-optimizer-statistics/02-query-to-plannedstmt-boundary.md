# PostgreSQL Query 树到 PlannedStmt 的结构转换

## 课程定位

前置知识：已经理解 optimizer 被分成多个阶段，也知道 executor 最终拿到的是 `PlannedStmt` 而不是 parser 的 `Query`。

本节唯一主问题：

```text
`Query`、`PlannerInfo`、`Plan` 和 `PlannedStmt` 分别保存什么状态，为什么 optimizer 不能直接在 `Query` 上生成可执行节点，而要先构造 planner-local 的搜索上下文？
```

核心矛盾：Query 树需要保留 SQL 语义和重写结果，执行计划需要变成稳定、扁平、可执行、可失效的契约；优化期间又需要大量临时状态。如果把三类状态混在一个结构里，最终计划会携带过多搜索垃圾，或者 executor 会依赖 optimizer 内部细节。

学完后应能判断一个字段属于输入语义、planner 临时状态、Path 搜索结果、Plan 运行契约，还是 PlannedStmt 顶层依赖和失效信息。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节承接入口阶段边界，专门解释结构转换。下一节会深入 `PlannerInfo` 这个工作台内部，说明为什么大量优化状态必须集中在 root 上，而不是散落在 Query 节点或 Path 节点中。

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
Query 是语义输入；PlannerInfo 是当前 Query level 的工作台；Path 是候选实现；Plan 是选中候选的执行器树；PlannedStmt 把 Plan tree 与 flat rtable、subplans、rowmarks、invalidation 和 Param 类型组合成 executor 可消费的完整契约。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/include/nodes/parsenodes.h`：`Query`、`RangeTblEntry`、`FromExpr`、`JoinExpr` 等输入语义结构。 |
| 2 | `src/include/nodes/pathnodes.h`：`PlannerGlobal`、`PlannerInfo`、`RelOptInfo`、`Path`、`RestrictInfo` 等 planner-local 状态。 |
| 3 | `src/include/nodes/plannodes.h`：`PlannedStmt`、`Plan` 和具体 Plan node 的执行契约。 |
| 4 | `src/backend/optimizer/plan/planner.c`：`standard_planner()` 构造 `PlannerGlobal`、调用 `subquery_planner()`、填充 `PlannedStmt`。 |
| 5 | `src/backend/optimizer/plan/createplan.c`：`create_plan()` 和各类 `create_*_plan()` 从 Path 生成 Plan。 |
| 6 | `src/backend/optimizer/plan/setrefs.c`：`set_plan_references()` 让 Plan 脱离 planner-local 指针。 |
| 7 | `src/backend/optimizer/plan/subselect.c`：SubPlan / InitPlan 如何进入 `PlannerGlobal` 和 `PlannedStmt`。 |
| 8 | `src/include/utils/plancache.h`：计划缓存如何保存并失效 PlannedStmt。 |

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
| Query | 保留 SQL 级结构：commandType、rtable、jointree、targetList、havingQual、windowClause、setOperations、limit 等。 |
| PlannerGlobal | 跨 Query level 的结果收集器：subplans、finalrtable、finalrowmarks、paramExecTypes、relationOids、invalItems。 |
| PlannerInfo | 当前 Query level 的工作根：parse、glob、simple_rel_array、join_rel_list、eq_classes、upper_rels、processed_tlist。 |
| RelOptInfo | 某个 base rel、joinrel 或 upper rel 的候选路径容器。 |
| Path | 候选实现，保留 planner 比较需要的信息，不要求 executor 能直接运行。 |
| Plan | 执行器节点树，包含 targetlist、qual、lefttree、righttree、cost 和 row estimate。 |
| PlannedStmt | 最终输出，连接 Plan tree、flat rtable、subplans、Param 类型、rowMarks、权限信息和依赖失效信息。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：Query

保留 SQL 级结构：commandType、rtable、jointree、targetList、havingQual、windowClause、setOperations、limit 等。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：PlannerGlobal

跨 Query level 的结果收集器：subplans、finalrtable、finalrowmarks、paramExecTypes、relationOids、invalItems。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：PlannerInfo

当前 Query level 的工作根：parse、glob、simple_rel_array、join_rel_list、eq_classes、upper_rels、processed_tlist。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：RelOptInfo

某个 base rel、joinrel 或 upper rel 的候选路径容器。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：Path

候选实现，保留 planner 比较需要的信息，不要求 executor 能直接运行。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：Plan

执行器节点树，包含 targetlist、qual、lefttree、righttree、cost 和 row estimate。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：PlannedStmt

最终输出，连接 Plan tree、flat rtable、subplans、Param 类型、rowMarks、权限信息和依赖失效信息。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| `standard_planner()` 初始化 `PlannerGlobal` | 此时还没有 Plan，只有跨层共享的收集容器和参数输入。 |
| `subquery_planner()` 创建 `PlannerInfo` | root 指向当前 Query，后续改写和搜索都在 root 上沉淀临时状态。 |
| preprocessing 改写 Query | 子查询 pullup、表达式 canonicalize、outer join reduction 会直接修改 parse tree，但目标是让后续搜索语义更清晰。 |
| `query_planner()` 生成 `RelOptInfo` 和 `Path` | 这个阶段保留大量未选择候选，不能暴露给 executor。 |
| `fetch_upper_rel()` 取得 final rel | 最终 upper rel 是当前 Query level 的 Path 竞争终点。 |
| `create_plan()` 递归转换 Path | 只沿选中的 best_path 生成 Plan，未选 Path 留在 planner memory context 中。 |
| `set_plan_references()` 整理引用 | 把 Var、Param、rtable 和 SubPlan 关系压成 executor 可独立解释的形式。 |
| `PlannedStmt` 填充 | 把 planTree、rtable、permInfos、subplans、rowMarks、relationOids、invalItems 汇总。 |
| 计划交给上层 | Portal、plancache 或 executor 使用 PlannedStmt，不再依赖 root 的生命周期。 |

### 5.1. `standard_planner()` 初始化 `PlannerGlobal`

此时还没有 Plan，只有跨层共享的收集容器和参数输入。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. `subquery_planner()` 创建 `PlannerInfo`

root 指向当前 Query，后续改写和搜索都在 root 上沉淀临时状态。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. preprocessing 改写 Query

子查询 pullup、表达式 canonicalize、outer join reduction 会直接修改 parse tree，但目标是让后续搜索语义更清晰。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. `query_planner()` 生成 `RelOptInfo` 和 `Path`

这个阶段保留大量未选择候选，不能暴露给 executor。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. `fetch_upper_rel()` 取得 final rel

最终 upper rel 是当前 Query level 的 Path 竞争终点。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. `create_plan()` 递归转换 Path

只沿选中的 best_path 生成 Plan，未选 Path 留在 planner memory context 中。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. `set_plan_references()` 整理引用

把 Var、Param、rtable 和 SubPlan 关系压成 executor 可独立解释的形式。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `PlannedStmt` 填充

把 planTree、rtable、permInfos、subplans、rowMarks、relationOids、invalItems 汇总。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.9. 计划交给上层

Portal、plancache 或 executor 使用 PlannedStmt，不再依赖 root 的生命周期。

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
| 1 | `Query` 不是不可变对象；标准 planner 会在当前 memory context 中修改它，因此重复规划前要 copy。 |
| 2 | `Path` 的语义依赖 root、RelOptInfo、EquivalenceClass 和 RestrictInfo，不能直接跨阶段保存给 executor。 |
| 3 | `Plan` 必须经过 setrefs，才能让 executor 按 rangetable index 和 Param id 稳定访问。 |
| 4 | `PlannedStmt` 中的 `relationOids` 和 `invalItems` 是计划缓存正确失效的基础。 |
| 5 | SubPlan 的 Param 类型和 extParam/allParam 关系必须在 finalization 后一致，否则 executor 无法正确传参。 |

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
| 1 | 如果计划依赖 role 或权限相关状态，`dependsOnRole` 会进入 PlannedStmt，plan cache 必须考虑角色变化。 |
| 2 | 如果 rewrite 或 planning 中引用了 relation、operator、function 等对象，依赖会进入 `relationOids` 或 `invalItems`。 |
| 3 | 如果 `set_plan_references()` 之前就把 Plan 当最终契约使用，会遇到未扁平化 rtable 或未整理 Var 的不稳定状态。 |
| 4 | 如果 extension 在 planner hook 中保存 root 指针到查询结束之后，通常会变成悬挂指针。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果计划依赖 role 或权限相关状态，`dependsOnRole` 会进入 PlannedStmt，plan cache 必须考虑角色变化。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果 rewrite 或 planning 中引用了 relation、operator、function 等对象，依赖会进入 `relationOids` 或 `invalItems`。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果 `set_plan_references()` 之前就把 Plan 当最终契约使用，会遇到未扁平化 rtable 或未整理 Var 的不稳定状态。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果 extension 在 planner hook 中保存 root 指针到查询结束之后，通常会变成悬挂指针。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | 保留 Path 搜索状态会消耗 planner memory context，但它只需要活到规划结束。 |
| 2 | PlannedStmt 被缓存或执行，因此必须剥离大部分搜索临时状态，降低保留成本。 |
| 3 | setrefs 不是执行成本，但它决定 executor 后续每次访问字段的稳定性和简洁性。 |
| 4 | 计划缓存的失效粒度越准确，越能避免不必要 replanning，也能避免 stale plan。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | `EXPLAIN (VERBOSE)` 展示 Plan / PlannedStmt 层面的结果，不展示 PlannerInfo 的搜索中间态。 |
| 2 | 断在 `create_plan()` 可以看到 Path 仍然携带 RelOptInfo 上下文；断在 `ExecutorStart()` 时只能看到 PlannedStmt。  |
| 3 | 开启 `debug_print_parse`、`debug_print_rewritten`、`debug_print_plan` 可以比较 Query 与 Plan 的结构差异。 |
| 4 | 查询 `pg_prepared_statements` 可以观察计划缓存存在，但无法直接看到 PlannerInfo。 |

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

### 实验 1：结构打印

对同一条带子查询的 SQL 开启 parse、rewritten、plan 日志，比较 rtable、jointree 和最终 planTree。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：gdb root 生命周期

断在 `subquery_planner()`、`create_plan()`、`standard_planner()` 返回前，观察 root 是否还被最终 PlannedStmt 引用。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：依赖失效

准备语句后修改索引或统计信息，观察重新规划行为和 `invalItems` 的意义。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：SubPlan 参数

用相关子查询触发 PARAM_EXEC，观察 `glob->paramExecTypes` 和最终 PlannedStmt。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 为什么 Query 适合表达 SQL 语义，却不适合承载 Path 搜索状态？

2. 如果 PlannedStmt 保留所有 Path，能带来什么诊断便利，又会制造什么生命周期和内存问题？

3. 计划缓存失效属于 planner、executor 还是 catalog cache 的职责边界？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/include/nodes/parsenodes.h`

`Query`、`RangeTblEntry`、`FromExpr`、`JoinExpr` 等输入语义结构。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/include/nodes/pathnodes.h`

`PlannerGlobal`、`PlannerInfo`、`RelOptInfo`、`Path`、`RestrictInfo` 等 planner-local 状态。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/include/nodes/plannodes.h`

`PlannedStmt`、`Plan` 和具体 Plan node 的执行契约。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/planner.c`

`standard_planner()` 构造 `PlannerGlobal`、调用 `subquery_planner()`、填充 `PlannedStmt`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/plan/createplan.c`

`create_plan()` 和各类 `create_*_plan()` 从 Path 生成 Plan。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/backend/optimizer/plan/setrefs.c`

`set_plan_references()` 让 Plan 脱离 planner-local 指针。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/backend/optimizer/plan/subselect.c`

SubPlan / InitPlan 如何进入 `PlannerGlobal` 和 `PlannedStmt`。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/include/utils/plancache.h`

计划缓存如何保存并失效 PlannedStmt。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. `standard_planner()` 初始化 `PlannerGlobal`

此时还没有 Plan，只有跨层共享的收集容器和参数输入。

进入前先确认 `Query` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerGlobal` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. `subquery_planner()` 创建 `PlannerInfo`

root 指向当前 Query，后续改写和搜索都在 root 上沉淀临时状态。

进入前先确认 `PlannerGlobal` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. preprocessing 改写 Query

子查询 pullup、表达式 canonicalize、outer join reduction 会直接修改 parse tree，但目标是让后续搜索语义更清晰。

进入前先确认 `PlannerInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `RelOptInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. `query_planner()` 生成 `RelOptInfo` 和 `Path`

这个阶段保留大量未选择候选，不能暴露给 executor。

进入前先确认 `RelOptInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Path` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. `fetch_upper_rel()` 取得 final rel

最终 upper rel 是当前 Query level 的 Path 竞争终点。

进入前先确认 `Path` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Plan` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. `create_plan()` 递归转换 Path

只沿选中的 best_path 生成 Plan，未选 Path 留在 planner memory context 中。

进入前先确认 `Plan` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannedStmt` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. `set_plan_references()` 整理引用

把 Var、Param、rtable 和 SubPlan 关系压成 executor 可独立解释的形式。

进入前先确认 `PlannedStmt` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Query` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `PlannedStmt` 填充

把 planTree、rtable、permInfos、subplans、rowMarks、relationOids、invalItems 汇总。

进入前先确认 `Query` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerGlobal` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.9. 计划交给上层

Portal、plancache 或 executor 使用 PlannedStmt，不再依赖 root 的生命周期。

进入前先确认 `PlannerGlobal` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerInfo` 是否被创建、改写、选择、缓存或转交给下游。

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

- 检查点 1：在 `src/include/nodes/parsenodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/include/nodes/pathnodes.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/include/nodes/plannodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/plan/createplan.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/backend/optimizer/plan/setrefs.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/backend/optimizer/plan/subselect.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/include/utils/plancache.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/include/nodes/parsenodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/include/nodes/pathnodes.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/include/nodes/plannodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/plan/createplan.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/backend/optimizer/plan/setrefs.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/backend/optimizer/plan/subselect.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/include/utils/plancache.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/include/nodes/parsenodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/include/nodes/pathnodes.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/include/nodes/plannodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/plan/createplan.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/backend/optimizer/plan/setrefs.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/backend/optimizer/plan/subselect.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/include/utils/plancache.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/include/nodes/parsenodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/include/nodes/pathnodes.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/include/nodes/plannodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/plan/createplan.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/backend/optimizer/plan/setrefs.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/backend/optimizer/plan/subselect.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/include/utils/plancache.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/include/nodes/parsenodes.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/include/nodes/pathnodes.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/include/nodes/plannodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/planner.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/plan/createplan.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/backend/optimizer/plan/setrefs.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/backend/optimizer/plan/subselect.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/include/utils/plancache.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/include/nodes/parsenodes.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/include/nodes/pathnodes.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/include/nodes/plannodes.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/planner.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/plan/createplan.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/backend/optimizer/plan/setrefs.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/backend/optimizer/plan/subselect.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/include/utils/plancache.h` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

`Query` 不是不可变对象；标准 planner 会在当前 memory context 中修改它，因此重复规划前要 copy。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/nodes/parsenodes.h`。

### 18.2. 正确性边界

`Path` 的语义依赖 root、RelOptInfo、EquivalenceClass 和 RestrictInfo，不能直接跨阶段保存给 executor。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.3. 正确性边界

`Plan` 必须经过 setrefs，才能让 executor 按 rangetable index 和 Param id 稳定访问。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/include/nodes/plannodes.h`。

### 18.4. 正确性边界

`PlannedStmt` 中的 `relationOids` 和 `invalItems` 是计划缓存正确失效的基础。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.5. 正确性边界

SubPlan 的 Param 类型和 extParam/allParam 关系必须在 finalization 后一致，否则 executor 无法正确传参。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/createplan.c`。

### 18.6. 异常或 fallback

如果计划依赖 role 或权限相关状态，`dependsOnRole` 会进入 PlannedStmt，plan cache 必须考虑角色变化。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/plan/setrefs.c`。

### 18.7. 异常或 fallback

如果 rewrite 或 planning 中引用了 relation、operator、function 等对象，依赖会进入 `relationOids` 或 `invalItems`。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/plan/subselect.c`。

### 18.8. 异常或 fallback

如果 `set_plan_references()` 之前就把 Plan 当最终契约使用，会遇到未扁平化 rtable 或未整理 Var 的不稳定状态。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/utils/plancache.h`。

### 18.9. 异常或 fallback

如果 extension 在 planner hook 中保存 root 指针到查询结束之后，通常会变成悬挂指针。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/nodes/parsenodes.h`。

### 18.10. 成本传播

保留 Path 搜索状态会消耗 planner memory context，但它只需要活到规划结束。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/include/nodes/pathnodes.h`。

### 18.11. 成本传播

PlannedStmt 被缓存或执行，因此必须剥离大部分搜索临时状态，降低保留成本。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/include/nodes/plannodes.h`。

### 18.12. 成本传播

setrefs 不是执行成本，但它决定 executor 后续每次访问字段的稳定性和简洁性。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.13. 成本传播

计划缓存的失效粒度越准确，越能避免不必要 replanning，也能避免 stale plan。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/createplan.c`。

### 18.14. 观测入口

`EXPLAIN (VERBOSE)` 展示 Plan / PlannedStmt 层面的结果，不展示 PlannerInfo 的搜索中间态。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/plan/setrefs.c`。

### 18.15. 观测入口

断在 `create_plan()` 可以看到 Path 仍然携带 RelOptInfo 上下文；断在 `ExecutorStart()` 时只能看到 PlannedStmt。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/plan/subselect.c`。

### 18.16. 观测入口

开启 `debug_print_parse`、`debug_print_rewritten`、`debug_print_plan` 可以比较 Query 与 Plan 的结构差异。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/utils/plancache.h`。

### 18.17. 观测入口

查询 `pg_prepared_statements` 可以观察计划缓存存在，但无法直接看到 PlannerInfo。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/nodes/parsenodes.h`。

## 19. 本节小结

- `Query -> PlannerInfo -> Path -> Plan -> PlannedStmt` 是状态降维过程。
- planner 阶段需要丰富临时上下文，executor 阶段需要稳定契约，这就是结构转换存在的主要原因。
- 诊断优化器问题时，先定位状态停留在哪一层，再选择日志、gdb 或源码入口。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
