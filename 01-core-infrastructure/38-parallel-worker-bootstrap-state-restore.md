# PostgreSQL ParallelWorkerMain 状态恢复与 worker backend bootstrap

## 课程定位

前置知识：已经理解 `InitializeParallelDSM()` 写入的 `PARALLEL_KEY_*` 状态，也理解 worker attach error queue 是错误可回传的边界。

本节唯一主问题：

```text
ParallelWorkerMain() 为什么要先 attach DSM / TOC 和 error queue，
再恢复 database connection、role/security context、transaction snapshot、GUC、session DSM、
serializable xact 与 lock group membership，
最后才调用 ParallelWorkerEntryPoint？
```

核心矛盾：worker 必须在足够早的时候把错误回传 leader，但又必须按严格顺序恢复 leader 的身份、事务、snapshot、GUC 和 catalog 相关状态；顺序错了，轻则诊断丢失，重则 catalog lookup、锁、可见性或安全上下文与 leader 不一致。

学完后应能判断：worker bootstrap 中哪些状态必须先于 error queue、transaction、snapshot、GUC 或 entrypoint 恢复，哪些失败能经 error queue 回传，哪些早期失败只能表现为 attach/startup failure。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

本节接在 worker launch 之后：

```text
postmaster 启动 dynamic background worker
  -> ParallelWorkerMain(main_arg = DSM handle)
     -> attach DSM / error queue
     -> restore leader state
     -> EnterParallelMode()
     -> entrypoint(seg, toc)
```

后续课程会讲 worker 运行中如何通过 error queue 回传 ERROR / NOTICE，以及 executor worker 如何运行 `ParallelQueryMain()`。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ParallelWorkerMain() 用 bgw_main_arg attach parallel DSM，
用 bgw_extra 得到 worker number 并 attach 自己的 error queue，
加入 leader lock group，恢复身份、database connection、library、transaction state、snapshot、GUC、
session DSM、serializable transaction 和安全上下文，
最后 lookup entrypoint 并在 parallel mode 中调用它。
```

tension 是：

```text
worker 要尽早具备错误回传能力
  vs
worker 在真正执行入口前必须完整恢复 leader 的运行语义
```

因此 bootstrap 顺序不是实现细节，而是正确性边界。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/parallel.c` | `ParallelWorkerMain()` 主流程、`ParallelWorkerShutdown()`。 |
| 2 | `src/backend/libpq/pqmq.c` | `pq_redirect_to_shm_mq()` 如何把协议消息写入 shm_mq。 |
| 3 | `src/backend/access/transam/xact.c` | `SetParallelStartTimestamps()`、`StartParallelWorkerTransaction()`、worker commit/abort 差异。 |
| 4 | `src/backend/utils/time/snapmgr.c` | `RestoreTransactionSnapshot()` 和 active snapshot 恢复。 |
| 5 | `src/backend/storage/lmgr/lock.c` | lock group member 语义。 |
| 6 | `src/backend/executor/execParallel.c` | `ParallelQueryMain()` 如何作为 executor entrypoint 被调用。 |

## 4. 关键数据结构与状态

### worker 早期全局状态

| 状态 | 作用 |
| --- | --- |
| `ParallelWorkerNumber` | 从 `MyBgworkerEntry->bgw_extra` 读取，决定 error queue / tuple queue slot。 |
| `InitializingParallelWorker` | bootstrap 期间标记，某些路径可据此区别普通 backend。 |
| `MyFixedParallelState` | 指向 DSM 中的 `FixedParallelState`，供其他模块读取 leader identity。 |
| `ParallelLeaderPid` / `ParallelLeaderProcNumber` | worker 退出时通知 leader。 |

### error queue 边界

worker attach error queue 后调用：

```text
pq_redirect_to_shm_mq(seg, mqh)
pq_set_parallel_leader(fps->parallel_leader_pid, fps->parallel_leader_proc_number)
```

从这一刻起，worker 的 ErrorResponse、NoticeResponse、NotifyResponse、progress 等协议消息可以走 shm_mq 回到 leader。

## 5. 主流程源码 walkthrough

`ParallelWorkerMain()` 早期启动：

```text
InitializingParallelWorker = true
BackgroundWorkerUnblockSignals()
ParallelWorkerNumber = bgw_extra
CurrentMemoryContext = AllocSetContextCreate(TopMemoryContext, "Parallel worker", ...)

seg = dsm_attach(main_arg)
toc = shm_toc_attach(PARALLEL_MAGIC, dsm_segment_address(seg))
fps = shm_toc_lookup(PARALLEL_KEY_FIXED)
MyFixedParallelState = fps

ParallelLeaderPid = fps->parallel_leader_pid
ParallelLeaderProcNumber = fps->parallel_leader_proc_number
before_shmem_exit(ParallelWorkerShutdown, seg)

error_queue_space = shm_toc_lookup(PARALLEL_KEY_ERROR_QUEUE)
mq = error_queue_space + ParallelWorkerNumber * PARALLEL_ERROR_QUEUE_SIZE
shm_mq_set_sender(mq, MyProc)
mqh = shm_mq_attach(mq, seg, NULL)
pq_redirect_to_shm_mq(seg, mqh)
```

这解释了为什么 worker 在 attach error queue 之前的错误只能表现为“failed to initialize”。leader 还没有可靠通道接收具体 ErrorData。

随后恢复运行语义：

```text
BecomeLockGroupMember(fps->parallel_leader_pgproc, fps->parallel_leader_pid)
SetParallelStartTimestamps(fps->xact_ts, fps->stmt_ts)

entrypointstate = shm_toc_lookup(PARALLEL_KEY_ENTRYPOINT)
entrypt = LookupParallelWorkerFunction(library_name, function_name)

SetAuthenticatedUserId(...)
SetSessionAuthorization(...)
SetCurrentRoleId(...)
BackgroundWorkerInitializeConnectionByOid(...)
SetClientEncoding(GetDatabaseEncoding())

StartTransactionCommand()
RestoreLibraryState(PARALLEL_KEY_LIBRARY)
CommitTransactionCommand()

StartParallelWorkerTransaction(PARALLEL_KEY_TRANSACTION_STATE)
RestorePendingSyncs()
RestoreRelationMap()
RestoreReindexState()
RestoreComboCIDState()
AttachSession(PARALLEL_KEY_SESSION_DSM)

RestoreSnapshot(active / transaction snapshot)
RestoreTransactionSnapshot(tsnapshot, leader PGPROC)
PushActiveSnapshot(asnapshot)
InvalidateSystemCaches()

RestoreGUCState()
SetUserIdAndSecContext()
SetTempNamespaceState()
RestoreUncommittedEnums()
RestoreClientConnectionInfo()
InitializeSystemUser()
AttachSerializableXact()

InitializingParallelWorker = false
EnterParallelMode()
entrypt(seg, toc)
ExitParallelMode()
```

顺序中的几个关键点：

1. lock group membership 必须早于可能拿 heavyweight lock 的操作。
2. timestamp 恢复必须早于会启动事务的路径。
3. library restore 早于 GUC restore，因为自定义库可能定义 GUC。
4. snapshot restore 后要 `InvalidateSystemCaches()`，因为可见性边界变了。
5. current user / security context 要在 GUC restore 后恢复，避免 GUC check hook 顺序问题。

## 6. 生命周期 / ownership / cleanup

worker 有自己的 ResourceOwner、memory context、buffer pin、catcache ref。它不是共享 leader 的资源 owner。

退出时：

```text
ParallelWorkerShutdown()
  -> 通知 leader
  -> 必要时 report last WAL end
  -> detach DSM / queues
```

worker 的 commit/abort 类似顶层事务 cleanup，但有差异：不写 commit/abort record，不清理 leader 的 temp namespace，最终事务结果由 leader 负责。

## 7. 正确性机制层次

| 层次 | 机制 | 不变量 |
| --- | --- | --- |
| 错误传播 | error queue + `pq_redirect_to_shm_mq()` | worker 错误能回到 leader。 |
| 锁 | lock group member | group 内锁等待不制造不可检测死锁。 |
| 身份 | user / role / security context | 权限检查与 leader 一致。 |
| database | `BackgroundWorkerInitializeConnectionByOid()` | catalog / relcache 指向同一 database。 |
| 可见性 | restored snapshots + transaction state | worker 看到的 tuple 与 leader 计划语义一致。 |
| GUC / library | restore library before GUC | 自定义 GUC 和 check hook 可正确恢复。 |
| SSI | `AttachSerializableXact()` | serializable 事务依赖追踪仍属于 leader 事务。 |

## 8. 错误路径 / 异常路径 / fallback

关键错误窗口：

1. `dsm_attach()` / `shm_toc_attach()` 失败：worker 还未 attach error queue，leader 只能看到初始化失败。
2. attach error queue 后 ERROR：ErrorResponse 通过 shm_mq 回传，leader rethrow。
3. `BecomeLockGroupMember()` 失败：leader 已消失，worker 安静返回。
4. entrypoint 内 ERROR：走并行消息传播，由 leader 在 `ProcessParallelMessages()` 中抛出。

worker bootstrap 的目标不是“尽量继续”。任何恢复状态失败都意味着 worker 无法保证 leader 语义，必须停止。

## 9. 成本、资源与跨模块传播

worker 启动成本包括：

```text
background worker bootstrap
DSM attach 和 TOC lookup
database connection init
library load / GUC restore
transaction and snapshot restore
session DSM attach
cache invalidation
entrypoint-specific executor init
```

这解释了为什么并行查询对短查询可能不划算，也解释了 planner 成本模型中 parallel setup cost 的来源。

## 10. 观测与诊断入口

| 入口 | 说明 |
| --- | --- |
| server log | error queue attach 前的失败通常只有 log 有细节。 |
| `pg_stat_activity.backend_type` | 可看到 parallel worker。 |
| wait event | leader 可能等待 `BgWorkerStartup`、`ParallelFinish`。 |
| gdb | worker 中打印 `ParallelWorkerNumber`、`MyFixedParallelState`、TOC key。 |

调试建议：在 `ParallelWorkerMain()` 的 error queue attach 前后各设断点，观察同样的 ERROR 在 leader 侧表现差异。

## 11. 常见误区

1. 误以为 worker 是 fork 出来就天然有 leader 的 backend-local 状态。dynamic background worker 需要显式恢复。
2. 误以为先连接数据库再 attach error queue 更自然。那会让连接和 catalog 初始化错误无法回传 leader。
3. 误以为 GUC restore 可以最早做。自定义 GUC 的定义依赖 library restore，check hook 又可能依赖 database / role / catalog 状态。
4. 误以为 worker 可以独立提交事务。parallel worker 的事务状态是 leader 事务的执行分支，最终提交/回滚由 leader 负责。

## 12. 课堂实验

1. 在 `ParallelWorkerMain()` 中对 `pq_redirect_to_shm_mq()` 前后设断点，手工触发 ERROR，比较 leader 看到的诊断。
2. 打印 `ParallelWorkerNumber`，确认它与 `bgw_extra` 和 error queue slot 对应。
3. 在 `RestoreGUCState()`、`RestoreTransactionSnapshot()`、`AttachSerializableXact()` 设断点，观察 worker 恢复顺序。

## 13. 讨论题

1. 为什么 lock group membership 要在任何可能拿 heavyweight lock 的操作之前完成？
2. 低隔离级别下没有事务级 snapshot，为什么 worker 仍要用 active snapshot 恢复 transaction snapshot 边界？
3. 如果 worker 先执行 entrypoint 再恢复 GUC，会破坏哪些用户可见语义？

## 14. 源码深挖：bootstrap 顺序为什么不能重排

`ParallelWorkerMain()` 最容易被误读成一串“恢复状态”的 API 调用。实际上它是一条严格的依赖链：

```text
先建立最小诊断通道
  -> 再加入 lock group
  -> 再恢复时间戳和入口函数
  -> 再恢复身份和 database connection
  -> 再加载 library
  -> 再启动 parallel worker transaction
  -> 再恢复 catalog / transaction / session / snapshot state
  -> 再恢复 GUC 和安全上下文
  -> 最后进入 entrypoint
```

这条顺序的每一步都在避免一个具体错误窗口。

### 14.1 signal handler 早于 DSM attach

```text
InitializingParallelWorker = true
BackgroundWorkerUnblockSignals()
```

dynamic background worker 启动时信号最初是屏蔽的。worker 需要恢复可响应状态，否则 leader 或 postmaster 的信号可能无法及时处理。

`InitializingParallelWorker` 则告诉其它模块：

```text
当前 backend 处于 parallel worker bootstrap 阶段，
某些输出变量或身份状态会由 ParallelWorkerMain 手动设置。
```

这不是普通后台进程启动路径。

### 14.2 worker number 早于 error queue

```text
memcpy(&ParallelWorkerNumber, MyBgworkerEntry->bgw_extra, sizeof(int))
```

worker number 决定了它在所有 per-worker arrays 中的位置：

```text
error queue offset
tuple queue offset
BufferUsage / WalUsage slot
instrumentation slot
JIT instrumentation slot
```

如果 worker number 读取错误，worker 可能写到另一个 worker 的 queue 或 statistics slot，表现为 leader 收到错乱消息、tuple stream 混乱或 instrumentation 无法解释。

### 14.3 DSM / TOC 早于 error queue

error queue 本身在 parallel DSM 中，所以 worker 必须先：

```text
seg = dsm_attach(main_arg)
toc = shm_toc_attach(PARALLEL_MAGIC, dsm_segment_address(seg))
fps = shm_toc_lookup(PARALLEL_KEY_FIXED)
```

这一步失败时，worker 还没有协议通道告诉 leader 具体错误。leader 只能通过 bgworker stopped + sender NULL 判断初始化失败。

因此 DSM handle 是 worker bootstrap 的根。如果 `bgw_main_arg` 错、DSM 已被 detach、TOC magic 不匹配，后续所有恢复都没有意义。

### 14.4 fixed state 早于 leader signal

worker 从 `FixedParallelState` 拿到：

```text
parallel_leader_pid
parallel_leader_proc_number
parallel_leader_pgproc
```

然后设置：

```text
ParallelLeaderPid = fps->parallel_leader_pid
ParallelLeaderProcNumber = fps->parallel_leader_proc_number
before_shmem_exit(ParallelWorkerShutdown, PointerGetDatum(seg))
```

如果 worker 退出时不通知 leader，leader 可能继续等：

```text
WaitForParallelWorkersToAttach()
WaitForParallelWorkersToFinish()
TupleQueueReaderNext()
```

`ParallelWorkerShutdown()` 即使在非正常退出时也会发送 `PROCSIG_PARALLEL_MESSAGE`，促使 leader 再读一次 error queue。

### 14.5 error queue 早于 database connection

代码有意在连接 database 之前建立 error queue：

```text
error_queue_space = shm_toc_lookup(PARALLEL_KEY_ERROR_QUEUE)
mq = error_queue_space + worker_number * PARALLEL_ERROR_QUEUE_SIZE
shm_mq_set_sender(mq, MyProc)
mqh = shm_mq_attach(mq, seg, NULL)
pq_redirect_to_shm_mq(seg, mqh)
```

否则下面这些操作的错误都无法回传 leader：

```text
BecomeLockGroupMember()
BackgroundWorkerInitializeConnectionByOid()
RestoreLibraryState()
StartParallelWorkerTransaction()
RestoreSnapshot()
RestoreGUCState()
```

这就是“错误可回传边界”。本课所有诊断都应该围绕它分前后。

### 14.6 lock group 早于 heavyweight lock

worker 加入 lock group：

```text
BecomeLockGroupMember(fps->parallel_leader_pgproc,
                      fps->parallel_leader_pid)
```

注释说明它必须早于任何可能获取 heavyweight lock 的操作。否则可能出现：

```text
worker 持有/等待一个 lock
leader 持有/等待另一个 lock
外部 backend 等待 leader
deadlock detector 不知道 worker 和 leader 属于同一 parallel group
```

加入 lock group 之后，锁管理器能把 parallel group 作为一个协作单元处理，避免不可检测死锁。

### 14.7 timestamp 早于事务启动

```text
SetParallelStartTimestamps(fps->xact_ts, fps->stmt_ts)
```

必须早于任何可能启动事务的路径。否则 `transaction_timestamp()`、`statement_timestamp()` 或 xact.c 内部 assert 会看到 worker 自己的启动时间，而不是 leader 的语义时间。

### 14.8 entrypoint lookup 早于 library restore 的微妙点

`ParallelWorkerMain()` 先读取 entrypoint state，并调用 `LookupParallelWorkerFunction()`。对于核心 backend 函数，`InternalParallelWorkers[]` 能直接找到；对于外部库，可能需要 load。

同时后面还会 `RestoreLibraryState()`。这看似重复，但职责不同：

```text
entrypoint lookup:
  找到本次 worker 要执行的 main function

library state restore:
  恢复 leader 已加载的库集合，使 worker 的运行环境对齐
```

不能把二者混为一谈。

### 14.9 user / role 早于 connection

```text
SetAuthenticatedUserId()
SetSessionAuthorization()
SetCurrentRoleId()
BackgroundWorkerInitializeConnectionByOid()
```

连接数据库时需要知道 authenticated user，且 worker 会绕过部分 allowconn / role login 检查：

```text
BGWORKER_BYPASS_ALLOWCONN
BGWORKER_BYPASS_ROLELOGINCHECK
```

理由是 leader 已经通过这些检查；parallel worker 不应因为连接期间环境变化而让一个已经开始执行的查询失败。

### 14.10 library 早于 GUC

`RestoreLibraryState()` 在一个短事务中运行：

```text
StartTransactionCommand()
RestoreLibraryState(libraryspace)
CommitTransactionCommand()
```

然后很久以后才：

```text
RestoreGUCState(gucspace)
```

这条顺序保护 extension-defined GUC。没有加载库，自定义 GUC 不存在；过早 restore GUC 会失败或跳过语义。

### 14.11 transaction state 早于 snapshot

worker 先：

```text
StartParallelWorkerTransaction(tstatespace)
```

再恢复 snapshot：

```text
RestoreTransactionSnapshot(tsnapshot, leader_pgproc)
PushActiveSnapshot(asnapshot)
```

snapshot 不是孤立对象。它依赖事务状态、ProcArray / PGPROC 语义、TransactionXmin 等边界。先恢复 snapshot 而没有 parallel worker transaction，会让 snapmgr 和 xact.c 的不变量错位。

### 14.12 catalog state 早于 GUC check hook

```text
RestorePendingSyncs()
RestoreRelationMap()
RestoreReindexState()
RestoreComboCIDState()
AttachSession()
RestoreSnapshot()
InvalidateSystemCaches()
RestoreGUCState()
```

GUC check hook 可能做 catalog lookup。catalog lookup 又依赖：

```text
database connection
relation map
snapshot
system cache invalidation
role/security context
```

所以 GUC 恢复不能放在 database connection 刚完成之后。

### 14.13 security context 在 GUC 后恢复

代码注释指出，`SetUserIdAndSecContext()` 不能早于 `RestoreGUCState()`，因为 session_authorization 和 role 的 GUC 恢复有自己的假设。

这说明 bootstrap 顺序不是简单的“安全越早越好”。某些状态需要临时处在可恢复状态，等 GUC 完成后再设置最终 security context。

## 15. worker 恢复的状态分类

### 15.1 进程级状态

```text
ParallelWorkerNumber
InitializingParallelWorker
CurrentMemoryContext
signal handlers
debug_query_string later in ParallelQueryMain
pgstat activity
```

这些是 worker 自己的 backend-local 状态，不来自 leader。它们让 worker 成为一个能运行 PostgreSQL backend 代码的进程。

### 15.2 leader identity state

```text
ParallelLeaderPid
ParallelLeaderProcNumber
MyFixedParallelState
parallel_leader_pgproc
```

这些状态用于：

```text
向 leader 发送 PROCSIG_PARALLEL_MESSAGE
加入 leader lock group
把 restored snapshot 绑定到 leader PGPROC
上报 last WAL end
```

### 15.3 session / authorization state

```text
authenticated_user_id
session_user_id
outer_user_id
current_user_id
sec_context
session_user_is_superuser
role_is_superuser
client connection info
system user
```

这些状态让权限检查、审计信息和安全上下文与 leader 一致。worker 不重新认证用户。

### 15.4 transaction / visibility state

```text
transaction state
combo CID
transaction snapshot
active snapshot
serializable transaction handle
uncommitted enums
```

这些状态决定 tuple visibility、当前事务内写入解释、SSI 冲突检测和 enum 可见性。

### 15.5 catalog / storage state

```text
pending syncs
relation map
reindex state
session DSM
temp namespace state
```

这些状态影响 catalog lookup、relation file mapping、session-local typmod registry、temp namespace 搜索语义。

## 16. entrypoint 调用边界

`ParallelWorkerMain()` 最后：

```text
InitializingParallelWorker = false
EnterParallelMode()
entrypt(seg, toc)
ExitParallelMode()
PopActiveSnapshot()
EndParallelWorkerTransaction()
DetachSession()
pq_putmessage(PqMsg_Terminate, NULL, 0)
```

### 16.1 为什么 entrypoint 在 parallel mode 内

worker 执行 entrypoint 时也必须受 parallel mode 限制。它和 leader 一样不能做破坏并行事务语义的操作。

例如 worker 中如果调用会分配 XID、改 GUC、执行 DDL 的路径，也应被 `IsInParallelMode()` / `IsParallelWorker()` 检查挡住。

### 16.2 为什么退出 parallel mode 后才 pop snapshot

代码注释：

```text
Must exit parallel mode to pop active snapshot.
```

snapmgr.c 对 parallel mode 下 snapshot stack 的变化有额外限制。entrypoint 执行结束后，worker 需要退出 parallel mode，才能按正常规则弹出 active snapshot。

### 16.3 为什么最后发送 Terminate

```text
pq_putmessage(PqMsg_Terminate, NULL, 0)
```

这不是客户端协议中的普通连接终止，而是通过 error queue 告诉 leader：

```text
这个 worker 已经干净完成；
leader 可以 detach 对应 error queue；
finish 阶段不应把它当成 lost connection。
```

### 16.4 worker 写 WAL 的边界

parallel worker 通常不应独立提交事务，但某些路径可能写 WAL。`ParallelWorkerReportLastRecEnd()` 用 `FixedParallelState.last_xlog_end` 把最大 WAL end 位置回报给 leader：

```text
SpinLockAcquire(&fps->mutex)
if fps->last_xlog_end < last_xlog_end:
  fps->last_xlog_end = last_xlog_end
SpinLockRelease()
```

leader 在 `WaitForParallelWorkersToFinish()` 后：

```text
if fps->last_xlog_end > XactLastRecEnd:
  XactLastRecEnd = fps->last_xlog_end
```

这保证 leader 的事务结束逻辑不会漏掉 worker 写过的 WAL 边界。

## 17. ERROR 传播前后对照

同一类 ERROR，发生位置不同，leader 看到的语义不同。

### 17.1 error queue attach 前

```text
dsm_attach() 失败
shm_toc_attach() 失败
PARALLEL_KEY_FIXED lookup 失败
error_queue_space lookup 失败
```

此时 worker 还没有 `pq_redirect_to_shm_mq()`。结果：

```text
worker 可能在 server log 中留下错误
leader 只知道 bgworker stopped before sender attached
leader 报 parallel worker failed to initialize
```

### 17.2 error queue attach 后、entrypoint 前

例如：

```text
BackgroundWorkerInitializeConnectionByOid() 失败
RestoreLibraryState() 失败
RestoreGUCState() 失败
RestoreTransactionSnapshot() 失败
```

此时错误通过 shm_mq 回传，leader 的 `ProcessParallelMessages()` 会 parse ErrorResponse 并 `ThrowErrorData()`。错误上下文会附加 parallel worker 信息。

### 17.3 entrypoint 运行中

executor worker 的 `ParallelQueryMain()` 中如果 `ExecutorRun()` ERROR：

```text
worker ereport(ERROR)
  -> pqmq sends ErrorResponse
  -> PROCSIG_PARALLEL_MESSAGE
  -> leader CHECK_FOR_INTERRUPTS()
  -> ProcessParallelMessages()
  -> ThrowErrorData()
```

leader 需要在 tuple routing 和 finish wait 中频繁 `CHECK_FOR_INTERRUPTS()`，否则错误传播会延迟。

### 17.4 worker 非正常退出但没有 ErrorResponse

`ParallelWorkerShutdown()` 会：

```text
SendProcSignal(leader, PROCSIG_PARALLEL_MESSAGE, leader_proc_number)
dsm_detach(seg)
```

这至少让 leader 再读一次 error queue。如果队列断开但没有 terminate，leader 可能报 lost connection 或 initialization failure，取决于 attach 阶段。

## 18. worker bootstrap 实验

### 18.1 观察恢复顺序

gdb 断点：

```gdb
break ParallelWorkerMain
break pq_redirect_to_shm_mq
break BecomeLockGroupMember
break BackgroundWorkerInitializeConnectionByOid
break RestoreLibraryState
break StartParallelWorkerTransaction
break RestoreTransactionSnapshot
break RestoreGUCState
break AttachSerializableXact
break ParallelQueryMain
```

在 worker 进程中打印：

```gdb
p ParallelWorkerNumber
p MyFixedParallelState
p ParallelLeaderPid
p ParallelLeaderProcNumber
p InitializingParallelWorker
```

### 18.2 比较 ERROR 前后

源码实验：

```text
在 pq_redirect_to_shm_mq() 前插入 elog(ERROR, ...)
  -> leader 多半只报 failed to initialize

在 pq_redirect_to_shm_mq() 后插入 elog(ERROR, ...)
  -> leader 能显示具体 ERROR
```

这能直观看到 error queue 边界。

### 18.3 检查 snapshot

在 worker 到达 `RestoreTransactionSnapshot()` 后：

```gdb
p asnapshot->xmin
p asnapshot->xmax
p tsnapshot
p MyProc->xmin
```

对 READ COMMITTED 和 REPEATABLE READ 各跑一次，比较 transaction snapshot 是否独立存在。

### 18.4 检查 lock group

在 `BecomeLockGroupMember()` 后：

```gdb
p MyProc->lockGroupLeader
p MyProc->lockGroupMember
p MyFixedParallelState->parallel_leader_pgproc
```

字段名随版本可能变化，核心是确认 worker 已归入 leader 的 lock group。

## 19. 版本差异和扩展入口

`InternalParallelWorkers[]` 当前包含核心入口，例如：

```text
ParallelQueryMain
parallel_vacuum_main
_bt_parallel_build_main
_brin_parallel_build_main
_gin_parallel_build_main
```

扩展也可以通过 library/function name 提供并行入口。需要注意：

```text
entrypoint 接收 dsm_segment *seg, shm_toc *toc
扩展必须定义自己的 TOC keys
扩展不能假设 leader 的 backend-local pointer 可用
扩展必须遵守 parallel mode 限制
扩展必须考虑 worker 少启动或启动失败
```

如果扩展入口需要额外状态，应该在 `InitializeParallelDSM()` 前估算空间，之后写入 TOC，并在 worker entrypoint 中 lookup。

## 20. 源码检查清单：worker bootstrap 新增恢复项

如果要给 parallel worker 增加新的恢复状态，审查时至少要写清：

```text
leader 侧在哪里 estimate / serialize？
DSM key 是通用 key、executor key，还是 caller-specific key？
worker 侧在哪里 lookup / restore？
restore 应该位于哪个顺序点？
失败时是 ERROR、quiet return，还是 no-worker fallback？
cleanup 是否需要 worker exit hook 或 DSM detach hook？
```

### 20.1 顺序点选择

常见顺序点：

| 顺序点 | 适合恢复什么 |
| --- | --- |
| error queue attach 前 | 几乎不要放，失败无法回传。 |
| error queue attach 后、lock group 前 | 很少使用，拿锁前必须加入 lock group。 |
| lock group 后、database connection 前 | timestamp、入口函数、身份基础状态。 |
| database connection 后、library 前 | 需要 database 但不依赖 extension GUC 的状态。 |
| library 后、transaction 前 | 依赖已加载库但不依赖 worker transaction 的状态。 |
| transaction / snapshot 后、GUC 前 | 依赖 catalog / visibility 的状态。 |
| GUC 后、安全上下文后 | 最终执行上下文。 |
| entrypoint 内 | caller-specific 或 executor-specific 状态。 |

### 20.2 不要恢复的状态

不要试图恢复：

```text
leader 的 ResourceOwner 指针
leader 的 MemoryContext 指针
leader 的 file descriptor
leader 的 local buffer pin
leader 的 catcache entry 指针
leader 的 function pointer
```

应该恢复的是可重建身份：

```text
Oid、Xid、SubTransactionId、Snapshot bytes、GUC bytes、DSM handle、dsa_pointer、string name。
```

### 20.3 扩展入口约束

扩展若使用 parallel worker entrypoint，应遵守：

```text
entrypoint 只能依赖 seg/toc 和 worker 已恢复状态；
不能假设 leader-local 全局变量已经同步；
不能在 worker 中执行 parallel-unsafe 操作；
必须处理 leader 终止或 queue detached；
必须让错误通过正常 ereport 路径回传。
```

## 21. 故障模式速查表

| 现象 | 可能阶段 | 优先检查 |
| --- | --- | --- |
| leader 只看到初始化失败 | error queue attach 前 | `dsm_attach()`、TOC magic、`PARALLEL_KEY_FIXED`、server log。 |
| leader 看到 worker 具体 ERROR | error queue attach 后 | `ProcessParallelMessage()`、worker ErrorResponse。 |
| worker 无法访问 catalog | database / snapshot restore | `BackgroundWorkerInitializeConnectionByOid()`、`RestoreTransactionSnapshot()`。 |
| GUC 恢复失败 | library / GUC 顺序 | custom GUC 所属库是否已加载。 |
| 权限语义不一致 | role/security context | user/role restore 顺序和 `SetUserIdAndSecContext()`。 |
| serializable 行为异常 | SSI attach | `ShareSerializableXact()` / `AttachSerializableXact()`。 |
| worker 能启动但 tuple 类型不兼容 | session DSM | `GetSessionDsmHandle()` / `AttachSession()`。 |

## 22. 本节小结

`ParallelWorkerMain()` 是把一个普通 dynamic background worker 变成“leader 事务的并行执行分支”的 bootstrap。它先建立错误回传通道，再按严格顺序恢复身份、数据库、事务、snapshot、GUC、session 和 serializable 状态，最后才调用真正的并行入口。

可迁移规律：

```text
跨进程 worker bootstrap 的顺序本身就是正确性协议；
诊断通道、身份、锁、事务、可见性和配置恢复不能随意重排。
```
