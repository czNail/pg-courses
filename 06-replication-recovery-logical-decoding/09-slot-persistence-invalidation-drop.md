# PostgreSQL Replication Slot 持久化、恢复、失效与删除
## 课程定位
前置知识：已经理解 WAL segment、checkpoint、restartpoint、physical / logical slot 的基本字段，
也知道 logical decoding 为什么依赖 `restart_lsn`、`confirmed_flush_lsn` 和 `catalog_xmin`。
本节唯一主问题：
```text
replication slot 如何落盘、崩溃后如何恢复，
WAL 丢失、数据库删除、catalog xmin 过旧和手工 drop
分别会让 slot 进入什么边界状态？
```
核心矛盾：slot 的承诺是“消费者断线或 primary 崩溃后仍能继续追赶”，
因此 `restart_lsn`、`xmin`、`catalog_xmin`、`confirmed_flush` 和 invalidation 状态必须可靠保存；
但 primary 又不能因为一个永远不追赶的 slot 无限保留 WAL、catalog tuple 或已经删除的 database。
PostgreSQL 的选择不是把 slot 做成 catalog row，
而是把它拆成 `pg_replslot/<name>/state` 磁盘状态、shared memory 运行态、
checkpoint 批量刷盘、startup 恢复、invalidation 标记和 crash-safe drop rename。
学完后应能判断：
```text
1. pg_replslot/<slot>/state 里保存的是哪个语义层。
2. SaveSlotToPath() 为什么写 state.tmp、fsync、rename、再 fsync 目录。
3. RestoreSlotFromDisk() 为什么只恢复 RS_PERSISTENT，并在 crash recovery 前执行。
4. CheckPointReplicationSlots() 为什么要和 slot WAL reservation 互锁。
5. WAL 丢失、rows_removed、wal_level_insufficient、idle_timeout 如何变成 invalidated slot。
6. DROP DATABASE 为什么删除 database-specific logical slot，而不是把它标成 invalidated。
7. 手工 pg_drop_replication_slot() 为什么先 rename 目录到 .tmp，再清 shared memory。
8. invalidated slot 在 pg_replication_slots 中怎么观察，以及如何安全删除。
```
本课基于本地 `/home/nail/postgres` 源码，
分支 `master`，
提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
上一节已经建立 replication slot 的基本生命周期：
```text
ReplicationSlotCreate()
  -> acquire 当前 backend 的 active ownership
  -> physical / logical 按各自协议推进 restart_lsn、xmin、confirmed_flush
  -> release 时 temporary / ephemeral 清理，persistent 保留
  -> pg_replication_slots 暴露当前共享状态
```
本节继续沿同一个对象往后走，
但问题换成 crash boundary 和 irreversible cleanup boundary。
slot 最危险的地方不在创建成功那一刻，
而在这些时间点：
```text
1. slot 推进了 restart_lsn，但 state 文件还没保存。
2. logical slot 放宽了 catalog_xmin，VACUUM 可能删除旧 catalog tuple。
3. checkpoint 准备删除旧 WAL segment。
4. standby recovery 正在 replay 会移除旧行版本的 WAL record。
5. 用户 DROP DATABASE，目标 database 上仍有 logical slot。
6. 用户或 replication protocol drop 一个 slot。
7. postmaster crash 后，只剩 pg_replslot 目录可用。
```
这些时间点共同指向一个问题：
```text
slot 是一个资源保留承诺。
承诺可以继续、可以保守、可以失效、可以删除；
但不能在 crash 后变成“不知道之前承诺过什么”。
```
所以本节不再泛讲所有 slot 字段。
我们只看会影响持久化和边界状态的字段：
```text
data.persistency
data.restart_lsn
data.xmin
data.catalog_xmin
data.invalidated
data.confirmed_flush
dirty / just_dirtied
effective_xmin / effective_catalog_xmin
last_saved_restart_lsn
last_saved_confirmed_flush
active_proc
inactive_since
```
线性主链路是：
```text
创建或推进 slot 状态
  -> 标脏或立即保存
  -> checkpoint 批量落盘
  -> startup 从 pg_replslot 恢复
  -> 恢复后重新发布 WAL / xmin 保留边界
  -> 当保留承诺无法满足时 invalidation
  -> 当用户明确删除时 crash-safe drop
```
下一节可以继续讲 logical decoding 如何在 `restart_lsn` 和 `confirmed_flush_lsn` 之间做 reorder buffer、
snapbuild 和消费确认推进。
本节只回答 slot 自己能不能继续存在。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
persistent slot 的 durable truth 是 pg_replslot/<name>/state；
backend 更新 shared memory 后用 dirty 标记延迟刷盘，
需要 correctness 的路径显式 ReplicationSlotSave()；
checkpoint 通过 CheckPointReplicationSlots() 保存 dirty slot；
startup 在 crash recovery 删除旧 WAL 或 tuple 前先 StartupReplicationSlots()；
如果 WAL、row horizon、wal_level 或 idle timeout 使承诺不再可满足，
InvalidateObsoleteReplicationSlots() 把 slot 标成 invalidated 并持久化；
如果用户 drop，则 ReplicationSlotDropPtr() 先把目录 rename 到 .tmp，
再清 in_use 和资源保留边界。
```
这条模型里有两个不能混淆的状态轴。
第一条是存在性：
```text
磁盘目录存在且 state 可恢复:
  startup 可以把 persistent slot 重新放入 shared memory。
shared memory in_use = true:
  当前 postmaster 生命周期内 slot 存在于数组中。
active_proc != INVALID_PROC_NUMBER:
  当前有一个 backend 独占持有 slot。
```
第二条是语义有效性：
```text
data.invalidated == RS_INVAL_NONE:
  slot 仍然代表有效的资源保留承诺。
data.invalidated != RS_INVAL_NONE:
  slot 还存在，但承诺已经失败；
  它不再参与 WAL / xmin 保留，acquire 时会被拒绝。
```
这就是本节的核心 tension：
```text
为了 crash safety，slot 状态必须保守地持久化；
为了资源回收，slot 又必须能明确进入 invalidated 或 dropped 状态。
```
`invalidated` 不是 drop。
drop 也不是 invalidation。
database drop 又不是 WAL removed。
这三者会留下不同的观测结果和不同的恢复行为。
## 3. 核心文件分工与阅读顺序
推荐按这条顺序读源码：
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/slot.h` | `ReplicationSlotPersistentData`、`ReplicationSlot`、`ReplicationSlotInvalidationCause`。 |
| 2 | `src/backend/replication/slot.c` | `CreateSlotOnDisk()`、`SaveSlotToPath()`、`RestoreSlotFromDisk()`、`CheckPointReplicationSlots()`、`InvalidateObsoleteReplicationSlots()`、`ReplicationSlotDropPtr()`。 |
| 3 | `src/backend/replication/slotfuncs.c` | SQL 创建、删除、推进 slot，以及 `pg_get_replication_slots()` 的字段来源。 |
| 4 | `src/backend/replication/walsender.c` | replication protocol 创建 / 删除 slot，standby reply 如何推进 physical slot。 |
| 5 | `src/backend/replication/logical/logical.c` | `LogicalConfirmReceivedLocation()` 如何先保存 catalog xmin / restart_lsn，再放宽 effective horizon。 |
| 6 | `src/backend/storage/ipc/standby.c` | recovery conflict 如何通过 `RS_INVAL_HORIZON` 让 logical slot 失效。 |
| 7 | `src/backend/access/transam/xlog.c` | startup、checkpoint、restartpoint、WAL 删除前如何调用 slot 层。 |
| 8 | `src/backend/catalog/system_views.sql` | `pg_replication_slots` view 如何包装 `pg_get_replication_slots()`。 |
| 9 | `src/backend/commands/dbcommands.c` | `DROP DATABASE` 如何检查并删除 database-specific logical slot。 |
本节的读法不是“从 `slot.c` 顶部读到底”。
更有效的入口是时间线：
```text
CreateSlotOnDisk()
  -> SaveSlotToPath()
  -> CheckPointReplicationSlots()
  -> StartupReplicationSlots()
  -> RestoreSlotFromDisk()
  -> InvalidateObsoleteReplicationSlots()
  -> ReplicationSlotDropPtr()
```
然后再回到观测：
```text
pg_get_replication_slots()
  -> wal_status / safe_wal_size / conflicting / invalidation_reason
  -> pg_drop_replication_slot()
```
## 4. 磁盘状态：`pg_replslot/<name>/state`
`slot.c` 文件头部已经说明，
每个 slot 在 `$PGDATA/pg_replslot` 下有自己的目录：
```text
pg_replslot/
  <slot_name>/
    state
```
`state` 文件不是普通文本配置。
它对应 `slot.c` 内部的 `ReplicationSlotOnDisk`：
```text
ReplicationSlotOnDisk:
  magic
  checksum
  version
  length
  ReplicationSlotPersistentData slotdata
```
`magic` 和 `version` 用来拒绝错误格式。
`length` 用来确认当前版本的 payload 大小。
`checksum` 覆盖 version、length 和 slotdata，
避免 startup 在损坏的 state 文件上继续恢复一个错误承诺。
真正的语义保存在 `ReplicationSlotPersistentData`：
| 字段 | 崩溃恢复语义 |
| --- | --- |
| `name` | 目录名和 slot identity。 |
| `database` | `InvalidOid` 表示 physical；非 `InvalidOid` 表示 logical 并绑定 database。 |
| `persistency` | startup 是否恢复；只有 `RS_PERSISTENT` 恢复。 |
| `xmin` | 数据 tuple horizon 的 durable 值。 |
| `catalog_xmin` | catalog tuple horizon 的 durable 值。 |
| `restart_lsn` | crash 后仍可能需要读取的 WAL 起点。 |
| `invalidated` | 失效状态也必须跨 restart 保留。 |
| `confirmed_flush` | logical consumer 已确认位置，shutdown checkpoint 会尽量避免它倒退太多。 |
| `two_phase_at` / `two_phase` | two-phase decoding 能力和生效 LSN。 |
| `plugin` | logical output plugin。 |
| `synced` / `failover` | failover slot / slot sync 相关 durable 标记。 |
注意这里没有 `active_proc`。
也没有 `dirty`。
也没有 `candidate_restart_lsn`。
它们都是当前 postmaster 生命周期里的运行态。
startup 能重建的只有 durable truth：
```text
slotdata:
  slot 存在、绑定哪个 database、是否 persistent、
  最保守需要保留的 WAL / xmin / catalog_xmin、
  是否已经 invalidated。
```
所以一个关键结论是：
```text
crash 后系统不能相信 crash 前 shared memory 里的“更靠前进度”；
只能相信 state 文件里已经校验、fsync、rename 完成的状态。
```
这解释了后面 `last_saved_restart_lsn` 为什么参与 WAL 保留计算。
## 5. shared memory 状态：持久承诺与运行态分离
`ReplicationSlot` 位于 `ReplicationSlotCtl->replication_slots[]`。
它把 durable `data` 和运行态放在一起，
但语义必须分层理解。
与本节最相关的是这些字段组合：
| 字段 | 语义边界 |
| --- | --- |
| `in_use` | 当前 shared memory 数组槽位是否定义了 slot。 |
| `active_proc` | 当前哪个 `PGPROC` 持有 slot。 |
| `dirty` / `just_dirtied` | `data` 是否需要写回 `state`。 |
| `effective_xmin` | 当前真正发布给 ProcArray 的数据 tuple horizon。 |
| `effective_catalog_xmin` | 当前真正发布给 ProcArray 的 catalog tuple horizon。 |
| `data` | 会保存到磁盘并在 startup 恢复的持久状态。 |
| `io_in_progress_lock` | 单 slot 文件写入互斥。 |
| `last_saved_restart_lsn` | 最近一次成功写入 state 的 `restart_lsn`。 |
| `last_saved_confirmed_flush` | 最近一次成功写入 state 的 `confirmed_flush`。 |
| `inactive_since` | slot 变 inactive 的时间，idle invalidation 使用。 |
`data.restart_lsn` 和 `last_saved_restart_lsn` 的关系是本节最重要的不变量之一。
当 persistent slot 的 `data.restart_lsn` 前进但还没保存时：
```text
data.restart_lsn:
  当前内存中认为可以从这里开始读取。
last_saved_restart_lsn:
  crash 后 startup 真正能恢复到的最老已保存位置。
```
`ReplicationSlotsComputeRequiredLSN()` 对 persistent slot 会更保守：
```text
if persistent and last_saved_restart_lsn valid
   and restart_lsn > last_saved_restart_lsn:
  use last_saved_restart_lsn as retention boundary
else:
  use restart_lsn
```
原因很直接：
```text
如果 checkpoint 删除了 last_saved_restart_lsn 到 restart_lsn 之间的 WAL，
但系统随后 crash，
startup 只能从 old state 文件恢复 last_saved_restart_lsn。
这时 slot 会需要一段已经删除的 WAL。
```
所以持久 slot 的 WAL 保留边界不是“最新内存进度”，
而是“crash 后能证明不会后退到更早位置的进度”。
logical slot 还多一个 `effective_catalog_xmin` 边界。
`data.catalog_xmin` 是将要保存的 durable 值，
`effective_catalog_xmin` 是系统当前允许 VACUUM 参考的实际 horizon。
`LogicalConfirmReceivedLocation()` 中的顺序是：
```text
先把新的 data.catalog_xmin / data.restart_lsn 写入 shared memory
  -> ReplicationSlotMarkDirty()
  -> ReplicationSlotSave()
  -> 保存成功后再更新 effective_catalog_xmin
  -> 重新计算全局 xmin / LSN
```
这个顺序避免一种错误：
```text
VACUUM 已经按新的 catalog_xmin 删除旧 catalog tuple，
但 crash 后 state 文件还停在更老的 catalog_xmin；
logical decoding 重启后以为旧 catalog 仍可用，实际已经不可解释。
```
## 6. 创建时如何落盘：先 `.tmp`，再原子可见
创建 slot 的共享入口是 `ReplicationSlotCreate()`。
它在持有 `ReplicationSlotAllocationLock` 后，
先找空闲数组槽位并初始化 `slot->data` 和运行态字段。
但是它不会先设置 `in_use`。
它先调用 `CreateSlotOnDisk(slot)`。
创建磁盘目录的顺序是：
```text
CreateSlotOnDisk(slot)
  -> path    = pg_replslot/<name>
  -> tmppath = pg_replslot/<name>.tmp
  -> 清理可能残留的 <name>.tmp
  -> MakePGDirectory(tmppath)
  -> fsync_fname(tmppath, true)
  -> slot->dirty = true
  -> SaveSlotToPath(slot, tmppath, ERROR)
  -> rename(tmppath, path)
  -> START_CRIT_SECTION()
       fsync_fname(path, true)
       fsync_fname(PG_REPLSLOT_DIR, true)
     END_CRIT_SECTION()
```
这个顺序把“slot 是否对 startup 可见”变成目录 rename 边界。
如果 crash 发生在 rename 前：
```text
pg_replslot/<name>.tmp 可能存在；
StartupReplicationSlots() 会把 .tmp 目录删除；
slot 不会被恢复。
```
如果 crash 发生在 rename 后且 fsync 成功：
```text
pg_replslot/<name>/state 可见；
如果 state 内容是 RS_PERSISTENT，
startup 会恢复它。
```
为什么创建时还没有 `in_use`？
因为在 state 文件成功写入和目录可见之前，
其它 backend 不应该扫描到一个“半创建”的 slot。
`ReplicationSlotCreate()` 的注释也强调：
此时还没 mark allocated，
所以 `CreateSlotOnDisk()` ERROR 不需要特殊 cleanup。
目录 rename 和 `in_use` 的顺序是：
```text
磁盘 durable slot 已经准备好
  -> ReplicationSlotControlLock exclusive
  -> slot->in_use = true
  -> slot->active_proc = MyProcNumber
  -> MyReplicationSlot = slot
```
也就是说：
```text
对当前 postmaster 内其它 backend 可见之前，
slot 已经有一个可由 startup 解释的磁盘形态。
```
## 7. `SaveSlotToPath()`：state 文件的 crash-safe 覆盖
`SaveSlotToPath()` 是 slot 状态落盘的核心函数。
`ReplicationSlotSave()` 只是对当前 `MyReplicationSlot` 拼路径后调用它。
`CreateSlotOnDisk()` 和 checkpoint 也复用它。
它的主流程是：
```text
SaveSlotToPath(slot, dir, elevel)
  -> 持 slot->mutex 读取 dirty，并清 just_dirtied
  -> 如果不 dirty，直接返回
  -> LWLockAcquire(slot->io_in_progress_lock)
  -> 构造 ReplicationSlotOnDisk cp
  -> 持 slot->mutex memcpy slot->data 到 cp.slotdata
  -> 计算 CRC
  -> 写 dir/state.tmp
  -> fsync state.tmp
  -> close state.tmp
  -> rename state.tmp 到 dir/state
  -> critical section:
       fsync state
       fsync dir
       fsync pg_replslot
  -> 持 slot->mutex:
       如果没有再次 just_dirtied，则 dirty=false
       last_saved_confirmed_flush = cp.slotdata.confirmed_flush
       last_saved_restart_lsn = cp.slotdata.restart_lsn
  -> release io lock
```
这里有三个正确性点。
第一，写的是 `state.tmp`，
不是原地覆盖 `state`。
如果 crash 发生在写临时文件或 fsync 临时文件期间，
旧 `state` 仍然可用。
`RestoreSlotFromDisk()` 启动时会先删除残留的 `state.tmp`。
第二，rename 后会 fsync 文件、slot 目录和 `pg_replslot` 父目录。
原因是 state 文件内容和目录项都属于 crash safety 的一部分。
只有文件 fsync 不够，
目录 rename 也必须落盘。
第三，`dirty` 不是无条件清掉。
`SaveSlotToPath()` 开始时清 `just_dirtied`，
写完后只有 `just_dirtied` 没被并发路径重新设置，才清 `dirty`。
这保证写盘期间发生的新更新不会被这次旧 snapshot 的写入吞掉。
`elevel` 也有语义。
前台必须保证某个状态落盘时会用 `ERROR`。
checkpoint 调用时使用 `LOG`，
因为 checkpoint 不应该因为某个 slot state 写失败就按普通 ERROR 展开；
但 critical section 内 fsync 失败会按更严重的路径处理。
## 8. 哪些路径只是标脏，哪些路径必须立即保存
不是每次 slot 变化都立即 fsync。
PostgreSQL 有意把普通进度更新和 correctness-critical 更新区分开。
physical standby 反馈 flush position 时，
`walsender.c` 的 `PhysicalConfirmReceivedLocation()` 会：
```text
slot->data.restart_lsn = flushPtr
ReplicationSlotMarkDirty()
ReplicationSlotsComputeRequiredLSN()
```
但它不会立刻 `ReplicationSlotSave()`。
源码注释说明：
此处丢失信息最坏只是更保守地保留 WAL，
或者统计视图显示旧位置。
对 correctness 来说，
crash 后恢复到旧的 `restart_lsn` 仍然安全，
只是不够激进。
logical slot 推进 catalog xmin 或 restart LSN 时则不同。
`LogicalConfirmReceivedLocation()` 在 candidate 条件满足后，
如果更新了 `data.catalog_xmin` 或 `data.restart_lsn`，
会立刻：
```text
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```
然后才让 `effective_catalog_xmin` 前进，
并重新计算全局 xmin 或 LSN。
这个差异可以压缩成一个判断规则：
```text
如果 crash 后回退只会更保守，允许 checkpoint 批量保存。
如果 crash 后回退会让系统误判已经删除的历史仍可用，必须先保存再放宽清理边界。
```
`ReplicationSlotPersist()` 也必须立即保存。
它把 `RS_EPHEMERAL` 或 `RS_TEMPORARY` 转成 `RS_PERSISTENT`：
```text
slot->data.persistency = RS_PERSISTENT
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```
否则 persistent logical slot 创建成功返回给客户端后，
crash 后可能仍按 ephemeral 清理，
破坏用户看到的 DDL 承诺。
## 9. checkpoint 如何保存 slot 状态
checkpoint 入口在 `xlog.c` 的 `CheckPointGuts()`：
```text
CheckPointGuts()
  -> CheckPointRelationMap()
  -> CheckPointReplicationSlots(flags & CHECKPOINT_IS_SHUTDOWN)
  -> CheckPointSnapBuild()
  -> CheckPointLogicalRewriteHeap()
  -> CheckPointReplicationOrigin()
  -> ...
```
`CheckPointReplicationSlots()` 做两件事。
第一，它批量保存 dirty slot：
```text
LWLockAcquire(ReplicationSlotAllocationLock, LW_SHARED)
for each in-use slot:
  path = pg_replslot/<name>
  SaveSlotToPath(s, path, LOG)
LWLockRelease(ReplicationSlotAllocationLock)
```
它拿的是 `ReplicationSlotAllocationLock` shared，
而不是 `ReplicationSlotControlLock`。
原因是 checkpoint 不想阻塞普通扫描和 acquire，
但必须阻止 slot 创建 / 删除和 WAL reservation 在关键边界上穿插。
第二，shutdown checkpoint 对 logical slot 的 `confirmed_flush` 更积极：
```text
if is_shutdown and SlotIsLogical(s)
   and slot valid
   and data.confirmed_flush > last_saved_confirmed_flush:
  mark dirty
```
这不是 correctness 必需。
如果不保存，重启后 logical consumer 的 server-side `confirmed_flush_lsn` 可能倒退，
客户端可能需要重新确认或重读更多 WAL。
shutdown checkpoint 尽量减少这种不必要倒退。
`CheckPointReplicationSlots()` 还会观察：
```text
if last_saved_restart_lsn != data.restart_lsn:
  last_saved_restart_lsn_updated = true
```
保存完成后如果这个标记为 true，
它会重新 `ReplicationSlotsComputeRequiredLSN()`。
因为成功保存后，
persistent slot 的 WAL 保留边界可能可以从更老的 `last_saved_restart_lsn`
推进到新的 `restart_lsn`。
## 10. startup 如何恢复 slot
startup 入口在 `xlog.c`。
在进入 crash recovery 资源删除之前，
代码调用：
```text
StartupReplicationSlots()
```
源码注释写得很明确：
```text
Initialize replication slots, before there's a chance to remove required resources.
```
这说明 slot restore 不是普通后台加载。
它必须发生在旧 WAL 或旧 tuple 有机会被清理之前。
`StartupReplicationSlots()` 扫描 `pg_replslot`：
```text
AllocateDir(PG_REPLSLOT_DIR)
for each entry:
  skip . and ..
  skip non-directory
  if name endswith ".tmp":
     rmtree(path)
     fsync pg_replslot
     continue
  RestoreSlotFromDisk(name)
FreeDir()
ReplicationSlotsComputeRequiredXmin(false)
ReplicationSlotsComputeRequiredLSN()
```
`.tmp` 目录是 create / drop 的 crash residue。
startup 的策略不是尝试补完旧操作，
而是把 `.tmp` 当作“不可识别为有效 slot 的临时目录”清掉。
`RestoreSlotFromDisk()` 恢复单个 slot 的顺序是：
```text
unlink pg_replslot/<name>/state.tmp
open pg_replslot/<name>/state
fsync state
fsync slot directory
read fixed header
check magic
check version
check length
read full payload
check CRC
if persistency != RS_PERSISTENT:
   rmtree(slotdir)
   fsync pg_replslot
   return
check wal_level / standby requirements
copy slotdata into first free ReplicationSlot
effective_xmin = slotdata.xmin
effective_catalog_xmin = slotdata.catalog_xmin
last_saved_confirmed_flush = slotdata.confirmed_flush
last_saved_restart_lsn = slotdata.restart_lsn
active_proc = INVALID_PROC_NUMBER
inactive_since = now
```
这里有几个边界非常重要。
第一，`RS_EPHEMERAL` 和 `RS_TEMPORARY` 不恢复。
如果 persistent logical slot 创建过程中 crash，
磁盘上可能留下 `RS_EPHEMERAL` state；
startup 会删除它。
这正是 `RS_EPHEMERAL` 作为“未完成 persistent slot”的意义。
第二，恢复时只遍历到 `max_replication_slots`，
不包括 `max_repack_replication_slots`。
如果重启时配置比 shutdown 前更小，
可能报：
```text
too many replication slots active before shutdown
```
第三，恢复后的 slot 都是 inactive。
`active_proc` 不会跨进程恢复。
这符合进程模型：
crash 后旧 backend 已不存在，
只能恢复 slot 的资源保留承诺，
不能恢复旧 owner。
第四，恢复完会重新计算：
```text
ProcArraySetReplicationSlotXmin(...)
XLogSetReplicationSlotMinimumLSN(...)
```
也就是把 `pg_replslot` 中的 durable truth 重新发布给事务清理和 WAL 删除模块。
## 11. WAL 丢失：slot 进入 `wal_removed` invalidated 状态
WAL 丢失边界发生在 checkpoint 或 restartpoint 准备删除旧 WAL segment 时。
`xlog.c` 的 checkpoint 路径在 `RemoveOldXlogFiles()` 前调用：
```text
KeepLogSeg(...)
InvalidateObsoleteReplicationSlots(
  RS_INVAL_WAL_REMOVED | RS_INVAL_IDLE_TIMEOUT,
  oldestSegno,
  InvalidOid,
  InvalidTransactionId)
```
restartpoint 路径也做同类检查。
如果某个 slot 的 `restart_lsn` 早于即将保留的最老 LSN，
`DetermineSlotInvalidationCause()` 返回：
```text
RS_INVAL_WAL_REMOVED
```
`slot.c` 中这个 enum 映射到 SQL 可见字符串：
```text
RS_INVAL_WAL_REMOVED -> "wal_removed"
```
如果 slot 当前 inactive，
`InvalidatePossiblyObsoleteSlot()` 会直接 acquire 它：
```text
s->active_proc = MyProcNumber
s->data.invalidated = RS_INVAL_WAL_REMOVED
s->data.restart_lsn = InvalidXLogRecPtr
s->last_saved_restart_lsn = InvalidXLogRecPtr
ReplicationSlotMarkDirty()
ReplicationSlotSave()
ReplicationSlotRelease()
```
如果 slot 当前 active，
checkpoint 不会在持锁下强行改别人的 slot。
它会记录 owner pid，
释放 `ReplicationSlotControlLock`，
然后：
```text
普通进程:
  kill(active_pid, SIGTERM)
startup process:
  SignalRecoveryConflict(..., RECOVERY_CONFLICT_LOGICALSLOT)
```
随后等待 `slot->active_cv`，
重新检查 slot 状态。
注意这里有一个 race 是刻意允许的。
active owner 在被信号处理前可能把 `restart_lsn` 推进到安全位置。
重新检查时如果不再冲突，
invalidation 会取消。
这是安全的：
WAL 还没删除，
如果消费者已经追上，
就没有必要失效。
一旦 `wal_removed` 持久化，
这个 slot 的状态是：
```text
in_use = true
data.invalidated = RS_INVAL_WAL_REMOVED
data.restart_lsn = InvalidXLogRecPtr
不再参与 ReplicationSlotsComputeRequiredLSN()
ReplicationSlotAcquire(..., error_if_invalid=true) 会 ERROR
pg_replication_slots.invalidation_reason = 'wal_removed'
pg_replication_slots.wal_status = 'lost'
```
这是“存在但不可继续使用”的状态。
安全处理方式通常是：
```text
1. 确认下游不能从该 slot 继续。
2. 重新初始化消费者或重建订阅 / base backup / logical slot。
3. 用 pg_drop_replication_slot() 删除 invalidated slot。
```
不要把 `wal_removed` 理解成手工 drop。
它仍在 `pg_replication_slots` 里，
因为系统要把失败原因暴露出来，
并让操作者明确清理。
## 12. catalog xmin 过旧：logical slot 进入 `rows_removed` invalidated 状态
catalog xmin 过旧的典型触发点在 standby recovery。
`standby.c` 的 `ResolveRecoveryConflictWithSnapshot()` 处理 replay conflict 后，
如果逻辑解码可用且 WAL record 涉及 catalog relation，
会调用：
```text
InvalidateObsoleteReplicationSlots(
  RS_INVAL_HORIZON,
  0,
  locator.dbOid,
  snapshotConflictHorizon)
```
源码 enum 叫：
```text
RS_INVAL_HORIZON
```
SQL 可见原因叫：
```text
"rows_removed"
```
当前 master 中没有单独名为 `RS_INVAL_ROWS_REMOVED` 的 enum。
课程和诊断时要把内部名和显示名对应起来：
```text
RS_INVAL_HORIZON -> rows_removed
```
判断条件在 `DetermineSlotInvalidationCause()`：
```text
if SlotIsLogical(s)
   and (dboid == InvalidOid or dboid == s->data.database)
   and (
        effective_xmin <= snapshotConflictHorizon
        or effective_catalog_xmin <= snapshotConflictHorizon
       ):
  return RS_INVAL_HORIZON
```
这里用的是 `effective_xmin` / `effective_catalog_xmin`，
不是裸 `data.xmin` / `data.catalog_xmin`。
原因是 invalidation 判断关心“当前真正阻止清理的 horizon”，
也就是系统已经对外发布的资源保护边界。
`rows_removed` 的语义是：
```text
logical slot 需要的行版本或 catalog 版本已经被 recovery replay 移除；
继续 decoding 可能无法正确解释历史 tuple / schema；
slot 必须失效，而不是假装可以从 confirmed_flush 继续。
```
失效后的观测通常是：
```text
pg_replication_slots.invalidation_reason = 'rows_removed'
pg_replication_slots.conflicting = true
slot 仍存在
slot 不再保留 xmin / catalog_xmin
acquire 会报 can no longer access replication slot
```
`conflicting` 只对 logical slot 有意义。
`slotfuncs.c` 里把下面两类 reason 视为 recovery conflict：
```text
RS_INVAL_HORIZON
RS_INVAL_WAL_LEVEL
```
所以 `rows_removed` 不是“WAL segment 被删了”。
它是“slot 的 MVCC / catalog horizon 承诺已经被 replay 破坏”。
## 13. wal_level 不足：logical slot 进入 `wal_level_insufficient`
`RS_INVAL_WAL_LEVEL` 对应显示字符串：
```text
wal_level_insufficient
```
它主要服务 standby logical decoding 边界。
`xlog.c` 在 redo logical decoding status 变化时，
如果 primary 端关闭 logical decoding，
standby 处于 hot standby，
会调用：
```text
InvalidateObsoleteReplicationSlots(
  RS_INVAL_WAL_LEVEL,
  0,
  InvalidOid,
  InvalidTransactionId)
```
`DetermineSlotInvalidationCause()` 对这个原因的判断很直接：
```text
if possible_causes includes RS_INVAL_WAL_LEVEL
   and SlotIsLogical(s):
  return RS_INVAL_WAL_LEVEL
```
这类 invalidation 的核心不是某段 WAL 已经删除，
而是当前上游 WAL 内容已经不能保证 logical decoding 所需信息。
观测结果：
```text
invalidation_reason = 'wal_level_insufficient'
conflicting = true
slot 仍存在，但不能再被有效 acquire
```
startup restore 还有另一个 wal_level 边界。
`RestoreSlotFromDisk()` 在读出 `state` 后会检查 slot requirements。
如果已有 persistent slot，
但本地配置 `wal_level < replica`，
physical 或 logical slot 都会让 startup FATAL。
这不是 invalidation。
区别是：
```text
startup requirement 不满足:
  还没有进入正常运行，不能可靠恢复 slot 承诺，直接 FATAL。
运行中的 standby 发现 effective wal_level 不足:
  logical slot 已经存在于运行系统中，标成 invalidated 并持久化。
```
## 14. idle timeout：inactive slot 进入 `idle_timeout`
当前源码还有一个 invalidation 原因：
```text
RS_INVAL_IDLE_TIMEOUT -> "idle_timeout"
```
它由 GUC `idle_replication_slot_timeout` 控制，
`0` 表示禁用。
checkpoint / restartpoint 调用 `InvalidateObsoleteReplicationSlots()` 时，
会同时传入：
```text
RS_INVAL_WAL_REMOVED | RS_INVAL_IDLE_TIMEOUT
```
`CanInvalidateIdleSlot()` 要求：
```text
idle_replication_slot_timeout_secs != 0
data.restart_lsn 有效
inactive_since > 0
不是 recovery 中正在从 primary 同步的 synced slot
```
然后 `DetermineSlotInvalidationCause()` 比较：
```text
TimestampDifferenceExceedsSeconds(
  s->inactive_since,
  now,
  idle_replication_slot_timeout_secs)
```
所以 idle timeout 只针对 inactive slot。
active slot 的 `inactive_since` 会在 acquire 时重置为 0。
失效后：
```text
invalidation_reason = 'idle_timeout'
slot 仍存在
slot 不再参与 WAL / xmin 保留
操作者需要 drop 或重新创建
```
它和 `wal_removed` 的差异是：
```text
wal_removed:
  因 required WAL 已经低于可保留边界，restart_lsn 会被置为 Invalid。
idle_timeout:
  因 slot 闲置超过配置，原因是运维策略，不是 WAL 已经必然删除。
```
但对消费者来说两者都不能继续当作有效 slot 使用。
## 15. 数据库删除：logical slot 被 drop，而不是 invalidated
`DROP DATABASE` 是另一条边界。
它不是 `InvalidateObsoleteReplicationSlots()` 的一个 cause。
原因是 database-specific logical slot 的 `data.database` 指向目标 database。
database 被删除后，
这个 slot 没有可绑定的 database 语义。
PostgreSQL 选择直接删除这些 slot。
`dbcommands.c` 在真正删除 database 前先检查：
```text
ReplicationSlotsCountDBSlots(db_id, &nslots, &nslots_active)
if nslots_active:
  ERROR database is used by an active logical replication slot
```
这里只阻止 active logical slot。
inactive logical slot 不阻止 `DROP DATABASE`，
因为后续会被清理。
真正执行删除时，
路径会调用：
```text
ReplicationSlotsDropDBSlots(db_id)
```
WAL redo `XLOG_DBASE_DROP` 的路径也会调用它。
在 hot standby 中，
redo 前还会处理 database conflict，
并用 database lock 避免新连接重新进入目标 database。
`ReplicationSlotsDropDBSlots()` 会扫描所有 in-use slot：
```text
只处理 SlotIsLogical(s)
只处理 s->data.database == dboid
包括已经 invalidated 的 slot
如果 inactive，就临时把 active_proc 设置成 MyProcNumber
释放 ControlLock
ReplicationSlotDropAcquired(false)
restart scan
```
这意味着 database 删除后的边界状态是：
```text
目标 database 的 logical slot 消失。
pg_replication_slots 不再显示这些 slot。
不会留下 invalidation_reason = 'database_dropped'。
physical slot 不受 database oid 约束，不在这里删除。
```
如果你在诊断中看到 database 已经删除，
但想找“该 database 的 slot 为什么 invalidated”，
方向就是错的。
正常路径下它应该已经被 drop。
## 16. 手工 drop：先 rename 到 `.tmp`，再清共享状态
SQL drop 入口是 `slotfuncs.c`：
```text
pg_drop_replication_slot(name)
  -> CheckSlotPermissions()
  -> CheckSlotRequirements(false)
  -> ReplicationSlotDrop(name, true)
```
replication protocol 入口在 `walsender.c`：
```text
DropReplicationSlot(cmd)
  -> ReplicationSlotDrop(cmd->slotname, !cmd->wait)
```
`ReplicationSlotDrop()` 先 acquire：
```text
ReplicationSlotAcquire(name, nowait, false)
```
第三个参数是 `false`，
所以 invalidated slot 也可以被 acquire 用于 drop。
这是安全清理 invalidated slot 的关键。
如果 slot active：
```text
nowait = true:
  ERROR replication slot "... " is active for PID ...
nowait = false:
  等 active_cv，直到 owner release，再重试 acquire。
```
真正删除在 `ReplicationSlotDropPtr()`。
顺序是：
```text
LWLockAcquire(ReplicationSlotAllocationLock, LW_EXCLUSIVE)
path    = pg_replslot/<name>
tmppath = pg_replslot/<name>.tmp
rename(path, tmppath)
START_CRIT_SECTION()
  fsync_fname(tmppath, true)
  fsync_fname(PG_REPLSLOT_DIR, true)
END_CRIT_SECTION()
LWLockAcquire(ReplicationSlotControlLock, LW_EXCLUSIVE)
  slot->active_proc = INVALID_PROC_NUMBER
  slot->in_use = false
LWLockRelease(ReplicationSlotControlLock)
ConditionVariableBroadcast(&slot->active_cv)
ReplicationSlotsComputeRequiredXmin(false)
ReplicationSlotsComputeRequiredLSN()
rmtree(tmppath)
pgstat_drop_replslot(slot) if logical
LWLockRelease(ReplicationSlotAllocationLock)
```
为什么先 rename 目录？
因为 crash-safe drop 的 durable 边界在文件系统。
如果先清 `in_use`，
然后 crash，
startup 仍会看到 `pg_replslot/<name>/state` 并恢复它。
shared memory 的删除没有跨 crash 意义。
rename 后：
```text
pg_replslot/<name> 不再存在；
StartupReplicationSlots() 不会把它当作 valid slot；
如果 crash 留下 <name>.tmp，startup 会删除 .tmp。
```
所以手工 drop 后的状态是：
```text
slot 不存在
in_use = false
不再参与 WAL / xmin 保留
pg_replication_slots 不再显示
磁盘目录最终被 rmtree 删除
```
这和 invalidated slot 的“存在但无效”完全不同。
## 17. invalidated slot 的观测
`pg_replication_slots` 定义在 `system_views.sql`，
底层来自 `pg_get_replication_slots()`。
`slotfuncs.c` 的读取模型是：
```text
LWLockAcquire(ReplicationSlotControlLock, LW_SHARED)
for each in-use slot:
  SpinLockAcquire(&slot->mutex)
  slot_contents = *slot
  SpinLockRelease(&slot->mutex)
  根据拷贝组装 SQL 行
LWLockRelease(ReplicationSlotControlLock)
```
因此视图是一瞬间的 shared memory 拷贝。
它不是因果证明，
也不是未来 checkpoint 行为的保证。
对 invalidated slot，关键列是：
```sql
select slot_name,
       slot_type,
       database,
       active,
       active_pid,
       restart_lsn,
       xmin,
       catalog_xmin,
       wal_status,
       safe_wal_size,
       conflicting,
       invalidation_reason,
       inactive_since
from pg_replication_slots
order by slot_name;
```
解释顺序：
```text
invalidation_reason:
  NULL 表示没有 invalidated。
  wal_removed / rows_removed / wal_level_insufficient / idle_timeout 表示承诺失败原因。
wal_status:
  如果 slot invalidated，pg_get_replication_slots() 直接按 WALAVAIL_REMOVED 处理，
  通常显示 lost。
conflicting:
  logical slot 上 rows_removed 或 wal_level_insufficient 时为 true。
restart_lsn:
  wal_removed 会把它置为 NULL。
  其它 invalidation cause 可能保留旧值，但 slot 已经不再保护资源。
safe_wal_size:
  lost 或 max_slot_wal_keep_size 未配置时为 NULL。
```
常见诊断结论：
```text
invalidation_reason is not null:
  不要试图继续使用该 slot。
  先确认下游重建策略，再 drop。
wal_status = unreserved and invalidation_reason is null:
  还没 lost，但下一次 checkpoint 可能变成 wal_removed。
active = false and inactive_since 很老:
  可能触发 idle_timeout，或者已经说明消费者长期不再使用。
active = true:
  invalidation 可能需要先终止 owner 或等待 owner release。
```
## 18. invalidated slot 的安全删除
invalidated slot 不会自动消失。
它仍然占用 slot 名称和 shared memory 槽位，
也可能保留统计项。
安全删除步骤应围绕“下游是否已经重建”：
```text
1. 查 pg_replication_slots，确认 invalidation_reason。
2. 查 consumer 或 subscription 状态，确认它不能从旧 slot 继续。
3. 如果是 logical subscription，按业务流程重建 slot 或重建订阅。
4. 在 primary 上执行 pg_drop_replication_slot('<slot>')。
5. 再查 pg_replication_slots，确认 slot 消失。
```
SQL 形态：
```sql
select slot_name, slot_type, active, active_pid,
       invalidation_reason, wal_status
from pg_replication_slots
where invalidation_reason is not null;
select pg_drop_replication_slot('slot_name');
```
如果 drop 报 active PID，
不要直接删除文件系统目录。
先定位 owner：
```sql
select pid, backend_type, state, wait_event_type, wait_event, query
from pg_stat_activity
where pid = <active_pid>;
```
再选择停止 consumer、断开 walsender，
或者使用 replication protocol 的 `DROP_REPLICATION_SLOT ... WAIT` 语义等待 release。
不建议手工 `rm -rf pg_replslot/<name>`。
这样会绕过：
```text
ReplicationSlotAllocationLock
ReplicationSlotControlLock
active_cv wakeup
ReplicationSlotsComputeRequiredXmin()
ReplicationSlotsComputeRequiredLSN()
pgstat_drop_replslot()
crash-safe rename / fsync
```
手工删目录可能让当前 postmaster 的 shared memory 仍认为 slot 存在，
直到 restart 后状态才被迫对齐。
## 19. 四个边界状态对照
本节唯一主问题最后要落到四个具体边界。
WAL 丢失：
```text
触发:
  checkpoint / restartpoint 删除旧 WAL 前发现 restart_lsn 太老。
源码:
  InvalidateObsoleteReplicationSlots(RS_INVAL_WAL_REMOVED | RS_INVAL_IDLE_TIMEOUT, ...)
状态:
  data.invalidated = RS_INVAL_WAL_REMOVED
  restart_lsn 通常被置为 Invalid
  slot 仍存在
  不再保护 WAL
观测:
  invalidation_reason = 'wal_removed'
  wal_status = 'lost'
```
数据库删除：
```text
触发:
  DROP DATABASE 或 redo database drop。
源码:
  ReplicationSlotsCountDBSlots()
  ReplicationSlotsDropDBSlots()
状态:
  目标 database 的 logical slots 被 drop。
  active logical slot 会阻止 DROP DATABASE。
  不产生 invalidation_reason。
观测:
  slot 从 pg_replication_slots 消失。
```
catalog xmin 过旧：
```text
触发:
  standby recovery replay 移除 logical slot 仍需要的 row / catalog horizon。
源码:
  ResolveRecoveryConflictWithSnapshot()
  InvalidateObsoleteReplicationSlots(RS_INVAL_HORIZON, ...)
状态:
  data.invalidated = RS_INVAL_HORIZON
  slot 仍存在
  不再保护 xmin / catalog_xmin
观测:
  invalidation_reason = 'rows_removed'
  conflicting = true
```
手工 drop：
```text
触发:
  pg_drop_replication_slot()
  DROP_REPLICATION_SLOT
源码:
  ReplicationSlotDrop()
  ReplicationSlotDropAcquired()
  ReplicationSlotDropPtr()
状态:
  rename pg_replslot/<name> 到 <name>.tmp
  in_use = false
  目录最终删除
  slot 不再存在
观测:
  pg_replication_slots 不再显示。
```
这个对照不是独立清单，
而是同一条 slot 生命周期在四个不可逆边界上的分叉。
诊断时先判断分叉类型，
再决定是等消费者追赶、重建下游、调整配置，还是直接 drop。
## 20. 常见误区
1. 误以为 `pg_replication_slots` 里看不到 slot 就说明它只是 invalidated。
   invalidated slot 仍会显示；看不到通常是 dropped 或未恢复。
2. 误以为 `wal_removed` 只是 `wal_status='lost'` 的另一个名字。
   `wal_removed` 是持久 invalidation reason；`wal_status` 是视图对 WAL 可用性的推断。
3. 误以为 `RS_INVAL_HORIZON` 和 `rows_removed` 是两个原因。
   前者是源码 enum，后者是 SQL 显示字符串。
4. 误以为 database drop 会留下 invalidated logical slot。
   正常路径是 `ReplicationSlotsDropDBSlots()` 直接删除目标 DB 的 logical slots。
5. 误以为 active slot 不能被 invalidation 影响。
   checkpoint / startup process 会 signal owner，等待 release 后重试。
6. 误以为 `confirmed_flush_lsn` 保存不及时一定破坏 correctness。
   普通倒退通常导致多读或重复确认；catalog xmin 放宽前未保存才是 correctness 边界。
7. 误以为删除 `pg_replslot` 目录等价于 `pg_drop_replication_slot()`。
   它绕过锁、条件变量、全局边界重算和 stats cleanup。
8. 误以为 slot state 是 WAL redo 出来的。
   它是 `pg_replslot/<name>/state` 的独立 crash-safe 文件协议。
## 21. 课堂实验
### 21.1 观察 state 文件和 drop rename 边界
创建 persistent physical slot：
```sql
select * from pg_create_physical_replication_slot('p_persist_demo', true, false);
select slot_name, active, restart_lsn, wal_status, invalidation_reason
from pg_replication_slots
where slot_name = 'p_persist_demo';
```
在数据目录观察：
```text
pg_replslot/p_persist_demo/state
```
源码断点：
```text
ReplicationSlotCreate
CreateSlotOnDisk
SaveSlotToPath
ReplicationSlotRelease
ReplicationSlotDropPtr
```
drop：
```sql
select pg_drop_replication_slot('p_persist_demo');
```
要画出的状态：
```text
pg_replslot/<name>
pg_replslot/<name>.tmp
in_use
active_proc
data.persistency
```
### 21.2 观察 invalidated slot 的 SQL 形态
这个实验不要求在生产配置上真的制造 WAL 丢失。
可以在测试实例中降低 `max_slot_wal_keep_size`，
停止消费者，
持续写 WAL 并触发 checkpoint。
观察：
```sql
select slot_name, active, restart_lsn,
       wal_status, safe_wal_size, invalidation_reason
from pg_replication_slots
where slot_name = '<slot>';
```
源码断点：
```text
InvalidateObsoleteReplicationSlots
InvalidatePossiblyObsoleteSlot
DetermineSlotInvalidationCause
ReportSlotInvalidation
ReplicationSlotSave
```
预期解释：
```text
有效 slot 先可能进入 unreserved 风险区。
checkpoint 发现 required WAL 已越界后，
slot 变成 invalidated，reason 为 wal_removed。
```
### 21.3 database drop 与 logical slot
创建 logical slot 后尝试 drop database。
如果 slot active，
预期 `DROP DATABASE` 报 active logical slot。
如果 slot inactive，
drop database 会删除该 database 的 logical slot。
断点：
```text
ReplicationSlotsCountDBSlots
ReplicationSlotsDropDBSlots
ReplicationSlotDropAcquired
ReplicationSlotDropPtr
```
要解释：
```text
这条路径不会产生 invalidation_reason。
slot 消失是 database drop 的一部分。
```
## 22. 讨论题
1. 为什么 `SaveSlotToPath()` 必须写 `state.tmp`，而不是原地覆盖 `state`？
2. 为什么 `RestoreSlotFromDisk()` 在 crash recovery 删除旧资源前执行？
3. 为什么 persistent slot 的 WAL 保留要看 `last_saved_restart_lsn`？
4. 为什么 logical slot 推进 `catalog_xmin` 要先保存，再更新 `effective_catalog_xmin`？
5. `RS_INVAL_HORIZON` 为什么在视图中显示为 `rows_removed`？
6. `wal_removed` 和手工 drop 在观测上有什么根本区别？
7. `DROP DATABASE` 为什么删除 logical slots，而不是给它们加一个 database_dropped reason？
8. 如果 drop slot 时 `rmtree(<name>.tmp)` 失败，为什么可以只 WARNING？
## 23. 本节小结
Replication slot 持久化的核心链路是：
```text
shared memory 中更新 slot->data
  -> dirty / just_dirtied 表示需要保存
  -> SaveSlotToPath() 用 state.tmp + fsync + rename 保存 durable truth
  -> checkpoint 批量保存 dirty slot
  -> startup 在资源删除前 RestoreSlotFromDisk()
  -> 恢复后重新计算 WAL / xmin 保留边界
```
边界状态要分清：
```text
WAL 丢失:
  invalidated，reason = wal_removed，slot 仍存在。
catalog xmin 过旧:
  invalidated，reason = rows_removed，logical slot 仍存在但不能继续。
wal_level 不足:
  invalidated，reason = wal_level_insufficient，主要是 standby logical decoding 边界。
idle timeout:
  invalidated，reason = idle_timeout，是运维策略触发。
数据库删除:
  database-specific logical slot 被 drop，不留下 invalidation reason。
手工 drop:
  目录 rename 到 .tmp，shared in_use 清零，slot 消失。
```
本节可迁移的系统规律是：
```text
持久保护机制必须区分“当前内存进度”和“crash 后可证明的 durable 进度”；
放宽全局清理边界前，必须先让恢复路径能重建同样或更保守的承诺；
当承诺已经无法满足时，系统要显式进入 invalidated 或 dropped 状态，
而不是让消费者在不完整历史上继续运行。
```
诊断时不要只看一个字段。
`restart_lsn`、`last_saved_restart_lsn`、`catalog_xmin`、`effective_catalog_xmin`、
`invalidated`、`active_proc`、`wal_status`、`conflicting` 和 filesystem state
合在一起，才是 slot 在崩溃恢复和资源回收边界上的真实语义。
