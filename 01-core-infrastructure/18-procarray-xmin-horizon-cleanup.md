# PostgreSQL xmin horizon 与 cleanup 边界

## 课程定位

前置知识：已经理解 `PGPROC` 是 backend 在 shared memory 中的身份，`ProcArray` 是事务状态 publication set，`GetSnapshotData()` 会把全局 running XID 集合压缩成本地 `SnapshotData`。

本节唯一主问题：

```text
为什么一个 backend 的 active snapshot、replication / recovery 相关状态会阻止 VACUUM 移除旧 tuple version，ProcArray 如何把局部 xmin 汇总成全局 cleanup horizon？
```

核心矛盾：heap tuple version 一旦被更新或删除，就可能已经对“当前最新查询”不可见；但它仍可能对某个旧 snapshot、standby query、logical decoding、prepared transaction、catalog decoding 或 `pg_subtrans` 查询有意义。PostgreSQL 不能只按当前事务提交状态立刻清理旧版本，也不能为了绝对精确而在每次剪枝、VACUUM、visibility map 判断时完整扫描所有 backend。因此系统把每个观察者的最低需求发布成 `xmin` / slot xmin / recovery running XID，再由 `ComputeXidHorizons()` 汇总成一组 relation-sensitive cleanup horizon。

学完后应能判断：

```text
active snapshot 为什么会推进或钉住 MyProc->xmin；
VACUUM 的 OldestXmin 为什么来自 GetOldestNonRemovableTransactionId(rel)；
ComputeXidHorizons() 为什么同时看 proc->xmin 和 proc 的 top-level xid；
shared / catalog / data / temp horizon 为什么不同；
PROC_IN_VACUUM、PROC_IN_LOGICAL_DECODING、PROC_AFFECTS_ALL_HORIZONS 如何改变 horizon 计算；
replication slot 的 xmin / catalog_xmin 如何阻止 row version 和 catalog tuple 被过早移除；
recovery 下 KnownAssignedXids 为什么也参与 cleanup 边界；
GlobalVisState 的 maybe_needed / definitely_needed 如何减少频繁精确计算；
为什么 horizon 可以保守、可以回退，但不能过于激进。
```

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `6b48f5d1a741`。

## 1. 本节在总主线中的位置

上一节讲的是 snapshot 获取：

```text
GetSnapshotData()
  -> 扫描 ProcArray
  -> 构造 xmin / xmax / xip / subxip
  -> 安装 MyProc->xmin
  -> 后续 tuple visibility 查本地 SnapshotData
```

本节换到 cleanup 视角：

```text
一个 tuple version 被 UPDATE / DELETE 变成旧版本
  -> 删除事务可能已经提交
  -> 当前查询也许已经看不到它
  -> 但某个旧 snapshot / replication slot / standby / prepared xact 仍可能需要它
  -> VACUUM 必须先求一个全局 cleanup horizon
  -> 只有删除 XID 早于 horizon 的旧版本才可以被物理移除
```

这里的关键转变是：

```text
SnapshotData 解决“我现在能看见什么”；
xmin horizon 解决“全系统以后还有谁可能需要旧版本”。
```

一个常见误解是：tuple 的 `xmax` 已提交，所以该 tuple 一定能删。对于当前 snapshot 来说它可能确实不可见，但 VACUUM 要回答的问题更强：

```text
是否不存在任何仍然合法的观察者会把这个 deleting transaction 视为未完成？
```

这个问题不能由 heap page 自己回答，也不能由单个 backend-local 状态回答。它必须从 ProcArray、replication slots、recovery running XIDs、prepared transactions 和当前 relation 类型共同推导。

本节不展开 tuple visibility 的所有 `infomask` 分支，也不深入 wraparound freeze 的全部策略。它只围绕一条线：

```text
局部 xmin 发布
  -> 全局 horizon 汇总
  -> VACUUM / pruning / visibility map 消费
  -> cleanup 是否安全
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
每个 backend 用 MyProc->xmin 发布自己仍持有的最老 snapshot；
replication slot 用 replication_slot_xmin / replication_slot_catalog_xmin 发布复制或解码仍需要的最老事务；
recovery 用 KnownAssignedXids 发布 standby 视角下仍可能 running 的 XID；
ComputeXidHorizons() 在 ProcArrayLock shared 下读取这些状态，计算 shared / catalog / data / temp / oldest-considered-running 等 horizon；
VACUUM 和 pruning 用这些 horizon 判断旧 tuple version、HOT chain、visibility map、pg_subtrans 截断边界是否可以推进。
```

这背后的 tension 是：

```text
cleanup 想尽快回收空间、缩短 HOT chain、推进 relfrozenxid；
MVCC / replication / recovery correctness 要保证任何合法观察者都不会需要已被移除的版本。
```

PostgreSQL 的折中不是“一个全局 xmin”这么简单，而是一组分层边界：

| 边界 | 保护对象 | 为什么不能合并 |
| --- | --- | --- |
| `oldest_considered_running` | `pg_subtrans` 等“还可能被认为 running”的事务状态查询 | lazy VACUUM 自己也可能需要查 `pg_subtrans`，不能像 tuple cleanup 一样忽略。 |
| `shared_oldest_nonremovable` | shared relation 中的旧 tuple version | shared relation 可被所有数据库 backend 看到。 |
| `catalog_oldest_nonremovable` | 当前数据库 catalog relation 的旧 tuple version | logical decoding 需要 catalog 历史，受 slot `catalog_xmin` 保护。 |
| `data_oldest_nonremovable` | 当前数据库普通用户表旧 tuple version | 其它数据库的普通 snapshot 不可能看到本数据库普通表。 |
| `temp_oldest_nonremovable` | 当前 session 的 temp relation | temp relation 只需要考虑本 backend 自己。 |
| `GlobalVisState` | 高频 pruning / all-visible 判断 | 用近似边界减少重复精确扫描。 |

本节要建立的系统规律是：

```text
回收边界不是“某个对象已经没人看见”；
而是“所有可能观察路径都已经发布了自己不再需要它，并且 cleanup 方按最保守的可见性类别取了交集后的安全下界”。
```

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/backend/utils/time/snapmgr.c` | snapshot active / registered 生命周期，`SnapshotResetXmin()` 如何维护 `MyProc->xmin`。 |
| 2 | `src/backend/storage/ipc/procarray.c` | `ComputeXidHorizons()`、`GetOldestNonRemovableTransactionId()`、`GlobalVisTest*` 主流程。 |
| 3 | `src/include/storage/proc.h` | `PGPROC->xmin`、`statusFlags`、`PROC_IN_VACUUM`、`PROC_IN_LOGICAL_DECODING`、`PROC_AFFECTS_ALL_HORIZONS` 语义。 |
| 4 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()` 如何获得 `OldestXmin`，lazy VACUUM 如何设置 `PROC_IN_VACUUM`。 |
| 5 | `src/backend/access/heap/heapam_visibility.c` | `HeapTupleSatisfiesVacuum()` 如何用 `OldestXmin` 把 `RECENTLY_DEAD` 转成 `DEAD`。 |
| 6 | `src/backend/access/heap/pruneheap.c` | on-access pruning 和 VACUUM pruning 如何用 `GlobalVisTestIsRemovableXid()`。 |
| 7 | `src/backend/access/heap/vacuumlazy.c` | lazy VACUUM 如何组合 `OldestXmin`、`GlobalVisState`、freeze cutoff 和 visibility map。 |
| 8 | `src/backend/replication/slot.c` | replication slot 如何把各 slot 的 `effective_xmin` 汇总到 ProcArray。 |
| 9 | `src/backend/replication/walsender.c` / `src/backend/replication/walreceiver.c` | hot standby feedback 如何把 standby horizon 传回 primary。 |
| 10 | `src/backend/access/transam/twophase.c` | prepared transaction 如何通过 dummy `PGPROC` 留在 ProcArray 中。 |
| 11 | `src/backend/access/transam/README` | snapshot-taking、transaction end、`ComputeXidHorizons()` 的 interlocking 证明。 |

推荐阅读顺序：

```text
先读 snapmgr.c 看 MyProc->xmin 从哪里来
  -> 读 procarray.c 的 ComputeXidHorizons() 看各类 observer 如何被汇总
  -> 读 vacuum.c / heapam_visibility.c 看 OldestXmin 如何变成 tuple cleanup 决策
  -> 读 pruneheap.c / vacuumlazy.c 看 GlobalVis 近似边界如何服务 hot path
  -> 最后读 replication slot / hot standby / two-phase 入口，理解非普通 backend 如何钉住 horizon
```

不要从 VACUUM 的全部扫描流程开始读。VACUUM 代码里有 page skipping、parallel vacuum、index cleanup、freeze、visibility map、eager scan 等很多主题。本节的主线只有一个：

```text
哪些 XID 边界使一个旧 tuple version 还不能被移除？
```

## 4. 关键数据结构与状态

### `MyProc->xmin`: backend 对外发布的最老 snapshot 需求

`MyProc->xmin` 在 `PGPROC` 中，是 shared memory 状态，不是 backend-local 私有变量。它的含义不是“当前事务的 XID”，而是：

```text
本 backend 当前仍持有的所有 active / registered snapshot 中最老的 xmin。
```

它由 snapshot manager 维护。上一节看到 `GetSnapshotData()` 会在获取 snapshot 时安装：

```c
MyProc->xmin = TransactionXmin = snapshot->xmin;
```

后续当 active snapshot 被 pop、registered snapshot 被 unregister、catalog snapshot 被 invalidated 或事务结束时，`snapmgr.c` 会重新计算或清空它。`SnapshotResetXmin()` 的关键语义是：

```text
如果没有 active snapshot，也没有 registered snapshot:
  MyProc->xmin = InvalidTransactionId

如果还有 registered snapshot:
  MyProc->xmin = 其中最小的 snapshot->xmin
```

`RegisteredSnapshots` 用 pairing heap 按 `xmin` 排序，目的就是快速找到最老 snapshot。

要注意：

```text
MyProc->xmin 保护的是“这个 backend 仍可能用旧 snapshot 看数据”；
MyProc->xid 保护的是“这个 backend 自己的事务还在 running set 中”。
```

`ComputeXidHorizons()` 必须同时考虑二者，因为一个事务可能还没有 snapshot 但已经有 XID，也可能已经有 snapshot 但还没有分配 XID。

### `ComputeXidHorizonsResult`: cleanup 方真正使用的一组 horizon

`procarray.c` 中的 `ComputeXidHorizonsResult` 把“全局 xmin”拆成多类：

| 字段 | 语义 |
| --- | --- |
| `latest_completed` | 计算 horizon 时看到的 `TransamVariables->latestCompletedXid`。 |
| `slot_xmin` | 所有 replication slot 的 data xmin 聚合结果。 |
| `slot_catalog_xmin` | 所有 replication slot 的 catalog xmin 聚合结果。 |
| `oldest_considered_running` | 任意 backend 仍可能认为 running 的最老 XID，主要用于 `pg_subtrans` 截断这类状态保留。 |
| `shared_oldest_nonremovable` | shared relation 中不能移除的最老删除 XID。 |
| `shared_oldest_nonremovable_raw` | 未受 slot `catalog_xmin` 影响的 shared horizon，hot standby feedback 需要分开发送。 |
| `catalog_oldest_nonremovable` | 当前数据库 catalog relation 中不能移除的最老删除 XID。 |
| `data_oldest_nonremovable` | 当前数据库普通 relation 中不能移除的最老删除 XID。 |
| `temp_oldest_nonremovable` | 当前 session temp relation 的 cleanup horizon。 |

这个结构体现了一个重要边界：

```text
cleanup horizon 是 relation-sensitive 的；
同一个删除 XID，对 shared catalog、普通用户表和 temp table 的安全删除边界可能不同。
```

为什么需要这么多字段？因为 PostgreSQL 既支持多 database，又支持 shared catalogs、logical decoding、physical standby feedback 和 temp tables。如果只用一个最保守全局 xmin，正确性简单，但普通表 VACUUM 会被其它数据库和 catalog decoding 过度拖住。

### `GlobalVisState`: 高频判断使用的近似边界

`GlobalVisState` 是 `GlobalVisTest*` 家族使用的 backend-lifetime 状态：

```c
struct GlobalVisState
{
    FullTransactionId definitely_needed;
    FullTransactionId maybe_needed;
};
```

它的含义是：

```text
xid < maybe_needed:
  一定没有 snapshot 还认为它 running，可以移除。

xid >= definitely_needed:
  很可能仍被某个 snapshot 需要，不能移除。

maybe_needed <= xid < definitely_needed:
  不确定；必要时调用 ComputeXidHorizons() 刷新精确边界后再判断。
```

这里的关键词是近似。`GetSnapshotData()` 以前承担过更精确的 oldest-xmin 计算，但 PostgreSQL v14 后把它拆出来。原因在 `transam/README` 里说得很清楚：

```text
snapshot 内容只依赖其它 backend 的 xid；
oldest xmin 依赖其它 backend 的 xmin；
xmin 变化比 xid 更频繁，如果 GetSnapshotData() 每次都读取所有 xmin，会造成不必要的 cacheline ping-pong。
```

所以现在的结构是：

```text
GetSnapshotData():
  负责高频 snapshot correctness，顺便维护 approximate GlobalVis boundary。

ComputeXidHorizons():
  在 VACUUM / pruning / slot feedback 需要更精确 cleanup 边界时调用。
```

`GlobalVisState` 用 `FullTransactionId` 而不是 32-bit `TransactionId`，是为了减少 wraparound 附近的边界歧义。`GlobalVisTestIsRemovableXid()` 接收 32-bit XID 时，会根据已有 full-xid 边界转换。

### `ProcArray` 中的 slot xmin 聚合

`ProcArrayStruct` 里有两个 replication slot 相关字段：

```c
TransactionId replication_slot_xmin;
TransactionId replication_slot_catalog_xmin;
```

它们不是单个 slot 的状态，而是所有 slot 的聚合最小值。`src/backend/replication/slot.c` 中的 `ReplicationSlotsComputeRequiredXmin()` 会扫描 slot 数组：

```text
遍历所有 in_use slot
  -> 读取 effective_xmin / effective_catalog_xmin
  -> 忽略 invalidated slot
  -> 取最小值
  -> ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin)
```

这一步把 replication subsystem 的状态投影到 ProcArray horizon 计算里。之后 `ComputeXidHorizons()` 不需要知道每个 slot 的细节，只需要读取这两个聚合字段。

### `statusFlags`: 同一个 xmin 在不同上下文里的解释方式

本节特别关注这些 `PGPROC->statusFlags`：

| flag | 影响 |
| --- | --- |
| `PROC_IN_VACUUM` | lazy VACUUM 的 snapshot 不阻止其它 VACUUM 移除 heap tuple，但仍会影响 `oldest_considered_running`，以保护 `pg_subtrans` 查询。 |
| `PROC_IN_LOGICAL_DECODING` | logical decoding backend 的 tuple 保留主要由 replication slot 管理，因此普通 nonremovable horizon 计算跳过它。 |
| `PROC_AFFECTS_ALL_HORIZONS` | 该 backend 的 xmin 必须影响所有 database 的 horizon，例如 hot standby feedback 相关 walsender。 |
| `PROC_VACUUM_FOR_WRAPAROUND` | wraparound 紧急 VACUUM 标记，本节只关注它与 `PROC_IN_VACUUM` 的生命周期绑定。 |

同一个 `proc->xmin` 如果来自普通 query，会阻止相关 relation 的 cleanup；如果来自 lazy VACUUM，则不会阻止其它 relation tuple cleanup；如果来自 logical decoding，则应由 slot xmin / catalog xmin 保护历史版本。

这就是写内核代码时必须记住的规则：

```text
raw field 不是语义；
field + flag + relation kind + lifecycle state + lock context 才是语义。
```

### prepared transaction 的 dummy `PGPROC`

prepared transaction 没有正在执行的 backend，但它仍是未完成事务。`twophase.c` 文件头注释直接说明：

```text
global transaction 有一个 dummy PGPROC；
这使它能够继续持有 locks，并继续被其它 backend 视为 running transaction。
```

`MarkAsPrepared()` 会调用：

```c
ProcArrayAdd(GetPGProcByNumber(gxact->pgprocno));
```

并且这个 dummy entry 必须在原 backend 清除自己的 XID 之前进入 ProcArray。源码注释说明这样会有一个短窗口同一个 XID 在 ProcArray 中出现两次，这是可以接受的；反过来如果出现一个窗口里该 XID 完全不在 ProcArray 中，别人就可能把它当作 crash / not running，破坏 correctness。

对于本节来说，prepared transaction 是一个重要诊断点：

```text
没有 active SQL query，也可能有 prepared transaction 钉住 cleanup horizon。
```

## 5. 主流程源码 walkthrough

### 5.1 普通 snapshot 如何钉住 cleanup horizon

从一个普通查询开始：

```text
GetTransactionSnapshot()
  -> GetSnapshotData()
     -> 扫描 ProcArray 构造 SnapshotData
     -> 设置 MyProc->xmin = snapshot->xmin
  -> PushActiveSnapshot() 或 RegisterSnapshot()
     -> snapshot 进入 active / registered 生命周期
```

这个 `MyProc->xmin` 是对其它 backend 的承诺：

```text
我可能仍按这个 snapshot 的规则访问 heap tuple；
请不要移除我可能还需要的旧版本。
```

当 snapshot 生命周期结束：

```text
PopActiveSnapshot()
UnregisterSnapshot()
InvalidateCatalogSnapshot()
AtEOXact_Snapshot()
  -> SnapshotResetXmin()
     -> 如果没有剩余 snapshot，清空 MyProc->xmin
     -> 否则把 MyProc->xmin 推进到剩余 snapshot 的最小 xmin
```

注意 `SnapshotResetXmin()` 清空 `MyProc->xmin` 不需要拿 `ProcArrayLock`，源码依赖 XID fetch/store 的原子性。这样的后果是：

```text
ComputeXidHorizons() 的结果是保守下界；
重复调用时 horizon 可能看起来“回退”；
但它不能返回过于激进的值。
```

### 5.2 `ComputeXidHorizons()` 的核心扫描

入口在 `procarray.c`：

```c
static void
ComputeXidHorizons(ComputeXidHorizonsResult *h)
```

第一步，拿 `ProcArrayLock` shared：

```text
LWLockAcquire(ProcArrayLock, LW_SHARED)
```

这与 snapshot-taking 的 interlocking 一致。事务结束要在清除 running XID 和推进 `latestCompletedXid` 时拿 `ProcArrayLock` exclusive。因此当 `ComputeXidHorizons()` 持有 shared lock 时：

```text
没有带 XID 的事务可以从 running set 中消失；
当前扫描看到的 oldest running XID 不会在扫描期间被清掉。
```

第二步，用 `latestCompletedXid + 1` 初始化各 horizon：

```text
initial = latestCompletedXid + 1

oldest_considered_running = initial
shared_oldest_nonremovable = initial
data_oldest_nonremovable = initial
temp_oldest_nonremovable = MyProc->xid 或 initial
```

这个 `initial` 是一个安全下界：如果当前没有任何 active XID / xmin 约束，那么不早于 `latestCompletedXid + 1` 的事务才可能在未来出现。

第三步，在锁内读取 slot 聚合 xmin：

```text
h->slot_xmin = procArray->replication_slot_xmin
h->slot_catalog_xmin = procArray->replication_slot_catalog_xmin
```

源码特意在持锁期间读取 slot horizon，利用 LWLock acquire / release 作为 barrier，保证这个读取与 ProcArray horizon 计算在同一个同步窗口里。

第四步，扫描所有 ProcArray entries：

```c
for (int index = 0; index < arrayP->numProcs; index++)
{
    PGPROC *proc = &allProcs[pgprocno];
    TransactionId xid = UINT32_ACCESS_ONCE(other_xids[index]);
    TransactionId xmin = UINT32_ACCESS_ONCE(proc->xmin);

    xmin = TransactionIdOlder(xmin, xid);
    ...
}
```

这里有三个细节很关键。

第一，`xid` 和 `xmin` 都只读一次。`GetNewTransactionId()` 可以不拿 `ProcArrayLock` 写入 `ProcGlobal->xids[]`，所以读者必须避免多次读取同一字段后假设结果稳定。

第二，取 `TransactionIdOlder(xmin, xid)`。原因是：

```text
一个 backend 可能有 xmin 但还没有 xid；
也可能已经有 xid，但还没有设置 snapshot xmin；
cleanup horizon 必须防住两种窗口。
```

第三，先更新 `oldest_considered_running`，再根据 flags 决定是否影响 tuple nonremovable horizon：

```text
oldest_considered_running:
  包含 lazy VACUUM 和 logical decoding。

shared / data / catalog nonremovable:
  跳过 PROC_IN_VACUUM 和 PROC_IN_LOGICAL_DECODING。
```

这一区分是本节的核心之一。lazy VACUUM 不需要阻止其它表的旧 tuple version 移除，因为它通常不做普通 snapshot-based lookup；但它可能需要继续查 `pg_subtrans` 来解释事务树，所以 `oldest_considered_running` 不能忽略它。

### 5.3 shared / data / catalog horizon 如何分化

扫描每个 backend 时，`shared_oldest_nonremovable` 对所有 database backend 生效：

```text
shared relation 可以被所有 database 看到；
所以 shared horizon 必须取所有 backend 的最小 xmin/xid。
```

普通 data relation 只考虑当前数据库，除非有特殊情况：

```text
proc->databaseId == MyDatabaseId:
  普通同库 backend，需要考虑。

MyDatabaseId == InvalidOid:
  当前 backend 还没绑定数据库，必须保守。

PROC_AFFECTS_ALL_HORIZONS:
  例如 hot standby feedback，不能归属单个 database。

in_recovery:
  recovery 下不能精确做 per-database horizon，统一保守处理。
```

扫描结束后，catalog horizon 才从 data horizon 分化：

```text
catalog_oldest_nonremovable = data_oldest_nonremovable
catalog_oldest_nonremovable = older(catalog_oldest_nonremovable, slot_catalog_xmin)
```

也就是说，普通 data table 受 slot `xmin` 保护；catalog table 额外受 slot `catalog_xmin` 保护。这样 logical decoding 可以保留足够的 catalog 历史，用旧 catalog snapshot 解释 WAL 中的数据变化。

### 5.4 recovery 下 KnownAssignedXids 如何进入 horizon

在 hot standby recovery 中，standby backend 的 snapshot running XID 来源不是 primary 上的 `PGPROC`，而是 WAL replay 维护的 `KnownAssignedXids`。

`ComputeXidHorizons()` 在 recovery 下会读取：

```text
kaxmin = KnownAssignedXidsGetOldestXmin()
```

释放 `ProcArrayLock` 后，把它合并到：

```text
oldest_considered_running
shared_oldest_nonremovable
data_oldest_nonremovable
```

这说明 standby 上的 cleanup horizon 不能只看本地 backend 的 `MyProc->xmin`。WAL 中已经分配但尚未完成的事务，也会影响 standby query 的可见性边界。

### 5.5 replication slot 如何把外部消费者纳入 cleanup 边界

Replication slot 是“没有普通 snapshot 生命周期也需要保留历史”的典型来源。

`ReplicationSlotsComputeRequiredXmin()` 扫描所有 slot：

```text
for each slot:
  if !in_use: skip
  read effective_xmin / effective_catalog_xmin
  if invalidated: skip
  agg_xmin = minimum(effective_xmin)
  agg_catalog_xmin = minimum(effective_catalog_xmin)

ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin)
```

`ProcArraySetReplicationSlotXmin()` 把聚合结果写入 ProcArray：

```c
procArray->replication_slot_xmin = xmin;
procArray->replication_slot_catalog_xmin = catalog_xmin;
```

之后 `ComputeXidHorizons()` 会应用：

```text
shared_oldest_nonremovable = older(shared_oldest_nonremovable, slot_xmin)
data_oldest_nonremovable = older(data_oldest_nonremovable, slot_xmin)

shared_oldest_nonremovable = older(shared_oldest_nonremovable, slot_catalog_xmin)
catalog_oldest_nonremovable = older(catalog_oldest_nonremovable, slot_catalog_xmin)
```

这解释了为什么一个 stale replication slot 可能让 VACUUM 无法移除大量旧版本：

```text
slot 没有 active query；
但 slot 的 effective_xmin / effective_catalog_xmin 仍通过 ProcArray 聚合 horizon 保护旧版本。
```

### 5.6 VACUUM 如何消费 `OldestXmin`

VACUUM 的关键入口在 `vacuum_get_cutoffs()`：

```c
cutoffs->OldestXmin = GetOldestNonRemovableTransactionId(rel);
```

`GetOldestNonRemovableTransactionId(rel)` 内部调用 `ComputeXidHorizons()`，再根据 relation 类型选择：

```text
shared relation 或 rel == NULL:
  shared_oldest_nonremovable

catalog relation 或 logical decoding 可访问 relation:
  catalog_oldest_nonremovable

普通非 temp relation:
  data_oldest_nonremovable

local temp relation:
  temp_oldest_nonremovable
```

`HeapTupleSatisfiesVacuum()` 之后用这个 `OldestXmin` 判断：

```text
如果 tuple 删除事务已提交，但 deleting XID >= OldestXmin:
  HEAPTUPLE_RECENTLY_DEAD
  仍可能被某个 open transaction 看到，不能移除。

如果 deleting XID < OldestXmin:
  HEAPTUPLE_DEAD
  可以被 VACUUM 移除。
```

源码注释直接把这个语义写在 `heapam_visibility.c`：

```text
Tuples deleted by XIDs >= OldestXmin are deemed "recently dead";
they might still be visible to some open transaction.
```

所以 `OldestXmin` 不是“最老活跃事务”这么泛泛的概念，而是 VACUUM 的删除安全线。

### 5.7 lazy VACUUM 为什么先设置 `PROC_IN_VACUUM`

`vacuum_rel()` 在 non-FULL lazy VACUUM 中会先设置 flags：

```c
LWLockAcquire(ProcArrayLock, LW_EXCLUSIVE);
MyProc->statusFlags |= PROC_IN_VACUUM;
if (params.is_wraparound)
    MyProc->statusFlags |= PROC_VACUUM_FOR_WRAPAROUND;
ProcGlobal->statusFlags[MyProc->pgxactoff] = MyProc->statusFlags;
LWLockRelease(ProcArrayLock);
```

然后才：

```c
PushActiveSnapshot(GetTransactionSnapshot());
```

顺序很重要：

```text
先让其它 backend 知道“我的 xmin 属于 lazy VACUUM”；
再获取自己的 snapshot 并设置 MyProc->xmin。
```

如果反过来，lazy VACUUM 的 `xmin` 可能先暴露出来，短时间内被其它 VACUUM 当成普通 query 的 xmin，从而让 cleanup horizon 不必要地后退。

为什么 FULL VACUUM 不设置这个 flag？源码注释给出原因：FULL VACUUM 可能执行 functional index 中的用户定义函数，这些函数如果访问其它表，就需要普通 snapshot 语义保护，不能被其它 VACUUM 忽略。

### 5.8 pruning 与 `GlobalVisTest*`

on-access pruning 不会像 VACUUM 那样先计算一整套 `VacuumCutoffs`，因为它发生在普通 heap page 访问路径上，必须快速退出。

`heap_page_prune_opt()` 先看 page header 中的 `prune_xid`：

```text
没有 prune_xid:
  直接返回，不求 horizon。

有 prune_xid:
  state = GlobalVisTestFor(relation)
  if !GlobalVisTestIsRemovableXid(state, prune_xid, true):
      返回
```

只有当全局近似边界认为存在可剪枝机会时，才尝试拿 buffer cleanup lock 并进入真正 page pruning。

`heap_prune_satisfies_vacuum()` 也会组合两类边界：

```text
如果 VACUUM 提供了 OldestXmin，并且 dead_after < OldestXmin:
  必须认为 DEAD。

否则用 GlobalVisTestIsRemovableXid() 看当前 GlobalVisState 是否已经能证明可移除。
```

lazy VACUUM 初始化时也会：

```c
vacrel->aggressive = vacuum_get_cutoffs(rel, params, &vacrel->cutoffs);
vacrel->vistest = GlobalVisTestFor(rel);
```

注释强调了一个细节：

```text
OldestXmin 和 vistest 必须组合使用；
heap_page_prune_and_freeze() 不能在“该 freeze 还是该 remove”上产生矛盾。
```

### 5.9 all-visible 判断也需要 horizon

cleanup horizon 不只影响“删不删 dead tuple”，还影响 visibility map。

`vacuumlazy.c` 在处理页面 live tuples 后，会检查页面上最新的 live tuple `xmin`：

```text
如果 newest_live_xid 仍可能被某个 snapshot 认为 running:
  页面不能标记 all-visible
```

这是因为 all-visible 的语义是：

```text
所有可能的 snapshot 都能看到该页面上的所有 tuple。
```

如果一个 live tuple 的插入事务仍可能被某个 snapshot 认为 running，那么这个 tuple 对那个 snapshot 不可见，页面自然不能 all-visible。

这说明 horizon 的用途比 VACUUM 删除更宽：

```text
它也是“是否对所有观察者稳定可见”的边界。
```

## 6. 生命周期 / ownership / cleanup

### active snapshot 的生命周期

创建者：

```text
GetTransactionSnapshot()
GetLatestSnapshot()
GetCatalogSnapshot()
SetTransactionSnapshot()
```

持有者：

```text
ActiveSnapshot stack
RegisteredSnapshots heap
CatalogSnapshot
FirstXactSnapshot
```

发布状态：

```text
MyProc->xmin
TransactionXmin
RecentXmin
```

释放路径：

```text
PopActiveSnapshot()
UnregisterSnapshot()
InvalidateCatalogSnapshot()
AtEOXact_Snapshot()
ProcArrayEndTransaction()
```

`AtEOXact_Snapshot()` 注释说明，正常 commit 路径中 `ProcArrayEndTransaction()` 会先 reset `MyProc->xmin`，所以 snapshot cleanup 不需要重复处理。ERROR / abort 路径会清理 active snapshot stack 和 registered snapshots，并通过事务 cleanup 兜底。

### VACUUM flags 的生命周期

lazy VACUUM 在事务中设置：

```text
PROC_IN_VACUUM
PROC_VACUUM_FOR_WRAPAROUND 可选
```

这些 flags 保持到 `CommitTransaction` 或 `AbortTransaction`。源码注释说明不要提前清掉：

```text
如果先清 flag，再 reset MyProc->xid/xmin，
GetOldestNonRemovableTransactionId() 可能看起来后退。
```

这是一类典型的 shared-state cleanup 顺序问题：

```text
先保证其它 backend 对这个 xmin 的解释仍正确；
再撤销影响 horizon 的字段。
```

### replication slot xmin 的生命周期

slot 自己的状态在 replication slot shared memory 中，关键字段包括：

```text
data.xmin
data.catalog_xmin
effective_xmin
effective_catalog_xmin
candidate_catalog_xmin
```

逻辑 slot 尤其强调 `effective_xmin` / `effective_catalog_xmin` 的持久性语义：这些值可能代表已经安全持久化的 xmin，避免 crash 后 logical decoding 缺失所需历史。

聚合状态在 ProcArray 中：

```text
replication_slot_xmin
replication_slot_catalog_xmin
```

当 slot 推进、释放、invalidated、复制确认位置变化或 slot sync 更新时，replication subsystem 会重新计算聚合 xmin。VACUUM 不直接扫描每个 slot，而是通过 ProcArray 聚合结果感知 slot 对 cleanup 的约束。

### prepared transaction 的生命周期

创建：

```text
PREPARE TRANSACTION
  -> MarkAsPreparing()
  -> 把锁等资源转移到 dummy PGPROC
  -> MarkAsPrepared()
  -> ProcArrayAdd(dummy PGPROC)
```

持有：

```text
TwoPhaseState->prepXacts
dummy PGPROC
ProcArray entry
2PC state file / WAL record
```

释放：

```text
COMMIT PREPARED / ROLLBACK PREPARED
  -> 从 ProcArray 移除 dummy PGPROC
  -> RemoveGXact()
  -> 释放锁和 2PC 状态
```

只要 prepared transaction 留在 ProcArray 中，它就会像 running transaction 一样影响 `ComputeXidHorizons()`，因此也可能钉住 VACUUM cleanup。

### recovery KnownAssignedXids 的生命周期

在 standby recovery 中：

```text
RecordKnownAssignedTransactionIds()
  -> WAL replay 观察到新 XID，加入 KnownAssignedXids

ExpireTreeKnownAssignedTransactionIds()
ExpireOldKnownAssignedTransactionIds()
ExpireAllKnownAssignedTransactionIds()
  -> WAL replay 观察到事务结束或边界推进，移除 KnownAssignedXids
```

`KnownAssignedXidsGetOldestXmin()` 给 `ComputeXidHorizons()` 一个 recovery 视角的最老可能 running XID。这个状态由 startup process 维护，普通 standby backend 消费。

## 7. 正确性机制层次

### 7.1 `ProcArrayLock` 与 running set 不缩小

`ComputeXidHorizons()` 和 `GetSnapshotData()` 都拿 `ProcArrayLock` shared。事务结束清除 XID 时拿 exclusive。

这保证了：

```text
在 horizon 计算期间，带 XID 的事务不会从 running set 消失。
```

如果没有这个保证，`ComputeXidHorizons()` 可能错过正在结束但仍会影响并发 snapshot 的事务，从而给出过于新的 cleanup horizon。

### 7.2 为什么 active XID 也要参与 xmin 计算

`transam/README` 对这个点解释得很细：

```text
某个 backend 可能还没有设置 MyProc->xmin，但它已经有 active XID；
并发 GetSnapshotData() 稍后可能会基于这个 active XID 算出更老的 snapshot xmin。
```

所以 `ComputeXidHorizons()` 不能只取 `proc->xmin` 的最小值，还要把 active `xid` 放进 MIN calculation。

这条规则防止这样的 race：

```text
T1 已分配 XID，但还没设置 snapshot xmin
T2 正在 ComputeXidHorizons()
如果 T2 只看 xmin，可能认为 horizon 可以推进到更后
随后 T1 设置 snapshot，得到一个更老 xmin
VACUUM 已经删掉 T1 snapshot 需要的旧版本
```

把 active XID 纳入计算后，T2 的 horizon 不会超过并发 snapshot 可能计算出的最小 running XID。

### 7.3 为什么 `latestCompletedXid + 1` 是安全初值

当没有 active XID / xmin 时，`ComputeXidHorizons()` 使用：

```text
latestCompletedXid + 1
```

这是因为 `GetNewTransactionId()` 必须在释放 `XidGenLock` 前把新 XID 存入 ProcArray。结合事务结束更新 `latestCompletedXid` 的 exclusive lock 规则，可以保证：

```text
小于等于 latestCompletedXid 的 XID 要么已经完成，要么如果仍 running 就必须在 ProcArray 中可见。
```

因此当 ProcArray scan 没有看到更老约束时，用 `latestCompletedXid + 1` 初始化不会过于激进。

### 7.4 relation kind 是 correctness 边界，不只是优化

`GlobalVisHorizonKindForRel(rel)` 根据 relation 类型选择：

```text
rel == NULL / relisshared / recovery:
  VISHORIZON_SHARED

catalog relation 或 logical decoding 可访问 relation:
  VISHORIZON_CATALOG

非 local 普通 relation:
  VISHORIZON_DATA

local temp relation:
  VISHORIZON_TEMP
```

这不仅是性能优化。对于 shared relation，如果忽略其它数据库 backend，可能删除仍被其它数据库 snapshot 需要的 tuple。对于普通 relation，如果总是使用 shared horizon，则会让无关数据库的长事务拖住本数据库 VACUUM。

### 7.5 slot `catalog_xmin` 与 logical decoding correctness

Logical decoding 不只是需要 heap row version，还需要能用历史 catalog 解释 WAL：

```text
当 WAL 中出现某个 relation / column / type / index 相关变化时，
decoder 需要在当时的 catalog 语义下解析。
```

因此 slot 有两个边界：

```text
xmin:
  保护 data tuples。

catalog_xmin:
  额外保护 catalog tuples。
```

`catalog_oldest_nonremovable` 会应用 `slot_catalog_xmin`，普通 `data_oldest_nonremovable` 不应用它。这就是 catalog horizon 和 data horizon 分离的主要原因。

### 7.6 保守和回退是允许的，过于激进是不允许的

源码注释明确说，`ComputeXidHorizons()` 的结果可能在重复调用之间后退。原因包括：

```text
XID-less transaction 可以异步清空 MyProc->xmin；
新事务开始后 snapshot xmin 可能把其它 database 的更老 running XID 纳入；
walsender 的 xmin 可能基于 standby replay 状态；
slot 聚合 xmin 可能由于并发计算出现保守回退。
```

这些都可以接受，因为后果是：

```text
少删一些、晚删一些、少推进一些。
```

不能接受的是过于激进：

```text
把仍被合法 snapshot / decoder / standby 需要的旧版本删除。
```

## 8. 错误路径 / 异常路径 / fallback

### GlobalVis 的不确定区间 fallback

`GlobalVisTestIsRemovableFullXid()` 的判断顺序：

```text
fxid < maybe_needed:
  return true

fxid >= definitely_needed:
  return false

中间区间:
  如果 allow_update 且应该更新:
      GlobalVisUpdate()
      重新比较 maybe_needed
  否则:
      return false
```

这是一种典型的 safe fallback：

```text
不确定时返回“不可移除”。
```

错误结果最多导致 pruning/VACUUM 少做一些工作，不会删除仍需要的版本。

### imported snapshot 的 xmin 安装失败

`SetTransactionSnapshot()` 导入 snapshot 时，先调用 `GetSnapshotData()` 更新 GlobalVis state，再把 source snapshot 内容拷贝过来。之后必须通过：

```text
ProcArrayInstallRestoredXmin()
或
ProcArrayInstallImportedXmin()
```

把导入的 `xmin` 安装到当前 `PGPROC`。如果 source transaction 已经不再 running，就报错：

```text
could not import the requested snapshot
The source transaction is not running anymore.
```

这里的 correctness 点是：

```text
不能让一个没有合法来源的旧 snapshot 随意把全局 cleanup horizon 往回拉。
```

### hot standby 没有 slot 时的边界

`ComputeXidHorizons()` 注释提到：如果 walsender 根据 standby 状态设置 xmin，horizon 可能后退；如果 standby 没有 replication slot 持久保护，那么 primary 仍可能清掉 standby query 想要的数据。Hot Standby 的处理方式不是让 primary 无条件保留所有历史，而是在 standby 上取消需要访问已移除数据的查询。

因此：

```text
hot_standby_feedback 是动态反馈；
replication slot 是更持久的保护；
两者都可能带来 primary bloat 风险。
```

### stale replication slot

如果 logical / physical slot 长时间不推进：

```text
effective_xmin 或 effective_catalog_xmin 保持很旧
  -> ProcArray aggregation 保持很旧
  -> GetOldestNonRemovableTransactionId() 返回很旧 horizon
  -> VACUUM 只能把大量 tuple 判为 RECENTLY_DEAD
  -> heap bloat、catalog bloat、wraparound warning 风险上升
```

这不是 VACUUM “没工作”，而是 cleanup correctness 边界被外部消费者钉住。

### prepared transaction 长期存在

prepared transaction 没有普通 `pg_stat_activity` query，却有 dummy `PGPROC` 留在 ProcArray。它可能导致：

```text
OldestXmin 长期不推进；
VACUUM WARNING 提示 commit or roll back old prepared transactions；
database drop 等操作看到 prepared transaction 仍在使用数据库。
```

诊断时必须同时查 `pg_prepared_xacts`，不能只看活跃会话。

## 9. 成本、资源与跨模块传播

### `ComputeXidHorizons()` 的成本

`ComputeXidHorizons()` 至少包含：

```text
一次 ProcArrayLock shared
一次 ProcArray linear scan
一次 slot horizon 聚合读取
recovery 下一次 KnownAssignedXids oldest 读取
若由 GlobalVisUpdate 触发，还会更新多个 GlobalVisState
```

复杂度主要随：

```text
MaxBackends / 当前 ProcArray numProcs
replication slot 数量
KnownAssignedXids 数量
调用频率
ProcArrayLock 竞争
```

这解释了为什么 `GetSnapshotData()` 不再每次精确计算 oldest-xmin，也解释了为什么 pruning 先用 page `prune_xid` 快速过滤，再按需调用 `GlobalVisTest*`。

### VACUUM 中的 horizon 传播

`vacuum_get_cutoffs()` 得到的 `OldestXmin` 会传播到：

```text
HeapTupleSatisfiesVacuum()
heap_page_prune_and_freeze()
FreezeLimit clamp
NewRelfrozenXid 初始值
all-visible / all-frozen 判断
wraparound warning
```

这里有一个重要关系：

```text
FreezeLimit 必须 <= OldestXmin。
```

如果 freeze cutoff 比 cleanup horizon 更新，VACUUM 可能试图冻结仍可能被某些 snapshot 解释为 running 的事务，这会破坏 visibility 语义。

### bloat 的跨模块传播

一个旧 horizon 会传播到多个层面：

```text
heap:
  dead tuple 不能移除，HOT chain 变长。

index:
  heap TID 仍可能需要，index cleanup 收益下降。

visibility map:
  页面不能 all-visible，index-only scan 命中率下降。

catalog:
  catalog tuple 保留过多，planning 和 cache invalidation 压力上升。

pg_xact / pg_subtrans:
  事务状态文件截断边界受 oldest_considered_running 影响。

autovacuum:
  freeze / wraparound pressure 上升，可能出现 warning 或更 aggressive 的 vacuum。
```

### relation-sensitive horizon 的性能价值

如果所有 relation 都用 shared horizon：

```text
数据库 A 的长事务会拖住数据库 B 普通表 VACUUM；
logical decoding 的 catalog_xmin 会拖住所有普通 data table；
temp table cleanup 也会被其它 backend 影响。
```

当前代码通过 shared / catalog / data / temp 拆分，把不必要的保守性限制在最小范围内。

这是一种常见内核设计：

```text
全局 correctness 边界先保守建模；
再按对象可见范围逐层收窄。
```

## 10. 观测与诊断入口

### SQL 侧可见状态

查 backend 发布的 XID / xmin：

```sql
SELECT pid, datname, state, backend_xid, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xid IS NOT NULL OR backend_xmin IS NOT NULL
ORDER BY backend_xmin NULLS LAST, xact_start NULLS LAST;
```

查 replication slot horizon：

```sql
SELECT slot_name, slot_type, active, active_pid, xmin, catalog_xmin,
       restart_lsn, confirmed_flush_lsn, wal_status, invalidation_reason
FROM pg_replication_slots
ORDER BY xmin NULLS LAST, catalog_xmin NULLS LAST;
```

查 prepared transactions：

```sql
SELECT gid, prepared, owner, database, transaction
FROM pg_prepared_xacts
ORDER BY prepared;
```

估算 relation wraparound / freeze pressure：

```sql
SELECT n.nspname, c.relname, c.relfrozenxid, age(c.relfrozenxid) AS xid_age
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind IN ('r', 'm', 't')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;
```

### 日志和 warning

`vacuum_get_cutoffs()` 在 `OldestXmin` 过旧时会 warning：

```text
cutoff for removing and freezing tuples is far in the past
Close open transactions soon to avoid wraparound problems.
You might also need to commit or roll back old prepared transactions, or drop stale replication slots.
```

这个 hint 本身就列出了本节三类主要来源：

```text
open transactions
prepared transactions
stale replication slots
```

VACUUM VERBOSE 可以观察 dead / recently dead 行为，但要记住：

```text
看到 dead tuples 没有被移除，不一定是 VACUUM bug；
先查 horizon 是否被钉住。
```

### gdb / 断点入口

建议断点：

```text
ComputeXidHorizons
GetOldestNonRemovableTransactionId
GlobalVisTestIsRemovableXid
vacuum_get_cutoffs
HeapTupleSatisfiesVacuum
heap_page_prune_opt
ReplicationSlotsComputeRequiredXmin
ProcArraySetReplicationSlotXmin
SnapshotResetXmin
```

可观察变量：

```text
h->oldest_considered_running
h->shared_oldest_nonremovable
h->catalog_oldest_nonremovable
h->data_oldest_nonremovable
h->slot_xmin
h->slot_catalog_xmin
MyProc->xmin
MyProc->statusFlags
ProcGlobal->statusFlags[MyProc->pgxactoff]
```

### 源码级定位问题的问法

遇到 “VACUUM 为什么不回收” 时，按这个顺序问：

```text
1. 被删除 tuple 的 deleting XID 是否已提交？
2. deleting XID 是否 < 当前 relation 的 OldestXmin？
3. OldestXmin 是 shared / catalog / data / temp 哪一种？
4. 是哪个 backend_xmin、prepared xact、slot xmin、catalog_xmin 或 recovery state 把它钉住？
5. GlobalVis 不确定区间是否导致 pruning 保守退出？
6. VACUUM 是因为 horizon 不允许，还是因为 page skipping / cleanup lock / index cleanup 等其它原因没有处理？
```

## 11. 常见误区

### 误区一：`xmin` 就是当前事务 ID

`backend_xmin` / `MyProc->xmin` 不是当前事务 ID。它是该 backend 当前仍需要保护的最老 snapshot xmin。当前事务 ID 是 `backend_xid` / `MyProc->xid`。

一个 read-only transaction 可以没有 XID，但持有 snapshot，从而有 `backend_xmin` 并影响 VACUUM。

### 误区二：tuple 的 deleting transaction 已提交就一定能删

已提交只说明“删除动作真实发生”。能否物理移除还要看：

```text
deleting XID < relation-specific OldestXmin
```

如果 deleting XID 仍可能被某个 snapshot 视为 running，该 tuple 对那个 snapshot 可能仍是可见旧版本。

### 误区三：VACUUM 本身的 snapshot 一定会拖住其它 VACUUM

lazy VACUUM 会设置 `PROC_IN_VACUUM`，其它 VACUUM 在 tuple nonremovable horizon 中会忽略它。但它仍参与 `oldest_considered_running`，因为 `pg_subtrans` 访问需要保护。

FULL VACUUM 不走这个 lazy VACUUM 简化语义。

### 误区四：replication slot 只保留 WAL

slot 也可能通过 `xmin` / `catalog_xmin` 保留 heap 或 catalog row versions。`restart_lsn` 保护 WAL；`xmin` / `catalog_xmin` 保护 MVCC 历史。

只看 WAL retained size 不足以解释 heap/catalog bloat。

### 误区五：`GetSnapshotData()` 应该顺便给 VACUUM 最精确 OldestXmin

这正是 PostgreSQL 后来拆开的点。snapshot 内容依赖 running XID；cleanup horizon 依赖 backend xmin。让高频 snapshot 获取读取频繁变化的 xmin，会增加 cacheline 竞争。现在通过 GlobalVis 近似边界和按需 `ComputeXidHorizons()` 分摊成本。

### 误区六：horizon 应该单调前进

源码明确允许重复计算的 horizon 看起来后退。只要它是保守下界，就不会破坏 correctness。系统更在乎“不误删”，而不是每次报告的数值都单调。

### 误区七：只查 `pg_stat_activity` 就能找到所有 blocker

不够。prepared transaction 和 replication slot 都可能没有普通 active query。hot standby feedback / walsender、logical decoding、slot sync 也可能通过不同路径影响 horizon。

诊断 cleanup 问题至少同时看：

```text
pg_stat_activity.backend_xmin
pg_replication_slots.xmin / catalog_xmin
pg_prepared_xacts
VACUUM warning / verbose output
```

## 12. 课堂实验

### 实验一：长事务如何阻止 VACUUM 移除旧版本

Session A：

```sql
CREATE TABLE horizon_demo(id int primary key, v text);
INSERT INTO horizon_demo
SELECT g, 'old' FROM generate_series(1, 10000) g;

BEGIN;
SELECT count(*) FROM horizon_demo;
```

保持 Session A 不提交。

Session B：

```sql
UPDATE horizon_demo SET v = 'new';
VACUUM (VERBOSE) horizon_demo;

SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

观察：

```text
Session A 的 backend_xmin 存在；
VACUUM 可能把许多旧版本判为 recently dead，而不是可移除；
提交 Session A 后再次 VACUUM，cleanup horizon 可以推进。
```

源码回扣：

```text
Session A:
  GetSnapshotData() 设置 MyProc->xmin

Session B VACUUM:
  vacuum_get_cutoffs()
    -> GetOldestNonRemovableTransactionId()
      -> ComputeXidHorizons()
        -> 读取 Session A 的 proc->xmin
```

### 实验二：read-only transaction 没有 XID 也能钉住 horizon

Session A：

```sql
BEGIN;
SELECT count(*) FROM horizon_demo;

SELECT txid_current_if_assigned();
```

如果返回 `NULL`，说明当前事务尚未分配 XID。

Session B：

```sql
SELECT pid, backend_xid, backend_xmin, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL;
```

观察：

```text
backend_xid 可以为空；
backend_xmin 仍然存在；
VACUUM 仍必须尊重这个 snapshot。
```

源码回扣：

```text
ComputeXidHorizons() 同时读取 proc->xmin 和 ProcGlobal->xids[index]；
只有 xmin、没有 xid 的 backend 仍影响 horizon。
```

### 实验三：prepared transaction 如何隐藏在普通 activity 之外

前提：测试实例启用了 `max_prepared_transactions > 0`。

Session A：

```sql
BEGIN;
INSERT INTO horizon_demo VALUES (-1, 'prepared');
PREPARE TRANSACTION 'horizon-prepared-1';
```

Session B：

```sql
SELECT * FROM pg_prepared_xacts;

UPDATE horizon_demo SET v = 'again' WHERE id > 0;
VACUUM (VERBOSE) horizon_demo;
```

观察：

```text
没有原来的 active backend；
但 prepared transaction 仍存在；
它通过 dummy PGPROC 留在 ProcArray 中。
```

清理：

```sql
ROLLBACK PREPARED 'horizon-prepared-1';
```

源码回扣：

```text
twophase.c:
  MarkAsPrepared()
    -> ProcArrayAdd(dummy PGPROC)
```

### 实验四：replication slot 的 xmin / catalog_xmin

在允许创建 logical slot 的测试实例上：

```sql
SELECT * FROM pg_create_logical_replication_slot('horizon_slot', 'test_decoding');

SELECT slot_name, active, xmin, catalog_xmin, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'horizon_slot';
```

产生数据变化后，不消费 slot，再观察：

```sql
SELECT slot_name, xmin, catalog_xmin, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'horizon_slot';
```

清理：

```sql
SELECT pg_drop_replication_slot('horizon_slot');
```

源码回扣：

```text
slot.c:
  ReplicationSlotsComputeRequiredXmin()
    -> ProcArraySetReplicationSlotXmin()

procarray.c:
  ComputeXidHorizons()
    -> slot_xmin / slot_catalog_xmin 参与 shared / data / catalog horizon
```

### 实验五：on-access pruning 的快速退出

思路：

```text
1. 找一个发生过 UPDATE / DELETE 的 heap page。
2. 在 heap_page_prune_opt() 和 GlobalVisTestIsRemovableXid() 下断点。
3. 分别在有长事务和无长事务时访问同一 relation。
4. 观察 prune_xid 先过滤，再由 GlobalVisTest* 决定是否继续。
```

gdb 断点：

```text
break heap_page_prune_opt
break GlobalVisTestIsRemovableXid
break ComputeXidHorizons
```

观察重点：

```text
有些页面没有 prune_xid，根本不会求 horizon；
有些 XID 在 maybe_needed / definitely_needed 中间，会触发 GlobalVisUpdate；
不确定时 pruning 保守返回。
```

## 13. 讨论题

1. 为什么 `ComputeXidHorizons()` 必须同时考虑 `proc->xmin` 和 active `xid`？如果只看 `xmin`，哪个 race 会导致 VACUUM 误删？

2. lazy VACUUM 为什么可以被其它 VACUUM 在 tuple cleanup horizon 中忽略，但不能在 `oldest_considered_running` 中忽略？

3. 为什么 logical decoding 需要 `catalog_xmin`，而普通 data table horizon 不应该被所有 catalog history 无条件拖住？

4. 如果所有 relation 都使用 `shared_oldest_nonremovable`，正确性会怎样？性能和 bloat 会怎样？

5. 为什么 `GlobalVisTestIsRemovableXid()` 在不确定时返回 false，而不是强行调用精确计算或乐观认为可删除？

6. 为什么 stale replication slot 会导致 heap / catalog bloat，而不仅仅是 WAL 保留？

7. prepared transaction 没有 active backend，为什么仍必须留在 ProcArray 中？

8. `ComputeXidHorizons()` 的结果为什么允许在重复调用之间后退？这会不会破坏 VACUUM correctness？

9. `GetSnapshotData()` 不再精确计算 oldest xmin 后，哪些路径承担了精确 cleanup horizon 的责任？

10. 如果你要定位一次 “VACUUM VERBOSE 显示 dead tuples 但空间不回收” 的问题，应该按什么顺序排除 active snapshot、prepared transaction、slot、page skipping 和 cleanup lock？

## 14. 本节小结

本节的主线是：

```text
旧 tuple version 能否移除，不由当前查询是否可见决定；
而由所有可能观察者的最低需求共同决定。
```

核心链路：

```text
GetSnapshotData()
  -> 发布 MyProc->xmin

ReplicationSlotsComputeRequiredXmin()
  -> 发布 replication_slot_xmin / replication_slot_catalog_xmin

KnownAssignedXids
  -> 发布 recovery 视角下仍可能 running 的 XID

prepared transaction
  -> dummy PGPROC 留在 ProcArray

ComputeXidHorizons()
  -> 汇总 shared / catalog / data / temp / oldest-considered-running horizon

VACUUM / pruning / visibility map
  -> 根据 horizon 决定 tuple 是否 DEAD、page 是否可剪枝、页面是否 all-visible、事务状态是否还能截断
```

本节最重要的工程规律是：

```text
cleanup 边界必须以“最保守的仍可能观察者”为准；
但为了可扩展性，系统会把观察者按对象可见范围和使用场景拆分成多个 horizon，并在高频路径上使用可证明安全的近似边界。
```

读源码时要特别警惕三类混淆：

```text
MyProc->xid != MyProc->xmin；
committed != removable；
一个全局 xmin != relation-sensitive cleanup horizon。
```

下一节会继续沿着 ProcArray 生命周期往后走：当 commit、abort、FATAL exit 和 postmaster cleanup 发生时，`PGPROC` / ProcArray 状态如何按顺序清理，避免 stale backend state 被后续进程误读。
