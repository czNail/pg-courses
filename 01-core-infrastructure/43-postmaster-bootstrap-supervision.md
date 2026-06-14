# PostgreSQL postmaster bootstrap supervision

## 课程定位

前置知识：已经理解 PostgreSQL 的多进程模型、main shared memory 启动期创建、signal / latch、`proc_exit()` / `shmem_exit()` 的基本语义。

本节唯一主问题：

```text
postmaster 如何完成集群级启动、监听、子进程监督和 crash policy？
```

核心矛盾：

```text
postmaster 必须掌握整个集群的生死和重启节奏
  vs
postmaster 又必须尽量不参与 shared memory 中的复杂运行时协议，
否则一个损坏的 backend 可能把监督者一起拖死。
```

学完后应能判断：哪些状态属于 postmaster 私有监督状态，哪些状态属于 shared memory 中的 child 活跃标记；哪些 child 退出可以局部清理，哪些必须触发集群级 crash restart。

本课基于本地源码：

```text
/home/nail/postgres
commit 0e1f1ed157e
```

本节只讲 postmaster 作为集群监督者的主链路。客户端认证、database attach、role / session 初始化和完整 background worker API 留给后续课程。

## 1. 本节在总主线中的位置

前面课程已经建立了：

```text
main shared memory
  -> PGPROC / ProcArray
  -> signal / latch
  -> DSM / parallel worker
```

这些机制都有一个共同前提：必须先有一个稳定的集群级父进程，决定 shared memory 何时创建，普通 backend 何时可进入，以及异常 child 退出后当前 shared memory 是否还能被信任。

这个父进程就是 postmaster。

它不是普通 backend：

- 不加入 `ProcArray`。
- 不分配自己的 `PGPROC`。
- 不参与 lock manager。
- 一般不在 shared memory 上做复杂操作。

本节的问题不是“一个 backend 如何启动”，而是：

```text
postmaster 如何把启动、监听、fork child、收割 child、crash restart
压成一个集群级监督状态机？
```

推荐把本节放在 shared memory 启动期课程之后阅读。原因是 postmaster 先创建 main shmem，启动 startup / checkpointer / bgwriter，再由 startup process 推进到可接受连接的状态；之后普通 backend 才能获得 `PGPROC` 并进入 ProcArray / lock / latch 体系。

后续第 44 节会继续追客户端连接从 postmaster fork/exec 到 backend bootstrap 的细节。本节只讲 postmaster 侧。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
PostmasterMain() 完成数据目录、GUC、shared memory、监听 socket 和首批辅助进程启动后，
进入 ServerLoop()；
ServerLoop() 用 latch + listen sockets 响应 signal、连接和子进程退出；
每个 child 都挂在 postmaster 私有 PMChild / ActiveChildList 上，
同时用 shared memory 中的 PMChildFlags 暴露“是否已经活跃使用 shared memory”；
一旦 child 异常退出或未干净 detach，postmaster 不尝试局部修补 shared memory，
而是让其它 children quickdie，等待它们退出，重建 shared memory，并重新启动 startup process。
```

这条模型有三个边界：

```text
启动边界:
  data directory lock
    -> control file
    -> preload / shmem request
    -> CreateSharedMemoryAndSemaphores()
    -> listen sockets
    -> startup process

监听边界:
  pm_wait_set
    -> latch for signals
    -> WL_SOCKET_ACCEPT for listen sockets
    -> BackendStartup()

监督边界:
  SIGCHLD
    -> waitpid()
    -> identify PMChild
    -> release child slot
    -> decide local cleanup or cluster reset
```

postmaster 要知道 enough to supervise，但不能知道 too much about shared memory internals。因此它关心的是：

- 当前集群状态是否能接受连接。
- 哪些 child 还活着。
- 哪个 child slot 被哪个 PID 占用。
- child 退出时有没有正常跑完 shared memory cleanup。
- crash 后是否允许自动 restart。
- 后台进程是否应该在当前 `pmState` 下启动。

它不应该关心：

- 某个 backend 具体持有哪些 buffer pin。
- 某个事务在什么 subtransaction 层级。
- 某个 lock wait queue 是否仍一致。
- 某个 shared hash table 是否还可被信任。

一旦这些细粒度状态可能被损坏，postmaster 的策略是：

```text
停止所有可能使用旧 shared memory 的进程
  -> 重建 shared memory
  -> 由 startup process 通过 WAL recovery 恢复一致状态
```

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/main/main.c` | dispatch 到 `PostmasterMain()` 或 `SubPostmasterMain()`。 |
| 2 | `src/backend/postmaster/postmaster.c` | 启动、`ServerLoop()`、child cleanup、`PostmasterStateMachine()`、crash policy。 |
| 3 | `src/include/postmaster/postmaster.h` | `PMChild`、postmaster death pipe、child slot API。 |
| 4 | `src/backend/postmaster/pmchild.c` | `ActiveChildList`、固定 child pool、dead-end child、slot assign / release。 |
| 5 | `src/backend/storage/ipc/pmsignal.c` | `PMSignalData`、`PMChildFlags`、`RegisterPostmasterChildActive()`。 |
| 6 | `src/include/storage/pmsignal.h` | `PMSIGNAL_*`、`QuitSignalReason`、`PostmasterIsAlive()`。 |
| 7 | `src/backend/postmaster/launch_backend.c` | `postmaster_child_launch()`、fork 和 `EXEC_BACKEND` bootstrap。 |
| 8 | `src/backend/tcop/backend_startup.c` | `BackendMain()`、pre-auth 阶段、`CAC_state`。 |
| 9 | `src/backend/tcop/postgres.c` | `PostgresMain()`、`quickdie()`、`die()`。 |
| 10 | `src/backend/storage/ipc/ipc.c` | `proc_exit()`、`shmem_exit()`、`on_exit_reset()`。 |
| 11 | `src/backend/postmaster/bgworker.c` | background worker 的 postmaster 侧启动、重启、退避和 crash reset。 |

推荐阅读顺序：

```text
PostmasterMain()
  -> postmaster 私有监督状态
  -> PMChildFlags
  -> ServerLoop()
  -> BackendStartup() / StartChildProcess()
  -> process_pm_child_exit()
  -> HandleChildCrash()
  -> PostmasterStateMachine()
  -> crash restart 重建 shared memory
```

`postmaster.c` 很长，但本课只取集群级 supervision 这条线，不展开认证、各后台进程内部逻辑和 background worker API 细节。

## 4. 关键数据结构与状态

### 4.1 `pmState`

`pmState` 是 postmaster private memory 中的集群状态机，不在 shared memory 中。

主要状态可以按生命周期读：

```text
PM_INIT
  -> PM_STARTUP
  -> PM_RECOVERY / PM_HOT_STANDBY
  -> PM_RUN
  -> PM_STOP_BACKENDS / PM_WAIT_BACKENDS
  -> PM_WAIT_XLOG_SHUTDOWN / PM_WAIT_XLOG_ARCHIVAL
  -> PM_WAIT_IO_WORKERS / PM_WAIT_CHECKPOINTER
  -> PM_WAIT_DEAD_END
  -> PM_NO_CHILDREN
```

`PM_RUN` 表示正常运行；`PM_HOT_STANDBY` 表示 recovery 未结束但允许只读连接；后面的 `PM_WAIT_*` 状态表示 shutdown 或 crash restart 中等待不同 child 集合退出。

`pmState` 不单独表达原因。真实语义要结合：

| 状态 | 含义 |
| --- | --- |
| `Shutdown` | smart / fast / immediate shutdown 请求。 |
| `FatalError` | 是否处在 crash recovery 的 reset 流程中。 |
| `StartupStatus` | startup process 是否 running / signaled / crashed。 |
| `connsAllowed` | smart shutdown 下是否还允许普通连接进入。 |

所以 `PM_WAIT_BACKENDS` 既可能是正常 shutdown，也可能是 crash restart 的等待阶段。

### 4.2 `PMChild` 和 `ActiveChildList`

`PMChild` 定义在 `src/include/postmaster/postmaster.h`，是 postmaster 监督 child 的最小单位。

关键字段是 `pid`、`child_slot`、`bkend_type`、`rw`、`bgworker_notify` 和链表节点 `elem`。它们让 postmaster 能从 `waitpid()` 得到的 PID 回到 child 类型和 supervision policy。

`PMChild` 是 postmaster private state，不是 shared memory。

`ActiveChildList` 挂住所有 postmaster 知道的活跃 child，包括普通 backend、辅助进程、background worker、syslogger 和 dead-end backend。

`SignalChildren()` 和 `CountChildren()` 都扫描这个链表，因此 shutdown、crash restart、worker storm 时成本随活跃 child 数线性增长。它不是 SQL hot path，但会影响 postmaster loop。

`pmchild.c` 用不同 pool 给不同 child 类型预留 slot。例如普通 backend pool 是：

```text
2 * (MaxConnections + max_wal_senders)
```

这是为了容纳认证中的连接。它不等价于已经进入 `PGPROC` 的 backend 数。

### 4.3 `PMChildFlags`

`PMChildFlags` 位于 shared memory 的 `PMSignalData` 中，只表达 child slot 的活跃状态。

| 状态 | 语义 |
| --- | --- |
| `PM_CHILD_UNUSED` | slot 可用。 |
| `PM_CHILD_ASSIGNED` | postmaster 已分配 slot，但 child 尚未活跃使用 shared memory，或已干净 cleanup。 |
| `PM_CHILD_ACTIVE` | child 正在使用 shared memory。 |
| `PM_CHILD_WALSENDER` | 类似 ACTIVE，但 postmaster 可把 backend 重识别为 WAL sender。 |

典型状态推进：

```text
postmaster AssignPostmasterChildSlot()
  -> MarkPostmasterChildSlotAssigned()
  -> ASSIGNED

child active 使用 shared memory 后
  -> RegisterPostmasterChildActive()
  -> ACTIVE
  -> 注册 on_shmem_exit(MarkPostmasterChildInactive)

child 正常 proc_exit()
  -> shmem_exit()
  -> MarkPostmasterChildInactive()
  -> ASSIGNED

postmaster waitpid()
  -> ReleasePostmasterChildSlot()
  -> MarkPostmasterChildSlotUnassigned()
  -> UNUSED
```

如果 child 在 ACTIVE 时 `_exit()`、`SIGKILL`、core dump 或 quickdie，它不会跑 `on_shmem_exit()`。postmaster 收割时发现 slot 不是 `ASSIGNED`，`ReleasePostmasterChildSlot()` 返回 false，并把退出升级为 crash。

这就是 postmaster 的 dead man switch。

### 4.4 `PMSIGNAL_*` 和 `QuitSignalReason`

child 通知 postmaster 使用 `PMSIGNAL_*`：

```text
child:
  PMSignalState->PMSignalFlags[reason] = true
  kill(PostmasterPid, SIGUSR1)

postmaster signal handler:
  pending_pm_pmsignal = true
  SetLatch(MyLatch)

ServerLoop:
  process_pm_pmsignal()
```

它是 flag，不是队列。同一 reason 的重复信号可能合并。因此它适合表达“请重新检查某类状态”，不适合表达精确计数事件。

反方向上，postmaster 用 `SetQuitSignalReason()` 写入 `QuitSignalReason`。child 收到 SIGQUIT 后，`quickdie()` 通过 `GetQuitSignalReason()` 区分 crash restart 和 immediate stop，并尽量向客户端发出不同 warning。

### 4.5 postmaster death pipe

非 Windows 平台上，postmaster 创建 `postmaster_alive_fds`：child 持有读端 `POSTMASTER_FD_WATCH`，postmaster 持有写端 `POSTMASTER_FD_OWN`。child 在 `ClosePostmasterPorts()` 中尽早关闭写端。这样 postmaster 死亡时，child 读端能看到 EOF。

这个机制的目标是：child 发现监督者不在了，就不能继续假装集群状态仍被监督。

## 5. 主流程源码 walkthrough

### 5.1 程序入口

`src/backend/main/main.c` 根据 first dispatch option 进入不同模式：

```text
DISPATCH_POSTMASTER
  -> PostmasterMain(argc, argv)

DISPATCH_FORKCHILD
  -> SubPostmasterMain(argc, argv)    -- EXEC_BACKEND
```

普通 server 运行进入 `PostmasterMain()`。Windows 或 `EXEC_BACKEND` 构建中，child 通过 `SubPostmasterMain()` 恢复到类似 fork 后的状态。

### 5.2 `PostmasterMain()` 启动主线

启动主线可以压缩为：

```text
InitProcessGlobals()
setup PostmasterContext / signal handlers / GUC
SelectConfigFiles()
checkDataDir() / checkControlFile()
CreateDataDirLockFile()
LocalProcessControlFile(false)

RegisterBuiltinShmemCallbacks()
process_shared_preload_libraries()
InitializeMaxBackends()
InitPostmasterChildSlots()
process_shmem_requests()
ShmemCallRequestCallbacks()
InitializeShmemGUCs()
CreateSharedMemoryAndSemaphores()

InitPostmasterDeathWatchHandle()
open listen sockets
RemovePgTempFiles()
autovac_init()
load_hba() / load_ident()

UpdatePMState(PM_STARTUP)
StartChildProcess(B_CHECKPOINTER)
StartChildProcess(B_BG_WRITER)
StartChildProcess(B_STARTUP)
StartupStatus = STARTUP_RUNNING
maybe_start_bgworkers()

ServerLoop()
```

关键不变量：data directory lock 早于监听 socket；shared memory 创建早于普通 backend；`InitPostmasterChildSlots()` 早于 `PMSignalShmemRequest()` 的 sizing；postmaster 不自己 redo，而是启动 `B_STARTUP` 并等待它推进状态。

### 5.3 signal handler 和 `ServerLoop()`

postmaster handler 只设置 pending flag 并 set latch。`SIGHUP` 表示 reload；`SIGTERM` / `SIGINT` / `SIGQUIT` 分别进入 smart / fast / immediate shutdown；`SIGUSR1` 表示 child 或 pg_ctl 通知；`SIGCHLD` 表示 child 退出。

复杂逻辑在 `ServerLoop()` 中执行：

```text
ConfigurePostmasterWaitSet(true)

for (;;):
  WaitEventSetWait(pm_wait_set, DetermineSleepTime(), ...)

  for each event:
    if latch set:
      ResetLatch(MyLatch)

    process pending shutdown / reload / child exit / pmsignal

    if WL_SOCKET_ACCEPT:
      AcceptConnection()
      BackendStartup()
      close accepted socket in postmaster

  LaunchMissingBackgroundProcesses()
  maybe signal autovac launcher
  check SIGKILL timeout
  recheck postmaster.pid once per minute
  touch Unix socket files every 58 minutes
```

`WaitEventSetWait()` 的 wait event 参数是 0，源码注释明确 postmaster 不发布 wait events。因此不要期待在 `pg_stat_activity` 中看到 postmaster 自己的 wait event。

### 5.4 `canAcceptConnections()` 和 dead-end child

`canAcceptConnections()` 只服务 `B_BACKEND` 和 `B_AUTOVAC_WORKER`。

核心判断是：只有 `PM_RUN` / `PM_HOT_STANDBY` 可以接受普通连接；shutdown、startup、尚未 hot standby 的 recovery、crash recovery 会分别返回 `CAC_SHUTDOWN`、`CAC_STARTUP`、`CAC_NOTHOTSTANDBY` 或 `CAC_RECOVERY`。smart shutdown 还会通过 `connsAllowed` 禁止新的普通连接。

`BackendStartup()` 会先尝试普通 `B_BACKEND` slot。如果状态不允许或 slot 不足，则分配 dead-end child：

```text
cac = canAcceptConnections(B_BACKEND)
if cac == CAC_OK:
  bn = AssignPostmasterChildSlot(B_BACKEND)
  if no slot:
    cac = CAC_TOOMANY

if no normal slot:
  bn = AllocDeadEndChild()

startup_data.canAcceptConnections = cac
postmaster_child_launch(...)
```

dead-end child 的作用是用正常协议向客户端返回“starting up / shutting down / recovery / too many clients”等错误。它仍挂在 `ActiveChildList`，因为它可能 attach shared memory，shutdown 或 crash restart 必须等它 drain。

### 5.5 `postmaster_child_launch()`

Unix 非 `EXEC_BACKEND` 的 child 路径：

```text
fork_process()

child:
  MyBackendType = child_type
  ClosePostmasterPorts(child_type == B_LOGGER)
  InitPostmasterChild()

  if child type does not need shmem:
    dsm_detach_all()
    PGSharedMemoryDetach()

  MemoryContextSwitchTo(TopMemoryContext)
  MyPMChildSlot = child_slot
  copy client socket if present
  child_process_kinds[child_type].main_fn(startup_data, startup_data_len)
```

`ClosePostmasterPorts()` 释放 postmaster wait set、关闭 death pipe 写端、关闭 listen sockets，并按 child 类型关闭不需要的 syslogger / Bonjour fd。

`InitPostmasterChild()` 设置 `IsUnderPostmaster`，重新初始化 process globals 和 latch，清空继承来的 postmaster exit callbacks，尝试 `setsid()`，安装默认 SIGQUIT crash-exit handler，并初始化 postmaster death 检测。

`on_exit_reset()` 是 fork inheritance 的安全边界：child 不能继承并运行 postmaster 的 `on_proc_exit()` / `on_shmem_exit()` callbacks。

### 5.6 `EXEC_BACKEND` 等价启动

`EXEC_BACKEND` 下 child 不继承完整进程镜像。postmaster 把关键变量写入 `BackendParameters`，child 在 `SubPostmasterMain()` 中恢复：

```text
parse --forkchild=<kind>
read_backend_variables()
restore_backend_variables()
ClosePostmasterPorts()
InitPostmasterChild()
PGSharedMemoryReAttach() or PGSharedMemoryNoReAttach()
read_nondefault_variables()
checkDataDir() / LocalProcessControlFile(false)
RegisterBuiltinShmemCallbacks()
process_shared_preload_libraries()
InitShmemAllocator()
ShmemCallRequestCallbacks()
dispatch to child main_fn
```

稳定语义不是“必须 fork 继承”，而是 child 进入自己的 `Main` 前必须具有等价的 postmaster-child 环境。

### 5.7 `BackendMain()` 和 pre-auth 边界

`BackendMain()` 的主线：

```text
BackendInitialize(MyClientSocket, bsdata->canAcceptConnections)
InitProcess()
MemoryContextSwitchTo(TopMemoryContext)
PostgresMain(database_name, user_name)
```

`BackendInitialize()` 读取 SSL / startup packet，并根据 `CAC_state` 向客户端发送拒绝原因。

关键边界：读取 startup packet 前，代码要求还没有触碰需要 shared-memory cleanup 的状态。SIGTERM 和 startup packet timeout handler 都直接 `_exit(1)`，理由是此时尚未触碰 shared memory，可以不跑 `proc_exit()`。

函数末尾调用：

```text
check_on_shmem_exit_lists_are_empty()
```

之后 `InitProcess()` 才创建 per-backend `PGPROC`，backend 才能使用 LWLock 和 shared memory。

### 5.8 `PostgresMain()` 的 SIGTERM / SIGQUIT 语义

`PostgresMain()` 中普通 backend 的相关 signal handlers：

```text
SIGTERM -> die()
SIGQUIT -> quickdie()    -- under postmaster
SIGINT  -> StatementCancelHandler()
SIGHUP  -> SignalHandlerForConfigReload()
SIGUSR1 -> procsignal_sigusr1_handler()
```

`die()` 是温和退出：设置 `InterruptPending` 和 `ProcDiePending`，再由安全点的 `CHECK_FOR_INTERRUPTS()` 转成 FATAL 或 `proc_exit()`。

`quickdie()` 是 crash / immediate shutdown 路径：直接 `_exit(2)`，不跑 `proc_exit()`、`shmem_exit()`、transaction abort、lock release、DSM callbacks 或 `MarkPostmasterChildInactive()`。

这不是遗漏，而是 crash policy。如果 shared memory 已不可信，backend 不能继续在旧 shared memory 上做复杂 cleanup。

### 5.9 child 退出收割

`SIGCHLD` handler 只设置 pending flag。`process_pm_child_exit()` 真正处理：

```text
while waitpid(-1, &exitstatus, WNOHANG) > 0:
  if pid is StartupPMChild:
    handle startup-specific policy
  else if pid is special child:
    release slot and apply per-process policy
  else if maybe_reap_io_worker(pid):
    handle io worker policy
  else if FindPostmasterChildByPid(pid):
    CleanupBackend(pmchild, exitstatus)
  else:
    unknown child policy

PostmasterStateMachine()
```

源码中并非所有 nonzero exit 都是 crash：

- `exit 0` 通常表示正常退出。
- `exit 1` 通常表示 FATAL，许多 child 类型可接受。
- 其它 exit status 或 signal termination 通常视为 crash。

但还有第二个判断：child 是否干净撤销 `PMChildFlags` active state。

### 5.10 `CleanupBackend()` 和 crash 判定

普通 backend / background worker 统一走 `CleanupBackend()`。

核心逻辑：

```text
crashed = !(exit status 0 or 1)

if !ReleasePostmasterChildSlot(bp):
  crashed = true

if crashed:
  HandleChildCrash(pid, exitstatus, procname)
  return

cancel bgworker notifications if needed
if autovac worker:
  signal launcher
if background worker:
  update rw_crashed_at / rw_terminate
  ReportBackgroundWorkerExit()
```

`ReleasePostmasterChildSlot()` 会把 `PMChild` 从 `ActiveChildList` 删除，回收到 pool，并调用 `MarkPostmasterChildSlotUnassigned()`。

如果 child slot 不是 `ASSIGNED`，说明 child 没有正常跑 `MarkPostmasterChildInactive()`，即使 exit status 表面正常，也会升级为 crash。

background worker 有两层“crash”概念：

```text
集群级:
  abnormal exit 或未清理 active slot -> HandleChildCrash()

worker 自身 restart:
  非 0 exit 但 shared memory cleanup 干净 -> rw_crashed_at = now，按 bgw_restart_time 重启
```

不要混用这两层。

### 5.11 `HandleChildCrash()` 到 crash restart

`HandleChildCrash()`：

```text
if FatalError or immediate shutdown:
  return

LogChildExit(LOG, ...)
log "terminating any other active server processes"
HandleFatalError(PMQUIT_FOR_CRASH, true)
```

`HandleFatalError()`：

```text
SetQuitSignalReason(reason)
TerminateChildren(SIGQUIT or SIGABRT)
FatalError = true
UpdatePMState(PM_WAIT_BACKENDS or later state)
AbortStartTime = time(NULL)
```

如果 children 在 `SIGKILL_CHILDREN_AFTER_SECS` 内不退出，`ServerLoop()` 会发送 `SIGKILL` 或配置要求的 `SIGABRT`。当前常量是 5 秒。

### 5.12 `PostmasterStateMachine()` 重建 shared memory

crash restart 的主链路：

```text
FatalError = true
pmState = PM_WAIT_BACKENDS

wait target children
  -> PM_WAIT_DEAD_END
  -> PM_NO_CHILDREN

if FatalError && pmState == PM_NO_CHILDREN:
  log "all server processes terminated; reinitializing"
  RemovePgTempFiles() if configured
  ResetBackgroundWorkerCrashTimes()
  shmem_exit(1)
  ResetShmemAllocator()
  ShmemCallRequestCallbacks()
  CreateSharedMemoryAndSemaphores()
  UpdatePMState(PM_STARTUP)
  StartChildProcess(B_STARTUP)
  ConfigurePostmasterWaitSet(true)
```

这说明 crash restart 不是“重新 fork 崩溃的 backend”，而是：

```text
杀掉所有仍可能引用旧 shared memory 的 child，
重建 main shared memory / semaphores，
启动 startup process 通过 WAL recovery 让集群重新一致。
```

如果 `restart_after_crash = off`，postmaster 在 `PM_NO_CHILDREN` 后退出，让外部 supervisor 接管。旧 shared memory 仍然不可信。

## 6. 生命周期 / ownership / cleanup

postmaster 创建或初始化：

- data directory lock file。
- `PostmasterContext`。
- signal handlers 和 local latch。
- main shared memory / semaphores。
- postmaster death pipe。
- listen sockets。
- postmaster child slot pool。
- startup / checkpointer / bgwriter / syslogger 等 children。
- `ActiveChildList` 中的 `PMChild` bookkeeping。

普通 backend child 逐步获得：

```text
postmaster-child environment
  -> client socket
  -> local latch
  -> MyPMChildSlot
  -> PGPROC via InitProcess()
  -> database / role / session via InitPostgres()
  -> transaction / ResourceOwner / MemoryContext during execution
```

postmaster 不直接持有 backend 的事务资源、buffer pin、lock、snapshot 或 catalog ref。

postmaster 只持有监督 bookkeeping：

```text
PMChild(pid, child_slot, bkend_type)
```

以及通过 `PMChildFlags` 判断 child 是否干净退出 shared memory。

正常 child 退出：

```text
child proc_exit()
  -> shmem_exit()
     -> before_shmem_exit callbacks
     -> dsm_backend_shutdown()
     -> on_shmem_exit callbacks
        -> MarkPostmasterChildInactive()
  -> on_proc_exit callbacks

postmaster waitpid()
  -> ReleasePostmasterChildSlot()
  -> MarkPostmasterChildSlotUnassigned()
```

quickdie / crash 退出：

```text
child quickdie()
  -> _exit(2)
  -> no proc_exit()
  -> no shmem_exit()
  -> PMChildFlags remains ACTIVE / WALSENDER

postmaster waitpid()
  -> ReleasePostmasterChildSlot() returns false
  -> HandleChildCrash()
```

crash restart 的 cleanup 分层：backend-local memory / fd 由 OS 在进程退出时回收；旧 shared memory 整体废弃并重建；一致性由 startup process recovery；temp files 按 `remove_temp_files_after_crash` 删除；bgworker metadata 由 `ResetBackgroundWorkerCrashTimes()` 处理。

postmaster 自己退出必须走 `ExitPostmaster(status) -> proc_exit(status)`，不要直接 `exit()`。但 crash restart 通常不是 postmaster 退出，而是在同一个 postmaster 进程内重建 shared memory 并重启 startup process。

## 7. 正确性机制层次

| 机制 | 保证什么 | 不保证什么 |
| --- | --- | --- |
| OS process isolation | child 崩溃不会直接破坏 postmaster private memory。 | child 不会破坏 shared memory。 |
| postmaster 不加入 `PGPROC` | 监督者避免被 lock / ProcArray 状态拖死。 | postmaster 完全不读 shared memory。 |
| `PMChild` / `ActiveChildList` | postmaster 知道自己 fork 过哪些 child。 | child 内部资源已释放。 |
| `PMChildFlags` | child 是否 active 使用 shared memory 并是否干净退出。 | shared memory 具体哪里损坏。 |
| signal pending flag + latch | 异步事件转成主循环同步处理。 | 同一 reason 精确计数。 |
| `QuitSignalReason` | child 知道 SIGQUIT 来自 crash restart 还是 immediate stop。 | child 一定能成功发客户端 warning。 |
| `proc_exit()` / `shmem_exit()` | 正常 FATAL / exit 时按 callback 顺序清理。 | signal handler 中复杂 cleanup 安全。 |
| `quickdie()` | 在不可信 shared memory 下快速退出。 | 保留 backend 局部 cleanup。 |
| startup process + WAL recovery | 重建 shared memory 后恢复一致状态。 | 保留崩溃 backend 的事务进度。 |

核心不变量：

```text
postmaster 监督状态必须比 backend shared memory 状态更可信。
```

因此 postmaster 的关键状态大多在 private memory，只读取少量 lockless / crash-tolerant shared flags。

background worker 共享数组也遵循这个约束：backend 可以拿 `BackgroundWorkerLock` 写 slot，但 postmaster 不能拿锁，甚至不能拿 spinlock。backend 写完 slot 后用 memory barrier，再设置 `in_use = true`；postmaster 无锁读取并做防御性检查。

这是 postmaster 特有的监督者约束，不是通用 shared memory 访问模式。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 fork 普通 backend 失败

路径：

```text
BackendStartup()
  -> AssignPostmasterChildSlot()
  -> postmaster_child_launch()
     -> fork_process() fails
  -> ReleasePostmasterChildSlot()
  -> log "could not fork new process for connection"
  -> report_fork_failure_to_client()
```

postmaster 不崩溃，已分配的 `PMChild` slot 被释放。向客户端报告时 socket 会设成 nonblocking，只尝试发送一次，避免 postmaster 被单个客户端阻塞。

### 8.2 启动中或 shutdown 中收到连接

路径：

```text
ServerLoop()
  -> AcceptConnection()
  -> BackendStartup()
  -> canAcceptConnections() != CAC_OK
  -> AllocDeadEndChild()
  -> fork child
  -> BackendInitialize()
  -> 根据 CAC_state 发送 FATAL
```

这让客户端看到明确错误，而不是单纯 connection reset。dead-end child 仍要被 postmaster 追踪。

### 8.3 backend FATAL exit

`ereport(FATAL)` 通常走 `proc_exit(1)`。

如果 backend 已 active 使用 shared memory，正常 cleanup 会跑 `shmem_exit()` 并让 `PMChildFlags` 回到 `ASSIGNED`。

postmaster 收割时：

```text
exit status 1
ReleasePostmasterChildSlot() returns true
=> not cluster crash
```

所以 `exit 1` 不等于 postmaster crash restart。

### 8.4 backend abnormal exit

如果 backend 被 `SIGKILL`、segfault、assert crash 或直接 `_exit(2)`，通常不会跑 `shmem_exit()`。

postmaster 收割时：

```text
exit status not 0/1
or PMChildFlags still ACTIVE / WALSENDER
=> HandleChildCrash()
```

结果是其它连接被断开，shared memory 重建，startup process recovery。

### 8.5 startup process 异常退出

startup process 是集群恢复 gatekeeper。异常退出被视为 catastrophic。

如果 `StartupStatus` 变成 `STARTUP_CRASHED`，最终 `PM_NO_CHILDREN` 时 postmaster 记录：

```text
shutting down due to startup process failure
```

然后退出，而不是无限重试。

### 8.6 immediate shutdown / crash 后 child 不退出

postmaster 设置 `AbortStartTime`。如果 children 在 5 秒内不退出，`ServerLoop()` 发送 `SIGKILL` 或配置要求的 `SIGABRT`。

这是最后兜底：牺牲 child cleanup，换取 postmaster 完成 shutdown / restart。

### 8.7 bgworker restart

bgworker fork 失败或运行后非零退出但 shared memory cleanup 干净时，通常不是集群 crash。

postmaster 记录：

```text
rw_crashed_at = now
rw_pid = 0
HaveCrashedWorker = true
```

之后 `maybe_start_bgworkers()` 根据 `bgw_restart_time` 决定何时重启。`BGW_NEVER_RESTART` 的 worker 会被忘掉。

crash restart 前 `ResetBackgroundWorkerCrashTimes()` 会清零可重启 worker 的 crash time，使其在新 shared memory 初始化后可以立即重启。

## 9. 成本、资源与跨模块传播

启动期成本随这些变量扩张：

| 变量 | 影响 |
| --- | --- |
| `MaxConnections` | backend pool、PGPROC / ProcArray / lock table。 |
| `max_wal_senders` | backend pool 和 WAL sender 相关状态。 |
| `max_worker_processes` | bgworker slots、PMChild pool、MaxBackends。 |
| `autovacuum_worker_slots` | autovac worker pool。 |
| `shared_buffers` | buffer descriptors / buffer blocks。 |
| preload extensions | shmem request、hooks、static bgworkers。 |

连接启动成本包括：

```text
accept()
PMChild slot allocation
fork or fork/exec
ClosePostmasterPorts()
InitPostmasterChild()
BackendInitialize()
InitProcess()
InitPostgres()
```

postmaster 不做认证，而是 fork child 处理 startup packet 和 authentication。这样避免 SSL / PAM / DNS / auth library 阻塞 postmaster，代价是连接风暴会变成大量进程创建和早期 child 初始化成本。

`ActiveChildList` 扫描成本随活跃 child 数线性增长。平时不在 query hot path，但在 fast shutdown、immediate shutdown、crash restart、bgworker storm 和连接风暴时会显性影响 postmaster loop。

`PMSIGNAL_*` 成本低但精度低。同一 reason 的多次通知可能合并，因此 caller 必须把它当成“重新检查状态”的触发器。

跨模块边界：

| 模块 | postmaster 边界 |
| --- | --- |
| shared memory | 创建和重建 main shmem，但尽量不参与复杂 shared state 操作。 |
| WAL recovery | 启动 startup process 并监督状态推进，不自己 redo。 |
| connection startup | accept / fork，child 做协议、认证和 `InitPostgres()`。 |
| ProcArray / PGPROC | child `InitProcess()` 后进入，postmaster 不加入。 |
| background worker | postmaster 启动和重启，backend 通过 shared slot 注册 dynamic worker。 |
| signal / latch | handler 只设 flag，主循环处理复杂逻辑。 |
| logging | syslogger 是特殊 child，退出时 postmaster 先尝试重启 logger。 |

## 10. 观测与诊断入口

能直接观察：

| 入口 | 能看到什么 |
| --- | --- |
| server log | startup、ready、shutdown、child crash、reinitializing、fork failure。 |
| `postmaster.pid` | PID、data directory lock、listen addr、socket dir、PM status 行。 |
| OS `ps` / `pgrep` | postmaster 和 children 的 PID / process title。 |
| `pg_stat_activity` | 普通 backend / autovac / wal sender 等，不包含 postmaster。 |
| `EXPLAIN ANALYZE` | parallel worker 的 `Workers Planned` / `Workers Launched`，间接体现 worker launch。 |

典型 crash restart 日志链：

```text
server process (PID ...) was terminated by signal ...
terminating any other active server processes
all server processes terminated; reinitializing
database system is ready to accept connections
```

不同版本和配置下文字可能略变，但状态链不变：

```text
child crash
  -> FatalError
  -> terminate other children
  -> no children
  -> reinitialize shared memory
  -> startup process recovery
  -> PM_RUN
```

判断 FATAL 还是 crash restart：

```text
只有一个连接断开:
  多半是 session 局部 FATAL / admin termination。

所有连接断开并出现 reinitializing:
  postmaster crash policy 被触发。
```

推荐 gdb 断点：`PostmasterMain`、`UpdatePMState`、`BackendStartup`、`process_pm_child_exit`、`CleanupBackend`、`HandleChildCrash`、`HandleFatalError`、`PostmasterStateMachine`、`RegisterPostmasterChildActive`、`MarkPostmasterChildInactive`、`quickdie`。

推荐观察变量：`pmState`、`Shutdown`、`FatalError`、`StartupStatus`、`connsAllowed`、`AbortStartTime`、`PMChild.pid`、`PMChild.child_slot`、`PMChild.bkend_type`、`PMSignalState->PMChildFlags[slot - 1]`。

只能推断或需要 gdb / 临时日志的状态：

- `pmState` 当前值。
- `FatalError` 是否为 true。
- 某个 `PMChildFlags` slot 是否 ACTIVE。
- `ActiveChildList` 中 dead-end child 数。
- `StartWorkerNeeded` / `HaveCrashedWorker`。

不要把 `pg_stat_activity` 当完整因果。postmaster 自己不是其中的一行，`ServerLoop()` 也不发布 wait event。

## 11. 常见误区

1. 把 postmaster 当成特殊 backend。它不是，它是 supervisor，不加入 `PGPROC`，不参与 ProcArray 和 lock manager。

2. 认为任何 backend FATAL 都会重启集群。`exit 1` 且 shared memory cleanup 干净，通常只是局部 backend 退出。

3. 认为 kill 一个 backend 只会断开那个连接。`SIGTERM` 通常温和退出；`SIGKILL` / SIGQUIT / segfault 可能触发 crash restart。

4. 忽视 dead-end backend。它负责用协议向客户端返回拒绝原因，也必须被 postmaster 等待。

5. 把 bgworker 非零退出都当成集群 crash。非零退出但 cleanup 干净通常只触发 worker restart policy。

6. 认为 postmaster 可以精确诊断 shared memory 损坏位置。postmaster 故意只读少量粗粒度状态；越过可信边界后选择整体重建。

7. 误解 `restart_after_crash=off`。它不是允许局部修复，而是 crash 后 postmaster 不自动 reinitialize，改由外部 supervisor 或人工处理。

## 12. 课堂实验

### 实验 1：跟读正常启动到 `PM_RUN`

目标：观察 `PostmasterMain()` 如何创建 shared memory，启动 startup process，再由 startup process 推进到 `PM_RUN`。

建议断点：

```text
PostmasterMain
UpdatePMState
StartChildProcess
process_pm_child_exit
```

记录状态：

```text
PM_INIT
  -> PM_STARTUP
  -> PM_RUN
```

回到源码核对：

- `PostmasterMain()` 中 `UpdatePMState(PM_STARTUP)`。
- `StartupPMChild = StartChildProcess(B_STARTUP)`。
- startup process exit 0 后 `UpdatePMState(PM_RUN)`。

思考：为什么普通 backend 不能早于 startup process ready 进入？

### 实验 2：区分 FATAL 退出和 crash restart

在临时开发集群中开两个 psql。

第一个 psql：

```sql
select pg_backend_pid();
```

温和终止：

```bash
kill -TERM <pid>
```

预期：该连接断开，其它连接通常仍可用，日志没有 reinitializing。

重新连接，获取新 PID，然后异常终止：

```bash
kill -KILL <pid>
```

预期日志：

```text
terminating any other active server processes
all server processes terminated; reinitializing
database system is ready to accept connections
```

回到源码核对：

- `die()`。
- `quickdie()`。
- `CleanupBackend()`。
- `HandleChildCrash()`。
- `PostmasterStateMachine()`。

只在临时开发集群执行，不要在共享环境或生产环境执行。

### 实验 3：观察 dead-end child 拒绝路径

目标：理解 postmaster 为什么在不能接受连接时仍可能 fork child。

方法：停库同时循环连接。

```bash
pg_ctl -D <datadir> stop -m fast
```

另一个终端：

```bash
while true; do psql -h <host> -p <port> -d postgres -c 'select 1'; sleep 0.2; done
```

观察客户端错误类似：

```text
the database system is shutting down
```

回到源码核对：

- `canAcceptConnections()`。
- `BackendStartup()`。
- `AllocDeadEndChild()`。
- `BackendInitialize()` 中根据 `cac` 发送 FATAL。

## 13. 讨论题

1. 为什么 postmaster 不应该成为 `PGPROC` 成员？

2. 为什么 postmaster 不能在 backend crash 后遍历 lock table 并尝试释放那个 backend 的锁？

3. `PMChild` 和 `PMChildFlags` 分别属于哪个状态边界？

4. 为什么 `exit 1` 通常不是集群级 crash，而 `_exit(2)` 或 `SIGKILL` 往往是？

5. dead-end backend 为什么仍然需要挂在 `ActiveChildList`？

6. 如果同一个 `PMSIGNAL_*` 被连续发送两次但 postmaster 只观察到一次，为什么仍然可以正确工作？

7. background worker 的 restart interval 和 crash restart 后的 `ResetBackgroundWorkerCrashTimes()` 分别解决什么问题？

8. 为什么 immediate shutdown 和 crash restart 都会使用 SIGQUIT，但后续 policy 不完全相同？

9. 如果 server log 只有一个 backend FATAL，没有 reinitializing 日志，应优先怀疑 session 局部错误还是 shared memory corruption？

10. 为什么 `pg_stat_activity` 不能作为 postmaster supervision 的完整诊断入口？

## 14. 本节小结

postmaster 主链路是：`PostmasterMain()` 创建 shared memory / listen sockets，启动 startup 和辅助进程，然后进入 `ServerLoop()` 处理 accept、signal、child exit 和 state machine。

核心状态分两层：postmaster private 的 `pmState`、`Shutdown`、`FatalError`、`StartupStatus`、`ActiveChildList`、`PMChild`；shared minimal flags 中的 `PMSIGNAL_*`、`QuitSignalReason`、`PMChildFlags`。

正常退出链路是：child `proc_exit()` 跑 `shmem_exit()`，`MarkPostmasterChildInactive()` 把 slot 退回 `ASSIGNED`，postmaster `waitpid()` 后 `ReleasePostmasterChildSlot()` 做局部 cleanup 或 bgworker restart。

异常退出链路是：child abnormal exit 或 active slot 未清理，postmaster `HandleChildCrash()` 进入 `FatalError`，向其它 children 发送 SIGQUIT / SIGABRT，等待无 child 后重建 shared memory，再启动 startup process recovery。

可观测入口以 server log 和 OS process tree 为主。SQL 视图只能看到普通 backend 和后台进程的间接状态，不能完整解释 postmaster supervision。

可迁移规律：

```text
监督者不能依赖被监督对象可能破坏的复杂状态来恢复系统。
可靠的 crash policy 往往把监督状态压缩成少量私有、粗粒度、可在损坏环境下读取的事实；
一旦越过可信边界，就不要局部修补，而要重建状态并从日志 / 持久化记录恢复。
```

边界条件：

- fork / exec 成本依赖 OS 和平台。
- crash restart latency 依赖 shared memory size、WAL recovery 量和存储性能。
- 日志文本随版本和配置变化。
- background worker 行为还受扩展代码自己的 exit policy 影响。
- SQL 统计视图只能辅助判断，不能替代 server log、gdb 和源码。

下一节建议继续追：

```text
客户端连接如何从 postmaster fork/exec 进入 backend bootstrap，
哪些状态来自 inherited process image，
哪些状态必须在 child 中重新 attach 或初始化。
```
