# PostgreSQL Slot 监控与保留风险诊断
## 课程定位
前置知识：已经理解 replication slot 的类型、生命周期、`restart_lsn` 的 WAL 保留语义，以及 `xmin` / `catalog_xmin` 如何影响 VACUUM 清理边界。
本节唯一主问题：
```text
如何从 pg_replication_slots、WAL 目录增长、inactive slot、restart_lsn 滞后和 vacuum bloat
判断问题来自消费者停滞、反馈缺失还是保留策略过宽？
```
核心矛盾：slot 视图暴露的是多个模块在同一时刻留下的影子，而生产故障里的症状往往是混合的。
WAL 目录增长可能来自消费者真的停了，也可能只是 checkpoint 还没回收，或者 `max_slot_wal_keep_size = -1` 让系统继续保护一个业务上已经废弃的 slot。
VACUUM bloat 可能来自 physical standby feedback，也可能来自 logical slot 的 `catalog_xmin`，还可能完全不是 slot，而是普通长事务或 prepared transaction。
诊断必须把每个观测字段放回源码里的状态推进链，而不是把单个列当成因果。
学完后应能判断：
```text
pg_replication_slots 中哪些字段来自 slot shared memory，哪些来自 xlog 模块即时计算；
为什么 restart_lsn 滞后解释 WAL 保留，却不能单独解释 vacuum bloat；
为什么 confirmed_flush_lsn 滞后更像 logical consumer ack 问题；
为什么 inactive slot 可能是真正停滞，也可能只是刚重启后的持久 slot；
为什么 physical standby feedback 缺失通常不会导致主库 bloat，反而会让 standby 查询更容易被取消；
为什么 wal_status = unreserved 是 checkpoint 前的危险边界，不是已经损坏；
为什么 safe_wal_size 为 NULL 有时是无限保护，有时是已经 lost；
如何把消费者停滞、反馈缺失和保留策略过宽分层排查。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面几节已经回答：
```text
slot 是什么；
restart_lsn 如何保护 WAL；
xmin / catalog_xmin 如何保护 tuple 和 catalog 版本；
slot 如何持久化、失效和删除。
```
这一节不再重复 slot 生命周期。
它从运维现场进入：
```text
pg_wal 持续增长；
某个 slot active = false；
restart_lsn 比 pg_current_wal_lsn 落后很多；
logical slot 的 confirmed_flush_lsn 不动；
主库表膨胀，VACUUM 清不掉 dead tuple；
wal_status 从 reserved 变成 extended / unreserved / lost。
```
这些现象看起来都指向 replication slot。
但它们不在同一条因果链上。
最常见的误判是：
```text
看到 pg_wal 增长，就认为 consumer 停了；
看到 inactive_since 很久，就直接 drop slot；
看到 xmin 不为空，就认为 slot 一定导致所有表 bloat；
看到 confirmed_flush_lsn 不动，就认为 WAL sender 没有发送数据；
看到 wal_status = reserved，就认为没有风险；
看到 safe_wal_size = NULL，就认为没有上限。
```
本节的目标是建立一条线性判断链：
```text
问题现象
  -> 当前状态来自哪个源码字段
  -> 状态由哪条主流程推进
  -> 哪些边界会让字段滞后或不可见
  -> 异常路径如何改变可观测结果
  -> 最后判断根因属于消费者、反馈还是策略
```
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
pg_replication_slots 先从 slot shared memory 拷贝持久字段和运行态字段，
再用 xlog.c 的 GetWALAvailability() 按当前 WAL 写入位置、checkpoint 删除边界、
wal_keep_size 和 max_slot_wal_keep_size 计算 wal_status / safe_wal_size；
walsender 根据 physical 或 logical feedback 推进 restart_lsn、confirmed_flush、xmin；
checkpoint 才真正让 WAL segment 删除或 recycling 可见；
VACUUM 通过 ProcArray 中的 slot xmin horizon 间接受影响。
```
这一节的 tension 是：
```text
slot 必须把消费者的最低需求暴露给系统清理路径
  vs
监控视图只能看到若干时间点的投影，不能直接给出根因
```
诊断时要把状态拆成三层。
第一层是 WAL 保留：
```text
restart_lsn
  -> ReplicationSlotsComputeRequiredLSN()
  -> XLogSetReplicationSlotMinimumLSN()
  -> KeepLogSeg()
  -> checkpoint / restartpoint 后 RemoveOldXlogFiles()
```
第二层是消费确认：
```text
logical consumer flush ack
  -> ProcessStandbyReplyMessage()
  -> LogicalConfirmReceivedLocation()
  -> confirmed_flush_lsn 可能前进
  -> candidate restart_lsn / catalog_xmin 才可能应用
```
第三层是 VACUUM horizon：
```text
slot effective_xmin / effective_catalog_xmin
  -> ReplicationSlotsComputeRequiredXmin()
  -> ProcArraySetReplicationSlotXmin()
  -> ComputeXidHorizons() / GlobalVis*
  -> VACUUM 判断 tuple 是否可移除
```
这三层可以同时滞后，也可以只有一层滞后。
例如：
```text
physical standby 断线但 hot_standby_feedback 关闭:
  restart_lsn 滞后，pg_wal 可能增长；
  xmin / catalog_xmin 为空，主库 bloat 不应归因于 slot feedback。
logical consumer 活着但不 ack flush:
  active = true；
  active_pid 有 walsender；
  sent_lsn 可能前进；
  confirmed_flush_lsn 不动；
  restart_lsn 和 catalog_xmin 可能长期不能推进。
保留策略过宽:
  consumer 确实已经不需要这个 slot；
  但 persistent slot 仍存在；
  max_slot_wal_keep_size = -1 或设置过大；
  系统按配置继续保护旧 restart_lsn。
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/catalog/system_views.sql` | `pg_replication_slots` view 如何包装 `pg_get_replication_slots()` 并补上 database 名称。 |
| 2 | `src/backend/replication/slotfuncs.c` | `pg_get_replication_slots()` 如何拷贝 slot、计算 `wal_status`、`safe_wal_size`、`inactive_since`、`invalidation_reason`。 |
| 3 | `src/include/replication/slot.h` | `ReplicationSlotPersistentData`、`ReplicationSlot`、`ReplicationSlotInvalidationCause` 的字段边界。 |
| 4 | `src/backend/replication/slot.c` | `ReplicationSlotsComputeRequiredLSN()`、`ReplicationSlotsComputeRequiredXmin()`、`ReplicationSlotRelease()`、`InvalidateObsoleteReplicationSlots()`、`CheckPointReplicationSlots()`。 |
| 5 | `src/backend/access/transam/xlog.c` | `GetWALAvailability()`、`KeepLogSeg()`、checkpoint 后 `RemoveOldXlogFiles()`。 |
| 6 | `src/backend/replication/walsender.c` | `ProcessStandbyReplyMessage()`、`PhysicalConfirmReceivedLocation()`、`PhysicalReplicationSlotNewXmin()`、`ProcessStandbyHSFeedbackMessage()`。 |
| 7 | `src/backend/replication/walreceiver.c` | `XLogWalRcvSendHSFeedback()` 如何发送 standby xmin / catalog_xmin。 |
| 8 | `src/backend/replication/logical/logical.c` | `LogicalConfirmReceivedLocation()` 如何推进 `confirmed_flush`、候选 `restart_lsn` 和 `catalog_xmin`。 |
| 9 | `src/backend/utils/adt/pgstatfuncs.c` | `pg_stat_wal` 与 `pg_stat_replication_slots` 的统计粒度。 |
| 10 | `src/backend/access/transam/xlogfuncs.c` 和 `src/backend/utils/adt/genfile.c` | `pg_current_wal_lsn()`、`pg_walfile_name()`、`pg_ls_waldir()` 的观测边界。 |
推荐阅读顺序不是横向背字段。
先读视图入口，确认每列来自哪里。
再沿两条主链路往下：
```text
WAL 空间:
  restart_lsn
  -> required LSN
  -> KeepLogSeg
  -> checkpoint 删除或 recycling
VACUUM bloat:
  xmin / catalog_xmin
  -> required xmin
  -> ProcArray horizon
  -> VACUUM cutoffs / GlobalVis
```
最后回到 walsender / walreceiver：
```text
消费者到底有没有发送 flush ack？
standby 到底有没有发送 Hot Standby feedback？
slot 是 active 但无反馈，还是 inactive 根本没人持有？
```
## 4. `pg_replication_slots` 不是一个单一真相
`system_views.sql` 中的视图形态是：
```text
CREATE VIEW pg_replication_slots AS
  SELECT L.slot_name, L.plugin, L.slot_type, L.datoid,
         D.datname AS database, ...
  FROM pg_get_replication_slots() AS L
       LEFT JOIN pg_database D ON (L.datoid = D.oid);
```
真正采集状态的是 `slotfuncs.c` 的 `pg_get_replication_slots()`。
它做的第一件关键事情是：
```text
currlsn = GetXLogWriteRecPtr()
ReplicationSlotControlLock shared
  for each in-use slot:
    SpinLockAcquire(&slot->mutex)
    slot_contents = *slot
    SpinLockRelease(&slot->mutex)
    用 slot_contents 组装一行
```
这意味着：
```text
slot 字段是一瞬间的 shared memory 拷贝；
wal_status 又用当前 WAL 写入位置即时计算；
active_pid 可能在函数返回前就已经变化；
视图不是一个跨所有 slot、WAL 删除状态和 walsender 状态的事务级一致快照。
```
诊断时不能只看一列。
要把列分成四类。
第一类来自 persistent data：
```text
slot_name
plugin
slot_type
datoid
temporary
xmin
catalog_xmin
restart_lsn
confirmed_flush_lsn
two_phase
two_phase_at
invalidation_reason
failover
synced
```
这些字段主要来自 `slot_contents.data`。
它们描述 slot 要保护什么，或者已经失效的原因。
第二类来自 shared memory 运行态：
```text
active
active_pid
inactive_since
slotsync_skip_reason
```
`active` 是：
```text
slot_contents.active_proc != INVALID_PROC_NUMBER
```
`active_pid` 是：
```text
GetPGProcByNumber(active_proc)->pid
```
`active_proc` 不是 pid，而是 `ProcNumber`。
所以 `active_pid` 是诊断入口，不是 owner 语义本身。
第三类来自 xlog 模块即时计算：
```text
wal_status
safe_wal_size
```
`pg_get_replication_slots()` 对有效 `restart_lsn` 调用：
```text
GetWALAvailability(slot_contents.data.restart_lsn)
```
`safe_wal_size` 还会读取：
```text
max_slot_wal_keep_size_mb
wal_keep_size_mb
currlsn = GetXLogWriteRecPtr()
wal_segment_size
```
所以它不是 slot 内部保存的字段。
第四类是派生语义：
```text
conflicting
```
当前源码中，physical slot 的 `conflicting` 为 NULL。
logical slot 只有在 invalidation cause 是：
```text
RS_INVAL_HORIZON
RS_INVAL_WAL_LEVEL
```
时才显示 true。
这对应视图里的：
```text
rows_removed
wal_level_insufficient
```
## 5. `wal_status` 和 `safe_wal_size` 的真实含义
`wal_status` 的字符串来自 `GetWALAvailability()` 的枚举：
```text
reserved
extended
unreserved
lost
NULL
```
源码注释给出的语义更接近：
```text
reserved:
  target LSN 对应 segment 仍在 max_wal_size 期望范围内。
extended:
  target LSN 仍可用，但主要因为 slot 或 wal_keep_size 把 WAL 保留到 max_wal_size 之外。
unreserved:
  已经不再被保留，但文件还没删除；下一次 checkpoint 可能删除。
lost:
  需要的 WAL 已经移除，使用该 LSN 的复制流无法继续。
NULL:
  restart_lsn 无效，slot 还没有真正开始保留 WAL，或状态已不适合按 WAL availability 解释。
```
`GetWALAvailability()` 内部先拿当前写位置：
```text
currpos = GetXLogWriteRecPtr()
```
再计算：
```text
oldestSlotSeg:
  KeepLogSeg(currpos, &oldestSlotSeg)
oldestSeg:
  XLogGetLastRemovedSegno() + 1
oldestSegMaxWalSize:
  currSeg - ConvertToXSegs(max_wal_size_mb) - 1 的近似边界
```
然后判断 target segment 是否：
```text
仍被 slot / wal_keep_size 保留；
仍在 max_wal_size 范围内；
已经不保留但尚未删除；
已经早于 last removed segment。
```
所以：
```text
wal_status 不是复制延迟；
它是 restart_lsn 对应 WAL segment 的可用性状态。
```
`safe_wal_size` 的计算在 `slotfuncs.c`。
只有两个条件同时满足才计算：
```text
slot 没有被判断为 lost；
max_slot_wal_keep_size_mb >= 0。
```
如果 `max_slot_wal_keep_size = -1`，`safe_wal_size` 是 NULL。
这不是“安全空间无限大”的可观测证明。
它表示源码没有配置上限，因此视图不计算这个剩余量。
计算思路是：
```text
targetSeg = segment(restart_lsn)
slotKeepSegs = max_slot_wal_keep_size in segments
keepSegs = wal_keep_size in segments
failSeg = targetSeg + Max(slotKeepSegs, keepSegs) + 1
failLSN = start of failSeg
safe_wal_size = failLSN - currlsn
```
注意这里用了 `Max(slotKeepSegs, keepSegs)`。
这解释一个容易忽视的现象：
```text
即使 max_slot_wal_keep_size 较小，
wal_keep_size 设得很大也会让 safe_wal_size 看起来更宽。
```
同时，`safe_wal_size` 是按当前写位置估算的余量。
它没有预测未来 WAL 生成速率，也不知道下次 checkpoint 何时删除。
如果系统每分钟生成 50GB WAL，一个看似很大的余量也可能很快耗尽。
## 6. `restart_lsn`、`confirmed_flush_lsn`、`xmin` 和 `catalog_xmin`
诊断 slot 保留风险，首先要把四个 LSN / XID 字段拆开。
`restart_lsn` 的问题是：
```text
如果从这里之前的 WAL 被删除，这个 slot 可能无法继续读取所需历史。
```
它直接进入 WAL 保留链：
```text
ReplicationSlotsComputeRequiredLSN()
  -> 找所有有效 slot 中最小 restart_lsn
  -> persistent slot 还会考虑 last_saved_restart_lsn
  -> XLogSetReplicationSlotMinimumLSN(min_required)
```
所以 `restart_lsn` 滞后主要解释：
```text
pg_wal 为什么不能回收；
wal_status 为什么进入 extended / unreserved；
max_slot_wal_keep_size 为什么可能 invalidation。
```
`confirmed_flush_lsn` 的问题是：
```text
logical consumer 已确认安全收到到哪里。
```
它主要存在于 logical slot。
`walsender.c` 的 `ProcessStandbyReplyMessage()` 收到 flush LSN 后：
```text
if logical slot:
  LogicalConfirmReceivedLocation(flushPtr)
else:
  PhysicalConfirmReceivedLocation(flushPtr)
```
`LogicalConfirmReceivedLocation()` 先防止 `confirmed_flush` 后退：
```text
if lsn > data.confirmed_flush:
  data.confirmed_flush = lsn
```
然后只有在 consumer 确认位置越过候选边界时，才可能推进：
```text
candidate_catalog_xmin
candidate_restart_lsn
```
因此：
```text
confirmed_flush_lsn 不动:
  更像 logical consumer 没有发送 flush ack，
  或者 walsender 没收到 / 没处理 ack。
confirmed_flush_lsn 前进但 restart_lsn 不动:
  更像 decoder 仍需要旧 WAL，
  常见原因是长事务、大事务、prepared transaction、reorder buffer 中还有无法释放的历史。
```
`xmin` 的问题是：
```text
哪些普通数据 tuple 版本不能被 VACUUM 认为对所有人都无用。
```
`catalog_xmin` 的问题是：
```text
哪些 catalog tuple 版本不能被删除，因为 logical decoding 还可能用旧 schema 解释 WAL。
```
它们进入清理边界的链路是：
```text
ReplicationSlotsComputeRequiredXmin()
  -> 聚合所有有效 slot 的 effective_xmin / effective_catalog_xmin
  -> ProcArraySetReplicationSlotXmin()
  -> procArray->replication_slot_xmin
  -> procArray->replication_slot_catalog_xmin
```
VACUUM 侧不是直接扫 slot。
`vacuum_get_cutoffs()` 调用：
```text
GetOldestNonRemovableTransactionId(rel)
```
可见性判断还会通过：
```text
GlobalVisTestFor(rel)
GlobalVisTestIsRemovableXid()
ComputeXidHorizons()
```
这就是为什么：
```text
restart_lsn 很旧，能解释 WAL 保留；
xmin / catalog_xmin 很旧，才能解释 slot 相关 bloat；
两者可以同时旧，也可以只有一个旧。
```
## 7. 主流程源码 walkthrough：从一条反馈到两个保留边界
### 7.1 physical standby 的普通状态反馈
standby 的 walreceiver 周期性发送 status update。
primary 上 `walsender.c` 的 `ProcessStandbyReplyMessage()` 解析：
```text
writePtr
flushPtr
applyPtr
replyTime
replyRequested
```
它先更新 `WalSnd` shared state：
```text
walsnd->write = writePtr
walsnd->flush = flushPtr
walsnd->apply = applyPtr
walsnd->replyTime = replyTime
```
这就是 `pg_stat_replication` 中：
```text
write_lsn
flush_lsn
replay_lsn
reply_time
```
的主要来源。
如果当前 walsender 持有 physical slot，且 flushPtr 有效：
```text
PhysicalConfirmReceivedLocation(flushPtr)
```
会把：
```text
slot->data.restart_lsn = flushPtr
ReplicationSlotMarkDirty()
ReplicationSlotsComputeRequiredLSN()
```
这里没有立即 `ReplicationSlotSave()`。
源码注释解释得很直接：
```text
physical slot 的这类落盘滞后最坏只会让系统更保守地保留 WAL。
```
因此 physical slot 诊断顺序是：
```text
pg_stat_replication.flush_lsn 是否推进
  -> 对应 slot.restart_lsn 是否推进
  -> pg_wal 是否仍被这个 restart_lsn 钉住
```
如果 `flush_lsn` 不动，问题通常在 standby 接收、写盘、网络或反馈链路。
如果 `flush_lsn` 推进但 slot `restart_lsn` 不推进，要先确认这个 walsender 是否真的使用这个 slot。
### 7.2 physical standby 的 Hot Standby feedback
standby 另一路反馈是 Hot Standby feedback。
`walreceiver.c` 的 `XLogWalRcvSendHSFeedback()` 只在合适条件下发送：
```text
wal_receiver_status_interval > 0
hot_standby_feedback = on
HotStandbyActive()
```
它调用：
```text
GetReplicationHorizons(&xmin, &catalog_xmin)
```
然后发送：
```text
PqReplMsg_HotStandbyFeedback
xmin
xmin_epoch
catalog_xmin
catalog_xmin_epoch
```
primary 上 `ProcessStandbyHSFeedbackMessage()` 收到后：
```text
如果 feedbackXmin 和 feedbackCatalogXmin 都无效:
  MyProc->xmin = InvalidTransactionId
  如果有 slot，PhysicalReplicationSlotNewXmin(invalid, invalid)
  return
如果 xid epoch 不合理:
  忽略
如果有 replication slot:
  PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)
else:
  MyProc->xmin = min(feedbackXmin, feedbackCatalogXmin)
```
`PhysicalReplicationSlotNewXmin()` 更新：
```text
slot->data.xmin
slot->effective_xmin
slot->data.catalog_xmin
slot->effective_catalog_xmin
ReplicationSlotsComputeRequiredXmin(false)
```
这个分支解释一个非常重要的诊断边界：
```text
使用 physical slot 时，standby feedback 的保留边界可能在 slot.xmin / catalog_xmin；
不使用 slot 时，保留边界可能在 walsender 的 MyProc->xmin。
```
所以 `pg_stat_replication.backend_xmin` 为 NULL 不一定说明没有 feedback 风险。
如果使用 slot，要看 `pg_replication_slots.xmin` 和 `catalog_xmin`。
反过来，如果 `hot_standby_feedback` 关闭：
```text
primary 不会为了 standby 查询保留旧 tuple；
主库 bloat 通常不会由这个 feedback 缺失导致；
代价是 standby 查询更容易因为 recovery conflict 被取消。
```
这和很多现场直觉相反。
feedback 缺失保护的是 standby 查询的失败风险，不是 primary 的 bloat 风险。
### 7.3 logical consumer 的 flush ack
logical replication protocol 也走 `ProcessStandbyReplyMessage()`。
但 flushPtr 的含义不同。
对 logical slot：
```text
flushPtr 表示 consumer 已经确认安全接收到的 logical stream 位置。
```
`LogicalConfirmReceivedLocation(flushPtr)` 会：
```text
推进 confirmed_flush_lsn；
如果确认位置越过 candidate_xmin_lsn，保存新的 catalog_xmin；
如果确认位置越过 candidate_restart_valid，保存新的 restart_lsn；
保存成功后再放宽 effective_catalog_xmin；
最后 recompute required xmin / LSN。
```
这里的顺序服务 crash safety：
```text
先把更靠后的 catalog_xmin / restart_lsn 写入 slot state；
再让 VACUUM 或 WAL removal 使用更宽松的边界。
```
诊断上要分两种慢：
```text
confirmed_flush_lsn 慢:
  consumer 没有确认，或者确认没到达 primary。
restart_lsn 慢但 confirmed_flush_lsn 快:
  decoder 仍需要更早 WAL。
  consumer 可能健康，但 workload 中有长事务、大事务或 2PC 让 restart_lsn 不能前推。
```
## 8. checkpoint 与 pg_wal 增长：不是写完就删
WAL 文件删除主要在 checkpoint / restartpoint 后发生。
普通 checkpoint 的相关顺序在 `xlog.c`：
```text
CreateCheckPoint()
  -> CheckPointGuts()
  -> END_CRIT_SECTION()
  -> KeepLogSeg(recptr, &_logSegNo)
  -> InvalidateObsoleteReplicationSlots(RS_INVAL_WAL_REMOVED | RS_INVAL_IDLE_TIMEOUT, ...)
  -> 如果 invalidated，重新 KeepLogSeg()
  -> _logSegNo--
  -> RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr, timeline)
  -> PreallocXlogFiles()
```
`KeepLogSeg()` 会综合：
```text
XLogGetReplicationSlotMinimumLSN()
max_slot_wal_keep_size
wal_keep_size
oldest unsummarized WAL
```
然后把可删除边界往回退。
`RemoveOldXlogFiles()` 扫描 `pg_wal`：
```text
只处理 WAL segment 或 .partial；
只删除或 recycle 早于边界的文件；
还要满足 archive status 已完成；
文件名比较忽略 timeline 前 8 个字符，以免误删父 timeline 上仍需的 WAL。
```
所以看到 `pg_wal` 增长时，要先问：
```text
增长是持续跨 checkpoint，还是 checkpoint 前的正常累积？
归档是否阻止删除？
wal_keep_size 是否主动要求保留？
WAL summarization 是否也在保留旧 WAL？
slot restart_lsn 是否是最老边界？
```
`max_wal_size` 也不是硬上限。
它影响 checkpoint 触发和目标保留范围。
当 slot 或 `wal_keep_size` 需要更多 WAL 时，`pg_wal` 可以超过 `max_wal_size`。
这就是 `wal_status = extended` 的核心含义：
```text
WAL 还在，但已经不是 max_wal_size 常规范围内的保留。
```
`wal_status = unreserved` 更危险：
```text
当前文件可能还在；
但源码已经认为它不再受保留策略保护；
下一次 checkpoint 可能把它删除。
```
诊断时不要把 `unreserved` 当作“还好文件没删”。
它是“消费者必须立刻追上或调整策略”的边界。
## 9. inactive slot 的三种解释
`active = false` 来自：
```text
slot->active_proc == INVALID_PROC_NUMBER
```
`inactive_since` 的设置位置主要有三个。
acquire 时清零：
```text
ReplicationSlotAcquire()
  -> active_proc = MyProcNumber
  -> ReplicationSlotSetInactiveSince(s, 0, false)
```
release persistent slot 时设置当前时间：
```text
ReplicationSlotRelease()
  -> active_proc = INVALID_PROC_NUMBER
  -> ReplicationSlotSetInactiveSince(slot, now, false)
```
startup 恢复 persistent slot 时也设置当前时间：
```text
RestoreSlotFromDisk()
  -> active_proc = INVALID_PROC_NUMBER
  -> ReplicationSlotSetInactiveSince(slot, now, false)
```
因此 `inactive_since` 不是“consumer 第一次坏掉的时间”。
它更准确地表示：
```text
当前这次内存生命周期里，slot 从何时起没有 owner。
```
三种解释必须分开。
第一种是真消费者停滞：
```text
active = false；
inactive_since 很久；
restart_lsn 很旧；
wal_status = extended / unreserved；
业务上这个 slot 应该一直被消费。
```
这时问题接近：
```text
consumer 没有连接；
复制任务、subscription、pg_recvlogical、CDC 服务或 standby 停了。
```
第二种是计划内空闲：
```text
slot 是用于定时解码、手工导出、迁移窗口或备用消费者；
inactive_since 很久；
但业务策略就是让它跨天保留；
max_slot_wal_keep_size 也按这个 RPO 设计。
```
这不是消费者 bug。
它是保留策略在消耗磁盘。
如果磁盘不可接受，根因是策略过宽，不是 PostgreSQL 没回收。
第三种是重启后的假新鲜：
```text
数据库刚重启；
persistent slot 从磁盘恢复；
inactive_since 变成本次 startup 附近的时间；
但 restart_lsn 可能已经落后很久。
```
这时要看 `restart_lsn` 和历史监控，而不是只看 `inactive_since`。
`inactive_since` 只能说明当前进程生命周期内的 inactive 起点。
## 10. 三类根因的判定模型
### 10.1 消费者停滞
消费者停滞的核心特征是：
```text
slot 保护边界需要推进，但推进它的消费者状态没有前进。
```
physical slot 场景：
```text
pg_replication_slots.active = false
  或 active_pid 对应 walsender 存在但 pg_stat_replication.flush_lsn 不动；
restart_lsn 与 pg_current_wal_lsn 差距持续扩大；
reply_time 变旧或 flush_lsn 长期低于 sent_lsn；
wal_status 逐步从 reserved 到 extended / unreserved。
```
logical slot 场景：
```text
confirmed_flush_lsn 不动；
active = false 或 active_pid 的 walsender 没收到 fresh reply；
pg_stat_replication.flush_lsn 不动或 NULL；
pg_stat_replication_slots 的 spill / stream 统计可能继续增长，也可能没有变化；
restart_lsn 和 catalog_xmin 长期不前进。
```
关键区别是：
```text
physical consumer 的 flush_lsn 直接推进 restart_lsn；
logical consumer 的 flush_lsn 先推进 confirmed_flush_lsn，
restart_lsn 还受 decoder 内部事务重组边界约束。
```
因此 logical slot 中：
```text
consumer 停滞:
  confirmed_flush_lsn 不动。
decoder 被旧事务钉住:
  confirmed_flush_lsn 可能动，restart_lsn 不动。
```
这两个不能混为一谈。
### 10.2 反馈缺失
反馈缺失不是一个单一概念。
对 physical standby，有两类 feedback。
第一类是普通 status update：
```text
write_lsn / flush_lsn / replay_lsn / reply_time
```
缺失时，primary 不知道 standby 已经写到哪里。
如果使用 physical slot：
```text
restart_lsn 不推进；
WAL 保留压力上升。
```
第二类是 Hot Standby feedback：
```text
xmin / catalog_xmin
```
缺失时，primary 不会为了 standby 查询保留旧 tuple。
结果通常是：
```text
primary bloat 压力下降；
standby 查询 cancellation 风险上升。
```
所以当你看到主库 bloat 时，不能说“因为 hot_standby_feedback 缺失”。
更合理的判断是：
```text
如果 physical slot 的 xmin / catalog_xmin 非空且很旧:
  feedback 存在，并且正在保留旧版本。
如果 xmin / catalog_xmin 为空:
  slot 只解释 WAL 保留，不解释 VACUUM bloat。
  bloat 应该去查普通长事务、prepared xact、autovacuum、表级 workload 或 logical slot。
```
对 logical replication，反馈缺失通常表现为 flush ack 缺失：
```text
walsender active；
可能已经发送数据；
consumer 端可能处理了部分数据；
但没有发送或持久化 flush LSN；
confirmed_flush_lsn 不推进。
```
这类问题不是 `hot_standby_feedback`。
它是 logical protocol 的 status update / flush ack 问题。
### 10.3 保留策略过宽
保留策略过宽的核心特征是：
```text
PostgreSQL 正在按配置正确保留资源，
但这个保留目标已经超过业务愿意支付的磁盘或 bloat 成本。
```
典型形态：
```text
max_slot_wal_keep_size = -1；
safe_wal_size = NULL；
inactive slot 很旧；
restart_lsn 很旧；
没有明确恢复这个 consumer 的计划。
```
这时系统没有办法自动知道：
```text
这个 slot 是业务关键的恢复点；
还是已经废弃的 CDC 消费者。
```
另一个形态是：
```text
wal_keep_size 过大；
slot lag 不大；
但 pg_wal 保留仍明显超过预期；
safe_wal_size 受 Max(max_slot_wal_keep_size, wal_keep_size) 影响。
```
还有一种形态是：
```text
业务允许 standby 断线 24 小时继续追赶；
WAL 生成速率又很高；
max_slot_wal_keep_size 被设置到巨大值；
wal_status 长期 extended。
```
这不是 consumer 停滞的单点故障。
它是可用性策略换磁盘空间。
诊断报告里应该写成：
```text
当前配置选择保护消费者追赶能力；
代价是 pg_wal 可能超过 max_wal_size 并持续占用磁盘；
如果 primary 磁盘可用性优先，应降低上限、缩短 RPO、drop 废弃 slot 或改用归档补偿。
```
## 11. 一条可执行的诊断 SQL 主线
第一步，找最老 WAL 保留边界：
```sql
SELECT slot_name,
       slot_type,
       temporary,
       active,
       active_pid,
       restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS restart_lag,
       confirmed_flush_lsn,
       wal_status,
       pg_size_pretty(safe_wal_size) AS safe_wal_size,
       xmin,
       catalog_xmin,
       inactive_since,
       conflicting,
       invalidation_reason
FROM pg_replication_slots
ORDER BY restart_lsn NULLS LAST;
```
这条 SQL 回答：
```text
谁的 restart_lsn 最旧？
它是否 active？
它是否已经失效？
它是否还剩可计算的 safe_wal_size？
它是否同时发布 xmin / catalog_xmin？
```
第二步，把 active slot 接到 walsender：
```sql
SELECT s.slot_name,
       s.slot_type,
       s.active_pid,
       r.state,
       r.sent_lsn,
       r.write_lsn,
       r.flush_lsn,
       r.replay_lsn,
       r.reply_time,
       r.sync_state
FROM pg_replication_slots s
LEFT JOIN pg_stat_replication r
       ON r.pid = s.active_pid
ORDER BY s.slot_name;
```
解释要分 physical 和 logical。
physical：
```text
flush_lsn 不动:
  standby 没有确认写盘；
  restart_lsn 通常也不会前进。
reply_time 旧:
  反馈链路不活跃。
sent_lsn 远大于 flush_lsn:
  primary 已发送或尝试发送，瓶颈更靠近网络、standby 接收或写盘。
```
logical：
```text
flush_lsn 是 consumer ack；
confirmed_flush_lsn 应该跟 ack 方向一致；
confirmed_flush_lsn 不动时优先看 consumer 是否发送 status update；
confirmed_flush_lsn 动但 restart_lsn 不动时再看解码事务边界。
```
第三步，看 WAL 目录本身：
```sql
SELECT count(*) AS files,
       pg_size_pretty(sum(size)) AS total_size,
       min(name) AS oldest_file,
       max(name) AS newest_file
FROM pg_ls_waldir()
WHERE name ~ '^[0-9A-F]{24}$';
```
把最老 slot 的 `restart_lsn` 映射到文件：
```sql
SELECT slot_name,
       restart_lsn,
       pg_walfile_name(restart_lsn) AS restart_file
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL
ORDER BY restart_lsn;
```
注意：
```text
pg_ls_waldir() 看到的是文件系统当前状态；
pg_walfile_name() 是 LSN 到文件名的映射；
wal_status 是源码根据保留边界计算的可用性状态；
三者粒度不同。
```
第四步，看 WAL 生成速率：
```sql
SELECT wal_records,
       pg_size_pretty(wal_bytes) AS wal_bytes,
       wal_buffers_full,
       stats_reset
FROM pg_stat_wal;
```
`pg_stat_wal` 是实例累计统计。
它不能告诉你哪个 slot 导致增长。
它回答的是：
```text
在当前 reset 周期内，系统 WAL 生成量有多快；
safe_wal_size 按这个速率还能撑多久。
```
第五步，看 bloat 是否真的能归因到 slot。
先看 slot horizon：
```sql
SELECT slot_name,
       slot_type,
       xmin,
       age(xmin) AS xmin_age,
       catalog_xmin,
       age(catalog_xmin) AS catalog_xmin_age,
       active,
       inactive_since
FROM pg_replication_slots
WHERE xmin IS NOT NULL
   OR catalog_xmin IS NOT NULL
ORDER BY greatest(age(xmin), age(catalog_xmin)) DESC NULLS LAST;
```
再看普通 backend horizon：
```sql
SELECT pid,
       backend_type,
       state,
       backend_xmin,
       age(backend_xmin) AS backend_xmin_age,
       xact_start,
       query_start,
       wait_event_type,
       wait_event,
       query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```
判断逻辑：
```text
slot xmin / catalog_xmin 很旧:
  slot 可以解释 VACUUM 清理受阻。
slot 没有 xmin / catalog_xmin，但有普通 backend_xmin 很旧:
  bloat 更可能来自长事务或长 snapshot。
slot 只有 restart_lsn 很旧:
  它解释 WAL 保留，不解释 tuple bloat。
```
## 12. 异常路径与失效状态
slot invalidation 会改变视图解释。
`slot.h` 中的 invalidation cause 是：
```text
RS_INVAL_WAL_REMOVED
RS_INVAL_HORIZON
RS_INVAL_WAL_LEVEL
RS_INVAL_IDLE_TIMEOUT
```
`slot.c` 映射到视图字符串：
```text
wal_removed
rows_removed
wal_level_insufficient
idle_timeout
```
`InvalidateObsoleteReplicationSlots()` 主要在这些场景调用：
```text
checkpoint / restartpoint 将删除旧 WAL:
  RS_INVAL_WAL_REMOVED
idle_replication_slot_timeout 触发:
  RS_INVAL_IDLE_TIMEOUT
recovery 中 logical slot 与 xid horizon 冲突:
  RS_INVAL_HORIZON
standby 上 logical decoding 条件不足:
  RS_INVAL_WAL_LEVEL
```
如果 slot inactive，可以直接 acquire 后标记 invalidated 并保存：
```text
s->active_proc = MyProcNumber
s->data.invalidated = cause
ReplicationSlotMarkDirty()
ReplicationSlotSave()
ReplicationSlotRelease()
```
如果 slot active，源码会 signal owner process 并等待 release 后重试。
这解释为什么有时日志是：
```text
terminating process ... to release replication slot
```
有时日志是：
```text
invalidating obsolete replication slot
```
`wal_status` 在 invalidated slot 上也要小心。
`slotfuncs.c` 里只要：
```text
slot_contents.data.invalidated != RS_INVAL_NONE
```
就先把 `walstate` 当成 `WALAVAIL_REMOVED` 路径处理。
因此：
```text
wal_status = lost
```
不一定只表示原始原因是 WAL 被删除。
必须同时看：
```text
invalidation_reason
```
例如：
```text
invalidation_reason = rows_removed:
  根因是需要的 rows / catalog horizon 已经被 recovery 冲掉。
invalidation_reason = wal_level_insufficient:
  根因是 standby logical decoding 条件不足。
invalidation_reason = idle_timeout:
  根因是 idle_replication_slot_timeout 策略。
```
## 13. 常见误区与误判边界
| 误判 | 正确边界 |
| --- | --- |
| `active = true` 表示 consumer 健康 | 它只表示某个 backend 持有 slot；还要看 `reply_time`、`flush_lsn`、`confirmed_flush_lsn` 是否推进。 |
| `inactive_since` 是故障开始时间 | 它会在 release 和 startup restore 时设置；实例重启后只能说明本次内存生命周期内的 inactive 起点。 |
| `restart_lsn` 很旧就能解释 bloat | `restart_lsn` 解释 WAL；tuple / catalog bloat 要看 `xmin`、`catalog_xmin`、普通 `backend_xmin` 和 prepared xact。 |
| `confirmed_flush_lsn` 是所有 slot 的消费位置 | 它主要服务 logical slot；physical slot 要看 standby status update 的 `flush_lsn` 如何推进 `restart_lsn`。 |
| `safe_wal_size = NULL` 表示安全 | NULL 可能表示 `max_slot_wal_keep_size = -1`、slot 已 lost、`restart_lsn` 无效或无法计算。 |
| `wal_status = reserved` 表示没有风险 | 它只说明 target segment 仍在当前计算的保留范围内，不预测 WAL 生成速率和下次 checkpoint。 |
| `wal_status = lost` 必然说明根因是 WAL 删除 | invalidated slot 会被强制走 removed-like 路径；必须同时看 `invalidation_reason`。 |
| `pg_wal` 当前大小就是泄漏 | 删除依赖 checkpoint、archive status、recycling、`wal_keep_size`、slot required LSN 和 WAL summarization。 |
误判的共同点是把 raw field 当语义。
本节真正要训练的是组合解释：
```text
field + 推进者 + checkpoint/VACUUM 边界 + invalidation 状态
```
单个字段只能给方向，不能直接给根因。
## 14. 课堂实验：分层排查 slot retention 风险
### 14.1 logical ack 与 `confirmed_flush_lsn`
用 `test_decoding` 创建 logical slot：
```sql
CREATE TABLE slot_diag_t(id int primary key, v text);
SELECT * FROM pg_create_logical_replication_slot('diag_logical', 'test_decoding');
INSERT INTO slot_diag_t SELECT g, md5(g::text) FROM generate_series(1, 1000) AS g;
```
先查 `restart_lsn`、`confirmed_flush_lsn`、`wal_status`，再执行：
```sql
SELECT * FROM pg_logical_slot_peek_changes('diag_logical', NULL, 10);
SELECT * FROM pg_logical_slot_get_changes('diag_logical', NULL, 10);
```
对比两次之后的 slot 状态。
`peek` 能看到变更，但不代表 durable ack；`get` 会走 `LogicalConfirmReceivedLocation()`。
如果 `confirmed_flush_lsn` 变了而 `restart_lsn` 没变，回到 `logical.c` 看 candidate restart 是否已经满足。
清理：
```sql
SELECT pg_drop_replication_slot('diag_logical');
DROP TABLE slot_diag_t;
```
### 14.2 inactive persistent slot 与 WAL 保留
创建一个立即保留 WAL 的 physical slot：
```sql
SELECT * FROM pg_create_physical_replication_slot('diag_physical', true, false);
```
生成 WAL 并 checkpoint：
```sql
CREATE TABLE slot_wal_pressure(id bigint, payload text);
INSERT INTO slot_wal_pressure
SELECT g, repeat(md5(g::text), 10) FROM generate_series(1, 200000) AS g;
CHECKPOINT;
```
观察：
```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag,
       wal_status, pg_size_pretty(safe_wal_size) AS safe, inactive_since
FROM pg_replication_slots
WHERE slot_name = 'diag_physical';
```
这个实验验证：没有 consumer 连接时，persistent slot 仍按 `restart_lsn` 保留 WAL。
源码回看 `ReplicationSlotReserveWal()`、`ReplicationSlotsComputeRequiredLSN()`、`KeepLogSeg()` 和 `RemoveOldXlogFiles()`。
清理：
```sql
SELECT pg_drop_replication_slot('diag_physical');
DROP TABLE slot_wal_pressure;
```
### 14.3 WAL retention 与 VACUUM horizon 分离
在有 physical standby 的环境中，对比 `hot_standby_feedback = off/on`：
```text
standby 开启长只读事务；
primary 对同一表大量 UPDATE / DELETE；
primary 执行 VACUUM；
primary 查询 pg_replication_slots.restart_lsn、xmin、catalog_xmin。
```
预期边界：
```text
hot_standby_feedback = off:
  physical slot 仍可能保留 WAL；
  但 primary 不应因为 standby query horizon 保留旧 tuple。
hot_standby_feedback = on:
  standby horizon 会反馈给 primary；
  使用 slot 时可能体现在 slot.xmin / catalog_xmin；
  primary VACUUM 更可能被保守 horizon 拦住。
```
源码回看 `XLogWalRcvSendHSFeedback()`、`ProcessStandbyHSFeedbackMessage()`、`PhysicalReplicationSlotNewXmin()` 和 `ReplicationSlotsComputeRequiredXmin()`。
### 14.4 低上限观察 `unreserved` / `lost`
开发环境中设置较小 `max_slot_wal_keep_size`，创建 inactive slot，持续生成 WAL 并多次 checkpoint。
观察：
```sql
SELECT slot_name, restart_lsn, wal_status, safe_wal_size, invalidation_reason
FROM pg_replication_slots
WHERE slot_name = '...';
```
目标是看到状态从 `reserved` 到 `extended`，再到 `unreserved` 或 `lost / wal_removed`。
这个实验会故意制造 slot 失效，不应在生产执行。
## 15. 讨论题
1. 为什么 `pg_replication_slots.wal_status` 不能直接等价为 consumer lag？
2. 一个 physical slot 的 `restart_lsn` 很旧，但 `xmin` 和 `catalog_xmin` 都是 NULL，它能解释哪类资源压力，不能解释哪类资源压力？
3. logical slot 的 `confirmed_flush_lsn` 持续推进，但 `restart_lsn` 不推进，为什么不能马上判断 consumer 停了？
4. 为什么 `inactive_since` 在实例重启后可能误导故障起点判断？
5. `safe_wal_size` 为 NULL 时，分别有哪些可能含义？
6. 为什么 Hot Standby feedback 关闭通常不会造成 primary bloat，却可能造成 standby 查询取消？
7. `wal_status = unreserved` 和 `invalidation_reason = wal_removed` 分别处在哪个阶段？
8. 如果业务要求 CDC consumer 可以离线 12 小时，应该用哪些指标估算磁盘预算，而不是只看 `max_wal_size`？
## 16. 本节小结
本节的核心链路是：
```text
pg_wal 增长
  -> 找最老 restart_lsn
  -> 用 wal_status / safe_wal_size 判断 WAL 可用性和上限风险
  -> 用 active / active_pid / pg_stat_replication 判断推进者是否还活着
  -> 用 confirmed_flush_lsn 判断 logical ack 是否推进
  -> 用 xmin / catalog_xmin 判断是否影响 VACUUM horizon
  -> 用 inactive_since / invalidation_reason 判断生命周期和异常边界
```
核心状态和边界：
```text
restart_lsn:
  WAL 保留起点。
confirmed_flush_lsn:
  logical consumer 确认位置。
xmin / catalog_xmin:
  VACUUM horizon 输入。
wal_status / safe_wal_size:
  基于当前 WAL 写入位置和配置即时计算的风险标签。
active / inactive_since:
  slot owner 生命周期，不等于 consumer 健康证明。
```
ownership / cleanup 的关键点：
```text
walsender acquire slot 后才能推进主字段；
release persistent slot 只清 active_proc 并设置 inactive_since；
checkpoint 保存 dirty slot，并在删除 WAL 前 invalidation obsolete slots；
invalidated slot 不再参与 required LSN / xmin 聚合。
```
错误路径如何收尾：
```text
WAL 超过 max_slot_wal_keep_size:
  checkpoint 路径可能把 slot invalidated 为 wal_removed。
idle slot 超过 idle_replication_slot_timeout:
  slot 可能 invalidated 为 idle_timeout。
logical standby 与 horizon 或 wal_level 冲突:
  invalidation_reason 可能是 rows_removed 或 wal_level_insufficient。
```
可观测与不可观测：
```text
能直接看到:
  pg_replication_slots、pg_stat_replication、pg_stat_wal、pg_ls_waldir。
只能推断:
  consumer 是否已经持久化处理业务事件；
  logical restart_lsn 不推进是因为哪个事务或 reorder buffer 状态。
几乎不可从 SQL 单点确认:
  checkpoint 下一次具体删除哪个文件；
  某个 bloat 字节完全由哪个 horizon 造成。
```
可迁移规律：
```text
资源保留诊断不要从资源量开始归因；
先找最低保护边界，再找负责推进这个边界的反馈链，
最后判断边界不推进是 consumer 停滞、feedback 缺失，还是策略故意允许长期保留。
```
哪些判断仍需近似：
```text
safe_wal_size 到耗尽时间依赖 WAL 生成速率；
pg_wal 当前大小依赖 checkpoint、archive、recycling 和 preallocation；
bloat 归因依赖 workload、表结构、autovacuum、长事务和 prepared transaction；
logical decoding 的 restart_lsn 推进依赖事务形状、2PC、streaming 和 output/plugin 行为。
```
