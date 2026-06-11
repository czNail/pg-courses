# PostgreSQL Limit / Result / ProjectSet 的轻节点控制流

## 课程定位

前置知识：理解 ExecProcNode、表达式上下文、targetlist
projection、SRF 和 Limit/Offset 语义。

本节唯一主问题：

```text
Limit、Result 和 ProjectSet 为什么看起来状态很少，却能显著改变 tuple 流的停止、过滤、投影和扩展行为？
```

核心矛盾：轻节点应尽量不保存大量数据、不打断流水线；但 SQL 需要
LIMIT/OFFSET、one-time qual、常量结果、targetlist SRF
展开等控制流语义，这些语义必须落到明确状态上。

学完后应能判断：一个计划中的轻节点是成本可忽略的控制节点，还是因为 loops、SRF 或 LIMIT
WITH TIES 让控制流成本被放大。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 22 节分析了会保存大量 tuple 的阻塞节点。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：Limit、Result 和 ProjectSet
为什么看起来状态很少，却能显著改变 tuple 流的停止、过滤、投影和扩展行为？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 24 节会进入 ModifyTable，观察 DML 如何把 tuple 流变成表修改、副作用和
RETURNING。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Limit 通过状态机决定何时跳过、返回、停止或处理 WITH TIES。
Result 可以没有 outerPlan，用 one-time qual 和 projection 产生常量或门控输出。
ProjectSet 把一个输入 tuple 展开成多行 SRF 输出，并用 pending_srf_tuples 记录未完成状态。
三者都不应被当作“没有成本”，它们只是把成本集中在控制流和表达式边界。
```

这句话背后有三层含义。

第一层是执行协议：上层仍然通过 `ExecProcNode()` 一次要一个 slot。

第二层是状态边界：节点内部可以缓存、重扫、阻塞、展开或副作用化 tuple，但必须给上层稳定语义。

第三层是诊断边界：EXPLAIN 看到的是状态推进后的事实，不能直接等同于优化器估算或单个函数耗时。

本节的 tension 可以压缩成：

```text
保持统一 tuple 拉取协议和低 hot path 成本
  vs
节点必须保存足够状态，才能实现本节 SQL 语义、异常 cleanup 和可观测性
```

读源码时要不断把这组 tension 放回当前节点。否则很容易把某个 helper 函数误读成独立主题。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | src/backend/executor/nodeLimit.c | `ExecLimit()` 状态机、OFFSET/COUNT 计算、WITH TIES 和 rescan。 |
| 2 | src/backend/executor/nodeResult.c | `ExecResult()`、one-time qual、constant projection 和 mark/restore。 |
| 3 | src/backend/executor/nodeProjectSet.c | `ExecProjectSet()`、`ExecProjectSRF()`、pending SRF tuples 和 argcontext。 |
| 4 | src/backend/executor/execSRF.c | `ExecInitFunctionResultSet()`、`ExecMakeFunctionResultSet()` 管理 SRF 调用状态。 |
| 5 | src/include/nodes/execnodes.h | `LimitState`、`ResultState`、`ProjectSetState` 的真实字段。 |
| 6 | src/backend/executor/execExpr.c | targetlist、one-time qual 和表达式状态初始化。 |
| 7 | src/backend/executor/execTuples.c | 结果 slot 和 TupleDesc 初始化。 |
| 8 | src/backend/executor/execAmi.c | 轻节点 rescan 调度。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `LimitState.lstate`

Limit 的状态机：initial、in window、window end、subplan eof
等。

owner / 生命周期：ExecLimit 每次调用推进。

诊断边界：不能只看 offset/count 推断是否还能输出。

单独看到 `LimitState.lstate` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `offset` / `count` / `position`

描述当前跳过和返回窗口。

owner / 生命周期：由 limitOffset/limitCount 表达式计算后保存。

诊断边界：参数化 LIMIT 重扫时必须重新计算。

单独看到 `offset` / `count` / `position`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `LimitState.subSlot` / `last_slot`

当前子计划返回行和 WITH TIES 比较所需的上一行。

owner / 生命周期：ExecLimit 持有。

诊断边界：WITH TIES 让 Limit 需要额外 equality 判断。

单独看到 `LimitState.subSlot` / `last_slot`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `ResultState.resconstantqual`

one-time qual 的表达式状态。

owner / 生命周期：ExecResult 在第一次需要时检查。

诊断边界：失败后 rs_done 可以让节点不再拉取子计划。

单独看到 `ResultState.resconstantqual`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `ProjectSetState.pending_srf_tuples`

当前输入 tuple 的 SRF 是否还有剩余输出。

owner / 生命周期：ExecProjectSRF 设置，ExecProjectSet 消费。

诊断边界：它让一个输入 tuple 跨多次 ExecProcNode 调用。

单独看到 `ProjectSetState.pending_srf_tuples`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `ProjectSetState.argcontext`

SRF 参数求值的内存上下文。

owner / 生命周期：ExecInitProjectSet 创建，rescan/reset 时清理。

诊断边界：SRF 多行输出期间参数必须保持有效。

单独看到 `ProjectSetState.argcontext`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitLimit()` -> 初始化 LimitState、limit 表达式、结果类型和子计划。
`ExecLimit()` 计算边界 -> 第一次执行或 rescan 后计算 offset/count/noCount。
跳过 OFFSET -> 反复拉取子计划直到 position 达到 offset。
返回窗口 tuple -> 在 LIMIT 窗口内返回 subSlot。
WITH TIES -> 到达 count 后继续比较排序 key 是否与 last_slot 相等。
`ExecResult()` -> 检查 one-time qual，再投影 outerPlan 或常量 targetlist。
`ExecProjectSet()` -> 若 pending_srf_tuples 为 true，继续为同一输入 tuple 生成 SRF 行。
`ExecMakeFunctionResultSet()` -> 维护单个 SRF 的调用状态和 ExprDoneCond。
```

### 5.1 `ExecInitLimit()`

初始化 LimitState、limit 表达式、结果类型和子计划。

状态变化：`lstate` 从 LIMIT_INITIAL 开始。

正确性或资源边界：初始化不提前跳过 tuple。

调试时可以在 `ExecInitLimit()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 `ExecLimit()` 计算边界

第一次执行或 rescan 后计算 offset/count/noCount。

状态变化：position 清零，状态进入跳过或窗口。

正确性或资源边界：LIMIT 参数表达式错误会在这里暴露。

调试时可以在 `ExecLimit()` 计算边界
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 跳过 OFFSET

反复拉取子计划直到 position 达到 offset。

状态变化：被跳过 tuple 不输出。

正确性或资源边界：子计划仍然真实执行，OFFSET 不是免费。

调试时可以在 跳过 OFFSET
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 返回窗口 tuple

在 LIMIT 窗口内返回 subSlot。

状态变化：position 增加。

正确性或资源边界：上层看到普通 slot，但控制流已经被 Limit 截断。

调试时可以在 返回窗口 tuple
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 WITH TIES

到达 count 后继续比较排序 key 是否与 last_slot 相等。

状态变化：相等则继续输出，不相等才停止。

正确性或资源边界：WITH TIES 依赖 ORDER BY 语义。

调试时可以在 WITH TIES
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 `ExecResult()`

检查 one-time qual，再投影 outerPlan 或常量 targetlist。

状态变化：rs_done 防止重复输出常量行。

正确性或资源边界：Result 可作为 gating 节点。

调试时可以在 `ExecResult()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 `ExecProjectSet()`

若 pending_srf_tuples 为 true，继续为同一输入 tuple 生成 SRF 行。

状态变化：否则拉取下一输入 tuple。

正确性或资源边界：一个输入 tuple 可以产生零到多输出行。

调试时可以在 `ExecProjectSet()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 `ExecMakeFunctionResultSet()`

维护单个 SRF 的调用状态和 ExprDoneCond。

状态变化：ProjectSet 汇总多个 SRF 的完成情况。

正确性或资源边界：SRF 语义是轻节点成本放大的主要来源。

调试时可以在 `ExecMakeFunctionResultSet()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

三个节点都在 ExecInit* 中创建少量 backend-local 状态和结果 slot。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

Limit 持有窗口位置，Result 持有 one-time qual 状态，ProjectSet
持有 SRF 调用状态和 argcontext。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

ExecReScanLimit 重新计算参数；Result 清理 rs_done；ProjectSet
清理 pending SRF 和 argcontext。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEnd* 关闭子节点，表达式内存随 executor context 删除。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

表达式或 SRF 抛 ERROR 时，argcontext 和 per-tuple context
必须通过 reset/delete 收束。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| LIMIT 窗口 | OFFSET tuple 必须真实消费，不能只改计数。 |
| WITH TIES | 必须保存 last_slot 并按 equality 判断额外输出。 |
| one-time qual | Result 的 gate 语义不能被普通 filter 完全替代。 |
| SRF 展开 | ProjectSet 必须在同一输入 tuple 上连续输出剩余 SRF 行。 |
| 内存上下文 | SRF 参数和返回值生命周期必须跨多次调用保持。 |
| rescan | 参数化 LIMIT 或 SRF 状态不能污染下一轮执行。 |

LIMIT 窗口 这一层保证的是：OFFSET tuple 必须真实消费，不能只改计数。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

WITH TIES 这一层保证的是：必须保存 last_slot 并按 equality 判断额外输出。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

one-time qual 这一层保证的是：Result 的 gate 语义不能被普通 filter
完全替代。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

SRF 展开 这一层保证的是：ProjectSet 必须在同一输入 tuple 上连续输出剩余 SRF
行。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

内存上下文 这一层保证的是：SRF 参数和返回值生命周期必须跨多次调用保持。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

rescan 这一层保证的是：参数化 LIMIT 或 SRF 状态不能污染下一轮执行。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### LIMIT 0

Limit 可以不拉取子计划或快速结束，取决于参数计算路径。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### OFFSET 很大

仍需消费并丢弃大量 tuple。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### one-time qual false

Result 可以阻止整个子树继续输出。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### SRF 返回空集

ProjectSet 对当前输入 tuple 不输出行。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### SRF 多次调用

pending_srf_tuples 让上层看到多行，loops 和 rows 被放大。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### OFFSET cost

OFFSET 越大，丢弃成本越大。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### early stop

LIMIT 可以显著减少下游或上游继续执行。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### WITH TIES

可能输出超过 count 的行，并需要额外 equality 比较。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### SRF expansion

输出行数可能远大于输入行数。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### expression CPU

轻节点主要成本通常来自表达式和函数调用。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN Limit

看实际 rows 是否等于 count，WITH TIES 时可能超过。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### 子节点 actual rows

OFFSET 会让子节点 rows 大于 Limit 输出 rows。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Result one-time filter

EXPLAIN 可能显示 One-Time Filter。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### ProjectSet rows

比较输入 rows 和 ProjectSet 输出 rows，判断 SRF 扩张倍数。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Function timing

SRF 很慢时可结合 track_functions 或 profiler。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb

断点 `ExecLimit()`、`ExecResult()`、`ExecProjectSRF()`。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

计划节点看起来只是 Limit、Result 或 ProjectSet，但实际 rows、loops
或函数调用次数与直觉不符。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步区分它改变的是停止边界、one-time gate，还是一行到多行的基数扩张。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到 `LimitState.lstate`、`ResultState.rs_done` 或
`ProjectSetState.pending_srf_tuples`，看控制流是否跨多次调用延续。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再检查 OFFSET、WITH TIES、SRF argcontext 和
`ExecMakeFunctionResultSet()`，确认轻节点是否放大了子节点消费。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果 Limit 输出很少但子节点 rows 很多，问题不是 Limit 慢，而是
OFFSET、排序或上游路径仍要真实执行。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecLimit
break ExecResult
break ExecProjectSRF
break ExecMakeFunctionResultSet
```

断点只用于确认状态推进顺序，不建议在高频循环里长时间单步。更有效的做法是记录入口参数、关键状态字段和返回
slot 是否为空。

完成一次闭环后，再决定是否需要扩大到 buffer、lock、temporary
file、statistics 或 planner estimate 层面。

#### 课堂复盘口径

复盘时把结论压成三句话。

第一句说明看到的 runtime 现象，例如
rows、loops、temp、WAL、Buffers、trigger time、wait event
或函数栈。

第二句说明源码中的主状态如何推进，必须点名一个字段、一个 owner 或一个状态机分支。

第三句说明这个状态为什么会产生观察到的现象，并指出它属于正常路径、fallback、rescan、cleanup
还是异常路径。

本节的具体落点是：把 lstate、rs_done、pending_srf_tuples
和输出基数变化连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

对轻节点来说，复盘还要明确它是否改变了基数；Limit 和 Result 多半改变控制流，ProjectSet 则经常直接改变输出行数。

特别是 ProjectSet，要把单个输入 tuple 的多次返回和上层看到的 actual rows 分开记录。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：LIMIT 不是总能让查询快，OFFSET 仍会消耗前面的 tuple。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：Result 不是“无意义节点”，它可承载 one-time qual 和常量投影。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：ProjectSet 不是普通 Projection，SRF 会改变基数。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：LIMIT WITH TIES 可能返回多于 count 的行。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：轻节点没有大状态，不代表没有控制流成本。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：OFFSET 成本

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big ORDER BY k LIMIT 10 OFFSET 0;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big ORDER BY k LIMIT 10 OFFSET 100000;
```

观察重点：观察子节点 rows 和时间。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：One-Time Filter

```sql
EXPLAIN (ANALYZE) SELECT * FROM big WHERE false AND expensive_pred(v);
```

观察重点：观察 Result 或 gating 行为。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：ProjectSet 扩张

```sql
EXPLAIN (ANALYZE) SELECT id, generate_series(1, n) FROM s;
```

观察重点：比较输入行数与输出行数。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecLimit
break ExecResult
break ExecProjectSRF
break ExecMakeFunctionResultSet
```

观察重点：记录状态字段变化。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 为什么 OFFSET 不能直接跳到第 N 行？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. Result 节点和普通 Filter 的语义边界在哪里？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. SRF 为什么需要 argcontext，而不是只用 per-tuple context？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. LIMIT 如何影响 Sort 的 bounded optimization？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 哪些轻节点会改变 cardinality，哪些只改变控制流？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

Limit、Result、ProjectSet 都是轻状态控制节点。

Limit 改变停止边界，Result 改变 gate/projection，ProjectSet
改变一行到多行的输出节奏。

轻节点的成本主要来自子节点消费、表达式和 SRF 扩张。

诊断时不要只看节点本身耗时，要看它如何改变上游 rows 和 loops。

可迁移规律：小状态节点也可能改变整棵执行树的控制流。

判断轻节点是否真的“轻”，不能只看它有没有大块私有内存。

Limit 可能减少下游输出，却仍然消费 OFFSET 前的大量 tuple。

Result 可能一次性关闭整棵子树，也可能只是普通投影边界。

ProjectSet 可能把少量输入扩展成大量输出，让上层 rows、loops 和函数调用次数全部改变。

所以轻节点诊断的关键是控制流和基数变化，而不是结构体大小。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
