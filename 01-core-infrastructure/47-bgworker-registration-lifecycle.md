# PostgreSQL background worker 注册生命周期
## 课程定位
前置知识：已经理解 shared memory request/init/attach、`PGPROC` 与 `ProcArray` membership、latch 的进程级异步唤醒，以及 postmaster 作为进程监督者的基本职责。
本节唯一主问题：
```text
background worker 如何注册、启动、重启、连接数据库，并接入 latch / signal / shared memory？
```
核心矛盾：
```text
扩展和内核子系统需要一个通用入口来运行长期后台代码；
postmaster 又必须在不信任普通 backend 写入的 shared memory、
不持有普通 shared memory lock、不执行用户代码的前提下，
安全地接收请求、fork 进程、报告 pid、处理退出、决定是否重启并释放 slot。
```
学完后应能判断：
```text
static worker 为什么只能在 shared_preload_libraries 阶段注册；
dynamic worker 为什么要通过 shared slot + PMSIGNAL 交给 postmaster；
in_use、pid、generation、terminate 分别表达生命周期哪一层；
fork 成功、worker started、database connected、application ready 为什么不是同一件事；
worker main 中 signal handler、BackgroundWorkerUnblockSignals()、WaitLatch() 应如何组合；
exit code、bgw_restart_time、terminate flag 如何决定重启和 cleanup；
哪些状态能从 pg_stat_activity / wait event / log 看到，哪些只能断点或从模块协议推断。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
前面的基础设施课程分别讲过：
```text
shared memory:
  postmaster 启动期声明固定大小区域，backend 后续 attach。
PGPROC / ProcArray:
  backend 进入 shared memory 协作前必须先取得进程身份。
Latch:
  后台循环用 sticky wakeup bit 等待 signal、postmaster death、timeout 和业务通知。
ParallelContext:
  parallel worker 把 dynamic background worker 当作执行计划的一部分。
```
本节把这些机制串成一条完整 worker 生命周期：
```text
register descriptor
  -> shared slot 或 postmaster private list
  -> postmaster state change
  -> StartBackgroundWorker()
  -> postmaster_child_launch()
  -> BackgroundWorkerMain()
  -> InitProcess()
  -> extension / internal entrypoint
  -> optional BackgroundWorkerInitializeConnection()
  -> latch + signal driven main loop
  -> proc_exit()
  -> CleanupBackend()
  -> restart or ForgetBackgroundWorker()
```
本节不展开普通客户端连接认证，也不展开某个具体 worker 的业务 SQL。
重点是 registration lifecycle 的状态边界。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
static worker 在 postmaster 加载 shared_preload_libraries 时写入 BackgroundWorkerList；
dynamic worker 由普通 backend 把 BackgroundWorker descriptor 发布到 BackgroundWorkerData slot，
再通过 PMSIGNAL_BACKGROUND_WORKER_CHANGE 唤醒 postmaster；
postmaster 将 slot 内容复制到私有 RegisteredBgWorker，按 pmState 和 restart policy fork 子进程；
worker 子进程进入 BackgroundWorkerMain() 后取得 PGPROC、加载入口函数，
由用户代码决定何时连接数据库、如何等待 latch、如何处理 signal 与 shared memory 协议。
```
这里必须拆开四层状态：
```text
registration:
  系统是否登记了一个 worker descriptor，slot 是否还归这个 worker。
process liveness:
  postmaster 是否已经 fork 出 pid，这个 pid 是否仍在运行。
database attachment:
  worker 是否已经调用 InitPostgres()，拥有 databaseId、roleId、cache、sinval 和事务环境。
application readiness:
  worker 是否已经完成模块自己的 DSM attach、queue attach、shared slot 初始化或业务握手。
```
PostgreSQL 不把这些状态压成一个布尔值，因为不同失败点的 fallback 不同：
```text
slot 不足:
  RegisterDynamicBackgroundWorker() 返回 false，parallel query 可以少 worker 继续。
fork 失败:
  postmaster 记录 rw_crashed_at，稍后按 restart policy 再试。
worker ERROR:
  child 以 exit status 1 退出，postmaster 决定是否 restart。
业务 ready 失败:
  只能由 parallel error queue、test_shm_mq ready counter 或模块自己的协议判断。
```
这也是 background worker 比“开一个后台进程”复杂的原因：
```text
它既是 public extension ABI，
又是 postmaster 监督协议，
还是 shared memory / signal / latch / database session 的连接点。
```
## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/postmaster/bgworker.h` | public API、flag、start time、restart policy、handle status。 |
| 2 | `src/backend/postmaster/bgworker.c` | static/dynamic 注册、shared slot 协议、worker main、连接数据库、wait/terminate API。 |
| 3 | `src/include/postmaster/bgworker_internals.h` | postmaster 私有 `RegisteredBgWorker`。 |
| 4 | `src/backend/postmaster/postmaster.c` | `BackgroundWorkerStateChange()` 调用点、`maybe_start_bgworkers()`、`StartBackgroundWorker()`、`CleanupBackend()`。 |
| 5 | `src/backend/postmaster/launch_backend.c` | fork/EXEC_BACKEND 下如何把 startup data 送到 `BackgroundWorkerMain()`。 |
| 6 | `src/include/storage/subsystemlist.h` | `BackgroundWorkerShmemCallbacks` 进入核心 shared memory 初始化序列的位置。 |
| 7 | `src/backend/storage/ipc/shmem.c` | request/init callback 创建 `Background Worker Data` 的阶段。 |
| 8 | `src/backend/storage/lmgr/proc.c` | `InitProcess()` 给 worker 分配 `PGPROC` 并切换到 `PGPROC.procLatch`。 |
| 9 | `src/backend/utils/init/postinit.c` | `InitPostgres()` 把 worker 推进到数据库 backend 状态。 |
| 10 | `src/include/storage/latch.h` 与 `src/backend/storage/ipc/latch.c` | `MyLatch` 的等待模式、postmaster death 和 signal 唤醒。 |
| 11 | `src/backend/postmaster/interrupt.c` 与 `src/backend/tcop/postgres.c` | SIGHUP/SIGTERM handler 设置 flag 并 `SetLatch(MyLatch)`。 |
| 12 | `src/test/modules/worker_spi/worker_spi.c` | 扩展 worker 的 signal、database connection、SPI transaction、latch loop。 |
| 13 | `src/test/modules/test_shm_mq/worker.c` | dynamic worker 通过 `bgw_main_arg` attach DSM，并用 registrant latch 通知 ready。 |
| 14 | `src/backend/access/transam/parallel.c` | parallel worker 如何把 registration failure 变成少 worker fallback。 |
推荐阅读顺序：
```text
bgworker.h public contract
  -> bgworker.c shared slot 注释
  -> RegisterDynamicBackgroundWorker()
  -> BackgroundWorkerStateChange()
  -> maybe_start_bgworkers()
  -> StartBackgroundWorker()
  -> BackgroundWorkerMain()
  -> worker_spi_main()
```
不要先从某个 contrib worker 的业务逻辑读起。
业务 SQL 通常会掩盖 registration、process、database attachment 的边界。
## 4. 关键数据结构与状态
### 4.1 `BackgroundWorker`
`BackgroundWorker` 是 public registration descriptor，不是运行时状态对象。
关键字段：

| 字段 | 语义 |
| --- | --- |
| `bgw_name` | 进程显示名和日志标识。 |
| `bgw_type` | backend type 字符串，pgstat 和错误报告会使用；空值回退到 `bgw_name`。 |
| `bgw_flags` | 声明 shmem、database connection、interruptible、parallel class 能力。 |
| `bgw_start_time` | postmaster 到哪个状态后才允许启动。 |
| `bgw_restart_time` | 非零退出后的 restart delay，或 `BGW_NEVER_RESTART`。 |
| `bgw_library_name` | 入口函数所在库；核心函数使用 `"postgres"`。 |
| `bgw_function_name` | 入口函数名；跨进程传字符串，不传函数指针。 |
| `bgw_main_arg` | 一个 `Datum` 启动参数，常用于 DSM handle、slot number 或 worker number。 |
| `bgw_extra` | 固定长度字节数组，常用于 OID、flags、worker number 或小型启动包。 |
| `bgw_notify_pid` | dynamic caller 希望 start/stop 时被 SIGUSR1 唤醒的 backend pid。 |
flag 的边界：
```text
BGWORKER_SHMEM_ACCESS:
  当前源码要求所有 bgworker 都声明它。
BGWORKER_BACKEND_DATABASE_CONNECTION:
  只表示允许稍后调用 BackgroundWorkerInitializeConnection*()。
  它不会自动连接数据库。
BGWORKER_INTERRUPTIBLE:
  表示数据库冲突处理可以请求终止这个 worker。
  它要求同时有 shmem access 和 database connection。
BGWORKER_CLASS_PARALLEL:
  内部用于 parallel worker 计数，第三方 worker 不应使用。
```
`bgw_main_arg` 不是跨进程指针传送机制。
安全用法是传 DSM handle、OID、小整数、slot number。
普通 backend-local 指针在 worker 进程中没有语义。
### 4.2 `RegisteredBgWorker`
`RegisteredBgWorker` 是 postmaster 私有监督状态，定义在 `bgworker_internals.h`。
关键字段：

| 字段 | 语义 |
| --- | --- |
| `rw_worker` | postmaster 接收后的 descriptor 副本。 |
| `rw_pid` | 当前运行 pid，0 表示未运行。 |
| `rw_crashed_at` | 最近一次非零退出或启动失败时间，用于 restart delay。 |
| `rw_shmem_slot` | 对应 `BackgroundWorkerSlot` 编号。 |
| `rw_terminate` | postmaster 侧“不要再运行”的标记。 |
| `rw_lnode` | `BackgroundWorkerList` 链表节点。 |
postmaster 把 dynamic slot 内容复制到 `RegisteredBgWorker` 的原因：
```text
shared slot 是普通 backend 写入的；
postmaster 不能把它当作长期可信对象；
监督状态必须在 postmaster private memory 中维护；
用户入口函数也不能在 postmaster 中加载或执行。
```
### 4.3 `BackgroundWorkerSlot`
`BackgroundWorkerSlot` 是 `bgworker.c` 内部 shared memory slot。
它位于 `BackgroundWorkerData`，slot 数量来自 `max_worker_processes`。
字段语义：

| 字段 | 语义 |
| --- | --- |
| `in_use` | backend 与 postmaster 交接 slot ownership 的核心 flag。 |
| `terminate` | 例外字段；slot 归 postmaster 后 backend 仍可设置它请求终止。 |
| `pid` | `InvalidPid` 表示尚未启动，0 表示停止，正数表示运行中。 |
| `generation` | slot 重用计数，防止旧 handle 命中新 worker。 |
| `worker` | `BackgroundWorker` descriptor 副本。 |
`in_use` 协议：
```text
in_use == false:
  postmaster 忽略 slot，backend 持 BackgroundWorkerLock 后可初始化。
backend 初始化 slot:
  写 worker、pid、generation、terminate。
  执行 pg_write_barrier()。
  最后设置 in_use = true。
postmaster 看到 in_use:
  执行 pg_read_barrier()。
  再读 worker descriptor。
in_use == true:
  slot 归 postmaster，backend 不再改 descriptor。
  terminate 是唯一允许的例外。
```
postmaster 不拿 `BackgroundWorkerLock`。
如果 postmaster 等待普通 shared memory lock，而 shared memory 或 lock state 已坏，整个实例的监督者可能被卡住。
所以这里是 lockless handoff，backend 之间才用 LWLock 互斥。
### 4.4 `BackgroundWorkerHandle`
dynamic registration 可以返回 `BackgroundWorkerHandle`。
它只记录：
```text
slot:
  shared slot 编号。
generation:
  注册时的 slot generation。
```
`GetBackgroundWorkerPid()` 每次都重新检查：
```text
handle.generation != slot.generation:
  旧 handle 已失效，返回 BGWH_STOPPED。
!slot.in_use:
  slot 已释放，返回 BGWH_STOPPED。
slot.pid == InvalidPid:
  已注册但尚未启动，返回 BGWH_NOT_YET_STARTED。
slot.pid > 0:
  返回 BGWH_STARTED 和 pid。
```
没有 `generation`，旧 caller 可能把复用同一个 slot 的新 worker 误认为自己启动的 worker。
### 4.5 `BackgroundWorkerArray`
`BackgroundWorkerArray` 包含：
```text
total_slots:
  应等于 max_worker_processes。
parallel_register_count:
  backend 注册 parallel worker 时递增，受 BackgroundWorkerLock 保护。
parallel_terminate_count:
  postmaster forget/terminate parallel worker 时递增，lockless。
slot[]:
  所有 worker 的 shared slot。
```
parallel worker active count 通过两个计数器相减得到。
这是为了避免 postmaster 取锁。
计数可能短暂陈旧，但足以限制 `max_parallel_workers`。
## 5. 主流程源码 walkthrough
### 5.1 启动期创建 background worker shared state
核心路径：
```text
PostmasterMain()
  -> RegisterBuiltinShmemCallbacks()
     -> subsystemlist.h
        -> BackgroundWorkerShmemCallbacks
  -> shared memory sizing / creation
  -> ShmemInitRequested()
     -> BackgroundWorkerShmemRequest()
     -> BackgroundWorkerShmemInit()
```
`BackgroundWorkerShmemRequest()` 申请 `Background Worker Data`。
大小是 `BackgroundWorkerArray` 头部加 `max_worker_processes` 个 slot。
`BackgroundWorkerShmemInit()` 做三件事：
```text
1. 初始化 total_slots 和 parallel counters。
2. 把 postmaster private BackgroundWorkerList 中已有 static worker 拷贝到 shared slot。
3. 剩余 slot 标记为 in_use = false，留给 dynamic worker。
```
这说明 static worker 的注册发生在 shared memory 初始化之前。
postmaster 先把 descriptor 放入 private list，shared memory 创建后再建立 shared slot 对应关系。
### 5.2 static registration
static worker 入口：
```text
RegisterBackgroundWorker(BackgroundWorker *worker)
```
典型来源是扩展 `_PG_init()`，例如 `worker_spi`：
```text
worker.bgw_flags = BGWORKER_SHMEM_ACCESS | BGWORKER_BACKEND_DATABASE_CONNECTION;
worker.bgw_start_time = BgWorkerStart_RecoveryFinished;
worker.bgw_restart_time = BGW_NEVER_RESTART;
worker.bgw_library_name = "worker_spi";
worker.bgw_function_name = "worker_spi_main";
worker.bgw_notify_pid = 0;
RegisterBackgroundWorker(&worker);
```
static registration 的边界：
```text
必须在 postmaster 早期环境；
通常来自 shared_preload_libraries；
不能发生在 BackgroundWorkerData 已初始化之后；
不能设置 bgw_notify_pid；
数量不能超过 max_worker_processes。
```
`SanityCheckBackgroundWorker()` 检查：
```text
必须有 BGWORKER_SHMEM_ACCESS；
database connection worker 不能 BgWorkerStart_PostmasterStart；
BGWORKER_INTERRUPTIBLE 必须同时有 database connection；
restart_time 必须合法；
parallel worker 不能配置自动 restart；
bgw_type 为空则复制 bgw_name。
```
static 注册失败通常 LOG 并返回。
这是 postmaster 启动期路径，不应把普通扩展配置错误扩大成不受控状态。
### 5.3 dynamic registration
dynamic worker 入口：
```text
RegisterDynamicBackgroundWorker(worker, &handle)
```
主流程：
```text
if !IsUnderPostmaster:
  return false
SanityCheckBackgroundWorker(worker, ERROR)
LWLockAcquire(BackgroundWorkerLock, LW_EXCLUSIVE)
  if parallel active count >= max_parallel_workers:
    release and return false
  scan BackgroundWorkerData->slot[]
  find !slot->in_use
    memcpy slot->worker
    slot->pid = InvalidPid
    slot->generation++
    slot->terminate = false
    maybe parallel_register_count++
    pg_write_barrier()
    slot->in_use = true
LWLockRelease()
SendPostmasterSignal(PMSIGNAL_BACKGROUND_WORKER_CHANGE)
fill handle(slot, generation)
return true
```
`RegisterDynamicBackgroundWorker()` 返回 true 只说明 shared slot 已发布。
此时 `slot->pid` 仍可能是 `InvalidPid`，postmaster 尚未启动进程。
没有 slot 时返回 false。
parallel query 把它当作少 worker fallback；某些扩展会把它变成 SQL ERROR。
### 5.4 PMSIGNAL 唤醒 postmaster
dynamic backend 不直接 fork worker。
它调用：
```text
SendPostmasterSignal(PMSIGNAL_BACKGROUND_WORKER_CHANGE)
```
`pmsignal.c` 的语义是设置 reason flag，并向 postmaster 发 `SIGUSR1`。
同一 reason 的多次信号可能合并，这没问题，因为 postmaster 会重新扫描 shared slots。
postmaster 状态机中：
```text
if CheckPostmasterSignal(PMSIGNAL_BACKGROUND_WORKER_CHANGE):
  BackgroundWorkerStateChange(pmState < PM_STOP_BACKENDS)
  StartWorkerNeeded = true
```
如果已经进入停止阶段，`allow_new_workers` 为 false。
postmaster 会让新请求走终止/释放路径，并通知等待者，而不是让 caller 永远等一个不会启动的 worker。
### 5.5 postmaster 接收 dynamic request
`BackgroundWorkerStateChange()` 运行在 postmaster。
它不能假设 shared memory 内容完全可信。
主流程：
```text
for slot in BackgroundWorkerData:
  if !slot->in_use:
    continue
  pg_read_barrier()
  rw = FindRegisteredWorkerBySlotNumber(slotno)
  if rw exists:
    if slot->terminate && !rw->rw_terminate:
      rw->rw_terminate = true
      if rw->rw_pid != 0:
        kill(rw->rw_pid, SIGTERM)
      else:
        ReportBackgroundWorkerPID(rw)
    continue
  if !allow_new_workers:
    slot->terminate = true
  if slot->terminate:
    save notify_pid
    maybe parallel_terminate_count++
    slot->pid = 0
    pg_memory_barrier()
    slot->in_use = false
    if notify_pid != 0:
      kill(notify_pid, SIGUSR1)
    continue
  allocate RegisteredBgWorker in PostmasterContext
  copy strings with ascii_safe_strlcpy()
  copy fixed fields
  validate bgw_notify_pid with PostmasterMarkPIDForWorkerNotify()
  push rw into BackgroundWorkerList
```
两个细节很重要。
第一，字符串用 `ascii_safe_strlcpy()`。
如果 shared memory 被写坏或没有 NUL 结尾，postmaster 仍不应被拖垮。
第二，`bgw_notify_pid` 要能在 `ActiveChildList` 中找到。
postmaster 找到后把对应 `PMChild.bgworker_notify` 标为 true，以便该 backend 退出时取消未来通知。
### 5.6 启动时机
`bgw_start_time` 与 postmaster state 绑定。
```text
BgWorkerStart_PostmasterStart:
  PM_INIT / PM_STARTUP / PM_RECOVERY 等早期即可启动。
BgWorkerStart_ConsistentState:
  hot standby 达到 consistent state 后可启动。
BgWorkerStart_RecoveryFinished:
  PM_RUN 后可启动。
```
`bgworker_should_start_now()` 做这个判断。
若 worker 声明 `BGWORKER_BACKEND_DATABASE_CONNECTION`，则不能要求 `BgWorkerStart_PostmasterStart`。
数据库连接需要 catalog、ProcArray、recovery state 等环境，早期 postmaster start 阶段不能满足。
### 5.7 postmaster 启动 worker
`maybe_start_bgworkers()` 扫描 `BackgroundWorkerList`：
```text
skip rw_pid != 0
if rw_terminate:
  ForgetBackgroundWorker(rw)
if rw_crashed_at != 0:
  if restart_time == BGW_NEVER_RESTART:
    ForgetBackgroundWorker(rw)
  else if restart delay not elapsed:
    HaveCrashedWorker = true
    continue
if bgworker_should_start_now():
  rw_crashed_at = 0
  StartBackgroundWorker(rw)
```
`StartBackgroundWorker()`：
```text
AssignPostmasterChildSlot(B_BG_WORKER)
  -> fail: rw_crashed_at = now, return false
postmaster_child_launch(B_BG_WORKER, child_slot, &rw->rw_worker, sizeof(BackgroundWorker), NULL)
  -> fail: release child slot, rw_crashed_at = now, return false
rw->rw_pid = worker_pid
PMChild.pid = worker_pid
ReportBackgroundWorkerPID(rw)
```
`ReportBackgroundWorkerPID()` 把 pid 写回 shared slot。
如果 `bgw_notify_pid != 0`，postmaster 向 caller 发送 `SIGUSR1`。
这仍然只说明 worker process 已经 fork。
它不说明 worker 完成入口函数加载、数据库连接或业务 ready。
### 5.8 fork / EXEC_BACKEND 启动数据
`postmaster_child_launch()` 把 `BackgroundWorker` 作为 startup data 传给 child。
普通 fork 平台：
```text
fork_process()
  -> child:
     MyBackendType = B_BG_WORKER
     ClosePostmasterPorts()
     InitPostmasterChild()
     MemoryContextSwitchTo(TopMemoryContext)
     MyPMChildSlot = child_slot
     child_process_kinds[B_BG_WORKER].main_fn(startup_data, len)
```
EXEC_BACKEND 平台：
```text
internal_forkexec()
  -> save_backend_variables()
  -> exec postgres --forkchild=...
  -> SubPostmasterMain()
     -> read_backend_variables()
     -> PGSharedMemoryReAttach()
     -> RegisterBuiltinShmemCallbacks()
     -> process_shared_preload_libraries()
     -> InitShmemAllocator()
     -> ShmemCallRequestCallbacks()
     -> main_fn(startup_data, len)
```
这解释了为什么 descriptor 里传 `bgw_library_name` 和 `bgw_function_name`，而不是函数指针。
不同进程或 EXEC_BACKEND 后，动态库地址不一定相同。
`LookupBackgroundWorkerFunction()` 对 `"postgres"` 使用内部表 `InternalBGWorkers[]`，对扩展库使用 `load_external_function()`。
### 5.9 `BackgroundWorkerMain()`
worker child 入口：
```text
BackgroundWorkerMain(startup_data, startup_data_len)
```
主流程：
```text
copy BackgroundWorker into TopMemoryContext
delete PostmasterContext
MyBgworkerEntry = worker
init_ps_display(bgw_name)
install signal handlers
InitializeTimeouts()
set sigjmp ERROR boundary
InitProcess()
BaseInit()
entrypt = LookupBackgroundWorkerFunction(library, function)
entrypt(bgw_main_arg)
proc_exit(0)
```
`InitProcess()` 在用户入口函数之前执行。
它会：
```text
从 ProcGlobal->bgworkerFreeProcs 取 PGPROC；
初始化 pid、backendType、lock/wait fields；
OwnLatch(&MyProc->procLatch)；
SwitchToSharedLatch()；
让 MyLatch 指向 shared procLatch；
让 wait_event_info 写入 MyProc。
```
所以入口函数开始时，worker 已经能使用 shared memory、LWLock、`MyProc`、`MyLatch` 和 wait event。
但它还没有数据库连接。
`BackgroundWorkerMain()` 的 ERROR 路径：
```text
sigsetjmp resumes
error_context_stack = NULL
HOLD_INTERRUPTS()
BackgroundWorkerUnblockSignals()
EmitErrorReport()
proc_exit(1)
```
exit status 1 会交给 postmaster restart policy。
### 5.10 数据库连接
连接 API：
```text
BackgroundWorkerInitializeConnection(dbname, username, flags)
BackgroundWorkerInitializeConnectionByOid(dboid, useroid, flags)
```
共同流程：
```text
if !(MyBgworkerEntry->bgw_flags & BGWORKER_BACKEND_DATABASE_CONNECTION):
  ereport(FATAL)
translate BGWORKER_BYPASS_ALLOWCONN / BGWORKER_BYPASS_ROLELOGINCHECK
InitPostgres(...)
if !IsInitProcessingMode():
  ereport(ERROR)
SetProcessingMode(NormalProcessing)
```
`InitPostgres()` 把 worker 推进到数据库 backend 状态：
```text
InitProcessPhase2():
  加入 ProcArray，从此对其它 backend 可见。
pgstat_beinit() / pgstat_bestart_*():
  初始化 pg_stat_activity 可见状态。
SharedInvalBackendInit(false):
  接入 shared invalidation。
ProcSignalInit():
  接入 procsignal barrier 和 cancel key。
RelationCacheInitialize() / InitCatalogCache():
  准备 relcache/syscache。
before_shmem_exit(ShutdownPostgres, 0):
  注册数据库层退出 cleanup。
StartTransactionCommand() and role initialization:
  建立初始化事务和身份。
```
因此 database connection 不是 registration flag 的自动副作用。
它是 worker main 中显式推进的一段 backend bootstrap。
### 5.11 signal、latch 与 worker 主循环
`worker_spi_main()` 是标准样式：
```text
pqsignal(SIGHUP, SignalHandlerForConfigReload)
pqsignal(SIGTERM, die)
BackgroundWorkerUnblockSignals()
BackgroundWorkerInitializeConnection(...)
for (;;):
  WaitLatch(MyLatch,
            WL_LATCH_SET | WL_TIMEOUT | WL_EXIT_ON_PM_DEATH,
            naptime,
            worker_wait_event)
  ResetLatch(MyLatch)
  CHECK_FOR_INTERRUPTS()
  if ConfigReloadPending:
    ProcessConfigFile(PGC_SIGHUP)
  StartTransactionCommand()
  SPI_connect()
  PushActiveSnapshot(GetTransactionSnapshot())
  SPI_execute(...)
  SPI_finish()
  PopActiveSnapshot()
  CommitTransactionCommand()
```
顺序的含义：
```text
先安装本 worker 需要的 signal handler；
再 unblock signals；
再连接数据库；
idle 时等待 MyLatch，而不是裸 pg_usleep()；
醒来后 ResetLatch 并重新检查业务 predicate；
用 CHECK_FOR_INTERRUPTS() 把 ProcDiePending 等异步事件转成安全点处理；
每轮业务 SQL 自己建立事务和 snapshot。
```
`die()` handler 不直接做复杂 cleanup。
它设置 `InterruptPending`、`ProcDiePending`、记录 sender，并 `SetLatch(MyLatch)`。
真正 FATAL 退出发生在 `CHECK_FOR_INTERRUPTS()` 触发的 `ProcessInterrupts()`。
长期 worker 等待时应包含 `WL_EXIT_ON_PM_DEATH` 或明确处理 `WL_POSTMASTER_DEATH`。
postmaster 死亡后继续访问 shared memory 通常不是安全设计。
### 5.12 startup/shutdown wait
dynamic caller 想等 worker pid：
```text
worker.bgw_notify_pid = MyProcPid
RegisterDynamicBackgroundWorker(&worker, &handle)
WaitForBackgroundWorkerStartup(handle, &pid)
```
`WaitForBackgroundWorkerStartup()` 循环：
```text
status = GetBackgroundWorkerPid(handle, &pid)
if status == BGWH_STARTED:
  return pid
if status != BGWH_NOT_YET_STARTED:
  return status
WaitLatch(MyLatch,
          WL_LATCH_SET | WL_POSTMASTER_DEATH,
          0,
          WAIT_EVENT_BGWORKER_STARTUP)
ResetLatch(MyLatch)
```
`WaitForBackgroundWorkerShutdown()` 同理等待 `BGWH_STOPPED`，wait event 是 `WAIT_EVENT_BGWORKER_SHUTDOWN`。
caller 必须设置 `bgw_notify_pid`，否则 postmaster 状态变化不会及时 `SIGUSR1` 唤醒它。
但 startup wait 只等到 pid 写回，不等模块 ready。
parallel worker 额外等待 error queue sender attach。
`test_shm_mq` 额外等待 `workers_ready` 计数，并由 worker `SetLatch(&registrant->procLatch)`。
## 6. 生命周期 / ownership / cleanup
### 6.1 谁创建
```text
static descriptor:
  extension _PG_init()
  -> RegisterBackgroundWorker()
  -> postmaster private BackgroundWorkerList
  -> BackgroundWorkerShmemInit() assigns shared slot
dynamic descriptor:
  regular backend
  -> RegisterDynamicBackgroundWorker()
  -> shared BackgroundWorkerSlot
  -> PMSIGNAL
  -> postmaster copies into RegisteredBgWorker
worker process:
  postmaster
  -> AssignPostmasterChildSlot()
  -> postmaster_child_launch()
database session:
  worker entrypoint
  -> BackgroundWorkerInitializeConnection*()
  -> InitPostgres()
```
### 6.2 谁持有

| 对象 | owner |
| --- | --- |
| `RegisteredBgWorker` | postmaster `PostmasterContext`。 |
| `BackgroundWorkerSlot` | shared memory；`in_use` 表示交接状态。 |
| `BackgroundWorkerHandle` | dynamic caller backend-local memory。 |
| child process | postmaster `PMChild` / `ActiveChildList`。 |
| `PGPROC` | worker process，退出时 `ProcKill` 释放。 |
| `MyLatch` | worker process 拥有的 `PGPROC.procLatch`。 |
| database state | `InitPostgres()` 注册的 callbacks、ResourceOwner、MemoryContext 等。 |
| DSM / shm_mq | 具体模块协议，常由 `on_dsm_detach()` 或 ResourceOwner 清理。 |
### 6.3 正常退出
```text
entrypt returns
  -> BackgroundWorkerMain() proc_exit(0)
  -> postmaster CleanupBackend()
  -> rw_crashed_at = 0
  -> rw_terminate = true
  -> rw_pid = 0
  -> ReportBackgroundWorkerExit()
  -> ForgetBackgroundWorker() if no restart
```
exit code 0 的 public contract 是不重启。
这适合 one-shot worker 或显式完成任务的 worker。
### 6.4 ERROR / FATAL 退出
```text
ereport(ERROR)
  -> BackgroundWorkerMain error handler
  -> EmitErrorReport()
  -> proc_exit(1)
  -> CleanupBackend()
  -> rw_crashed_at = now
  -> ReportBackgroundWorkerExit()
  -> maybe_start_bgworkers() after restart_time
```
`ereport(FATAL)` 也会以非零但非 crash 的方式退出。
是否重启由 `bgw_restart_time` 决定。
### 6.5 crash 退出
如果 child exit status 既不是 0 也不是 1，`CleanupBackend()` 将其视为 crash。
它调用 `HandleChildCrash()`，可能触发系统 reset cycle。
在 crash restart 之后：
```text
ResetBackgroundWorkerCrashTimes()
  if restart_time == BGW_NEVER_RESTART:
    ForgetBackgroundWorker(rw)
  else:
    rw_crashed_at = 0
    rw_pid = 0
    rw_worker.bgw_notify_pid = 0
```
parallel worker 必须是 `BGW_NEVER_RESTART`。
否则 `parallel_register_count` / `parallel_terminate_count` 无法跨 crash cycle 维持语义。
### 6.6 notify pid cleanup
dynamic caller 可能在 worker 启动或停止前退出。
postmaster 不能以后再向这个 pid 发送 SIGUSR1。
路径：
```text
PostmasterMarkPIDForWorkerNotify(pid)
  -> PMChild.bgworker_notify = true
CleanupBackend() of caller:
  if bp_bgworker_notify:
    BackgroundWorkerStopNotifications(bp_pid)
BackgroundWorkerStopNotifications():
  scan BackgroundWorkerList
  set matching rw_worker.bgw_notify_pid = 0
```
这是防 pid reuse 的 cleanup，不是 worker 业务状态 cleanup。
### 6.7 database conflict cleanup
`BGWORKER_INTERRUPTIBLE` 支持数据库级冲突处理。
相关路径在 `procarray.c`：
```text
TerminateBackgroundWorkersForDatabase(databaseId)
  -> scan BackgroundWorkerData slots
  -> if slot.in_use and BGWORKER_INTERRUPTIBLE:
       proc = BackendPidGetProc(slot.pid)
       if proc && proc->databaseId == databaseId:
         slot->terminate = true
         SendPostmasterSignal(PMSIGNAL_BACKGROUND_WORKER_CHANGE)
```
只有已经连接数据库并发布 `PGPROC.databaseId` 的 worker 才能被这条路径识别。
不声明 interruptible 的 worker 不会因为这个机制被自动终止。
## 7. 正确性机制层次

| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `BackgroundWorkerLock` | backend 之间分配 dynamic slot 互斥。 | postmaster 与 backend 互斥。 |
| `in_use` + barrier | postmaster 看到完整 slot 内容。 | descriptor 业务合法。 |
| `generation` | 旧 handle 不误认复用 slot。 | pid 一直存活。 |
| PMSIGNAL | postmaster 会重新扫描状态。 | 每次事件都排队计数。 |
| postmaster private list | 监督状态不依赖 backend 后续写入。 | worker 业务 ready。 |
| `PMChild` | child pid/type/notify cleanup 可追踪。 | child 内部资源不泄漏。 |
| `PGPROC` | worker 参与 lock、latch、wait event、ProcArray。 | database 已连接。 |
| `InitPostgres()` | database identity、cache、sinval、transaction 环境。 | 模块业务状态已初始化。 |
| latch | 可靠唤醒和 postmaster death 等待。 | 消息内容、次数或业务 predicate。 |
| exit status | postmaster 判断 restart/forget。 | 失败根因。 |
关键不变量：
```text
postmaster 不拿 BackgroundWorkerLock；
backend 先填 slot，再 write barrier，再置 in_use；
postmaster 看到 in_use 后 read barrier，再读 descriptor；
dynamic handle 必须校验 generation；
database connection flag 只是能力声明；
signal handler 只设置 flag 并 SetLatch(MyLatch)；
长期等待必须处理 postmaster death；
worker started 不等于 application ready。
```
这节课最重要的 boundary 感是：
```text
memory ordering 管 registration handoff；
postmaster child slot 管 process supervision；
PGPROC 管 shared memory 协作身份；
InitPostgres 管数据库会话身份；
模块自己的 shared/DSM 协议管业务 ready。
```
## 8. 错误路径 / 异常路径 / fallback
### 8.1 descriptor 不合法
`SanityCheckBackgroundWorker()` 处理 API 边界错误：
```text
缺少 BGWORKER_SHMEM_ACCESS；
database connection worker 要求 PostmasterStart 启动；
BGWORKER_INTERRUPTIBLE 没有 database connection；
restart interval 非法；
parallel worker 配置 restart。
```
static registration 用 LOG 级别并返回。
dynamic registration 用 ERROR 级别。
这是因为 dynamic caller 是普通 backend，可以安全接收 ERROR。
### 8.2 slot 不足
`RegisterDynamicBackgroundWorker()` 找不到 free slot 时返回 false。
可能表现：
```text
parallel query:
  Workers Launched 少于 Workers Planned，但 query 可以继续。
worker_spi_launch():
  返回 NULL 或转成 could not start background process。
logical replication:
  cleanup 自己的 worker slot，WARNING out of background worker slots。
```
不要把 slot 不足直接解释成 worker crash。
优先检查 `max_worker_processes`、`max_parallel_workers` 和其它 worker 消耗。
### 8.3 postmaster child slot 或 fork 失败
`StartBackgroundWorker()` 中：
```text
AssignPostmasterChildSlot() fails:
  LOG no slot available
  rw_crashed_at = now
  return false
postmaster_child_launch() fails:
  LOG could not fork
  ReleasePostmasterChildSlot()
  rw_crashed_at = now
  return false
```
postmaster 把它当成 crash-like 状态，稍后重试。
这样避免资源压力下 tight retry。
### 8.4 入口函数加载失败
入口函数查找发生在 child 中：
```text
LookupBackgroundWorkerFunction()
  -> InternalBGWorkers[] for "postgres"
  -> load_external_function() for extension library
```
失败会让 worker ERROR/FATAL 退出。
postmaster 不在自己进程里加载用户函数，也不会因此直接 crash。
### 8.5 未声明 database connection 却连接
`BackgroundWorkerInitializeConnection*()` 检查 flag。
```text
if !(bgw_flags & BGWORKER_BACKEND_DATABASE_CONNECTION):
  ereport(FATAL)
```
这是 FATAL，不是普通 ERROR。
worker 进程退出，postmaster 再按 restart policy 处理。
### 8.6 等待 startup 时 postmaster 死亡
`WaitForBackgroundWorkerStartup()` 依赖 postmaster 写回 pid。
如果 postmaster 死亡，状态不会再推进。
返回：
```text
BGWH_POSTMASTER_DIED
```
这不是 worker 自身失败，而是监督者不可用。
### 8.7 `BGWH_STOPPED` 的歧义
`BGWH_STOPPED` 可能表示：
```text
worker 正常退出；
worker BGW_NEVER_RESTART；
worker 从未启动就被 unregister；
TerminateBackgroundWorker() 在启动前生效；
handle generation 已经过期；
worker 临时停止，稍后会按 restart_time 重启。
```
因此它不是根因。
必须结合 log、模块 shared state、ready 协议和 restart policy 判断。
### 8.8 SIGTERM 与安全点
postmaster 终止 worker 时发送 SIGTERM。
默认 handler `die()` 做：
```text
InterruptPending = true
ProcDiePending = true
SetLatch(MyLatch)
```
真正处理在 `CHECK_FOR_INTERRUPTS()`。
如果 worker 主循环不检查中断，或者长时间阻塞在不受 latch 管理的 syscall，shutdown 会延迟。
### 8.9 terminate 未启动 worker
dynamic caller 可能在 postmaster fork 前调用 `TerminateBackgroundWorker()`。
postmaster 看到 terminate 后会：
```text
slot->pid = 0
slot->in_use = false
notify bgw_notify_pid
```
否则等待 startup 的 caller 可能永远看不到 `STARTED` 或 `STOPPED`。
### 8.10 业务 ready 失败
background worker API 不定义业务 ready。
例子：
```text
parallel worker:
  leader 还要等 error queue sender attach。
test_shm_mq:
  registrant 还要等 workers_ready 计数。
logical replication:
  launcher 还要看自己的 LogicalRepWorker slot 和 proc 字段。
```
这类 ready predicate 必须由模块自己设计，并通常通过 shared memory + latch 传达。
## 9. 成本、资源与跨模块传播
slot 资源：
```text
max_worker_processes 同时约束 static bgworker、dynamic bgworker、
parallel worker、logical replication worker、autoprewarm worker 等。
```
slot 不足会传播为：
```text
parallel 少 worker；
logical replication worker 无法启动；
扩展启动函数 ERROR 或返回 NULL；
后台任务延迟。
```
启动成本：
```text
postmaster child slot；
fork 或 EXEC_BACKEND；
shared memory attach；
InitProcess() 分配 PGPROC；
BaseInit()；
动态库加载；
可选 InitPostgres()；
模块自己的 DSM/cache/transaction 初始化。
```
如果任务很短且频繁，反复 dynamic worker 启动会把 fork/InitPostgres 成本放大。
更常见的设计是长期 worker + queue，而不是每个任务一个进程。
共享状态传播：
```text
InitProcess():
  worker 进入 PGPROC / latch / wait event 世界。
InitPostgres():
  worker 进入 ProcArray / pgstat / sinval / cache / transaction 世界。
worker SQL:
  进一步参与 lock manager、snapshot、WAL、buffer、ResourceOwner cleanup。
```
所以连接数据库的 background worker 不是“免费后台线程”。
它是实例级并发状态的一员。
## 10. 观测与诊断入口
本节锚定的 runtime truth：
```text
RegisterDynamicBackgroundWorker() 成功只说明 shared slot 已发布；
WaitForBackgroundWorkerStartup() 返回 STARTED 只说明 pid 已写回；
application ready 必须看模块自己的 ready predicate。
```
SQL 入口：
```sql
select pid, backend_type, datname, usename, state, wait_event_type, wait_event, query
from pg_stat_activity
where backend_type like '%worker%';
```
能看到：
```text
pid；
backend_type / bgw_type；
database/user；
state；
wait_event；
worker 自己报告的 activity。
```
看不到：
```text
BackgroundWorkerSlot.in_use；
generation；
rw_crashed_at；
rw_terminate；
restart delay 是否未到；
postmaster private BackgroundWorkerList；
模块 ready predicate 是否满足。
```
wait event：
```text
WAIT_EVENT_BGWORKER_STARTUP:
  等待方正在等 worker pid/startup state。
WAIT_EVENT_BGWORKER_SHUTDOWN:
  等待方正在等 worker stop。
extension wait event:
  worker 或 caller 自己定义的业务等待点，例如 WorkerSpiMain。
```
wait event 只能说明当前睡在哪个等待点。
它不说明谁设置了 latch、事件发生了几次、worker 是否要重启。
server log 常见线索：
```text
registering background worker "...";
starting background worker process "...";
unregistering background worker "...";
could not fork background worker process;
background worker exited with exit code;
terminating background worker due to administrator command;
out of background worker slots.
```
很多 registration/start 日志是 DEBUG1。
开发环境可以临时提高 `log_min_messages`。
断点建议：
```text
RegisterDynamicBackgroundWorker
BackgroundWorkerStateChange
StartBackgroundWorker
ReportBackgroundWorkerPID
BackgroundWorkerMain
BackgroundWorkerInitializeConnection
CleanupBackend
ReportBackgroundWorkerExit
ForgetBackgroundWorker
TerminateBackgroundWorker
```
观察变量：
```text
BackgroundWorkerData->slot[i].in_use
BackgroundWorkerData->slot[i].pid
BackgroundWorkerData->slot[i].generation
BackgroundWorkerData->slot[i].terminate
rw->rw_pid
rw->rw_crashed_at
rw->rw_terminate
MyBgworkerEntry->bgw_name
MyProc->databaseId
MyProc->backendType
```
诊断时先分类：
```text
registration failure:
  关注 slot、max_worker_processes、BackgroundWorkerLock。
fork/start failure:
  关注 postmaster log、StartBackgroundWorker()、child slot。
connection failure:
  关注 BackgroundWorkerInitializeConnection*()、InitPostgres()、database/role。
business ready timeout:
  关注模块 shared state、DSM、shm_mq、latch predicate。
runtime slow:
  关注 worker 业务 SQL、SPI、locks、IO，不要先怪 bgworker API。
```
## 11. 常见误区
1. 注册成功等于进程启动。
   不对。dynamic registration 成功只是 slot 发布，pid 可能仍是 `InvalidPid`。
2. `BGWH_STARTED` 等于业务 ready。
   不对。它只表示 postmaster 写回了 pid。
3. `BGWORKER_BACKEND_DATABASE_CONNECTION` 会自动连接数据库。
   不对。worker 必须显式调用 `BackgroundWorkerInitializeConnection*()`。
4. background worker 是线程。
   不对。它是独立进程，不能传普通指针。
5. `bgw_main_arg` 可以传 backend-local 指针。
   不对。传 DSM handle、OID、小整数或 slot number。
6. postmaster 会清理 worker 的所有业务状态。
   不对。postmaster 管 registration 和 process supervision；DSM、queue、业务 slot 要由模块 cleanup。
7. worker 主循环可以随便 `pg_usleep()`。
   长期 worker 应用 `WaitLatch()`，处理 signal、timeout 和 postmaster death。
8. `BGWH_STOPPED` 就是失败原因。
   不对。它是多种 stopped/forgotten/reused/restart 状态的合并结果。
9. `BGWORKER_INTERRUPTIBLE` 会终止所有相关 worker。
   不对。它只作用于声明 interruptible 且 `PGPROC.databaseId` 匹配的 worker。
## 12. 课堂实验
### 实验 1: 跟踪 worker_spi 生命周期
目标：看到 registration、pid startup、database connection、latch loop 是不同阶段。
准备：
```text
编译安装 src/test/modules/worker_spi；
在测试实例中设置 shared_preload_libraries = 'worker_spi'；
设置 log_min_messages = debug1；
重启实例。
```
观察：
```text
server log:
  registering / starting worker_spi。
pg_stat_activity:
  backend_type like 'worker_spi%'。
```
断点：
```text
RegisterBackgroundWorker
BackgroundWorkerShmemInit
maybe_start_bgworkers
StartBackgroundWorker
BackgroundWorkerMain
worker_spi_main
BackgroundWorkerInitializeConnection
```
要画出的状态：
```text
BackgroundWorkerList
  -> BackgroundWorkerSlot
  -> RegisteredBgWorker.rw_pid
  -> MyBgworkerEntry
  -> MyProc.databaseId
  -> pg_stat_activity row
```
验证点：
```text
worker entrypoint 开始时已有 PGPROC 和 MyLatch；
调用 BackgroundWorkerInitializeConnection*() 后才有 database identity；
主循环通过 WaitLatch() 处理 timeout、signal 和 postmaster death。
```
### 实验 2: 观察 dynamic registration 与 startup wait
目标：区分 slot 发布和 pid 写回。
使用 `worker_spi_launch()` 或其它 dynamic worker 启动函数。
断点：
```text
RegisterDynamicBackgroundWorker
BackgroundWorkerStateChange
ReportBackgroundWorkerPID
WaitForBackgroundWorkerStartup
```
记录：
```text
slot->generation；
slot->pid 从 InvalidPid 到正数；
handle->slot；
handle->generation；
WaitForBackgroundWorkerStartup() 返回状态。
```
验证点：
```text
RegisterDynamicBackgroundWorker() true 时，worker 可能还没有 pid；
bgw_notify_pid = MyProcPid 是及时 wait startup 的必要条件；
BGWH_STARTED 不证明业务 ready。
```
### 实验 3: 制造 slot 不足
目标：区分 registration failure 与 worker crash。
方法：
```text
在测试环境降低 max_worker_processes；
运行会请求 parallel worker 的查询，或同时启动多个 dynamic worker；
观察 RegisterDynamicBackgroundWorker() 返回 false 的调用方行为。
```
并行查询入口：
```sql
explain (analyze, verbose)
select count(*) from large_table;
```
观察：
```text
Workers Planned；
Workers Launched；
server log 中是否有 out of background worker slots；
是否有 worker crash / exit code 日志。
```
验证点：
```text
少 worker 可以是正常 fallback；
没有 pid 的 registration failure 不是 worker crash。
```
### 实验 4: 给 ready 协议加日志
目标：证明 application ready 是模块协议，不是 bgworker API。
选择 `src/test/modules/test_shm_mq/worker.c`：
```text
在 dsm_attach() 成功后加日志；
在 shm_toc_attach() 成功后加日志；
在 workers_ready++ 前后加日志；
在 SetLatch(&registrant->procLatch) 前后加日志。
```
对照 `setup.c` 中等待：
```text
workers_ready >= nworkers 才认为 ready；
check_worker_status() 用 BackgroundWorkerHandle 判断 worker 是否 stopped；
WaitLatch(MyLatch) 等 worker ready 或失败。
```
验证点：
```text
postmaster 写回 pid 早于 workers_ready++；
caller 必须重新检查 shared predicate；
latch 是唤醒提示，不是消息队列。
```
## 13. 讨论题
1. 为什么 dynamic bgworker 不能由普通 backend 直接 fork？
2. `in_use` 为什么必须配合 `pg_write_barrier()` / `pg_read_barrier()`？
3. postmaster 为什么要把 shared slot 内容复制到私有 `RegisteredBgWorker`？
4. `BGWH_STARTED` 和 application ready 之间缺了哪一层协议？
5. 为什么 database connection 不能由 `BackgroundWorkerMain()` 自动完成？
6. worker 退出时，哪些资源由 `proc_exit()` / `ProcKill` 释放，哪些必须由模块自己释放？
7. dynamic caller 设置 `bgw_notify_pid` 后退出，PostgreSQL 如何避免信号打到复用 pid？
8. 为什么 parallel worker 不能配置自动 restart？
9. `BGWORKER_INTERRUPTIBLE` 依赖 `PGPROC.databaseId`，这带来什么边界？
10. 你如何区分 slot 不足、fork 失败、worker ERROR、业务 ready timeout？
## 14. 本节小结
background worker lifecycle 的核心链路：
```text
BackgroundWorker descriptor
  -> static private list or dynamic shared slot
  -> postmaster BackgroundWorkerStateChange()
  -> maybe_start_bgworkers()
  -> StartBackgroundWorker()
  -> BackgroundWorkerMain()
  -> InitProcess()
  -> user entrypoint
  -> optional InitPostgres()
  -> latch/signal main loop
  -> proc_exit()
  -> CleanupBackend()
  -> restart or ForgetBackgroundWorker()
```
核心状态边界：
```text
BackgroundWorker:
  public descriptor。
BackgroundWorkerSlot:
  shared handoff and pid/status observation。
RegisteredBgWorker:
  postmaster private supervision state。
BackgroundWorkerHandle:
  caller-local slot + generation。
PGPROC:
  worker 接入 locks、latch、wait event、ProcArray 的身份。
```
最重要的不变量：
```text
postmaster 不用普通 shared memory lock 接收 dynamic registration；
backend 通过 slot + barrier + PMSIGNAL 发布请求；
worker fork 成功不等于数据库连接完成；
pid started 不等于业务 ready；
signal handler 只设置 flag 和 latch；
exit status、terminate flag、restart_time 共同决定重启和 cleanup。
```
能直接观测的是 pid、backend_type、wait_event、日志中的 start/exit 和调用者等待点。
不能直接从 SQL 看到的是 slot generation、in_use、rw_crashed_at、restart delay、postmaster private list 和业务 ready predicate。
可迁移 mental model：
```text
跨进程后台执行机制不能用一个状态位表达全部生命周期。
把 registration、process liveness、database attachment、application readiness、
cleanup ownership 拆开，每一层只承诺自己能可靠推进和可靠回收的状态。
```
