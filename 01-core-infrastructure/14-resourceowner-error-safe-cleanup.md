# PostgreSQL ERROR-safe cleanup、PG_TRY 与诊断边界

## 课程定位

前置知识：已经理解 `ResourceOwner` tree 的 ownership 边界、`Enlarge()` / `Remember()` / `Forget()` 的获取安全、事务 / 子事务 / Portal owner 的传播，以及 `ResourceOwnerRelease()` 的三阶段释放顺序。

本节唯一主问题：

```text
PG_TRY / PG_CATCH 负责恢复 CurrentResourceOwner 等全局执行状态，事务 abort 和 ResourceOwnerRelease() 负责兜底释放资源；
commit warning、ResourceOwnerDesc callback 和 gdb / 日志能看到什么、看不到什么？
```

核心矛盾：`ERROR` 在 PostgreSQL 里是 `longjmp` 风格的非局部退出。它必须快速逃离任意深度的 executor / access method / cache / lock 代码；但逃离之后，backend 还要继续服务同一个 session。于是系统不能要求每一层 C 栈都写完美的 unwind 代码，也不能把所有清理都丢给最外层一把扫。PostgreSQL 的选择是把错误恢复拆成两类：

```text
PG_TRY / PG_CATCH:
  恢复可继续执行所需的 backend-local 全局指针和局部状态。

AbortCurrentTransaction() + ResourceOwnerRelease():
  按事务边界释放 query-lifespan 外部资源，并让 owner tree 兜底。
```

学完后应能判断：什么时候在 `PG_CATCH` 里只恢复 `CurrentResourceOwner` / `ActivePortal` / `PortalContext` 并 `PG_RE_THROW()`；什么时候必须依赖事务 abort；什么时候要用 `ResourceOwnerDesc` 而不是老式 release callback；为什么 commit 泄漏只打 `WARNING` 不打 `ERROR`；以及日志、gdb、`ResourceOwnerDesc.DebugPrint` 能帮你定位什么，不能证明什么。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

这一组 ResourceOwner 课到这里已经形成完整闭环：

```text
第 10 课:
  为什么外部资源不交给 MemoryContext，而要挂 ResourceOwner？

第 11 课:
  获取资源时怎样避免“资源已获取但还没记账”窗口？

第 12 课:
  资源挂到 top transaction、subtransaction 还是 Portal owner？

第 13 课:
  owner 结束时，为什么要按 before-locks / locks / after-locks 释放？

第 14 课:
  ERROR 发生时，谁恢复执行状态，谁兜底释放资源，诊断信息边界在哪里？
```

本节不是再讲一遍三阶段 release，而是回答一个真实调试问题：

```text
SQL 执行中途 ERROR 后，backend 为什么还能继续处理下一条命令？
```

答案不是“C 语言异常自动析构”，也不是“PG_CATCH 释放所有资源”。PostgreSQL 的错误恢复分成几层：

```text
局部 PG_CATCH:
  把 ActivePortal、CurrentResourceOwner、PortalContext、CurrentMemoryContext 等指针恢复到进入局部执行前的值。

顶层错误恢复:
  EmitErrorReport()
  AbortCurrentTransaction()
  PortalErrorCleanup()
  FlushErrorState()

事务 abort:
  AtAbort_Memory()
  AtAbort_ResourceOwner()
  LWLockReleaseAll()
  LockErrorCleanup()
  AtAbort_Portals()
  ResourceOwnerRelease(... false ...)
  CleanupTransaction()

ResourceOwner:
  对仍挂在 owner tree 上的 buffer pin、lock、catcache ref、snapshot、file 等资源做兜底释放。
```

这几层边界不能混在一起。`PG_CATCH` 如果顺手释放一堆事务资源，容易破坏事务状态机；`AbortTransaction()` 如果不知道当前局部 `CurrentResourceOwner` 被谁改过，就可能把后续 cleanup 记到错误 owner；`ResourceOwnerRelease()` 如果在 callback 里做高层逻辑或分配内存，abort 自身也可能再次出错。

本节的主线就是：

```text
ERROR 非局部退出
  -> 局部恢复执行指针
  -> 顶层进入事务 abort
  -> ResourceOwner 兜底释放
  -> 日志 / gdb / DebugPrint 只暴露部分状态
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ereport(ERROR) 通过 PG_exception_stack longjmp 到最近的 handler；
局部 handler 只恢复执行上下文并重新抛出；
最外层 postgres.c 报错、abort 当前事务、清理 Portal 和 ErrorContext；
xact.c 在 abort 中把 CurrentResourceOwner 重置到事务 owner，并调用 ResourceOwnerRelease(... isCommit=false ...) 静默释放遗留资源。
```

本节 tension 是：

```text
错误路径必须足够短，能从任意源码位置逃出来
  vs
backend 必须在错误后保持可复用，不能留下 pin、lock、snapshot、file、Portal、ErrorContext 等悬挂状态
```

PostgreSQL 的分工是：

| 层次 | 负责什么 | 不负责什么 |
| --- | --- | --- |
| `elog.c` / `PG_TRY` | 建立 `sigsetjmp` handler、恢复 error context stack、传递错误控制流。 | 不理解 buffer、lock、snapshot、Portal 语义。 |
| 局部 `PG_CATCH` | 恢复自己临时改过的全局指针和局部状态。 | 不做事务级资源释放。 |
| `postgres.c` 顶层恢复 | 报告错误，调用 `AbortCurrentTransaction()`，清理 Portal 和 ErrorContext，回到命令循环。 | 不逐类释放 buffer pin / lock / cache ref。 |
| `xact.c` abort | 切到 abort memory context，释放低层等待 / 锁等待 / LWLock，按事务状态机做 abort。 | 不知道每个资源类型的具体释放函数。 |
| `resowner.c` | 遍历 owner tree，按 phase / priority 调用资源 callback，commit 时打印泄漏 warning。 | callback 不应做用户可见、可能失败的高层操作。 |

这套设计的关键是：

```text
ERROR-safe cleanup 不是一个函数，而是一组边界清楚的恢复协议。
```

所以调试时也要沿协议看：

```text
PG_CATCH 有没有恢复它改过的全局指针？
事务 abort 有没有重新建立 CurrentResourceOwner？
资源有没有被 Remember 到正确 owner？
ResourceOwnerRelease 有没有进入对应 phase？
commit warning 的 DebugPrint 能不能定位资源对象？
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/utils/elog.h` | `PG_TRY`、`PG_CATCH`、`PG_FINALLY`、`PG_RE_THROW` 的宏语义，以及文档里对错误恢复代码的约束。 |
| 2 | `src/backend/utils/error/elog.c` | `errfinish()` 如何在 `ERROR` 时跳转，`FlushErrorState()` 如何清空错误上下文，`FATAL` / `PANIC` 为什么不是普通 `PG_CATCH` 能处理的范围。 |
| 3 | `src/backend/tcop/postgres.c` | backend 主循环最外层 `sigsetjmp` 错误恢复：报错、abort 当前事务、Portal cleanup、清空 ErrorContext、回到下一条命令。 |
| 4 | `src/backend/tcop/pquery.c` | `PortalStart()`、`PortalRun()`、`PortalRunFetch()` 中局部 `PG_TRY` 如何保存并恢复 `ActivePortal`、`CurrentResourceOwner`、`PortalContext`。 |
| 5 | `src/include/utils/portal.h` | `PortalData` 的 `resowner`、`cleanup`、`status`、`portalSnapshot`、`holdSnapshot` 等状态边界。 |
| 6 | `src/backend/utils/mmgr/portalmem.c` | `MarkPortalFailed()`、`PortalDrop()`、`AtAbort_Portals()`、`AtCleanup_Portals()` 如何把失败 Portal 与事务资源释放分开。 |
| 7 | `src/backend/access/transam/xact.c` | `AbortCurrentTransaction()`、`AbortTransaction()`、`CleanupTransaction()`、`AtAbort_Memory()`、`AtAbort_ResourceOwner()`。 |
| 8 | `src/include/utils/resowner.h` | `ResourceOwnerDesc`、`ReleaseResource`、`DebugPrint`、`ResourceReleaseCallback` 的约束。 |
| 9 | `src/backend/utils/resowner/resowner.c` | `ResourceOwnerReleaseInternal()`、commit leak warning、release callback、`releasing` / `sorted` 标志。 |
| 10 | `src/include/storage/ipc.h`、`src/backend/storage/ipc/ipc.c` | `PG_ENSURE_ERROR_CLEANUP` 和 `before_shmem_exit`，说明 `ERROR` 与 `FATAL` 清理边界不同。 |

推荐阅读顺序：

```text
先读 elog.h 的 PG_TRY 注释
  -> 看 pquery.c 的局部恢复模板
  -> 看 postgres.c 顶层错误恢复
  -> 看 xact.c abort 如何重建 memory/resource owner
  -> 最后看 resowner.c 如何释放资源并打印诊断
```

不要从所有 `PG_CATCH` 横向搜索开始。PostgreSQL 里的 `PG_CATCH` 用途很多：有的用于 rethrow，有的用于保存错误信息，有的用于 shared-memory cleanup，有的用于协议恢复。本节只围绕 ResourceOwner 和 ERROR-safe cleanup 的主链路。

## 4. 关键数据结构与状态边界

### `PG_exception_stack` 与 `error_context_stack`

`elog.h` 的 `PG_TRY` 宏核心做两件事：

```c
sigjmp_buf *_save_exception_stack = PG_exception_stack;
ErrorContextCallback *_save_context_stack = error_context_stack;
sigjmp_buf _local_sigjmp_buf;

if (sigsetjmp(_local_sigjmp_buf, 0) == 0)
{
    PG_exception_stack = &_local_sigjmp_buf;
    ...
}
else
{
    PG_exception_stack = _save_exception_stack;
    error_context_stack = _save_context_stack;
    ...
}
```

这里的语义是：

```text
PG_exception_stack:
  当前 ERROR 要跳到哪里。

error_context_stack:
  错误上下文回调链，用于补充错误消息里的 context。
```

`PG_CATCH` 只恢复这两个 error subsystem 状态。它不会自动恢复：

```text
CurrentResourceOwner
CurrentMemoryContext
ActivePortal
PortalContext
CurrentUserId
lock wait state
LWLock
buffer content lock
snapshot refcount
catcache refcount
```

这些都必须由调用者、事务 abort 或 ResourceOwner 机制负责。

`elog.h` 的注释也给了本节最重要的规则：

```text
error recovery code 要么 PG_RE_THROW，要么做 transaction abort；
否则系统可能处于不一致状态。
```

也就是说，`PG_CATCH` 不是“捕获后继续”的普通异常处理。除非调用者明确知道自己在做 soft-error 或局部子事务语义，否则应当恢复局部状态后继续向外抛。

### `ErrorContext` 与 `FlushErrorState()`

`elog.c` 的 `errfinish()` 在 `ERROR` 级别不会返回调用者，而是：

```text
切到 ErrorContext
补齐错误数据
调用 error_context_stack
清理 InterruptHoldoffCount / QueryCancelHoldoffCount / CritSectionCount
PG_RE_THROW()
```

最外层错误恢复完成后，`postgres.c` 调用：

```c
MemoryContextSwitchTo(MessageContext);
FlushErrorState();
```

`FlushErrorState()` 会：

```text
errordata_stack_depth = -1
recursion_depth = 0
MemoryContextReset(ErrorContext)
```

这说明错误消息本身也有生命周期：

```text
ERROR 发生后:
  错误详情还活在 ErrorContext，handler 可以 CopyErrorData()。

FlushErrorState() 后:
  这次错误的 ErrorData 和上下文内存被清空。
```

所以在 `PG_CATCH` 里如果要做复杂处理，常见模式是：

```text
CopyErrorData()
FlushErrorState()
做可能再次 ERROR 的工作
ReThrowError() 或转换为别的语义
```

但 ResourceOwner 释放路径通常不走这种高层处理。它更偏向“低层、不可失败、直接释放”。

### `PortalData`

`portal.h` 中和本节有关的字段：

| 字段 | 语义 |
| --- | --- |
| `portalContext` | Portal 的附属内存上下文，executor state、query desc 等对象常放这里。 |
| `resowner` | Portal 级 ResourceOwner。执行 Portal 时 `CurrentResourceOwner` 会临时切到它。 |
| `cleanup` | Portal cleanup hook，通常用于关闭 executor 等 Portal 知道的状态。 |
| `status` | `PORTAL_NEW`、`DEFINED`、`READY`、`ACTIVE`、`DONE`、`FAILED`。ERROR 后常转成 `FAILED`。 |
| `queryDesc` | 如果 executor 活着，`PortalCleanup` 要能关掉它。 |
| `portalSnapshot` | Portal 执行期 active snapshot。 |
| `holdStore` / `holdContext` / `holdSnapshot` | holdable cursor 或 materialized result 的跨事务状态。 |

一个 Portal 出错后不能简单 `MemoryContextDelete(portalContext)`，因为：

```text
cleanup hook 可能还需要其中的 executor state。
portal->resowner 可能已经会在事务 abort 中释放。
holdStore 可能用了跨事务临时文件，需要显式结束。
FAILED portal 的 PortalDrop 要避免重复释放已由 abort 清掉的 owner。
```

这就是 `MarkPortalFailed()`、`AtAbort_Portals()`、`AtCleanup_Portals()` 分开的原因。

### `ResourceOwnerData.releasing` / `sorted`

`resowner.c` 中 owner 的关键错误路径状态：

| 字段 | 语义 |
| --- | --- |
| `releasing` | 已经开始 bulk release。之后不能再 `Remember()` / `Forget()` 单个资源。 |
| `sorted` | `arr` / `hash` 已经按 release priority 排序，不再保持 hash 查找语义。 |
| `arr` / `hash` | 普通 ResourceOwnerDesc 资源。 |
| `locks` | lock manager 的小型 fast path 缓存。 |
| `aio_handles` | AIO handle，因可能在 critical section 注册，单独链表管理。 |

一旦进入 `ResourceOwnerRelease()`：

```text
owner->releasing = true
ResourceOwnerSort(owner)
owner->sorted = true
```

之后 callback 不能再调用 `ResourceOwnerRemember()` 或 `ResourceOwnerForget()`。这是 release 本身 ERROR-safe 的一部分：bulk release 正在按排序位置推进，如果 callback 又改变资源集合，release 游标就不可靠。

### `ResourceOwnerDesc`

`resowner.h` 中每类资源用 `ResourceOwnerDesc` 描述：

```c
typedef struct ResourceOwnerDesc
{
    const char *name;
    ResourceReleasePhase release_phase;
    ResourceReleasePriority release_priority;
    void (*ReleaseResource) (Datum res);
    char *(*DebugPrint) (Datum res);
} ResourceOwnerDesc;
```

本节关注两点：

```text
ReleaseResource:
  abort / commit release 时真正释放资源。必须做低层 cleanup，不能做可能失败的高层操作。

DebugPrint:
  commit 路径发现资源未被正常 Forget 时，用来生成 WARNING 内容。
```

`DebugPrint` 是诊断接口，不是 correctness 机制。它能告诉你“commit 时 owner 里还剩这个资源”，但不能证明：

```text
资源什么时候泄漏的；
是哪条调用链第一次漏了 Forget；
abort 路径是否也会漏；
资源的全部外部影响是否已经清完。
```

## 5. 主流程源码 walkthrough

### 5.1 一个普通 ERROR 怎样逃到顶层

假设 executor 深处某个函数执行：

```c
ereport(ERROR, ...);
```

`elog.c` 中 `errfinish()` 的关键动作：

```text
1. 切到 ErrorContext。
2. 补齐 ErrorData，调用 error_context_stack。
3. 如果 elevel == ERROR：
     清掉 interrupt / query cancel holdoff 计数；
     清掉 CritSectionCount；
     PG_RE_THROW()。
4. PG_RE_THROW() 调用 pg_re_throw()。
5. pg_re_throw() 对 PG_exception_stack 做 siglongjmp。
```

这时 C 栈中间层不会逐个返回：

```text
ExecutorRun()
  -> access method
     -> bufmgr / catcache / lock / file
        -> ereport(ERROR)
           -> longjmp 到最近 handler
```

这就是为什么 PostgreSQL 不能依赖普通 `return` 路径逐层释放资源。任何需要 ERROR-safe 的资源，要么：

```text
获取后立刻被 Remember 到 ResourceOwner；
要么用 PG_TRY/PG_CATCH 局部保护；
要么用 PG_ENSURE_ERROR_CLEANUP 覆盖 FATAL 边界；
要么在进程退出 callback 中兜底。
```

### 5.2 `PortalStart()` 的局部恢复

`pquery.c` 的 `PortalStart()` 是局部 `PG_TRY` 的标准模板。

进入执行前保存：

```c
saveActivePortal = ActivePortal;
saveResourceOwner = CurrentResourceOwner;
savePortalContext = PortalContext;
```

进入 `PG_TRY` 后切换：

```c
ActivePortal = portal;
if (portal->resowner)
    CurrentResourceOwner = portal->resowner;
PortalContext = portal->portalContext;
oldContext = MemoryContextSwitchTo(PortalContext);
```

这样 executor / snapshot / buffer / lock 路径里调用 `CurrentResourceOwner` 时，会把资源记到这个 Portal owner 上。

如果中途 ERROR：

```c
PG_CATCH();
{
    MarkPortalFailed(portal);

    ActivePortal = saveActivePortal;
    CurrentResourceOwner = saveResourceOwner;
    PortalContext = savePortalContext;

    PG_RE_THROW();
}
PG_END_TRY();
```

注意这里没有：

```text
ResourceOwnerRelease(portal->resowner, ...)
AbortCurrentTransaction()
ExecutorEnd()
MemoryContextDelete(portal->portalContext)
FlushErrorState()
```

`PortalStart()` 只清理自己改过的“执行指针”。它把 Portal 标成 `FAILED`，然后把错误继续抛给外层。资源释放留给事务 abort 和 Portal cleanup。

这就是第一条边界：

```text
局部 PG_CATCH 负责恢复 control-plane 指针；
不负责释放 data-plane 资源集合。
```

### 5.3 `PortalRun()` 的 ugly 但重要的恢复

`PortalRun()` 比 `PortalStart()` 多保存了：

```c
saveTopTransactionResourceOwner = TopTransactionResourceOwner;
saveTopTransactionContext = TopTransactionContext;
saveMemoryContext = CurrentMemoryContext;
```

原因在源码注释里很直接：某些 utility command，例如 `VACUUM`、`CLUSTER`，可能内部 start / commit transaction。进入 `PortalRun()` 时保存的 `TopTransactionResourceOwner`，在命令内部 commit/restart 后可能已经被销毁并替换。

所以 catch 和正常返回时都有特殊判断：

```c
if (saveResourceOwner == saveTopTransactionResourceOwner)
    CurrentResourceOwner = TopTransactionResourceOwner;
else
    CurrentResourceOwner = saveResourceOwner;
```

这段代码看起来别扭，但它服务一个非常具体的 ERROR-safe 场景：

```text
PortalRun() 进入时 CurrentResourceOwner 指向当时的 top owner。
utility command 内部切换事务，旧 top owner 被销毁。
如果 ERROR 后仍恢复成旧指针，后续 cleanup 会使用悬空 owner。
```

因此 `PG_CATCH` 不是简单保存 / 恢复所有变量，而是要理解这些变量的生命周期。能恢复成原值的恢复原值；原值可能已被释放的，要恢复成当前有效替代物。

### 5.4 顶层 `postgres.c` 错误恢复

backend 主循环没有用外层 `PG_TRY` 包无限循环，而是直接设置最外层 `sigsetjmp`。源码注释解释原因：这是 exception stack 的底部；如果用普通 `PG_TRY`，在 catch 部分就没有更外层 handler 保护。保留最外层 setjmp，可以让错误恢复过程中再次 ERROR 时仍有机会处理。

当 longjmp 回来时，`postgres.c` 做：

```text
error_context_stack = NULL
HOLD_INTERRUPTS()
disable_all_timeouts(false)
QueryCancelPending = false
DoingCommandRead = false
pq_comm_reset()
EmitErrorReport()
debug_query_string = NULL
AbortCurrentTransaction()
WalSndErrorCleanup()
PortalErrorCleanup()
ReplicationSlotRelease()
ReplicationSlotCleanup(false)
jit_reset_after_error()
MemoryContextSwitchTo(MessageContext)
FlushErrorState()
ignore_till_sync = true   // extended query protocol 场景
xact_started = false
RESUME_INTERRUPTS()
```

这个流程里和 ResourceOwner 最相关的是：

```text
AbortCurrentTransaction()
PortalErrorCleanup()
FlushErrorState()
```

`EmitErrorReport()` 发生在 abort 前，是因为还需要把原始错误报告出去。`FlushErrorState()` 发生在 cleanup 后，是因为错误恢复期间可能还需要错误数据和 ErrorContext。

顶层错误恢复直接提醒我们：

```text
如果你在局部 PG_CATCH 里吞掉 ERROR，却没有做子事务 abort 或完整恢复，就绕开了这套顶层协议。
```

这通常是危险的。

### 5.5 `AbortCurrentTransaction()` 与状态机循环

`AbortCurrentTransaction()` 本身只是循环调用：

```c
while (!AbortCurrentTransactionInternal())
{
}
```

为什么要循环？因为可能存在子事务状态：

```text
TBLOCK_SUBABORT_END
TBLOCK_SUBABORT_PENDING
TBLOCK_SUBRESTART
TBLOCK_SUBABORT_RESTART
```

一个 ERROR 可能发生在嵌套 savepoint 内。系统不能递归地一层层 abort，因为错误恢复路径本身已经很脆弱，所以这里用迭代方式把当前事务状态机推进到稳定点。

普通 top-level implicit transaction ERROR 场景大致是：

```text
AbortCurrentTransaction()
  -> AbortTransaction()
  -> CleanupTransaction()
  -> blockState = TBLOCK_DEFAULT
```

显式事务块中发生 ERROR 时，则可能进入 aborted transaction block 状态，等待用户 `ROLLBACK`。

### 5.6 `AbortTransaction()` 先恢复低层安全边界

`xact.c` 的 `AbortTransaction()` 一开始不是直接释放 ResourceOwner，而是先建立 cleanup 所需的基本安全环境：

```text
HOLD_INTERRUPTS()
AtAbort_Memory()
AtAbort_ResourceOwner()
LWLockReleaseAll()
WaitLSNCleanup()
pgstat_report_wait_end()
pgaio_error_cleanup()
UnlockBuffers()
XLogResetInsertion()
ConditionVariableCancelSleep()
LockErrorCleanup()
reschedule_timeouts()
sigprocmask(SIG_SETMASK, &UnBlockSig, NULL)
```

这里有几个非常重要的顺序点。

第一，`AtAbort_Memory()`：

```text
切到 TransactionAbortContext；
如果还没创建它，就切到 TopMemoryContext。
```

abort 过程中不能依赖当前 `CurrentMemoryContext`，因为 ERROR 可能发生在任意局部上下文中，甚至那个上下文会在后面被 reset/delete。

第二，`AtAbort_ResourceOwner()`：

```c
CurrentResourceOwner = TopTransactionResourceOwner;
```

如果 ERROR 发生时 `CurrentResourceOwner` 正指向 Portal owner 或某个局部 owner，abort 必须先恢复到 top transaction owner。否则 cleanup 期间某些路径如果需要临时 owner 语义，会落到错误 owner 上。

第三，先释放 LWLock / lock wait / buffer content lock 等低层状态：

```text
LWLockReleaseAll()
UnlockBuffers()
LockErrorCleanup()
```

这些不是 ResourceOwner 三阶段释放的一部分。它们是“不能带着等待状态和轻量锁进入复杂 abort”的低层安全边界。

### 5.7 `AbortTransaction()` 再进入 ResourceOwner 三阶段

完成事务 abort 的 WAL / clog / ProcArray 等工作后，如果存在 `TopTransactionResourceOwner`，`AbortTransaction()` 调用：

```c
ResourceOwnerRelease(TopTransactionResourceOwner,
                     RESOURCE_RELEASE_BEFORE_LOCKS,
                     false, true);
AtEOXact_Aio(false);
AtEOXact_Buffers(false);
AtEOXact_RelationCache(false);
AtEOXact_TypeCache();
AtEOXact_Inval(false);
AtEOXact_MultiXact();
ResourceOwnerRelease(TopTransactionResourceOwner,
                     RESOURCE_RELEASE_LOCKS,
                     false, true);
ResourceOwnerRelease(TopTransactionResourceOwner,
                     RESOURCE_RELEASE_AFTER_LOCKS,
                     false, true);
smgrDoPendingDeletes(false);
```

这里 `isCommit=false` 是本节诊断边界的关键：

```text
abort 路径依赖 ResourceOwner 兜底释放资源；
所以 abort 时 owner 里还剩资源是正常现象；
因此 ResourceOwnerReleaseAll(... printLeakWarnings=false) 不打印泄漏 WARNING。
```

相反，commit 时 `isCommit=true`：

```text
正常路径应该已经 ReleaseBuffer / ReleaseCatCache / UnregisterSnapshot / FileClose；
如果 ResourceOwner 里仍有非锁资源，说明代码路径漏了 Forget；
ResourceOwnerRelease() 会打印 WARNING，然后仍然释放资源。
```

因此同一个 callback 在 commit 和 abort 中的诊断意义不同：

```text
commit warning:
  程序员错误或 cleanup 漏洞信号。

abort silent release:
  ERROR-safe cleanup 的正常兜底机制。
```

### 5.8 `CleanupTransaction()` 删除 owner 和事务内存

`AbortTransaction()` 结束后，状态仍是 `TRANS_ABORT`。真正删除事务级 owner 的是 `CleanupTransaction()`：

```text
AtCleanup_Portals()
AtEOXact_Snapshot(false, true)
CurrentResourceOwner = NULL
ResourceOwnerDelete(TopTransactionResourceOwner)
TopTransactionResourceOwner = NULL
AtCleanup_Memory()
```

这里 `AtCleanup_Portals()` 在 `ResourceOwnerRelease()` 之后运行。`AtAbort_Portals()` 早已把失败 Portal 的 `portal->resowner` 置空：

```text
Any resources belonging to the portal will be released in the upcoming transaction-wide cleanup;
portal->resowner = NULL;
```

所以到 `AtCleanup_Portals()` 删除 Portal 数据结构时，它不会再次释放那些已经由 transaction-wide owner tree 清掉的资源。

这条链路把 Portal、ResourceOwner、MemoryContext 三者分开：

```text
AtAbort_Portals:
  运行 cleanup hook，断开 portal->resowner，释放子上下文孩子。

ResourceOwnerRelease:
  释放 owner tree 里的外部资源。

AtCleanup_Portals:
  删除失败 transaction 中创建的 Portal 结构。

AtCleanup_Memory:
  reset/delete transaction memory。
```

## 6. 生命周期 / ownership / cleanup

### 正常 Portal 执行

一个普通 unnamed portal 执行成功：

```text
CreatePortal()
  portal->resowner = ResourceOwnerCreate(CurTransactionResourceOwner, "Portal")

PortalStart()
  CurrentResourceOwner = portal->resowner
  executor / snapshot / buffer / cache 路径把资源记到 portal owner
  正常退出后恢复 CurrentResourceOwner

PortalRun()
  再次切到 portal owner
  执行 query
  正常退出后恢复 CurrentResourceOwner

PortalDrop() 或事务结束
  普通资源应已按正常路径 Forget
  剩余锁可能转交 parent transaction owner
```

正常路径的理想状态：

```text
ResourceOwnerRelease(... isCommit=true ...)
  除锁语义外，不应发现还没释放的资源。
```

### Portal 执行 ERROR

同一个 Portal 如果中途 ERROR：

```text
PortalRun()
  PG_CATCH:
    MarkPortalFailed(portal)
    restore ActivePortal / CurrentResourceOwner / PortalContext / CurrentMemoryContext
    PG_RE_THROW()

postgres.c top recovery:
  EmitErrorReport()
  AbortCurrentTransaction()

AbortTransaction():
  AtAbort_Portals()
    运行 cleanup hook
    PortalReleaseCachedPlan()
    portal->resowner = NULL
  ResourceOwnerRelease(top owner, ..., false, true)
    释放 Portal owner 仍挂在 tree 里的资源

CleanupTransaction():
  AtCleanup_Portals()
    删除失败事务内创建的 portal
```

这就是“失败 Portal 的 resowner 被置空但资源仍能释放”的关键：

```text
portal->resowner 指针断开；
ResourceOwner object 仍作为 top transaction owner 的 child 留在 owner tree；
transaction-wide ResourceOwnerRelease() 递归 child-first 清理它。
```

### 子事务 ERROR

子事务 abort 时，Portal 逻辑更细：

```text
AtSubAbort_Portals(mySubid, parentSubid, myXactOwner, parentXactOwner)
```

如果 Portal 是当前子事务创建的，且还活着：

```text
PORTAL_READY / PORTAL_ACTIVE -> MarkPortalFailed()
运行 cleanup hook
释放 cached plan ref
portal->resowner = NULL
MemoryContextDeleteChildren(portal->portalContext)
```

如果 Portal 是上层创建、在当前子事务中用过，并且当前子事务把它搞成 `FAILED`：

```text
ResourceOwnerNewParent(portal->resowner, myXactOwner)
portal->resowner = NULL
```

这防止了一个 corner case：

```text
上层 Portal 内部仍引用当前失败子事务创建的对象；
如果它的资源不并入当前子事务 owner 释放，后续 cleanup 可能在对象已销毁后触碰悬挂引用。
```

子事务清理的模型仍然是：

```text
PG_CATCH 恢复局部执行指针；
xact.c subabort 释放当前子事务资源；
Portal 状态机负责把失败 Portal 从可执行状态移走。
```

### 进程退出和 FATAL 边界

`elog.h` 明确说：

```text
ereport(FATAL) will not be caught by this construct;
control will exit straight through proc_exit().
```

所以普通 `PG_TRY` 只覆盖 `ERROR`，不覆盖完整进程退出清理。

如果某段代码临时改变 shared-memory 状态，且 `FATAL` 也必须恢复，就要考虑 `PG_ENSURE_ERROR_CLEANUP`：

```c
PG_ENSURE_ERROR_CLEANUP(cleanup_function, arg);
{
    ... code that might throw ERROR or FATAL ...
}
PG_END_ENSURE_ERROR_CLEANUP(cleanup_function, arg);
```

这个宏来自 `storage/ipc.h`，内部会：

```text
before_shmem_exit(cleanup_function, arg)
PG_TRY()
...
成功路径 cancel_before_shmem_exit()
ERROR catch:
  cancel_before_shmem_exit()
  cleanup_function()
  PG_RE_THROW()
FATAL 路径:
  proc_exit -> shmem_exit -> before_shmem_exit callback
```

因此不要把 `PG_TRY` 想成进程级 finally。它只处理可恢复 ERROR。

## 7. 正确性机制层次

### 第一层：`ERROR` 之后不能继续使用坏的全局指针

`CurrentResourceOwner`、`ActivePortal`、`PortalContext` 是 backend-local 全局变量。它们本身不在 shared memory 里，但它们决定后续操作会把资源记到哪里、在哪个内存上下文分配。

如果局部 `PG_CATCH` 没恢复这些指针，后果可能是：

```text
下一段 cleanup 把资源 Remember 到失败 Portal owner；
MemoryContext 分配落到即将删除的 PortalContext；
ActivePortal 指向 FAILED portal，错误上下文或 cursor 操作误判当前执行对象；
top transaction owner 已替换，但 CurrentResourceOwner 仍指向旧 owner。
```

所以局部恢复的正确性不是“清理资源”，而是“保证外层 cleanup 在正确执行状态下运行”。

### 第二层：事务 abort 必须先离开危险等待 / 锁状态

`AbortTransaction()` 先释放：

```text
LWLock
buffer content lock
condition variable sleep
lock wait
timeout state
```

这类状态有共同点：

```text
它们不是普通 query-lifespan resources；
它们可能阻塞其他 backend 或让当前 backend 在 cleanup 中再次进入等待；
它们常常必须在复杂事务 cleanup 之前清掉。
```

ResourceOwner 主要管“获取后可挂账、结束时可批量释放”的资源。低层等待状态和临界区状态不能完全交给 ResourceOwner。

### 第三层：ResourceOwner 兜底 release 必须可重复进入相邻 phase

`resowner.c` 的注释说，如果在 release phase 之间发生 ERROR，`AbortTransaction()` 可能再次对同一个 owner 调用 `ResourceOwnerRelease()`。

因此 `ResourceOwnerReleaseInternal()` 对 `owner->releasing` 的处理不是简单 assert 第一次：

```text
第一次 before-locks:
  owner->releasing = true
  sort resources

后续 locks / after-locks:
  继续使用已排序资源列表

phase 之间 ERROR 后再次进入:
  owner 已经 releasing，也要能继续清理
```

这也是为什么 release 开始后禁止 `Remember()` / `Forget()`：

```text
release 需要一个稳定的资源集合和排序游标。
```

### 第四层：commit warning 是 bug 信号，abort release 是正常语义

`ResourceOwnerReleaseAll()` 里：

```c
if (printLeakWarnings)
{
    res_str = kind->DebugPrint ? kind->DebugPrint(value) : ...;
    elog(WARNING, "resource was not closed: %s", res_str);
}
kind->ReleaseResource(value);
```

`printLeakWarnings` 来自 `isCommit`。

这带来一个诊断不变量：

```text
同样是 owner 里剩资源：
  commit 时是“正常路径漏清理”的证据；
  abort 时是“ERROR-safe 兜底正在工作”的证据。
```

所以线上看到：

```text
WARNING: resource was not closed: ...
```

优先查正常路径：

```text
是否获取后忘了 Forget？
是否某个 early return 绕过 Release？
是否 cleanup hook 在成功路径没跑？
是否 portal/subtransaction commit 的 owner 转移语义误用？
```

不要把它简单解释成“abort 没清干净”。abort 通常不打印这类 warning。

## 8. 错误路径 / 异常路径 / fallback

### `PG_CATCH` 里再次 ERROR

`elog.h` 提醒：error recovery section 应尽量简单，因为恢复代码里再次 ERROR 只支持有限层数。`elog.c` 内部有 `errordata` stack 和 `recursion_depth`，用于避免错误报告递归无限增长。

这就是为什么很多 `PG_CATCH` 只做：

```text
标记状态
恢复指针
PG_RE_THROW()
```

复杂 cleanup 应该放到：

```text
transaction abort callbacks
ResourceOwner release callback
on_shmem_exit / before_shmem_exit
```

但这些 callback 自身也有约束，不能借此做任意高层逻辑。

### `ReleaseResource` callback 不能失败

`resowner.h` 对 `ResourceOwnerDesc` 注释：

```text
callbacks occur post-commit or post-abort,
can only do noncritical cleanup and must not fail.
```

`utils/resowner/README` 也强调：

```text
ReleaseResource callback during transaction abort
must perform only low-level cleanup with no user visible effects;
should not perform operations that could fail, like allocate memory.
```

原因很直接：

```text
ResourceOwnerRelease() 本来就是错误恢复的一部分；
如果 callback 再 ERROR，系统只能依赖外层顶级恢复继续兜底；
这会让资源集合处在部分释放状态，并扩大诊断难度。
```

所以一个新资源类型的 release callback 应尽量长这样：

```text
从 Datum 找到对象
清掉 refcount / owner 字段 / pin / fd / local list
调用不分配内存、不执行 SQL、不触发用户代码的低层释放函数
返回
```

不应该长这样：

```text
打开系统表
分配复杂对象
触发 invalidation 之外的用户可见行为
执行 SPI
依赖当前 MemoryContext 正常
依赖 CurrentResourceOwner 能 Remember 新资源
```

### `PortalDrop()` 宁愿泄漏一点内存也避免错误恢复死循环

`PortalDrop()` 有一段很有代表性的注释：

```text
Remove portal from hash table before subsequent steps.
Better to leak a little memory than to get into an infinite error-recovery loop.
```

它先从 hash table 删除 Portal，再继续释放 cached plan、snapshot、resowner、tuplestore、memory context。

这说明错误恢复代码的优先级不是“零泄漏”，而是：

```text
不重复处理同一个已失败对象；
不形成无限错误恢复循环；
让 backend 尽可能回到可继续处理命令的状态。
```

这也是 PostgreSQL 内核里常见的恢复哲学：

```text
在 ERROR 恢复路径上，可接受少量 backend-local 泄漏；
不可接受 shared state、lock、pin、wait 状态悬挂；
不可接受错误恢复无限递归。
```

### `AtCleanup_Portals()` 不再调用可能执行用户代码的 cleanup

`AtCleanup_Portals()` 删除失败事务中的 Portal。如果发现 `portal->cleanup` 还没运行：

```text
elog(WARNING, "skipping cleanup for portal \"%s\"", portal->name);
portal->cleanup = NULL;
```

为什么不补跑？因为 post-abort cleanup 已经到了很晚阶段：

```text
事务资源已释放；
对象可能已无效；
cleanup hook 可能触发用户定义代码或 executor shutdown 复杂路径；
再次 ERROR 会破坏 cleanup。
```

所以它选择：

```text
打 WARNING
跳过 hook
直接 PortalDrop()
```

这类 warning 的诊断意义不同于 `resource was not closed`：

```text
resource warning:
  ResourceOwner commit 正常路径漏 Forget。

skipping cleanup for portal:
  Portal cleanup hook 没能在 abort 早期运行，晚期 cleanup 不敢再碰。
```

### lock owner fallback

上一节讲过，ResourceOwner 对 lock 有小型缓存：

```text
MAX_RESOWNER_LOCKS = 15
```

如果 owner 持有锁太多，`ResourceOwnerReleaseInternal()` 在非 top-level release 时传给 lock manager 的 `locks` 可能为 `NULL`，让 lock manager 扫描自身结构。

这个 fallback 和 ERROR-safe cleanup 的关系是：

```text
即使 ResourceOwner 的 lock 快速列表不够，abort 也不能漏锁；
代价是 fallback 扫描更贵，但 correctness 优先。
```

### FATAL / PANIC 边界

`PG_TRY` 捕获 `ERROR`，不捕获 `FATAL`。`errfinish()` 对 `FATAL` 会：

```text
EmitErrorReport()
proc_exit(1)
```

`PANIC` 更严重，会导致 postmaster 观察到异常退出并处理整个集群级影响。

因此：

```text
需要在 FATAL 时清理 shared-memory 临时状态:
  不能只靠 PG_CATCH。

普通 query-lifespan resource:
  进程退出会让 OS 释放进程本地资源；
  shared memory 注册项、PGPROC、slot 等需要 proc_exit / shmem_exit callback。
```

本节主线的 `AbortCurrentTransaction()` 是可恢复 `ERROR` 的路径，不是所有 backend 退出路径的总称。

## 9. 成本、资源与跨模块传播

### `PG_TRY` 的运行成本

`PG_TRY` 基于 `sigsetjmp` / `siglongjmp`。正常路径也要建立 handler、保存少量指针。因此 hot path 上不会在极细粒度操作中随意包 `PG_TRY`。

典型使用位置是：

```text
PortalStart / PortalRun / PortalRunFetch:
  一次 query/portal 执行边界。

extension 或 utility command:
  有明确局部状态要恢复。

soft-error 或隔离错误:
  需要把 ERROR 转换成可控返回。
```

buffer pin、catcache ref、snapshot、file 这些高频资源不靠每次获取都包 `PG_TRY`，而是靠：

```text
ResourceOwnerEnlarge()
真实 acquire
ResourceOwnerRemember()
正常路径 ResourceOwnerForget()
abort 路径 bulk ResourceOwnerRelease()
```

### ResourceOwner release 的成本

错误发生后，ResourceOwner release 成本主要来自：

```text
owner tree 递归
资源数组 / hash 排序
每类资源 callback
lock release / reassign
fallback 扫描
```

这些成本放在事务结束或错误恢复路径，而不是每个资源获取路径。PostgreSQL 接受这种 trade-off：

```text
正常 hot path:
  Remember / Forget 尽量轻。

ERROR / transaction end:
  承担 bulk release 和诊断成本。
```

commit warning 的 `DebugPrint` 也只在异常诊断路径调用，因此可以比 hot path 稍重，但仍不能依赖不安全的高层状态。

### Portal cleanup 和事务 cleanup 的跨模块传播

一个 query ERROR 会穿过多个模块边界：

```text
executor:
  可能留下 queryDesc、snapshot、buffer pin、tupledesc、catcache ref。

pquery.c:
  标记 Portal failed，恢复执行指针。

portalmem.c:
  早期 abort 跑 cleanup hook，晚期 cleanup 删除 Portal。

xact.c:
  状态机 abort，释放事务资源。

resowner.c:
  按 owner tree 和 phase 调资源 callback。

postgres.c:
  协议层恢复，ErrorContext 清理，回到命令循环。
```

这解释了为什么 ResourceOwner 不能只被看作一个“资源数组”。它是多个模块之间约定错误恢复边界的中心对象。

### 诊断信息的成本边界

内核诊断越靠近 hot path，越需要克制：

```text
ResourceOwnerDesc.name:
  常驻静态字符串，成本低。

DebugPrint:
  只在 commit leak warning 时调用。

RESOWNER_STATS:
  编译期宏，默认关闭。

gdb breakpoint:
  调试时可用，线上不可依赖。

日志 WARNING:
  有 I/O 和噪声成本，只在违反正常路径假设时输出。
```

所以 PostgreSQL 不会为每次 `ResourceOwnerRemember()` 保存完整 C backtrace。那会极大拉高 hot path 成本。想定位“谁漏了 Forget”，通常需要 gdb、临时断点、定制日志或本地调试补丁。

## 10. 观测与诊断入口

### 10.1 日志能看到什么

commit 泄漏时常见日志：

```text
WARNING:  resource was not closed: ...
```

来自 `resowner.c`：

```text
ResourceOwnerReleaseAll()
  if printLeakWarnings:
    kind->DebugPrint(value) 或 generic 格式
    elog(WARNING, "resource was not closed: %s", res_str)
```

它能告诉你：

```text
某个 ResourceOwner 在 commit release 时仍持有某类资源；
资源类型来自 ResourceOwnerDesc.name；
如果 DebugPrint 实现较好，可以看到 buffer refcount、catcache tuple、file 等局部细节。
```

它不能告诉你：

```text
资源是哪一行代码获取的；
中间是否经历过 owner 转移；
是否是 Portal owner 还是 transaction owner 最初获取；
abort 路径是否存在同样问题；
业务 SQL 的哪部分触发了泄漏。
```

另一个 Portal 相关 warning：

```text
WARNING: skipping cleanup for portal "..."
```

它说明晚期 cleanup 不敢再运行 cleanup hook。它更像“错误恢复路径已经错过最佳 cleanup 时机”的信号。

### 10.2 `DebugPrint` 能看到什么

内置资源例子：

| 资源 | 文件 | DebugPrint 价值 |
| --- | --- | --- |
| buffer pin / IO | `bufmgr.c` | 可打印 buffer refcount 相关信息，帮助判断 pin 泄漏。 |
| catcache tuple/list | `catcache.c` | 可打印 cache object 线索，帮助定位 catalog cache ref 泄漏。 |
| tupledesc | `tupdesc.c` | 可打印 tuple descriptor 相关描述。 |
| file | `fd.c` | 可打印 VFD file 线索。 |
| snapshot | `snapmgr.c` | 当前使用默认格式，信息较粗。 |

`DebugPrint` 的边界：

```text
可以读资源对象已有字段；
可以格式化一小段诊断字符串；
不应该获取新资源、执行查询、依赖复杂上下文；
输出是“释放时看到的对象状态”，不是“泄漏发生时的调用栈”。
```

因此看到 buffer warning 后，更有效的下一步通常是：

```text
给 ResourceOwnerRememberBuffer / ResourceOwnerForgetBuffer / ReleaseBuffer 打断点；
观察同一个 buffer id 的 Remember / Forget 是否配对；
结合 CurrentResourceOwner->name 判断挂在哪个 owner；
再回到具体调用链修 early return 或 ERROR 窗口。
```

### 10.3 gdb 断点建议

调试 Portal ERROR 恢复：

```gdb
break PortalRun
break MarkPortalFailed
break AtAbort_Portals
break AbortTransaction
break ResourceOwnerRelease
break AtCleanup_Portals
```

观察点：

```gdb
print ActivePortal
print CurrentResourceOwner
print TopTransactionResourceOwner
print portal->status
print portal->resowner
```

调试 ResourceOwner commit warning：

```gdb
break ResourceOwnerReleaseAll
commands
  silent
  printf "phase=%d printLeakWarnings=%d owner=%p\n", phase, printLeakWarnings, owner
  continue
end
```

如果已经知道资源类型，可以断在对应 release callback：

```gdb
break ResOwnerReleaseBuffer
break ResOwnerReleaseCatCache
break ResOwnerReleaseSnapshot
break ResOwnerReleaseFile
```

调试局部 `PG_CATCH` 是否恢复 owner：

```gdb
break PortalStart
break PortalRun
watch CurrentResourceOwner
```

注意 watch 全局变量可能很吵；更实用的是在 `PG_CATCH` 附近打断点，手工看保存值和恢复值。

### 10.4 SQL 层能看到什么

SQL 层不能直接看到 `CurrentResourceOwner` 或 owner tree。能观察到的是结果现象：

```text
ERROR 后连接是否仍可用；
显式事务块是否进入 aborted state；
cursor 是否变成不可继续执行；
锁是否释放或仍因事务块未 ROLLBACK 而保留；
pg_stat_activity 是否进入 idle in transaction (aborted)；
日志是否出现 resource warning。
```

一个简单观察：

```sql
BEGIN;
SELECT 1 / 0;
SELECT 1;
ROLLBACK;
SELECT 1;
```

预期：

```text
SELECT 1 / 0 报错后，显式事务块进入 aborted state；
第二个 SELECT 1 会提示 current transaction is aborted；
ROLLBACK 后连接恢复；
最后 SELECT 1 成功。
```

这能验证事务状态机行为，但不能直接验证某个 buffer pin 是否释放。资源级诊断要进日志、gdb 或源码 instrumentation。

### 10.5 源码搜索入口

定位某类资源的 ResourceOwnerDesc：

```bash
rg -n "ResourceOwnerDesc .*desc|release_phase|ReleaseResource|DebugPrint" \
  /home/highgo/postgres/src/backend /home/highgo/postgres/src/include
```

定位局部 ERROR 恢复：

```bash
rg -n "PG_TRY|PG_CATCH|PG_RE_THROW|CurrentResourceOwner|ActivePortal|PortalContext" \
  /home/highgo/postgres/src/backend/tcop \
  /home/highgo/postgres/src/backend/utils/mmgr
```

定位顶层错误恢复：

```bash
rg -n "AbortCurrentTransaction|PortalErrorCleanup|FlushErrorState|EmitErrorReport" \
  /home/highgo/postgres/src/backend/tcop/postgres.c \
  /home/highgo/postgres/src/backend/access/transam/xact.c
```

定位 commit warning：

```bash
rg -n "resource was not closed|DebugPrint|printLeakWarnings" \
  /home/highgo/postgres/src/backend/utils/resowner/resowner.c \
  /home/highgo/postgres/src/backend
```

## 11. 常见误区

### 误区一：`PG_CATCH` 会自动释放资源

不会。`PG_CATCH` 只恢复 `PG_exception_stack` 和 `error_context_stack`，用户代码必须自己恢复它改过的其他状态。

资源释放来自：

```text
正常路径的 Release / Forget；
abort 路径的 ResourceOwnerRelease；
进程退出路径的 proc_exit / shmem_exit callback；
特殊 FATAL 保护的 PG_ENSURE_ERROR_CLEANUP。
```

### 误区二：ERROR 后只要 MemoryContext reset 就够了

不够。MemoryContext 只能回收内存。它不能：

```text
unpin buffer
release heavyweight lock
decrement catcache refcount
unregister snapshot
close VFD file
detach DSM
end buffer IO
```

这些需要 ResourceOwner 或专门 cleanup。

### 误区三：abort 路径没打印 warning，说明没有资源残留

不对。abort 路径 `isCommit=false`，ResourceOwner 正常会静默释放残留资源。没有 warning 只能说明这不是 commit leak warning 场景。

如果想知道 abort 释放了什么，需要：

```text
gdb 断 ReleaseResource callback；
临时 instrumentation；
资源模块自身 debug 日志；
或观察外部现象，比如锁等待是否解除。
```

### 误区四：commit warning 可以直接定位 bug 行

不能。commit warning 是 release 时的结果，不是获取时的调用栈。

它通常只能给：

```text
资源类型
资源对象当前状态
释放时机
```

真正定位要沿 Remember / Forget 配对查。

### 误区五：release callback 可以随手调用高层代码

不能。ResourceOwner callback 运行在 post-commit / post-abort cleanup，尤其 abort 时系统状态很脆弱。它必须是低层、不可失败、无用户可见副作用的 cleanup。

### 误区六：`FATAL` 也会走普通 `PG_CATCH`

不会。需要覆盖 `FATAL` 的 shared-memory cleanup，要使用 `before_shmem_exit` / `on_shmem_exit` 或 `PG_ENSURE_ERROR_CLEANUP` 这一类机制。

### 误区七：`CurrentResourceOwner = NULL` 一定是 bug

不一定。`resowner` README 明确说，不在事务内或处于 failed transaction 时，`CurrentResourceOwner` 可以是 `NULL`。这时不应获取 query-lifespan 资源。

真正危险的是：

```text
需要获取资源时 CurrentResourceOwner 为 NULL；
或者 ERROR 后 CurrentResourceOwner 指向已经被销毁的 owner。
```

## 12. 课堂实验

### 实验一：观察显式事务 ERROR 后的状态

目标：从 SQL 层看到顶层错误恢复和事务状态机边界。

步骤：

```sql
BEGIN;
SELECT 1 / 0;
SELECT 42;
ROLLBACK;
SELECT 42;
```

观察：

```text
除零 ERROR 后，连接没有断。
同一事务块内继续执行 SELECT 会失败。
ROLLBACK 后恢复。
```

回到源码解释：

```text
postgres.c:
  AbortCurrentTransaction()

xact.c:
  显式事务块进入 aborted transaction block 状态。

ResourceOwner:
  当前失败语句的资源被 abort cleanup 释放。
```

### 实验二：断住 Portal ERROR 恢复

目标：观察 `PG_CATCH` 只恢复指针并 rethrow。

gdb 断点：

```gdb
break PortalRun
break MarkPortalFailed
break AbortCurrentTransaction
break AbortTransaction
break ResourceOwnerRelease
```

SQL：

```sql
SELECT 1 / 0;
```

观察顺序：

```text
PortalRun() 进入；
ERROR 后 MarkPortalFailed()；
PortalRun() catch 恢复 ActivePortal / CurrentResourceOwner / PortalContext；
错误继续到 postgres.c；
AbortCurrentTransaction()；
ResourceOwnerRelease(... isCommit=false ...)。
```

要点：

```text
如果只停在 MarkPortalFailed()，不要误以为资源已经释放；
真正 bulk release 在事务 abort 里。
```

### 实验三：跟踪 `CurrentResourceOwner` 切换

目标：理解执行指针恢复和资源 owner 释放不是一回事。

gdb：

```gdb
break PortalStart
break PortalRun
break AtAbort_ResourceOwner
break ResourceOwnerReleaseInternal
```

观察：

```gdb
print CurrentResourceOwner
print portal->resowner
print TopTransactionResourceOwner
```

预期：

```text
Portal 执行期间 CurrentResourceOwner 可能切到 portal->resowner；
catch 后恢复为保存值；
AbortTransaction 开始时 AtAbort_ResourceOwner 切回 TopTransactionResourceOwner；
ResourceOwnerReleaseInternal 释放具体 owner 时会临时 CurrentResourceOwner = owner。
```

这解释了为什么“当前 owner 指针”既是资源记账入口，也是 release callback 的上下文。

### 实验四：阅读一个 ResourceOwnerDesc

目标：判断某类资源的诊断能力。

选择 buffer：

```bash
rg -n "buffer_resowner_desc|ResOwnerReleaseBuffer|ResOwnerPrintBuffer" \
  /home/highgo/postgres/src/backend/storage/buffer/bufmgr.c
```

问题：

```text
release phase 是 before-locks 还是 after-locks？
ReleaseResource 做了哪些低层动作？
DebugPrint 能打印哪些信息？
如果 commit warning 出现，它能不能告诉你谁忘了 ReleaseBuffer？
```

选择 snapshot：

```bash
rg -n "snapshot_resowner_desc|ResOwnerReleaseSnapshot|RegisterSnapshot|UnregisterSnapshot" \
  /home/highgo/postgres/src/backend/utils/time/snapmgr.c
```

比较：

```text
snapshot 的 DebugPrint 为 NULL，默认消息更粗；
因此 snapshot leak warning 的定位通常更依赖调用链断点。
```

### 实验五：模拟设计一个新 ResourceOwner 资源

目标：把本节规律迁移到新资源类型。

假设你有一个扩展资源 `MyHandle`：

```text
获取后必须关闭；
关闭只改 backend-local C struct；
不影响其他 backend；
DebugPrint 可以输出 handle id；
关闭不分配内存。
```

设计：

```text
release_phase:
  RESOURCE_RELEASE_AFTER_LOCKS

release_priority:
  RELEASE_PRIO_LAST 或相对内置资源选择一个稳定位置

ReleaseResource:
  MyHandleCloseNoError(handle)

DebugPrint:
  psprintf("my handle %u", handle->id)

获取路径:
  ResourceOwnerEnlarge(CurrentResourceOwner)
  handle = MyHandleOpen()
  ResourceOwnerRemember(CurrentResourceOwner, PointerGetDatum(handle), &my_desc)

正常释放:
  ResourceOwnerForget(CurrentResourceOwner, PointerGetDatum(handle), &my_desc)
  MyHandleCloseNoError(handle)
```

讨论：

```text
如果 MyHandleClose 可能 ERROR，这个设计不合格；
如果 MyHandle 对其他 backend 可见，after-locks 可能不合格；
如果 DebugPrint 要查系统表，这个诊断设计不合格。
```

## 13. 讨论题

1. 为什么 `PortalStart()` 的 `PG_CATCH` 不直接调用 `ResourceOwnerRelease(portal->resowner, ...)`？如果这么做，会和 transaction-wide abort 产生哪些重复释放或 owner tree 顺序问题？

2. `PortalRun()` 为什么不能总是把 `CurrentResourceOwner` 恢复成 `saveResourceOwner`？utility command 内部 commit/restart transaction 时，这个指针可能发生什么？

3. commit 路径发现资源未关闭时，为什么 PostgreSQL 打 `WARNING` 后仍继续调用 `ReleaseResource`，而不是 `ERROR`？如果这里 `ERROR`，错误恢复会进入什么尴尬状态？

4. 为什么 `ReleaseResource` callback 不应该分配内存？如果一定要生成诊断字符串，应该放在 `DebugPrint` 还是 release 主逻辑？

5. `PG_ENSURE_ERROR_CLEANUP` 解决了 `PG_TRY` 的哪个边界？为什么它适合 shared-memory 临时状态，而不是每个普通 buffer pin 都用它？

6. 如果日志里只有 `resource was not closed: Snapshot ...`，你能确定是哪条 SQL 泄漏的吗？还需要哪些 gdb 断点或临时 instrumentation？

7. `AtCleanup_Portals()` 发现 cleanup hook 还没执行时为什么选择跳过并 warning？这里为什么“少量泄漏”比“尝试补救”更可接受？

8. 在一个扩展里，如果你想在 `PG_CATCH` 中吞掉 ERROR 并继续执行，必须证明哪些状态已经恢复？为什么“只调用 FlushErrorState”不够？

## 14. 本节小结

本节的可迁移模型：

```text
ERROR-safe cleanup 要分成 control-plane recovery 和 resource-plane cleanup。
```

在 PostgreSQL 里：

```text
PG_TRY / PG_CATCH:
  捕获 ERROR 控制流，恢复 error stack 和调用者自己改过的全局执行状态。

postgres.c:
  顶层报告错误，abort 事务，清理 Portal / ErrorContext / 协议状态。

xact.c:
  切到 abort-safe memory/resource owner，释放低层等待状态，推进事务 abort 状态机。

resowner.c:
  对 owner tree 中仍挂账的外部资源做三阶段 bulk release。

ResourceOwnerDesc:
  定义资源如何低层释放，以及 commit leak warning 怎样打印。
```

最重要的诊断边界：

```text
commit warning 说明正常路径漏清理；
abort silent release 说明兜底机制在工作；
DebugPrint 展示 release 时的资源状态，不展示获取时调用栈；
gdb 可以看到 owner / portal / callback 时序；
SQL 只能看到事务状态、cursor 行为、锁等待等外部现象。
```

最终规律：

```text
一个可靠的内核错误恢复系统，不要求每层业务代码都能完美 unwind；
它要求资源获取时立即记账，局部 ERROR handler 恢复执行指针，事务 abort 统一推进状态机，资源 owner 以低层 callback 兜底释放，并把诊断信息控制在不会破坏恢复路径的边界内。
```

这也解释了为什么 ResourceOwner 看起来只是一个 owner tree，实际上却连接了 PostgreSQL 的错误模型、事务状态机、Portal 执行、内存上下文、锁释放、资源泄漏诊断和扩展 API。
