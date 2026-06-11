# PostgreSQL Planner 入口与优化器阶段边界

## 课程定位

前置知识：已经理解 parse、analyze、rewrite 会产出语义完整的 `Query`，也知道 executor 最终只接受 `PlannedStmt` 中的 `Plan` tree。

本节唯一主问题：

```text
一棵已经 parse / analyze / rewrite 完成的 `Query` 进入 `planner()` 后，为什么还要经过 `standard_planner()`、`subquery_planner()`、`grouping_planner()` 和 `query_planner()` 多层分工，而不是由一个大函数直接生成执行计划？
```

核心矛盾：SQL 语义必须在改写和搜索之间保持稳定，但优化器又必须不断重写表达式、拉平子查询、构造等价类、生成 Path 并选择最低代价；如果阶段边界混在一起，语义保护、搜索空间和最终执行契约都会互相污染。

学完后应能判断一段 planner 逻辑属于语义预处理、scan/join 搜索、upper planning、Path 到 Plan 转换，还是扩展 hook / GUC 只影响搜索空间的边界。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

本节是 05 目录的入口，回答优化器为什么按阶段推进。后续课程会把这些阶段中的关键状态逐一拆开：`Query` 到 `PlannedStmt` 的结构边界、`PlannerInfo` 的全局上下文、hook / GUC 的可插拔边界，以及 preprocessing 如何把表达式和 join tree 改写成后续估算可以消费的形态。

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
planner() 只负责选择 hook 或标准入口；standard_planner() 创建跨子查询共享的 PlannerGlobal；subquery_planner() 为每个 Query level 建立 PlannerInfo 并做一次性语义预处理；grouping_planner() 在 scan/join 结果之上构造 upper rel；query_planner() 生成 base/join Path 搜索空间，最终再由 create_plan() 和 set_plan_references() 固化成 PlannedStmt。
```

这个模型里有三个稳定判断。

- 语义状态和搜索状态必须分开。
- planner-local 状态可以很丰富，但不能泄漏给 executor。
- 每一次改写都要能回到后续可验证的 Path、Plan 或 EXPLAIN 现象。

如果读源码时只记函数名，很容易把当前阶段的临时状态误认为最终语义。正确的读法是沿着一个状态对象追踪：它在哪里创建，谁写入关键字段，什么时候被下游消费，什么时候被抛弃。

## 3. 核心文件分工与阅读顺序

| 项 | 说明 |
| --- | --- |
| 1 | `src/backend/optimizer/plan/planner.c`：`planner()`、`standard_planner()`、`subquery_planner()`、`grouping_planner()`，主流程入口和阶段切换。 |
| 2 | `src/backend/optimizer/plan/planmain.c`：`query_planner()`，scan/join 搜索的核心入口。 |
| 3 | `src/backend/optimizer/path/allpaths.c`：`make_one_rel()`、`set_base_rel_pathlists()`，把 joinlist 转成 RelOptInfo / Path 搜索空间。 |
| 4 | `src/backend/optimizer/plan/createplan.c`：`create_plan()`，把选中的 Path 递归转成 executor 可运行的 Plan。 |
| 5 | `src/backend/optimizer/plan/setrefs.c`：`set_plan_references()`，整理 rtable、Var、Param 和 plan node metadata。 |
| 6 | `src/include/optimizer/planner.h`：planner hook 和 planner 入口声明。 |
| 7 | `src/include/optimizer/planmain.h`：query_planner、create_plan、set_plan_references 等阶段接口声明。 |
| 8 | `src/backend/optimizer/README`：优化器目录分工、Path / RelOptInfo / join search 的设计说明。 |

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
| Query | 输入语义树，已经包含 parse/analyze/rewrite 后的 range table、jointree、targetList、qual 和上层 SQL 属性。 |
| PlannerGlobal | 一次 planner invocation 共享状态，贯穿多个 Query level，保存 subplans、finalrtable、rowmarks、paramExecTypes、parallelModeNeeded 等全局结果。 |
| PlannerInfo | 单个 Query level 的优化上下文，约定名是 root；后续绝大部分 planner 函数都围绕它读写状态。 |
| RelOptInfo | 搜索空间里的 relation-like 节点，表示 base rel、joinrel 或 upper rel，并挂载候选 Path。 |
| Path | 候选实现方式，保存 rows、cost、pathkeys、parameterization、parallel 属性等可比较信息。 |
| Plan | 被选中 Path 的执行器形态，不再保留所有候选，只保留 executor 需要的节点树。 |
| PlannedStmt | planner 输出契约，包含 planTree、rtable、subplans、rowMarks、invalItems、paramExecTypes 和并行/JIT 等执行边界。 |

判断这些状态时有一个通用规则：

```text
raw field 不是语义；
field + 生命周期 + 阶段边界 + 访问者，才是 planner 中可诊断的语义。
```

planner 的很多对象都在普通 backend-local memory context 中分配。它们不是 shared memory，不跨 backend 可见，也不应该被保存到执行期长期结构里。

### 状态切片 1：Query

输入语义树，已经包含 parse/analyze/rewrite 后的 range table、jointree、targetList、qual 和上层 SQL 属性。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 2：PlannerGlobal

一次 planner invocation 共享状态，贯穿多个 Query level，保存 subplans、finalrtable、rowmarks、paramExecTypes、parallelModeNeeded 等全局结果。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 3：PlannerInfo

单个 Query level 的优化上下文，约定名是 root；后续绝大部分 planner 函数都围绕它读写状态。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 4：RelOptInfo

搜索空间里的 relation-like 节点，表示 base rel、joinrel 或 upper rel，并挂载候选 Path。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 5：Path

候选实现方式，保存 rows、cost、pathkeys、parameterization、parallel 属性等可比较信息。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 6：Plan

被选中 Path 的执行器形态，不再保留所有候选，只保留 executor 需要的节点树。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

### 状态切片 7：PlannedStmt

planner 输出契约，包含 planTree、rtable、subplans、rowMarks、invalItems、paramExecTypes 和并行/JIT 等执行边界。

读到这里要问三个问题：它是输入语义、搜索状态还是输出契约；它是否会被后续阶段继续改写；它的生命周期是否短于最终 `PlannedStmt`。

## 5. 主流程源码 walkthrough

下面按时间顺序跟主链路。每个小段都只回答一个问题：当前状态发生了什么变化。

| 项 | 说明 |
| --- | --- |
| `planner()` | 检查 `planner_hook`。扩展可以包裹整个 planner，但通常仍要调用 `standard_planner()` 才能得到完整语义。 |
| `standard_planner()` | 创建 `PlannerGlobal`，初始化 subplan、finalrtable、parallel hazard、path generation mask 和 tuple_fraction。 |
| `subquery_planner()` | 为当前 `Query` 创建 `PlannerInfo`，处理 CTE、RTE、SubLink、函数 RTE、子查询 pullup、表达式 preprocessing、outer join reduction。 |
| `grouping_planner()` | 处理 set operation、aggregation、window、distinct、sort、limit、modify table 等 upper planning 阶段。 |
| `query_planner()` | 初始化 join_rel、pathkeys、placeholder、join_info；建立 base RelOptInfo，分发 RestrictInfo，生成 joinlist。 |
| `make_one_rel()` | 基于 joinlist 生成 base path 和 join path，完成 scan/join 搜索。 |
| `fetch_upper_rel(UPPERREL_FINAL)` | 取得最终 upper rel，里面保存能实现完整查询结果的 Path 候选。 |
| `get_cheapest_fractional_path()` | 根据 tuple_fraction 选择满足 fast-start 或全量读取目标的最优 Path。 |
| `create_plan()` | 把选中的 Path tree 递归转换为 Plan tree。 |
| `set_plan_references()` | 把 planner 阶段的引用整理成 executor 能独立使用的下标、Param 和 flat rtable。 |
| `PlannedStmt` 构造 | 把计划树、依赖、权限、row mark、subplan、JIT 和并行模式信息集中到输出契约中。 |

### 5.1. `planner()`

检查 `planner_hook`。扩展可以包裹整个 planner，但通常仍要调用 `standard_planner()` 才能得到完整语义。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.2. `standard_planner()`

创建 `PlannerGlobal`，初始化 subplan、finalrtable、parallel hazard、path generation mask 和 tuple_fraction。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.3. `subquery_planner()`

为当前 `Query` 创建 `PlannerInfo`，处理 CTE、RTE、SubLink、函数 RTE、子查询 pullup、表达式 preprocessing、outer join reduction。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.4. `grouping_planner()`

处理 set operation、aggregation、window、distinct、sort、limit、modify table 等 upper planning 阶段。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.5. `query_planner()`

初始化 join_rel、pathkeys、placeholder、join_info；建立 base RelOptInfo，分发 RestrictInfo，生成 joinlist。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.6. `make_one_rel()`

基于 joinlist 生成 base path 和 join path，完成 scan/join 搜索。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.7. `fetch_upper_rel(UPPERREL_FINAL)`

取得最终 upper rel，里面保存能实现完整查询结果的 Path 候选。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.8. `get_cheapest_fractional_path()`

根据 tuple_fraction 选择满足 fast-start 或全量读取目标的最优 Path。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.9. `create_plan()`

把选中的 Path tree 递归转换为 Plan tree。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

这一段常见的误读是把“现在能看到的字段”当成“最终执行器会看到的字段”。多数情况下，当前字段只是下一阶段搜索或整理的材料。

### 5.10. `set_plan_references()`

把 planner 阶段的引用整理成 executor 能独立使用的下标、Param 和 flat rtable。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

### 5.11. `PlannedStmt` 构造

把计划树、依赖、权限、row mark、subplan、JIT 和并行模式信息集中到输出契约中。

这里不要停在函数名上。断点停住以后，先打印当前输入对象，再打印本阶段刚写入的 planner 状态，最后确认下游会从哪里消费它。

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
| 1 | preprocessing 阶段可以改写 Query，但必须保持 SQL 语义。 |
| 2 | query_planner 阶段可以搜索多个 Path，但不能把未选择的候选泄漏给 executor。 |
| 3 | create_plan 阶段只能固化已选择 Path，不能重新扩大搜索空间。 |
| 4 | setrefs 阶段完成 planner 到 executor 的引用整理，使执行器不需要理解 RelOptInfo / Path 搜索结构。 |
| 5 | hook 可以观察或扩展阶段，但不能绕过最终 PlannedStmt 的语义契约。 |

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
| 1 | 如果 parse tree 含 parallel-unsafe 函数，parallelModeOK 会在早期被关闭，后续不应再生成需要 parallel mode 的路径。 |
| 2 | 如果 cursor 要求 backward scan，而 top_plan 不支持反向扫描，标准 planner 会在顶层补 Material。 |
| 3 | 如果 SubPlan 产生 PARAM_EXEC，`SS_finalize_plan()` 必须先处理 subplan，再处理 main plan 的 extParam / allParam。 |
| 4 | 如果 hook 复制或重复规划 Query，必须注意 `standard_planner()` 会修改输入 Query。 |

遇到 fallback 时要先问它保护的是语义、生命周期、资源成本还是扩展契约。不同原因对应的修复方向完全不同。

### 异常观察 1

如果 parse tree 含 parallel-unsafe 函数，parallelModeOK 会在早期被关闭，后续不应再生成需要 parallel mode 的路径。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 2

如果 cursor 要求 backward scan，而 top_plan 不支持反向扫描，标准 planner 会在顶层补 Material。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 3

如果 SubPlan 产生 PARAM_EXEC，`SS_finalize_plan()` 必须先处理 subplan，再处理 main plan 的 extParam / allParam。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

### 异常观察 4

如果 hook 复制或重复规划 Query，必须注意 `standard_planner()` 会修改输入 Query。

验证时不要只看最终节点名，还要回到触发保守路径的源码条件。

## 9. 成本、资源与跨模块传播

planner 成本不只来自最终计划，也来自规划期搜索空间和状态传播。

| 项 | 说明 |
| --- | --- |
| 1 | tuple_fraction 会让 fast-start path 和 total-cost path 的排序不同。 |
| 2 | enable_* GUC 在 `default_pgs_mask` 中变成候选生成策略，影响搜索空间，不直接改变 SQL 语义。 |
| 3 | preprocessing 的开销多数只付一次，但会影响后续每个 Path 的数量和质量。 |
| 4 | join search 的成本随 base rel 数量指数增长，阶段边界让 GEQO、join_collapse_limit 等退化策略有明确插入点。 |

这些成本最后会在三个地方表现出来：planner CPU 时间、规划期间内存占用、以及最终 executor 计划的 runtime 成本。诊断时要把三者分开。

## 10. 观测与诊断入口

优化器内部状态多数不可直接从 SQL 看到，但可以通过最终 Plan、日志和断点间接定位。

| 项 | 说明 |
| --- | --- |
| 1 | 用 `EXPLAIN (VERBOSE)` 看 final Plan，只能看到阶段输出，看不到被剪枝的 Path。 |
| 2 | 用 `debug_print_plan` 能看到 PlannedStmt 输出，但仍不是 Path 搜索过程的完整记录。 |
| 3 | gdb 断在 `standard_planner`、`subquery_planner`、`query_planner`、`create_plan`，可以观察同一条 SQL 的状态从 Query 到 Path 再到 Plan 的变化。 |
| 4 | 比较 `SET enable_hashjoin = off` 前后的 EXPLAIN，可以验证 GUC 影响候选路径而不是改变 join 语义。 |

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

### 实验 1：阶段断点

对 `standard_planner`、`subquery_planner`、`query_planner`、`create_plan` 设断点，执行三表 join，逐层打印 `root->query_level`、`root->all_baserels`、`final_rel->pathlist` 长度。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 2：fast-start

使用 cursor 或 LIMIT 查询，比较 `get_cheapest_fractional_path()` 选择的 path 是否偏向 startup cost。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 3：禁用 join 方法

分别关闭 hashjoin、mergejoin、nestloop，观察 `default_pgs_mask` 和最终计划变化。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

### 实验 4：反向游标

用 scroll cursor 触发顶层 Material，确认补节点发生在 `create_plan()` 之后。

实验记录至少包含三项：SQL、EXPLAIN 输出中的关键差异、对应源码断点或函数入口。

实验不追求覆盖所有路径，只追求把一个可见现象解释回源码中的状态变化。

## 13. 讨论题

1. 如果一个优化规则既会改写 Query 又会影响 Path 数量，应该放在 preprocessing 还是 path generation？判断依据是什么？

2. 为什么 executor 不应该直接消费 Path tree？

3. debug_print_plan 为什么无法解释所有 planner 选择？还缺少哪些阶段状态？

回答这些问题时，需要明确引用源码阶段和状态对象，避免只用抽象概念解释抽象概念。

## 14. 源码分段阅读

这一节把第 3 节的源码顺序再拆成可执行的阅读任务。每个任务都要能回到本节唯一主问题，而不是停留在“看过这个文件”。

### 14.1. `src/backend/optimizer/plan/planner.c`

`planner()`、`standard_planner()`、`subquery_planner()`、`grouping_planner()`，主流程入口和阶段切换。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.2. `src/backend/optimizer/plan/planmain.c`

`query_planner()`，scan/join 搜索的核心入口。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.3. `src/backend/optimizer/path/allpaths.c`

`make_one_rel()`、`set_base_rel_pathlists()`，把 joinlist 转成 RelOptInfo / Path 搜索空间。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.4. `src/backend/optimizer/plan/createplan.c`

`create_plan()`，把选中的 Path 递归转成 executor 可运行的 Plan。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.5. `src/backend/optimizer/plan/setrefs.c`

`set_plan_references()`，整理 rtable、Var、Param 和 plan node metadata。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.6. `src/include/optimizer/planner.h`

planner hook 和 planner 入口声明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.7. `src/include/optimizer/planmain.h`

query_planner、create_plan、set_plan_references 等阶段接口声明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

### 14.8. `src/backend/optimizer/README`

优化器目录分工、Path / RelOptInfo / join search 的设计说明。

进入这个文件之前，先写下当前阶段的输入对象和预期输出对象。

阅读时优先找函数入口、状态写入点、错误返回点和下游消费点。

读完后用一个断点或一条 EXPLAIN 现象验证：这个文件确实参与了本节讨论的状态变化。

如果这个文件只提供结构定义，就把字段按生命周期分组，而不是逐字段背诵。

## 15. 状态迁移检查

下面把主流程再拆成状态迁移。它们不是新的知识点，而是防止把函数调用链读成 API 清单。

### 15.1. `planner()`

检查 `planner_hook`。扩展可以包裹整个 planner，但通常仍要调用 `standard_planner()` 才能得到完整语义。

进入前先确认 `Query` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerGlobal` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.2. `standard_planner()`

创建 `PlannerGlobal`，初始化 subplan、finalrtable、parallel hazard、path generation mask 和 tuple_fraction。

进入前先确认 `PlannerGlobal` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.3. `subquery_planner()`

为当前 `Query` 创建 `PlannerInfo`，处理 CTE、RTE、SubLink、函数 RTE、子查询 pullup、表达式 preprocessing、outer join reduction。

进入前先确认 `PlannerInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `RelOptInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.4. `grouping_planner()`

处理 set operation、aggregation、window、distinct、sort、limit、modify table 等 upper planning 阶段。

进入前先确认 `RelOptInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Path` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.5. `query_planner()`

初始化 join_rel、pathkeys、placeholder、join_info；建立 base RelOptInfo，分发 RestrictInfo，生成 joinlist。

进入前先确认 `Path` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Plan` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.6. `make_one_rel()`

基于 joinlist 生成 base path 和 join path，完成 scan/join 搜索。

进入前先确认 `Plan` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannedStmt` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.7. `fetch_upper_rel(UPPERREL_FINAL)`

取得最终 upper rel，里面保存能实现完整查询结果的 Path 候选。

进入前先确认 `PlannedStmt` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Query` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.8. `get_cheapest_fractional_path()`

根据 tuple_fraction 选择满足 fast-start 或全量读取目标的最优 Path。

进入前先确认 `Query` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerGlobal` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.9. `create_plan()`

把选中的 Path tree 递归转换为 Plan tree。

进入前先确认 `PlannerGlobal` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `PlannerInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.10. `set_plan_references()`

把 planner 阶段的引用整理成 executor 能独立使用的下标、Param 和 flat rtable。

进入前先确认 `PlannerInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `RelOptInfo` 是否被创建、改写、选择、缓存或转交给下游。

如果这一阶段只是筛掉候选，不要把被筛掉的路径解释成执行器仍会看到的状态。

如果这一阶段会改写 Query 或 root 字段，下一次断点要放在真正消费这些字段的函数，而不是继续看本函数尾部。

### 15.11. `PlannedStmt` 构造

把计划树、依赖、权限、row mark、subplan、JIT 和并行模式信息集中到输出契约中。

进入前先确认 `RelOptInfo` 是否已经存在，以及它来自输入语义、planner-local 状态还是上一阶段输出。

离开时再确认 `Path` 是否被创建、改写、选择、缓存或转交给下游。

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

- 检查点 1：在 `src/backend/optimizer/plan/planner.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 2：在 `src/backend/optimizer/plan/planmain.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 3：在 `src/backend/optimizer/path/allpaths.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 4：在 `src/backend/optimizer/plan/createplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 5：在 `src/backend/optimizer/plan/setrefs.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 6：在 `src/include/optimizer/planner.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 7：在 `src/include/optimizer/planmain.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 8：在 `src/backend/optimizer/README` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 9：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 10：在 `src/backend/optimizer/plan/planmain.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 11：在 `src/backend/optimizer/path/allpaths.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 12：在 `src/backend/optimizer/plan/createplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 13：在 `src/backend/optimizer/plan/setrefs.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 14：在 `src/include/optimizer/planner.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 15：在 `src/include/optimizer/planmain.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 16：在 `src/backend/optimizer/README` 中验证：字段是否会越过 planner 生命周期。
- 检查点 17：在 `src/backend/optimizer/plan/planner.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 18：在 `src/backend/optimizer/plan/planmain.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 19：在 `src/backend/optimizer/path/allpaths.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 20：在 `src/backend/optimizer/plan/createplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 21：在 `src/backend/optimizer/plan/setrefs.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 22：在 `src/include/optimizer/planner.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 23：在 `src/include/optimizer/planmain.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 24：在 `src/backend/optimizer/README` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 25：在 `src/backend/optimizer/plan/planner.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 26：在 `src/backend/optimizer/plan/planmain.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 27：在 `src/backend/optimizer/path/allpaths.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 28：在 `src/backend/optimizer/plan/createplan.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 29：在 `src/backend/optimizer/plan/setrefs.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 30：在 `src/include/optimizer/planner.h` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 31：在 `src/include/optimizer/planmain.h` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 32：在 `src/backend/optimizer/README` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 33：在 `src/backend/optimizer/plan/planner.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 34：在 `src/backend/optimizer/plan/planmain.c` 中验证：字段是否会越过 planner 生命周期。
- 检查点 35：在 `src/backend/optimizer/path/allpaths.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 36：在 `src/backend/optimizer/plan/createplan.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 37：在 `src/backend/optimizer/plan/setrefs.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 38：在 `src/include/optimizer/planner.h` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 39：在 `src/include/optimizer/planmain.h` 中验证：字段是否被后续阶段继续改写。
- 检查点 40：在 `src/backend/optimizer/README` 中验证：字段是否会越过 planner 生命周期。
- 检查点 41：在 `src/backend/optimizer/plan/planner.c` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 42：在 `src/backend/optimizer/plan/planmain.c` 中验证：最终计划是否能从源码中的同一条状态线解释。
- 检查点 43：在 `src/backend/optimizer/path/allpaths.c` 中验证：入口函数是否只负责分发，还是也创建长期状态。
- 检查点 44：在 `src/backend/optimizer/plan/createplan.c` 中验证：当前对象是否属于 Query、PlannerInfo、RelOptInfo、Path、Plan 或 PlannedStmt。
- 检查点 45：在 `src/backend/optimizer/plan/setrefs.c` 中验证：字段是否被后续阶段继续改写。
- 检查点 46：在 `src/include/optimizer/planner.h` 中验证：字段是否会越过 planner 生命周期。
- 检查点 47：在 `src/include/optimizer/planmain.h` 中验证：候选路径数量是否受 GUC、统计信息或语义约束影响。
- 检查点 48：在 `src/backend/optimizer/README` 中验证：最终计划是否能从源码中的同一条状态线解释。

## 18. 误判与校正

下面这些条目用于把课堂讨论落到排查动作上。每一条都从一个容易误判的现象开始，再给出回到源码的校正路径。

### 18.1. 正确性边界

preprocessing 阶段可以改写 Query，但必须保持 SQL 语义。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.2. 正确性边界

query_planner 阶段可以搜索多个 Path，但不能把未选择的候选泄漏给 executor。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/planmain.c`。

### 18.3. 正确性边界

create_plan 阶段只能固化已选择 Path，不能重新扩大搜索空间。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/path/allpaths.c`。

### 18.4. 正确性边界

setrefs 阶段完成 planner 到 executor 的引用整理，使执行器不需要理解 RelOptInfo / Path 搜索结构。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/createplan.c`。

### 18.5. 正确性边界

hook 可以观察或扩展阶段，但不能绕过最终 PlannedStmt 的语义契约。

先确认语义是否保持，再确认候选路径是否只是搜索空间变化。

源码回看点：`src/backend/optimizer/plan/setrefs.c`。

### 18.6. 异常或 fallback

如果 parse tree 含 parallel-unsafe 函数，parallelModeOK 会在早期被关闭，后续不应再生成需要 parallel mode 的路径。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/optimizer/planner.h`。

### 18.7. 异常或 fallback

如果 cursor 要求 backward scan，而 top_plan 不支持反向扫描，标准 planner 会在顶层补 Material。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/include/optimizer/planmain.h`。

### 18.8. 异常或 fallback

如果 SubPlan 产生 PARAM_EXEC，`SS_finalize_plan()` 必须先处理 subplan，再处理 main plan 的 extParam / allParam。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/README`。

### 18.9. 异常或 fallback

如果 hook 复制或重复规划 Query，必须注意 `standard_planner()` 会修改输入 Query。

先找到保守分支的源码条件，再判断它保护的是语义、资源还是生命周期。

源码回看点：`src/backend/optimizer/plan/planner.c`。

### 18.10. 成本传播

tuple_fraction 会让 fast-start path 和 total-cost path 的排序不同。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/planmain.c`。

### 18.11. 成本传播

enable_* GUC 在 `default_pgs_mask` 中变成候选生成策略，影响搜索空间，不直接改变 SQL 语义。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/path/allpaths.c`。

### 18.12. 成本传播

preprocessing 的开销多数只付一次，但会影响后续每个 Path 的数量和质量。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/createplan.c`。

### 18.13. 成本传播

join search 的成本随 base rel 数量指数增长，阶段边界让 GEQO、join_collapse_limit 等退化策略有明确插入点。

同时记录 planner CPU、候选数量和 executor runtime，避免把三种成本混在一起。

源码回看点：`src/backend/optimizer/plan/setrefs.c`。

### 18.14. 观测入口

用 `EXPLAIN (VERBOSE)` 看 final Plan，只能看到阶段输出，看不到被剪枝的 Path。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/optimizer/planner.h`。

### 18.15. 观测入口

用 `debug_print_plan` 能看到 PlannedStmt 输出，但仍不是 Path 搜索过程的完整记录。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/include/optimizer/planmain.h`。

### 18.16. 观测入口

gdb 断在 `standard_planner`、`subquery_planner`、`query_planner`、`create_plan`，可以观察同一条 SQL 的状态从 Query 到 Path 再到 Plan 的变化。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/README`。

### 18.17. 观测入口

比较 `SET enable_hashjoin = off` 前后的 EXPLAIN，可以验证 GUC 影响候选路径而不是改变 join 语义。

用 EXPLAIN 或 gdb 只确认一个假设，确认后再扩大到完整 SQL。

源码回看点：`src/backend/optimizer/plan/planner.c`。

## 19. 本节小结

- planner 的多层入口不是代码风格问题，而是把语义改写、搜索空间、upper planning 和执行契约分开。
- 标准主线从 `Query` 进入，以 `PlannedStmt` 输出；中间的 `PlannerInfo`、`RelOptInfo` 和 `Path` 都是 planner-local 状态。
- 诊断慢 SQL 时，先判断问题发生在语义预处理、行数估算、Path 生成、Path 剪枝还是 Plan 固化，而不是直接从最终 Plan 倒推所有原因。

把本节内容压缩成一个可迁移规律：

```text
复杂优化器的关键不是把所有规则塞进一个入口，
而是把语义改写、候选搜索、成本比较和执行契约放在不同生命周期里。
```
