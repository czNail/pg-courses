# PostgreSQL 从 actual / estimate 偏差定位问题层次

## 课程定位

前置知识：熟悉 EXPLAIN 的 Plan
Rows、Actual Rows、Actual
Loops、Rows Removed by
Filter、Join Filter、统计信息和执行器节点语义。

本节唯一主问题：

```text
rows 估计错误、loops 异常、过滤率异常和 join order 异常分别更可能属于统计信息、优化器选择、执行器行为还是数据分布问题？
```

核心矛盾：用户看到的是一个 estimate / actual
差值；但这个差值可能来自统计信息陈旧、选择率模型限制、参数化循环、运行期过滤、并行倾斜、执行器重扫或数据分布，而不是单一模块错误。

学完后应能判断：能把 EXPLAIN 偏差拆成
plan-time 估计、runtime
loops、filter/recheck、join order
和数据分布几层，避免把所有慢 SQL 都归因于执行器。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节建立了从 EXPLAIN 节点名回到源码入口的方法。

04 目录的总主线不是背执行节点名称，而是把
PlanState、tuple
流、执行期状态和观测指标放到同一条生命周期里。

```text
Plan / QueryDesc
  -> ExecutorStart 构造运行态
  -> ExecProcNode 推进 tuple 流
  -> instrumentation / pg_stat / wait event 记录现象
  -> EXPLAIN 把现象格式化
  -> 源码入口解释现象
```

本节只保留一个主问题。相邻机制会被引用，但只服务这条主线：先确认状态在哪一层产生，再确认它怎样被推进、观测和清理。

后续课程可以继续把 Buffers、I/O
timing、wait event 和 profiler
栈接入同一条诊断链。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
先看每个节点的 Plan Rows 与 Actual Rows/Loops，再看过滤计数和 join 形态；如果偏差在 scan 处出现，优先查统计和谓词选择率；如果偏差在 join 后放大，优先查 join selectivity、相关性和 join order；如果 loops 异常，回到 executor 调度和参数化路径。
```

这个模型要避免两个极端。一个极端是把源码读成 API
清单，知道每个函数名，却不知道哪一个状态在时间线上发生了变化。另一个极端是只看
EXPLAIN 或日志现象，把所有异常都归因到同一个模块。

读本节时要一直追问三个问题：第一，当前状态是
plan-time、executor
runtime、worker-local、DSM
还是格式化阶段状态；第二，它的 owner 和 cleanup
点是谁；第三，用户能通过哪个观测入口看到它的影响。

如果一个字段或函数不能帮助回答本节唯一主问题，就不要把它扩展成并列主题。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/commands/explain.c` | ExplainNode() 输出 Plan Rows、Actual Rows、Actual Loops，show_instrumentation_count() 输出 Rows Removed by Filter / Join Filter。 |
| 2 | `src/backend/executor/instrument.c` | InstrStopNode()、InstrEndLoop() 维护 ntuples、nloops、nfiltered。 |
| 3 | `src/include/executor/instrument.h` | NodeInstrumentation 字段解释 actual rows、loops、filtered 的来源。 |
| 4 | `src/backend/optimizer/path/costsize.c` | 计划成本和 rows 估算的主要入口，例如 cost_seqscan、cost_index、cost_hashjoin 等。 |
| 5 | `src/backend/optimizer/path/clausesel.c` | clause selectivity 组合和谓词选择率入口。 |
| 6 | `src/backend/optimizer/path/joinrels.c` | joinrel 构造和 join order 搜索的一部分。 |
| 7 | `src/backend/optimizer/plan/planner.c` | planner 主入口，连接 Query、PlannerInfo、Path 和 PlannedStmt。 |
| 8 | `src/backend/commands/analyze.c` | ANALYZE 如何采样并更新 pg_class / pg_statistic。 |
| 9 | `src/backend/statistics/extended_stats.c` | 扩展统计信息用于相关列、ndistinct 和 dependencies。 |

阅读顺序按 mental model
排列，不按文件名排序。先找入口，再找状态结构，再找
ownership 和 cleanup，最后才看观测输出。

本节源码核对基线：

```text
cd /home/highgo/postgres
git rev-parse --abbrev-ref HEAD
git rev-parse HEAD
```

期望输出分别是 `master` 和
`bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

## 4. 关键数据结构与状态边界

| 状态 | 本节语义 |
| --- | --- |
| `plan->plan_rows` | planner 写入的估计行数，ExplainNode 输出为 Plan Rows 或 text 中 rows。 |
| `NodeInstrumentation.ntuples` | 运行期累计输出 tuple 数，EXPLAIN 显示时除以 nloops。 |
| `NodeInstrumentation.nloops` | 节点完成的 run cycle 数，参数化内侧扫描和 rescan 会放大。 |
| `nfiltered1 / nfiltered2` | 节点过滤计数，ExplainNode 根据节点类型显示为 Rows Removed by Filter、Join Filter 或 Index Recheck。 |
| `pg_class.reltuples / relpages` | 表级规模估计基础，ANALYZE 或 VACUUM 后更新。 |
| `pg_statistic` | 列级 MCV、histogram、null fraction、correlation 等统计来源。 |
| `extended statistics` | 跨列相关性、ndistinct 和 functional dependency 的补充信息。 |
| `join order` | planner 根据 rows 和 cost 选择的连接顺序，早期偏差会沿 join tree 放大。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`plan->plan_rows` 的关键点是：planner
写入的估计行数，ExplainNode 输出为 Plan
Rows 或 text 中 rows。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`NodeInstrumentation.ntuples`
的关键点是：运行期累计输出 tuple 数，EXPLAIN
显示时除以 nloops。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`NodeInstrumentation.nloops`
的关键点是：节点完成的 run cycle 数，参数化内侧扫描和
rescan 会放大。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`nfiltered1 / nfiltered2`
的关键点是：节点过滤计数，ExplainNode
根据节点类型显示为 Rows Removed by
Filter、Join Filter 或 Index
Recheck。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pg_class.reltuples / relpages`
的关键点是：表级规模估计基础，ANALYZE 或 VACUUM
后更新。 诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pg_statistic` 的关键点是：列级
MCV、histogram、null
fraction、correlation 等统计来源。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`extended statistics`
的关键点是：跨列相关性、ndistinct 和
functional dependency 的补充信息。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`join order` 的关键点是：planner 根据
rows 和 cost 选择的连接顺序，早期偏差会沿 join
tree 放大。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

先在 EXPLAIN 中从叶子 scan 节点向上看 Plan
Rows 与 Actual Rows。

如果第一个大偏差出现在
scan，优先怀疑表统计、列统计、谓词选择率或数据分布。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

再看 Actual Loops。

如果内侧 index scan 每次 actual rows
很小但 loops 很大，慢可能来自 nested loop
参数化次数，而不是单次 scan 错误。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

看 Rows Removed by Filter / Join
Filter / Index Recheck。

过滤计数大说明节点拿到了很多候选
tuple，实际输出少，需要区分访问路径和 qual 选择。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

把偏差沿 join tree 往上跟踪。

scan 小偏差在多表 join 后可能指数放大，join
后才出现的大偏差则优先看 join selectivity。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

回到 `ExplainNode()`，确认 rows 显示口径是
ntuples / nloops。

不要把显示 rows 误读成全执行累计输出。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

回到 `InstrEndLoop()`，确认 loops
什么时候增加。

rescan、cursor、nested loop inner
path 都可能改变 loops 口径。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

如果怀疑统计，检查 `ANALYZE` 是否覆盖相关表和列。

ANALYZE 更新 pg_class 和
pg_statistic，但采样和统计目标仍有边界。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

如果怀疑相关列，检查是否需要 extended
statistics。

单列统计无法表达某些列间相关性，join 或多谓词选择率会失真。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

如果偏差来自并行节点，先看 worker rows 分布。

总 actual rows 正常不代表 worker
之间没有倾斜。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

最后才判断执行器行为是否异常。

执行器通常按 planner 给出的计划执行，偏差首先是
estimate 和数据事实之间的差，而不是
ExecProcNode 自己估错。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

把这条链路合起来看，本节关注的不是某个函数是否复杂，而是这些函数共同维护了一个可解释的状态故事。

## 6. 关键状态随时间推进

同一状态在不同时间点有不同含义。下面把本节主状态压成一条时间线，方便读源码时定位断点。

| 阶段 | 阅读问题 |
| --- | --- |
| 准备阶段 | 输入对象已经存在，但本节的 runtime 状态可能还没有创建；此时错误通常由调用者或上层协议负责。 |
| 初始化阶段 | owner、memory context、DSM slot、instrumentation 或 hook wrapper 被建立；这一步之后 cleanup 责任开始变明确。 |
| 首次执行 | 延迟初始化、worker launch、函数指针首调用或状态机首个 phase 往往在这里发生。 |
| 稳定推进 | tuple、计数器、phase、reader 或 filter 计数随调用推进；诊断时最需要把单次调用和累计口径分开。 |
| 尾部收敛 | EOF、worker finish、trigger drain、queue detach、InstrEndLoop 或 ExplainFlushWorkersState 会把临时状态固化成可输出指标。 |
| 清理释放 | MemoryContext、ResourceOwner、DSM、DSA、reader、worker_instrument 等按 owner 释放；这之后裸指针不再有语义。 |

准备阶段 的判断标准是：输入对象已经存在，但本节的
runtime
状态可能还没有创建；此时错误通常由调用者或上层协议负责。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

初始化阶段 的判断标准是：owner、memory
context、DSM slot、instrumentation
或 hook wrapper 被建立；这一步之后 cleanup
责任开始变明确。 如果现场问题发生在这个阶段，应该优先回看本节第
5 节对应的源码入口，而不是跳到无关模块。

首次执行 的判断标准是：延迟初始化、worker
launch、函数指针首调用或状态机首个 phase
往往在这里发生。 如果现场问题发生在这个阶段，应该优先回看本节第
5 节对应的源码入口，而不是跳到无关模块。

稳定推进
的判断标准是：tuple、计数器、phase、reader 或
filter
计数随调用推进；诊断时最需要把单次调用和累计口径分开。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

尾部收敛 的判断标准是：EOF、worker
finish、trigger drain、queue
detach、InstrEndLoop 或
ExplainFlushWorkersState
会把临时状态固化成可输出指标。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

清理释放
的判断标准是：MemoryContext、ResourceOwner、DSM、DSA、reader、worker_instrument
等按 owner 释放；这之后裸指针不再有语义。
如果现场问题发生在这个阶段，应该优先回看本节第 5
节对应的源码入口，而不是跳到无关模块。

## 7. 生命周期 / ownership / cleanup

生命周期问题要回答谁创建、谁持有、谁释放，以及 ERROR
或提前结束时谁兜底。

Plan Rows 在 planning 阶段写入 Plan
node，执行期间不会随着实际 rows 自动修正。

Actual Rows / Loops 在执行期间由
instrumentation 累计，EXPLAIN 打印前通过
InstrEndLoop 固化。

Rows Removed 由节点在 qual、joinqual
或 recheck 路径上更新，语义随节点类型不同。

pg_statistic 和 pg_class 是
catalog 状态，ANALYZE 更新后新计划才会使用。

prepared statement 的 generic
plan 可能继续使用旧选择，诊断时要确认计划生成时机。

并行 worker instrumentation 在
cleanup 后复制到 leader，EXPLAIN 才能输出
per-worker 明细。

这里最重要的边界是：MemoryContext 只能保证
backend-local 内存随 context 释放；DSM
/ DSA / tuple queue / worker /
hook 全局变量 / instrumentation
counter 还需要各自的 owner 协议。

因此不能写成“事务结束会释放”。本节所有对象都要说清楚是
query context、node private
context、ParallelContext、DSM、worker-local
state，还是扩展全局 hook 链。

## 8. 正确性机制层次

| 层次 | 机制与不变量 |
| --- | --- |
| 估计与执行分层 | plan_rows 是优化器估计，actual rows 是执行器观测，二者不同不等于执行器错误。 |
| loops 归一化 | EXPLAIN 显示的 actual rows 是每 loop 平均值，累计值要乘以 loops 近似恢复。 |
| 过滤口径 | Rows Removed by Filter、Join Filter、Index Recheck 对应不同 qual 层次。 |
| 统计时效 | ANALYZE 后新计划才使用新统计，已缓存计划可能不立刻变化。 |
| 数据相关性 | 单列统计不能表达所有跨列关系，extended statistics 也只覆盖声明过的组合。 |
| 模块归因 | 优化器选择、统计状态、执行器重扫、并行倾斜和 I/O 等待必须分别验证。 |

估计与执行分层 这一层保证的是：plan_rows
是优化器估计，actual rows
是执行器观测，二者不同不等于执行器错误。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

loops 归一化 这一层保证的是：EXPLAIN 显示的
actual rows 是每 loop 平均值，累计值要乘以
loops 近似恢复。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

过滤口径 这一层保证的是：Rows Removed by
Filter、Join Filter、Index Recheck
对应不同 qual 层次。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

统计时效 这一层保证的是：ANALYZE
后新计划才使用新统计，已缓存计划可能不立刻变化。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

数据相关性
这一层保证的是：单列统计不能表达所有跨列关系，extended
statistics 也只覆盖声明过的组合。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

模块归因
这一层保证的是：优化器选择、统计状态、执行器重扫、并行倾斜和
I/O 等待必须分别验证。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 把 rows 差异直接等同于 bug | 统计模型是近似，偏差本身先是诊断入口。 |
| 忽略 loops | inner scan rows=1 loops=100000 和 rows=100000 loops=1 是完全不同的问题。 |
| 只看根节点 | 根节点偏差可能是叶子多处小偏差累积。 |
| 忽略 Rows Removed | 大量过滤说明访问路径产生了候选 tuple，但 qual 在运行期丢掉它们。 |
| ANALYZE 后不重新计划 | 缓存计划或 prepared statement 可能仍用旧计划。 |
| 把并行总和当均衡 | worker 倾斜会隐藏在总 actual rows 中。 |

场景“把 rows 差异直接等同于
bug”的处理思路是：统计模型是近似，偏差本身先是诊断入口。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忽略 loops”的处理思路是：inner scan
rows=1 loops=100000 和
rows=100000 loops=1 是完全不同的问题。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“只看根节点”的处理思路是：根节点偏差可能是叶子多处小偏差累积。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忽略 Rows
Removed”的处理思路是：大量过滤说明访问路径产生了候选
tuple，但 qual 在运行期丢掉它们。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“ANALYZE 后不重新计划”的处理思路是：缓存计划或
prepared statement 可能仍用旧计划。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把并行总和当均衡”的处理思路是：worker
倾斜会隐藏在总 actual rows 中。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

提高 statistics target 会增加 ANALYZE
成本和 pg_statistic 体积，但可能降低选择率误差。

extended statistics 能改善相关列估计，但需要
DBA 明确声明组合，不能自动覆盖所有关系。

选择更稳的计划有时牺牲理想情况下的最优成本，换取估计偏差下的鲁棒性。

记录更多 EXPLAIN
选项会增加诊断信息，也增加运行和日志成本。

参数化 nested loop 在单次 inner
很快时仍可能因 loops 放大而慢。

并行计划估计错误可能同时放大 worker 启动、同步、I/O
和 tuple routing 成本。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

从叶子到根标出每个节点的 Plan Rows、Actual
Rows、Actual Loops。

把 actual rows 乘 loops
得到近似累计输出，再和父节点输入关系对照。

查看 Rows Removed by Filter / Join
Filter / Index Recheck，判断候选
tuple 在哪层被丢弃。

用 `ANALYZE VERBOSE` 和 pg_stats /
pg_stats_ext 查看统计覆盖情况。

用 FORMAT JSON 保留数值字段，避免 text
中缩进和四舍五入影响自动分析。

对 prepared statement 分别测试 custom
plan 和 generic plan 的差异。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. 叶子偏差定位

构造一张倾斜分布表

执行带稀有值和热门值谓词的 EXPLAIN ANALYZE

比较 scan 节点 Plan Rows 与 Actual
Rows

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. loops 放大定位

构造 Nested Loop + Index Scan 查询

观察 inner index scan 的 rows 和
loops

计算累计 inner tuple 访问次数

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. 过滤率观察

添加不能被索引完全利用的过滤条件

查看 Rows Removed by Filter

回到节点源码确认过滤计数更新位置

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 扩展统计验证

对相关列创建 extended statistics

ANALYZE 后重新 EXPLAIN

比较 join 或多列谓词估计是否改善

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 缓存计划差异

PREPARE 带参数查询

用不同参数 EXECUTE

观察 generic plan 与 custom plan
可能的估计偏差

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

这些实验不要求一次证明所有结论。每次只验证一个状态边界，符合本目录“一个主问题一课”的写法。

## 13. 常见误区

把 EXPLAIN 的显示文本当作内部结构本身。

把一个计数器当成完整语义，忽略
loops、worker、phase 或 cleanup。

把 planner-time 的状态和
executor-time 的状态混用。

把 leader 本地状态和 worker 本地状态混用。

把 DSM 中的共享数据和 backend-local
指针混用。

只看正常路径，不读
ERROR、fallback、shutdown 或
EXPLAIN ONLY 路径。

这些误区的共同根源，是没有先问“这个状态属于哪个生命周期”。只要先定位生命周期，大多数源码路径都会自然收敛。

## 14. 讨论题

1. 为什么 estimate / actual
偏差不能直接归因到执行器？

2. 如何用 loops 区分单次节点慢和重复调用过多？

3. Rows Removed by Filter 与
Index Recheck 分别提示什么层次的问题？

4. ANALYZE 之后偏差仍然大，可能还缺哪些统计表达能力？

5. 并行 worker rows 倾斜与 Plan Rows
偏差有什么关系？

6. 什么时候应该改 SQL / schema /
statistics，而不是改 GUC 或源码？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：estimate / actual 偏差如何分层归因

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. scan 处已经偏差巨大

现象是叶子 Seq Scan 或 Index Scan 的
Plan Rows 与 Actual Rows
差几个数量级。优先查
pg_class、pg_statistic、谓词选择率和数据分布。

先运行 ANALYZE，再用 pg_stats 查看
n_distinct、most_common_vals、histogram_bounds
和 correlation。

结论通常在统计层或数据分布层，而不是执行器把 tuple
数算错。

### 15.2. join 后偏差才放大

现象是单表 scan 估计还可以，但 join 后 rows
激增或骤降。优先查 join
selectivity、列相关性、外键关系和 join
order。

读 optimizer 的 clausesel、join
selectivity 相关路径，并对比是否需要
extended statistics。

结论是 join
偏差常由相关性和独立性假设触发。修复可能是扩展统计、改写 SQL
或更新统计。

### 15.3. inner scan rows 小但 loops 巨大

现象是 EXPLAIN 中 inner Index Scan
rows=1，却 loops=数十万。慢点不是单次 index
scan，而是参数化 Nested Loop 重复次数。

把 rows 乘 loops 估算累计访问，再回到
NestLoop 的 outer 驱动和 PARAM_EXEC
传递。

结论是 loops 是执行器调度事实，不能只看每 loop
rows。

### 15.4. Rows Removed by Filter 很高

现象是访问路径产生大量候选
tuple，但过滤后输出很少。要区分索引条件、recheck
条件、scan qual 和 join qual。

沿 ExplainNode 的
show_instrumentation_count()
看不同节点把 nfiltered1 / nfiltered2
显示成什么标签。

结论是过滤发生在哪层，决定该改索引、谓词、统计还是 join
形态。

### 15.5. ANALYZE 后仍然偏差

现象是统计已更新，但估计仍不准。可能是采样边界、表达式谓词、跨列相关、参数值分布或
generic plan。

检查 statistics target、extended
statistics、表达式索引或 prepared
statement 的 custom/generic plan
差异。

结论是统计更新不是万能修复。要继续判断模型表达能力和计划缓存边界。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
EXPLAIN node
  -> Plan Rows from plan->plan_rows
  -> Actual Rows from ntuples / nloops
  -> Actual Loops from InstrEndLoop()
  -> Rows Removed from nfiltered1 / nfiltered2
ANALYZE
  -> pg_class reltuples / relpages
  -> pg_statistic per-column stats
  -> extended statistics for correlations
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

### 15.7. estimate / actual 偏差复盘表

这张复盘表用于课后把现场现象、源码断点和最终归因放到同一页。它不是新的主题，只是把本节主问题落到一次可重复的诊断记录。

| 复盘项 | 记录方式 | 判断边界 |
| --- | --- | --- |
| 首次偏差点 | 从叶子到根记录 Plan Rows 与 Actual Rows | 先找偏差首次出现的节点 |
| loops 放大 | 记录 Actual Loops 和累计 rows 估算 | 区分单次慢和重复多 |
| 过滤层次 | 记录 Filter、Join Filter、Index Recheck | 判断 tuple 在哪层被丢弃 |
| 统计证据 | 记录 pg_stats、pg_stats_ext、ANALYZE 时间 | 区分统计陈旧和模型限制 |
| 计划缓存 | 记录 custom/generic plan 与参数值 | 区分当前数据事实和缓存计划选择 |

首次偏差点：从叶子到根记录 Plan Rows 与 Actual
Rows 这一项的判断边界是：先找偏差首次出现的节点 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

loops 放大：记录 Actual Loops 和累计 rows
估算 这一项的判断边界是：区分单次慢和重复多 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

过滤层次：记录 Filter、Join Filter、Index
Recheck 这一项的判断边界是：判断 tuple 在哪层被丢弃
记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

统计证据：记录
pg_stats、pg_stats_ext、ANALYZE 时间
这一项的判断边界是：区分统计陈旧和模型限制 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

计划缓存：记录 custom/generic plan 与参数值
这一项的判断边界是：区分当前数据事实和缓存计划选择 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

### 15.8. 归因顺序的最后确认

做完复盘表后，再按一个固定顺序确认结论：先看统计证据，再看 join
order，再看 loops
和过滤层次，最后才判断执行器路径是否异常。这个顺序能减少把统计问题误报成执行器问题的概率。

如果每一层证据都指向同一个节点，才进入源码级修复讨论。如果证据分散，优先补充可重复实验，例如固定参数、刷新统计、关闭并行、改用
FORMAT JSON 或在 InstrEndLoop()
附近下断点。

这一步不新增课程主题，只是保护诊断闭环：EXPLAIN
现象必须能回到一个明确的状态字段，状态字段必须能回到一个真实源码入口。

## 16. 本节小结

Plan Rows 是估计，Actual Rows/Loops
是执行观测，二者需要按层归因。

诊断顺序应从叶子 scan 开始，再沿 join tree
向上追偏差放大点。

loops、filter count、join type 和
parallel worker 分布共同决定解释。

统计信息陈旧、模型限制、数据相关性和执行器重扫是不同问题。

可迁移规律是：先定位偏差首次出现的层，再决定该改统计、计划、SQL
还是运行环境。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

后续课程可以继续把 Buffers、I/O timing、wait
event 和 profiler 栈接入同一条诊断链。
