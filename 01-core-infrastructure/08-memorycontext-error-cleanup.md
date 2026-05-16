# PostgreSQL ERROR 路径与 MemoryContext cleanup

## 课程定位

前置知识：已经理解 MemoryContext tree 如何表达 backend-local 内存 ownership；已经理解短生命周期 context 为什么依赖 reset；也已经看过事务、Portal 与 ResourceOwner 如何承载比单条语句更长、但不能无限期挂在 `TopMemoryContext` 下的状态。

本节唯一主问题：

```text
ERROR longjmp 后，哪些内存会被 context reset/delete 自动回收，哪些资源不能只靠 MemoryContext 兜底？
```

核心矛盾：PostgreSQL 允许深层代码用 `ereport(ERROR)` 直接跳出当前调用栈，避免每一层手写错误返回值；但 C 语言的 longjmp 不会执行普通栈展开、析构函数或函数尾部 cleanup。因此系统必须把 cleanup 拆成多层：错误报告放在 `ErrorContext`，事务内存靠 transaction context reset，Portal 内存靠 Portal cleanup，锁、pin、snapshot、文件、等待状态等非内存资源靠 ResourceOwner、模块级 abort hook 和进程退出回调。

学完后应能判断：一段 PostgreSQL 内核代码在 `ERROR` 后能不能只依赖 MemoryContext；什么时候必须注册 ResourceOwner、MemoryContext reset callback、`PG_TRY`/`PG_CATCH` cleanup、`PG_ENSURE_ERROR_CLEANUP` 或 `before_shmem_exit`/`on_shmem_exit`；也能读懂顶层 error recovery 为什么先报告错误、再 abort transaction、再清 Portal、最后 reset `ErrorContext`。

本课基于本地 `~/postgres` 源码，分支 `master`，提交 `8eba2edb8010`。

## 1. 本节在总主线中的位置

前三节逐步建立了 MemoryContext 生命周期模型：

```text
MemoryContext tree:
  palloc chunk 属于某个 context，context tree 表达 backend-local ownership

短生命周期 reset:
  per-tuple / per-expression 临时对象不能跨 reset 边界

事务与 Portal:
  有些状态要活过单条语句，但必须在事务结束、PortalDrop 或 abort 时释放
```

这些模型在正常返回路径上已经足够解释大多数内存释放。但 PostgreSQL 的错误路径不是普通返回：

```text
deep function
  -> ereport(ERROR)
     -> errfinish()
        -> PG_RE_THROW()
           -> siglongjmp 到最近的 PG_TRY 或 PostgresMain 顶层 handler
```

被跳过的 C 栈帧不会自然运行尾部 cleanup：

```text
oldcxt = MemoryContextSwitchTo(ctx);
resource = acquire_resource();

... ereport(ERROR) ...

release_resource(resource);          // 不会执行
MemoryContextSwitchTo(oldcxt);       // 不会执行
```

本节要回答的不是“MemoryContext 能不能自动释放内存”，因为答案已经是“能，在正确生命周期边界上能”。本节真正要拆开的是：

```text
ERROR 恢复需要恢复哪些东西？

内存:
  palloc chunk、context children、错误消息字符串

非内存资源:
  lock、LWLock、buffer pin、snapshot 注册、AIO、临时文件、wait state、Portal executor 状态

控制流状态:
  CurrentMemoryContext、CurrentResourceOwner、error_context_stack、ReadyForQuery 状态、extended protocol sync 状态
```

MemoryContext 是其中一层，但不是兜底全部资源的万能层。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ereport(ERROR) 把错误信息构造在 ErrorContext 后 longjmp；
顶层 handler 报告错误、abort transaction、清 Portal 和特殊资源；
transaction / portal / message context reset/delete 回收 palloc 内存；
ResourceOwner 和各模块 abort hook 释放不能靠 palloc 回收的资源。
```

这里的 tension 是：

```text
深层代码需要用 ERROR 快速逃离复杂调用栈
  vs
系统必须恢复到能继续处理下一条客户端消息的一致 backend 状态
```

如果只有 MemoryContext，系统最多能保证：

```text
挂在将被 reset/delete 的 context 下的 palloc 内存会消失
```

但它不能天然保证：

```text
已经持有的 heavyweight lock 被释放或重分配
buffer pin 被 unpin
LWLock 不再保持
snapshot 从注册栈中移除
Portal 不再指向失败的 executor state
等待队列状态被清理
临时文件被关闭或删除
共享内存中的 transient 标志被撤销
协议状态回到 ReadyForQuery 或等待 Sync
```

因此 `ERROR` cleanup 是一个分层系统：

```text
ErrorContext:
  让错误报告本身在 OOM 或递归错误中仍有内存可用

MemoryContext tree:
  批量回收 backend-local palloc 内存

ResourceOwner:
  释放锁、pin、snapshot、AIO 等外部资源和引用

事务 / Portal cleanup hook:
  按模块语义关闭 executor、释放 cached plan refcount、清理等待状态

进程退出 callback:
  兜底 FATAL/PANIC 或 shared-memory transient state
```

这就是本节要沉淀的系统规律：

```text
MemoryContext 管 ownership；
ERROR recovery 管一致性恢复；
二者相交，但不能互相替代。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/include/utils/elog.h` | `PG_TRY` / `PG_CATCH` / `PG_FINALLY` / `PG_RE_THROW` 宏；说明 handler 必须 rethrow 或 abort，否则系统可能不一致。 |
| 2 | `src/backend/utils/error/elog.c` | `errstart()`、`errfinish()`、`EmitErrorReport()`、`FlushErrorState()`；错误信息如何放进 `ErrorContext`，以及 `ERROR` 如何 longjmp。 |
| 3 | `src/backend/tcop/postgres.c` | `PostgresMain` 顶层 `sigsetjmp` handler；从错误报告到 `AbortCurrentTransaction()`、`PortalErrorCleanup()`、`FlushErrorState()` 的主恢复路径。 |
| 4 | `src/backend/access/transam/xact.c` | `AbortTransaction()`、`CleanupTransaction()`、`AbortCurrentTransaction()`、`AtAbort_Memory()`、`AtCleanup_Memory()`；事务 abort 中内存和资源 cleanup 的顺序。 |
| 5 | `src/backend/utils/resowner/resowner.c` / `src/include/utils/resowner.h` | `ResourceOwnerRelease()` 三阶段释放；为什么 lock、pin、snapshot 等不能只靠 MemoryContext。 |
| 6 | `src/backend/utils/mmgr/mcxt.c` | `MemoryContextReset()`、`MemoryContextDelete()`、reset callback 调用；context cleanup 能做什么，不能做什么。 |
| 7 | `src/backend/utils/mmgr/portalmem.c` | `AtAbort_Portals()`、`AtCleanup_Portals()`、`PortalErrorCleanup()`；ERROR 后 Portal 何时 mark failed、何时 delete children、何时 drop。 |
| 8 | `src/backend/commands/portalcmds.c` | `PortalCleanup()`；executor 相关 Portal 状态如何在失败时收口。 |
| 9 | `src/include/storage/ipc.h` / `src/backend/storage/ipc/ipc.c` | `PG_ENSURE_ERROR_CLEANUP`、`before_shmem_exit`、`on_shmem_exit`；涉及 shared memory transient state 或 FATAL 兜底时为什么不能只靠 `PG_CATCH`。 |
| 10 | `src/backend/utils/adt/mcxtfuncs.c` / `src/backend/catalog/system_views.sql` | `pg_backend_memory_contexts`、`pg_log_backend_memory_contexts()`；内存侧观测入口。 |

阅读顺序的关键是先抓住控制流：

```text
ERROR 构造
  -> longjmp
  -> 顶层 error recovery
  -> transaction abort
  -> resource owner release
  -> memory context reset
  -> 回到 main loop
```

不要从 `MemoryContextReset()` 开始读。它只解释“palloc chunk 怎么被释放”，解释不了为什么 abort 中要先 `LWLockReleaseAll()`、为什么锁释放分 phase、为什么 Portal cleanup 被拆成 abort 和 cleanup 两段。

## 4. 关键数据结构与状态

### `ErrorContext`

`ErrorContext` 在 `mcxt.c` 的 `MemoryContextInit()` 中创建，是 `TopMemoryContext` 的子 context。它有两个特殊点：

```text
创建很早
保留少量可用空间
```

它的语义不是“出错对象的归宿”，而是：

```text
错误报告机制自己的工作内存
```

`elog.c` 中 `errstart()` 为当前 error stack entry 设置：

```text
edata->assoc_context = ErrorContext
```

`errfinish()` 会切到 `ErrorContext`：

```text
oldcontext = MemoryContextSwitchTo(ErrorContext);
```

然后执行错误上下文回调、补齐 message/detail/hint/context 等字段。对于 `ERROR` 级别，`errfinish()` 不返回原调用者，而是调用 `PG_RE_THROW()`。

顶层 handler 在错误已经报告、事务已经 abort 后会：

```text
MemoryContextSwitchTo(MessageContext);
FlushErrorState();
```

`FlushErrorState()` 重置 error stack 深度，并 `MemoryContextReset(ErrorContext)`。这说明：

```text
错误报告字符串只需要活到报告完成；
不能把 ErrorContext 当成普通长期状态容器；
CopyErrorData() 后如果要继续使用错误信息，需要按 API 语义复制并及时 FlushErrorState()。
```

### `PG_exception_stack` 与 `error_context_stack`

`PG_exception_stack` 是当前可跳转的异常处理栈。`PG_TRY` 会保存旧栈，设置新的局部 `sigjmp_buf`；`PG_CATCH` 恢复旧栈；`PG_RE_THROW()` 最终调用 `pg_re_throw()`，如果有外层 handler 就 `siglongjmp`。

`error_context_stack` 是错误报告中的上下文回调链，例如 parser、executor、COPY、PL 语言或扩展可以把“正在处理哪一行、哪个函数、哪个参数”追加到错误报告。

顶层 `PostgresMain` handler 不是普通 `PG_TRY`，而是直接保留一个最外层 `sigsetjmp`。源码注释给出的理由很关键：

```text
这是异常栈底部；
如果用普通 PG_TRY 包住无限循环，CATCH 部分运行时外层可能没有 handler；
保留最外层 setjmp 可以让 error recovery 自己出错时仍有机会被捕获。
```

顶层 handler 进入后会手动：

```text
error_context_stack = NULL;
```

因为它没有经过 `PG_CATCH` 宏的自动恢复路径。

### `TransactionAbortContext`

事务 abort 期间不能继续依赖当前失败语句或失败事务的普通 context。`xact.c` 的 `AtAbort_Memory()` 会切到 `TransactionAbortContext`：

```text
if (TransactionAbortContext != NULL)
    MemoryContextSwitchTo(TransactionAbortContext);
else
    MemoryContextSwitchTo(TopMemoryContext);
```

这个 context 的作用是：

```text
让 abort cleanup 本身有一块相对可靠的工作内存
```

它不是业务数据容器。`AtCleanup_Memory()` 后会 reset 它：

```text
MemoryContextReset(TransactionAbortContext);
```

如果把 abort cleanup 需要的少量工作分配在将被删除的 transaction context 里，就可能在 cleanup 尚未完成时把自己脚下的内存释放掉。

### `TopTransactionContext` / `CurTransactionContext`

事务正常结束时：

```text
AtCommit_Memory()
  -> MemoryContextReset(TopTransactionContext)
```

事务 abort cleanup 时：

```text
AtCleanup_Memory()
  -> MemoryContextReset(TransactionAbortContext)
  -> MemoryContextReset(TopTransactionContext)
  -> CurTransactionContext = NULL
```

这能自动回收：

```text
事务内 palloc 的 List、Node、snapshot copy 的内存部分
子事务 context
executor/query/Portal 中已经挂到事务子树下的 backend-local chunk
```

但它不能自动释放：

```text
资源 owner 记录的 pin / lock / snapshot registration
共享内存状态
打开的内核 fd 或第三方库 malloc 内存，除非有 callback 或模块 cleanup
```

### `ResourceOwner`

ResourceOwner 是 MemoryContext 的并列机制，不是 MemoryContext 的子功能。它记录“内存之外的可释放资源和引用”。

`ResourceOwnerRelease()` 注释强调它通常要分三次调用：

```text
RESOURCE_RELEASE_BEFORE_LOCKS
RESOURCE_RELEASE_LOCKS
RESOURCE_RELEASE_AFTER_LOCKS
```

原因不是代码风格，而是正确性顺序：

```text
有些资源必须在释放锁前处理；
有些资源就是锁；
有些资源必须在锁释放后处理；
xact.c 还要在这些 phase 之间插入模块级 cleanup。
```

在 abort 路径中，`ResourceOwnerRelease(..., isCommit=false, ...)` 会安静释放残留资源；在 commit 路径中，残留资源往往意味着 executor 或模块忘记正常清理，所以可以 warning。

### `MemoryContext` reset callback

`MemoryContextRegisterResetCallback()` 可以把一个 callback 挂到 context 上，在 reset/delete 前调用。`mcxt.c` 中 callback 调用有一个细节：

```text
调用前先从 reset_cbs 链上弹出；
如果 callback 自己 ERROR，后续 reset/delete 不会重复调用同一个 callback。
```

它适合处理：

```text
与某个 palloc 对象绑定的少量外部资源
长生命周期 cache 对象 refcount
第三方库 malloc 出来的附属内存
临时文件对象的附属 cleanup
```

但它不适合替代 ResourceOwner。原因是：

```text
ResourceOwner 能按事务、Portal、subtransaction 和 release phase 表达顺序；
MemoryContext callback 只知道“这个 context 要 reset/delete 了”；
它不知道锁释放顺序、commit/abort 语义、subtransaction reassign 语义。
```

## 5. 主流程源码 walkthrough

本节主流程从一个普通 SQL 执行中发生 `ERROR` 开始：

```sql
BEGIN;
SELECT 1 / 0;
```

除零错误本身不重要。重要的是控制流从 executor 深处直接跳到顶层恢复逻辑。

### 第 1 步：深层代码调用 `ereport(ERROR)`

典型入口是：

```text
ereport(ERROR, (...))
  -> errstart(ERROR, ...)
  -> errmsg()/errdetail()/errhint() 等填充 ErrorData
  -> errfinish()
```

`errstart()` 决定这个错误是否需要输出。如果错误级别是 `ERROR`，但当前没有可用的 `PG_exception_stack`，或者已经在 `proc_exit` 中，它会升级成 `FATAL`。这说明：

```text
ERROR 可恢复的前提是当前 backend 有可跳转的 error handler；
没有 handler 的 ERROR 不能被当成普通异常继续跑。
```

`errfinish()` 做三件关键事：

```text
切到 ErrorContext
调用 error_context_stack 上的上下文回调
如果 elevel == ERROR，重置部分中断/critical section 计数后 PG_RE_THROW()
```

源码里有一句容易被忽略的注释：

```text
ERROR longjmp 前会留下 CurrentMemoryContext 指向 ErrorContext；
handler 应该很快切到别的 context。
```

这就是顶层 handler 后面要显式：

```text
MemoryContextSwitchTo(MessageContext);
```

的原因之一。

### 第 2 步：控制流跳到 `PostgresMain` 顶层 handler

`postgres.c` 中主循环开始前设置最外层 `sigsetjmp(local_sigjmp_buf, 1)`。发生 `ERROR` 后，控制流进入：

```text
if (sigsetjmp(local_sigjmp_buf, 1) != 0)
{
    ...
}
```

这个块不是业务 cleanup 的大杂烩。源码注释明确提醒：

```text
如果想在这里加代码，很可能应该加到 AbortTransaction() 中；
这里应该只放 outer-level error recovery 才需要的东西。
```

顶层 handler 的核心顺序是：

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
WalSndErrorCleanup()        // walsender 场景
PortalErrorCleanup()
ReplicationSlotRelease()    // 顶层错误不能继续持有的 slot
ReplicationSlotCleanup(false)
jit_reset_after_error()
MemoryContextSwitchTo(MessageContext)
FlushErrorState()
ignore_till_sync = true     // extended protocol 错误时
xact_started = false
RESUME_INTERRUPTS()
```

这条顺序体现了几个边界：

```text
先报告错误:
  ErrorContext 中的错误数据还不能 reset

再 abort transaction:
  事务资源和事务内存必须按 xact.c 的顺序清理

再做顶层专属 cleanup:
  auto-held Portal、replication slot、JIT 等不是纯事务内存

最后清 ErrorContext:
  错误已经发给客户端/日志，可以 reset 错误报告内存
```

### 第 3 步：`AbortCurrentTransaction()` 处理事务状态机

`AbortCurrentTransaction()` 不是简单调用一次 `AbortTransaction()`。它循环调用 `AbortCurrentTransactionInternal()`，因为错误可能发生在子事务、隐式事务、显式事务块、事务启动中途等不同状态。

对于单条语句隐式事务，核心路径是：

```text
AbortCurrentTransaction()
  -> AbortCurrentTransactionInternal()
     -> AbortTransaction()
     -> CleanupTransaction()
     -> blockState = TBLOCK_DEFAULT
```

对于显式事务块：

```sql
BEGIN;
SELECT 1 / 0;
SELECT 1;
```

第一次错误会 abort 当前事务，但 session 不一定回到完全 idle 的成功状态。客户端会看到事务块处于 failed transaction 状态，后续 SQL 通常报：

```text
current transaction is aborted, commands ignored until end of transaction block
```

直到 `ROLLBACK`。这不是 MemoryContext 的行为，而是 transaction block state machine 的语义。

### 第 4 步：`AbortTransaction()` 先恢复低层资源，再做事务语义 cleanup

`AbortTransaction()` 进入后先：

```text
HOLD_INTERRUPTS()
AtAbort_Memory()
AtAbort_ResourceOwner()
```

这两个调用建立 cleanup 自身的运行环境：

```text
CurrentMemoryContext -> TransactionAbortContext
CurrentResourceOwner -> TopTransactionResourceOwner
```

然后尽快清掉容易阻塞系统或污染后续 cleanup 的状态：

```text
LWLockReleaseAll()
WaitLSNCleanup()
pgstat_report_wait_end()
pgstat_progress_end_command()
pgaio_error_cleanup()
UnlockBuffers()
XLogResetInsertion()
ConditionVariableCancelSleep()
LockErrorCleanup()
reschedule_timeouts()
sigprocmask(SIG_SETMASK, &UnBlockSig, NULL)
```

这段非常适合说明“不能只靠 MemoryContext”的原因：

```text
LWLock 是共享内存并发控制状态；
buffer content lock 不是 palloc chunk；
WAL insertion state 不是普通内存生命周期；
等待队列和 timeout 状态如果不清，会影响下一次等待或误报 timeout；
lock manager 的等待状态必须先清掉，否则后续再等锁会出错。
```

随后事务进入 `TRANS_ABORT`，开始模块级 abort：

```text
SetUserIdAndSecContext(...)
ResetReindexState(...)
AtEOXact_Parallel(false)
AfterTriggerEndXact(false)
AtAbort_Portals()
smgrDoPendingSyncs(false, ...)
AtEOXact_LargeObject(false)
AtAbort_Notify()
AtEOXact_RelationMap(false, ...)
AtAbort_Twophase()
RecordTransactionAbort(false)
ProcArrayEndTransaction(...)
```

这说明 abort 不只是“释放资源”，还要把事务可见性和全局状态推进到一致点：

```text
如果分配了 XID，要记录 abort；
要从 ProcArray 中结束事务；
要让其它 backend 不再把当前事务当成 in-progress；
要取消事务内 pending side effects。
```

### 第 5 步：ResourceOwner 分 phase 释放非内存资源

`AbortTransaction()` 的后半段在 `TopTransactionResourceOwner != NULL` 时执行：

```text
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
```

然后还会运行一批模块级 end-of-xact cleanup：

```text
smgrDoPendingDeletes(false)
AtEOXact_GUC(false, 1)
AtEOXact_SPI(false)
AtEOXact_Enum()
AtEOXact_on_commit_actions(false)
AtEOXact_Namespace(false, ...)
AtEOXact_SMgr()
AtEOXact_Files(false)
AtEOXact_ComboCid()
AtEOXact_HashTables(false)
AtEOXact_PgStat(false, ...)
...
```

这段调用链传递的是：

```text
MemoryContext reset 可以回收这些模块分配的内存；
但模块状态的语义撤销必须由模块自己完成。
```

例如 GUC nesting、SPI stack、pending deletes、enum uncommitted values、namespace search path 临时状态，都不是靠“释放一些 chunk”就能恢复一致。

### 第 6 步：`CleanupTransaction()` 删除事务对象和事务内存

`AbortTransaction()` 结束时，事务状态仍是 `TRANS_ABORT`。真正释放事务内存发生在 `CleanupTransaction()`：

```text
AtCleanup_Portals()
AtEOXact_Snapshot(false, true)

CurrentResourceOwner = NULL
ResourceOwnerDelete(TopTransactionResourceOwner)
TopTransactionResourceOwner = NULL

AtCleanup_Memory()
```

顺序很重要：

```text
Portal cleanup 可能还需要访问 PortalData、cleanup hook、cached plan 信息；
snapshot cleanup 需要在 resource owner 删除前完成；
ResourceOwnerDelete 发生在资源 release 之后；
transaction memory 最后 reset。
```

`AtCleanup_Memory()` 才是内存侧收口：

```text
MemoryContextSwitchTo(s->priorContext)
MemoryContextReset(TransactionAbortContext)
MemoryContextReset(TopTransactionContext)
CurTransactionContext = NULL
```

因此，在 `ERROR` 后被自动回收的内存主要是：

```text
ErrorContext 中的错误报告数据
当前 MessageContext 中的消息级临时数据
TopTransactionContext / CurTransactionContext 下的事务级数据和子 context
Portal cleanup/delete 能触达的 portalContext 子树
executor per-query/per-tuple context
```

前提是这些 chunk 挂在了正确的 context 下。

### 第 7 步：Portal 的两阶段错误 cleanup

Portal 在 ERROR 路径上有两层 cleanup：

```text
AbortTransaction()
  -> AtAbort_Portals()

CleanupTransaction()
  -> AtCleanup_Portals()

PostgresMain 顶层 handler
  -> PortalErrorCleanup()
```

`AtAbort_Portals()` 的职责不是马上删除所有 Portal 结构。它会：

```text
把相关 Portal 标记为 failed
调用 portal->cleanup hook
释放 cached plan 引用
把 portal->resowner 置空，因为事务资源会在 transaction-wide cleanup 中释放
删除 portalContext 的 children，释放 executor 等附属内存
```

`AtCleanup_Portals()` 在 post-abort cleanup 阶段删除不应保留的 Portal。

`PortalErrorCleanup()` 是顶层错误恢复特有逻辑，源码注释说明它不同于 transaction abort cleanup：

```text
auto-held portals are cleaned up on error but not on transaction abort
```

这体现出一个模式：

```text
事务 abort 能处理事务语义；
顶层 ERROR recovery 还要处理协议或命令执行框架临时制造的特殊对象。
```

## 6. 生命周期 / ownership / cleanup

### ERROR 后会自动回收的内存

可以依赖 MemoryContext 的对象必须满足两个条件：

```text
它是 backend-local palloc 内存；
它挂在会被当前 error recovery reset/delete 的 context 子树中。
```

典型安全对象：

| 对象 | 释放边界 |
| --- | --- |
| 错误 message/detail/hint/context 字符串 | `FlushErrorState()` reset `ErrorContext` |
| 当前协议消息解析中间对象 | 下一轮 main loop reset `MessageContext` |
| 单条语句 executor 工作内存 | executor end、Portal cleanup 或事务 abort |
| per-tuple 表达式临时结果 | `ResetExprContext()` 或 executor cleanup |
| 事务级 pending list 的内存部分 | `AtCommit_Memory()` / `AtCleanup_Memory()` reset `TopTransactionContext` |
| 子事务 context 中的对象 | subtransaction abort cleanup 或 top-level cleanup |
| Portal 子 context 下的 executor 附属对象 | `AtAbort_Portals()` / `PortalDrop()` / `AtCleanup_Portals()` |

这类对象的关键不是“是否显式 pfree”，而是：

```text
如果 ERROR 跳过了普通返回路径，是否仍有上层 lifecycle owner 能统一 reset/delete？
```

### 不能只靠 MemoryContext 的资源

不能只靠 MemoryContext 的对象有一个共同特征：

```text
释放它不是单纯把当前进程的一段 palloc 内存还给 allocator。
```

常见类别：

| 资源 | 为什么不能只靠 MemoryContext |
| --- | --- |
| heavyweight lock / predicate lock | 需要更新 lock manager 共享状态，且有 commit/abort/subtransaction reassign 语义。 |
| LWLock | 共享内存轻量锁，必须尽快 `LWLockReleaseAll()`，否则可能阻塞其它 backend 或 cleanup 自己。 |
| buffer pin / buffer content lock | pin 影响 buffer replacement 和并发访问；content lock 是并发控制状态。 |
| snapshot registration | 影响 xmin horizon 和可见性边界；需要 snapshot manager / ResourceOwner 释放。 |
| relation/cache refcount | 影响 cache entry 是否仍被使用；可能需要 refcount decrement 和 invalidation 语义。 |
| cached plan refcount | Portal 持有 plan 引用时，必须通过 `PortalReleaseCachedPlan()` 等路径释放。 |
| 临时文件 / fd / DSM / AIO handle | OS 或 shared-memory 资源，不是 palloc chunk。 |
| 等待队列 / condition variable sleep | 必须从等待状态撤出，否则后续等待或唤醒语义会混乱。 |
| shared memory transient flag | 其它进程可见，当前 backend 的 MemoryContext reset 不会改共享状态。 |

这些资源通常应该挂到：

```text
ResourceOwner
模块级 AtEOXact / AtAbort hook
MemoryContext reset callback
PG_TRY / PG_CATCH cleanup
PG_ENSURE_ERROR_CLEANUP
before_shmem_exit / on_shmem_exit callback
```

具体选哪一个，取决于资源语义，而不是写代码时哪个 API 更方便。

### `PG_TRY` / `PG_CATCH` 的 ownership 语义

`PG_TRY` 适合局部恢复：

```text
临时切换全局变量，ERROR 后必须恢复
临时注册 callback，ERROR 后必须 unregister
调用第三方库或模块时，需要把局部状态恢复后再 rethrow
```

但 `elog.h` 注释给了一个非常强的约束：

```text
error recovery code either rethrows or aborts transaction;
否则系统可能处于不一致状态。
```

也就是说，`PG_CATCH` 不是“吞掉错误继续执行”的普通异常处理。除非代码非常清楚自己把所有相关状态恢复到了可继续执行的状态，否则应该：

```text
PG_RE_THROW();
```

或执行明确的事务/subtransaction abort 流程。

另一个边界是：

```text
PG_TRY 捕获 ERROR；
FATAL 不会被这个 construct 捕获，而是走 proc_exit。
```

所以涉及 shared-memory transient state 时，只在 `PG_CATCH` 里清理通常不够。

### `PG_ENSURE_ERROR_CLEANUP` 与进程退出 callback

`src/include/storage/ipc.h` 提供 `PG_ENSURE_ERROR_CLEANUP(cleanup_function, arg)`。它的目标很明确：

```text
一段代码可能 ERROR 或 FATAL 退出；
cleanup_function 必须在错误退出时运行；
典型场景是撤销 shared-memory transient state。
```

这个宏通过 `before_shmem_exit` 加 `PG_TRY` 组合实现：

```text
ERROR:
  PG_CATCH 中取消 before_shmem_exit，然后调用 cleanup_function，再 rethrow

FATAL:
  进程退出路径运行 before_shmem_exit callback
```

这和 MemoryContext 是完全不同的层：

```text
MemoryContext reset 只发生在当前进程仍可恢复的 cleanup 流程中；
FATAL/PANIC 可能直接进入进程退出；
shared memory 中的状态必须考虑 backend 死亡时谁来撤销。
```

## 7. 正确性机制层次

### 层次 1：错误报告自身必须可完成

错误报告可能发生在 OOM、递归错误、signal handler 打断之后。`ErrorContext` 的存在是为了让错误报告本身有较高概率完成。

相关机制：

```text
ErrorContext 早期初始化
保留少量可用空间
错误递归时 reset ErrorContext
in_error_recursion_trouble() 时放弃部分 context traceback 和 debug query
```

这层解决的是：

```text
至少能把错误发出去，并跳到 handler。
```

### 层次 2：控制流必须回到已知 handler

`PG_exception_stack` 保证 `ERROR` 有目标 handler。没有 handler 时，`ERROR` 会被升级为更严重的退出路径。

顶层 `PostgresMain` handler 保证普通 backend 在可恢复错误后能够回到主循环。它还负责协议相关状态：

```text
extended query protocol 错误后 ignore_till_sync
ReadyForQuery 状态由 transaction block status 决定
protocol synchronization lost 时 FATAL
```

这层解决的是：

```text
backend 和客户端协议不会在半条消息中继续错位执行。
```

### 层次 3：事务可见性和全局状态必须恢复

`AbortTransaction()` 不是简单 cleanup。它要把事务从 in-progress 推进到 aborted，并让其它 backend 看到一致状态：

```text
RecordTransactionAbort()
ProcArrayEndTransaction()
AtEOXact_Inval(false)
AtEOXact_MultiXact()
AtEOXact_PgStat(false, ...)
```

这层解决的是：

```text
其它 backend 的 visibility、lock wait、cache invalidation、统计状态不会因为本 backend ERROR 而停在中间态。
```

### 层次 4：资源引用必须按语义释放

ResourceOwner 的 phase 顺序、Portal cleanup、snapshot cleanup、buffer cleanup 都属于这一层。

这层解决的是：

```text
“这个进程还引用着某个共享对象或外部对象吗？”
```

不是：

```text
“这段内存有没有 free？”
```

### 层次 5：backend-local 内存最后批量回收

事务和消息级 cleanup 的尾部才适合 reset context：

```text
AtCleanup_Memory()
MemoryContextReset(MessageContext)
FlushErrorState()
```

原因是前面的 cleanup 还可能需要访问这些 context 中的结构体。太早 reset 会制造 dangling pointer。

这层解决的是：

```text
长连接不会保留失败语句、失败事务或失败 Portal 的 palloc 内存。
```

## 8. 错误路径 / 异常路径 / fallback

### 错误报告时再次出错

`elog.c` 明确处理递归错误：

```text
if (recursion_depth++ > 0 && elevel >= ERROR)
    MemoryContextReset(ErrorContext);
```

递归错误时，原错误报告可能丢失部分信息。系统宁愿丢掉一些诊断信息，也要避免在错误报告构造中无限递归。

如果递归很严重，还会放弃：

```text
error_context_stack
debug_query_string
```

这不是功能缺陷，而是错误路径的生存策略：

```text
在系统已经不稳定时，优先保证能退出当前错误，而不是保留最完整的诊断上下文。
```

### OOM 也是 ERROR

`palloc()` 默认 OOM 时不会返回 `NULL`，而是 `ereport(ERROR)`。这意味着：

```text
多数代码不需要检查 palloc 返回值；
但必须保证 ERROR cleanup 能释放已经分配的 transient state。
```

`ErrorContext` 保留少量空间，就是为了让 OOM 能像普通 ERROR 一样报告，而不是立即 FATAL。

但这不代表所有 OOM 都能优雅恢复。如果错误发生在错误处理本身、进程退出中、critical section 或无 handler 区域，可能升级。

### subtransaction abort

子事务错误通常通过 subtransaction abort 隔离。内存侧：

```text
AtSubAbort_Memory()
  -> switch to TransactionAbortContext

AtSubCleanup_Memory()
  -> MemoryContextDelete(failed subxact CurTransactionContext)
```

ResourceOwner 和 Portal 也有 subtransaction 对应路径：

```text
AtSubAbort_ResourceOwner()
AtSubAbort_Portals()
AtSubCleanup_Portals()
ResourceOwnerRelease(s->curTransactionOwner, ..., false, false)
```

这带来一个扩展开发中常见的坑：

```text
如果上层事务列表保存了指向子事务 context 中对象的指针；
子事务 abort 后这个指针会悬空；
必须在 subabort callback 中从上层状态里 delink。
```

### FATAL / PANIC 路径

`PG_TRY` 不捕获 `FATAL`。`FATAL` 会走 `proc_exit()` / `shmem_exit()` 方向，执行：

```text
before_shmem_exit callbacks
dsm_backend_shutdown()
on_shmem_exit callbacks
on_proc_exit callbacks
```

因此：

```text
process-local palloc 内存不需要逐块释放，进程退出会交给 OS；
shared memory 中的进程注册、ProcArray、semaphore、DSM、replication slot 等必须有退出 callback 兜底；
临时 shared state 不能只在 PG_CATCH 中撤销。
```

### `PG_CATCH` 里再次 ERROR

`elog.h` 注释提醒：恢复段里可以传播新的 `ERROR`，但嵌套层数有限，而且恢复代码最好足够简单。

实践规则：

```text
PG_CATCH 中少分配、少拿锁、少做复杂查询；
先恢复必要局部状态；
尽快 PG_RE_THROW()；
如果需要保留原错误信息，使用 CopyErrorData() 后及时 FlushErrorState()。
```

### MemoryContext reset callback 自己 ERROR

`MemoryContextCallResetCallbacks()` 在调用 callback 前把它从链上摘掉。这样后续再次 reset/delete 不会重复调用同一个 callback，避免无限循环。

但 callback 抛 ERROR 仍然危险：

```text
当前 context 可能没有完成 reset/delete；
父 context tree 可能只能选择泄漏而不是崩溃；
cleanup callback 应尽量不可失败。
```

所以 reset callback 应保持小而确定，不要把复杂事务语义塞进去。

## 9. 成本、资源与跨模块传播

### ERROR cleanup 的成本不是常规 hot path 成本

正常执行中，`ERROR` 路径不是 hot path。但错误路径成本仍然重要：

```text
频繁 constraint violation
大量 PL/pgSQL exception block
COPY 中大量坏行处理
扩展用 ERROR 做控制流
测试或批处理反复触发错误
```

在这些 workload 中，成本来自：

```text
错误消息构造和 error context callback
transaction abort cleanup
ResourceOwnerRelease 扫描和排序
Portal / executor cleanup
MessageContext / transaction context reset
协议同步恢复
```

把 `ERROR` 当普通分支使用，通常会把慢路径变成工作负载的主路径。

### ResourceOwner release 成本随持有资源数增长

ResourceOwner 会跟踪不同类型资源。释放时需要按 phase 和 priority 处理。事务中持有的：

```text
buffer pins
catcache/relcache refs
snapshot refs
locks
AIO handles
```

越多，abort cleanup 越重。

这解释了为什么 PostgreSQL 很多模块强调正常路径要及时释放资源：

```text
正常释放减少 abort cleanup 压力；
commit 时残留资源还能暴露 programmer error；
abort 时则需要安静兜底。
```

### context reset 成本随 context tree 和 allocated blocks 增长

`MemoryContextReset(context)` 会先 delete children，再 reset 当前 context。树越深、children 越多、allocator 持有 blocks 越多，cleanup 成本越高。

这不是说要避免创建 context，而是要避免：

```text
把大量短生命周期 context 挂到过长生命周期下
让失败路径积累很多本可逐批 reset 的对象
在 TopMemoryContext 下制造只能 backend 退出才释放的 retention
```

### cleanup 顺序跨模块传播

`AbortTransaction()` 里一长串 `AtEOXact_*` 调用不是偶然。每个模块都有自己的状态语义：

```text
executor:
  QueryDesc / EState / tuple table / ExprContext

storage:
  buffer pin / local buffer / pending deletes / smgr state

lock manager:
  heavyweight locks / wait state / predicate locks

snapshot manager:
  registered snapshot / active snapshot / exported snapshot

catalog/cache:
  relcache / catcache refcount and invalidation

protocol / tcop:
  MessageContext / ReadyForQuery / extended protocol Sync
```

错误恢复是这些模块的共同边界。一个模块如果在 ERROR 后留下状态，下一条完全无关的 SQL 也可能被污染。

## 10. 观测与诊断入口

### SQL 观测 MemoryContext

当前 backend 可以看：

```sql
SELECT name, level, total_bytes, used_bytes, free_bytes
FROM pg_backend_memory_contexts
ORDER BY level, name;
```

关注这些名字：

```text
TopMemoryContext
ErrorContext
MessageContext
TopTransactionContext
PortalContext 或 Portal 相关 context
ExecutorState
ExprContext
```

一个简单现象：

```sql
SELECT name, used_bytes
FROM pg_backend_memory_contexts
WHERE name IN ('ErrorContext', 'MessageContext', 'TopTransactionContext');

SELECT 1 / 0;

SELECT name, used_bytes
FROM pg_backend_memory_contexts
WHERE name IN ('ErrorContext', 'MessageContext', 'TopTransactionContext');
```

你通常不会在第二次查询后看到 ErrorContext 长期保留大量错误字符串，因为顶层 recovery 会 `FlushErrorState()`。

注意：`pg_backend_memory_contexts` 只能看当前 backend 的 context 统计。它看不到：

```text
锁是否释放
buffer pin 是否还在
snapshot 是否注册
等待队列是否清理
shared memory transient flag 是否撤销
```

### 日志观测 memory context

可以用：

```sql
SELECT pg_log_backend_memory_contexts(pg_backend_pid());
```

它会把目标 backend 的 memory context tree 打到 server log。适合在怀疑 ERROR 后内存没有释放时观察：

```text
某个 context 是否仍挂在 TopMemoryContext 下
某个 Portal / ExecutorState context 是否异常保留
MessageContext 是否每轮被 reset
```

但它仍然只是内存视角。

### SQL 观测事务状态

在显式事务中制造错误：

```sql
BEGIN;
SELECT 1 / 0;
SELECT txid_current();
ROLLBACK;
```

第二条 `SELECT` 会被事务状态机拦住。这个现象说明：

```text
ERROR 后 backend 没有崩；
但当前 transaction block 已进入 failed 状态；
需要 ROLLBACK 才能回到普通 idle。
```

这属于 transaction block state，不是 MemoryContext 泄漏。

### 观测锁和等待状态

如果怀疑 ERROR 后锁或等待状态没有正确释放，可以从另一个 session 看：

```sql
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE pid = <target_pid>;

SELECT pid, wait_event_type, wait_event, state
FROM pg_stat_activity
WHERE pid = <target_pid>;
```

对于普通语句 ERROR，事务 abort 后大多数事务级锁应释放。但显式事务块 failed 状态下，有些事务语义相关状态仍要等 `ROLLBACK` 收尾。诊断时要区分：

```text
statement-level error recovery 已完成
transaction block 仍处于 failed transaction
backend session 仍然存活
```

### gdb 断点阅读路径

适合课堂或本地源码阅读的断点：

```text
errfinish
pg_re_throw
PostgresMain 中 sigsetjmp handler 块
AbortCurrentTransaction
AbortTransaction
CleanupTransaction
AtAbort_Memory
AtCleanup_Memory
ResourceOwnerRelease
MemoryContextReset
FlushErrorState
AtAbort_Portals
AtCleanup_Portals
PortalErrorCleanup
```

观察点：

```text
CurrentMemoryContext 何时指向 ErrorContext
何时切到 TransactionAbortContext
何时切回 MessageContext
TopTransactionContext 何时 reset
TopTransactionResourceOwner 何时 release/delete
```

## 11. 常见误区

### 误区 1：ERROR 后 MemoryContext 会清理所有东西

MemoryContext 只负责它能负责的东西：

```text
palloc chunk
context children
reset callback 中显式处理的附属资源
```

锁、pin、snapshot、共享内存状态、OS fd、等待队列都需要自己的 cleanup 机制。

### 误区 2：只要注册 MemoryContext reset callback，就不用 ResourceOwner

reset callback 不知道 commit/abort、subtransaction reassign、release phase、lock ordering。它适合绑定对象附属资源，不适合作为事务资源总账。

### 误区 3：PG_CATCH 可以随便吞掉 ERROR

`PG_CATCH` 后如果不 rethrow、不 abort，也没有完整恢复所有状态，系统可能继续在半失败状态下运行。绝大多数 handler 应做局部恢复后 `PG_RE_THROW()`。

### 误区 4：ErrorContext 是错误对象的长期保存区

`ErrorContext` 是错误报告工作区。顶层 recovery 会 `FlushErrorState()` reset 它。需要跨出 error handler 使用错误信息时，要用 `CopyErrorData()` 等 API，并理解复制后的 ownership。

### 误区 5：ERROR 和 FATAL cleanup 一样

`ERROR` 可以被 `PG_TRY` 或顶层 handler 捕获并恢复到 main loop。`FATAL` 走进程退出，`PG_TRY` 不捕获。涉及 shared memory 或外部资源时，必须考虑 `before_shmem_exit` / `on_shmem_exit` 兜底。

### 误区 6：顶层 handler 里可以随便加模块 cleanup

`PostgresMain` 顶层 handler 注释明确建议：大多数 cleanup 应该放在 `AbortTransaction()` 或模块自己的 xact cleanup 中。顶层 handler 只处理 outer-level error recovery 特有的协议、连接和全局边界。

### 误区 7：看到 TopMemoryContext 增长就一定是 leak

有些长生命周期对象本来就在 `TopMemoryContext` 或其长期子 context 下，例如 cache、portal manager 基础设施、fd 管理表。诊断要看：

```text
对象是否应该跨整个 backend session
是否有模块自己的 delete/reset
是否随每次 ERROR 单调增长
```

## 12. 课堂实验

### 实验 1：观察 ErrorContext 不长期保存错误字符串

步骤：

```sql
SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name = 'ErrorContext';

SELECT 1 / 0;

SELECT name, total_bytes, used_bytes
FROM pg_backend_memory_contexts
WHERE name = 'ErrorContext';
```

预期现象：

```text
错误已经报告给客户端；
后续查询仍可执行；
ErrorContext 不应因为错误消息长期明显膨胀。
```

回到源码解释：

```text
errfinish() 在 ErrorContext 构造错误报告；
PostgresMain 顶层 handler EmitErrorReport() 后调用 FlushErrorState()；
FlushErrorState() reset ErrorContext。
```

### 实验 2：显式事务中 ERROR 后观察 transaction block 状态

步骤：

```sql
BEGIN;
SELECT 1 / 0;
SELECT 42;
ROLLBACK;
SELECT 42;
```

预期现象：

```text
第一次 SELECT 报错；
第二次 SELECT 在 failed transaction 中被拒绝；
ROLLBACK 后 session 恢复。
```

回到源码解释：

```text
ERROR 后 AbortCurrentTransaction() 按 blockState 处理；
显式事务块不会简单回到 TBLOCK_DEFAULT；
ReadyForQuery 会反映 transaction block status。
```

### 实验 3：对比内存 cleanup 和锁 cleanup

Session A：

```sql
BEGIN;
LOCK TABLE pg_class IN ACCESS SHARE MODE;
SELECT 1 / 0;
```

Session B：

```sql
SELECT pid, locktype, relation::regclass, mode, granted
FROM pg_locks
WHERE relation = 'pg_class'::regclass;
```

然后 Session A：

```sql
ROLLBACK;
```

观察点：

```text
错误后事务块处于 failed 状态；
锁释放行为要按事务 block 语义理解；
内存是否 reset 和锁是否释放不是同一个问题。
```

回到源码解释：

```text
锁由 lock manager 和 ResourceOwnerRelease 处理；
MemoryContext reset 不会直接释放 lock manager 中的状态。
```

### 实验 4：用 gdb 跟踪 `SELECT 1 / 0`

建议断点：

```text
errfinish
pg_re_throw
AbortCurrentTransaction
AbortTransaction
CleanupTransaction
AtAbort_Memory
ResourceOwnerRelease
AtCleanup_Memory
FlushErrorState
```

观察变量：

```text
CurrentMemoryContext->name
TopTransactionContext
CurTransactionContext
CurrentResourceOwner
errordata_stack_depth
```

预期路径：

```text
errfinish 切到 ErrorContext
pg_re_throw longjmp
PostgresMain handler 报告错误
AbortTransaction 切到 TransactionAbortContext
CleanupTransaction reset TopTransactionContext
FlushErrorState reset ErrorContext
主循环下一轮 reset MessageContext
```

### 实验 5：源码练习，找一个 cleanup 不能只靠 MemoryContext 的例子

任选一个入口阅读：

```text
LockErrorCleanup()
UnlockBuffers()
AtEOXact_Buffers(false)
AtEOXact_Snapshot(false, true)
PortalReleaseCachedPlan()
AtEOXact_Files(false)
```

回答：

```text
它清理的状态在哪里？
是否涉及共享内存、refcount、pin、fd 或全局列表？
如果只 reset MemoryContext，会留下什么错误状态？
```

## 13. 讨论题

1. 为什么 `PostgresMain` 顶层 handler 要先 `EmitErrorReport()`，再 `AbortCurrentTransaction()`，最后 `FlushErrorState()`？如果先 reset `ErrorContext` 会怎样？

2. `AtAbort_Memory()` 为什么切到 `TransactionAbortContext`，而不是继续使用当前 `CurrentMemoryContext`？

3. 为什么 `AbortTransaction()` 一开始要尽快 `LWLockReleaseAll()`，而普通 heavyweight locks 不能同样简单地全部先释放？

4. 一个扩展在 shared memory 中设置了“我正在处理任务 X”的标志。它只在 `PG_CATCH` 中清这个标志够不够？如果 `FATAL` 呢？

5. `MemoryContextRegisterResetCallback()` 和 ResourceOwner 都能做 cleanup。如何判断一个资源应该放在哪边？

6. 为什么 commit 路径中残留资源可能 warning，而 abort 路径中通常安静释放？

7. 子事务 abort 后，为什么上层事务中不能保留指向子事务 context 的对象指针？

8. 如果一个 workload 把唯一约束冲突当作高频控制流，ERROR cleanup 会把哪些原本的慢路径成本变成主成本？

## 14. 本节小结

本节的唯一主问题是：

```text
ERROR longjmp 后，哪些内存会被 context reset/delete 自动回收，哪些资源不能只靠 MemoryContext 兜底？
```

答案可以压缩成四句话：

```text
ErrorContext 只负责错误报告本身，并在报告完成后 reset。

挂在 MessageContext、TopTransactionContext、Portal context、executor context 等正确生命周期下的 backend-local palloc 内存，会在对应 reset/delete 边界被批量回收。

lock、LWLock、buffer pin、snapshot、cached plan refcount、fd、DSM、等待状态、shared-memory transient state 等不是普通 palloc chunk，必须由 ResourceOwner、模块级 abort hook、Portal cleanup、reset callback 或进程退出 callback 处理。

PG_TRY/PG_CATCH 是局部恢复工具，不是随意吞掉 ERROR 的异常系统；多数 cleanup 后仍应 rethrow 或进入明确的 abort 流程。
```

可迁移的系统规律是：

```text
异常恢复不能只问“内存会不会释放”；
还要问“哪些外部引用、共享状态、协议状态和可见性状态必须回到一致点”。
```

MemoryContext 给 PostgreSQL 提供了可靠的 backend-local 内存回收骨架；`ERROR` recovery 则把这个骨架接到事务、Portal、ResourceOwner、IPC callback 和协议状态机上。下一节可以进一步进入 allocator 与诊断：当 cleanup 边界已经正确时，为什么还要区分 `AllocSet`、`Generation`、`Slab`、`Bump`，以及如何用 memory context 统计判断内存形态是否符合预期。
