# PostgreSQL BitmapHeapScan 的两阶段访问模型

## 课程定位

前置知识：已经理解 TupleTableSlot、ExecScan、IndexScan、MVCC
snapshot 和 table AM / index AM 的基本边界。

本节唯一主问题：

```text
为什么 BitmapIndexScan 要先构造 TID bitmap，再由 BitmapHeapScan 按 page 回表取 tuple，而不是让索引扫描直接把 tuple 一个个交给上层？
```

核心矛盾：索引能快速定位候选 TID，但堆访问、MVCC 可见性、lossy page recheck
和 I/O locality 都发生在 heap page
侧；如果把两者硬合成一个节点，要么丢失按页聚合访问的机会，要么让 index AM 承担 heap
语义。

学完后应能判断：一个 Bitmap Heap Scan 的慢，是 bitmap 构造慢、lossy
recheck 多、回表 I/O 重，还是上层 qual 把候选 tuple 过滤掉。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

04 目录的主线已经从
ExecutorStart、PlanState、TupleTableSlot、ExprContext 和
executor memory context 走到具体执行节点。

第 16 节已经建立 scan node 与 table AM / index AM 的边界。

本组课程的读法不是把节点函数逐个背下来，而是抓住一个运行时对象如何随 ExecProcNode
的一次次调用推进。

每节都会把源码状态落回 EXPLAIN、pg_stat、wait event、临时文件或 gdb
断点中能看到的现象。

本节只回答一个问题：为什么 BitmapIndexScan 要先构造 TID bitmap，再由
BitmapHeapScan 按 page 回表取 tuple，而不是让索引扫描直接把 tuple
一个个交给上层？

后续阅读不要把本节写成节点百科。所有函数、字段、指标都必须回到这一条主线。

第 18 节会把主线转到 Nested Loop，观察 outer tuple 如何反复改变 inner
scan 的参数。

从课程组位置看，17-24 节是常见执行节点源码 walkthrough。它夹在 scan/access
method 边界和 EXPLAIN instrumentation
之间，因此每节都要同时回答两个问题：节点如何产生或消费 tuple，以及这个行为最终如何被观测。

本节的重点不是覆盖所有分支，而是建立一个能迁移到慢 SQL 诊断的 mental model。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
BitmapIndexScan、BitmapAnd 或 BitmapOr 先运行 MultiExec 路径，产出只描述候选位置的 TIDBitmap。
BitmapHeapScan 再按 page 迭代 TIDBitmap，交给 table_scan_bitmap_next_tuple 做可见性、exact/lossy 判断和 slot 填充。
如果 page 是 lossy，executor 必须重新执行 recheck qual，因为 bitmap 只说明这一页可能有匹配 tuple。
prefetch 和并行 bitmap 只改变访问调度，不改变“先候选位置、后 heap 语义”的两阶段边界。
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
| 1 | src/backend/executor/nodeBitmapIndexscan.c | `MultiExecBitmapIndexScan()` 扫描 index AM，把匹配 TID 写入 TIDBitmap；它不是普通 tuple-producing 节点。 |
| 2 | src/backend/executor/nodeBitmapAnd.c | `MultiExecBitmapAnd()` 把多个 bitmap 做交集，保留多索引条件同时满足的候选位置。 |
| 3 | src/backend/executor/nodeBitmapOr.c | `MultiExecBitmapOr()` 把多个 bitmap 做并集，表达 OR 条件下的候选 page/TID 集合。 |
| 4 | src/backend/nodes/tidbitmap.c | `tbm_create()`、`tbm_add_tuples()`、`tbm_begin_iterate()`、`tbm_iterate()` 管理 exact/lossy bitmap 的状态推进。 |
| 5 | src/include/nodes/tidbitmap.h | 定义 `TIDBitmap`、`TBMIterateResult` 和 bitmap iterator 的公开边界。 |
| 6 | src/backend/executor/nodeBitmapHeapscan.c | `ExecBitmapHeapScan()`、`BitmapHeapNext()` 使用 bitmap 回表并处理 recheck、prefetch、rescan 和 cleanup。 |
| 7 | src/include/access/tableam.h | `table_scan_bitmap_next_tuple()` 是 executor 与 table AM 的 bitmap 回表接口。 |
| 8 | src/backend/access/heap/heapam.c | heap AM 的 bitmap 扫描实现负责 page 内可见性、offset 迭代和 slot 填充。 |
| 9 | src/backend/executor/execAmi.c | `ExecReScanBitmapHeapScan()` 的调度入口说明 bitmap 结果何时失效并需要重建。 |

阅读顺序按 mental model
展开：入口状态、状态结构、主流程、rescan/cleanup、异常或 fallback、观测入口。

不要从文件顶部线性读到文件底部。先找谁创建状态、谁推进状态、谁释放状态，再补充具体分支。

## 4. 关键数据结构与状态边界

本节不复制结构体源码。关键是把 raw field 放回 owner、生命周期、并发或资源边界中解释。

### `TIDBitmap`

backend-local 或 shared iterator 可访问的候选位置集合；它保存 page
和 offset 的近似或精确形态，不保存 tuple 内容。

owner / 生命周期：由 bitmap index/and/or
子树创建或填充，BitmapHeapScan 持有并迭代。

诊断边界：不能把它理解成“已经可见的结果集”，它只代表需要回表验证的候选位置。

单独看到 `TIDBitmap` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `TBMIterateResult`

一次迭代返回一个 heap block 的候选 tuple offsets，或者返回 lossy
page 标记。

owner / 生命周期：由 `tbm_iterate()` 在 BitmapHeapScan
拉取下逐页产生。

诊断边界：exact page 可以只检查列出的 offsets，lossy page 需要扫描更多
tuple 并重做条件。

单独看到 `TBMIterateResult` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `BitmapHeapScanState.tbm`

当前执行周期内的 bitmap 指针。

owner / 生命周期：第一次执行时由 outer bitmap 子计划生成，rescan 或 end
时释放或置空。

诊断边界：参数变化后旧 bitmap 不能继续使用，因为 TID 集合可能对应旧外部参数。

单独看到 `BitmapHeapScanState.tbm` 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `BitmapHeapScanState.ss.ss_currentScanDesc`

table AM bitmap heap scan 的描述符。

owner / 生命周期：`ExecInitBitmapHeapScan()` 建立 scan
state，实际访问由 table AM 延迟推进。

诊断边界：它的 snapshot 和 relation 语义来自 scan state，不来自
bitmap 本身。

单独看到 `BitmapHeapScanState.ss.ss_currentScanDesc`
的值并不等于理解了语义。必须同时看当前 PlanState、ExprContext、slot 或资源
owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### `recheck` / `lossy` 标记

告诉 executor 是否必须再次执行 index quals。

owner / 生命周期：index AM 或 tidbitmap 在 exact/lossy
转换时传播，BitmapHeapScan 在回表时消费。

诊断边界：EXPLAIN 中 Rows Removed by Index Recheck
只在需要重查时有解释价值。

单独看到 `recheck` / `lossy` 标记 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

### prefetch 状态

提前把后续 block 访问交给 buffer manager，降低随机 I/O 暴露在执行栈上的时间。

owner / 生命周期：由 BitmapHeapScan 在迭代 bitmap 时推进。

诊断边界：prefetch 不改变可见性，也不能让 lossy page 跳过 recheck。

单独看到 prefetch 状态 的值并不等于理解了语义。必须同时看当前
PlanState、ExprContext、slot 或资源 owner。

如果这个状态出现在 EXPLAIN 或 gdb 中，应先判断它属于本次调用、本轮 rescan、本条
SQL，还是整个 backend 生命周期。

## 5. 主流程源码 walkthrough

主流程按时间推进。下面的链路不是 API 清单，而是本节状态从初始化、被消费、发生边界变化，到返回上层
slot 或完成副作用的故事。

```text
`ExecInitBitmapHeapScan()` -> 构造 `BitmapHeapScanState`、初始化 scan tuple slot、表达式和结果类型。
`MultiExecBitmapIndexScan()` -> BitmapIndexScan 进入 MultiExec 路径，调用 index AM 获取匹配 TID。
`MultiExecBitmapAnd()` / `MultiExecBitmapOr()` -> 多个 bitmap 子节点把 TID 集合做集合运算。
`tbm_begin_iterate()` -> BitmapHeapScan 取得 iterator，准备按 block 顺序访问 heap。
`BitmapHeapNext()` -> 向 iterator 要下一个 block，然后交给 table AM 取 tuple。
`table_scan_bitmap_next_tuple()` -> table AM 在指定 block 内按 snapshot 和 offsets 取可见 tuple。
recheck qual -> 如果当前 page 或 tuple 标记要求重查，执行器重新计算 bitmap index quals。
`ExecReScanBitmapHeapScan()` / `ExecEndBitmapHeapScan()` -> 参数变化、节点重扫或执行结束时清理 iterator、scan descriptor 和 bitmap。
```

### 5.1 `ExecInitBitmapHeapScan()`

构造 `BitmapHeapScanState`、初始化 scan tuple
slot、表达式和结果类型。

状态变化：此时通常还没有真正的 `TIDBitmap`，因为 bitmap 子树在执行期才跑。

正确性或资源边界：初始化阶段只建立 owner 和接口，不提前访问 heap page。

调试时可以在 `ExecInitBitmapHeapScan()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.2 `MultiExecBitmapIndexScan()`

BitmapIndexScan 进入 MultiExec 路径，调用 index AM 获取匹配
TID。

状态变化：候选位置通过 `tbm_add_tuples()` 写入 bitmap。

正确性或资源边界：这一段只形成候选位置，不做完整 heap tuple 输出。

调试时可以在 `MultiExecBitmapIndexScan()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.3 `MultiExecBitmapAnd()` / `MultiExecBitmapOr()`

多个 bitmap 子节点把 TID 集合做集合运算。

状态变化：AND/OR 改变候选空间大小，也可能推动 bitmap 变 lossy。

正确性或资源边界：集合运算仍然不证明 tuple 当前可见。

调试时可以在 `MultiExecBitmapAnd()` /
`MultiExecBitmapOr()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.4 `tbm_begin_iterate()`

BitmapHeapScan 取得 iterator，准备按 block 顺序访问 heap。

状态变化：bitmap 从构造阶段进入迭代阶段。

正确性或资源边界：迭代顺序服务 page locality，不服务 SQL 输出排序。

调试时可以在 `tbm_begin_iterate()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.5 `BitmapHeapNext()`

向 iterator 要下一个 block，然后交给 table AM 取 tuple。

状态变化：当前 block、offset 列表、lossy/recheck 标记开始影响 tuple
输出。

正确性或资源边界：这是两阶段模型从“候选位置”进入“heap 语义”的边界。

调试时可以在 `BitmapHeapNext()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.6 `table_scan_bitmap_next_tuple()`

table AM 在指定 block 内按 snapshot 和 offsets 取可见 tuple。

状态变化：slot 被填充，MVCC 可见性和 HOT 路径在 table AM/heap AM
中完成。

正确性或资源边界：executor 不直接解释 heap page layout。

调试时可以在 `table_scan_bitmap_next_tuple()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.7 recheck qual

如果当前 page 或 tuple 标记要求重查，执行器重新计算 bitmap index quals。

状态变化：候选 tuple 可能被 Rows Removed by Index Recheck 淘汰。

正确性或资源边界：recheck 是 correctness 边界，不是优化器估算的装饰字段。

调试时可以在 recheck qual
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

### 5.8 `ExecReScanBitmapHeapScan()` / `ExecEndBitmapHeapScan()`

参数变化、节点重扫或执行结束时清理 iterator、scan descriptor 和 bitmap。

状态变化：旧候选集合失效，下一轮需要重新构造或重新迭代。

正确性或资源边界：cleanup 依赖 executor context 和 node end 顺序。

调试时可以在 `ExecReScanBitmapHeapScan()` /
`ExecEndBitmapHeapScan()`
附近断点，但不要长时间单步热路径。更可靠的办法是记录本节主状态在进入和离开该函数时的变化。

如果这一步返回一个 slot，要继续问 slot 内容是谁持有、何时失效、是否已经
materialize。

如果这一步不返回 slot，要判断它是在建立缓存、推进状态机、执行副作用，还是在等待下一次调用继续。

## 6. 生命周期 / ownership / cleanup

### 创建

Bitmap 子计划在第一次执行 BitmapHeapScan 时被 MultiExec
拉起，bitmap memory 进入当前 executor 生命周期。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 持有

BitmapHeapScanState 持有 bitmap 指针、iterator 和 scan
descriptor；上层只看到它一次返回一个 slot。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 重扫

outer param 或 chgParam 变化后，旧 bitmap
的候选集合失去语义，ExecReScan 路径需要重建。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 释放

ExecEndBitmapHeapScan 关闭 heap scan、释放
bitmap/iterator，剩余 backend-local 内存随 executor
context 删除。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

### 异常

ERROR 路径不会逐层按普通 C 返回执行所有局部清理，内存主要靠 executor context
和 ResourceOwner 边界兜底。

这一路径的核心不是谁调用了 `pfree()`，而是对象挂在哪个 executor context、哪个
ResourceOwner 或哪个节点私有状态下。

正常路径和 ERROR 路径都要能回到同一个 owner
边界，否则就会出现泄漏、重复释放或使用失效状态。

在 gdb 中检查生命周期时，建议同时看
`estate->es_query_cxt`、节点私有指针、slot 类型和子节点是否已经 end。

## 7. 正确性机制层次

| 层次 | 本节机制 |
| --- | --- |
| MVCC snapshot | 候选 TID 必须回表后按 snapshot 判断可见性，bitmap 本身不包含可见性语义。 |
| lossy recheck | lossy page 只代表页面可能包含匹配 tuple，必须重新执行 index quals 或 recheck cond。 |
| table AM 边界 | heap page 格式、visibility 和 HOT 细节留在 table AM/heap AM，不泄漏给通用 executor。 |
| buffer pin / lock | page 访问中的 pin、visibility 和 tuple deform 生命周期由存储层维护。 |
| rescan invalidation | 参数化 bitmap 子树的参数变化必须让旧 bitmap 失效。 |
| parallel safety | 并行 bitmap 需要 shared iterator 或 DSM 协议，普通 backend-local 指针不能直接跨 worker。 |

MVCC snapshot 这一层保证的是：候选 TID 必须回表后按 snapshot
判断可见性，bitmap 本身不包含可见性语义。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

lossy recheck 这一层保证的是：lossy page 只代表页面可能包含匹配
tuple，必须重新执行 index quals 或 recheck cond。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

table AM 边界 这一层保证的是：heap page 格式、visibility 和 HOT
细节留在 table AM/heap AM，不泄漏给通用 executor。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

buffer pin / lock 这一层保证的是：page 访问中的 pin、visibility 和
tuple deform 生命周期由存储层维护。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

rescan invalidation 这一层保证的是：参数化 bitmap 子树的参数变化必须让旧
bitmap 失效。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

parallel safety 这一层保证的是：并行 bitmap 需要 shared iterator
或 DSM 协议，普通 backend-local 指针不能直接跨 worker。

它不能替代其它层次。例如 MVCC 不等于互斥，内存上下文不等于语义仍然有效，资源预算也不等于正确性。

PostgreSQL 在 executor 中通常不是靠单一机制保证正确性，而是靠状态机、slot
生命周期、表达式语义、存储层回调和 cleanup 顺序共同收束。

## 8. 错误路径 / 异常路径 / fallback

### work_mem 不足

TIDBitmap 会把部分 exact tuple offsets 降级成 lossy
page，节省内存但增加回表 recheck 成本。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 候选集合为空

BitmapHeapScan 可以很快结束，EXPLAIN 里 Bitmap Heap Scan 的
actual rows 可能为零。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 索引条件不够精确

GiST、GIN 或 lossy bitmap 可能产生大量候选，真正过滤发生在回表和 recheck。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 缓存未命中

按 page 聚合改善局部性，但不能消除随机访问和 visibility map、buffer miss
带来的成本。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

### 参数变化

Nested Loop 内侧的 bitmap scan 如果依赖外层参数，必须每个外层值重建候选集合。

这个分支不应该被看成“少见边角”。它通常正是线上问题、性能突变或解释 EXPLAIN 异常字段的入口。

阅读源码时要确认 fallback 是否改变输出语义，还是只改变资源使用、执行顺序或等待方式。

如果某个 fallback 发生在热路径中，建议先用 EXPLAIN、日志或计数确认频率，再决定是否进入
gdb。

## 9. 成本、资源与跨模块传播

### bitmap 构造成本

受索引选择性、索引 AM、BitmapAnd/Or 子树数量和 work_mem 限制影响。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### heap 回表成本

受候选 block 数、缓存命中率、tuple 可见性和 HOT 链长度影响。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### lossy 成本

节省 bitmap memory，但把精确 offset 成本换成 page 内重扫和 recheck
qual 成本。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### prefetch 成本

可能隐藏 I/O latency，但会引入额外 block 调度和并发 I/O 压力。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

### 观测成本

EXPLAIN ANALYZE 的 timing/buffers
会改变热路径开销，诊断时要和普通执行区分。

这个成本通常不会停在本节点内部。它会传播到 buffer manager、临时文件、表达式执行、上层
loops、并行 worker 或统计输出。

诊断时要避免只看节点总耗时。更好的问题是：这个成本随
rows、groups、loops、work_mem、tuple width
或并发度中的哪一个变量扩张。

成本分析必须回到
workload。相同节点在小结果集、热缓存、冷缓存、重复参数或高并发下可能呈现完全不同的瓶颈。

## 10. 观测与诊断入口

### `EXPLAIN (ANALYZE, BUFFERS)`

看 Bitmap Index Scan 产出 rows、Bitmap Heap Scan 的 Heap
Blocks exact/lossy，以及 Rows Removed by Index Recheck。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### `work_mem` 对比

降低 work_mem 观察 exact blocks 向 lossy blocks 转换，确认
lossy 是内存压力的结果。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### Buffers 指标

区分 bitmap 构造阶段访问 index blocks，和 BitmapHeapScan 阶段访问
heap blocks。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### I/O timing

打开 track_io_timing 后观察 heap random read 是否成为主成本。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### gdb 断点

在
`BitmapHeapNext()`、`tbm_iterate()`、`table_scan_bitmap_next_tuple()`
上断点，看候选 page 如何变成 slot。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

### pg_stat_all_indexes

辅助判断索引扫描次数和 idx_tup_read/idx_tup_fetch，但不要把累计指标当成单条
SQL 事实。

看到这个现象后，下一步要回到第 3 节的源码入口，找哪个状态被创建、推进或清理。

不要把累计视图、单次 EXPLAIN、采样 profiler 和 gdb
断点混成同一种证据。它们的时间窗口不同。

一个可靠诊断闭环应该长这样：先看到 runtime
现象，再定位状态边界，然后用源码解释为什么该状态会产生这个输出。

### 诊断闭环示例

这一小节给出一条从 runtime 现象回到源码的完整路径。它不引入新的主题，只把第 10
节的观测入口串成一个可操作的排查顺序。

#### 现场现象

一条查询显示 Bitmap Heap Scan 总耗时高，Bitmap Index Scan
本身很快，但 Heap Blocks lossy 很多。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 第一步

第一步不要急着怀疑索引。先确认 Bitmap Index Scan 产出的候选量、Bitmap Heap
Scan 的 exact/lossy blocks，以及 Rows Removed by Index
Recheck。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 回到状态

如果 lossy blocks 明显，回到 `TIDBitmap` 和
`TBMIterateResult`，解释 work_mem 如何把精确 offset 集合压缩成
page 级候选。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 确认边界

再从 `BitmapHeapNext()` 进入
`table_scan_bitmap_next_tuple()`，确认真正的 MVCC 可见性和
slot 填充发生在 heap/table AM 边界。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 排除相邻原因

如果 lossy 不多而 heap buffers 很高，瓶颈更可能是随机回表或可见性检查，而不是
bitmap 压缩。

这一步要写下一个具体证据，而不是只写“可能是执行器慢”。证据可以来自
EXPLAIN、日志、pg_stat、wait event、临时文件统计、perf 栈或 gdb 断点。

如果证据不能指向本节主状态，就说明还没有回到正确层次，需要退回计划树重新定位。

#### 最小断点路径

```text
break MultiExecBitmapIndexScan
break tbm_iterate
break BitmapHeapNext
break table_scan_bitmap_next_tuple
```

断点只用于确认状态推进顺序，不建议在高频循环里长时间单步。更有效的做法是记录入口参数、关键状态字段和返回
slot 是否为空。

完成一次闭环后，再决定是否需要扩大到 buffer、lock、temporary
file、statistics 或 planner estimate 层面。

## 11. 常见误区

误区 1：Bitmap Heap Scan 不是“先排序后扫描”的通用机制，它只是按 block
聚合候选访问，不承诺 SQL 输出顺序。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 2：Bitmap Index Scan 不直接返回用户 tuple；它返回 TIDBitmap
给上层 MultiExec 消费。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 3：Rows Removed by Index Recheck 不一定表示索引坏了，lossy
page 或 lossy index AM 都可能要求 recheck。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 4：work_mem 增大不总是更快，如果候选 block 本来很少，瓶颈可能在 heap
visibility 或上层 qual。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

误区 5：prefetch 不是 correctness 机制；打开或关闭只改变访问调度。

纠正这个误区的办法，是把结论拆成状态、生命周期、成本变量和观测证据四个部分。

如果四个部分不能互相解释，说明还没有真正回到源码主线。

## 12. 课堂实验

实验目标不是制造某个固定计划，而是练习从可见现象回到源码边界。不同数据分布和版本可能得到不同计划，重点是解释差异。

### 实验 1：exact 与 lossy 对照

```sql
SET enable_seqscan = off;
SET work_mem = '64kB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t WHERE a BETWEEN 1 AND 50000 AND b = 7;
SET work_mem = '64MB';
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t WHERE a BETWEEN 1 AND 50000 AND b = 7;
```

观察重点：观察 Heap Blocks exact/lossy 和 Rows Removed by
Index Recheck 的变化。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 2：BitmapAnd 入口

```sql
CREATE INDEX ON t(a);
CREATE INDEX ON t(b);
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM t WHERE a BETWEEN 10 AND 1000 AND b BETWEEN 20 AND 2000;
```

观察重点：确认 BitmapAnd 是否出现，并区分两个 Bitmap Index Scan
与回表阶段的 buffers。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 3：参数化 bitmap

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM outer_t o WHERE EXISTS (SELECT 1 FROM t WHERE t.a = o.a AND t.b BETWEEN 1 AND 100);
```

观察重点：如果内侧选择 bitmap 路径，重点看 loops 是否让 bitmap 构造成本被放大。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

### 实验 4：源码断点

```text
break MultiExecBitmapIndexScan
break tbm_iterate
break BitmapHeapNext
break table_scan_bitmap_next_tuple
```

观察重点：单步观察候选 TID 如何被拆成 page 级访问。

如果结果和预期不同，先检查统计信息、GUC、数据分布和并发环境，再回到源码判断是否走了相邻路径。

把 EXPLAIN 中的 rows、loops、buffers、temp、WAL 或 trigger
time 与本节主状态对应起来。

实验结束后建议恢复会话级 GUC，避免影响后续课程。

## 13. 讨论题

1. 为什么 BitmapHeapScan 不把 recheck 留给上层通用 Filter
节点，而是在本节点中处理？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

2. 如果 bitmap 永远保持 exact，会损失什么资源边界？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

3. 为什么 bitmap 迭代顺序不等于 SQL ORDER BY？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

4. 当 Bitmap Heap Scan 慢时，哪些证据能证明瓶颈在 index
阶段，哪些证据能证明瓶颈在 heap 阶段？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

5. 并行 bitmap 要解决的主要问题是共享候选集合，还是共享 tuple slot？

回答时要求同时给出一个源码入口、一个运行时状态和一个可观测现象。只给概念性答案不够。

讨论题的目标是把本节 runtime 现象压缩成可迁移规律，而不是展开新的主题。

## 14. 本节小结

BitmapHeapScan 的核心不是“换一种索引扫描”，而是把候选位置和 heap
语义拆成两个阶段。

这个拆分让 index AM 专注产生 TID，让 table AM 负责
visibility、page 访问和 slot 输出。

lossy bitmap 是资源边界下的 fallback，节省内存但把成本转移到 recheck。

诊断时要同时看 Bitmap Index Scan、Bitmap Heap Scan、Heap
Blocks exact/lossy、Buffers 和 loops。

可迁移规律：当一个组件只能给出候选集合时，正确性必须在拥有真实对象语义的边界重新确认。

把本节带回 04 目录主线，可以得到一个稳定读法：PlanState 不是静态结构，而是每次
ExecProcNode 调用都会推进的运行时状态。

下一节继续沿同一读法，只是把主状态换成相邻执行节点或观测对象。
