# PostgreSQL 并行查询 cleanup 与指标汇总顺序

## 课程定位

前置知识：理解
ParallelExecutorInfo、tuple
queue、worker finish、BufferUsage
/ WalUsage、NodeInstrumentation 和
EXPLAIN 打印时机。

本节唯一主问题：

```text
ExecParallelFinish() / ExecParallelCleanup() 为什么要先处理 tuple queue 和 worker finish，再汇总 instrumentation、buffer、WAL、JIT 和内存指标？
```

核心矛盾：leader 需要尽快告诉 worker 不再需要
tuple，并等待 worker 完成；但 EXPLAIN
又必须在 DSM 销毁前把 worker 指标完整复制出来。

学完后应能判断：能解释并行查询 teardown 中
finish、cleanup、retrieve
instrumentation、DestroyParallelContext
的顺序，以及顺序错了会丢哪些诊断信息。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节把 parallel-aware 节点自己的共享状态和通用
DSM 布局区分开。

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

下一节会把视角转向慢 SQL 诊断：从 EXPLAIN
节点名定位具体 Exec 源码入口。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Finish 阶段停止 tuple 通信并等待 worker 结束，累加 query 级 BufferUsage/WalUsage；Cleanup 阶段再从仍然存在的 DSM 复制节点和 JIT instrumentation，释放 DSA、销毁 ParallelContext。
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
| 1 | `src/backend/executor/execParallel.c` | ExecParallelFinish()、ExecParallelCleanup()、ExecParallelRetrieveInstrumentation()、ExecParallelRetrieveJitInstrumentation()。 |
| 2 | `src/backend/executor/nodeGather.c` | ExecShutdownGatherWorkers() 和 ExecShutdownGather() 如何分别调用 finish / cleanup。 |
| 3 | `src/backend/executor/nodeGatherMerge.c` | GatherMerge 的 shutdown 顺序与 Gather 对齐。 |
| 4 | `src/backend/executor/instrument.c` | InstrAccumParallelQuery()、InstrEndParallelQuery()、BufferUsageAdd()、WalUsageAdd()。 |
| 5 | `src/include/executor/execParallel.h` | ParallelExecutorInfo.finished 标记和 leader 持有字段。 |
| 6 | `src/include/executor/instrument.h` | BufferUsage、WalUsage、WorkerNodeInstrumentation。 |
| 7 | `src/backend/commands/explain.c` | ExplainOnePlan() 在 ExecutorEnd 前打印 plan，因此需要 query context 中的 instrumentation 明细。 |
| 8 | `src/backend/access/transam/parallel.c` | WaitForParallelWorkersToFinish() 所在并行 worker 生命周期基础设施。 |

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
| `pei->finished` | ExecParallelFinish() 的幂等标记，避免重复 detach queue 和重复累加 usage。 |
| `pei->tqueue` | worker 写 tuple 的 shm_mq handle 数组，finish 开始时尽快 detach。 |
| `pei->reader` | leader 的 tuple queue reader，等待 worker 前销毁本地 reader。 |
| `pcxt->nworkers_launched` | finish 只等待实际启动的 worker。 |
| `pei->buffer_usage / wal_usage` | worker 写入的 query 级增量数组，worker finish 后才能累加。 |
| `pei->instrumentation` | DSM 中的节点级 per-worker instrumentation，cleanup 阶段复制。 |
| `pei->jit_instrumentation` | worker JIT instrumentation，cleanup 阶段聚合到 estate。 |
| `pei->area / param_exec` | DSA 和序列化 PARAM_EXEC 值，必须在不再被 worker 使用后释放。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`pei->finished`
的关键点是：ExecParallelFinish()
的幂等标记，避免重复 detach queue 和重复累加
usage。 诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pei->tqueue` 的关键点是：worker 写
tuple 的 shm_mq handle 数组，finish
开始时尽快 detach。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pei->reader` 的关键点是：leader 的
tuple queue reader，等待 worker
前销毁本地 reader。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pcxt->nworkers_launched`
的关键点是：finish 只等待实际启动的 worker。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pei->buffer_usage / wal_usage`
的关键点是：worker 写入的 query
级增量数组，worker finish 后才能累加。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pei->instrumentation` 的关键点是：DSM
中的节点级 per-worker
instrumentation，cleanup 阶段复制。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pei->jit_instrumentation`
的关键点是：worker JIT
instrumentation，cleanup 阶段聚合到
estate。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`pei->area / param_exec`
的关键点是：DSA 和序列化 PARAM_EXEC
值，必须在不再被 worker 使用后释放。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

Gather 或 GatherMerge 结束时先调用节点的
worker shutdown helper。

这会进入
`ExecParallelFinish()`，而不是立刻销毁
ParallelContext。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

`ExecParallelFinish()` 先 detach
每个 tuple queue handle。

如果 leader 不再消费 tuple，尽早 detach
可以让仍在运行的 worker 感知不需要继续发送。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

随后销毁 leader 本地 TupleQueueReader
数组。

reader 是 leader local 资源，和 DSM
中的 instrumentation 不是同一类对象。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

leader 调用
`WaitForParallelWorkersToFinish()`
等待 worker 全部结束。

只有 worker 结束后，buffer/WAL arrays
才可被视为完整。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

finish 阶段循环调用
`InstrAccumParallelQuery()`，把每个
worker 的 BufferUsage / WalUsage
加到 leader 全局计数。

这是 query 级 usage 汇总，不是每个节点的
worker 明细复制。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

`pei->finished = true` 后，重复
finish 会成为 no-op。

这保护 Gather shutdown 和 executor
teardown 中的重复调用。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

`ExecParallelCleanup()` 在 DSM
仍存在时读取 `pei->instrumentation`。

它调用
`ExecParallelRetrieveInstrumentation()`，把
worker 节点明细复制到 PlanState query
context。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

同一 cleanup 阶段再处理 JIT
instrumentation。

worker JIT 明细要在 DSM 销毁前复制到
estate。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

释放 DSA 中的 param_exec，并 detach
DSA area。

这要求 worker 已经结束，否则 worker
可能仍访问参数或节点共享状态。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

最后 `DestroyParallelContext()` 销毁
ParallelContext 和
DSM，`pfree(pei)`。

这之后任何指向 DSM 的指针都无效，只能使用复制到本地
context 的指标。

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

Finish 可以早于 Cleanup，以便 EXPLAIN
或调用者在 worker 结束后仍检查 DSM 内容。

Cleanup 是不可逆的资源销毁点，之后
ParallelExecutorInfo 不再可用。

BufferUsage/WalUsage 累加进入
backend 全局计数，生命周期越过并行 DSM。

WorkerNodeInstrumentation 复制进入
query context，生命周期持续到
ExecutorEnd 清理 EState。

JIT worker instrumentation 复制进入
estate 后由 EXPLAIN summary 读取。

DSA area 和 serialized params
的释放必须晚于 worker finish。

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
| 先停通信 | leader 不再需要 tuple 时先 detach queue，避免 worker 继续阻塞发送。 |
| 再等完成 | worker 结束前读取 usage 可能不完整。 |
| 先复制后销毁 | 节点 instrumentation 和 JIT 明细必须在 DestroyParallelContext 前复制出来。 |
| 幂等 finish | finished 标记防止重复累加 BufferUsage/WalUsage。 |
| query context 保存 | EXPLAIN 打印发生在 ExecutorEnd 前，需要指标仍挂在 EState 生命周期内。 |
| 节点 cleanup 分层 | Gather 的 worker shutdown 和 parallel context cleanup 分开，保证中间可观测窗口存在。 |

先停通信 这一层保证的是：leader 不再需要 tuple
时先 detach queue，避免 worker
继续阻塞发送。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

再等完成 这一层保证的是：worker 结束前读取 usage
可能不完整。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

先复制后销毁 这一层保证的是：节点
instrumentation 和 JIT 明细必须在
DestroyParallelContext 前复制出来。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

幂等 finish 这一层保证的是：finished
标记防止重复累加 BufferUsage/WalUsage。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

query context 保存 这一层保证的是：EXPLAIN
打印发生在 ExecutorEnd 前，需要指标仍挂在
EState 生命周期内。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

节点 cleanup 分层 这一层保证的是：Gather 的
worker shutdown 和 parallel
context cleanup 分开，保证中间可观测窗口存在。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 过早 DestroyParallelContext | worker instrumentation 还没 retrieve，EXPLAIN 只能看到总计划或缺失 worker 明细。 |
| 等待 worker 前读取 usage | BufferUsage/WalUsage 可能少算。 |
| 重复调用 finish 没有保护 | worker usage 会重复加到 leader 全局计数。 |
| 只 cleanup 不 finish | worker 可能仍在运行或 queue 未 detach，资源释放顺序错误。 |
| 把 reader 当 DSM 资源 | reader 是 leader 本地对象，销毁它不等于 worker instrumentation 已取回。 |
| 在 ExecutorEnd 后打印 EXPLAIN | query context 和 PlanState instrumentation 可能已经释放。 |

场景“过早
DestroyParallelContext”的处理思路是：worker
instrumentation 还没
retrieve，EXPLAIN 只能看到总计划或缺失
worker 明细。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“等待 worker 前读取
usage”的处理思路是：BufferUsage/WalUsage
可能少算。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“重复调用 finish
没有保护”的处理思路是：worker usage 会重复加到
leader 全局计数。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“只 cleanup 不
finish”的处理思路是：worker 可能仍在运行或
queue 未 detach，资源释放顺序错误。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 reader 当 DSM
资源”的处理思路是：reader 是 leader
本地对象，销毁它不等于 worker
instrumentation 已取回。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“在 ExecutorEnd 后打印
EXPLAIN”的处理思路是：query context 和
PlanState instrumentation
可能已经释放。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

finish 等待 worker 是并行查询尾部延迟的集中点，慢
worker 会拖住整体结束。

usage 累加成本随实际启动 worker 数增长。

instrumentation retrieve 成本随
plan node 数乘 worker 数增长。

复制 worker_instrument 会消耗 query
context 内存，但换来 DSM 销毁后的稳定输出。

JIT instrumentation 聚合只在启用 JIT
时出现，不能无条件归因于并行基础设施。

过度详细的 EXPLAIN 选项会把 cleanup
后的格式化成本显著放大。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

断在 `ExecParallelFinish()` 开头，打印
`pei->finished`、`pcxt->nworkers_launched`。

观察 tqueue detach 后 worker
是否更快结束。

断在
`InstrAccumParallelQuery()`，确认每个
worker usage 只累加一次。

断在 `ExecParallelCleanup()`，确认
instrumentation retrieve 发生在
DestroyParallelContext 前。

在 EXPLAIN 输出中查看 worker
明细是否存在，若缺失回查 cleanup 是否执行。

用 wait event 和 gdb 区分 leader 等
worker finish 与 leader 正在格式化
EXPLAIN 输出。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. finish 幂等验证

在 gdb 中手动观察 Gather shutdown
可能多次进入

打印 `pei->finished`

确认第二次 ExecParallelFinish 直接返回

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. usage 汇总断点

断在 `InstrAccumParallelQuery`

执行并行查询

记录调用次数是否等于 nworkers_launched

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. cleanup 前后指针

断在 `ExecParallelCleanup`

打印 `pei->instrumentation` 和
`planstate->worker_instrument`

继续到 DestroyParallelContext
后只使用本地复制明细

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 尾部 worker 影响

构造倾斜并行任务

观察 leader 在 finish 等待最长 worker

结合 per-worker rows 判断尾部来源

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. EXPLAIN 时机

断在 `ExplainOnePlan` 中
ExecutorRun、ExecutorFinish、ExplainPrintPlan、ExecutorEnd

确认打印计划发生在 ExecutorEnd 前

解释为什么 instrumentation 仍可读

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

1. 为什么 finish 和 cleanup
要分成两个函数，而不是一个 teardown 函数？

2. BufferUsage/WalUsage 为什么在
finish 阶段累加，节点 instrumentation
为什么在 cleanup 阶段 retrieve？

3. 如果 worker ERROR，leader
如何避免继续读取半完整指标？

4. 为什么 EXPLAIN 要在 ExecutorEnd
前打印 plan？

5. 并行查询尾部延迟如何从 cleanup 顺序中体现出来？

6. 哪些指标在 DSM 销毁后仍可读，哪些不能？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：并行 teardown 顺序错读

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. EXPLAIN worker 明细缺失

现象是 worker 启动并执行了，但打印计划时没有
worker 明细。重点检查
ExecParallelCleanup() 是否在
DestroyParallelContext() 前
retrieve instrumentation。

断在 ExecParallelCleanup() 开头和
DestroyParallelContext() 前后，打印
pei->instrumentation 和
planstate->worker_instrument。

结论是 DSM 销毁前必须复制指标，销毁后只能读取 query
context 中的副本。

### 15.2. BufferUsage 被重复累加

现象是并行查询后全局 buffer/WAL 计数异常偏大。源码上
ExecParallelFinish() 用
pei->finished 做幂等保护。

断在
ExecParallelFinish()，确认第二次进入时直接返回，不再循环
InstrAccumParallelQuery()。

结论是 finish 可重复进入但不能重复计账。任何自定义
teardown 都要保留幂等边界。

### 15.3. leader 长时间卡在结束阶段

现象是查询主体已无输出，但 leader 仍等待。源码上
WaitForParallelWorkersToFinish()
必须等所有 worker 结束后才能读完整 usage。

断在 ExecParallelFinish() 的等待前后，查看
nworkers_launched 和 worker 状态。

结论是尾部 worker 会决定并行查询收尾延迟。要回看
per-worker rows、wait event
和下层节点。

### 15.4. 提前 detach tuple queue 后 worker 报错

现象是 leader 不再需要 tuple，worker
可能在发送路径感知 receiver 消失。源码上 finish
先 detach tqueue，是有意通知生产者停止。

观察 tqueue
detach、TupleQueueReader 销毁和
worker finish 的顺序。

结论是这不是指标丢失，而是停止生产者的 cleanup 协议。

### 15.5. JIT worker summary 不完整

现象是普通节点 worker 明细存在，但 JIT worker
summary 缺失。源码上 JIT
instrumentation 有单独
SharedJitInstrumentation 和
retrieve 函数。

断在
ExecParallelRetrieveJitInstrumentation()，打印
shared_jit->num_workers 和
estate->es_jit_worker_instr。

结论是 JIT 指标与节点 instrumentation
平行汇总，不能只看一个 retrieve 分支。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ExecShutdownGatherWorkers()
  -> ExecParallelFinish()
     -> detach tuple queues
     -> destroy readers
     -> wait workers
     -> InstrAccumParallelQuery()
ExecShutdownGather()
  -> ExecParallelCleanup()
     -> retrieve node instrumentation
     -> retrieve JIT instrumentation
     -> dsa_detach()
     -> DestroyParallelContext()
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

### 15.7. 并行 cleanup 复盘表

这张复盘表用于课后把现场现象、源码断点和最终归因放到同一页。它不是新的主题，只是把本节主问题落到一次可重复的诊断记录。

| 复盘项 | 记录方式 | 判断边界 |
| --- | --- | --- |
| finish 幂等 | 记录 pei->finished 的进入前后值 | 防止 usage 重复累加 |
| queue 关闭 | 记录 tqueue detach 和 reader destroy 顺序 | 确认先停止 tuple 生产/消费 |
| worker 完成 | 记录 WaitForParallelWorkersToFinish 前后的 worker 状态 | 确认 usage 已完整 |
| 指标复制 | 记录 worker_instrument 与 worker_jit_instrument 创建点 | 确认 DSM 销毁前已经复制 |
| 资源销毁 | 记录 dsa_detach 和 DestroyParallelContext | 确认不再持有共享指针 |

finish 幂等：记录 pei->finished 的进入前后值
这一项的判断边界是：防止 usage 重复累加 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

queue 关闭：记录 tqueue detach 和 reader
destroy 顺序 这一项的判断边界是：确认先停止 tuple
生产/消费 记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

worker 完成：记录
WaitForParallelWorkersToFinish 前后的
worker 状态 这一项的判断边界是：确认 usage 已完整
记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

指标复制：记录 worker_instrument 与
worker_jit_instrument 创建点
这一项的判断边界是：确认 DSM 销毁前已经复制 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

资源销毁：记录 dsa_detach 和
DestroyParallelContext
这一项的判断边界是：确认不再持有共享指针 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

## 16. 本节小结

并行 cleanup 的核心顺序是停 tuple queue、等
worker、累加 query usage、复制节点指标、销毁
DSM。

Finish 和 Cleanup 分离给 EXPLAIN
留出了读取共享指标的窗口。

Buffer/WAL usage 与 node
instrumentation 是不同层次的汇总。

幂等 finish 防止重复累加，query context
复制防止 DSM 销毁后指标丢失。

可迁移规律是：先关闭生产者，再读取完整账本，最后销毁账本载体。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会把视角转向慢 SQL 诊断：从 EXPLAIN
节点名定位具体 Exec 源码入口。
