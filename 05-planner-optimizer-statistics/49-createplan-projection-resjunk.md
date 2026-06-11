# PostgreSQL createplan 中 projection、resjunk 与表达式替换边界

## 课程定位

前置知识：已经理解 PostgreSQL parser / analyzer / rewriter 之后的 Query 进入 planner，知道 RelOptInfo、Path、Plan、PlannedStmt 和 executor 的基本边界。

本节唯一主问题：

```text
为什么最终 Plan 的 targetlist 不等于 SQL 输出列列表，resjunk、sortgroupref、PlaceHolderVar 和 projection 节点如何保证 executor 能拿到排序、分组、锁行和 join 所需的中间值？
```

核心矛盾：SQL 结果列应该只暴露用户请求的输出，但 executor 运行计划时还需要排序键、分组键、row identity、PlaceHolderVar、join 输入引用和参数化表达式；如果把这些工作列泄漏到结果，语义错误；如果过早删除，执行器又无法完成节点协议。

学完后应能判断一个 targetlist entry 是用户可见输出、executor 工作列、排序/分组标签，还是等待 setrefs 替换的中间表达式。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一组课程已经从 Path 搜索推进到最优 Path 的选择。本节进入 Path 到 Plan 的最后转换层，关注 targetlist 为什么会在这个阶段变形。

05 目录的主线不是把 optimizer 文件逐个讲完，而是跟踪一个 SQL 如何被逐步压缩成可执行判断：语义树先变成搜索状态，统计和选择率给出行数，成本模型比较 Path，最后 createplan 与 setrefs 把搜索结果交给 executor。

```text
Query -> PlannerInfo -> RelOptInfo / Path / PathTarget -> Plan -> PlannedStmt -> Executor
```

本节只处理这个链条中的一个主问题。相邻模块会被提到，但只在它们解释本节的状态推进、正确性边界或诊断证据时展开。

阅读时把注意力放在时间线上：哪个状态先被写入，哪个函数消费它，哪些信息会进入最终计划，哪些只在 planner memory context 中短暂存在。

如果后面诊断慢 SQL 时发现某个字段“看起来不合理”，也要先问它属于 Query、Path、Plan、PlanState 还是 catalog 统计；不同层的字段不能直接互相替代。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
planner 先在 processed_tlist 和 PathTarget 中保留所有下游需要的表达式；createplan.c 按节点是否能投影决定 targetlist 下推或注入 projection；setrefs.c 最后把 Var、PlaceHolderVar 和 sortgroupref 改写成 executor 能按 slot 读取的引用。
```

这个模型隐含两条边界。第一，planner 的判断必须在执行前完成，不能等 executor 运行一半再重新搜索全部 Path。第二，最终交给 executor 的结构必须足够稳定，不能还依赖 optimizer 内部的临时链表和搜索历史。

| 侧面 | 本节判断 |
| --- | --- |
| 输入事实 | Query、catalog、统计信息、GUC、schema、已有 Path 或最终 Plan 字段。 |
| 局部状态 | 通常挂在 PlannerInfo、RelOptInfo、Path、Plan 或 PlanState 上，生命周期不同。 |
| 正确性边界 | 不能破坏 SQL 语义、outer join 约束、Param 作用域、权限、plan cache invalidation 或 executor contract。 |
| 成本边界 | planner 只能比较估算成本，不能直接预测每次执行的真实毫秒。 |
| 观测结果 | 多数内部候选不可见，只能通过 EXPLAIN、GUC 对照、catalog、日志、断点和源码路径还原。 |

因此，本节的阅读顺序固定为：先看可观测现象，再定位源码入口；先判断状态边界，再讨论修复手段。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/optimizer/prep/preptlist.c` | preprocess_targetlist() 把 SQL targetlist 扩展成 planner 需要的 processed_tlist，包括 DML row identity 和 RETURNING 相关工作列。 |
| 2 | `src/include/nodes/primnodes.h` | TargetEntry 的 expr、resno、resname、ressortgroupref、resjunk 定义了输出列和工作列的共同载体。 |
| 3 | `src/include/nodes/pathnodes.h` | PathTarget、PlannerInfo.processed_tlist、RowIdentityVarInfo 和 PlaceHolderVar 描述 planner 阶段的投影需求。 |
| 4 | `src/backend/optimizer/plan/createplan.c` | build_path_tlist()、create_projection_plan()、inject_projection_plan()、prepare_sort_from_pathkeys() 把 PathTarget 转为 Plan targetlist。 |
| 5 | `src/backend/optimizer/plan/setrefs.c` | set_plan_references()、fix_scan_expr()、fix_join_expr()、fix_upper_expr()、search_indexed_tlist_for_sortgroupref() 完成最终引用替换。 |
| 6 | `src/include/nodes/plannodes.h` | Plan、Sort、PlanRowMark、ModifyTable 等结构说明哪些工作列会继续影响 executor contract。 |

阅读顺序按 mental model 排列，不按目录名排序。先看入口和状态结构，再看引用替换、fallback、成本传播和观测输出。

源码核对命令：

```bash
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

本节只引用当前本地源码中真实存在的路径和函数。未来版本如果移动实现位置，应优先保持这里的系统边界，再更新具体入口。

## 4. 从可观测现象进入源码

本节从 `projection / resjunk 边界` 的可观测现象进入源码。先确认现象属于 Query、Path、Plan、PlannedStmt、PlanState 还是 catalog 统计，再决定要读哪个文件。

```text
现象
  -> 阶段边界
     -> 状态写入点
        -> 状态消费点
           -> 单变量复测
```

### 现象 1：ORDER BY 表达式不出现在结果列

排序键可以作为 `resjunk TargetEntry` 留在子计划 targetlist 中，用户协议层仍只返回 SELECT 列。

`prepare_sort_from_pathkeys()` 会检查输入 tlist，缺少排序表达式时追加工作列。

记录时把 SELECT 输出列、EXPLAIN VERBOSE 的 Sort Key / Output、子计划 targetlist 和 `prepare_sort_from_pathkeys()` 的追加列对应起来，判断它是排序工作列还是用户输出。

### 现象 2：EXPLAIN VERBOSE 的 Output 与 SELECT 不同

Output 展示当前 Plan 节点的输出协议，不是客户端最终列描述。

`build_path_tlist()` 和 `set_plan_references()` 之间能看到 PathTarget 到 Plan targetlist 的转换。

记录时把 EXPLAIN VERBOSE 的每层 Output 与 `build_path_tlist()` 生成的 Plan targetlist 对齐，避免把中间节点输出误读成客户端列描述。

### 现象 3：UPDATE / DELETE 出现 row identity

DML 需要定位被修改的行，定位字段不是用户可见输出。

`preprocess_targetlist()` 与 `RowIdentityVarInfo` 解释这些列为何进入 processed_tlist。

记录时把 UPDATE / DELETE 的 target relation、rowmark、ctid / whole-row identity 和 `preprocess_targetlist()` 的工作列放在同一张表里，确认它们只服务定位而不暴露给用户。

### 现象 4：GROUP BY / DISTINCT 依赖 ressortgroupref

排序分组语义需要稳定编号，不能靠列名或表达式文本临时匹配。

`PathTarget.sortgrouprefs` 被复制到 TargetEntry，setrefs 后仍要保持对应。

记录时把 SortGroupClause 编号、TargetEntry.ressortgroupref、GROUP / DISTINCT key 和 setrefs 后的 tlist 一起核对，避免把编号稳定性误判成列顺序问题。

### 现象 5：PlaceHolderVar 在最终计划中变形

PHV 可能被替换成子计划输出 Var，也可能在合适层级展开。

`fix_join_expr()`、`fix_upper_expr()` 与 indexed_tlist 决定替换边界。

记录时把 outer join 层级、PlaceHolderVar 的 eval_at relids、EXPLAIN Output 和 `fix_join_expr()` / `fix_upper_expr()` 断点对应起来，确认替换发生在哪个 plan node。

### 现象 6：额外 Result projection 节点

有些节点不能直接输出上层需要的 targetlist，需要插入投影层。

`inject_projection_plan()` 说明这是执行协议调整，不一定是性能问题。

记录时比较子计划 Output、注入的 Result targetlist 和上层需求；若只有 projection 变化而 rows / cost 不变，应先按 executor 输出协议解释。

## 5. 关键数据结构与状态边界

本节只解释会影响诊断的字段组合。字段本身不是语义；字段加上阶段、owner、生命周期和下游消费者，才构成可用判断。

| 状态 | 一句话语义 |
| --- | --- |
| `TargetEntry.expr` | 待计算或传递的表达式树，可能是用户列、排序键、row identity 或 PHV 展开结果。 |
| `TargetEntry.resjunk` | 工作列标志；列存在于执行协议中，但不属于最终用户输出。 |
| `TargetEntry.ressortgroupref` | 把 TargetEntry 与 SortGroupClause / GroupClause 连接起来的编号。 |
| `PathTarget.exprs` | Path 阶段的投影需求，服务宽度估算、投影成本和上层表达式依赖。 |
| `PlannerInfo.processed_tlist` | 预处理后的全局目标列表，可能已经包含 DML 和 rowmark 工作列。 |
| `PlaceHolderVar` | outer join 与表达式延迟求值之间的语义占位。 |
| `indexed_tlist` | setrefs 临时建立的子计划输出索引，用来快速匹配 Var、PHV 和 sortgroupref。 |
| `Plan.targetlist` | executor 节点之间的输出协议，不等于 SQL SELECT 列清单。 |

### `TargetEntry.expr`

待计算或传递的表达式树，可能是用户列、排序键、row identity 或 PHV 展开结果。

它服务于 `projection / resjunk 边界` 的第 1 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `TargetEntry.resjunk`

工作列标志；列存在于执行协议中，但不属于最终用户输出。

它服务于 `projection / resjunk 边界` 的第 2 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `TargetEntry.ressortgroupref`

把 TargetEntry 与 SortGroupClause / GroupClause 连接起来的编号。

它服务于 `projection / resjunk 边界` 的第 3 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `PathTarget.exprs`

Path 阶段的投影需求，服务宽度估算、投影成本和上层表达式依赖。

它服务于 `projection / resjunk 边界` 的第 4 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `PlannerInfo.processed_tlist`

预处理后的全局目标列表，可能已经包含 DML 和 rowmark 工作列。

它服务于 `projection / resjunk 边界` 的第 5 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `PlaceHolderVar`

outer join 与表达式延迟求值之间的语义占位。

它服务于 `projection / resjunk 边界` 的第 6 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `indexed_tlist`

setrefs 临时建立的子计划输出索引，用来快速匹配 Var、PHV 和 sortgroupref。

它服务于 `projection / resjunk 边界` 的第 7 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `Plan.targetlist`

executor 节点之间的输出协议，不等于 SQL SELECT 列清单。

它服务于 `projection / resjunk 边界` 的第 8 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

## 6. 主流程源码 walkthrough

下面按时间线阅读主流程。每一步都要问：输入状态是什么，输出状态是什么，哪些信息继续进入下一阶段。

```text
预处理目标列
  -> PathTarget 表示投影需求
  -> best path 进入 create_plan
  -> 构造 Plan targetlist
  -> 补齐排序键
  -> 注入 projection
  -> setrefs 替换引用
  -> executor 消费契约
```

### 步骤 1：预处理目标列

`preprocess_targetlist()` 把 DML row identity、RETURNING、rowmark 相关工作列加入 planner 输入。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 2：PathTarget 表示投影需求

RelOptInfo 和 upperrel 用 PathTarget 表达当前阶段需要输出的表达式集合。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 3：best path 进入 create_plan

被剪掉的投影候选不会进入最终 Plan，EXPLAIN 默认也看不到。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 4：构造 Plan targetlist

`build_path_tlist()` 把 PathTarget 变成 TargetEntry，并保留 sortgroupref。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 5：补齐排序键

`prepare_sort_from_pathkeys()` 需要时追加 resjunk 排序列。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 6：注入 projection

`create_projection_plan()` 或 `inject_projection_plan()` 调整节点输出形状。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 7：setrefs 替换引用

`set_plan_references()` 把 Var / PHV 改成 executor 可按 slot 读取的引用。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 8：executor 消费契约

Sort、Agg、ModifyTable、LockRows 和 Result 按 targetlist 与列号运行。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

## 7. 生命周期 / ownership / cleanup

planner 诊断里常见错误，是把一个阶段的对象当成另一个阶段仍然有效。先理清生命周期，后面的字段解释才有落点。

| 问题 | 边界 |
| --- | --- |
| 创建 | 相关状态在 planner、catalog、executor 或诊断报告中形成；必须区分运行时对象和外部证据。 |
| 持有 | planner-local 对象只在一次规划中有效；长期报告应持有 SQL、EXPLAIN、catalog、GUC 和实验记录。 |
| 释放 | 临时 GUC、实验索引、事务内 DML 和测试数据应在实验后撤回。 |
| ERROR | 普通 palloc 对象依赖 MemoryContext cleanup；外部资源依赖 ResourceOwner 或上层调用路径。 |
| 失效 | 统计、schema、权限、GUC、plan cache 和版本变化都会让同一 SQL 的判断不同。 |
| 交接 | 另一个工程师应能按记录复跑并得到同类 planner 判断。 |

### 创建

相关状态在 planner、catalog、executor 或诊断报告中形成；必须区分运行时对象和外部证据。

长期报告保存外部事实；gdb 中的 backend-local 指针只适合辅助一次调试，不适合作为可复现证据。

### 持有

planner-local 对象只在一次规划中有效；长期报告应持有 SQL、EXPLAIN、catalog、GUC 和实验记录。

长期报告保存外部事实；gdb 中的 backend-local 指针只适合辅助一次调试，不适合作为可复现证据。

### 释放

临时 GUC、实验索引、事务内 DML 和测试数据应在实验后撤回。

长期报告保存外部事实；gdb 中的 backend-local 指针只适合辅助一次调试，不适合作为可复现证据。

### ERROR

普通 palloc 对象依赖 MemoryContext cleanup；外部资源依赖 ResourceOwner 或上层调用路径。

长期报告保存外部事实；gdb 中的 backend-local 指针只适合辅助一次调试，不适合作为可复现证据。

### 失效

统计、schema、权限、GUC、plan cache 和版本变化都会让同一 SQL 的判断不同。

长期报告保存外部事实；gdb 中的 backend-local 指针只适合辅助一次调试，不适合作为可复现证据。

### 交接

另一个工程师应能按记录复跑并得到同类 planner 判断。

长期报告保存外部事实；gdb 中的 backend-local 指针只适合辅助一次调试，不适合作为可复现证据。

## 8. 正确性机制层次

优化器可以选择不同执行方式，但不能选择语义不同的执行方式。本节的 correctness 多数来自阶段边界、合法性检查和 executor contract。

| 层次 | 不变量 |
| --- | --- |
| 1 | 一个阶段的字段不能直接解释另一个阶段的语义。 |
| 2 | 优化器可以换路径，不能换 SQL 结果。 |
| 3 | 每次实验只改变一个 planner 可见输入。 |
| 4 | 现象、状态、源码入口和复测必须能互相支撑。 |
| 5 | EXPLAIN ANALYZE、DML、CREATE INDEX 和全局 GUC 都可能影响生产。 |
| 6 | 任何修复都要写清影响哪些相邻查询和写入路径。 |

### 不变量 1：阶段边界

一个阶段的字段不能直接解释另一个阶段的语义。

围绕 `projection / resjunk 边界` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 2：语义优先

优化器可以换路径，不能换 SQL 结果。

围绕 `projection / resjunk 边界` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 3：单变量验证

每次实验只改变一个 planner 可见输入。

围绕 `projection / resjunk 边界` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 4：证据闭环

现象、状态、源码入口和复测必须能互相支撑。

围绕 `projection / resjunk 边界` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 5：副作用可控

EXPLAIN ANALYZE、DML、CREATE INDEX 和全局 GUC 都可能影响生产。

围绕 `projection / resjunk 边界` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 6：范围说明

任何修复都要写清影响哪些相邻查询和写入路径。

围绕 `projection / resjunk 边界` 做优化时，任何建议都必须先说明没有越过这个边界。

## 9. 错误路径 / 异常路径 / fallback

很多分支不是错误，而是在信息不足或语义受限时选择保守路径。诊断时要区分“缺少事实”和“事实被错误解释”。

### fallback 1：把 SELECT 列表当 Plan targetlist

在 `projection / resjunk 边界` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 2：看到 resjunk 就认为可以删除

在 `projection / resjunk 边界` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 3：只看顶层 Output

在 `projection / resjunk 边界` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 4：把 PHV 当普通表达式

在 `projection / resjunk 边界` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 5：用列名解释 sortgroupref

在 `projection / resjunk 边界` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

## 10. 成本、资源与跨模块传播

一个 planner 判断会穿过多个模块：统计影响 rows，rows 影响 cost，cost 影响 path 保留，path 形态又决定 executor 的内存、I/O、并行和观测结果。

| 传播点 | 影响 |
| --- | --- |
| 规划时间 | 复杂 SQL、join search、partition pruning、统计读取都会增加 Planning Time。 |
| 执行资源 | BUFFERS、temp I/O、JIT、worker 输出解释实际耗时。 |
| 统计维护 | ANALYZE、statistics target、extended stats 有采样和存储成本。 |
| 索引维护 | 索引提高部分读取，也增加写入、vacuum、存储和规划成本。 |
| 全局参数 | cost、work_mem、parallel、JIT 调整会影响整个 workload。 |
| 诊断成本 | 单变量复测较慢，但能避免把偶然计划变化写成根因。 |

### 规划时间

复杂 SQL、join search、partition pruning、统计读取都会增加 Planning Time。

诊断报告要标出它属于 planner 阶段估算、executor 阶段实际资源，还是 EXPLAIN 阶段观测。

如果修复会改变全局行为，需要用代表性 workload 证明收益覆盖副作用。

### 执行资源

BUFFERS、temp I/O、JIT、worker 输出解释实际耗时。

诊断报告要标出它属于 planner 阶段估算、executor 阶段实际资源，还是 EXPLAIN 阶段观测。

如果修复会改变全局行为，需要用代表性 workload 证明收益覆盖副作用。

### 统计维护

ANALYZE、statistics target、extended stats 有采样和存储成本。

诊断报告要标出它属于 planner 阶段估算、executor 阶段实际资源，还是 EXPLAIN 阶段观测。

如果修复会改变全局行为，需要用代表性 workload 证明收益覆盖副作用。

### 索引维护

索引提高部分读取，也增加写入、vacuum、存储和规划成本。

诊断报告要标出它属于 planner 阶段估算、executor 阶段实际资源，还是 EXPLAIN 阶段观测。

如果修复会改变全局行为，需要用代表性 workload 证明收益覆盖副作用。

### 全局参数

cost、work_mem、parallel、JIT 调整会影响整个 workload。

诊断报告要标出它属于 planner 阶段估算、executor 阶段实际资源，还是 EXPLAIN 阶段观测。

如果修复会改变全局行为，需要用代表性 workload 证明收益覆盖副作用。

### 诊断成本

单变量复测较慢，但能避免把偶然计划变化写成根因。

诊断报告要标出它属于 planner 阶段估算、executor 阶段实际资源，还是 EXPLAIN 阶段观测。

如果修复会改变全局行为，需要用代表性 workload 证明收益覆盖副作用。

## 11. 观测与诊断入口

观测目标不是看到所有内部状态，而是收集足够证据还原 planner 做过的关键判断。

| 入口 | 用途 |
| --- | --- |
| EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) | 保存估算、实际、资源、输出和 GUC。 |
| EXPLAIN (FORMAT JSON) | 便于脚本 diff 和计算 rows ratio。 |
| pg_class / pg_stats / pg_statistic_ext | 确认 planner 能看到的数据分布。 |
| SHOW / pg_settings | 保存 cost、enable、parallel、JIT、work_mem 和 plan cache 设置。 |
| gdb / lldb 断点 | 观察状态写入和消费，不把一次指针值当长期事实。 |

```text
SQL text and bind parameter shape
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
schema, index, constraint and partition definitions
pg_class / pg_stats / pg_statistic_ext summary
planner-related GUCs
PostgreSQL version and source commit
one-variable comparison experiments
```

`EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)` 的价值在于：保存估算、实际、资源、输出和 GUC。 收集后要回到 `projection / resjunk 边界` 的主问题，而不是把指标堆成附件。

`EXPLAIN (FORMAT JSON)` 的价值在于：便于脚本 diff 和计算 rows ratio。 收集后要回到 `projection / resjunk 边界` 的主问题，而不是把指标堆成附件。

`pg_class / pg_stats / pg_statistic_ext` 的价值在于：确认 planner 能看到的数据分布。 收集后要回到 `projection / resjunk 边界` 的主问题，而不是把指标堆成附件。

`SHOW / pg_settings` 的价值在于：保存 cost、enable、parallel、JIT、work_mem 和 plan cache 设置。 收集后要回到 `projection / resjunk 边界` 的主问题，而不是把指标堆成附件。

`gdb / lldb 断点` 的价值在于：观察状态写入和消费，不把一次指针值当长期事实。 收集后要回到 `projection / resjunk 边界` 的主问题，而不是把指标堆成附件。

## 12. 常见误区

### 误区 1：把 SELECT 列表当 Plan targetlist

这个判断在 `projection / resjunk 边界` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 2：看到 resjunk 就认为可以删除

这个判断在 `projection / resjunk 边界` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 3：只看顶层 Output

这个判断在 `projection / resjunk 边界` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 4：把 PHV 当普通表达式

这个判断在 `projection / resjunk 边界` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 5：用列名解释 sortgroupref

这个判断在 `projection / resjunk 边界` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 6：把 Result projection 都当性能问题

这个判断在 `projection / resjunk 边界` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

## 13. 课堂实验

实验不追求覆盖所有计划形态，只要求把本节主问题放进可复测闭环。建议在独立测试库执行，并记录每次 GUC、统计和数据规模。

### 实验 1：ORDER BY 工作列

比较 Output 与 Sort Key，定位 `prepare_sort_from_pathkeys()`。

```sql
CREATE TABLE t_proj(id int, k int, v text);
INSERT INTO t_proj SELECT g, g % 10, md5(g::text) FROM generate_series(1, 1000) g;
ANALYZE t_proj;
EXPLAIN (VERBOSE) SELECT id FROM t_proj ORDER BY k + 1 LIMIT 5;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 2：GROUP BY 标签

观察表达式、Group Key 和 sortgroupref 的对应。

```sql
EXPLAIN (VERBOSE) SELECT k, count(*) FROM t_proj GROUP BY k ORDER BY k;
EXPLAIN (VERBOSE) SELECT k + 1 AS g, count(*) FROM t_proj GROUP BY k + 1 ORDER BY g;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 3：DML row identity

区分 ModifyTable 定位输入与 RETURNING 输出。

```sql
BEGIN;
EXPLAIN (VERBOSE) DELETE FROM t_proj WHERE k = 3 RETURNING id;
ROLLBACK;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 4：projection 位置

比较表达式在 scan 层还是 Result 层出现。

```sql
EXPLAIN (VERBOSE) SELECT length(v) FROM t_proj WHERE k = 1;
EXPLAIN (VERBOSE) SELECT length(v) FROM (SELECT v FROM t_proj WHERE k = 1) s;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 5：断点

打印 tlist 长度、resjunk、ressortgroupref 和节点类型。

```gdb
break build_path_tlist
break prepare_sort_from_pathkeys
break inject_projection_plan
break set_plan_references
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

## 14. 源码练习

### 练习 1

围绕 `TargetEntry.expr` 设计一个断点或日志输出，说明它在 `projection / resjunk 边界` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 2

围绕 `TargetEntry.resjunk` 设计一个断点或日志输出，说明它在 `projection / resjunk 边界` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 3

围绕 `TargetEntry.ressortgroupref` 设计一个断点或日志输出，说明它在 `projection / resjunk 边界` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 4

围绕 `PathTarget.exprs` 设计一个断点或日志输出，说明它在 `projection / resjunk 边界` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 5

围绕 `PlannerInfo.processed_tlist` 设计一个断点或日志输出，说明它在 `projection / resjunk 边界` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

## 15. 讨论题

1. 如果只能保存一条证据，哪条最能回答：为什么最终 Plan 的 targetlist 不等于 SQL 输出列列表？

2. `projection / resjunk 边界` 中哪些状态是 planner-local，哪些会进入 PlannedStmt 或 PlanState？

3. 遇到计划不符合直觉时，如何区分 SQL 语义限制、统计不足、成本参数偏差和搜索空间剪枝？

4. 如果修复需要全局 GUC 或新增索引，它会影响哪些其它查询？

5. 这个机制体现的可迁移规律是提前规范化、保守 fallback、显式 contract，还是候选剪枝？

讨论要落到可观测现象、源码入口和单变量复测。只给经验描述，不能支撑内核级诊断结论。

## 16. 诊断记录模板

这一节的诊断记录围绕 `projection / resjunk 边界` 展开。模板不是替代分析，而是保证每次判断都能回到事实、状态和源码。

### 必须保存的事实

- 完整 SQL、参数形态、版本、schema、索引、统计和 planner GUC。
- 基线 EXPLAIN 与所有单变量对照 EXPLAIN。

### 判断问题

- 当前证据是否直接回答 `为什么最终 Plan 的 targetlist 不等于 SQL 输出列列表？`
- 结论对应的是 rows、cost、path generation、setrefs、executor resource，还是 plan cache 行为？

### 可复测动作

- 保存基线计划，不改变任何 GUC。
- 只改变一个输入事实，例如统计、索引、成本参数、SQL 形态或 enable_* 开关。
- 保存变化后的 EXPLAIN，并比较 rows、cost、path shape 和资源指标。
- 把变化映射到第 3 节列出的源码入口。
- 撤回实验修改，再验证基线是否恢复。

### 修复风险

- 说明修复对相邻查询、写入路径、规划时间和执行资源的影响。
- 如果只是短期止血，要写清长期方案和回滚条件。

最终报告不要只写“计划变好了”。更有价值的结论是：哪个 planner 输入改变了，哪个源码阶段消费了它，哪些相邻查询可能被同一修改影响。

### 案例复盘写法

下面这段不是额外模板，而是把 `projection / resjunk 边界` 落到一份诊断报告时应有的叙述粒度。

#### 1. 现象边界

先写清楚现象：ORDER BY 表达式没有出现在 SELECT 输出中。

这句话要避免混入修复建议。现象段只描述可观察事实，例如 EXPLAIN 字段、SQL 形态、参数分布、统计快照或资源指标。

如果现象来自线上慢 SQL，还要写明采集窗口、版本、数据库、schema、参数值来源和是否使用 prepared statement。

#### 2. 第一处判断

然后指出第一处能支撑结论的 planner 判断，证据集中在：Output、Sort Key、resjunk、ressortgroupref。

这里不要跳到最终根节点。根节点往往已经混合了多层传播，真正可修复的位置通常更靠近 scan、join 输入、path 生成或 finalization 边界。

如果只能说明“计划看起来不合理”，还没有找到第一处判断；继续回到叶子节点、catalog 统计或 path 生成断点。

#### 3. 源码落点

源码落点写成一个短链路，例如：`prepare_sort_from_pathkeys()` / `set_plan_references()`。

链路里每个函数只承担一个角色：有的写状态，有的消费状态，有的做合法性检查，有的只是把结果打印出来。

报告中应说明你引用的是哪个角色，而不是把函数名当成结论。

#### 4. 单变量对照

对照实验只改变一个输入。

可以改变统计、索引、SQL 形态、session 级 GUC 或参数值，但一次只改一个。

每个对照都记录三件事：改动命令、预期变化、实际计划差异。

如果计划没有按预期变化，不要继续叠加第二个改动；先解释第一个假设为什么没有成立。

#### 5. 修复边界

修复风险要显式写出：删除或下推工作列可能破坏 Sort、Agg、ModifyTable 或 LockRows。

短期止血可以是 session 级开关、临时索引或 SQL 改写；长期方案必须说明为什么它让 planner 看到更准确的事实，或者为什么它缩小了错误搜索空间。

如果方案影响全局参数、共享统计或写入路径，要补一组代表性查询回归，而不是只给一条 SQL 的 before/after。

#### 6. 报告结论

结论段建议压缩成四句话。

第一句：本案属于 `projection / resjunk 边界`，不是泛泛的“优化器选错”。

第二句：首个错误判断点在哪里，证据是什么。

第三句：源码入口是 `prepare_sort_from_pathkeys()` / `set_plan_references()`，它怎样消费这些证据。

第四句：最小修复是什么，回归风险是什么。

这样写的好处是，另一个工程师不需要相信你的经验；他可以按同一组 SQL、catalog、GUC 和源码入口复测。

#### 7. 复查口径

复查时先检查证据是否仍然成立，再检查修复是否仍然最小。

统计刷新、版本升级、索引变化、参数分布变化和 workload 变化，都可能让旧结论失效。

因此报告要保存采集时间和源码基线；否则后续复盘只能看到结论，看不到结论成立的条件。

## 17. 本节小结

本节唯一主问题是：为什么最终 Plan 的 targetlist 不等于 SQL 输出列列表？

- `projection / resjunk 边界` 必须按阶段解释，不能只看最终计划形状。
- 可靠诊断要把现象、状态、源码入口和单变量复测连成闭环。
- EXPLAIN 展示的是胜出计划和执行观测，不展示完整搜索历史。
- 修复建议要说明影响范围和回归风险。

当后续遇到新的 planner 问题时，先定位阶段，再判断边界，最后选择最小修复。
遇到 targetlist 相关问题时，优先复查 resjunk、sortgroupref、PHV 和 setrefs 后 Var 形态是否仍然指向同一个执行协议。
