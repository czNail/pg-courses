# PostgreSQL Output plugin relation schema 与 replica identity 输出边界

## 课程定位
前置知识：已经理解 logical slot 如何保护 WAL 与 catalog history，也理解 `decode.c` 如何把 heap WAL record 解释成 `ReorderBufferChange`，以及 `ReorderBuffer` 如何在 commit 边界把 committed change 交给 output plugin。
本节唯一主问题：
```text
输出 UPDATE / DELETE 时为什么需要 replica identity，
plugin 何时能输出 old key、完整 old tuple 或只输出不可定位的变更？
```
核心矛盾：logical decoding 的输入是历史 WAL 中的物理 tuple image，logical replication 的下游 apply 却必须定位“要更新或删除哪一行”。`INSERT` 只需要 new tuple；`UPDATE` 和 `DELETE` 必须携带足够的旧身份，否则 subscriber 无法判断目标行。
PostgreSQL 不让 output plugin 在输出时回表查旧行。旧行可能已经被更新、删除、prune 或 VACUUM，也可能只能通过 historic catalog snapshot 解释 schema，而不能用普通 MVCC snapshot 重新读取用户表。定位信息必须在 heap WAL 写入端就被捕获：
```text
heap_update / heap_delete
  -> 根据 relation replica identity 决定 old key 或 full old tuple 是否进入 WAL
decode.c
  -> 按 WAL flags 填充 change->data.tp.oldtuple/newtuple
pgoutput.c
  -> 按 publication、row filter、column list 和 relation schema 决定是否输出
proto.c
  -> 用 K / O / N action byte 表达 old key、old tuple 和 new tuple
subscriber relation cache
  -> 把 remote relation schema 映射到本地 relation、key 和定位路径
```
学完后应能判断：为什么 `UPDATE` 有时只发 `N` 而不发 `K` / `O`；为什么 `DELETE` 没有 old image 时 `pgoutput` 会跳过；为什么 `oldtuple != NULL` 不等于旧整行；为什么 `REPLICA IDENTITY FULL` 能输出完整 old tuple 但成本最高；为什么 column list 和 row filter 会反过来约束 replica identity；以及诊断 UPDATE / DELETE 不输出、apply 报错或 subscriber 找不到行时，应该先检查哪一层边界。
本课基于本地源码：
```text
/home/nail/postgres
master commit 0e1f1ed157e90741e12a3715909e1b2d71ff9344
```

## 1. 本节在总主线中的位置
前几节已经建立 logical decoding 的前半条链路：
```text
WAL record
  -> LogicalDecodingProcessRecord()
  -> DecodeUpdate() / DecodeDelete()
  -> ReorderBufferChange
  -> ReorderBufferCommit()
  -> output plugin callback
```
上一节补上 catalog visibility：输出插件不能只拿 tuple bytes，它还必须在正确 historic catalog snapshot 下知道 relation schema、type、replica identity、publication row filter 和 column list。本节继续向 output plugin 边界推进，只跟一类 change：
```text
heap UPDATE / DELETE
  -> old key / full old tuple 是否进入 WAL
  -> decode 后 oldtuple / newtuple 如何出现
  -> pgoutput 是否能发送 UPDATE / DELETE
  -> logical replication protocol 如何表示 K / O / N
  -> subscriber 如何用 logical relation cache 定位本地行
```
先看一个最小现场：
```sql
CREATE TABLE t(id int PRIMARY KEY, payload text);
CREATE PUBLICATION p FOR TABLE t WITH (publish = 'update, delete');
UPDATE t SET payload = 'b' WHERE id = 1;
UPDATE t SET id = 2 WHERE id = 1;
DELETE FROM t WHERE id = 2;
```
第一条 `UPDATE` 没改 primary key，下游可以用 new tuple 里的 `id = 1` 找到旧行。第二条 `UPDATE` 改了 primary key，下游不能用 new tuple 的 `id = 2` 找旧行，必须知道 old key `id = 1`。第三条 `DELETE` 没有 new tuple，必须知道 old key 或 full old tuple。
这就是 replica identity 的基本作用：
```text
它定义 UPDATE / DELETE 在 logical stream 中用什么旧信息定位目标行。
```
本节不展开冲突处理、权限、触发器和 constraint failure。那些属于 apply worker 的后续问题。本节只回答定位信息从哪里来、在哪里丢失、在哪里变成协议语义。

## 2. 核心矛盾与一句话运行模型
一句话运行模型：
```text
publisher 在 heap WAL 写入时按 replica identity 记录必要 old image；
decoder 按 WAL flags 恢复 oldtuple/newtuple；
pgoutput 按 publication 和 row filter 决定动作是否输出或转换；
protocol 用 K/O/N 标记 tuple role；
subscriber 用 relation message 建出的 logical relation cache 找本地 key/index/seqscan apply。
```
这条链路有两个不能混淆的边界。
第一个边界在 publisher 写 WAL 时。它决定 WAL 中有没有 old image、old image 是 replica identity key 还是 full old tuple。这个决定之后不能由 output plugin 补救。
第二个边界在 subscriber apply 时。它决定收到的 key/full tuple 能不能映射到本地列，本地有没有可用 replica identity index、primary key、FULL 可用索引或顺序扫描路径。
`pgoutput` 位于中间。它既不能重建 WAL 中不存在的 old key，也不负责证明 subscriber 一定能找到行。它只保证自己输出的 logical replication protocol message 语义自洽：
```text
RELATION:
  relation id、schema、name、replica identity 和列级 key flags。
UPDATE:
  可能带 K 或 O，一定带 N。
DELETE:
  必须带 K 或 O。
tuple data:
  可以用 unchanged TOAST marker 表示未变化的大字段。
```
这解释了一个常见现象：
```text
UPDATE 没有 oldtuple 不一定是错。
DELETE 没有 oldtuple 通常不能作为 pgoutput DELETE 发出。
```
因为 UPDATE 如果 replica identity key 没变，new tuple 中的 key 仍能定位旧行。DELETE 没有 new tuple，没有 old key 或 old tuple 就没有定位依据。

## 3. 核心文件分工与阅读顺序
| 顺序 | 文件 | 本节关注点 |
| --- | --- | --- |
| 1 | `src/include/catalog/pg_class.h` | `REPLICA_IDENTITY_DEFAULT`、`NOTHING`、`FULL`、`INDEX` 四种语义常量。 |
| 2 | `src/backend/utils/cache/relcache.c` | `RelationGetReplicaIndex()`、`RelationGetIndexAttrBitmap()` 如何得到 replica identity index 和 key bitmap。 |
| 3 | `src/backend/commands/tablecmds.c` | `ATExecReplicaIdentity()`、`relation_mark_replica_identity()` 如何修改 `pg_class.relreplident` 与 `pg_index.indisreplident`。 |
| 4 | `src/backend/executor/execReplication.c` | `CheckCmdReplicaIdentity()` 如何在发布 UPDATE / DELETE 前做执行期检查。 |
| 5 | `src/backend/access/heap/heapam.c` | `heap_update()`、`heap_delete()`、`ExtractReplicaIdentity()`、`log_heap_update()` 如何把 old key/full old tuple 写进 WAL。 |
| 6 | `src/include/access/heapam_xlog.h` | `XLH_UPDATE_CONTAINS_OLD_KEY`、`XLH_UPDATE_CONTAINS_OLD_TUPLE`、`XLH_DELETE_CONTAINS_OLD_KEY`、`XLH_DELETE_CONTAINS_OLD_TUPLE`。 |
| 7 | `src/backend/replication/logical/decode.c` | `DecodeUpdate()`、`DecodeDelete()` 如何根据 WAL flags 填充 `change->data.tp.oldtuple/newtuple`。 |
| 8 | `src/backend/replication/logical/reorderbuffer.c` | commit replay 前的 relation 打开、TOAST reconstruction 和 callback 调用。 |
| 9 | `src/backend/replication/pgoutput/pgoutput.c` | `RelationSyncEntry`、`maybe_send_schema()`、`pgoutput_change()`、row filter 和 column list 输出边界。 |
| 10 | `src/backend/replication/logical/proto.c` | `logicalrep_write_rel()`、`logicalrep_write_update()`、`logicalrep_write_delete()`、`logicalrep_write_tuple()`。 |
| 11 | `src/include/replication/logicalproto.h` | message type、`LogicalRepRelation`、`LOGICALREP_COLUMN_UNCHANGED`。 |
| 12 | `src/backend/replication/logical/relation.c` | subscriber 的 `LogicalRepRelMapEntry`、`logicalrep_relmap_update()`、`logicalrep_rel_open()`、`FindLogicalRepLocalIndex()`。 |
| 13 | `src/backend/replication/logical/worker.c` | `apply_handle_update()`、`apply_handle_delete()`、`FindReplTupleInLocalRel()` 如何使用收到的 key/full tuple。 |
推荐阅读顺序是：先确认四种 replica identity，再读 heap WAL 写入端，然后读 decode flags，最后读 `pgoutput` 和 subscriber relation cache。不要从 `logicalrep_write_update()` 开始推理，否则容易误以为 `K` / `O` 是 output plugin 自己临时选择的。

## 4. 核心状态：定位语义不是单个字段
本节有四组状态。
第一组是 relation 的 replica identity 配置。它来自 catalog：
```text
pg_class.relreplident
pg_index.indisreplident
primary key metadata
index key attribute metadata
```
在 relcache 中常见入口是：
```text
relation->rd_rel->relreplident
relation->rd_replidindex
RelationGetReplicaIndex(relation)
RelationGetIndexAttrBitmap(relation, INDEX_ATTR_BITMAP_IDENTITY_KEY)
```
第二组是 heap WAL flags：
```text
XLH_UPDATE_CONTAINS_OLD_TUPLE
XLH_UPDATE_CONTAINS_OLD_KEY
XLH_UPDATE_CONTAINS_NEW_TUPLE
XLH_DELETE_CONTAINS_OLD_TUPLE
XLH_DELETE_CONTAINS_OLD_KEY
XLH_DELETE_NO_LOGICAL
```
这些 flags 才是 decode 端能否看到 old image 的直接来源。
第三组是 `ReorderBufferChange` 中的 tuple pointers：
```text
change->data.tp.oldtuple
change->data.tp.newtuple
change->data.tp.rlocator
change->data.tp.clear_toast_afterwards
```
这里有一个关键陷阱：`change->data.tp.oldtuple` 可能是 full old tuple，也可能只是 old key tuple。`decode.c` 不把这两种 old image 放到不同字段。
第四组是 protocol relation schema。`logicalrep_write_rel()` 发送 relation id、namespace、relation name、replica identity mode 和 attribute list。subscriber 的 `LogicalRepRelation` 保存：
```text
remoteid
nspname
relname
natts
attnames
atttyps
replident
attkeys
```
定位语义来自这些状态的组合：
```text
catalog identity
  + WAL old image flags
  + reorder buffer tuple image
  + protocol relation schema
```
只看 `relreplident`、`oldtuple` 或 protocol action byte 中任意一个字段，都会误判。

## 5. replica identity 四种模式
四个常量在 `src/include/catalog/pg_class.h`：
```text
REPLICA_IDENTITY_DEFAULT = 'd'
REPLICA_IDENTITY_NOTHING = 'n'
REPLICA_IDENTITY_FULL    = 'f'
REPLICA_IDENTITY_INDEX   = 'i'
```
`DEFAULT` 的含义不是“永远有 key”。它表示如果有非 deferrable primary key，就用这个 primary key；否则没有 replica identity index。`relcache.c` 的 `RelationGetIndexList()` 会设置 `rd_pkindex` 和 `rd_replidindex`。当 `relreplident == DEFAULT` 且有可用 primary key 时，`rd_replidindex` 指向 primary key index，否则为 `InvalidOid`。
`INDEX` 表示用户显式选择一个可用 index。DDL 路径在 `tablecmds.c`：
```text
ATExecReplicaIdentity()
  -> 检查 index 存在且属于该表
  -> 要求 unique 或符合 WITHOUT OVERLAPS 相关条件
  -> 要求 non-immediate 不可用
  -> 拒绝 expression index
  -> 拒绝 partial index
  -> 拒绝 nullable key column
  -> relation_mark_replica_identity()
```
`relation_mark_replica_identity()` 更新 `pg_class.relreplident` 和 `pg_index.indisreplident`，并在需要时 `CacheInvalidateRelcache(rel)`。
`FULL` 表示完整旧行作为 replica identity。它不需要 index 定义，但 UPDATE / DELETE 可能把整行旧值写入 WAL 和 logical stream。对宽表、大 text/jsonb、toast 列，这个代价会从 WAL 带宽扩散到 decoding memory、network 和 subscriber apply CPU。
`NOTHING` 表示不记录 old identity。如果表发布 UPDATE 或 DELETE，普通 SQL 执行路径会被 `CheckCmdReplicaIdentity()` 拦住。典型错误是：
```text
cannot update table "..." because it does not have a replica identity and publishes updates
cannot delete from table "..." because it does not have a replica identity and publishes deletes
```
这说明“不输出 old key”不是 output plugin 的临时决定。对正常 publication DML，执行器会尽早拒绝没有定位语义的 UPDATE / DELETE。

## 6. 执行期先做第一道保护
UPDATE / DELETE 的第一道保护不在 `pgoutput.c`，而在 executor。`execReplication.c` 的 `CheckCmdReplicaIdentity()` 对 UPDATE / DELETE 做检查，普通 modify table 路径会在执行时触发相关逻辑。
它先构造：
```text
RelationBuildPublicationDesc(rel, &pubdesc)
```
然后检查三类 publication 约束：
```text
row filter:
  UPDATE / DELETE 用到的 row filter 列必须在 replica identity 中。
column list:
  publication column list 必须覆盖 replica identity 列。
generated columns:
  replica identity 中的 generated columns 必须被合法发布。
```
如果不满足，publisher 本地执行 UPDATE / DELETE 时就报错。典型错误包括：
```text
Column used in the publication WHERE expression is not part of the replica identity.
Column list used by the publication does not cover the replica identity.
Replica identity must not contain unpublished generated columns.
```
随后它检查 identity 本身：
```text
RelationGetReplicaIndex(rel)
  -> 有 replica identity index 或 primary key，则允许。
rel->rd_rel->relreplident == REPLICA_IDENTITY_FULL
  -> 也允许。
否则如果发布 UPDATE / DELETE
  -> ERROR。
```
这解释了 publication column list 和 row filter 为什么不是纯输出格式问题。下游定位时必须至少收到 replica identity 列。row filter 对 UPDATE / DELETE 还需要能在 old tuple 上判断旧行是否属于发布集合；如果 filter 引用了非 replica identity 列，而 UPDATE 没有 full old tuple，publisher 无法正确判断 UPDATE 应继续发送、丢弃，还是转换成 INSERT/DELETE。

## 7. heap UPDATE：什么时候记录 old key
`heap_update()` 在进入 critical section 前会先计算：
```text
hot_attrs
sum_attrs
key_attrs
id_attrs
interesting_attrs
```
其中 `id_attrs` 来自：
```text
RelationGetIndexAttrBitmap(relation, INDEX_ATTR_BITMAP_IDENTITY_KEY)
```
随后调用 `HeapDetermineColumnsInfo(..., id_attrs, ..., &id_has_external)`。它不只判断哪些列被修改，还判断未修改的 replica identity key 列中是否有 external TOAST datum。注释给出原因：如果未修改的 replica identity key 是 external toast，new tuple 中不会 WAL log flattened value，所以必须把它放进 old key tuple。
然后 `heap_update()` 调用：
```text
old_key_tuple = ExtractReplicaIdentity(
  relation,
  &oldtup,
  bms_overlap(modified_attrs, id_attrs) || id_has_external,
  &old_key_copied
)
```
这个 `key_required` 条件非常重要。UPDATE 不总是记录 old key。它只在两类情况下需要 old key：
```text
replica identity key columns changed
replica identity key columns include external toast value that new tuple cannot carry fully
```
如果普通 UPDATE 只改非 key 列，old key 可以省略。下游 apply 会用 new tuple 里的 key 定位旧行。这就是为什么你可能看到 UPDATE message 只有 `N`：key 没变，new tuple 中的 replica identity 值足以定位。
`ExtractReplicaIdentity()` 的规则是：
```text
relation not logically logged:
  return NULL
REPLICA_IDENTITY_NOTHING:
  return NULL
REPLICA_IDENTITY_FULL:
  return full old tuple
  如果含 external toast，先 toast_flatten_tuple()
DEFAULT / INDEX:
  如果 key_required 为 false，return NULL
  否则构造只包含 replica identity columns 的 tuple
  非 key 列置 NULL
  如果 key tuple 仍有 external toast，flatten
```
然后 `log_heap_update()` 写 WAL。当 `need_tuple_data = walLogical && RelationIsLogicallyLogged(reln)` 为真，它设置：
```text
XLH_UPDATE_CONTAINS_NEW_TUPLE
```
如果 `old_key_tuple` 非空，再根据 relation 当前 identity 设置：
```text
REPLICA_IDENTITY_FULL:
  XLH_UPDATE_CONTAINS_OLD_TUPLE
DEFAULT / INDEX:
  XLH_UPDATE_CONTAINS_OLD_KEY
```
注意变量名：`old_key_tuple` 在 FULL 情况下也可能指向完整 old tuple。真正写入 WAL 的语义在 flags，而不是变量名。

## 8. heap DELETE：没有 new tuple，所以必须尝试 old identity
DELETE 没有 new tuple。如果下游要定位目标行，只能依赖 old key 或 full old tuple。所以 `heap_delete()` 在进入 critical section 前调用：
```text
old_key_tuple = walLogical
  ? ExtractReplicaIdentity(relation, &tp, true, &old_key_copied)
  : NULL
```
DELETE 固定传 `key_required = true`。但 `ExtractReplicaIdentity()` 仍可能返回 NULL：
```text
relation not logically logged
REPLICA_IDENTITY_NOTHING
DEFAULT 但没有 primary key
INDEX 但没有有效 replica identity index
```
写 WAL 时，如果 `old_key_tuple != NULL`：
```text
REPLICA_IDENTITY_FULL:
  XLH_DELETE_CONTAINS_OLD_TUPLE
DEFAULT / INDEX:
  XLH_DELETE_CONTAINS_OLD_KEY
```
如果调用者传入 `TABLE_DELETE_NO_LOGICAL`，还会设置：
```text
XLH_DELETE_NO_LOGICAL
```
`decode.c` 会直接跳过这类 delete。这是本节第一个异常边界：DELETE 没有 old identity，不一定是 decoder 丢了 tuple，可能是写入端本来就没有写 old identity。普通 publication DML 通常会被 executor 提前拦住；如果特殊调用路径或非 pgoutput 插件看到了没有 oldtuple 的 delete change，它必须把这类 change 当作不可定位变更处理。

## 9. decode.c：old key 和 full old tuple 进入同一个字段
`decode.c` 不理解 subscriber 如何定位。它只按 WAL flags 构造 reorder buffer change。
`DecodeUpdate()` 的主线是：
```text
DecodeUpdate(ctx, buf)
  -> 检查 target_locator.dbOid 是否等于 slot database
  -> FilterByOrigin()
  -> change->action = REORDER_BUFFER_CHANGE_UPDATE
  -> 如果 XLH_UPDATE_CONTAINS_NEW_TUPLE:
       DecodeXLogTuple(...) -> change->data.tp.newtuple
  -> 如果 XLH_UPDATE_CONTAINS_OLD:
       DecodeXLogTuple(...) -> change->data.tp.oldtuple
  -> ReorderBufferQueueChange(...)
```
`XLH_UPDATE_CONTAINS_OLD` 是：
```text
XLH_UPDATE_CONTAINS_OLD_TUPLE | XLH_UPDATE_CONTAINS_OLD_KEY
```
因此 `oldtuple` 字段只表示“WAL 中有某种 old image”，不告诉你是 `K` 还是 `O`。
`DecodeDelete()` 类似：
```text
DecodeDelete(ctx, buf)
  -> 如果 XLH_DELETE_NO_LOGICAL，return
  -> 检查 database 和 origin
  -> change->action = REORDER_BUFFER_CHANGE_DELETE
  -> 如果 XLH_DELETE_CONTAINS_OLD:
       DecodeXLogTuple(...) -> change->data.tp.oldtuple
  -> ReorderBufferQueueChange(...)
```
诊断时要用两条规则。`ReorderBufferChange.oldtuple == NULL` 不能直接归咎于 output plugin，要回到 `heapam.c` 看 relation 是否 logically logged、walLogical 是否打开、replica identity 是什么、UPDATE 是否真的改了 identity key、identity key 是否含 external toast、DELETE 是否被 `TABLE_DELETE_NO_LOGICAL` 标记。`oldtuple != NULL` 也不能直接说“有完整旧行”，它可能只是包含 key 列、其它列为 NULL 的 tuple。

## 10. ReorderBuffer：交给 plugin 前才重组 TOAST
`ReorderBufferQueueChange()` 把 change 挂到对应 transaction。真正调用 output plugin 在 commit replay：
```text
ReorderBufferProcessTXN()
  -> 打开 relation
  -> 跳过不 logically logged 的 relation
  -> 跳过 rewrite temp heaps
  -> 跳过 sequence
  -> 对用户 relation:
       ReorderBufferToastReplace()
       ReorderBufferApplyChange()
       必要时 ReorderBufferToastReset()
```
TOAST 对本节很关键。UPDATE 的 new tuple 可能含 external toast pointer。如果相应 toast chunk 在同一事务中被 WAL logged，`ReorderBufferToastReplace()` 会把 new tuple 中的 on-disk toast pointer 替换成内存中重组出来的 value。
但源码注释明确说 unchanged toast tuples 无法替换，所以仍然指向 on-disk toast data。这会传到 protocol 层。`logicalrep_write_tuple()` 遇到仍是 on-disk external datum 的 varlena 列，发送：
```text
LOGICALREP_COLUMN_UNCHANGED = 'u'
```
这不是值丢失，而是表示该列在这次 change 中没有发送新值，apply 端要保留本地已有值。
对 replica identity key 来说，如果 unchanged toast key 需要参与定位，heap 写入端会因为 `id_has_external` 记录 old key。`pgoutput_row_filter()` 还有专门处理：unchanged toasted replica identity columns 只在 old tuple 中 logged 时，会把它们复制到 new tuple 用于 row filter。TOAST marker 和 replica identity 共同保证：不发送大字段新值，不能破坏 UPDATE / DELETE 的定位和 row filter 判断。

## 11. pgoutput 的 relation schema message
`pgoutput` 在发送 DML 前要确保 downstream 已知道 relation schema。入口在 `maybe_send_schema()`：
```text
maybe_send_schema(ctx, change, relation, relentry)
  -> 判断当前 relation schema 是否已经发送
  -> streaming transaction 下按 top xid 单独记录 schema_sent
  -> 如果 publish_as_relid 是 ancestor，先发送 ancestor schema
  -> send_relation_and_attrs(relation, ...)
```
`send_relation_and_attrs()` 必要时先发送 user-created type message，然后调用：
```text
logicalrep_write_rel(ctx->out, xid, relation, columns, include_gencols_type)
```
`logicalrep_write_rel()` 写出的消息类型是：
```text
LOGICAL_REP_MSG_RELATION = 'R'
```
消息内容包括：
```text
RelationGetRelid(rel)
namespace
relation name
rel->rd_rel->relreplident
published attributes
```
`logicalrep_write_attrs()` 对每个发布列写 flags。replica identity flag 的规则是：
```text
replidentfull = (rel->rd_rel->relreplident == REPLICA_IDENTITY_FULL)
if replidentfull:
  每个发布列都带 LOGICALREP_IS_REPLICA_IDENTITY
else:
  idattrs = RelationGetIdentityKeyBitmap(rel)
  只有 idattrs 中的发布列带 LOGICALREP_IS_REPLICA_IDENTITY
```
所以 relation schema message 不只是列名表，它也是 subscriber 构造定位语义的输入。如果 relation message 中没有某个 key column，subscriber 的 `attkeys` 就不会包含它。正因如此，publisher 对 column list 有前置检查：发布 UPDATE / DELETE 时，column list 必须覆盖 replica identity。

## 12. 主流程源码 walkthrough：pgoutput_change 从 change 到 wire message
`pgoutput_change()` 是本节 output plugin 主入口：
```text
pgoutput_change(ctx, txn, relation, change)
  -> is_publishable_relation()
  -> get_rel_sync_entry(data, relation)
  -> 检查 pubactions
  -> 把 oldtuple/newtuple 放入 TupleTableSlot
  -> 必要时按 publish_via_partition_root 做 attrmap 转换
  -> pgoutput_row_filter()
  -> 必要时发送 BEGIN
  -> maybe_send_schema()
  -> logicalrep_write_insert/update/delete()
```
如果 publication 不发布某个 action，直接 return。DELETE 有一个额外边界：
```text
if (!change->data.tp.oldtuple)
{
  elog(DEBUG1, "didn't send DELETE change because of missing oldtuple");
  return;
}
```
这里没有 ERROR。普通用户能否执行这类 DELETE 已由 `CheckCmdReplicaIdentity()` 在更早阶段限制。到 `pgoutput` 这里，它只是面对一个不可编码成 logical replication DELETE 的 change。
对 UPDATE，`pgoutput_change()` 不要求 `oldtuple` 必须存在。它会把存在的 oldtuple 放入 `old_slot`，把 newtuple 放入 `new_slot`。如果 `oldtuple` 不存在，后续 protocol 写出时就是：
```text
UPDATE:
  N new tuple follows
```
`pgoutput_row_filter()` 可能改变 action。对 UPDATE，它会分别评估 old tuple 和 new tuple：
```text
old 不匹配，new 不匹配:
  drop change
old 不匹配，new 匹配:
  UPDATE -> INSERT
old 匹配，new 不匹配:
  UPDATE -> DELETE
old 匹配，new 匹配:
  UPDATE
```
column list 也在这里生效。`relentry->columns` 传给 `logicalrep_write_rel()` 和 `logicalrep_write_tuple()`，结果是 relation message 和 tuple message 都只包含被发布的列。但 UPDATE / DELETE 的 locator columns 必须包含在其中，否则下游无法定位。

## 13. K / O / N：当前协议的真实消息形态
当前源码中 message type 在 `logicalproto.h`：
```text
LOGICAL_REP_MSG_INSERT = 'I'
LOGICAL_REP_MSG_UPDATE = 'U'
LOGICAL_REP_MSG_DELETE = 'D'
LOGICAL_REP_MSG_RELATION = 'R'
```
DML message 内部还有 action byte。`INSERT` 的形态是：
```text
I
  relation id
  N
  new tuple
```
`UPDATE` 的形态是：
```text
U
  relation id
  optional K old key
  optional O old tuple
  N new tuple
```
`logicalrep_write_update()` 如果 `oldslot != NULL`：
```text
REPLICA_IDENTITY_FULL:
  send 'O'
  send old tuple
DEFAULT / INDEX:
  send 'K'
  send old key
```
然后总是发送：
```text
'N'
new tuple
```
`logicalrep_read_update()` 也按这个状态机解析：first action must be `K`、`O` or `N`；如果是 `K/O`，先读 old tuple，再要求下一个 action 是 `N`；如果一开始就是 `N`，`has_oldtuple = false`。
所以当前协议没有“`N?`”这种消息。`N` 是 new tuple action byte，它在 INSERT 和 UPDATE 中都表示 new tuple follows。
`DELETE` 的形态是：
```text
D
  relation id
  K old key
```
或：
```text
D
  relation id
  O old tuple
```
`logicalrep_write_delete()` 不接受没有 oldslot 的协议形态，也不会发送 `N`。因为 DELETE 没有 new tuple。

## 14. old key、old tuple 与不可定位 change
什么时候输出 old key？
```text
relation 是 DEFAULT 且有 non-deferrable primary key，
或 relation 是 INDEX 且有有效 replica identity index；
UPDATE 修改了 identity key 或 key 有 external toast；
DELETE 总是需要 old identity；
heap WAL flags 记录了 OLD_KEY；
pgoutput 最终没有被 publication / row filter 过滤掉。
```
协议表现是：
```text
UPDATE: K old key, N new tuple
DELETE: K old key
```
什么时候输出完整 old tuple？
```text
relation 是 REPLICA_IDENTITY_FULL；
heap 写入端需要 old identity；
UPDATE 或 DELETE 的 WAL flags 记录 OLD_TUPLE；
pgoutput 输出该 change。
```
协议表现是：
```text
UPDATE: O old tuple, N new tuple
DELETE: O old tuple
```
FULL 的 old tuple 会尽量 flatten external toast。FULL 可以让 row filter 引用更多旧列，也可以让 subscriber 在没有 key index 时用 full tuple 顺序扫描。
什么时候只剩不可定位 change？要区分 UPDATE 和 DELETE。
UPDATE 没有 old image 不一定不可定位。如果 replica identity key 没变，new tuple 中的 identity key 仍然是旧行 key。`worker.c` 的 `apply_handle_update()` 就是这样：
```text
slot_store_data(remoteslot, rel, has_oldtup ? &oldtup : &newtup)
```
只有当 key 变化却没有 old key、下游映射不到 key columns，或本地没有可用定位路径时，UPDATE 才会变成 apply 边界问题。
DELETE 没有 old image 则没有定位输入。`pgoutput` 直接不发送。custom output plugin 如果仍要输出，只能把它输出成自己协议里的“不可定位 delete”，不能伪装成 logical replication protocol 的普通 DELETE。

## 15. subscriber logical relation cache
Relation schema message 到达 subscriber 后，`proto.c` 先读成 `LogicalRepRelation`：
```text
logicalrep_read_rel()
  -> remoteid
  -> nspname / relname
  -> replident
  -> attributes
```
attribute flags 中的 replica identity 位会形成 `remoterel->attkeys`。接着 `relation.c` 的 `logicalrep_relmap_update()` 更新：
```text
LogicalRepRelMap[remoteid] = LogicalRepRelMapEntry
```
这个 entry 保存：
```text
remoterel:
  publisher 发来的 relation schema 和 attkeys。
localrel:
  subscriber 本地打开的 relation。
attrmap:
  本地列号到远端列号的映射。
updatable:
  本地 replica identity 是否足够 apply UPDATE / DELETE。
localindexoid:
  用于定位本地 tuple 的 index，如果有。
localrelvalid:
  本地 DDL invalidation 后是否需要重建。
```
`logicalrep_rel_open()` 是 apply 时的入口。如果 entry 还 valid，它按 `localreloid` 打开 relation；如果 pending invalidation 让 entry 失效，它按远端 schema/name 重新找本地 relation，重建 attrmap、updatable 和 localindexoid。
`logicalrep_rel_mark_updatable()` 的判断很保守：
```text
优先使用本地 replica identity key。
如果没有，fallback 到本地 primary key。
如果本地没有 key，而远端不是 REPLICA_IDENTITY_FULL:
  updatable = false。
```
然后它要求本地 key columns 必须能映射到远端 `attkeys`，否则 `updatable = false`。错误不是立刻抛出，而是在 apply UPDATE / DELETE 前由 `check_relation_updatable()` 抛出：
```text
publisher did not send replica identity column expected by the logical replication target relation
logical replication target relation "..." has neither REPLICA IDENTITY index nor PRIMARY KEY and published relation does not have REPLICA IDENTITY FULL
```
如果远端是 FULL，subscriber 还可以尝试 `FindUsableIndexForReplicaIdentityFull()`。没有可用 index 时，`FindReplTupleInLocalRel()` 可以走 `RelationFindReplTupleSeq()` 顺序扫描。FULL 提供的是完整比较输入，不保证高性能。

## 16. apply worker 如何真正定位行
`apply_handle_update()` 读协议：
```text
logicalrep_read_update(s, &has_oldtup, &oldtup, &newtup)
```
然后打开 relation cache entry：
```text
rel = logicalrep_rel_open(relid, RowExclusiveLock)
```
它先调用 `check_relation_updatable(rel)`，再构造 search tuple：
```text
slot_store_data(remoteslot, rel, has_oldtup ? &oldtup : &newtup)
```
这再次说明：UPDATE 没有 old tuple 时，new tuple 被当作定位输入。
真正查找在：
```text
FindReplTupleInLocalRel()
```
如果 `localindexoid` 有效：
```text
RelationFindReplTupleByIndex()
```
否则要求远端 `replident == REPLICA_IDENTITY_FULL`，然后：
```text
RelationFindReplTupleSeq()
```
找到本地行后，`slot_modify_data()` 用 new tuple 修改本地 tuple，再走 `ExecSimpleRelationUpdate()`。
`apply_handle_delete()` 类似，但它读的是：
```text
logicalrep_read_delete(s, &oldtup)
```
DELETE 总是用收到的 old tuple/key 构造 search tuple。如果找不到行，worker 进入 conflict reporting 路径。本节只需要记住：protocol 的 K/O/N 是定位输入，logical relation cache 决定这些输入能否映射到本地 relation，worker 决定用 index 还是 seqscan 找本地 tuple。

## 17. schema change 与 cache invalidation
发送端和接收端都有 relation cache，但它们解决的问题不同。
发送端 `pgoutput.c` 的 `RelationSyncEntry` 保存：
```text
schema_sent
streamed_txns
pubactions
row filter ExprState
tuple slots
publish_as_relid
attrmap
columns
entry_cxt
```
`init_rel_sync_cache()` 注册：
```text
CacheRegisterRelcacheCallback(rel_sync_cache_relation_cb, ...)
CacheRegisterSyscacheCallback(NAMESPACEOID, rel_sync_cache_publication_cb, ...)
```
relation invalidation 到来时，callback 不直接释放 entry 内部对象，只设置 `entry->replicate_valid = false`。原因是 invalidation 可能发生在 output plugin callback 内部，外层仍在使用 entry。下一次 `get_rel_sync_entry()` 再集中清理旧 tuple slot、attrmap、row filter executor state，并重建 publication actions、column list、row filter 和 schema_sent 状态。
接收端 `relation.c` 的 `LogicalRepRelMapEntry` 也有 invalidation callback。`logicalrep_relmap_invalidate_cb()` 把本地 relation 对应 entry 标记为 `entry->localrelvalid = false`。下一次 `logicalrep_rel_open()` 重新找本地 relation，重建 attrmap、updatable 和 localindexoid。
所以 schema change 的正确性来自两层：
```text
publisher:
  historic snapshot 下读取当时的 schema；
  relation message 变更后重新发送。
subscriber:
  relation message 更新 remote schema；
  local DDL invalidation 后重建本地映射和定位路径。
```
invalidation 管语义过期，不是并发互斥，也不是立即释放。

## 18. publication column list 与 row filter 的边界
column list 和 row filter 不是输出美化。它们会改变 UPDATE / DELETE 的合法性和最终 action。
column list 的影响有三层：
```text
relation message 只发送被发布列；
tuple message 只发送被发布列；
发布 UPDATE / DELETE 时，column list 必须覆盖 replica identity。
```
对 `REPLICA IDENTITY FULL`，当前源码还禁止使用 column list。因为 FULL 表示所有列都可能是定位比较的一部分；如果只发布部分列，FULL 的“完整旧行”语义就不成立。
row filter 的影响更像状态转换。对 INSERT，只评估 new tuple；对 DELETE，评估 old tuple；对 UPDATE，`pgoutput_row_filter()` 可能评估 old 和 new，并把 action 转成 INSERT、DELETE 或 drop。
这让 replica identity 再次变成前提。如果 UPDATE 没有 full old tuple，row filter 不能随意引用旧行任意列。所以 publisher 要求：
```text
用于 UPDATE / DELETE 的 row filter 列必须属于 replica identity。
```
FULL 是例外，因为 FULL 中所有列都属于 replica identity。这解释了一个常见疑问：我只是想过滤，不是想定位，为什么报 replica identity 错？答案是 UPDATE row filter 必须判断旧行是否在发布集合中，这个判断本身需要稳定旧值。

## 19. 错误路径与输出边界
本节最重要的异常路径有五类。
第一类：publisher 执行期拒绝。如果 relation 发布 UPDATE / DELETE，但没有 replica identity index 或 FULL，`CheckCmdReplicaIdentity()` 报错。这是最早、也最清晰的边界。
第二类：publication 定义和执行期约束不满足。例如 row filter 引用了非 replica identity 列、column list 没覆盖 replica identity、replica identity 中有未发布 generated column、FULL 却使用 column list。这些错误说明 output plugin 即使想发送也不能保证下游定位。
第三类：DELETE change 到 pgoutput 时缺 oldtuple。`pgoutput_change()` 记录 DEBUG1 并 return。消息没有发出。诊断时要看 server log 的 DEBUG1 级别，或用自定义 output plugin / gdb 检查 `change->data.tp.oldtuple`。
第四类：subscriber relation cache 认为不可更新。`check_relation_updatable()` 报错。常见原因是 publisher 没发 subscriber 本地 key 需要的列、本地表没有 replica identity / primary key、publisher 不是 FULL 且 subscriber 没有定位路径。
第五类：subscriber 能定位但找不到行。这不再是 replica identity 定义问题，可能是初始同步缺失、手工改了 subscriber、不同 origin 或事务顺序问题。这里要分清：
```text
没有 key:
  定位输入不存在或不足。
有 key但找不到行:
  定位输入存在，目标端数据状态不匹配。
```

## 20. 成本、资源与跨模块传播
replica identity 的成本不只影响网络包大小。
第一层在 WAL 写入端。`REPLICA IDENTITY FULL` 可能让 UPDATE / DELETE 写完整 old tuple；如果 old tuple 有 external toast，`ExtractReplicaIdentity()` 会 `toast_flatten_tuple()`。这会增加 foreground CPU、WAL bytes、WAL insert pressure、replication lag 和 slot retained WAL。
第二层在 reorder buffer。old tuple、new tuple、TOAST chunks、toast reconstruction 都进入 decoding memory accounting。FULL 让每条 UPDATE / DELETE 的 change 更大，更容易触发 `logical_decoding_work_mem` 相关压力和 spill。
第三层在 output plugin。`logicalrep_write_tuple()` 要遍历 published columns，按 text/binary 输出类型值，并处理 unchanged toast marker。row filter 还要构造 executor state、slot，每条 change 执行表达式。
第四层在 subscriber。如果有可用 replica identity index 或 primary key，apply 是 index lookup。如果远端 FULL 但本地没有可用 index，可能走顺序扫描：
```text
每条 UPDATE / DELETE 成本接近 O(local table rows)
```
因此 FULL 是 correctness fallback，不是性能建议。它常用于没有稳定 key 的表，但在高频 UPDATE / DELETE 表上要谨慎。

## 21. 观测与诊断入口
诊断 UPDATE / DELETE 输出问题时，按链路分层。
第一层：publisher 是否允许这条 DML。执行 SQL 时是否报：
```text
cannot update table ... because it does not have a replica identity
cannot delete from table ... because it does not have a replica identity
Column list used by the publication does not cover the replica identity.
Column used in the publication WHERE expression is not part of the replica identity.
```
如果是，问题在 executor/publication/replica identity 前置检查。
第二层：relation 的 identity 配置。SQL 侧可以看：
```sql
SELECT relreplident
FROM pg_class
WHERE oid = 'public.t'::regclass;
```
也可以用 `\d+` 查看 replica identity。对 INDEX 模式，还要确认 `pg_index.indisreplident` 对应的 index 仍有效。
第三层：logical stream 是否带 K/O/N。`pgoutput` 是 binary protocol，肉眼看不方便。如果要源码断点，优先打：
```text
heapam.c: ExtractReplicaIdentity
heapam.c: log_heap_update
decode.c: DecodeUpdate / DecodeDelete
pgoutput.c: pgoutput_change
proto.c: logicalrep_write_update / logicalrep_write_delete
```
观察：
```text
relation->rd_rel->relreplident
old_key_tuple
xlrec.flags
change->data.tp.oldtuple
oldslot != NULL
action byte K/O/N
```
第四层：subscriber relation cache。断点或日志看：
```text
logicalrep_relmap_update
logicalrep_rel_open
logicalrep_rel_mark_updatable
FindLogicalRepLocalIndex
check_relation_updatable
FindReplTupleInLocalRel
```
要分清：
```text
updatable = false:
  schema/key mapping 不足。
FindReplTupleInLocalRel returned false:
  key 有了，但本地数据没找到。
```
第五层：统计和 lag。`pg_stat_subscription` 能看到 apply worker 进度和错误后停止状态，`pg_stat_replication` 能看 publisher 到 subscriber 的 LSN 差距，`pg_stat_wal` 能看 WAL 量增加。但这些指标看不到某条 UPDATE 是否带 `K` 或 `O`。这类状态要靠协议解析、DEBUG 日志、gdb 断点或源码推理。

## 22. 常见误区
误区一：`oldtuple != NULL` 就是旧整行。实际它可能只是 old key tuple。full old tuple 只有 `REPLICA_IDENTITY_FULL` 才成立。
误区二：UPDATE 没有 old tuple 就无法 apply。如果 identity key 没变，new tuple 可以作为 search tuple。
误区三：`REPLICA IDENTITY DEFAULT` 等于 primary key。只有存在可用 non-deferrable primary key 时才成立；没有 primary key时 DEFAULT 等于没有 identity。
误区四：column list 只影响下游看到哪些列。对 UPDATE / DELETE，column list 必须覆盖 replica identity。
误区五：row filter 是 output plugin 末端过滤，不影响执行期。row filter 用于 UPDATE / DELETE 时，引用列必须满足 replica identity 约束。
误区六：`REPLICA IDENTITY FULL` 一定解决性能问题。FULL 解决定位信息不足，但可能引入 full old tuple WAL、toast flatten、decode memory、network 和 subscriber seqscan 成本。

## 23. 课堂实验
实验一：UPDATE 是否发送 old key。
```sql
CREATE TABLE ri_u(id int PRIMARY KEY, v text);
CREATE PUBLICATION p_ri_u FOR TABLE ri_u WITH (publish = 'update');
INSERT INTO ri_u VALUES (1, 'a');
UPDATE ri_u SET v = 'b' WHERE id = 1;
UPDATE ri_u SET id = 2 WHERE id = 1;
```
断点：
```text
ExtractReplicaIdentity()
log_heap_update()
DecodeUpdate()
logicalrep_write_update()
```
要求解释：第一条 UPDATE 的 `key_required` 是否为 false，`old_key_tuple` 是否为 NULL，protocol 是否只有 `N`；第二条 UPDATE 是否记录 old key，protocol 是否为 `K + N`。
实验二：DELETE 缺 identity 的边界。
```sql
CREATE TABLE ri_d(id int, v text);
CREATE PUBLICATION p_ri_d FOR TABLE ri_d WITH (publish = 'delete');
INSERT INTO ri_d VALUES (1, 'a');
DELETE FROM ri_d WHERE id = 1;
```
预期：publisher 本地报没有 replica identity 且发布 deletes。然后：
```sql
ALTER TABLE ri_d ADD PRIMARY KEY (id);
INSERT INTO ri_d VALUES (1, 'a');
DELETE FROM ri_d WHERE id = 1;
```
断点观察 `heap_delete()`、`ExtractReplicaIdentity(..., key_required=true)`、`XLH_DELETE_CONTAINS_OLD_KEY`、`logicalrep_write_delete() -> K`。
实验三：FULL 与 unchanged toast marker。
```sql
CREATE TABLE ri_full(id int, big text);
ALTER TABLE ri_full REPLICA IDENTITY FULL;
CREATE PUBLICATION p_ri_full FOR TABLE ri_full WITH (publish = 'update, delete');
INSERT INTO ri_full VALUES (1, repeat('x', 100000));
UPDATE ri_full SET id = 2 WHERE id = 1;
UPDATE ri_full SET big = repeat('y', 100000) WHERE id = 2;
```
源码观察：
```text
ExtractReplicaIdentity()
toast_flatten_tuple()
ReorderBufferToastReplace()
logicalrep_write_tuple()
LOGICALREP_COLUMN_UNCHANGED
```
要求解释：FULL 为什么可能发送 `O`；unchanged toast marker 为什么不是值丢失；FULL 对 WAL、reorder buffer 和 network 的放大在哪里。

## 24. 讨论题
1. 为什么 PostgreSQL 不让 output plugin 在输出 DELETE 时回表查旧行？
2. `UPDATE` 没有 `oldtuple` 时，apply worker 为什么仍然可能定位本地行？
3. 为什么 `REPLICA IDENTITY DEFAULT` 在没有 primary key 时不能发布 UPDATE / DELETE？
4. 为什么 column list 必须覆盖 replica identity，而不是让 subscriber 自己处理缺列？
5. 为什么 UPDATE row filter 可能把 UPDATE 转换成 INSERT 或 DELETE？
6. `REPLICA IDENTITY FULL` 为什么能作为 correctness fallback，却可能让 subscriber 退化到 seqscan？
7. 如果 subscriber 报 “publisher did not send replica identity column”，你会先查 publisher 的 WAL flags，还是查 relation message 和 attrmap？
8. schema invalidation callback 为什么只标记 entry invalid，而不是在 callback 中直接释放 tuple slot 和 exprstate？

## 25. 本节小结
本节主链路是：
```text
relation replica identity
  -> heap WAL old image decision
  -> DecodeUpdate / DecodeDelete 填充 oldtuple/newtuple
  -> ReorderBuffer commit replay 和 TOAST reconstruction
  -> pgoutput publication / row filter / column list
  -> RELATION + UPDATE / DELETE protocol K/O/N
  -> subscriber logical relation cache
  -> apply worker index lookup or seqscan
```
核心状态不是单个字段。`relreplident` 说明配置，`rd_replidindex` 和 identity bitmap 说明可用 key，WAL flags 说明实际写入了什么 old image，`change->data.tp.oldtuple` 只说明 decode 后有某种 old image，RELATION message 的 `attkeys` 说明 downstream 如何解释这些值。
ownership 和 cleanup 也分层。发送端 `RelationSyncEntry` 存在于 output plugin cache context，invalidation 只标记 `replicate_valid = false`，下一次 `get_rel_sync_entry()` 重建 slots、attrmap、row filter 和 column list。接收端 `LogicalRepRelMapEntry` 存在于 apply worker relation map context，relation message 更新 remote schema，local relcache invalidation 让 `localrelvalid = false`，下一次 `logicalrep_rel_open()` 重建本地映射。
错误路径要按层定位：
```text
publisher executor ERROR:
  没有 replica identity，或 publication filter/list 不满足 identity 约束。
pgoutput DEBUG return:
  DELETE 缺 oldtuple，无法编码成 logical replication DELETE。
subscriber updatable ERROR:
  relation message 和本地 key schema 不匹配。
subscriber missing row conflict:
  key 有了，但目标端数据状态不匹配。
```
`pg_stat_replication`、`pg_stat_subscription`、`pg_stat_wal` 能看到进度、lag 和 WAL 量，但看不到某条 UPDATE 是否带 `K` 或 `O`。这类状态要靠协议解析、DEBUG 日志、gdb 断点或源码路径推理。
可迁移的系统规律是：
```text
跨系统复制中的 UPDATE / DELETE 首先是稳定 identity 的传播问题。
identity 必须在数据还存在、metadata 仍可解释、过滤规则仍能判断时被捕获。
一旦写入端没有捕获，后面的 cache、plugin、protocol 和 apply worker 都只能暴露边界，不能补造语义。
```
哪些判断仍然依赖 workload：FULL 是否可接受取决于行宽、toast 比例、UPDATE / DELETE 频率、网络带宽、subscriber index 状态和 apply 延迟。
哪些判断依赖版本：本课的 protocol version、parallel apply、generated column publication、column list / row filter 约束，都基于 `/home/nail/postgres` master commit `0e1f1ed157e90741e12a3715909e1b2d71ff9344`。读其它版本资料时，要回到当前源码确认 `logicalproto.h`、`pgoutput.c`、`publicationcmds.c` 和 `relation.c` 的实际行为。
