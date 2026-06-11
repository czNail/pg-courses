# PostgreSQL Parallel Hash / Parallel Append 共享执行状态观测

## 课程定位

前置知识：理解并行查询
DSM、DSA、ParallelContext、parallel-aware
PlanState，以及 Hash Join、Append
的基本执行策略。

本节唯一主问题：

```text
并行感知节点如何在执行器层共享进度、批次、分配和完成状态，观测时如何区分算法共享状态与基础设施共享内存机制？
```

核心矛盾：并行节点要让多个 worker
协作推进同一个算法状态；但基础设施只能提供
DSM、DSA、barrier、LWLock
等通用原语，真正的算法语义必须由节点自己定义。

学完后应能判断：能区分 ExecInitParallelPlan
创建的通用共享布局和 Parallel Hash /
Parallel Append 节点自己放入 DSM / DSA
的运行状态。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节聚焦 EXPLAIN 如何把 worker 明细输出出来。

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

下一节会回到 cleanup 顺序，解释为什么指标汇总必须发生在
worker finish 和 DSM cleanup 之间。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
并行执行器先给每个 parallel-aware 节点一次 estimate / initialize / worker attach 机会；节点把自己的共享状态放进 TOC 或 DSA，并用 barrier、LWLock、phase、finished bitmap 等字段协调 worker。
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
| 1 | `src/backend/executor/execParallel.c` | ExecParallelEstimate()、ExecParallelInitializeDSM()、ExecParallelInitializeWorker() 调度 parallel-aware 节点回调。 |
| 2 | `src/backend/executor/nodeHashjoin.c` | ExecParallelHashJoin()、ExecHashJoinInitializeDSM()、ExecHashJoinInitializeWorker() 和 parallel hash join 状态机注释。 |
| 3 | `src/backend/executor/nodeHash.c` | MultiExecParallelHash()、ExecHashInitializeDSM()、ExecHashInitializeWorker()、ExecHashRetrieveInstrumentation()。 |
| 4 | `src/include/executor/hashjoin.h` | ParallelHashJoinState、ParallelHashJoinBatch、PHJ_BUILD_*、PHJ_BATCH_*、PHJ_GROW_* phase。 |
| 5 | `src/backend/executor/nodeAppend.c` | ParallelAppendState、ExecAppendInitializeDSM()、ExecAppendInitializeWorker()、pa_next_plan / pa_finished。 |
| 6 | `src/include/executor/nodeAppend.h` | Append 节点并行 DSM 初始化接口。 |
| 7 | `src/include/storage/barrier.h` | Barrier phase 和 arrive/wait/detach 的并发协调原语。 |
| 8 | `src/include/storage/lwlock.h` | Parallel Append 使用 LWLock 保护共享选择状态。 |

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
| `ParallelHashJoinState` | Parallel Hash Join 的共享根状态，包含 batches、growth、nparticipants 和多个 barrier。 |
| `ParallelHashJoinBatch` | 每个 batch 的共享 barrier、chunk 链、tuplestore 位置和统计边界。 |
| `PHJ_BUILD_* phase` | build 阶段的共享进度，从 elect、allocate、hash inner 到 run / free。 |
| `PHJ_BATCH_* phase` | 单个 batch 的 allocate、load、probe、scan、free 状态机。 |
| `PHJ_GROWTH_*` | 并行 hash 在 build 中发现内存压力或倾斜后触发 buckets / batches 增长。 |
| `ParallelAppendState` | Parallel Append 的共享选择状态，包含 pa_lock、pa_next_plan 和 pa_finished 数组。 |
| `AppendState.as_pstate` | worker attach 后指向共享 ParallelAppendState，用于选择下一个 subplan。 |
| `node-specific instrumentation` | Hash、Append 等节点自己决定哪些共享状态最终转成 EXPLAIN 指标。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`ParallelHashJoinState`
的关键点是：Parallel Hash Join
的共享根状态，包含
batches、growth、nparticipants 和多个
barrier。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ParallelHashJoinBatch` 的关键点是：每个
batch 的共享 barrier、chunk
链、tuplestore 位置和统计边界。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`PHJ_BUILD_* phase` 的关键点是：build
阶段的共享进度，从 elect、allocate、hash
inner 到 run / free。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`PHJ_BATCH_* phase` 的关键点是：单个
batch 的
allocate、load、probe、scan、free
状态机。 诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`PHJ_GROWTH_*` 的关键点是：并行 hash 在
build 中发现内存压力或倾斜后触发 buckets /
batches 增长。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ParallelAppendState`
的关键点是：Parallel Append 的共享选择状态，包含
pa_lock、pa_next_plan 和
pa_finished 数组。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`AppendState.as_pstate`
的关键点是：worker attach 后指向共享
ParallelAppendState，用于选择下一个
subplan。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`node-specific instrumentation`
的关键点是：Hash、Append
等节点自己决定哪些共享状态最终转成 EXPLAIN 指标。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

`ExecInitParallelPlan()` 的
estimate 遍历遇到 parallel-aware
节点时调用节点 estimate 回调。

通用执行器不理解 hash batch 或 append
subplan，只负责给节点申请空间机会。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

initialize DSM
阶段，`ExecParallelInitializeDSM()`
再次遍历 PlanState。

对 HashJoin / Hash / Append
等节点，它调用节点的 DSM 初始化函数，把算法共享状态放入
TOC 或 DSA。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

worker 在 `ParallelQueryMain()`
中本地 `ExecutorStart()` 后，调用
`ExecParallelInitializeWorker()`。

这一步让 worker 的本地 PlanState attach
到节点共享状态。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

Parallel Hash Join 的 build 由
`MultiExecParallelHash()` 和
barrier phase 协调。

多个 worker 在 build_barrier 上推进
PHJ_BUILD_ALLOCATE、PHJ_BUILD_HASH_INNER、PHJ_BUILD_RUN
等阶段。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

如果 build 中需要增加 batch 或
bucket，growth barrier 会组织一次增长周期。

这不是 DSM 基础设施自动完成的，而是 hash
节点的算法状态机。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

每个 batch 又有自己的
`ParallelHashJoinBatch` 和
batch_barrier。

batch 级状态控制 load、probe、unmatched
scan 和 free。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

Parallel Append
的共享状态更小：`pa_lock` 保护
`pa_next_plan` 和
`pa_finished[]`。

worker 通过锁选择下一个 subplan，完成后标记
finished。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

Append 的共享状态回答“哪个 worker 接下来扫哪个
child”。

Hash 的共享状态回答“所有 worker 共同推进哪个
build / batch phase”。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

EXPLAIN 能看到的 hash
buckets、batches、memory 或 append
rows，是节点把算法状态转成 instrumentation
后的结果。

看不到的 barrier phase 只能通过
gdb、trace 或 wait event 间接观察。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

cleanup 时，节点共享状态必须在所有 worker
detach 后释放。

否则仍在 probe 或 append 选择的 worker
会访问已销毁状态。

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

节点共享状态的空间估算发生在 DSM 创建前，初始化发生在
DSM 创建后。

leader 和 worker 的本地 PlanState 分别
attach 到同一算法共享状态，但仍各自拥有本地表达式和
slot。

Parallel Hash 的 DSA 分配、batch
accessor 和 barrier detach 必须按
phase 收尾。

Parallel Append 的 pa_finished
只在同一次并行执行中有效，rescan 需要重新初始化。

节点 retrieve instrumentation 在
leader cleanup 阶段把共享统计转到本地
PlanState。

ERROR 或 worker 退出依赖
ParallelContext 和 barrier detach
规则防止其他参与者永久等待。

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
| 通用与专用分层 | DSM/DSA/barrier/LWLock 是基础设施，PHJ_BUILD_* 和 pa_next_plan 才是节点算法语义。 |
| phase 不变量 | Parallel Hash 只能在正确 phase 执行 allocate、hash、probe、scan 和 free。 |
| 锁保护 | Parallel Append 选择 subplan 时必须持 pa_lock，防止多个 worker 重复选择同一 finished plan。 |
| worker attach | worker 本地 PlanState 必须先 attach 节点共享状态，再执行 ExecProcNode。 |
| rescan 边界 | 并行节点 rescan 需要 reinitialize 共享状态，不能沿用上次执行的 finished / batch phase。 |
| 观测边界 | EXPLAIN 指标是共享状态的投影，不是共享状态本身完整快照。 |

通用与专用分层
这一层保证的是：DSM/DSA/barrier/LWLock
是基础设施，PHJ_BUILD_* 和 pa_next_plan
才是节点算法语义。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

phase 不变量 这一层保证的是：Parallel Hash
只能在正确 phase 执行
allocate、hash、probe、scan 和 free。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

锁保护 这一层保证的是：Parallel Append 选择
subplan 时必须持 pa_lock，防止多个 worker
重复选择同一 finished plan。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

worker attach 这一层保证的是：worker 本地
PlanState 必须先 attach 节点共享状态，再执行
ExecProcNode。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

rescan 边界 这一层保证的是：并行节点 rescan 需要
reinitialize 共享状态，不能沿用上次执行的
finished / batch phase。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

观测边界 这一层保证的是：EXPLAIN
指标是共享状态的投影，不是共享状态本身完整快照。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 把 DSM 机制当算法语义 | 知道有 shared memory 不等于知道 hash join 当前处于哪个 phase。 |
| 忽略 barrier phase | Parallel Hash 的等待可能来自 build/grow/batch phase，不是普通锁等待。 |
| 把 Append rows 倾斜全部归因于 planner | Parallel Append 的运行时 subplan 选择和 child 完成顺序也会影响分布。 |
| rescan 没有重置共享状态 | pa_finished 或 hash batch phase 残留会让下一轮执行错误跳过工作。 |
| 只看最终 batches | Parallel Hash 可能经历增长过程，最终值不能说明增长成本发生在哪里。 |
| 误把 worker 本地内存当共享内存 | batch accessor、slot、ExprContext 仍是本地状态。 |

场景“把 DSM 机制当算法语义”的处理思路是：知道有
shared memory 不等于知道 hash join
当前处于哪个 phase。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忽略 barrier
phase”的处理思路是：Parallel Hash
的等待可能来自 build/grow/batch
phase，不是普通锁等待。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 Append rows 倾斜全部归因于
planner”的处理思路是：Parallel Append
的运行时 subplan 选择和 child
完成顺序也会影响分布。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“rescan
没有重置共享状态”的处理思路是：pa_finished 或
hash batch phase
残留会让下一轮执行错误跳过工作。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“只看最终 batches”的处理思路是：Parallel
Hash 可能经历增长过程，最终值不能说明增长成本发生在哪里。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“误把 worker
本地内存当共享内存”的处理思路是：batch
accessor、slot、ExprContext
仍是本地状态。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

Parallel Hash 的 barrier 同步成本随
worker 数和 batch/growth 次数增加。

batch 增长会引入
repartition、tuplestore 和额外 DSA
分配成本。

Parallel Append 的 LWLock 临界区短，但
child 数多且 worker 竞争激烈时仍可见。

共享状态越细，负载均衡越好，但 coordination
成本也越高。

EXPLAIN 只显示部分算法结果，无法直接显示每次
barrier 等待成本。

worker 数增加不必然降低 runtime，可能把瓶颈从
CPU 推到同步和内存带宽。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

Parallel Hash 先看 Hash 节点的
Buckets、Batches、Peak Memory
Usage 和 worker 明细。

如果 batch 数明显增加，回到 `PHJ_GROWTH_*`
和 work_mem / hash_mem_multiplier
判断增长原因。

Parallel Append 看各 child 的 rows
和 worker 分布，判断是否有某些 child
成为尾部拖慢者。

gdb 中打印
`BarrierPhase(&pstate->build_barrier)`
可以看到 Parallel Hash 当前 phase。

打印
`AppendState->as_pstate->pa_next_plan`
和 `pa_finished[]` 可以观察 subplan
分配。

wait event 中的 hash build /
barrier 相关等待可以辅助区分同步等待和执行热点。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. 观察 Parallel Hash batch

降低 work_mem 或增大 build side

执行 parallel hash join 的 EXPLAIN
ANALYZE

观察 Hash Batches 是否增加以及 worker
rows 是否倾斜

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. 断点看 build phase

gdb 断在 `MultiExecParallelHash`

打印 `BarrierPhase(build_barrier)`

跟随 PHJ_BUILD_HASH_INNER 到
PHJ_BUILD_RUN

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. 观察 Parallel Append 分配

对分区表执行 parallel append 查询

断在 `ExecAppendInitializeWorker`
和选择 subplan 的代码

打印 `pa_next_plan` 与
`pa_finished`

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 比较 worker 数

修改
max_parallel_workers_per_gather

观察同步成本和 rows 分布是否线性改善

记录 Batches、Buffers 和 Execution
Time

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 区分共享和本地

打印 worker 本地 PlanState 地址

再打印共享 ParallelHashJoinState /
ParallelAppendState 地址或 DSA 指针

确认二者生命周期不同

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

1. 为什么 parallel-aware
节点要自己定义共享状态，而不是由 ExecParallel
统一处理？

2. Parallel Hash 的 phase
设计解决了什么正确性问题？

3. Parallel Append 为什么用一个较小的锁保护
subplan 选择，而不是预先静态分配所有 child？

4. 哪些 EXPLAIN
指标能反映算法共享状态，哪些只能通过源码调试看到？

5. worker
增加后性能下降，如何判断是同步、I/O、内存还是计划问题？

6. parallel-aware 节点 rescan
为什么比普通本地节点更敏感？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：算法共享状态与并行基础设施的边界

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. Parallel Hash 等待集中在 build 阶段

现象是 worker 都启动了，但 join
长时间不产出。源码上 build_barrier phase
决定当前是 allocate、hash inner、hash
outer 还是 run。

断在 MultiExecParallelHash()，打印
BarrierPhase(build_barrier) 和
pstate->growth。

结论是等待可能来自算法 phase，而不是 worker 启动或
tuple queue。

### 15.2. batch 数增长导致性能突然下降

现象是 Hash Batches 从 1
增加到多批，运行时间和临时文件增加。源码上
PHJ_GROWTH_NEED_MORE_BATCHES 会触发
repartition。

断在
ExecParallelHashIncreaseNumBatches()
和
ExecParallelHashRepartitionFirst/Rest()。

结论是最终 batches 只显示结果，真正成本发生在
growth cycle。

### 15.3. Parallel Append worker 分配不均

现象是某些 worker 处理大量
child，另一些很快结束。源码上 pa_next_plan 和
pa_finished 受 pa_lock 保护。

断在 nodeAppend.c 中选择并行子计划的位置，打印
pa_next_plan、as_whichplan、pa_finished[]。

结论是运行时 child 分配和 child 数据量共同决定
worker 分布，不能只看 planner 的 child
数。

### 15.4. 把 DSA 指针当本地指针使用

现象是 worker 访问共享状态崩溃或读到无意义地址。源码上
DSA 指针必须通过 dsa_get_address 或节点
attach 逻辑转换。

检查节点 InitializeDSM 和
InitializeWorker 是否分别写入 DSA 状态和
attach 本地 accessor。

结论是 DSM / DSA 只提供共享载体，worker
仍需要本地 accessor 和生命周期管理。

### 15.5. EXPLAIN 看不到 phase 历史

现象是 EXPLAIN 只能看到最终
Batches、Memory、rows，无法直接知道经历了几次
growth。

结合 gdb phase 断点、wait event 和
EXPLAIN worker 明细记录运行过程。

结论是 EXPLAIN
是结果投影。需要源码级诊断时，要在算法状态机上采样。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ExecParallelInitializeDSM()
  -> node-specific DSM initialize
ExecParallelInitializeWorker()
  -> node-specific worker attach
Parallel Hash
  -> PHJ_BUILD_* phase
  -> PHJ_GROW_* phase
  -> PHJ_BATCH_* phase
Parallel Append
  -> pa_lock
  -> pa_next_plan
  -> pa_finished[]
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

### 15.7. parallel-aware 节点复盘表

这张复盘表用于课后把现场现象、源码断点和最终归因放到同一页。它不是新的主题，只是把本节主问题落到一次可重复的诊断记录。

| 复盘项 | 记录方式 | 判断边界 |
| --- | --- | --- |
| 算法 phase | 记录 PHJ_BUILD_*、PHJ_BATCH_* 或 pa_next_plan | 区分算法状态和 DSM 基础设施 |
| 共享载体 | 记录 TOC key、DSA 指针或 shared state 地址 | 确认共享对象不是 backend-local 指针 |
| worker attach | 记录 InitializeWorker 后本地 accessor | 确认 worker 已接上共享状态 |
| 增长事件 | 记录 batches、buckets、growth 状态 | 区分最终结果和增长过程 |
| 负载分配 | 记录 worker rows 和 subplan 完成顺序 | 区分数据倾斜和调度选择 |

算法 phase：记录
PHJ_BUILD_*、PHJ_BATCH_* 或
pa_next_plan 这一项的判断边界是：区分算法状态和 DSM
基础设施 记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

共享载体：记录 TOC key、DSA 指针或 shared
state 地址 这一项的判断边界是：确认共享对象不是
backend-local 指针 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

worker attach：记录 InitializeWorker
后本地 accessor 这一项的判断边界是：确认 worker
已接上共享状态 记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

增长事件：记录 batches、buckets、growth 状态
这一项的判断边界是：区分最终结果和增长过程 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

负载分配：记录 worker rows 和 subplan 完成顺序
这一项的判断边界是：区分数据倾斜和调度选择 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

## 16. 本节小结

并行节点共享状态分为通用执行器布局和节点算法状态两层。

Parallel Hash 用 barrier phase
表达协作算法进度，Parallel Append 用锁和
finished 数组表达工作分配。

EXPLAIN 显示的是共享状态的结果投影，不是完整 phase
历史。

诊断并行节点要同时看 worker 分布、算法状态和基础设施等待。

可迁移规律是：共享内存只是载体，真正的语义在状态机和
ownership 中。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会回到 cleanup 顺序，解释为什么指标汇总必须发生在
worker finish 和 DSM cleanup 之间。
