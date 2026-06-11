# PostgreSQL Nested Loop 的参数化内侧扫描

## 课程定位

前置知识：已经理解 PlanState tree、ExprContext 的 outer/inner
tuple、PARAM_EXEC 和普通 scan node 的 rescan 边界。

本节唯一主问题：

```text
Nested Loop 如何用每一个 outer tuple 改写 inner scan 的运行时参数，并在 tuple-by-tuple 协议下保持 join 语义正确？
```

核心矛盾：Nested Loop
能把外层值直接变成内侧索引点查，延迟低、状态少；但这也意味着内侧计划会被反复
rescan，参数、join qual、outer join 填充和 loops 成本都被放大。

学完后应能判断：一个 Nested Loop
是因为外层很小而合理，还是因为错误估计导致内侧扫描被放大成性能问题。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 17 节看到 BitmapHeapScan 把候选位置和 heap 语义拆成两阶段。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：Nested Loop 如何用每一个 outer tuple 改写 inner
scan 的运行时参数，并在 tuple-by-tuple 协议下保持 join 语义正确？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 19 节会进入 Hash Join，比较另一种先构建内侧状态再探测的 join 形态。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecNestLoop 先向 outerPlan 拉一个 tuple，把其中的 nestParams 写入 PARAM_EXEC slot。
随后它调用 ExecReScan(innerPlan)，让内侧计划用新的参数重新定位候选 tuple。
内侧每返回一个 tuple，Nested Loop 先检查 joinqual，再检查 ps.qual，最后投影或继续拉取。
如果 inner 用 IndexScan、BitmapHeapScan 或 Memoize，参数化边界会直接改变它们的 scan key、bitmap 或 cache key。
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
| 1 | src/backend/executor/nodeNestloop.c | `ExecNestLoop()` 的状态机、`nl_NeedNewOuter`、outer tuple 取值、inner rescan 和 outer join fill。 |
| 2 | src/include/nodes/execnodes.h | `NestLoopState`、`JoinState`、`ExprContext` 和 `ParamExecData` 的状态边界。 |
| 3 | src/backend/executor/execAmi.c | `ExecReScan()` 根据节点类型分派到具体 rescan 函数，并处理 `chgParam`。 |
| 4 | src/backend/executor/execExprInterp.c | 表达式执行时读取 `PARAM_EXEC`，把外层 tuple 值带入内侧 qual。 |
| 5 | src/backend/executor/nodeIndexscan.c | 参数化 inner IndexScan 在 rescan 时更新 runtime scan keys。 |
| 6 | src/backend/executor/nodeMemoize.c | Memoize 可位于 Nested Loop 内侧，用参数值缓存重复查找结果。 |
| 7 | src/backend/executor/execProcnode.c | PlanState 初始化和 `ExecProcNode()` 调度说明 join 节点如何拉取子节点。 |
| 8 | src/include/executor/executor.h | `ExecReScan()`、`ExecProcNode()`、`ResetExprContext()` 等公共执行器边界。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `NestLoopState.nl_NeedNewOuter`

决定下一次 `ExecNestLoop()` 是否必须先拉取新的 outer tuple。

owner / 生命周期：由 `ExecInitNestLoop()` 初始化，由 inner
结束、匹配成功或 outer join fill 路径更新。

诊断边界：它是状态机位，不是成本估算字段。

单独看到 `NestLoopState.nl_NeedNewOuter`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `ExprContext.ecxt_outertuple`

当前 outer tuple 的可见句柄，join qual 和投影会读取它。

owner / 生命周期：Nested Loop 在拉到 outer tuple 后设置，随下一轮
outer tuple 更新。

诊断边界：不能把 slot 内容保存到更长生命周期，除非显式 materialize。

单独看到 `ExprContext.ecxt_outertuple`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `PARAM_EXEC` slot

把 outer tuple 中的参数值传给 inner plan。

owner / 生命周期：NestLoop 从 `nestParams` 中取值写入 EState 的
paramExecVals。

诊断边界：参数值变化后 inner 的旧 scan key 或 cache 语义失效。

单独看到 `PARAM_EXEC` slot 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `innerPlan->chgParam`

告诉 inner 子树哪些参数发生变化。

owner / 生命周期：Nested Loop 写入参数后推动
`ExecReScan(innerPlan)`。

诊断边界：没有这个边界，参数化 IndexScan 可能继续用旧 key。

单独看到 `innerPlan->chgParam` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `joinqual` 与 `ps.qual`

joinqual 决定匹配关系，ps.qual 是 join 之后的额外过滤。

owner / 生命周期：`JoinState` 和 `PlanState` 分别持有。

诊断边界：outer join 是否需要填充 NULL 行，取决于 joinqual
的匹配语义，不是所有过滤都等价。

单独看到 `joinqual` 与 `ps.qual` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `nl_MatchedOuter`

记录当前 outer tuple 是否已经匹配过 inner tuple。

owner / 生命周期：semi/anti/outer join 路径会消费这个状态。

诊断边界：它只对当前 outer tuple 有意义，不能跨 outer tuple 解释。

单独看到 `nl_MatchedOuter` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitNestLoop()` -> 初始化 outer/inner PlanState、join qual、projection 和结果 slot。
进入 `ExecNestLoop()` -> 如果 `nl_NeedNewOuter` 为 true，先从 outerPlan 拉取一个 tuple。
写入 `nestParams` -> 从 outer tuple 取值并写入 `PARAM_EXEC`。
`ExecReScan(innerPlan)` -> 通知 inner 子树按新参数重新开始。
拉取 inner tuple -> Nested Loop 反复调用 `ExecProcNode(innerPlan)`。
检查 `joinqual` -> 先判断两个 tuple 是否形成 join match。
检查 `ps.qual` 并投影 -> 通过 joinqual 的 pair 还要经过其它 qual 和 projection。
outer join fill / end -> inner 无匹配时，根据 join type 可能用 NULL inner slot 输出填充行。
```

### 5.1 `ExecInitNestLoop()`

初始化 outer/inner PlanState、join qual、projection 和结果
slot。

状态变化：`nl_NeedNewOuter` 初始为 true。

正确性或资源边界：初始化阶段不执行 join，只准备状态。

调试时可以在 `ExecInitNestLoop()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 进入 `ExecNestLoop()`

如果 `nl_NeedNewOuter` 为 true，先从 outerPlan 拉取一个 tuple。

状态变化：没有 outer tuple 时整个 join 结束。

正确性或资源边界：这是 tuple-by-tuple 拉取协议在 join 节点上的入口。

调试时可以在 进入 `ExecNestLoop()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 写入 `nestParams`

从 outer tuple 取值并写入 `PARAM_EXEC`。

状态变化：inner 子树的参数语义被更新。

正确性或资源边界：参数写入必须早于 inner rescan。

调试时可以在 写入 `nestParams`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 `ExecReScan(innerPlan)`

通知 inner 子树按新参数重新开始。

状态变化：IndexScan 更新 scan key，Bitmap 子树重建
bitmap，Memoize 可能查询 cache。

正确性或资源边界：rescan 是语义边界，不是简单把指针拨回开头。

调试时可以在 `ExecReScan(innerPlan)`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 拉取 inner tuple

Nested Loop 反复调用 `ExecProcNode(innerPlan)`。

状态变化：每个 inner slot 与当前 outer slot 组成候选 join pair。

正确性或资源边界：inner 结束会推动 `nl_NeedNewOuter` 回到 true。

调试时可以在 拉取 inner tuple
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 检查 `joinqual`

先判断两个 tuple 是否形成 join match。

状态变化：成功后 `nl_MatchedOuter` 可能被置位。

正确性或资源边界：outer/anti/semi join 的行为在这里分叉。

调试时可以在 检查 `joinqual`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 检查 `ps.qual` 并投影

通过 joinqual 的 pair 还要经过其它 qual 和 projection。

状态变化：返回 slot 给上层，或者继续找下一个 pair。

正确性或资源边界：Rows Removed by Join Filter 和 Filter 的解释不同。

调试时可以在 检查 `ps.qual` 并投影
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 outer join fill / end

inner 无匹配时，根据 join type 可能用 NULL inner slot 输出填充行。

状态变化：当前 outer tuple 结束后状态转向下一个 outer tuple。

正确性或资源边界：fill 语义来自 join 类型，不来自 inner scan 是否为空。

调试时可以在 outer join fill / end
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

ExecInitNestLoop 递归初始化两个子计划，并在 executor context 下建立
join qual、projection 和结果 slot。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

NestLoopState 持有当前 outer slot、inner slot 和少量布尔状态；实际
tuple 内容仍由子节点 slot 管理。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

ExecReScanNestLoop 重置 `nl_NeedNewOuter` 和匹配状态，并按
`chgParam` 让子树各自处理 rescan。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndNestLoop 关闭 inner 和 outer 子节点，释放表达式上下文和结果
slot 的生命周期由上层 executor 统一收束。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

ERROR 发生在 join qual、投影函数或 inner scan 中时，普通 C
局部路径不会完整返回，资源依赖 MemoryContext 和 ResourceOwner 回收。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| 参数传播 | PARAM_EXEC 必须先写入，再设置 inner chgParam，再 rescan inner。 |
| slot 生命周期 | outer slot 在 inner 扫描期间必须保持语义有效，表达式不能跨轮保存裸 slot 指针。 |
| join qual 分层 | joinqual 与 other qual 对 outer join 填充语义不同，不能为了简化全部合并。 |
| MVCC | inner scan 仍按 executor snapshot 判断可见性，外层参数不改变 snapshot 边界。 |
| rescan 语义 | 不同 inner 节点的 rescan 成本不同，IndexScan、BitmapHeapScan、Material、Memoize 各自维护状态。 |
| 短路语义 | semi/anti join 找到匹配后可能立刻切换 outer tuple，这是正确性和成本共同结果。 |

参数传播 这一层保证的是：PARAM_EXEC 必须先写入，再设置 inner chgParam，再
rescan inner。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

slot 生命周期 这一层保证的是：outer slot 在 inner
扫描期间必须保持语义有效，表达式不能跨轮保存裸 slot 指针。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

join qual 分层 这一层保证的是：joinqual 与 other qual 对 outer
join 填充语义不同，不能为了简化全部合并。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

MVCC 这一层保证的是：inner scan 仍按 executor snapshot
判断可见性，外层参数不改变 snapshot 边界。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

rescan 语义 这一层保证的是：不同 inner 节点的 rescan
成本不同，IndexScan、BitmapHeapScan、Material、Memoize
各自维护状态。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

短路语义 这一层保证的是：semi/anti join 找到匹配后可能立刻切换 outer
tuple，这是正确性和成本共同结果。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### inner 为空

当前 outer tuple 可能被丢弃，也可能根据 left join 输出 NULL 填充行。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 参数为 NULL

inner index qual 可能直接无匹配，也可能按 SQL 三值逻辑继续检查 joinqual。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 重复参数值

Memoize 可以把重复 inner 查找转成缓存命中，但命中率依赖数据分布。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 估计错误

优化器误判 outer rows 过小，会让 inner loops 在执行期爆炸。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### inner 不支持廉价 rescan

计划器可能插入 Materialize，或执行期每轮都付出昂贵重扫成本。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### outer rows

Nested Loop 成本通常近似随 outer actual rows 乘以内侧平均成本扩张。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### inner startup

每个 outer tuple 都可能触发 inner rescan，startup cost 会被
loops 放大。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### index parameterization

内侧索引点查很快时 Nested Loop 可以极低延迟地产生首行。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### qual CPU

joinqual 和 ps.qual 在每个候选 pair 上执行，表达式复杂度会直接进入热点。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### cache locality

重复参数和热索引页可能让实际成本低于冷缓存估计。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN loops

看 inner 节点 `loops` 是否接近 outer actual
rows，这是参数化内侧扫描的第一证据。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Index Cond 中的外层引用

内侧 Index Scan 的条件如果含 outer 别名，说明 PARAM_EXEC 正在驱动
scan key。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Rows Removed by Join Filter

区分 joinqual 淘汰和普通 filter 淘汰，尤其在 outer join 中很重要。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Buffers 放大

inner 节点 buffers 乘以 loops 后可能解释大部分 I/O。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Memoize 指标

如果计划中有 Memoize，看 hits/misses/evictions，判断重复参数是否被利用。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb 断点

在 `ExecNestLoop()`、`ExecReScan()`、`ExecIndexScan()`
上断点，观察每个 outer tuple 如何改变 inner。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

一条计划里 Nested Loop 的内侧 Index Scan 单次很快，但 loops
极高，总耗时集中在内侧重复执行。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步先把 outer actual rows 和 inner loops 对齐。如果两者接近，说明外层
tuple 正在驱动内侧参数化扫描。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到 `NestLoopState.nl_NeedNewOuter`、`PARAM_EXEC` 和
`innerPlan->chgParam`，看每个 outer tuple 如何让 inner scan
key 失效并重建。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再进入 `ExecReScan(innerPlan)`，区分内侧是
IndexScan、BitmapHeapScan、Material 还是 Memoize，因为它们的
rescan 成本完全不同。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果 inner loops 不高而总耗时仍大，问题可能在 joinqual 表达式、outer
scan、锁等待或上层节点，不要把所有慢都归给 Nested Loop。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecNestLoop
break ExecReScan
break ExecIndexScan
break ExecScan
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

本节的具体落点是：把 inner loops、PARAM_EXEC 更新和 inner rescan
成本连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：Nested Loop 不是天然慢；外层很小、内侧索引选择性高时，它是最低延迟 join。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：inner loops 多不一定错误，关键看每轮 inner 的实际成本和输出行数。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：Materialize 或 Memoize 出现在内侧不是装饰，它们改变 rescan
成本模型。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：把 joinqual 和 filter 混读，会误判 outer join 的 NULL
填充行为。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：参数化内侧扫描不是 planner 的文字游戏，它在 executor 中落实为
PARAM_EXEC 和 chgParam。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：索引点查型 Nested Loop

```sql
SET enable_hashjoin = off;
SET enable_mergejoin = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM o JOIN i ON i.k = o.k WHERE o.id < 100;
```

观察重点：观察 inner Index Scan 的 loops 与 outer rows 是否接近。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：估计错误放大

```sql
ANALYZE o;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM o JOIN i ON i.k = o.k WHERE o.skew_col = 1;
```

观察重点：如果 outer actual rows 远大于 estimate，观察 inner
buffers 和 loops 如何放大。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：Memoize 对照

```sql
SET enable_memoize = on;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM o JOIN i ON i.k = o.repeated_k;
SET enable_memoize = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM o JOIN i ON i.k = o.repeated_k;
```

观察重点：比较 repeated key 情况下 inner 重复查找成本。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecNestLoop
break ExecReScan
break ExecIndexScan
```

观察重点：记录每次 outer tuple 改变 PARAM_EXEC 后 inner scan key
的变化。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 什么时候 Nested Loop 的首行延迟优势比总成本更重要？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. 为什么 PARAM_EXEC 必须属于 EState，而不是直接存在 NestLoopState
中？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. inner rescan 的成本如何随
IndexScan、BitmapHeapScan、Material、Memoize 变化？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. EXPLAIN 中 inner loops 很大时，哪些证据能说明计划仍然合理？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 如果要给扩展节点支持参数化执行，必须实现哪些 rescan 语义？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

Nested Loop 的核心是 outer tuple 驱动 inner rescan。

PARAM_EXEC、chgParam 和 ExecReScan 是这个驱动关系的执行期实现。

joinqual、ps.qual 和 outer join fill 必须分层理解。

诊断 Nested Loop 时先看 outer actual rows、inner
loops、inner buffers 和参数化条件。

可迁移规律：低延迟的局部点查模型一旦被错误基数放大，就会变成最明显的重复成本。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
