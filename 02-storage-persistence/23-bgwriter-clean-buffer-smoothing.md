# PostgreSQL Background Writer 与 reusable clean buffer
## 课程定位
前置知识： 你已经读过 shared buffer identity、mapping partition lock、clock sweep、dirty page LSN、WAL-before-data、eviction writeback 和 checkpoint 生命周期。
你应该知道 `BufferTag` 标识一个 shared buffer slot 当前绑定的磁盘 block。 你也应该知道 `BufferDesc.state` 里的 `refcount`、`usage_count`、`BM_DIRTY`、`BM_VALID`、`BM_TAG_VALID`、`BM_IO_IN_PROGRESS` 不是独立语义，而是一个生命周期状态组合。
本节唯一主问题：
```text
PostgreSQL 如何在不知道未来 buffer allocation 精确需求的情况下，用 background writer 提前制造一批 reusable clean buffer，从而平滑前台 miss 的写脏页延迟，同时又不把 checkpoint correctness 交给 bgwriter？
```
核心矛盾：
```text
前台 backend 在 buffer miss 时需要尽快拿到一个 clean reusable slot；
但只有前台真实知道下一次 miss 会落到哪里，buffer 的 pin、usage_count 和 dirty 状态又持续变化；
后台进程只能根据过去的 allocation 速率和 clock-sweep 位置做估算，机会性写出可复用的 dirty buffer。
```
本节只讲一个问题： `background writer 如何平滑 clean reusable buffer 的供给。`
它不重新讲完整 clock sweep。 它不重新讲 dirty victim 的完整 WAL / smgr / fsync 链路。
它不讲 checkpoint redo pointer 的生命周期。 这些内容在前几节已经作为独立主问题展开。
本节会引用它们，但只服务一个判断： `bgwriter 是 latency smoothing worker，不是 crash correctness owner。`
源码基线：
```text
/home/nail/postgres-lab
branch: master
commit: bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8
```
重点源码：
```text
src/backend/postmaster/bgwriter.c
src/backend/storage/buffer/bufmgr.c
src/backend/storage/buffer/freelist.c
src/include/storage/buf_internals.h
src/backend/utils/activity/pgstat_bgwriter.c
src/backend/utils/activity/pgstat_io.c
src/backend/utils/activity/wait_event_names.txt
src/backend/catalog/system_views.sql
src/backend/postmaster/checkpointer.c
```
学完后应能独立判断：
- 为什么 `bgwriter` 只平滑 clean buffer supply，不承担 checkpoint correctness。
- 为什么 `buffers_clean` 不是 checkpoint 写页数。
- 为什么 `buffers_alloc` 是 bgwriter 控制回路的输入，不是 SQL block read 计数。
- 为什么 `maxwritten_clean` 上升说明 bgwriter 每轮写页上限成为约束，但不等于系统已经不安全。
- 为什么 `BGWRITER_HIBERNATE` 不是“没有 dirty page”的证明。
- 为什么前台 backend 写 dirty victim 是正常 fallback，不是 bgwriter 失效的 bug。
- 为什么 `bgwriter_flush_after` / `WritebackContext` 是 writeback hint，不是 fsync 或 durability 边界。
本节 runtime truth：
```text
在 shared buffer miss 压力下，如果后台来不及把 reusable dirty buffer 预先写成 clean，
前台 backend 会在 GetVictimBuffer() 路径自己写 dirty victim；
bgwriter 的目标只是降低这种前台同步写发生的概率和突发度。
```
## 1. 本节在总主线中的位置
前几节已经讲过 buffer allocation 的前台路径。 核心链路是：
```text
PinBufferForBlock()
  -> BufferAlloc()
  -> GetVictimBuffer()
  -> StrategyGetBuffer()
  -> optional FlushBuffer() for dirty victim
  -> InvalidateVictimBuffer()
  -> install new BufferTag
```
这条路径说明了一个直接问题： 如果 miss 找到的 victim 是 dirty，前台 backend 不能直接覆盖它。
它必须先把旧 page 写出去。 如果这是 permanent relation，它还要先把 WAL flush 到该 page LSN。
这会把一次普通 read miss 或 extension path 放大成 WAL flush、data file write、writeback hint 和 mapping invalidation。 这就是 bgwriter 出现的位置。
它不改变前台 replacement 的正确性协议。 它也不替代 checkpointer。
它只是周期性扫描 clock-sweep 前方的一段 buffer descriptor。 如果遇到：
```text
refcount == 0
usage_count == 0
BM_VALID
BM_DIRTY
```
它就尝试把这个 buffer 写成 clean。 这样下一次前台 `StrategyGetBuffer()` 扫到同一个 slot 时，更可能只需要 pin、invalidate old tag、安装 new tag，而不需要自己执行 dirty write。
这也是本节的线性故事：
```text
前台 allocation 产生需求信号
  -> StrategyControl 记录 allocation 速率和 clock hand 位置
  -> bgwriter 周期性读取这个信号
  -> BgBufferSync() 估算未来需要多少 reusable clean buffer
  -> SyncOneBuffer() 只写可复用 dirty buffer
  -> 前台 miss 获得更平滑的 clean victim 供给
  -> 如果估算失败，前台路径仍然写 dirty victim
```
注意这里的对象不是“所有 dirty page”。 本节对象是：
`将来可能被 replacement 复用的一小段 dirty buffer。` 这和 checkpoint 的对象不同。
checkpoint 关心的是： `checkpoint 开始时哪些 dirty permanent pages 必须在 checkpoint 完成前写出并 fsync。`
bgwriter 关心的是： `clock-sweep 即将经过的 replacement 候选中，有多少应该提前变 clean。`
两个问题有交集。 一个被 bgwriter 写干净的 checkpoint-needed page 也会减少 checkpointer 后续写页工作。
但 checkpoint correctness 不依赖 bgwriter 是否及时运行。 这条边界必须先固定下来。
否则本节很容易被误写成“调参数让 checkpoint 更平滑”。 那会把主问题带偏。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
bgwriter 读取 clock-sweep 消费速率，估算下一轮需要的 clean reusable buffer 数量，沿 replacement hand 前方扫描，只对未 pin、usage_count 为 0 的 dirty valid buffer 做机会性写出；若它写不够，前台 allocation 路径仍按完整 eviction 协议同步写 dirty victim。
```
这里有三个关键词。 第一，`reusable`。
在本节语境下，一个 buffer 可复用，至少要求：
```text
shared refcount == 0
usage_count == 0
```
这意味着 replacement 算法可以选择它作为 victim。 但它可能仍然是 dirty。
`reusable` 不等于 `clean`。 `clean` 只说明 `BM_DIRTY` 没有设置。
一个 clean buffer 如果仍被 pin，就不是 replacement 可复用对象。 一个 dirty buffer 如果 refcount 和 usage_count 都为 0，它是可复用候选，但复用前要先写出。
所以本节的目标状态不是“clean buffer 越多越好”。 目标状态是：
`clock-sweep 即将消费的区域里，有足够多 clean reusable buffer。` 第二，`smoothing`。
bgwriter 不是按每个 backend 的 miss 请求同步服务。 它没有一个 request queue。
它也不知道下一次 miss 会需要哪个 relation block。 它只能根据：
```text
StrategyControl.nextVictimBuffer
StrategyControl.completePasses
StrategyControl.numBufferAllocs
```
推断全局 replacement 消费速度。 这就是一个反馈控制问题。
反馈来自过去的 allocation。 动作是未来一小段 clock-sweep 区域的 dirty write。
误差不可避免。 第三，`correctness elsewhere`。
即使 bgwriter 完全没有运行，PostgreSQL 仍必须正确。 前台 backend 可以在 `GetVictimBuffer()` 中写 dirty victim。
checkpointer 可以在 checkpoint 中写需要 checkpoint 的 dirty buffers。 `FlushBuffer()` 自己会在 permanent buffer 上执行 WAL-before-data。
`smgrwrite()` 和 sync request / fsync 机制负责把写过的 data file 纳入 checkpoint durability。 bgwriter 的缺席只会改变 latency 和 I/O burst pattern。
它不能改变 crash recovery 的正确性。 这个分层是本节最重要的不变量。
可以把三个角色压缩成一张表：
| 角色 | 主要问题 | 不能误解为 |
| --- | --- | --- |
| 前台 backend | miss 时必须拿到可用 slot | 必须等待 bgwriter 准备好 |
| bgwriter | 平滑可复用 clean buffer 供给 | checkpoint correctness owner |
| checkpointer | checkpoint 集合写出和 fsync 完成 | 负责所有普通前台 miss latency |
本节所有源码细节都围绕这张表展开。
## 3. 核心文件分工与阅读顺序
建议按运行模型读，不按文件名读。
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/postmaster/bgwriter.c` | `BackgroundWriterMain()` 的主循环、错误恢复、latch wait、hibernation、`BgBufferSync()` 调用和 stats report。 |
| 2 | `src/backend/storage/buffer/freelist.c` | `BufferStrategyControl`、`StrategyGetBuffer()` 如何记录 allocation、`StrategySyncStart()` 如何给 bgwriter 读 clock hand 和 alloc count、`StrategyNotifyBgWriter()` 如何唤醒 hibernate 的 bgwriter。 |
| 3 | `src/include/storage/buf_internals.h` | `BufferDesc.state` 的 refcount / usage_count / flags，`BM_DIRTY`、`BM_VALID`、`BM_CHECKPOINT_NEEDED`、`WritebackContext` 和 buffer I/O ResourceOwner。 |
| 4 | `src/backend/storage/buffer/bufmgr.c` | `BgBufferSync()` 的反馈控制，`SyncOneBuffer()` 的单页写出，`FlushBuffer()` 的 WAL-before-data，`BufferAlloc()` / `GetVictimBuffer()` 的前台 fallback。 |
| 5 | `src/backend/postmaster/checkpointer.c` 与 `bufmgr.c` 中 `BufferSync()` | checkpoint 如何标记 `BM_CHECKPOINT_NEEDED`，为什么它和 bgwriter 共享 `SyncOneBuffer()` 但语义不同。 |
| 6 | `src/backend/utils/activity/pgstat_bgwriter.c` | `PendingBgWriterStats` 如何累计并上报 `buffers_clean`、`maxwritten_clean`、`buffers_alloc`。 |
| 7 | `src/backend/utils/activity/pgstat_io.c` 与 `src/backend/catalog/system_views.sql` | `pg_stat_io` 如何按 `backend_type`、`object`、`context`、I/O op 归因 bgwriter 写页和 writeback。 |
| 8 | `src/backend/utils/activity/wait_event_names.txt` | `BGWRITER_MAIN`、`BGWRITER_HIBERNATE`、`DATA_FILE_WRITE`、`DATA_FILE_FLUSH` 等 wait event 的边界。 |
先读 `bgwriter.c` 的文件头注释。 它直接给出两个结论。
第一，background writer 的目标是让普通 backend 尽量不要在为新 page 腾 slot 时写 dirty shared buffer。 第二，从 PostgreSQL 9.2 起，bgwriter 不再处理 checkpoints。
这两句足够定义本节边界。 然后跳到 `freelist.c`。
如果不知道 `numBufferAllocs` 怎么产生，就无法理解 `BgBufferSync()` 为什么不是扫描全池 dirty page。 再回到 `bufmgr.c`。
`BgBufferSync()` 和 `SyncOneBuffer()` 解释了 bgwriter 每一轮做多少事。 `BufferAlloc()` / `GetVictimBuffer()` 解释了 bgwriter 做不够时前台如何兜底。
最后读 stats 和 wait event。 诊断时必须知道哪些现象能直接看到，哪些只能推断。
## 4. 关键数据结构与状态
### 4.1 `BufferDesc.state` 中的 reusable 条件
`src/include/storage/buf_internals.h` 把 buffer state 合在一个 64-bit 原子变量里。 本节关注四组状态：
```text
refcount
usage_count
flags
content lock bits
```
`refcount` 是 replacement 的硬门槛。 只要 `BUF_STATE_GET_REFCOUNT(state) != 0`，这个 buffer slot 就不能被 bgwriter 当成 reusable target 写。
前台 replacement 也不能复用它。 pin 表示某个 backend、I/O path 或内部流程还在持有这个 buffer 的生命周期权利。
pin 不保证 page 内容语义正确。 但它保证 buffer slot 不能被 replacement 换成另一个 tag。
`usage_count` 是 clock sweep 的近似热度。 `usage_count > 0` 表示这个 buffer 近期被访问过。
bgwriter 调 `SyncOneBuffer(..., skip_recently_used = true, ...)`。 这意味着它不会为了平滑未来 miss 去写一个仍然 pinned 或 recently-used 的 page。
这个选择很关键。 bgwriter 的目标不是“尽量多写 dirty page”。
它的目标是“写那些 replacement 很可能马上要消费的 dirty page”。 `BM_DIRTY` 表示 data page 内容需要写出。
`BM_VALID` 表示 page 内容有效。 `BM_TAG_VALID` 表示 buffer mapping table 中有这个 tag 对应关系。
`BM_IO_IN_PROGRESS` 表示这个 buffer 当前有读或写 I/O 正在进行。 `BM_CHECKPOINT_NEEDED` 表示 checkpoint 开始时该 page 属于 checkpoint 需要处理的集合。
把这些位组合起来，本节最重要的状态是：
```text
refcount == 0
usage_count == 0
BM_VALID
BM_DIRTY
```
这表示： `这个 page 当前没有使用者，也不是最近使用对象，内容有效但仍脏。`
它是 bgwriter 最值得写的对象。 写完以后，`TerminateBufferIO(clear_dirty = true, ...)` 会清掉 `BM_DIRTY`。
如果这个 page 同时带有 `BM_CHECKPOINT_NEEDED`，成功写出也会清掉这个 flag。 这会减少后续 checkpointer 写页工作。
但这只是共享写出结果，不是 bgwriter 接管 checkpoint。
### 4.2 `BufferStrategyControl` 是需求信号，不是 dirty page 索引
`src/backend/storage/buffer/freelist.c` 中的 `BufferStrategyControl` 是 shared memory。 本节关注四个字段：
```text
buffer_strategy_lock
nextVictimBuffer
completePasses
numBufferAllocs
bgwprocno
```
`nextVictimBuffer` 是 clock-sweep hand。 它不是一个永远落在 `[0, NBuffers)` 的普通 index。
源码注释说明它只递增，实际访问 descriptor 时再取 modulo。 `completePasses` 是完整扫过 buffer pool 的次数。
`StrategySyncStart()` 需要在 spinlock 下读 `nextVictimBuffer` 和 `completePasses`，给 bgwriter 一个一致的 hand 位置。 `numBufferAllocs` 是 allocation 计数。
`StrategyGetBuffer()` 在全局 clock sweep 路径中递增它。 如果 strategy ring 直接复用自己的 buffer 并提前返回，这个复用不会计入 `numBufferAllocs`。
这说明 `buffers_alloc` 不是“所有 SQL 读取了多少 block”。 它是 bgwriter 控制回路的需求信号。
这个信号有意忽略一部分 ring reuse。 因为 bulk read / bulk write / vacuum 的 ring 本来就是为了限制对全局 shared buffer pool 的污染。
`bgwprocno` 是 hibernation wakeup 的轻量机制。 当 bgwriter 决定长睡时，它把自己的 proc number 写入这里。
下一次 `StrategyGetBuffer()` 看到这个值，会清掉它并 `SetLatch()` 唤醒 bgwriter。 这不是 correctness 信号。
源码注释承认这里可能把 latch 设置给错误进程，最坏结果也是多余或错过一次唤醒。 因为 hibernation 有 timeout，前台也有 fallback。
### 4.3 `BgBufferSync()` 的静态状态
`BgBufferSync()` 中有几组 `static` 状态。 这些状态只属于 bgwriter 进程本地。
它们不是 shared memory。 主要包括：
```text
saved_info_valid
prev_strategy_buf_id
prev_strategy_passes
next_to_clean
next_passes
smoothed_alloc
smoothed_density
```
`prev_strategy_*` 用于计算 clock-sweep hand 自上轮以来前进了多少 buffer。 `next_to_clean` 和 `next_passes` 表示 bgwriter 已经扫描到哪里。
如果 bgwriter 落后于 clock-sweep hand，它会直接跳到 strategy point。 它不会执着补扫旧区域。
原因很实际： 旧区域已经被前台 replacement 消费或重新改变状态。
bgwriter 追着旧状态写，只会增加无效工作。 `smoothed_alloc` 是近期 allocation 的平滑值。
它有 fast attack / slow decline 行为。 如果 allocation 增加，它立即追上。
如果 allocation 下降，它按 `smoothing_samples` 逐步下降。 这让 bgwriter 对突然增压更敏感，对短暂空闲不立刻降到零。
`smoothed_density` 是“扫多少 buffer 才能找到一个 reusable buffer”的估计。 初始值是 `10.0`。
这不是系统不变量。 它只是控制回路的初始猜测。
真实 density 随 workload、shared_buffers、hot set、pin pattern 和 dirty ratio 变化。
### 4.4 `WritebackContext` 是 OS writeback hint 队列
`WritebackContext` 定义在 `buf_internals.h`。 它只保存 pending writeback 的 `BufferTag` 数组和上限指针。
`bgwriter.c` 在启动时调用：
```text
WritebackContextInit(&wb_context, &bgwriter_flush_after)
```
当 `SyncOneBuffer()` 写出一个 dirty buffer 后，会调用：
```text
ScheduleBufferTagForWriteback(wb_context, IOCONTEXT_NORMAL, &tag)
```
如果 pending 数达到 `bgwriter_flush_after` 指向的限制，`IssuePendingWritebacks()` 会排序、合并相邻 block，并调用 `smgrwriteback()`。 这只是告诉 OS 尽早 writeback 数据。
它不携带 LSN。 它不记录 fsync 成功。
它不保证数据 durable。 如果 writeback hint 没有生效，correctness 仍由 WAL 和 checkpoint fsync 保证。
### 4.5 `PendingBgWriterStats` 是累计统计，不是当前状态
`PendingBgWriterStats` 是 bgwriter 本地 pending counters。 `BgBufferSync()` 每轮更新：
```text
PendingBgWriterStats.buf_alloc += recent_alloc
PendingBgWriterStats.buf_written_clean += num_written
PendingBgWriterStats.maxwritten_clean++
```
`BackgroundWriterMain()` 每轮调用：
```text
pgstat_report_bgwriter()
pgstat_report_wal(true)
```
`pgstat_report_bgwriter()` 把 pending counters 累加到 cumulative stats shared entry，然后清零 pending 结构。 `pg_stat_bgwriter` 看到的是累计值。
它不是当前 buffer pool 中 clean reusable buffer 的数量。 PostgreSQL 没有直接暴露 `next_to_clean`、`smoothed_alloc`、`smoothed_density` 或当前 reusable density。
这些只能通过源码断点、debug build 或统计现象近似推断。
## 5. 主流程源码 walkthrough
### 5.1 前台 miss 先定义需求
先从前台路径看。 `BufferAlloc()` 在 lookup miss 后会释放 new tag 的 mapping partition lock。
然后调用：
```text
GetVictimBuffer(strategy, io_context)
```
`GetVictimBuffer()` 继续调用：
```text
StrategyGetBuffer(strategy, &buf_state, &from_ring)
```
`StrategyGetBuffer()` 从 strategy ring 或全局 clock sweep 找 candidate。 全局路径中，它会：
```text
pg_atomic_fetch_add_u32(&StrategyControl->numBufferAllocs, 1)
```
这就是 bgwriter 后续看到的 allocation 信号。 如果 `StrategyGetBuffer()` 找到的 candidate 是 clean，前台后续成本相对低。
它可以 invalid old tag，然后安装 new tag。 如果 candidate 是 dirty，前台必须在 `GetVictimBuffer()` 中处理 dirty write。
路径包括：
```text
conditional content lock
optional XLogNeedsFlush() ring rejection
FlushBuffer()
ScheduleBufferTagForWriteback(&BackendWritebackContext, ...)
InvalidateVictimBuffer()
```
这条路径就是 bgwriter 想减少的前台慢路径。 它不是想替代这条路径。
它只是提前把一部分未来 candidate 从 dirty 变 clean。 所以前台 miss 是需求来源。
bgwriter 是异步供应调节。
### 5.2 `BackgroundWriterMain()` 的外层循环
`BackgroundWriterMain()` 是 bgwriter auxiliary process 的入口。 启动后，它安装信号处理。
`SIGHUP` 用于 reload config。 `SIGTERM` 用于正常退出。
`SIGQUIT` 按 backend crash 方式处理。 然后它创建独立 memory context：
```text
Background Writer
```
这个 context 用于错误恢复后 reset，避免在 `TopMemoryContext` 中泄漏长期对象。 它初始化：
```text
WritebackContext wb_context
```
主循环每次做这几件事：
```text
ResetLatch(MyLatch)
ProcessMainLoopInterrupts()
can_hibernate = BgBufferSync(&wb_context)
pgstat_report_bgwriter()
pgstat_report_wal(true)
maybe smgrdestroyall() after checkpoint
maybe LogStandbySnapshot()
WaitLatch(... WAIT_EVENT_BGWRITER_MAIN)
maybe hibernate with WAIT_EVENT_BGWRITER_HIBERNATE
```
这里有两个容易忽略的事实。 第一，`BgBufferSync()` 是每轮 dirty-buffer smoothing 的唯一核心调用。
第二，bgwriter 主循环还承担一些“周期性后台进程方便做”的杂务。 例如定期记录 running xacts snapshot，帮助 standby 更快进入一致状态。
这不是本节主问题。 不要把它和 reusable clean buffer 混在一起。
本节只把它作为边界说明： `bgwriter 进程做的事不全是 buffer smoothing，但本节只研究 buffer smoothing。`
### 5.3 `StrategySyncStart()` 读取需求信号
`BgBufferSync()` 的第一步是：
```text
strategy_buf_id = StrategySyncStart(&strategy_passes, &recent_alloc)
```
`StrategySyncStart()` 在 `freelist.c` 中做三件事。 第一，读当前 `nextVictimBuffer`。
第二，返回 `completePasses`。 第三，如果调用者传入 `num_buf_alloc`，用 atomic exchange 读出并清零 `numBufferAllocs`。
这意味着每一轮 bgwriter 看到的是：
```text
自上轮 BgBufferSync() 以来全局 clock-sweep allocation 的数量
```
不是当前未命中的 backend 数。 不是 dirty buffer 数。
不是 shared buffer hit/miss。 这就是 `pg_stat_bgwriter.buffers_alloc` 的来源。
`BgBufferSync()` 立刻把它记到 pending stats：
```text
PendingBgWriterStats.buf_alloc += recent_alloc
```
然后才开始计算本轮扫描目标。
### 5.4 如果 bgwriter LRU scan 被关闭
`BgBufferSync()` 读完 stats 后检查：
```text
if (bgwriter_lru_maxpages <= 0)
```
如果为 true，它设置：
```text
saved_info_valid = false
return true
```
这一步很有边界感。 即使 background writing 被禁用，bgwriter 仍然读 allocation stats 并上报。
但它不做 cleaning scan。 下一次重新打开 LRU scan 时，旧的 `next_to_clean`、density 位置不能继续信任。
所以它让 saved state 失效。 返回 `true` 表示可以 hibernate。
禁用 bgwriter LRU scan 不会破坏 correctness。 代价是前台 backend 和 checkpointer 要吸收更多写出压力。
### 5.5 判断 bgwriter 是 ahead 还是 behind
如果 saved info 有效，`BgBufferSync()` 计算：
```text
strategy_delta = 当前 strategy point - 上轮 strategy point
```
这个 delta 还要加上完整 pass 差值乘以 `NBuffers`。 它表示 clock-sweep hand 在两轮之间前进了多少 buffer。
然后 bgwriter 比较自己的 `next_to_clean` 和 strategy point。 如果 bgwriter 仍在 strategy point 前方，它继续从 `next_to_clean` 扫。
如果 bgwriter 已经落后，它直接跳到 strategy point。 源码里的意思是：
```text
如果已经追不上前台消费，就不要补扫旧区域；
从当前 replacement hand 开始重新预清理。
```
这和 bgwriter 的 smoothing 定位一致。 它不是历史债务清算器。
它只服务未来 allocation。
### 5.6 用 density 估算需要扫多少
`BgBufferSync()` 接着估算两个量。 第一，近期 allocation 速率。
```text
smoothed_alloc
```
第二，扫描到 reusable buffer 的密度。
```text
smoothed_density
```
如果 `strategy_delta > 0` 且 `recent_alloc > 0`，它计算：
```text
scans_per_alloc = strategy_delta / recent_alloc
```
然后用 `smoothing_samples = 16` 更新 `smoothed_density`。 这不是精确数学模型。
它是一个低成本近似。 bgwriter 不愿在每轮都全池统计 dirty/reusable 比例。
它只利用 clock-sweep 已经产生的消费轨迹。 随后它估计：
```text
bufs_ahead = NBuffers - bufs_to_lap
reusable_buffers_est = bufs_ahead / smoothed_density
upcoming_alloc_est = smoothed_alloc * bgwriter_lru_multiplier
```
`reusable_buffers_est` 表示 bgwriter 已经扫在前方的区域里，估计有多少 reusable buffer。 `upcoming_alloc_est` 表示下一轮可能需要多少 reusable buffer。
如果估计已经足够，就少扫。 如果估计不足，就继续扫并写。
### 5.7 空闲系统也要缓慢扫完整个 pool
`BgBufferSync()` 还有一个最小扫描量：
```text
scan_whole_pool_milliseconds = 120000.0
min_scan_buffers = NBuffers / (scan_whole_pool_milliseconds / BgWriterDelay)
```
这意味着即使 allocation 很少，bgwriter 也会缓慢推进。 目标是：
`系统空闲一段时间后，尽量让更多 reusable buffer 已经 clean。` 这不是 correctness 要求。
它是 latency smoothing。 一个典型场景是业务短暂写入后进入空闲。
如果 bgwriter 完全停止，下一波读写可能马上碰到 dirty victims。 如果它在空闲期慢慢清理，下一波负载的前台 miss 延迟更平滑。
### 5.8 扫描循环只写 reusable dirty buffer
真正写页的循环是：
```text
while (num_to_scan > 0 && reusable_buffers < upcoming_alloc_est)
{
    sync_state = SyncOneBuffer(next_to_clean, true, wb_context);
    advance next_to_clean;
    if (sync_state & BUF_WRITTEN)
        reusable_buffers++;
    else if (sync_state & BUF_REUSABLE)
        reusable_buffers++;
}
```
这里传给 `SyncOneBuffer()` 的第二个参数是 `true`。 含义是：
```text
skip_recently_used = true
```
这决定了 bgwriter 的行为。 它不会写 pinned buffer。
它不会写 usage_count 非零的 buffer。 它只写 replacement 可能马上消费的对象。
如果写出的页数达到：
```text
bgwriter_lru_maxpages
```
它增加：
```text
PendingBgWriterStats.maxwritten_clean
```
然后结束本轮。 `maxwritten_clean` 的语义不是“写失败”。
它表示本轮因为配置上限停止。 如果这个计数持续增长，说明 bgwriter 被 `bgwriter_lru_maxpages` 限住，前台 fallback 写 dirty victim 的概率可能增加。
但这仍然是性能信号，不是 correctness 信号。
### 5.9 `SyncOneBuffer()` 的单页状态转移
`SyncOneBuffer()` 接收 `buf_id`。 它先为 pin 和 ResourceOwner 做准备：
```text
ReservePrivateRefCountEntry()
ResourceOwnerEnlarge(CurrentResourceOwner)
```
然后 lock buffer header，检查：
```text
refcount == 0
usage_count == 0
```
如果成立，它把 `BUF_REUSABLE` 放入返回值。 如果不成立且 `skip_recently_used` 为 true，它直接返回。
这就是 bgwriter 跳过 hot / pinned buffer 的关键。 接着它检查：
```text
BM_VALID
BM_DIRTY
```
如果无效或不脏，没什么可写。 如果有效且脏，它调用：
```text
PinBuffer_Locked(bufHdr)
FlushUnlockedBuffer(bufHdr, NULL, IOOBJECT_RELATION, IOCONTEXT_NORMAL)
UnpinBuffer(bufHdr)
ScheduleBufferTagForWriteback(wb_context, IOCONTEXT_NORMAL, &tag)
```
`PinBuffer_Locked()` 的作用是，在写出期间保护 buffer slot 不被 replacement 偷走。 `FlushUnlockedBuffer()` 会拿 content lock，然后进入 `FlushBuffer()`。
`FlushBuffer()` 成功后，`TerminateBufferIO(clear_dirty = true, ...)` 会清 `BM_DIRTY` 和 `BM_CHECKPOINT_NEEDED`。 最后 `SyncOneBuffer()` 返回 `BUF_WRITTEN`。
从 bgwriter 角度看，一个 dirty reusable candidate 变成了 clean reusable candidate。 从 checkpoint 角度看，如果它本来是 checkpoint-needed page，那么它也已经被写过。
但 checkpoint 仍要由 checkpointer 负责整体 completion 和 fsync。
### 5.10 `FlushBuffer()` 仍然执行 WAL-before-data
bgwriter 写 dirty buffer 时不会绕过 WAL 规则。 `FlushBuffer()` 读取 page LSN：
```text
recptr = BufferGetLSN(buf)
```
如果 buffer 带 `BM_PERMANENT`，它执行：
```text
XLogFlush(recptr)
```
然后才调用 `smgrwrite()` 写 data page。 这说明 bgwriter 虽然不负责 checkpoint correctness，但它写 data page 时仍必须遵守 WAL-before-data。
这里的分层是：
```text
WAL-before-data 是每次数据页写出的局部正确性规则；
checkpoint correctness 是某个 checkpoint 集合最终 durable 的全局规则。
```
bgwriter 参与前者。 它不拥有后者。
`FlushBuffer()` 成功写完后会记录 I/O stats：
```text
pgstat_count_io_op_time(IOOBJECT_RELATION, IOCONTEXT_NORMAL, IOOP_WRITE, ...)
```
因为调用者是 bgwriter，`pg_stat_io` 中这次写会归到 `backend_type = 'background writer'`。
### 5.11 hibernation 和 allocation wakeup
`BgBufferSync()` 返回：
```text
bufs_to_lap == 0 && recent_alloc == 0
```
表示 bgwriter 认为没有需要继续做的 cleaning work。 `BackgroundWriterMain()` 不会第一次看到这个结果就长睡。
它需要连续两轮满足：
```text
can_hibernate && prev_hibernate
```
然后调用：
```text
StrategyNotifyBgWriter(MyProcNumber)
WaitLatch(... BgWriterDelay * HIBERNATE_FACTOR, WAIT_EVENT_BGWRITER_HIBERNATE)
StrategyNotifyBgWriter(-1)
```
`HIBERNATE_FACTOR` 当前是 50。 默认 `BgWriterDelay = 200ms` 时，长睡 timeout 大约是 10 秒。
如果有 backend 再次分配 buffer，`StrategyGetBuffer()` 会看到 `bgwprocno`，清掉它，并设置 bgwriter latch。 源码注释说明这里有 race：
backend 可能在 bgwriter 看到 zero alloc 和设置 notify 之间分配了 buffer。 解决方式不是建立更强同步。
而是：
```text
要求连续两轮 hibernate 条件；
并且 hibernate 有 timeout；
前台 allocation 有 fallback。
```
这再次说明 hibernation 是能耗和 wakeup 优化，不是 correctness 机制。
### 5.12 与 checkpointer 的共享函数和不同语义
`SyncOneBuffer()` 也被 checkpointer 使用。 区别在调用参数和外层语义。
checkpointer 的 `BufferSync()` 在 checkpoint 开始时扫描全池。 它为 checkpoint 需要的 dirty permanent buffers 设置：
```text
BM_CHECKPOINT_NEEDED
```
随后按排序和平衡 tablespace 的方式写这些 buffers。 它调用：
```text
SyncOneBuffer(buf_id, false, &wb_context)
```
这里 `skip_recently_used = false`。 因为 checkpoint 不能因为 page 最近使用过就永远不写。
checkpoint 需要完成一个 redo pointer 对应的 durable 边界。 bgwriter 则调用：
```text
SyncOneBuffer(buf_id, true, wb_context)
```
因为 bgwriter 不该追 hot pages。 它只清理 future replacement candidates。
共享函数不表示共享职责。 这是读源码时必须警惕的点。
一个 helper 被两个后台进程调用，不代表两个后台进程承担同一层 correctness。
## 6. 生命周期 / ownership / cleanup
### 6.1 shared strategy state 的生命周期
`StrategyControl` 在共享内存中创建。 `StrategyCtlShmemRequest()` 请求名为：
```text
Buffer Strategy Status
```
的 shared memory。 `StrategyCtlShmemInit()` 初始化：
```text
buffer_strategy_lock
nextVictimBuffer = 0
completePasses = 0
numBufferAllocs = 0
bgwprocno = -1
```
这个状态被所有 backend 和 bgwriter 共享。 前台 `StrategyGetBuffer()` 更新 allocation 计数和 clock hand。
bgwriter `StrategySyncStart()` 读取并重置 allocation 计数。 `bgwprocno` 用于 hibernation wakeup。
这不是一个拥有者明确的 queue。 它是一个共享的 replacement strategy control block。
### 6.2 bgwriter 进程的生命周期
bgwriter 是 postmaster 启动的 auxiliary process。 正常退出由 `SIGTERM` 驱动。
紧急退出由 `SIGQUIT` 驱动。 源码注释明确：
如果 bgwriter 异常退出，postmaster 把它视为 backend crash。 原因是它接触 shared memory。
shared memory 可能处于不可信状态。 postmaster 会杀掉其他 backends 并启动 recovery cycle。
这不是因为 bgwriter 对 checkpoint correctness 不可或缺。 而是因为任何接触 shared memory 的 backend-like process 异常崩溃，都可能留下不一致的 shared state。
### 6.3 每轮写出的 ownership
`SyncOneBuffer()` 写一个 buffer 前要确保可以记录 pin。 它使用：
```text
ReservePrivateRefCountEntry()
ResourceOwnerEnlarge(CurrentResourceOwner)
```
写出期间：
```text
PinBuffer_Locked()
StartSharedBufferIO()
ResourceOwnerRememberBufferIO()
TerminateBufferIO()
UnpinBuffer()
```
pin 的 owner 是当前 bgwriter 进程的 ResourceOwner。 buffer I/O 的 owner 也记录在 ResourceOwner。
如果正常完成，`TerminateBufferIO(... forget_owner = true, ...)` 会忘掉 buffer I/O owner。 `UnpinBuffer()` 会释放 pin 并从 ResourceOwner forget buffer。
如果发生 ERROR，ResourceOwner cleanup 会释放未完成的 buffer I/O 和 pin。 这就是为什么 bgwriter 顶层 error handler 不只是 reset memory。
它还必须释放 shared resources。
### 6.4 ERROR cleanup 路径
`BackgroundWriterMain()` 使用 `sigsetjmp()` 作为顶层 ERROR 恢复点。 ERROR 后，它执行一组最小化的 abort cleanup：
```text
LWLockReleaseAll()
ConditionVariableCancelSleep()
pgaio_error_cleanup()
UnlockBuffers()
ReleaseAuxProcessResources(false)
AtEOXact_Buffers(false)
AtEOXact_SMgr()
AtEOXact_Files(false)
AtEOXact_HashTables(false)
MemoryContextReset(bgwriter_context)
WritebackContextInit(&wb_context, &bgwriter_flush_after)
pg_usleep(1000000L)
```
这条路径说明 bgwriter 虽然没有用户事务，但它仍然会持有：
```text
LWLocks
condition variable sleep state
buffer locks
buffer pins
buffer I/O ownership
SMgr references
temporary file state
hash table state
```
`ReleaseAuxProcessResources(false)` 会释放 auxiliary process 的 ResourceOwner 资源。 buffer I/O ResourceOwner callback 会走到 `AbortBufferIO()`。
失败的 buffer I/O 会标记 `BM_IO_ERROR` 并清理 `BM_IO_IN_PROGRESS`。 然后等待者会通过 condition variable 被唤醒。
这保证 ERROR 后不会把某个 buffer 永久留在 I/O in progress 状态。 写错误很可能重复发生，所以 error recovery 后 bgwriter 至少 sleep 1 秒。
这避免日志被同一个 I/O error 快速刷爆。
### 6.5 smgr 对象的 cleanup
bgwriter 不处理 shared invalidation messages。 它也不会像普通 backend 那样在每个事务边界完整处理 relcache / smgr 生命周期。
所以主循环里有一段：
```text
if (FirstCallSinceLastCheckpoint())
    smgrdestroyall()
```
源码注释说明： checkpoint 后释放所有 smgr objects。
否则对 dropped relations 的 smgr object 可能一直留在 bgwriter 进程里。 这是一个很典型的 background process cleanup 边界。
bgwriter 可以凭 `BufferTag` 写 buffer。 但它不能假设自己会收到普通 backend 的 cache invalidation 生命周期事件。
因此它用 checkpoint 后的粗粒度 cleanup 兜底。
## 7. 正确性机制层次
### 7.1 pin / refcount 保证 replacement 生命周期
pin 保证 buffer slot 不被 replacement 改 tag。 bgwriter 在 `SyncOneBuffer()` 中只对 `refcount == 0` 的 buffer 继续处理。
写出前它自己 pin 住 buffer。 这样在写出过程中，其他 replacement path 不能把这个 slot 偷走。
pin 不保证 page 内容不变。 page content 的稳定性由 content lock 负责。
所以正确性第一层是：
```text
refcount / pin: slot lifecycle
content lock: page content stability
```
### 7.2 header lock / atomic state 保证 descriptor 组合状态
`BufferDesc.state` 可以通过 atomic CAS 更新。 某些组合变化仍需要 buffer header lock。
`SyncOneBuffer()` lock header 后同时判断 refcount、usage_count、flags。 它不能只读一个字段就得出语义。
例如： `BM_DIRTY` 有值只说明 page 需要写出。
如果 refcount 非零或 usage_count 非零，bgwriter 仍会跳过。 `usage_count == 0` 也不说明可以覆盖。
如果 `BM_DIRTY` 还在，前台复用前仍要写。 这就是标准里强调的：
```text
raw field 不是语义；
field + flag + lifecycle state + lock context 才是语义。
```
### 7.3 WAL-before-data 保证单页写出顺序
`FlushBuffer()` 在 permanent buffer 上先执行 `XLogFlush(page LSN)`。 然后才 `smgrwrite()` data page。
这保证 crash 后不会看到一个 data page 已经包含某次修改，而 WAL 中没有对应 redo record。 bgwriter 写页必须遵守这条规则。
checkpointer 写页也必须遵守。 前台 backend 写 dirty victim 也必须遵守。
所以 WAL-before-data 是 `FlushBuffer()` 级别的 invariant。 它不属于某一个后台进程。
### 7.4 checkpoint correctness 由 checkpointer 和 checkpoint 协议保证
checkpoint 的关键状态是 `BM_CHECKPOINT_NEEDED`、checkpoint redo pointer、checkpointer 写出进度和 fsync completion。 `BufferSync()` 在 checkpoint 开始时标记需要处理的 dirty permanent buffers。
之后即使某个 page 被前台或 bgwriter 写掉，`TerminateBufferIO(clear_dirty = true, ...)` 也会清掉 `BM_CHECKPOINT_NEEDED`。 这是一种协作。
但 checkpoint completion 的判断仍由 checkpointer 负责。 `pg_stat_checkpointer.buffers_written` 也只统计 checkpointer 自己写的 buffers。
它不会把 bgwriter 或前台写的页加进去。 因此不能用：
```text
pg_stat_bgwriter.buffers_clean + pg_stat_checkpointer.buffers_written
```
简单推出某次 checkpoint 的完整写页集合。 它们是不同统计口径。
### 7.5 writeback hint 不是 durability
`ScheduleBufferTagForWriteback()` 和 `IssuePendingWritebacks()` 的目标是改善 OS I/O 调度。 它们调用的是 `smgrwriteback()`。
`smgrwriteback()` 不是 `fsync()`。 如果 `enableFsync` 关闭，或者 direct I/O 配置使 writeback hint 无意义，源码会直接跳过 pending writeback。
即使 writeback hint 执行成功，也只是让内核提前回写某个范围。 它不提供 checkpoint durable 边界。
诊断时不要把 `pg_stat_io.writebacks` 当成 persisted pages。
### 7.6 latch / wait event 只保证进程调度，不保证工作完成
`WAIT_EVENT_BGWRITER_MAIN` 表示 bgwriter 主循环睡眠。 `WAIT_EVENT_BGWRITER_HIBERNATE` 表示它进入长睡。
这些 wait events 说明 background writer 当前没在执行 CPU work。 它们不说明 buffer pool 中没有 dirty page。
也不说明下一次 foreground miss 一定不会写 dirty victim。 latch 是 wakeup primitive。
它不是 work queue completion primitive。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 bgwriter 写不够时，前台写 dirty victim
最重要的 fallback 是前台路径。 如果 bgwriter 没有提前写出足够 clean reusable buffers，`GetVictimBuffer()` 会自己处理 dirty victim。
这包括：
```text
拿 content lock
必要时 flush WAL
smgrwrite data page
清 BM_DIRTY
登记 writeback
invalidate old tag
```
因此前台 backend 出现 data file write 不一定是 bug。 它可能只是 bgwriter 的平滑能力不够，或者 workload 让 dirty victim 不可避免。
常见原因包括：
- dirty working set 大于 bgwriter 每轮可写上限。
- `bgwriter_lru_maxpages` 太低。
- `bgwriter_delay` 太大。
- 大量 buffer 被 pin 或 usage_count 非零，bgwriter 按设计跳过。
- workload 使用 bulk strategy ring，global allocation 信号不代表全部局部复用压力。
- WAL flush 或 data file write 本身很慢。
### 8.2 `bgwriter_lru_maxpages = 0` 禁用 cleaning scan
如果 `bgwriter_lru_maxpages` 设置为 0，`BgBufferSync()` 不做 LRU cleaning scan。 它仍然上报 allocation stats。
它会返回可以 hibernate。 这不会破坏 WAL 或 checkpoint。
但前台 dirty victim 写和 checkpointer 写页压力会更明显。 这是一个显式性能退化模式。
不要把它解释成 checkpoint 被禁用。
### 8.3 每轮 hit maxpages 是压力信号
如果 `num_written >= bgwriter_lru_maxpages`，本轮停止并递增 `maxwritten_clean`。 这意味着 bgwriter 还有可能继续写，但配置上限让它停下。
持续增长的 `maxwritten_clean` 通常说明：
```text
bgwriter supply cap < workload dirty reusable demand
```
但这个判断仍要结合 `pg_stat_io` 看前台 backend 写页是否上升。 如果前台写页不高，`maxwritten_clean` 上升可能只是 bgwriter 被设置得很保守，但系统实际还可接受。
### 8.4 hibernation missed wakeup 不影响 correctness
源码注释承认： backend 可能在 `BgBufferSync()` 看到 no allocations 之后、`StrategyNotifyBgWriter()` 设置通知之前分配 buffer。
这可能导致 bgwriter 仍然进入 hibernation。 PostgreSQL 没有为这个 race 设计复杂同步。
原因是后果有限。 第一，hibernation 有 timeout。
第二，下一个 allocation 仍可能唤醒它。 第三，前台 allocation 可以 fallback。
这是一种典型的性能路径 race： `允许偶发 missed optimization，不允许破坏 state invariant。`
### 8.5 `StartSharedBufferIO()` 发现别人已完成 I/O
`FlushBuffer()` 会调用 `StartSharedBufferIO(buf, false, true, NULL)`。 如果返回 `BUFFER_IO_ALREADY_DONE`，说明在它准备写时，buffer 已经不需要写了。
这可能是其他 backend 已经完成了写出。 `FlushBuffer()` 直接返回。
这种路径说明 bgwriter 的单页行为也不是“我看到 dirty 就必然写”。 shared buffer 状态可以在它检查后改变。
正确做法是重新验证并让 I/O ownership bit 串行化实际 I/O。
### 8.6 写错误后的 buffer I/O cleanup
如果 `FlushBuffer()` 过程中发生 ERROR，bgwriter 顶层 handler 会释放 ResourceOwner。 buffer I/O ResourceOwner callback 调用 `AbortBufferIO()`。
失败的 buffer 会被标记：
```text
BM_IO_ERROR
```
并清掉：
```text
BM_IO_IN_PROGRESS
```
等待该 buffer I/O 的 backend 会被 condition variable 唤醒。 buffer 仍可能保持 dirty。
后续写出可能再次失败。 如果多次失败，源码会发出 warning，提示写错误可能是永久性的。
这条路径的目标不是吞掉错误。 它的目标是：
`不让 shared buffer 卡在半完成 I/O 状态。`
### 8.7 bgwriter 异常退出由 postmaster 提升为 crash recovery
如果 bgwriter process 异常退出，postmaster 不会简单重启一个 bgwriter 并继续。 源码注释说得很直接：
shared memory 可能已经被破坏。 因此它会按 backend crash 处理。
这说明“bgwriter 不负责 checkpoint correctness”不能推导成“bgwriter 崩了无所谓”。 不负责某个语义层，不等于它不参与 shared memory correctness。
## 9. 成本、资源与跨模块传播
### 9.1 控制回路成本随 `NBuffers` 和 delay 扩张
bgwriter 每轮可能扫描多个 buffer descriptors。 最小扫描目标包含：
```text
NBuffers / (120000ms / BgWriterDelay)
```
因此 shared_buffers 越大，空闲期为了两分钟扫完整个 pool 的每轮最小扫描量越大。 `BgWriterDelay` 越小，唤醒越频繁。
每轮最小扫描量可能下降，但总 wakeup / loop overhead 上升。 `BgWriterDelay` 越大，单轮间隔更长。
反馈控制滞后可能增加。 这不是单调优化问题。
### 9.2 I/O 成本随 dirty reusable density 和 WAL flush 扩张
bgwriter 写一个 dirty reusable buffer 时，成本可能包括：
```text
buffer header lock
content lock
StartSharedBufferIO
XLogFlush(page LSN)
smgrwrite()
pgstat accounting
ScheduleBufferTagForWriteback()
```
如果 WAL flush 落后，`XLogFlush()` 可能让 bgwriter 自己推动 WAL 写出和 flush。 如果 data file write 慢，bgwriter 会消耗 I/O queue。
如果 writeback hint 合并较差，`smgrwriteback()` 也可能形成额外系统调用。 这些成本最终会和前台 backend、checkpointer、autovacuum、walwriter 竞争同一套存储资源。
### 9.3 `bgwriter_lru_multiplier` 是需求估算放大器
`upcoming_alloc_est` 来自：
```text
smoothed_alloc * bgwriter_lru_multiplier
```
更大的 multiplier 会让 bgwriter 对未来 allocation 做更激进准备。 它可能减少前台 dirty victim 写。
也可能把更多 I/O 提前打到系统上。 如果 workload 很快又把同一批 page 弄脏，过激的 bgwriter 可能产生无效写放大。
所以它不是越大越好。 它调的是：
```text
前台 latency smoothing vs 后台提前写放大
```
### 9.4 `bgwriter_lru_maxpages` 是每轮输出上限
`bgwriter_lru_maxpages` 限制每轮最多写多少 buffer。 它保护系统不被 bgwriter 单轮写爆。
但如果它低于实际 dirty reusable 供给缺口，`maxwritten_clean` 会持续上升。 此时前台 fallback 写页可能增加。
这就是一个典型资源传播：
```text
bgwriter per-round cap
  -> clean reusable buffer supply 不足
  -> foreground GetVictimBuffer() 写 dirty victim
  -> query latency 抖动
  -> pg_stat_io 中 client backend writes/write_time 上升
```
### 9.5 ring strategy 会改变 bgwriter 看到的 demand
`StrategyGetBuffer()` 在 strategy object 的 ring 直接命中时，不递增 `numBufferAllocs`。 这符合 ring 的设计目标。
例如大顺序扫描不应该把全局 pool 的 allocation 信号夸大到完全代表扫描规模。 但诊断时要记住：
`pg_stat_bgwriter.buffers_alloc` 不等于所有 block access pressure。如果 workload 大量使用 bulkread / bulkwrite / vacuum ring，bgwriter 的需求信号和 SQL 读写规模之间会更间接。
这不表示统计错。 这是替换策略边界。
### 9.6 与相邻模块的资源传播
bgwriter 连接至少六个模块。 第一，buffer replacement。
它读取 `StrategyControl`，沿 `next_to_clean` 扫 buffer descriptors。 第二，WAL。
`FlushBuffer()` 在 permanent page 上执行 `XLogFlush(page LSN)`。 第三，smgr / md。
`smgrwrite()` 把 block 写给 relation file，`smgrwriteback()` 请求 OS writeback。 第四，checkpointer。
bgwriter 写掉的 dirty page 可能清 `BM_CHECKPOINT_NEEDED`，但 checkpoint completion 仍由 checkpointer 管。 第五，stats。
`pgstat_report_bgwriter()` 上报 bgwriter counters，并 flush I/O stats。 第六，latch / wait events。
`WaitLatch()` 让 bgwriter 以 fixed delay 或 hibernation 模式睡眠。 跨模块传播路径可以画成：
```text
foreground allocation
  -> StrategyControl.numBufferAllocs
  -> BgBufferSync estimates future demand
  -> SyncOneBuffer writes selected dirty reusable buffers
  -> FlushBuffer may flush WAL and write data file
  -> ScheduleBufferTagForWriteback may ask OS for writeback
  -> pg_stat_bgwriter / pg_stat_io expose cumulative symptoms
  -> foreground fallback absorbs misses when supply is insufficient
```
这条链没有任何一步把 bgwriter 变成 checkpoint owner。
## 10. 观测与诊断入口
### 10.1 能直接观测什么
`pg_stat_bgwriter` 当前视图包含：
```sql
SELECT *
FROM pg_stat_bgwriter;
```
重要列是：
```text
buffers_clean
maxwritten_clean
buffers_alloc
stats_reset
```
`buffers_clean` 是 bgwriter 写出的 clean buffers 累计数。 `maxwritten_clean` 是 bgwriter 因每轮 `bgwriter_lru_maxpages` 上限停止的累计次数。
`buffers_alloc` 是 `StrategySyncStart()` 读到的 allocation 计数累计值。 这些都是 instance-level cumulative stats。
它们不是单 query 指标。 它们也不是当前 buffer pool 状态。
`pg_stat_io` 可以按 backend type 看写页归因：
```sql
SELECT backend_type,
       object,
       context,
       writes,
       write_time,
       writebacks,
       writeback_time,
       evictions,
       reuses
FROM pg_stat_io
WHERE backend_type IN ('background writer', 'client backend', 'checkpointer')
  AND object = 'relation'
ORDER BY backend_type, context;
```
这里可以区分：
```text
background writer 的 normal relation writes
client backend 的 foreground dirty victim writes
checkpointer 的 checkpoint writes
```
具体可见行取决于当前版本、配置和是否有对应 I/O。 `pg_stat_activity` 可以看 bgwriter 当前 wait：
```sql
SELECT pid, backend_type, wait_event_type, wait_event
FROM pg_stat_activity
WHERE backend_type = 'background writer';
```
常见 wait event：
```text
Activity / BgWriterMain
Activity / BgWriterHibernate
IO / DataFileWrite
IO / DataFileFlush
```
`BgWriterMain` 或 `BgWriterHibernate` 表示它正在睡。 `DataFileWrite` 表示它正在等待 relation data file write。
这些 wait event 是瞬时状态。 不要把一次采样当成完整因果。
### 10.2 只能推断什么
以下状态不能从 SQL 直接读到：
```text
next_to_clean
next_passes
smoothed_alloc
smoothed_density
bufs_to_lap
reusable_buffers_est
upcoming_alloc_est
```
也不能直接看到：
```text
当前有多少 clean reusable buffer
当前 clock-sweep 前方 dirty reusable density
下一次 foreground miss 是否会写 dirty victim
```
只能通过组合现象推断。 例如：
```text
maxwritten_clean 持续上升
background writer writes 上升
client backend writes 仍上升
foreground latency 抖动
checkpoint writes 不一定同步上升
```
这更像： `bgwriter supply 仍低于 replacement dirty pressure。`
但还要排除：
```text
WAL flush latency
storage write saturation
long pin / hot buffer
bulk strategy ring
checkpoint 同期 I/O
autovacuum I/O
```
### 10.3 几乎不可见的状态
单个 buffer 是否被 bgwriter 写成 clean，在普通 SQL 视图里不可见。 `BufferDesc.state` 当前组合状态也不可见。
要精确观察，需要：
```text
gdb breakpoint
临时 debug log
自建 instrumentation
perf / flamegraph
tracepoints
```
例如可以断在：
```text
BgBufferSync
StrategySyncStart
SyncOneBuffer
FlushBuffer
GetVictimBuffer
```
观察：
```text
recent_alloc
num_written
reusable_buffers
sync_state
bufHdr->state
```
这类实验适合源码学习。 生产诊断不能依赖它们。
### 10.4 一条实战诊断链
看到现象：
```text
高写入负载下 query latency 偶发抖动；
pg_stat_io 中 client backend relation writes/write_time 上升；
pg_stat_bgwriter.maxwritten_clean 持续增长；
background writer writes 也增长。
```
先用本节模型解释：
```text
前台 miss 经常拿到 dirty victim；
bgwriter 每轮也在写，但 hit 了 maxpages cap；
clean reusable buffer supply 不足以覆盖 foreground allocation rate。
```
再回源码验证：
```text
BgBufferSync() 在 num_written >= bgwriter_lru_maxpages 时停止；
GetVictimBuffer() 在 dirty victim 上自己 FlushBuffer()；
FlushBuffer() 对 permanent page 可能先 XLogFlush()，再 smgrwrite()。
```
最后确认边界：
```text
这不是 checkpoint correctness 问题；
要同时看 pg_stat_checkpointer、pg_stat_wal、storage latency 和 workload dirty rate。
```
## 11. 常见误区
误区一： `bgwriter 负责 checkpoint。`
不对。 当前基线 `bgwriter.c` 注释明确说明 PostgreSQL 9.2 起 bgwriter 不再处理 checkpoints。
checkpoint 的核心 worker 是 checkpointer。 bgwriter 可能顺手写掉带 `BM_CHECKPOINT_NEEDED` 的 dirty page。
但 checkpoint completion 和 fsync 不是它负责。 误区二：
`buffers_clean 越高，checkpoint 越安全。` 不对。
`buffers_clean` 是 bgwriter 写出的 clean buffers 累计数。 它不是 checkpoint durable pages。
安全性来自 WAL-before-data、checkpoint write/sync 协议和 crash recovery。 误区三：
`buffers_alloc 是 shared buffer miss 数。` 不准确。
它来自 `StrategyControl.numBufferAllocs`。 它统计的是 replacement strategy allocation 请求。
strategy ring 的直接 reuse 不计入。 它也不是单 query 的 miss counter。
误区四： `BGWRITER_HIBERNATE 表示没有 dirty buffers。`
不对。 它只表示 bgwriter 最近看到 allocation 需求很低，并认为可以长睡。
buffer pool 中仍可能有 dirty pages。 这些 pages 可能不是当前 replacement 近期需要的对象。
也可能只是下一轮才会被扫到。 误区五：
`maxwritten_clean 上升说明数据库不正确。` 不对。
它说明 bgwriter 每轮写页 hit 了配置上限。 这是性能和资源压力信号。
foreground fallback 和 checkpointer 仍然维持 correctness。 误区六：
`把 bgwriter_lru_multiplier 调大总能优化。` 不一定。
它会更积极提前写。 这可能降低前台 dirty victim 写。
也可能增加无效写、WAL flush 干扰和 storage queue pressure。 误区七：
`writeback 等于 fsync。` 不对。
`bgwriter_flush_after` 控制的是 pending writeback hint。 `smgrwriteback()` 不是 checkpoint fsync。
durability 不能从 writeback count 推出。 误区八：
`clean buffer 一定 reusable。` 不对。
clean 只是没有 `BM_DIRTY`。 如果 refcount 非零，replacement 仍不能复用。
如果 usage_count 非零，clock sweep 也会先给 second chance。
## 12. 课堂实验
### 实验 1：观察 bgwriter counters 和 I/O 归因
目标： 看见 bgwriter 写 clean buffers，同时区分 bgwriter、client backend 和 checkpointer 的写页统计。
建议使用临时测试集群。 如果要调 `shared_buffers`，需要重启。
准备：
```sql
SELECT pg_stat_reset_shared('bgwriter');
SELECT pg_stat_reset_shared('io');
SELECT pg_stat_reset_shared('checkpointer');
CREATE TABLE bgw_t AS
SELECT g AS id, repeat('x', 200) AS payload
FROM generate_series(1, 2000000) AS g;
CHECKPOINT;
UPDATE bgw_t
SET payload = repeat('y', 200)
WHERE id % 3 = 0;
```
等待几秒：
```sql
SELECT pg_sleep(5);
```
观察：
```sql
SELECT *
FROM pg_stat_bgwriter;
SELECT backend_type,
       object,
       context,
       writes,
       write_time,
       writebacks,
       writeback_time
FROM pg_stat_io
WHERE backend_type IN ('background writer', 'client backend', 'checkpointer')
  AND object = 'relation'
ORDER BY backend_type, context;
```
预期解释： 如果 bgwriter 有机会运行，`buffers_clean` 可能上升。
`pg_stat_io` 中 `background writer` 的 relation writes 可能上升。 如果测试同时触发 foreground miss，`client backend` relation writes 也可能上升。
这不是互斥关系。 它说明 bgwriter smoothing 和 foreground fallback 可以同时发生。
回源码：
```text
BgBufferSync()
  -> SyncOneBuffer(..., true, ...)
  -> FlushUnlockedBuffer()
  -> FlushBuffer()
```
### 实验 2：故意制造 maxpages cap 压力
目标： 理解 `maxwritten_clean` 是每轮上限信号，不是 correctness failure。
准备一个测试实例，设置较小每轮上限：
```sql
ALTER SYSTEM SET bgwriter_lru_maxpages = 1;
ALTER SYSTEM SET bgwriter_delay = '200ms';
SELECT pg_reload_conf();
SELECT pg_stat_reset_shared('bgwriter');
SELECT pg_stat_reset_shared('io');
```
运行写入和替换压力。 可以用 `pgbench`，也可以用两个大表交替更新和扫描。
观察：
```sql
SELECT buffers_clean, maxwritten_clean, buffers_alloc
FROM pg_stat_bgwriter;
SELECT backend_type,
       writes,
       write_time
FROM pg_stat_io
WHERE backend_type IN ('background writer', 'client backend')
  AND object = 'relation'
ORDER BY backend_type;
```
预期解释： `maxwritten_clean` 增长表示 bgwriter 经常一轮只写 1 个 page 就停。
如果 `client backend` writes 也增长，说明前台正在吸收 dirty victim fallback。 然后把 `bgwriter_lru_maxpages` 调回默认或更高值，重复观察。
注意： 这不是证明“更高一定更好”。
它只是帮助你看到 supply cap 对 foreground fallback 的影响。
### 实验 3：源码断点观察控制回路
目标： 看到 `BgBufferSync()` 的估算变量不是 SQL 可见指标。
在 debug build 上 attach bgwriter pid。 SQL 找 pid：
```sql
SELECT pid
FROM pg_stat_activity
WHERE backend_type = 'background writer';
```
gdb 断点建议：
```text
break BgBufferSync
break StrategySyncStart
break SyncOneBuffer
commands
  silent
  continue
end
```
更细观察时，在 `BgBufferSync()` 内打印：
```text
recent_alloc
strategy_delta
smoothed_alloc
smoothed_density
reusable_buffers_est
upcoming_alloc_est
num_written
```
在 `SyncOneBuffer()` 内观察：
```text
buf_id
buf_state
BUF_STATE_GET_REFCOUNT(buf_state)
BUF_STATE_GET_USAGECOUNT(buf_state)
buf_state & BM_DIRTY
buf_state & BM_VALID
```
预期解释： 你会看到 bgwriter 大量返回“不写”的 buffer。
这不是浪费。 如果 buffer pinned、usage_count 非零、无效或已 clean，bgwriter 按设计跳过。
它不是全池 dirty page writer。 它是 replacement-frontier cleaner。
## 13. 讨论题
1. 为什么 bgwriter 使用 `numBufferAllocs` 和 clock-sweep hand 做估算，而不是每轮统计全池 dirty page 数？
2. 一个 buffer 满足 `BM_DIRTY`，但 `usage_count > 0`，为什么 bgwriter 默认不写它？
3. `SyncOneBuffer()` 同时被 bgwriter 和 checkpointer 调用，为什么不能因此说两者承担同一层 correctness？
4. 如果 `pg_stat_bgwriter.maxwritten_clean` 持续增长，但 `pg_stat_io` 中 client backend writes 很低，你会怎么解释？
5. 为什么 `BGWRITER_HIBERNATE` 不能证明系统里没有 dirty page？
6. `bgwriter_flush_after` 触发的 writeback 和 checkpoint fsync 有什么根本区别？
7. 如果 bgwriter 写 page 时发生 I/O ERROR，哪些 shared buffer 状态必须被清理，为什么不能只记录日志后继续？
8. 为什么 foreground backend 写 dirty victim 是必要 fallback，而不是应该完全避免的异常？
## 14. 本节小结
本节只解决一个问题： `bgwriter 如何平滑 reusable clean buffer 的供给。`
核心链路是：
```text
foreground BufferAlloc() demand
  -> StrategyControl.numBufferAllocs / clock hand
  -> BgBufferSync() feedback estimate
  -> SyncOneBuffer(... skip_recently_used = true ...)
  -> FlushBuffer() writes dirty reusable pages
  -> foreground misses see fewer dirty victims
```
核心状态是：
```text
refcount
usage_count
BM_DIRTY
BM_VALID
BM_CHECKPOINT_NEEDED
StrategyControl.numBufferAllocs
nextVictimBuffer / completePasses
PendingBgWriterStats
WritebackContext
```
这些状态分布在 shared memory、bgwriter backend-local static state 和 cumulative stats 中。 不能用任意单个字段解释语义。
生命周期上，`StrategyControl` 是 shared memory。 bgwriter 是 postmaster 管理的 auxiliary process。
每次写 buffer 都要通过 pin、content lock、buffer I/O ownership 和 ResourceOwner cleanup 收尾。 ERROR 后，顶层 handler 释放 LWLocks、condition variable sleep、buffer locks、buffer pins、buffer I/O、SMgr 和 memory context。
正确性层次上，bgwriter 写页仍遵守 WAL-before-data。 但 checkpoint correctness 不由它负责。
checkpointer 标记和完成 checkpoint 集合。 writeback hint 不等于 fsync。
foreground dirty victim write 是必要 fallback。 错误和退化路径上，`bgwriter_lru_maxpages = 0` 会禁用 cleaning scan。
`maxwritten_clean` 表示每轮 cap 压力。 hibernate missed wakeup 只影响优化时机。
I/O ERROR 通过 ResourceOwner 和 `AbortBufferIO()` 清理 `BM_IO_IN_PROGRESS`，避免 shared buffer 卡死。 观测上，`pg_stat_bgwriter` 能看到累计 counters。
`pg_stat_io` 能按 backend type 看 bgwriter、client backend 和 checkpointer 的 I/O 归因。 `pg_stat_activity` 能看 bgwriter 当前 wait event。
但 clean reusable buffer 的当前数量、smoothed density 和 next_to_clean 都不可直接 SQL 观测。 需要通过 stats 组合、源码断点或 profiling 近似推断。
可迁移的系统规律是：
```text
当后台 worker 只能看到滞后的全局需求信号，而前台路径必须保持 correctness 时，
后台 worker 应该承担 smoothing 和 amortization；
前台路径仍保留完整 fallback；
checkpoint / durability 等全局正确性边界不能隐式转交给这个 smoothing worker。
```
这个规律不仅适用于 bgwriter。 也适用于 WAL writer、autovacuum、prefetch、writeback hint 和许多后台维护任务。
判断这类机制时，要先问：
```text
它是在保证 correctness，还是在降低 foreground tail latency？
如果它慢了，谁兜底？
如果它错过一次 wakeup，系统是否仍正确？
哪些指标只是累计症状，哪些状态才是不可变边界？
```
对 bgwriter，本节答案很明确： `它只平滑 clean reusable buffer，不承担 checkpoint correctness。`
