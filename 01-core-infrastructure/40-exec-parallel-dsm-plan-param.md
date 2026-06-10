# PostgreSQL executor parallel DSM：plan、params、DSA 与 node shared state

## 课程定位

前置知识：已经理解 `ParallelContext`、parallel DSM 启动包、worker bootstrap 和 error queue。还应熟悉 executor 的 `PlanState`、`EState`、`ParamListInfo`、`PARAM_EXEC`、Instrumentation 和 `TupleTableSlot` 基本概念。

本节唯一主问题：

```text
ExecInitParallelPlan() 为什么要序列化 PlannedStmt、ParamListInfo、PARAM_EXEC、query text、
instrumentation 和 DSA area，
ExecParallelEstimate() / ExecParallelInitializeDSM() 又如何让并行感知 plan node
在同一个 DSM 中建立自己的共享状态？
```

核心矛盾：parallel worker 已经通过 `ParallelWorkerMain()` 恢复了事务、snapshot、GUC 和身份，但 executor 仍需要可执行计划、参数、tuple 输出队列、per-worker usage、instrumentation、JIT 统计、DSA 和各 plan node 的共享状态。leader 不能把 `PlanState *` 或 `EState *` 直接传给 worker，只能把 worker 可重建执行器所需的状态序列化进 DSM。

学完后应能判断：`ParallelContext` 提供的是通用并行基础设施，`ExecInitParallelPlan()` 才是 parallel query 的 executor-specific DSM 构建层；也能解释为什么计划序列化、参数序列化、DSA 和 node callbacks 必须在 worker launch 之前完成。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

前面课程已经把 worker 启动到可运行状态：

```text
ParallelContext
  -> InitializeParallelDSM()
  -> ParallelWorkerMain()
  -> entrypoint(seg, toc)
```

对于 parallel query，entrypoint 是：

```text
ParallelQueryMain(dsm_segment *seg, shm_toc *toc)
```

本节讲 leader 如何在 worker 启动前准备 `ParallelQueryMain()` 所需的 executor DSM：

```text
ExecInitParallelPlan()
  -> serialize plan / params / query text
  -> setup tuple queues
  -> setup usage / instrumentation / JIT
  -> create DSA
  -> initialize parallel-aware node DSM
```

下一节会讲 `Gather` / `GatherMerge` 如何启动 worker、创建 tuple readers，并在 leader 本地执行与 worker tuple stream 之间取舍。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ExecInitParallelPlan() 先强制求值需要发送给 worker 的 initplan 参数，
复制并序列化 plan 为一个 worker 可独立 ExecutorStart() 的 PlannedStmt，
创建 ParallelContext 并估算 executor fixed state、query text、ParamListInfo、Buffer/WAL usage、
tuple queues、node shared state、instrumentation、JIT 和 DSA；
InitializeParallelDSM() 创建 DSM 后，leader 把这些对象逐项写入 TOC，
再遍历 PlanState tree 调用每类节点的 Estimate / InitializeDSM callback，
让 worker 后续在 ParallelQueryMain() 中重建 QueryDesc 并执行。
```

tension 是：

```text
worker 必须执行“同一个计划”
  vs
leader 的 PlanState/EState/Param 指针都是 backend-local runtime object
```

PostgreSQL 的解决方案是：

```text
Plan:
  序列化为 PlannedStmt 文本形式

ParamListInfo:
  序列化为 DSM bytes

PARAM_EXEC:
  运行期可变，用 DSA 存 dsa_pointer

PlanState shared state:
  由各 node 自己 estimate / initialize / worker attach
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/executor/execParallel.c` | `ExecInitParallelPlan()`、`ExecSerializePlan()`、`ExecParallelEstimate()`、`ExecParallelInitializeDSM()`、`ParallelQueryMain()`。 |
| 2 | `src/include/executor/execParallel.h` | `ParallelExecutorInfo` 字段与 executor 对外接口。 |
| 3 | `src/backend/executor/tqueue.c` | tuple queue receiver / reader 如何基于 shm_mq。 |
| 4 | `src/backend/executor/nodeSeqscan.c`、`nodeAppend.c`、`nodeHashjoin.c` 等 | parallel-aware node 的 Estimate / InitializeDSM / InitializeWorker callback。 |
| 5 | `src/include/storage/shm_toc.h` | executor DSM key 如何放入同一 TOC。 |
| 6 | `src/include/utils/dsa.h` | executor DSA area 和 `dsa_pointer` 参数传递。 |

阅读顺序建议：

```text
ExecInitParallelPlan()
  -> ExecSerializePlan()
  -> ExecParallelEstimate()
  -> InitializeParallelDSM()
  -> ExecParallelSetupTupleQueues()
  -> ExecParallelInitializeDSM()
  -> ParallelQueryMain()
```

不要从单个 plan node 的 parallel callback 开始读。先建立 executor DSM 总布局，再看各 node 如何插入自己的共享状态。

## 4. 关键数据结构与状态

### `ParallelExecutorInfo`

定义在 `src/include/executor/execParallel.h`：

| 字段 | 语义 |
| --- | --- |
| `planstate` | leader 中要并行执行的 plan subtree。 |
| `pcxt` | 通用 `ParallelContext`。 |
| `buffer_usage` / `wal_usage` | DSM 中每 worker 的 BufferUsage / WalUsage 数组。 |
| `instrumentation` | DSM 中 worker 节点 instrumentation 汇总区。 |
| `jit_instrumentation` | DSM 中 worker JIT instrumentation 汇总区。 |
| `area` | executor query DSA area。 |
| `param_exec` | serialized `PARAM_EXEC` 参数的 DSA pointer。 |
| `finished` | `ExecParallelFinish()` 是否已执行。 |
| `tqueue` | worker 输出 tuple queues 的 leader-side shm_mq handles。 |
| `reader` | `TupleQueueReader` 数组，`Gather` / `GatherMerge` 用它读取 tuple。 |

这个结构是 leader-local executor ownership 对象。它把通用 parallel context 和 executor 的 tuple/instrumentation/DSA 状态合在一起。

### executor DSM keys

`execParallel.c` 定义的 key 大于 32-bit 范围，避免和 plan node 自定义 key 冲突：

| key | 内容 |
| --- | --- |
| `PARALLEL_KEY_EXECUTOR_FIXED` | `FixedParallelExecutorState`。 |
| `PARALLEL_KEY_PLANNEDSTMT` | 序列化的 worker-side `PlannedStmt`。 |
| `PARALLEL_KEY_PARAMLISTINFO` | 序列化的 external params。 |
| `PARALLEL_KEY_BUFFER_USAGE` | 每 worker 一个 `BufferUsage`。 |
| `PARALLEL_KEY_TUPLE_QUEUE` | 每 worker 一个 tuple output queue。 |
| `PARALLEL_KEY_INSTRUMENTATION` | shared executor instrumentation。 |
| `PARALLEL_KEY_DSA` | executor DSA area in-place storage。 |
| `PARALLEL_KEY_QUERY_TEXT` | query string。 |
| `PARALLEL_KEY_JIT_INSTRUMENTATION` | shared JIT instrumentation。 |
| `PARALLEL_KEY_WAL_USAGE` | 每 worker 一个 `WalUsage`。 |

这些 key 共享 `ParallelContext.toc`，也就是和 `PARALLEL_KEY_FIXED`、error queue、GUC state 等通用 parallel keys 在同一个 DSM TOC 中。

### `FixedParallelExecutorState`

```text
tuples_needed:
  传给 worker 的 tuple bound，用于 LIMIT / Gather 需求。

param_exec:
  DSA pointer，指向 serialized PARAM_EXEC values。

eflags:
  worker ExecutorStart() 使用的 executor flags。

jit_flags:
  worker plannedstmt->jitFlags。
```

这个结构固定小，但它把 leader 的 executor runtime demand 传给 worker。

## 5. 主流程源码 walkthrough：`ExecInitParallelPlan()`

入口：

```text
ExecInitParallelPlan(PlanState *planstate,
                     EState *estate,
                     Bitmapset *sendParams,
                     int nworkers,
                     int64 tuples_needed)
```

### 5.1 强制求值 initplan 输出

```text
ExecSetParamPlanMulti(sendParams, GetPerTupleExprContext(estate))
```

`sendParams` 表示 worker 需要的 `PARAM_EXEC` 参数。如果这些参数来自 initplan，必须先在 leader 中求值，否则 worker 只拿到 param id，没有实际值。

这里使用 per-output-tuple ExprContext，源码注释承认可能有轻微 intra-query memory retention，但 `ExecSetParamPlan()` 正常不泄漏，没必要为这个路径复杂化 API。

### 5.2 序列化 plan

```text
pstmt_data = ExecSerializePlan(planstate->plan, estate)
```

`ExecSerializePlan()` 做几件重要事情：

```text
copyObject(plan)
把 targetlist 中 resjunk 标记改为 false
构造 dummy PlannedStmt
填充 range table、paramExecTypes、partition pruning info 等 worker 需要的字段
只传 parallel-safe subplans，不安全 subplan 位置留 NULL
nodeToString(pstmt)
```

为什么要把 resjunk 改成 false？worker 输出 tuple 给 leader，不是直接给客户端。leader 可能需要 junk columns 做 row identity、sort/group 或 upper node 处理。worker 如果像 top-level executor 一样过滤 junk，会破坏上层执行器需求。

为什么 unsafe subplans 留 NULL？worker 不应该初始化或执行非 parallel-safe subplan，但 list index 要保留，否则 param / subplan id 对不上。

### 5.3 创建 ParallelContext

```text
pcxt = CreateParallelContext("postgres", "ParallelQueryMain", nworkers)
```

这说明 executor worker 的 entrypoint 是 core backend 中的 `ParallelQueryMain()`。它不是直接启动某个 plan node 函数。

### 5.4 估算 executor DSM

估算顺序：

```text
FixedParallelExecutorState
query text
serialized PlannedStmt
serialized ParamListInfo
BufferUsage[nworkers]
WalUsage[nworkers]
tuple queues
per-node shared state via ExecParallelEstimate()
instrumentation / JIT
DSA minimum area
```

这里有两个独立层次：

```text
executor fixed DSM:
  所有 parallel query 都需要

node-specific DSM:
  只有 parallel-aware node 或 instrumentation callback 需要
```

### 5.5 active snapshot assert

```text
Assert(GetActiveSnapshot() == estate->es_snapshot)
```

通用 `InitializeParallelDSM()` 会序列化 active snapshot，worker 后续用它作为 executor snapshot。如果 `EState.es_snapshot` 不是当前 active snapshot，worker 会用和 leader 不同的可见性视图执行同一计划。

### 5.6 创建 DSM 并写入 executor state

```text
InitializeParallelDSM(pcxt)

fpes = shm_toc_allocate(PARALLEL_KEY_EXECUTOR_FIXED)
query_string = shm_toc_allocate(PARALLEL_KEY_QUERY_TEXT)
pstmt_space = shm_toc_allocate(PARALLEL_KEY_PLANNEDSTMT)
paramlistinfo_space = shm_toc_allocate(PARALLEL_KEY_PARAMLISTINFO)
bufusage_space = shm_toc_allocate(PARALLEL_KEY_BUFFER_USAGE)
walusage_space = shm_toc_allocate(PARALLEL_KEY_WAL_USAGE)
pei->tqueue = ExecParallelSetupTupleQueues(pcxt, false)
instrumentation = shm_toc_allocate(PARALLEL_KEY_INSTRUMENTATION)
jit_instrumentation = shm_toc_allocate(PARALLEL_KEY_JIT_INSTRUMENTATION)
area_space = shm_toc_allocate(PARALLEL_KEY_DSA)
pei->area = dsa_create_in_place(...)
```

注意：tuple queues 在 worker launch 前已经创建，worker 后续按 `ParallelWorkerNumber` attach 自己的 queue。

### 5.7 PARAM_EXEC 使用 DSA

`PARAM_EXEC` 参数用 DSA，而不是 main parallel DSM：

```text
pei->param_exec = SerializeParamExecParams(estate, sendParams, pei->area)
fpes->param_exec = pei->param_exec
```

源码注释说明原因：worker 可能被 relaunch，参数值可能改变，序列化后的大小也可能改变。主 parallel DSM 的 TOC 空间不适合动态调整，DSA 支持运行期 allocate/free。

### 5.8 初始化 node shared state

```text
estate->es_query_dsa = pei->area
ExecParallelInitializeDSM(planstate, &d)
estate->es_query_dsa = NULL
```

设置 `es_query_dsa` 是为了让 plan node callback 可以把需要跨进程的动态对象放入同一个 DSA area。

最后检查：

```text
if (e.nnodes != d.nnodes)
  elog(ERROR, "inconsistent count of PlanState nodes")
```

Estimate 和 InitializeDSM 遍历的是同一棵 planstate tree。节点数量不同表示 plan tree 在两阶段之间发生了不应发生的变化。

## 6. node callbacks：Estimate / InitializeDSM / InitializeWorker

`ExecParallelEstimate()` 遍历 `PlanState`：

```text
e->nnodes++
switch nodeTag(planstate):
  if parallel_aware:
    ExecSeqScanEstimate()
    ExecAppendEstimate()
    ExecHashJoinEstimate()
    ...
  instrumentation callbacks even when not parallel-aware
```

`ExecParallelInitializeDSM()` 再次遍历：

```text
if instrumentation:
  instrumentation->plan_node_id[d->nnodes] = plan_node_id
d->nnodes++
switch nodeTag(planstate):
  if parallel_aware:
    ExecSeqScanInitializeDSM()
    ExecAppendInitializeDSM()
    ExecHashJoinInitializeDSM()
    ...
```

worker 侧 `ExecParallelInitializeWorker()` 第三次遍历：

```text
if parallel_aware:
  ExecSeqScanInitializeWorker()
  ExecAppendInitializeWorker()
  ExecHashJoinInitializeWorker()
instrumentation worker init callbacks
```

这三阶段协议很重要：

| 阶段 | 运行方 | 作用 |
| --- | --- | --- |
| Estimate | leader | 声明 DSM/TOC 空间需求。 |
| InitializeDSM | leader | 真正 allocate / insert / 初始化 shared state。 |
| InitializeWorker | worker | lookup shared state，绑定到 worker-local PlanState。 |

如果新增一个 parallel-aware node，必须实现成对的 estimate / initialize / worker attach 逻辑，否则 worker 可能拿不到共享进度状态。

## 7. `ParallelQueryMain()` worker 消费路径

worker 进入 `ParallelQueryMain(seg, toc)` 后：

```text
fpes = shm_toc_lookup(PARALLEL_KEY_EXECUTOR_FIXED)
receiver = ExecParallelGetReceiver(seg, toc)
instrumentation = shm_toc_lookup(PARALLEL_KEY_INSTRUMENTATION, true)
jit_instrumentation = shm_toc_lookup(PARALLEL_KEY_JIT_INSTRUMENTATION, true)
queryDesc = ExecParallelGetQueryDesc(toc, receiver, instrument_options)
debug_query_string = queryDesc->sourceText
pgstat_report_activity(STATE_RUNNING, debug_query_string)
area = dsa_attach_in_place(PARALLEL_KEY_DSA)
queryDesc->plannedstmt->jitFlags = fpes->jit_flags
ExecutorStart(queryDesc, fpes->eflags)
estate->es_query_dsa = area
if fpes->param_exec valid:
  RestoreParamExecParams(dsa_get_address(area, fpes->param_exec), estate)
ExecParallelInitializeWorker(queryDesc->planstate, &pwcxt)
ExecSetTupleBound(fpes->tuples_needed, queryDesc->planstate)
InstrStartParallelQuery()
ExecutorRun(queryDesc, ForwardScanDirection, tuple bound)
ExecutorFinish(queryDesc)
InstrEndParallelQuery(&buffer_usage[worker], &wal_usage[worker])
ExecParallelReportInstrumentation()
copy JIT instrumentation
```

这说明 worker 的 executor 与 leader 是两套 `EState` / `PlanState`，但它们通过 DSM/DSA 共享必要的协调状态。

## 8. 生命周期 / ownership / cleanup

### leader owns

```text
ParallelExecutorInfo
ParallelContext
leader-side tuple queue handles
TupleQueueReader array
DSA area handle
serialized param_exec pointer
```

### DSM owns

```text
serialized PlannedStmt
serialized ParamListInfo
query text
BufferUsage / WalUsage arrays
instrumentation arrays
tuple queue memory
DSA in-place control area
node shared state
```

### worker owns

```text
worker-local QueryDesc
worker-local EState / PlanState
worker-side tuple queue receiver
worker DSA attach handle
worker instrumentation before report
```

### cleanup boundaries

Normal path:

```text
ExecParallelFinish()
  -> detach tuple queues
  -> destroy readers
  -> wait worker finish
  -> accumulate Buffer/WAL usage

ExecParallelCleanup()
  -> retrieve instrumentation / JIT
  -> free serialized PARAM_EXEC from DSA
  -> dsa_detach()
  -> DestroyParallelContext()
```

参数用 DSA，所以 relaunch/rescan 时可以 free old pointer and allocate new one without rebuilding whole DSM。

## 9. 正确性机制层次

| 层次 | 机制 | 解决的问题 |
| --- | --- | --- |
| plan identity | serialized `PlannedStmt` | worker 能构造自己的 QueryDesc。 |
| visibility | active snapshot from `InitializeParallelDSM()` | worker executor snapshot 与 leader 一致。 |
| external params | serialized `ParamListInfo` | prepared statement / SQL params 可用。 |
| internal params | DSA `PARAM_EXEC` | initplan / nestloop params 可随 rescan 更新。 |
| tuple routing | per-worker tuple queues | worker tuple 回到 leader。 |
| shared algorithm state | node callbacks | parallel-aware scan/join/append 等协调进度。 |
| observability | Buffer/WAL/instrumentation/JIT DSM arrays | worker 指标可汇总进 leader EXPLAIN / stats。 |
| dynamic allocation | executor DSA | 跨 worker 小对象和可变参数不挤压固定 TOC。 |

## 10. 错误路径 / 异常路径 / fallback

### 10.1 计划序列化不完整

如果 worker 需要的字段没有进入 dummy `PlannedStmt`，可能表现为：

```text
worker ExecutorStart() ERROR
plan node 初始化缺少 range table / param type / partition info
```

因此 `ExecSerializePlan()` 是一个白名单式 contract。新增 executor 功能如果 parallel worker 需要，也要补进序列化字段。

### 10.2 unsafe subplan

非 parallel-safe subplan 被置 NULL。worker 不应执行它。如果某个路径错误地依赖 worker 执行 unsafe subplan，会在初始化或执行中暴露为缺失 subplan 或参数错误。

### 10.3 PARAM_EXEC 大小变化

rescan 时参数可能变化，旧 DSA pointer 必须释放：

```text
if DsaPointerIsValid(fpes->param_exec):
  dsa_free(pei->area, fpes->param_exec)
```

否则 DSA 中会保留旧参数，长期重复 rescan 造成内存增长。

### 10.4 DSM fallback 到 no worker

如果通用 `InitializeParallelDSM()` 把 `pcxt->nworkers` 降为 0，executor 仍可能已经写入 TOC，但不会 launch worker。`Gather` 会发现 `nworkers_launched == 0`，由 leader 本地执行。

### 10.5 node estimate/init 不匹配

`e.nnodes != d.nnodes` 会 ERROR。更隐蔽的是某个 node estimate 了空间但 init 时没有 insert，worker lookup 时才失败。这类 bug 通常发生在新增 parallel-aware node 或 instrumentation callback 时。

## 11. 成本、资源与跨模块传播

parallel executor DSM 的空间随这些因素增长：

```text
nworkers:
  tuple queues、BufferUsage、WalUsage、instrumentation per-worker slots

plan tree size:
  SharedExecutorInstrumentation plan_node_id array
  NodeInstrumentation num_plan_nodes * nworkers

query text / plan serialization:
  SQL 长度、plan tree 复杂度、ParamListInfo 大小

node-specific shared state:
  parallel append / scan / hash join / bitmap heap / instrumentation objects

PARAM_EXEC:
  DSA 动态分配，随参数个数和 datum 大小变化
```

跨模块：

```text
planner:
  标记 parallel_aware / parallel_safe，产生 Gather/GatherMerge

executor:
  序列化计划、启动 worker、读 tuple

storage:
  parallel-aware scan 使用 shared block allocation state

hash join / aggregate / sort:
  使用 DSM/DSA 保存共享批次或 instrumentation

stats / EXPLAIN:
  从 worker DSM arrays 汇总指标
```

## 12. 观测与诊断入口

| 入口 | 能看到什么 |
| --- | --- |
| `EXPLAIN (ANALYZE, VERBOSE)` | worker 数、per-worker rows / time、parallel-aware 节点信息。 |
| `EXPLAIN (ANALYZE, BUFFERS, WAL, JIT)` | worker Buffer/WAL/JIT 指标是否汇总。 |
| gdb `ExecInitParallelPlan()` | TOC key 写入顺序、`pcxt->estimator`、`pei->area`。 |
| gdb `ParallelQueryMain()` | worker 是否成功重建 QueryDesc、attach DSA。 |
| `pg_stat_activity` | worker 的 `debug_query_string` / running state。 |
| wait events | tuple queue 或 finish 阶段等待。 |

调试断点：

```gdb
break ExecInitParallelPlan
break ExecSerializePlan
break ExecParallelEstimate
break ExecParallelInitializeDSM
break ExecParallelSetupTupleQueues
break ParallelQueryMain
break ExecParallelInitializeWorker
```

## 13. 常见误区

1. 误以为 worker 执行 leader 的 `PlanState *`。worker 是反序列化 plan 后自己 `ExecutorStart()`。
2. 误以为 `ParamListInfo` 和 `PARAM_EXEC` 是一类参数。前者来自外部参数列表，后者是 executor 内部运行期参数。
3. 误以为所有 plan node 都需要 DSM。只有 parallel-aware node 或 instrumentation callback 需要。
4. 误以为 DSA 可选。只要需要动态跨进程对象或 relaunch 参数，DSA 就是正确边界。
5. 误以为 instrumentation 是 leader 本地累加。worker 先写 DSM，leader finish/cleanup 后再汇总。

## 14. 课堂实验

### 14.1 查看 worker 计划重建

gdb：

```gdb
break ExecSerializePlan
break ExecParallelGetQueryDesc
break ParallelQueryMain
```

在 worker 中：

```gdb
p queryDesc->plannedstmt->planTree
p queryDesc->sourceText
p queryDesc->params
```

观察 worker 中的 `PlanState` 地址与 leader 不同。

### 14.2 检查 executor DSM keys

在 `ExecInitParallelPlan()` 调用 `InitializeParallelDSM()` 后：

```gdb
p shm_toc_lookup(pcxt->toc, PARALLEL_KEY_QUERY_TEXT, 0)
p shm_toc_lookup(pcxt->toc, PARALLEL_KEY_PLANNEDSTMT, 0)
p shm_toc_lookup(pcxt->toc, PARALLEL_KEY_TUPLE_QUEUE, 0)
p shm_toc_lookup(pcxt->toc, PARALLEL_KEY_DSA, 0)
```

### 14.3 PARAM_EXEC 实验

构造带 initplan 或相关参数的并行查询，给 `SerializeParamExecParams()` 和 `RestoreParamExecParams()` 设断点，观察 param id、datum size 和 DSA pointer。

### 14.4 instrumentation 实验

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, WAL)
SELECT count(*) FROM t_parallel_ctx WHERE id > 100;
```

观察 per-worker rows / loops / buffers 是否出现。关闭 ANALYZE 后，instrumentation DSM 可能不分配。

## 15. 讨论题

1. 为什么 `ExecSerializePlan()` 要把 resjunk 改成 false？
2. 为什么非 parallel-safe subplan 要保留 NULL hole，而不是从 list 中删除？
3. `ParamListInfo` 和 `PARAM_EXEC` 分别服务哪类参数？为什么一个放 DSM，一个放 DSA？
4. 如果新增一个 parallel-aware node，只实现 worker attach 不实现 estimate，会发生什么？
5. 为什么 `e.nnodes != d.nnodes` 是严重错误？

## 16. 源码索引：executor DSM key 到 worker 消费点

| DSM key | leader 写入点 | worker 消费点 | 如果缺失会怎样 |
| --- | --- | --- | --- |
| `PARALLEL_KEY_EXECUTOR_FIXED` | `ExecInitParallelPlan()` | `ParallelQueryMain()` | 无 tuple bound、eflags、JIT flags、PARAM_EXEC pointer。 |
| `PARALLEL_KEY_QUERY_TEXT` | `ExecInitParallelPlan()` | `ExecParallelGetQueryDesc()` | worker queryDesc/sourceText 和 pgstat activity 缺失。 |
| `PARALLEL_KEY_PLANNEDSTMT` | `ExecInitParallelPlan()` | `stringToNode()` | worker 无法构造 QueryDesc。 |
| `PARALLEL_KEY_PARAMLISTINFO` | `SerializeParamList()` | `RestoreParamList()` | SQL 外部参数不可用。 |
| `PARALLEL_KEY_BUFFER_USAGE` | allocate array | `InstrEndParallelQuery()` | worker buffer usage 无处写回。 |
| `PARALLEL_KEY_WAL_USAGE` | allocate array | `InstrEndParallelQuery()` | worker WAL usage 无处写回。 |
| `PARALLEL_KEY_TUPLE_QUEUE` | `ExecParallelSetupTupleQueues()` | `ExecParallelGetReceiver()` | worker 无法返回 tuple。 |
| `PARALLEL_KEY_INSTRUMENTATION` | allocate/init | worker report + leader retrieve | EXPLAIN worker 节点指标缺失。 |
| `PARALLEL_KEY_JIT_INSTRUMENTATION` | allocate/init | worker copy + leader retrieve | worker JIT 指标缺失。 |
| `PARALLEL_KEY_DSA` | `dsa_create_in_place()` | `dsa_attach_in_place()` | PARAM_EXEC 和节点动态共享对象不可用。 |

这个表适合调试 worker startup 中的 `shm_toc_lookup()` ERROR。先看缺哪个 key，再回到 leader 写入路径。

## 17. 源码检查清单：新增 parallel-aware executor node

新增一个并行感知节点时，至少要实现或确认这些边界：

```text
planner:
  Plan.parallel_aware 是否正确设置
  path rows / cost / parallel safety 是否合理

executor leader estimate:
  ExecParallelEstimate() switch 中是否调用你的 Estimate
  是否估算 chunk 和 key

executor leader initialize:
  ExecParallelInitializeDSM() switch 中是否 allocate / insert shared state
  是否初始化锁、计数器、队列、批次或 shared cursor

worker attach:
  ExecParallelInitializeWorker() switch 中是否 lookup shared state
  worker-local PlanState 是否保存 DSM pointer / DSA pointer

rescan:
  ExecParallelReInitializeDSM() 是否需要 reset shared state
  ExecReScan() 是否只 reset local state

instrumentation:
  是否需要 InstrumentEstimate / InitDSM / InitWorker / Retrieve
```

### 17.1 key 设计

executor 保留了大于 32-bit 的 key：

```text
0xE000000000000001 ...
```

注释说明小于 `2^32` 的值可由 individual parallel nodes 使用。新增节点要避免 key 冲突，通常以 plan node id 或 node-private key scheme 区分。

### 17.2 DSA 使用

如果共享对象大小固定，TOC chunk 足够；如果对象大小随执行变化、rescan 变化或需要小对象分配，应考虑 DSA：

```text
fixed shared header:
  shm_toc_allocate()

dynamic arrays / params / hash table entries:
  dsa_allocate()
```

不要把可变大小对象硬塞进一次性 TOC 空间。

## 18. 故障模式速查表

| 现象 | 可能层次 | 排查 |
| --- | --- | --- |
| worker `stringToNode()` 失败 | plan serialization | `ExecSerializePlan()` 输出是否完整。 |
| worker 参数值 NULL / 错误 | ParamListInfo / PARAM_EXEC | `SerializeParamList()`、`SerializeParamExecParams()`。 |
| worker 无 tuple 返回 | tuple queue | `PARALLEL_KEY_TUPLE_QUEUE`、receiver、worker `DestReceiver`。 |
| parallel-aware scan 重复/漏扫 block | node shared state | Estimate/InitializeDSM/ReInitializeDSM callback。 |
| EXPLAIN 没有 worker details | instrumentation | `estate->es_instrument`、SharedExecutorInstrumentation。 |
| rescan 后用旧参数 | DSA param_exec | 是否 free old pointer 并更新 `fpes->param_exec`。 |
| worker JIT 指标缺失 | JIT DSM | `es_jit_flags`、`PARALLEL_KEY_JIT_INSTRUMENTATION`。 |

## 19. 本节小结

`ExecInitParallelPlan()` 是 parallel query 的 executor-specific DSM 构建层。通用 `ParallelContext` 只能让 worker 进入与 leader 一致的事务和会话环境；executor 还必须显式提供计划、参数、tuple queue、usage 统计、instrumentation、JIT、DSA 和每个并行感知节点的共享状态。

可迁移规律：

```text
跨进程执行一棵运行时计划树时，不能共享运行时指针；
要把计划身份、参数值、输出通道、动态共享 heap 和节点级共享状态拆成明确的序列化/attach 协议。
```
