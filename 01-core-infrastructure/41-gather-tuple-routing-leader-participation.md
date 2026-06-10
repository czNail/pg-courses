# PostgreSQL Gather / GatherMerge tuple routing 与 leader participation

## 课程定位

前置知识：已经理解 executor parallel DSM、tuple queue、worker bootstrap 和 parallel-aware plan node 的共享状态初始化。

本节唯一主问题：

```text
ExecGather() / ExecGatherMerge() 为什么延迟到第一次执行才启动 worker，
ExecParallelSetupTupleQueues()、ExecParallelCreateReaders()、TupleQueueReader
和 parallel_leader_participation 如何在 worker tuple stream、leader 本地执行、背压和有序 merge 之间取舍？
```

核心矛盾：并行计划需要把多个 worker 产生的 tuple 汇聚回单个 leader 输出，但 worker 启动和 DSM 分配成本很高，worker 数可能不足，tuple queue 可能背压，leader 是否也执行子计划会影响吞吐和延迟。`Gather` 选择无序 funnel，`GatherMerge` 选择保持有序 merge，它们都必须在 worker stream 与 leader local execution 之间维持 executor 的 pull 模型。

学完后应能判断：`Gather` 和 `GatherMerge` 何时启动 worker、何时只读 worker queue、何时由 leader 本地执行，以及 queue 背压、少 worker 和 leader participation 分别会把成本推到哪里。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

上一节讲了 `ExecInitParallelPlan()` 如何准备 worker 执行计划所需的 DSM：

```text
plan / params / tuple queue / DSA / instrumentation
```

本节进入顶层消费节点：

```text
Gather:
  多个 worker queue + leader local scan
  -> 任意顺序返回 tuple

GatherMerge:
  多个有序 worker queue + leader local sorted stream
  -> merge 后按 pathkeys 返回 tuple
```

下一节会讲 rescan、finish、instrumentation 和 cleanup ordering。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
Gather / GatherMerge 在第一次 ExecProcNode() 时才调用 ExecInitParallelPlan() 和 LaunchParallelWorkers()；
worker 通过 tuple queue 把 MinimalTuple 发回 leader；
leader 用 ExecParallelCreateReaders() 建立 TupleQueueReader 数组，
每次被上层拉取 tuple 时，先尝试从 worker queues 非阻塞读取，
必要时自己执行 outerPlan；
Gather 直接 funnel 任意 worker 的 tuple，
GatherMerge 则为每个 reader 维护 slot 并用 sort key heap 做有序归并。
```

tension：

```text
越早启动 worker，越可能隐藏启动延迟
  vs
如果计划节点根本不被执行，提前创建 DSM 和 worker 就是浪费

leader 参与执行可增加 CPU 并行度
  vs
leader 同时负责读 queue、投影、上层输出和错误处理，参与过多可能反而拖慢汇聚

worker queue 保持背压可限制内存
  vs
queue 满会阻塞 worker，queue 空会让 leader 等待
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/executor/nodeGather.c` | `ExecGather()`、`gather_getnext()`、`gather_readnext()`、shutdown/rescan。 |
| 2 | `src/backend/executor/nodeGatherMerge.c` | `ExecGatherMerge()`、reader 初始化、merge heap、有序读取。 |
| 3 | `src/backend/executor/execParallel.c` | `ExecParallelSetupTupleQueues()`、`ExecParallelCreateReaders()`、`ExecParallelGetReceiver()`。 |
| 4 | `src/backend/executor/tqueue.c` | `CreateTupleQueueDestReceiver()`、`TupleQueueReaderNext()`。 |
| 5 | `src/include/nodes/execnodes.h` | `GatherState`、`GatherMergeState` 中 reader、nreaders、need_to_scan_locally 字段。 |
| 6 | `src/backend/optimizer/plan/planner.c` / `costsize.c` | `parallel_leader_participation` GUC 和成本估算。 |

## 4. 关键数据结构与状态

### `GatherState`

核心字段：

| 字段 | 语义 |
| --- | --- |
| `initialized` | 是否已经启动/初始化 parallel context。 |
| `pei` | `ParallelExecutorInfo`，包含 `pcxt`、tuple queues、DSA。 |
| `nworkers_launched` | 实际成功注册 worker 数，供 EXPLAIN 和执行逻辑使用。 |
| `nreaders` | 当前仍活跃的 tuple queue readers 数。 |
| `reader` | 当前工作 reader 数组，可能随 reader done 被压缩。 |
| `nextreader` | round-robin 读取起点。 |
| `need_to_scan_locally` | leader 是否还要本地执行 outer plan。 |
| `funnel_slot` | worker MinimalTuple 存入 leader slot 的中转槽。 |

### `GatherMergeState`

除了类似字段，还维护：

```text
gm_slots[]:
  每个 worker/leader stream 当前 tuple

gm_sortkeys:
  merge 比较键

binary heap / slot index:
  选择下一个全局最小 tuple
```

`GatherMerge` 的核心不是单纯读 queue，而是把多个已排序 stream 做 k-way merge。

### `TupleQueueReader`

`TupleQueueReader` 是 `tqueue.c` 内部 opaque struct，只保存：

```text
shm_mq_handle *queue
```

读取：

```text
TupleQueueReaderNext(reader, nowait, &done)
  -> shm_mq_receive()
  -> success: return MinimalTuple
  -> would block: return NULL, done=false
  -> detached: return NULL, done=true
```

返回 tuple 指针在下一次读取后失效，因此 `GatherMerge` 如果要缓存 tuple 必须 copy。

## 5. 主流程源码 walkthrough：延迟启动 worker

`ExecGather()` 的第一段：

```text
CHECK_FOR_INTERRUPTS()

if !node->initialized:
  estate = node->ps.state
  gather = node->ps.plan

  if gather->num_workers > 0 && estate->es_use_parallel_mode:
    if node->pei == NULL:
      node->pei = ExecInitParallelPlan(...)
    else:
      ExecParallelReinitialize(...)

    pcxt = node->pei->pcxt
    LaunchParallelWorkers(pcxt)
    node->nworkers_launched = pcxt->nworkers_launched
    estate counters += planned / launched

    if launched > 0:
      ExecParallelCreateReaders(node->pei)
      copy pei->reader into node->reader
      node->nreaders = launched
    else:
      node->nreaders = 0
      node->reader = NULL

  node->need_to_scan_locally =
    (node->nreaders == 0)
    || (!gather->single_copy && parallel_leader_participation)

  node->initialized = true
```

为什么不在 `ExecInitGather()` 里启动？

```text
ExecInitNode() 只是初始化计划树；
上层 Limit、runtime partition pruning、one-time qual、cursor count 等可能导致这个 Gather 永远不执行。
```

DSM 分配和 worker 启动都很重，只有第一次真正拉取 tuple 时才启动，避免无用资源消耗。

## 6. tuple queue 创建与 reader attach

leader 在 `ExecInitParallelPlan()` 中调用：

```text
ExecParallelSetupTupleQueues(pcxt, false)
```

它做：

```text
if pcxt->nworkers == 0:
  return NULL

allocate PARALLEL_TUPLE_QUEUE_SIZE * pcxt->nworkers from TOC
for each worker slot:
  mq = shm_mq_create(offset, PARALLEL_TUPLE_QUEUE_SIZE)
  shm_mq_set_receiver(mq, MyProc)
  responseq[i] = shm_mq_attach(mq, pcxt->seg, NULL)

shm_toc_insert(PARALLEL_KEY_TUPLE_QUEUE, tqueuespace)
```

worker 侧 `ParallelQueryMain()`：

```text
ExecParallelGetReceiver(seg, toc)
  -> lookup PARALLEL_KEY_TUPLE_QUEUE
  -> mqspace += ParallelWorkerNumber * PARALLEL_TUPLE_QUEUE_SIZE
  -> shm_mq_set_sender(mq, MyProc)
  -> CreateTupleQueueDestReceiver(shm_mq_attach(mq, seg, NULL))
```

leader 在 worker launch 后才：

```text
ExecParallelCreateReaders(pei)
  -> for each launched worker:
       shm_mq_set_handle(pei->tqueue[i], worker[i].bgwhandle)
       pei->reader[i] = CreateTupleQueueReader(pei->tqueue[i])
```

注意 `pcxt->nworkers` 和 `nworkers_launched` 的差异：

```text
tuple queue memory:
  按预算 nworkers 分配

TupleQueueReader:
  只为实际 launched worker 创建
```

注册失败的 worker slot 不会有 reader。

## 7. Gather 的 pull 模型

`ExecGather()` 每次被上层拉一行：

```text
ResetExprContext()
slot = gather_getnext(node)
if NULL: return NULL
if no projection: return slot
else ExecProject()
```

`gather_getnext()`：

```text
while nreaders > 0 || need_to_scan_locally:
  CHECK_FOR_INTERRUPTS()

  if nreaders > 0:
    tup = gather_readnext()
    if tuple:
      ExecStoreMinimalTuple(tup, funnel_slot, false)
      return funnel_slot

  if need_to_scan_locally:
    estate->es_query_dsa = pei ? pei->area : NULL
    outerTupleSlot = ExecProcNode(outerPlan)
    estate->es_query_dsa = NULL
    if tuple:
      return outerTupleSlot
    need_to_scan_locally = false

return clear funnel_slot
```

这里的调度策略：

1. 优先尝试 worker tuple。
2. 如果 worker queues 暂时没 tuple，且 leader 仍应参与，就本地执行一行。
3. 如果 leader 本地也结束，继续等 worker。

这保留 executor pull 模型：上层每次要一行，`Gather` 才做一小步。

## 8. `gather_readnext()` 的背压与公平性

`gather_readnext()` 用 nonblocking read：

```text
tup = TupleQueueReaderNext(reader, true, &readerdone)
```

如果当前 reader 有 tuple，立即返回。否则轮到下一个 reader。

一个重要优化：

```text
不再每读一行就切 reader；
而是尽量继续读当前 queue，直到 would-block。
```

源码注释说这样更高效。原因是：

```text
同一个 worker queue 中可能已有一批 tuple
连续读取减少 reader 切换和 latch 等待
```

当所有 reader 都没有 tuple：

```text
if need_to_scan_locally:
  return NULL  -- 让 leader 本地执行一行
else:
  WaitLatch(... WAIT_EVENT_EXECUTE_GATHER)
```

这个 wait event 表示 leader 正在等 worker 产出、queue 变化或并行消息。

## 9. reader done 与 worker finish

`TupleQueueReaderNext()` 返回：

```text
done = true
tuple = NULL
```

表示 queue detached。`gather_readnext()` 会：

```text
remove reader from active array
nreaders--
if nreaders == 0:
  ExecShutdownGatherWorkers()
  return NULL
```

`ExecShutdownGatherWorkers()` 调用：

```text
ExecParallelFinish(pei)
```

这一步不只是释放 reader。它必须等待 worker finish，并处理最后 error messages / usage accumulation。也就是说：

```text
tuple stream done
  不等于
worker cleanup done
```

## 10. leader participation

`parallel_leader_participation` 是 GUC，默认 on。`Gather` 中：

```text
need_to_scan_locally =
  nreaders == 0
  || (!single_copy && parallel_leader_participation)
```

含义：

| 情况 | leader 是否执行 outer plan |
| --- | --- |
| 没有 worker | 必须执行，否则没有结果。 |
| 有 worker、非 single_copy、GUC on | 执行，增加并行度。 |
| 有 worker、非 single_copy、GUC off | 不执行，只负责汇聚。 |
| single_copy | 通常不参与，除非无 worker。 |

leader participation 的 trade-off：

```text
收益:
  多一个执行者，减少 worker 数不足影响

成本:
  leader 同时要读 queue、处理 projection、上层输出、parallel messages
  leader 忙于本地执行时，worker queue 可能更容易满
```

因此在某些 workload 中关闭 leader participation 反而更稳定，特别是 worker 产出很快、leader 汇聚压力大的场景。

## 11. GatherMerge 的不同

`ExecGatherMerge()` 启动 worker 的逻辑与 `Gather` 相似，但消费 tuple 不同。

### 11.1 有序流前提

`GatherMerge` 的每个 worker 和 leader local path 都必须按相同 sort key 输出。leader 不能任意返回先到的 tuple，而要维护全局顺序。

### 11.2 reader tuple 要 copy

`TupleQueueReaderNext()` 返回的 tuple 在下一次 reader call 后可能失效。`GatherMerge` 需要把每个 stream 的当前 tuple 缓存在 slot 中参与比较，因此：

```text
gm_readnext_tuple()
  -> TupleQueueReaderNext()
  -> heap_copy_minimal_tuple(tup, 0)
```

这是 `Gather` 和 `GatherMerge` 的一个实际成本差异。

### 11.3 merge heap

`GatherMerge` 用 sort support 比较各 stream 当前 slot：

```text
heap_compare_slots()
  -> for each sort key:
       ApplySortComparator()
```

每次返回最小 tuple 后，只从对应 stream 再取下一行，维护 k-way merge。

### 11.4 leader participation in GatherMerge

`GatherMerge` 中：

```text
if parallel_leader_participation || nreaders == 0:
  need_to_scan_locally = true
```

与 `Gather` 相比，它没有 `single_copy` 条件，但本地执行产生的 tuple 也必须加入 merge 结构。leader 本地 stream 是有序输入之一。

## 12. 正确性机制层次

本节的正确性不靠 tuple queue 单独保证，而是由几层边界叠加：

| 层次 | 保证什么 | 不能保证什么 |
| --- | --- | --- |
| executor pull 模型 | 上层每次只消费一个 tuple，`Gather` / `GatherMerge` 能在 worker queue 与本地执行之间切换。 | 不保证 worker 一定及时产出。 |
| shm_mq / TupleQueueReader | bounded queue、sender/receiver detach 和 reader done 语义。 | 不解释 tuple 可见性或排序正确性。 |
| `ExecParallelFinish()` | tuple stream 结束后仍等待 worker finish，处理最后 ERROR 和 usage。 | 不释放 `ParallelContext`，也不替代 cleanup。 |
| sort key / merge heap | `GatherMerge` 维持全局有序输出。 | 不消除 copy、compare 和 leader merge 成本。 |
| `CHECK_FOR_INTERRUPTS()` | leader 在拉取过程中处理 parallel message。 | 不把普通 tuple data 变成 error queue 消息。 |

因此，`Gather` 的主边界是“tuple stream 是否还能返回数据”，`GatherMerge` 还多一个“每个 stream 当前 tuple 是否仍能参与全局比较”的顺序边界。

## 13. 错误路径 / 异常路径 / fallback

### 13.1 worker 未启动

`nworkers_launched == 0`：

```text
node->nreaders = 0
node->need_to_scan_locally = true
```

查询仍可由 leader 完成。

### 13.2 worker 初始化失败

`TupleQueueReaderNext()` 对未初始化 worker 可能返回 NULL / done。`Gather` 注释明确：

```text
先把它当成没有产出 tuple；
WaitForParallelWorkersToFinish() 会在后面报错。
```

这避免 tuple routing 路径直接承担 bgworker startup diagnosis。

### 13.3 leader 提前停止消费

上层 Limit 或客户端停止接收可能导致 leader 不再需要 worker tuple。`ExecParallelFinish()` 会尽快 detach tuple queues：

```text
worker 发现 tuple queue detached
  -> 停止发送 tuple
  -> finish / cleanup
```

### 13.4 queue full 背压

worker `tqueueReceiveSlot()`：

```text
shm_mq_send(queue, tuple->t_len, tuple, false, false)
```

如果 queue 满，worker 会等待 reader 进展。leader 如果忙于本地执行或上层输出，worker 可能被背压限制。这是 bounded memory 的代价。

### 13.5 GatherMerge 顺序错误

如果 worker 输入并非按预期 pathkeys 排序，`GatherMerge` 不会重新排序全部数据，只按比较器 merge 当前 stream。错误会表现为全局输出顺序不正确。这是 planner/executor contract，不能靠 GatherMerge 自行修复。

## 14. 成本、资源与跨模块传播

资源：

```text
每 worker 一个 tuple queue: PARALLEL_TUPLE_QUEUE_SIZE
leader 每 worker 一个 TupleQueueReader
Gather 一个 funnel_slot
GatherMerge 每 stream slot + tuple copy + merge heap
leader local execution 需要共享 DSA area
```

成本来源：

```text
worker startup
tuple serialization to MinimalTuple
shm_mq send/receive
leader projection
queue wakeup / latch
GatherMerge comparison and copying
leader participation 调度
```

跨模块：

```text
planner:
  选择 Gather 或 GatherMerge，决定 num_workers、single_copy、pathkeys

execParallel:
  准备 worker plan、tuple queues、readers

tqueue:
  tuple transport over shm_mq

nodeGather/nodeGatherMerge:
  汇聚和 leader participation 策略
```

## 15. 观测与诊断入口

| 入口 | 能看到什么 |
| --- | --- |
| `EXPLAIN ANALYZE` | `Gather` / `Gather Merge`、Workers Planned/Launched、per-worker rows。 |
| `parallel_leader_participation` | 改变 leader 是否执行 outer plan。 |
| `pg_stat_activity.wait_event = ExecuteGather` | leader 等待 worker tuple / queue 进展。 |
| gdb `node->nreaders` / `need_to_scan_locally` | leader 当前汇聚状态。 |
| gdb `TupleQueueReaderNext()` | queue 是否 would-block / detached。 |
| perf | leader CPU 是否花在 tuple deform/projection/merge compare。 |

实验 SQL：

```sql
SET max_parallel_workers_per_gather = 4;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM t_parallel_ctx WHERE id > 0;

SET parallel_leader_participation = off;
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM t_parallel_ctx WHERE id > 0;
```

比较 worker rows、leader 行为和总耗时。

## 16. 常见误区

1. 误以为 `Gather` 在 `ExecInitNode()` 时启动 worker。实际第一次 `ExecGather()` 才启动。
2. 误以为 `Gather` 会保持 worker 输出顺序。普通 `Gather` 是无序 funnel。
3. 误以为 worker 失败会在 `TupleQueueReaderNext()` 立即精确报错。精确错误通常在 parallel message / finish 中处理。
4. 误以为 leader participation 永远更快。leader 也承担汇聚和上层输出，参与执行可能加重背压。
5. 误以为 tuple queue 和 error queue 可以共用。它们协议、大小和生命周期不同。

## 17. 课堂实验

### 17.1 延迟启动观察

gdb：

```gdb
break ExecInitGather
break ExecGather
break ExecInitParallelPlan
break LaunchParallelWorkers
```

确认 `ExecInitGather` 不启动 worker，第一次 `ExecGather` 才启动。

### 17.2 reader 状态

```gdb
break ExecParallelCreateReaders
break gather_readnext
commands gather_readnext
  silent
  p gatherstate->nreaders
  p gatherstate->nextreader
  p gatherstate->need_to_scan_locally
  continue
end
```

观察 reader 被 done 后从数组中删除。

### 17.3 GatherMerge copy 成本

对带 `ORDER BY` 的并行查询设置断点：

```gdb
break gm_readnext_tuple
break heap_compare_slots
```

观察 worker tuple 被 copy 到 merge slot，并参与 sort key 比较。

### 17.4 queue 背压

在 `tqueueReceiveSlot()` 和 `TupleQueueReaderNext()` 设断点，或者用 perf 观察 worker 是否卡在 `shm_mq_send()`。如果 leader 慢，worker 会等待 queue 空间。

## 18. 讨论题

1. 为什么 worker 启动延迟到第一次执行，而不是 node init？
2. `Gather` 为什么先尝试 worker queue，再让 leader local scan？换顺序会怎样？
3. `parallel_leader_participation` 对 CPU-heavy 和 tuple-output-heavy 查询的影响为什么可能相反？
4. 为什么 `GatherMerge` 必须 copy worker tuple，而普通 `Gather` 可以直接存入 funnel slot？
5. queue 满造成 worker 阻塞，是缺陷还是必要的背压机制？

## 19. 源码索引：Gather 和 GatherMerge 字段对照

| 语义 | Gather | GatherMerge |
| --- | --- | --- |
| 第一次执行启动 worker | `ExecGather()` | `ExecGatherMerge()` |
| worker reader 数组 | `reader` / `nreaders` | `reader` / `nreaders` |
| leader 是否本地执行 | `need_to_scan_locally` | `need_to_scan_locally` |
| worker tuple 存储 | `funnel_slot` | `gm_slots[]` |
| reader 选择 | round-robin + nonblocking | merge heap 选最小 tuple |
| 顺序保证 | 不保证全局顺序 | 保证按 sort key merge |
| tuple copy | 通常不 copy reader tuple | reader tuple 需要 copy 后缓存 |
| wait event | `WAIT_EVENT_EXECUTE_GATHER` | 类似读取/merge wait 路径 |
| shutdown | `ExecShutdownGatherWorkers()` | `ExecShutdownGatherMergeWorkers()` |
| rescan | `ExecReScanGather()` | `ExecReScanGatherMerge()` + clear tuples |

这个对照能解释为什么 `GatherMerge` 更重：它不只是“Gather 后排序”，而是在线维护多个有序 stream 的当前 tuple。

## 20. 故障模式速查表

| 现象 | 可能原因 | 排查 |
| --- | --- | --- |
| `Gather` 下 worker 行数严重倾斜 | block 分配、leader participation、worker 启动时机 | per-worker rows、parallel scan shared state。 |
| leader CPU 很高 | leader 同时汇聚、投影、本地执行 | 关闭 `parallel_leader_participation` 对比。 |
| worker 卡在 shm_mq send | tuple queue 满 | leader 是否在 `ExecuteGather` 等待或忙于上层输出。 |
| `GatherMerge` 比 `Gather` 慢很多 | copy + compare + merge heap 成本 | sort keys 数量、tuple 宽度、worker 数。 |
| worker launched 但无 tuple | worker 初始化失败或任务分配为空 | finish 阶段错误、per-worker rows。 |
| 有 LIMIT 但 worker 做了很多工作 | tuple bound / queue detach 时机 | `tuples_needed`、ExecParallelFinish 是否及时。 |

## 21. 调参实验矩阵

可以用同一张表，分别测试：

```sql
SET max_parallel_workers_per_gather = 0;
SET max_parallel_workers_per_gather = 2;
SET max_parallel_workers_per_gather = 4;

SET parallel_leader_participation = on;
SET parallel_leader_participation = off;
```

观察：

```text
Workers Planned / Launched
per-worker rows
leader 是否有本地执行贡献
ExecuteGather wait event
总时间和 first-row latency
```

对输出很多 tuple 的查询：

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT * FROM t_parallel_ctx WHERE id > 0;
```

对聚合后输出很少 tuple 的查询：

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM t_parallel_ctx WHERE id > 0;
```

两者对 leader participation 的敏感性可能完全不同。前者 leader 汇聚/输出压力大，后者 worker 本地聚合后输出少，leader 参与可能更划算。

## 22. 本节小结

`Gather` 和 `GatherMerge` 是 parallel query 从多 worker 回到单 leader executor pipeline 的汇聚边界。它们延迟到第一次执行才启动 worker，用 tuple queue 接收 worker MinimalTuple，并根据 worker 数、`single_copy` 和 `parallel_leader_participation` 决定 leader 是否也执行子计划。

普通 `Gather` 追求无序 funnel 和吞吐，`GatherMerge` 用更高的 copy/compare 成本维持全局有序输出。二者都必须在 bounded queue 背压、worker shortage、leader 本地执行和 executor pull 模型之间折中。

可迁移规律：

```text
并行执行的汇聚节点不是简单 merge results；
它同时是 worker 启动时机、输出背压、leader 是否参与、错误处理和上层 pull 协议的交汇点。
```
