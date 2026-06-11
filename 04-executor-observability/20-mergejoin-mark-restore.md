# PostgreSQL Merge Join 的有序流推进与 mark / restore

## 课程定位

前置知识：理解排序输入、join qual、outer
join、TupleTableSlot、Material/Sort 的 mark/restore 能力。

本节唯一主问题：

```text
Merge Join 如何在两个有序输入流之间推进匹配组，并用 mark / restore 处理一对多匹配而不把两边全部物化？
```

核心矛盾：有序流可以低内存地合并匹配，但 SQL join 允许重复 key 和 outer join
填充；执行器必须在顺序前进、重复回看 inner 组、NULL 语义和 mark/restore
成本之间取平衡。

学完后应能判断：Merge Join 慢是排序成本、mark/restore
代价、匹配组过大、outer join fill，还是输入本身已经有序但估计错误。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 19 节分析了 Hash Join 如何用 build/probe 和 batch spill
管理大状态。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：Merge Join 如何在两个有序输入流之间推进匹配组，并用 mark /
restore 处理一对多匹配而不把两边全部物化？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 21 节会进入 Agg，观察 group boundary 和 transition state
如何推进。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Merge Join 假设 outer 和 inner 都按 merge key 排序。
状态机比较当前 outer key 和 inner key，小的一侧前进，相等时进入匹配组输出。
遇到新的 outer key 可能需要把 inner 恢复到标记位置，重新扫描同一匹配组。
outer/full/right join 还要用 matched 标记决定 NULL 填充输出。
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
| 1 | src/backend/executor/nodeMergejoin.c | `ExecMergeJoin()` 状态机、`MJCompare()`、`MarkInnerTuple`、outer/full join fill。 |
| 2 | src/include/nodes/execnodes.h | `MergeJoinState` 字段：`mj_JoinState`、`mj_MarkedTupleSlot`、matched flags、fill slots。 |
| 3 | src/backend/executor/execAmi.c | `ExecMarkPos()`、`ExecRestrPos()` 和 `ExecReScanMergeJoin()` 的调度边界。 |
| 4 | src/backend/executor/nodeSort.c | Sort 节点可为 Merge Join 提供有序输入和 mark/restore 能力。 |
| 5 | src/backend/executor/nodeMaterial.c | 当 inner 不支持必要回看时，Materialize 可提供 tuplestore 支撑。 |
| 6 | src/backend/executor/execExpr.c | merge clauses、joinqual 和 projection 的表达式初始化。 |
| 7 | src/include/executor/nodeMergejoin.h | Merge Join executor 入口声明。 |
| 8 | src/include/executor/executor.h | mark/restore 和 rescan 公共接口。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `mj_JoinState`

Merge Join 的核心状态机。

owner / 生命周期：由 `ExecInitMergeJoin()`
初始化，`ExecMergeJoin()` 在每次调用中推进。

诊断边界：它表示下一步该取 outer、取 inner、比较、输出还是填充。

单独看到 `mj_JoinState` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `mj_OuterTupleSlot` / `mj_InnerTupleSlot`

当前两个输入流的候选 tuple。

owner / 生命周期：由子节点 slot 提供，状态机保存指针语义。

诊断边界：slot 内容随子节点推进变化，不应跨状态误用。

单独看到 `mj_OuterTupleSlot` / `mj_InnerTupleSlot`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `mj_MarkedTupleSlot`

保存当前 inner 匹配组的起点。

owner / 生命周期：在发现 key 相等时通过 `MarkInnerTuple` 或
mark/restore 记录。

诊断边界：没有它，一对多匹配会漏掉后续 outer tuple 与同一 inner 组的组合。

单独看到 `mj_MarkedTupleSlot` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `mj_SkipMarkRestore`

优化路径，说明可以跳过真实 mark/restore。

owner / 生命周期：planner 或初始化阶段根据输入能力和语义设置。

诊断边界：它不是所有 Merge Join 都安全启用。

单独看到 `mj_SkipMarkRestore` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `mj_MatchedOuter` / `mj_MatchedInner`

outer/full/right join 判断是否需要 NULL 填充。

owner / 生命周期：输出匹配行时置位，进入 fill 状态时消费。

诊断边界：match 标记只属于当前 key 或当前 tuple，不是全局计数。

单独看到 `mj_MatchedOuter` / `mj_MatchedInner`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `mj_OuterEContext` / `mj_InnerEContext`

分别评估 merge key 和 nullability。

owner / 生命周期：MJEvalOuterValues、MJEvalInnerValues 设置。

诊断边界：NULL key 在普通 merge join 中可能不可匹配，影响提前结束和 fill。

单独看到 `mj_OuterEContext` / `mj_InnerEContext`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitMergeJoin()` -> 初始化两个子计划、merge clause、marked slot、null fill slot 和状态机。
初始化 outer -> 从 outerPlan 取第一行并评估 merge key。
初始化 inner -> 从 innerPlan 取第一行并评估 merge key。
`MJCompare()` -> 比较当前 outer key 与 inner key。
输出匹配组 -> `EXEC_MJ_JOINTUPLES` 组合当前 outer/inner，检查 joinqual 和 ps.qual。
推进 inner -> 同一 outer tuple 继续扫 inner 匹配组。
恢复 inner 标记 -> 新的 outer key 仍等于已标记 inner key 时，恢复 inner 到组起点。
outer/full join fill -> 某侧 tuple 未匹配时，用 null slot 生成填充行。
```

### 5.1 `ExecInitMergeJoin()`

初始化两个子计划、merge clause、marked slot、null fill slot
和状态机。

状态变化：`mj_JoinState` 从初始化 outer 开始。

正确性或资源边界：初始化会根据 join type 决定 fill 行能力。

调试时可以在 `ExecInitMergeJoin()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 初始化 outer

从 outerPlan 取第一行并评估 merge key。

状态变化：不可匹配或 EOF 会进入 skip/fill/end 分支。

正确性或资源边界：NULL key 语义在这里已经影响状态机。

调试时可以在 初始化 outer
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 初始化 inner

从 innerPlan 取第一行并评估 merge key。

状态变化：同样区分 matchable、nonmatchable、end。

正确性或资源边界：两边都可匹配后才能比较。

调试时可以在 初始化 inner
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 `MJCompare()`

比较当前 outer key 与 inner key。

状态变化：小的一侧前进，相等则标记 inner 组并进入 jointuples。

正确性或资源边界：这一步依赖排序不变量。

调试时可以在 `MJCompare()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 输出匹配组

`EXEC_MJ_JOINTUPLES` 组合当前 outer/inner，检查 joinqual 和
ps.qual。

状态变化：成功后返回投影 slot。

正确性或资源边界：返回一行不代表组结束，下一次调用会继续同一组。

调试时可以在 输出匹配组
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 推进 inner

同一 outer tuple 继续扫 inner 匹配组。

状态变化：inner key 仍相等则继续输出，否则切换 outer。

正确性或资源边界：一对多匹配成本在这里显现。

调试时可以在 推进 inner
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 恢复 inner 标记

新的 outer key 仍等于已标记 inner key 时，恢复 inner 到组起点。

状态变化：重复利用同一 inner 组。

正确性或资源边界：需要 mark/restore 或已物化输入支持。

调试时可以在 恢复 inner 标记
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 outer/full join fill

某侧 tuple 未匹配时，用 null slot 生成填充行。

状态变化：matched flags 被消费。

正确性或资源边界：fill 是 join 语义，不是错误路径。

调试时可以在 outer/full join fill
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

ExecInitMergeJoin 创建 MergeJoinState、独立
ExprContext、marked slot 和必要 null slots。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

状态机持有当前 outer/inner slot、marked slot 和 matched
flags。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

ExecReScanMergeJoin 清空 marked slot、重置状态机和 matched
flags，再 rescan 两个子计划。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndMergeJoin 关闭 outer/inner 子节点，slot 和表达式内存随
executor context 清理。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

sort/material 临时文件、slot 内容和 expression memory 依赖子节点
end、ResourceOwner 和 context 兜底。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| 排序不变量 | 两侧必须按 merge key 有序，否则“小的一侧前进”会漏匹配。 |
| mark/restore | 重复 key 需要回到 inner 组起点，保证多对多组合完整。 |
| NULL 语义 | merge key 为 NULL 的 tuple 可能不可匹配，outer join 仍可能需要 fill。 |
| joinqual 分层 | merge clause 只决定候选组，额外 joinqual 仍需执行。 |
| matched flags | outer/full join 的 NULL 填充依赖匹配标记。 |
| rescan | 重扫必须清理标记槽和状态机，不能从旧组继续。 |

排序不变量 这一层保证的是：两侧必须按 merge key 有序，否则“小的一侧前进”会漏匹配。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

mark/restore 这一层保证的是：重复 key 需要回到 inner
组起点，保证多对多组合完整。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

NULL 语义 这一层保证的是：merge key 为 NULL 的 tuple
可能不可匹配，outer join 仍可能需要 fill。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

joinqual 分层 这一层保证的是：merge clause 只决定候选组，额外 joinqual
仍需执行。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

matched flags 这一层保证的是：outer/full join 的 NULL
填充依赖匹配标记。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

rescan 这一层保证的是：重扫必须清理标记槽和状态机，不能从旧组继续。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### 输入未排序

计划中需要 Sort 或利用已有索引顺序。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### inner 不支持 mark/restore

可能插入 Materialize 或使用 Sort 的能力。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 匹配组巨大

Merge Join 仍保持流式，但一个 key 组内会产生大量组合和回看。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### NULL key

普通等值 merge join 中会进入 nonmatchable 或 fill 分支。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### const false join

初始化 quals 时可能发现永不匹配，状态机走简化路径。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### 排序成本

如果两侧需要 Sort，startup 和 temp I/O 可能主导。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### 匹配组成本

重复 key 会让 inner 组被反复扫描。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### mark/restore 成本

Material 或 Sort 支撑回看会增加内存和临时文件压力。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### 首行延迟

两侧排序完成前通常没有输出。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### CPU compare

merge key 比较函数在状态机推进中高频执行。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN Merge Join

看子节点是否 Sort/Material，actual rows 与 loops 是否符合匹配组大小。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Sort Method

Sort 节点的 Memory/Disk 解释 Merge Join 的启动成本。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Rows Removed by Join Filter

说明 merge key 匹配后还有 joinqual 淘汰。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Buffers/Temp

区分排序/物化的临时 I/O 与基表访问。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb 状态机

在 `ExecMergeJoin()`、`MJCompare()` 和 `ExecRestrPos()`
断点看状态跳转。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### enable_* 对照

关闭 hashjoin 或 mergejoin 对比计划，理解 optimizer tradeoff。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

计划选择 Merge Join，子节点有 Sort 或
Materialize，实际耗时集中在排序启动或匹配组推进。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步确认两侧输入是否已经有序。如果需要显式 Sort，先看 Sort
Method、Memory、Disk 和 temp I/O。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到
`MergeJoinState.mj_JoinState`、`mj_MarkedTupleSlot` 和
matched flags，看重复 key 组如何让 inner 组被回看。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再从 `MJCompare()` 和 mark/restore
路径判断当前成本来自比较推进、回看能力，还是 outer/full join 的填充语义。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果 Sort 很快而 Merge Join 慢，问题通常不在排序，而在重复 key
组、joinqual 或输出基数膨胀。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecMergeJoin
break MJCompare
break ExecMaterialMarkPos
break ExecMaterialRestrPos
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

本节的具体落点是：把 Sort/Material、mj_MarkedTupleSlot 和重复 key
匹配组连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

对 Merge Join 来说，复盘还要明确当前成本是来自排序启动、匹配组回看，还是外连接填充；这三者在源码中落到不同状态。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：Merge Join 不等于只适合小表，它适合已有有序输入或排序成本可接受的流式合并。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：有 Sort 不代表 Merge Join 错误，关键看排序是否比 hash
build/spill 更划算。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：mark/restore 不是随机访问整张表，而是处理当前匹配组回看。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：Join Filter 淘汰不破坏排序不变量，但会改变实际输出行数。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：outer join fill 是状态机正常分支。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：已有索引顺序

```sql
CREATE INDEX ON a(k);
CREATE INDEX ON b(k);
SET enable_hashjoin = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM a JOIN b USING(k);
```

观察重点：观察是否利用 index order 避免显式 Sort。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：重复 key 组

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM a JOIN b USING(k) WHERE a.k BETWEEN 1 AND 10;
```

观察重点：构造大量重复 key，观察 actual rows 如何膨胀。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：mark/restore 支撑

```sql
SET enable_material = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM a LEFT JOIN b USING(k);
```

观察重点：观察 Materialize 是否出现及关闭后的计划变化。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecMergeJoin
break MJCompare
break ExecMaterialMarkPos
break ExecMaterialRestrPos
```

观察重点：记录状态机和 mark/restore 的关系。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 为什么 Merge Join 需要状态机而不是简单 while 循环？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. mark/restore 和 Materialize 的边界如何影响内存与临时文件？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. 重复 key 很多时，Hash Join 与 Merge Join 的成本差异在哪里？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. outer join fill 为什么必须与 matched flags 绑定？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 如何从 EXPLAIN 判断排序成本还是 join 本身成本更大？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

Merge Join 的核心是两个有序流的状态机。

mark/restore 让重复 key 的匹配组可以被重新扫描。

outer/full join 依赖 matched flags 和 null slot 生成填充行。

诊断时要看 Sort/Material、匹配组大小、Rows Removed by Join
Filter 和 temp I/O。

可迁移规律：流式算法为了避免全量物化，通常会把复杂性集中在局部回看和状态机边界。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
