# PostgreSQL Agg 的 group、hash 与 transition state

## 课程定位

前置知识：理解表达式执行、MemoryContext、GROUP BY、聚合函数的
transition/final function，以及 work_mem。

本节唯一主问题：

```text
Agg 节点如何在 sorted aggregation、hash aggregation 和 mixed strategy 中维护每个 group 的 transition state，并在内存压力下保持正确输出？
```

核心矛盾：聚合想把大量输入压缩成少量 group 输出，但 transition value 可能是
pass-by-reference 对象、可能需要 DISTINCT/ORDER BY 排序，也可能因
group 数过多而超出内存。

学完后应能判断：聚合慢是输入未排序、group 数过多、transition
函数昂贵、DISTINCT/ORDER BY 子排序、hashagg spill，还是 final
projection 成本。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 20 节看到 Merge Join 用有序流和 mark/restore 管理匹配组。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：Agg 节点如何在 sorted aggregation、hash
aggregation 和 mixed strategy 中维护每个 group 的
transition state，并在内存压力下保持正确输出？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 22 节会把焦点转到 Sort、Incremental Sort 和 Material
这类阻塞节点。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecAgg 不只是把行数数一遍；它围绕 group boundary 和 transition state 推进。
sorted/group aggregation 依赖输入顺序，在 group 切换时 finalize 并输出。
hash aggregation 以 group key 查找 hash entry，在每个 entry 的 pergroup state 中累积 transition value。
DISTINCT/ORDER BY aggregate 和 hashagg spill 会引入 tuplesort、tape 和 batch refill。
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
| 1 | src/backend/executor/nodeAgg.c | `ExecAgg()`、`advance_transition_function()`、`agg_fill_hash_table()`、`hash_agg_enter_spill_mode()`。 |
| 2 | src/include/executor/nodeAgg.h | `AggStatePerTransData`、`AggStatePerAggData`、`AggStatePerGroupData`、`AggStatePerHashData`。 |
| 3 | src/include/nodes/execnodes.h | `AggState` 中 phases、pertrans、pergroups、hash_pergroup 和 aggcontexts。 |
| 4 | src/backend/executor/execExpr.c | `ExecBuildAggTrans()` 生成聚合 transition 表达式。 |
| 5 | src/backend/executor/execExprInterp.c | transition opcode、`ExecAggInitGroup()`、`ExecAggCopyTransValue()`。 |
| 6 | src/backend/utils/sort/tuplesort.c | DISTINCT/ORDER BY aggregate 的内部排序和 statistics。 |
| 7 | src/backend/executor/execAmi.c | `ExecReScanAgg()` 调度边界。 |
| 8 | src/backend/executor/execParallel.c | partial/final aggregate 的并行 instrumentation 入口。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `AggState`

聚合节点的总运行时状态。

owner / 生命周期：ExecInitAgg 创建，ExecAgg 推进。

诊断边界：它同时服务 sorted、hashed、mixed 和 grouping
sets，不能只按一种策略解释。

单独看到 `AggState` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `AggStatePerTrans`

一个 transition 函数和输入表达式的执行期描述。

owner / 生命周期：ExecInitAgg 根据 Aggref 构造。

诊断边界：多个 aggregate 可能共享 transition state。

单独看到 `AggStatePerTrans` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `AggStatePerGroup`

某个 group 下某个 trans 的当前 value、isnull 等工作状态。

owner / 生命周期：sorted 聚合按当前 group 持有，hash 聚合放在 hash
entry 附加空间。

诊断边界：transition value 的内存归属决定是否能跨 tuple 保留。

单独看到 `AggStatePerGroup` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `aggcontexts`

聚合 transition value 的生命周期边界。

owner / 生命周期：group 切换、rescan 或 hash table reset
时按策略重置。

诊断边界：不能把 per-tuple 临时值错挂到 aggcontext。

单独看到 `aggcontexts` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `perhash` / hash table

HashAgg 的 group lookup 状态。

owner / 生命周期：build_hash_tables
创建，lookup_hash_entries 更新。

诊断边界：group 数多时会进入 spill 逻辑。

单独看到 `perhash` / hash table 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `sortstates`

DISTINCT 或 ORDER BY aggregate 的内部排序状态。

owner / 生命周期：initialize_aggregate
创建，finalize_aggregates 消费。

诊断边界：它是聚合内部阻塞点，不等于上层 Sort 节点。

单独看到 `sortstates` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitAgg()` -> 分析 Agg plan，建立 phases、pertrans、peragg、pergroups 和表达式。
`ExecAgg()` -> 根据策略进入 direct 或 hash retrieval。
`fetch_input_tuple()` -> 从 outerPlan 拉取输入 slot。
`advance_transition_function()` -> 调用 transition function 更新 transValue。
sorted group boundary -> 输入 key 变化时 finalize 当前 group。
`lookup_hash_entries()` -> HashAgg 用 group key 找到或创建 hash entry。
`hash_agg_enter_spill_mode()` -> 超过内存边界后把部分 tuple 或 batch 写入 tape。
`finalize_aggregate()` / `project_aggregates()` -> 执行 final function 并投影输出 slot。
```

### 5.1 `ExecInitAgg()`

分析 Agg plan，建立 phases、pertrans、peragg、pergroups
和表达式。

状态变化：聚合策略被映射到运行时状态。

正确性或资源边界：初始化不读取输入。

调试时可以在 `ExecInitAgg()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 `ExecAgg()`

根据策略进入 direct 或 hash retrieval。

状态变化：第一次调用可能先消费大量输入。

正确性或资源边界：聚合通常是阻塞或半阻塞节点。

调试时可以在 `ExecAgg()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 `fetch_input_tuple()`

从 outerPlan 拉取输入 slot。

状态变化：输入 tuple 进入聚合表达式上下文。

正确性或资源边界：slot 生命周期仍属于子节点。

调试时可以在 `fetch_input_tuple()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 `advance_transition_function()`

调用 transition function 更新 transValue。

状态变化：pergroup state 被改变。

正确性或资源边界：pass-by-ref 值必须复制到正确 context。

调试时可以在 `advance_transition_function()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 sorted group boundary

输入 key 变化时 finalize 当前 group。

状态变化：旧 group 输出后初始化新 group。

正确性或资源边界：排序不变量决定正确性。

调试时可以在 sorted group boundary
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 `lookup_hash_entries()`

HashAgg 用 group key 找到或创建 hash entry。

状态变化：transition state 存在 hash entry 附加空间。

正确性或资源边界：内存压力会被 hash_agg_check_limits 看到。

调试时可以在 `lookup_hash_entries()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 `hash_agg_enter_spill_mode()`

超过内存边界后把部分 tuple 或 batch 写入 tape。

状态变化：聚合从单轮内存路径进入多批次路径。

正确性或资源边界：spill 保持 group 完整性而增加 I/O。

调试时可以在 `hash_agg_enter_spill_mode()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 `finalize_aggregate()` / `project_aggregates()`

执行 final function 并投影输出 slot。

状态变化：transition value 变成用户可见结果。

正确性或资源边界：final function 不能破坏后续 group 状态。

调试时可以在 `finalize_aggregate()` /
`project_aggregates()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

AggState 和 pertrans/peragg arrays 在 executor context
中创建。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

transition value 存在 aggcontext 或 hash entry
附加空间，per-tuple 表达式内存短期 reset。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

ExecReScanAgg 重置 group state、hash table、spill state
和 sortstates。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndAgg 结束 tuplesort、hash spill state 和表达式上下文。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

聚合函数 ERROR 时依赖 memory context 和 tuplesort/logical
tape cleanup 兜底。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| transition/final 分层 | 每行只推进 transition，输出时才 finalize。 |
| 内存上下文 | transition value 必须在 group 生命周期内有效。 |
| 排序不变量 | GroupAggregate 依赖输入按 group key 排序。 |
| hash key equality | HashAgg 必须用 equality 和 hash function 共同定义 group。 |
| DISTINCT/ORDER BY | 每个 aggregate 的内部排序改变输入给 transition 的顺序和去重边界。 |
| spill 分批 | batch refill 不能把同一 group 的 partial state 错分到不同最终 group。 |

transition/final 分层 这一层保证的是：每行只推进 transition，输出时才
finalize。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

内存上下文 这一层保证的是：transition value 必须在 group 生命周期内有效。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

排序不变量 这一层保证的是：GroupAggregate 依赖输入按 group key 排序。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

hash key equality 这一层保证的是：HashAgg 必须用 equality 和
hash function 共同定义 group。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

DISTINCT/ORDER BY 这一层保证的是：每个 aggregate 的内部排序改变输入给
transition 的顺序和去重边界。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

spill 分批 这一层保证的是：batch refill 不能把同一 group 的 partial
state 错分到不同最终 group。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### 空输入

无 GROUP BY 的聚合仍可能输出一行初始/final 结果。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### strict transition

NULL 输入可能跳过 transition 调用。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### DISTINCT 过大

内部 tuplesort 可能落盘。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### HashAgg 内存不足

进入 spill mode 并多轮读取 batch。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### rescan

必须清空旧 transition state，避免上一轮结果污染。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### input rows

transition 调用次数随输入行数增长。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### group cardinality

HashAgg memory 和 GroupAgg 输出次数随 group 数增长。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### transition width

pass-by-reference state 越大，复制和 context retention 越重。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### internal sort

DISTINCT/ORDER BY aggregate 引入额外 work_mem 和 temp
I/O。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### spill

HashAgg batch/tape 让 CPU 聚合变成 I/O 路径。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN Aggregate 类型

区分 Aggregate、GroupAggregate、HashAggregate 和 Mixed。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### HashAggregate 指标

看 Batches、Memory Usage、Disk Usage 或 temp I/O。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Sort 节点

GroupAggregate 前的 Sort 可能是主要成本。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### track_io_timing

解释 internal sort 或 hashagg spill 的 I/O 时间。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### pg_backend_memory_contexts

观察 AggContext、TupleSort 或 executor context 峰值。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb

断点
`advance_transition_function()`、`hash_agg_enter_spill_mode()`
看 state 更新。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

聚合计划显示 HashAggregate 或 GroupAggregate 很慢，输出 group
数不多，但输入行很多或 temp I/O 明显。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步区分策略：HashAggregate 看
Batches/Memory/Disk，GroupAggregate 看前置 Sort 和 group
boundary。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到 `AggStatePerGroup`、`AggStatePerTrans` 和
aggcontext，确认每个输入 tuple 如何推进 transition value。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再检查 DISTINCT/ORDER BY aggregate 是否有内部
tuplesort，避免只盯着上层 Sort 节点。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果 group 数不多但 CPU 很高，可能是 transition function、final
function 或表达式求值昂贵，不一定是聚合框架问题。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecAgg
break advance_transition_function
break hash_agg_enter_spill_mode
break finalize_aggregate
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

本节的具体落点是：把输入行、pergroup transition value 和 hashagg
spill 或 group boundary 连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

对 Agg 来说，复盘还要明确 transition value 的生命周期；同一个慢现象可能来自 per-tuple 表达式、aggcontext retention 或 hash batch。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：Agg 不等于只在最后执行一次，transition 在每个输入 tuple 上推进。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：HashAggregate 的内存不是无限增长，超过边界会 spill。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：GroupAggregate 快不快高度依赖输入是否已排序。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：DISTINCT aggregate 的排序可能隐藏在 Agg 内部，不一定表现为上层
Sort 节点。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：transition function 可以看到 AggState
context，因此扩展聚合要谨慎处理内存。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：HashAgg spill

```sql
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT k, count(*), sum(v) FROM big GROUP BY k;
SET work_mem = '128MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT k, count(*), sum(v) FROM big GROUP BY k;
```

观察重点：观察 Batches、Disk 或 temp I/O。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：GroupAgg 排序成本

```sql
SET enable_hashagg = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT k, count(*) FROM big GROUP BY k;
```

观察重点：看 Sort 是否成为主要启动成本。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：DISTINCT aggregate

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(DISTINCT v) FROM big GROUP BY k;
```

观察重点：观察内部排序导致的时间和 temp 使用。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecAgg
break advance_transition_function
break hash_agg_enter_spill_mode
```

观察重点：观察 pergroup transValue 如何变化。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 为什么 transition value 需要自己的内存生命周期？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. HashAgg spill 和 Hash Join spill 的相同点与差异在哪里？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. GroupAggregate 什么时候比 HashAggregate 更稳定？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. DISTINCT aggregate 为什么不能简单共用上层 Sort？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 聚合函数扩展应如何避免泄漏到长生命周期 context？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

Agg 的核心状态是 per-group transition value。

策略不同，transition state 的存放和推进方式不同。

hashagg spill 是 group 数和内存边界冲突下的 fallback。

诊断时要把输入 rows、group cardinality、transition
width、sort/spill 分开看。

可迁移规律：把多行压缩成状态的节点，正确性取决于状态生命周期和边界事件。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
