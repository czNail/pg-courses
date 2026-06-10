# PostgreSQL parallel message、ERROR 传播与 worker finish 协议

## 课程定位

前置知识：已经理解 `ParallelContext` 生命周期、parallel DSM 初始化、worker 启动和 `ParallelWorkerMain()` 如何 attach error queue。

本节唯一主问题：

```text
worker 中的 ereport()、NOTICE、NOTIFY、progress 和 terminate 消息
如何经 pq_redirect_to_shm_mq()、HandleParallelMessageInterrupt()、
ProcessParallelMessages() 与 WaitForParallelWorkersToFinish() 回到 leader，
并在 DestroyParallelContext() 中完成中断、等待和队列清理？
```

核心矛盾：parallel worker 是独立 backend，错误和通知在 worker 进程内产生；但用户连接只连着 leader，事务结果也由 leader 决定。PostgreSQL 必须把 worker 的协议消息可靠转发给 leader，同时避免 leader 在 worker 仍可能报错时过早提交、过早释放 DSM 或无限等待。

学完后应能判断：一个 worker ERROR 为什么会在 leader 中重新抛出，为什么 worker attach error queue 前后的失败表现不同，为什么 `WaitForParallelWorkersToFinish()` 即使在看似完成后仍必须调用，以及 `DestroyParallelContext()` 为什么不能代替 clean finish。

本课基于本地 `~/postgres-lab` 源码，分支 `master`，提交 `bd4bd30ce6a7`。

## 1. 本节在总主线中的位置

前几节建立了并行启动链：

```text
CreateParallelContext()
  -> InitializeParallelDSM()
  -> LaunchParallelWorkers()
  -> ParallelWorkerMain()
     -> attach error queue
     -> pq_redirect_to_shm_mq()
     -> entrypoint(seg, toc)
```

本节讲 worker 运行和退出期间的反向通道：

```text
worker protocol message
  -> shm_mq error queue
  -> procsignal leader
  -> CHECK_FOR_INTERRUPTS()
  -> ProcessParallelMessages()
  -> leader rethrow / forward / mark finished
```

下一节会进入 executor-specific tuple queue 和 plan/param DSM；本节只讲控制消息和错误传播，不讲普通 tuple data。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ParallelWorkerMain() 把 worker 的 libpq 输出重定向到每 worker 一个 shm_mq；
worker 每写一条协议消息，就给 leader 发送 PROCSIG_PARALLEL_MESSAGE；
leader 的 signal handler 只设置 ParallelMessagePending；
下一次 CHECK_FOR_INTERRUPTS() 调用 ProcessParallelMessages()；
ProcessParallelMessages() 遍历 pcxt_list 中所有 worker error queues，解析协议消息并在 leader 中重新执行语义；
WaitForParallelWorkersToFinish() 循环等待所有 worker 发送 terminate 或退出，确保最后的 ERROR 没被遗漏。
```

这里的 tension 是：

```text
错误要像发生在 leader 中一样被用户看到
  vs
worker 是独立进程，消息通道可能满、断开、早期失败或在 cleanup 中关闭
```

PostgreSQL 的选择不是共享 `ErrorData *`，而是复用 frontend/backend protocol message 格式，把 ErrorResponse / NoticeResponse / NotificationResponse 等通过 shm_mq 发给 leader。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/libpq/pqmq.c` | `pq_redirect_to_shm_mq()`、`mq_putmessage()` 如何把协议消息写到 shm_mq 并 signal leader。 |
| 2 | `src/backend/access/transam/parallel.c` | `HandleParallelMessageInterrupt()`、`ProcessParallelMessages()`、`ProcessParallelMessage()`、`WaitForParallelWorkersToFinish()`。 |
| 3 | `src/include/access/parallel.h` | `ParallelMessagePending`、`ParallelWorkerInfo.error_mqh`、parallel message API。 |
| 4 | `src/backend/storage/ipc/procsignal.c` | `PROCSIG_PARALLEL_MESSAGE` 如何进入 interrupt 路径。 |
| 5 | `src/backend/storage/ipc/shm_mq.c` | `shm_mq_sendv()`、`shm_mq_receive()`、detach / would-block 语义。 |
| 6 | `src/backend/executor/execParallel.c` | `ExecParallelFinish()` 何时等待 worker 并汇总 usage。 |
| 7 | `src/backend/executor/tqueue.c` | 对照 tuple queue：tuple data 不走 error queue。 |

推荐阅读顺序：

```text
先读 pqmq.c，理解 worker 如何把普通 ereport 输出改道
  -> 再读 parallel.c 中 leader 如何处理消息
  -> 最后读 ExecParallelFinish() 看 executor 为什么必须等待
```

不要把 error queue 和 tuple queue 混在一起。它们都基于 `shm_mq`，但语义完全不同。

## 4. 关键数据结构与状态

### 4.1 worker 侧 `pqmq.c` 状态

`pqmq.c` 中有几个关键静态状态：

| 状态 | 语义 |
| --- | --- |
| `PqCommMethods = &PqCommMqMethods` | worker 的 libpq 输出方法改为写 shm_mq。 |
| `pq_mq_handle` | 当前 worker 的 error queue handle。 |
| `pq_mq_parallel_leader_pid` | 每次写消息后通知哪个 leader pid。 |
| `pq_mq_parallel_leader_proc_number` | leader 的 proc number，供 `SendProcSignal()` 定位。 |
| `pq_mq_busy` | 防止递归发送消息时破坏队列协议。 |

`pq_redirect_to_shm_mq(seg, mqh)` 还会注册 `on_dsm_detach()` cleanup：

```text
DSM detach
  -> pq_cleanup_redirect_to_shm_mq()
  -> pq_mq_handle = NULL
  -> whereToSendOutput = DestNone
```

这避免 DSM 已消失后仍尝试向 queue 写消息。

### 4.2 leader 侧 `ParallelWorkerInfo`

leader 对每个 worker 追踪：

| 字段 | 语义 |
| --- | --- |
| `bgwhandle` | 查询 worker pid / status，等待 shutdown，强制 terminate。 |
| `error_mqh` | leader 侧读取 error queue 的 handle；收到 terminate 后会 detach 并置 NULL。 |

`error_mqh == NULL` 在 leader 侧有明确含义：

```text
这个 worker 的控制消息通道已经结束；
finish / attach wait 不应再尝试读它。
```

### 4.3 global flags

```text
volatile sig_atomic_t ParallelMessagePending
```

signal handler 中只能做 async-signal-safe 的最小动作：

```text
InterruptPending = true
ParallelMessagePending = true
```

真正的解析、内存分配、错误抛出都推迟到普通执行上下文中的 `ProcessParallelMessages()`。

## 5. 主流程源码 walkthrough：worker 发送一条 ERROR

worker 已经在 `ParallelWorkerMain()` 中完成：

```text
shm_mq_set_sender(mq, MyProc)
mqh = shm_mq_attach(mq, seg, NULL)
pq_redirect_to_shm_mq(seg, mqh)
pq_set_parallel_leader(leader_pid, leader_proc_number)
```

现在 entrypoint 中发生：

```c
ereport(ERROR, ...);
```

路径变成：

```text
ereport()
  -> 生成 ErrorResponse protocol message
  -> pq_putmessage()
     -> mq_putmessage()
        -> shm_mq_sendv(pq_mq_handle, [msgtype, payload], nowait? false, force_flush true)
        -> SendProcSignal(leader_pid, PROCSIG_PARALLEL_MESSAGE, leader_proc_number)
```

几个细节：

1. `mq_putmessage()` 不写长度字，`shm_mq_receive()` 已经提供消息长度。
2. `force_flush = true`，保证共享队列状态在 signal leader 前可见。
3. 发送 signal 后，leader 不会在 signal handler 中解析消息，只设置 pending flag。

leader 侧：

```text
procsignal_sigusr1_handler()
  -> HandleParallelMessageInterrupt()
     -> InterruptPending = true
     -> ParallelMessagePending = true

leader 下一个 CHECK_FOR_INTERRUPTS()
  -> ProcessInterrupts()
  -> ProcessParallelMessages()
```

`ProcessParallelMessages()`：

```text
HOLD_INTERRUPTS()
create/reset hpm_context
ParallelMessagePending = false

for each ParallelContext in pcxt_list:
  for each launched worker:
    while error_mqh != NULL:
      res = shm_mq_receive(error_mqh, &nbytes, &data, true)
      if WOULD_BLOCK:
        break
      if SUCCESS:
        msg = copy bytes into StringInfo
        ProcessParallelMessage(pcxt, i, &msg)
      else:
        ERROR "lost connection to parallel worker"

reset context
RESUME_INTERRUPTS()
```

`ProcessParallelMessage()` 遇到 `PqMsg_ErrorResponse`：

```text
pq_parse_errornotice(msg, &edata)
edata.elevel = Min(edata.elevel, ERROR)
append context "parallel worker"
save current error_context_stack
error_context_stack = pcxt->error_context_stack
ThrowErrorData(&edata)
```

这就是 worker ERROR 在 leader 中重新抛出的完整路径。

## 6. 主流程源码 walkthrough：NOTICE、NOTIFY、progress 与 terminate

`ProcessParallelMessage()` 根据第一个 byte 判断消息类型。

### 6.1 ErrorResponse / NoticeResponse

```text
PqMsg_ErrorResponse
PqMsg_NoticeResponse
  -> pq_parse_errornotice()
  -> edata.elevel = Min(edata.elevel, ERROR)
  -> add "parallel worker" context
  -> use pcxt->error_context_stack
  -> ThrowErrorData()
```

`Min(edata.elevel, ERROR)` 的含义：

```text
worker 的 FATAL / PANIC 不应直接让 leader 自杀；
对 leader 来说，worker 死亡通常转换成 ERROR。
```

这不代表原始 worker 不严重，而是 leader 的用户事务应以 ERROR 形式看到并回滚。

### 6.2 NotificationResponse

```text
PqMsg_NotificationResponse
  -> parse pid / channel / payload
  -> NotifyMyFrontEnd(channel, payload, pid)
```

worker 产生的 NOTIFY 需要通过 leader 的客户端连接转发给前端。用户并没有直接连接 worker。

### 6.3 Progress

```text
PqMsg_Progress
  -> index = pq_getmsgint()
  -> incr = pq_getmsgint64()
  -> pgstat_progress_incr_param(index, incr)
```

当前只支持增量 progress reporting。worker 不直接修改 leader 的 progress struct，而是把增量作为协议消息交给 leader 应用。

### 6.4 Terminate

worker 正常完成后：

```text
pq_putmessage(PqMsg_Terminate, NULL, 0)
```

leader 处理：

```text
shm_mq_detach(pcxt->worker[i].error_mqh)
pcxt->worker[i].error_mqh = NULL
```

这一步把 worker 标记为控制消息通道干净结束。后续 attach / finish wait 看到 `error_mqh == NULL`，就不会继续等待这个 queue。

## 7. 生命周期 / ownership / cleanup

### worker 侧

worker 侧 error queue handle 由 `pq_redirect_to_shm_mq()` 接管：

```text
pq_mq_handle = mqh
on_dsm_detach(seg, pq_cleanup_redirect_to_shm_mq)
```

worker 退出时，如果 DSM detach，`pq_mq_handle` 会被清掉，避免 shutdown 后期 DEBUG/NOTICE 继续写已经无效的 queue。

### leader 侧

leader 的 `ParallelContext.worker[i].error_mqh` 在三个地方可能变为 NULL：

1. `ProcessParallelMessage()` 收到 `PqMsg_Terminate`。
2. `DestroyParallelContext()` 强制 detach。
3. `LaunchParallelWorkers()` 注册失败时 detach unused queue。

这三个 NULL 的语义不同：

| 位置 | 语义 |
| --- | --- |
| terminate message | worker 干净完成。 |
| destroy | leader 正在强制清理，不再接收消息。 |
| registration failure | 这个 slot 没有 worker，不参与后续协议。 |

阅读代码时必须结合阶段解释 `error_mqh == NULL`。

## 8. 正确性机制层次

| 层次 | 机制 | 正确性目的 |
| --- | --- | --- |
| 传输 | per-worker `shm_mq` | worker 协议消息不与 tuple queue 混用。 |
| 通知 | `PROCSIG_PARALLEL_MESSAGE` | worker 写消息后唤醒 leader。 |
| interrupt | `ParallelMessagePending` | signal handler 只设 flag，避免在 handler 中做复杂工作。 |
| 内存 | `hpm_context` | 解析消息时不污染任意当前 memory context。 |
| 错误语义 | `ThrowErrorData()` in leader | 用户从 leader 连接看到 worker 错误。 |
| lifecycle | `WaitForParallelWorkersToFinish()` | 不遗漏 worker 最后消息。 |
| cleanup | `DestroyParallelContext()` | 异常路径终止 worker、detach queue / DSM。 |

核心不变量：

```text
只要 worker 可能还会发送 ERROR，leader 就不能完成并行操作的事务边界。
```

## 9. `WaitForParallelWorkersToFinish()` 深入

这个函数经常被误解成“等 worker 退出”。更准确地说，它等待 worker 的最后控制消息被 leader 接收，并确保启动失败能转成 ERROR。

主循环：

```text
for (;;):
  CHECK_FOR_INTERRUPTS()

  for each worker:
    if error_mqh == NULL:
      nfinished++
    else if known_attached_workers[i]:
      anyone_alive = true
      break

  if !anyone_alive:
    if nfinished == nworkers_launched:
      break

    inspect bgworker stopped-before-attach
      -> ERROR if sender NULL

  WaitLatch(WAIT_EVENT_PARALLEL_FINISH)
```

### 9.1 为什么先 `CHECK_FOR_INTERRUPTS()`

worker terminate / error 会先让 `ParallelMessagePending = true`。只有处理 interrupt，`ProcessParallelMessages()` 才会把：

```text
PqMsg_Terminate
  -> error_mqh = NULL
```

如果不先处理消息，finish loop 可能永远认为 queue 还活着。

### 9.2 为什么只要有 known attached worker 就 wait

`known_attached_workers[i]` 表示 worker 已经 attach 过 error queue。这样的 worker 退出时，leader 应能收到消息或 latch notification。它是“可以等待”的对象。

未 known attached 的 worker 则需要额外检查 bgworker status，因为它可能根本没有启动成功。

### 9.3 为什么 finish 还检查 initialization failure

有些调用方可能不调用 `WaitForParallelWorkersToAttach()`。因此 finish 阶段必须兜底：

```text
worker stopped
error queue sender NULL
  -> parallel worker failed to initialize
```

否则 leader 可能把“worker 根本没初始化”误认为“worker 没有产出 tuple”。

### 9.4 `last_xlog_end`

finish 结束时：

```text
fps = shm_toc_lookup(PARALLEL_KEY_FIXED)
if fps->last_xlog_end > XactLastRecEnd:
  XactLastRecEnd = fps->last_xlog_end
```

worker 可能写 WAL，但事务提交由 leader 负责。leader 必须把 worker 的 WAL end 纳入自己的事务 WAL 边界。

## 10. `DestroyParallelContext()` 与 clean finish 的区别

`WaitForParallelWorkersToFinish()` 是 clean finish：

```text
处理 worker ERROR / NOTICE / terminate
确认 worker 结束
更新 XactLastRecEnd
```

`DestroyParallelContext()` 是资源销毁：

```text
dlist_delete()
TerminateBackgroundWorker()
shm_mq_detach(error_mqh)
dsm_detach()
WaitForParallelWorkersToExit()
free memory
```

如果跳过 finish 直接 destroy：

```text
可能丢掉 worker 最后的 ERROR / NOTICE
可能把仍在运行的 worker 强杀
仍能保证资源不泄漏
但不是正常执行完成语义
```

因此 executor clean path 是：

```text
ExecParallelFinish()
  -> WaitForParallelWorkersToFinish()
  -> accumulate Buffer/WAL usage

ExecParallelCleanup()
  -> retrieve instrumentation / JIT
  -> dsa_detach()
  -> DestroyParallelContext()
```

## 11. 错误路径 / 异常路径 / fallback

### 11.1 worker attach 前失败

表现：

```text
sender NULL
bgworker stopped
leader ERROR "parallel worker failed to initialize"
```

具体原因可能只在 server log 中：

```text
DSM attach failed
TOC magic mismatch
early bootstrap ERROR
```

### 11.2 worker attach 后 ERROR

表现：

```text
leader rethrows worker ErrorResponse
context includes "parallel worker"
transaction aborts in leader
AtEOXact_Parallel(false) destroys remaining contexts
```

### 11.3 queue lost

`shm_mq_receive()` 返回非 success / would-block 时：

```text
ERROR "lost connection to parallel worker"
```

这说明控制消息通道断了，leader 不能再相信 worker 会正常报告结果。

### 11.4 recursive message send

`pqmq.c` 中 `pq_mq_busy` 防止递归发送：

```text
worker 正在向 queue 写消息
  -> 被 interrupt
  -> interrupt 处理又尝试发送消息
```

这种情况下代码选择 detach queue 并返回 EOF，而不是无限嵌套。它牺牲部分消息完整性，保护 worker 不在错误报告中死锁。

### 11.5 leader 不处理 interrupts

如果 leader 长时间不 `CHECK_FOR_INTERRUPTS()`：

```text
ParallelMessagePending 已置位
worker queue 可能满
worker 可能阻塞发送
leader 仍在 CPU loop 或不可中断区
```

这就是为什么并行基础设施强调 leader 代码必须定期处理 interrupts。

## 12. 成本、资源与跨模块传播

每个 worker 至少有一个 `PARALLEL_ERROR_QUEUE_SIZE` 的 error queue。这个 queue 只传控制/协议消息，不传 tuple。tuple 使用 executor 的 `PARALLEL_TUPLE_QUEUE_SIZE`。

消息传播成本包括：

```text
worker 构造 libpq protocol payload
shm_mq_sendv() 复制到 DSM
SendProcSignal()
leader signal handler 设置 flag
leader interrupt path 遍历所有 active pcxt / workers
解析 ErrorResponse / NoticeResponse
可能 ThrowErrorData()
```

当 active parallel contexts 或 workers 多时，`ProcessParallelMessages()` 会扫描多个 queue。它用 nonblocking receive，一次尽可能 drain，避免每条消息都单独唤醒/解析。

跨模块关系：

```text
ereport / libpq protocol
  -> pqmq.c
  -> shm_mq.c
  -> procsignal.c
  -> parallel.c
  -> xact / executor cleanup
```

## 13. 观测与诊断入口

| 入口 | 能看到什么 |
| --- | --- |
| `pg_stat_activity.wait_event = ParallelFinish` | leader 正在等 worker 完成并处理最后消息。 |
| `pg_stat_activity.wait_event = ExecuteGather` | leader 可能在等 tuple queue 数据，同时也会处理 worker messages。 |
| server log | attach 前失败、worker 原始 ERROR、重复日志。 |
| gdb `ParallelMessagePending` | leader 是否有未处理 parallel message interrupt。 |
| gdb `pcxt->worker[i].error_mqh` | worker 控制通道是否仍活着。 |
| gdb `pq_mq_handle` in worker | worker 是否仍能发送协议消息。 |

调试断点：

```gdb
break pq_redirect_to_shm_mq
break mq_putmessage
break HandleParallelMessageInterrupt
break ProcessParallelMessages
break ProcessParallelMessage
break WaitForParallelWorkersToFinish
```

## 14. 常见误区

1. 误以为 worker ERROR 直接跨进程 longjmp 到 leader。实际是协议消息重放。
2. 误以为 signal handler 会读取 shm_mq。它只设置 pending flag。
3. 误以为 `DestroyParallelContext()` 会正常接收所有 worker 错误。clean path 必须先 finish。
4. 误以为 error queue 也传 tuple。tuple queue 是 executor 单独创建的 shm_mq。
5. 误以为 worker FATAL 一定让 leader FATAL。leader 通常把 worker 错误限制到 ERROR。

## 15. 课堂实验

### 15.1 worker ERROR 回传

可以在 `ParallelQueryMain()` 中临时加入：

```c
ereport(ERROR, (errmsg("parallel worker injected error")));
```

执行并行查询，观察 leader session 中是否显示：

```text
ERROR: parallel worker injected error
CONTEXT: parallel worker
```

### 15.2 NOTICE 回传

在 worker entrypoint 中临时加入：

```c
ereport(NOTICE, (errmsg("worker %d says hello", ParallelWorkerNumber)));
```

预期 leader 客户端看到 NOTICE，但实际条数取决于 worker 是否启动和执行到该路径。

### 15.3 Terminate 处理

gdb：

```gdb
break ProcessParallelMessage
condition <bpnum> msgtype == 'X'
```

实际 `PqMsg_Terminate` 的字符常量可在源码中确认。观察执行后：

```gdb
p pcxt->worker[i].error_mqh
```

应被置 NULL。

### 15.4 leader interrupt 延迟

在 leader 中给 `ProcessParallelMessages()` 设断点。执行并行查询，当 worker ERROR 后，观察 leader 并不会立即在 signal handler 中进入该函数，而是在下一个 `CHECK_FOR_INTERRUPTS()` 进入。

## 16. 讨论题

1. 为什么 worker 的错误用 libpq protocol message 表达，而不是把 `ErrorData` 结构体直接放进 shared memory？
2. 为什么 `ProcessParallelMessages()` 要使用单独的 `hpm_context`？
3. 如果 leader 在收到 worker ERROR 前已经释放 DSM，会出现什么诊断问题？
4. 为什么 worker 正常结束要发 `PqMsg_Terminate`，而不是只让进程退出？
5. `WaitForParallelWorkersToFinish()` 和 `WaitForParallelWorkersToExit()` 的语义差别是什么？

## 17. 源码索引：消息类型到处理动作

下面把本节涉及的消息路径压缩成一个索引，便于调试时快速定位。

| worker 动作 | worker 侧函数 | queue 内容 | leader 侧处理 |
| --- | --- | --- | --- |
| `ereport(ERROR)` | `mq_putmessage(PqMsg_ErrorResponse, ...)` | ErrorResponse payload | `pq_parse_errornotice()` + `ThrowErrorData()`。 |
| `ereport(NOTICE)` | `mq_putmessage(PqMsg_NoticeResponse, ...)` | NoticeResponse payload | `ThrowErrorData()` 以 NOTICE 等级输出。 |
| `NOTIFY` | protocol NotificationResponse | pid/channel/payload | `NotifyMyFrontEnd()`。 |
| progress increment | `PqMsg_Progress` | index + int64 increment | `pgstat_progress_incr_param()`。 |
| worker clean exit | `pq_putmessage(PqMsg_Terminate, NULL, 0)` | terminate byte | detach error queue，`error_mqh = NULL`。 |
| queue lost | shm_mq detach / receive failure | none | `lost connection to parallel worker`。 |

### 17.1 ErrorResponse 为什么要重新 parse

worker 不把 `ErrorData` 结构体直接放入 DSM，因为：

```text
ErrorData 中有指针、内存上下文所有权和本地字符串生命周期；
不同进程地址空间不能共享这些指针；
frontend/backend protocol 已经有稳定的错误消息编码。
```

leader parse protocol message 后重建 `ErrorData`，再用自己的 error machinery 抛出。

### 17.2 为什么 context 使用创建 ParallelContext 时的 stack

`ProcessParallelMessage()` 临时设置：

```text
error_context_stack = pcxt->error_context_stack
```

原因是 worker ERROR 可能在 leader 的任意 `CHECK_FOR_INTERRUPTS()` 点被处理。当前 leader call stack 未必和启动并行操作的上下文相关。例如 leader 正在 `WaitLatch()` 或上层 executor 节点中处理中断。使用 `pcxt->error_context_stack` 能把错误归因到创建并行上下文时的执行语境。

### 17.3 为什么 `DEBUG_PARALLEL_REGRESS` 特殊

源码中对 `debug_parallel_query == DEBUG_PARALLEL_REGRESS` 跳过追加 `parallel worker` context。原因是回归测试需要稳定输出：

```text
同一个查询可能因环境不同启动或不启动 worker；
如果错误 context 取决于是否使用 parallel worker，测试输出会不稳定。
```

这不是普通用户路径。

## 18. 源码检查清单：新增 worker 消息类型

如果要新增一种从 worker 到 leader 的控制消息，不要直接在 shm_mq 中塞私有格式后就结束。至少检查：

```text
消息是否应该复用 libpq protocol message？
leader 是否能在 ProcessParallelMessage() 中解析？
解析过程是否可能 ERROR？
是否需要 hpm_context 防止内存泄漏？
是否需要标记 known_attached_workers？
消息是否需要在 finish 前全部 drain？
worker exit 时未发送该消息是否安全？
```

### 18.1 消息是否可以丢

不同消息可靠性要求不同：

| 消息 | 可否丢失 | 原因 |
| --- | --- | --- |
| ErrorResponse | 不可丢 | 事务语义必须回滚并报告。 |
| Terminate | 不应丢 | leader 用它判断 clean finish。 |
| NoticeResponse | 不应随意丢 | 用户可见诊断。 |
| Progress | 可以是增量但不能乱序解释 | 统计语义可能允许近似，但不能破坏计数。 |
| DEBUG late shutdown message | 可能忽略 | DSM 已 detach 后发送无意义。 |

### 18.2 消息是否会递归发送

如果解析或处理消息时又触发 ERROR，可能再次尝试发送/处理消息。worker 侧 `pq_mq_busy` 已处理发送递归；leader 侧 `ProcessParallelMessages()` 用 `HOLD_INTERRUPTS()` 避免递归进入。新增消息处理逻辑也不能在内部做长时间可中断等待。

### 18.3 消息是否与 tuple queue 混淆

不要把控制消息放到 tuple queue，也不要把 data tuple 放到 error queue：

```text
error queue:
  小型协议消息，必须及时唤醒 leader

tuple queue:
  大量 MinimalTuple，受 executor pull 和背压控制
```

混用会让错误传播受 tuple 消费速度影响，或者让 tuple routing 误读协议消息。

## 19. 故障模式速查表

| 现象 | 阶段 | 可能原因 | 排查点 |
| --- | --- | --- | --- |
| `lost connection to parallel worker` | receive | queue detached 但没有 clean terminate | worker crash、DSM detach、异常退出。 |
| leader 没及时看到 worker ERROR | interrupt | leader 长时间不 `CHECK_FOR_INTERRUPTS()` | CPU loop、不可中断区、扩展阻塞。 |
| worker 卡在发送 ERROR | queue | error queue 满，leader 未 drain | leader 未处理中断或死等其它资源。 |
| NOTICE 出现两次日志 | logging | worker 原始 log + leader rethrow | `README.parallel` 提到的已知权衡。 |
| worker FATAL 变 leader ERROR | error level clamp | `edata.elevel = Min(..., ERROR)` | 正常设计，不是级别丢失 bug。 |
| finish 阶段才报错 | late message | worker 在 shutdown 或 drain 时 ERROR | `WaitForParallelWorkersToFinish()` 正在补漏。 |

## 20. 实战调试脚本

### 20.1 leader 侧 gdb 命令

```gdb
break HandleParallelMessageInterrupt
break ProcessParallelMessages
break ProcessParallelMessage
break WaitForParallelWorkersToFinish

commands ProcessParallelMessages
  silent
  printf "ParallelMessagePending=%d\\n", ParallelMessagePending
  continue
end
```

### 20.2 worker 侧 gdb 命令

```gdb
break pq_redirect_to_shm_mq
break mq_putmessage
commands mq_putmessage
  silent
  printf "msgtype=%c len=%zu worker=%d\\n", msgtype, len, ParallelWorkerNumber
  continue
end
```

### 20.3 SQL 触发并行消息

```sql
SET max_parallel_workers_per_gather = 2;
SET min_parallel_table_scan_size = 0;
SET parallel_setup_cost = 0;
SET parallel_tuple_cost = 0;

EXPLAIN (ANALYZE)
SELECT count(*) FROM t_parallel_ctx;
```

配合源码临时 `ereport(NOTICE, ...)` 或 `ereport(ERROR, ...)`，观察消息路径。

## 21. 本节小结

parallel worker 的错误传播不是异常跨进程跳转，而是协议消息转发：worker 把 ErrorResponse / NoticeResponse / Notify / progress / terminate 写入 error queue，signal leader；leader 在 interrupt-safe 时机读取、解析并在自己的上下文中执行对应语义。

`WaitForParallelWorkersToFinish()` 是 clean finish 的关键。它确保 worker 最后一条错误或 terminate 消息没有被遗漏，并把 worker WAL end 反馈给 leader。`DestroyParallelContext()` 则是资源清理和异常兜底，不应替代正常 finish。

可迁移规律：

```text
跨进程错误传播要把“传输协议”和“本地错误语义”分开；
worker 负责可靠送达结构化消息，leader 负责在自己的事务和诊断上下文中重新解释它。
```
