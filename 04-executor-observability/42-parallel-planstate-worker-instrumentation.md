# PostgreSQL 并行 PlanState 与 worker instrumentation 布局

## 课程定位

前置知识：理解 PlanState
tree、Instrumentation、EXPLAIN
ANALYZE，以及 parallel worker 通过
DSM / shm_toc 接收 leader 状态的基本过程。

本节唯一主问题：

```text
ExecInitParallelPlan() 如何为并行查询准备计划、参数和 instrumentation 共享布局，哪些内容属于执行器观测边界而不是 DSM / worker bootstrap 基础设施？
```

核心矛盾：leader 和 worker 要分别拥有自己的
backend-local PlanState 指针；但
EXPLAIN 又要按同一棵逻辑计划汇总每个 worker 的
rows、loops、buffer、WAL 和 JIT 统计。

学完后应能判断：能解释为什么并行 instrumentation
通过 plan_node_id 和 DSM 数组对齐，而不是把
leader 的 PlanState 指针直接交给
worker。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节把 hook 放回了正确层次，本节进入并行查询的观测边界。

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

下一节会把这些共享布局落到 Gather /
GatherMerge 的 tuple routing 和
leader participation 上。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
leader 先序列化 plan 和参数，估算 PlanState 节点数，为每个 plan_node_id 和每个 worker 分配 NodeInstrumentation 数组；worker 本地执行后按 ParallelWorkerNumber 写回，leader cleanup 时再聚合到 PlanState。
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
| 1 | `src/backend/executor/execParallel.c` | ExecInitParallelPlan()、ExecParallelEstimate()、ExecParallelInitializeDSM()、ParallelQueryMain()、ExecParallelReportInstrumentation()、ExecParallelRetrieveInstrumentation()。 |
| 2 | `src/include/executor/execParallel.h` | ParallelExecutorInfo 的 leader 侧句柄，保存 ParallelContext、BufferUsage、WalUsage、instrumentation、DSA。 |
| 3 | `src/include/executor/instrument.h` | Instrumentation、NodeInstrumentation、WorkerNodeInstrumentation 的字段语义。 |
| 4 | `src/backend/executor/instrument.c` | InstrStartNode()、InstrStopNode()、InstrEndLoop()、InstrAggNode()、parallel query usage 累加。 |
| 5 | `src/backend/executor/execProcnode.c` | PlanState 创建时根据 es_instrument 分配节点 instrumentation。 |
| 6 | `src/include/nodes/execnodes.h` | PlanState 中 instrument、worker_instrument、worker_jit_instrument 的位置。 |
| 7 | `src/backend/access/transam/parallel.c` | ParallelWorkerMain() 负责 worker bootstrap，说明本节 executor 状态建立在更底层并行基础设施之上。 |
| 8 | `src/backend/commands/explain.c` | ExplainNode() 读取 worker_instrument 并输出 per-worker actual rows / loops。 |

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
| `ParallelExecutorInfo` | leader 侧持有的 executor parallel 句柄，记录 pcxt、planstate、tuple queue、buffer/WAL usage、shared instrumentation 和 DSA。 |
| `SharedExecutorInstrumentation` | DSM 中的共享 instrumentation 头部，包含 instrument_options、num_workers、num_plan_nodes 和 plan_node_id 数组。 |
| `NodeInstrumentation array` | 按 plan node 下标乘 worker 数展开的二维数组，worker 用 ParallelWorkerNumber 写自己的槽。 |
| `plan_node_id` | leader 和 worker 之间匹配逻辑 PlanState 的稳定键，不是内存地址。 |
| `WorkerNodeInstrumentation` | leader retrieval 后挂回 PlanState 的 per-worker 明细，供 EXPLAIN 输出。 |
| `BufferUsage / WalUsage arrays` | 每个 worker 一个槽，统计 executor run 期间的 buffer 和 WAL 增量。 |
| `SharedJitInstrumentation` | 可选 JIT per-worker 明细，只有启用 JIT instrumentation 时分配。 |
| `DSA area` | 为并行执行中的参数和节点共享状态提供动态共享内存，不等同于 instrumentation 本身。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`ParallelExecutorInfo`
的关键点是：leader 侧持有的 executor
parallel 句柄，记录
pcxt、planstate、tuple
queue、buffer/WAL usage、shared
instrumentation 和 DSA。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`SharedExecutorInstrumentation`
的关键点是：DSM 中的共享 instrumentation
头部，包含
instrument_options、num_workers、num_plan_nodes
和 plan_node_id 数组。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`NodeInstrumentation array`
的关键点是：按 plan node 下标乘 worker
数展开的二维数组，worker 用
ParallelWorkerNumber 写自己的槽。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`plan_node_id` 的关键点是：leader 和
worker 之间匹配逻辑 PlanState
的稳定键，不是内存地址。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`WorkerNodeInstrumentation`
的关键点是：leader retrieval 后挂回
PlanState 的 per-worker 明细，供
EXPLAIN 输出。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`BufferUsage / WalUsage arrays`
的关键点是：每个 worker 一个槽，统计 executor
run 期间的 buffer 和 WAL 增量。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`SharedJitInstrumentation`
的关键点是：可选 JIT per-worker 明细，只有启用
JIT instrumentation 时分配。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`DSA area`
的关键点是：为并行执行中的参数和节点共享状态提供动态共享内存，不等同于
instrumentation 本身。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

Gather 或 GatherMerge 第一次执行时调用
`ExecInitParallelPlan()`。

这一步不是 worker bootstrap 本身，而是
executor 为并行子计划准备 DSM 布局。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

`ExecSetParamPlanMulti()`
先强制计算要传给 worker 的 initplan 输出。

否则 worker 看到的 PARAM_EXEC
可能没有稳定值。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

leader 用 `ExecSerializePlan()` 把
plan 序列化，再用
`CreateParallelContext()` 建立
ParallelContext。

这里传入的入口是 `ParallelQueryMain`，说明
worker 将在 executor 层重建
QueryDesc。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

`ExecParallelEstimate()` 遍历
PlanState
tree，给并行感知节点估算共享空间，同时统计节点数量。

节点数不是为了遍历好看，而是为了后面构造
plan_node_id 到 worker
instrumentation 的布局。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

如果 estate->es_instrument
非零，leader 为
`SharedExecutorInstrumentation`
估算并分配 DSM 空间。

数组的前半部分是 plan_node_id，后半部分是
`num_plan_nodes * nworkers` 个
NodeInstrumentation。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

`ExecParallelInitializeDSM()`
再次遍历 PlanState tree，把每个
plan_node_id 写入共享数组，并让节点初始化自己的
DSM 状态。

同一次遍历也把 instrumentation
槽和节点逻辑身份对齐。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

worker 进入 `ParallelQueryMain()`
后，从 TOC 查找 instrumentation，并用相同
instrument_options 创建本地
QueryDesc。

worker 的 PlanState 是本地重新初始化的，不共享
leader 指针。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

worker 执行完成后调用
`ExecParallelReportInstrumentation()`。

它先 `InstrEndLoop()`，再按
plan_node_id 查槽，并把本地
NodeInstrumentation 聚合到自己的
worker 槽。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

leader 在 `ExecParallelCleanup()`
中调用
`ExecParallelRetrieveInstrumentation()`。

它把 worker 槽累加到 leader 的
planstate->instrument，并把明细复制到
es_query_cxt 下的
worker_instrument。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

EXPLAIN 输出阶段再读取
`planstate->worker_instrument`。

所以 per-worker 信息的生命周期是 query
context，而不是 DSM 永久可读。

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

ParallelExecutorInfo 由 leader 在
ExecInitParallelPlan() 中
palloc，最终由 ExecParallelCleanup()
pfree。

SharedExecutorInstrumentation 位于
parallel query DSM，随
ParallelContext 销毁而失效。

worker 本地 PlanState 和
instrumentation 随 worker 的
QueryDesc / ExecutorEnd 生命周期释放。

leader 复制出来的
WorkerNodeInstrumentation 挂到
es_query_cxt，供 ExecutorEnd 前的
EXPLAIN 打印。

BufferUsage / WalUsage arrays
必须等 worker 完成后才能累加，否则会读取到未完成统计。

JIT worker instrumentation 和节点
instrumentation 一样需要先从 DSM 汇总，再让
EXPLAIN 读取。

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
| 地址边界 | PlanState 指针是 backend-local，跨进程只能用 plan_node_id、序列化 plan 和 DSM offset 对齐。 |
| 遍历一致性 | estimate 和 initialize 阶段的 PlanState 节点数必须一致，源码不一致时会 ERROR。 |
| worker 槽隔离 | 每个 worker 用 ParallelWorkerNumber 写自己的 NodeInstrumentation，避免多个 worker 写同一槽。 |
| loop 完结 | worker 报告前必须 InstrEndLoop()，否则 running 状态里的 counter 没有进入 totals。 |
| 聚合顺序 | leader 先等 worker 结束，再聚合 buffer/WAL，再在 cleanup 阶段取 instrumentation。 |
| 输出生命周期 | EXPLAIN 读取的是复制到 query context 的 per-worker 明细，不应该在 DSM 销毁后直接指向共享内存。 |

地址边界 这一层保证的是：PlanState 指针是
backend-local，跨进程只能用
plan_node_id、序列化 plan 和 DSM
offset 对齐。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

遍历一致性 这一层保证的是：estimate 和
initialize 阶段的 PlanState
节点数必须一致，源码不一致时会 ERROR。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

worker 槽隔离 这一层保证的是：每个 worker 用
ParallelWorkerNumber 写自己的
NodeInstrumentation，避免多个 worker
写同一槽。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

loop 完结 这一层保证的是：worker 报告前必须
InstrEndLoop()，否则 running 状态里的
counter 没有进入 totals。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

聚合顺序 这一层保证的是：leader 先等 worker
结束，再聚合 buffer/WAL，再在 cleanup 阶段取
instrumentation。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

输出生命周期 这一层保证的是：EXPLAIN 读取的是复制到
query context 的 per-worker
明细，不应该在 DSM 销毁后直接指向共享内存。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 把 PlanState 指针放进 DSM | worker 进程地址空间不同，指针值没有语义。 |
| estimate 和 initialize 遍历条件不一致 | plan_node_id 数组和 NodeInstrumentation 数组下标错位，源码会检测节点数量不一致。 |
| worker 未调用 InstrEndLoop | actual rows 和 loops 留在当前 cycle，leader 聚合后偏小或缺失。 |
| leader 过早 DestroyParallelContext | EXPLAIN 还没复制 worker 明细，DSM 状态就不可读。 |
| 只看总 instrumentation | worker 间倾斜会被平均值掩盖，尤其在 parallel scan 或 parallel hash 中。 |
| 把 BufferUsage 当节点级 | buffer/WAL arrays 是 worker query 级增量，节点级 buffer 仍在 NodeInstrumentation 内。 |

场景“把 PlanState 指针放进
DSM”的处理思路是：worker
进程地址空间不同，指针值没有语义。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“estimate 和 initialize
遍历条件不一致”的处理思路是：plan_node_id 数组和
NodeInstrumentation
数组下标错位，源码会检测节点数量不一致。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“worker 未调用
InstrEndLoop”的处理思路是：actual rows
和 loops 留在当前 cycle，leader
聚合后偏小或缺失。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“leader 过早
DestroyParallelContext”的处理思路是：EXPLAIN
还没复制 worker 明细，DSM 状态就不可读。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“只看总
instrumentation”的处理思路是：worker
间倾斜会被平均值掩盖，尤其在 parallel scan 或
parallel hash 中。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 BufferUsage
当节点级”的处理思路是：buffer/WAL arrays 是
worker query 级增量，节点级 buffer 仍在
NodeInstrumentation 内。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

instrumentation DSM 大小随 plan
node 数和 worker 数相乘增长。

启用 timing 会让每个 instrumented
ExecProcNode 边界承担计时成本。

plan_node_id 查找当前是线性扫描，超大 plan
tree 会放大 retrieval 和 report 成本。

BufferUsage 和 WalUsage 即使没有
EXPLAIN 也会为并行查询分配，因为 leader
无法知道是否有人关注全局计数。

把 per-worker 明细复制到 es_query_cxt
会增加 EXPLAIN ANALYZE 的内存占用，但换来
DSM cleanup 后仍可格式化输出。

并行节点自己的共享状态和 instrumentation
共享布局是两类成本，不应混为一个 DSM overhead。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

断在 `ExecInitParallelPlan()`，打印
`nworkers`、`estate->es_instrument`
和 `e.nnodes`。

断在
`ExecParallelInitializeDSM()`，观察
`instrumentation->plan_node_id[d->nnodes]`
如何写入。

断在 worker 的
`ParallelQueryMain()`，确认
`instrument_options` 来自共享
instrumentation。

断在
`ExecParallelReportInstrumentation()`，打印
`ParallelWorkerNumber` 和
plan_node_id。

断在 leader 的
`ExecParallelRetrieveInstrumentation()`，观察
`planstate->worker_instrument`
的分配 context。

用 `EXPLAIN (ANALYZE, VERBOSE)`
查看 worker 明细是否出现在对应节点下。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. 构造并行 scan

调高
`max_parallel_workers_per_gather`

对大表执行 `EXPLAIN (ANALYZE,
VERBOSE, BUFFERS)`

观察 Gather 子节点下是否出现 Worker N 明细

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. 关闭 timing 观察扰动

对同一 SQL 分别执行 `EXPLAIN (ANALYZE,
TIMING ON)` 和 `TIMING OFF`

比较节点 total time 和 rows 是否仍能输出

确认 rows/loops 不依赖 timer

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. worker 槽定位

在 gdb 中给
`ExecParallelReportInstrumentation`
下断点

打印 `ParallelWorkerNumber` 和
instrumentation->num_workers

确认每个 worker 写不同槽位

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 计划节点对齐

断在 leader retrieve 阶段

打印
`planstate->plan->plan_node_id`

在 shared plan_node_id 数组中找到相同值

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 聚合顺序验证

断在 `ExecParallelFinish` 和
`ExecParallelCleanup`

确认 buffer/WAL usage 在 Finish 累加

确认节点 worker_instrument 在 Cleanup
复制

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

1. 为什么 plan_node_id 比 PlanState
指针更适合作为 leader-worker 对齐键？

2. 如果 retrieval 阶段改成 hash 查找
plan_node_id，会改变哪些成本和复杂度？

3. 为什么 BufferUsage/WalUsage 的
worker arrays 即使不打印 EXPLAIN
也要准备？

4. 为什么 worker_instrument 要复制到
es_query_cxt 而不是保留 DSM 指针？

5. parallel-aware 节点 DSM 状态和
instrumentation DSM 状态有什么不同？

6. 如何从 worker 明细判断并行效率问题，而不是只看
leader 汇总？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：worker 指标缺失或错位

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. Workers 存在但节点没有 worker 明细

现象是 Gather 显示 Workers Launched
大于零，但某些子节点没有 Worker N actual
rows。先确认 `estate->es_instrument`
是否非零，再看该节点是否有 nloops。

断点放在
ExecParallelRetrieveInstrumentation()，打印
plan_node_id、num_plan_nodes、num_workers
和 planstate->worker_instrument。

结论可能是没有 ANALYZE、hide
workers、生效节点未执行，或节点没有对应的 worker
instrumentation 明细。

### 15.2. worker 明细和总 rows 对不上

现象是主节点 rows 与 Worker N rows
简单相加不一致。需要回到 nloops
归一化口径：EXPLAIN 显示 rows 通常是
ntuples / nloops。

断点读
`src/backend/executor/instrument.c`
的 InstrEndLoop() 和
InstrAggNode()。同时打印
ntuples、nloops、startup、total。

结论是必须用 loops 解释 rows。总计口径和
per-worker 明细口径不同，不能用显示值直接相加。

### 15.3. 并行计划节点数不一致

现象是源码报 inconsistent count of
PlanState nodes 或 retrieval 找不到
plan node id。重点检查 estimate 和
initialize 遍历条件是否一致。

沿 ExecParallelEstimate() 和
ExecParallelInitializeDSM() 分别统计
nnodes，并打印每次写入的 plan_node_id。

结论是并行共享布局依赖两次遍历一致。新节点或扩展节点加入并行路径时，必须同步
estimate、initialize 和 worker
attach 逻辑。

### 15.4. worker 写错 instrumentation 槽

现象是某个 worker 指标覆盖另一个 worker，或
Worker Number 输出异常。源码上 worker 用
ParallelWorkerNumber 选择槽位。

断在
ExecParallelReportInstrumentation()，打印
ParallelWorkerNumber、num_workers
和 instrument 指针偏移。

结论是 worker 槽位必须由并行 worker
bootstrap 正确设置。自定义并行入口不能伪造
worker number。

### 15.5. JIT 明细丢失

现象是总 JIT summary 有信息，但
per-worker JIT 明细为空。检查
SharedJitInstrumentation 是否分配，以及
worker 是否把 estate->es_jit->instr
写回。

断在 ParallelQueryMain() 结束前和
ExecParallelRetrieveJitInstrumentation()，确认
shared_jit->num_workers 和
worker_jit_instrument。

结论是 JIT worker 指标只在启用 JIT
instrumentation 且 cleanup
阶段复制后可见。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ExecInitParallelPlan()
  -> ExecParallelEstimate()
  -> allocate SharedExecutorInstrumentation
  -> ExecParallelInitializeDSM()
ParallelQueryMain()
  -> ExecutorStart()
  -> ExecParallelInitializeWorker()
  -> ExecutorRun()
  -> ExecParallelReportInstrumentation()
ExecParallelCleanup()
  -> ExecParallelRetrieveInstrumentation()
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

## 16. 本节小结

并行 instrumentation 的核心不是共享
PlanState，而是用 plan_node_id 对齐逻辑节点。

leader 和 worker 各自有本地
PlanState，DSM 只保存可序列化状态、共享数组和
per-worker 指标。

worker 报告、leader 聚合、EXPLAIN
输出是三个不同生命周期边界。

worker 明细是诊断并行倾斜的入口，总计值只能回答总体成本。

可迁移规律是：跨进程观测必须传稳定身份和聚合数据，不能传本地指针。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会把这些共享布局落到 Gather / GatherMerge
的 tuple routing 和 leader
participation 上。
