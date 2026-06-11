# PostgreSQL Gather / GatherMerge tuple routing 与 leader participation 观测

## 课程定位

前置知识：理解并行查询 worker、tuple
queue、PlanState、ExecProcNode、EXPLAIN
ANALYZE 和
parallel_leader_participation。

本节唯一主问题：

```text
Gather / GatherMerge 如何从 worker tuple queue 和 leader 本地执行路径收 tuple，parallel_leader_participation 如何影响 rows、loops 和时间解读？
```

核心矛盾：并行查询要把多个生产者的 tuple
汇成单一上层流；但 leader
既可能是消费者，也可能参与执行子计划，导致
rows、loops、等待和局部执行时间不能简单按 worker
数平均解释。

学完后应能判断：能从 EXPLAIN 中区分 worker
生产、leader 本地生产、tuple queue 等待和
GatherMerge 排序归并的不同成本来源。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节讲清楚了 worker instrumentation
如何从 DSM 回到 leader。

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

下一节会沿 EXPLAIN 输出路径展开 per-worker
rows、loops、sort/hash/buffer
信息如何被格式化。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Gather 是多路 tuple queue 加可选 leader 本地扫描的 funnel；GatherMerge 在同样 worker/leader 输入之上维护有序 reader heap，保证输出顺序。
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
| 1 | `src/backend/executor/nodeGather.c` | ExecInitGather()、ExecGather()、gather_getnext()、gather_readnext()、ExecShutdownGather()。 |
| 2 | `src/backend/executor/nodeGatherMerge.c` | ExecInitGatherMerge()、ExecGatherMerge()、gather_merge_getnext()、gather_merge_readnext()、ExecShutdownGatherMerge()。 |
| 3 | `src/backend/executor/execParallel.c` | ExecInitParallelPlan()、LaunchParallelWorkers() 后的 tuple queue reader 创建与 cleanup。 |
| 4 | `src/include/optimizer/optimizer.h` | parallel_leader_participation GUC 的全局变量声明。 |
| 5 | `src/include/nodes/execnodes.h` | GatherState、GatherMergeState、PlanState 中相关运行态字段。 |
| 6 | `src/backend/commands/explain.c` | ExplainNode() 输出 Workers Planned / Workers Launched / Single Copy 和 worker instrumentation。 |
| 7 | `src/backend/executor/tqueue.c` | TupleQueueReaderNext() 读取 worker 输出 tuple。 |
| 8 | `src/backend/storage/ipc/shm_mq.c` | worker tuple queue 的底层共享消息队列。 |

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
| `GatherState.pei` | 并行执行器信息，持有 ParallelContext、tuple queues、DSA 和 instrumentation。 |
| `nworkers_launched` | 实际成功启动的 worker 数，EXPLAIN 中 Workers Launched 的来源。 |
| `nreaders / reader[]` | leader 当前还在读取的 TupleQueueReader 数组，worker 完成后会从 active readers 中移除。 |
| `nextreader` | Gather 的轮询位置，当前源码倾向继续读同一 queue 直到会阻塞。 |
| `need_to_scan_locally` | leader 是否执行 outerPlan 的本地副本，受 single_copy、nreaders 和 parallel_leader_participation 影响。 |
| `funnel_slot` | Gather 把 worker MinimalTuple 存入的输出 slot。 |
| `gm_heap / gm_slots` | GatherMerge 用 binaryheap 维护各参与者当前最小 tuple 的有序输出。 |
| `tuples_needed` | 上层 Limit 等节点传下来的 tuple bound，会影响 worker 和 leader 需求。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`GatherState.pei`
的关键点是：并行执行器信息，持有
ParallelContext、tuple queues、DSA
和 instrumentation。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`nworkers_launched`
的关键点是：实际成功启动的 worker 数，EXPLAIN 中
Workers Launched 的来源。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`nreaders / reader[]`
的关键点是：leader 当前还在读取的
TupleQueueReader 数组，worker 完成后会从
active readers 中移除。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`nextreader` 的关键点是：Gather
的轮询位置，当前源码倾向继续读同一 queue 直到会阻塞。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`need_to_scan_locally`
的关键点是：leader 是否执行 outerPlan
的本地副本，受 single_copy、nreaders 和
parallel_leader_participation
影响。 诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`funnel_slot` 的关键点是：Gather 把
worker MinimalTuple 存入的输出 slot。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`gm_heap / gm_slots`
的关键点是：GatherMerge 用 binaryheap
维护各参与者当前最小 tuple 的有序输出。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`tuples_needed` 的关键点是：上层 Limit
等节点传下来的 tuple bound，会影响 worker 和
leader 需求。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

`ExecInitGather()` 创建
GatherState，但不立刻启动 worker。

大 DSM 和 worker
启动成本被延迟到首次执行，避免未执行分支提前付费。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

`ExecGather()` 首次进入时，如果 plan 要求
worker 且 parallel mode 可用，则调用
`ExecInitParallelPlan()` 或
reinitialize。

这里建立子计划 worker 侧执行所需的序列化
plan、tuple queue 和共享状态。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

`LaunchParallelWorkers()` 尝试启动
worker，实际数量写入
`nworkers_launched`。

EXPLAIN 中 planned 和 launched
不一致，通常来自 worker 资源不足或运行时无法启动。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

如果 worker 数大于零，leader 调用
`ExecParallelCreateReaders()` 建立
tuple queue reader。

reader[] 是 leader 本地读取句柄，不是
worker 的 PlanState。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

Gather 根据 `single_copy` 和
`parallel_leader_participation`
计算 `need_to_scan_locally`。

leader 参与执行时，它既消费 worker
queue，也可能自己调用 outerPlan 产 tuple。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

每次取 tuple 前，Gather reset
per-tuple ExprContext。

这和普通节点一致，说明 funnel 节点也有表达式和
projection 生命周期。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

`gather_readnext()` 非阻塞轮询
reader，读到 MinimalTuple 后存入
funnel_slot。

如果所有 reader 暂时没有 tuple，且 leader
还能本地执行，就返回给 gather_getnext 走本地
outerPlan。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

没有本地执行可做时，leader 等待 latch，wait
event 是
`WAIT_EVENT_EXECUTE_GATHER`。

这类时间可能表现为 leader 等 worker，而不是
leader 自己 CPU 慢。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

GatherMerge 的启动类似，但
`gather_merge_getnext()`
会先为每个参与者读一条 tuple，放入 binaryheap。

输出顺序来自每个 reader 的当前 tuple
比较，不是简单 round-robin。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

Gather / GatherMerge shutdown 先
`ExecParallelFinish()`，再
`ExecParallelCleanup()`。

tuple queue、worker
finish、instrumentation merge 和
DSM cleanup 有明确顺序。

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

GatherState 在 ExecInitGather()
中创建，随 PlanState tree 生命周期由
ExecEndGather() 清理。

ParallelExecutorInfo
在首次执行时懒创建，ExecShutdownGather() 中
cleanup 后置空。

worker tuple queue reader 是
leader 本地对象，worker 完成或 shutdown
时释放。

need_to_scan_locally
是运行期状态，不能从计划中的 Workers Planned
直接推断。

GatherMerge 的 binaryheap 和 slots
属于节点私有状态，节点结束时释放。

worker
失败或提前结束时，TupleQueueReaderNext()
可能返回 done，真正错误会在等待 worker finish
时上抛。

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
| 单一输出流 | 上层节点只看到 Gather / GatherMerge 返回的一个 tuple 流，不直接面对多个 worker。 |
| leader 参与边界 | leader 本地执行 outerPlan 必须安装 es_query_dsa，避免访问并行节点共享状态时缺少 DSA。 |
| queue 完成语义 | reader done 后从 active readers 移除，避免再次轮询已结束 worker。 |
| 有序输出 | GatherMerge 通过每个 reader 当前 tuple 和 binaryheap 保持全局有序，而 Gather 不保证 worker 输出顺序。 |
| interrupt 和 latch | 轮询和等待路径都有 CHECK_FOR_INTERRUPTS，避免 leader 无响应。 |
| cleanup 顺序 | 先停止 worker 和 queue，再汇总 instrumentation，最后销毁 ParallelContext。 |

单一输出流 这一层保证的是：上层节点只看到 Gather /
GatherMerge 返回的一个 tuple
流，不直接面对多个 worker。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

leader 参与边界 这一层保证的是：leader 本地执行
outerPlan 必须安装
es_query_dsa，避免访问并行节点共享状态时缺少
DSA。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

queue 完成语义 这一层保证的是：reader done
后从 active readers 移除，避免再次轮询已结束
worker。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

有序输出 这一层保证的是：GatherMerge 通过每个
reader 当前 tuple 和 binaryheap
保持全局有序，而 Gather 不保证 worker 输出顺序。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

interrupt 和 latch
这一层保证的是：轮询和等待路径都有
CHECK_FOR_INTERRUPTS，避免 leader
无响应。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

cleanup 顺序 这一层保证的是：先停止 worker 和
queue，再汇总 instrumentation，最后销毁
ParallelContext。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 把 Workers Planned 当实际并行度 | 实际 Workers Launched 可能更少，甚至为零，此时 leader 可能独自执行。 |
| 把 Gather rows 平均分给 worker | leader participation、tuple routing 和 worker 倾斜都会让平均数误导。 |
| 忽略 Single Copy | single_copy 会改变 leader 是否执行本地副本，也改变可解释的 rows 来源。 |
| 把 Gather wait 当 CPU 慢 | leader 可能在 latch 上等 worker tuple queue，不是本地执行热点。 |
| 把 GatherMerge 当 Gather 加 Sort | GatherMerge 维护有序输入 heap，不等同于先收完所有 tuple 再排序。 |
| 过早清理 pei | EXPLAIN 需要 shutdown 后仍有机会从 DSM 汇总 worker instrumentation。 |

场景“把 Workers Planned
当实际并行度”的处理思路是：实际 Workers
Launched 可能更少，甚至为零，此时 leader
可能独自执行。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 Gather rows 平均分给
worker”的处理思路是：leader
participation、tuple routing 和
worker 倾斜都会让平均数误导。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忽略 Single
Copy”的处理思路是：single_copy 会改变
leader 是否执行本地副本，也改变可解释的 rows 来源。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 Gather wait 当 CPU
慢”的处理思路是：leader 可能在 latch 上等
worker tuple queue，不是本地执行热点。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 GatherMerge 当 Gather 加
Sort”的处理思路是：GatherMerge 维护有序输入
heap，不等同于先收完所有 tuple 再排序。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“过早清理 pei”的处理思路是：EXPLAIN 需要
shutdown 后仍有机会从 DSM 汇总 worker
instrumentation。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

Gather 的成本包括 worker 启动、tuple
queue 通信、leader 轮询、可选本地执行和
projection。

GatherMerge 额外承担每个参与者当前 tuple
的比较和 binaryheap 维护成本。

parallel_leader_participation
可能提高吞吐，也可能让 leader 没有足够时间消费
queue。

worker 启动不足会让计划成本估计和实际执行形态偏离。

上层 Limit 通过 tuple bound 减少需要的
tuple，但 worker 启动和 DSM
初始化成本仍可能已经付出。

队列读取策略保留当前 reader
直到会阻塞，减少轮询开销但会影响短期 rows 分布观察。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

EXPLAIN 中先看 Workers Planned 和
Workers Launched，不要直接按 planned
worker 数解释时间。

打开 VERBOSE 看 worker rows /
loops，判断是否存在明显倾斜。

观察
`parallel_leader_participation`
开关前后，Gather 子计划 leader 和 worker
rows 是否变化。

用 wait event 查看 leader 是否停在
execute gather 等待。

断在 `gather_readnext()`，打印
nreaders、nextreader 和
readerdone。

断在 `gather_getnext()`，打印
need_to_scan_locally，确认 leader
是否调用 outerPlan。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. 比较 leader participation

设置
`parallel_leader_participation =
on`

执行并行聚合或大表 scan 的 EXPLAIN ANALYZE

再设置为 off，对比 worker rows 和总时间

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. 制造 worker 启动不足

降低 `max_parallel_workers` 或占满
worker slot

执行同一并行计划

观察 Workers Planned 与 Workers
Launched 差异

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. 观察 Gather wait

在慢 worker 查询上查看 pg_stat_activity
wait_event

若 leader 等 tuple queue，可能出现
ExecuteGather 相关等待

结合 perf 判断是否真是 CPU 热点

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 比较 Gather 和 GatherMerge

对无序并行 scan 看 Gather

对需要保持 order 的查询看 Gather Merge

观察 GatherMerge 下方通常要求有序子路径

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 断点阅读

gdb 断在
`ExecGather`、`gather_readnext`、`ExecShutdownGather`

打印 `node->nreaders` 和
`node->need_to_scan_locally`

确认状态随执行推进变化

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

1. leader 参与执行为什么既可能提升性能，也可能降低
tuple queue 消费效率？

2. 为什么 Gather 不保证 worker tuple
的全局顺序，而 GatherMerge 可以？

3. 看到 Workers Launched 为 0
时，应该怎样重新解释并行计划？

4. Gather 节点 total time 是否能直接代表
worker CPU 时间？

5. 如果某个 worker rows
极低，是数据倾斜、启动晚、queue 等待还是局部过滤？

6. 为什么 shutdown 必须把 worker
finish 和 instrumentation cleanup
分开？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：Gather 读数与实际并行效率不一致

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. Workers Planned 大于 Workers Launched

现象是计划看起来并行度很高，但实际启动 worker 少。先看
GatherState->nworkers_launched，而不是根据
plan 中 num_workers 推断。

断点放在 ExecGather() 调用
LaunchParallelWorkers() 后，打印
pcxt->nworkers_to_launch 和
pcxt->nworkers_launched。

结论是运行时 worker
资源不足会改变执行形态。EXPLAIN 中 planned 和
launched 必须分开读。

### 15.2. leader 参与导致 rows 分布难解释

现象是 worker rows 加起来少于总输出，或者关闭
leader participation 后分布变化。源码上
need_to_scan_locally 决定 leader
是否执行 outerPlan。

断在 gather_getnext()，打印
need_to_scan_locally、nreaders 和
outerTupleSlot 是否来自
ExecProcNode(outerPlan)。

结论是 leader 同时可能是消费者和生产者。Gather
节点总 rows 不是 worker rows 的简单包装。

### 15.3. Gather 等待被误判为 CPU 热点

现象是 SQL 慢，但 perf 中 leader
不忙，pg_stat_activity 看到等待。源码上
gather_readnext() 在没有本地工作时等待
latch。

断在 WaitLatch 调用附近，确认 wait event
是 execute gather 相关路径，并检查 worker
是否仍有 active reader。

结论是 leader 等 tuple queue 与
leader 执行慢是两种问题。下一步应看 worker
rows、I/O 和下层节点。

### 15.4. GatherMerge 输出慢但子节点 rows 正常

现象是每个 worker 都有输出，但 Gather Merge
节点耗时高。源码上它维护 binaryheap，并持续从各
reader 补 tuple。

断在 gather_merge_getnext() 和
gather_merge_readnext()，观察
gm_heap、reader、nowait 和 done 状态。

结论是有序归并本身有协调和比较成本。它不是普通 Gather
后再排序。

### 15.5. worker 提前结束后仍被轮询

现象是某个 worker 无 tuple
后仍似乎占用轮询。源码上 readerdone 后会从
active reader 数组中 memmove 删除。

在 gather_readnext() 中打印
readerdone 前后的 nreaders 和
nextreader。

结论是 active reader 集合会收缩。若诊断工具仍按原
worker 数平均，解释会偏差。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ExecGather() first call
  -> ExecInitParallelPlan()
  -> LaunchParallelWorkers()
  -> ExecParallelCreateReaders()
  -> need_to_scan_locally decision
gather_getnext()
  -> gather_readnext()
  -> optional local ExecProcNode()
ExecShutdownGather()
  -> ExecParallelFinish()
  -> ExecParallelCleanup()
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

### 15.7. Gather 现场复盘表

这张复盘表用于课后把现场现象、源码断点和最终归因放到同一页。它不是新的主题，只是把本节主问题落到一次可重复的诊断记录。

| 复盘项 | 记录方式 | 判断边界 |
| --- | --- | --- |
| worker 启动差异 | 记录 Workers Planned、Workers Launched、nworkers_launched | 区分计划并行度和实际并行度 |
| leader 本地执行 | 记录 need_to_scan_locally 和 parallel_leader_participation | 区分 leader 生产 tuple 与消费 queue |
| queue 等待 | 记录 wait event、nreaders、nextreader | 区分等待 worker 与本地 CPU 热点 |
| 归并成本 | GatherMerge 记录 gm_heap 和 reader done 状态 | 区分普通合流和有序归并 |

worker 启动差异：记录 Workers
Planned、Workers
Launched、nworkers_launched
这一项的判断边界是：区分计划并行度和实际并行度 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

leader 本地执行：记录
need_to_scan_locally 和
parallel_leader_participation
这一项的判断边界是：区分 leader 生产 tuple 与消费
queue 记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

queue 等待：记录 wait
event、nreaders、nextreader
这一项的判断边界是：区分等待 worker 与本地 CPU 热点
记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

归并成本：GatherMerge 记录 gm_heap 和
reader done 状态
这一项的判断边界是：区分普通合流和有序归并 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

## 16. 本节小结

Gather 是并行子计划和上层单一 tuple 流之间的
funnel。

GatherMerge 在 funnel
之上增加有序归并状态，不是普通 Gather 的输出再排序。

leader participation 改变
rows、loops、等待和时间的解释口径。

Workers Planned、Workers
Launched、worker rows 和 wait event
必须一起看。

可迁移规律是：并行节点的观测要先拆开生产者、消费者和合流点。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会沿 EXPLAIN 输出路径展开 per-worker
rows、loops、sort/hash/buffer
信息如何被格式化。
