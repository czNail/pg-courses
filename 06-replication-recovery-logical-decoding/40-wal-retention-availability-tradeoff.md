# PostgreSQL WAL retention availability tradeoff

## 课程定位

前置知识：已经知道 WAL 是 crash recovery、streaming replication、archive recovery、base backup 和 logical decoding 共同依赖的日志流。 也已经读过本目录前面的物理复制握手课程，知道 standby 通过 `START_REPLICATION` 从某个 LSN 和 timeline 开始追 WAL。 本节唯一主问题：
```text
复制链路中断时，slot、wal_keep_size、archive、max_slot_wal_keep_size 和 backup retention 如何共同决定： 保护消费者继续追赶，还是优先保护 primary 磁盘可用性？
```
核心矛盾：
```text
消费者断线越久，primary 越需要保留更多旧 WAL； 但 primary 的 pg_wal 空间不是无限资源，保留过多会让生产库先死。
```
PostgreSQL 没有用一个全局开关回答这个问题。 它把 WAL retention 拆成几种不同语义：
```text
wal_keep_size: 无身份、无确认、只保留最近一段 WAL 的 cluster-wide floor。 replication slot: 有消费者身份，按 restart_lsn 保留所需 WAL。 max_slot_wal_keep_size: 给 slot retention 加硬上限，超过后允许 slot 失效。 archive: 先把完整 WAL segment 交给外部归档，再允许本地删除。 base backup / backup retention: 规定某个备份从哪个 WAL 起点恢复，以及外部需要保存哪些 WAL。
```
学完后应能独立判断：
```text
一个 standby 断了以后，为什么有时能追上，有时必须重做 base backup； 为什么 wal_keep_size 不是 replication slot 的替代品； 为什么 archive 可以救恢复，但也可能让 primary pg_wal 被归档积压撑满； 为什么 max_slot_wal_keep_size 不是提前限速，而是 checkpoint 清理时的失败边界； 为什么 safe_wal_size 是剩余风险估计，不是消费者追赶 SLA； 为什么 base backup retention 主要是 archive retention 问题，不是 slot 自动解决的问题。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。 当前本地源码里，GUC 参数定义在 `src/backend/utils/misc/guc_parameters.dat`。 当前本地源码还出现 `max_repack_replication_slots`，slot 扫描路径会把它和 `max_replication_slots` 一起计入数组上限。 本节主线仍然讨论 PostgreSQL replication slot 的 WAL retention 语义。

## 1. 本节在总主线中的位置

上一节讲的是 standby 如何进入 physical streaming。 那条链路的终点是：
```text
walsender 接受 START_REPLICATION -> WalSndLoop() -> XLogSendPhysical()
```
本节从断线现场开始。 如果 standby 已经连上并持续反馈，primary 可以不断推进对应 slot 的 `restart_lsn`。 如果 standby 断线，primary 还在生成 WAL。 于是系统必须回答一个更硬的问题：
```text
旧 WAL segment 到底还能不能删除？
```
这个问题不是 walsender 单独能回答。 它跨过这些模块：
```text
checkpoint / restartpoint: 定期决定哪些 WAL segment 已经早于恢复需要。 replication slot: 把消费者仍需的最老 LSN 汇总给 xlog。 WAL archive: 决定一个 segment 是否已经安全交给外部归档。 base backup: 记录某个备份需要从哪个 WAL 起点开始恢复。 SQL views: 把 restart_lsn、wal_status、safe_wal_size 暴露给操作者。
```
本节不展开同步复制提交等待。 同步复制关心事务什么时候对客户端返回。 本节关心旧 WAL 什么时候可以从 primary 本地移除。 本节也不展开 logical decoding 输出插件细节。 逻辑 slot 会影响 WAL 和 catalog xmin retention，但本节只讲它们如何进入同一个保留边界。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：
```text
checkpoint/restartpoint 完成后，xlog.c 从 redo 位置出发， 用 replication slots、wal_keep_size、WAL summarization 和 archive 状态 共同计算最老必须保留的 WAL segment； 如果某些 slot 的 restart_lsn 已落到可删除边界之前， InvalidateObsoleteReplicationSlots() 会让这些 slot 失效； 随后 RemoveOldXlogFiles() 只删除或回收已经归档完成的 segment。
```
这句话里有三个重要边界。 第一，WAL retention 主要在 checkpoint / restartpoint 后收敛。 不是每生成一条 WAL 都立刻检查所有旧 segment。 `XLogCheckpointNeeded()` 会在 segment 消耗超过 `CheckPointSegments` 时请求 checkpoint。 `CheckPointSegments` 又来自 `max_wal_size` 和 `checkpoint_completion_target`。 所以 WAL 删除不是连续过程，而是 checkpoint 批处理过程。 第二，slot retention 和 archive retention 是两种语义。 slot 说的是：
```text
这个消费者还没有确认到 restart_lsn 之后，所以 primary 本地最好别删。
```
archive 说的是：
```text
这个完整 segment 是否已经交给外部归档系统。
```
两者都能挡住本地删除，但回答的问题不同。 slot 能告诉你哪个消费者落后。 archive 能支持未来 restore / PITR / 断线后从 archive 补 WAL。 第三，`max_slot_wal_keep_size` 是 primary 磁盘可用性的刹车。 它不会让消费者自动追快。 它也不会提前把 WAL 写少。 它只是在 checkpoint 清理旧 WAL 时说：
```text
slot 最多只能把保留边界往回拉这么远； 再旧的 restart_lsn 允许导致 slot 失效。
```
这个选择牺牲的是消费者可追赶性。 保护的是 primary 不因一个失联消费者无限保留 WAL。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/access/transam/xlog.c` | `CreateCheckPoint()`、`CreateRestartPoint()`、`KeepLogSeg()`、`RemoveOldXlogFiles()` 和 `GetWALAvailability()` 如何计算 WAL 保留和删除边界。 |
| 2 | `src/backend/replication/slot.c` | slot 如何创建、持有、释放、保存、汇总 `restart_lsn`，以及如何因 WAL removed 失效。 |
| 3 | `src/include/replication/slot.h` | `ReplicationSlotPersistentData` 和 `ReplicationSlot` 的字段语义，特别是 `restart_lsn`、`confirmed_flush`、`invalidated`、`last_saved_restart_lsn`。 |
| 4 | `src/backend/replication/walsender.c` | physical / logical walsender 如何 acquire slot、读 WAL、处理 standby feedback，并在 WAL 缺失时报错。 |
| 5 | `src/backend/replication/slotfuncs.c` | `pg_get_replication_slots()` 如何计算 `wal_status` 和 `safe_wal_size`。 |
| 6 | `src/backend/access/transam/xlogarchive.c` | `XLogArchiveNotify()`、`XLogArchiveCheckDone()`、`RestoreArchivedFile()` 如何连接本地 `pg_wal` 与外部 archive。 |
| 7 | `src/backend/postmaster/pgarch.c` | archiver 如何扫描 `.ready` 文件、执行 archive command/library、完成后改成 `.done`。 |
| 8 | `src/backend/backup/basebackup.c` | `perform_base_backup()` 如何启动/停止在线备份，何时包含 WAL，何时等待 archive。 |
| 9 | `src/backend/access/transam/xlogbackup.c` | `build_backup_content()` 如何生成 `backup_label` 和 backup history 内容。 |
| 10 | `src/backend/catalog/system_views.sql` | `pg_replication_slots`、`pg_stat_replication`、`pg_stat_archiver`、`pg_stat_wal` 的 SQL 视图边界。 |
| 11 | `src/backend/utils/misc/guc_parameters.dat` | `wal_keep_size`、`max_slot_wal_keep_size`、`archive_mode`、`archive_command`、`archive_library` 等 GUC 的当前定义。 |
推荐阅读顺序从 `xlog.c` 的清理阶段开始。 不要先从 `pg_replication_slots` 开始。 视图只是把内部状态投影出来。 真正的决策点在 checkpoint 后：
```text
CreateCheckPoint() -> KeepLogSeg() -> InvalidateObsoleteReplicationSlots() -> RemoveOldXlogFiles()
```
standby 上的 restartpoint 是对应路径：
```text
CreateRestartPoint() -> KeepLogSeg() -> InvalidateObsoleteReplicationSlots() -> RemoveOldXlogFiles() -> archive_cleanup_command
```
理解这两条链路后，再读 slot 和 archive 的各自状态。 这样能避免把 `wal_status` 误读成一个独立状态机。 它其实是从 WAL 删除边界、slot `restart_lsn` 和配置推导出来的观测结果。

## 4. 关键状态与边界

### 4.1 `restart_lsn`: slot 的 WAL retention 锚点

`ReplicationSlotPersistentData.restart_lsn` 是本节最重要的字段。 它表示：
```text
这个 slot 可能仍需要的最老 WAL LSN。
```
注意这里是“可能仍需要”。 它不是消费者的 apply LSN。 它不是 walsender 当前已经发送到哪里。 它也不是 archive 中最老可恢复 LSN。 physical slot 的 `restart_lsn` 通常由 standby feedback 推进。 在 `walsender.c` 中，standby reply 进入 `ProcessStandbyReplyMessage()` 后，物理 slot 会调用：
```text
PhysicalConfirmReceivedLocation(flush_lsn) -> slot->data.restart_lsn = flush_lsn -> ReplicationSlotsComputeRequiredLSN()
```
也就是说 physical slot 关注的是 standby 已经 flush 到本地的 WAL 位置。 logical slot 更复杂。 logical slot 既有 `restart_lsn`，又有 `confirmed_flush`。 `confirmed_flush` 表示客户端确认收到的解码输出位置。 `restart_lsn` 表示 logical decoding 仍可能需要重新读取的最老 WAL。 因为 decoding 可能需要跨事务、catalog snapshot 和 reorder buffer 状态，`restart_lsn` 可以早于 `confirmed_flush`。 所以不要用 `confirmed_flush_lsn` 直接判断 WAL 是否能删除。 删除边界看的是 `restart_lsn`。

### 4.2 physical slot 和 logical slot 的边界

当前源码用这个宏区分：
```text
SlotIsPhysical(slot) = slot->data.database == InvalidOid SlotIsLogical(slot) = slot->data.database != InvalidOid
```
physical slot 不绑定数据库。 它主要保护 WAL 给 physical standby 继续接收。 logical slot 绑定数据库。 它除了保护 WAL，还可能保护 tuple / catalog 的 xmin horizon。 `slot.h` 中 `ReplicationSlot` 的注释明确区分：
```text
logical decoding 对 effective_xmin / effective_catalog_xmin 更严格； ordinary streaming replication 过早移除旧行版本的最坏结果通常是 standby query cancellation。
```
本节只关注 WAL retention。 但诊断时要记住：
```text
physical slot: 主要风险是 pg_wal 被 restart_lsn 拖住。 logical slot: 风险同时包括 pg_wal 保留和 VACUUM / catalog horizon 被拖住。
```
这就是为什么一个 inactive logical slot 经常比 inactive physical slot 更危险。 它既可能占 WAL，又可能阻止清理旧 tuple / catalog tuples。

### 4.3 `last_saved_restart_lsn`: crash-safe slot retention 的保守边界

`ReplicationSlot.data.restart_lsn` 是内存中的当前位置。 `last_saved_restart_lsn` 是最近已经 flush 到 slot on-disk state 的位置。 `ReplicationSlotsComputeRequiredLSN()` 对 persistent slot 有一个保守处理：
```text
如果 persistent slot 的 restart_lsn 已经前进， 但 last_saved_restart_lsn 更老， 则 WAL 删除边界按 last_saved_restart_lsn 计算。
```
原因是 crash 后 slot 会从磁盘状态恢复。 如果 checkpoint 删除了 `last_saved_restart_lsn` 到 `restart_lsn` 之间的 WAL，而 slot state 还没安全落盘，崩溃恢复后 slot 可能又需要这些 WAL。 `CheckPointReplicationSlots()` 会在 checkpoint 中保存 dirty slot。 保存完成后，如果 `last_saved_restart_lsn` 更新了，再调用 `ReplicationSlotsComputeRequiredLSN()` 重新汇总。 所以 slot retention 不是只看一个字段。 更准确的语义是：
```text
restart_lsn + persistency + last_saved_restart_lsn + invalidated 共同决定这个 slot 是否还参与 WAL 保留。
```

### 4.4 `XLogCtl->replicationSlotMinLSN`: slots 到 xlog 的汇总边界

slot 模块不会直接删除 WAL。 它把所有有效 slot 中最老的需要位置汇总到 xlog 模块：
```text
ReplicationSlotsComputeRequiredLSN() -> XLogSetReplicationSlotMinimumLSN(min_required)
```
`xlog.c` 后续通过：
```text
XLogGetReplicationSlotMinimumLSN()
```
拿到这个全局最老 slot LSN。 这里有一个容易漏掉的源码注释：
```text
ReplicationSlotsComputeRequiredLSN() 不把 max_slot_wal_keep_size 纳入计算， 因为 slot.c 不知道应该和哪个当前 WAL 位置比较。
```
这个比较发生在 `xlog.c` 的 `KeepLogSeg()` 中。 所以 `max_slot_wal_keep_size` 是 xlog 清理阶段才真正生效的配置。

### 4.5 `wal_keep_size`: 没有消费者身份的保留地板

`wal_keep_size` 的 GUC 定义在 `guc_parameters.dat`：
```text
name = wal_keep_size context = PGC_SIGHUP group = REPLICATION_SENDING unit = MB variable = wal_keep_size_mb boot_val = 0 min = 0
```
它的源码入口不在 walsender。 它在 `KeepLogSeg()` 的最后生效：
```text
如果 wal_keep_size_mb > 0， 保留至少这么多 segment 的最近 WAL。
```
这意味着 `wal_keep_size` 没有 slot 名字。 没有 active/inactive。 没有 feedback。 没有 invalidation。 它只是把删除边界往旧方向推一段固定距离。 它适合保护“短暂断线后从最近 WAL 追上”的场景。 它不适合承诺某个具体消费者一定能追上。 如果 primary 生成 WAL 的速度超过预估，固定大小的 `wal_keep_size` 会很快被消耗掉。

### 4.6 `max_slot_wal_keep_size`: slot retention 的失败边界

`max_slot_wal_keep_size` 的 GUC 定义在 `guc_parameters.dat`：
```text
name = max_slot_wal_keep_size context = PGC_SIGHUP group = REPLICATION_SENDING unit = MB variable = max_slot_wal_keep_size_mb boot_val = -1 min = -1
```
`-1` 表示没有这个上限。 源码描述说 slot 如果占用这么多 WAL 空间，会被标记为 failed，并释放 segment 供删除或回收。 关键点是：
```text
它只约束 replication slot 造成的保留。 它不约束 wal_keep_size。 它不约束 archive backlog。 它不约束 WAL summarization。 它也不约束 max_wal_size 自身的 checkpoint 行为。
```
在 `KeepLogSeg()` 中，slot 最老 LSN 先把 `segno` 拉回。 随后如果 `max_slot_wal_keep_size_mb >= 0`，源码把 slot 能拉回的距离裁剪到：
```text
currSegNo - ConvertToXSegs(max_slot_wal_keep_size_mb)
```
如果 slot 需要的 `restart_lsn` 更老，`KeepLogSeg()` 不会继续保护它。 随后 `InvalidateObsoleteReplicationSlots()` 会看到该 slot 的 `restart_lsn` 早于将要保留的边界，并把它 invalidated。

### 4.7 `wal_status`: 可观测的 retention 状态，不是持久字段

`pg_replication_slots.wal_status` 来自 `pg_get_replication_slots()`。 这个 SRF 在 `slotfuncs.c` 中读取 slot 内容，然后调用：
```text
GetWALAvailability(slot.restart_lsn)
```
`GetWALAvailability()` 在 `xlog.c` 中返回：
```text
reserved: target LSN 对应 segment 还可用，并且在 max_wal_size 范围内。 extended: segment 还可用，但主要是 slot 把保留扩展到了 max_wal_size 之外。 unreserved: segment 目前还没被删，但已经不再被保留，下一次 checkpoint 可能删除。 removed: segment 已经被移除。 invalid_lsn: slot 还没有有效 restart_lsn。
```
`pg_replication_slots` 显示时还会把 invalidated slot 映射成 `lost`。 所以视图中的常见文本是：
```text
reserved extended unreserved lost NULL
```
`wal_status` 是瞬时推导值。 它不是 slot on-disk 文件里的持久状态。 真正持久的失效原因在 `data.invalidated`。 视图中的 `invalidation_reason` 会显示类似：
```text
wal_removed rows_removed wal_level_insufficient idle_timeout
```
本节关注的是 `wal_removed`。

### 4.8 `safe_wal_size`: 到失败边界的剩余字节估计

`safe_wal_size` 也来自 `pg_get_replication_slots()`。 它只在两个条件满足时计算：
```text
slot 还没有 lost； max_slot_wal_keep_size 已配置为非 -1。
```
源码使用的核心计算是：
```text
targetSeg = segment(restart_lsn) slotKeepSegs = max_slot_wal_keep_size in segments keepSegs = wal_keep_size in segments failSeg = targetSeg + max(slotKeepSegs, keepSegs) + 1 safe_wal_size = failLSN - current_lsn
```
注意 `wal_keep_size` 也进入这里。 因为即使 slot 上限较小，`wal_keep_size` 仍可能让相同 segment 在更长时间内不被删除。 但这个值仍然不是 SLA。 它没有预测未来 WAL 生成速度。 它也没有预测 archive backlog、checkpoint 时间、standby 网络恢复时间。 它只是回答：
```text
按当前 restart_lsn、当前 WAL 位置和配置， 再生成多少 WAL 后这个 slot 会跨过删除风险边界。
```

### 4.9 archive status: `.ready` 和 `.done`

`xlogarchive.c` 用 `pg_wal/archive_status` 下的小文件表示归档状态。 `XLogArchiveNotify()` 创建：
```text
<segment>.ready
```
archiver 完成后，`pgarch.c` 中 `pgarch_archiveDone()` 把它改成：
```text
<segment>.done
```
`XLogArchiveCheckDone()` 是删除旧 WAL 前的关键门槛。 如果 archiving 没启用，返回 true。 如果启用归档并且还没有 `.done`，它会确保 `.ready` 存在并返回 false。 这表示：
```text
本地 checkpoint 已经认为这个 segment 不再需要， 但只要它尚未成功归档，本地 pg_wal 也不会删除它。
```
这同时保护 restore 能力，也引入 primary 磁盘风险。 归档失败时，旧 WAL 会持续堆在 `pg_wal`。 `max_slot_wal_keep_size` 不会解决 archive backlog。

## 5. 主流程源码 walkthrough

### 5.1 WAL segment 写满时通知 archive 和 checkpoint

WAL 删除不是从删除开始的。 先看生成端。 `xlog.c` 的 WAL 写出路径在完成一个 segment 时会做几件事：
```text
issue_xlog_fsync() WalSndWakeupRequest() LogwrtResult.Flush = LogwrtResult.Write if XLogArchivingActive(): XLogArchiveNotifySeg(openLogSegNo, tli) XLogCtl->lastSegSwitchTime = now if XLogCheckpointNeeded(openLogSegNo): RequestCheckpoint(CHECKPOINT_CAUSE_XLOG)
```
这里的 `XLogArchiveNotifySeg()` 只是创建 archive notification。 它不代表 archive 已经成功。 它只是告诉 archiver：
```text
这个完整 segment 可以被复制到外部 archive。
```
`XLogCheckpointNeeded()` 则用当前 segment 与 `RedoRecPtr` 的距离判断是否已经超过 checkpoint 触发阈值。 这个阈值来自 `CalculateCheckpointSegments()`。 当前源码按 `max_wal_size` 和 `checkpoint_completion_target` 计算：
```text
CheckPointSegments = ConvertToXSegs(max_wal_size_mb) / (1 + checkpoint_completion_target)
```
所以 `max_wal_size` 的作用不是直接“删除到这么小”。 它先影响 checkpoint 触发频率。 真正删除哪些 segment，要等 checkpoint 后的清理阶段。

### 5.2 checkpoint 完成后才进入旧 WAL 清理

`CreateCheckPoint()` 的前半部分做的是 crash recovery 正确性工作。 它确定新的 redo pointer。 它写 checkpoint record。 它 flush WAL。 它更新 control file。 它调用 `CheckPointGuts()` 刷各类持久状态。 这些不是本节重点。 本节重点在 checkpoint critical updates 结束之后：
```text
SyncPostCheckpoint() UpdateCheckPointDistanceEstimate() XLByteToSeg(RedoRecPtr, _logSegNo) KeepLogSeg(recptr, &_logSegNo) InvalidateObsoleteReplicationSlots(..., _logSegNo, ...) if invalidated: XLByteToSeg(RedoRecPtr, _logSegNo) KeepLogSeg(recptr, &_logSegNo) _logSegNo-- RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr, currentTLI) PreallocXlogFiles()
```
`RedoRecPtr` 给出 crash recovery 还需要的最老位置。 `KeepLogSeg()` 可能把 `_logSegNo` 再往旧方向退。 退的原因包括：
```text
replication slots WAL summarization wal_keep_size
```
`KeepLogSeg()` 返回后，`_logSegNo` 表示最老必须保留的 segment。 调用者再执行：
```text
_logSegNo--
```
然后把这个值传给 `RemoveOldXlogFiles()`。 因此 `RemoveOldXlogFiles()` 删除的是：
```text
小于最老必须保留 segment 的旧文件。
```
这个 off-by-one 边界在读源码时很重要。 如果不看 `_logSegNo--`，很容易误以为 `KeepLogSeg()` 算出来的 segment 也会被删除。

### 5.3 `KeepLogSeg()` 如何合并 slot、max_slot_wal_keep_size、wal_keep_size

`KeepLogSeg(recptr, &logSegNo)` 的 mental model 是：
```text
从当前 WAL 位置 recptr 得到 currSegNo。 先假设只保留到 currSegNo。 如果 slot 需要更老的 WAL，把 segno 拉回 slot minimum LSN。 如果配置 max_slot_wal_keep_size，把 slot 能拉回的距离裁剪。 如果 WAL summarization 需要更老 WAL，再拉回。 如果 wal_keep_size 要求保留最近 N MB，再保证至少保留 N MB。 最后如果算出的 segno 比调用者传入的 logSegNo 更老，就更新调用者。
```
伪代码：
```text
currSegNo = segment(recptr) segno = currSegNo keep = XLogGetReplicationSlotMinimumLSN() if valid(keep) and keep < recptr: segno = segment(keep) if max_slot_wal_keep_size >= 0: if currSegNo - segno > slot_keep_segs: segno = currSegNo - slot_keep_segs keep = GetOldestUnsummarizedLSN() if valid(keep): segno = min(segno, segment(keep)) if wal_keep_size > 0: segno = min(segno, currSegNo - keep_segs) *logSegNo = min(*logSegNo, segno)
```
这个顺序揭示两个结论。 第一，`max_slot_wal_keep_size` 只裁剪 slot 那一段。 如果 `wal_keep_size` 更大，后面仍会把保留边界拉得更旧。 第二，archive 不在 `KeepLogSeg()` 中。 archive 不参与计算“逻辑上最老需要保留的 segment”。 archive 在真正删除前通过 `XLogArchiveCheckDone()` 再挡一次。 这就是为什么诊断 pg_wal 膨胀时必须分两步问：
```text
旧 WAL 是因为逻辑保留边界太旧？ 还是逻辑上可以删，但 archive 没完成？
```

### 5.4 slot 超过上限时如何失效

如果 `max_slot_wal_keep_size` 把 slot 保留边界裁剪到了更靠前的位置，某些 slot 的 `restart_lsn` 可能落在边界之前。 这时 `CreateCheckPoint()` 调用：
```text
InvalidateObsoleteReplicationSlots( RS_INVAL_WAL_REMOVED | RS_INVAL_IDLE_TIMEOUT, oldestSegno, InvalidOid, InvalidTransactionId)
```
`InvalidateObsoleteReplicationSlots()` 把 `oldestSegno` 转成 `oldestLSN`。 然后逐个扫描 slot。 核心判断在 `DetermineSlotInvalidationCause()`：
```text
if possible_causes includes RS_INVAL_WAL_REMOVED: if slot.restart_lsn is valid and slot.restart_lsn < oldestLSN: return RS_INVAL_WAL_REMOVED
```
如果 slot inactive，它会：
```text
MyReplicationSlot = s s->active_proc = MyProcNumber s->data.invalidated = RS_INVAL_WAL_REMOVED s->data.restart_lsn = InvalidXLogRecPtr s->last_saved_restart_lsn = InvalidXLogRecPtr ReplicationSlotMarkDirty() ReplicationSlotSave() ReplicationSlotRelease()
```
如果 slot active，它不能直接把另一个进程正在使用的 slot 改完就走。 源码会记录 active PID。 非 startup 进程路径用 `kill(active_pid, SIGTERM)`。 startup 进程路径用 `SignalRecoveryConflict(..., RECOVERY_CONFLICT_LOGICALSLOT)`。 然后等待 slot 的 condition variable。 等 owning process 退出或释放后，再重试并完成 invalidation。 这解释了一个运行时现象：
```text
max_slot_wal_keep_size 被触发时， 你可能先看到 walsender / logical worker 被终止， 然后 pg_replication_slots 中该 slot 变成 lost / invalidation_reason = wal_removed。
```
它不是普通的优雅 catch-up。 它是 availability tradeoff 中 primary 侧选择了保护本地磁盘边界。

### 5.5 invalidation 后为什么要重新计算删除边界

`CreateCheckPoint()` 在 invalidation 返回 true 后，会重新执行：
```text
XLByteToSeg(RedoRecPtr, _logSegNo) KeepLogSeg(recptr, &_logSegNo)
```
原因是：
```text
刚才有 slot 失效了； 失效 slot 不再参与 ReplicationSlotsComputeRequiredLSN()； 因此全局 slot minimum LSN 可能变新； 旧 WAL 删除边界也可能继续向前推进。
```
这是一个典型的内核状态推进模式：
```text
先按当前状态计算边界； 发现某些对象已经越界，标记失效； 失效改变全局最小值； 再按新状态重算边界； 最后执行不可逆资源回收。
```
这个顺序很重要。 如果先删 WAL 再标记 slot，active walsender 可能还以为 slot 有效。 如果标记 slot 后不重算，又会保守地少删一些 WAL。 PostgreSQL 选择在 checkpoint 清理阶段接受一次 restart loop 风格的复杂性。

### 5.6 `RemoveOldXlogFiles()` 删除前还要问 archive

`RemoveOldXlogFiles(segno, lastredoptr, endptr, insertTLI)` 扫描 `pg_wal`。 它忽略文件名中的 timeline 部分，只按 segment 编号比较新旧。 源码注释说明这样可以避免过早删除 parent timeline 上的 segment。 对每个旧 segment，它先调用：
```text
XLogArchiveCheckDone(xlde->d_name)
```
只有返回 true，才会继续：
```text
UpdateLastRemovedPtr(xlde->d_name) RemoveXlogFile(...)
```
`RemoveXlogFile()` 再决定 recycle 还是 unlink。 是否 recycle 取决于：
```text
wal_recycle 是否还需要未来预分配 segment 文件是否是普通文件 InstallXLogFileSegmentActive InstallXLogFileSegment() 是否成功
```
不管是 recycle 还是 remove，最后都会调用：
```text
XLogArchiveCleanup(segname)
```
清掉对应 `.done` 或 `.ready` 状态文件。 因此 archive 的保护发生在本地删除前最后一步。 它不是消费者身份。 它是外部 durability handshake。

### 5.7 standby 上的 restartpoint 也会清理 WAL

standby 不执行普通 checkpoint。 它执行 restartpoint。 `CreateRestartPoint()` 与 `CreateCheckPoint()` 类似，但它使用 recovery 中已经 replay 的 checkpoint record 建立新的恢复起点。 旧 WAL 清理阶段大致是：
```text
XLByteToSeg(RedoRecPtr, _logSegNo) receivePtr = GetWalRcvFlushRecPtr() replayPtr = GetXLogReplayRecPtr() endptr = max(receivePtr, replayPtr) KeepLogSeg(endptr, &_logSegNo) InvalidateObsoleteReplicationSlots(...) _logSegNo-- RemoveOldXlogFiles(_logSegNo, RedoRecPtr, endptr, replayTLI) PreallocXlogFiles(endptr, replayTLI) archive_cleanup_command
```
standby 上 `KeepLogSeg()` 用的是 `endptr`。 也就是已接收和已回放中更靠后的 LSN。 这是因为 standby 本地 `pg_wal` 中可能已经接收到但还没 replay 的 WAL。 restartpoint 不能只看 replay 位置就误删还在接收链路中的 segment。 最后的 `archive_cleanup_command` 是另一个边界。 它清理的是外部 archive 中不再需要的文件。 PostgreSQL 会把 `%r` 替换成 last restartpoint 文件名。 但外部 archive retention 策略仍由用户命令决定。

## 6. slot 生命周期 / ownership / cleanup

### 6.1 谁创建 slot

SQL 函数路径在 `slotfuncs.c`。 创建 physical slot 的入口会调用：
```text
ReplicationSlotCreate(name, false, persistency, two_phase=false, failover=false) if restart_lsn invalid: ReplicationSlotReserveWal() else: slot->data.restart_lsn = restart_lsn ReplicationSlotSave() ReplicationSlotRelease()
```
`ReplicationSlotReserveWal()` 根据 slot 类型选择初始 `restart_lsn`。 physical slot：
```text
restart_lsn = GetRedoRecPtr()
```
logical slot：
```text
primary 上使用 GetXLogInsertRecPtr() standby 上使用 GetXLogReplayRecPtr()
```
这里 physical slot 使用 redo pointer 是因为 physical standby 可以从最后 checkpoint 的 redo 点开始恢复。 logical slot 则需要能从某个 decoding 可建立一致快照的位置开始。 `ReplicationSlotReserveWal()` 持有 `ReplicationSlotAllocationLock` 的 exclusive 模式。 源码注释解释这是为了和 `CheckPointReplicationSlots()` 串行。 目标是不让 checkpoint 在 slot 刚保留 WAL 的同时，把所需 WAL 删除掉。

### 6.2 谁持有 slot

slot 当前 owner 由 `active_proc` 表示。 `ReplicationSlotAcquire(name, nowait, error_if_invalid)` 做几件事：
```text
查找 slot 是否存在。 如果 active_proc 已被其他进程占用，按 nowait 决定报错或等待。 把 active_proc 设为 MyProcNumber。 清掉 inactive_since。 如果 error_if_invalid 且 slot 已 invalidated，报错。 设置 MyReplicationSlot。
```
physical walsender 在 `StartReplication()` 中 acquire physical slot。 它还会拒绝 logical slot：
```text
cannot use a logical replication slot for physical replication
```
logical walsender 在 `StartLogicalReplication()` 中 acquire logical slot，并创建 decoding context。 一个关键边界：
```text
StartReplication() 不校验 slot.restart_lsn 是否覆盖客户端请求的 startpoint。 如果 WAL segment 不存在，会在后续读取 WAL 时失败。
```
这解释了为什么某些错误不是在 `START_REPLICATION` 刚发出时出现。 消费者可能已经进入 COPY BOTH，随后读到缺失 segment 才报：
```text
requested WAL segment ... has already been removed
```

### 6.3 谁推进 physical slot 的 `restart_lsn`

physical replication 的 `restart_lsn` 由 standby 反馈推进。 在 `walsender.c` 中：
```text
ProcessStandbyReplyMessage() -> update MyWalSnd write / flush / apply -> if physical slot: PhysicalConfirmReceivedLocation(flushPtr)
```
`PhysicalConfirmReceivedLocation()` 只在值变化时：
```text
slot->data.restart_lsn = flushPtr ReplicationSlotMarkDirty() ReplicationSlotsComputeRequiredLSN() PhysicalWakeupLogicalWalSnd()
```
源码还特意说明，不会每次都 `ReplicationSlotSave()`。 因为这会浪费 IO。 最坏情况只是统计视图短期不准，或者 WAL 删除更保守。 这和 `last_saved_restart_lsn` 的 crash-safe 逻辑一致。 内存中可以快速推进。 持久边界在 checkpoint 时收敛。

### 6.4 谁推进 logical slot 的 `restart_lsn`

logical walsender 通过 logical decoding context 读取 WAL。 `StartLogicalReplication()` 中：
```text
CreateDecodingContext(cmd->startpoint, ...) XLogBeginRead(reader, MyReplicationSlot->data.restart_lsn) sentPtr = MyReplicationSlot->data.confirmed_flush MyWalSnd->sentPtr = MyReplicationSlot->data.restart_lsn WalSndLoop(XLogSendLogical)
```
这里有一个看似矛盾的状态组合。 对客户端来说，下一批逻辑变更从 `confirmed_flush` 之后继续。 对 WAL reader 来说，可能要从更老的 `restart_lsn` 开始读。 原因是 decoding 需要重建事务和 snapshot 状态。 所以 logical slot 的 WAL retention 不能只看客户端已经确认到哪个 commit。 必须看 decoding 仍可能需要从哪里读取 WAL。

### 6.5 谁释放 slot

`ReplicationSlotRelease()` 处理几种情况。 ephemeral slot：
```text
ReplicationSlotDropAcquired()
```
persistent slot：
```text
如果只是临时 effective_xmin，清掉并重算 required xmin。 active_proc = INVALID_PROC_NUMBER inactive_since = now ConditionVariableBroadcast(active_cv) MyReplicationSlot = NULL
```
注意 release 并不意味着资源保留结束。 persistent slot 释放后仍然存在。 它的 `restart_lsn` 仍然参与 `ReplicationSlotsComputeRequiredLSN()`。 这正是 disconnected consumer 能被保护的原因。 也是 inactive slot 能拖满 `pg_wal` 的原因。

### 6.6 ERROR / abort 时谁兜底

slot 模块在 `ReplicationSlotInitialize()` 注册：
```text
before_shmem_exit(ReplicationSlotShmemExit, 0)
```
`ReplicationSlotShmemExit()` 如果 `MyReplicationSlot != NULL`，会调用 `ReplicationSlotRelease()`。 然后执行 `ReplicationSlotCleanup(false)`。 这保证进程异常退出时不会永久留下 active owner。 但 persistent slot 本身不会因为 walsender 退出而自动 drop。 它只会变 inactive。 所以运维判断时要区分：
```text
active=false: 这个 slot 当前没有进程持有。 slot 不存在: 这个 slot 已被 drop。 invalidated: slot 仍可能存在，但已经不再能访问原来的资源。
```

## 7. archive 与 backup retention 的源码边界

### 7.1 archive 如何挡住本地删除

当一个 WAL segment 写满时，`XLogArchiveNotifySeg()` 创建 `.ready`。 archiver 进程在 `pgarch.c` 中：
```text
pgarch_MainLoop() -> pgarch_ArchiverCopyLoop() -> pgarch_readyXlog() -> pgarch_archiveXlog() -> pgarch_archiveDone()
```
`pgarch_readyXlog()` 扫描 `pg_wal/archive_status` 中的 `.ready` 文件。 `pgarch_archiveXlog()` 调用 archive library 或 shell archive command。 `pgarch_archiveDone()` 把 `.ready` 改成 `.done`。 旧 WAL 清理时，`XLogArchiveCheckDone()` 只认 `.done` 或 archiving inactive。 如果 `.ready` 存在，它返回 false。 如果两个状态文件都没有，它会重新创建 `.ready`，返回 false。 这意味着 archive failure 会让 `pg_wal` 增长。 而且这种增长和 slot 无关。 诊断时如果 `pg_replication_slots` 都很健康，但 `pg_wal` 仍增长，应立即看：
```text
pg_stat_archiver pg_wal/archive_status/*.ready server log 中 archive_command 失败
```

### 7.2 archive 不是消费者 ack

archive 成功只说明：
```text
这个 segment 已经复制到外部归档位置。
```
它不说明 standby 已经接收。 它不推进 physical slot 的 `restart_lsn`。 它不推进 logical slot 的 `confirmed_flush`。 它也不会让 `pg_stat_replication.flush_lsn` 前进。 所以 archive 可以支持后续 restore 或 standby 通过 `restore_command` 补 WAL。 但它不是 streaming replication 的 feedback。

### 7.3 restore 如何使用 archive

`RestoreArchivedFile()` 是 archive recovery 读取外部 WAL 的入口。 它只在 `ArchiveRecoveryRequested` 且 `recoveryRestoreCommand` 非空时尝试 restore。 源码明确说明，archive recovery 总是偏好 archived log file，即使 `pg_wal` 中有同名文件。 原因是 base backup 拷贝出来的 `pg_wal` 文件可能是旧的、未填满的或部分填充的。 如果 restore command 失败，它返回 `pg_wal/<xlogfname>` 作为 fallback 路径。 也就是说 recovery 的顺序不是：
```text
只信 pg_wal。
```
而是：
```text
archive recovery 模式下优先 restore_command； restore 不可用时，再尝试 pg_wal 中现有文件。
```
这解释了为什么 archive retention 是 backup retention 的关键。 一个 base backup 能否恢复，不只取决于 backup tar 是否在。 还取决于从 `backup_label` 的 start WAL location 到目标恢复点之间的 WAL 是否还在 archive 中。

### 7.4 base backup 期间 PostgreSQL 保证什么

`perform_base_backup()` 的主线：
```text
do_pg_backup_start() -> force checkpoint / restartpoint -> 得到 backup startpoint 和 start timeline -> 发送 backup_label -> 发送数据文件 -> basebackup_progress_wait_wal_archive() -> do_pg_backup_stop() -> 可选包含 WAL -> 发送 backup manifest
```
`do_pg_backup_start()` 会增加 `runningBackups`。 这样在线备份期间需要 full-page writes 来修复 torn page 风险。 然后它强制 checkpoint。 checkpoint 的 redo pointer 成为 backup 需要的最小 WAL 起点。 `xlogbackup.c` 的 `build_backup_content()` 会把这些写入 `backup_label`：
```text
START WAL LOCATION CHECKPOINT LOCATION BACKUP METHOD BACKUP FROM START TIME LABEL START TIMELINE
```
如果生成 backup history，还会包含 STOP WAL LOCATION 和 STOP TIME。 这些文件不是 WAL retention 机制。 它们是恢复时解释 base backup 需要哪些 WAL 的 metadata。

### 7.5 base backup 默认不把 `pg_wal` 当普通目录拷贝

`sendDir()` 扫描数据目录时遇到 `./pg_wal`，会把它作为空目录写入 tar。 源码注释说：
```text
WAL segments need to be fetched from WAL archive anyway.
```
它还会写空的：
```text
./pg_wal/archive_status ./pg_wal/summaries
```
这避免把运行中的 `pg_wal` 半成品当普通文件备份进去。 如果 base backup 选择 include WAL，`perform_base_backup()` 会在 `do_pg_backup_stop()` 之后再扫描 `pg_wal`。 它收集覆盖 `startptr` 到 `endptr` 的 WAL segment。 然后用 `CheckXLogRemoved()` 检查所需 segment 是否已经被移除。 如果缺失，会报错。 这就是 `pg_basebackup -X fetch` 类路径的风险模型：
```text
备份结束时再从 primary pg_wal 抓所需 WAL； 如果备份期间 WAL 生成太多且没有足够 retention，可能抓不到。
```

### 7.6 backup stop 如何等待 archive

`do_pg_backup_stop(state, waitforarchive)` 做几件事。 非 recovery 中：
```text
写 XLOG_BACKUP_END record。 RequestXLogSwitch(false)。 写 backup history file。 CleanupBackupHistory()，顺便通知 archiver。
```
如果 `waitforarchive` 为 true 且 archiving active，它会等待：
```text
XLogArchiveIsBusy(lastxlogfilename) == false XLogArchiveIsBusy(histfilename) == false
```
等待期间会周期性打印 notice / warning。 如果 archiving 没启用，它会 notice：
```text
必须通过其他方式确保所有 required WAL segments 被复制。
```
这段源码把 backup retention 的责任边界说得很直：
```text
PostgreSQL 可以等待 archive 交付所需 WAL； 如果没有 archive，就只能依赖 streaming、wal_keep_size 或外部脚本； 系统无法知道外部是否真的保存了完整 WAL 链。
```

### 7.7 backup retention 是外部策略，不是 slot 自动完成

一个可恢复备份需要：
```text
base backup 文件 backup_label / manifest / history metadata 从 backup startpoint 到 restore target 的所有 WAL 正确的 timeline history
```
PostgreSQL 的 base backup 代码会产生 metadata。 它可以选择等待 archive。 它可以选择把一段 WAL 包进备份 tar。 但它不会替你决定：
```text
外部 archive 保留几天； 哪些 base backup 已过期； 哪些 WAL 仍被最老保留 backup 需要； PITR 目标是否还在保留窗口内。
```
这属于 backup retention policy。 如果外部备份系统删掉了某个 base backup 之后的 WAL 链，PostgreSQL 的 slot 不会知道。 反过来，如果 slot 仍保留 WAL，也不代表某个历史 base backup 可恢复。 slot 是 live consumer retention。 backup retention 是 restore chain retention。 两者交集是 WAL 文件，但 ownership 完全不同。

## 8. WAL availability 状态机

`GetWALAvailability(targetLSN)` 可以看作一段可观测状态机。 它先处理无效 LSN：
```text
targetLSN invalid -> WALAVAIL_INVALID_LSN
```
然后计算当前写入位置：
```text
currpos = GetXLogWriteRecPtr() currSeg = segment(currpos)
```
接着通过 `KeepLogSeg(currpos, &oldestSlotSeg)` 得到考虑 slot、wal_keep_size、summarization 后的最老保留 segment。 它还读取：
```text
oldestSeg = XLogGetLastRemovedSegno() + 1 oldestSegMaxWalSize = currSeg - ConvertToXSegs(max_wal_size_mb) - 1 的近似边界 targetSeg = segment(targetLSN)
```
然后返回：
```text
targetSeg >= oldestSlotSeg: 如果 targetSeg >= oldestSegMaxWalSize: reserved 否则: extended targetSeg >= oldestSeg: unreserved 否则: removed
```
这个状态机有几个诊断含义。 `reserved` 表示还在正常 max_wal_size 范围内。 `extended` 表示 slot 等 retention 机制把 WAL 保留扩展到了 max_wal_size 之外。 `unreserved` 是最危险的预警状态。 它表示文件现在还在，但系统已经不承诺继续保留。 下一次 checkpoint 可能删除。 `lost` 表示对这个 slot 来说，所需 WAL 已经不可用。 消费者不能靠继续等待恢复。 它需要重新初始化、重建 slot、重做 base backup 或走其它恢复链路。

## 9. 错误路径 / 异常路径 / fallback

### 9.1 没有 slot，只靠 `wal_keep_size`

场景：
```text
standby 断线； primary 没有配置 physical replication slot； wal_keep_size = 1GB； primary 在断线期间生成 10GB WAL； 多次 checkpoint 后旧 segment 被删除或回收。
```
结果：
```text
standby 重连时请求旧 LSN； walsender 尝试打开对应 segment； WalSndSegmentOpen() 发现 ENOENT； 报 requested WAL segment ... has already been removed。
```
如果 standby 有 `restore_command` 且 archive 还保留这些 WAL，它可能先从 archive 补上。 如果 archive 也没有，就只能重做 base backup。 这里 `wal_keep_size` 没有失败事件。 它没有 slot 可标记 invalidated。 它只是固定窗口被 WAL 生成速度冲掉了。

### 9.2 有 slot，但没有 `max_slot_wal_keep_size`

场景：
```text
primary 上有 persistent physical slot； standby 断线； max_slot_wal_keep_size = -1； primary 持续生成 WAL。
```
结果：
```text
slot.restart_lsn 不推进； ReplicationSlotsComputeRequiredLSN() 让 xlog 保留从 restart_lsn 开始的 WAL； KeepLogSeg() 会把删除边界拉到 slot 需要的位置； pg_wal 可能无限增长。
```
这最大化消费者可追赶性。 也最大化 primary 被拖死的风险。 适合有外部磁盘容量监控和明确修复流程的环境。 不适合把 slot 当“永远安全”的默认开关。

### 9.3 有 slot，并配置 `max_slot_wal_keep_size`

场景：
```text
primary 上有 persistent slot； standby 断线； max_slot_wal_keep_size = 20GB； primary 生成 WAL 超过这个边界； checkpoint 开始清理旧 WAL。
```
结果：
```text
KeepLogSeg() 不再让这个 slot 把边界拉过 20GB。 InvalidateObsoleteReplicationSlots() 判断 restart_lsn < oldestLSN。 slot invalidated with RS_INVAL_WAL_REMOVED。 active walsender 被终止或 inactive slot 被直接标记。 旧 WAL 可以被删除或回收。
```
消费者失去追赶能力。 primary 磁盘可用性得到保护。 这就是本节主矛盾的源码落点。

### 9.4 archive 卡住

场景：
```text
archive_mode = on； archive_command 持续失败； slot 都健康； wal_keep_size 很小； checkpoint 已经认为旧 WAL 可以删除。
```
结果：
```text
RemoveOldXlogFiles() 调用 XLogArchiveCheckDone()； 发现没有 .done； 如果没有 .ready 就创建 .ready； 返回 false； 旧 segment 留在 pg_wal。
```
这里 `pg_wal` 增长不是复制 slot 问题。 是 archive durability handshake 没完成。 `max_slot_wal_keep_size` 不会帮你删除这些文件。 正确诊断入口是 archiver。

### 9.5 base backup 等待 archive 卡住

场景：
```text
BASE_BACKUP 默认要求等待 archive； archive_command 失败； 备份的数据文件已经发送完。
```
`do_pg_backup_stop()` 会等待 last WAL file 和 history file 不再 busy。 一段时间后日志会提示：
```text
base backup done, waiting for required WAL segments to be archived still waiting for all required WAL segments to be archived
```
这不是普通复制延迟。 它是 backup 正确性边界：
```text
没有 required WAL，base backup 不可用。
```
如果用户选择 `nowait` 或取消等待，就必须用其它机制确保 WAL 链完整。 否则备份文件本身存在也不能恢复。

## 10. 成本、资源与跨模块传播

### 10.1 WAL retention 的主要资源压力

WAL retention 首先消耗磁盘。 压力来源包括：
| 来源 | 传播路径 |
| --- | --- |
| disconnected physical slot | `restart_lsn` 停住，`pg_wal` 保留旧 segment。 |
| inactive logical slot | `restart_lsn` 停住，同时 `catalog_xmin` 可能拖住 VACUUM。 |
| `wal_keep_size` 过大 | 无条件保留最近 N MB WAL。 |
| archive failure | `.ready` 堆积，`XLogArchiveCheckDone()` 阻止删除。 |
| long base backup with fetch WAL | 需要 `startptr..endptr` 之间 WAL 在结束时仍可读。 |
| high WAL generation rate | safe window 按时间迅速缩短。 |
| checkpoint 延迟 | 删除动作延后，状态可能停留在 unreserved。 |

### 10.2 checkpoint 是资源回收批处理点

WAL 删除和回收集中在 checkpoint / restartpoint 后。 这会带来两个运行时现象。 第一，`pg_wal` 空间可能阶梯式变化。 WAL 持续生成，但旧 segment 删除常在 checkpoint 后批量发生。 第二，slot 状态也可能在 checkpoint 后突然变化。 `wal_status` 从 `extended` 到 `unreserved` 再到 `lost`，不是每个 WAL record 都刷新一次。 它取决于 checkpoint、last removed segno、当前写入位置和 slot restart_lsn。

### 10.3 `max_wal_size` 不是硬磁盘上限

`max_wal_size` 参与 checkpoint 触发和 `XLOGfileslop()` 的 recycle 估计。 但 `pg_wal` 可以超过它。 原因包括：
```text
slot 保留扩展到 max_wal_size 之外。 archive 未完成阻止删除。 wal_keep_size 大于默认窗口。 checkpoint 尚未完成。 WAL summarization 仍需要旧 WAL。
```
所以看到 `pg_wal` 大于 `max_wal_size` 时，不能直接判断是 bug。 要回到 retention owner。

### 10.4 slot 数量带来的扫描成本

`ReplicationSlotsComputeRequiredLSN()` 会扫描 slot 数组。 当前本地源码扫描范围是：
```text
max_replication_slots + max_repack_replication_slots
```
每个 in-use slot 会短暂持有其 spinlock 读取状态。 `InvalidateObsoleteReplicationSlots()` 也要扫描 slot。 如果有大量 slot，checkpoint 清理阶段的管理成本会随 slot 上限增长。 通常这不是 WAL retention 的主要瓶颈。 但在大量 logical slots、频繁确认、频繁 checkpoint 的场景中，slot 管理路径会变得更可见。

### 10.5 logical slot 的双重传播

logical slot 会把压力传播到两个方向。 WAL 方向：
```text
restart_lsn 停住 -> pg_wal 保留增长。
```
MVCC / catalog 方向：
```text
effective_xmin / effective_catalog_xmin 停住 -> VACUUM 不能移除某些旧版本。
```
`max_slot_wal_keep_size` 只解决前者的一部分。 它不会自动解除 catalog xmin 带来的 bloat 风险。 slot invalidation 的其它 cause 中有 `RS_INVAL_HORIZON`，但那是 tuple / row removal 冲突路径，不是本节的 WAL removed 主线。

## 11. 观测与诊断入口

### 11.1 先看配置边界

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
SHOW wal_keep_size;
SHOW max_slot_wal_keep_size;
SHOW max_wal_size;
SHOW min_wal_size;
SHOW archive_mode;
SHOW archive_command;
SHOW archive_library;
SHOW archive_timeout;
```
解释时要带上 GUC context。 当前源码中：
```text
wal_keep_size: PGC_SIGHUP，可以 reload。 max_slot_wal_keep_size: PGC_SIGHUP，可以 reload。 archive_command / archive_library: PGC_SIGHUP，可以 reload。 archive_mode: PGC_POSTMASTER，需要重启。 max_replication_slots / max_wal_senders: PGC_POSTMASTER，需要重启。 wal_level: PGC_POSTMASTER，需要重启。
```
这决定了现场修复速度。 例如 `max_slot_wal_keep_size` 可以 reload 后等 checkpoint 生效。 但 `max_replication_slots` 不能在线增加。

### 11.2 看 slot 是否是 retention owner

核心查询：
```sql
SELECT
  slot_name,
  slot_type,
  active,
  active_pid,
  restart_lsn,
  confirmed_flush_lsn,
  wal_status,
  safe_wal_size,
  inactive_since,
  invalidation_reason
FROM pg_replication_slots
ORDER BY restart_lsn NULLS LAST;
```
判断顺序：
```text
restart_lsn 最老的是主要 WAL retention suspect。 wal_status = extended 说明它正在把 WAL 保留扩展到 max_wal_size 之外。 wal_status = unreserved 说明它已经到了下一次 checkpoint 可能丢的边界。 wal_status = lost 说明已经不能靠继续等待追上。 safe_wal_size 接近 0 说明 max_slot_wal_keep_size 边界很近。
```
计算 slot 落后字节：
```sql
SELECT
  slot_name,
  pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
```
对 logical slot，还要看：
```sql
SELECT
  slot_name,
  catalog_xmin,
  xmin,
  confirmed_flush_lsn,
  restart_lsn
FROM pg_replication_slots
WHERE slot_type = 'logical';
```
`confirmed_flush_lsn` 很新但 `restart_lsn` 很旧，说明 decoding 仍需要较老 WAL。 不要只看 confirmed flush。

### 11.3 看 walsender 当前进度

```sql
SELECT
  pid,
  application_name,
  state,
  sent_lsn,
  write_lsn,
  flush_lsn,
  replay_lsn,
  write_lag,
  flush_lag,
  replay_lag,
  sync_state,
  reply_time
FROM pg_stat_replication
ORDER BY replay_lsn NULLS FIRST;
```
`pg_stat_replication` 来自 `system_views.sql` 中对 `pg_stat_get_wal_senders()` 的 join。 它显示的是 walsender shared state。 它不是 slot state。 常见组合：
```text
slot active = true 且 pg_stat_replication 有对应 pid: 消费者在线，但可能落后。 slot active = false: 没有进程持有，restart_lsn 不会靠 streaming 自动前进。 pg_stat_replication flush_lsn 前进，但 slot restart_lsn 不前进: 需要确认是否使用的是同一个 slot，或是 logical/physical 语义误读。
```

### 11.4 看 archive 是否阻止删除

```sql
SELECT * FROM pg_stat_archiver;
```
重点看：
```text
archived_count last_archived_wal last_archived_time failed_count last_failed_wal last_failed_time
```
还可以从文件系统看：
```bash
find "$PGDATA/pg_wal/archive_status" -name '*.ready' | wc -l
find "$PGDATA/pg_wal/archive_status" -name '*.done' | wc -l
```
`.ready` 大量堆积通常说明 archiver 未完成。 如果 slot 视图正常但 `.ready` 堆积，问题在 archive path。 如果 `.ready` 不多但某个 slot `restart_lsn` 很老，问题在 slot consumer。

### 11.5 看 WAL 生成速度和窗口时间

`safe_wal_size` 是字节，不是时间。 要转成时间，需要估算 WAL 生成速率。 简单采样：
```sql
SELECT now(), pg_current_wal_lsn();
```
隔一段时间再执行：
```sql
SELECT
  pg_size_pretty(pg_wal_lsn_diff('LSN2'::pg_lsn, 'LSN1'::pg_lsn)) AS wal_generated;
```
然后估算：
```text
time_to_failure ~= safe_wal_size / wal_bytes_per_second
```
这个估算依赖 workload。 批量导入、CREATE INDEX、VACUUM、checkpoint full-page writes、logical decoding spill 都会改变 WAL 速率。 不要把过去 5 分钟速率当成全天承诺。

### 11.6 看 pg_wal 本地实际占用

```bash
du -sh "$PGDATA/pg_wal"
ls -1 "$PGDATA/pg_wal" | grep -E '^[0-9A-F]{24}(\\.partial)?$' | wc -l
```
把 segment 数量乘以 `wal_segment_size` 可以得到大致 WAL 文件占用。 SQL 侧查看 segment size：
```sql
SHOW wal_segment_size;
```
注意 `pg_wal` 里还有：
```text
archive_status summaries timeline history files partial files
```
所以 `du` 不是纯 WAL segment 数量。

### 11.7 错误日志定位

常见日志和含义：
```text
requested WAL segment ... has already been removed walsender 或 WAL reader 需要的 segment 已不在 pg_wal。 can no longer access replication slot ... This replication slot has been invalidated due to "wal_removed". slot 已被 invalidated。 base backup done, waiting for required WAL segments to be archived base backup 正在等 archive 完成 required WAL。 still waiting for all required WAL segments to be archived archive command/library 可能失败或太慢。 archive command failed archive backlog 可能阻止 RemoveOldXlogFiles 删除。
```
日志能告诉你 failure boundary。 它不能自动告诉你业务上应该保消费者还是保 primary。 这个决策来自 RPO、RTO、磁盘余量、是否有可用 base backup、archive retention 和复制拓扑。

## 12. 常见误区

误区一：`wal_keep_size` 能保证 standby 一定追上。 实际它只保留最近固定大小的 WAL。 如果断线时间乘以 WAL 生成速率超过这个窗口，standby 仍会缺 WAL。 误区二：replication slot 永远安全。 slot 安全的是消费者追赶能力。 如果没有 `max_slot_wal_keep_size` 和磁盘监控，slot 可以把 primary 的 `pg_wal` 撑满。 误区三：`max_slot_wal_keep_size` 会让复制变慢或限流。 实际它不在 WAL 生成 hot path 上限流。 它在 checkpoint 清理阶段限制 slot 能保护多老的 WAL。 超过后是 slot 失效，不是自动 backpressure。 误区四：archive 成功等于 standby 已追上。 archive 只说明 segment 已交给外部归档。 standby flush/apply 进度要看 `pg_stat_replication` 或 standby 自身。 误区五：`safe_wal_size` 是还能安全运行多久。 实际它是字节差值。 时间取决于未来 WAL 生成速度和 checkpoint 行为。 误区六：base backup 文件在就一定能恢复。 恢复还需要从 backup startpoint 到目标点的 WAL 链和 timeline history。 这些通常由 archive retention 保证。 误区七：`pg_wal` 大于 `max_wal_size` 就是配置失效。 `max_wal_size` 不是硬上限。 slot、archive、wal_keep_size、summarization、checkpoint 时机都会让 `pg_wal` 超过它。 误区八：logical slot 的 `confirmed_flush_lsn` 新，就说明 WAL 可以删。 logical decoding 可能仍需要从更老的 `restart_lsn` 重读。 WAL retention 看 `restart_lsn`。

## 13. 课堂实验

### 实验 1：只靠 `wal_keep_size` 的断线风险

目标：观察没有 slot 时，standby 断线后只能依赖固定 WAL 窗口或 archive。 准备：
```text
primary: wal_level = replica max_wal_senders > 0 wal_keep_size = 64MB archive_mode = off standby: 不使用 primary_slot_name
```
步骤：
```bash

# 停 standby

pg_ctl -D "$STANDBY" stop

# 在 primary 生成超过 wal_keep_size 的 WAL

pgbench -i -s 50 postgres
pgbench -c 8 -j 8 -T 120 postgres

# 强制 checkpoint

psql -d postgres -c "CHECKPOINT"

# 启 standby

pg_ctl -D "$STANDBY" start
```
观察：
```text
standby 或 primary 日志中可能出现 requested WAL segment ... has already been removed。
```
回到源码：
```text
KeepLogSeg() 只按 wal_keep_size 保留最近固定窗口。 WalSndSegmentOpen() 找不到旧 segment 后报错。
```
如果启用 archive 并保留完整 WAL，再重复实验，standby 可能通过 restore_command 先补旧 WAL。 这说明 archive 和 wal_keep_size 的保护对象不同。

### 实验 2：slot 从 `extended` 到 `lost`

目标：观察 `max_slot_wal_keep_size` 的失败边界。 准备：
```text
primary: max_replication_slots > 0 max_slot_wal_keep_size = 128MB wal_keep_size = 0 archive_mode = off
```
创建 physical slot：
```sql
SELECT * FROM pg_create_physical_replication_slot('lab_phys');
```
让 standby 使用：
```text
primary_slot_name = 'lab_phys'
```
停 standby 后，在 primary 生成 WAL：
```bash
pgbench -c 8 -j 8 -T 180 postgres
```
周期观察：
```sql
SELECT
  slot_name,
  active,
  restart_lsn,
  wal_status,
  pg_size_pretty(safe_wal_size) AS safe,
  invalidation_reason
FROM pg_replication_slots
WHERE slot_name = 'lab_phys';
```
强制 checkpoint：
```sql
CHECKPOINT;
```
预期：
```text
safe_wal_size 下降； wal_status 可能进入 unreserved； checkpoint 后 slot 可能 lost，invalidation_reason = wal_removed。
```
回到源码：
```text
KeepLogSeg() 用 max_slot_wal_keep_size 裁剪 slot 保留距离。 InvalidateObsoleteReplicationSlots() 把 restart_lsn < oldestLSN 的 slot invalidated。
```

### 实验 3：archive backlog 阻止本地删除

目标：区分 archive backlog 和 slot retention。 准备一个测试实例，不要在生产库执行。 配置：
```text
archive_mode = on archive_command = 'false' wal_keep_size = 0 max_slot_wal_keep_size = 64MB
```
生成 WAL 并 checkpoint：
```bash
pgbench -c 8 -j 8 -T 60 postgres
psql -d postgres -c "CHECKPOINT"
```
观察：
```sql
SELECT * FROM pg_stat_archiver;
```
文件系统：
```bash
find "$PGDATA/pg_wal/archive_status" -name '*.ready' | wc -l
du -sh "$PGDATA/pg_wal"
```
回到源码：
```text
XLogArchiveCheckDone() 没看到 .done 时返回 false。 RemoveOldXlogFiles() 因此不会 RemoveXlogFile()。
```
把 `archive_command` 修好并 reload 后，archiver 完成 `.ready -> .done`，下一轮 checkpoint 才能清理旧 segment。

### 实验 4：base backup 的 WAL 链依赖

目标：确认 base backup 不只是文件拷贝。 执行一个普通 base backup，并观察 `backup_label`。
```bash
pg_basebackup -D /tmp/pgbb-lab -X none -Fp -v
```
打开备份目录中的 `backup_label`，记录：
```text
START WAL LOCATION CHECKPOINT LOCATION START TIMELINE
```
然后在 primary 看 archive 或 pg_wal 是否仍有对应 WAL。 源码对应：
```text
do_pg_backup_start() 取 checkpoint redo 作为 startpoint。 build_backup_content() 写 backup_label。 do_pg_backup_stop() 可等待 archive。 sendDir() 默认不递归备份 pg_wal。
```
讨论：
```text
如果没有 -X stream / -X fetch，也没有 archive retention， 这个 base backup 是否可恢复？
```
正确答案取决于 required WAL 是否通过其它方式保存。

## 14. 源码练习

### 练习 1：画出 checkpoint 清理边界

在 `xlog.c` 中给这些位置设断点：
```text
CreateCheckPoint KeepLogSeg InvalidateObsoleteReplicationSlots RemoveOldXlogFiles XLogArchiveCheckDone RemoveXlogFile
```
记录每次 checkpoint 的：
```text
RedoRecPtr recptr _logSegNo before KeepLogSeg _logSegNo after KeepLogSeg 是否 invalidated slot 传给 RemoveOldXlogFiles 的 segno
```
要求画出：
```text
RedoRecPtr segment slot restart_lsn segment wal_keep_size floor max_slot_wal_keep_size cap actual remove <= segment
```

### 练习 2：验证 `wal_status` 不是持久字段

阅读：
```text
slotfuncs.c: pg_get_replication_slots() xlog.c: GetWALAvailability() system_views.sql: pg_replication_slots
```
回答：
```text
wal_status 是从哪些 runtime 状态推导出来的？ 哪些字段真正保存在 slot on-disk state？ 为什么 active process 存在时，removed 可能被显示成 unreserved？
```

### 练习 3：比较 physical 和 logical slot 的 WAL reader 起点

阅读：
```text
walsender.c: StartReplication() walsender.c: StartLogicalReplication() walsender.c: PhysicalConfirmReceivedLocation() slot.c: ReplicationSlotReserveWal()
```
回答：
```text
physical slot 的初始 restart_lsn 为什么用 GetRedoRecPtr()？ logical walsender 为什么 XLogBeginRead() 从 restart_lsn，而不是 confirmed_flush？ standby feedback 的 flush_lsn 如何推进 physical slot？
```

### 练习 4：base backup 失败清理

阅读：
```text
basebackup.c: perform_base_backup() xlog.c: do_pg_backup_start() xlog.c: do_pg_backup_stop() xlog.c: do_pg_abort_backup()
```
回答：
```text
perform_base_backup() 为什么把 do_pg_backup_start() 之后的工作放进 PG_ENSURE_ERROR_CLEANUP？ runningBackups 如果泄漏会影响什么？ 为什么 base backup 的 WAL 等待不能只靠文件拷贝成功判断？
```

## 15. 操作诊断流程

### 15.1 现场问题：primary `pg_wal` 快满

第一步，确认增长是否真的在 `pg_wal`：
```bash
du -sh "$PGDATA/pg_wal"
```
第二步，看 slot：
```sql
SELECT
  slot_name,
  slot_type,
  active,
  restart_lsn,
  wal_status,
  pg_size_pretty(safe_wal_size) AS safe,
  inactive_since,
  invalidation_reason
FROM pg_replication_slots
ORDER BY restart_lsn NULLS LAST;
```
第三步，看 archive：
```sql
SELECT * FROM pg_stat_archiver;
```
第四步，看 walsender：
```sql
SELECT
  pid,
  application_name,
  state,
  sent_lsn,
  flush_lsn,
  replay_lsn,
  reply_time
FROM pg_stat_replication;
```
第五步，判断 owner：
```text
old restart_lsn + extended: slot owner。 many .ready + archiver failures: archive owner。 no slot, no archive, huge recent WAL: workload + checkpoint + wal_keep_size window owner。 base backup waiting: backup stop 等 archive owner。
```

### 15.2 修复动作的取舍

可选动作一：修复消费者。 适合：
```text
slot 还未 lost； safe_wal_size 还有余量； standby / subscriber 可以快速恢复网络或 apply。
```
效果：
```text
consumer ack 推进 restart_lsn； ReplicationSlotsComputeRequiredLSN() 更新全局 slot minimum； 下一次 checkpoint 后旧 WAL 可清理。
```
可选动作二：提高磁盘空间或临时迁移 `pg_wal`。 适合：
```text
必须保护消费者追赶； 不能接受 slot invalidation； 短时间内能加容量。
```
风险：
```text
没有解决根因； WAL 生成继续时窗口仍会耗尽。
```
可选动作三：调低或设置 `max_slot_wal_keep_size`。 适合：
```text
primary 磁盘可用性优先； 消费者可重建； 业务接受重新初始化 standby/subscriber。
```
效果：
```text
下一次 checkpoint 清理时可能 invalidated lagging slot。
```
注意：
```text
这不是立即删除命令。 需要 checkpoint/restartpoint 清理阶段生效。
```
可选动作四：drop 不需要的 slot。 适合：
```text
确认 slot 对应消费者已经废弃。
```
效果：
```text
ReplicationSlotDropAcquired() ReplicationSlotsComputeRequiredLSN() 后续 checkpoint 可以清理旧 WAL。
```
风险：
```text
对应消费者不能继续从旧位置追赶。
```
可选动作五：修复 archive。 适合：
```text
.ready 堆积； pg_stat_archiver failed_count 增长； slot 并非主要 owner。
```
效果：
```text
archiver 把 .ready 改成 .done； 下一次 RemoveOldXlogFiles() 可以通过 XLogArchiveCheckDone()。
```
可选动作六：重做 base backup。 适合：
```text
slot 已 lost； archive 没有所需 WAL； standby 或 subscriber 无法从当前状态追赶。
```
这是牺牲消费者连续性后的恢复动作。 不要试图手工把 slot 的 `restart_lsn` 改回去。 所需 WAL 已经被移除时，改元数据不能恢复事实。

## 16. 讨论题

1. 为什么 PostgreSQL 不让 `wal_keep_size` 绑定某个 standby 身份？
2. 如果 `max_slot_wal_keep_size` 在 WAL insert hot path 上强制限流，会引入哪些 correctness 和 latency 问题？
3. `pg_replication_slots.wal_status = unreserved` 和 `lost` 的操作含义有什么不同？
4. 为什么 archive backlog 能让 `pg_wal` 超过 `max_wal_size`，即使没有任何 replication slot？
5. `last_saved_restart_lsn` 为什么可能比 `restart_lsn` 更重要？
6. 一个 logical slot 的 `confirmed_flush_lsn` 已经接近当前 LSN，但 `restart_lsn` 很旧。可能发生了什么？
7. base backup 已经成功生成 tar 文件，但 `do_pg_backup_stop()` 提示 archiving 未启用。这个备份一定可用吗？
8. standby 断线后，什么时候应该优先保留 slot，什么时候应该让 slot 失效并重建？

## 17. 本节小结

本节主链路是：
```text
WAL segment 写满 -> archive notification -> checkpoint / restartpoint -> KeepLogSeg() 合并 slot、wal_keep_size、summarization -> max_slot_wal_keep_size 裁剪 slot retention -> InvalidateObsoleteReplicationSlots() 标记越界 slot -> RemoveOldXlogFiles() 在 archive done 后删除或回收
```
核心状态是：
```text
slot.restart_lsn: 消费者仍可能需要的最老 WAL。 XLogCtl->replicationSlotMinLSN: 所有有效 slot 汇总给 xlog 的最老 LSN。 wal_keep_size_mb: 无身份的最近 WAL 保留地板。 max_slot_wal_keep_size_mb: slot 能把保留边界往回拉的上限。 archive_status .ready / .done: segment 是否已交给外部 archive 的删除门槛。 backup_label startpoint: 一个 base backup 恢复所需 WAL 链的起点。
```
ownership / cleanup 的边界：
```text
slot 由 backend 或 walsender acquire。 persistent slot release 后仍保留资源。 process exit 会 release active owner。 checkpoint 保存 dirty slot 并更新 last_saved_restart_lsn。 slot invalidation 会持久化 invalidated 状态。 archive cleanup 由 archiver 和 XLogArchiveCheckDone() 协作。 base backup 用 PG_ENSURE_ERROR_CLEANUP 防止 runningBackups 泄漏。
```
错误路径的核心：
```text
没有 slot 或窗口不够: walsender 读旧 segment 时报 requested WAL segment removed。 slot 超过 max_slot_wal_keep_size: checkpoint invalidates slot with wal_removed。 archive failed: XLogArchiveCheckDone() 阻止本地删除，pg_wal 继续涨。 backup 缺 required WAL: backup 文件存在也可能不可恢复。
```
能直接观测的是：
```text
pg_replication_slots.restart_lsn / wal_status / safe_wal_size / invalidation_reason pg_stat_replication.sent_lsn / flush_lsn / replay_lsn pg_stat_archiver pg_stat_wal pg_wal/archive_status server log
```
不能直接从单个指标判断的是：
```text
未来还剩多少时间； archive retention 是否满足某个 PITR 目标； 一个 backup 是否完整可恢复； logical decoding 为什么还需要旧 restart_lsn； drop slot 的业务后果。
```
可迁移规律：
```text
当一个生产系统要同时服务 live consumers 和 long-term recovery 时， 日志保留必须拆成多个 owner： consumer identity、fixed recent window、external archive、backup chain。 系统不会免费同时最大化所有 owner 的可用性。 一旦共享日志空间接近耗尽，必须显式选择： 保消费者追赶，还是保 primary 继续写入。
```
在 PostgreSQL 中，这个选择最终落到 checkpoint 清理阶段。 `KeepLogSeg()` 和 `InvalidateObsoleteReplicationSlots()` 是源码层面的 tradeoff 核心。 `pg_replication_slots` 和 `pg_stat_archiver` 只是把这个 tradeoff 的当前结果暴露给操作者。
