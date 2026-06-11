# PostgreSQL replication slot restart_lsn 与 WAL 保留压力
## 课程定位
前置知识：已经理解 WAL segment、checkpoint、restartpoint、WAL sender / receiver、replication slot 类型与生命周期，也知道 logical decoding 必须从 WAL 中重建事务和 catalog 语义。
本节唯一主问题：
```text
为什么一个 replication slot 的 restart_lsn 会阻止主库回收旧 WAL，
max_slot_wal_keep_size、checkpoint 和 WAL recycling 如何把保留需求变成磁盘压力？
```
核心矛盾：slot 必须让复制消费者断线后还能从安全位置继续读取 WAL；但 primary 的 `pg_wal` 不能无限增长。PostgreSQL 的选择不是让 slot 直接删除或保留文件，而是把每个 slot 的 `restart_lsn` 聚合成 xlog 模块的全局最低保留 LSN，再由 checkpoint / restartpoint 在删除旧 WAL 前解释这个边界，并在超过 `max_slot_wal_keep_size` 时把“继续保留”转成“slot 失效”。
学完后应能判断：`restart_lsn` 为什么不是消费者已经收到的位置；为什么 persistent slot 要同时考虑 `last_saved_restart_lsn`；为什么 slot 空闲也会继续保留 WAL；为什么 `max_slot_wal_keep_size` 不是提前限速器；以及 `pg_replication_slots.wal_status`、`safe_wal_size`、`pg_wal` 文件增长分别能说明什么。
本课基于本地 `/home/nail/postgres` 源码，分支 `master`，提交 `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。
## 1. 本节在总主线中的位置
上一节已经把 slot 拆成两层状态：
```text
shared memory ReplicationSlot:
  当前 backend 能扫描、acquire、release、更新的运行态。

pg_replslot/<name>/state:
  persistent slot crash 后恢复所需的磁盘状态。
```
这一节只追踪其中一个字段：
```text
ReplicationSlotPersistentData.restart_lsn
```
它的名字很容易误导。
`restart_lsn` 不是“当前连接从哪里开始发送”的瞬时参数。
它是这个 slot 仍可能需要读取的最老 WAL 位置。
只要这个位置所在的 segment 仍可能被 slot 读取，checkpoint 就不能把它当成普通旧 WAL 删除。
这条链路把复制状态传播到了存储层：
```text
slot->data.restart_lsn
  -> ReplicationSlotsComputeRequiredLSN()
  -> XLogSetReplicationSlotMinimumLSN()
  -> KeepLogSeg()
  -> InvalidateObsoleteReplicationSlots()
  -> RemoveOldXlogFiles()
```
这也是为什么一个看似“只是复制元数据”的 slot，最终能让 `$PGDATA/pg_wal` 增长。
`restart_lsn` 不直接占磁盘。
它改变的是“哪些 WAL segment 还不能被 remove 或 recycle”的边界。
## 2. 一句话运行模型
一句话运行模型：
```text
slot 创建或推进时写入 restart_lsn；
slot 层扫描所有有效 slot，取最老 restart_lsn 发布给 xlog；
checkpoint / restartpoint 删除 WAL 前调用 KeepLogSeg() 计算最早必须保留的 segment；
如果 max_slot_wal_keep_size 允许继续保留，旧 WAL 留在 pg_wal；
如果保留需求超过上限，checkpoint 先 invalidation 相关 slot，再删除或 recycle 不再受保护的 segment。
```
这里有三个时间尺度：
```text
WAL 生成:
  前台事务持续推进 insert/write LSN。

slot 进度:
  physical slot 依赖 standby flush feedback 推进 restart_lsn；
  logical slot 依赖 decoding、confirmed_flush 和 candidate_restart_lsn 推进 restart_lsn。

WAL 回收:
  checkpoint / restartpoint 周期性计算删除边界，真正 remove/recycle segment。
```
磁盘压力来自这三个时间尺度的错位。
WAL 可以持续生成。
slot 可能长时间不推进。
checkpoint 只能在边界允许时删除。
如果 `restart_lsn` 停在很早的位置，那么新的 WAL segment 会不断堆积在 `pg_wal` 中。
如果 `max_slot_wal_keep_size = -1`，slot 可以让这种堆积持续到磁盘耗尽。
如果设置了有限值，checkpoint 可能在超过上限后把 slot 标成 lost，而不是无限保护它。
## 3. 核心文件分工与阅读顺序
| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/slot.h` | `restart_lsn`、`last_saved_restart_lsn`、`dirty`、`inactive_since`、`invalidated` 的组合语义。 |
| 2 | `src/backend/replication/slot.c` | `ReplicationSlotReserveWal()`、`ReplicationSlotsComputeRequiredLSN()`、slot dirty/save/checkpoint、invalidation。 |
| 3 | `src/backend/replication/walsender.c` | physical streaming feedback 如何推进 physical slot 的 `restart_lsn`。 |
| 4 | `src/backend/replication/logical/logical.c` | logical slot 如何在确认消费后推进 `restart_lsn` 并保存。 |
| 5 | `src/backend/access/transam/xlog.c` | `KeepLogSeg()`、`GetWALAvailability()`、checkpoint 删除旧 WAL 的边界。 |
| 6 | `src/backend/access/transam/xlogrecovery.c` | recovery / restartpoint 语境下 WAL 文件读取和时间线边界。 |
| 7 | `src/backend/replication/slotfuncs.c` | `pg_get_replication_slots()` 如何计算 `wal_status` 和 `safe_wal_size`。 |
| 8 | `src/backend/catalog/system_views.sql` | `pg_replication_slots` 视图字段如何暴露 slot 诊断入口。 |
推荐阅读顺序不是按文件名排序。
先读 `slot.h`，确认哪些字段是持久语义，哪些只是运行态解释。
再读 `ReplicationSlotReserveWal()`，看初始 `restart_lsn` 如何被选中。
然后读 `ReplicationSlotsComputeRequiredLSN()`，看多个 slot 如何聚合成一个全局最低 LSN。
之后转到 `xlog.c` 的 `KeepLogSeg()` 和 checkpoint 删除路径。
最后再回到 `pg_get_replication_slots()`，解释监控值为什么只是对当前边界的解释，而不是 slot 层存储的真实状态。
## 4. `restart_lsn` 的语义：最老可能读取位置
`ReplicationSlotPersistentData` 中的 `restart_lsn` 会写入 `pg_replslot/<slot>/state`。
它代表：
```text
这个 slot 如果要继续工作，最早还可能需要从哪个 WAL LSN 开始读。
```
对 physical slot，它通常跟 standby 已经 flush 的位置相关。
standby 确认自己已经持久化到某个位置后，primary 端 physical slot 才能把 `restart_lsn` 推进。
对 logical slot，它更保守。
logical decoding 需要重建事务边界、subtransaction、catalog snapshot 和 reorder buffer 状态。
即使 consumer 已经确认到 `confirmed_flush_lsn`，slot 也不一定能立刻把 `restart_lsn` 推到同一个位置。
`confirmed_flush` 表示 consumer 已确认收到的逻辑输出进度。
`restart_lsn` 表示 decoder 重新启动时还需要读取的 WAL 起点。
这两个位置相差很大时，常见原因不是“slot 坏了”，而是 decoder 仍需要较早 WAL 中的信息来完整解释后续事务。
`logical.c` 的注释直接体现了这点：
```text
Start reading at the slot's restart_lsn
```
logical slot 的推进需要先提出 `candidate_restart_lsn`。
只有 consumer 确认的 LSN 达到 `candidate_restart_valid` 后，`LogicalConfirmReceivedLocation()` 才能把候选值应用到 `data.restart_lsn`。
这避免了 crash 后丢失 decoder 仍需要的 WAL。
所以本节的第一个不变量是：
```text
restart_lsn 是保守的读取下界；
不是连接当前发送位置；
不是已落盘 WAL 的最新位置；
也不是 confirmed_flush_lsn 的别名。
```
这也是诊断时最常见的误区。
看到 `restart_lsn` 很旧，不应先问“walsender 为什么不发送”。
应先问：
```text
这个 slot 的消费者是否还在推进？
physical standby 是否持续反馈 flush LSN？
logical consumer 是否持续确认并让 decoder 能安全推进 restart_lsn？
slot 是否已经 inactive？
```
## 5. 关键状态组合
本节只需要记住几组字段。
它们都在 `src/include/replication/slot.h`。
| 字段 | 所在结构 | 本节语义 |
| --- | --- | --- |
| `data.restart_lsn` | `ReplicationSlotPersistentData` | slot 当前发布的 WAL 读取下界。 |
| `last_saved_restart_lsn` | `ReplicationSlot` | 最近一次成功写入 `pg_replslot` 的 `restart_lsn`。 |
| `data.invalidated` | `ReplicationSlotPersistentData` | slot 是否已失效；失效后不再参与 WAL 保留。 |
| `data.persistency` | `ReplicationSlotPersistentData` | persistent slot crash 后仍存在；temporary slot 不需要 crash-safe WAL 保留。 |
| `dirty` / `just_dirtied` | `ReplicationSlot` | 内存状态是否需要写回磁盘。 |
| `active_proc` | `ReplicationSlot` | 当前是否有 backend 持有 slot。 |
| `inactive_since` | `ReplicationSlot` | persistent slot 变为空闲后的时间，idle timeout 和诊断使用。 |
这里最关键的是 `restart_lsn` 与 `last_saved_restart_lsn` 的组合。
如果 persistent slot 的内存 `restart_lsn` 已经向前推进，但还没有保存到磁盘，crash 后服务器只能恢复旧的 `restart_lsn`。
这意味着 crash recovery 后仍可能需要旧位置到新位置之间的 WAL。
因此 `ReplicationSlotsComputeRequiredLSN()` 对 persistent slot 不能只看内存里的 `data.restart_lsn`。
它必须在某些情况下选择更旧的 `last_saved_restart_lsn`。
这不是保守过度。
这是 crash-safety 要求：
```text
如果 slot 的放宽边界没有可靠写入 pg_replslot，
WAL 删除边界就不能假设这个放宽已经 durable。
```
源码还提醒不要假设 `restart_lsn` 永远单调前进。
physical streaming 初期可能从 segment 起点重新接收部分 WAL。
walreceiver 可能报告一个比之前更早的位置。
所以正确 mental model 不是“每个 slot 的 restart_lsn 只增不减”。
更准确的是：
```text
restart_lsn 是当前 slot 声明的读取下界；
checkpoint 删除 WAL 时要使用 crash 后仍能证明安全的下界。
```
## 6. 主流程一：创建 slot 时如何第一次保留 WAL
先看 `ReplicationSlotReserveWal()`。
它是物理和逻辑 slot 初始保留 WAL 的关键入口。
SQL 创建 physical slot 时，`slotfuncs.c` 的 `create_physical_replication_slot()` 在需要立即保留 WAL 时会调用它。
replication protocol 的 `CREATE_REPLICATION_SLOT ... RESERVE_WAL` 路径也会调用它。
logical slot 初始化时，如果调用者没有给定 `restart_lsn`，也会通过 logical 初始化路径调用它。
主链路是：
```text
ReplicationSlotCreate()
  -> slot 已经 in_use，并由当前 backend active 持有
  -> ReplicationSlotReserveWal()
     -> 选择初始 restart_lsn
     -> 写入 slot->data.restart_lsn
     -> ReplicationSlotsComputeRequiredLSN()
     -> 检查所需 segment 是否已被并发删除
  -> ReplicationSlotMarkDirty()
  -> persistent slot 立即 ReplicationSlotSave()
```
`ReplicationSlotReserveWal()` 选择初始 LSN 的逻辑很有信息量：
```text
physical slot:
  GetRedoRecPtr()

logical slot on standby:
  GetXLogReplayRecPtr(NULL)

logical slot on primary:
  GetXLogInsertRecPtr()
```
physical slot 从 redo pointer 起步，是因为 standby 做物理恢复时需要从 checkpoint redo 位置保证可恢复。
logical slot 在 primary 上需要从当前插入位置附近开始，并且后续还要写 standby snapshot。
logical slot 在 standby 上不能写 WAL，只能基于 replay 位置等待 primary 已经写出的 running-xacts 信息。
更重要的是锁边界。
`ReplicationSlotReserveWal()` 持有 `ReplicationSlotAllocationLock` exclusive。
注释解释它要和 checkpoint 的 slot flush / minimum LSN 计算串行化。
这条互锁保证：
```text
如果 slot 先 reserved WAL，
checkpoint 计算删除边界时会看到新的 restart_lsn。

如果 checkpoint 已经先完成相关计算，
后创建的 slot 选择的位置会不早于这个 checkpoint 的 redo pointer。
```
否则会出现最危险的竞态：
```text
slot 认为自己从某个旧 LSN 开始安全；
checkpoint 同时认为这个旧 segment 不再需要并删除它。
```
设置 `restart_lsn` 后，函数马上调用 `ReplicationSlotsComputeRequiredLSN()`。
注释写得很直接：
```text
prevent WAL removal as fast as possible
```
这说明 slot 保留 WAL 的动作不是等 checkpoint 顺手发现。
slot 一旦拿到初始 `restart_lsn`，就尽快把这个需求发布给 xlog 模块。
随后它还会把 `restart_lsn` 转成 segment number，并与 `XLogGetLastRemovedSegno()` 比较。
如果需要的 WAL 已经被并发删除，就 ERROR。
这条错误路径说明 slot 保留不是魔法。
它只能阻止未来删除。
已经被删除的 segment 不能靠创建 slot 恢复。
## 7. 主流程二：所有 slot 如何变成一个全局保留 LSN
`ReplicationSlotsComputeRequiredLSN()` 是本节最核心的 slot 层函数。
它做的事很小：
```text
扫描所有 in-use slot；
跳过 invalidated slot；
为 persistent slot 选择 crash-safe 的 restart_lsn；
取最小值；
调用 XLogSetReplicationSlotMinimumLSN(min_required)。
```
它不关心 `max_slot_wal_keep_size`。
源码注释说这个参数理论上相关，但 slot 模块不知道该跟什么比较。
这非常重要。
`ReplicationSlotsComputeRequiredLSN()` 输出的是“slot 需求”。
`max_slot_wal_keep_size` 是 xlog 删除边界解释这个需求时的“保留上限”。
这两个层次不能混在一起。
伪调用链是：
```text
ReplicationSlotsComputeRequiredLSN()
  -> LWLockAcquire(ReplicationSlotControlLock, LW_SHARED)
  -> for each ReplicationSlot
     -> if !in_use: continue
     -> read persistency, restart_lsn, invalidated, last_saved_restart_lsn under spinlock
     -> if invalidated: continue
     -> if persistent and last_saved_restart_lsn is older than restart_lsn:
          use last_saved_restart_lsn
     -> min_required = min(min_required, restart_lsn)
  -> XLogSetReplicationSlotMinimumLSN(min_required)
```
这里有两个边界。
第一，invalidated slot 不参与保留。
一旦 slot 因 WAL removed、horizon、wal_level 或 idle timeout 失效，它就不应该再继续阻止清理。
第二，persistent slot 的“内存最新值”不一定是删除边界。
如果 `data.restart_lsn` 已推进到 `0/5000000`，但 `last_saved_restart_lsn` 仍是 `0/3000000`，删除 `0/3000000` 之后的 WAL 可能让 crash 后恢复出的 slot 无法继续。
因此删除边界必须使用旧值。
这个设计把两个目标绑在一起：
```text
运行时尽量及时放宽 WAL 保留；
crash 后不能发现 slot 状态回退，而 WAL 已经被删。
```
所以 `last_saved_restart_lsn` 不是统计字段。
它是 WAL 删除正确性的一部分。
如果一次 checkpoint 成功保存了 slot state，`SaveSlotToPath()` 会更新 `last_saved_restart_lsn`。
随后 `CheckPointReplicationSlots()` 如果发现它发生变化，会再次调用 `ReplicationSlotsComputeRequiredLSN()`。
这让全局 minimum LSN 从“旧的 durable 边界”推进到“新的 durable 边界”。
## 8. 主流程三：slot dirty/save 与 checkpoint 的关系
slot 状态不是每次变化都同步写盘。
`ReplicationSlotMarkDirty()` 只是设置：
```text
just_dirtied = true
dirty = true
```
它的注释强调：
```text
actual flush to disk can be delayed for a long time
```
如果调用者需要 correctness 立即 durable，就必须显式调用 `ReplicationSlotSave()`。
创建 persistent physical slot 并保留 WAL 后会立即 save。
logical slot 更新 catalog xmin 或 restart_lsn 后，`LogicalConfirmReceivedLocation()` 也会 mark dirty 并 save。
但 physical slot 收到 standby flush feedback 后通常只 mark dirty、重新计算 RequiredLSN，不立即 save。
`walsender.c` 的注释说明，这样最坏只是 crash 后更保守地保留 WAL，浪费一些空间，而不是破坏 correctness。
checkpoint 负责批量把 dirty persistent slot 写回磁盘。
`CheckPointReplicationSlots(bool is_shutdown)` 的主链路是：
```text
LWLockAcquire(ReplicationSlotAllocationLock, LW_SHARED)
for each in-use slot:
  if shutdown checkpoint and logical confirmed_flush advanced:
    mark dirty
  if last_saved_restart_lsn != data.restart_lsn:
    remember that recomputation may be needed
  SaveSlotToPath(slot, pg_replslot/<name>, LOG)
LWLockRelease(ReplicationSlotAllocationLock)
if last_saved_restart_lsn updated:
  ReplicationSlotsComputeRequiredLSN()
```
这里再次看到 `ReplicationSlotAllocationLock`。
checkpoint 不拿 `ReplicationSlotControlLock` 来长时间扫和写文件。
它拿更强的 allocation lock，防止 slot create/drop 改变 `in_use`，同时与 `ReplicationSlotReserveWal()` 串行化。
`SaveSlotToPath()` 的磁盘写入顺序是：
```text
检查 dirty；
拿 slot->io_in_progress_lock；
写 pg_replslot/<slot>/state.tmp；
fsync state.tmp；
rename state.tmp -> state；
fsync state；
fsync slot 目录；
fsync pg_replslot 目录；
清 dirty；
更新 last_saved_confirmed_flush 和 last_saved_restart_lsn。
```
这不是普通配置文件写入。
它是 slot crash-safety 的边界。
如果写入中途失败，旧 `state` 仍然存在。
如果 rename 和 fsync 成功，`last_saved_restart_lsn` 才能代表 durable 状态。
这解释了为什么 checkpoint 不只是“让监控值更新”。
checkpoint 可以改变后续 WAL 删除边界。
当 persistent slot 的新 `restart_lsn` 可靠保存后，旧 WAL 才真正可以被删除。
## 9. 主流程四：checkpoint 如何把 minimum LSN 变成删除边界
slot 层只发布一个最低 LSN。
真正删除 WAL 的代码在 `src/backend/access/transam/xlog.c`。
普通 checkpoint 写完 checkpoint record、刷盘并做 post-checkpoint cleanup 后，会进入旧 WAL 删除路径。
核心片段可以压缩成：
```text
XLByteToSeg(RedoRecPtr, _logSegNo, wal_segment_size)
KeepLogSeg(recptr, &_logSegNo)
InvalidateObsoleteReplicationSlots(RS_INVAL_WAL_REMOVED | RS_INVAL_IDLE_TIMEOUT,
                                   _logSegNo, InvalidOid, InvalidTransactionId)
if any slot invalidated:
  recompute _logSegNo from RedoRecPtr
  KeepLogSeg(recptr, &_logSegNo)
_logSegNo--
RemoveOldXlogFiles(_logSegNo, RedoRecPtr, recptr, timeline)
```
`RedoRecPtr` 给出 checkpoint 自己需要保留的最低 WAL。
`KeepLogSeg()` 会把 `_logSegNo` 往更早的位置退。
它会综合：
```text
replication slot minimum LSN
max_slot_wal_keep_size
WAL summarization 的 oldest unsummarized LSN
wal_keep_size
```
对本节来说最重要的是 replication slot 部分。
`KeepLogSeg()` 先读：
```text
keep = XLogGetReplicationSlotMinimumLSN()
```
如果这个 LSN 有效且早于当前 `recptr`，就把它转成 segment number。
这就是 slot 阻止 WAL 删除的直接点。
随后它处理 `max_slot_wal_keep_size`：
```text
if max_slot_wal_keep_size_mb >= 0 and not IsBinaryUpgrade:
  slot_keep_segs = ConvertToXSegs(max_slot_wal_keep_size_mb)
  if currSegNo - segno > slot_keep_segs:
    segno = currSegNo - slot_keep_segs
```
换句话说，slot 想保留到很早的位置。
但如果它超过配置上限，`KeepLogSeg()` 会把删除边界向前推到“最多保留这么多 slot WAL”的位置。
这时相关 slot 可能已经不再被完整保护。
所以 checkpoint 接着调用 `InvalidateObsoleteReplicationSlots()`。
如果某个 slot 的 `restart_lsn` 早于新的 oldest LSN，它会被 invalidated。
然后 checkpoint 重新计算删除边界。
最后 `_logSegNo--`。
`KeepLogSeg()` 返回的是最早还要保留的 segment。
`RemoveOldXlogFiles()` 接收的是“可以删除到哪个 segment 为止”。
所以要减一，表示只删除更老的文件。
这就是 WAL removal 的真实边界：
```text
first segment to keep = KeepLogSeg() result
last segment removable = first segment to keep - 1
```
`RemoveOldXlogFiles()` 会扫描 `pg_wal`。
对文件名早于或等于删除边界的 WAL segment，它先检查 archive 状态。
如果 `XLogArchiveCheckDone()` 允许，再调用 `RemoveXlogFile()`。
`RemoveXlogFile()` 可能删除，也可能 recycle 给未来 WAL 使用。
所以磁盘现象可能是：
```text
文件数量下降；
文件名跳变；
旧文件被回收成未来 segment；
总大小短期不明显下降。
```
这些现象都不改变核心边界：
只要 segment 被 slot minimum LSN 保护，它就不能被当成可删除对象。
## 10. restartpoint 与 standby 上的相同压力
standby 上没有普通 checkpoint 删除边界，而是 restartpoint。
`xlog.c` 的 restartpoint 删除路径与 checkpoint 类似。
它先取 redo pointer，再取接收或重放到的较新位置：
```text
receivePtr = GetWalRcvFlushRecPtr(NULL, NULL)
replayPtr = GetXLogReplayRecPtr(&replayTLI)
endptr = max(receivePtr, replayPtr)
KeepLogSeg(endptr, &_logSegNo)
InvalidateObsoleteReplicationSlots(...)
RemoveOldXlogFiles(...)
```
这说明 slot WAL 保留压力不只发生在 primary。
如果 standby 上有 slot，或者有 synced logical failover slot，restartpoint 也必须考虑它们。
但本节的唯一主问题仍然是 `restart_lsn` 如何阻止 WAL 回收。
standby 只是同一个模型在 recovery 语境下的实例。
`xlogrecovery.c` 负责恢复读取和时间线跟随。
旧 WAL 文件能不能删，最终仍要回到 `xlog.c` 的 restartpoint 删除边界。
promotion、timeline 切换和历史文件会让文件名判断更复杂。
但 segment retention 的内核问题不变：
```text
本地仍可能需要读的 WAL 不能删除；
slot 把这种“仍可能需要”发布成 restart_lsn。
```
## 11. `max_slot_wal_keep_size`：不是限速器，而是失效边界
`max_slot_wal_keep_size` 最容易被误解。
它不会阻止事务继续产生 WAL。
它不会在 slot lag 达到阈值时让前台写入等待。
它也不会让 `restart_lsn` 自动推进。
它只在 WAL 回收边界计算中起作用。
更准确地说：
```text
max_slot_wal_keep_size = -1:
  slot 需求不设上限，旧 WAL 可能无限保留。

max_slot_wal_keep_size >= 0:
  checkpoint / restartpoint 计算删除边界时，
  slot 最多只能要求保留当前 WAL 位置向后这么多 WAL。
```
超过这个上限时，系统不是“压缩 WAL”。
它只能承认继续保留不可接受，于是让相关 slot 失去继续读取所需 WAL。
`InvalidateObsoleteReplicationSlots()` 会把这种情况记录为 `RS_INVAL_WAL_REMOVED`。
日志细节中会指出 slot 的 `restart_lsn` 超过限制多少字节，并提示可能需要增加 `max_slot_wal_keep_size`。
这就是主问题中的“把保留需求变成磁盘压力”：
```text
没有上限:
  保留需求表现为 pg_wal 持续增长。

有上限:
  保留需求先表现为 pg_wal 增长；
  到 checkpoint 删除边界超过 slot restart_lsn 后，
  表现为 slot invalidation / wal_status lost。
```
所以 `max_slot_wal_keep_size` 是损失边界，不是背压机制。
如果业务要求 slot 永远可追赶，就不能只靠这个参数。
必须让消费者持续推进，或设计监控在达到边界前报警。
## 12. `wal_status` 与 `safe_wal_size` 如何解释
`pg_replication_slots` 是视图。
它来自 `system_views.sql`：
```text
CREATE VIEW pg_replication_slots AS
  SELECT ... L.restart_lsn, L.wal_status, L.safe_wal_size, ...
  FROM pg_get_replication_slots() AS L
```
真实计算在 `slotfuncs.c` 的 `pg_get_replication_slots()`。
函数先复制 slot 内容。
如果 slot 没有 invalidated，就调用：
```text
GetWALAvailability(slot_contents.data.restart_lsn)
```
`GetWALAvailability()` 返回五类状态。
```text
reserved:
  restart_lsn 对应 segment 仍可用，并且落在 max_wal_size 常规保留范围内。

extended:
  segment 仍可用，但保留已经超出 max_wal_size，主要因为 slot 等额外需求。

unreserved:
  segment 目前还在 pg_wal 中，但已不再受 slot 保留边界保护，下一次 checkpoint 可能删除。

lost:
  segment 已经删除，slot 无法继续。

NULL:
  restart_lsn 无效，slot 没有保留 WAL 或已被清空。
```
这里要注意两个事实。
第一，`wal_status` 不是 `ReplicationSlot` 的字段。
它是视图查询时根据当前 WAL 文件边界临时计算出来的解释。
第二，`extended` 不等于错误。
它只是说明 `pg_wal` 为 slot 保留了超过 `max_wal_size` 常规范围的 WAL。
`max_wal_size` 本来就不是硬上限。
`safe_wal_size` 也不是绝对保证。
`slotfuncs.c` 只有在 `max_slot_wal_keep_size` 已配置且 slot 尚未 lost 时才计算它。
它大致按：
```text
targetSeg = segment(restart_lsn)
failSeg = targetSeg + max(slotKeepSegs, wal_keep_size_segs) + 1
safe_wal_size = failLSN - current_write_lsn
```
所以 `safe_wal_size` 表达的是：
```text
在当前写入位置下，距离这个 slot 因保留上限失去 WAL 还有多少字节空间。
```
它依赖当前 `pg_current_wal_lsn()`、segment size、`wal_keep_size` 和 `max_slot_wal_keep_size`。
它不是一个被后台线程实时维护的倒计时。
如果 WAL 生成速率突增，它会很快变小。
如果 checkpoint 尚未运行，`wal_status` 可能还停在 `unreserved` 而不是 `lost`。
如果 walsender 在查询期间刚推进 `restart_lsn`，视图还会做一次保守修正，把某些 race 下的 `removed` 解释成 `unreserved`。
因此诊断时不要把 `wal_status` 当唯一事实。
它要和 `restart_lsn`、`pg_wal_lsn_diff()`、`active`、`inactive_since`、`pg_wal` 文件数量一起看。
## 13. inactive slot 为什么仍然增长磁盘
persistent slot release 后不会删除。
`ReplicationSlotRelease()` 对 persistent slot 做的是：
```text
active_proc = INVALID_PROC_NUMBER
inactive_since = now
ConditionVariableBroadcast(active_cv)
```
它不会清空 `restart_lsn`。
它不会把 slot 从 `in_use` 数组移除。
它也不会从 `pg_replslot` 删除 state。
这正是 persistent slot 的承诺：
消费者断开后，下次还能从原位置继续。
因此 inactive slot 仍会被 `ReplicationSlotsComputeRequiredLSN()` 扫到。
只要它：
```text
in_use = true
invalidated = RS_INVAL_NONE
restart_lsn 有效
```
它就会继续参与全局 minimum LSN。
没有 active walsender 意味着没有人推进它。
physical slot 没有 standby flush feedback，`restart_lsn` 停住。
logical slot 没有 consumer 确认，candidate restart 不能应用。
checkpoint 每次都看到同一个旧边界。
新 WAL 持续产生。
旧 WAL 不能删除。
这就是 inactive slot 引起 `pg_wal` 增长的完整故事。
`inactive_since` 只是观测和 idle timeout 的辅助状态。
它不会自动释放 WAL。
只有配置 `idle_replication_slot_timeout` 后，checkpoint / restartpoint 的 `InvalidateObsoleteReplicationSlots()` 才会把 idle timeout 作为可能原因之一。
即便如此，失效也发生在 checkpoint 类路径中。
它不是 slot release 时立即执行的删除。
所以监控 inactive slot 时，最重要的问题不是“它 inactive 多久”本身。
而是：
```text
它的 restart_lsn 距离 current WAL LSN 多远？
它的 wal_status 是否 extended 或 unreserved？
safe_wal_size 是否正在逼近 0？
业务是否还需要这个 slot？
```
## 14. slot invalidation 的边界
`InvalidateObsoleteReplicationSlots()` 处理多类失效原因。
本节关注 WAL removed 和 idle timeout。
函数接收一个 `oldestSegno`。
它转成 `oldestLSN` 后，对每个 slot 判断：
```text
RS_INVAL_WAL_REMOVED:
  restart_lsn 有效，并且 restart_lsn < oldestLSN。

RS_INVAL_IDLE_TIMEOUT:
  idle timeout 已配置；
  restart_lsn 有效；
  slot inactive；
  inactive 时长超过阈值；
  recovery 中 synced slot 例外。
```
如果 slot 当前不 active，invalidation 可以直接完成：
```text
把 MyReplicationSlot 临时指向这个 slot；
active_proc = MyProcNumber；
data.invalidated = cause；
如果 cause 是 WAL_REMOVED，清空 data.restart_lsn 和 last_saved_restart_lsn；
ReplicationSlotMarkDirty();
ReplicationSlotSave();
ReplicationSlotRelease();
```
如果 slot 当前 active，checkpointer 不能直接修改它。
它会记录持有者 pid，释放 control lock，向持有者发 `SIGTERM` 或 recovery conflict，然后等 `active_cv`。
等 slot release 后再重试。
这里有一个重要 race 处理。
等待期间 slot 可能追上了。
源码注释明确说，如果重新检查时 `restart_lsn` 或 xmin 已经足够推进，就没有理由 invalidation。
这是安全的，因为相关 WAL 或 tuple 还没有被删除。
所以 invalidation 不是一次读取视图后的机械判决。
它是在删除前、持锁和重试边界内做的最后确认。
失效后的观测结果通常是：
```text
pg_replication_slots.invalidation_reason = 'wal_removed' 或其它原因；
wal_status = 'lost'；
restart_lsn 可能为 NULL；
active consumer 被终止或无法继续使用该 slot。
```
对 logical slot，`conflicting` 字段还会在 horizon 或 wal_level 相关原因下反映 recovery conflict。
对 physical slot，`conflicting` 为 NULL。
## 15. WAL recycling/removal 与磁盘压力
PostgreSQL 删除旧 WAL 时不是简单 `unlink`。
`RemoveOldXlogFiles()` 会遍历 `pg_wal`。
对达到删除边界的 segment，它先检查归档状态。
如果可以处理，`RemoveXlogFile()` 会根据未来需要选择 remove 或 recycle。
recycling 的意思是：
```text
旧 segment 文件可能被重命名为未来 segment 文件；
文件仍占磁盘，但可被后续 WAL 写入复用。
```
这解释了一个常见现场：
checkpoint 后 `pg_wal` 文件名变化，但目录大小没有立刻下降。
这不一定表示 slot 还在保留同一批 WAL。
可能是 WAL 文件被 recycle 作为预分配空间。
反过来，如果 slot 阻止回收，旧文件既不能删除也不能 recycle 到未来位置。
它们必须保持原来的文件名和内容，因为消费者可能按 `restart_lsn` 重新打开这些 segment。
磁盘压力的传播链是：
```text
consumer lag / inactive slot
  -> restart_lsn 不推进
  -> XLogGetReplicationSlotMinimumLSN() 停在旧位置
  -> KeepLogSeg() 把删除边界退到旧 segment
  -> RemoveOldXlogFiles() 找不到足够可删除或可 recycle 文件
  -> pg_wal 增长
  -> checkpoint 之后仍不能释放空间
  -> 达到 max_slot_wal_keep_size 后 slot 可能 lost
```
这条链里，checkpoint 是“压力显形”的时刻。
WAL 生成期间，磁盘只是增长。
checkpoint 才会决定哪些旧 segment 可以处理，哪些必须继续留下，哪些 slot 已经超过上限。
所以一个系统在 checkpoint 前后出现 `wal_status` 从 `extended` 到 `unreserved` 或 `lost` 的变化，是符合源码模型的。
## 16. 诊断入口：从视图回到源码
第一组查询看 slot 自身。
```sql
SELECT slot_name,
       slot_type,
       active,
       active_pid,
       inactive_since,
       restart_lsn,
       confirmed_flush_lsn,
       wal_status,
       safe_wal_size,
       invalidation_reason
FROM pg_replication_slots
ORDER BY slot_name;
```
解释顺序应该是：
```text
active / active_pid:
  是否有 backend 持有 slot。

inactive_since:
  persistent slot 从何时开始没有持有者。

restart_lsn:
  WAL 保留下界。

confirmed_flush_lsn:
  logical consumer 确认进度，不等于 WAL 保留下界。

wal_status:
  当前 restart_lsn 对应 WAL segment 的可用性解释。

safe_wal_size:
  在配置了 max_slot_wal_keep_size 时，距离失去 WAL 的字节估算。

invalidation_reason:
  slot 是否已经退出保护链路。
```
第二组查询把 slot lag 转成字节。
```sql
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained_by_restart_lsn,
       wal_status,
       pg_size_pretty(safe_wal_size) AS safe_wal_size
FROM pg_replication_slots
WHERE restart_lsn IS NOT NULL
ORDER BY pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) DESC;
```
这个结果不是 `pg_wal` 目录大小。
它只是当前 WAL insert LSN 到 slot `restart_lsn` 的逻辑距离。
实际目录大小还受 segment size、archive、recycle、preallocation、wal_keep_size、summarization、checkpoint 时机影响。
第三组查询看 `pg_wal` 文件。
```sql
SELECT count(*) AS wal_files,
       pg_size_pretty(sum(size)) AS total_size,
       min(name) AS oldest_file,
       max(name) AS newest_file
FROM pg_ls_waldir()
WHERE name ~ '^[0-9A-F]{24}$';
```
如果 `restart_lsn` 很旧，而 `oldest_file` 对应的 segment 也很旧，说明 slot 可能真的在保留旧 WAL。
如果 `wal_status = unreserved`，说明文件可能还在，但已经不再被 slot 边界保护。
下一次 checkpoint 可能改变目录。
第四组入口是操作系统：
```bash
du -sh "$PGDATA/pg_wal"
ls -lh "$PGDATA/pg_wal" | head
```
操作系统只能看到空间。
它不能告诉你空间是 slot、archiver、checkpoint 间隔、wal_keep_size 还是预分配导致的。
所以必须回到 `pg_replication_slots` 和源码边界解释。
一个可靠诊断叙述应该长这样：
```text
看到 pg_wal 增长
  -> 找到 restart_lsn 最旧的 slot
  -> 确认它 active/inactive 和 inactive_since
  -> 看 wal_status 是否 extended/unreserved/lost
  -> 用 safe_wal_size 判断 max_slot_wal_keep_size 边界
  -> 结合 checkpoint 时机判断何时可能删除或 invalidation
```
## 17. 可复现实验：inactive physical slot 保留 WAL
不要在生产环境直接做这个实验。
准备一个测试实例。
建议设置有限的上限，便于观察：
```conf
max_replication_slots = 10
wal_level = replica
max_slot_wal_keep_size = '256MB'
checkpoint_timeout = '1min'
```
创建一个立即保留 WAL 的 physical slot：
```sql
SELECT * FROM pg_create_physical_replication_slot('hold_wal', true);
```
此时查询：
```sql
SELECT slot_name, active, restart_lsn, wal_status, safe_wal_size
FROM pg_replication_slots
WHERE slot_name = 'hold_wal';
```
生成 WAL：
```sql
CREATE TABLE wal_retention_probe(id bigint, payload text);
INSERT INTO wal_retention_probe
SELECT g, repeat(md5(g::text), 20)
FROM generate_series(1, 1000000) AS g;
CHECKPOINT;
```
再次查询 slot：
```sql
SELECT slot_name,
       active,
       inactive_since,
       restart_lsn,
       wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS lag,
       pg_size_pretty(safe_wal_size) AS safe
FROM pg_replication_slots
WHERE slot_name = 'hold_wal';
```
预期现象：
```text
active = false；
restart_lsn 不推进；
lag 增大；
wal_status 可能从 reserved 到 extended，再到 unreserved 或 lost；
safe_wal_size 逐渐变小。
```
如果 checkpoint 后 slot lost，再尝试使用这个 slot 的复制流会失败。
清理实验：
```sql
SELECT pg_drop_replication_slot('hold_wal');
CHECKPOINT;
```
注意，drop slot 后目录大小可能不会马上按字节下降。
旧 segment 可能被 recycle。
这时应观察后续 WAL 文件名和总数，而不是只看一次 `du`。
## 18. 源码跟读实验
第一条跟读链：
```text
slotfuncs.c:create_physical_replication_slot()
  -> ReplicationSlotCreate()
  -> ReplicationSlotReserveWal()
  -> ReplicationSlotsComputeRequiredLSN()
```
断点建议：
```text
ReplicationSlotReserveWal
ReplicationSlotsComputeRequiredLSN
XLogSetReplicationSlotMinimumLSN
```
观察点：
```text
MyReplicationSlot->data.restart_lsn
MyReplicationSlot->last_saved_restart_lsn
XLogGetReplicationSlotMinimumLSN()
```
第二条跟读链：
```text
CreateCheckPoint()
  -> KeepLogSeg()
  -> InvalidateObsoleteReplicationSlots()
  -> RemoveOldXlogFiles()
```
断点建议：
```text
KeepLogSeg
InvalidateObsoleteReplicationSlots
InvalidatePossiblyObsoleteSlot
RemoveOldXlogFiles
```
观察点：
```text
recptr
_logSegNo
keep = XLogGetReplicationSlotMinimumLSN()
max_slot_wal_keep_size_mb
slot->data.invalidated
```
第三条跟读链看 logical slot 推进：
```text
LogicalConfirmReceivedLocation()
  -> apply candidate_restart_lsn
  -> ReplicationSlotMarkDirty()
  -> ReplicationSlotSave()
  -> ReplicationSlotsComputeRequiredLSN()
```
观察点：
```text
data.confirmed_flush
candidate_restart_valid
candidate_restart_lsn
data.restart_lsn
last_saved_restart_lsn
```
这条链能帮助区分：
```text
consumer 确认了输出；
decoder 是否已经能安全放宽 WAL 读取起点。
```
## 19. 常见误区
误区一：把 `restart_lsn` 当成 standby 当前 replay 位置。
physical slot 的 `restart_lsn` 更接近 primary 端根据反馈保存的 WAL 保留下界。
standby 的 replay/apply 位置是另一组反馈字段。
误区二：认为 `confirmed_flush_lsn` 推进就一定释放 WAL。
logical slot 释放 WAL 依赖 `restart_lsn` 推进。
`confirmed_flush_lsn` 是必要输入，但不是删除边界。
误区三：认为 `max_wal_size` 可以限制 slot 导致的 `pg_wal` 增长。
`max_wal_size` 影响 checkpoint 触发和常规保留范围。
slot 可以让 WAL 保留超过它，因此 `wal_status` 才有 `extended`。
误区四：认为 `max_slot_wal_keep_size` 会保护磁盘且不影响复制。
它保护的是磁盘上限，代价是 slot 可能 lost。
它不是消费者追赶能力的保证。
误区五：认为 inactive slot 不占资源。
inactive 只说明没有 backend 持有。
只要 persistent slot 有有效 `restart_lsn` 且未 invalidated，它仍然保留 WAL。
误区六：看到 `pg_wal` 没立刻变小就判断 drop slot 无效。
checkpoint、archive、recycling、preallocation 都会影响目录大小。
要看删除边界和文件名变化，而不是只看一次容量。
## 20. 讨论题
1. 为什么 `ReplicationSlotsComputeRequiredLSN()` 不直接应用 `max_slot_wal_keep_size`？
2. persistent slot 的 `data.restart_lsn` 已经向前推进时，为什么还可能用 `last_saved_restart_lsn` 计算删除边界？
3. physical slot 为什么可以在收到 feedback 后只 mark dirty，而不立即 `ReplicationSlotSave()`？
4. `wal_status = extended` 和 `wal_status = unreserved` 的运维含义有什么不同？
5. active slot 超过 WAL 保留边界时，checkpointer 为什么要 signal 持有者而不是直接改字段？
6. 为什么 drop slot 后 `pg_wal` 目录大小可能不会立刻下降？
7. 对 logical slot，为什么 `confirmed_flush_lsn` 不能单独作为 WAL 删除下界？
8. 如果 `safe_wal_size` 还有 10GB，为什么仍不能保证一天内安全？
## 21. 本节小结
本节的核心链路是：
```text
restart_lsn
  -> global minimum slot LSN
  -> KeepLogSeg()
  -> checkpoint / restartpoint old WAL removal boundary
  -> RemoveOldXlogFiles() remove/recycle
```
`restart_lsn` 的本质是 slot 的最老 WAL 读取下界。
它不是 consumer 当前收到的位置，也不是 logical `confirmed_flush_lsn`。
persistent slot 的删除边界还要考虑 `last_saved_restart_lsn`，因为 crash 后只能相信已经写入 `pg_replslot` 的状态。
`ReplicationSlotReserveWal()` 用 allocation lock 与 checkpoint 串行化，确保新 slot 选择的起点不会被并发删除。
`ReplicationSlotsComputeRequiredLSN()` 扫描有效 slot，把最老需求发布给 xlog 模块。
`CheckPointReplicationSlots()` 和 `SaveSlotToPath()` 决定 slot 状态何时 durable，从而决定 WAL 删除边界何时可以真正放宽。
`max_slot_wal_keep_size` 不会减慢 WAL 生成。
它在 `KeepLogSeg()` 中限制 slot 最多能要求保留多少 WAL。
超过这个边界后，checkpoint 会让 slot invalidated，而不是继续无限保留。
inactive persistent slot 仍然保留 WAL。
它只是没有 active owner，`restart_lsn` 仍参与 minimum LSN 计算。
诊断时应把 `pg_replication_slots` 与 `pg_wal` 一起看：
```text
restart_lsn:
  谁在保留 WAL。

wal_status:
  当前 segment 可用性解释。

safe_wal_size:
  有上限时距离 lost 的估算。

inactive_since:
  为什么没人推进。

pg_ls_waldir / du:
  磁盘现象。
```
可迁移的系统规律是：
```text
一个持久化进度状态只有在 crash 后仍能证明安全时，
才能用来放宽底层资源回收边界；
如果消费者进度长期不推进，上层 correctness 承诺会自然传播成底层磁盘保留压力。
```
哪些判断仍然依赖环境？
WAL 增长速度依赖 workload。
目录大小变化依赖 checkpoint、archive、recycling 和 segment size。
`safe_wal_size` 依赖当前写入速率和配置，不是时间保证。
slot 是否应该 drop、advance、等待消费者追赶，取决于复制拓扑和业务恢复目标。
