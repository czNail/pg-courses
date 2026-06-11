# PostgreSQL 行数估错定位：统计信息还是谓词模型

## 课程定位

前置知识：已经理解 PostgreSQL parser / analyzer / rewriter 之后的 Query 进入 planner，知道 RelOptInfo、Path、Plan、PlannedStmt 和 executor 的基本边界。

本节唯一主问题：

```text
慢 SQL 中 rows 估错时，如何沿 plan tree 找第一个失真节点，并判断问题来自 stale stats、低统计目标、列相关性、表达式统计缺失还是 selectivity fallback？
```

核心矛盾：planner 必须在执行前用有限样本、catalog 统计和谓词模型估计未来行数；真实数据分布、相关性、表达式、参数值和过期统计却可能让这些假设偏离。诊断既要承认估算是近似，又要找到可修复的第一处错误。

学完后应能把 rows 偏差拆成 base relation、restriction clause、join clause、group distinct 和 upper rel 几类入口，并为每类选择合适的验证方法。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节把 EXPLAIN 字段拆成 planner 判断和 executor 观测。本节聚焦最常见根因：rows 估算从哪里开始失真。

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
从 EXPLAIN 中找最早 rows ratio 异常的节点；如果是 scan，检查 reltuples、pg_statistic、选择率函数和 extended stats；如果是 join，检查 join selectivity、ndistinct、MCV、outer join 语义和前置节点 rows；如果是 aggregate 或 distinct，检查 group 数估算。
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
| 1 | `src/backend/optimizer/path/costsize.c` | set_baserel_size_estimates()、set_joinrel_size_estimates()、calc_joinrel_size_estimate()、clamp_row_est() 传播行数。 |
| 2 | `src/backend/optimizer/path/clausesel.c` | clauselist_selectivity() 和 clauselist_selectivity_ext() 组合 restriction / join clause 选择率。 |
| 3 | `src/backend/optimizer/path/allpaths.c` | set_rel_size()、set_plain_rel_size() 决定 base relation rows 何时写入。 |
| 4 | `src/backend/optimizer/util/plancat.c` | restriction_selectivity()、join_selectivity() 通过操作符统计函数进入 selfuncs。 |
| 5 | `src/backend/utils/adt/selfuncs.c` | eqsel、scalarltsel、scalarineqsel 等函数使用 MCV、histogram、nullfrac、ndistinct。 |
| 6 | `src/backend/statistics/extended_stats.c` | statext_clauselist_selectivity() 在多列统计可用时修正独立性假设。 |
| 7 | `src/backend/commands/analyze.c` | do_analyze_rel()、acquire_sample_rows() 说明 pg_statistic 来自采样。 |
| 8 | `src/include/catalog/pg_statistic.h` | pg_statistic 的 nullfrac、width、distinct、kind/operator/value slots 是列统计存储边界。 |

阅读顺序按 mental model 排列，不按目录名排序。先看入口和状态结构，再看引用替换、fallback、成本传播和观测输出。

源码核对命令：

```bash
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

本节只引用当前本地源码中真实存在的路径和函数。未来版本如果移动实现位置，应优先保持这里的系统边界，再更新具体入口。

## 4. 从可观测现象进入源码

本节从 `rows 估错定位` 的可观测现象进入源码。先确认现象属于 Query、Path、Plan、PlannedStmt、PlanState 还是 catalog 统计，再决定要读哪个文件。

```text
现象
  -> 阶段边界
     -> 状态写入点
        -> 状态消费点
           -> 单变量复测
```

### 现象 1：Seq Scan rows 偏差明显

先判断 reltuples 是否过期，再看单列 MCV、histogram、null_frac 和 n_distinct。

入口是 `set_baserel_size_estimates()`。

记录时把 pg_class.reltuples、ANALYZE 时间、pg_stats 的 null_frac / n_distinct / MCV / histogram 和该 scan 的 Plan / Actual Rows 放在同一处，先判断统计输入是否过期。

### 现象 2：Index Scan 使用索引但 rows 仍错

索引可用性和选择率准确性是两件事。

`cost_index()` 会消费 indexSelectivity 和 baserel->tuples。

记录时区分 Index Cond 的选择率、Filter 的残余选择率、indexSelectivity 与 heap rows；索引被使用不代表行数模型已经正确。

### 现象 3：两个条件单独准，组合后低估

列相关性被独立性相乘放大，extended statistics 可能修正。

看 `clauselist_selectivity_ext()` 是否调用 `statext_clauselist_selectivity()`。

记录时分别保存单条件 EXPLAIN、组合条件 EXPLAIN、extended stats 定义和 dependencies / MCV 命中情况，判断是否是独立性相乘造成低估。

### 现象 4：表达式谓词估算很粗

函数表达式、类型转换或不可统计表达式可能走默认选择率。

考虑表达式统计、生成列、函数索引或 SQL 改写。

记录时把原始谓词、表达式索引或生成列、函数 volatility、统计对象和 fallback 选择率放在一起，避免把表达式缺统计误判成索引问题。

### 现象 5：join 输入准但 join rows 错

问题可能在 join selectivity、ndistinct、MCV 或 join 语义。

入口是 `set_joinrel_size_estimates()` 和 `calc_joinrel_size_estimate()`。

记录时先证明两个输入 rel 的 rows 已经接近，再比较 join clause、ndistinct、MCV 和 join type 下界；否则 join 节点只是继承前置错误。

### 现象 6：ANALYZE 后计划恢复

根因大概率在统计输入，但仍要区分过期统计和模型不足。

保存 ANALYZE 前后的 pg_class、pg_stats 和 EXPLAIN 差异。

记录时保存 ANALYZE 前后的 reltuples、统计 slots、EXPLAIN JSON 和计划形态差异；若只在采样后短暂恢复，要继续看数据倾斜和统计目标。

## 5. 关键数据结构与状态边界

本节只解释会影响诊断的字段组合。字段本身不是语义；字段加上阶段、owner、生命周期和下游消费者，才构成可用判断。

| 状态 | 一句话语义 |
| --- | --- |
| `pg_class.reltuples` | 表级 tuple 数估计，是 base rows 的起点。 |
| `pg_statistic.stanullfrac` | 列中 NULL 比例。 |
| `pg_statistic.stadistinct` | distinct 数估算，负值表示相对行数比例。 |
| `MCV list` | 常见值和频率列表。 |
| `histogram bounds` | 范围谓词的分布边界。 |
| `StatisticExtInfo` | extended statistics 在 planner 中的载体。 |
| `RelOptInfo.rows` | 当前 relation 或 joinrel 的估算输出行数。 |
| `clamp_row_est()` | 把行数压到有效范围。 |

### `pg_class.reltuples`

表级 tuple 数估计，是 base rows 的起点。

它服务于 `rows 估错定位` 的第 1 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `pg_statistic.stanullfrac`

列中 NULL 比例。

它服务于 `rows 估错定位` 的第 2 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `pg_statistic.stadistinct`

distinct 数估算，负值表示相对行数比例。

它服务于 `rows 估错定位` 的第 3 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `MCV list`

常见值和频率列表。

它服务于 `rows 估错定位` 的第 4 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `histogram bounds`

范围谓词的分布边界。

它服务于 `rows 估错定位` 的第 5 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `StatisticExtInfo`

extended statistics 在 planner 中的载体。

它服务于 `rows 估错定位` 的第 6 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `RelOptInfo.rows`

当前 relation 或 joinrel 的估算输出行数。

它服务于 `rows 估错定位` 的第 7 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `clamp_row_est()`

把行数压到有效范围。

它服务于 `rows 估错定位` 的第 8 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

## 6. 主流程源码 walkthrough

下面按时间线阅读主流程。每一步都要问：输入状态是什么，输出状态是什么，哪些信息继续进入下一阶段。

```text
找首个偏差节点
  -> 区分 scan 与 join
  -> 读取 catalog 事实
  -> 定位选择率函数
  -> 检查组合假设
  -> 验证统计修复
  -> 确认传播链
  -> 记录剩余误差
```

### 步骤 1：找首个偏差节点

从叶子向根比较 estimated rows 与 actual rows。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 2：区分 scan 与 join

scan 偏差查 base stats；join 偏差先检查输入 rows。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 3：读取 catalog 事实

保存 pg_class、pg_stats、pg_statistic_ext 和统计目标。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 4：定位选择率函数

按操作符进入 restriction_selectivity、join_selectivity、eqsel 或 scalarineqsel。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 5：检查组合假设

多个 AND 条件默认近似独立，extended stats 只在匹配到列组时介入。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 6：验证统计修复

ANALYZE、提高 statistics target 或 CREATE STATISTICS 后复测同一 SQL。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 7：确认传播链

首个节点修复后上层 join/agg/sort 自动改善，根因更可信。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 8：记录剩余误差

统计刷新后仍错，说明模型可能无法表达参数敏感性或业务分布。

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

围绕 `rows 估错定位` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 2：语义优先

优化器可以换路径，不能换 SQL 结果。

围绕 `rows 估错定位` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 3：单变量验证

每次实验只改变一个 planner 可见输入。

围绕 `rows 估错定位` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 4：证据闭环

现象、状态、源码入口和复测必须能互相支撑。

围绕 `rows 估错定位` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 5：副作用可控

EXPLAIN ANALYZE、DML、CREATE INDEX 和全局 GUC 都可能影响生产。

围绕 `rows 估错定位` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 6：范围说明

任何修复都要写清影响哪些相邻查询和写入路径。

围绕 `rows 估错定位` 做优化时，任何建议都必须先说明没有越过这个边界。

## 9. 错误路径 / 异常路径 / fallback

很多分支不是错误，而是在信息不足或语义受限时选择保守路径。诊断时要区分“缺少事实”和“事实被错误解释”。

### fallback 1：根节点 rows 错就是根因

在 `rows 估错定位` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 2：ANALYZE 后变好就结束

在 `rows 估错定位` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 3：所有组合条件都靠 extended stats 修

在 `rows 估错定位` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 4：索引没用就是 rows 错

在 `rows 估错定位` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 5：提高全局统计目标无副作用

在 `rows 估错定位` 中，这类情况通常说明当前证据还不能支撑最终修复。

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

`EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)` 的价值在于：保存估算、实际、资源、输出和 GUC。 收集后要回到 `rows 估错定位` 的主问题，而不是把指标堆成附件。

`EXPLAIN (FORMAT JSON)` 的价值在于：便于脚本 diff 和计算 rows ratio。 收集后要回到 `rows 估错定位` 的主问题，而不是把指标堆成附件。

`pg_class / pg_stats / pg_statistic_ext` 的价值在于：确认 planner 能看到的数据分布。 收集后要回到 `rows 估错定位` 的主问题，而不是把指标堆成附件。

`SHOW / pg_settings` 的价值在于：保存 cost、enable、parallel、JIT、work_mem 和 plan cache 设置。 收集后要回到 `rows 估错定位` 的主问题，而不是把指标堆成附件。

`gdb / lldb 断点` 的价值在于：观察状态写入和消费，不把一次指针值当长期事实。 收集后要回到 `rows 估错定位` 的主问题，而不是把指标堆成附件。

## 12. 常见误区

### 误区 1：根节点 rows 错就是根因

这个判断在 `rows 估错定位` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 2：ANALYZE 后变好就结束

这个判断在 `rows 估错定位` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 3：所有组合条件都靠 extended stats 修

这个判断在 `rows 估错定位` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 4：索引没用就是 rows 错

这个判断在 `rows 估错定位` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 5：提高全局统计目标无副作用

这个判断在 `rows 估错定位` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 6：把业务知识当 planner 已知事实

这个判断在 `rows 估错定位` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

## 13. 课堂实验

实验不追求覆盖所有计划形态，只要求把本节主问题放进可复测闭环。建议在独立测试库执行，并记录每次 GUC、统计和数据规模。

### 实验 1：列相关性低估

观察组合谓词是否被独立性假设低估。

```sql
CREATE TABLE t_corr(a int, b int);
INSERT INTO t_corr SELECT g % 100, g % 100 FROM generate_series(1, 100000) g;
ANALYZE t_corr;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_corr WHERE a = 7 AND b = 7;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 2：dependencies 统计

比较 rows 变化。

```sql
CREATE STATISTICS st_corr_dep (dependencies) ON a, b FROM t_corr;
ANALYZE t_corr;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_corr WHERE a = 7 AND b = 7;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 3：范围谓词

结合 histogram_bounds 解释估算。

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_corr WHERE a BETWEEN 10 AND 20;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 4：表达式统计缺失

判断是否需要表达式统计或生成列。

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_corr WHERE (a + b) = 14;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 5：断点

记录 clause 和 selectivity 返回值。

```gdb
break clauselist_selectivity_ext
break restriction_selectivity
break eqsel
break scalarineqsel
break statext_clauselist_selectivity
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

## 14. 源码练习

### 练习 1

围绕 `pg_class.reltuples` 设计一个断点或日志输出，说明它在 `rows 估错定位` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 2

围绕 `pg_statistic.stanullfrac` 设计一个断点或日志输出，说明它在 `rows 估错定位` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 3

围绕 `pg_statistic.stadistinct` 设计一个断点或日志输出，说明它在 `rows 估错定位` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 4

围绕 `MCV list` 设计一个断点或日志输出，说明它在 `rows 估错定位` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 5

围绕 `histogram bounds` 设计一个断点或日志输出，说明它在 `rows 估错定位` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

## 15. 讨论题

1. 如果只能保存一条证据，哪条最能回答：rows 估错来自统计信息还是谓词模型？

2. `rows 估错定位` 中哪些状态是 planner-local，哪些会进入 PlannedStmt 或 PlanState？

3. 遇到计划不符合直觉时，如何区分 SQL 语义限制、统计不足、成本参数偏差和搜索空间剪枝？

4. 如果修复需要全局 GUC 或新增索引，它会影响哪些其它查询？

5. 这个机制体现的可迁移规律是提前规范化、保守 fallback、显式 contract，还是候选剪枝？

讨论要落到可观测现象、源码入口和单变量复测。只给经验描述，不能支撑内核级诊断结论。

## 16. 诊断记录模板

这一节的诊断记录围绕 `rows 估错定位` 展开。模板不是替代分析，而是保证每次判断都能回到事实、状态和源码。

### 必须保存的事实

- 完整 SQL、参数形态、版本、schema、索引、统计和 planner GUC。
- 基线 EXPLAIN 与所有单变量对照 EXPLAIN。

### 判断问题

- 当前证据是否直接回答 `rows 估错来自统计信息还是谓词模型？`
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

下面这段不是额外模板，而是把 `rows 估错定位` 落到一份诊断报告时应有的叙述粒度。

#### 1. 现象边界

先写清楚现象：两个谓词单独估算合理，组合后显著低估。

这句话要避免混入修复建议。现象段只描述可观察事实，例如 EXPLAIN 字段、SQL 形态、参数分布、统计快照或资源指标。

如果现象来自线上慢 SQL，还要写明采集窗口、版本、数据库、schema、参数值来源和是否使用 prepared statement。

#### 2. 第一处判断

然后指出第一处能支撑结论的 planner 判断，证据集中在：pg_stats、extended stats、estimated rows、actual rows。

这里不要跳到最终根节点。根节点往往已经混合了多层传播，真正可修复的位置通常更靠近 scan、join 输入、path 生成或 finalization 边界。

如果只能说明“计划看起来不合理”，还没有找到第一处判断；继续回到叶子节点、catalog 统计或 path 生成断点。

#### 3. 源码落点

源码落点写成一个短链路，例如：`clauselist_selectivity_ext()` / `statext_clauselist_selectivity()`。

链路里每个函数只承担一个角色：有的写状态，有的消费状态，有的做合法性检查，有的只是把结果打印出来。

报告中应说明你引用的是哪个角色，而不是把函数名当成结论。

#### 4. 单变量对照

对照实验只改变一个输入。

可以改变统计、索引、SQL 形态、session 级 GUC 或参数值，但一次只改一个。

每个对照都记录三件事：改动命令、预期变化、实际计划差异。

如果计划没有按预期变化，不要继续叠加第二个改动；先解释第一个假设为什么没有成立。

#### 5. 修复边界

修复风险要显式写出：盲目建索引会掩盖统计输入或选择率模型问题。

短期止血可以是 session 级开关、临时索引或 SQL 改写；长期方案必须说明为什么它让 planner 看到更准确的事实，或者为什么它缩小了错误搜索空间。

如果方案影响全局参数、共享统计或写入路径，要补一组代表性查询回归，而不是只给一条 SQL 的 before/after。

#### 6. 报告结论

结论段建议压缩成四句话。

第一句：本案属于 `rows 估错定位`，不是泛泛的“优化器选错”。

第二句：首个错误判断点在哪里，证据是什么。

第三句：源码入口是 `clauselist_selectivity_ext()` / `statext_clauselist_selectivity()`，它怎样消费这些证据。

第四句：最小修复是什么，回归风险是什么。

这样写的好处是，另一个工程师不需要相信你的经验；他可以按同一组 SQL、catalog、GUC 和源码入口复测。

#### 7. 复查口径

复查时先检查证据是否仍然成立，再检查修复是否仍然最小。

统计刷新、版本升级、索引变化、参数分布变化和 workload 变化，都可能让旧结论失效。

因此报告要保存采集时间和源码基线；否则后续复盘只能看到结论，看不到结论成立的条件。

## 17. 本节小结

本节唯一主问题是：rows 估错来自统计信息还是谓词模型？

- `rows 估错定位` 必须按阶段解释，不能只看最终计划形状。
- 可靠诊断要把现象、状态、源码入口和单变量复测连成闭环。
- EXPLAIN 展示的是胜出计划和执行观测，不展示完整搜索历史。
- 修复建议要说明影响范围和回归风险。

当后续遇到新的 planner 问题时，先定位阶段，再判断边界，最后选择最小修复。
