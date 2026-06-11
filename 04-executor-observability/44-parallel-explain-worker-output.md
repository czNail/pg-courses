# PostgreSQL 并行节点的 per-worker EXPLAIN 输出

## 课程定位

前置知识：理解 EXPLAIN
ANALYZE、NodeInstrumentation、WorkerNodeInstrumentation、Sort/Hash/Agg/Bitmap
等节点的 runtime instrumentation。

本节唯一主问题：

```text
EXPLAIN 如何展示每个 worker 的 actual rows、loops、sort / hash / buffer / timing 信息，为什么 worker 间倾斜比总耗时更能解释并行效率？
```

核心矛盾：用户想看一个简洁的总计划；但并行执行的真实性能问题常藏在
worker 之间的
rows、loops、内存、spill、buffer 和 I/O
差异中。

学完后应能判断：能沿 `ExplainNode()` 找到
per-worker 输出从哪里来，并能解释哪些指标是
worker 明细、哪些是 leader
聚合、哪些只是节点级总值。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

上一节跟踪了 Gather 和 GatherMerge 如何把
worker tuple 流汇到 leader。

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

下一节会继续看 Parallel Hash 和 Parallel
Append 这类节点自身共享运行状态如何影响观测。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
leader 在 parallel cleanup 后把 DSM 中的 worker instrumentation 复制到 PlanState；ExplainNode() 为每个节点创建 ExplainWorkersState，在主节点输出后按 worker 打开临时输出组，再 flush 到 Workers 区域。
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
| 1 | `src/backend/commands/explain.c` | ExplainNode()、ExplainCreateWorkersState()、ExplainOpenWorker()、ExplainCloseWorker()、ExplainFlushWorkersState() 以及 sort/hash/buffer worker 输出。 |
| 2 | `src/backend/executor/execParallel.c` | ExecParallelRetrieveInstrumentation() 和节点级 RetrieveInstrumentation 回调。 |
| 3 | `src/include/executor/instrument.h` | WorkerNodeInstrumentation 和 NodeInstrumentation 字段。 |
| 4 | `src/backend/executor/instrument.c` | InstrAggNode() 聚合 rows、loops、filtered、buffer、WAL。 |
| 5 | `src/backend/executor/nodeSort.c` | ExecSortRetrieveInstrumentation() 提供 worker sort stats。 |
| 6 | `src/backend/executor/nodeHash.c` | ExecHashRetrieveInstrumentation() 提供 hash buckets、batches、memory 等信息。 |
| 7 | `src/backend/executor/nodeAgg.c` | ExecAggRetrieveInstrumentation() 提供 HashAgg worker memory / batch / disk 信息。 |
| 8 | `src/backend/executor/nodeBitmapHeapscan.c` | Bitmap heap scan worker block / prefetch instrumentation。 |

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
| `planstate->worker_instrument` | 每个节点的 per-worker NodeInstrumentation 明细，ExplainNode() 的 worker actual rows / loops 来源。 |
| `ExplainWorkersState` | EXPLAIN 格式化时的临时 worker 输出缓冲，避免 worker 行打断主节点输出。 |
| `es->hide_workers` | 控制是否隐藏 worker 明细，影响 ExplainNode 是否创建 workers_state。 |
| `NodeInstrumentation.nloops` | per-worker loops，rows 会除以 loops 后显示，不能只看 ntuples 原始和。 |
| `SortInstrumentation` | Sort 节点的 worker sort method、space used、space type。 |
| `HashInstrumentation` | Hash 节点的 buckets、batches、original batches、peak memory usage。 |
| `BufferUsage` | 节点级或 worker 节点级 buffer 统计，是否出现取决于 BUFFERS 和 instrumentation options。 |
| `JitInstrumentation` | worker JIT 明细可以单独输出，不等同于节点 rows。 |

这些状态不能孤立读取。raw field 不是语义；field
加 flag、lifecycle
state、owner、调用阶段和观测口径，才构成本节真正需要掌握的语义。

`planstate->worker_instrument`
的关键点是：每个节点的 per-worker
NodeInstrumentation
明细，ExplainNode() 的 worker actual
rows / loops 来源。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`ExplainWorkersState`
的关键点是：EXPLAIN 格式化时的临时 worker
输出缓冲，避免 worker 行打断主节点输出。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`es->hide_workers` 的关键点是：控制是否隐藏
worker 明细，影响 ExplainNode 是否创建
workers_state。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`NodeInstrumentation.nloops`
的关键点是：per-worker loops，rows 会除以
loops 后显示，不能只看 ntuples 原始和。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`SortInstrumentation` 的关键点是：Sort
节点的 worker sort method、space
used、space type。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`HashInstrumentation` 的关键点是：Hash
节点的 buckets、batches、original
batches、peak memory usage。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`BufferUsage` 的关键点是：节点级或 worker
节点级 buffer 统计，是否出现取决于 BUFFERS 和
instrumentation options。
诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

`JitInstrumentation`
的关键点是：worker JIT 明细可以单独输出，不等同于节点
rows。 诊断时不要只打印这个字段本身，还要同时打印它所在的
PlanState、EState、DSM、worker
number 或 ExplainState 边界。

## 5. 主流程源码 walkthrough

下面按时间顺序走一条主链路。每一步都回答“状态改变了什么”，而不是只记录函数名。

### 5.1. 步骤 1

parallel worker
执行结束后，`ExecParallelReportInstrumentation()`
把每个节点的 NodeInstrumentation 写回
DSM。

这是 EXPLAIN 能看到 worker actual
rows / loops 的前提。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.2. 步骤 2

leader 在
`ExecParallelRetrieveInstrumentation()`
中聚合到
`planstate->instrument`，并复制明细到
`worker_instrument`。

因此主节点行显示的是聚合口径，Worker N
行显示的是明细口径。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.3. 步骤 3

`ExplainNode()` 进入每个 PlanState
时，如果 worker_instrument
存在、analyze 开启且未 hide workers，就创建
`ExplainWorkersState`。

这一步只准备输出缓冲，不改变执行状态。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.4. 步骤 4

主节点先输出 Node Type、cost、actual
rows、loops 和普通属性。

这保证即使 worker 明细很长，计划树主干仍然可读。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.5. 步骤 5

如果 VERBOSE 且有 worker
instrumentation，ExplainNode 会为每个
worker 输出 actual
time、rows、loops。

TEXT 格式使用 `Worker N:` 行，非 TEXT
格式写入 Worker group。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.6. 步骤 6

节点专属输出函数再按需要打开 worker 输出。

例如 sort、incremental
sort、hash、memoize、hash
agg、bitmap heap scan 都有自己的
worker 明细分支。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.7. 步骤 7

`ExplainOpenWorker()` 和
`ExplainCloseWorker()` 成对维护每个
worker 的临时输出位置。

这样不同属性可以追加到同一个 Worker N 块中。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.8. 步骤 8

节点属性输出完成后，`ExplainFlushWorkersState()`
把 worker 缓冲合并到最终输出。

这也是为什么 worker
明细在格式化阶段组织，而不是执行阶段直接打印。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

### 5.9. 步骤 9

如果某个 worker nloops 为零，通用 actual
行可能跳过它。

这不是 worker 不存在，而是该节点在这个 worker
中未执行或未产生完成 loop。

这一步的阅读重点是确认调用者协议：它是否允许重复进入，失败时是否已经拥有
cleanup 边界，返回后上层能依赖什么不变量。

如果在这里下断点，优先打印 owner
指针和阶段标志，再打印计数器。单独的计数值经常会误导，因为它还没有放回生命周期。

### 5.10. 步骤 10

非 TEXT 格式会显式记录 Worker
Number，适合机器解析。

诊断自动化最好优先使用 JSON/YAML，而不是解析缩进文本。

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

worker instrumentation 的采集发生在
worker ExecutorEnd 之前，确保节点状态还可读。

leader 的 worker_instrument 明细挂在
query context，保证 EXPLAIN 打印前仍有效。

ExplainWorkersState 是格式化阶段临时状态，随
ExplainState 使用完释放。

node-specific worker stats
通常由对应节点的 RetrieveInstrumentation
回调复制出来。

EXPLAIN 打印完成后才允许 ExecutorEnd 清理
queryDesc，防止 instrumentation
被提前释放。

hide_workers 只影响输出，不改变执行和聚合行为。

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
| 明细与汇总分离 | planstate->instrument 是聚合口径，worker_instrument 是明细口径，不能互相替代。 |
| 输出延迟 | worker 输出不是 worker 直接写日志，而是 leader 在 EXPLAIN 阶段格式化。 |
| nloops 归一化 | actual rows 显示为 ntuples / nloops，loops 不同时不能只比较 rows。 |
| 节点特化 | Sort、Hash、Agg、Bitmap 等节点的额外 worker 指标必须由节点自己 retrieve。 |
| 格式化配对 | ExplainOpenWorker 和 ExplainCloseWorker 必须配对，否则 TEXT / JSON 结构会错乱。 |
| 选项边界 | 没有 ANALYZE 或被 hide workers 时，worker 明细不会输出，但计划仍可执行。 |

明细与汇总分离
这一层保证的是：planstate->instrument
是聚合口径，worker_instrument
是明细口径，不能互相替代。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

输出延迟 这一层保证的是：worker 输出不是 worker
直接写日志，而是 leader 在 EXPLAIN 阶段格式化。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

nloops 归一化 这一层保证的是：actual rows
显示为 ntuples / nloops，loops
不同时不能只比较 rows。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

节点特化
这一层保证的是：Sort、Hash、Agg、Bitmap
等节点的额外 worker 指标必须由节点自己
retrieve。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

格式化配对 这一层保证的是：ExplainOpenWorker
和 ExplainCloseWorker 必须配对，否则
TEXT / JSON 结构会错乱。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

选项边界 这一层保证的是：没有 ANALYZE 或被 hide
workers 时，worker 明细不会输出，但计划仍可执行。
它不能顺便保证其它层的语义。比如锁不等于统计准确，instrumentation
不等于计划选择正确，DSM 可见也不等于指针可跨进程使用。

PostgreSQL
的执行器正确性通常来自多层机制叠加，而不是某一个字段或某一个锁。读源码时要把这些层次拆开。

## 9. 错误路径 / 异常路径 / fallback

| 场景 | 应如何解释 |
| --- | --- |
| 只看 Gather 总时间 | 总时间可能掩盖某个 worker 的 spill、低 rows 或等待。 |
| 忽略 loops | worker rows 是按 loops 平均后的显示值，loops 不同会改变解释。 |
| TEXT 解析过度依赖缩进 | 版本和格式选项会改变文本形态，自动工具应使用 JSON。 |
| 把 worker 0 当 leader | Worker Number 是 worker 槽号，不是 leader 线程编号。 |
| 未打开 VERBOSE 就找不到通用 worker actual 行 | 某些 worker 明细依赖 verbose 输出。 |
| 把缺失 worker 行当错误 | 该 worker 可能未执行该节点，或 nloops 为零。 |

场景“只看 Gather
总时间”的处理思路是：总时间可能掩盖某个 worker 的
spill、低 rows 或等待。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“忽略 loops”的处理思路是：worker rows
是按 loops 平均后的显示值，loops 不同会改变解释。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“TEXT
解析过度依赖缩进”的处理思路是：版本和格式选项会改变文本形态，自动工具应使用
JSON。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把 worker 0 当
leader”的处理思路是：Worker Number 是
worker 槽号，不是 leader 线程编号。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“未打开 VERBOSE 就找不到通用 worker
actual 行”的处理思路是：某些 worker 明细依赖
verbose 输出。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

场景“把缺失 worker 行当错误”的处理思路是：该
worker 可能未执行该节点，或 nloops 为零。
如果现场现象类似，先确认它发生在哪个生命周期阶段，再决定应该查
hook、planner、executor、parallel
cleanup、EXPLAIN 格式化还是统计信息。

fallback 路径通常比正常路径更能暴露边界。比如
worker 启动失败、EXPLAIN
ONLY、不启动并行、nloops 为零、previous
hook 为空，都不是异常噪音，而是系统有意保留的边界。

## 10. 成本、资源与跨模块传播

per-worker 明细让 EXPLAIN
输出变大，复杂计划和大量 worker 会显著增加日志体积。

节点特化 instrumentation
需要额外内存和复制成本，尤其 sort/hash/agg
有数组和统计结构。

输出格式化本身在 EXPLAIN ANALYZE 路径上，慢
SQL 日志中也会放大延迟。

timing 打开后每个 worker 每个节点都承担计时开销。

BUFFERS/WAL/MEMORY 等选项会扩展
NodeInstrumentation
或节点私有统计的采集范围。

worker
倾斜诊断需要保留明细，不能为了缩短输出而只打印总计。

成本传播要沿调用链解释：一个本地字段变化，可能在 EXPLAIN
输出中变成 rows/loops；一个 DSM
布局选择，可能变成 worker
明细是否可读；一个统计误差，可能变成 join order 和
I/O 放大。

不要只问“这个函数贵不贵”。更有用的问题是：这个成本随
tuple 数、worker 数、plan node
数、loops 数、输出格式、日志频率或统计目标如何扩张。

## 11. 观测与诊断入口

使用 `EXPLAIN (ANALYZE, VERBOSE,
BUFFERS, FORMAT JSON)` 获取机器可读
worker 明细。

先看每个并行子节点的 Worker N rows /
loops，再看 Gather 的 Workers
Launched。

Sort 节点关注 worker Sort
Method、Sort Space Used、Sort
Space Type。

Hash / HashAgg 关注 Batches、Peak
Memory Usage、Disk Usage 和 worker
间差异。

Bitmap Heap Scan 关注 Exact/Lossy
Heap Blocks 的 worker 分布。

如果总时间高但 worker rows 分布均衡，再转向
wait event、I/O timing 或 leader
funnel。

诊断时建议保留三个层次的证据：SQL / EXPLAIN
看到的现象，gdb
或日志看到的运行态字段，以及源码中对应的状态推进点。只有三者能互相解释，结论才稳。

如果只有
EXPLAIN，没有源码入口，容易过度归因；如果只有源码，没有可复现现象，容易把正常实现细节误判成问题。

## 12. 课堂实验

### 12.1. 输出 JSON worker 明细

执行 `EXPLAIN (ANALYZE, VERBOSE,
BUFFERS, FORMAT JSON)`

定位 Plan 下的 Workers 数组

比较 Worker Number、Actual
Rows、Actual Loops

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.2. 制造 sort spill 差异

降低 work_mem

执行并行排序查询

观察各 worker Sort Space Type 是否同为
Disk

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.3. 观察 worker 倾斜

构造分布不均的分区或谓词

执行并行 scan / aggregate

比较 Worker N rows 差异

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.4. 隐藏 worker 输出

执行带 `SUMMARY` 和不同 VERBOSE 设置的
EXPLAIN

观察通用 worker actual 行是否出现

确认执行本身不因输出隐藏改变

实验记录至少包含 SQL、EXPLAIN 选项、相关
GUC、断点位置和观察到的关键字段。只记录最终耗时，不足以回到源码解释。

### 12.5. 源码断点

断在 `ExplainOpenWorker` 和
`ExplainFlushWorkersState`

打印 `es->format` 和
`es->workers_state`

确认 worker 输出是格式化阶段组织的

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

1. 为什么 worker 明细比并行计划总耗时更适合定位倾斜？

2. 为什么 EXPLAIN 不让 worker
直接输出自己的计划片段？

3. 什么时候 TEXT 格式足够，什么时候必须用 JSON？

4. worker rows
均衡但总时间不降，可能是哪几类问题？

5. 为什么节点特化 worker 指标不能全部放进通用
NodeInstrumentation？

6. 如何在日志体积和诊断价值之间选择 EXPLAIN 选项？

讨论时不要停在观点层面。每个回答都应能指回一个源码入口、一个状态字段和一个可复现的
runtime 现象。

## 15. 诊断案例：per-worker EXPLAIN 怎么读才不误判

下面用现场诊断方式把本节模型压实。每个案例都从可见现象进入，再回到源码入口，最后给出归因边界。

### 15.1. Worker N 行缺失

现象是某些 worker 没有 actual 行。先检查
nloops 是否为零，以及 EXPLAIN 是否开启
ANALYZE、VERBOSE、hide_workers。

断在 ExplainNode() 中处理
worker_instrument 的循环，打印
instrument->nloops。

结论是缺失行不必然是 worker 没启动，也可能是该节点在该
worker 上没有完成 loop。

### 15.2. Sort worker 内存差异大

现象是一个 worker 使用 Disk，另一个 worker
使用 Memory。源码上 show_sort_info()
会按 worker 输出 Sort Method 和 Sort
Space。

断在 show_sort_info() 的 worker
分支，确认 worker sort
instrumentation 是否来自
ExecSortRetrieveInstrumentation()。

结论是并行倾斜可能表现为局部 spill。总 Sort
节点信息不足以解释尾部延迟。

### 15.3. Hash Batches 只看总值不够

现象是 Hash 节点显示 batches 增加，但不知道是哪类
worker 或 batch 触发。源码上
HashRetrieveInstrumentation
汇总的是节点投影后的信息。

结合 Worker N rows、Hash
Batches、Peak Memory Usage 和
Parallel Hash phase 断点一起看。

结论是 EXPLAIN 的 hash 明细是诊断入口，不是完整的
growth 历史。

### 15.4. BUFFERS 明细和总 buffer 不一致

现象是节点级 worker buffer 与查询总 buffer
对不上。源码上节点 instrumentation 和
query-level BufferUsage 是不同汇总层。

看 ExplainNode() 中
show_buffer_usage() 的 worker
分支，再看 ExecParallelFinish() 中
InstrAccumParallelQuery()。

结论是节点级 buffer 解释节点，query-level
buffer 解释整个 worker 执行增量。

### 15.5. TEXT 输出被解析错

现象是工具按缩进解析 Worker N
行，版本或选项变化后字段错位。源码上
ExplainOpenWorker / CloseWorker
针对 TEXT 和非 TEXT 格式走不同输出路径。

使用 FORMAT JSON，检查 Worker
Number、Actual Rows、Actual Loops
等结构化字段。

结论是自动化诊断应优先 JSON。TEXT 更适合人工快速阅读。

### 15.6. 最小断点路线

如果课堂时间有限，只保留下面这条断点路线。它覆盖本节主问题中的状态创建、推进、观测和
cleanup 边界。

```text
ExplainNode()
  -> create ExplainWorkersState
  -> print main node actual rows / loops
  -> ExplainOpenWorker(n)
  -> print worker-specific properties
  -> ExplainCloseWorker(n)
  -> node-specific worker sections
  -> ExplainFlushWorkersState()
```

跑实验时不要一次打开所有断点。先确认第一处状态变化，再沿返回值或
owner
指针走到下一处。这样更容易判断偏差是来自调用层次、共享状态、worker
明细，还是 EXPLAIN 格式化。

### 15.7. worker EXPLAIN 复盘表

这张复盘表用于课后把现场现象、源码断点和最终归因放到同一页。它不是新的主题，只是把本节主问题落到一次可重复的诊断记录。

| 复盘项 | 记录方式 | 判断边界 |
| --- | --- | --- |
| 通用 worker 行 | 记录 Worker Number、Actual Rows、Actual Loops | 先确认 nloops 是否大于零 |
| 节点特化指标 | 记录 Sort Method、Hash Batches、Peak Memory、Disk Usage | 区分通用 instrumentation 与节点私有统计 |
| buffer 层次 | 分别记录节点 BUFFERS 和 query-level BufferUsage | 区分节点级与查询级汇总 |
| 输出格式 | 保留 FORMAT JSON 原始结果 | 避免 TEXT 缩进解析误差 |
| 倾斜判断 | 比较各 worker rows、loops、spill 和 buffer | 先看分布，再看总和 |

通用 worker 行：记录 Worker
Number、Actual Rows、Actual Loops
这一项的判断边界是：先确认 nloops 是否大于零 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

节点特化指标：记录 Sort Method、Hash
Batches、Peak Memory、Disk Usage
这一项的判断边界是：区分通用 instrumentation
与节点私有统计 记录时要写明 SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

buffer 层次：分别记录节点 BUFFERS 和
query-level BufferUsage
这一项的判断边界是：区分节点级与查询级汇总 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

输出格式：保留 FORMAT JSON 原始结果
这一项的判断边界是：避免 TEXT 缩进解析误差 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

倾斜判断：比较各 worker rows、loops、spill 和
buffer 这一项的判断边界是：先看分布，再看总和 记录时要写明
SQL、GUC、EXPLAIN
选项和断点位置，避免只留下一个耗时数字。

## 16. 本节小结

per-worker EXPLAIN 输出是 leader
格式化出来的 worker 明细，不是 worker 直接打印。

总计 instrumentation 和
worker_instrument 明细回答不同问题。

rows、loops、sort/hash/buffer
指标要按节点语义一起解释。

并行效率问题常常表现为 worker 间差异，而不是总耗时本身。

可迁移规律是：先看分布，再看总和；总和只能说明问题存在，分布才提示问题在哪。

本节最后沉淀的系统规律是：观测入口必须和状态生命周期对齐。入口选错，看到的不是不完整，就是语义已经过期。

下一节会继续看 Parallel Hash 和 Parallel
Append 这类节点自身共享运行状态如何影响观测。
