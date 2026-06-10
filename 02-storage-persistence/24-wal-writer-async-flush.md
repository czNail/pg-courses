# PostgreSQL WAL Writer 与异步 WAL flush

## 课程定位

本节主题：WAL writer 怎样周期性写出和 flush WAL，以及异步提交怎样把本地 WAL flush 等待从前台 commit 路径移到后台。

前置知识：已理解 WAL record 插入、record end LSN、WAL buffer、`XLogFlush()`、WAL-before-data、commit record 和 `synchronous_commit`。

本节唯一主问题：
当前台事务选择不等待本地 WAL flush 时，PostgreSQL 如何降低 commit latency，同时让“commit 已返回但 WAL 尚未 durable”的窗口保持可解释、可观测、可恢复？

本节核心矛盾：
前台 commit 希望避免每次都执行 write/fsync；但系统不能让 WAL、`pg_xact`、hint bit、同步复制和客户端承诺之间的顺序失去边界。

本节主流程：
`RecordTransactionCommit()` 的 async 分支记录 `asyncXactLSN`。
`XLogSetAsyncXactLSN()` 更新共享状态并可能唤醒 walwriter。
`WalWriterMain()` 周期性调用 `XLogBackgroundFlush()`。
`XLogBackgroundFlush()` 按完整 WAL page、async commit LSN、`wal_writer_delay` 和 `wal_writer_flush_after` 推进 `LogwrtResult.Write/Flush`。

本节必须区分四个事实：
WAL record 插入 WAL buffer，不等于 WAL 已经写到文件。
WAL 已经 write，不等于 WAL 已经 fsync。
async commit 返回客户端，不等于 commit record 已经 durable。
walwriter 后台 flush，不等于同步提交可以跳过 `XLogFlush(commit_lsn)`。

本节不泛泛介绍 WAL。
不重新讲 WAL record 格式。
不展开 checkpoint、archive、segment recycle 全流程。
这些内容只在解释 walwriter 边界时出现。

读完本节，你应该能回答：
- walwriter 每轮主循环推进什么状态。
- `XLogBackgroundFlush()` 为什么优先写完整 WAL page。
- async commit 为什么需要 `asyncXactLSN`。
- `XLogSetAsyncXactLSN()` 什么时候唤醒 walwriter。
- `wal_writer_delay` 和 `wal_writer_flush_after` 分别控制什么。
- `XLogFlush()` 和 walwriter 如何共享 `WALWriteLock`。
- `synchronous_commit = off` 跳过什么，没有跳过什么。
- `synchronous_commit = local/on/remote_write/remote_apply` 与 walwriter 的边界在哪里。
- WAL write/fsync 失败、walwriter ERROR、postmaster death 时系统如何收尾。
- 如何从 SQL、wait event、`pg_stat_io`、`pg_stat_wal` 和断点观察这条链。

## 源码基线

源码仓库：`/home/nail/postgres-lab`
基线：`master` = `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`

本节重点阅读：`src/backend/postmaster/walwriter.c`、`src/backend/access/transam/xlog.c`、`src/backend/access/transam/xlogwait.c`、`src/backend/access/transam/xloginsert.c`、`src/include/access/xlog.h`、`src/include/access/xlog_internal.h`。

辅助核对：`xact.c`、`transam.c`、`clog.c`、`slru.c`、`heapam_visibility.c`、`syncrep.c`、`xact.h`、`slru.h`、`syncrep.h`、`walwriter.h`、`guc_parameters.dat`、`wait_event_names.txt`。

行号来自 `nl -ba <source-file>`。

---

## 1. 先给结论

walwriter 是 WAL 后台写出进程。
它不是普通提交正确性的唯一执行者。
它的目标是减少 backend 自己 write/fsync WAL 的概率，并给 async commit 的未 flush 窗口一个上界。

`walwriter.c:5-15` 的文件头说明了这个定位。
walwriter 尝试让 regular backend 不必写出和 fsync WAL page。
它也保证没有在 commit 时立即 sync 的 commit record 会在可知时间内到达磁盘。
同一段注释强调：普通 backend 仍然可以自己写出和 fsync WAL。

后台主链路是：

```text
WalWriterMain()
  -> XLogBackgroundFlush()
       -> choose write target
       -> maybe choose flush target
       -> WaitXLogInsertionsToFinish()
       -> acquire WALWriteLock
       -> XLogWrite()
       -> WaitLSNWakeup(WAIT_LSN_TYPE_PRIMARY_FLUSH, LogwrtResult.Flush)
  -> pgstat_report_wal(false)
  -> WaitLatch(..., wal_writer_delay or hibernation timeout)
```

异步提交链路是：

```text
RecordTransactionCommit()
  -> XactLogCommitRecord()
  -> XLogSetAsyncXactLSN(XactLastRecEnd)
  -> TransactionIdAsyncCommitTree(..., XactLastRecEnd)
  -> return before local WAL flush is guaranteed
```

同步提交链路是：

```text
RecordTransactionCommit()
  -> XactLogCommitRecord()
  -> XLogFlush(XactLastRecEnd)
  -> TransactionIdCommitTree(...)
  -> optional SyncRepWaitForLSN(XactLastRecEnd, true)
```

核心不变量是：

```text
WAL inserted >= WAL written >= WAL flushed
```

`XLogInsert()` 推进 inserted frontier。
`XLogWrite()` 推进 write frontier。
`issue_xlog_fsync()` 成功后推进 flush frontier。
commit durability 关心 flush frontier。

`XLogBackgroundFlush()` 通常只写完整 WAL page。
如果完整 page target 已经 flush 到位，它会检查 `XLogCtl->asyncXactLSN`。
这让停在当前不完整 page 里的 async commit record 也能被后台写出。

`XLogBackgroundFlush()` 是否 fsync，由两个 GUC 控制。
`wal_writer_delay` 是时间阈值。
`wal_writer_flush_after` 是未 flush WAL block 的数量阈值。
默认 `wal_writer_delay = 200ms`。
默认 `wal_writer_flush_after = 1MB`，来自 `DEFAULT_WAL_WRITER_FLUSH_AFTER = (1024 * 1024) / XLOG_BLCKSZ`。

async commit 的窗口是：
客户端已收到 commit 成功。
commit record 已在 WAL buffer 中。
`pg_xact` 内存状态可能已标记 committed。
但 commit record 还没到达 `LogwrtResult.Flush`。
如果 PostgreSQL 或操作系统在这个窗口内崩溃，恢复后这个事务可能消失。
这是 `synchronous_commit = off` 的语义，而不是 bug。

这个窗口不是任意长。
`walwriter.c` 和 `XLogBackgroundFlush()` 注释说明 async commit 最多在三个 `wal_writer_delay` 周期内到达磁盘。
这个上界来自 walwriter 周期、完整 page 策略、`flexible` 写出可能多等一轮，以及 hibernation 时 async commit 会唤醒 walwriter。
真实系统仍会受调度、I/O stall、fsync 行为和故障影响。

最重要的边界是：
walwriter 可以提前把 WAL flush frontier 推过某个 commit LSN。
如果它已经做到，前台 `XLogFlush()` 会快速返回。
如果它没做到，同步提交 backend 必须自己 flush 或等待别的 backend flush。
walwriter 不能把 `synchronous_commit = on` 改造成“等后台以后再说”。

---

## 2. 核心文件分工与阅读顺序

先读 `src/backend/postmaster/walwriter.c`。
它回答 walwriter 何时启动、如何处理信号、如何从 ERROR 恢复、何时 hibernate、每轮调用什么。

再读 `src/backend/access/transam/xlog.c`。
重点块是 `XLogCtlData`、`XLogSetAsyncXactLSN()`、`XLogFlush()`、`XLogBackgroundFlush()`、`XLogWrite()` 和 `issue_xlog_fsync()`。

接着读 `src/backend/access/transam/xlogwait.c`。
当前基线里，主库 WAL flush 后会调用 `WaitLSNWakeup(WAIT_LSN_TYPE_PRIMARY_FLUSH, LogwrtResult.Flush)`。
这解释等待某个 primary flush LSN 的进程如何被唤醒。

再读 `src/backend/access/transam/xloginsert.c`。
本节只需要确认 `XLogInsert()` 返回 record end LSN，以及 `XLogInsertRecord()` 成功后更新 `XactLastRecEnd`。

然后读 `xlog.h` 和 `xlog_internal.h`。
前者暴露 `XLogFlush()`、`XLogBackgroundFlush()`、`XLogSetAsyncXactLSN()`、`GetFlushRecPtr()` 和 `SetWalWriterSleeping()`。
后者定义 WAL page、segment、文件名和 offset 宏。

最后辅助读 `xact.c`、`transam.c`、`clog.c`、`slru.c` 和 `heapam_visibility.c`。
这些文件解释 async commit 不是放弃 WAL-before-state，而是把 commit LSN 带到 `pg_xact` 和 hint bit 边界。

阅读顺序要围绕状态推进。
不要从 `XLogWrite()` 的系统调用细节开始。
本节首先要知道谁对哪个 LSN 作出承诺。

---

## 3. 关键状态与边界

### 3.1 `XactLastRecEnd`

`XactLastRecEnd` 是 backend-local 变量。
定义在 `xlog.c:260-262`。
`XLogInsertRecord()` 成功后设置：

```text
ProcLastRecPtr = StartPos
XactLastRecEnd = EndPos
```

普通写事务插入 commit record 后，`XactLastRecEnd` 就是 commit record end LSN。
`RecordTransactionCommit()` 用它调用 `XLogFlush()` 或 `XLogSetAsyncXactLSN()`。
它不是 durable LSN。
它必须和 `LogwrtResult.Flush` 比较才有持久化意义。

### 3.2 `LogwrtResult.Write` 与 `LogwrtResult.Flush`

`XLogwrtResult` 在 `xlog.c:332-336`。
`Write` 是 WAL 已写出的位置。
`Flush` 是 WAL 已 fsync 或等价持久化的位置。

每个进程有本地 `LogwrtResult` 缓存。
共享状态在 `XLogCtl->logWriteResult` 和 `XLogCtl->logFlushResult`。
`RefreshXLogWriteResult()` 先读 flush，再读屏障，再读 write。
`XLogWrite()` 发布结果时先写 write，再写屏障，再写 flush。
这个顺序保证读者不会看到 flush 超过 write 的组合。

### 3.3 `XLogCtl->LogwrtRqst`

`LogwrtRqst` 是共享 request。
字段是 `Write` 和 `Flush`。
它受 `info_lck` 保护。
它表示系统中已有进程请求 WAL write/flush 至少推进到某个位置。

WAL record 跨 page boundary 时，`XLogInsertRecord()` 会推进 `LogwrtRqst.Write`。
前台 `XLogFlush()` 也会读取它，顺手写出更多 WAL。
这使后台写出、前台 flush 和 group commit 能共享同一个 request frontier。

### 3.4 `XLogCtl->asyncXactLSN`

`asyncXactLSN` 保存最新 async commit 或 async abort LSN。
它受 `info_lck` 保护。
`XLogSetAsyncXactLSN()` 只在传入 LSN 更大时推进它。

这个字段不是事务队列。
它只保存最大 LSN。
因为 WAL 是顺序日志，flush 到最大 async LSN 自然覆盖更早 async commit record。

### 3.5 `XLogCtl->WalWriterSleeping`

`WalWriterSleeping` 表示 walwriter 可能处于低功耗 hibernation。
它受 `info_lck` 保护。
walwriter 在即将 hibernate 前设置。
async commit 看到它时会设置 walwriter 的 latch。

这个 flag 不表示 walwriter 是否健康。
也不保证唤醒后立即 fsync。
它只解决“不要让 async commit 等 hibernation 长睡眠自然结束”的问题。

### 3.6 `ProcGlobal->walwriterProc`

walwriter 在 `WalWriterMain()` 中发布自己的 `MyProcNumber`。
`XLogSetAsyncXactLSN()` 如果决定唤醒，就用这个 proc number 找到 `procLatch` 并 `SetLatch()`。

如果该值是 `INVALID_PROC_NUMBER`，说明没有可唤醒的 walwriter proc。
这不破坏 correctness。
普通 backend 仍然可以自己 flush WAL。

### 3.7 `WALWriteLock`

`WALWriteLock` 是 WAL buffer 写到 WAL file 的互斥边界。
walwriter 和普通 backend 都要拿它执行 `XLogWrite()`。
持锁期间可能发生 `pg_pwrite()` 和 fsync。

这把锁不是事务语义锁。
同步提交是否能返回，取决于目标 commit LSN 是否已经被 `LogwrtResult.Flush` 覆盖。

### 3.8 `WaitLSNState`

`xlogwait.c` 维护等待某类 LSN 到达的共享状态。
主库 flush wait 类型是 `WAIT_LSN_TYPE_PRIMARY_FLUSH`。
walwriter 或 backend flush 后调用 `WaitLSNWakeup()`。

这不是普通 `XLogFlush()` 的主等待方式。
普通 commit path 自己 flush 或等 `WALWriteLock`。
但这个设施是当前基线中观测 primary flush LSN wait 的重要连接点。

---

## 4. WalWriterMain 生命周期

`WalWriterMain()` 从 `walwriter.c:89` 开始。
它由 auxiliary process 框架启动。
进程类型是 `B_WAL_WRITER`。
`proctypelist.h` 中名字是 `walwriter`。

启动后先执行 `AuxiliaryProcessMainCommon()`。
然后安装信号处理。
`SIGHUP` 用于配置 reload。
`SIGTERM` 请求正常退出。
`SIGQUIT` 走 postmaster child 的紧急退出处理。
`SIGINT` 被忽略，因为 walwriter 没有用户查询可 cancel。

walwriter 创建自己的 memory context：

```text
AllocSetContextCreate(TopMemoryContext, "Wal Writer", ...)
```

这不是装饰。
顶层错误恢复会 reset 这个 context，避免后台进程长期运行时内存泄漏。

`walwriter.c:146-192` 是顶层 `sigsetjmp` 错误恢复。
遇到可恢复 ERROR，它会清理：
LWLocks。
condition variable sleep。
wait event。
AIO 错误状态。
buffer lock。
auxiliary process resources。
buffer、smgr、file、hash table 的 end-of-xact 状态。
然后 reset memory context，睡 1 秒再继续。

这条路径只处理可恢复 ERROR。
WAL write 和 WAL fsync 的严重错误通常是 PANIC，不会被 walwriter 主循环吞掉。

主循环前，walwriter 初始化：

```text
left_till_hibernate = LOOPS_UNTIL_HIBERNATE
hibernating = false
SetWalWriterSleeping(false)
ProcGlobal->walwriterProc = MyProcNumber
```

当前基线中：
`LOOPS_UNTIL_HIBERNATE = 50`。
`HIBERNATE_FACTOR = 25`。
默认 `wal_writer_delay = 200ms` 时，长时间无工作后的 timeout 约是 5 秒。

这就是 async commit 唤醒逻辑存在的原因。
如果 walwriter hibernate 后只等自然超时，async commit 到盘窗口可能被拉长。
所以 backend 会在必要时 `SetLatch()`。

---

## 5. WalWriterMain 主循环

主循环在 `walwriter.c:218-269`。
每轮的关键顺序是：

```text
maybe SetWalWriterSleeping(hibernating)
ResetLatch(MyLatch)
ProcessMainLoopInterrupts()
if XLogBackgroundFlush():
    left_till_hibernate = LOOPS_UNTIL_HIBERNATE
else:
    left_till_hibernate--
pgstat_report_wal(false)
WaitLatch(..., cur_timeout, WAIT_EVENT_WAL_WRITER_MAIN)
```

`SetWalWriterSleeping()` 放在 `ResetLatch()` 前。
源码注释解释，这是为了避免丢失 async commit 发来的 wakeup。
如果先 reset latch，后设置 sleeping flag，backend 可能错过唤醒时机。

`XLogBackgroundFlush()` 返回 true 表示这轮有 WAL work。
即使最后发现别人已经替它写出或 flush，它也返回 true。
这样 walwriter 不会过早进入 hibernation。

`pgstat_report_wal(false)` 会上报 WAL usage 和 IO stats。
`false` 表示尽量不阻塞在统计锁上。
walwriter 主循环不能因为统计 flush 变成新的瓶颈。

`WaitLatch()` 使用：
`WL_LATCH_SET`。
`WL_TIMEOUT`。
`WL_EXIT_ON_PM_DEATH`。
等待事件是 `WAIT_EVENT_WAL_WRITER_MAIN`。

`WAL_WRITER_MAIN` 通常表示 walwriter 在主循环 sleep。
它不是 WAL flush 慢的直接证据。
诊断 WAL I/O 要看 `WAL_WRITE`、`WAL_SYNC` 和 `pg_stat_io`。

---

## 6. XLogBackgroundFlush 的目标选择

`XLogBackgroundFlush()` 在 `xlog.c:3003`。
它的注释说：
后台写出和 flush WAL，但不指定精确位置。
这正是它和 `XLogFlush(record)` 的差异。

第一步：

```text
if (RecoveryInProgress())
    return false
```

walwriter 只负责主库正常运行时本地生成 WAL 的后台写出。

第二步，读取当前 timeline。
因为不在 recovery，`InsertTimeLineID` 已设置且不会变化，可以无锁读。

第三步，读取共享 write request：

```text
SpinLockAcquire(&XLogCtl->info_lck)
WriteRqst = XLogCtl->LogwrtRqst
SpinLockRelease(&XLogCtl->info_lck)
```

第四步，把 write target 回退到完整 WAL page 边界：

```text
WriteRqst.Write -= WriteRqst.Write % XLOG_BLCKSZ
```

这说明 walwriter 平时优先写完整 page。
当前 page 的半条或尾部 WAL record 不会因为后台周期醒来就立即被写。
这样可以减少小写和无效 fsync。

第五步，刷新本地 `LogwrtResult`。
如果完整 page target 已经 flush 到位，就检查 async commit：

```text
if (WriteRqst.Write <= LogwrtResult.Flush)
{
    WriteRqst.Write = XLogCtl->asyncXactLSN;
    flexible = false;
}
```

这段是 async WAL flush 的核心。
低 WAL 量系统中，commit record 可能停在不完整 WAL page。
如果 walwriter 只写完整 page，async commit record 可能长时间不到盘。
所以当完整 page 已经没有工作时，walwriter 会用 `asyncXactLSN` 作为 write target。

`flexible = false` 表示必须写到目标位置。
普通完整 page 写出可以 `flexible = true`。
`XLogWrite()` 在 flexible 模式下可以在方便边界停下。
async commit target 不能这样，否则可能仍没写到 commit record。

如果 async LSN 也不超过当前 flush position，说明没有工作。
walwriter 会顺手关闭不再需要的 open WAL segment fd，然后返回 false。
这避免后台进程持有旧 segment 文件句柄影响删除或回收。

---

## 7. XLogBackgroundFlush 的 flush 决策

有 write target 后，函数决定本轮是否 flush。
计算：

```text
flushblocks =
    WriteRqst.Write / XLOG_BLCKSZ
  - LogwrtResult.Flush / XLOG_BLCKSZ
```

决策顺序是：

```text
if WalWriterFlushAfter == 0 or lastflush == 0:
    WriteRqst.Flush = WriteRqst.Write
else if now - lastflush > WalWriterDelay:
    WriteRqst.Flush = WriteRqst.Write
else if flushblocks >= WalWriterFlushAfter:
    WriteRqst.Flush = WriteRqst.Write
else:
    WriteRqst.Flush = InvalidXLogRecPtr
```

`wal_writer_delay` 是时间阈值。
超过这个间隔，walwriter 至少 flush 一次。
它是 async commit 到盘窗口的关键参数。

`wal_writer_flush_after` 是数量阈值。
未 flush WAL block 足够多时，提前 fsync。
设置为 0 会禁用 block-based 延迟，walwriter 有 work 时直接把 flush target 设到 write target。

如果本轮不 flush，`WriteRqst.Flush = InvalidXLogRecPtr`。
`XLogWrite()` 会写 WAL，但不会执行 `issue_xlog_fsync()`。
这降低 fsync 频率，也让 async commit 的 durable 时间滞后于 commit 返回。

---

## 8. XLogBackgroundFlush 的执行路径

执行写出前，walwriter 进入 critical section：

```text
START_CRIT_SECTION()
WaitXLogInsertionsToFinish(WriteRqst.Write)
LWLockAcquire(WALWriteLock, LW_EXCLUSIVE)
RefreshXLogWriteResult(LogwrtResult)
if target still ahead:
    XLogWrite(WriteRqst, insertTLI, flexible)
LWLockRelease(WALWriteLock)
END_CRIT_SECTION()
```

`WaitXLogInsertionsToFinish()` 必须在 `WALWriteLock` 之前调用。
源码注释说这是为了避免 deadlock。
正在插入 WAL 的 backend 可能需要回收 WAL buffer。
回收脏 WAL buffer 又可能需要 WAL write。
如果 walwriter 拿着 `WALWriteLock` 等插入者，而插入者等 `WALWriteLock`，就可能死锁。

`WaitXLogInsertionsToFinish(upto)` 返回一个位置。
这个位置之前的 WAL 已经完整拷贝进 WAL buffer，可以写出。
它通过 WAL insertion lock 的 `insertingAt` 判断哪些旧插入仍在进行。

拿到 `WALWriteLock` 后还要刷新 `LogwrtResult`。
因为等待插入和等锁期间，别的 backend 可能已经完成 write/flush。
如果目标已经满足，就不重复写。

写出后，walwriter 唤醒 walsender：

```text
WalSndWakeupProcessRequests(true, !RecoveryInProgress())
```

然后唤醒等待 primary flush LSN 的进程：

```text
WaitLSNWakeup(WAIT_LSN_TYPE_PRIMARY_FLUSH, LogwrtResult.Flush)
```

最后调用 `AdvanceXLInsertBuffer(InvalidXLogRecPtr, insertTLI, true)`。
这会尝试初始化未来可用的 WAL buffers，把一部分工作从前台 insert path 移走。

---

## 9. XLogWrite 的 I/O 边界

`XLogWrite()` 在 `xlog.c:2324`。
调用者必须持有 `WALWriteLock`。
调用者还必须先保证目标之前的 WAL insertion 已完成。

`XLogWrite()` 推进 `LogwrtResult.Write`。
它尽量把连续 WAL buffer page 合并成较少的 `pg_pwrite()`。
等待事件是 `WAL_WRITE`。
IO 统计进入 `pg_stat_io` 的 `object = 'wal'`、`op = write`。

写 WAL 文件失败时，除 `EINTR` 重试外，会 PANIC。
错误信息包括 WAL file name、offset 和 length。
WAL write 失败不能只让当前事务 ERROR 后继续运行。
共享 WAL 状态和后续 durability 承诺已经不可信。

如果写到 segment 末尾，`XLogWrite()` 会立即 `issue_xlog_fsync()`。
这样以后不必为了 fsync 旧 segment 再打开文件。
它还会通知 archiver、更新 segment switch LSN，并在 WAL 消耗过多时请求 checkpoint。

如果 request 中 `WriteRqst.Flush` 有效，并且当前 flush position 落后于 write position，`XLogWrite()` 也会调用 `issue_xlog_fsync()`。

`issue_xlog_fsync()` 根据 `wal_sync_method` 选择 `fsync`、`fdatasync` 或 write-through 方法。
等待事件是 `WAL_SYNC`。
IO 统计进入 `pg_stat_io` 的 `op = fsync`。

如果 `enableFsync` 关闭，或者 `wal_sync_method` 是 `open`/`open_datasync`，`issue_xlog_fsync()` 会 quick exit。
语义依赖配置和同步写方式。

fsync 失败是 PANIC。
源码注释明确写着 `PANIC if failed to fsync`。

最后 `XLogWrite()` 更新共享状态。
它把 `XLogCtl->LogwrtRqst.Write/Flush` 至少推进到当前 result。
再按 write-barrier-flush 顺序更新 atomic result。
所有 backend、walwriter、SQL 函数和 LSN wait 都依赖这个发布顺序。

---

## 10. async commit 从哪里进入

async commit 的入口在 `RecordTransactionCommit()`。
普通写事务先插入 commit record。
`XactLogCommitRecord()` 最终调用 `XLogInsert(RM_XACT_ID, info)`。
底层成功后 `XactLastRecEnd` 指向 commit record end LSN。

提交路径的核心条件是：

```text
if ((wrote_xlog && markXidCommitted &&
     synchronous_commit > SYNCHRONOUS_COMMIT_OFF) ||
    forceSyncCommit || nrels > 0)
{
    XLogFlush(XactLastRecEnd);
    TransactionIdCommitTree(...);
}
else
{
    XLogSetAsyncXactLSN(XactLastRecEnd);
    TransactionIdAsyncCommitTree(..., XactLastRecEnd);
}
```

`wrote_xlog` 表示事务写过 WAL。
`markXidCommitted` 表示有 XID 要标记 committed。
`forceSyncCommit` 表示当前命令要求强制同步。
`nrels > 0` 表示有非临时 relation 文件待删除。

有 pending relation deletes 时不能 async commit。
如果 commit record 未 flush，系统却先删除文件，崩溃恢复可能无法解释文件状态。
所以这类事务必须立即 `XLogFlush()`。

`synchronous_commit = off` 只跳过当前 backend 对 commit LSN 的本地 flush 等待。
它不跳过 commit record 插入。
它不跳过 `pg_xact` 状态更新。
它不允许状态页或 hint bit 无边界地越过 commit WAL。

---

## 11. XLogSetAsyncXactLSN

`XLogSetAsyncXactLSN()` 在 `xlog.c:2630`。
注释说它记录 asynchronous transaction commit/abort 的 LSN，并在有工作时 nudge walwriter。
它不应该用于 synchronous commits。

流程是：

```text
WriteRqstPtr = asyncXactLSN
lock info_lck
sleeping = XLogCtl->WalWriterSleeping
prevAsyncXactLSN = XLogCtl->asyncXactLSN
if old < new:
    XLogCtl->asyncXactLSN = new
unlock info_lck

if new <= prev:
    return

if sleeping:
    wakeup = true
else:
    RefreshXLogWriteResult(LogwrtResult)
    flushblocks = new / XLOG_BLCKSZ - LogwrtResult.Flush / XLOG_BLCKSZ
    if WalWriterFlushAfter == 0 or flushblocks >= WalWriterFlushAfter:
        wakeup = true

if wakeup and walwriterProc valid:
    SetLatch(walwriter latch)
```

如果另一个 backend 已登记更靠后的 async LSN，本 backend 直接返回。
因为 flush 到更靠后的 LSN 自然包含当前 LSN。

唤醒条件有两个。
第一，walwriter 处于 sleeping/hibernation。
第二，未 flush WAL block 已达到 `wal_writer_flush_after`。
否则让 walwriter 按正常 `wal_writer_delay` 周期醒来。

`XLogSetAsyncXactLSN()` 不调用 `XLogWrite()`。
不拿 `WALWriteLock`。
不等待 fsync。
它只是把后台需要推进到哪里写进共享状态。

---

## 12. async commit 与 pg_xact

异步提交最容易误解的点是：
commit record 没有 flush，为什么可以标记 `pg_xact` committed？

答案是：
内存中可以先标记。
但任何可能把这个状态变成持久化事实的路径，都必须携带 commit LSN 检查 WAL flush。

`TransactionIdAsyncCommitTree()` 调用：

```text
TransactionIdSetTreeStatus(...,
                           TRANSACTION_STATUS_COMMITTED,
                           lsn)
```

这个 `lsn` 是 commit record end LSN。

`clog.c` 的 `TransactionIdSetStatusBit()` 收到有效 LSN 时，会更新 `pg_xact` SLRU slot 的 `group_lsn[]`。
`group_lsn[]` 不是每个 XID 一个 LSN。
它按 page 内 group 保存该组事务状态需要的最大 WAL flush LSN。

`slru.h` 对 `group_lsn[]` 的注释说：
如果不为 NULL，写 SLRU page 前必须 flush WAL。
这个机制对 `pg_xact` 为 true，对其他 SLRU 不一定存在。

`slru.c` 的 `SlruPhysicalWritePage()` 写 page 前，会取该 page 所有 group 的最大 LSN。
如果有效，就在 critical section 中执行：

```text
XLogFlush(max_lsn)
```

然后才写 SLRU page。

所以 async commit 允许客户端早返回。
但它不允许 `pg_xact` page 先于 commit WAL 安全落盘。

hint bit 也有边界。
`heapam_visibility.c` 的 `SetHintBitsExt()` 在设置 committed hint bit 前，会取 `TransactionIdGetCommitLSN(xid)`。
如果 `XLogNeedsFlush(commitLSN)`，且 data page LSN 小于 commit LSN，它不会设置 hint bit。

原因是 hint bit 也可能把“这个 XID 已提交”的事实持久化到 heap page。
如果 commit WAL 未 flush，就不能让 data page 先携带这个事实落盘。

---

## 13. 前台 XLogFlush

`XLogFlush(record)` 在 `xlog.c:2801`。
它和 walwriter 的差别是：
它有明确目标 LSN。
调用者要求 WAL 至少 flush 到 `record`。

普通路径先 quick exit：

```text
if (record <= LogwrtResult.Flush)
    return
```

如果本地缓存不够新，后面会刷新共享 atomic result。

接着进入 critical section，并循环：

```text
RefreshXLogWriteResult(LogwrtResult)
if record <= LogwrtResult.Flush:
    break

read XLogCtl->LogwrtRqst.Write
insertpos = WaitXLogInsertionsToFinish(WriteRqstPtr)

if cannot acquire WALWriteLock immediately:
    wait for release
    loop and recheck

RefreshXLogWriteResult(LogwrtResult)
if record <= LogwrtResult.Flush:
    release lock
    break

maybe wait CommitDelay
WriteRqst.Write = insertpos
WriteRqst.Flush = insertpos
XLogWrite(WriteRqst, insertTLI, false)
```

`LWLockAcquireOrWait()` 支持 group commit。
如果另一个 backend 正在 flush，当前 backend 先等锁释放，再检查自己的 target 是否已被覆盖。
如果已覆盖，就不再抢锁执行 fsync。

`CommitDelay` 只在拿到 `WALWriteLock` 后发生。
条件是 `CommitDelay > 0`、`enableFsync` 打开、并且活跃事务数达到 `commit_siblings`。
等待事件是 `COMMIT_DELAY`。
它用更高单事务延迟换取更多 backend 共享一次 fsync 的机会。

`XLogFlush()` 完成后会唤醒 walsender 和 primary flush LSN waiters。
然后检查 `LogwrtResult.Flush < record`。
如果仍不满足，报 ERROR。
源码注释说明，来自 `xact.c` commit critical section 的 ERROR 会被提升为 PANIC。
来自坏 data page LSN 的调用不一定导致整个系统重启。

---

## 14. synchronous_commit 边界

`xact.h` 中的枚举是：

```text
SYNCHRONOUS_COMMIT_OFF
SYNCHRONOUS_COMMIT_LOCAL_FLUSH
SYNCHRONOUS_COMMIT_REMOTE_WRITE
SYNCHRONOUS_COMMIT_REMOTE_FLUSH
SYNCHRONOUS_COMMIT_REMOTE_APPLY
```

默认 `SYNCHRONOUS_COMMIT_ON` 等于 `SYNCHRONOUS_COMMIT_REMOTE_FLUSH`。

在 `RecordTransactionCommit()` 中，只要：
事务写了 WAL。
事务有 XID。
`synchronous_commit > SYNCHRONOUS_COMMIT_OFF`。
就会先执行本地 `XLogFlush(XactLastRecEnd)`。

所以 `local`、`on`、`remote_write`、`remote_apply` 都先等待本地 flush。
差异在是否继续等同步备库。

`SyncRepRequested()` 是：

```text
max_wal_senders > 0
and synchronous_commit > SYNCHRONOUS_COMMIT_LOCAL_FLUSH
```

还要有同步备库配置。
否则 `SyncRepWaitForLSN()` 会快速返回。

`remote_write` 里的 write 是备库侧 write。
它不是主库本地只 write 不 fsync。
主库进入同步复制等待前，本地已经完成 `XLogFlush()`。

没有同步备库时，`synchronous_commit = on` 实际主要等待本地 flush。
有同步备库时，前台 commit 在本地 flush 和 `pg_xact` 标记后继续等待同步复制确认。

`syncrep.c` 中同步复制等待期间如果收到 cancel 或 terminate，不能直接 ERROR。
因为事务已经本地 committed。
源码会发 WARNING，取消等待并收尾。
这说明 commit path 的客户端语义不能由后台 walwriter 来模糊处理。

---

## 15. 正确性机制层次

第一层是 WAL insertion。
`XLogInsert()` 把 record 放入 WAL buffer 并返回 end LSN。
它不代表 durable。

第二层是 WAL write/flush。
`XLogWrite()` 在 `WALWriteLock` 下推进 write/flush frontier。
`LogwrtResult.Flush` 是本地持久化判断的核心。

第三层是事务提交策略。
`synchronous_commit = off` 走 async 分支。
其它同步级别先本地 `XLogFlush()`。
`forceSyncCommit` 和非临时 relation delete 会强制同步。

第四层是 `pg_xact`。
async commit 把 commit LSN 传给 `TransactionIdAsyncCommitTree()`。
`pg_xact` page 写出前用 `group_lsn[]` 反向要求 WAL flush。

第五层是 heap hint bit。
设置 committed hint bit 前要检查 commit LSN。
这避免 data page 把 committed 事实持久化到 commit WAL 之前。

第六层是 replication。
walwriter 和 backend 推进 local flush。
walsender 和 `SyncRepWaitForLSN()` 推进 remote write/flush/apply。
两条 frontier 相关，但不是同一个状态。

这也是 walwriter 不能替代同步 commit 的原因。
walwriter 推进的是全局后台 flush frontier。
同步 commit 要求当前调用路径证明自己的 commit LSN 已经被 durable frontier 覆盖。

---

## 16. 错误路径与 fallback

walwriter 顶层 ERROR：
释放 LWLocks。
取消 condition variable sleep。
结束 wait event。
清理 AIO、buffer、smgr、file、hash table 等资源。
reset walwriter memory context。
睡 1 秒后继续。

WAL write 失败：
`XLogWrite()` 中 `pg_pwrite()` 失败时，除 `EINTR` 重试外，PANIC。
这会触发 postmaster 杀掉其他进程并恢复。

WAL fsync 失败：
`issue_xlog_fsync()` 中 fsync/fdatasync 失败也是 PANIC。
错误信息包含 WAL segment 文件名。

flush request 不满足：
`XLogFlush()` 最后检查目标是否满足。
如果 `LogwrtResult.Flush < record`，报 ERROR。
commit critical section 中的 ERROR 会升级为系统级失败。

postmaster death：
walwriter `WaitLatch()` 使用 `WL_EXIT_ON_PM_DEATH`。
`xlogwait.c` 中等待 LSN 的进程遇到 postmaster death 会 FATAL。
同步复制等待会取消等待并避免把本地已提交事务报告成 abort。

walwriter 跟不上：
这不是错误。
普通 backend 仍然可以自己 `XLogFlush()`。
可见结果是业务 backend 的 WAL write/fsync 增加，commit latency 上升。

walwriter 意外退出：
文件头说明 postmaster 会按 backend crash 处理。
因为共享内存可能损坏，剩余 backend 会被 SIGQUIT，然后进入恢复。

---

## 17. 成本、资源与传播

成本一：fsync 频率。
`wal_writer_delay` 越小，async commit 到盘越快，但后台 fsync 可能更频繁。
`wal_writer_flush_after` 越小，按量触发越早，也可能增加 I/O 干扰。

成本二：`WALWriteLock` 竞争。
walwriter 和 backend 都要拿这把锁执行 `XLogWrite()`。
慢 fsync、高并发 commit、WAL buffer 压力都会把等待暴露到前台。

成本三：等待 WAL insertion 完成。
写 WAL 前必须 `WaitXLogInsertionsToFinish()`。
这不是性能装饰，而是避免写出尚未拷贝完成的 WAL bytes。

成本四：I/O 合并与尾延迟。
完整 page 写出和延迟 fsync 能减少小写。
代价是 async commit durable 时间滞后于客户端返回。

成本五：前台 fallback。
如果 walwriter 没提前推进 flush frontier，同步 commit backend 会自己 flush。
所以性能分析不能只看 walwriter。

跨模块传播：
WAL insertion 产生 LSN。
事务提交决定同步或异步。
walwriter 或 backend 推进 local flush。
`pg_xact` 和 hint bit 通过 commit LSN 维护 WAL-before-state。
walsender 基于 local WAL 进度推进复制。
checkpoint 基于 WAL-before-data 和 data file fsync 发布恢复起点。

---

## 18. 观测与诊断入口

### 18.1 insert/write/flush 差距

主库上执行：

```sql
SELECT
  pg_current_wal_insert_lsn() AS insert_lsn,
  pg_current_wal_lsn() AS write_lsn,
  pg_current_wal_flush_lsn() AS flush_lsn,
  pg_wal_lsn_diff(pg_current_wal_insert_lsn(),
                  pg_current_wal_flush_lsn()) AS insert_flush_gap;
```

`pg_current_wal_insert_lsn()` 是 insert frontier。
`pg_current_wal_lsn()` 当前基线中是 write location。
源码注释说明它不一定 synced。
`pg_current_wal_flush_lsn()` 是 flush frontier。

这个查询是瞬时观察。
它不能证明某个具体事务是不是 async commit。
它能展示 WAL insert 与 flush 是否存在 backlog。

### 18.2 `pg_stat_wal`

当前基线的 `pg_stat_wal` 列是：

```text
wal_records
wal_fpi
wal_bytes
wal_fpi_bytes
wal_buffers_full
stats_reset
```

它主要描述 WAL 生成侧。
`wal_buffers_full` 升高说明 WAL buffer 压力可能把写出工作推回前台。
它不是 walwriter 慢的完整证据。

当前基线里 WAL write/fsync 次数和时间不在 `pg_stat_wal`。
不要按旧版本经验写不存在的列。

### 18.3 `pg_stat_io`

看 WAL I/O：

```sql
SELECT backend_type, object, context,
       writes, write_bytes, write_time,
       fsyncs, fsync_time
FROM pg_stat_io
WHERE object = 'wal'
ORDER BY backend_type, context;
```

重点比较 `backend_type = 'walwriter'` 和普通 backend。
如果普通 backend 的 WAL fsync 很多，说明很多 flush 回到了前台路径。
如果 walwriter 的 WAL write/fsync 多，说明后台承担了更多推进工作。

时间列依赖 `track_wal_io_timing`。
如果关闭，主要看计数和 bytes。

### 18.4 wait event

相关 wait event：

```text
WAL_WRITER_MAIN      walwriter 主循环 sleep
WAL_WRITE            WAL 文件 write
WAL_SYNC             WAL 文件 fsync/fdatasync
COMMIT_DELAY         group commit 前人为等待
WAIT_FOR_WAL_FLUSH   等待某类 WAL flush LSN 到达
SYNC_REP             等待同步复制确认
XACT_GROUP_UPDATE    等待 group leader 更新 pg_xact
```

`WAL_WRITER_MAIN` 通常表示 walwriter idle。
不要把它当成 WAL 卡住。
要判断 commit latency，看业务 backend 是否在 `WAL_WRITE`、`WAL_SYNC`、`COMMIT_DELAY` 或 `SYNC_REP`。

### 18.5 gdb 断点

建议断点：`WalWriterMain`、`XLogBackgroundFlush`、`XLogSetAsyncXactLSN`、`XLogFlush`、`XLogWrite`、`issue_xlog_fsync`、`WaitLSNWakeup`。
关键变量：`XLogCtl->LogwrtRqst.Write/Flush`、`XLogCtl->asyncXactLSN`、`XLogCtl->WalWriterSleeping`、`LogwrtResult.Write/Flush`、`WriteRqst.Write/Flush`、`flexible`。
断点会改变调度和时间。
它适合验证状态变化，不适合测量 async commit 真实延迟。

---

## 19. 常见误区

误区一：
`XLogInsert()` 返回就表示 WAL durable。
实际只表示 record 插入 WAL buffer，并得到 end LSN。

误区二：
walwriter 负责所有 WAL fsync。
实际普通 backend 也可以进入 `XLogFlush()` 和 `XLogWrite()`。

误区三：
`synchronous_commit = off` 表示不写 commit WAL。
实际 commit record 仍写入 WAL buffer。
跳过的是当前 backend 的本地 flush 等待。

误区四：
async commit 返回后，`pg_xact` 状态可以无约束落盘。
实际 `group_lsn[]` 会让 SLRU 写页前 flush commit WAL。

误区五：
`wal_writer_delay` 调小没有代价。
它可能缩短 async commit 窗口，也可能提高 fsync 频率和 I/O 干扰。

误区六：
`WAL_WRITER_MAIN` 表示 WAL writer 忙。
实际它是主循环等待事件，通常表示 sleep。

误区七：
`pg_current_wal_lsn()` 是 flush LSN。
当前基线中它是 write location。
flush location 要看 `pg_current_wal_flush_lsn()`。

误区八：
`remote_write` 表示主库本地只 write 不 fsync。
不是。
主库先本地 `XLogFlush()`，再等备库 write。

---

## 20. 课堂实验一：观察 async commit 的 flush gap

目标：
观察 `synchronous_commit = off` 下 insert frontier 和 flush frontier 的短暂差距。

准备测试表：`CREATE TABLE IF NOT EXISTS walwriter_async_demo(id bigint generated always as identity, payload text);`

观测窗口反复查 `pg_current_wal_insert_lsn()`、`pg_current_wal_lsn()`、`pg_current_wal_flush_lsn()` 和 `pg_wal_lsn_diff(pg_current_wal_insert_lsn(), pg_current_wal_flush_lsn())`。

负载窗口用 autocommit 反复执行 `SET synchronous_commit = off; INSERT INTO walwriter_async_demo(payload) VALUES (repeat('x', 200));`。

观察 `insert_lsn` 是否短暂领先 `flush_lsn`。
差距会被 walwriter 或其他 backend 缩小。

回到源码：
commit path 调用 `XLogSetAsyncXactLSN()`。
后台 `XLogBackgroundFlush()` 后续把 `asyncXactLSN` 纳入 write target。

---

## 21. 课堂实验二：区分 walwriter 与 backend WAL I/O

目标：
判断 WAL write/fsync 是后台 walwriter 做的，还是业务 backend 自己做的。

先执行 `SELECT pg_stat_reset_shared('io');` 和 `SELECT pg_stat_reset_shared('wal');`。

运行 async 负载：

```sql
SET synchronous_commit = off;
INSERT INTO walwriter_async_demo(payload)
SELECT repeat('a', 100)
FROM generate_series(1, 100000);
```

查看 `pg_stat_wal` 的 WAL 生成量，再查 `pg_stat_io` 中 `object = 'wal'` 的 `backend_type`、`writes`、`write_bytes`、`fsyncs`、`write_time`、`fsync_time`。

再把 `synchronous_commit` 设为 `on`，运行同等规模 INSERT。

对比普通 backend 和 walwriter 的 WAL I/O。
如果同步提交时普通 backend fsync 增多，说明前台 `XLogFlush()` 正在承担更多工作。

---

## 22. 课堂实验三：断点跟读 async commit 唤醒

目标：
确认 async commit 不直接 flush，而是登记 LSN 并可能唤醒 walwriter。

建议断点：`XLogSetAsyncXactLSN`、`XLogBackgroundFlush`、`XLogWrite`、`issue_xlog_fsync`。

执行：

```sql
SET synchronous_commit = off;
INSERT INTO walwriter_async_demo(payload) VALUES ('one async commit');
```

在 `XLogSetAsyncXactLSN()` 看 `asyncXactLSN`、`XLogCtl->asyncXactLSN`、`XLogCtl->WalWriterSleeping`、`WalWriterFlushAfter`、`LogwrtResult.Flush`、`wakeup`。
在 walwriter 的 `XLogBackgroundFlush()` 看 `WriteRqst.Write/Flush`、`flexible`、`lastflush`、`flushblocks`。

预期：
async commit backend 不进入 `XLogWrite()`。
walwriter 后续进入 `XLogBackgroundFlush()`。
如果完整 page target 已经 flush，它会使用 `asyncXactLSN` 并设置 `flexible = false`。

---

## 23. 讨论题

1. 为什么 walwriter 通常只写完整 WAL page，但 async commit 会让它写当前不完整 page？
2. `asyncXactLSN` 为什么保存最大 LSN 就够了？
3. 如果 walwriter 已经 hibernating，为什么 async commit 必须通过 latch 唤醒它？
4. `XLogSetAsyncXactLSN()` 为什么不直接调用 `XLogWrite()`？
5. `synchronous_commit = on` 在没有同步备库时，为什么仍要本地 `XLogFlush()`？
6. `remote_write` 为什么不是主库只 write 不 fsync？
7. async commit 已经把 `pg_xact` 标记 committed，为什么 `pg_xact` page 仍不能越过 commit WAL 安全落盘？
8. `SetHintBitsExt()` 为什么要看 commit LSN？
9. WAL write/fsync 失败为什么是 PANIC？
10. 看到 `WAL_WRITER_MAIN`，你能判断什么，不能判断什么？
11. 如果普通 backend 的 WAL fsync 很多，而 walwriter 很少，可能有哪些解释？
12. `wal_writer_delay` 调小会改善哪些现象，又可能引入哪些代价？

---

## 24. 本节小结

本节核心链路是：
commit record 先通过 `XLogInsert()` 进入 WAL buffer。
同步提交走 `XLogFlush(XactLastRecEnd)`。
异步提交走 `XLogSetAsyncXactLSN(XactLastRecEnd)`。
walwriter 在 `WalWriterMain()` 周期中调用 `XLogBackgroundFlush()`，按完整 page、async LSN、时间阈值和 block 阈值推进 WAL write/flush。

核心状态是：
`XactLastRecEnd` 表示当前事务最近 WAL record end。
`LogwrtResult.Write` 表示已写出位置。
`LogwrtResult.Flush` 表示已持久化位置。
`LogwrtRqst` 表示共享 request frontier。
`asyncXactLSN` 表示最新 async commit/abort LSN。
`WalWriterSleeping` 表示 walwriter 可能需要被唤醒。

正确性边界是：
walwriter 可以减少前台 WAL I/O。
walwriter 不能替代同步提交的 `XLogFlush()`。
async commit 允许客户端返回早于 WAL flush。
但 `pg_xact` SLRU 写页、hint bit 设置、data page 写出仍必须维护 WAL-before-state 和 WAL-before-data。

错误路径是：
walwriter 顶层 ERROR 会清理资源并继续。
WAL write 和 WAL fsync 失败会 PANIC。
flush target 不满足会 ERROR，在 commit critical section 中会升级为更严重的系统失败。
postmaster death 会让等待者和后台进程按各自边界退出。

观测入口是：
用 `pg_current_wal_insert_lsn()`、`pg_current_wal_lsn()`、`pg_current_wal_flush_lsn()` 看 frontier 差距。
用 `pg_stat_wal` 看 WAL 生成量。
用 `pg_stat_io` 看 WAL write/fsync 按 backend_type 的分布。
用 `pg_stat_activity` wait event 区分 `WAL_WRITE`、`WAL_SYNC`、`COMMIT_DELAY`、`WAIT_FOR_WAL_FLUSH`、`SYNC_REP` 和 `WAL_WRITER_MAIN`。
用 gdb 断点回到 `XLogSetAsyncXactLSN()`、`XLogBackgroundFlush()`、`XLogFlush()` 和 `XLogWrite()`。

可迁移规律：
后台 flush 进程能降低前台延迟，但不能自动替代前台承诺。
只要系统对外承诺 durable boundary，调用路径就必须证明自己的 LSN 已被持久化 frontier 覆盖。
如果选择异步返回，就必须把风险窗口、状态页写出、可见性优化和观测指标分层管理。

仍然依赖环境的判断包括：
async commit 实际窗口受 `wal_writer_delay`、`wal_writer_flush_after`、I/O 延迟、调度和负载影响。
commit latency 是否改善取决于事务大小、并发度、fsync 成本、WAL buffer 压力和同步复制配置。
`pg_stat_*` 只能给出聚合和瞬时观测。
解释单次 commit 尾延迟时，通常还要结合 wait event、客户端计时、I/O timing、断点或 profiling。
