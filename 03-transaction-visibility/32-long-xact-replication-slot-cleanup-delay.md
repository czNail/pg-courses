# PostgreSQL 长事务、prepared transaction、replication slot 与 cleanup delay

## 课程定位

前置知识：已经理解 snapshot xmin、cleanup horizon、dead / recently dead、VACUUM、freeze、visibility map all-visible / all-frozen。

本节唯一主问题：

```text
为什么表里旧版本已经对大多数查询不可见，VACUUM 仍然会因为长事务、prepared transaction、replication slot 或 standby feedback 而长期无法清理？
```

核心矛盾：primary 上的空间回收希望尽快推进 cleanup horizon，但系统必须保护所有仍可能观察旧世界的对象，包括本地长事务、已经 prepare 但未结束的事务、逻辑解码需要的 catalog 历史、物理 standby 上的查询，以及 slot 持久化的 xmin。

学完后应能判断：`backend_xmin`、`pg_prepared_xacts`、`pg_replication_slots.xmin/catalog_xmin`、hot standby feedback、walsender `MyProc->xmin` 如何共同进入 `ComputeXidHorizons()`，以及如何定位 cleanup delay 的真正来源。

本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`；源码阅读顺序放在第 3 节。

## 1. 本节在总主线中的位置

第 26 节讲过：

```text
deleted tuple 是否 removable
  -> 取决于 dead_after 是否早于 cleanup horizon
```

第 28 节讲过：

```text
VACUUM 能清理多少
  -> 取决于 dead tuple、索引清理和 page-level 操作
```

第 29 节讲过：

```text
freeze 能推进多少
  -> 取决于 OldestXmin、FreezeLimit 和页面扫描证明
```

本节把这些线合在一起。

当 cleanup horizon 被外部观察者钉住时，所有后续机制都会受影响。

```text
dead tuple:
  继续 recently dead

VACUUM:
  扫描但不能移除

index cleanup:
  dead item 减少有限

freeze:
  cutoff 不能越过 OldestXmin

visibility map:
  页面可能迟迟不能 all-visible / all-frozen

autovacuum:
  反复运行但效果有限
```

本节不是重复讲 cleanup horizon。

本节聚焦谁会把 horizon 钉住，以及源码里这些对象如何进入同一个计算。

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
ComputeXidHorizons() 扫 ProcArray 中的 PGPROC.xid/xmin、prepared transaction 的 dummy PGPROC、replication slot 聚合出的 xmin/catalog_xmin，以及 recovery/standby 相关 horizon；
GetOldestNonRemovableTransactionId() 按 relation kind 选择最保守的可清理边界；
VACUUM、pruning 和 freeze 都只能在这个边界之后推进。
```

本节核心矛盾是：

```text
本地 heap cleanup 希望按 primary 当前负载尽快回收
  vs
MVCC、2PC、logical decoding 和 standby query 都可能要求 primary 保留旧版本
```

这些对象看起来不同。

长事务是本地 backend。

prepared transaction 没有普通 backend pid。

replication slot 是持久复制状态。

hot standby feedback 来自另一台机器。

但它们进入 cleanup 判断时，都会变成同一种东西：

```text
一个不能越过的 xmin / catalog_xmin horizon
```

这就是本节的统一模型。

## 3. 核心文件分工与阅读顺序

| 阅读顺序 | 文件 | 作用 |
| --- | --- | --- |
| 1 | `src/backend/storage/ipc/procarray.c` | 本节主文件。阅读 `ComputeXidHorizons()`、`GetOldestNonRemovableTransactionId()`、`ProcArrayGetReplicationSlotXmin()`、`GetReplicationHorizons()`。 |
| 2 | `src/include/storage/proc.h` | 对照 `PGPROC.xid`、`xmin`、`statusFlags`、databaseId 等字段。 |
| 3 | `src/backend/utils/time/snapmgr.c` | active / registered snapshot 如何让 backend 暴露 xmin。 |
| 4 | `src/backend/access/transam/twophase.c` | `MarkAsPreparing()`、dummy `PGPROC`、`ProcArrayAdd()`、`ProcArrayRemove()`。 |
| 5 | `src/backend/replication/slot.c` | `ReplicationSlotsComputeRequiredXmin()` 聚合 slot 的 `effective_xmin` / `effective_catalog_xmin`。 |
| 6 | `src/backend/replication/walreceiver.c` | `XLogWalRcvSendHSFeedback()` 在 standby 侧计算并发送 `xmin/catalog_xmin`。 |
| 7 | `src/backend/replication/walsender.c` | `ProcessStandbyHSFeedbackMessage()` 在 primary 侧把 feedback 应用到 walsender PGPROC 或 physical slot。 |
| 8 | `src/backend/commands/vacuum.c` | `vacuum_get_cutoffs()` 消费 cleanup horizon 并发出 warning。 |
| 9 | `src/backend/access/heap/vacuumlazy.c` | VACUUM verbose、failsafe、dead but not yet removable 的运行反馈。 |

阅读顺序要先读 `ComputeXidHorizons()`。

它是所有现象的汇合点。

不要先从 autovacuum 参数入手。

autovacuum 只是执行者。

horizon 被谁钉住，才是根因。

## 4. 一个 runtime 现象先定锚

最小复现仍然是长事务。

Session A：

```sql
DROP TABLE IF EXISTS cleanup_delay_demo;
CREATE TABLE cleanup_delay_demo(id int primary key, payload text);

INSERT INTO cleanup_delay_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 100000) AS g;

BEGIN;
SELECT count(*) FROM cleanup_delay_demo;
-- 保持事务打开。
```

Session B：

```sql
DELETE FROM cleanup_delay_demo WHERE id <= 80000;
VACUUM (VERBOSE) cleanup_delay_demo;
```

观察：

```sql
SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

你会看到 Session A 的 `backend_xmin`。

VACUUM verbose 中可能出现大量：

```text
dead but not yet removable
```

现在让 Session A：

```sql
COMMIT;
```

再执行：

```sql
VACUUM (VERBOSE) cleanup_delay_demo;
```

清理能力会明显变化。

这只是最简单版本。

真实系统中钉住 horizon 的对象可能没有普通 SQL session 那么明显。

它可能是：

```text
prepared transaction
logical replication slot
physical replication slot + hot standby feedback
standby 上长查询
logical decoding 消费滞后
```

本节后面把这些对象都映射到源码。

## 5. `ComputeXidHorizons()` 的统一模型

`ComputeXidHorizons()` 会计算多个 horizon。

它不是只返回一个全局 xmin。

原因是不同 relation 需要不同保护范围。

普通数据表可以忽略其他数据库的普通 backend。

shared relation 要考虑所有数据库。

catalog relation 要考虑 logical decoding 的 `catalog_xmin`。

temporary relation 可以更激进。

因此源码中会产生类似这些结果：

```text
oldest_considered_running
shared_oldest_nonremovable
data_oldest_nonremovable
catalog_oldest_nonremovable
temp_oldest_nonremovable
slot_xmin
slot_catalog_xmin
```

核心循环扫描 ProcArray。

对每个 `PGPROC`，取：

```text
proc xid
proc xmin
statusFlags
databaseId
```

然后合并到不同 horizon。

重点是这一句模型：

```text
xmin = older(proc->xmin, proc->xid)
```

一个 backend 可能还没设置 snapshot xmin，但已经有 xid。

一个 backend 也可能有 snapshot xmin，但还没分配 xid。

两者都可能保护旧版本。

所以计算必须同时看。

之后，slot horizon 也会加入。

```text
replication_slot_xmin
replication_slot_catalog_xmin
```

这些值不是普通 backend 的 `PGPROC.xmin`。

它们由 replication slot 子系统聚合后写进 ProcArray。

最后 `GetOldestNonRemovableTransactionId(rel)` 根据 relation kind 选择一个 horizon。

VACUUM 不需要知道所有来源。

它只拿到“这张 relation 能安全清到哪里”。

## 6. 长事务：最直接的 horizon pin

长事务通过 backend 的 `PGPROC.xmin` 影响 horizon。

当一个 backend 获取 snapshot，并且 snapshot 需要注册到事务级生命周期时，`MyProc->xmin` 会暴露给 ProcArray。

其他 backend 计算 horizon 时会看到它。

运行时表现：

```sql
SELECT pid, state, backend_xmin, xact_start, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

如果某个 backend_xmin 很老，VACUUM 的 removable cutoff 就可能被钉住。

长事务不一定一直 active。

它可以是：

```text
idle in transaction
长时间 cursor
长时间 repeatable read transaction
客户端开启事务后不提交
应用连接池泄漏事务
```

长事务的问题不是它占 CPU。

而是它让系统必须保留旧世界。

因此 autovacuum 反复跑也没用。

它不能越过仍公开的 xmin。

## 7. prepared transaction：没有普通 backend 也能钉住 horizon

prepared transaction 更隐蔽。

它已经离开普通 backend 执行流。

但它还没有 commit prepared 或 rollback prepared。

PostgreSQL 不能忘记它。

`twophase.c` 中 `MarkAsPreparing()` 会分配 `GlobalTransaction`。

`MarkAsPreparingGuts()` 初始化一个 dummy `PGPROC`：

```text
proc->xid = xid
proc->pid = 0
proc->databaseId = databaseid
proc->roleId = owner
```

然后 `MarkAsPrepared()` 会调用：

```text
ProcArrayAdd(GetPGProcByNumber(gxact->pgprocno))
```

这让 prepared transaction 进入 ProcArray。

它没有普通 backend pid。

但 horizon 计算仍会把它当成运行中的事务。

直到：

```text
COMMIT PREPARED
ROLLBACK PREPARED
```

对应路径会记录事务结果，并：

```text
ProcArrayRemove(proc, latestXid)
```

运行时观察：

```sql
SELECT gid, prepared, owner, database, transaction
FROM pg_prepared_xacts
ORDER BY prepared;
```

如果这里有很老的 prepared transaction，它可以解释为什么 `pg_stat_activity` 看不到长事务，但 VACUUM horizon 仍然很老。

## 8. replication slot：把 xmin 持久化到复制语义

replication slot 的作用之一是保护复制消费者需要的资源。

对于 cleanup horizon，关键字段是：

```text
xmin
catalog_xmin
```

`slot.c` 中 `ReplicationSlotsComputeRequiredXmin()` 会遍历所有 slot。

对每个 slot 读取：

```text
effective_xmin
effective_catalog_xmin
invalidated
```

然后聚合最老值。

最后调用：

```text
ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin, already_locked)
```

ProcArray 之后就能在 `ComputeXidHorizons()` 中看到 slot horizon。

logical slot 尤其容易影响 catalog cleanup。

逻辑解码需要读历史 catalog tuple，才能解释 WAL 中的变化。

因此 `catalog_xmin` 可能阻止 catalog tuple 被清理。

运行时观察：

```sql
SELECT slot_name,
       slot_type,
       active,
       xmin,
       catalog_xmin,
       restart_lsn,
       confirmed_flush_lsn,
       wal_status,
       safe_wal_size
FROM pg_replication_slots
ORDER BY slot_name;
```

如果 slot inactive 但 `xmin` 或 `catalog_xmin` 很老，它仍可能钉住 cleanup。

slot 不是必须有活跃 walsender 才有影响。

持久 slot 的意义就是跨会话保留需求。

## 9. hot standby feedback：远端查询影响 primary cleanup

physical standby 上的查询也可能需要旧版本。

否则 primary 清掉旧 tuple 后，standby replay 到 cleanup WAL 时，standby query 可能发生 conflict。

如果开启 `hot_standby_feedback`，standby 会把自己的 horizon 发给 primary。

standby 侧入口是：

```text
XLogWalRcvSendHSFeedback()
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

primary 侧 walsender 收到后走：

```text
ProcessStandbyHSFeedbackMessage()
```

如果没有 replication slot，它可能设置：

```text
MyProc->xmin = feedbackXmin 或 feedbackCatalogXmin
```

这样 walsender 的 `PGPROC` 会进入 horizon 计算。

如果使用 physical replication slot，它会调用：

```text
PhysicalReplicationSlotNewXmin()
```

再通过 slot 机制聚合进 ProcArray。

这就是 hot standby feedback 的取舍：

```text
减少 standby query 被 cleanup conflict cancel
  vs
primary 上 dead tuple、catalog tuple、bloat 和 freeze delay 增加
```

## 10. 主流程源码 walkthrough

把四类来源合成一条主流程：

```text
长事务 / prepared xact / slot / standby feedback
  -> 暴露 xmin 或 catalog_xmin
  -> ComputeXidHorizons()
  -> GetOldestNonRemovableTransactionId(rel)
  -> vacuum_get_cutoffs()
  -> HeapTupleSatisfiesVacuum()
  -> lazy_scan_prune()
  -> VACUUM verbose / stats
```

第一步，本地 backend 获取 snapshot。

`MyProc->xmin` 被设置。

ProcArray 可以看到。

第二步，prepared transaction 进入 2PC。

`twophase.c` 创建 dummy PGPROC。

`ProcArrayAdd()` 让它成为 horizon 计算的一部分。

第三步，replication slot 更新 effective xmin。

`ReplicationSlotsComputeRequiredXmin()` 聚合后写入 ProcArray。

第四步，standby feedback 到达 primary。

walsender 更新自己的 `MyProc->xmin`，或更新 physical slot 的 xmin。

第五步，VACUUM 计算 cutoffs。

`vacuum_get_cutoffs()` 调用 `GetOldestNonRemovableTransactionId(rel)`。

如果 cutoff 太老，会 warning。

第六步，heap visibility 判断。

`HeapTupleSatisfiesVacuum()` 看到删除事务提交。

但 `dead_after >= OldestXmin`。

结果保持 `HEAPTUPLE_RECENTLY_DEAD`。

第七步，VACUUM verbose 输出：

```text
tuples are dead but not yet removable
removable cutoff
```

这不是 VACUUM 不努力。

这是 correctness boundary。

## 11. 生命周期 / ownership / cleanup

cleanup delay 的生命周期可以分成六个阶段。

阶段一：旧版本产生。

UPDATE / DELETE 让旧 tuple version 出现。

阶段二：删除事务提交。

旧版本对新 snapshot 不可见。

阶段三：保护对象仍存在。

可能是：

```text
backend_xmin
prepared transaction dummy PGPROC
replication slot xmin
replication slot catalog_xmin
walsender MyProc->xmin
```

阶段四：horizon 被钉住。

`ComputeXidHorizons()` 选择更老的边界。

阶段五：VACUUM 保守。

tuple 维持 recently dead。

freeze cutoff 被限制。

VM all-visible / all-frozen 可能无法建立。

阶段六：保护对象释放或前进。

```text
COMMIT / ROLLBACK
COMMIT PREPARED / ROLLBACK PREPARED
logical consumer confirm flush
slot drop or advance
standby query 结束
feedback xmin 前进
```

horizon 前移后，下一次 VACUUM 才能清理。

ownership 视角：

```text
长事务:
  backend PGPROC owns xmin

prepared transaction:
  dummy PGPROC owns xid until finish prepared

replication slot:
  slot persistent state owns effective_xmin/catalog_xmin

standby feedback:
  walsender PGPROC or physical slot owns upstream-visible xmin
```

它们最终都变成 ProcArray 可见的 horizon 输入。

## 12. 正确性机制层次

第一层是 MVCC snapshot safety。

只要某个 snapshot 可能需要旧版本，primary 不能删除。

第二层是 2PC durability。

prepared transaction 已经承诺可以未来 commit。

它必须像 running transaction 一样保留冲突和可见性边界。

第三层是 logical decoding correctness。

逻辑解码需要历史 catalog 解释 WAL。

`catalog_xmin` 保护 catalog tuple。

第四层是 physical standby query safety。

hot standby feedback 让 primary 保留 standby query 可能需要的旧版本。

第五层是 relation-aware horizon。

普通数据表、catalog、shared relation 选择不同 horizon。

不要把一个全局 xmin 套到所有表。

第六层是 conservative aggregation。

多个来源中取最老的。

任何一个对象不前进，整体 cleanup 都不能越过它。

第七层是 observability boundary。

`pg_stat_activity` 看得到长事务。

`pg_prepared_xacts` 看 prepared。

`pg_replication_slots` 看 slot。

standby feedback 需要结合 walsender、slot 和 standby 侧查询判断。

没有一个视图能单独覆盖全部来源。

## 13. 错误路径 / 异常路径 / fallback

### idle in transaction

这是最常见的本地来源。

应用忘记提交。

backend CPU 很低。

但 `backend_xmin` 长期存在。

### abandoned prepared transaction

prepared transaction 可能因为事务管理器故障遗留。

它没有普通 backend。

只查 `pg_stat_activity` 会漏掉。

### inactive logical slot

logical slot 不活跃也可能保留 `catalog_xmin`。

如果消费端长期停止，catalog bloat 会增长。

### physical slot + hot standby feedback

standby 上长查询通过 feedback 影响 primary。

如果使用 physical slot，这个 xmin 可能进入 slot 状态。

### feedback disabled

关闭 hot standby feedback 后，standby query 可能不再拖住 primary cleanup。

但 standby replay cleanup WAL 时，查询可能被 conflict cancel。

这是明确 trade-off。

### slot invalidated

slot 如果 invalidated，`ReplicationSlotsComputeRequiredXmin()` 会跳过它。

但这通常意味着复制消费者已经失去连续性或需要重建。

### vacuum warning

`vacuum_get_cutoffs()` 如果发现 cutoff 太老，会 warning：

```text
Close open transactions soon...
commit or roll back old prepared transactions...
drop stale replication slots...
```

这个 warning 直接指向本节主题。

## 14. 成本、资源与跨模块传播

cleanup delay 的成本会跨多个指标出现。

| 表现 | 可能来源 |
| --- | --- |
| `n_dead_tup` 高 | long transaction、slot、prepared xact、standby feedback。 |
| VACUUM verbose 中 dead but not yet removable 多 | OldestXmin 被钉住。 |
| catalog 膨胀 | logical slot `catalog_xmin` 老。 |
| relfrozenxid age 增长 | freeze cutoff 被钉住或 VACUUM 跟不上。 |
| index bloat | dead heap TID 不能完成索引清理。 |
| all-visible / all-frozen 比例低 | 页面保留 recently dead 或未冻结 tuple。 |
| standby 查询少被 cancel | 可能是 hot standby feedback 保护 primary 保留旧版本。 |
| primary 存储增长 | 同一个 feedback 造成保留成本。 |

跨模块传播：

```text
snapmgr.c:
  active / registered snapshot 让 backend 暴露 xmin

twophase.c:
  prepared xact 通过 dummy PGPROC 进入 ProcArray

slot.c:
  slots 聚合 xmin / catalog_xmin

walreceiver.c:
  standby 计算 feedback xmin

walsender.c:
  primary 接收 feedback 并设置 PGPROC 或 slot

procarray.c:
  ComputeXidHorizons 聚合所有来源

vacuum.c / vacuumlazy.c:
  使用 horizon，决定 removable / freeze / verbose
```

这个链路说明：cleanup delay 是系统级现象，不是单表局部问题。

## 15. 观测与诊断入口

第一步，查长事务。

```sql
SELECT pid, usename, datname, state,
       backend_xmin, xact_start, now() - xact_start AS xact_age,
       query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

第二步，查 prepared transaction。

```sql
SELECT gid, prepared, now() - prepared AS age,
       owner, database, transaction
FROM pg_prepared_xacts
ORDER BY prepared;
```

第三步，查 replication slot。

```sql
SELECT slot_name, slot_type, active,
       xmin, catalog_xmin,
       restart_lsn, confirmed_flush_lsn,
       wal_status, safe_wal_size
FROM pg_replication_slots
ORDER BY slot_name;
```

第四步，查 walsender。

```sql
SELECT pid, application_name, state, sent_lsn, write_lsn,
       flush_lsn, replay_lsn, sync_state
FROM pg_stat_replication
ORDER BY application_name;
```

第五步，查 VACUUM 输出。

```sql
VACUUM (VERBOSE) cleanup_delay_demo;
```

关注：

```text
dead but not yet removable
removable cutoff
new relfrozenxid
index scan bypassed by failsafe
```

第六步，查表年龄。

```sql
SELECT oid::regclass AS rel,
       age(relfrozenxid) AS xid_age,
       mxid_age(relminmxid) AS mxid_age,
       relpages, relallvisible, relallfrozen
FROM pg_class
WHERE relkind IN ('r', 'm', 't')
ORDER BY age(relfrozenxid) DESC
LIMIT 20;
```

源码断点建议：

```text
ComputeXidHorizons
GetOldestNonRemovableTransactionId
ProcArrayGetReplicationSlotXmin
ReplicationSlotsComputeRequiredXmin
MarkAsPreparing
MarkAsPrepared
ProcessStandbyHSFeedbackMessage
XLogWalRcvSendHSFeedback
vacuum_get_cutoffs
```

断点里关注：

```text
proc->xmin
proc->xid
proc->databaseId
statusFlags
h->slot_xmin
h->slot_catalog_xmin
h->data_oldest_nonremovable
h->catalog_oldest_nonremovable
cutoffs->OldestXmin
```

## 16. 常见误区

误区一：VACUUM 频率加高就一定能清掉 bloat。

正确理解：如果 horizon 被钉住，VACUUM 只能保守保留。

误区二：没有长 SQL 正在跑，就没有 horizon pin。

正确理解：prepared transaction、slot 和 standby feedback 可能没有普通 active query。

误区三：replication slot 只保留 WAL。

正确理解：slot 还可能保留 `xmin` / `catalog_xmin`，影响 tuple cleanup。

误区四：hot standby feedback 是免费减少 standby cancel 的开关。

正确理解：它把 standby query 的保留成本转移到 primary。

误区五：`backend_xmin` 最老的 backend 一定是唯一根因。

正确理解：要同时看 prepared xact 和 slots。

最老 horizon 可能来自别处。

误区六：catalog bloat 和普通表 bloat 完全相同。

正确理解：logical slot 的 `catalog_xmin` 对 catalog cleanup 有特殊影响。

误区七：drop stale slot 永远安全。

正确理解：drop slot 会释放资源保护，但可能意味着复制消费者需要重建或丢失连续性。

这必须结合业务复制语义判断。

## 17. 课堂实验

### 实验一：长事务钉住 cleanup

Session A：

```sql
DROP TABLE IF EXISTS cleanup_delay_demo;
CREATE TABLE cleanup_delay_demo(id int primary key, payload text);

INSERT INTO cleanup_delay_demo
SELECT g, repeat('x', 200)
FROM generate_series(1, 100000) AS g;

BEGIN;
SELECT count(*) FROM cleanup_delay_demo;
```

Session B：

```sql
DELETE FROM cleanup_delay_demo WHERE id <= 80000;
VACUUM (VERBOSE) cleanup_delay_demo;

SELECT pid, backend_xmin, xact_start, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin;
```

然后 Session A：

```sql
COMMIT;
```

Session B：

```sql
VACUUM (VERBOSE) cleanup_delay_demo;
```

比较两次输出。

### 实验二：prepared transaction

测试前确认允许 2PC。

```sql
SHOW max_prepared_transactions;
```

如果值为 0，本实验需要调整配置后重启。

Session A：

```sql
BEGIN;
INSERT INTO cleanup_delay_demo VALUES (-1, 'prepared');
PREPARE TRANSACTION 'cleanup-delay-demo';
```

观察：

```sql
SELECT gid, prepared, transaction
FROM pg_prepared_xacts;
```

清理：

```sql
ROLLBACK PREPARED 'cleanup-delay-demo';
```

讨论 prepared xact 为什么没有普通 backend 仍影响 horizon。

### 实验三：观察 slot

在支持逻辑复制的测试库中：

```sql
SELECT *
FROM pg_create_logical_replication_slot('cleanup_delay_slot', 'test_decoding');
```

执行一些 DML 后观察：

```sql
SELECT slot_name, slot_type, active, xmin, catalog_xmin,
       restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 'cleanup_delay_slot';
```

清理：

```sql
SELECT pg_drop_replication_slot('cleanup_delay_slot');
```

如果没有 `test_decoding` 或 wal_level 不支持，记录缺少的配置，不要在生产库临时开启。

### 实验四：源码断点

```gdb
break ComputeXidHorizons
break GetOldestNonRemovableTransactionId
break ReplicationSlotsComputeRequiredXmin
break ProcessStandbyHSFeedbackMessage
```

执行：

```sql
VACUUM cleanup_delay_demo;
```

观察：

```gdb
print h->data_oldest_nonremovable
print h->catalog_oldest_nonremovable
print h->slot_xmin
print h->slot_catalog_xmin
```

把源码 horizon 和 SQL 视图中的对象对应。

## 18. 讨论题

1. 为什么 prepared transaction 必须进入 ProcArray，而不是只存在 twophase 文件里？

2. logical replication slot 的 `catalog_xmin` 为什么可能比普通 `xmin` 更难处理？

3. hot standby feedback 解决了 standby conflict 的哪一半问题，又把哪一半成本推给 primary？

4. 如果 `pg_stat_activity` 没有老 `backend_xmin`，下一步应该查哪些视图？

5. 为什么 relation kind 会影响 cleanup horizon？

6. inactive slot 长期存在时，drop slot、推进消费、重建订阅各有什么风险？

7. 为什么 cleanup delay 会同时表现为 dead tuple、index bloat、freeze age 和 VM bit 比例下降？

## 19. 本节小结

本节把 cleanup delay 的外部来源统一到一个模型。

```text
任何仍可能观察旧世界的对象
  -> 暴露 xmin / catalog_xmin
  -> ComputeXidHorizons 聚合
  -> GetOldestNonRemovableTransactionId 选择 relation-aware horizon
  -> VACUUM / pruning / freeze 只能在 horizon 之后推进
```

长事务通过 backend `PGPROC.xmin` 进入。

prepared transaction 通过 dummy `PGPROC` 进入。

replication slot 通过 `effective_xmin` / `effective_catalog_xmin` 聚合进入。

hot standby feedback 通过 walsender `MyProc->xmin` 或 physical slot 进入。

本节可迁移规律是：

```text
存储清理不是只由本地数据页状态决定；
它还受所有外部观察者的最老语义需求约束。
```

定位 cleanup delay 时，必须同时看：

```text
pg_stat_activity
pg_prepared_xacts
pg_replication_slots
pg_stat_replication
VACUUM VERBOSE
pg_class age
```

到这里，第三目录从单个 snapshot 的字段，推进到了全实例 cleanup horizon、VACUUM、freeze、visibility map 和复制反馈对可见性清理的共同影响。
