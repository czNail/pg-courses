# PostgreSQL 可重复的 Planner 调试流程

## 课程定位

前置知识：已经理解 PostgreSQL parser / analyzer / rewriter 之后的 Query 进入 planner，知道 RelOptInfo、Path、Plan、PlannedStmt 和 executor 的基本边界。

本节唯一主问题：

```text
面对一条慢 SQL，如何按“收集 EXPLAIN、确认 schema / stats、定位首个估错点、验证替代 path、最小化 SQL、选择修复手段”的顺序形成可复现的优化器诊断报告？
```

核心矛盾：慢 SQL 现场往往混合了统计、参数、索引、SQL 写法、缓存、并发和执行期等待。没有流程时，诊断会变成试参数、猜索引和一次性经验；有流程时，每一步都要留下可复测证据，但流程又不能重到阻塞生产排障。

学完后应能独立产出一份 planner 诊断报告，说明现象、证据、根因候选、已排除路径、最小复现、修复方案和回归风险。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前几节分别讨论 EXPLAIN 判断、rows 估错、cost 参数和 path 搜索。本节把这些能力收束成一套可重复的慢 SQL planner 诊断流程。

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
诊断从不可变事实开始：SQL、schema、统计、GUC、数据规模和 EXPLAIN；然后找第一处 planner 判断偏差；接着用最小变量对照验证替代 path；最后选择最小修复并写清它影响的 optimizer 阶段。
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
| 1 | `src/backend/commands/explain.c` | EXPLAIN 是诊断入口，输出估算、实际、buffer、wal、settings 和节点特有资源。 |
| 2 | `src/backend/optimizer/plan/planner.c` | standard_planner() 给出 Query 到 PlannedStmt 的阶段边界。 |
| 3 | `src/backend/optimizer/path/allpaths.c` | base rel、upper rel、join search 的 path 生成入口。 |
| 4 | `src/backend/optimizer/path/costsize.c` | rows 与 cost 判断的集中实现。 |
| 5 | `src/backend/optimizer/path/clausesel.c` | 选择率组合和 extended stats 介入位置。 |
| 6 | `src/backend/optimizer/path/indxpath.c` | 索引 path 是否生成的关键证据。 |
| 7 | `src/backend/optimizer/path/joinrels.c` | join order 是否可枚举的关键证据。 |
| 8 | `src/backend/optimizer/util/pathnode.c` | path 被保留、剪枝和选择的关键证据。 |
| 9 | `src/backend/commands/analyze.c` | 统计刷新与采样边界。 |

阅读顺序按 mental model 排列，不按目录名排序。先看入口和状态结构，再看引用替换、fallback、成本传播和观测输出。

源码核对命令：

```bash
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

本节只引用当前本地源码中真实存在的路径和函数。未来版本如果移动实现位置，应优先保持这里的系统边界，再更新具体入口。

## 4. 从可观测现象进入源码

本节从 `可复现 planner 诊断报告` 的可观测现象进入源码。先确认现象属于 Query、Path、Plan、PlannedStmt、PlanState 还是 catalog 统计，再决定要读哪个文件。

```text
现象
  -> 阶段边界
     -> 状态写入点
        -> 状态消费点
           -> 单变量复测
```

### 现象 1：同一 SQL 今天突然变慢

先比较 schema、统计时间、数据量、GUC、plan cache、参数值和版本。

报告第一部分写变化点和不可变事实。

记录时按时间线列出 schema 变更、ANALYZE / autovacuum、数据规模、GUC、参数值、plan cache 状态和前后 EXPLAIN，先确定哪个输入变了。

### 现象 2：开发环境复现不了生产计划

数据分布、统计目标、collation、扩展、缓存和版本都可能不同。

最小复现必须包含 catalog 摘要和 GUC。

记录时把版本、locale / collation、扩展、统计目标、pg_stats 摘要、表大小和代表性参数带进最小复现，避免只迁移 SQL 文本。

### 现象 3：ANALYZE 后恢复

根因可能是 stale stats，也可能是采样覆盖热点。

保存 ANALYZE 前后 pg_stats 与 EXPLAIN 差异。

记录时保存 ANALYZE 前后统计 slots、采样目标、计划 JSON、rows ratio 和持续时间，判断修复来自统计刷新还是偶然采样。

### 现象 4：enable_* 开关让计划变好

这是定位证据，不是长期修复。

说明它证明的是候选存在、成本落选还是 join method 受限。

记录时把被关闭的 path 类型、替代计划 cost、实际耗时、rows ratio 和源码阶段写清楚；enable_* 只说明诊断方向，不是默认修复。

### 现象 5：新增索引后查询变快但写入变慢

planner 修复常常把成本转移到 DML、存储和维护。

报告需要包含副作用和适用查询范围。

记录时同时写查询收益、DML 写放大、索引大小、维护成本、适用谓词和回退方案，避免把单条 SQL 的改善当全局收益。

### 现象 6：计划快慢随参数变化

generic plan 可能看不到具体常量分布，custom plan 规划成本更高。

保存参数分布和 plan_cache_mode 对照。

记录时按参数桶保存 custom / generic plan、Plan Rows、Actual Rows、plan_cache_mode 和执行频率，说明问题是常量不可见还是规划成本权衡。

## 5. 关键数据结构与状态边界

本节只解释会影响诊断的字段组合。字段本身不是语义；字段加上阶段、owner、生命周期和下游消费者，才构成可用判断。

| 状态 | 一句话语义 |
| --- | --- |
| `SQL text` | 诊断对象的完整文本。 |
| `bind parameter shape` | 参数类型、典型值、热点值和分布。 |
| `schema and indexes` | 表结构、索引、约束、分区、collation 和 partial/expression index。 |
| `statistics snapshot` | pg_class、pg_stats、pg_statistic_ext 和 statistics target。 |
| `planner GUC snapshot` | cost、enable、parallel、JIT、work_mem、plan_cache_mode 等设置。 |
| `baseline EXPLAIN` | 最初计划证据。 |
| `first wrong judgement` | 第一个可证明偏离的 planner 判断。 |
| `one-variable experiment` | 只改变一个输入后的复测。 |

### `SQL text`

诊断对象的完整文本。

它服务于 `可复现 planner 诊断报告` 的第 1 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `bind parameter shape`

参数类型、典型值、热点值和分布。

它服务于 `可复现 planner 诊断报告` 的第 2 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `schema and indexes`

表结构、索引、约束、分区、collation 和 partial/expression index。

它服务于 `可复现 planner 诊断报告` 的第 3 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `statistics snapshot`

pg_class、pg_stats、pg_statistic_ext 和 statistics target。

它服务于 `可复现 planner 诊断报告` 的第 4 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `planner GUC snapshot`

cost、enable、parallel、JIT、work_mem、plan_cache_mode 等设置。

它服务于 `可复现 planner 诊断报告` 的第 5 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `baseline EXPLAIN`

最初计划证据。

它服务于 `可复现 planner 诊断报告` 的第 6 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

### `first wrong judgement`

第一个可证明偏离的 planner 判断。

它服务于 `可复现 planner 诊断报告` 的第 7 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

### `one-variable experiment`

只改变一个输入后的复测。

它服务于 `可复现 planner 诊断报告` 的第 8 个状态边界。调试时要同时记录写入函数、消费函数和所属 plan node 或 catalog 对象。

如果 EXPLAIN 中看不到它，不要立刻认为它没有参与；很多 planner-local 状态只影响候选生成或最终替换，不会直接打印。

这类状态尤其不能跨阶段比较。Path 阶段的值、Plan 阶段的值和执行期 instrumentation 的值，即使名字相似，也可能回答不同问题。

## 6. 主流程源码 walkthrough

下面按时间线阅读主流程。每一步都要问：输入状态是什么，输出状态是什么，哪些信息继续进入下一阶段。

```text
冻结现场
  -> 确认是否真是 planner 问题
  -> 定位首个判断偏差
  -> 映射源码入口
  -> 构造最小复现
  -> 做单变量对照
  -> 选择最小修复
  -> 写回归风险
```

### 步骤 1：冻结现场

保存 SQL、参数、版本、GUC、schema、统计、EXPLAIN 和影响范围。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 2：确认是否真是 planner 问题

用 waits、locks、I/O、CPU、JIT、BUFFERS 排除纯执行期资源问题。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 3：定位首个判断偏差

从 EXPLAIN 叶子到根找 rows、cost、width、path 或 loops 异常。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 4：映射源码入口

rows 去 clausesel/costsize，索引去 indxpath，join 去 joinrels/joinpath。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 5：构造最小复现

删减 SQL 但保留触发同一 planner 判断的分布和语义结构。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 6：做单变量对照

只改统计、索引、GUC、SQL 形态或参数中的一个。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 7：选择最小修复

优先让 planner 看到更准事实；全局 GUC 和强制开关放最后。

断点应尽量放在状态写入之后；函数入口只能证明代码路径经过，写入后的字段才能解释下游行为。

如果这一步没有发生，优先检查调用条件、query level、path 类型和 GUC，而不是从最终计划补猜测。

### 步骤 8：写回归风险

列出受同一统计、索引、GUC、SQL pattern 影响的其它查询和写入路径。

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

围绕 `可复现 planner 诊断报告` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 2：语义优先

优化器可以换路径，不能换 SQL 结果。

围绕 `可复现 planner 诊断报告` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 3：单变量验证

每次实验只改变一个 planner 可见输入。

围绕 `可复现 planner 诊断报告` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 4：证据闭环

现象、状态、源码入口和复测必须能互相支撑。

围绕 `可复现 planner 诊断报告` 做优化时，任何建议都必须先说明没有越过这个边界。

### 不变量 5：副作用可控

EXPLAIN ANALYZE、DML、CREATE INDEX 和全局 GUC 都可能影响生产。

围绕 `可复现 planner 诊断报告` 做优化时，任何建议都必须先说明没有越过这个边界。

源码中这类约束通常表现为合法性检查、保守 fallback、Assert 或候选根本不生成。

### 不变量 6：范围说明

任何修复都要写清影响哪些相邻查询和写入路径。

围绕 `可复现 planner 诊断报告` 做优化时，任何建议都必须先说明没有越过这个边界。

## 9. 错误路径 / 异常路径 / fallback

很多分支不是错误，而是在信息不足或语义受限时选择保守路径。诊断时要区分“缺少事实”和“事实被错误解释”。

### fallback 1：先建索引再找根因

在 `可复现 planner 诊断报告` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 2：只贴慢计划截图

在 `可复现 planner 诊断报告` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 3：同时改多个参数

在 `可复现 planner 诊断报告` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 4：把 enable_* 当长期配置

在 `可复现 planner 诊断报告` 中，这类情况通常说明当前证据还不能支撑最终修复。

处理方式是补充 planner 可见事实，或调整 SQL 形态让已有事实可用；直接压制 fallback 往往会把风险转给其它查询。

复测时只改变一个输入，再确认计划变化是否符合源码路径。

### fallback 5：最小复现过度简化

在 `可复现 planner 诊断报告` 中，这类情况通常说明当前证据还不能支撑最终修复。

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

`EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)` 的价值在于：保存估算、实际、资源、输出和 GUC。 收集后要回到 `可复现 planner 诊断报告` 的主问题，而不是把指标堆成附件。

`EXPLAIN (FORMAT JSON)` 的价值在于：便于脚本 diff 和计算 rows ratio。 收集后要回到 `可复现 planner 诊断报告` 的主问题，而不是把指标堆成附件。

`pg_class / pg_stats / pg_statistic_ext` 的价值在于：确认 planner 能看到的数据分布。 收集后要回到 `可复现 planner 诊断报告` 的主问题，而不是把指标堆成附件。

`SHOW / pg_settings` 的价值在于：保存 cost、enable、parallel、JIT、work_mem 和 plan cache 设置。 收集后要回到 `可复现 planner 诊断报告` 的主问题，而不是把指标堆成附件。

`gdb / lldb 断点` 的价值在于：观察状态写入和消费，不把一次指针值当长期事实。 收集后要回到 `可复现 planner 诊断报告` 的主问题，而不是把指标堆成附件。

## 12. 常见误区

### 误区 1：先建索引再找根因

这个判断在 `可复现 planner 诊断报告` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 2：只贴慢计划截图

这个判断在 `可复现 planner 诊断报告` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 3：同时改多个参数

这个判断在 `可复现 planner 诊断报告` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 4：把 enable_* 当长期配置

这个判断在 `可复现 planner 诊断报告` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 5：最小复现过度简化

这个判断在 `可复现 planner 诊断报告` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

### 误区 6：只优化一条 SQL

这个判断在 `可复现 planner 诊断报告` 中容易把阶段边界混在一起。

更可靠的做法是先确认当前证据能解释哪个阶段；解释不了时补证据，不提前给修复结论。

## 13. 课堂实验

实验不追求覆盖所有计划形态，只要求把本节主问题放进可复测闭环。建议在独立测试库执行，并记录每次 GUC、统计和数据规模。

### 实验 1：冻结现场

把输出放入报告开头。

```sql
SELECT version();
SHOW random_page_cost;
SHOW effective_cache_size;
SHOW work_mem;
SHOW max_parallel_workers_per_gather;
SHOW plan_cache_mode;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 2：保存基线计划

真实诊断时替换为目标 SQL。

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS) SELECT 1;
EXPLAIN (FORMAT JSON, ANALYZE, BUFFERS, SETTINGS) SELECT 1;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 3：保存 schema 和统计

只保存相关对象时要说明筛选条件。

```sql
SELECT relname, reltuples, relpages FROM pg_class WHERE relkind IN ('r','p') ORDER BY relname;
SELECT schemaname, tablename, attname, null_frac, n_distinct, correlation FROM pg_stats ORDER BY schemaname, tablename, attname;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 4：单变量对照

写明这个对照验证哪一个假设。

```sql
SET enable_hashjoin = off;
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) SELECT 1;
RESET enable_hashjoin;
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

### 实验 5：报告草稿

用草稿约束报告，而不是直接写结论。

```sql
-- 1. 现象和影响范围
-- 2. 基线计划和环境
-- 3. 首个错误判断点
-- 4. 单变量对照
-- 5. 源码入口
-- 6. 修复建议和回归风险
```

记录基线、改动、预期、实际计划差异和源码入口。执行后撤回临时 GUC 或事务改动，保证下一次实验从可解释状态开始。

## 14. 源码练习

### 练习 1

围绕 `SQL text` 设计一个断点或日志输出，说明它在 `可复现 planner 诊断报告` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 2

围绕 `bind parameter shape` 设计一个断点或日志输出，说明它在 `可复现 planner 诊断报告` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 3

围绕 `schema and indexes` 设计一个断点或日志输出，说明它在 `可复现 planner 诊断报告` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 4

围绕 `statistics snapshot` 设计一个断点或日志输出，说明它在 `可复现 planner 诊断报告` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

### 练习 5

围绕 `planner GUC snapshot` 设计一个断点或日志输出，说明它在 `可复现 planner 诊断报告` 中第一次写入和第一次被消费的位置。

达标条件是能把这个字段和一个 EXPLAIN 现象对应起来，而不是只说明断点命中。

## 15. 讨论题

1. 如果只能保存一条证据，哪条最能回答：如何形成可重复的 planner 调试流程？

2. `可复现 planner 诊断报告` 中哪些状态是 planner-local，哪些会进入 PlannedStmt 或 PlanState？

3. 遇到计划不符合直觉时，如何区分 SQL 语义限制、统计不足、成本参数偏差和搜索空间剪枝？

4. 如果修复需要全局 GUC 或新增索引，它会影响哪些其它查询？

5. 这个机制体现的可迁移规律是提前规范化、保守 fallback、显式 contract，还是候选剪枝？

讨论要落到可观测现象、源码入口和单变量复测。只给经验描述，不能支撑内核级诊断结论。

## 16. 诊断记录模板

这一节的诊断记录围绕 `可复现 planner 诊断报告` 展开。模板不是替代分析，而是保证每次判断都能回到事实、状态和源码。

### 必须保存的事实

- 完整 SQL、参数形态、版本、schema、索引、统计和 planner GUC。
- 基线 EXPLAIN 与所有单变量对照 EXPLAIN。

### 判断问题

- 当前证据是否直接回答 `如何形成可重复的 planner 调试流程？`
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

下面这段不是额外模板，而是把 `可复现 planner 诊断报告` 落到一份诊断报告时应有的叙述粒度。

#### 1. 现象边界

先写清楚现象：同一 SQL 在生产变慢但开发环境复现不了。

这句话要避免混入修复建议。现象段只描述可观察事实，例如 EXPLAIN 字段、SQL 形态、参数分布、统计快照或资源指标。

如果现象来自线上慢 SQL，还要写明采集窗口、版本、数据库、schema、参数值来源和是否使用 prepared statement。

#### 2. 第一处判断

然后指出第一处能支撑结论的 planner 判断，证据集中在：SQL、参数、schema、stats、GUC、EXPLAIN JSON。

这里不要跳到最终根节点。根节点往往已经混合了多层传播，真正可修复的位置通常更靠近 scan、join 输入、path 生成或 finalization 边界。

如果只能说明“计划看起来不合理”，还没有找到第一处判断；继续回到叶子节点、catalog 统计或 path 生成断点。

#### 3. 源码落点

源码落点写成一个短链路，例如：`standard_planner()` / `ExplainOnePlan()` / `plancache.c`。

链路里每个函数只承担一个角色：有的写状态，有的消费状态，有的做合法性检查，有的只是把结果打印出来。

报告中应说明你引用的是哪个角色，而不是把函数名当成结论。

#### 4. 单变量对照

对照实验只改变一个输入。

可以改变统计、索引、SQL 形态、session 级 GUC 或参数值，但一次只改一个。

每个对照都记录三件事：改动命令、预期变化、实际计划差异。

如果计划没有按预期变化，不要继续叠加第二个改动；先解释第一个假设为什么没有成立。

#### 5. 修复边界

修复风险要显式写出：缺少现场事实会让报告变成不可验证的经验建议。

短期止血可以是 session 级开关、临时索引或 SQL 改写；长期方案必须说明为什么它让 planner 看到更准确的事实，或者为什么它缩小了错误搜索空间。

如果方案影响全局参数、共享统计或写入路径，要补一组代表性查询回归，而不是只给一条 SQL 的 before/after。

#### 6. 报告结论

结论段建议压缩成四句话。

第一句：本案属于 `可复现 planner 诊断报告`，不是泛泛的“优化器选错”。

第二句：首个错误判断点在哪里，证据是什么。

第三句：源码入口是 `standard_planner()` / `ExplainOnePlan()` / `plancache.c`，它怎样消费这些证据。

第四句：最小修复是什么，回归风险是什么。

这样写的好处是，另一个工程师不需要相信你的经验；他可以按同一组 SQL、catalog、GUC 和源码入口复测。

#### 7. 复查口径

复查时先检查证据是否仍然成立，再检查修复是否仍然最小。

统计刷新、版本升级、索引变化、参数分布变化和 workload 变化，都可能让旧结论失效。

因此报告要保存采集时间和源码基线；否则后续复盘只能看到结论，看不到结论成立的条件。

## 17. 本节小结

本节唯一主问题是：如何形成可重复的 planner 调试流程？

- `可复现 planner 诊断报告` 必须按阶段解释，不能只看最终计划形状。
- 可靠诊断要把现象、状态、源码入口和单变量复测连成闭环。
- EXPLAIN 展示的是胜出计划和执行观测，不展示完整搜索历史。
- 修复建议要说明影响范围和回归风险。

当后续遇到新的 planner 问题时，先定位阶段，再判断边界，最后选择最小修复。
