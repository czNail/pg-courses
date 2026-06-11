# PostgreSQL Heap rewrite、CLUSTER 与持久化替换边界

## 课程定位

上一节已经讲清楚 FSM 和 VM 如何作为 heap storage 的辅助状态存在。
这一节继续向外走一层：
当一个表的物理内容需要整体重排、收缩、改写或改变持久化属性时，PostgreSQL 为什么不在原文件上原地覆盖？
前置知识：
- 已理解 relation fork、relfilenode、buffer、WAL-before-data 和 checkpoint/fsync 的基本边界。
- 已理解 heap tuple header、`xmin/xmax/t_ctid`、HOT/update chain 和 recently dead tuple。
- 已理解 TOAST、index rebuild、relcache invalidation、MVCC snapshot 的基本语义。
- 已理解普通 heap insert/update/delete 的 WAL 和 visibility map 维护。
本节唯一主问题：
为什么表重写要先创建一个新的物理 relation，再在 catalog/relmapper 边界上替换，而不是直接在旧 relfilenode 里边读边改？
本节围绕的核心矛盾：
表重写希望一次性改善物理布局、压缩 bloat、改变 tuple 形态、改变 tablespace/access method/persistence。
但原表 OID、权限、依赖、触发器、外键、SQL 名字和并发可见性都必须保持稳定。
如果原地覆盖，任何中途 ERROR、crash、取消、TOAST 指针错误、index TID 不一致或逻辑解码映射缺失，都可能留下半新半旧的表。
如果完全创建一张新表再改名，又会破坏用户依赖的逻辑身份。
PostgreSQL 的选择是：

```text
旧表 OID 保持为逻辑身份。
新 relfilenode 承接新物理内容。
复制完成后交换物理身份。
索引、TOAST、relcache、WAL、pending delete 在各自边界收尾。
```

学完本节，你应该能独立判断：
- 哪些命令会触发表级 heap rewrite，哪些只是 metadata change。
- `CLUSTER` 与 `ALTER TABLE` 重写在复制路径上有什么差异。
- 为什么 `CLUSTER` 复制 tuple 时不能用普通 `heap_insert()`。
- 为什么 live 和 recently dead tuple 都可能被复制。
- 为什么 dead tuple 仍然要被 `rewrite_heap_dead_tuple()` 看一眼。
- 为什么新 heap 在 copy 完成前不能被其他 backend 读取。
- 为什么原表 OID 不变，但 `relfilenode` 和物理路径会变化。
- TOAST 为什么有“按内容交换”和“按链接交换”两类边界。
- 为什么主表索引通常在 heap swap 后重建，而不是复制时同步插入。
- 逻辑解码为什么需要 old TID 到 new TID 的 rewrite mapping。
- ERROR/abort 时新文件、旧文件、catalog tuple 和 mapping file 分别怎样收尾。
- 哪些状态可以用 SQL 看到，哪些只能靠源码断点或日志推断。

源码基线：本课基于本地 `/home/highgo/postgres` 源码，分支 `master`，提交 `bd4bd30ce6a7f08e95390c3fa068f2bfbe9fcee8`。行号来自 `nl -ba <source-file>`；重点源码统一放在第 3 节的阅读顺序里。

这个基线没有 `src/backend/commands/cluster.c`。
本分支把原来常被称为 CLUSTER/VACUUM FULL 主体的代码放在 `src/backend/commands/repack.c`。
`repack.c:4` 的文件头也说明它负责 REPACK，曾称为 CLUSTER，`VACUUM FULL` 也复用这条路径。
因此本课读源码时用这个分支真实存在的 `repack.c`，同时保留 SQL 层 `CLUSTER` 的语义名称。

## 1. 本节在总主线中的位置

第 27 到 33 节一直在回答 heap 的局部问题：
一个 page 里 tuple 怎样放置。
一个 tuple version 怎样通过 header 表达 MVCC 状态。
HOT/pruning/FSM/VM 怎样在 page 周边维护辅助状态。
本节的问题是全表级的：
如果整个 heap 的物理形态都要换，边界在哪里？
典型触发入口有三类。
第一类是 `CLUSTER`。
它按 index 顺序重排 heap tuple，使物理顺序接近 index 顺序。
第二类是 `VACUUM FULL`。
它用同一类重写方式收缩表文件，把仍需保留的 tuple 写入新的紧凑 heap。
第三类是某些 `ALTER TABLE`。
例如改变列类型、重算 stored generated column、设置 access method、改变 logged/unlogged persistence、某些 default 改写。
这些入口表面不同，但底层问题相同：

```text
旧表已经有一套稳定逻辑身份。
新数据必须完整构造后才能被宣布为旧表的新物理内容。
中途失败必须看起来像什么都没发生。
```

这不是一个普通 copy 问题。
heap tuple 的物理 TID 会变。
index tuple 里保存的是 TID。
TOAST 指针里保存 toast relation OID 和 value OID。
logical decoding 可能记住 old relfilelocator 和 ctid 到 command id 的关系。
catalog cache 可能持有旧 relcache 描述。
predicate lock 可能落在旧 tuple/page 上。
因此 heap rewrite 的核心不是“复制所有行”。
核心是一个替换边界：

```text
copy phase: 构造新物理事实，但不对外承诺。
switch phase: 在 catalog/relmapper 上把新物理事实接到旧逻辑身份。
cleanup phase: 重建依赖状态，删除旧物理事实，通知缓存失效。
```

## 2. 核心矛盾与一句话运行模型

一句话运行模型：

```text
heap rewrite 用新 relfilenode 吸收不可中断的物理重构成本，用事务性 catalog swap 把成功结果接到旧 OID，用 pending delete 和 WAL/fsync 边界保证失败或 crash 后不会暴露半成品。
```

这里有两个身份。
逻辑身份是 OID。
用户权限、依赖、外键、触发器、规则、SQL 名字都围绕 OID 组织。
物理身份是 `RelFileLocator`，其中最直观的是 `relfilenode`。
磁盘文件路径、main/fsm/vm/init fork、buffer tag 都围绕物理身份组织。
表重写必须改变后者，但尽量保留前者。
这就是为什么 CLUSTER 注释说：

```text
new clustered table
swap relfilenumbers
preserve original table OID
```

如果直接重写旧 relfilenode，会遇到三个无法同时满足的要求。
第一，不能边写新布局边让旧 index 继续指向旧 TID。
第二，不能在失败后判断哪些 page 是旧布局，哪些 page 是新布局。
第三，不能让 crash recovery 同时相信旧 catalog 和新数据页。
如果直接创建新表再 rename，也有三个问题。
第一，原表 OID 会变化，依赖会断裂或要级联更新。
第二，权限、comments、statistics、publication、rule、trigger 等对象都要迁移。
第三，大量扩展和内部缓存依赖对象身份稳定。
因此 PostgreSQL 把问题拆开。

```text
数据页复制: 允许很慢，但不对外可见。
catalog 替换: 必须短、受锁保护、可回滚。
索引重建: 放在 heap 已经挂到旧 OID 之后。
旧存储删除: 延迟到事务 commit。
新存储删除: 延迟到事务 abort。
```

这个拆分是本节所有边界的根。

## 3. 核心文件分工与阅读顺序

推荐阅读顺序如下。
不要按文件名排序。
按“从 SQL 入口到物理替换”的时间顺序读。

| 顺序 | 文件 | 关键入口 | 本节关注点 |
| --- | --- | --- | --- |
| 1 | `src/backend/commands/repack.c` | `cluster_rel()` | `CLUSTER`、`VACUUM FULL`、`REPACK` 共用的顶层重写入口 |
| 2 | `src/backend/commands/repack.c` | `rebuild_relation()` | 创建 transient heap、复制、关闭 relation、调用 `finish_heap_swap()` |
| 3 | `src/backend/commands/repack.c` | `make_new_heap()` | 新 heap 的 catalog 身份、reloptions、TOAST 表创建 |
| 4 | `src/backend/commands/repack.c` | `copy_table_data()` | 决定扫描顺序、TOAST swap 策略、freeze cutoffs、调用 table AM copy |
| 5 | `src/backend/access/heap/heapam_handler.c` | `heapam_relation_copy_for_cluster()` | heap AM 如何扫描旧 heap、判断 tuple deadness、调用 rewriteheap |
| 6 | `src/backend/access/heap/rewriteheap.c` | `begin_heap_rewrite()`、`rewrite_heap_tuple()`、`end_heap_rewrite()` | 保留 visibility、修复 `t_ctid` 链、raw heap insert |
| 7 | `src/backend/commands/repack.c` | `swap_relation_files()` | 真正交换 relfilenode/tablespace/persistence/toast link |
| 8 | `src/backend/commands/repack.c` | `finish_heap_swap()` | reindex、drop transient relation、rename toast、清 missing attrs |
| 9 | `src/backend/commands/tablecmds.c` | `ATRewriteTable()` | `ALTER TABLE` 重写路径，使用普通 insert 和表达式重算 |
| 10 | `src/backend/catalog/storage.c` | `RelationCreateStorage()`、`RelationDropStorage()`、`smgrDoPendingDeletes()` | 成功/失败时新旧物理文件的事务性删除 |
| 11 | `src/backend/utils/cache/relcache.c` | `RelationSetNewRelfilenumber()`、`RelationAssumeNewRelfilelocator()` | 直接换 relfilenode 的另一种边界和 relcache 生命周期 |
| 12 | `src/backend/access/common/toast_internals.c` | `toast_save_datum()` | rewrite 中 TOAST pointer 的真实 toast OID 和 value OID 选择 |
| 13 | `src/backend/storage/smgr/bulk_write.c` | `smgr_bulk_write()`、`smgr_bulk_finish()` | 绕过 shared buffers 的批量写、WAL 和 fsync 边界 |
| 14 | `src/backend/access/heap/heapam.c` | `heap_freeze_tuple()`、`heap_insert()` | 普通 insert 与 rewrite insert 的 WAL/visibility 差异 |

这条阅读顺序对应三条主流程。
第一条是 `CLUSTER/VACUUM FULL`：

```text
cluster_rel()
  -> rebuild_relation()
  -> make_new_heap()
  -> copy_table_data()
  -> table_relation_copy_for_cluster()
  -> heapam_relation_copy_for_cluster()
  -> begin_heap_rewrite()
  -> rewrite_heap_tuple()
  -> end_heap_rewrite()
  -> finish_heap_swap()
  -> swap_relation_files()
  -> reindex_relation()
  -> performDeletion(transient heap)
```

第二条是 `ALTER TABLE`：

```text
ATRewriteTables()
  -> make_new_heap()
  -> ATRewriteTable()
  -> table_tuple_insert()
  -> finish_heap_swap()
```

第三条是 storage cleanup：

```text
RelationCreateStorage(register_delete=true)
  -> pending delete at abort
RelationDropStorage()
  -> pending delete at commit
smgrDoPendingDeletes(isCommit)
  -> physically unlink matching relfilenodes
```

这三条必须合起来读。
只读 `rewriteheap.c` 会误以为重写的核心只是 tuple copy。
只读 `repack.c` 会漏掉 visibility 和 logical decoding mapping。
只读 `storage.c` 会看不到为什么 pending delete 正好能兜住 swap 失败。

## 4. 关键状态与边界

本节先把状态分层。
第一层是逻辑 catalog 状态。
核心字段在 `pg_class`。
最重要的是：

```text
pg_class.oid
pg_class.relfilenode
pg_class.reltablespace
pg_class.relpersistence
pg_class.relam
pg_class.reltoastrelid
pg_class.relfrozenxid
pg_class.relminmxid
pg_class.relpages/reltuples/relallvisible/relallfrozen
pg_class.relrewrite
```

`oid` 是逻辑身份。
`relfilenode` 是物理文件身份。
`reltoastrelid` 把主 heap 指到 TOAST heap。
`relrewrite` 用于标识正在被重写的目标关系，在 toast rename/reset 等路径上会被清理。
第二层是 relcache 状态。
`RelationData` 里有 `rd_locator`、`rd_backend`、`rd_rel`、`rd_toastoid`、`rd_createSubid`、`rd_newRelfilelocatorSubid` 等。
本节尤其关注 `rd_toastoid`。
`src/include/utils/rel.h` 明确说这是 CLUSTER、rewriting ALTER TABLE 等场景的 hack。
当向新 heap 写 toast pointer 时，指针可能必须指向旧表真实 toast table 的 OID，而不是 transient heap 新建的 toast table OID。
第三层是物理 storage 状态。
`RelFileLocator` 决定磁盘文件名。
它不仅包括 main fork。
同一个 relation 还可能有：

```text
MAIN_FORKNUM
FSM_FORKNUM
VISIBILITYMAP_FORKNUM
INIT_FORKNUM
```

重写新 heap 时，新的 main fork 会被批量写入。
FSM/VM 通常不是逐 tuple 复制出来的事实。
后续 VACUUM、index-only scan、visibility map 维护会重新推进。
第四层是 rewrite-local 状态。
`rewriteheap.c` 的 `RewriteStateData` 是 backend-local。
关键字段包括：

```text
rs_old_rel
rs_new_rel
rs_bulkstate
rs_buffer
rs_blockno
rs_oldest_xmin
rs_freeze_xid
rs_cutoff_multi
rs_unresolved_tups
rs_old_new_tid_map
rs_logical_rewrite
rs_logical_mappings
rs_begin_lsn
```

这些状态不进 shared memory。
其他 backend 不会直接访问。
它们只在当前重写命令的 copy phase 中有效。
`rs_unresolved_tups` 和 `rs_old_new_tid_map` 不是性能缓存。
它们维护 update chain 正确性。
旧 heap 里 `t_ctid` 指向旧 TID。
新 heap 里 tuple 被写到新 TID。
如果把旧 `t_ctid` 原样复制，新 heap 的 version chain 会断。
第五层是 pending storage cleanup。
`storage.c` 里的 `PendingRelDelete` 链表放在 `TopMemoryContext`。
它不是 catalog 状态。
它描述事务结束时要删除哪些物理 storage。
字段语义是：

```text
rlocator: 哪个物理文件身份
procNumber: temp relation 后端号
atCommit=false: abort 时删
atCommit=true: commit 时删
nestLevel: 当前事务/子事务层级
```

这条链表是理解失败边界的关键。
新 relfilenode 创建后登记“abort 删”。
旧 relfilenode 被 drop 后登记“commit 删”。
这两个登记在 swap 前后交错存在，刚好覆盖成功和失败。

## 5. 为什么是新 relfilenode 而不是原地覆盖

先看一个直观事实。
`CLUSTER` 之后，表 OID 不应该变。
用户不会重新授权。
外键不会重新绑定。
视图和函数依赖不会改指向。
但物理路径通常会变。
这说明系统替换的是 storage，不是 relation object。
`repack.c` 的 `cluster_rel()` 注释把这个目标说得很清楚：
创建一个新的、已聚簇的表。
交换新旧表的 relfilenumber。
保留原表 OID。
这不是简单重命名。
`make_new_heap()` 创建的是一张真实 catalog relation。
它有自己的 OID。
名字类似 `pg_temp_<old_oid>`。
它复制旧 heap 的 tuple descriptor、owner、reloptions、access method、tablespace、persistence。
它通常不复制 defaults 和 constraints。
因为它只是 transient 承载体，最终会被删除。
如果旧 heap 有 TOAST 表，`make_new_heap()` 可能给新 heap 创建 TOAST 表。
这一步必须先做。
因为 copy phase 中大 tuple 可能要写外部 TOAST rows。
但是这张新 heap 还不是用户表。
它只是即将被填充的物理容器。
为什么不直接把旧 heap 文件改成新布局？
第一，旧 index 的 TID 指向旧 heap page。
只要 heap TID 变化，旧 index 就不能继续解释新 heap。
如果边改 heap 边改 index，中途失败会有两个方向的不一致。
第二，旧 TOAST pointer 可能指向旧 toast relation 和 value id。
rewrite 中可能要保留、复用或重新生成 TOAST value。
如果原地覆盖，重复 TOAST value、已删除版本、recently dead 版本之间的关系会更难回滚。
第三，旧 visibility 信息不能被普通 insert 重写。
`heap_insert()` 会写当前事务的 `xmin/cmin`。
`CLUSTER/VACUUM FULL` 需要保留被复制 tuple 的 visibility 事实。
第四，crash recovery 需要一个清楚的 redo 边界。
新文件的 page 可以通过 WAL 或 fsync 边界保证。
旧文件直到 commit 前都应保持可恢复。
第五，ERROR/abort 必须简单。
如果重写失败，系统只需要丢掉新 relfilenode。
旧 heap 不需要“反向修复”。
因此核心替换模式是：

```text
old OID -> old relfilenode  对外稳定
new OID -> new relfilenode  当前事务私有承载
copy tuples into new relfilenode
swap pg_class/relmapper physical identity
old OID -> new relfilenode  对外成为新事实
new OID -> old relfilenode  随 transient relation 删除
```

注意这里的“swap”不是一个 CPU 原子指令。
它是受 `AccessExclusiveLock`、catalog update、relcache invalidation、transaction commit 共同保护的替换边界。
其他 backend 在锁释放和 invalidation 之后看到的是新 relfilenode。
在此之前，它们不能并发写入或读取正在被改写的目标 relation。

## 6. `CLUSTER/VACUUM FULL` 主流程

`CLUSTER` 入口最终进入 `cluster_rel()`。
这个函数先确定锁级别。
普通非并发路径使用能阻塞读写的强锁。
然后它做几类前置检查。
包括：

```text
relation 是否仍然符合重写条件
index 是否可用于 clustering
是否是 other backend 的 temp table
当前事务是否还有 active scan 或 pending trigger
materialized view 是否已 populated
```

`CheckTableNotInUse()` 很关键。
它防止同一事务内部已经持有的 scan 或 AFTER trigger 状态在 tuple 移动后继续引用旧物理位置。
普通非并发 `CLUSTER` 还会调用 `TransferPredicateLocksToHeapRelation()`。
因为 tuple/page 级 predicate lock 会随 tuple 移动失效。
把它们提升到 relation 级，是一种保守正确性边界。
之后进入 `rebuild_relation()`。
这一步先记住旧表的 access method、tablespace 和 persistence。
如果指定 index，会调用 `mark_index_clustered()` 标记聚簇 index。
然后调用：

```text
OIDNewHeap = make_new_heap(...)
NewHeap = table_open(OIDNewHeap, NoLock)
copy_table_data(NewHeap, OldHeap, index, ...)
finish_heap_swap(...)
```

`copy_table_data()` 做三件事。
第一，决定 TOAST 策略。
如果旧表和新表都有 TOAST 表，且不是 concurrent 路径，就设置 `swap_toast_by_content=true`，并把 `NewHeap->rd_toastoid` 指向旧表 TOAST OID。
第二，计算 aggressive freeze cutoffs。
既然全表都要重写，系统顺手冻结足够老的 XID/MultiXact。
但 `relfrozenxid` 和 `relminmxid` 不允许倒退，所以会和旧表的值取更保守边界。
第三，决定扫描顺序。
如果有 btree index，planner 可以决定使用 index scan，或 seqscan 加 sort。
否则按物理顺序复制。
真正复制由 table AM callback 完成：

```text
table_relation_copy_for_cluster()
  -> heapam_relation_copy_for_cluster()
```

heap AM 在这里分两类路径。
非 concurrent 的 CLUSTER/VACUUM FULL 使用 `rewriteheap.c`。
concurrent REPACK 路径使用普通 `heap_insert()` 和 logical decoding catch-up，本节只把它作为旁路。
普通路径的核心调用是：

```text
begin_heap_rewrite()
for each tuple:
  HeapTupleSatisfiesVacuum()
  if dead:
    rewrite_heap_dead_tuple()
  else:
    reform_and_rewrite_tuple()
      -> rewrite_heap_tuple()
end_heap_rewrite()
```

这说明 `CLUSTER` 不是只复制当前 snapshot 可见的 tuple。
它复制所有仍需保留 MVCC 语义的 tuple version。
包括 live。
也包括 recently dead。
还包括某些 insert/delete in progress 状态。
完全 dead 的 tuple 不复制，但仍会传给 `rewrite_heap_dead_tuple()`，用于释放可能已经暂存的 update chain 端点。
复制完成后，`finish_heap_swap()` 做替换。
它先调用 `swap_relation_files()`。
然后根据需要 `CacheInvalidateCatalog()`。
然后 `reindex_relation()` 重建主表索引。
然后 `performDeletion()` 删除 transient heap。
最后在 toast link swap 场景下 rename toast table/index，并清理 missing attribute 设置。
从用户角度看，命令完成后原表 OID 还在。
但 `pg_relation_filepath()` 和 `pg_class.relfilenode` 通常已经不同。

## 7. `ALTER TABLE` 重写主流程

`ALTER TABLE` 重写由 `tablecmds.c` 管理。
它和 `CLUSTER` 共享 `make_new_heap()` 和 `finish_heap_swap()`，但复制 tuple 的方式不同。
`ATRewriteTables()` 在 phase 3 判断是否需要 rewrite。
典型原因包括：

```text
AT_REWRITE_DEFAULT_VAL
AT_REWRITE_COLUMN_REWRITE
AT_REWRITE_ACCESS_METHOD
AT_REWRITE_ALTER_PERSISTENCE
```

如果需要重写普通 heap，它会先打开旧 relation。
然后拒绝几类目标：

```text
system relation
used as catalog table
other session temp table
```

拒绝系统 catalog 的原因不是不能理论上复制。
而是 ALTER TABLE 的 schema rewrite 与 mapped catalog、catalog self-update、toast link 等边界组合太复杂，收益不值得。
确定 tablespace、access method、persistence 后，代码会触发 `table_rewrite` event trigger。
然后调用：

```text
OIDNewHeap = make_new_heap(...)
ATRewriteTable(tab, OIDNewHeap)
finish_heap_swap(..., RecentXmin, ReadNextMultiXactId(), persistence)
```

`ATRewriteTable()` 的工作是表达式级改写。
它用 executor state 准备 new values、generated expression、check constraint、not null、partition qual。
如果传入了 `OIDNewHeap`，它为旧 tuple 和新 tuple 分别建 slot。
循环中做：

```text
从 oldslot 取全部属性
复制到 newslot
把 dropped attributes 置 NULL
计算 default/generated/type conversion 表达式
验证 not null/check/partition constraint
table_tuple_insert(newrel, insertslot, mycid, TABLE_INSERT_SKIP_FSM, bistate)
```

这和 `CLUSTER` 很不一样。
`ALTER TABLE` rewrite 生成的是当前命令插入的新 tuple。
普通 `table_tuple_insert()` 会设置当前事务的 visibility 信息。
所以完成后可以把新表 `relfrozenxid` 设置为 `RecentXmin`。
它不需要保留旧 tuple 的 `xmin/xmax/t_ctid` 链。
为什么可以这样？
因为 `ALTER TABLE` 持有强锁，目标表在这期间没有并发写入。
它重写的是当前命令语义下的一张新版本表。
旧的历史 tuple version 不需要被继续暴露给并发 snapshot。
与此相对，`CLUSTER/VACUUM FULL` 的实现保留了更底层的 heap visibility 事实。
这就是同样“新 relfilenode 再 swap”，但 copy path 不同的原因。
还有一个容易混淆的点：
`ALTER TABLE ... CLUSTER ON index` 不是物理重写。
`tablecmds.c` 的 `ATExecClusterOn()` 只检查 index 是否可 cluster，然后修改 `pg_index.indisclustered`。
真正物理重写发生在 SQL `CLUSTER` 命令，走 `cluster_rel()`。

## 8. Tuple visibility 与 update chain

`rewriteheap.c` 开头的注释已经把难点讲出来：
完全重写 heap 本身不难，难的是保留 visibility 信息和 update chain。
旧 heap 的 version chain 由 `t_ctid` 连接。
如果 tuple A 被更新成 tuple B，A 的 `t_ctid` 指向 B 的旧 TID。
重写后 B 的新 TID 变了。
A 的 `t_ctid` 必须改成 B 的新 TID。
但是扫描顺序不保证先遇到 A 还是先遇到 B。
所以 `rewrite_heap_tuple()` 使用两个 hash table。
如果先遇到 A：

```text
A.t_ctid 指向 B 的旧 TID。
B 还没有被写入新 heap。
把 A 放入 rs_unresolved_tups。
等遇到 B 后，用 B 的新 TID 修正 A.t_ctid，再写 A。
```

如果先遇到 B：

```text
B 被写入新 heap。
记录 old B TID -> new B TID 到 rs_old_new_tid_map。
以后遇到 A 时直接修正 A.t_ctid。
```

hash key 不只用 TID。
它还包含 `xmin`。
原因是旧 TID 可能被 dead tuple 或 page reuse 场景干扰。
`TidHashKey` 把 `xmin + old tid` 作为匹配语义，降低误配风险。
`rewrite_heap_tuple()` 还会复制 tuple 的 `HeapTupleFields`。
它不让 `heap_insert()` 重写 visibility。
它会清理并保留事务相关 infomask 位。
它会调用 `heap_freeze_tuple()` 对足够老的 `xmin/xmax/MultiXact` 进行无 WAL 的 in-memory freeze。
为什么无 WAL？
因为这些 tuple 还没有进共享 buffer。
最终新 heap page 会由 bulk writer 作为新 page WAL 记录或 fsync 边界写出。
`heapam_relation_copy_for_cluster()` 使用 `SnapshotAny` 扫旧 heap。
它再用 `HeapTupleSatisfiesVacuum(tuple, OldestXmin, buf)` 判定 tuple 状态。
这里会锁旧 buffer 为 exclusive。
注释说原因很直接：
需要保证 hint bits 被设置。
否则 `rewrite_heap_tuple()` 可能因 `HEAP_XMAX_INVALID` 等判断缺失而错误处理 update chain。
这也是一个不漂亮但现实的边界。
运行 `VACUUM FULL` 这种本来为了处理 bloat 的命令，仍可能写旧表 hint bits。
因为 correctness 比避免旧页写放大更重要。
tuple 状态处理如下：

```text
HEAPTUPLE_DEAD:
  不复制，但调用 rewrite_heap_dead_tuple()
HEAPTUPLE_RECENTLY_DEAD:
  复制，并计入 tups_recently_dead
HEAPTUPLE_LIVE:
  复制
HEAPTUPLE_INSERT_IN_PROGRESS:
  通常是当前事务或 system catalog 特殊情况，按 live 复制
HEAPTUPLE_DELETE_IN_PROGRESS:
  按 recently dead 复制
```

`rewrite_heap_dead_tuple()` 的存在容易被忽视。
dead tuple 不进入新 heap，但它可能是某个 unresolved chain 的 B 端。
如果之前暂存了一个 recently dead A，后来发现 B 已经 dead，那么 A 也可以释放，不必写入新 heap。
这让 `tups_vacuumed` 和 `tups_recently_dead` 的统计更接近真实。
`end_heap_rewrite()` 会处理残留 unresolved tuple。
如果还有未解决项，它把 `t_ctid` 设 invalid，然后调用 `raw_heap_insert()`。
注释说这类情况通常意味着 tuple 事实上已经 dead，只是判定不够精确。
系统选择偏保守地写出，而不是冒险丢掉仍可能需要的版本。
这就是 visibility 边界的风格：

```text
能确定 dead: 不复制。
不能确定 dead: 复制并保持链路尽量正确。
链路无法完全解析: 保守写出。
```

## 9. `raw_heap_insert()` 与普通 `heap_insert()` 的差异

`CLUSTER/VACUUM FULL` 非 concurrent 路径不能用普通 `heap_insert()`。
原因不是性能，而是语义。
普通 `heap_insert()` 会把 tuple 变成“当前事务插入的新 tuple”。
它会设置当前 `xmin/cmin`。
它会走 buffer manager。
它会写 per-tuple heap WAL。
它会维护 VM clear、logical tuple data 等普通 DML 边界。
`rewriteheap.c` 需要的是：

```text
把一个已经有 MVCC 历史的 tuple version 原样搬到新 page。
只在必要时修正 t_ctid 和 freeze bits。
```

所以 `raw_heap_insert()` 直接构造 page。
它维护 `rs_buffer` 和 `rs_blockno`。
当前 page 放不下时，调用 `smgr_bulk_write()` 写出 page。
最后 `end_heap_rewrite()` 写出最后一页并 `smgr_bulk_finish()`。
`raw_heap_insert()` 仍然会处理 TOAST。
如果 tuple 太大，或含有来自其他 relation 的 external toast pointer，它调用：

```text
heap_toast_insert_or_update(state->rs_new_rel, tup, NULL, options)
```

其中 options 包含：

```text
HEAP_INSERT_SKIP_FSM
HEAP_INSERT_NO_LOGICAL
```

TOAST rows 通过正常 heap/bufmgr 路径写入 TOAST relation。
这和主 heap page 的 bulk writer 路径不同。
所以本节不能把“rewrite 绕过 shared buffers”理解成所有数据都绕过。
准确说：

```text
主 heap page 使用 raw heap insert + smgr bulk write。
TOAST table entries 仍走普通 heap insert 和 toast index insert。
```

bulk writer 的边界在 `bulk_write.c`。
它绕过 buffer manager，直接 `smgrextend()` 或 `smgrwrite()`。
好处是避免大量 buffer lock 和 shared buffer 污染。
代价是第一次访问新 heap 时需要重新读入 shared buffers。
如果需要 WAL，`smgr_bulk_flush()` 会用 `log_newpages()` 把多个 page 合并进 WAL。
`smgr_bulk_finish()` 还要处理 fsync 或 checkpoint 注册。
如果 WAL 正常记录了新 page，但 bulk write 期间发生 checkpoint，可能出现 checkpoint redo pointer 之后不再重放早期 WAL 的窗口。
因此代码记录 `start_RedoRecPtr`。
如果发现 checkpoint 已经推进，必要时直接 `smgrimmedsync()`。
这就是 rewrite 的 WAL/fsync 边界：

```text
不是每个 tuple 一条 WAL。
而是新 page 批量 WAL 或新 relation commit-time sync。
还要处理绕过 buffer manager 后与 checkpoint 的互锁。
```

## 10. TOAST 边界：按内容交换与按链接交换

TOAST 是 heap rewrite 中最容易出错的边界之一。
主 heap tuple 里的 external datum 指针包含 toast relation OID 和 toast value OID。
rewrite 后主 heap 的物理位置变了，但 TOAST pointer 仍必须指向能找到数据的 toast relation。
`copy_table_data()` 有两种策略。
第一种是 swap by content。
条件是旧表和新表都有 TOAST 表，且不是 concurrent 路径。
这时设置：

```text
NewHeap->rd_toastoid = OldHeap->rd_rel->reltoastrelid
```

含义是：
往 NewHeap 写 toast pointer 时，pointer 里放旧表真实 toast OID。
等最后递归交换 TOAST 表内容后，这些 pointer 仍然有效。
`toast_save_datum()` 看到 `rd_toastoid` 后，也会尝试保留旧 toast value OID。
如果旧 live/recently dead 多个 heap tuple 指向同一个 toast value，新 TOAST 表只写一份。
源码中特意处理了这个 corner case。
如果 value id 已经存在，就跳过重复写 data chunks。
这避免了最近死亡版本带来的 toast bloat。
第二种是 swap by links。
如果不能或不需要按内容交换，就交换主表 `pg_class.reltoastrelid` 链接。
然后更新 dependency。
这种方式可以处理旧表有 toast、新表没有 toast，或新旧 TOAST 形态不同的情况。
但是系统 catalog 不能走这种路径。
因为修改 catalog 自己的 TOAST dependency 太危险。
`swap_relation_files()` 对系统 catalog 做了 backstop 检查。
如果按内容交换 TOAST 表，还要交换 TOAST 表的 valid index。
`swap_relation_files()` 会调用 `toast_get_valid_index()` 找到双方 TOAST index，然后递归交换 index relation files。
注意这里不是“主表索引重建”的同一件事。
TOAST index 是 TOAST 表内部查 chunk 用的索引。
它必须和 TOAST 表内容一起保持一致。
主表普通索引则在 `finish_heap_swap()` 中通过 `reindex_relation()` 重建。
`ALTER TABLE` rewrite 通常不关心系统 catalog 的 toast pointer 缓存稳定性。
它传给 `finish_heap_swap()` 的 `swap_toast_by_content=false`。
因此它走按链接交换并在必要时 rename toast table/index。

## 11. Index 边界：读旧 index，重建新 index

`CLUSTER` 使用 index 的方式容易误解。
给定 `CLUSTER t USING idx`，旧 index 只用于决定扫描顺序。
它不是被“同步维护”成新 heap 的 index。
如果 planner 认为 btree index scan 不如 seqscan + sort，`copy_table_data()` 还可以选择 sort。
因此物理排序来源有两种：

```text
index scan old heap in index order
seqscan old heap, then tuplesort_begin_cluster()
```

无论哪种，新 heap 的主表索引都会在 swap 后重建。
`finish_heap_swap()` 里调用：

```text
reindex_relation(NULL, OIDOldHeap, reindex_flags, &reindex_params)
```

为什么不在复制每个 tuple 时向新 index 插入？
第一，新 heap 的 TID 是新生成的，所有旧 index TID 都失效。
第二，bulk-loading heap 后再建 index 通常比逐 tuple 维护 index 更便宜。
第三，swap 前新 heap 不是用户可见的目标 relation。
第四，系统 catalog 场景要求在 drop transient relation 前先让目标 catalog index 可用。
`finish_heap_swap()` 的注释强调了这个顺序。
如果正在处理系统 catalog，DROP transient relation 可能需要使用 catalog index。
所以先 reindex，再 drop。
`reindex_relation()` 经由 index build 扫描新 heap。
此时原表 OID 已经接到了新 relfilenode。
index 建出来的 TID 指向新 heap。
源码注释还说这里不会设置 `indcheckxmin`。
因为新 heap 不包含 HOT chains。
这句话不要理解成没有任何 version chain。
rewrite 会保留必要的 `t_ctid` 语义，但会清理 HOT status bits。
索引重建时不需要因 broken HOT chain 延迟可用性。
`ALTER TABLE ... CLUSTER ON` 只改 `indisclustered`。
它不会触发这个 rebuild。
真正的 rebuild 在 `CLUSTER`、`VACUUM FULL`、某些 `ALTER TABLE` rewrite 和 refresh matview 等路径里。

## 12. Catalog swap 与 relcache invalidation

`swap_relation_files()` 是替换边界的核心。
普通非 mapped relation 的分支会交换：

```text
relfilenode
reltablespace
relam
relpersistence
reltoastrelid   -- 仅 swap by links 时
```

它还会设置目标表新的：

```text
relfrozenxid
relminmxid
relpages
reltuples
relallvisible
relallfrozen
```

这说明 swap 不只是换文件名。
它还把新 heap 的统计和冻结边界转移到原表逻辑身份上。
如果 relation 是 mapped relation，`pg_class.relfilenode` 无效。
这时不能直接改 `pg_class.relfilenode`。
必须通过 relmapper：

```text
RelationMapUpdateMap(r1, relfilenumber2, ...)
RelationMapUpdateMap(r2, relfilenumber1, ...)
```

mapped relation 的边界更脆弱。
例如 `pg_class` 自身的重写不能靠更新它即将丢弃的旧 tuple 来完成关键变化。
所以 `finish_heap_swap()` 对 `OIDOldHeap == RelationRelationId` 有额外处理。
在新 `pg_class` index 可用后，再更新 `relfrozenxid/relminmxid`。
swap 后还要处理 relcache 状态。
`swap_relation_files()` 会打开 r1 和 r2 的 relcache entry。
把 rel2 的 create/new relfilelocator subid 信息调整为 rel1 的状态。
然后对 rel1 调用 `RelationAssumeNewRelfilelocator()`。
`RelationAssumeNewRelfilelocator()` 的注释很重要：
修改 `pg_class.reltablespace` 或 `pg_class.relfilenode` 的代码必须调用它。
调用位置应靠近使 catalog 变化可见的 `CommandCounterIncrement()`。
原因是后续 WAL replay 需要知道哪些修改属于新的 physical locator。
catalog update 通过 `CatalogTupleUpdateWithInfo()` 维护 `pg_class` indexes。
如果目标是 `pg_class` 自己，不能更新即将被扔掉的旧 `pg_class` 数据。
这时会手工发 relcache invalidation。
系统 catalog 还可能调用 `CacheInvalidateCatalog(OIDOldHeap)`。
这会在 CCI 时让相关 catcache 失效。
因此替换边界有四层：

```text
disk: 新 relfilenode 的 bytes 已经写好
catalog/relmapper: old OID 指向新物理身份
relcache/catcache: 本地和其他 backend 的缓存收到失效
transaction: commit 后删除旧物理文件，释放锁
```

只看其中任何一层都不完整。

## 13. Storage cleanup：失败为什么不破坏旧表

失败清理的关键在 `storage.c`。
`RelationCreateStorage()` 创建新 storage 时，如果 `register_delete=true`，会登记：

```text
atCommit = false
```

意思是：
如果事务 abort，删除这个新 storage。
`make_new_heap()` 内部通过 catalog creation 创建新 relation storage。
因此 copy phase 期间如果 ERROR，pending delete 会在 abort 清理新 relfilenode。
旧表 relfilenode 没被改，不需要恢复。
`RelationDropStorage()` 做相反的事。
它登记：

```text
atCommit = true
```

意思是：
如果事务 commit，删除这个 storage。
如果事务 abort，不删。
`performDeletion()` 删除 transient heap 时，会把 transient heap 当前持有的物理 storage 登记为 commit 删除。
注意 swap 后 transient heap 持有的正是旧表原来的 relfilenode。
于是成功路径是：

```text
新 relfilenode 的 abort-delete entry 被忽略。
旧 relfilenode 的 commit-delete entry 被执行。
```

失败路径是：

```text
catalog swap 回滚。
旧表 OID 重新指向旧 relfilenode。
旧 relfilenode 的 commit-delete entry 不执行。
新 relfilenode 的 abort-delete entry 执行。
```

这就是为什么 swap 后再 ERROR 也不会把旧表物理文件删掉。
真正执行删除的是：

```text
smgrDoPendingDeletes(isCommit)
```

它遍历 `pendingDeletes`。
只删除 `pending->atCommit == isCommit` 的 relfilenode。
子事务 abort 时也会立即处理当前 nest level 的 pending deletes。
这让局部失败能及时清理物理文件。
这个机制也解释了为什么不要把 catalog row delete 等同于文件立即 unlink。
PostgreSQL 延迟物理 unlink，正是为了让事务语义和 crash recovery 有清晰边界。
`RelationPreserveStorage()` 是另一个相关边界。
relation mapping 的变化可能独立于整体事务提交。
某些路径需要把本来登记的 pending delete 移除。
本节不展开 relmapper commit 协议，但要记住：
mapped relation 的文件保留/删除比普通 pg_class relfilenode swap 更敏感。

## 14. WAL、logical rewrite 与 crash 边界

heap rewrite 的 WAL 不只有一种。
第一种是 storage creation WAL。
`RelationCreateStorage()` 对 permanent relation 调用 `log_smgrcreate()`。
这写 `XLOG_SMGR_CREATE`。
如果 `XLogIsNeeded()` 为 false，永久 relation 可能走 pending sync 边界。
第二种是新 heap page WAL。
`raw_heap_insert()` 把 tuple 放到本地 page。
`smgr_bulk_write()` 排队写 page。
`smgr_bulk_flush()` 如果 `use_wal`，用 `log_newpages()` WAL-log 一批新 page。
这不是普通 heap insert WAL。
第三种是 TOAST row 的普通 heap WAL。
TOAST 通过 `heap_insert()` 写 toast relation。
它有自己的 heap insert WAL 和 toast index insertion。
但 rewrite 会传 `HEAP_INSERT_NO_LOGICAL`，避免这些内部数据被逻辑解码当成用户变更输出。
第四种是 logical rewrite mapping WAL。
`rewriteheap.c` 的 logical support 针对 catalog relation 和 logical decoding。
逻辑解码可能依赖 `(relfilelocator, ctid) -> (cmin, cmax)` 的关系。
heap rewrite 改变了 relfilelocator 和 ctid。
所以如果表可被逻辑解码访问，并且存在 logical slot xmin，系统要记录 old locator/TID 到 new locator/TID 的映射。
这条路径从：

```text
logical_begin_heap_rewrite()
logical_rewrite_heap_tuple()
logical_rewrite_log_mapping()
logical_end_heap_rewrite()
```

`logical_begin_heap_rewrite()` 会先判断：

```text
RelationIsAccessibleInLogicalDecoding(old_rel)
ProcArrayGetReplicationSlotXmin(...)
```

如果没有相关 logical slot，就不做额外工作。
mapping 文件放在 `pg_logical/mappings`。
文件名包含 database oid、relation oid、rewrite start LSN、mapped xid、执行 rewrite 的 xid。
这样 abort 后可以根据事务提交状态忽略无效 mapping。
这里的 crash safety 很特别。
注释明确说它偏离普通 WAL 模式。
mapping file 不在 shared buffers 里，不能依赖 buffer lock 和 checkpoint 互锁。
所以它先写 mapping file，再插入 `XLOG_HEAP2_REWRITE`。
WAL record 中带 offset。
redo 时可以把文件 truncate 到 WAL 记录认可的 offset。
`CheckPointLogicalRewriteHeap()` 负责 checkpoint 时清理或 fsync mapping files。
它还根据 logical restart LSN 删除不再需要的 mapping。
结论：

```text
主 heap 新 page 的 crash safety 由 bulk write WAL/fsync 保证。
TOAST rows 的 crash safety 由普通 heap/index WAL 保证。
logical decoding 的 ctid 映射由 mapping file + XLOG_HEAP2_REWRITE 保证。
旧 storage 是否删除由 pending delete 的 commit/abort 边界保证。
```

这些机制相邻，但互不替代。
不要把“有 WAL”笼统理解成所有边界都被同一条 WAL record 覆盖。

## 15. Lock、snapshot 与 visibility 正确性层次

表重写依赖多层正确性机制。
第一层是 heavyweight lock。
普通 `CLUSTER/VACUUM FULL/ALTER TABLE rewrite` 需要强锁。
它防止其他 backend 在 copy/swap 期间并发读写目标 relation。
它不负责 crash safety。
第二层是 MVCC 和 `HeapTupleSatisfiesVacuum()`。
`CLUSTER` 复制阶段用 `SnapshotAny` 读取旧 heap。
然后用 `OldestXmin` 判定 tuple 是否可丢。
这个判定不是简单当前 snapshot 可见性。
它是“是否还有任何可能需要这个 version”的全局 horizon 判断。
第三层是 buffer content lock 和 hint bits。
非 concurrent copy 会对旧 buffer 加 exclusive lock。
这不是为了保护新 heap。
它是为了让 `HeapTupleSatisfiesVacuum()` 能可靠设置 hint bits，供后续 update-chain 判断使用。
第四层是 predicate lock promotion。
tuple 移动后，旧 page/tuple predicate lock 不再能定位同一个事实。
所以系统把它们提升到 relation 级。
这是保守但正确的替代。
第五层是 WAL/fsync。
新 page、TOAST rows、logical mapping、storage create/drop 各有自己的持久化协议。
第六层是 relcache/catcache invalidation。
catalog swap 后，其他 backend 必须丢弃旧 relfilenode、toast link、stats 等缓存语义。
第七层是 transaction end cleanup。
pending delete 在 commit/abort 区分新旧 storage 命运。
这几层不能互相替代。
例如：

```text
AccessExclusiveLock 不保证 crash 后文件一致。
WAL 不阻止其他 backend 用旧 relcache。
relcache invalidation 不负责删除旧文件。
MVCC visibility 不负责 index TID 重建。
```

这也是本节的核心抽象：
一个大替换操作通常不是一个“全能锁”或“一条 WAL”解决的。
它是多个只负责自己边界的小协议叠在一起。

## 16. 错误路径、异常路径与 fallback

第一类异常是 copy phase ERROR。
可能来自：

```text
用户取消
OOM
tuple 过大
TOAST 写失败
磁盘写失败
checksum 校验失败
ALTER TABLE 表达式转换失败
约束验证失败
```

如果发生在 swap 前，旧表 catalog 和旧 storage 没变。
新 heap 是当前事务创建的对象。
abort 时通过 pending delete 删除新 relfilenode。
第二类异常是 swap 后、commit 前 ERROR。
这看起来危险，但 pending delete 设计正是为它存在。
catalog update 回滚后，原表指回旧 relfilenode。
旧 relfilenode 的 commit-delete 不执行。
新 relfilenode 的 abort-delete 执行。
第三类异常是 deadness 判断不精确。
`HeapTupleSatisfiesVacuum()` 基于 `OldestXmin` 的 dead/recently dead 判断可能不能完全解析 update chain。
`rewrite_heap_dead_tuple()` 和 `end_heap_rewrite()` 都选择保守处理。
能确定 dead 的释放。
不能确定的写出。
第四类异常是 TOAST 表被独立 autovacuum。
`copy_table_data()` 如果旧表有 TOAST，会锁旧 toast relation。
注释说原因是 autovacuum 可以独立 vacuum toast table。
如果它用更晚的 `OldestXmin` 删除了 rewrite 仍认为 recently dead 的 toast rows，copy 就可能失败。
所以 TOAST lock 是 visibility 边界的一部分。
第五类异常是 logical mapping 文件 crash。
mapping file 名字包含 rewrite xid。
未提交 rewrite 的 mapping 可被忽略。
WAL record 带 offset，redo 时可以 truncate。
checkpoint 会清理不再需要的 mapping。
第六类异常是系统 catalog 或 mapped relation。
普通用户表可以交换 `pg_class.relfilenode`。
mapped catalog 要通过 relmapper。
`ALTER TABLE` 直接拒绝系统 relation rewrite。
`CLUSTER/VACUUM FULL` 路径有更复杂的 mapped relation 支持，但对 toast link、tablespace、persistence 做了 emergency backstop。
第七类 fallback 是扫描方式选择。
`CLUSTER` 并不一定使用 index scan。
如果 btree index 排序经 planner 判断不划算，可以 seqscan 后 tuplesort。
这影响观测到的 phase、临时文件和 IO 模型，但不改变替换边界。

## 17. 成本、资源与跨模块传播

heap rewrite 是高放大操作。
成本随 heap tuple 数、recently dead tuple、update chain、TOAST 数据量、index 数量、WAL/fsync 压力、sort spill、锁等待和 cache churn 扩张。
每个需保留 tuple 都要读、判断、变形或复制、写入新 heap。
`rewriteheap.c` 还要为未解析的 chain 端点维护 hash table。
极端情况下，例如同一事务中大表全量 UPDATE 后立刻 CLUSTER，recently dead 版本太多可能耗尽内存。
实现没有 spill-to-disk，这是社区接受的正常 workload 取舍。
宽字段会触发 toast rewrite、toast index insert、toast value OID 检查。
索引越多，`finish_heap_swap()` 的 rebuild phase 越重。
普通 rewrite 的 `AccessExclusiveLock` 等待时间由“排队等待锁 + copy + swap + reindex + cleanup”共同组成。
新 heap page 绕过 shared buffers 写入，命令完成后第一次读还会重新进入 shared buffers。
资源传播路径可以概括为：

```text
heap size -> data read/write IO
live+recently_dead tuple -> rewrite CPU/memory/WAL
wide tuple -> TOAST heap/index IO
index count -> reindex CPU/WAL/IO
sort choice -> temp file
lock duration -> application latency
checkpoint timing -> fsync behavior
logical slots -> rewrite mapping files
```

## 18. 观测与诊断入口

本节锚定的 runtime truth 是：

```text
表重写完成后，原表 OID 不变，但物理 relfilenode/path 通常变化；在命令完成前，新物理内容不可被普通查询观察为原表。
```

直接观测入口：

```sql
SELECT c.oid,
       c.relname,
       c.relfilenode,
       c.reltoastrelid,
       c.relfrozenxid,
       c.relminmxid,
       pg_relation_filepath(c.oid) AS path
FROM pg_class c
WHERE c.relname IN ('rewrite_demo', 'rewrite_demo_pkey', 'rewrite_demo_k_idx');
```

`oid` 稳定，`relfilenode/path` 可能变化，`relfrozenxid/relminmxid` 可能推进。
`reltoastrelid` 是否变化，取决于 TOAST swap 策略和命令类型。
进度入口：

```sql
SELECT *
FROM pg_stat_progress_cluster
WHERE relid = 'rewrite_demo'::regclass;
```

这个分支内部使用 `PROGRESS_COMMAND_REPACK`，`pg_stat_progress_cluster` 是兼容视图。
底层也可以看：

```sql
SELECT *
FROM pg_stat_progress_repack
WHERE relid = 'rewrite_demo'::regclass;
```

常见 phase 包括：

```text
seq/index scanning heap, sorting tuples, writing new heap,
swapping relation files, rebuilding index, final cleanup
```

锁等待入口：

```sql
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query ILIKE '%rewrite_demo%';
```

```sql
SELECT locktype, relation::regclass, mode, granted, pid
FROM pg_locks
WHERE relation = 'rewrite_demo'::regclass
ORDER BY granted, mode;
```

WAL 和 TOAST 只能近似归因：

```sql
SELECT wal_records, wal_fpi, wal_bytes
FROM pg_stat_wal;
```

```sql
SELECT c.oid, c.relname, c.relfilenode, pg_relation_filepath(c.oid) AS path
FROM pg_class c
WHERE c.oid = (
  SELECT reltoastrelid FROM pg_class WHERE oid = 'rewrite_demo'::regclass
);
```

注意 `CLUSTER` 自身不是普通 executor query，`EXPLAIN` 不能覆盖它。
WAL 统计是实例累计粒度，需要实验前后取差值。
只能断点或日志观察的状态：

```text
rs_unresolved_tups / rs_old_new_tid_map
raw_heap_insert page flush
rd_toastoid
logical rewrite mapping
pendingDeletes.atCommit
```

建议断点集中在：

```text
cluster_rel / make_new_heap / copy_table_data
heapam_relation_copy_for_cluster
begin_heap_rewrite / rewrite_heap_tuple / raw_heap_insert / end_heap_rewrite
swap_relation_files / finish_heap_swap
RelationCreateStorage / RelationDropStorage / smgrDoPendingDeletes
toast_save_datum
```

诊断时要小心三类误判。
第一，`relfilenode` 变化不等于 OID 变化。
第二，progress phase 到 `swapping relation files` 不代表已经 commit。
第三，WAL bytes 上升不能直接说明瓶颈在 WAL flush，也可能是 sort/temp/io/index rebuild。

## 19. 课堂实验 1：观察 OID 稳定与 relfilenode 替换

目标：
看到 `CLUSTER` 后原表 OID 稳定，但物理文件身份变化。
准备、记录、执行、复查：

```sql
DROP TABLE IF EXISTS rewrite_demo;
CREATE TABLE rewrite_demo (
  id bigint PRIMARY KEY,
  k int NOT NULL,
  payload text
);
INSERT INTO rewrite_demo
SELECT g,
       (100000 - g)::int,
       repeat(md5(g::text), 20)
FROM generate_series(1, 100000) AS g;
CREATE INDEX rewrite_demo_k_idx ON rewrite_demo(k);
ANALYZE rewrite_demo;
```

记录 rewrite 前的物理身份，执行 `CLUSTER`，再复查：

```sql
SELECT c.oid,
       c.relfilenode,
       c.reltoastrelid,
       pg_relation_filepath(c.oid) AS path,
       c.relfrozenxid,
       c.relminmxid
FROM pg_class c
WHERE c.oid = 'rewrite_demo'::regclass;

CLUSTER rewrite_demo USING rewrite_demo_k_idx;

SELECT c.oid,
       c.relfilenode,
       c.reltoastrelid,
       pg_relation_filepath(c.oid) AS path,
       c.relfrozenxid,
       c.relminmxid
FROM pg_class c
WHERE c.oid = 'rewrite_demo'::regclass;
```

你应该看到：

```text
oid: 不变
relfilenode/path: 通常变化
relfrozenxid/relminmxid: 可能推进
```

源码回扣：

```text
cluster_rel()
  -> make_new_heap()
  -> copy_table_data()
  -> finish_heap_swap()
  -> swap_relation_files()
```

如果表很小，命令很快完成。
可以增加数据量或 payload 宽度，让 progress view 更容易抓到。

## 20. 课堂实验 2：观察 rewrite 失败后旧表仍完整

目标：
验证 copy phase 中失败不会把旧表替换成半成品。
用类型转换失败触发 `ALTER TABLE` rewrite 中断：

```sql
DROP TABLE IF EXISTS rewrite_fail;
CREATE TABLE rewrite_fail (
  id int PRIMARY KEY,
  v text
);
INSERT INTO rewrite_fail VALUES
  (1, '10'),
  (2, '20'),
  (3, 'not_an_int');

SELECT oid, relfilenode, pg_relation_filepath(oid) AS path
FROM pg_class
WHERE oid = 'rewrite_fail'::regclass;

ALTER TABLE rewrite_fail
  ALTER COLUMN v TYPE int
  USING v::int;

SELECT oid, relfilenode, pg_relation_filepath(oid) AS path
FROM pg_class
WHERE oid = 'rewrite_fail'::regclass;
SELECT * FROM rewrite_fail ORDER BY id;
```

你应该看到：

```text
ALTER TABLE 报 invalid input syntax for type integer。
relfilenode/path 没有切到半成品。
旧数据仍然可读。
```

源码解释：

```text
make_new_heap() 已经创建 transient heap。
ATRewriteTable() 在表达式转换中 ERROR。
事务 abort。
RelationCreateStorage() 登记的新 relfilenode atCommit=false，被 smgrDoPendingDeletes(false) 删除。
旧 relfilenode 没有被 swap。
```

## 21. 课堂实验 3：TOAST 与 progress

目标：
观察宽 tuple 重写时 TOAST relation 和 progress phase。
准备宽表并查看主表、TOAST 和 progress：

```sql
DROP TABLE IF EXISTS rewrite_toast;
CREATE TABLE rewrite_toast (
  id bigint PRIMARY KEY,
  k int,
  payload text
);
INSERT INTO rewrite_toast
SELECT g,
       (100000 - g)::int,
       repeat(md5(g::text), 200)
FROM generate_series(1, 50000) AS g;
CREATE INDEX rewrite_toast_k_idx ON rewrite_toast(k);
ANALYZE rewrite_toast;

WITH main AS (
  SELECT oid, reltoastrelid FROM pg_class WHERE oid = 'rewrite_toast'::regclass
)
SELECT 'main' AS kind, c.oid, c.relname, c.relfilenode, pg_relation_filepath(c.oid) AS path
FROM pg_class c JOIN main m ON c.oid = m.oid
UNION ALL
SELECT 'toast', c.oid, c.relname, c.relfilenode, pg_relation_filepath(c.oid)
FROM pg_class c JOIN main m ON c.oid = m.reltoastrelid;
```

另开 session 执行：

```sql
CLUSTER rewrite_toast USING rewrite_toast_k_idx;
```

原 session 轮询：

```sql
SELECT relid::regclass, phase, heap_blks_total, heap_blks_scanned,
       heap_tuples_scanned, heap_tuples_written, index_rebuild_count
FROM pg_stat_progress_cluster
WHERE relid = 'rewrite_toast'::regclass;
```

命令完成后再次查主表和 toast 表。
重点观察：

```text
主表 OID 稳定。
主表 relfilenode 变化。
TOAST relation OID/relfilenode 的变化取决于 swap 策略。
progress 中能看到 scan/sort/write/swap/rebuild/cleanup，但看不到 rs_unresolved_tups。
```

回到源码解释：

```text
copy_table_data() 设置 rd_toastoid。
raw_heap_insert() 可能调用 heap_toast_insert_or_update()。
toast_save_datum() 选择 toast pointer OID 和 value ID。
swap_relation_files() 递归处理 toast relation 和 toast index。
```

## 22. 常见误区

- `CLUSTER` 不是只重排 live rows；recently dead version 也可能被复制。
- `ALTER TABLE ... CLUSTER ON` 只改 `pg_index.indisclustered`，不做物理重写。
- `AccessExclusiveLock` 只解决并发访问；失败清理还依赖 pending delete、WAL/fsync 和 invalidation。
- `relfilenode` 变化不等于表被 drop/recreate；OID 才是用户对象身份。
- TOAST pointer 里有 relation OID 和 value OID，swap by content 与 swap by links 是不同边界。
- 主表普通索引通常在 heap swap 后重建；旧 index 只是排序来源或 eligibility check。
- WAL 统计不能完整解释 rewrite 性能，还要看 data IO、sort、TOAST、index build、lock wait 和 checkpoint。
- `pg_stat_progress_cluster` 看不到 update-chain hash、pendingDeletes、rd_toastoid、logical rewrite mapping。

## 23. 讨论题

1. 为什么 `CLUSTER` 不能把旧 heap page 原地重排，然后再逐个更新 index TID？
2. 如果 copy phase 已经写了很多新 heap page，但还没 swap，此时 ERROR 后哪些状态需要清理，哪些状态不需要恢复？
3. `rewrite_heap_tuple()` 为什么要用 `xmin + old tid` 作为 hash key，而不是只用 old TID？
4. 为什么 dead tuple 不复制，但仍然可能调用 `rewrite_heap_dead_tuple()`？
5. `rd_toastoid` 为什么设置在新 main heap 的 relcache 上，而不是设置在 TOAST relation 上？
6. 为什么主表普通索引在 `finish_heap_swap()` 里重建，而不是在 `raw_heap_insert()` 后同步插入？
7. 如果存在 logical replication slot，heap rewrite 为什么需要额外 mapping 文件？
8. `AccessExclusiveLock`、WAL、relcache invalidation、pending delete 各自保证什么？哪个机制不能被另一个替代？
9. `ALTER TABLE` rewrite 为什么可以用普通 `table_tuple_insert()`，而 `CLUSTER/VACUUM FULL` 非 concurrent 路径不能？
10. 观测到 `relfilenode` 没有变化，能否断定没有 rewrite？还需要考虑哪些 relation kind、mapped relation 或命令路径？

## 24. 本节小结

本节唯一主问题是：
为什么 heap rewrite 要先创建新 relfilenode，再在 catalog/relmapper 边界上替换。
核心链路是：

```text
make_new_heap()
  -> copy_table_data()/ATRewriteTable()
  -> heap rewrite or normal insert
  -> finish_heap_swap()
  -> swap_relation_files()
  -> reindex_relation()
  -> performDeletion()
```

`CLUSTER/VACUUM FULL` 用 `SnapshotAny + HeapTupleSatisfiesVacuum()` 判断 tuple 命运，用 rewrite hash 修复 `t_ctid`，用 `raw_heap_insert()` 保留旧 visibility，再由 bulk writer 写新 heap page。
`ALTER TABLE` 侧重表达式重算和 constraint validation，可以把旧行转成当前命令插入的新 tuple。
两者复制路径不同，但替换边界相同。
TOAST 处理 pointer OID/value OID，index rebuild 处理新 heap TID，logical rewrite mapping 处理 old locator/TID 到 new locator/TID，relcache invalidation 处理缓存语义，pending delete 处理 commit/abort 后的物理文件命运。
可以直接观测的是 OID 稳定、relfilenode/path 变化、progress phase、lock wait、WAL 增量、toast relation。
不能直接观测的是 rewrite hash table、pendingDeletes 内容、rd_toastoid、logical mapping 启用条件。
从本节抽象出的可迁移规律是：

```text
大对象替换不要把“构造新事实”和“宣布新事实”混在一起。
先在私有或不可见物理身份上完整构造。
再用短事务边界切换逻辑引用。
最后让 cleanup、invalidation、redo 和 background fsync 各自完成自己的部分。
```

这个规律也适用于索引重建、物化视图刷新、relation file copy、storage engine compaction 等 copy-on-replace 设计。
