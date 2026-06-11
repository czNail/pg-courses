# PostgreSQL xmin horizon 与旧 tuple cleanup 边界

## 课程定位

前置知识：已经理解 MVCC snapshot 字段、tuple header 中的 `xmin` / `xmax`、CLOG/pg_xact 事务结果、SubXID overflow fallback，以及上一节的 active / registered snapshot 生命周期。

本节唯一主问题：

```text
为什么一个旧 tuple version 对当前查询已经不可见，PostgreSQL 仍然不能立刻把它 cleanup 掉？
```

本节围绕的核心矛盾：

```text
单个查询希望旧版本尽快消失，heap page 尽快腾出空间，index cleanup 尽快减少 bloat；
但系统必须保证所有仍可能观察旧世界的对象都不被破坏，包括其他 backend 的 snapshot、cursor 固定的 `MyProc->xmin`、prepared transaction、replication slot、logical decoding、standby query 和 hot standby feedback。
```

一句话运行模型：

```text
tuple visibility 回答“这个 tuple 对当前 snapshot 是否可见”；
cleanup horizon 回答“这个 tuple 是否已经不可能被任何仍受保护的观察者需要”；
PostgreSQL 通过 `ComputeXidHorizons()` 把 ProcArray、slot、prepared xact、recovery 和 relation kind 压缩成 relation-aware horizon，再由 `vacuum_get_cutoffs()`、`HeapTupleSatisfiesVacuum()` 和 `GlobalVisTestIsRemovableXid()` 决定能否删除或 pruning。
```

学完后应能独立判断：

- 为什么 `DELETE` 已提交不等于旧版本马上 `HEAPTUPLE_DEAD`。
- 为什么 `HeapTupleSatisfiesMVCC()` 和 `HeapTupleSatisfiesVacuum()` 的问题不同。
- 为什么 `OldestXmin` 是保守下界，不是当前查询 snapshot 的 `xmin`。
- 为什么普通表、catalog、shared relation、temp relation 使用不同 horizon。
- 为什么 replication slot 的 `xmin` 和 `catalog_xmin` 会阻止 cleanup。
- 为什么 prepared transaction 没有 backend pid，仍会像 running transaction 一样进入 ProcArray。
- 为什么 hot standby feedback 既能减少 standby conflict，也能在 primary 制造 bloat。
- 为什么 `GlobalVisState` 使用 `maybe_needed` / `definitely_needed` 两个近似边界，而不是每次都精确扫 ProcArray。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

本节不展开 index vacuum 的全部细节、freezing 的完整 MultiXact 规则、logical decoding reorder buffer。这些模块只在它们参与 cleanup horizon 时出现。

## 1. 本节在总主线中的位置

上一节把 snapshot 的本地生命周期讲完了。一个 backend 可以因为 active snapshot 或 registered snapshot 暴露 `MyProc->xmin`。

这个 `xmin` 会进入全局可见性保守边界。本节把范围扩大到整个实例。

问题从“我的 backend 还持有哪些 snapshot”变成“整个系统是否还有任何观察者需要旧版本”。这一步很容易被误解。

当前查询看不到一个旧 tuple version，不代表所有观察者都看不到它。一个更早开始的事务可能还在使用旧 snapshot。

一个 cursor 可能已经 idle，但 snapshot 仍 registered。一个 prepared transaction 可能没有普通 backend pid，却仍被 ProcArray 当成 running。

一个 logical replication slot 可能需要 catalog tuple 来继续 decoding。一个 physical standby 可能通过 hot standby feedback 要求 primary 保留旧版本，避免 standby query 被 recovery conflict cancel。

所以 cleanup 不是 local visibility 的直接后果。cleanup 是全局保守判断。

本节跟踪一条主链：

```text
DELETE/UPDATE 产生旧 tuple version
  -> deleter commit
  -> 当前查询不再可见旧版本
  -> VACUUM 获取 relation-aware OldestXmin
  -> HeapTupleSatisfiesVacuum() 把旧版本判为 RECENTLY_DEAD 或 DEAD
  -> pruning / vacuum 根据 OldestXmin 与 GlobalVisState 决定是否真正移除
```

这条链把 tuple visibility、ProcArray、replication、2PC 和 VACUUM 连接起来。本节的核心不是“VACUUM 做什么步骤”。

核心是：

```text
PostgreSQL 如何证明一个旧版本已经对所有仍受保护的观察者都不需要了。
```

如果证明不了，它就必须保留。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
tuple visibility 只回答当前 snapshot 是否能看到某个 tuple version；
cleanup horizon 要证明所有受保护观察者都不再需要这个 version；
VACUUM 和 HOT pruning 因此必须通过 relation-aware OldestXmin、GlobalVisState、replication slot、prepared xact 和 standby feedback 共同判断。
```

本节的核心矛盾是：

```text
空间回收希望尽快删除旧版本；
可见性和复制恢复要求只在全局证明安全后才能删除旧版本。
```

所以本节不把 `HeapTupleSatisfiesMVCC()` 的不可见结论当成 cleanup 许可。

真正的主线是：旧版本先变成当前查询不可见，再经过 horizon 证明，最后才可能被 VACUUM 或 pruning 移除。

## 3. 核心文件分工与阅读顺序

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/storage/procarray.h` | 先看 horizon 计算对外暴露的结构和入口。 |
| 2 | `src/backend/storage/ipc/procarray.c` | 再读 `ComputeXidHorizons()`、ProcArray、prepared xact 与 global visibility 状态。 |
| 3 | `src/include/storage/proc.h` | 对照 `PGPROC` 中 `xmin`、replication 和 backend 状态字段。 |
| 4 | `src/include/utils/snapmgr.h` | 看 active / registered snapshot 如何间接影响 cleanup。 |
| 5 | `src/include/commands/vacuum.h` | 明确 VACUUM cutoff 和 relation-aware horizon 的结构。 |
| 6 | `src/backend/commands/vacuum.c` | 跟 `vacuum_get_cutoffs()` 如何把全局 horizon 转成 relation cutoff。 |
| 7 | `src/backend/access/heap/heapam_visibility.c` | 区分 `HeapTupleSatisfiesMVCC()` 与 `HeapTupleSatisfiesVacuum()`。 |
| 8 | `src/backend/access/heap/vacuumlazy.c` | 看 lazy VACUUM 如何消费 cutoffs 和 global visibility。 |
| 9 | `src/backend/access/heap/pruneheap.c` | 看 HOT pruning 如何用 removable 判断做页内 cleanup。 |
| 10 | `src/backend/replication/slot.c` | 看 replication slot 的 `xmin/catalog_xmin` 如何阻止 cleanup。 |
| 11 | `src/backend/replication/walreceiver.c` / `src/backend/replication/walsender.c` | 对照 hot standby feedback 的 horizon 传播。 |
| 12 | `src/backend/access/transam/twophase.c` | 最后核对 prepared transaction 为什么也进入保护边界。 |

## 4. 从 runtime 现象进入

先做一个最小实验。Session A：

```sql
DROP TABLE IF EXISTS cleanup_demo;
CREATE TABLE cleanup_demo(id int primary key, payload text);
INSERT INTO cleanup_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 30000) AS g;

BEGIN;
SELECT count(*) FROM cleanup_demo;
-- 保持事务打开。
```

Session B：

```sql
DELETE FROM cleanup_demo WHERE id <= 20000;
VACUUM (VERBOSE) cleanup_demo;

SELECT relname, n_live_tup, n_dead_tup
FROM pg_stat_user_tables
WHERE relname = 'cleanup_demo';

SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

现象通常是：

- DELETE 已经提交。
- 新查询看不到被删除的 20000 行。
- VACUUM 不能完全移除旧版本，或者 verbose 输出和统计显示仍有 dead tuple 压力。
- Session A 的 `backend_xmin` 很老。

现在让 Session A：

```sql
COMMIT;
```

Session B 再执行：

```sql
VACUUM (VERBOSE) cleanup_demo;
```

第二次 VACUUM 更有机会移除旧版本。这个实验的关键点不是统计数字精确多少。

`n_dead_tup` 是统计估算，刷新也有时机差。关键是状态关系：

```text
旧版本对新查询不可见；
但 Session A 的旧 snapshot 仍可能需要它；
VACUUM 不能以当前查询为准；
VACUUM 必须以全局 cleanup horizon 为准。
```

这就是本节唯一主问题的 runtime 入口。

## 5. 当前查询可见性与 cleanup 可见性不是同一个问题

`HeapTupleSatisfiesMVCC()` 解决的是当前 snapshot 下是否可见。它会看 tuple 的 `xmin`、`xmax`、hint bit、当前事务、snapshot 的 `xip/subxip`、CLOG 结果。

它的问题是：

```text
这个 tuple version 是否应该被当前扫描返回？
```

`HeapTupleSatisfiesVacuum()` 解决的是 VACUUM 视角。它的问题是：

```text
这个 tuple version 是否仍可能对任何 running 或受保护的观察者可见？
```

这两个问题用到同一批事务状态，但结论层次不同。一个 tuple 对当前 snapshot 不可见，只能说明当前观察者不需要它。

VACUUM 要删除 tuple，必须证明没有任何受保护观察者需要它。源码注释在 `heapam_visibility.c` 里直接说了这一点。

`HeapTupleSatisfiesVacuum()` 主要想知道 tuple 是否 potentially visible to any running transaction。如果是，就不能删除。

`OldestXmin` 是 cutoff。deleted by XIDs >= `OldestXmin` 的 tuple 被认为 recently dead。

这些 tuple 的删除事务可能已经 commit。但仍有开放事务可能看见删除前的旧版本。

因此它不能直接变成 dead。这里的状态转换是：

```text
deleter committed
  -> HeapTupleSatisfiesVacuumHorizon() 返回 HEAPTUPLE_RECENTLY_DEAD，并给出 dead_after = xmax
  -> HeapTupleSatisfiesVacuum() 比较 dead_after 与 OldestXmin
  -> dead_after < OldestXmin 才升级为 HEAPTUPLE_DEAD
```

所以 `HEAPTUPLE_RECENTLY_DEAD` 不是“删除事务还没提交”。它经常表示删除事务已经提交，但删除时间还不够老。

这就是很多诊断误判的来源。

## 6. `HeapTupleSatisfiesVacuumHorizon()` 的状态压缩

先从 tuple 本身看。旧版本通常来自 `UPDATE` 或 `DELETE`。

旧 tuple 的 `xmax` 指向删除或更新它的事务。如果 `xmax` 没有提交，tuple 不能按 dead 处理。

如果 `xmax` 只是 lock-only，tuple 仍 live。如果 `xmax` 是 MultiXact，还要找真正 updater。

这些分支本节不逐项展开。本节只抓住普通删除已提交的路径。

`HeapTupleSatisfiesVacuumHorizon()` 先确认插入者 `xmin`。如果插入事务 abort，tuple 从未对其他事务可见，可以直接 `HEAPTUPLE_DEAD`。

如果插入事务仍 running，返回 `HEAPTUPLE_INSERT_IN_PROGRESS`。如果插入事务 committed，继续看删除者。

如果没有有效 `xmax`，tuple live。如果 `xmax` 是 lock-only，tuple live。

如果 `xmax` 是普通事务，并且删除事务仍 running，返回 `HEAPTUPLE_DELETE_IN_PROGRESS`。如果删除事务 abort，tuple live。

如果删除事务 committed：

```text
*dead_after = HeapTupleHeaderGetRawXmax(tuple)
return HEAPTUPLE_RECENTLY_DEAD
```

这一步非常关键。它没有自己决定是否可以删除。

它把“删除发生在什么 XID”交给调用者。调用者拿 `dead_after` 对比 horizon。

`HeapTupleSatisfiesVacuum()` 的封装是：

```text
res = HeapTupleSatisfiesVacuumHorizon(...)
if res == HEAPTUPLE_RECENTLY_DEAD:
    if dead_after < OldestXmin:
        res = HEAPTUPLE_DEAD
```

所以 `OldestXmin` 是 RECENTLY_DEAD 到 DEAD 的门。这个门不是当前查询 snapshot。

它来自 `GetOldestNonRemovableTransactionId(rel)`。

## 7. `vacuum_get_cutoffs()` 如何取得 `OldestXmin`

`vacuum_get_cutoffs()` 位于 `commands/vacuum.c`。它处理的是 VACUUM relation 级别的 cutoff。

本节关心第一步：

```text
cutoffs->OldestXmin = GetOldestNonRemovableTransactionId(rel)
```

源码注释说，VACUUM 总是可以忽略正在运行 lazy vacuum 的进程。原因是这里的值只用于决定表里哪些 tuple 必须保留。

lazy vacuum 通常不会把自己的 XID 写入用户表中的 tuple。即使它有 snapshot，它也不需要用 snapshot-based lookup 保护别的表里的旧版本。

但是这个忽略只适用于 removable horizon。`ComputeXidHorizons()` 还会计算 `oldest_considered_running`。

那个值用于 pg_subtrans truncate 等场景，不能忽略 lazy vacuum。这就是同一个 ProcArray 扫描里会有多个 horizon 的原因。

`vacuum_get_cutoffs()` 还会检查 wraparound 压力。如果 `OldestXmin` 被拖得太老，它会 warning：

```text
cutoff for removing and freezing tuples is far in the past
Close open transactions soon to avoid wraparound problems.
You might also need to commit or roll back old prepared transactions, or drop stale replication slots.
```

这个 hint 本身就是本节主线的 DBA 可见出口。旧事务、prepared xact、replication slot 都可能把 cleanup horizon 固定住。

`vacuum_get_cutoffs()` 还计算 `FreezeLimit`。`FreezeLimit` 必须不晚于 `OldestXmin`。

原因是不能把一个仍可能被旧 snapshot 解释的 XID 冻结成更老的语义。

本节不展开 freezing。但要记住：

```text
cleanup horizon 和 freeze horizon 相互约束；
一个过老 snapshot 不只阻止删除旧版本，也会阻止 relfrozenxid 推进。
```

## 8. `ComputeXidHorizons()` 的核心模型

`ComputeXidHorizons()` 是本节最重要的源码入口。它位于 `procarray.c`。

它扫描 ProcArray，同时读取 replication slot 聚合 xmin。输出是 `ComputeXidHorizonsResult`。

这个 result 不是一个值，而是一组用途不同的 horizon。主要字段：

- `latest_completed`
- `slot_xmin`
- `slot_catalog_xmin`
- `oldest_considered_running`
- `shared_oldest_nonremovable`
- `shared_oldest_nonremovable_raw`
- `catalog_oldest_nonremovable`
- `data_oldest_nonremovable`
- `temp_oldest_nonremovable`

这些名字很长，但它们都围绕一个问题。不同 relation 能被哪些 observer 看到？

shared relation 能被所有数据库看到。普通 data relation 只能被当前数据库的 backend 看到。

catalog relation 还可能被 logical decoding 通过 `catalog_xmin` 看到。temp relation 只能被当前 session 看到。

所以一个全局 `OldestXmin` 会过度保守。PostgreSQL 选择 relation-aware horizon。

`ComputeXidHorizons()` 初始化时用 `latestCompletedXid + 1` 作为起点。这个值是一个保守下界。

它保护后续可能进入 ProcArray 的事务。然后它扫描每个 proc。

对每个 proc，它读取：

```text
xid = ProcGlobal->xids[index]
xmin = proc->xmin
xmin = older(xmin, xid)
```

为什么同时考虑 `xid` 和 `xmin`？因为事务可能已经有 XID 但还没设置 snapshot xmin。

反过来，也可能有 `xmin` 但没有 XID。只看其中一个会漏掉边界。

如果两者都 invalid，该 proc 不影响 horizon。否则它先影响 `oldest_considered_running`。

然后，如果该 proc 是 `PROC_IN_VACUUM` 或 `PROC_IN_LOGICAL_DECODING`，removable horizon 会跳过它。逻辑 decoding 的 xmin 由 slot 管。

lazy vacuum 对删除 tuple 的保留需求可以忽略。接着 shared horizon 考虑所有数据库。

data horizon 通常只考虑当前数据库。但是有例外。

如果当前 backend 还没有 `MyDatabaseId`，不能安全忽略其他数据库。如果 proc 设置 `PROC_AFFECTS_ALL_HORIZONS`，也必须纳入所有数据库。

recovery 中也不能精确按 database 切分，因为 running XID 来自 KnownAssignedXids。如果在 recovery 中，`KnownAssignedXidsGetOldestXmin()` 也会被纳入。

扫描结束后，slot horizon 会进一步把边界往老处拉。`slot_xmin` 影响 shared 和 data。

`slot_catalog_xmin` 影响 shared 和 catalog。最后，`oldest_considered_running` 会被修正到不晚于这些 nonremovable horizons。

这保证 pg_subtrans 等 running 判断仍有足够历史可查。

## 9. relation-aware horizon

`GetOldestNonRemovableTransactionId(rel)` 是 VACUUM 用的 public wrapper。它调用 `ComputeXidHorizons()`。

然后根据 `GlobalVisHorizonKindForRel(rel)` 返回某个字段。规则是：

```text
rel == NULL 或 shared relation 或 recovery:
  shared_oldest_nonremovable

catalog relation 或 logical decoding 可访问 relation:
  catalog_oldest_nonremovable

普通非本地 relation:
  data_oldest_nonremovable

local temp relation:
  temp_oldest_nonremovable
```

这说明 `OldestXmin` 不是固定全局值。同一时刻，不同 relation 可以有不同 cleanup cutoff。

为什么 shared relation 更保守？因为 shared catalog 可以被所有 database 的 backend 访问。

不能只看当前 database 的 snapshots。为什么 catalog relation 比普通 data relation 更保守？

因为 logical decoding 可能需要 catalog tuple 来解释历史 WAL。即使当前数据库的普通查询不需要，slot 的 `catalog_xmin` 仍可能需要。

为什么 temp relation 更激进？因为其他 backend 看不到这个 session 的 temp table。

它不需要被其他 backend snapshot 保护。但是本 backend 自己的 top XID 仍可能影响 temp horizon。

这套 relation-aware 模型解释了一个常见现象。用户表 VACUUM 似乎不受某些其他数据库长事务影响。

shared catalog 或 database-wide freeze 相关操作却可能受影响。这不是不一致。

这是 relation visibility domain 不同。

## 10. replication slot 如何进入 horizon

Replication slot 不是 snapshot。但它可以固定 xmin。

`slot.c` 中 `ReplicationSlotsComputeRequiredXmin()` 会扫描所有 slot。它读取每个 slot 的 `effective_xmin` 和 `effective_catalog_xmin`。

无效或 invalidated slot 不参与。最终得到最老的 `agg_xmin` 和 `agg_catalog_xmin`。

然后调用：

```text
ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin, already_locked)
```

`ProcArraySetReplicationSlotXmin()` 把它们写到：

```text
procArray->replication_slot_xmin
procArray->replication_slot_catalog_xmin
```

`ComputeXidHorizons()` 在持 `ProcArrayLock` 时读取这两个字段。读取发生在锁内，是为了和 ProcArray horizon 计算形成一致边界。

随后它把 `slot_xmin` 应用到 shared/data horizon。把 `slot_catalog_xmin` 应用到 shared/catalog horizon。

这解释了为什么一个没有 active SQL 的 replication slot 仍能阻止 VACUUM。slot 表达的是下游仍可能需要的数据或 catalog 历史。

它不是 backend 当前正在执行查询。逻辑 replication slot 最常见的是固定 `catalog_xmin`。

逻辑 decoding 需要 catalog 历史来解释 WAL 中的 tuple、type、relation metadata。如果 catalog tuple 被过早 vacuum，decoding 无法正确解释历史变化。

物理 replication slot 更多固定 WAL 保留。但 hot standby feedback 或 physical slot feedback 也可能固定 xmin。

诊断时要看：

```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

`xmin` 和 `catalog_xmin` 不为空时，它们就是 cleanup horizon 的直接输入。删除 stale slot 可能释放 horizon。

但这可能导致下游复制或 decoding 无法继续。所以处理 slot 不是单纯 DBA 清理动作。

它是 correctness 与空间回收之间的取舍。

## 11. prepared transaction 如何像 running 一样保留

prepared transaction 没有普通 backend 正在执行。`pg_stat_activity` 里也不表现成一个普通活动 pid。

但它仍然不能被当成 finished。两阶段提交的语义是：

```text
PREPARE TRANSACTION 后，事务结果还没有 commit 或 abort。
系统必须保留它的锁、XID、subxids 和足够状态，直到 COMMIT PREPARED 或 ROLLBACK PREPARED。
```

源码在 `twophase.c` 顶部注释说，每个 global transaction 有一个 dummy `PGPROC`。这个 dummy `PGPROC` 让 `TransactionIdIsInProgress()` 把 prepared xact 视为 running。

也方便把 locks 挂到这个 `PGPROC` 上。`MarkAsPreparingGuts()` 初始化 dummy proc。

它设置：

- `proc->xid = xid`
- `proc->pid = 0`
- `proc->databaseId = databaseid`
- `proc->roleId = owner`
- `proc->subxidStatus`
- `proc->subxids`

`GXactLoadSubxactData()` 把 prepared transaction 的 subxacts 加载到 dummy proc。如果超过 `PGPROC_MAX_CACHED_SUBXIDS`，设置 overflow。

`MarkAsPrepared()` 最后调用：

```text
ProcArrayAdd(GetPGProcByNumber(gxact->pgprocno))
```

从这时起，prepared xact 进入 ProcArray。ProcArray 扫描会看到它。

`ComputeXidHorizons()` 会把它的 `xid` 纳入 horizon。所以 VACUUM 必须保留它可能需要的旧版本。

完成 prepared transaction 时，`FinishPreparedTransaction()` 的顺序也很关键。它先写 commit/abort WAL。

再更新 pg_xact。然后：

```text
ProcArrayRemove(proc, latestXid)
```

只有从 ProcArray 移除后，它才不再影响 running set 和 cleanup horizon。后面才处理 callbacks 和 `RemoveGXact()`。

runtime 观察入口：

```sql
SELECT transaction, gid, prepared, owner, database
FROM pg_prepared_xacts;
```

如果有很老的 prepared transaction，它可能拖住 VACUUM。`vacuum_get_cutoffs()` 的 warning hint 也明确提到 commit or roll back old prepared transactions。

## 12. hot standby feedback 的两面

standby 上的查询也可能需要 primary 保留旧版本。否则 primary VACUUM 删除了 standby query 需要的 row version，standby 在 replay cleanup WAL 时就可能出现 recovery conflict。

PostgreSQL 的一个选择是取消 standby query。另一个选择是 standby 通过 hot standby feedback 请求 primary 保留较老 xmin。

`walreceiver.c` 的 `XLogWalRcvSendHSFeedback()` 负责在 standby 端发送 feedback。它先检查 `hot_standby_feedback` 和 `wal_receiver_status_interval`。

Hot Standby 未可用时不发送。启用时调用：

```text
GetReplicationHorizons(&xmin, &catalog_xmin)
```

`GetReplicationHorizons()` 内部调用 `ComputeXidHorizons()`。但它返回的是给上游看的两个值。

`xmin` 使用 `shared_oldest_nonremovable_raw`。`catalog_xmin` 使用 `slot_catalog_xmin`。

源码注释说明，不想把 slot catalog xmin 混进普通 xmin。这样 primary 可以更激进地清理 data table，同时单独保护 catalog horizon。

primary 的 walsender 在 `ProcessStandbyHSFeedbackMessage()` 中处理这些值。它先验证 epoch 和 xid 没有明显不合理。

然后如果使用 replication slot，就调用 `PhysicalReplicationSlotNewXmin(feedbackXmin, feedbackCatalogXmin)`。如果没有 slot，就把值放进 walsender 的 `MyProc->xmin`。

如果 catalog xmin 更老而没有 slot 能分别记录，就取两者更老者。这解释了 hot standby feedback 的两面。

它能减少 standby query cancellation。但它把 standby 的旧 snapshot 需求传回 primary。

primary 的 `ComputeXidHorizons()` 会把这个 xmin 当成 cleanup horizon 输入。结果是 primary 上 VACUUM 可能更久不能移除旧版本。

这不是 bug。这是用户选择了 query continuity over primary space reclamation。

如果 standby 断开且没有 slot，feedback 不是持久保护。源码注释也承认这种场景下 primary 可能已经移除 standby 想要的数据。

Hot Standby 会通过 conflict 处理取消需要已删除数据的 standby query。如果使用 slot，保护更持久，但也更容易产生长期 bloat。

## 13. `GlobalVisState` 为什么有两个边界

VACUUM relation 开始时用 `OldestXmin` 做一个稳定 cutoff。但 heap pruning 和一些 on-access cleanup 不总是走完整 VACUUM relation 流程。

频繁精确扫描 ProcArray 成本也高。所以 `procarray.c` 维护 `GlobalVisState`。

它有两个 `FullTransactionId` 边界：

- `definitely_needed`
- `maybe_needed`

注释给出的语义是：

```text
rows deleted by XIDs >= definitely_needed are definitely still visible to someone;
rows deleted by XIDs < maybe_needed can definitely be removed;
between them，需要时可以重新 ComputeXidHorizons() 获得更精确答案。
```

这不是精确区间索引。它是降低 ProcArray 扫描频率的近似缓存。

`GlobalVisTestFor(rel)` 根据 relation kind 选择四个全局 state 之一：

- `GlobalVisSharedRels`
- `GlobalVisCatalogRels`
- `GlobalVisDataRels`
- `GlobalVisTempRels`

它要求调用时已有 active 或 registered snapshot。源码中有 `Assert(RecentXmin)`。

因为 32-bit XID 转 FullTransactionId 需要 wraparound 保护上下文。`GlobalVisTestIsRemovableFullXid()` 的逻辑：

```text
if fxid < maybe_needed:
    true
if fxid >= definitely_needed:
    false
if allow_update && GlobalVisTestShouldUpdate(state):
    GlobalVisUpdate()
    return fxid < maybe_needed
else:
    false
```

也就是说，它宁可保守地返回 false。false 表示不能证明 removable。

这可能少回收一些 tuple，但不会错误删除仍需要的版本。`GlobalVisTestShouldUpdate()` 用 `RecentXmin` 是否变化作为启发。

如果最近 snapshot 的 xmin 没变，重新计算通常收益不大。这说明 cleanup 判断也有 hot path 成本模型。

不能每个 tuple 都全量扫描 ProcArray。所以 PostgreSQL 接受近似和延迟更新，只要 false negative 是安全的。

## 14. HOT pruning 如何使用 `GlobalVisTestIsRemovableXid()`

HOT pruning 不总是在 VACUUM 中发生。普通访问页面时也可能尝试 prune。

`heap_page_prune_opt()` 先检查 `RecoveryInProgress()`。recovery 中不能写 WAL，所以不清理页面。

然后看 page header 的 `prune_xid`。如果没有 valid `prune_xid`，没有必要计算 horizon。

如果有，它获取：

```text
vistest = GlobalVisTestFor(relation)
```

然后：

```text
if (!GlobalVisTestIsRemovableXid(vistest, prune_xid, true))
    return;
```

这里的判断是页面级快速过滤。如果 page 上最早可能 prune 的 XID 仍不能证明 removable，就不值得继续。

如果通过，才尝试 cleanup lock。真正逐 tuple 判断在 `heap_prune_satisfies_vacuum()`。

它调用 `HeapTupleSatisfiesVacuumHorizon()`。如果返回不是 `RECENTLY_DEAD`，直接返回。

如果是 `RECENTLY_DEAD`，先看 VACUUM cutoffs。VACUUM 场景下，如果 `dead_after < OldestXmin`，必须能 prune。

因为 dead tuple 的 `xmax` 不能被 freeze，VACUUM 需要保证 OldestXmin 与 freezing 推进一致。如果没有 cutoffs，或 `dead_after` 不早于 `OldestXmin`，再用 `GlobalVisTestIsRemovableXid()`。

如果 GlobalVis 认为 removable，就返回 `HEAPTUPLE_DEAD`。否则保持 `RECENTLY_DEAD`。

这个路径说明：

```text
OldestXmin 是 VACUUM relation 级稳定 cutoff；
GlobalVisState 是 pruning/visibility helper 的近似动态 cutoff；
两者必须保持不让 VACUUM 对同一页面产生相互矛盾的 freeze/prune 判断。
```

`vacuumlazy.c` 中的注释也强调，heap vacuum 需要 `vistest` 和 `OldestXmin` 组合，确保 `heap_page_prune_and_freeze()` 和后续 tuple removal 不会在 horizon 上自相矛盾。

## 15. lazy VACUUM 主流程

`heap_vacuum_rel()` 是 heap lazy vacuum 的入口之一。它先调用：

```text
vacrel->aggressive = vacuum_get_cutoffs(rel, params, &vacrel->cutoffs)
```

然后保存 relation page 数。接着：

```text
vacrel->vistest = GlobalVisTestFor(rel)
```

顺序很重要。先获取稳定 cutoffs。

再获取 pruning 使用的 global visibility state。`vacrel->NewRelfrozenXid` 初始化为 `cutoffs.OldestXmin`。

这说明 relation 级 freeze 推进也从 cleanup horizon 开始。扫描页面时，VACUUM 会对 tuple 调 `HeapTupleSatisfiesVacuum()`。

如果结果是：

- `HEAPTUPLE_LIVE`：保留。
- `HEAPTUPLE_INSERT_IN_PROGRESS`：不能删除。
- `HEAPTUPLE_DELETE_IN_PROGRESS`：不能删除。
- `HEAPTUPLE_RECENTLY_DEAD`：删除已提交但不够老，不能删除。
- `HEAPTUPLE_DEAD`：可以进入删除处理。

`RECENTLY_DEAD` 是本节最重要的返回值。它让 VACUUM 在 correctness 和 reclaim 之间保守。

如果清理太早，旧 snapshot 可能找不到它应该看到的版本。如果清理太晚，只是 bloat 和 I/O 成本。

PostgreSQL 在这里选择 false negative。也就是可删除但暂时不删，比误删仍需版本更可接受。

## 16. recovery 与 KnownAssignedXids

Hot standby 模式下，standby 的 ProcArray 不代表 primary 上正在运行的事务。standby backend 自身不会运行带 XID 的写事务。

但 standby query 的 MVCC snapshot 必须知道 primary 上哪些 XID 在 WAL 位置上仍 running。`procarray.c` 顶部注释说，hot standby 使用 KnownAssignedXids 数组维护这些 running XIDs。

如果 standby snapshot 漏掉这些 XID，它会把它们误认为已经完成，造成 MVCC failure。`GetSnapshotData()` 在 recovery 中把 KnownAssignedXids 放进 `subxip[]`，`xip[]` 留空。

这是一个简化实现。因为 recovery 中不总是知道 top-level 和 subxact 的完整区分。

`ComputeXidHorizons()` 在 recovery 中也会调用 `KnownAssignedXidsGetOldestXmin()`。然后把 `kaxmin` 应用到：

- `oldest_considered_running`
- `shared_oldest_nonremovable`
- `data_oldest_nonremovable`

temp relation 在 recovery 中不能访问。这说明 standby 的 cleanup horizon 也不是只看本机 backends。

它要看 WAL replay 中已知仍 running 的 primary XIDs。如果 primary 已经 cleanup 了 standby 需要的 tuple，standby 会通过 recovery conflict 取消查询。

hot standby feedback 则是把这个需求提前反馈给 primary，尝试避免冲突。

## 17. 为什么 horizon 可能后退

`ComputeXidHorizons()` 的注释承认，重复调用得到的值可能向后移动。这听起来反直觉。

cleanup horizon 不是只会前进吗？源码给了几个原因。

如果当前数据库没有 running transaction，普通 data horizon 可能是 `latestCompletedXid + 1`。之后一个新事务开始并获取 snapshot。

它的 `xmin` 可能包含其他数据库中更早开始的事务。下一次当前数据库的 horizon 可能比上一次更老。

这仍然安全。因为第一次得到的值在当时对当前 relation 是保守正确的。

另一个原因是 replication。walsender 可能根据 standby feedback 设置 xmin。

这个 xmin 对应的是 primary 上已经不 running、但 standby 仍在 replay 或查询需要的事务。如果没有 replication slot 持久保护，feedback 中断后 primary 可能无法继续保证数据存在。

Hot Standby 会用冲突取消兜底。这说明 cleanup horizon 不是单调事务计数器。

它是当前系统约束的保守快照。它可能因新的 holder、slot、feedback、database context、recovery state 而变化。

课程里不要把 `OldestXmin` 当作一个全局单调水位。更准确的模型是：

```text
每次 cleanup 决策使用当时可证明安全的 relation-aware lower bound。
```

## 18. 成本模型

cleanup horizon 的成本分三层。第一层是计算成本。

`ComputeXidHorizons()` 要扫描 ProcArray。成本随 backend 数、prepared xact 数、walsender、slot 状态增长。

它还要考虑 relation kind、database id、status flags 和 recovery。这就是为什么 `GlobalVisState` 不愿每次 tuple 判断都重算。

第二层是保守成本。任何老 `xmin` 都会拖住 cleanup。

表现为 dead tuple 留在 heap page、HOT chain 变长、page pruning 失败、index vacuum 需要处理更多 dead TID、visibility map all-visible 推进变慢、freeze 不能推进。这些成本随 tuple 数、更新频率、long transaction 时长扩张。

第三层是跨节点成本。replication slot 和 hot standby feedback 把下游状态传回 primary。

primary 的空间回收会为下游查询连续性或 decoding correctness 让步。这类成本不一定在 primary 上有明显 active query。

它可能只表现为 slot 的 `xmin/catalog_xmin` 不动、WAL retained、dead tuples 积累。第四层是 slow path 成本。

SubXID overflow、pg_subtrans lookup、CLOG miss、MultiXact lookup 都可能在 visibility 或 vacuum 判断中出现。

本节不展开它们，但要记住 cleanup 判断不是纯比较 `xmax < OldestXmin`。它前面还要确认插入者、删除者、locker、MultiXact 和 hint bit 状态。

性能诊断时要分清：

```text
不能删除，是 horizon 太老；
判断变慢，是 visibility/check 状态复杂；
删除成本高，是 dead tuple / index / I/O 放大。
```

三者可能同时出现，但根因不同。

## 19. 观测与诊断入口

第一组入口是 snapshot holder。

```sql
SELECT pid, datname, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

`backend_xmin` 非空说明 backend 对 horizon 有贡献。它不能告诉你是否 active、registered、cursor、catalog snapshot 或 exported snapshot。

但它是第一入口。第二组入口是 prepared transaction。

```sql
SELECT transaction, gid, prepared, owner, database
FROM pg_prepared_xacts
ORDER BY prepared;
```

老 prepared xact 会以 dummy `PGPROC` 方式进入 ProcArray。它可能没有普通 backend pid。

第三组入口是 replication slot。

```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
ORDER BY xmin NULLS LAST, catalog_xmin NULLS LAST;
```

`xmin` 和 `catalog_xmin` 直接影响 `ComputeXidHorizons()`。第四组入口是 table bloat 与 vacuum。

```sql
VACUUM (VERBOSE) cleanup_demo;

SELECT relname, n_live_tup, n_dead_tup, vacuum_count, autovacuum_count
FROM pg_stat_user_tables
WHERE relname = 'cleanup_demo';
```

`n_dead_tup` 是估算。VACUUM VERBOSE 更接近本次操作现场。

但它也不是完整因果。需要和 `backend_xmin`、slot、prepared xact、standby 组合看。

第五组入口是 standby feedback。在 standby 看：

```sql
SHOW hot_standby_feedback;
SHOW wal_receiver_status_interval;
```

在 primary 看 walsender 和 slots。如果使用 physical slot，feedback xmin 可能体现在 slot。

没有 slot 时，walsender 的 `MyProc->xmin` 会参与 horizon。第六组入口是源码断点。

建议断点：

- `ComputeXidHorizons()`
- `GetOldestNonRemovableTransactionId()`
- `vacuum_get_cutoffs()`
- `HeapTupleSatisfiesVacuumHorizon()`
- `HeapTupleSatisfiesVacuum()`
- `GlobalVisTestIsRemovableXid()`
- `ProcArraySetReplicationSlotXmin()`
- `MarkAsPrepared()`
- `FinishPreparedTransaction()`
- `ProcessStandbyHSFeedbackMessage()`

观察变量：

- `h->data_oldest_nonremovable`
- `h->catalog_oldest_nonremovable`
- `h->shared_oldest_nonremovable`
- `h->slot_xmin`
- `h->slot_catalog_xmin`
- `cutoffs->OldestXmin`
- `dead_after`
- `state->maybe_needed`
- `state->definitely_needed`

## 20. 常见误区

误区一：把“当前查询看不到”理解成“VACUUM 可以删”。当前查询只是一个观察者。

VACUUM 要考虑所有受保护观察者。误区二：把 `xmax` 已提交理解成 `HEAPTUPLE_DEAD`。

`HeapTupleSatisfiesVacuumHorizon()` 对 committed delete 通常先返回 `RECENTLY_DEAD`。只有 `dead_after < OldestXmin` 才能升级。

误区三：把 `OldestXmin` 当成当前 snapshot 的 `xmin`。`OldestXmin` 来自 relation-aware cleanup horizon。

它可能受其他 backend、slot、prepared xact、standby feedback 影响。误区四：只查 `pg_stat_activity`，不查 `pg_prepared_xacts` 和 `pg_replication_slots`。

prepared xact 和 slot 都可以在没有明显 active query 的情况下固定 horizon。误区五：认为 hot standby feedback 是免费减少 conflict。

它把 standby query 的需求转移到 primary 空间回收压力。误区六：认为 `GlobalVisTestIsRemovableXid()` 返回 false 就说明 XID 一定 running。

false 只说明不能证明 removable。它可能是保守近似。

误区七：把所有表共用一个 cleanup horizon。shared、catalog、data、temp relation 的 horizon 不同。

诊断时要先看 relation kind。

## 21. 课堂实验

实验一：long transaction 阻止 cleanup。按第 4 节的 Session A / Session B 执行。

在 Session A 提交前后各跑一次 `VACUUM (VERBOSE)`。结合 `pg_stat_activity.backend_xmin` 解释差异。

回到源码看：

```text
vacuum_get_cutoffs()
  -> GetOldestNonRemovableTransactionId(rel)
  -> ComputeXidHorizons()
  -> HeapTupleSatisfiesVacuum()
```

实验二：prepared transaction 固定 horizon。需要 `max_prepared_transactions > 0`。

如果当前实例没有开启，不要在生产实例随意改。测试库中执行：

```sql
BEGIN;
UPDATE cleanup_demo SET payload = payload || 'p' WHERE id = 25000;
PREPARE TRANSACTION 'cleanup-horizon-demo';

SELECT * FROM pg_prepared_xacts WHERE gid = 'cleanup-horizon-demo';
```

另一个 session 删除并 vacuum。观察 warning 或 cleanup 受阻。

最后必须执行：

```sql
ROLLBACK PREPARED 'cleanup-horizon-demo';
```

回到源码看 `MarkAsPrepared()` 如何 `ProcArrayAdd()`，以及 `FinishPreparedTransaction()` 如何 `ProcArrayRemove()`。实验三：replication slot horizon。

在测试实例上创建 logical slot 需要 `wal_level = logical`。如果没有配置，不要强行在当前实例做。

已有 slot 时观察：

```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin
FROM pg_replication_slots;
```

结合 `ProcArraySetReplicationSlotXmin()` 和 `ComputeXidHorizons()` 解释 `catalog_xmin` 为什么影响 catalog cleanup。实验四：断点观察 `dead_after`。

在测试实例上给 `HeapTupleSatisfiesVacuumHorizon()` 和 `HeapTupleSatisfiesVacuum()` 打断点。删除一行并 vacuum。

观察 committed delete 如何先形成 `HEAPTUPLE_RECENTLY_DEAD`，再由 `OldestXmin` 决定是否变成 `HEAPTUPLE_DEAD`。

## 22. 讨论题

1. 为什么 `HeapTupleSatisfiesVacuum()` 不能直接复用当前查询的 `HeapTupleSatisfiesMVCC()` 结论？
2. 为什么 `ComputeXidHorizons()` 同时考虑 `proc->xmin` 和 `proc->xid`？
3. 为什么 lazy vacuum 可以被 removable horizon 忽略，但不能随便被 `oldest_considered_running` 忽略？
4. 为什么 catalog relation 需要考虑 `slot_catalog_xmin`，普通 user table 通常不需要？
5. prepared transaction 的 dummy `PGPROC` 为什么必须进入 ProcArray，而不是只存在 `pg_prepared_xacts` 视图里？
6. `GlobalVisTestIsRemovableXid()` 为什么宁可保守返回 false？
7. hot standby feedback 关闭后，standby query conflict 和 primary bloat 的取舍如何变化？
8. 为什么 relation-aware horizon 可能让不同表在同一时刻有不同 cleanup 结果？

## 23. 本节小结

本节只回答一个问题：旧 tuple 对当前查询不可见以后，为什么不能立刻 cleanup。答案是 current visibility 和 cleanup safety 是两个不同问题。

当前查询只代表一个 snapshot。cleanup 必须证明没有任何受保护观察者仍需要旧版本。

`HeapTupleSatisfiesVacuumHorizon()` 把 committed delete 压成 `RECENTLY_DEAD + dead_after`。`HeapTupleSatisfiesVacuum()` 用 `OldestXmin` 决定它是否能升级为 `DEAD`。

`OldestXmin` 来自 `GetOldestNonRemovableTransactionId(rel)`。这个函数通过 `ComputeXidHorizons()` 把 ProcArray、backend `xmin/xid`、relation kind、replication slot、prepared xact、recovery KnownAssignedXids 和 hot standby feedback 压缩成 relation-aware horizon。

`GlobalVisState` 用 `maybe_needed` 和 `definitely_needed` 在 hot path 上保守判断 removability。它减少反复 ProcArray 扫描，但允许 false negative。

VACUUM 和 HOT pruning 宁可少删，也不能错删。replication slot、prepared xact 和 standby feedback 都不是边角特例。

它们都是“仍可能观察旧世界”的不同表达。slot 用 `xmin/catalog_xmin` 表达 downstream 或 decoding 需求。

prepared xact 用 dummy `PGPROC` 表达未完成事务。hot standby feedback 把 standby snapshot 需求传回 primary。

可迁移规律是：

```text
visibility 是某个 observer 的局部判断；
reclaim 是所有受保护 observer 的全局证明。
系统无法证明安全时，正确做法是保留旧状态并把成本转化为空间、I/O 和后续 cleanup 压力。
```

到这里，事务可见性这组课完成了从 XID 出现、事务结果、snapshot 字段、SubXID fallback、隔离级别、command id、snapshot 生命周期到 cleanup horizon 的闭环。
