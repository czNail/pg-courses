# PostgreSQL Hash Join 的 build / probe 与 batch spill

## 课程定位

前置知识：理解 join qual、TupleTableSlot、work_mem、临时文件和
executor 节点按需返回 tuple 的协议。

本节唯一主问题：

```text
Hash Join 如何把 inner side 预先构造成 hash table，并在内存不足时退化为多 batch spill 而不破坏 join 语义？
```

核心矛盾：Hash Join 想用一次 build 换取大量 outer probe 的 O(1)
查找，但 hash table 的真实大小依赖 inner rows、row width、skew 和
work_mem；内存不能无限增长时，只能把一部分 tuple 分批写到临时文件再重放。

学完后应能判断：Hash Join 慢是 build side 太大、probe side
太大、hash qual 选择性差、batch spill 过多，还是 outer join
unmatched 扫描导致的额外阶段。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 18 节看到 Nested Loop 把 outer tuple 逐个变成 inner 参数。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：Hash Join 如何把 inner side 预先构造成 hash
table，并在内存不足时退化为多 batch spill 而不破坏 join 语义？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 20 节会比较 Merge Join：它不建 hash
table，而是依赖有序流、mark/restore 和匹配组推进。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecHashJoin 先进入 HJ_BUILD_HASHTABLE 状态，驱动 inner Hash 节点构建 HashJoinTable。
outer tuple 逐个计算 hash value，定位 bucket，并在 bucket chain 内检查 hash clauses 和 joinqual。
如果 hash table 超过内存预算，nodeHash.c 会增加 batch，把部分 inner/outer tuple 保存到批次文件。
每个 batch 的重新装载和 probe 保持同一 join 语义，只是把一次内存 join 拆成多轮磁盘路径。
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
| 1 | src/backend/executor/nodeHashjoin.c | `ExecHashJoinImpl()` 的状态机、build/probe/new batch、outer join unmatched 路径。 |
| 2 | src/backend/executor/nodeHash.c | `ExecHashTableCreate()`、`ExecHashTableInsert()`、batch 文件、memory limit 和 parallel hash 细节。 |
| 3 | src/include/executor/hashjoin.h | `HashJoinTableData`、batch、bucket、skew 和 parallel state 的结构定义。 |
| 4 | src/include/nodes/execnodes.h | `HashJoinState` 字段说明 executor 层 join 状态与 hash table 指针如何相邻。 |
| 5 | src/include/executor/nodeHash.h | hash table 操作的 executor 内部接口。 |
| 6 | src/backend/executor/execAmi.c | `ExecReScanHashJoin()` 调度说明 hash table 何时保留、重置或销毁。 |
| 7 | src/backend/storage/file/buffile.c | hash join spill 背后的临时文件抽象。 |
| 8 | src/backend/executor/execParallel.c | 并行 hash join instrumentation 和 DSM 初始化入口。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `HashJoinState.hj_JoinState`

驱动 build、probe、new batch 和 unmatched scan 的状态机。

owner / 生命周期：由 `ExecInitHashJoin()`
初始化，`ExecHashJoinImpl()` 在执行中推进。

诊断边界：状态值只能结合当前 batch、outer slot 和 hash table 解释。

单独看到 `HashJoinState.hj_JoinState`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `HashJoinTable`

inner side 的 hash buckets、batch 信息、memory limit 和
skew bucket 的总 owner。

owner / 生命周期：由 `ExecHashTableCreate()` 创建，由
HashJoinState 持有。

诊断边界：它是 executor-private 状态，不是共享 catalog 或 planner
估算对象。

单独看到 `HashJoinTable` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `nbatch` / batch files

把超出内存的 tuple 按 hash value 分到多个批次。

owner / 生命周期：nodeHash.c 在 build 或 probe 中决定是否增长
batch 并写临时文件。

诊断边界：batch 变多意味着成本从 CPU/cache 转向临时 I/O 和重复扫描。

单独看到 `nbatch` / batch files 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `hj_CurTuple` / bucket scan

记录当前 outer tuple 在 bucket chain 中的扫描位置。

owner / 生命周期：probe 阶段每个 outer tuple 更新。

诊断边界：joinqual 失败不等于 hash qual 失败，需要分层看。

单独看到 `hj_CurTuple` / bucket scan 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### skew hash state

为常见 inner MCV 提供更快的 bucket。

owner / 生命周期：根据统计和内存预算建立。

诊断边界：它优化热点值，不改变普通 batch 的正确性。

单独看到 skew hash state 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### unmatched flags

outer/full/right join 需要知道 inner tuple 是否已匹配。

owner / 生命周期：hash tuple 中的 match 标记或状态在 probe 后被扫描。

诊断边界：没有 unmatched 阶段，外连接语义会丢失。

单独看到 unmatched flags 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitHashJoin()` -> 初始化 HashJoinState、表达式、slot 和 hash clauses。
`HJ_BUILD_HASHTABLE` -> 驱动 inner Hash 节点把 build side 读完。
`ExecHashTableInsert()` -> 每个 inner tuple 计算 hash value 并插入 bucket 或写入 batch 文件。
读取 outer tuple -> `ExecHashJoinOuterGetTuple()` 从 outerPlan 拉取 probe side。
bucket probe -> `ExecScanHashBucket()` 或 parallel 变体扫描 bucket chain。
输出匹配 tuple -> joinqual 和其它 qual 通过后，projection 生成结果 slot。
`ExecHashJoinNewBatch()` -> 当前 batch 完成后，重置 hash table 并装载下一个 inner batch。
unmatched scan / cleanup -> 外连接需要扫描未匹配 inner tuple，结束时销毁 hash table 和临时状态。
```

### 5.1 `ExecInitHashJoin()`

初始化 HashJoinState、表达式、slot 和 hash clauses。

状态变化：`hj_JoinState` 设为 build 状态。

正确性或资源边界：还没有读取完整 inner。

调试时可以在 `ExecInitHashJoin()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 `HJ_BUILD_HASHTABLE`

驱动 inner Hash 节点把 build side 读完。

状态变化：`ExecHashTableCreate()` 分配 hash table 并设置
memory budget。

正确性或资源边界：build 是阻塞阶段，首行输出必须等待它完成。

调试时可以在 `HJ_BUILD_HASHTABLE`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 `ExecHashTableInsert()`

每个 inner tuple 计算 hash value 并插入 bucket 或写入 batch
文件。

状态变化：hash table、spaceUsed、batch 状态更新。

正确性或资源边界：内存压力可能触发 batch 增长。

调试时可以在 `ExecHashTableInsert()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 读取 outer tuple

`ExecHashJoinOuterGetTuple()` 从 outerPlan 拉取 probe
side。

状态变化：outer hash value 决定 bucket 或 batch。

正确性或资源边界：outer side 是输出驱动，不一定一次读完。

调试时可以在 读取 outer tuple
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 bucket probe

`ExecScanHashBucket()` 或 parallel 变体扫描 bucket chain。

状态变化：候选 pair 进入 hash clauses 和 joinqual。

正确性或资源边界：hash value 相同不等于 join 条件一定成立。

调试时可以在 bucket probe
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 输出匹配 tuple

joinqual 和其它 qual 通过后，projection 生成结果 slot。

状态变化：当前 outer tuple 可能继续扫描 bucket。

正确性或资源边界：多匹配会连续返回多行。

调试时可以在 输出匹配 tuple
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 `ExecHashJoinNewBatch()`

当前 batch 完成后，重置 hash table 并装载下一个 inner batch。

状态变化：outer batch 文件也被重新读入 probe。

正确性或资源边界：磁盘路径保持 join 语义但增加延迟。

调试时可以在 `ExecHashJoinNewBatch()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 unmatched scan / cleanup

外连接需要扫描未匹配 inner tuple，结束时销毁 hash table 和临时状态。

状态变化：match flags 和 tuplestore 被消费或释放。

正确性或资源边界：cleanup 顺序要先处理依赖 hashCxt 的对象。

调试时可以在 unmatched scan / cleanup
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

HashJoinState 在 ExecInitHashJoin 中创建，HashJoinTable
延迟到第一次执行 build 阶段创建。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

HashJoinState 持有 hash table、当前 outer tuple、当前 bucket
扫描位置和批次状态。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

如果参数没有改变，某些 hash table 状态可复用或重置 match
flags；参数改变时必须销毁并重建。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndHashJoin 先释放 null tuple store，再销毁
HashJoinTable，因为后者会删除 hashCxt。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

spill 文件和内存上下文依赖 ResourceOwner、BufFile 和 executor
cleanup 兜底。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| build/probe 分离 | 只有 build side 完整进入当前 batch 的 hash table 后，probe 才能找到全部匹配。 |
| hash qual 与 joinqual | hash value 只是定位候选 bucket，真正 join 条件仍需表达式检查。 |
| batch 分区 | inner 和 outer 使用一致 hash partition，保证跨 batch 不会漏匹配。 |
| outer join match flags | 未匹配输出依赖 match 标记，不能只看输出行数推断。 |
| memory limit | work_mem 和 hash_mem_multiplier 是资源边界，不是 correctness 来源。 |
| parallel hash | 共享状态需要 DSM 和同步点，普通 HashJoinTable 指针不能裸跨 worker。 |

build/probe 分离 这一层保证的是：只有 build side 完整进入当前 batch 的
hash table 后，probe 才能找到全部匹配。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

hash qual 与 joinqual 这一层保证的是：hash value 只是定位候选
bucket，真正 join 条件仍需表达式检查。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

batch 分区 这一层保证的是：inner 和 outer 使用一致 hash
partition，保证跨 batch 不会漏匹配。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

outer join match flags 这一层保证的是：未匹配输出依赖 match
标记，不能只看输出行数推断。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

memory limit 这一层保证的是：work_mem 和 hash_mem_multiplier
是资源边界，不是 correctness 来源。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

parallel hash 这一层保证的是：共享状态需要 DSM 和同步点，普通
HashJoinTable 指针不能裸跨 worker。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### inner 为空

inner join 可快速结束，outer join 仍可能输出 null-filled rows。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 内存不足

增加 batches，把部分 tuple 写入临时文件。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### skew 失败

MCV 优化不命中时回到普通 bucket/batch 路径。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### hash collision

bucket chain 中继续执行 equality 和 joinqual，保证正确性。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 参数变化

rescan 不能盲目复用旧 hash table。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### build cost

受 inner rows、row width、hash function、skew 和内存预算影响。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### probe cost

受 outer rows、bucket chain 长度和 joinqual CPU 成本影响。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### spill cost

batch 变多会带来临时写、临时读和重复装载 hash table。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### startup latency

Hash Join 通常首行要等 build 完成。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### parallel overhead

并行 hash 增加 DSM、barrier 和 worker skew 诊断成本。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### EXPLAIN Hash node

看 Buckets、Batches、Memory Usage 和 actual rows。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Temp Read/Write

有 temp I/O 时优先怀疑 batch spill 或上游 sort/hashagg。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Buffers

区分 build side 和 probe side 的访问压力。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### loops

Hash 节点通常 loops 小，若被外层参数化反复执行，需要回到上层 join。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### wait event

临时文件 I/O 或 LWLock 等等待可解释 spill 后的延迟。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb

在
`ExecHashJoinImpl()`、`ExecHashTableInsert()`、`ExecHashJoinNewBatch()`
断点看状态机。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

EXPLAIN 中 Hash Join 显示 Batches 大于 1，并伴随 Temp
Read/Write 或明显 I/O timing。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步确认 build side 的实际行数、row width、Hash 节点 Memory
Usage 和 Batches，判断是否进入 batch spill。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

回到 `HashJoinTable`、`nbatch`、batch file 和
`hj_JoinState`，理解一次内存 join 如何被拆成多轮 build/probe。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再看 `ExecHashJoinNewBatch()`，确认 spill 是资源边界下的等价路径，而不是
join 结果错误。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果 Batches 等于 1 但 Hash Join 仍慢，优先检查 probe side
行数、bucket chain、joinqual CPU 或上游扫描。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break ExecHashJoinImpl
break ExecHashTableInsert
break ExecHashJoinNewBatch
break ExecHashTableReset
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

本节的具体落点是：把 Batches、HashJoinTable 内存边界和 batch
临时文件连成一条线。

如果三句话无法首尾相接，说明诊断仍停留在症状描述，还没有完成从 runtime 到 source
的闭环。

这个复盘口径也能帮助区分执行器问题和 planner estimate、schema
设计、数据分布、GUC 或硬件 I/O 问题。

## 11. 常见误区

误区 1：Hash Join 不是总能完全在内存中完成，Batches 大于 1 是资源边界的直接信号。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：Hash node 的 Memory Usage 不是整条 SQL 的全部内存。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：hash collision 不代表错误，只是候选 bucket 需要继续检查。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：work_mem 提大可能减少 batch，但也可能把压力转移到并发内存。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：Hash Join 的 build side 通常是 inner
side，但理解时要以实际计划为准。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：batch spill

```sql
SET work_mem = '1MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big_a JOIN big_b USING (k);
SET work_mem = '128MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big_a JOIN big_b USING (k);
```

观察重点：观察 Batches 和 Temp Read/Write。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：build side 对照

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM small_s JOIN big_b USING (k);
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM big_b JOIN small_s USING (k);
```

观察重点：看计划选择的 build side 是否改变。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：joinqual CPU

```sql
EXPLAIN (ANALYZE) SELECT * FROM a JOIN b ON a.k=b.k AND expensive_pred(a.v,b.v);
```

观察重点：区分 hash bucket 定位和 joinqual 成本。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break ExecHashJoinImpl
break ExecHashTableInsert
break ExecHashJoinNewBatch
```

观察重点：记录 `hj_JoinState` 与 `nbatch` 的变化。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 为什么 Hash Join 首行延迟通常高于 Nested Loop？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. batch spill 为什么不会漏掉跨 batch 的匹配？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. Hash Join 的内存预算和并发查询数之间有什么运维张力？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. 什么时候 Merge Join 比 Hash Join 更可诊断？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 并行 hash 的共享状态应该在哪里观测？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

Hash Join 用 build/probe 拆分换取高吞吐 join。

HashJoinTable 是运行时状态，不是 planner 估算。

batch spill 是内存边界下的正确 fallback。

诊断时优先看 Batches、Memory Usage、Temp I/O、build/probe 行数。

可迁移规律：把大状态放进内存的算法必须有资源边界下的等价分片路径。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
