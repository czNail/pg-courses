# PostgreSQL shm_mq attach、detach 与 worker 失败传播

## 课程定位

前置知识：已经理解 DSM / `shm_toc` 如何创建并发现共享对象，也理解上一节 `shm_mq` 用 single-reader / single-writer ring buffer 完成消息搬运和背压。

本节唯一主问题：

```text
发送方或接收方可能尚未启动、提前退出或 ERROR 时，
shm_mq_wait_for_attach()、BackgroundWorkerHandle、
SHM_MQ_WOULD_BLOCK / SHM_MQ_DETACHED 和 on-detach callback
如何把“等不到对端”变成可诊断的状态？
```

核心矛盾：`shm_mq` 是两个进程之间的通道，但两个端点并不是同时出生、同时 attach、同时退出。leader 可能先创建 queue，再注册 worker；worker 可能还没 fork 成功就失败；worker 可能 attach 后 ERROR；leader 可能因为事务 abort detach DSM；任一端都可能正在 ring full / ring empty / attach wait 中睡眠。系统必须避免“对端不会再来了，但我还在等”的永久阻塞，同时又不能把正常的尚未启动、暂时无数据、干净退出误判为错误。

学完后应能判断：

```text
为什么 shm_mq_attach() 只是建立本地 handle，不等于对端已经 attach；
为什么 mqh_handle 可以让一端在对端尚未 attach 前就开始 send / receive；
为什么没有 BackgroundWorkerHandle 时，等待未 attach 对端可能永久阻塞；
为什么 SHM_MQ_WOULD_BLOCK 表示“暂时不能推进”，SHM_MQ_DETACHED 表示“通道不会再推进”；
为什么 shm_mq_detach() 要先 flush pending writes，再设置 mq_detached；
为什么 on_dsm_detach callback 只做共享状态 detach，不碰本地 shm_mq_handle；
为什么 parallel.c 还要在 shm_mq 之外检查 worker attach 和 shutdown。
```

本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `2c4bd2bf5700`。

## 1. 本节在总主线中的位置

上一节讲的是 `shm_mq` 的数据面：

```text
ring buffer
  -> mq_bytes_written / mq_bytes_read
  -> memory barrier
  -> latch wait
  -> message boundary
```

这些机制解决的是：

```text
两端都在，如何搬消息？
```

本节讲控制面：

```text
对端还没来、不会来了、已经走了，当前进程如何知道？
```

典型 parallel query 场景是：

```text
leader:
  创建 DSM
  -> 创建 worker error queue / tuple queue
  -> 把自己设为 receiver
  -> attach 本地 shm_mq_handle
  -> 注册 background worker
  -> 之后才知道 worker 是否 fork、attach、发送消息或退出

worker:
  attach DSM
  -> lookup 自己的 queue
  -> 把自己设为 sender
  -> attach 本地 shm_mq_handle
  -> 用 pqmq / tqueue 发送消息
  -> 退出时 detach 或随 DSM detach 传播断开
```

这一节的重点不是 ring 中有哪些字节，而是 `mq_sender` / `mq_receiver` / `mq_detached` / `BackgroundWorkerHandle` / latch wait 如何把端点生命周期串起来。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
shm_mq 的共享对象用 mq_sender / mq_receiver 表示两端是否 attach，
用 mq_detached 表示通道已经不会继续推进；
本地 shm_mq_handle 可选保存 BackgroundWorkerHandle，
让等待方在对端未 attach 时能判断 worker 是否已经死亡；
detach 和 DSM detach callback 负责设置 mq_detached 并唤醒对端。
```

本节 tension 是：

```text
允许异步启动和异步退出
  vs
不能让 send / receive / attach wait 无限睡眠，也不能误杀正常未就绪状态
```

如果只用一个简单布尔值：

```text
ready = true / false
```

无法区分：

```text
worker 还没被 postmaster 启动；
worker 已经启动但还没 attach queue；
worker attach 后暂时没写数据；
worker attach 前就失败；
worker attach 后发送了最后一条消息并退出；
leader 自己 abort，把 DSM detach 了。
```

PostgreSQL 把这些状态拆到三个层次：

```text
endpoint pointer:
  mq_sender / mq_receiver 是否非 NULL

worker lifecycle:
  BackgroundWorkerHandle 能否证明 worker 已经不可能 attach

channel lifecycle:
  mq_detached 是否已经发布给对端
```

这三个层次组合起来，才是 `shm_mq` attach / detach / failure 传播的语义。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/storage/shm_mq.h` | 公开 `shm_mq_result`、`shm_mq_attach()`、`shm_mq_wait_for_attach()`、`shm_mq_detach()`。 |
| 2 | `src/backend/storage/ipc/shm_mq.c` | endpoint 设置、attach handle、wait internal、counterparty gone、detach internal、DSM detach callback。 |
| 3 | `src/backend/access/transam/parallel.c` | leader 如何创建 error queue、注册 worker、等待 attach、处理 worker 消息和初始化失败。 |
| 4 | `src/backend/executor/execParallel.c` | tuple queue 如何延迟绑定 worker handle，让 tuple reader 能识别未启动 worker。 |
| 5 | `src/backend/libpq/pqmq.c` | worker 如何把协议消息写到 queue，重入发送时如何通过 detach 断开。 |
| 6 | `src/backend/executor/tqueue.c` | tuple send / receive 如何把 `SHM_MQ_DETACHED` 转成执行器层面的结束或失败。 |
| 7 | `src/test/modules/test_shm_mq/` | 专用测试模块如何用 handle、worker ready 和 queue detach 练习这些边界。 |

推荐阅读顺序：

```text
先读 shm_mq.c 顶部 struct shm_mq_handle 注释
  -> 读 shm_mq_attach() 和 shm_mq_set_handle()
  -> 读 shm_mq_wait_for_attach() / shm_mq_wait_internal()
  -> 读 shm_mq_counterparty_gone()
  -> 读 shm_mq_detach() / shm_mq_detach_internal() / shm_mq_detach_callback()
  -> 最后读 parallel.c 如何在更高层补充 worker 状态判断
```

不要孤立理解 `SHM_MQ_DETACHED`。它既可能来自对端主动 detach，也可能来自 DSM detach callback，也可能来自本端通过 `BackgroundWorkerHandle` 发现 worker 不会再 attach 后把 `mq_detached` 标成 true。

## 4. 关键数据结构与状态

### 4.1 DSM 内 `shm_mq` 的控制面字段

本节只关心控制面字段：

| 字段 | 位置 | 更新者 | 语义 |
| --- | --- | --- | --- |
| `mq_receiver` | DSM | receiver 设置一次 | 接收端已经 attach 身份。 |
| `mq_sender` | DSM | sender 设置一次 | 发送端已经 attach 身份。 |
| `mq_mutex` | DSM | 双方短持有 | 保护 endpoint pointer 从 NULL 到非 NULL。 |
| `mq_detached` | DSM | 任一端 | 通道已经断开，等待者不应继续无限等待。 |
| `mq_bytes_written` | DSM | sender | detach 前用于让 receiver drain 已发布数据。 |

endpoint pointer 的语义很窄：

```text
mq_sender != NULL:
  sender 曾经把自己的 PGPROC 发布到 queue

mq_receiver != NULL:
  receiver 曾经把自己的 PGPROC 发布到 queue
```

它不保证对端此刻还活着，也不保证未来还会写更多数据。它只回答：

```text
对端是否完成过 queue attach 身份发布？
```

`mq_detached` 则回答另一个问题：

```text
是否已经确定这条通道不应再等待更多推进？
```

这两个问题不能混在一起。

### 4.2 backend-local `shm_mq_handle`

`shm_mq_attach()` 返回的是本地 handle：

| 字段 | 语义 |
| --- | --- |
| `mqh_queue` | 指向 DSM 中的 `shm_mq`。 |
| `mqh_segment` | queue 所在 DSM；非 NULL 时注册 DSM detach callback。 |
| `mqh_handle` | 可选 `BackgroundWorkerHandle`，用于判断未 attach 对端是否已经死亡。 |
| `mqh_counterparty_attached` | 本 backend 是否已经确认对端 attach。 |
| `mqh_send_pending` | detach 前必须 flush 的本地未发布写入。 |
| `mqh_context` | handle 和接收 buffer 的本地 MemoryContext。 |

本节最重要的是 `mqh_handle`：

```text
它不是 queue 的 owner；
它也不是对端 PGPROC；
它是 postmaster background worker 生命周期的观测句柄。
```

有了它，等待方可以在对端尚未设置 `mq_sender` / `mq_receiver` 时调用：

```text
GetBackgroundWorkerPid(handle, &pid)
```

并把 worker 已经停止、不会 attach 的情况转成 `SHM_MQ_DETACHED`。

### 4.3 `shm_mq_result`

`shm_mq` 对外只暴露三个结果：

| 结果 | 语义 |
| --- | --- |
| `SHM_MQ_SUCCESS` | 本次 send / receive / wait attach 完成。 |
| `SHM_MQ_WOULD_BLOCK` | 非阻塞模式下暂时不能推进，但通道仍可能继续。 |
| `SHM_MQ_DETACHED` | 通道已经断开，或对端已经确定不会再推进。 |

关键区别：

```text
WOULD_BLOCK:
  等 latch 后可以用同一 handle 继续尝试。

DETACHED:
  不要再把它当作可恢复等待；上层必须结束、报错或切换路径。
```

## 5. 主流程源码 walkthrough

### 5.1 leader 创建 queue，但 worker 尚未 attach

以 parallel error queue 为例，`InitializeParallelDSM()` 中 leader 先创建每个 worker 的 error queue：

```text
mq = shm_mq_create(start, PARALLEL_ERROR_QUEUE_SIZE)
shm_mq_set_receiver(mq, MyProc)
pcxt->worker[i].error_mqh = shm_mq_attach(mq, pcxt->seg, NULL)
```

此时共享状态是：

```text
mq_receiver = leader MyProc
mq_sender = NULL
mq_detached = false
leader 有 error_mqh
worker 还不存在或还没 attach
```

注意：`shm_mq_attach()` 没有等待 sender。它只是给 leader 建立本地 handle，并在 `seg != NULL` 时注册：

```text
on_dsm_detach(seg, shm_mq_detach_callback, mq)
```

这允许 leader 先把所有 queue 放进 DSM，再去启动 worker。

### 5.2 worker 启动后发布 sender

worker 在 `ParallelWorkerMain()` attach DSM、lookup TOC 后找到自己的 error queue：

```text
mq = error_queue_space + worker_number * PARALLEL_ERROR_QUEUE_SIZE
shm_mq_set_sender(mq, MyProc)
mqh = shm_mq_attach(mq, seg, NULL)
pq_redirect_to_shm_mq(seg, mqh)
```

`shm_mq_set_sender()` 的状态变化：

```text
SpinLockAcquire(mq_mutex)
  mq_sender = worker MyProc
  receiver = mq_receiver
SpinLockRelease(mq_mutex)

if receiver != NULL:
  SetLatch(receiver)
```

这不是普通赋值，而是 attach 事件的发布：

```text
等待 sender attach 的 receiver 可以被唤醒；
后续 receiver 看到 mq_sender != NULL 后，可把 counterparty_attached 记到本地 handle。
```

### 5.3 等待对端 attach：`shm_mq_wait_for_attach()`

`shm_mq_wait_for_attach()` 先判断当前进程是哪一端：

```text
if mq_receiver == MyProc:
  victim = &mq_sender
else:
  victim = &mq_receiver
```

然后进入：

```text
shm_mq_wait_internal(mq, victim, mqh_handle)
```

`shm_mq_wait_internal()` 每轮做四件事：

```text
1. 持 mq_mutex 读取 victim pointer 是否非 NULL
2. 如果 mq_detached 为 true，返回 false
3. 如果 victim 已经非 NULL，返回 true
4. 如果有 BackgroundWorkerHandle，检查 worker 是否已经不可能 attach
5. 否则 WaitLatch(... WAIT_EVENT_MESSAGE_QUEUE_INTERNAL)
```

这里的锁只保护 endpoint pointer 的瞬时读取。长等待交给 latch，不持锁睡眠。

如果 `handle == NULL`，源码注释明确指出：

```text
对端如果永远不 attach，可能永久等待；
不过仍然会检查 interrupts。
```

这就是为什么上层有时必须传入 `BackgroundWorkerHandle`，或者像 `parallel.c` 那样在更高层检查 worker 生命周期。

### 5.4 send path 中等待未 attach receiver

sender 不一定要先等 receiver attach 才开始 send。`shm_mq_send_bytes()` 在 ring 还有空间时可以先写数据到 queue；但当 queue full 且还不知道 receiver attach 时，必须处理这个边界：

```text
available == 0 && !mqh_counterparty_attached
```

此时：

```text
nowait = true:
  如果 counterparty_gone(handle):
    返回 SHM_MQ_DETACHED
  如果 mq_receiver 仍为 NULL:
    返回 SHM_MQ_WOULD_BLOCK
  否则记录 counterparty_attached

nowait = false:
  shm_mq_wait_internal(mq, &mq_receiver, handle)
  如果失败:
    mq_detached = true
    返回 SHM_MQ_DETACHED
```

这个设计允许：

```text
worker 还没 attach receiver 时，sender 先写一点数据；
但不能在 queue 满后继续盲等一个可能永远不会来的 receiver。
```

`SHM_MQ_WOULD_BLOCK` 和 `SHM_MQ_DETACHED` 的区别在这里尤其重要：

```text
WOULD_BLOCK:
  对端可能只是还没 attach。

DETACHED:
  通过 mq_detached 或 BackgroundWorkerHandle 已经确定对端不会推进。
```

### 5.5 receive path 中等待未 attach sender

receiver 进入 `shm_mq_receive()` 时，如果还没确认 sender attach，会先处理：

```text
if (!mqh_counterparty_attached)
```

`nowait = true` 分支有一个竞态处理顺序：

```text
先调用 shm_mq_counterparty_gone()
再检查 shm_mq_get_sender(mq) 是否 NULL
```

源码注释解释了原因：

```text
sender 可能快速 attach 后又 detach；
如果先看 sender 是否曾经 attach，再看 detached，可能误判成“从未 attach 且还可能会来”。
```

所以正确顺序是：

```text
先判断对端是否已经明确 gone；
再判断 sender pointer 是否仍为 NULL；
如果 sender NULL 且 not gone:
  返回 WOULD_BLOCK
如果 sender NULL 且 gone:
  返回 DETACHED
```

`nowait = false` 分支则调用 `shm_mq_wait_internal()`。如果等待失败，并且 sender 仍为 NULL：

```text
mq_detached = true
return SHM_MQ_DETACHED
```

这把“等不到 sender attach”转成显式状态，而不是继续空等。

### 5.6 `BackgroundWorkerHandle` 如何证明对端 gone

`shm_mq_counterparty_gone()` 的判断很克制：

```text
if mq_detached:
  return true

if handle != NULL:
  status = GetBackgroundWorkerPid(handle, &pid)
  if status != BGWH_STARTED && status != BGWH_NOT_YET_STARTED:
    mq_detached = true
    return true

return false
```

它只在能确定 worker 已经不可能 attach 时返回 true。

`BGWH_STARTED` 和 `BGWH_NOT_YET_STARTED` 都不是 failure：

```text
STARTED:
  worker 已经活着，但可能还没 set sender / receiver。

NOT_YET_STARTED:
  postmaster 还可能启动它。
```

其它状态则说明：

```text
worker 已经停止，或者 postmaster 相关状态不再允许它成为 queue 对端。
```

此时将 `mq_detached` 标成 true，是为了让其它等待路径也看到通道已经结束。

### 5.7 detach：主动断开并唤醒对端

`shm_mq_detach()` 的顺序非常关键：

```text
1. 如果 mqh_send_pending > 0:
     shm_mq_inc_bytes_written()
     mqh_send_pending = 0

2. shm_mq_detach_internal(mq)

3. cancel_on_dsm_detach(...)

4. 释放本地 buffer 和 handle
```

第一步是为了避免丢最后一段已经写入 ring 但尚未发布的消息。只有发布 `mq_bytes_written` 后，receiver 才有机会 drain。

`shm_mq_detach_internal()` 做共享断开：

```text
SpinLockAcquire(mq_mutex)
  victim = 对端 PGPROC
  mq_detached = true
SpinLockRelease(mq_mutex)

if victim != NULL:
  SetLatch(victim)
```

它不释放 DSM 内 queue，也不清空 endpoint pointer。它只发布：

```text
这条通道已经断开；
正在等待的对端应该醒来重新检查状态。
```

### 5.8 DSM detach callback：为什么只做 shared detach

`shm_mq_attach()` 注册的 callback 是：

```text
on_dsm_detach(seg, shm_mq_detach_callback, mq)
```

callback 实现很短：

```text
shm_mq_detach_callback(seg, arg):
  mq = arg
  shm_mq_detach_internal(mq)
```

它故意不访问 `shm_mq_handle`。源码注释说明了原因：

```text
如果 callback 触发，本地 handle 可能已经被 pfree；
callback 只需要把共享 queue 标成 detached 并唤醒对端。
```

这和第 28 节的 detach callback 边界一致：

```text
callback 是 DSM mapping 生命周期的保护网；
不是本地对象析构函数。
```

本地内存由 `shm_mq_detach()` 或 MemoryContext 生命周期处理；共享断开由 callback 兜底。

### 5.9 receiver 在 sender detach 后仍可 drain

detach 不等于 receiver 立刻读不到数据。`shm_mq_receive_bytes()` 的顺序是：

```text
先判断当前已发布数据是否足够；
如果足够，仍然返回 SUCCESS；
只有不足时才检查 mq_detached。
```

看到 `mq_detached` 后还会做一次防竞态检查：

```text
pg_read_barrier()
if written != atomic_read(mq_bytes_written):
  continue
return SHM_MQ_DETACHED
```

这处理的是：

```text
sender 先推进 mq_bytes_written，
随后设置 mq_detached；
receiver 可能先读到旧 written，再读到 detached。
```

receiver 不能因为看到 detached 就丢掉最后已发布的数据。它必须确认 `mq_bytes_written` 没有新变化。

## 6. 生命周期 / ownership / cleanup

`shm_mq` 的 attach / detach 生命周期跨三种 owner：

| 对象 | owner | cleanup |
| --- | --- | --- |
| DSM 中的 queue bytes | 创建 DSM 的上层模块 | 随 DSM destroy / detach 失效。 |
| 本地 `shm_mq_handle` | attach 时的 MemoryContext | `shm_mq_detach()` 释放，或更外层 context 清理。 |
| queue 断开状态 | `mq_detached` 共享字段 | 任一端 detach、DSM detach callback、worker gone 检测都可设置。 |
| worker 生命周期 | postmaster / bgworker registry | `BackgroundWorkerHandle` 只用于观测和等待，不拥有 worker。 |

parallel context 的典型 cleanup：

```text
DestroyParallelContext()
  -> TerminateBackgroundWorker()
  -> shm_mq_detach(error_mqh)
  -> dsm_detach(seg)
  -> WaitForParallelWorkersToExit()
```

顺序里有两个要点：

```text
先 detach error queue，避免 leader 继续等 worker 消息；
dsm_detach 会触发 queue 的 on_dsm_detach callback，兜底传播断开。
```

`pqmq.c` 还有一个特殊场景：如果 worker 正在通过 queue 发送协议消息，过程中被 interrupt 触发重入发送，代码会主动：

```text
shm_mq_detach(pq_mq_handle)
pfree(pq_mq_handle)
pq_mq_handle = NULL
```

这是因为原发送上下文无法恢复，与其无限推迟 interrupt 响应，不如断开 queue，让 leader 看到连接失败。

## 7. 正确性机制层次

| 层次 | 机制 | 解决的问题 |
| --- | --- | --- |
| endpoint 发布 | `shm_mq_set_sender()` / `shm_mq_set_receiver()` + `mq_mutex` | 对端是否曾经 attach queue。 |
| attach 等待 | `shm_mq_wait_internal()` + latch | 不持锁等待 endpoint pointer 出现。 |
| worker 生死判断 | `BackgroundWorkerHandle` + `GetBackgroundWorkerPid()` | 对端未 attach 时，区分“还没来”和“不会来了”。 |
| 通道断开 | `mq_detached` | 让 send / receive / wait attach 看到终止状态。 |
| 唤醒传播 | `SetLatch(counterparty)` | 对端如果正在 full / empty / attach wait，能醒来重新检查。 |
| DSM 兜底 | `on_dsm_detach()` callback | 本端 DSM mapping 消失时，自动把 queue 标为 detached。 |
| 最后消息 drain | pending flush + receive-side recheck | sender detach 不丢已发布数据。 |
| 上层补充判断 | `parallel.c` worker attach / finish loops | 捕获 shm_mq 无法单独表达的 worker 初始化失败和干净退出。 |

`mq_detached` 本身不需要复杂锁保护，因为它只从 false 变 true。重复写 true 没问题。源码依赖 latch 的 barrier 语义：

```text
设置 mq_detached
  -> SetLatch(counterparty)
  -> 对端 WaitLatch 返回 / ResetLatch
  -> 对端重新读 mq_detached
```

这让“断开”成为一个可传播的状态变化，而不是只存在于本进程栈上的错误。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 注册 worker 失败

`LaunchParallelWorkers()` 里，如果 `RegisterDynamicBackgroundWorker()` 失败：

```text
bgwhandle = NULL
shm_mq_detach(error_mqh)
error_mqh = NULL
```

为什么要 detach error queue？

```text
leader 已经为这个 worker 分配了 queue；
但 worker 根本不会启动；
如果不 detach，后续可能等待一个永远不会 attach 的 sender。
```

这里的 fallback 是减少实际 launched worker 数，而不是让 queue 悬空。

### 8.2 worker attach 前停止

`WaitForParallelWorkersToAttach()` 和 `WaitForParallelWorkersToFinish()` 都会检查：

```text
GetBackgroundWorkerPid(bgwhandle, &pid) == BGWH_STOPPED
且 shm_mq_get_sender(error_mq) == NULL
```

此时报错：

```text
parallel worker failed to initialize
```

这是上层比 `shm_mq` 更懂业务语义的地方：

```text
worker 还没 attach error queue 就停止，
leader 没有办法从 queue 里收到 ErrorResponse；
只能报告初始化失败，并提示更多细节可能在 server log。
```

### 8.3 attach 后 ERROR

worker attach error queue 后，`pq_redirect_to_shm_mq()` 把协议消息写入 queue。leader 在 `HandleParallelMessages()` 中非阻塞读取：

```text
res = shm_mq_receive(error_mqh, &nbytes, &data, true)
if res == WOULD_BLOCK:
  break
if res == SUCCESS:
  ProcessParallelMessage(...)
else:
  ERROR "lost connection to parallel worker"
```

这说明：

```text
attach 前失败:
  leader 只能判断 failed to initialize

attach 后 ERROR:
  worker 可以通过 error queue 发送 ErrorResponse / NoticeResponse

queue 异常断开:
  leader 报 lost connection
```

### 8.4 receiver 提前 detach

如果 receiver detach，sender 的后续 send 会在这些位置看到 `SHM_MQ_DETACHED`：

```text
shm_mq_send_bytes():
  compiler barrier 后检查 mq_detached

ring full 等 receiver attach 或消费时:
  counterparty_gone / wait_internal 返回失败

shm_mq_sendv() 完成后:
  再检查 mq_detached
```

上层例子：

```text
tqueueReceiveSlot():
  send tuple 返回 DETACHED -> 返回 false

pqmq.c:
  send protocol message 返回 DETACHED -> 返回 EOF
```

### 8.5 sender 提前 detach

receiver 的语义更细：

```text
如果 ring 里还有完整消息:
  继续返回 SUCCESS

如果没有足够数据且 mq_detached:
  返回 SHM_MQ_DETACHED
```

这就是“正常结束”和“连接丢失”之间需要上层解释的地方。`TupleQueueReaderNext()` 把 `SHM_MQ_DETACHED` 当作：

```text
没有剩余 tuple，done = true
```

但 error queue 中收到 `DETACHED` 可能被 leader 解释成：

```text
lost connection to parallel worker
```

同一个底层结果，在不同通道语义下对应不同上层含义。

### 8.6 没有 `BackgroundWorkerHandle`

`shm_mq_wait_internal()` 的注释很直接：

```text
handle == NULL 时，如果对端失败且永远不 attach，可能永远等待。
```

这不是 `shm_mq` 的疏漏，而是边界：

```text
queue 只知道 endpoint pointer；
不知道某个未来进程是否应该来 attach。
```

如果调用者需要诊断“未来进程不会来了”，就必须提供可观测生命周期，例如 `BackgroundWorkerHandle`，或在上层建立自己的 worker-ready 协议。

## 9. 成本、资源与跨模块传播

`shm_mq` failure 传播的成本不在 hot data copy，而在等待与状态检查：

| 成本 | 场景 | 影响 |
| --- | --- | --- |
| `mq_mutex` 短持有 | endpoint pointer 读写 | 很短，只保护 attach 身份发布。 |
| `GetBackgroundWorkerPid()` | 未 attach 对端等待 / nowait 检查 | 需要查询 bgworker 状态，通常不在每个消息 hot path。 |
| `SetLatch()` | endpoint attach / detach / read progress / write progress | 唤醒跨进程等待，成本比普通内存写高。 |
| wait event | attach、send、receive、parallel finish | 提供诊断入口，也意味着调用者进入睡眠。 |
| callback 注册/取消 | `shm_mq_attach()` / `shm_mq_detach()` | 让 DSM detach 成为 queue detach 的兜底。 |

跨模块传播最明显的是 parallel query：

```text
shm_mq 只报告 queue 状态；
parallel.c 把 queue 状态和 bgworker 状态组合成用户可理解的错误：
  parallel worker failed to initialize
  lost connection to parallel worker
```

这是一条重要边界：

```text
底层 primitive 不应该猜测业务语义；
上层模块必须把 DETACHED 翻译成“正常 EOF”还是“异常断连”。
```

## 10. 观测与诊断入口

### 10.1 wait event

| wait event | 位置 | 含义 |
| --- | --- | --- |
| `WAIT_EVENT_MESSAGE_QUEUE_INTERNAL` | `shm_mq_wait_internal()` | 正在等对端 attach。 |
| `WAIT_EVENT_MESSAGE_QUEUE_SEND` | `shm_mq_send_bytes()` | sender 等 receiver 消费或 attach。 |
| `WAIT_EVENT_MESSAGE_QUEUE_RECEIVE` | `shm_mq_receive_bytes()` | receiver 等 sender 写入。 |
| `WAIT_EVENT_MESSAGE_QUEUE_PUT_MESSAGE` | `pqmq.c` | worker 发送协议消息遇到 queue 不可写。 |
| `WAIT_EVENT_BGWORKER_STARTUP` | `parallel.c` | leader 等 postmaster 启动 worker。 |
| `WAIT_EVENT_PARALLEL_FINISH` | `parallel.c` | leader 等 worker 完成并处理尾部消息。 |

诊断时先分层：

```text
MESSAGE_QUEUE_INTERNAL:
  端点还没 attach，先看 bgworker 是否启动。

MESSAGE_QUEUE_SEND / RECEIVE:
  端点通常已经 attach，更多是数据面背压或空等。

BGWORKER_STARTUP:
  还在 worker lifecycle 层，queue 可能尚未参与。
```

### 10.2 gdb 断点

适合断点：

```text
break shm_mq_attach
break shm_mq_set_sender
break shm_mq_set_receiver
break shm_mq_wait_internal
break shm_mq_counterparty_gone
break shm_mq_detach_internal
break shm_mq_detach_callback
break WaitForParallelWorkersToAttach
break HandleParallelMessages
```

关键观察：

```text
p mq->mq_sender
p mq->mq_receiver
p mq->mq_detached
p mqh->mqh_handle
p mqh->mqh_counterparty_attached
p pg_atomic_read_u64(&mq->mq_bytes_written)
p mqh->mqh_send_pending
```

如果怀疑 worker attach 前失败，重点看：

```text
GetBackgroundWorkerPid() 返回值
mq_sender 是否仍为 NULL
parallel.c 是否报 failed to initialize
```

如果怀疑正常结束被误解为断连，重点看：

```text
这是 tuple queue 还是 error queue；
上层如何处理 SHM_MQ_DETACHED；
sender detach 前是否已经 flush pending writes。
```

### 10.3 日志和 SQL 现象

常见现象：

```text
parallel worker failed to initialize
  worker 没有 attach error queue 就停止。

lost connection to parallel worker
  leader 读 error queue 时发现 queue detached，而不是收到完整错误消息。

parallel 查询等待 message queue internal
  可能在等 worker attach queue。
```

这些现象都不能只从 `shm_mq` 判断根因。需要结合：

```text
server log 中 worker 早期错误；
max_worker_processes / max_parallel_workers；
pg_stat_activity wait_event；
并行计划和 worker 启动数量；
是否发生 postmaster death / backend interrupt / transaction abort。
```

## 11. 常见误区

误区一：`shm_mq_attach()` 成功说明对端也 attach 了。

不是。它只说明当前 backend 建立了本地 handle，并可能注册了 DSM detach callback。对端 attach 要看 `mq_sender` / `mq_receiver`。

误区二：`mq_sender != NULL` 说明 worker 仍然活着。

不是。它只说明 sender 曾经发布过自己的 `PGPROC`。worker 是否仍活着需要上层 worker lifecycle 判断。

误区三：`SHM_MQ_WOULD_BLOCK` 是错误。

不是。它表示非阻塞模式下暂时不能推进。调用者通常应等 latch 或稍后重试。

误区四：`SHM_MQ_DETACHED` 总是异常。

不是。tuple queue 中它可以表示数据流结束；error queue 中它可能表示 worker 连接丢失。语义由上层通道用途决定。

误区五：有 DSM detach callback 就不需要 `shm_mq_detach()`。

不对。显式 `shm_mq_detach()` 会 flush pending writes、取消 callback、释放本地 buffer 和 handle。callback 是兜底，不是完整析构。

误区六：没有 `BackgroundWorkerHandle` 也能可靠判断未来 worker 不会 attach。

不能。`shm_mq` 只能看到 queue 自己的字段；未来进程是否会出现是 bgworker 层或调用者协议的信息。

## 12. 课堂实验

### 实验一：观察 attach 前等待

在 `shm_mq_wait_internal()` 下断点，运行一个会启动 parallel worker 的查询或 `test_shm_mq`。

观察：

```text
ptr 指向 mq_sender 还是 mq_receiver
*ptr 何时从 NULL 变成 PGPROC *
WaitLatch 的 wait_event 是否为 MESSAGE_QUEUE_INTERNAL
```

目标：理解 attach wait 等的是 endpoint pointer，而不是消息数据。

### 实验二：模拟 worker 启动失败

把 `max_worker_processes` 或相关 parallel worker 配额调低，运行需要 parallel worker 的路径。观察：

```text
RegisterDynamicBackgroundWorker() 是否失败；
parallel.c 是否 detach 对应 error_mqh；
实际 launched worker 数是否少于计划 worker 数。
```

目标：理解注册失败和 attach 失败是不同阶段。

### 实验三：观察 worker attach 前停止

在 worker 初始化早期制造 ERROR，位置尽量早于 `shm_mq_set_sender()`。观察 leader 是否报：

```text
parallel worker failed to initialize
```

再把 ERROR 放到 `pq_redirect_to_shm_mq()` 之后，比较 leader 是否能从 error queue 收到更完整的错误消息。

目标：理解 error queue attach 是 worker 错误可传播的分界线。

### 实验四：验证 detach 前 flush pending writes

在 `shm_mq_detach()` 和 `shm_mq_inc_bytes_written()` 下断点，让 sender 写入后立即退出。

观察：

```text
mqh_send_pending > 0 时 detach 是否先推进 mq_bytes_written；
receiver 是否还能读到最后已发布消息；
随后 receive 是否返回 DETACHED。
```

目标：理解 detach 不是丢弃消息，而是先发布可 drain 的尾部状态。

### 实验五：区分 tuple queue 和 error queue 的 DETACHED

对比：

```text
tqueue.c:
  SHM_MQ_DETACHED -> done = true 或 send false

parallel.c HandleParallelMessages:
  SHM_MQ_DETACHED -> lost connection to parallel worker
```

目标：理解底层 result 需要上层语义翻译。

## 13. 讨论题

1. 为什么 `shm_mq_wait_internal()` 不能只等 `mq_detached` 或 endpoint pointer，而需要可选 `BackgroundWorkerHandle`？
2. `mq_sender` / `mq_receiver` 为什么设置后不清空？如果 detach 时清空会引入哪些竞态？
3. 为什么 DSM detach callback 不能直接 `pfree(mqh)`？
4. `SHM_MQ_DETACHED` 在 tuple queue 和 error queue 中为什么不是同一种业务含义？
5. 如果一个上层模块不用 background worker，而用普通 backend 协作，它需要用什么协议替代 `BackgroundWorkerHandle`？
6. 为什么 worker attach error queue 之前的 ERROR 不能通过 error queue 传回 leader？
7. `WaitForParallelWorkersToAttach()` 为什么建议 leader 尽量晚调用，而不是启动 worker 后立刻等所有 worker attach？

## 14. 本节小结

`shm_mq` 的 attach / detach / failure 传播不是另一个消息格式问题，而是一个端点生命周期问题：

```text
mq_sender / mq_receiver:
  对端是否曾经发布 attach 身份。

BackgroundWorkerHandle:
  对端如果还没 attach，是否还有可能出现。

mq_detached:
  通道是否已经不会继续推进。

SetLatch:
  把 attach、detach、progress 变成等待方能重新检查的事件。

on_dsm_detach callback:
  当 DSM mapping 消失时，兜底发布通道断开。
```

本节最重要的判断是：

```text
SHM_MQ_WOULD_BLOCK 是“现在不能推进，但未来可能可以”；
SHM_MQ_DETACHED 是“这条通道不应再被当作可推进通道等待”。
```

可迁移规律：

```text
跨进程通道不能只设计数据结构；
还必须设计端点出现、端点消失、等待被取消、最后状态被 drain 的协议。
底层 primitive 负责把状态变化做成可靠信号，
上层模块负责把这些信号翻译成业务上的 EOF、初始化失败或连接丢失。
```

下一节会离开 `shm_mq`，进入 DSA：当 DSM 不再只承载固定 queue 或 TOC chunk，而要支持大量跨进程动态对象时，PostgreSQL 如何用 `dsa_area` 和 `dsa_pointer` 建立共享 heap。
