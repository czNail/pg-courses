# PostgreSQL InitializeParallelDSM 与 leader 状态序列化

## 课程定位

前置知识：已经理解 `ParallelContext` 的 ownership 边界，知道 DSM / shm_toc 只能共享显式放进去的状态，不能自动复制 leader 的 backend-local 指针。

本节唯一主问题：

```text
并行 worker 不能直接继承 leader 的 backend-local 指针时，
InitializeParallelDSM() 如何用 shm_toc_estimator、FixedParallelState、snapshot、GUC、combo CID、
transaction state、session DSM handle 和 entrypoint key 构造 worker 可恢复的最小共享启动包？
```

核心矛盾：worker 必须像 leader 一样解释 tuple、catalog、权限、GUC、事务状态和入口函数；但这些状态大多是进程私有指针、内存上下文或全局变量，不能跨进程直接引用。PostgreSQL 的选择是把必要状态序列化进 DSM，再由 worker 反序列化恢复。

学完后应能判断：哪些 leader 状态必须进入 parallel DSM，哪些只放 fixed state，哪些需要独立 key，为什么 `InitializeParallelDSM()` 能在 no-worker 场景下退化到 backend-private memory。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

上一节讲了 `ParallelContext` 如何被创建、登记和销毁。本节进入它的核心初始化：

```text
CreateParallelContext()
  -> 调用方和 executor 先向 pcxt->estimator 追加自定义需求
  -> InitializeParallelDSM()
     -> 估算 parallel infrastructure 自己需要的状态
     -> 创建 DSM / fallback private memory
     -> 写入 fixed state、serialized state、error queues、entrypoint
```

下一节会讲这些 DSM 内容如何被 worker 启动和 attach。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
InitializeParallelDSM() 先估算 worker 启动所需的固定状态、GUC、snapshot、事务状态、
session DSM handle、error queues 和 entrypoint，
再创建一个 PARALLEL_MAGIC 的 shm_toc，
把 leader 当前语义序列化成若干 PARALLEL_KEY_* 条目，
使 worker 后续只凭 DSM handle 就能恢复到可运行状态。
```

tension 是：

```text
worker 要复现 leader 的执行语义
  vs
leader 的 backend-local 指针、全局变量和上下文不能跨进程共享
```

这不是完整复制 backend。它只复制 parallel worker 运行入口前必须一致的最小状态。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/parallel.c` | `InitializeParallelDSM()`、`FixedParallelState`、`PARALLEL_KEY_*`。 |
| 2 | `src/include/access/parallel.h` | `ParallelContext.estimator`、`seg`、`private_memory`、`toc`。 |
| 3 | `src/include/storage/shm_toc.h` | `shm_toc_estimator`、`shm_toc_estimate_chunk()`、`shm_toc_insert()`。 |
| 4 | `src/backend/utils/time/snapmgr.c` | `SerializeSnapshot()` / `RestoreSnapshot()` / `RestoreTransactionSnapshot()`。 |
| 5 | `src/backend/utils/misc/guc.c` | GUC state estimate / serialize / restore。 |
| 6 | `src/backend/access/common/session.c` | per-session DSM handle 与 `AttachSession()`。 |
| 7 | `src/backend/executor/execParallel.c` | executor 如何在 `InitializeParallelDSM()` 前追加自己的 DSM 需求。 |

## 4. 关键数据结构与状态

### `FixedParallelState`

`FixedParallelState` 是 `parallel.c` 内部结构，通过 `PARALLEL_KEY_FIXED` 放入 TOC。它保存 worker bootstrap 早期必须立即知道的固定字段：

| 字段类别 | 语义 |
| --- | --- |
| database / user / role | worker 用同一 database、session user、current role、security context 运行。 |
| temp namespace | 恢复 search path / temp namespace 相关语义。 |
| leader identity | `parallel_leader_pgproc`、pid、proc number，供锁组、snapshot、消息返回使用。 |
| timestamps | leader 的 transaction / statement start timestamp。 |
| serializable handle | worker attach 到 leader 的 serializable transaction。 |
| `last_xlog_end` + mutex | worker 回报 WAL 位置，leader finish 后更新 `XactLastRecEnd`。 |

这些字段多是值拷贝或共享身份，不是普通指针共享。

### `PARALLEL_KEY_*`

TOC 中的 key 形成 worker bootstrap 所需的目录：

| key | 内容 |
| --- | --- |
| `PARALLEL_KEY_FIXED` | `FixedParallelState`。 |
| `PARALLEL_KEY_LIBRARY` | leader 已加载库状态。 |
| `PARALLEL_KEY_GUC` | GUC 状态。 |
| `PARALLEL_KEY_COMBO_CID` | combo CID 状态。 |
| `PARALLEL_KEY_TRANSACTION_SNAPSHOT` | REPEATABLE READ / SERIALIZABLE 的事务级 snapshot。 |
| `PARALLEL_KEY_ACTIVE_SNAPSHOT` | 当前 active snapshot。 |
| `PARALLEL_KEY_TRANSACTION_STATE` | 事务状态。 |
| `PARALLEL_KEY_SESSION_DSM` | per-session DSM handle。 |
| `PARALLEL_KEY_PENDING_SYNCS` | pending fsync / sync 状态。 |
| `PARALLEL_KEY_REINDEX_STATE` | reindex 状态。 |
| `PARALLEL_KEY_RELMAPPER_STATE` | relation map 状态。 |
| `PARALLEL_KEY_UNCOMMITTEDENUMS` | 未提交 enum 状态。 |
| `PARALLEL_KEY_CLIENTCONNINFO` | client connection info。 |
| `PARALLEL_KEY_ERROR_QUEUE` | 每个 worker 一个 error shm_mq。 |
| `PARALLEL_KEY_ENTRYPOINT` | library name + function name。 |

raw bytes 不是语义。只有对应的 restore 函数按正确顺序消费这些 key，worker 才能安全运行。

## 5. 主流程源码 walkthrough

`InitializeParallelDSM()` 的时间线：

```text
MemoryContextSwitchTo(TopTransactionContext)
estimate FixedParallelState

if !INTERRUPTS_CAN_BE_PROCESSED():
  pcxt->nworkers = 0

if nworkers > 0:
  session_dsm_handle = GetSessionDsmHandle()
  if invalid:
    pcxt->nworkers = 0

if nworkers > 0:
  EstimateLibraryStateSpace()
  EstimateGUCStateSpace()
  EstimateComboCIDStateSpace()
  EstimateSnapshotSpace(transaction snapshot if needed)
  EstimateSnapshotSpace(active snapshot)
  EstimateTransactionStateSpace()
  EstimatePendingSyncsSpace()
  EstimateReindexStateSpace()
  EstimateRelationMapSpace()
  EstimateUncommittedEnumsSpace()
  EstimateClientConnectionInfoSpace()
  estimate error queues
  estimate entrypoint

segsize = shm_toc_estimate(&pcxt->estimator)
if nworkers > 0:
  pcxt->seg = dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS)

if seg exists:
  pcxt->toc = shm_toc_create(PARALLEL_MAGIC, dsm_segment_address(seg), segsize)
else:
  pcxt->nworkers = 0
  pcxt->private_memory = MemoryContextAlloc(TopMemoryContext, segsize)
  pcxt->toc = shm_toc_create(PARALLEL_MAGIC, private_memory, segsize)
```

注意 fallback：即使没有 worker，仍会有一个 TOC，只是它可能位于 backend-private memory。这让调用方可以用同一套 `shm_toc_allocate()` / `insert()` API 初始化结构，但后续不会 launch worker。

固定状态写入：

```text
fps = shm_toc_allocate(sizeof(FixedParallelState))
fps->database_id = MyDatabaseId
fps->authenticated_user_id = GetAuthenticatedUserId()
fps->session_user_id = GetSessionUserId()
fps->outer_user_id = GetCurrentRoleId()
GetUserIdAndSecContext(...)
GetTempNamespaceState(...)
fps->parallel_leader_pgproc = MyProc
fps->parallel_leader_pid = MyProcPid
fps->parallel_leader_proc_number = MyProcNumber
fps->xact_ts = GetCurrentTransactionStartTimestamp()
fps->stmt_ts = GetCurrentStatementStartTimestamp()
fps->serializable_xact_handle = ShareSerializableXact()
SpinLockInit(&fps->mutex)
fps->last_xlog_end = InvalidXLogRecPtr
shm_toc_insert(PARALLEL_KEY_FIXED, fps)
```

如果仍有 worker 预算，继续写入 serialized state、error queue 和 entrypoint。entrypoint 用字符串而不是函数指针：

```text
entrypointstate = library_name '\0' function_name '\0'
shm_toc_insert(PARALLEL_KEY_ENTRYPOINT, entrypointstate)
```

这是因为 `EXEC_BACKEND` 构建下，不同进程中函数地址不能假定相同。

## 6. 生命周期 / ownership / cleanup

### 谁估算

`ParallelContext.estimator` 在 `CreateParallelContext()` 后就可被调用方使用。executor 会先估算 `PlannedStmt`、ParamListInfo、tuple queue、instrumentation、DSA 和并行节点共享状态，然后才调用 `InitializeParallelDSM()`。

### 谁创建

leader 创建 DSM。worker 只通过 handle attach。`InitializeParallelDSM()` 中创建的 `pcxt->seg` 由 `DestroyParallelContext()` detach。

### 谁拥有 serialized bytes

TOC 中 bytes 的所有权属于 `ParallelContext` 的 DSM 生命周期。worker 反序列化后会在自己的 backend-local memory 中建立对应状态。

### cleanup

`DestroyParallelContext()` detach DSM 后，TOC 中所有 key 都失效。worker 如果还活着，会先被终止并等待退出。

## 7. 正确性机制层次

| 层次 | 机制 | 正确性目的 |
| --- | --- | --- |
| sizing | `shm_toc_estimator` | 在创建 DSM 前确定总大小，避免运行期扩容破坏 offset。 |
| identity | `FixedParallelState` | worker 知道 database、user、leader PGPROC、timestamps。 |
| visibility | serialized snapshots | worker 使用不早于 leader active snapshot 的 MVCC 视图。 |
| catalog / GUC | library + GUC + relmapper + reindex state | worker 的 catalog lookup 和 check hooks 与 leader 对齐。 |
| tuple semantics | combo CID、session DSM | worker 能解释 leader 事务内 tuple 和 typmod registry。 |
| error propagation | per-worker error queue | worker 初始化后错误可回传 leader。 |

`InitializeParallelDSM()` 对 `INTERRUPTS_CAN_BE_PROCESSED()` 的检查很关键。如果 leader 当前不能处理中断，就不能安全启动 worker，因为 worker 的错误和终止信号依赖 leader 后续 `CHECK_FOR_INTERRUPTS()` 处理。

## 8. 错误路径 / 异常路径 / fallback

典型 fallback：

1. `nworkers == 0`：不创建 DSM，使用 backend-private memory 建 TOC。
2. 不能处理中断：把 `pcxt->nworkers` 降为 0。
3. `GetSessionDsmHandle()` 失败：降为 0，避免 worker tuple typmod 不兼容。
4. `dsm_create(..., DSM_CREATE_NULL_IF_MAXSEGMENTS)` 返回 NULL：降为 0 并使用 private memory。

这些 fallback 都服务一个原则：

```text
能不并行就不并行；
不要为了并行计划本身破坏查询语义或直接失败。
```

但如果后续调用方假定一定有 worker，就必须检查 `pcxt->nworkers` 或 `nworkers_launched`。

## 9. 成本、资源与跨模块传播

DSM 大小来自两部分：

```text
parallel infrastructure:
  fixed state + serialized backend state + error queues + entrypoint

caller-specific state:
  executor plan / params / tuple queues / DSA / instrumentation
  或 parallel vacuum / index build 自己的 shared state
```

worker 越多，error queue、BufferUsage / WalUsage、tuple queue 和 instrumentation 空间都会随 worker 数线性增长。snapshot、GUC、transaction state 的大小则取决于当前 session / transaction 状态。

## 10. 观测与诊断入口

可观察入口：

| 入口 | 能看到什么 |
| --- | --- |
| gdb `p *pcxt` | `seg` 是否为 NULL、`private_memory` 是否启用、`nworkers` 是否被降为 0。 |
| gdb `shm_toc_lookup(pcxt->toc, PARALLEL_KEY_FIXED, false)` | fixed state 中的 leader identity。 |
| `EXPLAIN ANALYZE` | 计划 worker 和实际 worker 差异，间接反映 DSM/worker fallback。 |
| server log | DSM 创建失败、worker 初始化失败、parallel worker 错误。 |

课堂调试时可以给 `dsm_create()`、`GetSessionDsmHandle()`、`InitializeParallelDSM()` 设断点，观察从估算到 TOC 写入的顺序。

## 11. 常见误区

1. 误以为 worker 继承 leader 进程内存。PostgreSQL 是多进程模型，必须序列化。
2. 误以为 active snapshot 和 transaction snapshot 是同一回事。低隔离级别下可能没有 transaction-lifetime snapshot，但 worker 仍需要 active snapshot 设定 xmin 边界。
3. 误以为 DSM 创建失败应当 ERROR。对 parallel query 来说，退化为非并行通常更合适。
4. 误以为 TOC key 只是“配置清单”。它们定义了 worker bootstrap 的恢复顺序和语义边界。

## 12. 课堂实验

1. 在 `InitializeParallelDSM()` 中断，执行并行查询，打印 `pcxt->nworkers`、`pcxt->seg`、`pcxt->private_memory`。
2. 调低 `max_worker_processes` 或制造 DSM 压力，观察并行计划退化时 `Workers Launched` 的变化。
3. 在 `shm_toc_insert(PARALLEL_KEY_GUC, ...)`、`PARALLEL_KEY_ACTIVE_SNAPSHOT` 等位置设断点，确认哪些状态进入 DSM。

## 13. 讨论题

1. 为什么 `InitializeParallelDSM()` 要先让调用方估算自定义空间，再创建 DSM？
2. 为什么 entrypoint 要保存 library/function name，而不是函数指针？
3. 如果 worker 没有 session DSM handle，会在哪些 tuple / typmod 语义上出问题？

## 14. 源码深挖：`InitializeParallelDSM()` 的两阶段 sizing

`InitializeParallelDSM()` 的第一层设计是两阶段：

```text
estimate phase:
  所有参与者只声明需要多少空间和多少 key

allocation phase:
  创建 DSM / private memory
  用同一个 TOC 顺序 allocate / insert
```

这与 main shared memory 的启动期 sizing 很像，但边界不同：

| 维度 | main shmem | parallel DSM |
| --- | --- | --- |
| 创建时机 | postmaster 启动期 | 单次并行操作运行期 |
| 生命周期 | postmaster 生命周期 | `ParallelContext` 生命周期 |
| 大小来源 | 全局 GUC、核心子系统、preload 扩展 | parallel infrastructure + 调用方本次操作 |
| fallback | 启动失败 | worker 数降为 0，可能继续非并行执行 |
| 指针语义 | fork 后多数地址稳定，EXEC_BACKEND 需 attach | worker 只通过 DSM handle + TOC key 发现对象 |

### 14.1 为什么 caller 先估算

`CreateParallelContext()` 只初始化：

```text
shm_toc_initialize_estimator(&pcxt->estimator)
```

它不知道 executor、parallel vacuum、parallel index build 需要哪些业务状态。调用方必须在 `InitializeParallelDSM()` 前追加自己的估算：

```text
shm_toc_estimate_chunk(&pcxt->estimator, caller_size)
shm_toc_estimate_keys(&pcxt->estimator, caller_keys)
```

executor 侧就是典型例子：

```text
ExecInitParallelPlan()
  -> estimate FixedParallelExecutorState
  -> estimate query text
  -> estimate serialized PlannedStmt
  -> estimate ParamListInfo
  -> estimate BufferUsage / WalUsage arrays
  -> estimate tuple queues
  -> ExecParallelEstimate(planstate, &e)
  -> estimate instrumentation / JIT instrumentation
  -> estimate DSA minimum area
  -> InitializeParallelDSM(pcxt)
```

这个顺序说明：

```text
InitializeParallelDSM() 不是“只初始化基础设施”；
它也是所有 caller-specific TOC 空间的最终 segment creator。
```

如果 caller 在 DSM 创建后才发现空间不够，TOC 没有自动扩容机制，设计就会退化成复杂的二级 DSM 或错误路径。

### 14.2 为什么 infrastructure 自己也要估算

`InitializeParallelDSM()` 进入后先估算 fixed state：

```text
shm_toc_estimate_chunk(&pcxt->estimator, sizeof(FixedParallelState))
shm_toc_estimate_keys(&pcxt->estimator, 1)
```

只有确认可能启动 worker 后，才估算其它 worker bootstrap 状态：

```text
library state
GUC state
combo CID state
transaction snapshot
active snapshot
transaction state
session DSM handle
pending syncs
reindex state
relation map
uncommitted enums
client connection info
error queues
entrypoint
```

这避免 `nworkers == 0` 时为不需要的 worker 启动包浪费空间。即便 fallback 到 private memory，fixed state 仍会写入，因为 caller 可能使用同一个 TOC 初始化路径。

### 14.3 `shm_toc_estimate_keys()` 的易错点

每个 `shm_toc_insert()` 都需要 key slot。`parallel.c` 中有注释：

```text
If you add more chunks here, you probably need to add keys.
```

这是课程里值得强调的细节。估算 chunk 但忘记 key，可能不是在估算阶段失败，而是在后续 insert 时暴露。阅读源码时要成对检查：

```text
shm_toc_estimate_chunk(... library_len)
...
shm_toc_insert(PARALLEL_KEY_LIBRARY, libraryspace)
```

对调用方也是一样：

```text
estimate chunk
estimate keys
InitializeParallelDSM()
allocate
insert
```

这是一种手工协议。它没有类型系统自动保证。

## 15. `PARALLEL_KEY_*` 消费者地图

理解 DSM 启动包，最有效的方法不是背 key 名，而是问：

```text
谁写入？
谁读取？
读取后改变了哪个 backend-local 状态？
```

### 15.1 fixed state

| 项 | 写入者 | 读取者 | 恢复效果 |
| --- | --- | --- | --- |
| `database_id` | leader `InitializeParallelDSM()` | worker `ParallelWorkerMain()` | `BackgroundWorkerInitializeConnectionByOid()` 连接同一 database。 |
| user / role ids | leader | worker | 恢复 authenticated user、session user、current role。 |
| security context | leader | worker | `SetUserIdAndSecContext()` 恢复执行安全上下文。 |
| temp namespace ids | leader | worker | search path / temp namespace 语义与 leader 对齐。 |
| leader PGPROC / pid / proc number | leader | worker | 加入 lock group、恢复 snapshot、回传 signal。 |
| timestamps | leader | worker | 事务/语句时间函数与 leader 一致。 |
| serializable handle | leader | worker | worker attach 到同一 serializable transaction。 |
| `last_xlog_end` | worker | leader finish | 更新 leader `XactLastRecEnd`。 |

这张表说明 fixed state 不是“配置”。它是 worker 能否进入同一事务语义的根。

### 15.2 serialized library state

写入：

```text
library_len = EstimateLibraryStateSpace()
SerializeLibraryState(library_len, libraryspace)
shm_toc_insert(PARALLEL_KEY_LIBRARY, libraryspace)
```

读取：

```text
StartTransactionCommand()
RestoreLibraryState(libraryspace)
CommitTransactionCommand()
```

为什么要在 worker 中打开一个短事务来恢复 library state？因为加载库和初始化相关状态可能依赖事务上下文和 catalog access。恢复 library 必须早于 GUC：

```text
extension library 可能定义 custom GUC
  -> 没加载库就 RestoreGUCState()
  -> custom GUC 名称可能不存在
```

### 15.3 serialized GUC state

GUC 的恢复顺序被推迟到 snapshot / transaction / catalog 相关状态之后：

```text
RestoreGUCState(gucspace)
```

原因：

```text
GUC check hook 可能访问 catalog
session_authorization / role 相关 GUC 假定 role OID 已经设置
自定义 GUC 依赖 library restore
```

因此 GUC state 不是越早恢复越好。它需要前置身份和 catalog 可见性。

### 15.4 combo CID state

combo CID 解决同一事务内 tuple 的 `cmin` / `cmax` 组合表达问题。leader 写：

```text
EstimateComboCIDStateSpace()
SerializeComboCIDState()
```

worker 恢复：

```text
RestoreComboCIDState(combocidspace)
```

如果 worker 没有 combo CID state，它可能无法正确解释 leader 事务内已经写过的 tuple header。并行查询通常是读路径，但也可能遇到当前事务内已修改数据的可见性判断。

### 15.5 transaction snapshot 和 active snapshot

两种 snapshot 不同：

```text
transaction snapshot:
  只在 IsolationUsesXactSnapshot() 时序列化
  REPEATABLE READ / SERIALIZABLE 需要

active snapshot:
  当前执行路径正在使用
  worker 必须 PushActiveSnapshot()
```

低隔离级别下没有 transaction-lifetime snapshot，但 worker 仍需要设置 transaction snapshot 边界。`ParallelWorkerMain()` 中的策略是：

```text
asnapshot = RestoreSnapshot(active)
tsnapshot = transaction snapshot if exists else asnapshot
RestoreTransactionSnapshot(tsnapshot, leader_pgproc)
PushActiveSnapshot(asnapshot)
```

这样 worker 后续即使调用 `GetTransactionSnapshot()` 或 `GetLatestSnapshot()`，也不会获得早于 active snapshot 的视图。

### 15.6 transaction state

写入：

```text
tstatelen = EstimateTransactionStateSpace()
SerializeTransactionState()
```

恢复：

```text
StartParallelWorkerTransaction(tstatespace)
```

这个状态让 worker 进入一个“属于 leader 事务的 worker transaction”。它不是普通 top-level transaction，因此后续结束时：

```text
EndParallelWorkerTransaction()
```

而不是普通提交。worker 不写 commit record，最终事务结果仍由 leader 决定。

### 15.7 pending syncs、relation map、reindex state

这些状态看似和查询执行不直接相关，但它们影响 catalog 和 storage 解释：

```text
RestorePendingSyncs()
RestoreRelationMap()
RestoreReindexState()
```

例如 relation map 决定某些系统 catalog 的 relfilenode 映射；reindex state 影响正在重建索引时的访问语义。worker 如果看到不同状态，可能读取错误的 relation 或做错误的索引选择。

### 15.8 uncommitted enums

未提交 enum 值在事务内有特殊可见性。leader 如果在事务内创建 enum 值，worker 需要知道哪些 enum 仍未提交：

```text
SerializeUncommittedEnums()
RestoreUncommittedEnums()
```

这类状态很容易被忽视，因为它不是执行器主路径，却会影响表达式求值和 catalog 语义。

### 15.9 client connection info

`ClientConnectionInfo` 会被序列化并在 worker 中恢复：

```text
SerializeClientConnectionInfo()
RestoreClientConnectionInfo()
InitializeSystemUser()
```

这服务于审计、系统用户识别和某些安全上下文一致性。它也说明 parallel worker 不是匿名后台进程；它要尽量代表 leader 的会话上下文执行。

### 15.10 session DSM handle

`GetSessionDsmHandle()` 是一个关键 fallback 点。session DSM 中可能包含 record typmod registry 等 per-session objects。worker 与 leader 交换 tuple 时，如果 typmod registry 不一致，就可能无法解释 tuple descriptor。

所以：

```text
session_dsm_handle == DSM_HANDLE_INVALID
  -> pcxt->nworkers = 0
```

PostgreSQL 选择不启动 worker，而不是启动后赌 tuple 都不需要这些 session state。

### 15.11 error queue

每个 worker 一个 fixed-size error queue：

```text
PARALLEL_ERROR_QUEUE_SIZE * nworkers
```

leader 在 `InitializeParallelDSM()` 中创建 queue 并设置 receiver：

```text
mq = shm_mq_create(start, PARALLEL_ERROR_QUEUE_SIZE)
shm_mq_set_receiver(mq, MyProc)
pcxt->worker[i].error_mqh = shm_mq_attach(mq, pcxt->seg, NULL)
```

worker 后续按 `ParallelWorkerNumber` 找到自己的 queue，设置 sender 并 `pq_redirect_to_shm_mq()`。

### 15.12 entrypoint state

entrypoint 保存成：

```text
library_name\0function_name\0
```

worker 读取后：

```text
LookupParallelWorkerFunction(library_name, function_name)
```

核心函数使用 `InternalParallelWorkers[]`，外部库函数走 `load_external_function()`。这让并行基础设施可以服务 executor、VACUUM、index build 和扩展。

## 16. fallback 路径细化

`InitializeParallelDSM()` 里有多次把 `nworkers` 降为 0 的路径。它们不是等价的。

### 16.1 caller 本来就不需要 worker

```text
pcxt->nworkers == 0
```

这种情况下不会估算 worker bootstrap 状态，不创建 error queues，不保存 entrypoint。TOC 仍然可能存在，因为 caller 可能用统一路径初始化某些本地结构。

### 16.2 leader 不能处理中断

```text
if (!INTERRUPTS_CAN_BE_PROCESSED())
  pcxt->nworkers = 0
```

worker 错误传播依赖：

```text
worker sends PROCSIG_PARALLEL_MESSAGE
leader CHECK_FOR_INTERRUPTS()
ProcessParallelMessages()
```

如果 leader 当前处于不能处理中断的临界区，启动 worker 会让错误和退出消息无法及时处理。降级比启动后卡住更安全。

### 16.3 session DSM 不可用

```text
session_dsm_handle = GetSessionDsmHandle()
if invalid:
  pcxt->nworkers = 0
```

这是 tuple 解释正确性边界，不是性能问题。没有 session DSM，worker 和 leader 可能无法共享 record typmod registry 等状态。

### 16.4 DSM segment 达到上限

```text
pcxt->seg = dsm_create(segsize, DSM_CREATE_NULL_IF_MAXSEGMENTS)
if pcxt->seg == NULL:
  pcxt->nworkers = 0
  pcxt->private_memory = MemoryContextAlloc(TopMemoryContext, segsize)
```

`DSM_CREATE_NULL_IF_MAXSEGMENTS` 是一个明确的策略选择：

```text
DSM segment 数达到上限
  -> 返回 NULL
  -> 并行退化
  -> 查询仍可继续
```

对于普通 parallel query，这比直接 ERROR 更符合用户预期。但对于某些必须并行完成的内部算法，调用方仍需检查实际 worker 数。

## 17. executor 与基础设施的交界

`InitializeParallelDSM()` 本身不懂 executor plan。executor 在它之前追加估算，在它之后写入内容。

### 17.1 初始化前：executor 估算

```text
pcxt = CreateParallelContext("postgres", "ParallelQueryMain", nworkers)

shm_toc_estimate_chunk(&pcxt->estimator, sizeof(FixedParallelExecutorState))
shm_toc_estimate_chunk(&pcxt->estimator, query_len + 1)
shm_toc_estimate_chunk(&pcxt->estimator, pstmt_len)
shm_toc_estimate_chunk(&pcxt->estimator, paramlistinfo_len)
shm_toc_estimate_chunk(&pcxt->estimator, sizeof(BufferUsage) * nworkers)
shm_toc_estimate_chunk(&pcxt->estimator, sizeof(WalUsage) * nworkers)
shm_toc_estimate_chunk(&pcxt->estimator, PARALLEL_TUPLE_QUEUE_SIZE * nworkers)
ExecParallelEstimate(planstate, &e)
estimate instrumentation / JIT
estimate DSA minimum area
InitializeParallelDSM(pcxt)
```

这个边界很干净：

```text
parallel.c:
  worker bootstrap state

execParallel.c:
  executor-specific shared state
```

### 17.2 初始化后：executor allocate / insert

`InitializeParallelDSM()` 返回后，executor 开始真正 allocate：

```text
PARALLEL_KEY_EXECUTOR_FIXED
PARALLEL_KEY_QUERY_TEXT
PARALLEL_KEY_PLANNEDSTMT
PARALLEL_KEY_PARAMLISTINFO
PARALLEL_KEY_BUFFER_USAGE
PARALLEL_KEY_WAL_USAGE
PARALLEL_KEY_TUPLE_QUEUE
PARALLEL_KEY_INSTRUMENTATION
PARALLEL_KEY_JIT_INSTRUMENTATION
PARALLEL_KEY_DSA
```

这说明本节的 DSM 启动包不是最终全部内容。它只是先创建容器并写入基础设施状态，之后 caller 继续填充业务状态。

### 17.3 为什么 active snapshot assert 在 executor 中

`ExecInitParallelPlan()` 有：

```text
Assert(GetActiveSnapshot() == estate->es_snapshot)
```

原因是 `InitializeParallelDSM()` 会把 active snapshot 传给 worker，并由 worker 用它设置 executor snapshot。executor 不能在 child 侧设置一个和 leader `EState.es_snapshot` 不同的视图。

这是跨模块 invariant：

```text
snapmgr.c 的 active snapshot
  必须等于
executor EState 的 es_snapshot
  才能安全传给 worker
```

## 18. 源码实验手册

### 18.1 观察 TOC 大小

gdb：

```gdb
break InitializeParallelDSM
commands
  silent
  printf "nworkers before=%d\\n", pcxt->nworkers
  continue
end

break shm_toc_estimate
```

在 `InitializeParallelDSM()` 末尾打印：

```gdb
p pcxt->nworkers
p pcxt->seg
p pcxt->private_memory
p pcxt->toc
```

解释：

```text
seg != NULL:
  真正创建了 DSM，可启动 worker

seg == NULL && private_memory != NULL:
  fallback 到本 backend 私有 TOC，不会启动 worker
```

### 18.2 检查 fixed state

```gdb
set $fps = (FixedParallelState *) shm_toc_lookup(pcxt->toc, PARALLEL_KEY_FIXED, 0)
p *$fps
p $fps->database_id
p $fps->parallel_leader_pid
p $fps->parallel_leader_pgproc
```

如果查询在 SERIALIZABLE 隔离级别下执行，可以观察：

```gdb
p $fps->serializable_xact_handle
```

### 18.3 检查 snapshot key

在 `IsolationUsesXactSnapshot()` 为 false 的 READ COMMITTED 下：

```text
PARALLEL_KEY_TRANSACTION_SNAPSHOT 可能不存在
PARALLEL_KEY_ACTIVE_SNAPSHOT 必须存在
```

在 REPEATABLE READ 下：

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
EXPLAIN (ANALYZE) SELECT count(*) FROM t_parallel_ctx;
COMMIT;
```

gdb 中检查：

```gdb
p shm_toc_lookup(pcxt->toc, PARALLEL_KEY_TRANSACTION_SNAPSHOT, 1)
p shm_toc_lookup(pcxt->toc, PARALLEL_KEY_ACTIVE_SNAPSHOT, 1)
```

### 18.4 检查 fallback

制造 fallback 的方法：

```text
降低 max_parallel_workers 或 max_parallel_workers_per_gather:
  主要影响 LaunchParallelWorkers()

达到 DSM segment 上限:
  影响 InitializeParallelDSM()

让 GetSessionDsmHandle() 返回 invalid:
  需要源码实验或断点修改返回值
```

观察 `EXPLAIN ANALYZE`：

```text
Workers Planned: N
Workers Launched: 0
```

这不能单独说明是哪类 fallback，需要结合 gdb 或日志判断发生在 DSM 初始化还是 worker launch 阶段。

## 19. 常见版本差异与阅读防线

并行启动包会随版本演化。例如新版本可能增加：

```text
新的 progress message 类型
新的 per-session state
新的 instrumentation 字段
新的 executor DSM key
```

阅读时不要把字段清单当成系统本质。稳定的是这条线：

```text
worker 启动前必须恢复 leader 语义
  -> leader 把不可共享的 backend-local 状态序列化
  -> DSM TOC 只保存 bytes 和 key
  -> worker 按固定顺序恢复成自己的 backend-local 状态
```

如果将来增加一个需要同步的 backend-local 状态，判断它是否应该进入 `InitializeParallelDSM()` 可以问：

1. worker 在执行 entrypoint 前是否必须看到它？
2. 它能否通过普通 shared memory / invalidation 在运行中获得？
3. 它是否依赖恢复顺序，例如必须早于 GUC 或晚于 database connection？
4. 它是否影响 tuple visibility、catalog lookup、权限、安全上下文或错误诊断？
5. 它是否必须在 no-worker fallback 时也保留统一 API？

## 20. 源码检查清单：新增一个需要 worker 恢复的状态

如果你发现某个 backend-local 状态在 parallel worker 中不一致，不能马上把它塞进 DSM。先按下面顺序判断。

### 20.1 是否真需要启动前恢复

问：

```text
worker 是否在 entrypoint 开始前就需要这个状态？
worker 能否通过普通 catalog lookup / syscache invalidation 得到？
worker 是否只在某个具体 plan node 中需要？
```

如果只在某个 executor node 中需要，应该放到 node-specific DSM callback，而不是 `InitializeParallelDSM()` 的通用启动包。

### 20.2 状态是否可序列化

危险状态：

```text
backend-local pointer
MemoryContext pointer
ResourceOwner pointer
File descriptor
local hash table bucket address
回调函数指针
```

这些不能原样写进 DSM。要转换成：

```text
Oid / Xid / LSN / enum / flags
DSM handle
dsa_pointer
TOC key
字符串形式的 library/function name
可反序列化的 byte stream
```

### 20.3 恢复顺序

新增状态要明确插入点：

```text
database connection 前？
library restore 前？
transaction state 后？
snapshot 后？
GUC restore 前？
security context 后？
entrypoint 前？
```

例如依赖 catalog 的状态通常不能早于 database connection 和 snapshot；定义 GUC 的 library 必须早于 GUC restore。

### 20.4 fallback 语义

新增状态如果无法序列化，应该：

```text
ERROR？
降 nworkers 到 0？
跳过某个优化？
交给 caller fallback？
```

parallel query 通常更偏向降级；但如果状态不一致会产生错误结果，就不能 silent fallback 到不完整 worker 状态。

## 21. worker 启动包与 executor DSM 的边界图

```text
ParallelContext / parallel.c
  PARALLEL_KEY_FIXED
  PARALLEL_KEY_LIBRARY
  PARALLEL_KEY_GUC
  PARALLEL_KEY_COMBO_CID
  PARALLEL_KEY_TRANSACTION_SNAPSHOT
  PARALLEL_KEY_ACTIVE_SNAPSHOT
  PARALLEL_KEY_TRANSACTION_STATE
  PARALLEL_KEY_SESSION_DSM
  PARALLEL_KEY_ERROR_QUEUE
  PARALLEL_KEY_ENTRYPOINT

execParallel.c
  PARALLEL_KEY_EXECUTOR_FIXED
  PARALLEL_KEY_QUERY_TEXT
  PARALLEL_KEY_PLANNEDSTMT
  PARALLEL_KEY_PARAMLISTINFO
  PARALLEL_KEY_BUFFER_USAGE
  PARALLEL_KEY_WAL_USAGE
  PARALLEL_KEY_TUPLE_QUEUE
  PARALLEL_KEY_DSA
  PARALLEL_KEY_INSTRUMENTATION
  PARALLEL_KEY_JIT_INSTRUMENTATION

plan node callbacks
  key < 2^32 or node-specific key scheme
  shared scan / hash / append / instrumentation state
```

这张图能帮助定位“某个状态应该放哪里”：

```text
所有 parallel worker 都需要，且与 executor 无关:
  parallel.c 启动包

parallel query worker 需要，且 executor 通用:
  execParallel.c keys

只有某个 plan node 需要:
  node-specific callbacks
```

## 22. 本节小结

`InitializeParallelDSM()` 把 leader 的并行启动语义压缩成一个可 attach 的 DSM 启动包。它不复制整个 backend，而是把 worker bootstrap 必须一致的状态按 key 放入 TOC。

可迁移规律：

```text
跨进程恢复状态时，不要共享进程私有指针；
要显式定义最小状态集、序列化格式、恢复顺序和 fallback 语义。
```
