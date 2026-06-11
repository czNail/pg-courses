# PostgreSQL 从 EXPLAIN 节点名定位 Exec 源码入口

## 课程定位

前置知识：熟悉 EXPLAIN 输出、Plan /
PlanState、ExecInitNode、ExecProcNode、ExecEndNode
以及常见执行节点。

本节唯一主问题：

```text
如何把 EXPLAIN 中的 Seq Scan、Hash Join、Aggregate、Gather 等节点名映射到 ExecInit*、Exec*、ExecEnd* 源码入口？
```

核心矛盾：EXPLAIN
输出为人类阅读做了命名、简化和历史兼容；源码执行却按
nodeTag、PlanState
类型和函数指针分派，节点名不能机械替换成文件名。

学完后应能判断：能从一个 EXPLAIN 节点名回到
ExplainNode() 的命名分支，再回到
execProcnode.c 的 init / run /
end 分派和具体 node*.c 文件。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

前面几节把执行器观测指标如何产生、汇总和输出放到了源码路径里。

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

下一节会在定位源码入口后，继续处理 estimate /
actual 偏差该归因到哪一层。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
先在 ExplainNode() 找节点名如何由 nodeTag(plan) 生成，再到 ExecInitNode() 找 Plan 到 PlanState 的构造函数，最后看 PlanState->ExecProcNode 指向的运行函数和 ExecEndNode 的清理分支。
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
| 1 | `src/backend/commands/explain.c` | ExplainNode() 中 nodeTag(plan) 到显示节点名、cost、actual、filter、worker 输出的映射。 |
| 2 | `src/backend/executor/execProcnode.c` | ExecInitNode()、ExecProcNodeFirst()、ExecSetExecProcNode()、ExecEndNode() 的分派入口。 |
| 3 | `src/include/nodes/plannodes.h` | SeqScan、HashJoin、Agg、Gather 等 Plan node 结构。 |
| 4 | `src/include/nodes/execnodes.h` | SeqScanState、HashJoinState、AggState、GatherState 等 PlanState 结构。 |
| 5 | `src/backend/executor/nodeSeqscan.c` | ExecInitSeqScan()、ExecSeqScan()、ExecEndSeqScan()。 |
| 6 | `src/backend/executor/nodeHashjoin.c` | ExecInitHashJoin()、ExecHashJoin()、ExecParallelHashJoin()、ExecEndHashJoin()。 |
| 7 | `src/backend/executor/nodeAgg.c` | ExecInitAgg()、ExecAgg()、ExecEndAgg()。 |
| 8 | `src/backend/executor/nodeGather.c` | ExecInitGather()、ExecGather()、ExecEndGather()。 |
| 9 | `src/backend/executor/nodeSort.c` | ExecInitSort()、ExecSort()、ExecEndSort()。 |
| 10 | `src/backend/executor/nodeModifyTable.c` | ExecInitModifyTable()、ExecModifyTable()、ExecEndModifyTable()。 |

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
| `pname / sname` | ExplainNode 中的显示名，TEXT 和非 TEXT 格式可能使用不同字符串。 |
| `nodeTag(plan)` | 计划节点类型的源码分派键，定位显示名和执行函数的第一入口。 |
| `PlanState->ExecProcNode` | 运行期函数指针，真正每次拉取 tuple 时调用。 |
| `ExecProcNodeReal` | instrumentation wrapper 存放的真实节点函数指针。 |
| `plan_node_id` | EXPLAIN、instrumentation 和并行 worker 对齐逻辑节点的稳定编号。 |
| `outerPlanState / innerPlanState` | 从父节点继续追到子节点源码入口的关系。 |
| `ps_ProjInfo / qual` | 很多节点输出前还会执行过滤和投影，不能只看 access method。 |
| `node-specific state` | HashJoinState、AggState、SortState 等决定同名节点内部策略和状态。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`pname / sname`
的关键点是：ExplainNode 中的显示名，TEXT 和非
TEXT 格式可能使用不同字符串。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`nodeTag(plan)`
的关键点是：计划节点类型的源码分派键，定位显示名和执行函数的第一入口。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`PlanState->ExecProcNode`
的关键点是：运行期函数指针，真正每次拉取 tuple 时调用。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ExecProcNodeReal`
的关键点是：instrumentation wrapper
存放的真实节点函数指针。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`plan_node_id`
的关键点是：EXPLAIN、instrumentation
和并行 worker 对齐逻辑节点的稳定编号。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`outerPlanState /
innerPlanState`
的关键点是：从父节点继续追到子节点源码入口的关系。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ps_ProjInfo / qual`
的关键点是：很多节点输出前还会执行过滤和投影，不能只看
access method。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`node-specific state`
的关键点是：HashJoinState、AggState、SortState
等决定同名节点内部策略和状态。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

拿到 EXPLAIN 输出后，先记录节点名、关系名、Join
Type、Strategy、Partial Mode 和
Parallel Aware。

这些字段共同决定要读哪个分支，而不是只读第一行节点名。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

到 `ExplainNode()` 中搜索显示名，例如 `Seq
Scan`、`Hash Join`、`Aggregate`。

这里能看到显示名来自 `nodeTag(plan)`，以及
Agg strategy、join type 等额外修饰。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

用同一个 Plan node type 到
`ExecInitNode()` 中找 init 分派。

例如 T_SeqScan 对应
ExecInitSeqScan，T_HashJoin 对应
ExecInitHashJoin。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

打开对应 `node*.c` 文件，看 init 函数如何创建
PlanState、slot、表达式、子节点和函数指针。

诊断时 init 阶段常解释为什么
instrumentation、projection 或
child state 存在。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

在运行函数中找 `ExecProcNode` 真实入口。

一些节点会通过 `ExecSetExecProcNode()`
切换到特殊路径，如 parallel hash join。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

如果 EXPLAIN 显示 Parallel 或
Async，回到 ExplainNode 的
parallel_aware / async_capable
输出，再读对应节点初始化。

Parallel Aware 不是简单文件名前缀，而是
plan->parallel_aware
标志和节点实现共同决定。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

遇到 `Aggregate` 要继续看 Strategy /
Partial Mode。

Plain、Sorted、Hashed、Mixed、Partial、Finalize
都会影响 nodeAgg.c 中的状态路径。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

遇到 `ModifyTable` 要看 Operation 和
RETURNING / trigger / partition
routing。

显示名可能是 Insert / Update / Delete
/ Merge，但源码主入口仍在
nodeModifyTable.c。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

遇到 `Custom Scan` 或 `Foreign
Scan` 要转向扩展 / FDW 回调。

这时 core executor 只提供协议，实际实现可能不在
core 源码文件中。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

最后到 `ExecEndNode()` 或节点 End 函数读
cleanup。

很多慢查询问题来自运行后清理、spill 文件释放或
trigger drain，不只来自 ExecProcNode
hot path。

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

Plan node 来自 planner，ExplainNode
读取的是计划描述和执行后 instrumentation。

PlanState 在 ExecutorStart /
ExecInitNode 阶段构造，运行期由
ExecProcNode 推进。

Instrumentation wrapper 可能让
PlanState->ExecProcNode 先进入
ExecProcNodeInstr，再调用真实函数。

节点私有状态挂在 EState query context
或节点私有 context，End 函数负责清理外部资源。

EXPLAIN 打印发生在 ExecutorEnd
前，因此能读取 PlanState 和
instrumentation。

并行 worker 的 PlanState
是本地重建，EXPLAIN 中 worker 明细由
leader 复制后输出。

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
| 显示名不是入口函数 | `Hash Join` 显示名帮助定位，但真实入口要看 ExecInitHashJoin 和 ExecHashJoin / ExecParallelHashJoin。 |
| 策略字段必须一起读 | Aggregate、SetOp、Join、ModifyTable 等节点需要额外字段才能定位内部路径。 |
| 函数指针可变 | 节点可以在 init 或运行中设置 ExecProcNode 指针，不能只搜索函数名。 |
| 子节点关系 | ExplainNode 的缩进对应 PlanState 子树，但 outer/inner 的语义要看节点类型。 |
| 扩展边界 | CustomScan 和 ForeignScan 需要读回调表，core 文件只给协议。 |
| cleanup 也是源码入口 | ExecEnd* 和 ExecShutdown* 经常解释资源释放和指标汇总。 |

显示名不是入口函数 这一层保证的是：`Hash Join`
显示名帮助定位，但真实入口要看 ExecInitHashJoin
和 ExecHashJoin /
ExecParallelHashJoin。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

策略字段必须一起读
这一层保证的是：Aggregate、SetOp、Join、ModifyTable
等节点需要额外字段才能定位内部路径。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

函数指针可变 这一层保证的是：节点可以在 init 或运行中设置
ExecProcNode 指针，不能只搜索函数名。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

子节点关系 这一层保证的是：ExplainNode 的缩进对应
PlanState 子树，但 outer/inner
的语义要看节点类型。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

扩展边界 这一层保证的是：CustomScan 和
ForeignScan 需要读回调表，core 文件只给协议。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

cleanup 也是源码入口 这一层保证的是：ExecEnd*
和 ExecShutdown* 经常解释资源释放和指标汇总。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 直接把节点名 lower 后找文件 | 例如 Materialize 对应 nodeMaterial.c，Aggregate 对应 nodeAgg.c，命名并不总是机械。 |
| 忽略 Agg strategy | HashAggregate 和 GroupAggregate 都显示 Aggregate 相关，但状态和成本边界不同。 |
| 只读 Exec 函数不读 Init | slot、qual、projection、instrumentation、parallel state 多在 init 建立。 |
| 只读父节点 | 父节点时间可能包含子节点调用，必须沿 outer/inner 子树继续追。 |
| 把 EXPLAIN text 当稳定 API | 机器工具应使用 FORMAT JSON，并用 Node Type / Strategy 等字段定位。 |
| 忘记 End 路径 | Sort、Hash、Gather 等节点的 cleanup 和指标 retrieve 可能是诊断关键。 |

场景“直接把节点名 lower 后找文件”的处理思路是：例如
Materialize 对应
nodeMaterial.c，Aggregate 对应
nodeAgg.c，命名并不总是机械。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忽略 Agg
strategy”的处理思路是：HashAggregate 和
GroupAggregate 都显示 Aggregate
相关，但状态和成本边界不同。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“只读 Exec 函数不读
Init”的处理思路是：slot、qual、projection、instrumentation、parallel
state 多在 init 建立。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“只读父节点”的处理思路是：父节点时间可能包含子节点调用，必须沿
outer/inner 子树继续追。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 EXPLAIN text 当稳定
API”的处理思路是：机器工具应使用 FORMAT
JSON，并用 Node Type / Strategy
等字段定位。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忘记 End
路径”的处理思路是：Sort、Hash、Gather 等节点的
cleanup 和指标 retrieve 可能是诊断关键。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

从 EXPLAIN 回源码的成本主要是避免错读：多看一个
strategy 字段比读错文件便宜。

使用 rg 搜索显示名时要回到
ExplainNode，而不是全仓库泛搜导致路径分散。

并行和 JIT 会插入 wrapper 和 worker
明细，源码入口比非并行路径多一层。

Custom/Foreign 节点需要跨 core
和扩展源码，定位成本取决于扩展是否可读。

节点总时间可能包含子节点时间，源码阅读要区分 inclusive
和局部工作。

EXPLAIN
输出越详细，能减少猜测，但日志和格式化成本也更高。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

用 `EXPLAIN (ANALYZE, VERBOSE,
FORMAT JSON)` 获取 Node
Type、Strategy、Partial Mode。

在源码中从 `ExplainNode()` 的 switch
确认显示名。

再到 `ExecInitNode()` 的 switch 找
init 函数。

在 init 函数中找 `ps.ExecProcNode =
...` 或 `ExecSetExecProcNode()`。

运行时用 gdb 打印 `node->ExecProcNode`
和 `node->ExecProcNodeReal`。

结束时断在 `ExecEndNode()` 和具体
`ExecEnd*`，确认资源释放路径。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. Seq Scan 定位

执行 `EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM pg_class`

在 explain.c 找 `Seq Scan` 分支

跳到 execProcnode.c 的 T_SeqScan 和
nodeSeqscan.c

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. Hash Join 定位

构造 join 查询产生 Hash Join

记录 Join Type 和 Hash 子节点

分别读 nodeHashjoin.c 和 nodeHash.c

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. Aggregate 策略定位

分别构造 GroupAggregate 和
HashAggregate

观察 Strategy 字段

对照 nodeAgg.c 中不同 strategy 初始化状态

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. Gather 定位

执行并行查询产生 Gather

从 ExplainNode 的 T_Gather 到
ExecInitGather / ExecGather

继续读 ExecShutdownGather

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 函数指针验证

gdb 断在 ExecProcNodeFirst

打印 planstate->ExecProcNodeReal

确认 instrumentation wrapper 是否介入

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

1. 为什么从 ExplainNode 开始，比直接猜
node*.c 文件更可靠？

2. 哪些 EXPLAIN 字段决定源码入口，哪些只是显示增强？

3. 为什么 init / run / end 三个入口都要读？

4. Custom Scan 和 Foreign Scan
为什么不能只看 core executor？

5. 如何判断父节点时间是否主要来自子节点？

6. FORMAT JSON 对源码定位工具有什么优势？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：从节点名回到正确源码入口

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. Hash Join 定位到了 nodeHash.c 后仍看不懂

现象是 EXPLAIN 显示 Hash Join，读者直接打开
nodeHash.c，却漏掉 join 状态机。源码上 Hash
Join 的主运行在 nodeHashjoin.c，Hash
子节点在 nodeHash.c。

从 ExplainNode() 的 T_HashJoin
分支回到 ExecInitHashJoin()，再看
outer/inner 子节点中的 Hash。

结论是 EXPLAIN 的相邻节点共同构成执行路径。Hash
Join 和 Hash 节点不能混成一个入口。

### 15.2. Aggregate 节点读错策略

现象是 EXPLAIN 显示 Aggregate，但实际是
HashAggregate 或
GroupAggregate。ExplainNode() 会根据
aggstrategy 和 aggsplit 改写 pname。

记录 Strategy 和 Partial Mode，再进入
nodeAgg.c 读对应 strategy 初始化和运行路径。

结论是节点名必须和 Strategy 一起解释。只搜索
Aggregate 会丢掉关键分支。

### 15.3. Materialize 被当成普通输出节点

现象是 Materialize
时间高，读者只看父节点。源码入口在
nodeMaterial.c，关键是 tuplestore 和
rescan 需求。

从 ExecInitNode 的 T_Material 分支进入
ExecInitMaterial，再看 ExecMaterial
如何读写 tuplestore。

结论是阻塞或缓存节点的成本常来自状态保存，而不是复杂表达式。

### 15.4. Foreign Scan 找不到执行逻辑

现象是 core
源码中只有协议，真正访问逻辑不明显。ForeignScan 通过
FDW routine 调用扩展实现。

在 ExplainNode() 确认 Foreign Scan
后，到 nodeForeignscan.c 和 FDW
handler 回调表继续追。

结论是 core executor 提供节点协议，扩展或 FDW
才提供具体访问方法。

### 15.5. 父节点 total time 被误读为自身 CPU

现象是 Nested Loop 或 Gather total
time
很高，直接归因父节点。执行器模型是父节点反复调用子节点，时间通常包含子调用。

沿 ExplainNode 的子节点输出和
ExecProcNode
调用关系，计算子节点是否解释大部分时间。

结论是从节点名回源码后还要区分 inclusive
时间和局部工作。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ExplainNode()
  -> nodeTag(plan) chooses display name
ExecInitNode()
  -> Plan node becomes PlanState
node*.c ExecInit*()
  -> set ExecProcNode function pointer
ExecProcNode()
  -> run one tuple step
ExecEndNode()
  -> node-specific cleanup
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

### 15.7. 节点名定位复盘表

这张复盘表用于课后把现场现象、源码断点和最终归因放到同一页。它不是新的主题，只是把本节主问题落到一次可重复的诊断记录。

| 复盘项 | 记录方式 | 判断边界 |
| --- | --- | --- |
| 显示名 | 记录 ExplainNode 中 pname、sname、Strategy | 先还原 nodeTag |
| 初始化入口 | 记录 ExecInitNode 分支和 ExecInit* 函数 | 确认 Plan 到 PlanState 的状态化 |
| 运行入口 | 记录 ExecProcNodeReal 或节点 Exec* 函数 | 确认每 tuple 推进路径 |
| 清理入口 | 记录 ExecEndNode 和 ExecEnd* 函数 | 确认资源释放和指标收尾 |
| 子节点关系 | 记录 outer/inner/subplan 关系 | 区分父节点局部成本和子节点成本 |

显示名：记录 ExplainNode 中
pname、sname、Strategy 这一项的判断边界是：先还原
nodeTag 记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

初始化入口：记录 ExecInitNode 分支和
ExecInit* 函数 这一项的判断边界是：确认 Plan 到
PlanState 的状态化 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

运行入口：记录 ExecProcNodeReal 或节点 Exec*
函数 这一项的判断边界是：确认每 tuple 推进路径 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

清理入口：记录 ExecEndNode 和 ExecEnd* 函数
这一项的判断边界是：确认资源释放和指标收尾 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

子节点关系：记录 outer/inner/subplan 关系
这一项的判断边界是：区分父节点局部成本和子节点成本 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

## 16. 本节小结

EXPLAIN 节点名是定位线索，不是源码入口本身。

稳定路径是 ExplainNode -> ExecInitNode
-> 节点 init/run/end 函数。

Strategy、Join Type、Partial
Mode、Parallel Aware 决定内部路径。

运行函数指针和 cleanup 路径同样重要。

可迁移规律是：先把显示语义还原成 nodeTag，再沿
runtime 分派读源码。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会在定位源码入口后，继续处理 estimate /
actual 偏差该归因到哪一层。
