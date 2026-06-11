# PostgreSQL abort 路径与未完成事务判定

## 课程定位

前置知识：

- 你已经知道 tuple header 中的 `xmin` / `xmax` 会保存写入者或删除者的 XID。
- 你已经知道 `pg_xact` / CLOG 用 2 bit 保存事务结果状态。
- 你已经知道普通 MVCC snapshot 用 `xmin`、`xmax`、`xip`、`subxip` 描述读视图。
- 你不需要先掌握 2PC、logical decoding 或 MultiXact 的完整实现。

本节唯一主问题：

- 一个 XID 已经写进 tuple header，但事务后来 ERROR、显式 `ROLLBACK` 或 backend 崩溃时，可见性代码如何区分“它还没结束”和“它已经失败”？

本节核心矛盾：

- tuple header 必须很早写出 XID，其他 backend 才能判断版本来源。
- 事务结果却必须到结束路径才确定，甚至可能因为崩溃没有显式 abort 记录。
- 可见性检查又必须在高频 heap scan 中尽量便宜，不能每次都全局同步。
- 因此 PostgreSQL 把“正在运行”放在 `ProcArray` / snapshot，把“最终结果”放在 `pg_xact`，并要求读取顺序遵守边界。

学完后你应该能独立判断：

- 为什么不能把 `pg_xact` 中的 `IN_PROGRESS` 直接解释为“当前还在运行”。
- 为什么 `TransactionIdDidAbort()` 不是 heap visibility 的通用答案。
- 为什么 abort 结束时必须先写 `pg_xact`，再从 `ProcArray` 清掉 XID。
- 为什么 MVCC snapshot 中的“in progress”是 snapshot 语义，而不是此刻真值。
- 为什么崩溃后的未完成事务可以按 aborted 解释，而不一定有显式 CLOG aborted 位。

本课基线：

- PostgreSQL 源码路径：`/home/highgo/postgres`。
- 分支：`master`。
- commit：`bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。

本节不展开：

- 子事务父链的完整推导，留到下一节 `pg_subtrans`。
- MultiXact 中 `xmax` 的锁语义，留到 row lock 主题。
- commit WAL flush 顺序的完整细节，上一节已经覆盖。


## 1. 本节在总主线中的位置

事务可见性第一组课程在回答一个连续问题：

- XID 什么时候分配。
- XID 结果存在哪里。
- commit 如何把结果安全地暴露给别人。
- abort 或崩溃时，别人如何解释已经写出的 XID。
- 子事务如何把局部结果折叠到顶层事务。
- subxid overflow 后成本如何退化。

本节处在 commit 之后、subtrans 之前。上一节关注：

- committed 状态何时能被别人信任。
- WAL flush、CLOG、ProcArray 之间的提交顺序。

本节关注：

- 没有 committed 结果时，tuple version 应该如何被排除。
- abort 结果何时显式写入 `pg_xact`。
- 没有显式 abort 结果时，visibility 为什么仍然能保持正确。

下一节会继续追问：

- 如果这个 XID 是 subtransaction XID，为什么不能只看它自己的 CLOG 状态。
- `SUB_COMMITTED` 为什么必须追溯到 parent XID。

本节的 runtime truth：

- 一个插入事务 ERROR 之后，heap page 上可能仍然保留带 `xmin` 的物理 tuple。
- 普通 MVCC SELECT 看不到它。
- page inspection 或 debugger 能看到 tuple header 里的 XID。
- 这个差异不是“数据丢了”，而是 visibility 通过事务结果边界把它解释为无效。


## 2. 核心矛盾与一句话运行模型

一句话运行模型：

- tuple header 只记录“谁写过”，`ProcArray` / snapshot 记录“谁在读视图中仍未完成”，`pg_xact` 记录“谁最终成功或失败”；可见性必须先排除未完成，再查询最终结果。

更具体地说：

- `xmin` 写进 tuple 时，写事务可能还没有决定 commit 或 abort。
- 其他 backend 可能马上扫描到这个 tuple。
- 如果写事务仍在运行，普通 MVCC snapshot 不应该看到它。
- 如果写事务已经 abort，普通 MVCC snapshot 也不应该看到它。
- 这两个结果相同，原因不同。
- 一个是“snapshot 边界内仍未完成”。
- 一个是“最终失败，不属于任何可见历史”。

PostgreSQL 不能把这两个状态合成一个简单字段。原因一：

- `pg_xact` 初始值是全 0。
- 全 0 对应 `TRANSACTION_STATUS_IN_PROGRESS`。
- 但崩溃前未完成的事务也可能长期在 CLOG 中保持这个值。
- 因此 CLOG 的 `IN_PROGRESS` 不是“此刻仍有 backend 正在运行”的完整事实。

原因二：

- `ProcArray` 是 volatile shared memory。
- backend 结束时会从 `ProcArray` 清掉自己的 XID。
- 崩溃重启后，旧 backend 的 `ProcArray` 状态已经不存在。
- 因此 `ProcArray` 只能回答当前运行集或 recovery 中的 known-assigned 集合。

原因三：

- MVCC snapshot 不是实时状态查询。
- snapshot 保存的是“拍快照时正在运行”的集合。
- 一个事务在 snapshot 之后 commit 或 abort，当前语句仍然要按 snapshot 语义解释。
- 因此 visibility 不能为了设置 hint bit 而随意重新查询全局运行状态并改变可见性结果。

这形成本节核心矛盾：

- correctness 需要严格区分 running、committed、aborted、crashed。
- performance 希望 heap scan 不要频繁碰 `ProcArrayLock` 和 CLOG SLRU。
- crash safety 又允许 abort 不必像 commit 一样强制 WAL flush。
- PostgreSQL 的做法是把状态拆到多个层次，并用固定读取顺序组合出语义。

本节最重要的不变量：

- 事务结束路径必须先把最终结果写入 `pg_xact`，再从 `ProcArray` 中移除 running XID。
- visibility 判断必须先判断“对当前 snapshot 是否仍未完成”，再查询 `pg_xact` 的最终结果。
- 对于没有显式 committed 的事务，visibility 常用“not in progress + not committed => aborted or crashed”的排除法。
- hint bit 只能缓存已经安全判定的结果，不能改变事务语义。


## 3. 核心文件分工与阅读顺序

建议阅读顺序不是按文件名排序，而是按状态生命周期排序。第 1 步：先读 `heapam_visibility.c` 文件头。

- 文件：`src/backend/access/heap/heapam_visibility.c`。
- 关键位置：文件开头关于 `TransactionIdIsInProgress`、`TransactionIdDidCommit`、`TransactionIdDidAbort` 的注释。
- 目的：先建立 visibility 的读取顺序。
- 重点：非 MVCC snapshot 要先查 `TransactionIdIsInProgress()`，再查 `TransactionIdDidCommit()`。
- 重点：MVCC snapshot 用 `XidInMVCCSnapshot()` 替代实时 `TransactionIdIsInProgress()`。
- 重点：不能普遍使用 `TransactionIdDidAbort()`，因为崩溃中止事务可能仍在 CLOG 里像 `IN_PROGRESS`。

第 2 步：读 `xact.c` 的事务状态结构。

- 文件：`src/backend/access/transam/xact.c`。
- 关键结构：`TransState`、`TBlockState`、`TransactionStateData`。
- 目的：区分 server 内部低层事务状态和 SQL 事务块状态。
- 重点：`TBLOCK_ABORT` 表示事务已经 abort，等待用户 `ROLLBACK` 清理。
- 重点：`TRANS_ABORT` 表示低层 abort 正在或已经进入 abort cleanup。

第 3 步：读 `AbortCurrentTransaction()` 状态机。

- 文件：`src/backend/access/transam/xact.c`。
- 关键函数：`AbortCurrentTransaction()`、`AbortCurrentTransactionInternal()`。
- 目的：理解 ERROR 路径和显式 ROLLBACK 路径如何进入同一个 abort 核心。
- 重点：ERROR 在事务块内会执行 `AbortTransaction()`，然后停留在 `TBLOCK_ABORT`。
- 重点：显式 ROLLBACK 在 live block 中先设置 `TBLOCK_ABORT_PENDING`，再由 `CommitTransactionCommand()` 执行 abort。

第 4 步：读 `AbortTransaction()`。

- 文件：`src/backend/access/transam/xact.c`。
- 关键函数：`AbortTransaction()`。
- 目的：抓住 abort 的顺序边界。
- 重点：先释放 LWLock、buffer content lock、wait state 等易死锁资源。
- 重点：调用 `RecordTransactionAbort(false)` 写 abort 结果。
- 重点：调用 `ProcArrayEndTransaction(MyProc, latestXid)` 宣告自己不再 running。
- 重点：之后才释放事务资源和锁。

第 5 步：读 `RecordTransactionAbort()`。

- 文件：`src/backend/access/transam/xact.c`。
- 关键函数：`RecordTransactionAbort(bool isSubXact)`。
- 目的：理解 abort 是否写 WAL、是否 flush、何时写 CLOG。
- 重点：没有分配 XID 的事务，不需要向 `pg_xact` 宣告 abort。
- 重点：abort 会写 `XLOG_XACT_ABORT`，但不强制 flush。
- 重点：abort 后调用 `TransactionIdAbortTree()` 标记 CLOG aborted。

第 6 步：读 `transam.c` 的高层接口。

- 文件：`src/backend/access/transam/transam.c`。
- 关键函数：`TransactionIdDidCommit()`、`TransactionIdDidAbort()`、`TransactionIdAbortTree()`。
- 目的：理解 heap visibility 调用的不是 CLOG 低层接口，而是事务语义包装。
- 重点：`TransactionIdDidAbort()` 只对显式 aborted 返回 true。
- 重点：多数 visibility 路径应使用 `TransactionIdDidCommit()` 加 running check 的组合。

第 7 步：读 `clog.h` 和 `clog.c`。

- 文件：`src/include/access/clog.h`。
- 文件：`src/backend/access/transam/clog.c`。
- 关键状态：`TRANSACTION_STATUS_IN_PROGRESS`、`COMMITTED`、`ABORTED`、`SUB_COMMITTED`。
- 关键函数：`TransactionIdSetTreeStatus()`、`TransactionIdSetStatusBit()`、`TransactionIdGetStatus()`。
- 目的：理解 CLOG 只保存小状态，不保存 running 的完整来源。
- 重点：CLOG 初始全 0，所以 `IN_PROGRESS` 是默认值。
- 重点：abort 设置不要求 WAL flush 后才能落 CLOG。

第 8 步：读 `procarray.c`。

- 文件：`src/backend/storage/ipc/procarray.c`。
- 关键函数：`ProcArrayEndTransaction()`、`ProcArrayEndTransactionInternal()`、`TransactionIdIsInProgress()`。
- 目的：理解 running 判定从哪里来。
- 重点：`ProcArrayEndTransaction()` 用于 commit 和 abort，调用者必须已经写好 WAL / `pg_xact`。
- 重点：它清掉 `proc->xid`、`ProcGlobal->xids[]`、subxid cache，并推进 `latestCompletedXid`。
- 重点：`TransactionIdIsInProgress()` 先用 shortcut，再扫 ProcArray，必要时走 subtrans slow path。

第 9 步：最后回到 `HeapTupleSatisfiesMVCC()`。

- 文件：`src/backend/access/heap/heapam_visibility.c`。
- 关键函数：`HeapTupleSatisfiesMVCC()`。
- 目的：把 abort 结果映射成 tuple visible / invisible。
- 重点：`xmin` 未提交且不在 snapshot 中时，查 `TransactionIdDidCommit()`。
- 重点：如果没 commit，就按 aborted or crashed 处理并可设置 `HEAP_XMIN_INVALID`。
- 重点：`xmax` 未提交且不在 snapshot 中时，如果没 commit，就按删除失败处理并可设置 `HEAP_XMAX_INVALID`。


## 4. 关键数据结构与状态

### 4.1 `TransactionStateData`

位置：

- `src/backend/access/transam/xact.c`。

性质：

- backend-local。
- 每个 backend 自己维护当前事务栈。
- 其他 backend 不能直接用它判断 visibility。

关键字段组合：

- `fullTransactionId`：当前 transaction 或 subtransaction 的 FullXID。
- `subTransactionId`：本地 subtransaction 层级标识，不等同于全局 XID。
- `state`：低层 server transaction 状态，例如 `TRANS_INPROGRESS`、`TRANS_ABORT`。
- `blockState`：SQL transaction block 状态，例如 `TBLOCK_INPROGRESS`、`TBLOCK_ABORT`。
- `curTransactionContext`：事务生命周期内存上下文。
- `curTransactionOwner`：事务资源 owner。
- `childXids` / `nChildXids`：已经 subcommitted 的子事务 XID，供顶层 commit / abort 汇总。
- `parent`：子事务指向父事务。

本节只需要记住：

- `TransactionStateData` 是本 backend 的控制面。
- 它解释“我现在处于什么事务路径”。
- 它不直接回答“别的 backend 应该看见我的 tuple 吗”。

常见边界：

- `state = TRANS_ABORT` 不意味着 tuple header 已经被清掉。
- `blockState = TBLOCK_ABORT` 不意味着本地资源都已经释放。
- ERROR 后事务块可能已经 abort，但还在等待用户发送 `ROLLBACK` 完成 cleanup。

### 4.2 `TransState` 和 `TBlockState`

`TransState` 是低层状态：

- `TRANS_DEFAULT`：idle。
- `TRANS_START`：事务启动中。
- `TRANS_INPROGRESS`：有效事务中。
- `TRANS_COMMIT`：commit in progress。
- `TRANS_ABORT`：abort in progress。
- `TRANS_PREPARE`：prepare in progress。

`TBlockState` 是 SQL 事务块状态：

- `TBLOCK_STARTED`：单语句事务。
- `TBLOCK_INPROGRESS`：显式事务块中。
- `TBLOCK_ABORT`：事务已经失败，等待 `ROLLBACK`。
- `TBLOCK_ABORT_PENDING`：用户在 live block 中请求 `ROLLBACK`。
- `TBLOCK_ABORT_END`：失败事务块收到了 `ROLLBACK`，等待 cleanup。
- `TBLOCK_SUBABORT` 等状态用于 savepoint / subtransaction。

为什么这两个状态都需要：

- SQL 协议需要知道之后的命令是否只能报 “current transaction is aborted”。
- 内核 cleanup 需要知道资源是否已经走过 abort 核心路径。
- 可见性只关心最终 XID 结果，不直接读取这些 backend-local 状态。

### 4.3 `pg_xact` / CLOG 状态

定义位置：

- `src/include/access/clog.h`。

状态集合：

- `TRANSACTION_STATUS_IN_PROGRESS = 0x00`。
- `TRANSACTION_STATUS_COMMITTED = 0x01`。
- `TRANSACTION_STATUS_ABORTED = 0x02`。
- `TRANSACTION_STATUS_SUB_COMMITTED = 0x03`。

关键性质：

- 每个 XID 用 2 bit。
- all-zero 是初始状态，所以默认就是 `IN_PROGRESS`。
- `pg_xact` 是持久化状态的一部分。
- 它通过 SLRU 页面读写。
- 它不保存 backend PID。
- 它不保存 snapshot。
- 它不保存 tuple TID。

本节要特别小心：

- `IN_PROGRESS` 在 CLOG 中不是“当前一定有 backend 正在运行”。
- 它还可能表示“这个 XID 从未完成为 committed，也没有显式 abort 标记”。
- 崩溃前 in-progress 的事务，恢复后可按 aborted 解释。
- 所以 heap visibility 不能靠 `TransactionIdDidAbort()` 覆盖所有失败事务。

### 4.4 `PGPROC`、`ProcGlobal->xids[]` 与 running set

定义位置：

- `src/include/storage/proc.h`。
- `src/backend/storage/ipc/procarray.c`。

性质：

- shared memory。
- 当前活跃 backend 的事务身份入口。
- 高频路径使用密集数组 `ProcGlobal->xids[]`、`subxidStates[]`、`statusFlags[]` 降低扫描成本。

关键字段组合：

- `proc->xid`：backend 当前顶层 XID。
- `ProcGlobal->xids[pgxactoff]`：密集数组镜像，用于快速扫描。
- `proc->subxids.xids[]`：PGPROC 中缓存的 subxid。
- `proc->subxidStatus.count`：缓存 subxid 数。
- `proc->subxidStatus.overflowed`：subxid cache 是否溢出。
- `proc->xmin`：backend 当前声明的 xmin horizon。
- `proc->vxid.lxid`：virtual transaction 身份。

本节要记住：

- running 判定主要来自 `ProcArray`，不是 CLOG。
- 事务结束必须在 `ProcArrayLock` 下清掉 shared running 信息。
- 清掉 running 信息之前，事务结果必须已经写到 `pg_xact`。

### 4.5 tuple header 中的事务字段和 hint bits

定义位置：

- `src/include/access/htup_details.h`。

关键字段：

- `t_choice.t_heap.t_xmin`：插入者 XID。
- `t_choice.t_heap.t_xmax`：删除者、更新者或 locker。
- `t_infomask`：visibility 相关 hint bits 和语义位。
- `t_ctid`：版本链或自身 TID。

本节相关 hint bits：

- `HEAP_XMIN_COMMITTED`：`xmin` 已知 committed。
- `HEAP_XMIN_INVALID`：`xmin` invalid / aborted。
- `HEAP_XMAX_COMMITTED`：`xmax` 已知 committed。
- `HEAP_XMAX_INVALID`：`xmax` invalid / aborted。
- `HEAP_XMAX_LOCK_ONLY`：`xmax` 只是锁，不是删除者。
- `HEAP_XMAX_IS_MULTI`：`xmax` 是 MultiXactId。

本节只关注：

- `xmin` abort 意味着这个 tuple version 从未出生。
- `xmax` abort 意味着删除或更新失败，旧版本仍然活着。
- hint bit 是缓存，不是事务结果的来源。
- hint bit 可以滞后，也可能尚未设置。

### 4.6 `SnapshotData`

定义位置：

- `src/include/utils/snapshot.h`。

本节相关字段：

- `xmin`：小于它的 XID 对此 snapshot 不再 in-progress。
- `xmax`：大于等于它的 XID 对此 snapshot 不可见。
- `xip`：普通运行事务 XID 集合。
- `subxip`：运行子事务 XID 集合，recovery 中也可能装所有 known assigned XID。
- `suboverflowed`：subxip 是否不完整。
- `takenDuringRecovery`：snapshot 形态是否来自 recovery。
- `curcid`：同事务内 command 边界。

关键边界：

- `XidInMVCCSnapshot(xid, snapshot)` 问的是“按这个 snapshot，xid 是否仍未完成”。
- 它不是实时 `ProcArray` 查询。
- 它允许当前语句在事务后来结束后仍保持一致读视图。


## 5. 主流程源码 walkthrough

本节只走一条主线：

- 一个事务插入 tuple。
- XID 已经写入 `xmin`。
- 事务遇到 ERROR 或 ROLLBACK。
- abort 路径写结果并清 running set。
- 另一个 MVCC reader 扫描这个 tuple。
- reader 把 tuple 解释为不可见。

### 5.1 XID 先于事务结果出现

入口：

- 写 tuple 前，代码需要 `GetCurrentTransactionId()` 或等价路径拿到 XID。

相关函数：

- `GetCurrentTransactionId()`。
- `AssignTransactionId()`。
- `GetNewTransactionId()`。

重要顺序：

- `AssignTransactionId()` 为当前 `TransactionState` 分配 FullXID。
- 顶层事务保存到 `XactTopFullTransactionId`。
- 子事务会先确保父事务有 XID。
- 子事务会写 `pg_subtrans` parent 信息。
- XID 会登记到 `PGPROC` / shared transaction machinery。
- 随后 heap insert / update 可以把 XID 写入 tuple header。

此时还没有事务结果。tuple header 表达的是：

- “这个 tuple version 由 XID N 创建。”

它不表达：

- “XID N 一定会提交。”
- “XID N 已经对某个 snapshot 可见。”
- “XID N 已经不在运行。”

### 5.2 ERROR 路径进入 `AbortCurrentTransaction()`

当事务块内某条 SQL ERROR：

- PostgreSQL 通过 error recovery 进入事务 abort 处理。
- 外层会调用 `AbortCurrentTransaction()`。
- `AbortCurrentTransaction()` 循环调用 `AbortCurrentTransactionInternal()`。
- 循环存在是为了避免 subtransaction 递归过深。

核心状态分支：

- `TBLOCK_STARTED`：单语句事务，直接 `AbortTransaction()` + `CleanupTransaction()`。
- `TBLOCK_INPROGRESS`：显式事务块中，执行 `AbortTransaction()`，然后进入 `TBLOCK_ABORT`。
- `TBLOCK_ABORT`：已经 abort，后续错误只保持状态。
- `TBLOCK_ABORT_END`：用户发来 `ROLLBACK`，执行 `CleanupTransaction()`。
- `TBLOCK_SUBINPROGRESS`：只 abort 当前 subtransaction，进入 `TBLOCK_SUBABORT`。

这解释了一个常见现象：

- 显式 `BEGIN` 里某条语句 ERROR 后，事务已经被 abort。
- 但 session 还没有回到 idle。
- 后续普通 SQL 会报当前事务已失败。
- 直到用户执行 `ROLLBACK`，`CleanupTransaction()` 才收尾。

### 5.3 显式 `ROLLBACK` 路径进入同一个 abort 核心

显式 `ROLLBACK` 入口：

- `UserAbortTransactionBlock(bool chain)`。

它并不立即执行所有 abort 工作。它主要改变 `blockState`：

- live transaction block：`TBLOCK_INPROGRESS` -> `TBLOCK_ABORT_PENDING`。
- already failed block：`TBLOCK_ABORT` -> `TBLOCK_ABORT_END`。
- subtransaction：逐层标记 `TBLOCK_SUBABORT_PENDING` 或 `TBLOCK_SUBABORT_END`。

真正执行 abort 的位置：

- 后续 `CommitTransactionCommand()` 看到 `TBLOCK_ABORT_PENDING`。
- 然后调用 `AbortTransaction()`。
- 然后调用 `CleanupTransaction()`。

因此 ERROR 和 ROLLBACK 的共同核心是：

- `AbortTransaction()`。
- `RecordTransactionAbort()`。
- `ProcArrayEndTransaction()`。
- `CleanupTransaction()`。

差异只在 SQL block state：

- ERROR 后显式事务块会停留在 `TBLOCK_ABORT` 等待 `ROLLBACK`。
- 用户主动 `ROLLBACK` 通常会直接 abort 并 cleanup 回到 idle。

### 5.4 `AbortTransaction()` 的前半段：先让 cleanup 能安全运行

`AbortTransaction()` 一开始做的不是写 CLOG。它先建立可继续 cleanup 的环境：

- `HOLD_INTERRUPTS()` 阻止 cancel / die 打断 abort cleanup。
- 关闭 transaction timeout。
- `AtAbort_Memory()` 切到可用的 abort memory context。
- `AtAbort_ResourceOwner()` 确保 resource owner 状态可用。
- `LWLockReleaseAll()` 尽快释放所有 LWLock。
- `WaitLSNCleanup()` 清理 LSN 等待状态。
- `pgstat_report_wait_end()` 清掉 wait event。
- `pgstat_progress_end_command()` 清掉 progress command。
- `pgaio_error_cleanup()` 清理异步 IO error 状态。
- `UnlockBuffers()` 清理 buffer content locks。
- `XLogResetInsertion()` 重置 WAL record construction。
- `ConditionVariableCancelSleep()` 取消 condition variable sleep。
- `LockErrorCleanup()` 清掉 lock wait 状态。
- `reschedule_timeouts()` 恢复 timeout infrastructure。
- `sigprocmask(SIG_SETMASK, &UnBlockSig, NULL)` 恢复信号屏蔽。

为什么先做这些：

- ERROR 可能发生在持有 LWLock、buffer lock、condition variable wait、lock wait 的任意位置。
- abort cleanup 自己可能还要访问这些子系统。
- 如果不先释放或重置，cleanup 可能死锁或破坏 lock manager 状态。

注意边界：

- regular locks 不在这里全部释放。
- tuple / relation 等 heavyweight locks 要等到事务结果已经对外可见之后再释放。
- 这样等待者醒来时能看到一致的事务完成状态。

### 5.5 `AbortTransaction()` 的中段：进入 `TRANS_ABORT`

`AbortTransaction()` 检查低层状态：

- 正常应该是 `TRANS_INPROGRESS` 或 `TRANS_PREPARE`。
- 顶层事务要求 `s->parent == NULL`。

然后设置：

- `s->state = TRANS_ABORT`。

接下来清理事务语义相关状态：

- 恢复 user id 和 security context。
- 清掉 REINDEX 状态。
- 重置 logical streaming state。
- 重置 exported snapshot state。
- 清理未完成 parallel operation。
- `AfterTriggerEndXact(false)`。
- `AtAbort_Portals()`。
- `smgrDoPendingSyncs(false, is_parallel_worker)`。
- `AtEOXact_LargeObject(false)`。
- `AtAbort_Notify()`。
- `AtEOXact_RelationMap(false, is_parallel_worker)`。
- `AtAbort_Twophase()`。

这些调用的共同目标：

- 撤销或丢弃事务内尚未对外成为 committed 语义的副作用。
- 让后续写 abort result 前，本 backend 不再持有危险的中间状态。

但此时：

- 本事务 XID 仍然可能在 `ProcArray` 中。
- 其他 backend 仍可能认为它 running。
- tuple header 里写出的 `xmin` / `xmax` 仍然存在。

### 5.6 `RecordTransactionAbort(false)`：向事务系统宣告 abort

`AbortTransaction()` 之后调用：

- `RecordTransactionAbort(false)`。

这个函数回答：

- 本事务是否有 XID。
- 如果有，写什么 abort WAL。
- 写哪些 child XID。
- 何时设置 CLOG aborted。
- 返回哪个 latest XID 给 ProcArray。

第一条分支很关键：

- 如果 `GetCurrentTransactionIdIfAny()` 无效，直接返回 `InvalidTransactionId`。
- 没有 XID 的事务没有 tuple header 需要别人解释。
- 它可能仍有本地资源要清理，但没有必要污染 `pg_xact`。

有 XID 时：

- abort 路径会写 ABORT record。
- 它不会像 commit 那样强制 flush WAL。
- 源码注释明确说 crash 后默认假设事务 aborted。
- 因此 abort record 的持久性要求弱于 commit record。

panic guard：

- `RecordTransactionAbort()` 会检查 `TransactionIdDidCommit(xid)`。
- 如果一个事务已经 committed 却又 abort，会 `PANIC`。
- 这是防止 commit critical path 中途被错误反向解释。

写 abort record 前收集：

- pending delete relfilenode。
- committed child XIDs。
- transactional stats drops。
- replication origin 状态。
- transaction stop timestamp。

然后进入 critical section：

- `XactLogAbortRecord(...)` 写 abort WAL record。
- replication origin 需要推进 origin LSN。
- 顶层事务调用 `XLogSetAsyncXactLSN(XactLastRecEnd)` 提醒 WAL writer。
- `TransactionIdAbortTree(xid, nchildren, children)` 写 CLOG aborted。
- 离开 critical section。

最后：

- `TransactionIdLatest(xid, nchildren, children)` 得到 latest XID。
- subtransaction abort 会调用 `XidCacheRemoveRunningXids()`。
- 顶层 abort 的 running 清理留给 `ProcArrayEndTransaction()`。
- 顶层 abort 重置 `XactLastRecEnd`。

### 5.7 `TransactionIdAbortTree()`：高层事务状态接口

`TransactionIdAbortTree()` 在 `transam.c`。它是 `clog.c` 的语义包装：

- 输入顶层 XID。
- 输入 child XID 数组。
- 调用 `TransactionIdSetTreeStatus(..., TRANSACTION_STATUS_ABORTED, InvalidXLogRecPtr)`。

源码注释说明：

- abort 的非原子行为不需要像 commit 那样复杂。
- onlooker 会把所有未 committed 的事务视为 not-yet-committed。
- 对 visibility 来说，未完成和最终 aborted 都不会让插入版本可见。

这句话容易误读。它不是说 abort 随便乱写也行。

它的真实含义是：

- commit 需要让整个事务树原子地从 invisible 变成 visible。
- abort 不需要制造一个 visible 时刻。
- 如果有观察者看到某些 child 尚未标记 aborted，它仍不会把它们当 committed。
- 因此 abort 的 CLOG 更新可以少一些 commit 侧的 sub-commit 原子性负担。

### 5.8 `clog.c`：2 bit 状态如何落页

`TransactionIdSetTreeStatus()` 负责把事务树状态分布到 CLOG page。对 commit：

- 如果所有 XID 在同一页，可一次加锁设置。
- 如果跨页，先把非第一页 subxid 设为 `SUB_COMMITTED`。
- 再把 top XID 和同页 subxid 设为 `COMMITTED`。
- 最后把其它页 subxid 设为 `COMMITTED`。
- 这样并发读者看到 top-level commit 时能追到一致结果。

对 abort：

- 直接按页设置 `ABORTED`。
- 不需要先 `SUB_COMMITTED`。
- 因为 abort 不会让任何 tuple version 变成可见出生。

`TransactionIdSetPageStatus()` 的成本边界：

- 需要 CLOG SLRU bank lock。
- 同页、少量 subxid、且与 `MyProc` 中 XID 匹配时，可能使用 group update。
- 不能 group update 时，fallback 为独占获取 bank lock，单独更新。

`TransactionIdSetStatusBit()` 的不变量：

- 当前状态应从 0 或 `SUB_COMMITTED` 变到目标状态。
- recovery 中重复设置可视为 no-op。
- async commit 会记录 group LSN。
- abort 使用 `InvalidXLogRecPtr`。

`TransactionIdGetStatus()`：

- 通过 `SimpleLruReadPage_ReadOnly()` 读 CLOG page。
- 从字节和 bit shift 中取 2 bit 状态。
- 返回状态和 group LSN。
- 调用方通常不直接用它，而是用 `transam.c` 的 `TransactionLogFetch()`。

### 5.9 `ProcArrayEndTransaction()`：从 running set 中消失

`AbortTransaction()` 在 `RecordTransactionAbort()` 之后调用：

- `ProcArrayEndTransaction(MyProc, latestXid)`。

源码注释明确：

- commit / abort 必须已经报告给 WAL 和 `pg_xact`。
- 这个函数才会把事务标记为不再 running。

这是本节最重要顺序。如果反过来：

- backend 先从 `ProcArray` 消失。
- 另一个 backend 拿 snapshot 或做 instant visibility。
- 它看不到 XID running。
- 它再查 CLOG，可能还看到默认 `IN_PROGRESS`。
- 它可能把这个状态按 crash / abort 排除。
- 对 commit 路径，这会破坏刚提交事务的可见性。
- 对 abort 路径，也会制造 inconsistent observation。

当前实现中：

- `latestXid` valid 时需要 `ProcArrayLock` exclusive。
- 能立即拿锁就调用 `ProcArrayEndTransactionInternal()`。
- 拿不到时走 `ProcArrayGroupClearXid()`，由 leader 批量清。

`ProcArrayEndTransactionInternal()` 做的事情：

- 清 `ProcGlobal->xids[pgxactoff]`。
- 清 `proc->xid`。
- 清 `proc->vxid.lxid`。
- 清 `proc->xmin`。
- 清 `delayChkptFlags`。
- 清 vacuum 状态位。
- 清 subxid cache count 和 overflowed。
- 调用 `MaintainLatestCompletedXid(latestXid)`。
- 递增 `xactCompletionCount`。

对 visibility 的意义：

- 在这之前，实时 running check 应该还能把 XID 判定为 running。
- 在这之后，实时 running check 不应再把它判定为 running。
- 此时 CLOG 已经能提供 final result。

### 5.10 `CleanupTransaction()`：本地资源收尾

ERROR 后显式事务块会停在 `TBLOCK_ABORT`。用户之后发 `ROLLBACK`：

- `UserAbortTransactionBlock()` 把状态改成 `TBLOCK_ABORT_END`。
- `CommitTransactionCommand()` 调用 `CleanupTransaction()`。

`CleanupTransaction()` 要求：

- `s->state == TRANS_ABORT`。

它做：

- `AtCleanup_Portals()`。
- `AtEOXact_Snapshot(false, true)` 释放事务 snapshot。
- 删除 `TopTransactionResourceOwner`。
- `AtCleanup_Memory()` 重置事务内存。
- 清 `fullTransactionId`、`subTransactionId`、nesting、childXids。
- 清 `XactTopFullTransactionId`。
- 清 parallel current XIDs。
- 设置 `s->state = TRANS_DEFAULT`。

注意：

- 这一步是本 backend 的生命周期 cleanup。
- 可见性层面的 abort 结果已经在 `RecordTransactionAbort()` 和 `ProcArrayEndTransaction()` 完成。
- 因此其他 backend 不需要等这个 session 发 `ROLLBACK` 才知道 XID 已经不是 running。

### 5.11 reader 侧：`HeapTupleSatisfiesMVCC()` 如何解释 aborted `xmin`

普通 SELECT 使用 MVCC snapshot。核心分支简化如下：

```text
if xmin hint says invalid:
    tuple invisible
else if xmin is current transaction:
    use command id rules
else if XidInMVCCSnapshot(xmin, snapshot):
    tuple invisible because inserting xact is in-progress for this snapshot
else if TransactionIdDidCommit(xmin):
    set HEAP_XMIN_COMMITTED if safe
    continue to xmax checks
else:
    set HEAP_XMIN_INVALID if safe
    tuple invisible because inserting xact aborted or crashed
```

关键点：

- `XidInMVCCSnapshot()` 在 `TransactionIdDidCommit()` 之前。
- 这保证当前 snapshot 认为 running 的事务，即使后来已经结束，也不会影响本语句结果。
- `TransactionIdDidAbort()` 没有出现在这个核心分支。
- 代码用 “not in snapshot + not committed” 推出 “aborted or crashed”。

为什么这个推理正确：

- 如果事务仍在这个 snapshot 的 running set 中，前面的分支已经返回 invisible。
- 如果事务不在 snapshot running set 中，就应该有最终结果，或者是 crash-left default 状态。
- 只有 committed 会让 inserted tuple 继续参与可见性。
- 非 committed 都不能让 tuple 出生。

### 5.12 reader 侧：`xmax` abort 的相反含义

`xmin` abort：

- 插入失败。
- tuple 从未出生。
- 返回 invisible。

`xmax` abort：

- 删除或更新失败。
- 旧 tuple 没有被这个 `xmax` 杀死。
- 返回 visible，前提是 `xmin` 已经 visible。

`HeapTupleSatisfiesMVCC()` 中对应逻辑：

- 如果 `xmax` 在 snapshot 中，说明删除者对该 snapshot 仍未完成，旧 tuple 继续可见。
- 如果 `TransactionIdDidCommit(xmax)` 为 true，说明删除 / 更新成功，旧 tuple 不可见。
- 如果没有 commit，说明 aborted or crashed，设置 `HEAP_XMAX_INVALID` 后旧 tuple 可见。

因此同一个 aborted 结果在 `xmin` 和 `xmax` 上方向相反：

- aborted `xmin` 让版本不存在。
- aborted `xmax` 让版本继续存在。

### 5.13 backend 崩溃：没有显式 abort 也要能解释

backend 崩溃可能发生在：

- XID 已分配。
- tuple header 已写。
- WAL / buffer 状态处于中间阶段。
- 没有来得及执行 `RecordTransactionAbort()`。

重启或 recovery 后：

- 旧 backend 的 `PGPROC` 不存在。
- CLOG 中该 XID 可能仍是 all-zero `IN_PROGRESS`。
- 没有 commit record 能把它变成 committed。
- visibility 不能把它当 running 永久保留。

PostgreSQL 的规则是：

- crash 后没有成功 commit 的事务按 aborted 处理。
- CLOG `IN_PROGRESS` 不是可靠的当前运行事实。
- heap visibility 使用 “not running / not in snapshot + not committed => aborted or crashed”。

在 standby / hot standby 中：

- startup process 用 `KnownAssignedXids` 表达 primary 上已知分配但未完成的 XID。
- abort redo 调用 `TransactionIdAbortTree()`。
- 然后调用 `ExpireTreeKnownAssignedTransactionIds()` 类似 primary 上的 `ProcArrayEndTransaction()`。
- 如果 primary backend 消失且没有 abort record，KnownAssignedXids 可能暂时保留这些 XID。
- 注释说明这不影响可见性，因为 running 和 aborted 对未提交变化都不可见。
- 后续 `XLOG_RUNNING_XACTS` 会 prune，避免数组长期膨胀。


## 6. 生命周期 / ownership / cleanup

### 6.1 XID 生命周期

创建者：

- 当前 backend 在需要写事务身份时调用 `AssignTransactionId()`。

持有者：

- backend-local `TransactionStateData.fullTransactionId`。
- 顶层 `XactTopFullTransactionId`。
- shared `PGPROC` / `ProcGlobal->xids[]`。
- tuple header 中的 `xmin` / `xmax`。
- CLOG page 中的 2 bit 状态。
- WAL abort / commit record 中的事务完成信息。

释放或失效：

- backend-local XID 在 `CleanupTransaction()` 中清掉。
- shared running XID 在 `ProcArrayEndTransaction()` 中清掉。
- tuple header 中的 XID 不会在 abort 时立即清掉。
- tuple header 可能后续通过 pruning、vacuum、freeze 或 hint bit 被改写。
- CLOG 旧段会在 `TruncateCLOG()` 中按 oldest XID 边界截断。

ERROR / abort 兜底：

- `AbortTransaction()` 确保有 XID 时写 abort result。
- `ProcArrayEndTransaction()` 确保别人不再把它看成 running。
- `CleanupTransaction()` 确保本地资源和内存释放。

### 6.2 tuple version 生命周期

创建：

- heap insert 在 page 上写 tuple。
- tuple header 写入 `xmin`。
- 此时 tuple 可能还只对当前事务可见。

正常 commit：

- commit result 进入 `pg_xact`。
- running XID 从 `ProcArray` 清除。
- 后续 snapshot 可按 CLOG committed 解释 `xmin`。
- hint bit 可设置 `HEAP_XMIN_COMMITTED`。

abort：

- abort result 进入 `pg_xact`，或 crash 后没有 committed 结果。
- running XID 从 `ProcArray` 或 KnownAssignedXids 消失。
- 后续 reader 发现 `xmin` not in snapshot and not committed。
- reader 可设置 `HEAP_XMIN_INVALID`。
- tuple 物理空间等待 pruning / vacuum 回收。

ownership 边界：

- 插入事务拥有事务结果。
- heap page 不拥有事务结果。
- reader 可以缓存 hint bit，但不能创造事务结果。
- VACUUM 可以回收不可见物理版本，但必须尊重 oldest snapshot / horizon。

### 6.3 `ResourceOwner` 与 `MemoryContext`

`MemoryContext` 管：

- 事务生命周期内存。
- abort cleanup 需要切换到 `TransactionAbortContext` 或 `TopMemoryContext`。
- `CleanupTransaction()` 最终通过 `AtCleanup_Memory()` 重置。

`ResourceOwner` 管：

- buffer pin。
- catcache / relcache refcount。
- tuple descriptors。
- snapshots。
- files。
- 其他需要显式 release 的外部资源。

二者不能混用：

- 内存 reset 不会释放 heavyweight lock。
- ResourceOwner release 不会自动让 CLOG 状态变 aborted。
- CLOG 状态改变不意味着本地 memory context 已经清理。

abort 路径先做：

- 确保 abort memory 可用。
- 确保 resource owner 可用。
- 释放容易阻塞 cleanup 的 LWLock / buffer lock。

abort 路径后做：

- `ResourceOwnerRelease(... BEFORE_LOCKS ...)`。
- 事务级模块 cleanup。
- `ResourceOwnerRelease(... LOCKS ...)`。
- `ResourceOwnerRelease(... AFTER_LOCKS ...)`。
- `smgrDoPendingDeletes(false)`。
- 本地 GUC、SPI、namespace、files、stats cleanup。

### 6.4 lock ownership

abort 中有两个层次：

- LWLock / buffer content lock 等短期内存并发锁。
- heavyweight lock / tuple lock / transaction lock 等事务语义锁。

短期锁：

- ERROR 后尽快释放。
- 防止 cleanup 自己重新进入子系统时死锁。

事务语义锁：

- 不能过早释放。
- 等待者醒来时必须能看到事务已经完成。
- 因此要在 `RecordTransactionAbort()` 和 `ProcArrayEndTransaction()` 之后释放。

这解释了 abort 顺序：

- 先清危险的低层等待和短锁。
- 写事务结果。
- 从 running set 消失。
- 再释放事务可见资源和锁。

### 6.5 hint bit ownership

hint bit 写入者：

- 任意访问 tuple 的 backend。
- 但必须持有 buffer pin 和至少 shared buffer content lock。
- 设置 hint bit 前需要满足 page checksum / IO interlock 约束。

hint bit 语义：

- `HEAP_XMIN_INVALID` 可以缓存 aborted / crashed inserted tuple。
- `HEAP_XMAX_INVALID` 可以缓存 aborted / crashed delete / update。
- committed hint bit 还要考虑 commit LSN 是否已安全 flush。

hint bit 不拥有：

- 事务状态真相。
- snapshot 语义。
- tuple 可回收权。

hint bit 的正确理解：

- 它是 visibility 判定的缓存。
- 它不是 `pg_xact` 的替代品。
- 它可以缺失。
- 它可以由后来的 reader 设置。


## 7. 正确性机制层次

### 7.1 running 判定层

机制：

- `ProcArray`。
- MVCC snapshot 的 `xip` / `subxip`。
- recovery 中的 `KnownAssignedXids`。

保证：

- 判断一个 XID 对某个实时检查或某个 snapshot 是否仍未完成。

不保证：

- 不保证事务最终会 commit。
- 不保证 tuple 物理上可回收。
- 不保证 CLOG 已经被 hint bit 缓存。

### 7.2 final result 层

机制：

- `pg_xact` / CLOG。
- `TransactionIdDidCommit()`。
- `TransactionIdDidAbort()`。

保证：

- committed 事务最终结果可查。
- 显式 aborted 事务可查。
- subcommitted 事务可追 parent。

不保证：

- CLOG `IN_PROGRESS` 不保证当前仍在运行。
- `TransactionIdDidAbort()` 不覆盖 crash-aborted 的所有情况。
- CLOG 状态不表达 snapshot 时间点。

### 7.3 snapshot 层

机制：

- `SnapshotData`。
- `XidInMVCCSnapshot()`。

保证：

- 当前语句或事务应该把哪些 XID 当作仍未完成。
- 在 snapshot 生命周期内保持可见性稳定。

不保证：

- 不保证反映当前最新 commit / abort。
- 不保证降低 CLOG lookup 成本。
- 不负责资源回收。

### 7.4 WAL / crash safety 层

commit 路径：

- 必须满足 WAL-before-CLOG committed。
- committed 一旦暴露，crash recovery 必须能重放。

abort 路径：

- 会写 abort record。
- 默认不强制 flush。
- crash 后没有 commit 的事务可按 aborted。
- 因此 abort CLOG 写入可早于 abort WAL 持久化。

不要误解：

- “abort 不 flush” 不等于“不写 abort WAL”。
- abort WAL 对 standby、replication、pending delete、lock release 等仍有价值。
- 只是前台不必像 commit 一样等待持久化。

### 7.5 lock / ordering 层

关键 ordering：

- 写 CLOG final status。
- 再清 ProcArray running status。
- 再释放事务 locks 和资源。

正确性来源：

- 读取者如果看到 running，就按 running 处理。
- 读取者如果看不到 running，就能在 CLOG 或 crash-default 规则中得到 final interpretation。
- 等待者被 lock 唤醒时，事务完成状态已经可见。

### 7.6 hint bit 层

保证：

- 减少后续 CLOG lookup。
- 把已经判定的事务结果缓存到 tuple page。

不保证：

- 不负责事务结果持久性。
- 不负责 snapshot consistency。
- 不负责并发互斥。


## 8. 错误路径 / 异常路径 / fallback

### 8.1 ERROR in transaction block

路径：

- SQL 执行中 ERROR。
- `AbortCurrentTransaction()`。
- `AbortTransaction()`。
- `RecordTransactionAbort(false)`。
- `TransactionIdAbortTree()`。
- `ProcArrayEndTransaction()`。
- `blockState = TBLOCK_ABORT`。
- 等待用户 `ROLLBACK`。

可见性结果：

- 其他 backend 不需要等用户 `ROLLBACK`。
- XID 已经不在 running set。
- CLOG 已经可显示 aborted。
- 物理 tuple 可能仍在 heap page。
- 普通 SELECT 看不到 aborted insert。

### 8.2 显式 `ROLLBACK`

路径：

- `UserAbortTransactionBlock()` 只设置 block state。
- `CommitTransactionCommand()` 看到 `TBLOCK_ABORT_PENDING`。
- 执行 `AbortTransaction()`。
- 执行 `CleanupTransaction()`。
- 回到 `TBLOCK_DEFAULT`。

差异：

- 不会停在 failed transaction block 等用户补发 rollback。
- 但核心 abort 语义相同。

### 8.3 backend crash before abort record

路径：

- XID 已写入 tuple。
- backend 崩溃。
- 没有 `RecordTransactionAbort()`。
- 没有显式 CLOG aborted 位。
- 重启或 recovery 后 running set 中没有该 backend。

visibility fallback：

- 不在 running set。
- `TransactionIdDidCommit()` false。
- heap visibility 解释为 aborted or crashed。
- inserted tuple invisible。
- deleted tuple still visible。

为什么不依赖 `TransactionIdDidAbort()`：

- CLOG 可能仍是 `IN_PROGRESS`。
- `TransactionIdDidAbort()` 对这种 crash-aborted XID 返回 false。
- 因此它不能作为 heap visibility 的通用失败判定。

### 8.4 subtransaction abort

路径：

- `AbortSubTransaction()`。
- `RecordTransactionAbort(true)`。
- `TransactionIdAbortTree()`。
- `XidCacheRemoveRunningXids()`。
- `CleanupSubTransaction()`。

本节只取一个边界：

- 已经 aborted 的 subxid 需要尽快从 PGPROC subxid cache 中移除。
- 否则 `XactLockTableWait` 和 running 判定可能继续把失败 subxact 当作需要等待。
- 但 subtransaction committed 的 parent 追溯问题留到下一节。

### 8.5 CLOG group update fallback

`TransactionIdSetPageStatus()` 先尝试：

- 同页。
- XID 与 `MyProc` 匹配。
- subxid 数不超过阈值。
- subxid cache 完整匹配。
- conditional acquire SLRU bank lock。

如果竞争存在：

- 尝试 `TransactionGroupUpdateXidStatus()`。

如果 group update 不适用或失败：

- 获取 SLRU bank lock。
- 单独调用 `TransactionIdSetPageStatusInternal()`。

正确性：

- group update 是性能优化。
- fallback 不改变事务语义。
- 失败时不是跳过 CLOG，而是退回独占锁更新。

### 8.6 ProcArray group clear fallback

`ProcArrayEndTransaction()` 先尝试：

- `LWLockConditionalAcquire(ProcArrayLock, LW_EXCLUSIVE)`。

如果拿不到：

- `ProcArrayGroupClearXid()`。
- 第一个 backend 成为 leader。
- leader 持有 `ProcArrayLock` 后为一组 backend 批量清 XID。
- followers 等待 `WAIT_EVENT_PROCARRAY_GROUP_UPDATE`。

正确性：

- 批量清理仍在 `ProcArrayLock` exclusive 下执行。
- 每个成员仍调用 `ProcArrayEndTransactionInternal()`。
- 顺序边界仍是 CLOG 已写后再清 ProcArray。

### 8.7 parallel worker abort

`AbortTransaction()` 对 parallel worker 特判：

- parallel worker 不写顶层 abort record。
- 用户 backend / leader 负责事务完成记录。
- worker 会设置 async LSN 提醒 WAL writer。

边界：

- parallel worker 可以参与执行和写 WAL。
- 但事务结果 ownership 在 leader。
- 不能让多个 worker 各自把同一个顶层 XID 标成 final status。

### 8.8 recovery / hot standby

recovery 中没有普通 backend 的 live `PGPROC` running set。替代机制：

- `KnownAssignedXids`。
- `RecordKnownAssignedTransactionIds()`。
- `ExpireTreeKnownAssignedTransactionIds()`。

abort redo 顺序：

- `xact_redo_abort()`。
- `TransactionIdAbortTree()`。
- `ExpireTreeKnownAssignedTransactionIds()`。
- release standby locks。

边界：

- standby 上 running 的含义是 primary WAL 流中已知分配但尚未看到完成记录。
- 已完成后从 KnownAssignedXids 移除。
- 如果缺少 abort record，后续 running-xacts 信息可 prune。


## 9. 成本、资源与跨模块传播

### 9.1 visibility hot path 成本

每个 tuple visibility check 可能触发：

- hint bit 检查。
- current transaction check。
- snapshot range check。
- `xip` / `subxip` search。
- CLOG lookup。
- hint bit 设置。

成本随这些变量增长：

- 扫描 tuple 数。
- snapshot 中 running XID 数。
- subxid 数。
- subxid overflow 是否发生。
- CLOG page 是否在 SLRU cache。
- hint bit 是否已经设置。

本节重点：

- abort tuple 第一次被新 snapshot 访问时，可能需要 CLOG lookup。
- 设置 `HEAP_XMIN_INVALID` 后，后续访问通常更便宜。
- 如果 page 不能安全写 hint bit，后续可能反复查。

### 9.2 ProcArray 成本

`TransactionIdIsInProgress()` 可能：

- 用 `RecentXmin` shortcut 返回 false。
- 用 cached completed XID 返回 false。
- 用 current transaction check 返回 true。
- 获取 `ProcArrayLock` shared。
- 扫 `ProcGlobal->xids[]`。
- 扫 PGPROC subxid cache。
- recovery 中查 KnownAssignedXids。
- subxid overflow 时走 `pg_subtrans` slow path。

成本随这些变量增长：

- backend 数。
- running transaction 数。
- cached subxid 总量。
- subxid overflow 频率。
- recovery KnownAssignedXids 数。

为什么 MVCC 使用 snapshot：

- 普通 SELECT 不应该每个 tuple 都实时扫 ProcArray。
- snapshot 把 running set 复制到本地结构。
- `HeapTupleSatisfiesMVCC()` 用 `XidInMVCCSnapshot()`，降低高频共享锁压力。

### 9.3 CLOG / SLRU 成本

CLOG 成本来源：

- SLRU page lookup。
- bank lock。
- page miss 后 IO。
- group LSN 维护。
- commit / abort status bit 写入。

成本随这些变量增长：

- 事务结束速率。
- XID 分布跨 CLOG page 的程度。
- subtransaction 数。
- CLOG SLRU cache 命中率。
- 同页 status update 竞争。

abort 特点：

- 不等待 abort WAL flush。
- 仍需要 CLOG page update。
- 仍可能争用 CLOG bank lock。
- 大量 abort workload 会制造 SLRU 和 ProcArray 清理压力。

### 9.4 WAL 与 replication 传播

abort WAL 的资源传播：

- 记录 abort 时间。
- 记录 subxact。
- 记录 pending delete relfilocator。
- 记录 dropped stats。
- 记录 replication origin。
- 在 standby 上驱动 CLOG aborted 和 KnownAssignedXids expire。

不强制 flush 的影响：

- 前台 abort latency 通常低于 sync commit。
- WAL writer 后台推进 abort WAL。
- streaming replication 中大量 abort 仍可能形成 WAL backlog。
- 源码中 `XLogSetAsyncXactLSN()` 明确用于提醒 WAL writer。

### 9.5 vacuum / cleanup horizon 传播

abort inserted tuple：

- 对 SQL 不可见。
- 但物理空间还在 heap page。
- VACUUM / pruning 需要在安全 horizon 后回收。

相关传播：

- long-running snapshot 会延迟某些 cleanup。
- `xmin` horizon 影响可移除边界。
- aborted insert 通常比 committed-delete 的版本更容易被判 dead，但仍要遵守 page / index / HOT 结构约束。

本节不深入 VACUUM：

- 这里只强调 visibility 判定不等于物理回收。
- dead tuple 何时可回收是后续 VACUUM 课程的问题。

### 9.6 跨模块边界

事务模块：

- 负责 XID 分配、commit / abort、CLOG 状态、事务生命周期。

ProcArray：

- 负责 running set、snapshot 构建、completion count、horizon 输入。

Heap access method：

- 负责 tuple header 解释、hint bit、visibility routine。

WAL / recovery：

- 负责 crash 后重放 commit / abort 完成记录。
- 在 standby 上维护 KnownAssignedXids。

ResourceOwner / MemoryContext：

- 负责本 backend 的 cleanup。
- 不负责事务结果语义。

Lock manager：

- 负责等待和唤醒。
- 等待者必须在事务结果已可见后再继续。


## 10. 观测与诊断入口

### 10.1 能直接观测的状态

SQL 层：

- 普通 SELECT 是否能看到 tuple。
- ERROR 后 session 是否处于 failed transaction block。
- `ROLLBACK` 后 session 是否回到 idle。

系统视图：

- `pg_stat_activity.state` 可看到 `idle in transaction (aborted)`。
- `pg_locks` 可看到 transactionid lock 等等待关系。
- wait event 可看到 `ProcArrayGroupUpdate` 相关等待。

扩展：

- `pageinspect` 的 `heap_page_items()` 可看到物理 tuple header 字段。
- 可以看到 `t_xmin`、`t_xmax`、`t_infomask`。

源码调试：

- gdb 断点 `AbortTransaction`。
- gdb 断点 `RecordTransactionAbort`。
- gdb 断点 `TransactionIdAbortTree`。
- gdb 断点 `ProcArrayEndTransaction`。
- gdb 断点 `HeapTupleSatisfiesMVCC`。

### 10.2 只能推断的状态

事务是否因为 crash 被隐式 aborted：

- 不能简单从 `TransactionIdDidAbort()` 直接确认。
- 可以从 “不在 running set + not committed + crash/recovery 语境” 推断。

hint bit 为什么没设置：

- 可能 page 没有安全设置条件。
- 可能尚未被新 snapshot 访问。
- 可能 buffer content lock 模式不允许。

某次 SELECT 为什么没查 CLOG：

- 可能 hint bit 已经足够。
- 可能 snapshot 直接判定 in-progress。
- 可能 current transaction branch 处理掉。

### 10.3 基本不可见的状态

`TransactionStateData`：

- backend-local。
- SQL 不能直接读。
- 需要 debugger 或日志 instrumentation。

CLOG 原始 2 bit：

- 没有普通 SQL 接口直接读。
- 可以通过源码断点、扩展、page 文件分析或自定义 instrumentation 观察。

`ProcGlobal->xids[]`：

- shared memory 内部结构。
- 普通 SQL 不能直接枚举。
- snapshot 和系统视图只提供间接视角。

### 10.4 一个可验证 runtime 现象

现象：

- session A 在事务中 insert 后 ERROR。
- session B 普通 SELECT 看不到这行。
- pageinspect 仍可能看到物理 tuple 的 `t_xmin`。
- session A 在 `ROLLBACK` 前处于 failed transaction block。

源码解释：

- ERROR 已经触发 `AbortTransaction()`。
- `RecordTransactionAbort()` 已经写 aborted 状态。
- `ProcArrayEndTransaction()` 已经从 running set 移除 XID。
- tuple 物理版本没有立即被移除。
- `HeapTupleSatisfiesMVCC()` 看到 `xmin` not in snapshot 且 not committed，于是 invisible。

诊断边界：

- 普通 SELECT 只能证明 SQL 可见性。
- pageinspect 只能证明物理 tuple 还在页上。
- 不能仅凭 `pg_stat_activity` 判断 CLOG 2 bit。
- 不能仅凭 hint bit 是否已设置判断 abort 是否发生。


## 11. 常见误区

误区 1：

- “CLOG 里 `IN_PROGRESS` 就表示事务还在运行。”

纠正：

- CLOG all-zero 初始状态就是 `IN_PROGRESS`。
- crash-aborted 事务可能仍是这个值。
- 当前运行事实要看 `ProcArray` / snapshot / KnownAssignedXids。

误区 2：

- “看见 `xmin` 有值就说明 tuple 曾经成功插入。”

纠正：

- `xmin` 表示哪个 XID 尝试创建这个版本。
- 只有 `xmin` committed 且对 snapshot 可见，tuple 才算出生。

误区 3：

- “heap visibility 应该用 `TransactionIdDidAbort()` 判断失败。”

纠正：

- `TransactionIdDidAbort()` 只覆盖显式 aborted。
- crash-aborted 可能没有 CLOG aborted 位。
- visibility 使用 not-running / not-in-snapshot 加 not-committed 的排除法。

误区 4：

- “ERROR 后事务还没 ROLLBACK，所以别人应该还把它看成 running。”

纠正：

- 显式事务块 ERROR 后已经执行 `AbortTransaction()`。
- 它可能停在 `TBLOCK_ABORT` 等用户 cleanup。
- 但 XID 已经可从 running set 清除。

误区 5：

- “hint bit 是事务真相。”

纠正：

- hint bit 是缓存。
- 真相来自事务状态、snapshot 和 CLOG 的组合。
- hint bit 可滞后，也可缺失。

误区 6：

- “abort 不 flush WAL，所以 standby 不关心 abort record。”

纠正：

- abort record 对 standby CLOG、KnownAssignedXids、锁释放、pending delete 仍重要。
- 只是前台 crash safety 不需要像 commit 一样等待持久化。

误区 7：

- “tuple 不可见就表示物理空间已经回收。”

纠正：

- visibility 和 physical cleanup 是两个边界。
- aborted insert 可以不可见但仍占 heap page 空间。
- pruning / VACUUM 决定何时回收。


## 12. 课堂实验

### 实验 1：ERROR 后 tuple 物理存在但 SQL 不可见

目标：

- 观察 abort 后 SQL visibility 和 heap physical tuple 的差异。

准备：

- 需要 superuser 或有权限安装 `pageinspect`。

Session A：

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
DROP TABLE IF EXISTS abort_vis_demo;
CREATE TABLE abort_vis_demo(id int primary key, note text);
BEGIN;
INSERT INTO abort_vis_demo VALUES (1, 'will abort');
SELECT txid_current() AS xid_before_error;
SELECT 1 / 0;
```

Session A 此时：

- 事务已经 failed。
- 后续普通 SQL 会报当前事务已失败。
- 还没有执行 `ROLLBACK` cleanup。

Session B：

```sql
SELECT * FROM abort_vis_demo;
SELECT lp, t_xmin, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('abort_vis_demo', 0));
```

观察：

- 普通 SELECT 不返回 `id = 1`。
- pageinspect 可能看到物理 tuple。
- `t_xmin` 对应 Session A 的 XID。
- `t_infomask` 是否已有 invalid hint bit 取决于访问和 hint bit 设置时机。

回到源码解释：

- ERROR 触发 `AbortTransaction()`。
- `RecordTransactionAbort()` 写 abort 状态。
- `ProcArrayEndTransaction()` 清 running XID。
- `HeapTupleSatisfiesMVCC()` 对 `xmin` 走 not-in-snapshot + not-committed 分支。

收尾：

```sql
-- Session A
ROLLBACK;
-- Session B
VACUUM abort_vis_demo;
SELECT lp, t_xmin, t_xmax, t_infomask, t_infomask2
FROM heap_page_items(get_raw_page('abort_vis_demo', 0));
```

注意：

- VACUUM 后物理 tuple 是否还在同一页，取决于 pruning、page layout 和版本行为。
- 本实验的核心不是强制证明空间立刻回收，而是证明可见性和物理存在分离。

### 实验 2：断点验证 abort 顺序

目标：

- 验证 “CLOG result before ProcArray clear”。

建议断点：

```gdb
break AbortCurrentTransaction
break AbortTransaction
break RecordTransactionAbort
break TransactionIdAbortTree
break ProcArrayEndTransaction
break ProcArrayEndTransactionInternal
break HeapTupleSatisfiesMVCC
```

步骤：

- 启动一个 debug build PostgreSQL。
- Session A 执行 `BEGIN; INSERT ...; SELECT 1/0;`。
- 在 gdb 中观察断点顺序。
- 记录 `xid`、`latestXid`、`MyProc->xid`。
- 在 `RecordTransactionAbort()` 后检查 XID 仍在 `MyProc->xid`。
- 在 `ProcArrayEndTransactionInternal()` 后检查 `proc->xid` 被清空。

预期：

- `TransactionIdAbortTree()` 发生在 `ProcArrayEndTransaction()` 前。
- `ProcArrayEndTransactionInternal()` 清掉 `ProcGlobal->xids[]` 和 `proc->xid`。
- 事务 locks 的释放发生在 post-abort cleanup 阶段。

讨论：

- 如果把清 ProcArray 放到写 CLOG 之前，会破坏什么？
- 为什么 commit 和 abort 都遵守同一个 ProcArray end 边界？

### 实验 3：对比 running、aborted、committed 的 reader 分支

目标：

- 用两个 session 观察同一个 tuple XID 在不同时间点的解释。

Session A：

```sql
DROP TABLE IF EXISTS abort_vis_cmp;
CREATE TABLE abort_vis_cmp(id int, note text);
BEGIN;
INSERT INTO abort_vis_cmp VALUES (1, 'open xact');
SELECT txid_current() AS writer_xid;
```

Session B：

```sql
SELECT * FROM abort_vis_cmp;
SELECT lp, t_xmin, t_infomask
FROM heap_page_items(get_raw_page('abort_vis_cmp', 0));
```

观察点 1：

- Session A 未结束。
- Session B 普通 SELECT 看不到 row。
- 原因是 snapshot / running 分支，不是 final abort。

Session A：

```sql
ROLLBACK;
```

Session B：

```sql
SELECT * FROM abort_vis_cmp;
SELECT lp, t_xmin, t_infomask
FROM heap_page_items(get_raw_page('abort_vis_cmp', 0));
```

观察点 2：

- 普通 SELECT 仍看不到 row。
- 原因变成 final not committed。
- hint bit 可能已经或尚未反映 invalid。

变体：

- 把 Session A 的 `ROLLBACK` 换成 `COMMIT`。
- Session B 新语句应该能看到 row。
- 这能对比同一个 `xmin` 在 CLOG committed 与 aborted 下的分支差异。


## 13. 讨论题

1. 为什么 `pg_xact` 的 `TRANSACTION_STATUS_IN_PROGRESS` 不能单独作为“事务正在运行”的事实来源？
2. `AbortTransaction()` 为什么要先释放 LWLock 和 buffer content lock，却不能过早释放事务级 locks？
3. 如果 `ProcArrayEndTransaction()` 在 `TransactionIdAbortTree()` 之前执行，一个并发 reader 可能看到哪些不一致状态？
4. 为什么 `HeapTupleSatisfiesMVCC()` 在 `xmin` 分支中使用 `XidInMVCCSnapshot()` 后再调用 `TransactionIdDidCommit()`，而不是直接查 CLOG？
5. `xmin` aborted 和 `xmax` aborted 对同一个 tuple version 的可见性方向为什么相反？
6. ERROR 后 session 还处于 failed transaction block，为什么其他 backend 仍然可以把它的 XID 视为已经完成？
7. crash-aborted 事务没有显式 aborted CLOG 位时，visibility 如何保持正确？这个推理依赖哪些前提？
8. hint bit 缺失时，哪些现象会变慢？哪些语义不会改变？


## 14. 本节小结

本节唯一主问题是：

- 已写入 tuple header 的 XID 在 abort、ERROR、ROLLBACK 或 crash 后，如何被 visibility 判定为失败而不是仍未完成？

核心链路：

- XID 先写入 tuple header。
- ERROR 或 ROLLBACK 进入 `AbortCurrentTransaction()`。
- `AbortTransaction()` 先清理危险等待和短锁。
- `RecordTransactionAbort()` 写 abort WAL 并调用 `TransactionIdAbortTree()`。
- `TransactionIdAbortTree()` 写 `pg_xact` aborted。
- `ProcArrayEndTransaction()` 再从 running set 清掉 XID。
- `CleanupTransaction()` 收尾本地资源和内存。
- `HeapTupleSatisfiesMVCC()` 用 snapshot running 判定加 CLOG committed 判定解释 tuple。

核心状态和边界：

- `TransactionStateData` 是 backend-local 控制面。
- `ProcArray` / snapshot 是 running 边界。
- `pg_xact` 是最终事务结果边界。
- tuple header 只保存 XID 和 hint bits。
- hint bit 是缓存，不是语义源头。

正确性关键：

- 先 final result，后离开 running set。
- 先 snapshot / running 判定，后 CLOG final result 判定。
- 对 insert：not committed 意味着 tuple 不可见。
- 对 delete / update：not committed 意味着旧 tuple 仍可见。
- crash 后没有 committed 的事务按 aborted 解释。

错误路径收尾：

- ERROR in transaction block 会执行 abort，但可能等待用户 `ROLLBACK` 做 cleanup。
- 显式 `ROLLBACK` 和 ERROR 共享同一个 abort 核心。
- backend crash 可能没有显式 abort CLOG 位，因此不能依赖 `TransactionIdDidAbort()` 覆盖所有失败。
- subtransaction abort 会立即移除 failed subxid 的 running cache。

观测边界：

- SQL SELECT 能看到可见性结果。
- `pageinspect` 能看到物理 tuple 和 header。
- `pg_stat_activity` 能看到 failed transaction block。
- `pg_locks` / wait event 能看到等待和 group update 迹象。
- 原始 CLOG 2 bit、`TransactionStateData`、`ProcGlobal->xids[]` 多数需要 debugger 或 instrumentation。

可迁移系统规律：

- 高频读路径中，“未完成”和“最终失败”常常产生同一个用户可见结果，但它们必须由不同状态层表达。
- 正确系统不会把 raw field 当语义。
- 语义来自 field、lifecycle state、shared running set、durable result、snapshot boundary 和 cleanup ordering 的组合。

仍需标注的推断边界：

- hint bit 是否出现依赖访问时机、buffer 状态和版本细节。
- pageinspect 看到的物理布局依赖 pruning、VACUUM 和 heap page 状态。
- ProcArray / CLOG contention 的成本依赖 backend 数、事务结束速率、subxid 数和硬件 cache 行为。
- recovery 中 KnownAssignedXids 的观察依赖 WAL 流、standby 状态和 running-xacts 记录。
