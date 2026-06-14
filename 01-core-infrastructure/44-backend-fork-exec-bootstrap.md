# PostgreSQL backend fork/exec bootstrap

## 课程定位
前置知识：已经理解 main shared memory 的 request / init / attach 生命周期，也理解 `PGPROC` 是 backend 进入 shared memory 世界前必须领取的身份槽。
本节唯一主问题：

```text
客户端连接如何从 postmaster fork/exec 进入 backend bootstrap，
哪些状态来自 inherited process image，
哪些状态必须重新 attach？
```
核心矛盾：PostgreSQL 希望普通 Unix backend 启动尽量便宜，直接依赖 `fork()` 继承 postmaster 已初始化的地址空间；但它还必须支持 `EXEC_BACKEND`，也就是子进程重新 `exec` 后失去大多数进程私有内存、函数指针和库装载状态。系统必须在“继承很便宜”和“重新接线可移植”之间保持同一套 backend bootstrap 语义。
学完后应能判断：
- 一个状态是 fork 继承、参数恢复、物理 shared memory reattach，还是 shmem callback attach。
- 为什么 `BackendInitialize()` 在读取 startup packet 前刻意不触碰 shared memory。
- 为什么 `InitProcess()` 必须先分配 `PGPROC`，再在 `EXEC_BACKEND` 下调用 `AttachSharedMemoryStructs()`。
- 为什么 `InitProcessPhase2()` 要等到 `AttachSharedMemoryStructs()` 后再加入 `ProcArray`。
- 连接启动失败时，postmaster、dead-end backend、client backend 分别负责清理什么。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e9`。

## 1. 本节在总主线中的位置
上一节应回答 postmaster 如何完成集群级启动、监听、子进程监督和 crash policy。
本节只追一个客户端连接。
它的时间线不是“socket accepted 后直接进入 SQL”。
更准确的分层是：

```text
postmaster accept socket
  -> 为潜在 backend 分配 postmaster child slot
  -> fork 或 fork/exec 一个子进程
  -> 子进程恢复最小 postmaster-child 环境
  -> 读取 startup packet，必要时拒绝连接
  -> 领取 PGPROC
  -> attach 本进程的 shared memory 指针
  -> PostgresMain / BaseInit / InitPostgres
  -> 认证、database attach、role/session 初始化
  -> NormalProcessing
```
本节只讲进入 backend bootstrap 的边界。
认证、database attach、role/session 和 startup packet GUC 语义留给下一节。
这里要避免两个跳跃：
- 不能把 `fork()` 等同于“已经是 backend”。
- 不能把 shared memory 映射成功等同于“所有 shared state 指针都能用”。
本节的主状态故事是：

```text
一个 client socket
  -> 被 postmaster 接收
  -> 被传给子进程
  -> 变成 MyClientSocket / MyProcPort
  -> 在 startup packet 成功后触发 InitProcess
  -> 领取 MyProc / MyProcNumber
  -> 进入 PostgresMain
  -> 退出时按 shmem_exit / proc_exit 顺序释放
```

## 2. 核心矛盾与一句话运行模型
一句话运行模型：

```text
postmaster 只接受连接并创建子进程；
fork backend 继承 postmaster 的全局变量、已映射 shared memory、预加载库和打开的部分句柄；
EXEC_BACKEND backend 通过 BackendParameters 恢复必要全局变量、重新映射 shared memory、重新加载 preload libraries、重放 shmem request；
两条路径最终都进入 BackendMain()，再由 InitProcess() 建立 PGPROC 并完成本进程 shared memory 指针 attach。
```
如果只看非 `EXEC_BACKEND` Unix 路径，设计似乎简单：
- postmaster 已经创建 main shared memory segment。
- postmaster 已经初始化 `ProcGlobal`、`PMSignalState`、`ShmemIndex` 等全局指针。
- fork 复制虚拟地址空间。
- 子进程看到的 C 全局变量数值和父进程相同。
- shared memory 映射也位于同一虚拟地址。
但这个模型隐含一个前提：

```text
进程私有指针值可以从 postmaster 延续到 backend。
```
`EXEC_BACKEND` 破坏这个前提。
exec 后的新进程需要显式恢复或重建：
- 数据目录路径、执行路径、pkglib 路径。
- postmaster pid、启动时间、reload 时间。
- postmaster alive pipe 或 Windows handle。
- client socket 或重复出来的 socket。
- main shared memory id / handle / expected address。
- `ProcGlobal`、`PMSignalState`、`ProcSignal` 这类 early shared pointer。
- `shared_preload_libraries` 装载及其 callback 函数地址。
- 每个子系统的 C 全局 shared pointer。
这些状态分成四类：
| 状态来源 | 典型例子 | 语义 |
| --- | --- | --- |
| fork inherited process image | 非 `EXEC_BACKEND` 下的 C 全局变量、preload library text、open fd、signal mask | 子进程复制父进程地址空间，但之后各自独立。 |
| `BackendParameters` 显式传递 | `UsedShmemSegID`、`UsedShmemSegAddr`、`PostmasterPid`、`MyPMChildSlot`、client socket | exec 后必须恢复的最小启动参数。 |
| physical shmem reattach | `PGSharedMemoryReAttach()` | 把 main shared memory segment 映射到 postmaster 使用的地址。 |
| shmem callback attach | `ShmemAttachRequested()`、各子系统 `attach_fn` | 把当前进程的 C 全局指针变量重新指到 shared memory 对象，并初始化 per-backend 辅助状态。 |
本节所有源码细节都围绕这张表展开。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/postmaster.c` | `ServerLoop()` 接收 socket，`BackendStartup()` 分配 `PMChild`、构造 `BackendStartupData`、调用 `postmaster_child_launch()`。 |
| 2 | `src/backend/postmaster/launch_backend.c` | fork 与 `EXEC_BACKEND` 的统一 launch 层；`postmaster_child_launch()`、`internal_forkexec()`、`SubPostmasterMain()`、`save_backend_variables()`、`restore_backend_variables()`。 |
| 3 | `src/backend/postmaster/fork_process.c` | `fork_process()` 如何在 fork 前阻塞信号、刷新 stdio，并在 child 中设置 `MyProcPid` 和随机数状态。 |
| 4 | `src/include/tcop/backend_startup.h` | `BackendStartupData`、`CAC_state`、`ConnectionTiming`。 |
| 5 | `src/backend/tcop/backend_startup.c` | `BackendMain()`、`BackendInitialize()`、startup packet、早期错误、`InitProcess()` 调用边界。 |
| 6 | `src/backend/utils/init/miscinit.c` | `InitPostmasterChild()` 初始化 postmaster child 的本地运行环境、signal mask、local latch、postmaster death 监测。 |
| 7 | `src/backend/storage/ipc/ipci.c` | `CreateSharedMemoryAndSemaphores()` 与 `AttachSharedMemoryStructs()`，区分 postmaster 初始化和 `EXEC_BACKEND` attach。 |
| 8 | `src/backend/storage/ipc/shmem.c` | `ShmemCallRequestCallbacks()`、`ShmemInitRequested()`、`ShmemAttachRequested()` 的状态机。 |
| 9 | `src/backend/storage/lmgr/proc.c` | `InitProcess()` 分配 `PGPROC`，在 `EXEC_BACKEND` 下完成 `AttachSharedMemoryStructs()`，退出时 `ProcKill()` 释放。 |
| 10 | `src/backend/storage/ipc/pmsignal.c` | `PMChildFlags`、`RegisterPostmasterChildActive()`、`MarkPostmasterChildInactive()`。 |
| 11 | `src/backend/port/sysv_shmem.c` | Unix-like `EXEC_BACKEND` 下 `PGSharedMemoryReAttach()` 如何按原地址 attach System V shmem。 |
| 12 | `src/backend/port/win32_shmem.c` | Windows `PGSharedMemoryReAttach()` 和 `pgwin32_ReserveSharedMemoryRegion()` 的地址预留。 |
| 13 | `src/backend/tcop/postgres.c` | `PostgresMain()` 的 early initialization、`BaseInit()`、cancel key、`InitPostgres()`、顶层错误恢复。 |
| 14 | `src/backend/utils/init/postinit.c` | `BaseInit()` 和 `InitPostgres()` 中 `InitProcessPhase2()`、认证、database attach 的后续边界。 |
| 15 | `src/include/postmaster/proctypelist.h` | 哪些 postmaster child 类型需要 `shmem_attach`，以及 `B_BACKEND` / `B_DEAD_END_BACKEND` 都进入 `BackendMain()`。 |
推荐阅读顺序：

```text
ServerLoop accept
  -> BackendStartup child slot and startup_data
  -> postmaster_child_launch fork or exec
  -> child InitPostmasterChild
  -> BackendMain / BackendInitialize
  -> InitProcess / AttachSharedMemoryStructs
  -> PostgresMain / BaseInit / InitPostgres
  -> proc_exit / shmem_exit cleanup
```
读源码时要标出：
- 哪些代码运行在 postmaster。
- 哪些代码运行在刚 fork 出来的 child。
- 哪些代码只在 `EXEC_BACKEND` 编译条件下存在。
- 哪些状态在 child 拿到 `PGPROC` 前不能碰。
- 哪些 cleanup 只有注册了 `on_shmem_exit` 后才会发生。

## 4. 关键数据结构与状态

### 4.1 `ClientSocket` 与 client fd
`ServerLoop()` 里 `AcceptConnection()` 得到 `ClientSocket`。
postmaster 自己不处理 startup packet。
它只把 socket 交给子进程。
fork 路径中，socket fd 可以被子进程继承。
`postmaster_child_launch()` 会在 child 中复制一份 `MyClientSocket`。
postmaster 回到 `ServerLoop()` 后关闭自己那份 socket。
`EXEC_BACKEND` 路径下，socket 不能简单假设跨 exec 可用。
`save_backend_variables()` 会把 socket 写入 `BackendParameters`。
非 Windows 下 `write_inheritable_socket()` 只是保存原 socket。
Windows 下会通过 `WSADuplicateSocket()` 生成可在新进程中创建 socket 的信息。
client socket 的 ownership 是：

```text
postmaster 接收并短暂持有
  -> launch 层传给 child
  -> child 的 BackendInitialize() 用 pq_init() 包装成 Port
  -> postmaster 关闭自己的 fd
```

### 4.2 `BackendStartupData` 与 `CAC_state`
`BackendStartupData` 是 postmaster 传给 `BackendMain()` 的小型启动包。
它包含：
- `canAcceptConnections`。
- `socket_created`。
- `fork_started`。
`CAC_state` 不是认证结果。
它描述 postmaster 在 accept 时刻看到的实例状态。
| 值 | 含义 |
| --- | --- |
| `CAC_OK` | 可以继续处理 startup packet 和认证。 |
| `CAC_STARTUP` | 数据库系统仍在启动。 |
| `CAC_SHUTDOWN` | 系统正在关闭。 |
| `CAC_RECOVERY` | 恢复模式不接受普通连接。 |
| `CAC_NOTHOTSTANDBY` | standby 还不能接受 hot standby 查询。 |
| `CAC_TOOMANY` | 没有普通 backend child slot，或连接数超限。 |
postmaster 不直接向 client 讲完整 backend 协议。
它创建 child。
child 在 `BackendInitialize()` 中读取 startup packet，并用协议格式返回错误。
如果没有 child slot，还会启动 `B_DEAD_END_BACKEND`。
dead-end backend 仍进入 `BackendMain()`，但 `CAC_TOOMANY` 会让它早期报错后退出。

### 4.3 `PMChild`、child slot 与 `PMChildFlags`
`PMChild` 是 postmaster 私有的子进程登记项。
`AssignPostmasterChildSlot(B_BACKEND)` 从对应 pool 拿 entry。
它的 `child_slot` 对应 shared memory 中 `PMSignalState->PMChildFlags[]` 的槽。
状态线是：

```text
PM_CHILD_UNUSED
  -> postmaster AssignPostmasterChildSlot()
  -> PM_CHILD_ASSIGNED
  -> child InitProcess() / InitAuxiliaryProcess()
  -> RegisterPostmasterChildActive()
  -> PM_CHILD_ACTIVE
  -> normal shmem_exit
  -> MarkPostmasterChildInactive()
  -> PM_CHILD_ASSIGNED
  -> postmaster ReleasePostmasterChildSlot()
  -> PM_CHILD_UNUSED
```
这个状态服务 crash policy。
如果 child 死时还处于 `ACTIVE`，postmaster 能判断它可能没有干净 detach shared memory。
它不是事务状态。
它也不是 `PGPROC`。

### 4.4 `BackendParameters`
`BackendParameters` 只存在于 `EXEC_BACKEND` 路径。
它模拟 fork 会继承的那部分必要状态。
它保存：
- `client_sock` 和可继承 socket 信息。
- `DataDir`。
- `MyPMChildSlot`。
- `UsedShmemSegID` / `UsedShmemSegAddr`。
- Windows 下的 `ShmemProtectiveRegion`。
- `ProcGlobal`、`AuxiliaryProcs`、`PreparedXactProcs`。
- `PMSignalState`、`ProcSignal`。
- `PostmasterPid`。
- 启动时间和 reload 时间。
- syslogger pipe。
- `my_exec_path`、`pkglib_path`。
- `MaxBackends`、`num_pmchild_slots`。
- `startup_data`。
这些字段只是让 `SubPostmasterMain()` 能继续 bootstrap。
它们不是完整 backend 内存镜像。
尤其要注意：

```text
BackendParameters 恢复 ProcGlobal 指针，
但不会恢复每个子系统自己的全局指针和 callback 函数地址。
```
后者必须靠重新注册 callbacks、重新加载 preload libraries、`ShmemCallRequestCallbacks()` 和 `ShmemAttachRequested()`。

### 4.5 `UsedShmemSegID` 与 `UsedShmemSegAddr`
`UsedShmemSegID` 标识 postmaster 创建的 main shared memory segment。
`UsedShmemSegAddr` 是 postmaster 映射该 segment 的虚拟地址。
fork 路径中，child 继承同一个映射。
`EXEC_BACKEND` 路径中，child 必须重新把 segment 映射到同一个地址。
`PGSharedMemoryReAttach()` 的不变量是：

```text
reattach 后得到的 header 地址必须等于 postmaster 传下来的 UsedShmemSegAddr。
```
如果地址不一致，源码会 `elog(FATAL)`。
Windows 路径更复杂。
postmaster 在 `CreateProcess()` 后、`ResumeThread()` 前调用 `pgwin32_ReserveSharedMemoryRegion()`。
它在 child 进程地址空间预留 `ShmemProtectiveRegion` 和 `UsedShmemSegAddr`。
如果 ASLR 或安全软件导致预留失败，会终止该 child 并重试，最多 100 次。

### 4.6 shmem callbacks 与 pending requests
`registered_shmem_callbacks` 是当前进程私有 list。
postmaster 启动期会 `RegisterBuiltinShmemCallbacks()`。
`shared_preload_libraries` 的 `_PG_init()` 也可能注册 callbacks。
fork 路径中，这个 list 的内容会被子进程继承。
`EXEC_BACKEND` 路径中，exec 后函数指针地址不能继承。
因此 `SubPostmasterMain()` 必须：
- `RegisterBuiltinShmemCallbacks()`。
- `process_shared_preload_libraries()`。
- `InitShmemAllocator(UsedShmemSegAddr)`。
- `ShmemCallRequestCallbacks()`。
`pending_shmem_requests` 不是全局 registry。
它只是当前进程这次 attach 的计划：

```text
我要找哪些 named shmem object；
找到后把地址写回哪个本地全局变量；
最后调用哪些 attach_fn。
```
真正跨进程存在的 registry 是 shared memory 中的 `ShmemIndex`。

### 4.7 `MyProc`、`MyProcNumber` 与 `PGPROC`
`MyProc` 是当前进程的 backend-local 全局变量。
它指向 shared memory 里的 `PGPROC` slot。
`MyProcNumber` 是该 slot 在 `ProcGlobal->allProcs[]` 中的编号。
`InitProcess()` 做：
- 从合适 freelist 领取 `PGPROC`。
- 设置 `MyProc` / `MyProcNumber`。
- 初始化等待、锁、事务状态字段。
- `OwnLatch(&MyProc->procLatch)`。
- `SwitchToSharedLatch()`。
- `pgstat_set_wait_event_storage(&MyProc->wait_event_info)`。
- `on_shmem_exit(ProcKill, 0)`。
- `InitLWLockAccess()`。
- `InitDeadLockChecking()`。
- `EXEC_BACKEND` 下调用 `AttachSharedMemoryStructs()`。
没有 `PGPROC`，backend 不能安全进入 LWLock wait queue。
没有 LWLock access，`EXEC_BACKEND` 又不能安全扫描 `ShmemIndex`。
因此顺序是：

```text
先拿 PGPROC
  -> 才能初始化 LWLock access
  -> 才能 ShmemAttachRequested()
  -> 才能 ProcArrayAdd()
```

### 4.8 `Port`、`MyProcPort` 与 `ProcessingMode`
`BackendInitialize()` 调用 `pq_init(client_sock)` 创建 `Port`。
结果保存在 `MyProcPort`。
`Port` 及其附属字符串在 `TopMemoryContext` 下。
它要活过 `BackendInitialize()` 并进入 `PostgresMain()` 和 `InitPostgres()`。
backend 进入 `PostgresMain()` 时仍处于 `InitProcessing`。
`PostgresMain()` 完成 `BaseInit()` 和 `InitPostgres()` 后才：

```text
SetProcessingMode(NormalProcessing)
```
已经有 OS process、socket、Port、PGPROC，不等于已经进入正常 SQL 执行状态。

## 5. 主流程源码 walkthrough

### 5.1 postmaster idle loop 接收连接
主入口在 `postmaster.c`：

```text
ServerLoop()
  -> WaitEventSetWait(pm_wait_set, ...)
  -> WL_SOCKET_ACCEPT
  -> AcceptConnection(fd, &s)
  -> BackendStartup(&s)
  -> closesocket(s.sock) in postmaster
```
postmaster 的职责很窄。
它做 accept、状态判断、子进程创建和监督。
它不做认证。
它不解析 SQL。
它不进入普通 backend 的 MemoryContext / ResourceOwner 生命周期。
这条边界保证 postmaster 尽量小而稳定。

### 5.2 `BackendStartup()` 分配 child slot
`BackendStartup()` 先记录：

```text
startup_data.socket_created = GetCurrentTimestamp()
```
然后检查：

```text
cac = canAcceptConnections(B_BACKEND)
```
如果 `CAC_OK`，尝试：

```text
bn = AssignPostmasterChildSlot(B_BACKEND)
```
如果没有 slot，`cac` 被置为 `CAC_TOOMANY`。
之后可能走：

```text
bn = AllocDeadEndChild()
```
这说明 too many clients 时仍可能 fork 一个 dead-end backend。
它的任务是用协议格式告诉 client 错误。
接着：

```text
startup_data.canAcceptConnections = cac
pid = postmaster_child_launch(...)
```
父进程成功后只记录：

```text
bn->pid = pid
```
失败时释放 child slot，并尝试 `report_fork_failure_to_client()`。
这段失败报告不能依赖 backend libpq，因为 backend 还没起来。

### 5.3 非 `EXEC_BACKEND`：fork 后进入 child main
普通 Unix-like 路径：

```text
postmaster_child_launch()
  -> fork_process()
  -> child:
       MyBackendType = child_type
       ClosePostmasterPorts(...)
       InitPostmasterChild()
       optionally detach shared memory if this child does not need it
       MemoryContextSwitchTo(TopMemoryContext)
       MyPMChildSlot = child_slot
       MyClientSocket = copy(client_sock)
       child_process_kinds[child_type].main_fn(startup_data, startup_data_len)
```
`fork_process()` fork 前会：
- `fflush(NULL)`，避免 stdio buffer 被父子双写。
- 阻塞信号。
child 中会：
- `MyProcPid = getpid()`。
- 可选恢复 profiling timer。
- 可选调整 Linux OOM score。
- `pg_strong_random_init()`。
parent 中会恢复 signal mask。
非 `EXEC_BACKEND` 下，下列状态来自 fork inherited process image：
- `ProcGlobal`、`PMSignalState`、`ShmemIndex` 相关全局指针值。
- main shared memory segment 映射。
- `registered_shmem_callbacks` list。
- `shared_preload_libraries` 已加载代码和静态变量。
- GUC 基础状态。
- postmaster alive pipe fd。
- syslogger pipe fd。
但 child 仍必须：
- `ClosePostmasterPorts()`。
- `on_exit_reset()`。
- 重新初始化 local latch / wait event support。
- 重新设置信号处理。
- 后续删除不需要的 `PostmasterContext`。
fork 继承的是初始材料，不是无需清理的完整环境。

### 5.4 `EXEC_BACKEND` parent：保存参数并启动新进程
`EXEC_BACKEND` 下调用：

```text
internal_forkexec(child_type, child_slot, startup_data, startup_data_len, client_sock)
```
非 Windows 测试路径：

```text
save_backend_variables(param, ...)
write param to temporary backend_var file
argv[1] = "--forkchild=<kind>"
argv[2] = tmpfilename
fork_process()
child: execv(postgres_exec_path, argv)
parent: return pid
```
Windows 路径：

```text
CreateFileMapping()
MapViewOfFile()
CreateProcess(..., CREATE_SUSPENDED, ...)
save_backend_variables(param, child process handle, child pid, ...)
pgwin32_ReserveSharedMemoryRegion(child process)
ResumeThread()
pgwin32_register_deadchild_callback()
```
两个关键 fallback：
- 参数保存失败，终止未启动或半启动 child。
- Windows 地址预留失败，终止 child 并重试，最多 100 次。
`save_backend_variables()` 保存的是 exec 后继续 bootstrap 所需的最小状态。
它不能保存函数指针语义。
这决定了后续必须重新加载库和注册 callbacks。

### 5.5 `SubPostmasterMain()` 恢复到接近 fork child 的环境
exec 后新进程从 `SubPostmasterMain()` 进入。
主链路是：

```text
SubPostmasterMain(argc, argv)
  -> IsPostmasterEnvironment = true
  -> whereToSendOutput = DestNone
  -> InitializeGUCOptions()
  -> parse --forkchild=<kind>
  -> MyBackendType = child_type
  -> read_backend_variables()
       -> restore_backend_variables()
  -> ClosePostmasterPorts(...)
  -> InitPostmasterChild()
  -> PGSharedMemoryReAttach() or PGSharedMemoryNoReAttach()
  -> read_nondefault_variables()
  -> checkDataDir()
  -> LocalProcessControlFile(false)
  -> RegisterBuiltinShmemCallbacks()
  -> process_shared_preload_libraries()
  -> InitShmemAllocator(UsedShmemSegAddr)
  -> ShmemCallRequestCallbacks()
  -> child_process_kinds[child_type].main_fn(startup_data, startup_data_len)
```
这时还没有调用 `ShmemAttachRequested()`。
`SubPostmasterMain()` 只做到：
- 物理 shmem segment 已映射。
- shmem allocator 能找到。
- 当前进程知道要 attach 哪些 named objects。
- callbacks 已重新注册。
它还没有：
- 领取 `PGPROC`。
- 初始化 LWLock access。
- 把所有子系统全局指针接上。
- 加入 `ProcArray`。
为什么不在这里直接 `ShmemAttachRequested()`？
因为 `ShmemAttachRequested()` 要读 `ShmemIndex`。
读 `ShmemIndex` 要拿 `ShmemIndexLock`。
拿 LWLock 需要 `MyProc`。
所以必须等 `InitProcess()` 分配 `PGPROC` 后。

### 5.6 `BackendMain()` 是连接 backend 的统一入口
无论 fork 还是 `EXEC_BACKEND`，客户端 backend 都进入：

```text
BackendMain(const void *startup_data, size_t startup_data_len)
```
它先检查：

```text
startup_data_len == sizeof(BackendStartupData)
MyClientSocket != NULL
```
`EXEC_BACKEND` 下，如果启用 SSL，还需要在 child 中重新初始化 SSL。
原因是 SSL context 里可能有函数指针，不能通过参数文件继承。
随后：

```text
BackendInitialize(MyClientSocket, bsdata->canAcceptConnections)
InitProcess()
MemoryContextSwitchTo(TopMemoryContext)
PostgresMain(MyProcPort->database_name, MyProcPort->user_name)
```
`BackendInitialize()` 在 `InitProcess()` 之前。
因此读取 startup packet、SSL/GSS 协商、解析用户名和数据库名、基于 `CAC_state` 拒绝连接，都发生在没有 `PGPROC` 的阶段。

### 5.7 `BackendInitialize()` 的早期无 shared memory 区
`BackendInitialize()` 注释明确说它不依赖 shared memory。
`EXEC_BACKEND` 下，进程可能已经物理 attach shared memory。
但物理 attach 不等于 shared memory structures 可用。
此时多数本地指针还没接好。
`BackendInitialize()` 做：
- `ReserveExternalFD()` 记录 client socket 是长期 fd。
- `PreAuthDelay` 供调试早期认证。
- `ClientAuthInProgress = true`。
- `pq_init(client_sock)` 创建 `MyProcPort`。
- `whereToSendOutput = DestRemote`。
- 设置 startup packet 阶段的 SIGTERM 和 timeout 行为。
- `ProcessSSLStartup()`。
- `ProcessStartupPacket()`。
- 根据 `CAC_state` 报 FATAL。
- 关闭 startup packet timeout。
- `check_on_shmem_exit_lists_are_empty()`。
- 若 startup packet 是 cancel request 或坏包，`proc_exit(0)`。
- 设置 ps title。
早期 SIGTERM 和 timeout 的处理是：

```text
process_startup_packet_die()
  -> _exit(1)
StartupPacketTimeoutHandler()
  -> _exit(1)
```
之所以可以直接 `_exit()`，是因为还没有注册 shared memory cleanup，也没有修改 shared memory。
这是本节最重要的错误路径不变量。

### 5.8 startup packet 与 cancel request
`ProcessStartupPacket()` 读取包长、协议版本或特殊请求码。
特殊请求包括：
- cancel request。
- SSL request。
- GSS request。
cancel request 调用：

```text
ProcessCancelRequestPacket()
  -> SendCancelRequest(backendPID, cancelAuthCode, len)
```
然后返回失败状态，使 `BackendInitialize()` 停止继续启动普通 session。
一个被 fork 出来的 `B_BACKEND` 进程，不一定最终成为 SQL session。
它可能只是处理 cancel request。
这种进程不会进入 `InitProcess()`。
因此不会有 `MyProc`。
也不会出现在 `pg_stat_activity` 中。

### 5.9 `InitProcess()`：从 OS process 进入 shared identity
`BackendInitialize()` 成功后才调用：

```text
InitProcess()
```
主链路是：

```text
InitProcess()
  -> RegisterPostmasterChildActive()
  -> choose ProcGlobal freelist
  -> pop PGPROC under freeProcsLock
  -> MyProc = ...
  -> MyProcNumber = GetNumberFromPGProc(MyProc)
  -> initialize wait / xid / lock / latch / status fields
  -> OwnLatch(&MyProc->procLatch)
  -> SwitchToSharedLatch()
  -> pgstat_set_wait_event_storage(&MyProc->wait_event_info)
  -> PGSemaphoreReset(MyProc->sem)
  -> on_shmem_exit(ProcKill, 0)
  -> InitLWLockAccess()
  -> InitDeadLockChecking()
  -> EXEC_BACKEND: AttachSharedMemoryStructs()
```
这一步之后，backend 才具备：
- 被其它 backend 通过 `PGPROC` 识别。
- 进入 LWLock wait queue。
- 发布 wait event。
- 退出时通过 `ProcKill()` 释放 PGPROC slot。
- 在 postmaster child slot 中标记 active 使用 shared memory。
如果这里没有可用 `PGPROC`，会 FATAL。
这和 postmaster 阶段的 `CAC_TOOMANY` 不是同一个检测点。

### 5.10 `AttachSharedMemoryStructs()`：`EXEC_BACKEND` 的本地指针接线
`AttachSharedMemoryStructs()` 只在 `EXEC_BACKEND` 下编译。
它断言：

```text
MyProc != NULL
IsUnderPostmaster
```
主链路是：

```text
AttachSharedMemoryStructs()
  -> InitializeFastPathLocks()
  -> ShmemAttachRequested()
  -> shmem_startup_hook()
```
`ShmemAttachRequested()` 做：

```text
Assert(shmem_request_state == SRS_REQUESTING)
shmem_request_state = SRS_ATTACHING
LWLockAcquire(ShmemIndexLock, LW_SHARED)
foreach pending request:
    AttachShmemIndexEntry(request, false)
foreach registered callback:
    callbacks->attach_fn(...)
LWLockRelease(ShmemIndexLock)
shmem_request_state = SRS_DONE
```
这一步重新建立的是当前进程私有指针。
普通 fork backend 不走这一步。
它已经通过 fork 继承了物理 mapping 和指针。
但不要说 fork backend “没有 attach shared memory”。
它只是没有重新执行 `ShmemAttachRequested()`。

### 5.11 `PostgresMain()`：进入主循环前的最后初始化
`BackendMain()` 之后进入：

```text
PostgresMain(dbname, username)
```
早期流程是：

```text
PostgresMain()
  -> set backend signal handlers
  -> InitializeTimeouts()
  -> BaseInit()
  -> unblock SIGINT etc
  -> generate cancel key
  -> InitPostgres(dbname, username, flags, ...)
  -> delete PostmasterContext
  -> SetProcessingMode(NormalProcessing)
  -> BeginReportingGUCOptions()
  -> pgstat_report_connect(MyDatabaseId)
  -> send cancellation info
  -> ReadyForQuery loop
```
`BaseInit()` 断言 `MyProc != NULL`。
`InitPostgres()` 一开始会调用 `InitProcessPhase2()`。
`InitProcessPhase2()` 把 `MyProc` 加入 `ProcArray`。
拆成 phase 2 的原因是：

```text
InitProcess() 之前不能拿 LWLock；
EXEC_BACKEND 下 ProcArrayAdd() 又必须等 AttachSharedMemoryStructs() 后才可用。
```
因此 backend identity 的生命周期是三段：

```text
InitProcess:
  领取 PGPROC，能等待和访问低层 shared memory。
AttachSharedMemoryStructs:
  EXEC_BACKEND 下重建本地 shared pointers。
InitProcessPhase2:
  加入 ProcArray，成为 snapshot / transaction visibility scan 的成员。
```

## 6. 生命周期 / ownership / cleanup
postmaster 持有：
- 监听 socket。
- 短暂 accept 出来的 client socket。
- `PMChild` entry。
- child pid。
- child slot 的 postmaster 侧状态。
postmaster 不持有：
- `MyProcPort`。
- 当前 session 的 MemoryContext。
- 当前 session 的 ResourceOwner。
- 当前 backend 的 PGPROC slot。
accept 后，postmaster 把 client socket 传给 child。
`ServerLoop()` 之后关闭自己那份 socket。
launch 失败时，postmaster 释放 child slot。
fork 失败时，postmaster 尝试给 client 发送简化错误包。
`PGPROC` 由 postmaster 启动期在 shared memory 中一次性创建。
普通 backend 在 `InitProcess()` 中领取。
退出时由 `on_shmem_exit(ProcKill, 0)` 释放。
`ProcKill()` 会：
- `SyncRepCleanupAtProcExit()`。
- `LWLockReleaseAll()`。
- `WaitLSNCleanup()`。
- `ConditionVariableCancelSleep()`。
- `SwitchBackToLocalLatch()`。
- `DisownLatch(&MyProc->procLatch)`。
- 处理 lock group membership。
- `pgstat_reset_wait_event_storage()`。
- `MyProc = NULL`。
- `MyProcNumber = INVALID_PROC_NUMBER`。
- `proc->pid = 0`。
- 清理 `vxid`。
- 把 slot 放回对应 freelist。
`DisownLatch()` 必须早于 PGPROC 回到 freelist。
否则新 backend 拿到同一个 slot 并 `OwnLatch()` 时可能发现 latch 仍属于旧进程。
`pq_init()` 在 `TopMemoryContext` 下创建 `Port`。
它要活到 `PostgresMain()` 和 `InitPostgres()`。
`PostmasterContext` 是 fork 继承或启动期携带的上下文。
`PostgresMain()` 在 `InitPostgres()` 完成后删除它。
这说明 fork 继承来的父进程内存不是 session 长期 ownership。
早期 `_exit()` 与 later `proc_exit()` 的边界是：

```text
startup packet 前：
  没有 PGPROC，没有 shared memory cleanup，可以 _exit。
InitProcess 后：
  已 active 使用 shared memory，必须走 shmem_exit / ProcKill。
```
这条分界对诊断非常关键。
连接卡在 startup packet 阶段，可能完全没有 `pg_stat_activity` 记录。
连接已经 `InitProcess()`，则应该能在 shared-memory 观测入口看到部分痕迹。

## 7. 正确性机制层次
第一层是 signal mask ordering。
postmaster fork child 时先阻塞信号。
child 调用 `InitPostmasterChild()` 和 `PostgresMain()` 设置自己的 handler。
之后才逐步解除阻塞。
这避免 child 在 backend handler 未安装前运行 postmaster handler。
第二层是 fd ownership。
child 要关闭 postmaster 的监听 socket。
postmaster 要关闭自己那份 client socket。
否则 fd 生命周期、连接关闭语义和监督边界都会变复杂。
第三层是 physical shmem attach vs logical pointer attach。
`PGSharedMemoryReAttach()` 只保证 main segment 映射到正确地址。
`ShmemAttachRequested()` 才把当前进程的 C 全局变量接到 named shared object。
fork 路径中，两者都由继承隐式满足。
第四层是 callback function pointer 不能跨 exec。
`registered_shmem_callbacks` 保存函数指针。
exec 后库可能装载到不同地址。
因此 `SubPostmasterMain()` 必须重新注册内置 callbacks，并重新加载 `shared_preload_libraries`。
第五层是 `PGPROC` 早于 LWLock access。
LWLock wait queue 需要等待者身份。
等待者用 `PGPROC` / `ProcNumber` 表示。
所以 `InitProcess()` 必须先拿 `PGPROC`，再 `InitLWLockAccess()`，最后才能 `ShmemAttachRequested()`。
第六层是 `ProcArrayAdd()` 晚于 shmem attach。
`InitProcessPhase2()` 在 `InitPostgres()` 早期执行。
在 `EXEC_BACKEND` 下，如果还没完成 `AttachSharedMemoryStructs()`，ProcArray 相关本地指针可能不可用。
第七层是 startup packet 前不改 shared memory。
`BackendInitialize()` 早期的 `_exit()` 依赖这个前提。
源码用 `check_on_shmem_exit_lists_are_empty()` 自检这个边界。

## 8. 错误路径 / 异常路径 / fallback
`canAcceptConnections()` 可能返回非 `CAC_OK`。
postmaster 仍会尝试启动 child。
child 在 `BackendInitialize()` 读完 startup packet 后发 FATAL。
好处是 client 得到协议格式错误，postmaster 不需要参与 backend 协议。
fork 失败时，`BackendStartup()`：
- 保存 `errno`。
- `ReleasePostmasterChildSlot(bn)`。
- 记录 LOG。
- `report_fork_failure_to_client()`。
`report_fork_failure_to_client()` 设置 socket nonblocking，只尝试发送一次。
它不允许 postmaster 因一个 client socket 阻塞。
`EXEC_BACKEND` 参数传递可能失败在：
- 创建临时 backend variables 文件。
- 写文件。
- `FreeFile()`。
- `execv()`。
- `CreateProcess()`。
- socket / handle duplication。
- shared memory address reservation。
- `ResumeThread()`。
这些失败多发生在 child 真正进入 backend main 前。
postmaster 负责清理半启动进程和句柄。
`PGSharedMemoryReAttach()` 失败会 FATAL。
典型原因包括：
- segment id / handle 无效。
- 无法映射到指定地址。
- 返回地址与 expected address 不一致。
- Windows 返回的 header magic 不对。
这不是可降级错误。
startup packet 阶段：

```text
SIGTERM -> _exit(1)
timeout -> _exit(1)
SIGQUIT -> crash-exit handler
```
如果 client 打开 TCP 连接后不发 startup packet，backend 会消耗一个 OS process，但不会占用 `PGPROC`。
这类现象在 `pg_stat_activity` 中通常看不到。
cancel request 由一个短命 backend 处理。
它读取 cancel packet 后调用 `SendCancelRequest()`。
然后 `BackendInitialize()` 因 `status != STATUS_OK` 调用 `proc_exit(0)`。
它不会继续 `InitProcess()`。
`InitProcess()` 没有 `PGPROC` 时会 FATAL。
普通 backend 报 `sorry, too many clients already`。
WAL sender 报 max_wal_senders 相关错误。
这个失败点发生在 startup packet 成功之后、`PostgresMain()` 之前。
一旦 `RegisterPostmasterChildActive()` 成功，child slot 变成 active。
如果进程异常死亡，`MarkPostmasterChildInactive()` 可能来不及执行。
postmaster 回收 child slot 时会发现 child 没有干净 detach。
这和 pre-shmem startup failure 是两类完全不同的事件。

## 9. 成本、资源与跨模块传播
非 `EXEC_BACKEND` 路径的优势是 fork 继承。
大量只读代码、预加载库和基础状态通过 copy-on-write 共享。
`EXEC_BACKEND` 路径成本更高：
- 创建新进程或 fork 后 exec。
- 传递 `BackendParameters`。
- 重新初始化 GUC 基础状态。
- 重新读取 nondefault variables。
- 重新检查 data directory。
- 重新读取 control file。
- 重新加载 preload libraries。
- 重新注册 shmem callbacks。
- 重新 attach shared pointers。
- 可能重新初始化 SSL。
连接启动相关资源随这些变量放大：
| 变量 | 影响 |
| --- | --- |
| 连接建立速率 | fork/exec、DNS、SSL、auth、GUC 初始化被放大。 |
| `MaxBackends` | PGPROC、semaphore、child slot、ProcArray 等 shared memory 上限。 |
| `max_connections` | 普通 backend slot、内存和进程数上限。 |
| `max_wal_senders` | WAL sender freelist 和连接启动失败边界。 |
| `shared_preload_libraries` 数量 | exec 路径重新加载和 callback 注册成本。 |
| SSL/GSS 使用 | startup packet 前后的协商成本。 |
| `log_hostname` | 可能引入 reverse DNS 延迟。 |
| 认证方式 | 下一节会展开，可能引入网络、文件、PAM、LDAP、GSS 等外部等待。 |
短连接 workload 常把成本推到：
- OS process creation。
- TLS handshake。
- auth。
- catalog / relcache 初始化。
- ProcArray / lock manager 初始化。
涉及的 fd / handle 包括：
- postmaster listen socket。
- client socket。
- postmaster alive pipe。
- syslogger pipe。
- EXEC_BACKEND 参数文件或 file mapping。
- shared memory handle。
`ReserveExternalFD()` 告诉 fd.c 有长期外部 fd 被占用。
fd accounting 错误会让后续 file open 误判安全余量。
本节连接的相邻模块包括：
- postmaster supervision。
- libpq backend。
- shared memory。
- process identity。
- ProcArray。
- stats / ps display。
- signal / latch。
- memory context。
一个连接启动问题通常不能只在一个文件里解释。

## 10. 观测与诊断入口
直接可观测入口包括：
- server log 中的 `connection received`。
- `log_connections` 的 authentication / authorization / setup duration。
- `Trace_connection_negotiation` 的 SSL/GSS 协商日志。
- `ps` / `top` 中的 backend process title。
- `pg_stat_activity` 中已经进入 shared identity 和 stats 上报的 backend。
- `pg_stat_activity.wait_event`，但必须等 `MyProc` 的 wait event storage 建好。
- `DEBUG2` 的 forked backend 日志。
- `DEBUG3` 的 shared memory attach 日志。
- OS 层 `strace -f` / Process Monitor。
- gdb breakpoints。
能推断但不能直接看到的状态包括：
- backend 是否还停在 startup packet 阶段。
- child 是否已经调用 `InitProcess()`。
- `EXEC_BACKEND` 是否已完成 physical reattach。
- `ShmemAttachRequested()` 是否已完成所有 callbacks。
- `PostmasterContext` 是否已删除。
- cancel request backend 是否已经退出。
一个 client 连接已经建立，但 `pg_stat_activity` 没有记录，可能原因包括：
- 还没发 startup packet。
- 还在 SSL/GSS 协商。
- 已经因为 bad startup packet 退出。
- 只是 cancel request。
- 还没到 `InitProcessPhase2()`。
源码跟读断点：

```text
break ServerLoop
break BackendStartup
break postmaster_child_launch
break fork_process
break SubPostmasterMain
break BackendMain
break BackendInitialize
break InitProcess
break AttachSharedMemoryStructs
break ShmemAttachRequested
break PostgresMain
break InitPostgres
```
非 `EXEC_BACKEND` Linux 调试 fork child 时：

```text
set follow-fork-mode child
set detach-on-fork off
```
观察点：
- `MyBackendType`。
- `MyPMChildSlot`。
- `MyClientSocket`。
- `MyProc`。
- `MyProcNumber`。
- `IsUnderPostmaster`。
- `UsedShmemSegAddr`。
- `shmem_request_state`。
- `whereToSendOutput`。
- `ProcessingMode`。
诊断慢连接要按阶段拆：

```text
accept 等待
  -> fork/exec 创建
  -> startup packet / SSL / GSS
  -> CAC rejection
  -> InitProcess / shmem attach
  -> authentication
  -> InitPostgres database attach
  -> ready for query
```
`pg_stat_activity` 主要覆盖已经进入较后阶段的 backend。
startup packet 前的问题更多依赖日志、OS 进程观察、packet capture、gdb 或 `strace`。

## 11. 常见误区
误区 1：fork 后就已经是 PostgreSQL backend。
fork 只创建 OS process。
PostgreSQL backend identity 还需要 `InitPostmasterChild()`、`BackendInitialize()`、`InitProcess()` 和 `PostgresMain()`。
误区 2：shared memory 映射成功就等于所有共享状态可用。
`PGSharedMemoryReAttach()` 只解决 physical mapping。
`ShmemAttachRequested()` 才解决当前进程的本地指针接线。
`InitProcessPhase2()` 才让 backend 进入 `ProcArray`。
误区 3：`BackendParameters` 是完整 backend state snapshot。
它只保存 exec 后继续 bootstrap 所需的少量变量。
它不能保存有效函数指针，也不能替代 shmem callbacks。
误区 4：`pg_stat_activity` 能看到所有连接启动问题。
startup packet 前没有 `PGPROC`。
cancel request backend 不进入普通 session。
bad startup packet 可能很早退出。
误区 5：`CAC_TOOMANY` 和 `InitProcess()` 的 too-many 是同一个点。
前者是 postmaster accept 后的连接可接受性和 child slot 判断。
后者是 child 已经启动后领取 `PGPROC` 失败。
误区 6：fork 继承意味着不需要 child cleanup。
fork child 必须关闭 postmaster fd、重置信号、重置 exit handlers、设置 local latch、最终删除 `PostmasterContext`。
误区 7：可以在 startup packet 前随便访问 shared memory。
这样会破坏 `_exit()` 快速失败的不变量。
误区 8：`EXEC_BACKEND` 只是 Windows 平台细节。
它也是暴露 fork 隐式依赖的测试工具。
扩展或内核 patch 如果只在 fork 平台正常，仍可能违反跨 exec bootstrap 语义。

## 12. 课堂实验

### 实验 1：跟踪一个普通 fork backend
目标：确认 fork inherited process image 的边界。
步骤：
1. 用 debug build 启动本地 PostgreSQL。
2. 在 postmaster 上 attach gdb。
3. 设置：

```text
set follow-fork-mode child
set detach-on-fork off
break BackendStartup
break postmaster_child_launch
break fork_process
break BackendMain
break BackendInitialize
break InitProcess
break PostgresMain
```
4. 发起一个 `psql` 连接。
5. 在 child 中观察 `IsUnderPostmaster`、`MyBackendType`、`MyClientSocket`、`MyProc`、`UsedShmemSegAddr`、`ProcGlobal`。
预期现象：
- child 刚 fork 后已经有 `UsedShmemSegAddr` 和 `ProcGlobal`。
- `BackendInitialize()` 阶段 `MyProc` 仍为 NULL。
- `InitProcess()` 后 `MyProc` 和 `MyProcNumber` 有效。
- 非 `EXEC_BACKEND` 不会进入 `SubPostmasterMain()`。

### 实验 2：观察 startup packet 前的不可见连接
目标：验证“没有 `PGPROC` 就不会成为普通 `pg_stat_activity` backend”。
步骤：
1. 设置较大的 `authentication_timeout`。
2. 用 `nc` 连接 PostgreSQL 端口但不发送 startup packet。
3. 观察 `pg_stat_activity`。
4. 用 `ps` 观察 postgres child process。
5. 查看 connection receipt 日志。
预期现象：
- OS 上可能看到 backend child。
- `pg_stat_activity` 不一定出现对应正常 session。
- 超时后 child 直接退出。
源码解释：
- `BackendInitialize()` 读取 startup packet 前没有 `InitProcess()`。
- timeout handler 走 `_exit(1)`。
- 没有 shared memory cleanup，也没有 `ProcArray` membership。

### 实验 3：用 `PreAuthDelay` 定位早期边界
目标：给 gdb 留时间观察 `BackendInitialize()` 和 `InitProcess()` 之间的状态。
配置：

```conf
pre_auth_delay = 10
log_connections = 'all'
```
发起 `psql` 连接，在 delay 窗口 attach 到新 child。
观察：

```text
p ClientAuthInProgress
p MyProcPort
p MyProc
p whereToSendOutput
```
预期现象：
- `MyProcPort` 可能已经建立。
- `MyProc` 仍未建立。
- `whereToSendOutput` 后续会变成 `DestRemote`。

### 实验 4：源码阅读对照 `EXEC_BACKEND`
目标：画出 fork 和 exec 两条路径差异。
阅读并画表：

```text
postmaster_child_launch()
  fork path:
    fork_process()
    child directly calls main_fn
  EXEC_BACKEND path:
    save_backend_variables()
    fork/exec or CreateProcess()
    SubPostmasterMain()
    restore_backend_variables()
    PGSharedMemoryReAttach()
    ShmemCallRequestCallbacks()
    main_fn
```
回答：
- `UsedShmemSegAddr` 何时恢复？
- preload libraries 何时重新加载？
- callbacks 何时重新注册？
- `ShmemAttachRequested()` 为什么不在 `SubPostmasterMain()` 直接调用？
- `MyProc` 何时有效？

### 实验 5：区分两类 too-many 错误
阅读源码位置：

```text
BackendStartup()
  -> canAcceptConnections()
  -> AssignPostmasterChildSlot()
  -> CAC_TOOMANY
InitProcess()
  -> ProcGlobal freelist
  -> "sorry, too many clients already"
```
解释：
- postmaster child slot 服务监督和 crash policy。
- `PGPROC` slot 服务 shared memory identity、等待和事务状态发布。

## 13. 讨论题
1. 为什么 postmaster 不直接读取 startup packet 并认证客户端，而是 fork 出 child？
2. 在非 `EXEC_BACKEND` 路径中，哪些状态虽然来自 fork 继承，但 child 仍必须主动重置或关闭？
3. 为什么 `BackendParameters` 传递了 `ProcGlobal`，却仍然需要 `ShmemAttachRequested()`？
4. `PGSharedMemoryReAttach()` 成功后，为什么还不能认为 backend 已经完成 shared memory bootstrap？
5. 为什么 `BackendInitialize()` 在 startup packet timeout 时可以 `_exit(1)`，而 `InitProcess()` 之后不可以？
6. `InitProcess()` 为什么必须早于 `AttachSharedMemoryStructs()`，而 `InitProcessPhase2()` 又必须晚于它？
7. 如果一个扩展在 fork 平台工作、在 `EXEC_BACKEND` 下崩溃，优先检查哪些隐藏依赖？
8. 一个连接出现在 OS 进程表里但不在 `pg_stat_activity` 中，可能处于哪些阶段？
9. `PMChild` slot 和 `PGPROC` slot 分别解决什么问题？为什么不能合并成一个结构？
10. 慢连接诊断中，哪些现象能靠 SQL 视图看到，哪些必须靠日志、gdb 或 OS tracing？

## 14. 本节小结
本节核心链路是：

```text
ServerLoop()
  -> AcceptConnection()
  -> BackendStartup()
  -> postmaster_child_launch()
  -> fork_process() or internal_forkexec()
  -> BackendMain()
  -> BackendInitialize()
  -> InitProcess()
  -> EXEC_BACKEND: AttachSharedMemoryStructs()
  -> PostgresMain()
  -> InitPostgres()
```
fork 路径中，很多状态来自 inherited process image：
- 已映射 main shared memory。
- postmaster 已设置的 shared pointer。
- preload libraries。
- callback list。
- fd / pipe / signal mask 的初始材料。
但 child 仍要重新建立自己的运行环境：
- 关闭 postmaster fd。
- 初始化 postmaster child 状态。
- 创建 `Port`。
- 读取 startup packet。
- 领取 `PGPROC`。
- 设置 signal handlers。
- 进入 `PostgresMain()`。
`EXEC_BACKEND` 路径中，必须显式恢复和重新 attach：
- `BackendParameters` 恢复最小全局变量和 socket。
- `PGSharedMemoryReAttach()` 恢复物理 shmem mapping。
- `RegisterBuiltinShmemCallbacks()` 和 `process_shared_preload_libraries()` 恢复 callback 函数地址。
- `ShmemCallRequestCallbacks()` 重建 attach 计划。
- `InitProcess()` 后的 `AttachSharedMemoryStructs()` 才把本地 C 全局指针接到 shared memory 对象。
生命周期边界是：

```text
startup packet 前：
  不碰 shared memory，超时或 SIGTERM 可以 _exit。
InitProcess 后：
  已 active 使用 shared memory，必须通过 shmem_exit / ProcKill / child slot cleanup 收尾。
InitProcessPhase2 后：
  进入 ProcArray，开始参与 snapshot 和事务状态发布。
```
错误路径的核心判断：
- fork 失败由 postmaster 报告并释放 child slot。
- `CAC_TOOMANY` 可以通过 dead-end backend 协议化返回。
- startup packet timeout 早期 `_exit()`。
- shmem reattach 或 pointer attach 失败是 FATAL。
- `PGPROC` 分配失败发生在 child 已启动后。
- active child 异常死亡可能触发 postmaster crash policy。
观测上要记住：
- `pg_stat_activity` 不是连接启动全过程的真相。
- startup packet 前、cancel request、bad packet 可能只在日志或 OS 进程层可见。
- `log_connections`、`Trace_connection_negotiation`、`PreAuthDelay`、gdb 断点、OS tracing 是连接 bootstrap 诊断的主要入口。
可迁移规律：

```text
进程模型中的“继承”只能传递初始材料；
一旦状态跨进程可见，就必须有明确的 attach、identity、ownership 和 cleanup 边界。
```
判断任何 backend startup bug 时，先问：

```text
这个状态到底来自继承、参数恢复、物理映射、逻辑 attach，还是后续 InitPostgres？
```
答清这个问题，才能避免把连接启动、shared memory、认证、事务可见性和 crash cleanup 混成一团。
