# PostgreSQL Parallel mode 与 ParallelContext ownership 边界

## 课程定位

前置知识：已经理解 DSM / shm_toc / shm_mq、ResourceOwner、MemoryContext、PGPROC / ProcArray 和后台 worker 的基本模型。

本节唯一主问题：

```text
为什么并行操作必须包在 parallel mode 中，
CreateParallelContext()、pcxt_list、subtransaction id、AtEOSubXact_Parallel() / AtEOXact_Parallel()
和 DestroyParallelContext() 如何把 worker、DSM、error queue 等资源绑定到可清理的事务边界？
```

核心矛盾：并行 worker 需要复用 leader 的事务、snapshot、锁和 backend-local 状态；但 leader 不能在 worker 仍可能使用这些状态时任意修改事务语义、释放资源或提交/回滚。PostgreSQL 用 parallel mode 禁止危险状态变化，再用 `ParallelContext` 把 worker、DSM 和消息队列挂到事务/子事务 cleanup 链上。

学完后应能判断：哪些操作必须在 `EnterParallelMode()` / `ExitParallelMode()` 之间运行，为什么并行上下文不能泄漏到事务结束以后，以及 `DestroyParallelContext()` 为什么既是正常收尾，也是 ERROR-safe 兜底。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

第 27-34 节已经讲过 DSM、shm_toc、shm_mq 和 DSA 的底层能力。本节开始进入更高一层的并行基础设施：

```text
DSM / shm_toc / shm_mq
  -> ParallelContext
  -> parallel worker bootstrap
  -> executor parallel plan / tuple queue
```

本节只回答 ownership 边界：一个并行操作如何被创建、登记、销毁和事务级兜底清理。下一节再讲 `InitializeParallelDSM()` 往 DSM 中序列化哪些 leader 状态。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
调用方先 EnterParallelMode() 禁止不安全的事务状态变化，
再 CreateParallelContext() 把一个 ParallelContext 放入 TopTransactionContext 和 pcxt_list，
后续 InitializeParallelDSM() / LaunchParallelWorkers() 使用它管理 DSM、worker 和 error queue，
正常结束时 DestroyParallelContext() 移除链表并等待 worker 退出，
异常路径由 AtEOSubXact_Parallel() / AtEOXact_Parallel() 扫 pcxt_list 兜底销毁。
```

这里的 tension 是：

```text
并行执行需要让多个 backend 共享同一个事务语义
  vs
每个 backend 又有独立 resource owner、内存上下文、错误路径和进程退出时序
```

PostgreSQL 没有把并行 worker 伪装成线程。它保留多进程隔离，把共享状态显式放进 DSM，把生命周期显式挂到 `ParallelContext`。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/README.parallel` | 并行基础设施的设计约定、error reporting 和 coding pattern。 |
| 2 | `src/include/access/parallel.h` | `ParallelContext`、`ParallelWorkerInfo`、`pcxt` 对外 API。 |
| 3 | `src/backend/access/transam/xact.c` | `EnterParallelMode()`、`ExitParallelMode()`、`IsInParallelMode()` 以及并行模式禁止的事务行为。 |
| 4 | `src/backend/access/transam/parallel.c` | `CreateParallelContext()`、`DestroyParallelContext()`、事务/子事务 cleanup。 |
| 5 | `src/backend/executor/execMain.c` | executor 如何根据计划进入/退出 parallel mode。 |
| 6 | `src/backend/executor/execParallel.c` | executor 侧如何创建 `ParallelContext` 并把它封装进 `ParallelExecutorInfo`。 |

推荐先读 `README.parallel` 的 coding pattern，再进入 `parallel.c`。不要先从 `nodeGather.c` 开始，否则容易把“某个执行节点如何使用并行”误认为基础设施本身。

## 4. 关键数据结构与状态

### `ParallelContext`

`ParallelContext` 定义在 `src/include/access/parallel.h`。本节关注这些字段：

| 字段 | 语义 |
| --- | --- |
| `node` | 挂入 backend-local 的 `pcxt_list`，供事务结束兜底扫描。 |
| `subid` | 创建该 context 的子事务 id，`AtEOSubXact_Parallel()` 用它只清理当前子事务创建的 context。 |
| `nworkers` / `nworkers_to_launch` / `nworkers_launched` | 预算 worker 数、实际准备启动数、成功注册的 worker 数。 |
| `library_name` / `function_name` | worker 入口的名字，而不是函数指针；`EXEC_BACKEND` 下跨进程函数地址不能直接共享。 |
| `error_context_stack` | 创建 context 时的 error context，worker 错误回传到 leader 时用来恢复诊断上下文。 |
| `estimator` / `seg` / `private_memory` / `toc` | DSM sizing、实际 DSM segment、无 worker fallback 的私有内存、TOC。 |
| `worker[]` | 每个 worker 的 `BackgroundWorkerHandle` 和 error queue handle。 |
| `known_attached_workers` | leader 是否确认 worker 已 attach error queue。 |

这个结构是 backend-local ownership 对象，不是所有 worker 共享的全局对象。worker 通过 DSM 中的 `FixedParallelState` 和 TOC 恢复状态，而不是读取 leader 的 `ParallelContext`。

### `pcxt_list`

`pcxt_list` 是 leader backend 内的活动并行上下文链表。它的意义不是查询执行调度，而是 cleanup：

```text
正常路径:
  DestroyParallelContext(pcxt)
    -> dlist_delete(&pcxt->node)

异常路径:
  AtEOSubXact_Parallel()
  AtEOXact_Parallel()
    -> while pcxt_list not empty
       -> DestroyParallelContext(pcxt)
```

也就是说，只要 context 留在链表上，事务结束就会认为它仍需要强制清理。

## 5. 主流程源码 walkthrough

典型 executor parallel query 路径：

```text
ExecutorStart()
  -> estate->es_use_parallel_mode = use_parallel_mode
  -> EnterParallelMode()

ExecGather() / ExecGatherMerge() 第一次执行
  -> ExecInitParallelPlan()
     -> CreateParallelContext("postgres", "ParallelQueryMain", nworkers)
     -> InitializeParallelDSM()
     -> executor 写入自己的 DSM 状态
  -> LaunchParallelWorkers()

执行结束 / shutdown
  -> ExecParallelFinish()
     -> WaitForParallelWorkersToFinish()
  -> ExecParallelCleanup()
     -> DestroyParallelContext()

ExecutorEnd()
  -> ExitParallelMode()
```

`CreateParallelContext()` 的关键动作很少，但边界很重要：

```text
Assert(IsInParallelMode())
MemoryContextSwitchTo(TopTransactionContext)
pcxt = palloc0_object(ParallelContext)
pcxt->subid = GetCurrentSubTransactionId()
pcxt->nworkers = nworkers
pcxt->library_name = pstrdup(library_name)
pcxt->function_name = pstrdup(function_name)
pcxt->error_context_stack = error_context_stack
shm_toc_initialize_estimator(&pcxt->estimator)
dlist_push_head(&pcxt_list, &pcxt->node)
```

这里有三个设计点：

1. 必须在 parallel mode 中创建，说明调用方已经承诺不做危险状态变化。
2. 放到 `TopTransactionContext`，避免短生命周期 context reset 后丢失 cleanup handle。
3. 立即挂入 `pcxt_list`，使后续任何 ERROR 都能由事务 cleanup 找到它。

`DestroyParallelContext()` 的顺序同样有语义：

```text
dlist_delete(&pcxt->node)
for each launched worker:
  TerminateBackgroundWorker()
  shm_mq_detach(error_mqh)
dsm_detach(pcxt->seg)
pfree(private_memory) if fallback
HOLD_INTERRUPTS()
  WaitForParallelWorkersToExit()
RESUME_INTERRUPTS()
free worker array and strings
pfree(pcxt)
```

先从链表删除，是为了避免销毁过程中再 ERROR 时被 end-of-xact cleanup 重复销毁。真正事务提交/回滚不能完成到 worker 还活着，所以等待 worker exit 时会 `HOLD_INTERRUPTS()`。

## 6. 生命周期 / ownership / cleanup

### 谁创建

调用方在 parallel mode 中调用 `CreateParallelContext()`。executor 路径中，`ExecInitParallelPlan()` 创建；并行 VACUUM、并行索引构建也会直接使用同一套 API。

### 谁持有

leader 持有 `ParallelContext`。DSM mapping 和 error queue handle 也由 leader 通过该结构追踪。worker 不持有这个 C 指针；worker 只拿到 DSM handle，然后 attach TOC。

### 谁释放

正常路径应先调用 `WaitForParallelWorkersToFinish()` 接收 worker 最后的消息，再调用 `DestroyParallelContext()`。如果正常路径漏掉，`AtEOSubXact_Parallel()` / `AtEOXact_Parallel()` 会发 warning 并销毁。

### ERROR 时怎么办

ERROR 跳出执行器后，事务 abort cleanup 会扫 `pcxt_list`。如果 context 仍在链表上，`DestroyParallelContext()` 会终止 worker、detach queue / DSM，并等待 worker 退出。这个兜底是 ERROR-safe 的核心。

## 7. 正确性机制层次

| 层次 | 机制 | 解决的问题 |
| --- | --- | --- |
| 事务语义 | `EnterParallelMode()` / `ExitParallelMode()` | 禁止 leader 在并行期间做不安全状态变化。 |
| 生命周期 | `pcxt_list` + `subid` | 子事务/事务结束能找到未销毁 context。 |
| 进程清理 | `TerminateBackgroundWorker()` + `WaitForParallelWorkersToExit()` | leader 不能在 worker 未退出时完成事务边界。 |
| 资源传播 | DSM / shm_toc / shm_mq | 跨进程显式传递状态与错误。 |
| 诊断上下文 | `error_context_stack` | worker 错误回到 leader 时保留创建 context 时的上下文。 |

parallel mode 的限制散落在多个模块中，例如 `varsup.c` 阻止并行模式下分配新 XID，`utility.c` 阻止部分 utility command，`snapmgr.c` 防止 command id / snapshot 语义在并行中被破坏。这些限制都在服务同一个不变量：

```text
worker 正在使用的 leader 事务视图，不能被 leader 在并行期间改成另一套语义。
```

## 8. 错误路径 / 异常路径 / fallback

常见异常路径：

1. 创建 context 后、创建 DSM 前 ERROR：`pcxt_list` 已登记，事务 cleanup 会清理。
2. worker 启动后 leader ERROR：`DestroyParallelContext()` 终止 worker 并等待退出。
3. context 正常执行但忘记销毁：`AtEOXact_Parallel(true)` 会 warning `leaked parallel context`。
4. DSM 创建失败或不需要 worker：后续课程会看到 `InitializeParallelDSM()` 可能退化到 backend-private memory，并把 `nworkers` 降为 0。

不要把 `DestroyParallelContext()` 理解成普通内存 free。它包含 worker 进程管理、DSM detach、message queue detach 和不可中断等待。

## 9. 成本、资源与跨模块传播

每个 `ParallelContext` 至少带来：

```text
TopTransactionContext 中的 backend-local metadata
一个 DSM segment 或 backend-private TOC memory
nworkers 个 background worker handle
nworkers 个 error queue
后续 executor 可能追加 tuple queue、DSA、instrumentation
```

它跨越的模块包括：

```text
xact.c: parallel mode 和 end-of-xact cleanup
parallel.c: context、worker、DSM、message
bgworker.c: dynamic background worker
shm_mq.c / pqmq.c: protocol message forwarding
execParallel.c: executor DSM 内容
```

因此并行不是“多开几个线程”。它会消耗 worker slot、DSM、latch wakeup、共享队列和额外 cleanup 路径。

## 10. 观测与诊断入口

可观察入口：

| 入口 | 能看到什么 |
| --- | --- |
| `EXPLAIN (ANALYZE)` | planned workers、launched workers、per-worker instrumentation。 |
| `pg_stat_activity.wait_event` | `BgWorkerStartup`、`ParallelFinish`、`ExecuteGather` 等等待点。 |
| server log | worker 初始化失败、parallel worker 错误上下文、leaked parallel context warning。 |
| gdb | leader 的 `pcxt_list`、`ParallelContext.nworkers_launched`、`worker[i].error_mqh`。 |

调试建议：在 `CreateParallelContext()`、`DestroyParallelContext()`、`AtEOXact_Parallel()` 设断点，可以看到正常路径和异常路径是否都经过同一个 cleanup 边界。

## 11. 常见误区

1. 误以为 parallel mode 表示“已经启动 worker”。实际上它首先是事务安全限制，worker 可能还没有启动。
2. 误以为 `ParallelContext` 是共享内存对象。它是 leader-local ownership handle。
3. 误以为 worker 数不足应该立即 ERROR。很多调用方必须能接受少 worker 或 0 worker fallback。
4. 误以为 `DestroyParallelContext()` 可以随便在中途跳过。跳过会把 worker、DSM 和 error queue 留给事务 cleanup 兜底，commit 时还会 warning。

## 12. 课堂实验

1. 在 `~/postgres-lab` 中对 `CreateParallelContext()`、`DestroyParallelContext()` 和 `AtEOXact_Parallel()` 设置 gdb 断点，执行一个并行查询，观察 `pcxt_list` 的增删。
2. 执行并行查询时对 `max_parallel_workers` 设置较小值，观察 `Workers Planned` 和 `Workers Launched` 的差异。
3. 在调试构建中人为让执行器路径在创建 context 后 ERROR，确认事务 abort 会进入 `AtEOXact_Parallel(false)`。

## 13. 讨论题

1. 为什么 `ParallelContext` 要分配在 `TopTransactionContext`，而不是当前 memory context？
2. 如果 `DestroyParallelContext()` 先 detach DSM 再终止 worker，会出现什么诊断或资源风险？
3. parallel mode 禁止 XID 分配、DDL 和某些 GUC 修改，本质上是在保护 worker 的哪类状态？

## 14. 源码深挖：parallel mode 到底限制了什么

`EnterParallelMode()` 本身看起来非常小：

```text
EnterParallelMode()
  -> CurrentTransactionState->parallelModeLevel++

ExitParallelMode()
  -> assert parallelModeLevel > 0
  -> assert 外层退出时没有 active ParallelContext
  -> parallelModeLevel--

IsInParallelMode()
  -> parallelModeLevel != 0 || parallelChildXact
```

如果只看这几行，很容易误解 parallel mode 只是一个计数器。真正的含义在其它模块的检查点中展开：

```text
进入 parallel mode
  -> leader 承诺不做会让 worker 语义失配的状态变化
  -> 其它模块通过 IsInParallelMode() 拒绝危险操作
  -> worker 也通过 IsParallelWorker() 进入类似限制
```

本节要把这些分散检查重新压回同一个模型：

```text
并行期间被复制给 worker 的状态，不能在 worker 运行期间被 leader 改成另一套语义。
```

### 14.1 事务 ID 分配边界

在 `src/backend/access/transam/varsup.c` 中，XID 分配路径会检查 parallel mode。原因不是“worker 不能写 WAL”这么简单，而是 XID 分配会改变事务对外身份：

```text
leader 尚未分配 XID:
  worker 已经拿到 serialized transaction state
  worker 的 snapshot / combo CID / command id 边界基于这套状态恢复

leader 并行期间突然分配 XID:
  tuple header、ProcArray publication、subxact state、combo CID 语义都可能变化
  worker 不会自动获得这次变化
```

因此 parallel query 本质上要求：

```text
如果执行需要会改变事务身份的写操作，
就不应该让它发生在已启动或可能启动 parallel worker 的窗口里。
```

这也是为什么本节不把 parallel mode 当成“性能开关”。它是一个 correctness fence。

### 14.2 CommandId / snapshot 边界

在 `snapmgr.c` 中，parallel mode 下某些 command id 变化会被拒绝。原因是 worker 通过 `InitializeParallelDSM()` 得到 active snapshot 和 transaction snapshot，后续 `ParallelWorkerMain()` 用它们恢复：

```text
leader active snapshot:
  SerializeSnapshot()
  -> DSM
  -> worker RestoreSnapshot()
  -> PushActiveSnapshot()

leader 如果并行期间推进 command counter:
  本事务内 tuple 的 cmin/cmax 可见性语义改变
  worker 不会自动得到新的 curcid / combo CID 组合
```

这条边界对应课程标准里的一个长期规律：

```text
field 不是语义；
field + lifecycle state + visibility context 才是语义。
```

`curcid`、combo CID、active snapshot、transaction snapshot 必须作为一组解释。

### 14.3 DDL / utility command 边界

`ProcessUtility()` 相关路径会在 parallel mode 下拒绝许多 utility command。这里保护的是 catalog / relcache / invalidation 语义：

```text
worker bootstrap:
  RestoreLibraryState()
  RestoreRelationMap()
  RestoreReindexState()
  RestoreGUCState()
  InvalidateSystemCaches()

leader 并行期间执行 DDL:
  catalog tuple 变化
  relcache / catcache invalidation 传播
  relation map / dependency / lock 状态变化
```

并行 worker 不是一个持续同步 leader 全部 backend-local cache 的线程。它只在启动时恢复一组状态，然后通过正常 invalidation 机制处理运行中事件。让 leader 在并行期间随意做 DDL，会让“启动时复制的状态”不再是合理边界。

### 14.4 GUC 修改边界

`guc.c` 和 `guc_funcs.c` 中也能看到 parallel mode 检查。GUC 状态会被序列化：

```text
EstimateGUCStateSpace()
SerializeGUCState()
RestoreGUCState()
```

如果 leader 在 worker 已经恢复 GUC 后修改运行参数：

```text
work_mem
enable_* planner GUC
search_path 相关 GUC
extension 自定义 GUC
```

worker 不一定能同步看到。更麻烦的是某些 GUC check hook 会访问 catalog 或依赖当前 role/security context，所以恢复顺序必须固定，不能把 GUC 当作普通字符串字典。

### 14.5 临时命名空间和写路径边界

parallel worker 不应该创建自己的 pg_temp namespace，也不能安全访问 leader 的 backend-local temp namespace 状态。`FixedParallelState` 只记录 temp namespace state，让 worker 的解析语义与 leader 对齐；它不是授权 worker 随意创建或清理临时对象。

这解释了 worker commit/abort 与普通事务不同：

```text
普通 top-level transaction:
  写 commit/abort record
  清理 temp namespace
  释放本 backend 资源

parallel worker transaction:
  不写 commit/abort record
  不清理 leader 的 temp namespace
  只释放 worker 自己的 pins、cache refs、resource owners
```

parallel mode 的限制就是为了避免这些不同被用户可见语义放大。

## 15. 源码深挖：`pcxt_list` 的状态机

`pcxt_list` 的表面作用是“保存 active contexts”。更准确的模型是：

```text
backend-local cleanup registry for parallel ownership
```

一个 `ParallelContext` 在 leader 中经历这些状态：

```text
ALLOCATED
  -> REGISTERED_IN_PCXT_LIST
  -> DSM_INITIALIZED
  -> WORKERS_LAUNCHED
  -> WORKERS_FINISHED
  -> DESTROYING
  -> UNLINKED
  -> FREED
```

但代码没有显式 enum。状态由字段组合表达：

| 状态 | 字段组合 |
| --- | --- |
| 刚创建 | `node` 已入 `pcxt_list`，`seg == NULL`，`worker == NULL`。 |
| DSM 初始化后 | `toc != NULL`，`seg != NULL` 或 `private_memory != NULL`。 |
| worker 预算存在 | `worker != NULL`，`nworkers > 0`。 |
| 已启动部分 worker | `nworkers_launched > 0`，部分 `bgwhandle != NULL`。 |
| attach 已知 | `known_attached_workers != NULL`，计数递增。 |
| finish 后 | error queue 可能已 detach，`ExecParallelFinish()` 设置 executor 层 `finished`。 |
| destroy 中 | 已 `dlist_delete()`，后续不应再被事务 cleanup 看到。 |

### 15.1 为什么先入链表

`CreateParallelContext()` 在还没有 DSM、worker、error queue 时就把 context 放入 `pcxt_list`。原因是 ERROR 可以发生在任意一步：

```text
CreateParallelContext()
  -> pstrdup(library_name)
  -> pstrdup(function_name)
  -> caller 继续估算 DSM 空间
  -> InitializeParallelDSM()
  -> caller 写自定义 TOC 内容
```

如果只有 DSM 创建后才登记，那么中间 ERROR 会遗留已经分配的 context 或部分初始化状态。提前入链表让 cleanup 规则简单：

```text
只要 context 创建成功，它就必须在 pcxt_list 中，直到 DestroyParallelContext() 首行删除它。
```

### 15.2 为什么先从链表删除

`DestroyParallelContext()` 的第一个动作是：

```text
dlist_delete(&pcxt->node)
```

这看起来违反“先释放资源再删除登记”的直觉，但它是为了防止二次销毁。销毁过程可能涉及：

```text
TerminateBackgroundWorker()
shm_mq_detach()
dsm_detach()
WaitForParallelWorkersToExit()
pfree()
```

如果这些路径中发生 ERROR，而 context 仍在 `pcxt_list`，end-of-xact cleanup 会再次拿到同一个半销毁对象。先 unlink 等于声明：

```text
从现在起，这个 DestroyParallelContext() 调用独占清理责任。
```

### 15.3 子事务顺序为什么靠链表头

`AtEOSubXact_Parallel()` 的循环只清理链表头部 `subid == mySubId` 的 contexts：

```text
while !dlist_is_empty(pcxt_list):
  pcxt = dlist_head_element(...)
  if pcxt->subid != mySubId:
    break
  DestroyParallelContext(pcxt)
```

这依赖一个使用约束：

```text
ParallelContext 按创建顺序 push head；
子事务 cleanup 从当前子事务向外层展开；
当前子事务创建的 context 会位于链表前部。
```

如果调用方在错误的事务层级保存并复用 context，就会破坏这个假设。因此 `subid` 不只是诊断字段，它是 cleanup 边界。

### 15.4 commit warning 的意义

如果 `AtEOXact_Parallel(true)` 发现仍有 context：

```text
elog(WARNING, "leaked parallel context")
```

这是开发者错误信号，不是用户级业务错误。正常 commit 前，调用方应已经：

```text
WaitForParallelWorkersToFinish()
DestroyParallelContext()
ExitParallelMode()
```

commit 时才发现泄漏，说明某个路径漏掉 executor shutdown 或并行操作 cleanup。PostgreSQL 仍会清理，以保护进程和后续事务。

## 16. 源码深挖：`DestroyParallelContext()` 的顺序证明

`DestroyParallelContext()` 中每一步都有依赖。

### 16.1 先终止 worker，再 detach DSM

如果先 detach DSM：

```text
leader:
  dsm_detach(seg)

worker:
  仍在使用 DSM 中的 TOC、tuple queue、error queue、executor shared state
```

DSM 的 refcount 和 mapping 机制能避免底层段立即消失，但 leader 已经失去 TOC 和 queue handle，后续诊断和清理会变得不可靠。因此代码先对每个仍有 `error_mqh` 的 worker：

```text
TerminateBackgroundWorker(bgwhandle)
shm_mq_detach(error_mqh)
error_mqh = NULL
```

`TerminateBackgroundWorker()` 是请求 worker 停止，不是同步等待。真正等待在后面。

### 16.2 detach error queue 的语义

leader detach error queue 有两个效果：

1. leader 不再读取该 worker 的协议消息。
2. worker 如果继续写 shm_mq，可能看到 detached 状态并停止发送。

在正常路径中，应该先 `WaitForParallelWorkersToFinish()` 接收所有错误和 terminate 消息，再 destroy。直接 destroy 表示异常或强制清理路径，优先保证资源不泄漏。

### 16.3 detach DSM 后为什么还要等 worker exit

`dsm_detach(pcxt->seg)` 只处理 leader 的 mapping。worker 进程可能还在退出路径上执行：

```text
before_shmem_exit callbacks
ResourceOwner release
stats flush
DSM detach callbacks
```

事务 commit/abort 不能在 worker 未退出时完成，否则 leader 的事务状态可能先被清理，而 worker 仍引用 lock group、snapshot source PGPROC 或 shared state。因此：

```text
HOLD_INTERRUPTS()
WaitForParallelWorkersToExit(pcxt)
RESUME_INTERRUPTS()
```

这段不可中断等待是 deliberate trade-off：事务边界不能被普通 interrupt 打断到半清理状态。

### 16.4 postmaster death 的特殊处理

`WaitForParallelWorkersToExit()` 如果看到 `BGWH_POSTMASTER_DIED`：

```text
ereport(FATAL, "postmaster exited during a parallel transaction")
```

原因是 leader 已无法可靠知道 worker 何时退出，也无法保证共享状态的后续清理。此时继续当前 session 不安全，FATAL 让 backend 退出，由 postmaster 重启/清理体系处理。

### 16.5 private memory fallback 的 cleanup

当 `InitializeParallelDSM()` 因无 worker 或 DSM 创建失败而使用 `private_memory`：

```text
pcxt->seg == NULL
pcxt->private_memory != NULL
pcxt->toc 指向 private_memory 中的 TOC
```

`DestroyParallelContext()` 必须走不同分支：

```text
if pcxt->seg != NULL:
  dsm_detach()

if pcxt->private_memory != NULL:
  pfree()
```

这也是为什么调用方不应该直接根据 `toc != NULL` 判断是否存在 DSM。`toc` 只是对象目录，底层可能是 shared segment，也可能是私有内存。

## 17. 三条主流程对照

同一个 `ParallelContext` 在不同调用方中的流程相似，但语义压力不同。

### 17.1 parallel query

```text
ExecutorRun()
  -> EnterParallelMode()
  -> ExecGather() first call
     -> ExecInitParallelPlan()
        -> CreateParallelContext("postgres", "ParallelQueryMain", nworkers)
        -> InitializeParallelDSM()
        -> executor 写入 plan / params / DSA / tuple queues
     -> LaunchParallelWorkers()
     -> ExecParallelCreateReaders()
  -> tuple routing
  -> ExecShutdownGather()
     -> ExecParallelFinish()
     -> ExecParallelCleanup()
        -> DestroyParallelContext()
  -> ExitParallelMode()
```

这里 `ParallelContext` 是 executor 节点生命周期的一部分，但 cleanup 必须受事务保护，因为 worker 共享事务语义。

### 17.2 parallel VACUUM

`vacuumparallel.c` 也使用：

```text
EnterParallelMode()
CreateParallelContext("postgres", "parallel_vacuum_main", ...)
InitializeParallelDSM()
LaunchParallelWorkers()
WaitForParallelWorkersToFinish()
DestroyParallelContext()
ExitParallelMode()
```

不同点是它不是 `Gather` tuple routing，而是多个 worker 对 index vacuum 等任务做协作。相同点仍然是：

```text
parallel mode protects leader-visible state;
ParallelContext owns worker and DSM lifetime.
```

### 17.3 parallel index build

B-tree、GIN、BRIN parallel build 会注册 `_bt_parallel_build_main`、`_gin_parallel_build_main`、`_brin_parallel_build_main`。这些入口在 `InternalParallelWorkers[]` 中列出。

它们说明 `ParallelContext` 不是 executor-only abstraction：

```text
executor parallel query:
  entrypoint = ParallelQueryMain

parallel index build:
  entrypoint = _bt_parallel_build_main / ...

parallel vacuum:
  entrypoint = parallel_vacuum_main
```

共同边界是 `ParallelWorkerMain()` bootstrap，差异在 entrypoint 自己如何解释 TOC 中的业务状态。

## 18. 可复现诊断：从 SQL 到 `pcxt_list`

本节给一个可以在调试构建中复现的流程。

### 18.1 准备并行查询

```sql
SET max_parallel_workers_per_gather = 4;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

CREATE TABLE t_parallel_ctx AS
SELECT g AS id, md5(g::text) AS payload
FROM generate_series(1, 1000000) AS g;

ANALYZE t_parallel_ctx;

EXPLAIN (ANALYZE, VERBOSE)
SELECT count(*) FROM t_parallel_ctx WHERE id > 10;
```

目标是让计划中出现 `Gather` 或 parallel aggregate。

### 18.2 gdb 断点

```gdb
break EnterParallelMode
break CreateParallelContext
break InitializeParallelDSM
break LaunchParallelWorkers
break DestroyParallelContext
break AtEOXact_Parallel
commands CreateParallelContext
  silent
  printf "CreateParallelContext nworkers=%d\\n", nworkers
  continue
end
```

观察点：

```text
EnterParallelMode() 是否早于 CreateParallelContext()
CreateParallelContext() 后 pcxt_list 是否非空
DestroyParallelContext() 是否早于 ExitParallelMode()
commit 时 AtEOXact_Parallel() 是否不再看到 active context
```

### 18.3 打印 context

```gdb
p *pcxt
p pcxt->nworkers
p pcxt->nworkers_to_launch
p pcxt->nworkers_launched
p pcxt->seg
p pcxt->private_memory
p pcxt->toc
p pcxt->worker
```

这些字段要结合阶段解释：

```text
CreateParallelContext 后:
  seg/private_memory/toc 可能都还没有意义

InitializeParallelDSM 后:
  toc 必须存在
  seg 或 private_memory 二选一

LaunchParallelWorkers 后:
  nworkers_launched 才有意义
```

### 18.4 人为触发 cleanup

可以在 `ExecInitParallelPlan()` 创建 context 后、launch worker 前临时插入 `elog(ERROR, ...)` 做实验。预期行为：

```text
ERROR
  -> AbortCurrentTransaction
  -> AtEOXact_Parallel(false)
  -> DestroyParallelContext()
```

注意这是源码实验，不建议在普通工作树中长期保留改动。实验后应恢复源码。

## 19. 版本与边界说明

本课基于 `~/postgres-lab` 的 `bd4bd30ce6a7`。不同 PostgreSQL 版本中，以下细节可能变化：

```text
FixedParallelState 字段增减
worker progress message 类型
executor instrumentation / JIT 统计字段
并行 VACUUM / index build 使用的 TOC key
```

但以下语义较稳定：

```text
parallel mode 是事务安全限制
ParallelContext 是 leader-local ownership handle
DSM / TOC 显式传递 worker 启动状态
pcxt_list 是 end-of-xact cleanup registry
DestroyParallelContext() 是 worker/DSM/error queue 的统一销毁边界
```

源码阅读时要区分：

| 稳定语义 | 当前实现 |
| --- | --- |
| 并行期间禁止破坏 worker 已恢复的 leader 语义 | `parallelModeLevel` + 分散 `IsInParallelMode()` 检查。 |
| 并行上下文必须事务级兜底清理 | backend-local `pcxt_list`。 |
| worker 必须在 leader 事务完成前退出 | `DestroyParallelContext()` 中不可中断等待。 |
| 跨进程入口不能传函数指针 | DSM 中保存 library/function name。 |

## 20. 源码检查清单：新增并行调用方时怎么审

如果你在 PostgreSQL 中新增一个使用 `ParallelContext` 的调用方，不要只问“能不能启动 worker”。应该按下面清单审。

### 20.1 parallel mode 边界

必须明确：

```text
谁调用 EnterParallelMode()
谁保证所有 parallel contexts 销毁后再 ExitParallelMode()
ERROR 时是否会走事务 cleanup
是否可能嵌套进入 parallel mode
```

典型安全形态：

```text
EnterParallelMode()
PG_TRY:
  pcxt = CreateParallelContext(...)
  InitializeParallelDSM(pcxt)
  LaunchParallelWorkers(pcxt)
  WaitForParallelWorkersToFinish(pcxt)
  DestroyParallelContext(pcxt)
PG_CATCH:
  交给 AtEOXact_Parallel / AtEOSubXact_Parallel 兜底
ExitParallelMode()
```

实际 executor 不一定显式写 `PG_TRY`，因为事务 abort cleanup 是全局兜底。但你的代码仍要知道 cleanup 发生在哪里。

### 20.2 memory context 边界

检查所有 leader-local ownership 对象：

```text
ParallelContext:
  CreateParallelContext() 内部放到 TopTransactionContext

caller-specific state:
  如果 cleanup 依赖它，不能放到 per-tuple context

DSM payload:
  必须通过 shm_toc / DSA 管理，不能保存 backend-local pointer
```

常见错误是把某个后续 cleanup 还需要的数组分配在短生命周期 context 下，导致 ERROR 或 rescan 后悬空。

### 20.3 worker 数 fallback

新调用方必须回答：

```text
nworkers 被 InitializeParallelDSM() 降到 0 时怎么办？
LaunchParallelWorkers() 只启动部分 worker 时怎么办？
worker attach 前失败是否允许继续？
是否需要 WaitForParallelWorkersToAttach()？
```

parallel query 可以 0 worker fallback；某些并行 build 或 vacuum 也可能有 leader 本地路径。但如果算法必须至少一个 worker，就要显式检查并报错，而不是隐式等 queue。

### 20.4 cleanup 顺序

对每个资源列清楚：

```text
worker handle:
  DestroyParallelContext()

error queue:
  worker terminate / ProcessParallelMessage / DestroyParallelContext()

caller tuple or data queue:
  caller-specific finish

DSM:
  DestroyParallelContext()

DSA:
  caller-specific cleanup before DestroyParallelContext()

instrumentation:
  worker finish 后、DSM detach 前 retrieve
```

只要有一个资源的 cleanup 依赖另一个资源存在，就必须在代码顺序中体现。

## 21. 故障模式速查表

| 现象 | 优先检查 | 常见根因 |
| --- | --- | --- |
| commit 时出现 `leaked parallel context` | 正常路径是否调用 `DestroyParallelContext()` | executor shutdown 没覆盖某个 ERROR / early return 分支。 |
| `ExitParallelMode()` assertion failure | 是否还有 active `ParallelContext` | destroy 顺序错，或 cleanup 分支被跳过。 |
| worker 长时间不退出 | leader 是否 detach tuple queue，是否在 `ParallelFinish` 等待 | worker 阻塞在 shm_mq send 或持锁路径。 |
| worker 初始化失败但客户端错误很笼统 | 是否失败在 error queue attach 前 | DSM attach、TOC magic、早期 bootstrap 失败。 |
| workers planned 大于 launched | `nworkers_to_launch`、worker slot、DSM fallback | 系统 worker 资源不足或 DSM 创建失败。 |
| 并行期间 utility command 被拒绝 | `IsInParallelMode()` 检查 | parallel mode 正在保护 worker 已恢复的 leader 状态。 |

这个表的使用方式是先定位阶段：

```text
parallel mode 阶段？
context 创建阶段？
DSM 初始化阶段？
worker launch / attach 阶段？
tuple execution 阶段？
finish / cleanup 阶段？
```

不要直接从错误文本跳到 executor 或 planner 结论。并行基础设施的错误通常是 lifecycle 阶段错位。

## 22. 本节小结

`ParallelContext` 是 PostgreSQL 并行基础设施的 ownership 核心。parallel mode 先把 leader 的危险状态变化收住，`ParallelContext` 再把 worker、DSM、error queue 和 cleanup 顺序放到一个可由事务兜底销毁的对象里。

可迁移规律：

```text
跨进程并行不是只要共享数据结构；
必须先定义谁拥有并行生命周期、谁能在异常路径找到它、以及事务边界何时允许继续推进。
```
