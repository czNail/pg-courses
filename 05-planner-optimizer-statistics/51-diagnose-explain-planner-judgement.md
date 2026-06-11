# PostgreSQL 从 EXPLAIN 中读出 planner 的判断

## 课程定位

前置知识：已经理解 PostgreSQL parser / analyzer / rewriter 之后的 Query 进入 planner，知道 RelOptInfo、Path、Plan、PlannedStmt 和 executor 的基本边界。

本节唯一主问题：

```text
如何从 EXPLAIN (ANALYZE, BUFFERS) 的 estimated rows、actual rows、loops、startup / total cost、width 和 chosen node 看出 planner 当时相信了什么？
```

核心矛盾：EXPLAIN 只展示胜出的 Plan tree，不展示被淘汰的 Path、统计取值和每次 cost 比较；诊断者既不能把计划当成黑盒，也不能凭经验直接改参数，必须把可见字段还原为 planner 的阶段性判断。

学完后应能把一个执行计划拆成 rows 判断、cost 判断、节点选择、循环次数和实际观测五类证据，并知道每类证据应回到哪个源码入口。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前两节讲 Path 如何变成 executor contract。本节转向慢 SQL 诊断：先学会从 EXPLAIN 反推出 planner 当时相信了什么。

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
EXPLAIN 的估算字段来自 Plan 节点中由 Path 复制而来的 startup_cost、total_cost、plan_rows、plan_width；ANALYZE 字段来自 executor instrumentation；诊断要先比较同一节点的估算与实际，再沿 plan tree 找到最早失真的判断点。
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
| 1 | `src/backend/commands/explain.c` | ExplainOnePlan()、ExplainPrintPlan()、ExplainNode() 负责把 Plan 和 PlanState 输出为 EXPLAIN。 |
| 2 | `src/backend/executor/instrument.c` | InstrStartNode()、InstrStopNode() 和 BufferUsageAccumDiff() 生成 ANALYZE、BUFFERS、WAL 等实际指标。 |
| 3 | `src/include/executor/instrument.h` | NodeInstrumentation、BufferUsage、WalUsage 定义实际执行指标边界。 |
| 4 | `src/include/nodes/plannodes.h` | Plan 节点保存 startup_cost、total_cost、plan_rows、plan_width。 |
| 5 | `src/backend/optimizer/path/costsize.c` | cost_*() 和 rows estimate 入口决定估算值如何写进 Path。 |
| 6 | `src/backend/optimizer/util/pathnode.c` | add_path()、set_cheapest() 解释为什么 EXPLAIN 只展示胜出的候选。 |
| 7 | `src/backend/optimizer/plan/createplan.c` | copy_generic_path_info() 把 Path 的估算值复制到 Plan。 |

阅读顺序按 mental model 排列，不按目录名排序。先看入口和状态结构，再看引用替换、fallback、成本传播和观测输出。

源码核对命令：

```bash
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

本节只引用当前本地源码中真实存在的路径和函数。未来版本如果移动实现位置，应优先保持这里的系统边界，再更新具体入口。

## 4. 从可观测现象进入源码

本节从 `EXPLAIN 诊断 planner 判断` 的可观测现象进入源码。先确认现象属于 Query、Path、Plan、PlannedStmt、PlanState 还是 catalog 统计，再决定要读哪个文件。

```text
现象
  -> 阶段边界
     -> 状态写入点
        -> 状态消费点
           -> 单变量复测
```

### 现象 1：estimated rows 小但 actual rows 大

planner 已在该节点或子节点低估基数，下游 join 形态可能只是连锁结果。

先找最早 rows ratio 偏离节点，再回 costsize、clausesel 或统计。

记录时保存同一节点的 Plan Rows、Actual Rows、Actual Loops 和最早偏离节点路径，再回到 `copy_generic_path_info()` 之前的 rows 写入入口。

### 现象 2：rows 基本准但节点仍慢

问题可能是 I/O、work_mem、JIT、CPU 表达式、锁等待或缓存现实。

把 BUFFERS、I/O timing、Sort Method、Hash Batches、JIT section 分开记录。

记录时把 rows ratio 与 BUFFERS、I/O Timings、Sort / Hash / JIT 小节拆开，先证明慢点不是基数判断，再讨论执行期资源。

### 现象 3：Nested Loop 内层 loops 很高

内层 actual rows 需要结合 loops 才能理解总访问量。

`ExplainNode()` 输出 Actual Rows 与 Actual Loops，报告中必须同时保存。

记录时同时写 Actual Rows、Actual Loops、内层索引条件和外层行数；只看单次 rows 会低估总访问量。

### 现象 4：startup cost 低的计划被选中

LIMIT、cursor 或 semi join 可能更重视早返回。

不要只比较 total cost；回到 cheapest startup / total path。

记录时保留 startup cost、total cost、LIMIT / cursor / semi join 语义和 cheapest startup path 证据，避免只用总耗时反推 planner。

### 现象 5：width 估错导致内存节点异常

Hash、Sort、Material 受 avg width 和 targetlist 影响。

查看 plan_width、pg_stats.avg_width、toasted 数据和投影表达式。

记录时把 Plan Width、pg_stats.avg_width、Output 表达式、Sort / Hash 内存或 temp 文件放在一起，判断问题是宽度估算还是内存边界。

### 现象 6：Planning Time 高于 Execution Time

瓶颈可能在 join search、partition pruning、统计读取或表达式处理。

这类问题要读 planner 调用链，而不是 executor 节点。

记录时把 Planning Time、join 数、partition 数、GEQO / collapse GUC 和是否使用 prepared statement 放在一起，定位 planner 阶段而不是 executor 节点。

## 5. 关键数据结构与状态边界

本节只解释会影响诊断的字段组合。字段本身不是语义；字段加上阶段、owner、生命周期和下游消费者，才构成可用判断。

| 状态 | 一句话语义 |
| --- | --- |
| `Plan.plan_rows` | planner 认为该节点输出的行数。 |
| `Plan.startup_cost` | 产生第一行前的估算成本。 |
| `Plan.total_cost` | 完整消费节点输出的估算成本。 |
| `Plan.plan_width` | 平均 tuple 宽度估算。 |
| `Instrumentation.ntuples` | executor 实际返回 tuple 数。 |
| `Instrumentation.nloops` | 节点被执行的轮数。 |
| `BufferUsage` | shared/local/temp block 读写计数。 |
| `WalUsage` | DML 或相关节点产生的 WAL 观测。 |

### `Plan.plan_rows`

planner 认为该节点输出的行数。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 1 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `Plan.startup_cost`

产生第一行前的估算成本。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 2 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `Plan.total_cost`

完整消费节点输出的估算成本。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 3 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `Plan.plan_width`

平均 tuple 宽度估算。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 4 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `Instrumentation.ntuples`

executor 实际返回 tuple 数。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 5 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `Instrumentation.nloops`

节点被执行的轮数。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 6 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `BufferUsage`

shared/local/temp block 读写计数。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 7 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `WalUsage`

DML 或相关节点产生的 WAL 观测。

它服务于 `EXPLAIN 诊断 planner 判断` 的第 8 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

## 6. 主流程源码 walkthrough

下面按时间线阅读主流程。每一步都要问：输入状态是什么，输出状态是什么，哪些信息继续进入下一阶段。

```text
生成或接收 PlannedStmt
  -> 建立执行环境
  -> 输出估算字段
  -> 采集执行指标
  -> 递归打印计划树
  -> 补充节点资源
  -> 定位首个偏差
  -> 回到源码入口
```

### 步骤 1：生成或接收 PlannedStmt

ExplainOneQuery 可以先调用 planner，也可以解释已有 plan。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 2：建立执行环境

ExplainOnePlan 在 ANALYZE 情况下通过 executor 跑计划。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 3：输出估算字段

ExplainNode 读取 Plan 的 cost、rows、width 和节点类型。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 4：采集执行指标

InstrStartNode / InstrStopNode 累计时间、tuple、loops 和 buffer 差值。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 5：递归打印计划树

Explain 递归输出 Outer、Inner、InitPlan、SubPlan 等关系。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 6：补充节点资源

Sort、Hash、Memoize、Gather 等节点输出自己的资源细节。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 7：定位首个偏差

从叶子向根比较 estimated 与 actual。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 8：回到源码入口

rows 偏差回统计/选择率，cost 偏差回 costsize，候选缺失回 path generation。

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

围绕 `EXPLAIN 诊断 planner 判断` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 2：语义优先

优化器可以换路径，不能换 SQL 结果。

围绕 `EXPLAIN 诊断 planner 判断` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 3：单变量验证

每次实验只改变一个 planner 可见输入。

围绕 `EXPLAIN 诊断 planner 判断` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 4：证据闭环

现象、状态、源码入口和复测必须能互相支撑。

围绕 `EXPLAIN 诊断 planner 判断` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 5：副作用可控

EXPLAIN ANALYZE、DML、CREATE INDEX 和全局 GUC 都可能影响生产。

围绕 `EXPLAIN 诊断 planner 判断` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 6：范围说明

任何修复都要写清影响哪些相邻查询和写入路径。

围绕 `EXPLAIN 诊断 planner 判断` 做优化时，任何建议都必须先说明没有越过这个边界。

## 9. 错误路径 / 异常路径 / fallback

很多分支不是错误，而是在信息不足或语义受限时选择保守路径。诊断时要区分“缺少事实”和“事实被错误解释”。

### fallback 1：只看最慢节点

在 `EXPLAIN 诊断 planner 判断` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 2：把 cost 当实际时间

在 `EXPLAIN 诊断 planner 判断` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 3：忽略 loops

在 `EXPLAIN 诊断 planner 判断` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 4：看到 BUFFERS 就认定 planner 错

在 `EXPLAIN 诊断 planner 判断` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 5：用 enable_* 当修复

在 `EXPLAIN 诊断 planner 判断` 中，这类情况通常说明当前证据还不能支撑最终修复。

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

`EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)` 的价值在于：保存估算、实际、资源、输出和 GUC。 收集后要回到 `EXPLAIN 诊断 planner 判断` 的主问题，而不是把指标堆成附件。

`EXPLAIN (FORMAT JSON)` 的价值在于：便于脚本 diff 和计算 rows ratio。 收集后要回到 `EXPLAIN 诊断 planner 判断` 的主问题，而不是把指标堆成附件。

`pg_class / pg_stats / pg_statistic_ext` 的价值在于：确认 planner 能看到的数据分布。 收集后要回到 `EXPLAIN 诊断 planner 判断` 的主问题，而不是把指标堆成附件。

`SHOW / pg_settings` 的价值在于：保存 cost、enable、parallel、JIT、work_mem 和 plan cache 设置。 收集后要回到 `EXPLAIN 诊断 planner 判断` 的主问题，而不是把指标堆成附件。

`gdb / lldb 断点` 的价值在于：观察状态写入和消费，不把一次指针值当长期事实。 收集后要回到 `EXPLAIN 诊断 planner 判断` 的主问题，而不是把指标堆成附件。

## 12. 常见误区

### 误区 1：只看最慢节点

这个判断在 `EXPLAIN 诊断 planner 判断` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 2：把 cost 当实际时间

这个判断在 `EXPLAIN 诊断 planner 判断` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 3：忽略 loops

这个判断在 `EXPLAIN 诊断 planner 判断` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 4：看到 BUFFERS 就认定 planner 错

这个判断在 `EXPLAIN 诊断 planner 判断` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 5：用 enable_* 当修复

这个判断在 `EXPLAIN 诊断 planner 判断` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 6：只保存截图

这个判断在 `EXPLAIN 诊断 planner 判断` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

## 13. 课堂实验

实验不追求覆盖所有计划形态，只要求把本节主问题放进可复测闭环。建议在独立测试库执行，并记录每次 GUC、统计和数据规模。

### 实验 1：rows 偏差

比较 scan 节点 estimated / actual rows。

```sql
CREATE TABLE t_exp(a int, b int);
INSERT INTO t_exp SELECT g % 10, g % 10 FROM generate_series(1, 100000) g;
ANALYZE t_exp;
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT * FROM t_exp WHERE a = 1 AND b = 1;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 2：loops 放大

读取内层节点 rows 与 loops。

```sql
CREATE TABLE t_outer(id int);
INSERT INTO t_outer SELECT generate_series(1, 100);
ANALYZE t_outer;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t_outer o WHERE EXISTS (SELECT 1 FROM t_exp e WHERE e.a = o.id % 10);
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 3：cost 与时间

同时保存 cost、actual time、buffers 和 settings。

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) SELECT count(*) FROM t_exp;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 4：TIMING 开销

比较 TIMING ON/OFF 对信息和开销的影响。

```sql
EXPLAIN (ANALYZE, BUFFERS, TIMING OFF) SELECT count(*) FROM t_exp;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 5：断点

说明 plan_rows 与 ntuples 分别在何处写入。

```gdb
break ExplainNode
break InstrStopNode
break copy_generic_path_info
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

## 14. 源码练习

### 练习 1

围绕 `Plan.plan_rows` 设计一个断点或日志输出，说明它在 `EXPLAIN 诊断 planner 判断` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 2

围绕 `Plan.startup_cost` 设计一个断点或日志输出，说明它在 `EXPLAIN 诊断 planner 判断` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 3

围绕 `Plan.total_cost` 设计一个断点或日志输出，说明它在 `EXPLAIN 诊断 planner 判断` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 4

围绕 `Plan.plan_width` 设计一个断点或日志输出，说明它在 `EXPLAIN 诊断 planner 判断` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 5

围绕 `Instrumentation.ntuples` 设计一个断点或日志输出，说明它在 `EXPLAIN 诊断 planner 判断` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

## 15. 讨论题

1. 如果只能保存一条证据，哪条最能回答：如何从 EXPLAIN 字段读出 planner 当时相信了什么？

2. `EXPLAIN 诊断 planner 判断` 中哪些状态是 planner-local，哪些会进入 PlannedStmt 或 PlanState？

3. 遇到计划不符合直觉时，如何区分 SQL 语义限制、统计不足、成本参数偏差和搜索空间剪枝？

4. 如果修复需要全局 GUC 或新增索引，它会影响哪些其它查询？

5. 这个机制体现的可迁移规律是提前规范化、保守 fallback、显式 contract，还是候选剪枝？

讨论要落到可观测现象、源码入口和单变量复测。只给经验描述，不能支撑内核级诊断结论。

## 16. 诊断记录模板

这一节的诊断记录围绕 `EXPLAIN 诊断 planner 判断` 展开。模板不是替代分析，而是保证每次判断都能回到事实、状态和源码。

### 必须保存的事实

- 完整 SQL、参数形态、版本、schema、索引、统计和 planner GUC。
- 基线 EXPLAIN 与所有单变量对照 EXPLAIN。

### 判断问题

- 当前证据是否直接回答 `如何从 EXPLAIN 字段读出 planner 当时相信了什么？`
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

下面这段不是额外模板，而是把 `EXPLAIN 诊断 planner 判断` 落到一份诊断报告时应有的叙述粒度。

#### 1. 现象边界

先写清楚现象：estimated rows 与 actual rows 在某个节点第一次分叉。

这句话要避免混入修复建议。现象段只描述可观察事实，例如 EXPLAIN 字段、SQL 形态、参数分布、统计快照或资源指标。

如果现象来自线上慢 SQL，还要写明采集窗口、版本、数据库、schema、参数值来源和是否使用 prepared statement。

#### 2. 第一处判断

然后指出第一处能支撑结论的 planner 判断，证据集中在：rows、loops、startup cost、total cost、BUFFERS。

这里不要跳到最终根节点。根节点往往已经混合了多层传播，真正可修复的位置通常更靠近 scan、join 输入、path 生成或 finalization 边界。

如果只能说明“计划看起来不合理”，还没有找到第一处判断；继续回到叶子节点、catalog 统计或 path 生成断点。

#### 3. 源码落点

源码落点写成一个短链路，例如：`ExplainNode()` / `copy_generic_path_info()`。

链路里每个函数只承担一个角色：有的写状态，有的消费状态，有的做合法性检查，有的只是把结果打印出来。

报告中应说明你引用的是哪个角色，而不是把函数名当成结论。

#### 4. 单变量对照

对照实验只改变一个输入。

可以改变统计、索引、SQL 形态、session 级 GUC 或参数值，但一次只改一个。

每个对照都记录三件事：改动命令、预期变化、实际计划差异。

如果计划没有按预期变化，不要继续叠加第二个改动；先解释第一个假设为什么没有成立。

#### 5. 修复边界

修复风险要显式写出：只看最慢节点会把下游执行结果误判为根因。

短期止血可以是 session 级开关、临时索引或 SQL 改写；长期方案必须说明为什么它让 planner 看到更准确的事实，或者为什么它缩小了错误搜索空间。

如果方案影响全局参数、共享统计或写入路径，要补一组代表性查询回归，而不是只给一条 SQL 的 before/after。

#### 6. 报告结论

结论段建议压缩成四句话。

第一句：本案属于 `EXPLAIN 诊断 planner 判断`，不是泛泛的“优化器选错”。

第二句：首个错误判断点在哪里，证据是什么。

第三句：源码入口是 `ExplainNode()` / `copy_generic_path_info()`，它怎样消费这些证据。

第四句：最小修复是什么，回归风险是什么。

这样写的好处是，另一个工程师不需要相信你的经验；他可以按同一组 SQL、catalog、GUC 和源码入口复测。

#### 7. 复查口径

复查时先检查证据是否仍然成立，再检查修复是否仍然最小。

统计刷新、版本升级、索引变化、参数分布变化和 workload 变化，都可能让旧结论失效。

因此报告要保存采集时间和源码基线；否则后续复盘只能看到结论，看不到结论成立的条件。

对 EXPLAIN 诊断而言，这个复查口径尤其重要：同一条 SQL 在统计、GUC 或参数形态变化后，可能仍然打印相似节点名，却已经代表不同 planner 判断。

## 17. 本节小结

本节唯一主问题是：如何从 EXPLAIN 字段读出 planner 当时相信了什么？

- `EXPLAIN 诊断 planner 判断` 必须按阶段解释，不能只看最终计划形状。
- 可靠诊断要把现象、状态、源码入口和单变量复测连成闭环。
- EXPLAIN 展示的是胜出计划和执行观测，不展示完整搜索历史。
- 修复建议要说明影响范围和回归风险。

当后续遇到新的 planner 问题时，先定位阶段，再判断边界，最后选择最小修复。
