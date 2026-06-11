# PostgreSQL Sort / Incremental Sort / Material 的阻塞节点特征

## 课程定位

前置知识：理解 tuple-by-tuple
执行协议、work_mem、TupleTableSlot、tuplestore、tuplesort 和
rescan。

本节唯一主问题：

```text
Sort、Incremental Sort 和 Material 为什么会打断流水线，它们分别用什么运行时状态换取排序、回看或重扫能力？
```

核心矛盾：执行器默认按需拉一个 tuple 就返回一个 tuple，但排序和可重扫要求节点先保存一批
tuple；这会带来首行延迟、内存占用和临时文件，同时给上层提供有序或可回看的语义。

学完后应能判断：一个阻塞节点是必要的语义边界，还是因为计划形态导致了可避免的
materialization、sort spill 或重复读取。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 21 节看到 Agg 如何把输入累积到 transition state。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：Sort、Incremental Sort 和 Material
为什么会打断流水线，它们分别用什么运行时状态换取排序、回看或重扫能力？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 23 节会看 Limit、Result、ProjectSet 这类轻节点如何以很少状态改变控制流。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Sort 首次执行时先消耗整个 outer subplan，把 tuple 放入 tuplesort，再 performsort。
Incremental Sort 利用输入的 presorted prefix，把全局排序拆成多个前缀组内排序。
Material 不是排序，它把子计划输出缓存到 tuplestore，以支持 rescan、mark/restore 或重复消费。
三者都把 tuple-by-tuple 流水线改造成“先存，再读”的状态机。
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
| 1 | src/backend/executor/nodeSort.c | `ExecSort()`、`ExecInitSort()`、`ExecReScanSort()` 和 sort instrumentation。 |
| 2 | src/backend/executor/nodeIncrementalSort.c | `ExecIncrementalSort()` 的 presorted group、fullsort/prefixsort 状态。 |
| 3 | src/backend/executor/nodeMaterial.c | `ExecMaterial()`、tuplestore 缓存、mark/restore 和 rescan 行为。 |
| 4 | src/backend/utils/sort/tuplesort.c | tuplesort 通用状态机、work_mem、external sort 和统计。 |
| 5 | src/backend/utils/sort/tuplesortvariants.c | heap/datum/index sort 变体。 |
| 6 | src/backend/utils/sort/tuplestore.c | tuplestore begin/put/get/rescan/end 和 spill。 |
| 7 | src/include/nodes/execnodes.h | `SortState`、`IncrementalSortState`、`MaterialState` 字段。 |
| 8 | src/backend/executor/execAmi.c | mark/restore、rescan 和 `ExecMaterializesOutput()` 边界。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `SortState.tuplesortstate`

Sort 节点私有 tuplesort 指针。

owner / 生命周期：首次 ExecSort 时创建，ExecEndSort 释放。

诊断边界：为空不代表没有排序需求，可能只是还未执行。

单独看到 `SortState.tuplesortstate` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `sort_Done`

说明输入已经被完整送入 tuplesort 并完成排序。

owner / 生命周期：ExecSort 第一次调用中置位。

诊断边界：它体现阻塞节点的阶段切换。

单独看到 `sort_Done` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `IncrementalSortState.execution_status`

Incremental Sort 当前处于加载 fullsort、加载 prefixsort、读取哪个
sort 的状态。

owner / 生命周期：ExecIncrementalSort 在前缀组之间推进。

诊断边界：它解释为什么部分输入有序时仍可能出现排序状态。

单独看到 `IncrementalSortState.execution_status`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `group_pivot`

保存当前 presorted prefix group 的代表 tuple。

owner / 生命周期：Incremental Sort 用它判断组边界。

诊断边界：pivot slot 的生命周期必须覆盖组内排序。

单独看到 `group_pivot` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `MaterialState.tuplestorestate`

Material 节点缓存子计划输出的 tuplestore。

owner / 生命周期：按需创建，可能内存或磁盘。

诊断边界：它不改变 tuple 内容，只改变可重复读取能力。

单独看到 `MaterialState.tuplestorestate`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `eflags` / randomAccess

上层是否要求 rewind、mark/restore 或 backward scan。

owner / 生命周期：ExecInitMaterial/Sort 根据 eflags 选择能力。

诊断边界：能力越强，缓存和内存/磁盘成本越高。

单独看到 `eflags` / randomAccess 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitSort()` -> 初始化 SortState、结果 slot 和子计划。
`ExecSort()` 首次调用 -> 创建 tuplesort，然后循环拉取 outer tuple。
`tuplesort_performsort()` -> 输入结束后完成排序，可能写临时文件。
Sort 输出 -> 反复调用 `tuplesort_gettupleslot()` 返回排序结果。
Incremental Sort 加载前缀组 -> 读取已按 prefix 有序的输入，按 group_pivot 划分组。
`ExecMaterial()` 首次读取 -> 从 tuplestore 读不到时拉取 outer tuple 并放入 tuplestore。
Material rescan -> 如果 tuplestore 已存在，`tuplestore_rescan()` 把读指针回到开头。
End / cleanup -> Sort 结束 tuplesort，Material 结束 tuplestore，Incremental Sort 清理两个 sort states。
```

### 5.1 `ExecInitSort()`

初始化 SortState、结果 slot 和子计划。

状态变化：不立即创建 tuplesort。

正确性或资源边界：延迟到首次执行以匹配 executor demand model。

调试时可以在 `ExecInitSort()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 `ExecSort()` 首次调用

创建 tuplesort，然后循环拉取 outer tuple。

状态变化：每个 tuple 被 `tuplesort_puttupleslot()` 写入。

正确性或资源边界：这一步阻塞上层输出。

调试时可以在 `ExecSort()` 首次调用
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 `tuplesort_performsort()`

输入结束后完成排序，可能写临时文件。

状态变化：`sort_Done` 置为 true。

正确性或资源边界：首行输出从这里之后才可能发生。

调试时可以在 `tuplesort_performsort()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 Sort 输出

反复调用 `tuplesort_gettupleslot()` 返回排序结果。

状态变化：slot 由 tuplesort 填充。

正确性或资源边界：输出阶段又回到 tuple-by-tuple。

调试时可以在 Sort 输出
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 Incremental Sort 加载前缀组

读取已按 prefix 有序的输入，按 group_pivot 划分组。

状态变化：组内 tuple 进入 fullsort 或 prefixsort。

正确性或资源边界：减少内存峰值但增加状态复杂度。

调试时可以在 Incremental Sort 加载前缀组
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 `ExecMaterial()` 首次读取

从 tuplestore 读不到时拉取 outer tuple 并放入 tuplestore。

状态变化：上层得到一行，同时缓存一行。

正确性或资源边界：Material 可半阻塞，不一定先读完整输入。

调试时可以在 `ExecMaterial()` 首次读取
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 Material rescan

如果 tuplestore 已存在，`tuplestore_rescan()` 把读指针回到开头。

状态变化：不必重新执行子计划。

正确性或资源边界：这是 Material 的核心价值。

调试时可以在 Material rescan
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 End / cleanup

Sort 结束 tuplesort，Material 结束 tuplestore，Incremental
Sort 清理两个 sort states。

状态变化：临时文件和内存释放。

正确性或资源边界：ERROR 路径依赖资源 owner。

调试时可以在 End / cleanup
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

Sort 和 Material 的外层 PlanState 初始化较轻，真正大状态延迟到执行期。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

tuplesort/tuplestore 持有 materialized tuple
副本，不应依赖子节点 slot 后续有效。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

Sort 可复用已排序结果或重建，Material 可直接 rescan tuplestore，取决于
chgParam 和 eflags。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndSort/Material/IncrementalSort 必须结束私有
sort/store 状态。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

临时文件和内存通过 tuplesort/tuplestore cleanup、ResourceOwner
和 context reset 收束。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| 排序语义 | Sort 必须在输出前看到所有影响顺序的 tuple。 |
| 前缀不变量 | Incremental Sort 只能利用输入已经按 prefix 排序的事实。 |
| materialize 语义 | Material 保存的是 tuple 值副本，不是子节点 slot 指针。 |
| mark/restore | 需要回看能力的上层节点依赖 Sort/Material 实现 mark/restore。 |
| work_mem 边界 | 内存不足时 external sort/store 不能改变输出语义。 |
| chgParam | 参数变化后缓存结果必须失效。 |

排序语义 这一层保证的是：Sort 必须在输出前看到所有影响顺序的 tuple。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

前缀不变量 这一层保证的是：Incremental Sort 只能利用输入已经按 prefix
排序的事实。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

materialize 语义 这一层保证的是：Material 保存的是 tuple 值副本，不是子节点
slot 指针。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

mark/restore 这一层保证的是：需要回看能力的上层节点依赖 Sort/Material 实现
mark/restore。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

work_mem 边界 这一层保证的是：内存不足时 external sort/store
不能改变输出语义。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

chgParam 这一层保证的是：参数变化后缓存结果必须失效。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### Sort spill

tuplesort 超过 work_mem 后使用外部排序。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### bounded sort

上层 LIMIT 可以设置 tuple bound，减少需要排序的数量。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### Incremental Sort 退化

prefix group 很大时，组内排序接近普通 Sort。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### Material spill

tuplestore 超过 work_mem 后写临时文件。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### rescan 参数变化

旧缓存不可复用，需要重新执行子计划。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### startup latency

Sort 必须先消耗输入，首行延迟高。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### memory

tuplesort/tuplestore 受 work_mem 限制。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### temp I/O

外部 sort/store 产生临时读写。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### CPU compare

排序比较函数可能成为热点。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### pipeline break

阻塞节点会影响上游/下游并行度和响应时间。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN Sort Method

看 quicksort、external merge、top-N
heapsort、Memory/Disk。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Incremental Sort 指标

看 Presorted Key、Full-sort Groups 和 Sort Method。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Materialize 节点

注意它的 loops 和下层 loops 是否被削减。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Temp Read/Write

确认 spill 是否发生。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### trace_sort

开发环境可用 `trace_sort` 日志观察 tuplesort 阶段。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb

断点
`ExecSort()`、`tuplesort_performsort()`、`ExecMaterial()`、`tuplestore_rescan()`。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

计划中出现 Sort、Incremental Sort 或 Materialize，首行延迟高，或者
temp 文件突然增多。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步确认这个节点提供的语义：Sort 提供顺序，Incremental Sort
利用前缀顺序，Material 提供重扫或回看。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到 `sort_Done`、`tuplesortstate`、`execution_status` 或
`tuplestorestate`，看节点处于加载阶段还是读取阶段。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再沿着 `ExecReScanSort()` 或 `tuplestore_rescan()`
判断缓存能否复用，还是因为 chgParam 必须重建。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果没有上层节点使用顺序、mark/restore 或 rescan
能力，就需要重新审视计划形态，而不是只调 work_mem。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecSort
break tuplesort_performsort
break ExecIncrementalSort
break ExecMaterial
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

本节的具体落点是：把首行延迟、tuplesort/tuplestore 状态和上层
rescan/顺序需求连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

对阻塞节点来说，复盘还要明确它打断流水线后换来了什么能力；如果能力没有被上层使用，才值得回头质疑计划形态。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：Materialize 不是复制计划树，它缓存 tuple 输出。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：Sort 节点慢不一定是比较函数慢，可能是 temp I/O。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：Incremental Sort 不是免费排序，prefix group 很大时仍然昂贵。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：work_mem 是每个节点/操作的预算近似，不是整条 SQL 的全局上限。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：阻塞节点有时是正确性需求，不是优化器多此一举。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：external sort

```sql
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big ORDER BY wide_text;
SET work_mem = '128MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big ORDER BY wide_text;
```

观察重点：观察 Sort Method 与 Disk。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：top-N sort

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big ORDER BY score DESC LIMIT 100;
```

观察重点：观察 bounded sort 或 top-N heapsort。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：Material rescan

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM small s JOIN big b ON b.k=s.k ORDER BY b.k;
```

观察重点：观察 Materialize 是否降低 inner 子树重复执行。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecSort
break tuplesort_performsort
break ExecMaterial
break tuplestore_rescan
```

观察重点：记录阻塞阶段和输出阶段。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 什么时候阻塞节点的首行延迟不可接受？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. 为什么 Material 能支持 rescan，但不提供排序语义？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. work_mem 调大对并发系统有什么风险？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. Incremental Sort 的收益为什么依赖数据分布？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 如何从 EXPLAIN 证明 Materialize 是语义需要而不是多余？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

Sort、Incremental Sort、Material 都通过保存 tuple 改变流水线。

Sort 提供顺序，Incremental Sort 利用前缀顺序，Material 提供重复读取。

work_mem 和 temp I/O 是诊断这些节点的第一资源边界。

rescan 和 mark/restore 解释很多看似多余的缓存节点。

可迁移规律：任何打断流式执行的节点，都应该用它提供的语义能力来证明成本值得。

判断阻塞节点是否合理时，可以先问两个问题。

第一，它提供的是输出顺序、可回看能力，还是重复消费能力。

第二，这个能力有没有被上层 Merge Join、Limit、WindowAgg、rescan 或 mark/restore 真正使用。

如果答案都不清楚，先不要急着调 work_mem，而要回到计划形态确认这个节点为什么存在。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
