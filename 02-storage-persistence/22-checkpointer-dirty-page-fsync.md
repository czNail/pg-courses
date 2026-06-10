# PostgreSQL Checkpointer 脏页扫描、写回与 fsync

## 课程定位
本节需要你已经理解 shared buffer 的 tag、pin、content lock、dirty bit、page LSN、WAL-before-data，以及上一节 checkpoint redo pointer 的基本语义。
本节只回答一个主问题：
PostgreSQL 如何在业务 backend 持续制造新脏页时，为一次 checkpoint 定义一个有限的脏页集合，并把这个集合安全推进到 write 和 fsync 完成？
核心矛盾是：
checkpoint 必须给 crash recovery 一个已经持久化的数据边界；
但在线系统不能为了得到这个边界而长期阻塞所有新的 buffer 修改和 WAL 插入。
学完后，你应该能独立判断：
- `buffers_written` 高，到底说明 checkpoint 写了很多页，还是 bgwriter/backend 已经提前写掉了一部分。
- `sync_time` 高，到底是 data-file fsync、WAL fsync、checkpoint write pacing，还是 backend queue fallback。
- 一个 buffer 的 `BM_DIRTY`、`BM_CHECKPOINT_NEEDED`、`BM_IO_IN_PROGRESS` 分别处在什么生命周期。
- 为什么 checkpoint 期间新变脏的页通常不属于本轮 checkpoint 的责任。
- 为什么数据页 `write()` 成功以后，还必须有后续 fsync 请求队列和 sync 阶段。
本课基于本地源码 `/home/nail/postgres-lab`，commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。
重点源码文件是：
- `src/backend/postmaster/checkpointer.c`
- `src/backend/storage/buffer/bufmgr.c`
- `src/backend/storage/sync/sync.c`
- `src/backend/access/transam/xlog.c`
- `src/include/storage/buf_internals.h`
本节不是 checkpointer 源码百科。
我们只沿一条主链路走：
backend 写脏页和登记 fsync 责任；
checkpointer 接受 checkpoint 请求；
`CreateCheckPoint()` 固定 redo 边界；
`BufferSync()` 扫描并写回本轮脏页集合；
`ProcessSyncRequests()` fsync 已写文件；
最后 checkpoint WAL record 和 control file 把边界发布给 crash recovery。

## 1. 本节在总主线中的位置
前面的存储持久化课程已经拆过几个相邻问题。
buffer tag 和 descriptor 说明了“一个 shared buffer 指向哪个 relation fork block”。
buffer lookup、clock sweep、pin 和 content lock 说明了“这个 buffer 如何被并发访问和替换”。
dirty page LSN 和 WAL-before-data 说明了“为什么数据页落盘之前必须先保证 WAL 落盘到 page LSN”。
fsync request queue 说明了“backend 写出 relation segment 后，如何把 fsync 责任交给 checkpointer”。
checkpoint lifecycle redo pointer 说明了“checkpoint 记录如何成为 recovery 起点”。
本节把这些线接起来。
它不再问“checkpoint 记录里有什么”。
它问的是：
当 checkpoint 已经决定了一个 redo 起点，buffer pool 里哪些页必须被写回？
这些页写回后，哪些文件必须 fsync？
这些责任如何在 backend、bgwriter、checkpointer、WAL 层和 OS page cache 之间传播？
为什么一次 checkpoint 不会无限追赶不断新增的 dirty page？
这个问题比“checkpointer 做什么”更窄，也更适合源码阅读。
因为真正容易出错的地方不是函数名，而是边界：
- dirty bit 何时进入本轮责任集合。
- buffer write 和 file fsync 为什么是两段。
- fsync 请求何时被吸收到 checkpointer 的本地 hash。
- 新请求为什么留给下一轮。
- 异常时哪些状态可以重试，哪些失败必须升级。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
checkpoint 先用 redo pointer 和 `BM_CHECKPOINT_NEEDED` 给本轮建立一个有限集合，再通过 buffer write、fsync request absorption、pendingOps cycle counter 和 checkpoint WAL/control file 顺序，把这个集合推进到 crash-safe。
这里有两个“有限集合”。
第一个有限集合是 buffer 层的集合。
`BufferSync()` 扫描 `NBuffers`，把 checkpoint 开始时符合条件的 dirty permanent buffer 标记为 `BM_CHECKPOINT_NEEDED`。
后续写回循环只追这个标记。
checkpoint 期间新变脏的 buffer 可以继续存在，但通常没有 `BM_CHECKPOINT_NEEDED`，所以属于下一轮。
第二个有限集合是 sync 层的集合。
relation segment 被 `smgrwrite()` 写过以后，会形成 `FileTag` 粒度的 sync request。
checkpointer 把共享 ring buffer 里的 request 吸收到自己的 `pendingOps` hash。
`ProcessSyncRequests()` 增加 `sync_cycle_ctr`，只处理本轮之前已经存在的 fsync entry。
处理过程中新增的 fsync entry 会带新 cycle，留给下一轮。
这两个有限集合解决同一个矛盾：
系统要给 crash recovery 一个确定的“此前修改已经可以从数据文件看到”的承诺。
但 PostgreSQL 又允许 checkpoint 期间继续有新 WAL、新 dirty page、新 relation segment write。
如果没有边界，checkpoint 可能永远完成不了。
如果边界过强，所有写入都要停下等待 checkpoint。
PostgreSQL 的选择是：
用 redo pointer 确定 recovery 起点；
用 dirty-before-checkpoint 标记确定本轮 buffer 写回责任；
用 fsync request cycle 确定本轮文件持久化责任；
用 WAL-before-data 和 final checkpoint record flush 确定 crash safety 顺序。
这不是单个锁能解决的问题。
它是多个状态机按顺序拼出来的正确性边界。

## 3. 核心文件分工与阅读顺序
本课建议按下面顺序读源码，而不是按文件名排序。

| 顺序 | 文件 | 入口 | 本节只关心的职责 |
| --- | --- | --- | --- |
| 1 | `src/include/storage/buf_internals.h` | `BufferDesc`、`BM_*` flags | 定义 buffer identity、dirty、checkpoint-needed、I/O in-progress 等共享状态 |
| 2 | `src/backend/postmaster/checkpointer.c` | `RequestCheckpoint()`、`CheckpointerMain()` | 接受 checkpoint 请求、合并 flags、等待开始/完成、接收 backend fsync requests |
| 3 | `src/backend/access/transam/xlog.c` | `CreateCheckPoint()`、`CheckPointGuts()` | 决定 redo pointer、组织 write/sync 顺序、写 checkpoint WAL record、更新 control file |
| 4 | `src/backend/storage/buffer/bufmgr.c` | `CheckPointBuffers()`、`BufferSync()`、`SyncOneBuffer()`、`FlushBuffer()` | 扫描脏页、打本轮标记、写出 buffer、清 dirty/checkpoint-needed |
| 5 | `src/backend/storage/sync/sync.c` | `SyncPreCheckpoint()`、`AbsorbSyncRequests()`、`ProcessSyncRequests()` | 将 backend fsync 请求吸收到 `pendingOps`，按 cycle 处理 data-file fsync |
| 6 | `src/backend/storage/smgr/md.c` | `register_dirty_segment()`、`mdsyncfiletag()` | 解释 `smgrwrite()` 之后 fsync request 从哪里来，以及队列满时的 backend fallback |
虽然用户常把这些都叫“checkpoint 写盘”，源码里至少有四个不同动作。
第一，buffer write。
这是 `smgrwrite()` 把 8KB page 内容写给内核。
它不等于持久化。
第二，writeback hint。
`checkpoint_flush_after` 通过 `WritebackContext` 和 `smgrwriteback()` 给 OS 调度 hint。
它也不等于持久化。
第三，data-file fsync。
`ProcessSyncRequests()` 对 relation segment、CLOG、multixact 等 file tag 调用各自 sync handler。
这是 data file 层面的 durable boundary。
第四，WAL fsync。
`FlushBuffer()` 会在写 permanent buffer 前 `XLogFlush(page_lsn)`。
`CreateCheckPoint()` 最后会 `XLogFlush(checkpoint_record_lsn)`。
`issue_xlog_fsync()` 失败直接是 WAL durability 层面的严重错误。
读源码时要保留这些不整齐的边界。
PostgreSQL 不是“一个 checkpoint 函数把所有东西刷干净”。
它是多个模块在不同粒度上交接责任。

## 4. 关键数据结构与状态

### 4.1 `BufferDesc.state` 不是一个 dirty bool
`src/include/storage/buf_internals.h` 中的 `BufferDesc` 是每个 shared buffer 的共享 descriptor。
它包含：
- `tag`：这个 buffer 当前代表哪个 relation fork block。
- `buf_id`：buffer pool 内固定编号。
- `state`：一个 atomic `uint64`，组合 flags、refcount、usagecount。
- `wait_backend_pgprocno`：等待 sole pin 的 backend。
- `io_wref`：异步 I/O wait reference。
- `lock_waiters`：content lock 等待队列。
本节最重要的 flags 是：

| flag | 本节语义 |
| --- | --- |
| `BM_DIRTY` | buffer 内容和磁盘上的对应 block 不一致，需要写出 |
| `BM_CHECKPOINT_NEEDED` | 本轮 checkpoint 需要这个 buffer 被写出 |
| `BM_IO_IN_PROGRESS` | 有进程拥有这个 buffer 的 I/O 权利 |
| `BM_IO_ERROR` | 上一次 I/O 失败，后续路径需要看到失败历史 |
| `BM_PERMANENT` | 该 buffer 属于需要 WAL/data durability 的永久对象 |
| `BM_VALID` | buffer 内容有效，可以被写出或读取 |
raw field 不是语义。
`BM_DIRTY` 单独只说明“这页曾被修改且还没清 dirty”。
`BM_DIRTY + BM_CHECKPOINT_NEEDED` 才说明“这页属于当前 checkpoint 的写回集合”。
`BM_DIRTY + page LSN + BM_PERMANENT` 才说明“写这页之前必须先 flush WAL 到 page LSN”。
`BM_IO_IN_PROGRESS + ResourceOwner` 才说明“某个进程正在承担这个 buffer 的 I/O cleanup 责任”。
`tag + pin` 才说明“我读到的 tag 在使用期间不会被替换掉”。

### 4.2 `BM_CHECKPOINT_NEEDED` 是本轮边界，不是持久化状态
`BM_CHECKPOINT_NEEDED` 很容易被误读。
它不是“已经写入 checkpoint record”。
它不是“已经 fsync”。
它也不是“这个页永远必须由 checkpointer 写”。
它只表示：
在本次 `BufferSync()` 的第一轮扫描里，这个 buffer 符合本轮 checkpoint 要写的条件。
如果后续 bgwriter 或 backend 先把它写干净，`TerminateBufferIO(clear_dirty=true)` 会清掉 `BM_DIRTY` 和 `BM_CHECKPOINT_NEEDED`。
如果 checkpointer 自己写它，也会清掉这两个 bit。
如果写失败，它可能留着 `BM_CHECKPOINT_NEEDED`。
源码注释明确说这没问题，因为下一次 checkpoint 尝试也一定需要写它。
所以 `BM_CHECKPOINT_NEEDED` 的 lifecycle 是：
- checkpoint 扫描发现 dirty permanent page。
- 设置 `BM_CHECKPOINT_NEEDED`。
- 任意写出者成功写出后清掉。
- 写失败时可能残留，等待下一轮。
这个 flag 的存在让 checkpoint 不必追逐“扫描后才变脏”的页。

### 4.3 `CheckpointerShmemStruct` 有两个不同责任
`checkpointer.c` 的共享内存结构同时服务两个通道。
第一个通道是 checkpoint request。
相关字段是：
- `ckpt_lck`
- `ckpt_started`
- `ckpt_done`
- `ckpt_failed`
- `ckpt_flags`
- `start_cv`
- `done_cv`
backend 调 `RequestCheckpoint()` 时 OR flags，唤醒 checkpointer。
如果带 `CHECKPOINT_WAIT`，backend 等 `ckpt_started` 变化，再等 `ckpt_done >= new_started`。
如果 `ckpt_failed` 变化，等待者报 “checkpoint request failed”。
第二个通道是 fsync request queue。
相关字段是：
- `num_requests`
- `max_requests`
- `head`
- `tail`
- `requests[]`
这些字段由 `CheckpointerCommLock` 保护。
backend 写 relation segment 后通过 `RegisterSyncRequest()` 走到 `ForwardSyncRequest()`。
它把 `FileTag + SyncRequestType` 放进这个 ring buffer。
checkpointer 在多个位置调用 `AbsorbSyncRequests()`，把 ring buffer 中的 request 拿出来，转成本地 `pendingOps` hash 状态。
这两个通道不能混为一谈。
checkpoint request 说“请开始一个 checkpoint”。
fsync request 说“这个文件段被写过，下一次 checkpoint sync 阶段必须考虑它”。
一次 checkpoint 可以没有新的外部 request，而由 timeout 或 WAL consumption 触发。
一个 backend fsync request 也不是在请求立刻 checkpoint。

### 4.4 `pendingOps` 是 checkpointer-local，不在共享内存
`sync.c` 里的 `pendingOps` 是一个 hash table。
它只在 standalone backend 或 checkpointer 进程中存在。
普通 backend 不维护自己的 pending fsync table。
普通 backend 只尝试把 request forward 给 checkpointer。
`PendingFsyncEntry` 的核心字段是：
- `tag`：文件标识，当前包括 handler、forknum、relfilelocator、segno。
- `cycle_ctr`：这个 entry 最早 request 所在的 sync cycle。
- `canceled`：是否被 forget/filter 取消。
这里的粒度不是 buffer page。
它是 file tag。
对于 relation storage，通常是 relation fork 的 segment 文件。
这就是为什么 checkpoint log 里能打印 “sync files=N”，但不能直接告诉你 “fsync 了多少 buffer page”。

### 4.5 `SyncRequestType` 不是只有 fsync
`sync.h` 定义的 request type 包括：
- `SYNC_REQUEST`：安排下次 checkpoint 调 sync handler。
- `SYNC_UNLINK_REQUEST`：安排 checkpoint 后 unlink。
- `SYNC_FORGET_REQUEST`：忘掉某个 tag 的 sync。
- `SYNC_FILTER_REQUEST`：按 match function 忘掉一组请求。
这解释了 `sync.c` 的复杂性。
它不只是一个 “fsync 文件列表”。
它还要处理 relation drop、truncate、database drop 和 tablespace 相关删除。
如果一个 fsync 请求对应的 segment 已经被 unlink，简单忽略 `ENOENT` 可能掩盖真正错误。
所以失败时要先 absorb 新来的 cancel/filter request，再决定是否重试或报错。

### 4.6 `CheckpointStatsData` 是单次 checkpoint 的局部统计
`src/include/access/xlog.h` 中的 `CheckpointStatsData` 包含：
- `ckpt_start_t`
- `ckpt_write_t`
- `ckpt_sync_t`
- `ckpt_sync_end_t`
- `ckpt_end_t`
- `ckpt_bufs_written`
- `ckpt_slru_written`
- `ckpt_segs_added`
- `ckpt_segs_removed`
- `ckpt_segs_recycled`
- `ckpt_sync_rels`
- `ckpt_longest_sync`
- `ckpt_agg_sync_time`
这些不是直接暴露给 SQL 的共享实时状态。
`LogCheckpointEnd()` 会用它们生成 `log_checkpoints` 的单次 checkpoint 日志。
`PendingCheckpointerStats` 会累计 `write_time`、`sync_time`、`buffers_written` 等，然后由 `pgstat_report_checkpointer()` 汇总到 cumulative stats。
`pg_stat_checkpointer` 是累计视图。
它不能还原每一次 checkpoint 的全部内部状态。

### 4.7 `WritebackContext` 是 OS 调度 hint，不是 fsync 责任
`WritebackContext` 保存 pending writeback tags。
`ScheduleBufferTagForWriteback()` 把写过的 buffer tag 收集起来。
`IssuePendingWritebacks()` 排序、合并相邻 blocks，然后调用 `smgrwriteback()`。
这最终走到 `FileWriteback()`，通常对应让 OS 尽早开始回写 dirty page cache。
它的注释强调：
writeback 只是改善 OS I/O scheduling，尽量不报错。
它不是 durability boundary。
真正 checkpoint 是否 durable，要看后续 `ProcessSyncRequests()` 里的 fsync。

## 5. 主流程源码 walkthrough
下面沿一次 online checkpoint 的主流程走。
为了减少跳转，先看总调用链。

```text
backend / WAL pressure / SQL CHECKPOINT
  -> RequestCheckpoint()
  -> CheckpointerMain()
     -> CreateCheckPoint()
        -> SyncPreCheckpoint()
        -> insert XLOG_CHECKPOINT_REDO, choose redo pointer
        -> CheckPointGuts()
           -> CheckPointBuffers()
           -> ProcessSyncRequests()
              -> AbsorbSyncRequests()
              -> mdsyncfiletag() / other sync handlers
        -> insert and XLogFlush(XLOG_CHECKPOINT_ONLINE)
        -> UpdateControlFile()
        -> SyncPostCheckpoint()
```

脏页写出子链路再展开：

```text
CheckPointBuffers()
  -> BufferSync()
     -> mark BM_CHECKPOINT_NEEDED
     -> sort and balance writes
     -> SyncOneBuffer()
        -> FlushUnlockedBuffer()
           -> FlushBuffer()
              -> StartSharedBufferIO()
              -> XLogFlush(page_lsn)
              -> smgrwrite(skipFsync=false)
              -> register_dirty_segment()
              -> TerminateBufferIO(clear_dirty=true)
        -> ScheduleBufferTagForWriteback()
  -> IssuePendingWritebacks()
```
这条链路有一个容易被忽略的事实：
`smgrwrite()` 发生在 buffer write 阶段。
`mdsyncfiletag()` 发生在 sync 阶段。
两者中间可能有很多新 backend 继续写入、继续提交、继续产生 WAL。
checkpoint 能完成，是因为它不把所有“此刻新发生的一切”都纳入本轮。

### 5.1 backend 先制造脏页
普通数据页修改通常先持有 buffer pin 和 content lock。
重要修改会写 WAL。
`MarkBufferDirty()` 要求 shared buffer 被 pin，并持有 exclusive content lock。
它通过 atomic CAS 设置 `BM_DIRTY`。
hint bit 的路径 `MarkSharedBufferDirtyHint()` 也强调一个顺序：
如果需要 WAL full-page image 来保护 hint bit，它必须先标脏，再 `XLogInsert()`。
源码注释说明原因：
如果先写 WAL 后标脏，checkpoint 可能在两者之间开始。
那样 checkpoint 的 redo pointer 可能认为这个 WAL record 不需要 replay，却没有把对应 page 纳入写回。
所以本节第一个不变量是：
会影响 WAL recovery 的 page 修改，必须在可被 checkpoint 漏掉之前进入 dirty 状态。
注意这不意味着 checkpoint 立刻写它。
它只意味着后续 `BufferSync()` 的 dirty scan 有机会正确分类它。

### 5.2 backend 或 WAL 层发起 checkpoint request
checkpoint request 有多种来源。
手工 `CHECKPOINT` 走 `ExecCheckpoint()`，再走 `RequestCheckpoint()`。
WAL segment 消耗过多时，`XLogWrite()` 在 segment end 附近可能调用 `RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)`。
checkpointer 主循环自己也会因为 `checkpoint_timeout` 到期而做 timed checkpoint。
`RequestCheckpoint()` 的共享状态操作很短。
它在 `ckpt_lck` 下读取 `old_failed`、`old_started`，把 flags OR 进 `ckpt_flags`。
然后它通过 checkpointer proc latch 唤醒 checkpointer。
如果请求者带 `CHECKPOINT_WAIT`，它等两个阶段：
第一，等 `ckpt_started` 变化。
这只说明某个 checkpoint 已经开始，并且已经看到了 request flags。
第二，等 `ckpt_done >= new_started`。
这说明对应 started counter 的 checkpoint 已完成。
最后比较 `ckpt_failed`。
如果失败计数变化，backend 报 checkpoint request failed。
这里没有把等待者和某个 checkpoint object 做一对一绑定。
源码依赖 counter 和 modulo arithmetic。
这是一个低成本共享状态协议。

### 5.3 `CheckpointerMain()` 合并请求并开始 checkpoint
checkpointer 主循环醒来后，先 `AbsorbSyncRequests()`，再处理信号和配置 reload。
它检查 `ckpt_flags` 是否非零，也检查 timeout。
决定要 checkpoint 后，它在 `ckpt_lck` 下：
- 把 `CheckpointerShmem->ckpt_flags` OR 到本地 `flags`。
- 清空共享 `ckpt_flags`。
- 增加 `ckpt_started`。
然后广播 `start_cv`。
等待 checkpoint start 的 backend 可以继续进入第二阶段等待。
checkpointer 同时设置进程私有状态：
- `ckpt_active = true`
- `ckpt_start_recptr = GetInsertRecPtr()` 或 recovery 下的 replay ptr
- `ckpt_start_time = now`
- `ckpt_cached_elapsed = 0`
这些状态只用于本次 checkpoint 的 pacing。
它们不是 crash safety 状态。
它们服务 `CheckpointWriteDelay()` 中的“是否按进度睡眠”判断。

### 5.4 `CreateCheckPoint()` 先做 pre-checkpoint，再固定 redo
`CreateCheckPoint()` 一开始清空 `CheckpointStats`，记录 `ckpt_start_t`。
然后调用 `SyncPreCheckpoint()`。
这个调用必须在确定 checkpoint redo pointer 之前发生。
原因不是数据页本身，而是 unlink request 的 cycle 边界。
`SyncPreCheckpoint()` 会先 `AbsorbSyncRequests()`，确保 checkpoint 开始前已经 forward 的 unlink request 被本轮看到。
然后它增加 `checkpoint_cycle_ctr`。
后面到 `SyncPostCheckpoint()` 时，只处理旧 cycle 的 unlink。
这保证不会把 checkpoint 期间才来的删除请求提前执行。
随后 `CreateCheckPoint()` 进入 critical section，检查是否可以跳过 idle checkpoint。
online checkpoint 和 shutdown checkpoint 的 redo 处理不同。
online checkpoint 会插入一个 `XLOG_CHECKPOINT_REDO` record。
这个 record 的起始 LSN 成为新的 redo pointer。
插入后，WAL insert 可以继续。
这正是 online checkpoint 能并发运行的关键。
本轮 checkpoint 只承诺：
redo pointer 之前需要的数据状态最终会被写盘并 fsync。
redo pointer 之后的新修改仍然靠 WAL recovery。

### 5.5 checkpoint 等待 commit critical section 边界
在真正刷 buffer 之前，`CreateCheckPoint()` 会检查 delaying checkpoint 的 virtual transactions。
代码使用 `GetVirtualXIDsDelayingChkpt()` 和 `HaveVirtualXIDsDelayingChkpt()`。
它等待 `DELAY_CHKPT_START`。
核心例子是事务提交。
如果某事务刚好在 redo pointer 前插入 commit WAL record，但 pg_xact 更新还在 commit critical section 中，那么 crash recovery 从 redo pointer 开始不会 replay 那条 commit record。
checkpoint 必须确保 pg_xact 更新已经会被刷下去。
所以 checkpoint 需要等待这些短 critical section 结束。
等待期间仍然调用 `AbsorbSyncRequests()`。
源码注释指出，不吸收可能造成死锁：
阻塞 checkpoint 的 backend 可能正试图把 fsync request 放进 checkpointer 队列。
如果队列满而 checkpointer 不吸收，它自己也无法继续。
这个细节说明：
checkpoint 延迟等待不是静态睡眠。
它还要继续推进 fsync request ownership。

### 5.6 `CheckPointGuts()` 把 write 阶段和 sync 阶段分开
`CheckPointGuts()` 是本节主链路的中段。
它先 checkpoint 一组非 buffer pool 的状态：
- relation map
- replication slots
- snapbuild
- logical rewrite
- replication origin
- CLOG
- CommitTs
- SUBTRANS
- MultiXact
- predicate lock state
然后调用 `CheckPointBuffers(flags)`。
这进入 shared buffer pool 写回。
写回完成后，`CheckPointGuts()` 调用 `ProcessSyncRequests()`。
这才进入 queued fsync 阶段。
这两个 timestamp 也被分开记录：
- `ckpt_write_t`：开始写脏 buffer 和 SLRU。
- `ckpt_sync_t`：开始 fsync。
- `ckpt_sync_end_t`：结束 fsync。
`log_checkpoints` 和 `pg_stat_checkpointer` 的 `write_time`、`sync_time` 正是沿这个边界来的。

### 5.7 `BufferSync()` 第一遍扫描 `NBuffers`
`BufferSync()` 首先决定 mask。
普通 online checkpoint 只写 permanent dirty buffers。
shutdown、end-of-recovery 或显式 flush unlogged 时，会包含 unlogged dirty buffers。
然后它扫描整个 buffer pool。
对每个 `buf_id`：
- 取 `BufferDesc`。
- `LockBufHdr()`。
- 检查 `(state & mask) == mask`。
- 如果符合，设置 `BM_CHECKPOINT_NEEDED`。
- 把 `buf_id` 和 tag 中的 tablespace、rel、fork、block 放入 `CkptBufferIds`。
- 解锁 buffer header。
这里有两个成本和两个正确性点。
第一个成本是 `O(NBuffers)`。
即使 dirty page 很少，也要扫描整个 shared buffer pool。
第二个成本是记录和排序 `num_to_scan` 个候选项。
dirty page 越多，后续排序和写回越重。
第一个正确性点是：
header lock 足够检查 dirty bit 和设置 checkpoint-needed bit。
这里不需要 content lock，因为还不读写 page 内容。
第二个正确性点是：
`BM_CHECKPOINT_NEEDED` 只给扫描时已经 dirty 的 page 打标。
扫描后新 dirty 的 page 不会自动带这个 bit。
所以 checkpoint 的工作集合是有边界的。

### 5.8 `BufferSync()` 排序并按 tablespace 平衡写
扫描得到候选 buffer 后，`BufferSync()` 调用 `sort_checkpoint_bufferids()`。
排序目标是降低随机 I/O。
候选项里带有 tablespace、relation、fork、block。
单纯按物理位置排序可能导致先集中写一个 tablespace，再写另一个 tablespace。
在多 tablespace 或不同底层设备上，这可能造成某个设备被集中打满，而其他设备空闲。
因此代码又构建 per-tablespace 状态 `CkptTsStatus`，用 binary heap 在 tablespace 间平衡进度。
这不是 correctness 必需。
这是资源传播控制。
它试图让 checkpoint write pressure 分布得更平滑。

### 5.9 `SyncOneBuffer()` 重新检查状态
写某个 buffer 前，`BufferSync()` 先用 atomic read 快速看 `BM_CHECKPOINT_NEEDED`。
这个检查有 race，但源码认为可接受。
如果刚检查后别人已经写掉并清掉 flag，`SyncOneBuffer()` 会发现 clean 后什么都不做。
如果极少数情况下 buffer 被写掉、替换成另一个 page、又被 dirty，`SyncOneBuffer()` 可能写一个并非本轮真正需要的 page。
源码认为这不值得额外防护。
因为多写一个 page 不破坏正确性。
`SyncOneBuffer()` 内部先准备 private refcount entry 和 resource owner 空间。
它拿 buffer header lock。
如果 refcount 和 usagecount 都是 0，它标记 `BUF_REUSABLE`。
如果 caller 要跳过 recently-used buffer，并且该 buffer 不可复用，它返回。
checkpointer 调 `SyncOneBuffer(buf_id, false, ...)`，不会因为 recently-used 跳过。
然后它检查：
- 必须 `BM_VALID`。
- 必须 `BM_DIRTY`。
不满足则解锁返回。
满足时，`PinBuffer_Locked()` 在持 header lock 情况下 pin 住 buffer。
接着走 `FlushUnlockedBuffer()`。

### 5.10 `FlushUnlockedBuffer()` 获取 content lock
`FlushUnlockedBuffer()` 很小，但语义重要。
它根据 buffer id 获取 `Buffer`，然后拿 `BUFFER_LOCK_SHARE_EXCLUSIVE`。
再调用 `FlushBuffer()`。
最后释放 content lock。
为什么这里不是只靠 header lock？
因为 header lock 保护的是 descriptor state 和 tag 变更。
它不保护 page bytes。
写出 page 内容时必须防止并发修改造成 torn logical image。
`FlushBuffer()` 中读取 page LSN、计算 checksum、调用 `smgrwrite()`，都依赖 content lock 保护页面内容。

### 5.11 `FlushBuffer()` 先拥有 I/O，再 flush WAL
`FlushBuffer()` 首先调用 `StartSharedBufferIO(buf, false, true, NULL)`。
如果返回 already done，说明别人已经把它写干净，当前调用直接返回。
否则当前进程设置 `BM_IO_IN_PROGRESS`，并通过 ResourceOwner 记住 buffer I/O ownership。
这个 ownership 很关键。
如果后续 ERROR，ResourceOwner cleanup 能调用 `AbortBufferIO()`，清理 I/O 状态，唤醒等待者。
拿到 I/O 权利后，`FlushBuffer()` 设置 error context。
它打开对应 smgr relation。
然后读取 buffer 的 page LSN。
因为持有至少 share-exclusive content lock，page LSN 在写出期间不会变化。
如果 buffer 是 permanent，`FlushBuffer()` 调 `XLogFlush(recptr)`。
这就是 WAL-before-data 在 buffer flush 路径上的落点。
只有 WAL 已经 durable 到 page LSN，数据页才可以写给 OS。
unlogged relation 不走这个规则。
源码还特别说明了 fake LSN：
某些 unlogged index page 可能有内部使用的 fake LSN。
如果拿 fake LSN 去 flush WAL，可能 flush 到不存在或超前的位置。
所以只有 `BM_PERMANENT` 才 `XLogFlush(page_lsn)`。

### 5.12 `smgrwrite()` 写数据页并登记 fsync 责任
`FlushBuffer()` 之后调用：
- `PageSetChecksum()`
- `smgrwrite(reln, fork, block, bufBlock, false)`
最后一个参数 `skipFsync=false` 很重要。
在 md storage manager 中，写成功后，如果不是 temp relation，会调用 `register_dirty_segment()`。
`register_dirty_segment()` 构造 `FileTag`。
然后调用 `RegisterSyncRequest(&tag, SYNC_REQUEST, false)`。
如果本进程有本地 `pendingOps`，就直接 `RememberSyncRequest()`。
checkpointer 有这个本地 hash。
standalone backend 也有。
普通 backend 没有本地 hash，所以会 `ForwardSyncRequest()` 到 checkpointer ring buffer。
如果 forward 失败且 `retryOnError=false`，`register_dirty_segment()` 会在 backend 本地对这个 segment 立刻 `FileSync()`。
这就是 backend fsync fallback。
它通常很贵，但比丢失 fsync 责任正确。

### 5.13 `TerminateBufferIO()` 清理 buffer 写状态
`smgrwrite()` 成功后，`FlushBuffer()` 统计 I/O，并调用：
`TerminateBufferIO(buf, true, 0, true, false)`
`clear_dirty=true` 会清掉：
- `BM_DIRTY`
- `BM_CHECKPOINT_NEEDED`
同时清掉：
- `BM_IO_IN_PROGRESS`
- 之前的 `BM_IO_ERROR`
`forget_owner=true` 会从当前 ResourceOwner 中忘掉 buffer I/O。
最后广播 buffer I/O condition variable。
等待该 buffer I/O 的 backend 可以继续。
这里清 dirty 的语义只是：
buffer 内容已经成功交给内核的 data file write。
它不等于 file 已经 durable。
durability 还要等 fsync request 在 sync 阶段完成。

### 5.14 `CheckpointWriteDelay()` 控制写速率并吸收请求
每处理一个候选 buffer 后，`BufferSync()` 调 `CheckpointWriteDelay(flags, progress)`。
如果不是 fast checkpoint，不在 shutdown，没收到新的 fast request，而且 `IsCheckpointOnSchedule(progress)` 认为进度合适，checkpointer 会睡 100ms。
睡前它会：
- 处理 config reload。
- `AbsorbSyncRequests()`。
- `CheckArchiveTimeout()`。
- `pgstat_report_checkpointer()`。
如果不能睡，也不是完全不吸收。
它每 `WRITES_PER_ABSORB` 次写操作吸收一次 fsync request。
这防止 checkpoint 高速写 buffer 时，backend fsync request queue 溢出。
`IsCheckpointOnSchedule()` 同时比较两条进度线：
- 时间进度：相对于 `checkpoint_timeout`。
- WAL 消耗进度：相对于 `CheckPointSegments`。
这就是为什么 `checkpoint_completion_target` 不是简单的 sleep 比例。
如果 WAL 消耗很快，checkpoint 会认为自己落后，减少睡眠。

### 5.15 `ProcessSyncRequests()` 在写阶段之后兜住 race
`CheckPointGuts()` 在 `CheckPointBuffers()` 之后调用 `ProcessSyncRequests()`。
这个函数开头再次 `AbsorbSyncRequests()`。
源码解释了最紧的 race：
一个 backend 可能在 `BufferSync()` 即将访问某 buffer 前，先把它写出并清 dirty。
只要该 backend 在清 dirty 前已经 queued fsync request，checkpoint 后面的 `AbsorbSyncRequests()` 就能把它纳入本轮 sync。
所以 buffer write 和 fsync queue 的正确性是配套的。
如果没有这个吸收点，checkpoint 可能看到 buffer 已经干净，就不写；
但对应文件 fsync request 还在共享 ring buffer 里，sync 阶段没看到。
那会破坏 checkpoint durability。

### 5.16 `ProcessSyncRequests()` 用 cycle 防止无限追赶
`ProcessSyncRequests()` 的关键步骤是：
- 如果上次 sync 没完成，先把所有 entry 的 `cycle_ctr` 归到当前 cycle。
- `sync_cycle_ctr++`。
- 设置 `sync_in_progress = true`。
- 扫描 `pendingOps` hash。
- 跳过 `entry->cycle_ctr == sync_cycle_ctr` 的新 entry。
- 对旧 entry 调 sync handler。
- 成功后从 hash 中 remove。
- 全部完成后 `sync_in_progress = false`。
这个 cycle 机制和 `BM_CHECKPOINT_NEEDED` 是同构的。
它们都在说：
本轮只处理边界之前已经进入责任集合的对象。
边界之后新增的对象不丢，但留给下一轮。
如果不跳过新 entry，checkpoint sync 阶段可能被持续写入拖成 never-ending。
如果不保留新 entry，下一轮会漏 fsync。

### 5.17 fsync 失败时先吸收 cancel，再决定错误
relation segment 可能在 fsync 前被 drop 或 truncate。
`ProcessSyncRequests()` 不能简单把 `ENOENT` 当成功。
它的策略是：
第一次遇到可能已删除的错误时，打 DEBUG1，然后 `AbsorbSyncRequests()`。
如果对应的 unlink、forget 或 filter request 已经到达，entry 会变成 `canceled`。
循环条件是 `!entry->canceled`。
如果吸收后仍未 canceled，再次失败就报 `data_sync_elevel(ERROR)`。
这个设计把“文件真的被合法删除了”和“fsync 失败但文件还应该存在”分开。
前者通过 cancel/filter request 证明。
后者不能被吞掉。

### 5.18 checkpoint record 和 control file 最后发布边界
`CheckPointGuts()` 完成 write 和 sync 后，`CreateCheckPoint()` 还没有结束。
它还要等待 `DELAY_CHKPT_COMPLETE` 的事务边界。
如果 hot standby 需要，还会 `LogStandbySnapshot()`。
然后重新进入 critical section。
它插入 checkpoint WAL record：
- shutdown checkpoint 用 `XLOG_CHECKPOINT_SHUTDOWN`。
- online checkpoint 用 `XLOG_CHECKPOINT_ONLINE`。
然后 `XLogFlush(recptr)`。
只有这个 record durable 后，才更新 control file：
- `ControlFile->checkPoint = ProcLastRecPtr`
- `ControlFile->checkPointCopy = checkPoint`
- `minRecoveryPoint` 清理
- `UpdateControlFile()`
这形成最终发布顺序。
如果 checkpoint record 没 flush，却先更新 control file，crash 后 control file 可能指向一个不存在或不可用的 checkpoint record。
PostgreSQL 不这样做。

### 5.19 checkpoint 后清理旧文件和 WAL
checkpoint record 和 control file 更新后，`CreateCheckPoint()` 调 `SyncPostCheckpoint()`。
这会处理旧 cycle 的 pending unlink。
再更新 checkpoint distance estimate。
再根据 redo horizon 和 replication slots 等约束删除或回收旧 WAL segment。
这部分不是本节主问题，但它解释了为什么 fsync 和 control file 顺序不能乱。
只有当 checkpoint 确认完成，旧文件删除和旧 WAL 移除才有安全边界。

## 6. 生命周期 / ownership / cleanup

### 6.1 dirty page 的生命周期
一个 shared buffer page 的本节生命周期可以写成：
1. backend pin buffer，并持有 content lock。
2. 修改 page bytes。
3. 调 `MarkBufferDirty()` 或 hint-bit dirty path。
4. `BM_DIRTY` 被设置。
5. checkpoint 扫描时如果 page 符合条件，设置 `BM_CHECKPOINT_NEEDED`。
6. checkpointer、bgwriter 或 backend 某一路径写出该 buffer。
7. 写出者通过 `StartSharedBufferIO()` 设置 `BM_IO_IN_PROGRESS`。
8. 写出者在 permanent page 上 `XLogFlush(page_lsn)`。
9. 写出者 `smgrwrite(skipFsync=false)`，并登记 file tag fsync request。
10. 写出成功后 `TerminateBufferIO(clear_dirty=true)`。
11. `BM_DIRTY` 和 `BM_CHECKPOINT_NEEDED` 被清理。
12. 后续 `ProcessSyncRequests()` fsync 对应文件。
owner 在不同阶段变化。
修改阶段 owner 是持有 pin 和 content lock 的 backend。
checkpoint-needed flag 是 shared descriptor 状态，不属于单个 backend。
I/O 阶段 owner 是成功设置 `BM_IO_IN_PROGRESS` 的进程。
fsync request 阶段 owner 先是 shared ring buffer，再是 checkpointer-local `pendingOps`。
filesystem dirty page cache 阶段 owner 已经不在 PostgreSQL 内部，而在 OS。
PostgreSQL 通过 fsync 把责任收回来。

### 6.2 checkpoint request 的生命周期
checkpoint request 的状态生命周期是：
1. requestor 在 `ckpt_lck` 下 OR flags 到 `ckpt_flags`。
2. requestor 记录 `old_started`、`old_failed`。
3. requestor 唤醒 checkpointer latch。
4. checkpointer 在 `ckpt_lck` 下读取并清空 flags。
5. checkpointer 增加 `ckpt_started`。
6. checkpointer 广播 `start_cv`。
7. checkpoint 完成后设置 `ckpt_done = ckpt_started`。
8. checkpointer 广播 `done_cv`。
9. 如果中间失败，checkpointer 增加 `ckpt_failed`。
10. 等待者根据 `ckpt_failed` 是否变化决定成功或报错。
这里没有 heap object 表示“一次 checkpoint request”。
状态由 counters 和 flags 表达。
所以不要试图在源码中寻找 request object lifecycle。
真实 lifecycle 在共享计数器上。

### 6.3 fsync request 的生命周期
普通 relation data file fsync request 的生命周期是：
1. `smgrwrite()` 写某 relation fork block。
2. md layer 调 `register_dirty_segment()`。
3. 构造 `FileTag(handler=SYNC_HANDLER_MD, forknum, rlocator, segno)`。
4. `RegisterSyncRequest()` 判断是否有 local `pendingOps`。
5. 普通 backend 走 `ForwardSyncRequest()`。
6. request 进入 `CheckpointerShmem->requests[]` ring buffer。
7. checkpointer 调 `AbsorbSyncRequests()`。
8. request 被复制出 ring buffer。
9. 在 critical section 中调用 `RememberSyncRequest()`。
10. `pendingOps` hash 中创建或复用 `PendingFsyncEntry`。
11. `ProcessSyncRequests()` 按 cycle 扫描旧 entry。
12. 调 `mdsyncfiletag()`。
13. `FileSync()` 成功后从 hash remove。
这条 lifecycle 解释了一个常见疑问：
为什么 `smgrwrite()` 之后不直接 fsync？
因为把多个 page write 合并成一个 file segment fsync，通常更便宜。
checkpoint sync 阶段按 file tag 处理，避免每个 buffer write 都独立 fsync。

### 6.4 ResourceOwner 在 buffer I/O 中兜底
`StartSharedBufferIO()` 设置 `BM_IO_IN_PROGRESS` 后，调用 `ResourceOwnerRememberBufferIO()`。
正常成功路径中，`TerminateBufferIO(... forget_owner=true ...)` 会 `ResourceOwnerForgetBufferIO()`。
如果 ERROR 在中间发生，ResourceOwner release callback `ResOwnerReleaseBufferIO()` 会调用 `AbortBufferIO()`。
`AbortBufferIO()` 对 valid dirty buffer 设置 `BM_IO_ERROR`，并调用 `TerminateBufferIO(clear_dirty=false, set_flag_bits=BM_IO_ERROR, forget_owner=false, release_aio=false)`。
它不清 `BM_DIRTY`。
这是正确的。
写失败后 dirty page 仍然需要后续重试或升级错误。
如果清掉 dirty，就等于丢失了持久化责任。

### 6.5 checkpointer ERROR cleanup
`CheckpointerMain()` 有自己的 `sigsetjmp` 顶层错误恢复。
发生 ERROR 后，它会：
- 清 `error_context_stack`。
- `HOLD_INTERRUPTS()`。
- `EmitErrorReport()`。
- `LWLockReleaseAll()`。
- `ConditionVariableCancelSleep()`。
- `pgstat_report_wait_end()`。
- `pgaio_error_cleanup()`。
- `UnlockBuffers()`。
- `ReleaseAuxProcessResources(false)`。
- `AtEOXact_Buffers(false)`。
- `AtEOXact_SMgr()`。
- `AtEOXact_Files(false)`。
- `AtEOXact_HashTables(false)`。
如果 `ckpt_active`，还会：
- 增加 `ckpt_failed`。
- 设置 `ckpt_done = ckpt_started`。
- 广播 `done_cv`。
- 清 `ckpt_active`。
然后 reset checkpointer memory context。
最后睡 1 秒，避免持续写错误刷爆日志。
这说明 checkpointer 不是简单让 ERROR 杀死进程。
但也不是所有错误都可恢复。
一些 data sync failure 会通过 `data_sync_elevel()` 升级为 PANIC。

### 6.6 smgr cache cleanup
checkpoint 完成后，`CheckpointerMain()` 调 `smgrdestroyall()`。
注释说明原因：
checkpointer 不处理 shared invalidation messages，也不在正常事务边界调用 `AtEOXact_SMgr()`。
如果不定期释放 smgr objects，drop relation 后的 smgr state 可能长期留在 checkpointer。
这不是 data durability 主链路，但属于 long-lived auxiliary process 的 ownership cleanup。
长期后台进程不能假设 backend 的 transaction cleanup 自然发生。

## 7. 正确性机制层次
本节的正确性不是一个机制保证的。
它至少有八层。

### 7.1 WAL-before-data
`FlushBuffer()` 在写 permanent buffer 之前调用 `XLogFlush(BufferGetLSN(buf))`。
它保证：
如果数据页落盘带着某个修改，那么描述该修改的 WAL 至少已经 durable。
它不保证：
该数据页本身已经 durable。
写页只是进入 OS。
后续还要 data-file fsync。

### 7.2 redo pointer 边界
online checkpoint 插入 `XLOG_CHECKPOINT_REDO`，redo pointer 指向这个位置。
checkpoint 完成后插入 `XLOG_CHECKPOINT_ONLINE`，记录这个 redo pointer。
crash recovery 从 checkpoint record 中的 redo pointer 开始 replay。
这要求 redo pointer 之前的必要数据状态已经通过 checkpoint 写回和 fsync 固化。
redo pointer 之后的修改不要求本轮 checkpoint 写入数据文件。
它们靠 WAL replay。

### 7.3 `BM_CHECKPOINT_NEEDED` 的有界集合
`BufferSync()` 第一遍扫描 dirty permanent buffers 并设置 `BM_CHECKPOINT_NEEDED`。
这个 flag 是本轮写回集合。
它保证：
checkpoint 不会无限追逐新 dirty page。
它不保证：
所有 `BM_DIRTY` page 在 checkpoint 结束时都变 clean。
如果 checkpoint 期间有新写入，结束时仍可存在 dirty buffers。
这不破坏本轮 redo 边界。

### 7.4 content lock 和 header lock 分工
buffer header lock 保护 `BufferDesc.state` 和 `tag` 变更。
content lock 保护 page bytes 和 page LSN 的一致读取/写出。
pin 保证 buffer identity 不会在你使用期间被替换。
所以 `FlushBuffer()` 需要：
- pin：防止 buffer 被回收。
- share-exclusive content lock：防止 page bytes 和 LSN 并发变化。
- `BM_IO_IN_PROGRESS`：防止两个进程同时写同一 buffer。
这三个不能互相替代。

### 7.5 fsync request queue 承接 write 和 sync 的缝隙
`smgrwrite()` 成功后必须登记 fsync request。
这弥补了 `write()` 和 `fsync()` 分离带来的责任缝隙。
如果 backend 或 bgwriter 替 checkpointer 写掉了某个 `BM_CHECKPOINT_NEEDED` buffer，checkpoint sync 阶段仍然要 fsync 对应 file segment。
所以 `ProcessSyncRequests()` 开头必须 `AbsorbSyncRequests()`。
这是 checkpoint write 阶段和 sync 阶段的 ordering bridge。

### 7.6 `sync_cycle_ctr` 防止新请求污染本轮
`ProcessSyncRequests()` 增加 `sync_cycle_ctr` 后，跳过新 cycle entry。
它保证本轮 sync 集合有限。
如果 fsync 失败导致下一次重试，`sync_in_progress` 会让函数先把旧 entry 的 cycle 归一，避免 counter wrap 或 stale cycle 让旧责任被误认为新责任。
这不是性能细节。
它是 retry correctness。

### 7.7 checkpoint record flush 先于 control file
`CreateCheckPoint()` 插入 checkpoint WAL record 后立即 `XLogFlush(recptr)`。
然后才更新 control file。
control file 是 crash recovery 找 checkpoint 的入口。
它不能指向未 durable 的 WAL record。
这里的 ordering 比单个 data file fsync 更高层。
它发布的是整个 checkpoint 边界。

### 7.8 delayed unlink 和 cancel/filter
relation 删除不能随便立刻删除所有文件。
`SYNC_UNLINK_REQUEST` 进入 pending unlink list。
`SyncPostCheckpoint()` 在 checkpoint 之后处理旧 cycle unlink。
`SYNC_FORGET_REQUEST` 和 `SYNC_FILTER_REQUEST` 可以取消 pending fsync/unlink。
这个机制保证：
checkpoint 前后跨越的文件生命周期不会和 fsync 阶段互相踩踏。
它也解释了为什么 `ProcessSyncRequests()` 在 `ENOENT` 上要 absorb cancel request 后再决定。

## 8. 错误路径 / 异常路径 / fallback

### 8.1 checkpoint request 通知失败
`RequestCheckpoint()` 唤醒 checkpointer 时，可能发现 `ProcGlobal->checkpointerProc` 还没设置。
如果 request 不等待，失败只 LOG。
因为 `ckpt_flags` 已经在共享内存中设置，checkpointer 启动后仍可能看到。
如果 request 带 `CHECKPOINT_WAIT`，它会最多重试 `MAX_SIGNAL_TRIES` 次。
超过后报 ERROR。
这个 fallback 说明：
设置 request flags 和唤醒 latch 是两个动作。
latch 只是加速。
共享 flags 才是 request state。

### 8.2 fsync request queue 满
`ForwardSyncRequest()` 在 `CheckpointerCommLock` 下检查 queue。
如果满，它先尝试 `CompactCheckpointerRequestQueue()`。
compact 用 hash 找重复 request，保留后出现的 request，跳过更早重复项。
为什么不随便删旧项？
因为中间可能有 `SYNC_FORGET_REQUEST` 或 `SYNC_FILTER_REQUEST` 改变语义。
如果 compact 后仍满，`ForwardSyncRequest()` 返回 false。
md layer 的 `register_dirty_segment()` 在 `retryOnError=false` 时会 backend 本地 fsync。
这是性能退化路径。
它把延迟从 checkpointer 转回 backend。
但它守住了 durability。
如果是 unlink/forget/filter 等不能丢的 request，调用者通常用 `retryOnError=true`，`RegisterSyncRequest()` 会等待后重试。
等待事件是 `REGISTER_SYNC_REQUEST`。

### 8.3 buffer write 失败
`FlushBuffer()` 中的 `smgrwrite()` 可能 ERROR。
比如磁盘满、I/O error、权限问题。
错误上下文会报告 relation path、fork、block。
因为 `StartSharedBufferIO()` 已经把 buffer I/O 记到 ResourceOwner，ERROR cleanup 会走 `AbortBufferIO()`。
`AbortBufferIO()` 不清 dirty。
它设置 `BM_IO_ERROR` 并清 `BM_IO_IN_PROGRESS`。
等待该 buffer I/O 的 backend 会被 condition variable 唤醒。
后续再写该 buffer 时能看到失败历史。

### 8.4 data-file fsync 失败
`ProcessSyncRequests()` 调 sync handler 失败时，最终通过 `data_sync_elevel(ERROR)` 报错。
`data_sync_retry` 默认 off 时，`data_sync_elevel()` 返回 PANIC。
源码注释给出的理由很硬：
某些 OS 在 write-back failure 后可能丢弃 dirty data。
如果 PostgreSQL 继续运行，下一次 fsync 可能“成功”，但实际已经没有那份 dirty data 了。
WAL 可能是唯一剩余副本。
因此默认不允许继续尝试后续 checkpoint。
如果 `data_sync_retry=on`，会按 ERROR 处理，checkpoint 失败但系统可以继续尝试。
这个配置只适合明确知道 OS 不会丢 dirty buffered data 的场景。

### 8.5 `ENOENT` 不是立即成功
fsync 时发现文件不存在，可能是合法 drop/truncate。
也可能是真错误。
`ProcessSyncRequests()` 第一次遇到 `FILE_POSSIBLY_DELETED(errno)` 时，不立刻忽略。
它吸收 pending requests。
如果对应 cancel/filter 到了，entry 变 `canceled`，循环结束。
如果没有 cancel，下一次失败就按 data sync failure 报错。
这比“忽略 ENOENT”保守。
它要求删除路径显式发送 forget/filter/unlink 语义。

### 8.6 WAL fsync 失败直接 PANIC
`issue_xlog_fsync()` 根据 `wal_sync_method` 选择 `fsync`、`fdatasync` 或 write-through。
如果 `enableFsync=off` 或 WAL 以 `open_datasync` 方式已经同步，它可以直接返回。
否则 fsync 失败会 `ereport(PANIC)`。
WAL 是 crash recovery 的根。
PostgreSQL 不能在 WAL durability 不可信时继续假装事务持久。
这和 data-file fsync 的 retry 讨论不同。
data file 还有 WAL 可恢复。
WAL 自己没有更低层的 redo。

### 8.7 `enableFsync=off` 是正确性降级，不是优化开关
`enableFsync` 关闭时：
- `ScheduleBufferTagForWriteback()` 不再追踪 writeback。
- `ProcessSyncRequests()` 会跳过实际 file sync。
- `issue_xlog_fsync()` 会 quick exit。
这会显著改变 durability。
它只能用于测试或明确接受 crash 后数据损坏风险的环境。
课程讨论性能时不能把 `fsync=off` 当正常调优建议。

### 8.8 restartpoint 的特殊退化
`CheckpointerMain()` 在 recovery 中可能做 restartpoint 而非 checkpoint。
如果无法执行 restartpoint，源码会把 `last_checkpoint_time` 调整到 15 秒后再试。
这不是本节主链路，但它提醒我们：
standby recovery 下的持久化边界受已 replay 的 checkpoint record 限制。
不能简单套用 primary online checkpoint 的全部判断。

## 9. 成本、资源与跨模块传播

### 9.1 `NBuffers` 扫描成本
`BufferSync()` 每次 checkpoint 都扫描 `NBuffers`。
这是和 `shared_buffers` 大小线性相关的 CPU/cache 成本。
即使 dirty buffers 只有很少，扫描也要读每个 descriptor state。
descriptor 被 cache line padding 后降低 false sharing，但大 buffer pool 仍然意味着大量内存访问。
所以 `shared_buffers` 增大后，checkpoint 的固定成本也增大。
这不一定是瓶颈。
但在极大 buffer pool、频繁 checkpoint、dirty set 很小的 workload 中，它会变得可见。

### 9.2 dirty set 排序和写回成本
候选 dirty buffer 数记为 `D`。
`BufferSync()` 要对 `D` 个 `CkptSortItem` 排序。
还要按 tablespace 构建进度状态和 heap。
写回成本主要随 `D`、page 分布、relation segment 分布和底层 I/O 并发能力变化。
同样是 100GB dirty pages：
顺序分布在少数大 relation 上，和随机分布在大量小 relation 上，fsync 行为会非常不同。
`buffers_written` 只告诉你 page 数。
它不告诉你文件分布、write amplification、fsync queue depth 或设备 flush latency。

### 9.3 checkpoint pacing 的两个压力源
`CheckpointWriteDelay()` 同时看时间和 WAL 消耗。
`checkpoint_completion_target` 会把目标进度压到 checkpoint interval 的某个比例。
但如果 WAL segment 消耗过快，checkpoint 会减少 sleep。
这会把压力提前释放到 I/O 子系统。
所以 checkpoint 突刺不只由 `checkpoint_completion_target` 决定。
它还受：
- `max_wal_size`
- WAL generation rate
- dirty page generation rate
- checkpoint_timeout
- storage writeback behavior
- bgwriter/backend 提前清脏能力
共同影响。

### 9.4 bgwriter 和 checkpointer 的边界
bgwriter 调 `BgBufferSync()`，主要围绕 clock sweep 清理可复用 buffer。
它可以提前写掉 dirty buffers。
如果写的是 checkpoint-needed buffer，`TerminateBufferIO()` 也会清 `BM_CHECKPOINT_NEEDED`。
因此 checkpointer 的 `buffers_written` 不包含所有本轮曾经 checkpoint-needed 的 buffer。
它只统计 checkpointer 自己写的数量。
bgwriter 的 `buffers_clean` 和 checkpointer 的 `buffers_written` 是不同职责。
一个 workload 中，bgwriter 写得多可能让 checkpoint write phase 变轻。
但 fsync responsibility 仍然会通过 request queue 进入 checkpoint sync phase。

### 9.5 backend write 和 backend fsync fallback
当 backend 找不到可用 clean buffer，或者 bulk 操作绕过部分常规路径时，backend 也可能直接写 dirty buffer。
backend 写成功后同样要登记 fsync request。
如果 checkpointer request queue 满到无法 forward，backend 会本地 fsync。
这会把 checkpoint 压力扩散到前台 query latency。
诊断时，如果看到普通 backend 等 `DataFileSync` 或 `RegisterSyncRequest`，不要只盯 checkpointer。
这说明 fsync responsibility 已经从后台传播到了前台。

### 9.6 fsync 粒度是 file tag，不是 page
多个 buffer write 可以对应同一个 relation segment。
`pendingOps` hash 会合并重复 `FileTag`。
所以 fsync count 更接近“被写过的文件段数量”，不是“被写过的 page 数量”。
大量小 relation、小 partition、多个 fork、多个 tablespace，可能让 `sync files` 增加。
单个巨大 relation 的大量顺序 page write，可能 `buffers_written` 很高但 `sync files` 不高。
这就是 schema shape 对 checkpoint sync phase 的影响。

### 9.7 WAL pressure 会反向触发 checkpoint
`XLogWrite()` 在 segment 完成时会判断是否消耗了太多 WAL。
如果 `XLogCheckpointNeeded(openLogSegNo)` 为真，会 `RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)`。
这条边界说明：
checkpoint 不只是定时任务。
WAL generation rate 会反向推动 checkpoint frequency。
如果 checkpoint 太频繁，`CheckpointerMain()` 会在 `CHECKPOINT_CAUSE_XLOG` 且间隔小于 `checkpoint_warning` 时提示增加 `max_wal_size`。
这个日志不是“checkpointer 太慢”的直接证据。
它更常表示 WAL 产生速度和 checkpoint/WAL retention 配置不匹配。

### 9.8 OS page cache 和 storage cache 是外部状态
`smgrwrite()` 成功后，数据通常只是进入 OS page cache 或 kernel I/O 队列。
`FileWriteback()` 是 hint。
`FileSync()` 才请求 durable storage。
但即使 fsync 返回，真实硬件行为仍受：
- filesystem
- mount option
- drive cache
- storage controller
- cloud volume implementation
- write barrier 支持
影响。
PostgreSQL 能调用正确系统调用并按顺序组织。
它不能证明硬件真的没有撒谎。
这是诊断中的 hardware-dependent 边界。

### 9.9 参与状态推进的后台进程
本节至少涉及这些后台进程或辅助进程：
- checkpointer：主导 checkpoint request、dirty scan、write pacing、sync phase。
- bgwriter：提前清理 dirty buffers，可能清掉 `BM_CHECKPOINT_NEEDED`。
- walwriter：推进 WAL 写/flush，间接影响 `FlushBuffer()` 中 `XLogFlush()` 等待时间。
- backend：制造 dirty pages、写 WAL、可能写 buffer、forward fsync requests。
- startup process：recovery 下执行 restartpoint，而非普通 online checkpoint。
- archiver：WAL segment complete 后被通知，和 checkpoint/WAL retention 有边界。
- walsender / replication slots：影响旧 WAL 是否能移除。
不能把 checkpoint 观测结果归因给单个进程。
正确模型是资源压力跨模块传播。

## 10. 观测与诊断入口

### 10.1 本节锚定的 runtime truth
本节要锚定的 runtime truth 是：
一次 checkpoint 的 write phase 和 sync phase 是分离的；
`BufferSync()` 只追 checkpoint 开始时标记的 dirty set；
checkpoint 期间新写入可以继续发生，并通过下一轮 dirty scan 或 sync cycle 接管。
如果你只看“checkpoint complete”日志，很容易漏掉这个真相。
诊断要把现象拆成三段：
1. checkpoint 是否频繁开始。
2. write phase 是否写了大量 buffers。
3. sync phase 是否卡在少数慢文件或大量文件上。

### 10.2 `pg_stat_checkpointer`
SQL 入口：

```sql
SELECT *
FROM pg_stat_checkpointer;
```
重要字段：
- `num_timed`：timeout 触发次数。
- `num_requested`：request 触发次数。
- `num_done`：完成次数。
- `write_time`：累计 write phase 时间，毫秒。
- `sync_time`：累计 sync phase 时间，毫秒。
- `buffers_written`：checkpointer 自己写出的 shared buffers。
- `slru_written`：checkpoint 写出的 SLRU buffers。
粒度：
这是 instance 累计统计。
它不是单次 checkpoint。
它也不是当前 shared memory 状态。
诊断时最好在实验前后记录 delta。

### 10.3 `log_checkpoints`
打开 `log_checkpoints=on` 后，checkpoint 结束日志会包含：
- wrote N buffers
- wrote N SLRU buffers
- WAL file added/removed/recycled
- write time
- sync time
- total time
- sync files
- longest sync
- average sync
- distance / estimate
- checkpoint lsn / redo lsn
这是单次 checkpoint 最有用的入口。
它能把 write 和 sync 拆开。
例如：
- write 很长，sync 很短：更像写脏页量大或 pacing 影响。
- write 不长，sync 很长：更像 fsync latency 或文件分布问题。
- sync files 很多：更像 relation/partition/segment 分散。
- longest 远大于 average：更像少数文件或设备 flush 抖动。
但它仍然看不到：
- 哪些 buffer 带过 `BM_CHECKPOINT_NEEDED`。
- 哪个 backend 提前写掉了哪些 buffer。
- request queue 是否曾经 compact。
- `pendingOps` hash 的瞬时大小。

### 10.4 `pg_stat_io`
当前版本有 `pg_stat_io`。
可按 backend_type、object、context、operation 看 I/O。
本节重点看：
- checkpointer 的 relation write。
- checkpointer 的 relation fsync。
- checkpointer 的 relation writeback。
- client backend 的 relation fsync。
- WAL 的 write/fsync。
如果普通 backend 出现明显 relation fsync，可能说明 backend fallback 或 immediate sync 路径参与了前台延迟。
如果 WAL fsync 时间高，`FlushBuffer()` 的 `XLogFlush(page_lsn)` 也可能变慢。
这时 checkpoint write phase 的等待不完全是 data file write。
它可能是在等待 WAL durability。

### 10.5 wait events
相关 wait events 包括：

| wait event | 说明 |
| --- | --- |
| `CheckpointerMain` | checkpointer 主循环睡眠 |
| `CheckpointWriteDelay` | checkpoint 按 pacing 在两次 write 间睡眠 |
| `CheckpointStart` | backend 等 checkpoint 开始 |
| `CheckpointDone` | backend 等 checkpoint 完成 |
| `CheckpointDelayStart` | checkpoint 等阻塞 start 边界的 backend |
| `CheckpointDelayComplete` | checkpoint 等阻塞 complete 边界的 backend |
| `RegisterSyncRequest` | backend 因 fsync request queue 满而等待 |
| `DataFileWrite` | relation data file write |
| `DataFileSync` | relation data file fsync |
| `DataFileFlush` | writeback hint |
| `WALSync` | WAL fsync |
wait event 是当前位置，不是完整因果。
一个 backend 在 `CheckpointDone` 上等待，只说明它请求了 wait checkpoint。
它不告诉你 checkpoint 现在卡在 write、sync、WAL flush、delay xact 还是 storage。
必须结合日志和统计。

### 10.6 gdb / probe / tracepoint
源码跟踪时，建议断点：

```gdb
break RequestCheckpoint
break CheckpointerMain
break CreateCheckPoint
break BufferSync
break SyncOneBuffer
break FlushBuffer
break ProcessSyncRequests
break mdsyncfiletag
```
在 `BufferSync()` 中观察：
- `num_to_scan`
- `CkptBufferIds[0]`
- `bufHdr->state`
- `BM_CHECKPOINT_NEEDED`
在 `FlushBuffer()` 中观察：
- `BufferGetLSN(buf)`
- `BM_PERMANENT`
- `StartSharedBufferIO()` 返回值
在 `ProcessSyncRequests()` 中观察：
- `sync_cycle_ctr`
- `sync_in_progress`
- `entry->cycle_ctr`
- `entry->canceled`
注意：
直接在生产系统 gdb checkpointer 会暂停持久化进程。
这个实验只适合本地 lab。

### 10.7 能看到、只能推断、基本不可见

| 状态 | 可见性 |
| --- | --- |
| checkpoint 开始/结束、write/sync/total 时间 | `log_checkpoints` 可见 |
| checkpoint 累计次数和累计时间 | `pg_stat_checkpointer` 可见 |
| 当前等待点 | `pg_stat_activity.wait_event` 可见 |
| checkpointer 写出的 buffers 数 | 日志和 stats 可见 |
| bgwriter/backend 提前写掉的 checkpoint-needed page | 只能间接推断 |
| `BM_CHECKPOINT_NEEDED` 当前在哪些 buffers 上 | 普通 SQL 不可见，需要源码/调试 |
| `pendingOps` hash 当前大小 | 普通 SQL 不可见，需要 instrumentation |
| request queue compact 是否发生 | 默认只有 DEBUG1 log，通常不可见 |
| OS 是否真的把 fsync 传到可靠介质 | PostgreSQL 内部不可证明 |

### 10.8 一个诊断推理例子
现象：
`log_checkpoints` 显示 checkpoint complete 中 `write=2s, sync=45s, sync files=12000, longest=8s, average=4ms`。
第一层判断：
write phase 不长，checkpointer 写 dirty buffers 不是主时间。
第二层判断：
sync phase 很长，且 files 很多。
第三层判断：
平均很低但最长很高，说明多数文件 fsync 快，少数文件/设备 flush 卡住。
第四层回源码：
`ProcessSyncRequests()` 是按 `pendingOps` hash entry 调 sync handler。
每个 relation segment 是一个 file tag。
大量小 relation 或 partition 写入会增加 file tag 数。
第五层限制：
日志不能告诉你是哪一个 relation segment 最慢。
需要 `log_checkpoints` DEBUG1 per-file sync、`strace`、eBPF、filesystem trace 或定制 instrumentation。

## 11. 常见误区

### 误区 1：checkpoint 会把所有 dirty buffers 写干净
不对。
online checkpoint 只需要保证 redo pointer 边界之前的必要数据状态 durable。
`BufferSync()` 用 `BM_CHECKPOINT_NEEDED` 固定扫描时的 dirty set。
扫描后新 dirty 的 page 可以留给下一轮。
checkpoint 结束时仍可能存在 dirty buffers。

### 误区 2：`write()` 成功就等于 checkpoint 已经安全
不对。
`FlushBuffer()` 的 `smgrwrite()` 只把 page 写给 OS。
checkpoint 还需要 `ProcessSyncRequests()` 对相关 file tags fsync。
最后还要 checkpoint WAL record flush 和 control file update。

### 误区 3：`BM_CHECKPOINT_NEEDED` 表示这个页一定由 checkpointer 写
不对。
bgwriter 或 backend 可能先写掉该 buffer。
成功写出时会清 `BM_CHECKPOINT_NEEDED`。
checkpointer 后续看到 flag 没了就跳过。

### 误区 4：`pg_stat_checkpointer.buffers_written` 等于本轮所有被 checkpoint 覆盖的页
不对。
它统计 checkpointer 自己写出的 buffers。
其他进程提前写掉的 page 不在这里。
另外这是累计统计，不是单次值。

### 误区 5：fsync request queue 满只是后台慢一点
不对。
queue 满后会 compact。
compact 失败时，backend 可能本地 fsync。
这会把 checkpoint/storage 压力传导到前台 query latency。

### 误区 6：`DataFileSync` 等待一定是 checkpointer
不对。
checkpointer sync phase 会出现 data file fsync。
但 backend fallback、immediate sync、bulk load 边界也可能出现。
必须结合 `backend_type`、调用路径和 workload。

### 误区 7：增加 `checkpoint_completion_target` 一定降低 checkpoint 压力
不一定。
它让 checkpoint 更分散，但如果 WAL 消耗过快或 checkpoint 已经落后，`CheckpointWriteDelay()` 会减少睡眠。
如果 storage sync phase 才是瓶颈，单纯调 pacing 不会消除慢 fsync。

### 误区 8：把 `fsync=off` 当性能优化
这是 durability 降级。
它会让 WAL fsync、data-file fsync、writeback tracking 的语义都变化。
测试可以用。
不能把它当生产调优结论。

## 12. 课堂实验

### 实验 1：用 SQL 和日志拆分 write phase / sync phase
目标：
观察 checkpoint 的 write time、sync time、buffers_written 与 WAL pressure 的关系。
步骤：
1. 在本地测试实例设置：
   `log_checkpoints=on`、较小 `max_wal_size`、合适 `checkpoint_timeout`。
2. 重启或 reload 对应配置。
3. 创建一张大表并持续写入。
4. 在写入前后查询 `pg_stat_checkpointer`。
5. 查看日志中的 checkpoint start/complete。
示例 SQL：

```sql
SELECT pg_stat_reset_shared('checkpointer');
CREATE TABLE ckpt_dirty_demo AS
SELECT g AS id, repeat('x', 2000) AS payload
FROM generate_series(1, 1000000) AS g;
UPDATE ckpt_dirty_demo
SET payload = repeat('y', 2000)
WHERE id % 3 = 0;
CHECKPOINT;
SELECT *
FROM pg_stat_checkpointer;
```
观察点：
- `buffers_written` 是否明显增加。
- `write_time` 和 `sync_time` 哪个增长更多。
- checkpoint 日志中的 `sync files` 是否多。
- `CHECKPOINT` 前后是否有 `CheckpointDone` 等待。
回源码解释：
- `buffers_written` 对应 `BufferSync()` 中 checkpointer 自己写出的 buffers。
- `write_time` 覆盖 `CheckPointBuffers()` 所在阶段。
- `sync_time` 覆盖 `ProcessSyncRequests()`。
- 日志中的 `sync files` 对应 `pendingOps` 被处理的 file tag 数。

### 实验 2：源码断点跟 `BM_CHECKPOINT_NEEDED`
目标：
确认 checkpoint 的 dirty set 是先扫描打标，再逐步写回。
建议只在本地 lab 跑。
步骤：
1. 启动 PostgreSQL debug build。
2. 附加 checkpointer 进程。
3. 在 `BufferSync()`、`SyncOneBuffer()`、`TerminateBufferIO()` 下断点。
4. 触发 `CHECKPOINT`。
5. 在 `BufferSync()` 第一轮扫描后观察某个 `bufHdr->state`。
6. 在 `TerminateBufferIO(clear_dirty=true)` 后观察同一个 buffer 的 flags。
预期：
- 扫描命中时 `BM_DIRTY` 和 `BM_CHECKPOINT_NEEDED` 同时存在。
- 写回开始后出现 `BM_IO_IN_PROGRESS`。
- 写回成功后 `BM_DIRTY`、`BM_CHECKPOINT_NEEDED`、`BM_IO_IN_PROGRESS` 都被清掉。
回源码解释：
- 打标发生在 `BufferSync()` 第一轮扫描。
- I/O ownership 发生在 `StartSharedBufferIO()`。
- 清标发生在 `TerminateBufferIO(clear_dirty=true)`。

### 实验 3：给 `pendingOps` 加最小 instrumentation
目标：
观察 fsync request 从 shared queue 被吸收到 checkpointer-local hash，再按 cycle 处理。
这是源码练习，不要求提交产品 patch。
步骤：
1. 在 lab 源码的 `sync.c` 中给 `RememberSyncRequest()` 加 DEBUG log，打印 `type`、`handler`、`segno`、`sync_cycle_ctr`。
2. 给 `ProcessSyncRequests()` 开头和跳过新 entry 的分支加 DEBUG log。
3. 运行持续写入 workload。
4. 触发 `CHECKPOINT`。
5. 对比 checkpoint 开始前、sync phase 中、checkpoint 后新增的 request。
预期：
- checkpoint write phase 中仍会不断吸收 request。
- `ProcessSyncRequests()` 增加 `sync_cycle_ctr` 后，新 entry 被留给下一轮。
- 同一 file tag 的重复 request 会在 hash 中合并。
回源码解释：
- `AbsorbSyncRequests()` 从 shared ring 搬运 request。
- `RememberSyncRequest()` 合并到 `pendingOps`。
- `ProcessSyncRequests()` 的 cycle counter 给本轮 sync 集合划边界。

## 13. 讨论题
1. 为什么 `BufferSync()` 要先扫描并设置 `BM_CHECKPOINT_NEEDED`，而不是一边扫描一边把所有当前 dirty buffers 写到没有 dirty 为止？
2. `BM_DIRTY`、`BM_CHECKPOINT_NEEDED`、`BM_IO_IN_PROGRESS` 三个 bit 分别解决什么问题？哪两个 bit 不能互相替代？
3. 如果 backend 在 checkpoint 期间写掉了一个本轮 checkpoint-needed buffer，checkpoint 为什么仍然不会漏掉 fsync？
4. 为什么 `ProcessSyncRequests()` 要在开始时再次 `AbsorbSyncRequests()`？它防的是哪条 race？
5. 为什么 data-file fsync 失败可能通过 `data_sync_elevel()` 变成 PANIC，而不是简单让 checkpoint 下次重试？
6. 如果 `log_checkpoints` 显示 `buffers_written` 很低但 `sync files` 很高，你会优先怀疑哪些 workload/schema/storage 特征？
7. `checkpoint_completion_target` 能平滑哪一段成本？它不能平滑哪一段成本？
8. 为什么 control file 必须在 checkpoint WAL record flush 之后更新？如果顺序反过来，crash recovery 会面对什么风险？

## 14. 本节小结
本节唯一主问题是：
PostgreSQL 如何在持续写入下，为一次 checkpoint 定义并完成一个有限的脏页和 fsync 责任集合？
核心链路是：
backend 修改 page 并设置 `BM_DIRTY`；
checkpoint 固定 redo pointer；
`BufferSync()` 扫描 `NBuffers` 并给本轮 dirty permanent pages 设置 `BM_CHECKPOINT_NEEDED`；
`SyncOneBuffer()` 和 `FlushBuffer()` 持 pin、content lock 和 `BM_IO_IN_PROGRESS` 写出 page；
写出前用 `XLogFlush(page_lsn)` 保证 WAL-before-data；
`smgrwrite()` 写 data file 并登记 fsync request；
`ProcessSyncRequests()` 吸收 request，用 cycle counter 只处理本轮之前的 `pendingOps`；
checkpoint 最后插入并 flush checkpoint record，再更新 control file。
核心状态和边界是：
- `BM_DIRTY` 表示 buffer 需要写。
- `BM_CHECKPOINT_NEEDED` 表示本轮 checkpoint 要追这个 buffer。
- `BM_IO_IN_PROGRESS` 表示某进程拥有 buffer I/O cleanup 责任。
- `FileTag` 和 `pendingOps` 表示 file-level fsync 责任。
- `sync_cycle_ctr` 表示本轮 sync 的时间边界。
- checkpoint WAL record 和 control file 表示 crash recovery 可见的发布边界。
ownership / cleanup 的关键是：
buffer pin 防止 identity 被替换；
content lock 保护 page bytes 和 LSN；
ResourceOwner 兜底 buffer I/O；
checkpointer-local memory context 管 pending ops；
`TerminateBufferIO()` 清 dirty 和 checkpoint-needed；
checkpointer ERROR cleanup 释放 LWLocks、condition variable wait、buffer resources、smgr/file/hash 状态，并通知等待者 checkpoint failed。
错误路径的关键是：
queue 满先 compact，失败则 backend fsync fallback；
write 失败不清 dirty；
data-file fsync 失败默认可 PANIC；
`ENOENT` 要靠 cancel/filter request 证明合法；
WAL fsync 失败直接 PANIC；
`fsync=off` 是 durability 降级。
观测上：
`pg_stat_checkpointer` 看累计次数、write time、sync time、buffers written。
`log_checkpoints` 看单次 checkpoint 的 write/sync/total、sync files、longest/average。
wait events 看当前位置。
`pg_stat_io` 能区分 backend_type、object、operation。
但普通 SQL 看不到 `BM_CHECKPOINT_NEEDED` 分布、`pendingOps` hash、request queue compact 或每个 file tag 的完整生命周期。
可迁移的系统规律是：
在线持久化边界通常不是“把所有当前状态刷到零”。
更可扩展的做法是先定义一个有时间边界的责任集合，再用显式状态、ownership cleanup、retry/fallback 和最终发布记录，把这个集合推进到 durable。
这个规律在 checkpoint、WAL flush、buffer eviction、vacuum horizon、replication slot retention 中都会反复出现。
仍需保留的推断边界是：
write/sync 时间强依赖 workload、relation 分布、tablespace、filesystem、storage、WAL rate 和版本实现。
统计指标只能说明某个层面的结果。
真正的因果通常需要把 SQL 现象、日志、wait event、pg_stat、源码状态和 OS I/O 观测放在同一条时间线上解释。
