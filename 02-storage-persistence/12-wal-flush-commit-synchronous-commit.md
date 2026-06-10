# PostgreSQL Commit record、WAL flush 与 synchronous_commit
## 课程定位
本节主题：一次普通事务提交时，PostgreSQL 如何写入 commit record，如何把 WAL flush 到稳定存储，以及 `synchronous_commit` 怎样改变提交路径上的等待。
前置知识：已理解 WAL insertion、record end LSN、page LSN 和 WAL-before-data。
本节唯一主问题：COMMIT 返回客户端之前，PostgreSQL 到底要等待本地 WAL、事务状态和同步备库推进到哪个边界？
本节核心矛盾：提交路径要尽快返回并复用 group commit；但一旦向客户端承诺 durability，就必须能在 crash recovery 和同步复制中解释这个承诺。
本节主流程：`CommitTransaction()` -> `RecordTransactionCommit()` -> `XactLogCommitRecord()` -> `XLogFlush()` 或 async commit -> `TransactionIdCommitTree()` -> 可选 `SyncRepWaitForLSN()`。
错误路径 / 异常路径包括本地 WAL write/fsync 失败、同步复制等待期间中断、standby 消失、postmaster death，以及 async commit 在 crash 窗口内丢失事务承诺。
观测与诊断入口是 `pg_stat_wal`、`pg_stat_activity` 的 SyncRep/WAL wait event、`pg_stat_replication` 的 write/flush/replay LSN、commit latency 分布，以及 `XLogFlush()`/`SyncRepWaitForLSN()` 断点。
本节只看提交路径。
不展开 heap insert、buffer eviction、checkpoint 全流程。
但会反复回到一条核心规则：
数据页或事务状态页可以落盘之前，相关 WAL 必须已经持久化到足够的 LSN。
读完本节，你应该能回答：
- `RecordTransactionCommit()` 为什么是普通事务提交的耐久化中心。
- commit record 是怎样由 `XactLogCommitRecord()` 组装并通过 `XLogInsert()` 插入 WAL 的。
- `XLogInsert()` 返回的 LSN 为什么是后续 flush 和 page LSN 的边界。
- `XLogFlush()` 等待的到底是 WAL buffer 写入完成、系统调用 write 完成，还是 fsync 完成。
- `synchronous_commit = off/local/on/remote_write/remote_apply` 分别等待到哪里。
- 没有同步备库时，`synchronous_commit = on` 为什么主要退化成本地 flush。
- group commit 在 `XLogFlush()` 里如何自然发生。
- `commit_delay` 和 `commit_siblings` 是怎样给更多 backend 加入同一次 fsync 的机会。
- `SyncRepWaitForLSN()` 为什么发生在本地 commit 已经完成之后。
- walwriter 能减少后台 WAL 写入和 async commit 窗口，但为什么不能替代同步 commit 的本地 flush。
- 提交路径上的错误、崩溃、取消和 postmaster death 各自意味着什么。
## 源码基线
源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`
本节重点阅读：
- `src/backend/access/transam/xact.c`
- `src/backend/access/transam/xlog.c`
- `src/backend/access/transam/xlogwait.c`
- `src/backend/replication/syncrep.c`
- `src/backend/postmaster/walwriter.c`
辅助核对：
- `src/backend/access/transam/xloginsert.c`
- `src/include/access/xact.h`
- `src/include/access/xlog.h`
- `src/include/access/xlogwait.h`
- `src/include/replication/syncrep.h`
- `src/include/replication/walsender_private.h`
- `src/backend/utils/misc/guc_tables.c`
- `src/backend/utils/misc/guc_parameters.dat`
- `src/backend/access/transam/README`
- `src/backend/replication/walsender.c`
---
## 1. 先给结论：提交正确性顺序
一次真正分配了 XID、写过 WAL 的普通事务，在提交时会进入 `RecordTransactionCommit()`。 这个函数做三件和耐久性直接相关的事。
第一，构造并插入 commit WAL record。 第二，根据 `synchronous_commit`、`forceSyncCommit` 和待删除 relation 等条件，决定同步 `XLogFlush()`，还是把 commit LSN 交给 async commit 机制。
第三，在本地事务状态已经标记为 committed 之后，如果配置了同步复制等待，再进入 `SyncRepWaitForLSN()` 等待同步备库。 核心顺序是：
```text
CommitTransaction()
  -> RecordTransactionCommit()
       -> XactLogCommitRecord()
            -> XLogBeginInsert()
            -> XLogRegisterData(...)
            -> XLogInsert(RM_XACT_ID, XLOG_XACT_COMMIT...)
       -> XLogFlush(XactLastRecEnd) 或 XLogSetAsyncXactLSN(...)
       -> TransactionIdCommitTree(...) 或 TransactionIdAsyncCommitTree(...)
       -> SyncRepWaitForLSN(XactLastRecEnd, true)
```
这里最容易误解的是“commit record 插入 WAL”和“commit record 已经持久化”不是一回事。 `XLogInsert()` 把 WAL record 拷贝进共享 WAL buffer，并返回 record end LSN。
它不会因为这是 commit record 就自动 fsync。 真正要求 WAL 到达磁盘 flush 边界的是 `XLogFlush(record_end_lsn)`。
`synchronous_commit = off` 允许 backend 不等待这个 flush。 它会记录最新 async commit LSN，随后由 walwriter 或其他 backend 的后续 flush 把它带到磁盘。
这就是异步提交的风险窗口。 如果 PostgreSQL 或 OS 在 commit 返回客户端之后、WAL 真正持久化之前崩溃，这个事务可能在恢复后消失。
`synchronous_commit = local`、`on`、`remote_write`、`remote_apply` 都会先等待本地 `XLogFlush()`。 区别只在是否还要等待同步备库，以及等待备库到 write、flush 还是 apply。
本基线里 `on` 等同于枚举 `SYNCHRONOUS_COMMIT_REMOTE_FLUSH`。 但同步复制等待还要求 `max_wal_senders > 0` 且 `synchronous_commit > local`，并且确实定义了同步备库。
所以没有同步备库配置时，`on` 的实际效果就是本地 flush。 `XLogFlush()` 内部天然支持 group commit。
多个 backend 同时要 commit 时，不一定每个 backend 都自己 fsync。 一个 backend 拿到 `WALWriteLock` 后，会尽量把已经插入的更多 WAL 一起写出并 flush。
其他 backend 如果拿不到锁，会等待锁释放，然后重新检查自己的 commit LSN 是否已经被别人 flush 到。 `commit_delay` 只是这个机制上的可选人为延迟。
当 `CommitDelay > 0`、`fsync` 打开、且并发活跃事务达到 `commit_siblings` 门槛时，拿到 `WALWriteLock` 的 backend 会短暂睡眠，让更多提交赶上同一次 fsync。 walwriter 负责周期性或被唤醒后做 `XLogBackgroundFlush()`。
它可以写出完整 WAL page，处理 async commit LSN，并按 `wal_writer_delay` / `wal_writer_flush_after` 控制 flush 节奏。 它不能替代同步 commit 的 `XLogFlush()`。
同步 commit 的客户端确认必须由当前提交路径确认自己的 commit LSN 已经满足 durability 条件。
---
## 2. 本节的几个 LSN
`XLogRecPtr` 是 WAL 位置。 它是 64 bit 指针。
`InvalidXLogRecPtr` 是 0。 WAL record 的返回值通常是 record end LSN，也就是“下一个 record 的起点”。
`XLogInsertRecord()` 的注释明确说，这个返回值可以作为被该 WAL action 影响的数据页的 LSN。 更准确地说：
如果一个数据页记录了某个 LSN，那么写出这个数据页之前，WAL 必须至少 flush 到这个 LSN。 本节中最常出现的变量是 `XactLastRecEnd`。
`XLogInsertRecord()` 在成功插入 WAL 后会设置：
```text
ProcLastRecPtr = StartPos
XactLastRecEnd = EndPos
```
所以事务提交时，`XactLastRecEnd` 指向最近一次 WAL record 的 end LSN。 对普通写事务来说，commit record 插入后，`XactLastRecEnd` 就是 commit record 的 end LSN。
`RecordTransactionCommit()` 随后用它调用 `XLogFlush()` 或 `XLogSetAsyncXactLSN()`。 还有两个容易混淆的位置：
`LogwrtResult.Write` 表示 WAL 已经写到内核或文件的哪一个位置。 `LogwrtResult.Flush` 表示 WAL 已经 fsync 或等效持久化到哪一个位置。
提交耐久性关心的是 flush，不只是 write。 `remote_write` 里的 write 是“备库已经写入操作系统”的含义。
它不是主库本地 `LogwrtResult.Write` 的等待级别。 主库在进入同步复制等待之前，已经完成本地 `XLogFlush()`。
---
## 3. 从 CommitTransaction 进入 RecordTransactionCommit
先读 `src/backend/access/transam/xact.c:2370-2431`。 `CommitTransaction()` 在大量 pre-commit hook 和状态切换之后，会进入提交的耐久化点。
普通 backend 走：
`latestXid = RecordTransactionCommit();`
附近注释写得很直接： 这里要把 XID 标记为 committed in `pg_xact`。
这就是本地 durable commit 发生的地方。 `RecordTransactionCommit()` 返回之后，`CommitTransaction()` 才调用：
`ProcArrayEndTransaction(MyProc, latestXid)`
这说明一个细节： 同步复制等待期间，本地事务已经在 `pg_xact` 中标记为 committed。
但 backend 仍然显示为 running，并继续持有锁。 这一点在 `RecordTransactionCommit()` 的同步复制注释里也会再次出现。
这不是偶然。 如果已经本地 committed，再因为等待同步复制时收到 cancel 就直接报 ERROR，客户端会以为事务 aborted。
所以 `SyncRepWaitForLSN()` 对取消和终止信号采用 WARNING 加收尾的方式。
---
## 4. RecordTransactionCommit 的输入材料
读 `src/backend/access/transam/xact.c:1337-1379`。 函数开头取当前顶层 XID：
```text
xid = GetTopTransactionIdIfAny()
markXidCommitted = TransactionIdIsValid(xid)
```
如果事务没有 XID，就不需要写普通 commit record。 例如纯只读事务通常不会分配 XID。
它也不需要把某个 XID 标记 committed。 函数还收集几类提交 record 需要的信息：
- pending deletes，对应 `smgrGetPendingDeletes(true, &rels)`。
- committed children，对应子事务 XID。
- transactional stats drops。
- standby invalidation messages。
- relcache init file invalidation 标志。
这些信息不是所有 commit record 都有。 `XactLogCommitRecord()` 会根据是否存在这些信息，设置不同的 `xinfo` 位，并把可选数据接到 WAL record 后面。
`wrote_xlog = (XactLastRecEnd != 0)` 也很关键。 一个事务可能有 XID，但只操作临时表或 unlogged table。
它最后可能没有写 WAL。 这种事务即使提交，也不需要为了普通 WAL durability 调用 `XLogFlush()`。
相反，一个没有 XID 的事务也可能因为 HOT pruning 等原因写过 WAL。 这种 WAL 需要被后台适时 flush，但 crash 后丢失不影响事务可见性语义。
---
## 5. 没有 XID 的特殊路径
读 `src/backend/access/transam/xact.c:1380-1436`。 如果 `markXidCommitted` 为 false，函数不会写普通 commit record。
它先检查不能有 pending deletes 或 dropped stats。 原因很清楚：
删除 relation 文件是事务性提交的一部分。 如果没有 XID 却有待删除文件，系统无法用普通 commit record 和事务状态来保护这个动作。
此时源码直接 `elog(ERROR)`。 没有 XID 的事务仍可能有 invalidation messages。
源码会用 `LogStandbyInvalidations()` 写一个专门 record。 但它不是普通 commit record。
如果没有写任何 WAL，就直接跳到 cleanup。 如果写了 WAL，后面仍会走 flush 或 async 逻辑。
不过因为没有 `markXidCommitted`，它不会更新 `pg_xact`，也不会等待同步复制。 本节后面讲 `synchronous_commit` 时，默认讨论“有 XID 且写过 WAL”的普通写事务。
---
## 6. commit critical section
读 `src/backend/access/transam/xact.c:1448-1484`。 普通有 XID 的提交会进入 commit critical section。
代码会：
```text
START_CRIT_SECTION()
MyProc->delayChkptFlags |= DELAY_CHKPT_IN_COMMIT
pg_write_barrier()
XactLogCommitRecord(...)
```
这里的关键不是防止别的 backend 读到什么。 关键是 checkpoint。
源码注释说，如果 checkpoint 的 REDO 点落在 commit record 之后，但 `pg_xact` 更新没有被 flush 到磁盘，崩溃恢复可能跳过 commit record，又丢失 `pg_xact` 的 committed 状态。 所以提交 record 插入和 `pg_xact` 更新之间，checkpoint 必须知道这个 backend 处于 commit critical section。
checkpoint 侧也有配套逻辑。 读 `src/backend/access/transam/xlog.c:7666-7694`。
在 flush checkpoint record 前，checkpointer 会等待正在这些关键动作中的事务。 这解释了为什么 commit record 插入和 CLOG 更新可以分成两个步骤。
分成两个步骤可以降低锁竞争。 但 checkpoint 必须避开中间窗口。
---
## 7. XactLogCommitRecord 组装 commit record
读 `src/backend/access/transam/xact.c:5864-6033`。 `XactLogCommitRecord()` 是普通 commit 和 twophase commit 共用的 WAL record 组装函数。
如果 `twophase_xid` 无效，`info = XLOG_XACT_COMMIT`。 否则是 `XLOG_XACT_COMMIT_PREPARED`。
普通事务提交会走前者。 函数先填 `xl_xact_commit`：
`xlrec.xact_time = commit_time`
然后根据实际内容设置 `xl_xinfo.xinfo`。 例如：
- relcache init file invalidation 会设置 `XACT_COMPLETION_UPDATE_RELCACHE_FILE`。
- `forceSyncCommit` 会设置 `XACT_COMPLETION_FORCE_SYNC_COMMIT`。
- access exclusive lock 信息会设置 `XACT_XINFO_HAS_AE_LOCKS`。
- 有 subxacts 会设置 `XACT_XINFO_HAS_SUBXACTS`。
- 有待删除 relation 会设置 `XACT_XINFO_HAS_RELFILELOCATORS`。
- 有 invalidation messages 会设置 `XACT_XINFO_HAS_INVALS`。
- 有 replication origin 会设置 `XACT_XINFO_HAS_ORIGIN`。
如果 `synchronous_commit >= SYNCHRONOUS_COMMIT_REMOTE_APPLY`，还会设置：
`XACT_COMPLETION_APPLY_FEEDBACK`
这不是本地 durability 所必需的。 它是给 standby 端 apply feedback 用的。
`remote_apply` 要等备库 replay/apply 到该 commit LSN。 commit record 携带这个标志，能让 standby 在 apply 后更及时反馈。
组装阶段的最后几步很典型：
```text
XLogBeginInsert()
XLogRegisterData(&xlrec, sizeof(xl_xact_commit))
XLogRegisterData(...)
XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN)
return XLogInsert(RM_XACT_ID, info)
```
commit record 没有像 heap page 修改那样注册 buffer block。 它主要注册的是事务元数据。
`RM_XACT_ID` 说明这个 WAL record 由事务资源管理器解释。 返回值就是这条 commit record 的 end LSN。
调用者没有保存返回值，是因为底层 `XLogInsertRecord()` 会更新 `XactLastRecEnd`。
---
## 8. XLogInsert 在哪里
本节重点源码列表没有写 `xloginsert.c`，但当前基线里 `XLogInsert()` 包装函数实际在这个文件。 读 `src/backend/access/transam/xloginsert.c:470-540`。
`XLogInsert(rmid, info)` 要求此前已经调用 `XLogBeginInsert()`。 如果没有，直接 ERROR。
它还检查 `info` 里调用者能设置的位。 然后循环：
```text
GetFullPageWriteInfo(...)
XLogRecordAssemble(...)
EndPos = XLogInsertRecord(...)
```
如果 `XLogInsertRecord()` 返回 `InvalidXLogRecPtr`，说明 checkpoint 或 full page write 状态变化导致需要重新组装 record。 循环会重新 assemble。
成功后调用 `XLogResetInsertion()` 并返回 `EndPos`。 `XLogInsert()` 的注释也强调：
返回的 end LSN 可以作为受影响数据页的 LSN。 这个点要和 commit record 区分。
普通数据页修改会把这个 LSN 写进 page header。 commit record 的 LSN 则被事务提交逻辑拿来决定 flush 和同步复制等待。
---
## 9. XLogInsertRecord 的两阶段插入
读 `src/backend/access/transam/xlog.c:824-855`。 `XLogInsertRecord()` 的注释把插入 WAL buffer 分成两步。
第一步，reserve WAL 空间。 共享的当前插入头在 `Insert->CurrBytePos`。
它由 `insertpos_lck` 保护。 第二步，把已经组装好的 WAL record 拷贝到保留出来的 WAL buffer 区间。
这一步可以由多个 backend 并发做。 为此，每个插入者持有一个 WAL insertion lock。
这个 lock 不只是互斥。 它还带着一个 `insertingAt` 位置，用来告诉 flush 端“我已经插到哪里，哪些更早 WAL 可以安全写出”。
读 `src/backend/access/transam/xlog.c:898-904`。 普通 record 会调用：
`ReserveXLogInsertLocation(rechdr->xl_tot_len, &StartPos, &EndPos, &rechdr->xl_prev)`
读 `src/backend/access/transam/xlog.c:1130-1193`。 这个函数只在短时间持有 `insertpos_lck`。
它把 `CurrBytePos` 增加 record size，并计算 `StartPos`、`EndPos`、`PrevPtr`。 然后读 `src/backend/access/transam/xlog.c:944-989`。
插入者计算 CRC，把 record copy 到 WAL buffer，然后释放 WAL insertion lock。 最后读 `src/backend/access/transam/xlog.c:1109-1127`。
成功插入后设置：
```text
ProcLastRecPtr = StartPos
XactLastRecEnd = EndPos
```
这就是 `RecordTransactionCommit()` 后面能使用 `XactLastRecEnd` 的原因。 注意：
到这里为止，commit record 已经存在于共享 WAL buffer。 它还不一定写进 WAL 文件。
更不一定 fsync。
---
## 10. RecordTransactionCommit 的 durability 分支
读 `src/backend/access/transam/xact.c:1515-1574`。 这里是本节最重要的 if。
源码条件可以整理为：
```text
if ((wrote_xlog && markXidCommitted &&
     synchronous_commit > SYNCHRONOUS_COMMIT_OFF) ||
    forceSyncCommit || nrels > 0)
{
    XLogFlush(XactLastRecEnd);
    if (markXidCommitted)
        TransactionIdCommitTree(...);
}
else
{
    XLogSetAsyncXactLSN(XactLastRecEnd);
    if (markXidCommitted)
        TransactionIdAsyncCommitTree(..., XactLastRecEnd);
}
```
普通同步提交必须同时满足：
- 写过 WAL。
- 有 XID 要标记 committed。
- `synchronous_commit > off`。
如果命令调用过 `ForceSyncCommit()`，也必须同步 flush。 如果 `nrels > 0`，也就是有非临时 relation 文件要删除，也必须同步 flush。
原因是文件删除不能早于 commit record 持久化。 如果先删除文件，再 crash，而 commit record 没有持久化，恢复后系统会认为事务没提交，但文件已经消失。
异步提交路径调用 `XLogSetAsyncXactLSN(XactLastRecEnd)`。 然后用 `TransactionIdAsyncCommitTree()` 标记事务状态。
这里不是立即普通标记 committed。 它会记录这个 `pg_xact` 状态页必须等待哪个 commit LSN flush 后才能安全写出。
这和 `src/backend/access/transam/README:815-874` 里的 async commit 说明一致。
---
## 11. synchronous_commit 的枚举
读 `src/include/access/xact.h:70-84`。 本基线里同步提交级别定义如下：
```text
SYNCHRONOUS_COMMIT_OFF
SYNCHRONOUS_COMMIT_LOCAL_FLUSH
SYNCHRONOUS_COMMIT_REMOTE_WRITE
SYNCHRONOUS_COMMIT_REMOTE_FLUSH
SYNCHRONOUS_COMMIT_REMOTE_APPLY
```
默认：
`SYNCHRONOUS_COMMIT_ON = SYNCHRONOUS_COMMIT_REMOTE_FLUSH`
这意味着配置里的 `on` 并不是一个独立枚举值。 它就是 remote flush 级别。
读 `src/backend/utils/misc/guc_tables.c:341-358`。 用户可见字符串映射为：
| 配置值 | 枚举值 | 本地等待 | 同步备库等待 |
| --- | --- | --- | --- |
| `off` | `SYNCHRONOUS_COMMIT_OFF` | 不等待本地 flush | 不等待 |
| `local` | `SYNCHRONOUS_COMMIT_LOCAL_FLUSH` | 等待本地 flush | 不等待 |
| `remote_write` | `SYNCHRONOUS_COMMIT_REMOTE_WRITE` | 等待本地 flush | 等待备库 write |
| `on` | `SYNCHRONOUS_COMMIT_REMOTE_FLUSH` | 等待本地 flush | 等待备库 flush |
| `remote_apply` | `SYNCHRONOUS_COMMIT_REMOTE_APPLY` | 等待本地 flush | 等待备库 apply |
GUC 还接受 `true/false/yes/no/1/0` 这些别名。 但教学和调试时最好只用文档化的值。
读 `src/backend/utils/misc/guc_parameters.dat:2951-2957`。 `synchronous_commit` 是 USERSET。
一个 session、一个事务、甚至一个 transaction block 内都可以改。 注意：
它控制的是当前事务提交时的等待级别。 它不改变 WAL record 是否生成。
也不改变 WAL redo 的含义。
---
## 12. local、on 和同步备库配置的关系
读 `src/include/replication/syncrep.h:18-20`。 同步复制等待的快速判断是：
```text
max_wal_senders > 0 &&
synchronous_commit > SYNCHRONOUS_COMMIT_LOCAL_FLUSH
```
所以 `off` 和 `local` 一定不会进入同步复制等待。 `remote_write`、`on`、`remote_apply` 理论上会请求同步复制。
但还要看 `synchronous_standby_names`。 读 `src/backend/replication/syncrep.c:179-182`。
如果没有请求同步复制，或者 checkpointer 已经确认没有定义同步 standby，`SyncRepWaitForLSN()` 会快速返回。 因此：
`synchronous_commit = on` 并不保证一定有备库参与。 它表达的是“如果配置了同步复制，等待远端 flush；无同步复制时至少等待本地 flush”。
这个结论对排查线上延迟很重要。 如果用户说“我们 synchronous_commit=on，为什么没有等 standby”，应先检查：
- `synchronous_standby_names` 是否为空。
- standby 的 `application_name` 是否匹配。
- walsender 是否处于 streaming 或 stopping。
- standby 是否已经有有效 flush LSN。
- `max_wal_senders` 是否大于 0。
---
## 13. XLogFlush 的入口行为
读 `src/backend/access/transam/xlog.c:2794-2823`。 `XLogFlush(record)` 的目标是：
确保所有 WAL 数据至少 flush 到 `record`。 如果当前进程不允许插入 WAL，例如恢复期间，它不会写 WAL。
它会调用 `UpdateMinRecoveryPoint(record, false)`。 这对应 standby/recovery 上的语义。
本节主要看 primary 上的普通提交。 primary 上先做快速退出：
```text
if (record <= LogwrtResult.Flush)
    return;
```
如果本进程本地缓存已经知道 flush LSN 达到目标，就不需要任何锁。 否则进入 critical section。
`XLogFlush()` 注释明确说，它不同于 `XLogWrite()` 的地方在于： 调用者还没有持有 `WALWriteLock`。
并且它会尽量避免不必要地获取这个锁。
---
## 14. XLogFlush 先等插入完成
读 `src/backend/access/transam/xlog.c:2858-2867`。 在真正写 WAL 文件之前，`XLogFlush()` 必须保证要写的 WAL buffer 内容已经被所有插入者完整拷贝。
它会先用 `XLogCtl->LogwrtRqst.Write` 扩大目标。 然后调用：
`insertpos = WaitXLogInsertionsToFinish(WriteRqstPtr)`
这个函数名很具体。 它等的不是 fsync。
它等的是“更早的 WAL 插入已经完成到 WAL buffers”。 不能把还没 copy 完的 WAL buffer 写出。
读 `src/backend/access/transam/xlog.c:1530-1654`。 `WaitXLogInsertionsToFinish(upto)` 会检查 `logInsertResult`。
如果已经知道插入完成位置超过 `upto`，直接返回。 否则读取当前 reserved WAL 位置。
如果请求 flush 的 LSN 超过已经 reserved 的 WAL，它会 LOG，并把目标收缩到 reservedUpto。 这通常意味着调用者传了损坏页面上的假 LSN。
然后它遍历所有 WAL insertion locks。 如果某个插入者仍在 `upto` 之前，它会通过 `LWLockWaitForVar()` 等待。
等到安全后，它用 `pg_atomic_monotonic_advance_u64()` 推进 `logInsertResult`。 返回值表示“至少到哪里之前的 WAL 都已完成插入，可以写出”。
---
## 15. WAL insertion lock 的 wait/wakeup
读 `src/backend/access/transam/xlog.c:1480-1528`。 插入者释放 WAL insertion lock 时，会清理 `insertingAt` 并唤醒等待者。
如果插入者在过程中可能阻塞，例如需要初始化或回收 WAL buffer，它会更新 `insertingAt`。 这让 flush 端知道：
这个插入者已经完成了某个位置之前的拷贝。 如果 flush 目标早于这个位置，就不必等它彻底结束。
这个设计避免一个写很靠后的 backend 阻塞前面已经完整的 WAL 被 flush。 它也解释了为什么 `WaitXLogInsertionsToFinish()` 必须在拿 `WALWriteLock` 之前调用。
源码注释在 `src/backend/access/transam/xlog.c:1538-1542` 说得很清楚。 如果持有 `WALWriteLock` 再等插入者，而插入者需要回收旧 WAL buffer 又要 `WALWriteLock`，就会死锁。
所以顺序必须是：
```text
WaitXLogInsertionsToFinish()
LWLockAcquire(WALWriteLock)
XLogWrite()
```
`commit_delay` 后的第二次 `WaitXLogInsertionsToFinish(insertpos)` 是例外。 那次调用不应该真正等待，只是允许 `insertpos` 往前推进。
源码注释说明它只传入之前已经确认完成的位置。
---
## 16. XLogFlush 获取 WALWriteLock
读 `src/backend/access/transam/xlog.c:2868-2891`。 `XLogFlush()` 尝试：
`LWLockAcquireOrWait(WALWriteLock, LW_EXCLUSIVE)`
如果没拿到锁，它会等待锁释放，但并不马上持锁。 它回到循环开头，重新检查 `record <= LogwrtResult.Flush`。
这就是 group commit 的自然形态。 如果另一个 backend 刚才已经把自己的 commit LSN 一起 flush 了，当前 backend 直接退出。
不需要重复 fsync。 如果还没满足，再继续竞争 `WALWriteLock`。
这个逻辑对高并发提交非常重要。 瓶颈通常不是插入 WAL record，而是 fsync。
让等待者复用前一个 fsync，可以明显减少每个事务一次 fsync 的成本。
---
## 17. commit_delay 与 commit_siblings
读 `src/backend/access/transam/xlog.c:2893-2920`。 拿到 `WALWriteLock` 且确认还需要 flush 后，代码可能睡眠。
条件是：
```text
CommitDelay > 0
enableFsync
MinimumActiveBackends(CommitSiblings)
```
`CommitDelay` 在 `src/backend/access/transam/xlog.c:139` 定义，默认 0 微秒。 `CommitSiblings` 在 `src/backend/access/transam/xlog.c:140` 定义，默认 5。
GUC 定义在 `src/backend/utils/misc/guc_parameters.dat:486-500`。 `commit_delay` 是 SUSET，范围 0 到 100000 微秒。
`commit_siblings` 是 USERSET，范围 0 到 1000。 睡眠期间会报告 wait event：
`WAIT_EVENT_COMMIT_DELAY`
睡眠目的不是让当前事务更安全。 目的只是让更多已经或即将插入完成的 commit record 赶上同一次 fsync。
所以它会增加单个事务延迟。 只有在 fsync 成本高且并发提交足够多时，才可能提高吞吐。
在现代存储、低延迟 NVMe、或业务更在乎 tail latency 的系统中，默认 0 往往是合理选择。
---
## 18. XLogWrite 的 write 与 flush
读 `src/backend/access/transam/xlog.c:2313-2580`。 `XLogWrite()` 需要调用者已经持有 `WALWriteLock`。
它接收一个 `XLogwrtRqst`：
```text
WriteRqst.Write
WriteRqst.Flush
```
`Write` 表示至少写到哪里。 `Flush` 表示至少同步到哪里。
`XLogFlush()` 调用时设置二者都等于 `insertpos`。 也就是同步提交要写并 flush 到已经完整插入的最新位置。
`XLogBackgroundFlush()` 调用时可能只要求 write，不一定每次 fsync。 写 WAL 文件使用 `pg_pwrite()`。
等待事件是 `WAIT_EVENT_WAL_WRITE`。 真正 fsync 由 `issue_xlog_fsync()` 负责。
读 `src/backend/access/transam/xlog.c:9353-9420`。 如果 `fsync` 关闭，或者 `wal_sync_method` 是 `open` / `open_datasync` 这类 write 已经同步的方式，`issue_xlog_fsync()` 直接返回。
否则按 `wal_sync_method` 调用 `pg_fsync_no_writethrough()`、`pg_fsync_writethrough()` 或 `pg_fdatasync()`。 fsync 失败是 PANIC。
WAL sync 失败不是普通用户错误。 它意味着数据库无法保证 WAL durability。
---
## 19. XLogFlush 完成后的唤醒
读 `src/backend/access/transam/xlog.c:2933-2942`。 `XLogFlush()` 释放高竞争锁之后，会做两个唤醒动作。
第一：
`WalSndWakeupProcessRequests(true, !RecoveryInProgress())`
这会唤醒 walsender，让备库可以继续接收新的 WAL。 第二：
`WaitLSNWakeup(WAIT_LSN_TYPE_PRIMARY_FLUSH, LogwrtResult.Flush)`
这会唤醒等待 primary flush LSN 的进程。 `xlogwait.c` 是本基线新增或较新的通用 LSN 等待设施。
读 `src/backend/access/transam/xlogwait.c:1-44`。 它的思想是：
等待者把自己需要的 LSN 发布到共享内存。 合适的进程在对应 LSN 到达后设置 latch。
primary 上，backend 或 walwriter 在 flush WAL 后唤醒等待 primary flush 的进程。 standby 上，startup 或 walreceiver 根据 replay、write、flush 位置唤醒等待者。
普通事务提交不直接用 `WaitForLSN()` 等待本地 flush。 提交路径直接调用 `XLogFlush()`。
但 `XLogFlush()` 完成后会通知这套等待设施。
---
## 20. XLogFlush 的坏 LSN 边界
读 `src/backend/access/transam/xlog.c:2944-2977`。 `XLogFlush()` 最后会检查：
```text
if (LogwrtResult.Flush < record)
    elog(ERROR, ...)
```
注释说明最常见原因是请求 flush 的位置超过生成过的 WAL。 例如数据页上有损坏的 LSN。
这里不是一律 PANIC。 如果来自 `bufmgr.c` 这类非 critical section 调用，抛 ERROR 可以避免因为单个坏页让整个系统重启。
但如果来自 `xact.c` 提交路径，情况不同。 `RecordTransactionCommit()` 调 `XLogFlush()` 时仍在 critical section。
critical section 中的 ERROR 会升级为 PANIC。 这符合提交路径的安全要求。
在本地 commit 中途遇到不可恢复错误，不能随意让 backend 继续运行并留下不确定共享状态。
---
## 21. 本地 pg_xact 更新
回到 `src/backend/access/transam/xact.c:1544-1550`。 同步路径先 `XLogFlush(XactLastRecEnd)`。
然后才：
`TransactionIdCommitTree(xid, nchildren, children)`
这一步把事务树标记 committed。 异步路径则是：
```text
XLogSetAsyncXactLSN(XactLastRecEnd)
TransactionIdAsyncCommitTree(..., XactLastRecEnd)
```
区别在于： 同步路径已经确认 commit WAL 持久化，所以可以直接标记 committed。
异步路径还没确认 commit WAL 持久化，所以事务状态页必须记住相关 LSN。 `src/backend/access/transam/README:815-874` 解释了为什么 async commit 还要跟 hint bit 协作。
如果关系页上的 committed hint bit 比 commit WAL 更早落盘，崩溃后就可能出现“页面说已提交，WAL/pg_xact 恢复后却没有这个提交”的不一致。 因此 async commit 要延迟某些 hint bit 设置，或者确保写出相关状态页前先 flush WAL。
---
## 22. 离开 commit critical section
读 `src/backend/access/transam/xact.c:1576-1584`。 本地 `pg_xact` 处理结束后，如果进入过 commit critical section，就清掉：
```text
DELAY_CHKPT_IN_COMMIT
END_CRIT_SECTION()
```
这意味着 checkpoint 不再需要等待这个事务完成提交关键段。 注意时间点：
同步复制等待发生在这之后。 也就是说，同步备库慢不会让 backend 长时间待在 commit critical section。
这是一个非常重要的边界。 等待远端复制可能很久。
如果它也处于 critical section，会放大系统级风险。 PostgreSQL 的选择是：
本地 commit 已经完成。 退出 critical section。
再按用户请求等待远端确认。
---
## 23. SyncRepWaitForLSN 发生在哪里
读 `src/backend/access/transam/xact.c:1589-1599`。 源码注释说：
只有写过 WAL 且有 XID 的事务才等待同步复制。 没有 XID、没有 WAL、只涉及临时或 unlogged 的情况，不需要等。
调用是：
`SyncRepWaitForLSN(XactLastRecEnd, true)`
第二个参数 `true` 表示这是 commit record。 然后 `XactLastCommitEnd = XactLastRecEnd`，并把 `XactLastRecEnd` 重置为 0。
注释还强调： 此时已经标记了 clog。
但事务仍在 procarray 中显示为 running，并继续持锁。 这就是为什么同步复制等待会阻塞客户端收到 COMMIT 结果，也会延迟释放锁。
但它不是本地 crash recovery 所需的等待。 本地 crash recovery 所需的 commit WAL flush 已经在前面完成。
---
## 24. assign_synchronous_commit
读 `src/backend/replication/syncrep.c:1133-1149`。 `synchronous_commit` 的 assign hook 会把 GUC 枚举映射为 `SyncRepWaitMode`。
映射如下：
- `remote_write` -> `SYNC_REP_WAIT_WRITE`
- `on` / `remote_flush` 枚举 -> `SYNC_REP_WAIT_FLUSH`
- `remote_apply` -> `SYNC_REP_WAIT_APPLY`
- 其他 -> `SYNC_REP_NO_WAIT`
这里的“其他”包括 `off` 和 `local`。 这解释了 `local` 的语义：
它仍然让 `RecordTransactionCommit()` 调用本地 `XLogFlush()`。 但 `SyncRepWaitForLSN()` 里不会等待远端。
`remote_write`、`on`、`remote_apply` 都会先完成本地 flush。 然后才由 `SyncRepWaitMode` 决定远端等待级别。
---
## 25. SyncRepWaitForLSN 快速退出
读 `src/backend/replication/syncrep.c:149-245`。 函数一开始断言：
`InterruptHoldoffCount > 0`
提交路径在 `CommitTransaction()` 中已经 `HOLD_INTERRUPTS()`。 这样可以避免队列清理被外部中断打断。
快速退出条件包括：
- 没有请求同步复制。
- checkpointer 已经确认没有同步 standby 定义。
- 当前等待 LSN 已经小于等于对应模式下已确认的 `WalSndCtl->lsn[mode]`。
- 同步 standby 数据尚未初始化，但 GUC 也没定义同步 standby。
这些 fast path 很重要。 `SyncRepWaitForLSN()` 每次 commit 都会调用。
没有同步复制的系统不能因为这个调用付出明显成本。 如果不是 commit record，并且用户配置了 `remote_apply`，mode 会被 cap 到 `SYNC_REP_WAIT_FLUSH`。
原因在注释里： 只有 commit record 提供 apply feedback。
非 commit LSN 不能要求 remote apply 语义。
---
## 26. SyncRep 队列
读 `src/backend/replication/syncrep.c:247-255`。 真正需要等待时，backend 设置：
```text
MyProc->waitLSN = lsn
MyProc->syncRepState = SYNC_REP_WAITING
SyncRepQueueInsert(mode)
```
队列按 LSN 排序。 读 `src/backend/replication/syncrep.c:376-410`。
`SyncRepQueueInsert()` 从队尾向前找插入点。 通常新提交的 LSN 更大，所以会插到尾部。
但 out-of-order 到达也能维护排序。 `src/include/replication/walsender_private.h:83-103` 定义了共享状态：
- 每个等待级别一个 `SyncRepQueue`。
- 每个等待级别一个当前已确认 `lsn`。
- 一个 `sync_standbys_status`，让 backend 快速知道是否定义了同步 standby。
队列由 `SyncRepLock` 保护。 等待本身不持有这个锁。
---
## 27. SyncRepWaitForLSN 的等待循环
读 `src/backend/replication/syncrep.c:266-355`。 backend 使用自己的 latch 等待。
循环中先 `ResetLatch(MyLatch)`。 然后检查：
`MyProc->syncRepState == SYNC_REP_WAIT_COMPLETE`
这个状态由 walsender 侧设置。 如果有 `ProcDiePending`，也就是终止连接请求到达，不能直接 ERROR。
源码注释解释得很重要： 事务已经本地 committed。
如果此时报 ERROR 或 FATAL，客户端会误以为事务 aborted。 所以它发 WARNING，说明事务已经本地提交但可能没复制到 standby，然后取消等待并停止后续输出。
`QueryCancelPending` 也是类似处理。 用户取消等待时，会 WARNING：
事务已经本地提交，但可能还没复制到 standby。 如果 postmaster death，backend 也取消等待并准备退出。
所有这些情况都说明： 同步复制等待不是本地事务能否回滚的阶段。
它是“提交已经发生后，是否还能向客户端承诺远端也达到某个位置”的阶段。
---
## 28. walsender 如何释放等待者
读 `src/backend/replication/walsender.c:2501-2597`。 standby 定期发 reply，包含：
```text
writePtr
flushPtr
applyPtr
replyTime
replyRequested
```
walsender 把这些位置写入自己的共享 `WalSnd` 结构。 然后在非 cascading walsender 上调用：
`SyncRepReleaseWaiters()`
读 `src/backend/replication/syncrep.c:484-583`。 `SyncRepReleaseWaiters()` 先确认自己连接的 standby 是潜在同步 standby。
要求：
- `sync_standby_priority != 0`
- walsender 状态是 streaming 或 stopping
- standby flush 位置有效
然后持有 `SyncRepLock`，调用 `SyncRepGetSyncRecPtr()` 计算当前同步位置。 如果满足要求，就更新：
```text
WalSndCtl->lsn[SYNC_REP_WAIT_WRITE]
WalSndCtl->lsn[SYNC_REP_WAIT_FLUSH]
WalSndCtl->lsn[SYNC_REP_WAIT_APPLY]
```
并分别唤醒对应队列。
---
## 29. priority 与 quorum 的同步位置
读 `src/backend/replication/syncrep.c:596-664`。 `SyncRepGetSyncRecPtr()` 会取得当前候选同步 standby。
如果候选数量少于 `synchronous_standby_names` 要求的 `num_sync`，返回 false。 如果当前 walsender 不是同步 standby，也不释放等待者。
priority 模式下，等待位置取同步 standby 中最旧的位置。 读 `src/backend/replication/syncrep.c:670-696`。
这意味着所有被选中的同步 standby 都要达到该 LSN。 quorum 模式下，等待位置取第 N 新的位置。
读 `src/backend/replication/syncrep.c:703-742`。 这意味着任意 N 个候选 standby 达到即可。
用户配置 `FIRST` 和 `ANY` 时，背后就是这两种计算方式。 本节不展开同步复制配置语法。
但要记住： `synchronous_commit` 只决定等待 write、flush 还是 apply。
等待哪些 standby、等几个 standby，由 `synchronous_standby_names` 和当前 walsender 状态决定。
---
## 30. SyncRepWakeQueue 的唤醒边界
读 `src/backend/replication/syncrep.c:907-963`。 `SyncRepWakeQueue(all, mode)` 从队头开始走。
队列按 LSN 升序排列。 如果不是 `all`，遇到第一个 `proc->waitLSN` 大于当前确认 LSN 的 backend，就停止。
对可以释放的 backend：
```text
dlist_delete_thoroughly(&proc->syncRepLinks)
pg_write_barrier()
proc->syncRepState = SYNC_REP_WAIT_COMPLETE
SetLatch(&proc->procLatch)
```
写屏障是为了保证等待者看到状态完成时，队列链接已经被移除。 等待者醒来后会清理自己的 `syncRepState` 和 `waitLSN`。
这是一套典型的“共享状态 + latch”等待模型。 不是条件变量。
也不是每个 commit record 单独向 standby 发送确认请求。 standby 只上报自己的 WAL 进度。
primary 根据进度释放所有满足条件的等待者。
---
## 31. walwriter 的定位
读 `src/backend/postmaster/walwriter.c:1-31`。 文件头注释已经把 walwriter 的职责说清楚：
它尝试让普通 backend 少做 WAL write 和 fsync。 它保证没有在 commit 时立即 sync 的 async commit record 会在可知时间内到达磁盘。
最坏大约是三倍 `wal_writer_delay` 周期。 同时注释也明确说：
普通 backend 仍然可以自己发起 WAL write 和 fsync。 walwriter 不是必需进程。
不能把额外重功能塞进 walwriter，因为它的周期直接影响 async commit 的最大延迟。 这几句话能回答一个常见问题：
为什么有 walwriter，commit 还要自己 `XLogFlush()`？ 因为同步 commit 的客户端响应不能等后台进程下一轮碰巧刷到。
当前 backend 必须确认自己的 commit LSN 已经满足要求。
---
## 32. WalWriterMain 主循环
读 `src/backend/postmaster/walwriter.c:89-269`。 walwriter 是 postmaster 启动的辅助进程。
它设置信号处理、创建内存上下文、注册自己的 proc number。 主循环中：
```text
if (XLogBackgroundFlush())
    left_till_hibernate = LOOPS_UNTIL_HIBERNATE;
else
    left_till_hibernate--;
pgstat_report_wal(false);
WaitLatch(..., WalWriterDelay 或 WalWriterDelay * HIBERNATE_FACTOR, ...)
```
如果长期无事可做，它会进入 hibernating。 进入 hibernating 前会调用：
`SetWalWriterSleeping(true)`
这样 async commit backend 可以看到 walwriter 正在睡，并设置它的 latch。 读 `src/backend/access/transam/xlog.c:10186-10195`。
`SetWalWriterSleeping()` 只是在 `XLogCtl->info_lck` 下更新一个共享 bool。
---
## 33. XLogSetAsyncXactLSN
读 `src/backend/access/transam/xlog.c:2624-2680`。 异步提交路径会调用：
`XLogSetAsyncXactLSN(XactLastRecEnd)`
这个函数更新共享的 `XLogCtl->asyncXactLSN`。 它只记录更大的 LSN。
如果传入 LSN 不比已有 async LSN 新，直接返回。 然后判断是否要唤醒 walwriter。
如果 walwriter 正在 sleeping，唤醒。 如果没 sleeping，则根据 `wal_writer_flush_after` 计算未 flush WAL block 数。
超过门槛也唤醒。 唤醒动作是设置 walwriter 进程 latch：
`SetLatch(&GetPGProcByNumber(walwriterProc)->procLatch)`
这个函数的注释写着： 它不应该用于同步提交。
同步提交走 `XLogFlush()`，不只是登记一个后台任务。
---
## 34. XLogBackgroundFlush
读 `src/backend/access/transam/xlog.c:2979-3144`。 `XLogBackgroundFlush()` 是 walwriter 周期调用的核心。
它通常只写完整 WAL page。 如果已经 flush 到完整 page 边界，它会考虑 `asyncXactLSN`。
这样在活动停止、最后一个 commit record 落在当前未满 WAL page 里时，也能把 async commit 推到磁盘。 它的 flush 节奏由两个 GUC 控制：
- `wal_writer_delay`
- `wal_writer_flush_after`
如果距离上次 flush 超过 `WalWriterDelay`，会 flush。 如果未 flush blocks 超过 `WalWriterFlushAfter`，会 flush。
否则可能只 write 不 flush，或者本轮不做事。 它调用 `XLogWrite(WriteRqst, insertTLI, flexible)`。
`flexible = true` 时，可以在方便边界停下，减少高负载下的重复写。 这也是 async commit 最坏三倍 `wal_writer_delay` 的原因之一。
`XLogBackgroundFlush()` 完成后也会：
```text
WalSndWakeupProcessRequests(...)
WaitLSNWakeup(WAIT_LSN_TYPE_PRIMARY_FLUSH, LogwrtResult.Flush)
AdvanceXLInsertBuffer(...)
```
所以 walwriter 不只是刷异步提交。 它也会为未来 WAL 插入预初始化 buffer，降低前台插入路径压力。
---
## 35. walwriter 能做什么
walwriter 能做：
- 周期性写出完整 WAL page。
- 在 async commit 场景下，把最新 async commit LSN 推向磁盘。
- 根据 `wal_writer_delay` 和 `wal_writer_flush_after` 控制后台 flush 节奏。
- 被 async commit backend 唤醒，避免 hibernation 放大提交丢失窗口。
- 唤醒 walsender，让复制继续消费 WAL。
- 唤醒等待 primary flush LSN 的 `xlogwait` 等待者。
- 预初始化未来 WAL buffer，减轻前台路径。
walwriter 不能做：
- 不能决定事务是否提交。
- 不能替代 `RecordTransactionCommit()` 写 commit record。
- 不能替代同步 commit 的 `XLogFlush()`。
- 不能让 `synchronous_commit = off` 变成零丢失风险。
- 不能保证 standby write/flush/apply。
- 不能绕过 `SyncRepWaitForLSN()` 的队列和 walsender reply。
- 不能在 WAL fsync 失败时继续假装提交安全。
一句话： walwriter 是后台 amortization 机制。
不是当前事务的同步确认机制。
---
## 36. xlogwait.c 与本节关系
读 `src/backend/access/transam/xlogwait.c:80-142`。 `GetCurrentLSNForWaitType()` 支持四种等待：
- standby replay
- standby write
- standby flush
- primary flush
primary flush 使用：
`GetFlushRecPtr(NULL)`
读 `src/backend/access/transam/xlogwait.c:340-360`。 `WaitLSNWakeup()` 先看 `minWaitedLSN` fast path。
如果当前 LSN 还没到最小等待 LSN，直接返回。 否则进入 heap，唤醒所有已满足的等待者。
读 `src/backend/access/transam/xlogwait.c:394-495`。 `WaitForLSN()` 会把当前进程加入对应 LSN type 的 pairing heap。
然后用 latch 等待。 如果 standby wait type 在等待期间恢复结束，会返回 NOT_IN_RECOVERY。
如果 timeout 到了，返回 TIMEOUT。 这些函数不是 `synchronous_commit` 的同步复制等待实现。
同步复制等待用的是 `syncrep.c` 中的 `SyncRepQueue` 和 `syncRepState`。 但本地 WAL flush 后的 `WaitLSNWakeup(WAIT_LSN_TYPE_PRIMARY_FLUSH, ...)` 是本基线中 WAL flush 事件的统一通知点。
---
## 37. 崩溃边界：synchronous_commit=off
`synchronous_commit = off` 的承诺是： 事务 commit 可以先返回客户端。
commit record 稍后由 walwriter 或其他 flush 带到磁盘。 它不是不写 WAL。
它也不是不写 commit record。 普通有 XID、写过 WAL 的事务仍然会插入 commit record。
只是 `RecordTransactionCommit()` 不等待 `XLogFlush(XactLastRecEnd)`。 崩溃边界是：
如果 PostgreSQL crash、OS crash、断电发生在 commit record flush 前，事务可能丢失。 如果只是单个 backend 进程在返回后崩溃，而 postmaster 和共享内存仍正常，walwriter 或别的 backend 仍可能 flush 它。
但用户不能把 async commit 当成单事务持久化保证。 异步提交适合：
- 可以容忍少量已确认事务在 crash 后消失的日志类工作负载。
- 高并发小事务，并且吞吐明显受 fsync 限制。
- 应用层有幂等或补偿机制。
不适合：
- 金融账务。
- 外部世界已经不可回滚执行的动作。
- 用户看到成功后必须永久可见的状态变化。
---
## 38. 崩溃边界：local/on/remote*
`local`、`on`、`remote_write`、`remote_apply` 都会在本地调用 `XLogFlush()`。 只要 `XLogFlush()` 成功返回，主库本地 crash recovery 可以重放到 commit record。
本地事务不会因为主库崩溃而丢失。 `remote_write` 额外等待同步 standby 报告已经 write。
这通常意味着 standby 的 OS 已接收并写入文件系统缓存，但不一定 fsync。 如果 standby OS crash 或断电，可能仍丢。
`on` 额外等待 standby flush。 这意味着同步 standby 把 WAL 持久化到自己的稳定存储。
如果 primary 丢失，可以更可靠地从 standby 接管到该 commit。 `remote_apply` 额外等待 standby apply。
这意味着 standby 上 replay/apply 到该 commit LSN。 读-only 查询在该 standby 上可以看到提交结果。
代价是延迟更高。 它受 standby replay 速度、冲突、apply 队列影响。
---
## 39. 错误边界：本地 WAL write/fsync
`XLogWrite()` 中 `pg_pwrite()` 失败会 PANIC。 读 `src/backend/access/transam/xlog.c:2445-2477`。
WAL 写失败不能作为普通 ERROR 处理。 否则系统可能继续运行在 WAL 持久性不明的状态。
`issue_xlog_fsync()` 失败也会 PANIC。 读 `src/backend/access/transam/xlog.c:9410-9420`。
这类错误通常会导致 postmaster 重启并进入恢复。 这也是为什么“fsync=off”只是测试或特殊场景配置。
`fsync=off` 会让 `issue_xlog_fsync()` 快速返回。 它提高性能，但放弃操作系统或硬件崩溃时的数据一致性保证。
本节所有关于 flush 的耐久性讨论，都默认 `fsync=on` 且存储正确实现 flush 语义。
---
## 40. 错误边界：同步复制等待
`SyncRepWaitForLSN()` 的等待阶段，本地 commit 已经发生。 因此错误处理很特殊。
如果管理员终止连接： 它 WARNING，并说明事务已经本地 committed，但可能没复制到 standby。
如果用户 cancel： 它 WARNING，并取消等待。
如果 postmaster death： 它取消等待并准备退出。
这些行为看起来“不像普通 SQL 错误”。 原因是此时不允许把事务结果伪装成 aborted。
这也是应用层需要理解的边界： 客户端可能收到连接中断或 warning。
事务在 primary 本地可能已经提交。 如果业务需要精确确认，应在重连后用业务 key 或事务结果查询确认状态。
---
## 41. 错误边界：同步 standby 消失
同步 standby 消失或 `synchronous_standby_names` 改变时，等待者不会由 standby 主动通知。 primary 上的 checkpointer 会调用 `SyncRepUpdateSyncStandbysDefined()` 更新共享状态。
读 `src/backend/replication/syncrep.c:965-1029`。 如果同步 standby 名单被清空，它会唤醒所有等待队列。
这样 backend 不会永远卡住。 如果只是某个同步 standby 断开，在 priority 模式下可能由下一个候选 standby 接替。
如果候选数量不足，等待会继续，直到配置变化、standby 恢复，或会话被取消/终止。 所以同步复制是可用性和数据安全之间的选择。
配置成必须等待 standby，就要接受 standby 不可用时提交可能阻塞。
---
## 42. group commit 不是 batch commit
group commit 容易被误解成 PostgreSQL 把多个事务合并成一个 commit record。 不是。
每个事务仍有自己的 commit record。 每个 commit record 有自己的 LSN。
group commit 合并的是 WAL flush/fsync。 在 `XLogFlush()` 中，一个 backend 拿到 `WALWriteLock` 后，会把 `insertpos` 之前已经完整插入的 WAL 一起写出并 flush。
这些 WAL 可能包含多个事务的 commit records。 其他 backend 重新检查自己的 LSN，发现已经 flush 到，就直接完成。
所以 group commit 不改变事务原子性。 不改变 WAL record 格式。
也不改变恢复逻辑。 它只减少昂贵同步 I/O 的次数。
---
## 43. commit_delay 的使用建议
`commit_delay` 默认 0。 这是有原因的。
它只在非常特定的 workload 下有价值。 需要满足：
- 并发提交很多。
- 单次 fsync 成本高。
- 事务延迟可以增加一点。
- group commit 没有自然发生到足够程度。
它不适合盲目开启。 如果开启，也应先观察：
- `pg_stat_activity` 中是否出现 `CommitDelay` 等待。
- TPS 是否上升。
- p95/p99 commit latency 是否可接受。
- WAL sync 时间是否下降。
在 `synchronous_commit = off` 场景下，普通 commit 不走同步 `XLogFlush()`，因此这个延迟通常不是主路径。 在 `synchronous_commit > off` 的高并发小事务场景下，它才可能影响明显。
---
## 44. 源码跟读练习一：普通同步提交
目标：沿着一次 `synchronous_commit = local` 的普通 insert commit 走源码。 步骤：
1. 在 `xact.c` 找到 `CommitTransaction()` 调用 `RecordTransactionCommit()` 的位置。
2. 进入 `RecordTransactionCommit()`。
3. 确认 `markXidCommitted` 什么时候为 true。
4. 找到 `XactLogCommitRecord()` 调用。
5. 进入 `XactLogCommitRecord()`。
6. 确认 `info = XLOG_XACT_COMMIT` 的条件。
7. 看 `XLogBeginInsert()` 和多次 `XLogRegisterData()`。
8. 看最后 `XLogInsert(RM_XACT_ID, info)`。
9. 进入 `xloginsert.c` 的 `XLogInsert()`。
10. 跟到 `xlog.c` 的 `XLogInsertRecord()`。
11. 找到 `ProcLastRecPtr = StartPos` 和 `XactLastRecEnd = EndPos`。
12. 回到 `RecordTransactionCommit()` 的 if 条件。
13. 因为 `local > off`，确认会调用 `XLogFlush(XactLastRecEnd)`。
14. 确认 `SyncRepWaitForLSN()` 不会真正等待远端。
15. 回到 `CommitTransaction()`，看 `ProcArrayEndTransaction()` 发生在提交记录之后。
检查点： 你应该能说清楚 commit record 什么时候生成，什么时候 flush，什么时候事务从 procarray 退出。
---
## 45. 源码跟读练习二：异步提交
目标：沿着一次 `synchronous_commit = off` 的普通写事务走源码。 步骤：
1. 在 session 中设置 `SET synchronous_commit = off;`。
2. 对照 `guc_tables.c` 确认它映射到 `SYNCHRONOUS_COMMIT_OFF`。
3. 在 `RecordTransactionCommit()` 中找到 durability if。
4. 观察普通写事务满足 `wrote_xlog && markXidCommitted`。
5. 但 `synchronous_commit > off` 为 false。
6. 如果没有 `forceSyncCommit` 和 `nrels > 0`，进入 else。
7. 跟到 `XLogSetAsyncXactLSN()`。
8. 看它如何更新 `asyncXactLSN`。
9. 看它什么时候唤醒 walwriter。
10. 回到 `RecordTransactionCommit()`。
11. 看 `TransactionIdAsyncCommitTree()` 为什么需要传入 `XactLastRecEnd`。
12. 读 `access/transam/README` 的 async commit 段落。
13. 确认 async commit 不是“不写 commit record”。
14. 确认 async commit 风险窗口在哪。
检查点： 你应该能解释为什么 async commit 返回成功后，crash recovery 仍可能看不到这个事务。
---
## 46. 源码跟读练习三：group commit
目标：理解多个 backend 如何共享一次 flush。 步骤：
1. 打开 `xlog.c` 的 `XLogFlush()`。
2. 找到 `record <= LogwrtResult.Flush` 的 quick exit。
3. 找到 `WaitXLogInsertionsToFinish(WriteRqstPtr)`。
4. 找到 `LWLockAcquireOrWait(WALWriteLock, LW_EXCLUSIVE)`。
5. 注意没拿到锁时，函数不是立即拿锁，而是回到循环重新检查 flush 状态。
6. 找到拿到锁后的第二次 `record <= LogwrtResult.Flush`。
7. 找到 `CommitDelay` 条件。
8. 找到 `WriteRqst.Write = insertpos` 和 `WriteRqst.Flush = insertpos`。
9. 跟到 `XLogWrite()`。
10. 看 `LogwrtResult.Flush` 如何更新。
11. 回到 `XLogFlush()` 看 walsender 和 WaitLSN 的唤醒。
检查点： 你应该能解释为什么 group commit 不需要一个显式“commit group”对象。
---
## 47. 源码跟读练习四：同步复制等待
目标：理解 `synchronous_commit = remote_apply` 的等待路径。 步骤：
1. 在 `xact.h` 确认 `remote_apply` 的枚举值。
2. 在 `guc_tables.c` 确认字符串到枚举的映射。
3. 在 `syncrep.c` 的 `assign_synchronous_commit()` 确认 wait mode 是 `SYNC_REP_WAIT_APPLY`。
4. 在 `XactLogCommitRecord()` 中找到 `XACT_COMPLETION_APPLY_FEEDBACK`。
5. 在 `RecordTransactionCommit()` 中确认本地 `XLogFlush()` 先完成。
6. 进入 `SyncRepWaitForLSN(XactLastRecEnd, true)`。
7. 看 fast path 什么时候返回。
8. 看 `MyProc->waitLSN` 和 `syncRepState` 怎么设置。
9. 看 `SyncRepQueueInsert()` 如何保持队列按 LSN 排序。
10. 看等待循环如何处理 cancel、die、postmaster death。
11. 到 `walsender.c` 的 `ProcessStandbyReplyMessage()`。
12. 看 standby reply 中的 write、flush、apply 如何进入共享状态。
13. 回到 `SyncRepReleaseWaiters()`。
14. 看 priority/quorum 如何计算同步位置。
15. 看 `SyncRepWakeQueue()` 如何设置 `SYNC_REP_WAIT_COMPLETE` 并唤醒 backend。
检查点： 你应该能解释为什么远端等待失败时，事务不能简单回滚。
---
## 48. 实验一：观察本地 flush 等待
准备一个测试库。 确保 `fsync=on`。
执行：
```sql
SHOW synchronous_commit;
SET synchronous_commit = local;
CREATE TABLE wal_commit_demo(id bigint generated always as identity, payload text);
```
在一个 session 中重复插入：
```sql
INSERT INTO wal_commit_demo(payload)
SELECT repeat('x', 1000)
FROM generate_series(1, 1000);
```
另一个 session 观察：
```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL;
```
你可能看到 `WALWrite`、`WALSync` 或存储相关等待。 是否能看到取决于机器速度和 workload。
在很快的本地开发机上，等待可能一闪而过。 这个实验的目标不是稳定复现某个等待事件。
目标是把 SQL 层 commit 和源码里的 `XLogFlush()` 联系起来。
---
## 49. 实验二：比较 synchronous_commit=off
同一张表上执行：
```sql
SET synchronous_commit = off;
INSERT INTO wal_commit_demo(payload)
SELECT repeat('y', 1000)
FROM generate_series(1, 1000);
```
观察提交延迟是否下降。 再执行：
```sql
SELECT pg_current_wal_insert_lsn(), pg_current_wal_flush_lsn();
```
在有持续写入时，insert LSN 可能领先 flush LSN。 这不是错误。
它表示 WAL 已插入或写入进度和持久化进度不是同一个概念。 然后切回：
```sql
SET synchronous_commit = local;
```
再次观察延迟。 注意：
不要用 kill -9 或断电来“验证数据丢失”，除非在可丢弃测试集群。 异步提交的风险是真实 crash 边界，不应该在重要实例上实验。
---
## 50. 实验三：观察 group commit 倾向
开多个客户端并发执行小事务。 例如用 `pgbench`：
```bash
pgbench -i -s 10 testdb
pgbench -c 32 -j 8 -T 60 -N testdb
```
分别测试：
```sql
ALTER SYSTEM SET commit_delay = 0;
ALTER SYSTEM SET commit_siblings = 5;
SELECT pg_reload_conf();
```
和一个谨慎的非零值：
```sql
ALTER SYSTEM SET commit_delay = 20;
ALTER SYSTEM SET commit_siblings = 5;
SELECT pg_reload_conf();
```
观察 TPS 和延迟分布。 如果要更明确地看影响，可以增加 `pgbench` 客户端数。
但不要只看平均 TPS。 要看 latency，尤其是 p95/p99。
`commit_delay` 可能提高吞吐，也可能只是增加延迟。 这取决于存储和并发模式。
实验结束后恢复：
```sql
ALTER SYSTEM RESET commit_delay;
ALTER SYSTEM RESET commit_siblings;
SELECT pg_reload_conf();
```
---
## 51. 讨论题

1. 为什么 `XLogInsert()` 返回 commit record end LSN 后，仍不能直接向客户端承诺同步提交完成？
2. `synchronous_commit = off` 返回成功后，PostgreSQL crash 和 OS crash 分别可能暴露什么风险窗口？
3. 为什么 `local`、`on`、`remote_write`、`remote_apply` 都先完成本地 `XLogFlush()`，再讨论同步备库等待？
4. 没有有效同步 standby 时，`synchronous_commit = on` 为什么主要退化成本地 flush？
5. group commit 为什么可以不引入显式“commit group”对象，而依赖 `WALWriteLock` 和 flush position 重检？
6. 同步复制等待期间收到 cancel，为什么不能把事务简单回滚？
7. `pg_stat_replication` 能看到 write/flush/replay LSN，但为什么仍不能完整解释一次 commit latency？
8. 本节的可迁移规律是什么：怎样把“本地持久化承诺”和“远端可见/持久化承诺”拆成连续但不同的等待边界？
---
## 52. 常见误区一：COMMIT record 和 pg_xact 谁说了算
恢复时，commit record 是重建事务状态的重要依据。 运行时，其他事务判断可见性通常查 `pg_xact` 和快照。
所以提交路径需要两者一致。 `RecordTransactionCommit()` 先写 commit WAL record。
再确保 WAL 持久化。 再更新 `pg_xact`。
checkpoint 还要避开 commit record 已经在 WAL 中但 `pg_xact` 更新尚未安全的窗口。 这不是重复劳动。
WAL 是恢复日志。 `pg_xact` 是运行时事务状态存储。
两者服务不同阶段。
---
## 53. 常见误区二：on 一定等备库
`synchronous_commit = on` 的枚举是 remote flush。 但是否真的等备库，还要看同步复制是否被请求并定义。
如果 `synchronous_standby_names` 为空，`SyncRepWaitForLSN()` 快速返回。 此时 `on` 的效果是本地 flush。
这也是 PostgreSQL 默认可以在没有备库的单机上正常运行的原因。 不要只看 `SHOW synchronous_commit` 判断是否有同步复制。
还要看 `SHOW synchronous_standby_names` 和 `pg_stat_replication.sync_state`。
---
## 54. 常见误区三：walwriter 会替我同步提交
walwriter 可以帮助写 WAL。 也可以及时处理 async commit。
但同步提交的语义是： 当前事务返回成功前，它自己的 commit LSN 已经满足本地或远端等待条件。
这个语义不能交给后台周期任务碰运气。 即使 walwriter 刚好已经 flush 到你的 LSN，`XLogFlush()` 也会通过 quick exit 立刻返回。
但当前 backend 仍然要检查这个事实。 所以 walwriter 是优化路径，不是语义路径。
---
## 55. 常见误区四：remote_apply 更安全所以总该开
`remote_apply` 等 standby replay/apply。 它的额外价值是：
提交返回后，同步 standby 上的只读查询可以看到结果。 在某些 failover 策略中，它也减少“WAL 已持久化但尚未 replay”的可见性差距。
但它不总是必要。 如果目标只是确保备用节点有持久 WAL，`on` 即 remote flush 通常已经表达这个需求。
`remote_apply` 会受 standby apply 速度影响。 长查询、冲突、慢 replay 都可能拖慢主库提交。
所以选择级别时，要先说清楚业务要等的是 write、flush 还是可见。
---
## 56. 常见误区五：commit_delay 是提交安全参数
`commit_delay` 不增强安全。 它只改变性能形态。
它让拿到 `WALWriteLock` 的 backend 在 flush 前等一小会儿。 目的是聚合更多 commit record 到同一次 fsync。
如果设置太大，会直接增加 commit latency。 如果并发不够，`MinimumActiveBackends(CommitSiblings)` 不满足，它不会生效。
如果 `fsync=off`，它也不会生效。 所以它不是“让提交更稳”的参数。
它是一个很窄的 group commit 调优旋钮。
---
## 57. 源码断点建议
如果用 gdb 跟一次普通提交，可以设置这些断点：
```text
break RecordTransactionCommit
break XactLogCommitRecord
break XLogInsert
break XLogInsertRecord
break XLogFlush
break XLogWrite
break XLogSetAsyncXactLSN
break SyncRepWaitForLSN
break SyncRepReleaseWaiters
break XLogBackgroundFlush
```
单机无备库时，`SyncRepWaitForLSN` 可能命中后快速返回。 `synchronous_commit = off` 时，`XLogFlush` 不一定在提交路径命中。
但 walwriter 后续会走 `XLogBackgroundFlush`。 如果要观察 `commit_delay`，需要：
- 设置非零 `commit_delay`。
- 确认 `fsync=on`。
- 有足够并发 active backend。
否则断点可能不进入 sleep 分支。
---
## 58. 一张提交路径图
普通同步提交：
```text
backend
  XactLogCommitRecord
    XLogInsert
      reserve WAL
      copy record into WAL buffers
      XactLastRecEnd = commit end LSN
  XLogFlush(commit end LSN)
    wait older insertions finish
    acquire or wait WALWriteLock
    maybe commit_delay
    XLogWrite(write=flush=insertpos)
    wake walsenders and LSN waiters
  TransactionIdCommitTree
  leave commit critical section
  SyncRepWaitForLSN, if needed
  ProcArrayEndTransaction
  release locks and resources
```
异步提交：
```text
backend
  XactLogCommitRecord
    XLogInsert
      XactLastRecEnd = commit end LSN
  XLogSetAsyncXactLSN(commit end LSN)
  TransactionIdAsyncCommitTree(..., commit end LSN)
  leave commit critical section
  no sync rep wait for off
  return to client

walwriter or later backend
  XLogBackgroundFlush or XLogFlush
  write/flush WAL including async commit record
```
同步复制提交：
```text
backend
  local commit path, including XLogFlush
  TransactionIdCommitTree
  leave commit critical section
  SyncRepWaitForLSN(commit end LSN)
    enqueue by wait mode
    sleep on latch

walsender
  receive standby write/flush/apply reply
  SyncRepReleaseWaiters
  SyncRepWakeQueue
  SetLatch(waiting backend)
```
---
## 59. 主线收束
读完源码后，把判断收束到三个边界。
第一，commit record 只是进入 WAL byte stream；它的 end LSN 还要被本地 flush 或 async commit 机制消费。
第二，`pg_xact` 是运行时事务状态，WAL 是 crash recovery 事实来源；提交路径必须让两者在 critical section 内按顺序对齐。
第三，同步复制等待发生在本地 commit 之后，它改变客户端等待和 failover 语义，不改变本地事务已经 committed 的事实。
如果一次诊断不能先说清楚这三个边界，就不要直接从 `synchronous_commit` 的字符串值推断 latency 或 durability。
---
## 60. 本节小结
commit record 是事务提交进入 WAL 历史的标记。 `RecordTransactionCommit()` 是普通事务本地提交的核心函数。
它通过 `XactLogCommitRecord()` 写入 `RM_XACT_ID` 的 commit record。 `XLogInsert()` 只负责把 record 插入 WAL buffer 并返回 end LSN。
本地持久化由 `XLogFlush(XactLastRecEnd)` 完成。 `synchronous_commit = off` 跳过当前 backend 的本地 flush 等待，改由 async commit LSN 和 walwriter 后续处理。
`local` 等本地 flush，不等同步备库。 `on` 等本地 flush，并在配置了同步复制时等备库 flush。
`remote_write` 等备库 write。 `remote_apply` 等备库 apply。
group commit 发生在 `XLogFlush()` 的 `WALWriteLock` 竞争和 piggyback flush 中。 `commit_delay` 是可选延迟，用来让更多 backend 赶上同一次 fsync。
同步复制等待发生在本地 commit 已经完成之后。 因此等待期间的取消、终止、postmaster death 都不能被解释成事务回滚。
walwriter 能降低前台 WAL I/O 压力，并约束 async commit 的落盘延迟。 但它不是同步提交的语义保证。
提交路径最终要守住的边界仍是： 在向客户端承诺某个 durability 级别之前，commit record 的 LSN 必须已经达到该级别要求的位置。
