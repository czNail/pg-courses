# PostgreSQL parallel worker launch、attach barrier 与少 worker fallback

## 课程定位

前置知识：已经理解 `ParallelContext` 的生命周期和 `InitializeParallelDSM()` 如何创建包含 fixed state、error queue 和 entrypoint 的 DSM。

本节唯一主问题：

```text
LaunchParallelWorkers() 如何用 dynamic background worker、BackgroundWorkerHandle、
lock group leader 和 error queue 启动并行 worker，
WaitForParallelWorkersToAttach() 又如何把注册失败、启动失败和实际 worker 数不足变成 executor 可以继续或报错的状态？
```

核心矛盾：并行计划希望获得若干 worker 来降低执行时间，但 worker slot、fork、初始化、DSM attach 都可能失败；leader 既不能假定 worker 一定存在，也不能在必须等待 worker 的地方无限等待。PostgreSQL 把“少 worker”作为正常 fallback，把“已注册但未成功初始化”作为可诊断错误。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

前两节已经建立：

```text
ParallelContext ownership
  -> InitializeParallelDSM() 写入启动包
```

本节关注启动：

```text
LaunchParallelWorkers()
  -> RegisterDynamicBackgroundWorker()
  -> worker attach error queue
  -> WaitForParallelWorkersToAttach()
  -> leader 知道哪些 worker 可靠可用
```

下一节会进入 worker 自身的 `ParallelWorkerMain()` bootstrap。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
leader 先 BecomeLockGroupLeader()，
再用 RegisterDynamicBackgroundWorker() 为每个 worker slot 注册 ParallelWorkerMain，
把 DSM handle 作为 bgw_main_arg、worker 编号放入 bgw_extra，
注册成功的 worker 通过 error queue handle 与 leader 绑定；
WaitForParallelWorkersToAttach() 通过 background worker 状态和 shm_mq sender 判断 worker 是否真正初始化成功。
```

tension 是：

```text
并行执行要尽早启动 worker
  vs
worker 不足、尚未启动、启动失败、已干净退出都必须被 leader 区分
```

因此 `nworkers_to_launch`、`nworkers_launched` 和 `known_attached_workers` 分别表达不同阶段，不应该混用。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/parallel.c` | `LaunchParallelWorkers()`、`WaitForParallelWorkersToAttach()`、`WaitForParallelWorkersToFinish()` 中的初始化失败判断。 |
| 2 | `src/include/access/parallel.h` | `ParallelWorkerInfo`、worker handle 与 error queue 字段。 |
| 3 | `src/backend/postmaster/bgworker.c` | dynamic background worker 的注册、状态和启动入口。 |
| 4 | `src/backend/storage/ipc/shm_mq.c` | error queue 的 sender / receiver attach 状态。 |
| 5 | `src/backend/executor/nodeGather.c` | executor 如何接受少 worker fallback。 |
| 6 | `src/backend/executor/nodeGatherMerge.c` | GatherMerge 的同类启动路径。 |

## 4. 关键数据结构与状态

### worker 数字段

| 字段 | 含义 |
| --- | --- |
| `nworkers` | `InitializeParallelDSM()` 后仍允许使用的最大 worker 数。可能因 fallback 变成 0。 |
| `nworkers_to_launch` | 本轮准备启动的 worker 数，可由 `ReinitializeParallelWorkers()` 调低。 |
| `nworkers_launched` | `RegisterDynamicBackgroundWorker()` 成功注册的 worker 数。 |

`nworkers_launched` 不是“已经 attach 成功”。worker 可能已注册但还未 fork，或启动后在 attach error queue 前失败。

### `ParallelWorkerInfo`

每个 worker slot 包含：

| 字段 | 语义 |
| --- | --- |
| `bgwhandle` | dynamic background worker handle，用于查询 pid、等待 shutdown、terminate。 |
| `error_mqh` | leader 侧 attach 的 error queue handle；worker attach 后，sender 会变成 worker 的 `MyProc`。 |

## 5. 主流程源码 walkthrough

`LaunchParallelWorkers()` 主流程：

```text
if nworkers == 0 or nworkers_to_launch == 0:
  return

BecomeLockGroupLeader()
assert pcxt->seg != NULL
switch to TopTransactionContext

configure BackgroundWorker:
  bgw_name = "parallel worker for PID ..."
  bgw_type = "parallel worker"
  bgw_flags = SHMEM_ACCESS | BACKEND_DATABASE_CONNECTION | CLASS_PARALLEL
  bgw_start_time = BgWorkerStart_ConsistentState
  bgw_restart_time = BGW_NEVER_RESTART
  bgw_library_name = "postgres"
  bgw_function_name = "ParallelWorkerMain"
  bgw_main_arg = dsm_segment_handle(pcxt->seg)
  bgw_notify_pid = MyProcPid

for i in 0..nworkers_to_launch-1:
  bgw_extra = i
  if RegisterDynamicBackgroundWorker(...):
    shm_mq_set_handle(worker[i].error_mqh, bgwhandle)
    nworkers_launched++
  else:
    mark registration failure
    detach unused error queue

if nworkers_launched > 0:
  known_attached_workers = palloc0(bool[nworkers_launched])
```

`BecomeLockGroupLeader()` 是正确性边界：worker 后续会加入 leader 的 lock group，避免 parallel group 内部的锁关系被当成独立 backend 造成不可检测死锁。

`WaitForParallelWorkersToAttach()` 的判断循环：

```text
CHECK_FOR_INTERRUPTS()
for each launched worker:
  if already known attached:
    continue
  if error_mqh == NULL:
    mark known attached   -- worker 已干净退出
  else if bgworker status == BGWH_STARTED:
    if shm_mq_get_sender(error queue) != NULL:
      mark known attached
  else if status == BGWH_STOPPED:
    if sender still NULL:
      ERROR "parallel worker failed to initialize"
    else:
      mark known attached
  else:
    WaitLatch(... WAIT_EVENT_BGWORKER_STARTUP)

until all launched workers known
```

这个函数不是所有 caller 都必须马上调用。`README.parallel` 明确建议 leader 尽量先做有用工作，避免为了罕见 startup failure 提前 idle。但如果 leader 要等待某个 worker 或所有 worker，就必须先确认 attach，避免无限等一个根本没启动成功的 worker。

## 6. 生命周期 / ownership / cleanup

### 注册失败

注册失败通常表示 worker slot 不足。`LaunchParallelWorkers()` 不报错，而是 detach 后续 unused error queue，并减少实际 launched worker。executor 的 `Gather` 会把 `Workers Planned` 和 `Workers Launched` 区分显示。

### 启动失败

注册成功但 worker 没有 attach error queue 就退出，`WaitForParallelWorkersToAttach()` 或后续 `WaitForParallelWorkersToFinish()` 会报 `parallel worker failed to initialize`。

### 干净退出

如果 worker attach error queue 后很快完成并发送 terminate，leader 可以把它视为已知状态。它可能没有产生 tuple，但不会被当成启动失败。

## 7. 正确性机制层次

| 机制 | 作用 |
| --- | --- |
| dynamic background worker | 由 postmaster fork，拥有 backend database connection 和 shmem access。 |
| DSM handle in `bgw_main_arg` | worker 找到并行启动包。 |
| worker number in `bgw_extra` | worker 找到自己的 error queue / tuple queue slot。 |
| error queue sender | leader 判断 worker 是否越过“错误可回传”的初始化点。 |
| `known_attached_workers` | 避免重复判断，并供 finish 阶段区分已知可通知 worker。 |
| latch wait event | leader 等待 worker 状态变化时可观测。 |

## 8. 错误路径 / 异常路径 / fallback

三类状态要区分：

```text
注册失败:
  没有 worker 进程，不报错，少 worker fallback。

注册成功但未 attach error queue 即停止:
  worker 初始化失败，ERROR。

attach 后停止:
  worker 已经能回传消息；是否成功取决于它是否发过 ErrorResponse / Terminate 等协议消息。
```

如果 caller 从未调用 `WaitForParallelWorkersToAttach()`，`WaitForParallelWorkersToFinish()` 仍会检查未 attach 失败，避免错误被吞掉。

## 9. 成本、资源与跨模块传播

启动 worker 会消耗：

```text
max_worker_processes / parallel worker slot
postmaster fork / backend bootstrap 时间
DSM attach 成本
error queue 和 latch/procsignal 通信
lock group 成员关系
```

因此 executor 选择在 `Gather` / `GatherMerge` 第一次执行时才 launch，而不是 `ExecInitNode()` 阶段就启动，避免从未执行到的节点提前消耗资源。

## 10. 观测与诊断入口

| 入口 | 说明 |
| --- | --- |
| `EXPLAIN ANALYZE` | `Workers Planned` vs `Workers Launched`。 |
| `pg_stat_activity.wait_event` | `BgWorkerStartup` 表示 leader 正在等 worker 启动。 |
| server log | worker 初始化失败详情可能只在 log 中。 |
| gdb | `pcxt->nworkers_to_launch`、`nworkers_launched`、`known_attached_workers[]`、`shm_mq_get_sender()`。 |

## 11. 常见误区

1. 误以为 worker 注册成功等于 worker 可用。必须等 attach error queue 才能确认初始化越过诊断边界。
2. 误以为 worker 不足一定是错误。并行查询通常要能继续非并行或少并行执行。
3. 误以为 `WaitForParallelWorkersToAttach()` 总该马上调用。leader 能做本地工作时，过早等待会浪费时间。
4. 误以为 `nworkers_launched` 就是 executor 实际并行度。leader participation 还会影响实际执行者数量。

## 12. 课堂实验

1. 设置 `max_parallel_workers` 很小，执行并行查询，观察 planned / launched 差异。
2. 在 `LaunchParallelWorkers()` 的注册失败分支设断点，确认 unused error queue 会被 detach。
3. 在 `WaitForParallelWorkersToAttach()` 中观察 `shm_mq_get_sender()` 从 NULL 变为 worker `PGPROC`。

## 13. 讨论题

1. 为什么 worker 初始化失败不能简单当成“少 worker fallback”？
2. leader 为什么需要成为 lock group leader 后再启动 worker？
3. 如果 caller 要等待 worker 产出 tuple，却不确认 worker 是否 attach，会出现什么风险？

## 14. 源码深挖：worker 启动状态机

`LaunchParallelWorkers()` 和 `WaitForParallelWorkersToAttach()` 没有显式 enum，但可以还原出 worker slot 的状态机：

```text
SLOT_BUDGETED
  -> REGISTER_ATTEMPTED
     -> REGISTER_FAILED
     -> REGISTERED
        -> NOT_YET_STARTED
        -> STARTED_NOT_ATTACHED
        -> ATTACHED
        -> STOPPED_BEFORE_ATTACH
        -> STOPPED_AFTER_ATTACH
        -> FINISHED_AND_QUEUE_DETACHED
```

这些状态不是装饰性分类。每个状态对应不同处理：

| 状态 | 代码判定 | leader 行为 |
| --- | --- | --- |
| budgeted | `i < nworkers_to_launch` | 尝试注册 dynamic background worker。 |
| register failed | `RegisterDynamicBackgroundWorker()` false | detach error queue，不增加 `nworkers_launched`，少 worker fallback。 |
| registered | `bgwhandle != NULL` | 设置 error queue handle 对应 bgworker handle。 |
| not yet started | `GetBackgroundWorkerPid() == BGWH_NOT_YET_STARTED` | latch wait `WAIT_EVENT_BGWORKER_STARTUP`。 |
| started not attached | `BGWH_STARTED` 且 error queue sender NULL | 继续轮询，不认为 worker 可用。 |
| attached | error queue sender 非 NULL | 标记 `known_attached_workers[i] = true`。 |
| stopped before attach | `BGWH_STOPPED` 且 sender NULL | ERROR，worker 初始化失败。 |
| stopped after attach | `BGWH_STOPPED` 且 sender 非 NULL | 可以认为 worker 已越过诊断边界。 |
| finished queue detached | `error_mqh == NULL` | worker 已发送 terminate 或 leader 已 cleanup。 |

### 14.1 为什么 registration failure 不报错

注册失败通常发生在：

```text
max_worker_processes 已满
max_parallel_workers 已满
postmaster 暂时无法接受新的 dynamic worker
```

此时 worker 根本没有进程实体，也没有开始执行用户计划。对于并行查询，正确 fallback 是：

```text
Workers Planned: 4
Workers Launched: 0..3
leader 本地仍可执行计划
```

如果这里 ERROR，会把一个性能优化失败变成用户可见语义失败。PostgreSQL 选择少 worker 继续。

### 14.2 为什么一旦某次注册失败就跳过后续注册

代码中有：

```text
bool any_registrations_failed = false;

if (!any_registrations_failed &&
    RegisterDynamicBackgroundWorker(...))
  success
else
  any_registrations_failed = true
```

这不是偷懒。注册失败往往意味着全局 worker slot 或 postmaster capacity 已经不足，后续 slot 大概率也失败。继续调用注册只会增加无效工作。更重要的是，后续 slot 已经在 DSM 中预算了 error queue：

```text
必须 detach unused error queue
否则 leader 后续可能以为这些 worker 需要启动或结束
```

因此失败分支仍然要遍历所有剩余 slot。

### 14.3 为什么 `nworkers_launched` 只统计成功注册

`nworkers_launched` 在成功注册后递增：

```text
pcxt->nworkers_launched++;
```

它不等于：

```text
实际开始运行的 worker 数
实际 attach error queue 的 worker 数
实际产出 tuple 的 worker 数
```

它只表示：

```text
leader 已经拿到 BackgroundWorkerHandle，并且 postmaster 接受了启动请求。
```

因此 `EXPLAIN` 中的 launched workers 是一个比 planned 更接近实际的数字，但仍不能说明每个 worker 都产出了数据。worker 可能启动后发现无任务、快速退出，或在 early startup 后 ERROR。

### 14.4 为什么 `known_attached_workers` 在 launch 后才分配

`known_attached_workers` 的长度是 `nworkers_launched`，而不是 `nworkers`。因为注册失败的 slot 不再参与 attach / finish 协议：

```text
nworkers:
  DSM 预算容量

nworkers_to_launch:
  本轮尝试启动数量

nworkers_launched:
  需要后续状态追踪的 worker handle 数量
```

这避免 leader 后续等待一个从未注册成功的 slot。

## 15. dynamic background worker 配置逐字段解释

`LaunchParallelWorkers()` 构造 `BackgroundWorker worker`。每个字段都在服务并行 worker 的 bootstrap。

### 15.1 `bgw_name` 和 `bgw_type`

```text
bgw_name = "parallel worker for PID %d"
bgw_type = "parallel worker"
```

用户在 `pg_stat_activity`、日志或进程列表中看到的 backend type/name 来自这些字段。它们不是 correctness 字段，但对诊断很重要。

### 15.2 `bgw_flags`

```text
BGWORKER_SHMEM_ACCESS
BGWORKER_BACKEND_DATABASE_CONNECTION
BGWORKER_CLASS_PARALLEL
```

含义：

| flag | 为什么需要 |
| --- | --- |
| `SHMEM_ACCESS` | worker 要 attach DSM、访问 PGPROC、lock、ProcArray 等 shared memory。 |
| `BACKEND_DATABASE_CONNECTION` | worker 要连接 leader 的 database，做 catalog lookup 和 executor 初始化。 |
| `CLASS_PARALLEL` | postmaster / worker accounting 把它视为 parallel worker。 |

没有 database connection，`ParallelWorkerMain()` 就不能恢复 catalog-dependent 状态，也不能执行 parallel query plan。

### 15.3 `bgw_start_time`

```text
BgWorkerStart_ConsistentState
```

parallel worker 只在数据库达到 consistent state 后启动。这对 standby / recovery 场景有意义：worker 不能在数据库还不能安全执行查询时进入 executor。

### 15.4 `bgw_restart_time`

```text
BGW_NEVER_RESTART
```

parallel worker 是单次并行操作的执行分支。它失败后不应该被 postmaster 自动重启，因为：

```text
DSM 中的任务状态可能已经变化
leader 可能已经进入 cleanup 或 ERROR
tuple queue / error queue 的协议状态不可重放
```

自动重启会把一次执行变成重复执行风险。

### 15.5 `bgw_main_arg`

```text
worker.bgw_main_arg = UInt32GetDatum(dsm_segment_handle(pcxt->seg))
```

worker 只通过 DSM handle 进入共享启动包。它不接收 `ParallelContext *`，因为那个指针只在 leader 进程地址空间有效。

### 15.6 `bgw_extra`

```text
memcpy(worker.bgw_extra, &i, sizeof(int))
```

worker number 用于：

```text
error queue offset:
  error_queue_space + ParallelWorkerNumber * PARALLEL_ERROR_QUEUE_SIZE

tuple queue offset:
  tuple_queue_space + ParallelWorkerNumber * PARALLEL_TUPLE_QUEUE_SIZE

instrumentation slot:
  instrumentation[ParallelWorkerNumber]
```

这个数字是 worker 在本次 `ParallelContext` 内的局部编号，不是 backend id，也不是 `MyProcNumber`。

### 15.7 `bgw_notify_pid`

```text
worker.bgw_notify_pid = MyProcPid
```

postmaster 用它通知 leader worker 状态变化。`WaitForParallelWorkersToAttach()` 和 finish 路径中的 latch wakeup 都依赖这类通知。

## 16. attach barrier 的细节

`WaitForParallelWorkersToAttach()` 的任务不是启动 worker，而是确认：

```text
每个成功注册的 worker，要么已经 attach error queue，
要么已经在 attach 之后干净退出，
否则如果 attach 前停止，就报错。
```

### 16.1 为什么检查 error queue sender

leader 侧在 `InitializeParallelDSM()` 中：

```text
shm_mq_set_receiver(mq, MyProc)
```

worker 侧在 `ParallelWorkerMain()` 中：

```text
shm_mq_set_sender(mq, MyProc)
```

因此 `shm_mq_get_sender(mq) != NULL` 表示 worker 至少执行到了 error queue attach 边界。这个边界比 `BGWH_STARTED` 更强：

```text
BGWH_STARTED:
  OS/backend worker 已经开始

error queue sender non-NULL:
  worker 已经能把后续 ErrorResponse / NoticeResponse 发回 leader
```

### 16.2 为什么 `CHECK_FOR_INTERRUPTS()` 在循环顶部

worker attach 或报错会通过 procsignal/latch 通知 leader。leader 在 attach wait 中必须处理 pending parallel messages：

```text
CHECK_FOR_INTERRUPTS()
  -> ProcessInterrupts()
  -> if ParallelMessagePending
       ProcessParallelMessages()
```

否则 worker 已经发了 ErrorResponse，leader 仍可能只看 bgworker status，错过更具体的错误。

### 16.3 `error_mqh == NULL` 为什么算 attached

`error_mqh == NULL` 的含义在 attach wait 中是：

```text
leader 已经通过消息处理知道这个 worker 干净退出，
或队列已经被 detach。
```

如果 worker 已经发送 terminate，`ProcessParallelMessage()` 会：

```text
shm_mq_detach(pcxt->worker[i].error_mqh)
pcxt->worker[i].error_mqh = NULL
```

这个 worker 不需要再等待 attach。它可能没有留下可读 tuple，但 finish 阶段不会把它误判成 startup failure。

### 16.4 `BGWH_STOPPED` 的两个分支

如果 `GetBackgroundWorkerPid()` 返回 `BGWH_STOPPED`：

```text
sender == NULL:
  worker 死在 attach error queue 前
  leader 没有可靠错误详情
  ERROR "parallel worker failed to initialize"

sender != NULL:
  worker 至少 attach 过
  后续消息处理或 terminate 协议可以解释结果
```

这就是“初始化失败”和“worker 已结束”的分界线。

### 16.5 为什么 wait event 是 `WAIT_EVENT_BGWORKER_STARTUP`

当 worker 还没 started，leader：

```text
WaitLatch(MyLatch, WL_LATCH_SET | WL_EXIT_ON_PM_DEATH, -1, WAIT_EVENT_BGWORKER_STARTUP)
```

这让 `pg_stat_activity` 能看到 leader 正在等 worker 启动，而不是 CPU 忙等或普通 lock wait。诊断时如果大量 backend 卡在这个 wait event，应首先看 worker slot、postmaster 状态和系统 fork 压力。

## 17. 少 worker fallback 在 executor 中如何被消费

`Gather` 的第一次执行：

```text
if gather->num_workers > 0 && estate->es_use_parallel_mode:
  node->pei = ExecInitParallelPlan(...)
  LaunchParallelWorkers(pcxt)
  node->nworkers_launched = pcxt->nworkers_launched
  estate->es_parallel_workers_to_launch += pcxt->nworkers_to_launch
  estate->es_parallel_workers_launched += pcxt->nworkers_launched
```

然后：

```text
if pcxt->nworkers_launched > 0:
  ExecParallelCreateReaders()
  node->nreaders = pcxt->nworkers_launched
else:
  node->nreaders = 0
  node->reader = NULL
```

最后决定 leader 是否本地执行：

```text
node->need_to_scan_locally =
  (node->nreaders == 0)
  || (!gather->single_copy && parallel_leader_participation)
```

这条公式很关键：

```text
没有 worker:
  leader 必须本地扫描，否则没有人产生 tuple

有 worker 且不是 single_copy:
  如果 parallel_leader_participation = on，leader 也参与

single_copy:
  通常只希望一个执行者跑子计划，leader participation 受限制
```

### 17.1 为什么 executor 不直接报错

planner 选择 parallel plan 时，是基于成本估算和可用 worker 参数；执行时系统资源可能变化。executor 必须把 worker shortage 当成 runtime condition，而不是 planner bug。

### 17.2 EXPLAIN 的两个数字

```text
Workers Planned:
  plan node 中的 num_workers

Workers Launched:
  runtime pcxt->nworkers_launched
```

常见解释：

| 现象 | 可能原因 |
| --- | --- |
| Planned > 0, Launched = 0 | worker slot 不足、DSM fallback、parallel mode 被禁用、leader 本地执行。 |
| Launched < Planned | 部分注册失败。 |
| Launched = Planned 但速度不提升 | tuple queue 背压、worker skew、leader bottleneck、I/O 或锁等待。 |

## 18. finish 阶段为什么还检查 startup failure

如果 caller 没调用 `WaitForParallelWorkersToAttach()`，`WaitForParallelWorkersToFinish()` 仍要兜底：

```text
not all workers finished
  -> inspect bgworker status
  -> if stopped and error queue sender == NULL
       ERROR "parallel worker failed to initialize"
```

这防止一种危险情况：

```text
leader 不等待 attach
leader 开始读 tuple queue
某个 worker 根本没启动成功
tuple queue 永远没有数据
leader 如果不检查 bgworker status，可能等待错误对象
```

`WaitForParallelWorkersToFinish()` 还负责最后处理 worker 消息：

```text
CHECK_FOR_INTERRUPTS()
  -> ProcessParallelMessages()
```

即使 leader 认为执行已经完成，也必须等待 worker finish，以免漏掉 worker shutdown 时发出的 ERROR。

## 19. 实验与故障注入

### 19.1 worker shortage

```sql
SET max_parallel_workers_per_gather = 4;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM t_parallel_ctx;
```

同时在另一个 session 占用 parallel workers，观察：

```text
Workers Planned: 4
Workers Launched: 1
```

如果 launched 少于 planned，但查询成功，这是正常 fallback。

### 19.2 attach barrier 断点

gdb：

```gdb
break LaunchParallelWorkers
break WaitForParallelWorkersToAttach
break ParallelWorkerMain
break pq_redirect_to_shm_mq
```

观察：

```gdb
p pcxt->nworkers
p pcxt->nworkers_to_launch
p pcxt->nworkers_launched
p pcxt->known_attached_workers[0]
p shm_mq_get_sender(shm_mq_get_queue(pcxt->worker[0].error_mqh))
```

在 worker 到达 `pq_redirect_to_shm_mq()` 前，sender 可能仍为 NULL。

### 19.3 startup failure 实验

源码实验可以在 `ParallelWorkerMain()` attach error queue 之前临时插入：

```c
elog(ERROR, "fail before error queue");
```

预期：

```text
leader 看到:
  parallel worker failed to initialize

server log 中可能有更具体信息
```

如果把 ERROR 放到 `pq_redirect_to_shm_mq()` 之后，leader 应该能收到 worker 的 ErrorResponse，并显示 parallel worker context。

### 19.4 wait event 观测

当 leader 等 worker 启动：

```sql
SELECT pid, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE wait_event IN ('BgWorkerStartup', 'ParallelFinish', 'ExecuteGather');
```

解释：

```text
BgWorkerStartup:
  等 worker 进入 attached / stopped 状态

ParallelFinish:
  等 worker 完成并处理最后消息

ExecuteGather:
  Gather 没有可读 tuple，等待 tuple queue / worker 进展
```

## 20. 版本与边界说明

本课基于 `bd4bd30ce6a7`。未来版本可能调整：

```text
worker class / bgworker flag
错误消息类型
progress reporting protocol
worker accounting
```

但下列边界稳定：

```text
requested workers != registered workers
registered workers != attached workers
attached workers != produced tuples
registration failure is fallback
pre-attach worker death is initialization failure
finish must still drain worker errors
```

调试并行启动问题时，应按状态层次定位，不要只看 `Workers Launched` 一个数字。

## 21. 源码检查清单：等待 worker 前要确认什么

任何新调用方只要需要等待 worker，就应该先回答：

```text
我等的是 worker 启动，还是 worker 完成？
我等的是某个 worker，还是所有 worker？
worker 没有注册成功时是否允许 fallback？
worker 注册成功但 attach 前死亡时是否必须 ERROR？
等待期间是否会 CHECK_FOR_INTERRUPTS()？
```

### 21.1 等 attach

适用场景：

```text
leader 准备等待某个 worker 先产出状态
leader 准备访问 worker 初始化后才会设置的共享字段
leader 没有其它本地工作可以安全推进
```

调用：

```text
WaitForParallelWorkersToAttach(pcxt)
```

返回后能保证：

```text
成功注册的 worker 都已经越过 error queue attach 边界，
或已经在越过该边界后干净退出。
```

不能保证：

```text
worker 仍然活着
worker 一定会产出 tuple
worker 的业务任务已经完成
```

### 21.2 等 finish

适用场景：

```text
并行操作结束
leader 不再需要 worker tuple / result
必须接收 worker 最后的 ERROR / NOTICE / terminate
必须汇总 worker usage
```

调用：

```text
WaitForParallelWorkersToFinish(pcxt)
```

返回后能保证：

```text
worker 最后的控制消息已经被 leader 处理，
pre-attach startup failure 不会被吞掉，
FixedParallelState.last_xlog_end 已可用于更新 leader XactLastRecEnd。
```

不能替代：

```text
DestroyParallelContext()
```

因为 finish 不释放 DSM、worker handles 和 `ParallelContext`。

### 21.3 等 exit

`WaitForParallelWorkersToExit()` 是 `parallel.c` 内部 helper，`DestroyParallelContext()` 使用它。它比 finish 更底层：

```text
finish:
  收到 worker 最后消息

exit:
  worker 进程实际 shutdown
```

事务边界最终需要 exit，因为 leader 不能在 worker 进程还活着时完成提交/回滚 cleanup。

## 22. 故障模式速查表

| 现象 | 阶段 | 解释 |
| --- | --- | --- |
| `Workers Launched: 0` 但查询成功 | registration/fallback | 没有 worker，leader 本地执行。 |
| `parallel worker failed to initialize` | pre-attach failure | worker 注册了，但没 attach error queue。 |
| leader 等 `BgWorkerStartup` 很久 | startup wait | postmaster 还没启动 worker，或 worker slot / fork 压力。 |
| leader 等 `ParallelFinish` 很久 | finish wait | worker 仍在运行、阻塞、或最后消息未处理。 |
| worker 没产出 tuple 但最后 ERROR | finish 阶段 | tuple path 先当作无 tuple，finish 才发现 startup / runtime error。 |
| launched 少于 planned | registration | 资源不足或部分注册失败。 |

诊断顺序：

```text
先看 planned/launched
  -> 再看 wait_event
  -> 再看 server log 中 worker startup error
  -> 再用 gdb 看 error queue sender
```

## 23. 本节小结

`LaunchParallelWorkers()` 把“请求 N 个 worker”变成“注册了 M 个 worker”，`WaitForParallelWorkersToAttach()` 再把“注册成功”推进到“worker 已经越过错误可回传边界”。少 worker 是正常 fallback，未 attach 的早期失败是必须报告的错误。

可迁移规律：

```text
异步启动的 worker 要分清 requested、registered、attached、finished；
每个状态都需要不同的 fallback 和诊断语义。
```
