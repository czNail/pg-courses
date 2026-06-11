# PostgreSQL standby feedback 与 bloat 取舍
## 课程定位
前置知识：已经理解 physical streaming replication 的 walsender / walreceiver 主链路，Hot Standby query 如何持有 snapshot，以及 standby conflict 为什么会取消查询。
本节唯一主问题：
```text
hot_standby_feedback 可以减少 standby 查询取消，
为什么也可能放大 primary bloat，
以及如何结合 conflict 计数、xmin 滞留和业务查询时长判断取舍？
```
核心矛盾：standby 上的长查询希望 primary 不要过早清理旧 tuple version；primary 上的 VACUUM 希望尽快移除 dead tuple、推进 freeze cutoff、缩短 HOT chain 和索引回表成本。PostgreSQL 不能同时保证“standby 长查询永不取消”和“primary cleanup horizon 永远尽快前进”。`hot_standby_feedback` 的本质不是免费开关，而是把一部分 snapshot conflict 从 standby replay 阶段提前转移到 primary cleanup horizon 阶段。
学完后应能判断：
```text
XLogWalRcvSendHSFeedback() 发出的 xmin / catalog_xmin 从哪里来；
ProcessStandbyHSFeedbackMessage() 如何把 feedback 写入 walsender PGPROC 或 physical slot；
PhysicalReplicationSlotNewXmin() 为什么让 slot 场景的 xmin 变成持久边界；
hot_standby_feedback=off 时最终 invalid feedback 如何清除 primary 端边界；
max_standby_streaming_delay 和 hot_standby_feedback 分别牺牲什么；
pg_stat_database_conflicts 的 snapshot 计数为什么不能单独决定是否开启 feedback；
xmin / catalog_xmin 滞留如何进入 ComputeXidHorizons() 和 VACUUM OldestXmin；
slot 与非 slot feedback 在精度、持久性和故障边界上的差异；
如何把 primary bloat、standby 查询时长和 conflict 计数放到同一个判断框架里。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前面关于 Hot Standby conflict 的课程已经建立了这条链：
```text
primary VACUUM / pruning / index cleanup 写出带 cleanup horizon 的 WAL
  -> standby startup process replay WAL
  -> ResolveRecoveryConflictWithSnapshot() 找到 xmin 太老的 standby backend
  -> 等待不超过 max_standby_streaming_delay
  -> 超时后取消查询或终止连接
  -> pg_stat_database_conflicts.confl_snapshot 累加
```
本节换一个方向看同一个现象：
```text
standby backend 持有长 snapshot
  -> walreceiver 周期性把 standby cleanup horizon 发给 primary
  -> primary walsender 把这个 horizon 发布成 MyProc->xmin 或 replication slot xmin
  -> primary ComputeXidHorizons() 把它纳入 OldestXmin
  -> primary VACUUM 不再移除 standby 可能需要的旧版本
  -> standby snapshot conflict 减少
  -> primary dead tuple、page 空洞、index bloat 和 freeze 压力可能增加
```
这不是两个互斥机制。
它们经常同时存在：
```text
hot_standby_feedback=off:
  primary cleanup 自由度较高；
  standby 遇到 cleanup WAL 时更可能取消旧 snapshot query。
hot_standby_feedback=on:
  standby snapshot query 更可能存活；
  primary cleanup horizon 可能被 standby 的 xmin / catalog_xmin 拉老。
```
`max_standby_streaming_delay` 调整的是 standby 端已经发生冲突后的等待时长。
`hot_standby_feedback` 调整的是 primary 端是否提前避免生成一部分 cleanup conflict。
把二者混成一个“读写分离稳定性参数”是诊断中的常见错误。
本节不重新讲所有 conflict 类型。
我们只围绕 snapshot cleanup conflict：
```text
旧 tuple version 是否还能被某个 standby snapshot 需要？
```
lock conflict、DROP DATABASE、DROP TABLESPACE 和 buffer pin conflict 会作为边界被提及，因为 `hot_standby_feedback` 不能解决它们。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
walreceiver 在 standby 上用 GetReplicationHorizons() 计算本机仍需要的 xmin / catalog_xmin，
通过 HotStandbyFeedback replication message 发给 upstream walsender；
walsender 校验 epoch 后，
无 slot 时把最保守 xmin 写入自己的 PGPROC->xmin，
有 physical slot 时写入 slot->data/effective xmin 与 catalog_xmin；
ProcArray 的 ComputeXidHorizons() 把这些边界并入 relation-sensitive cleanup horizon；
primary VACUUM / pruning 因 OldestXmin 变老而保留更多 recently-dead tuple；
standby 因 primary 少生成 cleanup WAL 而减少 snapshot conflict。
```
这里有两个不能同时最优的目标：
```text
standby query continuity:
  长报表、导出、在线分析不希望被 recovery conflict 取消。
primary cleanup freedom:
  OLTP 主库希望 dead tuple 尽快被移除，HOT chain 尽快缩短，
  relfrozenxid 和 catalog horizon 能持续推进。
```
`hot_standby_feedback` 的 tradeoff 不是“开启好还是关闭好”。
它要回答：
```text
把 standby 长查询的成本放在哪里更可接受？
放在 standby:
  查询被取消、客户端重试、报表失败、只读体验差。
放在 primary:
  dead tuple 保留、表和索引膨胀、autovacuum 工作量增加、
  freeze warning 变早、写路径和读路径都可能变慢。
```
这个成本归属必须结合 workload 判断。
一个 20 分钟的报表偶尔被取消，和一个全天候 BI 查询把主库 `xmin` 钉住 6 小时，是完全不同的系统风险。
本节最后要形成的判断框架是：
```text
先看 standby 是否真的在被 snapshot conflict 取消；
再看 primary 是否出现 xmin/catalog_xmin 滞留和 bloat；
最后把滞留时长与业务可接受查询时长、重试成本、主库写入强度放在一起决定。
```
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/replication/walreceiver.c` | `XLogWalRcvSendHSFeedback()` 如何计算、限频和发送 standby feedback，以及关闭时的 invalid feedback。 |
| 2 | `src/backend/storage/ipc/procarray.c` | `GetReplicationHorizons()`、`ComputeXidHorizons()`、`GetOldestNonRemovableTransactionId()` 如何把 feedback 变成 cleanup horizon。 |
| 3 | `src/backend/replication/walsender.c` | `ProcessStandbyHSFeedbackMessage()` 如何校验并应用 feedback；`PhysicalReplicationSlotNewXmin()` 如何更新 physical slot。 |
| 4 | `src/backend/replication/slot.c` | `ReplicationSlotsComputeRequiredXmin()` 如何扫描 slot 的 `effective_xmin` / `effective_catalog_xmin` 并投影到 ProcArray。 |
| 5 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()` 如何把 ProcArray horizon 变成 `OldestXmin` 和 freeze cutoff。 |
| 6 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesVacuum()` 如何用 `OldestXmin` 把 `RECENTLY_DEAD` 转成 `DEAD`。 |
| 7 | `src/backend/access/heap/pruneheap.c` | `heap_page_prune_and_freeze()` 及 `GlobalVisTestIsRemovableXid()` 如何在 heap cleanup hot path 使用 horizon。 |
| 8 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 如何取得 cutoffs、`GlobalVisState`，并在扫描中保留或移除 tuple。 |
| 9 | `src/backend/storage/ipc/standby.c` | `max_standby_streaming_delay` 的等待和取消边界，与 feedback 的作用点对比。 |
| 10 | `src/backend/catalog/system_views.sql` | `pg_stat_database_conflicts`、`pg_stat_replication`、`pg_replication_slots`、`pg_stat_activity` 的观测字段。 |
建议阅读顺序不是从 GUC 文档开始。
先沿一条实际状态链走：
```text
standby 当前最老 xmin
  -> feedback message
  -> primary walsender / slot state
  -> ProcArray horizon
  -> VACUUM OldestXmin
  -> tuple cleanup decision
  -> standby conflict counter 与 primary bloat 指标
```
这样能避免把课程写成参数解释。
## 4. 关键状态与边界
### 4.1 standby 端 feedback message
`XLogWalRcvSendHSFeedback(bool immed)` 是 standby walreceiver 发 Hot Standby feedback 的入口。
它维护一个 backend-local static 状态：
```text
primary_has_standby_xmin
```
这个变量的语义不是“primary 当前一定持有 xmin”。
它只是 walreceiver 本地记忆：
```text
我上一次是否给 primary 发送过有效 xmin 或 catalog_xmin；
如果用户关闭 feedback，我需要至少再发一次 invalid 值让 primary 清掉旧边界。
```
函数的前置过滤有几层：
```text
wal_receiver_status_interval <= 0 且 primary_has_standby_xmin=false:
  不发。
hot_standby_feedback=false 且 primary_has_standby_xmin=false:
  不发。
immed=false 且未到下一次 wakeup:
  不发。
HotStandbyActive() 为 false:
  不发。
```
`HotStandbyActive()` 之后才计算 xmin。
源码注释说明这样可以避免在 standby 还没读到自己 slot 状态前，错误告诉 primary 丢弃仍需要的 `xmin` / `catalog_xmin`。
当 `hot_standby_feedback=on` 时：
```text
GetReplicationHorizons(&xmin, &catalog_xmin)
```
当 `hot_standby_feedback=off` 时：
```text
xmin = InvalidTransactionId
catalog_xmin = InvalidTransactionId
```
随后 walreceiver 读取 `ReadNextFullTransactionId()` 来计算 epoch。
如果 `nextXid < xmin`，说明 xmin 和 next xid 位于 epoch 边界两侧，发送的 `xmin_epoch` 要减一。
消息内容是：
```text
PqReplMsg_HotStandbyFeedback
reply timestamp
xmin
xmin_epoch
catalog_xmin
catalog_xmin_epoch
```
发送后：
```text
只要 xmin 或 catalog_xmin 有效:
  primary_has_standby_xmin = true
否则:
  primary_has_standby_xmin = false
```
这个细节决定了关闭 feedback 的清理边界。
只要 walreceiver 仍运行并触发一次 immediate feedback，primary 会收到 invalid 值。
如果连接已经断掉，非 slot 场景通常随着 walsender 退出释放 `PGPROC->xmin`。
slot 场景则不同：slot 的 `data.xmin` / `data.catalog_xmin` 是复制槽状态，需要后续 invalid feedback、slot drop 或 slot invalidation 才能移除这个持久边界。
### 4.2 GetReplicationHorizons: feedback 不直接读查询列表
`GetReplicationHorizons()` 在 `procarray.c` 中。
它不是 walreceiver 自己扫描 backend。
它调用：
```text
ComputeXidHorizons(&horizons)
```
然后返回：
```text
xmin = horizons.shared_oldest_nonremovable_raw
catalog_xmin = horizons.slot_catalog_xmin
```
这里的 `raw` 很重要。
`shared_oldest_nonremovable` 已经包含 slot `catalog_xmin` 的影响。
feedback 不想把 catalog horizon 混到 data horizon 里，因为 upstream primary 可以分别处理：
```text
data xmin:
  保留普通 data table 仍需要的旧版本。
catalog xmin:
  保留 logical decoding 或 catalog snapshot 仍需要的 catalog 历史。
```
所以源码特意发送 `shared_oldest_nonremovable_raw` 和 `slot_catalog_xmin`。
这也是 slot 与非 slot 差异的伏笔。
有 slot 时 primary 可以分别记录 data xmin 和 catalog xmin。
无 slot 时 walsender 只能把二者压成一个 `MyProc->xmin`。
### 4.3 primary 端 walsender PGPROC
`ProcessStandbyHSFeedbackMessage()` 是 primary walsender 处理 feedback 的入口。
它先解码消息：
```text
replyTime
feedbackXmin
feedbackEpoch
feedbackCatalogXmin
feedbackCatalogEpoch
```
然后更新 `MyWalSnd->replyTime`，这属于 walsender status，不是 cleanup horizon 本身。
真正影响 cleanup 的分支在后面。
如果两个值都不是 normal transaction id：
```text
MyProc->xmin = InvalidTransactionId
如果 MyReplicationSlot != NULL:
  PhysicalReplicationSlotNewXmin(invalid, invalid)
return
```
这就是 `hot_standby_feedback=off` 的最终清除路径。
关闭 feedback 不是让 primary “自然知道不用保留”。
standby 必须发出 invalid feedback，或者 non-slot walsender 生命周期结束。
如果 feedback 值有效，walsender 会用 `TransactionIdInRecentPast()` 检查 `xmin/epoch` 是否合理。
不合理的值会被忽略。
源码注释强调这个检查只关心是不是未来或已经 wraparound 到太久以前，不检查 clog 是否还存在。
校验通过后，分两种应用方式。
无 replication slot：
```text
如果 feedbackCatalogXmin 更老:
  MyProc->xmin = feedbackCatalogXmin
否则:
  MyProc->xmin = feedbackXmin
```
有 physical replication slot：
```text
PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)
```
此外，walsender 初始化阶段还有一个相关边界。
如果 client 没有指定 database，`walsender.c` 会把：
```text
PROC_AFFECTS_ALL_HORIZONS
```
写入 `MyProc->statusFlags` 和 `ProcGlobal->statusFlags[]`。
源码注释说明这是为了让 physical replication client 的 hot standby feedback 影响所有数据库的 vacuum horizon。
这解释了一个线上现象：
```text
一个 standby 的长查询不一定只影响某个数据库里看起来相关的表；
physical feedback 可能通过 walsender PGPROC 影响 primary 上所有 database 的 cleanup horizon。
```
### 4.4 PhysicalReplicationSlotNewXmin
`PhysicalReplicationSlotNewXmin()` 是 slot 场景的关键函数。
它在 `walsender.c` 中，不在 `slot.c` 中。
函数先拿 slot spinlock：
```text
SpinLockAcquire(&slot->mutex)
MyProc->xmin = InvalidTransactionId
```
`MyProc->xmin` 被清掉，是因为 physical slot 会承担 xmin 保留责任。
随后分别处理 data xmin 和 catalog xmin：
```text
如果 slot->data.xmin 非 normal，
或者 feedbackXmin 非 normal，
或者 slot->data.xmin 早于 feedbackXmin:
  slot->data.xmin = feedbackXmin
  slot->effective_xmin = feedbackXmin
catalog_xmin 同理。
```
这个条件有两个含义。
第一，valid feedback 一般只推动 slot xmin 向前。
如果 standby 发来比当前 slot 更老的 xmin，当前源码不会把 slot horizon 后退。
第二，invalid feedback 会清掉 slot 中对应的 xmin。
这就是关闭 feedback 时清理持久边界的实际路径。
如果发生变化：
```text
ReplicationSlotMarkDirty()
ReplicationSlotsComputeRequiredXmin(false)
```
`ReplicationSlotMarkDirty()` 让 slot 状态后续可持久化。
`ReplicationSlotsComputeRequiredXmin()` 把所有 slot 的 `effective_xmin` / `effective_catalog_xmin` 重新汇总到 ProcArray。
### 4.5 ProcArray slot 聚合状态
`ProcArrayStruct` 中有两个聚合字段：
```text
replication_slot_xmin
replication_slot_catalog_xmin
```
它们不是某一个 slot 的状态。
它们是所有有效 slot 的最老需求。
`ReplicationSlotsComputeRequiredXmin()` 在 `slot.c` 中扫描 slot 数组：
```text
for each slot:
  如果 !in_use: skip
  读取 effective_xmin / effective_catalog_xmin / invalidated
  如果 slot 已 invalidated: skip
  agg_xmin = min(agg_xmin, effective_xmin)
  agg_catalog_xmin = min(agg_catalog_xmin, effective_catalog_xmin)
ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin)
```
`ProcArraySetReplicationSlotXmin()` 拿 `ProcArrayLock` exclusive 后写入：
```text
procArray->replication_slot_xmin
procArray->replication_slot_catalog_xmin
```
后续 `ComputeXidHorizons()` 不需要知道每个 slot 的细节。
它只读取这两个聚合边界。
这也是为什么 `pg_replication_slots` 诊断要看每个 slot 的 `xmin` / `catalog_xmin`，而源码的 cleanup horizon 计算只看聚合最小值。
一个最老 slot 就足以钉住整个实例的相关 cleanup horizon。
### 4.6 OldestXmin 与 tuple cleanup
primary VACUUM 的入口之一是 `vacuum_get_cutoffs()`。
当前源码中它调用：
```text
cutoffs->OldestXmin = GetOldestNonRemovableTransactionId(rel)
```
`GetOldestNonRemovableTransactionId()` 内部调用 `ComputeXidHorizons()`，再按 relation 类型选择：
```text
shared relation:
  shared_oldest_nonremovable
catalog relation:
  catalog_oldest_nonremovable
普通 data relation:
  data_oldest_nonremovable
temp relation:
  temp_oldest_nonremovable
```
`HeapTupleSatisfiesVacuum()` 的注释和逻辑给出了最直接的 cleanup 边界：
```text
tuple 已被 committed XID 删除或更新；
如果 dead_after < OldestXmin:
  HEAPTUPLE_DEAD，可以移除；
否则:
  HEAPTUPLE_RECENTLY_DEAD，仍可能被某个 open transaction 看见，不能移除。
```
`pruneheap.c` 中的 `heap_prune_satisfies_vacuum()` 也用相同边界，并且结合 `GlobalVisTestIsRemovableXid()`。
因此 feedback 造成 bloat 的机制不是抽象说法。
它是具体的 cutoff 变化：
```text
standby query xmin 老
  -> primary walsender / slot xmin 老
  -> ComputeXidHorizons() 的 data/catalog/shared horizon 老
  -> VACUUM OldestXmin 老
  -> 更多 tuple 维持 RECENTLY_DEAD
  -> page 不能充分 prune，index entry 不能及时失效回收
  -> 表和索引空间增长，HOT chain 和 visibility check 成本增加
```
## 5. 主流程源码 walkthrough
### 5.1 standby 发出 feedback
从 standby 上一个长查询开始：
```text
BEGIN;
SELECT ... FROM large_table ...;
```
查询获取 snapshot 后，backend 的 `PGPROC->xmin` 会发布它仍可能需要的最老 snapshot horizon。
walreceiver 不是直接读取这个 backend。
它周期性进入：
```text
XLogWalRcvSendHSFeedback(false)
```
这通常发生在 walreceiver 主循环中，和 status reply 相邻。
配置 reload 或连接建立后也会以 `immed=true` 发送。
如果 `hot_standby_feedback=on` 且 Hot Standby 已经 active：
```text
XLogWalRcvSendHSFeedback()
  -> GetReplicationHorizons()
     -> ComputeXidHorizons()
        -> 扫描 ProcArray 中的 PGPROC->xmin / xid
        -> 合并 recovery 下 KnownAssignedXids
        -> 合并本机 replication slot xmin/catalog_xmin
     -> 返回 shared_oldest_nonremovable_raw 和 slot_catalog_xmin
  -> 计算 epoch
  -> walrcv_send(PqReplMsg_HotStandbyFeedback, ...)
```
这条链的状态含义是：
```text
standby 并没有告诉 primary “我正在跑哪个 SQL”；
它只告诉 primary “我这里仍可能需要不早于哪个 xmin 的旧版本”。
```
如果 `wal_receiver_status_interval` 较大，feedback 传播就有粒度。
如果 primary 在 feedback 到达前已经 VACUUM 并写出 cleanup WAL，standby 后续仍可能发生 conflict。
`hot_standby_feedback` 不是时间倒流机制。
它只影响 primary 未来的 cleanup 决策。
### 5.2 primary 接收 feedback
primary walsender 在 replication protocol 中收到 `PqReplMsg_HotStandbyFeedback` 后进入：
```text
ProcessStandbyHSFeedbackMessage()
```
这一步先处理 invalid feedback：
```text
feedbackXmin 非 normal 且 feedbackCatalogXmin 非 normal:
  MyProc->xmin = InvalidTransactionId
  如果使用 slot:
    PhysicalReplicationSlotNewXmin(invalid, invalid)
  return
```
这是关闭 feedback 的最终边界。
如果不是 invalid，它会检查：
```text
TransactionIdInRecentPast(feedbackXmin, feedbackEpoch)
TransactionIdInRecentPast(feedbackCatalogXmin, feedbackCatalogEpoch)
```
任何一个 normal feedback 不合理，函数直接返回，不应用新 horizon。
然后进入 slot / non-slot 分叉。
无 slot：
```text
ProcessStandbyHSFeedbackMessage()
  -> MyProc->xmin = min(feedbackXmin, feedbackCatalogXmin)
```
`min` 这里指 TransactionId older。
因为 PGPROC 只有一个 `xmin` 字段，无法表达 data horizon 与 catalog horizon 的差异。
有 physical slot：
```text
ProcessStandbyHSFeedbackMessage()
  -> PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)
     -> slot->data.xmin = feedbackXmin
     -> slot->effective_xmin = feedbackXmin
     -> slot->data.catalog_xmin = feedbackCatalogXmin
     -> slot->effective_catalog_xmin = feedbackCatalogXmin
     -> ReplicationSlotsComputeRequiredXmin(false)
```
这里的 slot 场景有两个后果。
第一，`catalog_xmin` 能被单独保留。
普通 data relation 的 horizon 不必被更老的 catalog horizon 过度拉老。
第二，slot state 可以跨 walsender 进程生命周期存在。
如果 standby 或网络异常导致连接断开，slot 保留的 xmin 不会像普通 walsender `PGPROC->xmin` 那样自动随进程退出消失。
这是 slot 的可靠性优势，也是 bloat 风险来源。
### 5.3 primary ComputeXidHorizons 合并 feedback
primary 上 autovacuum 或手动 VACUUM 进入：
```text
vacuum_get_cutoffs(rel, params, &cutoffs)
  -> GetOldestNonRemovableTransactionId(rel)
     -> ComputeXidHorizons(&horizons)
```
`ComputeXidHorizons()` 在 `ProcArrayLock` shared 下做几件事：
```text
latest_completed = TransamVariables->latestCompletedXid
initial = latest_completed + 1
slot_xmin = procArray->replication_slot_xmin
slot_catalog_xmin = procArray->replication_slot_catalog_xmin
for each PGPROC:
  xid = ProcGlobal->xids[index]
  xmin = proc->xmin
  xmin = older(xmin, xid)
  如果 invalid: skip
  oldest_considered_running = older(...)
  如果 PROC_IN_VACUUM 或 PROC_IN_LOGICAL_DECODING: skip nonremovable horizon
  shared_oldest_nonremovable = older(shared, xmin)
  如果同库，或 PROC_AFFECTS_ALL_HORIZONS，或 recovery:
    data_oldest_nonremovable = older(data, xmin)
```
如果 feedback 是 non-slot，它以 walsender `PGPROC->xmin` 的形式出现在扫描中。
如果 physical walsender 没有绑定 database，`PROC_AFFECTS_ALL_HORIZONS` 让它影响所有 database 的 data horizon。
如果 feedback 是 slot，它在扫描 PGPROC 之后以聚合字段形式应用：
```text
shared_oldest_nonremovable = older(shared, slot_xmin)
data_oldest_nonremovable = older(data, slot_xmin)
shared_oldest_nonremovable_raw = shared_oldest_nonremovable
shared_oldest_nonremovable = older(shared, slot_catalog_xmin)
catalog_oldest_nonremovable = older(data, slot_catalog_xmin)
```
`oldest_considered_running` 随后也会被这些 horizon 拉老。
注意这里的分层：
```text
slot_xmin:
  影响普通 data 和 shared cleanup。
slot_catalog_xmin:
  影响 catalog cleanup，也影响 shared 因为 shared relation 里可能有 catalog。
walsender MyProc->xmin:
  作为一个 PGPROC xmin 进入普通 horizon 扫描；
  如果带 PROC_AFFECTS_ALL_HORIZONS，不能按数据库过滤。
```
这说明 bloat 风险不是简单“slot 才会 bloat”。
non-slot feedback 也能钉住 primary horizon，只是它通常跟 walsender 进程生命周期绑定。
slot feedback 的风险在于更持久、更不容易随连接消失。
### 5.4 VACUUM 消费 OldestXmin
`vacuum_get_cutoffs()` 取到 `OldestXmin` 后，还会检查 wraparound 安全边界。
如果 `OldestXmin` 比 `nextXID - autovacuum_freeze_max_age` 还老，当前源码会发出 warning：
```text
cutoff for removing and freezing tuples is far in the past
```
hint 会提到关闭长事务、prepared transaction 或 stale replication slot。
这条 warning 是 feedback bloat 诊断中的强信号。
随后 heap cleanup 走到 tuple 可见性判断：
```text
HeapTupleSatisfiesVacuum(tuple, OldestXmin, buffer)
  -> HeapTupleSatisfiesVacuumHorizon(...)
  -> 如果结果是 RECENTLY_DEAD:
       dead_after < OldestXmin ? DEAD : RECENTLY_DEAD
```
lazy VACUUM 的 heap scan 还会先设置：
```text
vacrel->aggressive = vacuum_get_cutoffs(...)
vacrel->vistest = GlobalVisTestFor(rel)
vacrel->NewRelfrozenXid = cutoffs.OldestXmin
```
`pruneheap.c` 中 `heap_prune_satisfies_vacuum()` 也保证：
```text
deleted tuple 的 xmax 早于 OldestXmin 时，VACUUM 必须能 prune；
否则可以继续用 GlobalVisState 判断是否已经可移除。
```
当 feedback 把 `OldestXmin` 拉老时，直接后果是：
```text
更多 tuple 的 delete/update XID 不满足 dead_after < OldestXmin；
它们保持 RECENTLY_DEAD；
page pruning 不能回收 line pointer 或压缩 HOT chain；
index cleanup 也无法完全摆脱这些 heap versions 的影响。
```
standby 查询被保护了。
primary 的空间和 CPU 成本开始积累。
### 5.5 standby conflict 是否真的减少
当 primary 不再移除 standby 可能需要的旧 tuple version，就不会为这些版本生成同样激进的 cleanup WAL。
standby startup process 后续在 replay heap cleanup、btree page reuse 等 WAL 时，遇到 snapshot conflict 的概率降低。
但不能推出：
```text
hot_standby_feedback=on => pg_stat_database_conflicts 全部归零
```
原因包括：
```text
feedback 有发送间隔，primary 可能已经清理过；
feedback 只保护 snapshot cleanup，不保护 lock/drop/tablespace/buffer pin；
without slot 的 feedback 只在 walsender 连续存在时有效；
invalid 或过旧/未来的 feedback 会被 walsender 忽略；
catalog_xmin 和 data xmin 的传播边界不同；
多级级联复制中，下游 feedback 需要逐级传递。
```
因此 `confl_snapshot` 降低可以证明一部分 cleanup conflict 被转移了。
`confl_lock`、`confl_tablespace`、`confl_bufferpin` 不应期待由 feedback 解决。
## 6. 生命周期 / ownership / cleanup
### 6.1 standby 端 ownership
standby query 持有 snapshot。
它通过自己的 backend `PGPROC->xmin` 影响 standby 本地 horizon。
walreceiver 不拥有这些 snapshot。
walreceiver 只在发送 feedback 时读取 `ComputeXidHorizons()` 的结果。
`XLogWalRcvSendHSFeedback()` 中真正跨调用保存的状态只有：
```text
primary_has_standby_xmin
wakeup[WALRCV_WAKEUP_HSFEEDBACK]
```
`primary_has_standby_xmin` 用于决定是否需要最终 invalid feedback。
它不是 shared state。
walreceiver crash 或连接重建后，新进程会重新从初始语义出发。
源码注释说该变量初始为 true，这样 first connect 时也会发送一条 feedback。
这样做是为了清理 previous connection 可能在 primary slot 上留下的 xmin。
### 6.2 primary non-slot ownership
无 replication slot 时，feedback 的 owner 是 walsender backend 的 `PGPROC`。
应用路径是：
```text
ProcessStandbyHSFeedbackMessage()
  -> MyProc->xmin = feedback horizon
```
清理路径有两条：
```text
收到 invalid feedback:
  MyProc->xmin = InvalidTransactionId
walsender 进程退出:
  PGPROC 生命周期结束，ProcArray 不再看到这个 xmin
```
non-slot 模式的优点是 stale feedback 不容易长期残留。
缺点是断线窗口内 primary 可能继续 VACUUM，standby 重新连接后需要的旧版本可能已经被清理。
源码在 `ComputeXidHorizons()` 注释中明确承认这种可能：
```text
除非 standby 使用 replication slot 让 xmin 持久化，
否则 walsender 不连续运行时，想保留的数据可能已经丢失；
Hot Standby 通过取消需要已移除数据的 standby query 来处理。
```
这不是数据一致性 bug。
这是 non-slot feedback 的生命周期边界。
### 6.3 primary slot ownership
有 physical replication slot 时，feedback 的 owner 是 slot。
应用路径是：
```text
PhysicalReplicationSlotNewXmin()
  -> slot->data.xmin
  -> slot->effective_xmin
  -> slot->data.catalog_xmin
  -> slot->effective_catalog_xmin
  -> ReplicationSlotMarkDirty()
  -> ReplicationSlotsComputeRequiredXmin()
  -> ProcArraySetReplicationSlotXmin()
```
清理路径包括：
```text
standby 关闭 hot_standby_feedback 并发送 invalid feedback；
slot 被 drop；
slot 被 invalidated 后 ReplicationSlotsComputeRequiredXmin() 忽略它；
slot 状态被新反馈推进到更新的 xmin；
```
slot release 本身不等于清理 `data.xmin`。
这就是 persistent slot 的意义。
它保证 standby 断线后 primary 仍保留必要历史，但也让 stale slot 可以长期持有 primary cleanup horizon。
### 6.4 bloat 的 cleanup 生命周期
即使 feedback 清除了，primary bloat 也不会瞬间消失。
cleanup 需要后续 VACUUM 或 opportunistic pruning 再次访问相关 page。
生命周期大致是：
```text
feedback horizon 清除或前进
  -> 下一次 ComputeXidHorizons() 得到更新 OldestXmin
  -> autovacuum / manual VACUUM 扫描 relation
  -> heap pruning 移除 DEAD tuple 或整理 line pointer
  -> index cleanup 移除不再需要的 index tuple
  -> 空间变成可复用
```
普通 VACUUM 通常把空间还给 relation 内部复用，不一定把文件大小还给操作系统。
如果用户只看 `pg_total_relation_size()`，可能觉得“关闭 feedback 后 bloat 没变”。
更准确的观察是：
```text
n_dead_tup 是否下降；
后续写入是否复用空间；
索引大小是否继续增长；
VACUUM verbose 是否显示 removable tuples 变多；
relfrozenxid age 是否开始下降。
```
## 7. 正确性机制层次
### 7.1 MVCC horizon 不是锁
`hot_standby_feedback` 保护的是 MVCC cleanup horizon。
它不在 primary 上拿 relation lock，也不阻塞 UPDATE / DELETE 本身。
主库仍然可以继续修改数据。
被延迟的是：
```text
删除或更新后产生的旧 tuple version 何时能被物理移除。
```
所以 primary 上可能同时看到：
```text
TPS 正常；
autovacuum 正常启动；
dead tuple 却清不掉；
表和索引逐步变大。
```
这不是 VACUUM 没跑。
可能是 VACUUM 的 `OldestXmin` 被反馈 horizon 拉住。
### 7.2 ProcArrayLock 保证 horizon 读取边界
`ComputeXidHorizons()` 持有 `ProcArrayLock` shared 时读取：
```text
latestCompletedXid
procArray->replication_slot_xmin
procArray->replication_slot_catalog_xmin
PGPROC->xmin
ProcGlobal->xids[]
ProcGlobal->statusFlags[]
```
slot 聚合写入时通过 `ProcArraySetReplicationSlotXmin()` 拿 exclusive lock。
这给 horizon 计算提供同步边界。
不过 `ProcessStandbyHSFeedbackMessage()` 写 non-slot `MyProc->xmin` 时不拿 ProcArrayLock。
源码注释说明这里假设 xid 字段写入是原子的。
如果 moving forward，安全。
如果 moving backward，数据本来已经有风险，因为 VACUUM 可能已经用旧 horizon 做过决定。
这说明 feedback horizon 本身是一个保守同步协议，不是强事务屏障。
### 7.3 catalog horizon 与 data horizon
`catalog_xmin` 不能简单等同于 `xmin`。
普通 data table cleanup 只需要考虑查询是否仍可能看见旧 row version。
catalog cleanup 还要考虑 logical decoding 或 catalog snapshot 是否需要旧 catalog tuple 来解释 WAL 中的历史数据。
当前源码把它们拆开：
```text
slot_xmin:
  影响 data 和 shared horizon。
slot_catalog_xmin:
  影响 catalog 和 shared horizon。
```
`GetReplicationHorizons()` 发送 `shared_oldest_nonremovable_raw`，避免把 local slot catalog horizon 混进 data xmin。
`ProcessStandbyHSFeedbackMessage()` 无 slot 时又不得不把 data/catalog 取更老者写入单一 `MyProc->xmin`。
因此：
```text
slot 模式更持久，也更能表达 data/catalog 分离；
non-slot 模式生命周期短，但 data/catalog 边界更粗。
```
### 7.4 feedback 不能撤销已完成 cleanup
如果 primary 已经根据较新的 horizon 移除了 tuple version，后来的 feedback 不能恢复它。
standby replay 到相关 WAL 时，只能：
```text
等待旧 snapshot 结束；
超过 max_standby_streaming_delay 后取消查询；
如果需要的旧版本已经不可用，就不能继续让查询读。
```
这就是为什么 feedback 必须及时发送，并且为什么 slot 可以提高保护连续性。
但是及时和持久本身就是 bloat 风险。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 feedback 关闭后的 invalid message
用户把 `hot_standby_feedback` 从 on 改为 off 后，walreceiver 在 config reload 路径会重新计算 wakeup 并调用：
```text
XLogWalRcvSendHSFeedback(true)
```
由于 `primary_has_standby_xmin` 可能为 true，即使 `hot_standby_feedback=false`，函数也会继续发送一次 invalid feedback。
primary 端收到后：
```text
ProcessStandbyHSFeedbackMessage()
  -> MyProc->xmin = InvalidTransactionId
  -> 如果有 slot:
       PhysicalReplicationSlotNewXmin(invalid, invalid)
```
边界是：
```text
这需要 walreceiver 仍能连到同一个 upstream 并成功发送消息；
如果 standby 已经离线，non-slot 依赖 walsender 退出清理；
slot 依赖后续 invalid feedback、slot drop 或 invalidation。
```
所以线上关闭 `hot_standby_feedback` 后，应验证 primary 端：
```text
pg_stat_replication.backend_xmin 是否消失或前进；
pg_replication_slots.xmin / catalog_xmin 是否消失或前进；
VACUUM 的 OldestXmin warning 是否停止出现。
```
只改配置不验证 horizon，是不完整操作。
### 8.2 feedback 不合理时被忽略
`ProcessStandbyHSFeedbackMessage()` 会校验 normal feedback 的 epoch。
如果 standby 发来未来的 xid，或 wraparound 解释不合理，primary 直接忽略。
观测上可能出现：
```text
standby 以为自己开启了 feedback；
primary pg_stat_replication.backend_xmin 没有相应变化；
standby 仍有 confl_snapshot。
```
这类问题通常要看 walsender DEBUG2 日志或断点。
普通系统视图不会告诉你“某条 feedback 被忽略的原因”。
### 8.3 Hot Standby 尚未 active
`XLogWalRcvSendHSFeedback()` 在 `HotStandbyActive()` 为 false 时不发送 feedback。
启动早期或恢复尚未允许连接时，standby 不应该向 primary 发布一个还没建立好的 horizon。
这也意味着：
```text
standby 刚启动、slot 状态尚未读完、尚未接受查询时，
不要期待它已经保护了所有下游查询。
```
### 8.4 max_standby_streaming_delay fallback
当 feedback 没有阻止 primary 生成 cleanup WAL，standby replay 仍会进入：
```text
ResolveRecoveryConflictWithSnapshot()
  -> GetConflictingVirtualXIDs(snapshotConflictHorizon, dbOid)
  -> ResolveRecoveryConflictWithVirtualXIDs(...)
```
`standby.c` 中 `GetStandbyLimitTime()` 使用：
```text
last WAL receipt time + max_standby_streaming_delay
```
如果 delay 为 `-1`，表示 wait forever。
否则超过 limit 后发 recovery conflict signal。
这条 fallback 的成本是 standby replay lag。
它不会让 primary 保留更多旧版本。
这就是和 feedback 的本质差异：
```text
max_standby_streaming_delay:
  已经有冲突，standby 等多久。
hot_standby_feedback:
  在 primary cleanup 前，是否让 primary 别生成一部分冲突。
```
### 8.5 slot stale 边界
physical slot 可以让 feedback xmin 跨连接存在。
如果 standby 被下线但 slot 没 drop，primary 可能继续保留：
```text
pg_replication_slots.xmin
pg_replication_slots.catalog_xmin
```
这会持续进入 `ReplicationSlotsComputeRequiredXmin()` 的聚合结果。
如果 slot 已 invalidated，当前源码扫描 slot 时会跳过它。
但 stale active-or-inactive slot 在未 invalidated 前仍可能是最老边界。
诊断时必须区分：
```text
slot 的 restart_lsn 导致 WAL retention；
slot 的 xmin/catalog_xmin 导致 heap/catalog cleanup retention。
```
它们经常一起出现，但不是同一个资源。
## 9. 成本、资源与跨模块传播
### 9.1 primary bloat 成本模型
feedback 拉老 `OldestXmin` 后，成本通常按这些变量放大：
| 变量 | 放大路径 |
| --- | --- |
| standby 最老查询时长 | 查询越久，feedback xmin 越久不能前进。 |
| primary 更新/删除速率 | 每秒产生的 dead tuple 越多，滞留期间积累越快。 |
| 表的 HOT update 比例 | HOT chain 不能及时 prune，会增加 heap page 内遍历成本。 |
| 索引数量 | 每个更新/删除可能留下更多 index cleanup 工作。 |
| autovacuum 周期 | VACUUM 即使频繁运行，也可能因为 horizon 老而只能跳过清理。 |
| catalog churn | DDL、临时对象、分区维护、logical decoding 会让 catalog_xmin 风险更突出。 |
一个简化估算：
```text
primary 每分钟产生 2GB dead tuple；
standby 最老 xmin 被 30 分钟报表钉住；
那么该窗口内可能累积约 60GB 不能及时回收的 heap 旧版本，
还不包括索引和 page fragmentation 成本。
```
这个估算不精确。
但它比只看 `hot_standby_feedback=on/off` 更接近容量判断。
### 9.2 standby cancellation 成本模型
关闭 feedback 后，成本通常按这些变量放大：
| 变量 | 放大路径 |
| --- | --- |
| standby 查询时长 | 越容易跨过 primary cleanup WAL。 |
| primary VACUUM / pruning 频率 | cleanup WAL 越频繁，snapshot conflict 机会越多。 |
| standby replay lag | 已经落后时，`max_standby_streaming_delay` 剩余宽限更少。 |
| 客户端重试能力 | 报表可重试和交互式查询不可重试的业务成本不同。 |
| 查询事务边界 | idle in transaction 会持有 xmin，但没有实际工作产出。 |
如果 standby 只承载短查询，`confl_snapshot` 很低，开启 feedback 可能只是把一个小问题转移成 primary bloat 风险。
如果 standby 承载必须完成的长报表，且主库更新压力较低，开启 feedback 可能是合理选择。
### 9.3 catalog_xmin 的特殊风险
`catalog_xmin` 拉老时，影响的是 catalog tuple cleanup。
常见触发包括：
```text
logical decoding slot；
standby 上同步或保留的 slot；
需要旧 catalog snapshot 解释 WAL 的消费者；
频繁 DDL 或分区 attach/detach。
```
catalog bloat 比普通业务表更容易被忽略。
它可能表现为：
```text
系统表和 toast 表增长；
relcache/syscache 压力增加；
DDL 变慢；
autovacuum 对 catalog 更频繁；
logical slot invalidation 或 conflict 计数出现。
```
本节主线是 physical standby feedback。
但源码里 `GetReplicationHorizons()` 和 `ComputeXidHorizons()` 都明确区分 `xmin` 和 `catalog_xmin`，所以诊断时不能只看 data table bloat。
### 9.4 资源传播路径
这一机制跨越多个模块：
```text
standby backend snapshot
  -> standby ProcArray horizon
  -> walreceiver replication message
  -> primary walsender PGPROC 或 physical slot
  -> primary ProcArray slot aggregate
  -> VACUUM OldestXmin / GlobalVisState
  -> heap pruning / index cleanup / freeze
  -> standby recovery conflict counters
```
涉及的后台进程包括：
```text
standby walreceiver:
  发送 feedback。
primary walsender:
  接收 feedback，发布 xmin。
primary autovacuum worker:
  消费 OldestXmin，决定能否回收。
standby startup process:
  replay WAL，无法避免时取消 conflict query。
logical slot sync / logical decoding 相关 worker:
  可能通过 catalog_xmin 影响同一组 horizon。
```
这解释了为什么单看一个进程状态经常不够。
你必须同时看 primary 和 standby。
## 10. 观测与诊断入口
### 10.1 standby 侧：先确认是否是 snapshot conflict
`system_views.sql` 中 `pg_stat_database_conflicts` 暴露 `confl_tablespace`、`confl_lock`、`confl_snapshot`、`confl_bufferpin`、`confl_deadlock`、`confl_active_logicalslot` 和 `stats_reset`。
本课最关注 `confl_snapshot`，它对应 cleanup WAL 与旧 snapshot 的冲突。
如果开启 feedback 前 `confl_snapshot` 持续增长，开启后增长明显下降，说明一部分 query cancel 被转移到了 primary cleanup horizon。
但其它计数要分开解释：
```text
confl_lock:
  DDL / AccessExclusiveLock 类冲突，feedback 不解决。
confl_bufferpin:
  startup process 等 buffer cleanup lock，feedback 通常不解决。
confl_tablespace:
  DROP TABLESPACE / temp file 类冲突，feedback 不解决。
confl_active_logicalslot:
  logical slot 与 catalog horizon 的特殊边界。
```
standby 当前是谁在钉住 horizon，看 `pg_stat_activity.backend_xmin`：
```sql
SELECT pid, state, xact_start, query_start, backend_xmin,
       now() - COALESCE(xact_start, query_start) AS age,
       wait_event_type, wait_event, left(query, 120) AS query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```
`pg_stat_activity` 在当前 `system_views.sql` 中暴露 `backend_xmin`。
如果这里没有旧 `backend_xmin`，但 primary 仍被钉住，就要转向 slot、prepared transaction、logical decoding 或其它 database 的 horizon。
### 10.2 primary 侧：确认 feedback 是否变成 retention
non-slot feedback 看 `pg_stat_replication.backend_xmin`：
```sql
SELECT pid, application_name, client_addr, state, backend_xmin,
       sent_lsn, write_lsn, flush_lsn, replay_lsn, replay_lag
FROM pg_stat_replication
ORDER BY backend_xmin NULLS LAST;
```
当前 `system_views.sql` 中 `pg_stat_replication` 暴露 `backend_xmin`。
如果这个值很老，说明 walsender `PGPROC->xmin` 正在影响 primary horizon。
slot 场景看 `pg_replication_slots`：
```sql
SELECT slot_name, slot_type, active, active_pid,
       xmin, catalog_xmin, restart_lsn, wal_status,
       safe_wal_size, inactive_since, invalidation_reason
FROM pg_replication_slots
ORDER BY xmin NULLS LAST, catalog_xmin NULLS LAST;
```
`pg_replication_slots` 在当前 `system_views.sql` 中暴露 `xmin` 和 `catalog_xmin`。
三类保留必须分开：
```text
restart_lsn 老:
  WAL 文件保留风险。
xmin 老:
  heap data cleanup bloat 风险。
catalog_xmin 老:
  catalog cleanup 和 logical decoding 相关风险。
```
### 10.3 primary 侧：确认是否已经变成 bloat
常用入口是 `pg_stat_all_tables.n_dead_tup`、`last_autovacuum`、`autovacuum_count`、`pg_total_relation_size()` 和 `age(relfrozenxid)`：
```sql
SELECT c.oid::regclass AS rel,
       pg_total_relation_size(c.oid) AS total_bytes,
       age(c.relfrozenxid) AS xid_age,
       s.n_dead_tup,
       s.last_autovacuum,
       s.autovacuum_count
FROM pg_class c
LEFT JOIN pg_stat_all_tables s ON s.relid = c.oid
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY s.n_dead_tup DESC NULLS LAST
LIMIT 30;
```
`n_dead_tup` 是估计，`pg_total_relation_size()` 不等于可回收 bloat，普通 VACUUM 也不保证文件缩小。
更可靠的是时间线：
```text
standby backend_xmin 变老
  -> primary backend_xmin 或 slot xmin/catalog_xmin 变老
  -> autovacuum 仍运行但 n_dead_tup 不下降
  -> relation / index size 增长
  -> standby confl_snapshot 下降或停止增长
```
如果只看到最后一个点，只能证明 standby 少取消，不能证明系统更健康。
### 10.4 日志、wait event 与不可见状态
standby query 被取消时，日志或客户端可能看到：
```text
canceling statement due to conflict with recovery
```
snapshot conflict 的 DETAIL 通常指向 row versions that must be removed。
startup process 等冲突时，`standby.c` 使用 `WAIT_EVENT_RECOVERY_CONFLICT_SNAPSHOT` 等 wait event。
primary 侧如果 `OldestXmin` 被拉得过老，`vacuum_get_cutoffs()` 可能发出：
```text
cutoff for removing and freezing tuples is far in the past
```
能直接观测的状态包括 conflict 计数、standby `backend_xmin`、primary `backend_xmin`、slot `xmin/catalog_xmin`、dead tuple 估计、relation size、xid age 和日志。
只能推断的是某个具体 standby query 让某张 primary 表多保留了多少 tuple、某次 VACUUM 没删掉 tuple 是否完全由 feedback 导致、feedback 到达与 cleanup 决策之间的竞态。
几乎不可见的状态通常要靠 DEBUG2、断点或插桩，例如 `TransactionIdInRecentPast()` 忽略某条 feedback、`ComputeXidHorizons()` 中每个 PGPROC 的贡献、`PhysicalReplicationSlotNewXmin()` 因 feedback 回退而不更新。
## 11. 常见误区
### 11.1 把 feedback 当成查询加速参数
feedback 不让查询更快，只减少一类 cleanup WAL 导致的 cancel。
如果慢查询来自缺索引、IO、统计信息或执行计划，开启 feedback 不解决根因，还可能让 primary 更慢。
### 11.2 只看 confl_snapshot
`confl_snapshot` 下降说明 standby 少取消，但如果 primary `n_dead_tup`、relation size、slot `xmin` 持续变老，就只是把成本转移了。
正确问题是：
```text
下降的 conflict 是否值得 primary 付出 cleanup 滞留成本？
```
### 11.3 混淆 delay 与 feedback
`max_standby_streaming_delay` 是 standby replay 已经被挡住后等待多久，风险是 replay lag。
`hot_standby_feedback` 是 primary 是否提前保留旧版本，风险是 primary bloat。
delay 设为 `-1` 不会让 primary bloat，feedback 打开也不会解决 lock conflict。
### 11.4 以为没有 slot 就没有 bloat 风险
non-slot feedback 会写 walsender `MyProc->xmin`，只要 walsender 连接持续存在，它同样能拉老 primary horizon。
区别是 non-slot 跟进程生命周期绑定，slot 跟 replication slot 状态绑定并可能跨连接持久化。
### 11.5 以为 slot 只保留 WAL
slot 的 `restart_lsn` 影响 WAL 保留，`xmin/catalog_xmin` 影响 heap/catalog cleanup。
`pg_wal` 爆满和 heap bloat 可能都与 slot 有关，但观测字段不同。
### 11.6 以为关闭 feedback 后空间立刻下降
关闭 feedback 只让 future horizon 可以前进。
已经膨胀的 relation 需要后续 VACUUM/pruning 复用空间；文件缩小还取决于尾页截断、`VACUUM FULL`、`CLUSTER` 或重建。
### 11.7 忽略 catalog_xmin
`catalog_xmin` 老可能表现为 catalog、toast、DDL 或 logical decoding 问题。
源码里 data horizon 和 catalog horizon 明确分开，诊断时只看 `xmin` 会漏掉一类风险。
## 12. 课堂实验
### 实验 1：观察 feedback 如何从 standby 进入 primary horizon
准备一主一备 physical streaming replication，记录是否使用 `primary_slot_name`。
standby 上开启 feedback，并启动一个长 snapshot：
```sql
-- standby
ALTER SYSTEM SET hot_standby_feedback = on;
SELECT pg_reload_conf();
BEGIN;
SELECT count(*) FROM large_table, pg_sleep(120);
```
primary 上观察 non-slot feedback：
```sql
SELECT pid, application_name, backend_xmin, state, replay_lsn
FROM pg_stat_replication;
```
如果使用 physical slot，观察 slot retention：
```sql
SELECT slot_name, active, xmin, catalog_xmin, restart_lsn
FROM pg_replication_slots;
```
源码回扣：
```text
standby XLogWalRcvSendHSFeedback()
  -> primary ProcessStandbyHSFeedbackMessage()
  -> non-slot MyProc->xmin 或 slot data/effective xmin
  -> ComputeXidHorizons()
```
目标是确认 standby 长 snapshot 能在 primary 侧变成 `backend_xmin` 或 slot `xmin/catalog_xmin`。
### 实验 2：对比 conflict 计数与 bloat 增长
primary 准备一张更新频繁的表：
```sql
CREATE TABLE hsfb_t(id int primary key, payload text);
INSERT INTO hsfb_t
SELECT g, repeat('x', 200)
FROM generate_series(1, 100000) g;
UPDATE hsfb_t
SET payload = md5(random()::text)
WHERE id % 10 = 0;
```
standby 开长事务查询，分别测试：
```text
hot_standby_feedback=off
hot_standby_feedback=on
```
观察 standby：
```sql
SELECT datname, confl_snapshot, confl_lock, confl_bufferpin, stats_reset
FROM pg_stat_database_conflicts;
```
观察 primary：
```sql
SELECT relname, n_dead_tup, last_autovacuum, autovacuum_count
FROM pg_stat_all_tables
WHERE relname = 'hsfb_t';
SELECT pg_total_relation_size('hsfb_t');
```
源码回扣：
```text
off:
  primary VACUUM 可以更积极移除旧版本；
  standby replay cleanup WAL 时更可能 ResolveRecoveryConflictWithSnapshot()。
on:
  primary OldestXmin 被拉老；
  HeapTupleSatisfiesVacuum() 更常保持 RECENTLY_DEAD；
  standby confl_snapshot 可能下降。
```
### 实验 3：关闭 feedback 后确认 invalid cleanup
standby 关闭 feedback：
```sql
ALTER SYSTEM SET hot_standby_feedback = off;
SELECT pg_reload_conf();
```
primary 验证 non-slot 与 slot 两条 cleanup 入口：
```sql
SELECT pid, application_name, backend_xmin
FROM pg_stat_replication;
SELECT slot_name, xmin, catalog_xmin
FROM pg_replication_slots;
```
预期：
```text
non-slot:
  backend_xmin 消失或不再保持旧值。
slot:
  xmin/catalog_xmin 应被 invalid feedback 清掉或前进；
  如果仍很老，需要检查 standby 是否仍连接同一 upstream、
  是否真的发送了 feedback、slot 是否 stale。
```
源码断点建议：
```text
standby:
  XLogWalRcvSendHSFeedback
primary:
  ProcessStandbyHSFeedbackMessage
  PhysicalReplicationSlotNewXmin
  ReplicationSlotsComputeRequiredXmin
  ProcArraySetReplicationSlotXmin
```
断点处记录：
```text
feedbackXmin / feedbackCatalogXmin
MyProc->xmin
slot->data.xmin / slot->effective_xmin
procArray->replication_slot_xmin
```
附加练习：把 `hot_standby_feedback=off` 分别配 `max_standby_streaming_delay=30s` 和 `-1`，再与 `hot_standby_feedback=on` 对比。
回到源码时只问一句：`standby.c` 的 `GetStandbyLimitTime()` 是否改变 primary `OldestXmin`？答案应该是否定的。
## 13. 业务取舍框架
### 13.1 三个必须同时看的事实
判断是否开启或关闭 `hot_standby_feedback`，至少同时看三件事。
第一，standby 取消成本：
```text
pg_stat_database_conflicts.confl_snapshot 增长速度；
取消日志出现频率；
被取消查询的业务类型和重试成本；
查询是否可拆分、可重试、可移到离线系统。
```
第二，primary horizon 滞留：
```text
pg_stat_replication.backend_xmin；
pg_replication_slots.xmin；
pg_replication_slots.catalog_xmin；
standby pg_stat_activity.backend_xmin；
长事务 xact_start/query_start。
```
第三，primary cleanup 后果：
```text
n_dead_tup 是否持续上涨；
relation / index size 是否增长；
autovacuum 是否运行但无明显清理效果；
VACUUM cutoff warning 是否出现；
relfrozenxid age 是否接近风险边界。
```
缺任何一项，判断都容易偏。
### 13.2 典型决策矩阵
| 现象 | 更可能的选择 | 理由 |
| --- | --- | --- |
| `confl_snapshot` 高，primary 写入低，报表必须完成 | 开启 feedback 或使用 slot，并限制报表最长事务时间 | standby query continuity 更重要，bloat 可控。 |
| `confl_snapshot` 低，primary bloat 高 | 关闭 feedback 或缩短 standby 长事务 | 没有足够 cancellation 收益，不值得牺牲 cleanup。 |
| `confl_snapshot` 高，同时 primary bloat 已经高 | 不应只调一个参数，先改查询形态或隔离 workload | 两边都在付成本，说明架构容量不匹配。 |
| `confl_lock` / `confl_bufferpin` 高 | 不依赖 feedback 解决 | 这些不是 data xmin cleanup horizon 问题。 |
| slot `catalog_xmin` 老但 `xmin` 不老 | 检查 logical decoding / catalog churn / slot sync | 风险集中在 catalog horizon。 |
| non-slot `backend_xmin` 老但断线后消失 | feedback 生命周期跟 walsender 绑定 | 可接受性取决于断线后 standby query 被取消的代价。 |
| inactive slot 仍有老 `xmin` | drop/reconfigure stale slot 或恢复 standby | slot 正在持久化 cleanup retention。 |
### 13.3 查询时长上限比开关更重要
`hot_standby_feedback` 最怕没有业务上限。
如果 standby 允许任意长事务：
```text
BEGIN;
SELECT ...;
客户端挂起数小时；
```
那么 feedback 会把这个“无上限等待”传导到 primary cleanup。
更稳妥的设计通常是：
```text
给 standby 报表设置 statement_timeout 或 idle_in_transaction_session_timeout；
把超长分析迁移到延迟副本、ETL、快照库或列式系统；
把大查询拆分成可重试批次；
对关键 OLTP primary 禁止无限期 feedback retention；
对必须完成的审计/导出任务单独安排低写入窗口。
```
这不是 PostgreSQL 源码能替业务决定的事。
源码只提供成本转移机制。
系统设计要定义成本上限。
### 13.4 slot 策略
physical slot 与 feedback 经常一起使用，但要区分 `restart_lsn` 的 WAL retention 和 `xmin/catalog_xmin` 的 heap/catalog cleanup retention。
使用 slot 的建议不是“永远开”或“永远关”，而是：
```text
必须监控 pg_replication_slots.xmin/catalog_xmin 和 inactive_since；
必须定义 standby 最大离线时间；
必须有 drop stale slot 或切换 slot 的操作手册；
必须区分 WAL retention 告警和 heap/catalog bloat 告警。
```
临时报表节点未必需要 persistent slot 加 feedback；关键只读节点如果不能接受断线后大量查询取消，slot 的持久保护才可能值得。
## 14. 讨论题
1. `hot_standby_feedback=on` 后 `confl_snapshot` 下降，但 primary 上 `pg_replication_slots.xmin` 老了 3 小时。你会先关 feedback、drop slot，还是先限制 standby 查询时长？为什么？
2. 为什么 `GetReplicationHorizons()` 发送 `shared_oldest_nonremovable_raw`，而不是 `shared_oldest_nonremovable`？
3. non-slot feedback 已经写入 `pg_stat_replication.backend_xmin`。为什么说它仍然不如 slot 持久？
4. `max_standby_streaming_delay=-1` 和 `hot_standby_feedback=on` 都能减少查询取消吗？它们分别把成本放到哪里？
5. standby 上 `confl_lock` 高，但 `confl_snapshot` 很低。开启 feedback 会解决什么，不能解决什么？
6. 一个 stale physical slot 的 `restart_lsn` 很老但 `xmin` 为空，和 `xmin` 很老但 `restart_lsn` 不太老，分别意味着什么资源风险？
7. 为什么关闭 `hot_standby_feedback` 后要验证 invalid feedback 是否到达 primary？slot 和 non-slot 的验证入口有什么不同？
8. 如果 primary bloat 上升，但 standby 没有明显 long query，应该如何从 `backend_xmin`、`catalog_xmin`、prepared transaction、logical slot 方向继续排查？
## 15. 本节小结
`hot_standby_feedback` 的核心链路是：
```text
standby ProcArray horizon
  -> XLogWalRcvSendHSFeedback()
  -> ProcessStandbyHSFeedbackMessage()
  -> MyProc->xmin 或 PhysicalReplicationSlotNewXmin()
  -> ReplicationSlotsComputeRequiredXmin()
  -> ComputeXidHorizons()
  -> GetOldestNonRemovableTransactionId()
  -> VACUUM / heap pruning cleanup decision
```
它减少的是一类 snapshot cleanup conflict。
它付出的代价是 primary cleanup horizon 可能被 standby query 或 slot 钉住。
核心状态边界：
```text
standby backend_xmin:
  谁在 standby 上持有旧 snapshot。
primary pg_stat_replication.backend_xmin:
  non-slot feedback 如何通过 walsender PGPROC 生效。
primary pg_replication_slots.xmin/catalog_xmin:
  slot feedback 如何持久影响 data/catalog cleanup horizon。
pg_stat_database_conflicts.confl_snapshot:
  standby 因 cleanup WAL 取消查询的累计结果。
```
ownership 与 cleanup：
```text
non-slot:
  owner 是 walsender PGPROC；
  invalid feedback 或 walsender 退出清理。
slot:
  owner 是 replication slot；
  invalid feedback、slot drop、slot invalidation 或新 feedback 推进后清理。
primary bloat:
  horizon 前进后仍需后续 VACUUM/pruning 才能回收或复用空间。
```
错误路径与 fallback：
```text
feedback 不合理会被忽略；
Hot Standby 未 active 时不发送；
feedback 未及时到达时仍会走 standby conflict cancel；
max_standby_streaming_delay 只控制 standby 等待，不控制 primary cleanup；
feedback 不能解决 lock、drop、tablespace、buffer pin 等非 snapshot horizon 冲突。
```
可迁移的系统规律：
```text
任何“减少读侧取消”的机制，如果通过延长写侧 cleanup horizon 实现，
就必须同时给 retention 时长、写入速率和 cleanup 后果设置观测与上限。
没有上限的 reader friendliness，最终会变成 writer-side storage 和 vacuum debt。
```
最终判断依赖 workload。
对于低写入、强报表一致性需求的系统，feedback 可能是合理成本。
对于高更新 OLTP primary，长期 feedback xmin 滞留通常比偶发 standby query cancel 更危险。
源码不会替你选择。
它只把取舍清楚地暴露在 `confl_snapshot`、`backend_xmin`、`xmin/catalog_xmin` 和 VACUUM cleanup 行为里。
