# PostgreSQL slot xmin / catalog_xmin 与 VACUUM 清理边界


## 课程定位

前置知识：已经理解 MVCC snapshot、`PGPROC->xmin`、replication slot 的 `restart_lsn` / `confirmed_flush_lsn` 生命周期，以及 WAL sender / WAL receiver 的基本反馈链路。

本节唯一主问题：
```text
logical slot 为什么会发布 catalog_xmin，physical standby feedback 为什么会影响 xmin horizon，
哪些场景会让 VACUUM 无法移除旧 tuple 或 catalog 版本？
```
核心矛盾：VACUUM 想尽快移除旧 tuple version、推进 visibility map 和 freeze 边界；logical decoding、standby query、下游 slot 和长事务却可能仍需要用一个旧时间点解释数据或 catalog。PostgreSQL 不能把所有旧版本无限保留，也不能在 consumer 断开、standby 延迟或 catalog 变更频繁时提前清掉仍然需要的版本。

本节的运行模型是：
```text
observer 发布自己仍需要的最老 XID
  -> slot 把 xmin / catalog_xmin 分成 data 与 catalog 两条边界
  -> ReplicationSlotsComputeRequiredXmin() 聚合 effective 值
  -> ProcArray 把 slot 边界混入 relation-sensitive horizon
  -> VACUUM 只移除早于 OldestXmin 的旧版本
```
学完后应能独立判断：
```text
pg_replication_slots.catalog_xmin 老，为什么普通表未必被拖住；
hot_standby_feedback 开启后，为什么 primary 上 VACUUM 可能不再清掉旧版本；
logical consumer 停止确认后，为什么 catalog_xmin 不前进；
pg_stat_all_tables.n_dead_tup 增长时，如何区分长事务、slot 和 standby feedback；
哪些状态能从 SQL 直接看到，哪些只能从源码和现象推断。
```
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。

## 1. 本节在总主线中的位置

上一节讲 replication slot 的类型、持久化、失效和生命周期。 那一节回答的是：
```text
slot 如何存在，如何 active，如何保留 WAL，如何落盘，如何失效。
```
本节只抓住 slot 对 VACUUM 的一条影响链：
```text
slot 或 standby 发布 xmin
  -> ProcArray 形成 cleanup horizon
  -> VACUUM 决定旧版本能否移除
```
这里不要把 slot 想成一个复制进度表。 它同时是一个 resource retention contract。 这个 contract 说：
```text
在我声明的 xmin / catalog_xmin 之前，系统不能假设所有观察者都不再需要旧版本。
```
本节也不会完整展开 logical decoding 的 snapshot builder。 后续 catalog visibility 课程会专门讲 `SnapBuild` 如何解释旧 catalog。 本节只解释为什么那个需求必须向 VACUUM 暴露成 `catalog_xmin`。

## 2. 核心矛盾与一句话运行模型

一个被 UPDATE / DELETE 替换掉的 tuple version 对当前事务可能已经不可见。 但 VACUUM 的问题更强：
```text
有没有任何仍然合法的观察路径可能需要它？
```
这些观察路径不只有普通 SQL 查询。 它们还包括：
```text
logical decoding 需要旧 catalog tuple 解释 WAL；
standby 上的查询需要 primary 保留还可能可见的旧 data tuple；
standby 上的 logical slot 还可能把自己的 catalog_xmin 通过 feedback 传回 primary；
consumer 断开后 persistent slot 仍保留最后发布的边界；
长事务和 prepared transaction 继续钉住普通 snapshot horizon。
```
一句话运行模型：
```text
logical slot 用 catalog_xmin 保护历史 catalog 可见性；
physical standby feedback 用 xmin / catalog_xmin 把 standby 的可见性需求反馈给 primary；
slot.c 把每个 slot 的 effective_xmin / effective_catalog_xmin 聚合到 ProcArray；
procarray.c 根据 relation 类型计算 data / catalog / shared horizon；
vacuum.c 和 vacuumlazy.c 用这些 horizon 决定哪些旧 tuple 仍只能算 recently dead。
```
这条链上最重要的分离是：
```text
xmin:
  保护普通 data tuple 的旧版本。

catalog_xmin:
  额外保护 catalog tuple 的旧版本。
```
如果 PostgreSQL 只用一个最保守 xmin，logical decoding 的 catalog 需求会拖住所有普通表。 如果完全不发布 `catalog_xmin`，logical decoding 可能读到 WAL 中一条旧数据变更，却找不到当时有效的 relation schema、type、replica identity 或 catalog tuple。 所以本节的核心 tension 是：
```text
普通表 cleanup 想尽量激进
  vs
logical / standby observer 需要历史可见性不能被提前破坏
```
PostgreSQL 的折中是拆出 relation-sensitive horizon。 普通 data relation 主要受 `slot_xmin` 影响。 catalog relation 额外受 `slot_catalog_xmin` 影响。 shared relation 需要更保守，因为它跨 database 可见。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/slot.h` | `data.xmin`、`data.catalog_xmin`、`effective_xmin`、`effective_catalog_xmin`、candidate 字段的语义边界。 |
| 2 | `src/backend/replication/logical/logical.c` | logical slot 初始化、`LogicalIncreaseXminForSlot()`、`LogicalConfirmReceivedLocation()`。 |
| 3 | `src/backend/replication/logical/snapbuild.c` | `SnapBuildProcessRunningXacts()` 如何把 snapshot xmin 变成候选 `catalog_xmin`。 |
| 4 | `src/backend/replication/logical/decode.c` | fast-forward 时也要 build snapshot，因为 candidate `catalog_xmin` 依赖 snapshot xmin。 |
| 5 | `src/backend/replication/walreceiver.c` | `XLogWalRcvSendHSFeedback()` 如何发送 standby 的 xmin / catalog_xmin。 |
| 6 | `src/backend/replication/walsender.c` | `ProcessStandbyHSFeedbackMessage()`、`PhysicalReplicationSlotNewXmin()` 如何在 primary 应用 feedback。 |
| 7 | `src/backend/replication/slot.c` | `ReplicationSlotsComputeRequiredXmin()` 如何聚合所有 slot 的 effective horizon。 |
| 8 | `src/backend/storage/ipc/procarray.c` | `ComputeXidHorizons()`、`GetReplicationHorizons()`、`GetOldestNonRemovableTransactionId()`。 |
| 9 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()` 如何取得 `OldestXmin`。 |
| 10 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 如何用 `OldestXmin` 和 visibility 判断 dead / recently dead。 |
| 11 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesVacuum()` 如何把 `OldestXmin` 变成 tuple 状态。 |
| 12 | `src/backend/catalog/system_views.sql` | `pg_replication_slots` 与 `pg_stat_all_tables` 的 SQL 观测入口。 |
推荐阅读顺序不是从 VACUUM 的全部流程开始。 VACUUM 代码中还有 page skipping、index cleanup、freeze、eager scan、visibility map。 本节先从 observer 发布需求开始。 然后再看 VACUUM 消费需求。 这能避免把现象误解释为“VACUUM 不工作”。 大多数 bloat 现场的真实问题不是 VACUUM 没跑。 而是 VACUUM 正确地判断旧版本还不能删。

## 4. 关键数据结构与状态

`slot.h` 里需要区分两组字段。 第一组是持久 slot data：
```text
ReplicationSlotPersistentData.xmin
ReplicationSlotPersistentData.catalog_xmin
ReplicationSlotPersistentData.restart_lsn
ReplicationSlotPersistentData.confirmed_flush
```
这些字段跟随 persistent slot 落盘。 `pg_replication_slots.xmin` 和 `pg_replication_slots.catalog_xmin` 展示的是这组 `data` 字段的拷贝。 第二组是当前真正发布给全局 horizon 的内存态：
```text
ReplicationSlot.effective_xmin
ReplicationSlot.effective_catalog_xmin
```
它们不等于 SQL view 中一定可见的字段。 这一点在 logical slot 初始 snapshot 特别重要。 创建 logical slot 时，如果需要导出 data snapshot，代码可能临时设置 `effective_xmin`。 但 `data.xmin` 仍可能是 invalid。 SQL 侧看 `pg_replication_slots.xmin` 可能是 NULL。 VACUUM 却可能已经被 `effective_xmin` 钉住。 所以诊断时不能把 `pg_replication_slots` 当成完整 shared memory dump。 它是一个有选择的投影。

### 4.1 `xmin` 与 `catalog_xmin`

`xmin` 保护 data tuple 旧版本。 它回答：
```text
有没有 observer 仍可能把某个 data deleting XID 看成 running 或可见？
```
`catalog_xmin` 保护 catalog tuple 旧版本。 它回答：
```text
logical decoding 或 downstream catalog observer 是否仍需要旧 catalog 版本？
```
这两个值都用 XID 表示。 但它们的 relation 作用域不同。 `ComputeXidHorizons()` 中普通 data horizon 只应用 `slot_xmin`。 catalog horizon 额外应用 `slot_catalog_xmin`。 shared horizon 也要应用 `slot_catalog_xmin`，因为 shared relation 里也可能有 catalog 语义。

### 4.2 `data.*` 与 `effective_*`

`data.xmin` / `data.catalog_xmin` 是 crash 后可恢复的 slot contract。 `effective_xmin` / `effective_catalog_xmin` 是本次运行中真正参与 ProcArray 聚合的边界。 logical slot 对这两个层次特别谨慎。 当 consumer 确认足够的 LSN 后，`LogicalConfirmReceivedLocation()` 先更新并保存 `data.catalog_xmin`。 只有保存成功后，才把 `effective_catalog_xmin` 推进到新的 `data.catalog_xmin`。 原因是：
```text
如果先放宽 effective_catalog_xmin，
VACUUM 可能清掉旧 catalog tuple；
随后 primary crash；
重启后 slot data 文件还记录旧 catalog_xmin；
logical decoding 会以为旧 catalog 仍可用，但实际已经被清理。
```
这个顺序是 correctness，不是保守优化。 physical standby feedback 的风险模型不同。 `PhysicalReplicationSlotNewXmin()` 注释说明，physical replication 不需要 logical 那种 interlock。 漏掉一次 `xmin` 前进的最坏后果主要是 standby query cancellation。 不是 logical decoding 输出错误。 因此 physical slot 可以同时设置 `data.xmin` 和 `effective_xmin`。

### 4.3 candidate 字段

logical slot 还有一组候选字段：
```text
candidate_catalog_xmin
candidate_xmin_lsn
candidate_restart_lsn
candidate_restart_valid
```
它们表示：
```text
decoder 已经知道未来可以把 catalog_xmin 推进到某个 XID；
但必须等 consumer 确认收到对应 LSN 后才能真正应用。
```
为什么不能立刻应用？ 因为 consumer 可能崩溃或断开。 如果 server 先放宽 `catalog_xmin`，consumer 实际没有持久保存对应输出，那么重连后需要从旧位置重新解码。 此时旧 catalog tuple 可能已经被 VACUUM 清掉。 所以 `candidate_catalog_xmin` 是 logical decoding 的 ack boundary。 它连接的是 WAL 位置和 catalog visibility。

### 4.4 ProcArray 中的 slot 聚合值

`procarray.c` 的 `ProcArrayStruct` 保存：
```text
replication_slot_xmin
replication_slot_catalog_xmin
```
这不是某个 slot 的字段。 这是所有 valid slot 的最小 effective horizon。 计算入口是：
```text
ReplicationSlotsComputeRequiredXmin()
  -> 扫描 ReplicationSlotCtl->replication_slots
  -> 跳过未使用 slot
  -> 跳过 invalidated slot
  -> 读取 effective_xmin / effective_catalog_xmin
  -> 取最老值
  -> ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin)
```
VACUUM 不直接扫描所有 slot。 VACUUM 只问 ProcArray：
```text
当前 relation 的 oldest non-removable XID 是什么？
```
这样 slot 更新路径把复制状态压缩成 ProcArray 可消费的边界。

### 4.5 VACUUM 的 `OldestXmin`

`vacuum.c` 的 `vacuum_get_cutoffs()` 调用：
```text
GetOldestNonRemovableTransactionId(rel)
```
返回值写入：
```text
cutoffs->OldestXmin
```
`HeapTupleSatisfiesVacuum()` 用这个值决定：
```text
deleting XID < OldestXmin:
  old version 可以成为 HEAPTUPLE_DEAD

deleting XID >= OldestXmin:
  old version 仍是 HEAPTUPLE_RECENTLY_DEAD
```
`RECENTLY_DEAD` 不是“刚刚死掉”。 它的语义是：
```text
删除事务已经提交，
但还不能证明所有 observer 都不需要这个旧版本。
```
因此 bloat 现场看到 `n_dead_tup` 增长时，要问：
```text
是谁让 OldestXmin 不能前进？
```
而不是先问：
```text
为什么 autovacuum 没有把它删掉？
```

## 5. 主流程源码 walkthrough

本节主流程从 logical slot 和 physical feedback 两条发布路径开始。 两条路径最后汇入同一个 slot aggregation 和 ProcArray horizon。

### 5.1 logical slot 创建时先发布 `catalog_xmin`

logical slot 初始化入口在 `logical.c`：
```text
CreateInitDecodingContext()
```
这条路径首先做 slot 类型和 database 检查。 然后设置 output plugin 名称。 再进入最关键的 xmin horizon 初始化。 源码中的顺序是：
```text
LWLockAcquire(ReplicationSlotControlLock, LW_EXCLUSIVE)
LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE)

xmin_horizon = GetOldestSafeDecodingTransactionId(!need_full_snapshot)

slot->effective_catalog_xmin = xmin_horizon
slot->data.catalog_xmin = xmin_horizon
if need_full_snapshot:
  slot->effective_xmin = xmin_horizon

ReplicationSlotsComputeRequiredXmin(true)

LWLockRelease(ProcArrayLock)
LWLockRelease(ReplicationSlotControlLock)
```
这里同时拿 `ReplicationSlotControlLock` 和 `ProcArrayLock`。 原因不是为了普通互斥。 原因是防止出现这样一个窗口：
```text
decoder 计算出一个安全 xmin_horizon
  -> 还没把它发布到 slot / ProcArray
  -> 并发 VACUUM 计算 horizon
  -> VACUUM 清掉 decoder 需要的旧版本
```
`GetOldestSafeDecodingTransactionId()` 的注释也要求调用者持有 `ProcArrayLock`。 它还会拿 `XidGenLock`，避免计算期间新事务分配 XID 后破坏 safe horizon。 这一步回答了第一个关键问题：
```text
logical slot 为什么一创建就可能让 VACUUM 更保守？
```
因为 decoder 需要一个从 WAL 中构造出来的历史观察点。 在那个观察点安全发布前，VACUUM 不能抢先清理。

### 5.2 为什么 logical slot 主要发布 `catalog_xmin`

logical decoding 输出 data changes。 但 decoding 旧 WAL 时，真正必须跨时间读取的是 catalog。 例如 WAL 里只有 tuple bytes 和 relation 信息。 decoder 要把它解释成输出插件能理解的 change。 这通常需要读取：
```text
relation schema
column metadata
type metadata
replica identity
toast / index / catalog dependencies
```
这些元数据在用户事务期间可能已经被 DDL 改过。 如果 VACUUM 移除了旧 catalog tuple，decoder 可能无法重建当时的语义。 所以 logical slot 的长期 xmin contract 主要是：
```text
保留 catalog relation 中足够旧的版本。
```
这就是 `catalog_xmin`。 普通 data tuple 旧版本不需要被 logical slot 长期保留。 logical decoding 读取的是 WAL，不是回表扫描旧 data tuple。 例外是初始 snapshot 导出。 如果创建 slot 时需要一个 full data snapshot，`CreateInitDecodingContext()` 会临时设置 `effective_xmin`。 这个临时 data xmin 保护的是：
```text
客户端拿着初始 snapshot 扫描表数据期间，旧 data tuple 不能被清掉。
```
当 slot release 且没有持久 `data.xmin` 时，`ReplicationSlotRelease()` 会清掉这个临时 `effective_xmin` 并重新计算 required xmin。

### 5.3 snapbuild 如何推进候选 `catalog_xmin`

logical decoding 持续读取 WAL。 在 `decode.c` 中，heap record 处理有一个容易忽略的注释。 即使处于 fast-forward 模式，也必须让 `SnapBuildProcessChange()` 构造 base snapshot。 原因是：
```text
candidate catalog_xmin 依赖 snapshot 的 xmin。
```
当 WAL 中出现 running xacts record 时，`snapbuild.c` 的 `SnapBuildProcessRunningXacts()` 会计算：
```text
xmin = ReorderBufferGetOldestXmin(builder->reorder)
if xmin invalid:
  xmin = running->oldestRunningXid

LogicalIncreaseXminForSlot(lsn, xmin)
```
这一步不是立刻放宽全局 horizon。 它只是告诉 slot：
```text
当 consumer 确认收到 current_lsn 以后，
catalog_xmin 可以推进到 xmin。
```
`LogicalIncreaseXminForSlot()` 进入 slot mutex。 它先检查新的 xmin 是否真的比当前 `data.catalog_xmin` 更新。 如果当前 LSN 已经不超过 `confirmed_flush`，候选值可以直接应用。 否则它写入：
```text
candidate_catalog_xmin = xmin
candidate_xmin_lsn = current_lsn
```
如果已有候选还没应用，函数不会无限覆盖。 这防止 ack 太慢时 candidate 一直被刷新，最后永远无法满足应用条件。

### 5.4 consumer ack 后才应用 `catalog_xmin`

consumer 的确认位置最终进入：
```text
LogicalConfirmReceivedLocation(lsn)
```
这条路径先避免 `confirmed_flush` 后退。 然后判断：
```text
candidate_xmin_lsn <= lsn
```
如果成立，slot 可以把：
```text
data.catalog_xmin = candidate_catalog_xmin
candidate_catalog_xmin = InvalidTransactionId
candidate_xmin_lsn = InvalidXLogRecPtr
```
但这仍然还没有放宽 VACUUM。 接下来必须：
```text
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```
只有 slot state 已经安全写到磁盘后，代码才执行：
```text
effective_catalog_xmin = data.catalog_xmin
ReplicationSlotsComputeRequiredXmin(false)
```
这一段是本节最重要的 crash-safety 顺序。 `data.catalog_xmin` 是“重启后我承认不再需要更早 catalog”的持久证据。 `effective_catalog_xmin` 是“现在允许 VACUUM 以此为边界”的运行时发布。 持久证据必须先于运行时放宽。

### 5.5 physical standby feedback 的发送路径

standby 侧发送 feedback 的入口在 `walreceiver.c`：
```text
XLogWalRcvSendHSFeedback()
```
它受两个配置影响：
```text
wal_receiver_status_interval
hot_standby_feedback
```
如果用户关闭 feedback，walreceiver 会发送一次 final message。 这次消息把 `xmin` 和 `catalog_xmin` 都设为 invalid。 目的是让 primary 忘掉之前的 standby xmin。 如果 feedback 开启，standby 会调用：
```text
GetReplicationHorizons(&xmin, &catalog_xmin)
```
这里的命名很关键。 `GetReplicationHorizons()` 不直接返回最保守的 `shared_oldest_nonremovable`。 它返回：
```text
xmin = horizons.shared_oldest_nonremovable_raw
catalog_xmin = horizons.slot_catalog_xmin
```
`shared_oldest_nonremovable_raw` 是未混入 slot `catalog_xmin` 的 shared horizon。 这么做是为了把 data 和 catalog 分开发给 upstream。 如果 primary 使用 slot 接收 feedback，就可以只让 `catalog_xmin` 影响 catalog relation。 否则 catalog 需求会不必要地拖住普通 data relation。 发送消息时，walreceiver 还要携带 epoch。 primary 会用 epoch 检查反馈 XID 是否处于 recent past。

### 5.6 primary 如何应用 feedback

primary 上的入口在 `walsender.c`：
```text
ProcessStandbyHSFeedbackMessage()
```
它先解析：
```text
replyTime
feedbackXmin
feedbackEpoch
feedbackCatalogXmin
feedbackCatalogEpoch
```
然后处理三种情况。 第一种，两个反馈值都不是 normal XID。 这通常表示 downstream 关闭了 `hot_standby_feedback`。 primary 会：
```text
MyProc->xmin = InvalidTransactionId
if MyReplicationSlot != NULL:
  PhysicalReplicationSlotNewXmin(invalid, invalid)
return
```
第二种，反馈值看起来不在 recent past。 primary 直接忽略。 这是防止 standby 发来未来 XID 或已经 wraparound 风险的旧值。 第三种，反馈值有效。 如果 walsender 持有 replication slot：
```text
PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)
```
否则：
```text
MyProc->xmin = min(feedbackXmin, feedbackCatalogXmin)
```
没有 slot 时，primary 不能在 `PGPROC->xmin` 中分别表达 data 与 catalog horizon。 所以它只能把两者折叠成一个更保守的 xmin。 如果 walsender 没有 database 连接，启动时还会设置：
```text
PROC_AFFECTS_ALL_HORIZONS
```
这样 physical replication client 的 feedback xmin 会影响所有 database 的 horizon。 这解释了为什么一个 standby 查询可能让 primary 上多个数据库的 VACUUM 变保守。

### 5.7 `PhysicalReplicationSlotNewXmin()` 更新 slot

`PhysicalReplicationSlotNewXmin()` 持有 slot mutex。 它先清空：
```text
MyProc->xmin = InvalidTransactionId
```
因为如果有 slot，就不再通过 walsender 的 `PGPROC->xmin` 发布同一份需求。 然后它更新：
```text
slot->data.xmin = feedbackXmin
slot->effective_xmin = feedbackXmin
slot->data.catalog_xmin = feedbackCatalogXmin
slot->effective_catalog_xmin = feedbackCatalogXmin
```
只要发生变化，就调用：
```text
ReplicationSlotMarkDirty()
ReplicationSlotsComputeRequiredXmin(false)
```
这里没有像 logical slot 一样先 `ReplicationSlotSave()` 再推进 `effective_*`。 源码注释明确区分了风险：
```text
physical feedback 漏掉一次 increase 的后果是 standby query 可能取消；
logical decoding 漏掉 catalog 历史的后果是输出错误。
```
这不是 physical slot 不重要。 而是 correctness model 不同。

### 5.8 slot 聚合到 ProcArray

所有发布路径最终都汇入 `slot.c`：
```text
ReplicationSlotsComputeRequiredXmin()
```
它扫描 `ReplicationSlotCtl->replication_slots`。 对每个 in-use slot：
```text
读取 effective_xmin
读取 effective_catalog_xmin
读取 invalidated 标记
```
invalidated slot 不再参与。 因为它已经不能保证可继续解码或复制。 对有效 slot，函数取最老的 effective data xmin 和 catalog xmin：
```text
agg_xmin = min(valid effective_xmin)
agg_catalog_xmin = min(valid effective_catalog_xmin)
```
然后调用：
```text
ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin, already_locked)
```
`already_locked` 为 true 时，调用方已经按顺序持有：
```text
ReplicationSlotControlLock
ProcArrayLock
```
这在 logical slot 初始化时很关键。 普通推进路径用 shared `ReplicationSlotControlLock`，再在 `ProcArraySetReplicationSlotXmin()` 中拿 exclusive `ProcArrayLock`。 源码允许并发计算导致 required xmin 短暂回退。 这是保守的。 回退会让 VACUUM 少删一点。 不会让 VACUUM 提前删掉需要的 tuple。

### 5.9 ProcArray 计算 relation-sensitive horizon

`procarray.c` 的核心函数是：
```text
ComputeXidHorizons()
```
它先在 `ProcArrayLock` 下读取：
```text
latestCompletedXid
replication_slot_xmin
replication_slot_catalog_xmin
每个 PGPROC 的 xid / xmin / statusFlags
```
普通 backend 的 `proc->xmin` 和 active `xid` 都会参与。 原因是某个 backend 可能还没设置 snapshot xmin，但已经分配了 XID。 或者已经有 snapshot xmin，但还没有 XID。 `ComputeXidHorizons()` 对每个 backend 取更老的一个。 然后根据 status flags 决定是否影响 data horizon。 `PROC_IN_VACUUM` 可以被普通 tuple cleanup 忽略。 `PROC_IN_LOGICAL_DECODING` 的 xmin 也不按普通 query 处理，因为 logical decoding 通过 slot horizon 保护。 `PROC_AFFECTS_ALL_HORIZONS` 会让一个 backend 的 xmin 影响所有 database。 这正是 physical walsender feedback 的跨 database 边界。 锁释放后，函数把 slot horizon 混入各类结果：
```text
shared_oldest_nonremovable = older(shared_oldest_nonremovable, slot_xmin)
data_oldest_nonremovable = older(data_oldest_nonremovable, slot_xmin)

shared_oldest_nonremovable_raw = shared_oldest_nonremovable

shared_oldest_nonremovable = older(shared_oldest_nonremovable, slot_catalog_xmin)
catalog_oldest_nonremovable = data_oldest_nonremovable
catalog_oldest_nonremovable = older(catalog_oldest_nonremovable, slot_catalog_xmin)
```
这里完成了 `xmin` 和 `catalog_xmin` 的分流。 普通 data relation 不应用 `slot_catalog_xmin`。 catalog relation 应用。 shared relation 应用。

### 5.10 VACUUM 消费 horizon

`vacuum_get_cutoffs()` 进入时，目标 relation 已知。 它调用：
```text
GetOldestNonRemovableTransactionId(rel)
```
该函数内部再次调用 `ComputeXidHorizons()`。 然后按 relation 类型选择：
```text
shared relation:
  shared_oldest_nonremovable

catalog relation 或 logical decoding 可访问 relation:
  catalog_oldest_nonremovable

普通非 local relation:
  data_oldest_nonremovable

temp relation:
  temp_oldest_nonremovable
```
返回的值成为 `cutoffs->OldestXmin`。 `HeapTupleSatisfiesVacuum()` 再用它处理已经提交删除的 tuple：
```text
if dead_after < OldestXmin:
  HEAPTUPLE_DEAD
else:
  HEAPTUPLE_RECENTLY_DEAD
```
`vacuumlazy.c` 的扫描路径会把 `HEAPTUPLE_RECENTLY_DEAD` 计入不能移除的旧版本。 这就是可见现象：
```text
VACUUM 跑了；
pg_stat_all_tables.last_autovacuum 更新了；
n_dead_tup 仍然增长或回落很慢；
因为 OldestXmin 被 slot / feedback / 长事务钉住。
```

## 6. 生命周期 / ownership / cleanup

slot xmin 的 owner 不是 VACUUM。 VACUUM 只是 consumer。 发布者可能是 logical decoding backend、physical walsender、walreceiver 反馈链路或 slot sync。 logical slot 创建时：
```text
backend acquire slot
  -> CreateInitDecodingContext()
  -> 发布 data.catalog_xmin / effective_catalog_xmin
  -> 如果需要 full snapshot，临时发布 effective_xmin
```
logical decoding 运行时：
```text
snapbuild 根据 WAL running xacts 计算候选 catalog_xmin
  -> consumer ack LSN
  -> LogicalConfirmReceivedLocation()
  -> 保存 data.catalog_xmin
  -> 推进 effective_catalog_xmin
  -> 重算 ProcArray slot horizon
```
consumer 断开时：
```text
persistent slot 保留 data.catalog_xmin、restart_lsn、confirmed_flush
active 变 false
effective_catalog_xmin 仍继续参与 ProcArray horizon
```
这就是 inactive logical slot 仍可能导致 catalog bloat 的原因。 inactive 不等于无影响。 slot drop 或 invalidation 才会解除对应保留边界。 physical standby feedback 运行时：
```text
standby walreceiver 周期性发送 xmin / catalog_xmin
primary walsender 接收
if using slot:
  写入 physical slot 的 data/effective xmin
else:
  写入 walsender MyProc->xmin
```
关闭 `hot_standby_feedback` 时：
```text
standby 发送 invalid xmin / catalog_xmin
primary 清掉 walsender xmin 或 slot xmin
```
但这依赖消息能到达 primary。 如果连接已经断开，primary 侧持有 slot 的情况会继续保留最后一次 slot xmin。 这也是 slot 与裸 walsender feedback 的一个重要区别。 crash recovery 时：
```text
persistent slot 从磁盘恢复 data.xmin / data.catalog_xmin
slot->effective_xmin = cp.slotdata.xmin
slot->effective_catalog_xmin = cp.slotdata.catalog_xmin
ReplicationSlotsComputeRequiredXmin(false)
```
因此持久化顺序决定重启后是否还能正确保护历史。

## 7. 正确性机制层次

本节的正确性不是单个锁保证的。

它由几个层次叠加。

第一层是发布 interlock。

logical slot 初始化时同时持有：
```text
ReplicationSlotControlLock
ProcArrayLock
```
然后才计算 safe decoding XID 并发布到 slot。

这收住了 safe horizon 计算与 VACUUM 之间的窗口。

第二层是持久化顺序。

logical slot 推进 `catalog_xmin` 时：
```text
先更新 data.catalog_xmin
再 ReplicationSlotSave()
最后更新 effective_catalog_xmin
```
这保证 crash 后不会丢失“已经允许 VACUUM 清理”的证据。

第三层是 relation-sensitive horizon。

`ComputeXidHorizons()` 不把所有关系都当成全局 shared data。它按 shared、catalog、data、temp 分开。这样可以在保护 logical catalog correctness 的同时，避免过度拖住普通 data table。

第四层是 conservative fallback。

`ReplicationSlotsComputeRequiredXmin()` 并发重算可能短暂得到更老的聚合 xmin。这会推迟 cleanup，但不会导致 early removal。PostgreSQL 在 cleanup horizon 上宁可保守。

第五层是 invalidation。

当 slot 因 WAL removed、horizon conflict、wal_level 降低或 idle timeout 等原因失效后，slot 不再继续参与 required xmin 计算。此时 correctness contract 已经断开，系统不能再假装还能安全解码。

## 8. 让 VACUUM 无法移除旧版本的典型边界

这节的现象可以压缩成一句话：
```text
只要某个 observer 发布的 horizon 早于 tuple 的 deleting XID，
VACUUM 就不能把那个旧版本从 recently dead 推进成 dead。
```
但不同 observer 的症状不同。

### 8.1 长事务

普通长事务通过 `MyProc->xmin` 影响 ProcArray。 典型现场：
```text
pg_stat_activity.backend_xmin 很老
pg_replication_slots 没有可疑 xmin / catalog_xmin
pg_stat_all_tables.n_dead_tup 增长
```
这不是 slot 问题。 VACUUM 被 active / registered snapshot 钉住。 `ComputeXidHorizons()` 扫描 PGPROC 时会把这个 xmin 纳入 data horizon。

### 8.2 logical consumer 断开

persistent logical slot inactive 后仍保留：
```text
restart_lsn
confirmed_flush_lsn
catalog_xmin
```
如果 consumer 长时间不确认，`candidate_catalog_xmin` 不能应用。 SQL view 只能看到 `catalog_xmin` 不前进。 看不到 `candidate_catalog_xmin`。 典型现场：
```text
pg_replication_slots.slot_type = logical
active = false
catalog_xmin 很老
confirmed_flush_lsn 长时间不前进
catalog 表或 user_catalog_table 膨胀
```
普通用户表是否被拖住，取决于是否还有 `xmin` 或 full snapshot 的临时 `effective_xmin`。 不能只因为 `catalog_xmin` 老，就断言所有 data table 都会 bloat。

### 8.3 logical consumer 在线但 ack 慢

consumer 在线不代表 `catalog_xmin` 快速前进。 `LogicalIncreaseXminForSlot()` 只是产生候选。 `LogicalConfirmReceivedLocation()` 需要收到足够 LSN 的确认。 如果 output plugin、网络、下游事务应用或 flush 策略慢，server 端会保留旧 `catalog_xmin`。 典型现场：
```text
active = true
active_pid 有值
confirmed_flush_lsn 落后
catalog_xmin 老
restart_lsn 也可能落后
```
这时 VACUUM 被拖住是正确行为。 真正需要处理的是 consumer 端吞吐或确认策略。

### 8.4 initial snapshot 导出

创建 logical slot 并导出 snapshot 时，slot 可能临时发布 `effective_xmin`。 这个值不一定出现在 `pg_replication_slots.xmin`。 现象可能是：
```text
pg_replication_slots.xmin 为 NULL
但 VACUUM 仍然不能移除某些旧 data tuple
```
源码解释在 `CreateInitDecodingContext()` 和 `ReplicationSlotRelease()`。 这是 SQL view 投影不完整的典型例子。 需要结合创建 slot 的 session、是否导出 snapshot、以及当前是否仍持有 slot 来判断。

### 8.5 hot_standby_feedback 开启

standby 上有长查询时，本地 horizon 会变老。 walreceiver 把它通过 `XLogWalRcvSendHSFeedback()` 发给 primary。 primary 的 walsender 把它写入 slot 或 `MyProc->xmin`。 典型现场：
```text
primary 上 pg_stat_all_tables.n_dead_tup 增长
primary 上没有本地长事务
standby 上有长查询或 replay 延迟
hot_standby_feedback = on
```
这不是 primary 的 VACUUM 失效。 这是 primary 为了减少 standby cleanup conflict 主动保留旧版本。 如果 primary walsender 使用 physical slot，反馈会持久在 slot 的 `xmin` / `catalog_xmin` 中。 如果没有 slot，反馈存在 walsender 的 `PGPROC->xmin` 中，连接断开后随进程结束释放。

### 8.6 standby 上的 logical slot 反向影响 primary

一个容易漏掉的场景是：
```text
primary -> physical standby
standby 上创建 logical slot
standby 开启 hot_standby_feedback
```
standby 的 local logical slot 会让 standby 的 `slot_catalog_xmin` 变老。 `GetReplicationHorizons()` 会把这个 `slot_catalog_xmin` 作为 feedback 的 `catalog_xmin` 发送给 primary。 primary 如果通过 slot 接收，就会把它作为 physical slot 的 `catalog_xmin`。 结果是：
```text
standby 上的 logical decoding 需求
  -> 通过 hot standby feedback 传播到 primary
  -> primary 的 catalog VACUUM horizon 被保守化
```
这不是 bug。 这是为了保证 standby 上的 logical slot 后续仍能访问所需 catalog 历史。

### 8.7 prepared transaction

prepared transaction 不通过 replication slot 发布。 但它也能钉住 ProcArray horizon。 如果现场中 slot 看起来正常，仍有老 XID 阻止 cleanup，要检查 two-phase 状态。 `vacuum_get_cutoffs()` 的 WARNING hint 也同时提醒：
```text
old prepared transactions
stale replication slots
```
这说明 wraparound 风险现场不能只查 slot。

## 9. 成本、资源与跨模块传播

slot xmin 看起来只是两个 XID。

但它的资源传播很宽。

第一类传播是 heap bloat。

旧 tuple 不能移除，page 内可用空间减少。HOT chain 变长。index cleanup 可能做了但 heap 仍保留 recently dead tuple。

第二类传播是 catalog bloat。

`catalog_xmin` 老时，系统 catalog 的旧版本不能及时清掉。这会影响 relcache / syscache lookup 的局部性。DDL 频繁系统更明显。

第三类传播是 WAL retention。

同一个 logical slot 通常还会有落后的 `restart_lsn`。`restart_lsn` 拖住 WAL segment，`catalog_xmin` 拖住 catalog tuple。这两个问题常同时出现，但不是同一个字段造成的。

第四类传播是 autovacuum wraparound 压力。

`vacuum_get_cutoffs()` 会检查：
```text
OldestXmin 是否早于 safeOldestXmin
```
如果太老，会发出：
```text
cutoff for removing and freezing tuples is far in the past
```
hint 会提示关闭长事务、处理 prepared transaction 或 drop stale replication slot。

第五类传播是 ProcArray 和 slot 扫描成本。

`ComputeXidHorizons()` 要扫描 PGPROC。`ReplicationSlotsComputeRequiredXmin()` 要扫描 slot array。这些通常不是 bloat 的主要成本来源。但在 slot 很多、feedback 或 logical confirmation 很频繁时，重算 horizon 会增加 shared lock traffic。源码因此用 shared `ReplicationSlotControlLock` 扫 slot，并接受保守回退。

## 10. 观测与诊断入口

诊断的第一步是把状态分成三类。 直接可见：
```text
pg_replication_slots.xmin
pg_replication_slots.catalog_xmin
pg_replication_slots.active
pg_replication_slots.active_pid
pg_replication_slots.restart_lsn
pg_replication_slots.confirmed_flush_lsn
pg_stat_all_tables.n_dead_tup
pg_stat_all_tables.last_autovacuum
pg_stat_all_tables.vacuum_count
pg_stat_all_tables.autovacuum_count
pg_stat_activity.backend_xmin
```
只能间接推断：
```text
slot effective_xmin
slot effective_catalog_xmin
candidate_catalog_xmin
ProcArray 的当前 replication_slot_xmin 聚合值
某次 VACUUM 实际使用的 OldestXmin
```
基本不可从 SQL 直接看到：
```text
SnapBuild 当前内部 snapshot xmin
LogicalIncreaseXminForSlot() 尚未应用的候选细节
某次 ComputeXidHorizons() 的中间 shared / catalog / data horizon
```

### 10.1 从 slot 看保留边界

先看所有 slot：
```sql
select slot_name,
       slot_type,
       active,
       active_pid,
       xmin,
       catalog_xmin,
       restart_lsn,
       confirmed_flush_lsn,
       wal_status,
       safe_wal_size,
       inactive_since,
       invalidation_reason
from pg_replication_slots
order by catalog_xmin nulls last, xmin nulls last;
```
解释时要按 slot type 分开。 physical slot 的 `xmin` / `catalog_xmin` 多来自 standby feedback。 logical slot 的 `catalog_xmin` 多来自 logical decoding snapshot builder。 logical slot 的 `xmin` 常常为 NULL。 这不表示它没有任何 VACUUM 影响。 它可能仍然通过 `catalog_xmin` 影响 catalog relation。 也可能在 initial snapshot 期间通过不可见的 `effective_xmin` 影响 data relation。

### 10.2 从表统计看 VACUUM 是否“跑了但删不掉”

再看表级统计：
```sql
select schemaname,
       relname,
       n_dead_tup,
       n_live_tup,
       last_vacuum,
       last_autovacuum,
       vacuum_count,
       autovacuum_count
from pg_stat_all_tables
where n_dead_tup > 0
order by n_dead_tup desc
limit 30;
```
`pg_stat_all_tables` 定义在 `system_views.sql`。 `n_dead_tup` 来自 stats collector 的 relation entry。 它是估算和累计观测，不是 heap page 的实时精确扫描。 但它足以提示：
```text
dead tuple 产生速度超过 cleanup 能力
或 cleanup horizon 被钉住
```
如果 `last_autovacuum` 一直更新，`n_dead_tup` 却不降，要优先怀疑 horizon。 如果 `last_autovacuum` 很久没更新，还要检查 autovacuum 调度、threshold、cost delay、lock conflict。

### 10.3 查普通长事务

长事务入口：
```sql
select pid,
       backend_xid,
       backend_xmin,
       state,
       xact_start,
       query_start,
       wait_event_type,
       wait_event,
       left(query, 120) as query
from pg_stat_activity
where backend_xmin is not null
   or backend_xid is not null
order by backend_xmin nulls last, xact_start nulls last;
```
如果这里有很老的 `backend_xmin`，先解释普通 MVCC horizon。 不要急着归因 slot。 如果 primary 上没有老事务，但 standby 有长查询，同时 `hot_standby_feedback` 开启，再回到 feedback 链路。

### 10.4 判断是 data bloat 还是 catalog bloat

普通表膨胀时，关注：
```text
slot xmin
standby feedback xmin
backend_xmin
prepared transactions
```
catalog 膨胀时，额外关注：
```text
logical slot catalog_xmin
standby 通过 feedback 回传的 catalog_xmin
user_catalog_table
频繁 DDL 或扩展元数据 churn
```
`ComputeXidHorizons()` 的分流意味着：
```text
catalog_xmin 老:
  首先解释 catalog relation 的旧版本保留。

xmin 老:
  解释普通 data relation 和 shared relation 的旧版本保留。
```
这不是绝对二分。 shared relation 和 logical decoding accessible relation 会更保守。 源码判断在：
```text
GlobalVisHorizonKindForRel()
```

### 10.5 用日志或断点确认源码链

源码调试时，建议断点按顺序设：
```text
LogicalIncreaseXminForSlot
LogicalConfirmReceivedLocation
ReplicationSlotsComputeRequiredXmin
ProcArraySetReplicationSlotXmin
ComputeXidHorizons
vacuum_get_cutoffs
HeapTupleSatisfiesVacuum
```
physical feedback 路径改成：
```text
XLogWalRcvSendHSFeedback
ProcessStandbyHSFeedbackMessage
PhysicalReplicationSlotNewXmin
ReplicationSlotsComputeRequiredXmin
```
观察点不是只打印字段。 要画出时间线：
```text
谁发布了更老的 xmin
何时被聚合进 ProcArray
VACUUM 本次拿到哪个 OldestXmin
tuple deleting XID 与 OldestXmin 的大小关系是什么
```

## 11. 常见误区

误区一：`pg_replication_slots.catalog_xmin` 老，所以所有普通表都一定不能 vacuum。

不对。普通 data horizon 不直接应用 `slot_catalog_xmin`。catalog relation 和 shared relation 才会额外受它影响。普通表是否受影响，要看 `xmin`、长事务、standby feedback 以及 relation 类型。

误区二：slot `active = false` 就没有资源影响。

不对。persistent slot inactive 后仍保留 `restart_lsn`、`xmin`、`catalog_xmin`。它只是当前没有 owner backend。

误区三：`pg_replication_slots.xmin` 为 NULL，VACUUM 就不可能被 slot 拖住。

不对。logical slot 可能通过 `catalog_xmin` 拖住 catalog。initial snapshot 期间还可能有 SQL view 不显示的 `effective_xmin`。

误区四：hot_standby_feedback 只是 standby 配置，不会影响 primary。

不对。它的目的就是让 standby 把 cleanup horizon 反馈给 primary，从而减少 standby cleanup conflicts。代价就是 primary bloat 风险。

误区五：VACUUM 跑完后 `n_dead_tup` 不降就是 VACUUM bug。

不对。如果 deleting XID 不早于 `OldestXmin`，`HeapTupleSatisfiesVacuum()` 会把 tuple 归为 `RECENTLY_DEAD`。VACUUM 正确地不能删。

误区六：`catalog_xmin` 可以延迟保存，反正只是诊断字段。

不对。logical slot 必须先保存 `data.catalog_xmin`，再放宽 `effective_catalog_xmin`。这是 crash safety 边界。

误区七：一个 xmin 值单独就能解释所有行为。

不对。raw XID 只有放在这些上下文里才有语义：
```text
字段来源
slot type
effective 还是 persistent
relation kind
是否 hot standby feedback
是否 consumer ack
是否 crash 后恢复
```

## 12. 课堂实验


### 实验 1：logical slot 断开后观察 `catalog_xmin`

目标：确认 inactive logical slot 仍能保留 catalog horizon。 步骤：
```sql
-- session A
select * from pg_create_logical_replication_slot('s1', 'test_decoding');

-- 记录 catalog_xmin
select slot_name, active, xmin, catalog_xmin, confirmed_flush_lsn
from pg_replication_slots
where slot_name = 's1';
```
然后制造 catalog churn：
```sql
create table t_slot_xmin(a int);
alter table t_slot_xmin add column b text;
alter table t_slot_xmin drop column b;
vacuum verbose pg_class;
```
观察：
```sql
select slot_name, active, xmin, catalog_xmin, confirmed_flush_lsn
from pg_replication_slots
where slot_name = 's1';
```
源码回扣：
```text
CreateInitDecodingContext()
  -> data.catalog_xmin / effective_catalog_xmin
  -> ReplicationSlotsComputeRequiredXmin()
  -> ComputeXidHorizons()
  -> catalog_oldest_nonremovable
```
实验结束要 drop slot：
```sql
select pg_drop_replication_slot('s1');
```

### 实验 2：consumer ack 推进 `catalog_xmin`

目标：观察 `confirmed_flush_lsn` 与 `catalog_xmin` 的关系。 步骤：
```sql
select * from pg_create_logical_replication_slot('s2', 'test_decoding');

create table t_decode(a int);
insert into t_decode values (1);

select * from pg_logical_slot_get_changes('s2', null, null);

select slot_name, catalog_xmin, confirmed_flush_lsn
from pg_replication_slots
where slot_name = 's2';
```
多执行几轮 DDL / DML / get_changes。 观察 `confirmed_flush_lsn` 是否推进。 源码回扣：
```text
SnapBuildProcessRunningXacts()
  -> LogicalIncreaseXminForSlot()
  -> candidate_catalog_xmin
  -> pg_logical_slot_get_changes()
  -> LogicalConfirmReceivedLocation()
  -> ReplicationSlotSave()
  -> effective_catalog_xmin
```
实验结束：
```sql
select pg_drop_replication_slot('s2');
```

### 实验 3：hot standby feedback 对 primary VACUUM 的影响

目标：理解 standby 长查询如何让 primary 保留旧版本。 准备 primary / standby 后，在 standby 开启：
```conf
hot_standby_feedback = on
wal_receiver_status_interval = 1s
```
standby session A：
```sql
begin;
select count(*) from some_large_table;
-- 保持事务不结束
```
primary session B：
```sql
update some_large_table set payload = payload where id between 1 and 100000;
vacuum verbose some_large_table;
```
primary 诊断：
```sql
select n_dead_tup, last_autovacuum, vacuum_count, autovacuum_count
from pg_stat_all_tables
where relname = 'some_large_table';
```
如果 primary walsender 使用 physical slot，再看：
```sql
select slot_name, slot_type, active, xmin, catalog_xmin
from pg_replication_slots;
```
源码回扣：
```text
standby GetReplicationHorizons()
  -> XLogWalRcvSendHSFeedback()
  -> primary ProcessStandbyHSFeedbackMessage()
  -> PhysicalReplicationSlotNewXmin() 或 MyProc->xmin
  -> ComputeXidHorizons()
  -> vacuum_get_cutoffs()
```

### 实验 4：断点看 VACUUM 的 OldestXmin

目标：从源码确认某个旧 tuple 为什么只是 `RECENTLY_DEAD`。 断点：
```text
vacuum_get_cutoffs
HeapTupleSatisfiesVacuum
ReplicationSlotsComputeRequiredXmin
ProcArraySetReplicationSlotXmin
```
观察：
```text
cutoffs->OldestXmin
tuple xmax / dead_after
procArray->replication_slot_xmin
procArray->replication_slot_catalog_xmin
```
判断：
```text
dead_after < OldestXmin:
  VACUUM 可移除

dead_after >= OldestXmin:
  VACUUM 必须保留
```

## 13. 讨论题

1. 为什么 logical slot 长期保护的是 `catalog_xmin`，而不是所有 data tuple 的 `xmin`？
2. 为什么 `LogicalConfirmReceivedLocation()` 必须先保存 slot state，再推进 `effective_catalog_xmin`？
3. standby 发送 feedback 时，为什么 `GetReplicationHorizons()` 要返回 `shared_oldest_nonremovable_raw` 而不是最保守的 shared horizon？
4. 没有 physical slot 时，primary 为什么只能把 feedback `xmin` 和 `catalog_xmin` 折叠进 `MyProc->xmin`？
5. 为什么 `pg_replication_slots.catalog_xmin` 不能完全代表当前 VACUUM 使用的 slot catalog horizon？
6. 一个 inactive logical slot、一个 standby 长查询和一个普通长事务，在 `ComputeXidHorizons()` 中分别通过什么状态影响 cleanup？
7. 如果 `pg_stat_all_tables.n_dead_tup` 增长但 `last_autovacuum` 持续更新，应该如何沿源码链排查？
8. 为什么 slot horizon 的并发重算允许保守回退，却不能允许过于激进？

## 14. 源码检查清单：遇到 bloat 现场怎么回到代码

第一步，确认旧版本保留的是 data 还是 catalog。 看业务表膨胀，先查 `xmin`、长事务和 feedback。 看 system catalog 膨胀，必须查 `catalog_xmin`。 第二步，定位 observer。 SQL 侧用：
```sql
select slot_name, slot_type, active, active_pid,
       xmin, catalog_xmin, restart_lsn, confirmed_flush_lsn
from pg_replication_slots;
```
再看：
```sql
select pid, backend_xid, backend_xmin, state, xact_start, query_start
from pg_stat_activity
where backend_xmin is not null or backend_xid is not null;
```
第三步，映射到源码发布点。 logical slot：
```text
CreateInitDecodingContext()
LogicalIncreaseXminForSlot()
LogicalConfirmReceivedLocation()
```
physical feedback：
```text
XLogWalRcvSendHSFeedback()
ProcessStandbyHSFeedbackMessage()
PhysicalReplicationSlotNewXmin()
```
普通 backend：
```text
GetSnapshotData()
SnapshotResetXmin()
ComputeXidHorizons()
```
第四步，确认聚合点。 所有 slot 最终都要到：
```text
ReplicationSlotsComputeRequiredXmin()
ProcArraySetReplicationSlotXmin()
```
如果这里没有对应 old XID，VACUUM 被拖住的原因就不在 slot。 第五步，确认消费点。 目标 relation 的 horizon 在：
```text
GetOldestNonRemovableTransactionId(rel)
GlobalVisHorizonKindForRel(rel)
vacuum_get_cutoffs()
```
tuple 状态转换在：
```text
HeapTupleSatisfiesVacuum()
HeapTupleSatisfiesVacuumHorizon()
```
第六步，解释观测入口的粒度。 `pg_replication_slots` 是 slot 当前状态投影。 `pg_stat_all_tables` 是 relation 统计。 `pg_stat_activity.backend_xmin` 是 backend 当前发布的 snapshot xmin。 它们没有一个单独等于 VACUUM 的完整因果链。 需要把它们按时间串起来。

## 15. 本节小结

本节的核心链路是：
```text
logical decoding / hot standby feedback / long transaction 发布 xmin 需求
  -> slot.c 聚合 effective_xmin / effective_catalog_xmin
  -> procarray.c 计算 relation-sensitive cleanup horizon
  -> vacuum.c 得到 OldestXmin
  -> heapam_visibility.c 决定 old tuple 是 DEAD 还是 RECENTLY_DEAD
```
`xmin` 和 `catalog_xmin` 的区别不是字段命名差异。

它是 data tuple cleanup 与 catalog history correctness 的边界分离。

`data.xmin` / `data.catalog_xmin` 是持久 slot contract。

`effective_xmin` / `effective_catalog_xmin` 是当前真正发布给 VACUUM 的运行时边界。

logical slot 推进 `catalog_xmin` 必须先持久化再放宽 effective 值。

physical standby feedback 可以用不同顺序，因为它的失败后果主要是 standby conflict，而不是 logical decoding 产生错误结果。

VACUUM 不能移除旧版本时，常见原因包括：
```text
普通长事务发布老 backend_xmin；
inactive 或 ack 慢的 logical slot 保留 catalog_xmin；
initial snapshot 临时保留 effective_xmin；
hot_standby_feedback 把 standby 查询需求传播到 primary；
standby logical slot 通过 feedback 把 catalog_xmin 传播到 upstream；
prepared transaction 继续保留老 XID。
```
诊断时要把直接可见、间接可推断和不可见状态分开。

`pg_replication_slots` 能看到 `data.xmin` / `data.catalog_xmin`。

它看不到所有 effective / candidate 状态。

`pg_stat_all_tables` 能提示 dead tuple 与 autovacuum 行为。

它不能告诉你 OldestXmin 的来源。

可迁移的系统规律是：
```text
cleanup boundary 不是由被清理对象自己决定；
它由所有可能观察旧版本的对象发布需求，再由系统按 relation 类型取最保守安全下界。
```
只要这个下界还没越过 tuple 的 deleting XID，VACUUM 不移除旧版本就是正确行为。 真正的工程判断是找到发布下界的 observer。 然后决定是结束长事务、恢复 consumer、调整 feedback 策略、drop stale slot，还是接受 bloat 换取 standby / logical decoding correctness。
