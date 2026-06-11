# PostgreSQL Checkpoint 调优、写放大与延迟尖刺

## 课程定位

前置知识：已经理解 buffer dirty/page LSN、WAL-before-data、full-page image、checkpoint redo pointer、fsync request queue 和 checkpoint lifecycle。

本节唯一主问题：
为什么同一套业务写入，在某些 checkpoint 参数组合下会表现为周期性写入突刺、WAL/FPI 量上升、WAL 目录膨胀和 crash recovery 时间变长？
本节围绕的核心矛盾：
PostgreSQL 希望把脏页尽早、平滑地写到数据文件，降低 crash recovery 需要 replay 的 WAL；但过于频繁的 checkpoint 会制造更多 full-page image 和同步写压力，过于稀疏的 checkpoint 又会扩大 WAL 保留、恢复距离和单轮刷盘规模。
学完本节后，你应该能独立判断：
- 一个 checkpoint 尖刺主要来自 WAL 触发、时间触发、手动 fast checkpoint，还是 sync phase。
- `max_wal_size` 改变的是 checkpoint 触发距离，不是 WAL 目录的绝对硬上限。
- `checkpoint_timeout` 只有在 WAL 触发没有先到时才决定周期。
- `checkpoint_completion_target` 改变写入节奏，也会反向改变 WAL 触发阈值。
- checkpoint 越少通常减少 FPI 频率，但会增加恢复距离和 WAL 保留窗口。
- checkpoint 越多通常缩短恢复距离，但可能提高 WAL/FPI 量并制造更频繁的 I/O。
- `pg_stat_checkpointer`、`pg_stat_wal`、`pg_stat_io` 和 checkpoint log 分别能证明什么，不能证明什么。
本节不是 DBA 参数调优清单。
它是一节 runtime 诊断课。
我们从能看到的现象开始，再回到源码解释参数为什么会改变这个现象。

源码基线：本课使用当前实际源码路径 `/home/highgo/postgres`，branch `master`，commit `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；核心源码分工见第 3 节。

## 1. 本节在总主线中的位置

先看一个现场。
业务写入量没有突增。
但磁盘写吞吐每隔几分钟冲上去。
同时，延迟 P99 出现尖刺。
server log 里能看到类似信息：

```text
checkpoint starting: time
checkpoint complete: wrote ... buffers (...%); wrote ... SLRU buffers;
... write=... s, sync=... s, total=... s; sync files=...;
longest=... s, average=... s; distance=... kB, estimate=... kB;
lsn=..., redo lsn=...
```

或者：

```text
checkpoint starting: wal
checkpoints are occurring too frequently
```

这两个现象很不一样。
`checkpoint starting: time` 表示时间触发到达。
`checkpoint starting: wal` 表示 WAL 消耗触发到达。
如果日志里出现 too frequently 的 hint，它来自 `checkpointer.c:476-484`。
源码只在非 restartpoint、带 `CHECKPOINT_CAUSE_XLOG`、且距离上次 checkpoint 开始少于 `checkpoint_warning` 时打印这个 warning。
所以这个 warning 的含义不是“磁盘慢”。
它首先说明 WAL 消耗太快，以至于 WAL 触发比期望更频繁。
第二个现场是 WAL 目录膨胀。
`pg_wal` 目录里 segment 数量超过了你对 `max_wal_size` 的直觉。
这并不自动说明 bug。
`max_wal_size` 是 checkpoint 触发和保留计算中的目标量，不是一个每秒强制回收的硬 cap。
旧 WAL segment 要等 checkpoint 完成后才有机会移除或回收。
复制槽、`wal_keep_size`、归档、restartpoint、崩溃恢复中的 invalid page reference 都可能把保留窗口继续拉长。
第三个现场是 crash restart 时间变长。
实例被 kill 后启动，日志里能看到：

```text
redo starts at ...
redo in progress, elapsed time: ...
redo done at ... system usage: ...
```

这些日志来自 `xlogrecovery.c:1699-1701`、`xlogrecovery.c:1712-1714` 和 `xlogrecovery.c:1848-1851`。
恢复时间主要取决于从 checkpoint redo LSN 到可用 WAL 末端之间有多少 WAL record 需要读取、校验和 replay。
`max_wal_size` 和 `checkpoint_timeout` 变大，通常会拉长这个潜在距离。
第四个现场是 WAL 量和 FPI 量变化。
`pg_stat_wal` 里 `wal_bytes`、`wal_fpi`、`wal_fpi_bytes` 突然比调参前更高。
这常常发生在 checkpoint 变得更频繁之后。
原因不是 checkpoint record 本身大。
真正的放大来自 checkpoint 推进 redo pointer 后，页面在新 checkpoint 周期里的首次 WAL 修改更容易带 full-page image。
`xloginsert.c:678-695` 的核心判断是：

```c
needs_backup = (page_lsn <= RedoRecPtr);
```

这行判断把 checkpoint 周期和 WAL/FPI 量直接连起来。

## 2. 核心矛盾与一句话运行模型

本节的运行模型可以压缩成一句话：
checkpoint 调优是在三个窗口之间调平衡：脏页写出窗口、WAL 保留窗口、FPI 重新开始窗口。
脏页写出窗口由本轮 checkpoint 开始时有多少 dirty buffer 决定。
WAL 保留窗口由 redo pointer、当前 WAL 位置、复制槽、`wal_keep_size` 和 checkpoint 完成时机共同决定。
FPI 重新开始窗口由 redo pointer 推进决定。
checkpoint 越频繁，FPI 重新开始越频繁。
checkpoint 越稀疏，单轮 dirty set、WAL 保留和 crash recovery 距离通常越大。
`checkpoint_completion_target` 试图把本轮写 dirty buffer 的动作摊到 checkpoint interval 的某个比例内。
但它不是 I/O 限速器的完整替代品。
它只是 checkpointer 在 `BufferSync()` 每处理一个候选 buffer 后，用进度估算决定要不要睡 100ms。
如果系统已经落后于计划，它不会继续 sleep。
如果手动或 shutdown 请求带 `CHECKPOINT_FAST`，它也不会按 completion target 平滑。

## 3. 核心文件分工与阅读顺序

第一站读 `guc_parameters.dat`。
这里确认参数的真实名字、单位、上下文和默认值。
`checkpoint_completion_target` 在 `guc_parameters.dat:411-418`。
它是 `PGC_SIGHUP`，默认 `0.9`，范围 `0.0` 到 `1.0`，assign hook 是 `assign_checkpoint_completion_target`。
`checkpoint_timeout` 在 `guc_parameters.dat:430-437`。
它是秒，默认 `300`，最小 `30`，最大 `86400`。
`checkpoint_warning` 在 `guc_parameters.dat:439-447`。
它控制 WAL 触发过频时是否打 warning，默认 `30` 秒，`0` 表示关闭 warning。
`max_wal_size` 在 `guc_parameters.dat:2186-2194`。
它是 MB，默认来自 `DEFAULT_MAX_WAL_SEGS * segment_size`，assign hook 是 `assign_max_wal_size`。
`min_wal_size` 在 `guc_parameters.dat:2248-2255`。
它控制 checkpoint 后至少保留或回收到多少 WAL segment，而不是触发 checkpoint 的主阈值。
第二站读 `xlog.c:2188-2232`。
这里能看到 `max_wal_size` 和 `checkpoint_completion_target` 如何一起算出 `CheckPointSegments`。
第三站读 `xlog.c:2291-2310` 和 `xlog.c:2513-2525`。
这里是 WAL segment 写满后检查是否需要请求 checkpoint 的路径。
第四站读 `checkpointer.c:205-612`。
这里是 checkpointer 主循环。
它合并请求、处理时间触发、区分 checkpoint 和 restartpoint、更新统计、调用 `CreateCheckPoint()` 或 `CreateRestartPoint()`。
第五站读 `checkpointer.c:787-939`。
这里是平滑写入的核心：`CheckpointWriteDelay()` 和 `IsCheckpointOnSchedule()`。
第六站读 `xlog.c:7361-7896`。
这里是 `CreateCheckPoint()`。
它定义了在线 checkpoint 的安全顺序：redo 标记、刷脏页、sync、completion record、`pg_control`、旧 WAL 清理。
第七站读 `bufmgr.c:3551-3826`。
这里是 `BufferSync()`。
它决定本轮要写哪些 dirty buffer、如何排序、如何按 tablespace 平衡、如何调用 `CheckpointWriteDelay()`。
第八站读 `sync.c:284-476`。
这里是 checkpoint sync phase。
这段经常解释日志里 `sync=...`、`sync files=...`、`longest=...` 的尖刺。
第九站读 `xloginsert.c:512-535`、`xloginsert.c:678-695` 和 `xlog.c:735-955`。
这里解释 FPI 重新开始和 `RedoRecPtr` 更新之后的 WAL 放大。
第十站读 `system_views.sql:1247-1293`。
这里是 `pg_stat_checkpointer`、`pg_stat_io` 和 `pg_stat_wal` 的 SQL 视图定义。

## 4. 三个参数不是三个旋钮

`max_wal_size`、`checkpoint_timeout` 和 `checkpoint_completion_target` 经常被误解为三个彼此独立的旋钮。
源码里不是这样。
`checkpoint_timeout` 决定 checkpointer 主循环中时间触发何时到达。
`checkpointer.c:410-418` 用 `now - last_checkpoint_time >= CheckPointTimeout` 设置 `CHECKPOINT_CAUSE_TIME`。
如果没有外部请求，统计上会把它计为 timed checkpoint。
但 WAL 触发可以先到。
WAL 写路径在 segment 完成时进入 `xlog.c:2498-2525`。
如果 `XLogCheckpointNeeded(openLogSegNo)` 为真，就 `RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)`。
`XLogCheckpointNeeded()` 在 `xlog.c:2301-2310` 中比较当前新 segment 和 `RedoRecPtr` 所在 segment 的距离。
这个距离超过 `CheckPointSegments` 就认为需要 checkpoint。
`CheckPointSegments` 不是直接等于 `max_wal_size / wal_segment_size`。
它在 `xlog.c:2191-2218` 里这样计算：

```text
CheckPointSegments = floor(max_wal_size_segments / (1.0 + checkpoint_completion_target))
```

这个公式非常关键。
源码注释给出的假设是：checkpoint 期间还会继续消耗 WAL，消耗量大约是 checkpoint 间隔 WAL 量乘以 `checkpoint_completion_target`。
所以为了避免 checkpoint 完成时超过 `max_wal_size`，触发点要早于 `max_wal_size`。
默认 `checkpoint_completion_target = 0.9`。
这意味着 WAL 触发距离大约是 `max_wal_size / 1.9`，向下取整到 segment。
所以只把 `checkpoint_completion_target` 从 `0.5` 调到 `0.9`，在 `max_wal_size` 不变时，会让 WAL 触发点更早。
这点和很多人的直觉相反。
较高的 completion target 让写入更平滑。
但它也告诉系统：我打算在 checkpoint 期间花更长时间写脏页，所以需要更早开始下一次 checkpoint。
如果 `max_wal_size` 太小，调高 completion target 可能减少单次写入速率，却增加 WAL-triggered checkpoint 频率。
因此实际调参时，`max_wal_size` 和 `checkpoint_completion_target` 应该成对看。
`checkpoint_timeout` 只是在 WAL 没有先触发时发挥主导作用。
高 WAL 生成 workload 下，真正的周期常常由 `max_wal_size` 和 `checkpoint_completion_target` 推导出的 `CheckPointSegments` 决定。
低 WAL 生成 workload 下，周期才更接近 `checkpoint_timeout`。

## 5. 从请求到开始：checkpointer 主循环

checkpoint 请求可以来自多处。
时间触发来自 checkpointer 自己。
WAL 触发来自 WAL 写路径。
SQL `CHECKPOINT` 来自 `ExecCheckpoint()`。
shutdown 和 end-of-recovery 也会产生特殊 checkpoint。
`checkpointer.c:118-143` 定义 `CheckpointerShmemStruct`。
本节最关键的字段是：
- `ckpt_started`
- `ckpt_done`
- `ckpt_failed`
- `ckpt_flags`
- `start_cv`
- `done_cv`
- fsync request ring buffer
这些状态是 shared memory。
backend 通过 `RequestCheckpoint()` 设置 flag 并唤醒 checkpointer。
等待方用 condition variable 等 `ckpt_started` 和 `ckpt_done` 推进。
`RequestCheckpoint()` 在 `checkpointer.c:1069-1196`。
它先拿 `ckpt_lck`，把新 flags OR 到 `ckpt_flags`。
这里用 OR 很重要。
多个 backend 同时请求 checkpoint 时，系统不能让较弱请求覆盖较强请求。
例如一个 backend 请求 `CHECKPOINT_FAST`，另一个请求普通等待，OR 后 fast 语义不能丢。
源码注释在 `xlog.h:144-148` 明确要求 checkpoint flags 必须适合 OR 合并。
`RequestCheckpoint()` 设置 `CHECKPOINT_REQUESTED` 后，用 latch 唤醒 checkpointer。
如果调用方带 `CHECKPOINT_WAIT`，它先等 `ckpt_started` 变化，再等 `ckpt_done` 追上。
如果 checkpointer 报告失败，等待方收到 `checkpoint request failed`。
checkpointer 主循环在 `checkpointer.c:371-612`。
它每轮先 `AbsorbSyncRequests()`，再处理 signal 和 config reload。
如果 `ckpt_flags` 非零，就准备做 checkpoint。
如果距离 `last_checkpoint_time` 超过 `CheckPointTimeout`，也准备做 checkpoint，并设置 `CHECKPOINT_CAUSE_TIME`。
真正开始时，它拿锁取走 `ckpt_flags`，清零共享 flags，递增 `ckpt_started`，广播 `start_cv`。
然后根据 `RecoveryInProgress()` 决定调用 `CreateCheckPoint()` 还是 `CreateRestartPoint()`。
end-of-recovery checkpoint 例外。
即使还在 recovery 中，它也是一个真实 checkpoint，而不是 restartpoint。
完成后，checkpointer 更新 `ckpt_done` 并广播 `done_cv`。
这就是等待语义的边界。
`CHECKPOINT_WAIT` 等的是 checkpointer 认为本次 checkpoint 完成或失败。
它不是等待“所有未来新 dirty 页都落盘”。

## 6. 在线 checkpoint 的安全顺序

`CreateCheckPoint()` 的大框架在 `xlog.c:7361-7896`。
第 21 节已经详细讲过 checkpoint lifecycle 和 redo pointer。
这里只保留调参相关的顺序。
在线 checkpoint 的顺序是：

```text
SyncPreCheckpoint()
  -> 进入 critical section
  -> 若系统空闲且不是 force，可能 skip
  -> 插入 XLOG_CHECKPOINT_REDO
  -> 更新 RedoRecPtr
  -> 记录 checkpoint start log
  -> 采集 checkpoint payload 需要的事务/OID/MultiXact 等状态
  -> 等待 DELAY_CHKPT_START 相关事务关键区
  -> CheckPointGuts()
       -> SLRU checkpoint
       -> CheckPointBuffers()
       -> ProcessSyncRequests()
       -> CheckPointTwoPhase()
  -> 等待 DELAY_CHKPT_COMPLETE 相关事务关键区
  -> 写 XLOG_CHECKPOINT_ONLINE
  -> XLogFlush(checkpoint record)
  -> 更新 pg_control
  -> SyncPostCheckpoint()
  -> RemoveOldXlogFiles()
  -> PreallocXlogFiles()
  -> LogCheckpointEnd()
```

这条顺序解释了两个 runtime 细节。
第一，checkpoint 的写入动作可能持续很久。
`CreateCheckPoint()` 注释在 `xlog.c:7378-7387` 直接说，在线 checkpoint 在 flush disk buffers 期间允许其他 WAL record 并发写入，函数可能在繁忙系统上运行很多分钟。
第二，旧 WAL 清理发生在 checkpoint completion record flush 并更新 `pg_control` 之后。
`xlog.c:7845-7864` 才调用 `KeepLogSeg()`、slot invalidation 检查和 `RemoveOldXlogFiles()`。
所以 `pg_wal` 在 checkpoint 进行中继续增长是正常现象。
如果 checkpoint 很慢，WAL 触发和 WAL 保留压力会同时变大。

## 7. 脏页集合如何确定

checkpoint 并不是在整个过程中持续追踪所有新 dirty 页。
`BufferSync()` 在 `bufmgr.c:3551-3826`。
它开始时扫描所有 shared buffers。
对符合条件的 dirty buffer 设置 `BM_CHECKPOINT_NEEDED`。
`bufmgr.c:3585-3595` 的注释说明，这样可以只写 checkpoint 开始时已经 dirty 的页。
checkpoint 开始之后新 dirtied 的页没有这个 flag。
它们不属于本轮必须写出的集合。
这解释了一个常见现场。
checkpoint 正在运行时，业务继续写入。
磁盘写入量包括 checkpoint 写、backend replacement 写、bgwriter 写和 WAL 写。
但本轮 checkpoint 的 `wrote N buffers` 只统计 checkpointer 在 `BufferSync()` 里实际写出的那些 buffer。
`bufmgr.c:3820-3823` 还特别说明，这不包括其他 backend 或 bgwriter scan 写出的 buffer。
所以 `checkpoint complete: wrote ... buffers` 低，不代表这段时间系统没有数据文件写入。
反过来，`pg_stat_checkpointer.buffers_written` 高，也不等于所有 data file writes 都来自 checkpoint。
看总 I/O 要结合 `pg_stat_io`。
`BufferSync()` 把待写 buffer 排序。
排序目标是降低随机 I/O，并为 tablespace 间平衡写入做准备。
`bufmgr.c:3643-3650` 说明如果不平衡，一个 tablespace 可能被连续写爆。
之后它构建 per-tablespace progress heap。
主循环在 `bufmgr.c:3740-3807`。
每处理一个候选 buffer，就用 `SyncOneBuffer()` 尝试写出。
然后调用：

```text
CheckpointWriteDelay(flags, num_processed / num_to_scan)
```

注意传入的是“处理进度”，不是“实际写出进度”。
这是为了避免某些 buffer 已经被别人写掉时导致 tablespace 写入节奏失衡。

## 8. Buffer 写出和 WAL-before-data

`SyncOneBuffer()` 在 `bufmgr.c:4124-4198`。
它先检查 buffer 是否 still valid and dirty。
如果 dirty，就 pin buffer，拿 share-exclusive content lock，然后调用 `FlushUnlockedBuffer()`。
`FlushBuffer()` 在 `bufmgr.c:4496-4628`。
它的注释提醒：这里的 write 只是把 buffer 内容交给 kernel，真正落盘要靠后续 fsync。
checkpoint completion 前必须保证 fsync。
`FlushBuffer()` 的关键正确性 gate 在 `bufmgr.c:4553-4571`。
对于 permanent buffer，它先 `XLogFlush(recptr)`。
`recptr` 是 page LSN。
这保证 WAL-before-data。
因此 checkpoint 写脏页可能把前台事务拖进 WAL flush 等待。
如果某些数据页 LSN 领先于 WAL flush 位置，checkpointer 在写这些页前必须推进 WAL flush。
这能解释 checkpoint 期间前台 commit 或 WAL 相关等待变多的现场。
之后 `FlushBuffer()` 调 `smgrwrite()` 写数据文件。
`md.c:1109-1160` 里，写成功后如果不是 temp 且不跳过 fsync，会注册 dirty segment。
这个注册进入 fsync request 机制。
如果无法转发给 checkpointer，`md.c:1528-1555` 会在 backend 本地 fsync。
这条 fallback 很少见，但一旦出现会直接把 fsync 延迟打到前台 backend。

## 9. 写入平滑不是固定 sleep

`CheckpointWriteDelay()` 在 `checkpointer.c:787-859`。
它在每个 checkpoint buffer write 之后被调用。
正常情况下，如果不是 fast checkpoint、没有 shutdown、没有 pending fast request，并且 `IsCheckpointOnSchedule(progress)` 返回 true，checkpointer 会做几件事：
- 处理 config reload。
- 吸收 fsync requests。
- 检查 archive timeout。
- 上报阶段性统计。
- sleep 100ms。
如果已经落后于计划，它不 sleep。
如果队列压力大，它即使不 sleep，也每 `WRITES_PER_ABSORB` 次写操作吸收一次 fsync requests，避免 queue 溢出。
`IsCheckpointOnSchedule()` 在 `checkpointer.c:869-939`。
它先做：

```text
progress *= CheckPointCompletionTarget
```

然后用两个维度比较。
第一个维度是 WAL 进度。
它取当前 WAL insert pointer 或 replay pointer，计算自 checkpoint start 以来消耗了多少 segment，再除以 `CheckPointSegments`。
第二个维度是时间进度。
它用当前时间减 `ckpt_start_time`，再除以 `CheckPointTimeout`。
只要 checkpoint 写入进度落后于 WAL 进度或时间进度，就不再 sleep。
所以 completion target 的真实语义不是“每秒最多写多少 MB”。
它是“在估计的 checkpoint interval 中，计划用多大比例完成这批 checkpoint dirty buffer”。
实际写速率可以被多个因素打破：
- dirty buffer 太多。
- 存储吞吐不足。
- sync phase 慢。
- `CHECKPOINT_FAST` 禁用 delay。
- WAL 消耗太快，WAL 进度领先。
- `CheckPointSegments` 因为 `max_wal_size` 太小而太小。
- shutdown 或 end-of-recovery checkpoint 走特殊路径。
这就是为什么仅调大 `checkpoint_completion_target`，不一定消除延迟尖刺。
如果系统本来就落后，它只是减少 sleep 的机会。

## 10. Sync phase 为什么会尖刺

checkpoint 写 dirty buffer 后，还要处理 fsync requests。
`CheckPointGuts()` 在 `xlog.c:8048-8074`。
它先做 SLRU checkpoint 和 `CheckPointBuffers()`。
然后设置 `ckpt_sync_t`，调用 `ProcessSyncRequests()`，再设置 `ckpt_sync_end_t`。
`ProcessSyncRequests()` 在 `sync.c:284-476`。
它要确保 checkpointer 已经吸收了截至此刻 backend 转发过来的 fsync requests。
`sync.c:311-319` 解释了最紧的 race：某个 backend 可能刚好在 `BufferSync()` 访问某个 buffer 前把它写掉并清 dirty bit。
只要 backend 在清 dirty bit 前已经 queue fsync request，checkpoint 在 `ProcessSyncRequests()` 前再 absorb 一次，就不会漏 fsync。
sync phase 会遍历 pending fsync hash table。
每个 entry 调用对应 handler 的 `sync_syncfiletag`。
普通 relation segment 最终到 `mdsyncfiletag()`。
`md.c:1934-1944` 调用 `FileSync()`，并把 fsync 计入 IO stats。
日志里的 `sync files`、`longest`、`average` 来自 `CheckpointStats.ckpt_sync_rels`、`ckpt_longest_sync` 和 `ckpt_agg_sync_time`。
这些字段在 `sync.c:469-472` 设置。
因此如果 checkpoint log 中 `write` 时间不长，但 `sync` 时间很长，瓶颈不是写 dirty buffer 的循环。
它更可能是 storage cache flush、某些 relation segment fsync、文件系统或底层设备抖动。
`checkpoint_completion_target` 主要影响 write phase 的 pacing。
它不能把最终 fsync 成本完全摊平。
`checkpoint_flush_after` 可以给 kernel 提前 writeback hint。
它定义在 `guc_parameters.dat:420-428`。
在 buffer 写出后，`ScheduleBufferTagForWriteback()` 和 `IssuePendingWritebacks()` 会尝试把 pending writeback 以更顺序的方式发给 OS。
但源码也说这是 hint，`bufmgr.c:7741-7746` 明确说明它用于改善 OS I/O scheduling，尽量不报错。
所以它不是 fsync 延迟的严格上界。

## 11. WAL 触发与 max_wal_size

`max_wal_size` 最容易被误解。
它不是“`pg_wal` 目录永远不能超过这个大小”。
它首先参与 `CheckPointSegments` 计算。
然后 WAL 写路径在 segment 完成时调用 `XLogCheckpointNeeded()`。
`XLogCheckpointNeeded()` 用 `RedoRecPtr` 所在 segment 到新 segment 的距离判断是否超过阈值。
所以触发边界依赖 redo pointer。
redo pointer 只有 checkpoint 或 restartpoint 成功推进后才真正改变恢复起点。
如果 checkpoint 慢，WAL 仍会继续产生。
如果复制槽要求保留更旧 WAL，清理也不能按 `max_wal_size` 直接删除。
`GetWALAvailability()` 在 `xlog.c:8390-8477` 把 slot target LSN 的可用性分成几类。
其中 `WALAVAIL_RESERVED` 表示目标 segment 仍在 `max_wal_size` 范围内。
`WALAVAIL_EXTENDED` 表示因为 slot 或 `wal_keep_size` 保留了超出 `max_wal_size` 的 segment。
`WALAVAIL_UNRESERVED` 表示不再被保留但还没删除。
`WALAVAIL_REMOVED` 表示已经丢失。
这说明 WAL 保留是多因素结果。
`max_wal_size` 只是其中一个边界。
`KeepLogSeg()` 在 `xlog.c:8496` 之后会考虑 `wal_keep_size`、replication slots 和 `max_slot_wal_keep_size`。
checkpoint 结束时 `xlog.c:7849-7864` 从 `RedoRecPtr` 开始计算可删除边界，并在 slot invalidation 后重算。
所以看到 `pg_wal` 大于 `max_wal_size` 时，诊断顺序应该是：
1. 是否 checkpoint 正在运行或频繁失败。
2. 是否 checkpoint 完成后仍有 slot 或 `wal_keep_size` 保留。
3. 是否归档积压或 standby/restartpoint 使清理滞后。
4. `max_wal_size` 是否本来就太小，导致频繁触发但清理追不上增长。

## 12. FPI/WAL 量如何受 checkpoint 影响

FPI 与 checkpoint 的关系来自 redo pointer。
在线 checkpoint 开始后，`CreateCheckPoint()` 插入 `XLOG_CHECKPOINT_REDO`。
`XLogInsertRecord()` 对 `XLOG_CHECKPOINT_REDO` 走 special checkpoint class。
`xlog.c:929-941` 持有所有 WAL insertion locks，保留 WAL 空间，然后把 `RedoRecPtr` 和 `Insert->RedoRecPtr` 更新为这条 record 的 start position。
之后普通 WAL record 插入时，需要重新判断页面是否要带 FPI。
`XLogInsert()` 在 `xloginsert.c:512-535` 中循环：

```text
GetFullPageWriteInfo()
  -> XLogRecordAssemble()
  -> XLogInsertRecord()
  -> 如果 RedoRecPtr 变新导致 FPI 判断失效，返回 InvalidXLogRecPtr 重试
```

`XLogRecordAssemble()` 在 `xloginsert.c:678-695` 中按 page LSN 判断。
如果 `full_page_writes` 或 backup 要求生效，且 page LSN 小于等于 `RedoRecPtr`，就为该 block 生成 full-page image。
这带来一个可观测规律：
同一批 page，如果 checkpoint 更频繁，每个 page 更频繁地进入“本 checkpoint 周期首次修改”状态。
于是 `wal_fpi` 和 `wal_fpi_bytes` 更容易增加。
相反，checkpoint 更稀疏时，同一 page 在一个周期内多次修改，通常只有第一次修改需要 FPI。
这会减少 FPI 频率。
但不要把它简化为“调大 max_wal_size 一定减少 WAL 总量”。
如果稀疏 checkpoint 导致更大的 dirty set、更长 recovery、更多 pending fsync 或更长业务高峰覆盖窗口，总体成本可能迁移到别处。
此外，WAL 总量还受 schema、索引数量、HOT 比例、tuple 宽度、wal_compression、logical decoding、bulk load 方式、hint bit、full_page_writes 和 storage checksum 状态影响。
本节只强调 checkpoint 周期对 FPI 的一阶影响。
`pg_stat_wal` 的视图定义在 `system_views.sql:1285-1293`。
关键字段是：
- `wal_records`
- `wal_fpi`
- `wal_bytes`
- `wal_fpi_bytes`
- `wal_buffers_full`
`xlog.c:1115-1124` 在 WAL record 插入成功后累计这些计数。
这些是实例级累计统计。
它们不是单 query 统计。
单 query 视角可以用 `EXPLAIN (ANALYZE, WAL)`，但它看不到后台 checkpoint 自身的完整因果。

## 13. 恢复时间如何受 checkpoint 影响

crash recovery 从 checkpoint record 指向的 redo LSN 开始。
`xlogrecovery.c:850-875` 会判断 checkpoint record 的 redo 是否有效，以及是否需要 recovery。
如果 `checkPoint.redo < CheckPointLoc`，说明在线 checkpoint 的 redo pointer 早于 checkpoint completion record。
这会强制进入 recovery。
`PerformWalRecovery()` 在 `xlogrecovery.c:1611-1686` 中设置 replay 起点。
如果 `RedoStartLSN < CheckPointLoc`，它会从 redo LSN 读第一条 record，并校验那里必须是 `XLOG_CHECKPOINT_REDO`。
主 redo loop 在 `xlogrecovery.c:1708-1806`。
它持续 `ApplyWalRecord()`，直到读不到更多 WAL 或达到 recovery target。
恢复日志在 `xlogrecovery.c:1699-1701` 打印 redo start，在 `xlogrecovery.c:1848-1851` 打印 redo done。
所以 checkpoint 调参影响 recovery 的路径很直接：

```text
更大的 checkpoint interval
  -> redo pointer 推进较少
  -> crash 时 redo LSN 到 WAL end 的距离可能更长
  -> 需要读取和 replay 的 WAL record 更多
  -> recovery time 可能更长
```

但这仍然不是严格线性。
recovery 时间还取决于 WAL record 类型、FPI 比例、数据文件缓存状态、storage read bandwidth、CPU 校验成本、prefetch、是否 archive recovery、是否有 timeline/backup label，以及是否需要等待恢复目标。
FPI 对 recovery 也不是单向坏事。
FPI 增加 WAL 字节数。
但 replay 某些页面时可以直接 restore image，避免依赖 torn 或旧 page 状态。
真正调参时要同时看 recovery SLO、steady-state P99 和 WAL 存储预算。
不能只用日常延迟指标做决定。

## 14. 成本模型

先给一个近似模型。
设：

```text
D = checkpoint 开始时需要写出的 dirty bytes
T = 实际 checkpoint interval
C = checkpoint_completion_target
W = checkpoint 周期内 WAL generated bytes
M = max_wal_size bytes
S = wal_segment_size bytes
```

如果时间触发主导：

```text
T ≈ checkpoint_timeout
```

如果 WAL 触发主导：

```text
触发距离 ≈ floor((M / S) / (1 + C)) * S
T ≈ 触发距离 / WAL 生成速率
```

write phase 的目标平均写速率可以粗略看成：

```text
D / (T * C)
```

这只是近似。
源码实际使用 `num_processed / num_to_scan` 作为 progress，并同时比较时间进度和 WAL 进度。
如果 `D` 很大，`T` 很短，或者 storage 吞吐不足，checkpointer 会落后并停止 sleep。
于是你看到的就是突刺，而不是平滑。
FPI 放大可以粗略看成：

```text
每个 checkpoint 周期内，被 WAL 修改过的不同 page 数量 * page image size
```

checkpoint 更频繁会增加周期数。
周期内 page 热度越分散，FPI 放大越明显。
热点 page 反复更新时，周期内通常只有首次修改带 FPI，后续修改不带。
所以 FPI 成本对 workload shape 很敏感。
全表随机更新、索引多、page 工作集大、checkpoint 频繁，是 FPI 放大明显的组合。
少量热点 page、高局部性更新，FPI 增量可能相对可控。
WAL 保留成本可以粗略看成：

```text
max(redo 到 current WAL 的距离,
    slot restart_lsn 到 current WAL 的距离,
    wal_keep_size,
    archiving/recovery 需要的保留)
```

旧 segment 的物理删除还要等 checkpoint 完成后的 cleanup 路径。
所以 `pg_wal` 峰值可能大于任何一个单独参数的直觉。

## 15. 参数调优的源码级解释

调大 `max_wal_size` 的主要效果：
- 增大 WAL 触发 checkpoint 的距离。
- 降低 WAL-triggered checkpoint 频率。
- 通常减少 redo pointer 推进频率，从而减少 FPI 周期重启次数。
- 允许更长 checkpoint interval，单轮 dirty set 可能变大。
- 增加 crash recovery 的潜在 replay 距离。
- 增加 `pg_wal` 存储预算和峰值窗口。
调小 `max_wal_size` 的主要效果：
- 更早触发 checkpoint。
- 缩短潜在 recovery 距离。
- 提高 checkpoint 频率。
- 更频繁触发 FPI 周期。
- 如果 storage 不够快，可能造成连续 checkpoint 和写入尖刺。
调大 `checkpoint_timeout` 的主要效果：
- 在 time-triggered workload 中降低 checkpoint 频率。
- 减少空闲或低 WAL 负载系统的 FPI 重启频率。
- 可能增加单轮 dirty set 和恢复距离。
- 在 WAL-triggered workload 中，效果可能很小。
调小 `checkpoint_timeout` 的主要效果：
- 更频繁推进 redo pointer。
- 缩短低 WAL 场景下的 recovery 距离。
- 增加 FPI 周期和 checkpoint 写入次数。
- 如果设置过小，可能即使没有 WAL 压力也制造周期性 I/O。
调大 `checkpoint_completion_target` 的主要效果：
- 在有足够 checkpoint interval 的前提下，把 dirty buffer 写出摊得更长。
- 降低 write phase 的短时写速率。
- 通过 `CalculateCheckpointSegments()` 降低 WAL 触发阈值。
- 让系统更早开始 WAL-triggered checkpoint。
- 留给落后追赶的余量更小。
调小 `checkpoint_completion_target` 的主要效果：
- 更早完成 checkpoint 写 dirty buffer。
- 短时写速率可能更高。
- WAL 触发阈值相对更大。
- checkpoint 完成后到下一次触发之间可能有更长安静期。
- 对低延迟 workload 可能更容易制造写突刺。
经验上，调参时不要单独调 completion target。
如果目标是减少 checkpoint 写入尖刺，通常要同时评估：
- 是否应该增大 `max_wal_size`。
- 是否 checkpoint 已经由 WAL 触发主导。
- dirty set 是否来自 shared_buffers 太大、bgwriter 不足、批量写入或 autovacuum。
- sync phase 是否才是主要尖刺。
- FPI/WAL 量是否因 checkpoint 更频繁而上升。

## 16. 观测入口：先看日志

最直接入口是打开 `log_checkpoints`。
本基线默认值是 `true`，定义在 `guc_parameters.dat:1648-1652`。
checkpoint start log 来自 `xlog.c:7167-7183`。
checkpoint complete log 来自 `xlog.c:7188-7287`。
它会给出：
- checkpoint 类型和 flags。
- 写了多少 shared buffers。
- 写了多少 SLRU buffers。
- WAL files added/removed/recycled。
- write time。
- sync time。
- total time。
- sync files。
- longest sync。
- average sync。
- distance。
- estimate。
- checkpoint record LSN。
- redo LSN。
诊断时先分三类。
第一类是 `checkpoint starting: time`。
这说明时间触发是至少一个 cause。
如果业务高峰刚好跨越 `checkpoint_timeout`，写入尖刺可能与时间周期重合。
第二类是 `checkpoint starting: wal`。
这说明 WAL 消耗触发到了。
如果频繁出现，并且有 too frequently warning，优先考虑 `max_wal_size` 太小、WAL 生成太快、completion target 触发阈值太早，或 checkpoint 太慢。
第三类是 `checkpoint starting: force`、`fast`、`shutdown`、`end-of-recovery`。
这些通常不是普通自动调参能解释的周期。
例如手动 SQL `CHECKPOINT` 在 `ExecCheckpoint()` 中默认 `fast = true`。
`CHECKPOINT (mode=spread)` 才能显式请求 spread。
这个细节在 `checkpointer.c:1005-1049`。
如果运维脚本定期执行默认 `CHECKPOINT`，它可能绕过你对 completion target 的预期。

## 17. 观测入口：pg_stat_checkpointer

`pg_stat_checkpointer` 定义在 `system_views.sql:1247-1259`。
字段包括：
- `num_timed`
- `num_requested`
- `num_done`
- `restartpoints_timed`
- `restartpoints_req`
- `restartpoints_done`
- `write_time`
- `sync_time`
- `buffers_written`
- `slru_written`
- `stats_reset`
这些统计由 `pgstat_checkpointer.c:31-72` 上报。
`write_time` 和 `sync_time` 来自 `LogCheckpointEnd()` 里的累计。
`xlog.c:7200-7208` 计算 write phase 和 sync phase 的毫秒数，并累加到 `PendingCheckpointerStats`。
`num_timed` 和 `num_requested` 的更新在 `checkpointer.c:451-467`。
注意粒度。
`pg_stat_checkpointer` 是实例级累计。
它不是最近一次 checkpoint。
要看一个实验窗口，先 reset：

```sql
SELECT pg_stat_reset_shared('checkpointer');
```

然后跑 workload。
再取差值。
常用查询：

```sql
SELECT
  num_timed,
  num_requested,
  num_done,
  write_time,
  sync_time,
  buffers_written,
  slru_written,
  stats_reset
FROM pg_stat_checkpointer;
```

解释时要谨慎。
`num_requested` 包括 WAL 触发、手动请求、shutdown 路径等外部请求。
它不是“业务手动执行 CHECKPOINT 的次数”。
`buffers_written` 只统计 checkpointer 写出的 buffers。
它不包括 bgwriter 或 backend replacement 写出的所有数据页。
`write_time` 和 `sync_time` 是累计毫秒。
要做平均，至少要除以 `num_done` 的增量，并结合 checkpoint log 验证每次分布。

## 18. 观测入口：pg_stat_wal

`pg_stat_wal` 定义在 `system_views.sql:1285-1293`。
本节重点看：
- `wal_bytes`
- `wal_fpi`
- `wal_fpi_bytes`
- `wal_buffers_full`
实验窗口同样先 reset：

```sql
SELECT pg_stat_reset_shared('wal');
```

然后跑 workload。
查询：

```sql
SELECT
  wal_records,
  wal_fpi,
  wal_bytes,
  wal_fpi_bytes,
  wal_buffers_full
FROM pg_stat_wal;
```

如果调小 `max_wal_size` 或 `checkpoint_timeout` 后，`wal_fpi` 和 `wal_fpi_bytes` 明显上升，说明 FPI 周期重启成本变高。
但不能只用 `wal_fpi` 判断 checkpoint 是否“坏”。
FPI 是 crash safety 的一部分。
问题在于是否为了过短的 checkpoint 周期付出了不必要的 WAL 放大。
还要结合 `wal_bytes` 和业务吞吐。
例如：

```text
wal_fpi_bytes / wal_bytes
```

可以近似看 FPI 在 WAL 中的字节占比。
但这个比例也受 `wal_compression` 和 workload page locality 影响。

## 19. 观测入口：pg_stat_io、wait event 和系统层

`pg_stat_io` 定义在 `system_views.sql:1261-1283`。
checkpoint 写 dirty page 时，`FlushBuffer()` 会调用 `pgstat_count_io_op_time()`，对象是 relation，context 通常是 normal。
relation fsync 由 `mdsyncfiletag()` 计入 `IOOP_FSYNC`。
典型查询：

```sql
SELECT
  backend_type,
  object,
  context,
  writes,
  write_bytes,
  write_time,
  writebacks,
  writeback_time,
  fsyncs,
  fsync_time
FROM pg_stat_io
WHERE backend_type IN ('checkpointer', 'background writer', 'client backend')
ORDER BY backend_type, object, context;
```

`pg_stat_io` 能帮助区分 checkpointer、bgwriter 和 client backend 的 I/O。
它仍然是累计统计。
对于尖刺，还要看时间序列。
等待事件方面，源码里能看到：
- `WAIT_EVENT_CHECKPOINTER_MAIN`
- `WAIT_EVENT_CHECKPOINT_WRITE_DELAY`
- `WAIT_EVENT_CHECKPOINT_START`
- `WAIT_EVENT_CHECKPOINT_DONE`
- `WAIT_EVENT_CHECKPOINT_DELAY_START`
- `WAIT_EVENT_CHECKPOINT_DELAY_COMPLETE`
- `WAIT_EVENT_DATA_FILE_WRITE`
- `WAIT_EVENT_DATA_FILE_SYNC`
- `WAIT_EVENT_REGISTER_SYNC_REQUEST`
这些 wait event 能说明当前正在等什么。
它们不能单独解释根因。
例如看到 `DataFileSync`，可能是 checkpoint sync phase，也可能是 backend fallback fsync。
需要结合 backend type、日志、`pg_stat_io` 和源码路径。
系统层还应看：
- 块设备 write latency。
- fsync latency。
- dirty page ratio。
- I/O scheduler。
- cloud volume burst credit。
- cgroup throttling。
- filesystem mount option。
checkpoint 调优不能修复底层 fsync 持续抖动。
它只能改变 PostgreSQL 把压力送到存储层的节奏和批次。

## 20. 常见误区

误区一：把 `max_wal_size` 当硬上限。
源码里它参与 `CheckPointSegments` 和 WAL 清理策略。
checkpoint 进行中、slot 保留、`wal_keep_size`、归档和 restartpoint 都可能让 `pg_wal` 超过它。
误区二：以为 `checkpoint_timeout` 一定决定 checkpoint 周期。
高 WAL 生成负载下，WAL 触发常常先到。
此时周期主要由 `max_wal_size`、`checkpoint_completion_target` 和 WAL 生成速率决定。
误区三：以为 `checkpoint_completion_target = 1.0` 最平滑。
它确实尽量把写入摊得更长。
但它也让 `CheckPointSegments = max_wal_size_segments / 2` 左右。
在 `max_wal_size` 不变时，WAL 触发更早。
而且如果系统落后，`CheckpointWriteDelay()` 会停止 sleep。
误区四：看到 checkpoint `write` 时间长，就只调 completion target。
如果 `sync` 时间才是尖刺主因，completion target 对最终 fsync 延迟帮助有限。
需要看 checkpoint log 的 `write` 和 `sync` 分解。
误区五：把 FPI 增多理解成“WAL 模块异常”。
checkpoint 变频繁后，page 首次修改更频繁带 FPI，是源码预期。
真正问题是 checkpoint 周期是否过短，或者 workload page locality 是否让 FPI 放大不可接受。
误区六：只看 `pg_stat_checkpointer.buffers_written` 判断数据写压力。
它不包括所有 backend 和 bgwriter 写。
要结合 `pg_stat_io` 和系统层 I/O。
误区七：用手动 `CHECKPOINT` 做平滑 checkpoint 实验，却忘了默认 fast。
本基线 SQL `CHECKPOINT` 默认 `fast = true`。
要实验 spread 行为，应使用 `CHECKPOINT (mode=spread)`。

## 21. 诊断流程

第一步，确认 checkpoint 是 time 触发还是 WAL 触发。
看 checkpoint start log。
如果没有日志，先打开 `log_checkpoints`，或在实验环境打开。
第二步，取一个稳定窗口的累计统计差值。
重置 `pg_stat_checkpointer` 和 `pg_stat_wal`。
跑固定 workload。
取 `pg_stat_checkpointer`、`pg_stat_wal`、`pg_stat_io`。
第三步，拆 write phase 和 sync phase。
如果 `write_time` 增长大，且 checkpoint log 的 `wrote buffers` 很高，说明本轮 dirty set 或 write pacing 是重点。
如果 `sync_time`、`longest` 或 `fsync_time` 高，说明 storage sync 是重点。
第四步，判断 checkpoint 触发是否过频。
如果 `num_requested` 增长快、日志频繁 `wal`、出现 `checkpoint_warning`，优先看 `max_wal_size` 是否太小。
还要记住 completion target 会降低 `CheckPointSegments`。
第五步，观察 FPI 比例。
如果 `wal_fpi_bytes / wal_bytes` 在调小 checkpoint 周期后明显上升，说明 WAL 放大是调参副作用之一。
第六步，确认 WAL 保留边界。
看复制槽、`wal_keep_size`、归档状态和 checkpoint 完成情况。
不要只盯 `max_wal_size`。
第七步，评估 recovery SLO。
如果打算增大 `max_wal_size` 或 `checkpoint_timeout`，必须接受 crash recovery 可能 replay 更多 WAL。
用实验环境 kill/restart 验证，而不是只凭公式。

## 22. 源码 walkthrough：WAL 触发一次 checkpoint

下面走一条最常见的 WAL-triggered 主链路。
业务 backend 生成 WAL。
WAL writer 或 backend 刷 WAL segment。
在 `XLogWrite()` 中，如果写完一个 segment，进入 `finishing_seg` 分支。
`xlog.c:2498-2512` 会 fsync segment、通知 archiver、更新 last segment switch 时间。
然后 `xlog.c:2513-2525` 检查是否需要 checkpoint。
它先用本地 `RedoRecPtr` 快速判断。
如果看起来需要 checkpoint，就调用 `GetRedoRecPtr()` 刷新本地 copy，再判断一次。
仍然需要，就调用：

```text
RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)
```

`RequestCheckpoint()` 把 flags OR 到 `ckpt_flags`。
它唤醒 checkpointer，但不等待，除非调用者带 `CHECKPOINT_WAIT`。
checkpointer 主循环醒来。
它看到 `ckpt_flags` 非零，设置 `do_checkpoint = true`。
开始 checkpoint 前，它拿锁取走 flags，清空 `ckpt_flags`，递增 `ckpt_started`。
由于 flags 带 `CHECKPOINT_CAUSE_XLOG`，如果距离上次 checkpoint 开始小于 `checkpoint_warning`，它打印 too frequently warning。
之后设置 `ckpt_start_recptr = GetInsertRecPtr()`。
这个值会被 `IsCheckpointOnSchedule()` 用来估计 checkpoint 期间 WAL 进度。
然后进入 `CreateCheckPoint(flags)`。
`CreateCheckPoint()` 插入 `XLOG_CHECKPOINT_REDO`，推进 `RedoRecPtr`。
这会让后续 WAL 插入端的 FPI 判断进入新周期。
接着 `CheckPointGuts()` 写 SLRU、写 dirty buffers、处理 fsync。
`BufferSync()` 按本轮开始时标记的 dirty buffer 写。
每处理一个候选 buffer，调用 `CheckpointWriteDelay()` 判断是否 sleep。
当所有本轮需要的 buffer 写出并 sync 完，`CreateCheckPoint()` 写 `XLOG_CHECKPOINT_ONLINE`，flush 它，更新 `pg_control`。
最后才清理旧 WAL。
这条链路有三个延迟尖刺位置：
- 写 dirty buffers 时的 data file write。
- 写 dirty buffer 前被迫 `XLogFlush(page_lsn)`。
- sync phase 的 data file fsync。
这条链路也有两个放大位置：
- redo pointer 推进导致后续 FPI 周期重启。
- checkpoint 完成前 WAL 继续生成，导致保留和触发压力增加。

## 23. 源码 walkthrough：time-triggered checkpoint

time-triggered checkpoint 不经过 WAL segment 完成时的请求路径。
checkpointer 主循环自己计算时间。
`checkpointer.c:410-418`：

```text
elapsed_secs = now - last_checkpoint_time
if elapsed_secs >= CheckPointTimeout:
    do_checkpoint = true
    flags |= CHECKPOINT_CAUSE_TIME
```

如果同时有外部 request，flags 会合并。
如果没有外部 request，统计上增加 `num_timed`。
checkpoint 完成后，`checkpointer.c:523-533` 把 `last_checkpoint_time` 更新为 checkpoint start time，而不是 end time。
源码注释说这样 time-driven checkpoints 会有可预测间隔。
这也意味着一个很慢的 checkpoint 结束后，下一次 time trigger 可能离结束时间并不远。
如果 checkpoint duration 接近或超过 `checkpoint_timeout`，系统可能进入很不健康的状态。
表现是 checkpoint 几乎连续运行。
这种情况下调大 `checkpoint_completion_target` 往往无效。
因为系统已经没有足够时间窗口可以平滑。
正确方向通常是：
- 增大 checkpoint interval 或 `max_wal_size`。
- 降低 dirty rate。
- 提高存储写入和 fsync 能力。
- 减少不必要的手动 fast checkpoint。
- 分析 bgwriter、backend writes 和 bulk workload。

## 24. 生命周期、ownership 与 cleanup

checkpoint 请求状态属于 shared memory。
`CheckpointerShmemStruct` 由 checkpointer shmem init 创建。
backend 只通过 `RequestCheckpoint()` 设置 flags 和等待 condition variable。
checkpointer 是推进 `ckpt_started`、`ckpt_done`、`ckpt_failed` 的 owner。
fsync request queue 也在 checkpointer shared memory 中。
backend 写 relation segment 后，通过 `RegisterSyncRequest()` 尝试把 fsync request 转发给 checkpointer。
checkpointer 调 `AbsorbSyncRequests()` 把 shared queue 里的请求搬到自己的 pendingOps hash table。
`ProcessSyncRequests()` 在 checkpoint sync phase 处理这些 pending ops。
如果 queue 满，`ForwardSyncRequest()` 会尝试 compact duplicate。
`checkpointer.c:1274-1284` 说明 queue 满会导致严重性能问题，因为 fallback 是 backend 自己 fsync。
checkpoint 失败时，checkpointer 的异常恢复在 `checkpointer.c:285-345`。
它释放 LWLocks，取消 condition variable sleep，结束 wait reporting，清理 AIO error，解锁 buffers，释放 auxiliary process resources，执行 buffer/smgr/file/hash cleanup。
如果当时 `ckpt_active`，它会递增 `ckpt_failed`，把 `ckpt_done` 追到 `ckpt_started`，广播 `done_cv`。
等待的 backend 会收到错误。
这说明 checkpoint 失败不是静默忽略。
但失败前已经发生的一些 WAL 或 control file 更新不一定能“事务式回滚”。
`xlog.c:7552-7558` 注释说明，如果 checkpoint 未完成但 `RedoRecPtr` 已经推进，后果是后续 `XLogInsert` 可能写一些本来不必要的 FPI。
这是安全但更贵的 fallback。

## 25. 正确性机制层次

checkpoint 调优不能破坏 crash safety。
这里有多层机制共同约束。
第一层是 WAL-before-data。
`FlushBuffer()` 写 permanent buffer 前必须 `XLogFlush(page_lsn)`。
它保证数据页变化不会先于描述它的 WAL durable。
第二层是 redo pointer 原子发布。
在线 checkpoint 先写 redo 标记，最后写 completion checkpoint record 并更新 `pg_control`。
crash recovery 只相信已经发布到 `pg_control` 的 checkpoint record。
第三层是 dirty set 边界。
`BM_CHECKPOINT_NEEDED` 只标记 checkpoint 开始时已 dirty 的 buffer。
checkpoint 开始后新 dirty 页不强行纳入本轮。
这是在线 checkpoint 能和业务并发的关键。
第四层是 fsync request 边界。
写数据文件只是进入 kernel。
checkpoint completion 前必须处理 pending fsync requests。
`ProcessSyncRequests()` 用 cycle counter 区分本轮应处理和下一轮处理的请求。
第五层是 WAL insertion lock 对 `RedoRecPtr` 和 `fullPageWrites` 的保护。
`xlog.c:846-847` 说明持有 insertion lock 时，这两个值不会在插入过程中改变。
如果插入端发现自己的 FPI 判断过期，会返回 `InvalidXLogRecPtr` 让上层重新 assemble WAL record。
第六层是 replication slot 和 WAL 保留边界。
checkpoint 可以推进 redo pointer，但不能随意删除 slot 仍需要的 WAL。
如果配置 `max_slot_wal_keep_size`，checkpoint 清理时可能 invalidate 超限 slots。
这些机制分别保证不同事情。
它们不能互相替代。
调参只能改变节奏、窗口和成本。
不能绕过这些正确性边界。

## 26. 异常路径与 fallback

第一个异常路径是 checkpoint 写入或 fsync 失败。
`CreateCheckPoint()` 在写 dirty buffer 前退出 critical section。
`xlog.c:7655-7663` 注释说明，I/O 可能失败，checkpoint 可以失败，但没有理由强制系统 panic。
失败会回到 checkpointer 顶层异常恢复。
等待方通过 `ckpt_failed` 看到失败。
第二个异常路径是 fsync request queue 满。
`ForwardSyncRequest()` 在 `checkpointer.c:1218-1272`。
如果 queue 满，会先 compact duplicate。
如果仍然失败，返回 false。
`register_dirty_segment()` 在 `md.c:1528-1555` 中 fallback 到 backend 本地 fsync。
这个 fallback 保 correctness，但会把延迟打到前台。
第三个异常路径是 checkpoint 期间被新的 fast request 打断节奏。
`FastCheckpointRequested()` 检查共享 flags 中是否有 pending `CHECKPOINT_FAST`。
`CheckpointWriteDelay()` 如果看到 pending fast request，就不再按原计划 sleep。
所以一个手动 fast checkpoint 请求可能影响正在进行的 checkpoint pacing。
第四个异常路径是 restartpoint 做不成。
standby recovery 期间 `CreateRestartPoint()` 只能基于已经 replay 到的 checkpoint record。
`RecoveryRestartPoint()` 在 `xlog.c:8087-8114` 中，如果有 unresolved invalid page references，就不记录 restartpoint。
restartpoint 做不成时，checkpointer 会把下一次尝试安排在约 15 秒后。
这会影响 recovery 中的 WAL 保留和 replay 距离。
第五个异常路径是 shutdown checkpoint。
shutdown checkpoint 带 `CHECKPOINT_FAST`。
并且 shutdown 时不允许并发 WAL 插入。
`xlog.c:7770-7776` 如果发现 shutdown checkpoint 期间仍有并发 WAL activity，会 PANIC。
这不是调参问题，而是 correctness 断言。

## 27. 课堂实验一：观察 WAL 触发和 time 触发

目标：确认 `checkpoint_timeout` 和 `max_wal_size` 谁主导 checkpoint 周期。
环境要求：只在实验实例做，不要在生产直接执行。
准备：

```sql
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET checkpoint_timeout = '5min';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET max_wal_size = '128MB';
SELECT pg_reload_conf();
```

重置统计：

```sql
SELECT pg_stat_reset_shared('checkpointer');
SELECT pg_stat_reset_shared('wal');
```

运行写入 workload，例如 `pgbench`：

```bash
pgbench -i -s 50 postgres
pgbench -c 16 -j 4 -T 300 -N postgres
```

观察：

```sql
SELECT * FROM pg_stat_checkpointer;
SELECT wal_records, wal_fpi, wal_bytes, wal_fpi_bytes FROM pg_stat_wal;
```

再把 `max_wal_size` 增大：

```sql
ALTER SYSTEM SET max_wal_size = '2GB';
SELECT pg_reload_conf();
SELECT pg_stat_reset_shared('checkpointer');
SELECT pg_stat_reset_shared('wal');
```

重复同样 workload。
预期观察：
- 小 `max_wal_size` 更容易看到 `checkpoint starting: wal`。
- `num_requested` 增长可能更快。
- `wal_fpi` 和 `wal_fpi_bytes` 可能更高。
- 增大 `max_wal_size` 后，checkpoint 周期可能更接近 `checkpoint_timeout`。
回到源码解释：
- `assign_max_wal_size()` 触发 `CalculateCheckpointSegments()`。
- `XLogCheckpointNeeded()` 用 `RedoRecPtr` 到新 segment 的距离判断。
- `CheckpointerMain()` 把 WAL request 合并成 checkpoint。

## 28. 课堂实验二：观察 FPI 周期重启

目标：看到 checkpoint 后 page 首次修改带来的 FPI 增加。
准备一张足够大的表：

```sql
DROP TABLE IF EXISTS ckpt_fpi_demo;
CREATE TABLE ckpt_fpi_demo (id int PRIMARY KEY, payload text);
INSERT INTO ckpt_fpi_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 500000) AS g;
VACUUM ckpt_fpi_demo;
```

重置 WAL 统计并强制一次 checkpoint：

```sql
SELECT pg_stat_reset_shared('wal');
CHECKPOINT;
```

执行一次分散更新：

```sql
UPDATE ckpt_fpi_demo
SET payload = repeat('a', 200)
WHERE id % 10 = 0;
```

查询 WAL：

```sql
SELECT wal_records, wal_fpi, wal_bytes, wal_fpi_bytes
FROM pg_stat_wal;
```

不做 checkpoint，再执行一次更新同一批 page：

```sql
UPDATE ckpt_fpi_demo
SET payload = repeat('b', 200)
WHERE id % 10 = 0;
```

再次查询 WAL。
预期观察：
- checkpoint 后第一次分散更新更容易产生 FPI。
- 同一 checkpoint 周期内再次更新，FPI 增速通常下降。
注意：
`CHECKPOINT` 默认 fast。
这个实验的目的不是观察平滑写，而是人为制造一个新的 redo pointer，让 FPI 周期重启。
回到源码解释：
- checkpoint 推进 `RedoRecPtr`。
- `xloginsert.c:692-695` 用 page LSN 和 `RedoRecPtr` 判断 FPI。
- `xlog.c:1118-1121` 累计 `wal_fpi` 和 `wal_fpi_bytes`。

## 29. 课堂实验三：观察 write phase 和 sync phase

目标：区分写 dirty buffer 慢和 fsync 慢。
配置：

```sql
ALTER SYSTEM SET log_checkpoints = on;
ALTER SYSTEM SET track_io_timing = on;
ALTER SYSTEM SET checkpoint_timeout = '2min';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
SELECT pg_reload_conf();
```

重置：

```sql
SELECT pg_stat_reset_shared('checkpointer');
```

跑写入 workload。
在 workload 期间每 5 秒采样：

```sql
SELECT now(), *
FROM pg_stat_checkpointer;
```

同时采样 I/O：

```sql
SELECT now(), backend_type, object, context,
       writes, write_time, fsyncs, fsync_time
FROM pg_stat_io
WHERE backend_type IN ('checkpointer', 'client backend', 'background writer');
```

观察 checkpoint log：

```text
write=... s, sync=... s, total=... s; sync files=...; longest=... s
```

判断：
- `write` 高，`wrote buffers` 高：dirty set 和 write pacing 是重点。
- `sync` 高，`longest` 高：fsync latency 是重点。
- `client backend` fsync 增多：可能触发了 fsync request fallback 或前台写压力。
回到源码解释：
- `BufferSync()` 负责 write phase。
- `CheckpointWriteDelay()` 只调节 write phase 的 sleep。
- `ProcessSyncRequests()` 负责 sync phase。
- `mdsyncfiletag()` 执行 relation fsync。

## 30. 课堂实验四：恢复时间窗口

目标：验证更大 checkpoint 间隔对 crash recovery 距离的影响。
只在可丢弃实验环境执行。
准备两组配置。
配置 A：

```sql
ALTER SYSTEM SET checkpoint_timeout = '1min';
ALTER SYSTEM SET max_wal_size = '128MB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
SELECT pg_reload_conf();
```

配置 B：

```sql
ALTER SYSTEM SET checkpoint_timeout = '15min';
ALTER SYSTEM SET max_wal_size = '4GB';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
SELECT pg_reload_conf();
```

每组都执行相同写入 workload。
在 workload 中途用外部方式 kill postmaster 或主进程，模拟 crash。
重启实例，记录日志里的：

```text
redo starts at ...
redo in progress ...
redo done at ... system usage: ...
```

比较：
- redo start LSN 到 redo done LSN 的距离。
- redo elapsed time。
- `pg_wal` 峰值。
- steady-state checkpoint 尖刺。
预期：
配置 B 往往减少 checkpoint 频率和 FPI 周期重启，但 crash recovery 可能 replay 更多 WAL。
这个实验用来训练权衡。
它不是证明某个配置永远更好。

## 31. 源码练习

练习一：跟 `CheckPointSegments`。
在 `xlog.c:2191` 给 `CalculateCheckpointSegments()` 加临时 DEBUG 日志。
打印 `max_wal_size_mb`、`CheckPointCompletionTarget`、`wal_segment_size` 和 `CheckPointSegments`。
修改 `max_wal_size` 和 `checkpoint_completion_target` 后 reload。
观察这个值如何变化。
练习二：跟 `CheckpointWriteDelay()`。
在 `checkpointer.c:800` 设置断点或临时日志。
打印 `progress`、`ckpt_cached_elapsed`、是否 sleep。
用小 `max_wal_size` 和大写入 workload 观察 WAL 进度如何让 checkpointer 停止 sleep。
练习三：跟 FPI 判断。
在 `xloginsert.c:692-695` 观察某个 relation page 的 `page_lsn` 和 `RedoRecPtr`。
在 checkpoint 前后执行同一类 UPDATE。
画出 page LSN 何时从小于等于 redo pointer 变成大于 redo pointer。
练习四：跟 sync request。
在 `sync.c:320` 和 `sync.c:416` 设置断点。
观察 checkpoint sync phase 前如何吸收请求，以及每个 filetag 如何进入 fsync handler。
重点不是背函数。
重点是确认 runtime 指标背后的状态边界。

## 32. 讨论题

1. 为什么 `checkpoint_completion_target` 越大，`CheckPointSegments` 反而越小？
2. 如果日志里 checkpoint 频繁显示 `wal`，但 `checkpoint_timeout` 很大，下一步应该看哪些参数和指标？
3. 为什么 `max_wal_size` 不能保证 `pg_wal` 目录永远低于该值？
4. 为什么 checkpoint 更频繁可能降低 recovery time，却增加 `wal_fpi_bytes`？
5. `pg_stat_checkpointer.buffers_written` 低，但磁盘写吞吐很高，可能有哪些解释？
6. checkpoint log 中 `write=1s, sync=20s` 时，调大 `checkpoint_completion_target` 为什么可能帮助有限？
7. 手动执行默认 `CHECKPOINT` 为什么可能制造与自动 checkpoint 不同的延迟尖刺？
8. 如果增大 `max_wal_size` 后 P99 好了，但 crash recovery 变慢，这是不是 PostgreSQL 行为异常？

## 33. 本节小结

checkpoint 调优的核心不是找到一个“更大”或“更小”的参数。
核心是识别当前系统的主导窗口。
如果 WAL 触发主导，`max_wal_size`、`checkpoint_completion_target` 和 WAL 生成速率决定 checkpoint 周期。
如果时间触发主导，`checkpoint_timeout` 决定周期。
如果 write phase 主导尖刺，重点看 dirty set、write pacing、storage write bandwidth 和是否 fast checkpoint。
如果 sync phase 主导尖刺，重点看 pending fsync、storage flush latency、`checkpoint_flush_after` 的效果边界和 backend fallback fsync。
如果 WAL/FPI 量上升，回到 `RedoRecPtr` 和 page LSN 的判断。
checkpoint 推进 redo pointer 后，页面在新周期里的首次 WAL 修改更容易带 FPI。
如果 WAL 保留超预期，回到 redo pointer、checkpoint completion、replication slots、`wal_keep_size`、`max_slot_wal_keep_size` 和归档状态。
如果 recovery time 超预期，回到 checkpoint record 指向的 redo LSN 到 WAL end 的 replay 距离。
本节的可迁移规律是：
后台刷盘机制的调优，本质上是在“平摊日常成本”和“限制失败恢复成本”之间移动边界。
把一个窗口调宽，通常会把成本从 steady-state 延迟迁移到 WAL 空间、FPI 周期、fsync phase 或 crash recovery。
源码诊断的任务不是寻找无成本参数，而是确认成本被移到了哪里，以及这个移动是否符合你的 workload、硬件和恢复 SLO。
