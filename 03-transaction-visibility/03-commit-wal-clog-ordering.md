# PostgreSQL 提交顺序、WAL 与 CLOG 持久化边界

## 课程定位

前置知识：

- 你已经知道 PostgreSQL 使用 MVCC tuple version 表达可见性。
- 你已经知道 tuple header 中有 `xmin`、`xmax`、hint bit。
- 你已经知道 WAL 的基本规则是 write-ahead logging。
- 你已经知道事务 ID 会进入 ProcArray，snapshot 会读取 running XID 集合。

本节唯一主问题：

- 一个事务执行 `COMMIT` 时，PostgreSQL 如何安排 “提交返回”、“对其他事务可见”、“commit WAL 持久化” 与 “CLOG/pg_xact 状态持久化” 的顺序？

本节核心矛盾：

- 提交路径希望低延迟、高并发、少 fsync、少锁竞争。
- crash recovery 又要求系统不能在重启后把一个没有持久 commit WAL 的事务当成 committed。
- 这两个目标不能靠一个简单的 “先写 WAL、再写 CLOG、最后返回” 线性流程完全满足。

学完后你应该能独立判断：

- 为什么同步提交和异步提交都可以更新内存中的 CLOG，但持久边界不同。
- 为什么 CLOG 页写盘前还要检查 WAL flush LSN。
- 为什么 commit record 的 LSN 不是 “已经持久” 的同义词。
- 为什么事务从 ProcArray 消失必须发生在 CLOG 更新之后。
- 为什么 hint bit 设置也会受到 commit WAL flush 状态约束。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

术语说明：

- 源码文件和注释仍常用 `CLOG`。
- 数据目录中的事务状态文件今天叫 `pg_xact`。
- 本课用 “CLOG/pg_xact” 表示同一类事务状态持久化机制。
- 本课不把 2PC、logical decoding、standby replay 展开成主线。
- 它们只在能解释普通一阶段提交边界时出现。

## 1. 本节在总主线中的位置

上一节你已经看到 tuple 可见性不是只看 tuple header。`xmin` 和 `xmax` 只是指向事务状态的引用。

真正回答 “这个写入是否提交” 的，是事务系统维护的状态。这节课把问题推进到提交瞬间：

- 一个 backend 说自己提交了。
- WAL buffer 里可能已经有 commit record。
- WAL 文件可能还没有 fsync。
- CLOG buffer 里可能已经有 committed bit。
- CLOG 文件可能还没有写盘。
- 这个 backend 可能还没有从 ProcArray 消失。
- 其他 backend 可能已经在做可见性判断。

这些状态不是同一时刻改变。PostgreSQL 故意把它们拆开。

拆开的目的不是让实现更复杂。拆开的目的是让 hot path 不必把所有持久化工作都塞进 `COMMIT` 返回前。

但拆开之后就必须有新的不变量。本节只追一条链路：

```text
SQL COMMIT
  -> CommitTransaction()
  -> RecordTransactionCommit()
  -> XactLogCommitRecord()
  -> XLogInsert()
  -> XLogFlush() 或 XLogSetAsyncXactLSN()
  -> TransactionIdCommitTree() 或 TransactionIdAsyncCommitTree()
  -> ProcArrayEndTransaction()
  -> post-commit cleanup
```

这条链路贯穿本节所有章节。我们不会把 WAL insertion、SLRU、ProcArray、hint bit 分别讲成四节独立材料。

它们都只服务一个问题：

- commit WAL 与 CLOG 状态之间，到底谁先持久，谁可以先被观察，谁负责补齐边界？

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

- `RecordTransactionCommit()` 先写 commit WAL record，再根据提交模式决定是否等待 WAL flush，然后更新 CLOG 状态；异步提交会把 commit LSN 存进 CLOG 页的 `group_lsn[]`，由 SLRU 写页和 hint bit 路径继续执行 WAL-before-CLOG/data 的持久化约束。

这个模型里有四个时间点。第一，commit record 被插入 WAL buffers。

- 入口是 `XactLogCommitRecord()`。
- 它调用 `XLogBeginInsert()`、`XLogRegisterData()`、`XLogInsert()`。
- `XLogInsert()` 最终进入 `XLogInsertRecord()`。
- 返回的 `XactLastRecEnd` 是 commit record 结束位置。
- 这个位置是 “需要 flush 到这里” 的 LSN。
- 它不是 “已经 fsync 到这里”。

第二，commit WAL 被 flush。

- 同步提交路径调用 `XLogFlush(XactLastRecEnd)`。
- `XLogFlush()` 会等待相关 WAL insertion 完成。
- 它会争用 `WALWriteLock`。
- 它可能 piggyback 更多 WAL 到同一次 fsync。
- 返回后，`logFlushResult` 至少覆盖请求的 commit LSN。

第三，CLOG/pg_xact 内存状态被更新。

- 同步路径调用 `TransactionIdCommitTree()`。
- 异步路径调用 `TransactionIdAsyncCommitTree()`。
- 两者最终都调用 `TransactionIdSetTreeStatus()`。
- 区别在于同步路径传入 `InvalidXLogRecPtr`。
- 异步路径传入 commit record 的 LSN。
- 有效 LSN 会写入 CLOG buffer slot 的 `group_lsn[]`。

第四，CLOG/pg_xact 页被写盘。

- CLOG 是 SLRU 管理的页。
- `SimpleLruWriteAll()`、victim eviction 或 checkpoint 都可能触发写页。
- `SlruPhysicalWritePage()` 写 CLOG 页前会扫描 `group_lsn[]`。
- 如果发现有效 LSN，就先 `XLogFlush(max_lsn)`。
- 这就是异步提交下 CLOG 落盘不能越过 WAL 的边界。

这四个时间点可以部分重叠。它们也可能被不同进程推进。

普通 backend 可以推进 WAL flush。WAL writer 可以推进异步提交 WAL flush。

checkpointer 可以触发 CLOG SLRU 写页。其他 backend 的可见性检查可以读取 CLOG 内存状态。

heap hint bit 路径还会用 commit LSN 判断是否能把 committed hint bit 写到数据页。所以，本节的核心不是 “commit 先做什么再做什么”。

更准确的问题是：

- PostgreSQL 如何允许运行时先观察到 committed，同时防止 crash 后持久状态撒谎？

同步提交的答案比较直接。

- commit WAL flush 完成。
- 再设置 CLOG committed。
- 然后从 ProcArray 中移除事务。
- 其他 backend 看到它不再 running 时，CLOG 已经能回答 committed。

异步提交的答案更微妙。

- backend 可以不等待 WAL fsync 就返回。
- backend 仍然会把 CLOG 内存状态设置为 committed。
- 其他 backend 在当前进程生命周期中可以把它看成 committed。
- 但是 CLOG 页写盘前必须先 flush 对应 commit WAL。
- hint bit 持久到 heap 页前也必须确认 commit WAL 已 flush 或已有 LSN interlock。
- 如果 postmaster crash 发生在 WAL flush 前，这个事务允许丢失。

这不是违反持久性。这是 `synchronous_commit = off` 明确选择的 durability 语义。

它降低前台提交延迟。它把 “最终把 async commit WAL 推到磁盘” 的责任交给 WAL writer、后续 flush 或 CLOG 写页边界。

## 3. 核心文件分工与阅读顺序

阅读顺序不要按文件名排序。本节按状态推进阅读。

第一步读 `xact.c` 的提交入口。

- `CommitTransactionCommand()` 是 SQL 命令结束处的状态机入口。
- `CommitTransaction()` 是真正的 top-level commit 主体。
- `RecordTransactionCommit()` 是持久提交边界的核心函数。
- `RecordTransactionAbort()` 用来对照 abort 为什么不需要同样的 WAL flush 约束。

第二步读 `xact.c` 的 WAL record 构造。

- `XactLogCommitRecord()` 收集 commit timestamp、subxacts、pending deletes、invalidation、origin 等信息。
- 它只负责构造并插入 commit record。
- 它不负责 fsync。
- 它返回 commit record end LSN。

第三步读 `xloginsert.c`。

- `XLogBeginInsert()` 开始构造 WAL record。
- `XLogRegisterData()` 等函数注册 record 数据。
- `XLogInsert()` 反复调用 `XLogRecordAssemble()` 和 `XLogInsertRecord()`。
- 如果 full-page-write 条件变化，它可能重试。

第四步读 `xlog.c` 的 WAL insertion 与 flush。

- `XLogInsertRecord()` 在 WAL buffers 中预留并复制 record。
- `ReserveXLogInsertLocation()` 分配 WAL 地址。
- `WALInsertLock` 记录 insertion 是否还在进行。
- `WaitXLogInsertionsToFinish()` 保证要写出的 WAL 已经完整复制进 buffer。
- `XLogFlush()` 把 WAL 写到文件并 fsync 到目标 LSN。
- `XLogWrite()` 持有 `WALWriteLock` 执行实际写和 sync。
- `XLogSetAsyncXactLSN()` 把 async commit 目标告诉 WAL writer。
- `XLogBackgroundFlush()` 是 WAL writer 周期性推进异步提交的入口。

第五步读 `transam.c` 的 CLOG 高层接口。

- `TransactionIdCommitTree()` 是同步提交的 CLOG 更新入口。
- `TransactionIdAsyncCommitTree()` 是异步提交的 CLOG 更新入口。
- `TransactionIdAbortTree()` 是 abort 的 CLOG 更新入口。
- `TransactionLogFetch()` 读取 CLOG 并带一个单项 cache。
- `TransactionIdDidCommit()` 处理 `SUB_COMMITTED` 时会递归查父事务。
- `TransactionIdGetCommitLSN()` 给 hint bit 路径提供 commit LSN 边界。

第六步读 `clog.c`。

- `TransactionIdSetTreeStatus()` 处理主事务和子事务树。
- `TransactionIdSetPageStatus()` 处理单个 CLOG 页。
- `TransactionIdSetPageStatusInternal()` 修改 buffer 中的状态 bit。
- `TransactionIdSetStatusBit()` 写两位状态，并在异步提交时更新 `group_lsn[]`。
- `TransactionGroupUpdateXidStatus()` 在 CLOG bank lock 竞争时合并多个提交者。
- `TransactionIdGetStatus()` 返回状态和关联 LSN。
- `ExtendCLOG()` 为新 XID 所在页初始化零页并 WAL-log。
- `CheckPointCLOG()` 把脏 CLOG 页交给 SLRU 写出。
- `TruncateCLOG()` 删除旧 pg_xact 段前会写并 flush truncate WAL。

第七步读 `slru.c`。

- `SimpleLruReadPage()` 把 CLOG 页读入共享 buffer slot。
- `SimpleLruReadPage_ReadOnly()` 允许只读路径先拿 shared bank lock。
- `SlruSelectLRUPage()` 选择 victim，必要时写出脏页。
- `SlruInternalWritePage()` 标记 `WRITE_IN_PROGRESS`，释放 bank lock 做 I/O。
- `SlruPhysicalWritePage()` 是 WAL-before-CLOG-page-write 的实际执行点。
- `SimpleLruWriteAll()` 是 checkpoint 或 shutdown 写 SLRU 页的入口。

第八步读 `src/backend/access/transam/README`。

- 先看 transaction end 与 ProcArray 的关系。
- 再看 asynchronous commit 段落。
- 注意 README 说的是设计约束，不是每个函数的当前实现细节。

核心文件分工：

| 文件 | 本节关注点 |
| --- | --- |
| `xact.c` | commit/abort 主流程、critical section、sync/async 分支、ProcArray 前后顺序 |
| `xloginsert.c` | WAL record 构造与插入入口，解释 commit record LSN 从哪里来 |
| `xlog.c` | WAL insert/write/flush 的共享状态、group commit、WAL writer 异步推进 |
| `clog.c` | CLOG 状态 bit、subcommit 原子性、group update、CLOG 页 LSN |
| `slru.c` | CLOG buffer 页生命周期、写页前 WAL flush、I/O 错误处理 |
| `xact.h` | 事务入口和 `XactLogCommitRecord()` 原型 |
| `clog.h` | CLOG 状态枚举和公开接口 |
| `transam/README` | ProcArray、snapshot、async commit 的设计不变量 |

阅读时要特别避免一个误区：

- 不要从 `XLogInsert()` 推断事务已经 durable。
- 不要从 CLOG bit 推断 commit WAL 已经 durable。
- 不要从 backend 已经返回推断 crash 后一定保留。
- 这些结论都必须结合提交模式和 LSN 边界。

## 4. 关键数据结构与状态

### 4.1 CLOG 两位状态

`clog.h` 定义了事务状态：

```text
TRANSACTION_STATUS_IN_PROGRESS
TRANSACTION_STATUS_COMMITTED
TRANSACTION_STATUS_ABORTED
TRANSACTION_STATUS_SUB_COMMITTED
```

每个事务占两位。每个字节存四个事务。

一个 CLOG page 能覆盖 `BLCKSZ * 4` 个 XID。这解释了 CLOG 很小。

也解释了为什么 CLOG page contention 会集中在高并发提交热点上。但 raw bit 不是完整语义。

`SUB_COMMITTED` 尤其不能单独解释。子事务跨页提交时，PostgreSQL 会先把不在主事务页上的子事务标成 `SUB_COMMITTED`。

然后提交主事务。最后再把那些子事务补成 `COMMITTED`。

并发 reader 看到 `SUB_COMMITTED` 时必须继续查父事务。如果父事务 committed，子事务才 committed。

如果父事务没有 committed，子事务不能被当成 committed。所以语义不是：

- CLOG bit = final truth。

而是：

- CLOG bit + 是否子事务 + `pg_subtrans` 父链 + XID horizon = visibility truth。

本节不展开 `pg_subtrans`。但必须记住 `SUB_COMMITTED` 是 CLOG 跨页原子性策略的一部分。

### 4.2 `XactLastRecEnd` 与 commit record LSN

`XactLastRecEnd` 是 backend-local 的 WAL 位置状态。`XactLogCommitRecord()` 返回 `XLogInsert()` 的结果。

`RecordTransactionCommit()` 用它判断：

- 是否已经写过 WAL。
- 同步提交要 flush 到哪里。
- 异步提交要报告哪个 LSN 给 WAL writer。
- CLOG `group_lsn[]` 要记录哪个 commit LSN。

这个字段的关键边界：

- 它不是共享状态。
- 它不是 durable 标志。
- 它只是当前事务最近 WAL record 的 end LSN。
- `XLogFlush(XactLastRecEnd)` 返回之后才说明 WAL flush 覆盖该位置。

`XactLastCommitEnd` 记录上一个 commit record end。它用于后续状态和统计场景。

不要把它当作当前事务是否 committed 的判断入口。

### 4.3 `XLogCtlData`

`xlog.c` 中的 `XLogCtlData` 是共享内存里的 WAL 控制中心。本节只需要关注几个字段。

`XLogCtl->Insert` 负责 WAL insertion。其中 `CurrBytePos` 和 `PrevBytePos` 由 `insertpos_lck` 保护。

它们决定下一个 WAL record 预留在哪个位置。`WALInsertLocks` 是一组固定数量的插入锁。

每个插入锁都有：

- `lock`
- `insertingAt`
- `lastImportantAt`

`insertingAt` 的意义不是 “持有者是谁”。它表达：

- 这个 inserter 已经推进到哪个 LSN。
- flush 进程只需要等待会影响目标范围的 insertion。
- 避免所有 inserter 互相等待导致死锁。

`logInsertResult` 是已经完整插入 WAL buffers 的位置。`logWriteResult` 是已经写到 WAL 文件的位置。

`logFlushResult` 是已经 flush/fsync 到 durable boundary 的位置。读取时必须保持顺序意识。

`xlog.c` 用 barrier 保证看到的 flush 不会超过对应 write。`LogwrtRqst` 是共享的写请求。

WAL inserter 跨页时会推进 `LogwrtRqst.Write`。`XLogFlush()` 也会参考它，尽量把更多 WAL 一起写出。

`asyncXactLSN` 是异步提交的最新目标 LSN。`XLogSetAsyncXactLSN()` 会把它向前推进。

如果 WAL writer 在睡眠，还会通过 latch 唤醒它。

### 4.4 `SlruSharedData`

CLOG/pg_xact 页由 SLRU 管理。`SlruSharedData` 是共享内存状态。

本节关注字段：

- `page_buffer`
- `page_status`
- `page_dirty`
- `page_number`
- `page_lru_count`
- `buffer_locks`
- `bank_locks`
- `group_lsn`
- `lsn_groups_per_page`
- `latest_page_number`

`page_status` 有四种：

- `SLRU_PAGE_EMPTY`
- `SLRU_PAGE_READ_IN_PROGRESS`
- `SLRU_PAGE_VALID`
- `SLRU_PAGE_WRITE_IN_PROGRESS`

`page_dirty` 不是状态枚举的一部分。它和 `page_status` 组合解释。

一个页可以是 `WRITE_IN_PROGRESS` 且又被重新 dirty。这意味着：

- 旧版本正在写出。
- 写出期间又有新的事务状态更新。
- 写完后页仍然需要未来再次写。

`group_lsn[]` 只在某些 SLRU 中存在。对 CLOG 来说它存在。

它不是每个 XID 一个 LSN。它是每 32 个 XID 一组 LSN。

每组记录该范围内最高的异步 commit LSN。这个设计节省共享内存。

代价是 hint bit 或写页可能被一个更晚事务的 LSN 拖住。这是有意接受的近似。

### 4.5 `PGPROC` 中的提交相关状态

每个 backend 有一个 `PGPROC`。提交路径会用到：

- `MyProc->xid`
- `MyProc->subxids`
- `MyProc->subxidStatus`
- `MyProc->delayChkptFlags`
- `MyProc->clogGroupMember*`
- `MyProc->procArrayGroupMember*`

`delayChkptFlags` 中的 `DELAY_CHKPT_IN_COMMIT` 是本节关键字段。`RecordTransactionCommit()` 在 commit critical section 里设置它。

注释说明它会迫使并发 checkpoint 等待当前 backend 更新 `pg_xact`。否则可能出现：

- checkpoint 选择了 commit WAL record 之后的位置作为 redo。
- checkpoint 没有把对应 CLOG 更新刷到磁盘。
- crash 后从该 checkpoint 恢复时丢失 committed 状态。

这个字段的语义不是 “事务已经提交”。它表达的是：

- 当前 backend 正处在 commit WAL 与 CLOG 状态更新之间的危险窗口。
- checkpoint 不应该跨过这个窗口。

`clogGroupMember*` 用于 CLOG group update。`procArrayGroupMember*` 用于 ProcArray group clear。

这两类 group 机制解决的是不同锁的 contention。不要混为一谈。

### 4.6 `synchronous_commit`、`forceSyncCommit`、`nrels`

`RecordTransactionCommit()` 不是只看 `synchronous_commit`。它的同步 flush 条件大致是：

- 事务写过 WAL。
- 事务有 XID。
- `synchronous_commit > off`。

或者：

- `forceSyncCommit` 为 true。

或者：

- 有待删除的非临时 relation 文件。

这说明 `synchronous_commit = off` 不是绝对不 flush。有些副作用不能容忍 commit record 留在未 flush WAL buffers 中。

例如删除 relation 文件前必须确保 commit record 持久。否则可能出现文件系统变化已经发生，而 crash recovery 没有相应事务提交事实。

## 5. 主流程源码 walkthrough

### 5.1 从 SQL 结束进入 commit

主流程从 `CommitTransactionCommand()` 开始。它是事务 block 状态机的一部分。

真正提交工作在 `CommitTransaction()`。`CommitTransaction()` 先做 pre-commit 处理：

- deferred trigger。
- holdable portal。
- transaction callback。
- parallel worker cleanup。
- ON COMMIT action。
- pending sync。
- large object cleanup。
- NOTIFY queue。
- serializable conflict check。
- relation map commit。

这些步骤可能抛 ERROR。一旦还没有进入真正 durable commit 边界，ERROR 可以把事务转入 abort 路径。

所以 `CommitTransaction()` 的前半段不是本节的持久化边界。真正边界从这里开始：

- 设置 `s->state = TRANS_COMMIT`。
- 禁止 transaction timeout。
- 普通 backend 调用 `RecordTransactionCommit()`。
- parallel worker 不自己提交 leader 的事务。

`CommitTransaction()` 中有一句注释很重要：

- `RecordTransactionCommit()` 是 “where we durably commit”。

但这句话要结合同步/异步提交语义理解。同步提交下，它确实等待本地 WAL flush。

异步提交下，它建立运行时 committed 状态，但允许 crash 丢失。

### 5.2 `RecordTransactionCommit()` 先收集提交 record 所需信息

`RecordTransactionCommit()` 首先获取 top transaction ID：

- `GetTopTransactionIdIfAny()`
- 如果没有 XID，则 `markXidCommitted = false`。

随后收集 commit record 所需内容：

- `smgrGetPendingDeletes(true, &rels)`
- `xactGetCommittedChildren(&children)`
- `pgstat_get_transactional_drops(true, &droppedstats)`
- `xactGetCommittedInvalidationMessages(...)`
- `XactLastRecEnd != 0` 得出 `wrote_xlog`

这些信息决定 commit record 内容。也决定后续是否必须同步 flush。

如果事务没有 XID：

- 不能写普通 COMMIT record。
- 也不想为了 commit record 强制分配 XID。
- pending relation deletes 或 dropped stats 不应该出现在无 XID commit 中。
- standby invalidation 可能写 bespoke WAL record。
- 如果没有 WAL，就直接 cleanup。

无 XID 事务是重要边界。只读事务不需要进入 CLOG。

也不需要在 ProcArray 里推进 `latestCompletedXid`。这是 snapshot scalability 的基本优化之一。

### 5.3 进入 commit critical section

有 XID 的普通提交会进入 critical section：

```text
START_CRIT_SECTION()
MyProc->delayChkptFlags |= DELAY_CHKPT_IN_COMMIT
pg_write_barrier()
XactLogCommitRecord(...)
TransactionTreeSetCommitTsData(...)
...
clear DELAY_CHKPT_IN_COMMIT
END_CRIT_SECTION()
```

这里有三个边界。第一，critical section 中 ERROR 会升级为更严重的恢复动作。

提交状态不能半途留下普通 ERROR 可恢复的不一致。第二，`DELAY_CHKPT_IN_COMMIT` 在 commit timestamp 之前设置。

注释强调 barrier 让这个 flag 对其他进程可见。第三，checkpoint 被迫等待该事务完成 CLOG 更新。

否则 checkpoint redo 点可能跨过 commit WAL record。如果同时没有把 CLOG 状态刷入 checkpoint 数据集，恢复将看到不完整事务结果。

这就是 commit WAL 与 CLOG 持久边界之间的第一道互锁。

### 5.4 写 commit WAL record

`XactLogCommitRecord()` 做的是 record 组装。它决定 record 类型：

- `XLOG_XACT_COMMIT`
- `XLOG_XACT_COMMIT_PREPARED`

它按需附加信息：

- commit timestamp。
- relcache invalidation 标志。
- `forceSyncCommit` 标志。
- AccessExclusive lock 标志。
- subtransaction 列表。
- pending delete 的 relfilenode 列表。
- dropped stats。
- shared invalidation messages。
- two-phase xid/gid。
- replication origin。

然后它调用：

```text
XLogBeginInsert()
XLogRegisterData(...)
XLogSetRecordFlags(XLOG_INCLUDE_ORIGIN)
XLogInsert(RM_XACT_ID, info)
```

`XLogInsert()` 位于 `xloginsert.c`。它检查 `XLogBeginInsert()` 是否调用过。

它可能因为 full-page-write 条件变化而重试。最终进入 `XLogInsertRecord()`。

`XLogInsertRecord()` 位于 `xlog.c`。它进入 WAL insertion critical section。

它拿一个 `WALInsertLock`。它用 `ReserveXLogInsertLocation()` 预留 WAL 空间。

它填 `xl_prev`。它计算 CRC。

它把 record 拷贝进 WAL buffers。它释放 WAL insert lock。

它返回 `EndPos`。这个 `EndPos` 回到 `RecordTransactionCommit()` 后就是 `XactLastRecEnd`。

到这里为止：

- commit record 已在 WAL buffers。
- commit record 未必写到 WAL file。
- commit record 未必 fsync。
- CLOG 还未设置 committed。
- 事务还在 ProcArray 中 running。

这个点最容易被误读。`XLogInsert()` 的成功只说明 WAL record 已经可被后续 flush。

它不说明 commit 已 durable。

### 5.5 同步提交分支

同步分支条件包括：

```text
(wrote_xlog && markXidCommitted && synchronous_commit > SYNCHRONOUS_COMMIT_OFF)
  || forceSyncCommit
  || nrels > 0
```

进入同步分支后：

```text
XLogFlush(XactLastRecEnd)
TransactionIdCommitTree(xid, nchildren, children)
```

`XLogFlush()` 的目标是保证 WAL durable 到 commit LSN。它首先检查当前 backend 的 `LogwrtResult.Flush`。

如果已经覆盖目标 LSN，快速返回。否则进入 critical section。

它会刷新共享 `LogwrtResult`。它会根据 `LogwrtRqst.Write` 扩大写请求。

它会调用 `WaitXLogInsertionsToFinish()`。这一步保证目标范围内 WAL record 已经完整复制进 WAL buffers。

然后它尝试获取 `WALWriteLock`。拿不到时，它会等持锁者释放。

释放后重新检查目标 LSN 是否已经被别人 flush。这就是 group commit 的基础形状。

拿到 `WALWriteLock` 后：

- 可能执行 `commit_delay`。
- 可能 piggyback 后续已插入 WAL。
- 调用 `XLogWrite()`。
- 更新共享 `logWriteResult` 和 `logFlushResult`。
- 唤醒 wal sender 和等待 LSN 的进程。

`XLogWrite()` 是实际写 WAL 文件和 fsync 的函数。它会聚合连续 WAL pages。

它在需要 flush 时调用 `issue_xlog_fsync()`。它把 `LogwrtResult.Flush` 推进到写出位置。

`XLogFlush()` 返回后，同步提交的本地 WAL 持久边界已满足。然后 `TransactionIdCommitTree()` 才更新 CLOG。

它传入 `InvalidXLogRecPtr`。这表示：

- CLOG 页不需要为这次 commit 记 async LSN。
- 因为 WAL 已经 flush 到 commit record。
- CLOG 页未来写盘时不需要为这个 commit 再补 WAL flush。

注意顺序：

- 先 flush WAL。
- 再设置 CLOG committed。
- 后面才从 ProcArray 移除。

这是同步提交下最直观的不变量。

### 5.6 异步提交分支

异步提交路径进入 `else`。代码做两件事：

```text
XLogSetAsyncXactLSN(XactLastRecEnd)
TransactionIdAsyncCommitTree(xid, nchildren, children, XactLastRecEnd)
```

第一，`XLogSetAsyncXactLSN()` 推进 `XLogCtl->asyncXactLSN`。它持有 `info_lck` 修改共享状态。

如果 WAL writer 在睡，会设置 WAL writer 的 latch。如果未睡，则根据 `wal_writer_flush_after` 判断是否需要唤醒。

第二，`TransactionIdAsyncCommitTree()` 更新 CLOG。它和同步 commit 一样会把事务状态设为 committed。

区别是它把 `XactLastRecEnd` 作为 LSN 传下去。最终 `TransactionIdSetStatusBit()` 会更新 `group_lsn[]`。

这意味着：

- 当前运行中的其他 backend 可以读到 CLOG committed。
- 但 CLOG 页带着 “写盘前必须先 flush WAL 到这个 LSN” 的债务。

异步提交不是 “不更新 CLOG”。异步提交是 “更新内存 CLOG，但延迟 WAL durable，并把延迟边界挂到 CLOG page LSN 上”。

如果系统正常运行，WAL writer 会推进 `XLogBackgroundFlush()`。如果后续同步提交调用 `XLogFlush()`，也可能顺带 flush async commit WAL。

如果 checkpoint 或 SLRU eviction 要写出 CLOG 页，`SlruPhysicalWritePage()` 会强制 flush。这三条路径都会收敛到同一个 durable 边界。

### 5.7 CLOG 更新入口

同步和异步最终都进入：

```text
TransactionIdSetTreeStatus(xid, nsubxids, subxids, status, lsn)
```

提交时 `status` 是 `TRANSACTION_STATUS_COMMITTED`。abort 时 `status` 是 `TRANSACTION_STATUS_ABORTED`。

异步 commit 的 `lsn` 有效。同步 commit 和 abort 的 `lsn` 无效。

函数先判断子事务是否和父事务在同一个 CLOG page。如果都在同页：

- 一次调用 `TransactionIdSetPageStatus()`。
- 在同一页锁保护下更新父事务和子事务。

如果跨页：

- 先把不在父事务页上的子事务标为 `SUB_COMMITTED`。
- 再提交父事务页上的父事务和同页子事务。
- 最后回到其他页把子事务标为 `COMMITTED`。

这个顺序保证并发可见性检查不会看到 “子事务 committed，但父事务还没 committed” 的错误结论。主事务 commit 是原子语义中心。

跨页子事务状态只是为了让实现可以分多页更新。

### 5.8 CLOG 单页更新与 group update

`TransactionIdSetPageStatus()` 先计算该 CLOG page 的 SLRU bank lock。它有一个优化：

- 如果所有 XID 在同一页。
- 如果当前 XID 等于 `MyProc->xid`。
- 如果子事务数量不超过 `THRESHOLD_SUBTRANS_CLOG_OPT`。
- 如果 `MyProc` 中缓存的 subxids 与传入数组一致。
- 且立即拿不到 bank lock。

那么它尝试 `TransactionGroupUpdateXidStatus()`。CLOG group update 的目的：

- 多个 backend 同时提交时，不要让 bank lock 在进程间反复移交。
- 第一个进队列的 backend 成为 leader。
- leader 拿 bank lock。
- leader 替队列里多个 backend 更新 CLOG status。
- follower 睡在自己的 semaphore 上。

这是提交路径的 contention 优化。它不改变正确性语义。

如果不能 group：

- 回退到普通 `LWLockAcquire(lock, LW_EXCLUSIVE)`。
- 自己执行 `TransactionIdSetPageStatusInternal()`。

注意 fallback 是正确性必需。group update 只是性能优化。

任何条件不满足，都必须能回到单事务更新。

### 5.9 CLOG bit 与 group LSN

`TransactionIdSetPageStatusInternal()` 在已经持有 bank lock 的前提下工作。它调用：

```text
SimpleLruReadPage(XactCtl, pageno, !XLogRecPtrIsValid(lsn), &xid)
```

这里的 `write_ok` 参数非常关键。如果 `lsn` 无效：

- 同步 commit 或 abort。
- 可以允许在 `WRITE_IN_PROGRESS` 页上直接改。
- 因为不担心这个更新比对应 WAL commit 更早到达磁盘。

如果 `lsn` 有效：

- 异步 commit。
- 不允许在正在写出的页上直接改。
- 必须等待 active write 完成。
- 否则该异步 committed bit 可能被已经进行中的写页带到磁盘，而 commit WAL 尚未 flush。

这就是异步 commit 和 SLRU write-in-progress 的局部互锁。随后 `TransactionIdSetStatusBit()` 修改两位状态。

它先读当前 bit。它允许 recovery 中某些 no-op。

正常状态转换应从 `IN_PROGRESS` 或 `SUB_COMMITTED` 到目标状态。如果传入有效 LSN：

- 计算 `GetLSNIndex(slotno, xid)`。
- 如果当前 `group_lsn[lsnindex]` 小于该 LSN，则更新。

最后 `TransactionIdSetPageStatusInternal()` 设置：

- `XactCtl->shared->page_dirty[slotno] = true`

此时 CLOG 更新仍只是在 shared memory buffer 中。是否已经写到 `pg_xact` 文件，要看 SLRU 写页。

### 5.10 CLOG 页写盘边界

CLOG 页写盘有多种触发源：

- checkpoint 调用 `CheckPointCLOG()`。
- `CheckPointCLOG()` 调用 `SimpleLruWriteAll(XactCtl, true)`。
- SLRU victim selection 发现需要腾出脏页。
- shutdown 或其他 SLRU flush 路径。

写页主链路：

```text
SimpleLruWriteAll()
  -> SlruInternalWritePage()
     -> SlruPhysicalWritePage()
        -> maybe XLogFlush(max(group_lsn[]))
        -> pg_pwrite(pg_xact page)
        -> RegisterSyncRequest() or pg_fsync()
```

`SlruInternalWritePage()` 会先把页标为 `SLRU_PAGE_WRITE_IN_PROGRESS`。它清掉 `page_dirty`。

它拿 per-buffer lock。然后释放 bank lock 做 I/O。

写 I/O 期间，其他 backend 可能重新 dirty 这个页。写完后，如果写失败，会把 `page_dirty` 重新置 true。

如果写成功，但期间被重新 dirty，页仍然需要未来再次写。`SlruPhysicalWritePage()` 是本节最关键的边界之一。

它检查 `shared->group_lsn != NULL`。对 CLOG 来说该数组存在。

它扫描当前 slot 的所有 LSN group。取最大 `max_lsn`。

如果 `max_lsn` 有效：

- 进入 critical section。
- 调用 `XLogFlush(max_lsn)`。
- 退出 critical section。

只有在这个 flush 后，它才写 CLOG 页。所以异步 commit 的 CLOG durable 边界不是 commit backend 返回时。

它是在 CLOG 页真正写出前被强制补齐。这也解释了为什么 CLOG `group_lsn[]` 可以在页重新读入时清零。

旧页能从磁盘读入，说明它上一次写盘已经满足 WAL-before-CLOG-page-write。不再需要记住旧的 async commit LSN。

### 5.11 从 ProcArray 消失

`RecordTransactionCommit()` 返回 `latestXid`。`CommitTransaction()` 随后调用：

```text
ProcArrayEndTransaction(MyProc, latestXid)
```

注释要求：

- 必须在释放锁之前。
- 必须在 `RecordTransactionCommit()` 之后。

`ProcArrayEndTransaction()` 清除当前 backend advertised XID。如果有 XID，它必须持有 `ProcArrayLock` exclusive。

原因是 snapshot 构造会持有 `ProcArrayLock` shared。事务不能在 snapshot 读取 `latestCompletedXid` 和扫描 ProcArray 之间悄悄消失。

`ProcArrayEndTransactionInternal()` 做几件事：

- 清 `ProcGlobal->xids[pgxactoff]`。
- 清 `proc->xid`。
- 清 virtual xid。
- 清 `proc->xmin`。
- 清 subxid cache。
- 推进 `latestCompletedXid`。
- 增加 `xactCompletionCount`。

如果拿不到 `ProcArrayLock`，会走 `ProcArrayGroupClearXid()`。这是另一个 group commit 形状。

它减少多个事务结束时对 ProcArrayLock 的移交。它与 CLOG group update 分离。

最终顺序仍是：

- CLOG 已更新。
- 事务才离开 running set。
- 释放 locks 和资源。

这保证新的 snapshot 不会把一个已离开 ProcArray、但 CLOG 未知的事务错误处理。

### 5.12 同步复制等待位置

`RecordTransactionCommit()` 在 CLOG 更新后调用：

```text
SyncRepWaitForLSN(XactLastRecEnd, true)
```

条件是 `wrote_xlog && markXidCommitted`。注释明确说：

- 此时已经 marked clog。
- 仍然显示为 ProcArray running。
- 仍然持有 locks。

这说明 synchronous replication 等待不是本地 CLOG 更新的前置条件。它是对远端 flush/apply 语义的额外等待。

本课只把它作为边界：

- 本地 crash safety 由本地 WAL flush 与 CLOG 规则负责。
- synchronous replication 在此之上增加复制确认语义。

### 5.13 post-commit cleanup

`ProcArrayEndTransaction()` 之后，`CommitTransaction()` 进入 post-commit cleanup。注释说明：

- 这里已经太晚，不能 abort。
- 只应该释放非关键资源。

释放顺序也不是随机的：

- 先释放其他 backend 可见的资源。
- 再释放 locks。
- 最后释放 backend-local 资源。

典型调用包括：

- `ResourceOwnerRelease(... BEFORE_LOCKS ...)`
- `AtEOXact_Buffers(true)`
- `AtEOXact_RelationCache(true)`
- `AtEOXact_Inval(true)`
- `ResourceOwnerRelease(... LOCKS ...)`
- `smgrDoPendingDeletes(true)`
- `AtCommit_Notify()`
- `AtEOXact_GUC(true, 1)`
- `AtEOXact_Snapshot(true, false)`
- `ResourceOwnerDelete(TopTransactionResourceOwner)`
- `AtCommit_Memory()`

这部分不改变事务是否 committed。它清理所有围绕事务生命周期建立的本地和共享资源。

## 6. 生命周期 / ownership / cleanup

### 6.1 谁创建状态

事务状态由 backend 创建。`StartTransaction()` 建立 top transaction state。

XID 不是事务开始立即分配。第一次需要 XID 时才分配。

分配 XID 后，backend 把它广告到 ProcArray。这让其他 snapshot 能看到它 running。

WAL record 由当前 backend 创建。`XactLogCommitRecord()` 创建 commit record 内容。

`XLogInsertRecord()` 把它放入共享 WAL buffers。CLOG 页不是事务单独创建的对象。

它是按 XID 范围划分的 shared SLRU page。`ExtendCLOG()` 在新 XID 正好位于新 CLOG page 开头时 zero page。

它还写 `CLOG_ZEROPAGE` WAL record。

### 6.2 谁持有状态

事务运行中：

- 当前 backend 持有 `CurrentTransactionState`。
- `TopTransactionResourceOwner` 持有 buffer pins、locks 等外部资源。
- `TopTransactionContext` 持有事务周期内的内存。
- `PGPROC` 广告当前 XID、xmin、subxids。
- WAL buffers 持有尚未写盘的 WAL bytes。
- CLOG SLRU buffers 持有事务状态页。

CLOG 页没有单个 owner。它由 SLRU bank lock 和 per-buffer lock 协调共享访问。

WAL flush 也没有单个事务 owner。任何 backend 或 WAL writer 都可能把某个事务的 commit WAL 顺带刷到磁盘。

这点对理解 group commit 很重要。提交者不一定是唯一付 fsync 成本的人。

### 6.3 谁释放状态

`RecordTransactionCommit()` 只处理 commit 记录和 CLOG 状态。`CommitTransaction()` 负责事务生命周期收尾。

`ProcArrayEndTransaction()` 释放 running XID 对 snapshot 的影响。`ResourceOwnerRelease()` 释放资源所有权。

`AtEOXact_*` 系列函数清理各子系统状态。`AtCommit_Memory()` 清理 top transaction memory。

CLOG SLRU 页不会在事务结束时释放。它留在 shared SLRU cache 中。

未来由 LRU、checkpoint 或 truncation 处理。WAL buffers 也不会按事务释放。

它们按 WAL ring、flush、checkpoint、segment recycling 生命周期推进。

### 6.4 ERROR 与 abort 兜底

在 durable commit 边界之前，ERROR 可以进入 abort。`AbortTransaction()` 会调用 `RecordTransactionAbort(false)`。

如果事务没有 XID，abort 直接返回 `InvalidTransactionId`。如果有 XID：

- 写 ABORT record。
- 不等待 WAL flush。
- 调用 `XLogSetAsyncXactLSN()` 通知 WAL writer。
- 调用 `TransactionIdAbortTree()` 标记 CLOG aborted。

abort 不需要 commit 同样的 flush 规则。原因是 crash 后默认假设未提交事务 aborted。

丢失 abort record 不会让一个未提交事务变成 committed。这也是为什么 `RecordTransactionCommit()` 需要 `DELAY_CHKPT_IN_COMMIT`。

而 `RecordTransactionAbort()` 注释说不需要同等 checkpoint interlock。

### 6.5 长期对象如何失效

CLOG 旧段由 `TruncateCLOG()` 删除。它不能随意删除。

删除前必须知道系统不再需要那些 XID 的 commit status。删除前还会写 `CLOG_TRUNCATE` WAL record 并 `XLogFlush()`。

这是为了保证 freeze/truncation 相关记录先到磁盘。否则 crash 后可能有 tuple 仍引用已删除 CLOG。

事务可见性相关长期状态还有 hint bit。hint bit 位于 heap page。

它可以避免未来反复查 CLOG。但 committed hint bit 写入 permanent buffer 时也受 commit LSN 约束。

`SetHintBitsExt()` 会调用 `TransactionIdGetCommitLSN()`。如果 `XLogNeedsFlush(commitLSN)` 且 heap page LSN 小于 commit LSN，就不设置 hint bit。

这防止 heap page 上的 committed hint bit 先于 commit WAL 持久化。

## 7. 正确性机制层次

### 7.1 WAL-before-CLOG-page-write

最核心不变量：

- 如果一个 CLOG page 在磁盘上记录某 XID committed，则该 XID 的 commit WAL 必须已经 flush。

同步 commit 通过前台 `XLogFlush()` 满足。异步 commit 通过 `group_lsn[]` 加 `SlruPhysicalWritePage()` 满足。

这不是性能优化。这是 crash recovery correctness。

如果 CLOG committed 先落盘，而 commit WAL 没落盘：

- crash 后恢复可能看不到 commit record。
- 但 pg_xact 文件却说 committed。
- 系统会把一个未 WAL-protected 的事务当作 committed。

PostgreSQL 的设计明确避免这个状态。

### 7.2 WAL insertion 与 WAL flush 分层

`XLogInsertRecord()` 保证 record 已在 WAL buffers 中。`XLogFlush()` 保证 record 对应 WAL 持久。

`XLogBackgroundFlush()` 可以后台推进持久化。三者不是同一层。

这解释了为什么 commit path 可以先拿到 LSN，再由不同路径 flush。LSN 是 ordering token。

fsync 是 durability boundary。不要把 token 当 boundary。

### 7.3 CLOG 内存状态与 CLOG 文件状态分层

CLOG shared buffer 中的 bit 可以先改变。CLOG 文件中的 bit 只能在 WAL 边界满足后改变。

同步 commit 中这两个边界几乎紧邻。异步 commit 中它们可能相隔多个 WAL writer cycle。

这种分层让异步提交可以快速返回。也让系统在 crash 后保持一致。

### 7.4 ProcArray 与 CLOG 的顺序

事务从 ProcArray 消失之前，必须已经记录 commit/abort 到 WAL 和 CLOG。`ProcArrayEndTransaction()` 注释明确说：

- transaction commit/abort must already be reported to WAL and pg_xact。

原因是 snapshot 构造依赖 running set。一个事务如果不在 running set 中，snapshot 和 visibility 逻辑会去查它的最终状态。

如果 CLOG 还没准备好，其他 backend 会遇到不可解释的状态。顺序因此是：

- 先写 commit WAL。
- 再更新 CLOG。
- 再从 ProcArray 清掉 XID。

### 7.5 Checkpoint interlock

checkpoint 分三个阶段：

- 选择 redo。
- 写数据。
- WAL-log checkpoint。

`DELAY_CHKPT_IN_COMMIT` 防止 checkpoint 跨过 commit critical section。如果 checkpoint redo 点选在 commit WAL record 之后，checkpoint 必须也能涵盖对应 CLOG 状态。

否则 recovery 从 checkpoint 开始就可能遗漏事务完成状态。这个机制不保护所有数据页。

它专门保护 commit WAL 与 CLOG 更新之间的窗口。

### 7.6 Hint bit interlock

Heap hint bit 是跨模块传播的第二条边界。可见性检查知道某 XID committed，不代表可以马上把 committed hint bit 持久写到 heap page。

如果 commit 是异步的，commit WAL 可能未 flush。把 hint bit 写到 heap page 后，heap page 可能先于 commit WAL 落盘。

crash 后恢复会看到 hint bit，误以为事务 committed。`SetHintBitsExt()` 因此检查 commit LSN。

如果 WAL 未 flush 且 heap page 自身 LSN 没有覆盖 commit LSN，就跳过 hint bit。这说明 CLOG LSN 不只服务 CLOG 写页。

它还服务 heap 可见性优化的安全边界。

### 7.7 Locks 与 latches 的层次

本节涉及多类同步机制。`WALInsertLock` 保护 WAL insertion slot。

`WALWriteLock` 保护 WAL 文件写和 fsync。`SLRU bank lock` 保护 CLOG page slot 元数据和 bit 更新。

`SLRU buffer lock` 保护单个 slot 的 I/O。`ProcArrayLock` 保护 running XID 集合的清理。

`PGSemaphore` 用于 group update follower 睡眠。`Latch` 用于唤醒 WAL writer。

这些机制各自保证不同东西。它们都不单独等于事务语义。

事务语义来自这些机制按顺序组合。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 无 XID 事务

没有分配 XID 的事务不能写普通 commit record。也不需要写 CLOG。

它可能仍然写过某些 WAL。例如 HOT pruning 可能发生。

如果没有 WAL，`RecordTransactionCommit()` 直接 cleanup。如果有 standby invalidation，则可能写专门的 invalidation WAL record。

这个边界防止只读或轻量事务被强行纳入 XID/CLOG 路径。也减少 ProcArray 和 CLOG 压力。

### 8.2 只写 temp/unlogged 的事务

事务可能分配了 XID，但没有需要持久 WAL 的永久表修改。`RecordTransactionCommit()` 注释提到：

- temporary table crash 后会消失。
- unlogged table crash 后会被截断。
- 这类事务可能不用同步 flush commit WAL。

但 Hot Standby 和 KnownAssignedXids 等机制仍可能要求记录 XID assignment 或 commit 相关 WAL。所以实现不是简单地 “不写 WAL”。

它根据 WAL 活动和持久副作用选择 flush 边界。

### 8.3 relation 文件删除强制同步

如果 commit 要删除非临时 relation 文件，不能异步提交。`nrels > 0` 会强制同步分支。

原因是文件系统删除是外部持久副作用。如果先删除文件，再 crash，且 commit WAL 未持久：

- recovery 可能认为事务没有提交。
- 但文件已经不在。
- catalog 和文件系统就可能不一致。

因此 pending deletes 把事务推回同步 flush。

### 8.4 `forceSyncCommit`

某些命令有非事务性或难回滚副作用。它们调用 `ForceSyncCommit()`。

`XactLogCommitRecord()` 也会把 force sync 信息写入 commit record。正常运行时这强制前台 flush。

recovery 重放时如果看到该标志，也可能调用 `XLogFlush(lsn)` 更新最小恢复点。这是把 higher-level 副作用传播到 WAL/CLOG 边界的例子。

### 8.5 `XLogFlush()` 失败

`XLogFlush()` 在 commit 路径里处于 critical section。如果 flush 请求无法满足，代码会 `elog(ERROR)`。

但在 critical section 中，普通 ERROR 会升级为 PANIC 级别的处理。这是合理的。

commit critical section 中系统不能继续运行在未知持久状态下。相比之下，buffer manager 因坏页 LSN 调用 `XLogFlush()` 可能不在 critical section。

那种情况不一定要让整个系统 PANIC。同一个函数在不同调用上下文里有不同错误后果。

### 8.6 CLOG group update fallback

`TransactionGroupUpdateXidStatus()` 可能拒绝加入 group。常见原因：

- group leader 的 CLOG page 与当前 page 不同。
- 子事务数量太多。
- `MyProc` 的 subxid cache 与传入数组不匹配。
- 所有 XID 不在同一 page。

fallback 是普通拿 SLRU bank lock 自己更新。这说明 group update 不是语义依赖。

它只是 contention 优化。正确性不能依赖 group 一定成功。

### 8.7 异步 commit 与 write-in-progress CLOG 页

异步 commit 更新 CLOG 时传入有效 LSN。`SimpleLruReadPage()` 的 `write_ok` 变成 false。

如果目标页正在写出，必须等 I/O 完成。否则新 committed bit 可能被正在进行的写盘带走。

此时 commit WAL 还没 flush。这会破坏 WAL-before-CLOG-page-write。

同步 commit 和 abort 不需要这个等待。同步 commit 已经 flush WAL。

abort 即使丢失也默认 aborted。

### 8.8 SLRU 写失败

`SlruInternalWritePage()` 写前把页标成 `WRITE_IN_PROGRESS` 并清 `page_dirty`。如果 `SlruPhysicalWritePage()` 失败，返回后会重新设置 `page_dirty = true`。

然后才报告错误。这样 shared memory 中不会误以为脏页已经成功写出。

如果写页前需要 `XLogFlush(max_lsn)`，代码进入 critical section。注释说如果这个 flush 失败必须 PANIC。

因为这里不能用普通 ERROR 留下半清理共享状态。

### 8.9 CLOG 读取旧段缺失

`SlruPhysicalReadPage()` 在 recovery 中允许某些文件不存在。它会把不存在的页读成 zeroes。

注释说明 crash/restart 后可能收到设置已截断 CLOG 段中事务状态的 WAL 命令。实现选择接受这种引用。

这是 recovery robustness 的 fallback。正常运行中同样的 ENOENT 会报错。

### 8.10 缺失 `pg_subtrans` 父链

`TransactionIdDidCommit()` 看到 `SUB_COMMITTED` 会查父事务。如果 XID 早于 `TransactionXmin`，不能再查 `pg_subtrans`。

它会保守返回 false。如果父 XID 不存在，会 WARNING 并返回 false。

`TransactionIdDidAbort()` 中类似情况会保守返回 true。这是因为不能证明 committed 时不能把它当 committed。

可见性 correctness 优先于精确诊断。

### 8.11 recovery 中的 commit replay

`xact_redo_commit()` 在非 hot standby 状态下直接 `TransactionIdCommitTree()`。在 standby 相关状态下，它使用 `TransactionIdAsyncCommitTree(xid, ..., lsn)`。

注释说明这用于 hint bit 安全。直到 `minRecoveryPoint` 越过 commit record 前，不应该设置可能在 crash 后误导系统的 hint bit。

这说明 async commit protocol 不只服务用户设置 `synchronous_commit=off`。它也是 recovery/hint-bit 边界的复用机制。

## 9. 成本、资源与跨模块传播

### 9.1 前台 commit latency

同步提交的主要延迟来自 WAL flush。路径包括：

- 等待正在进行的 WAL insertion。
- 等待或获取 `WALWriteLock`。
- WAL write。
- WAL fsync。
- 可能的 `commit_delay`。
- 同步复制等待。

在高并发下，单个事务不一定独占一次 fsync。`XLogFlush()` 会让其他 backend join backlog。

持有 `WALWriteLock` 的 backend 可能 flush 超过自己的目标 LSN。吞吐提升的代价是个别事务 latency 变动更大。

### 9.2 CLOG bank lock contention

CLOG 每个事务只占两位。但最新 XID 集中落在少数 CLOG page。

大量小事务并发提交时会争用同一个 SLRU bank lock。`TransactionGroupUpdateXidStatus()` 减少锁交接。

但它只适用于同页、低子事务数量、`MyProc` 状态匹配的情况。大量子事务或跨页提交会回到普通路径。

这时成本会随子事务数量和涉及 CLOG page 数增长。

### 9.3 SLRU buffer pressure

`transaction_buffers` 控制 transaction SLRU buffer pool。太小会增加 `pg_xact` SLRU miss、read、write。

最新 CLOG page 受到 `latest_page_number` 保护，通常不被优先 evict。但长时间高 XID churn 仍会让旧页被换出。

换出 dirty page 可能触发 WAL flush。所以 async commit 的成本可能从前台 commit 转移到：

- WAL writer。
- checkpoint。
- 后续访问 CLOG 的 backend。
- SLRU eviction 路径。

### 9.4 WAL bandwidth 与 fsync 合并

WAL flush 是实例级共享资源。`logWriteResult` 和 `logFlushResult` 是全局进度。

一个事务的 `XLogFlush()` 可能推进其他事务的 commit WAL。反过来，其他事务也可能替它完成 flush。

这就是为什么诊断 commit latency 不能只看单个 SQL。你需要同时看：

- `pg_stat_wal`
- `pg_stat_io` 中 WAL I/O
- wait event
- fsync 配置
- 存储设备 latency
- synchronous replication 状态

### 9.5 Hint bit 延迟成本

异步 commit 可能让 committed XID 暂时不能设置 hint bit。未来可见性检查仍然需要查 CLOG。

如果扫描大量刚由 async commit 产生的 tuple，CLOG lookup 成本可能上升。这个成本不会直接显示为 “commit 慢”。

它会传播到后续读路径。这是 runtime 资源传播的一种典型形态：

- 前台 commit latency 降低。
- 后续读路径或 SLRU 写页承担部分成本。

### 9.6 Checkpoint 与 CLOG

checkpoint 会调用 `CheckPointCLOG()`。它写出 dirty CLOG pages。

如果这些页包含 async commit LSN，写页前会补 WAL flush。因此 checkpoint 可能间接承担 async commit durability debt。

同时 `DELAY_CHKPT_IN_COMMIT` 会让 checkpoint 避免跨过 commit critical section。这不是简单的后台刷盘。

它参与提交正确性边界。

### 9.7 与 autovacuum 和 xid horizon 的传播

CLOG truncation 依赖系统不再需要旧 XID 状态。长事务、replication slot、prepared transaction、hot standby feedback 都可能拖住 horizon。

虽然本节不展开 horizon 计算，但要知道：

- commit ordering 决定单个事务状态何时安全。
- xid horizon 决定旧状态何时可删除。

如果 horizon 被拖住，`pg_xact` 文件保留更久。这会增加磁盘占用和 SLRU 工作集。

## 10. 观测与诊断入口

### 10.1 可以直接观测的状态

`pg_stat_wal` 能看到实例级 WAL 活动：

- `wal_records`
- `wal_fpi`
- `wal_bytes`
- `wal_fpi_bytes`
- `wal_buffers_full`

它不能告诉你某个事务的 commit record 是否已经 flush。它是累计统计。

`pg_stat_slru` 能看到 SLRU 活动。对本节重点看：

```sql
SELECT *
FROM pg_stat_slru
WHERE name = 'transaction';
```

列包括：

- `blks_zeroed`
- `blks_hit`
- `blks_read`
- `blks_written`
- `blks_exists`
- `flushes`
- `truncates`

它能说明 transaction SLRU 压力。它不能直接显示 `group_lsn[]`。

`pg_stat_activity` 能看到 wait event。相关 wait event 包括：

- `WALWrite`
- `WALSync`
- `CommitDelay`
- `XactGroupUpdate`
- `ProcArrayGroupUpdate`
- `XactSLRU`
- `XactBuffer`

wait event 是瞬时状态。它不是完整因果归因。

### 10.2 只能间接推断的状态

commit record LSN 通常不能通过普通 SQL 直接拿到。`XactLastRecEnd` 是 backend-local 内部状态。

`XLogCtl->asyncXactLSN` 是共享内存内部状态。`CLOG group_lsn[]` 也是共享内存内部状态。

你可以通过这些现象间接推断：

- `synchronous_commit=off` 下 commit latency 明显下降。
- 后续 checkpoint 或 SLRU write 可能出现 WAL flush。
- `pg_stat_slru` 中 transaction `blks_written`、`flushes` 增长。
- 大量刚提交 tuple 的扫描仍需要查 CLOG，hint bit 不一定立即写入。

但不要把这些现象反推为单一原因。WAL writer、checkpoint、存储设备、同步复制、workload shape 都会影响结果。

### 10.3 需要 gdb 或断点的状态

如果要确认顺序，可以在源码中断：

- `RecordTransactionCommit()`
- `XactLogCommitRecord()`
- `XLogFlush()`
- `TransactionIdSetStatusBit()`
- `TransactionGroupUpdateXidStatus()`
- `SlruPhysicalWritePage()`
- `ProcArrayEndTransaction()`
- `SetHintBitsExt()`

推荐观察变量：

- `xid`
- `nchildren`
- `XactLastRecEnd`
- `wrote_xlog`
- `synchronous_commit`
- `forceSyncCommit`
- `nrels`
- `lsn`
- `XactCtl->shared->group_lsn[...]`
- `LogwrtResult.Flush`
- `MyProc->delayChkptFlags`

不要在 production 直接 attach 并随意停住 commit path。这会影响全局锁、WAL flush 和 ProcArray。

实验环境中才这样做。

### 10.4 一个 runtime truth

本节锚定的 runtime truth：

- `synchronous_commit=off` 的事务可以在 commit WAL fsync 前对其他 backend 可见，但系统仍然防止包含该 committed 状态的 CLOG page 或 heap committed hint bit 先于 commit WAL 安全持久化。

你可以这样解释它：

- 可见性是运行时语义。
- 持久性是 crash 后语义。
- 异步提交允许这两者短暂分离。
- `group_lsn[]` 和 hint bit LSN check 把分离限制在可恢复范围内。

能看到：

- commit latency 变化。
- WAL/SLRU 统计变化。
- wait event。

看不到：

- 单个 CLOG slot 的 `group_lsn[]`。
- 某个 XID 的 exact commit record LSN。
- 某次 hint bit 被跳过的直接 SQL 指标。

需要推断或断点：

- CLOG 页写盘前是否补 `XLogFlush(max_lsn)`。
- `SetHintBitsExt()` 是否因 commit LSN 未 flush 而返回。

## 11. 常见误区

误区一：

- `XLogInsert()` 返回了 LSN，所以事务已经持久。

纠正：

- `XLogInsert()` 只是把 record 插入 WAL buffers。
- `XLogFlush()` 才是本地 WAL durable 边界。

误区二：

- CLOG 中看到 committed，就说明 commit WAL 已 fsync。

纠正：

- 异步 commit 会先更新内存 CLOG。
- 对应 commit WAL 可能还没有 fsync。
- CLOG 页落盘前会通过 `group_lsn[]` 补 flush。

误区三：

- `synchronous_commit=off` 等于不写 WAL。

纠正：

- commit WAL 仍然写入 WAL buffers。
- WAL writer 或后续 flush 会推进它。
- crash 前未 flush 的 async commit 允许丢失。

误区四：

- abort 也应该像 commit 一样先 flush WAL 再改 CLOG。

纠正：

- crash 后未知事务默认不是 committed。
- 丢失 abort record 不会错误制造 committed 事务。
- 所以 abort CLOG 更新可以不等 abort WAL flush。

误区五：

- 子事务 CLOG bit 是独立最终状态。

纠正：

- `SUB_COMMITTED` 需要查父事务。
- 跨 CLOG 页提交时，子事务状态是分阶段更新的。
- 主事务 commit 才是整棵事务树的原子语义中心。

误区六：

- wait event 可以完整解释 commit 慢。

纠正：

- wait event 只显示采样瞬间等待点。
- commit latency 还可能来自 group commit、sync replication、checkpoint、storage fsync、SLRU 写页和 OS 调度。

误区七：

- hint bit 只是读优化，不影响持久化边界。

纠正：

- committed hint bit 如果先于 commit WAL 持久，会污染 crash 后判断。
- 因此 hint bit 设置必须检查 commit LSN。

## 12. 课堂实验

实验 1：比较同步与异步提交。用 `pgbench` 或小事务循环分别跑 `synchronous_commit=on` 和 `off`，观察 `pg_stat_wal`、`pg_stat_slru where name='transaction'` 和 `pg_stat_activity.wait_event`。

预期是异步提交降低前台等待，但仍生成 commit WAL，仍更新 CLOG。把现象回扣到 `RecordTransactionCommit()`：同步分支调用 `XLogFlush(XactLastRecEnd)`，异步分支调用 `XLogSetAsyncXactLSN()` 和 `TransactionIdAsyncCommitTree()`。

实验 2：跟踪 CLOG 写页前的 WAL flush。断点放在 `TransactionIdAsyncCommitTree`、`TransactionIdSetStatusBit`、`SlruPhysicalWritePage`、`XLogFlush`。

执行一次 `synchronous_commit=off` 的写事务，再触发 `CHECKPOINT`。观察 `group_lsn[]` 如何把 async commit 的 WAL 依赖延后到 CLOG page 写出前。

实验 3：观察 committed hint bit 的 LSN 约束。断 `SetHintBitsExt`、`TransactionIdGetCommitLSN`、`XLogNeedsFlush`。

让另一个会话读取刚提交的 tuple，比较“运行时可见”与“hint bit 可以安全落盘”的差别。结论应是：CLOG committed 可以让读者看到 tuple；heap page 上的 committed hint 还要满足额外 WAL 安全边界。

## 13. 讨论题

1. 为什么同步提交路径必须在 `TransactionIdCommitTree()` 之前调用 `XLogFlush(XactLastRecEnd)`？
2. 异步提交已经把 CLOG 内存状态设为 committed，为什么 crash 后仍然允许事务丢失？
3. `group_lsn[]` 为什么按一组 XID 存最高 LSN，而不是每个 XID 存精确 LSN？这带来什么性能收益和什么保守等待？
4. 为什么事务必须先更新 CLOG，再调用 `ProcArrayEndTransaction()` 从 running set 消失？
5. 如果一个子事务在 CLOG 中是 `SUB_COMMITTED`，visibility 代码为什么不能直接把它当 committed？
6. `RecordTransactionAbort()` 为什么不需要像 commit 一样等待 WAL flush？
7. 哪些现象可以通过 `pg_stat_wal`、`pg_stat_slru`、wait event 观察，哪些必须靠 gdb、日志或源码插桩？
8. 如果 workload 由大量小事务组成，瓶颈可能在 WAL fsync、CLOG bank lock、ProcArrayLock、hint bit 延迟之间如何迁移？

## 14. 本节小结

本节唯一主问题是：

- PostgreSQL 如何安排 commit 返回、可见性、WAL 持久化和 CLOG 持久化之间的顺序？

核心链路是：

```text
CommitTransaction()
  -> RecordTransactionCommit()
  -> XactLogCommitRecord()
  -> XLogInsert()
  -> XLogFlush() or XLogSetAsyncXactLSN()
  -> TransactionIdCommitTree() or TransactionIdAsyncCommitTree()
  -> ProcArrayEndTransaction()
```

同步提交的核心不变量：

- commit WAL flush 完成后，才能把 CLOG 设置为 committed。

异步提交的核心不变量：

- 可以先返回并更新内存 CLOG。
- 但 CLOG page 写盘前必须根据 `group_lsn[]` flush commit WAL。
- committed hint bit 写入 heap page 前也要检查 commit LSN。

核心状态边界：

- `XactLastRecEnd` 是 commit record end LSN，不是持久标志。
- `logFlushResult` 才表达 WAL flush 进度。
- CLOG bit 表达事务完成状态，但异步提交下还需要 `group_lsn[]` 保护落盘。
- ProcArray 表达 running set，必须在 CLOG 更新后清除。
- hint bit 是 heap 页上的派生状态，不能绕过 WAL 持久化边界。

ownership 与 cleanup：

- 当前 backend 创建 commit record 并更新事务状态。
- WAL buffers、CLOG SLRU buffers、ProcArray 都是 shared state。
- WAL writer、checkpointer、其他 backend 都可能推进某些持久化债务。
- `ResourceOwner` 和 `AtEOXact_*` 负责 post-commit cleanup。
- post-commit cleanup 太晚，不能再改变事务提交事实。

错误路径：

- 无 XID 事务不进普通 commit CLOG 路径。
- relation deletes 和 force sync 会把事务推回同步 flush。
- abort 不需要 commit 等价的 WAL flush，因为 crash 后默认不是 committed。
- CLOG group update 失败会 fallback 到普通单事务更新。
- SLRU 写页失败会恢复 dirty 标志再报错。
- critical section 中的 WAL flush 问题会升级为严重错误。

观测边界：

- `pg_stat_wal` 看 WAL 累计活动。
- `pg_stat_slru` 看 transaction SLRU 压力。
- `pg_stat_activity.wait_event` 看瞬时等待。
- 单个 commit LSN、`asyncXactLSN`、CLOG `group_lsn[]` 通常需要断点或插桩。

可迁移 mental model：

- 内核里的 durable state 很少是单字段事实。
- 它通常是 “状态 bit + LSN + flush 边界 + shared lock + cleanup 顺序” 的组合。
- 为了降低 hot path 延迟，系统可以允许运行时语义先行。
- 但只要会落盘，就必须用 LSN 或等价机制把 crash 后语义补齐。

版本和 workload 边界：

- 本课基于指定 PostgreSQL 源码基线。
- `transaction_buffers`、WAL writer 策略、wait event 名称和统计列可能随版本演进。
- commit latency 的瓶颈强依赖存储设备、fsync 方法、并发事务数、子事务数量、同步复制配置和 checkpoint 行为。
- 诊断时不要把单个统计指标解释成完整因果链。

下一节可以继续追问：

- 当 snapshot 读取 ProcArray、CLOG、subtrans 和 hint bit 时，如何把这些分层状态压缩成一条 tuple visibility 结论。
