# PostgreSQL Logical Decoding catalog 可见性与 catalog_xmin

## 课程定位
前置知识：已经理解 logical slot 保护 `restart_lsn`、`xmin` 和 `catalog_xmin` 的基本语义，也理解 ReorderBuffer 按事务提交顺序输出 WAL change。
本节唯一主问题：

```text
解码旧 WAL 时为什么还需要访问当时有效的 catalog 元数据，
catalog_xmin 如何保护 relation schema、type 和 replica identity 的可见版本？
```
核心矛盾：logical decoding 的数据变更来自 WAL，但 WAL record 并不携带完整 SQL metadata。解码器看到的是 relfilenumber、tuple image、xid、LSN、commit record 和 invalidation；output plugin 需要的是 relation name、namespace、列名、列顺序、type oid、typmod、replica identity、publication column list、row filter 和分区发布规则。metadata 的权威来源仍是 catalog。
因此 logical decoding 必须同时保留两条历史：

```text
WAL history:
  旧 change record 还能被重新读取。
catalog MVCC history:
  解码旧 change 时还能读到当时可见的 pg_class / pg_attribute /
  pg_type / pg_index / publication 等 catalog tuple version。
```
学完后应能判断：
`catalog_xmin` 保护的不是 relcache、typcache 或 `RelationSyncEntry` 这些内存对象；它保护的是能重建这些对象的历史 catalog tuple version。`historic snapshot` 也不是普通旧数据查询能力；它是 logical decoding 为 catalog access 临时安装的 `SNAPSHOT_HISTORIC_MVCC`。schema/type/replica identity 的 correctness 来自 `catalog_xmin`、SnapBuild、ReorderBuffer、cache invalidation 和 output plugin callback 的组合。
本课基于本地源码：

```text
/home/nail/postgres
master commit 0e1f1ed157e90741e12a3715909e1b2d71ff9344
```

## 1. 本节在总主线中的位置
前面 replication slot 课程回答的是：旧 WAL 是否还在，slot 如何因为 `restart_lsn` 和 `catalog_xmin` 阻止过早回收。ReorderBuffer 课程回答的是：按 LSN 到来的 record 如何重新组成事务级 logical stream。
本节补上第三块：即使 WAL 还在、事务顺序也重建了，解码器如何知道旧 WAL 发生时的 catalog 世界是什么样子。
先看一个最小时间线：

```text
LSN 10: CREATE TABLE t(a int PRIMARY KEY, b text)
LSN 20: INSERT INTO t VALUES (1, 'old')
LSN 30: ALTER TABLE t ADD COLUMN c text
LSN 40: ALTER TABLE t REPLICA IDENTITY FULL
LSN 50: UPDATE t SET b = 'new' WHERE a = 1
```
LSN 20 的 INSERT 必须按两列 schema 解释；LSN 50 的 UPDATE 必须按三列 schema 解释，并且 replica identity 规则要使用 LSN 40 之后的 `pg_class.relreplident` 和相关 index metadata。如果 output plugin 只打开当前 `t`，旧 WAL 会被新 schema 误解。
再看同一事务内的 DDL/DML：

```text
BEGIN;
ALTER TABLE t ADD COLUMN d int;
INSERT INTO t(a,b,c,d) VALUES (...);
COMMIT;
```
同一事务内，catalog tuple 的可见性还受 command id 影响。普通 MVCC snapshot 依赖 `curcid`、`cmin`、`cmax`；logical decoding 不能从普通 heap WAL 里直接拿到完整 command id 信息，所以 catalog 修改路径还要记录 `XLOG_HEAP2_NEW_CID`。
本节主线：

```text
slot 创建时钉住 catalog_xmin
  -> SnapBuild 从 WAL 构造 historic catalog snapshot
  -> heapam 为 catalog 修改写 NEW_CID
  -> ReorderBuffer 把 snapshot / command id / invalidation 放入事务流
  -> output plugin 在 SetupHistoricSnapshot() 下读取 catalog metadata
  -> pgoutput 用 RelationSyncEntry 缓存发送状态
  -> client ack 后 catalog_xmin 才能安全前移
```

## 2. 核心矛盾与一句话运行模型
一句话运行模型：

```text
logical slot 用 catalog_xmin 把 catalog tuple 的可移除边界钉住；
SnapBuild 根据 WAL 中的 running-xacts、commit、NEW_CID 和 invalidation
构造只用于 catalog 的 historic snapshot；
ReorderBuffer 在输出每个 change 前安装这个 snapshot；
relcache/syscache/typcache 因而读到旧 LSN 处可见的 catalog metadata；
downstream 确认足够 LSN 后，slot 才持久化并生效更高的 catalog_xmin。
```
tension 是：

```text
VACUUM 希望尽快清理旧 catalog tuple version
  vs
logical decoding 可能很久之后才解码依赖这些 version 的旧 WAL
```
PostgreSQL 没有把完整 catalog metadata 塞进每条 heap WAL。那会让 WAL 体积和编码复杂度显著上升，也会把 relation/type/index/publication 的知识复制到每种 change record。它也不能使用当前 catalog，因为当前 catalog 无法解释旧 WAL。
所以选择是：WAL 保存 data change 和必要边界；catalog 仍是 metadata 的权威来源；replication slot 用 `catalog_xmin` 延迟 catalog tuple 回收；SnapBuild 用 WAL 重建“当时”的 catalog MVCC 视图。
`restart_lsn` 保护的是：

```text
我还能读到哪些 WAL record。
```
`catalog_xmin` 保护的是：

```text
我读到这些 WAL record 后，还能否从 catalog 里读到解释它们所需的 tuple version。
```

## 3. 核心文件分工与阅读顺序
本课按状态边界阅读源码，不按文件名排序。

| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/replication/slot.h` | `data.catalog_xmin`、`effective_catalog_xmin`、`candidate_catalog_xmin`、`candidate_xmin_lsn`。 |
| 2 | `src/backend/replication/logical/logical.c` | `CreateInitDecodingContext()`、`LogicalIncreaseXminForSlot()`、`LogicalConfirmReceivedLocation()`。 |
| 3 | `src/backend/storage/ipc/procarray.c` | `GetOldestSafeDecodingTransactionId()`、`ComputeXidHorizons()`、`ProcArraySetReplicationSlotXmin()`。 |
| 4 | `src/include/replication/snapbuild.h` | `SnapBuildState` 的四个阶段。 |
| 5 | `src/include/replication/snapbuild_internal.h` | `struct SnapBuild` 的 `xmin`、`xmax`、`initial_xmin_horizon`、`committed`、`catchange`。 |
| 6 | `src/backend/replication/logical/snapbuild.c` | `SnapBuildProcessRunningXacts()`、`SnapBuildProcessChange()`、`SnapBuildCommitTxn()`。 |
| 7 | `src/backend/replication/logical/reorderbuffer.c` | `ReorderBufferProcessTXN()`、snapshot / command id / invalidation change。 |
| 8 | `src/backend/replication/logical/decode.c` | heap、xact、running-xacts、invalidation WAL record 分发。 |
| 9 | `src/backend/access/heap/heapam.c` | catalog 修改时 `log_heap_new_cid()`，DML WAL 保存 tuple/key 的规则。 |
| 10 | `src/backend/catalog/heap.c` | `AddNewRelationTuple()` 写 `pg_class` relation metadata。 |
| 11 | `src/backend/replication/pgoutput/pgoutput.c` | `RelationSyncEntry`、`get_rel_sync_entry()`、`maybe_send_schema()`。 |
| 12 | `src/backend/replication/logical/proto.c` | `logicalrep_write_rel()`、`logicalrep_write_typ()`、`logicalrep_write_attrs()`。 |
| 13 | `src/backend/utils/time/snapmgr.c` | `SetupHistoricSnapshot()`、`GetCatalogSnapshot()`、`TeardownHistoricSnapshot()`。 |
| 14 | `src/backend/utils/cache/relcache.c` | historic snapshot 下构造 relation descriptor，`RelationGetReplicaIndex()`。 |
还有一套订阅端 relation map：

```text
src/backend/replication/logical/relation.c
src/include/replication/logicalrelation.h
```
它的 `LogicalRepRelMapEntry` 把 publisher 发来的 relation id 映射到 subscriber 本地 relation。本节主问题在 publisher 解码端，但后面会把这套结构作为 protocol 边界提到。

## 4. slot 中的 catalog horizon 状态
`src/include/replication/slot.h` 中，持久化字段在 `ReplicationSlotPersistentData`：

```text
data.xmin:
  data tuple 的 xmin horizon。
data.catalog_xmin:
  catalog tuple 的 xmin horizon。
data.restart_lsn:
  最旧仍可能需要读取的 WAL LSN。
data.confirmed_flush:
  client 已确认收到的 LSN 边界。
```
内存生效字段在 `ReplicationSlot`：

```text
effective_xmin:
  当前真正参与全局 xmin 计算的数据 horizon。
effective_catalog_xmin:
  当前真正参与 catalog horizon 计算的 catalog horizon。
candidate_catalog_xmin:
  解码器已经知道可以推进到的 catalog xmin 候选值。
candidate_xmin_lsn:
  这个候选值需要 client 至少确认到哪个 LSN。
```
组合语义是：

```text
data.catalog_xmin:
  crash 后还能恢复的承诺。
effective_catalog_xmin:
  当前阻止 VACUUM 的生效值。
candidate_catalog_xmin + candidate_xmin_lsn:
  未来可释放的候选边界，必须等待 downstream ack。
```
`LogicalConfirmReceivedLocation()` 的顺序是核心不变量：

```text
client ack >= candidate_xmin_lsn
  -> 更新 data.catalog_xmin
  -> ReplicationSlotSave()
  -> 更新 effective_catalog_xmin
  -> ReplicationSlotsComputeRequiredXmin(false)
```
不能先改 `effective_catalog_xmin`。如果 VACUUM 因内存值前移清理了 catalog tuple，而 slot state 文件还没保存，crash 后 decoder 可能从旧位置重启，却再也读不到对应 catalog tuple。

## 5. SnapBuild 的 historic catalog snapshot
`SnapBuild` 是 logical decoding backend 私有状态，由 `AllocateSnapshotBuilder()` 创建，挂在 `LogicalDecodingContext.snapshot_builder` 上。它不是 shared memory。
`struct SnapBuild` 中与本节相关的字段：

```text
state:
  snapshot building 阶段。
xmin:
  所有小于它的事务已经结束。
xmax:
  所有大于等于它的事务在 snapshot 视角下未提交。
initial_xmin_horizon:
  slot 创建时得到的安全 decoding horizon。
start_decoding_at:
  不输出早于该 LSN 的 commit。
snapshot:
  当前可用于读 catalog 的 SNAPSHOT_HISTORIC_MVCC。
committed.xip:
  [xmin, xmax) 内需要被当作 committed 的 catalog-changing xids。
catchange.xip:
  restore serialized snapshot 后仍需记住的 catalog-changing xids。
```
`snapbuild.c` 顶部注释说明了它和普通 MVCC snapshot 的差异。普通 snapshot 记录运行中事务；historic catalog snapshot 主要记录在 `[xmin, xmax)` 内哪些 catalog-changing transactions 应当被视为 committed。
这样做的原因是：logical decoding 不需要用旧 snapshot 扫普通用户表。tuple data 已在 WAL 里，真正需要从 catalog 读取的是 relation/type/index/publication metadata。因此 SnapBuild 可以只精确追踪 catalog-changing transactions，而不是所有事务。
状态机在 `src/include/replication/snapbuild.h`：

```text
SNAPBUILD_START:
  还没有可用 running-xacts 信息。
SNAPBUILD_BUILDING_SNAPSHOT:
  已看到 running-xacts，等待当时已经运行的事务结束。
SNAPBUILD_FULL_SNAPSHOT:
  后续新事务已有足够信息可解码，但还不能输出。
SNAPBUILD_CONSISTENT:
  更早 running transactions 都结束，可以开始输出 commit 之后的事务。
```
这保证第一次输出前不会漏掉更早开始、但当时尚未完整收集的事务。

## 6. slot 创建时为什么先钉住 catalog_xmin
创建 logical slot 的关键路径：

```text
CreateInitDecodingContext()
  -> GetOldestSafeDecodingTransactionId(!need_full_snapshot)
  -> slot->effective_catalog_xmin = xmin_horizon
  -> slot->data.catalog_xmin = xmin_horizon
  -> ReplicationSlotsComputeRequiredXmin(true)
  -> ReplicationSlotSave()
  -> StartupDecodingContext()
  -> AllocateSnapshotBuilder(..., xmin_horizon, ...)
```
这段代码同时拿 `ReplicationSlotControlLock` 和 `ProcArrayLock`，避免“刚算出 safe horizon，别的 backend 立即推进 VACUUM horizon”的竞态。设置 slot 后，slot machinery 开始替它保护 catalog tuple。
`GetOldestSafeDecodingTransactionId()` 的返回值是一个保守边界：保证没有大于等于该 xid 的 catalog tuple 被 VACUUM 移除，除非相关事务 abort。SnapBuild 用这个值作为 `initial_xmin_horizon`。
后续 `SnapBuildFindSnapshot()` 遇到 running-xacts record 时会检查：

```text
running->oldestRunningXid < builder->initial_xmin_horizon
  -> 这个点太旧，可能缺 catalog rows
  -> 跳过，等待更新的 running-xacts record
```
所以 `catalog_xmin` 有两层作用：

```text
对 VACUUM:
  从现在开始保留需要的 catalog tuple version。
对 SnapBuild:
  不要从一个已经不保证 catalog history 完整的旧点构造 snapshot。
```
这也解释了为什么“只保留 WAL 文件”不能让一个新 slot 从任意旧 LSN 安全解码。slot 创建前已经被清理的 catalog tuple version 不会因为 WAL 还在而复活。

## 7. catalog_xmin 如何进入 VACUUM / pruning
slot 本地字段必须传播到 ProcArray，VACUUM 才会尊重它。路径是：

```text
ReplicationSlotsComputeRequiredXmin()
  -> 扫描所有 slots
  -> 取最小 effective_xmin
  -> 取最小 effective_catalog_xmin
  -> ProcArraySetReplicationSlotXmin(agg_xmin, agg_catalog_xmin, ...)
```
`ProcArraySetReplicationSlotXmin()` 写入：

```text
procArray->replication_slot_xmin
procArray->replication_slot_catalog_xmin
```
之后 `ComputeXidHorizons()` 把它们并入全局 horizon。关键区别是：

```text
slot_xmin:
  影响 shared/data horizon。
slot_catalog_xmin:
  额外影响 shared/catalog horizon。
```
源码注释说，catalog horizon 和 data horizon 的区别就是 slot 的 catalog xmin 会应用到 catalog horizon，这样 logical decoding 才能访问 catalogs。
heap pruning / VACUUM 消费这个结果：

```text
GlobalVisTestFor(relation)
  -> 根据 relation 选择 shared/catalog/data/temp horizon
heap_page_prune_opt()
  -> GlobalVisTestIsRemovableXid(vistest, prune_xid, true)
heap_prune_satisfies_vacuum()
  -> HeapTupleSatisfiesVacuumHorizon()
  -> GlobalVisTestIsRemovableXid()
```
对 catalog relation，`GlobalVisTestFor()` 选择 `GlobalVisCatalogRels`，其边界来自已经考虑 `replication_slot_catalog_xmin` 的 `catalog_oldest_nonremovable`。
因此保护不是特殊分支：

```text
if logical_decoding then don't vacuum pg_class
```
而是统一的 MVCC horizon：

```text
catalog tuple 的删除 xid 还没有早于 catalog horizon
  -> 仍可能被 historic snapshot 需要
  -> 不能 remove / prune
```

## 8. heapam 为什么为 catalog 修改写 NEW_CID
catalog 修改需要处理同一事务内 command visibility。`snapbuild.c` 顶部注释给出根因：`cmin/cmax` 不完整地保存在 heap tuple 中，combo CID 只在原 backend 内存中有意义，crash recovery 后也不能从 tuple 自身恢复完整 command boundary。
因此 `heapam.c` 在 catalog relation 修改时写额外 WAL：

```text
heap_insert()
  -> if RelationIsAccessibleInLogicalDecoding(relation)
       log_heap_new_cid(relation, heaptup)
heap_delete()
  -> if RelationIsAccessibleInLogicalDecoding(relation)
       log_heap_new_cid(relation, &tp)
heap_update()
  -> catalog relation 同样需要 NEW_CID
```
`RelationIsAccessibleInLogicalDecoding()` 在 `src/include/utils/rel.h`：

```text
XLogLogicalInfoActive()
  && RelationNeedsWAL(relation)
  && (IsCatalogRelation(relation) || RelationIsUsedAsCatalogTable(relation))
```
解码侧读到 `XLOG_HEAP2_NEW_CID` 后：

```text
SnapBuildProcessNewCid()
  -> ReorderBufferXidSetCatalogChanges()
  -> ReorderBufferAddNewTupleCids()
  -> ReorderBufferAddNewCommandId()
```
结果是：

```text
该 xid 被标记为 catalog-changing；
(relfilelocator, ctid) -> (cmin, cmax) 映射进入 ReorderBuffer；
change stream 中出现 command id 边界。
```
输出时 `ReorderBufferProcessTXN()` 会：

```text
ReorderBufferBuildTupleCidHash()
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
```
heap visibility 代码在 historic snapshot 下可以通过 `ResolveCminCmaxDuringDecoding()` 和 `HistoricSnapshotGetTupleCids()` 找到真实 `cmin/cmax`，从而解释 mixed DDL/DML transaction。

## 9. historic snapshot 在哪里安装
`src/backend/utils/time/snapmgr.c` 提供：

```text
SetupHistoricSnapshot(Snapshot historic_snapshot, HTAB *tuplecids)
TeardownHistoricSnapshot(bool is_error)
HistoricSnapshotActive()
HistoricSnapshotGetTupleCids()
```
安装后，catalog snapshot 获取路径变成：

```text
GetCatalogSnapshot()
  -> if HistoricSnapshotActive()
       return HistoricSnapshot
  -> otherwise GetNonHistoricCatalogSnapshot()
```
`GetTransactionSnapshot()` 在 logical decoding 期间也会返回 historic snapshot，但注释强调它只适合 catalog access，调用者必须保证不拿它做普通查询。
`heapam.c` 中有硬边界：

```text
if snapshot is SNAPSHOT_HISTORIC_MVCC
   and !RelationIsAccessibleInLogicalDecoding(relation)
  -> ERROR "cannot query non-catalog table ... during logical decoding"
```
`ReorderBufferProcessTXN()` 是安装 snapshot 的主要位置：

```text
ReorderBufferBuildTupleCidHash()
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
StartTransactionCommand() 或 BeginInternalSubTransaction()
迭代 change
  -> RelationIdGetRelation(reloid)
  -> output plugin callback
遇到 INTERNAL_SNAPSHOT
  -> TeardownHistoricSnapshot()
  -> 切换 snapshot_now
  -> SetupHistoricSnapshot()
遇到 INTERNAL_COMMAND_ID
  -> 更新 snapshot_now->curcid
  -> 重新 SetupHistoricSnapshot()
结束或 ERROR
  -> TeardownHistoricSnapshot()
  -> AbortCurrentTransaction()
```
所以 output plugin 回调里打开 relation、查 type、读 publication 规则，不是在当前 catalog 下读，而是在当前 change 所对应的 historic catalog snapshot 下读。

## 10. relation schema 与 RelationSyncEntry
pgoutput 的发送路径：

```text
pgoutput_change()
  -> get_rel_sync_entry(data, relation)
  -> maybe_send_schema(ctx, change, relation, relentry)
  -> send_relation_and_attrs(relation, xid, ctx, relentry)
  -> logicalrep_write_typ()
  -> logicalrep_write_rel()
```
`RelationSyncEntry` 定义在 `src/backend/replication/pgoutput/pgoutput.c`。核心字段：

```text
relid:
  hash key。
replicate_valid:
  entry 的发布规则和子结构是否仍有效。
schema_sent:
  当前 relation schema 是否已经发送给 downstream。
streamed_txns:
  streaming transaction 下已发送 schema 的 top-level xid 列表。
pubactions:
  insert/update/delete/truncate 哪些动作需要发布。
publish_as_relid:
  分区场景下是否用 ancestor schema 发布。
attrmap:
  partition tuple 到 ancestor TupleDesc 的列映射。
columns:
  publication column list。
exprstate[] / estate:
  row filter executor state。
entry_cxt:
  per-entry 私有内存上下文。
```
这个 cache 不是 correctness 的根。它只是 pgoutput 在 historic snapshot 之上构造的发送缓存。真正决定读哪个 schema 的是当前安装的 historic snapshot，真正保护旧 schema tuple 的是 `catalog_xmin`。
`get_rel_sync_entry()` 重建时会读 relation namespace、publication membership、schema publication、partition ancestors、column list、row filter 和 generated column 设置。源码注释指出：这里不需要锁 namespace system table，因为 entry 使用 historic snapshot 构造，后续变化会在解码 WAL 时吸收。
`send_relation_and_attrs()` 先为用户自定义类型发送 TYPE 消息，再发送 RELATION 消息：

```text
for each published attribute:
  if att->atttypid >= FirstGenbkiObjectId
    logicalrep_write_typ(...)
logicalrep_write_rel(ctx->out, xid, relation, columns, include_gencols_type)
```
`logicalrep_write_rel()` 会写 relation oid、namespace、relation name、`rel->rd_rel->relreplident`、attribute names、type oids、typmod 和 replica identity attribute flags。这些都来自 historic snapshot 下构造出的 `Relation` descriptor。

## 11. type metadata 也受 catalog_xmin 保护
schema 不只是列名。tuple value 的序列化需要 type metadata。
`proto.c` 中：

```text
logicalrep_write_typ()
  -> getBaseType(typoid)
  -> SearchSysCache1(TYPEOID, ...)
  -> 发送 namespace / typname
logicalrep_write_tuple()
  -> TupleDescAttr(desc, i)->atttypid
  -> SearchSysCache1(TYPEOID, atttypid)
  -> 根据 typsend / typoutput 选择 binary/text 输出
```
因此 `catalog_xmin` 保护的不只是 `pg_class`。它还保护：

```text
pg_attribute:
  列名、列顺序、atttypid、atttypmod、attisdropped、attgenerated。
pg_type:
  type name、namespace、base type、send/output metadata。
pg_index:
  primary key、replica identity index、index key columns。
pg_publication / pg_publication_rel:
  发布动作、column list、row filter。
pg_namespace:
  schema rename 对 RELATION / TYPE 消息的 namespace 解释。
```
syscache 和 typcache 可以失效重建，但如果底层历史 tuple version 被 VACUUM 删除，cache 再重建也没有来源。这就是 `catalog_xmin` 的真正保护对象：可重建 metadata 的历史 catalog tuple version。

## 12. replica identity 的两条路径
replica identity 在 logical decoding 中有两次作用。
第一次在写 WAL 时。`heapam.c` 根据 relation metadata 决定 UPDATE / DELETE 需要写入哪些 old tuple/key：

```text
RelationIsLogicallyLogged(relation)
relation->rd_rel->relreplident
RelationGetReplicaIndex(relation)
ExtractReplicaIdentity()
```
如果 `relreplident == REPLICA_IDENTITY_FULL`，old tuple 可能完整写入 WAL；如果使用 primary key 或 replica identity index，WAL 可以只携带 key columns。
第二次在输出协议中。`logicalrep_write_attrs()` 会：

```text
replidentfull = (rel->rd_rel->relreplident == REPLICA_IDENTITY_FULL)
if not full:
  idattrs = RelationGetIdentityKeyBitmap(rel)
for each published attribute:
  if full or attnum in idattrs:
    flags |= LOGICALREP_IS_REPLICA_IDENTITY
```
DDL 修改 replica identity 的入口在 `tablecmds.c`：

```text
relation_mark_replica_identity()
  -> 更新 pg_class.relreplident
  -> 更新 pg_index.indisreplident
```
创建 relation 的初始 metadata 在 `catalog/heap.c`：

```text
AddNewRelationTuple()
  -> values[Anum_pg_class_relreplident - 1] = rd_rel->relreplident
  -> values[Anum_pg_class_reltype - 1] = rd_rel->reltype
  -> values[Anum_pg_class_relnatts - 1] = rd_rel->relnatts
```
一个常见误解是：old key 已经在 WAL 里，所以不需要旧 replica identity metadata。问题在于 output protocol 还要告诉 subscriber 这些值对应哪些列、哪些列属于 identity、使用什么 type 和 typmod 解释。这仍然依赖 historic relation metadata。

## 13. DDL 与解码顺序
DDL 是 catalog table 的 DML。`ALTER TABLE` 可能修改 `pg_class`、`pg_attribute`、`pg_type`、`pg_index`、dependency 和 publication 相关 catalog，并产生 relcache/syscache invalidation。
logical decoding 不解析 DDL SQL 文本。它观察 DDL 带来的 catalog WAL 和 invalidation。
顺序是：

```text
DDL transaction 修改 catalog tuple
  -> heapam 写 NEW_CID
  -> xact WAL 记录 invalidation
  -> SnapBuildCommitTxn() 识别 catalog-changing xact
  -> 构造新的 historic snapshot
  -> SnapBuildDistributeSnapshotAndInval()
  -> ReorderBufferProcessTXN() 在 change stream 中按 LSN 切换 snapshot
  -> pgoutput 下一次 get_rel_sync_entry() 读到新 schema
```
并发 DDL 场景：

```text
T1: 长事务，先写 user table
T2: 并发 DDL，提交
T1: 后续继续写 user table
T1: 提交
```
T1 的 change stream 内部可能跨过 T2 的 DDL commit LSN。SnapBuild 会给 T1 插入 `REORDER_BUFFER_CHANGE_INTERNAL_SNAPSHOT`，让后续 change 使用新的 catalog snapshot。
同一事务 DDL/DML 场景则依赖：

```text
REORDER_BUFFER_CHANGE_INTERNAL_COMMAND_ID
tuplecid_hash
snapshot_now->curcid
```
PostgreSQL 不是让 DDL “当前立即全局生效”，也不是让整个事务只看一个 catalog 版本。它把 DDL 的 effect 放到 WAL LSN 和 command id 轴上，让 decoding 按相同时间轴切换 catalog view。

## 14. cache invalidation 不是锁
本节有三类 cache：

```text
通用 cache:
  relcache / syscache / catcache / typcache
ReorderBuffer 保存的 invalidation:
  txn->invalidations
  txn->invalidations_distributed
  REORDER_BUFFER_CHANGE_INVALIDATION
pgoutput 私有 cache:
  RelationSyncCache
  RelationSyncEntry
  publications_valid
```
ReorderBuffer 的 invalidation 路径：

```text
ReorderBufferAddInvalidations()
  -> 累积到 top transaction
  -> 也作为 REORDER_BUFFER_CHANGE_INVALIDATION 入队
ReorderBufferProcessTXN()
  -> change action == REORDER_BUFFER_CHANGE_INVALIDATION
       ReorderBufferExecuteInvalidations()
       LocalExecuteInvalidationMessage()
事务 cleanup
  -> 执行 txn->invalidations
  -> 执行 txn->invalidations_distributed
  -> overflow 时 InvalidateSystemCaches()
```
pgoutput 注册：

```text
CacheRegisterRelcacheCallback(rel_sync_cache_relation_cb, ...)
CacheRegisterSyscacheCallback(NAMESPACEOID, rel_sync_cache_publication_cb, ...)
CacheRegisterSyscacheCallback(PUBLICATIONOID, publication_invalidation_cb, ...)
```
`rel_sync_cache_relation_cb()` 只把 `entry->replicate_valid` 置 false，不直接释放 entry。原因是 invalidation 可能在 output plugin callback 中发生，栈上仍可能使用 entry 的子结构。下一次 `get_rel_sync_entry()` 再集中释放 tuple slots、attrmap、row filter state 和 `entry_cxt`。
结论：

```text
invalidation 不是并发互斥；
invalidation 只说明 cache 语义过期；
tuple version 是否还在由 catalog_xmin 和 horizon 决定；
读哪个版本由 historic snapshot 决定。
```

## 15. 主流程源码 walkthrough

### 15.1 创建 slot
入口：

```text
CreateInitDecodingContext()
```
状态变化：

```text
CheckLogicalDecodingRequirements()
  -> 确认数据库连接、wal_level / standby logical decoding 条件。
ReplicationSlotReserveWal()
  -> 设置初始 restart_lsn。
ReplicationSlotControlLock + ProcArrayLock
  -> GetOldestSafeDecodingTransactionId()
  -> 设置 slot->data.catalog_xmin / effective_catalog_xmin
  -> need_full_snapshot 时临时设置 effective_xmin
  -> ReplicationSlotsComputeRequiredXmin(true)
ReplicationSlotSave()
  -> 持久化 slot state。
StartupDecodingContext()
  -> ReorderBufferAllocate()
  -> AllocateSnapshotBuilder(..., xmin_horizon, ...)
```
此时 VACUUM 已经必须尊重该 logical slot 的 catalog horizon。

### 15.2 找 consistent point
入口：

```text
DecodingContextFindStartpoint()
  -> XLogBeginRead(slot->data.restart_lsn)
  -> XLogReadRecord()
  -> LogicalDecodingProcessRecord()
  -> DecodingContextReady()
```
读到 running-xacts：

```text
DecodeRunningXacts()
  -> SnapBuildProcessRunningXacts()
  -> SnapBuildFindSnapshot()
```
关键状态推进：

```text
running-xacts 太旧:
  running->oldestRunningXid < initial_xmin_horizon，跳过。
没有 running xacts:
  直接 SNAPBUILD_CONSISTENT。
START -> BUILDING_SNAPSHOT:
  记录 next_phase_at，等待旧事务结束。
BUILDING_SNAPSHOT -> FULL_SNAPSHOT:
  后续事务已有足够 catalog 信息。
FULL_SNAPSHOT -> CONSISTENT:
  旧事务全部结束，可以输出。
```

### 15.3 heap change 入 ReorderBuffer
`decode.c` 中：

```text
LogicalDecodingProcessRecord()
  -> DecodeInsert()
  -> DecodeUpdate()
  -> DecodeDelete()
```
heap change 入队前：

```text
SnapBuildProcessChange(builder, xid, lsn)
```
如果还没到 `SNAPBUILD_FULL_SNAPSHOT`，不保留该 change。该 xid 第一次需要解码时：

```text
if !ReorderBufferXidHasBaseSnapshot(reorder, xid):
  builder->snapshot = SnapBuildBuildSnapshot(builder)
  ReorderBufferSetBaseSnapshot(reorder, xid, lsn, builder->snapshot)
```
随后 change 进入 `ReorderBufferQueueChange()`，挂到对应 `ReorderBufferTXN`，还不会调用 output plugin。

### 15.4 catalog NEW_CID
读到 `XLOG_HEAP2_NEW_CID`：

```text
SnapBuildProcessNewCid()
  -> ReorderBufferXidSetCatalogChanges()
  -> ReorderBufferAddNewTupleCids()
  -> ReorderBufferAddNewCommandId()
```
这让后续输出能解释同一事务内 catalog tuple 在哪个 command 后可见。

### 15.5 commit record
commit record 入口：

```text
DecodeCommit()
  -> SnapBuildCommitTxn()
  -> ReorderBufferCommitChild()
  -> ReorderBufferCommit()
```
如果事务或 subxact 改 catalog：

```text
SnapBuildAddCommittedTxn()
builder->xmax 前移
builder->snapshot = SnapBuildBuildSnapshot(builder)
ReorderBufferSetBaseSnapshot() 或 ReorderBufferAddSnapshot()
SnapBuildDistributeSnapshotAndInval()
```
`SnapBuildDistributeSnapshotAndInval()` 会遍历仍在运行且已有 base snapshot 的事务，给它们排入新的 snapshot 和分布式 invalidation。这就是并发 DDL 对长事务后续 change 生效的边界。

### 15.6 输出事务
真正输出：

```text
ReorderBufferCommit()
  -> ReorderBufferProcessTXN()
```
核心操作：

```text
ReorderBufferBuildTupleCidHash()
SetupHistoricSnapshot(snapshot_now, txn->tuplecid_hash)
StartTransactionCommand()
ReorderBufferIterTXNNext()
  -> RelationIdGetRelation(reloid)
  -> RelationIsLogicallyLogged(relation)
  -> rb->apply_change()
TeardownHistoricSnapshot()
AbortCurrentTransaction()
ReorderBufferCleanupTXN()
```
pgoutput callback：

```text
change_cb_wrapper()
  -> pgoutput_change()
     -> get_rel_sync_entry()
     -> maybe_send_schema()
     -> logicalrep_write_insert/update/delete()
```
到这里，旧 WAL payload 和旧 catalog snapshot 在 output plugin 前重新合流。

## 16. 生命周期 / ownership / cleanup
slot horizon：

```text
创建:
  CreateInitDecodingContext()
持有:
  ReplicationSlot
  ProcArray replication_slot_catalog_xmin
推进:
  SnapBuildProcessRunningXacts()
  LogicalIncreaseXminForSlot()
  LogicalConfirmReceivedLocation()
释放:
  slot drop / invalidation / ReplicationSlotRelease()
  ReplicationSlotsComputeRequiredXmin()
crash 边界:
  data.catalog_xmin 先持久化，effective_catalog_xmin 后前移。
```
historic snapshot：

```text
创建:
  SnapBuildBuildSnapshot()
持有:
  SnapBuild.snapshot
  ReorderBufferTXN.base_snapshot
  INTERNAL_SNAPSHOT change
引用计数:
  SnapBuildSnapIncRefcount()
  SnapBuildSnapDecRefcount()
安装:
  SetupHistoricSnapshot()
拆除:
  TeardownHistoricSnapshot()
释放:
  ReorderBufferCleanupTXN()
  ReorderBufferFreeChange()
  FreeSnapshotBuilder()
```
pgoutput relation cache：

```text
创建:
  init_rel_sync_cache()
  get_rel_sync_entry()
持有:
  RelationSyncCache hash
  PGOutputData.cachectx
失效:
  rel_sync_cache_relation_cb()
  rel_sync_cache_publication_cb()
  publication_invalidation_cb()
释放:
  pgoutput_memory_context_reset()
  pgoutput_shutdown()
  MemoryContextDelete(ctx->context)
```
ERROR cleanup 中，`ReorderBufferProcessTXN()` 会 teardown historic snapshot、abort internal transaction、执行或全量触发 cache invalidation、恢复 `CurrentResourceOwner` / memory context，然后重新抛出错误。

## 17. 正确性机制层次

| 机制 | 保证 | 不保证 |
| --- | --- | --- |
| `restart_lsn` | WAL record 仍可读取。 | catalog tuple 仍存在。 |
| `catalog_xmin` | catalog tuple version 不被过早移除。 | 读哪个版本。 |
| SnapBuild | 构造某个 WAL 时间点的 historic catalog snapshot。 | 阻止当前 DDL。 |
| ReorderBuffer | 按事务、LSN、subxact、command id 输出。 | 自己解释 schema。 |
| NEW_CID WAL | mixed DDL/DML 下恢复 command visibility。 | 保存完整 catalog tuple。 |
| invalidation | cache 过期通知。 | 锁和 tuple 存活保护。 |
| relcache/syscache/typcache | materialize catalog metadata。 | 历史 tuple 被删后重建。 |
| RelationSyncEntry | pgoutput 发送缓存。 | historic visibility。 |
核心不变量：

```text
raw WAL change 只有在正确 historic catalog snapshot 下才有 logical schema 语义。
```

## 18. 异常路径与 fallback
初始 snapshot 太旧：

```text
SnapBuildFindSnapshot()
  -> running->oldestRunningXid < initial_xmin_horizon
  -> 等更晚 running-xacts record
```
这是保守策略。宁可更晚开始，也不能从不完整 catalog history 构造 snapshot。
output plugin ERROR：

```text
PG_CATCH()
  -> TeardownHistoricSnapshot(true)
  -> AbortCurrentTransaction()
  -> 执行 invalidation 或 InvalidateSystemCaches()
  -> 恢复上下文
  -> PG_RE_THROW()
```
distributed invalidation overflow：

```text
txn->txn_flags |= RBTXN_DISTR_INVAL_OVERFLOWED
txn->invalidations_distributed = NULL
txn->ninvalidations_distributed = 0
cleanup 时 InvalidateSystemCaches()
```
relmapper 没有 historic view：
`ReorderBufferProcessTXN()` 中对 catalog table rewrite 有特殊处理。注释指出 `relmapper has no "historic" view`。没有 tuple data 且本来会跳过的 mapped catalog tuple 可以跳过；否则无法把 filenumber 映射到 relation OID 时会 ERROR。
historic snapshot 查询普通表：
`heapam.c` 会报：

```text
cannot query non-catalog table ... during logical decoding
```
这说明 historic snapshot 的边界是 catalog access，不是旧数据查询能力。

## 19. 成本、资源与跨模块传播
`catalog_xmin` 滞留会让旧 catalog tuple version 不能被 VACUUM / prune 清理。影响包括：

```text
pg_class / pg_attribute / pg_type 等 catalog bloat；
syscache miss 时扫描更多 dead tuples；
relcache rebuild 更慢；
DDL-heavy workload 下 invalidation 和 cache rebuild 更频繁；
transaction status 保留压力和 wraparound 风险窗口变紧。
```
这个成本和业务表 DML 量不完全相关。频繁 `CREATE/DROP TABLE`、`ALTER TABLE`、`CREATE/DROP INDEX`、`ALTER TYPE`、修改 publication/column list/row filter 的 workload，即使数据量不大，也可能因为 logical slot lag 造成 catalog bloat。
SnapBuild 成本随 catalog-changing transactions 增长。`SnapBuildBuildSnapshot()` 会复制 `committed.xip` 并排序。通常这很小，因为多数事务不改 catalog；DDL-heavy workload 会放大它。
cache invalidation 成本随这些变量增长：

```text
并发长事务数量；
catalog-changing transaction 频率；
每个事务 invalidation message 数；
RelationSyncCache entry 数；
publication 数、分区层级、column list 和 row filter 复杂度。
```
pgoutput 的 `get_rel_sync_entry()` 重建并不轻。它可能读取 publication list、schema publication、relation publication、partition ancestors、row filter expression、column list、generated column 设置、TupleDesc 和 AttrMap。`RelationSyncEntry` 的作用是减少每条 tuple change 重复读 catalog，但 cache 命中不能绕开 historic snapshot correctness。
推进 `catalog_xmin` 还受 downstream ack 限制：

```text
SnapBuildProcessRunningXacts()
  -> LogicalIncreaseXminForSlot(lsn, xmin)
  -> candidate_catalog_xmin / candidate_xmin_lsn
client feedback
  -> LogicalConfirmReceivedLocation(flush_lsn)
  -> data.catalog_xmin 持久化
  -> effective_catalog_xmin 前移
```
consumer 慢、只 peek 不 consume、网络 ack 慢或 output plugin 慢，都会延长 catalog tuple 保留时间。

## 20. 观测与诊断入口
本节可观测的 runtime truth：

```text
logical slot 的 catalog_xmin 滞留会阻止 catalog tuple 被清理；
解码器仍能在旧 WAL 位置读到当时 schema/type/replica identity。
```
直接看 slot：

```sql
SELECT slot_name,
       active,
       plugin,
       restart_lsn,
       confirmed_flush_lsn,
       xmin,
       catalog_xmin,
       age(catalog_xmin) AS catalog_xmin_age,
       wal_status,
       safe_wal_size
FROM pg_replication_slots
WHERE slot_type = 'logical';
```
这能看到 slot horizon、WAL 保留边界和 ack 边界。看不到 SnapBuild 的 `committed.xip`、某个 `RelationSyncEntry` 是否命中、某个 typcache entry 是否来自 historic snapshot。
看 walsender ack：

```sql
SELECT pid,
       application_name,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       sync_state
FROM pg_stat_replication;
```
对 streaming logical replication，`flush_lsn` 是否追上 decoder 报告的位置，会影响 `LogicalConfirmReceivedLocation()` 能否推进 `catalog_xmin`。
看 catalog bloat 迹象：

```sql
SELECT schemaname, relname, n_live_tup, n_dead_tup,
       last_vacuum, last_autovacuum
FROM pg_stat_all_tables
WHERE schemaname = 'pg_catalog'
ORDER BY n_dead_tup DESC
LIMIT 20;
```
以及：

```sql
SELECT relname,
       pg_size_pretty(pg_total_relation_size(oid)) AS total_size
FROM pg_class
WHERE relnamespace = 'pg_catalog'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC
LIMIT 20;
```
这些指标只能说明结果，不能单独证明原因。要和 `pg_replication_slots.catalog_xmin`、DDL 频率、autovacuum 日志、slot active 状态一起判断。
源码断点建议：

```text
CreateInitDecodingContext
GetOldestSafeDecodingTransactionId
ReplicationSlotsComputeRequiredXmin
ProcArraySetReplicationSlotXmin
SnapBuildProcessRunningXacts
SnapBuildCommitTxn
SnapBuildDistributeSnapshotAndInval
ReorderBufferProcessTXN
SetupHistoricSnapshot
get_rel_sync_entry
logicalrep_write_rel
LogicalConfirmReceivedLocation
```
建议观察：

```text
MyReplicationSlot->data.catalog_xmin
MyReplicationSlot->effective_catalog_xmin
MyReplicationSlot->candidate_catalog_xmin
builder->state
builder->xmin / builder->xmax
txn->base_snapshot
HistoricSnapshotActive()
entry->replicate_valid
entry->schema_sent
relation->rd_rel->relreplident
relation->rd_replidindex
```
相关日志和 wait event：

```text
logical decoding found initial starting point
logical decoding found initial consistent point
logical decoding found consistent point
got new catalog xmin
updated xmin
xmin required by slots
SNAPBUILD_READ / SNAPBUILD_WRITE / SNAPBUILD_SYNC
```
`SNAPBUILD_*` wait event 说明 serialized historic catalog snapshot 文件 IO，不直接等同于 catalog_xmin 滞留。

## 21. 常见误区
误区一：WAL 有 tuple image，所以不需要 catalog。错误。tuple image 只是值，列名、列类型、typmod、replica identity 和 publication 规则仍来自 catalog。
误区二：`catalog_xmin` 保护 relcache。错误。relcache 是内存对象，`catalog_xmin` 保护能重建 relcache 的 catalog tuple version。
误区三：historic snapshot 可以查询旧用户表。错误。它只用于 catalog access，`heapam.c` 会拒绝在 logical decoding 中用 historic snapshot 扫普通表。
误区四：invalidation 能阻止 DDL 并发。错误。invalidation 只是 cache 过期通知；顺序由 WAL LSN、transaction commit 和 command id 决定。
误区五：只要 `restart_lsn` 还在就能解码。错误。还需要 `catalog_xmin` 保护历史 catalog tuple。旧 WAL 和旧 catalog history 缺一不可。
误区六：`RelationSyncEntry.schema_sent` 表示 schema 一定正确。错误。它只表示 pgoutput 认为当前 schema 已发给 downstream；schema 正确性来自 historic snapshot 和 invalidation。

## 22. 课堂实验
实验一：schema 改变前后的 RELATION 消息。

```sql
CREATE TABLE ld_demo(id int PRIMARY KEY, v text);
CREATE PUBLICATION pub_ld_demo FOR TABLE ld_demo;
SELECT * FROM pg_create_logical_replication_slot('s_ld_demo', 'pgoutput');
INSERT INTO ld_demo VALUES (1, 'a');
ALTER TABLE ld_demo ADD COLUMN extra text;
INSERT INTO ld_demo VALUES (2, 'b', 'x');
```
观察第一次 INSERT 前的 RELATION 消息，以及 ALTER 后下一次 change 前是否重新发送 RELATION。断点放在 `pgoutput_change()`、`maybe_send_schema()`、`send_relation_and_attrs()`、`logicalrep_write_rel()`、`rel_sync_cache_relation_cb()`。画出 `schema_sent`、`replicate_valid`、`HistoricSnapshotActive()` 和 `RelationGetDescr(relation)->natts` 的变化。
实验二：catalog_xmin 滞留与 catalog bloat。

```sql
SELECT * FROM pg_create_logical_replication_slot('s_hold_catalog', 'test_decoding');
DO $$
BEGIN
  FOR i IN 1..200 LOOP
    EXECUTE format('CREATE TABLE hold_%s(a int)', i);
    EXECUTE format('ALTER TABLE hold_%s ADD COLUMN b text', i);
    EXECUTE format('DROP TABLE hold_%s', i);
  END LOOP;
END $$;
SELECT slot_name, active, catalog_xmin, age(catalog_xmin),
       restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots
WHERE slot_name = 's_hold_catalog';
```
再观察 `pg_stat_all_tables` 中 `pg_catalog` 的 dead tuples，或者对 `pg_class`、`pg_attribute` 做 `VACUUM VERBOSE`。解释时回到 `ReplicationSlotsComputeRequiredXmin()`、`ComputeXidHorizons()`、`GlobalVisTestFor()` 和 `heap_prune_satisfies_vacuum()`。
实验三：replica identity 改变的边界。

```sql
CREATE TABLE ri_demo(id int PRIMARY KEY, v text, note text);
CREATE PUBLICATION pub_ri_demo FOR TABLE ri_demo;
SELECT * FROM pg_create_logical_replication_slot('s_ri_demo', 'pgoutput');
INSERT INTO ri_demo VALUES (1, 'a', 'n1');
UPDATE ri_demo SET v = 'b' WHERE id = 1;
ALTER TABLE ri_demo REPLICA IDENTITY FULL;
UPDATE ri_demo SET note = 'n2' WHERE id = 1;
```
断点放在 `relation_mark_replica_identity()`、`RelationGetReplicaIndex()`、`logicalrep_write_attrs()`、`logicalrep_write_update()`。观察 `pg_class.relreplident` 的 catalog tuple version、`relation->rd_rel->relreplident`、RELATION 消息里的 identity flags，以及 UPDATE 消息中 old tuple/key 的变化。

## 23. 讨论题
1. 为什么 `restart_lsn` 不能替代 `catalog_xmin`？
2. 为什么 historic snapshot 只允许 catalog access，而不是给用户表也提供旧版本查询能力？
3. `SnapBuildBuildSnapshot()` 为什么跟踪 catalog-changing committed xids，而不是像普通 snapshot 那样记录所有 running xids？
4. `RelationSyncEntry.schema_sent` 为什么不能作为 schema correctness 的根？
5. 如果 `LogicalConfirmReceivedLocation()` 先更新 `effective_catalog_xmin`，再保存 slot state，crash 后可能出现什么问题？
6. DDL 和 DML 在同一事务内混合时，为什么仅有 transaction commit LSN 不足以解释 catalog visibility？
7. cache invalidation overflow 时为什么可以退化为 `InvalidateSystemCaches()`？
8. catalog_xmin 滞留导致性能问题时，哪些现象来自内核机制，哪些来自 workload 的 DDL pattern 或 downstream ack 策略？

## 24. 本节小结
本节主链路：

```text
logical slot 创建时设置 catalog_xmin
  -> ProcArray 把 slot catalog horizon 纳入 catalog_oldest_nonremovable
  -> VACUUM / prune 不能移除仍可能被 historic snapshot 需要的 catalog tuple
  -> SnapBuild 用 WAL 构造 SNAPSHOT_HISTORIC_MVCC
  -> ReorderBuffer 按 LSN 安装 snapshot / command id / invalidation
  -> relcache/syscache/typcache 在 historic snapshot 下重建
  -> pgoutput 读取 relation/type/publication metadata
  -> logicalrep_write_rel / typ / tuple 输出正确 schema 和 replica identity
  -> downstream ack 后 catalog_xmin 先持久化再生效前移
```
核心边界：

```text
catalog_xmin 保护 catalog tuple version；
historic snapshot 决定读哪个版本；
invalidation 让 cache 过期；
RelationSyncEntry 只是 pgoutput 的发送缓存；
restart_lsn 只保护 WAL，不保护 catalog history。
```
错误路径如何收尾：

```text
初始 running-xacts 太旧时等更晚 snapshot；
output plugin ERROR 时 teardown historic snapshot 并 abort internal transaction；
invalidation 太多时全量 InvalidateSystemCaches；
relmapper 没有 historic view 的边界用跳过或 ERROR 处理；
普通表不能被 historic snapshot 查询。
```
可观测入口：

```text
pg_replication_slots.catalog_xmin
age(catalog_xmin)
restart_lsn / confirmed_flush_lsn
pg_stat_replication flush_lsn
pg_catalog 表的 dead tuple 和 size
logical decoding / snapbuild 日志
gdb 中的 SnapBuild、ReplicationSlot、RelationSyncEntry 状态
```
可迁移规律：

```text
当日志系统把 data change 和解释 data 所需的 metadata 分开保存时，
只保留日志本身不够；
还必须为 metadata 提供同一时间轴上的可见性、回收边界和 cache invalidation。
```
在 PostgreSQL logical decoding 中，这条规律具体落在 WAL LSN 轴、catalog MVCC 轴、slot `catalog_xmin` horizon、ReorderBuffer transaction ordering 和 relcache/syscache invalidation ordering 的组合上。线上判断时不要只问 slot 落后多少 WAL，还要问它把 `catalog_xmin` 钉在哪里、这段时间发生了多少 DDL/type/publication/replica identity 变化、downstream 是否确认到了能释放这些 catalog version 的 LSN。
