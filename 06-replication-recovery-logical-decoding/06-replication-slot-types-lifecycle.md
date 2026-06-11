# PostgreSQL Replication Slot 类型与生命周期
## 课程定位
前置知识：已经理解 WAL segment、checkpoint、WAL sender / WAL receiver、Hot Standby feedback、MVCC xmin horizon 的基本概念，也知道逻辑解码需要从 WAL 中重建事务和 catalog 语义。
本节唯一主问题：
```text
physical slot、logical slot、temporary slot 和 persistent slot 分别保护什么状态，
创建、持有、释放和崩溃恢复时哪些字段必须可靠保存？
```
核心矛盾：slot 要让复制消费者断线后还能继续追赶，因此必须保留 WAL、xmin 和 logical decoding 进度；但 primary 不能无限保留 WAL 和旧 tuple，也不能让已经无效的 slot 继续阻止清理。PostgreSQL 的选择是把 slot 拆成共享内存运行态和可校验的磁盘状态，并用 active ownership、persistency、invalidation 和 checkpoint flush 顺序共同约束它。
学完后应能判断：为什么 `physical` / `logical` 不是和 `temporary` / `persistent` 同一维度；为什么 `restart_lsn` 对所有 slot 都关键，但 `confirmed_flush` 主要是 logical slot 的消费位置；为什么 logical slot 的 `catalog_xmin` 必须先可靠保存再放宽全局清理边界；为什么 `in_use`、`active_proc`、`invalidated` 不能单独解释；以及 `pg_replication_slots` 里 `active_pid`、`wal_status`、`safe_wal_size`、`invalidation_reason` 分别能诊断什么。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
前几节复制课已经建立了 physical streaming 的主链路：
```text
standby 建立 replication connection
  -> primary 上 walsender 读取 WAL
  -> standby 上 walreceiver 写入 WAL segment
  -> standby 周期性反馈 write / flush / apply 位置
  -> primary 可能根据反馈推进同步复制等待
```
这条链路还缺一个状态问题：
```text
消费者断开时，primary 怎么知道哪些 WAL 和旧版本不能清？
```
如果只靠连接中的 walsender，连接断开后状态就消失。
如果把所有可能需要的 WAL 都永久保留，primary 的磁盘会被复制拓扑拖死。
如果把所有旧 tuple 都等 standby 或 logical consumer 确认后再清，VACUUM 会失去推进边界。
Replication slot 就是这个折中点：
```text
slot 把复制消费者的最低需求发布成 primary 上的持久状态；
WAL 删除、VACUUM 清理、logical decoding 起点和监控视图都围绕这个状态判断。
```
本节只讲 slot 类型和生命周期。
下一节会把焦点收窄到 `restart_lsn` 如何变成 WAL 保留边界，以及 `max_slot_wal_keep_size` 为什么会把保护需求转化成磁盘压力。
## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
postmaster 启动期按 max_replication_slots 分配 ReplicationSlot 共享数组；
创建 slot 时先写 pg_replslot/<name>/state，再把共享数组中的 in_use 和 active_proc 打开；
walsender 或 SQL 函数 acquire 后独占更新 slot 的 restart_lsn、xmin、catalog_xmin、confirmed_flush 等字段；
release 时 persistent slot 只清 active_proc，temporary / ephemeral slot 会被 drop；
checkpoint 把 dirty 的 persistent slot 原子写回磁盘；
startup 从磁盘只恢复 RS_PERSISTENT slot，并据此重新发布 WAL 和 xmin 保留边界。
```
这里有两个正交分类：
```text
physical vs logical:
  slot 保护的是物理 WAL 流，还是逻辑解码语义。
persistent vs temporary:
  slot 状态是否应该跨 release 和 crash restart 保留。
```
还有一个内部过渡态：
```text
RS_EPHEMERAL:
  创建 persistent logical slot 时的 not-yet-ready 状态；
  初始化失败或 ERROR 时像临时对象一样被丢弃；
  只有初始化完成后 ReplicationSlotPersist() 才变成 RS_PERSISTENT。
```
关键 tension 是：
```text
slot 必须足够持久，避免 crash 后忘记消费者还需要的 WAL / catalog tuple
  vs
slot 必须足够可失效、可删除、可释放，避免长期阻塞 WAL 回收和 VACUUM
```
这不是一个字段能解决的问题。
`restart_lsn`、`confirmed_flush`、`xmin`、`catalog_xmin`、`invalidated`、`in_use`、`active_proc`、`persistency`、`last_saved_restart_lsn` 和 `effective_catalog_xmin` 合在一起，才构成 slot 的真实语义。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/slot.h` | `ReplicationSlotPersistentData`、`ReplicationSlot`、`ReplicationSlotPersistency`、`ReplicationSlotInvalidationCause`。 |
| 2 | `src/backend/replication/slot.c` | slot shared memory、create/acquire/release/drop、save/restore/checkpoint、invalidation、全局 xmin/LSN 计算。 |
| 3 | `src/backend/replication/slotfuncs.c` | SQL 函数、`pg_get_replication_slots()`、`pg_replication_slots` 可观测字段来源。 |
| 4 | `src/backend/replication/logical/logical.c` | logical slot 初始化、consistent point、`confirmed_flush`、candidate xmin / restart_lsn 推进。 |
| 5 | `src/backend/replication/walsender.c` | replication protocol 创建 slot、physical/logical streaming acquire、standby reply 如何推进 slot。 |
| 6 | `src/backend/access/transam/xlog.c` | startup / checkpoint / WAL 删除如何调用 slot 层，`GetWALAvailability()` 如何解释 `wal_status`。 |
| 7 | `src/backend/catalog/system_views.sql` | `pg_replication_slots` view 如何包装 `pg_get_replication_slots()`。 |
推荐阅读顺序：
```text
先读 slot.h 的两个结构体
  -> 看 ReplicationSlotCreate() 如何建立 shared + disk state
  -> 看 ReplicationSlotAcquire()/Release() 如何独占 active_proc
  -> 看 physical 和 logical 分别如何推进 restart_lsn / confirmed_flush / xmin
  -> 看 CheckPointReplicationSlots() 和 StartupReplicationSlots()
  -> 最后回到 pg_get_replication_slots() 解释监控字段
```
不要从 `pg_replication_slots` 开始背字段。
视图只是运行态拷贝。
正确入口是“哪个字段阻止哪类清理，以及 crash 后必须如何重建这个阻止关系”。
## 4. 关键数据结构与状态
### 4.1 `ReplicationSlotPersistentData`: 会写入 `pg_replslot` 的语义
`ReplicationSlotPersistentData` 定义在 `src/include/replication/slot.h`。
它是 slot 的磁盘状态，也是 crash 后可恢复语义的核心。
| 字段 | 语义 |
| --- | --- |
| `name` | slot 标识，也是 `pg_replslot/<name>` 目录名。 |
| `database` | `InvalidOid` 表示 physical slot；非 `InvalidOid` 表示 database-specific logical slot。 |
| `persistency` | `RS_PERSISTENT`、`RS_EPHEMERAL`、`RS_TEMPORARY`。 |
| `xmin` | 数据 tuple 的保留 horizon，主要来自 physical standby feedback 或 logical 初始 snapshot 需要。 |
| `catalog_xmin` | catalog tuple 的保留 horizon，logical decoding 依赖它解释历史 schema。 |
| `restart_lsn` | 这个 slot 可能还需要读取的最老 WAL LSN。 |
| `invalidated` | slot 是否已失效，以及失效原因。 |
| `confirmed_flush` | logical consumer 已确认安全收到的最老消费位置之后的进度。 |
| `two_phase_at` / `two_phase` | two-phase decoding 能力和生效 LSN。 |
| `plugin` | logical slot 使用的 output plugin 名。 |
| `synced` / `failover` | logical failover slot 和 slot synchronization 相关状态。 |
这里最容易误读的是 `database`：
```text
database == InvalidOid:
  physical slot。
  保护 WAL 流和可选的 standby feedback xmin。
database != InvalidOid:
  logical slot。
  绑定创建时所在 database，需要 plugin 和 catalog horizon。
```
所以 `SlotIsPhysical(slot)` 和 `SlotIsLogical(slot)` 只是根据 `data.database` 判断。
它们不是独立字段。
### 4.2 `ReplicationSlot`: shared memory 运行态
`ReplicationSlot` 也在 `slot.h`。
它位于 `ReplicationSlotCtl->replication_slots[]` 共享数组中。
所有 backend 都能扫描，但只有 acquire 当前 slot 的 backend 才能更新它的主状态。
| 字段组 | 语义 |
| --- | --- |
| `mutex` | 单个 slot 字段的短临界区。 |
| `in_use` | 这个数组槽位是否定义了一个 slot。由 `ReplicationSlotControlLock` 保护。 |
| `active_proc` | 当前哪个 `PGPROC` 正在持有 slot，空闲时为 `INVALID_PROC_NUMBER`。 |
| `just_dirtied` / `dirty` | 是否需要把 `data` 写回磁盘。 |
| `effective_xmin` / `effective_catalog_xmin` | 当前真正发布给全局清理边界的 xmin。 |
| `data` | 会保存到磁盘的 `ReplicationSlotPersistentData`。 |
| `io_in_progress_lock` | 单 slot 磁盘写入互斥。 |
| `active_cv` | acquire/drop 等待 `active_proc` 改变。 |
| `candidate_catalog_xmin` / `candidate_xmin_lsn` | logical slot 等待 consumer 确认后才能应用的 catalog xmin 候选。 |
| `candidate_restart_valid` / `candidate_restart_lsn` | logical slot 等待确认后才能应用的 restart_lsn 候选。 |
| `last_saved_confirmed_flush` | 上次已落盘的 `confirmed_flush`，shutdown checkpoint 用它决定是否强制保存。 |
| `last_saved_restart_lsn` | 上次已落盘的 `restart_lsn`，persistent slot 计算 WAL 保留时要考虑它。 |
| `inactive_since` | slot 变为空闲的时间，idle timeout 和诊断使用。 |
| `slotsync_skip_reason` | slot synchronization 最近一次跳过原因，只在内存中有意义。 |
`active_proc` 存的是 `ProcNumber`，不是 pid。
视图里的 `active_pid` 是 `GetPGProcByNumber(active_proc)->pid` 反查出来的。
这一点解释了两个现象：
```text
active = true:
  当前有 backend 持有 slot。
active_pid:
  只是当前持有者的 OS pid。
  release、进程退出、slot 被 invalidation 终止后都会变化。
```
### 4.3 锁模型：数组定义和单 slot 字段分开
`slot.c` 文件头部注释给出锁模型：
```text
ReplicationSlotAllocationLock:
  allocate / free slot，保护目录创建、删除和名称复用。
ReplicationSlotControlLock:
  shared 模式扫描 slot；
  exclusive 模式修改 in_use。
slot->mutex:
  保护单个 slot 的 active_proc、data 字段、dirty 标记等。
slot->io_in_progress_lock:
  同一 slot 的 state 文件写入互斥。
```
这形成一个重要边界：
```text
in_use 说明数组槽位是否定义；
active_proc 说明是否被某个 backend 持有；
dirty 说明磁盘状态是否滞后；
invalidated 说明语义是否已经不能继续使用。
```
任何一个字段单独都不是完整状态。
例如：
```text
in_use = true, active_proc = INVALID_PROC_NUMBER:
  persistent slot 存在但当前空闲。
in_use = true, active_proc = MyProcNumber:
  当前 backend 独占持有，可以更新主字段。
in_use = true, invalidated != RS_INVAL_NONE:
  slot 仍存在，视图能看到，但不再保护资源，acquire 时会报错。
in_use = false:
  这个数组槽位没有定义 slot，其它字段不应被解释。
```
### 4.4 physical/logical 与 temporary/persistent 是两条轴
把两条轴分清，后面生命周期才不会混乱。
| 维度 | 值 | 决定什么 |
| --- | --- | --- |
| 类型轴 | physical | `data.database == InvalidOid`，保护物理 WAL 流，可能承载 standby feedback xmin。 |
| 类型轴 | logical | `data.database != InvalidOid`，保护 logical decoding 所需 WAL、catalog tuple 和 confirmed progress。 |
| 持久性轴 | persistent | crash 后从 `pg_replslot` 恢复，release 不删除。 |
| 持久性轴 | temporary | session / ERROR / restart 后删除，release 后仍作为 temporary cleanup 对象处理。 |
| 持久性轴 | ephemeral | 创建 persistent logical slot 的内部过渡态，失败时删除，成功后转 persistent。 |
因此下面这些组合都可能出现：
```text
persistent physical slot:
  常见 physical standby slot。
temporary physical slot:
  当前会话临时保留 WAL，session 结束清理。
persistent logical slot:
  publication / logical consumer 常见状态。
temporary logical slot:
  临时解码或调试使用，不跨会话。
```
`RS_EPHEMERAL` 不应被理解成用户类型。
它服务创建流程的 error-safety：
```text
先创建一个会自动清理的 logical slot
  -> 找到 consistent point
  -> 写好 restart_lsn / catalog_xmin / confirmed_flush
  -> 成功后 ReplicationSlotPersist()
```
如果中间失败，slot 不会留下一个半初始化的 persistent 保护边界。
## 5. 主流程源码 walkthrough
### 5.1 启动期：预分配共享数组
slot 共享状态不是按需 `malloc`。
`ReplicationSlotsShmemRequest()` 按 GUC 预估大小：
```text
offsetof(ReplicationSlotCtlData, replication_slots)
  + (max_replication_slots + max_repack_replication_slots) * sizeof(ReplicationSlot)
```
`ReplicationSlotsShmemInit()` 初始化每个数组元素：
```text
slot->active_proc = INVALID_PROC_NUMBER
SpinLockInit(&slot->mutex)
LWLockInitialize(&slot->io_in_progress_lock, LWTRANCHE_REPLICATION_SLOT_IO)
ConditionVariableInit(&slot->active_cv)
```
注意它没有把任何 slot 标成 `in_use`。
`in_use` 只有从磁盘 restore 或 create 新 slot 时才打开。
当前 backend 的全局指针是：
```text
MyReplicationSlot:
  当前 backend acquire 的 slot；
  没有持有 slot 时为 NULL。
```
进程退出时 `ReplicationSlotInitialize()` 注册 `before_shmem_exit(ReplicationSlotShmemExit, 0)`。
退出回调做两件事：
```text
如果 MyReplicationSlot != NULL:
  ReplicationSlotRelease()
然后:
  ReplicationSlotCleanup(false)
  清理当前 session 创建的 temporary slots
```
这就是 temporary slot 的 ERROR / session 兜底路径。
### 5.2 创建 physical slot：先落盘目录，再公开 `in_use`
SQL 入口是 `pg_create_physical_replication_slot()`，实现位于 `slotfuncs.c`。
replication protocol 入口是 `walsender.c` 的 `CreateReplicationSlot()`。
两者最终都会调用：
```text
ReplicationSlotCreate(name, false, persistency, ...)
```
其中 `db_specific = false`，所以：
```text
slot->data.database = InvalidOid
```
创建流程在 `ReplicationSlotCreate()` 中展开：
```text
ReplicationSlotValidateName()
LWLockAcquire(ReplicationSlotAllocationLock, LW_EXCLUSIVE)
  -> 扫描名称冲突和空闲槽位
  -> 初始化 slot->data
  -> 初始化 shared-memory-only 字段
  -> CreateSlotOnDisk(slot)
  -> ReplicationSlotControlLock exclusive
       slot->in_use = true
       slot->active_proc = MyProcNumber
       MyReplicationSlot = slot
  -> logical slot 时创建 stats entry
LWLockRelease(ReplicationSlotAllocationLock)
ConditionVariableBroadcast(&slot->active_cv)
```
这里顺序很关键：
```text
先 CreateSlotOnDisk()
  再 in_use = true
```
在 `in_use` 打开前，别的 backend 不会把这个 slot 当成已定义对象扫描。
如果 `CreateSlotOnDisk()` 失败，也不需要复杂 cleanup，因为 slot 还没有公开。
如果用户要求立即保留 WAL：
```text
ReplicationSlotReserveWal()
ReplicationSlotMarkDirty()
ReplicationSlotSave()   // persistent physical slot 才需要立即写回
```
`ReplicationSlotReserveWal()` 对 physical slot 的起点选择是：
```text
SlotIsPhysical(slot):
  restart_lsn = GetRedoRecPtr()
```
这个值表示从 checkpoint redo 位置开始，base backup / physical standby 才有可恢复起点。
### 5.3 创建 logical slot：先 ephemeral，再找 consistent point
logical slot 创建路径比 physical 更严格。
SQL 入口 `pg_create_logical_replication_slot()` 调用 `create_logical_replication_slot()`。
replication protocol 的 `CREATE_REPLICATION_SLOT ... LOGICAL` 走 `walsender.c` 的 `CreateReplicationSlot()`。
核心步骤：
```text
ReplicationSlotCreate(name, true,
  temporary ? RS_TEMPORARY : RS_EPHEMERAL,
  two_phase, ...)
EnsureLogicalDecodingEnabled()
CreateInitDecodingContext(plugin, ..., restart_lsn = InvalidXLogRecPtr, ...)
DecodingContextFindStartpoint(ctx)
FreeDecodingContext(ctx)
if !temporary:
  ReplicationSlotPersist()
ReplicationSlotRelease()
```
`db_specific = true` 会设置：
```text
slot->data.database = MyDatabaseId
```
`CreateInitDecodingContext()` 做三类事情。
第一，写入 plugin 名并设置 `restart_lsn`：
```text
namestrcpy(&slot->data.plugin, plugin)
if restart_lsn invalid:
  ReplicationSlotReserveWal()
else:
  slot->data.restart_lsn = restart_lsn
```
logical slot 的 `ReplicationSlotReserveWal()` 起点不同：
```text
standby 上:
  restart_lsn = GetXLogReplayRecPtr(NULL)
primary 上:
  restart_lsn = GetXLogInsertRecPtr()
  然后 LogStandbySnapshot()
  XLogFlush(flushptr)
```
第二，建立初始 decoding xmin horizon：
```text
LWLockAcquire(ReplicationSlotControlLock, LW_EXCLUSIVE)
LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE)
  xmin_horizon = GetOldestSafeDecodingTransactionId(...)
  slot->effective_catalog_xmin = xmin_horizon
  slot->data.catalog_xmin = xmin_horizon
  if need_full_snapshot:
    slot->effective_xmin = xmin_horizon
  ReplicationSlotsComputeRequiredXmin(true)
release locks
```
这里同时设置 `data.catalog_xmin` 和 `effective_catalog_xmin`。
`data.catalog_xmin` 是 crash 后能恢复的持久边界。
`effective_catalog_xmin` 是当前内存中发布给 VACUUM / horizon 计算的边界。
第三，立即保存：
```text
ReplicationSlotMarkDirty()
ReplicationSlotSave()
```
这样 logical slot 在开始构造 snapshot 后，不会因为 crash 忘记自己保护了 catalog tuple。
### 5.4 找到 logical consistent point 并设置 `confirmed_flush`
`DecodingContextFindStartpoint()` 从 `slot->data.restart_lsn` 开始读 WAL：
```text
XLogBeginRead(ctx->reader, slot->data.restart_lsn)
loop:
  record = XLogReadRecord(ctx->reader, &err)
  LogicalDecodingProcessRecord(ctx, ctx->reader)
  if DecodingContextReady(ctx):
    break
```
找到 consistent initial decoding snapshot 后：
```text
slot->data.confirmed_flush = ctx->reader->EndRecPtr
if slot->data.two_phase:
  slot->data.two_phase_at = ctx->reader->EndRecPtr
```
这解释了 logical slot 创建函数返回的 `consistent_point`。
它不是随便的当前 WAL 末尾，而是 logical decoding 已经能从此后输出一致变更的位置。
创建 persistent logical slot 最后调用：
```text
ReplicationSlotPersist()
  -> slot->data.persistency = RS_PERSISTENT
  -> ReplicationSlotMarkDirty()
  -> ReplicationSlotSave()
```
这一步完成后，slot 才真正承诺 crash 后存在。
如果前面任何一步 ERROR：
```text
RS_EPHEMERAL:
  ReplicationSlotRelease() 时会 ReplicationSlotDropAcquired()
  不会留下半成品 persistent logical slot。
```
### 5.5 Acquire：把 `active_proc` 从空闲变成当前 backend
使用已有 slot 的入口是 `ReplicationSlotAcquire(name, nowait, error_if_invalid)`。
典型调用点：
```text
walsender.c StartReplication():
  physical replication 使用 physical slot。
walsender.c StartLogicalReplication():
  logical replication 使用 logical slot。
slotfuncs.c pg_replication_slot_advance():
  SQL 手工推进 slot。
slot.c ReplicationSlotDrop():
  drop 前先 acquire。
```
`ReplicationSlotAcquire()` 的主流程：
```text
retry:
  LWLockAcquire(ReplicationSlotControlLock, LW_SHARED)
  s = SearchNamedReplicationSlot(name, false)
  if not found:
    ERROR
  if !nowait:
    ConditionVariablePrepareToSleep(&s->active_cv)
  SpinLockAcquire(&s->mutex)
    if s->active_proc == INVALID_PROC_NUMBER:
      s->active_proc = MyProcNumber
    active_proc = s->active_proc
    ReplicationSlotSetInactiveSince(s, 0, false)
  SpinLockRelease(&s->mutex)
  active_pid = GetPGProcByNumber(active_proc)->pid
  LWLockRelease(ReplicationSlotControlLock)
  if active_proc != MyProcNumber:
    if !nowait:
      ConditionVariableSleep(...)
      goto retry
    else:
      ERROR "replication slot is active for PID ..."
  MyReplicationSlot = s
  if error_if_invalid && s->data.invalidated != RS_INVAL_NONE:
    ERROR
  ConditionVariableBroadcast(&s->active_cv)
```
这里有两个边界。
第一，slot 是单 owner。
`active_proc` 只能有一个当前持有者。
所以同一个 slot 不能同时被两个 walsender 读取。
第二，失效检查必须在 acquire 之后做。
源码注释说明这是为了避免 checkpointer 在检查之后立刻 invalidation 的竞态。
只有当前 backend 成为 owner 后，后续语义才稳定。
### 5.6 Release：persistent 保留，ephemeral / temporary 清理
`ReplicationSlotRelease()` 根据 `persistency` 分支：
```text
if slot->data.persistency == RS_EPHEMERAL:
  ReplicationSlotDropAcquired(is_logical)
else:
  如果创建初始 snapshot 时临时设置了 effective_xmin:
    清掉 effective_xmin
    ReplicationSlotsComputeRequiredXmin(false)
  now = GetCurrentTimestamp()
  if RS_PERSISTENT:
    slot->active_proc = INVALID_PROC_NUMBER
    inactive_since = now
    ConditionVariableBroadcast(&slot->active_cv)
  else:
    ReplicationSlotSetInactiveSince(slot, now, true)
  MyReplicationSlot = NULL
```
看起来 `RS_TEMPORARY` release 没有在这里直接 drop。
这是因为 temporary slot 由 session cleanup 兜底：
```text
ReplicationSlotShmemExit()
  -> ReplicationSlotRelease()
  -> ReplicationSlotCleanup(false)
```
`ReplicationSlotCleanup(false)` 扫描所有 slot：
```text
if s->active_proc == MyProcNumber:
  Assert(s->data.persistency == RS_TEMPORARY)
  ReplicationSlotDropPtr(s)
```
也就是说 temporary slot 的核心承诺是：
```text
session 结束或 ERROR cleanup 后不保留；
crash restart 后也不会恢复。
```
### 5.7 Physical streaming：standby flush 反馈推进 `restart_lsn`
physical walsender 在 `StartReplication()` 中：
```text
if cmd->slotname:
  ReplicationSlotAcquire(cmd->slotname, true, true)
  if SlotIsLogical(MyReplicationSlot):
    ERROR
WalSndLoop(XLogSendPhysical)
if cmd->slotname:
  ReplicationSlotRelease()
```
standby 周期性发送 write / flush / apply。
primary 的 `ProcessStandbyReplyMessage()` 读取：
```text
writePtr
flushPtr
applyPtr
replyTime
replyRequested
```
如果当前 walsender 持有 slot：
```text
if SlotIsLogical(MyReplicationSlot):
  LogicalConfirmReceivedLocation(flushPtr)
else:
  PhysicalConfirmReceivedLocation(flushPtr)
```
physical slot 的推进很直接：
```text
PhysicalConfirmReceivedLocation(lsn)
  -> if slot->data.restart_lsn != lsn:
       slot->data.restart_lsn = lsn
       ReplicationSlotMarkDirty()
       ReplicationSlotsComputeRequiredLSN()
       PhysicalWakeupLogicalWalSnd()
```
源码注释特别强调：
```text
不立即 ReplicationSlotSave()。
physical slot 丢失最近一次 restart_lsn 更新，最坏只是更保守地保留 WAL，
或统计视图短期不精确；不是 logical decoding wrong answers。
```
这和 logical slot 完全不同。
### 5.8 Physical standby feedback：用 slot 承载 xmin
Hot Standby feedback 由 `ProcessStandbyHSFeedbackMessage()` 处理。
如果 walsender 没有 slot，反馈 xmin 写入 `MyProc->xmin`。
如果 walsender 持有 physical slot，则调用：
```text
PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)
```
该函数在 slot mutex 下更新：
```text
slot->data.xmin = feedbackXmin
slot->effective_xmin = feedbackXmin
slot->data.catalog_xmin = feedbackCatalogXmin
slot->effective_catalog_xmin = feedbackCatalogXmin
ReplicationSlotMarkDirty()
ReplicationSlotsComputeRequiredXmin(false)
```
注释说明 physical replication 不需要 logical 那种 `data` / `effective` 的强 interlock。
漏掉一次 xmin 前进的后果主要是 standby 上可能出现查询取消，而不是 logical decoding 输出错误。
所以 physical slot 保护的状态可以概括为：
```text
必须保护:
  restart_lsn 对应的 WAL。
可选保护:
  standby feedback 发布的 xmin / catalog_xmin。
```
### 5.9 Logical streaming：`restart_lsn` 是读取起点，`confirmed_flush` 是消费承诺
logical walsender 在 `StartLogicalReplication()` 中：
```text
ReplicationSlotAcquire(cmd->slotname, true, true)
logical_decoding_ctx = CreateDecodingContext(cmd->startpoint, ...)
XLogBeginRead(logical_decoding_ctx->reader,
              MyReplicationSlot->data.restart_lsn)
sentPtr = MyReplicationSlot->data.confirmed_flush
WalSndLoop(XLogSendLogical)
FreeDecodingContext(logical_decoding_ctx)
ReplicationSlotRelease()
```
`CreateDecodingContext()` 中，如果客户端没有指定 start LSN：
```text
start_lsn = slot->data.confirmed_flush
```
如果客户端指定了更早的 start LSN：
```text
start_lsn < slot->data.confirmed_flush:
  LOG 后前推到 confirmed_flush
```
这解释了两个位置的差异：
```text
restart_lsn:
  decoder 必须从这里读 WAL。
  因为有些未提交或未完整输出事务需要从更早 WAL 重组。
confirmed_flush:
  consumer 已确认安全收到的位置。
  断线重连时 logical stream 不能回到它之前。
```
consumer 确认 flush 后，`LogicalConfirmReceivedLocation(lsn)` 更新 `confirmed_flush`，并在条件满足时应用候选 xmin / restart LSN。
最关键的顺序在这里：
```text
如果 candidate_catalog_xmin 可以应用:
  slot->data.catalog_xmin = candidate_catalog_xmin
  ReplicationSlotMarkDirty()
  ReplicationSlotSave()
保存成功后:
  slot->effective_catalog_xmin = slot->data.catalog_xmin
  ReplicationSlotsComputeRequiredXmin(false)
```
为什么要先写磁盘？
```text
如果先放宽 effective_catalog_xmin，让 VACUUM 清掉旧 catalog tuple；
随后 crash，而 data.catalog_xmin 还没落盘；
restart 后 slot 会以为仍需要更老的 catalog tuple；
但那些 tuple 已经没了，logical decoding 可能产生错误答案。
```
这就是 logical slot 比 physical slot 更严格的 crash-safety 边界。
`restart_lsn` 的推进也通过 candidate 机制：
```text
LogicalIncreaseRestartDecodingForSlot(current_lsn, restart_lsn)
  -> 设置 candidate_restart_valid / candidate_restart_lsn
LogicalConfirmReceivedLocation(lsn)
  -> confirmed flush 越过 candidate_restart_valid 后
     data.restart_lsn = candidate_restart_lsn
     ReplicationSlotSave()
     ReplicationSlotsComputeRequiredLSN()
```
### 5.10 Drop：先让目录不可见，再清 `in_use`
`pg_drop_replication_slot()` 调用 `ReplicationSlotDrop(name, true)`。
`ReplicationSlotDrop()` 先 acquire slot，再调用：
```text
ReplicationSlotDropAcquired(SlotIsLogical(MyReplicationSlot))
  -> MyReplicationSlot = NULL
  -> ReplicationSlotDropPtr(slot)
  -> 如果 logical 且需要，RequestDisableLogicalDecoding()
```
`ReplicationSlotDropPtr()` 的关键顺序：
```text
LWLockAcquire(ReplicationSlotAllocationLock, LW_EXCLUSIVE)
rename(pg_replslot/<name>, pg_replslot/<name>.tmp)
fsync tmp dir
fsync pg_replslot
ReplicationSlotControlLock exclusive:
  slot->active_proc = INVALID_PROC_NUMBER
  slot->in_use = false
ConditionVariableBroadcast(&slot->active_cv)
ReplicationSlotsComputeRequiredXmin(false)
ReplicationSlotsComputeRequiredLSN()
rmtree(pg_replslot/<name>.tmp)
pgstat_drop_replslot(slot)  // logical slot
LWLockRelease(ReplicationSlotAllocationLock)
```
这里先 rename 目录，是安全删除边界。
如果 crash 发生在 rename 后、rmtree 前，startup 会看到 `.tmp` 目录并清理。
如果 rename 失败，persistent slot drop 会 ERROR；temporary / ephemeral 场景会尽量 WARNING，因为它们常在 cleanup 路径中执行。
## 6. 持久化、checkpoint 与崩溃恢复
### 6.1 state 文件格式
磁盘文件是：
```text
pg_replslot/<slot_name>/state
```
`slot.c` 中的 `ReplicationSlotOnDisk` 包含：
```text
magic
checksum
version
length
ReplicationSlotPersistentData slotdata
```
`magic`、`version`、`length` 和 CRC 让 startup 能判断 state 文件是否属于当前格式且未损坏。
不要把 `ReplicationSlot` 整个结构体想象成落盘。
落盘的只有 `ReplicationSlotPersistentData`。
`active_proc`、`dirty`、`candidate_*`、`effective_*`、`last_saved_*`、`inactive_since` 等运行态会在 restore 时重建。
### 6.2 `SaveSlotToPath()`: dirty check、临时文件、rename 和 fsync
`ReplicationSlotSave()` 只是当前持有 slot 的包装：
```text
ReplicationSlotSave()
  -> SaveSlotToPath(MyReplicationSlot, "pg_replslot/<name>", ERROR)
```
`SaveSlotToPath()` 的顺序：
```text
SpinLock:
  was_dirty = slot->dirty
  slot->just_dirtied = false
if !was_dirty:
  return
LWLockAcquire(&slot->io_in_progress_lock, LW_EXCLUSIVE)
open state.tmp with O_CREAT | O_EXCL
SpinLock:
  memcpy(&cp.slotdata, &slot->data, sizeof(...))
compute CRC
write state.tmp
fsync state.tmp
close
rename state.tmp -> state
critical section:
  fsync state
  fsync slot dir
  fsync pg_replslot
SpinLock:
  if !slot->just_dirtied:
    slot->dirty = false
  slot->last_saved_confirmed_flush = cp.slotdata.confirmed_flush
  slot->last_saved_restart_lsn = cp.slotdata.restart_lsn
release io lock
```
`just_dirtied` 解决一个并发细节：
```text
SaveSlotToPath() 正在写旧 snapshot 时，
另一个路径可能又把 slot 标 dirty。
保存完成后不能无条件 dirty=false，
否则会丢掉刚发生的新修改。
```
### 6.3 checkpoint：平时延迟保存，shutdown 更积极保存 logical `confirmed_flush`
`xlog.c` 的 `CheckPointGuts()` 会调用：
```text
CheckPointReplicationSlots(flags & CHECKPOINT_IS_SHUTDOWN)
```
`CheckPointReplicationSlots()` 做：
```text
LWLockAcquire(ReplicationSlotAllocationLock, LW_SHARED)
for each in-use slot:
  if shutdown && SlotIsLogical(s):
    if valid && data.confirmed_flush > last_saved_confirmed_flush:
      dirty = true
  if last_saved_restart_lsn != data.restart_lsn:
    remember to recompute required LSN
  SaveSlotToPath(s, path, LOG)
release lock
if any last_saved_restart_lsn updated:
  ReplicationSlotsComputeRequiredLSN()
```
这里不是每次 `confirmed_flush` 更新都同步落盘。
logical walsender 客户端应自己持久化消费位置。
但 shutdown checkpoint 会尽力保存更新过的 `confirmed_flush`，避免干净重启后不必要地回退。
physical slot 的 `restart_lsn` 也可以延迟保存。
但 persistent slot 计算 WAL 删除边界时会考虑 `last_saved_restart_lsn`：
```text
if persistent && last_saved_restart_lsn valid && data.restart_lsn > last_saved_restart_lsn:
  用 last_saved_restart_lsn 作为更保守的 WAL 保留边界
```
原因是 crash 后只能恢复上次保存的 `restart_lsn`。
如果 checkpoint 前就按更靠后的内存 `restart_lsn` 删除 WAL，crash 后恢复出来的 slot 可能需要已经删掉的旧 WAL。
### 6.4 startup：只恢复 persistent slot
`xlog.c` 在启动恢复路径中调用：
```text
StartupReplicationSlots()
```
`slot.c` 注释说它需要在有机会删除所需资源之前运行。
这保证 crash recovery / restart 早期就把 slot 的 WAL 和 xmin 保护边界恢复出来。
`StartupReplicationSlots()` 扫描 `pg_replslot`：
```text
跳过 . 和 ..
如果目录名以 .tmp 结尾:
  rmtree()
  fsync pg_replslot
  continue
否则:
  RestoreSlotFromDisk(name)
```
`RestoreSlotFromDisk()` 先清理残留 `state.tmp`，再读取 `state`：
```text
open state
fsync state
fsync slotdir
read constant part
verify magic
verify version
verify length
read rest
verify CRC
```
如果 `cp.slotdata.persistency != RS_PERSISTENT`：
```text
rmtree(slotdir)
fsync pg_replslot
return
```
所以 crash 后只有 persistent slot 会回到共享内存。
ephemeral 和 temporary 即使曾经写过目录，也会被删除。
恢复 persistent slot 时：
```text
memcpy(&slot->data, &cp.slotdata, sizeof(...))
slot->effective_xmin = cp.slotdata.xmin
slot->effective_catalog_xmin = cp.slotdata.catalog_xmin
slot->last_saved_confirmed_flush = cp.slotdata.confirmed_flush
slot->last_saved_restart_lsn = cp.slotdata.restart_lsn
slot->candidate_* = invalid
slot->in_use = true
slot->active_proc = INVALID_PROC_NUMBER
inactive_since = now
```
最后：
```text
ReplicationSlotsComputeRequiredXmin(false)
ReplicationSlotsComputeRequiredLSN()
```
也就是说 crash 后恢复的 slot 一定是 inactive 的。
它仍然保护资源，但没有 owner。
## 7. 正确性机制层次
### 7.1 WAL 保留边界
slot 层把最老需要的 LSN 传给 xlog 层：
```text
ReplicationSlotsComputeRequiredLSN()
  -> 扫描所有 in-use 且未 invalidated 的 slot
  -> 取最小 restart_lsn
  -> persistent slot 必要时用 last_saved_restart_lsn 更保守
  -> XLogSetReplicationSlotMinimumLSN(min_required)
```
`xlog.c` 中 `KeepLogSeg()` 读取：
```text
XLogGetReplicationSlotMinimumLSN()
```
并结合：
```text
wal_keep_size
max_slot_wal_keep_size
max_wal_size
current segment
```
决定旧 WAL segment 删除边界。
因此 `restart_lsn` 的语义不是“消费者当前读到哪里”。
它是：
```text
这个 slot 仍可能需要的最老 WAL 位置。
```
logical slot 常常会让 `restart_lsn` 落后于 `confirmed_flush`。
这是因为 decoder 可能需要更早 WAL 来重组未完成事务或推进 snapshot builder。
### 7.2 xmin / catalog_xmin 保留边界
slot 层把最老 xmin 传给 ProcArray：
```text
ReplicationSlotsComputeRequiredXmin()
  -> 扫描所有 in-use 且未 invalidated 的 slot
  -> 取最小 effective_xmin
  -> 取最小 effective_catalog_xmin
  -> ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin, ...)
```
这影响 VACUUM 和全局 horizon 判断。
区别仍然在 `effective_*`：
```text
data.xmin / data.catalog_xmin:
  crash 后能恢复的持久承诺。
effective_xmin / effective_catalog_xmin:
  当前运行中真正发布给清理边界的值。
```
logical slot 先保存 `data.catalog_xmin`，再推进 `effective_catalog_xmin`。
physical feedback 则可以同时更新，因为 missed advancement 的正确性代价不同。
### 7.3 active ownership
`active_proc` 保证同一 slot 同时只有一个 owner。
这不是为了 SQL 权限，而是为了字段更新的单写者模型：
```text
owner backend:
  可在 slot mutex 下更新 restart_lsn、confirmed_flush、xmin 等字段。
其它 backend:
  只能在 control lock + mutex 下拷贝观察，或等待 active_cv。
```
drop、invalidation 和 acquire 都围绕 `active_cv` 关闭竞态。
例如 drop 会先 acquire。
如果 slot 正在被别的 backend 持有，`nowait=true` 会报：
```text
replication slot "<name>" is active for PID <pid>
```
### 7.4 invalidation
`ReplicationSlotInvalidationCause` 有：
```text
RS_INVAL_WAL_REMOVED
RS_INVAL_HORIZON
RS_INVAL_WAL_LEVEL
RS_INVAL_IDLE_TIMEOUT
```
失效后的基本语义：
```text
slot 仍可在 pg_replication_slots 中看到；
invalidated slot 不再参与 required LSN / xmin 计算；
ReplicationSlotAcquire(..., error_if_invalid=true) 会 ERROR；
invalidation_reason 暴露失效原因。
```
`invalidated` 不是删除。
它是“这个 slot 的保护承诺已经无法继续满足”的显式标记。
## 8. 错误路径 / 异常路径 / fallback
### 8.1 创建中失败
physical persistent slot 创建时：
```text
CreateSlotOnDisk() 在 in_use 打开之前执行。
```
如果失败，数组槽位没有公开。
logical persistent slot 创建时：
```text
先用 RS_EPHEMERAL 创建；
找到 consistent point 后才 ReplicationSlotPersist()。
```
如果 plugin 加载失败、无法建立 consistent snapshot、snapshot option 不合法或读取 WAL 出错，release cleanup 会删除 ephemeral slot。
这就是 `RS_EPHEMERAL` 存在的主要理由。
### 8.2 保存失败
`SaveSlotToPath()` 的 `elevel` 由调用者决定。
前台需要强保证的路径用 `ERROR`。
checkpoint 用 `LOG`，因为它处于后台刷新场景。
写入路径通过：
```text
state.tmp
  -> fsync
  -> rename state
  -> fsync state
  -> fsync slot dir
  -> fsync pg_replslot
```
避免 crash 后看到半写 state。
如果 startup 看到 `state.tmp`，会删除它。
### 8.3 WAL 或 tuple 已经无法满足
`InvalidateObsoleteReplicationSlots()` 在 checkpoint / restartpoint 或相关边界调用。
它根据 possible causes 判断 slot 是否需要失效：
```text
RS_INVAL_WAL_REMOVED:
  restart_lsn < oldestLSN
RS_INVAL_HORIZON:
  logical slot 的 effective xmin / catalog xmin <= conflict horizon
RS_INVAL_WAL_LEVEL:
  logical slot 但 effective wal_level 不足
RS_INVAL_IDLE_TIMEOUT:
  inactive 超过 idle_replication_slot_timeout
```
如果 slot inactive：
```text
在 mutex 下设置 active_proc = MyProcNumber
设置 data.invalidated = cause
必要时清 restart_lsn
ReplicationSlotSave()
ReplicationSlotRelease()
```
如果 slot active：
```text
取 active_pid
释放 control lock
LOG
kill(active_pid, SIGTERM)
ConditionVariableSleep(&active_cv)
重新扫描
```
startup process 场景下使用 `SignalRecoveryConflict(..., RECOVERY_CONFLICT_LOGICALSLOT)`。
这里要注意一个安全退让：
```text
检查到冲突后，slot owner 可能在被终止前推进了 restart_lsn 或 xmin。
重新检查时如果已经不冲突，就不 invalidation。
```
这是有意的。
只要资源还没真正被删除，slot catch up 后就没必要失效。
### 8.4 Drop 中 crash
drop 先把目录 rename 到 `.tmp`。
startup 看到 `.tmp` 会删除。
因此 drop 的 crash 边界是：
```text
rename 前 crash:
  旧 slot 仍以正常目录存在，startup 可恢复。
rename 后 crash:
  startup 认为这是未完成删除或创建残留，清理 .tmp。
```
共享内存中的 `in_use=false` 不会跨 crash 保留。
真实持久语义以 `pg_replslot` 目录为准。
## 9. 成本、资源与跨模块传播
slot 的主要成本不是 CPU。
它的成本来自资源保留和全局边界传播。
### 9.1 WAL 空间
每个有效 slot 的 `restart_lsn` 都可能阻止 WAL segment 删除。
`ReplicationSlotsComputeRequiredLSN()` 取所有 slot 中最小值。
一个落后的 inactive persistent slot 就足以把 WAL 保留边界钉住。
后续 `KeepLogSeg()` 会受 `max_slot_wal_keep_size` 限制。
超过限制后，slot 可能进入：
```text
wal_status = unreserved
  还没删，但下一次 checkpoint 可能删除。
wal_status = lost
  需要的 WAL 已删除，slot 不能继续。
```
这些状态由 `GetWALAvailability()` 和 `pg_get_replication_slots()` 组合产生。
### 9.2 VACUUM 和 catalog bloat
logical slot 的 `catalog_xmin` 会保留 catalog tuple。
physical standby feedback 的 `xmin` / `catalog_xmin` 也可能保留旧版本。
传播链是：
```text
slot effective xmin
  -> ReplicationSlotsComputeRequiredXmin()
  -> ProcArraySetReplicationSlotXmin()
  -> ComputeXidHorizons() / VACUUM horizon
  -> dead tuple 和 catalog tuple 是否可清
```
这会把复制消费者停滞转化成 primary 上的 bloat 风险。
### 9.3 checkpoint I/O
slot state 文件很小，但 checkpoint 会扫描所有 slot。
写入时有：
```text
state.tmp write
fsync state.tmp
rename
fsync state
fsync slot dir
fsync pg_replslot
```
slot 数量通常不大，所以 CPU 扫描不是问题。
真正需要注意的是频繁推进 logical xmin / restart_lsn 时，前台路径会调用 `ReplicationSlotSave()`，带来同步 I/O。
### 9.4 相邻模块
slot 和这些模块形成硬边界：
| 模块 | 连接点 |
| --- | --- |
| xlog | `XLogSetReplicationSlotMinimumLSN()`、`KeepLogSeg()`、`GetWALAvailability()`、checkpoint / startup 调用。 |
| ProcArray / VACUUM | `ProcArraySetReplicationSlotXmin()` 发布 slot xmin horizon。 |
| walsender | acquire slot，接收 standby reply，更新 physical / logical 进度。 |
| logical decoding | `CreateInitDecodingContext()`、`DecodingContextFindStartpoint()`、`LogicalConfirmReceivedLocation()`。 |
| checkpointer / startup | checkpoint 保存 slot，startup restore slot，checkpoint invalidation obsolete slots。 |
| pgstat | logical slot stats entry 创建、acquire、drop 和 `pg_stat_replication_slots`。 |
slot 不是复制协议的附属字段。
它是 WAL 删除、VACUUM、logical decoding、checkpoint 和监控共同依赖的 shared state。
## 10. 观测与诊断入口
### 10.1 `pg_replication_slots`
`system_views.sql` 中：
```text
CREATE VIEW pg_replication_slots AS
  SELECT ... FROM pg_get_replication_slots() AS L
  LEFT JOIN pg_database D ON (L.datoid = D.oid)
```
`pg_get_replication_slots()` 在 `slotfuncs.c` 中：
```text
ReplicationSlotControlLock shared
for each in-use slot:
  SpinLockAcquire(&slot->mutex)
  slot_contents = *slot
  SpinLockRelease(&slot->mutex)
  组装一行视图输出
```
这说明视图是一瞬间的拷贝。
它不是全局一致事务快照，也不能证明之后 slot 没变化。
关键列解释：
| 列 | 主要来源 | 诊断含义 |
| --- | --- | --- |
| `slot_type` | `data.database` | `physical` 或 `logical`。 |
| `temporary` | `data.persistency == RS_TEMPORARY` | 是否不跨 session / restart 保留。 |
| `active` | `active_proc != INVALID_PROC_NUMBER` | 是否有 backend 持有。 |
| `active_pid` | `GetPGProcByNumber(active_proc)->pid` | 当前 owner pid。 |
| `xmin` | `data.xmin` | 数据 tuple 保留 horizon。 |
| `catalog_xmin` | `data.catalog_xmin` | catalog tuple 保留 horizon。 |
| `restart_lsn` | `data.restart_lsn` | WAL 保留起点。 |
| `confirmed_flush_lsn` | `data.confirmed_flush` | logical consumer 确认进度。 |
| `wal_status` | `GetWALAvailability(restart_lsn)` | WAL 是否仍可用。 |
| `safe_wal_size` | 当前 WAL 位置和保留上限估算 | 还有多少字节余量才可能丢。 |
| `inactive_since` | `inactive_since` | slot 变 inactive 的时间。 |
| `conflicting` | logical standby conflict 相关 invalidation | 是否因 recovery conflict 类型失效。 |
| `invalidation_reason` | `data.invalidated` | `wal_removed`、`rows_removed`、`wal_level_insufficient`、`idle_timeout` 等。 |
### 10.2 `wal_status` 的边界
`GetWALAvailability()` 可能返回：
```text
reserved:
  目标 WAL 在 max_wal_size 范围内。
extended:
  目标 WAL 因 slot 或 wal_keep_size 超出 max_wal_size 仍被保留。
unreserved:
  不再被保留，但还没删除；下一次 checkpoint 可能丢。
lost:
  需要的 WAL 已经删除。
NULL:
  restart_lsn 无效，slot 尚未真正保留 WAL 或已失效。
```
`unreserved` 是最容易误判的状态。
它不是“已经坏了”，而是“继续不推进会在 checkpoint 后坏”。
### 10.3 诊断主线
面对 WAL 目录增长，先按这条链路看：
```text
select slot_name, slot_type, temporary, active, active_pid,
       restart_lsn, confirmed_flush_lsn,
       wal_status, safe_wal_size,
       xmin, catalog_xmin,
       inactive_since, invalidation_reason
from pg_replication_slots
order by restart_lsn nulls last;
```
解释顺序：
```text
1. restart_lsn 最老的是谁？
2. slot 是否 active？
3. 如果 active，active_pid 对应哪个 walsender？
4. logical slot 的 confirmed_flush_lsn 是否明显领先于 restart_lsn？
5. wal_status 是 extended、unreserved 还是 lost？
6. invalidation_reason 是否已经说明保护承诺失败？
7. catalog_xmin / xmin 是否长期不前进，解释 VACUUM bloat？
```
不能只看 `active=false`。
inactive persistent slot 仍然会保护 WAL 和 xmin。
inactive 只是没有当前 owner，不等于不保留资源。
### 10.4 wait event 和日志
相关 wait event 包括：
```text
WAIT_EVENT_REPLICATION_SLOT_WRITE
WAIT_EVENT_REPLICATION_SLOT_SYNC
WAIT_EVENT_REPLICATION_SLOT_READ
WAIT_EVENT_REPLICATION_SLOT_RESTORE_SYNC
WAIT_EVENT_REPLICATION_SLOT_DROP
```
相关日志包括：
```text
acquired physical/logical replication slot
released physical/logical replication slot
invalidating obsolete replication slot
terminating process ... to release replication slot
```
这些入口能确认 slot 生命周期正在推进。
但它们不能直接告诉你业务消费者为什么停滞。
那需要结合 replication lag、网络、apply worker 或 logical consumer 自己的 durable offset。
## 11. 常见误区
1. 误以为 `physical` 就一定 persistent。physical/logical 是类型轴，temporary/persistent 是持久性轴。
2. 误以为 `active=false` 表示 slot 不再保留 WAL。persistent inactive slot 仍然保留 `restart_lsn` 之后需要的 WAL。
3. 误以为 `confirmed_flush_lsn` 是 WAL 删除边界。logical slot 的 WAL 删除边界是 `restart_lsn`，`confirmed_flush_lsn` 是 consumer 确认进度。
4. 误以为 `restart_lsn` 一定单调前进。`slot.h` 注释说明 physical streaming 初期可能因为按 segment 起点重收 WAL 而让 `restart_lsn` 看起来后退。
5. 误以为 `catalog_xmin` 可以像普通统计字段一样延迟保存。logical slot 必须先保存新的 catalog xmin，再放宽 `effective_catalog_xmin`。
6. 误以为 drop 只是清共享内存。安全 drop 的关键是先 rename `pg_replslot/<name>` 目录并 fsync。
7. 误以为 invalidated slot 等同删除。invalidated slot 仍存在，视图可见，但不再保护资源，acquire 会失败。
8. 误以为 `pg_replication_slots` 是因果链。它是瞬时状态拷贝，需要结合 checkpoint、WAL 目录、walsender 反馈和日志解释。
## 12. 课堂实验
### 12.1 physical temporary 与 persistent 对比
步骤：
```sql
select * from pg_create_physical_replication_slot('p_tmp', true, true);
select * from pg_create_physical_replication_slot('p_perm', true, false);
select slot_name, slot_type, temporary, active,
       restart_lsn, wal_status, inactive_since
from pg_replication_slots
where slot_name in ('p_tmp', 'p_perm');
```
观察点：
```text
p_tmp:
  temporary = true。
  当前 session 结束后应被 cleanup。
p_perm:
  temporary = false。
  release 后仍存在，并继续保留 restart_lsn。
```
源码断点：
```text
ReplicationSlotCreate
ReplicationSlotReserveWal
ReplicationSlotRelease
ReplicationSlotCleanup
```
需要画出的状态变化：
```text
in_use
active_proc
data.persistency
data.restart_lsn
last_saved_restart_lsn
```
### 12.2 logical slot 创建 consistent point
步骤：
```sql
select * from pg_create_logical_replication_slot('l_demo', 'test_decoding', false, false, false);
select slot_name, plugin, slot_type, database,
       restart_lsn, confirmed_flush_lsn,
       catalog_xmin, temporary, active
from pg_replication_slots
where slot_name = 'l_demo';
```
源码断点：
```text
ReplicationSlotCreate
CreateInitDecodingContext
ReplicationSlotReserveWal
DecodingContextFindStartpoint
ReplicationSlotPersist
SaveSlotToPath
```
观察点：
```text
创建开始时 persistency 是 RS_EPHEMERAL。
找到 consistent point 后 confirmed_flush 被设置。
最后 ReplicationSlotPersist() 才变成 RS_PERSISTENT。
```
### 12.3 手工观察 release 与 active_pid
准备一个使用 slot 的 walsender 或在 gdb 中让 backend 持有 slot。
另一会话查询：
```sql
select slot_name, active, active_pid, inactive_since
from pg_replication_slots
where slot_name = 'l_demo';
```
断开消费者后再查。
源码断点：
```text
ReplicationSlotAcquire
ReplicationSlotRelease
ConditionVariableBroadcast
```
要解释：
```text
active_pid 消失只说明 owner release。
persistent slot 的 restart_lsn / catalog_xmin 仍继续参与保留边界。
```
## 13. 讨论题
1. 为什么 `RS_EPHEMERAL` 只作为创建 persistent logical slot 的中间态，而不是暴露成普通用户类型？
2. 为什么 physical slot 可以延迟保存 standby flush 推进后的 `restart_lsn`，而 logical slot 推进 `catalog_xmin` 必须先 `ReplicationSlotSave()`？
3. 如果 `confirmed_flush_lsn` 已经很靠前，但 `restart_lsn` 仍很老，logical slot 为什么仍可能阻止 WAL 删除？
4. `active=false`、`inactive_since` 有值、`restart_lsn` 很老，这三者合起来说明什么？还不能说明什么？
5. 为什么 drop 要先 rename slot 目录到 `.tmp`，而不是先把 `in_use=false`？
6. `last_saved_restart_lsn` 为什么会影响 persistent slot 的 WAL 保留计算？
7. invalidated slot 为什么还要保留在视图里，而不是自动删除？
8. 如果 `wal_status='unreserved'`，你会先检查消费者、checkpoint，还是 `max_slot_wal_keep_size`？为什么？
## 14. 本节小结
Replication slot 的主线是：
```text
创建时建立 pg_replslot 磁盘状态和 shared memory slot
  -> acquire 后用 active_proc 建立单 owner
  -> physical 主要推进 restart_lsn 和 standby feedback xmin
  -> logical 用 restart_lsn 读 WAL，用 confirmed_flush 表示消费确认，用 catalog_xmin 保护历史 catalog
  -> release 时 persistent 保留、temporary / ephemeral 清理
  -> checkpoint 把 dirty persistent 状态写回 state 文件
  -> startup 只恢复 RS_PERSISTENT，并重新发布 WAL / xmin 保留边界
```
核心判断框架：
```text
physical vs logical:
  保护对象不同。
temporary vs persistent:
  crash 和 release 后保留策略不同。
data.* vs effective_*:
  持久承诺和当前清理边界不同。
restart_lsn vs confirmed_flush:
  WAL 重读起点和 logical 消费确认不同。
in_use vs active_proc vs invalidated:
  定义、持有和语义有效性不同。
```
本节可迁移的系统规律是：
```text
一个持久保护机制不能只保存“当前进度”；
它必须保存 crash 后重建保护边界所需的最保守状态，
并且在放宽全局清理边界前先让这个状态可靠落盘。
```
`pg_replication_slots` 能看到 slot 的当前共享状态和一部分 WAL 可用性推断。
它看不到消费者是否真正 durable 了自己的 offset，也看不到未来 checkpoint 是否马上删除未保留 WAL。
因此诊断时要把视图、WAL 目录、checkpoint 行为、walsender 反馈和 logical consumer 状态放到同一条时间线上解释。
