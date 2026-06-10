# PostgreSQL parallel rescan、instrumentation 汇总与 cleanup ordering

## 课程定位

前置知识：已经理解 `ExecInitParallelPlan()` 的 executor DSM 布局、`Gather` / `GatherMerge` 的 tuple routing，以及 `WaitForParallelWorkersToFinish()` 的消息收尾语义。

本节唯一主问题：

```text
ExecParallelReinitialize() 如何支持重新发起一批 worker，
ExecParallelFinish() / ExecParallelCleanup() 为什么要先 detach tuple queue、再等待 worker、
再汇总 buffer / WAL / JIT / instrumentation，最后释放 DSA 和 ParallelContext？
```

核心矛盾：一个 parallel plan subtree 可能被 rescan，多批 worker 复用同一个 DSM；但 worker 上一轮的 tuple queue、PARAM_EXEC、node shared state、usage 和 instrumentation 都必须在正确时机清理或汇总。过早释放 DSM 会丢指标，过晚 detach tuple queue 会让不再需要的 worker 继续阻塞，rescan 时不重置 shared state 会污染下一轮执行。

学完后应能判断：rescan、finish 和 cleanup 分别负责重置哪些状态，哪些指标必须等 worker finish 后、DSM 释放前汇总，以及提前停止消费 tuple 时哪些 cleanup 顺序仍不能省略。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

本节收束 ParallelContext 与 executor parallel 基础设施：

```text
ExecInitParallelPlan()
  -> LaunchParallelWorkers()
  -> Gather / GatherMerge consume tuple
  -> ExecParallelFinish()
  -> ExecParallelCleanup()

rescan:
  -> ExecParallelReinitialize()
  -> LaunchParallelWorkers() again
```

前几节分别讲启动、消息、tuple routing。本节讲复用和收尾。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
rescan 时 Gather 先 ExecParallelFinish() 关闭上一批 worker，
再把 node->initialized 置 false；
下一次执行调用 ExecParallelReinitialize()，它复用同一个 ParallelContext / DSM，
重建 error queues 和 tuple queues，释放旧 PARAM_EXEC 并序列化新参数，
调用各 parallel-aware node 的 ReInitializeDSM callback；
执行结束时 ExecParallelFinish() 先 detach tuple queues、销毁 readers、等待 worker finish、汇总 Buffer/WAL usage；
ExecParallelCleanup() 再汇总 instrumentation/JIT，释放 PARAM_EXEC、detach DSA，最后 DestroyParallelContext()。
```

tension：

```text
DSM 复用能减少重复初始化成本
  vs
每一轮 worker 的队列、参数、共享进度和指标必须有清晰边界

用户可能提前停止消费 tuple
  vs
worker 仍可能运行、报错或写 usage/instrumentation

指标要在 DSM 释放前取出
  vs
worker 必须先结束，否则指标可能不完整
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/executor/execParallel.c` | `ExecParallelReinitialize()`、`ExecParallelFinish()`、`ExecParallelCleanup()`、instrumentation retrieve。 |
| 2 | `src/backend/access/transam/parallel.c` | `ReinitializeParallelDSM()`、`WaitForParallelWorkersToFinish()`、`DestroyParallelContext()`。 |
| 3 | `src/backend/executor/nodeGather.c` | `ExecReScanGather()`、`ExecShutdownGatherWorkers()`。 |
| 4 | `src/backend/executor/nodeGatherMerge.c` | `ExecReScanGatherMerge()`、tuple clearing、merge state reset。 |
| 5 | `src/backend/executor/instrument.c` / `jit` 相关文件 | `InstrAggNode()`、JIT instrumentation 汇总。 |
| 6 | plan node 源码 | `Exec*ReInitializeDSM()` callbacks。 |

## 4. 关键数据结构与状态

### `ParallelExecutorInfo.finished`

`finished` 是 `ExecParallelFinish()` 的幂等保护：

```text
if (pei->finished)
  return;
```

它表示：

```text
tuple queues 已 detach
readers 已销毁
worker finish 已等待
Buffer/WAL usage 已汇总
```

它不表示：

```text
instrumentation 已取回
DSA 已 detach
ParallelContext 已销毁
```

这些属于 `ExecParallelCleanup()`。

### `FixedParallelExecutorState.param_exec`

rescan 时会变化：

```text
old fpes->param_exec
  -> dsa_free()
  -> InvalidDsaPointer
  -> SerializeParamExecParams()
  -> new dsa_pointer
```

`pei->param_exec` 和 `fpes->param_exec` 必须保持一致，否则 cleanup 或 worker restore 会使用错指针。

### node shared DSM

parallel-aware node 的 shared state 分三类：

| 类型 | rescan 时处理 |
| --- | --- |
| 共享进度状态 | 需要 `Exec*ReInitializeDSM()` 重置。 |
| instrumentation state | 可能无需重置，或由 instrumentation callbacks 管理。 |
| 一次性结构 | 已在初始 DSM 中创建，rescan 只重置内容。 |

源码中有些节点明确：

```text
these nodes have DSM state, but no reinitialization is required
```

所以不能机械认为所有 DSM state 都要重建。

## 5. 主流程源码 walkthrough：rescan

`ExecReScanGather()`：

```text
ExecShutdownGatherWorkers(node)
node->initialized = false

if gather->rescan_param >= 0:
  outerPlan->chgParam = bms_add_member(...)

if outerPlan->chgParam == NULL:
  ExecReScan(outerPlan)
```

`ExecShutdownGatherWorkers()`：

```text
if node->pei != NULL:
  ExecParallelFinish(node->pei)
pfree local reader array
node->reader = NULL
```

注意这里只 finish，不 cleanup。`ParallelExecutorInfo` 和 DSM 保留，用于下一轮 `ExecParallelReinitialize()`。

下一次 `ExecGather()`：

```text
if !node->initialized:
  if node->pei == NULL:
    ExecInitParallelPlan()
  else:
    ExecParallelReinitialize(outerPlanState(node), node->pei, initParam)

  LaunchParallelWorkers()
  ExecParallelCreateReaders()
  node->initialized = true
```

`ExecParallelReinitialize()`：

```text
Assert(pei->finished)
ExecSetParamPlanMulti(sendParams, per_tuple_context)
ReinitializeParallelDSM(pei->pcxt)
pei->tqueue = ExecParallelSetupTupleQueues(pei->pcxt, true)
pei->reader = NULL
pei->finished = false

fpes = shm_toc_lookup(PARALLEL_KEY_EXECUTOR_FIXED)
if fpes->param_exec valid:
  dsa_free(pei->area, fpes->param_exec)
  fpes->param_exec = InvalidDsaPointer

if sendParams not empty:
  pei->param_exec = SerializeParamExecParams(...)
  fpes->param_exec = pei->param_exec

estate->es_query_dsa = pei->area
ExecParallelReInitializeDSM(planstate, pei->pcxt)
estate->es_query_dsa = NULL
```

`ReinitializeParallelDSM()` 来自 `parallel.c`，会：

```text
WaitForParallelWorkersToFinish()
WaitForParallelWorkersToExit()
nworkers_launched = 0
free known_attached_workers
reset FixedParallelState.last_xlog_end
recreate error queues
```

这说明 executor reinitialize 复用了同一个 DSM segment，但重建了 per-round queues 和 state。

## 6. `ReInitializeDSM` 与 `ReScan` 的分工

`nodeGather.c` 注释给出很好的规则：

```text
ReInitializeDSM should reset only shared state,
ReScan should reset only local state,
anything that depends on both being finished must wait until first ExecProcNode call.
```

这条规则解决一个 race-like ordering 问题：

```text
shared state:
  worker 和 leader 都可能访问，必须在新 worker 启动前重置

local state:
  当前 backend 的 PlanState 内部字段，由 ExecReScan() 重置

dependent state:
  同时依赖两者，不能在其中一个阶段提前假设另一个已经完成
```

对于新增 parallel-aware node，这是一条实现指南。不要在 `Exec*ReInitializeDSM()` 中偷偷重置 leader-local scan position，也不要在 `ExecReScan()` 中改 DSM 共享进度。

## 7. 主流程源码 walkthrough：finish

`ExecParallelFinish()`：

```text
nworkers = pei->pcxt->nworkers_launched
if pei->finished:
  return

if pei->tqueue != NULL:
  for each worker:
    shm_mq_detach(pei->tqueue[i])
  pfree(pei->tqueue)
  pei->tqueue = NULL

if pei->reader != NULL:
  for each reader:
    DestroyTupleQueueReader(reader)
  pfree(pei->reader)
  pei->reader = NULL

WaitForParallelWorkersToFinish(pei->pcxt)

for each worker:
  InstrAccumParallelQuery(&buffer_usage[i], &wal_usage[i])

pei->finished = true
```

### 7.1 为什么先 detach tuple queues

leader 可能不再需要 tuple：

```text
上层 LIMIT 满足
cursor 停止
executor shutdown
ERROR cleanup
```

如果不 detach tuple queues，worker 可能继续向已无人消费的 queue 写 tuple，最终阻塞在 `shm_mq_send()`。提前 detach 是告诉 worker：

```text
结果不再需要，尽快停止 tuple 输出并走 cleanup。
```

### 7.2 为什么 destroy readers 可以在等待 worker 时做

`DestroyTupleQueueReader()` 只释放 reader wrapper，不 detach queue。queue 已经在 `pei->tqueue` 分支 detach。reader 是 leader-local 辅助对象，不需要等 worker 结束。

### 7.3 为什么 Buffer/WAL usage 必须等 worker finish

worker 在 `ParallelQueryMain()` 末尾：

```text
InstrEndParallelQuery(&buffer_usage[worker], &wal_usage[worker])
```

如果 leader 提前读取：

```text
worker 还在执行
usage 仍在变化
读到不完整指标
```

因此 usage 汇总必须在 `WaitForParallelWorkersToFinish()` 后。

## 8. 主流程源码 walkthrough：cleanup

`ExecParallelCleanup()`：

```text
if instrumentation:
  ExecParallelRetrieveInstrumentation(planstate, instrumentation)

if jit_instrumentation:
  ExecParallelRetrieveJitInstrumentation(planstate, jit_instrumentation)

if param_exec valid:
  dsa_free(pei->area, pei->param_exec)
  pei->param_exec = InvalidDsaPointer

if area != NULL:
  dsa_detach(pei->area)
  pei->area = NULL

if pcxt != NULL:
  DestroyParallelContext(pei->pcxt)
  pei->pcxt = NULL

pfree(pei)
```

cleanup 必须晚于 finish，因为 instrumentation 和 JIT 数据在 worker DSM 中写入，worker 必须已经结束。

但 instrumentation retrieve 又必须早于 `DestroyParallelContext()`：

```text
DestroyParallelContext()
  -> dsm_detach()
  -> TOC / instrumentation memory invalid
```

所以顺序不能改成“先 destroy context，再汇总指标”。

## 9. 生命周期 / ownership / cleanup 与正确性机制层次

这条链路的 ownership 边界可以压缩成三层：

| 阶段 | owner | 必须保留的状态 | 正确性边界 |
| --- | --- | --- | --- |
| rescan | `ParallelExecutorInfo` 继续持有 `ParallelContext` / DSM / DSA。 | fixed state、TOC、DSA area、node shared state。 | 旧 worker 已 finish，新 `PARAM_EXEC` 和 node shared state 必须重新写入。 |
| finish | leader 仍持有 DSM，worker 逐步退出。 | tuple queues、error queues、usage arrays。 | 先 detach tuple queue，再等 worker finish，避免遗漏最后 ERROR / usage。 |
| cleanup | leader 最后释放 executor 并行资源。 | instrumentation、JIT、DSA param、`ParallelContext`。 | 指标先从 DSM 复制到 query context，再释放 DSA 和 destroy context。 |

相关机制各管一层，不要互相替代：

```text
shm_mq detach:
  告诉 worker tuple stream 不再被消费

WaitForParallelWorkersToFinish():
  等 worker 结束并处理 parallel messages

ExecParallelRetrieveInstrumentation():
  在 DSM 还活着时复制 worker 明细

DestroyParallelContext():
  最后释放 worker handle、error queue、DSM 和 pcxt_list ownership
```

因此，本节的正确性来自 ordering：finish 不能晚于 cleanup，instrumentation retrieve 不能晚于 DSM destroy，rescan 不能复用上一轮 worker-local progress。

## 10. instrumentation 汇总

worker 侧：

```text
ExecParallelReportInstrumentation(queryDesc->planstate, instrumentation)
  -> for each PlanState
     -> find plan_node_id index
     -> instrument += worker slot offset
     -> InstrAggNode(&instrument[ParallelWorkerNumber], planstate->instrument)
```

leader cleanup：

```text
ExecParallelRetrieveInstrumentation(planstate, instrumentation)
  -> find plan_node_id
  -> for each worker:
       InstrAggNode(planstate->instrument, &instrument[n])
  -> allocate worker_instrument in es_query_cxt
  -> copy per-worker detail
  -> call node-specific RetrieveInstrumentation()
```

两个输出：

1. 聚合到 leader planstate 的总 instrumentation。
2. 保存 per-worker detail，供 EXPLAIN 显示 worker 明细。

### 10.1 为什么按 plan_node_id 匹配

leader 和 worker 是两棵不同 `PlanState`，指针地址不同。不能用 pointer identity，只能用 plan node id：

```text
plan_node_id -> instrumentation array offset
```

如果 plan node id 缺失或不一致，retrieve 会 ERROR。

### 10.2 为什么分 per-query context 分配

worker instrumentation 明细需要活到 EXPLAIN 输出阶段，而不是只在 cleanup 函数栈中有效。因此：

```text
MemoryContextSwitchTo(planstate->state->es_query_cxt)
palloc worker_instrument
```

这和课程早先讲的 MemoryContext 生命周期一致。

## 11. JIT instrumentation 汇总

worker 侧如果有 JIT：

```text
jit_instrumentation->jit_instr[ParallelWorkerNumber] =
  queryDesc->estate->es_jit->instr
```

leader：

```text
ExecParallelRetrieveJitInstrumentation()
  -> allocate es_jit_worker_instr if needed
  -> for each worker: InstrJitAgg()
  -> copy per-worker SharedJitInstrumentation to planstate->worker_jit_instrument
```

JIT 指标和普通 plan node instrumentation 分开，因为它属于 executor/JIT 子系统，不是一棵 plan tree 上每个节点都有的 `NodeInstrumentation`。

## 12. 错误路径 / 异常路径 / fallback

### 12.1 finish 被重复调用

`ExecParallelFinish()` 幂等：

```text
if (pei->finished)
  return
```

这是必要的，因为 `ExecShutdownGatherWorkers()` 可能从正常结束、rescan、executor shutdown 多个路径进入。

### 12.2 cleanup 前未 finish

正常调用方应先 finish 再 cleanup。`ExecShutdownGather()` 做：

```text
ExecShutdownGatherWorkers()
ExecParallelCleanup()
```

如果扩展或新节点绕过 finish 直接 cleanup，可能：

```text
worker 未结束
usage/instrumentation 不完整
DestroyParallelContext() 强杀 worker
```

### 12.3 worker ERROR during finish

`WaitForParallelWorkersToFinish()` 会处理 parallel messages。如果 worker ERROR，leader 在 finish 中抛出 ERROR。随后事务 abort cleanup 会走 `DestroyParallelContext()` 兜底。

这解释了为什么 finish 不是“无错误清理函数”。它仍可能把 worker late ERROR 反馈给用户。

### 12.4 instrumentation retrieve 失败

如果 plan node id 找不到：

```text
elog(ERROR, "plan node %d not found")
```

这通常说明 estimate/init/report/retrieve 的 plan tree 不一致，属于 executor bug 或扩展 custom scan bug。

### 12.5 rescan 中 PARAM_EXEC 序列化失败

如果新参数 datum 无法序列化或 DSA 分配失败，rescan ERROR。旧 worker 已 finish，旧 param pointer 已 free 或即将 free；事务 cleanup 会销毁整个 context。

## 13. 成本、资源与跨模块传播

finish / cleanup 成本包括：

```text
detach tuple queues: O(workers)
destroy readers: O(workers)
wait worker finish: runtime dependent
Buffer/WAL usage accumulation: O(workers)
instrumentation retrieval: O(plan_nodes * workers)
JIT aggregation: O(workers)
node-specific retrieve callbacks
DSA detach and context destroy
```

对大 plan tree 和多 worker，instrumentation 汇总成本可见，尤其是 `EXPLAIN ANALYZE` 场景。普通执行如果没有 instrumentation，就跳过大部分 retrieve。

跨模块：

```text
Gather/GatherMerge:
  决定何时 finish / rescan / cleanup

execParallel:
  维护 executor DSM 和 usage/instrumentation

parallel.c:
  维护 worker finish / exit / DSM lifecycle

instrument / jit:
  定义指标聚合语义

DSA:
  管理动态 PARAM_EXEC 和 node shared object
```

## 14. 观测与诊断入口

| 入口 | 能看到什么 |
| --- | --- |
| `EXPLAIN ANALYZE` | worker instrumentation 是否汇总，per-worker details 是否存在。 |
| `EXPLAIN (ANALYZE, BUFFERS, WAL, JIT)` | Buffer/WAL/JIT 指标是否包含 worker。 |
| `pg_stat_activity.wait_event = ParallelFinish` | leader 正在 finish 等 worker。 |
| gdb `pei->finished` | finish 是否已完成，rescan 是否允许。 |
| gdb `fpes->param_exec` | 当前 PARAM_EXEC DSA pointer。 |
| gdb `planstate->worker_instrument` | cleanup 后 per-worker instrumentation 是否复制到 query context。 |

断点：

```gdb
break ExecParallelReinitialize
break ReinitializeParallelDSM
break ExecParallelFinish
break WaitForParallelWorkersToFinish
break ExecParallelRetrieveInstrumentation
break ExecParallelCleanup
```

## 15. 常见误区

1. 误以为 rescan 必须重新创建 DSM。实际复用 DSM，重置 queues、params 和 node shared state。
2. 误以为 `ExecParallelFinish()` 会释放 `ParallelContext`。它只 finish worker 和 usage，释放在 cleanup。
3. 误以为 instrumentation 可以在 worker 未结束时读取。那会读到不完整数据。
4. 误以为 `ReInitializeDSM` 和 `ReScan` 可以随意互相重置状态。一个管 shared state，一个管 local state。
5. 误以为 `DestroyParallelContext()` 之前可以 detach DSA。node instrumentation 或 param cleanup 可能仍需要 DSA / DSM。

## 16. 课堂实验

### 16.1 rescan 路径

构造会 rescan inner side 的计划，例如 nested loop + Gather 子计划，或用 cursor 重复扫描。gdb：

```gdb
break ExecReScanGather
break ExecParallelReinitialize
break ReinitializeParallelDSM
break ExecParallelReInitializeDSM
```

观察：

```gdb
p pei->finished
p fpes->param_exec
p pei->pcxt->nworkers_launched
```

### 16.2 finish / cleanup 顺序

```gdb
break ExecParallelFinish
break ExecParallelCleanup
break DestroyParallelContext
commands ExecParallelFinish
  silent
  printf "finish: tqueue=%p reader=%p finished=%d\\n", pei->tqueue, pei->reader, pei->finished
  continue
end
```

确认 finish 先于 cleanup，cleanup 先 retrieve instrumentation 再 destroy context。

### 16.3 instrumentation

```sql
EXPLAIN (ANALYZE, VERBOSE, BUFFERS, WAL)
SELECT count(*) FROM t_parallel_ctx WHERE id > 10;
```

在 `ExecParallelRetrieveInstrumentation()` 中断，打印：

```gdb
p instrumentation->num_workers
p instrumentation->num_plan_nodes
p planstate->plan->plan_node_id
```

### 16.4 JIT

打开 JIT 后执行并行查询：

```sql
SET jit = on;
SET jit_above_cost = 0;
EXPLAIN (ANALYZE, JIT)
SELECT sum(id) FROM t_parallel_ctx;
```

在 `ExecParallelRetrieveJitInstrumentation()` 设断点，观察 worker JIT 指标汇总。

## 17. 讨论题

1. 为什么 `ExecParallelFinish()` 要先 detach tuple queue，再等待 worker？
2. `ExecParallelFinish()` 和 `ExecParallelCleanup()` 为什么拆成两个函数？
3. rescan 时哪些状态必须重置，哪些状态应该保留？
4. 为什么 instrumentation 用 plan node id 匹配，而不是 PlanState 指针？
5. 如果新增 parallel-aware node，把 shared state reset 放在 `ExecReScan()` 会有什么问题？

## 18. 源码索引：finish / cleanup 状态变化

| 阶段 | `tqueue` | `reader` | worker | usage | instrumentation | DSA | pcxt |
| --- | --- | --- | --- | --- | --- | --- | --- |
| running | attached | active | running | worker-local | worker-local / DSM slots | attached | active |
| `ExecParallelFinish()` start | detach | destroy | asked to finish via detached queue | incomplete | incomplete | attached | active |
| after `WaitForParallelWorkersToFinish()` | NULL | NULL | control messages drained | complete | written by worker | attached | active |
| after usage accumulation | NULL | NULL | finished | accumulated | still in DSM | attached | active |
| `ExecParallelCleanup()` instrumentation | NULL | NULL | finished | accumulated | copied to leader | attached | active |
| DSA cleanup | NULL | NULL | finished | accumulated | copied | detached | active |
| context destroy | NULL | NULL | exited / waited | accumulated | copied | detached | destroyed |

这张表说明为什么 cleanup 不能简单提前：

```text
instrumentation 需要 DSM 仍在；
DSA param cleanup 需要 area 仍 attach；
DestroyParallelContext() 会 detach DSM；
worker exit wait 需要 bgworker handles；
```

## 19. 源码检查清单：新增并行节点的 rescan 支持

新增 parallel-aware node 时，rescan 支持经常被漏掉。检查：

```text
节点是否有 shared cursor / shared block allocator / shared hash state？
这些 shared state 是否跨 worker 批次复用？
下一轮 worker 启动前是否必须归零？
leader local PlanState 是否也有独立 rescan 状态？
ReInitializeDSM 和 ExecReScan 是否各司其职？
如果 sendParams 变化，worker 是否能看到新参数？
```

### 19.1 只重置 shared state 的例子

```text
parallel seq scan shared block counter
parallel append shared worker allocation
parallel bitmap heap shared iterator
parallel hash join phase / batch state
```

这些必须在 worker launch 前 reset，否则新 worker 会接着上一轮的进度跑。

### 19.2 只重置 local state 的例子

```text
leader PlanState 当前 slot
表达式上下文
本地 tuple cache
当前 outer tuple / inner tuple
```

这些不应放进 `Exec*ReInitializeDSM()`，否则 worker 看不到或会造成跨进程不一致。

### 19.3 两者都依赖的状态

如果某个状态同时依赖 shared reset 和 local rescan，不要在任一阶段提前消费它。让第一次 `ExecProcNode()` 在两者完成后初始化。

这正是 `nodeGather.c` 注释给出的规则。

## 20. 故障模式速查表

| 现象 | 可能原因 | 排查 |
| --- | --- | --- |
| rescan 后结果重复 | shared progress 未 reset | `ExecParallelReInitializeDSM()` callback。 |
| rescan 后结果缺失 | shared cursor 仍停在末尾 | node-specific reinitialize。 |
| rescan 后参数还是旧值 | old `param_exec` 未 free / new pointer 未写 | `fpes->param_exec`、DSA contents。 |
| EXPLAIN worker 明细缺失 | cleanup 前 DSM 已销毁 | `ExecParallelRetrieveInstrumentation()` 是否执行。 |
| Buffers/WAL 偏低 | usage 在 worker finish 前汇总 | `ExecParallelFinish()` 顺序。 |
| worker 卡住 | tuple queue 未 detach | finish 是否先 detach `pei->tqueue`。 |
| cleanup ERROR 后 context 泄漏 | destroy 未执行或重复销毁 | `pcxt_list`、`pei->pcxt`。 |

## 21. 诊断实验：提前停止消费

构造 LIMIT：

```sql
SET max_parallel_workers_per_gather = 4;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM t_parallel_ctx LIMIT 10;
```

观察点：

```text
上层很快拿到 10 行
worker 可能仍已启动
ExecParallelFinish() detach tuple queues
worker 看到 queue detached 后停止发送
WaitForParallelWorkersToFinish() 仍要处理最后消息
```

gdb：

```gdb
break ExecParallelFinish
break shm_mq_detach
break WaitForParallelWorkersToFinish
break ExecParallelCleanup
```

这个实验能说明：

```text
不再需要 tuple
  !=
可以跳过 worker finish
```

## 22. 本节小结

并行执行的结束不是一个简单 free。`ExecParallelReinitialize()` 支持同一 parallel DSM 重新发起 worker 批次；`ExecParallelFinish()` 负责停止 tuple 传输、等待 worker、汇总 Buffer/WAL usage；`ExecParallelCleanup()` 在 DSM 仍有效时取回 instrumentation/JIT，释放 DSA 参数，最后销毁 `ParallelContext`。

可迁移规律：

```text
并行执行收尾要把“停止产出、等待工作者、汇总指标、释放动态对象、销毁上下文”拆成有序阶段；
否则要么丢错误和指标，要么让 worker 卡在无人消费的队列上，要么在下一轮执行中复用脏 shared state。
```
