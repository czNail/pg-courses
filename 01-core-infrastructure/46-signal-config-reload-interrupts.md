# PostgreSQL signal config reload interrupts
## 课程定位
前置知识：已经理解 backend 进程拥有 `PGPROC`、`MyLatch`、`ProcSignal` slot，也理解 `Latch` 的语义是“唤醒等待者重新检查真实状态”，不是消息队列。
本节唯一主问题：
```text
SIGHUP、SIGTERM、SIGINT、ProcDiePending、QueryCancelPending
和 CHECK_FOR_INTERRUPTS 如何在安全边界处理异步事件？
```
核心矛盾：OS signal 可以在任意指令边界到达，但 PostgreSQL 的事务、锁、buffer pin、catalog cache、MemoryContext、ResourceOwner 和 FE/BE 协议只在少数边界上自洽。
如果 signal handler 立刻执行重活，响应很快，但可能在持锁、critical section、协议消息半读、错误栈半更新、shared memory 状态半修改时进入 abort、GUC reload 或 `ereport()`。
如果完全等 query 结束再处理，状态安全，但 `pg_cancel_backend()`、statement timeout、fast shutdown、配置重载、postmaster death 和 ProcSignal barrier 的响应延迟不可接受。
PostgreSQL 的基本解法是两阶段：
```text
signal handler:
  只设置 volatile sig_atomic_t flag；
  必要时 SetLatch(MyLatch) 打断等待。
safe boundary:
  CHECK_FOR_INTERRUPTS() 或主循环检查 pending flag；
  在 holdoff / critical section / protocol 边界允许时，
  把 flag 转换成 ERROR、FATAL、proc_exit、GUC reload 或 barrier processing。
```
学完后应能判断：为什么 handler 不能直接 `AbortTransaction()` 或 `ProcessConfigFile()`；为什么 `SIGINT` 通常设置 `QueryCancelPending`；为什么 `SIGTERM` 通常设置 `ProcDiePending`；为什么 `ProcDiePending` 压过 `QueryCancelPending`；为什么 `CHECK_FOR_INTERRUPTS()` 不是随处可放的宏；为什么 idle 状态的 query cancel 可能被吞掉；为什么 SIGHUP reload 不是全局原子切换。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e`。
## 1. 本节在总主线中的位置
前几节已经建立三层基础设施。
```text
backend identity:
  InitProcess()
    -> PGPROC
    -> MyProc / MyLatch
    -> ProcSignal slot
waiting and wakeup:
  WaitLatch()
    -> backend 睡眠
  SetLatch(MyLatch)
    -> backend 醒来重新检查业务状态
ERROR-safe cleanup:
  ereport(ERROR)
    -> longjmp 到顶层恢复
    -> AbortCurrentTransaction()
    -> ResourceOwner / MemoryContext / lock cleanup
```
本节把这三层串起来。
signal 本身不是完整控制流。
signal 只是“异步事件已经发生”的最小事实。
PostgreSQL 必须把这个事实压缩成可合并的本地状态，然后在系统自洽的边界上消费。
三条主链路如下。
```text
SIGINT:
  StatementCancelHandler()
    -> QueryCancelPending = true
    -> InterruptPending = true
    -> SetLatch(MyLatch)
    -> CHECK_FOR_INTERRUPTS()
    -> ProcessInterrupts()
    -> ereport(ERROR)
    -> 顶层错误恢复 abort 当前事务
SIGTERM:
  die()
    -> ProcDiePending = true
    -> InterruptPending = true
    -> SetLatch(MyLatch)
    -> CHECK_FOR_INTERRUPTS()
    -> ProcessInterrupts()
    -> ereport(FATAL) 或 proc_exit()
    -> 退出 callback 清理 shared/local state
SIGHUP:
  SignalHandlerForConfigReload()
    -> ConfigReloadPending = true
    -> SetLatch(MyLatch)
    -> 主循环 convenient point
    -> ProcessConfigFile(PGC_SIGHUP)
```
注意：`SIGHUP` 不设置 `InterruptPending`，也不由 `CHECK_FOR_INTERRUPTS()` 直接消费。
SIGHUP 是配置重载提示，不是 query cancel / die interrupt。
本节主线是：
```text
异步到达 -> pending flag -> latch wakeup -> safe boundary
  -> ERROR / FATAL / reload / barrier -> cleanup 或继续主循环
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
PostgreSQL 把异步 signal 降级为 backend-local pending flag，
用 MyLatch 缩短等待延迟，
再在 CHECK_FOR_INTERRUPTS()、client read/write wrapper
或进程主循环的明确边界上消费这些 flag。
```
这个模型有四个不变量。
第一，signal handler 只做最小动作。
```text
允许做:
  设置 volatile sig_atomic_t flag
  记录少量 sender pid / uid
  SetLatch(MyLatch)
  SIGQUIT 等极端路径 _exit(2)
避免做:
  解析配置文件
  分配复杂内存
  等待其它 backend
  访问 catalog
  abort transaction
  释放锁或 buffer pin
```
第二，`InterruptPending` 是总入口，不是具体原因。
它表示下一次 `CHECK_FOR_INTERRUPTS()` 应进入 `ProcessInterrupts()`。
它不表示一定要 cancel query，不表示一定要退出进程，也不表示一定来自 OS signal。
第三，安全边界由上下文决定。
`ProcessInterrupts()` 能否真正处理 pending interrupt 取决于：
```text
InterruptHoldoffCount
QueryCancelHoldoffCount
CritSectionCount
DoingCommandRead
client write 是否 blocked
当前是否允许 ERROR/FATAL longjmp
```
第四，SIGHUP reload 是旁路。
`ConfigReloadPending` 是 backend-local reload 提示。
它由 signal handler 设置，由普通 backend command loop 或后台进程 main loop 消费。
多次 SIGHUP 可合并，也没有“所有进程都 reload 完”的统一确认点。
## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/miscadmin.h` | 中断 flag、holdoff counter、`CHECK_FOR_INTERRUPTS()` contract。 |
| 2 | `src/backend/utils/init/globals.c` | `InterruptPending`、`QueryCancelPending`、`ProcDiePending`、`MyLatch` 定义。 |
| 3 | `src/backend/tcop/postgres.c` | `PostgresMain()`、`die()`、`StatementCancelHandler()`、`quickdie()`、`ProcessInterrupts()`。 |
| 4 | `src/backend/postmaster/interrupt.c` | `ConfigReloadPending`、`ShutdownRequestPending`、`ProcessMainLoopInterrupts()`。 |
| 5 | `src/backend/utils/misc/guc-file.l` | `ProcessConfigFile()` 入口、reload 临时 memory context。 |
| 6 | `src/backend/utils/misc/guc.c` | `ProcessConfigFileInternal()` 解析、应用、报错、`pending_restart`。 |
| 7 | `src/backend/utils/init/postinit.c` | `InitPostgres()` 中 `ProcSignalInit()`、timeout handler 注册和 startup holdoff。 |
| 8 | `src/backend/storage/ipc/procsignal.c` | `SIGUSR1` reason flag、cancel key、barrier generation、slot cleanup。 |
| 9 | `src/include/storage/procsignal.h` | `ProcSignalReason`、`ProcSignalBarrierType`、cancel key 长度。 |
| 10 | `src/backend/storage/ipc/latch.c` | handler 为什么必须 `SetLatch(MyLatch)`。 |
| 11 | `src/backend/libpq/pqcomm.c` | client connection lost 如何转成 interrupt。 |
| 12 | `src/backend/storage/lmgr/proc.c` | `LockErrorCleanup()` 在 cancel/die 前清 lock wait 状态。 |
| 13 | `src/backend/postmaster/postmaster.c` | postmaster 收到 SIGHUP 后 reload 并 `SignalChildren(SIGHUP)`。 |
推荐阅读顺序：
```text
miscadmin.h
  -> globals.c
  -> postgres.c signal handlers
  -> ProcessInterrupts()
  -> PostgresMain() command loop
  -> postmaster/interrupt.c
  -> guc-file.l / guc.c
  -> procsignal.c
  -> latch.c
```
不要从全仓库所有 `CHECK_FOR_INTERRUPTS()` 调用点开始读。
本节重点不是“哪里调用了宏”，而是“pending flag 何时变成数据库内部动作”。
## 4. 关键状态与边界
### `InterruptPending`
定义位置：
```text
src/backend/utils/init/globals.c
src/include/miscadmin.h
```
语义：当前 backend 有某类 interrupt 需要在安全点处理。
主要写入者：
```text
die()
StatementCancelHandler()
timeout handler
HandleProcSignalBarrierInterrupt()
pqcomm.c 中 client connection lost 路径
部分模块自定义 interrupt handler
```
主要读取者：
```text
INTERRUPTS_PENDING_CONDITION()
CHECK_FOR_INTERRUPTS()
ProcessInterrupts()
```
边界：backend-local，`volatile sig_atomic_t`，不记录次数，多个原因可合并，清掉它不等于所有原因都消失。
### `QueryCancelPending`
典型来源：
```text
SIGINT
pg_cancel_backend()
statement_timeout
lock_timeout
autovacuum cancel
部分 replication / wait path
```
通常结果：
```text
ProcessInterrupts()
  -> QueryCancelPending = false
  -> 判断 timeout indicator
  -> LockErrorCleanup()
  -> ereport(ERROR)
  -> 顶层错误恢复 abort 当前事务或当前 statement
```
它不表示 session 要结束。
它也不保证立即生效。
如果 `QueryCancelHoldoffCount != 0`，`ProcessInterrupts()` 会重新设置 `InterruptPending`，稍后再处理。
如果 `DoingCommandRead` 为真，普通用户 cancel 通常被清掉，因为 idle 状态没有正在执行的 query。
### `ProcDiePending`
典型来源：
```text
SIGTERM
pg_terminate_backend()
postmaster fast shutdown
background worker 被要求退出
```
通常结果：
```text
ProcessInterrupts()
  -> 先处理 ProcDiePending
  -> 清 QueryCancelPending
  -> LockErrorCleanup()
  -> 根据进程类型 ereport(FATAL) 或 proc_exit()
```
`ProcDiePending` 改变 session 生命周期。
它压过 `QueryCancelPending`，避免 backend 先抛一次普通 `ERROR` 回到 idle loop，再退出。
### `ConfigReloadPending`
定义位置：
```text
src/backend/postmaster/interrupt.c
src/include/postmaster/interrupt.h
```
handler：
```text
SignalHandlerForConfigReload()
  -> ConfigReloadPending = true
  -> SetLatch(MyLatch)
```
消费位置：
```text
PostgresMain() command loop
ProcessMainLoopInterrupts()
autovacuum launcher / worker 专用循环
walsender / walreceiver / logical worker 专用点
postmaster 的 process_pm_reload_request()
```
边界：不设置 `InterruptPending`，不由 `CHECK_FOR_INTERRUPTS()` 直接消费，多次 SIGHUP 可合并，不同进程消费时间不同，是否生效取决于 GUC context。
### holdoff counters
`miscadmin.h` 暴露三类 counter。
```text
InterruptHoldoffCount:
  阻止 cancel 和 die 被实际处理。
QueryCancelHoldoffCount:
  只阻止 query cancel，die 仍可处理。
CritSectionCount:
  表示处在不能安全 ERROR/FATAL 的关键区；
  严重错误通常要升级为 PANIC。
```
`SocketBackend()` 读 FE/BE 消息时使用 `HOLD_CANCEL_INTERRUPTS()`，只 hold cancel，不 hold die。
原因是 query cancel 不能破坏消息边界，但终止连接可以停止继续读。
### `MyLatch`
`globals.c` 注释说明 `MyLatch` 指向当前进程 signal handling 使用的 latch；没有 `PGPROC` 时是 local latch，有 `PGPROC` 后通常是 `PGPROC->procLatch`。
handler 设置 pending flag 后调用 `SetLatch(MyLatch)`。
`SetLatch()` 不携带原因，也不是消息队列。
它只保证等待中的 backend 尽快醒来重新检查 flag。
## 5. 主流程 walkthrough: SIGINT 取消查询
假设会话 A 正在执行：
```sql
SELECT pg_sleep(60);
```
会话 B 调用：
```sql
SELECT pg_cancel_backend(<pid>);
```
第一步，定位目标 backend。
`procsignal.c` 的 `SendCancelRequest()` 扫描 `ProcSignal` slot。
它会读 `pss_pid`，加 `pss_mutex` 后重新确认 pid，比较 `pss_cancel_key_len` 和 `pss_cancel_key`，匹配后 `kill(pid, SIGINT)`。
源码注释承认这里有 race：backend 可能退出，PID 可能被 OS 复用，但 cancel key 错配概率极低；发送 signal 本身也天然受 PID race 影响。
第二步，目标 backend 的 handler 设置 flag。
`PostgresMain()` 安装：
```text
pqsignal(SIGINT, StatementCancelHandler)
```
`StatementCancelHandler()` 做：
```text
if (!proc_exit_inprogress)
{
    InterruptPending = true;
    QueryCancelPending = true;
}
SetLatch(MyLatch);
```
第三步，执行路径到达安全点。
执行器、排序、COPY、VACUUM、索引构建和各种长循环会定期调用：
```text
CHECK_FOR_INTERRUPTS();
```
宏逻辑：
```text
if (INTERRUPTS_PENDING_CONDITION())
    ProcessInterrupts();
```
没有 pending interrupt 时，成本主要是一个 unlikely 分支。
第四步，`ProcessInterrupts()` 判断是否允许处理。
```text
if (InterruptHoldoffCount != 0 || CritSectionCount != 0)
    return;
InterruptPending = false;
```
如果当前不能安全处理，`QueryCancelPending` 留着，稍后再来。
第五步，cancel 分支转成 ERROR。
```text
if (QueryCancelPending && QueryCancelHoldoffCount != 0)
    InterruptPending = true;
else if (QueryCancelPending)
{
    QueryCancelPending = false;
    lock_timeout_occurred = get_timeout_indicator(LOCK_TIMEOUT, true);
    stmt_timeout_occurred = get_timeout_indicator(STATEMENT_TIMEOUT, true);
    LockErrorCleanup();
    ereport(ERROR, ...);
}
```
普通用户 cancel 的结果是：
```text
canceling statement due to user request
```
statement timeout 和 lock timeout 会使用不同错误消息。
如果两个 timeout indicator 都已触发，源码会比较 finish time，报告更早完成的 timeout，平局倾向 lock timeout。
第六步，ERROR 进入顶层恢复。
`ereport(ERROR)` 不返回原调用点继续执行，而是 longjmp 到 `PostgresMain()` 外层 `sigsetjmp`。
顶层恢复会 hold interrupts，disable timeouts，清 `QueryCancelPending`，清 `DoingCommandRead`，重置 libpq 通信状态，发送错误报告，`AbortCurrentTransaction()`，`PortalErrorCleanup()`，释放 top-level replication slot，清理 JIT / ErrorContext，必要时进入 extended protocol 的 `ignore_till_sync`。
结论：
```text
handler 只设 flag；
ProcessInterrupts() 只把 cancel 转成 ERROR；
事务和资源 cleanup 属于顶层错误恢复。
```
## 6. 主流程 walkthrough: SIGTERM 终止 backend
普通 backend 在 `PostgresMain()` 中安装：
```text
pqsignal(SIGTERM, die)
```
`die()` 做：
```text
if (!proc_exit_inprogress)
{
    InterruptPending = true;
    ProcDiePending = true;
    if (ProcDieSenderPid == 0)
        记录 sender pid / uid;
}
pgStatSessionEndCause = DISCONNECT_KILLED;
SetLatch(MyLatch);
```
进入 `ProcessInterrupts()` 后，`ProcDiePending` 最先处理。
它会保存 sender pid / uid，清 `ProcDiePending`，清 sender 信息，清 `QueryCancelPending`，调用 `LockErrorCleanup()`，然后按进程类型 `FATAL` 或 `proc_exit()`。
不同进程的结果不同。
```text
普通 backend:
  FATAL "terminating connection due to administrator command"
autovacuum worker:
  FATAL "terminating autovacuum process due to administrator command"
logical launcher:
  DEBUG1 后 proc_exit(1)，让 background worker restart
IO worker:
  DEBUG1 后 proc_exit(0)
ClientAuthInProgress:
  避免向 client 冒险发送不安全输出
```
`LockErrorCleanup()` 在 FATAL 前执行，因为 backend 可能正在 lock wait queue 中。
FATAL 之后进入进程退出路径。
退出 callback 继续清理事务状态、ResourceOwner、ProcArray membership、ProcSignal slot、`PGPROC` / latch / semaphore、锁、buffer、DSM 和临时文件。
`ProcDiePending` 和 `QueryCancelPending` 的本质差异：
```text
SIGINT:
  当前 statement / transaction attempt 结束，session 通常继续。
SIGTERM:
  session lifecycle 结束，backend 退出。
```
## 7. 主流程 walkthrough: SIGHUP 配置重载
SIGHUP 有 postmaster 路径和子进程路径。
postmaster 安装：
```text
pqsignal(SIGHUP, handle_pm_reload_request_signal)
```
handler 只做：
```text
pending_pm_reload_request = true;
SetLatch(MyLatch);
```
postmaster 主循环稍后调用 `process_pm_reload_request()`。
主流程：
```text
pending_pm_reload_request = false
LOG "received SIGHUP, reloading configuration files"
ProcessConfigFile(PGC_SIGHUP)
SignalChildren(SIGHUP, btmask_all_except(B_DEAD_END_BACKEND))
load_hba()
load_ident()
reload SSL config
EXEC_BACKEND 下 write_nondefault_variables(PGC_SIGHUP)
```
普通 backend、bgwriter、walwriter、autovacuum、walsender 等进程通常安装：
```text
pqsignal(SIGHUP, SignalHandlerForConfigReload)
```
handler：
```text
ConfigReloadPending = true;
SetLatch(MyLatch);
```
普通 backend 在 `PostgresMain()` command loop 中消费：
```text
DoingCommandRead = true;
firstchar = ReadCommand(&input_message);
disable idle timeouts;
CHECK_FOR_INTERRUPTS();
DoingCommandRead = false;
if (ConfigReloadPending)
{
    ConfigReloadPending = false;
    ProcessConfigFile(PGC_SIGHUP);
}
```
这里先处理 cancel/die，再清 `DoingCommandRead`，再处理 reload，再处理新命令。
后台进程常见模式：
```text
for (;;)
{
    ResetLatch(MyLatch);
    ProcessMainLoopInterrupts();
    do_one_cycle();
    WaitLatch(...);
}
```
`ProcessMainLoopInterrupts()` 会处理 barrier、config reload、shutdown request 和 log memory context request。
`ProcessConfigFile()` 在 `guc-file.l` 中。
它创建临时 memory context `"config file processing"`，调用 `ProcessConfigFileInternal(context, true, elevel)`，再删除 context。
这样 repeated SIGHUP 中的临时分配不会堆在进程生命周期 context 里。
`ProcessConfigFileInternal()` 在 `guc.c` 中完成解析和应用：解析 `postgresql.conf`，解析 `postgresql.auto.conf`，标记 `GUC_IS_IN_FILE`，处理从文件中移除的设置，调用 `set_config_option()` 应用可运行时变化的参数，对 postmaster-only 参数设置 `GUC_PENDING_RESTART`，记录 `PgReloadTime`，记录配置错误或 unaffected changes。
SIGHUP reload 的不变量：
```text
postmaster 负责传播；
每个进程自己 reload；
没有全局 atomic cutover；
没有所有 backend reload 完成的确认点；
GUC context 决定当前进程能否应用。
```
## 8. ProcSignal: SIGUSR1 多路复用
PostgreSQL 还有很多 backend 间通知：sinval catchup、LISTEN/NOTIFY、parallel message、walsender stopping、global barrier、log memory context、slot sync message、recovery conflict。
这些不各自占用一个 OS signal。
`procsignal.c` 用 shared memory reason flag 复用 `SIGUSR1`。
```text
发送方:
  找到目标 ProcSignalSlot；
  在 pss_mutex 下设置 pss_signalFlags[reason]；
  kill(pid, SIGUSR1)。
接收方:
  procsignal_sigusr1_handler()
    -> CheckProcSignal(reason)
    -> 调用对应 Handle...Interrupt()
    -> SetLatch(MyLatch)。
```
`procsignal.h` 明确说明：不同 reason 可以并发设置；同一 reason 快速重复发送，接收者可能只观察到一次。
这和本节模型一致：
```text
signal 不是队列；
pending flag 不是计数器；
真实语义必须能靠状态重查恢复。
```
### ProcSignal 启动边界
普通 backend 在 `InitPostgres()` 中：
```text
HOLD_INTERRUPTS();
ProcSignalInit(MyCancelKey, MyCancelKeyLength);
InitLocalDataChecksumState();
RESUME_INTERRUPTS();
```
注册 ProcSignal 后，本进程可能收到 barrier。
但本地 checksum cache 尚未初始化前不能吸收 barrier，否则状态转换不合法。
所以注册和本地状态初始化之间用 `HOLD_INTERRUPTS()` 包住。
### ProcSignal cleanup
`ProcSignalInit()` 注册：
```text
on_shmem_exit(CleanupProcSignalState, 0)
```
退出时 `CleanupProcSignalState()` 会清 `MyProcSignalSlot`，加 `pss_mutex`，确认 `pss_pid` 是 `MyProcPid`，把 `pss_pid` 置 0，清 cancel key 长度，把 `pss_barrierGeneration` 设成 `PG_UINT64_MAX`，释放 spinlock，并 broadcast `pss_barrierCV`。
`pss_barrierGeneration = PG_UINT64_MAX` 的含义是：退出进程不应阻塞未来 `WaitForProcSignalBarrier()`。
### barrier 的更强语义
普通 reason flag 可以合并。
barrier 不行，因为发起者可能要等所有 backend 已吸收某类全局状态变化。
`EmitProcSignalBarrier(type)` 会给所有 slot 的 `pss_barrierCheckMask` 设置 type bit，递增全局 `psh_barrierGeneration`，给有 pid 的 slot 设置 `PROCSIG_BARRIER` 并发送 `SIGUSR1`。
接收端 handler 只做：
```text
InterruptPending = true;
ProcSignalBarrierPending = true;
```
实际处理在 `ProcessProcSignalBarrier()`：读取 local/shared generation，exchange 清 `pss_barrierCheckMask`，在 `PG_TRY` 中逐个处理 barrier type，失败或 ERROR 时 `ResetProcSignalBarrierBits(flags)`，成功后写 `pss_barrierGeneration = shared_gen` 并 broadcast CV。
signal handler 不直接读 64-bit generation，因为某些平台 64-bit atomic 可能用 spinlock emulation。
barrier 也不适合高频路径；fan-out 成本随 `MaxBackends` 和慢 backend 数放大。
## 9. client read/write 边界
FE/BE 协议是中断处理最敏感的边界之一。
如果在读取一个 frontend message 的中间抛 `ERROR`，backend 可能不知道当前 message 在哪里结束。
因此 `SocketBackend()` 读消息时用：
```text
HOLD_CANCEL_INTERRUPTS();
pq_startmsgread();
qtype = pq_getbyte();
pq_getmessage(inBuf, maxmsglen);
RESUME_CANCEL_INTERRUPTS();
```
`PostgresMain()` 等命令时设置：
```text
DoingCommandRead = true;
firstchar = ReadCommand(&input_message);
...
CHECK_FOR_INTERRUPTS();
DoingCommandRead = false;
```
`ProcessInterrupts()` 的 cancel 分支看到 `DoingCommandRead` 时不会给 idle session 发送普通 cancel ERROR。
所以 `pg_cancel_backend(idle_backend_pid)` 可能返回 true，但目标 session 没有报错，下一条 query 正常执行。
这不是丢 signal，而是 query cancel 的语义：取消正在执行的 query，不是终止 session。
client write 也有特殊处理。
如果 backend 正在向不读数据的客户端写大量结果，又收到 `SIGTERM`，继续试图发送 FATAL 响应可能永远卡在 write。
`ProcessClientWriteInterrupt(true)` 会在可安全处理 `ProcDiePending` 时：
```text
whereToSendOutput = DestNone;
CHECK_FOR_INTERRUPTS();
```
这样 backend 可以退出，不再为了向坏连接发送错误消息而阻塞。
`pqcomm.c` 还有非 signal 路径。
发送失败时会设置：
```text
ClientConnectionLost = true;
InterruptPending = true;
```
下一次 `ProcessInterrupts()` 会转成 `whereToSendOutput = DestNone` 和 `ereport(FATAL, "connection to client lost")`。
所以 `ProcessInterrupts()` 已经是 backend 异步事件的统一安全点，不只是 OS signal dispatcher。
## 10. 生命周期 / ownership / cleanup
backend-local pending flag 在 `globals.c` 中定义。
每个进程有自己的 `InterruptPending`、`QueryCancelPending`、`ProcDiePending`、`ConfigReloadPending`、`ShutdownRequestPending`、`ProcSignalBarrierPending` 和 `ClientConnectionLost`。
`ProcSignalSlot` 是 shared memory。
postmaster 初始化数组，backend / auxiliary process 调用 `ProcSignalInit()` 发布 pid 和 cancel key。
pending flag 的 owner 是当前进程。
其它进程不能直接清你的 `QueryCancelPending`。
其它进程通常只能发 OS signal、设置 ProcSignal reason flag，或通过 postmaster 传播 SIGHUP / shutdown。
pending flag 不释放，只在消费或错误恢复中重置。
典型重置点：
```text
ProcessInterrupts() 清 QueryCancelPending；
PostgresMain() 错误恢复清 QueryCancelPending；
ProcessInterrupts() 清 ProcDiePending 后通常 FATAL；
PostgresMain() 或后台循环清 ConfigReloadPending；
ProcessProcSignalBarrier() 清 ProcSignalBarrierPending。
```
普通 cancel 的 cleanup owner 是顶层错误恢复。
`PostgresMain()` 的外层 `sigsetjmp` 负责 hold interrupts、disable timeouts、清 pending cancel、重置 FE/BE 通信状态、发送错误报告、`AbortCurrentTransaction()`、`PortalErrorCleanup()`、释放 top-level replication slot、清理 JIT / ErrorContext、恢复可处理中断。
后台进程也有自己的 outer `sigsetjmp`。
例如 bgwriter 在 ERROR 后会释放 LWLock、CV sleep、AIO、buffer、SMgr、文件和 hash table 相关状态，再 reset 自己的 memory context。
`ereport(FATAL)` 进入进程退出。
退出 callback 负责 `before_shmem_exit`、`on_shmem_exit`、ProcSignal slot cleanup、ProcArray / PGPROC cleanup、lock / buffer / DSM / temp file cleanup 和 `proc_exit` finalization。
`quickdie()` 是例外。
`SIGQUIT` handler 会 `_exit(2)`，故意不运行 `proc_exit()` 或 `atexit()` callback。
原因是 SIGQUIT 通常表示 shared memory 可能已经不可信；正确动作是快速离开，让 postmaster crash/reset cycle 重建共享状态。
## 11. 正确性机制层次
signal-safety 层保证不会在任意指令边界执行复杂数据库逻辑，代价是事件可合并、flag 不记录次数、处理延迟依赖下一次安全点。
latch 层保证等待中的 backend 能尽快醒来重查 pending flag，但不保证唤醒原因、唤醒次数或业务状态仍然为真。
holdoff 层保证不会在协议半读、critical section、资源半更新时抛 ERROR/FATAL，代价是 cancel / terminate 可能延迟，且 SQL 视图通常看不到 holdoff counter。
ERROR/FATAL cleanup 层保证 cancel 依赖 ERROR 路径收事务和资源，terminate 依赖 FATAL/proc_exit 路径收 session 和 shared state。
GUC reload 层保证配置解析、内存分配、assign hook 和日志记录不发生在 signal handler，但不保证所有参数可运行时变更，也不保证所有进程同时采用新值。
ProcSignal barrier 层保证需要时可以等待所有参与 ProcSignal 的 backend 吸收某类全局变化，代价是 fan-out 到 `MaxBackends`，慢 backend 会拖住等待者。
## 12. 错误路径 / fallback
cancel 到达但当前不能处理：
```text
状态:
  QueryCancelPending = true
  QueryCancelHoldoffCount != 0
处理:
  ProcessInterrupts()
    -> InterruptPending = true
    -> return
现象:
  客户端感觉 cancel 延迟；
  日志通常没有“holdoff 中”的默认记录；
  需要 gdb、trace 或源码判断。
```
idle command read 中收到 cancel：
```text
状态:
  DoingCommandRead = true
  QueryCancelPending = true
处理:
  普通 user cancel 不抛 ERROR；
  QueryCancelPending 被清；
  session 继续等待下一条命令。
现象:
  pg_cancel_backend() 可能返回 true；
  目标 session 无可见错误。
```
client write blocked 中收到 terminate：
```text
状态:
  backend 正在写结果
  客户端不读
  ProcDiePending = true
fallback:
  ProcessClientWriteInterrupt(true)
    -> whereToSendOutput = DestNone
    -> CHECK_FOR_INTERRUPTS()
    -> FATAL / exit
现象:
  server log 有 terminate；
  client 可能只看到连接断开。
```
config reload 文件有错误：
`PGC_POSTMASTER` startup 中硬错误会导致启动失败。
`PGC_SIGHUP` reload 中，错误通常记录日志，已安全应用的 unaffected changes 可能保留，不能运行时改变的参数会变成 pending restart。
ProcSignal barrier 处理失败：
```text
ProcessProcSignalBarrier()
  -> ResetProcSignalBarrierBits(flags)
  -> ProcSignalBarrierPending = true
  -> InterruptPending = true
```
之后每次安全点都可能重试。
`WaitForProcSignalBarrier()` 每 5 秒可能记录还在等待哪个 PID 接受 barrier。
SIGQUIT quickdie：
```text
HOLD_INTERRUPTS();
尽量向客户端发 warning；
_exit(2)。
```
它不 abort transaction，不清 ProcSignal slot，不释放 ResourceOwner。
正确性依赖 postmaster 后续重置共享内存。
## 13. 成本、资源与跨模块传播
`CHECK_FOR_INTERRUPTS()` 在无 pending 情况下主要是读 `InterruptPending` 和一个 unlikely 分支。
成本随 tuple 数、loop iteration、COPY 行数、index tuple 数、逻辑复制处理量和扩展代码 CPU loop 次数增长。
长 CPU loop 要定期检查；极短 hot path 不要滥加；持 spinlock、critical section 或不可 longjmp 区域不能调用。
一次 SIGHUP reload 可能导致 postmaster 解析配置、给所有子进程发 SIGHUP、每个子进程各自解析配置、每个子进程运行相关 GUC assign hook、某些模块重建本地状态或释放 waiters。
成本随 backend 数、include 文件数、GUC 数、hook 复杂度和日志设置增长。
典型跨模块例子：walsender reload 后调用 `SyncRepInitConfig()`；同步复制要求变化时 `SyncRepReleaseWaiters()`。
`SendProcSignal()` 如果没有 `ProcNumber`，可能扫描 `NumProcSignalSlots`。
`EmitProcSignalBarrier()` 更重：设置所有 slot 的 barrier bit，递增全局 generation，向所有有 pid 的 slot 发 `SIGUSR1`，可选等待所有 generation 追上。
barrier 成本随 `MaxBackends` 和慢 backend 数放大，因此不适合频繁路径。
本节连接的模块边界：
| 模块 | 连接点 | 边界 |
| --- | --- | --- |
| latch | `SetLatch(MyLatch)` | wakeup，不携带语义。 |
| GUC | `ProcessConfigFile(PGC_SIGHUP)` | reload 在进程上下文，不在 handler。 |
| lock manager | `LockErrorCleanup()` | cancel/die 前收 lock wait 状态。 |
| transaction | `AbortCurrentTransaction()` | ERROR cleanup，不由 handler 直接做。 |
| libpq protocol | `DoingCommandRead`、`HOLD_CANCEL_INTERRUPTS()` | 保护 FE/BE message 边界。 |
| ProcSignal | `SIGUSR1` reason flag | 多路复用、可合并、barrier 才有确认。 |
| postmaster | `SignalChildren(SIGHUP)` | 传播信号，不保证子进程同刻 reload。 |
## 14. 观测与诊断入口
配置 reload：
```sql
SELECT pg_reload_conf();
SELECT name, setting, source, pending_restart
FROM pg_settings
WHERE name IN ('work_mem', 'log_min_duration_statement');
SELECT name, applied, error
FROM pg_file_settings
WHERE error IS NOT NULL OR NOT applied;
```
典型日志：
```text
received SIGHUP, reloading configuration files
parameter "..." changed to "..."
parameter "..." cannot be changed without restarting the server
configuration file "..." contains errors
```
cancel / terminate：
```sql
SELECT pg_cancel_backend(pid);
SELECT pg_terminate_backend(pid);
```
典型结果：
```text
canceling statement due to user request
canceling statement due to statement timeout
canceling statement due to lock timeout
terminating connection due to administrator command
connection to client lost
```
当前活动：
```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE pid = <target_pid>;
```
SQL 通常看不到：
```text
InterruptPending
QueryCancelPending
ProcDiePending
ConfigReloadPending
InterruptHoldoffCount
QueryCancelHoldoffCount
CritSectionCount
MyLatch 是否 set
ProcSignalSlot.p ss_signalFlags[] 的完整历史
```
这些状态需要 gdb、临时 DEBUG 日志、tracepoint、perf、`strace -e signal` 或源码调用链推断。
gdb 断点建议：
```text
普通 cancel:
  break StatementCancelHandler
  break ProcessInterrupts
  break AbortCurrentTransaction
SIGHUP:
  break SignalHandlerForConfigReload
  break ProcessConfigFile
  break ProcessConfigFileInternal
ProcSignal barrier:
  break procsignal_sigusr1_handler
  break HandleProcSignalBarrierInterrupt
  break ProcessProcSignalBarrier
```
在 `ProcessInterrupts()` 中可观察：
```text
p InterruptPending
p QueryCancelPending
p ProcDiePending
p QueryCancelHoldoffCount
p DoingCommandRead
```
`pg_stat_activity.wait_event` 能说明 backend 当前等在哪里，但不能说明 signal 是否已经到达、pending flag 是否被 holdoff、哪一次 `SetLatch()` 唤醒了 backend、SIGUSR1 的 reason 是 notify 还是 barrier、`ConfigReloadPending` 是否尚未消费。
诊断 cancel 延迟时，应把 wait event 当线索，而不是完整因果。
## 15. 常见误区
1. 误以为 signal handler 直接取消查询。实际 handler 只设 flag，取消发生在安全点，cleanup 发生在 ERROR/FATAL 路径。
2. 误以为 `CHECK_FOR_INTERRUPTS()` 可放在任何位置。实际它可能 longjmp，不能放在持锁短临界区、critical section、协议半读或不可 ERROR 的 cleanup callback 中。
3. 误以为 SIGHUP 全局立即生效。实际 postmaster 传播信号，每个进程在自己的 convenient point reload。
4. 误以为 `SetLatch(MyLatch)` 表示消息送达。实际 latch 只叫醒进程重新检查 pending flag。
5. 误以为 `pg_cancel_backend()` 返回 true 就表示目标 query 已取消。实际目标可能 idle、已结束、处于 holdoff，或稍后才处理。
6. 误以为 SIGTERM 只是更严重的 SIGINT。实际 SIGINT 是 statement/transaction attempt 边界，SIGTERM 是 session lifecycle 边界。
7. 误以为 SIGUSR1 有固定含义。实际 reason 存在 `ProcSignalSlot`，普通 reason 可合并。
8. 误以为 wait event 可解释所有 cancel 延迟。实际 pending flag、holdoff counter 和 latch state 多数不可直接观测。
## 16. 课堂实验
### 实验一：观察普通 query cancel
会话 A：
```sql
SELECT pg_backend_pid();
SELECT pg_sleep(60);
```
会话 B：
```sql
SELECT pg_cancel_backend(<pid>);
```
预期：会话 A 报 `canceling statement due to user request`；连接仍可用；事务块中需要 `ROLLBACK`。
源码跟读：
```text
procsignal.c: SendCancelRequest()
postgres.c: StatementCancelHandler()
miscadmin.h: CHECK_FOR_INTERRUPTS()
postgres.c: ProcessInterrupts()
postgres.c: PostgresMain() error recovery
```
要画出的状态变化：
```text
InterruptPending false -> true -> false
QueryCancelPending false -> true -> false
active transaction -> ERROR abort -> idle/aborted state
```
### 实验二：观察 SIGTERM 压过 cancel
会话 A：
```sql
SELECT pg_backend_pid();
SELECT pg_sleep(60);
```
会话 B：
```sql
SELECT pg_cancel_backend(<pid>);
SELECT pg_terminate_backend(<pid>);
```
或 shell：
```bash
kill -INT <pid>
kill -TERM <pid>
```
源码跟读：
```text
postgres.c: die()
postgres.c: ProcessInterrupts()
  -> ProcDiePending branch
  -> QueryCancelPending = false
  -> LockErrorCleanup()
  -> FATAL / proc_exit
```
预期：连接关闭；server log 出现 administrator command；不会稳定先返回普通 cancel 后继续 session。
### 实验三：观察 SIGHUP reload
执行：
```sql
SELECT pg_reload_conf();
SELECT name, setting, source, pending_restart
FROM pg_settings
WHERE name = 'work_mem';
SELECT name, applied, error
FROM pg_file_settings
WHERE name = 'work_mem';
```
源码跟读：
```text
postmaster.c: handle_pm_reload_request_signal()
postmaster.c: process_pm_reload_request()
postmaster/interrupt.c: SignalHandlerForConfigReload()
postgres.c: ConfigReloadPending in command loop
guc-file.l: ProcessConfigFile()
guc.c: ProcessConfigFileInternal()
```
观察 postmaster log、`pg_file_settings.applied/error`、`pg_settings.pending_restart`，以及长时间运行 query 是否立刻表现出新配置。
结论：`pg_reload_conf()` 成功不等于所有 backend 已 reload；postmaster-only GUC 需要 restart；每个进程有自己的 reload 点。
### 实验四：idle cancel 与 `DoingCommandRead`
让会话 A idle，gdb attach 到它的 backend pid。
断点：
```text
break ProcessInterrupts
```
会话 B：
```sql
SELECT pg_cancel_backend(<pid>);
```
观察：
```text
p DoingCommandRead
p QueryCancelPending
p QueryCancelHoldoffCount
```
预期：idle command read 中普通 cancel 不产生 user-visible ERROR。
### 实验五：ProcSignal barrier 源码练习
画出：
```text
EmitProcSignalBarrier(type)
  -> pss_barrierCheckMask OR bit
  -> psh_barrierGeneration++
  -> set PROCSIG_BARRIER
  -> kill(SIGUSR1)
procsignal_sigusr1_handler()
  -> HandleProcSignalBarrierInterrupt()
  -> InterruptPending = true
  -> ProcSignalBarrierPending = true
CHECK_FOR_INTERRUPTS()
  -> ProcessInterrupts()
  -> ProcessProcSignalBarrier()
  -> pss_barrierGeneration = shared_gen
  -> ConditionVariableBroadcast()
```
回答：为什么普通 reason 可合并；为什么 barrier 需要 generation；为什么 handler 不直接读 shared generation。
## 17. 源码练习
练习一：找所有写 `QueryCancelPending = true` 的地方。
```bash
rg -n "QueryCancelPending = true" /home/nail/postgres/src
```
按 OS SIGINT、timeout、模块内部 interrupt、autovacuum / background worker、replication 相关路径分类。
练习二：找所有消费 `ConfigReloadPending` 的地方。
```bash
rg -n "ConfigReloadPending" /home/nail/postgres/src/backend
```
回答哪些进程用 `ProcessMainLoopInterrupts()`，哪些进程有额外 reload 行为，哪些进程只在特定 wait loop 中检查。
练习三：判断一个新 C 扩展循环是否需要 `CHECK_FOR_INTERRUPTS()`。
场景：
```text
扫描百万级数组；
不持 LWLock；
不处于 critical section；
用户希望 pg_cancel_backend() 快速生效。
```
建议：在外层循环定期检查；不要在短临界区检查；确认 ERROR 后 MemoryContext / ResourceOwner 能兜底；不要在不可 longjmp 的 callback 中检查。
练习四：判断 GUC assign hook 是否能做重活。
讨论：assign hook 不在 signal handler 中执行，但可能在 postmaster 或无数据库连接的后台进程中执行；它会被多个进程分别调用；等待锁或访问 catalog 可能引入 reload 延迟、上下文不适用或死锁风险。
## 18. 讨论题
1. 为什么 `ProcDiePending` 要在 `QueryCancelPending` 前处理？
2. `InterruptPending` 已经为 true 时又收到 SIGINT，会不会记录两次 cancel？
3. 为什么 `SocketBackend()` 只 `HOLD_CANCEL_INTERRUPTS()`，不 `HOLD_INTERRUPTS()`？
4. 为什么 `ConfigReloadPending` 不直接放进 `ProcessInterrupts()` 里处理？
5. `ProcessConfigFile()` 使用临时 memory context 解决什么生命周期问题？
6. 一个后台进程只 `pg_usleep()`、不 `WaitLatch()`，SIGHUP 和 shutdown 会有什么延迟问题？
7. 普通 ProcSignal reason 可以合并，barrier 为什么还需要 generation？
8. `pg_stat_activity.wait_event` 能帮助定位 cancel 延迟，但不能告诉你哪些状态？
9. `quickdie()` 为什么不运行 `proc_exit()` callback？
10. 要请求所有 backend 刷新本地缓存时，你会选 SIGHUP、ProcSignal reason、ProcSignal barrier 还是 shared invalidation？
## 19. 本节小结
核心链路：
```text
OS signal 或 ProcSignal reason
  -> backend-local pending flag
  -> SetLatch(MyLatch)
  -> CHECK_FOR_INTERRUPTS() 或主循环 convenient point
  -> ERROR / FATAL / proc_exit / ProcessConfigFile / ProcessProcSignalBarrier
  -> 顶层 cleanup 或继续运行
```
`SIGINT` 的 mental model：
```text
QueryCancelPending 不是错误本身；
它是“下一次安全点应取消当前工作”的请求；
真正 cleanup 由 ERROR 路径完成。
```
`SIGTERM` 的 mental model：
```text
ProcDiePending 改变 session 生命周期；
它压过 QueryCancelPending；
通常进入 FATAL 或 proc_exit。
```
`SIGHUP` 的 mental model：
```text
ConfigReloadPending 是每进程 reload 提示；
postmaster 负责传播；
每个进程在自己的主循环点 ProcessConfigFile(PGC_SIGHUP)；
不存在全局原子 reload 完成点。
```
`CHECK_FOR_INTERRUPTS()` 的 mental model：
```text
它是安全边界，不是普通 callback；
调用点必须允许 ERROR/FATAL longjmp；
holdoff 和 critical section 决定 pending interrupt 是否能被消费。
```
ownership / cleanup：
```text
pending flag 是 backend-local；
ProcSignalSlot 是 shared state；
cancel ERROR 由 PostgresMain() 顶层错误恢复收尾；
terminate FATAL 由 proc_exit / shmem exit callback 收尾；
quickdie() 故意跳过精细 cleanup。
```
观测边界：
```text
日志和 SQL 能看到结果；
pg_stat_activity 能看到 state / wait_event；
pg_file_settings 和 pg_settings 能看到配置应用结果；
pending flag、holdoff counter、latch state 和 signal coalescing
通常只能通过源码、gdb、trace 或推断看到。
```
可迁移规律：
```text
异步事件处理的关键不是尽快执行 handler，
而是把任意时刻到达的事实压缩成可合并状态，
并在系统状态自洽的边界上消费它。
```
判断边界：响应延迟依赖 workload、循环检查频率、wait primitive、客户端 I/O、holdoff 时间、critical section 长度和版本实现；不要把一次 cancel 延迟简单归因成 signal 丢失或 PostgreSQL 不响应。
